# Config 客户端配置

| 名称                     | 类型                                                        | 描述                                                         |
| ------------------------ | ----------------------------------------------------------- | ------------------------------------------------------------ |
| use_https                | bool                                                        | 是否使用 HTTPS 协议，默认为 false                            |
| upload_token_lifetime    | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒） | 上传凭证默认有效期，默认为 3600 秒                           |
| batch_max_operation_size | uint                                                        | 最大批量操作数，默认为 1000                                  |
| upload_threshold         | uint64                                                      | 如果上传文件尺寸大于该值，将自动使用分片上传，否则，使用表单上传。单位为字节，默认为 4 MB |
| upload_block_size        | uint                                                        | 上传分块尺寸，尺寸越小越适合弱网环境，必须是 4 MB 的倍数。单位为字节，默认为 4 MB |
| http_request_retries     | uint                                                        | 对于一个 URL 的重试次数，默认为 3 次                         |
| http_request_retry_delay | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒） | 在每次重试 URL 前的等待时间，默认为 500 毫秒或 1 秒          |
| domains_manager          | DomainsManager                                              | 全局域名管理器                                               |
| domain_freeze_duration   | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒） | 对 URL 重试多次失败后，冻结 URL 时间长度，默认为 10 分钟     |
| download_url_lifetime    | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒） | 下载地址默认有效期，默认为 3600 秒                           |
| file_recorder_path       | Path                                                        | 分片上传时，用于保存上传信息的目录路径                       |
