---
title: rockchip多媒体处理概览
author: kernel
date: 2023-11-10 16:34:00 +0800
categories: [视频处理, rockchip]
tags: [MPP, rockchip, 视频处理]
---

MPP（Media Process Platform：媒体处理平台）是Rockchip平台的视频编解码器和硬件抽象层的应用库统称。



## 获取源代码

您可以从git获取mpp源代码。

```shell
git clone -b release https://github.com/rockchip-linux/mpp.git
```

## 构建

### Unix/Linux

如果您使用的是Debian相关的发行版，源代码中有debian构建规则。您可以在debian目录中查看进一步的信息。

您可以使用以下命令构建deb软件包，支持的架构有：armhf、arm64。

```shell
DEB_BUILD_OPTIONS="parallel=4 nocheck" dpkg-buildpackage -a<arch>
```

请先安装所有需要的交叉编译工具和Debian软件包工具。

您可以使用以下命令在开发板中直接构建它：

```shell
cmake -DRKPLATFORM=ON -DHAVE_DRM=ON && make
```

### 安卓

在Android中构建需要使用android ndk软件包，通常我们使用android-ndk-r10d。  
您可以从谷歌网站下载ndk，并在`make-Android.bash`中修改ndk路径。自动路径为  
`/home/pub/ndk/android-ndk-r10d`。  
然后，执行`build/andorid/xx/make-Android.bash`。第一次执行时可能会出现一些错误  
重新执行`make-Android.bash`，问题即可解决。

### Windows

执行`build/xx-x86_64/build-all.bat`和`make-solutions.bat`

## 系统框图

```
                +---------------------------------------+
                |                                       |
                | OpenMax  / libva                      |
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
            |   |  control  | |  reg_gen  |    |        |   |
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

## 关于Mpp

mpp是Rockchip SoC跨平台媒体处理的中间件库。  
mpp的主要目的是为多媒体（主要是视频和图像）处理提供高性能、高灵活性和可扩展性。  

mpp 的设计目标是连接不同的Rockchip硬件内核驱动程序和不同的用户空间应用程序。  

Rockchip有两套硬件内核驱动程序。

第一个是`vcodec_service/vpu_service/mpp_service`，它基于无状态框架，是一个高性能的基础内核驱动程序。该驱动程序支持硬件可提供的所有编解码器。该驱动程序同时用于Android和Linux。

第二个是为ChromeOS开发的v4l2驱动程序。它目前支持H.264/H.265/vp8/vp9。该驱动可用于ChomeOS和Linux。

Mpp计划支持多种用户空间应用程序，包括OpenMax、libva。

## Mpp功能

1. 跨平台
目标操作系统包括Android、Linux、ChromeOS和windows。Mpp使用cmake在不同平台上编译。

2. 高性能
Mpp支持同步/异步接口，以减少接口阻塞时间。而且mpp在内部使硬件和软件并行运行。当硬件运行时软件将同时准备下一个硬件任务。

3. 高度灵活性
mpi（媒体处理接口）易于扩展不同的控制功能。输入/输出元素和包/帧/缓冲区易于扩展到不同的组件。

## Mpp组件

### OSAL（操作系统抽象层）

本模块介绍了不同操作系统之间的差异，并提供了包括内存、时间、线程、日志和物理内存分配器在内的基本组件。

### MPI（媒体处理接口）/ MPP

该模块负责与外部用户进行交互。Mpi层有两种种方式。
简单方式——用户可以使用数据包/图像帧的put/get函数接口。  
高级方式——用户必须配置MppTask并使用dequeue/enqueue函数接口与mpp通信。MppTask可以携带不同的元数据并完成复杂的工作。  

### 编解码器（编码器/解码器）

该模块实现了高效的内部工作流程。编解码器模块提供了不同视频格式的一般调用流程。软件层实现了与具体特定硬件的分离。软件层通过一个附带buffer信息和编解码信息的通用任务接口与硬件通信。  

### 解析器/控制器和hal（硬件抽象层）

该层实现了对不同视频格式和不同硬件的功能调用。解码器解析器提供视频流解析函数和输出格式相关的语法结构。硬件抽象层将把语法结构转换为不同硬件上的寄存器集。当前硬件抽象层支持`vcodec_service`内核驱动程序，并计划支持v4l2驱动程序。

## Mpp内核驱动程序

Rockchip有两套硬件内核驱动程序。

第一个是`vcodec_service/vpu_service/mpp_service`，它基于无状态框架，是一个高性能的基础内核驱动程序。该驱动程序支持硬件可提供的所有编解码器。该驱动程序可以用于Android和Linux。

第二个是为ChromeOS开发的v4l2驱动程序。它目前支持H.264/H.265/vp8/vp9。该驱动可用于ChomeOS和Linux

下面是`vcodec_service`内核驱动程序框架图。

```
    +-------------+             +-------------+             +-------------+
    |  client  A  |             |  client  B  |             |  client  C  |
    +-------------+             +-------------+             +-------------+

