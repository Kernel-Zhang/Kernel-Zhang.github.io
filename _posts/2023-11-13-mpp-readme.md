---
title: rockchip MPP模块使用说明
author: kernel
date: 2023-11-13 12:34:00 +0800
categories: [视频处理, rockchip]
tags: [MPP, rockchip, 视频处理]
---

## 整体说明

媒体处理平台 (MPP) 模块目录说明：

MPP：媒体处理平台（Media Process Platform）
MPI：媒体处理接口（Media Process Interface）
HAL：硬件抽象层（Hardware Abstract Layer）
OSAL：操作系统抽象层（Operation System Abstract Layer）

规则：
1. 头文件排列规则

    a. 每个模块文件夹下的inc目录供外部模块使用。

    b. 模块内部头文件应与实现文件放在一起。

    c. 头文件不应包含任何相对路径或绝对路径，所有包含路径都应保留在`Makefile`中。

2. 编译系统规则

    a. 跨平台编译时使用`cmake`作为编译管理系统。

    b. 使用`cmake`在代码的外部进行编译，最终生成的二进制文件和库将安装到`out/`目录。

3. 头文件包含顺序

    a. MODULE_TAG

    b. 系统头文件

    c. osal头文件

    d. 模块头

注意：

1. 不再支持Windows操作系统。

2. Mpp现在支持所有rockchip芯片组，包括：

   RK29XX/RK30XX/RK31XX

   RK3288/RK3368/RK3399

   RK3228/RK3229/RK3228H/RK3328

   RK3528/RK3528A

   RK3562

   RK3566/RK3568

   RK3588

   RV1108/RV1107

   RV1109/RV1126

3. 除VC1外，Mpp支持所有硬件可支持的格式。

4. 您可以获得有关MPP应用于Linux和Android的演示。

    Liunx : 

    <https://github.com/WainDing/mpp_linux_cpp>

    <https://github.com/MUZLATAN/ffmpeg_rtsp_mpp>

    Android : 

    <https://github.com/c-xh/RKMediaCodecDemo>

5. 官方github: <https://github.com/rockchip-linux/mpp>

   开发中的github: <https://github.com/HermanChen/mpp>

   开发中的gitee : <https://gitee.com/hermanchen82/mpp>

6. 提交信息格式应基于：<https://keepachangelog.com/en/1.0.0/>

更多文档，请访问：<https://opensource.rock-chips.com/wiki_Mpp>

## 目录结构

下面是MPP源代码目录结构：

```
----                             顶层
   |
   |----- build                  CMake源码外编译目录
   |  |
   |  |----- cmake               cmake脚本目录
   |  |
   |  |----- android             android构建目录
   |  |
   |  |----- linux               linux构建目录
   |  |
   |  |----- vc10-x86_64         visual studio 2010下x86_64构建目录
   |  |
   |  |----- vc12-x86_64         visual studio 2013下x86_64构建目录
   |
   |----- doc                    MPP设计文档
   |
   |----- inc                    供外部使用的头文件包括
   |                             平台头文件和MPI头文件
   |
   |----- mpp                    媒体处理平台：mpi函数私有化
   |  |                          实现和mpp基础架构（vpu_api
   |  |                          私有层）
   |  |
   |  |----- base                基本组件，包括 MppBuffer、MppFrame、
   |  |                          MppPacket, MppTask, MppMeta 等。
   |  |
   |  |----- common              通用视频编解码器协议语法接口，可同时用于
   |  |                          编解码解析器和硬件抽象层
   |  |
   |  |----- codec               所有视频编解码解析器，将流转换为
   |  |  |                       协议结构
   |  |  |
   |  |  |----- inc              编码解码器模块提供的头文件，用于
   |  |  |                       外部使用
   |  |  |
   |  |  |----- dec
   |  |  |  |
   |  |  |  |----- dummy         解码器解析器工作流程示例
   |  |  |  |
   |  |  |  |----- h263
   |  |  |  |
   |  |  |  |----- h264
   |  |  |  |
   |  |  |  |----- h265
   |  |  |  |
   |  |  |  |----- m2v           mpeg2解析器
   |  |  |  |
   |  |  |  |----- mpg4          mpeg4解析器
   |  |  |  |
   |  |  |  |----- vp8
   |  |  |  |
   |  |  |  |----- vp9
   |  |  |  |
   |  |  |  |----- jpeg
   |  |  |
   |  |  |----- enc
   |  |     |
   |  |     |----- dummy         编码器工作流程示例
   |  |     |
   |  |     |----- h264
   |  |     |
   |  |     |----- h265
   |  |     |
   |  |     |----- jpeg
   |  |
   |  |----- hal                 硬件抽象层（HAL）：mpi中使用的模块
   |  |  |
   |  |  |----- inc              头文件由hal提供，供外部使用
   |  |  |
   |  |  |----- iep              iep用户库
   |  |  |
   |  |  |----- pp               后处理用户库
   |  |  |
   |  |  |----- rga              rga用户库
   |  |  |
   |  |  |----- deinter          反交错函数模块，包括pp、iep、rga
   |  |  |
   |  |  |----- rkdec            生成rockchip硬件解码器寄存器
   |  |  |  |
   |  |  |  |----- h264d         根据H.264 语法信息生成寄存器文件
   |  |  |  |
   |  |  |  |----- h265d         根据 H.265 语法信息生成寄存器文件
   |  |  |  |
   |  |  |  |----- vp9d          根据 vp9 语法信息生成寄存器文件
   |  |  |
   |  |  |----- vpu              vpu寄存器生成库
   |  |     |
   |  |     |----- h263d         根据h263d语法信息生成寄存器文件
   |  |     |
   |  |     |----- h264d         根据h264d语法信息生成寄存器文件
   |  |     |
   |  |     |----- h265d         根据h265d语法信息生成寄存器文件
   |  |     |
   |  |     |----- jpegd         根据jpegd语法信息生成寄存器文件
   |  |     |
   |  |     |----- jpege         根据jpege语法信息生成寄存器文件
   |  |     |
   |  |     |----- m2vd          根据m2vd语法信息生成寄存器文件
   |  |     |
   |  |     |----- mpg4d         根据mpg4d语法信息生成寄存器文件
   |  |     |
   |  |     |----- vp8d          根据vp8d语法信息生成寄存器文件
   |  |
   |  |----- legacy              生成新的libvpu，包含了旧的vpuapi补丁
   |  |                          和新的MPP补丁
   |  |
   |  |----- test                MPP内部视频协议单元测试和示例
   |
   |----- test                   MPP的buffer和packet组件单元测试以及
   |                             mpp、mpi、vpu_api的示例文件
   |
   |----- out                    最终发布文件的输出目录
   |  |
   |  |----- bin                 可执行文件的输出目录
   |  |
   |  |----- inc                 头文件的输出目录
   |  |
   |  |----- lib                 库文件输出目录
   |
   |----- osal                   操作系统抽象层：
   |  |                          对不同的操作系统进行统一抽象
   |  |
   |  |----- allocator           支持的内存分配器：Android的ion和
   |  |                          Linux的drm
   |  |
   |  |----- android             谷歌android
   |  |
   |  |----- inc                 用于MPP模块的osal头文件
   |  |
   |  |----- linux               linux主线内核
   |  |
   |  |----- windows             微软windows
   |  |
   |  |----- test                OSAL单元测试
   |
   |----- tools                  代码风格格式化工具
   |
   |----- utils                  小型函数
```

