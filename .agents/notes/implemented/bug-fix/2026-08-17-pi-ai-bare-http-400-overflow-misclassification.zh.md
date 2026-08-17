# Agent Note: 停止将非 Cerebras 提供方的裸 HTTP 400/413（无 body）误判为上下文溢出

Status: implemented

English | [中文](2026-08-17-pi-ai-bare-http-400-overflow-misclassification.md)

## Problem

`dsh-llm-pi-ai` 的 `mapStopReason` 将溢出检测委托给 pi-ai 的 `isContextOverflow`，后者用正则 `/^4(?:00|13)\s*(?:status code)?\s*\(no body\)/i` 匹配裸 `400/413 (no body)` 拒绝。这条正则仅针对 Cerebras——它的网关用空的 HTTP 400/413 表示上下文溢出。其他 OpenAI 兼容网关在完全无关的原因（配额、鉴权、请求守卫、瞬时故障）下也会返回同样的空拒绝，且没有 body 可供区分。

当非 Cerebras 提供方（如 SiliconFlow）因上述无关原因返回 `400 status code (no body)` 时，pi-ai 的匹配器把它升级为 `CONTEXT_WINDOW_EXCEEDED`。Harness 随后运行溢出恢复：触发 compaction，其摘要请求向同一个失败的提供方重发并以同样方式失败，整个回合陷入重试循环，最终把一个与上下文长度无关的失败呈现为"上下文溢出"。

## Decision

当错误文本是裸 `400/413 (no body)` 且模型的 `provider` 不是 `cerebras` 时，`mapStopReason` 跳过 pi-ai 的 `isContextOverflow` 判定。错误随后落入既有的 `classifyPiAiError` 路径，把 `400` 归类为 `INVALID_REQUEST` 而非 `CONTEXT_WINDOW_EXCEEDED`。Cerebras 保留溢出升级，而描述性溢出文本（由 `harnessOverflow` 分支经 `isContextWindowExceededError` 匹配）不受影响。

## Alternatives considered

**在 pi-ai 的 `isContextOverflow` 内修复。** 否决：pi-ai 是上游依赖；那条正则在 pi-ai 内本就是 Cerebras 专用，但匹配器在 harness 边界被无差别应用。在 harness 边界设防，让修复留在拥有提供方路由的包里，无需等待上游发版。

**仅凭 HTTP 状态归类空 400（任何裸 400 都归为 `INVALID_REQUEST`）。** 否决：会回退 Cerebras——它的真实溢出信号恰恰就是这个形状。提供方守卫保留了该行为。

**通过修改 pi-ai 转发原始 Error 来暴露原始 body。** 否决：更广的上游改动且无即时路径；harness 层的守卫现在就能修复误判，原始 body 的缺口仍由 `classifyPiAiError` 中既有的 `XXX(pi-ai upstream)` 注释跟踪。

## Consequences

- 非 Cerebras 提供方的裸 `400/413 (no body)` 被报告为 `INVALID_REQUEST`，因此 harness 不再运行注定失败的 compaction 重试循环，真实的拒绝到达用户而非被编造成"上下文溢出"。
- Cerebras 的溢出检测不变：提供方守卫为其网关保留了 `CONTEXT_WINDOW_EXCEEDED`。
- 描述性溢出错误（如 `exceeds the context window`）仍经 `isContextWindowExceededError` 映射为 `CONTEXT_WINDOW_EXCEEDED`，与此守卫无关。
- 空 400 的真实原因（配额、鉴权、守卫）仍无法从响应 body 恢复；它被报告为 `INVALID_REQUEST`，这在线上可揭示的范围内是准确的。
