# Announcing the Tokio Doc Push (we need you!)

过去，一直有反馈说 Tokio 很难理解。我认为缺乏良好的文档是造成这一问题的一个重要原因。是时候解决这个问题了。

而且因为 Tokio 是开源的，所以我们（社区）有责任实现这一点！👏

但别担心，这不是一个漫无目的的文档贡献请求。然而，它确实需要参与。无论之前有何种程度的 Tokio 经验，都有办法参与其中。

## The Tokio documentation push

以下是计划。已设置临时存储库 [doc-push](http://github.com/tokio-rs/doc-push)，文档工作将在此进行协调。README 中包含了入门步骤。大致而言，该过程如下：

1. 阅读现有文档，追踪令人困惑或未解答的问题的部分。
2. 在 doc-push 存储库上打开一个 issue 来报告困惑。
3. 修复该 issue。
4. 重复。

其中“修复问题”是修复现有指南或编写新指南。

## Writing new guides

为了引导编写新指南的努力，doc-push 存储库中已经植入了一个[大纲](https://github.com/tokio-rs/doc-push/blob/master/outline/README.md)，代表了我对指南应如何构建的最佳猜测。

任何人都可以自愿根据此大纲编写一页。只需提交 PR 并在页面旁边添加您的 Github handle 即可。例如，如果您想自愿编写有关超时的指南，您可以提交 PR 来更新该[部分](https://github.com/tokio-rs/doc-push/blob/master/outline/tracking-time.md#timeouts)，将状态：**Unassigned**  更改为状态： **Assigned (@myname)**。

此外，非常感谢您对大纲结构的反馈和建议。请针对大纲打开 issue 和 PR。

还有一个新的 [Gitter](https://gitter.im/tokio-rs/doc-blitz) 频道专门用于文档推送。如果你想参与其中，但需要一些指导。加入频道并联系我们。也许您想尝试编写指南，但还不太能胜任这项任务。联系我们，我们会帮助您完成。

## An experiment

文档推送是一个实验。我不知道它会如何进行，但我希望它会成功。

这也是一项反复的努力。一旦指南得到改进，我们就需要回到步骤 1)，让新来者尝试使用新文档学习 Tokio。这将暴露出需要解决的新差距。

## FAQ

**我还不了解 Tokio！**

首先，这不是一个问题。

第二，太棒了！你就是我们希望参与的人。我们需要新的视角来审查指南并报告他们遇到的问题。

您可以做的事情：

- 阅读现有指南并报告 issue。
- 集思广益，为 [cookbook](https://github.com/tokio-rs/doc-push/issues/23) 或 "[gotchas](https://github.com/tokio-rs/doc-push/issues/14)"部分提出建议。
- 审查并向网站提供有关 [PR](https://github.com/tokio-rs/website/pulls) 的反馈。

**我想贡献指南，但我不确定我是否能够**

仍然不是一个问题！

你知道他们说的：“教学是最好的学习方式”。这是学习 Tokio 的好机会。首先，有一些大纲可以帮助你入门。这些大纲可能会导致你产生疑问或需要一些指导。在编写指南之前，也许您需要先了解该主题！

我们在 [Gitter](https://gitter.im/tokio-rs/doc-blitz) 频道热切地等待着为您提供帮助。我们将帮助您学习所需的知识。作为交换，您将贡献一份指南😊。

这是您帮助让 Tokio 更易于学习的机会。如果您不自愿参与文档推送，这将不会发生。简而言之：

1. 加入 [Gitter](https://gitter.im/tokio-rs/doc-blitz)。
2. 观看 [repo](https://github.com/tokio-rs/doc-push)。
3. 参与其中！
