# Uploader 上传器

上传器体系分为三层

| 名称          | 描述     |
| ------------- | -------- |
| UploadManager | 上传管理器，仅仅用做根据给定参数判定存储空间，然后创建 BucketUploader |
| BucketUploader | 存储空间上传器，用于锁定上传域名，然后创建 FileUploader |
| FileUploader | 文件上传器，调用 FormUploader 或 ResumeableUploader 进行实际上传 |

调用者可以从中间任何一层开始，逐层创建直到 FileUploader 为止。 

## UploadManager

| 名称       | 类型       | 描述                              |
| ---------- | ---------- | --------------------------------- |
| credential | Credential | 客户端存储的 AccessKey，SecretKey |
| config     | Config     | 客户端所用配置信息                |

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

##### 返回参数

| 名称    | 类型     | 描述                                                 |
| ------- | -------- | ---------------------------------------------------- |
| uploader | FileUploader | 文件上传器 |

## BucketUploader

| 名称       | 类型       | 描述                              |
| ---------- | ---------- | --------------------------------- |
| upload_manager | UploadManager | 上传管理器 |
| bucket_name     | String     | 存储空间名称               |
| up_urls_list     | [[String]]     | 上传域名列表                |
| client     | Client     | HTTP 客户端                |

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
| vars  | map<String, String> | 自定义变量 |
| metadata  | map<String, String> | 元信息 |
| checksum_enabled | bool | 是否启用校验，总是默认为启用 |
| resumeable_policy | ResumablePolicy | 恢复策略，默认为 `Threshold(config.upload_threshold)` |

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

#### upload_file()

上传文件系统中的文件

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
switch resumeable_policy {
case Threshold:
	if file_path.file_size() > size {
		ResumeableUploader.upload_file(file_path, file_name, mime, upload_token, key, vars, metadata, checksum_enabled)
	} else {
		FormUploader.upload_file(file_path, file_name, mime, upload_token, key, vars, metadata, checksum_enabled)
	}
case Always:
	ResumeableUploader.upload_file(file_path, file_name, mime, upload_token, key, vars, metadata, checksum_enabled)
case Never:
	FormUploader.upload_file(file_path, file_name, mime, upload_token, key, vars, metadata, checksum_enabled)
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
switch resumeable_policy {
case Threshold, Always:
	ResumableUploader.upload_stream(stream, file_name, mime, upload_token, key, vars, metadata, checksum_enabled)
case Never:
	FormUploader.upload_stream(stream, file_name, mime, upload_token, key, vars, metadata, checksum_enabled)
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
