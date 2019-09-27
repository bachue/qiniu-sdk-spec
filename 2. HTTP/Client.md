# Client HTTP 客户端

## HTTPError

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| retry_policy | RetryPolicy     | 重试策略                |
| is_retry_safe | bool | 是否重试安全 |
| error | Error(异常类型与编程语言相关) | 原始错误信息 |

## RetryPolicy 重试策略

枚举类，有四种可能的值

| 名称       | 描述       |
| ---------- | ---------- |
|Retryable|总是可重试的|
|ZoneUnretryable|当前主机不可重试|
|HostUnretryable|当前域名不可重试|
|Unretryable|不可重试的|

### 重试策略，幂等性与重试安全之间的关系

当多次发送同一个 HTTP 请求具有相同效果，则认为该 HTTP 请求具有幂等性。根据 HTTP RESTful API 的设计理念，GET / PUT / HEAD / PATCH / DELETE 请求被认为总是幂等的，而对于非幂等的 HTTP 方法，可以手动设置 idempotent 属性将其设置为幂等。如果一个幂等的 HTTP 请求发生了错误，则总是被认为是重试安全的。

对于非幂等的 HTTP 请求发生了错误，则是否重试安全取决于该错误发生时，请求是否已经完整发出。如果错误发生在域名解析阶段，连接阶段或请求发送期间，则一般认为是重试安全的，如果请求已经完整发出，则之后发生的错误是重试不安全的。

对于每一种 HTTP 请求时发生的错误，都应该设置其重试策略，以便于外部 HTTP 客户端检测到错误时，决定将其重试，还是切换主机后再重试，还是不予重试直接抛出错误。重试策略共四种，已经在上面的表格中列出，一般来说，如果是由于 HTTP 请求自身格式存在问题，或是收到了来自服务器的 4xx 响应，则被认为是不可重试的，而如果是服务器发生异常，或域名解析无法成功，或无法连接到，或 SSL 认证无法通过，则被认为只是当前主机不可重试，可以切换到其他主机再重试。如果只是连接超时，或是发送接受数据存在超时或突然中断，则被认为是可重试的。

即使根据重试策略需要重试的错误，客户端也未必一定会重试，这取决于两个因素，是否重试安全和是否已经达到重试次数上限。因此，对于容易发生错误的请求（例如上传下载之类的），要尽可能将其 HTTP 调用设置为幂等，否则就可能因为发生错误的时机不佳而无法重试。

## Client

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| config     | Config     | 客户端所用配置信息                |

### 支持接口

#### send_request()

发送 HTTP 请求

##### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| path | String | HTTP 请求的路径部分（不含 query） |
| hosts | [String] | HTTP 请求可用域名 |
| headers | map<String, String> | HTTP 请求头 |
| query | map<String, String> | HTTP URL 中的 query 部分 |
| body | [uint8] | HTTP 请求体 |
| token | Token | HTTP 签名方式 |
| idempotent | bool | 本次请求是否具有幂等性 |
| full_read | bool | 是否将整个响应体全部读入内存 |
| response_callback | fn(Response, Request) -> HTTPError | 回调函数 |

##### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| response | Response | HTTP 响应 |
| error | HTTPError | HTTP 错误信息 |

##### 伪代码实现

```
fn is_retry_safe(request, idempotent, err) {
	switch request.method {
	case GET, PUT, HEAD, PATCH, DELETE:
		true
	default:
		idempotent || err.is_retry_safe
	}
}

fn make_error_from_response(response) {
	switch response.status {
	case 400..499, 501, 573, 608, 612, 614, 615, 616, 619, 630, 631, 640, 701:
		HTTPError(Unretryable, false, parse_json(response.body).get("error"))
	case 502, 503, 571:
		HTTPError(HostUnretryable, false, parse_json(response.body).get("error"))
	default:
		HTTPError(Retryable, false, parse_json(response.body).get("error"))
	}
}

fn try_url(url, headers, body, token, idempotent, full_read, response_callback) {
	request = build_request(url, headers, body)
	if token {
		token.sign(request)
	}
	for retried = 0; retried < client.config.http_request_retries; retried += 1 {
		response, err = client.call(request) // 发送 HTTP 请求，这里假设该方法会自动处理 HTTP 转发
		if !err {
			if (200..300).include(response.status_code) { // 只有返回值在 [200, 300) 之间才被允许
				if full_read {
					response.body, ok = response.body.read() // 从响应体中读出全部数据，可能会出错
					if !ok {
						err = HTTPError(Retryable, false, IOError) // 读取请求体失败，被认为是 IO 错误，可重试但重试不安全
					} else if response.body.len() != response.header("ContentLength") {
						err = HTTPError(Retryable, false, EOFError) // 读取到的数据量不匹配，被认为是 EOF 错误，可重试但重试不安全
					}
				}
				if !err {
					err = response_callback(response, request) // 即使响应体状态码正确，依然需要调用回调函数判定其内容是否确实正确
				}
				if !err {
					return response, null
				}
			} else {
				err = make_error_from_response(response)   // 根据状态码决定错误类型
			}
		}
		if err && err.retry_kind == Retryable && is_retry_safe(request, idempotent, err) {
			wait(client.config.http_request_retry_relay)
			continue
		} else {
			return null, err
		}
	}
	null, err // 重试次数达到上限，返回最近的一个错误
}

prev_err = null
for host in hosts {
	if client.config.domains_manager.is_frozen(host) {
		continue
	}
	response, err = try_url(make_url(host, path, query))
	if err && (err.retry_kind == Retryable || err.retry_kind == HostUnretryable) && is_retry_safe(request, idempotent, err) {
		client.config.domains_manager.freeze(host, client.config.domain_freeze_duration)
		prev_err = err
	} else {
		return response
	}
}
prev_err || HTTPError(HostUnretryable, true, NoHostAvailable) // 如果 hosts 为空或所有 hosts 都已经被冻结，则返回 NoHostAvailable 错误
```

