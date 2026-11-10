# 领克 GRIC 车辆接口 —— 逆向发现与接入笔记

> 本文记录本次「从提取 GRIC 密钥 → 打通车辆查询接口 → 接入 GitHub Actions → 新增控车能力」的完整发现与结论。
> 所有敏感值（密钥、JWT、VIN、车牌、设备号、SIM 号、坐标等）均以占位/打码形式出现，**真实值只放本地 `geely_env.json` / 仓库 Secrets，切勿提交**。

---

## 0. 结论速览

- 领克 App 的**车辆相关接口走的是吉利 GRIC 网关**，与「签到/分享」用的领克 `app-api-gw-toc.lynkco.com` 是**两套完全不同的系统**（签名算法、密钥、域名都不同）。
- GRIC 请求签名 = `Base64(HMAC-SHA256(key=gricSecret, msg=STS))`，STS 规则见 §4。
- 认证链：**领克登录态 → accessCode → GRIC JWT**；Android 现行版本用 `v4/code` 直接拿 accessCode（无需 `appsecret`）。
- 业务接口（心跳/状态/详情/控制）当前 App 走 **`gric-g-api.geely.com`**（旧代码里的 `gric-hf-api` 是旧域名，会导致「人车校验失败 / 刷新令牌无效」）。
- 车辆上下文用两个头：`x-vehicle-identifier`（AES 加密的 VIN）与 `x-vehicle-series`（Base64 车型代码），**本地 AES 默认密钥与抓包完全一致**。
- 还必须有**非空且≥18 位的 `x-device-id`**（用真实设备号最稳），否则网关报 `00A02 header property 'X-DEVICE-ID' is required`。
- 远程控制接口 `remoteControl/control` 已摸清，**已证实一条：闪灯寻车（`serviceId=RHL`, `rhl=light-flash`）**；其余控车动作需按动作抓包补 `serviceId`。

---

## 1. 两套体系的区分

| 体系 | 网关 | 签名 | 密钥来源 | 用途 |
|---|---|---|---|---|
| 领克签到/分享/探索 | `app-api-gw-toc.lynkco.com`、`app-services.lynkco.com.cn` | 阿里云 `x-ca-*` HMAC-SHA256 | `env.json` 的 `nativeAppKey/nativeAppSecret` | 签到、分享、积分 |
| **吉利 GRIC 车辆** | `gric-g-api.geely.com` / `gric-api.geely.com` | 自有 `X-SIGNATURE v2.1` | `geely_env.json` 的 `gricSecret` | 车辆心跳/状态/控制 |

> 关键教训：不要把 `lykcoAppsecret`、`nativeAppSecret`、`gricSecret` 混为一谈，它们是三把不同的钥匙（用途/长度都不同）。

---

## 2. 认证链（auth）

### 2.1 现行 Android 流程（推荐，v4/code）

```
领克登录态(token/refreshToken)
   │  GET https://app-services.lynkco.com.cn/auth/oauth2/v4/code
   │      ?responseType=code&clientId=30000028&scope=snsapiUserinfo
   │  （原生 x-ca 签名，仅签 x-ca-nonce / x-ca-timestamp / x-ca-key；带 token 头）
   ▼
data.accessCode （一次性、短时效）
   │  POST https://gric-api.geely.com/<get/token 路径>
   │  body {"authCode": accessCode, "identityType":"1"}
   │  GRIC 签名（此请求无 authorization 头）
   ▼
GRIC JWT（约 7 天有效）
```

- `clientId=30000028`、`scope=snsapiUserinfo`、`responseType=code` 是固定参数（`clientId` 可用环境变量 `GEELY_V4_CLIENT_ID` 覆盖）。
- 该流程**不需要 `lykcoAppsecret`**（它是 iOS 旧流程 `access-code` 用的）。

### 2.2 旧 iOS 流程（保留为回退）

```
GET https://app-services.lynkco.com.cn/auth/oauth2/access-code?scope=openid&state=vehiclenet
请求头需带：appsecret: <lykcoAppsecret>  （从 App 抓包 access-code 请求头提取）
```

