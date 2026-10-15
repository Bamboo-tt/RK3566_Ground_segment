# 04 设备树与 overlay：把硬件连接交给驱动

```text
构建与组合
SoC/BSP 控制器、时钟和引脚定义
  └─ CM3 核心板 DTSI ─ IO 板 DTS ─ 编译 ─ 基础 DTB ─────┐
AIRCAS 产品 overlay DTS ────────── 编译 ─ 产品 DTBO ────┤
                                                       └─ 解析引用并叠加
                                                          └─ 最终设备树

本章贯穿的触摸资源关系
最终树中的 I2C1 ─ gt9xx@14 ─ 地址与 compatible ─ I2C 核心
                     ├─ touch-gpio / reset-gpio ─ GPIO 与 IRQ
                     ├─ pinctrl-0 ─ 引脚复用及上下拉
                     └─ tp-supply ─ vcc_lcd_tp ─ 供电提供者
                              各项资源交给同一个 GT9XX probe
```

核心依据：[CM3 DTSI](../../src/rk3566-radxa-cm3.dtsi)、[IO 板 DTS](../../src/rk3566-radxa-cm3-io.dts)、[AIRCAS overlay](../../设备树包/aircas-compact-GCS_overlay.dts)；属性消费者采用[云奥 V2.2 的 `goodix_ts_probe()`](<../../云奥-270nit-6inch/mipi_touch (1)/mipi_touch/gt9xx.c>)，另与[阿美林 V2.8.7 的 `gtp_parse_dt()`](../../阿美林-1000nit-7inch/GT9XX通用调试包/gt9xx.c)比较。

上一章已把连接器、GPIO、总线和电源对应起来。本章把这些连接组织成驱动能够查询的资源：先弄清基础树提供了什么，再看产品 overlay 改了什么，最后沿同一个 GT9XX 节点追到 C 代码。驱动的初始化与回收由[下一章](05_驱动匹配与初始化.md)展开。

## 4.1 基础树保存共性，产品 overlay 补入具体外设

### 4.1.1 include 形成一份完整的基础描述

`rk3566-radxa-cm3-io.dts` 包含 `rk3566-radxa-cm3.dtsi`；后者又包含 `rk3566.dtsi`、`rk3568-linux.dtsi` 及 GPIO、PWM、pinctrl 等绑定头文件。SoC 层描述控制器寄存器、时钟、中断和可选引脚；CM3 层加入核心板电源、RK817、eMMC、无线等；IO 板层再选择 HDMI、SD 卡、USB、风扇等连接。

这几个层次最终编译成一份 DTB。`.dtsi` 的职责是复用源码片段，内核读取的是编译后的节点和属性，并不在启动时逐个打开这些文本文件。本地保存的是板级 DTS/DTSI，完整构建还需要 include 指向的 SoC/BSP 文件及相应头文件。

同一节点可以在不同层补充属性。核心板文件先将 `pcie2x1` 设为 `disabled`，IO 板文件再通过 `&pcie2x1` 改为 `okay` 并补入复位、供电和 pinctrl。编译器处理的是同一节点的最终属性集合，不会因此生成两套 PCIe 控制器。对同一属性的后续定义替换前面的值，未重定义的属性继续保留。

### 4.1.2 overlay 修改基础描述，不重新声明整个 SoC

产品文件以 `/plugin/;` 表明它是叠加描述。它通过 `&i2c1`、`&dsi0`、`&pwm5` 修改基础树已有节点，又通过 `&{/}` 向根节点加入电源、背光和电池对象。编译结果 DTBO 保存这些变更和待解析引用，必须与能够提供相应目标的基础 DTB 配合。

采用 overlay 可以把 CM3 的共性配置与屏幕、电池等产品配置分开。代价是基础树仍参与最终结果：更换基础 DTB 后，控制器状态、引脚选择和已有消费者也可能变化。产品文件没有写 `sdmmc0`，不意味着最终树里没有 SD 卡控制器。

当前 `src/` 和 `设备树包/` 中两组同名 DTS/DTSI 内容相同；[带 `_v1` 的文件](../../设备树包/rk3566-radxa-cm3-io_v1.dts)则以裸 regulator 节点开始，缺少独立入口结构，应作为历史片段阅读。正式组合应明确采用哪份完整基础描述和哪份产品 overlay。

## 4.2 标签、路径与 phandle 让描述能够互相引用

考虑产品 overlay 中这一段源码摘录；中文注释说明各字段在本例中的角色，省略其他设备节点：

