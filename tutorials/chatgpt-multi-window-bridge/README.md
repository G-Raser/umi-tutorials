# ChatGPT 多窗口 Bridge：会话隔离、Listener 接管与顺序投递

**Umi & CatTea**

> 这篇不从“怎么把浏览器和本地后端连起来”开始讲。
>
> 默认你已经有一个能工作的 Bridge：本地前端可以把消息送进某个已登录的 ChatGPT / Work 对话，也能把回复回写到自己的前端。
>
> 这里专门讲单窗口 Bridge 往后最容易变复杂的一层：**多个前端会话如何稳定地绑定到不同官方对话，并避免串窗、抢消息、旧 listener 抢活、同一会话乱序，以及切换宿主后旧请求继续投递。**

---

## 1. 为什么“能发一条消息”之后，事情才刚开始

单窗口 Bridge 很容易做出 MVP：

```text
Frontend
  ↓
Local backend
  ↓
Bridge queue
  ↓
One ChatGPT tab
  ↓
reply callback
  ↓
Frontend
```

真正开始同时使用多个会话以后，会马上遇到另一组问题：

- A 前端窗口的消息会不会被 B 官方窗口拿走？
- 两个 listener 同时挂在同一条通道上时，谁有资格 claim？
- 一个旧 listener 刷新前还活着，新 listener 打开后，旧的会不会继续抢请求？
- 同一个本地会话连续发两条消息，第二条会不会在第一条回复之前先进入官方对话？
- 两个不同本地会话能不能真正并行？
- 从 Chat 切到 Work 时，旧通道里还没发送的消息怎么办？
- 浏览器前端断开后，官方回复已经回来，结果会不会丢？

这些问题不能只靠“多开几个 tab”解决。

我们最后采用的是一个比较清楚的分层：

```text
session_id           = 本地会话身份
binding_id           = 这一个本地会话对应的 Bridge 通道
listener_token       = 当前有资格消费该 binding 的 listener 实例
listener_generation  = listener 接管代数
request_id           = 一次具体消息投递
```

这几个 ID 不要合并成一个概念。

---

## 2. 核心原则：一个本地会话，持久化绑定一条专用 lane

最重要的一条规则是：

> **不要让所有本地会话共享一个全局 Bridge binding。**

共享全局 binding 在单窗口时看起来完全正常，一旦多窗并行，就很容易产生串窗。

更稳妥的做法是：创建一个需要走官方 Bridge 的本地会话时，由后端生成一个 opaque binding，例如：

```text
chat-p-a13f9c22d5e1
work-p-47b2e93fd821
```

前缀可以表示宿主类型，后面的随机部分只负责区分 lane。

示意代码：

```python
from uuid import uuid4

def new_binding(host_kind: str) -> str:
    if host_kind not in {"chat", "work"}:
        raise ValueError("invalid host kind")
    return f"{host_kind}-p-{uuid4().hex[:12]}"
```

生成之后，把它**持久化到本地 session**：

```json
{
  "id": "local-session-001",
  "transport": "chatgpt_bridge",
  "bridge": {
    "host_kind": "chat",
    "binding_id": "chat-p-a13f9c22d5e1",
    "dedicated": true
  }
}
```

之后这个会话发出的所有 Bridge 请求，都必须使用已经保存的 `binding_id`。

### 一个重要边界

专用 binding 最好由后端生成和管理。

前端可以把当前 session 已经持有的 binding 带回后端，但不应该随便自己制造一个新的 dedicated binding。否则前端刷新、旧缓存、恶意请求或 UI bug 都可能悄悄改变会话的配对关系。

伪代码：

```python
if session.bridge.binding_id:
    assert request.binding_id in {None, session.bridge.binding_id}
    return session.bridge.binding_id

if request.binding_id_is_dedicated:
    raise Error("dedicated binding must be server-managed")
```

---

## 3. `session_id` 和 `binding_id` 为什么必须分开

这两个东西看起来都像“窗口 ID”，但职责完全不同。

`session_id` 属于你自己的应用。它决定：

