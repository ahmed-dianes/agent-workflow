# AGENT-WORKFLOW.md

> **For humans:** give this file to a coding agent and say *"Set up this workflow in this repo"*, or *"Onboard me
> to this repo"*. It sets things up locally and asks before touching GitHub.

## Team decisions (edit before sending)

| | |
|---|---|
| Git host | GitHub |
| Into `main` | A human approves (1 review, not the PR's author). Branch up to date with `main`, squash merge. Agents never merge into `main`. |
| Required CI | One aggregate check, `ci-ok` |
| CI speed | A PR run finishes in under ~10 minutes |
| Where checks run | CI jobs and agents' heavy checks run on boat.dev VMs from one saved snapshot. GitHub-hosted runners are the fallback. (Part 4) |
| Environments | `main` deploys to staging. A human promotes a commit that passed staging to production. Database: Supabase. (Part 5) |
| AI review | On every PR, advisory. A different model family from the one that writes most of the code. |
| Performance | One deterministic budget test in `ci-ok`. Ceilings move only with a human's sign-off. |
| Pull requests | An agent opens one when asked, or as a swarm worker |
| Swarm mode | Agents merge into a `feature/*` branch on their own; a human approves the final PR into `main` (Part 3) |

---

## Part 1 — Setting it up

Use the tools the repo already has. For a new project, use the current standard for its ecosystem. Local changes are
yours to make. Ask before anything that leaves the machine (pushing, opening PRs, GitHub settings, secrets) and before
deleting tracked files. Install on a new branch, commit there, and don't push until asked. Where the repo's own rules
conflict with this file, ask which wins.

**Already set up?** If `AGENTS.md` contains `<!-- agent-workflow: v2 -->`, read it and give the developer a short
briefing: what the project is, where code lives, the commands, the dangers, the merge rules. Change nothing.

**Otherwise,** read the repo and its current GitHub settings first, then install the following.

1. **`AGENTS.md`** from Part 2, filled in with this repo's facts. If one already exists, keep its rules and fit them
   into the template; don't drop them to match its size hints. Delete template sections that don't apply, and keep it
   under ~150 lines, since agents read it on every task. `CLAUDE.md` contains just `@AGENTS.md`. Other agent rule
   files point to `AGENTS.md`. Remove any that contradict it.
   - Two things matter most and only come from reading the repo: **how an agent could do real damage here** (real
     data, paid calls, shared processes, migrations, secrets), and **every surface a change has to reach**.
   - Reusable procedures, such as how to run and check the app, go in skills (`.agents/skills/`), not in
     `AGENTS.md`.
2. **Commands** for setup, dev, focused test/lint/typecheck, the full check and the perf budget. They must be
   non-interactive and exit non-zero on failure. Reuse existing scripts.
3. **One worktree per task.** Add a setup script that installs from the lockfile and links gitignored env files from
   the main checkout, never replacing a real file. Run it wherever worktrees are created. Dev state (databases,
   caches) lives per worktree, never in shared or production locations. Worktrees mustn't fight over ports, and
   there's demo data to test against rather than an empty database.
   - Agents edit and run quick checks on this machine. Heavy checks (the full suite, database, browser) run on a
     remote VM from the CI snapshot (Part 4), so agents never queue for this machine.
4. **CI** on PRs and on pushes to `main`, built to Part 4 so it stays fast. Jobs run in parallel and the token is
   read-only. The only required check is the aggregate `ci-ok` job (Part 6). A job may be skipped only when the
   change-detection job said it isn't needed; any other skip fails `ci-ok`.
   - A pre-commit hook may format staged files, nothing slower. Skip the hook if the codebase isn't formatted yet;
     that's the developer's call.
5. **AI review** using the prompt in Part 6, in its own workflow. Run it on same-repo, non-draft PRs only, because
   forks must not get secrets. It needs `pull-requests: write`; if the action doesn't post comments itself, add a step
   that posts its output. It needs a token: ask the developer to add it as a secret, and never ask for the value.
6. **A performance budget.**
   - Pick the one path whose cost matters most. Run a deterministic, heavy fixture through the real code, and
     measure structural numbers (bytes, messages, queries, rows, renders), not time.
   - Put the ceilings ~30% above today's numbers, in a constant with a comment saying what the numbers don't cover.
   - The test writes its numbers before asserting, as JSON to the path in `PERF_RESULT_PATH`. The CI job fails if
     that file is missing, and is part of `ci-ok`.
   - Later, if needed: a sticky PR comment comparing the PR with `main`, built on `workflow_run`. That job runs
     publisher code from the default branch only and treats PR artifacts as data.
   - Noisy numbers, such as timings, are reported, never enforced.
   - For a profiling pass: one tracking issue, one finding per PR. A finding closes only with a measured
     improvement or a written "not worth it".
7. **PR tools.**
   - A PR template asking for: problem, change, verification (what ran, what was seen, what couldn't be checked,
     screenshots), and the model and harness.
   - `scripts/pr-status.sh` from Part 6. If the agent harness has its own PR watcher, prefer that.
   - For UI projects: `scripts/pr-evidence.sh` from Part 6, plus one command that runs the app with demo data and no
     sign-in.
8. **Environments and deploys**, if any, as in Part 5.

**GitHub settings** (show the developer first; needs admin):
- Squash merge only, with the PR title as the commit message. Delete branches on merge. Allow auto-merge and
  "update branch". Create the `needs-human` label.
- A ruleset on `main`: PR required, 1 approval, stale approvals dismissed on push, threads resolved, `ci-ok` required
  and up to date, no force-push or deletion. Leave the bypass list empty, or admins can merge without approval.
- A ruleset on `production`, if there is one (Part 5): no force-push or deletion, `ci-ok` required.
- Repository variables `CI_RUNNER` and `CI_RUNNER_SMALL` (Part 4). The developer adds the `BOAT_API_KEY` secret.
- Private repos on GitHub Free get neither rulesets nor branch protection. They need a paid plan (Pro or Team);
  until then, the rules are only the ones written in `AGENTS.md`. Tell the developer.

**Prove it, then report.**
- Run the full check locally.
- Break things on purpose and watch them fail: a lint error, and a perf ceiling set below today's number.
- With approval, open a throwaway PR (not a draft, which the AI review skips) and confirm three things: `ci-ok` goes red when a job fails, the AI review
  posts, and merge stays blocked without an approval.
- Note that PR's CI time, and the slowest job. If it's over the target, name the next step from Part 4. Run it once
  on the fallback runner too, and compare time and cost.
- Report what was verified, what is only configured, and what is waiting on someone.

---

## Part 2 — `AGENTS.md` template

````markdown
<!-- agent-workflow: v2 -->
# <Project>

<What it is and who uses it, in one or two sentences.>

## Non-negotiables
<3–5 product rules every change keeps.>

## Ways to hurt yourself
<The dangerous things in this repo: where real data lives, what costs money, other people's processes, migrations, secrets.>

## Hit every surface
The most common defect is a change that works where you tried it and is missing elsewhere. Before calling work done,
say which of these applied:
- <entry points: UI, API, background jobs…>
- <clients or platforms>
- <shared contracts in `<path>`>
- <reverse actions: every way in has a way out>
- <permissions>
- <docs that become wrong>

## Where code lives
- `<path>`: <what goes here, and its rule>

## Commands
| | |
|---|---|
| Set up a worktree | `<…>` |
| Run with demo data, no sign-in | `<…>` |
| Focused test / lint / typecheck | `<…>` |
| Full check (what CI runs) | `<…>` |
| Heavy check, off this machine | `boat run --from ci-base --type large --push . -- <install from lockfile && check>` |
| Perf budget | `<…>`; ceilings in `<path>` |
| A PR's state | `scripts/pr-status.sh [n]`; wait with `gh pr checks <n> --watch` |
| Screenshots for a PR | `scripts/pr-evidence.sh <pr> <files>` |

## How we work
- One task, one branch, one worktree. For a new feature, talk it through with the developer until they say go.
- Check what you touched; CI runs the full suite. A behavior change comes with a test that fails without it.
  Anything heavier than a quick check runs off this machine, using the command above.
- UI changes get before and after screenshots in the PR (a video for motion), uploaded with `pr-evidence.sh`, never
  committed.
- Open a PR only when asked. Use the template and end with the model and harness.
- For each review finding, check it against the code. Fix it or answer it with evidence, then resolve the thread.
  Use `pr-status.sh` and `gh pr checks --watch`; don't write your own polling. Done when `ci-ok` is green and
  nothing is unresolved.
- Merging into `main` is a human's call. Never push to `main`, force-push a shared branch, or get round the branch
  rules.
- Never weaken a test, check or perf ceiling to get green. Raising a ceiling needs a reason and a human's sign-off.
  When you make something smaller, lower its ceiling in the same PR.

## Taste
<A few project-specific style rules. Any that keeps being broken becomes a lint rule.>

If a rule here fights the task, say so and get a decision before breaking it.
````

---

## Part 3 — Swarm mode

For work that lands as **one PR into `main`**. The developer and an agent shape it until the developer says go. Then
an orchestrator agent runs workers in parallel, their PRs merge into an integration branch on their own, and the human
approves only the final PR.

**Before starting**
- The integration branch is `feature/<name>`, created from the latest `main`. A long-running overhaul can use its
  existing branch.
- This workflow is installed on that branch, and `ci-ok` is green on it.
- The foundation is stable. If shared contracts and structure are still moving, one agent finishes them first;
  parallel agents on a moving base mostly produce conflicts.
- Limits are set: workers at once (default 3), attempts per task (default 3).

**GitHub** (ask first)
- CI also runs on pushes to `feature/**`, and to an existing integration branch if one is used.
- A ruleset for `feature/**` and any existing integration branch: PR required, **0 approvals**, `ci-ok` required,
  threads resolved, squash only. Don't require the branch to be up to date there: with several workers, every merge
  would make all the other PRs re-run CI. CI runs after each merge instead, and a red branch stops new merges.
- A PR's author can't approve it. If agents use the developer's account, the final PR is the developer's own, so
  give the agents their own account or GitHub App.
- A merge queue does the up-to-date check without the re-runs, but GitHub only offers it on public repos owned by an
  organization and on Enterprise Cloud. Use it there with more than ~5 workers.

**The run**
- The orchestrator splits the work into small, independent, checkable tasks in a tracking issue, grouped in waves.
  Tasks that change shared contracts go first, one at a time.
- One worker per task, in its own worktree from the latest integration branch, following `AGENTS.md`.
- Branches go two levels deep only: `swarm/*` into the integration branch. A task that needs another task's work
  waits for it to merge (a later wave) instead of branching from it. Branch chains are where the time goes.
- Only one task at a time changes database migrations.
- Each PR is reviewed by the other model family. When it's green and resolved, the worker turns on auto-merge.
- If the integration branch goes red, nothing new starts until it's fixed.
- At the end, the orchestrator runs the whole feature once and adds before and after screenshots of every changed
  screen. It opens the PR into `main`, takes it through the same review loop, then requests the human's review and
  stops.

| Workers may | Workers may not |
|---|---|
| Push their own `swarm/*` branch; upload evidence | Push to the integration branch or `main` directly; force-push anything shared |
| Open PRs into the integration branch | Open, approve or merge anything into `main` |
| Answer reviews; turn on auto-merge once green; lower a perf ceiling they beat | Change CI, branch rules, settings, secrets, `AGENTS.md` or lint config; raise a perf ceiling |

A task that needs a forbidden change gets the label `needs-human` and doesn't auto-merge. To enforce this rather than
trust it, put those paths in `CODEOWNERS` and require code owner review on the `feature/**` ruleset.

**Escalate in the tracking issue, and keep other tasks going, when:**
- a product decision isn't answered;
- a task needs something only a human may give;
- a task fails its attempts;
- review and author still disagree after two rounds;
- or the integration branch stays red after one fix.

---

## Part 4 — CI that stays fast

Agents open many PRs, and every one of them waits on CI, so a slow run slows down all the work. Most of a slow run is
waste: the same thing built in every job, tests whose inputs didn't change run again, and shards that finish at
different times. t3code runs about 1,900 test files in 2–7 minutes per PR this way; its `ci.yml`,
`scripts/test-cached.ts` and `.github/scripts/ci-changes.cjs` are a working reference.

**From the start, in every repo**
- Cancel superseded PR runs, and set a `timeout-minutes` on every job from measured times plus headroom (Part 6).
  Matrices use `fail-fast: false`, so one failing shard doesn't hide the others.
- Cache dependencies by lockfile, and large downloads (browsers, Electron, toolchains) by version. Install system
  libraries only when they're missing.
- Long jobs start at once instead of waiting for a setup job whose work they can do themselves.

**Where jobs run: boat.dev** (docs: `https://docs.boat.dev/ci.md`, index at `https://docs.boat.dev/llms.txt`)
- Each job's `runs-on` comes from a repository variable, with GitHub's runner as the fallback:
  `runs-on: ${{ vars.CI_RUNNER || 'ubuntu-latest' }}`, where `CI_RUNNER` is `boat/from=ci-base`. Use one variable
  per size (`CI_RUNNER_SMALL` is `boat-small/from=ci-base`). Switching runners is then a setting, not a PR.
- `boat-small` (2 vCPU) is for small jobs such as `changes` and `ci-ok`, and `boat` (4 vCPU) for the rest. Go bigger
  only per step 6. Billing is per second. At the time of writing it's about 20× cheaper per minute than GitHub's runner
  of the same size; check boat's pricing page rather than memory. It needs boat's $20/month plan.
- The developer connects it with `boat ci connect`, which installs boat's GitHub app and needs a browser and admin.
  Jobs get only GitHub Actions secrets. Public repos don't run on boat unless that's turned on; leave it off.
- **The `ci-base` snapshot** holds the toolchain, `jq`, the package manager's store filled from the lockfile, test
  browsers and Docker images. No secrets.
  - A `ci-snapshot` workflow rebuilds it when the lockfile or toolchain files change on `main`, and weekly. It uses a
    `BOAT_API_KEY` secret, a key with the `ci` preset. Saving under the same name replaces the snapshot; jobs already
    running keep their copy.
  - Jobs still install from the lockfile, so a stale snapshot costs time, never correctness.
- **What a snapshot keeps.**
  - Not `~/.cache`. Install browsers elsewhere (`PLAYWRIGHT_BROWSERS_PATH=$HOME/.local/share/ms-playwright`, set both
    in the snapshot and in the workflow's `env`), and move other tool caches out of it too.
  - A Docker image is kept only if a container uses it (`docker create --name keep-<x> <image>`). Registry images
    are pulled again at each start, so database tests use one Postgres image with the migrations applied. The full
    local Supabase stack runs only in jobs that need it.
- **Agents' heavy checks** use the same snapshot through `boat run`, as in the `AGENTS.md` commands. `--push` sends
  tracked files and new files that aren't ignored, so `node_modules`, build output, `.git` and gitignored env files
  stay behind. The command installs from the lockfile first. Test variables go in with `-e` or boat's stored
  secrets, never production keys.
- Start limits per minute, hour and day apply to both. `boat ci` shows each job's wait, run time and cost; read it
  alongside step 1.

**When a run passes the target, measure, then take these steps in order.** Stop once it's under the target.
1. **Measure** where the minutes go, per step, in the last green run:
   `gh api repos/OWNER/REPO/actions/runs/RUN_ID/jobs --jq '.jobs[] | .name as $j | .steps[] | "\($j) | \(.name) | \(.started_at) → \(.completed_at)"'`.
   It usually shows a build repeated in every job, one shard much slower than the rest, or setup costing more than
   the tests.
2. **Build shared things once.** One job builds the artifact (binary, bundle, generated code) and hands it to the
   others as a workflow artifact. Cache the output itself, keyed on `hashFiles()` of everything the build reads:
   sources, lockfile, toolchain files. Point the tests at the prebuilt artifact, or they quietly rebuild it.
3. **Skip what a change can't affect.** A small first job, with a sparse checkout, maps the changed files to
   packages and their dependents, and outputs `"true"` or `"false"` for each job, keyed by job name. It fails open:
   if the diff can't be worked out, everything runs. The `ci-ok` in Part 6 accepts exactly these skips.
4. **Split tests by measured time**, not file count. Keep seconds per file in a committed weights file refreshed from
   test reports. Assign the longest files first to the lightest shard, and give new files the median. Add shards
   while the wall time still drops; they're cheap only once each shard's setup is small.
5. **A test-result cache**, the biggest win and the one that needs care. A test file whose inputs are byte-for-byte
   the same as when it passed will pass again, so record the pass and skip the file next time.
   - The key hashes the file's import graph (transitive, aliases resolved, setup files included), the global inputs
     (lockfile, manifests, test config, toolchain versions, the hash of any prebuilt artifact) and the files it reads
     at run time (fixtures, directories it walks). Run-time reads don't show up in imports, so find them
     deliberately. When unsure, put it in the key: a missing input passes a test that should have run.
   - Record a pass only when every test in the file passed. Store records where CI can read them: a remote task
     cache, an object store, or an orphan branch with one file per key. Local and pre-merge runs may write them too.
   - Every run re-runs at least one cached pass for real, picked from the run id. If it fails, the key is missing an
     input: fix the key.
   - Keep a manual full run that ignores the records (`workflow_dispatch`). Never cache perf or timing tests.
     Keep tracking flaky tests, since a cached pass hides a flake.
   - Prefer the cache your test runner or build tool already has (t3code uses Vite+'s, with `--verify-hits 1`) over
     writing one.
6. **Bigger runners, last.** More vCPUs only help parallel work. A suite that runs one file at a time on purpose
   keeps about 1.5 cores busy, so spread it over more machines instead. On boat, a size with twice the vCPUs costs
   twice as much per second, so it pays only when the job's time halves.

A CI speedup is a change like any other. Compare wall time and billed minutes before and after on the same commit,
and break a test on purpose to confirm it still fails `ci-ok`.

---

## Part 5 — Environments and deploys

An environment is a deployment of a commit, not a branch that work gets merged through. Every extra merge step adds
another CI run, more conflicts and more waiting. It also means production would run a commit that nobody tested.

- **Per PR:** preview deploys, if the host has them. A database preview only for PRs that change the database. With
  Supabase that's Branching with "Supabase changes only" turned on. A preview builds the schema from
  `supabase/migrations` and its data from `seed.sql`, never from production data.
- **Staging is `main`.** Each push to `main` deploys to staging once `ci-ok` has passed for that commit: migrations
  first, then the app. Then a smoke test runs against the staging URL. For UI changes, an agent also clicks through
  in a remote VM's browser and posts screenshots on the PR.
- **Production is a promoted commit.** A human runs a "Promote" workflow (`workflow_dispatch`) that fast-forwards the
  `production` branch to a commit that passed staging. Production then runs exactly what staging tested: migrations
  first, then the app.
  - The `production` ruleset blocks force-push and deletion and requires `ci-ok`.
  - Agents never promote.
  - Promotion becomes automatic only when the team decides staging's checks are enough.
- **Supabase.** Use one of these, following Supabase's current docs:
  - Branching, with `production` as the production branch and a persistent branch for staging.
  - Two projects, with `supabase db push` from the deploy workflows ("Managing environments" in their docs).
- **Migrations.**
  - The database and the app never deploy at the same instant, so every migration must work with both the old and
    the new app code. Add first; remove old columns in a later release.
  - A migration rebased onto newer ones gets a new timestamp, so the order stays linear.
  - Supabase Branching runs each migration in a transaction, so `create index concurrently` fails there.
  - Migrations only roll forward.
- **Rolling back** means promoting a revert through the same path, or using the host's instant rollback for the app.
- Deploy secrets live in GitHub environments for staging and production, never in a VM or snapshot.

---

## Part 6 — Exact pieces

These are subtle or tested; copy them rather than rewriting them.

**`ci-ok`.** The only required check. Every other job goes in `needs`. Without `always()`, a skipped or cancelled run
reports success. A job passes if it succeeded, or if it was skipped and the `changes` job (Part 4, step 3) said
`"false"` for it. Without a `changes` job, any skip fails.

```yaml
  ci-ok:
    name: ci-ok
    if: always()
    needs: [typecheck, lint, test, build, perf]   # ADAPT: every job in this workflow, `changes` included if present
    runs-on: ${{ vars.CI_RUNNER_SMALL || 'ubuntu-latest' }}   # needs jq, so the snapshot has it
    timeout-minutes: 5
    steps:
      - env:
          RESULTS: ${{ toJSON(needs) }}
        run: |
          echo "$RESULTS" | jq -r 'to_entries[] | "\(.key): \(.value.result)"'
          echo "$RESULTS" | jq -e '(.changes.outputs // {}) as $run
            | all(to_entries[]; .value.result == "success"
                or (.value.result == "skipped" and $run[.key] == "false"))' > /dev/null
```

**Cancel superseded runs.** At the top of the CI workflow. Pushes to `main` and integration branches are never
cancelled.

```yaml
concurrency:
  group: ci-${{ github.event.pull_request.number || github.sha }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```

**AI review** (`anthropics/claude-code-action@v1`, or `openai/codex-action` for the other family). Guard the job with
`if: github.event.pull_request.draft == false && github.event.pull_request.head.repo.full_name == github.repository`,
and use this prompt:

```text
Review PR #<number>. Read AGENTS.md first; this review enforces it. Review only the changed code. Report bugs with a
concrete failure scenario, security problems, behavior changes without a test, broken AGENTS.md rules and missed
surfaces. Skip what a linter would catch. Post findings as inline comments. If nothing significant is found, say so
in one comment.
```

**`scripts/pr-status.sh`.** The one way agents read a PR's state, including the reviewers' latest verdicts.

```bash
#!/usr/bin/env bash
# One-shot PR status for agents: CI rollup, review decision, the latest reviews and comments (where reviewers
# give their verdict) and unresolved threads, in ONE GraphQL call.
# Usage: scripts/pr-status.sh [pr-number]   (defaults to the PR for the current branch)
# Wait for CI first with: gh pr checks <n> --watch --interval 60   (don't write your own poller)
set -euo pipefail
n="${1:-$(gh pr view --json number -q .number)}"
repo="${GH_REPO:-$(gh repo view --json nameWithOwner -q .nameWithOwner)}"
gh api graphql -F owner="${repo%/*}" -F name="${repo#*/}" -F n="$n" -f query='
query($owner:String!,$name:String!,$n:Int!){repository(owner:$owner,name:$name){pullRequest(number:$n){
  url headRefOid reviewDecision mergeStateStatus
  commits(last:1){nodes{commit{statusCheckRollup{state}}}}
  reviews(last:5){nodes{author{login} state body submittedAt url}}
  comments(last:5){nodes{author{login} body createdAt url}}
  reviewThreads(first:100){nodes{isResolved isOutdated path line
    comments(last:1){totalCount nodes{author{login} body url}}}}}}}' \
| jq -r '.data.repository.pullRequest as $p
  | def short: gsub("<!--[\\s\\S]*?-->";"") | gsub("<[^>]+>";"") | gsub("\\s+";" ") | ltrimstr(" ") | .[0:200];
  "PR:      \($p.url)\nhead:    \($p.headRefOid[0:7])\nchecks:  \($p.commits.nodes[0].commit.statusCheckRollup.state // "NONE")\nreview:  \($p.reviewDecision // "NONE")\nmerge:   \($p.mergeStateStatus)",
  "latest reviews and comments:",
  ([($p.reviews.nodes[] | select(.body != "") | {at: .submittedAt, who: .author.login, what: .state, body, url}),
    ($p.comments.nodes[] | {at: .createdAt, who: .author.login, what: "COMMENT", body, url})]
   | sort_by(.at) | .[-5:][] | "- \(.at[0:16]) [\(.who)] \(.what): \(.body | short)\n  \(.url)"),
  "unresolved threads: \([$p.reviewThreads.nodes[] | select(.isResolved|not)] | length)",
  ($p.reviewThreads.nodes[] | select(.isResolved|not) | .comments as $c
    | "- \(.path):\(.line // "?")\(if .isOutdated then " (outdated)" else "" end) [\($c.nodes[0].author.login)] (\($c.totalCount) comments, latest shown) \($c.nodes[0].body | short)\n  \($c.nodes[0].url)")'
```

**`scripts/pr-evidence.sh`.** GitHub has no API for the image upload you get by dragging into a PR. This script
uploads the files to a `pr-evidence` branch that is never merged, and prints the links. Check once that the images
show on the first real PR.

```bash
#!/usr/bin/env bash
# Uploads screenshots or videos for a pull request to the `pr-evidence` branch and prints Markdown to put
# in the pull request. That branch is never merged, so evidence stays out of the code and its history.
# Usage: scripts/pr-evidence.sh <pr-number> <file>...
set -euo pipefail
n="$1"; shift
repo="${GH_REPO:-$(gh repo view --json nameWithOwner -q .nameWithOwner)}"
branch=pr-evidence
stamp="$(date +%Y%m%d-%H%M%S)"
index="$(mktemp -d)/index"
trap 'rm -rf "$(dirname "$index")"' EXIT

upload() {
  git fetch -q origin "refs/heads/$branch:refs/remotes/origin/$branch" 2>/dev/null || true
  local parent tree commit
  parent="$(git rev-parse -q --verify "refs/remotes/origin/$branch" || true)"
  rm -f "$index"
  [ -n "$parent" ] && GIT_INDEX_FILE="$index" git read-tree "$parent"
  for f in "$@"; do
    GIT_INDEX_FILE="$index" git update-index --add \
      --cacheinfo "100644,$(git hash-object -w --no-filters "$f"),pr-$n/$stamp-$(basename "$f")"
  done
  tree="$(GIT_INDEX_FILE="$index" git write-tree)"
  commit="$(git commit-tree "$tree" ${parent:+-p "$parent"} -m "Evidence for #$n")"
  git push -q origin "$commit:refs/heads/$branch"
}

# Several agents may upload at once: on a rejected push, fetch and try again.
for attempt in 1 2 3; do upload "$@" && break; [ "$attempt" = 3 ] && exit 1; sleep $((attempt * 2)); done

for f in "$@"; do
  url="https://github.com/$repo/blob/$branch/pr-$n/$stamp-$(basename "$f")?raw=true"
  case "$f" in
    *.png|*.jpg|*.jpeg|*.gif|*.webp) echo "![$(basename "$f")]($url)" ;;
    *) echo "[$(basename "$f")]($url)" ;;
  esac
done
```

---

## Keeping it alive

- When agents make the same mistake twice, turn it into a lint rule or a test. Write it in `AGENTS.md` only if
  neither works. Delete a rule once a tool enforces it.
- Change `AGENTS.md` in the same PR as the code it describes. Contradicting instructions are worse than none.
- Every bug that reaches `main` gets a test that fails without the fix. A performance bug gets a budget too.
- When PR runs creep past the CI target, take the next step in Part 4.
