# 自定义声线 AI 电话：连续收音、转写、会话投递与 TTS 回放

**Umi & CatTea**

> 这篇只讲一条可以实际落地的技术通路：**麦克风连续收音 → 自动分段 → ASR 转写 → 投递到已有 AI 会话 → 拿回文字回复 → 自定义声线 TTS → 顺序播放。**
>
> 不讨论具体角色设定、UI 视觉、私人 prompt 或某个生产环境的部署细节。
>
> 默认你已经有一个能工作的文本聊天入口，以及一个可以根据文本返回音频的 TTS 服务。ASR、LLM 和 TTS 都可以替换供应商。

---

## 1. 先把“电话”理解成现有会话的语音外壳

最重要的一条原则是：

> **不要为了做电话，再单独造一套聊天上下文。**

电话最好只是现有 session 的另一种输入 / 输出方式。

```text
Microphone
   ↓
VAD / segmentation
   ↓
Audio attachment
   ↓
ASR
   ↓
existing conversation/session
   ↓
assistant text reply
   ↓
custom TTS
   ↓
ordered audio playback
```

文本输入和电话输入最后应该进入同一个会话：

```text
typed message ─┐
               ├─→ same session / same context / same memory
voice segment ─┘
```

这样电话天然继承：

- 已有聊天历史；
- system prompt / memory；
- 原模型路由；
- session 持久化；
- 原来的回复同步逻辑。

电话层只负责音频 I/O 和 turn 生命周期，不重新发明聊天系统。

---

## 2. 第一层：连续收音 + 本地 VAD 自动切句

浏览器前台版本可以直接用：

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

再接一个 `AudioContext` 读取单声道 PCM。

最小 VAD 不一定要先引入复杂模型。对个人工具，能工作的第一版可以用：

```text
frame RMS
  ↓
和动态 noise floor 比较
  ↓
连续若干帧超过 threshold → speech start
  ↓
持续静音一段时间 → speech end
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

  const threshold = Math.max(0.018, noiseFloor * 2.9);

  if (speechActive) {
    capture(samples);
    if (rms > threshold * 0.65) lastSpeechAt = now;
    if (now - lastSpeechAt > SILENCE_END_MS) finalizeSegment();
    return;
  }

  keepPreRoll(samples);

  if (rms > threshold) {
    voicedFrames += 1;
    if (voicedFrames >= 2) beginSegment(now);
  } else {
    voicedFrames = 0;
    noiseFloor = clamp(noiseFloor * 0.985 + rms * 0.015);
  }
}
```

### 为什么需要 pre-roll

如果只在检测到 speech 后才开始存 PCM，开头几个音节很容易被切掉。

因此建议一直保留最近约几百毫秒的输入：

```text
... silence ... [pre-roll buffer] speech begins
                         ↑
                  segment 从这里开始
```

### 不要把所有声音都送去 ASR

至少过滤：

- 极短 bump；
- 单次碰撞声；
- 很短的背景噪音；
- 明显低于有效语音时长的片段。

否则会白白增加 STT 请求和误识别。

---

## 3. PCM 最简单可以封成单声道 WAV

如果前端已经拿到 Float32 PCM，可以直接构造 16-bit PCM WAV。

```text
RIFF header
WAVE
fmt  chunk
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

  // write RIFF/WAVE header...
  // write each float sample as int16...

  return new Blob([buffer], { type: 'audio/wav' });
}
```

这样有几个好处：

- 后端解码简单；
- Whisper 类 ASR 普遍支持；
- 调试时可以直接保存播放；
- 不依赖浏览器 `MediaRecorder` 的容器差异。

如果你已经有稳定的 WebM / Ogg / M4A 上传链，也可以继续用，不必强制转 WAV。

---

## 4. 语音上传和 ASR 最好拆成“附件状态机”

不要让一次 HTTP 请求一直卡着等 ASR 完成。

更稳的结构是：