```dts
&i2c1 {                              /* 修改基础树已有的 I2C1 控制器 */
    status = "okay";                 /* 允许该控制器参与设备初始化 */
    pinctrl-0 = <&i2c1_xfer>;         /* 选择 SDA/SCL 对应的引脚组 */
    #address-cells = <1>;             /* 子节点地址占一个 32 位单元 */
    #size-cells = <0>;                /* I2C 地址后没有内存区域长度 */

    gt9xx: gt9xx@14 {
        status = "okay";
        compatible = "goodix,gt9xx"; /* 提供给驱动匹配表的身份字符串 */
        reg = <0x14>;                /* 7 位 I2C 器件地址 */
        pinctrl-names = "default";
        pinctrl-0 = <&gt9xx_gpio>;    /* INT/RST 的复用及上下拉配置 */
        touch-gpio = <&gpio0 RK_PC5 IRQ_TYPE_LEVEL_HIGH>;
        reset-gpio = <&gpio0 RK_PC6 GPIO_ACTIVE_HIGH>;
        max-x = <1280>;              /* 是否影响上报范围，要看消费者 */
        max-y = <800>;
        tp-size = <9112>;            /* 厂商驱动的产品配置选择值 */
        tp-supply = <&vcc_lcd_tp>;    /* 指向本 overlay 新建的电源 */
    };
};
```

`gt9xx:` 是源码标签；`gt9xx@14` 是节点名，其中 `@14` 表示单位地址；真正交给 I2C 核心使用的地址来自 `reg`。同样是 `reg`，I2C 子节点用它表达器件地址，SoC 控制器节点则可能用它表达寄存器物理地址和长度，具体布局由父节点的地址与长度单元数决定。

`&i2c1` 按标签找到节点，`&{/}` 按绝对路径找到根节点。`<&vcc_lcd_tp>` 等引用在二进制树中通过 phandle 连接到目标；phandle 是树内部的节点标识，不是内存指针，也不是 Linux GPIO 编号。`&gpio0` 后面的引脚和标志由 GPIO 提供者的单元格式解释，`&pwm5` 后的通道、周期和极性则由 PWM 绑定解释。

这种引用还要区分用途：`pinctrl-0` 描述引脚状态，`tp-supply` 描述供电依赖，显示节点的 `remote-endpoint` 描述数据端口连接。三者都引用节点，但没有同一种物理含义。将 DSI 两端连起来不会给面板供电，写出 GPIO 资源也不会自动选择正确的复用功能。

## 4.3 沿 GT9XX 完成基础定义、叠加与属性消费

### 4.3.1 基础 I2C1 提供控制器，overlay 加入产品连接

基础 I2C1 的控制器寄存器、时钟、中断及 `i2c1_xfer` 引脚组来自 SoC/BSP 描述。[内核包](../../debs/linux-image-6.1.84-12-rk2410-nocsf_6.1.84-12_arm64.deb)成员 `usr/lib/linux-image-6.1.84-12-rk2410-nocsf/rockchip/rk3566-radxa-cm3-io.dtb` 中，`__symbols__.i2c1` 指向 `/i2c@fe5a0000`，`i2c1_xfer` 指向 `/pinctrl/i2c1/i2c1-xfer`。

该控制器的 `compatible` 为 `rockchip,rk3568-i2c`、`rockchip,rk3399-i2c`，`reg = <0 0xfe5a0000 0 0x1000>` 表示从 `0xfe5a0000` 开始、长度 `0x1000` 字节的控制器寄存器区域；基础状态是 `disabled`，尚无产品触摸节点。产品 overlay 将这一节点改为 `okay`，选择 `i2c1_xfer` 并加入 `gt9xx@14`。控制器的寄存器地址与子设备的 `0x14` 属于两种地址空间。

最终 I2C1 同时保留基础控制器资源和产品增加的子节点。触摸依赖还继续伸向 overlay 中新建的两个对象：

- `gt9xx_gpio` 位于 `&pinctrl` 下，把 GPIO0_C5 配为无上下拉的 GPIO、GPIO0_C6 配为上拉 GPIO；触摸节点通过 `pinctrl-0` 选择这一组状态。
- `vcc_lcd_tp` 位于根节点下，以 `regulator-fixed` 表示供电开关，通过 GPIO3_D0 控制触摸电源；其 `pinctrl-0` 又引用 `vcc_tp_pin`，触摸消费者通过 `tp-supply` 使用它。

只复制 `gt9xx@14` 还不够：父控制器必须可用，引用的目标必须存在，供电与引脚提供者也必须建立。节点写在文件前面还是后面不决定初始化顺序；真正影响资源取得的是引用关系和驱动的错误返回。

### 4.3.2 驱动按约定读取，属性名称本身不会执行动作

