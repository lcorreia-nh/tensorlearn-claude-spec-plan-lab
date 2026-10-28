# Implementation plan

## Ordered steps

This plan implements `spec.md` (R1 to R10, D1 to D9) in dependency order, and each step names the tests that must pass before the next step starts. No code or test file is written in this lab and no dependency is installed; the steps describe a later implementation task.

1. **Delivery ledger and safe logger (R5, R7, R9):** Create new file `src/service/delivery-ledger.mjs`, which stores each idempotency key (the released result version, written `<resultId>:<version>`) with its delivery state (`queued`, `sent` or `failed`) and writes the only log lines, each holding just an event ID and a delivery state. It depends on no other file. Tests: T4, T6.
2. **Release gate (R1, R2):** Create new file `src/service/release-gate.mjs`, which passes a result-released event on only when a doctor has reviewed and released that result version, and drops every other event. It depends on no other file. Tests: T1.
3. **Notification builder and enqueue (R3, R4, R6, R7, R10):** Create new file `src/service/notify-released.mjs`, which takes a gated event, skips it if the ledger already holds its key as `queued` or `sent`, builds the generic message with a link from `resultRoute` in the existing `src/portal/result-route.mjs`, and queues the job through `enqueueNotification` in the existing `src/service/queue.mjs`; both existing files are imported and reused unchanged. It depends on steps 1 and 2. Tests: T2, T3, T5, T6, T9.
4. **Provider throttle and bounded retries (R8, R9):** Create new file `src/service/provider-throttle.mjs`, which takes jobs from the queue, sends them in batches of at most 20 messages with no more than 50 requests in any 1-second window, and retries only keys not yet `sent`, reusing the same idempotency key, for up to 3 attempts in total before marking them `failed`. The provider client is passed in as a parameter, so no channel-specific file is proposed while OPEN-1 is open. It depends on step 1 and on the jobs queued in step 3. Tests: T7, T8.
5. **Verify the whole path (all requirements):** Run T1 to T9 together, confirm that `git diff` shows no change to `src/portal/result-route.mjs` or `src/service/queue.mjs`, and confirm that no file imports the archived helper, which is rejected and not used. It depends on steps 1 to 4.

## Paths and dependencies

Existing files, checked with `ls` and reused unchanged:

- `src/portal/result-route.mjs` exports `resultRoute` (`/portal/results/:resultId`); step 3 imports it to build the portal link, and the portal redesign is rejected.
- `src/service/queue.mjs` exports `enqueueNotification(queue, job)`; step 3 calls it to queue each notification job, and step 4 takes those jobs from the same queue.

Proposed files, none of which exist yet:

- New file `src/service/delivery-ledger.mjs` (step 1) has no dependencies and is used by steps 3 and 4.
- New file `src/service/release-gate.mjs` (step 2) has no dependencies and is used by step 3.
- New file `src/service/notify-released.mjs` (step 3) depends on new file `src/service/delivery-ledger.mjs`, new file `src/service/release-gate.mjs`, the existing `src/portal/result-route.mjs` and the existing `src/service/queue.mjs`.
- New file `src/service/provider-throttle.mjs` (step 4) depends on new file `src/service/delivery-ledger.mjs` and on the provider client passed in by its caller.

Steps 1 and 2 have no dependencies and can be built in parallel; step 3 needs steps 1 and 2; step 4 needs step 1 and the jobs from step 3; step 5 needs all of them. The external inputs are the result-released event, which comes from the system where a doctor releases results (outside this repository), and the provider client, which stays a parameter until OPEN-1 picks a channel. No new package is installed.

The archived helper from `inputs/archived-design.md` is rejected and not used: it was never implemented, and nothing in this plan depends on it.

## Risks and verification

The release gate means a patient is notified only after a doctor has reviewed and released the result, and the idempotency key is the released result version, so one released version is never delivered twice. Clinical content stays out of notifications and logs.

