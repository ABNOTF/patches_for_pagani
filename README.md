# OnePlus 13T (pagani) LineageOS 23.2 非官方构建指南

适用于一加 13T（PKX110，代号 `pagani`，sm8750 / 骁龙 8 至尊版）的 LineageOS 23.2（Android 16）构建流程。

> ⚠️ 本构建为非官方（unofficial）版本，刷机有风险，操作前请备份数据。如遇与本教程出入较大的问题，请善用各种AI助手。仓库中不包含任何专有文件（blobs），需自行从 ColorOS/OOS 固件提取，提取产物禁止二次分发。

## 免责声明

> **刷机有风险，操作需谨慎。**
>
> 本指南及所有相关构建均为非官方（unofficial）内容，按"现状"（AS IS）提供，
> 不附带任何明示或默示的担保。因参考本指南进行解锁、刷写等操作而导致的
> 设备变砖、数据丢失、保修失效,等一切后果，由操作者自行承担。

**继续阅读即表示你已理解并接受上述风险**

## 注意事项
- **COS/OOS**：即ColorOS和OxygenOS对应中国版系统和外国版系统
- **/path/to/xxx**：即你的文件目录

## 参考配置

笔者构建环境（天选 6 Pro）：

| 项目 | 配置 |
|---|---|
| CPU | Ultra 7 255HX |
| 内存 | Samsung DDR5 5600MHz 16 GB |
| 硬盘 | Samsung MZVL81T0HELB-00BTW |
| 系统 | Arch Linux (rolling, testing) |
| swap | 24 GB |
| zram | 8 GB (lz4) |

合计约 48 GB 可用内存时 `-j6` 可编译通过；物理内存更小请降低 `-j` 线程数，避免链接阶段 OOM。

- **补丁 `git am` 失败**：上游源码已变动，需手动对照补丁内容合并。
- **`extract-files.py` 报缺文件**：检查 eUICC / Hotword 组件是否已备齐 OOS 文件，或已按说明注释对应行。
- **同步上游后功能异常**：检查两个补丁是否仍在新源码上，必要时重新应用。


## 前置要求

- 正常可用的 AOSP 编译环境
- 磁盘空间：**约 330 GB 以上**（源码 + 编译产物）
- 可用内存建议 48 GB 以上（含 swap/zram）

### 安装构建依赖

**Arch Linux**（笔者环境）：

```bash
# 注意：/etc/pacman.conf 需启用 [multilib] 仓库（lib32-* 包依赖它）
sudo pacman -S --needed base-devel bc bison ccache curl flex git git-lfs gnupg gperf \
    imagemagick protobuf python-protobuf lib32-readline lib32-zlib elfutils gnutls \
    lz4 lzo openssl libxml2 libxslt pngcrush rsync squashfs-tools xxd zip unzip zlib \
    repo zstd p7zip
```

**Debian / Ubuntu**（与 LineageOS 官方 wiki 一致）：

```bash
sudo apt install bc bison build-essential ccache curl flex g++-multilib gcc-multilib \
    git git-lfs gnupg gperf imagemagick protobuf-compiler python3-protobuf \
    lib32readline-dev lib32z1-dev libdw-dev libelf-dev libgnutls28-dev lz4 \
    libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool \
    squashfs-tools xsltproc xxd zip zlib1g-dev
```

### 使用北外（BFSU）镜像站同步源码（国内推荐）

北外镜像站（`mirrors.bfsu.edu.cn`）完整镜像了 LineageOS 与 AOSP，可替代默认的 GitHub / googlesource 源：

```bash
# 1. 让 repo 工具本身也走镜像
export REPO_URL='https://mirrors.bfsu.edu.cn/git/git-repo'

# 2. 用镜像初始化 manifest
repo init -u https://mirrors.bfsu.edu.cn/git/lineageOS/LineageOS/android.git -b lineage-23.2

# 3. 把 manifest 里的远端地址替换为镜像（GitHub 与 googlesource 两处）
sed -i \
    -e 's|https://github.com/|https://mirrors.bfsu.edu.cn/git/lineageOS/|g' \
    -e 's|https://android.googlesource.com|https://mirrors.bfsu.edu.cn/git/AOSP|g' \
    .repo/manifests/default.xml

# 4. 同步（-c 只拉取当前分支，显著节省时间和流量）
repo sync -c -j4 --fail-fast
```