属性名、数据格式、单位及用途共同构成驱动与设备树之间的绑定约定。以下是云奥 V2.2 `goodix_ts_probe()` 中的源码摘录，省略资源操作及其错误路径，只保留读取关系；完整执行顺序见第 05 章。

```c
/* np 是这个 I2C client 对应的设备树节点。 */
struct device_node *np = client->dev.of_node;

/* 名称 tp 对应 tp-supply，由 regulator 框架解析其 phandle。 */
ts->tp_regulator = devm_regulator_get(&client->dev, "tp");

/* 本分支把 touch-gpio 的最后一格存成 IRQ 申请标志。 */
ts->irq_pin = of_get_named_gpio_flags(np, "touch-gpio", 0,
                    (enum of_gpio_flags *)(&ts->irq_flags));
ts->rst_pin = of_get_named_gpio_flags(np, "reset-gpio", 0,
                                     &rst_flags);

if (of_property_read_u32(np, "max-x", &val)) {
    dev_err(&client->dev, "no max-x defined\n");
    return -EINVAL;                  /* 缺少属性会结束本次 probe */
}
/* ts->abs_x_max = val; */          /* 原源码已注释，读到值却未赋给范围 */
```

这里有两类消费。`tp-supply` 经框架转换为可操作的 regulator 句柄；`max-x` 只是一次整数读取。后者通过了存在性检查，却没有写入坐标上限；最终范围还会由触摸配置或芯片现有配置读取，再在注册 input 时按需要交换轴。

`tp-size = <9112>` 同样要追到条件分支。本地 V2.2 只处理 89、101、911、970、910，没有 9112 分支。在初次探测且未指定 `configfile-num` 时，原始全局默认值会把组 0 选为 `gtp_dat_8_9`；若芯片配置版本进入固定配置分支，则改读芯片现有范围。写入 9112 并没有为这份 C 文件新增一种配置。

IRQ 语义尤其容易被名字误导：`touch-gpio` 使用 `IRQ_TYPE_LEVEL_HIGH`，旧代码将取得的 flags 直接与 `IRQF_ONESHOT` 合并后申请 IRQ，表达高电平触发。`reset-gpio` 末尾的 `GPIO_ACTIVE_HIGH` 描述 GPIO 有效极性，而这份旧代码复位时使用整数 GPIO API 直接输出物理高低。触发方式与有效极性不能交换解释；触摸 CFG 内的中断设置还必须与实际申请方式配套。

### 4.3.3 换驱动时，匹配字符串之外的约定也要一起换

阿美林 V2.8.7 也匹配 `goodix,gt9xx`，但属性解析已经换了一套接口：

| 要表达的内容 | 产品 overlay / 云奥 V2.2 | 阿美林 V2.8.7 |
| --- | --- | --- |
| INT GPIO、触发方式 | `touch-gpio`，flags 送入 IRQ 申请 | `irq-gpios` 与独立的 `irq-flags` |
| 复位 GPIO | `reset-gpio` | `reset-gpios` |
| 电源消费者名称 | `tp`，对应 `tp-supply` | `vdd_ana`、`vcc_i2c`，对应各自 supply |
| input 坐标范围 | `max-x/max-y` 被读入，直接赋值已注释 | `touchscreen-size-x/y`，缺失使用 720×1080 默认值 |
| 配置和坐标变换 | `tp-size`、头文件配置组及全局变换标志 | `goodix,cfg-groupN`、`goodix,swap-x2y` 等 |
| 多点上报协议 | 头文件宏选择 Type A/B | `goodix,type-a-report` 选择，默认 Type B |

V2.8.7 的 [DTS 说明](../../阿美林-1000nit-7inch/GT9XX通用调试包/dtsi/gt9xx_dts.txt)文字写过 `irq-gpio/rst-gpio`，示例与实际解析函数使用的却是 `irq-gpios/reset-gpios`。适配时应沿 `gtp_parse_dt()` 确定名称；相同 compatible 会把设备交给候选驱动，后续资源读取仍取决于该分支的约定。

## 4.4 编译、符号解析、合并和驱动读取分别发生什么

源码可以直接写标签，二进制文件需要保存解析标签所需的信息。按本项目这种标签引用式 overlay，过程分为四步：

1. **预处理与编译。** 头文件提供宏值，include 合入基础定义；`dtc` 将基础树和插件分别编为 DTB、DTBO。带符号编译保存 `__symbols__` 等信息。
2. **解析目标与引用。** 应用 overlay 时，外部标签通过基础树符号定位；插件内部的引用也要调整，避免 phandle 与基础树冲突。`__fixups__` 与 `__local_fixups__` 记录需要修正的位置。
3. **合并节点属性。** 本例改变基础 I2C1 的状态，加入触摸子节点，并将根节点电源和 pinctrl 组接入树。多个 overlay 改同一属性时，应用次序影响最后的值。
4. **内核使用最终描述。** 控制器驱动建立总线，I2C 核心建立 client，芯片驱动再读取 GPIO、supply 和配置。前面的编译和引用解析没有执行触摸复位或读坐标。