- 本地聊天记录存到哪里；
- 哪些消息属于同一段历史；
- 哪个前端窗口正在读哪份 ledger。

`binding_id` 属于 Bridge transport。它决定：

- 哪个 listener 能看到这批请求；
- 这批请求应该进入哪一个官方 ChatGPT / Work 对话。

因此关系更接近：

```text
local session 1 ── binding A ── official ChatGPT tab A
local session 2 ── binding B ── official ChatGPT tab B
local session 3 ── binding C ── official Work tab C
```

不要用官方 conversation id 直接替代自己的 `session_id`，也不要让所有 session 共用一个 `binding_id`。

把本地会话身份和传输 lane 解耦后，后面做切换、归档、重连、迁移都会轻很多。

---

## 4. Listener 也需要“唯一现任”：token + generation

只做到“一会话一 binding”仍然不够。

现实里经常会出现：

1. 旧官方页面上已经开着一个 listener；
2. 页面刷新或重新打开；
3. 新 listener 也开始轮询同一个 binding；
4. 两边都认为自己有资格 claim。

如果只判断：

```text
binding_id 相同
```

那么旧 listener 和新 listener 都可能继续抢请求。

我们的处理方式是：**每次打开新的 listener，都激活一个新的 listener token，并递增 generation。**

表里可以保存：

```text
binding_id
active_listener_token
listener_generation
last_seen_at
listener_paused
```

激活新 listener 时：

```python
listener_token = uuid4().hex

UPDATE bridge_bindings
SET active_listener_token = listener_token,
    listener_generation = listener_generation + 1,
    listener_paused = 0
WHERE binding_id = ?
```

之后 claim 时，不只校验 binding，还校验 token：

```python
if request.listener_token != binding.active_listener_token:
    return "listener_superseded"
```

这样新 listener 一接管，旧 listener 就算还在后台轮询，也只能得到“你已经被替代”，不能继续拿请求。

### generation 有什么用

`listener_generation` 不是必须参与每一次鉴权，但非常适合做：

- UI 显示；
- 调试；
- 日志追踪；
- 判断“这个 listener 是第几次重新打开的”；
- 分析 stale listener 问题。

它让“谁是现任”从一个模糊状态变成可观察状态。

---

## 5. 真正的顺序保证：同一 session 串行，不同 session 并行

这是多窗口里最关键的一层。

如果只按 binding 取最早 pending 请求：

```sql
SELECT *
FROM bridge_requests
WHERE binding_id = ?
  AND status = 'pending'
ORDER BY created_at
LIMIT 1;
```

看起来已经有序了，但仍然有一个问题：

第一条请求一旦被 claim，它就不再是 pending；下一次轮询马上就可能把第二条也 claim 出去。

于是同一个官方对话里可能出现：

```text
request 1 还在生成
request 2 已经又被提交
```

这会破坏对话顺序。

### 我们最终使用的约束

候选请求除了满足：

```text
binding_id 相同
status = pending
```

还必须满足：

> **这个 candidate 所属的 session 当前不存在 claimed / dispatched 请求。**

伪 SQL：

```sql
SELECT candidate.*
FROM bridge_requests AS candidate
WHERE candidate.binding_id = ?
  AND candidate.status = 'pending'
  AND NOT EXISTS (
    SELECT 1
    FROM bridge_requests AS active
    WHERE active.session_id = candidate.session_id
      AND active.status IN ('claimed', 'dispatched')
  )
ORDER BY candidate.created_at
LIMIT 1;
```

这个约束带来的效果非常重要：

```text
同一 session:
A1 → 等回复 → A2 → 等回复 → A3

不同 session:
A1 ─────────────→
B1 ───────→
C1 ─────────────────→
```

也就是说：

- **同一会话严格串行**；
- **不同会话可以并行**。

这比全局单队列更实用，也比完全并发安全得多。

---

## 6. Request 生命周期不要只写 pending / done

为了排错，建议至少区分：

```text
pending
claimed
dispatched
replied
expired
cancelled
```

