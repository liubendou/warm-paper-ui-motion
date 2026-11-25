# 流式输出双层平滑 — 架构与铁律

目标：长回复流式输出「丝滑不顿挫」，IPC 开销低，结束/中断可靠收尾。

## 双层架构

```
LLM SSE token（可达每秒几十个）
   │
   ▼
【粗平滑】主进程 StreamBridge —— 16ms 帧窗口聚合
  push() 首帧立即 flush（保 TTFT 观感），后续 16ms 定时聚合
  end() 前必须 flush（否则丢尾字）
  → IPC 消息数降 10-50 倍
   │
   ▼
【细平滑】渲染层 StreamScheduler —— tau 匀速吐字
  push() 把增量切成 grapheme（Intl.Segmenter，防代理对撕裂）入 backlog
  rate = min(MAX_RATE, max(MIN_RATE, backlog / tau))
    backlog 大 → 追速快；追平 → 减速 → 「丝滑与一顿一停的分界线」
    参考值：MIN_RATE 20 / MAX_RATE 800（字/秒）/ FORCE_FLUSH 4096 直喷
  backlog 吐完且 ending → onDone() → finalize（拉落库终稿、清流式态）
```

三模式 tau（期望消化积压的时间常数，秒）：快速 0.18 / 标准 0.25 / 高质量 0.35。质量越高吐得越从容。

## 状态纪律（性能红线）

- 流式正文与思考流放 **ephemeral store**（模块级变量 + 轻订阅），**绝不进 zustand 全局**——每个 token 触发全消息列表重渲是灾难
- 全局 store 只存低频相位：streaming 相位（waiting/thinking/text）、reqId、重试提示
- 思考计时秒表：rAF 直改 `textContent`，不走 setState
- 思考流只本地展示与落库，组装下一轮请求时**永不回传**（省 token 且稳缓存）

## rAF 生命周期铁律（实战踩坑：一个死循环毁掉整个对话功能）

1. **ending 必须可收敛**：tick 循环条件不能写成 `if (backlog.length || ending)`——ending 恒真会让 rAF 永不停止、onDone 永不触发、流式态永远挂着（表现为：回答完了还在转、点停止不停、新对话/历史全被静默拦截）。正确形态：

```ts
const tick = (now: number): void => {
  const dt = Math.min((now - this.last) / 1000, 0.1)  // 铁律3
  this.last = now
  this.drain(dt)
  if (this.backlog.length) { this.raf = requestAnimationFrame(tick); return }
  this.raf = 0
  if (this.ending) { this.dispose(); this.opts.onDone?.() }  // 吐完即收尾
  // 非 ending：停 rAF 等 push
}
```

2. **重入 guard 不能被长驻资源阻塞**：`start()` 的 guard 若写成 `if (raf || fallback) return`，而 fallback 定时器长期存在（仅 hidden 时干活），则 rAF 停转后 push 再也无法重启循环——流直接断死。guard 只查 `raf`；fallback 只建一次、dispose 时清除

3. **dt 必须钳制**：切回前台时 dt 可能是几秒，不钳会一次性喷完积压；上限 0.1s

4. **后台兜底**：页面 hidden 时 rAF 暂停，低频 interval（50ms）兜底 drain；hidden 页 interval 被 Chromium 钳到 ≥1s，**必须按真实 elapsed 计 dt**（钳 2s）；rAF 挂起时 tick 永不执行，ending 完成的 onDone 必须在兜底路径补发，否则后台完成的流永远卡在流式态

## 中断与竞态

- abort：渲染层按钮点击后**立即**给反馈（文案变「正在停止…」+ disabled），流清空后复位。主进程 AbortController 贯穿 fetch 与读循环，abort 抛 `DeepSeekError('abort')` 不可重试，catch 里落库半截内容（status=2）后发 `end(reason:'abort')`
- end 事件带 reason：`abort` / `error` 时**立即 finalize** 不等匀速吐完（半截内容已在库里）；`done` 才走「吐完积压再 finalize」的丝滑收尾
- ack 竞态：快速流可能在 ack 返回前已 finalize（streaming=null）——ack 处理时先查当前 streaming，不能无脑复活
- finalize 与新流竞态：finalize 前记录 sid，且执行时校验当前 reqId 仍是自己的，防止把下一轮流式态清掉
- 生成中拦截会话切换（newSession/selectSession 的 streaming guard）必须**可见**：按钮点击时弹 toast 提示，静默 return = 用户以为按钮坏了
- 等待观感：waiting 相位超 8 秒显示「模型思考中，首次回答可能需要 30–60 秒」——thinking 模型首字等待长，不提示就是「卡住了」

## 提示词缓存纪律（配套）

- 系统提示分层：L0 core 逐字节固定 / L1 模式 skill 会话内固定 / L2 学习者画像**会话开始时定格** / L3 历史 append-only / L4 本轮 RAG 只拼在最后一条 user 消息上
- 任何层的字节变化都会击穿缓存命中——L2 快照禁止随对话更新
