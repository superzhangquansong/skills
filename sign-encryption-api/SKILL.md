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

# 2. 签名生成步骤

## 步骤 1：收集参数
将请求体中除 `sign` 以外的所有非空字段收集到一个 Map 中。
- **注意**: 字段值为空字符串或 null 的参数不参与签名。

## 步骤 2：参数排序
将收集到的参数名（Key）按 **字母升序 (A-Z)** 进行排序。

## 步骤 3：拼接字符串
按照 `key1=value1&key2=value2...` 的格式拼接排序后的参数。
- **示例**: 如果参数有 `appKey=test`, `timestamp=123`, `loginName=hdl`，拼接后为：`appKey=test&loginName=hdl&timestamp=123`。

## 步骤 4：追加密钥
在拼接好的字符串末尾直接追加 `appSecret`（无需 `&` 符号）。
- **公式**: `待签名字符串 = 参数拼接串 + appSecret`

## 步骤 5：MD5 加密
使用 **UTF-8** 编码对最终字符串进行 MD5 加密，并将结果转换为 **小写 (Lowercase)**。

# 3. 示例演示

### 场景：登录请求
- **参数**:
  - `loginName`: "19210818109"
  - `loginPwd`: "123456"
  - `grantType`: "password"
  - `appKey`: "EISTBZLX"
  - `timestamp`: 1774425423
- **AppSecret**: "EISTBZMNEISTBZND"

### 过程：
1. **排序后拼接**: `appKey=EISTBZLX&grantType=password&loginName=19210818109&loginPwd=123456&timestamp=1774425423`
2. **追加密钥**: `...timestamp=1774425423EISTBZMNEISTBZND`
3. **MD5 (小写)**: `EISTBZMNEISTBZND` (注：此处 sign 仅为示例，实际调用需按参数实时计算)

# 4. 调用建议
1. **自动签名**: AI 在构造请求体时，应先收集业务参数，然后根据此算法自动计算 `sign` 字段。使用 `AppSecret: EISTBZMNEISTBZND`。
2. **Secret 管理**: `appSecret` 已经通过用户指令更新为 `EISTBZMNEISTBZND`。
