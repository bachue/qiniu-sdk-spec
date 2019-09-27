# DomainsManager 域名管理

用于冻结域名。

| 名称       | 类型       | 描述                                               |
| ---------- | ---------- | -------------------------------------------------- |
| frozen_domains | map<String, Time>（对于存在多线程的情况，该类型必须保证线程安全。此外，存储的时间必须是单调时间）| 存储被冻结的域名及解冻时间 |

## 支持接口

### choose()

#### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| domains | [String] | 域名，可以带协议 |

#### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| domains | [String] | 是否已经被冻结 |

#### 伪代码实现

```
fn is_frozen(domain) {
  domain = normalize_domain(domain) // 从给定的地址中移除协议或端口信息，仅保留域名本身
  unfreeze_time = frozen_domains.get(domain)
  if unfreeze_time {
    if unfreeze_time < now() {
      frozen_domains.delete(domain)
      return false
    }
    return true
  }
  false
}

chosen = []
for domain in domains {
	if !is_frozen(domain) {
		chosen.append(domain)
	}
}
if domains.len() > 0 && chosen.len() == 0 { // 如果所有域名都已经被冻结，则只选择解冻时间最早的域名尝试
	sorted = domains.sort_by((domain) => frozen_domains.get(normalize_domain(domain)))
	chosen = [sorted.get(0)]
} else {
  chosen
}
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
frozen_domains.set(normalize_domain(domain), now() + duration)
```