userspace
+------------------------------------------------------------------------------+
 kernel

    +-------------+             +-------------+             +-------------+
    |  session A  |             |  session B  |             |  session C  |
    +-------------+             +-------------+             +-------------+
    |             |             |             |             |             |
 waiting         done        waiting         done        waiting         done
    |             |             |             |             |             |
+---+---+     +---+---+     +---+---+     +---+---+                   +---+---+
| task3 |     | task0 |     | task1 |     | task0 |                   | task0 |
+---+---+     +-------+     +-------+     +-------+                   +---+---+
    |                                                                     |
+---+---+                                                             +---+---+
| task2 |                                                             | task1 |
+-------+                                                             +-------+
                                 +-----------+
                                 |  service  |
                       +---------+-----------+---------+
                       |               |               |
                    waiting         running          done
                       |               |               |
                 +-----+-----+   +-----+-----+   +-----+-----+
                 |  task A2  |   |  task A1  |   |  task C0  |
                 +-----+-----+   +-----------+   +-----+-----+
                       |                               |
                 +-----+-----+                   +-----+-----+
                 |  task B1  |                   |  task C1  |
                 +-----+-----+                   +-----+-----+
                       |                               |
                 +-----+-----+                   +-----+-----+
                 |  task A3  |                   |  task A0  |
                 +-----------+                   +-----+-----+
                       |
                 +-----+-----+
                 |  task B0  |
                 +-----------+

```

这种设计的原理是将用户任务处理和硬件资源管理分开，尽量减少两个硬件进程操作之间的内核串行进程时间。

驱动程序使用会话作为通信渠道。每个用户空间客户端（客户机）都有一个内核会话。客户端会向会话提交任务。然后由服务（vpu_service/vcodec_service）管理硬件。服务将提供在会话中处理任务的能力。

当客户端向内核提交任务时，任务将被设置为等待状态，并同时链接到会话等待列表和服务等待列表。然后，服务会将任务从等待列表转入运行列表并运行。当硬件完成一项任务时，该任务将被移至已完成列表，并同时放入服务已完成列表和会话已完成列表。最后，客户端将从会话中获取已完成的任务。

## Mpp缓冲区

Mpp缓冲区是硬件使用的缓冲区。硬件通常无法使用cpu的malloc缓冲区。因此，我们为不同平台上的不同内存分配器设计了MppBuffer。目前，它设计为在Android上是ion buffer，在Linux上是drm buffer。以后可能会支持v4l2设备上的vb2_buffer。

为了管理不同用户的缓冲区使用情况，mpp缓冲模块引入了MppBufferGroup作为缓冲区管理器。MppBufferGroup提供分配器服务和缓冲区重用能力。

不同的MppBufferGroup有不同的分配器。除了普通的malloc/free函数外，分配器还可以接受来自外部文件描述符的缓冲区。

MppBufferGroup有两个列表，未使用的缓冲区列表和已使用的缓冲区列表。当缓冲区空闲时，缓冲区不会立即释放。缓冲区将被移至未使用列表，以便以后重新使用。这样做是有原因的。当视频分辨率达到4K时，缓冲区的大小将超过12M。分配缓冲区和为其生成映射表将需要很长时间。因此，重复使用缓冲区将节省分配/释放时间。

下面是 Mpp 缓冲区状态事务的示意图。

```
            +----------+     +---------+
            |  create  |     |  commit |
            +-----+----+     +----+----+
                  |               |
                  |               |
                  |          +----v----+
                  +----------+ unused  <-----------+
                  |          +---------+           |
                  |          |         |           |
                  |          |         |           |
            +-----v----+     |         |     +-----+----+
            |  malloc  |     |         |     |   free   |
            +----------+     |         |     +----------+
            | inc_ref  |     +---------+     |  dec_ref |
            +-----+----+                     +-----^----+
                  |                                |
                  |                                |
                  |          +---------+           |
                  +---------->  used   +-----------+
                             +---------+
                             |         |
                             |         |
                             |         |
                             |         |
                             |         |
                             +----^----+
                                  |
                                  |
                             +----+----+
                             | import  |
                             +---------+
