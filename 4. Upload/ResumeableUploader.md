# ResumeableUploader 可恢复上传

该类为内部类

| 名称              | 类型                                                       | 描述                      |
| ----------------- | ---------------------------------------------------------- | ------------------------- |
| bucket_uploader   | BucketUploader                                             | 上传管理器                |
| upload_logger | UploadLogger | 上传日志，可选 |
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
| vars             | [String:String] | 自定义变量                   |
| metadata         | [String:String] | 元信息                       |
| uploading_progress_callback | fn(uint, uint) | 上传进度回调方法，第一个参数为已上传的数据量，第二个参数为总共将要上传的数据量 |
| thread_pool | ThreadPool | 线程池，可以为空 |
| recorder | UploadRecord | 上传日志记录仪，用于尝试恢复之前的上传 |
| checksum_enabled | bool                | 是否启用校验，总是默认为启用 |

##### 返回参数

| 名称   | 类型         | 描述          |
| ------ | ------------ | ------------- |
| result | UploadResult | 上传结果      |
| error  | HTTPError    | HTTP 错误信息 |

##### 伪代码实现

```
fn encode_key(key) {
	if key == null {
		"~"
	} else {
		urlsafe_base64_encode(key)
	}
}

fn make_response_callback(api_type) {
	(response, duration) -> {
		err = upload_response_callback(response)
		if !err {
			if upload_logger {
				upload_logger.log(make_upload_log_record_from_response(response, duration, api_type, response.body.len(), response.body.len()))
			}
		}
		err
	}
}

fn make_error_callback(api_type) {
	(base_url, err, duration) -> {
		if upload_logger {
			upload_logger.log(make_upload_log_record_from_error(duration, api_type, base_url, err, response.body.len()))
		}
	}
}

fn init_parts(path, up_urls, authorization) {
	response, err = client.send_request(POST, path, up_urls, {Authorization: authorization}, {}, null, null, true, true, false, null, null, make_response_callback(InitParts), make_error_callback(InitParts))
	if err {
		return null, err
	}
	parse_json(response.body).get("uploadId"), null
}

fn upload_part(path, up_urls, authorization, body, part_number, block_size, on_progress, on_error, record_medium) {
	headers = {Authorization: authorization, "Content-Type": "application/octet-stream"}
	if checksum_enabled {
		headers.set("Content-MD5", calculate_md5(body))
	}
	original_error_callback = make_error_callback(UploadPart)
	error_callback = (base_url, err, duration) -> {
		on_error()
		original_error_callback(base_url, err, duration)
	}
	response, err = client.send_request(PUT, path, up_urls, headers, {}, body, null, true, true, false, on_progress, null, make_response_callback(UploadPart), error_callback)
	if err {
		return null, err
	}
	etag = parse_json(response.body).get("etag")
	if record_medium {
		record_medium.append(etag, part_number, block_size)
	}
	etag, null
}

fn complete_parts(path, up_urls, authorization, completed_parts) {
	completed_parts.sort_by((part) -> { part.part_number }) // 由于并发上传可能无法保证顺序，需要手动做一次排序
	response, err = client.send_request(
		POST, path, up_urls,
		{Authorization: authorization, "Content-Type": "application/json"}, {},
		dump_json({
			parts: completed_parts, fname: file_name, mimeType: mime, metadata: metadata,
			customVars: vars.reduce({}, (vars, key, val) => vars.add("x:${key}", val)
		}),
		null, true, true, false, null, null, make_response_callback(CompleteParts), make_error_callback(CompleteParts))
	if err {
		return null, err
	}
	make_upload_result(response), null
}

fn start_uploading_blocks(io_status_manager, uploaded, total_size, part_number, base_path, up_urls, authorization, completed_parts, block_size, record_medium, thread_pool) {
	// 这里创建多个原子变量，在线程池中使用
	uploaded_atomic = make_atomic(uploaded)
	completed_atomic = make_atomic(uploaded)
	part_number_atomic = make_atomic(part_number)
	// 创建一把锁，保护 completed_parts
	completed_parts_mutex = make_mutex()
	thread_pool.scope((scope) => { // 这里创建的是 Scope 线程，即当且仅当所有 Scope 线程都处理完毕后，scope() 方法才结束
    for _ in 0..thread_pool.size() { // 线程池有多大，就有多少线程同时并行
      scope.spawn(() => {
      	loop {
          body = io_status_manager.read(block_size) // 尝试获取文件的下一个 block
          if body {
            part_number = part_number_atomic.add(1) + 1
            last_block_uploaded = 0
            on_progress = (block_uploaded, _) -> {
            	if uploading_progress_callback {
            		added_size = block_uploaded - last_block_uploaded
            		last_block_uploaded = block_uploaded
            		uploading_progress_callback(completed_atomic.add(added_size) + added_size, total_size)
            	}
            }
            on_error = () -> {
            	completed_atomic.add(-last_block_uploaded)
            	last_block_uploaded = 0
            }
            etag, err = upload_part("${base_path}/${part_number}", up_urls, authorization, body, part_number, block_size, on_progress, on_error, record_medium)
            if err {
            	io_status_manager.error(err) // 告知 io_status_manager 发生了错误，其他线程也将无法获取任何数据，从而自动退出
            	return
            } else {
            	completed_parts_mutex.lock()
            	completed_parts.push({ etag: etag, part_number: part_number })
            	completed_parts_mutex.unlock()
            	uploaded_atomic.add(block_size)
            }
          } else {
          	return // 无法获取到下一个 block，线程结束
          }  	
      	}
      })
    }
	})
	err = io_status_manager.result()
	if !err {
		complete_parts(base_path, up_urls, authorization, completed_parts), uploaded_atomic.load()
	} else if err is IOError {
		null, HTTPError(Unretryable, false, IOError), uploaded_atomic.load()
	} else if err is HTTPError {
		null, err, uploaded_atomic.load()
	}
}

fn try_to_resume(base_path, file, total_size, block_size, authorization, thread_pool) {
	metadata, blocks, err = recorder.load(file_path, key)
	if err {
		return null, err
	}
	if !metadata {
		return null, null
	}
	record_medium, err = recorder.open_for_appending(file_path, key)
	if err {
		return null, err
	}
	io_offset = 0
	completed_parts = []
	blocks.for_each((block) -> {
		completed_parts.push({ etag: block.etag, part_number: block.part_number })
		io_offset += block.block_size
	})
	err = file.seek(io_offset)
	if err {
		return null, err
	}
	path = "${base_path}/${metadata.upload_id}"
	begin_at = now()
	result, err, uploaded = start_upload_blocks(make_io_status_manager(file), io_offset, total_size, blocks.len(), path, metadata.up_urls, authorization, completed_parts, block_size, record_medium, thread_pool)
	if err {
		if upload_logger {
			upload_logger.log(make_upload_log_record_from_chunked_error(now() - begin_at, ChunkedV2, err, uploaded - io_offset, total_size - io_offset))
		}
	} else {
		if upload_logger {
			upload_logger.log(make_upload_log_record_from_chunked_success(now() - begin_at, ChunkedV2, uploaded - io_offset, uploaded - io_offset))
		}
	}
	result, err
}

fn upload(base_path, up_urls, file, total_size, block_size, authorization, thread_pool) {
	upload_id, err = init_parts(path, up_urls, authorization)
	if err {
		return null, err, 0
	}
	path = "${base_path}/${upload_id}"
	if file.path() { // 确定必须是文件，如果不是则无法记录
		record_medium = recorder.open_and_write_metadata(file.path(), key, upload_id, up_urls)
	}
	start_upload_blocks(make_io_status_manager(file), 0, total_size, 0, path, up_urls, authorization, [], block_size, record_medium, thread_pool)
}

fn upload_and_log(base_path, up_urls, file, total_size, block_size, authorization, thread_pool) {
	begin_at = now()
	result, err, uploaded = upload(base_path, up_urls, file, block_size, authorization, thread_pool)
	if err {
		if upload_logger {
			upload_logger.log(make_upload_log_record_from_chunked_error(now() - begin_at, ChunkedV2, err, uploaded, total_size))
		}
	} else {
		if upload_logger {
			upload_logger.log(make_upload_log_record_from_chunked_success(now() - begin_at, ChunkedV2, total_size, total_size))
		}
	}
	result, err
}

file = open(file_path)
total_size = file.size()
block_size = bucket_uploader.upload_manager.config.upload_block_size
authorization = "UpToken ${upload_token}"
base_path = "/buckets/${bucket_uploader.bucket_name}/objects/{encode_key(key)}/uploads"
thread_pool = thread_pool || bucket_uploader.thread_pool || make_thread_pool(3) // 优先使用参数中的线程池，如果不存在，使用 BucketUploader 的，如果依然不存在，则创建一个线程池，初始化为 3 个线程
result, err = try_to_resume(base_path, file, total_size, block_size, authorization, thread_pool) // 先尝试恢复，恢复失败也不要紧
if !err && result {
	return result, null
}
prev_err = null
for up_urls in bucket_uploader.up_urls_list { // 无法恢复，进入正常上传逻辑
	file.rewind() // 文件指针移动到起始位置
	result, err = upload_and_log(base_path, up_urls, file, total_size, block_size, authorization, thread_pool)
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
| vars  | [String:String] | 自定义变量 |
| metadata  | [String:String] | 元信息 |
| uploading_progress_callback | fn(uint, uint) | 上传进度回调方法，第一个参数为已上传的数据量，第二个参数为总共将要上传的数据量 |
| thread_pool | ThreadPool | 线程池，可以为空 |
| checksum_enabled | bool                | 是否启用校验，总是默认为启用 |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| result | UploadResult | 上传结果 |
| error | HTTPError | HTTP 错误信息 |

##### 伪代码实现

```
block_size = bucket_uploader.upload_manager.config.upload_block_size

# upload_and_log() 实现参考之前的伪代码实现
upload(up_urls.get(0), stream, file_name, mime, upload_token, key, vars, metadata, block_size, checksum_enabled)

authorization = "UpToken ${upload_token}"
base_path = "/buckets/${bucket_uploader.bucket_name}/objects/{encode_key(key)}/uploads"
thread_pool = thread_pool || bucket_uploader.thread_pool || make_thread_pool(3) // 优先使用参数中的线程池，如果不存在，使用 BucketUploader 的，如果依然不存在，则创建一个线程池，初始化为 3 个线程

upload_and_log(base_path, up_urls.get(0), stream, null, block_size, authorization, thread_pool)
```

#### 补充说明

对于上传文件接口，用户如果没有传入 `file_name` 和 `mime` 参数，此时 `file_name` 可以通过给定的 `file_path` 得出，而 `mime` 可以通过 `file_path` 或 `file_name` 的扩展名得出，计算方式由编程语言所使用的库决定。

而对于上传输入流接口，如果用户没有传入 `file_name` 和 `mime` 参数，只能由七牛服务器端来决定处理方案。
