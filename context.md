# Android-on-OpenHarmony 项目上下文

## 项目目标
在 OpenHarmony 6.0 (RK3568) 上通过容器化方式运行 Android 13。

## 当前状态 🎉

### 重大突破：init 完整启动成功！

**Android 13 init 已经可以在 OpenHarmony 上持续运行！**

### 已完成 ✅
1. **init 完整启动** - 不再崩溃，持续运行
2. **Property service 正常** - init 初始化了所有 property
3. **大量服务已启动**：
   - servicemanager, surfaceflinger
   - vendor.audio-hal, vendor.bluetooth-1-1
   - vendor.gralloc-2-0, vendor.hwcomposer-2-1
   - audioserver, credstore, gpu 等
4. **rootfs 位置** - `/data/android-rootfs/`
5. **SELinux 已禁用** - `setenforce 0`

### 未解决 ❌
1. **adb 授权** - 待测试（property service 正常后可能能解决）
2. **图形输出** - surfaceflinger 启动了但没有显示

## 启动 Android init 的完整步骤

### 1. 前置条件
```bash
# 关闭 SELinux（必须！）
setenforce 0
```

### 2. rootfs 修改（只需做一次）
```bash
# 彻底删除 vold（容器里没有块设备，vold 会崩溃）
rm -f /data/android-rootfs/system/etc/init/vold.rc
rm -f /data/android-rootfs/system/bin/vold

# 注释掉 init.rc 里的 start vold
sed -i 's/^    start vold/#    start vold/' /data/android-rootfs/system/etc/init/hw/init.rc
sed -i 's/^  start vold/#  start vold/' /data/android-rootfs/system/etc/init/hw/init.rc

# 创建 fstab
cat > /data/android-rootfs/vendor/etc/fstab.redroid << 'EOF'
# Android fstab for redroid container
# <src>     <mnt_point>  <type>  <mnt_flags>  <fs_mgr_flags>
none        /cache       tmpfs   nosuid,nodev defaults
none        /metadata    tmpfs   nosuid,nodev defaults
EOF
```

### 3. 启动 init
```bash
# 清空 /dev（必须！让 init 自己创建设备节点）
rm -rf /data/android-rootfs/dev/*

# 挂载 APEX（init 需要 linker）
mkdir -p /data/android-rootfs/apex/com.android.runtime
mount --bind /data/android-rootfs/system/apex/com.android.runtime /data/android-rootfs/apex/com.android.runtime

# 启动 init
unshare --mount --pid --fork chroot /data/android-rootfs /init qemu=1 androidboot.hardware=redroid
```

## 关键技术发现

1. **SELinux 必须关闭** - OpenHarmony 的 SELinux 会阻止 Android init 的操作
2. **`/dev/` 必须清空** - redroid 镜像自带的 `/dev/__properties__/` 文件会导致 init 空指针崩溃
3. **vold 必须彻底删除** - 不能只改名为 `.disabled`，init 会读取所有 .rc 文件
4. **fstab 需要存在** - 但内容可以是空的或最小配置

## 调试 init 崩溃的方法

### 用 strace 跟踪
```bash
unshare --mount --pid --fork chroot /data/android-rootfs /system/bin/strace -f -o /data/init_strace.log /init qemu=1 androidboot.hardware=redroid
tail -100 /data/android-rootfs/data/init_strace.log
```

### 看 dmesg
```bash
dmesg | grep -i init | tail -50
```

## 下一步方向
1. 测试 adb 授权是否能通过
2. 研究图形输出方案（软件渲染 + VNC 或图形透传）

## 详细日志
见 [DevHistory.md](./DevHistory.md)