- `get_access_code()` 已实现：先试 v4/code，失败自动回退到旧的 access-code(appsecret) 流程。

### 2.3 网关域名映射（重要）

| 用途 | 域名 |
|---|---|
| get/token、favorite-vehicles | `gric-api.geely.com` |
| **心跳 hb / 车辆状态 status / 远程控制 control** | **`gric-g-api.geely.com`** |
| （旧/其它） | `gric-hf-api.geely.com`（旧域名，勿用） |

> 签名 STS **不含 host**，所以换域名不影响签名；但**用错域名会被不同后端判为「人车校验失败 / 刷新令牌无效」**。

---

## 3. 配置文件与密钥

`geely_env.json`（已 gitignore）关键字段：

| 字段 | 说明 |
|---|---|
| `gricSecret` | GRIC X-SIGNATURE 的 HMAC 密钥（由 `extract-gric-key` 工作流提取） |
| `authorization` | GRIC JWT（auth 自动写入，约 7 天） |
| `device.id` | 设备号，**必须≥18位**；建议填 App 抓包里的真实 `x-device-id` |
| `vehicle.identifier` | `Base64(AES-128-CBC-PKCS5(key,iv,VIN))` |
| `vehicle.series` | `Base64(车型代码)`，如 `Base64("L743") = "TDc0Mw=="` |
| `lykcoAppsecret` | 仅旧 iOS access-code 流程需要，v4/code 不需要 |

环境变量优先级：**单字段环境变量 > `LYNKCO_APP_SECRETS`/JSON > `geely_env.json` > 内置默认**。
常用：`GEELY_GRIC_SECRET`、`GEELY_DEVICE_ID`、`GEELY_VEHICLE_IDENTIFIER`、`GEELY_VEHICLE_SERIES`、`GEELY_GRIC_BASE`、`GEELY_GRIC_BASE_API`。

---

## 4. X-SIGNATURE v2.1 签名规则

```
X-SIGNATURE = Base64( HMAC-SHA256( key=gricSecret, msg=STS ) )
```

STS（string-to-sign）按行用 `\n` 拼接、**无尾随换行**：

```
accept-language:<值>                 ← 固定参与
authorization:<JWT>                  ← 固定参与（签名里包含 JWT！）
<全部 x-* 请求头，按头名升序，"name:value">
<Base64(MD5(body))>                   ← 仅当 body 非空
<query string 原文，如 os=1>          ← 仅当 URL 带 query
<HTTP 方法，POST/GET>
<path（不含 query）>
```

> 注意：`accept-language` 参与签名；`accept` / `content-type` **不**参与签名，可自由设置。

---

## 5. 车辆上下文头 & 常见网关错误

| 请求头 | 值 / 规则 | 缺失报错 |
|---|---|---|
| `x-app-id` | `GEELYCNCH001M0001` | — |
| `x-tenant-id` / `x-sales-platform` | 必须 `LYNKCO`（否则身份解析到别的用户） | — |
| `x-device-id` | **≥18 位**，建议用真实设备号 | `00A02 ... 'X-DEVICE-ID' is required` |
| `x-vehicle-identifier` | AES 密文（见上） | `00A02 ... 'X-VEHICLE-IDENTIFIER' is required` |
| `x-vehicle-series` | Base64(车型代码) | — |

**AES 口径**：`identifier = Base64(AES-128-CBC-PKCS5(key16, iv16, VIN))`。
实测：本车抓包的 `x-vehicle-identifier` 与本地 AES 默认 key/iv 计算值**逐字符一致**，说明该口径正确。
若服务端报 `Decrypt X-VEHICLE-IDENTIFIER failed`，说明密钥轮换 → 用 `tools/extract_gric_key.py vehicle --write` 从 App 堆里取真实密文，或覆盖 `GEELY_VEH_AES_KEY/IV`。

---

## 6. 已打通的接口（全部返回 `code:0 success`）

