---
name: cn35-apk-packaging
description: 引导式打包 CN3.5 梅露可物语重打包 APK。先让用户在两种模式中选择：(A) 服务端+APK——files/remote_assets 由桌面 mercd35 下发，APK 内不嵌服务；(B) 单 APK（集成 AAR）——gomobile 把 mercd35 编进 APK，BootProvider 进程内起服，inputs/files 并入 assets/unity 随包解压。当用户要求打包、重打包、构建 APK、单 APK、集成 AAR，或游戏卡在"数据解压中"时使用。
---

# CN3.5 APK 打包引导

本 skill 是一个**交互式 workflow**：不要一次跑完所有命令，按步骤推进，每步先验证前置条件再执行。

## 步骤 0：确认打包模式

**必须先问用户选哪种模式**（用 ask_user_question）：

- **模式 A · 服务端 + APK**：桌面/服务器跑 `mercd35`，`inputs/files`（remote_assets）由服务端通过网络下发。APK 较小（约 325 MB 级 unity），但需要外部服务可达（模拟器用 `adb reverse`，真机用 LAN IP）。
- **模式 B · 单 APK（集成 AAR）**：`gomobile bind` 生成 `mercd35.aar`，`BootProvider` 在进程内起服（127.0.0.1:18091）；`inputs/files` 并入 `assets/unity` 走 StreamingAssets。APK 约 1.5 GB，完全离线可玩。

模式决定一切后续步骤，**不可混用**：模式 A 若残留 BootProvider/classes2.dex/assets/backend，包内服务会抢占 18091 并遮蔽外部服务端（这是"卡数据解压中"的常见根因）；模式 B 若缺 `inputs/files` 并入，客户端会永远等远程资源下载。

## 步骤 1：环境前置检查（两种模式共用）

逐项检查，缺哪个就停下来让用户装/配：

```sh
java -version                                   # apktool/d8 需要；d8 需 JDK 11+（javac 侧用 JDK8 + bootclasspath）
python3 --version                               # prepare.py / build.sh 读 baseline.json
apktool --version  # 或 APKTOOL_JAR 指向的 jar   # 必须恰好 3.0.3（client/build.sh 强校验）
```

- 原包：`inputs/apk/国服-3.5.0.apk` 存在，SHA-256 须等于 `client/baseline.json` 的 `originalApkSha256`（`d9412c15…9118a2`）。`prepare.py` 会自动校验。
- Android SDK：`ANDROID_SDK_ROOT`/`ANDROID_HOME` 下有 `build-tools`（zipalign；签名还要 apksigner）和 `platforms/android.jar`。
- 签名（`--sign` 时必需）四个环境变量：`ANDROID_KEYSTORE`、`ANDROID_KEY_ALIAS`、`ANDROID_KEYSTORE_PASS`、`ANDROID_KEY_PASS`。没有 keystore 可引导用户生成：
  ```sh
  keytool -genkeypair -v -keystore mercd.keystore -alias mercd -keyalg RSA -keysize 2048 -validity 10000
  ```
- Windows 注意：所有脚本都是 bash（Git Bash / MSYS2 / WSL 任一），`client/android-bootstrap/build.sh` 里 `/usr/libexec/java_home` 是 macOS 专有——Windows 上须显式设 `D8_JAVA_HOME`（JDK 11+）与 `JAVAC8`（JDK8 javac）；`cp -Rc` 的 clonefile 会自动 fallback 到普通 `cp -R`，无需处理。

## 步骤 2：解码客户端（两种模式共用，每棵树只做一次）

`client/decoded` 已存在时 `prepare.py` 拒绝覆盖。若已存在且模式未变可跳过；若要切换模式（A↔B），先确认 decoded 树归属，**切到 B 需要重新 prepare**（Manifest 要注入 provider）：

```sh
# 模式 A（server-host 换成目标地址；模拟器+adb reverse 用 127.0.0.1:18091，真机用 LAN IP）
python3 client/prepare.py --mode server --server-host 127.0.0.1:18091

# 模式 B（server-host 固定 127.0.0.1:18091，脚本强制）
python3 client/prepare.py --mode standalone
```

prepare 只改两处：Manifest（改包名 `com.happyelements.miraclecn.uc.cn350dbg`、debuggable；standalone 额外注入 `com.mercd.backend.BootProvider`）和 `assets/gsp.txt` 第 6 字段（服务地址）。**DLL 不做任何修改**——若用户声称要改 DLL，指出服务地址在 gsp.txt，不在 DLL。

## 步骤 3A：服务端 + APK 流程

1. **清掉单机残留**（若 decoded 树曾被 android-bootstrap/build.sh 碰过）：
   ```sh
   rm -f client/decoded/classes2.dex client/decoded/lib/armeabi-v7a/libgojni.so
   rm -rf client/decoded/assets/backend
   grep -n BootProvider client/decoded/AndroidManifest.xml   # 有残留则需重新 prepare（provider 是 prepare 写入的）
   ```
