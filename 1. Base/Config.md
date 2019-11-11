# Config 七牛客户端配置

| 名称                     | 类型                                                        | 描述                                                         |
| ------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------ |
| use_https                | bool                                                        | 是否使用 HTTPS 协议，默认为 false                            |
| upload_token_lifetime    | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒） | 上传凭证默认有效期，默认为 3600 秒                           |
| batch_max_operation_size | uint                                                        | 最大批量操作数，默认为 1000                                  |
| upload_threshold         | uint64                                                      | 如果上传文件尺寸大于该值，将自动使用分片上传，否则，使用表单上传。单位为字节，默认为 4 MB |
| upload_block_size        | uint                                                        | 上传分块尺寸，尺寸越小越适合弱网环境，必须是 4 MB 的倍数。单位为字节，默认为 4 MB |
| upload_block_lifetime    | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒） | 上传分块尺寸，尺寸越小越适合弱网环境，必须是 4 MB 的倍数。单位为字节，默认为 4 MB |
| always_flush_records    | bool | 是否在每次写入上传记录后刷新磁盘，默认不刷新 |
| recorder | Recorder | 记录仪，该类型是一个接口，默认为文件系统实现，文件存储在 $TMPDIR/qiniu_sdk/records |
| recorder_key_generator | fn(String, Path, String) -> String | 回调函数用于生成记录仪的键，传入记录仪名称，文件路径和对象名称，返回记录仪的键，默认为参数连接后经过 SHA1编码后的值 |
| uplog_server_url | String | 上传日志服务器 URL |
| uplog_disabled    | bool | 是否禁止记录上传日志，默认为不禁止 |
| uplog_upload_threshold    | uint64 | 如果上传日志大于该值，则开始异步上传日志文件。单位为字节，默认为 4 KB |
| max_uplog_size    | uint64 | 上传日志最大尺寸，如果上传日志大于该值，则不再记录新的日志项进去。单位为字节，默认为 4 MB |
| http_request_retries     | uint                                                        | 对于一个 URL 的重试次数，默认为 3 次                         |
| http_request_retry_delay | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒） | 在每次重试 URL 前的等待时间，默认在 500 毫秒到 1 秒之间      |
| domains_manager          | DomainsManager                                              | 全局域名管理器                                               |

