# marble XNU —— iOS 16 内核跑在 Redmi Note 12 Turbo 上

把 Apple 开源的 **XNU 8796.141.3**（iOS 16 同期内核）跑到
**Redmi Note 12 Turbo（marble / SM7475 / Snapdragon 7+ Gen 2）** 上，
一直启动到 iOS 16 用户态（SpringBoard）。

启动方式是 **shim + `fastboot boot`，纯 RAM 运行**：不刷 boot、不写
userdata、不改 GPT。测试结束内核自己复位，手机回到 TWRP/Android。

> 这是一个研究项目，只用于学习和实验。仓库**不包含** Apple、Qualcomm
> 或小米的任何二进制；请不要把它当成日常可用的手机系统。

## 安全规则（先读）

1. **只允许 `fastboot boot`**。禁止 `flash` / `erase` / `format` /
   `set_active` / `oem` 写命令；不要碰 GPT / persist / modemst。
2. 测试前确认电量 > 50%，并知道怎么进 fastboot（Power + Vol Down）和 EDL。
3. 只对**你自己的研究机**使用。本项目默认机器序列号 `2aa26095`，
   所有 `adb` / `fastboot` 命令都应带 `-s`。
4. XNU 源码由 CI 从公开源构建；Apple 用户态镜像、ADT、ADSP 固件
   都需要你从自己的设备/IPSW 提取（见 [docs/required-assets.md](docs/required-assets.md)）。

## 机器信息

测试机是 2023 年的 Redmi Note 12 Turbo（代号 **marble**，POCO F5 同 SoC）。
下表中的寄存器地址来自本项目在真机上读到的 DT / 日志。

| 项目 | 值 |
|---|---|
| SoC | Qualcomm **SM7475-AB**（Snapdragon 7+ Gen 2），4 nm |
| CPU | 1× Cortex-X2 2.91 GHz + 3× Cortex-A710 2.49 GHz + 4× Cortex-A510 1.8 GHz（ARMv9，8 核） |
| GPU | **Adreno 725**（gen7-4-0）；`kgsl-3d0` @ `0x3d00000`，`kgsl-smmu` @ `0x3da0000`，GMU @ `0x3d68000`，GPUCC（`qcom,cape-gpucc`）@ `0x3d90000` |
| 内存 / 存储 | LPDDR5 / UFS 3.1 |
| 屏幕 | 6.67" 1080×2400 OLED 120 Hz；厂商背光是 **`bl_ctrl_dcs`**（DSI DCS `0x51`），不是 WLED |
| 触摸 | Goodix **GT9916S**，XNU 下用 GPIO 软件 SPI 驱动 |
| 音频 | TDM-LPAIF-RX-**TERTIARY**（96 kHz / 32 bit）→ 2× **AW882xx**（i2c `0x34` / `0x35`），无 WSA |
| Wi-Fi / BT | **WCN6750** 家族，DT 为 `qcom,cnss-qca6490`（PCIe rc-num 0 + IPA） |
| PMIC | PMK8350（RTC/SPMI）、PM8350C、PM8350B（HAP 振动） |
| 充电 / 电量 | pmic_glink **battmgr**（owner `0x800A`），经 ADSP/GLINK 访问 |
| 电池 | 5000 mAh，67 W 有线（公开规格） |
| iOS root | userdata 尾部 HFS+ 镜像：`ufsroot_lba=0x3603600`，`blocks=5243387` |
| 其他 | 侧边指纹、3.5 mm 耳机孔、红外；相机/传感器/基带未移植 |

## 现在能跑什么（内核 v99，2026-10-29）

| 子系统 | 状态 | 说明 |
|---|---|---|
| 引导到 SpringBoard | ✅ | shim + 自建 ADT + XNU 8796，`fastboot boot` 纯 RAM |
| 显示 | ✅ | 1080×2400 framebuffer；背光亮度尚未控制（见下） |
| 触摸 / 按键 | ✅ | GT9916S 软件 SPI；触摸有已知限制，见 status-log |
| USB | ✅ | DWC3 device 模式 + CDC-ECM；v94 修掉 CLEAR_FEATURE 风暴 |
| UFS / HFS+ root | ✅ | userdata 尾部挂载，可读写 |
| 8 核 SMP | ✅ | `gicrstride=0x40000` 修复后 8/8 起核并有调度采样 |
| ADSP / GLINK / GPR / APM | ✅ | 控制面全通，GRAPH_OPEN/PREPARE/START、TDM SET_CFG 均 status 0 |
| 音频数据面 | 🔴 | DSP 一旦读主机共享内存整机冻结；缺 18 KB APM SET_CFG（[audio-closeout](docs/audio-closeout.md)） |
| 电池（pmic_glink） | 🟡 | v99 双属性槽 + 通知解析 + `sweetbattsweep`，待真机验证 |
| RTC | 🟡 | 能读；PMK8350 RTC 归 EE4 所有，AP 写入被拒；时间来自 `sweettime` |
| 振动 | ✅ | PM8350B HAP 电源键触发；Apple 触觉栈有 stub（0196–0201） |
| 背光 | 🔴 | 需要 DSI DCS `0x51`，XNU 侧未实现；没有 WLED 可用 |
| GPU | 🟡 | shim 只读探针 level 6/7：按 ID1 算出的 CB 窗口 + sACR + GPUCC；无 KGSL 驱动 |
| Wi-Fi / BT / 相机 / 传感器 | ⬜ | 未开始；WCN6750 可参考 HakiOS |
| 调制解调器 | ⬜ | 未开始 |