注意事项：

- 本文档"二、克隆设备相关仓库"中的个人仓库（本仓库、Neveark 的 fork 等）**不在镜像中**，仍需从 GitHub 克隆。
- manifest 里的 gitlab 源（如 TheMuppets）同样不在镜像中。
- 想恢复官方源：还原 `.repo/manifests/default.xml` 的改动，或去掉镜像参数重新 `repo init`。
- 镜像站为公益服务，请合理使用，务必带 `-c` 减少不必要的拉取。
- **如果您不清楚以上四条所述意思,请严格按照教程操作**

## 一、应用补丁

在克隆设备仓库之前，先给源码树应用两个补丁：

| 补丁 | 目标仓库 | 作用 |
|---|---|---|
| `0001-tmp-add-trigger-for-udfps.patch` | `frameworks/base` | SystemUI 在指纹按下/抬起时直接写 `notify_fppress` 节点，触发 FOD |
| `0002-tmp-hide-cameraid1-for-aperture.patch` | `packages/apps/Aperture` | 让 Aperture 的摄像头忽略列表对主摄（ID 0/1）同样生效 |

```bash
cd /path/to/source
patch -p1 -i ~/path/to/patch/android_develop/0001-tmp-add-trigger-for-udfps.patch

cd /path/to/source
patch -p1 -i ~/path/to/patch/android_develop/0001-tmp-add-trigger-for-udfps.patch
```

如需还原源码树原状，在对应仓库执行 `git reset --hard HEAD~1` 即可。

## 二、克隆设备相关仓库

在源码根目录执行：

```bash
# 设备树（本仓库）
git clone https://github.com/ABNOTF/android_device_oneplus_pagani -b lineage-23.2 device/oneplus/pagani
git clone https://github.com/ABNOTF/android_device_oneplus_sm8750-common -b lineage-23.2 device/oneplus/sm8750-common

# 内核模块（二选一：Neveark 或我的 fork）
git clone https://github.com/Neveark/android_kernel_oneplus_sm8750-modules -b lineage-23.2 kernel/oneplus/sm8750-modules
# git clone https://github.com/ABNOTF/android_kernel_oneplus_sm8750-modules -b lineage-23.2 kernel/oneplus/sm8750-modules

# 内核与设备树（LineageOS 官方）
git clone https://github.com/LineageOS/android_kernel_oneplus_sm8750-devicetrees -b lineage-23.2 kernel/oneplus/sm8750-devicetrees
git clone https://github.com/LineageOS/android_kernel_oneplus_sm8750 -b lineage-23.2 kernel/oneplus/sm8750

# OPLUS 硬件抽象层
git clone https://github.com/LineageOS/android_hardware_oplus -b lineage-23.2 hardware/oplus
```

## 三、提取专有文件（blobs）

### 1. 准备工具

**payload-dumper-go**（解包 payload.bin）：

