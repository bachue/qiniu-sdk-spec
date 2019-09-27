# DomainsManager 域名管理

用于冻结域名。

| 名称       | 类型       | 描述                                               |
| ---------- | ---------- | -------------------------------------------------- |
| frozen_domains | map<String, Time>（对于存在多线程的情况，该类型必须保证线程安全。此外，存储的时间必须是单调时间）| 存储被冻结的域名及解冻时间 |

## 支持接口

### is_frozen()

#### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| domain | String | 域名，可以带协议 |

#### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| frozen | bool | 是否已经被冻结 |

#### 伪代码实现

```
normalize_domain(domain) // 从给定的地址中移除协议或端口信息，仅保留域名本身
unfreeze_time = frozen_domains.get(domain)
if unfreeze_time {
	if unfreeze_time < now() {
		frozen_domains.delete(domain)
		return false
	}
	return true
}
false
```

### freeze_domain()

#### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| domain | String | 域名，可以带协议 |
| duration | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒）| 冻结时长 |

#### 返回参数

无

#### 伪代码实现

```
normalize_domain(domain) // 从给定的地址中移除协议或端口信息，仅保留域名本身
frozen_domains.set(domain, now() + duration)
```