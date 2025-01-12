原文地址：https://www.android-x86.org/source.html

# 从源码开始编译Android-x86内核

本页面描述了如何为x86平台构建Android的最新信息。要浏览我们的源代码，请参阅...

 - [Android-x86 at OSDN Gitiweb](https://git.osdn.net/view?a=project_list;pf=android-x86)
 - [Code repository list at OSDN](https://osdn.net/projects/android-x86/scm)
 - [Code repository list at SourceForge](https://sourceforge.net/p/android-x86/_list/git)

从我们的git仓库中编译Android到x86平台是很容易的。编译出来的镜像应该能够在真实的x86设备以及虚拟机（比如qemu或virtual box）上良好运行。只需按照以下指示进行操作。不需要额外的补丁。

## 在Android-x86的版本库中，存在多个分支

由于Android Open Source Project（AOSP）发展得非常迅速，我们已经创建了不同的分支来对应AOSP的不同版本发布。

- **r-x86**: 基于 Android 11.0 版本，代号为 R.
- **q-x86**: 基于 Android 10.0 版本，代号为 Q.
- **pie-x86**: 基于 Android 9.0 版本，代号为 Pie，且为 Pie QPR2 版本.
- **oreo-x86**: 基于 Android 8.1 版本，代号为 Oreo，且为 Oreo MR1 版本.
- **nougat-x86**: 基于 Android 7.1 版本，代号为 Nougat，且为 Nougat MR2 版本.
- **marshmallow-x86**: 基于 Android 6.0 版本，代号为 Marshmallow.
- **lollipop-x86**: 基于 Android 5.1 版本，代号为 Lollipop.
- **kitkat-x86**: 基于 Android 4.4 版本，代号为 KitKat.
- **jb-x86**: 基于 Android 4.3 版本，代号为 Jelly Bean.
- **ics-x86**: 基于 Android 4.0 版本，代号为 Ice Cream Sandwich.
- **honeycomb-x86**: 基于 Android 3.2 版本，代号为 Honeycomb.
- **gingerbread-x86**: 基于 Android 2.3 版本，代号为 Gingerbread.
- **froyo-x86**: 基于 Android 2.2 版本，代号为 Froyo.
- **eclair-x86**: 基于 Android 2.1 版本，代号为 Eclair.
- **donut-x86**: 基于 Android 1.6 版本，代号为 Donut.
- **cupcake-x86 (aka android-x86-b0.9)**: 基于 Android 1.5 版本，代号为 Cupcake，也被称为 android-x86-b0.9.

## 获取 Android-x86 的源代码

首先，查看AOSP页面“建立构建环境”以配置你的构建环境。对于Ubuntu 18.04，请安装以下所需的软件包：
```
$ sudo apt -y install git gcc curl make repo libxml2-utils flex m4
$ sudo apt -y install openjdk-8-jdk lib32stdc++6 libelf-dev mtools
$ sudo apt -y install libssl-dev python-enum34 python-mako syslinux-utils
```

然后，将Android-x86的源代码库下载到你的工作目录中，方法是：
```
$ mkdir android-x86
$ cd android-x86
$ repo init -u git://git.osdn.net/gitroot/android-x86/manifest -b $branch
$ repo sync --no-tags --no-clone-bundle
```

其中$branch是上一节中描述的分支名称之一。请注意，由 android-x86 创建或修改的项目是从我们的 Git 服务器获取的。所有其他项目仍从 [AOSP（Android 开源项目）](https://android.googlesource.com/)的代码库下载。

如果你在使用 Git 协议进行同步时遇到问题，可以尝试使用替代的 HTTP 协议来进行同步:
```
$ repo init -u http://scm.osdn.net/gitroot/android-x86/manifest -b $branch
$ repo sync --no-tags --no-clone-bundle
```

如果你希望继续将你的代码库与 Android-x86 仓库保持同步，只需执行 repo sync 命令即可，无需再次执行 repo init 命令。然而，有时在执行 repo sync 时可能会遇到冲突。请参阅“如何解决冲突”部分，了解如何解决这种情况。

注意：Android-x86 的代码库非常大（例如 oreo-x86 的版本就超过了 20GB）。如果你在同步（sync）这个代码库时遇到了问题，那么很可能是网络问题或者我们的服务器太忙了。你应该反复运行 repo sync 命令，直到它能够成功完成同步且没有任何错误为止。不要因为同步过程中遇到的任何问题来打扰我们。

## 获取LineageOS for x86源代码

要获取[LineageOS（前CyanogenMod）](https://www.lineageos.org/)移植到Android-x86的源代码，请在初始化仓库时通过额外的选项 -m cm.xml 指定清单文件：
```
$ repo init -u git://git.osdn.net/gitroot/android-x86/manifest -m cm.xml -b $branch
$ repo sync --no-tags --no-clone-bundle
```
目前，LineageOS移植仅在nougat-x86（cm 14.1）和marshmallow-x86（cm 13.0）分支中可用。

## 编译镜像

当代码库同步完成后，你可以构建一个CDROM ISO镜像文件。构建kitkat-x86版本之前（包括kitkat-x86）的分支时，你需要Oracle Java 1.6（OpenJDK可能不适用）。自lollipop-x86版本起，需要Java 1.7，并且支持OpenJDK。自nougat-x86版本起，需要OpenJDK 1.8。

**注意：**
在Froyo-x86版本（包括Froyo-x86）之前，你可以在32位或64位的主机上进行构建。从Gingerbread-x86版本开始，推荐使用64位的构建环境。

## 选择一个目标

你需要为你要使用或测试的x86设备选择一个目标。我们为不同的分支提供了几个目标：

### Lollipop-x86 及以后版本
- **android_x86_64**: 适用于 64 位 x86_64 平台。
- **android_x86**: 适用于 32 位 x86 平台。

### KitKat-x86 / Jelly Bean-x86
- **android_x86**: 适用于 x86 平台。

### Ice Cream Sandwich-x86 / Honeycomb-x86
- **generic_x86**: 适用于通用的 x86 PC/笔记本电脑.
- **amd_brazos**: 适用于 AMD Brazos 平台.
- **eeepc**: 仅适用于 ASUS EeePC 系列.
- **asus_laptop**: 适用于某些 ASUS 笔记本电脑.
- **tegav2**: 适用于 Tegatech Tegav2（可能也适用于其他基于 Atom N45x 的平板电脑）.

### Gingerbread-x86 / Froyo-x86
- **generic_x86**: 适用于通用的 x86 PC/笔记本电脑.
- **eeepc**: 仅适用于 ASUS EeePC 系列.
- **asus_laptop**: 适用于某些 ASUS 笔记本电脑.
- **tegav2**: 适用于 Tegatech Tegav2（可能也适用于其他基于 Atom N45x 的平板电脑）.
- **sparta**: 适用于 Dell Inspiron Mini Duo 平台.
- **vm**: 适用于虚拟机（Virtual Box、QEMU、VMware）.
- **motion_m1400**: 适用于 Motion M1400（基于 Intel Centrino M，配备 Intel PRO/Wireless）.

### Eclair-x86
- **generic_x86**: 适用于通用的 x86 PC/笔记本电脑.
- **eeepc**: 仅适用于 ASUS EeePC 系列.
- **q1u**: 适用于 Samsung Q1U.
- **s5**: 适用于 Viliv S5.

### Donut-x86
- **eeepc**: 仅适用于 ASUS EeePC 系列.
- **q1u**: 适用于 Samsung Q1U.
- **s5**: 适用于 Viliv S5.

### LineageOS for x86
- **cm_android_x86_64**: 适用于 64 位 x86_64 平台.
- **cm_android_x86**: 适用于 32 位 x86 平台.

如果你不是在构建一个非常古老的Android分支（比如较新的版本），你应该使用android_x86_64作为64位目标设备，或者使用android_x86作为32位目标设备。这两个目标是为所有x86设备设计的通用目标，适用于大多数情况。

在较早的Android版本中（从Eclair-x86到ICS-x86分支），通常建议使用generic_x86作为目标设备。然而，这个目标可能没有针对特定设备进行优化，也就是说它可能无法充分发挥特定硬件的性能。对于更早的版本（如Donut-x86分支或之前的版本），如果你没有特定设备的支持目标，可以使用eeepc作为通用的x86目标设备。

如果你是一名开发者，并且希望为你的设备创建一个自定义的目标设备，你可以基于android_x86来创建。看这篇[文章](https://www.android-x86.org/documentation/new_target.html)。除非你是一名经验丰富的Android平台开发者，否则不建议这样做，因为这可能需要深入的了解和复杂的操作。

## 使用lunch命令（推荐）

你可以将文件 build/envsetup.sh 引入到你的 Bash 环境中，以便获得一些帮助构建过程的 shell 函数：
```
$ source build/envsetup.sh
```
现在你可以通过使用lunch命令来选择一个目标。
```
$ lunch $TARGET_PRODUCT-$TARGET_BUILD_VARIANT
```

其中，$TARGET_PRODUCT 可以是前一节中描述的任何目标，$TARGET_BUILD_VARIANT 的可能值是 eng、user 和 userdebug。
例如，
```
$ lunch android_x86_64-userdebug
```

然后你可以通过使用 m 命令来创建一个 ISO 文件:
```
$ m -jX iso_img
```
m 命令与 make 命令等效，但你可以在 android-x86 项目的任何子目录中使用它。将 X 替换为你拥有的处理器数量。例如，如果你有一个四核 CPU，则将 X 替换为 4。
```
$ m -j4 iso_img
```
从froyo-x86版本开始，我们在lunch命令中也增加了菜单选择功能。只需输入lunch，你就会得到一个可用目标的列表。输入数字来选择一个目标。或者，你也可以直接输入lunch $number来选择目标。

构建一个rpm文件：
```
$ m -jX rpm
```
在开始构建之前，你需要安装一个特定的软件包。具体来说：
如果你使用的是基于 Fedora 的 Linux 发行版，你需要安装名为 rpm-build 的软件包。
如果你使用的是基于 Debian 或 Ubuntu 的 Linux 发行版，你需要安装名为 rpm 的软件包。

rpm文件可以直接安装到Linux发行版中。

## 直接编译

你可以通过设置一个名为 TARGET_PRODUCT 的变量来指定要构建的目标。例如，如果你想为名为 android_x86 的目标构建一个 ISO 镜像，你可以输入相应的命令来实现这一目标：
```
$ make -jX iso_img TARGET_PRODUCT=android_x86
```
要为 tegav2 生成一个可启动的 CD-ROM ISO 文件，输入以下命令:
```
$ make -jX iso_img TARGET_PRODUCT=tegav2
```
然后你会得到一个 ISO 文件，位于 out/target/product/x86/android_x86.iso，等等。

## 使用buildspec.mk文件

你可以在你的 android-x86 目录中创建一个名为 buildspec.mk 的文件，以便记住你经常构建的某个特定目标产品：
```
TARGET_PRODUCT := android_x86_64
TARGET_BUILD_VARIANT := userdebug
TARGET_BUILD_TYPE := release
TARGET_KERNEL_CONFIG := android-x86_64_defconfig
```
当你在你的 android-x86 工作目录中有 buildspec.mk 文件时，你可以简单地通过某种方式来构建（或编译）项目:
```
make -jX iso_img
```

## 编译更小的镜像

自 Android-x86 的 marshmallow-x86 版本起，默认情况下生成的 Android-x86 核心文件系统将使用 squashfs 进行压缩。在该版本之前，仅当你在宿主机上安装了 squashfs-tools 4.0（旧版本将无法工作）时才会使用 squashfs。使用 squashfs 压缩后生成的 ISO 文件会小很多（大约只有原来的 30-40%）。然而，如果你出于某些原因希望禁用 squashfs 压缩，可以在 make 命令中添加 USE_SQUASHFS=0。你可以将此设置放入 buildspec.mk 文件中：
```
USE_SQUASHFS := 0
```

在froyo-x86版本（包括该版本）之前，如果你希望获得一个更小的镜像文件，你可以通过移除调试符号来实现这一点：
```
TARGET_STRIP := 1
```
从 gingerbread-x86 版本开始，默认情况下会移除调试符号。因此，相关的选项（可能是用于控制是否移除调试符号的选项）变得没有必要了。

----------------下面的内容相对没那么重要---------------------------------

## 测试
