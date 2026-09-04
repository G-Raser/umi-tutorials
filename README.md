# Umi Tutorials

一组由 **Umi & CatTea** 共同整理、持续维护的实用教程与实现笔记。

这里主要放“怎么做”的公开教程：尽量保留可复现步骤、最小示例、踩坑记录和必要的架构说明。私人项目、生产环境配置、真实密钥、内部接口与完整私有源码不会因为教程整理而公开。

## Tutorials

### Available

- [ChatGPT 思考卡渲染器复刻指南](tutorials/chatgpt-thinking-card/README.md)  
  人类说明 + AI 施工规格。用于制作接近 ChatGPT 浏览器端 / Android 思考过程样式的本地 PNG 渲染器。

- [ChatGPT 多窗口 Bridge：会话隔离、Listener 接管与顺序投递](tutorials/chatgpt-multi-window-bridge/README.md)  
  略过基础 Bridge 搭建，重点整理多窗口长期使用后真正需要处理的 session / binding 隔离、listener 接管、同会话顺序、跨会话并行和回复持久化。

更多 Umi / CatTea 工具教程以后按实际完成情况逐步加入。

## Repository structure

```text
umi-tutorials/
├── README.md
└── tutorials/
    ├── chatgpt-thinking-card/
    │   └── README.md
    ├── chatgpt-multi-window-bridge/
    │   └── README.md
    └── ...
```

每个教程尽量独立放在自己的目录中。需要示例代码、配置片段或图片时，和对应教程放在一起，避免所有附件堆进同一个目录。

## Publishing principles

- 教程可公开，私人生产项目继续保持 private。
- 公开示例优先使用最小、去隐私化、可复现版本。
- 不公开真实 token、密钥、私人 prompt、内部域名、生产服务器细节或个人数据。
- 官方文档与实际测试结果冲突时，会明确区分“官方说明”和“实测结果”。
- 教程会随着软件版本和实测结果继续修订。

## Reuse & attribution

本仓库的教程正文默认采用 **CC BY-NC-SA 4.0**（Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International）许可，除非某个文件或教程单独标注了其他许可。

转载、翻译、节选或二次整理时，请：

- 保留 **Umi & CatTea** 的原作者署名；
- 附上原仓库或原教程链接；
- 如果修改、翻译或重新组织了内容，请明确说明做过修改；
- 不要把本教程内容直接用于商业转载、付费分发或其他商业用途；
- 基于本教程正文制作并公开分发的改编版本，请沿用相同许可。

较完整的可运行代码、独立软件项目或第三方代码如果需要其他软件许可证，会在对应目录中单独标注；其许可优先于本节说明。

详见 [LICENSE.md](LICENSE.md)。

## Authors

**Umi & CatTea**

部分项目、代码与参考实现可能有各自独立的原作者或仓库；具体归属以对应教程中的说明为准。
