# Bucket 存储空间

| 名称          | 类型                          | 描述         |
| ------------- | ----------------------------- | ------------ |
| name  | String                   | 空间名称     |
| credential | Credential | 认证信息 |
| upload_manager | UploadManager | 上传管理器 |
| region | Region | 空间所在区域，允许用户设置，如果没有设置则懒加载 |
| backup_regions | [Region] | 空间备用区域，允许用户设置，如果没有设置则懒加载 |
| domains | [String] | 下载域名列表，允许用户设置，如果没有设置则懒加载 |
| http_client | Client | HTTP 客户端 |