引用处理可对照 [Linux v6.1 overlay 说明](https://www.kernel.org/doc/html/v6.1/devicetree/overlay-notes.html)与[动态引用解析说明](https://www.kernel.org/doc/html/v6.1/devicetree/dynamic-resolution-notes.html)。本项目包内提供 DTBO 构建及安装回调，实际在哪个启动阶段合成、选中了哪些插件，由启动配置决定；选择链集中在[第 10 章](10_部署生效检查.md)。

DKMS 在这里承担 overlay 构建集成。[普通 overlay 包](../../debs/radxa-overlays-dkms_0.2.4_all.deb)中的 `usr/src/radxa-overlays-0.2.4/Makefile` 遍历各厂商 overlay 生成 DTBO，同时构建 `radxa-overlays.ko` 占位模块；该模块只打印装载和退出日志。DTBO 由 `dkms.post_build/post_install` 保存和链接，应用 overlay 的动作不由这个模块完成，GT9XX 驱动也不在该包中生成。普通包的 `arch/arm64/boot/dts/rockchip/overlays/aircas-compact-GCS.dts` 与工作区产品文件按行一致；[PWM 包](../../debs/radxa-overlays-dkms_pwm.deb)还含独立的 `aircas-compact-GCS-pwm.dts`。两包都使用 0.2.4 版本，选择对象要落实到具体成员；完整路径见[资料索引](13_关键资料阅读索引.md)。

## 4.5 最终树必须让资源用途相容

设备树合并解决的是节点与属性组合，物理引脚仍只有一套。当前 IO DTS 启用 SDMMC0 并选择 `sdmmc0_bus4`，其中的 GPIO1_D5 又被产品 overlay 用作面板电源使能。包内基础 DTB 已禁用 SDMMC0；按当前 IO DTS 重建并叠加相同产品文件，会重新改变这个条件。此时要决定 SD 卡和面板电源分别使用什么连接，不能只调整触摸节点或反复重编 DTBO。

`metadata.exclusive` 中的 `dsi0/vp0/video_phy0/pwm5` 是管理工具的资源提示，未列尽 GPIO，也不负责在内核中锁住所有资源。最终启用的消费者、pinctrl 选择和原理图连线要共同相容。GPIO3_C1、GPIO3_B4 在不同底板图中的用途差异，以及 fixed regulator 对 `enable-active-high` 的兼容解析，见[第 03 章](03_硬件资源与供电.md)。

读完一个节点后，可以沿交接确定停在了哪里：最终树是否包含并启用父总线与子节点；总线是否创建了设备；匹配驱动是否进入 probe；资源与通信是否完成；input 是否注册并持续交付事件。每个问题都对应后续的一段代码，不宜用一个 `status = "okay"` 概括整条运行链。

## 4.6 核心工程问题与深度理解

### 4.6.1 overlay 写了 1280×800，input 范围却没有变化，应先追哪里

先追属性消费者与后续赋值。云奥 V2.2 读取 `max-x/max-y` 后没有执行直接赋值，范围来自 CFG 或芯片固定配置，并可能在 input 注册时交换轴。应先确定实际驱动分支、配置来源和上报范围，再决定修改设备树、CFG 还是坐标变换；重复写入同一属性不会改变读取逻辑。

### 4.6.2 相同 DTBO 换一份基础 DTB 后，为什么可能出现新的引脚冲突

DTBO 只覆盖自己声明的差异，基础树未被改动的消费者会继续存在。本项目的 SDMMC0 就展示了这种条件变化：交付基础 DTB 禁用了它，当前 IO DTS 却将其打开，GPIO1_D5 会与面板电源用途重叠。判断组合时必须看合成后的资源关系。

### 4.6.3 供电节点放到文件最前面，能否解决触摸 probe 的依赖等待

不能。消费者通过 `tp-supply` 引用供电提供者，再由 `devm_regulator_get()` 取得句柄；提供者尚未就绪时，需要返回并保留相应错误，框架才能安排后续探测。文本位置没有建立这种执行同步，下一章将沿真实错误路径解释它。

[上一章：硬件资源与供电](03_硬件资源与供电.md) · [下一章：驱动匹配与初始化](05_驱动匹配与初始化.md) · [返回总览](README.md)