一个典型流程：

```text
pending
  ↓ listener claim
claimed
  ↓ 已提交到官方对话
dispatched
  ↓ callback 回来
replied
  ↓ 写入本地聊天 ledger
applied
```

其中 `applied` 可以不一定作为 request status 存储，也可以额外使用一个：

```text
ledger_applied = 0 / 1
```

这样 transport 状态和“前端历史是否已经持久化”不会混在一起。

### 为什么这很有用

假设官方回复已经回到 VPS，但本地前端刚好刷新。

如果系统把“HTTP 请求有没有保持连接”当成唯一真相，回复很容易丢。

更稳的方式是：

1. callback 先把 request 标记成 `replied` 并保存回复；
2. 后台 reconcile worker 定期扫描 `replied AND ledger_applied = 0`；
3. 把回复写进对应 session 的本地消息 ledger；
4. 再标记 `ledger_applied = 1`。

于是官方回复是否成功保存，不再依赖发送消息时那个浏览器 HTTP 连接还活着。

---

## 7. 前端连接时，只做“读取配对”，不要重新决定配对

前端需要的其实很少。

对于当前 session，它只要读：

```json
{
  "transport": "chatgpt_bridge",
  "bridge": {
    "host_kind": "chat",
    "binding_id": "chat-p-a13f9c22d5e1"
  }
}
```

发送时把已有配对带回去：

```js
function bridgeRequestConfig(session) {
  if (session.transport !== 'chatgpt_bridge') return {};
  return {
    bridge_host_kind: session.bridge.host_kind,
    bridge_binding_id: session.bridge.binding_id,
  };
}
```

后端再做最终校验。

### Listener 指令也从已保存的 binding 生成

例如 UI 可以提供一个“复制 listener 指令”的按钮：

```text
Open a listener for binding_id="chat-p-a13f9c22d5e1" and keep polling it.
```

关键点不在具体提示词，而在于：**这条指令里的 binding 必须来自 session 已持久化的配对，而不是临时随机生成。**

这样前端 session、Bridge queue、官方窗口三个对象才能稳定对上。

---

## 8. 切换 Chat / Work：轮换 binding，而不是复用旧 lane

如果一个本地会话原来绑定 Chat，后来切到 Work，最省事的做法看起来是：

```text
把 host_kind 从 chat 改成 work
binding_id 保持不变
```

不建议这样做。

因为旧 listener 可能还在消费这个 binding。

更安全的方式是：

```text
old: chat-p-a13f9c22d5e1
new: work-p-8c11f7084c29
```

切换宿主时生成新 binding，并更新 session 配对。

同时，对旧 lane 做一个非常克制的清理：

> 只取消这个 session 在旧 binding 上**还没有被 claim 的 owner request**。

不要把已经进入官方对话的请求直接当作不存在，也不要删除本地历史。

伪代码：

```python
UPDATE bridge_requests
SET status = 'cancelled_by_rotation'
WHERE session_id = ?
  AND binding_id = ?
  AND event_type = 'owner_message'
  AND status = 'pending';
```

这能避免切换完成后，旧 listener 突然又把一条旧消息送进原来的官方窗口。

---

## 9. SQLite 很适合做这层“小型持久队列”

如果你的 Bridge 是单机或小规模自用，没有必要一开始就上 Redis / RabbitMQ。

SQLite 已经足够支撑：

- request 生命周期；
- binding 状态；
- listener generation；
- claim 排他；
- callback 去重；
- 本地重启后的恢复；
- 小规模并发。

建议至少打开：

```sql
PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=15000;
```

关键 claim / listener takeover 操作使用事务：

```sql
BEGIN IMMEDIATE;
```

这样“查 candidate → 标 claimed”可以放在同一个写事务里完成，减少两个 listener 同时拿到同一条请求的机会。

---

## 10. 建议的数据表最小形态

可以从下面这个骨架开始：

