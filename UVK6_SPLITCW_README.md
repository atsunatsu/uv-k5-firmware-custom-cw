# UV-K5 / UV-K6 NR7Y MAIN RX / SUB TX + INV TRACK

中文说明在前，English documentation follows the Chinese section.

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

## 新增 CW 输入方式

`CWkin` 在原有 0–9 项之后追加三个选项，旧 EEPROM 菜单编号不变：

- `CEC HandKey`：未改机也可通过现有 10K/20K CEC 电阻线使用外接直键。任一电阻编码触点闭合都会直接开启载波，松开则关闭载波；不会按 WPM 自动生成点或划。和其他 CEC/ADC 模式一样，机身 PTT 此时不能使用。
- `PTT dah / EXIT dit`：机身 PTT 作为划、面板 EXIT 作为点，支持 squeeze 与 Iambic A/B。
- `PTT dit / EXIT dah`：上述按键的点划方向反转。

PTT+EXIT 模式在主屏幕、CPO 练习屏幕和 CW 宏录制期间接管 EXIT；其他菜单中 EXIT 仍正常执行返回。CPO 中选择 PTT+EXIT 时使用 MENU 退出练习屏幕，宏录制时按 MENU 保存并退出。

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
10. 选择 `CEC HandKey`，分别闭合 CEC 线的 10K 与 20K 触点；确认两者都按实际按压时长控制载波，长按不自动重复点划，松开约 300 ms 后回 MAIN RX。
11. 依次选择两个 PTT+EXIT 模式，检查单点、单划、同时按压 squeeze、Iambic A/B 和正反向映射；在 CWmsg 录制中确认 EXIT 可作为桨键且 MENU 能保存，退出录制后确认 EXIT 仍能正常返回。
12. 在五个 RxMode 间反复切换并重启；确认用户选择保持，旧四模式下 DWR/XB 行为正常，第五模式下旧 scheduler 不切换 RX VFO。

## 必须真机确认的项目

RF 实际落点、PA 启停、CEC ADC 阈值、首个 dit、快速 squeeze、Iambic A/B 手感、约 300 ms 返回时序、状态栏在充电/锁键/F 键等组合下的实际观感，以及所有硬件拒发条件，无法仅靠交叉编译完全验证。

---

# English documentation

This custom build is based on `zerodrool/uv-k5-firmware-custom-cw` `main` at commit `f91d365ae134c01e557fc0ffcc96f83e7e4e8bf8` (2026-04-24).

## Functionality

The fifth `RxMode`, `MAIN RX / SUB TX`, behaves as follows:

- MAIN remains the selected on-screen VFO, idle receiver, keypad-entry target, and UP/DOWN tuning target.
- SUB is a temporary transmit VFO and does not alter the persistent `gEeprom.TX_VFO` selection.
- TX start temporarily points `gTxVfo/gCurrentVfo` to SUB; normal completion, timeout, or TX rejection always restores MAIN.
- During TX the MAIN selection marker is unchanged, while the SUB row shows `TX` and its configured `pTX` frequency. In normal CW this is the PLL frequency; `CWx` adds `CWfreq` to the actual carrier and that extra offset is not included in the current display.
- Keyer availability is determined by the SUB transmit-role modulation. MAIN may receive CW or USB as long as SUB is CW.
- Selecting this mode disables `DUAL_WATCH` and `CROSS_BAND_RX_TX`; selecting an older mode leaves Split mode.
- The upstream approximately 300 ms CW hang time is retained.

`INV TRACK`, assigned to `F2Shrt` after a full EEPROM reset:

- With INV off, only MAIN tunes.
- With INV on, each MAIN frequency delta is applied equally and oppositely to SUB; the status line shows `INV`.
- It covers UP/DOWN step tuning and direct frequency entry.
- The paired SUB is checked against RX limits, its resulting TX frequency, and TX lock. Failure leaves both frequencies unchanged and emits an error beep.
- Doppler steps update SUB in RAM without an EEPROM write for every step.
- An INV request during CW TX is deferred until the complete session ends and MAIN receive is restored.

CEC Cable, reversed cable, Iambic A/B, ADC paddle detection, macros, and the CW return delay retain the upstream implementation.

## New CW input modes

Three items are appended after the original `CWkin` entries 0–9, preserving older EEPROM menu numbers:

