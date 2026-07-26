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

## 前置要求

- 已初始化的 LineageOS 源码树（`lineage-23.2` 分支，完成 `repo sync`）
- 正常可用的 AOSP 编译环境
- 磁盘空间：**约 330 GB 以上**（源码 + 编译产物）
- 可用内存建议 32 GB 以上（含 swap/zram，见文末参考配置）

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
cd /path/to/source(源码路径)
patch -p1 -i ~/path/to/patch(补丁路径)/android_develop/0001-tmp-add-trigger-for-udfps.patch

cd /path/to/source(源码路径)
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

准备一份 ColorOS（COS）或 OxygenOS（OOS）固件的提取目录（下称 *dump*），然后：

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

## 参考配置与编译资源

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