> 音频虽然"控制面全绿"，但只要 DSP 真的去读共享内存，整机在几毫秒内冻结，
> 且不是可记录的 fault。已排除 apps-smmu 配置、S1 页表、`CFCFG` 和 buffer
> 物理地址四个维度。完整证据见 [docs/audio-status.md](docs/audio-status.md)。

## 实现概要

```
fastboot boot boot.img (v4)
        │
        ▼
  shim (AArch64 裸机)
    ├─ 解析 cmdline，构造 ADT / 保留内存 / framebuffer
    ├─ 容错 MMIO 探针（同步 abort 可恢复），结果页 @ 0xa2e0f000
    └─ 跳转 XNU payload
        │
        ▼
  XNU 8796 (DEVELOPMENT, SWEET)
    ├─ 16K 页 / GICv3 / vtimer / DWC3 / UFS / HFS+
    ├─ bsd/dev/sweet_glink.c    GLINK + GPR + APM + battmgr + RTC + HAP
    └─ 启动 iOS 16 launchd → SpringBoard（HFS+ root）
```

- XNU 补丁全部在 [`xnu/patches/`](xnu/patches/)（194 个）。
- shim 在 [`shim/`](shim/)，结果页 / GPU 探针的字段和 level 说明见
  [docs/gpu-probe.md](docs/gpu-probe.md)。
- 内核日志走 **SWEETLOG**：userdata 尾部的环形区，内核跑完用
  `scripts/pull_logs.sh` 拉回，原理见 [docs/sweetlog.md](docs/sweetlog.md)。

## 从零构建与部署

### 0. 你需要

- 一台 bootloader 已解锁的 marble（本机序列号 `2aa26095`）。
- macOS（Apple Silicon / Intel 均可）+ `adb` / `fastboot` / Python 3。
- 构建 shim 需要 **Android NDK**（`ANDROID_NDK_HOME` 指向 NDK 目录）。
- 构建 ADT 需要 Apple 的 **D22AP 设备树**（从你自己的 iPhone X IPSW 提取，
  仓库不包含）：放到 `research/ipsw-x/d22ap.adt`，或设置 `D22AP_ADT`。
- 手机端需要 root 才能抽取 ADSP 固件（本机用 SukiSU Ultra）。
- 一个写入 userdata 尾部的 HFS+ iOS 用户态镜像（Apple 内容，本项目不分发）。

完整清单、来源与法律说明见 [docs/required-assets.md](docs/required-assets.md)。

### 1. 构建 XNU 内核（GitHub Actions）

Fork 本仓库后：

- 仓库 → **Actions** → `xnu` → **Run workflow**
  - `macos_version`: `13.5`（xnu-8796）
  - `kernel_config`: `DEVELOPMENT`
  - `machine_config`: `SWEET`
- 等 30–60 分钟。成功后 release `kernel-latest` 滚动更新，产物
  `kernel.development.sweet`。CI 还会校验每个补丁都真的应用、且内核里
  存在关键字符串标记，避免"补丁被跳过但构建仍绿"。

取回内核（必须用脚本处理过的版本）：

```bash
GH_REPO=lietxia/ios-port-marble ./scripts/fetch_kernel.sh
# 输出 out/kernel.adsp（校验 sha256，并把 EL1 不可用的 MSR CNTP_CTL_EL0 改成 NOP）
```

> 不提供本地编译脚本：xnu-8796 需要 Xcode 15.x 的 `iig` 和对应 KDK，
> Actions 的 `macos-14` 已配好；本地环境不一致会出现 IORPC ABI 错误。

### 2. 从手机抽取 ADSP 固件（不随仓库分发）

