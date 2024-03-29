---
title: 常见问题：GStreamer应用程序和插件的许可
author: kernel
date: 2024-02-06 15:31:00 +0800
categories: [gstreamer, 常见问题]
tags: [gst常见问题, gstreamer, 多媒体]
---

本文件是 GStreamer 社区内部以及与社区外利益相关者多次讨论的结果。其中包括与 FSF 官方代表在内的律师讨论的结果，以帮助我们确保尽可能正确地涵盖法律问题。这并不意味着 FSF 或其他任何人认可本页中的观点。这些意见仅代表 GStreamer 社区的粗略共识。这里所包含的建议旨在为使用 GStreamer 库开发自由和开放源码软件的人们提供信息和指导，以便他们了解其选择的后果。开发专有软件或发布 GStreamer 的用户也可参考本文档，以了解 GStreamer 如何在许可范围内工作。

这篇文章也是为了说明我们对如何处理软件专利问题的一点想法，在多媒体领域，软件专利是比其他编程领域更令人头疼的问题。

有关许可的更多信息，请查看我们的[法律常见问题](../GStreamerLegalIssues/)

## 为 GStreamer 自身贡献的代码颁发许可证

GStreamer 是一个基于插件的框架，采用 LGPL 许可。之所以选择这种许可方式，是为了确保每个人都能使用 GStreamer 构建应用程序，并使用自己想要的许可方式。

为了保持这一政策的可行性，GStreamer 社区为 GStreamer 内核或 GStreamer 官方模块（如我们的插件包）中的代码制定了一些许可规则。**我们要求所有进入核心软件包的代码都必须使用 LGPL** 协议。对于插件代码，我们要求**所有从零开始编写的插件或链接到外部库的插件使用 LGPL**。唯一的例外是当插件包含 BSD 和 MIT 许可下的旧代码时。我们希望所有新编写的代码至少使用 LGPL 双重许可。我们不接受将 GPL 代码添加到我们的插件模块中，但我们接受在某些插件模块中使用外部 GPL 库的 LGPL 许可插件。我们之所以要求插件使用 LGPL 许可，即使它们使用的是 GPL 库，是因为其他开发者可能希望将插件代码用作链接到非 GPL 库的插件模板。我们也接受双重许可的插件，只要为双重许可提供的许可之一是 LGPL 即可。

我们也不允许任何许可证下的插件进入我们的核心、基础或优秀软件包，如果这些插件存在已知的专利问题的话。 这意味着，即使是 LGPL/MIT 许可的贡献实现，如果有许可证机构要求付费，这些插件也需要进入我们的 gst-plugins-ugly 模块。

所有新插件，无论是否获得许可或专利，都必须在我们的孵化模块 gst-plugins-bad 中经历一段时间，然后才能转入 ugly、base 或 good。

## 使用 GStreamer 许可应用程序

GStreamer 的授权与 GTK+ 或 glibc 等其他库并无不同：我们使用的是[LGPL](https://www.fsf.org/licenses/lgpl.html)。GStreamer 的复杂之处在于其基于插件的设计，以及许多多媒体编解码器的专利和专有性质。虽然目前世界上只有少数国家（美国和澳大利亚是其中最重要的国家）允许软件专利，但问题是，由于美国在世界经济和计算机行业中的核心地位，无论在哪里，软件专利都很难被忽视。

在这种情况下，许多公司，包括主要的 GNU/Linux 发行版，都陷入了这样的困境：要么因为缺乏开箱即用的媒体播放功能而获得差评（迄今为止，试图教育评论者的努力收效甚微），要么违背了自己和自由软件运动避免使用专有软件的愿望。迫于竞争压力，大多数公司选择增加一些支持。如果通过纯粹的自由软件解决方案来这样做，他们将冒着被专利所有者起诉和惩罚的巨大风险。因此，当决定加入对专利编解码器的支持时，他们只能选择使用专门的专利应用程序，或者尝试通过专利插件将对这些编解码器的支持集成到 GStreamer 提供的多媒体基础架构中。两害相权取其轻，GStreamer 社区当然更倾向于第二种选择。

现在出现的问题是，大多数自由软件和开源应用程序都使用 GPL 作为许可证。虽然这通常是件好事，但却给那些想制作发行版的人带来了困境。他们面临的困境是，如果在 GStreamer 中加入专有插件，以合法的方式支持专利格式，就有可能违反应用程序的 GPL 许可证。关于这是否真的是个问题，我们从律师那里得到了一些相互矛盾的报告，但 FSF 的官方立场是这是个问题。我们认为 FSF 是这方面的权威，因此我们倾向于遵循他们对 GPL 许可证的解释。

那么，作为应用程序开发人员，这意味着什么呢？嗯，这意味着**你必须积极决定是否希望你的应用程序与专有插件一起使用**。你在这里做出的决定也将影响商业发行版和 Unix 供应商推出你的应用程序的机会。GStreamer 社区建议你使用允许非自由、专利实施或非 GPL 兼容插件与 GStreamer 和你的应用程序捆绑的许可证来授权你的软件，以确保尽可能多的供应商使用 GStreamer 而不是不那么自由的解决方案。反过来，我们希望并认为这将使 GStreamer 成为更广泛使用[Xiph.org](https://www.xiph.org/)等免费格式的工具。

如果您决定允许在应用程序中使用非自由插件，您有多种选择。最简单的方法之一是在应用程序中使用 LGPL、MPL 或 BSD 等许可证，而不是 GPL。或者，你可以在 GPL 许可证中添加一个例外条款，说明 GStreamer 插件不受 GPL 的约束。

以 Totem 视频播放器项目为例，GPL 例外条款就是一个很好的例子：

_Totem 视频播放器的开发者特此允许非 GPL 兼容的 GStreamer 插件与 GStreamer 和 Totem 一起使用和发布。此许可高于 Totem 所适用的 GPL 许可。如果你修改了此代码，你可以将此例外扩展到你的代码版本，但你没有义务这样做。如果你不想这样做，请从你的版本中删除此例外声明。_

在这些选择中，我们的建议是使用 LGPL 许可证，因为它与 GPL 最为相似，而且与主要的 GNU/Linux 桌面项目（如 GNOME 和 KDE）有很好的许可证契合度。它还允许你更开放地与拥有兼容许可证的项目共享代码。正如你可能推断的那样，没有上述条款的纯 GPL 许可代码是不能在 GPL 加例外条款下在你的应用程序中重复使用的，除非你让纯 GPL 代码的作者允许重新许可为 GPL 加例外条款。如果选择 LGPL，就不需要例外条款，因此代码可以在您的应用程序和其他使用 LGPL 的项目之间自由共享。

我们在上文概述了 GStreamer 社区建议您允许在应用程序中使用非自由插件的实际原因。我们认为，在多媒体领域，自由软件社区的力量还不足以制定议程，阻止在我们的基础设施中使用非自由插件对我们的伤害大于对专利所有者及其同类的伤害。

并非所有人都赞同这一观点。[自由软件基金会](https://www.fsf.org/)敦促您为您的应用程序使用未经修改的 GPL，以抵制使用非自由插件的诱惑。他们说，既然不是每个人都有能力因为这些插件不道德而拒绝它们，那么他们就需要你的帮助，给他们一个这样做的合法理由。
