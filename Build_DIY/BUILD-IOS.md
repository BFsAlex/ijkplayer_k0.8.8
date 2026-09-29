# iOS SDK 编译操作手册（branch_ArmAndSim）

适用分支：`branch_ArmAndSim`（基线 ijkplayer k0.8.8 + Bilibili FFmpeg 3.4 fork + OpenSSL 1.0.2n）

本手册记录从源码编译出 **真机 + 模拟器** 双 `IJKMediaFrameworkWithSSL` framework / xcframework 的完整流程。
所有命令已在 Xcode 26.5 / macOS 26 (Apple Silicon) 上实机验证通过；Xcode 15 及以上均可照此操作。

---

## 1. 本分支功能配置

| 能力 | 说明 |
|---|---|
| RTSP 播放 | RTSP demuxer + SDP/RTP demuxer，over UDP 与 over TCP 均支持 |
| G.711 音频 | `pcm_alaw / pcm_mulaw` 解码器（监控摄像头常见音频） |
| MJPEG | 解码器 + 解复用器（HTTP MJPEG 流） |
| HTTPS / RTSPS | OpenSSL 1.0.2n（`--enable-openssl`） |
| H.264 / HEVC | HEVC 走 VideoToolbox 硬解（沿用上游补丁） |
| 常规格式 | HLS / RTMP / MP4 / FLV / MP3 / AAC 等 module-lite 原有能力 |

产物切片：

- `ios-arm64`：真机
- `ios-arm64_x86_64-simulator`：模拟器（Apple Silicon + Intel Mac）

本分支相对上游的改动（commit `cdd5d8cf`）：

1. `config/module-lite.sh` 合并开启 RTSP/SDP/RTP、G.711、MJPEG、OpenSSL；
2. `ios/compile-ffmpeg.sh`、`ios/compile-openssl.sh`、`ios/tools/do-compile-*.sh` 新增 **arm64-sim** 架构支持；
3. 新工具链适配：min iOS 7.0→12.0、移除 `-fembed-bitcode`、注入 `-std=gnu17 -Wno-error=…` 降级标志（老代码过新版 clang）。

---

## 2. 环境要求

- macOS + Xcode 15 及以上，已安装 Command Line Tools；
- git；无需 Homebrew / CocoaPods；
- **能访问 github.com**（`init-ios.sh` / `init-ios-openssl.sh` 需要从 GitHub 拉取 FFmpeg / OpenSSL）；
- 磁盘剩余空间 ≥ 10 GB。

> ⚠️ **路径不要包含 `$` 字符**
> 诸如 `/Volumes/subaifu$/...` 这类含 `$` 的挂载路径会破坏 make 变量（OpenSSL 的 `make install` 会把 `$S` 当成空变量展开，静默安装到错误路径）。
> 如果必须放在这种卷上，先建一个不含 `$` 的软链接，后续全部经由软链接进入：
>
> ```bash
> mkdir -p ~/ijkbuild
> ln -s '/Volumes/subaifu$/SDK/IJK/Code_k0.8.8/ijkplayer' ~/ijkbuild/ijkplayer
> # 之后所有命令都在 ~/ijkbuild/ijkplayer 下执行
> ```
>
> 代码放在本机磁盘（如 ~/Desktop）则无此问题。

---

## 3. 获取代码

方式 A —— 从 GitHub 克隆（推荐）：

```bash
git clone -b branch_ArmAndSim https://github.com/BFsAlex/ijkplayer_k0.8.8.git ijkplayer
cd ijkplayer
```

方式 B —— 从文件服务器拷贝整目录（可离线编译）：

把 `SDK/IJK/Code_k0.8.8/ijkplayer` 整目录复制到本机磁盘。若 `extra/` 和 `ios/ffmpeg-*`、`ios/openssl-*`
已随目录带过来，可跳过第 4 节的初始化，直接进入第 5 节。

---

## 4. 初始化依赖（首次编译必做）

> FFmpeg / OpenSSL 源码是嵌套 git 仓库，均被 `.gitignore` 排除，**克隆代码后需要单独拉取**。

```bash
./init-ios.sh arm64        # → extra/ffmpeg + ios/ffmpeg-arm64
./init-ios.sh x86_64       # → ios/ffmpeg-x86_64
./init-ios-openssl.sh      # → extra/openssl + ios/openssl-{armv7,v7s,arm64,i386,x86_64}
```

`init-ios.sh` 尚不认识 `arm64-sim` 架构名，需要手动补一份源码目录：

```bash
cd ios
cp -Rp ffmpeg-arm64 ffmpeg-arm64-sim
cp -Rp openssl-arm64 openssl-arm64-sim
```

