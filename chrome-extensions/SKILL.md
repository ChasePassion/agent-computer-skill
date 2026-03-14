---
name: chrome-extensions
description: Use when operating Google Chrome extension management on Windows, especially `chrome://extensions/`, including extension enable/disable, load unpacked, update, remove, keyboard shortcuts, and Browser Assist Locator loading and troubleshooting. Triggers on Chrome 扩展, Browser Assist Locator, chrome://extensions, Load unpacked, Update extension, 扩展快捷键.
---

# Google Chrome 扩展管理手册
> 适用：Windows 桌面端 Chrome | 更新：2026-03-14

## 执行原则
1. 页面必须是 `chrome://extensions/`，浏览器保持最大化
2. `Browser Assist Locator` 只负责定位，不执行点击 / 输入 / 滚动
3. 最终坐标映射由 daemon 完成，真实动作由 `agent-computer` 执行
4. 扩展代码更新后表现异常，优先点 `Update`
5. 排障顺序：daemon → status → 扩展加载 → 前台浏览器 → locate

---

## 界面元素与行为

| 元素 | 操作 | 结果 |
| :--- | :--- | :--- |
| 搜索框 | 输入关键词 | 实时过滤扩展列表 |
| 插件开关 | 点击 | 蓝色=开启，灰色=关闭 |
| Details | 点击 | 进入详情页（版本、权限、无痕模式） |
| Remove | 点击 | 弹出确认框，确认后卸载 |
| Developer mode | 右上角开关 | 开启后左上角出现 Load unpacked / Pack extension / Update |
| Load unpacked | 点击 | 弹出文件夹选择器，加载本地扩展 |
| Pack extension | 点击 | 弹出打包对话框，生成 `.crx` |
| Update | 点击 | 强制重新加载所有已安装扩展 |
| Keyboard shortcuts | 左侧导航 | 进入快捷键配置页 |
| Chrome Web Store | 左下角链接 | 打开 Chrome 网上应用店 |

**关键控件坐标**（浏览器最大化 + `chrome://extensions/`）：
- Developer mode：`(1890, 180)`
- Load unpacked：`(100, 250)`
- Update：`(450, 250)`

> 窗口尺寸、系统缩放或浏览器缩放改变时坐标会漂移。

---

## 常用流程

**管理扩展**：搜索框过滤 → 开关启停 → Details 查详情 → Remove 卸载

**加载本地扩展**：开启 Developer mode → Load unpacked → 选中扩展根目录 → 选择文件夹

**更新扩展**：确认 Developer mode 开启 → 点 Update → 等待左下角 `Updating...` 消失

**设置快捷键**：左侧 Keyboard shortcuts → 找到目标扩展 → 点 `Not set` → 按下组合键

---

## Browser Assist Locator 专项

### 扩展职责
- **负责**：查找网页元素，返回 `page / viewport / browser / matches`，返回元素 `rect` 和 `clickablePoint`
- **不负责**：点击、输入、滚动、浏览器自动化执行

### 相关文件
| 用途 | 路径 |
| :--- | :--- |
| 扩展根目录 | `E:/code/agent-computer/extensions/browser-assist-locator` |
| 打包脚本 | `E:/code/agent-computer/scripts/package_browser_assist_extension.ps1` |
| 请求样例 | `E:/code/agent-computer/docs/examples/browser-assist-request.json` |
| Token 文件 | `E:/code/agent-computer/.agent/browser_assist.json` |

### 完整使用流程

**第 1 步：启动 daemon 并确认状态**
```powershell
.\windows-launcher.ps1 daemon start
.\windows-launcher.ps1 browser-assist-status
```
daemon 正常但扩展未连接时返回 `"connected": false`，属于正常，继续加载扩展即可。

**第 2 步：加载扩展（两种方法）**

方法 A — 直接加载源码目录：
1. `chrome://extensions/` → 开启 Developer mode `(1890, 180)`
2. Load unpacked `(100, 250)` → 选择 `extensions/browser-assist-locator` → 选择文件夹

方法 B — 先打包再加载：
```powershell
.\scripts\package_browser_assist_extension.ps1
```
产物位于 `artifacts/browser-assist-locator-package/browser-assist-locator`，再按方法 A 流程加载该目录。

**第 3 步：确认连接成功**
```powershell
.\windows-launcher.ps1 browser-assist-status
```
成功时返回 `connected: true` + `extensionVersion` / `browserName` / `lastSeenAt`。

**第 4 步：打开目标页并保持前台**
```powershell
.\windows-launcher.ps1 browser-open-url --url "<目标 URL>"
```
Browser Assist v1 会校验前台窗口是否为浏览器，以及 URL / 标题是否一致，执行 locate 前浏览器需保持前台。

**第 5 步：发起 locate 请求**
```powershell
.\windows-launcher.ps1 browser-assist-locate --input-file .\docs\examples\browser-assist-request.json
```
请求体示例：
```json
{
  "query": {
    "text": "收藏",
    "role": "button",
    "hint": "当前职位详情区域里的收藏按钮",
    "selectorHint": null,
    "index": 0
  },
  "options": {
    "visibleOnly": true,
    "interactiveOnly": true,
    "maxCandidates": 5
  }
}
```

**第 6 步：读取返回结果并执行点击**

返回结构关键字段：
- `matches[].rect` — 网页视口内 DOM 几何框
- `matches[].clickablePoint` — 网页内部点击点
- `browser.contentLeftOnScreen` / `contentTopOnScreen` — 内容区锚点
- `mapped.screenCandidates[].screenPoint` — daemon 映射后的桌面绝对坐标（用这个）

```powershell
.\windows-launcher.ps1 click --x <screen_x> --y <screen_y>
```

**第 7 步：截图校验**
```powershell
.\windows-launcher.ps1 capture-grid --grid-size 50
```

### 代码更新后刷新扩展
回到 `chrome://extensions/` → 确认 Developer mode 开启 → 点 Update `(450, 250)`

### 结束使用
- 保留扩展：关闭开关即可
- 彻底卸载：扩展页点 Remove
- 停止后端：`.\windows-launcher.ps1 daemon stop`（daemon 停止后扩展仍存在但处于未连接状态）

---

## 排障顺序
1. `.\windows-launcher.ps1 daemon start`
2. `.\windows-launcher.ps1 browser-assist-status`
3. 确认扩展已加载且开关开启
4. 确认浏览器是前台窗口
5. 确认当前 URL 为目标页
6. 点 Update 刷新扩展
7. 重新执行 `browser-assist-locate`

---

## 链路总结
```
扩展页加载 → 扩展连接 daemon → status 显示 connected:true
→ locate 返回 raw+mapped → agent-computer click 执行真实动作
```
**扩展负责定位，daemon 负责映射，computer 负责执行。**