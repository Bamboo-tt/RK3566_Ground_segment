# 02 BSP 构建与交付

```text
内核源码 + 补丁 + 配置 + 基础 DTS
    -> Kconfig/Kbuild、编译器、dtc -> 内核 / 模块 / DTB
    -> Debian 包装 -> linux-image、headers、libc-dev --------+
                                                          |
产品 overlay + Makefile -> overlay DKMS 源码包             |
    -> 按目标 kernel release 构建 -> DTBO -> 安装链接 -------+-> 系统文件
                                                          |   + 启动选择
发行版包 + 桌面 + 程序配置 -> RootFS -----------------------+
```

核心入口是[历史构建记录](<../../RADXA CM3地面站编译.docx>)、[内核包](../../debs/linux-image-6.1.84-12-rk2410-nocsf_6.1.84-12_arm64.deb)及[普通 overlay 包](../../debs/radxa-overlays-dkms_0.2.4_all.deb)。图描述文件生产与组合：内核、产品插件和 RootFS 分别准备，在镜像制作或已有目标系统中汇合。

## 2.1 先判断一次修改改变什么

BSP（Board Support Package）是支持具体板卡的软件、配置和构建材料的组合。在这里，它包含内核与驱动、硬件描述、编译配置和交付包装。触摸 RST 换引脚通常改变设备树；触点字节解析错误要改变驱动；桌面要新增电量展示则改变用户态程序与配置。这些变化最后可以进入同一镜像，输入和构建工具却不同。

| 改动 | 输入与工具 | 产物 | 最终消费者 |
| --- | --- | --- | --- |
| 器件操作、协议解析 | C 源码、配置、Kbuild、编译器 | 内核映像或 `.ko` | 处理设备的内核驱动 |
| 接线、地址、供电、产品组合 | DTS/DTSI、overlay、预处理器与 dtc | DTB、DTBO | 树合并程序和内核属性解析 |
| 桌面、服务、应用与库 | 软件包及系统配置、镜像工具 | RootFS 内容 | 用户空间进程 |

改 DTBO 之所以能改变某项行为，是因为驱动提供了相应属性入口。属性没人读取时，二进制树即使包含新值，执行路径仍保持原样。第 04、05 章会把这件事追到 GT9XX 的读取函数。

## 2.2 源码和配置怎样成为内核及模块

### 2.2.1 Kconfig 选择能力，Kbuild 组合目标

Kconfig 声明功能选项与依赖，最终 `.config` 保存一次构建的选择，Kbuild 根据它组织编译和链接。三态选项中的 `y` 进入内建对象，`m` 生成可加载模块。

下面摘录[供应商 V2.8.7 Makefile](../../阿美林-1000nit-7inch/GT9XX通用调试包/Makefile)，添加中文注释。它用于说明这套源码的构建入口：

```makefile
# CONFIG 为 y/m 时，分别把复合目标纳入内建/模块集合。
obj-$(CONFIG_TOUCHSCREEN_GT9XX) += gt9xx_core.o
# 这三个对象组成 gt9xx_core；这里的 -y 表示组成关系。
gt9xx_core-y := gt9xx.o gt9xx_update.o goodix_tool.o
```