```bash
SERIAL=2aa26095 ./scripts/extract_adsp_fw.sh
# 默认写到 ./adsp-fw/（adsp2.mdt + adsp2.bNN）
```

### 3. 生成 RAM-only 启动镜像

```bash
D22AP_ADT=/path/to/d22ap.adt ADSP_FW_DIR=$PWD/adsp-fw ./scripts/mkimage.sh
# 输出 images/shim-iosroot-adsp.img
```

内部流程：`mkfwblob.py` 打包固件 → `mkpayload.py` 生成 XNU payload →
`shim/build_xnu.sh` 编译 shim 并封成 boot.img v4。

### 4. 启动（不写任何分区）

```bash
SERIAL=2aa26095 ./scripts/boot_ram.sh images/shim-iosroot-adsp.img
```

内核跑完（cmdline 里 `sweetreboot=1`）会自己复位，手机通常落在
TWRP/recovery。

### 5. 取回日志

```bash
SERIAL=2aa26095 ./scripts/pull_logs.sh run1
# logs/raw/ring-run1.bin + logs/ring-run1.txt
```

## 启动参数速查

`mkimage.sh` 默认写入的 cmdline 里已包含 `sweetglink=1 sweetadsp=1
sweetbatt=1 sweetkeys=1 marble_touch=1 gicrstride=0x40000 sweetreboot=1`
等。常用调试开关：

| 参数 | 作用 |
|---|---|
| `sweetglink=1` | 启用 GLINK/GPR/APM/电池/RTC/振动这条总线 |
| `sweetadsp=1` | 启动 ADSP（PAS 加载） |
| `sweetbatt=1` | 打开 pmic_glink 电池客户端（默认） |
| `sweetbattsweep=1` | 开机扫 battery 0..31、USB 0..11 并打印原始值（属性表自检） |
| `sweetbattery=1` | 启用早期的 PM6150 电压估算回退路径 |
| `sweetsmp=1 sweetsmpn=20 sweetsmpms=1000` | 多核调度采样，采样完主动复位（测试用） |
| `sweetgpu=1` | shim 在跳转 XNU 前做 GPU SMMU 只读探针（构建时用 `GPU_PROBE_LEVEL` 选级别） |
| `sweethap=1` | 开机时让 PM8350B HAP 响一下（默认安静） |
| `sweetshmhi=1` | APM SHM_MAP 高字用 stream index（音频实验，见 audio-closeout） |
| `sweettime=<epoch>` | 构建镜像时写入，iOS 日历时间的来源 |
| `sweetlog=0` | 关闭 SWEETLOG 环形写盘（keepalive 测试用） |

`kern.sweetbatt{cap,mv,ma,temp,cycle}` 是 v99 新增的只读 sysctl，
可在 iOS 里直接看 battmgr 的缓存值，不必每次都拉 ring。

## 同类实现与上游进度（2026-10-29 调研）

