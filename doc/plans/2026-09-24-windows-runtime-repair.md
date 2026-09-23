# Windows runtime repair and source-managed deployment

Date: 2026-09-24. Revision: 1. Status: proposed; independent review pending.

## Outcome and scope

Make Windows execution ownership and pre-provider review waits reliable, then reconcile only proven affected task state. Preserve company boundaries, single-assignee document writes, review/approval gates, budgets, explicit pauses, cancellation evidence, and unknown-action holds.

This document is a repair plan, not a deployment approval or a claim that repairs passed. The source fork and checkout are prepared. The existing deployed package remains unchanged. Application tests, source-to-deployment parity, restore rehearsal, and repair canaries are NOT_TESTED.

## Source and deployment contract

- `origin`: the operator-owned fork. `upstream`: `paperclipai/paperclip`, fetch-only in this checkout. Keep `master` suitable for upstream synchronization. Use small topic branches and reviewed commits for repairs.
- This planning branch starts at upstream `f55759942b8c17b258168848ca769ae99d9243ec`. It is not the production build base.
- Current deployment reports `2026.916.0` with additional local patches. Upstream tag `v2026.916.0` resolves to `dffc2b3ca1b9e88fa21cb17493083e682dffd1ca`. Version text alone does not establish source parity.
- The stable release `v2026.916.1` resolves to `d554c4789ed3930f8a53ac9fdf6503b3187097da`. An update to that release is not a demonstrated fix for these failures. A rolling-master upgrade is outside this repair.
- Before implementation, inventory deployed package versions, immutable package integrity, all patched files and preimages, external dependency patches, launcher behavior, and both manifests. Compare the original release payload with the deployment. Map every difference to a source commit, dependency patch, or explicit retained configuration. Record any unexplained difference as a blocker.
- Preserve the existing Codex quota parser and ACP quota classification, Windows Hermes stdin transport, bounded gateway cancellation, run-scoped gateway authentication/instructions, Windows skill junctions, Codex MCP header compatibility, ACP session-mode behavior, and qualified engine selection. An old disabled quota-clock patch is not an accepted repair.
- Preferred first repair base: the verified `v2026.916.0` source plus individually ported, reviewed existing fixes. If parity cannot be established or a patch needs a newer base, stop and propose a separately qualified upgrade. Do not copy whole patched generated files into a different release.
- Build from a recorded source commit and frozen lockfile into a separate versioned deployment directory. Record Node/package-manager versions, dependency patch hashes, source SHA, build commands, artifact hashes, migration set, and test evidence. Keep runtime data, secrets, prompts, sessions, database dumps, machine paths, and private incident evidence outside the public fork.
- The legacy payload is a rollback artifact, not a source checkout. Creating the fork does not retroactively make that payload reproducible. No new direct edits to the deployed package are planned.

## Evidence and upstream position

The investigation found three distinct failure classes. Database responsiveness and a healthy HTTP endpoint do not establish task progress.

1. A task's child assignee attempted to update its parent's plan document. The document endpoint rejected it with `403: Agent cannot mutate another agent's issue`. Provider success did not mean the document revision was saved. This is an ownership mismatch, not a reason to widen authorization.
2. Legacy runs cancelled with `issue_continuation_waiting_on_review`, null start/PID metadata, and no recorded `adapter.invoke` can become persistent `legacy_execution_requires_reconciliation` holds. A resolved recovery row still blocks when its evidence retains `automaticRecovery.replay=blocked`.
3. A Windows PID was reused by a different process. `Get-Process.StartTime` was unreadable, while CIM supplied a creation time incompatible with the stored provider identity. The continuation path treated uncertain identity as a definitely active provider.

