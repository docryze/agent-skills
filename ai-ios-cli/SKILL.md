---
name: ai-ios-cli
description: 纯命令行 iOS 原生应用开发全流程（真机调试）— xcodegen 建工程、xcodebuild 构建/测试/归档、devicectl 真机安装与日志、altool 上架，零 GUI、零额外安装
---

# ai-ios-cli · 纯 CLI 的 iOS 原生开发流程（真机调试）

在**纯命令行、零额外安装**下完成 iOS 原生应用全生命周期：脚手架、依赖、签名、构建、真机调试、测试、归档、上架、产物分析，全程不打开 Xcode GUI、**严禁使用模拟器（一律真机调试）**。

> 真机前提：已用数据线/网络连接 iPhone/iPad，设备已信任本机，且钥匙串中有 **Apple Development** 签名证书 + 包含该设备 UDID 的描述文件。

---

## 1. 工具清单

| 类别 | 工具 | 简介 |
|------|------|------|
| **Xcode 工具链** | Xcode / Swift | IDE/SDK + 编译器（经 `xcodebuild` 编译、`swift package` 管依赖） |
| | swift package (SPM) | **唯一依赖管理器**（替代 CocoaPods/Carthage） |
| | xcodebuild | 构建 / 测试 / 归档 / 导出 ipa 核心命令（device 目标） |
| | xcrun | 统一调度 Xcode 子工具 |
| | xcode-select | 切换活跃 Xcode |
| | **devicectl** | **真机**：列举/安装/启动/终止/卸载/查进程，`--console` 流式日志 |
| | altool | 向 App Store/TestFlight **校验+上传**（替代 Fastlane 上传） |
| | security / codesign | 钥匙串证书查询 / 代码签名（真机构建必需） |
| | lipo / plutil | 架构分析合并 / plist 读写 |
| | assetutil / actool / ibtool | 资源体积分析 / Assets 编译（含 AppIcon）/ XIB 编译 |
| **工程生成 / 版本** | xcodegen | 由 `project.yml` 声明式生成 `.xcodeproj`（替代 Tuist，AI 可读可改） |
| | xcodes | 多版本 Xcode 安装/切换 |
| **包管理 / 运行时** | Homebrew / Node | 装额外 CLI、脚本运行时 |
| **通用辅助** | git / jq / fd / fzf | 版本管理 / JSON 解析 / 产物检索 / 交互筛选 |

> ⛔ 不依赖任何未安装工具：无 CocoaPods / Fastlane / SwiftLint / Mint / Tuist / Carthage / gh / ios-deploy。
> ⛔ **严禁模拟器——一律真机调试**（`devicectl`）。禁止执行任何模拟器相关命令：不下载 runtime（`xcodebuild -downloadPlatform`）、不创建实例（`xcrun simctl create`）、不启动（`xcrun simctl boot` / `open -a Simulator`）、不以 `iphonesimulator` 为 SDK/destination 构建。无真机连接时仅用 `generic/platform=iOS` 做编译验证，行为验证一律等真机就绪后在真机上做。

---

## 2. 单一工具链（每个环节只有一条主路径）

| 环节 | 唯一方式 |
|------|---------|
| 工程创建 | `xcodegen`（`project.yml` → `.xcodeproj`） |
| 依赖管理 | `swift package`（SPM） |
| 签名准备 | `security find-identity` + 工程 automatic signing（真机必需） |
| 构建 | `xcodebuild build`（`-destination platform=iOS`） |
| 真机调试 | `devicectl`（list→install→launch `--console`） |
| 测试 | `xcodebuild test`（device 目标） |
| 归档/ipa | `xcodebuild archive` + `-exportArchive` |
| 上传 | `altool`（统一 App Store Connect API Key 认证） |

---

## 3. 工作流总览（11 阶段）

