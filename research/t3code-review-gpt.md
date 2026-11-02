**T3 Code’s strongest transferable idea is a division of responsibility:** agents make focused changes and supply behavioral evidence; executable rules constrain common mistakes; CI independently validates the whole repo; maintainers retain product and merge authority.

It is a fast, pragmatic pipeline, but it is **not** a universally enforced “two reviewers, all bots green, then merge” system.

Reviewed on **14 October 2026** using a fresh depth-200 clone at commit [`acb90279`](https://github.com/pingdotgg/t3code/commit/acb90279e2adad19a63586944543fe7346499467). I read repository configuration and sampled 30 merged PRs, CI runs, releases, and the live main ruleset. **No project scripts, installs, or tests were executed; all GitHub operations were read-only.**

The reusable draft is available as WORKFLOW.md (not published). Research evidence (not published) preserves the public API responses and tracked-file inventory.

## 1. Pipeline map

### Fresh change → isolated workspace

Outside contributors must establish the problem and scope before substantial work. Small obvious bug fixes and narrowly scoped configuration of existing capabilities have exceptions. Features and intentional behavior changes generally require explicit maintainer approval. Eligibility is judged by purpose and behavioral impact, not a file-count threshold. [CONTRIBUTING.md:28–59](https://github.com/pingdotgg/t3code/blob/acb90279/CONTRIBUTING.md#L28-L59)

For worktrees created through T3, `t3.json` automatically invokes `scripts/setup-worktree.ts`. That script:

- Installs dependencies noninteractively.
- Links selected gitignored environment files from the main checkout, preserving existing real files.
- Warms the web dependency cache.

This is app-integrated worktree setup, not a universal Git hook. [t3.json:4–10](https://github.com/pingdotgg/t3code/blob/acb90279/t3.json#L4-L10), [setup-worktree.ts:13–54](https://github.com/pingdotgg/t3code/blob/acb90279/scripts/setup-worktree.ts#L13-L54)

Runtime state is isolated separately: linked worktrees default to their own `.t3/userdata`, ahead of an ambient `T3CODE_HOME`; explicit `--home-dir` wins. Ports derive from worktree identity but shift when occupied, so agents must read actual ports from output. The main checkout uses a separate development home. [development.md:27–36](https://github.com/pingdotgg/t3code/blob/acb90279/docs/operations/development.md#L27-L36), [dev-runner.ts:648–677](https://github.com/pingdotgg/t3code/blob/acb90279/scripts/dev-runner.ts#L648-L677)

**Why this matters for agents:** several tasks can run concurrently without sharing application databases or blindly fighting over fixed ports.

### Repository structure → implementation boundaries

The current stack is **pnpm through Vite+**, with Node 24 and `tsc-rs` typechecking. Bun remains in some recording utilities; it is not the primary workflow. There is no first-party Turbo configuration in the reviewed tracked files. [package.json:5–66](https://github.com/pingdotgg/t3code/blob/acb90279/package.json#L5-L66), [pnpm-workspace.yaml:1–6](https://github.com/pingdotgg/t3code/blob/acb90279/pnpm-workspace.yaml#L1-L6)

The workspace separates:

- `apps/server`: backend, orchestration, persistence, transports.
- `apps/web`: React/Vite client.
- `apps/desktop`: Electron shell around web and server.
- `apps/mobile`: separate React Native client.
- `apps/marketing`: Astro site.
- `packages/contracts`: wire schemas and small derived helpers.
- `packages/client-runtime`: logic shared by web and mobile.
- `packages/shared`: shared runtime utilities.
- Provider and source-control packages: adapter boundaries.
- `infra/relay`: independently deployed cloud infrastructure.

The agent guide explicitly routes changes to these locations and keeps complexity at adapter boundaries, business behavior in services, and transports thin. [AGENTS.md:141–155](https://github.com/pingdotgg/t3code/blob/acb90279/AGENTS.md#L141-L155)

Shared packages expose explicit source subpaths. The build graph declares web → server → desktop dependencies; these build tasks explicitly disable task-result caching. [shared/package.json:5–19](https://github.com/pingdotgg/t3code/blob/acb90279/packages/shared/package.json#L5-L19), [server/vite.config.ts:69–75](https://github.com/pingdotgg/t3code/blob/acb90279/apps/server/vite.config.ts#L69-L75), [desktop/vite.config.ts:24–46](https://github.com/pingdotgg/t3code/blob/acb90279/apps/desktop/vite.config.ts#L24-L46)

The `.repos/` tree contains **vendored read-only upstream references**, particularly Effect and Alchemy. Agents are instructed to consult them, never edit or import them. They are excluded from routine lint, test discovery, formatting, CI checkout, and bot review. Their nested agent files and upstream hooks are not T3’s contributor workflow. [AGENTS.md:148](https://github.com/pingdotgg/t3code/blob/acb90279/AGENTS.md#L148), [reference-repos.ts](https://github.com/pingdotgg/t3code/blob/acb90279/scripts/lib/reference-repos.ts), [vite.config.ts:62–125](https://github.com/pingdotgg/t3code/blob/acb90279/vite.config.ts#L62-L125)

### Local development → focused proof

First-checkout commands are:

```sh
vp i
vp run dev
```

The developer opens the printed pairing URL. Web development uses a single origin with backend proxying, supporting remote browsers without baking localhost into the bundle. [development.md:3–14](https://github.com/pingdotgg/t3code/blob/acb90279/docs/operations/development.md#L3-L14), [development.md:38–57](https://github.com/pingdotgg/t3code/blob/acb90279/docs/operations/development.md#L38-L57)

Before declaring a change done, agents run **scoped** checks:

```sh
vp test run <files>
vp lint <files>
vp run --filter <package> typecheck
```

Full-repo checking belongs to CI unless requested. Backend behavior changes require focused tests; implementation-mirroring tests and arbitrary sleeps are discouraged. [AGENTS.md:96–105](https://github.com/pingdotgg/t3code/blob/acb90279/AGENTS.md#L96-L105), [development.md:99–119](https://github.com/pingdotgg/t3code/blob/acb90279/docs/operations/development.md#L99-L119)

Verification is broader than “the edited component works.” Agents must consider other entrypoints, clients, providers, contracts, agent-facing APIs, connection modes, reverse actions, and documentation. [AGENTS.md:65–76](https://github.com/pingdotgg/t3code/blob/acb90279/AGENTS.md#L65-L76)

### Testing strategy

The tracked first-party tree contains **1,807 files with `.test.` or `.spec.` in their names**. That is an inventory count, not a count of executed tests.

The meaningful layers are:

- **Unit and service tests:** primarily Vite+/Vitest, with Effect test layers and service-boundary testing. Policy says to replace external dependencies rather than mock the logic being verified. [effect-services.md:69–91](https://github.com/pingdotgg/t3code/blob/acb90279/docs/internals/effect-services.md#L69-L91)
- **Integration tests:** real persistence, orchestration, Git/worktrees, HTTP, WebSocket, lifecycle and recovery paths. Server test files run serially within each runner to avoid load-sensitive interference. [server/vite.config.ts:133–144](https://github.com/pingdotgg/t3code/blob/acb90279/apps/server/vite.config.ts#L133-L144)
- **Provider protocol replay:** recorded transcripts run through multiple real adapter/harness implementations, with deterministic runtime support and explicit replay gates. This provides repeatable protocol coverage without requiring live models for every PR. [OrchestratorReplayFixtures.integration.test.ts:12–47](https://github.com/pingdotgg/t3code/blob/acb90279/apps/server/src/orchestration-v2/testkit/OrchestratorReplayFixtures.integration.test.ts#L12-L47), [replayGate.ts](https://github.com/pingdotgg/t3code/blob/acb90279/packages/provider-testing/src/replayGate.ts)
- **Opt-in live-provider tests:** these exist, but require binaries and explicit environment configuration. They should not be mistaken for routinely exercised CI coverage. [OpenCode2OrchestratorV2.live.test.ts:1–18](https://github.com/pingdotgg/t3code/blob/acb90279/apps/server/src/orchestration-v2/OpenCode2OrchestratorV2.live.test.ts#L1-L18), [live-test gate:339](https://github.com/pingdotgg/t3code/blob/acb90279/apps/server/src/orchestration-v2/OpenCode2OrchestratorV2.live.test.ts#L339)
- **Real browser subsystem tests:** Chromium tests exercise locators, interaction and stale-reference behavior. These test the browser machinery, not a comprehensive end-to-end suite of T3 user journeys. [ServerBrowserPage.test.ts:13–35](https://github.com/pingdotgg/t3code/blob/acb90279/apps/server/src/preview/ServerBrowserPage.test.ts#L13-L35)
- **Performance regression tests:** synthetic retained-history fixtures exercise production persistence and transport paths. Enforced ceilings include wire bytes, decoded bytes and message counts. SQL and reconnect measurements also appear in reports, but are not all enforced by those ceilings. [TransferBudgetReport.integration.ts:20–61](https://github.com/pingdotgg/t3code/blob/acb90279/apps/server/integration/TransferBudgetReport.integration.ts#L20-L61), [budget enforcement:162–195](https://github.com/pingdotgg/t3code/blob/acb90279/apps/server/integration/TransferBudgetReport.integration.ts#L162-L195)
- **Release infrastructure tests:** ordinary PR CI tests preview ZIP validation and nightly policy, and exercises version updates, lockfile regeneration and updater-manifest transformations using disposable fixtures. [ci.yml:184–188](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/ci.yml#L184-L188), [release-smoke.ts:211–252](https://github.com/pingdotgg/t3code/blob/acb90279/scripts/release-smoke.ts#L211-L252)

I found no tracked `.snap` files or configured coverage threshold. The current web test project is a unit project; UI proof principally comes from targeted real-client verification and PR evidence. [web/vite.config.ts:77–89](https://github.com/pingdotgg/t3code/blob/acb90279/apps/web/vite.config.ts#L77-L89), [web/vite.config.ts:289–290](https://github.com/pingdotgg/t3code/blob/acb90279/apps/web/vite.config.ts#L289-L290)

### Commit → PR

The pre-commit hook runs `vp staged`. Staged processing **formats only**—no lint or typecheck on commit. The prepare script invokes Vite+ configuration, and editor settings use Oxc formatting and fixes. [pre-commit](https://github.com/pingdotgg/t3code/blob/acb90279/.vite-hooks/pre-commit), [vite.config.ts:80–83](https://github.com/pingdotgg/t3code/blob/acb90279/vite.config.ts#L80-L83), [package.json:6](https://github.com/pingdotgg/t3code/blob/acb90279/package.json#L6), [.vscode/settings.json](https://github.com/pingdotgg/t3code/blob/acb90279/.vscode/settings.json)

Conventional titles are documented; I found no first-party commitlint enforcement, branch-name validator, Husky/Lefthook configuration, Changesets setup, or CODEOWNERS file. Sample branches use several conventions, including `t3/`, `fix/`, `perf/` and `docs/`; those are observed practice, not enforced policy. [AGENTS.md:107–115](https://github.com/pingdotgg/t3code/blob/acb90279/AGENTS.md#L107-L115), tracked-file inventory (not published), sample PR metadata (not published)

The PR template requests problem, change, scope/approval, behavioral verification, UI evidence and model/harness attribution. Outside contributions must solve one underlying problem. Maintainers have a documented batching exception. [pull_request_template.md](https://github.com/pingdotgg/t3code/blob/acb90279/.github/pull_request_template.md), [CONTRIBUTING.md:63–98](https://github.com/pingdotgg/t3code/blob/acb90279/CONTRIBUTING.md#L63-L98)

### PR → CI: all 19 workflows

| Workflow file | Trigger and purpose |
|---|---|
| [ci.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/ci.yml#L1-L14) | PRs and main pushes; full validation. Superseded PR runs cancel; main runs do not. |
| [windows-tests.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/windows-tests.yml#L1-L30) | Manual investigation lane, optionally package/file scoped. Explicitly not a required gate. |
| [pr-vouch.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/pr-vouch.yml#L3-L18) | PR events, `/recheck-vouch`, and trust-file changes; synchronizes contributor-status labels. |
| [pr-size.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/pr-size.yml#L127-L158) | PR events; labels effective diff size, ignoring whitespace and excluding test lines in mixed PRs. |
| [cursor-hygiene-webhook.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/cursor-hygiene-webhook.yml#L3-L29) | Forwards selected main, PR, issue and discussion events to a configured Cursor endpoint. Missing credentials cause a skip. |
| [thread-transfer-report.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/thread-transfer-report.yml#L3-L28) | After CI; trusted publisher validates measurement artifacts and updates PR impact comments. |
| [web-preview.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/web-preview.yml#L18-L45) | `preview:web` label, subsequent pushes/reopens; same-repo Vercel preview deployment. |
| [desktop-macos-preview.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/desktop-macos-preview.yml#L3-L32) | One-shot `preview:mac` label; builds PR JavaScript without secrets and with read-only permissions. |
| [desktop-macos-preview-publish.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/desktop-macos-preview-publish.yml#L3-L25) | Successful preview build; checks current PR/head, label and author trust, then packages/signs using trusted code. Closure or label withdrawal cleans up downloads. |
| [mobile-fingerprint-check.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/mobile-fingerprint-check.yml#L3-L18) | PR events; compares native fingerprints and signals native changes. Advisory, not a required compatibility gate. |
| [mobile-eas-preview.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/mobile-eas-preview.yml#L3-L35) | Opt-in mobile deployment label; fingerprint-based preview builds/updates. Skips without Expo credentials. |
| [mobile-eas-production.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/mobile-eas-production.yml#L35-L90) | Relevant main pushes or manual dispatch; reconciles store builds and production OTA updates. |
| [mobile-android-apk.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/mobile-android-apk.yml#L3-L33) | Hourly/manual; mirrors compatible Play APKs to a rolling GitHub prerelease. |
| [mobile-showcase-screenshots.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/mobile-showcase-screenshots.yml#L3-L51) | Manual iOS/Android showcase capture across selected devices, appearances and themes; validates store assets. |
| [deploy-relay.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/deploy-relay.yml#L3-L34) | Every main push or manual main dispatch; serialized production relay deployment. |
| [release.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release.yml#L3-L37) | Stable tags, scheduled nightly checks and manual stable/nightly/preview releases. |
| [release-desktop.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release-desktop.yml#L3-L10) | Reusable target-specific packaging, native builds, signing and CLI archive smoke tests. |
| [publish-aur.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/publish-aur.yml#L5-L35) | Release call or manual dispatch; validates and publishes Arch packaging. |
| [issue-labels.yml](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/issue-labels.yml#L3-L14) | Relevant main changes/manual dispatch; synchronizes managed issue labels. |

Core CI contains:

- Lint/format plus Knip unused-code checks.
- Typechecking.
- Desktop build and preload-output validation.
- Non-server tests and a separate web test job.
- Six duration-weighted server shards.
- Three Rust-crate formatting/test checks.
- Conditionally executed macOS mobile-native static analysis.
- Release smoke checks.
- Transfer-artifact validation.
- A final aggregate `Check`.

The implementation uses cached dependency setup, sparse checkout excluding `.repos`, filtered installs where appropriate, parallel jobs, and background system-package installation. [ci.yml:20–130](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/ci.yml#L20-L130), [ci.yml:138–343](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/ci.yml#L138-L343), [weightedShardSequencer.ts:9–41](https://github.com/pingdotgg/t3code/blob/acb90279/apps/server/src/testUtils/weightedShardSequencer.ts#L9-L41)

The native-change detector treats incomplete API results or detection failure as a reason to **run** analysis. The aggregate explicitly rejects failures, cancellations and unexpected skips; only the justified native-analysis skip is allowed. [ci.yml:346–454](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/ci.yml#L346-L454), [ci.yml:481–514](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/ci.yml#L481-L514)

### Review → merge

**Verified live main ruleset:**

- PR required.
- Eight required checks: `Test`, `Check`, `Mobile Native Static Analysis`, `Release Smoke`, `Test Server 1–3`, and `Rust`.
- **Zero required approving reviews.**
- No required code-owner review, latest-push approval, or review-thread resolution.
- Required checks need not be strict against the latest base.
- Squash and rebase allowed; merge commits excluded by this ruleset.
- No merge-queue rule in the returned ruleset.

The six current server shards remain covered through aggregate `Check`, despite only three being individually named. I could not verify privileged bypass settings or private environment approval configuration. [Live ruleset](https://github.com/pingdotgg/t3code/rules/13856153), captured JSON (not published)

Observed main history uses single-parent commits with PR numbers, consistent with squash merging. That is observed behavior; rebase remains permitted.

AI review has several distinct roles:

- **CodeRabbit:** automatic review, project-specific architecture guidance, and `request_changes_workflow: false`. Its comments are advisory rather than configured change-request blockers. [.coderabbit.config.ts](https://github.com/pingdotgg/t3code/blob/acb90279/.coderabbit.config.ts)
- **Macroscope:** approvability rules plus incremental Effect-convention and UI-consistency agents. Both custom agents require `Check`, have budgets, and specify neutral conclusions. [.macroscope/approvability.md](https://github.com/pingdotgg/t3code/blob/acb90279/.macroscope/approvability.md), [Effect agent:1–25](https://github.com/pingdotgg/t3code/blob/acb90279/.macroscope/check-run-agents/effect-service-conventions.md#L1-L25), [UI agent:1–23](https://github.com/pingdotgg/t3code/blob/acb90279/.macroscope/check-run-agents/ui-consistency.md#L1-L23)
- **Copilot:** reviews appeared in sampled PRs. I found no checked-in Copilot review workflow establishing its trigger policy.
- **Contribution triage:** a detailed skill exists, but its own documentation explicitly distinguishes intended rollout from deployed automation. Vouch labels do not grant triage exemption. [CONTRIBUTING.md:114–142](https://github.com/pingdotgg/t3code/blob/acb90279/CONTRIBUTING.md#L114-L142)

The evidence does **not** support claiming all review bots must finish before merge: [#19310](https://github.com/pingdotgg/t3code/pull/19310) had successful required CI and Macroscope approval while its retrieved CodeRabbit status remained pending.

### Main → release/deploy

There are separate distribution paths:

**Desktop and CLI:** scheduled checks run twice hourly, but automatic nightlies require new commits and a six-hour publication gap. Manual nightlies bypass those scheduling conditions. Manual stable releases select the latest published nightly’s commit, rather than current main HEAD; explicit stable tags provide another path. [.github/scripts/check-nightly-release.cjs:1–102](https://github.com/pingdotgg/t3code/blob/acb90279/.github/scripts/check-nightly-release.cjs#L1-L102), [release.yml:65–100](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release.yml#L65-L100)

The release reruns quality checks and tests, builds common JavaScript once, then packages six desktop targets and five CLI archives. CLI archives execute smoke tests on their target runners. npm publishing uses OIDC/provenance and a dry run before publishing. GitHub release publication follows required artifacts and CLI publication. [release.yml:233–337](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release.yml#L233-L337), [release.yml:449–757](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release.yml#L449-L757), [release-desktop.yml:550–580](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release-desktop.yml#L550-L580)

**Hosted web and marketing:** deployments are staged early; public aliases/promotions wait for release publication. Preview-channel releases exclude normal update metadata and skip hosted-web/AUR promotion. Stable finalization pushes aligned versions back to main using a release app. [release.yml:826–840](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release.yml#L826-L840), [release.yml:960–1062](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release.yml#L960-L1062), [release.yml:1197–1319](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release.yml#L1197-L1319)

**Mobile:** relevant main changes reconcile each platform’s fingerprint. Matching completed binaries receive OTA updates; absent compatible binaries cause builds/submissions; active builds avoid duplicate builds. Store release still includes manual platform steps. [mobile-eas-production.yml:3–31](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/mobile-eas-production.yml#L3-L31), [reconciliation:268–338](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/mobile-eas-production.yml#L268-L338)

**Relay:** deploys independently on main pushes, serialized in a production environment. It does not wait for that push’s CI in the checked-in workflow. [deploy-relay.yml:22–34](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/deploy-relay.yml#L22-L34)

## 2. Agent setup

There is one first-party root `AGENTS.md`; `CLAUDE.md` contains exactly `@AGENTS.md`. Claude skills link to the shared `.agents/skills` directory. I found no tracked first-party `.codex` configuration or nested package `AGENTS.md` files. [.claude/skills](https://github.com/pingdotgg/t3code/blob/acb90279/.claude/skills), [CLAUDE.md](https://github.com/pingdotgg/t3code/blob/acb90279/CLAUDE.md)

Three short rules capture the intended discipline:

> “Do not run repo-wide checks.”

> “Backend behavior changes ship with focused tests for that behavior.”

> “Copy in, never symlink.”

[AGENTS.md:87–103](https://github.com/pingdotgg/t3code/blob/acb90279/AGENTS.md#L87-L103)

The guide also constrains agents to:

- Avoid pattern-based process killing and writes to the live installation.
- Preserve remote and multi-client behavior.
- Keep server features behind services and guarded command paths.
- Obtain explicit PR authority.
- Provide before/after UI evidence and recordings for timing.
- Keep plans, research notes and PR-only assets out of application source.
- Verify bot findings against code and explain unsupported findings.

These are instructions, not uniformly executable enforcement. [AGENTS.md:59–76](https://github.com/pingdotgg/t3code/blob/acb90279/AGENTS.md#L59-L76), [AGENTS.md:107–159](https://github.com/pingdotgg/t3code/blob/acb90279/AGENTS.md#L107-L159)

The procedures are deliberately reusable:

- **`test-t3-app`:** isolated dev server, meaningful data, T3 Browser panel, retained environment for inspection.
- **`test-t3-mobile`:** native-client fingerprint/build reconciliation, Device panel, Metro, pairing, evidence and scoped teardown.
- **`contribution-triage`:** upstream-pinned trusted policy, complete PR evidence, explicit outcomes, mutation rechecks and idempotent retries.

The `agents/openai.yaml` files are skill discovery metadata, not independent reviewer-subagent definitions. [App skill](https://github.com/pingdotgg/t3code/blob/acb90279/.agents/skills/test-t3-app/SKILL.md), [Mobile skill](https://github.com/pingdotgg/t3code/blob/acb90279/.agents/skills/test-t3-mobile/SKILL.md), [Triage skill:39–67](https://github.com/pingdotgg/t3code/blob/acb90279/.agents/skills/contribution-triage/SKILL.md#L39-L67)

Static constraints reinforce that prose:

- Strict TypeScript options and Effect diagnostics.
- Custom Oxlint rules for permission bypass, raw MCP registration, global runtime access, unsupported Hermes APIs, test construction and unexplained suppressions.
- UI rules for shared primitives, theme tokens and styling consistency.
- Knip checks for unused files, dependencies and runtime exports.

[tsconfig.base.json](https://github.com/pingdotgg/t3code/blob/acb90279/tsconfig.base.json), [vite.config.ts:126–177](https://github.com/pingdotgg/t3code/blob/acb90279/vite.config.ts#L126-L177), [vite.config.ts:250–364](https://github.com/pingdotgg/t3code/blob/acb90279/vite.config.ts#L250-L364), [development.md:121–137](https://github.com/pingdotgg/t3code/blob/acb90279/docs/operations/development.md#L121-L137)

## 3. What is genuinely excellent

These are judgments based on the verified mechanisms above.

**The instructions target actual failure modes.** Process ownership, live-database protection, missing client surfaces and remote-origin mistakes are concrete operational hazards. The guide explains them instead of offering generic “write clean code” advice.

**Local speed and global confidence are separated.** Formatting-only commits and focused local proof reduce wasted work; CI independently validates integration. The aggregate check preserves enforcement as the job layout evolves.

**Recurring mistakes become tools.** Architecture and UI expectations appear in agent prose, custom lint and review prompts. This is particularly useful for agents: the desired pattern becomes discoverable and violations produce actionable feedback.

**Testing reduces dependence on live AI services.** Protocol replay, deterministic runtimes and event-based synchronization allow extensive provider behavior coverage without making every PR depend on credentials, model availability or timing.

**Performance is treated as behavior.** Transport ceilings and a main-baseline PR report make a subtle data-volume regression visible before users experience it. The fixtures explicitly describe what they measure and what they do not.

**The privileged preview boundary is unusually careful.** Fork code builds without secrets; trusted publishing checks identity and current head, consumes a one-shot label, validates ZIP paths/types/sizes, and extracts outside the trusted checkout. This is a strong reusable pattern for signing unmerged artifacts. [stage-preview-bundle.py:9–72](https://github.com/pingdotgg/t3code/blob/acb90279/.github/scripts/stage-preview-bundle.py#L9-L72)

**Review is evidence-driven rather than obedient.** In [#19102](https://github.com/pingdotgg/t3code/pull/19102), one CodeRabbit finding was fixed with regression coverage; another was challenged using the shared restart policy and tests, and [the bot withdrew it](https://github.com/pingdotgg/t3code/pull/19102#discussion_r4261887901).

**The PR evidence is genuinely agent-oriented.** [#19310](https://github.com/pingdotgg/t3code/pull/19310) reports focused reproduction and lifecycle proof; [#18954](https://github.com/pingdotgg/t3code/pull/18954) clearly distinguishes validated configuration from unexercised hosted behavior. These are author-reported checks, supported by observed CI—not tests I independently reran.

## 4. Weaknesses, gaps and non-transferable choices

1. **Documentation drift can undermine the safety model.** Cursor Cloud rules recommend broad `vp check --fix`, describe shared state, and describe automatic origin wiring. These conflict with current canonical instructions and dev-runner behavior. Release documentation also retains older validation guidance alongside the newer preview channel. [.cursor/rules/cursor-cloud.mdc:20–32](https://github.com/pingdotgg/t3code/blob/acb90279/.cursor/rules/cursor-cloud.mdc#L20-L32), [release.md:504–520](https://github.com/pingdotgg/t3code/blob/acb90279/docs/operations/release.md#L504-L520)

2. **Human review is not a hard merge gate.** Zero required approvals, no required thread resolution, and non-strict latest-base checks may suit this team’s velocity. They should be conscious choices in an extraction, especially for security-sensitive projects.

3. **Passing PR CI does not guarantee main stays green.** A contemporaneous [repair PR, #19298](https://github.com/pingdotgg/t3code/pull/19298), reports broken main typechecking and startup. This demonstrates a real regression; it does not establish whether stale-base validation, bypass or another cause let it land.

4. **Some production paths do not wait for post-merge CI.** Relay and mobile production workflows trigger directly on main pushes. Universal installation should decide whether to require successful checks for the exact deployed revision.

5. **Platform verification is uneven.** Windows tests are manual and explicitly described as not ready for required CI. Routine browser coverage is not comprehensive product E2E coverage. The Electron smoke script exists, but I found no workflow invoking it; its success criterion is absence of selected fatal log strings. [windows-tests.yml:1–8](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/windows-tests.yml#L1-L8), [smoke-test.mjs:30–57](https://github.com/pingdotgg/t3code/blob/acb90279/apps/desktop/scripts/smoke-test.mjs#L30-L57)

6. **Reviewer exclusions are broad.** Macroscope excludes tests, package manifests, lockfiles and agent skills, among other files. These can materially affect verification and policy. A universal workflow should assign those changes another review path. [.macroscope/ignore.md](https://github.com/pingdotgg/t3code/blob/acb90279/.macroscope/ignore.md)

7. **Triage automation cannot be assumed deployed.** The policy documents partial event coverage and explicitly declines to claim full automation. The Cursor workflow forwards events; its downstream configuration is private and unverified.

8. **Release validation differs from merge validation.** Release tests use three server shards, while current PR CI uses six; the release graph does not repeat every CI gate. Maintaining separate graphs creates drift risk. [release.yml:233–337](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release.yml#L233-L337)

9. **Stable promotion selects source, not identical artifact bytes.** Stable rebuilds the nightly commit with stable version metadata. This gives source-level soak, but weaker artifact identity than promoting exactly the tested bytes.

10. **Publication is not fully atomic.** npm precedes GitHub release, and later promotion/finalization steps can fail. Signing credentials are optional and missing credentials can produce unsigned builds. No unified rollback workflow was found. [release.yml:695–772](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release.yml#L695-L772), [release-desktop.yml:437–484](https://github.com/pingdotgg/t3code/blob/acb90279/.github/workflows/release-desktop.yml#L437-L484)

11. **Some choices are inherently project-specific.** Effect service conventions, `tsc-rs`, Hermes compatibility, Expo fingerprints, Electron/WSL packaging, Tailscale pairing, Blacksmith runners and shared dev credentials require adaptation.

12. **Model bans are policy, not transferable engineering guarantees.** Current contribution policy rejects code authored by specified models while allowing those models as reviewers. Preserve model attribution; do not universalize a time-sensitive allowlist or denylist. [CONTRIBUTING.md:100–110](https://github.com/pingdotgg/t3code/blob/acb90279/CONTRIBUTING.md#L100-L110)

## 5. Universal extraction

The transferable principles are:

1. One canonical agent guide, with thin harness-specific entrypoints.
2. A short architecture map and explicit product invariants.
3. Isolated task workspaces, data, processes and ports.
4. Deterministic setup with pinned tools and audited install behavior.
5. Focused local verification plus independent full CI.
6. Regression tests for observable behavior.
7. Controlled async synchronization and replayable external boundaries.
8. Executable rules for recurring agent mistakes.
9. A stable aggregate merge check that rejects unexpected skips and cancellations.
10. PRs containing reproduction, evidence, limitations and attribution.
11. Review findings verified against source, with reasoned rejection permitted.
12. Trusted policy kept separate from submission content.
13. Untrusted builds kept separate from privileged publishing.
14. Releases tied to explicit revisions and tested distributed artifacts.
15. Ongoing maintenance of instructions, CI and release consistency.

A compact skeleton for the handoff file follows. The full draft (not published) expands this into an installation procedure and explicitly marks adaptation points.

```markdown
# WORKFLOW.md — Install an agent-first development pipeline

## 1. Discover the existing project
Read agent guidance, manifests, tests, hooks, CI, release code,
contribution policy and live repository rules.
Preserve working conventions; record current gaps.
ADAPT: stack, supported platforms, owners and action authority.

## 2. Establish canonical agent guidance
Document product invariants, code locations, architectural boundaries,
scoped commands, completion evidence and forbidden state/process actions.
Link harness-specific entrypoints to this source.
Remove conflicting and obsolete instructions.

## 3. Isolate each task
Provide deterministic, noninteractive workspace setup.
Pin runtime/package-manager versions and use the lockfile.
Separate mutable state, queues, credentials and ports.
Seed representative disposable data; retain process ownership.
ADAPT: worktrees, containers, databases and development credentials.

## 4. Define the command contract
SETUP = <adapt>
DEV = <adapt>
TEST_FILES = <adapt>
LINT_FILES = <adapt>
CHECK_PACKAGE_AND_CONSUMERS = <adapt>
FORMAT_STAGED = <adapt>
FULL_CI = <adapt>
SMOKE_DISTRIBUTED_ARTIFACT = <adapt>

Use the smallest meaningful local proof.
Keep hooks fast; CI independently enforces the full gate.

## 5. Encode recurring defects
Use formatting, static checks and unused-code detection.
Add reliable custom rules for architectural and permission boundaries.
Require reasons for suppressions and human review for weakened gates.
ADAPT: language, framework and lint mechanisms.

## 6. Verify changed behavior
Require focused regression proof for behavior changes.
Exercise real business logic; substitute external boundaries.
Wait on explicit completion signals rather than arbitrary sleeps.
Check affected clients, entrypoints, adapters and reverse actions.
ADAPT: integration, browser, native and performance verification.

## 7. Install independent CI
Run full validation on PRs and the default branch.
Parallelize safely; shard by cost when worthwhile.
Cancel superseded PR runs; serialize publishing.
Run checks when change detection is uncertain.
Require one stable aggregate that fails on failure, cancellation
or unexpected skip.
ADAPT: repository rules, platform matrix and merge policy.

## 8. Separate trust domains
PR/fork execution receives no production secrets.
Privileged jobs run trusted code and consume validated data.
Validate artifact identity, current revision and extraction boundaries.
Recheck state before mutations; make retries idempotent.
ADAPT: token permissions, signing and deployment environments.

## 9. Make PRs reviewable
Require problem, change, scope, verification, limitations and attribution.
Configure optional reviewers around maintained project rules.
Verify findings against current code; fix or explain each finding.
Keep eligibility, bot approval and merge authority distinct.
ADAPT: human review requirements and contribution policy.

## 10. Release deliberately
Ship an explicit revision and smoke-test distributed artifacts.
Separate experimental channels from stable defaults/update feeds.
Publish dependencies before dependent clients.
Gate production promotion on successful exact-revision checks.
Define compatibility, health checks and rollback.
ADAPT: versioning, registries, platforms and deployment targets.

## 11. Prove installation is complete
Report changed files, resolved commands, CI evidence, live rules,
known limitations and remaining external configuration.
Do not equate written configuration with verified hosted behavior.
```

The extraction should retain T3’s **focused agent loop and strong CI boundary**, while explicitly deciding the review requirements, latest-base policy, deployment gates and supported-platform coverage appropriate to the receiving project.
