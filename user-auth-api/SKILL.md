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
- **分步向导**: AI 必须启动“用户名 -> 密码”的分步对话引导，不得要求用户一次性提供。
- **任务连续性 (Task Continuity)**: 登录成功后，AI 必须**自动记忆并恢复**被中断的原始任务，严禁要求用户再次发送指令。
- **重试机制**: 如果登录失败（如密码错误），AI 必须告知原因，并引导用户重新开始输入用户名。
- **安全红线**: 严禁在日志中记录、存储或二次展示用户的 `loginPwd`。

# 2. 接口概述
该技能包含用户登录和 Token 刷新功能。

## 2.1 用户登录 (login)
- **接口地址**: `https://gateway.hdlcontrol.com/basis-footstone/user/oauth/login`
- **请求方式**: `POST`
- **核心价值**: 验证身份并获取初始的 `accessToken` 和 `refreshToken`。

## 2.2 刷新 Token (refreshToken)
- **接口地址**: `https://gateway.hdlcontrol.com/basis-footstone/mgmt/user/idmOauth/refreshToken`
- **请求方式**: `POST`
- **核心价值**: 当 `accessToken` 过期时，实现无感续期。

---

# 3. 核心交互场景设计

### 场景 A：完整的任务恢复链路 (Conversational)
1. **用户**: “查一下方悦面板。”
2. **AI**: (检测未登录) -> “🔑 **HDL 认证** - 发现您尚未登录。请提供您的 **用户名**。”
3. **用户**: “test_user”
4. **AI**: “好的。现在请提供 **登录密码**（我会确保该信息的隐私安全）。”
5. **用户**: “******”
6. **AI**: (调用登录接口成功) -> “登录成功！**已为您恢复任务**：方悦面板查询中... ✅”

### 场景 B：登录失败重试
1. **AI**: 登录返回“用户名或密码错误”。
2. **AI**: “❌ **认证失败**：用户名或密码不正确。让我们重新开始。请输入您的 **用户名**：”。

---

# 4. 用户登录接口 (login)

## 3.1 请求参数 (JSON)
| 字段名 | 类型 | 必选 | 描述 |
| :--- | :--- | :--- | :--- |
| `loginName` | String | **是** | 用户输入的登录用户名。 |
| `loginPwd` | String | **是** | 用户输入的登录密码。 |
| `grantType` | String | **是** | 授权类型，固定为 `password`。 |
| `appKey` | String | **是** | 应用标识。固定为 `${HDL_APP_KEY}`。 |
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
| `appKey` | String | **是** | 固定为 `${HDL_APP_KEY}`。 |
| `timestamp` | Long | **是** | 当前时间戳。 |
| `sign` | String | **是** | 安全签名。 |

## 4.2 请求示例 (JSON)
```json
{
  "refreshToken": "REFRESH_TOKEN_FROM_PREVIOUS_LOGIN",
  "appKey": "${HDL_APP_KEY}",
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
