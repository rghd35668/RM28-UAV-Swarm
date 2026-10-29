# Luckfox Pico (RV1103) 从零到 RKNN Demo —— 保姆级全程手册

> **适用板型**：Luckfox Pico（RV1103 基础版，你手上这块，丝印写着 `Luckfox Pico`）
> **你的器材**：32GB TF 卡 + 读卡器 + USB-TTL 串口模块 + 板子 + 摄像头
> **目标**：从"一张空卡"到"板子上跑通 YOLOv5 检测"再到"换成自己的装甲板模型"
> **总耗时**：约 2 周（每天 2–3 小时），其中下载和编译占大头
>
> 本文所有命令与参数均来自 Luckfox 官方文档（链接见第 16 节），已逐条核对。
> 官方没写明的推断，我会标 `【经验值】`。

---

RKNN = 瑞芯微 NPU 专用的“模型格式 + 软件栈”

NPU = 神经网络处理单元（Neural Processing Unit）

SDK = Software Development Kit，软件开发工具包/一句话：**一套“造东西的全套材料 + 工具 + 说明书”**。

Buildroot=**是一个"生成嵌入式 Linux 的工具"**（build + root）。

RTSP = Real Time Streaming Protocol，实时流传输协议

## 目录

| 章节 | 内容 | 预计耗时 |
|---|---|---|
| 0 | 先搞懂你在干什么（名词扫盲） | 10 分钟读 |
| 1 | 准备清单 | 下载 40 分钟 |
| 2 | **格式化 TF 卡** | 5 分钟 |
| 3 | **把系统写进 TF 卡** | 10 分钟 |
| 4 | **接串口** | 10 分钟 |
| 5 | **首次上电与登录** | 10 分钟 |
| 6 | **五项自检** | 5 分钟 |
| 7 | **TF 卡扩容（8G→28G）** | 5 分钟 |
| 8 | **摄像头出图（VLC）** | 15 分钟 |
| 9 | 装 SDK 并编译 | 1–3 小时（可挂机） |
| 10 | **跑官方 RKNN YOLOv5 demo** | 40 分钟 |
| 11 | 模型转换（ONNX→RKNN） | 30 分钟 |
| 12 | 换成你的装甲板模型 | 1 周 |
| 13 | 排错大全 | 随查随用 |
| 14 | 分阶段验收清单 | — |
| 15 | 两周计划表 | — |
| 16 | 官方链接全集 + 附录 | — |

---

# 0. 先搞懂你在干什么（10 分钟，别跳过）

你**从来没接触过嵌入式 Linux 板**的话，先看这一节，后面会顺很多。

## 0.1 你在做的事，本质是三件事

```
① 把"操作系统"写进一张 SD 卡      ← 相当于给新电脑装系统
② 板子从这张卡启动，通过串口登录    ← 相当于开机进桌面（但这板子是纯命令行）
③ 把你的程序（视觉识别）放上去跑    ← 相当于装软件
```

## 0.2 名词对照表（看不懂就来查）

| 名词 | 通俗解释 |
|---|---|
| **RV1103** | 板子上的芯片型号（瑞芯微出的，内置 CPU + NPU + 摄像头接口） |
| **NPU** | 专门算神经网络的小引擎（0.5 TOPS，只能跑小模型） |
| **镜像 / image** | 整个操作系统打包成的一个文件，写到卡里就能开机 |
| **烧录** | 把镜像"刻"进存储介质（这里是 TF 卡）的动作 |
| **分区 / partition** | 卡被切成几块，分别放引导程序、内核、根文件系统 |
| **根文件系统 / rootfs** | 存放所有程序、配置的那一块（相当于 Windows 的 C 盘） |
| **Buildroot** | 一个"生成精简 Linux 系统"的工具。**这一栏就是你要下的镜像类型** |
| **串口 / UART / TTL** | 一种最原始的通信方式，只要 3 根线。**板子起不来时唯一的救命通道** |
| **115200** | 串口的"语速"（波特率）。设错就全是乱码 |
| **ADB** | Android Debug Bridge，这里是板子通过 USB 提供的一个命令行通道 |
| **RNDIS** | 把板子的 USB 伪装成一张网卡，于是电脑能 `ssh` 过去 |
| **交叉编译** | 在电脑上编译，生成能在板子上跑的程序（因为板子太弱，编不动） |
| **ONNX** | 训练好的模型的通用交换格式 |
| **RKNN** | 瑞芯微 NPU 专用的模型格式（ONNX 转换而来） |
| **量化** | 把模型精度从 float32 降到 int8，变小变快（会掉一点精度） |
| **rkipc** | 板子自带的摄像头推流程序，开机自启 |
| **RTSP** | 一种视频流协议，用 VLC 能直接打开 |

## 0.3 关于你这块板，必须先知道的 5 件事

| # | 事实 | 后果 |
|---|---|---|
| 1 | **没有 SPI NAND Flash** | **必须靠 SD 卡启动**。卡没做好 = 板子完全没反应，不是坏了 |
| 2 | **总内存 64MB，摄像头占 24MB，实际可用约 35MB** | 看到 `free -m` 只有三十几 MB 是正常的，别以为坏了 |
| 3 | **默认分区只有 8GB** | 你 32GB 的卡不做扩容，22GB 是浪费的（第 7 章解决） |
| 4 | **没有板载 RJ45 网口**（只有 TXP/TXN/RXP/RXN 排针） | 别指望插网线。登录走 **USB(ADB/RNDIS)** 或 **串口** |
| 5 | **Debug 串口是 UART2**（不是 UART3） | 接线要找丝印 `UART2_TX`/`UART2_RX` |

## 0.4 一张总流程图

