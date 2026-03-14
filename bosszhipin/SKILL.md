---
name: bosszhipin
description: Use when operating BOSS直聘 on Windows desktop as a job seeker, including URL-first search, filter tuning, candidate evaluation, 收藏, 立即沟通, company pages, and security-check fallback. Triggers on BOSS直聘, zhipin.com, 职位搜索, 收藏岗位, 与 HR 沟通, 杭州岗位筛选, agent 岗位判断.
---

# BOSS直聘 Agent 操作手册
> 适用：PC 网页端 · 求职者视角 | 更新：2026-03-14

## 执行原则
1. 优先 URL 直达，不优先手点搜索框
2. 每次动作后先看页面反馈，再决定下一步
3. 左侧列表负责找候选，右侧详情负责最终判断
4. 收藏成功的唯一可靠反馈：按钮从"收藏"变为"取消收藏"
5. 命中安全页立即切换为人工接管或已登录会话

---

## 站点路由

| 页面 | URL |
| :--- | :--- |
| 首页 | `https://www.zhipin.com/` |
| 搜索结果页 | `https://www.zhipin.com/web/geek/jobs?city=<CITY>&query=<KW>` |
| 职位详情页 | `https://www.zhipin.com/job_detail/<JOB_ID>.html` |
| 职位详情页（活动域名） | `https://activity.zhipin.com/job_detail/<JOB_ID>.html` |
| 公司页 A | `https://www.zhipin.com/companys/<CO_ID>.html` |
| 公司页 B | `https://www.zhipin.com/gongsi/<CO_ID>.html` |
| 安全校验页 | `https://www.zhipin.com/web/common/security-check.html?callbackUrl=...` |
| 滑块验证页 | `https://www.zhipin.com/web/user/safe/verify-slider?callbackUrl=...` |

- 杭州城市代码：`city=101210100`
- 未登录或触发风控时，详情页可能被重定向到安全校验页

---

## 页面模型与坐标

搜索结果页（`/web/geek/jobs`）分为 4 个区域：

**① 顶部 header**（屏幕绝对坐标，浏览器最大化时有效）

| 控件 | 坐标 |
| :--- | :--- |
| 首页 | `(410, 185)` |
| 职位 | `(470, 185)` |
| 公司 | `(530, 185)` |
| 消息 | `(1460, 185)` |
| 简历 | `(1525, 185)` |
| 个人界面（hover 展开下拉） | `(1600, 185)` |

**② header 下方：岗位意向 / 搜索行**  
含推荐标签、已设置的岗位意向、"更多"（hover 展开剩余意向）、搜索框、搜索按钮。  
此行搜索框不是首选入口，URL 直达更稳定。

**③ 筛选条件行**（从左到右）：城市 · 求职类型 · 薪资待遇 · 工作经验 · 学历要求 · 公司行业 · 公司规模  
用于逐步收窄结果集，优先用 URL 参数表达，控件用于兜底。

**④ 左侧岗位列表**  
- 点击卡片非公司名区域 → 右侧详情更新，左侧高亮边框切换  
- 点击公司名称 → 进入公司详情页

**⑤ 右侧岗位详情**  
- 收藏按钮：`(1425, 460)`  
- 立即沟通按钮：`(1575, 460)`

> 坐标依赖浏览器最大化，系统缩放或浏览器缩放变化时会漂移。

---

## URL 筛选参数

常见可编码参数（部分筛选条件可直接写入 URL）：

