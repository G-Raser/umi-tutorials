# 从 MCP 到 Android：搭建一个受控的 AI 手机 Bridge

**Umi & CatTea**

> 本文整理自 Umi Mobile 的实际实现，目标是给 personal AI / AI companion / agent 项目提供一套可复用的 Android bridge 工程方案。
>
> 重点放在架构、协议、安全边界、实现顺序与测试方法。生产仓库、真实凭据、私人域名和完整私有 adapter 不公开。

---

## 1. 目标与边界

目标是让 AI 通过一组明确工具访问设备持有人自己的 Android 手机，例如：

- 读取当前前台 App；
- 读取 Accessibility UI tree；
- 截图；
- 点击、滑动、返回、Home；
- 向普通输入框写入文本；
- 按 package name 打开 App；
- 按需扩展 APK 安装、BLE、过滤后的 logcat、后台语音等原生能力。

核心要求：

1. 手机主动向服务器建立连接，不开放手机公网端口。
2. AI 侧只看到稳定的语义工具，不直接依赖 Android 实现细节。
3. Android 端每项能力都显式注册。
4. 密码、锁屏、支付、生物识别、系统安全确认等受保护流程不绕过。
5. 高权限能力默认按需启用，和普通 UI 自动化分层。
6. 设备持有人可以随时暂停 bridge 或撤销权限。

非目标：

- 静默远程控制陌生设备；
- 绕过 Android 安全机制；
- 无提示读取密码字段；
- 静默安装 APK；
- 用 ADB 作为长期运行依赖；
- 把整个业务后端塞进 Android App。

---

## 2. 总体架构

推荐结构：

```text
AI / Agent
    │
    │ MCP / tool calls
    ▼
Server-side Bridge
    │
    │ WSS / WebSocket
    ▼
Android Companion App
    │
    ├─ AccessibilityService
    ├─ Foreground Service
    ├─ Android native APIs
    └─ optional device-only capabilities
```

各层职责：

| 层 | 负责什么 |
| --- | --- |
| AI / Agent | 决策、上下文、任务规划 |
| MCP / Tool layer | 提供稳定语义接口 |
| Server-side Bridge | 设备在线状态、命令转发、超时、配对 |
| Android App | 真正执行手机侧能力 |
| Android OS | 权限、安全确认、受保护系统流程 |

### 关键设计：手机主动连接服务器

连接方向建议固定为：

```text
Android App → WSS endpoint → Bridge
```

不要让服务器尝试直接连接手机。

这样可以自然处理：

- NAT；
- 校园网 / 家庭 Wi-Fi；
- 4G / 5G；
- 手机公网 IP 变化；
- Wi-Fi 与蜂窝网络切换。

客户端需要实现：

```text
connect
→ heartbeat / ping
→ disconnect
→ exponential backoff
→ reconnect
```

一个简单的退避序列可以是：

```text
2s → 4s → 8s → 16s → 32s → 60s
```

连接成功后重置。

---

## 3. 推荐的工程结构

一个最小实现可以拆成：

```text
android-bridge/
├── app/
│   └── Android client
├── bridge/
│   ├── src/
│   │   ├── server.js
│   │   └── device-hub.js
│   └── package.json
└── docs/
```

Android 端建议至少拆出：

```text
BridgeClient
ConfigStore
PairingClient
PhoneAccessibilityService
```

按需要再增加：

```text
ApkInstaller
LogcatReader
BleInspector
CallService
...
```

不要把所有 command 逻辑都塞进一个 Activity。

---

## 4. Android 基础权限

最小 UI 自动化通常需要：

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

核心执行层通过 `AccessibilityService`：

```xml
<service
    android:name=".PhoneAccessibilityService"
    android:exported="true"
    android:permission="android.permission.BIND_ACCESSIBILITY_SERVICE">

    <intent-filter>
        <action android:name="android.accessibilityservice.AccessibilityService" />
    </intent-filter>

    <meta-data
        android:name="android.accessibilityservice"
        android:resource="@xml/accessibility_service_config" />
</service>
```

后续能力再按需加权限，例如：

