# ResumeableUploader 可恢复上传

该类为内部类

| 名称              | 类型                                                       | 描述                      |
| ----------------- | ---------------------------------------------------------- | ------------------------- |
| bucket_uploader   | BucketUploader                                             | 上传管理器                |
| multipart_encoder | Multipart 编码器，该类型在不同编程语言种可能有不一样的实现 | 编码 Multipart 类型的数据 |
| client            | Client                                                     | HTTP 客户端               |

### 支持接口

#### upload_file()

##### 接受参数

| 名称             | 类型                | 描述                         |
| ---------------- | ------------------- | ---------------------------- |
| file_path        | Path                | 文件路径                     |
| file_name        | String              | 文件名，可选                 |
| mime             | String              | MIME 类型，可选              |
| upload_token     | UploadToken         | 上传凭证                     |
| key              | String              | 对象名称                     |
| vars             | map<String, String> | 自定义变量                   |
| metadata         | map<String, String> | 元信息                       |
| checksum_enabled | bool                | 是否启用校验，总是默认为启用 |

##### 返回参数

| 名称   | 类型         | 描述          |
| ------ | ------------ | ------------- |
| result | UploadResult | 上传结果      |
| error  | HTTPError    | HTTP 错误信息 |

##### 伪代码

```
fn encode_key(key) {
	if key == null {
		"~"
	} else {
		urlsafe_base64_encode(key)
	}
}

fn init_parts(path, up_urls, authorization, callback) {
	response, err = client.send_request(POST, path, up_urls, {Authorization: authorization}, {}, null, null, true, true, callback)
	if err {
		return nil, err
	}
	parse_json(response.body).get("uploadId"), nil
}

fn upload_part(path, up_urls, authorization, part, md5, callback) {
	headers = {Authorization: authorization}
	if md5 {
		headers.set("Content-MD5", md5)
	}
	response, err = client.send_request(PUT, path, up_urls, headers, {}, null, null, true, true, callback)
	if err {
		return nil, err
	}
	parse_json(response.body).get("etag"), nil
}

fn complete_parts(path, up_urls, authorization, callback, parts, fname, mime, metadata, vars) {
	response, err = client.send_request(
		POST, path, up_urls, 
		{Authorization: authorization}, {}, 			
		dump_json({
			parts: parts, fname: fname, mimeType: mime, metadata: metadata, 
			customVars: vars.reduce({}, (vars, key, val) => vars.add("x:${key}", val)
		}), 
		null, true, true, callback)
	if err {
		return nil, err
	}
	make_upload_result(response), nil
}

fn upload(up_urls, file, file_name, mime, upload_token, key, vars, metadata, block_size, checksum_enabled) {
	base_path = "/buckets/${bucket_uploader.bucket_name}/objects/{encode_key(key)}/uploads"
	authorization = "UpToken " + upload_token
	part_number = 0
	completed_parts = []
	callback = make_response_callback(upload_token.policy())
	upload_id = init_parts(base_path, up_urls, authorization, callback)
	loop {
		part_number += 1
		part, ok = file.read(block_size) // 从文件中读取一个 block，可能会出错
		if !ok {
			return nil, HTTPError(Unretryable, false, IOError)
		} else if !part {
			break // 没有读到更多内容，表示 EOF，退出循环
		}
		if checksum_enabled {
			md5 = md5(part)
		}
		etag = upload_part("${base_path}/${upload_id}/${part_number}", up_urls, authorization, part, md5, callback)
		completed_parts.append({etag: etag, part_number: part_number})
	}
	complete_parts("${base_path}/${upload_id}", up_urls, authorization, callback, completed_parts, file_name, mime, metadata, vars)
}

file = open(file_path)
block_size = bucket_uploader.upload_manager.config.upload_block_size
prev_err = null
for up_urls in bucket_uploader.up_urls_list {
	file.rewind() // 文件指针移动到起始位置
	result, err = upload(up_urls, file, file_name, mime, upload_token, key, vars, metadata, block_size, checksum_enabled)
	if err {
		switch err.retry_kind {
			case Retryable, HostUnretryable, ZoneUnretryable:
				prev_err = err
			default:
				return null, err
		}
	} else {
		return result, null
	}
}
null, prev_err
```

#### upload_stream()

可重试的 `upload_stream()` 无法支持多区域重试，但可以支持检验和

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
| checksum_enabled | bool                | 是否启用校验，总是默认为启用 |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| result | UploadResult | 上传结果 |
| error | HTTPError | HTTP 错误信息 |

##### 伪代码

```
block_size = bucket_uploader.upload_manager.config.upload_block_size

# upload() 实现参考之前的伪代码
upload(up_urls.get(0), stream, file_name, mime, upload_token, key, vars, metadata, block_size, checksum_enabled)
```

#### 补充说明

对于上传文件接口，用户如果没有传入 `file_name` 和 `mime` 参数，此时 `file_name` 可以通过给定的 `file_path` 得出，而 `mime` 可以通过 `file_path` 或 `file_name` 的扩展名得出，计算方式由编程语言所使用的库决定。

而对于上传输入流接口，如果用户没有传入 `file_name` 和 `mime` 参数，只能由七牛服务器端来决定处理方案。