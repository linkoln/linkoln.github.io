[英语原文](https://android.googlesource.com/platform/external/qemu/+/refs/heads/studio-1.4-release/distrib/android-emugl/DESIGN)

Android硬件OpenGLES仿真设计概述
===================================================

简介:
-------------

在Android平台上，硬件OpenGLES仿真是通过多种组件的组合来实现的，这些组件包括:

  - 几个主机端的“翻译器”库。它们实现了由Khronos定义的EGL、GLES 1.1和GLES 2.0的应用二进制接口（ABI），并将相应的函数调用转换为适配桌面API的调用，即:

      - GLX (Linux), AGL (OS X) or WGL (Windows) for EGL
      - desktop GL 2.0 for GLES 1.1 and GLES 2.0

         _________            __________          __________
        |         |          |          |        |          |
        |TRANSLATOR          |TRANSLATOR|        |TRANSLATOR|     HOST
        |   EGL   |          | GLES 1.1 |        | GLES 2.0 |     TRANSLATOR
        |_________|          |__________|        |__________|     LIBRARIES
             |                    |                    |
       - - - | - - - - - - - - -  | - - - - - - - - -  | - - - - -
             |                    |                    |
         ____v____            ____v_____          _____v____      HOST
        |         |          |          |        |          |     SYSTEM
        |   GLX   |          |  GL 2.0  |        |  GL 2.0  |     LIBRARIES
        |_________|          |__________|        |__________|



  - 在被仿真的客户机系统（即虚拟系统）中，有若干系统库，它们实现了相同的EGL、GLES 1.1和GLES 2.0应用二进制接口（ABI）。

    它们收集EGL/GLES函数调用的序列，并将其翻译成一种自定义的通信协议流，然后通过一个名为“QEMU管道”的高速通信通道发送到模拟器程序中。

    目前，你只需要知道，这个管道是通过一个自定义的内核驱动程序实现的，并且它提供了非常快的带宽。从客户机的角度来看，所有对管道的读取（`read()`）和写入（`write()`）操作几乎是瞬间完成的。


         _________            __________          __________
        |         |          |          |        |          |
        |EMULATION|          |EMULATION |        |EMULATION |     GUEST
        |   EGL   |          | GLES 1.1 |        | GLES 2.0 |     SYSTEM
        |_________|          |__________|        |__________|     LIBRARIES
             |                    |                    |
       - - - | - - - - - - - - -  | - - - - - - - - -  | - - - - -
             |                    |                    |
         ____v____________________v____________________v____      GUEST
        |                                                   |     KERNEL
        |                       QEMU PIPE                   |
        |___________________________________________________|
                                  |
       - - - - - - - - - - - - - -|- - - - - - - - - - - - - - - -
                                  |
                                  v
                               EMULATOR

  - 模拟器程序中包含的特定代码能够将通信协议流传输到一个特殊的渲染库或进程（在这里称为“渲染器”），该渲染器能够理解这种格式。

                                 |
                                 |    PROTOCOL BYTE STREAM
                            _____v_____
                           |           |
                           |  EMULATOR |
                           |___________|
                                 |
                                 |   UNMODIFIED PROTOCOL BYTE STREAM
                            _____v_____
                           |           |
                           |  RENDERER |
                           |___________|


  - 渲染器从通信协议流中解码EGL/GLES命令，并将这些命令适当地分发到翻译器库中。

                                 |
                                 |   PROTOCOL BYTE STREAM
                            _____v_____
                           |           |
                           |  RENDERER |
                           |___________|
                               | |  |
             +-----------------+ |  +-----------------+
             |                   |                    |
         ____v____            ___v______          ____v_____
        |         |          |          |        |          |     HOST
        |TRANSLATOR          |TRANSLATOR|        |TRANSLATOR|     HOST
        |   EGL   |          | GLES 1.1 |        | GLES 2.0 |     TRANSLATOR
        |_________|          |__________|        |__________|     LIBRARIES



  - 实际上，协议流是双向流动的，尽管大多数命令的结果是数据从客户机流向主机。因此，完整的仿真过程图景应该是:





         _________            __________          __________
        |         |          |          |        |          |
        |EMULATION|          |EMULATION |        |EMULATION |     GUEST
        |   EGL   |          | GLES 1.1 |        | GLES 2.0 |     SYSTEM
        |_________|          |__________|        |__________|     LIBRARIES
             ^                    ^                    ^
             |                    |                    |
       - - - | - - - - - - - - -  | - - - - - - - - -  | - - - - -
             |                    |                    |
         ____v____________________v____________________v____      GUEST
        |                                                   |     KERNEL
        |                       QEMU PIPE                   |
        |___________________________________________________|
                                 ^
                                 |
      - - - - - - - - - - - - - -|- - - - - - - - - - - - - - - -
                                 |
                                 |    PROTOCOL BYTE STREAM
                            _____v_____
                           |           |
                           |  EMULATOR |
                           |___________|
                                 ^
                                 |   UNMODIFIED PROTOCOL BYTE STREAM
                            _____v_____
                           |           |
                           |  RENDERER |
                           |___________|
                               ^ ^  ^
                               | |  |
             +-----------------+ |  +-----------------+
             |                   |                    |
         ____v____            ___v______          ____v_____
        |         |          |          |        |          |
        |TRANSLATOR          |TRANSLATOR|        |TRANSLATOR|     HOST
        |   EGL   |          | GLES 1.1 |        | GLES 2.0 |     TRANSLATOR
        |_________|          |__________|        |__________|     LIBRARIES
             ^                    ^                    ^
             |                    |                    |
       - - - | - - - - - - - - -  | - - - - - - - - -  | - - - - -
             |                    |                    |
         ____v____            ____v_____          _____v____      HOST 
        |         |          |          |        |          |     SYSTEM
        |   GLX   |          |  GL 2.0  |        |  GL 2.0  |     LIBRARIES
        |_________|          |__________|        |__________|

    (注意：“GLX”仅适用于Linux，对于OS X应替换为“AGL”，对于Windows应替换为“WGL”)。


请注意，在上述图形中，只有底部的主机系统库不是由Android提供的。


设计要求:
--------------------

上述设计源于项目初期确定的几个重要要求。:

1 - 能够在与模拟器本身不同的进程中运行渲染器是重要的。

    由于各种实际原因，我们计划通过使用两个不同的进程，将核心QEMU仿真与UI窗口完全分离。因此，渲染器将作为UI程序中的一个库来实现，但它需要从QEMU进程接收协议字节。

    这两个进程之间的通信通道将是一个快速的Unix套接字或Win32命名管道。如果性能成为一个问题，还可能会使用带有适当同步原语的共享内存段。

    这解释了为什么模拟器不会更改甚至尝试解析协议字节流。它只是作为客户机系统和渲染器之间的一个“哑”代理。这也避免了在极其复杂的QEMU代码库中添加大量与GLES相关的代码。

2 - 能够使用供应商特定的桌面EGL/GLES库是很重要的。

    GPU供应商（如NVIDIA、AMD或ARM）都提供了主机版本的EGL/GLES库，用于模拟其各自的嵌入式图形芯片。

    渲染器库可以配置为使用这些供应商提供的库，而不是本项目提供的翻译器库。这有助于更准确地模拟特定设备的行为。

    此外，这些供应商库通常会暴露一些供应商特定的扩展功能，而这些功能是翻译器库所不提供的。虽然我们无法在不修改代码的情况下暴露这些扩展功能，但能够相对轻松地做到这一点是非常重要的。


代码组织:
------------------

上述组件的所有源代码分布在Android源代码树的多个目录中。:

  - 模拟器的源代码位于`$ANDROID/external/qemu`目录下，在本文档的其余部分，我们将用`$QEMU`来指代这
    目录。

  - 客户端库位于`$ANDROID/device/generic/goldfish/opengl`，在本文档中我们将其称为`$EMUGL_GUEST`。

  - 主机端渲染器和翻译器库位于`$QEMU/distrib/android-emugl`，在本文档中我们将其称为`$EMUGL_HOST`。

  - QEMU管道内核驱动程序位于`$KERNEL/drivers/misc/qemupipe`（适用于3.4内核版本）或`$KERNEL/     drivers/platform/goldfish/goldfish_pipe.c`（适用于3.10内核版本）。

其中，`$ANDROID`是开源Android源代码树的根目录，而`$KERNEL`是针对QEMU的内核源代码树的根目录（这里使用的是其中一个`android-goldfish-xxxx`分支）。

与该项目相关的模拟器源代码是:

   $QEMU/hw/android/goldfish/pipe.c -> 实现了QEMU管道虚拟硬件。
   $QEMU/android/opengles.c         -> 实现了GLES初始化。
   $QEMU/android/hw-pipe-net.c      -> 实现了QEMU管道与渲染器库之间的通信通道。

其他源代码是:

   $EMUGL_GUEST/system   -> 系统库
   $EMUGL_GUEST/shared   -> 客户机端的共享库副本
   $EMUGL_GUEST/tests    -> 各种测试程序

   $EMUGL_HOST/host      -> 主机库（翻译器+渲染器）
   $EMUGL_HOST/shared    -> 主机端的共享库副本

共享库实际上没有共享的原因是历史遗留问题：曾经客户机和主机代码都位于同一个地方。然而，这种方式在Android SDK的分支管理中被证明是不切实际的，并且无法支持一个单一模拟器二进制文件能够运行多个Android版本的要求。


翻译器库:
---------------------

这个项目提供了三个翻译器主机库:

   libEGL_translator       -> EGL 1.2 translation
   libGLES_CM_translator   -> GLES 1.1 translation
   libGLES_V2_translator   -> GLES 2.0 translation

库的完整名称将取决于主机系统。为了简化，只有库的名称后缀会发生变化（即在Windows上不会去掉“lib”前缀），例如:

   libEGL_translator.so    -> for Linux
   libEGL_translator.dylib -> for OS X
   libEGL_translator.dll   -> for Windows

这些库的源代码位于Android源代码树的以下路径中:

   $EMUGL_HOST/host/libs/Translator/EGL
   $EMUGL_HOST/host/libs/Translator/GLES_CM
   $EMUGL_HOST/host/libs/Translator/GLES_V2

翻译器库还使用了在以下路径定义的通用例程:

   $EMUGL_HOST/host/libs/Translator/GLcommon


通信协议概述:
----------------------

“通信协议”是按照以下方式实现的:

  - EGL/GLES函数调用是通过几个“规范”文件来描述的，这些文件定义了每个函数的类型、函数签名以及各种属性。

  - 这些文件由一个名为“emugen”的工具读取，该工具根据规范生成C语言源文件和头文件。这些文件对应于编码、解码以及“包装器”（稍后会详细说明）。

  - 使用这些生成的文件中的一部分构建了系统“编码器”静态库。它们包含可以将EGL/GLES调用序列化为简单的字节消息并通过通用的“I/O流”对象发送的代码。

  - 主机端的“解码器”静态库也是使用这些生成的文件中的一部分构建的。它们的代码从一个“I/O流”对象中提取字节消息，并将其转换为函数回调。

I/O流抽象:
- - - - - - - - - - -

“I/O流”是一个非常简单的抽象类，用于在客户机和主机之间发送字节消息。它通过一个共享的头文件`$EMUGL_HOST/include/libOpenglRender/IOStream.h`定义。

请注意，尽管路径如此，这个头文件被客户机和主机的源代码都包含了。I/O流设计的主要思想是，为了发送一条消息，需要按照以下步骤操作:

  1/ 调用`stream->allocBuffer(size)`，该函数返回一个至少包含`size`字节的内存缓冲区的地址。

  2/ 将序列化命令的内容（通常是头部加上一些有效载荷）直接写入该缓冲区。

  3/ 调用`stream->commitBuffer()`来发送它。

或者，也可以使用`stream->alloc()`和`stream->flush()`将多个命令打包到一个缓冲区中，如下所示:

  1/ buf1 =  stream->alloc(size1)
  2/ 将第一条命令的字节写入`buf1`
  3/ buf2 = stream->alloc(size2)
  4/ 将第二条命令的字节写入`buf2`
  5/ stream->flush()

最后，还有一些明确的读取/写入方法，例如`stream->readFully()`或`stream->writeFully()`，这些方法可以在你不想使用中间缓冲区时使用。在某些情况下，实现中会使用这些方法，例如在将纹理数据从客户机发送到主机时，避免中间的内存拷贝。

主机端的`IOStream`实现位于`$EMUGL_SHARED/OpenglCodecCommon/`，特别是以下内容:

   $EMUGL_HOST/shared/OpenglCodecCommon/TcpStream.cpp
      -> 使用本地TCP套接字
   $EMUGL_HOST/shared/OpenglCodecCommon/UnixStream.cpp
      -> 使用Unix套接字
   $EMUGL_HOST/shared/OpenglCodecCommon/Win32PipeStream.cpp
      -> 使用Win32命名管道

客户机端的`IOStream`实现使用了上面提到的`TcpStream.cpp`，以及一个替代的QEMU特定的源代码。:

   $EMUGL_GUEST/system/OpenglSystemCommon/QemuPipeStream.cpp
      -> 使用客户机端的QEMU管道

QEMU管道的实现显著更快（大约快20倍），原因有多个:

  - 从客户机的角度来看，通过它进行的所有成功的`read()`和`write()`操作都是瞬间完成的。

  - 所有缓冲区/内存拷贝操作都是由模拟器直接完成的，因此比在内核中通过模拟的ARM指令执行相同操作要快得多。

  - 它不需要经过内核的TCP/IP协议栈，将数据封装成TCP/IP/MAC数据包，发送到模拟的以太网设备，而该设备又连接到一个内部防火墙实现，防火墙会解包这些数据包，重新组装它们，然后通过BSD套接字发送到主机内核。

然而，如果有必要，你可以编写一个使用不同传输方式的客户机`IOStream`实现。如果你打算这么做，请查看`$EMUGL_GUEST/system/OpenglCodecCommon/HostConnection.cpp`，其中包含用于在每个线程基础上将客户机连接到主机的代码。


源代码自动生成:
- - - - - - - - - - - - - -

`emugen`工具位于`$EMUGL_HOST/host/tools/emugen`目录下。该目录下有一个`README`文件，其中解释了它是如何工作的。

你也可以查看以下规范文件。:

对于GLES 1.1:
    $EMUGL_HOST/host/GLESv1_dec/gl.types
    $EMUGL_HOST/host/GLESv1_dec/gl.in
    $EMUGL_HOST/host/GLESv1_dec/gl.attrib

对于GLES 2.0:
    $EMUGL_HOST/host/GLESv2_dec/gl2.types
    $EMUGL_HOST/host/GLESv2_dec/gl2.in
    $EMUGL_HOST/host/GLESv2_dec/gl2.attrib

对于EGL:
    $EMUGL_HOST/host/renderControl_dec/renderControl.types
    $EMUGL_HOST/host/renderControl_dec/renderControl.in
    $EMUGL_HOST/host/renderControl_dec/renderControl.attrib

请注意，EGL的规范文件位于一个名为`renderControl_dec`的目录中，且文件名以`renderControl`开头。

这主要是出于历史原因，但也与这样一个事实有关：这部分通信协议包含了一些支持函数/调用/规范，这些内容本身并不属于EGL规范，但添加了一些功能，以确保整个系统能够正常运行。例如，它们包含与“gralloc”系统库模块相关的调用，该模块用于在比EGL更低的层次上管理图形表面。

一般来说，客户机端的编码器源代码位于名为`$EMUGL_GUEST/system/<name>_enc/`的目录下，而相应的主机端解码器源代码则位于`$EMUGL_HOST/host/libs/<name>_dec/`目录下。

然而，所有这些源代码都使用位于解码目录下的相同规范文件。

编码器文件是由`emugen`工具根据位于`$EMUGL_HOST`的规范文件生成的，并通过`gen-encoder.sh`脚本复制到`$EMUGL_GUEST`中的编码器目录中。这些文件已经被纳入版本控制，以便某个特定版本的Android能够支持一种特定版本的协议，即使较新的渲染器版本（以及未来的Android版本）支持更新的协议版本。当协议发生变化时，需要手动执行这一步骤；这些变化还需要伴随着渲染器中的变化，以处理旧版本的协议。


系统库:
-----------------

元EGL/GLES系统库，以及`egl.cfg`:
- - - - - - - - - - - - - - - - - - - - - -

重要的是要理解，仿真专用的EGL/GLES库在运行时并不是直接被应用程序链接的。相反，系统提供了一组“元”EGL/GLES库，这些库会在首次使用时加载适当的硬件专用库。

更具体地说，系统库`libEGL.so`包含一个“加载器”，它将尝试加载:

  - 硬件专用的EGL/GLES库
  - 基于软件的渲染库（称为“libagl”）

系统库`libEGL.so`还能够将硬件库和软件库的EGL配置透明地合并给应用程序。系统库`libGLESv1_CM.so`和`libGLESv2.so`与之配合，确保线程的当前上下文将根据所选配置链接到硬件库或软件库。

为了记录，加载器的源代码位于`frameworks/base/opengl/libs/EGL/Loader.cpp`。它依赖于一个名为`/system/lib/egl/egl.cfg`的文件，该文件必须包含如下两条内容:

    0 1 <name>
    0 0 android

每行的第一个数字是显示设备编号，必须为0，因为系统的EGL/GLES库不支持其他编号。

第二个数字必须为1以表示硬件库，为0以表示软件库。如果存在硬件库，对应的行必须出现在软件库的行之前。

第三个字段是一个与共享库后缀对应的名称。它实际上意味着相应的库将被命名为`libEGL_<name>.so`、`libGLESv1_CM_<name>.so`和`libGLESv2_<name>.so`。此外，这些库必须放置在`/system/lib/egl/`目录下。

名称“android”被保留用于系统的软件渲染器。

这个项目附带的`egl.cfg`文件使用名称“emulation”来指代硬件库。这意味着它提供了一个包含以下内容的`egl.cfg`文件:

   0 1 emulation
   0 0 android

请查看`$EMUGL_GUEST/system/egl/egl.cfg`，以及更一般的以下构建文件:

   $EMUGL_GUEST/system/egl/Android.mk
   $EMUGL_GUEST/system/GLESv1/Android.mk
   $EMUGL_GUEST/system/GLESv2/Android.mk

以了解构建系统是如何为库命名的，以及放置在`/system/lib/egl/`下的。


仿真库:
- - - - - - - - - - -

特定于模拟器的库位于以下路径:

  $EMUGL_GUEST/system/egl/
  $EMUGL_GUEST/system/GLESv1/
  $EMUGL_GUEST/system/GLESv2/

GLESv1和GLESv2的代码量相对较小，因为它们主要链接到静态编码库。

EGL的代码相对复杂一些，因为它需要动态处理扩展功能。也就是说，如果某个扩展在主机上不可用，那么在运行时该库就不应该暴露这个扩展。因此，EGL代码会查询主机以获取可用扩展的列表，并将这些扩展返回给客户端。同样地，它也需要查询当前主机系统上有效的EGL配置（EGLConfigs）列表。


“gralloc”模块的实现:
- - - - - - - - - - - - - - - - -

除了EGL/GLES库之外，Android系统还需要一个硬件专用库，用于在比EGL更低的层次上管理图形表面。这个库必须是Android中所称的“硬件抽象层模块”（HAL模块）。

一个“硬件抽象层模块”（HAL模块）必须提供由Android的HAL（硬件抽象层库）定义的接口。这些接口定义可以在`$ANDROID/hardware/libhardware/include/`目录下找到。

在所有可能的HAL模块中，“gralloc”模块被系统的SurfaceFlinger用于分配帧缓冲区和其他图形内存区域，以及在需要时锁定、解锁和交换它们。

位于`$EMUGL/system/gralloc/`的代码实现了GLES仿真项目所需的模块。代码并不长，但有一些要点需要注意:

- 首先，它会探测客户机系统，以确定运行虚拟设备的模拟器是否真正支持GPU仿真。在某些情况下，这可能有点不可能。

  如果情况如此，那么该模块将把所有调用重定向到通常在启用纯软件渲染时由系统使用的“默认”gralloc模块。

  探测过程发生在`fallback_init`函数中，该函数在模块首次被打开时被调用。当需要时，它会将`sFallback`变量初始化为指向默认gralloc模块的指针。

- 其次，这个模块被SurfaceFlinger用于显示“软件表面”，即那些由系统内存像素缓冲区支持的表面，这些表面通过Skia图形库直接写入（即非加速的表面）。

  默认模块只是将表面的像素数据复制到虚拟帧缓冲区的I/O内存中，但本项目的gralloc模块则通过QEMU管道将数据发送到渲染器。

  事实证明，这使得整体渲染速度和帧率更快，因为在客户机内部的内存拷贝操作速度较慢，而通过QEMU管道的传输则直接在模拟器中完成，速度更快。


主机渲染器:
--------------

主机渲染器库位于`$EMUGL_HOST/host/libs/libOpenglRender`，它提供了一个由`$EMUGL_HOST/host/libs/libOpenglRender/render_api.h`头文件描述的接口（例如，供模拟器使用）。

简而言之，渲染库负责以下内容:

  - 提供一个虚拟的离屏视频表面，所有内容在运行时都会渲染到这个表面上。它的尺寸由`initOpenglRender()`调用确定，该调用必须在库初始化后立即发生。

  - 提供一种方法，将虚拟视频表面显示在主机应用程序的UI上。这是通过调用`createOpenGLSubWindow()`实现的，该函数的参数包括父窗口的窗口ID或句柄、一些显示尺寸以及旋转角度。这使得表面在显示时可以被缩放/旋转，即使视频表面的尺寸没有改变。

  - 提供一种方法来监听来自客户机的传入EGL/GLES命令。这是通过向`initOpenglRender()`提供一个所谓的“端口号”来实现的。

    默认情况下，端口号对应于一个本地TCP端口号，渲染器将绑定并监听该端口。每次连接到该端口都将对应于一个新的客户机与主机的连接，每个这样的连接对应于客户机系统中的一个独立线程。

    出于性能原因，可以监听Unix套接字（在Linux和OS X上）或Win32命名管道（在Windows上）。为此，需要在库初始化（即`initLibrary()`）和构造（即`initOpenglRender()`）之间调用`setStreamType()`。

    请注意，在这些模式下，端口号仍然用于区分多个模拟器实例。这些细节通常由模拟器代码处理，因此你不需要太在意。

请注意，早期版本的接口允许渲染器库的客户端提供自己的`IOStream`实现。然而，由于多种原因，这并不是非常方便。如果有必要，这或许可以再次实现，但目前的性能数据已经相当不错。


主机模拟器:
--------------

位于`$QEMU/android/opengles.c`的代码负责动态加载渲染库并正确地初始化/构造它。

对“opengles”服务的QEMU管道连接通过`$QEMU/android/hw-pipe-net.c`中的代码进行处理。可以查找`openglesPipe_init()`函数，该函数负责在客户机进程通过`/dev/qemu_pipe`打开“opengles”服务时，创建与渲染库的连接（根据配置，通过TCP套接字或Unix管道。模拟器尚未实现对Win32命名管道的支持）。

在`$QEMU/skin/window`目录下也有一些支持代码，用于显示GLES帧缓冲区（通过渲染库的子窗口）。

请注意，目前支持缩放和旋转。然而，亮度仿真（过去会修改硬件帧缓冲区中的像素值，然后再显示它们）目前无法工作。

另一个问题是，目前无法在GL子窗口上方显示任何内容。例如，这会遮挡模拟轨迹球的图像（通常在仿真过程中通过按Ctrl-T切换显示，或者通过按Delete键启用）。