```sql
CREATE TABLE bridge_requests (
  request_id TEXT PRIMARY KEY,
  client_request_id TEXT NOT NULL,
  session_id TEXT NOT NULL,
  binding_id TEXT NOT NULL,
  message TEXT NOT NULL,
  status TEXT NOT NULL,
  created_at REAL NOT NULL,
  claimed_at REAL,
  dispatched_at REAL,
  replied_at REAL,
  reply_text TEXT,
  ledger_applied INTEGER NOT NULL DEFAULT 0,
  UNIQUE(session_id, client_request_id)
);

CREATE INDEX bridge_requests_binding_status
ON bridge_requests(binding_id, status, created_at);

CREATE INDEX bridge_requests_session_status
ON bridge_requests(session_id, status, created_at);

CREATE TABLE bridge_bindings (
  binding_id TEXT PRIMARY KEY,
  last_seen_at REAL NOT NULL,
  active_listener_token TEXT,
  listener_generation INTEGER NOT NULL DEFAULT 0,
  listener_paused INTEGER NOT NULL DEFAULT 0
);
```

`UNIQUE(session_id, client_request_id)` 很值得保留，它可以让前端因为重试导致的重复 enqueue 变成幂等请求，而不是重复发送两次。

---

## 11. 最值得测的不是“能不能发”，而是这 8 个场景

### 1. 两个不同 session 同时发送

期望：

```text
A → binding A → listener A
B → binding B → listener B
```

互不串窗。

### 2. 同一 session 连续 enqueue 两条

期望：第二条保持 pending，直到第一条结束。

### 3. 同一个 binding 打开第二个 listener

期望：新 listener 接管；旧 listener 得到 `listener_superseded`，不能再 claim。

### 4. listener 暂时离线

期望：owner message 可以先持久化为 pending；listener 恢复后继续消费。

### 5. callback 重复发送

期望：第二次 callback 被识别为 duplicate，不重复写前端消息。

### 6. callback 已到，但前端页面刷新

期望：后台 reconcile 仍能把 replied request 写进本地 ledger。

### 7. Chat 切到 Work

期望：生成新 binding；旧 binding 未 claim 的消息被取消；已有本地历史不丢。

### 8. 前端伪造另一个 dedicated binding

期望：后端拒绝，不允许前端偷偷重配 session。

---

## 12. 一个更容易记住的模型

最后可以把整个设计记成四句话：

```text
Session owns a binding.
Binding owns one active listener.
One session has at most one in-flight request.
Replies become durable before the frontend sees them.
```

翻成中文：

```text
一个会话持有一条专用通道。
一条通道同一时刻只有一个现任 listener。
一个会话同一时刻最多有一条正在官方端执行的请求。
回复先持久化，再交给前端显示。
```

只要这四条不被破坏，多窗口 Bridge 就很难真正“串起来”。

---

## 13. 本篇不包含

为了把重点放在多窗口会话隔离与投递可靠性，本篇不展开以下基础内容：

- 如何第一次把 ChatGPT 网页和本地服务连通；
- Chrome extension / userscript / Playwright / Secure MCP Tunnel 的安装；
- 如何拿到登录态；
- 如何做一个基础 `/chat` API；
- 如何模拟 OpenAI-compatible endpoint；
- 某个特定 Bridge 项目的完整部署。

本文主要讨论已经具备基础 Bridge 之后的多窗口设计：**配对、隔离、listener 接管、顺序和持久化。**

---

## 14. 关于示例与隐私

本文是从一个实际长期运行的私人前端 / Bridge 系统中抽取出的通用设计。

公开示例遵循以下处理：

- binding 示例全部使用虚构值；
- 不公开真实域名、端口、token、PIN；
- 不公开私人 prompt；
- 不复制私人项目的完整生产源码；
- 只保留可以独立理解和复现的架构骨架。

如果你已经有自己的 Bridge，实现时只需要把这里的几个约束嵌入现有 transport 层，不需要照搬我们的项目结构。

---

## Authors

**Umi & CatTea**

本教程正文沿用仓库根目录的 **CC BY-NC-SA 4.0** 许可与署名规则。