- **Gate bypass (R1, R2):** An event could arrive for a result version that a doctor has not released; notifying on upload is rejected, so the gate checks release status itself instead of trusting the event. Verified by T1 and T2.
- **Clinical content leak (R4, R5):** A provider error message, a debug log or a new template field could carry clinical content or patient identifiers; the fixed template and the logger that accepts only an event ID and a delivery state block this. Verified by T3 and T4.
- **Result ID in the link (R4, R6):** The reused route puts the opaque result ID in the link; the link carries no clinical content and the result stays behind portal sign-in, but the OPEN-1 owners (product and privacy) should confirm this when they review the channel. Verified by T3 and T5.
- **Duplicate after an ambiguous failure (R7, R9):** A provider timeout after the provider has already accepted a message could lead to a second send; every attempt reuses the idempotency key and the ledger skips keys already `sent`, and the provider contract must be checked to confirm the provider honours the key. Verified by T6 and T7.
- **Throughput at the boundary (R8):** Off-by-one batching, or retries bursting on top of new sends, could exceed 20 messages per batch or 50 requests per second; retries pass through the same limiter. Verified by T8.
- **Unapproved retry bound (R9):** The 3-attempt bound is a proposal with no named owner (see Open decisions and owners in `spec.md`), so T7 takes the bound as a setting and the number can change without rewriting the test.
- **Changes to existing source:** `src/portal/result-route.mjs` and `src/service/queue.mjs` must stay unchanged; step 5 checks `git diff` on both.
- **Archived advice returning:** The archived helper, notifying on upload, putting clinical content in the message and rebuilding the portal are all rejected; step 5 confirms that no file imports the archived helper, and T1, T3 and T5 cover the other three.

| Requirement | Step | Existing path | Test |
| --- | --- | --- | --- |
| R1, R2 (I-1) release gate | Step 2 | - | T1 `release_gating_blocks_unreleased_version`: an event for a result version that a doctor has not released produces no `enqueueNotification` call and no ledger entry. |
| R1, R7 (I-1, I-4) gated enqueue | Step 3 | src/service/queue.mjs | T2 `release_gating_queues_released_version_once`: a released result version produces exactly 1 `enqueueNotification` call carrying the key `<resultId>:<version>`. |
| R4 (I-2) notification content | Step 3 | src/portal/result-route.mjs | T3 `privacy_notification_has_no_clinical_content`: the job holds only an event ID, the idempotency key, an opaque recipient reference and the portal link, and no result value, test name or diagnosis from the input appears anywhere in it. |
| R5 (I-2) logging | Step 1 | - | T4 `privacy_logs_only_event_id_and_state`: every log line written during send, retry and failure has exactly 2 fields, event ID and delivery state, with no patient identifier or clinical payload. |
| R6 (I-3) portal route | Step 3 | src/portal/result-route.mjs | T5 `portal_link_uses_existing_result_route`: the link is `resultRoute` from the existing file with only `:resultId` filled in. |
| R7 (I-4) idempotency | Steps 1 and 3 | - | T6 `duplicate_release_event_delivers_once`: the same released result version arriving twice leads to 1 provider delivery and a ledger state of `sent`. |
| R9 (I-4) bounded retries | Step 4 | - | T7 `duplicate_retry_after_provider_failure_delivers_once`: a provider that fails and then succeeds, and one that times out after accepting, each lead to 1 delivery with the same key on every attempt; a provider that always fails gets 3 attempts in total, then the key is `failed` and nothing more is sent. |
| R8 (I-4) throughput | Step 4 | - | T8 `throughput_boundary_50_per_second_20_per_batch`: 20 messages go in 1 batch and 21 go in 2; 1020 queued jobs make 51 batches, 50 requests go in the first 1-second window and the 51st waits for the next, and retries never push a window above 50. |
| R3, R10 (I-1, OPEN-1, OPEN-2) recipient and channel | Step 3 | - | T9 `patient_only_channel_neutral_job`: each job has exactly 1 recipient, the patient, and no field tied to SMS, email or app push. |