```
[电脑]                                [板子]
  |                                     |
  | ① 格式化 32GB 卡（用读卡器，板子不参与）
  |    SD Card Formatter
  |
  | ② 写系统镜像（还是用读卡器）
  |    SocToolKit → SDTool 模式
  |
  | ③ 卡从读卡器取出 → 插回板子
  |                                     |
  | ④ USB-TTL 接 UART2  ←──────────────┤ 串口
  |    115200                            |
  |                                     |
  | ⑤ 上电 → 看启动日志 → root/luckfox 登录
  |                                     |
  | ⑥ 扩容分区（U-Boot 里改）→ 8G 变 28G
  |                                     |
  | ⑦ VLC 打开 rtsp://172.32.0.93/live/0 ← 摄像头画面
  |                                     |
  | ⑧ 在 Ubuntu 上交叉编译 demo → adb push 到板子 → 跑起来
```

---

# 1. 准备清单

## 1.1 硬件

| 物品 | 状态 | 备注 |
|---|---|---|
| Luckfox Pico 板子 | ✅ 已有 | RV1103 |
| **32GB microSD 卡** | ✅ 已有 | SDHC，U1，正合适 |
| **读卡器** | ✅ 已有 | **写卡必需，不能用板子代替** |
| **USB-TTL 串口模块** | ✅ 已有 | CH340 / CP2102 都行 |
| 摄像头（MIPI CSI） | 你有 | RV1103 适配 **SC3336 3MP**；插错/插反不出图 |
| Type-C **数据线** | 必需 | 只能充电的线会让一切失败 |
| 打火机大小的螺丝刀 | 可选 | 有些板子需要松螺丝才能插排线 |

## 1.2 软件（3 个 + 1 个镜像）

| 软件 | 用途 | 去哪下 |
|---|---|---|
| `SDCardFormatterv5_WinEN_504.zip` | 格式化 TF 卡 | ✅ 你已下 |
| `SocToolKit_v1.98_..._win.zip` | 写卡工具 | 官方「镜像烧录」页 |
| `MobaXterm_Portable_v22.0.zip` | 串口/SSH 终端 | 官方「登录」页，或你习惯的串口助手 |
| **系统镜像** | 系统本体 | 见下方 ⬇️ |

### 要下的镜像（**唯一容易出错的地方**）

在官方「镜像烧录」页 → 「3. 镜像下载」→ 百度网盘。

下载 **Buildroot** 这一栏里、**Luckfox Pico** 对应的那个：

```
✅ Luckfox_Pico_MicroSD_<日期>        ← 就是这个
❌ Luckfox_Pico_Mini_A_MicroSD_...    （别的板型）
❌ Luckfox_Pico_Plus_MicroSD_...      （别的板型）
❌ Luckfox_Pico_*_Flash_...           （Flash 版是给 SPI NAND 用的，你的板子没有）
❌ Ubuntu_Luckfox_Pico_...            （Ubuntu 在 64MB 上跑不动）
```

> `<日期>` 形如 `250313`，**选最新的**。
> 下错版本的表现：写卡成功、上电后卡在 `Starting kernel...` 不动。

### 关于 DriverAssitant：**你这块板用不上**

很多人第一步就卡在这。原因：

| 操作 | 需要驱动吗 |
|---|---|
| **SD 卡路线（你的路线）** | ❌ **不需要**。SocToolKit 写的是**读卡器里的卡**，板子完全不参与 |
| USB 刷机路线（SPI NAND/eMMC） | ✅ 需要驱动 + 按住 BOOT 进 Maskrom |

**所以：SocToolKit 里看不到"板子设备"是正常的**，它找的是**盘符**。

> 你既然下好了 `DriverAssitant_v5.13.zip`，留着别删——以后万一换 Plus/Mini B 用得上。现在**不用装**。

---

# 2. 格式化 TF 卡（5 分钟）

## 2.1 为什么必须用读卡器

把卡插在板子上、板子再插 USB 到电脑，电脑看到的是「一个 Rockchip 芯片设备」，**不是 U 盘**，没有盘符。SocToolKit 的 SDTool 要找的是「USB 磁盘」。

**官方原文**：「将 SD 卡**插入读卡器**并连接至电脑，此时会在 **USB 磁盘**下会显示 SD 卡大小。」

## 2.2 操作

1. **板子断电**，把 32GB 卡从板子上抠出来；
2. 卡插进读卡器，读卡器插电脑；
3. ⚠️ **先拔掉电脑上其他 U 盘 / TF 卡**（防格错盘）；
4. 打开 **SD Card Formatter**；
5. `Select card` → 选中你的卡，**确认容量显示约 29–30 GB**；
6. `Formatting options` → 选 **Quick format**；
7. 点 `Format` → 确认。

### 你应当看到
格式化完成，文件系统显示 **FAT32**（32GB 卡默认就是 FAT32，正是我们要的）。

### ⚠️ 如果 Windows 弹"需要格式化才能使用"
**点取消。** 不要用 Windows 自带格式化。

---

# 3. 把系统写进 TF 卡（10 分钟）

> 这一步是全程最容易出错的地方，按顺序做。

## 3.1 解压镜像包

解压后会看到一个目录，里面是若干分区镜像文件，形如：

```
env.img
idblock.img
uboot.img
boot.img
rootfs.img
userdata.img
update.img     ← 注意这个
```

> 【经验值】不同版本的官方包里，可能是**目录**也可能是**整卡镜像**，以解压出来的实际内容为准。
> 官方**没有**指定启动文件必须包含哪些具体文件，只明确了一条：「**启动文件不包括 update.img**」。

## 3.2 打开 SocToolKit（必须管理员）

**右键 → 以管理员身份运行**。

普通权限打不开 USB 磁盘，会表现为"看不到卡"。

## 3.3 选芯片：**RV1106**

工具里没有 RV1103 这个选项，**RV1103 也选 RV1106**（同一个系列）。

## 3.4 切到 SDTool 模式

工具上有两种模式，**别选错**：

| 模式 | 写哪里 | 你的情况 |
|---|---|---|
| **SDTool** | 读卡器里的 SD 卡 | ✅ **选这个** |
| Firmware / eMMC/Flash | 板子里的 SPI NAND | ❌ 你的板子没有 |

## 3.5 官方五步（照做，顺序不能乱）

