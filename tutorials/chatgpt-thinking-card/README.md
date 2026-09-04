# ChatGPT 思考卡渲染器复刻指南
## 人类说明 + AI 施工规格

> 这是一个**视觉复刻 / 个人工具制作指南**，目标是把一段思考文本渲染成接近 ChatGPT 浏览器端或 Android App 思考过程样式的 PNG。
>
> **这不是 OpenAI 官方组件，也不包含 ChatGPT 原网页源码。**
>
> 最推荐的用法：**直接把这整个 Markdown 文件发给你常用的 ChatGPT / Claude / Codex / 其他 coding AI，然后让它按后半部分的施工规格做一个本地可用版本。**

---

# Part A｜给人看的

## 1. 这个东西是做什么的？

简单说：

**输入一段“思考标题 + 思考正文 + 耗时”，输出一张看起来像 ChatGPT 思考过程截图的图片。**

比较适合：

- 做自己的 AI 聊天存档；
- 给聊天归档配图；
- 把纯文字思考记录整理成比较像原客户端的视觉卡片；
- 做博客、朋友圈、笔记里的展示图；
- 单纯觉得 ChatGPT 的思考过程 UI 好看，想自己复刻一个。

不需要接入 ChatGPT 账号，也不需要抓网页。

## 2. 最省事的使用方法

如果你不想自己写代码：

1. 下载这份 `.md` 文件。
2. 整份发给一个会写代码的 AI。
3. 对它说：

> 按这个文件后半部分的规格，给我做一个可以本地打开使用的思考卡渲染器。优先做成单文件 HTML，不需要服务器。

4. 等它生成。
5. 本地打开 HTML，输入文字，导出 PNG。

就这样。

## 3. 推荐做成什么形式？

对普通人来说，**单文件 HTML** 最省事。

理想效果：

- 双击打开；
- 不需要安装 Python；
- 不需要数据库；
- 不需要登录；
- 不需要后端；
- 输入文字即可实时预览；
- 点击按钮下载 PNG。

如果你的 AI 更愿意拆文件，也可以做：

```text
thinking-card/
├─ index.html
├─ renderer.js
└─ style.css
```

但不是必须。

## 4. 输入内容怎么写？

最简单的一阶段：

```text
标题：
检查当前实现

正文：
需要先确认现在的排版参数，再决定下一步修改什么。

耗时：
32
```

如果你想手动写多个阶段，可以暂时约定用 Markdown 三级标题：

```markdown
### 检查当前实现

先确认当前版本的结构和参数。

### 调整排版细节

再修正字体、行距、标点和时间线的位置。
```

渲染器可以把每个 `###` 当成一个阶段。

> 注意：ChatGPT 真实多阶段思考在不同客户端 / 版本中的线和节点关系可能还有细微差异。**单阶段样式目前更适合作为稳定参考，多阶段建议拿自己的真实截图继续校准。**

## 5. 推荐保留的几个样式

至少可以做四种 preset：

1. **浏览器 · 电脑端**
2. **浏览器 · 手机端**
3. **Android App · 新版**
4. **Android App · 老版**

iOS 也可以做，但如果没有足够截图，建议标成“设计版 / 实验版”，不要假装已经 1:1 校准。

## 6. 最容易做错的细节

这些比“颜色大概像”重要得多。

### 中文标点

不要出现这种：

```text
这是第一行
：这是第二行
```

像下面这些符号，不应该自己跑到下一行最前面：

```text
，。！？；：、％%）)]】》〉」』”’…,:;.!?
```

同时，前引号 / 左括号也不要孤零零挂在一行末尾，例如：

```text
这句话准备说“
下一行内容”
```

更自然的是让 `“` 跟着后面的第一个字一起换行。

常见前置符号：

```text
（【〔〖〘〚《〈「『“‘
```

### 英文换行

英文不要像中文一样按字符硬切。

错误：

```text
implemen
tation
```

正确：

```text
implementation
```

正常英文应该按单词换行，因此右边缘会自然参差不齐。

只有某个**单独 token 本身比整行还宽**时，才允许内部拆分。

### 阶段标题和正文

一个很容易误判的地方：