```text
RECORD_AUDIO
FOREGROUND_SERVICE
FOREGROUND_SERVICE_MICROPHONE
POST_NOTIFICATIONS
REQUEST_INSTALL_PACKAGES
BLUETOOTH_SCAN
BLUETOOTH_CONNECT
BLUETOOTH_ADVERTISE
READ_LOGS
```

不要第一版就把所有权限全部申请。

---

## 5. 连接配置与本地状态

Android 端至少需要保存：

```text
endpoint
device_token
device_id
armed
```

其中：

- `endpoint`：WSS 地址；
- `device_token`：长期设备认证；
- `device_id`：设备本地生成的稳定 UUID；
- `armed`：是否允许 bridge 工作。

建议保存到私有 `SharedPreferences` 或更安全的本地存储中。

示意：

```java
public static boolean armed(Context context) {
    return prefs(context).getBoolean("armed", false);
}
```

服务启动时：

```text
armed = true  → 建立 WebSocket
armed = false → 主动断开并停止接收命令
```

App UI 里应该始终有一个明显的暂停 / 开启入口。

---

## 6. 首次配对

不要要求使用者手抄完整 endpoint 和 token。

推荐：

```text
server 生成短时 pairing code
        ↓
手机输入 code
        ↓
POST /pair/exchange
        ↓
返回 endpoint + device token
        ↓
手机本地保存
```

请求：

```json
{
  "code": "12345678",
  "device_id": "generated-device-id"
}
```

响应：

```json
{
  "ok": true,
  "endpoint": "wss://example.com/device",
  "token": "long-lived-device-token"
}
```

配对码建议：

- 短时有效；
- 单次使用；
- 服务端只存 hash；
- 使用后立即失效。

长期运行只依赖 device token，不重复使用 pairing code。

---

## 7. WebSocket 设备协议

### 7.1 设备上线

连接建立后，Android 端主动发 `hello`：

```json
{
  "type": "hello",
  "device_id": "device-uuid",
  "package": "current.foreground.package",
  "capabilities": [
    "status",
    "ui_tree",
    "screenshot",
    "tap",
    "swipe",
    "type",
    "back",
    "home",
    "open_app"
  ]
}
```

服务器登记：

```text
device_id
→ websocket
→ capabilities
→ connected_at
→ last_seen_at
```

### 7.2 下发命令

Bridge 发：

```json
{
  "id": "command-id",
  "action": "swipe",
  "params": {
    "x1": 500,
    "y1": 1700,
    "x2": 500,
    "y2": 700,
    "duration_ms": 350
  }
}
```

Android 返回：

```json
{
  "type": "result",
  "id": "command-id",
  "ok": true,
  "result": {
    "performed": true
  }
}
```

失败时：

```json
{
  "type": "result",
  "id": "command-id",
  "ok": false,
  "error": "focused node is a password field"
}
```

### 7.3 command timeout

服务器必须给命令设置 timeout。

例如：

```text
send command
→ store pending[id]
→ wait result
→ timeout
→ reject
→ delete pending[id]
```

手机断线时，也应该立即 reject 对应设备的 pending commands。

---

## 8. DeviceHub：把 WebSocket 和上层工具隔开

服务器侧建议做一个单独的 `DeviceHub`。

职责：

- 管理已连接设备；
- 按 `device_id` 选择连接；
- 下发 command；
- 保存 pending promise；
- result 回来后 resolve；
- 断线时清理状态；
- 对上层隐藏 WebSocket 细节。

接口可以非常小：

```js
hub.listDevices()
hub.command(action, params, deviceId)
```

伪代码：

```js
async function command(action, params, deviceId) {
  const device = getDevice(deviceId)
  if (!device) throw new Error("device offline")

  const id = crypto.randomUUID()

  return await new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      pending.delete(id)
      reject(new Error("command timeout"))
    }, COMMAND_TIMEOUT)

    pending.set(id, { resolve, reject, timer, deviceId })

    device.ws.send(JSON.stringify({
      id,
      action,
      params
    }))
  })
}
```

这一层稳定以后，上面接 MCP、REST、CLI 都很容易。

---

## 9. MCP 工具层

AI 不应该直接生成 WebSocket command JSON。

