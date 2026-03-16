---
name: chrome-extensions
description: Use when operating Google Chrome extension management on Windows, especially `chrome://extensions/`, including extension enable/disable, load unpacked, update, remove, keyboard shortcuts, and Browser Assist Locator loading and troubleshooting. Triggers on Chrome 扩展, Browser Assist Locator, chrome://extensions, Load unpacked, Update extension, 扩展快捷键.
---
# Google Chrome 扩展管理手册
> 适用：Windows 桌面端 Chrome | 更新：2026-03-14

## 1. 核心原则

1. 页面必须是 `chrome://extensions/`，浏览器保持最大化
2. `Browser Assist Locator` 只负责定位，不执行点击 / 输入 / 滚动
3. 最终坐标映射由 daemon 完成，真实动作由 `agent-computer` 执行
4. 扩展代码更新后表现异常，优先点 `Update`
5. 排障顺序：daemon → status → 扩展加载 → 前台浏览器 → locate

---

## 2. 主要使用流程

以下流程覆盖该应用的大部分实际使用场景。

### 主流程 A：管理已安装扩展

1. 访问 `chrome://extensions/`
2. 搜索框输入关键词，实时过滤扩展列表
3. 点击插件开关启停（蓝色=开启，灰色=关闭）
4. 需查详情时点击 Details（版本、权限、无痕模式）
5. 需卸载时点击 Remove → 确认弹窗

### 主流程 B：加载本地扩展

1. 访问 `chrome://extensions/`
2. 开启右上角 Developer mode `(1890, 180)`
3. 点击 Load unpacked `(100, 250)` → 弹出文件夹选择器
4. 选中扩展根目录 → 点击"选择文件夹"

### 主流程 C：更新扩展

1. 确认 `chrome://extensions/` 已开启 Developer mode
2. 点击 Update `(450, 250)`
3. 等待左下角 `Updating...` 消失，更新完成

### 主流程 D：设置快捷键

1. 访问 `chrome://extensions/`
2. 点击左侧导航 Keyboard shortcuts
3. 找到目标扩展，点击 `Not set`
4. 按下目标组合键完成绑定

### 主流程 E：Browser Assist Locator 完整使用流程

**第 1 步：启动 daemon 并确认状态**
```powershell
.\windows-launcher.ps1 daemon start
.\windows-launcher.ps1 browser-assist-status
```
> daemon 正常但扩展未连接时返回 `"connected": false`，属于正常，继续加载扩展即可。

**第 2 步：加载扩展（二选一）**

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
> Browser Assist v1 会校验前台窗口是否为浏览器，以及 URL / 标题是否一致，执行 locate 前浏览器需保持前台。

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
- `mapped.screenCandidates[].screenPoint` — daemon 映射后的桌面绝对坐标（**用这个**）

```powershell
.\windows-launcher.ps1 click --x <screen_x> --y <screen_y>
```

**第 7 步：截图校验**
```powershell
.\windows-launcher.ps1 capture-grid --grid-size 50
```

### 主流程 F：结束 Browser Assist Locator 使用

- **保留扩展**：回到扩展页关闭开关即可
- **彻底卸载**：扩展页点 Remove
- **停止后端**：`.\windows-launcher.ps1 daemon stop`（daemon 停止后扩展仍存在但处于未连接状态）

---

## 3. 稳定规则

### 3.1 路由 / 页面 / 模块

- 扩展管理主页：`chrome://extensions/`
- 快捷键配置页：`chrome://extensions/shortcuts`（或从主页左侧导航 Keyboard shortcuts 进入）
- Chrome 网上应用店：主页左下角 Chrome Web Store 链接

### 3.2 参数 / 状态 / 业务语义

- Developer mode 开启后，左上角才出现 Load unpacked / Pack extension / Update 三个按钮
- 插件开关：蓝色 = 开启；灰色 = 关闭
- `browser-assist-status` 返回 `"connected": false` 不代表异常，仅表示扩展尚未加载
- `browser-assist-status` 返回 `connected: true` 时，同步返回 `extensionVersion` / `browserName` / `lastSeenAt`
- daemon 停止后，扩展仍然存在但处于未连接状态

### 3.3 Browser Assist Locator 职责边界

- **负责**：查找网页元素，返回 `page / viewport / browser / matches`，返回元素 `rect` 和 `clickablePoint`
- **不负责**：点击、输入、滚动、浏览器自动化执行

**链路总结：**
```
扩展页加载 → 扩展连接 daemon → status 显示 connected:true
→ locate 返回 raw+mapped → agent-computer click 执行真实动作
```
扩展负责定位，daemon 负责映射，computer 负责执行。

### 3.4 判断标准

- **直接通过**：`browser-assist-status` 返回 `connected: true` → 可直接发起 locate
- **条件通过**：返回 `connected: false` → 先完成扩展加载流程再重新确认状态
- **直接跳过**：不需要元素定位时，无需启动 daemon 或加载扩展

---

## 4. UI / 坐标 / 页面映射

### 4.1 页面区域

- **顶部搜索框**：实时过滤已安装扩展列表
- **右上角 Developer mode 开关**：开启后解锁 Load unpacked / Pack extension / Update
- **左上角开发者按钮区**：Load unpacked · Pack extension · Update（仅 Developer mode 开启后可见）
- **扩展卡片区**：每张卡片含插件开关、Details、Remove
- **左侧导航**：Keyboard shortcuts 入口
- **左下角**：Chrome Web Store 链接