**阶段标题和思考正文是同字号。**

层级主要靠：

- 阶段标题更亮；
- 正文更灰；
- 位置不同；
- marker / timeline 提示结构。

不要靠把阶段标题放大一号来制造层级。

### 小圆点

Android 新版使用小圆点。

小圆点的**垂直中心大致对齐阶段标题字形的视觉中心**，不要简单对齐文字 baseline。

### 时间线

时间线：

- 比 marker 暗；
- 很细；
- 灰；
- marker 和线之间有一小段空气；
- 线的开始位置应在第一行思考正文**上方一点**，不要直接粘住圆点，也不要掉进第一行正文的字身中间。

## 7. 完成状态和生成中状态

这两个状态值得都保留。

### Android 新版 · 已完成

顶部：

```text
Worked for 32s
```

下面：

- 小圆点；
- 阶段标题；
- 思考正文；
- 时间线；
- 最后有带圆圈的勾 + `Done`。

### Android 新版 · 生成中

顶部直接显示当前思考标题。

特点：

- 顶部标题更强调；
- 下方仍出现阶段标题；
- 单阶段生成中只有一个点时，可以**没有时间线**；
- 不显示 `Done`。

### 一个方便的默认判断

如果有耗时：

```text
duration != null
```

默认视为已完成。

如果没有耗时：

```text
duration == null
```

默认视为生成中。

同时最好提供手动覆盖选项。

## 8. 浏览器端的一些区别

浏览器版和 Android 不是简单缩放。

浏览器端：

- marker 更像**思考小云朵**；
- 顶部会有 `Activity · 54s`；
- `Activity` 本身偏亮；
- 后面的时间偏灰；
- 下面有加粗的 `Thinking`；
- 阶段标题和正文仍然同字号；
- 已完成时有 Worked / Done 相关 footer。

所以建议每个平台有自己的 layout 参数，不要用一个万能 CSS 缩放所有平台。

## 9. 一个可以直接发帖的简短说明

可以自行改写：

> 做了一个 ChatGPT 思考过程截图渲染器，主要复刻浏览器端和 Android 的思考卡样式。
>
> 如果也想做，不需要部署我的页面：附件是一份“人类说明 + AI 施工规格”，直接整个丢给你常用的 coding AI，让它照着做一个本地 HTML 就行。
>
> 中文标点、英文整词换行、Android 小圆点 / 时间线位置这些比较容易翻车的地方也写进去了。
>
> 非官方组件，只是个人视觉复刻。

---

# Part B｜给 AI / coding agent 看的施工规格

## 10. 任务目标

实现一个**独立、本地可运行**的 ChatGPT-style 思考卡 PNG 渲染器。

首选交付：

```text
单文件 index.html
```

允许：

```text
index.html + renderer.js + style.css
```

但请避免：

- 后端服务器；
- 数据库；
- 登录；
- 第三方云 API；
- React / Vue 等大型依赖，除非使用者明确要求；
- 任何对 ChatGPT 网页 DOM 的依赖。

优先使用：

```text
HTML + CSS + Vanilla JavaScript + Canvas 2D
```

## 11. 功能要求

页面至少提供：

- 标题输入；
- 正文输入；
- 耗时输入（秒）；
- 样式 preset 选择；
- 状态选择：
  - 自动；
  - 生成中；
  - 已完成；
- 实时 Canvas 预览；
- 下载 PNG。

可选：

- 粘贴 Markdown；
- 从 `###` 自动拆多阶段；
- 语言选择；
- 背景色选择；
- PNG 文件名自定义。

## 12. 建议的数据结构

```js
{
  title: "Checking the current implementation",
  text: "Need to inspect the current renderer before making the next adjustment.",
  durationSec: 32,
  state: "auto",
  preset: "android-modern"
}
```

多阶段内部可转换为：

```js
[
  {
    title: "Checking the current implementation",
    text: "First stage body."
  },
  {
    title: "Refining the renderer",
    text: "Second stage body."
  }
]
```

## 13. 推荐 preset

```js
desktop-modern
web-mobile
android-modern
android-legacy
```

可选：

```js
ios-glass
```

