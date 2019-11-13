# UploadLogger 上传日志

该类的实现中比较重要的是如何保护日志文件。在服务器端 SDK 的实现中，因为 `fork()` 的存在，上传日志可能在多个进程中被共享，从而造成并发冲突（例如一个进程正在追加日志文件，而另一个进程刚刚上传日志文件完毕，并清空了日志文件）。因此会使用文件锁来保护文件，默认情况下，会在追加日志文件时对文件加共享锁（对于大部分文件系统，都能保证追加文件本身是并发安全的），在上传日志文件时对文件加排他锁。但由于上传日志的时间与网速相关，可能会出现上传缓慢造成日志文件长期被加排他锁的情况，造成日志无法追加，上传因此停滞。因此，会在这种情况使用内存缓冲区来临时存储上传日志，并且在日志文件被解锁后立刻将缓冲区的内容全部写入日志文件中。如果上传日志过大，则会绕过上传日志，直接异步上传日志缓冲区的内容。当 UploadLogger 对象即将被析构，为了尽量不丢失内存缓冲区中的内容，也会直接异步上传日志缓冲区的内容。

而如果是移动端 SDK 则相对较为简单，由于 `fork()` 系统调用无法使用，且每个应用都有自己的目录，只需要自身应用目录中创建一个固定文件即可，无需文件锁保护。

| 名称          | 类型               | 描述                      |
| ------------- | ------------------ | ------------------------- |
| server_url | String | SDK 客户端所用配置信息                |
| upload_threshold | uint32 | 当日志缓冲区和日志文件的尺寸大于该值，将异步进行上传。单位为字节，默认为 4 KB。 |
| max_size | uint32 | 当日志缓冲区和日志文件的尺寸大于该值，将不会再像其中写入任何新的数据。单位为字节，默认为 4 MB。 |
|log_buffer|[u8]| 日志缓冲区 |
|log_buffer_lock|RWMutex| 日志缓冲区的读写锁，对于多线程环境，对日志缓存的读写应该使用读写锁保护 |
|log_file|File| 日志文件 |
|log_file_lock|RWMutex| 日志文件句柄的读写锁，对于多线程环境，对日志文件的读写应该使用读写锁保护 |
|lock_policy|LockPolicy| 文件锁策略，默认为 `LockSharedDuringAppendingAndLockExclusiveDuringUploading`，即在追加写文件时加共享锁，上传文件时上排他锁 |
| http_client        | Client             | HTTP 客户端               |
| upload_token | String | 上传凭证，每次上传新的文件时都应该对 upload_token 进行赋值，确保在上传日志时使用最新的 upload_token |

## 内部依赖结构体

### UploadLoggerRecord

上传日志条目

| 名称          | 类型               | 描述                      |
| ------------- | ------------------ | ------------------------- |
| status_code | int32      | HTTP 响应状态码或错误码，如果不填则为 "null"                |
| request_id | String | 七牛请求 ID         |
| host | String | 主机域名（不含端口，不含协议） |
| up_type | UpType | 上传 API 类型，取值见下文 |
| server_ip | IpAddr | 最终连接的远程服务器地址，必须同时支持 IPv4 和 IPv6 |
| server_port | uint16 | 最终连接的远程服务器端口 |
| duration | Duration（如果没有 Duration 类型，则使用 uint64，单位为毫秒） | HTTP 请求时长 |
| sent | uint64 | 已经发送的数据量，单位为字节 |
| error_message | String | 错误内容 |
| total_size | uint64 | 总共需要发送的数据量，单位为字节 |
| timestamp | uint64 | 当前应用程序时间戳 |

### UpType

枚举类，包含 `Form`，`ChunkedV2`，`InitParts`，`UploadPart`，`CompleteParts` 等五种可选值，表明上传的类型。

### LockPolicy

枚举类，包含 `LockSharedDuringAppendingAndLockExclusiveDuringUploading`，`AlwaysLockExclusive`，`None` 等多种文件锁策略，其中 `LockSharedDuringAppendingAndLockExclusiveDuringUploading` 是最推荐的策略，适用于绝大多数情况，而 `AlwaysLockExclusive` 则适用于文件追加无法保证原子性的文件系统上（通常是网络文件系统），而 `None` 则表示不使用文件锁（不推荐，可能造成日志丢失）。

#### 锁方法

| 名称 | 描述 |
| ------------- | ------------------ |
| lock_for_appending() | 为追加文件上锁 |
| try_lock_for_appending() | 为追加文件尝试上锁 |
| lock_for_uploading() | 为上传文件上锁 |
| try_lock_for_uploading() | 为上传文件尝试上锁 |
| unlock() | 解锁 |

### 支持接口