```

## Mpp任务

Mpp任务是高级模式下与外部用户进行交互的控制组件。高级模式的目标是提供灵活、多输入/多输出扩展。

Mpp任务以mpp_meta作为多个内容的载体。Mpp元使用键和值对进行扩展。一个任务可以将多个数据输入或输出mpp。典型的例子是带有OSD和运动检测功能的编码器。一个任务可能包含作为输入的OSD缓冲区、运动检测缓冲区、帧缓冲区和流缓冲区，以及作为数据输出的流缓冲区和运动检测缓冲区。如果解码器希望输出一些辅助信息，这种情况也可用于解码器。

1. Mpp任务队列
Mpp任务队列是任务的管理器。由于用户可能会错误地使用任务，我们选择了在mpp中承担所有任务的设计。任务队列将承担人物的创建和释放。但任务队列不会直接与用户交互。我们使用端口和任务状态来控制事务。

2. Mpp端口
Mpp端口是任务队列的事务接口。外部用户和内部工作线程将使用`mpp_port_poll / mpp_port_dequeue / mpp_port_enqueue`接口来轮询、出列、入列任务队列。Mpp高级模式使用端口连接外部用户、接口存储和内部进程线程。每个任务队列有两个端口：输入端口和输出端口。从全局来看，任务总是从输入端口流向输出端口。

3. Mpp 任务状态
一个任务有四种状态。Mpp使用`list_head`表示状态。

INPUT_PORT : 输入端口用户要出列前的初始状态。或者当输出端口成功入列一个任务时，该任务处于此状态。

INPUT_HOLD : 当输入端口用户成功出列任务时，任务处于此状态。

OUTPUT_PORT：当输入端口用户成功地将任务入列时，任务处于OUTPUT_PORT状态。或者该任务已准备好从输出端口出列。

OUTPUT_HOLD：当输出端口用户成功出列任务时，任务处于此状态。

4. Mpp 任务/端口事务
当端口用户调用事务功能时，任务将从一个状态转移到下一个状态。从输入端口到输出端口的状态转换流是单向的。

任务/端口/状态的整体关系图如下所示。

```
                               1. task queue
                       +------------------------------+
                       |                              |
                  +----+----+  +--------------+  +----+----+
                  |    4    |  |       3      |  |   2.1   |
         +--------+ dequeue <--+    status    <--+ enqueue <---------+
         |        |         |  |  INPUT_PORT  |  |         |         |
         |        +---------+  |              |  +---------+         |
  +------v-----+  |         |  +--------------+  |         |  +------+------+
  |      3     |  |    2    |                    |    2    |  |      3      |
  |   status   |  |  input  |                    |  output |  |   status    |
  | INPUT_HOLD |  |   port  |                    |   port  |  | OUTPUT_HOLD |
  |            |  |         |                    |         |  |             |
  +------+-----+  |         |  +--------------+  |         |  +------^------+
         |        +---------+  |       3      |  +---------+         |
         |        |    4    |  |    status    |  |   2.1   |         |
         +--------> enqueue +-->  OUTPUT_PORT +--> dequeue +---------+
                  |         |  |              |  |         |
                  +----+----+  +--------------+  -----+----+
                       |                              |
                       +------------------------------+