> 🧹 **如果目录里可能有旧 configure 残留**（2023 年的旧产物、或别人编译失败留下的中间文件），
> 先在各源码目录执行一次 `make distclean`，否则脚本会提示 `reuse configure` 并沿用旧配置，
> 造成编译目标/SDK/依赖路径全错。典型症状：`openssl/bio.h not found`。
>
> ```bash
> (cd ffmpeg-arm64-sim  && make distclean)
> (cd ffmpeg-x86_64    && make distclean)
> (cd openssl-arm64-sim && make distclean)
> (cd openssl-x86_64   && make distclean)
> ```

---

## 5. 编译三方库（真机 + 模拟器）

在 `ios/` 目录按顺序执行（先 OpenSSL，后 FFmpeg）：

```bash
cd ios
./compile-openssl.sh arm64
./compile-openssl.sh arm64-sim
./compile-openssl.sh x86_64

./compile-ffmpeg.sh arm64      # 真机；完成会自动 lipo 到 build/universal/lib
./compile-ffmpeg.sh arm64-sim  # 模拟器（按本分支脚本不会覆盖真机库）
./compile-ffmpeg.sh x86_64
```

输出位置（相对 `ios/`）：

| 内容 | 路径 |
|---|---|
| OpenSSL 各架构 | `build/openssl-{arm64,arm64-sim,x86_64}/output/lib/` |
| FFmpeg 各架构 | `build/ffmpeg-{arm64,arm64-sim,x86_64}/output/lib/` |
| 真机合并库 | `build/universal/lib/`（libav*.a + libssl/libcrypto） |

---

## 6. 构建 Framework 与 xcframework

### 6.1 模拟器构建准备（手工步骤）

1）备份真机库，合并模拟器库进 `universal/lib`：

```bash
cd build
mkdir -p universal/lib-iphoneos universal-sim/lib
mv universal/lib/*.a universal/lib-iphoneos/

for f in libavcodec libavfilter libavformat libavutil libswscale libswresample; do
  xcrun lipo -create ffmpeg-arm64-sim/output/lib/$f.a ffmpeg-x86_64/output/lib/$f.a \
    -output universal-sim/lib/$f.a
  cp universal-sim/lib/$f.a universal/lib/$f.a
done
for f in libcrypto libssl; do
  xcrun lipo -create openssl-arm64-sim/output/lib/$f.a openssl-x86_64/output/lib/$f.a \
    -output universal-sim/lib/$f.a
  cp universal-sim/lib/$f.a universal/lib/$f.a
done
```

2）补分架构头文件（缺少会报 `x86_64/avconfig.h file not found`）：

```bash
U=universal/include
mkdir -p $U/libavutil/arm64 $U/libavutil/x86_64 $U/libffmpeg/arm64 $U/libffmpeg/x86_64
cp -f ffmpeg-arm64-sim/output/include/libavutil/avconfig.h  $U/libavutil/arm64/
cp -f ffmpeg-arm64-sim/output/include/libavutil/ffversion.h $U/libavutil/arm64/
cp -f ffmpeg-arm64-sim/output/include/libffmpeg/config.h    $U/libffmpeg/arm64/
cp -f ffmpeg-x86_64/output/include/libavutil/avconfig.h     $U/libavutil/x86_64/
cp -f ffmpeg-x86_64/output/include/libavutil/ffversion.h    $U/libavutil/x86_64/
cp -f ffmpeg-x86_64/output/include/libffmpeg/config.h       $U/libffmpeg/x86_64/
cd ..
```

### 6.2 构建 Framework（两次 xcodebuild）

```bash
cd IJKMediaPlayer

# ① 模拟器（此时 universal/lib 里是第 6.1 步合并的模拟器库）
xcodebuild -project IJKMediaPlayer.xcodeproj -scheme IJKMediaFrameworkWithSSL \
  -configuration Release -sdk iphonesimulator ARCHS='arm64 x86_64' ONLY_ACTIVE_ARCH=NO \
  ENABLE_BITCODE=NO IPHONEOS_DEPLOYMENT_TARGET=12.0 CODE_SIGNING_ALLOWED=NO build

# ② 切回真机库（再编真机时务必执行！）
cd ../build
rm -f universal/lib/*.a
mv universal/lib-iphoneos/*.a universal/lib/
cd ../IJKMediaPlayer

# ③ 真机
xcodebuild -project IJKMediaPlayer.xcodeproj -scheme IJKMediaFrameworkWithSSL \
  -configuration Release -sdk iphoneos -arch arm64 \
  ENABLE_BITCODE=NO IPHONEOS_DEPLOYMENT_TARGET=12.0 CODE_SIGNING_ALLOWED=NO build
```