如果实现 iOS，请标记为实验 / 设计版，除非使用者提供真实参考继续校准。

---

# Part C｜视觉参数参考

> 以下数值来自一次人工对照截图后的可用参考。
>
> 它们的意义是“帮助快速接近目标”，不是 OpenAI 官方设计 token。

## 14. 通用背景

推荐：

```text
#171717
```

Canvas 建议使用 2x 内部分辨率保证导出清晰：

```js
canvas.width = cssWidth * 2;
canvas.height = cssHeight * 2;
ctx.setTransform(2, 0, 0, 2, 0, 0);
```

后续绘制仍按 CSS px 计算。

## 15. 字体

### 浏览器

```css
ui-sans-serif,
system-ui,
-apple-system,
BlinkMacSystemFont,
"Segoe UI",
"Helvetica Neue",
Arial,
"PingFang SC",
"Microsoft YaHei",
sans-serif
```

### Android

```css
Roboto,
"Noto Sans SC",
"Noto Sans CJK SC",
"Microsoft YaHei",
Arial,
sans-serif
```

注意：

字体 fallback 会影响：

- 中文字宽；
- 行高视觉；
- 换行位置；
- 粗细；
- 标题居中。

因此不同电脑上无法保证像素级完全一致。

## 16. 浏览器 · 电脑端参考

画布宽：

```text
920px
```

关键尺寸：

```text
Activity       17.4px
Thinking       17.4px / semibold
阶段标题        16.2px
正文            16.2px
正文行高        25.5px
阶段标题行高     24px
```

推荐位置参考：

```text
Activity x ≈ 58
Activity baseline y ≈ 35

Thinking x ≈ 58
Thinking baseline y ≈ 73

timeline x ≈ 76
正文 x ≈ 104
右留白 ≈ 58

第一阶段起始 y ≈ 112
```

颜色关系：

```text
Activity             #f1f1f1 左右
Activity 后面的时间    #9f9f9f 左右
Thinking             #f1f1f1
阶段标题              #e7e7e7
正文                  #c7c7c7
marker                #d7d7d7
timeline              #666666 左右
```

marker：

```text
thought cloud / 思考小云朵
```

不要换成圆点。

## 17. 浏览器 · 手机端参考

画布宽：

```text
430px
```

关键尺寸：

```text
Activity       16.2px
Thinking       16.1px / semibold
阶段标题        15px
正文            15px
正文行高        23.8px
阶段标题行高     22.5px
```

参考位置：

```text
Activity x ≈ 20
Activity y ≈ 31

Thinking x ≈ 20
Thinking y ≈ 64

timeline x ≈ 29
正文 x ≈ 52
右留白 ≈ 20

第一阶段 y ≈ 96
```

## 18. Android App · 新版参考

画布宽：

```text
430px
```

顶部：

```text
top title        18.8px / bold
```

阶段区：

```text
阶段标题          17px
正文              17px
阶段标题行高       25.2px
正文行高          26.7px
标题与正文额外间隔 ≈ 1.5px
```

也就是说：

> **阶段标题和正文同字号。**

横向：

```text
timeline x ≈ 35
正文 x ≈ 63
右留白 ≈ 42
```

纵向参考：

```text
顶部标题 baseline ≈ 75
第一阶段起点 ≈ 120
```

底部：

```text
Done size ≈ 16.1px
底部留白 ≈ 20px
```

面板圆角：

```text
≈ 25px
```

### Android marker

使用：

```text
白色实心圆点
```

半径参考：

```text
≈ 3.8px
```

圆点视觉中心需要和**阶段标题字形中心**对齐。

### Android timeline

建议：

```text
颜色：#666666 左右
宽度：约 0.78px
```

重点：

- 不要和圆点粘住；
- 也不要空太远；
- 开始位置在第一行正文上方一点；
- 不应穿进第一行正文的字身中部。

## 19. Android App · 老版参考

宽：

```text
430px
```

参考：

```text
顶部标题      18.4px
正文          17px
正文行高      26.8px
Done          15.9px

正文 x        ≈ 56
右留白        ≈ 40
timeline x    ≈ 31

顶部标题 y    ≈ 71
正文起始 y    ≈ 106

底部留白      ≈ 21
面板圆角      ≈ 24
```

