# Continuous Claude v4.7 — Plain-English Usage Guide

This is the "what do I actually type" doc. No jargon, no orchestration diagrams — just what to run and what to expect.

## One-time setup — already done

`~/.cargo/bin` and `~/.local/bin` need to be on your shell PATH so the four tools below resolve in every terminal. If a fresh terminal can't find `bloks`, `tldr`, or `ouros`, append this one line to `~/.bashrc` (and `~/.zshrc` if you use zsh):

```
. "$HOME/.cargo/env"
```

Check everything's wired:

```bash
bloks --version
tldr --version
ouros --help
fastedit --help
```

If any are missing, that shell hasn't sourced `~/.cargo/env` yet.

## The four external tools (what each does, in one sentence)

| Tool | In human terms |
|------|----------------|
| `bloks` | A little notebook of "what I learned about library X" — short cards Claude can read instead of re-researching every time. |
| `tldr` | A smarter `grep` — shows structure (who calls what, where the dead code is, where the risk spots are) instead of dumping whole files. |
| `ouros` | A sandboxed Python scratchpad where research workers can pile up data without polluting Claude's context window. |
| `fastedit` | A local 1.7B model that applies edits ~10× cheaper than the default Edit tool. Optional — run `fastedit pull` once to download the model. |

You don't usually invoke these yourself; the v4.7 skills call them for you.

## The daily flow

### 1. Start in any project

```
/bootup
```

It'll ask three questions (new vs. existing? language? what do you want to do?), score the project's "readiness" (27 checks: linter, formatter, tests, lockfile, CODEOWNERS, dead code, complexity, etc.), auto-fix what it can, then route you to one of the three main modes.

### 2. Pick a mode

| You're saying… | Run | What happens |
|----------------|-----|--------------|
| "I have a specific thing to build." | `/autonomous "build X that does Y"` | Full pipeline: assess → plan → premortem → execute → validate → evolve. Workers do the coding; you approve milestones. |
| "I don't know what to build yet — help me figure it out." | `/research "question or topic"` | One pass through the sandbox, returns findings. Fast. |
| "I'm researching something deep — keep iterating." | `/autonomous-research "broad question"` | Loops: hypothesis → test → refine. Stops when it's confident or you stop it. |
| "I'm about to merge — sanity check my diff." | `/review` (or `/review --base-ref main`) | Runs bugbot + structural analysis + semantic review in parallel. |
| "Before I code this plan, poke holes in it." | `/premortem` | First-principles risk analysis: tigers (real threats), paper tigers (look scary, manageable), elephants (things everyone avoids discussing). |

### 3. When the session is getting long

You don't have to remember — v4.7's hooks do this automatically:

- Around 85 % of your context budget, the harness blocks and tells you to run `/create-handoff`.
- Before any automatic compaction, it snapshots your session state into `thoughts/shared/handoffs/<date>_<slug>.yaml`.
- Next session: run `/resume-handoff` and it picks up where you left off with a fresh context window.

### 4. Done

Commit as you normally would. The hooks will have already run your type-checker and linter after every edit, so surprises at commit time are rare.

## What lives where

```
ContinuousClaudeV4.7/
├── CLAUDE.md             ← project-level instructions (auto-loaded by Claude Code)
├── .claude/
│   ├── CLAUDE.md         ← detail doc for skills, hooks, sandbox, editing
│   ├── skills/           ← 9 slash commands (/autonomous, /research, /review, etc.)
│   ├── agents/           ← worker + oracle sub-agent definitions
│   ├── hooks/            ← 5 .mjs hooks (context tracking, diagnostics, handoffs)
│   └── settings.json     ← hook wiring + env config
├── scripts/
│   ├── readiness.sh      ← the 27-criterion health check `/bootup` runs
│   └── readiness-fix.sh  ← auto-remediation for the obvious gaps
├── tools/
│   ├── ouros_harness.py  ← sandbox Python REPL bridge
│   ├── exa_search.py     ← web search (needs EXA_API_KEY)
│   └── nia_docs.py       ← docs search (needs NIA_API_KEY)
└── .venv/                ← created during install; holds aiohttp for the bridges
```

## The two env keys you might want

Copy `.env.example` to `.env` and paste your keys:

```
EXA_API_KEY=...   # https://exa.ai — web search
NIA_API_KEY=...   # https://nia.ai — documentation + indexed code search
```

Without them, Ouros still works; it just can't do web lookups or docs search during research.

## "What happens under the hood?" — the short version

1. You type `/autonomous "build a login form"`.
2. A foreman (the skill) reads your request, assesses the project, and writes a plan with milestones.
3. For each milestone, the foreman spawns one or more worker agents. Each worker is a sub-process with its own context window — it reads only the files it needs, writes code, runs tests, and returns a short summary.
4. After every edit a worker makes, a hook silently runs your type-checker and linter, so problems surface immediately instead of at review time.
5. When the milestone finishes, the foreman validates against the plan's "definition of done" (written before any code was written, like a TDD contract).
6. An "evolve" step distills learnings: if workers kept hitting the same rake, the system prefers adding a lint rule or type check (real enforcement) over scribbling a note in CLAUDE.md.
7. Knowledge that's worth keeping becomes a `bloks` card, so next time you work with that library or pattern, the context is ready.

You rarely need to know any of that — just `/bootup` → pick a mode → approve or course-correct → commit.

## What got cleaned up on 2026-04-22

