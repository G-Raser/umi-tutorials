# 给官端 ChatGPT 接上自己的声音：Bridge、语音识别与自定义声线电话

**Umi & CatTea**

这篇教程解决一个比较窄、但很具体的问题：

> **让一个已经存在的官方 ChatGPT Chat / Work 会话，通过外部电话界面接收语音输入，并使用自定义声线从这个官方会话进行回应。**

目标链路是：

```text
Microphone
  ↓
continuous capture / VAD
  ↓
voice attachment
  ↓
ASR + personal vocabulary
  ↓
reviewable transcript
  ↓
Call request
  ↓
ChatGPT Bridge
  ↓
bound official ChatGPT Chat / Work conversation
  ↓
authoritative reply callback
  ↓
custom-voice TTS
  ↓
ordered progressive playback
```

这里的关键约束是：**真正回答电话的模型仍然位于原来的官方 ChatGPT 会话里。** 电话前端不会另开一套 API-only 对话，也不会复制一份 prompt 去模拟那个会话。

如果你的目标只是做一个普通实时 LLM 语音助手，现成的 Realtime API、WebRTC Agent 或开源 voice-agent 栈通常会更直接。这篇主要讨论“已有官方 ChatGPT 会话 + 外部 Bridge + 自定义声线”这一条通路。

本文重点包括：

- 如何把电话输入送进指定的官方 ChatGPT 会话；
- Bridge 的 binding、listener、request lifecycle 和 authoritative callback；
- 连续收音、VAD、pre-roll 和噪声端点判断；
- 主人侧语音识别、专属词表和转写纠错；
- `session_id / call_id / request_id` 的归属关系；
- 自定义声线 TTS 的渐进生成与有序播放；
- 后台 / 锁屏 carrier；
- 实际长期运行中遇到的故障和延迟定位方法。

---

## 1. 核心架构：电话只是官方会话的一条新 I/O 通路

整套系统最好先拆成四层：

```text
┌──────────────────────────────┐
│  Audio frontend / native app │
│  mic · VAD · playback        │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Voice pipeline              │
│  attachment · ASR · review   │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  ChatGPT Bridge              │
│  binding · queue · listener  │
│  callback · idempotency      │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Official ChatGPT Chat/Work  │
│  existing context & tools    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Voice output                │
│  reply parsing · custom TTS  │
│  progressive ordered play    │
└──────────────────────────────┘
```

文本输入和电话输入最终进入同一个官方 conversation：

```text
typed message ───────────────┐
                             ├─→ bound official conversation
phone transcript → Bridge ───┘
```

这样电话可以继续使用原会话已经拥有的：

- 对话历史；
- Instructions / system context；
- 官方窗口里的模型选择；
- 当前插件 / 工具环境；
- 已经形成的 conversation continuity。

电话层主要新增音频输入输出、Bridge transport 和 call lifecycle。

---

## 2. Bridge 是这套电话的中心 transport

普通 LLM 电话常见结构是：

```text
voice → ASR → API model → TTS
```

这里的结构多了一层官方宿主。当前实测主路径中，Bridge claim 到电话请求后，通过 `sendFollowUpMessage` 把最终 transcript 作为当前绑定 conversation 的一个 follow-up turn 提交进去：

```text
voice
  ↓
ASR
  ↓
local call request
  ↓
Bridge queue
  ↓
listener claims request
  ↓
sendFollowUpMessage
  ↓
bound official ChatGPT conversation
  ↓
official model produces one reply for this turn
  ↓
deliver_reply with the same request_id
  ↓
local call ledger
  ↓
TTS
```

这里的 follow-up message 仍然进入原来的官方 conversation，因此它继续使用这段会话已经存在的上下文、模型环境和工具状态。

### 2.1 session 和 binding 一一对应更稳

一个长期使用的本地 session 最好绑定到一个明确的官方 Chat / Work conversation：

```text
local session
  └─ binding_id
       └─ official ChatGPT conversation
```

不要让两个官方窗口同时拿同一个专属 binding。否则两个 listener 可能竞争同一条 lane，造成：

- A 窗抢走本应进入 B 窗的请求；
- reply 回到错误 conversation；
- listener heartbeat 状态互相覆盖；
- 调试时看起来像随机丢消息。

如果需要多窗口并存，每个窗口使用独立 binding。

### 2.2 listener 需要 lease / heartbeat