老版顶部文案更接近：

```text
Thought for 32s
```

或对应本地化版本。

---

# Part D｜文本排版规则

## 20. 英文换行算法

不要按字符直接 wrap 英文。

建议先 tokenize：

- English words；
- word + apostrophe；
- word + hyphen；
- 单词尾标点；
- CJK 单字符；
- 其他符号；
- 空白。

例如正则可参考：

```js
/[A-Za-z0-9]+(?:['’\-][A-Za-z0-9]+)*(?:[.,!?;:]+)?|[\u3400-\u9fff]|[^\sA-Za-z0-9\u3400-\u9fff]+|\s+/gu
```

基本算法：

1. 尝试把完整 token 加到当前行；
2. 超宽时先提交当前行；
3. token 放到下一行；
4. **只有 token 本身超过整行宽度时**，才按字符拆 token。

## 21. 中文行首禁则

以下 closing punctuation 不应独自成为下一行第一个字符：

```text
，。！？；：、％%）)]】》〉」』”’…,:;.!?
```

实现方式可以自由选择。

推荐直接在 wrap 算法中处理，而不是依赖 CSS。

一种思路：

当 candidate 超宽且下一个 token 是 closing punctuation 时，不立即断行；允许该标点“挂”在前一行末尾。

## 22. 中文行尾禁则

以下 opening punctuation 不应孤立在一行末尾：

```text
（【〔〖〘〚《〈「『“‘
```

如果某一行最后只能放进去一个 `“`，应该把这个 `“` 和后面的第一个正文字符一起移到下一行。

---

# Part E｜阶段和状态

## 23. 阶段解析

如果正文中不存在：

```markdown
### xxx
```

则视为单阶段：

```js
[
  {
    title: data.title || "Thinking",
    text: data.text
  }
]
```

如果存在三级标题，则拆分：

```markdown
### Choosing the next move
第一阶段正文

### Refining the answer
第二阶段正文
```

得到两个 stage。

## 24. 自动状态

```js
function resolveState(durationSec, override) {
  if (override === "in-progress") return "in-progress";
  if (override === "completed") return "completed";
  return Number.isFinite(Number(durationSec))
    ? "completed"
    : "in-progress";
}
```

## 25. Android 新版顶部文案

已完成：

```text
Worked for 32s
```

生成中：

```text
当前思考标题
```

顶部是强调层级。

下方阶段标题保持普通字重。

## 26. Browser 顶部层级

推荐：

```text
Activity · 32s

Thinking
```

其中：

```text
Activity
```

偏亮。

```text
· 32s
```

偏灰。

`Thinking` 加粗。

---

# Part F｜marker / timeline

## 27. Browser marker

浏览器固定使用：

```text
thought cloud
```

不要自动切圆点。

小云朵不需要完全复制某个官方 SVG；可以用 Canvas Bézier 做一个小型线框 thought-cloud，只要轮廓轻、尺寸克制即可。

## 28. Android marker

Android 固定：

```text
dot
```

不要换云朵。

## 29. timeline 的重要视觉规则

请重点遵守：

1. marker 和 timeline 不直接粘连；
2. gap 很小，但肉眼可见；
3. timeline 细、灰、弱于 marker；
4. Android marker 中心对应阶段标题视觉中心；
5. timeline 的第一段从正文第一行**上方一点**开始；
6. 单阶段生成中：
   - 有一个点；
   - 可以没有线；
   - 没有 Done；
7. 完成状态：
   - 有 timeline；
   - 有 completion UI。

## 30. 多阶段注意事项

目前可以先实现常规逻辑：

- 每阶段一个 marker；
- 阶段之间有 timeline；
- 每个阶段拥有自己的标题和正文。

但不要把这个当成最终 1:1 规则。

**真实 ChatGPT 多阶段思考有时看起来会表现成“两个点 + 分段线”的结构，具体线段断开与连接关系建议以后拿真实数据和截图再校准。**

因此代码应让 marker / line 位置参数化，避免写死成难以修改的一条长线。

---

# Part G｜动态高度

## 31. 不要固定 Canvas 高度

