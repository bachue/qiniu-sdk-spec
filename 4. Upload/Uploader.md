# Uploader 上传管理器

上传体系分为三层

| 名称          | 描述     |
| ------------- | -------- |
| UploadManager | 上传管理器，仅仅用做根据给定参数判定存储空间，然后创建 BucketUploader。可以复用。 |
| BucketUploader | 存储空间上传器，用于锁定上传域名，然后创建 FileUploader。可以复用。 |
| FileUploader | 文件上传器，调用 FormUploader 或 ResumeableUploader 进行实际上传。不可复用。 |

调用者可以从中间任何一层开始，逐层创建直到 FileUploader 为止。

考虑到用户可能没有存储空间的 Secret Key，因此整个上传体系均不要求用户必须传入 Credential，用户可以提供 Credential，也可以直接给出上传凭证。

## UploadManager

| 名称          | 类型         | 描述                                                         |
| ------------- | ------------ | ------------------------------------------------------------ |
| config        | Config       | 客户端所用配置信息                                           |
| upload_logger | UploadLogger | 上传日志，如果 `config.uplog_disabled` 为 true，则该属性为 `null` |

### 支持接口

#### for_bucket()

根据 Bucket 创建 BucketUploader

##### 接受参数

| 名称       | 类型       | 描述       |
| ---------- | ---------- | ---------- |
| bucket     | Bucket     | 存储空间   |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| uploader | BucketUploader | 存储空间上传器 |

#### for_bucket_name()

根据 Bucket 名称创建 BucketUploader

##### 接受参数

| 名称       | 类型       | 描述       |
| ---------- | ---------- | ---------- |
| bucket_name     | String     | 空间名称   |
| access_key | String | 七牛 Access Key |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| uploader | BucketUploader | 存储空间上传器 |

#### for_upload_token()

根据 UploadToken 创建 FileUploader（将从上传凭证中确定要上传的空间）

##### 接受参数

| 名称       | 类型       | 描述       |
| ---------- | ---------- | ---------- |
| upload_token     | UploadToken     | 上传凭证   |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| uploader | FileUploader | 文件上传器 |

#### for_upload_policy()

根据 UploadPolicy 创建 FileUploader（将从上传策略中确定要上传的空间）

##### 接受参数

| 名称       | 类型       | 描述       |
| ---------- | ---------- | ---------- |
| upload_policy     | UploadPolicy     | 上传策略   |
| credential | Credential | 七牛认证信息 |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| uploader | FileUploader | 文件上传器 |

## BucketUploader

| 名称       | 类型       | 描述                              |
| ---------- | ---------- | --------------------------------- |
| bucket_name     | String     | 存储空间名称               |
| up_urls_list     | [[String]]     | 上传域名列表                |
| client     | Client     | HTTP 客户端                |
| upload_logger | UploadLogger | 上传日志，尽可能引用 UploadManager 中的 upload_logger |
| recorder | UploadRecorder | 上传日志记录仪 |
| thread_pool | ThreadPool | 线程池，可以为空 |

### 支持接口

#### for_upload_token()

根据 UploadToken 创建 FileUploader

##### 接受参数

| 名称       | 类型       | 描述       |
| ---------- | ---------- | ---------- |
| upload_token     | UploadToken     | 上传凭证   |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| uploader | FileUploader | 文件上传器 |

#### for_upload_policy()

根据 UploadPolicy 创建 FileUploader

##### 接受参数

| 名称       | 类型       | 描述       |
| ---------- | ---------- | ---------- |
| upload_policy     | UploadPolicy     | 上传策略 |
| credential | Credential | 七牛认证信息 |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| uploader | FileUploader | 文件上传器 |

## FileUploader

| 名称       | 类型       | 描述                              |
| ---------- | ---------- | --------------------------------- |
| bucket_uploader | BucketUploader | 上传管理器 |
| upload_token     | UploadToken     | 上传凭证 |
| key   | String | 对象名称                |
| vars  | [String:String] | 自定义变量 |
| metadata  | [String:String] | 元信息 |
| checksum_enabled | bool | 是否启用校验，总是默认为启用 |
| resumeable_policy | ResumablePolicy | 恢复策略，默认为 `Threshold(config.upload_threshold)` |
| uploading_progress_callback | fn(uint, uint) | 上传进度回调函数，第一个参数为请求已经上传的数据尺寸，第二个参数为请求总共要上传的数据尺寸 |
| thread_pool | ThreadPool | 专用线程池，可以为空 |