建议在 MCP 层提供稳定工具，例如：

```text
phone_list_devices
phone_status
phone_ui_tree
phone_screenshot
phone_tap
phone_swipe
phone_type
phone_back
phone_home
phone_open_app
```

例如 `phone_swipe`：

```js
server.registerTool("phone_swipe", {
  inputSchema: {
    x1: z.number().nonnegative(),
    y1: z.number().nonnegative(),
    x2: z.number().nonnegative(),
    y2: z.number().nonnegative(),
    duration_ms: z.number().int().min(100).max(2000).optional(),
    device_id: z.string().optional()
  }
}, async (args) => {
  return await hub.command("swipe", args, args.device_id)
})
```

这样 Android 实现未来换掉时，AI 侧接口不需要一起变化。

---

## 10. AccessibilityService 的最小能力

第一版建议只实现：

```text
status
ui_tree
screenshot
tap
swipe
back
home
open_app
```

`type` 可以稍后加，因为输入字段需要额外安全判断。

### 10.1 UI tree

遍历节点时建议返回：

```json
{
  "class": "android.widget.Button",
  "text": "Submit",
  "content_description": "",
  "clickable": true,
  "editable": false,
  "password": false,
  "bounds": [40, 1200, 1040, 1320]
}
```

密码字段：

- 不返回真实文本；
- 不允许远程写入。

### 10.2 screenshot

Android 11+ 可以由 AccessibilityService 使用截图 API。

返回时建议：

```json
{
  "mime": "image/png",
  "base64": "..."
}
```

MCP 层再转换成 image content。

### 10.3 tap / swipe

用 `GestureDescription` 执行。

所有坐标先校验：

```text
0 <= x < screen_width
0 <= y < screen_height
```

### 10.4 open_app

通过 package name 查 launch intent。

不要让 AI 侧依赖桌面图标位置。

---

## 11. UI tree 和 screenshot 应该并存

只给一种通常不够。

### UI tree 优点

- 文本结构清楚；
- token 成本低；
- 容易定位 editable / clickable node；
- 适合稳定表单。

### screenshot 优点

- 能看到自绘 UI；
- 能看到视觉层级；
- 能处理 accessibility metadata 很差的 App；
- 对弹窗、图标、游戏 UI 更有效。

推荐任务流：

```text
status
→ ui_tree
→ 必要时 screenshot
→ action
→ 再次 inspect
```

不要默认每一步都截图。

---

## 12. 输入文本的安全限制

远程输入至少要拒绝：

- password node；
- 系统锁屏密码；
- PIN；
- 生物识别；
- 支付确认；
- 受保护系统弹窗。

例如：

```text
focused editable node
        ↓
isPassword?
  ├─ yes → reject
  └─ no  → ACTION_SET_TEXT
```

这类限制建议放在 Android 端，而不是只靠 AI prompt。

---

## 13. 高权限能力要单独分层

项目变大后，能力可以粗分：

### Level A：普通 UI 能力

```text
status
ui_tree
screenshot
tap
swipe
back
home
open_app
```

### Level B：设备持有人显式授权的敏感能力

```text
microphone
notification
Bluetooth
APK installer
```

### Level C：开发 / 调试 bootstrap 能力

```text
READ_LOGS
special debug permissions
```

不要让 Level C 成为默认安装后的常态。

---

## 14. 可选扩展：APK 安装链路

如果 AI 会开发 Android App，可以加一个私有 artifact pipeline：

```text
build APK
→ publish artifact
→ bridge returns artifact_id
→ phone downloads
→ verify SHA-256
→ verify package name
→ launch system installer
→ owner confirms
```

Android 端建议校验：

1. URL 必须是可信 HTTPS host；
2. APK 大小上限；
3. SHA-256；
4. package name；
5. 最后只拉起系统 installer，不静默安装。

MCP 工具可以只接受：

```text
artifact_id
device_id?
```

不要直接允许 AI 提供任意公网 APK URL。

---

## 15. 可选扩展：过滤后的 logcat

如果需要调试手机侧协议或 App 行为，可以增加 logcat reader。

这个能力风险明显高于截图和 UI tree。

建议强制：

