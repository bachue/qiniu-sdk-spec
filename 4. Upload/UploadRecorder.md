# UploadRecorder 上传日志记录仪

基于 Recorder 实现上传记录功能

| 名称       | 类型       | 描述                                               |
| ---------- | ---------- | -------------------------------------------------- |
| recorder | Recorder | 日志记录仪 |
| key_generator | fn(Path, String) -> String | 回调函数用于生成上传记录器的键，传入要上传的文件路径和对象名称，返回上传记录器的键，默认为参数连接后经过 SHA1编码后的值，以尽量保证不会冲突。 |
| upload_block_lifetime    | Duration（如果没有 Duration 类型，则使用 uint64，单位为秒） | 上传分块尺寸，尺寸越小越适合弱网环境，必须是 4 MB 的倍数。单位为字节，默认为 4 MB |
| always_flush_records    | bool | 是否在每次写入上传记录后刷新磁盘，默认不刷新 |

### 支持接口

#### create()

##### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| recorder | Recorder | 日志记录 |
| config | Config | SDK 客户端所用配置信息 |

##### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| error | IOError | IO 错误信息 |

##### 伪代码实现

```
UploadRecorder {
	recorder: recorder,
	key_generator: (path, key) -> {
		config.recorder_key_generator("upload", path, key)
	}
	upload_block_lifetime: config.upload_block_lifetime,
	always_flush_records: config.always_flush_records
}
````

#### open_and_write_metadata()

打开上传日志记录介质并写入元信息，该方法在上传一个新文件前调用

##### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| path | Path | 上传文件路径（对于没有路径的输入流，上传日志无法使用） |
| key | String | 对象名称 |
| upload_id | String | 上传 ID |
| up_urls | [String] | 当前区域的上传地址列表 |

##### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| medium |FileUploadRecordMedium| 返回文件上传日志介质 |
| error | IOError | IO 错误信息 |

##### 伪代码实现

```
metadata = FileUploadRecordMediumMetadata {
	file_size: path.file_size(),
	modified_timestamp: path.modified_timestamp(),
	upload_id: upload_id,
	up_urls: up_urls,
}
medium, err = recorder.open(key_generator(path, key), true)
if err {
	return null, err
}
err = medium.write("${dump_json(metadata)}\n")
if err {
	return null, err
}
if always_flush_records {
	err = medium.flush()
	if err {
		return null, err
	}
}
FileUploadRecordMedium { medium: medium, medium_lock: make_mutex(), always_flush_records: always_flush_records }, null
```

#### open_for_appending

打开一个已经存在的上传日志记录介质并追加数据，该方法在恢复一个上传到一半的文件时调用

##### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| path | Path | 上传文件路径（对于没有路径的输入流，上传日志无法使用） |
| key | String | 对象名称 |

##### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| medium |FileUploadRecordMedium| 返回文件上传日志介质 |
| error | IOError | IO 错误信息 |

##### 伪代码实现

```
medium, err = recorder.open(key_generator(path, key), false)
if err {
	return null, err
}
FileUploadRecordMedium { medium: medium, medium_lock: make_mutex(), always_flush_records: always_flush_records }, null
```

#### drop()

删除指定上传日志记录介质

##### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| path | Path | 上传文件路径（对于没有路径的输入流，上传日志无法使用） |
| key | String | 对象名称 |

##### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| error | IOError | IO 错误信息 |

##### 伪代码实现

```
recorder.delete(key_generator(path, key))
```

#### load()

加载指定上传日志记录介质，如果上传日志记录介质无法访问或内容与当前文件不匹配，则返回 null 表示弃用

##### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| path | Path | 上传文件路径（对于没有路径的输入流，上传日志无法使用） |
| key | String | 对象名称 |

##### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
|metadata|FileUploadRecordMediumMetadata|元信息|
|items|[FileUploadRecordMediumBlockItem]|分块项|
| error | IOError | IO 错误信息 |

##### 伪代码实现

```
reader = recorder.open((self.key_generator)(path, key), false)
line, err = reader.read_line()
if err {
	return null, null, err
}
if line.len() == 0 {
	return null, null, null
}
metadata = parse_json(line)
if metadata.file_size != path.file_size() || metadata.modified_timestamp != path.modified_timestamp() { // 文件内容可能已经被修改过
	return null, null, null
}
blocks = []
for {
	if err {
		return metadata, blocks, err
	}
	if line.len() == 0 {
		return metadata, blocks, null
	}
	block_item = parse_json(line)
	if now() > block_item.created_timestamp + upload_block_lifetime {
		return metadata, blocks, null
	}
	// 如何有需要，这里可以进一步考虑判断文件内容与记录中的 Etag 是否吻合
	blocks.push(block_item)
}
```

## FileUploadRecordMediumMetadata 文件上传日志记录介质元信息

记录当前文件的上传情况

| 名称       | 类型       | 描述                                               |
| ---------- | ---------- | -------------------------------------------------- |
| file_size | uint64 | 文件大小，单位为字节 |
| modified_timestamp    | uint64 | 文件修改时间戳 |
| upload_id    | 上传 ID |
| up_urls    | [String] | 当前区域的上传地址列表 |

## FileUploadRecordMediumBlockItem 文件上传日志记录分块项

| 名称       | 类型       | 描述                                               |
| ---------- | ---------- | -------------------------------------------------- |
| etag | String | 分块数据的 Etag |
| part_number    | uint | 分块号码 |
| created_timestamp    | uint64 | 创建分块的时间戳 |
| block_size    | uint64 | 分块大小，单位为字节 |

## FileUploadRecordMedium 文件上传日志记录介质


| 名称       | 类型       | 描述                                               |
| ---------- | ---------- | -------------------------------------------------- |
| medium | RecordMedium | 日志记录介质 |
| medium_lock | Mutex | 日志记录介质的锁，在多线程情况下，介质的写入需要加锁保护 |
| always_flush_records    | bool | 是否在每次写入上传记录后刷新磁盘，默认不刷新 |

### 支持接口

#### append()

加载指定上传日志记录介质

##### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| etag | String | 分块数据的 Etag |
| part_number | uint | 分块号码 |
| block_size | uint64 | 分块大小，单位为字节 |

##### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| error | IOError | IO 错误信息 |

##### 伪代码实现

```
item = FileUploadRecordMediumBlockItem {
	etag: etag,
	part_number: part_number,
	created_timestamp: now().to_timestamp(),
	block_size: block_size
}
medium_lock.lock()
err = medium.write("${dump_json(item)}\n")
if err {
	medium_lock.unlock()
	return err
}
if always_flush_records {
	err = medium.flush()
	if err {
		medium_lock.unlock()
		return err
	}
}
medium_lock.unlock()
null
```