| 参数 | 含义 | 可选值 |
| :--- | :--- | :--- |
| `city` | 城市代码 | `101210100`（杭州） |
| `query` | 搜索关键词（URL 编码） | `agent%E5%BC%80%E5%8F%91` |
| `jobType` | 求职类型（单选） | `1901` 全职 · `1902` 实习 · `1903` 兼职 |
| `salary` | 薪资待遇（单选） | `402` 3K以下 · `403` 3~5K · `404` 5~10K · `405` 10~20K · `406` 20~50K · `407` 50K以上 |
| `experience` | 工作经验（**多选**，逗号分隔） | `101` 不限 · `102` 应届生 · `103` 1年以内 · `104` 1~3年 · `105` 3~5年 · `106` 5~10年 · `107` 10年以上 · `108` 在校生 |
| `degree` | 学历要求（**多选**，逗号分隔） | `202` 大专 · `203` 本科 · `204` 硕士 · `205` 博士 · `206` 高中 · `208` 中专/中技 · `209` 初中及以下 |
| `scale` | 公司规模 | `303`（大公司） |
| `industry` | 公司行业 | `100020` |

**预筛 URL 示例：**
```
# 杭州 + agent开发 + 实习
https://www.zhipin.com/web/geek/jobs?city=101210100&query=agent%E5%BC%80%E5%8F%91&jobType=1902

# 杭州 + agent开发 + 大公司
https://www.zhipin.com/web/geek/jobs?city=101210100&query=agent%E5%BC%80%E5%8F%91&scale=303

# 杭州 + agent开发 + 实习 + 大公司
https://www.zhipin.com/web/geek/jobs?city=101210100&query=agent%E5%BC%80%E5%8F%91&jobType=1902&scale=303

# 实习 + 高薪 + 多经验段 + 多学历（多选用逗号分隔）
https://www.zhipin.com/web/geek/jobs?city=101210100&jobType=1902&salary=407&experience=107,106,105,104,103&degree=208,206,202
```
参数值若后续变化，以页面真实反馈为准。

**筛选优先顺序**：先构造最窄 URL → 进入结果页看反馈 → URL 无法表达时再点击筛选控件。

---

## 反馈驱动映射

| 动作 | 预期反馈 | 含义 | 下一步 |
| :--- | :--- | :--- | :--- |
| 打开搜索结果 URL | 左侧出现岗位卡片，右侧显示默认详情 | 搜索成功 | 看默认岗位是否符合意向，再决定是否调整筛选 |
| 点击筛选 pill | 弹出下拉选项层 | 进入筛选选择态 | 若可表达为 URL 参数则记录并切回 URL 路径 |
| 点击左侧岗位卡片 | 左侧高亮切换，右侧详情更新 | 上下文切换到该岗位 | 只在右侧详情做最终判断 |
| 点击收藏 | 按钮变为"取消收藏" | 收藏成功 | 记录，继续切换岗位 |
| `PageDown` | 页面整体下滚，左侧列表推进 | 批量推进 | 重新看左侧第一条和右侧详情，不假设选中态同步变化 |
| 鼠标滚轮 | 可能滚整页，也可能只滚右侧详情 pane | 焦点敏感，不稳定 | 批量推进优先 `PageDown`，精调详情再用滚轮 |

---

## 岗位判断规则

**直接通过**：
- 标题含 `AI Agent` / `Agent平台`
- 描述含 `Multi-Agent / RAG / Tool Calling / Memory / Planning / Execution / Prompt Engineering`
- 公司为已上市 / 大厂 / 头部互联网
- 地点为杭州

**条件通过**：
- `Agent + Java` → 通过
- 后端平台岗描述中明确包含 Agent 能力 / LLM 接入 / RAG / 工具调用 → 通过

**直接跳过**：
- 纯 Java CRUD / 纯 Spring 微服务，无 Agent / LLM / RAG 痕迹
- 销售、运营、招生、客服
- 非杭州
- 明显非大厂且无强 Agent 技术含量

---

## 推荐工作流

**初始化**（进入页面前先做 URL 预筛）：
1. 固定 `city=101210100`
2. 设定 `query`
3. 按需叠加 `jobType / scale / 融资阶段`
4. 确保浏览器最大化

**单轮循环**：
1. 通过预筛 URL 进入结果页
2. 滑动左侧岗位列表寻找候选
3. 点击一条疑似合适的卡片
4. 在右侧详情做最终判断
5. 符合意向 → 点收藏直到变"取消收藏"，或按需点"立即沟通"
6. 不符合 → 不收藏，回到左侧继续
7. 同屏卡片切换不稳定时，直接 `PageDown` 推进

