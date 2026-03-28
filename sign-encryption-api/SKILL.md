---
id: sign-encryption-api
name: 接口安全签名算法 (Sign Encryption)
description: 供 OpenClaw 调用，描述了 HDL 系统通用的 API 安全签名 (Sign) 生成算法。该算法确保了请求的完整性和安全性，是所有业务接口调用的核心前置步骤。
version: 1.0.0
tools: [getSign, sign]
tags: [security, sign, encryption, documentation]
permissions: [authenticated]
priority: 90
---

# 1. 算法概述
该签名算法基于 MD5 摘要，通过对请求参数进行排序并拼接密钥（AppSecret）生成。所有涉及安全验证的 HDL API 接口（如登录、产品查询等）均需在请求体中携带 `sign` 字段。

核心逻辑参考：`com.hdl.mcp.utils.RestUtil#getSign`

## 1. 签名算法核心步骤
所有 POST 请求的 JSON 根节点必须包含 `BaseDTO` 字段。其中 `sign` 的计算逻辑如下：

1. **收集参数**: 收集请求体中所有**非空**且**非 null** 的字段（排除 `sign` 字段本身）。
2. **排序参数**: 将所有收集到的字段名按 **ASCII 码从小到大** 排序（字典序）。
3. **拼接字符串**: 将排序后的参数按 `key=value` 格式拼接，并用 `&` 连接。
   - 示例: `appKey=${HDL_APP_KEY}&grantType=password&loginName=19210818109&loginPwd=123456&timestamp=1774425423`
4. **追加密钥**: 在拼接好的字符串末尾直接追加 `AppSecret`。
   - 示例: `...timestamp=1774425423${HDL_APP_SECRET}`
5. **计算 MD5**: 对最终生成的字符串进行 MD5 加密，并将结果转换为 **小写**。

## 2. 签名示例 (以登录请求为例)
- **BaseDTO 参数**:
  - `appKey`: "${HDL_APP_KEY}"
  - `timestamp`: 1774425423
- **AppSecret**: "${HDL_APP_SECRET}"

### 签名计算过程
1. **排序后拼接**: `appKey=${HDL_APP_KEY}&grantType=password&loginName=19210818109&loginPwd=123456&timestamp=1774425423`
2. **追加密钥**: `appKey=${HDL_APP_KEY}&grantType=password&loginName=19210818109&loginPwd=123456&timestamp=1774425423${HDL_APP_SECRET}`
3. **MD5 (小写)**: `3a5...` (注：此处 sign 仅为示例，实际调用需按参数实时计算)

## 3. 约束与安全
1. **自动签名**: AI 在构造请求体时，应先收集业务参数，然后根据此算法自动计算 `sign` 字段。使用环境变量 `${HDL_APP_SECRET}`。
2. **Secret 管理**: `appSecret` 已经统一存储在 `.env` 文件的 `HDL_APP_SECRET` 中，严禁在文档中硬编码。