Public references: [issue #13640](https://github.com/paperclipai/paperclip/issues/13640), [PR #13150](https://github.com/paperclipai/paperclip/pull/13150), and [stable release](https://github.com/paperclipai/paperclip/releases/tag/v2026.916.1). At inspection, PR #13150 was open and unmerged at `f2ca15db4cb314e44187e3c01f1d967583910ee1`. It supplies a narrow pending-review reconciliation proposal, not complete prevention or a qualified Windows deployment. Recheck its status before adopting code; preserve attribution and prefer an equivalent merged fix when available. Do not execute database workarounds from issue comments.

## Repair A: Windows process identity

Primary paths: `server/src/services/hot-restart.ts`, `server/src/services/conversation-continuation.ts`, and `server/src/services/execution-recovery-resolution.ts`. The latter's validator separately tests bare PID/group liveness and can reject reconciliation after a continuation check succeeds. Inventory all ownership/stop/reconciliation consumers, not only existing `readProcessStartedAt` callers, and use a consistent evidence contract. Keep destructive stop paths separately guarded.

1. Keep the existing process-start probe. If the Windows result is empty, invalid, or inaccessible, try a read-only `Get-CimInstance Win32_Process` creation-time lookup for the validated positive integer PID. Use hidden, noninteractive processes, fixed arguments, bounded output, and a shared total deadline. No WMI service changes or administrator requirement.
2. Normalize both persisted and observed values to UTC milliseconds, matching database/JavaScript precision. Distinguish `matching`, `mismatched`, `absent`, and `unknown` identity. Access denial, timeout, malformed output, missing expected identity, and conflicting observations are `unknown`, not proof that execution stopped.
3. A proven mismatched PID must not keep the old PID ownership active. A missing or mismatched root does not prove child/group, remote execution, controller lease, or environment cleanup is settled. Check the existing independent ownership gates. Never signal a reused PID or an unproven process group.
4. Preserve the blocker when ownership remains unknown, but describe the uncertainty accurately. A live matching owner stays blocked. Keep company scope and current pause/review/budget gates. Revalidate ownership under the existing task/dispatch locking boundary before a successor can start; identity observations are not transferable authorization to kill a process later.
5. Keep non-Windows behavior unchanged. Bound aggregate probe work across multiple candidate runs so a request cannot stall for an unbounded number of shell probes.

Required checks: matching owner; reused PID; missing PID; access denial in both probes; null/invalid timestamps; milliseconds/microseconds normalization; process exit/reuse between observations; held/failed-cleanup lease; live controller; child/group still active; repeated reads; hidden-window execution on native Windows; no signal sent. Verify all existing hot-restart consumers still fail safely.

## Repair B: pre-provider review cancellation

Primary paths: `server/src/services/legacy-execution-recovery.ts`, `server/src/services/heartbeat.ts`, `server/src/services/execution-recovery-resolution.ts`, `server/src/services/recovery/service.ts`, and `server/src/routes/issues.ts`.

1. Trace the exact cancellation write and dispatch boundary on the selected source base. Add durable, server-produced pre-dispatch evidence at the branch that cancels before any provider invocation. Do not exempt an error-code string alone or trust client-supplied `providerWorkStarted=false`.
2. Under the existing task/run locking and compare-and-swap rules, classify that verified pre-provider review wait as a governed wait/disposition-repair condition. Preserve the real review stage and participant. If there is no current typed wait target, retain the existing bounded disposition-repair policy and its durable attempt ceiling; do not create an endless retry.
3. Update every path that creates or reconstructs a legacy recovery hold, including startup, terminalization, and periodic stranded-work sweeps. Verify classifier ordering so an attempt counter cannot turn proven pre-provider work into an unknown-action hold. Genuine invoked/interrupted work and incomplete evidence remain held.
4. For historical rows, null `startedAt`/PID and absence of an event are corroborating evidence only. Verify the exact pre-dispatch code path, complete event lineage/retention, terminal reason, no controller/remote dispatch/lease activity, and no provider-side effects. Missing evidence remains blocked. Do not infer never-started from a pruned log.
5. Evaluate the pinned upstream reconciliation proposal against this source base. Preserve board-only recovery authority, company checks, reviewer, stage, return owner, and approval gates. Revalidate them in the transaction and again under the enqueue/dispatch lock. Reject stale review stages, unauthorized actors, actual starts, or unknown outcomes.
6. Reconcile every duplicate blocking recovery row for the same verified source run through the supported, audited service/API. Preserve historical evidence. Atomically record the decision before clearing the matching execution blocker. Deliver at most one continuation to the current authorized participant, with durable idempotency across retries and restarts. Do not mark the business task done.

Required checks: exact review-wait error; other queued-cancellation code; provider invoked despite null start; incomplete/pruned events; genuine failure/timeout/cancellation; quota/workspace waits; typed review target present/absent; exhausted disposition budget; pending human approval; actor/company denial; reassignment/stage change during delivery; duplicate recovery rows; concurrent resolvers; crash before/after decision and enqueue; repeated scheduler/startup sweeps. Prove no duplicate provider dispatch and no return of the false hold.

## Task-state correction after qualification

Keep this separate from runtime code repair. Read back the target's current assignee, plan revision, active runs, hold evidence, and explicit owner acceptance requirements. Use the Board-authorized assignment API for the intended writer with deferred wake when supported. Preserve the independent recovery hold until separately reconciled. Do not change global document authorization. Revalidate the revision and writer immediately before the document write; require exact revision/body readback and retain the human plan-acceptance step. A successful provider turn alone never proves success.

Do not broadly release older failed/timed-out runs, sweep deferred wakes, enable timers, switch engines, or dispatch business work as a health test. The earlier model/engine mismatch was already repaired; retain that qualified route and verify its real executable/model during acceptance.

## Qualification, cutover, and rollback

1. Independent Claude Opus 5.5 plan review is mandatory before repair implementation. Save the exact reviewed plan hash, requested and observed model identity, findings, and their disposition. An unavailable requested model or incomplete review does not satisfy this gate; do not silently substitute another model.
2. Use separate repair commits for process identity and review recovery. Add regressions that fail on the selected unpatched base. Run targeted tests first, then repository-required typecheck, test suite, and build before declaring implementation ready. Review the final implementation separately from this plan.
3. Test with synthetic fixtures in an isolated instance with a separate home/config, ports, database, storage and credentials. Disable automatic dispatch and outbound integrations before startup. Never let default development startup discover the production instance. Rehearse backup restoration without provider dispatch; the current backup was decompressed successfully but has not been restore-tested. If a private data clone is needed, keep it local, quarantined and outside Git.
4. Compare schema and migration sets before cutover. Prefer no schema migration for these repairs. If any migration is required, stop for a migration-specific plan and tested rollback; switching an old binary onto a migrated database is not an acceptable rollback assumption.
5. Immediately before live changes, preserve agent/dispatch/timer settings, quiesce through supported controls, and verify zero queued/running runs across all companies plus no active provider/controller ownership. Recheck after quiescence. Take and verify a fresh native database backup; preserve runtime, manifests, configuration, launcher/task definitions, local storage and required secret material privately. Prove the exact listener and process lineage.
6. Stop only the owned scheduled task/process tree. Verify its listener, embedded database and descendants are stopped. Install the separately built candidate and its reviewed manifest; switch the exact pinned entrypoint atomically with a retained prior pointer. Start the existing user-level task and wait for recovery-ready, with a bounded startup deadline.
7. Verify loaded commit/build provenance, hashes, version, process lineage, one listener and database readiness. Run bounded diagnostic tasks through affected real adapters: fresh response, actual model/CLI/engine evidence, positive token usage, authenticated completion callback on the same run before finish, task done, and run succeeded. Cover new/resumed/loaded sessions and cancellation where affected. No business task is a canary.
8. Recheck the proven incident cases, review gates and expected agent settings after at least two observed scheduler/recovery cycles and one controlled restart. Verify no duplicate hold or dispatch. Then apply separately authorized task-state corrections one at a time with readback.
9. If readiness, parity or canaries fail, stop the candidate, preserve failed evidence and revert only the deployment pointer/manifest/config changes to the prior immutable payload. Recheck schema compatibility first. Preserve legitimate task writes; do not restore the whole database merely to undo an application build. Data reconciliation needs a specific evidence-based compensation, or separately authorized restore if corruption requires it. Never auto-resume all business work as part of rollback.

## Longer-term maintenance

Keep a small fork patch set and retire each patch only after an upstream equivalent passes the same regression tests. Prepare public contributions from generic synthetic cases; publish no local issue identifiers or logs. Review upstream updates on demand, build a separate candidate, and admit it through this same qualification process. Do not run production from a mutable checkout or auto-update to master.

Add bounded local operational logging only through reviewed existing configuration: startup/recovery readiness, process-probe result class, blocking reason and database error diagnostics. Set retention and redaction; do not log SQL parameter values, credentials or full prompts, and do not enable a remote telemetry sink as part of this repair. Log rotation and a harmless local failure probe must be verified before relying on the logs. This is a planned improvement, not an active monitor.
