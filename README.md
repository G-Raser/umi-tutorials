# Umi Tutorials

一组由 **Umi & CatTea** 共同整理、持续维护的实用教程与实现笔记。

这里主要放“怎么做”的公开教程：尽量保留可复现步骤、最小示例、踩坑记录和必要的架构说明。私人项目、生产环境配置、真实密钥、内部接口与完整私有源码不会因为教程整理而公开。

## Tutorials

### Available

- [ChatGPT 思考卡渲染器复刻指南](tutorials/chatgpt-thinking-card/README.md)  
  人类说明 + AI 施工规格。用于制作接近 ChatGPT 浏览器端 / Android 思考过程样式的本地 PNG 渲染器。

### Planned / in progress

- **ChatGPT 多窗口分离与前端 Bridge 连接**  
  讲清楚 binding、listener、会话隔离、多窗口并行、前端发送与回传的整体结构，并提供去隐私化的最小可复现示例。

- **AI companion time anchor / 时间锚点**  
  整理让 AI 获取稳定时间上下文的实现思路、部署方式与常见问题。

- **更多 Umi / CatTea 工具教程**  
  以后按实际完成情况逐步加入，不为了凑目录提前写空壳。

## Repository structure

```text
umi-tutorials/
├── README.md
└── tutorials/
    ├── chatgpt-thinking-card/
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

## Authors

**Umi & CatTea**

部分项目、代码与参考实现可能有各自独立的原作者或仓库；具体归属以对应教程中的说明为准。