| 项目 | 状态 / 可借用度 |
|---|---|
| [jebbranch/ios-port-marble](https://github.com/jebbranch/ios-port-marble) | 本仓库的上游，main 停在 `a7b9177`（2026-10-28） |
| [G3134377393/ios-port-marble](https://github.com/G3134377393/ios-port-marble) | 领先上游 18 个提交：Adreno 725 stage-1 MMIO probe、GPUCC GDSC 读取、HFS 大端写修复。**GPU 下一步最直接的参考** |
| [UniXNU](https://github.com/UniXNU) / [marble 页](https://unixnu.com/devices/marble) | 全部面向 SM8550；marble 的驱动全部标 planned，A740 驱动不适用于 SM7475 |
| [oosexgang/hakios-xnu-kernel](https://github.com/oosexgang/hakios-xnu-kernel)（HakiOS） | 已实现 WCN6750 Wi-Fi、AW86927 触觉、battmgr、摄像头；Wi-Fi/触觉路线主要参考。许可证为 NOASSERTION，移植前需确认 |
| [dipa-studio/ios-sweet](https://github.com/dipa-studio/ios-sweet) | `xnu/patches/0072`–`0081` 是 DWC3 device 模式 + CDC-ECM 的现成参考 |
| [sing0626/UniXNU-marble](https://github.com/sing0626/UniXNU-marble) | 另一条旧路线，只到 iOS Hello，触摸和初始化未完成 |

调研结论：

- **电池**：HakiOS 在同一 ADSP 族上实测 vendor 固件在温度前多插一项
  （temp=13、counter=15、cycles=16、design=17、full=18）；mainline
  `qcom_battmgr.c` 的温度是 12，且把值除以 10 得到 0.1 °C。v99 用双槽 +
  合理性检查同时兼容两种编号，并用 `sweetbattsweep=1` 在真机上一次性确认。
- **背光**：marble 是 OLED，vendor 走 `bl_ctrl_dcs`（DSI DCS `0x51`）；
  没有任何 XNU 项目实现过它，也不是 WLED。
- **RTC**：没找到任何项目通过 ADSP/GLINK 设置 PMK8350 RTC；HakiOS 和
  ios-sweet 都由 loader/宿主注入时间，EE4 归属问题无人绕过。
- **触觉**：HakiOS 用 AW86927（I2C），marble 是 PM8350B HAP，二者不通用；
  本项目的 HAP kick 已在真机验证。
- **Wi-Fi/BT**：WCN6750 有 HakiOS 的 XNU 端口可参考；蓝牙没有现成方案。
- 没有找到 marble / POCO F5 的专门中文实录，也没有其他 SM7475 移植。

## 仓库结构

```
.github/workflows/xnu.yml  GitHub Actions：构建 XNU 内核并发布 kernel-latest
xnu/patches/               XNU 的全部移植补丁（含音频、UFS、触摸、USB、电池…）
shim/                      RAM-only 引导 shim（AArch64 裸机 + boot.img v4 打包）
scripts/                   取内核 / 打包 / 启动 / 拉日志 / 抽固件 / 探针
docs/                      硬件状态、音频、SMMU、GPU 探针、SWEETLOG 等文档
```

## 文档索引

| 文件 | 内容 |
|---|---|
| [docs/START-HERE.md](docs/START-HERE.md) | 入口与当前结论速览 |
| [docs/status-log.md](docs/status-log.md) | 按时间追加的完整实验日志（最新结论先看这里） |
| [docs/audio-status.md](docs/audio-status.md) / [audio-closeout](docs/audio-closeout.md) | 音频控制面、SHM_MAP 高字、剩余 18 KB 缺口 |
| [docs/smmu-dead-ends.md](docs/smmu-dead-ends.md) | apps-smmu 寄存器地图与已证伪路线 |
| [docs/gpu-probe.md](docs/gpu-probe.md) | kgsl-smmu / CB 偏移 / GPUCC 探针（level 1–7） |
| [docs/speaker-path.md](docs/speaker-path.md) | 扬声器 = TERTIARY TDM + 2×AW882xx |
| [docs/handoff.md](docs/handoff.md) | 常用命令、boot args 速查 |
| [docs/required-assets.md](docs/required-assets.md) | 需要的 Apple 资产及来源/法律说明 |

## 已知卡点（下一步）

1. **音频**：抓厂商那块 18 KB 的 APM 级 SET_CFG；拿到后才能让 DSP 读
   PCM 共享内存而不冻结。
2. **电池**：真机跑一次 `sweetbatt=1 sweetbattsweep=1`，确认属性表编号和
   `kern.sweetbatt*`；再验证拔线后充电图标消失。
3. **GPU**：先跑 level 6/7，确认 CB0 是否可写、GPUCC 时钟是否可控；之后
   才谈 KGSL 初始化。
4. **背光**：实现 DSI DCS `0x51` 写入，需要先确定 marble 面板的具体 panel。
5. **Wi-Fi**：WCN6750 的 WPSS/SMEM/SMP2P/IPCC/GLINK/QRTR 16 步流程，
   参考 HakiOS 的 `hakios.wifi.c`。

## 常见问题

| 现象 | 原因 / 处理 |
|---|---|
| `Undefined kernel instruction ... d51be228` | 用了未打 CNTP_CTL 补丁的内核，重新跑 `fetch_kernel.sh` |
| 启动后没有输出 | 用 `scripts/pull_logs.sh` 从 userdata 尾部读 SWEETLOG |
| `fastboot boot` 后设备停在 TWRP | 正常回退；直接拉日志即可 |
| 日志是上一轮的 | 看 `SWEET RTC: clock set from sweettime=` 是否等于本镜像的 `sweettime` |
| 内核 panic 花屏 | 拍照记录 pc/lr/far + 寄存器；重启进 TWRP 拉上一轮日志 |
| GitHub 下载很慢 | 设置 `MARBLE_PROXY=http://127.0.0.1:7890` |

## 许可与声明

- 本仓库的补丁、shim 与脚本按仓库 LICENSE 发布。
- XNU 源码版权归 Apple，按其开源许可使用；构建过程从公开源拉取。
- Qualcomm/Xiaomi 固件、Apple 用户态镜像与 D22AP ADT **不在仓库内**，
  需从你自己的设备/IPSW 提取；请勿上传到任何公开渠道。
- 仅供研究，不提供任何担保；因使用导致的设备损坏/数据丢失由使用者自负。