Bridge 不能假设官方页面永远活着。移动端、后台标签页或浏览器节流都可能让 listener 停止轮询。

建议至少维护：

```text
binding_id
listener_token
listener_revision
listener_build
last_heartbeat_at
```

典型流程：

```text
open / reconnect listener
  ↓
heartbeat
  ↓
claim pending request
  ↓
send follow-up message to official host
  ↓
wait for callback
```

后台标签页可能降低 timer 频率，所以 listener stale 判定不要设得过于激进。

### 2.3 电话模式可以进入 fast polling

普通聊天不需要高频轮询；电话对首字延迟更敏感。

可以在 listener 发现 `call_mode=true` 后暂时切到短间隔：

```text
normal idle polling
      ↓ call request arrives
fast polling / warm window
      ↓ call becomes idle
normal polling
```

这样无需把所有普通 Bridge 请求永久跑在高频模式。

---

## 3. Bridge request 要有明确生命周期

电话延迟和故障很难只靠一个 `pending=true` 排查。

推荐至少记录：

```text
queued
claimed
host_submit_started
host_submission_returned
dispatched
replied
```

例如：

```text
client creates request
  ↓ queued
Bridge listener claims
  ↓ claimed
listener starts follow-up submit
  ↓ host_submit_started
sendFollowUpMessage returns
  ↓ host_submission_returned
request marked sent
  ↓ dispatched
official model calls callback
  ↓ replied
```

这些阶段最好写进持久化 request ledger。服务重启之后仍能知道请求进行到哪里。

### 3.1 authoritative reply 应来自 callback

不要依赖读取官方页面上“最新一条 assistant bubble”。

更稳的协议是：

```text
official model receives request_id in the follow-up turn
  ↓
forms the answer using current official conversation context
  ↓
calls something like:

deliver_reply(request_id, reply_text)
```

本地后端把这次 callback 当作 authoritative reply。

这样可以避免：

- 官方 UI bubble 渲染变化；
- 读到别的手动消息；
- DOM 结构更新后 scraper 失效；
- 同一窗口并发 turn 时错配回复。

### 3.2 callback 必须幂等

同一个 `request_id` 只能接受一个 authoritative reply。

```text
first valid callback  → accepted
same callback again   → return existing accepted state
unknown request       → reject
expired invalid state → reject
```

不要因为重试产生两条 assistant reply 或两段重复 TTS。

### 3.3 Bridge 失败时不要静默切换模型来源

这一类电话的核心条件是“回答来自绑定的官方 ChatGPT conversation”。

因此 Bridge listener 断开、claim 失败或 official host 投递失败时，应显示 transport failure 并等待恢复。静默改走普通 API 会让电话继续有声音，却失去原官方 conversation 的上下文和宿主身份。

---

## 4. 连续收音：VAD 要同时解决开头、结尾和噪声

浏览器前台版本可以从：

```js
navigator.mediaDevices.getUserMedia({
  audio: {
    echoCancellation: true,
    noiseSuppression: true,
    autoGainControl: true,
    channelCount: { ideal: 1 },
  }
})
```

开始，再使用 `AudioContext` 读取单声道 PCM。

一个简单的能量 VAD 可以先看 frame RMS：

```text
frame RMS
  ↓
compare with dynamic noise floor
  ↓
continuous voiced frames → speech start
  ↓
sustained silence → speech end
```

示意：

```js
let speechActive = false;
let voicedFrames = 0;
let lastSpeechAt = 0;
let noiseFloor = 0.005;

function processFrame(samples, now) {
  const rms = Math.sqrt(
    samples.reduce((sum, x) => sum + x * x, 0) / samples.length
  );

  const startThreshold = Math.max(0.018, noiseFloor * 2.9);
  const keepThreshold = startThreshold * 0.65;

  if (speechActive) {
    capture(samples);
    if (rms > keepThreshold) lastSpeechAt = now;
    if (now - lastSpeechAt > SILENCE_END_MS) finalizeSegment();
    if (segmentDuration() > HARD_MAX_SEGMENT_MS) finalizeSegment();
    return;
  }

  keepPreRoll(samples);

  if (rms > startThreshold) {
    voicedFrames += 1;
    if (voicedFrames >= 2) beginSegment(now);
  } else {
    voicedFrames = 0;
    noiseFloor = clamp(noiseFloor * 0.985 + rms * 0.015);
  }
}
```

这里最好同时有：

