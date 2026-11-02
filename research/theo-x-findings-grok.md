# Theo / T3 Code on X: performance tracking + pipeline (Grok 4.7 high, 2026-10-14)

Verified on GitHub by Claude: PR #5350, PR #9157, PR #2829, issue #9661, and the bot comment on #9157.
X posts below are as reported by Grok (not independently re-fetched).

## The perf/regression tracker (it is three things)
1. PR comment "Thread transfer impact": added in https://github.com/pingdotgg/t3code/pull/5350 (merged 2026-08-07).
   `.github/workflows/thread-transfer-report.yml` runs after CI from trusted main code and updates ONE sticky bot comment:
   main baseline vs this PR vs ceiling, per provider, ✅/❌. Example: https://github.com/pingdotgg/t3code/pull/9157#issuecomment-5503260416
2. More metrics, not more gates: https://github.com/pingdotgg/t3code/pull/9157 added shell/second-client/reconnect/SQL rows with NO caps (too noisy between runs).
3. Audit tracking issue: https://github.com/pingdotgg/t3code/issues/9661 "Track performance audit fixes" (opened 2026-09-04, open).
   Checklist of ~70 audited findings (C/S/W/M/P/R ids). Rules: one concern per PR, focused tests, review before merge;
   "complete only after a verified improvement, or a written decision that the change is not justified".
   Most likely the tracker people remember (in the right month); Grok found no X post linking it.
Also: docs/internals/performance-regressions.md (landed in PR #2829, 2026-10-02) says what the numbers do and do NOT measure.

## X posts (per Grok)
- 2026-08-07 https://x.com/theo/status/2085662598897545625 "I cut the payload size by >85% four times" (ledger image of commits)
- 2026-08-26 https://x.com/theo/status/2092455706582499815 render +93.8%, data -80%; reply names the "Thread transfer impact" CI job; render speed is NOT tracked by it
- 2026-08-29 https://x.com/theo/status/2093491149407494273 first "someone's agent auto-merging an obvious regression to main"
- 2026-08-30 https://x.com/theo/status/2094188698510434738 "My fear of performance regressions was one of the main inspirations for building T3 Code"
- 2026-10-07 https://x.com/theo/status/2107736680413454440 PR watcher rebuilt ~30 times; 75% less rate-limit use (direct API, REST/GQL rotation)
- 2026-10-07 https://x.com/theo/status/2107765823637168287 "My agents wrote over 200 bad 'watch pr' scripts"
- 2026-10-09 https://x.com/theo/status/2108354520082436579 public "T3 Code Annoyances" page for the team
- 2026-10-12 https://x.com/theo/status/2109576403171872912 told the wrong thread "get this merged" -> shipped ts-rust
- 2026-10-12 https://x.com/theo/status/2109724741234028639 model ranking S: Opus 5.5, Muse Wonder; A: GPT-6.1 Sol
- 2026-10-12 https://x.com/theo/status/2109731846842196093 "I often have Opus ask Wonder for code review, copy work, and architecture review"
- Podcast 2026-10-05 https://www.youtube.com/watch?v=2iu5u3sUukc (not watched)