#### log()

记录日志

#### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| record | UploadLoggerRecord | 上传日志条目 |

#### 返回参数

无

#### 伪代码实现

```
fn append_record_to_log_buffer(record) {
	log_buffer_lock.rlock()
	log_buffer_size = log_buffer.size()
	log_buffer_lock.runlock()
	if log_buffer_size < max_size {
		data = "${record.to_string()}\n"
		data_len = data.len()
		if log_buffer_size + data_len < max_size {
			log_buffer_lock.lock()
			log_buffer.append(data)
			log_buffer_lock.unlock()
		}
	}
}

fn append_log_buffer_to_record_or_upload_log_buffer(ignore_upload_threshold) {
	log_file_lock.rlock()
	log_file_size = log_file.size()
	log_file_lock.runlock()
	if log_file_size < max_size {
		log_buffer_lock.rlock()
		log_buffer_size, err = log_buffer.size()
		log_buffer_lock.runlock()
		if err {
			return err
		}
		if log_buffer_size > 0 {
			if log_buffer_size + log_file_size < max_size {
				if log_file_lock.try_lock() { // 检测日志文件句柄是否能上写锁
					if lock_policy.try_lock_for_appending(log_file) // 检测日志文件是否能上排他锁 {
						log_buffer_lock.lock()
						log_buffer_content = log_buffer.clone() // 复制内存缓冲区的数据出来，然后将缓冲区置空
						log_buffer.clear()
						log_buffer_lock.unlock()
						err = log_file.write(log_buffer_content)
						lock_policy.unlock(log_file)
						if err {
							log_file_lock.unlock()
							return err
						}
						return nil
					}
					log_file_lock.unlock()
				}
			}
			// 前面并没有成功解锁日志文件，这里考虑日志缓冲区尺寸如果到达指定大小，绕过日志文件直接上传
			if ignore_upload_threshold || log_buffer_size > upload_threshold {
				log_buffer_lock.lock()
				log_buffer_content = log_buffer.clone() // 复制内存缓冲区的数据出来，然后将缓冲区置空
				log_buffer.clear()
				log_buffer_lock.unlock()
				if ignore_upload_threshold && log_buffer_content.len() > 0 || log_buffer_content.len() > upload_threshold {
					async_upload_log_buffer(log_buffer_content)
				}
			}
		}
	}
	null
}

fn async_lock_log_file_and_update_then_clean_if_needed() {
	log_file_lock.rlock()
	log_file_size, err = log_file.size()
	log_file_lock.runlock()
	if err {
		return err
	}
	if log_file_size > upload_threshold {
		async_lock_log_file_and_update_then_clean()
	}
	null
}

fn async_lock_log_file_and_update_then_clean() {
	global_thread_pool.spawn(() -> {
		lock_log_file_and_update_then_clean()
	})
}

fn lock_log_file_and_update_then_clean() {
	log_file_lock.rlock()
	log_file_size, err = log_file.size()
	log_file_lock.runlock()
	if err {
		return err
	}
	if log_file_size > upload_threshold {
		log_file_lock.lock()
		lock_policy.lock_for_uploading(log_file)
		err = upload_log_file_and_clean(log_file)
		lock_policy.unlock(log_file)
		log_file_lock.unlock()
		if err {
			return err
		}
	}
	null
}

fn upload_log_file_and_clean(log_file) {
	log_file_size, err = log_file.size()
	if err {
		return err
	}
	if log_file_size > upload_threshold {
		err = log_file.seek(0)
		if err {
			return err
		}
		content, err = log_file.read()
		if err {
			return err
		}
		err = upload_log_buffer(content)
		if err {
			return err
		}
		err = log_file.truncate()
		if err {
			return err
		}
		err = log_file.seek(0)
		if err {
			return err
		}
	}
	null
}

fn async_upload_log_buffer(log_buffer) {
	global_thread_pool.spawn(() -> {
		upload_log_buffer(log_buffer)
	})
}

fn upload_log_buffer(log_buffer) {
	_, err = http_client.send_request(POST, "/log/3", [server_url], {"Authorization": "UpToken ${upload_token}"}, {}, log_buffer, null, false, false, false, null, null, null, null)
	err
}

append_record_to_log_buffer(record)
err = append_log_buffer_to_record_or_upload_log_buffer(false)
if err {
	return err
}
err = async_lock_log_file_and_update_then_clean_if_needed()
if err {
	return err
}
null
```

#### 析构方法

由于部分上传日志存储在内存缓冲区中，可能会在析构时丢失，所以需要有一定的机制来处理这部分日志。

#### 伪代码实现

```
append_log_buffer_to_record_or_upload_log_buffer(true)
```
