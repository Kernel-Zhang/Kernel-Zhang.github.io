---
title: gstreamer插件编写指南：媒体类型和属性
author: kernel
date: 2023-11-15 15:29:00 +0800
categories: [gstreamer, 插件编写]
tags: [gst插件编写, gstreamer, 多媒体]
---

有大量的媒体类型可用于在元素之间传递数据。事实上，每个新定义的元素都可以使用一种新的数据格式（不过，除非至少有一个其他元素能识别这种格式，否则这种格式将毫无用处，因为没有任何元素能与之链接）。

为了让媒体类型发挥作用，并让自动插入器等系统正常运行，所有元素必须就媒体类型定义以及每种媒体类型所需的属性达成一致。GStreamer 框架本身只是提供了定义媒体类型和参数的能力，但并没有固定媒体类型和参数的含义，也没有强制执行创建新媒体类型的标准。这需要由策略来决定，而不是由技术系统来执行。

目前，策略很简单：

-   如果可以使用已有的媒体类型，请勿创建新的媒体类型。
    
-   如果要创建新的媒体类型，请先与其他 GStreamer 开发人员讨论，至少在以下一种方式中进行：IRC、邮件列表。
    
-   尽量确保新格式的名称不会与已创建的其他格式相冲突，也不会是一个过于笼统的名称。例如如果用“audio/compressed”来表示用 mp3 编解码器压缩的音频数据，这个名称就太笼统了。相反，"audio/mp3 "可能是一个合适的名称，或者 "audio/compressed "也可以存在，并有一个属性指示所使用的压缩类型。
    
-   确保在创建新媒体类型时，明确说明该类型，并将其添加到已知媒体类型列表中，以便其他开发人员在编写元素时能正确使用该媒体类型。
    

## 建立简单的测试格式

如果您需要的新格式尚未在我们的“已定义类型列表”中定义，那么您需要了解一些关于媒体类型命名、属性等方面的一般指导原则。媒体类型最好等同于 IANA 定义的 Mime-type；否则，应采用 type/x-name 的形式，其中 type 是该媒体类型处理的数据类型（音频、视频......），name 则是该特定类型的特定名称。音频和视频媒体类型应尽量支持一般的音频/视频属性（见列表），也可以使用自己的属性。要了解我们认为哪些属性是有用的，请再次参阅列表。

慢慢寻找适合自己类型的属性。不必急于求成。此外，进行试验通常也是一个好主意。经验告诉我们，理论上考虑周全的类型是好的，但它们仍然需要实际使用，以确保满足需要。 确保您的属性名称不会与其他类型中使用的类似属性相冲突。如果它们匹配，请确保它们的含义相同；**不允许**使用类型不同但名称相同的属性。

## 类型查找功能和自动插拔

仅仅定义了类型，我们还没有做到这一点。为了识别随机数据文件并将其作为随机数据文件播放，我们需要一种方法来快速识别其类型。为此，我们引入了“类型查找”。类型查找是检测数据流类型的过程。类型查找由两个独立的部分组成：首先，类型查找函数的数量是不限的，每个函数都能从输入流中识别一种或多种类型。其次，还有一个小型引擎，负责注册和调用每个函数。这就是类型查找核心。在这个类型查找核心之上，你通常会编写一个自动外挂程序，它能够使用这个类型检测系统，围绕输入流动态地构建一个流水线。在这里，我们将只关注类型查找函数。

类型查找函数通常位于`gst-plugins-base/gst/typefind/gsttypefindfunctions.c` 中，除非有充分的理由（如库依赖关系）将其放在其他地方。之所以要集中处理，是为了减少为检测数据流类型而需要加载的插件数量。下面是一个可以识别 AVI 文件的示例，文件以 "RIFF "标签开头，然后是文件大小，最后是 "AVI "标签：

```c
static GstStaticCaps avi_caps = GST_STATIC_CAPS ("video/x-msvideo");
#define AVI_CAPS gst_static_caps_get(&avi_caps)
static void
gst_avi_typefind_function (GstTypeFind *tf,
              gpointer     pointer)
{
  guint8 *data = gst_type_find_peek (tf, 0, 12);

  if (data &&
      GUINT32_FROM_LE (&((guint32 *) data)[0]) == GST_MAKE_FOURCC ('R','I','F','F') &&
      GUINT32_FROM_LE (&((guint32 *) data)[2]) == GST_MAKE_FOURCC ('A','V','I',' ')) {
    gst_type_find_suggest (tf, GST_TYPE_FIND_MAXIMUM, AVI_CAPS);
  }
}

GST_TYPE_FIND_REGISTER_DEFINE(avi, "video/x-msvideo", GST_RANK_PRIMARY,
    gst_avi_typefind_function, "avi", AVI_CAPS, NULL, NULL);

static gboolean
plugin_init (GstPlugin *plugin)
{
  return GST_TYPEFIND_REGISTER(avi, plugin);
}
```

请注意，`gst-plugins/gst/typefind/gsttypefindfunctions.c`包含一些简化宏，以减少代码量。如果您想提交带有新类型查找函数的类型查找补丁，请充分利用这些宏。

《应用程序开发手册》对自动插拔进行了详细讨论。

## 已定义类型列表