| 步骤 | 你做什么 | 你应当看到 |
|---|---|---|
| **①** | 切换到 `SDTool` 模式 | 界面变成 SD 卡工具的样子 |
| **②** | 把读卡器插上电脑 | 「USB 磁盘」区域显示 **SD 卡容量** |
| **③** | 选择「**SD 卡启动**」 | 该选项被选中 |
| **④** | 导入「**启动文件**」 | 选中 3.1 解压出来的目录 |
| **⑤** | 点「**创建 SD 卡**」 | 进度条开始走 |

## 3.6 关于「启动文件」的两个坑

**坑 1：`update.img`**

官方警告明确说「**启动文件不包括 update.img**」。如果工具报错：

- 把 `update.img` **剪切到另一个文件夹**，再选目录；
- 或者工具让你逐个选文件时，**跳过 `update.img`**。

**坑 2：目录里有子文件夹**

有些包解压后多一层（`Luckfox_Pico_MicroSD_250313/Luckfox_Pico_MicroSD_250313/…`）。选**真正放着 .img 文件的那一层**。

## 3.7 等待完成

32GB 卡通常 **2–5 分钟**。

> ⚠️ **中途绝对不要拔卡、不要断电。** 写一半断 = 卡变砖，得从第 2 步重来。

## 3.8 写完后的正常现象

把读卡器插回电脑，Windows 可能弹：

> "需要格式化才能使用这张磁盘吗？"

**点「取消」。** 因为里面刚写进去的是 Linux 分区表，Windows 不认识。**你要是点了格式化，前面全白做。**

---

# 4. 接串口（10 分钟）

## 4.1 为什么要串口（而不只是 USB）

- 板子能启动 → USB(ADB) 也能进；
- **但板子启动失败时，只有串口能看到卡在哪一步**。

对第一次玩的人来说，串口是"眼睛"。**强烈建议接上。**

## 4.2 找针脚

这块板**调试串口是 UART2**（官方：RV1103 有 UART2/3/4，**UART2 为调试串口**）。

在板子上找丝印：

```
UART2_TX
UART2_RX
GND
```

> 官方引脚图（Luckfox Pico 专用）：
> `https://wiki.luckfox.com/zh/assets/images/LUCKFOX-PICO-GPIO-ee9379565038f7ffd81a635a4ce29f33.webp`
> 打开对照着找。
>
> 注意：官方 UART 教程里举例用的「引脚 18/19/20（UART3）」是**做回环测试**用的，**不是调试口**，别接错。

## 4.3 接线（TX/RX 必须交叉）

```
USB-TTL 模块            Luckfox Pico
   TX      ────────►    UART2_RX
   RX      ◄────────    UART2_TX
  GND      ────────     GND
  VCC      ✗ 不要接    （板子用 USB 自己供电）
```

**三根线，注意三点：**

1. **TX 接 RX，RX 接 TX**（交叉）——接反不会烧，但收不到任何数据；
2. **GND 必须接**——不共地会出乱码；
3. **VCC 绝对不要接**——会和板子的 USB 供电打架。

## 4.4 打开串口终端（MobaXterm）

1. 插上 USB-TTL，打开**设备管理器**，记下它占用的 **COM 口**（如 `COM5`）；
2. MobaXterm → `Session` → `Serial`；
3. 填参数：

| 参数 | 值 |
|---|---|
| Serial port | 你看到的 COM 口 |
| **Baud rate** | **115200** ⚠️ 错一个数字就是乱码 |
| Data bits | 8 |
| Parity | None |
| Stop bits | 1 |
| Flow control | None |

4. 点 OK。

**此时板子还没上电，终端里应该什么都没有。**

---

# 5. 首次上电与登录（10 分钟）

## 5.1 上电顺序（重要）

```
① 卡从读卡器取出 → 插进板子的 MicroSD 槽（听到"咔"一声到位）
② 串口线接好（TX/RX 交叉，GND）
③ 摄像头排线插好（见第 8 章，蓝色面朝芯片）
④ 最后才插 USB 给板子上电
```

> 顺序反了容易在启动一瞬间受干扰。**先接好所有东西，最后通电。**

## 5.2 你会看到什么（逐段解读）

```
U-Boot 2017.09 ...            ← 引导程序启动（这一段说明卡被认出来了）
...
Starting kernel ...           ← 内核开始加载
[    0.000000] Booting Linux on physical CPU 0x0
[    0.xxxxxx] Memory: xxxxxK/xxxxxK available ...   ← 内存情况（约 64MB 总，30 多 MB 可用）
[    0.xxxxxx] ... mmc0: ...                          ← SD 卡被识别
...
[    3.xxxxxx] ...                              ← 各种驱动初始化
...
luckfox login:                ← ✅ 成功！到这一步就对了
```

**首次启动慢一点是正常的**【经验值 30–60 秒】。

### 卡在哪一段，说明什么问题

| 停在这里 | 原因 |
|---|---|
| 串口一个字都没有 | 接线错（TX/RX/GND）、波特率错、板子没供电 |
| 全是乱码 | 波特率不是 115200 |
| `Starting kernel...` 之后没了 | **镜像板型不对**（下成 Flash 版/别的型号） |
| 反复重启 | USB 供电不足 |

## 5.3 登录

```
luckfox login: root
Password: luckfox
```

> **输入密码时屏幕不会有任何显示**（连 `*` 都没有），这是 Linux 的正常行为，输完直接回车。

### 登录成功后长这样

```
# _
```
（`#` 表示你是 root 用户）

## 5.4 另外两种登录方式（串口不通时的备选）

### 方式 A：ADB（USB 直连，最简单）

电脑上装 adb（官方「登录」页有下载），然后：

```powershell
adb devices         # 应该看到一行设备
adb shell           # 进去
```

### 方式 B：SSH（USB 虚拟网卡 RNDIS）

1. 板子上电后，电脑「网络适配器」里会多出一个 **`Remote NDIS based Internet Sharing Device`**；
2. 双击它 → IPv4 → 手动设置：
   - IP：`172.32.0.100`
   - 掩码：`255.255.255.0`
3. 关闭 Windows 防火墙（RNDIS 常被拦）；
4. `ssh root@172.32.0.93`

板子的静态 IP **固定是 `172.32.0.93`**（官方）。