### 4.2 关键控件 / 坐标

> 坐标在浏览器最大化 + `chrome://extensions/` 时有效；窗口尺寸、系统缩放或浏览器缩放改变时会漂移。

| 元素 | 坐标 / 位置 | 说明 |
| :--- | :--- | :--- |
| Developer mode 开关 | `(1890, 180)` | 右上角，需先开启才能使用开发者按钮 |
| Load unpacked | `(100, 250)` | 左上角，Developer mode 开启后可见 |
| Update | `(450, 250)` | 左上角，Developer mode 开启后可见 |
| Pack extension | 左上角 | Developer mode 开启后可见，点击弹出打包对话框 |

### 4.3 反馈映射

| 动作 | 预期反馈 | 含义 | 下一步 |
| :--- | :--- | :--- | :--- |
| 搜索框输入关键词 | 扩展列表实时过滤 | 筛选生效 | 找到目标扩展后操作 |
| 点击插件开关 | 蓝色 ↔ 灰色切换 | 开启 / 关闭扩展 | 按需继续 |
| 点击 Details | 进入详情页 | 可查版本、权限、无痕模式 | 查看后返回主页 |
| 点击 Remove | 弹出确认框 | 等待确认后卸载 | 确认弹窗完成卸载 |
| 开启 Developer mode | 左上角出现三个开发者按钮 | 开发者功能解锁 | 可使用 Load unpacked / Update |
| 点击 Load unpacked | 弹出文件夹选择器 | 等待选择本地目录 | 选中扩展根目录后确认 |
| 点击 Update | 左下角出现 `Updating...` | 正在重新加载所有扩展 | 等待提示消失 |
| `browser-assist-status` 返回 `connected: true` | 附带 `extensionVersion` / `browserName` / `lastSeenAt` | 扩展已成功连接 daemon | 可发起 locate 请求 |
| `browser-assist-locate` 返回结果 | 含 `matches` / `mapped.screenCandidates` | 元素定位完成 | 取 `screenPoint` 执行 click |

---

## 5. 快捷键

暂无（Chrome 扩展管理页本身无专属快捷键；扩展的快捷键在 Keyboard shortcuts 页配置）

---

## 6. 常见失败与排障

### 排障优先顺序

1. `.\windows-launcher.ps1 daemon start`
2. `.\windows-launcher.ps1 browser-assist-status`
3. 确认扩展已加载且开关开启
4. 确认浏览器是前台窗口
5. 确认当前 URL 为目标页
6. 点 Update 刷新扩展
7. 重新执行 `browser-assist-locate`

### 现象：`browser-assist-status` 返回 `connected: false`

- 可能原因：
  - 扩展尚未加载到 Chrome
  - 扩展已加载但开关处于关闭状态
- 优先检查：
  1. `chrome://extensions/` 中是否存在 Browser Assist Locator 且开关为蓝色
  2. daemon 是否已正常启动
- 处理方式：
  - 按主流程 E 第 2 步加载扩展，或打开扩展开关；daemon 未启动则先执行 `daemon start`

### 现象：扩展加载后行为异常 / 功能不符预期

- 可能原因：
  - 本地扩展代码已更新，但 Chrome 仍加载旧版本
- 优先检查：
  1. 是否刚修改过扩展源码
- 处理方式：
  - `chrome://extensions/` → 确认 Developer mode 开启 → 点 Update `(450, 250)` → 等待 `Updating...` 消失

### 现象：locate 请求报错 / 无匹配结果

- 可能原因：
  - 浏览器不在前台，或当前页 URL / 标题与目标不一致
  - 目标元素不可见或不可交互
- 优先检查：
  1. 浏览器窗口是否为当前前台窗口
  2. 当前页 URL 是否为目标页
  3. 请求体中 `visibleOnly` / `interactiveOnly` 是否过滤掉了目标元素
- 处理方式：
  - 执行 `browser-open-url` 将浏览器置于前台；检查请求体中的筛选条件是否过严

### 现象：坐标点击偏移 / 未命中目标控件

- 可能原因：
  - 浏览器未最大化，或系统缩放 / 浏览器缩放发生变化
- 优先检查：
  1. 浏览器是否处于最大化状态
  2. 系统缩放比例是否变化
- 处理方式：
  - 重新最大化浏览器；若仍偏移，以 locate 返回的 `mapped.screenCandidates[].screenPoint` 为准，不使用硬编码坐标

---

## 7. Self-Improving

当执行过程中出现以下情况时，需要追加记录到 `references/self-improving.md`：

- 原有流程无法覆盖实际高频使用路径
- 排障过程中确认了新的失败模式
- 发现更稳定、更高效的执行方法
- 发现了值得沉淀的新事实

执行要求：

1. 禁止直接修改本 `SKILL.md` 的正文内容
2. 只允许把新增内容追加写入 `references/self-improving.md`
3. 只追加，不覆盖、不删除历史记录
4. 这里只记录观察到的事实，不记录推断，不记录猜测

写法请严格使用 `references/self-improving-template.md` 中提供的模板。