- 权限由设备持有人手动 bootstrap；
- 每次读取必须提供 literal filters；
- 限制 source lines 和 return lines；
- 手机端先脱敏，再返回 server；
- 默认只读；
- 不提供 clear logcat。

例如：

```text
phone_logcat_mark()
→ reproduce problem
→ phone_logcat_read(filters=["keyword"])
```

脱敏至少覆盖：

```text
Authorization: Bearer ...
access_token
refresh_token
token
cookie
password
session id
```

即使服务端也有过滤，首层脱敏仍建议放在手机端。

---

## 16. 可选扩展：BLE

BLE 适合作为独立 capability，而不是塞进 UI automation。

可以拆成：

```text
ble_scan
ble_connected_gatt
ble_advertise_status
ble_advertise_uuid
ble_advertise_stop
```

读操作与写操作分开暴露。

如果只是协议分析，优先实现 read-only inspect / preview。

---

## 17. 可选扩展：后台语音

网页 / PWA 在后台和锁屏场景下通常无法稳定承担：

- 长时间麦克风采集；
- 持续媒体播放；
- 音频焦点；
- 进程保活；
- 通话通知。

这类能力更适合下沉到 Android 原生 `ForegroundService`。

推荐结构：

```text
Call entry
   ↓
Foreground Service
   ├─ AudioRecord
   ├─ audio playback
   ├─ MediaSession
   ├─ ongoing notification
   └─ WakeLock
```

需要的典型权限：

```text
RECORD_AUDIO
FOREGROUND_SERVICE
FOREGROUND_SERVICE_MICROPHONE
FOREGROUND_SERVICE_MEDIA_PLAYBACK
POST_NOTIFICATIONS
WAKE_LOCK
```

后台通话应和普通 phone bridge 解耦，避免一个 Service 同时承担过多生命周期职责。

---

## 18. CatTea / Home 这一类私人 adapter 怎么接

通用 bridge 到这里已经足够。

私人系统通常还会再有一层 adapter：

```text
personal AI context
        ↓
owner / session policy
        ↓
phone tools
```

例如：

- 当前操作属于哪个长期会话；
- 是否允许主动操作；
- 哪些动作必须先询问；
- 电话与聊天 session 怎么对应；
- 结果怎样写回原对话。

这些属于 personal AI 的业务层，不建议塞进 Android transport。

公开教程可以保留这层结构，但生产 token、真实 session id、私人 prompt、内部 endpoint 应继续私有。

---

## 19. 安全模型

最低建议：

### Android 端

- 明确 pause / arm 开关；
- 不读取密码字段；
- 不填写密码字段；
- 不绕过锁屏；
- 不处理生物识别；
- 不确认支付；
- 不绕系统安装确认；
- 敏感能力单独申请权限；
- persistent notification 显示 bridge 状态。

### Server 端

- WSS；
- device token；
- command timeout；
- pairing code 单次使用；
- 不把 secret 写进 Git；
- artifact 下载需要认证；
- 高风险工具标记为写操作；
- 日志避免记录完整敏感 payload。

### Agent 端

- 先 inspect，后 action；
- 批量操作小步执行；
- destructive action 明确确认；
- 不根据旧截图盲点坐标。

安全限制最好同时存在于 Android、Bridge、Tool schema 三层。

---

## 20. 部署建议

生产环境推荐：

```text
Public HTTPS/WSS endpoint
        ↓
reverse proxy / tunnel
        ↓
bridge bound to localhost
        ↓
WebSocket device connections
```

Bridge 自身可以只监听：

```text
127.0.0.1
```

公网 TLS 交给：

- Caddy；
- Nginx；
- Cloudflare Tunnel；
- 其他你自己的入口层。

Android 端只保存公开 WSS endpoint。

---

## 21. 第一版实现顺序

推荐按这个顺序做。

### Step 1：Bridge health

实现：

```text
GET /health
```

确认服务正常。

### Step 2：手机建立 WebSocket

先只做：

```text
hello
device registry
reconnect
```

### Step 3：phone_status

这是第一条完整 E2E：

```text
AI
→ MCP
→ Bridge
→ WebSocket
→ Android
→ result
→ AI
```

