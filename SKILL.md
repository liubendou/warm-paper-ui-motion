---
name: warm-paper-ui-motion
description: Design warm paper-feel UI systems, silky streaming output, and physical micro-interactions. Use for UI design, interface polish, animation, micro-interaction, or streaming UX requests (UI 设计/界面美化/动效/微交互/丝滑). Do not use for backend or data work.
---

# 暖纸 UI · 丝滑流式 · 物理微互动

从「乐小恩」医学教学应用全量实战提炼的设计方法论。三大块：暖白纸感 UI 体系、流式输出双层平滑、kinetics 式物理微互动。做新界面、改视觉、加动效、优化流式体验时按此执行。

## 1. 暖白纸感设计令牌

「讲义感」= 纸色底 + 深墨字 + 衬线标题 + 发丝线 + 直角小圆角。禁止纯白背景 + 大圆角 + 重阴影的「SaaS 风」。

- 纸色系：`paper-base #FAF8F4`（页面底）/ `paper-raised #FFFFFF`（卡片）/ `paper-sunken #F5F1EA`（输入框、侧栏、模式切换底）/ `paper-quote`（引用块、思考块）
- 墨色系：`ink-primary` 深墨正文 / `ink-secondary` / `ink-tertiary` 辅助 / `ink-disabled` / `ink-inverse`（纸色反白字）
- 主色只用一个：靛蓝 indigo-500，仅用于主按钮、选中态、流式左边线。克制是气质的一半
- 功能色三件：朱砂 vermilion（错误/到期提醒）、苔绿 moss（成功）、赭石 ochre（等待/重试提示）
- 阴影只有三档：`shadow-1`（卡片）、`shadow-menu`、`shadow-dialog`。悬浮升档用 shadow-menu，没有 shadow-2
- 圆角 rounded-xs / rounded-sm（2-4px），卡片最大 rounded-md。大圆角立即破坏讲义感
- 字体：标题用衬线（如思源宋体子集，构建期内嵌，font-serif 变量），正文系统黑体。衬线只用于标题与仪式性文字
- 布局：页面自管通栏头（约 52px 会话头），内容列 max-w-[760-792px] 居中；左侧窄导航栏 + 可选右侧上下文面板
- 分隔用发丝线 rule-hair，强调用 rule-strong；不用粗分割条

## 2. 动效三档纪律（无障碍是硬约束）

- 三档：full（全部动效）/ reduced（保守）/ minimal（全静态）。系统 `prefers-reduced-motion: reduce` → 强制 minimal 且用户不可改
- 每个动效组件必须有 minimal 分支：动画退化为静态或瞬时（duration 0），**不是隐藏**
- 持续循环动画（呼吸 ping、光标闪烁）只在有意义状态渲染（如 dueCount>0），且 minimal 档不渲染循环层
- 过渡选择：位移/尺寸/弹性 → spring tokens；透明度/颜色 → duration + easeOut；流式文本永远匀速，不弹

## 3. 物理 spring tokens 与 kinetics 映射法

- tokens：`SPRING.default {stiffness:260, damping:30}`（页面进出）/ `soft {260,26}`（toast、面板）/ `snappy {380,30}`（按钮、chips、pill）；`DURATION.micro 120 / base 200 / slow 600 / draw 800`
- kinetics 库（colorion.co/ckissi，MIT）用法：**只取它调好的 stiffness/damping 物理参数映射进自有 tokens，实现一律走 motion/react**，不抄 CSS keyframes
- 采纳判定三条：反馈闭环（提交/判分有可见终态）、状态可见（后台状态有呼吸提醒）、空间连续（选中指示滑动而非跳变）。光效类（glow/gradient 流动）一律不硬凑
- 按压反馈统一 whileTap {scale:0.94-0.97} + SPRING.snappy

## 4. Radix 与 motion 分层

- 行为归 Radix（焦点管理/Esc/pointerdown/portal/roving-focus），视觉归 motion。菜单、对话框、下拉用 Radix 原语 + 自有纸感样式
- 选中指示滑动（Tab Pill Glide）：motion.span `layoutId="共享名"` 绝对定位铺底，文字层 relative z-[1]；minimal 档换静态 span
- 陷阱：居中用 `-translate-x-1/2` 会与 motion transform 互相覆盖 → 外层 `fixed inset-x-0 flex justify-center` 包裹，motion 只管内层
- 动画不干扰布局：抖动/描边等反馈动画放内容层嵌套 motion.div，不污染外层的 AnimatePresence/popLayout 切换动画

## 5. 流式输出双层平滑

架构与代码骨架、生命周期铁律见 [streaming-smoothness.md](references/streaming-smoothness.md)。核心结论：

- 主进程按 16ms 帧窗口聚合 token 再发 IPC（消息数降 10-50 倍，首帧立即 flush 保 TTFT）
- 渲染层按 tau 匀速吐字：`rate = min(MAX_RATE, max(MIN_RATE, backlog/tau))`，积压越多吐越快；超大积压 FORCE_FLUSH 直喷。三模式 tau：快 0.18 / 标准 0.25 / 高质量 0.35
- 铁律（实战踩坑）：rAF 循环的 ending 状态必须可收敛（吐完即 onDone + dispose）；start() 的重入 guard 不能被长驻 fallback 定时器阻塞；dt 必须钳制（后台切回前台一次性喷完）；页面 hidden 时用低频 interval 兜底并按真实 elapsed 计 dt
- 状态纪律：流式正文/思考流走 ephemeral store（模块级变量 + 订阅），绝不进 zustand 全局（否则每个 token 触发全列表重渲）；秒表类高频 UI 用 rAF 直改 textContent，不走 setState
- 等待观感：thinking 类模型首字可能 30-60 秒，waiting 超 8 秒必须明示「模型思考中」；停止生成必须有即时反馈（按钮变「正在停止…」并禁用）；生成中拦截会话切换必须弹可见提示，静默 return 会让用户以为按钮坏了

## 6. 微互动配方清单

十个配方的场景、最小实现、minimal 降级见 [motion-recipes.md](references/motion-recipes.md)：

Squish Button / Tab Pill Glide / Choice Chips / Submit States(DotsLoader) / Error Shake / Success Check(描边) / Typing Indicator / Notification Slide-in toast / Pulse Badge / Hover Lift

## 7. 验证纪律（CDP 探针）

- Radix 弹层不接受合成 `.click()`——探针必须派发完整序列 pointerdown→mousedown→pointerup→mouseup→click（带坐标与 pointerId）
- 探针经 CDP Runtime.evaluate 注入时，模板字符串里内嵌正则 `\d` 会被 JS 字符串吃掉成 `d`——改用 `[0-9]`
- 断言选择器要冗余：Radix portal 内容查 `[data-radix-popper-content-wrapper]` 或 `[role="menu"]`；菜单项状态同时查 aria-expanded
- 每个动效验证两态：正常态触发 + minimal 档确认静态