| 动作 | 方法与路径（host=`gric-g-api`，favorite 走 `gric-api`） | 说明 |
|---|---|---|
| `hb` | `POST /ms-app-online-center/api/v2.0/app/hb` | 心跳，body `{deviceType:1,enableWakeUp:false,hbType:N,ts}`；返回 `remoteControlMsgList` |
| `favorite` | `GET /ms-vehicle-core/api/v1.0/vehicle/favorite-vehicles`（`gric-api`） | 收藏车辆（车型/颜色/VIN/车牌…） |
| `detail` | `GET /ms-vehicle-account/api/v1.0/vehicle-detail?os=1` | 车辆详情（ICCID/IMSI/MSISDN/T-BOX ID/VIN） |
| `status` | `GET /ms-vehicle-status/api/v2.0/vehicle/status/latest?latest=&target=new` | 实时车况（续航/电量/油量/胎压/门窗锁/位置…） |
| `msg` | `GET /ms-app-message-center/api/v1.0/car/message/messageCount` | 返回 `alarmsTotal`（告警数） |

> `status` 必须带 query `latest=&target=new`，否则返回异常。

---

## 7. 远程控制（控车）

### 7.1 接口

```
POST https://gric-g-api.geely.com/ms-remote-control/api/v1.0/remoteControl/control
请求头：与查询相同的 x-vehicle-* / authorization / x-device-id / x-signature
body:
{
  "command": "",
  "serviceId": "<动作ID>",
  "setting": { "serviceParameters": [ { "key": "<k>", "value": "<v>" } ] }
}
```

响应：`{"code":"0","data":{"sessionId":"..."},"msg":"success"}` —— **异步**，真正执行结果在后续心跳 `hb` 的 `remoteControlMsgList` 里回报。

### 7.2 已证实的一条

闪灯寻车：`serviceId="RHL"`，`serviceParameters=[{"key":"rhl","value":"light-flash"}]`，返回 `code 0 + sessionId`。

### 7.3 已实现的命令

```bash
python geely_gric.py flash                               # 闪灯寻车（RHL/light-flash）
python geely_gric.py control RHL rhl=light-flash         # 通用回放
python geely_gric.py control <serviceId> k=v [k2=v2 ...] # 回放任意抓到的控制
```

### 7.4 更多控车项（需按动作抓包补 `serviceId`）

该车 status 字段表明可能支持：寻车闪灯/鸣笛、车门解/闭锁、后备箱、车窗、天窗、空调（开关/设温/预约/极速降温）、座椅加热通风、方向盘加热、充电（开始/停止/预约/健康充）、远程启动（视车型）。
**每个动作的 `serviceId`/参数不同，需在 App 里操作一次并抓对应的 `remoteControl/control` 请求来确认。**

### 7.5 高敏操作的门槛

App 的 `getBiometricConfig` 显示 `faceId/touchId` 已开启（周期 30s）。**解闭锁类高敏动作可能要求生物二次校验**，请求里可能多带一个验证 token，需抓包确认。

---

## 8. GitHub Actions 工作流

文件：`.github/workflows/geely-gric.yml`，手动触发（`workflow_dispatch`），可选 `action`（all/hb/favorite/detail/status/msg）。

流程：`auth → vehicle-init → 按选择跑查询`；`auth` 失败即终止，业务查询失败不阻断。

所需仓库 **Secrets（放在 Secrets，不是 Variables）**：

| Secret | 必需 | 说明 |
|---|---|---|
| `LYNKCO_REFRESH_TOKEN` + `LYNKCO_DEVICE_ID` | ✅ | 领克登录态（或 `LYNKCO_TOKEN`） |
| `LYNKCO_APP_SECRETS`（或 `LYNKCO_NATIVE_APP_KEY`+`LYNKCO_NATIVE_APP_SECRET`） | ✅ | 原生签名密钥（x-ca 签名） |
| `GEELY_GRIC_SECRET` | ✅ | GRIC 密钥 |
| `GEELY_DEVICE_ID` | 建议 | 真实 `x-device-id`（≥18位），与 App 一致 |
| `GEELY_VEHICLE_IDENTIFIER` / `GEELY_VEHICLE_SERIES` | 可选 | 仅当不想用 `vehicle-init` 本地算的值时 |
| `GEELY_LYNKCO_APPSECRET` | 可选 | 仅旧 iOS access-code 流程 |

