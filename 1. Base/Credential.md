# Credential 认证信息

| 名称       | 类型   | 类型                |
| ---------- | ------ | ------------------- |
| access_key | String | AccessKey，必须存在 |
| secret_key | String | SecretKey，必须存在 |

## 支持接口

### sign()

签发数据

#### 接受参数

| 名称       | 类型       | 类型                              |
| ---------- | ---------- | --------------------------------- |
| data | [uint8] | 被签名的数据 |

#### 返回参数

| 名称       | 类型       | 类型                              |
| ---------- | ---------- | --------------------------------- |
| signed_data | String | 签名 |

#### 伪代码实现

```
"${access_key}:${urlsafe_base64_encode(hmac_sha1_digest(data, secret_key))}"
```

### sign_with_data()

签发数据（包含数据本身）

#### 接受参数

| 名称       | 类型       | 类型                              |
| ---------- | ---------- | --------------------------------- |
| data | [uint8] | 被签名的数据 |

#### 返回参数

| 名称       | 类型       | 类型                              |
| ---------- | ---------- | --------------------------------- |
| signed_data | String | 签名（包含数据本身） |

#### 伪代码实现

```
encoded_data = urlsafe_base64_encode(data)
"${sign(encoded_data)}:${encoded_data}"
```

### authorization_v1_for_request()

针对给定的 HTTP 请求信息，生成 Authorization Header

#### 接受参数

| 名称       | 类型       | 类型                              |
| ---------- | ---------- | --------------------------------- |
| url | String | URL |
| content_type | String | Content-Type 信息 |
| body | [uint8] | 请求体 |

#### 返回参数

| 名称       | 类型       | 类型                              |
| ---------- | ---------- | --------------------------------- |
| authorization | String | Authorization Header |

#### 伪代码实现

```
parsed_url = parse_url(url)
data_to_sign = []
data_to_sign.append(parsed_url.path())
if parsed_url.query() {
	data_to_sign.append('?')
	data_to_sign.append(parsed_url.query())
}
data_to_sign.append('\n')
if content_type && body && content_type == "application/x-www-form-urlencoded" {
	data_to_sign.append(body)
}

"QBox ${sign(data_to_sign)}"
```

### authorization_v2_for_request()

针对给定的 HTTP 请求信息，生成 Authorization Header

#### 接受参数

| 名称       | 类型       | 类型                              |
| ---------- | ---------- | --------------------------------- |
| url | String | URL |
| method | String | HTTP 方法 |
| content_type | String | Content-Type 信息 |
| body | [uint8] | 请求体 |

#### 返回参数

| 名称       | 类型       | 类型                              |
| ---------- | ---------- | --------------------------------- |
| authorization | String | Authorization Header |

#### 伪代码实现

```
parsed_url = parse_url(url)
data_to_sign = []
data_to_sign.append(method.upper())
data_to_sign.append(' ')
data_to_sign.append(parsed_url.path())
if parsed_url.query() {
	data_to_sign.append('?')
	data_to_sign.append(parsed_url.query())
}
data_to_sign.append("\nHost: ${parsed_url.host()}")
if parsed_url.port() { // 这里的语义是，URL 中是否显式包含端口号
	data_to_sign.append(":${parsed_url.port()}")
}
data_to_sign.append('\n')
if content_type {
	data_to_sign.append("\nContent-Type: ${content_type}\n\n")
	if body && (content_type == "application/x-www-form-urlencoded" || content_type == "application/json") {
		data_to_sign.append(body)
	}
} else {
	data_to_sign.append('\n')
}

"Qiniu ${sign(data_to_sign)}"
```