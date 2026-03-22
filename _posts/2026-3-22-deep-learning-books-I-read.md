---
layout: post
title: 最近半年读过的深度学习相关的书
category: AI
---

从 2022 年 11 月，ChatGPT 横空出世以来，我一直对 AI 不太感兴趣，虽然 AI 的表现出了足够的「智能」，但是使用 AI 来做一些事情感觉还是很遥远，即使 2024 年的明星 AI 产品的 Cursor，也没感觉他和 IDEA 有多大不同。但是 2025 年年初开始，从 Manus 到 Claude Code，不断刷新我对 AI Agent 的认知，所以我开始学习 AI Agent 相关的技术。最开始我会尝试跑 Agent 的 demo 代码，想要看懂如何实现一个 Agent，比如 Qwen-Agent 中的自然语言生成图片的 demo，随着对 Agent 的认识逐渐加深，就开始看一些复杂并且在日常工作中会用到的产品代码。比如又看了 VS Code 的代码插件 CodeWebGo，了解这种产品如何与模型交互，如何生成代码, 然后看到基于 Gemini Cli 的 Qwen Code 的代码开源，就尝试去看这块的代码，了解 cli 的 Agent 的原理。这类 Agent 的原理了解以后，感觉主要的活还是模型提供的，所以很好奇模型是如何实现的。从去年国庆节假期决定开始学习 AI 和深度学习的原理，先后看了几本书，目前总算对深度学习有了一些认识。先通过这篇文章把这几本书都列出来。

# 深度学习入门

## 深度学习入门：基于Python的理论与实现 
《深度学习入门：基于Python的理论与实现》这本书是深度学习入门首选，这本书里对深度学习的基础概念介绍的还算比较清晰，神经网络，反向传播，梯度等等。书中通过实现一个手写数字识别的深度学习网络，将深度学习的基础知识点串起来了。
![深度学习入门：基于Python的理论与实现](/images/deep-learning-book-i-read/deep_learning_intro_based_on_py.jpg)

## 深度学习的数学
深度学习里会有概率，微积分，矩阵这些数学知识，这本书会讲解深度学习中使用这些数学知识的原理，特别是对梯度的数学原理介绍，如果看完《深度学习入门：基于Python的理论与实现》后对深度学习的数学原理不清楚，可以看看这本书。其实这两本书可以交替着看。

![深度学习的数学](/images/deep-learning-book-i-read/math_for_deep_learning.jpg)

## 深入浅出神经网络与深度学习
《深入浅出神经网络与深度学习》这本书的内容与《深度学习入门：基于Python的理论与实现》有一些重复，入门的话还是《深度学习入门：基于Python的理论与实现》。两本书介绍的风格有点不一样，可以交替着看。
![深度学习的数学](/images/deep-learning-book-i-read/nn_and_deep_learning.jpg)

# 深度学习进阶

## Understanding Deep Learning
【《Understanding Deep Learning》】(https://udlbook.github.io/udlbook/)这本书其实也是一本入门的书，但是介绍的内容比前面那基本入门的书更有深度一些，涉及到了深度学习的方方面面。作者开放了 PDF 版本，可以在官网上下载。

## 神经网络与深度学习
[《神经网络与深度学习》](https://nndl.github.io/) 这本书是复旦大学[邱锡鹏](https://nndl.github.io/)老师于 2021 年出版的，PDF 版本也是官方版本。不适合深度学习入门，但是入门知道大概原理以后，想学习深度学习的数学原理推导，可以看看。

## Dive into Deep Learning
[《Dive into Deep Learning》](https://d2l.ai/)这本书是一本开源书，作者之一是李沐大神。书些的比较详细，但是书的写作方式感觉读起来有点难懂，英文版和中文本都是这种感觉。我在读的时候，遇到其他书上的一些概念不懂的时候会去翻这本书。

# 自然语言处理
## 深度学习进阶：自然语言处理
《深度学习进阶：自然语言处理》这本书是《深度学习入门：基于Python的理论与实现》作者深度学习系列的第二本书，主要介绍了深度学习在自然语言处理方面的应用。目前在 Transformer 架构统治下的深度学习背景下，这本书的介绍可能看着有点过时了，但是也是可以看看这里介绍的神经网络模型，比如 RNN，seq2seq，Attention 等概念。毕竟 Transformer 架构也是站在了前人的肩膀上的创新。
![深度学习的数学](/images/deep-learning-book-i-read/deep_learning_nlp.jpg)

## Natural Language Processing
[《Natural Language Processing:Neural Networks and Large Language Models》](https://niutrans.github.io/NLPBook/) 这本书是东北大学的朱靖波和肖桐老师写的关于自然语言和深度学习的书，全书依据深度学习方法在自然语言处理中的应用脉络，由浅入深地展开介绍。其实 Transformer 架构也是被创造出来处理自然语言的，所以这本书对于想了解 Transformer 的话，也是值得一看。

# 大模型
## Build a Large Language Model From Scratch 
[《Build a Large Language Model From Scratch》](https://github.com/rasbt/LLMs-from-scratch) 这本书是关于如何动手写大模型的实践类书籍，由浅入深的介绍了怎么写 Tokenizer，Attention 机制，Transformer 块以及训练模型。想要读懂这本书需要掌握深度学习的知识。这本书还配到了每章的代码实现讲解，值得推荐。
