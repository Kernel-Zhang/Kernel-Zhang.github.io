---
title: 标记异常，使错误处理更顺畅
author: kernel
date: 2023-11-01 20:34:00 +0800
categories: [Nim语言, 博客翻译]
tags: [Nim语言, 编程技巧]
---

错误处理，一个有争议且经常讨论的话题。每种语言都有一些处理错误的方法，无论是错误代码、结果类型、异常，还是完全不同的方法或混合方法。几乎从我们开始编写程序起，如何正确处理程序中出现的错误就变得非常重要。有关这方面的论文、讲座层出不穷，还有无数的库被编写出来，将一种语言处理错误的方法引入到其他语言中。

但这并不是一篇介绍所有可能的错误处理方法的文章。相反，我尝试了一种对常见的异常处理方式的补充。你可能知道，异常是一种特殊的控制流机制，在这种机制下，过程可以说出错了，以至于无法返回任何有意义的信息，程序应该以不同的方式来处理。例如，如果我们试图向硬盘写入数据，但硬盘已满，我们可能会收到一个IO错误，告诉我们写入不可能成功，必须以另一种方式处理。这些错误很善于告诉我们*为什么*出错，但不善于告诉我们出错的*原因*。我的意思是，它们会告诉我们写操作失败的原因是IO错误（在某些情况下甚至是更具体的类型），但不会告诉我们是哪个写操作失败了。堆栈跟踪会告诉我们异常发生在代码的哪个部分，但遗憾的是，我们在处理异常时很少能获得这些信息。为了提供细粒度的错误信息，这意味着代码的每一部分都必须封装在自己的`try/except`部分中。

> `try/except`是Nim的术语，大多数语言称之为`try/catch`。

在许多语言中，这也意味着需要先在`try`代码块外声明变量，然后在`try`代码块内赋值，之后再使用变量。这还会带来一个隐患，即在错误处理部分忘记返回，从而导致代码继续处于错误状态。这也意味着，我们最终需要对错误处理进行大量复制/粘贴，从而增加了重构的难度，并使代码变得更加混乱。让我们想象一下网络服务器中的一个简单流程：一个用户进来，想要获取最新的新闻和相关事件：

```nim
let
  user = getUser(userInfo)
  news = getNewsForUser(user.id)
  relatedNews = getRelatedNews(news)
return %*{"data": {"news": news, "relatedNews": relatedNews}}
```

流程简单明了，现在让我们添加一些细粒度的错误处理：

```nim
var user: User
try:
  user = getUser(userInfo)
except CatchableError as e:
  echo "Cannot get user: " & e.msg
  return %*{"error": "Cannot get user " & userInfo.name}
var news: seq[News]
try:
  news = getNewsForUser(user.id)
except CatchableError as e:
  echo "Cannot get news for user: " & e.msg
  return %*{"error": "Cannot get news for user " & userInfo.name}
var relatedNews: Table[NewsId, seq[News]]
try:
  relatedNews = getRelatedNews(news)
except CatchableError as e:
  echo "Cannot get related news for user: " & e.msg
  return %*{"error": "Cannot get related news for user " & userInfo.name}
return %*{"data": {"news": news, "relatedNews": relatedNews}}
```

我们的代码不仅从5行增加到了19行，而且现在也很难一眼看出这些代码是做什么的，因为大部分都是错误处理。如果我们改用粗略的错误处理，效果会更好：

```nim
try:
  let
    user = getUser(userInfo)
    news = getNewsForUser(user.id)
    relatedNews = getRelatedNews(news)
  return %*{"data": {"news": news, "relatedNews": relatedNews}}
except CatchableError as e:
  echo "Cannot get content for user: " & e.msg
  return %*{"error": "Something went wrong fetching content for user " & userInfo.name}
```

虽然只有9行，流程也很清晰，但错误信息的细节却少了很多，这让我们为用户提供支持的工作难度大大增加。这里的问题是，所有这三个语句都可能以相同的错误失败，如果我们没有网络，可能是IO错误，如果我们需要刷新API访问令牌，可能是令牌过期错误，或者任何其他粒度的错误情况。这意味着，拥有更多的`except`块可以为我们提供更多关于代码失败*原因*的信息，但并不能说明代码的哪个部分确实失败了。

