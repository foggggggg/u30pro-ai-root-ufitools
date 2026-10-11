# 中兴 U30 Pro AI（MU5358）Root + UFI-TOOLS 安装指南

> **适用设备**：ZTE U30 Pro **AI 版**（型号 MU5358 ｜ Android 15 ｜ 展锐平台 ｜ 活动槽 A ｜ 固件线 `U30Pro_AIV1.0.0B05`）
> **目标**：在这台「官方固件删掉了 ADB」的机器上拿到 Root、打通 USB ADB，并安装 **UFI-TOOLS**（含高级后台）。
> **状态**：2026-10 实机全流程验证通关。发布时网上没有任何 AI 版可用的 Root / 强开 ADB 公开方案，本指南为实测记录整理。

> ⚠️ 版本判别：U30 Pro「AI 版」与「原版」硬件型号都是 MU5358，**看固件版本号区分**——AI 版固件带 `AI` 字样（实测 `U30Pro_AIV1.0.0B05`），原版走 B07 等固件线。本指南的偏移/参数以 AI 版 B05 实测为准，其它固件线请先自行核对再参考。

---

## ⚠️ 先读我（风险声明）

- 本文涉及**向系统分区写入数据**，属于高风险操作，可能导致**设备变砖、数据丢失、失去保修**。
- U30 Pro（含 AI 版）**官方不提供任何固件/刷机包**，没有"刷回去"的官方退路；实测机 B 槽（`trustos_b`）为**全零**，没有第二套启动链兜底。
- 你**唯一的后悔药 = 你自己的完整分区备份**。没有备份，不要进行任何写入。
- 所有素材（分区原件、候选镜像）**必须来自你自己设备的回读**。永远不要套用别人的镜像/哈希——每台设备的签名区、槽状态都不同。
- 本指南仅供个人学习与技术研究，风险自担；请勿用于商业刷机或任何违法用途。

---

## 0. 背景：为什么"正常装 UFI-TOOLS"走不通