```
┌──────────────────────────────────────────────────────────────┐
│ 0 环境     xcodes / xcode-select / xcrun      选 & 校验 SDK     │
│ 1 脚手架   xcodegen(project.yml)                              │
│ 2 依赖     swift package(resolve/update/show)  ← SPM only      │
│ 3 签名     security find-identity → 校验证书/描述文件（真机必需） │
│ 4 构建     xcodebuild build  (-destination platform=iOS)       │
│ 5 真机调试 devicectl: list→install→launch --console(日志流)    │
│ 6 测试     xcodebuild test  (-destination platform=iOS)        │
│ 7 归档     xcodebuild archive → -exportArchive(产 ipa)         │
│ 8 上架     altool --validate-app → --upload-app                │
│ 9 产物分析 lipo / plutil / assetutil                           │
│10 资源处理 actool / ibtool（AppIcon 随 Assets.xcassets）       │
└──────────────────────────────────────────────────────────────┘
```

> 真机全流程要点：① 签名（阶段 3）前置且强制；② 构建目标一律 `platform=iOS`；③ 调试反馈走 `launch --console` 日志流（非截图）。

---

## 4. 分阶段命令手册

### 阶段 0 · 环境校验
```bash
xcodes installed                                    # 已装 Xcode
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
xcrun --version && swift --version                  # 工具链就绪
xcrun devicectl list devices                        # 真机：列出已连接设备
```

### 阶段 1 · 工程脚手架（统一 xcodegen）

> 工程统一由 `project.yml` 声明生成：纯文本、git 友好、AI 可读可改，是**唯一**工程创建方式。`DEVELOPMENT_TEAM` + 自动签名是真机构建的前提。

```yaml
# project.yml
name: MyApp
options:
  bundleIdPrefix: com.example
  deploymentTarget: { iOS: "17.0" }
targets:
  MyApp:
    type: application
    platform: iOS
    sources: [MyApp]
    info: { path: MyApp/Info.plist }
    settings:
      base:
        DEVELOPMENT_TEAM: <TEAM_ID>          # 真机自动签名必需
        CODE_SIGN_STYLE: Automatic
```
```bash
xcodegen generate                                   # → MyApp.xcodeproj
```

### 阶段 2 · 依赖管理（SPM）
```bash
swift package resolve                               # 拉取
swift package update                                # 升级
swift package show-dependencies                     # 树形（喂上下文）
```

### 阶段 3 · 签名准备（真机必需）
```bash
security find-identity -v -p codesigning           # 必须存在 "Apple Development: ..." 证书
# 设备 UDID 须已加入对应 Profile；用 devicectl 确认设备在线：
xcrun devicectl list devices
xcrun devicectl device info details -d "<UDID或设备名>"
# 缺证书/设备未注册 → 先在 Apple Developer 后台加设备、下载 Profile，再继续
```

### 阶段 4 · 构建（device 目标）
```bash
xcodebuild -list -project MyApp.xcodeproj                                   # scheme/target
# -destination 三选一：id=<UDID> 精确｜name=<设备名>｜generic/platform=iOS 任意设备
xcodebuild build -scheme MyApp \
  -destination 'generic/platform=iOS' \
  -derivedDataPath build/ \
  -allowProvisioningUpdates \
  -quiet 2>&1 | grep -E "error:|warning:|BUILD"                             # 静默提取
# 产物 → build/Build/Products/Debug-iphoneos/MyApp.app（已签名）
```

### 阶段 5 · 真机调试（日志闭环）
```bash
DEV="<UDID或设备名>"
APP=$(fd "MyApp.app$" build -t f | grep Debug-iphoneos | head -1)          # fd 找已签名产物
xcrun devicectl device install app -d "$DEV" "$APP"                        # 安装
# 启动并接管 stdout/stderr：print / NSLog / 崩溃栈实时流式输出（核心调试信号）
xcrun devicectl device process launch --console -d "$DEV" com.example.MyApp
# 重新启动（先终止旧实例）：
xcrun devicectl device process launch --console --terminate-existing -d "$DEV" com.example.MyApp
# 主动终止：
xcrun devicectl device process terminate -d "$DEV" com.example.MyApp
# 卸载：
xcrun devicectl device uninstall app -d "$DEV" com.example.MyApp
```
> ⚠️ 视觉验证：devicectl **不支持截图**。如需画面：用 `xcrun xctrace record --device "$DEV" ...` 录屏，或借助 mobile-mcp（见 `device-debug-constraints` skill）。

### 阶段 6 · 测试（device 目标）
```bash
xcodebuild test -scheme MyApp \
  -destination "id=$DEV" \
  -allowProvisioningUpdates \
  2>&1 | grep -E "Test Case|failed|passed|BUILD"
```