> ⚠️ **你这块板没有 RJ45 网口**，所以 `ifconfig` 里看不到 `eth0` 的局域网 IP，只有 `usb0`。别以为网口坏了。

---

# 6. 五项自检（5 分钟）

登录后逐条敲，**看清预期值**（这里有两个坑，官方数据和我上一版不同）：

```bash
# ① 系统版本
uname -a
# 期望：Linux luckfox 5.10.x ... armv7l GNU/Linux

# ② 内存 —— ⚠️ 坑 1：数字比你想象的小
free -m
cat /proc/meminfo | head -3
```

**官方数据**（RV1103 全系）：

| 项目 | 数值 |
|---|---|
| 总运行内存 | 64 MB |
| 摄像头 CMA 预留 | **24 MB** |
| **实际可用** | **约 35 MB** |

> 所以 `free -m` 看到 total 五十几 MB、可用三十几 MB **完全正常**，不是板子坏了。

```bash
# ③ 存储 —— ⚠️ 坑 2：分区只有 8G 是正常的
lsblk
# 期望：mmcblk0 的总 SIZE ≈ 28-30G  ← 这个证明卡是真的 32GB
df -h
# 期望：根分区约 8G  ← 这是官方默认，第 7 章扩它
```

**怎么区分"分区小"和"假卡"：**

| 现象 | 含义 |
|---|---|
| `lsblk` 里 mmcblk0 ≈ 28–30G，`df -h` 只有 8G | ✅ **正常**，去做第 7 章扩容 |
| `lsblk` 里 mmcblk0 只有几 G | ❌ 真·扩容卡，退货 |

```bash
# ④ NPU 驱动
dmesg | grep -i rknpu
# 期望：有 rknpu 相关输出

# ⑤ 摄像头是否识别（插了摄像头才有效）
ls /userdata
# 期望：能看到 rkipc.ini、video0、video1、video2 等
# 官方说明：识别成功会生成 rkipc.ini 文件
```

---

# 7. TF 卡扩容：8GB → 28GB（5 分钟）

> **这一步很容易被漏掉，但不做的话你 32GB 卡只能用 8GB。**

## 7.1 为什么

**官方原文**：

> 「默认情况下，使用官方提供的 Buildroot 镜像烧录至 TF 卡后，即便 TF 卡容量为 16GB，**系统可用空间也仅为约 8GB**。这是因为当前系统镜像**并不支持根据 TF 卡容量自动扩展分区**，镜像中的分区大小由 SDK 构建配置决定，**默认设定为 8GB**。」

## 7.2 怎么做（U-Boot 里改，不用重编镜像）

**方法一：改分区表（推荐）**

1. 串口连好，**重启板子**（拔插 USB 或输入 `reboot`）；
2. 启动过程中**一直按 `Ctrl + C`**，打断启动，进入 U-Boot；
3. 看到提示符 `=>` 就对了；
4. 逐条输入（**一行一行来，别一次粘贴三行**）：

```bash
setenv blkdevparts 'mmcblk1:32K(env),512K@32K(idblock),256K(uboot),32M(boot),512M(oem),256M(userdata),-(rootfs)'
```
```bash
saveenv
```
```bash
reset
```

5. 板子重启后登录，验证：

```bash
df -h
# 期望：根分区从 ~8G 变成 ~28G
```

### ⚠️ 设备名要对

官方示例用的是 `mmcblk1`（SD 卡）。**如果你的卡是 `mmcblk0`，就要把命令里的 `mmcblk1` 改成 `mmcblk0`。**

先查：

```bash
lsblk
```

看 `mmcblkX` 哪个下面挂着 `boot`/`rootfs` 分区——那个 X 就是你要填的数字。

### 命令在讲什么

```
mmcblk1:                                  ← 卡设备名
  32K(env),                               ← 引导环境
  512K@32K(idblock),                      ← 从偏移 32K 开始放 idblock
  256K(uboot),                            ← 引导程序
  32M(boot),                              ← 内核
  512M(oem),                              ← 厂商分区
  256M(userdata),                         ← 用户数据
  -(rootfs)                               ← 【关键】"-" = 吃掉剩余全部空间
```

**只有最后一段是破折号**，表示"剩下的都给 rootfs"。

**方法二：改 SDK 重编（等你要重新编译时再考虑）**

改 `luckfox-pico/project/cfg/BoardConfig_IPC/<板型>.mk` 里的 `RK_PARTITION_CMD_IN_ENV`，把 rootfs 改成 `-(rootfs)`，再 `./build.sh`。

---

# 8. 摄像头出图（15 分钟）

## 8.1 先装好排线（这一步错了后面全白搭）

**官方原文的三条铁律**：

1. **先打开 CSI 排线的黑色卡扣**，且「应**双侧同时均匀用力缓慢打开**，避免单侧用力过大，否则容易损坏卡扣」；
2. **摄像头排线的蓝色面朝向开发板芯片**；
3. 插到底 → 按下卡扣锁紧。

> 插反不会烧板子，但一定不出图。这是新手最常见的失败原因。

## 8.2 检查摄像头是否被识别

```bash
# ① 看 I2C 上有没有摄像头（官方排查第一步）
i2cdetect -y 4
# 期望：某个地址（常见 0x30 / 0x36 之类）不是 "--"，说明 sensor 挂上了

# ② 看设备节点
v4l2-ctl --list-devices
# 期望：能看到 rkisp_mainpath (platform:rkisp-vir0) 下面挂着一堆 /dev/videoX
#       其中一个是 CSI 主通路（官方示例里是 /dev/video15，你的可能不同）

# ③ 看 /userdata 有没有 rkipc.ini
ls /userdata
# 期望：rkipc.ini 存在 = 摄像头识别成功
```

## 8.3 看实时画面（VLC，最简单）

`rkipc` 是开机自启的推流程序。确认它在跑：

```bash
ps | grep rkipc
```

电脑上打开 **VLC** → `媒体` → `打开网络串流` → 输入：

```
rtsp://172.32.0.93/live/0
```

点播放。

