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
- **动态登录**: 当 AI 检测到没有有效的 `accessToken` 时，必须启动分步式引导流程。
- **自动刷新**: 当业务请求返回 Token 过期（如 code: 401）时，AI 必须尝试调用 `refreshToken` 接口。
- **安全红线**: 严禁在日志中记录用户的 `loginPwd`。登录失败后，严禁继续执行后续业务技能。

# 2. 接口概述
该技能包含用户登录和 Token 刷新两个核心功能。

## 2.1 用户登录 (login)
- **接口地址**: `https://gateway.hdlcontrol.com/basis-footstone/user/oauth/login`
- **请求方式**: `POST`
- **核心价值**: 验证身份并获取初始的 `accessToken` 和 `refreshToken`。

## 2.2 刷新 Token (refreshToken)
- **接口地址**: `https://gateway.hdlcontrol.com/basis-footstone/mgmt/user/idmOauth/refreshToken`
- **请求方式**: `POST`
- **核心价值**: 当 `accessToken` 过期时，使用 `refreshToken` 获取新的令牌，无需用户再次输入密码。

---

# 3. 核心交互场景设计

### 场景 A：分步式引导登录
1. AI：“🔑 **HDL 身份认证** - 请提供您的**用户名**。”
2. 用户：“user123”
3. AI：“好的。现在请提供**登录密码**。”
4. 用户：“pwd456”
5. AI：计算签名 -> 调用 `login` 接口 -> 成功 -> 继续业务。

### 场景 B：登录失败重试
1. AI 调用 `login` 返回 401（密码错误）。
2. AI：“❌ **登录失败**：用户名或密码错误。让我们**重新开始**认证。请输入您的**用户名**：”。

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