### 阶段 7 · 归档 & 导出 ipa
```bash
xcodebuild archive \
  -scheme MyApp \
  -archivePath build/MyApp.xcarchive \
  -destination 'generic/platform=iOS' \
  -allowProvisioningUpdates
xcodebuild -exportArchive \
  -archivePath build/MyApp.xcarchive \
  -exportPath build/ipa \
  -exportOptionsPlist ExportOptions.plist        # → build/ipa/MyApp.ipa
```

最小导出配置（`ExportOptions.plist`）— 缺它 exportArchive 会报错：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>method</key><string>app-store</string>   <!-- app-store | ad-hoc | enterprise | development -->
  <key>teamID</key><string>TEAM_ID</string>
  <key>uploadBitcode</key><false/>
  <key>uploadSymbols</key><true/>
</dict></plist>
```

### 阶段 8 · 上架 App Store / TestFlight（无 Fastlane）
```bash
IPA=$(fd "\.ipa$" build/ipa | head -1)
# 1) 先校验（省时间）
xcrun altool --validate-app -f "$IPA" -t ios \
  --apiKey  $API_KEY --apiIssuer $API_ISSUER
# 2) 通过则上传
xcrun altool --upload-app  -f "$IPA" -t ios \
  --apiKey  $API_KEY --apiIssuer $API_ISSUER
# 认证统一用 App Store Connect API Key（官方推荐，可入 CI）
# .p8 放 ~/private_keys/ApiKey_$API_KEY.p8（或 ~/.appstoreconnect/private_keys/）
```

### 阶段 9 · 产物分析（自检 / 瘦身）
```bash
APPBIN=build/Build/Products/Debug-iphoneos/MyApp.app/MyApp
lipo -info   "$APPBIN"                           # 含哪些架构（通常 arm64）
lipo -archs  "$APPBIN"                           # 仅列架构名
# 合并切片：输入是二进制文件而非架构名，例如
#   lipo -create <slice_a> <slice_b> -output <combined>
plutil -p     MyApp.app/Info.plist               # 读 BundleId/版本/权限
xcrun assetutil --info MyApp.app/Assets.car      # 资源体积分布（文本输出，勿接 jq）
```

### 阶段 10 · 资源处理
```bash
# AppIcon 在 Assets.xcassets 内，随构建由 actool 自动编译；.icns 是 macOS 专用，iOS 不用
xcrun actool Assets.xcassets --compile build --platform iphoneos \
  --minimum-deployment-target 17.0
xcrun ibtool ViewController.xib --compile build/ViewController.nib
```

---

## 5. 端到端脚本（一键跑通最小 App · 真机）

改 4 个变量（`APP/SCHEME/BUNDLE/DEV`）即可复用：

```bash
#!/usr/bin/env bash
set -euo pipefail
APP=MyApp; SCHEME=MyApp; BUNDLE=com.example.MyApp; DEV="<设备UDID>"  # devicectl -d 也可用设备名；但 xcodebuild id= 必须 UDID

# 1 脚手架
xcodegen generate

# 2 依赖
swift package resolve 2>/dev/null || true

# 3 签名检查（必须有 Apple Development 证书）
security find-identity -v -p codesigning | grep -q "Apple Development" \
  || { echo "❌ 缺开发签名证书"; exit 1; }

# 4 构建（device）
xcodebuild build -scheme "$SCHEME" -destination "id=$DEV" \
  -derivedDataPath build/ -allowProvisioningUpdates -quiet

# 5 真机：安装→启动并接管日志（10s 摘要；交互场景去掉 timeout 保留 --console 常驻）
APPBIN=$(fd "$APP.app$" build -t f | grep Debug-iphoneos | head -1)
xcrun devicectl device install app -d "$DEV" "$APPBIN"
# 启动拉日志（这里超时 10s 摘要输出；交互场景去掉 timeout 保留 --console 常驻）
timeout 10 xcrun devicectl device process launch --console -d "$DEV" "$BUNDLE" \
  2>&1 | tail -30 || true

# 6 测试
xcodebuild test -scheme "$SCHEME" -destination "id=$DEV" \
  -allowProvisioningUpdates 2>&1 | grep -E "Test Suite.*passed|failed|BUILD" || true

