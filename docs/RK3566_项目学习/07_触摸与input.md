# 07 触摸与 input：把芯片记录转换成连续输入

```text
云奥 V2.2 的普通触摸运行链
GT9XX 扫描得到触点
  ├─ INT 通知 → IRQ 线程回调 → 禁用该 IRQ ────┐
  └─ 轮询模式 → hrtimer 到期 ─────────────────┤
                                             ↓ queue_work
                              goodix_wq → goodix_ts_work_func
                                             ↓
                    I2C 首读状态及首点 → 多点时续读 → 解码与坐标变换
                                             ↓
                  Type A 默认分支 / 编译选择的 Type B → input_sync
                                             ├─ input / evdev → 上层映射
                                             └─ 清除芯片就绪状态 → 恢复 IRQ

失败分支
读取失败 / 无有效帧 → 不更新 input 中的旧接触状态
有效零触点帧       → 上报释放 → 同步本帧
```

核心依据：[云奥 V2.2 `gt9xx.c`](<../../云奥-270nit-6inch/mipi_touch (1)/mipi_touch/gt9xx.c>) 的 `gtp_request_irq()`、`goodix_ts_work_func()`、I2C 读写及触点上报函数，配合[旧头文件](<../../云奥-270nit-6inch/mipi_touch (1)/mipi_touch/gt9xx.h>)的宏；[阿美林 V2.8.7 `gt9xx.c`](../../阿美林-1000nit-7inch/GT9XX通用调试包/gt9xx.c)的 `gtp_get_points()`、两种报告函数与采集入口。主图限定普通 GT9xx、非 raw/diff 模式，特殊维护分支放在正常链之后。

第 05 章已经建立 input 设备和采集入口。本章沿一个触点从寄存器到 evdev 追踪：谁执行读取，字节怎样组织，坐标怎样转换，以及下一帧怎样表达移动和释放。显示与触摸虽然共用屏幕模组，像素经过 DSI、坐标经过 I2C，显示的方向设置也不会自动改写这条输入链。

## 7.1 通知触发采集，I2C 在线程中取得数据

### 7.1.1 本地旧分支已经使用 IRQ 线程，随后再次排队

GT9XX 的 INT 线只通知主机有事件，不携带 X/Y。I2C 操作可能等待适配器完成传输，错误处理还可能重试和延时，因此读取需要可睡眠的执行上下文。

