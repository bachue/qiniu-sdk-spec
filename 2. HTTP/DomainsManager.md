# DomainsManager 域名管理

用于冻结域名和为域名缓存 DNS 解析结果，支持持久化和异步自动持久化及刷新 DNS 缓存。

| 名称       | 类型       | 描述                                               |
| ---------- | ---------- | -------------------------------------------------- |
| inner | DomainsManagerInnerData | 内部数据 |
| persistent_file_path | Path，该值可不填 | 自动持久化存储时的文件路径 |
| last_persistent_time | Time | 最近一次持久化时间，必须是单调时间。如果是在多线程环境下，为避免多个并发持久化请求，应该对持久化时间的读写加锁。 |
| last_refresh_time | Time | 最近一次刷新解析结果缓存的时间，必须是单调时间。如果是在多线程环境下，为避免多个并发刷新请求，应该对刷新时间的读写加锁。 |

## 内部依赖结构体

### CachedResolutions

缓存的 URL 解析结果

| 名称       | 类型       | 描述                                               |
| ---------- | ---------- | -------------------------------------------------- |
| socket_addrs | [SocketAddr]| 缓存的 IP 地址，必须同时支持 IPv4 和 IPv6 |
| cache_deadline | Time | 缓存过期时间，必须是单调时间 |

### DomainsManagerInnerData

域名管理器内部核心的数据，可以用于编码后持久化到文件系统或从文件系统加载后解码。

| 名称       | 类型       | 描述                                               |
| ---------- | ---------- | -------------------------------------------------- |
| frozen_urls | [String:Time]（对于存在多线程的情况，该类型必须保证线程安全。此外，存储的时间必须是单调时间）| 存储被冻结的基础 URL 及解冻时间 |
| resolutions | [String:CachedResolutions] | 存储基础 URL 及其解析结果和缓存过期时间 |
|url_frozen_duration|Duration（如果没有 Duration 类型，则使用 uint64，单位为秒）|URL 冻结时长，默认为 10 分钟|
|resolutions_cache_lifetime|Duration（如果没有 Duration 类型，则使用 uint64，单位为秒）|URL 解析结果缓存时长，默认为 1 小时|
|disable_url_resolution|bool|是否禁止 URL 解析功能，默认为 false|
|url_resolve_retries|uint|对于一个 URL 的解析重试次数，默认为 10 次|
|url_resolve_retry_delay|Duration（如果没有 Duration 类型，则使用 uint64，单位为秒）|在每次重试 URL 前的等待时间，默认在 500 毫秒到 1秒之间|
|refresh_resolutions_interval|Duration（如果没有 Duration 类型，则使用 uint64，单位为秒），该值可不填|自动刷新 URL 解析结果间隔时间，默认为 30 分钟。如果不填则关闭自动刷新。|
|persistent_interval|Duration（如果没有 Duration 类型，则使用 uint64，单位为秒），该值可不填|自动持久化间隔，默认为 30 分钟。如果不填则关闭自动持久化。|
|refresh_resolutions_interval|Duration（如果没有 Duration 类型，则使用 uint64，单位为秒），该值可不填|自动刷新 URL 解析结果间隔时间，默认为 30 分钟。如果不填则关闭自动刷新。|

## 支持接口

### 构造方法

DomainsManager 的构造方法既支持从持久化的文件中读取，也支持创建一个全新的实例。并且允许用户通过传入参数，使构造方法可以同步或异步地预取常用的七牛公有云域名，或刷新持久化文件中记录的域名的解析结果。

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
fn auto_persistent() {
	if inner.persistent_interval && inner.last_persistent_time + inner.persistent_interval > now() { // 在多线程情况下，inner.last_persistent_time 的所有读写都应该上锁
		persistent() // 参考下文的伪代码实现
	}
}

fn auto_refresh() {
	if inner.refresh_resolutions_interval {
		if inner.last_refresh_time > now() + inner.refresh_resolutions_interval { // 在多线程情况下，inner.last_refresh_time 的所有读写都应该上锁
			async_refresh() // 参考下文的伪代码实现
		}
	}
}

fn is_frozen(base_url) {
  unfreeze_time = inner.frozen_urls.get(base_url)
  if unfreeze_time {
    if unfreeze_time < now() {
      inner.frozen_urls.delete(base_url)
      return false
    }
    return true
  }
  false
}

