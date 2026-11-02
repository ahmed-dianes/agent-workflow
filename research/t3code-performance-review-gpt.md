# T3 Code performance and regression prevention — round 2

Research date: 2026-10-14. Read `AGENT-WORKFLOW.md` and the previous report first. Source inspection used the existing read-only clone at `614b432a79b0acae38c27c00b1e4eece223ecfe4` (main through #19304); later PR changes were inspected separately through read-only GitHub APIs. No repository scripts, installs, app launches, or GitHub writes were performed. `AGENT-WORKFLOW.md` was not edited. Raw GitHub evidence and method/inventory records are in the evidence directory (not published).

“Verified” below means inspected in source or retrieved from GitHub, not independently executed. PR authors’ benchmark results are labeled as reported. Negative findings are bounded to the inspected tree and searches; private dashboards and out-of-repository automation are unknown.

## 1. The long-lived tracking artifact — found

**[Issue #9661: Track performance audit fixes](https://github.com/pingdotgg/t3code/issues/9661)** is a strong match for the remembered artifact. Created September 4, still open on October 14. It is an audit implementation ledger, not the storage location for automated CI baselines.

Its body separates measured findings from profiling targets, groups work across client/server/web/mobile/marketing/resources, and contains checkboxes plus a `performance-implementation-status` table mapping audit IDs to implementation PRs. It records 40 merged audit PRs, remaining work, native-release holds, rejected/deferred optimizations, contributor credit, and verification limits. A merged PR can explicitly remain incomplete.

Verbatim rule from the issue:

> A target is complete only after a verified improvement, or a written decision that the change is not justified.

The issue is updated through maintainer-account edits and ordinary comments/references. GraphQL edit history identifies `t3dotgg` and `juliusmarminge`, with the latest body edit September 11; comments add evidence on September 14 and October 10. **No repository workflow/script reference to #9661 or its status marker was found.** Consequently, the verified claim is “maintainer-owned edits, not updated by any automation found in this tree”; whether an agent used the maintainer’s account cannot be established. The source policy says to track work in its owning issue/project and update that item when work lands: [AGENTS.md:131–133](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/AGENTS.md#L131-L133).

Useful examples inside the tracker:

- S3 checkpoint-index reuse was deferred because real Git tests found it could miss same-size edits with restored modification times. Correctness proof overruled the performance proposal.
- Mobile changes are held until compatible native fingerprints exist; rebuilding an old fingerprint is explicitly insufficient to exclude old OTA JavaScript.
- [September 14 comment](https://github.com/pingdotgg/t3code/issues/9661#issuecomment-5660853028): a user supplied 33 failed checkpoint operations and roughly 30-second delays on a 574,000-file monorepo.
- [October 10 comment](https://github.com/pingdotgg/t3code/issues/9661#issuecomment-6093571561): a contributor reported roughly 43 MB heap for reconstructed usage records, while explicitly separating measured results from static hypotheses.

Maintainers/agents use the issue to avoid duplicate changes, find existing PRs, assign follow-ups, record release prerequisites, and decide whether an optimization is justified. The issue explicitly instructs reviewing existing related work first. It is not pinned: the pinned-issue query returned #14871, the V1→V2 migration notice. Searches also found [PR #1669](https://github.com/pingdotgg/t3code/pull/1669), “Add perf regression test harnesses,” and its surviving branch `t3code/performance-regression-tests`; it is **closed and unmerged**. Do not count its proposed browser/frame/RPC harness as current main functionality.

A second, automated artifact is the sticky **Thread transfer impact** comment on individual PRs. Its measurements live in Actions artifacts, not in #9661, a baseline branch, commit comments, or a Pages dashboard. The searched refs contained ordinary performance feature branches, but no dedicated baseline/benchmark/gh-pages branch was identified. A private/external dashboard remains possible.

## 2. How the transfer system works end to end

### Workload and measurement boundary

The fixture has **10 historical turns × 5 command tools + one 900,000-byte MCP result per historical turn**, followed by **20 command tools + one 1,100,000-byte MCP result**. Both `codex` and `claudeAgent` are tested. Output is deterministic ASCII diagnostic content with hashed/cursor variation to exercise compression; it contains no user data. [fixtures/transferBudget.ts:2–6,48–74](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/fixtures/transferBudget.ts#L2-L6), [fixture construction:99–171](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/TransferBudgetV2Fixture.integration.ts#L99-L171).

This is synthetic **V2 domain-event** ingestion into the real EventSink/projection/SQLite path. SQLite is in-memory; management reads/streams delegate to the real stores, while unrelated service methods, authentication, and enrichment are substituted. It starts a real localhost HTTP/WebSocket server. It does **not** measure provider adapter ingestion, disk behavior, TLS/relay latency, whole-server RSS, or rendered clients. [integration:72–161](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/transferBudgetV2.integration.test.ts#L72-L161).

The sequence is:

1. Persist historical events.
2. GET the real thread and shell routes, decode with contract schemas, check status/gzip, expected records, absence of synthetic tool output, and matching sequence cursors.
3. Open three sockets: thread only; shell only; second client with both. Resume from the HTTP cursor and require replay synchronization rather than another snapshot.
4. Record counters after synchronization; persist the measured turn; await its terminal sequence on all subscriptions.
5. Verify large tool output is still in durable projection data.
6. Disconnect the second client, open fresh reconnect subscriptions from its pre-turn cursor, measure catch-up, and assert the thread uses snapshot fallback because the raw retained result exceeds replay budget.
7. Write Markdown and JSON **before** asserting no budget violations. [integration:225–355](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/transferBudgetV2.integration.test.ts#L225-L355).

HTTP wire bytes include the response status/headers and compressed body. WebSocket wire counts socket bytes for measured delivery; decoded bytes count received message bodies; messages count received messages. The live interval excludes subscription setup. TCP/IP and TLS overhead and the WebSocket upgrade are excluded. Compression negotiation is asserted. [HTTP measurement:34–85](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/NetworkTransferMeasurement.integration.ts#L34-L85), [WebSocket recorder:123–159](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/NetworkTransferMeasurement.integration.ts#L123-L159), [report description:202–204](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/TransferBudgetReport.integration.ts#L202-L204).

SQL counts are client spans named `sqlite` in this revision, not raw SQL tracing from arbitrary processes. The same counting tracer covers the test runtime and handlers. Older PR #9157 describes the former `sql.execute` span name. [SqlStatementCounter:9–24](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/SqlStatementCounter.integration.ts#L9-L24).

### Exactly what is enforced

The same five hard ceilings apply to both providers:

- Snapshot + thread-only live-turn wire: **7,000 bytes**.
- Thread snapshot HTTP wire: **5,000 bytes**.
- Thread-only live-turn WebSocket wire: **2,000 bytes**.
- Thread-only live-turn decoded bytes: **30,000 bytes**.
- Thread-only live-turn messages: **8**.

They live in [TransferBudgetReport.integration.ts:39–62](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/TransferBudgetReport.integration.ts#L39-L62). The comment states at least 30% headroom over the projected retained-result fixture. Missing provider configuration is a violation. [enforcement:162–195](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/TransferBudgetReport.integration.ts#L162-L195).

**Report-only:** decoded snapshot size, shell snapshot, shell-only traffic, second-client traffic, aggregate server egress, reconnect wire/decoded/messages/mode, and measured-turn/reconnect SQL counts. Scheduler-dependent 50 ms shell coalescing explains deliberately uncapped rows. Importantly, “total thread wire” is not total server egress. [report-only rationale:135–147](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/TransferBudgetReport.integration.ts#L135-L147), [rows:256–289](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/TransferBudgetReport.integration.ts#L256-L289).

Only the compact per-provider snapshot/live-thread measurements and ceilings go into JSON. SQL, shell, second-client, and reconnect numbers are **not** included in the baseline-vs-PR publisher input; they appear in the job Markdown/log. [JSON:77–112](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/integration/TransferBudgetReport.integration.ts#L77-L112).

### CI gate and artifact storage

CI runs on PRs and main pushes with read-only contents permission. Six weighted server shards run the ordinary server suite; exactly one produces the report/result. `always()` summary/upload steps preserve evidence even when the budget assertion fails. Results use the artifact name `thread-transfer-results`, retained **30 days**. A separate `Transfer report artifact` job requires the artifact, so silently missing measurements cannot produce a green aggregate. [ci.yml:261–312](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/workflows/ci.yml#L261-L312).

The aggregate required `Check` depends on server tests and the artifact job, and fails on failures/cancellations/unexpected skips. [ci.yml:483–514](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/workflows/ci.yml#L483-L514). Live ruleset [13856153](https://github.com/pingdotgg/t3code/rules/13856153) requires `Check` (among other checks); strict up-to-date-base enforcement is disabled. Bypass situations are outside this claim.

**Exceeding a ceiling fails the test and ordinary CI.** It is not comment-only. The separate publisher is advisory and cannot turn the measurement failure green. A positive main-relative delta **within** absolute ceilings does not fail this system.

Main produces baselines by running the **same CI test** on a push. Baselines are successful CI-run artifacts, neither committed JSON nor a dedicated ref. Verified example: [main CI run 38536377954](https://github.com/pingdotgg/t3code/actions/runs/38536377954), push at `2d14185f32b6c0141a3f1192caf4a8d181cbe10f`, success; artifact ID 11880965241, 507 compressed archive bytes, created October 14 08:40 UTC, expiry November 13. This establishes that the configured baseline mechanism actually ran.

### Trusted publisher and baseline selection

[thread-transfer-report.yml:3–31](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/workflows/thread-transfer-report.yml#L3-L31) receives completed CI runs through `workflow_run`. Its write-capable job runs only for PR source runs. It checks out **default-branch publisher code**, tests that code, then downloads artifacts as data; it does not execute PR code. Publisher permissions are actions read, contents read, PR write.

The resolver identifies the PR from the source run, or finds exactly one matching open PR associated with its SHA/ref. It rejects stale head results. For the baseline, it lists successful **push** runs of the same CI workflow on the PR’s base branch, prioritizes exact `pull.base.sha`, then checks at most 20 candidates for a nonexpired result artifact. If exact-base data is unavailable, it uses a recent successful base-branch measurement and says so. [resolver:237–316](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/scripts/thread-transfer-report.cjs#L237-L316).

The validator requires a regular file no larger than 64 KiB, fixed schema version, exact allowed fields/providers/metrics, known V1/V2 scenario IDs, and nonnegative safe integers up to one billion. It renders known numeric fields rather than artifact-supplied Markdown or code. [validator:36–96](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/scripts/thread-transfer-report.cjs#L36-L96).

Scenario IDs **and all workload dimensions** must match for deltas/ceiling-change comparisons. Different scenarios show a warning instead. Missing baseline shows no invented comparison; missing PR result posts a warning and preserves an existing successful same-SHA result. Ceiling changes show an explicit warning, but are not independently forbidden. [render:139–220](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/scripts/thread-transfer-report.cjs#L139-L220), [missing results:368–408](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/scripts/thread-transfer-report.cjs#L368-L408).

The publisher updates one `github-actions[bot]` comment bearing `<!-- t3-thread-transfer-report -->`, and rechecks head SHA immediately before publishing. Tests cover fixed-schema validation, ceiling/delta output, ambiguous PR resolution, stale heads, same-SHA missing-result preservation, and V1/V2 incompatibility. [upsert:319–365](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/scripts/thread-transfer-report.cjs#L319-L365), [publisher tests](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/scripts/thread-transfer-report.test.cjs#L44-L309).

**Budget updates:** ceilings are ordinary versioned source constants changed through PRs. No automatic ratchet/reset or dedicated performance approval mechanism was found. New main runs refresh observed baselines, not thresholds. [Original #5350](https://github.com/pingdotgg/t3code/pull/5350) introduced 15,500 / 7,500 / 8,000 / 68,000 / 21 ceilings; current V2 uses the tighter values above. Raising ceilings can make a larger result pass; the publisher exposes comparable changes and AGENTS instructs obtaining human sign-off when rules conflict. That is review policy, not a machine-enforced ceiling-raise veto.

## 3. Other prevention and diagnostic mechanisms

### Structural performance tests, not just stopwatch tests

[performance-regressions.md:3–50](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/docs/internals/performance-regressions.md#L3-L50) defines `vp run test:perf:v2-wire`; [package.json:35](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/package.json#L35) expands it into wire projection, paging/stream/shell, command policy, HTTP/contracts, and shared client reducer/history tests. It warns that older V1 built-app fixture timings can miss the active V2 path. Those older built-app harness files are not present in the inspected tracked main tree; the historical unmerged #1669 is not proof they are shipped.

- **Bounded wire projection:** a 600-row, 8,192-output-bytes-per-row fixture preserves an exact historical full snapshot of 10,375,121 RPC JSON bytes. Current bounded/projected snapshot must be ≤131,072 bytes and ≤2% of full; projected command event ≤1,024 bytes and ≤10% of its former size. It asserts output omission does not mutate the persisted source and even small dynamic-tool output is omitted. [ThreadTransportPerformance.test.ts:23–42,157–252](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/ThreadTransportPerformance.test.ts#L157-L252).
- **Cheap routine sends:** implicit-auto command dispatch must avoid a full projection read, fit ≤1,024 pre-compression RPC JSON bytes, and be ≤0.00006 of the historical 10,372,184-byte send. [commands.performance.test.ts:239–282](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/packages/client-runtime/src/operations/commands.performance.test.ts#L239-L282).
- **Production bounding behavior — important doc/code distinction:** the doc describes a 75-row/about-1 MiB cold-open target, but current code explicitly applies item/byte budgets only to histories **without user-turn starts**. Normal conversations preserve complete cohorts: 10 recent user turns, 20 older turns per page, regardless of tool count/payload size. A test retains **1,020 rows** for 10 × 102-row turns. Consequently, neither 75 rows nor 1 MiB is a universal snapshot cap; control/dependency records and the at-least-one-row rule are additional exceptions. Resume separately checks raw persisted JSON ≤1 MiB before decoding, then ≤128 events and ≤1 MiB projected JSON; otherwise snapshot fallback. Tests preserve paging/scroll/history and compatible old-client fallback. See [policy:9–19](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/threadHistoryPaging.ts#L9-L19), [SQL/complete-turn proof:716,860–889](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/ProjectionStore.test.ts#L860-L889). [doc:15–42](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/docs/internals/performance-regressions.md#L15-L42), [ThreadStream.ts:15–25](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/ThreadStream.ts#L15-L25).
- **Slow consumers:** each subscription bounds retained deliveries, including RPC batches waiting for ACK, to 1,000 items / 8 MiB serialized data. Overflow fails with a resumable error and releases resources. Tests cover blocked high-water/replay reads, unacknowledged batches, fast readers, unrelated filtering, and replay/live deduplication. [LiveStreamBudget.ts:17–101](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/LiveStreamBudget.ts#L17-L101), [tests:80–245](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/LiveStreamBudget.test.ts#L80-L245).
- **SQL scaling:** high-water/command replay tests inspect actual `EXPLAIN QUERY PLAN`, require index seeks, and reject multi-index OR/temp sorting, with 25,000 retained events for high-water checks. [OrchestrationEventStore.sequence.test.ts:211–324](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/persistence/OrchestrationEventStore.sequence.test.ts#L211-L324). Recovery tests select unfinished candidates without settled-history reads and include shared sessions, background rosters, archived children, missing/corrupt ancestry. [ProjectionRecovery.test.ts:264–756](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/ProjectionRecovery.test.ts#L264-L756). Rebuild reads bounded event pages through a fixed sequence inside its transaction. [ProjectionMaintenance.ts:122–164](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/ProjectionMaintenance.ts#L122-L164).
- **Startup/restart benchmark:** default 500 settled/20 active threads enforces continuation counts and idempotency while logging selection/shutdown/recovery timings. `T3_BENCH_RECOVERY=1` enables 5,000/50,000 settled × 10/100/1,000 active matrix. No latency ceiling is asserted; the large matrix is skipped in ordinary CI. [ProviderRuntimeRecoveryPerformance.test.ts:403–509](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/ProviderRuntimeRecoveryPerformance.test.ts#L403-L509).
- **Memory reachability gates:** isolated Node subprocesses with explicit GC assert initial RPC prefix/snapshot/replay history becomes unreachable while subscriptions remain open; startup histories release across regular/handoff/compaction/failure runs; traced results release while cache fibers survive; detached Git lookup callers release on success/failure. These are real retention assertions, not whole-app RSS ceilings. [rpcInitialItems.memory.test.ts:9–25](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/rpcInitialItems.memory.test.ts#L9-L25), [ProviderTurnStartService.memory.test.ts:9–23](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/ProviderTurnStartService.memory.test.ts#L9-L23), [observability.memory.test.ts:9–18](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/packages/shared/src/observability.memory.test.ts#L9-L18), [detachStackFrame.memory.test.ts:7–21](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/git/detachStackFrame.memory.test.ts#L7-L21).
- **Native telemetry history:** age 1 hour, count 3,600 snapshots, 20,000 retained process entries, 64 MiB **estimated** retained bytes; bounded process strings too. Rust tests cover eviction, external-process accounting, backward clocks and UTF-8 truncation; Rust tests are in required aggregate CI. [native main:12–23](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/native/resource-monitor/src/main.rs#L12-L23), [tests:1124–1318](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/native/resource-monitor/src/main.rs#L1124-L1318), [CI:316–344](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/workflows/ci.yml#L316-L344).

### Rendering, motion, lint, and microbenchmarks

The tree has microbenchmarks for sorting/selecting 1,000 threads, plan derivation and usage formatting; remote benchmarks cover in-memory HTTP processing and 200 updates with 100/1,000 loaded messages. They have no pass/fail performance thresholds. Web/client-runtime ordinary test configs include `.test` files, not these `.bench` files; no dedicated benchmark CI invocation was found. [web benchmark:47–80](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/web/src/performance.bench.ts#L47-L80), [remote benchmark:88–134](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/packages/client-runtime/src/remotePerformance.bench.ts#L88-L134), [web config:77–89](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/web/vite.config.ts#L77-L89), [client-runtime config:4–9](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/packages/client-runtime/vite.config.ts#L4-L9).

Lists/diffs use LegendList and Pierre virtualizers. Expanded tool groups have finite draw distance; unchanged item identity lets list/React memo boundaries skip work. This is implementation plus focused tests, not a universal lint rule banning unvirtualized lists. [MessagesTimeline.tsx:3732–3742](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/web/src/components/chat/MessagesTimeline.tsx#L3732-L3742), [identity note:4968](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/web/src/components/chat/MessagesTimeline.tsx#L4968). A strong 128-row palette test asserts zero queries initially and eight when eight rows become visible, while retaining cached badges. [ThreadStatusIndicators.subscriptions.test.tsx:90–176](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/web/src/components/ThreadStatusIndicators.subscriptions.test.tsx#L90-L176).

A React `Profiler` test verifies preview loading does not cause extra renders as one second elapses. [PreviewView.test.tsx:403–423](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/web/src/components/preview/PreviewView.test.tsx#L403-L423). Mobile review instrumentation emits dev-only performance marks, durations and event logs; no frame/latency assertion. [reviewPerf.ts:17–55](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/mobile/src/features/review/reviewPerf.ts#L17-L55). No global React profiling/React Scan CI budget was found.

Motion uses a shared IntersectionObserver/document-visibility/reduced-motion mechanism; it pauses offscreen/background/reduced-motion content and releases listeners after the final reference. Tests cover all those states and late callbacks. Some live CSS animations remain infinite but use stepped timing and visibility gates. Therefore the agent’s “no continuously repainting animations” prose is not a literal ban on every repeated animation, nor enforced by a dedicated animation lint rule. [visibleAnimation.ts:10–62](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/web/src/lib/visibleAnimation.ts#L10-L62), [tests:61–153](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/web/src/lib/visibleAnimation.test.ts#L61-L153), [CSS:545–617](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/web/src/index.css#L545-L617).

Lint includes perf-category warnings, a custom **warning** against repeatedly compiling static Schema encoders/decoders inside hot functions, and an **error** restriction requiring shared `StyledDiffCodeView` virtualized geometry. The schema rule catches specific immediately invoked calls, not every possible compilation pattern. No enforced CSS-animation or universal virtualization lint rule was found. [vite.config.ts:12–15,131–167](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/vite.config.ts#L131-L167), [rule:136–150](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/oxlint-plugin-t3code/rules/no-inline-schema-compile.ts#L136-L150).

No general bundle-size gate, browser frame/FPS CI ceiling, end-to-end startup-time benchmark gate, or process CPU/RSS pass/fail budget was found. Build/preload verification and packaged smoke readiness timeouts catch correctness/liveness failures; timeouts are not evidence of a calibrated performance SLA. [CI build:126–130](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/workflows/ci.yml#L126-L130), [CLI smoke:153–186](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/scripts/smoke-cli-archive.ts#L153-L186).

### Behavioral regression prevention and post-release observation

Recorded provider transcripts replay through real adapter/harness implementations for Claude, Codex, Cursor, ACP registry, Grok, OpenCode variants, Muse and Pi; workspace materialization and deterministic IDs/clocks make orchestration output comparable. Scenario catalogs include plans/tools, steering/restart, requests, subagents, background completion/interruption and recovery. Replay gates wait for named milestones and advance known debounce clocks rather than depending on arbitrary wall-clock sleeps. [OrchestratorReplayFixtures.integration.test.ts:12–47,99–119](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/testkit/OrchestratorReplayFixtures.integration.test.ts#L12-L47), [fixture catalog](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/orchestration-v2/testkit/fixtures/index.ts), [replayGate.ts:3–17,29–118](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/packages/provider-testing/src/replayGate.ts#L3-L17).

The agent policy requires focused backend behavior tests and explicit side-effect drains/persisted milestones. CI independently runs full test/type/lint/build graphs, mobile native static analysis where relevant, Rust and release smoke. Duration-weighted sharding is a speed/isolation mechanism, not a product performance gate: longest tests go to the lightest shard; 0.25-second file overhead; weights manually refreshed by a full-suite recorder when imbalance appears. [AGENTS.md:96–103](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/AGENTS.md#L96-L103), [sequencer:9–40](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/testUtils/weightedShardSequencer.ts#L9-L40), [weight recorder:3–7,30–54](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/scripts/update-test-shard-weights.ts#L3-L7).

A manual **live background verifier** uses fresh real server homes, real credentials and model usage, authenticated HTTP/typed WS, no seeded projection rows or substituted adapters. It requires the agent to retrieve/acknowledge a random gated result, checks roster/run/notification state, then restarts to prove persisted acknowledgement. Missing auth/capability/provider/model obedience is failure, not skip. A fail-gate negative control must fail; repetitions preserve every failed attempt and detailed local evidence. It is not wired into PR/nightly CI in the inspected workflows. It explicitly excludes UI/mobile/cancel and abrupt crash windows. [background-verification.md:3–52](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/docs/operations/background-verification.md#L3-L52).

Nightly releases are scheduled checks/builds/releases, **not an established long-running soak suite**. Release tests repeat ordinary checks with three server shards vs CI’s six; this changes partitioning, not necessarily test coverage. Manual stable releases build the previously published nightly commit. No automated burn-in duration or versioned alert gate was found. [release.yml:9–11,46–50,233–337](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/workflows/release.yml#L46-L50). A disposable relay/Cloudflare canary is documented, but its doc explicitly says it has not been run against a real Cloudflare account. [release.md:243–266](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/docs/operations/release.md#L243-L266). Windows tests are a manual lane whose header says the suite is not yet passing and should be folded into CI once green. [windows-tests.yml:1–12](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/.github/workflows/windows-tests.yml#L1-L12).

Runtime diagnosis is substantial:

- Always-on rotated local NDJSON spans, stdout logs, optional OTLP traces/metrics/logs; `t3 trace summary` gives counts/rates/latency percentiles even when server stalled/stopped. Metrics are not persisted locally. [observability.md:5–13,65–88](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/docs/operations/observability.md#L65-L88).
- RPC/provider-start/git durations, outbox queue wait and outcome counters, webhook delivery/held-delay/run metrics; monotonic timers and interruption finalizers. Provider start duration is not full turn runtime. [Metrics.ts:10–79,146–153](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/observability/Metrics.ts#L10-L79).
- Event-loop monitor samples every 30 seconds, warns for >2-second qualifying stalls with CPU/page-fault/context-switch/RSS context; startup sample skipped and sleep/threshold limitations documented. It cannot detect every janky UI frame or startup regression. [EventLoopMonitor.ts:77–130](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/observability/EventLoopMonitor.ts#L77-L130), [doc:90–120](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/docs/operations/observability.md#L90-L120).
- Rust sidecar CPU/RSS/virtual memory/I/O/process history plus Electron power/process metrics; live snapshots streamed only with diagnostics subscribers, on-demand history, bounded retention. WSL backend telemetry unavailable with Windows-only collector packaging. [resource-telemetry.md:3–27,47–50](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/docs/internals/resource-telemetry.md#L3-L27).
- SIGUSR2 heap snapshots are manual, pausing and sensitive; not CI. [observability.md:658–693](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/docs/operations/observability.md#L658-L693).
- Shared relay/mobile/client Axiom trace dataset and recent-spans view; responder queries preferred over log tailing. [relay-observability.md:5–15,46–53](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/docs/operations/relay-observability.md#L5-L15).
- Best-effort PostHog product events include version, platform, turn duration/result and main-agent token totals, with opt-out and no prompt/content transmission. Web trace resources also identify version. This enables release comparisons **by inference**; no checked-in automatic regression detector/rollback tied to those events was found. [AnalyticsService.ts:179–199](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/server/src/telemetry/AnalyticsService.ts#L179-L199), [telemetry guide:3–11](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/docs/user/telemetry.md#L3-L11), [web trace resource:17–24](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/apps/web/src/observability/clientTracing.ts#L17-L24).

## 4. Real examples and limits of their proof

### Actual automated comment

[#19324 comment](https://github.com/pingdotgg/t3code/pull/19324#issuecomment-6164419431), retrieved with `gh`, says:

> ✅ Thread transfer remains within every enforced ceiling.

One exact row:

```text
| Claude | Live turn messages | 1 | 2 | +1 (+100.0%) | 8 | ✅ |
```

The same comment links [main baseline 38536377954](https://github.com/pingdotgg/t3code/actions/runs/38536377954) and [PR result 38536433786](https://github.com/pingdotgg/t3code/actions/runs/38536433786), source CI success. Thus a 100% message-count increase can legitimately remain green under an absolute budget. This proves publication and comparison work; it does not prove that the transfer fixture measures the SQLite checkpoint optimization. That PR’s other evidence handles that concern.

### Checks that mattered to implementation

- **[#19317: Codex plan coalescing](https://github.com/pingdotgg/t3code/pull/19317)** reports recorded-fixture events **782→29**, bytes **2,110,566→70,780**. A timed 100-delta/20ms test reports plan updates **101→5**, with full final text and terminal streaming state preserved. Author explicitly reports the new test fails on main and 311 tests pass; live Codex plan mode was not verified. Read-only PR patch confirms a **2–7 plan-update bound**, not an exact five-update timing expectation. Quote: “It also checks that the final plan has the full text”. The transfer comment stayed green because synthetic domain-event injection does not measure this adapter ingestion amplification.
- **[#19226: SQLite read worker](https://github.com/pingdotgg/t3code/pull/19226)** reports a broader all-SQL worker proposal broke about **18 server tests** and changed user/queued-message ordering, so the accepted implementation offloads marked reads only. Reported A/B on a 3.6 GB copied database: max event-loop gap **54–67→12–18ms**, writes during reads **2→89–93**, with 360 focused tests plus 146 replay integration cases passing. Read-only source confirms reader crash/restart, read-only enforcement, nested transactions, fallback and cancellation tests. [nodeSqliteClient.test.ts:335–677](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/packages/shared/src/nodeSqliteClient.test.ts#L335-L677). This is behavioral regression checking shaping a perf design.
- **[#19305: native SQLite retention](https://github.com/pingdotgg/t3code/pull/19305)** reports cached large bindings held **+34.8 MiB RSS**, throwaway statements **+2.0–2.4 MiB**, 25 tests, and regressions failing when main/worker fix is restored. Tests cover large worker parameters as well as behavior. These are author-reported process measurements, not a CI RSS budget.
- **[#18077: panel rendering](https://github.com/pingdotgg/t3code/pull/18077)** reports a production build, 1,009-row thread, 54 gestures per variant, long animation frames **90→54**, total blocking time **3,332→1,114ms**. Remaining forced layout is explicitly outside the fix. This is manual profiling evidence, not an ongoing frame gate.
- **[#18191: stepped shimmer](https://github.com/pingdotgg/t3code/pull/18191)** reports real-app frames **44.7→12.7/s**, app CPU **8.1→4.3%**, GPU CPU **4.5→2.2%**, with recording and single-Mac/dev-build limits. Existing infinite motion is optimized, not prohibited outright.
- **[#18825: history replay](https://github.com/pingdotgg/t3code/pull/18825)** reports 3,600 churn samples **1,170.1→39.3ms**, output equivalence in seven cases, and 28 behavior tests. The new test protects deltas/absent-returning processes; benchmark timings are reported, not installed as thresholds.
- **[#12492: report regression](https://github.com/pingdotgg/t3code/pull/12492)** fixes the trusted publisher rejecting the V2 scenario and adds incompatible-baseline/unknown-scenario tests. Seven tests reportedly pass. This illustrates reporting machinery itself needing regression tests.
- **[#19298: bug reached main](https://github.com/pingdotgg/t3code/pull/19298)** states main typecheck/startup broke because UUID generation was treated as an Effect instead of called. Fix reports typecheck and all 11 runtime-state tests passing. It changes one line, adds no new test. Its baseline comment honestly warns exact-base artifact unavailable. Do not generalize that every escaped bug receives new coverage or infer the merge cause.

No sampled real comment was found showing exceeded transfer ceilings. The **failure path is source-verified**, and actual successful reports/artifacts are live-verified; an actual red budget run was not established.

## 5. Agent rules — verbatim

[AGENTS.md:17](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/AGENTS.md#L17):

> Lots of apps have gotten bogged down with bad tech decisions and "slop". We have not, and we're proud of the performance of T3 Code. We regularly audit for performance regressions, often caused by sending too much data over websockets, css animations causing gpu spikes, lists being hard to render, and more. Make sure all changes are considerate of performance impact.

[AGENTS.md:158](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/AGENTS.md#L158):

> Our users drive agents all day and notice a dropped frame, a lying spinner, and a stale label. No continuously repainting animations; they peg the GPU on high-refresh displays.

[AGENTS.md:101–102](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/AGENTS.md#L101-L102):

> Backend behavior changes ship with focused tests for that behavior.
>
> The server is event-sourced, and side effects run after the command commits. In tests, drain the effect worker (`OrchestrationEffectWorkerV2.drain`) or await the specific persisted event or `Deferred` that marks the milestone. Never wait on sleeps or polling. A test that needs a timeout to pass is wrong.

[AGENTS.md:112](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/AGENTS.md#L112):

> UI changes need before/after images. Motion or timing needs a short video.

[performance-regressions.md:44–50](https://github.com/pingdotgg/t3code/blob/614b432a79b0acae38c27c00b1e4eece223ecfe4/docs/internals/performance-regressions.md#L44-L50) additionally directs affected projection/paging/shell/thread changes to run the focused v2 command, scoped typechecks and affected real clients. Verbatim final sentence:

> Payload budgets belong in these tests so regressions fail locally.

AGENTS permits developer preferences to override defaults and requires human sign-off for rule conflicts; browser verification requires user authorization. This research only reads those instructions as evidence and does not run their procedures.

## 6. Assessment

**Excellent:** realistic data pressure without user content; real production serialization/compression boundary; deterministic count/byte/query-plan/retention assertions; behavior preserved alongside savings; required artifact-presence gate; trusted fork-safe publishing; explicit scenario incompatibility and budget changes; reproducible provider replay; published measurements acknowledging uncertainty; an audit ledger that can reject unsafe optimizations.

**Weak/report-only:** main-relative changes are advisory, ceiling changes can be self-approved by PR tests unless review objects, some meaningful metrics never enter baseline JSON, and 30-day artifacts are not a long-term trend store. The fixture covers two provider labels but bypasses their adapters and disk; it cannot catch plan-ingestion amplification or slow-disk stalls. The perf doc also understates current complete-turn paging exceptions; the structural fixtures without user-turn anchors cannot establish a universal row/byte cap. Recovery timing/microbenchmarks are not latency gates; no general UI frame/bundle/startup/RSS budget or automated soak was found. Runtime observability is diagnostic capacity, not verified alerting/rollback. Source/test instructions do not guarantee compliance: escaped-main repair #19298 is a counterexample, and strict base freshness is disabled. Tracker body was last edited September 11 despite later references; human-owned status can lag.

Do not copy every layer. Copy a **small required structural budget**, proof that catches the old bug, versioned baseline evidence when useful, and the distinction between an enforced gate, an advisory metric, and a hypothesis.

## 7. Universal extraction — proposed text only

### A. Part 1: add Step 6a — Performance and regression proof

```markdown
### Step 6a — Performance and regression proof

Minimal (target: one hour; reuse this repo's tools):
1. Choose one important path and a representative synthetic fixture, including
   retained/large input and an idle, repeated, or reconnect case where relevant.
   Use the active production boundary. Record what the fixture excludes.
2. Add `test:perf` to the command contract. Prefer one stable structural budget:
   bytes, requests, queries, retained entries, work per item, or mounted rows.
   Name the metric/unit, fixture version, ceiling, and why that ceiling is useful.
   Do not use a random wall-clock timeout as a performance claim.
3. Make the check exit nonzero on budget violations. Keep the budget in source.
   Prove the detector once with a known bad version or deliberately tightened cap.
   Add the job to `ci-ok`'s `needs`; missing measurements must fail.
4. For bug fixes, add a focused test that detects the original failure, or cite
   existing coverage that does. Record any proof you could not obtain.
5. Never silently raise a ceiling or shrink/change the fixture to get green.
   A proposed budget/fixture change needs measured rationale and human review.
   Main baseline refreshes update observations, not allowed ceilings.

Scale-up (only when justified):
- Preserve versioned measurements from successful main CI. Publish one advisory
  baseline-vs-PR comment with revision, workload, units and ceiling changes.
  Incompatible/missing baselines show 'not comparable'/'unavailable'.
- Use repeated timing measurements on comparable machines; record median/tails,
  sample count and noise. Gate timings only after calibrating a useful tolerance.
- Profile real affected clients for hot rendering/streaming/idle work. Pause or
  release hidden work; bound lists, queues, caches and history by suitable units.
- Keep one owning audit issue for multi-PR performance work: evidence vs hypothesis,
  PR links, unresolved work, owner and release prerequisites. Close a target only
  after verified improvement or a written decision to defer it.
- Add replay fixtures at external boundaries, a bounded manual/nightly soak for
  a known longevity risk, and version-tagged runtime latency/error/resource signals.
```

Minimal can be one new test and one CI step, rather than a new observability platform, bespoke provider simulator, performance reviewer, or dashboard. If the project has no substantial history/UI, choose an applicable equivalent rather than adding an irrelevant reconnect scenario.

### B. Part 2: add these AGENTS.md lines

```markdown
## Performance and regression proof
- Performance check: `<test:perf command>`; fixtures and budgets: `<path>`.
- Hot-path changes run that focused check. Measure the active production boundary
  with representative data and state what the measurement excludes.
- Fixes need a test that detects the original failure, or evidence that existing
  coverage does. Preserve correctness while reducing cost.
- Bound growing work: payloads, queries, queues, caches, history and rendered rows.
  Release hidden/unused subscriptions and background work. UI motion respects
  visibility and reduced motion; idle work should have a measured reason.
- Never weaken a budget or fixture to get green. Propose changes with measurements
  and obtain human review. A refreshed baseline is not permission to spend more.
- Report observed results, units, fixture/revision, and limits. Distinguish required
  gates, advisory measurements and unverified hypotheses.
```

Keep this short. Add the audit-issue link only when one actually exists; keep its implementation checklist on GitHub, not duplicated in AGENTS.md.

### C. Part 3: minimal required CI job

Add this job to the existing `CI` workflow; add `perf` to `ci-ok.needs`.

```yaml
  perf:
    name: Performance budget
    runs-on: ubuntu-latest
    timeout-minutes: 10
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false
      # ADAPT: pinned toolchain and frozen lockfile setup, as in other CI jobs.
      - run: <SETUP_COMMAND>
      - name: Measure and enforce
        env:
          PERF_RESULT_PATH: ${{ runner.temp }}/perf-result.json
        run: <PERF_CHECK_COMMAND>
      - name: Require measurement
        if: always()
        env:
          PERF_RESULT_PATH: ${{ runner.temp }}/perf-result.json
        run: test -s "$PERF_RESULT_PATH"
      - name: Keep evidence
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: perf-results
          path: ${{ runner.temp }}/perf-result.json
          if-no-files-found: error
          retention-days: 30
```

The adapted command must write its result before returning nonzero for violations, and fail if measurement cannot complete. It may be an existing focused test command wrapped with a tiny reporter. It must verify observations and budgets, not merely print a benchmark. Example result contract, adapted to one stable metric:

```json
{
  "schemaVersion": 1,
  "scenarioVersion": 1,
  "metrics": { "responseBytes": 42000 },
  "ceilings": { "responseBytes": 65536 }
}
```

Document the fixed fixture and metric/unit next to the test. Do not embed secrets, user content, arbitrary Markdown, URLs or commands in the result. Run this on main and PRs. Fork PRs get no credentials beyond read-only CI access; dependency installation must not receive production secrets.

### Optional scale-up: safe baseline-vs-PR publisher template

This is a workflow skeleton plus an explicit helper contract, **not a claim of a finished publisher implementation**. The helper must be written and reviewed before enabling it. The minimal workflow above is the one-hour adoption path.

```yaml
name: Performance report
on:
  workflow_run:
    workflows: [CI]
    types: [completed]
permissions:
  contents: read
  actions: read
  pull-requests: write
jobs:
  publish:
    if: github.event.workflow_run.event == 'pull_request'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.repository.default_branch }}
          sparse-checkout: .github/perf-publisher
          persist-credentials: false
      # No PR checkout, dependency install, PR scripts, or PR-supplied action.
      - uses: actions/github-script@v7
        with:
          script: |
            const publisher = require('./.github/perf-publisher/publish.cjs');
            await publisher.publish({ github, context, core });
```

Required helper behavior:

1. Validate the source repository/workflow identity, event and run; derive PR/run/SHA from GitHub metadata, never artifact-supplied PR numbers. Resolve exactly one matching open PR, including forks; recheck current head before writing.
2. Download exactly the expected `perf-results` artifact from that source run and a successful **push** run of the same workflow on the base branch. Prefer exact base SHA; explicitly identify any fallback. Do not run artifact code or install its dependencies.
3. Bound archive/download size and entry count; reject symlinks, traversal and unexpected files. Read only one bounded regular `perf-result.json`. Require exact schema, known metric names, finite nonnegative values and compatible scenario version/workload. Reject invalid results; missing results get an explicit warning.
4. Render trusted strings plus validated numbers into one bot-owned sticky comment: baseline/current revision and run links, absolute result, delta, ceiling, source CI conclusion. Show changed ceilings/fixtures prominently and withhold invalid comparisons. Do not turn a source CI failure green.
5. Test malformed artifacts, ambiguous/stale PRs, incompatible scenarios, missing baselines, same-SHA reruns and ceiling changes. Check out trusted publisher code only. Serialize updates per resolved PR; retain the final SHA guard.
6. Keep this publisher advisory. Required budget enforcement remains in `CI`/`ci-ok`. For long-term trends, store **validated observations** in a trusted append-only store after successful main CI; do not let fork artifacts directly modify thresholds, branches, dashboards or required checks.

For noisy latency metrics, a relative gate needs a separately calibrated rule, e.g. relative tolerance plus an absolute floor and minimum sample count. Do not indiscriminately apply a percentage rule to tiny counts: T3's real one-to-two-message result demonstrates the trap.
