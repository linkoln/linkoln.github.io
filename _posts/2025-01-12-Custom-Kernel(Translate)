# 自定义内核

Android构建系统不会即时编译内核。它只是包含一个预先构建好的内核二进制文件，这个文件会被添加到目标镜像中。对于arm模拟器目标来说，这种方法可能足够好，但对于x86平台则不太合适。因为x86平台有各种各样的硬件，内核二进制文件及其模块可能需要在编译时或运行时进行调整。

本文解释了android-x86构建系统的一个额外功能。也就是说，在构建过程中，能够通过预定义的或自定义的配置来构建内核和模块。

## 准备源代码树

我们已经修改了Android构建系统，使其能够在飞行中（即时）编译内核镜像。你需要使用我们的仓库才能获得这一功能。详情请阅读[文章](https://linkoln.github.io/2025/01/07/Android-x86/)。

## 构建目标的默认内核

我们在 kernel/arch/x86/configs/ 这个目录下放置了 android-x86 所支持目标的默认配置文件。当选择了一个特定的目标时，与之对应的 defconfig 配置文件会自动被使用。例如:
```
$ make iso_img TARGET_PRODUCT=android_x86_64
```

通过将 TARGET_PRODUCT 指定为 android_x86_64，构建系统会自动选择配置 android-x86_64_defconfig 来编译内核二进制文件及其模块。内核二进制文件将生成在 out/target/product/x86_64/kernel 路径下，模块则会被放置在 out/target/product/x86_64/system/lib/modules/ 路径下。最终的目标文件 out/target/product/x86_64/android_x86_64.iso 将包含内核二进制文件及其模块。

## 仅构建和更新内核

为了单独构建内核及其模块，需要将构建目标从iso_img改为kernel。
```
$ . build/envsetup.sh; lunch android_x86_64-userdebug
$ make kernel
```

然后你可以将$OUT/kernel和$OUT/system/lib/modules/复制到目标设备上。将前者放到android-x86的安装目录，后者放到/system/lib/modules。注意/system必须以读写模式安装。

## 指定内核架构

从Android 5.0版本开始，它支持32位和64位的镜像。通常情况下，32位的用户空间与32位的内核配合工作，而64位的用户空间则必须与64位的内核配合工作。Android-x86的构建系统自lollipop-x86版本起就同时支持32位和64位系统。

有时候你可能想要在64位的内核上运行32位的用户空间。在这种情况下，你可以通过TARGET_KERNEL_ARCH来指定内核架构:
```
$ . build/envsetup.sh; lunch android_x86-userdebug
$ m -j8 iso_img TARGET_KERNEL_ARCH=x86_64
```

这会构建一个32位的用户空间镜像，同时搭配64位的内核。

## 强制重新编译内核

Android和内核使用的是完全不同的构建系统。尽管我们将Android和内核组合在一起，但它们的集成效果并不理想。一个例子是，如果你修改了内核树，Android构建系统不会察觉到这个变化。这就意味着，如果你重新构建镜像，内核不会自动重新构建。

存在几种方法可以克服那个问题。其中一种方法是像这样触碰（或修改）defconfig（默认配置文件）:
```
$ touch kernel/arch/x86/configs/android-x86_*
```

或者删除out/目录中的内核配置:
```
$ rm $OUT/obj/kernel/.config
```

或者移除内核镜像:
```
$ rm $OUT/obj/kernel/arch/x86/boot/bzImage
```

然后像平常一样重新构建镜像。

## 替换内核

因为我们构建的内核支持模块，所以要替换已安装系统的内核，还必须同时替换相应的模块。要做到这一点，需要将内核镜像复制到android-x86安装目录，并将模块复制到/system/lib/modules目录。你可以通过启动到调试模式来完成这一操作，像这样:
```
Detecting Android-x86... found at /dev/sda1

Type 'exit' to continue booting...

Running MirBSD Korn Shell...
$ exit

Use Alt-F1/F2/F3 to switch between virtual consoles
Type 'exit' to enter Android...

Running MirBSD Korn Shell...
$ mount /dev/sdb1 /hd
$ cp /hd/kernel /src
$ rm -rf /system/lib/modules/*
$ cp -a /hd/modules/* /system/lib/modules
$ sync; umount /hd; reboot -f
```

这个例子假设新的内核镜像及其对应的模块位于USB磁盘的/dev/sdb1分区上，并且/system分区是以读写模式安装的。

## 构建一个定制化的内核

假设你已经有了一个适用于你硬件的可行的内核配置文件，那么让构建系统使用你的配置文件来构建iso文件是很简单的。你只需要将你的配置文件放到kernel/arch/x86/configs/目录下，然后运行相关命令（假设你的配置文件名称是my_defconfig）。
```
$ make iso_img TARGET_PRODUCT=android_x86 TARGET_KERNEL_CONFIG=my_defconfig
```

请注意，你不能使用普通Linux发行版（例如Ubuntu）的内核配置文件来为Android-x86构建内核。你需要启用Android内核特性才能使其正常工作。你可以查看android/configs/android-base.cfg文件，里面列出了内核支持Android系统所需的一系列配置选项。不过，对于Android-x86，需要移除其中特定于ARM架构的选项，例如PMEM。

## 自定义内核配置

永远不建议直接编辑内核配置文件，因为这样可能会产生错误的配置（例如依赖关系未满足等）。正确修改内核配置的方法是（在android - x86树的顶部）:
```
$ . build/envsetup.sh; lunch android_x86_64-userdebug
$ make -C kernel O=$OUT/obj/kernel ARCH=x86 menuconfig
```

如果你遇到了“Unknown option: -C”这样的错误信息，那么你应该将“make”命令改为“/usr/bin/make”。这是因为从Android 8开始，其构建系统用自己的make函数（用于调用soong规则）替换了系统的默认make命令，而这个自定义的make函数不识别“-C”选项。为了克服这个问题，只需使用系统的make命令即可。

生成的配置文件是$OUT/obj/kernel/.config。将它复制到你想要它所在的位置。

不要直接在kernel/目录下执行make menuconfig命令。如果你这样做了，可能会破坏构建规则。在这种情况下，可以尝试以下方法来恢复（在android-x86树的顶部）:
```
$ make -C kernel distclean
$ rm -rf $OUT/obj/kernel
```

## 使用预构建的内核

如果你有一个适用于你的硬件的可用的预编译内核二进制文件，那么你可以用它来生成iso文件:
```
$ make iso_img TARGET_PRODUCT=android_x86 TARGET_PREBUILT_KERNEL=<path to the prebuilt kernel>
```

## 为ARM编译内核（已弃用）

内核构建系统也可以用来为ARM架构编译内核。例如，要为arm模拟器中的goldfish CPU编译2.6.29版本的内核，可以运行（后面会跟具体的命令）:
```
$ cd kernel
$ git checkout x86/android-goldfish-2.6.29
$ cd ..
$ make kernel TARGET_KERNEL_CONFIG=goldfish_defconfig TARGET_NO_KERNEL=
```

编译出的内核二进制文件将会被生成在“out/target/product/generic/kernel”这个目录路径下。在编译ARM架构内核的过程中，将TARGET_NO_KERNEL设置为空是非常重要的。如果TARGET_NO_KERNEL没有被设置为空，那么内核的编译步骤将会被跳过，也就是说内核不会被编译生成。这是因为TARGET_NO_KERNEL这个变量在Android-x86的构建系统中是用来控制是否编译内核的标志，当它为空时，表示需要进行内核的编译过程；如果不为空，则会跳过内核编译，直接使用预编译的内核或者其他处理方式。