```

在高级模式下，mpp使用两个任务队列：输入任务队列和输出任务队列。

输入任务队列将输入端的外部用户连接到内部工作线程，输出任务队列将内部工作线程连接到输出端的外部用户。这将最大限度地提高mpp的效率。

工作流程如下图所示。

```
            +-------------------+           +-------------------+
            |  input side user  |           | output side user  |
            +----^---------+----+           +----^---------+----+
                 |         |                     |         |
            +----+----+----v----+           +----+----+----v----+
            | dequeue | enqueue |           | dequeue | enqueue |
            +----^----+----+----+           +----^----+----+----+
                 |         |                     |         |
            +----+---------+----+    MPP    +----+---------v----+
        +---+     input port    +-----------+    output port    +---+
        |   +-------------------+           +-------------------+   |
        |   |  input task queue |           | output task queue |   |
        |   +-------------------+           +-------------------+   |
        |   |    output port    |           |     input port    |   |
        |   +----+---------^----+           +----+---------^----+   |
        |        |         |                     |         |        |
        |   +----v----+----+----+           +----v----+----+----+   |
        |   | dequeue | enqueue |           | dequeue | enqueue |   |
        |   +----+----+----^----+           +----+----+----^----+   |
        |        |         |                     |         |        |
        |   +----v---------+---------------------v---------+----+   |
        |   |              internal work thread                 |   |
        |   +---------------------------------------------------+   |
        |                                                           |
        +-----------------------------------------------------------+
```

## Mpp用户指南

### 多线程

Mpp并不直接提供线程接口，因此您应该使用相应的操作系统接口。要使用多线程编码和解码，必须做一些事情：

1. 资源应申请多份而不是一份，以避免资源竞争。

2. 释放资源的时机不同，应在帧或数据包不再被使用后释放。

有关多线程的具体细节，请参阅`test/mpi_enc_test`和`test/mpi_dec_test`。

### 参数集

Mpp需要一些参数来决定工作模式，参数包括MPP、OSAL、CODEC、ISP、HAL和模式模块的命令。

所有命令都在`Inc/Rk_mpi_cmd.h`中定义。

通常命令是在`mpp_create`和`mpp_init`之后设置的，但工作模式应在`mpp_init`之前设置。  
您需要指明工作模式（单一或高级），以帮助mpp选择线程。  
选择线程。

```c
typedef enum {
    MPP_OSAL_CMD_BASE                   = CMD_MODULE_OSAL,
    MPP_OSAL_CMD_END,

    MPP_CMD_BASE                        = CMD_MODULE_MPP,
    MPP_ENABLE_DEINTERLACE,
    MPP_SET_INPUT_BLOCK,
    MPP_SET_OUTPUT_BLOCK,
    MPP_CMD_END,

    MPP_CODEC_CMD_BASE                  = CMD_MODULE_CODEC,
    MPP_CODEC_GET_FRAME_INFO,
    MPP_CODEC_CMD_END,

    MPP_DEC_CMD_BASE                    = CMD_MODULE_CODEC | CMD_CTX_ID_DEC,
    MPP_DEC_SET_FRAME_INFO,             /* vpu api legacy control for buffer slot dimension init */
    MPP_DEC_SET_EXT_BUF_GROUP,          /* IMPORTANT: set external buffer group to mpp decoder */
    MPP_DEC_SET_INFO_CHANGE_READY,
    MPP_DEC_SET_INTERNAL_PTS_ENABLE,
    MPP_DEC_SET_PARSER_SPLIT_MODE,      /* Need to setup before init */
    MPP_DEC_SET_PARSER_FAST_MODE,       /* Need to setup before init */
    MPP_DEC_GET_STREAM_COUNT,
    MPP_DEC_GET_VPUMEM_USED_COUNT,
    MPP_DEC_SET_VC1_EXTRA_DATA,
    MPP_DEC_SET_OUTPUT_FORMAT,
    MPP_DEC_CMD_END,

    MPP_ENC_CMD_BASE                    = CMD_MODULE_CODEC | CMD_CTX_ID_ENC,
    /* basic encoder setup control */
    MPP_ENC_SET_ALL_CFG,                /* set MppEncCfgSet structure */
    MPP_ENC_GET_ALL_CFG,                /* get MppEncCfgSet structure */
    MPP_ENC_SET_PREP_CFG,               /* set MppEncPrepCfg structure */
    MPP_ENC_GET_PREP_CFG,               /* get MppEncPrepCfg structure */
    MPP_ENC_SET_RC_CFG,                 /* set MppEncRcCfg structure */
    MPP_ENC_GET_RC_CFG,                 /* get MppEncRcCfg structure */
    MPP_ENC_SET_CODEC_CFG,              /* set MppEncCodecCfg structure */
    MPP_ENC_GET_CODEC_CFG,              /* get MppEncCodecCfg structure */
    /* runtime encoder setup control */
    MPP_ENC_SET_IDR_FRAME,              /* next frame will be encoded as intra frame */
    MPP_ENC_SET_OSD_PLT_CFG,            /* set OSD palette, parameter should be pointer to 
                                            MppEncOSDPlt */
    MPP_ENC_SET_OSD_DATA_CFG,           /* set OSD data with at most 8 regions, parameter 
                                            should be pointer to MppEncOSDData */
    MPP_ENC_GET_OSD_CFG,
    MPP_ENC_SET_EXTRA_INFO,
    MPP_ENC_GET_EXTRA_INFO,             /* get vps / sps / pps from hal */
    MPP_ENC_SET_SEI_CFG,                /* SEI: Supplement Enhancemant Information, parameter 
                                            is MppSeiMode */
    MPP_ENC_GET_SEI_DATA,               /* SEI: Supplement Enhancemant Information, parameter
                                            is MppPacket */
    MPP_ENC_CMD_END,

    MPP_ISP_CMD_BASE                    = CMD_MODULE_CODEC | CMD_CTX_ID_ISP,
    MPP_ISP_CMD_END,

    MPP_HAL_CMD_BASE                    = CMD_MODULE_HAL,
    MPP_HAL_CMD_END,

    MPI_CMD_BUTT,
} MpiCmd;

