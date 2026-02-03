# Android-on-OpenHarmony 开发日志

## 2026-02-03 项目启动

### 背景
公司任务：调研 OpenHarmony 上运行 Android 的可行性。

同事建议「Docker 移植到 OH，然后在 Docker 里跑 Android 虚拟机」—— 概念混淆，Docker 是容器不是虚拟机。

### 方案确定
**推荐方案：iSulad + 容器化 Android**

- 利用 OH 与 Android 共享 Linux 内核
- iSulad 比 Docker 轻量（C 写的，~15MB vs ~100MB）
- 华为生态内技术栈

详见 [Android-on-OpenHarmony方案.md](./Android-on-OpenHarmony方案.md)

### 权限分析
**结论：必须是系统级集成，不能作为普通 App**

容器需要 namespace、cgroup、mount 等内核特性，普通 App 沙箱里拿不到这些权限。

---

## 2026-02-03 环境检测

### 测试设备
- 硬件：RK3568 开发板
- 系统：OpenHarmony 6.0
- 内核：Linux 6.6.101 aarch64

### 检测命令

```bash
# 连接设备
hdc shell

# 1. 检查内核 namespace/cgroup 支持
cat /proc/config.gz | gunzip | grep -E "CONFIG_(NAMESPACES|CGROUP|NET_NS|PID_NS|USER_NS)"

# 2. 检查 Binder 驱动
ls -la /dev/binder* /dev/binderfs/ 2>/dev/null
cat /proc/config.gz | gunzip | grep BINDER

# 3. 检查 GPU 设备
ls -la /dev/dri/

# 4. 检查可用内存
free -h

# 5. 内核版本
uname -a
```

### 检测结果

通过 `hdc shell` 检测：

| 检查项 | 状态 | 详情 |
|--------|------|------|
| Namespace | ✅ | CONFIG_NAMESPACES=y, USER_NS=y, PID_NS=y, NET_NS=y |
| Cgroup | ✅ | CONFIG_CGROUPS=y, CGROUP_PIDS=y, CGROUP_FREEZER=y |
| Binder | ✅ | `/dev/binder` 存在，CONFIG_ANDROID_BINDER_IPC=y |
| GPU | ✅ | `/dev/dri/card0`, `/dev/dri/renderD128` (Mali) |
| 内存 | ⚠️ | 2G 总内存，可用 ~200MB，较紧张 |

**结论：内核条件完全满足跑容器化 Android。**

### 下一步计划

两条路：
1. **快速验证**：用 `unshare`/`nsenter` 手动起 Android 最小环境（零依赖）
2. **正式路线**：交叉编译 iSulad 到 aarch64，移植到 OH

决定先走快速验证路线，确认可行性后再投入 iSulad 移植工作。

---

## 2026-02-03 获取 Android rootfs

### 问题
Docker Hub 被墙，`docker pull` 超时。

### 解决方案：用 skopeo 绕过 Docker daemon

```bash
# 安装 skopeo
sudo apt update && sudo apt install -y skopeo

# 查看 redroid 可用 tag
skopeo list-tags docker://redroid/redroid

# 下载 arm64 镜像（redroid 是多架构镜像，需要 --override-arch 指定）
skopeo copy --override-arch arm64 docker://redroid/redroid:13.0.0-latest dir:./redroid-img

# 或者 Android 14
skopeo copy --override-arch arm64 docker://redroid/redroid:14.0.0-latest dir:./redroid-img
```

### 解包 rootfs

```bash
# 查看下载的文件
ls -lah ./redroid-img/

# 解包 blob（725MB 那个文件）为 rootfs
mkdir -p ./redroid-rootfs
tar -xf ./redroid-img/4fb8aea32c9d378f7832bcd29c3853125ac6a30d293da87f2abc0d87bf0017c8 -C ./redroid-rootfs

# 验证
ls ./redroid-rootfs/
```

### 结果
成功获取 Android 13 rootfs，包含完整目录结构：
`init`, `system`, `vendor`, `apex`, `bin`, `etc`, `dev`, `proc`, `sys` 等。

---

## 2026-02-03 传输 rootfs 到设备

### 问题
1. hdc 直接传目录失败 —— Windows 不识别 Linux 符号链接
2. OH 的 tar 不支持 `-C` 参数报错 `not under` 警告

