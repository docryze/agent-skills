---
name: device-debug-constraints
description: 设备调试安全约束 — adb 命令使用规范、禁止修改设备设置、仅限调试目标 app 操作
---

# 设备调试安全约束

使用 adb、mobile-mcp 或其他方式调试 Android/iOS 设备时，严格遵守以下约束。

---

## 1. 核心原则

**禁止修改设备的任何系统设置。仅允许只读操作和当前调试 app 的操作。**

---

## 2. 允许的操作

### 只读查询（不修改任何状态）
- `screenshot` / `screencap` — 截图
- `logcat` — 日志查看
- `dumpsys` — 系统状态查询
- `getprop` — 属性查询
- `settings get` — 设置项读取
- `pm list` — 包列表查询
- `ps` / `top` — 进程查询
- `ping` / `nslookup` — 网络诊断

### App 操作（仅限当前调试的 app 包名）
- `install` / `uninstall` — 安装/卸载调试 app
- `am start` / `am force-stop` — 启动/停止调试 app
- `pm clear` — 清除调试 app 数据

### UI 交互（仅模拟用户操作，不修改设置）
- `input tap` / `input swipe` / `input keyevent` — 触摸/按键模拟

---

## 3. 禁止的操作（非穷举）

### 系统设置修改
- `svc power stayon` — 屏幕常亮
- `svc wifi enable/disable` — WiFi 开关
- `svc bluetooth enable/disable` — 蓝牙开关
- `settings put` — 修改任何系统设置项
- `wm density` / `wm size` / `wm overscan` — 修改显示参数

### 权限修改
- `pm grant` / `pm revoke` — 授予/撤销权限

### 系统/通知/无障碍修改
- `cmd notification set_dnd` — 免打扰设置
- `cmd accessibility` — 无障碍设置
- 修改开发者选项、USB 调试设置等

### 非调试目标 app 操作
- 安装、卸载、启动、停止、清除**非调试目标 app**
- 例如：微信、支付宝、系统 app 等设备上的其他应用

---

## 4. 例外条件

- **如需修改设备设置，必须先征得用户明确同意**
- 修改后必须告知用户：改了什么、当前值、如何恢复
