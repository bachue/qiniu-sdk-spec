# UploadToken 上传凭证

上传凭证是一个枚举类，包含两种值

| 名称       | 描述       |
| ---------- | ---------- |
|Token(token)|包含一个凭证字符串|
|Policy(policy, credential)|包含上传策略以及认证信息|

## 支持接口

### token()

返回上传凭证字符串，对于有条件的编程语言，可以缓存结果（但务必保证与 Policy 同步）。

#### 接受参数

无

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| token | String | 上传凭证 |

#### 伪代码实现

```
switch token {
case Token:
	token
case Policy:
	credential.sign_upload_policy(policy)
}
```

### policy()

返回上传策略，对于有条件的编程语言，可以缓存结果（但务必保证与 Token 同步）。

#### 接受参数

无

#### 返回参数

| 名称       | 类型       | 描述                            |
| ---------- | ---------- | --------------------------------- |
| policy | UploadPolicy | 上传策略 |

#### 伪代码实现

```
switch token {
case Token:
	make_upload_policy_from_json(urlsafe_base64_decode(token.split(':', 3).get(2)))
case Policy:
	policy
}
```

