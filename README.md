# Contribution 2: `Add message queue support for V1 conversations during WebSocket connection`

**Contribution Number:** 2  
**Student:** Jessica Lang  
**Issue:** [OpenHands/OpenHands#12279](https://github.com/OpenHands/OpenHands/issues/12279)  
**Status:** Phase II investigation complete

**Important note as of June 14, 2026:**

- Issue `#12279` is still open.
- Related PR `#14692` is still open.
- The follow-up implementation branch created from this investigation is now published at `sreytouch/OpenHands:test/v1-pending-message-queueing`.
- That follow-up branch has since been submitted upstream as PR [OpenHands/OpenHands#14860](https://github.com/OpenHands/OpenHands/pull/14860), now ready for review.
- The current `main` branch in my local clone is commit `2a3f06a75d4b166bef77a0240143efdcc092cfc2` from June 9, 2026.
- That codebase already contains commit `238cab4d08ebb176bf972e37ef55fa40615dbf82` from March 16, 2026: `fix(frontend): prevent chat message loss during websocket disconnections or page refresh (#13380)`.

Because of that March 16, 2026 change, I cannot honestly claim that the original January 6, 2026 bug still reproduces unchanged on current `main`. My Phase II work therefore became a reproduction investigation and scope check: verify whether issue `#12279` is still a valid contribution target, or whether the remaining work has narrowed to a smaller reconnect or delivery edge case.

---

## Reproduction Process

### Environment Setup

For Phase II, I did the following local setup and code-tracing work:

1. Confirmed my Phase I issue choice was still the same target: `OpenHands/OpenHands#12279`.
2. Confirmed on June 16, 2026 that:
   - the issue is still open
   - the related PR `#14692` is also still open
3. Verified my fork exists at `https://github.com/jessicalang2595/OpenHands`.
4. Cloned my fork locally:
   - `git clone https://github.com/jessicalang2595/OpenHands.git`
5. Confirmed the local clone is on:
   - branch: `main`
   - HEAD commit: `2a3f06a75d4b166bef77a0240143efdcc092cfc2`
6. Inspected the current code paths most relevant to the issue:
   - `frontend/src/contexts/conversation-websocket-context.tsx`
   - `frontend/src/components/features/chat/interactive-chat-box.tsx`
   - `frontend/src/components/features/chat/chat-interface.tsx`
   - `frontend/src/api/pending-message-service/pending-message-service.api.ts`
   - `frontend/__tests__/conversation-websocket-handler.test.tsx`
7. Checked git history on the V1 WebSocket message path and found a later fix already landed on March 16, 2026 in commit `238cab4d08ebb176bf972e37ef55fa40615dbf82`.

### What I Found Before Full Runtime Reproduction

The current `main` branch no longer matches the original issue description exactly:

- `frontend/src/components/features/chat/interactive-chat-box.tsx` now explicitly allows users to submit messages during `LOADING` state and says those messages will be queued server-side and delivered when the conversation becomes ready.
- `frontend/src/contexts/conversation-websocket-context.tsx` now falls back to `PendingMessageService.queueMessage(...)` whenever the V1 WebSocket is not open.
- `frontend/src/components/features/chat/chat-interface.tsx` now distinguishes between immediately-sent and queued messages and avoids showing optimistic UI for queued messages.
- `frontend/src/api/pending-message-service/pending-message-service.api.ts` documents a dedicated API for storing pending messages until the conversation is ready.

This means the original problem statement from January 6, 2026 appears at least partially addressed in the current codebase.

### Steps to Reproduce or Verify on Current `main`

If I continue this issue, these are the manual verification steps I would follow on the current codebase:

1. Start OpenHands locally using the project's current setup instructions.
2. Open a V1 conversation while the runtime or sandbox is still starting.
3. Submit a user message before the native WebSocket reaches `OPEN`.
4. Observe whether the chat input accepts the message during startup.
5. Observe whether the message is queued and later delivered once the conversation becomes ready.
6. Repeat the test during a reconnect or refresh scenario to check for:
   - dropped messages
   - duplicate delivery
   - stale queued messages after conversation changes

### Current Reproduction Conclusion

My current conclusion is:

- I have strong code evidence that the original missing-queue behavior has already been addressed on `main`.
- I do not yet have honest end-to-end runtime evidence that the exact original bug still reproduces on the current branch.
- If this issue is still valid, the remaining bug is likely narrower than the January issue text and may be closer to reconnect timing, delivery confirmation, or an untested edge case.

### Reproduction Evidence

- Phase I issue-selection comment: [jessicalang2595 comment on #12279](https://github.com/OpenHands/OpenHands/issues/12279#issuecomment-4618034397)
- Current working branch available: [sreytouch/OpenHands/tree/test/v1-pending-message-queueing](https://github.com/sreytouch/OpenHands/tree/test/v1-pending-message-queueing)
- Later upstream PR from this investigation: [OpenHands/OpenHands#14860](https://github.com/OpenHands/OpenHands/pull/14860)
- Current local clone HEAD: `2a3f06a75d4b166bef77a0240143efdcc092cfc2` on June 9, 2026
- Relevant landed fix: [commit `238cab4d08ebb176bf972e37ef55fa40615dbf82`](https://github.com/OpenHands/OpenHands/commit/238cab4d08ebb176bf972e37ef55fa40615dbf82)
- Related issue still open: [OpenHands/OpenHands#12279](https://github.com/OpenHands/OpenHands/issues/12279)
- Related PR still open: [OpenHands/OpenHands#14692](https://github.com/OpenHands/OpenHands/pull/14692)

#### Code Evidence

- `frontend/src/contexts/conversation-websocket-context.tsx` lines `877-900`
  - V1 `sendMessage` now queues through `PendingMessageService` when the WebSocket is not open.
- `frontend/src/components/features/chat/interactive-chat-box.tsx` lines `154-159`
  - The chat box now allows submission during `LOADING` and documents server-side queueing.
- `frontend/src/components/features/chat/chat-interface.tsx` lines `184-187`
  - Queued messages are handled differently from immediately sent messages.
- `frontend/__tests__/conversation-websocket-handler.test.tsx`
  - Existing tests cover connected send behavior, but I did not find clear test coverage proving end-to-end queue-and-deliver behavior for the startup case described in `#12279`.

#### Branch Link Note

At Phase II time, I intentionally did not publish a branch yet because the issue scope looked stale on current `main`. After continuing into Phase III, I published the narrower follow-up work at:

- [sreytouch/OpenHands/tree/test/v1-pending-message-queueing](https://github.com/sreytouch/OpenHands/tree/test/v1-pending-message-queueing)

That branch reflects the actual test-coverage contribution that came out of this investigation rather than a misleading "queue support from scratch" implementation.

---

## Solution Approach

### Understand

The original issue said V1 conversations did not support queueing messages before the WebSocket connection was fully established. Based on the current codebase, that exact gap no longer appears to be the main problem. The code now attempts to preserve messages server-side when the WebSocket is unavailable.

So the real Phase II question is no longer "How do I add queueing from scratch?" It is now:

- Is there still a reproducible bug on current `main`?
- If yes, is it about reconnect behavior, delivery timing, duplicate sends, or missing test coverage rather than missing queue support itself?

### Match

The current code paths that seem most relevant are:

- `frontend/src/contexts/conversation-websocket-context.tsx`
  - V1 message sending and queue fallback
- `frontend/src/api/pending-message-service/pending-message-service.api.ts`
  - server-side pending-message API
- `frontend/src/components/features/chat/interactive-chat-box.tsx`
  - startup-state input behavior
- `frontend/src/components/features/chat/chat-interface.tsx`
  - optimistic UI behavior for queued versus immediate sends
- `frontend/__tests__/conversation-websocket-handler.test.tsx`
  - current V1 WebSocket behavior tests

The related open PR `#14692` also suggests the remaining concern may involve reconnect delivery rather than basic first-send queueing.

### Implementation Plan

**Plan:**

1. Confirm with maintainers, issue comments, or PR context whether `#12279` is still an active need on current `main`.
2. If maintainers confirm it is still valid, create a dedicated branch such as `fix-issue-12279`.
3. Reproduce the behavior against the current codebase using a precise startup or reconnect scenario, not only the original January issue description.
4. If a bug still exists, identify which of these is actually failing:
   - queue creation
   - automatic delivery when conversation becomes ready
   - reconnect delivery
   - stale queue clearing on conversation change or stop
5. Add or improve tests to cover the exact failing scenario before changing behavior.

**Likely files to touch if work continues:**

- `frontend/src/contexts/conversation-websocket-context.tsx`
- `frontend/src/api/pending-message-service/pending-message-service.api.ts`
- `frontend/src/components/features/chat/interactive-chat-box.tsx`
- `frontend/src/components/features/chat/chat-interface.tsx`
- `frontend/__tests__/conversation-websocket-handler.test.tsx`

**Likely verification steps if work continues:**

1. Manual startup test: send message before WebSocket `OPEN`.
2. Manual reconnect test: refresh or force reconnect and check pending-message delivery.
3. Automated test for queue fallback result when socket is unavailable.
4. Automated test for eventual delivery without duplicates.
5. Automated test for queue clearing when the conversation changes or is intentionally stopped.

### Review

If I continue on this issue, my self-review checklist will be:

- Does the fix avoid duplicate sends?
- Does it preserve message order?
- Does it avoid stale queued messages leaking into another conversation?
- Does it keep the UI behavior consistent between queued and immediate sends?
- Does the test coverage prove the real bug scenario rather than only the happy path?

### Evaluate

I would consider the issue truly verified and ready for Phase III only if I can show both:

1. a concrete current-branch reproduction scenario, and
2. automated or manual evidence that the chosen fix closes that exact scenario.

At the moment, the stronger conclusion is that this issue may already be partially or fully fixed on `main`, so maintainer confirmation or a pivot may be the safest next step.

---

## Recommendation

My current recommendation is:

1. Do not assume `#12279` is still a clean Phase III target just because the issue is open.
2. First confirm whether maintainers still want follow-up work after commit `238cab4d08ebb176bf972e37ef55fa40615dbf82`.
3. If maintainers confirm the issue is effectively outdated, pivot to a new issue rather than forcing work onto a stale target.
4. If maintainers confirm a remaining reconnect or delivery edge case, continue with a narrower bug statement and create the working branch at that point.

This is the most accurate Phase II conclusion I can give based on the current repository state on June 16, 2026.
