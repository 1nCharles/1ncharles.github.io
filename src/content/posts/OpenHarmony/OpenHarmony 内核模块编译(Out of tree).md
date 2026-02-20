---
title: OpenHarmony 内核模块编译(Out of tree)
published: 2026-02-20
category: OpenHarmony
description: " "
---
# OpenHarmony 内核模块编译(Out of tree)

参考文档：[openharmony5.0.0中kernel子系统编译构建流程概览(rk3568)_openharmony5 单独编译内核-CSDN博客](https://blog.csdn.net/gkxg001/article/details/148406175)

本文基于OpenHarmony 6.1 Release qemu x86_64编译

因为OpenHarmony的Kernel源码和rom源码是在同一个项目里面，而且官方推荐的使用build.sh的编译方法，需要sync整个OpenHarmony的源码（否则根本过不掉检查）

所以才有了这篇文章，只需要拉取必须文件即可编译内核

# 内核编译

## 同步源码

由于我们只需要编译内核，所以只需要同步我们需要的源码即可

​`chipsets/qemu.xml device/qemu vendor/ohemu` 根据实际情况来写即可

```shell
repo init -u https://gitee.com/openharmony/manifest.git -b OpenHarmony-6.0-Release -m chipsets/qemu.xml
```

```python
repo sync -c build kernel/linux/common_modules kernel/linux/build kernel/linux/linux-5.10 kernel/linux/patches kernel/linux/config device/qemu vendor/ohemu drivers/hdf_core third_party/FreeBSD
```

然后下载prebuilds，build目录下附带了下载脚本（准备NodeJs 和 python 环境）

```shell
build/prebuilts_download.py
```

## 确定参数

官方build.sh中，编译内核时的传参如下

```c
 action("build_kernel") {//定义一个构建任务,action参数用于定义构建过程中的具体操作
    script = "build_kernel.sh"//指定了用于构建内核的脚本文件为`build_kernel.sh`
    sources = [ kernel_source_dir ]//指定内核源码目录//kernel/linux/$linux_kernel_version

    deps = [ ":check_build" ]//在执行构建内核任务之前，必须先完成`:check_build`任务,check_build为上文定义的check_build.sh脚本
    product_path = "vendor/$product_company/$product_name"//产品自定义相关的目录
    build_type = "standard"//构建类型为标准构建
    outputs = [ "$root_build_dir/packages/phone/images/$kernel_image" ]//构建的最终输出文件的位置和名称
    args = [//列出了传递给构建脚本的参数
      rebase_path(kernel_build_script_dir, root_build_dir),//1.重新计算内核构建脚本目录相对于构建目录根路径的路径,kernel_build_script_dir = "//kernel/linux/build";，定位到out输出的目录
      rebase_path("$root_out_dir/../KERNEL_OBJ"),//2.重新计算内核对象文件目录相对于某个输出目录的路径
      rebase_path("$root_build_dir/packages/phone/images"),//3.重新计算内核镜像文件放置目录相对于构建目录根路径的路径
      build_type,//4.构建类型，这里已经定义为"standard"
      target_cpu,//5.目标CPU架构，构建内核时需要指定针对哪种CPU架构进行构建
      product_path,//6.产品路径，已在前面定义
      device_name,//7.设备名称，表示正在构建内核的具体设备型号
      linux_kernel_version,//8.Linux内核版本号，表示正在构建的内核的具体版本
    ]
  }

```

给个传参的示例

```shell
export OHOS_BUILD_HOME="/path/to/OpenHarmonySource"
./build_kernel.sh ${OHOS_BUILD_HOME}/kernel/linux/build  ${OHOS_BUILD_HOME}/out/ ${OHOS_BUILD_HOME}/out/images standard x86 vendor/ohemu/qemu_x86_64_linux_min qemu-x86_64-linux linux-5.10
```

这里需要注意的是 `device_name`​ 要与 `defconfig` 的文件名中的相同

后面会直接使用`device_name`​拼接出文件名来寻找 `defconfig`

一般都能在`kernel/linux/config/` 下找到

## 开始编译

一定要在 `kernel/linux/build`​ 这个目录下执行`build_kernel.sh`

![image](assets/image-20260220225739-tig07m3.png)

否则 `kernel.mk`​ 中的 `OHOS_BUILD_HOME` 将会是错误的路径

![image](assets/image-20260220231518-q4bckuz.png)

编译完成后的输出目录中的结构为

```shell
.
└── kernel
    ├── OBJ
    │   └── linux-5.10
    │       ├── arch
    │       ├── block
    │       ├── certs
    │       ├── crypto
    │       ├── drivers
    │       ├── fs
    │       ├── include
    │       ├── init
    │       ├── io_uring
    │       ├── ipc
    │       ├── kernel
    │       ├── lib
    │       ├── Makefile
    │       ├── mm
    │       ├── net
    │       ├── scripts
    │       ├── security
    │       ├── sound
    │       ├── source -> /home/ubun/Desktop/op/out/kernel/src_tmp/linux-5.10
    │       ├── tools
    │       ├── usr
    │       └── virt
    └── src_tmp
        └── linux-5.10
            ├── arch
            ├── base_defconfig
            ├── block
            ├── bounds_checking_function
            ├── certs
            ├── COPYING
            ├── CREDITS
            ├── crypto
            ├── Documentation
            ├── drivers
            ├── fs
            ├── hispark_taurus
            ├── imx8mm
            ├── include
            ├── init
            ├── io_uring
            ├── ipc
            ├── Kbuild
            ├── Kconfig
            ├── kernel
            ├── lib
            ├── LICENSES
            ├── MAINTAINERS
            ├── Makefile
            ├── mm
            ├── net
            ├── OAT.xml
            ├── qemu
            ├── README
            ├── README.OpenSource
            ├── rk3568
            ├── samples
            ├── scripts
            ├── security
            ├── sound
            ├── tools
            ├── type
            ├── unionpi_tiger
            ├── usr
            ├── virt
            └── yangfan

```

src tmp 为 拷贝的kernel源代码

OBJ才是编译产物

# 内核模块编译

有了OBJ下的编译产物我们就可以实现脱离源码树编译内核模块了

可以直接借鉴[DDK](https://github.com/Ylarod/ddk-module-template)的template

envset.sh

```shell
export KDIR=~/Desktop/op/out/kernel/OBJ/linux-5.10
export CLANG_PATH=~/Desktop/op/prebuilts/clang/ohos/linux-x86_64/llvm/bin

export PATH=$CLANG_PATH:$PATH
export CROSS_COMPILE=x86_64-unknown-linux-ohos-
export ARCH=x86_64
export LLVM=1
export LLVM_IAS=1
```

Makefile

```makefile
MODULE_NAME := Shami
$(MODULE_NAME)-objs := ko_sample.o
obj-m := $(MODULE_NAME).o

ccflags-y += -Wno-declaration-after-statement
ccflags-y += -Wno-unused-variable
ccflags-y += -Wno-int-conversion
ccflags-y += -Wno-unused-result
ccflags-y += -Wno-unused-function
ccflags-y += -Wno-builtin-macro-redefined -U__FILE__ -D__FILE__='""'

KDIR := $(KDIR)
MDIR := $(realpath $(dir $(abspath $(lastword $(MAKEFILE_LIST)))))

$(info -- KDIR: $(KDIR))
$(info -- MDIR: $(MDIR))

all:
	make -C $(KDIR) M=$(MDIR) modules
compdb:
	python3 $(MDIR)/.vscode/generate_compdb.py -O $(KDIR) $(MDIR)
clean:
	make -C $(KDIR) M=$(MDIR) clean
```

经过测试可以正常在DevEco的模拟器上运行（root且关闭SELinux条件下）

![image](assets/image-20260220231238-iptj7lc.png)

‍

打包了一份OpenHarmony 6.0 Release qemu的编译产物放在github上 需要的师傅自取

[Release 6.0 Release · 1nCharles/OpenHarmonyKernelCompileOutput](https://github.com/1nCharles/OpenHarmonyKernelCompileOutput/releases/tag/6.0Release)

‍

‍