### 解决方案

```bash
# WSL 中打包
tar -cvf redroid-rootfs.tar -C redroid-rootfs .
```

```powershell
# PowerShell 传输
hdc file send Z:\home\lmxxf\work\docker-android-on-openharmony\redroid-rootfs.tar /data/
```

```bash
# 设备上解包（hdc shell）
mkdir -p /data/android-rootfs
cd /data/android-rootfs
toybox tar -xf /data/redroid-rootfs.tar
# 忽略 "not under" 警告，实际解压成功
ls  # 验证文件存在
```

### 结果
rootfs 成功传输到设备 `/data/android-rootfs/`

---

## 2026-02-03 首次启动 Android Shell ✓

### 关键发现
1. Android 13 使用 APEX 模块化，linker64 在 `/apex/com.android.runtime/bin/linker64`
2. rootfs 的 `/apex/` 目录是空的，需要手动挂载

### 启动步骤

```bash
cd /data/android-rootfs

# 1. 挂载虚拟文件系统
mount -t proc proc ./proc
mount -t sysfs sysfs ./sys
mount -t tmpfs tmpfs ./dev
mkdir -p ./dev/pts
mount -t devpts devpts ./dev/pts

# 2. 创建设备节点
mknod ./dev/null c 1 3
mknod ./dev/zero c 1 5
mknod ./dev/random c 1 8
mknod ./dev/urandom c 1 9
mknod ./dev/binder c 10 121
mknod ./dev/hwbinder c 10 120
mknod ./dev/vndbinder c 10 119
chmod 666 ./dev/null ./dev/zero ./dev/random ./dev/urandom ./dev/binder ./dev/hwbinder ./dev/vndbinder

# 3. 挂载 APEX（关键！）
mkdir -p ./apex/com.android.runtime
mount --bind ./system/apex/com.android.runtime ./apex/com.android.runtime

# 4. 进入 Android shell
unshare --mount --pid --fork chroot /data/android-rootfs /system/bin/sh
```

### 结果
成功进入 Android 13 shell！

```
linker: Warning: failed to find generated linker configuration from "/linkerconfig/ld.config.txt"
/system/bin/sh: No controlling tty: open /dev/tty: No such file or directory
/system/bin/sh: warning: won't have full job control
:/ #
```

---

## 2026-02-03 14:01 servicemanager 启动成功 ✓

### 问题
Android init 跑完初始化后立即退出，没有持续运行服务。

### 分析
1. `/data` 目录为空且不可写
2. 缺少 cgroup 挂载（memcg、cpuctl）
3. 只挂载了 runtime APEX，缺少 art、i18n、conscrypt 等
4. 缺少 `/dev/tty` 设备
5. 缺少 `/linkerconfig/ld.config.txt`

### 解决方案

```bash
cd /data/android-rootfs

# 1. 让 /data 可写
mount -t tmpfs tmpfs ./data
mkdir -p ./data/local/tmp
chmod 777 ./data/local/tmp

# 2. 挂载 cgroup
mkdir -p ./dev/memcg
mount -t cgroup -o memory cgroup ./dev/memcg 2>/dev/null || mount -t tmpfs tmpfs ./dev/memcg
mkdir -p ./dev/cpuctl
mount -t cgroup -o cpu cgroup ./dev/cpuctl 2>/dev/null || mount -t tmpfs tmpfs ./dev/cpuctl

# 3. 挂载更多 APEX
mkdir -p ./apex/com.android.art
mount --bind ./system/apex/com.android.art ./apex/com.android.art
mkdir -p ./apex/com.android.i18n
mount --bind ./system/apex/com.android.i18n ./apex/com.android.i18n
mkdir -p ./apex/com.android.conscrypt
mount --bind ./system/apex/com.android.conscrypt ./apex/com.android.conscrypt

# 4. 创建 /dev/tty
mknod ./dev/tty c 5 0
chmod 666 ./dev/tty

# 5. 创建 linkerconfig
mkdir -p ./linkerconfig
touch ./linkerconfig/ld.config.txt
```

### 结果
进入 Android shell 后手动启动 servicemanager 成功：

```bash
/system/bin/servicemanager &
ps -ef | grep servicemanager
# root  23626 23621 0 06:00:14 pts/0 00:00:00 servicemanager
```