The old Continuous Claude v3 install was ripped out of `~/.claude/` and the separate `/Users/dannydasilva/Continuous-Claude-v3/` tree. Everything moved (not deleted — it's all recoverable) to:

```
~/.claude/backups/v3-cleanup-2026-04-22-124700/
```

What moved:

| Bucket | Count | Notes |
|--------|-------|-------|
| `hooks/` | 56 files | All v3 hooks. Only `persist-project-dir.sh` was kept in `~/.claude/hooks/`. |
| `plugins/braintrust-tracing` | 1 plugin | v3 observability. |
| `skills/` | ~57 directories + `archive/` | `continuity_ledger`, `recall`, `remember`, all 7 `agentica-*`, both `braintrust-*`, `implement_plan*`, `create_handoff`, `resume_handoff`, `plan-agent`, `validate-agent`, `workflow-router`, `task-list-swarm`, `tdd-migrate`, `tdd-migration-pipeline`, parallel-agent patterns, memory/orchestration patterns, `search-hierarchy`, `search-router`, `research-external`, `premortem`, `system_overview`, `tour`, `mot`, `onboard`, `opc-architecture`, `compound-learnings`, `help`, etc. |
| `rules/` | 5 rule files | `agent-memory-recall`, `cross-terminal-db`, `dynamic-recall`, `proactive-memory-disclosure`, `continuity`. Also trimmed an OPC reference out of `proactive-delegation.md`. |
| `scripts/` | 11 items | `core/`, `artifact_*`, `braintrust_analyze.py`, `*-reasoning.sh`, `init-project.sh`, `status.sh`. |
| `Continuous-Claude-v3/` | 988 MB | The whole v3 source tree. |
| `settings.json.before` | 1 file | Snapshot of your pre-cleanup settings. |

What stayed:

- Everything in `~/.claude/agents/` (56 agent personas — generic, re-usable).
- `~/.claude/scripts/statusline/` (your status line) and all the MCP tool wrappers (`morph_*`, `nia_docs.py`, `perplexity_search.py`, etc.).
- Generic rule files (claim-verification, destructive-commands, hooks, no-haiku, observe-before-editing, etc.).
- Generic skills (`commit`, `fix`, `build`, `refactor`, `debug`, `explore`, `review`, `test`, `tdd`, `security`, `qlty-*`, `tldr-*`, `morph-*`, `ast-grep-find`, `perplexity-search`, `nia-docs`, `firecrawl-scrape`, `github-search`, `repoprompt`, `rp-explorer`, math stack, etc.).
- `~/.claude/plugins/` (minus `braintrust-tracing`) — LSPs, Figma, Playwright, feature-dev, code-simplifier, frontend-design.

### Rolling back the cleanup

```bash
# restore everything in one go (be careful — it brings back the old hooks!)
BK=~/.claude/backups/v3-cleanup-2026-04-22-124700
cp "$BK/settings.json.before" ~/.claude/settings.json
mv "$BK/hooks/"* ~/.claude/hooks/
mv "$BK/skills/"* ~/.claude/skills/
mv "$BK/rules/"* ~/.claude/rules/
mv "$BK/scripts/"* ~/.claude/scripts/
mv "$BK/plugins/braintrust-tracing" ~/.claude/plugins/
mv "$BK/Continuous-Claude-v3" ~/Continuous-Claude-v3
```

### Deleting the backup (once you're sure everything works)

```bash
rm -rf ~/.claude/backups/v3-cleanup-2026-04-22-124700  # frees 1 GB
```

Don't do this until you've restarted Claude Code at least once and confirmed normal operation.

## About this session's error noise

While the cleanup was running, you saw a few errors like `Failed to spawn: hook_launcher.py`. Those are harmless log lines from the *currently running* Claude Code session — it cached the old hook wiring at startup and is still trying to call the removed files. The on-disk `~/.claude/settings.json` is already clean. **On your next Claude Code restart, the errors stop.**

A tiny no-op stub was left at `~/.claude/hooks/hook_launcher.py` to quiet the noisiest of them. You can delete that stub anytime after restarting.

## Troubleshooting quick-reference

| Problem | Check |
|---------|-------|
| `bloks`/`tldr`/`ouros` not found in a fresh terminal | Add `. "$HOME/.cargo/env"` to `~/.bashrc` |
| `fastedit pull` complains | It's downloading ~1.7 GB; make sure you have disk space + network |
| Ouros research can't do web searches | You haven't set `EXA_API_KEY` / `NIA_API_KEY` in `.env` |
| `/review` complains "tldr command not found" | `~/.cargo/bin` not on PATH for that shell |
| Hooks still seem to fire old v3 code | You haven't restarted Claude Code since the cleanup |
| Node hook throws in the project | `node --check ContinuousClaudeV4.7/.claude/hooks/<name>.mjs` to see the syntax error |

## One more thing — fastedit MCP tools (optional)

You currently have the `[mlx]` extra installed. If you want the `fast_edit`, `fast_read`, `fast_search`, etc. tools exposed as MCP in Claude Code, reinstall with the `[mcp]` extra too:

```bash
uv tool install --force \
  --with-editable /Users/dannydasilva/Documents/personal/parcadei/fastedit \
  "/Users/dannydasilva/Documents/personal/parcadei/fastedit[mlx,mcp]"
```

Then add to `~/.claude.json`:

```json
"mcpServers": {
  "fastedit": {
    "command": "python3",
    "args": ["-m", "fastedit.mcp_server"]
  }
}
```

Without this step, `fastedit-hook` still runs (it's on PATH and wired), but the MCP tools aren't exposed.

---

That's the whole picture. `/bootup` is always the right first step in a new project.
