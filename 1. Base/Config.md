# Config 七牛客户端配置

| 名称                     | 类型                                                        | 描述                                                         |
| ------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------ |
| appended_user_agent                | String                                                   | 追加 UserAgent，默认为空             |
| use_https                | bool                                                        | 是否使用 HTTPS 协议，默认为 false                            |
| uc_host | String | UC 主机地址，默认为 `"uc.qbox.me"` |
| rs_host | String | RS 主机地址，默认为 `"rs.qbox.me"` |
| rsf_host | String | RSF 主机地址，默认为 `"rsf.qbox.me"` |
| api_host | String | API 主机地址，默认为 `"api.qiniu.com"` |
| uplog_host | String | UpLog 主机地址，默认为 `"uplog.qbox.me"` |
| upload_token_lifetime    | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒） | 上传凭证默认有效期，默认为 3600 秒                           |
| batch_max_operation_size | uint                                                        | 最大批量操作数，默认为 1000                                  |
| upload_threshold         | uint32                                                    | 如果上传文件尺寸大于该值，将自动使用分片上传，否则，使用表单上传。单位为字节，默认为 4 MB |
| upload_block_size        | uint32                                                      | 上传分块尺寸，尺寸越小越适合弱网环境，必须是 4 MB 的倍数。单位为字节，默认为 4 MB |
| upload_logger | UploadLogger | 上传日志，默认存在，但可能为空 |
| upload_recorder | UploadRecorder | 上传日志记录仪，默认参数配置 |
| http_request_retries     | uint                                                        | 对于一个 URL 的重试次数，默认为 3 次                         |
| http_request_retry_delay | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒） | 在每次重试 URL 前的等待时间，默认在 500 毫秒到 1 秒之间      |
| domains_manager          | DomainsManager                                              | 全局域名管理器，默认参数配置                                        |