- `CEC HandKey`: uses either 10K/20K resistor-coded CEC contact as a straight key without internal radio modification. Carrier key-down follows the physical press duration and does not generate WPM-timed elements. Radio PTT is unavailable in this ADC mode.
- `PTT dah / EXIT dit`: PTT is dah and the front-panel EXIT key is dit, with squeeze and Iambic A/B support.
- `PTT dit / EXIT dah`: reversed mapping of the preceding mode.

PTT+EXIT owns EXIT on the main screen, Code Practice screen, and during CW macro recording. EXIT remains normal navigation in other menus. Use MENU to leave Code Practice or to save and leave macro recording.

## Split macro recording fix

`CWmsg1`–`CWmsg4` Record, Play, and Repeat validate `SPLITRX_GetTransmitRoleVfo()`. MAIN may therefore remain in USB while SUB is CW. This prevents the incorrect `no keyer!` error previously caused by checking idle MAIN.

## EEPROM compatibility

The implementation uses previously unused bit 5 of NR7Y CW settings byte `Data[2]` at `0x0F22`:

- `0`: `MAIN RX / SUB TX`.
- `1`: one of the four legacy RxModes, represented by the original `DUAL_WATCH` / `CROSS_BAND_RX_TX` bytes.

Older NR7Y code always wrote this bit as zero, so upgraded and fully erased configurations default to the new mode. Selecting a legacy RxMode writes one and persists across reboot. CW and no-CW builds both read the marker, and no-CW saves preserve the other CW-owned bits.

`INV TRACK` is appended to the action enum: ID 23 in the default CW build and ID 15 in the no-CW build. Existing action IDs and side-key EEPROM assignments are not shifted.

## Build

Default matrix:

```sh
make clean
make
```

No-CW matrix, with Code Practice also disabled:

```sh
make clean ENABLE_CW_MODULATOR=0 ENABLE_CODE_PRACTICE=0
make ENABLE_CW_MODULATOR=0 ENABLE_CODE_PRACTICE=0
```

The default Makefile enables CW and LTO. Flash `firmware.packed.bin`; `firmware.bin` is the raw image.

## Flashing

1. Charge the battery and turn the radio off.
2. Connect a reliable Quansheng programming cable.
3. Hold PTT while powering on to enter flashing mode.
4. Select `firmware.packed.bin` in [UVTools Flasher](https://egzumer.github.io/uvtools/) or a compatible local flasher.
5. Wait for success before powering off or disconnecting anything, then reboot normally.

A full EEPROM `Reset ALL` after flashing can be used when deterministic defaults are required.

## First hardware test checklist

1. Set `RxMode = MAIN RX / SUB TX`, MAIN to UHF CW/USB receive, and SUB to VHF CW; verify idle receive remains on MAIN.
2. Monitor SUB with an SDR and send `VVV`; verify SUB shows `TX` and RF appears at the expected frequency. Keep `CWx` off for carrier-equals-display testing.
3. Stop keying and verify MAIN receive returns after approximately 300 ms.
4. With INV off, move MAIN by 500 Hz and verify SUB does not move.
5. Enable INV and verify the status indicator; move MAIN down 500 Hz and verify SUB moves up 500 Hz.
6. Disable INV and verify later MAIN tuning no longer moves SUB.
7. Toggle INV during keying and verify it takes effect only after the full TX session ends.
8. Put SUB on a TX-locked frequency and verify an error beep, no RF, and a clean return to MAIN.
9. Test CEC Cable normal/reversed with `PARIS`, `CQ CQ`, `VVV`, and `599`, including first element, alternation, squeeze, and Iambic A/B.
10. Test both CEC HandKey resistor contacts and verify physical key-down duration, no automatic element repetition, and normal CW hang recovery.
11. Test both PTT+EXIT mappings, squeeze, Iambic A/B, and macro recording. EXIT must act as a paddle during recording, MENU must save, and EXIT must return to menu navigation afterwards.
12. Cycle all five RxModes and reboot, verifying persistence and correct legacy DWR/XB behaviour.

## Hardware validation still required

Cross-compilation cannot fully validate RF frequency, PA timing, CEC ADC thresholds, the first dit, fast squeeze behaviour, Iambic feel, the approximately 300 ms recovery, combined status-line states, or every hardware TX rejection path. Verify these on real hardware using minimum power, a dummy load, an SDR, or suitable test equipment.
