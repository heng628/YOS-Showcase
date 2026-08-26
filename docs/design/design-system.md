# 设计系统（Design System）

- 状态：草案（待用户批准）；**仅固化 MVP**
- 依据：Step 10 视觉语言设计（已确认）；`AGENTS.md` §16/§17
- 视觉方向：**「克制的高密度数据工作环境 + 环境智能」**
  - 关键词：克制 / 清晰层级 / 数据密度分级 / 可信 / 安静专注 / 系统感(OS) / 智能内敛 / 一致性

---

## 1. Token 架构：Primitive → Role → Component

- **Primitive**：原始值（如 `gray-500`、`spacing-4`）。
- **Role**：语义角色（如 `color.text-primary`、`radius.card`）——组件**只消费 Role**。
- **Component**：组件别名（如 `button.radius`），映射到 Role。
- 实现：**CSS Variables + Tailwind 配置**；未来 **Theme JSON 可整体替换 Role 值**。
- 铁律：**组件不得硬编码颜色/间距/字体；不得自定颜色，只能消费 Token。**

---

## 2. Color（语义色）

| 分组 | Role Token |
|---|---|
| 中性面 | `background` / `surface` / `surfaceElevated` / `border` / `text.primary` / `text.secondary` / `text.muted` |
| 数据与动作 | `accent`(主交互/数据强调) / `data.*`(图表系列) / `metric`(指标强调) |
| 状态 | `success` / `warning` / `error` / `info` |
| 过程 | `proposal` / `pending` |
| **AI** | `ai`（独立、低饱和；仅小标记/细边框/单行标签，**不做大块填充**） |

- AI 色与 `accent` 明确区分；禁止大面积 AI 渐变、发光、霓虹。

## 3. Typography

- 层级：`display / h1 / h2 / h3 / body / bodySmall / caption / dataNumber / table / code`
- **数据数字**：tabular figures（tnum）、右对齐、强层级；单位（¥/%）与小数位统一；± 变化带方向图标（不单靠颜色）；公式/定义用等宽（`code`）。

## 4. Spacing / Density

- 基座 4px；scale：`2 / 4 / 8 / 12 / 16 / 24 / 32 / 48 / 64`。
- **两档 Density：`comfortable`（默认，Simple 模式）与 `compact`（专业模式默认）**。
- 只改 Density + Visibility，**同一对象模型、同一组件体系**（绝不两套 UI）。

## 5. Radius

| 元素 | token |
|---|---|
| Button | sm(4) / md(8) |
| Input | md(8) |
| Card | md(8) |
| **Module** | **lg(12)**（内容单元） |
| Modal | lg(12) |
| Command Center | xl(16) / lg(12) |
| Toast | md(8) / full |
| Confirmation Card | lg(12) |

统一尺度：控件小（4–8）、卡片中（8–12）、浮层大（12–16）。

## 6. Shadow / Border

- **深度 = 背景层级优先**；`border` 用于表格行列/输入/分隔/选中；`shadow` 只给浮层（level1/level2）。
- 默认卡片无边框无阴影（靠 surface）；一个浮层最多一个阴影层级；三者可都不用时靠排版+间距。
- 禁止卡片套卡片、边框套边框、阴影套阴影。

## 7. Motion

- 需要：Module 拖拽、AI 内容出现（柔和浮现）、数据同步/分析（阶段更新）、状态切换、Confirmation、Toast、Command Center 打开。
- 禁止：普通内容入场炫技、数据滚动动画、装饰动画、拖慢操作的动效。
- 时长：fast 150ms / base 200ms / slow 300ms。原则：**让用户理解发生了什么，不是炫技。**

## 8. Z-index

- `base / sticky / overlay / commandCenter / modal / toast`（递增）。

## 9. Status 视觉（不单靠颜色）

- 状态 = **图标(主) + 文案 + 颜色(补充)**：
  - Healthy ✓ / Warning ⚠ / Error ✕ / Stale 时钟 / Disconnected 断链 / Pending 沙漏 / Running spinner / Completed ✓圈 / Paused ⏸ / Muted 铃关 / Archived 归档盒。
- 异步态（Running/Pending）加轻动效；色弱可读。

## 10. AI 视觉语言

- 区别方式：**小标识 + 低饱和 AI 色（细边框/小标记）+ 轻动效 + 同一卡片语法**。
- **禁止**：大面积 AI 渐变、发光、聊天头像式 UI、AI 变成另一个 App。
- 目标：一眼知道是 AI 内容，但不抢注意力。

## 11. Module 视觉（MVP 3 种展示）

- 结构：`Header(标题/状态/菜单) → Primary Data(数值) → Visualization → Context/Insight(可选) → Action(可选)`
- 展示形态仅 3 种：**KPI / Chart / Table**（Insight=Context 行+Overview 系统层；AI=横切能力，均非模块类型）。

## 12. Table 规则

- 固定头/排序箭头/选中(accent 微背景+左指示)/筛选 chips/骨架加载/内联错误/Empty(解释+下一步+AI 帮助)。
- 数据差异信号：**Changed**=accent 微高亮+小标识+before→after 提示；**New**=轻高亮+「新」；**Stale**=「数据截至 X 日」banner+静默降级。用 accent 而非大红告警。

## 13. Confirmation Card（风险分级）

- low/med：中性卡片 + accent 边框 + 标准 确认/修改/拒绝/忽略。
- high：amber 强调 + 强制展开影响与可撤销 + 风险标签 + 高确认意图。不用巨大红色警告（防疲劳）。

## 14. Charts（MVP：Line/Bar/Area/Number/Table）

- 每系列单色、无渐变、弱 grid、统一 tooltip（tnum）、对比用虚线、异常用**小而明确标记 + AI 注释**（不用大红尖峰）。

## 15. Empty / Loading / Error

- Empty：解释 + 推荐下一步 + AI 帮助（非大空盒子）。
- Loading：阶段过程/骨架，**不用 spinner-only**，AI 显示"正在做什么"。
- Error：什么/为什么/怎么办 + 可执行恢复动作。

## 16. Command Center（Ctrl/Cmd+K）

- 形态：**命令面板 + AI 工作空间**（非完整 Chat UI）：输入区(上下文提示) → 推荐命令 → AI 内联回答 → Proposal → 任务进度 → 历史。
- 视觉：浮层面板（radius-xl、浮层阴影、宽 560–680px）。

## 17. Desktop Layout

- Sidebar 220–260px（可收缩 icon 56–64px）；内容流体或 max ~1400px；页面 padding 16–24px；Module gap 12–16px（compact 8–12px）；顶栏 48–56px；Modal 640/960。

---

## MVP 只做

Default Theme + 完整 Token 体系 + 上述语义色/密度/状态/AI 语言/Module 3 形态/Confirmation 分级/Empty/Loading/Error/Motion/Command Center 面板。**不做 Theme Editor / Theme Marketplace。**