以下是 GStreamer 中所有已定义类型的列表。为了便于阅读，它们被分成音频、视频、容器、字幕和其他类型的单独表格。每个表格下面都有适用于该表格的注释列表。在定义每种类型时，我们尽可能遵循[IANA](https://www.iana.org/assignments/media-types)定义的类型和规则。

请注意，许多属性都不是必备属性，而是可选属性。这意味着这些属性大多可以从容器标头中提取，但如果容器标头没有提供这些属性，也可以通过解析流标头或流内容来提取。原则是，你的元素应该只通过解析自己的内容而不是其他元素的内容来提供它所知道的数据。例如：AVI 标头在标头中提供了所含音频流的采样率。而 MPEG 系统流则不会。这意味着 AVI 码流解码器会提供 MPEG 音频码流的采样率属性，而 MPEG 解码器则不会。需要这些数据的解码器需要一个流解析器来从标头中提取或从流中计算这些数据。

### 音频类型表

音频类型表  

| 媒体类型 | 说明 |
| --- | --- |
| 所有音频类型 |
| audio/\* | 所有音频类型 |
| channels | 整数 |
| channel-mask | 位掩码 |
| format | 字符串 |
| layout | 字符串 |
| 所有原始音频类型 |
| audio/x-raw | 非结构化和未压缩的原始音频数据 |
| 所有编码音频类型 |
| audio/x-ac3 | AC-3 或 A52 音频流 |
| audio/x-adpcm | ADPCM 音频流 |
| block\_align | 整数 |
| audio/x-cinepak | 在 Cinepak（Quicktime）流中提供的音频 |
| audio/x-dv | 数字视频流中提供的音频 |
| audio/x-flac | 自由无损音频编解码器 (FLAC) |
| audio/x-gsm | 由 GSM 编解码器编码的数据 |
| audio/x-law | A-Law 音频 |
| audio/x-mulaw | Mu-Law 音频 |
| audio/x-mace | MACE 音频（在 Quicktime 中使用） |
| audio/mpeg | 使用 MPEG 音频编码方案压缩的音频数据 |
| framed | 布尔 |
| layer | 整数 |
| bitrate | 整数 |
| audio/x-qdm2 | 由 QDM 第 2 版编解码器编码的数据 |
| audio/x-pn-realaudio | Realmedia 音频数据 |
| audio/x-speex | 由 Speex 音频编解码器编码的数据 |
| audio/x-vorbis | Vorbis 音频数据 |
| audio/x-wma | Windows 媒体音频 |
| audio/x-paris | Ensoniq PARIS 音频 |
| audio/x-svx | Amiga IFF / SVX8 / SV16 音频 |
| audio/x-nist | Sphere NIST 音频 |
| audio/x-voc | Sound Blaster VOC 音频 |
| audio/x-ircam | Berkeley/IRCAM/CARL 音频 |
| audio/x-w64 | Sonic Foundry 的 64 位 RIFF/WAV |

### 视频类型表

视频类型表  

| 媒体类型 | 说明 |
| --- | --- |
| 所有视频类型 |
| video/\* | 所有视频类型 |
| height | 整数 |
| framerate | 分数 |
| max-framerate | 分数 |
| views | 整数 |
| interlace-mode | 字符串 |
| chroma-site | 字符串 |
| colorimetry | 字符串 |
| pixel-aspect-ratio | 分数 |
| format | 字符串 |
| 所有原始视频类型 |
| video/x-raw | 非结构化和未压缩的原始视频数据 |
| 所有编码视频类型 |
| video/x-3ivx | 3ivx 视频 |
| video/x-divx | DivX 视频 |
| video/x-dv | 数字视频 |
| video/x-ffv | FFMpeg 视频 |
| video/x-h263 | H-263 视频 |
| h263version | 字符串 |
| video/x-h264 | H-264 视频 |
| video/x-huffyuv | Huffyuv 视频 |
| video/x-indeo | Indeo 视频 |
| video/x-intel-h263 | H-263 视频 |
| video/x-jpeg | Motion-JPEG 视频 |
| video/mpeg | MPEG 视频 |
| systemstream | 布尔 |
| video/x-msmpeg | 微软 MPEG-4 视频标准 |
| video/x-msvideocodec | 微软视频 1（老式编解码器） |
| video/x-pn-realvideo | Realmedia 视频。 |
| video/x-rle | RLE 动画格式。 |
| depth | 整数 |
| palette_data | GstBuffer |
| video/x-svq | Sorensen视频 |
| video/x-tarkin | Tarkin视频 |
| video/x-theora | Theora 视频 |
| video/x-vp3 | VP-3 视频。 |
| video/x-wmv | Windows 媒体视频 |
| video/x-xvid | XviD 视频 |
| 所有图像类型 |
| image/gif | 图形交换格式 |
| image/jpeg | 联合图片专家组图片 |
| image/png | 便携式网络图形图像 |
| image/tiff | 标签图像文件格式 |

### 容器类型表

容器类型表  

| 媒体类型 | 说明 |
| --- | --- |
| video/x-ms-asf | 高级流媒体格式（ASF） |
| video/x-msvideo | AVI |
| video/x-dv | 数字视频 |
| video/x-matroska | Matroska |
| video/mpeg | 电影专家组系统流 |
| application/ogg | Ogg |
| video/quicktime | Quicktime |
| application/vnd.rn-realmedia | RealMedia |
| audio/x-wav | WAV |

### 字幕类型表

字幕类型表  

| 媒体类型 | 说明 |
| --- | --- |
|  |  |

### 其他类型表

其他类型表  

| 媒体类型 | 说明 |
| --- | --- |
|  |  |