## 整体框架

下面是MPP实现的整体框架：

```
                +---------------------------------------+
                |                                       |
                | ffmpeg / OpenMax / gstreamer / libva  |
                |                                       |
                +---------------------------------------+

            +-------------------- MPP ----------------------+
            |                                               |
            |   +-------------------------+    +--------+   |
            |   |                         |    |        |   |
            |   |        MPI / MPP        |    |        |   |
            |   |   buffer queue manage   |    |        |   |
            |   |                         |    |        |   |
            |   +-------------------------+    |        |   |
            |                                  |        |   |
            |   +-------------------------+    |        |   |
            |   |                         |    |        |   |
            |   |          codec          |    |  OSAL  |   |
            |   |    decoder / encoder    |    |        |   |
            |   |                         |    |        |   |
            |   +-------------------------+    |        |   |
            |                                  |        |   |
            |   +-----------+ +-----------+    |        |   |
            |   |           | |           |    |        |   |
            |   |  parser   | |    HAL    |    |        |   |
            |   |  recoder  | |  reg_gen  |    |        |   |
            |   |           | |           |    |        |   |
            |   +-----------+ +-----------+    +--------|   |
            |                                               |
            +-------------------- MPP ----------------------+

                +---------------------------------------+
                |                                       |
                |                kernel                 |
                |       RK vcodec_service / v4l2        |
                |                                       |
                +---------------------------------------+
```

## MPI分层结构

这里是媒体处理接口的分层结构。MpiPacket和MpiFrame是流式I/O数据结构，而MpiBuffer则封装了不同的缓冲区实现，如Linux的dma-buf和Android的ion。

下面的框图是来自于ffmpeg：

```
                +-------------------+
                |                   |
                |        MPI        |
                |                   |
                +---------+---------+
                          |
                          |
                          v
                +---------+---------+
                |                   |
            +---+        ctx        +---+
            |   |                   |   |
            |   +-------------------+   |
            |                           |
            v                           v
    +-------+-------+           +-------+-------+
    |               |           |               |
    |     packet    |           |     frame     |
    |               |           |               |
    +---------------+           +-------+-------+
            |                           |
            |                           |
            |                           |
            |     +---------------+     |
            |     |               |     |
            +---->+     buffer    +<----+
                  |               |
                  +---------------+
```

## 各层交互流程

以H.264解码器为例。视频流首先进入MPI/MPP层队列，然后MPP将视频流发送到编解码器层，编解码器层解析视频流头并生成符合协议的标准输出。该输出将被发送到HAL层，以生成寄存器文件集并与硬件通信。硬件将完成任务并回传信息。MPP根据硬件返回的结果通知编解码器，编解码器按显示顺序输出解码帧。