### Step 4：UI inspection

加：

```text
ui_tree
screenshot
```

### Step 5：UI action

加：

```text
tap
swipe
back
home
open_app
```

### Step 6：type

完成 password-field guard 后再加。

### Step 7：pairing

把开发期手工 endpoint/token 配置替换为短码 exchange。

### Step 8：owner-visible controls

补：

- arm / pause；
- 状态通知；
- bridge state；
- connected / reconnecting / paused。

### Step 9：再扩展 native capabilities

只按真实需求增加。

---

## 22. 最小测试矩阵

### 连接

- [ ] 首次配对成功；
- [ ] token 错误时拒绝连接；
- [ ] Wi-Fi → 5G 后自动重连；
- [ ] server 重启后客户端自动恢复；
- [ ] pause 后不再连接；
- [ ] resume 后重新上线。

### 读取

- [ ] status 返回前台 package；
- [ ] ui_tree 不返回密码文本；
- [ ] screenshot 正常；
- [ ] App 自绘界面时 screenshot 仍可用。

### 操作

- [ ] tap；
- [ ] swipe；
- [ ] back；
- [ ] home；
- [ ] open_app；
- [ ] 普通输入框 type；
- [ ] password field type 被拒绝。

### Bridge

- [ ] command id 唯一；
- [ ] timeout 会清理 pending；
- [ ] 设备断线时 pending 立即失败；
- [ ] 多设备时 device_id 路由正确。

### 敏感能力

- [ ] 未授权时明确失败；
- [ ] 授权后只开放对应 capability；
- [ ] 撤销权限后正确降级；
- [ ] 日志脱敏在手机端生效。

---

## 23. 常见问题

### Q1：必须用 MCP 吗？

不必须。

`DeviceHub` 上面可以接：

- MCP；
- REST；
- CLI；
- Web UI；
- 自己的 agent protocol。

MCP 只是当前 personal AI 场景里比较方便的一层。

### Q2：必须用 AccessibilityService 吗？

如果只需要：

- 通知；
- 蓝牙；
- 传感器；
- 文件；
- 麦克风；

未必需要。

如果需要跨 App 的 UI inspection / tap / swipe，它比较合适。

### Q3：为什么不用 ADB 常驻？

ADB 适合开发，但长期 companion 使用体验不稳定，也不自然。

更好的结构是：

```text
ADB = bootstrap / debug
Android App = runtime
```

### Q4：为什么不用自动化框架直接跑在电脑？

可以，但那会让“手机是否在线、网络是否切换、USB 是否连着”变成额外依赖。

常驻 Android companion 更适合作为长期设备能力层。

### Q5：一定要 VPS 吗？

不一定。

只要中间层能被：

- Android App 访问；
- AI / MCP 访问；

即可。

可以是：

- VPS；
- 家中服务器；
- Tailscale 内网服务；
- cloud function + persistent WebSocket backend；
- 其他稳定 endpoint。

---

## 24. 公开版与生产版建议分开

如果你准备把自己的实现公开，建议保留：

- protocol；
- capability interface；
- example MCP tools；
- pairing flow；
- security model；
- test strategy。

保持私有：

- token；
- endpoint；
- 设备 ID；
- 真实账号信息；
- 私人 prompt；
- 生产 adapter 的敏感部分；
- signing key；
- runtime logs；
- artifact store 内容。

不要为了做教程把生产仓库本身改成 public。

---

## 25. 一个最小可行版本

如果只想验证这个方案，第一版做到这些就够：

```text
Android:
- AccessibilityService
- BridgeClient
- ConfigStore

Server:
- WebSocketServer
- DeviceHub
- phone_status
- phone_ui_tree
- phone_screenshot
- phone_tap
- phone_swipe

Security:
- WSS
- device token
- owner pause switch
- password redaction
- command timeout
```

这已经足够形成一个真正可用的 AI ↔ Android 闭环。

后续能力应该按实际需求继续扩展，而不是一次性设计一个“万能手机 agent”。

---

## Authors

**Umi & CatTea**

本教程正文沿用仓库根目录的 **CC BY-NC-SA 4.0** 许可与署名规则。
