# UV-K5 / UV-K6 NR7Y MAIN RX / SUB TX + INV TRACK

本定制基于 `zerodrool/uv-k5-firmware-custom-cw` 的 `main`，基线提交：
`f91d365ae134c01e557fc0ffcc96f83e7e4e8bf8`（2026-04-24）。

## 功能

`RxMode` 新增第五项 `MAIN RX / SUB TX`：

- MAIN 仍是屏幕选中的 VFO、空闲接收 VFO、键盘输入及 UP/DOWN 调谐 VFO。
- SUB 是临时发射 VFO；不会修改持久的 `gEeprom.TX_VFO` 主 VFO选择。
- 开始发射时只临时将 `gTxVfo/gCurrentVfo` 指向 SUB；正常结束、超时或拒发后均恢复 MAIN。
- 发射期间 MAIN 的用户选择标记不改变，但 SUB 行会显示 `TX` 及其实际 `pTX` 频率，便于直接确认 RF 使用的 VFO。
- CW keyer 的启用由发射角色 SUB 的 modulation 决定；MAIN 可用 CW 或 USB 接收，只要 SUB 为 CW，CEC/Iambic 电键都能启动 SUB CW TX。
- 选择该模式会关闭 `DUAL_WATCH` 与 `CROSS_BAND_RX_TX`。选择任一旧模式则退出本模式。
- CW hang time 保持上游的 300 ms。

`INV TRACK` 可分配给侧键，EEPROM 全复位后的默认分配是 `F2Shrt`：

- INV 关闭：只调 MAIN。
- INV 开启：MAIN 的频率差值会等量反向应用到 SUB；状态栏显示 `INV`。
- 覆盖频率模式的 UP/DOWN step tuning 与键盘直接输入频率。
- paired SUB 同时通过 RX 范围、实际 TX 频率与 TX lock 检查；失败时 MAIN/SUB 均不改变并发出错误音。
- SUB 的 Doppler step 只改 RAM，不为每一步额外写 EEPROM。
- CW TX 期间按 INV 时只记录 pending 值；完整 CW session 结束并恢复 MAIN RX 后才生效。

CEC Cable、CEC Cable Reversed、Iambic A/B、paddle ADC 判断、宏与 300 ms CW 返回时间均保持上游实现。

## EEPROM 兼容方案

使用 NR7Y CW 设置区 `0x0F22`（CW `Data[2]`）原未使用的 bit 5：

- `0`：`MAIN RX / SUB TX`。
- `1`：四个旧 RxMode 之一，仍由原 `DUAL_WATCH` / `CROSS_BAND_RX_TX` 字节表示。

旧 NR7Y 的保存代码总把 bit 5 写为 0，因此升级后默认进入新模式；全擦除值 `0xFF` 也被明确解释为新默认。用户选择旧 RxMode 后 bit 5 写 1，重启后保持选择。启用与禁用 CW modulator 的构建都会读取该标志；无 CW 构建保存时保留其余 CW 数据。读写实现对称。

`INV TRACK` 严格追加在现有 action enum 尾部：默认 CW 构建中为 ID 23；无 CW 构建中为 ID 15。这样不会改变各自构建配置中任何既有 action 的 ID，旧侧键 EEPROM 配置不会因本补丁而后移。

## 构建

默认矩阵：

```sh
make clean
make
```

无 CW modulator 矩阵（Code Practice 依赖 CW，因此同时关闭）：

```sh
make clean ENABLE_CW_MODULATOR=0 ENABLE_CODE_PRACTICE=0
make ENABLE_CW_MODULATOR=0 ENABLE_CODE_PRACTICE=0
```

默认 Makefile 已启用 `ENABLE_CW_MODULATOR=1` 与 `ENABLE_LTO=1`。可刷文件为 `firmware.packed.bin`；`firmware.bin` 是未打包镜像。

## 刷机

1. 确认电池电量充足，关闭电台。
2. 插紧可靠的 Quansheng 编程线。
3. 按住 PTT 开机进入刷机模式。
4. 在 https://egzumer.github.io/uvtools/ 的 Flasher 中选择 `firmware.packed.bin` 并刷写，或使用支持 UV-K5/K6 packed 固件的本地刷机工具。
5. 等待成功提示后关机、拔线、正常开机。刷写过程中不要断电或拔线。

固件会让旧 NR7Y 配置在升级后默认进入第五 RxMode；如需完全确定默认值，可在刷写后执行一次 EEPROM `Reset ALL`。

## 第一轮实机测试 SOP

1. `RxMode = MAIN RX / SUB TX`，MAIN=UHF CW、SUB=VHF CW；确认空闲只接收 MAIN，屏幕 MAIN 选择不变化。
2. SDR 监听 SUB，用 CEC 双桨发送 `VVV`；确认 SUB 行显示 `TX` 和 SUB 发射频率，且 RF 只出现在该频率。
3. 停止发报；确认约 300 ms 后恢复 MAIN RX。
4. INV OFF，MAIN 下调 500 Hz；确认 SUB 不变且无额外状态文字。
5. 短按 Side2 打开 INV；确认只显示 `INV`。MAIN 下调 500 Hz；确认 SUB 上调 500 Hz。
6. 再短按 Side2；确认 `INV` 消失，后续 MAIN 调谐不再联动 SUB。
7. 发报过程中短按 Side2；确认当前 dit/dah 及本次 TX session 不改变，结束回 MAIN RX 后 INV 才切换。
8. 把 SUB 设置到 TX lock 禁止的频率后尝试 CW；确认错误音、无 RF，并仍恢复 MAIN RX，内部 VFO 不会卡在 SUB。
9. 用 CEC Cable 与 CEC Cable Reversed 分别检查快速 `PARIS`、`CQ CQ`、`VVV`、`599`，覆盖首点、点划交替、squeeze、Iambic A 与 B。
10. 在五个 RxMode 间反复切换并重启；确认用户选择保持，旧四模式下 DWR/XB 行为正常，第五模式下旧 scheduler 不切换 RX VFO。

## 必须真机确认的项目

RF 实际落点、PA 启停、CEC ADC 阈值、首个 dit、快速 squeeze、Iambic A/B 手感、约 300 ms 返回时序、状态栏在充电/锁键/F 键等组合下的实际观感，以及所有硬件拒发条件，无法仅靠交叉编译完全验证。