> 注意：工作流曾因 `LYNKCO_APP_SECRETS` 被放到 Variables 而“变量都设了却报缺密钥”，务必确认在 **Secrets** 下。

---

## 9. 排错时间线（本次踩过的坑 → 原因 → 修复）

| 现象 | 原因 | 修复（提交） |
|---|---|---|
| extract-gric-key 绿灯但未提取到密钥 | 无 HAR、App 未登录无 STS 现场，无法验证；且抓堆崩溃即停只抓 2 轮 | 新增“在线签名校验 oracle”（错误密码被网关判 `00A06`，唯一通过者即真密钥）+ 崩溃重启续抓（`d40df9a`、`78ffacf`） |
| auth 报“缺少必需的签名密钥（NATIVE_APP_CODE / NATIVE_APP_KEY）” | CI 里 `LYNKCO_APP_SECRETS` 实际为空（多因放到 Variables） | 正确在 Secrets 配 `LYNKCO_APP_SECRETS`（JSON）或分字段（`4442a86`） |
| get/token `00A02 X-DEVICE-ID is required` | `device.id` 空被过滤；且需≥18位 | 新增 `_resolve_device_id()`（env>配置>生成并写回），`b384d7d` |
| auth 后 accessCode 拿不到 | 旧代码用 iOS `/auth/oauth2/access-code`+appsecret，与 Android 抓包不符 | `get_access_code` 优先走 Android `v4/code`，回退旧流程（`2450b50`） |
| hb「人车校验失败」/ detail `8040` / status `刷新令牌无效` | 业务域名用错（`gric-hf-api`→应为 `gric-g-api`）、status 缺 `latest=&target=new`、设备号非真实 | 切域名 + 补 query + 真实 `GEELY_DEVICE_ID`（`ec95cd3`） |
| 日志明文泄露 VIN/车牌/SIM/坐标 | 未脱敏 | 扩展 `_mask` 打码（`02aff4e`、`e1b7044`、`e6bb07e`） |
| 控车能力缺失 | 未实现 | 新增 `remote_control` + `control`/`flash`（`801d0e9`） |

---

## 10. 日志脱敏范围

日志打印（`_mask`）会对以下字段打码（首3尾2，过短整体 `***`）：
`userId/openId/accountId、VIN、车牌(plateNo)、identifier/x-vehicle-identifier、pno/pno18、deviceId、iccid/imsi/msisdn/tboxId、经纬度、手机号、token/authorization`。

> 配置文件里仍是真实值；脱敏只作用于**日志输出**。

---

## 11. 已知限制 / 待办

- 更多控车动作的 `serviceId`/参数需按动作抓包确认（工作流/代码已支持通用回放）。
- 高敏控制（解闭锁等）生物二次校验流程尚未摸清。
- `v4/code` 为 Android 流程；iOS 仍走 `access-code`+`appsecret`。
- `msg` 的语义是“告警数 alarmsTotal”，非未读消息数；未读消息其实走领克 `app-menu/countUserUnReadMessage`（另一套签名）。
- `status` 返回体很大，CI 日志冗长；可考虑加 `status --brief` 摘要。
- 控车是写操作且有风控/异步回执，**建议只手动触发，勿加入定时任务**。

---

## 12. 安全提醒

- `gricSecret`、JWT、`nativeAppKey/Secret`、`refreshToken`、真实 `x-device-id`、VIN/车牌/SIM/坐标均为敏感信息。
- 只放本地 `geely_env.json`（已 gitignore）或仓库 **Secrets**；不要写进代码、日志、Issue、截图。
- 抓包 HAR（含 JWT/VIN/driverId 等）不要提交（`*.har` 已 gitignore）。

---

## 13. 使用速查

