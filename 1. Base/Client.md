# Client 客户端

| 名称       | 类型       | 描述                              |
| ---------- | ---------- | --------------------------------- |
| credential | Credential | 客户端存储的 AccessKey，SecretKey |
| config     | Config     | 客户端所用配置信息                |

## 支持接口

### 构造方法

初始化 SDK 客户端

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| access_key | String | 七牛 Access Key |
| secret_key | String | 七牛 Secret Key |
| config | Config | 七牛客户端配置 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| client | Client | SDK 客户端 |

### storage_manager()

获取存储管理器

#### 接受参数

无

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| storage_manager | StorageManager | 存储管理器 |

### upload_manager()

获取上传管理器

#### 接受参数

无

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| uploader_manager | [UploadManager](../4. Upload/Uploader.md) | 上传管理器 |