fn lock_and_resolve_and_update_cache(base_url) {
	resolution = inner.resolutions.lock_and_get(base_url) // 在多线程环境下，修改缓存必须锁住可能修改的映射项，然后重新检查数据。否则可能发生多个相同的 URL 并发解析时，缓存完全无法起作用的情况。
	results = []
	if resolution {
		if resolution.cache_deadline < now() {
			socket_addrs, err = base_url.to_socket_addrs()
			if !err {
				inner.resolutions.set(base_url, CachedResolutions { socket_addrs: socket_addrs, cache_deadline: now() + inner.resolutions_cache_lifetime })
				results = socket_addrs
			}
		} else {
			results = resolution.socket_addrs
		}
	} else {
			socket_addrs, err = base_url.to_socket_addrs()
			if !err {
				inner.resolutions.set(base_url, CachedResolutions { socket_addrs: socket_addrs, cache_deadline: now() + inner.resolutions_cache_lifetime })
				results = socket_addrs
			}
	}
	resolution.unlock() // 离开前解锁
	results
}

fn resolve(base_url) {
	resolution = inner.resolutions.get(base_url)
	if resolution {
		if resolution.cache_deadline < now() {
			lock_and_resolve_and_update_cache(base_url)
		} else {
			return resolution.socket_addrs
		}
	} else {
		lock_and_resolve_and_update_cache(base_url)
	}
}

fn make_choice(base_url) {
	if inner.disable_url_resolution {
		return Choice { base_url: base_url, socket_addrs: [] }
	}
	results = resolve(base_url)
	shuffle(results) // 防止总是连接同一个 IP 地址，每次获取结果都会打乱顺序
	Choice { base_url: base_url, socket_addrs: results }
}

choices = []
for base_url in base_urls {
	if !is_frozen(base_url) {
		choices = make_choice(base_url)
		if choice {
			choices.append(choice)
		}
	}
}
if base_urls.len() > 0 && chosen.len() == 0 { // 如果所有域名都已经被冻结，则只选择解冻时间最早的域名尝试
	sorted = base_urls.sort_by((base_url) => inner.frozen_urls.get(base_url))
	chosen = [make_choice(sorted.get(0))]
}

auto_persistent()
auto_refresh()

chosen
```

### freeze_url()

#### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| base_url | String | 基础 URL（含协议，不含路径）|

#### 返回参数

无

#### 伪代码实现

```
inner.frozen_urls.set(base_url, now() + duration)
auto_persistent() // 参考上文的伪代码实现
```

### unfreeze_urls()

#### 接受参数

无

#### 返回参数

无

#### 伪代码实现

```
inner.frozen_urls.clear()
auto_persistent() // 参考上文的伪代码实现
```

### persistent()

将 DomainManager 的数据持久化到文件系统中

#### 接受参数

无

#### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| error | IOError | 持久化错误信息 |

#### 伪代码实现

```
fn persistent_to_file() {
  if inner.persistent_file_path != null {
  	err = open(inner.persistent_file_path).write(dump_json(inner))
  	return err
  }
	null
}

err = persistent_to_file()
if !err {
	last_persistent_time = now()
}
err
```

### async_refresh()

异步将 DomainManager 中缓存的 DNS 解析结果刷新。这里的逻辑基于后端 SDK 常用的多线程结构实现异步刷新。如果是移动端 SDK 应该改用操作系统提供的异步方式。

#### 接受参数

无

#### 返回参数

无

#### 伪代码实现

```
fn sync_resolve_urls(urls) { // 由于刷新本身是异步行为，所以可以在失败的时候多尝试几次
	for _ in 0..inner.url_resolve_retries { // 默认尝试 10 次
		urls = urls.filter(|url| -> { resolve(url).len() == 0 }) // resolve（）实现参考上文的伪代码实现
		if urls.len() == 0 {
			break
		} else {
			wait(inner.url_resolve_retry_delay) // 等待随机时长，随机范围为设置的等待时间的 50% - 100%
		}
	}
}

to_refresh_urls = []
inner.resolutions.for_each((url, resolution) -> {
	if resolution.cache_deadline <= now() {
		to_refresh_urls.push(url)
	}
})
if to_refresh_urls.len() > 0 {
	spawn_thread(() -> {
		sync_resolve_urls(to_refresh_urls)
		inner.last_refresh_time = now()
	})
} else {
	inner.last_refresh_time = now()
}
```
