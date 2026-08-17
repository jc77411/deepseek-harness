# Agent Note: Stop labeling bare HTTP 400/413 (no body) as context overflow for non-Cerebras providers

Status: implemented

English | [中文](2026-08-17-pi-ai-bare-http-400-overflow-misclassification.zh.md)

## Problem

`dsh-llm-pi-ai`'s `mapStopReason` delegated overflow detection to pi-ai's `isContextOverflow`, which matches a bare `400/413 (no body)` rejection against the pattern `/^4(?:00|13)\s*(?:status code)?\s*\(no body\)/i`. That pattern exists only for Cerebras, whose gateway reports context overflow as an empty HTTP 400/413. Other OpenAI-compatible gateways return the same empty rejection for unrelated reasons — quota, auth, request guard, transient service failure — with no body to distinguish them.

When a non-Cerebras provider (e.g. SiliconFlow) returned `400 status code (no body)` for one of those unrelated reasons, pi-ai's matcher promoted it to `CONTEXT_WINDOW_EXCEEDED`. The harness then ran its overflow recovery: it triggered compaction, whose summarization request re-sent the same failing provider and failed the same way, and the turn collapsed into a retry loop that surfaced "context overflow" for a failure that was never about context length.

## Decision

`mapStopReason` skips pi-ai's `isContextOverflow` verdict when the error text is a bare `400/413 (no body)` and the model's `provider` is not `cerebras`. The error then falls through to the existing `classifyPiAiError` path, which routes the `400` to `INVALID_REQUEST` instead of `CONTEXT_WINDOW_EXCEEDED`. Cerebras keeps the overflow promotion, and descriptive overflow text (matched by `isContextWindowExceededError` via the separate `harnessOverflow` arm) is unaffected.

## Alternatives considered

**Fix inside pi-ai's `isContextOverflow`.** Rejected: pi-ai is an upstream dependency; the pattern is deliberately Cerebras-scoped there but the matcher is applied provider-agnostically at the harness boundary. Guarding at the harness boundary keeps the fix in the package that owns the provider routing without waiting on an upstream release.

**Classify the empty 400 by HTTP status alone (`INVALID_REQUEST` for any bare 400).** Rejected: it would regress Cerebras, whose real overflow signal is exactly this shape. The provider guard preserves that behavior.

**Surface the raw body by patching pi-ai to forward the original Error.** Rejected: broader upstream change with no immediate path; the harness-level guard fixes the misclassification now and the raw-body gap remains tracked by the existing `XXX(pi-ai upstream)` comment in `classifyPiAiError`.

## Consequences

- A bare `400/413 (no body)` from a non-Cerebras provider is reported as `INVALID_REQUEST`, so the harness no longer runs a doomed compaction retry loop and the real rejection reaches the user instead of a fabricated "context overflow".
- Cerebras overflow detection is unchanged: the provider guard keeps `CONTEXT_WINDOW_EXCEEDED` for that gateway.
- Descriptive overflow errors (e.g. `exceeds the context window`) still map to `CONTEXT_WINDOW_EXCEEDED` through `isContextWindowExceededError`, independent of this guard.
- The true reason for an empty 400 (quota, auth, guard) is still not recoverable from the response body; it is reported as `INVALID_REQUEST`, which is accurate to the extent the wire can reveal.