2. 打包：
   ```sh
   client/build.sh --sign    # 或去掉 --sign 仅产出 aligned-unsigned
   ```
   产物在 `output/rebuilt/mercstoria-cn350*.apk`。
3. 配服务端 `.env`（`make env` 生成）确认：
   ```
   MERCD_ADDR=127.0.0.1:18091          # 真机/LAN 改为 0.0.0.0:端口
   MERCD_SERVER_HOST=127.0.0.1:18091   # 必须是客户端可达地址；真机填 LAN IP，并与 gsp.txt 一致
   MERCD_REMOTE_ASSETS=inputs/files    # remote_assets 由服务端下发
   ```
   `inputs/files` 必须存在且有内容（用户自备，不入库）。
4. 起服 `make run`，然后安装并连通：
   ```sh
   adb install -r output/rebuilt/mercstoria-cn350.apk
   adb reverse tcp:18091 tcp:18091     # 模拟器/真机 USB 调试均可；真机 WiFi/LAN 则不用
   ```
   `adb reverse` 在杀进程/重连后失效，每次调试用前重设。
5. 验证：先 `curl http://127.0.0.1:18091/mercd/health`，再进游戏。卡"数据解压中"时抓 `adb logcat` 与服务端请求日志（`MERCD_REQUEST_LOG`）对照——客户端首启会向服务端拉 remote_assets 清单并下载，服务端日志无请求即网络没通。

## 步骤 3B：单 APK（集成 AAR）流程

1. 确认 `inputs/files/` 存在且为用户自备的完整 remote_assets（缺失会导致卡"数据解压中"，这是该模式最常见失败）。
2. 生成 AAR（需要 Go ≥ go.mod 要求 + Android NDK；gomobile 需已 `gomobile init` 或新版直接可用）：
   ```sh
   cd server && gomobile bind -target=android/arm -androidapi 21 -o /tmp/mercd35.aar ./mobile
   ```
   只用 `android/arm`：客户端是 ARM32-only，多 ABI 会导致模拟器/64 位机闪退（build.sh 内注释有完整原因）。Windows 上输出路径如 `C:/temp/mercd35.aar`，记得同步改 `AAR=`。
3. 跑 bootstrap（编译 BootProvider→classes2.dex、拷 libgojni.so、拷 assets/backend、生成 decoded-single 并并入 inputs/files + 重生成资源清单）：
   ```sh
   AAR=/tmp/mercd35.aar client/android-bootstrap/build.sh
   ```
   成功标志：输出 `=> client/decoded-single (remote_assets 已并入 assets/unity，清单已重生成)`。若警告缺 `inputs/files` 则停下补资源，不要用缺资源的包继续。
4. 打包整包 staging 树（不是 decoded）：
   ```sh
   CLIENT_SOURCE_DIR=client/decoded-single client/build.sh --sign
   ```
5. 安装验证：`adb install -r`；首次启动 logcat 过滤 `mercd-boot` 应见 `backend assets extracted` / `Mobile.start returned`；之后进游戏不再依赖任何外部服务，`adb reverse` 不需要也不要配（避免端口歧义）。

## 常见坑速查

| 症状 | 根因 |
|---|---|
| 一直"数据解压中" | 模式混用：decoded 带 BootProvider 但 remote_assets 没并入；或模式 A 的 inputs/files 为空/服务端不可达 |
| 装不上/闪退 `Unable to find native library main` | APK 里出现 armeabi-v7a 以外的 ABI 目录；bootstrap 只应拷 armeabi-v7a |
| `apktool 3.0.3 required` | 版本强校验，换 `APKTOOL_JAR` 指向 3.0.3 |
| `client/decoded 已存在` | prepare 防覆盖；切换模式先备份/删除重建 |
| 起服冲突 18091 | BootProvider 残留（模式 A 误带包内服务），按 3A-1 清理 |
| 改过 APK 后又卡解压 | `adb reverse` 掉了；或 decoded-single 是旧 staging，重跑 android-bootstrap/build.sh |

## 关键文件索引

- `client/prepare.py`：校验原包 SHA-256 → apktool d → 改 Manifest/gsp.txt（唯一改客户端的入口）
- `client/build.sh`：apktool b → zipalign → 可选 apksigner；`CLIENT_SOURCE_DIR`/`CLIENT_OUTPUT_DIR` 可覆盖
- `client/android-bootstrap/build.sh`：AAR→classes2.dex+libgojni+assets/backend→decoded-single 整包 staging
- `client/merge_asset_manifest.py`：并入 remote_assets 后重生成 unity 清单
- `client/baseline.json`：原包路径/SHA-256/apktool 版本/输出名
- `server/mobile/mobile.go`：gomobile 入口（`Mobile.start(state, cn350, cards, port)`）
- `server/run.sh` / `.env`：模式 A 服务端运行参数
- `docs/运行与材料.md`：权威说明，与本 skill 冲突时以它为准