[UFI-TOOLS](https://github.com/kanoqwq/UFI-TOOLS) 是社区作者 kanokano 为中兴 F50/U30 系列开发的增强工具（网页后台在 `设备IP:2333`，支持流量统计、锁频、AT 指令、插件、高级后台等）。它的官方安装指南有两条路——"装到手机上用"、"装进设备里用"——**两条都要求设备能开 ADB**：

- 老固件常规入口：浏览器打开 `http://192.168.0.1` 后台 →「USB 调试」按钮（或 `#usb_port` 页）→ USB 连电脑 → ADB 安装工具；
- **U30 Pro AI 版固件实测删掉了这条路的写接口**：后台 API `USB_PORT_SETTING` 返回失败、`usb_port_switch` 恒为 0，前台菜单也没有入口（对照实验：同一套后台协议下其它设置项都能正常写入，唯独 USB 调试这一项被裁掉）。

结论：AI 版没有"先开 ADB 再装工具"的路。本方案改为：**先 Root（写补丁 TrustOS + Magisk），再打通 USB ADB，最后装 UFI-TOOLS**。

## 1. 方案总览（原理一句话 + 四阶段）

**原理**：展锐 BSP 的启动校验存在"**验证的内容 ≠ 实际执行的内容**"分离缺陷（社区研究：TomKing062 的 unisoc_chipram_signcheck_exploit）。把 TrustOS 中"校验失败就拒绝启动"的判断函数改成**恒返回通过**，设备就能带着"签名与内容不匹配"的镜像正常启动——从而刷入打了 Magisk 的 init_boot 拿到 Root。

| 阶段 | 内容 | 结果 |
|---|---|---|
| 0 | 只读取证 + 全分区备份 | 拿到本机素材（trustos_a / sml_a / init_boot_a / misc…）与回滚材料 |
| 1 | 写入**打补丁的 TrustOS**（活动槽） | 校验函数"恒通过"，为下一步放行 |
| 2 | 写入**Magisk init_boot**（保留原厂 vbmeta 签名元数据） | 获得 Magisk Root |
| 3 | 原厂 Web 后台改「USB 上网协议」→ RNDIS | **USB 出现 ADB 接口**（关键的最后一步） |
| 4 | ADB 安装 **UFI-TOOLS** + 权限配置 + 网页后台 | 完整版 UFI-TOOLS 可用（含高级后台） |

> 本指南基于**活动槽 A** 的设备；如果你的设备活动槽是 B，请把所有目标分区换成对应槽名（`trustos_b` / `init_boot_b` 等），偏移和构造细节需先自行核对。

## 2. 阶段 0：准备

### 2.1 硬件与软件

- Windows 电脑一台、可靠的 USB 数据线、设备电量 50% 以上；
- 展锐 USB 驱动：官方 SPRD U2S Diag 驱动，或用 Zadig 装 LibUSB/WinUSB（目标：下载模式端口 `SPRD U2S Diag` 能正常出现）；
- 写入工具：**spd_dump**（展锐 SPRD4/FDL2 下载工具；本流程使用 TomKing062/spreadtrum_flash 的预构建版：<https://github.com/TomKing062/spreadtrum_flash>）；
- 7-Zip、Python 3（组装脚本用）、Magisk 官方 APK：<https://github.com/topjohnwu/Magisk>（实测 30.7）；
- 候选镜像组装脚本：可参考社区 B24 工具包（见第 9 节）里的同款脚本，**把硬编码素材换成你自己的设备回读件**。

### 2.2 进入下载模式（写入/回读的唯一通道）

1. 设备**关机**，USB 保持连接；
2. **长按**电源键 3–5 秒开机（⚠️ 短按只会显示充电图标，不会进入系统）；
3. 开机瞬间只存在 **1–3 秒**的 `SPRD U2S Diag` 端口窗口——必须在这个窗口内完成握手：
   - 先发 10 字节 kick 帧：`7e 00 00 00 00 08 00 fe 82 7e`
   - 再发 `AT+SPREF="AUTOLOADER"` 帧（`7e 00 00 00 00 20 00 68 00 … 0d 0a 7e`）
4. 成功标志：工具直连得到 `SPRD4:AutoD`（FDL2 会话），此后可以 `read_part` / `w` / `reset`。
   - 提示：`spd_dump --kick --wait 30 …` 可让工具自己抢窗口；抢不到就自己用 30ms 级轮询抓端口（工具默认轮询间隔较大，命中靠运气）。

### 2.3 全分区只读备份（必做）

用下载模式把**至少**这些分区完整回读保存（以你的分区表为准）：
`trustos_a`、`sml_a`、`misc`、`init_boot_a`、`vbmeta_a` ……建议 45 个分区**全量备份**（约 10 GB）。

同时回读一次 `trustos_b` 确认槽状态（本机实测为全零 = 无兜底，请对你的设备做同样确认）。

> 💡 写入通道"是否真实"的判别法：向 `reserve1` 写一段 **8 MiB 非全零图案**，回读比对哈希。
> （注意：向本来就是全零的分区写全零数据、"回读一致"是**没有判别力**的——这是踩过的坑。）

**备份判据**：任意分区二次回读哈希与首次一致。没有备份，不要进行任何写入。

## 3. 阶段 1：写入补丁 TrustOS（活动槽）

### 3.1 构造候选（全部离线完成）

1. 取**你自己设备**的 `trustos_a`（6 MiB）。AI 版 B05 实测的补丁位在**文件偏移 `0x98418`**（正文偏移 `0x98218`），把 4 个字节：

   ```
   e0 03 1a 2a    原指令（校验判断，mov w0, w26）
   改为
   00 00 80 52    改为恒返回"通过"（mov w0, #0）
   ```

   ⚠️ 动手前先在你自己 dump 的文件里确认这几个字节存在、上下文一致；不一致就**停下**，先搞清楚差异。

2. 按同族已验证算法重组 6 MiB 候选：
   - 补丁后的 TrustOS 正文（5,046,272 B）＋ 本机 `sml` 正文（147,568 B）拼接；
   - 签名尾部的 `payload_offset` / `cert_offset` 各加上 TrustOS 正文长度；
   - 从 `0x4f4708` 起**原样保留原厂尾部**；
   - 产出的候选镜像："头部正文摘要与拼接正文不相符"是**故意**的——能否启动就依赖上一步的校验绕过。
3. 离线自检：结构、偏移、证书路径、总长度与分区大小一致（6 MiB）。

### 3.2 写入与验证（一次下载模式会话内完成）

```
read_part trustos_a <全分区>   → 写前快照（再次确认与备份一致）
w         trustos_a <候选镜像> （约 5–10 秒）
read_part trustos_a <全分区>   → 写后回读，哈希必须与候选一致
read_part misc      <4KiB>     → 终态探针
reset                          → 重启生效
```

重启后设备应**正常进入系统**（Wi-Fi、网页都正常）。此时 TrustOS 已处于"校验恒通过"状态。

### 3.3 回滚

把备份中的原始 `trustos_a` 用同样方式写回即可恢复原状。

## 4. 阶段 2：写入 Magisk init_boot（保留原厂 vbmeta）

### 4.1 构造候选（两步，全部离线）

1. **打 Magisk 补丁**：用 Magisk 官方 payload（从 30.7 APK 中提取 arm64 部分）给**你自己**的 `init_boot_a.bin`（8 MiB）打补丁——替换 init 为 magiskinit、注入 overlay 脚本（可选：把电脑的 ADB 公钥写进去，方便免密）。
2. **保留原厂 vbmeta（关键！）**：把候选的 vbmeta 区域换成**原厂 `init_boot_a` 里那份已签名的 vbmeta blob**，并按 blob 实际长度修正 footer（AI 版实测：footer 写到偏移 `2998272`、blob 长度 `2304`——请让脚本自动计算，不要照抄硬编码）。
   - ⚠️ 教训：直接用标准"AVB NONE 签名"的版本 **会 bootloop**（B24 同族实机验证过）；
   - "保留原厂签名元数据"版的特征：外部签名元数据与原厂一致（SHA256_RSA4096、父公钥链匹配），但**数据描述符 digest 与新 ramdisk 不匹配**——这正是需要阶段 1 的 TrustOS 校验绕过放行的原因。

### 4.2 写入与验证

同阶段 1 的方式：写前回读 → `w init_boot_a <候选>` → 写后回读哈希一致 → `reset`。

重启后：系统正常进入，且已经拿到 Magisk Root（此时 USB 里还没有 ADB——下一步解决）。

### 4.3 回滚

写回备份的原始 `init_boot_a`。

## 5. 阶段 3：打通 USB ADB（关键的最后一步）

重启后 USB 组合默认是 `rndis,mtp`，**不含 adb**，`adb devices` 为空。此时：

1. PC 通过 RNDIS 连上设备（网卡拿到 `192.168.0.x` 地址），浏览器打开 `http://192.168.0.1`，用你的后台密码登录；
2. 打开菜单「**USB 上网协议**」（`#usb_internet_protocol`）；
   - 原值 `自动(0)` → **改为 `RNDIS(1)`** → 保存；
3. 点「重启设备」（或断电重启）；
4. 重启后 USB 变成 `VID_19D2&PID_1373` 复合设备（含 ADB Interface），`adb devices` 立刻出现设备。

> 等效后台接口（给自动化脚本的线索）：`goformId=SET_USB_NETWORK_PROTOCAL&usb_network_protocal=1`，随后 `goformId=REBOOT_DEVICE`；调用需要带上 `Referer: http://192.168.0.1/index.html` 等请求头。

### 5.1 验证 Root

```bash
adb shell id                      # uid=2000(shell)
adb shell /debug_ramdisk/su -c id     # uid=0(root) context=u:r:magisk:s0 ⭐
adb shell /debug_ramdisk/magisk -V    # 30.7:MAGISK:R
```

> 注意：**裸 `su` 不在 PATH**，请使用绝对路径 `/debug_ramdisk/su`。

### 5.2 （可选）网络 ADB 与双通道固化

- 网络 ADB：`persist.adb.tcp.port 5555` 固定端口 + `setprop ctl.restart adbd`，之后 `adb connect 192.168.0.1:5555` 即可。
- 想"重启后自动恢复双通道"：用 Magisk `service.d` 开机脚本实现（要点与坑见第 8 节 FAQ）。

## 6. Root 后必做：修复 Magisk 环境（大坑）

首次启动后 `/data/adb/magisk/` 可能是**空的**（日志：`Magisk environment incomplete, abort`）→ `post-fs-data.d` / `service.d` / 模块系统**全部不执行**。修法：从官方 Magisk APK 里提取文件放入 `/data/adb/magisk/`（root:root；可执行文件 755、其余 644）：

| APK 内路径 | 目标文件名 |
|---|---|
| `lib/arm64-v8a/libmagisk.so` | `magisk64` |
| `lib/armeabi-v7a/libmagisk.so` | `magisk32` |
| `lib/arm64-v8a/libmagiskinit.so` | `magiskinit` |
| `lib/arm64-v8a/libmagiskpolicy.so` | `magiskpolicy` |
| `lib/arm64-v8a/libmagiskboot.so` | `magiskboot` |
| `lib/arm64-v8a/libbusybox.so` | `busybox` |
| `lib/arm64-v8a/libinit-ld.so` | `init-ld` |
| `assets/` 下的 `stub.apk`、`util_functions.sh`、`boot_patch.sh`、`module_installer.sh`、`addon.d.sh`、`uninstaller.sh`、`main.jar`、`chromeos/*` | 同名 |

修好后日志出现 `Running post-fs-data.d scripts / service.d scripts / module service scripts`，模块与脚本系统恢复正常。

## 7. 阶段 4：安装 UFI-TOOLS

### 7.1 安装 APK

从官方渠道下载最新 APK（Web 版）：<https://github.com/kanoqwq/UFI-TOOLS>（早期名 F50-SMS）或作者网盘 `pan.kanokano.cn/UFI-TOOLS-UPDATE`。安装到设备（包名 `com.minikano.f50_sms`）：

```bash
adb install -r UFI-TOOLS_WEB_V*.apk
adb shell appops set com.minikano.f50_sms GET_USAGE_STATS allow
adb shell dumpsys deviceidle whitelist +com.minikano.f50_sms
adb shell pm grant com.minikano.f50_sms android.permission.POST_NOTIFICATIONS
```

- `GET_USAGE_STATS` =「使用情况访问」权限，不开的话"当日流量统计"不可用；
- 加入电池优化白名单，保证后台服务与开机自启稳定。

### 7.2 设备端初始化

在设备上打开 UFI-TOOLS（或投屏操作）：

1. 允许通知等权限；
2. 关闭系统对它的「暂停未使用应用的活动」「电池优化」等限制；
3. 填网关 `192.168.0.1`、端口 `8080` → 启动服务（显示"服务运行中"后，开机自启随之生效）；
4. ⚠️ 安装/配置完成后**等约 3 分钟再断电**（等应用缓存完全写入；提前拔电可能丢 App）。

### 7.3 网页后台

- 同一网络下浏览器打开：`http://192.168.0.1:2333`
- 登录：工具口令（默认 `admin`，以工具说明为准）+ **设备后台密码**（即 `192.168.0.1` 网页的登录密码）。
- 面板功能：流量管理 / 接入设备 / WiFi 设置 / AT 指令 / 高级功能 / 数据开关 / USB调试 / ADB自启 / 定时任务 等。

### 7.4 建议：开启高级后台

面板「高级功能」→「添加高级功能」，可得到：

- **TTYD 网页终端**：`http://192.168.0.1:1146`（root shell，提示符 `#`）
- Samba 共享、开机脚本 `/sdcard/ufi_tools_boot.sh`、定时脚本 `/sdcard/ufi_tools_schedule.sh`
- 「禁用固件更新」等开关

### 7.5 开机脚本与插件机制

- 开机脚本 `ufi_tools_boot.sh`（在 `/sdcard/`）：由应用的 BootReceiver 在**真开机**时以 root 执行一次（应用自己重启不会执行）——适合放你的自启服务；
- 面板「插件功能」→ 添加插件（上传 `.txt` 插件文件）——社区插件生态入口。

## 8. FAQ 与踩坑合集

1. **后台登录不上 / 按钮全灰**：先点登录；登录 = 工具口令 + 设备后台密码。
2. **网页打开没数据**：地址要用「设备IP:2333」；网关填 `192.168.0.1`。
3. **拔电后 App 消失**：安装完成后等约 3 分钟再断电。
4. **没有温度/CPU 占用**：部分机型无对应传感器导出，属正常。
5. **开发者选项里「USB 调试」开关自动变回关**：开机脚本/看门狗里补 `settings put global adb_enabled 1`（及 `development_settings_enabled 1`）。
6. **改 USB 组合把 RNDIS 弄死 / adbd 被杀**：
   - 用框架命令 `svc usb setFunctions rndis,adb`，不要裸 `setprop sys.usb.config`；
   - 顺序很重要：先让 adbd 起来 → 再切 USB 组合 → 最后检查并 `setprop ctl.restart adbd`；
   - 组合切换会让 USB 重新枚举，PC 侧 adb 掉线几秒属正常。
7. **重启后 USB ADB 正常但 5555 不监听**：`ctl.start adbd` 对已在运行的 adbd 是空操作，要用 `ctl.restart adbd` 让它重读端口属性。
8. **开机后 1.5–3 分钟内 `192.168.0.1` 不通**：RNDIS 载波收敛慢，正常现象，别急着反复重枚举 USB。
9. **UFI-TOOLS 升级后插件没了**：新版改了插件机制，重新上传插件文件即可。
10. **变砖了怎么办**：回到下载模式（见 2.2），把备份分区原样写回。**所以：没有备份不要动手。**

## 9. 参考与致谢

- **UFI-TOOLS**：kanokano —— <https://github.com/kanoqwq/UFI-TOOLS>（旧名 F50-SMS）
- **Magisk**：topjohnwu —— <https://github.com/topjohnwu/Magisk>
- **spd_dump（下载/读写工具）**：TomKing062 —— <https://github.com/TomKing062/spreadtrum_flash>
- **TrustOS/启动链签名校验绕过思路**：TomKing062 —— <https://github.com/TomKing062/unisoc_chipram_signcheck_exploit>（展锐 UMS9360/9632 类平台，校验函数在 POSTROM）
- **B24 同族公开资料**（算法与工具链参考；AI 版需按本指南改造）：社区作者 qwXZhong 的 u30pro-b24-root-skill、SAMYUBA 的 u30pro-b24-root-kit
- 以及酷安社区关于 U30 Pro 系列的各种讨论帖。

## 10. 许可

MIT（随意转载，保留出处即可）。文中不含任何个人设备信息（序列号 / IMEI / 密钥 / 设备专属哈希均已移除）。
