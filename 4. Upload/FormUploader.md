# FormUploader 表单上传

该类为内部类

| 名称       | 类型       | 描述                              |
| ---------- | ---------- | --------------------------------- |
| bucket_uploader | BucketUploader | 上传管理器 |
| multipart_encoder | Multipart 编码器，该类型在不同编程语言种可能有不一样的实现 | 编码 Multipart 类型的数据 |
| client | Client | HTTP 客户端 |

### 支持接口

#### upload_file()

##### 接受参数

| 名称       | 类型       | 描述       |
| ---------- | ---------- | ---------- |
| file_path     | Path     | 文件路径   |
| file_name     | String     | 文件名，可选   |
| mime     | String     | MIME 类型，可选   |
| upload_token     | UploadToken     | 上传凭证 |
| key   | String | 对象名称                |
| vars  | map<String, String> | 自定义变量 |
| metadata  | map<String, String> | 元信息 |
| checksum_enabled | bool | 是否启用校验，总是默认为启用 |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| result | UploadResult | 上传结果 |
| error | HTTPError | HTTP 错误信息 |

##### 伪代码

```
multipart_encoder.add("token", upload_token)
if key {
	multipart_encoder.add("key", key)
}
if checksum_enabled {
	multipart_encoder.add("crc32", calculate_crc32(file_path))
}
for key, val in vars {
	multipart_encoder.add("x:${key}", val)
}
for key, val in metadata {
	multipart_encoder.add("x-qn-meta-${key}", val)
}
multipart_encoder.add("file", open(file_path), file_name, mime)

body = multipart_encoder.encode() // 这里将其转换为二进制数组，每次在使用时自动 rewind。如果编程语言只能提供输入流，那么需要在每次发出 HTTP 请求前显式 rewind。
prev_err = null
for up_urls in bucket_uploader.up_urls_list {
	response, err = client.send_request(POST, "/", up_urls, {}, {}, body, null, true, true, make_response_callback(upload_token.policy()))
	if err {
		switch err.retry_kind {
			case Retryable, HostUnretryable, ZoneUnretryable:
				prev_err = err
			default:
				return null, err
		}
	} else {
		return make_upload_result(response), null
	}
}
null, prev_err
```

#### upload_stream()

表单的 `upload_stream()` 无法支持检验和，但可以支持多区域重试

##### 接受参数

| 名称       | 类型       | 描述       |
| ---------- | ---------- | ---------- |
| stream | Reader  | 输入流 |
| file_name     | String     | 文件名，可选   |
| mime     | String     | MIME 类型，可选   |
| upload_token     | UploadToken     | 上传凭证 |
| key   | String | 对象名称                |
| vars  | map<String, String> | 自定义变量 |
| metadata  | map<String, String> | 元信息 |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| result | UploadResult | 上传结果 |
| error | HTTPError | HTTP 错误信息 |

##### 伪代码

```
multipart_encoder.add("token", upload_token)
if key {
	multipart_encoder.add("key", key)
}
for key, val in vars {
	multipart_encoder.add("x:${key}", val)
}
for key, val in metadata {
	multipart_encoder.add("x-qn-meta-${key}", val)
}
multipart_encoder.add("file", stream, file_name, mime)

body = multipart_encoder.encode() // 这里将其转换为二进制数组，每次在使用时自动 rewind。如果编程语言只能提供输入流，那么这里可能无法支持多区域重试
prev_err = null
for up_urls in bucket_uploader.up_urls_list {
	response, err = client.send_request(POST, "/", up_urls, {}, {}, body, null, true, true, make_response_callback(upload_token.policy()))
	if err {
		switch err.retry_kind {
			case Retryable, HostUnretryable, ZoneUnretryable:
				prev_err = err
			default:
				return null, err
		}
	} else {
		return make_upload_result(response), null
	}
}
null, prev_err
```

#### 补充说明

对于上传文件接口，用户如果没有传入 `file_name` 和 `mime` 参数，此时 `file_name` 可以通过给定的 `file_path` 得出，而 `mime` 可以通过 `file_path` 或 `file_name` 的扩展名得出，计算方式由编程语言所使用的库决定。

而对于上传输入流接口，如果用户没有传入 `file_name` 和 `mime` 参数，只能由七牛服务器端来决定处理方案。
