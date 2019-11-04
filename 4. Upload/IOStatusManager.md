# IOStatusManager IO 状态管理器

内部类，为并发上传任务提供状态管理

| 名称                 | 类型                    | 描述                                                         |
| -------------------- | ----------------------- | ------------------------------------------------------------ |
| status                | Status | 当前状态                                                     |
| lock                | Mutex | 锁                                                     |

## 内部依赖结构体

### Status 当前状态

枚举类，包含三种取值，Uploading（初始化状态），Error，Success

如果是 Uploading，则包含以下数据

| 名称           | 类型         | 描述                                      |
| -------------- | ------------ | ----------------------------------------- |
| reader   | Reader | 输入流 |

如果是 Error，则包含以下数据

| 名称           | 类型         | 描述                                      |
| -------------- | ------------ | ----------------------------------------- |
| error   | Error | 错误信息 |

如果是 Success，则不包含数据

## 支持接口

### read()

线程安全的方法，如果当前状态是 Uploading，则读取指定大小的数据并返回，否则返回 null

#### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| size | uint | 读取数据大小 |

#### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| data | [uint8] | 读取的数据流，或 null |

#### 伪代码实现

```
lock.lock()
if status == Uploading {
	data = 0
	loop { // 尽可能读取更多数据，确保返回的尺寸达到预期，否则将会造成 UploadPart 出错
		chunk, err = reader.read(size - data.len())
		if err {
			status = Error(err)
			lock.unlock()
			return null
		}
		if chunk.len() == 0 {
			lock.unlock()
			return data
		}
		data.push(chunk)
		if data.len() == size {
			lock.unlock()
			return data
		}
	}
} else {
	lock.unlock()
	null
}
```

### error()

线程安全的方法，如果外部上传出错，则应该调用该方法，传入错误信息，来改变状态

#### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| error | Error | 错误信息 |

#### 返回参数

无

#### 伪代码实现

```
lock.lock()
status = Error(error)
lock.unlock()
```

### result()

上传完毕后，获取错误信息，如果当前状态为 Success 则返回 null

#### 接受参数

无

#### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| error | Error | 错误信息 |

#### 伪代码实现

```
// 这里无需再加锁
if status == Error {
	error
} else {
	null
}
```