**servicemanager 是 Android Binder 服务框架的核心，它的成功运行意味着 Android 服务可以正常注册和通信。**

---

## 2026-02-03 14:05 启动更多 Android 服务

### 启动的服务

```bash
# 在 Android shell 里
/system/bin/servicemanager &
/system/bin/hwservicemanager &
/system/bin/logd &
/system/bin/surfaceflinger &
```

### 验证

```bash
ps -ef | grep -E "(servicemanager|surfaceflinger|logd)"
# root  23626 ... servicemanager
# root  23636 ... hwservicemanager
# root  23638 ... logd
# root  23649 ... surfaceflinger
```

### logd socket 创建成功

```bash
ls -la /dev/socket/
# srwxr-xr-x 1 root root 0 ... logd
# srwxr-xr-x 1 root root 0 ... logdr
# srwxr-xr-x 1 root root 0 ... logdw
```

### 图形设备补充

surfaceflinger 需要 DRI/framebuffer 设备，容器里没有，需要手动创建：

```bash
# 创建 framebuffer 设备
mkdir -p /dev/graphics
mknod /dev/graphics/fb0 c 29 0
chmod 666 /dev/graphics/fb0

# 创建 DRI 设备（GPU）
mkdir -p /dev/dri
mknod /dev/dri/card0 c 226 0
mknod /dev/dri/renderD128 c 226 128
chmod 666 /dev/dri/card0 /dev/dri/renderD128
```

### 图形输出问题

surfaceflinger 启动了，但：
- 没有打开 DRI 设备（`/proc/PID/fd` 里没有 dri）
- 屏幕仍然显示 OpenHarmony 界面
- 原因：OH 的图形栈占用了显示设备，Android 的 surfaceflinger 无法直接控制

**解决方向**：
1. 图形透传：Android 渲染 → OH 窗口系统显示
2. 软件渲染 + VNC：不用 GPU，远程显示

---

## 2026-02-03 14:17 adbd 启动成功，adb 连接成功 ✓

### 问题
需要 adb 连接来调试 Android 容器，但 adbd 不在默认路径。

### 解决方案

1. 挂载 adbd 的 APEX：

```bash
# 在 OH shell 里
cd /data/android-rootfs
mkdir -p ./apex/com.android.adbd
mount --bind ./system/apex/com.android.adbd ./apex/com.android.adbd
```

2. 网络 namespace 问题：
   - 最初用 `unshare --mount --pid --net` 创建了隔离的网络
   - adbd 监听的 5555 端口外部访问不到
   - 解决：不使用 `--net` 参数，共享宿主网络

```bash
# 正确的启动方式（共享宿主网络）
unshare --mount --pid --fork chroot /data/android-rootfs /system/bin/sh
```

3. 启动 adbd：

```bash
# 在 Android shell 里
/apex/com.android.adbd/bin/adbd &
```

### 结果

```
adbd I 02-03 06:14:02 main.cpp:187] adbd listening on tcp:5555
adbd I 02-03 06:14:02 main.cpp:296] adbd started
```

PC 端连接成功：

```powershell
PS C:\Users\lmxxf> adb connect 192.168.159.163:5555
connected to 192.168.159.163:5555
```

**这是重大突破——可以通过 adb 直接与 Android 容器交互！**

### adb 授权问题（未解决）

PC 连接后显示 `device unauthorized`，尝试了：
- 创建 `/data/misc/adb/adb_keys` 放入 PC 公钥
- 设置 `ADBD_AUTH=0` 环境变量
- 设置 `ADB_VENDOR_KEYS` 环境变量

都没用，adbd 还是要求授权：
```
adbd I auth.cpp:286] prompting user to authorize key
```

原因：容器里没有 UI 来弹授权对话框，adbd 又不认我们手动放的 keys 文件。

**解决方向**：
1. 使用 userdebug/eng 版本的 Android 镜像（默认 `ro.adb.secure=0`）
2. 或者修改 adbd 源码重新编译
3. 暂时可以用 `hdc shell` + `chroot` 替代

### 进一步尝试（均失败）

1. **添加 adb 公钥到 /data/misc/adb/adb_keys** —— adbd 不读取
2. **设置环境变量 ADBD_AUTH=0** —— 无效
3. **sed patch adbd 二进制替换 ro.adb.secure** —— 无效

