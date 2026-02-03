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

## 待办

- [x] 下载 redroid arm64 镜像
- [x] 解包 OCI 镜像为 rootfs
- [x] 传输 rootfs 到设备
- [x] Android shell 启动成功

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
- [ ] 用 unshare 启动最小 Android init
- [ ] 验证 Binder 驱动可用性
- [ ] 图形输出方案验证