产物 Framework 位置（该项目自定义了输出路径）：

```
~/Library/Developer/Xcode/DerivedData/DerivedData/Products/
├── Release-iphonesimulator/IJKMediaFrameworkWithSSL.framework   （arm64 + x86_64）
└── Release-iphoneos/IJKMediaFrameworkWithSSL.framework           （arm64）
```

### 6.3 封装 xcframework

```bash
DD=~/Library/Developer/Xcode/DerivedData/DerivedData/Products
xcodebuild -create-xcframework \
  -framework $DD/Release-iphoneos/IJKMediaFrameworkWithSSL.framework \
  -framework $DD/Release-iphonesimulator/IJKMediaFrameworkWithSSL.framework \
  -output $DD/IJKMediaFrameworkWithSSL.xcframework
```

### 6.4 产物自检

```bash
# 切片与平台戳（模拟器切片 platform 应为 IOSSIMULATOR）
xcrun lipo -info  <framework 路径>/IJKMediaFrameworkWithSSL
vtool -show-build <framework 路径>/IJKMediaFrameworkWithSSL | grep -E 'platform|minos'

# 功能符号：应能查到 ff_rtsp、_ff_pcm_alaw_decoder、_ff_mjpeg_decoder
nm <framework 路径>/IJKMediaFrameworkWithSSL | grep -E 'ff_rtsp|_ff_pcm_alaw_decoder|_ff_mjpeg_decoder'
```

---

## 7. App 集成

1. 把 `IJKMediaFrameworkWithSSL.xcframework` 拖入工程，General → Frameworks 选 **Embed & Sign**；
2. 真机 / 模拟器运行时自动匹配切片。

RTSP 播放示例（建议强制 TCP，摄像头场景最稳）：

```objc
IJKFFOptions *options = [IJKFFOptions optionsByDefault];
[options setOptionValue:@"tcp" forKey:@"rtsp_transport" ofCategory:kIJKFFOptionCategoryFormat];
[options setOptionIntValue:1 forKey:@"videotoolbox" ofCategory:kIJKFFOptionCategoryPlayer]; // 硬解

NSURL *url = [NSURL URLWithString:@"rtsp://user:pass@192.168.1.100:554/stream1"];
IJKFFMoviePlayerController *player =
    [[IJKFFMoviePlayerController alloc] initWithContentURL:url withOptions:options];
```

---

## 8. 常见问题（均为实机踩坑记录）

| 现象 | 原因 | 处置 |
|---|---|---|
| `make install` 静默写错路径，如生成 `/Volumes/subaifuSDK/...` | 路径含 `$`，被 make 当变量展开 | 用不含 `$` 的软链接进入（见第 2 节） |
| `'openssl/bio.h' file not found` 等依赖找不到 | 源码目录残留旧 configure 产物（日志出现 `reuse configure`） | 对应目录 `make distclean` 后重编 |
| `-Wincompatible-function-pointer-types` 等报 error | 新版 clang 把老警告升级为错误 | 分支脚本已内置 `-std=gnu17 -Wno-error=…`；若自行改过脚本需保留 |
| `x86_64/avconfig.h file not found`（编模拟器时） | `universal/include` 缺分架构头 | 执行第 6.1-2) 步 |
| bitcode / deployment target 报错 | 老工程默认设置 | 编译命令已带 `ENABLE_BITCODE=NO IPHONEOS_DEPLOYMENT_TARGET=12.0`；脚本侧已改 min 12.0（去 bitcode） |
| 模拟器链接报 "built for iOS, but linking for iOS Simulator" | `universal/lib` 处于真机库状态 | 先完成 6.1 库切换；构建真机前记得切回 |
| iPad/真机未插导致的 `device is not connected` 提示 | 仅自动签名噪音，framework 用 `CODE_SIGNING_ALLOWED=NO` 构建 | 可忽略 |

---

## 9. 快速命令备忘（按序完整跑齐）

```bash
git clone -b branch_ArmAndSim https://github.com/BFsAlex/ijkplayer_k0.8.8.git ijkplayer
cd ijkplayer
./init-ios.sh arm64 && ./init-ios.sh x86_64 && ./init-ios-openssl.sh
cd ios && cp -Rp ffmpeg-arm64 ffmpeg-arm64-sim && cp -Rp openssl-arm64 openssl-arm64-sim
./compile-openssl.sh arm64 && ./compile-openssl.sh arm64-sim && ./compile-openssl.sh x86_64
./compile-ffmpeg.sh arm64 && ./compile-ffmpeg.sh arm64-sim && ./compile-ffmpeg.sh x86_64
# ……然后按第 6 节构建 Framework 与 xcframework
```