## 带标签的异常

我所尝试的实验是一种给语句贴标签的方法，这种方法可以在错误处理程序中提供对异常负责的语句。这样，我们就能在错误信息中添加出错*原因*的_下文信息，同时让代码保持可读性、安全性和可重构性。让我们在粗略的错误处理示例中标注我们的语句：

```nim
labeledTry:
  let
    user = getUser(userInfo) |> UserErr
    news = getNewsForUser(user.id) |> NewsErr
    relatedNews = getRelatedNews(news) |> RelatedErr
  return %*{"data": {"news": news, "relatedNews": relatedNews}}
except CatchableError as e:
    const msgs: array[Label, string] =
      [NoLabel: "content", UserErr: "user details", NewsErr: "news", RelatedErr: "related news"]
  echo "Cannot get " & msgs[getLabel()] & " for user: " & e.msg
  return %*{"error": "Cannot get " & msgs[getLabel()] & " for user " & userInfo.name}
```

我们为代码中可能出现故障的部分添加了后缀 `|> LabelName`，并为每个部分分配了一个标签。然后，我们在异常处理程序中设置了一个数组，其中包含映射到每个可能标签的信息。请注意，由于我们指定了这是一个由`Label`和`string`组成的数组，因此我们知道该数组必须涵盖所有错误信息，并为每个信息分配一个字符串。下面的数组列出了枚举的每一个值和字符串，虽然有点冗长，但却避免了我们不小心对它们进行重新排列。然后，我们为日志行和用户错误挑出相关信息，确保我们的错误信息中包含出错的实际情况。

原始流程非常简洁，只有小的标签后缀。现在，错误处理可以区分所有贴有标签的错误起源和在未贴标签位置抛出异常的特殊`NoLabel`情况。这就为我们提供了最后一条关键信息，使我们能够构建良好的错误信息。由于标签是以枚举的形式创建的，因此我们可以保证用`case`语句（或如上所示由标签索引的数组）涵盖所有情况，因此在try代码块中添加或删除标签都会在异常处理程序中产生编译错误。这也让读者清楚地看到每个错误在哪里以及如何处理。这在普通的异常处理代码中很难做到，因为异常是在过程中定义的，不通过特别的工具一般是看不到的。

如果我们有不止一条语句可以抛出异常，而所有异常都与同一件事有关，那么我们还可以使用标签系统的块变体，即使用一个标签和一个代码块，该代码中抛出异常的每一部分都会被贴上相同的标签。这样，我们就可以根据自己的喜好控制异常标签的粒度。

## 结论

总之，我认为这种标记或标签系统可以为典型的异常系统提供非常有趣的补充。这里的语法是出于测试目的而任意选择的，目前的实现可能并不完美，但重要的是功能本身。如果你想玩一玩，可以在[我的GitHub](https://github.com/pmunch/labeltry)上找到Nim的代码库。此外，还要衷心感谢Nim社区中的用户ElegantBeef，在他的帮助下，我们找到了实现该功能的方法。

这是我喜欢Nim编程语言的众多原因之一。除了它编写起来非常容易、运行速度非常快以及几乎可以在任何地方运行之外。它通过宏系统提供的这种灵活性，使得像这样作为简单库编写语言实验成为可能，这一点令人震惊。你可以建立自己的专用语法，并根据自己的需要改进语言，这是一种巨大的能力。当然，能力越大，责任也就越大，尤其是在与他人合作时，切勿过度使用这类东西。不过，在需要的时候，这确实是一个很好很强大的工具。如果你想了解更多关于元编程的信息，我[这里](https://peterme.net/metaprogramming-and-read-and-maintainability-in-nim.html)有一篇较旧但仍然有效的文章。请注意，文章中提到的项目刚刚进行了第三次修订，而文章中开发的宏仍然在完美地完成它的工作！