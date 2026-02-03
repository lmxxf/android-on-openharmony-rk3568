# Android on OpenHarmony (RK3568)

在 OpenHarmony 6.0 上运行 Android 容器的实验项目。

## 项目状态

**当前进度**：Android 13 Shell 已成功启动 ✓

## 文档

- [技术方案](./Android-on-OpenHarmony方案.md) - 方案对比、架构设计、权限分析
- [开发日志](./DevHistory.md) - 详细的开发过程记录和命令

## 技术路线

采用 **iSulad/容器化** 方案，而非 Docker + 虚拟机。

核心思路：OpenHarmony 和 Android 共享 Linux 内核，用容器隔离 Android 用户态。

## 测试环境

| 项目 | 配置 |
|------|------|
| 硬件 | RK3568 开发板 |
| 系统 | OpenHarmony 6.0 |
| 内核 | Linux 6.6.101 aarch64 |
| Android | redroid 13.0.0 (arm64) |

## 快速开始

详见 [DevHistory.md](./DevHistory.md)

## License

MIT
