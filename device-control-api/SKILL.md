---
id: device-control-api
name: 智能家居设备查询与控制 API
description: 供 OpenClaw 调用，用于获取用户家庭中的设备列表及其状态，并执行开关、调光、温度调节等控制操作。
version: 1.1.1
tools: [getDeviceControlData, controlDevice, getDeviceInfo]
tags: [api, device, control, smart-home, documentation]
permissions: [authenticated]
priority: 60
---

# 1. 强制认证与安全约束
- **身份验证**: 所有接口调用必须在请求头中携带有效的 `Authorization: Bearer {accessToken}`。
- **动态登录**: 如果当前没有 `accessToken`，AI **必须**先调用 `user-auth-api` 技能，并提示用户输入用户名和密码以获取 Token。
- **Token 刷新**: 如果接口返回 Token 过期（如 401 错误或特定的过期提示），AI 应自动尝试使用 `refreshToken` 进行刷新。
- **无 Token 不业务**: 只要没有有效的 Token，严禁执行任何设备查询或控制逻辑。

# 2. 接口概述
该技能包含三个核心接口：用于**获取设备列表**、**发送控制指令**以及**获取设备详情**。所有接口均采用 `POST` 方式，并要求完整的安全签名。

## 1.1 获取设备列表 (getDeviceControlData)
- **接口地址**: `https://gateway.hdlcontrol.com/home-wisdom/app/device/list`
- **认证方式**: `Bearer Token`
- **核心逻辑**: 分页或全量获取当前家庭下的所有设备及其简要状态。

## 1.2 控制设备 (controlDevice)
- **接口地址**: `https://gateway.hdlcontrol.com/home-wisdom/app/device/control`
- **核心逻辑**: 向网关发送控制指令，支持批量控制。

## 1.3 获取设备详情 (getDeviceInfo)
- **接口地址**: `https://gateway.hdlcontrol.com/home-wisdom/app/device/info`
- **核心逻辑**: 根据设备 ID 列表获取设备的详细信息，包括物模型协议、属性配置、实时状态等。

---

# 2. 通用安全参数 (BaseDTO)
所有请求的 JSON 根节点必须包含以下安全验证字段：

| 字段名 | 类型 | 必选 | 描述 | 示例 |
| :--- | :--- | :--- | :--- | :--- |
| `appKey` | String | **是** | 应用标识，固定为 `${HDL_APP_KEY}`。 | `${HDL_APP_KEY}` |
| `timestamp` | Long | **是** | 13 位毫秒级时间戳。 | `1774425423000` |
| `sign` | String | **是** | 安全签名。 | `abc123xyz...` |

---

# 3. 获取设备列表接口 (getDeviceControlData)

## 3.1 请求参数 (AppDeviceListDTO)
| 字段名 | 类型 | 必选 | 描述 | 示例 |
| :--- | :--- | :--- | :--- | :--- |
| `homeId` | Long | **是** | 住宅房屋 ID。**必须固定使用：`${HDL_HOME_ID}`**。 | `${HDL_HOME_ID}` |
| `gatewayId` | Long | 否 | 指定网关 ID 查询。 | `1483281443578613762` |
| `searchType` | String | 否 | 查询方式：`ALL`(全量, 默认), `PAGE`(分页)。 | `"ALL"` |
| `roomId` | Long | 否 | 按房间 ID 过滤。 | `1483281443578613763` |
| `spk` | String | 否 | 按功能类型过滤（如 `light.dimming`）。 | `"light.dimming"` |
| `isGetProtocol` | Boolean | 否 | 是否查询设备协议，默认 `false`。 | `true` |
| `collect` | String | 否 | 是否只查询收藏设备：`"1"`(是), `"0"`(否)。 | `"1"` |
| `pageSize` | Long | 否 | 每页条数（`searchType=PAGE` 时生效）。 | `10` |
| `pageNo` | Long | 否 | 当前页码（`searchType=PAGE` 时生效）。 | `1` |

## 3.2 请求示例 (JSON)
```json
{
  "homeId": ${HDL_HOME_ID},
  "searchType": "ALL",
  "isGetProtocol": true,
  "appKey": "${HDL_APP_KEY}",
  "timestamp": 1774425423000,
  "sign": "abc123xyz..."
}
```