于是同一段 C 代码可以随内核进入注册流程，也可以先装载模块再注册；两种入口都要继续设备匹配与 probe。[Kbuild 文档](https://www.kernel.org/doc/html/v6.1/kbuild/modules.html)解释外部模块对配套配置及构建信息的依赖。

当前 `linux-image` 成员 `boot/config-6.1.84-12-rk2410-nocsf` 中，GT9XX、evdev、CW2015、gpio-charger、PWM 背光、Rockchip PWM 和 simple panel 为 `y`；pwm-fan、Panfrost 和另一项 `CONFIG_TOUCHSCREEN_GOODIX` 为 `m`。因此 GT9XX 不出现在 `lsmod` 中是正常现象，而风扇路径包含模块装载。名称相近的 `GOODIX` 与 `GT9XX` 是不同配置入口。

### 2.2.2 kernel release 把模块与内核配套

包使用的 kernel release 为 `6.1.84-12-rk2410-nocsf`，模块位于 `/lib/modules/6.1.84-12-rk2410-nocsf/`。这个标识连接了内核、模块目录和外部构建目标；模块还依赖目标架构、导出符号及相应配置。只把文件放进名称相同的目录，无法修正接口不兼容。

本地云奥目录的历史 `.o` 是 ELF32/ARM，而当前包为 ARM64。这些对象可帮助辨认旧构建产物，重新构建时应由相应工具链从源码生成目标对象。

### 2.2.3 主机、容器和目标架构分别承担什么

构建主机执行工具，交叉编译器生成 ARM64 指令，容器提供依赖环境。如果构建流程还需要执行 ARM64 程序，才会涉及 ARM64 执行环境或 QEMU/binfmt。历史记录中的 Ubuntu、devcontainer、交叉工具和 QEMU 分属这些职责；Docker 本身不决定所有编译都在模拟 ARM64 CPU 中运行。

## 2.3 硬件描述怎样成为另一类构建产物

DTS 是板级入口，DTSI 供它引入 SoC 或核心板公共描述。预处理展开 include 与宏，dtc 把树转换为 DTB；overlay 则以插件形式生成 DTBO，保留对基础树目标的引用关系。二者都是描述数据，`reg` 和 GPIO 属性不会在这一步变成操作器件的机器指令。

基础树给出控制器、资源及公共板级连接，产品 overlay 启用 DSI/I2C/PWM 并增加 panel、GT9XX、CW2015 等消费者。第 04 章解释标签、符号与合并；安装后的启动组件还要选择这份组合，内核才能读取最终属性。

历史记录的 CM3 移植包含两个动作：将 `rk3566-radxa-cm3-io.dts` 和 `rk3566-radxa-cm3.dtsi` 放入内核设备树目录，并将对应 DTB 加入该目录 Makefile 的构建目标。只复制源码而没有纳入目标，打包阶段就可能没有所需 DTB。

## 2.4 从历史工程定位当前交付版本

[构建记录](<../../RADXA CM3地面站编译.docx>)先使用 Linux 5.10/Debian 11 的 `bsp/rbuild`，后来转到 Linux 6.1/Debian 12，把 `linux-rk2410-nocsf`、`radxa-overlays` 和 RSDK 分开组织。其“CM3-IO移植”一节明确引用 Radxa 内核提交 `3bbac32c942f4d98fc87bde3852f185dce56ff56`，随后记录 CM3 DTS 移植及内核修改。[公开打包仓库的 6.1.84-12 标签](https://github.com/radxa-pkg/linux-rk2410-nocsf/tree/6.1.84-12)也将 `src` 指向这一提交。

这个关系提供了后文查阅 panel、CW2015、thermal 实现的固定入口。本地项目修改须再对照当前文件和包内容：例如包里有 `gtp_dat_gt9112` 符号，本地 V2.2 的配置选择却没有 9112；当前基础 DTS 与包内 DTB 在 SDMMC0、HDMI、PCIe 和 CH482D 节点上也有差异。公开基线、移植记录和实际交付物应分别保存。

内核工程的 `make deb` 经 Debian 构建规则组织打包，产生 image、headers 和 UAPI 包；overlay 工程的同名目标产生插件源码包。RSDK 则在系统构建阶段组合指定包和用户空间。历史记录中的命令如下，工作目录是当时的 RSDK 工程：

```sh
# 板型、发行版、桌面与 ./debs 中的包，在系统镜像准备阶段汇合。
rsdk build radxa-cm3-io bookworm xfce --no-cache --debs ./debs
```

RootFS 包含程序、库、配置和服务，内核为它们提供执行和硬件访问能力；镜像流程再组织启动文件和存储布局。用户空间内容可以采用已经构建好的内核包，无需每次从头编译内核。恢复历史构建时还需要配套工具版本、子模块、补丁及最终配置，命令所在工程也必须对应。

## 2.5 Debian 包把文件交给目标系统的 hooks

### 2.5.1 三种内核包的消费者不同

`.deb` 包含 control 档案与 data 档案：前者保存身份、依赖及维护脚本，后者保存部署文件。主要成员统一见[第 13 章](13_关键资料阅读索引.md)。

| 包 | 主要内容 | 用途 |
| --- | --- | --- |
| `linux-image-*` | 内核、模块、DTB、config、System.map | 目标系统运行及设备支持 |
| `linux-headers-*` | 配套头文件、配置与构建材料 | 外部模块或相应内核目标构建 |
| `linux-libc-dev` | Linux 用户态 UAPI 头文件 | 用户程序编译所需的公开接口 |

headers 与 libc-dev 服务不同消费者。应用需要知道系统调用或数据结构的公开接口；内核模块还需要相应内核的内部构建条件，因此 UAPI 包不能代替 headers。

### 2.5.2 postinst 把后续动作交给已安装系统

下面是当前内核包 control 档案中 `postinst` 的关键摘录，省略参数导出及脚本尾部，添加中文注释：

```sh
# 向目标系统 hooks 表达需要 initramfs 的意图。
export INITRD=Yes
# hooks 接收内核发行号和映像路径，随后按系统配置处理。
test -d /etc/kernel/postinst.d && \
    run-parts --arg="6.1.84-12-rk2410-nocsf" \
        --arg="/boot/vmlinuz-6.1.84-12-rk2410-nocsf" /etc/kernel/postinst.d
```

包管理器在配置阶段执行 `postinst`，后者调用目标系统已有的 hooks。更新 initramfs、维护启动项等具体动作由这些 hooks 实现；当前包携带的是调用入口，实际系统 hooks 和启动配置要在目标 RootFS 中寻找。

initramfs 是启动早期的内存文件系统，可先建立访问正式根文件系统的条件，再切换到正式 RootFS。例如某个根存储驱动若作为模块交付，就可能需要在早期环境中提供它。是否需要某个模块进入 initramfs，由内建/模块选择和根存储依赖决定。[Linux initramfs 说明](https://www.kernel.org/doc/html/v6.1/filesystems/ramfs-rootfs-initramfs.html)

## 2.6 DKMS 在本项目交付的是设备树插件

### 2.6.1 从源码目录走到内核对应的 DTBO 目录

两个 overlay 包均使用 `Package: radxa-overlays-dkms`、`Version: 0.2.4`、`Architecture: all`；data 档案公共目录为 `usr/src/radxa-overlays-0.2.4/`。`all` 是包架构声明，各个插件仍有自己的硬件适用条件。

```text
包 control/postinst -> DKMS common.postinst
    -> dkms.conf 声明模块身份与 POST_BUILD、POST_INSTALL 等
    -> Makefile 选择 vendor 与 dtbo 目标
    -> dkms.post_build 保存生成文件
    -> dkms.post_install 建立内核 vendor/overlays 链接
    -> 若有 rsetup：rebuild_overlays <kernelver>
    -> 目标系统保留的插件选择 -> 下次启动使用的树组合
```

DKMS 按 kernel release 与架构管理外部构建，具体产物由 Makefile 决定。包内顶层 Makefile 同时列出 vendor 子目录和 `obj-m := radxa-overlays.o`，因此会生成 DTBO，并构建一个配合 DKMS 的占位模块。`radxa-overlays.c` 的 init/exit 只打印日志，没有应用 overlay 或操作外设的函数；产品设备树的安装由下面的 hooks 完成。

下面摘录同一包的 `dkms.conf`，省略其他字段，注释说明模块与插件各自的位置：

```sh
# 占位模块满足 DKMS 的模块构建/安装流程。
BUILT_MODULE_NAME[0]="radxa-overlays"
DEST_MODULE_LOCATION[0]="/updates/dkms"
# DTBO 由独立回调保存到 DKMS 产物目录，再链接到内核目录。
POST_BUILD="dkms.post_build $kernelver $arch"
POST_INSTALL="dkms.post_install $kernelver $arch"
```

这份配置没有单独的 `MAKE` 字段，构建入口遵循 DKMS 的默认 Kbuild 流程。装载占位模块不会选中 DTBO，也不会注册 GT9XX 或 panel。

包的 `postinst` 在配置阶段调用 DKMS `common.postinst`。`dkms.conf` 设置 `AUTOINSTALL=yes`，将构建后的保存、安装和移除工作交给对应回调。`dkms.post_build` 从 vendor 目录收集 DTBO 到 DKMS 对应内核/架构的 `module/.../overlays`，使安装阶段有独立于临时构建目录的产物来源。

`dkms.post_install` 将结果链接到 `/usr/lib/linux-image-<kernelver>/<vendor>/overlays`。目标内核没有相应 vendor 目录，或该位置已有真实目录时会跳过相应处理；可以找到 `rsetup` 才调用 `rsetup rebuild_overlays <kernelver>`。因此检查时先区分构建失败、目标内核不符与链接跳过，再检查启动选择。第 10 章给出对应观察点。

### 2.6.2 相同包版本可以保存不同插件

[普通包](../../debs/radxa-overlays-dkms_0.2.4_all.deb)内的 `aircas-compact-GCS.dts` 与仓库产品 overlay 归一化换行后相同。[PWM 包](../../debs/radxa-overlays-dkms_pwm.deb)还包含 `aircas-compact-GCS-pwm.dts` 及对应构建目标。

普通亮度表有 256 项，PWM 变体为 `[0,60…255]` 共 197 项，两者默认索引均为 200。公开配套基线的背光驱动将变体的越界默认索引钳到最大索引 196，具体控制值与波形在第 06 章演算。包名、版本号都没有表达这项差异，因此交付记录同时需要包哈希、插件名和基础树身份。

## 2.7 拆分交付带来的选择与约束

独立 overlay 让多个产品共享基础内核，并各自维护器件组合；代价是基础树目标、属性解析和插件资源之间必须配套。将所有内容写入单一板级 DTS 可减少组合环节，但每种产品都要维护相应基础树。

内建驱动减少装载环节，更新它需要更换内核；模块可在接口匹配时单独更新，却增加装载、卸载和版本配套责任。DTBO 更新影响的是描述，不会替换内建驱动的函数体。

一次可重复交付应把硬件版本、公开基线、本地补丁、最终配置、基础树、产品插件和包内容连起来。当前包内 SDMMC0 被禁用，而本地 IO DTS 启用它，重新构建时就需要重新处理 GPIO1_D5 的屏幕供电关系。版本号相同无法替代这项实际组合判断。

## 2.8 核心工程问题与深度理解

### 2.8.1 更换触摸复位接线和修复触点解析，分别需要更新什么

前者通常更新硬件描述，后者更新驱动代码。新 DTBO 要被选中且属性被驱动读取；驱动若内建，就更新内核，若为模块则更新匹配模块。两者都要在运行设备及接口上检查结果，不能停在文件生成处。

### 2.8.2 包配置完成后，为什么系统可能仍使用旧行为

包配置只是部署过程中的一个阶段。可能运行的还是旧内核，也可能 DTBO 为另一个 kernel release 构建、链接被跳过，或启动项选择了旧插件。沿运行内核、DTBO 实际路径、启动选择和最终树逐项比较，才能确定哪一步保留了旧对象。

### 2.8.3 将所有内容改成内建、写进单一 DTS 是否就消除了配套问题

会减少模块与插件装载环节，但电路、属性解析、内核配置和启动选择仍要一致，而且产品变体会扩大基础树的维护范围。应按产品数量和发布方式选择，而不是用减少文件数代替资源与版本管理。

继续阅读：[第 03 章硬件资源与供电](03_硬件资源与供电.md)。
