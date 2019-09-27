# Region 区域

## Region

存储区域 

| 名称          | 类型                          | 描述         |
| ------------- | ----------------------------- | ------------ |
| id  | String                   | 区域 ID     |
| up_http_urls | [String]                          | HTTP 协议的上传地址 |
| up_https_urls | [String]                          | HTTPS 协议的上传地址 |
| io_http_urls | [String]                          | HTTP 协议的下载地址 |
| io_https_urls | [String]                          | HTTPS 协议的下载地址 |
| rs_http_url | String | HTTP 协议的 RS 地址 |
| rs_https_url | String | HTTPS 协议的 RS 地址 |
| rsf_http_url | String | HTTP 协议的 RSF 地址 |
| rsf_https_url | String | HTTPS 协议的 RSF 地址 |
| api_http_url | String | HTTP 协议的 API 地址 |
| api_https_url | String | HTTPS 协议的 API 地址 |

## RegionUtils

辅助方法，查询 Region 相关信息 ，也可以以 Region 的类方法的形态实现

### 支持接口

#### query()

获取指定 bucket 的区域列表

##### 接受参数

| 名称       | 类型       | 描述                              |
| ---------- | ---------- | --------------------------------- |
| bucket | String | 空间名称 |
| credential | Credential | 认证信息 |
| config | Config | 客户端配置 |

##### 返回参数

| 名称       | 类型       | 描述                              |
| ---------- | ---------- | --------------------------------- |
| regions | [Region] | 区域列表，第一个区域为当前区域，其他为备选区域 |