```bash
# 本地（需 env.json 有登录态 + geely_env.json 有 gricSecret）
cd LynkCoHelper
python geely_gric.py auth            # 登录态 -> accessCode -> JWT（写回 authorization）
python geely_gric.py vehicle-init    # 查车 + 生成 identifier/series
python geely_gric.py hb              # 心跳
python geely_gric.py status          # 车况
python geely_gric.py detail          # 详情（SIM/T-BOX）
python geely_gric.py favorite        # 车辆档案
python geely_gric.py msg             # 告警数
python geely_gric.py flash           # 闪灯寻车（控车）
python geely_gric.py control RHL rhl=light-flash
```

CI：GitHub → Actions → **geely-gric** → Run workflow。

---

## 附录：本次抓包关键样本（脱敏）

```
# 心跳（gric-g-api）
POST /ms-app-online-center/api/v2.0/app/hb
body {"deviceType":1,"enableWakeUp":false,"hbType":3,"ts":<ms>}
头 x-vehicle-identifier: <AES 密文，与本地计算一致>
头 x-vehicle-series: TDc0Mw==   # Base64("L743")
头 x-device-id: <真实设备号，≥18位>

# 车辆状态
GET /ms-vehicle-status/api/v2.0/vehicle/status/latest?latest=&target=new

# 远程控制（闪灯寻车，已证实）
POST /ms-remote-control/api/v1.0/remoteControl/control
body {"command":"","serviceId":"RHL","setting":{"serviceParameters":[{"key":"rhl","value":"light-flash"}]}}
resp {"code":"0","data":{"sessionId":"..."},"msg":"success"}

# 生物识别配置
GET app-services.lynkco.com.cn/auth/config/getBiometricConfig
resp {"code":"success","data":{"period":30,"touchIdIsOpen":true,"faceIdIsOpen":true}}
```

---

## 14. 车辆接口返回结构（实跑样例，脱敏）

> 来自 CI `geely-gric` 成功运行（`ec95cd3` 之后）。所有敏感值以占位/打码。`GwbClient.request()` 统一返回 `{"status": <HTTP状态码>, "data": <接口JSON>}`；接口 JSON 的一般外壳为 `{"code","data","debug":{"bizName","time","traceId"},"msg"}`。

### 14.1 auth 输出
```json
{"authorization": "***", "expireAt": "2026-11-17 12:48:09"}
```
打印行：`已使用 refreshToken 自动续期获取最新 token` → `accessCode 获取成功: <前8位>...` → `JWT 换取成功，有效期至 ...` → `已写回 authorization`。

### 14.2 vehicle-init 输出（脱敏）
```json
{
  "identifier": "f0e***s=",
  "series": "TDc0Mw==",
  "vin": "L6T***02",
  "seriesCode": "L743",
  "plateNo": "湘AG***86"
}
```

### 14.3 hb（心跳）
```json
{
  "status": 200,
  "data": {
    "code": "0",
    "data": {
      "hbType": 3,
      "iotStatusList": [],
      "qrvsVehicleStatusTs": -1,
      "remoteControlMsgList": [],
      "rvsVehicleStatusTs": 1794314844194
    },
    "debug": {"bizName": "", "time": 1794314892063, "traceId": ""},
    "msg": "success"
  }
}
```
> `remoteControlMsgList` 用于回报远程控制执行结果（异步回执）。

