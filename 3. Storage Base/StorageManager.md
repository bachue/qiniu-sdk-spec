# StorageManager 存储管理器

| 名称          | 类型         | 描述                                                         |
| ------------- | ------------ | ------------------------------------------------------------ |
| credential    | Credential   | 客户端存储的 AccessKey，SecretKey                            |
| http_client        | Client       | HTTP 客户端                                           |
| rs_url | String | RS 服务器 URL |

## 支持接口

### bucket_names()

列出存储空间名称

##### 接受参数

无

##### 返回参数

| 名称     | 类型           | 描述           |
| -------- | -------------- | -------------- |
| names | [String] | 存储空间上传器 |
| error | HTTPError | HTTP 错误消息 |

##### 伪代码实现

根据当前参数调用 `GET ${rs_url}/buckets` 获得存储空间名称列表。API 鉴权方式为 Qiniu。

### create_bucket()

创建存储空间

##### 接受参数

| 名称     | 类型           | 描述           |
| -------- | -------------- | -------------- |
| bucket | String | 存储空间名称 |
| region_id | String | 区域 ID |

##### 返回参数

| 名称     | 类型           | 描述           |
| -------- | -------------- | -------------- |
| error | HTTPError | HTTP 错误消息 |

##### 伪代码实现

根据当前参数调用 `POST ${rs_url}/mkbucketv3/${bucket}/region/{region_id}` 创建存储空间。API 鉴权方式为 Qiniu。

### drop_bucket()

删除存储空间。必须由用户确保该 Bucket 已经没有文件才允许删除。

##### 接受参数

| 名称     | 类型           | 描述           |
| -------- | -------------- | -------------- |
| bucket | String | 存储空间名称 |

##### 返回参数

| 名称     | 类型           | 描述           |
| -------- | -------------- | -------------- |
| error | HTTPError | HTTP 错误消息 |

##### 伪代码实现

根据当前参数调用 `POST ${rs_url}/drop/${bucket}` 删除存储空间。API 鉴权方式为 Qiniu。

### upload_manager()

获取上传管理器

##### 接受参数

无

##### 返回参数

| 名称     | 类型           | 描述           |
| -------- | -------------- | -------------- |
| upload_manager | UploadManager | 上传管理器 |

### bucket()

##### 接受参数

| 名称     | 类型           | 描述           |
| -------- | -------------- | -------------- |
| bucket | String | 存储空间名称 |

##### 返回参数

| 名称     | 类型           | 描述           |
| -------- | -------------- | -------------- |
| bucket | Bucket | 存储空间 |