- Arch Linux：添加 [archlinuxcn](https://www.archlinuxcn.org/) 镜像源后执行 `sudo pacman -S payload-dumper-go`
- Debian / Ubuntu：从 [GitHub Releases](https://github.com/ssut/payload-dumper-go/releases/tag/1.3.0) 下载对应平台的二进制

**erofs-utils**（解 EROFS 分区，必装）：

```bash
sudo pacman -S erofs-utils        # Arch
sudo apt install erofs-utils      # Debian / Ubuntu
```

### 2. 解包固件

从固件包中取出 `payload.bin` 然后：

```bash
mkdir -p ~/dumps/pagani
payload-dumper-go -o ~/dumps/pagani payload.bin
```

程序会把固件里的所有分区解成单独的 `.img` 文件。

### 3. 解出分区内容

其中文件系统分区是 EROFS 格式，需要再解一层。在 dump 目录下执行：

```bash
cd ~/dumps/pagani
mkdir system system_ext product vendor odm my_product my_stock
for part in system system_ext product vendor odm my_product my_stock; do
    fsck.erofs --extract="$part" "$part.img"
done
```

完成后 dump 目录应为：每个分区一个**同名文件夹**（解出的文件）+ 其余 `.img` **原样保留**——`boot.img`、`modem.img` 这类无需解包，`extract-files.py` 会直接读取。

⚠️ **注意事项**：若需要euicc和谷歌助手唤醒，需再进行一次如上操作，如对该注意事项不理解，请问ai

### 4. 运行提取脚本

```bash
cd device/oneplus/pagani
./extract-files.py --allow-prohibited-files /path/to/dump
```

> 注意：**dump 路径直接跟在参数后面**。

提取脚本默认包含两个可选组件（eUICC 与谷歌助手唤醒）。这两个组件的文件在 COS 中不存在，需要从**相同版本号**的 OOS dump 中取出，放到 COS dump 的**相同相对路径**下（即 `proprietary-files.txt` 中每条冒号前的路径）。不使用某组件则按下方说明注释对应行——**二选一：要么备齐文件，要么注释行，否则提取或编译会报错**。

### 可选：eUICC（eSIM）

需要从 OOS 补充的文件：

```
my_product/etc/permissions/EuiccGoogle_grant_permissions_list.xml
my_product/priv-app/EuiccGoogle/EuiccGoogle.apk
```

不需要 eSIM：注释 `device/oneplus/pagani/proprietary-files.txt` 第 **1306–1307** 行。

### 可选：谷歌助手唤醒（Hotword Enrollment）

需要从 OOS 补充的文件：

```
odm/etc/init/android.hardware.contexthub-service.qmi.rc
odm/etc/vintf/manifest/android.hardware.contexthub-service.qmi.xml
vendor/bin/hw/android.hardware.contexthub-service.qmi
vendor/etc/chre/preloaded_nanoapps.json
my_product/etc/sysconfig/com.android.hotwordenrollment.common.util.xml
my_product/framework/com.android.hotwordenrollment.common.util.jar
my_product/priv-app/HotwordEnrollmentXGoogleHEXAGON_WIDEBAND.apk
my_product/priv-app/HotwordEnrollmentYGoogleHEXAGON_WIDEBAND.apk
```

不需要助手唤醒：注释 `device/oneplus/sm8750-common/proprietary-files.txt` 第 **74–77** 行和第 **433–436** 行（共 8 行）。

## 四、编译

返回源码根目录：

```bash
breakfast pagani user    # 或 breakfast pagani eng / debug
m bacon -j6              # 线程数按 CPU 核数与内存酌情调整
```

编译产物输出在 `out/target/product/pagani/` 下（`lineage-23.2-*.zip`）。

首次刷入请参考 LineageOS 官方 wiki 的通用流程（解锁 → 刷入 recovery → `adb sideload` 升级包）。
<div align="center">
⚠️⚠️⚠️⚠️⚠️以下操作为选修⚠️⚠️⚠️⚠️⚠️<br>
⚠️⚠️⚠️⚠️⚠️以下操作为选修⚠️⚠️⚠️⚠️⚠️<br>
⚠️⚠️⚠️⚠️⚠️以下操作为选修⚠️⚠️⚠️⚠️⚠️<br>
⚠️⚠️⚠️⚠️⚠️以下操作为选修⚠️⚠️⚠️⚠️⚠️<br>
⚠️⚠️⚠️⚠️⚠️以下操作为选修⚠️⚠️⚠️⚠️⚠️
</div>
### 进阶:签名构建

默认编译产物用的是公开 testkey 签名：部分应用会将其视为不安全，这就是我们要签名的意义

 ⚠️ **注意事项**：签名后的包无法在签名前的系统中进行OTA更新

### 1. 生成密钥

在源码根目录执行（`subject` 里的信息改成自己的，不知道怎么改问ai）：

```bash
subject='/C=US/ST=California/L=Mountain View/O=Android/OU=Android/CN=Android/emailAddress=android@android.com'
mkdir ~/.android-certs
for cert in bluetooth cyngn-app media networkstack nfc platform releasekey sdk_sandbox shared testcert testkey verity; do \
    ./development/tools/make_key ~/.android-certs/$cert "$subject"; \
done
```
每个密钥都会提示设置密码，在这里我们直接全部回车不设

### 2. 生成 APEX 密钥

LineageOS 19.1+ 要求 APEX 使用 SHA256_RSA4096 密钥逐一生成，依旧猛按回车就行。
```bash
cp ./development/tools/make_key ~/.android-certs/
sed -i 's|2048|4096|g' ~/.android-certs/make_key

for apex in com.android.adbd com.android.adservices com.android.adservices.api com.android.appsearch com.android.art com.android.bluetooth com.android.bt com.android.btservices com.android.cellbroadcast com.android.compos com.android.configinfrastructure com.android.connectivity.resources com.android.conscrypt com.android.crashrecovery com.android.devicelock com.android.extservices com.android.graphics.pdf com.android.hardware.authsecret com.android.hardware.biometrics.face.virtual com.android.hardware.biometrics.fingerprint.virtual com.android.hardware.boot com.android.hardware.cas com.android.hardware.contexthub com.android.hardware.dumpstate com.android.hardware.gatekeeper.nonsecure com.android.hardware.neuralnetworks com.android.hardware.power com.android.hardware.rebootescrow com.android.hardware.thermal com.android.hardware.threadnetwork com.android.hardware.uwb com.android.hardware.vibrator com.android.hardware.wifi com.android.healthfitness com.android.hotspot2.osulogin com.android.i18n com.android.ipsec com.android.media com.android.media.swcodec com.android.mediaprovider com.android.nearby.halfsheet com.android.networkstack.tethering com.android.neuralnetworks com.android.nfcservices com.android.npumanager com.android.ondevicepersonalization com.android.os.statsd com.android.permission com.android.profiling com.android.resolv com.android.rkpd com.android.runtime com.android.safetycenter.resources com.android.scheduling com.android.sdkext com.android.support.apexer com.android.telephony com.android.telephonycore com.android.telephonymodules com.android.tethering com.android.tzdata com.android.uprobestats com.android.uwb com.android.uwb.resources com.android.virt com.android.vndk.current com.android.vndk.current.on_vendor com.android.webapp com.android.wifi com.android.wifi.dialog com.android.wifi.resources com.google.pixel.camera.hal com.google.pixel.vibrator.hal com.qorvo.uwb; do \
    subject='/C=US/ST=California/L=Mountain View/O=Android/OU=Android/CN='$apex'/emailAddress=android@android.com'; \
    ~/.android-certs/make_key ~/.android-certs/$apex "$subject"; \
    openssl pkcs8 -in ~/.android-certs/$apex.pk8 -inform DER -nocrypt -out ~/.android-certs/$apex.pem; \
done
```
### 3. 编译 target-files

用以下命令替代普通的 `m bacon`：

```bash
breakfast pagani user
m target-files-package otatools -j6
```

### 4. 签名并生成 OTA 包

```bash
croot
sign_target_files_apks -o -d ~/.android-certs \
    --extra_apks com.android.appsearch.apk.apk=$HOME/.android-certs/releasekey \
    --extra_apks AdServicesApk.apk=$HOME/.android-certs/releasekey \
    --extra_apks FederatedCompute.apk=$HOME/.android-certs/releasekey \
    --extra_apks HalfSheetUX.apk=$HOME/.android-certs/releasekey \
    --extra_apks HealthConnectBackupRestore.apk=$HOME/.android-certs/releasekey \
    --extra_apks HealthConnectController.apk=$HOME/.android-certs/releasekey \
    --extra_apks OsuLogin.apk=$HOME/.android-certs/releasekey \
    --extra_apks SafetyCenterResources.apk=$HOME/.android-certs/releasekey \
    --extra_apks ServiceConnectivityResources.apk=$HOME/.android-certs/releasekey \
    --extra_apks ServiceUwbResources.apk=$HOME/.android-certs/releasekey \
    --extra_apks ServiceWifiResources.apk=$HOME/.android-certs/releasekey \
    --extra_apks TelecomServiceResources.apk=$HOME/.android-certs/releasekey \
    --extra_apks TelecomUi.apk=$HOME/.android-certs/releasekey \
    --extra_apks WebAppService.apk=$HOME/.android-certs/releasekey \
    --extra_apks WifiDialog.apk=$HOME/.android-certs/releasekey \
    --extra_apks com.android.adbd.apex=$HOME/.android-certs/com.android.adbd \
    --extra_apks com.android.adservices.apex=$HOME/.android-certs/com.android.adservices \
    --extra_apks com.android.adservices.api.apex=$HOME/.android-certs/com.android.adservices.api \
    --extra_apks com.android.appsearch.apex=$HOME/.android-certs/com.android.appsearch \
    --extra_apks com.android.art.apex=$HOME/.android-certs/com.android.art \
    --extra_apks com.android.bluetooth.apex=$HOME/.android-certs/com.android.bluetooth \
    --extra_apks com.android.bt.apex=$HOME/.android-certs/com.android.bt \
    --extra_apks com.android.btservices.apex=$HOME/.android-certs/com.android.btservices \
    --extra_apks com.android.cellbroadcast.apex=$HOME/.android-certs/com.android.cellbroadcast \
    --extra_apks com.android.compos.apex=$HOME/.android-certs/com.android.compos \
    --extra_apks com.android.configinfrastructure.apex=$HOME/.android-certs/com.android.configinfrastructure \
    --extra_apks com.android.connectivity.resources.apex=$HOME/.android-certs/com.android.connectivity.resources \
    --extra_apks com.android.conscrypt.apex=$HOME/.android-certs/com.android.conscrypt \
    --extra_apks com.android.crashrecovery.apex=$HOME/.android-certs/com.android.crashrecovery \
    --extra_apks com.android.devicelock.apex=$HOME/.android-certs/com.android.devicelock \
    --extra_apks com.android.extservices.apex=$HOME/.android-certs/com.android.extservices \
    --extra_apks com.android.graphics.pdf.apex=$HOME/.android-certs/com.android.graphics.pdf \
    --extra_apks com.android.hardware.authsecret.apex=$HOME/.android-certs/com.android.hardware.authsecret \
    --extra_apks com.android.hardware.biometrics.face.virtual.apex=$HOME/.android-certs/com.android.hardware.biometrics.face.virtual \
    --extra_apks com.android.hardware.biometrics.fingerprint.virtual.apex=$HOME/.android-certs/com.android.hardware.biometrics.fingerprint.virtual \
    --extra_apks com.android.hardware.boot.apex=$HOME/.android-certs/com.android.hardware.boot \
    --extra_apks com.android.hardware.cas.apex=$HOME/.android-certs/com.android.hardware.cas \
    --extra_apks com.android.hardware.contexthub.apex=$HOME/.android-certs/com.android.hardware.contexthub \
    --extra_apks com.android.hardware.dumpstate.apex=$HOME/.android-certs/com.android.hardware.dumpstate \
    --extra_apks com.android.hardware.gatekeeper.nonsecure.apex=$HOME/.android-certs/com.android.hardware.gatekeeper.nonsecure \
    --extra_apks com.android.hardware.neuralnetworks.apex=$HOME/.android-certs/com.android.hardware.neuralnetworks \
    --extra_apks com.android.hardware.power.apex=$HOME/.android-certs/com.android.hardware.power \
    --extra_apks com.android.hardware.rebootescrow.apex=$HOME/.android-certs/com.android.hardware.rebootescrow \
    --extra_apks com.android.hardware.thermal.apex=$HOME/.android-certs/com.android.hardware.thermal \
    --extra_apks com.android.hardware.threadnetwork.apex=$HOME/.android-certs/com.android.hardware.threadnetwork \
    --extra_apks com.android.hardware.uwb.apex=$HOME/.android-certs/com.android.hardware.uwb \
    --extra_apks com.android.hardware.vibrator.apex=$HOME/.android-certs/com.android.hardware.vibrator \
    --extra_apks com.android.hardware.wifi.apex=$HOME/.android-certs/com.android.hardware.wifi \
    --extra_apks com.android.healthfitness.apex=$HOME/.android-certs/com.android.healthfitness \
    --extra_apks com.android.hotspot2.osulogin.apex=$HOME/.android-certs/com.android.hotspot2.osulogin \
    --extra_apks com.android.i18n.apex=$HOME/.android-certs/com.android.i18n \
    --extra_apks com.android.ipsec.apex=$HOME/.android-certs/com.android.ipsec \
    --extra_apks com.android.media.apex=$HOME/.android-certs/com.android.media \
    --extra_apks com.android.media.swcodec.apex=$HOME/.android-certs/com.android.media.swcodec \
    --extra_apks com.android.mediaprovider.apex=$HOME/.android-certs/com.android.mediaprovider \
    --extra_apks com.android.nearby.halfsheet.apex=$HOME/.android-certs/com.android.nearby.halfsheet \
    --extra_apks com.android.networkstack.tethering.apex=$HOME/.android-certs/com.android.networkstack.tethering \
    --extra_apks com.android.neuralnetworks.apex=$HOME/.android-certs/com.android.neuralnetworks \
    --extra_apks com.android.nfcservices.apex=$HOME/.android-certs/com.android.nfcservices \
    --extra_apks com.android.npumanager.apex=$HOME/.android-certs/com.android.npumanager \
    --extra_apks com.android.ondevicepersonalization.apex=$HOME/.android-certs/com.android.ondevicepersonalization \
    --extra_apks com.android.os.statsd.apex=$HOME/.android-certs/com.android.os.statsd \
    --extra_apks com.android.permission.apex=$HOME/.android-certs/com.android.permission \
    --extra_apks com.android.profiling.apex=$HOME/.android-certs/com.android.profiling \
    --extra_apks com.android.resolv.apex=$HOME/.android-certs/com.android.resolv \
    --extra_apks com.android.rkpd.apex=$HOME/.android-certs/com.android.rkpd \
    --extra_apks com.android.runtime.apex=$HOME/.android-certs/com.android.runtime \
    --extra_apks com.android.safetycenter.resources.apex=$HOME/.android-certs/com.android.safetycenter.resources \
    --extra_apks com.android.scheduling.apex=$HOME/.android-certs/com.android.scheduling \
    --extra_apks com.android.sdkext.apex=$HOME/.android-certs/com.android.sdkext \
    --extra_apks com.android.support.apexer.apex=$HOME/.android-certs/com.android.support.apexer \
    --extra_apks com.android.telephony.apex=$HOME/.android-certs/com.android.telephony \
    --extra_apks com.android.telephonycore.apex=$HOME/.android-certs/com.android.telephonycore \
    --extra_apks com.android.telephonymodules.apex=$HOME/.android-certs/com.android.telephonymodules \
    --extra_apks com.android.tethering.apex=$HOME/.android-certs/com.android.tethering \
    --extra_apks com.android.tzdata.apex=$HOME/.android-certs/com.android.tzdata \
    --extra_apks com.android.uprobestats.apex=$HOME/.android-certs/com.android.uprobestats \
    --extra_apks com.android.uwb.apex=$HOME/.android-certs/com.android.uwb \
    --extra_apks com.android.uwb.resources.apex=$HOME/.android-certs/com.android.uwb.resources \
    --extra_apks com.android.virt.apex=$HOME/.android-certs/com.android.virt \
    --extra_apks com.android.vndk.current.apex=$HOME/.android-certs/com.android.vndk.current \
    --extra_apks com.android.vndk.current.on_vendor.apex=$HOME/.android-certs/com.android.vndk.current.on_vendor \
    --extra_apks com.android.webapp.apex=$HOME/.android-certs/com.android.webapp \
    --extra_apks com.android.wifi.apex=$HOME/.android-certs/com.android.wifi \
    --extra_apks com.android.wifi.dialog.apex=$HOME/.android-certs/com.android.wifi.dialog \
    --extra_apks com.android.wifi.resources.apex=$HOME/.android-certs/com.android.wifi.resources \
    --extra_apks com.google.pixel.camera.hal.apex=$HOME/.android-certs/com.google.pixel.camera.hal \
    --extra_apks com.google.pixel.vibrator.hal.apex=$HOME/.android-certs/com.google.pixel.vibrator.hal \
    --extra_apks com.qorvo.uwb.apex=$HOME/.android-certs/com.qorvo.uwb \
    --extra_apex_payload_key com.android.adbd.apex=$HOME/.android-certs/com.android.adbd.pem \
    --extra_apex_payload_key com.android.adservices.apex=$HOME/.android-certs/com.android.adservices.pem \
    --extra_apex_payload_key com.android.adservices.api.apex=$HOME/.android-certs/com.android.adservices.api.pem \
    --extra_apex_payload_key com.android.appsearch.apex=$HOME/.android-certs/com.android.appsearch.pem \
    --extra_apex_payload_key com.android.art.apex=$HOME/.android-certs/com.android.art.pem \
    --extra_apex_payload_key com.android.bluetooth.apex=$HOME/.android-certs/com.android.bluetooth.pem \
    --extra_apex_payload_key com.android.bt.apex=$HOME/.android-certs/com.android.bt.pem \
    --extra_apex_payload_key com.android.btservices.apex=$HOME/.android-certs/com.android.btservices.pem \
    --extra_apex_payload_key com.android.cellbroadcast.apex=$HOME/.android-certs/com.android.cellbroadcast.pem \
    --extra_apex_payload_key com.android.compos.apex=$HOME/.android-certs/com.android.compos.pem \
    --extra_apex_payload_key com.android.configinfrastructure.apex=$HOME/.android-certs/com.android.configinfrastructure.pem \
    --extra_apex_payload_key com.android.connectivity.resources.apex=$HOME/.android-certs/com.android.connectivity.resources.pem \
    --extra_apex_payload_key com.android.conscrypt.apex=$HOME/.android-certs/com.android.conscrypt.pem \
    --extra_apex_payload_key com.android.crashrecovery.apex=$HOME/.android-certs/com.android.crashrecovery.pem \
    --extra_apex_payload_key com.android.devicelock.apex=$HOME/.android-certs/com.android.devicelock.pem \
    --extra_apex_payload_key com.android.extservices.apex=$HOME/.android-certs/com.android.extservices.pem \
    --extra_apex_payload_key com.android.graphics.pdf.apex=$HOME/.android-certs/com.android.graphics.pdf.pem \
    --extra_apex_payload_key com.android.hardware.authsecret.apex=$HOME/.android-certs/com.android.hardware.authsecret.pem \
    --extra_apex_payload_key com.android.hardware.biometrics.face.virtual.apex=$HOME/.android-certs/com.android.hardware.biometrics.face.virtual.pem \
    --extra_apex_payload_key com.android.hardware.biometrics.fingerprint.virtual.apex=$HOME/.android-certs/com.android.hardware.biometrics.fingerprint.virtual.pem \
    --extra_apex_payload_key com.android.hardware.boot.apex=$HOME/.android-certs/com.android.hardware.boot.pem \
    --extra_apex_payload_key com.android.hardware.cas.apex=$HOME/.android-certs/com.android.hardware.cas.pem \
    --extra_apex_payload_key com.android.hardware.contexthub.apex=$HOME/.android-certs/com.android.hardware.contexthub.pem \
    --extra_apex_payload_key com.android.hardware.dumpstate.apex=$HOME/.android-certs/com.android.hardware.dumpstate.pem \
    --extra_apex_payload_key com.android.hardware.gatekeeper.nonsecure.apex=$HOME/.android-certs/com.android.hardware.gatekeeper.nonsecure.pem \
    --extra_apex_payload_key com.android.hardware.neuralnetworks.apex=$HOME/.android-certs/com.android.hardware.neuralnetworks.pem \
    --extra_apex_payload_key com.android.hardware.power.apex=$HOME/.android-certs/com.android.hardware.power.pem \
    --extra_apex_payload_key com.android.hardware.rebootescrow.apex=$HOME/.android-certs/com.android.hardware.rebootescrow.pem \
    --extra_apex_payload_key com.android.hardware.thermal.apex=$HOME/.android-certs/com.android.hardware.thermal.pem \
    --extra_apex_payload_key com.android.hardware.threadnetwork.apex=$HOME/.android-certs/com.android.hardware.threadnetwork.pem \
    --extra_apex_payload_key com.android.hardware.uwb.apex=$HOME/.android-certs/com.android.hardware.uwb.pem \
    --extra_apex_payload_key com.android.hardware.vibrator.apex=$HOME/.android-certs/com.android.hardware.vibrator.pem \
    --extra_apex_payload_key com.android.hardware.wifi.apex=$HOME/.android-certs/com.android.hardware.wifi.pem \
    --extra_apex_payload_key com.android.healthfitness.apex=$HOME/.android-certs/com.android.healthfitness.pem \
    --extra_apex_payload_key com.android.hotspot2.osulogin.apex=$HOME/.android-certs/com.android.hotspot2.osulogin.pem \
    --extra_apex_payload_key com.android.i18n.apex=$HOME/.android-certs/com.android.i18n.pem \
    --extra_apex_payload_key com.android.ipsec.apex=$HOME/.android-certs/com.android.ipsec.pem \
    --extra_apex_payload_key com.android.media.apex=$HOME/.android-certs/com.android.media.pem \
    --extra_apex_payload_key com.android.media.swcodec.apex=$HOME/.android-certs/com.android.media.swcodec.pem \
    --extra_apex_payload_key com.android.mediaprovider.apex=$HOME/.android-certs/com.android.mediaprovider.pem \
    --extra_apex_payload_key com.android.nearby.halfsheet.apex=$HOME/.android-certs/com.android.nearby.halfsheet.pem \
    --extra_apex_payload_key com.android.networkstack.tethering.apex=$HOME/.android-certs/com.android.networkstack.tethering.pem \
    --extra_apex_payload_key com.android.neuralnetworks.apex=$HOME/.android-certs/com.android.neuralnetworks.pem \
    --extra_apex_payload_key com.android.nfcservices.apex=$HOME/.android-certs/com.android.nfcservices.pem \
    --extra_apex_payload_key com.android.npumanager.apex=$HOME/.android-certs/com.android.npumanager.pem \
    --extra_apex_payload_key com.android.ondevicepersonalization.apex=$HOME/.android-certs/com.android.ondevicepersonalization.pem \
    --extra_apex_payload_key com.android.os.statsd.apex=$HOME/.android-certs/com.android.os.statsd.pem \
    --extra_apex_payload_key com.android.permission.apex=$HOME/.android-certs/com.android.permission.pem \
    --extra_apex_payload_key com.android.profiling.apex=$HOME/.android-certs/com.android.profiling.pem \
    --extra_apex_payload_key com.android.resolv.apex=$HOME/.android-certs/com.android.resolv.pem \
    --extra_apex_payload_key com.android.rkpd.apex=$HOME/.android-certs/com.android.rkpd.pem \
    --extra_apex_payload_key com.android.runtime.apex=$HOME/.android-certs/com.android.runtime.pem \
    --extra_apex_payload_key com.android.safetycenter.resources.apex=$HOME/.android-certs/com.android.safetycenter.resources.pem \
    --extra_apex_payload_key com.android.scheduling.apex=$HOME/.android-certs/com.android.scheduling.pem \
    --extra_apex_payload_key com.android.sdkext.apex=$HOME/.android-certs/com.android.sdkext.pem \
    --extra_apex_payload_key com.android.support.apexer.apex=$HOME/.android-certs/com.android.support.apexer.pem \
    --extra_apex_payload_key com.android.telephony.apex=$HOME/.android-certs/com.android.telephony.pem \
    --extra_apex_payload_key com.android.telephonycore.apex=$HOME/.android-certs/com.android.telephonycore.pem \
    --extra_apex_payload_key com.android.telephonymodules.apex=$HOME/.android-certs/com.android.telephonymodules.pem \
    --extra_apex_payload_key com.android.tethering.apex=$HOME/.android-certs/com.android.tethering.pem \
    --extra_apex_payload_key com.android.tzdata.apex=$HOME/.android-certs/com.android.tzdata.pem \
    --extra_apex_payload_key com.android.uprobestats.apex=$HOME/.android-certs/com.android.uprobestats.pem \
    --extra_apex_payload_key com.android.uwb.apex=$HOME/.android-certs/com.android.uwb.pem \
    --extra_apex_payload_key com.android.uwb.resources.apex=$HOME/.android-certs/com.android.uwb.resources.pem \
    --extra_apex_payload_key com.android.virt.apex=$HOME/.android-certs/com.android.virt.pem \
    --extra_apex_payload_key com.android.vndk.current.apex=$HOME/.android-certs/com.android.vndk.current.pem \
    --extra_apex_payload_key com.android.vndk.current.on_vendor.apex=$HOME/.android-certs/com.android.vndk.current.on_vendor.pem \
    --extra_apex_payload_key com.android.webapp.apex=$HOME/.android-certs/com.android.webapp.pem \
    --extra_apex_payload_key com.android.wifi.apex=$HOME/.android-certs/com.android.wifi.pem \
    --extra_apex_payload_key com.android.wifi.dialog.apex=$HOME/.android-certs/com.android.wifi.dialog.pem \
    --extra_apex_payload_key com.android.wifi.resources.apex=$HOME/.android-certs/com.android.wifi.resources.pem \
    --extra_apex_payload_key com.google.pixel.camera.hal.apex=$HOME/.android-certs/com.google.pixel.camera.hal.pem \
    --extra_apex_payload_key com.google.pixel.vibrator.hal.apex=$HOME/.android-certs/com.google.pixel.vibrator.hal.pem \
    --extra_apex_payload_key com.qorvo.uwb.apex=$HOME/.android-certs/com.qorvo.uwb.pem \
    $OUT/obj/PACKAGING/target_files_intermediates/*-target_files*.zip \
    signed-target_files.zip
```
# 生成 OTA 包
```bash
ota_from_target_files -k ~/.android-certs/releasekey \
    signed-target_files.zip \
    signed-lineage-23.2-pagani.zip
```
输出包在源码目录下