### 如果 VLC 打不开（大概率是网络没通）

先解决电脑和板子的连通性：

1. 电脑 RNDIS 网卡设 `172.32.0.100`；
2. 关掉 Windows 防火墙；
3. 在电脑上 `ping 172.32.0.93`，**能 ping 通才谈推流**（官方排查步骤明确写了这一条）。

### 降低延迟（可选）

VLC 默认缓存 1 秒。想更实时：
`工具` → `首选项` → `输入/编解码器` → 网络缓存改 **300 ms**（官方建议不低于 300ms，再低会卡顿丢包）。

## 8.4 用 v4l2 抓原始图（进阶，调试用）

⚠️ **抓之前必须先停掉 rkipc**，否则摄像头被占用：

```bash
killall rkipc            # 或官方推荐的 RkLunch-stop.sh
```

然后抓 30 帧：

```bash
v4l2-ctl --device=/dev/video15 \
  --set-fmt-video=width=640,height=480,pixelformat=NV12 \
  --stream-mmap --stream-to=video50.yuv --stream-count=30
```

看到 `<<<<<<<<< 25 fps` 就说明抓成功了。拷回电脑用 ffplay 看：

```bash
ffplay -video_size 640x480 -pixel_format nv12 -framerate 10 -i video50.yuv
```

> 注意：v4l2 抓的是**未经 ISP 处理**的原始数据，画面会**发暗、偏绿**，这是官方明确说明的正常现象。想要正常画面就走 rkipc/RTSP。

---

# 9. 装 SDK 并编译（1–3 小时，可挂机）

## 9.1 前置条件

| 项目 | 要求 |
|---|---|
| 系统 | **Ubuntu 22.04 x86_64**（物理机 / 双系统 / **WSL2** 都行） |
| 磁盘 | **≥ 60 GB** 空闲 |
| 内存 | ≥ 8 GB |
| 网络 | 能访问 Gitee / GitHub |

> ⚠️ 官方明确只支持 Ubuntu 22.04 x86_64。用 20.04 或其他架构会遇到一连串玄学编译错误。

## 9.2 装依赖

```bash
sudo apt update
sudo apt-get install -y git ssh make gcc gcc-multilib g++-multilib \
  module-assistant expect g++ gawk texinfo libssl-dev bison flex fakeroot \
  cmake unzip gperf autoconf device-tree-compiler libncurses5-dev pkg-config \
  bc python-is-python3 passwd openssl openssh-server openssh-client vim file \
  cpio rsync curl
```

## 9.3 拉 SDK

```bash
# 国内用 Gitee 快；--recursive 千万不能漏（子模块很多）
git clone --recursive https://gitee.com/LuckfoxTECH/luckfox-pico.git
cd luckfox-pico
```

> 仓库有几个 GB，网速慢的话先挂着去干别的。

## 9.4 选板型

```bash
./build.sh lunch
```

在交互菜单里选 **RV1103 + 你的板型 + Buildroot**。对应关系：

| 你的板子 | 选哪个 |
|---|---|
| **Luckfox Pico** | `SD_CARD-Buildroot-RV1103_Luckfox_Pico-IPC` |
| Pico Mini A/B | `...RV1103_Luckfox_Pico_Mini-IPC` |
| Pico Plus | `...RV1103_Luckfox_Pico_Plus-IPC` |

> 菜单里名字可能略有差异，**认准 `RV1103_Luckfox_Pico` 这一串**。

## 9.5 全量编译

```bash
./build.sh
```

**首次 1–3 小时**（看 CPU）。产物在：

```
output/image/
├── update.img      ← 整包
├── boot.img  rootfs.img  uboot.img  env.img  idblock.img  userdata.img
```

### 日常单编单项（省时间）

```bash
./build.sh clean kernel && ./build.sh kernel
./build.sh clean uboot  && ./build.sh uboot
./build.sh clean rootfs && ./build.sh rootfs
./build.sh firmware        # 只重新打包
```

## 9.6 两个必踩的坑

**坑 1：WSL2 报 `Your PATH contains spaces, TABs, and/or newline`**

```bash
export PATH=$(echo "$PATH" | tr -d ' \t\n')
```

加进 `~/.bashrc` 一劳永逸。

**坑 2：Buildroot 下载软件包失败**

官方提供离线 `dl` 包（百度网盘），下载后替换到：

```
luckfox-pico/sysdrv/source/buildroot/buildroot-2023.02.6/dl
```

> 也**不要在编译过程中乱用 `sudo`**，会导致文件权限变化让编译失败（官方明确提醒）。

---

# 10. 跑官方 RKNN YOLOv5 demo（40 分钟）

> 这一步的目的：**先证明"摄像头 → NPU → 检测框"这条链路是通的**，再换成你自己的模型。
> 官方 demo 仓库里**已经放好了转换好的 `.rknn` 模型**，你不需要先训练就能跑。

仓库：`https://github.com/LuckfoxTECH/luckfox_pico_rknn_example`

## 10.1 设置 SDK 路径并下载 demo

```bash
# 必须绝对路径！
export LUCKFOX_SDK_PATH=/home/<你的用户名>/luckfox-pico

git clone https://github.com/LuckfoxTECH/luckfox_pico_rknn_example.git
cd luckfox_pico_rknn_example
```

## 10.2 编译

```bash
chmod a+x ./build.sh
./build.sh
```

会依次问你三个问题：

```
1) uclibc            ← 选 1（Buildroot 用的是 uclibc）
2) glibc

1) luckfox_pico_retinaface_facenet
2) luckfox_pico_retinaface_facenet_spidev
3) luckfox_pico_yolov5        ← 选 3

（只有选 spidev 时才会问 LUCKFOX_PICO_PLUS / PRO_MAX）
```

编译完成后在 `install/` 下得到：

```
install/luckfox_pico_yolov5_demo/
├── luckfox_pico_yolov5        ← 可执行文件
└── model/yolov5.rknn          ← 已有模型，直接能用
```

## 10.3 传到板子上

```bash
# 电脑端（板子已用 USB 连着）
adb push install/luckfox_pico_yolov5_demo /root/
```