根本原因分析：
- `getprop ro.adb.secure` 返回空值
- property service 没有运行，property 文件只有结构没有数据
- adbd 检查 property 失败后走默认的"需要授权"路径
- 镜像本身 `/system/system_ext/etc/build.prop` 里是 `ro.adb.secure=0`，但没被加载

**结论**：需要让 init 完整运行来初始化 property system，或者用 magisk resetprop 工具。暂时用 `hdc shell` + `chroot` 替代。

---

## 2026-02-03 14:50 尝试完整启动 init

### 问题
`/system/bin/init second_stage` 启动后立即崩溃，返回码 139（SIGSEGV）。

dmesg 显示：
```
init: init second stage started!
init: Failed to umount /debug_ramdisk: Invalid argument
```

然后就段错误退出了。

### 发现
查看 redroid 镜像的配置文件，发现正确的启动方式：

```json
{
  "Entrypoint": ["/init", "qemu=1", "androidboot.hardware=redroid"]
}
```

redroid 需要特定的内核参数：
- `qemu=1` - 告诉 Android 运行在虚拟/容器环境
- `androidboot.hardware=redroid` - 指定硬件类型

### 尝试正确的启动命令

```bash
unshare --mount --pid --fork chroot /data/android-rootfs /init qemu=1 androidboot.hardware=redroid &
```

### 结果
init 进入 second stage 但仍然崩溃（返回码 139）。

新的错误信息：
```
init: Failed to create FirstStageMount failed to read default fstab for first stage mount
init: Failed to mount required partitions early ...
init: init second stage started!
```

init 在找 fstab 文件失败。

### 下一步
创建空的 fstab 文件：
```bash
touch /data/android-rootfs/vendor/etc/fstab.redroid
touch /data/android-rootfs/fstab.redroid
mkdir -p /data/android-rootfs/first_stage_ramdisk
touch /data/android-rootfs/first_stage_ramdisk/fstab.redroid
```

---

## 2026-02-03 15:10 尝试 patch adbd 绕过授权

### 尝试
1. 找到 `ro.adb.secure` 字符串位置：
```bash
strings -t x /apex/com.android.adbd/bin/adbd | grep "ro.adb.secure"
# 输出: 13b8a ro.adb.secure
```

2. 用 sed 替换：
```bash
cp /apex/com.android.adbd/bin/adbd /data/adbd_patched
sed -i 's/ro\.adb\.secure/ro.adb.XXXXXXX/g' /data/adbd_patched
```

3. 验证替换成功：
```bash
strings /data/adbd_patched | grep -E "ro.adb.(secure|XXXXXXX)"
# 输出: ro.adb.XXXXXXX
```

### 结果
sed 替换破坏了 ELF 文件结构：
```
CANNOT LINK EXECUTABLE "/data/adbd_patched": empty/missing DT_HASH/DT_GNU_HASH
```

旧的 adbd 进程还在跑，但仍然检查授权 —— 说明 `ro.adb.secure` 不是唯一检查点。

### 结论
adb 授权问题需要：
1. property service 正常运行（init 完整启动）
2. 或重新编译 adbd 禁用授权
3. 或使用修改过的 redroid 镜像

**当前可行替代方案**：用 `hdc shell` + `chroot` 进入 Android，功能等价于 adb shell。

---

## 2026-02-03 15:30 二进制 Patch adbd 绕过授权

### 思路
用 xxd 精确修改 adbd 二进制，绕过 auth 检查。

### 分析过程

1. 导出反汇编：
```bash
sudo apt install -y binutils-aarch64-linux-gnu
aarch64-linux-gnu-objdump -d redroid-rootfs/system/apex/com.android.adbd/bin/adbd > /tmp/adbd.asm
```

2. 找到关键代码（adbd_main 函数）：
```
9a1bc:  adrp x9, 13000           # 加载 "ro.adb.secure" 地址
9a1c4:  add x9, x9, #0xb8a
9a1e8:  bl GetBoolProperty       # 读取 ro.adb.secure
9a1f4:  and w9, w0, #0x1         # 取结果最低位
9a1fc:  strb w9, [x10]           # 存储到 auth_required 全局变量
```

