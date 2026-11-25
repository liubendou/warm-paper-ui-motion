# 十个微互动配方 — 场景 · 最小实现 · minimal 降级

统一前提：物理参数取自 kinetics（colorion.co / github.com/ckissi，MIT）预设映射的 SPRING tokens；实现一律 motion/react；每个配方必须有 minimal 档静态分支。选取标准：反馈闭环 / 状态可见 / 空间连续，光效类不采纳。

## 1. Squish Button（按压回弹）
- 场景：所有主操作按钮（发送/生成/导入/新对话）
- 实现：motion.button `whileTap={{ scale: 0.97 }}` + SPRING.snappy；disabled 时 whileTap 传 undefined
- minimal：whileTap 传 undefined 即静态

## 2. Tab Pill Glide（选中指示滑动）
- 场景：分段切换（模式 快速/标准/高质量、Tab 组）
- 实现：每项 relative；选中项渲染 `<motion.span layoutId="唯一共享名" className="absolute inset-0 ..." transition={SPRING.snappy} />` 铺底，文字层 `relative z-[1]`。切换时 pill 自动从旧位置滑到新位置（layout 动画）
- minimal：选中项换静态 span（无 motion），视觉不变

## 3. Choice Chips（选项弹跳）
- 场景：小选项组（题数 3/5/8 题、筛选 chips）
- 实现：选中态 `animate={{ scale: [1, 1.12, 1] }}`（一次性 keyframes）+ SPRING.snappy + 选中填充色；whileTap 0.94
- minimal：跳过 scale keyframes，只留填充色变化

## 4. Submit States（提交态三点加载）
- 场景：异步主按钮（生成测验/导入 PDF）——按钮文字换「动作中 + DotsLoader」并 disabled
- 实现：DotsLoader 三个 motion.span `animate={{ y: [0, -4, 0] }}` repeat:Infinity，相位延迟 `i * 0.14`，duration 0.5；容器 flex items-center
- minimal：三点改 opacity 循环（0.4↔1），不动 y

## 5. Error Shake（错误摇头）
- 场景：判分答错（题目卡）、表单校验失败（API Key 输错）
- 实现：内容层嵌套 motion.div `animate={bad ? { x: [0, -9, 9, -6, 6, -2, 0] } : { x: 0 }}` `transition={{ duration: 0.42, ease: 'easeOut' }}`
- 关键：放嵌套层，不干扰外层 AnimatePresence/popLayout 的进出动画；卡片结尾会出现两个 </motion.div> 别误删
- minimal：始终 { x: 0 }，用红色文案静态表达错误

## 6. Success Check（成功描边）
- 场景：判分答对的判分行；与掌握度环同一 stroke 视觉语言
- 实现：motion.svg viewBox 18x18；motion.circle `pathLength: 0→1` duration DURATION.base/1000；对勾 path 延迟 0.16s 再描；strokeWidth 1.5-1.8
- minimal：initial={false} 直接满 pathLength（静态图形）

## 7. Typing Indicator（等待思考）
- 场景：LLM waiting/thinking 相位。已有「占位条 + 光标闪烁 + 计时秒表」即等价实现，不重复造
- 要点：waiting 超 8 秒追加「模型思考中，首次回答可能需要 30–60 秒」；秒表 rAF 直改 textContent

## 8. Notification Slide-in（顶部通知）
- 场景：导入成功等轻量结果反馈（页面级 toast，3.2s 自动退场）
- 实现：外层 `pointer-events-none fixed inset-x-0 top-10 z-50 flex justify-center`，内层 motion.div initial {opacity:0, y:-28, scale:0.96} → animate {opacity:1, y:0, scale:1} SPRING.soft，exit {opacity:0, y:-12}，包 AnimatePresence
- 陷阱：居中绝不能用 -translate-x-1/2（与 motion transform 互覆），靠外层 flex justify-center
- minimal：initial 只 {opacity:0}

## 9. Pulse Badge（呼吸提醒）
- 场景：后台状态需要被看见（到期复习卡数 > 0）
- 实现：Tailwind animate-ping 层（absolute inset-0 rounded-full bg-色/60）+ 静态实心点层；仅条件状态渲染
- minimal：不渲染 ping 层，保留静态点

## 10. Hover Lift（悬停浮起）
- 场景：可点击/可聚焦的统计卡
- 实现：纯 CSS `transition-[transform,box-shadow] duration-[160ms] ease-out hover:-translate-y-0.5 hover:shadow-menu`（阴影只有三档，升档用 shadow-menu）
- minimal/motion-reduce：`motion-reduce:hover:translate-y-0`，只升阴影

## kinetics 评估法（拿到一个预设库时怎么用）

1. 全量解析预设（名称+描述+参数），按「交互输入 / 反馈状态 / 表面动效」三类分组
2. 只对每条预设问三个问题：有没有对应的真实状态变化？是不是反馈闭环的一环？是否空间连续性可用 layoutId 表达？
3. 输出清单：preset → 适合哪个界面 → 为什么；没有合适场景的直说不硬凑（尤其 glow/光斑/渐变流动类一律排除）
4. 落地时只抄物理参数（stiffness/damping/duration），实现走自有 tokens + motion/react，保证全局一致与可降级
