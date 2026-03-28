---
id: user-auth-api
name: 用户身份认证 API
description: 供 OpenClaw 调用，用于通过用户名和密码获取 HDL 系统的身份令牌 (AccessToken)。该 Token 是调用其他业务接口（如产品查询）的前提。
version: 1.1.0
tools: [login, refreshToken]
tags: [auth, login, token, documentation, refresh]
permissions: [authenticated]
priority: 100
---

# 1. 强制认证与安全约束
- **二维码 H5 优先**: AI 必须优先生成一个**限时安全登录二维码**，引导用户扫码在外部 H5 页面进行登录。
- **WebSocket 凭据回传**: H5 登录成功后，会通过 WebSocket 将用户的 `loginName` 和 `loginPwd` 回传给 AI。
- **AI 代理登录**: AI 收到回传凭据后，必须立即调用 `login` 接口进行身份验证。
- **任务连续性 (Task Continuity)**: 登录成功后，AI 必须**自动恢复**被中断的原始业务任务，严禁要求用户再次发送指令。
- **重试机制**: 如果登录失败（如密码错误），AI 必须告知用户，并**重新生成**一个新的限时二维码引导扫码。
- **安全红线**: 严禁在日志中记录、存储或二次展示用户的 `loginPwd`。

# 2. 接口概述
该技能包含二维码生成、执行登录和 Token 刷新等核心功能。

## 2.1 安全登录二维码 (getLoginQRCode)
- **H5 地址**: `http://14.103.240.202:8588/ClawLogin.html?sessionId={sessionId}`
- **WS 端点**: `ws://14.103.240.202:8588/ws-openclaw-login`
- **有效期**: 5 分钟。
- **回传机制**: 用户在 H5 页面登录后，凭据通过 WebSocket 实时推送到 AI 侧。

## 2.2 用户登录 (login)
- **接口地址**: `https://gateway.hdlcontrol.com/basis-footstone/user/oauth/login`
- **核心价值**: 使用回传的凭据换取 `accessToken`。

---

# 3. 核心交互场景设计

### 场景 A：完整的二维码登录链路
1. **用户**: “帮我查询方悦面板。”
2. **AI**: 检测到未登录 -> 生成二维码。
3. **AI**: “🔑 **HDL 安全认证** - 请扫码登录。完成后我会自动为您继续执行之前的任务。
   ![二维码](https://api.qrserver.com/v1/create-qr-code/?size=150x150&data=http://14.103.240.202:8588/ClawLogin.html?sessionId=xxx)
   (二维码 5 分钟内有效)”
4. **用户**: (扫码并在 H5 页面输入用户名密码)
5. **AI**: (WebSocket 收到回传凭据 -> 自动调用 `login`) -> 成功。
6. **AI**: “登录成功！**已为您恢复任务**：方悦面板查询中... ✅”

### 场景 B：登录失败重试
1. **AI**: 自动登录返回“用户名或密码错误”。
2. **AI**: “❌ **登录失败**：您在 H5 页面输入的用户名或密码不正确。
   为了您的账户安全，我已为您生成了新的登录二维码，请重新扫码尝试：
   ![新二维码](https://api.qrserver.com/v1/create-qr-code/?size=150x150&data=http://14.103.240.202:8588/ClawLogin.html?sessionId=new_xxx)”

---

# 4. 用户登录接口 (login)

## 3.1 请求参数 (JSON)
| 字段名 | 类型 | 必选 | 描述 |
| :--- | :--- | :--- | :--- |
| `loginName` | String | **是** | 用户输入的登录用户名。 |
| `loginPwd` | String | **是** | 用户输入的登录密码。 |
| `grantType` | String | **是** | 授权类型，固定为 `password`。 |
| `appKey` | String | **是** | 应用标识。固定为 `EISTBZLX`。 |
| `timestamp` | Long | **是** | 当前请求的时间戳（秒）。 |
| `sign` | String | **是** | 安全签名。由 `sign-encryption-api` 计算。 |

## 3.2 响应结果
```json
{
  "code": 0,
  "isSuccess": true,
  "data": {
    "accessToken": "...",
    "refreshToken": "...",
    "expiresIn": 86400
  }
}
```

---

# 4. 刷新 Token 接口 (refreshToken)

## 4.1 请求参数 (JSON)
| 字段名 | 类型 | 必选 | 描述 |
| :--- | :--- | :--- | :--- |
| `refreshToken` | String | **是** | 登录成功后返回的刷新令牌。 |
| `appKey` | String | **是** | 固定为 `EISTBZLX`。 |
| `timestamp` | Long | **是** | 当前时间戳。 |
| `sign` | String | **是** | 安全签名。 |

## 4.2 请求示例 (JSON)
```json
{
  "refreshToken": "REFRESH_TOKEN_FROM_PREVIOUS_LOGIN",
  "appKey": "EISTBZLX",
  "timestamp": 1774517844,
  "sign": "..."
}
```

## 4.3 响应结果
```json
{
  "code": 0,
  "isSuccess": true,
  "data": {
    "accessToken": "NEW_ACCESS_TOKEN",
    "refreshToken": "NEW_REFRESH_TOKEN",
    "expiresIn": 86400
  }
}
```

# 5. 调用策略
1. **优先认证**: 业务操作前检查 Token。若无，引导用户登录。
2. **静默刷新**: 遇到 401 错误，AI 优先调用 `refreshToken`，成功后重试原业务请求。
3. **彻底注销**: 若刷新也失败，清除所有 Token 并提示用户重新登录。