# 7 归档→ipa
xcodebuild archive -scheme "$SCHEME" -archivePath build/$APP.xcarchive \
  -destination 'generic/platform=iOS' -allowProvisioningUpdates -quiet
xcodebuild -exportArchive -archivePath build/$APP.xcarchive \
  -exportPath build/ipa -exportOptionsPlist ExportOptions.plist

# 8 校验（不自动上传，避免误操作）
IPA=$(fd "\.ipa$" build/ipa | head -1)
xcrun altool --validate-app -f "$IPA" -t ios \
  --apiKey "$API_KEY" --apiIssuer "$API_ISSUER" \
  && echo "✅ $IPA 校验通过，可上传" || echo "❌ 校验失败"

# 9 产物自检
echo "架构: $(lipo -info "$APPBIN")"
echo "版本: $(plutil -extract CFBundleShortVersionString raw "$APPBIN/Info.plist")"
```

---

## 6. 执行约定

1. **可读优先**：工程统一由 `project.yml` 声明生成（纯文本、git 友好），避免二进制 `.pbxproj`；第三方依赖再走 SPM。
2. **真机唯一（严禁模拟器）**：所有构建/调试/测试 `destination` 一律 `platform=iOS`/`id=<UDID>`。**禁止任何模拟器操作**——不下载 runtime、不创建/启动模拟器实例、不打开 Simulator app、不以 `iphonesimulator` 为构建目标。无真机连接时用 `generic/platform=iOS` 做编译验证，行为验证等真机就绪后在真机做。
3. **签名前置**：device 构建前先 `security find-identity` 确认 `Apple Development` 证书存在；构建带 `-allowProvisioningUpdates`，工程用 automatic signing。
4. **静默构建**：`xcodebuild -quiet 2>&1 | grep -E "error:|BUILD"` 控制上下文长度。
5. **产物用 fd 定位**：`fd "MyApp.app$" build | grep Debug-iphoneos` / `fd "\.ipa$" build`。
6. **JSON 用 jq 解析**：`xcodebuild -json ... | jq`、`plutil` 输出喂 `jq`；devicectl 脚本消费用 `--json-output <file>`（官方稳定接口）。
7. **日志闭环（替代截图）**：`devicectl device process launch --console` 实时拉 stdout/stderr（print/NSLog/崩溃），形成"改代码→装→启动→看日志"。需画面时用 `xctrace record` 录屏或 mobile-mcp。
8. **依赖只走 SPM**：遇到 `Podfile` 立即提示"需补 CocoaPods 或迁移 SPM"，不假设已装。
9. **上传前必先 validate**：`altool --validate-app` 通过再 `--upload-app`，省时间、防误传。
10. **代码质量缺口标注**：基线无 lint/format，涉及代码规范任务时明确告知"需补 SwiftLint/swift-format"。

---

## 7. 能力缺口（按需补装）

| 缺口 | 影响 | 补救 |
|------|------|------|
| SwiftLint / swift-format | 无代码 lint/format 门禁 | 需要时 `brew install swiftlint swift-format` |
| 真机截图 | devicectl 无截图子命令 | `xcrun xctrace record` 录屏，或 mobile-mcp |
| CocoaPods | 无法构建 Pod-only 依赖的老项目 | 迁移到 SPM，或 `brew install cocoapods` |
| gh | 无 GitHub Release/CI 一键化 | `git + curl + token` 凑合，或 `brew install gh` |
| Fastlane | 无多语言截图/lane 编排 | devicectl+altool+脚本可覆盖核心；高频自动化时再装 |

---

## 8. MVP 闭环检查清单（真机）

- [ ] `xcodegen generate` 生成工程（含 `DEVELOPMENT_TEAM`）
- [ ] `security find-identity` 存在 Apple Development 证书
- [ ] `xcodebuild build -destination 'platform=iOS'` 产出 `Debug-iphoneos/*.app`
- [ ] `devicectl device install app` 安装到真机
- [ ] `devicectl device process launch --console` 启动并输出日志
- [ ] `xcodebuild test -destination 'platform=iOS'` 通过
- [ ] `xcodebuild archive + -exportArchive` 出 `.ipa`
- [ ] `altool --validate-app` 通过

八项全绿 = 具备**无 GUI、零额外安装、纯真机**的 iOS 全流程开发能力。
