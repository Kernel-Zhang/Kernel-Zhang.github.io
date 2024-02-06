---
title: gstreamer插件编写指南：服务质量（QoS）
author: kernel
date: 2023-11-16 11:00:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

GStreamer 中的服务质量是指测量和调整流水线的实时性能。实时性能总是相对于流水线时钟来测量的，通常发生在汇元素根据时钟同步缓冲区时。

当缓冲区到达汇的时间较晚，即缓冲区的运行时间小于时钟的运行时间时，我们就会说流水线出现了服务质量问题。以下是几个可能的原因：

-   CPU 负载高，CPU 没有足够的能力处理数据流，导致缓冲区延迟到达汇。
    
-   网络问题
    
-   其他资源问题，如磁盘负载、内存瓶颈等
    

测量结果会产生 QOS 事件，旨在调整一个或多个上游元素的数据传输速率。可进行两种类型的调整：

-   根据汇的最新观测结果进行短时间“紧急”修正。
    
-   根据在汇中观察到的趋势进行长期速率修正。
    

应用程序也可以人为地在同步缓冲区之间引入延迟，这被称为节流。例如，它可用于限制或降低帧速率。

## 衡量服务质量

在流水线的时钟上同步缓冲区的元素通常会测量当前的 QoS。它们还需要保留一些统计数据，以便生成 QOS 事件。

对于到达汇的每个缓冲区，元素需要计算它的延迟或提前程度。这就是抖动。抖动值为负数表示缓冲时间较早，为正数表示缓冲时间较晚。

同步元素还需要计算接收两个连续缓冲区之间的时间间隔。我们称之为处理时间，因为这是上游元素生成/处理缓冲区所需的时间。我们可以将这一处理时间与缓冲区的持续时间进行比较，以衡量上游生产数据的速度，这就是所谓的比例。例如，如果上游能够在 0.5 秒内生成 1 秒长的缓冲区，那么它的运行速度就是所需速度的两倍。反之，如果上游需要 2 秒钟才能生成 1 秒钟的缓冲区数据，则说明上游生成缓冲区的速度太慢，我们将无法保持同步。通常，我们会对这一比例进行平均。

同步元件还需要测量自身的性能，以确定性能问题是否出在自己的上游。

这些测量结果将用于构建 QOS 事件，并向上游发送。请注意，每个到达汇元素的缓冲区都会发送一个 QoS 事件。

## 处理服务质量

元素必须在其源衬底上安装一个事件函数，才能接收 QOS 事件。通常，元素需要存储 QOS 事件的值，并将其用于数据处理功能。元件需要使用锁来保护这些 QoS 值，如下图所示。还要确保将 QoS 事件传递到上游。

```c
    [...]

    case GST_EVENT_QOS:
    {
      GstQOSType type;
      gdouble proportion;
      GstClockTimeDiff diff;
      GstClockTime timestamp;

      gst_event_parse_qos (event, &type, &proportion, &diff, &timestamp);

      GST_OBJECT_LOCK (decoder);
      priv->qos_proportion = proportion;
      priv->qos_timestamp = timestamp;
      priv->qos_diff = diff;
      GST_OBJECT_UNLOCK (decoder);

      res = gst_pad_push_event (decoder->sinkpad, event);
      break;
    }

    [...]
```

通过 QoS 值，元素可以进行两种类型的修正：

### 短期修正

QOS 事件中的时间戳和抖动值可用于进行短期校正。如果抖动值为正，则说明前一个缓冲区到达较晚，我们可以肯定，时间戳 < 时间戳 + 抖动值的缓冲区也会较晚到达。因此，我们可以放弃所有时间戳小于时间戳 + 抖动的缓冲区。

如果缓冲区持续时间已知，那么下一个可能及时到达的时间戳的更好估算方法是：时间戳 + 2 \* 抖动 + 持续时间。

一种可能的算法通常是这样的：

```c
  [...]

  GST_OBJECT_LOCK (dec);
  qos_proportion = priv->qos_proportion;
  qos_timestamp = priv->qos_timestamp;
  qos_diff = priv->qos_diff;
  GST_OBJECT_UNLOCK (dec);

  /* calculate the earliest valid timestamp */
  if (G_LIKELY (GST_CLOCK_TIME_IS_VALID (qos_timestamp))) {
    if (G_UNLIKELY (qos_diff > 0)) {
      earliest_time = qos_timestamp + 2 * qos_diff + frame_duration;
    } else {
      earliest_time = qos_timestamp + qos_diff;
    }
  } else {
    earliest_time = GST_CLOCK_TIME_NONE;
  }

  /* compare earliest_time to running-time of next buffer */
  if (earliest_time > timestamp)
    goto drop_buffer;

  [...]

```

### 长期校正

长期校正的执行要困难一些。它们依赖于 QoS 事件中的比例值。元素应根据 QoS 消息中的比例字段减少其消耗的资源量。

以下是实现这一目标的一些可行策略：

-   永久性地丢弃帧或降低对 CPU 或带宽的要求。有些解码器可以跳过 B 帧的解码。
    
-   改用低质量处理或降低算法复杂度。需要注意的是，这样做不会导致视觉或听觉上的干扰。
    
-   切换到质量较低的信号源，以减少网络带宽。
    
-   为流水线的关键部分分配更多的 CPU 周期。例如，这可以通过提高线程优先级来实现。
    

在任何情况下，当 QoS 事件中的成员比例再次接近 1.0 的理想比例时，元素都应准备好恢复正常处理速度。

## 节流

与时钟同步的元素应公开一个属性，以便在节流模式下对其进行配置。在节流模式下，缓冲区之间的时间距离保持在可配置的节流间隔内。这意味着缓冲速率实际上被限制为每个节流间隔 1 个缓冲。例如，这可用于限制帧率。

当元素配置为节流模式时（通常只在汇上执行），它应在上游产生 QoS 事件，并将抖动字段设置为节流间隔。这将指示上游元素跳过或丢弃配置节流间隔内的剩余缓冲区。

比例字段被设置为获得所需节流间隔所需的减速。实施可以使用 QoS 节流类型、比例和抖动成员来调整其实施。

默认的汇基类具有用于此功能的`throttle-time`属性。您可以使用以下方法进行测试：`gst-launch-1.0 videotestsrc ！xvimagesink throttle-time=500000000`

## 服务质量信息

除了在流水线元素之间发送的 QoS 事件外，流水线总线上还会发布 QoS 消息，向应用程序通报 QoS 结果。QoS 消息包含丢弃事件发生的时间戳，以及丢弃和已处理项目的数量。元素必须在这些条件下发布 QoS 消息：

-   由于 QoS 原因，元素丢弃了缓冲区。
    
-   因 QoS 原因（质量）而改变处理策略的元素。这可能包括解码器决定丢弃每个 B 帧以提高处理速度，或效果元件改用低质量算法。