> 也可以用 MobaXterm 的 SFTP 面板直接拖文件，或 `scp -r`。

## 10.4 在板子上跑

```bash
cd /root/luckfox_pico_yolov5_demo
chmod a+x luckfox_pico_yolov5

# ⚠️ 关键一步：先释放摄像头（rkipc 占着）
RkLunch-stop.sh

./luckfox_pico_yolov5 ./model/yolov5.rknn
```

## 10.5 你应当看到什么

- **接了屏**（如 Pico-1.3-LCD）→ 屏幕上实时框出物体；
- **没接屏** → **检测结果直接打印在串口终端里**（官方明确支持），内容是类别 + 置信度 + 框坐标。

### ✅ 本步验收
把一张有公交车的图（或对着屏幕）给它看，能持续打印出 `bus` 之类的检测结果，**帧率 ≥5 fps 就算通**。

---

# 11. 模型转换：ONNX → RKNN（30 分钟）

后面换成自己的模型必须会这一步。

## 11.1 PC 端装 RKNN-Toolkit2

```bash
# 建议用 conda 隔离环境
conda create -n RKNN-Toolkit2 python=3.8
conda activate RKNN-Toolkit2

git clone https://github.com/airockchip/rknn-toolkit2.git
cd rknn-toolkit2

# 装依赖（cp38 对应 python3.8；慢就换镜像源）
pip install -r packages/x86_64/requirements_cp38-2.3.2.txt \
  -i https://pypi.mirrors.ustc.edu.cn/simple/

# 装 whl（文件名里的版本号和 cpXX 要和你环境对应）
pip install packages/x86_64/rknn_toolkit2-2.3.2-cp38-cp38-manylinux_2_17_x86_64.manylinux2014_x86_64.whl

# 验证
python -c "from rknn.api import RKNN; print('OK')"
```

> ⚠️ **最容易踩的坑**：`rknn-toolkit2` 的版本必须和**板端 runtime 版本**匹配。
> 模型在板子上加载失败时，**第一件事就是核对这两边版本号**，别怀疑模型结构。

## 11.2 导出 ONNX（用 Rockchip 版 YOLOv5）

```bash
git clone https://github.com/airockchip/yolov5.git
cd yolov5
conda create -n yolov5 python=3.9 && conda activate yolov5
pip install -r requirements.txt

# ⚠️ 必须带 --rknpu
python export.py --rknpu --weight yolov5s.pt
```

**为什么要 `--rknpu`**：标准 YOLOv5 的后处理（结果提取）用了 RKNPU 不支持的算子，加了 `--rknpu` 会把它去掉、交给 CPU 做。
**漏了这个参数 = 检测结果全乱。**

## 11.3 转成 RKNN

```bash
cd luckfox_pico_rknn_example/scripts/luckfox_onnx_to_rknn

# convert.py <onnx> <dataset.txt> <输出.rknn> <模型类型>
python convert.py ../model/yolov5.onnx ../dataset/yolov5_dataset.txt ../model/yolov5.rknn Yolov5
```

`dataset.txt` 是**量化校准图**的路径列表（10–50 张）。

## 11.4 先在 PC 上仿真验证（别急着上板）

```bash
cd sim
python yolov5.py
```

结果正常再上板，能省很多来回。

---

# 12. 换成你们的装甲板模型（1 周，真正的目标）

## 12.1 现实检查：先接受 RV1103 的上限

| 项目 | 数值 | 含义 |
|---|---|---|
| NPU | **0.5 TOPS** | 只能跑 nano 级模型 |
| **可用内存** | **约 35 MB** | **这是最硬的约束** |
| 模型建议 | **YOLOv5n @ 320×320** | |
| 预期帧率 | 15–25 fps【经验值】 | 416 输入会明显掉 |

**结论：RV1103 的角色是"验证链路"，不是"最终方案"。** 要 640 分辨率 + 高帧率，得换 RV1106（256MB）或 RK3566/RK3588。

## 12.2 数据集

| 要点 | 建议 |
|---|---|
| 数量 | 2000–3000 张起 |
| 场景 | **必须包含 RM 场地实拍**（往届比赛视频截图非常有用） |
| 光照 | 场馆灯、逆光、反光地胶、观众走动 |
| 类别 | **先只标"装甲板"一类**，别一上来多类 |
| 增强 | 亮度/对比度抖动、运动模糊、随机裁剪 |

## 12.3 训练 → 导出 → 转换 → 上板

```bash
# 1) 训练
python train.py --img 320 --batch 16 --epochs 100 \
  --data armor.yaml --weights yolov5n.pt

# 2) 导出（务必 --rknpu）
python export.py --rknpu --weight runs/train/exp/weights/best.pt

# 3) 转 RKNN（校准集要用真实场地图）
python convert.py best.onnx ./dataset/armor_dataset.txt ./armor.rknn Yolov5

# 4) 上板
adb push armor.rknn /root/luckfox_pico_yolov5_demo/model/
# 板子上：
RkLunch-stop.sh
./luckfox_pico_yolov5 ./model/armor.rknn
```

## 12.4 下一步：把结果吐出来（最重要的里程碑）

**先别急着做闭环控制。** 先做这一件事：

> **把检测框坐标通过串口打印出来**（`x, y, w, h` + 置信度）

这一步做成了，才意味着板子真正"看见"了目标，后面才有资格谈：
- 单目 PnP 测距（用 RM 装甲板标准尺寸反推距离）
- MAVLink 发给飞控（ArduPilot `GUIDED`）
- 闭环撞击

---

# 13. 排错大全（按阶段分组）

## 13.1 阶段一：写卡

| 现象 | 原因 | 解决 |
|---|---|---|
| SocToolKit 看不到 SD 卡 | 没用管理员运行 | 右键 → 以管理员身份运行 |
| 看不到任何设备 | **正常**（SD 路线不需要板子） | 确认你切到了 **SDTool** 模式，看的是「USB 磁盘」 |
| 写卡中途失败 | 扩容卡 / 读卡器接触不良 / HUB | 换直连 USB 口；用 H2testw 验卡 |
| 提示「启动文件」不合法 | 目录里混了 `update.img` | 把它挪走再选目录 |
| 写完 Windows 提示格式化 | 正常（Linux 分区） | **点取消** |