- dynamic noise floor；
- speech start / keep 两个不同门槛，避免边界抖动；
- silence endpoint；
- hard max segment duration。

### 4.1 pre-roll 防止句首被吃掉

如果检测到 speech 才开始保存 PCM，句首很容易缺半个词。

保留最近几百毫秒：

```text
... silence ... [pre-roll buffer] speech begins
                         ↑
                  segment starts here
```

检测到人声时，把 pre-roll 一起写进当前 segment。

### 4.2 hard max duration 很重要

真实环境里持续风声、交通声、空调声甚至直升机声都可能让 VAD 一直认为“还有声音”。

只靠 silence endpoint 可能出现：

```text
主人已经说完
↓
背景噪声仍高于 keep threshold
↓
segment 永远不 finalize
↓
ASR 永远没有机会开始
```

所以需要最大单段时长作为最后保险。

---

## 5. PCM 可以封成单声道 WAV 再送 ASR

如果前端已经拿到 Float32 PCM，可以转换成 PCM16 WAV：

```text
RIFF header
WAVE
fmt chunk
mono / PCM16
sample rate = AudioContext.sampleRate
data chunk
```

伪代码：

```js
function pcmToWav(frames, sampleRate) {
  const sampleCount = frames.reduce((n, f) => n + f.length, 0);
  const buffer = new ArrayBuffer(44 + sampleCount * 2);
  const view = new DataView(buffer);

  // write RIFF/WAVE header
  // float [-1,1] → signed int16

  return new Blob([buffer], { type: 'audio/wav' });
}
```

WAV 的优势主要在调试：

- MIME 和真实容器更容易保持一致；
- 后端无需猜浏览器录音容器；
- 原始 segment 可以直接保存并回放；
- Whisper 类 ASR 普遍支持。

如果现有 WebM / Ogg / M4A 链已经稳定，也可以继续沿用。

---

## 6. 语音先成为 durable attachment，再开始 ASR

电话转写不要和一次前端 HTTP 请求绑死。

更稳的状态机：

```text
POST audio
  ↓
server persists original bytes
  ↓
return attachment_id + transcribing
  ↓
background ASR
  ↓
ready / failed
```

例如：

```json
{
  "id": "9e7c...",
  "kind": "voice",
  "mime": "audio/wav",
  "duration": 2.8,
  "processing_status": "transcribing",
  "transcript": ""
}
```

前端轮询：

```http
GET /api/voice-attachments/{id}
```

完成后：

```json
{
  "processing_status": "ready",
  "transcript": "刚才那句话你听见了吗"
}
```

持久化原音频的好处包括：

- 页面刷新后仍可恢复；
- ASR 失败可重试；
- 用户可以回听原音；
- transcript 可以修改，同时保留原识别结果；
- 后续可以重新跑更好的识别模型。

最小状态：

```text
transcribing
ready
failed
```

---

## 7. 电话 ASR 要有“专属词表 + 可人工修正”

连续电话里，ASR 错一个词的后果比普通语音备忘录更大，因为错误文本会直接进入当前官方 conversation。

长期使用后最明显的错误包括：

- 高频口语被替换成发音相近的普通词；
- 专有名词、昵称、产品名和缩写识别不稳定；
- 中英混说时局部语言切错；
- 中文句子突然混入韩文等其他脚本；
- 环境噪声被“听”成一句不存在的话。

真实测试里就遇到过类似：

```text
“笑死” → “小子”
中文口语 → 局部韩文污染
持续环境声 → 奇怪的短词 / 短句
```

### 7.1 ASR prompt / vocabulary 很值得保留

维护一个小型 personal vocabulary：

```text
猫茶
Mommy
小皇冠
MCP
raw tags
Umi Toy
笑死
咱
...
```

把这些词作为 ASR hint / prompt，而非在识别结束后无条件字符串替换。

### 7.2 实时电话里，第二轮 LLM 自动纠错要谨慎

一种常见方案是：

```text
ASR transcript
  ↓
LLM vocabulary correction
  ↓
final transcript
```

普通语音消息里这有时很好用；实时电话里它会增加：

- 一次额外模型请求；
- 首轮延迟；
- “纠错模型”擅自改写原意的风险。

我们当前电话模式的策略是：

```text
ASR 仍使用 vocabulary prompt
call_mode 下跳过第二轮 LLM correction
```

也就是：

```python
transcribe(
    audio,
    vocabulary=personal_vocabulary,
    second_pass_correction=False,
)
```