3. **Patch 1**：把 `and w9, w0, #0x1` 改成 `mov w9, wzr`（强制返回 0）
```bash
# 原指令: 12000009 (and w9, w0, #0x1)
# 新指令: 2a1f03e9 (mov w9, wzr)
printf '\xe9\x03\x1f\x2a' | dd of=adbd bs=1 seek=$((0x9a1f4)) conv=notrunc
```

4. **Patch 2**：发现还有第二道检查在 `handle_packet` 函数里
```
a131c:  ldrb w8, [x24]           # 加载 auth_required
a1320:  tbnz w8, #0, a1938       # 如果 != 0，跳转到 auth 处理
```

把 `tbnz` 改成 `nop`：
```bash
# 原指令: 370030c8 (tbnz w8, #0, a1938)
# 新指令: d503201f (nop)
printf '\x1f\x20\x03\xd5' | dd of=adbd bs=1 seek=$((0xa1320)) conv=notrunc
```

### 结果
Patch 后仍然显示 `prompting user to authorize key`。

原因分析：auth 流程有多层检查，`adbd_auth_confirm_key` 函数会被调用，它直接请求用户授权，不经过 auth_required 检查。

### 结论
二进制 patch 方式太复杂，放弃。**adb 授权问题不是必须解决的**——`hdc shell` + `chroot` 功能上等价于 `adb shell`，足够用于调试。

---

## 2026-02-03 15:50 调试 init 崩溃

### 关键发现：SELinux 导致问题

dmesg 里大量 `avc: denied` 日志，OpenHarmony 的 SELinux 是 Enforcing 模式，阻止了 Android init 的操作。

### 解决方案

```bash
# 检查 SELinux 状态
getenforce
# 输出: Enforcing

# 临时关闭 SELinux
setenforce 0
getenforce
# 输出: Permissive
```

### init 启动进展

关闭 SELinux 后，init 可以跑到 second stage：

```
init: init first stage started!
init: [libfs_mgr]ReadFstabFromDt(): failed to read fstab from dt
init: [libfs_mgr]ReadDefaultFstab(): failed to find device default fstab
init: Failed to create FirstStageMount failed to read default fstab for first stage mount
init: Failed to mount required partitions early ...
init: init second stage started!
```

但仍然返回 139 (SIGSEGV) 退出。

### 当前卡点

init 在 second stage 启动后崩溃，可能原因：
1. fstab 文件格式不对（目前只有注释）
2. 缺少某些必要的环境配置

### 下一步
修复 fstab 文件格式，继续调试 init second stage。

---

## 2026-02-03 16:20 用 strace 找到 init 崩溃根因

### 分析过程

用 strace 跟踪 init 崩溃：

```bash
unshare --mount --pid --fork chroot /data/android-rootfs /system/bin/strace -f -o /data/init_strace.log /init qemu=1 androidboot.hardware=redroid
```

### 发现

strace 日志最后几行：
```
3     munmap(0x7f957e1000, 73240)       = 0
3     munmap(0x7f95976000, 10520)       = 0
3     --- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_MAPERR, si_addr=0x14} ---
```

**`si_addr=0x14`** = 空指针解引用（访问 NULL + 偏移量 20）

init 在初始化 property system 时，所有 property 文件返回 `EEXIST`（已存在），导致某个指针没初始化就被使用。

### 解决方案

清空 `/dev/__properties__/` 目录，让 init 自己创建：

```bash
rm -rf /data/android-rootfs/dev/__properties__/*
```

### 结果

清空后 init 进入 second stage 并开始正常运行！

---

## 2026-02-03 16:30 init second stage 启动成功 ✓

### 突破性进展

清空 `/dev/*` 后，init 成功进入 second stage 并开始初始化 property system：

```
init: init second stage started!
init: Setting product property ro.product.brand to 'redroid'
init: Setting product property ro.product.model to 'redroid13_arm64'
init: Created socket '/dev/socket/property_service', mode 666
```

但崩溃在 remount 根文件系统：

```
init: Failed to remount / as 104000: Invalid argument
init: SetupMountNamespaces failed: Invalid argument
```

### 解决 remount 问题

清空 `/dev` 让 init 自己创建设备节点和挂载：