## 13.2 阶段二：上电与登录（含官方 FAQ 前四条）

| 现象 | 原因 | 解决 |
|---|---|---|
| **串口一个字都没有** | ① TX/RX 没交叉 ② 没共地 ③ 波特率错 ④ 没供电 | 逐条查 |
| **串口全乱码** | 波特率不是 115200 | 改 115200 |
| **卡在 `Starting kernel...`** | 镜像板型不对 | 重下 `Luckfox_Pico_MicroSD_*` |
| 反复重启 | USB 供电不足 | 换**台式机后置 USB 口**，别用前面板 / HUB |
| `adb devices` 看不到设备 | **数据线只支持充电**（官方：极为重要） | 换原装线/品牌数据线 |
| 同上 | 杀毒软件拦截 | 临时关闭杀软 |
| 同上 | 虚拟机抢占 USB | 用物理机，或虚拟机里手动挂载 |

## 13.3 阶段三：系统校验

| 现象 | 原因 | 解决 |
|---|---|---|
| `free -m` 只有 35MB 可用 | **正常**（24MB 给摄像头） | 无需处理；不用摄像头可在 BoardConfig 里把预留改小后重编 |
| `df -h` 根分区只有 8G | **正常**（默认分区） | 做第 7 章扩容 |
| `lsblk` 总容量远小于 32G | 真·扩容卡 | 换卡 |
| 想改用笔记本 SSH，连不上 | RNDIS 未配置 / 防火墙 | 设 `172.32.0.100`、关防火墙、先 ping |

## 13.4 阶段四：摄像头

| 现象 | 原因 | 解决 |
|---|---|---|
| `ls /userdata` 没有 `rkipc.ini` | 摄像头没插好 | 重插，**蓝色面朝芯片**，卡扣双侧均匀用力 |
| `i2cdetect -y 4` 全是 `--` | sensor 没挂上 | 排线方向/接触；确认是 SC3336 |
| VLC 打不开 RTSP | 网络不通 | 先 `ping 172.32.0.93`；关防火墙 |
| 自己的程序报 `VIDIOC_STREAMON` 失败 | **rkipc 占着摄像头** | `RkLunch-stop.sh` 或 `killall rkipc` |
| 画面发暗发绿 | 走的是 v4l2 原始数据 | 正常，用 RTSP 路线 |
| 找不到 `/dev/video15` | 编号因板型而异 | `v4l2-ctl --list-devices` 查实际编号 |

## 13.5 阶段五：SDK / RKNN

| 现象 | 原因 | 解决 |
|---|---|---|
| 编译报 PATH 空格错误 | WSL2 默认 PATH | `export PATH=$(echo "$PATH" | tr -d ' \t\n')` |
| Buildroot 下载包失败 | 网络 | 用官方 `dl` 离线包替换 |
| 编译中途权限错误 | 用了 `sudo` | 别用 sudo，清掉重来 |
| demo 编译找不到工具链 | `LUCKFOX_SDK_PATH` 不是绝对路径 | 用绝对路径 |
| `.rknn` 加载失败 | toolkit2 与板端 runtime 版本不符 | 核对版本号 |
| 检测框全乱 | 导出 ONNX 漏了 `--rknpu` | 重新导出 |
| 程序被 kill | **35MB 内存 OOM** | 换更小模型/更小输入；关掉无关进程 |

---

# 14. 分阶段验收清单

## Day 1：能写卡、能开机
- [ ] 用**读卡器**格式化成功，FAT32
- [ ] SocToolKit（管理员 + RV1106 + SDTool）写卡 100%
- [ ] 卡插回板子，串口 115200 能看到完整启动日志
- [ ] `root` / `luckfox` 登录成功
- [ ] 看到 `#` 提示符

## Day 2：系统是健康的
- [ ] `uname -a` 有输出
- [ ] `free -m` 看到约 35MB 可用（知道这是正常的）
- [ ] `lsblk` 里 mmcblk0 ≈ 28–30G
- [ ] `dmesg | grep rknpu` 有输出

## Day 3：空间用满 + 摄像头出图
- [ ] 第 7 章扩容完成，`df -h` 根分区 ~28G
- [ ] `i2cdetect -y 4` 能看到 sensor
- [ ] `ls /userdata` 有 `rkipc.ini`
- [ ] VLC 打开 `rtsp://172.32.0.93/live/0` 有画面

## Day 4：备份卡（很重要）
- [ ] 用 Win32DiskImager 点 **Read** 备份成 `.img`
- [ ] 备份文件能重新写回另一张卡并开机

## Day 5–7：SDK 编出来
- [ ] Ubuntu 22.04 环境就绪，磁盘 ≥60G
- [ ] `./build.sh lunch` 选对 `RV1103_Luckfox_Pico`
- [ ] `./build.sh` 出 `output/image/`

## Day 8–10：demo 跑通
- [ ] `export LUCKFOX_SDK_PATH` 设好
- [ ] `./build.sh`（demo）编译成功
- [ ] 板子上 `./luckfox_pico_yolov5 ./model/yolov5.rknn` 有检测输出
- [ ] 帧率 ≥5 fps

## Day 11–14：自己的模型
- [ ] RKNN-Toolkit2 装好，`from rknn.api import RKNN` 无报错
- [ ] 自己的 ONNX 用 `--rknpu` 导出
- [ ] `convert.py` 出 `.rknn`，PC 仿真正常
- [ ] 板端跑自己的模型 ≥10 fps
- [ ] **检测框坐标能从串口打印出来** ← 里程碑

---

# 15. 两周计划表