当前 V2.2 的 `gtp_request_irq()` 使用 `devm_request_threaded_irq()`，硬中断处理函数参数为 NULL，`goodix_ts_irq_handler()` 作为线程函数。`IRQF_ONESHOT` 让 IRQ 在线程处理期间保持屏蔽，默认硬中断入口负责唤醒线程；它不表示只接收一次触摸。[Linux v6.1 IRQ 接口](https://www.kernel.org/doc/html/v6.1/core-api/genericirq.html)

但这个线程仍未读取坐标。下面摘录其完整回调，省略日志：

```c
static irqreturn_t goodix_ts_irq_handler(int irq, void *dev_id)
{
    struct goodix_ts_data *ts = dev_id;

    gtp_irq_disable(ts);                  /* 显式禁用，保持到后续 work 结束 */
    queue_work(goodix_wq, &ts->work);      /* 将同一设备的采集任务排队 */
    return IRQ_HANDLED;                   /* IRQ 线程返回，采集可能尚未执行 */
}
```

ONESHOT 的保护随 IRQ 线程结束，不会继续等待另一个 workqueue。显式禁用把保护区间延长到工作函数处理之后，再由它恢复 IRQ。该机制防止通知在本轮处理期间反复进入，不让触摸芯片停止扫描，也没有为每次扫描建立一份独立软件帧。

`ts->work` 在 probe 中通过 `INIT_WORK()` 关联 `goodix_ts_work_func()`；工作线程调用后，函数用 `container_of(work, struct goodix_ts_data, work)` 找回原来的 `ts`。单线程 `goodix_wq` 串行处理队列内工作，同一个尚未执行的 work 也不会无限重复入队，因而中断次数与输入帧数不要求一一相等。

### 7.1.2 轮询共用采集函数，但通知节奏不同

IRQ 申请失败后，旧分支创建 hrtimer。回调只调用 `queue_work()` 并重新设置到期时间，I2C 仍交给 `goodix_wq`。首次等待 1 s，随后名义间隔为 16 ms；排队、I2C 重试和复位会延后真正的 input 帧。

所以触摸时 IRQ 计数不增长，要先识别驱动是否已经选了轮询。反过来，IRQ 持续增长也只定位到通知入口；寄存器可能尚未 ready、通信可能失败、工作状态也可能禁止报告。有效触点必须继续经过下面的数据链。

## 7.2 一帧由状态字、触点记录和尾部信息组成

### 7.2.1 先分清器件地址、寄存器地址和返回消息数

I2C client 的 `0x14` 选择总线上的器件。一次坐标读取先发送内部地址 `81 4E`，再从该寄存器读取字节。旧 `gtp_i2c_read()` 将这两步组成两条 I2C 消息，完整成功时 `i2c_transfer(..., 2)` 返回 2，表示完成两条消息，和坐标占多少字节无关。

工作函数的 `point_data` 缓冲区前两个字节由软件预填地址。第一次调用长度为 12，所以真正从芯片读取 10 字节：状态 1 字节、第一点 8 字节，再加 1 字节后续内容。

| 缓冲区位置 | 内容 | 随触点数变化的含义 |
| --- | --- | --- |
| `point_data[0..1]` | `81 4E` | 请求地址前缀，不是读取结果 |
| `point_data[2]` | 状态字 | bit 7 为 ready，低 4 位为触点数 |
| `point_data[3..10]` | 第一点的 8 字节记录 | ID、X、Y、接触量及剩余字节 |
| `point_data[11]` | 第一次读取的最后一个数据字节 | 单点时到达尾部按键信息；多点时是第二点 ID |
| `point_data[12..]` | 多点续读填入的部分 | 从第二点剩余字段一直读到帧尾 |

有 N 个触点时，第 i 条记录的起点是 `3 + 8*i`，尾部按键字节位于 `3 + 8*N`。第二点的 ID 已被首读带回，所以续读从寄存器 `0x814E + 10` 开始，将得到的 `8*(N-1)` 个数据字节放到 `point_data[12]` 起。以 N=2 为例，首读 10 字节、续读 8 字节，共获得 18 字节，即状态 1、触点 16、尾部 1。

### 7.2.2 ready 决定是否更新，零触点与零状态有不同结果

普通 GT9xx 路径在首读后先处理 `finger == 0`，然后检查 ready，最后提取触点数并限制为不超过 `GTP_MAX_TOUCH=10`。状态 `0x81` 表示有效单点帧；`0x80` 表示有效零触点帧；`0x00` 则让本次采集直接退出。

有效零触点帧承担“上一帧的接触现在已经结束”的信息。零状态没有提供这次状态更新，因此代码保留旧接触。把无数据当成抬起，会在瞬时通信间隔中制造假释放；一直收不到有效释放帧，又会造成上层持续保留旧接触，这个取舍需要在错误恢复处处理。

下面摘录旧工作函数的续读段。读法和索引保持原样，注释同时指出当前缺少的判断：

```c
if (touch_num > 1) {
    u8 buf[8 * GTP_MAX_TOUCH] = {
        (GTP_READ_COOR_ADDR + 10) >> 8,
        (GTP_READ_COOR_ADDR + 10) & 0xff
    };
    ret = gtp_i2c_read(ts->client, buf, 2 + 8 * (touch_num - 1));
    /* 前 2 字节仍是地址；这里未检查 ret 就复制续读内容。 */
    memcpy(&point_data[12], &buf[2], 8 * (touch_num - 1));
}
```

正常完整读取时，这段代码把首读和续读接成一帧。读取失败时如何影响解析，集中放在 7.5 节；先沿完整帧追到 input。

### 7.2.3 同一条记录怎样得到 ID、X、Y 与接触量

采用一组构造数据演算，不把它写成芯片抓包：

```text
状态：81
记录：01 2C 01 64 00 10 00 00
字段：ID X低 X高 Y低 Y高 W低 W高 剩余字节
```

ID 低四位是 1。小端 X 为 `0x2C + (0x01 << 8) = 300`，Y 为 100，W 为 16。旧 Type A 循环用以下表达式解析同一组字节：

```c
coor_data = &point_data[i * 8 + 3];     /* 跳过地址前缀和状态字，每点 8 字节 */
id      = coor_data[0] & 0x0F;
input_x = coor_data[1] | (coor_data[2] << 8);
input_y = coor_data[3] | (coor_data[4] << 8);
input_w = coor_data[5] | (coor_data[6] << 8);
gtp_touch_down(ts, id, input_x, input_y, input_w);
```

W 被旧代码写入 `ABS_MT_TOUCH_MAJOR` 和 `ABS_MT_WIDTH_MAJOR`，新分支还把相同原始数值写为 pressure。这里是驱动选择的事件语义，没有进行以牛顿为单位的压力标定。

## 7.3 坐标变换之后，用完整生命周期报告输入

### 7.3.1 坐标和上报范围要采用同一个方向

旧 `gtp_touch_down()` 先交换 X/Y，再按开关分别计算 `abs_x_max-x`、`abs_y_max-y`。范围在 input 注册前也会随轴交换而交换，这样变换后的坐标与 input 声明采用同一坐标系。

继续使用原始点 `(300,100)`。假设某个已选配置给出原始范围 800×1280，要求交换轴并镜像 X，那么 input 范围应为 1280×800；点先变为 `(100,300)`，再按源码公式变为 `(1180,300)`。这些是明确给定条件的算例，镜像公式采用驱动的 max 值，不擅自改为 max−1。

实际方向由 `tp-size` 分支或 `goodix,swap-*` 属性决定，范围来源见第 05 章。原始坐标、驱动输出、显示模式与桌面映射是连续但不同的空间：panel 的 `rotation=90` 提供显示方向信息，GT9XX 没有因此自动读取并改变坐标。若驱动已交换轴，桌面又做相同变换，点击就可能二次旋转。

### 7.3.2 旧头文件默认采用 Type A，逐帧提供当前接触集合

旧 `gt9xx.h` 仅在 `CONFIG_CHROME_PLATFORMS` 定义时将 `GTP_ICS_SLOT_REPORT` 设为 1，否则为 0。[交付内核包](../../debs/linux-image-6.1.84-12-rk2410-nocsf_6.1.84-12_arm64.deb)的 `boot/config-6.1.84-12-rk2410-nocsf` 未设置该配置；按这份本地头文件构建会选 Type A，包内修改版的确切能力仍应与其源码、运行 input 能力对应。

Type A 每帧逐个发送所有当前触点，以 `input_mt_sync()` 分隔不同触点，再用 `input_sync()` 结束整帧。以下摘录 `gtp_touch_down()` 的 Type A 分支，坐标变换已在此之前执行：

```c
input_report_key(ts->input_dev, BTN_TOUCH, 1); /* 当前至少有一个接触 */
input_report_abs(ts->input_dev, ABS_MT_POSITION_X, x);
input_report_abs(ts->input_dev, ABS_MT_POSITION_Y, y);
input_report_abs(ts->input_dev, ABS_MT_TOUCH_MAJOR, w);
input_report_abs(ts->input_dev, ABS_MT_WIDTH_MAJOR, w);
input_report_abs(ts->input_dev, ABS_MT_TRACKING_ID, id);
input_mt_sync(ts->input_dev);                 /* 分隔这一触点，尚未结束整帧 */
```

`goodix_ts_work_func()` 的静态 `pre_touch` 在 Type A 中保存上次有效触点数。有触点时遍历本帧全部记录；当前触点数为零且 `pre_touch` 非零时，调用 `gtp_touch_up()` 报 `BTN_TOUCH=0`；随后把 `pre_touch` 更新为当前数目，统一 `input_sync()`。这份释放函数没有额外的 `input_mt_sync()`，但提供了 BTN_TOUCH 的释放状态，符合这种组合下的 Type A 零接触表达。[Linux v6.1 多点协议](https://www.kernel.org/doc/html/v6.1/input/multi-touch-protocol.html)

沿同一个 ID 看连续帧：

| 有效帧 | 芯片中的当前接触 | Type A 的表达 |
| --- | --- | --- |
| A | ID 1 位于原始 `(300,100)` | 报告变换后的该点及 ID，分隔触点并同步整帧 |
| B | ID 1 移动，新增 ID 3 | 报告本帧两点，各有 `SYN_MT_REPORT`，最后 `SYN_REPORT` |
| C | 仅剩 ID 3 | 当前接触集合只含 ID 3，ID 1 不再出现在集合中 |
| D | 状态 `0x80`，零点 | 旧状态非零，报告 `BTN_TOUCH=0` 并同步 |

移动时保留接触身份，抬起时让该身份从本帧集合消失；这与“只有位置变化才发一条消息”的思路不同。

### 7.3.3 Type B 用 slot 保存接触状态，缺席必须显式释放

Type B 的 slot 是持续保存一个接触状态的位置，tracking ID 区分当前接触。旧 slot 分支把芯片 ID 同时用作 slot 索引和 tracking ID；存在的点调用 `gtp_touch_down()`，缺失的点调用 `gtp_touch_up()`，后者向相应 slot 写入 `ABS_MT_TRACKING_ID=-1`。

同一 ID 从 A 帧移到 B 帧，仍更新同一 slot；C 帧缺少 ID 1 时，要显式将 slot 1 置为未使用；D 帧的有效零点会释放余下 slot。消费者保存了 slot 状态，单纯不再发送位置不会自动抬起手指。

旧分支的 `pre_touch` 此时保存 ID 位图。算法从槽 0 递增扫描到 9，只在处理当前记录后才推进到下一条记录，因而要求记录 ID 落在这一遍历范围并按相容顺序到达。例如先给 ID 3、再给 ID 1，循环读到第二条时已经越过槽 1，后者就无法按预期更新。虽然初始化申请了 16 个 slot，运行循环仍只处理 `GTP_MAX_TOUCH=10` 个位置；不能用 slot 数代替算法的实际范围。

Type A/B 是标准事件的组织方法，芯片 8 字节记录格式没有随之改变。选择一种报告协议后，解析、身份管理、释放和同步必须完整配套。

## 7.4 同步输入之后，还要完成芯片侧确认

`input_sync()` 产生 `SYN_REPORT`，让 input/evdev 将本轮变化交给上层。它不等待桌面绘制，也不清除触摸芯片的 ready 标志。

旧工作函数正常完成上报后，进入以下源码出口。`end_cmd` 的内容是 `81 4E 00`：前两字节选择状态寄存器，最后一字节写零。

```c
exit_work_func:
    if (!ts->gtp_rawdiff_mode) {             /* 原始数据调试模式保留另一确认策略 */
        ret = gtp_i2c_write(ts->client, end_cmd, 3);
        if (ret < 0)
            GTP_INFO("I2C write end_cmd error!");
    }
    if (ts->use_irq)
        gtp_irq_enable(ts);                 /* 解除回调开始时保持的显式禁用 */
```

正常次序是读取、解析、上报、同步、清芯片状态、恢复 IRQ。IRQ 标志为电平触发时，芯片若继续保持有效电平，重新使能后可能再次通知；确认写失败的日志因此要和 INT 电平、下一次读取共同分析。另一方面，ready 未置位、数量超限等分支也跳到该出口，但没有产生 input 帧；原始零状态和首读负错误则提前退出，行为不能合并。

evdev 交付的是设备坐标和接触事件。桌面输入栈把它映射到显示输出、窗口和控件；只有原始 input 范围、四角位置与释放都正确后，才应把偏移问题收敛到上层映射。这里的 input 帧和芯片确认完成条件各不相同。

## 7.5 错误路径怎样留下旧触点，怎样破坏本帧数据

### 7.5.1 短传输和续读失败需要按完整帧判断

旧 `gtp_i2c_read()` 最多尝试 5 次，只在返回 2 时结束重试；用尽后普通 GT9xx 会复位，但函数仍原样返回最后的 `ret`。工作函数首读只判断 `ret < 0`，因此 0 或 1 这种非负短传输仍可能继续解析。若缓冲区保持初始零值，会走零状态退出；若包含部分返回数据，则可能误把残缺内容当作状态或坐标。

多点续读更直接：7.2.2 中的 `memcpy()` 不检查返回值，失败后仍将续读缓冲区并入帧。第一点可能正确，第二点以后却来自不完整数据，形成“单指正常、多指错乱”的具体原因。增加重试只能降低某些瞬时错误的出现概率，不能修复调用者接受残帧的条件。

写操作完整成功要求返回 1，旧辅助函数失败后也保留原值，末尾只检查 `<0` 同样会漏过返回 0 的情况。不能只统计负错误日志来判断通信完整性。

### 7.5.2 丢弃无效帧不等于释放上一次接触

旧驱动首次读取负错误、零状态、ready 无效或数量超限时，均没有执行触点释放；`pre_touch` 与 input 中已有接触保持不变。复位芯片也没有同步清除这些软件状态。若下一轮得到有效零点，正常释放会补上；若错误持续或恢复失败，上层可能继续保留拖拽、按下等状态。

这解释了两种容易混淆的操作：清芯片寄存器处理硬件握手，报告 release 处理上层接触生命周期。复位成功、重新允许 IRQ、清掉睡眠标志，都没有替代一帧释放。本地代码没有在通信错误或复位之后统一释放旧接触的恢复步骤。

`pre_touch` 是函数静态变量，旧坐标配置和两套驱动的 `i2c_connect_client` 也存在全局状态。如果扩展为多个同类触摸设备，必须把每个设备的上一帧、坐标配置和维护状态分别归属，否则设备间会共享不该共享的状态。

## 7.6 V2.8.7 保留芯片布局，改变采集与报告组织

### 7.6.1 IRQ 线程直接处理，解析后先确认芯片再报告

阿美林 V2.8.7 的正常链如下，函数均来自该分支，不与旧 workqueue 混用：

```text
request_threaded_irq(NULL, gtp_irq_handler, IRQF_ONESHOT)
  → IRQ 线程调用 gtp_work_func
  → 检查 PANEL_RESETTING / WORK_THREAD_ENABLED
  → gtp_get_points：首读、续读、解析到 points[ID]
  → 非 RAW_DATA_MODE 时清芯片状态
  → gtp_mt_slot_report 或 gtp_type_a_report
  → input_sync → evdev
```

这条路径没有第二次采集 workqueue 投递，I2C 和重试耗时都包含在本轮 IRQ 线程处理中。`gtp_i2c_read()` 只有完整的两条传输成功才复制数据，重试间等待约 2 ms，用尽后统一返回 `-EAGAIN`；`gtp_get_points()` 也检查多点续读错误，将该帧状态置零后退出。

每条记录按 `id = coor_data[0] & 0x0f` 存进 `points[id]`，数组长度 `GTP_MAX_TOUCH_ID=16`，每个元素有 `status/id/x/y/w/p/tool_type`。默认 Type B 遍历这些槽，`status=1` 时调用 `input_mt_report_slot_state(..., true)` 并上报位置，缺席时报告 false。tracking ID 由 input MT helper 管理，不能套用旧版“直接上报芯片 ID 就是 tracking ID”的代码。

有效零点帧会先清零整个 points 数组，再将所有报告范围内的 slot 置为释放。无效帧或读取失败返回 0，`gtp_work_func()` 直接退出，仍保留上一帧 input 状态。其完整读取检查比旧版更严格，但没有额外补上错误后统一释放接触的机制。

硬件确认时机也变了：`gtp_get_points()` 在返回给报告函数前，已在 `exit_get_point` 尝试写零清状态；写失败只打印日志，不改变原本有效的 `finger_state`，后续仍可上报。旧版在 input 同步之后确认，新版在上报之前确认，这个差异会影响日志与事件的先后判读。

### 7.6.2 Type A 可选路径没有正确匹配稀疏 ID 布局

`goodix,type-a-report` 为 0 时使用上述 Type B，为非零时改走 `gtp_type_a_report()`。解析器已经按 ID 存成稀疏数组，报告函数却不检查 `points->status`，而是仅在 `i == points->id` 时使用该元素并把指针前移：

```c
for (i = 0; i < ts->pdata->max_touch_id; i++) {
    if (touch_num && i == points->id) {  /* 没有同时检查该元素 status */
        /* ... 上报 points 的 ID、坐标、接触量，调用 input_mt_sync ... */
        cur_touch |= 0x01 << points->id;
        points++;                       /* 仅匹配时前移，并非每轮前移 */
    } else if (pre_touch & 0x01 << i) {
        /* ... 处理前一帧工具状态 ... */
    }
}
```

例如本帧只有 ID 3，解析器将真实点放在 `points[3]`，其他元素为零。报告函数在 i=0 时遇到 `points[0].id=0`，即使 status 为 0 也报告该空元素；随后指针停在 `points[1]`，其 id 仍为 0，后续 i 不再匹配，真正的 ID 3 不会被读到。触发条件是启用本地新分支的 Type A 且出现这种稀疏 ID，默认 Type B 的逐槽 status 判断不走这段逻辑。

这是布局与遍历不一致造成的缺陷，修复应围绕按 ID 索引的有效元素遍历。更换屏幕旋转、修改最大坐标或调整 IRQ 极性都无法补回遗漏的真实记录。

### 7.6.3 无 IRQ 的 timer 直接采集，执行上下文不相容

新分支只有在既无有效 IRQ GPIO、也无正 IRQ 编号时启用 timer；已经有硬件 IRQ 但 `request_threaded_irq()` 失败，会返回错误，不自动改成轮询。

其 `gtp_timer_handler()` 直接调用 `gtp_work_func()`，继而进入 I2C、`usleep_range()` 重试，较长读取还可能 `GFP_KERNEL` 分配。普通非线程化 hrtimer 回调不允许这类可睡眠操作，因此这条无 IRQ 路径存在上下文错误。旧分支让 timer 仅排队、由线程采集，正是为了把到期通知和可等待的工作分开。相关睡眠约束见 [Linux v6.1 延迟接口说明](https://www.kernel.org/doc/html/v6.1/timers/timers-howto.html)。

## 7.7 复位、更新与调试访问会改变正在采集的芯片

两套代码的开关保护范围各不相同。旧 `irq_lock` 只保护 `irq_is_disable` 和开关操作，I2C 不在该自旋锁内；新 `WORK_THREAD_ENABLED/PANEL_RESETTING` 只控制入口是否继续，不能等待已越过检查的线程。I2C adapter 可串行安排单次传输，但不会把“复位—下发配置—读取—确认”自动变成不可打断的一组动作。挂起恢复与 remove 的具体顺序已在[第 05 章](05_驱动匹配与初始化.md)展开。

旧头文件的 `GTP_ESD_PROTECT/GTP_AUTO_UPDATE/GTP_CREATE_WR_NODE` 均为 0，普通路径不启用这些功能；独立的配置 proc 入口仍会在 probe 创建。`GTP_COMPATIBLE_MODE=1` 保留 GT9xxF 特殊请求：只有芯片识别为该类且状态为零，才在普通 ready 检查前读取配置、复位或主时钟请求。运行请求中的备份参考处理主体已注释，不能因为辅助函数存在就把它画进普通每帧路径。

V2.8.7 的 `goodix,esd-protect` 可以安排每 2 s 的延迟工作，读取看门狗相关状态，异常时尝试断电、上电、复位和重发配置。`gtp_esd_off()` 使用同步取消等待 ESD work；ESD mutex 管理的是这项任务的开关，并未把普通触点、调试读写和整次复位包在同一把锁里。

[新分支 `gt9xx_update.c`](../../阿美林-1000nit-7inch/GT9XX通用调试包/gt9xx_update.c)的 `gup_update_proc()` 用 `test_and_set_bit(FW_UPDATE_RUNNING)` 排除重复更新，取得并检查触摸固件后关闭普通上报与 ESD，进入更新模式，按 SS51/DSP/boot 等区域执行有界烧写重试，最后退出更新模式、释放固件并恢复工作。它更新的是触摸芯片程序，与 Linux 内核镜像、触摸 CFG 都是不同对象。阶段校验与重试可以处理部分传输失败，代码没有提供断电后的双分区原子回滚机制。

新 probe 还无条件调用 `init_wr_node()`。[`goodix_tool.c`](../../阿美林-1000nit-7inch/GT9XX通用调试包/goodix_tool.c)以 0666 权限建立 proc 维护入口，允许访问芯片和更新命令。写处理先从用户缓冲区复制固定命令头，再检查部分数据长度；地址长度、偏移与全局 `cmd_head` 的共享访问需要成套约束。该接口位于标准 evdev 之外，普通桌面触摸不依赖它；发布配置应限制其权限、输入边界及与采集的并发访问。

## 7.8 让观察结果决定沿哪一段继续查

| 现象 | 最有区分力的观察 | 结果怎样改变方向 |
| --- | --- | --- |
| input 设备缺失 | I2C client、driver 绑定和 input 注册日志 | 回到 probe，暂不调整坐标映射 |
| 没有触摸响应 | 当前 IRQ/轮询模式、INT 变化、工作开关 | 通知不到查引脚和触发；通知已到查工作状态及读取 |
| INT 持续有效 | 读取状态字、清零写结果及触发类型 | 区分未确认的有效事件与触发配置、电气电平错误 |
| 单指正常、多指异常 | 续读完整性、记录 ID 和报告分支 | 旧版查未检查的续读；新版 Type A 查稀疏数组遍历 |
| 抬起后残留 | 有效零点帧是否到达、释放事件是否上报 | 无有效帧查通信/恢复；已有释放则查上层消费 |
| input 四角正确，界面偏移 | input 范围与桌面输出映射 | 定位用户空间变换，避免再次修改正确的驱动坐标 |
| 唤醒后断续或无响应 | 完整 I2C 返回、旧接触释放及新帧 | 区分软件重新允许工作与芯片真正恢复 |

在目标 Linux 系统已有工具时，可按以下只读命令识别设备并观察事件，输出按真实设备判读：

```sh
# 先从名称、能力和 Handlers 找到 GT9XX 对应的 event 编号。
cat /proc/bus/input/devices

# 比较对应 IRQ 触摸前后的变化；轮询模式无需 IRQ 计数增长。
cat /proc/interrupts

# eventN 替换为上一步识别出的设备，观察位置、同步和释放。
evtest /dev/input/eventN
```

单次事件不是完整生命周期。至少把按下、移动、多点变化、全部抬起放在同一次观察中，再结合四角方向判断数据与上层坐标是否对应。具体故障分流见[第 11 章](11_故障排查与验证.md)。

## 7.9 核心工程问题与深度理解

### 7.9.1 首读成功而双指偶发跳点，应先检查哪个交接

先检查多点续读是否完整，再检查 ID 到报告对象的组织。旧分支首读只覆盖第一点与第二点开头，续读失败仍复制数据，因而第一点正常无法排除后半帧损坏；V2.8.7 已拒绝失败续读，但其可选 Type A 还存在稀疏 ID 遍历缺陷。先确定分支，才能选对观察点。

### 7.9.2 芯片复位成功，上层为什么还可能保持按下

复位只改变芯片状态，input 保留的接触需要 release 事件结束。两套本地分支在丢弃无效帧时都没有统一释放旧接触，软件恢复采集后仍要收到并处理有效零点或明确执行释放。清状态寄存器、清睡眠标志和 input 释放是三个不同动作。

### 7.9.3 IRQ 线程已经允许睡眠，旧版为什么仍要显式禁用 IRQ

因为它只排队 work 就返回，ONESHOT 的屏蔽区间不会延长到独立工作线程。显式禁用由 work 末尾解除，覆盖了本次读取与确认；但这个开关不负责等待 PM、调试或卸载路径中的所有并发访问，资源寿命还要按第 05 章处理。

### 7.9.4 显示转成横屏后，为什么不能直接再交换一次触摸 X/Y

应先确定控制器原始范围、驱动是否已交换或镜像、input 声明范围以及桌面映射。当前驱动会在注册能力和报告坐标时分别处理范围与点，显示方向又有自己的消费路径。盲目再交换一次可能重复变换；四角事件与显示位置的对应能区分缺少变换和多做了一次变换。

[上一章：显示与背光](06_显示与背光.md) · [下一章：电量与外部供电](08_电量与外部供电.md) · [返回总览](README.md)
