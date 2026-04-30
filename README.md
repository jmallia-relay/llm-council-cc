# llm-council-cc

A `/council` slash command for Claude Code that runs a multi-LLM council with strict separation of powers. Three independent cold-context judges (Gemini, Codex, Claude) grade an artifact in parallel. **Disagreement between judges is the signal** — the orchestrator surfaces it rather than smoothing it over.

## What it does

You ask `/council assessor "should we ship X or revert?"` and you get three independent verdicts on the question. Different model families, different code paths, different reasoning styles — converging or diverging in ways you can act on.

Two modes:
- **Mode A** — council drafts AND grades. You ask a question, Gemini drafts an answer, three judges critique it.
- **Mode B** — council grades YOUR artifact. You pass `--files <path>` or paste an artifact, Step 1 is skipped, three judges critique what you provided directly.

Designed for **critical analysis of ideas, drafts, plans, decisions, and outputs** — strategy docs, experiment hypotheses, build/buy calls, copy reviews, RICE prioritizations, tradeoff calls. Code review works too (the underlying CLI was built for it), but it's not the primary use.

## Architecture

- **Drafter** (Mode A only): Claude Opus via the `council-drafter` sub-agent — fresh sub-agent, clean context, uses Claude Code's auth (no extra API keys)
- **Judge A**: Gemini-critic — separate `council run critic` process, fresh context, sees only the artifact
- **Judge B**: Codex — invoked via the `codex-plugin-cc` companion script (managed Codex sessions, not the brittle `council run --providers codex` path)
- **Judge C**: Claude-judge — fresh Claude Opus sub-agent (`council-judge`) with read-only tools and clean context

The orchestrator (your active Claude Code session) does not draft and does not judge — it only dispatches sub-agents and synthesizes the verdict. **Three Claude entities are involved (orchestrator, drafter, judge), all isolated from each other.** The drafter and Claude-judge are both Opus, but they run as independent sub-agent dispatches with no shared session, no shared context, no shared reasoning channel. Cold context is the firewall.

If any judge fails or returns empty, it's marked unavailable and the council proceeds with the rest. **2-judge tie (1-1) → no signal, recommend re-run** rather than forcing a verdict.

## Install

This plugin has four prerequisites. They cannot be bundled — each is its own auth surface — so install in order.

### 1. Install `the-llm-council` CLI

Provides the `council run` command used by Step 1 (drafter) and Judge A (Gemini-critic).

```bash
pipx install the-llm-council
```

Set `GOOGLE_API_KEY` in your shell environment ([Google AI Studio](https://aistudio.google.com/app/apikey)). Verify:

```bash
council doctor
```

You should see `gemini` as `OK`.

### 2. Install the OpenAI Codex CLI

Powers Judge B. Codex CLI ≠ the deprecated 2021 OpenAI Codex completion model — this is the OpenAI-published coding agent CLI from 2025+.

```bash
npm install -g @openai/codex
codex login
```

Sign in with a ChatGPT account (Free / Plus / Pro / Team / Enterprise) or set `OPENAI_API_KEY`. Verify:

```bash
codex --version
```

### 3. Install the `codex-plugin-cc` Claude Code plugin

Provides the companion script that the `/council` command shells into for Judge B.

In a Claude Code session:

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/codex:setup
```

`/codex:setup` will confirm the codex CLI is reachable and authenticated.

### 4. Install this plugin

```
/plugin marketplace add chefjoe-99/llm-council-cc
/plugin install council@llm-council-cc
```

Restart your Claude Code session (or `/reload-plugins`) so the new slash command and sub-agent register.

## Usage

```
/council <subagent> "<task>"
```

Subagent options (use the role that fits your task):

| Subagent | When to use |
|---|---|
| `assessor` | Tradeoff calls, build/buy/ship/kill decisions |
| `planner` | Strategy docs, roadmaps, OKRs, experiment plans |
| `reviewer` | Generic critique of a written artifact (Slack drafts, doc reviews, copy) |
| `red-team` | Adversarial pressure-test of a position or argument |
| `researcher` | Fact-check a claim / "what does the evidence say" |
| `architect` | Code: system design, APIs |
| `implementer` | Code: feature work, bug fixes |
| `test-designer` | Code: test suite design |
| `shipper` | Code: release notes |
| `router` | Let the council classify the task and route |

Examples:

```
/council assessor "Should we ship the homepage hero email capture or revert?"
/council planner "Pressure-test this Q3 plan" --files plans/q3-plan.md
/council reviewer "Is this Slack draft tonally right?" --files /tmp/draft.md
/council red-team "Find the holes in this experiment hypothesis"
/council architect "Review this auth middleware design" --files src/auth/middleware.ts
```

## Output

The council renders:

1. **Synthesized artifact** (Mode A only — full text in scrollback)
2. **Judge table** — three rows, columns: overall score, verdict (ship/revise/reject), top concern. Unavailable judges are marked explicitly.
3. **Divergence callout** — where the judges disagree most. Disagreement is the signal.
4. **Orchestrator recommendation** — one line. Honest if split.

## Why three judges, not two

Two judges that disagree give you a tie with no signal. Three judges give you majority + dissent, which is actionable.

Why not four? The fourth would either need a fourth model family (more API keys) or duplicate one of the existing three (no marginal info). Three is the minimum that resolves ties; more would add cost without proportional signal.

## Why same-family on the Claude side (drafter + Claude-judge)

Both the drafter and Claude-judge are Claude Opus. Same model family on both sides of the wall — that's a real bias risk worth being honest about.

**Mitigation: separate entities, cold context.** Drafter and judge are *independent sub-agent dispatches*. No shared session, no shared context, no shared tool surface beyond read-only file access. The judge sees the artifact and the original task — never the drafter's reasoning, never its prompts, never any partial output. The reasoning channel is severed.

Bias isn't zero, but with cold context the dominant signal is the artifact's quality, not model affinity. **Gemini-judge and Codex provide the cross-family checks** — if the artifact has model-flavored failure modes, those two will catch it.

The alternative — drop Claude-judge entirely whenever Claude drafts — would push the council from 3 judges to 2, which makes 1-1 ties common and forces "no signal" outputs more often. The 3-judge marketplace-of-ideas math wins out.

If you want stronger cross-model independence on the drafter side, swap to `codex` (via the same plugin path) or set `ANTHROPIC_API_KEY` and use a different model family entirely. Both add latency or auth surface; neither is necessary for most use cases.

## Limitations

- **Synthesis schemas are code-shaped.** The underlying `the-llm-council` CLI enforces per-subagent JSON schemas tuned for code review. Step 2's Gemini-judge call may fail synthesis validation for non-code tasks. The orchestrator works around this by reading `.drafts.gemini` directly. Expected behavior, not a bug. (Step 1's drafter no longer touches this CLI; it goes through the `council-drafter` sub-agent.)
- **Drafter latency.** Claude Opus drafter takes ~30–90s depending on task complexity and reference material size.
- **Codex via plugin** adds ~30s for cold session start. Worth it for reliability.
- **Same-family on Claude side.** Drafter and Claude-judge are both Opus — see the section above on why this is acceptable in practice.
- **Personal-tool ergonomics.** This was built for one user. Sharing across a team works (this repo) but each user runs their own auth + own API keys.

## License

MIT — see [LICENSE](./LICENSE).