```
MPI                MPP              decoder             parser              HAL

 +                  +                  +                  +                  +
 |                  |                  |                  |                  |
 |   open context   |                  |                  |                  |
 +----------------> |                  |                  |                  |
 |                  |                  |                  |                  |
 |       init       |                  |                  |                  |
 +----------------> |                  |                  |                  |
 |                  |                  |                  |                  |
 |                  |       init       |                  |                  |
 |                  +----------------> |                  |                  |
 |                  |                  |                  |                  |
 |                  |                  |       init       |                  |
 |                  |                  +----------------> |                  |
 |                  |                  |                  |                  |
 |                  |                  |                  |       open       |
 |                  |                  +-----------------------------------> |
 |                  |                  |                  |                  |
 |      decode      |                  |                  |                  |
 +----------------> |                  |                  |                  |
 |                  |                  |                  |                  |
 |                  |   send_stream    |                  |                  |
 |                  +----------------> |                  |                  |
 |                  |                  |                  |                  |
 |                  |                  |   parse_stream   |                  |
 |                  |                  +----------------> |                  |
 |                  |                  |                  |                  |
 |                  |                  |                  |  reg generation  |
 |                  |                  +-----------------------------------> |
 |                  |                  |                  |                  |
 |                  |                  |                  |    send_regs     |
 |                  |                  +-----------------------------------> |
 |                  |                  |                  |                  |
 |                  |                  |                  |    wait_regs     |
 |                  |                  +-----------------------------------> |
 |                  |                  |                  |                  |
 |                  |                  |  notify_hw_end   |                  |
 |                  |                  +----------------> |                  |
 |                  |                  |                  |                  |
 |                  |   get_picture    |                  |                  |
 |                  +----------------> |                  |                  |
 |                  |                  |                  |                  |
 |                  |                  |   get_picture    |                  |
 |                  |                  +----------------> |                  |
 |                  |                  |                  |                  |
 |      flush       |                  |                  |                  |
 +----------------> |                  |                  |                  |
 |                  |                  |                  |                  |
 |                  |      flush       |                  |                  |
 |                  +----------------> |                  |                  |
 |                  |                  |                  |                  |
 |                  |                  |      reset       |                  |
 |                  |                  +----------------> |                  |
 |                  |                  |                  |                  |
 |      close       |                  |                  |                  |
 +----------------> |                  |                  |                  |
 |                  |                  |                  |                  |
 |                  |      close       |                  |                  |
 |                  +----------------> |                  |                  |
 |                  |                  |                  |                  |
 |                  |                  |      close       |                  |
 |                  |                  +----------------> |                  |
 |                  |                  |                  |                  |
 |                  |                  |                  |      close       |
 |                  |                  +-----------------------------------> |
 +                  +                  +                  +                  +
```

## 内存模式

解码器可支持三种内存使用模式：

模式1：纯内部模式
在该模式下，用户不会调用`MPP_DEC_SET_EXT_BUF_GROUP`控制解码器，只会调用`MPP_DEC_SET_INFO_CHANGE_READY`让解码器继续工作。然后解码器将在内部使用创建缓冲区，用户需要释放他们获得的每个帧。

优点
易于使用，可快速开发出应用程序
缺点：
1. 在解码器关闭之前，解码器可能不会返回缓冲区。因此可能会发生内存泄漏或崩溃。
2. 无法控制解码器的内存使用。解码器处于自由运行状态，会消耗掉它能获得的所有内存。
3. 难以实现零拷贝显示路径。

模式 2：半内部模式
这是当前`mpi_dec_test`代码使用的模式。用户需要根据`MppFrame`返回的更改信息创建`MppBufferGroup`，用户可以使用`mpp_buffer_group_limit_config`来限制解码器内存的使用。

优点
1. 易于使用
2. 用户可以在解码器关闭后释放`MppBufferGroup`。因此内存可以安全地保留更长时间。
3. 可通过`mpp_buffer_group_limit_config`限制内存使用量
缺点
1. 缓冲区限制仍然不准确。内存使用是100%固定的。
2. 也很难实现零拷贝显示路径。

模式 3：纯外部模式
在此模式下，需要创建空的`MppBufferGroup`并通过文件句柄从外部分配器导入内存。在安卓系统中，`surfaceflinger`会创建缓冲区。然后，`mediaserver`从`surfaceflinger`获取文件句柄，并提交到解码器的 `MppBufferGroup`。

优势
1. 最有效的零拷贝显示方式
缺点
1. 难以学习和使用。
2. 播放器工作流程可能会限制其使用。
3. 可能需要外部解析器为外部分配器获取正确的缓冲区大小。

## 缓冲区大小计算

所需缓冲区大小的计算：
对于像素数据：hor_stride * ver_stride * 3 / 2
对于额外信息：hor_stride * ver_stride / 2
总计：hor_stride * ver_stride * 2。

对于 H.264/H.265，20个以上的缓冲区就足够了。
对于其他编解码器，10个缓冲区就足够了。