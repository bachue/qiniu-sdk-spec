# UploadPolicy 上传策略

数据结构参照 https://developer.qiniu.com/kodo/manual/1206/put-policy 实现，内部实现不作约束，但应该提供更易于理解的 Initializer，Setter 和 Getter 方法。(根据各编程语言的习惯，可以做简单调整，比如直接使用 named arguments 或者 Builder，没有必要拘泥于某一种方式)

## 支持接口

### new_policy_for_bucket()

获取基于指定 Bucket 的上传策略

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| bucket | String | 空间名称 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| upload_policy | UploadPolicy | 上传策略 |

#### 伪代码实现

```
UploadPolicy { scope: bucket }
```
### new_policy_for_bucket()

获取基于指定 Bucket 的上传策略

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| bucket | String | 空间名称 |
| config | Config | 配置 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| upload_policy | UploadPolicy | 上传策略 |

#### 伪代码实现

```
policy = UploadPolicy { scope: bucket }
policy.set_token_lifetime(config.upload_token_lifetime)
policy
```

### new_policy_for_object()

获取基于指定 Object 的上传策略

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| bucket | String | 空间名称 |
| key | String | 对象名称 |

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| upload_policy | UploadPolicy | 上传策略 |

#### 伪代码实现

```
policy = UploadPolicy { scope: "${bucket}:${key}" }
policy.set_token_lifetime(config.upload_token_lifetime)
policy
```

### new_policy_for_objects_with_prefix()

获取基于指定 Object key 前缀的上传策略

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| bucket | String | 空间名称 |
| prefix | String | 对象名称前缀 |

#### 返回参数

UploadPolicy

#### 伪代码实现

```
policy = UploadPolicy { scope: "${bucket}:${key}", is_prefixal_scope: 1 }
policy.set_token_lifetime(config.upload_token_lifetime)
policy
```

### setter() 方法

| 名称 | 描述 | 接受参数 | 伪代码实现 |
| ---- | ---- | -------- | -------- |
| set_insert_only() | 设置 insert_only 属性 | 无 | `policy.insert_only = 1`           |
| set_overwriteable() | 移除 insert_only 属性 | 无 | `policy.insert_only = 0` |
| enable_mime_detection() | 设置 detect_mime 属性 | 无 | `policy.detect_mime = 1 ` |
| disable_mime_detection() | 移除 insert_only 属性 | 无 | `policy.detect_mim = 0` |
| set_token_lifetime() | 设置 deadline 属性 | lifetime | `policy.set_token_deadline(now() + lifetime)` |
| set_token_deadline() | 设置 deadline 属性 | deadline | `policy.deadline = deadline.unix() as uint32` |
| enable_infrequent_storage() | 设置 file_type | 无 | `policy.file_type = 1` |
| enable_normal_storage() | 设置 file_type | 无 | `policy.file_type = 0` |
| set_return_url() | 设置 return_url | return_url | `policy.return_url = return_url` |
| set_return_body() | 设置 return_body | return_body | `policy.return_url = return_body` |
| set_callback() | 设置 callback 相关属性 | urls, host, body, type | `policy.callback_url = urls.join(';'); policy.callback_host = host; policy.callback_body = body; policy.callback_body_type = type` |
| set_persistent_ops() | 设置 persistent_ops 相关属性 | ops, notify_url, pipeline | `policy.persistent_ops = ops.join(';'); policy.persistent_notify_url = notify_url; policy.persistent_pipeline = pipeline` |
| save_as() | 设置 save_key 相关属性 | save_key, force | `policy.save_key = save_key; policy.force_save_key = force` |
| set_file_size_range() | 设置 fsize | min, max | `policy.fsize_min = min; policy.fsize_limit = max` |
| limit_mime() | 设置 MIME 列表 | mime | `policy.mime_limit = mime.join(';')` |
| set_object_lifetime() | 设置对象有效期 | lifetime | `policy.delete_after_days = (lifetime + 86400 -1) / 86400` |
| set_object_deadline() | 设置对象过期时间 | deadline | `policy.set_object_lifetime(deadline - now())` |

### getter() 方法

| 名称 | 描述 | 类型 | 伪代码实现 |
| ---- | ---- | -------- | -------- |
| bucket() | Bucket 名称 | String | `policy.scope.split(':', 2).get(0)`           |
| key() | Object 名称 | String | `policy.scope.split(':', 2).get(1)` |
| prefixal() | 是否匹配 Object 名称前缀 | bool | `policy.is_prefixal_scope == 1` |
| insert_only() | 是否不可覆盖 | bool | `policy.insert_only == 1` |
| mime_detection() | 是否自动检测 MIME | bool | `policy.detect_ime == 1` |
| deadline() | 过期时间 | Time | `unix_time(policy.deadline)` |
| lifetime() | 生命周期 | Duration | `deadline() - now()` |
| return_url() | 重定向地址 | String | `policy.return_url` |
| return_body() | 返回内容 | String | `policy.return_body` |
| callback_urls() | 回调地址 | [String] | `policy.callback_url.split(';')` |
| callback_host() | 回调 HOST | String | `policy.callback_host` |
| callback_body() | 回调请求体 | String | `policy.callback_body` |
| callback_body_type() | 回调请求体类型 | String | `policy.callback_body_type` |
| persistent_ops() | 持久化操作指令 | [String] | `policy.persistent_ops.split(';')` |
| persistent_notify_url() | 持久化操作回调 URL | String | `policy.persistent_notify_url` |
| persistent_pipeline() | 持久化操作管道名称 | String | `policy.persistent_pipeline` |
| save_key() | 自定义 Object 名称 | String | `policy.save_key` |
| force_save_key() | 强制使用自定义 Object 名称 | String | `policy.force_save_key` |
| file_size_range() | 文件尺寸范围 | (uint, uint) | `(policy.fsize_min, policy.fsize_limit)` |
| mime_limit() | MIME 类型限制 | [String] | `policy.mime_limit.split(';')` |
| infrequent_storage() | 低频存储 | bool | `policy.file_type == 1` |
| object_lifetime() | 对象生命周期 | Duration | `policy.delete_after_days * 86400` |
| object_deadline() | 对象过期时间 | Time | `now() + object_lifetime()` |