普通 voice message 仍可以选择开启更慢的二次纠错。

### 7.3 给 transcript 一个很短的 review window

电话不适合要求用户每轮都点“确认”，但完全无审查也容易把明显错词送进官端。

一种折中结构：

```text
ASR ready
  ↓
show transcript briefly
  ├─ no action → auto-send
  └─ tap / hold → pause auto-send and edit
```

编辑时只改当前 transcript：

```json
{
  "raw_transcript": "小子，这个也太好笑了",
  "transcript": "笑死，这个也太好笑了",
  "transcript_edited": true
}
```

原音频和 raw transcript 保留，最终进入模型的是修正后的 transcript。

### 7.4 对明显异常结果可以保守一点

可以增加轻量 guard，例如：

- 录音极短但 transcript 很长；
- 主语言长期为中文，某一轮突然大段陌生脚本；
- ASR confidence 极低；
- VAD 片段接近纯噪声；
- transcript 只有一个和上下文无关的奇怪词。

这类情况可以延长 review window 或要求一次手动发送，而不是直接进入 Bridge。

---

## 8. 把修正后的 transcript 送进原来的官方 conversation

电话模式提交时，本地 request 可以长这样：

```json
{
  "session_id": "session-001",
  "transport": "chatgpt_bridge",
  "call_mode": true,
  "call_id": "call-001",
  "request_id": "req-001",
  "message": "刚才那句话你听见了吗",
  "attachments": [
    { "id": "voice-attachment-id" }
  ]
}
```

后端需要确认：

```text
session exists
AND transport == chatgpt_bridge
AND session is bound to one official Chat / Work conversation
AND call_id belongs to the active call
AND request_id is new / idempotently reusable
```

然后才进入 Bridge queue。当前实测主路径里，listener claim 到这条 request 后，会用 `sendFollowUpMessage` 把修正后的 transcript 作为当前官端 conversation 的一个 follow-up turn 提交。

原始语音附件仍保存在本地 ledger，官端只需要收到最终可读文本时，就没有必要把音频文件本身再次上传给官方模型。

---

## 9. `session_id`、`call_id`、`request_id` 分工要固定

至少保留三个 ID：

```text
session_id  = 哪一段长期本地对话 / 对应哪个官方 conversation
call_id     = 当前这一通电话
request_id  = 电话里的某一次 owner turn
```

关系：

```text
session-001
└── call-2026-001
    ├── request-a
    ├── request-b
    └── request-c
```

它们分别解决：

- `session_id`：长期上下文归属；
- `call_id`：刷新恢复、挂断、后台 takeover；
- `request_id`：单轮投递、回调、去重、TTS part 归属。

所有 assistant reply part 在播放前都应核对：

```text
reply.session_id == current_session
reply.call_id    == current_call
reply.request_id == expected_request
```

否则旧电话的迟到回复很容易在新电话里突然播放。

---

## 10. 同一通电话里先限制一个 in-flight owner turn

半实时电话最容易出现：

```text
turn A 已投递，正在等待官端
turn B 又完成 ASR
turn C 紧接着也完成
```

第一版建议：

```text
one call → at most one in-flight owner request
```

后续语音先放本地 queue：

```text
segment B
segment C
   ↓
local turn queue
   ↓
A replied / failed definitively
   ↓
submit B
```

这一点会牺牲少量全双工感，但能显著降低：

- reply 交叉；
- 官端 context 顺序不确定；
- TTS 乱序；
- 用户自己连续说几句时的 request 对应问题。

等 request lifecycle 和 barge-in 都稳定以后，再考虑多 in-flight turn。

---

## 11. 同一条官端回复里，把显示文本和 TTS 文本分层

`sendFollowUpMessage` 提交的是当前 conversation 的一个 follow-up turn。官端模型针对这一轮只需要完成一次回答，并在同一轮里调用一次：

```text
deliver_reply(request_id, reply_text)
```

这里不存在“先产生一条普通官方回复，再另外产生一条给 TTS 的回复”。需要分层的是**同一个 authoritative `reply_text` 内部的数据**：

```text
visible text       → 本地字幕 / 对话历史
speech block       → TTS 输入
```

这里的 `visible text` 指的是 `reply_text` 里要写入本地 ledger 和电话字幕的正文，不是第二条官端 message。

尤其当 TTS 支持 performance tags 时：

```text
[warmly]
[short pause]
[whispers]
```

