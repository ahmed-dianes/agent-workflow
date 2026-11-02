# Agent workflow

One Markdown file that sets up an agent-first development pipeline in a repo, from a new branch to a merge on `main`
and a deploy to production.

Give [`AGENT-WORKFLOW.md`](AGENT-WORKFLOW.md) to a coding agent (Claude Code, Codex, …) and say:

> Set up this workflow in this repo.

It reads the repo, sets everything up locally on a new branch, and asks before it pushes or touches GitHub settings
or secrets. In a repo that already has it, say *"Onboard me to this repo"* instead.

## What it sets up

- An `AGENTS.md` written for this repo: how an agent could do real damage here, every surface a change has to
  reach, the commands, and how work flows.
- One worktree per task, with heavy checks sent to a remote VM so agents never queue for your machine.
- CI with one required check (`ci-ok`), kept fast with the steps in Part 4: run only what a change affects, cache
  test results, split tests by measured time.
- An AI review on every PR by a different model family than the one that wrote the code.
- A deterministic performance budget that fails CI when a key path gets more expensive.
- PR tools: one status script, plus screenshot uploads for UI changes.
- Swarm mode: agents merge into a feature branch on their own, and a human approves only the final PR into `main`.
- Environments: `main` deploys to staging, and a human promotes a tested commit to production.

## Before you send it

Edit the **Team decisions** table at the top. It holds the choices that differ between teams: review rules, the CI
runner (boat.dev in this copy), the database (Supabase), and how production is promoted.

## Background

The pipeline is modeled on [pingdotgg/t3code](https://github.com/pingdotgg/t3code), whose CI and review flow handle
a high volume of agent-written PRs. [`research/`](research) has the notes behind it, written by AI agents in
October 2026, with links to their sources.