```bash
rm -rf /data/android-rootfs/dev/*
unshare --mount --pid --fork chroot /data/android-rootfs /init qemu=1 androidboot.hardware=redroid
```

### 新问题：vold 崩溃

init 成功启动了大量服务：
- system_suspend
- keystore2
- vendor.keymaster-4-1
- mediametrics, media, storaged, wificond
- vendor.media.omx, vendor.ril-daemon
- media.swcodec, statsd, gatekeeperd, apexd

但 vold (Volume Daemon) 崩溃导致系统重启：

```
init: Service 'vold' (pid 93) received signal 6
init: Service with 'reboot_on_failure' option failed, shutting down system.
init: Got shutdown_command 'reboot,vold-failed'
```

### 禁用 vold

vold 需要真实的块设备，我们容器里没有。禁用它：

```bash
# 禁用 vold.rc
mv /data/android-rootfs/system/etc/init/vold.rc /data/android-rootfs/system/etc/init/vold.rc.disabled

# 注释掉 init.rc 里的 start vold
sed -i 's/^    start vold/#    start vold/' /data/android-rootfs/system/etc/init/hw/init.rc
sed -i 's/^  start vold/#  start vold/' /data/android-rootfs/system/etc/init/hw/init.rc
```

### 当前状态

init 已经能启动大量 Android 服务，vold 是最后一个阻塞问题。

---

## 2026-02-03 16:50 彻底禁用 vold，init 持续运行成功！ ✓✓✓

### 问题分析

禁用 vold.rc 后，init 仍然报 `vold-failed` 重启。原因：
1. `vold.rc.disabled` 后缀的文件仍然被 init 解析（init 会读取所有 .rc 文件）
2. vold.rc 里有 `reboot_on_failure reboot,vold-failed` 设置

### 最终解决方案

彻底删除 vold 相关文件：

```bash
# 删除 vold.rc（不能只是改名！）
rm -f /data/android-rootfs/system/etc/init/vold.rc.disabled

# 删除 vold 可执行文件
mv /data/android-rootfs/system/bin/vold /data/android-rootfs/system/bin/vold.disabled

# 注释掉 init.rc 里的 start vold
sed -i 's/^    start vold/#    start vold/' /data/android-rootfs/system/etc/init/hw/init.rc
sed -i 's/^  start vold/#  start vold/' /data/android-rootfs/system/etc/init/hw/init.rc
```

### 结果

**🎉🎉🎉 init 持续运行，不再退出！**

启动命令：
```bash
rm -rf /data/android-rootfs/dev/*
unshare --mount --pid --fork chroot /data/android-rootfs /init qemu=1 androidboot.hardware=redroid
```

启动的服务列表（部分）：
- vendor.audio-hal
- vendor.bluetooth-1-1
- vendor.gralloc-2-0
- vendor.hwcomposer-2-1
- vendor.health-default
- vendor.wifi_hal_legacy
- audioserver
- servicemanager
- surfaceflinger
- credstore
- gpu
- ...

---

## 当前状态总结 (2026-02-03 16:50)

### 已实现 ✅
- [x] Android 13 rootfs 在 OpenHarmony 6.0 (RK3568) 上运行
- [x] Android shell 可用（通过 hdc + chroot）
- [x] SELinux 临时禁用（setenforce 0）
- [x] **init 完整启动成功！** 🎉
- [x] Property service 初始化成功
- [x] 大量 Android 服务成功启动

### 未解决
- [ ] adb 授权（init 跑起来后 property service 正常，可能现在能解决）
- [ ] 图形输出（surfaceflinger 启动了但没有显示）

### 关键技术发现

1. **SELinux 必须关闭**：`setenforce 0`
2. **`/dev/` 必须清空**：让 init 自己创建设备节点
3. **vold 必须彻底删除**：
   - 删除 `/system/etc/init/vold.rc`（不能改名！）
   - 删除 `/system/bin/vold`
   - 注释掉 init.rc 里的 `start vold`

4. **启动命令**：
   ```bash
   rm -rf /data/android-rootfs/dev/*
   unshare --mount --pid --fork chroot /data/android-rootfs /init qemu=1 androidboot.hardware=redroid
   ```

### 下一步
1. 验证 adb 授权是否能通过（property service 现在正常了）
2. 研究图形输出方案