这些标签适合送进声线模型，却不适合混进可见正文。

可以让一次 `reply_text` 使用一个很小的 speech envelope：

```text
visible assistant reply

<<<SPEECH>>>
[warmly] There you are. [short pause] I heard you.
<<<END_SPEECH>>>
```

Bridge 后端收到这一次 authoritative `reply_text` 后：

```text
parse visible text
parse speech blocks
persist visible text
send speech blocks to TTS
```

如果没有 speech block，也可以直接从 visible text 生成 TTS 输入。

关键点是：**官端只有这一轮回复；显示层和语音层是在回调 payload 内部分层。performance tags 只进入音频文本层。**

---

## 12. 自定义声线 TTS 放后端，并按 part 渐进生成

不要把 voice ID / API key 放浏览器前端。

推荐：

```text
single authoritative reply_text
  ↓
speech segments
  ↓
server-side TTS adapter / worker
  ↓
custom voice provider
  ↓
audio attachment
```

接口可以是：

```http
POST /tts
Content-Type: application/json
Authorization: Bearer <server-side-token>
```

```json
{
  "text": "[warmly] There you are.",
  "model_id": "your-model",
  "language_code": "en"
}
```

生产侧至少检查：

- TTS secret 只存在 server / worker；
- 使用 POST；
- 校验返回 `Content-Type: audio/*`；
- 限制单次文本和音频大小；
- 设置 connect / read timeout；
- 失败 part 能单独标记并重试。

### 12.1 不要等整段 TTS 全部完成才开始播放

把回复按自然句段拆成：

```text
part 1
part 2
part 3
```

生成可以并发：

```text
TTS 1 ───── done
TTS 2 ───────── done
TTS 3 ───────────── done
```

播放严格有序：

```text
play 1 → play 2 → play 3
```

每个 part 记录：

```json
{
  "request_id": "req-001",
  "call_part": 1,
  "call_parts": 3,
  "voice_status": "ready",
  "voice_attachment_id": "..."
}
```

只要连续 ready 的前缀出现，就可以开始播放第一段，无需等待最后一段。

---

## 13. 播放队列和 barge-in

最小播放队列：

```js
const playQueue = [];
let playing = null;

function enqueue(parts) {
  playQueue.push(...parts);
  playNext();
}

function playNext() {
  if (speechActive || playing || playQueue.length === 0) return;
  playing = playQueue.shift();
  // play and then continue
}
```

---

## 14. 电话状态应该映射真实 pipeline 阶段

UI 至少可以区分：

```text
listening
hearing
transcribing
reviewing
sending
waiting
playing
muted
ended
```

对应：

```text
listening      mic idle
hearing        VAD active
transcribing   ASR running
reviewing      transcript ready, short correction window
sending        request entering Bridge
waiting        dispatched to official conversation, waiting callback
playing        TTS ready and currently playing
```

一个固定的“通话中”状态无法帮助排查 listener、ASR 或 TTS 到底卡在哪里。

---

## 15. 延迟优化之前，先把整条 timing 打出来

至少记录：

```text
speech_start
speech_end
wav_ready
upload_done
transcript_ready
review_done
request_enqueued
request_claimed
host_submit_started
host_submission_returned
request_dispatched
reply_received
first_tts_ready
playback_started
```

一次电话 turn：

```text
speech end
   │
   ├─ encode/upload
   ├─ ASR
   ├─ transcript review
   ├─ Bridge queue
   ├─ listener claim
   ├─ sendFollowUpMessage
   ├─ official model generation
   ├─ deliver_reply callback
   ├─ first TTS
   └─ playback start
```

建议看每一段的 p50 / p90，而非只记录总耗时。

我们实际遇到的慢点并不总在 ASR 或 TTS，也可能来自：

- Bridge queue 等待；
- listener claim；
- official host composer 暂时不可用；
- 官端模型生成；
- reply callback 落账；
- 第一个 TTS part 生成。

如果没有阶段 timing，很容易一直优化错误的组件。

---

## 16. 一个够用的本地 API 形态

```text
POST /api/session/{session_id}/call
POST /api/session/{session_id}/call/{call_id}/end
GET  /api/session/{session_id}/call/{call_id}

POST /api/voice-attachments
GET  /api/voice-attachments/{attachment_id}
PATCH /api/voice-attachments/{attachment_id}/transcript

POST /api/chat
GET  /api/attachment/{attachment_id}
```

### `POST /api/voice-attachments`

