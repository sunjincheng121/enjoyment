---
title: Apache Flink 漫谈系列 - 概述
date: 2019-07-06 06:57:56
categories: Apache Flink
---
# Apache Flink 的命脉
"**命脉**" 即生命与血脉，常喻极为重要的事物。系列的首篇，首篇的首段不聊Apache Flink的历史，不聊Apache Flink的架构，不聊Apache Flink的功能特性，我们用一句话聊聊什么是 Apache Flink 的命脉？我的答案是：Apache Flink 是以"**批是流的特例**"的认知进行系统设计的。

# 唯快不破
我们经常听说 "天下武功，唯快不破"，大概意思是说 "任何一种武功的招数都是有拆招的，唯有速度快，快到对手根本来不及反应，你就将对手KO了，对手没有机会拆招，所以唯快不破"。 那么这与Apache Flink有什么关系呢？Apache Flink是Native Streaming(纯流式)计算引擎，在实时计算场景最关心的就是"**快**",也就是 "**低延时**"。

就目前最热的两种流计算引擎Apache Spark和Apache Flink而言，谁最终会成为No1呢？单从 "**低延时**" 的角度看，Spark是Micro Batching(微批式)模式，最低延迟Spark能达到0.5~2秒左右，Flink是Native Streaming(纯流式)模式，最低延时能达到微秒。很显然是相对较晚出道的 Apache Flink 后来者居上。 那么为什么Apache Flink能做到如此之 "**快**"呢？根本原因是Apache Flink 设计之初就认为 "**批是流的特例**"，整个系统是Native Streaming设计，每来一条数据都能够触发计算。相对于需要靠时间来积攒数据Micro Batching模式来说，在架构上就已经占据了绝对优势。

那么为什么关于流计算会有两种计算模式呢？归其根本是因为对流计算的认知不同，是"**流是批的特例**" 和 "**批是流的特例**" 两种不同认知产物。

TODO....

