---
title: 常见问题：GStreamer软件源常见问题
author: kernel
date: 2024-02-06 15:41:00 +0800
categories: [gstreamer, 常见问题]
tags: [gst常见问题, gstreamer, 多媒体]
---

## 内容摘要：monorepo是怎么回事？

GStreamer 多媒体框架是由一系列库和插件组成的，这些库和插件分为多个不同的模块，独立发布，迄今为止一直在[freedesktop.org](https://gitlab.freedesktop.org/gstreamer/)GitLab 的不同git 仓库中开发。

除了这些独立的 git 仓库之外，还有一个`gst-build`元仓库，它将使用 Meson 构建系统的子项目功能下载每个单独的模块，然后一次性构建所有模块。它还提供了一个开发环境，便于在 GStreamer 上工作，使用或测试系统安装的 GStreamer 版本之外的其他版本。

现在（截至 2021 年 9 月 28 日），所有这些模块都已合并到一个单一的 git 仓库（"Mono 仓库 "或 "monorepo"）中，这将简化开发工作流程和持续集成，尤其是需要同时对多个模块进行更改时。

此次 mono 仓库合并将主要影响 GStreamer 开发人员和贡献者，以及所有基于 GStreamer git 仓库进行工作流程的人员。

由于 Rust 绑定和 Rust 插件模块的发布周期不同，因此目前尚未合并到 mono 代码库中。

mono 仓库位于现有 GStreamer 核心 git 仓库的新`"main"`分支中，今后所有的开发工作都将在这个`"main"`分支上进行。

## 我是一名贡献者，该如何处理 Gitlab 中的待处理合并请求？

由于无法通过按一个按钮就把合并请求移到另一个模块中，而且我们也无法在 gitlab 中以你的名义创建新的合并请求（MR），所以我们写了一个小脚本，你可以自己运行它，把现有的 MR 从其他模块移到主 gstreamer 模块的 mono 仓库中。

脚本位于[gstreamer 代码库中](https://gitlab.freedesktop.org/gstreamer/gstreamer/-/tree/main)，只需调用它（[`scripts/move_mrs_to_monorepo.py`](https://gitlab.freedesktop.org/gstreamer/gstreamer/-/blob/main/scripts/move_mrs_to_monorepo.py)）并按照说明操作，就能开始移动待处理的 MR。如果您已经有一个捡出，可能需要切换到`main`分支。

有了这个脚本，你就能将 GStreamer 各个模块中任何开放的 MR 移至 gstreamer mono 仓库。

脚本将保留讨论历史并链接到之前的 MR。

脚本还会检查是否已有 MR，以避免重复。

在运行脚本之前，首先需要进入 GitLab 的个人资料页面生成一个 gitlab 访问令牌：

<https://gitlab.freedesktop.org/-/profile/personal_access_tokens>

在此页面上，你需要给出一个**令牌名称**，授予访问权限（API、read\_user、read\_api），然后按下 _创建个人访问令牌_ 按钮。

你应该会在 _令牌名称_ 编辑框中看到一个值。该名称仅供您使用，您可以随意命名（如 "scripts"）。

![](/assets/img/2024-02-06/faq-monorepo-gitlab-personal-access-token-dialog.png)

有了类似 "edEFoqK3tATMj-XD6pY\_"这样的令牌后，就可以访问 gitlab API 并运行脚本了：

```shell
GITLAB_API_TOKEN=edEFoqK3tATMj-XD6pY_ ./scripts/move_mrs_to_monorepo.py
```

如果您有多个合并请求，可以先只运行一个合并请求，例如，如果您要移动用于测试的初始合并请求是

<https://gitlab.freedesktop.org/gstreamer/gst-plugins-base/-/merge_requests/1277>

你可以跑：

```shell
GITLAB_API_TOKEN=zytXYboB5yi3uggRpBM6 ./scripts/move_mrs_to_monorepo.py -mr https://gitlab.freedesktop.org/gstreamer/gst-plugins-base/-/merge_requests/1277
```

准备就绪后，也可以不带任何参数运行相同的脚本来浏览和移动所有合并请求：

```shell
GITLAB_API_TOKEN=zytXYboB5yi3uggRpBM6 ./scripts/move_mrs_to_monorepo.py
```

别担心，脚本在执行任何操作之前都会提示你输入信息。

### 我必须使用脚本吗？我不能直接打开一个新的 MR 吗？

脚本将移动现有的讨论和评论。这对于已经审核过并有开放讨论项的 MR 尤其有用。这可以确保我们不会在有未决问题的情况下意外合并某些内容，如果你只是提交了一个新的 MR 并关闭了旧的合并请求，我们是不会知道的。

如果现有的合并请求还没有任何审核意见，或者只是一些简单的请求，那么只需在 mono 仓库中打开一个新的 MR，然后关闭旧的 MR 就可以了。

### GStreamer 资源库中的现有 MR 如何处理？

任何针对 GStreamer 核心仓库的现有 MR 都是针对 git`"master"`分支的。

您不需要使用任何脚本来更新这些合并请求，只需将它们重置到`"main"`分支之上即可。

您可以通过 GitLab 用户界面编辑问题，然后更改目标分支。

## 我是一名贡献者--我有一个 gst-plugins-XX 或其他模块的分支，但尚未向上游提出，如何才能将其重新发布到`gstreamer`软件源上？

我们在`gstreamer`软件源中提供了[scripts/rebase-branch-from-old-module.py](https://gitlab.freedesktop.org/gstreamer/gstreamer/-/blob/main/scripts/rebase-branch-from-old-module.py)脚本，你可以用它将旧版`GStreamer`模块软件源中的分支重植到主`gstreamer`软件源中。

这个脚本只会修改本地 gstreamer mono 仓库的捡出，不会上传任何东西到 GitLab，当然也不会创建任何合并请求。运行它甚至不需要 GitLab 账户。脚本的使用方法如下：

```shell
./scripts/rebase-branch-from-old-module.py https://gitlab.freedesktop.org/user/gst-plugins-bad my_wip_dev_branch
```

## 我使用或分发发行版压缩包 - 这对我有什么影响？

我们将继续以 tar 包的形式单独发布各种 GStreamer 模块，因此如果你只使用 tar 包，转用 mono 版本库应该不会有任何影响。

## 我使用或分发发行版压缩包，但不想再为所有这些单独的模块压缩包费心了--monorepo 会为我做什么吗？

今后（>= v1.19.3），您只需在 gitlab 中为[release 标签](https://gitlab.freedesktop.org/gstreamer/gstreamer/-/tags)下载 mono repo 压缩包，其中包含的所有模块将与 git 中的 monorepo 相同。

## 我使用的是 gst-build，现在应该使用什么？

简而言之，`gstreamer`资源库现在已成为过去的`gst-build`，唯一不同的是，所有主要 GStreamer 模块的代码现在都已在子项目文件夹下的资源库中，无需在构建过程中通过 Meson 下载。

只需`git clone https://gitlab.freedesktop.org/gstreamer/gstreamer.git`并从那里开始工作。它应该与您使用的 gst-build 非常相似。

`gst-build`已死，`gstreamer` 万岁！

## 我使用 cerbero - 这对我有什么影响？

如果您使用 cerbero，这不会对您造成影响。

Cerbero 已经更新，现在可以从 mono 代码库中构建最新、最好的 GStreamer。

如果你使用 cerbero 的`"master"`分支从 git master 构建 GStreamer，现在需要切换到`"main"`分支。今后所有的 cerbero 开发都将在`"main"`分支上继续，`"master"`分支在 1.19.2 版本中被冻结，今后可能会被移除。

cerbero 的稳定分支将一如既往地构建稳定版 GStreamer。

## 如果 GStreamer 现在是 gst-build，那么 GStreamer 核心库代码去哪儿了？

它还在，只是移到了`subprojects/gstreamer/`目录下。

## 这是否意味着所有提交哈希值都发生了变化？如何验证历史记录？

所有模块均按原样导入，历史记录和提交哈希值保持不变。

## 这是否意味着我们现在可以轻松地在模块间进行 git 切换？

为什么不呢？至少在未来，git 分叉到 pre-monorepo 历史是行不通的。

## 是否已将所有模块移至 mono 仓库？

不，Rust 绑定模块和 Rust 插件模块暂时是分开的，因为它们的发布周期不同。

Cerbero（我们用于 Windows、Android、macOS 和 iOS 的软件包生成器）也保留在自己的资源库中。

## 我现在应该在哪里提交新的合并请求？

请在`main`分支之上的[gitlab](https://gitlab.freedesktop.org/gstreamer/gstreamer)中提交所有针对gstreamer模块的新合并请求。

## 我现在应该在哪里提交新问题和功能请求？

请针对 gitlab 中的[gstreamer](https://gitlab.freedesktop.org/gstreamer/gstreamer)模块提交所有新问题和功能请求（最好先检查现有问题）。

## 我在 GitLab 中有一些开放的问题，有脚本可以移动这些问题吗？

没有，在接下来的几周或几个月内，GStreamer 开发人员会移动问题，如果你认为这些问题仍然适用于最新版本的 GStreamer，也可以自己移动。

如果是你自己的问题或你是 GStreamer 开发人员，右下方应该有一个`Move issue`按钮。

请将问题移至`gstreamer/gstreamer`模块。

在移动问题之前，请确保问题摘要行有一个适当的前缀，以便其他人在不查看标签或问题描述的情况下，也能轻松查看该问题与 GStreamer 的哪个组件有关（例如插件名称或辅助库）。如有需要，请在移动问题前编辑问题以调整摘要行。

## 在其他模块中，GitLab 中的所有开放问题和合并请求会如何处理？

在接下来的几周和几个月里，我们将齐心协力检查和分流所有开放问题以及作者尚未提交的合并请求。

[GStreamer](https://gitlab.freedesktop.org/gstreamer/gstreamer)开发者和维护者在遇到问题时，可随时使用 Gitlab 中的 `Move issue`按钮将其转移到gstreamer模块中。

## 你为什么不把所有的公开问题都集中起来？

GitLab 有几千个开放问题，几百个开放合并请求。

大量转移所有这些问题和 MR 只会导致几千封电子邮件通知，这些通知会被立即删除，没有人会去看这些问题。

mono 仓库迁移提供了一个独特的机会，让我们可以系统地审查所有开放问题和合并请求，并剔除任何看起来不再相关或有用的内容。

当然，我们仍可能决定在未来某个时候自动移动任何剩余的未决问题。

## 其他模块的现有 git 仓库会发生什么变化？

当然，GitLab 中现有的所有 git 仓库都将保留。

不过，我们可能会为新问题和合并请求关闭它们，以确保所有新问题和合并请求都是针对 mono 代码库提交的。

一旦我们处理完剩余的未决问题和合并请求，我们也可能在将来的某个时候将这些模块归档。不过，这只会影响模块在 GitLab 中的可见性：在这种情况下，问题、MR 和 git 仓库的 URL、分支和标签等仍会像以前一样工作。

## 所有模块中现有的 "master"分支会发生什么变化？

暂时保留所有 "master"分支，但冻结在 1.19.2 版本上。

如果你有使用 "master"分支的脚本或工作流程，你应该更新它们以使用新的 mono 版本库，或者将它们固定在 1.19.2 标签上。

我们可能会在未来某个时候移除 "master"分支。

## 我有从 git 构建 GStreamer 模块的工作流程，但不想使用 gst-build 类型的元构建设置，现在该怎么办？

你仍然可以单独构建每个 GStreamer 模块，方法是更改为`subprojects/$gst_module`目录，然后将该目录作为源代码目录运行 Meson。

## 我想从上游 mono 代码库中挑选一个补丁到我的 <= 1.18 代码库中：

由于子树在 monorepo 组织（即子项目/gst-pluigns-xx）中发生了变化，您可以尝试使用以下 git 命令来解决路径冲突：

```shell
$ git cherry-pick <commit> ... --strategy=subtree
```

## 我还有一个与 mono 代码库有关的问题——在哪里询问或寻求帮助最合适？

最好直接进入我们在 OFTC 网络上的 IRC 频道`#gstreamer`（也可通过 Matrix 访问），或向 gstreamer-devel 邮件列表发送邮件。

如果您有值得添加到本常见问题集的问题，也可以在 GitLab 中提交问题。

___

_本常见问题解答由 Thibault Saunier 和 Tim-Philipp Müller 编写，Mathieu Duponchelle 为其提供了帮助。_

_许可协议：[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)_
