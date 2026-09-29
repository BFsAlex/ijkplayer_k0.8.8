# Build_DIY —— 自定义操作手册目录

本目录存放围绕本仓库（branch_ArmAndSim__SSL 分支）整理的实操手册，
记录环境搭建、源码编译、产物封装、问题排查等可复现的操作步骤。

## 手册索引

| 手册 | 内容 |
|---|---|
| [BUILD-IOS.md](BUILD-IOS.md) | iOS SDK 编译全流程：依赖初始化 → OpenSSL/FFmpeg 多架构编译 → Framework / xcframework 封装、App 集成与常见问题排查（真机 arm64 + 模拟器 arm64/x86_64） |

## 后续新增手册的建议

1. 命名统一为 `BUILD-<平台或主题>.md`，例如：
   - `BUILD-ANDROID.md`（Android so 编译）
   - `BUILD-MACOS.md`（macOS 平台移植）
   - `BUILD-RELEASE.md`（发版 / 打 tag / 推送流程）
2. 每份手册保持统一结构：**环境要求 → 获取代码 → 初始化 → 编译 → 产物与自检 → 集成使用 → 常见问题**；
3. 新增手册后在上面索引表中登记一行，方便查找；
4. 手册中写"实机验证过的"命令请注明验证环境（macOS / Xcode 版本），避免过时误导。