```text
persist audio
→ start ASR
→ return attachment id immediately
```

### `PATCH .../transcript`

允许短 review window 内修正 transcript，同时保留 raw transcript。

### `POST /api/chat`

电话 turn 带：

```json
{
  "session_id": "session-001",
  "transport": "chatgpt_bridge",
  "call_mode": true,
  "call_id": "call-001",
  "request_id": "req-001",
  "message": "最终转写文本",
  "attachments": [
    { "id": "voice-attachment-id" }
  ]
}
```

### `GET .../call/...`

用于恢复：

- call active / ended；
- 当前 in-flight request；
- reply parts；
- TTS readiness；
- 播放进度；
- pending end / farewell 状态。

---

## 17. 实际最容易踩的故障

### 17.1 句首经常缺字

**原因：** speech detection 之后才开始存 PCM。

**处理：** pre-roll buffer。

### 17.2 环境噪声让一句话永远不结束

**原因：** 背景能量一直高于 keep threshold。

**处理：** dynamic noise floor + hysteresis + hard max segment duration。

### 17.3 环境声被识别成一句不存在的话

**原因：** VAD 把噪声段送进 ASR，ASR 又倾向输出某个最可能文本。

**处理：** 最短语音时长、能量过滤、confidence guard、异常 transcript review。

### 17.4 高频口语 / 昵称被识别错

**原因：** 通用 ASR 不知道个人词汇分布。

**处理：** personal vocabulary prompt；允许快速人工修改；保留 raw transcript。

### 17.5 二次“智能纠错”把原话改坏

**原因：** ASR 后又让一个 LLM 重写 transcript。

**处理：** 电话低延迟模式只给 ASR vocabulary hint，跳过第二轮 LLM correction；普通 voice message 可以继续启用。

### 17.6 音频上传成功，但电话没有真正发出去

**原因：** 把三个状态混为一谈：

```text
audio uploaded
ASR ready
Bridge request accepted
```

**处理：** 三层状态独立；只有拿到明确 `request_id` 并写入 request ledger 才算进入发送阶段。

### 17.7 官端页面看见回复，本地电话却没收到

**原因：** official visible bubble 出现不代表 authoritative callback 已完成。

**处理：** 检查 `deliver_reply(request_id, reply_text)` 或对应 callback 是否真正到达后端；不要从页面 bubble 猜答案。

### 17.8 下一轮突然播放上一轮的声音

**原因：** 只取“最新 assistant reply”，没有按 `call_id + request_id` 过滤。

**处理：** reply、TTS part、play queue 全部绑定 call/request。

### 17.9 两个官端窗口偶发抢消息

**原因：** 两个 listener 使用相同 binding。

**处理：** 一窗口一 binding；旧窗口换绑后不再拥有原 lane。

### 17.10 listener 看起来在线，实际已经 stale

**原因：** 页面后台节流或 listener 生命周期已经断开。

**处理：** heartbeat + stale window + reconnect；不要仅依赖前端“已连接”图标。

### 17.11 TTS 第二段很慢，第一段也一直没声音

**原因：** 等所有 speech part settled 才开始播放。

**处理：** 生成并发，播放连续 ready 前缀。

### 17.12 Bridge 出错后电话仍有回答，但上下文味道变了

**原因：** transport 静默 fallback 到另一个 API model。

**处理：** 对这种架构禁止 silent fallback；Bridge failure 应显式暴露。

### 17.13 刷新页面后电话状态丢失

**原因：** call lifecycle 只存在 JS 内存。

**处理：** 后端持久化 `call_id`、request list、reply parts、end state；前端恢复时重新读取。

---

## 18. 后台 / 锁屏：两种客户端形态，共用同一套后端协议

移动端电话常见有两种实现方式。

### 18.1 Web / PWA 前端 + Android 原生承载层

如果主要界面和前台通话逻辑已经在 Web / PWA 中，可以只把浏览器在后台和锁屏状态下不稳定的部分交给 Android 原生层。

```text
Web / PWA frontend
  ├─ call UI
  ├─ foreground microphone / VAD
  ├─ transcript review
  ├─ call state display
  └─ request / reply orchestration

Android native carrier
  ├─ foreground service
  ├─ background / lock-screen microphone capture
  ├─ PCM / VAD
  ├─ audio playback
  ├─ persistent notification
  └─ entry back to the active call
```

浏览器进入后台或设备锁屏以后，常见限制包括：