### 14.4 favorite（收藏车辆，返回数组，字段说明）
```json
{
  "status": 200,
  "data": {
    "code": "0",
    "data": [
      {
        "aiHost": "https://gric-aic-api.geely.com",
        "appearanceName": "哑光勃朗灰",
        "appinnerCode": "DS11-S1",
        "brandCode": "LYNKCO",
        "colorCode": "396",
        "colorName": "哑光勃朗灰",
        "defaultCarFlag": true,
        "defaultCarSetTime": "2026-08-29 18:51:00",
        "extAttrs": {"loopInterval": 7},
        "itemName": "07GT 四驱性能Ultra",
        "modelCode": "DS11-S1",
        "modelName": "DS11-S1",
        "nickName": "Fany-07GT",
        "ownerFlag": true,
        "plateNo": "湘AG***86",
        "pno18": "DSS***V6",
        "relationState": 1,
        "secondCategoryName": "07GT",
        "seriesCode": "L743",
        "seriesCodeVs": "DS11-S1",
        "seriesName": "领克07",
        "seriesNameVs": "DS11-S1",
        "skuImg": "https://ossdopimg.lynkco.com/....png",
        "tenantId": "LYNKCO",
        "tspFramework": 3,
        "tspHost": "https://gric-g-api.geely.com",
        "tspPlatform": 3,
        "userId": "283***96",
        "vin": "L6T***02",
        "webHost": "https://gric-g-static.geely.com",
        "wheelPosition": "0"
      }
    ],
    "debug": {"bizName": "ms-vehicle-core", "time": 1794314893226, "traceId": "..."},
    "msg": "success"
  }
}
```
> 关键：`tspHost` 直接给出了这台车的业务网关就是 `gric-g-api.geely.com`（与 §2.3 一致）；`ownerFlag/defaultCarFlag` 判断车主/默认车。

### 14.5 detail（车辆详情 / SIM-TBOX）
```json
{
  "status": 200,
  "data": {
    "code": "0",
    "data": {
      "iccid": "<masked>",
      "imsi": "<masked>",
      "msisdn": "<masked>",
      "tboxId": "<masked>",
      "vin": "L6T***02"
    },
    "debug": {"bizName": "ms-app-adapter", "time": 1794314894485, "traceId": "..."},
    "msg": "success"
  }
}
```

### 14.6 status（实时车况，结构分区）
```json
{
  "status": 200,
  "data": {
    "code": "0",
    "data": {
      "additionalMaintenanceStatus": {
        "tyreStatusDriver": "..", "tyreStatusDriverRear": "..", "tyreStatusPassenger": "..",
        "tyreStatusPassengerRear": "..", "tyreTempDriver": "..", "tyreTemp...": "..",
        "engineOilHealthLevel": "..", "odometer": "..", "daysToService": "..",
        "distanceToService": "..", "brakeFluidLevelStatus": "..", "voltage": ".."
      },
      "basic": {
        "distanceToEmpty": "..", "combinedDistanceToEmpty": "..", "distanceToEmptyOnBatteryOnly": "..",
        "engineStatus": "..", "speed": "..", "usageMode": "..", "prkgCameraActive": "true",
        "position": {"latitude": "<masked>", "longitude": "<masked>", "altitude": "..",
                     "direction": "..", "marsCoordinates": "false", "posCanBeTrusted": "true"}
      },
      "bigdataVehicleStatus": { "distanceToEmptyOnBatteryOnly": "..", "voltage": "..", "...": ".." },
      "climateStatus": {
        "interiorTemp": "..", "exteriorTemp": "..", "drvHeatLv": "..", "drvVentSts": "..",
        "steerWhlHeatingSts": "..", "winStatusDriver": "..", "winPosDriver": "..",
        "carFridgeStatus": {"...": ".."}, "carFreezerStatus": {"...": ".."}, "...": ".."
      },
      "drivingBehaviourStatus": { "gearAutoStatus": "..", "transimissionGearPostion": ".." },
      "drivingSafetyStatus": {
        "centralLockingStatus": "..", "doorLockStatusDriver": "..", "doorOpenStatusDriver": "..",
        "trunkLockStatus": "..", "trunkOpenStatus": "..", "electricParkBrakeStatus": "..",
        "seatBeltStatusDriver": "..", "...": ".."
      },
      "electricVehicleStatus": {
        "chargeLevel": "..", "chargeSts": "..", "chargeIAct": "..", "chargeUAct": "..",
        "dcChargeIAct": "..", "timeToFullyCharged": "..", "isCharging": "..", "...": ".."
      },
      "pollutionStatus": { "interiorPM25": "..", "exteriorPM25": "..", "relHumSts": ".." },
      "runningStatus": { "aveFuelConsumption": "..", "fuelLevel": "..", "fuelLevelPct": "..",
                         "avgSpeed": "..", "tripMeter1": "..", "tripMeter2": "..", "...": ".." },
      "temStatus": { "powerMode": "..", "rvsEnable": ".." },
      "testCall": true,
      "updateTime": 1794314844194
    },
    "debug": {"bizName": "ms-app-adapter", "time": 1794314895839, "traceId": "..."},
    "msg": "success"
  }
}
```
> 说明：字段极多，这里只列分区结构与代表字段；`basic.position` 是**实时经纬度（已脱敏）**；`basic.prkgCameraActive` 反映泊车摄像头状态；胎压/门窗锁/充电/油耗等在对应分区。

