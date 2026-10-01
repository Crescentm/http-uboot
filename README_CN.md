# Askey SBE1V1K Chainloader 与 HTTP 恢复

本项目为 Askey SBE1V1K 提供一个兼容原厂 `bootm` 的二级 U-Boot，以及一套基于 eMMC/GPT 的恢复环境。该设备采用 Qualcomm IPQ9570 SoC，2 GiB 内存，从 eMMC 启动。

Chainloader 用来解决这块板子特有的三个问题：

- 原厂 U-Boot 无法安全地直接加载当前的 U-Boot payload；
- OpenWrt 主线与大容量安装方案使用不同的 GPT 布局和启动参数；
- 固件镜像可能大到无法整个暂存在内存中。

> **警告**：固件和 chainloader 的更新本身就是破坏性操作。HTTP 服务器在接收镜像主体之前就会先擦除所选分区。断电、网络中断或镜像有误，都可能导致当前系统无法启动。在修改 GPT 或刷写镜像之前，请先为每台设备单独做好备份，并保留回到原厂 U-Boot 的途径：串口，或下文介绍的原厂网络恢复。

## 关于本分支

这是 [Crescentm/http-uboot](https://github.com/Crescentm/http-uboot) 的 `sbe1v1k-custom` 分支，基于 [YYH2913/http-uboot](https://github.com/YYH2913/http-uboot) 的 `sbe1v1k` 分支，新增了以下内容：

- **QSDK GPT 迁移**：使用 28 分区 QSDK 布局的设备可以在网页界面中迁移到 `large`、`mainline` 或 `qwrt`。
- **RAM 启动**：上传 initramfs 或 chainloader FIT，在不写入 eMMC 的情况下单次启动。
- **Netconsole**：恢复控制台同步镜像到 UDP，无需拆机即可观察和控制设备。基于 James Hilliard 的 lwIP netconsole 补丁系列，该系列尚未合入上游。
- **CI 构建**：每次 push 都会构建 FIT；打 `v*` 标签时发布为 release。

[English](README.md)

## 功能特性

| 功能                        | 实现方式                                                     |
| --------------------------- | ------------------------------------------------------------ |
| 兼容原厂 `bootm`            | 以一个小型 AArch64 shim 作为 FIT kernel 加载；shim 负责定位、复制并启动真正的 `u-boot.bin` payload |
| 多布局启动                  | U-Boot 和 HTTP 恢复均支持 `mainline`、`large`、`qwrt` 三种 profile |
| HTTP 恢复                   | 静态地址 `192.168.255.1`，内置 DHCP 辅助服务，为直连主机分配 `192.168.255.2` |
| 流式写入固件                | 先擦除目标分区，再通过固定的 1 MiB 缓冲区写入请求主体，不在内存中保留完整镜像 |
| OpenWrt 镜像支持            | 校验 USTAR 头和 SBE1V1K `CONTROL` 元数据，接受包含 `kernel` 和 `root` 成员的 sysupgrade tar 镜像，以及与 profile 匹配的原始恢复镜像 |
| Chainloader 自更新          | 根据检测到的布局自动选择 `rsvd_2` 或 `chainloader`；可接受的 FIT 上限为 4 MiB |
| GPT 迁移                    | 网页界面可将尾部重建为 `mainline`、`large` 或 `qwrt`，同时保留已校验的前缀分区 |
| 分区备份                    | 可单独下载 `boot0`、`boot1` 或任意 GPT 分区，也可将所有可读分区以流式方式打包为一个 tar |
| 多速率以太网                | 当前 NSS/PPE 通路支持 QCA8075 1G、QCA8081 2.5G 以及 RTL8261BE 多速率/10G PHY |
| 恢复状态指示灯              | 通过硬件 PWM 指示准备、擦除、写入、完成和错误状态 |
| QSDK GPT 迁移               | 识别并迁移 28 分区 QSDK 尾部布局：`0:WIFIFW` 通过可断点续传、带 CRC 校验的复制进行搬移，P1-P21 保持不变 |
| RAM 启动                    | `/upload/ramboot` 接受最大 256 MiB 的 FIT，校验后从内存启动，不触碰 eMMC |
| Netconsole                  | HTTP 恢复将控制台输出广播到 UDP 6666 端口，并在同一端口接收输入 |
| 串口诊断                    | 提供板级日志分析和 NSS/PPE/EDMA 计数器快照，便于排查网络问题 |

HTTP 恢复在正常运行时保持串口控制台安静。如需开启可选的生命周期和协议诊断输出，请在启动 `http_recovery` 之前设置 U-Boot 环境变量 `recovery_debug=1`；即使不开启，校验、存储和网络错误也照常显示。该开关与按需执行的 `nss_debug` 命令以及 `nss_debug_log` 跟踪设置相互独立。

## 首次安装

### 1. 临时启动 Chainloader FIT

将主机配置为 `192.168.1.2/24`，并把 `sbe1v1k-chainloader.itb` 放到 TFTP 根目录。在原厂 U-Boot 提示符下只使用临时变量：

```sh
setenv ipaddr 192.168.1.1
setenv serverip 192.168.1.2
tftpboot 0x80000000 sbe1v1k-chainloader.itb
bootm 0x80000000
```

不要执行 `saveenv`。TFTP 会把原始 FIT 放在 `0x80000000`；之后原厂的持久启动流程会从 eMMC 把已安装的 FIT 读到 `0x44000000`。shim 能识别这两个地址。

#### 没有串口时

原厂 U-Boot 自带一条不需要控制台的网络恢复路径：

1. 上电时按住 Reset，它会发起 DHCP 请求。
2. 如果 offer 中带有值为 `askey` 的 vendor option 43，它会通过 TFTP 从 `172.16.252.252` 获取 `rtq7300t_boot_auto_upgrade_fw.img`。
3. 然后执行该文件中的 `script` 镜像。

利用这条路径即可从内存启动 chainloader FIT：

1. 主机接在 LAN1 口，地址设为 `172.16.252.252/16`，并提供 DHCP（地址池 `172.16.252.10`-`172.16.252.20`，option 43 为 `askey`）和 TFTP 服务。例如：

   ```sh
   dnsmasq --interface=<if> --bind-interfaces --port=0 \
   	--dhcp-range=172.16.252.10,172.16.252.20,255.255.0.0,5m \
   	--dhcp-option=43,askey --enable-tftp --tftp-root=<dir>
   ```

2. 在 TFTP 根目录中放入 `sbe1v1k-chainloader.itb`，以及用 `mkimage -f auto.its` 生成的 `rtq7300t_boot_auto_upgrade_fw.img`。`.its` 文件中包含一个名为 `RTQ7300T`、大小为一字节的 `firmware` 镜像，以及一个内容如下的 `script` 镜像：

   ```sh
   setenv serverip 172.16.252.252
   if tftpboot 0x80000000 sbe1v1k-chainloader.itb; then
   	bootm 0x80000000
   fi
   ```

3. 断开电源。**先按住 Reset，再接通电源**。原厂 U-Boot 只在早期 autoboot 倒计时阶段检测按键；上电之后再按会直接启动已安装的系统。一直按住，直到 chainloader 传输完成后再多按几秒，让 chainloader 也检测到按键并进入 HTTP 恢复。

### 2. 启动 HTTP 恢复并备份设备

如果没有有效固件，二级 U-Boot 在启动失败后会自动进入 HTTP 恢复。也可以手动启动：

```text
http_recovery
```

所连接的主机保持 DHCP 即可，或手动配置为 `192.168.255.2/24`，不设网关。当 U-Boot 提示恢复服务器已在监听时，打开：

```text
http://192.168.255.1/
```

![首次安装时的备份页面](board/qualcomm/sbe1v1k-chainloader-fit/images/recovery-backup.png)

修改布局之前，必须备份 GPT 分区 `p1` 到 `p26`，以及 eMMC 的两个硬件 boot 分区。eMMC 规范中称这两块硬件区域为 Boot Partition 1 和 Boot Partition 2；U-Boot 和 Linux 中分别显示为 `boot0` 和 `boot1`，在备份包中保存为 `emmc-boot0.img` 和 `emmc-boot1.img`。

推荐在 Backup（备份）页面使用 **Download all (.tar)**（全部下载）。一键打包的归档包含 `boot0`、`boot1` 以及所有有效的 GPT 分区，`p1` 到 `p26` 自然都在其中。完整下载大约需要 15 分钟，具体取决于网络连接和目标存储；下载完成前请保持恢复服务器、浏览器和目标存储正常运行。

网页导出的归档是分区级的，不包含用户区的 GPT 头和未分配扇区。如需包含这些区域的完整扇区级镜像，请使用外部 eMMC 读卡器。

### 3. 选择并应用布局

打开 eMMC layout（eMMC 布局）页面：

1. 选择与固件匹配的布局。同时支持 QWRT 兼容布局。

2. 准确输入确认口令：

   ```text
   SBE1V1K_REPARTITION
   ```

3. 应用布局，等待 GPT、chainloader 和 APPSBLENV 校验完成。

4. 不要断电或重启，接着上传匹配的 OpenWrt 固件。

迁移会保留当前正在运行的 FIT，因此首次安装时无需另外上传 `sbe1v1k-chainloader-partition.img`。

### 4. 上传 OpenWrt 固件

首选输入是对应设备的 OpenWrt sysupgrade tar。它的文件名通常仍以 `.bin` 结尾，页面会根据文件内容识别出 tar 格式。

| 镜像格式               | 写入行为                                                     |
| ---------------------- | ------------------------------------------------------------ |
| OpenWrt sysupgrade tar | 边接收边解析 tar，只把 `kernel` 和 `root` 写入当前 profile 的目标分区 |
| 原始恢复镜像           | `mainline`/`qwrt`：前 7 MiB 写入 `0:HLOS`，其余写入 `rootfs`；`large`：前 32 MiB 写入 `kernel`，其余写入 `rootfs` |

浏览器先提交镜像长度。U-Boot 在主循环中擦除 kernel、rootfs 和 rootfs_data，返回 `prepared` 后，再接受主体长度完全一致的第二个请求。未经对应准备步骤的直接上传会被拒绝。

固件大小默认上限为 1 GiB，同时还受实际 kernel/rootfs 分区容量限制。原始镜像必须把 rootfs 放在正确的 7 MiB 或 32 MiB 边界上；原始镜像不能跨 profile 通用。

支持 QWRT 固件，请使用与所选分区布局匹配的镜像。

## 恢复网页界面

在二级 U-Boot 提示符下执行 `http_recovery`，然后打开：

```text
http://192.168.255.1/
```

内置的 DHCP 辅助服务通常会给所连接的电脑分配 `192.168.255.2/24`。如果 DHCP 不可用，请手动配置该地址，网关留空。

以下截图基于当前内嵌的 `index.html` 渲染，使用典型的 `large` 布局元数据。截图时没有进行任何破坏性操作。

### 公共控件与状态

侧边栏用于在五个操作页面之间切换。每个页面的标题栏显示当前目标、操作模式以及大小或边界限制。状态标签会视情况显示 `Idle`、`Ready`、`Preparing`、`Erasing`、`Writing`、`Done` 或某种错误状态。

Firmware 和 Chainloader 操作有三行进度条：

- **Upload** 显示浏览器到 U-Boot 的传输进度。
- **Erase** 显示 U-Boot 主循环对目标分区的准备进度。
- **Write** 显示已写入 eMMC 的字节数。

破坏性请求进行期间，导航和会产生冲突的控件都会被禁用。服务器端的固件写入、chainloader 写入、布局迁移和备份流互斥执行。

### Firmware 页面

![固件恢复页面](board/qualcomm/sbe1v1k-chainloader-fit/images/recovery-firmware.png)

**Firmware**（固件）页面用于将 OpenWrt 系统镜像安装到当前 profile。

- **Target** 会自动切换：`mainline`/`qwrt` 为 `0:HLOS + rootfs`，`large` 为 `kernel + rootfs`。
- **Choose file**（选择文件）接受 `.bin`、`.img`、`.itb` 或 `.tar`；浏览器会检查文件头，而不只看扩展名。
- OpenWrt sysupgrade tar 必须包含有效的 USTAR 头、位于 `kernel` 之前的 `CONTROL` 成员，以及可用的 `kernel` 和 `root` 成员。请使用与所选分区布局匹配的镜像。
- 原始恢复镜像必须在开头包含一个 FIT，并在 profile 边界处包含 SquashFS 根文件系统：`mainline`/`qwrt` 为 7 MiB，`large` 为 32 MiB。
- 页面显示的大小上限由实际分区容量和默认 1 GiB 上传上限共同决定。
- 点击 **Upload firmware** 后，浏览器先提交镜像长度。U-Boot 在 POST 回调之外擦除 kernel、rootfs 和 rootfs_data。只有状态变为 `prepared` 后，浏览器才会发送镜像主体。
- 写入成功后会安排重启。写入中断无法回滚。

对于流式传输的 sysupgrade tar，U-Boot 会在每个 512 字节的 tar 头到达时进行解析和校验，只写入需要的成员。写入完成后，它会从 eMMC 回读 FIT 并校验其内部哈希，再回读 SquashFS 超级块并核对其声明的大小。对于原始镜像，则先写入固定长度的 kernel 区段，再继续写 rootfs 目标，之后执行同样的回读校验。

### Chainloader 页面

![Chainloader 恢复页面](board/qualcomm/sbe1v1k-chainloader-fit/images/recovery-chainloader.png)

**Chainloader** 页面用于更新这套二级 U-Boot。

- 只接受原始的 `sbe1v1k-chainloader.itb`，不接受 `u-boot.bin`、OpenWrt 镜像或仅供分析用的 HLOS 封装。
- 可接受的镜像最大为 4 MiB。
- `mainline`/`qwrt` 选择 `0#rsvd_2`；`large` 选择 `0#chainloader`。
- 在流式写入 FIT 主体之前，会先擦除整个目标分区。
- 本页面不会修改其他 GPT 分区。
- 准备完成后断电，可能导致已安装的恢复启动路径丢失。

填充过的 `sbe1v1k-chainloader-partition.img` 用于离线写入；HTTP 页面应使用更小的原始 `.itb`。

### RAM 启动页面

**RAM boot**（RAM 启动）页面用于单次启动镜像，不写入存储。

- **Choose file** 接受最大 256 MiB 的 FIT（`.itb`）：可以是用于测试固件的 OpenWrt `initramfs` 镜像，也可以是用于测试新 chainloader 的 `sbe1v1k-chainloader.itb`。
- 镜像经 `fit_check_format()` 检查，且必须有默认配置。镜像不会写入 eMMC。
- chainloader FIT 会被移到 `0x44000000`，也就是 shim 查找它的位置。initramfs 启动时只带 `console=ttyMSM0,115200n8`，因此不会去等待已安装系统的根设备。
- 服务器先发送响应，然后停止服务、分离 netconsole，再执行 `bootm`。如果 `bootm` 返回，设备会复位。
- 断电重启即可回到已安装的系统。

### eMMC 布局页面

![eMMC 布局迁移页面](board/qualcomm/sbe1v1k-chainloader-fit/images/recovery-layout.png)

**eMMC layout**（eMMC 布局）页面用于切换受支持的分区 profile。

- **Partition profile** 支持 **OpenWrt mainline / factory-compatible**、**Large storage** 和 **QWRT factory** 兼容布局。
- 实时表格显示目标分区的标签、起始 LBA、大小和用途。
- 所有迁移 profile 都以 LBA `110626` 作为保留前缀的边界。
- 28 分区的 QSDK 源布局会被识别为 `qsdk`：
  - `0:WIFIFW` 从 LBA 40482 移到 40994，`0:WIFIFW_1` 写入一份经过校验的副本；
  - `0:LICENSE`、`0:HLOS` 和 `0:HLOS_1` 重建为空分区；
  - P1-P21 保持不变。

  `0:HLOS_1` 开头写有带 CRC 校验的标记，搬移中断后可以继续。
- 警告信息会说明哪部分尾部分区定义将被替换，以及重启前必须上传哪种固件。
- 在准确输入口令 `SBE1V1K_REPARTITION` 之前，操作按钮保持禁用。
- 迁移过程会校验固定的出厂锚点，保留正在运行的 FIT，写入并校验新的 GPT，将 FIT 重新安装到新目标，并更新和校验 `0:APPSBLENV`。
- 迁移完成后 HTTP 服务器继续运行，以便在重启前安装匹配的固件。

迁移会重写分区定义和启动配置，但无法恢复之前的布局已经覆盖掉的数据。

### Backup 页面

![分区备份页面](board/qualcomm/sbe1v1k-chainloader-fit/images/recovery-backup.png)

**Backup**（备份）页面是只读的。

- 下拉列表的内容来自 `GET /partitions`，包括 eMMC 的两个硬件 boot 区域 Boot Partition 1 和 Boot Partition 2（显示为 `boot0` 和 `boot1`），以及用户区中所有有效的 GPT 分区。
- **Download partition** 以原始 `.img` 文件流式下载所选分区，附带精确的 64 位 `Content-Length`。
- **Download all (.tar)** 请求 `GET /backup/all.tar`。U-Boot 生成一个标准的未压缩 ustar 包：先是两个硬件 boot 区域 `emmc-boot0.img` 和 `emmc-boot1.img`，之后每个 GPT 分区对应一个 `pNN-name.img`。
- 首次布局迁移前，必须保存好 `p1` 到 `p26`、`emmc-boot0.img` 和 `emmc-boot1.img`。一键归档已全部包含。
- 完整归档既不会暂存在内存中，也不会写到 eMMC 的临时分区。备份读取最多使用 16 MiB 的 DMA 对齐缓冲区。
- RPMB 不在备份范围内，因为它是需要认证的 eMMC 区域，不是普通的线性块分区。
- tar 包不含用户区 GPT 头和未分配扇区。如有需要，请使用外部读卡器对整个用户区做镜像。
- 出厂容量的完整备份超过 7 GiB。大约需要 15 分钟，具体取决于网络连接和目标存储。目标文件系统必须有足够的剩余空间，且不能是 FAT32。

原始备份中可能含有 MAC 地址、校准数据、密钥和授权信息。请将其作为设备专属的敏感数据妥善保管，并为重要的归档另行计算 SHA256。

## Netconsole

HTTP 恢复会把控制台同步镜像到 UDP，无需串口适配器即可查看输出和输入命令：

```sh
nc -u -l 6666               # watch: output is broadcast to 255.255.255.255:6666
nc -u 192.168.255.1 6666    # type: Ctrl-C leaves recovery for the U-Boot prompt
```

- 设置 `recovery_netconsole=0` 可将其关闭。设置 `ncip` 可改为只发送给指定主机而不广播，`ncinport` 和 `ncoutport` 用于修改端口。
- NSS 交换机会丢弃发往 CPU 的 UDP 包，DHCP、TFTP 端口和 netconsole 输入端口除外。
- 退出恢复模式后，netconsole 在 U-Boot 提示符下仍保持连接。执行 `http_recovery` 即可返回。
- 在操作系统启动之前，控制台会切回串口，`boot_openwrt` 和 RAM 启动都会这样做。`eth_halt()` 只会停止 PHY，此后如果再有控制台输出走网络，就会在新内核底下把链路和 EDMA 接收重新拉起来。

## 安全边界

- 本项目不会在写入前暂存完整固件，也不实现原子化的 A/B 更新。
- HTTP 恢复会检查所选目标、USTAR 校验和、SBE1V1K `CONTROL` 的位置、精确的请求长度、分区边界、写入后的 FIT 哈希以及 SquashFS 头。它不验证 OpenWrt 外层的 `fwtool`/`ucert` 签名，也不校验 rootfs 内容的端到端哈希。
- `mainline` 为保持与出厂分区编号兼容，保留了 `0:HLOS_1`、`rootfs_1` 和 `rootfs_data_1`。当前的更新程序只操作活动的 `0:HLOS`、`rootfs` 和 `rootfs_data`，不会切换到备用槽位。
- `large` 同样是单活动系统布局，不是 A/B。
- 恢复操作针对 eMMC 用户区的原始 GPT 分区。在这块板子上不会创建、调整或写入 UBI 卷。
- `boot0` 和 `boot1` 只通过只读的备份接口提供。正常安装、更新和离线恢复都不得把 chainloader 写入任一 eMMC 硬件 boot 区域。
- RPMB 被有意排除在备份和写入操作之外。
- 不要在原厂 U-Boot 中执行 `saveenv`。迁移代码只更新 `0:APPSBLENV` 中必要的变量，其余条目全部保留。

## 构建与产物

在 U-Boot 仓库根目录下执行：

```sh
make sbe1v1k_chainloader_defconfig
make CROSS_COMPILE=aarch64-linux-gnu- -j8
board/qualcomm/sbe1v1k-chainloader-fit/build-chainloader-fit.sh \
	--payload u-boot.bin \
	--outdir sbe1v1k-chainloader
```

打包脚本会输出每个产物的 SHA256 以及 FIT 中内嵌的 U-Boot 版本。在提供下载或刷写之前，请确认版本标识中不含 `dirty`。

| 文件                                | 用途                                               | 正确用法                                                     |
| ----------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| `sbe1v1k-chainloader.itb`           | 原厂 U-Boot TFTP 启动及 HTTP Chainloader 更新      | 作为原始 FIT 用于 TFTP 或 HTTP；不要当作离线整分区镜像使用   |
| `sbe1v1k-chainloader-partition.img` | 用零填充到正好 4 MiB 的原始 FIT                    | 仅用于离线写入正确的 chainloader 存储区域                    |
| `sbe1v1k-chainloader-hlos.elf`      | 用于分析和实验的 Askey HLOS 封装                   | 正常安装流程不使用                                           |
| `sbe1v1k-chainloader-shim.bin`      | 内嵌在 FIT 中的一级 shim                           | 构建中间产物，不要直接刷写                                   |
| `sbe1v1k-chainloader-control.dtb`   | 内嵌在 FIT 中的 control DTB                        | 构建中间产物，不要直接刷写                                   |
| `u-boot.bin`                        | 真正的二级 U-Boot payload                          | 切勿直接交给原厂 `bootm`，也不要直接写入 eMMC                |

GitHub Actions（`.github/workflows/sbe1v1k-chainloader.yml`）在每次 push 时构建同样的产物。每次运行都会将其保存为 workflow artifact，打 `v*` 标签时发布为 release。

## 刷写流程汇总

| 场景                        | 文件或操作                                                   | 实际写入位置                                                 |
| --------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 首次安装                    | 从原厂 U-Boot 通过 TFTP 启动 `sbe1v1k-chainloader.itb`，然后在网页界面中执行布局迁移 | 迁移会安装正在运行的 FIT 并更新 `0:APPSBLENV`；不要手动写入 HLOS 槽位 |
| HTTP 更新二级 U-Boot        | 在 Chainloader 页面上传原始 `sbe1v1k-chainloader.itb`        | `mainline`/`qwrt` 写入 `rsvd_2`；`large` 写入 `chainloader`  |
| HTTP 更新 OpenWrt           | 上传与当前 profile 匹配的 sysupgrade tar 或原始恢复镜像      | 写入活动的 kernel/rootfs 目标，并擦除 rootfs_data            |
| 离线修复二级 U-Boot         | 用 `dd` 写入 `sbe1v1k-chainloader-partition.img`             | `mainline`/`qwrt` 写入 `rsvd_2` 的前 4 MiB，`large` 写入整个 `chainloader` 分区 |
| 完整恢复 eMMC               | 使用同一台设备上经过校验的用户区、boot0 和 boot1 镜像        | 仅作为最后手段；正常安装不需要整盘写入                       |

切勿将 `u-boot.bin` 直接写入 eMMC。切勿将 chainloader 安装到 `0:HLOS`、`0:HLOS_1`、boot0 或 boot1。

## 支持的分区布局

下文所有 LBA 均以 512 字节扇区为单位。HTTP 恢复按标签选择分区并校验固定的 LBA，不依赖不稳定的分区序号。

### 布局对比

| Profile    | Kernel           | Rootfs            | 数据                                              | Chainloader                                | Linux root 参数     |
| ---------- | ---------------- | ----------------- | ------------------------------------------------- | ------------------------------------------ | ------------------- |
| `mainline` | `0:HLOS`，7 MiB  | `rootfs`，122 MiB | `rootfs_data`，512 MiB                            | `rsvd_2`，只加载前 4 MiB                   | `/dev/mmcblk0p27`   |
| `large`    | `kernel`，32 MiB | `rootfs`，1 GiB   | `rootfs_data`，占用全部剩余空间，约 6.2 GiB       | `chainloader`，4 MiB                       | `PARTLABEL=rootfs`  |
| `qwrt`     | `0:HLOS`，7 MiB  | `rootfs`，122 MiB | `rootfs_data`，512 MiB；QWRT 使用 rootfs 尾部的 overlay | `rsvd_2`，只加载前 4 MiB              | `/dev/mmcblk0p27`   |

### 兼容性矩阵

| 现有存储布局                                                 | 检测结果           | 需要的操作                                                   |
| ------------------------------------------------------------ | ------------------ | ------------------------------------------------------------ |
| 标准出厂 GPT，或已安装的 OpenWrt 主线布局                    | `mainline`         | 首次安装仍需执行目标 profile 的迁移。选择 `mainline` 保留主线分区边界，或选择 `large` 转换为大容量布局 |
| QWRT 固件 | `qwrt` | 使用匹配的 QWRT 镜像 |
| 已安装的大容量布局                                           | `large`            | 可直接启动；固件和 chainloader 更新会自动选择 large 布局的目标分区 |
| 为大容量布局构建的 QSDK 固件 | `large` | 使用与大容量布局匹配的镜像 |
| 没有 `0:HLOS_1` 的出厂变体 | `factory`（迁移前） | 先备份设备并执行布局迁移，再上传固件 |
| 标签、起始位置或容量与两种描述均不符                         | `unknown`          | 不要直接上传固件。先备份设备再尝试迁移；如果锚点校验失败，说明该布局不受支持 |
| NAND/UBI 或非 SBE1V1K 布局                 | 不支持             | 不要使用本项目的 GPT 操作或离线偏移量                        |

### `large` Profile

迁移要求存在从 P1 到 `0:HLOS` 的标准 P1-P25 前缀。如果 P26 `0:HLOS_1` 已存在且符合出厂几何参数，则予以保留。如果只有 P1-P25 而 P26 的范围未分配，恢复程序会把 `0:HLOS` 的全部 7 MiB 复制到该范围，并在写入 GPT 之前逐块校验。因此最终的 large 布局编号始终一致：

|            起始 LBA |               扇区数 |          大小 | 名称            | 用途                                                  |
| -------------------: | -------------------: | ------------: | --------------- | ----------------------------------------------------- |
|           `< 96290`  |               不变   |          不变 | P1-P25          | 启动链、校准数据、PHY/Wi-Fi 固件以及 `0:HLOS`         |
|    96290 (`0x17822`) |    14336 (`0x3800`) |         7 MiB | P26 `0:HLOS_1`  | 保留原有分区，或从 `0:HLOS` 复制                      |
|   110626 (`0x1b022`) |      8192 (`0x2000`) |         4 MiB | P27 `chainloader` | 原始 `sbe1v1k-chainloader.itb`                      |
|   118818 (`0x1d022`) |    65536 (`0x10000`) |        32 MiB | P28 `kernel`    | OpenWrt/QSDK kernel FIT                               |
|   184354 (`0x2d022`) | 2097152 (`0x200000`) |         1 GiB | P29 `rootfs`    | SquashFS/根文件系统镜像                               |
| 2281506 (`0x22d022`) |    直到最后一个可用 LBA | 约 6.2 GiB | P30 `rootfs_data` | F2FS overlay 或持久化数据                           |

该布局在末尾预留 33 个扇区，用于存放有效的备份 GPT。恢复程序按标签寻址分区，并校验固定的 LBA。

### `mainline` Profile

此 profile 重建与出厂兼容的尾部分区，并把原本空着的 `rsvd_2` 分区用作 chainloader 的容器：

|            起始 LBA |           扇区数 |    大小 | 名称          | 用途                                                    |
| -------------------: | ---------------: | ------: | ------------- | ------------------------------------------------------- |
|    81954 (`0x14022`) | 14336 (`0x3800`) |   7 MiB | `0:HLOS`      | OpenWrt kernel FIT                                      |
|   110626 (`0x1b022`) |           249856 | 122 MiB | `rootfs`      | OpenWrt 根文件系统镜像                                  |
|   610338 (`0x95022`) |          1048576 | 512 MiB | `rootfs_data` | Overlay 或持久化数据                                    |
| 5201954 (`0x4f6022`) |            65536 |  32 MiB | `rsvd_2`      | 原始 chainloader FIT；原厂 U-Boot 读取其前 4 MiB        |

`0:HLOS_1`、`rootfs_1` 和 `rootfs_data_1` 会被保留或重建，以保证主线的根设备仍为 `/dev/mmcblk0p27`。P26 缺失时，`0:HLOS_1` 会用 `0:HLOS` 的内容填充，而不是建成空分区。对于当前的恢复更新程序来说，它们并不是可切换的备用槽位。

### `qwrt` Profile

`qwrt` profile 用于支持兼容的 QWRT 固件。

QWRT profile 将 chainloader 保留在 `rsvd_2` 中，并提供相应的启动配置。

### 布局检测与迁移

HTTP 恢复先校验 `large` 的 GPT 几何参数，再校验 `mainline`：

1. 检查固定的分区标签、起始 LBA 和容量，包括已知的 `0:HLOS` 锚点。
2. `large` 必须匹配 `chainloader`、`kernel`、`rootfs` 和 `rootfs_data`；它仍是唯一一个 rootfs 为 1 GiB 的 profile。
3. `mainline` 必须匹配 `0:HLOS_1`、p27 `rootfs`、`rootfs_data` 和 `rsvd_2`。QWRT 兼容布局使用相同的分区几何参数。
4. 匹配成功后，设置该 profile 对应的 kernel、rootfs、data、chainloader、root 参数以及 `recovery_kernel_pad` 值。
5. 如果两种 GPT 几何参数都不匹配，但固定的启动链锚点和 `rsvd_2` 匹配，恢复程序报告 `factory`。此时只在 Chainloader 页面选择 `rsvd_2`，并在迁移完成前锁定固件上传。
6. 如果既不匹配任何 profile，也不匹配出厂启动路径，恢复程序报告 `unknown`。这种状态下不要依赖默认目标去刷写固件。

布局迁移另有一套独立的前缀安全检查：

1. 统计 LBA `110626` 以下的所有分区；要求恰好是标准的 P1-P25 前缀，后面可以跟一个标准的 P26 `0:HLOS_1`。
2. 校验 P1-P25 每个分区的标签、起始 LBA 和大小，然后保留磁盘 GUID 和当前活动的 chainloader FIT。
3. 如果没有 P26，则分块将 P25 `0:HLOS` 复制到 LBA `96290..110625`，并在修改 GPT 之前逐块比对目标数据。
4. 构建带有 P26 `0:HLOS_1` 的所选 GPT，执行 `gpt write`，并严格校验新的前缀和目标布局。
5. 擦除新的 chainloader 目标分区，重新安装保留下来的 FIT。
6. 更新 `0:APPSBLENV` 中的 `bootargs`、`boot_chainloader`、`do_boot`、`do_nothing` 和 `bootcmd`，重新计算 CRC，并完整回读校验。

从 `large` 转回 `mainline` 只会恢复分区定义，无法恢复之前已被覆盖的出厂尾部数据。

## 更新已安装的系统

### 更新 OpenWrt

进入 HTTP 恢复，打开 Firmware 页面，上传与检测到的 profile 匹配的 sysupgrade tar 或原始恢复镜像。在接收主体之前，活动的 kernel、rootfs 和 rootfs_data 会被擦除。

在运行中的系统里能否直接使用 OpenWrt 的 `sysupgrade`，取决于该固件自带的平台升级脚本。本文档只介绍二级 U-Boot 的 HTTP 路径。

### 更新二级 U-Boot

进入 HTTP 恢复，打开 Chainloader 页面，上传原始的 `sbe1v1k-chainloader.itb`：

| Profile    | 自动选择的目标                                               |
| ---------- | ------------------------------------------------------------ |
| `factory`  | `0#rsvd_2`；只允许更新 chainloader、备份和布局迁移           |
| `mainline` | `0#rsvd_2`；擦除整个分区，并将 FIT 写在分区开头              |
| `qwrt`    | `0#rsvd_2`；擦除整个分区，并将 FIT 写在分区开头              |
| `large`    | `0#chainloader`；擦除整个分区并写入 FIT                      |

不要在此页面上传 `u-boot.bin`、`*-hlos.elf` 或 OpenWrt 固件。4 MiB 的 `*-partition.img` 专用于离线整区写入。

<details>
<summary>可选：不重新编译固件，用 extroot 扩展可写空间</summary>

此方法将现有的一个数据分区用作 `/overlay`，不改动固件镜像、GPT、启动参数或 chainloader。要求固件支持 extroot（`block-mount`）和 ext4，且功能正常。

**警告**：下面的示例仅适用于标准 43 分区布局，其中 `mmcblk0p42` 为 `user_data`（约 4.7 GiB），`mmcblk0p40` 为存放 chainloader 的 `rsvd_2`。其他布局不要套用这些分区编号。请先核实标签、边界和挂载情况。分区未挂载或文件系统探测结果为空，并不能说明其中没有重要数据。

操作前请先备份设备，包括整个目标分区和当前配置。格式化会清除目标分区上的全部数据。不要动 `rsvd_2`、boot 分区和现有的内部 overlay。复制期间请停止会写入配置或应用数据的服务；迁移完成之前不要安装软件包或修改设置。

检查目标分区和当前的 overlay：

```sh
cat /sys/class/block/mmcblk0p42/uevent
cat /sys/class/block/mmcblk0p42/start /sys/class/block/mmcblk0p42/size
block info /dev/mmcblk0p42
mount
losetup -a
```

在本例中，目标分区必须满足 `PARTNAME=user_data`、起始 LBA 为 `5398562`、扇区数为 `9850846`，且未被挂载，也未被 loop 设备占用。现有的持久化 overlay 必须挂载在 `/overlay` 且包含 `upper`。不要对临时的 RAM overlay 执行这套复制流程。

确认备份和目标分区无误后，在同一个 shell 中执行以下命令。子 shell 遇到错误会停止；任何一步失败都不要重启。

```sh
(
    set -eu
    grep -qx 'PARTNAME=user_data' /sys/class/block/mmcblk0p42/uevent
    [ "$(cat /sys/class/block/mmcblk0p42/start)" = 5398562 ]
    [ "$(cat /sys/class/block/mmcblk0p42/size)" = 9850846 ]
    [ -d /overlay/upper ]

    cp -p /etc/config/fstab /etc/config/fstab.pre-extroot
    mkfs.ext4 -F -L sbe1v1k_extroot /dev/mmcblk0p42
    UUID="$(block info /dev/mmcblk0p42 | sed -n 's/.* UUID="\([^"]*\)".*/\1/p')"
    [ -n "$UUID" ]

    mkdir -p /tmp/extroot-new
    mount -t ext4 /dev/mmcblk0p42 /tmp/extroot-new
    mkdir -p /tmp/extroot-new/upper /tmp/extroot-new/work

    # Disable any other /overlay entry before enabling this one.
    uci set fstab.sbe_extroot='mount'
    uci set fstab.sbe_extroot.target='/overlay'
    uci set fstab.sbe_extroot.uuid="$UUID"
    uci set fstab.sbe_extroot.options='noatime'
    uci set fstab.sbe_extroot.enabled='1'
    uci commit fstab

    cp -a /overlay/upper/. /tmp/extroot-new/upper/
    sync
    umount /tmp/extroot-new
    echo 'Extroot prepared. Reboot to verify.'
)
```

新的 `work` 目录保持为空。如果现有 overlay 依赖扩展属性、ACL 或 file capabilities，请改用能保留元数据的迁移工具，不要想当然地认为系统自带的 `cp -a` 会保留这些信息。

只有在准备步骤全部成功后，才执行：

```sh
reboot
```

重启后，检查实际的挂载情况和可写容量：

```sh
mount | grep -E ' /overlay |overlayfs'
df -h / /overlay
```

预期的挂载结果为：

```text
/dev/mmcblk0p42 on /overlay type ext4
overlayfs:/overlay on / type overlay
```

可写容量应接近 4.7 GiB（扣除文件系统开销）。请保留原来的内部 overlay 以备恢复。如果迁移失败，通过串口或 failsafe 模式在原来的 overlay 上恢复 `fstab.pre-extroot`。只在新的 overlay 中禁用 extroot 是不够的：启动时会先从原来的 overlay 读取配置。

固件升级、恢复出厂设置和布局迁移都可能清除 extroot 配置或使其内容失效。请先备份，并在每次升级后重新检查 extroot；不要在不同固件之间盲目沿用旧的系统 overlay。

</details>

## 离线 eMMC 恢复

如果已安装的 chainloader 在出现二级 U-Boot 启动信息之前就失败，且原厂 shell 也已无法进入，只需重写 eMMC 用户区中对应 profile 的 chainloader 目标分区。不要重写整个设备，也不要写入 boot0 或 boot1。

| Profile    | 标签          |             起始 LBA |   字节偏移量 |         离线写入长度 |
| ---------- | ------------- | -------------------: | -----------: | -------------------: |
| `mainline` | `rsvd_2`      | 5201954 (`0x4f6022`) | `0x9ec04400` |                4 MiB |
| `qwrt`     | `rsvd_2`      | 5201954 (`0x4f6022`) | `0x9ec04400` |                4 MiB |
| `large`    | `chainloader` |   110626 (`0x1b022`) |  `0x3604400` |                4 MiB |

将 eMMC 接到 Linux 主机后，先确认设备和实际 GPT：

```sh
sudo sgdisk -p /dev/sdX
lsblk -o NAME,SIZE,START,PARTLABEL /dev/sdX
```

如果存在正确的分区节点，将填充后的 4 MiB 镜像写入该节点：

```sh
sudo dd if=sbe1v1k-chainloader/sbe1v1k-chainloader-partition.img \
	of=/dev/sdXN bs=4M conv=fsync status=progress
```

如果没有分区节点，则在整个 eMMC 用户区设备上使用核实过的起始 LBA：

```sh
sudo dd if=sbe1v1k-chainloader/sbe1v1k-chainloader-partition.img \
	of=/dev/sdX bs=512 seek=<profile-start-LBA> conv=notrunc,fsync \
	status=progress
```

对于 `/dev/mmcblkN`，分区节点的形式为 `/dev/mmcblkNpM`。切勿照抄本文档中的分区序号；务必对照所连接的设备，同时核对 `PARTLABEL` 和起始 LBA。

如果确实不得不进行整盘恢复，在此之前，若读卡器能访问到，请先保存用户区和两个硬件 boot 区域：

```sh
sudo dd if=/dev/sdX of=sbe1v1k-emmc-user-before.img \
	bs=4M conv=sync,noerror status=progress
sudo dd if=/dev/mmcblkNboot0 of=sbe1v1k-emmc-boot0-before.img \
	bs=4M conv=sync,noerror status=progress
sudo dd if=/dev/mmcblkNboot1 of=sbe1v1k-emmc-boot1-before.img \
	bs=4M conv=sync,noerror status=progress
```

出厂的尾部分区延伸到了备份 GPT 所需的扇区，这也是迁移要重建尾部的原因之一。但这并不意味着可以覆盖 `0:ART`、`0:APPSBLENV` 或 `0:LICENSE` 中每台设备独有的数据。

## 启动链实现

原厂 U-Boot 的 `bootm` 能处理 FIT 中的 kernel 和 FDT，但如果把真正的二级 U-Boot 作为 FIT 的 `loadables` 条目，它可能被预加载到原厂加载程序仍在使用的内存上。因此 `conf-1` 只声明 shim kernel 和 control DTB：

```text
stock XBL
  -> stock U-Boot bootm
     -> chainloader shim
        -> locate uboot-1 in the raw FIT
           -> copy it to 0x4a240000
              -> enter second-stage U-Boot
```

TFTP 方式下 FIT 位于 `0x80000000`；持久安装的 FIT 被加载到 `0x44000000`。shim 要等原厂 `bootm` 交出控制权后，才从原始 FIT 中复制 `uboot-1`，从而避免过早覆盖原厂加载程序。

迁移会写入与 profile 对应的原厂环境变量值：

| Profile    | `bootargs` root    | `boot_chainloader` 读取范围      |
| ---------- | ------------------ | -------------------------------- |
| `mainline` | `/dev/mmcblk0p27`  | LBA `0x4f6022`，`0x2000` 个扇区  |
| `qwrt`     | `/dev/mmcblk0p27`  | LBA `0x4f6022`，`0x2000` 个扇区  |
| `large`    | `PARTLABEL=rootfs` | LBA `0x1b022`，`0x2000` 个扇区   |

所有 profile 都会设置 `do_boot=run boot_chainloader`、`do_nothing=true`，以及一个可中断的三秒 `bootcmd`。原厂 U-Boot 启动时会重置 `bootargs`，所以 `bootcmd` 在每次启动 chainloader 之前都会重新赋值。

二级 U-Boot 的 `detect_layout` 在每次正常启动前检查是否存在 `kernel` 标签。存在则加载 `large` 的 kernel，否则加载主线的 `0:HLOS`。如果 `bootm` 返回，会自动启动 HTTP 恢复。HTTP 恢复使用的是上文所述更严格的几何参数校验。

## 流式写入与边界控制

固件和 chainloader 的 HTTP 更新按以下顺序进行：

```text
browser submits format and decimal length
  -> U-Boot resolves the detected profile and target capacities
  -> U-Boot fully erases the target partitions
  -> status changes to prepared
  -> browser sends an exact-length body
  -> U-Boot writes through a 1 MiB DMA buffer
  -> U-Boot checks received and per-target byte counts
```

原始固件流会在 profile 边界处从固定长度的 kernel 区段切换到 rootfs。sysupgrade 流会解析每个 512 字节的 tar 头，只接受所需的 kernel/root 条目。所有写入都被限制在解析出的分区起止 LBA 之内。

在分区和擦除组对齐、可以安全操作的情况下，擦除优先使用精确的 eMMC erase/TRIM。如果擦除组的取整可能波及相邻分区，则改为对目标分区的每个逻辑块填零。

这种设计会增加 eMMC 的 I/O，但避免了在 2 GiB 内存中保存大镜像。这也意味着一旦准备阶段完成，当前系统就已经被破坏；主体传输失败时无法自动回滚。

## 分区备份实现

动态备份接口如下：

```text
GET /partitions
GET /backup/boot0.bin
GET /backup/boot1.bin
GET /backup/partition-N.bin
GET /backup/all.tar
```

单个备份和完整备份最多使用 16 MiB 的 DMA 对齐缓冲区。`all.tar` 先扫描源数据的元信息，算出精确的 64 位 `Content-Length`，然后随 socket 的消费进度生成标准 ustar 输出：

```text
member header -> raw partition data -> 512-byte padding -> next member
```

成员顺序为 eMMC Boot Partition 1（`boot0`）、eMMC Boot Partition 2（`boot1`），然后是所有有效的 GPT 分区，最后是两个 512 字节的全零结束块。完整的 tar 既不会存在于内存中，也不会存在于 eMMC 暂存区。数据流关闭、完成或失败时，都会先切回 eMMC 用户硬件分区 0，然后才允许执行破坏性请求。

tar 校验和只覆盖每个 tar 头，不覆盖分区原始内容。设备读取出错时 HTTP 响应会被截断。请检查下载文件的大小，并为存档副本另行计算 SHA256。

## 以太网与恢复网络

| 物理标识       | PPE 端口 | PHY                 | 最高速率     | 接口模式       |
| -------------- | -------: | ------------------- | -----------: | -------------- |
| LAN2           |        3 | QCA8075 地址 18     |           1G | QSGMII         |
| LAN3           |        4 | QCA8075 地址 19     |           1G | QSGMII         |
| LAN1           |        5 | QCA8081 地址 28     |         2.5G | USXGMII        |
| WAN            |        6 | RTL8261BE 地址 0    |          10G | USXGMII        |

实际协商速率取决于对端设备和网线。恢复模式允许在没有活动链路的情况下初始化以太网，并使用配置成功且报告链路已连接的端口。以太网 MAC 地址优先取自运行时环境，否则回退到 `0:APPSBLENV` 中的 `ethaddr`。

## 上游参考


| 公开来源 | 参考的实现 |
| --- | --- |
| [U-Boot 上游基线 `a7830e87555a`](https://github.com/u-boot/u-boot/commit/a7830e87555abfb81cc69275cecb2bc0fbde5b28) | 基础的 `bootm`、FIT、eMMC/GPT、PHY 和 lwIP 网络基础设施 |
| [OpenWrt PR #21586：Askey SBE1V1K 支持](https://github.com/openwrt/openwrt/pull/21586) | 板级 DTS、以太网端口与 PHY 拓扑、出厂/主线分区几何参数、启动环境行为以及 OpenWrt 镜像定义 |
| [OpenWrt PR #24033：IPQ9574 USXGMII 修复](https://github.com/openwrt/openwrt/pull/24033) | 公开的 IPQ9574 PCS/UNIPHY 模式配置、复位与时钟时序，以及 USXGMII 带内自协商 |
| [Linux 上游 QCA808x PHY 驱动](https://github.com/torvalds/linux/blob/master/drivers/net/phy/qcom/qca808x.c) | QCA8081 2.5G 速率通告以及 SerDes FIFO 链路状态处理 |
| [YYH2913/http-uboot](https://github.com/YYH2913/http-uboot) | 本分支所基于的 SBE1V1K chainloader、HTTP 恢复、布局迁移和 NSS/PPE 网络实现 |
| [James Hilliard："net: share the lwIP runtime and support netconsole"（v2）](https://patchwork.ozlabs.org/project/uboot/list/?series=521255) | 共享的 lwIP 运行时和 lwIP netconsole 传输层，在上游评审完成前提前合入本项目 |
| [OpenWrt 提交 `6369c9e5c799`：Realtek 5G/10G PHY 支持](https://github.com/openwrt/openwrt/commit/6369c9e5c79994c380d0c63cfb003c935a974332) | 公开的 RTL8261BE/RTL8261N 识别、初始化补丁引擎、固件表处理及链路状态逻辑 |
