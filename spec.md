# Specification

## Requirements

Each requirement names the approved intent item it serves (`intent.md`, I-1 to I-4) and keeps the current constraints in `inputs/current-policy.md`.

1. **R1 (I-1) Release gate:** A patient is notified only after a doctor has reviewed and released a result version; a result version that has not been released produces no notification.
2. **R2 (I-1) Trigger:** The release of a result version is the only event that can start a notification; notifying on upload is rejected and excluded from scope.
3. **R3 (I-1, OPEN-2) Recipient:** The notification goes to the patient only; referring-doctor notifications are not in scope until OPEN-2 is approved.
4. **R4 (I-2) Notification content:** A notification contains only a generic "a result is ready" message and a link to the portal; it never includes result values, test names, diagnoses or any other clinical content.
5. **R5 (I-2) Logging:** Operational logs record only event IDs and delivery states, never patient identifiers or clinical payloads.
6. **R6 (I-3) Portal route:** The link uses the existing route `/portal/results/:resultId` exported as `resultRoute` by `src/portal/result-route.mjs`, which is reused unchanged; a portal redesign is rejected.
7. **R7 (I-4) Idempotency:** The idempotency key is the released result version, so each released result version is delivered at most once, even when a release event or a send is repeated.
8. **R8 (I-4) Provider throughput:** Calls to the provider never exceed 50 requests per second or 20 messages per batch, including retries.
9. **R9 (I-4) Provider failure:** A failed send is retried at most 3 attempts in total (a proposed bound, to be confirmed against the provider contract), every attempt reuses the same idempotency key so a retry never creates a duplicate delivery, and after the last attempt the delivery state becomes `failed` with no further sends.
10. **R10 (OPEN-1) Channel:** The notification flow stays channel-neutral; no delivery channel is chosen until OPEN-1 is approved.

## Design

The design reuses the two existing source files unchanged and adds a small notification path around them. A path marked "new file" does not exist yet; it is proposed here and would be created only in a later implementation task.

- **D1 Release-triggered flow (R1, R2):** A result-released event carrying the result ID and released version is the single entry point; new file `src/service/release-gate.mjs` confirms that a doctor has released that version and drops the event otherwise, before anything is queued.
- **D2 Content and link (R4, R6):** New file `src/service/notify-released.mjs` builds the message from a fixed generic template plus a link made from `resultRoute` in the existing `src/portal/result-route.mjs`; the patient signs in to the portal to see the result.
- **D3 Privacy boundary (R4, R5):** Clinical content stays inside the authenticated portal and out of every notification and every log line. A notification job carries only an event ID, the idempotency key, an opaque recipient reference and the portal link, and the logger accepts only an event ID and a delivery state, so no other field can reach a log.
- **D4 Queue reuse (R1):** Jobs are queued through `enqueueNotification` in the existing `src/service/queue.mjs`, which is reused unchanged; the queue adapter itself gains no new behaviour.
- **D5 Idempotency (R7, R9):** The key is the released result version, written as `<resultId>:<version>`. New file `src/service/delivery-ledger.mjs` records each key with its delivery state (`queued`, `sent` or `failed`); a key that is already `queued` or `sent` is skipped, so a repeated release event or retry never produces a second delivery, and the same key is passed to the provider on every attempt.
- **D6 Throughput (R8):** New file `src/service/provider-throttle.mjs` takes jobs from the queue, groups them into batches of at most 20 messages and spaces provider calls so that no more than 50 requests are made in any 1-second window; retries pass through the same limiter.
- **D7 Bounded retries (R9):** On a provider error or timeout, the throttle retries only the keys not yet recorded as `sent`, with backoff, up to 3 attempts in total; it then marks the remaining keys `failed` and logs only their event IDs and delivery state.
- **D8 Channel-neutral and patient-only (R3, R10):** The message builder assumes no SMS, email or app-push format; a channel is plugged in once OPEN-1 is decided, and the patient is the only recipient while OPEN-2 stays out of scope.
- **D9 Archived proposal rejected:** `inputs/archived-design.md` was never approved and is not used by this design:
  - The archived helper is rejected and not used; it was never implemented, and this design depends only on the existing files and the new files named above.
  - Sending every uploaded result immediately is rejected; R1 and R2 gate notification on release.
  - Putting result details in the message is rejected; R4 keeps clinical content in the portal.
  - Rebuilding or redesigning the portal is rejected; R6 reuses the existing route.

## Open decisions and owners

These decisions stay open: this spec does not make them, and each one keeps the owner named in `intent.md` or is flagged as still needing one.

- **OPEN-1 Channel choice. Owners: product and privacy.** Which delivery channel carries the notification (for example SMS, email or app push) needs approval from both product and privacy. Until they decide, R10 and D8 keep the flow channel-neutral and no channel is built or configured; whichever channel is chosen must still meet R4, R5 and R8.
- **OPEN-2 Referring-doctor notifications. Owners: clinical and privacy.** Whether the referring doctor is also notified needs approval from both clinical and privacy, and it is not currently in scope. Until it is approved, R3 and D8 send to the patient only, and no referring-doctor notification is designed, built or tested.
- **Retry bound. Owner: not named in the approved inputs.** The policy requires bounded retries but gives no number, so the 3 attempts in R9 and D7 are a proposal that nobody has approved. An owner must be assigned to confirm the number against the provider contract before implementation starts.

## Traceability

Each approved intent item is followed through its requirements, the `plan.md` steps that build them and the named tests that check them; the tests are described here and in `plan.md` and do not exist yet. R10 traces to OPEN-1 rather than to an approved item, and it is built in plan step 3 and checked by T9.

| Intent | Requirement | Test |
| --- | --- | --- |
| I-1 Notify the patient only after clinical review and release (release gate) | R1 release gate, R2 release is the only trigger and R3 patient-only recipient; built in plan steps 2, 3 and 5 | T1 `release_gating_blocks_unreleased_version` checks that a result version a doctor has not released is never queued; T2 `release_gating_queues_released_version_once` checks that a released version is queued exactly 1 time with its key; T9 `patient_only_channel_neutral_job` checks that each job has only the patient as recipient and no channel-specific field |
| I-2 Keep clinical content in the authenticated portal, out of notifications and logs | R4 generic message and portal link only and R5 logs hold only event IDs and delivery states; built in plan steps 1, 3 and 5 | T3 `privacy_notification_has_no_clinical_content` checks that no result value, test name or diagnosis appears in a notification job; T4 `privacy_logs_only_event_id_and_state` checks that every log line has only an event ID and a delivery state |
| I-3 Reuse the current portal route | R6 link built from `resultRoute` in the existing `src/portal/result-route.mjs`, reused unchanged; built in plan steps 3 and 5 | T5 `portal_link_uses_existing_result_route` checks that the link is the existing `resultRoute` with only `:resultId` filled in |
| I-4 No duplicate notification for a released result version, and provider failure handled within its contract | R7 the idempotency key is the released result version, R8 at most 50 requests per second and 20 messages per batch, and R9 at most 3 attempts reusing the same key; built in plan steps 1, 3, 4 and 5 | T6 `duplicate_release_event_delivers_once` checks that a repeated release event leads to 1 delivery; T7 `duplicate_retry_after_provider_failure_delivers_once` checks that retries reuse the key, deliver once and stop after 3 attempts; T8 `throughput_boundary_50_per_second_20_per_batch` checks that no batch exceeds 20 messages and no 1-second window exceeds 50 requests, including retries |