### 14.7 msg（告警数）
```json
{
  "status": 200,
  "data": {
    "code": "0",
    "data": {"alarmsTotal": 6},
    "debug": {"bizName": "", "time": 1794314897072, "traceId": ""},
    "msg": "success"
  }
}
```

### 14.8 实跑结果汇总（本次 Actions）
| 动作 | 结果 |
|---|---|
| auth | ✅ 续期 + accessCode + JWT 全成功 |
| vehicle-init | ✅ 湘AG***86 / DS11-S1 / L743 |
| hb | ✅ `code 0` |
| favorite | ✅ `code 0`（ownerFlag=true） |
| detail | ✅ `code 0`（ICCID/IMSI/T-BOX） |
| status | ✅ `code 0`（完整车况） |
| msg | ✅ `code 0`（alarmsTotal=6） |

---

## 15. 摄像头 / 哨兵 / 实时画面 —— 调研与结论

### 15.1 领克现状（用户确认）
- 领克 App **可以查看「泊车图片」**，**可以查看「哨兵视频」**，但**只显示高风险事件**（低风险事件 App 不展示）。
- 这些是**录制的图片/片段**（HTTP 可下载），与「**实时画面**」是两回事。
- 目标诉求：**直接调用摄像头看当前实时画面**（= 实时视频流）。

### 15.2 吉利系"看车摄像头"= 哨兵监控服务（分品牌部署）

