# Contribution 2: `Add message queue support for V1 conversations during WebSocket connection`

**Contribution Number:** 2  
**Student:** Sreytouch Lang (Jessica)  
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

### Setup Approach Used

I used the repository `README.md` and `CONTRIBUTING.md` as my setup baseline, then intentionally started with code tracing and git-history verification before doing a full runtime boot. That was the right Phase II path here because the issue description was from January 2026, while the current `main` branch had already moved substantially by June 2026.

### Real Challenges Encountered and How I Resolved Them

1. **The issue text was older than the current codebase.**  
   When I compared the January 6, 2026 issue text with the June 9, 2026 `main` branch, the behavior no longer matched cleanly. I resolved that by checking `git log` on the V1 message path and identifying the later March 16, 2026 fix commit `238cab4d08ebb176bf972e37ef55fa40615dbf82` before I claimed a reproduction that might no longer be true.

2. **The project setup is heavyweight, so I needed to avoid a misleading runtime claim.**  
   OpenHands uses Docker, Python 3.12, Node.js 22+, and Poetry 1.8+, which means a full local boot is more expensive than reading one frontend file in isolation. I resolved that by first validating the issue against the current source, setup docs, and existing test file so I could separate "setup work" from "honest reproduction evidence."

3. **The likely remaining bug surface had shifted from initial queue creation to lifecycle edge cases.**  
   The current code already showed queue fallback behavior, so the hard part was narrowing what still needed proof. I resolved that by reframing Phase II around startup/reconnect delivery, duplicate-send prevention, and stale-queue cleanup instead of pretending the original feature was entirely missing.

### What I Found Before Full Runtime Reproduction

The current `main` branch no longer matches the original issue description exactly:

- `frontend/src/components/features/chat/interactive-chat-box.tsx` now explicitly allows users to submit messages during `LOADING` state and says those messages will be queued server-side and delivered when the conversation becomes ready.
- `frontend/src/contexts/conversation-websocket-context.tsx` now falls back to `PendingMessageService.queueMessage(...)` whenever the V1 WebSocket is not open.
- `frontend/src/components/features/chat/chat-interface.tsx` now distinguishes between immediately-sent and queued messages and avoids showing optimistic UI for queued messages.
- `frontend/src/api/pending-message-service/pending-message-service.api.ts` documents a dedicated API for storing pending messages until the conversation is ready.

This means the original problem statement from January 6, 2026 appears at least partially addressed in the current codebase.

### Expected Behavior on Current `main`

- A user should be able to submit a message before the V1 WebSocket is fully open.
- That message should be queued rather than lost.
- The queued message should be delivered automatically once the conversation becomes ready.
- The queue should not create duplicates during reconnects or leak stale messages across conversation changes.

### Actual Behavior / Current Observation on Current `main`

- From code inspection, the current frontend now appears to accept the startup-state send path and queue through `PendingMessageService` when the V1 socket is unavailable.
- The remaining uncertainty is not basic queue creation; it is whether queued messages are always flushed correctly, without duplication, during reconnect, refresh, or conversation-change edge cases.
- Because I did not yet produce honest end-to-end runtime proof for those edge cases, I am treating them as an open hypothesis rather than overstating the reproduction result.

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

#### Specific Files and Functions Involved

- `frontend/src/contexts/conversation-websocket-context.tsx`
  - `sendMessage(...)` is the core V1 path that now decides between direct send and queue fallback.
- `frontend/src/api/pending-message-service/pending-message-service.api.ts`
  - `PendingMessageService.queueMessage(...)` is the storage path used when the WebSocket is unavailable.
- `frontend/src/components/features/chat/interactive-chat-box.tsx`
  - the startup-state chat input behavior determines whether a user can submit before the socket is ready.
- `frontend/src/components/features/chat/chat-interface.tsx`
  - the queued-versus-immediate message handling affects optimistic UI behavior and duplicate-risk interpretation.
- `frontend/__tests__/conversation-websocket-handler.test.tsx`
  - this is the most relevant existing test surface for proving queue fallback, delivery, and error behavior.

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

### Root Cause Hypothesis

My Phase II root-cause hypothesis is:

- **Original January root cause:** the V1 `sendMessage(...)` path did not yet mirror the queueing contract that V0 already had, so the ability to send depended too directly on WebSocket-open state.
- **Likely remaining current-main root cause:** queue creation now exists, but the full queue lifecycle is split across the provider, the pending-message API, and UI/optimistic-message handling. That means the remaining risk is not "message cannot be queued" but "queued message may not be flushed, cleared, or displayed correctly across reconnect and conversation-transition edges."

That hypothesis points most directly at:

- `frontend/src/contexts/conversation-websocket-context.tsx`
- `frontend/src/api/pending-message-service/pending-message-service.api.ts`
- `frontend/src/components/features/chat/chat-interface.tsx`
- `frontend/__tests__/conversation-websocket-handler.test.tsx`

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

**Why this plan follows from the root cause hypothesis:**

- If the remaining problem is lifecycle coordination rather than queue creation, then reproducing reconnect and cleanup behavior matters more than re-implementing the queue itself.
- If the provider, API, and UI all participate in the message lifecycle, then the first useful fix is likely a focused test that proves where the lifecycle breaks.
- If maintainers consider the issue effectively resolved already, the correct engineering move is to pivot rather than force unnecessary code churn onto a stale target.

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

## Testing Strategy

### Manual Verification to Run Next

1. Start OpenHands locally using the documented setup path.
2. Open a V1 conversation while the runtime is still initializing.
3. Send a message before the socket reaches `OPEN`.
4. Confirm whether the message queues, flushes exactly once, and appears correctly in the UI.
5. Repeat during reconnect and after a conversation change to check stale-queue cleanup.

### Automated Verification to Add Next

1. A test that proves disconnected `sendMessage(...)` falls back to `PendingMessageService.queueMessage(...)`.
2. A test that proves queued messages flush exactly once when the connection becomes ready.
3. A test that proves stale pending messages are cleared on conversation change or intentional stop.
4. A test that proves queueing errors surface clearly instead of silently dropping the message.

---

## Recommendation

My current recommendation is:

1. Do not assume `#12279` is still a clean Phase III target just because the issue is open.
2. First confirm whether maintainers still want follow-up work after commit `238cab4d08ebb176bf972e37ef55fa40615dbf82`.
3. If maintainers confirm the issue is effectively outdated, pivot to a new issue rather than forcing work onto a stale target.
4. If maintainers confirm a remaining reconnect or delivery edge case, continue with a narrower bug statement and create the working branch at that point.

This is the most accurate Phase II conclusion I can give based on the current repository state on June 16, 2026.