高度应根据：

- 标题行数；
- 正文换行数；
- 阶段数；
- footer；
- Done；
- bottom padding；

动态计算。

基本思路：

```js
contentHeight =
  topArea
  + sum(stageHeight)
  + optionalFooter
  + bottomPadding
```

先 measure，再设置 Canvas 高度，再正式 draw。

---

# Part H｜导出

## 32. PNG

推荐：

```js
canvas.toBlob(blob => {
  const url = URL.createObjectURL(blob);
  // create <a download>
}, "image/png");
```

不要靠 DOM screenshot library，Canvas 本身即可直接输出。

---

# Part I｜验收标准

## 33. 必测内容

请至少测试这些文本。

### 中文长句

```text
这里是一段比较长的中文思考摘要，用来检查换行后标点符号会不会自己跑到下一行最前面。
```

### 前引号

```text
接下来需要确认“这个引号不能自己挂在上一行末尾”是否正常。
```

### 中英混排

```text
需要检查 renderer: 这里的半角冒号也不能自己跑到下一行最前面。
```

### 英文

```text
The renderer should preserve complete English words instead of breaking implementation in the middle.
```

### 两阶段

```markdown
### Checking the current implementation

First inspect the existing renderer and layout values.

### Refining the final result

Then adjust only the remaining visual details.
```

## 34. 视觉验收

确认：

- [ ] 浏览器 marker 是云朵；
- [ ] Android marker 是圆点；
- [ ] 阶段标题与正文同字号；
- [ ] 阶段标题更亮，正文更灰；
- [ ] Android 圆点和阶段标题视觉中心对齐；
- [ ] marker 和线之间有小 gap；
- [ ] timeline 很细、很灰；
- [ ] timeline 起点略高于正文第一行；
- [ ] 英文完整单词换行；
- [ ] 中文 closing punctuation 不站行首；
- [ ] opening punctuation 不孤立行尾；
- [ ] 有 duration 时能显示完成状态；
- [ ] 无 duration 时能显示生成中；
- [ ] Canvas 高度随内容变化；
- [ ] PNG 清晰；
- [ ] 不依赖服务器即可使用。

---

# Part J｜可以直接复制给 AI 的 Prompt

## 35. 极简版

把本文件连同下面这句话一起发给 coding AI：

```text
请阅读我附上的《ChatGPT 思考卡渲染器复刻指南》。

按照其中 Part B–Part I 的规格，直接实现一个本地可运行的思考卡 PNG 渲染器。

优先交付单文件 index.html，使用 HTML + CSS + Vanilla JavaScript + Canvas 2D，不需要后端、数据库、登录或第三方 API。

请实现浏览器电脑端、浏览器手机端、Android 新版、Android 老版四个 preset；保留生成中 / 已完成状态；支持用 Markdown 三级标题拆阶段；正确处理英文整词换行和中文标点禁则；支持实时预览和下载 PNG。

不要只给伪代码或教程，请直接给完整可运行文件。
```

## 36. 如果你想让 AI 自己继续调像

可以再追加：

```text
实现后请把所有视觉参数集中放在 LAYOUTS / PRESETS 配置对象里，包括字号、行高、左右留白、marker 位置、timeline 位置、颜色和底部留白。

不要把这些数字散落在绘制函数里。

我之后会继续提供真实截图做像素级校准，因此代码必须方便单独调整某个平台，而不影响其他 preset。
```

---

# Part K｜最后说明

这份规格更适合作为：

```text
“让 AI 帮你快速做出类似效果的施工说明”
```

而不是：

```text
“OpenAI 官方前端实现解析”
```

这里的视觉参数来自人工参考和反复校准，不代表官方设计值。

目前最稳的是：

- 浏览器单阶段视觉；
- Android 新版单阶段视觉；
- 中文 / 英文换行；
- 完成 / 生成中状态差异。

仍建议根据真实截图继续微调：

- 不同系统字体 fallback；
- 多阶段 marker / timeline 的精确连接方式；
- iOS；
- 不同 ChatGPT 客户端版本。

如果只是想**做出一个很像的东西**，到这里已经足够让 coding AI 开工了。
