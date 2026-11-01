# dsh-superagent

> **研究原型，仅供参考，并非 DeepSeek 官方产品。** 这份代码源自我们团队的日常开发工作，其中大部分工程由编码智能体完成。请把它当作研究代码，而不是可直接投产的软件。它可能仍含有缺陷，编码风格与工程实践也可能并不统一。在你自己的环境中使用之前，请先审阅并做相应调整。
>
> 为便于公开发布，我们做了改编，因此它既不同于我们的内部版本，也不同于《Exploring Super-agent Design: Towards a More Autonomous Agent System》（探索超级智能体设计：走向更自主的智能体系统）一文所描述的实现。文章中讨论的部分设计属于未来发展方向，本原型并未实现。

dsh-superagent 是一个面向单个 GitHub 仓库的主动式编码智能体，以 bundle 的形式打包，供 [DeepSeek Harness](https://www.deepseek.com/en/harness/) 使用。它监听 GitHub，并可选监听飞书。当获得许可的人向智能体提出工作请求时，它会开立一个工作项，在属于自己的智能体会话和目录中处理这项请求，并在原始对话中作答，例如提交一个 pull request。Web 面板则展示哪些事项需要人来处理。

它同时也是一种与智能体协作的工作方式：**尽可能少占用人的注意力，尽可能多地交付价值。** 一个能为整个团队行动的智能体，也可能浪费整个团队的注意力。因此，它只在被请求时、或在团队写下的规则允许时才行动；只在有人必须采取行动、或某个结果对人有用时才开口。[docs/000-philosophy-zh.md](docs/000-philosophy-zh.md) 阐述了这一原则，以及它对运营者提出了什么要求。

## 安装配置

安装步骤较多，因此由智能体与你一起完成。请对你的编码智能体说：

> Read SETUP.md in this repository and set up dsh-superagent with me.
>
> （译：阅读本仓库中的 SETUP.md，和我一起安装配置 dsh-superagent。）

它会一次只问一个简短的问题，先完整展示一遍计划，然后安装并配置好全部内容。它所遵循的就是 [SETUP.md](SETUP.md)。

## 文档

请按顺序阅读这些带编号的文档；每一篇只建立在它之前那些文档之上。

- [000 设计理念](docs/000-philosophy-zh.md)
- [001 领域模型](docs/001-domain-zh.md)
- [002 授权](docs/002-authorization-zh.md)
- [003 执行](docs/003-execution-zh.md)
- [004 提示词](docs/004-prompts-zh.md)
- [005 面板](docs/005-panel-zh.md)
- [006 部署](docs/006-deployment-zh.md)：手动安装、配置、升级与故障排查
- [007 架构](docs/007-architecture-zh.md)
- [008 实操手册](docs/008-cookbook-zh.md)：新增一个来源

## 开发

`pnpm check` 会运行类型检查、测试和构建。

## 许可证

[MIT](LICENSE)