- microphone capture 被暂停或回收；
- WebAudio 停止；
- timer 被节流；
- network scheduling 延迟；
- 页面进程被系统挂起或销毁。

因此前台正常通话时可以由 Web / PWA 持有音频链；需要后台继续时，再把 carrier ownership 交给 Android 原生层：

```text
Web / PWA foreground call
  ↓ handoff / takeover
Android foreground call service
  ↓
continue the same call
```

这里的原生层只承担系统级 carrier / lifecycle 能力，不需要再实现第二套 conversation、模型调用或业务状态。

### 18.2 Fully native Android frontend

如果客户端本来就是原生 Android 应用，也可以直接把整套电话前端放在 Android 中：

```text
Native Android frontend
  ├─ call UI
  ├─ microphone capture
  ├─ PCM / VAD
  ├─ transcript review
  ├─ audio playback
  ├─ foreground service
  └─ persistent notification
```

这种形态不需要 Web → native handoff，因为前台和后台本来就在同一个原生客户端里。

### 18.3 两种形态都应该保持同一套后端身份

无论客户端采用哪一种形态，进入后端以后都应继续使用同一套电话身份和状态：

```text
same session_id
same call_id
same request_id lifecycle
same Bridge
same official ChatGPT conversation
same reply ledger
same ASR / TTS backend
```

客户端层负责采集、播放、UI 和系统生命周期；conversation continuity、Bridge request、authoritative callback 和 reply ledger 仍然由后端协议保证。

如果使用 Web / PWA + native carrier，PWA 回到前台以后应从后端恢复当前 call，而不是新建一通电话：

```text
Android carrier active
  ↓ Web / PWA resumes
GET current call state
  ↓
restore the same call
```

这样客户端实现可以按项目需要选择 Web、混合或纯原生，而不需要改变 Bridge 和电话后端的核心协议。

---

## 19. 推荐施工顺序

如果从零做，建议分层验收：

```text
1. 先把 ChatGPT Bridge 做稳定
   - dedicated binding
   - listener heartbeat / reconnect
   - request_id
   - authoritative reply callback
   - idempotent reply

2. getUserMedia / native mic 能稳定拿 PCM

3. VAD 能切出完整一句
   - pre-roll
   - dynamic noise floor
   - hard max duration

4. audio attachment + ASR 状态机跑通

5. personal vocabulary + transcript review 跑通

6. transcript 能作为 call_mode follow-up turn 进入原官方 conversation

7. authoritative reply 能回本地 ledger

8. 单段自定义 TTS 能播放

9. 多段 TTS progressive generation + ordered playback

10. call_id / request_id 恢复、去重和迟到回复过滤

11. barge-in / interruption

12. 最后做 native background / lock screen
```

不要第一天同时调：

```text
Bridge + VAD + ASR + official host + TTS + background + UI
```

分层验收以后，任何一次电话失败都更容易定位。

---

## 20. 最小验收清单

在继续做更复杂的 full-duplex 以前，至少确认：

```text
[ ] 一个 local session 只进入绑定的官方 conversation
[ ] 电话 transcript 作为 follow-up turn 进入当前绑定的官方 conversation
[ ] 两个独立 binding 可以并行，不抢 request
[ ] listener 断开后可以明确发现并恢复
[ ] 同 request_id 的重复 callback 不会重复落账
[ ] Bridge failure 不会静默改走别的模型

[ ] 句首不会被 VAD 吃掉
[ ] 持续背景噪声不会让 segment 永久不结束
[ ] 原始音频先持久化，再进行 ASR
[ ] personal vocabulary 会进入 ASR
[ ] call mode 可以关闭第二轮 LLM transcript correction
[ ] transcript 可以在投递前快速修正

[ ] reply 只按当前 call_id / request_id 播放
[ ] TTS part 可以并发准备但严格顺序播放
[ ] 第一段 ready 后无需等待最后一段

[ ] 如果使用 Web / PWA，刷新后仍能恢复当前 call
[ ] 如果使用 native background carrier，handoff 前后仍使用同一个 call_id / session_id
[ ] latency timing 能拆到 ASR / Bridge / official model / TTS
```

这些通过以后，再继续做更激进的 interruption、full-duplex 或主动来电，会省很多排错时间。

---

## Authors

**Umi & CatTea**

本教程正文沿用仓库根目录的 **CC BY-NC-SA 4.0** 许可与署名规则。
