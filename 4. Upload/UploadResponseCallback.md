# 上传响应回调

## 支持接口

### upload_response_callback

判断响应是否确实来自七牛，如果不是则让客户端重试。

#### 接受参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| response | Response | HTTP 响应 |

#### 返回参数

| 名称    | 类型    | 描述                  |
| ------- | ------- | --------------------- |
| error | HTTPError | HTTP 错误信息 |

#### 伪代码实现

```
if response.header("X-ReqId") != null { // 在返回值是 2xx 的情况下，X-ReqId 必然存在，否则肯定有问题 
	null
} else {
	HTTPError(Retryable, true, MaliciousResponse)
}
```