| 项目 | 品牌/地区 | 关键发现 |
|---|---|---|
| [borconi/openzeekr](https://github.com/borconi/openzeekr) | 极氪（海外） | 逆向出 `sentinel-monitoring-service`（**CN-only**）：<br>`GET /sentinel-monitoring-service/api/v1/pic/list`（泊车快照）<br>`GET /sentinel-monitoring-service/api/v1/alarm/event/query`（哨兵事件，返回 `alarmImageUrl`/`alarmVideoUrl`）<br>`POST .../alarm/event/launchUploadVideo`（取录像地址）<br>`POST .../alarm/live/app/getToken` + `GET .../alarm/live/app/launchLive`（实时 token，返回 `appId/roomId/accessToken/joinUserId`）<br>文档：**海外网关未部署（404 `00A01`）；实时画面还需 RTC 厂商 SDK（作者未识别）** |
| [suyunkai/geely-galaxy-assistant](https://github.com/suyunkai/geely-galaxy-assistant) | 吉利银河 E5（国内） | 已实现 **`photo`=获取驻车图片**：`galaxy-vc.geely.com/vc/app/v1/sentinel/queryPhotoList`；**哨兵模式开关**（`opensentry/closesentry`）；大量车控（`galaxy-vc.geely.com/vc/app/v1/vehicle/control/{door,window,climate,search,temperature,...}`）。⚠️基于**银河账号+`galaxy-*` 网关**，不支持领克；经纬度加密未解决 |
| [Fryyyyy/zeekr_ev_api](https://github.com/Fryyyyy/zeekr_ev_api) + [zeekr_homeassistant](https://github.com/Fryyyyy/zeekr_homeassistant) | 极氪（EU） | 云 API + HA；**有哨兵模式开关**、车况/车控，**无摄像头画面** |
| [dts88/zeekr-shortcut-car](https://github.com/dts88/zeekr-shortcut-car) | 极氪 7X | **车机端**「360环视行车记录」App（EVCam 分支），用 **Android Camera2 本地读环视摄像头**——本地预览/录制，非远程 |
| [Mikhail2092/geely-third-party-software-analysis](https://github.com/Mikhail2092/geely-third-party-software-analysis) | 银河 L7/L6/EX5 等 | 车机内置 App 的**静态分析资料**（信息娱乐/协议），不涉及云摄像头 |

### 15.3 对领克网关的实测（2026-11-10）
用本车真实凭据、对领克国内网关只读请求"极氪那套"服务名：
```
gric-g-api.geely.com  /sentinel-monitoring-service/api/v1/pic/list  -> 404 {"error_msg":"404 Route Not Found"}
gric-hf-api.geely.com /sentinel-monitoring-service/api/v1/pic/list  -> 404 Route Not Found
```
`404 Route Not Found` 是**网关层没有这条路由**（非鉴权失败）。说明极氪的服务名/主机不适用于领克，**领克用的是自己的一套（名字/主机不同）**。→ 需用**领克 App 抓包**定位领克的哨兵/摄像头真实接口（用户已确认 App 有泊车图片/哨兵视频）。

### 15.4 实时画面的技术本质与卡点
1. **信令**：先调接口拿加入房间参数，形如 `{appId, roomId, accessToken, joinUserId/joinRoomId, liveDuration}`；
2. **RTC 拉流**：用对应厂商 SDK 加入房间渲染视频。

**识别厂商（看信令里的 `appId` 形态）：**
- 32 位十六进制 → 多为**声网 Agora**（channel=roomId, uid=joinUserId, token=accessToken）
- 纯数字 `appId/sdkAppId` + `userId/userSig` → **腾讯 TRTC** 或 **即构 ZEGO**
- `wss:// + SDP` → **WebRTC**（浏览器 / aiortc 直接拉）
- `.flv / .m3u8 / rtmp://` → **直接 ffmpeg 拉**

> 参考：极氪 `liveToken` 返回模型为 `SentryLiveTokenResp{accessToken, expireAt, issuedAt, joinRoomId, joinUserId}` + `SentryLaunchLiveResp{appId, roomId, liveDuration}`——领克大概率同构（同一平台供应商）。

### 15.5 抓包要点（实时画面）
在 Reqable 打开「实时画面」看 3~5 秒并保存 HAR，重点保留：
1. **点开瞬间**返回 `appId/roomId/token/userId` 的那条 HTTP 的**完整请求头 + 完整响应体**（识别 RTC 的钥匙）；
2. 之后的**媒体长连接**：`wss://`、`rtmp://`、`.flv`、`.m3u8`、`webrtc/SDP`，或 `CONNECT ip:端口`；
3. 领克哨兵/摄像头接口的**主机与路径前缀**（确定是否可复用 GRIC 签名/主机）。

### 15.6 可行性结论与路线

| 目标 | 难度 | 结论 |
|---|---|---|
| 泊车图片 / 哨兵录像 | 🟢 简单 | 抓到领克真实接口后，HTTP 下载即可做成 `photo` / `clip` 命令 |
| **实时画面（当前诉求）** | 🟠 中偏难 | 取决于 RTC 厂商：Agora/ZEGO/TRTC/WebRTC/FLV 可尝试在 PC 拉流截帧；若为私有加密协议则难 |

**备选（非云）**：车机本地取流——把 App 装到车机用 `Camera2` 读环视（参考 `dts88/zeekr-shortcut-car`），属本地预览/录制，需要车机可安装/ADB 与相机权限。

### 15.7 限制
- 实时流可能有**时长限制**（`liveDuration`）、需车辆**在线/被唤醒**、可能有**加密或厂商私有协议**。
- 能做到"**拉下当前一帧 / 短时看几秒**"即视为打通；持续高清直播不保证。
- 跨品牌网关不可用（领克账号/VIN 不在银河/极氪体系）。