---

## 反脆弱规则

**命中安全页**（`/security-check` 或 `/verify-slider`）：立即停止高频重试，改用已登录浏览器窗口，必要时人工完成验证。

**滚动**：批量推进优先 `PageDown`，详情精调再用滚轮（右侧详情 pane 会抢滚动焦点）。

**操作节奏**：每次只做一个动作，确认反馈后再做下一步，不连续点击再回头看结果。

---

## Agent 规则摘要

```
1. 优先 URL 直达，不优先手点搜索框。
2. 搜索结果页主路由：/web/geek/jobs?city=<code>&query=<kw>
3. 有"实习 / 公司规模 / 融资阶段"意图时，优先把参数带进 URL（实习：jobType=1902）。
4. 批量推进优先 PageDown，不优先滚轮。
5. 右侧详情是最终判断区，不要只看左侧标题。
6. 收藏成功的唯一反馈："收藏" → "取消收藏"。
7. 命中 /security-check 或 /verify-slider 即人工接管。
8. 公司页优先 /companys/<id>.html，兼容 /gongsi/<id>.html。
9. 职位详情优先直跳 /job_detail/<id>.html。
10. 纯 Java 直接跳过；Agent+Java 可过；纯 Agent 优先。
11. 杭州是硬条件，不满足直接跳过。
```

---

## 求职者操作全指南（人工操作视角）

### 基础设置
- **城市切换**：点击搜索栏左侧城市名 → 弹出城市选择面板，支持热门城市、首字母或直接输入
- **顶部导航**：首页 · 职位 · 公司 · 消息 · 言职 等核心频道
- **个人账号**：hover 右上角头像 → 升级VIP / 个人中心 / 简历中心 / 隐私保护 / 切换招聘者

### 发现职位
- **分类导航**（随便看看）：首页左侧行业分类树，hover 主分类展开细分岗位
- **推荐信息流**（捡漏）：向下滑动，系统展示精选 / 最新 / 热招职位及热门企业
- **关键词搜索**（最常用）：顶部搜索框输入关键词 → 点击"搜索"

### 条件筛选
进入搜索结果页后，使用顶部筛选栏精准定位：行政区划 / 商圈 · 职位类型（全职/兼职/实习）· 薪资待遇 · 工作经验 · 学历要求 · 公司行业 · 公司规模 · 融资阶段。勾选后实时刷新列表。

### 职位评估（左侧列表 + 右侧详情）
- 左侧快速浏览岗位名称、薪资、公司名、融资情况、HR 活跃状态
- 点击左侧卡片 → 右侧加载完整 JD（职责、任职要求、工作地址、公司信息）
- 暂不沟通但感兴趣 → 点右上角"☆ 收藏"备用

### 发起沟通
1. 点击右侧绿色"立即沟通"
2. 确认弹窗（发送微简历 + 默认打招呼语）
3. 页面跳转到消息聊天窗口，可实时对话、发送附件简历、询问面试流程
4. 返回继续：点浏览器后退或顶部搜索栏，回到职位列表

**小贴士**：批量沟通前务必通过"简历中心"确保简历已更新，这是 HR 收到的第一块敲门砖。

---

## 参考链接

- 首页：<https://www.zhipin.com/>
- 搜索示例：<https://www.zhipin.com/web/geek/jobs?city=101210100&query=agent%E5%BC%80%E5%8F%91>
- 公司页示例：<https://www.zhipin.com/companys/074ff7b3eb1a94a903Z62dy4.html>
- 职位详情示例：<https://www.zhipin.com/job_detail/b34bf1e030cc298103V_0t68FVRX.html>
- 使用帮助：<https://www.zhipin.com/web/common/protocol/use-help.html>
- 防骗指南：<https://news.zhipin.com/web/common/protocol/cheat-guide.html>