// you can call like this.
    ret = mpi->control(ctx, xx_xx_cmd, &data);
```

### 码率控制

Mpp支持三种码率控制模式：CBR（恒定码率）、VBR（可变码率）和 CQP（固定QP）。  
Mpp根据三个参数集控制码率，包括`MppEncCodecCfg`、`MppEncPrepCfg`和`MppEncRcCfg`。  

---

MppEncPrepCfg：

| param    |  effect                     |
| --- | --- |
| format   |  input pix format, such as NV12, RGB24|
| width    |  wdith of picture              |
| height   |  height of picture              |
| stride   |  span of a line in picture, must align in 8 |

---
<br>
<br>

---

MppEncRcCfg：


| param    |  effect                     |
| --- | --- |
| rc_mode  |  0:CBR<br>1:VBR                     |
| bps_target |  target bitrate                |
| bps_max/min |  max/min value of target bitrate     |
| fps_in_num/denorm |  input frame rate / denorm        |
|fps_out_num/denorm | output frame rate / denorm        |
| gop     |  group of picture, usually set as 4 *fps |
| quality   | profile of VBR mode, worst,worse,medium,better,best and CQP,<br>note that it will let qp_max/qp_min lose effect.<br>{31,       51}, // worst<br>{28,       46}, // worse<br>{24,       42}, // medium<br> {20,       39}, // better<br>{16,       35}, // best<br>{0,         0}, // cqp <br> cqp means use qp_init,qp will be const in each frame. |

---

<br>
<br>

---

MppEncCodecCfg:

| param    |  effect                     |
| --- | --- |
| qp_init  |  usually set as 33, if set at 0, mpp will estimate it by width,height... |
| qp_min/qp_max |  min/max value of qp             |
| qp_max_step | The maximum range of QP differences between two frames |
|  profile |  44   CAVLC 4:4:4<br>66   Baseline<br>77   Main<br>88   Extended<br>100  High (suggest)         |
|  level  |  10   Level 1.0<br>99   Level 1.b<br>11   Level 1.1<br>12   Level 1.2<br>13   Level 1.3<br>20   Level 2.0<br>21   Level 2.1<br>22   Level 2.2<br>30   Level 3.0<br>31   Level 3.1<br>32   Level 3.2<br>40   Level 4.0<br>41   Level 4.1<br>42   Level 4.2<br>50   Level 5.0<br>51   Level 5.1               |
| entroy_coding_mode|  entroy code mode<br>1:CAVLC<br>0:CABAC         |
| cabac_init_idc|  effect when entroy mode is CABAC    |

---
<br>
<br>

#### 1. CBR

在CBR模式下，编码数据的码率控制在配置的最大码率和最小码率范围内。  

---

推荐参数：

| param    |  effect                     |
| --- | --- |
| rc_mode  |    MPP_ENC_RC_MODE_CBR             |
| qp_init  | 0(suggest, mpp internal will set it)<br>24(high bitrate)<br>32(medium bitrate)<br>40(low bitrate)   |
| qp_min   | suggest 0, mpp internal will set at 16 |
| qp_max   | suggest 0, mpp internal will set at 48 |
| qp_max_step | 16                         |

---

#### 2. VBR

在VBR模式下，mpp会尽量保持视频质量稳定，并使编码文件的码率在给定的最大和最小值范围内尽可能波动。  

---

推荐参数：

| param    |  effect                     |
| --- | --- |
| rc_mode   |   MPP_ENC_RC_MODE_VBR              |
| quality  |  worst(very low bitrate)<br>worse(low bitrate)<br>medium(medium bitrate)<br>better(high bitrate)<br>best(very high bitrate)<br>CQP（const qp,rate control lose effect）         |
| qp_init  |  0(suggest, mpp internal estimate)<br>24(high bitrate)<br>32(medium bitrate)<br>40(low bitrate)   |
| qp_min   |  0                                    |
| qp_max   |  0                               |
| qp_max_step | 8, set at 0 when CQP                |

---

### Mpp接口使用

1. `MPP_RET Mpp::put_packet(MppPacket packet)`
封装为`mpi>put_packet`，用于与解码器内部的数据包列表交互。

2. `MPP_RET Mpp::get_frame(MppFrame *frame)`  
封装为`mpi>get_frame`，用于与解码器中的内部帧列表交互。解码时调用。  
`mpi->put_packet`  
|  
`mpi->get_frame`

3. `MPP_RET Mpp::put_frame(MppFrame frame)`  
封装为`mpi->put_frame`，用于与编码器中的内部帧列表交互。

4. `MPP_RET Mpp::get_packet(MppPacket *packet)`  
封装为`mpi>get_packet`，用于在编码器中与内部数据包列表交互。编码时这样调用。  
`mpi->put_frame`  
|  
`mpi->get_packet`

5. `MPP_RET Mpp::poll(MppPortType type, MppPollType timeout)` 
封装为`mpi>poll`，poll通常用于任务模式（高级模式）。poll将会检查内部的任务列表，如果`MPP_INPUT_PORT/MPP_OUTPUT_PORT`为空，则poll将阻塞直到任务列表获取到节点，否则，轮询将立即返回。  
关于任务模式如何工作，请参见`doc/design/4.mpp_task.txt`。

6. `MPP_RET Mpp::dequeue(MppPortType type, MppTask *task)`  
封装为`mpi->dequeue`，将得到一个包含多个数据的任务。当端口为MPP_INPUT_PORT时，获取设置任务。当端口为MPP_OUTPUT_PORT时，获取一个包含编码数据的任务。

7. `MPP_RET Mpp::enqueue(MppPortType type, MppTask task)` 
封装为`mpi->enqueue`，将向内部任务列表发送任务，从而使列表获得一个节点。 编码时可以这样调用。  
`mpi->poll(input, xx) `  
|  
`mpi->dequeue(input, xx)`   
|  
`mpi->enqueue(input, xx)`  
|  
`mpi->poll(output, xx)`  
|  
`mpi->dequeue(output, xx)`  
|  
`mpi->enqueue(output, xx)`  

8. `MPP_RET Mpp::control(MpiCmd cmd, MppParam param)`  
封装为`mpi->control`，将更改mpp参数。

关于如何使用mpp编码、解码，请参见`test/mpi_encode_test.c`和`test/mpi_decode_test.c`

## Mpp开发参考手册

<https://opensource.rock-chips.com/wiki_File:MPP_Development_Reference.pdf>

## Mpp开发指南（CN）

<https://opensource.rock-chips.com/wiki_File:MPP_%E5%BC%80%E5%8F%91%E5%8F%82%E8%80%83_v0.3.pdf>