## 3.3 响应结果 (Result<PageVO<DeviceVO>>)
```json
{
  "code": 0,
  "isSuccess": true,
  "data": {
    "total": 1,
    "list": [
      {
        "deviceId": 1483281466097831937,
        "name": "客厅吸顶灯",
        "spk": "light.dimming",
        "gatewayId": 1483281443578613762,
        "online": true,
        "attributes": [
          { "key": "on_off", "value": "on" },
          { "key": "brightness", "value": "80" }
        ]
      }
    ]
  }
}
```

---

# 4. 控制设备接口 (controlDevice)

## 4.1 请求参数 (AppDeviceControlDTO)
| 字段名 | 类型 | 必选 | 描述 | 示例 |
| :--- | :--- | :--- | :--- | :--- |
| `homeId` | Long | **是** | 住宅房屋 ID。固定使用：`${HDL_HOME_ID}`。 | `${HDL_HOME_ID}` |
| `gatewayId` | Long | **是** | 设备所属的网关 ID。 | `1483281443578613762` |
| `actions` | List | **是** | 控制动作列表。 | (见下文) |

### Action 结构
- `deviceId` (Long, 必填): 目标设备 ID。
- `spk` (String, 可选): 功能类型。
- `attributes` (List<StatusBean>, 必填): 控制属性列表。
  - `key` (String): 属性键（如 `on_off`, `brightness`, `target_temperature`）。
  - `value` (String): 属性值（如 `on`, `off`, `50`, `26`）。

## 4.2 控制请求示例 (JSON)
```json
{
  "homeId": ${HDL_HOME_ID},
  "gatewayId": 1483281443578613762,
  "actions": [
    {
      "deviceId": 1483281466097831937,
      "spk": "light.dimming",
      "attributes": [
        { "key": "on_off", "value": "on" },
        { "key": "brightness", "value": "100" }
      ]
    }
  ],
  "appKey": "${HDL_APP_KEY}",
  "timestamp": 1774425423000,
  "sign": "abc123xyz..."
}
```

## 4.3 响应结果
```json
{
  "code": 0,
  "isSuccess": true,
  "data": true,
  "msg": "操作成功"
}
```

---

# 5. 获取设备详情接口 (getDeviceInfo)

## 5.1 请求参数 (AppDeviceGetDTO)
| 字段名 | 类型 | 必选 | 描述 | 示例 |
| :--- | :--- | :--- | :--- | :--- |
| `homeId` | Long | **是** | 住宅房屋 ID。固定使用：`${HDL_HOME_ID}`。 | `${HDL_HOME_ID}` |
| `deviceIds` | List<Long> | **是** | 需要查询详情的设备 ID 列表。 | `[1483281466097831937]` |

## 5.2 请求示例 (JSON)
```json
{
  "homeId": ${HDL_HOME_ID},
  "deviceIds": [1483281466097831937],
  "appKey": "${HDL_APP_KEY}",
  "timestamp": 1774425423000,
  "sign": "abc123xyz..."
}
```

## 5.3 响应结果 (Result<List<DeviceVO>>)
返回 `DeviceVO` 列表。每个对象包含：
- `deviceId`: 设备 ID。
- `name`: 设备名称。
- `spk`: 功能类型。
- `icon`: **设备图标 URL (用于 UI 展示)**。
- `imageUrl`: **设备实时状态图片 URL (若存在则建议展示)**。
- `attributes`: 当前属性列表（key/value）。
- `productProtocol`: 完整的物模型定义（包含该设备支持的所有控制指令及其取值范围）。
- `online`: 在线状态。

---

# 6. 调用策略与最佳实践
1. **先查后控**: AI 应当先通过 `getDeviceControlData` 获取设备列表，识别出 `deviceId`、`gatewayId` 和支持的 `spk`。
2. **状态反馈**: 控制成功后，AI 应当根据接口返回的 `isSuccess` 状态，结合控制时的属性值（如 `on_off: on`）告知用户操作结果。
3. **精准匹配**: 构造控制指令时，必须确保 `key` 和 `value` 与设备协议 (`productProtocol`) 中的定义完全一致。
4. **异常处理**: 若接口返回 `code != 0` 或 `isSuccess == false`，应清晰告知用户失败原因（如“网关离线”或“设备响应超时”）。