```text
POST audio
  ↓
server saves original audio
  ↓
return attachment_id + status=transcribing
  ↓
background ASR
  ↓
status=ready / failed
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

前端再轮询：

```http
GET /api/voice-attachments/{id}
```

直到：

```json
{
  "processing_status": "ready",
  "transcript": "刚才那句话你听见了吗"
}
```

### 为什么值得多这一层

因为 ASR 不是一个稳定的瞬时操作。

把音频先持久化，再异步转写，可以处理：

- 网络重试；
- 页面刷新；
- ASR 暂时失败；
- 手动修改转写；
- 后续重新分析原录音。

最小状态可以只有：

```text
transcribing
ready
failed
```

---

## 5. 把“转写后的文本”送进原会话

如果你的 LLM / Bridge 本身已经是文本通路，最简单的做法不是硬把原始音频塞给模型，而是：

1. 保存原始语音附件；
2. ASR 得到 transcript；
3. 把 transcript 作为本轮 owner/user message；
4. 在本地 message metadata 里保留原语音附件。

```json
{
  "role": "user",
  "content": "刚才那句话你听见了吗",
  "call_id": "call-001",
  "request_id": "req-001",
  "attachments": [
    {
      "kind": "voice",
      "id": "9e7c...",
      "transcript": "刚才那句话你听见了吗"
    }
  ]
}
```

这样模型收到的是干净文本，但本地仍然保留：

- 原音频；
- 原转写；
- 修正后的转写；
- call / request 归属。

如果以后想把笑声、喘息、停顿等副语言信息也交给模型，可以在 ASR 后额外加一层本地音频分析，再把**有限、可解释的标签**合并进文本上下文；这不是电话 MVP 的必要条件。

---

## 6. `call_id`、`request_id`、`session_id` 不要混在一起

最小电话系统建议至少保留三个 ID：

```text
session_id  = 哪一段长期对话
call_id     = 当前这一通电话
request_id  = 电话里的某一次 user turn
```

关系：

```text
session-001
└── call-2026-001
    ├── request-a
    ├── request-b
    └── request-c
```

这样才能正确处理：

- 旧回复晚到；
- 挂断后还有 TTS 在生成；
- 新电话已经开始，旧电话的 reply 才回来；
- 页面刷新后恢复当前 call；
- 同一通电话里连续说多句。

最重要的是：**播放音频前必须确认 reply 仍属于当前 call + 当前 request。**

---

## 7. 同一通电话里，先保证“一个 turn 完整结束”

电话最容易出现的 bug 之一是：

```text
user turn A 还在等 reply
user turn B 又已经提交
assistant reply A / B 开始交叉
```

第一版最好明确：

```text
一个 session / call 同一时刻最多一条 in-flight user request
```

用户后面说的话可以先进入本地 queue：

```text
captured segment B
captured segment C
        ↓
      local queue
        ↓
A reply settled
        ↓
submit B
```

这会牺牲一点极限实时感，但能显著降低乱序和上下文错位。

后面再考虑更激进的 full-duplex / interruption。

---

## 8. 自定义声线 TTS 放在后端，不要把 voice secret 放前端

拿到 assistant 文本后，后端再调用 TTS。

```text
assistant text
   ↓
TTS adapter / worker
   ↓
custom voice provider
   ↓
audio bytes
   ↓
