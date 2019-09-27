# Token HTTP 签名方式

枚举类，包含 QBox 和 Qiniu 两种方式（也可以称为 V1 和 V2）。此外，如果编程语言本身不支持 NULL，可以增加 NULL 这种方式（即表示不做签名）。

| 名称       | 类型   | 描述                |
| ---------- | ------ | ------------------- |
| credential | Credential | 七牛认证信息，当 Token 是 QBox 或 Qiniu 时必须存在 |

## 支持接口

### sign()

获取存储管理器

#### 接受参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| request | Request | 要签发的 HTTP Request |

#### 返回参数

无

#### 伪代码实现

```
switch token {
case QBox:
	request.headers.set("Authorization", credential.authorization_v1_for_request(request.url(), request.headers(), request.body()))
case Qiniu:
	request.headers.set("Authorization", credential.authorization_v2_for_request(request.url(), request.method(), request.headers(), request.body()))
default:
  // do nothing
}
```


