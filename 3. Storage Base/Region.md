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

##### 伪代码实现

根据当前参数调用 `http(s)://uc.qbox.me/v3/query` 获得 `hosts` 列表。
对于每一个 `host` 的 `up` 和 `io` 解析方式如下：

```
hosts.map((host) -> {
	Region {
		up_http_urls: [host.up.acc, host.up.src].map((domains) -> {
			[domains.main, domains.backup]
				.filter((domains) -> { domains != null }) // 考虑 backup 可能不存在的情况
				.flatten() // 将嵌套数组平铺
				.map((domain) -> "http://${domain}")
		})
		.flatten(), // 将嵌套数组平铺
		up_https_urls: [host.up.acc, host.up.src, host.up.old_acc, host.up.old_src].map((domains) -> {
			[domains.main, domains.backup]
				.filter((domains) -> { domains != null }) // 考虑 backup 可能不存在的情况
				.flatten() // 将嵌套数组平铺
				.map((domain) -> "https://${domain}")
		}),
		.flatten(), // 将嵌套数组平铺
		io_http_urls: [domains.main, domains.backup]
				.filter((domains) -> { domains != null }) // 考虑 backup 可能不存在的情况
				.flatten() // 将嵌套数组平铺
				.map((domain) -> "http://${domain}"),
		io_https_urls: [domains.main, domains.backup]
				.filter((domains) -> { domains != null }) // 考虑 backup 可能不存在的情况
				.flatten() // 将嵌套数组平铺
				.map((domain) -> "https://${domain}"),
	}
})
```

至于 rs, rsf 和 api 的域名，可以通过根据 io 的域名推测即可得到。

#### 默认内建区域

##### 华东地区

| 字段            | 值                                                           |
| --------------- | ------------------------------------------------------------ |
| `up_http_urls`  | `["http://upload.qiniup.com", "http://up.qiniup.com", "http://upload.qbox.me", "http://up.qbox.me"]` |
| `up_https_urls` | `["https://upload.qiniup.com", "https://up.qiniup.com", "https://upload.qbox.me", "https://up.qbox.me"]` |
| `io_http_urls`  | `["http://iovip.qbox.me"]`                                   |
| `io_https_urls` | `["https://iovip.qbox.me"]`                                  |
| `rs_http_url`   | `http://rs.qiniu.com`                                        |
| `rs_https_url`  | `https://rs.qbox.me`                                         |
| `rsf_http_url`  | `http://rsf.qiniu.com`                                       |
| `rsf_https_url` | `https://rsf.qbox.me`                                        |
| `api_http_url`  | `http://api.qiniu.com`                                       |
| `api_https_url` | `https://api.qiniu.com`                                      |

##### 华北地区

| 字段            | 值                                                           |
| --------------- | ------------------------------------------------------------ |
| `up_http_urls`  | `["http://upload-z1.qiniup.com", "http://up-z1.qiniup.com", "http://upload-z1.qbox.me", "http://up-z1.qbox.me"]` |
| `up_https_urls` | `["https://upload-z1.qiniup.com", "https://up-z1.qiniup.com", "https://upload-z1.qbox.me", "https://up-z1.qbox.me"]` |
| `io_http_urls`  | `["http://iovip-z1.qbox.me"]`                                   |
| `io_https_urls` | `["https://iovip-z1.qbox.me"]`                                  |
| `rs_http_url`   | `http://rs-z1.qiniu.com`                                        |
| `rs_https_url`  | `https://rs-z1.qbox.me`                                         |
| `rsf_http_url`  | `http://rsf-z1.qiniu.com`                                       |
| `rsf_https_url` | `https://rsf-z1.qbox.me`                                        |
| `api_http_url`  | `http://api-z1.qiniu.com`                                       |
| `api_https_url` | `https://api-z1.qiniu.com`                                      |

##### 华南地区

| 字段            | 值                                                           |
| --------------- | ------------------------------------------------------------ |
| `up_http_urls`  | `["http://upload-z2.qiniup.com", "http://up-z2.qiniup.com", "http://upload-z2.qbox.me", "http://up-z2.qbox.me"]` |
| `up_https_urls` | `["https://upload-z2.qiniup.com", "https://up-z2.qiniup.com", "https://upload-z2.qbox.me", "https://up-z2.qbox.me"]` |
| `io_http_urls`  | `["http://iovip-z2.qbox.me"]`                                   |
| `io_https_urls` | `["https://iovip-z2.qbox.me"]`                                  |
| `rs_http_url`   | `http://rs-z2.qiniu.com`                                        |
| `rs_https_url`  | `https://rs-z2.qbox.me`                                         |
| `rsf_http_url`  | `http://rsf-z2.qiniu.com`                                       |
| `rsf_https_url` | `https://rsf-z2.qbox.me`                                        |
| `api_http_url`  | `http://api-z2.qiniu.com`                                       |
| `api_https_url` | `https://api-z2.qiniu.com`                                      |

##### 东南亚地区

| 字段            | 值                                                           |
| --------------- | ------------------------------------------------------------ |
| `up_http_urls`  | `["http://upload-as0.qiniup.com", "http://up-as0.qiniup.com", "http://upload-as0.qbox.me", "http://up-as0.qbox.me"]` |
| `up_https_urls` | `["https://upload-as0.qiniup.com", "https://up-as0.qiniup.com", "https://upload-as0.qbox.me", "https://up-as0.qbox.me"]` |
| `io_http_urls`  | `["http://iovip-as0.qbox.me"]`                                   |
| `io_https_urls` | `["https://iovip-as0.qbox.me"]`                                  |
| `rs_http_url`   | `http://rs-as0.qiniu.com`                                        |
| `rs_https_url`  | `https://rs-as0.qbox.me`                                         |
| `rsf_http_url`  | `http://rsf-as0.qiniu.com`                                       |
| `rsf_https_url` | `https://rsf-as0.qbox.me`                                        |
| `api_http_url`  | `http://api-as0.qiniu.com`                                       |
| `api_https_url` | `https://api-as0.qiniu.com`                                      |

##### 北美地区

| 字段            | 值                                                           |
| --------------- | ------------------------------------------------------------ |
| `up_http_urls`  | `["http://upload-na0.qiniup.com", "http://up-na0.qiniup.com", "http://upload-na0.qbox.me", "http://up-na0.qbox.me"]` |
| `up_https_urls` | `["https://upload-na0.qiniup.com", "https://up-na0.qiniup.com", "https://upload-na0.qbox.me", "https://up-na0.qbox.me"]` |
| `io_http_urls`  | `["http://iovip-na0.qbox.me"]`                                   |
| `io_https_urls` | `["https://iovip-na0.qbox.me"]`                                  |
| `rs_http_url`   | `http://rs-na0.qiniu.com`                                        |
| `rs_https_url`  | `https://rs-na0.qbox.me`                                         |
| `rsf_http_url`  | `http://rsf-na0.qiniu.com`                                       |
| `rsf_https_url` | `https://rsf-na0.qbox.me`                                        |
| `api_http_url`  | `http://api-na0.qiniu.com`                                       |
| `api_https_url` | `https://api-na0.qiniu.com`                                      |

#### 获取内建区域方法

| 名称        | 描述         | 返回类型 |
| ----------- | ------------ | -------- |
| z0          | 华东地区     | Region   |
| z1          | 华北地区     | Region   |
| z2          | 华南地区     | Region   |
| as0         | 东南亚地区       | Region   |
| na0         | 北美地区       | Region   |
| all_regions | 全部内建地区 | [Region] |