| 时间 | 目标 | 风险点 |
|---|---|---|
| 第 1 天 | 写卡 → 开机 → 登录 | 镜像下错版本 |
| 第 2 天 | 五项自检 + 扩容 | 忘了扩容，后面空间不够 |
| 第 3 天 | 摄像头 + VLC | 排线方向 / 蓝色面 |
| 第 4 天 | 备份卡 | — |
| 第 5–7 天 | 装 Ubuntu + SDK 编译（挂机） | 磁盘不够 / 网络慢 |
| 第 8–10 天 | demo 跑通 | 忘记 `RkLunch-stop.sh` |
| 第 11–14 天 | 换自己的模型 + 串口输出 | 内存不足 / 忘 `--rknpu` |

---

# 16. 官方链接全集

## 必备

| 内容 | 链接 |
|---|---|
| **文档首页** | https://wiki.luckfox.com/zh/ |
| 版型介绍（RV1103 五款对比） | https://wiki.luckfox.com/zh/Luckfox-Pico/Luckfox-Pico-RV1103 |
| **镜像烧录**（含 SD 卡五步 + 工具下载） | https://wiki.luckfox.com/zh/Luckfox-Pico-Plus-Mini/Flash-image |
| **登录**（ADB / SSH / 串口） | https://wiki.luckfox.com/zh/Luckfox-Pico-Plus-Mini/Login |
| **SDK 镜像编译** | https://wiki.luckfox.com/zh/Luckfox-Pico-Plus-Mini/SDK-Image-Compilation |
| **RKNN 教程** | https://wiki.luckfox.com/zh/Luckfox-Pico-Plus-Mini/RKNN |
| **CSI 摄像头**（排线方向 + VLC + v4l2） | https://wiki.luckfox.com/zh/Luckfox-Pico-Plus-Mini/CSI-Camera |
| UART（串口引脚说明） | https://wiki.luckfox.com/zh/Luckfox-Pico-Plus-Mini/UART |
| 交叉编译 | https://wiki.luckfox.com/zh/Luckfox-Pico-Plus-Mini/Cross-Compile |
| 自启动 / 静态 IP | https://wiki.luckfox.com/zh/Luckfox-Pico-Plus-Mini/Autostart |
| luckfox-config | https://wiki.luckfox.com/zh/Luckfox-Pico-Plus-Mini/Luckfox-config |

## 常见问题（**遇到问题先看这里**）

| 专题 | 链接 |
|---|---|
| 常见问题总览 | https://wiki.luckfox.com/zh/Luckfox-Pico-RV1106/FAQ |
| **设备识别与镜像烧录**（线材/杀软/USB口/HUB） | https://wiki.luckfox.com/zh/Luckfox-Pico-RV1106/FAQ/Flashing |
| **摄像头连接与排查**（i2cdetect / 线序图） | https://wiki.luckfox.com/zh/Luckfox-Pico-RV1106/FAQ/Camera |
| **内存与存储管理**（内存分配 / TF 卡扩容） | https://wiki.luckfox.com/zh/Luckfox-Pico-RV1106/FAQ/Memory-Storage |
| 供电与电源 | https://wiki.luckfox.com/zh/Luckfox-Pico-RV1106/FAQ/Power |
| 系统登录与定制 | https://wiki.luckfox.com/zh/Luckfox-Pico-RV1106/FAQ/System-Customization |
| USB 连接与识别 | https://wiki.luckfox.com/zh/Luckfox-Pico-RV1106/FAQ/USB |
| **RKNN 相关问题** | https://wiki.luckfox.com/zh/Luckfox-Pico-RV1106/FAQ/RKNN |
| 其他接口与外设（串口乱码等） | https://wiki.luckfox.com/zh/Luckfox-Pico-RV1106/FAQ/Peripherals |

## 代码仓库

| 仓库 | 链接 |
|---|---|
| SDK | https://gitee.com/LuckfoxTECH/luckfox-pico |
| SDK（GitHub 镜像） | https://github.com/LuckfoxTECH/luckfox-pico |
| **RKNN demo** | https://github.com/LuckfoxTECH/luckfox_pico_rknn_example |
| RKNN-Toolkit2 | https://github.com/airockchip/rknn-toolkit2 |
| Rockchip 版 YOLOv5 | https://github.com/airockchip/yolov5 |
| 模型库 | https://github.com/airockchip/rknn_model_zoo |

---

# 附录 A：常用命令速查（登录后）

```bash
# —— 系统信息 ——
uname -a                      # 内核版本
free -m                       # 内存（可用约 35MB）
lsblk                         # 块设备（看卡真实容量）
df -h                         # 分区使用（扩容后应 ~28G）
cat /proc/device-tree/model   # 板型

# —— 摄像头 ——
ps | grep rkipc               # rkipc 在不在跑
RkLunch-stop.sh               # 【跑自己程序前必做】
killall rkipc                 # 同上（手动版）
i2cdetect -y 4                # 摄像头是否挂在 I2C 上
v4l2-ctl --list-devices       # 摄像头设备节点
ls /userdata                  # 有没有 rkipc.ini

# —— 网络 ——
ifconfig                      # usb0 = 172.32.0.93
ping <电脑IP>                 # 连通性

# —— 运行 demo ——
cd /root/luckfox_pico_yolov5_demo
chmod a+x luckfox_pico_yolov5
RkLunch-stop.sh
./luckfox_pico_yolov5 ./model/yolov5.rknn

# —— 关机/重启（别直接拔电） ——
reboot
poweroff
```

# 附录 B：你这块板的三个"反直觉"事实

| 反直觉 | 真相 | 为什么会误判 |
|---|---|---|
| "64MB 内存，怎么只剩 35MB？" | 24MB 给摄像头 CMA 预留了 | 以为内存坏了 |
| "32GB 卡怎么只有 8GB？" | 官方镜像默认分区 8GB，不自动扩 | 以为买到了假卡 |
| "SDTool 里怎么看不到板子？" | SD 路线写的是读卡器，跟板子无关 | 以为工具/驱动有问题 |

**记住这三条，能省你两天。**

---

> **最后一句**：这套流程里，真正会卡住新手的从来不是技术难点，而是三个"看起来像故障其实很正常"的现象（上面附录 B）和两个操作顺序（用读卡器而不是板子、跑程序前先 `RkLunch-stop.sh`）。
> 把这三条记住，剩下的都是耐心问题。