local attachment
```

接口可以很简单：

```http
POST /tts
Content-Type: application/json
Authorization: Bearer <server-side-token>
```

```json
{
  "text": "There you are. I heard you.",
  "model_id": "your-model",
  "language_code": "en"
}
```

生产里建议：

- voice ID / API key 只放 Worker / server；
- 前端只请求“为这段文本生成当前声线”；
- 使用 POST，不把私人文本放 query string；
- 校验返回的 `Content-Type` 必须是 `audio/*`；
- 给单次音频设最大尺寸；
- 给 TTS 请求设连接和生成 timeout。

如果使用 ElevenLabs、自建 voice worker 或其他克隆声线服务，这一层都可以保持同一个 adapter 接口。

---

## 9. TTS 不要等整段全部生成完再播放

如果 assistant 一次回复几句，最直接的实现是：

```text
full reply
  ↓
one long TTS request
  ↓
全部完成
  ↓
开始播放
```

这样首音延迟会很高。

更实用的做法是按自然句段拆成少量 chunk：

```text
reply
├── part 1
├── part 2
└── part 3
```

后台可以同时准备 1–2 段，但播放必须保持原顺序：

```text
TTS part 1 ─────── done ─→ play 1
TTS part 2 ─────────── done ─→ play 2
TTS part 3 ───────────────── done ─→ play 3
```

关键点：

> **生成可以并发，播放必须有序。**

每个 part 最好独立持久化：

```json
{
  "request_id": "req-001",
  "call_part": 1,
  "call_parts": 3,
  "voice_status": "ready",
  "voice_attachment_id": "..."
}
```

这样 part 1 好了就能先播，不必等待 part 2 / 3。

如果服务中途重启，也只需要补还没完成的 part。

---

## 10. 前端播放队列要和“正在听用户说话”互斥

一个安全的最小规则：

```text
如果用户正在说话 → 暂不播放新的 assistant 音频
如果 assistant 正在播放 → VAD 仍要防止扬声器回声被当成人声
```

播放队列可以是：

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
  // play audio, then clear and recurse
}
```

### 回声和 barge-in

手机扬声器播放 TTS 时，AEC 并不总能完全消掉回声。

如果只看普通 VAD threshold，很容易发生：

```text
assistant 自己播放的声音
→ 被 microphone 收到
→ VAD 认为用户开始说话
→ 自己打断自己
```

更稳的做法是：

- 播放刚开始的几百毫秒先建立 echo baseline；
- 播放期间提高 speech start threshold；
- 必须持续超过更高阈值一小段时间才算真正 barge-in；
- 真正检测到近场用户说话后，再暂停 / 停止当前播放。

第一版如果不需要打断，也可以更简单：assistant 播放时直接暂停 VAD 的 speech start，只保留麦克风采样。

---

## 11. 电话 UI 最值得暴露的是“真实阶段”，不是假进度条

建议状态至少区分：

```text
listening
hearing
transcribing
sending
waiting
playing
muted
ended
```

它们对应真实系统阶段：

```text
listening      麦克风空闲监听
hearing        VAD 已检测到用户说话
transcribing   ASR 处理中
sending        正在把 transcript 投递到会话
waiting        已投递，等待模型回复
playing        TTS 已准备并正在播放
```

不要用一个固定“通话中”覆盖所有状态，否则后台失败时 UI 仍然看起来正常，很难排错。

---

## 12. 延迟优化前先打完整 timing

电话体验不好时，不要先猜是 TTS 慢。

建议至少记录：

```text
speech_end
upload_done
transcript_ready
request_enqueued
request_claimed
request_dispatched
reply_received
first_tts_ready
playback_started
```

一次 turn 的时间线：

```text
speech end
   │
   ├─ ASR ───────────────┐
   │                     │
   ├─ queue / transport ─┤
   │                     │
   ├─ model generation ──┤
   │                     │
   ├─ first TTS ─────────┤
   │                     │
   └─ playback start ─────┘
```

最后看每一段的 p50 / p90。

很多系统里真正最大的等待并不在 ASR 或 TTS，而在：

- 请求排队；
- browser / listener claim；
- 外部模型开始生成之前；
- 回复持久化和同步。

只有量出来以后，才知道该优化哪一段。

---

## 13. 一个够用的最小 API 形态

可以从下面这些接口开始：

```text
POST /api/session/{session_id}/call
POST /api/session/{session_id}/call/{call_id}/end
GET  /api/session/{session_id}/call/{call_id}

POST /api/voice-attachments
GET  /api/voice-attachments/{attachment_id}

POST /api/chat
GET  /api/attachment/{attachment_id}
```

### `POST /api/voice-attachments`

负责：

```text
save audio
→ start ASR in background
→ return attachment id immediately
```

### `POST /api/chat`

电话模式额外带：

```json
{
  "session_id": "session-001",
  "call_mode": true,
  "call_id": "call-001",
  "request_id": "req-001",
  "message": "转写后的文本",
  "attachments": [
    { "id": "voice-attachment-id" }
  ]
}
```

### `GET /api/session/.../call/...`

用来恢复：

- 当前 call 是否仍 active；
- 当前 request；
- assistant reply parts；
- TTS 是否 ready；
- 是否已经挂断。

这比只依赖前端内存稳得多。

---

## 14. 最常见的几个故障

### 1. 句首被切掉

原因：VAD 检测到人声以后才开始录。

处理：保留 pre-roll buffer。

### 2. 背景噪音一直不结束

原因：固定 threshold 不适应环境。

处理：动态 noise floor + 最大单段时长。

### 3. 语音已经上传，但消息偶尔没发出去

原因：把“上传成功”“ASR ready”“聊天 request accepted”当成同一件事。

处理：三层状态分开；没有拿到明确 `request_id` 前不要算投递成功。

### 4. 下一轮回复播放成上一轮的声音

原因：只看“最新 assistant message”，没有按 request / call 归属过滤。

处理：所有 reply 和 voice part 都带 `request_id + call_id`。

### 5. 第二段 TTS 慢，导致第一段也一直不播

原因：等所有 part settled 才开始播放。

处理：按顺序播放“已经连续 ready 的前缀”。

### 6. 模型声音把自己 VAD 触发了

原因：扬声器回声进入麦克风。

处理：AEC + 播放期提高 barge-in 门槛，或者 MVP 直接禁用播放期 speech start。

### 7. 刷新页面后整通电话丢失

原因：call 状态只存在前端变量。

处理：后端持久化 `call_id`、request 列表、reply parts 和结束状态；前端恢复时重新读取。

---

## 15. 如果还要做后台 / 锁屏，浏览器电话和原生 carrier 要分层

前台 PWA / 浏览器版本可以先把整条通路跑通。

但移动端进入后台或锁屏后，浏览器可能限制：

- microphone capture；
- WebAudio；
- timer；
- network scheduling；
- 页面进程生命周期。

如果目标是“像真正电话一样锁屏还能继续”，更稳的结构通常是：

```text
Web UI
  ↓ takeover
Native foreground call service
  ├─ microphone
  ├─ playback
  ├─ persistent notification
  └─ same call_id / same backend session
```

重点是**原生层只接管音频 carrier，不要另造一份聊天会话。**

Web 页面回来以后继续读取同一个 `call_id` 的状态即可。

这部分已经属于第二阶段，不建议和第一版浏览器电话一起开工。

---

## 16. 推荐施工顺序

如果从零实现，建议按下面顺序：

```text
1. getUserMedia 能稳定拿到 PCM
2. VAD 能切出一段完整人声
3. 音频能保存并手动播放
4. ASR attachment 状态机跑通
5. transcript 能进入已有 session
6. assistant reply 能回到本地 ledger
7. 单段自定义 TTS 能播放
8. 多段 TTS 能首段先播、顺序不乱
9. 加 call_id / request_id 恢复与去重
10. 最后再做 barge-in、后台、锁屏和视觉
```

每一步都可以单独验收。

不要一开始同时调：

```text
VAD + ASR + model + TTS + background + UI
```

否则延迟和故障会很难定位。

---

## 17. 一个更容易记住的模型

整条链路可以压成五句话：

```text
Audio becomes a durable attachment before it becomes text.
Text enters the existing conversation instead of a second phone-only context.
Every turn is identified by session + call + request.
TTS may prepare in parallel, but playback stays ordered.
Measure each latency stage before optimizing it.
```

翻成中文：

```text
声音先变成可恢复的附件，再变成文字。
文字进入原有会话，不另建电话专属上下文。
每一轮都用 session + call + request 明确归属。
TTS 可以并发准备，但播放顺序不能乱。
优化前先量清楚每一段延迟。
```

---

## 18. 本篇不包含

为了把重点放在“自定义声线电话”的关键通路，本篇不展开：

- 具体 AI 角色 prompt；
- 私人记忆系统；
- 某个供应商的完整账号 / API 配置；
- 声线训练或克隆教程；
- 具体 ElevenLabs voice ID；
- 私人域名、token、PIN、服务器路径；
- 完整生产项目源码；
- 电话页面视觉设计；
- 主动来电、主动挂断、睡前陪伴等产品层策略。

本文主要讨论可以独立复现的工程骨架：**连续收音、VAD、ASR、会话投递、TTS、播放顺序、状态恢复和延迟测量。**

---

## 19. 关于示例与隐私

本文来自一个实际长期使用的私人 AI 电话系统，但所有公开示例都只保留通用结构。

公开版本不包含：

- 真实声线 ID；
- 真实密钥；
- 私人 prompt；
- 内部域名和生产地址；
- 私人聊天内容；
- 完整私有源码。

如果你已经有自己的聊天前端或 agent，只需要把本文的音频输入 / 输出链嵌入现有 session 层，不需要照搬我们的项目结构。

---

## Authors

**Umi & CatTea**

本教程正文沿用仓库根目录的 **CC BY-NC-SA 4.0** 许可与署名规则。