### 支持接口

#### setter() 方法

| 名称 | 描述 | 接受参数 | 伪代码实现 |
| ---- | ---- | -------- | -------- |
| set_key() | 设置对象名称 | key | `uploader.key = key`           |
| set_vars() | 设置自定义变量 | vars | `uploader.vars = vars`           |
| set_metadata() | 设置元数据 | metadata | `uploader.metadata = metadata`           |
| enable_checksum() | 启用校验和 | 无 | `uploader.checksum_enabled = true`           |
| disable_checksum() | 启用校验和 | 无 | `uploader.checksum_enabled = false`           |
| set_upload_threshold() | 设置上传分片阙值 | size | `uploader.resumeable_policy = Threshold(size)`           |
| always_be_resumeable() | 总是使用可恢复上传 | 无 | `uploader.resumeable_policy = Always` |
| never_be_resumeable() | 总是使用表单上传 | 无 | `uploader.resumeable_policy = Never` |
| on_uploading_progress() | 设置上传进度回调函数 | uploading_progress_callback | `uploader.uploading_progress_callback = uploading_progress_callback` |
| thread_pool | 设置专用线程池，注意由于上传不一定总会用到线程池，仅在确认会用到线程池的地方才使用 | thread_pool | `uploader.thread_pool = thread_pool` |

#### upload_file()

上传文件系统中的文件

如果条件允许，应该从上传日志记录仪中搜索恢复上传的可能。

##### 接受参数

| 名称       | 类型       | 描述       |
| ---------- | ---------- | ---------- |
| file_path | Path  | 要上传的文件路径 |
| file_name | String | 文件名，可选 |
| mime | String | MIME 类型，可选 |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| result | UploadResult | 上传结果 |

##### 伪代码

```
fn upload_file_by_form_uploader() {
	FormUploader.upload_file(file_path, file_name, mime, upload_token, key, vars, metadata, checksum_enabled, uploading_progress_callback)
}

fn upload_file_by_resumeable_uploader() {
	ResumeableUploader.upload_file(file_path, file_name, mime, upload_token, key, vars, metadata, checksum_enabled, uploading_progress_callback, thread_pool, bucket_uploader.recorder)
}

switch resumeable_policy {
case Threshold:
	if file_path.file_size() > size {
		upload_file_by_resumeable_uploader()
	} else {
		upload_file_by_form_uploader()
	}
case Always:
	upload_file_by_resumeable_uploader()
case Never:
	upload_file_by_form_uploader()
}
```

#### upload_stream()

上传输入流数据

由于输入流可能无法移动文件指针，因此功能相比于 `upload_file()` 更弱，在有些情况下可能无法支持校验和多区域重试。

##### 接受参数

| 名称       | 类型       | 描述       |
| ---------- | ---------- | ---------- |
| stream | Reader  | 输入流 |
| file_name | String | 文件名，可选 |
| mime | String | MIME 类型，可选 |

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| result | UploadResult | 上传结果 |

##### 伪代码

```
fn upload_stream_by_resumeable_uploader() {
	ResumableUploader.upload_stream(stream, file_name, mime, upload_token, key, vars, metadata, checksum_enabled, uploading_progress_callback, thread_pool)
}

fn upload_stream_by_form_uploader() {
	FormUploader.upload_stream(stream, file_name, mime, upload_token, key, vars, metadata, checksum_enabled, uploading_progress_callback)
}

switch resumeable_policy {
case Threshold, Always:
	upload_by_resumeable_uploader()
case Never:
	upload_stream_by_form_uploader()
}
```

## ResumablePolicy

恢复策略凭证是一个枚举类，包含三种值

| 名称       | 描述       |
| ---------- | ---------- |
|Threshold(size)|文件尺寸大于该尺寸则使用可恢复上传，否则使用表单上传|
|Always|总是使用可恢复上传|
|Never|总是使用表单上传|

## UploadResult

上传结果，由于可以通过上传策略改变响应体内容，因此该类设计与 JSON 库的功能密切相关

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| object | JSONObject | JSON 对象 |

### 支持接口

#### getter() 方法

| 名称 | 描述 | 类型 | 伪代码实现 |
| ---- | ---- | -------- | -------- |
| hash() | 获取 Etag，可能会返回 null | String | `object.get("hash")`           |
| key() | 获取 Object 名称，可能会返回 null | String | `object.get("key")` |
