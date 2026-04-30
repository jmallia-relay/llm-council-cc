# llm-council-cc

A `/council` slash command for Claude Code that runs a multi-LLM council with strict separation of powers. Three independent cold-context judges (Gemini, Codex, Claude) grade an artifact in parallel. **Disagreement between judges is the signal** — the orchestrator surfaces it rather than smoothing it over.

## What it does

You ask `/council assessor "should we ship X or revert?"` and you get three independent verdicts on the question. Different model families, different code paths, different reasoning styles — converging or diverging in ways you can act on.

Two modes:
- **Mode A** — council drafts AND grades. You ask a question, Gemini drafts an answer, three judges critique it.
- **Mode B** — council grades YOUR artifact. You pass `--files <path>` or paste an artifact, Step 1 is skipped, three judges critique what you provided directly.

Designed for **critical analysis of ideas, drafts, plans, decisions, and outputs** — strategy docs, experiment hypotheses, build/buy calls, copy reviews, RICE prioritizations, tradeoff calls. Code review works too (the underlying CLI was built for it), but it's not the primary use.

## Architecture

- **Drafter** (Mode A only): Gemini via `the-llm-council` CLI
- **Judge A**: Gemini-critic — separate `council run critic` process, fresh context, sees only the artifact
- **Judge B**: Codex — invoked via the `codex-plugin-cc` companion script (managed Codex sessions, not the brittle `council run --providers codex` path)
- **Judge C**: Claude-judge — fresh Claude sub-agent (`council-judge`) with read-only tools and clean context

Claude is the **orchestrator**, not a judge. The Claude-judge sub-agent is Claude's peer voice with a clean window so it can grade without anchoring to the orchestrator's reasoning.

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
/plugin marketplace add jmallia-relay/llm-council-cc
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

## Why Gemini judges Gemini

Self-preference bias is real — same-family judging is correlated with the drafter. The mitigation is **cold context**: Gemini-critic is a separate process from the Gemini drafter, sees only the artifact, has no access to the drafter's reasoning. The bias isn't zero, but it's small enough that the cross-check is still useful, and adding a fourth provider to fix it would double the auth surface.

If you want stronger cross-model independence, the cleanest path is to swap Gemini-critic for an Anthropic API or OpenAI API direct call — but that requires another API key.

## Limitations

- **Synthesis schemas are code-shaped.** The underlying `the-llm-council` CLI enforces per-subagent JSON schemas (`architect`, `critic`, etc.) tuned for code review. For non-code tasks, synthesis fails validation. The orchestrator works around this by reading `.drafts.gemini` directly. Expected behavior, not a bug.
- **Drafter call is slow** (~60–120s for Gemini synthesis with retries).
- **Codex via plugin is robust but adds latency** (~30s for cold session start). Worth it for reliability.
- **Personal-tool ergonomics.** This was built for one user. Sharing across a team works (this repo) but each user runs their own auth + own API keys.

## License

MIT — see [LICENSE](./LICENSE).
