# Plan: Manual Immediate Retry During Provider Retry Backoff

## Problem

When a provider returns a retryable error (e.g. Minimax 529 overloaded), the session enters an exponential backoff sleep. Currently the only way out for the user is to press `esc` to interrupt the whole turn. There is no key to say "skip the remaining wait and retry right now".

## Goal

Add a keybinding that, while the status bar shows a retry countdown, immediately cancels the sleep and re-attempts the provider call in the same turn.

## Non-goals

- Change the retry policy decision of what is retryable.
- Change the configured max delay cap.
- Add a visible button; this is a keyboard shortcut only.

## Proposed design

1. **Session status model**
   - Keep `type: "retry"` as the visible state.
   - Add a per-session `Deferred<void>` or `Latch` that the retry policy can await alongside the sleep. When triggered, the sleep is aborted and the next attempt starts immediately.
   - Expose a new `SessionRunState.retryNow(sessionID)` operation that completes the latch.

2. **Retry policy**
   - `SessionRetry.policy` already uses `Effect.sleep(Duration.millis(wait))`. Wrap the sleep in `Effect.raceFirst(wakeSignal)` or similar so an external wake signal short-circuits it.
   - Pass the wake signal / latch into `policy` from the processor.

3. **Processor**
   - In `SessionProcessor.process`, create a fresh wake latch for each `process` call and pass it to `SessionRetry.policy`.
   - On success/failure/interrupt, the latch is discarded (new `process` gets a new one).

4. **TUI keybinding**
   - Add a new keybind name `session_retry_now` in `packages/tui/src/config/keybind.ts`, default to `r` (or `ctrl+r`).
   - Map it to a new command `session.retry_now` in the command dispatch table.
   - In the session route (`packages/tui/src/routes/session/index.tsx`), wire the command to call the runtime's `retryNow` for the current session, but only when status is `type: "retry"`.

5. **Runtime bridge**
   - The TUI talks to the backend through the opencode command/runtime system. Need to find the existing path from TUI command to `SessionRunState.Service`.
   - Likely add a handler in the session route or a command shim that yields `SessionRunState.Service` and calls `retryNow`.

6. **Tests**
   - Unit test in `test/session/retry.test.ts`: a retry policy that receives a wake signal returns the next attempt immediately without sleeping.
   - Effect/integration test in `test/session/processor-effect.test.ts`: simulate a retryable failure, trigger retry-now, assert a second provider request is fired before the backoff delay elapses.
   - Optional TUI test: keybind is registered and command exists.

## Open questions

- What is the correct runtime bridge from TUI key command to `SessionRunState.Service`? Need to inspect existing session commands in `packages/tui/src/routes/session/index.tsx` and the opencode command system.
- Should `retry_now` be allowed while a subagent is retrying, or only the main session? Probably only the currently focused session.
- Should the wake signal also cancel an in-flight sleep in the lower-level `packages/llm/src/route/executor.ts` retry? Probably not for now; that layer has its own small caps and is per-request.

## Files expected to change

- `packages/opencode/src/session/retry.ts`
- `packages/opencode/src/session/processor.ts`
- `packages/opencode/src/session/run-state.ts`
- `packages/opencode/src/session/status.ts` (maybe, if we store latch reference in status)
- `packages/tui/src/config/keybind.ts`
- `packages/tui/src/routes/session/index.tsx`
- `packages/opencode/test/session/retry.test.ts`
- `packages/opencode/test/session/processor-effect.test.ts`
- This plan file may be deleted after implementation if not kept.
