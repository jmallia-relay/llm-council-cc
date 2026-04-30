---
name: council
description: Run a multi-LLM council with separation of powers — three independent cold-context judges (Gemini, Codex, Claude) critique an idea, draft, plan, decision, or output. Optionally with a Claude Opus drafter producing the artifact first.
arguments:
  - name: subagent
    description: "Subagent type: assessor, planner, reviewer, red-team, researcher, architect, implementer, test-designer, shipper, router"
    required: true
  - name: task
    description: Task description in quotes (or a question/decision/artifact to critique)
    required: true
---

Run the LLM Council with separation of powers: three independent cold-context judges grade an artifact. Disagreement between judges IS the signal — surface it, don't smooth it over.

## When to use this

**Primary use: critical analysis of ideas, drafts, plans, and outputs.** Examples:
- "Critique this Q2 strategy doc" (artifact in `--files`)
- "Should we ship the hero email capture or revert?" (decision)
- "Pressure-test this experiment hypothesis"
- "Is this Slack draft tonally right for a CRO update?"
- "Does this RICE prioritization make sense?"

**Secondary use: code review and design** (the original use case for the underlying CLI). Same flow, the schemas just happen to fit code well.

## Two modes

- **Mode A — Council drafts and judges (default).** User asks a question with no artifact. Step 1 (Gemini drafter) produces the artifact, Step 2 grades it.
- **Mode B — Council judges user's artifact.** User passed `--files <path>` or pasted an artifact in the task. **Skip Step 1** — there is nothing to draft. Go straight to Step 2 with the user's artifact as the artifact-under-review. This is the most common use case.

Decide which mode applies before starting. Heuristic: if the task includes words like "critique", "grade", "review", "pressure-test", "is this X", or the user passed `--files`, use Mode B. If the task asks for something to be produced, planned, or decided fresh, use Mode A.

## Roster

**Drafter:**
- **Claude Opus** via the `council-drafter` sub-agent — fresh sub-agent, isolated context, uses Claude Code's auth (no extra API key).

**Judges (parallel, cold context, each blind to the others and to the drafter's reasoning):**
- **Gemini-judge** — separate `council run critic` process via the `the-llm-council` CLI. Different model family from drafter; fully independent.
- **Codex** — invoked via the `codex-plugin-cc` companion script at `~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs`. Managed-session invocation — different code path than `council run --providers codex`, which historically returned empty responses. Different model family from drafter; fully independent.
- **Claude-judge** — fresh Claude Opus sub-agent via the `council-judge` agent (read-only tools, clean context). Same model family as drafter, but a *separate sub-agent entity* — see below.

**Same family, separate entities, no conflict of interest.** The drafter and Claude-judge are both Claude Opus, but they run as *independent sub-agent dispatches* with no shared session, no shared context, no shared tools beyond read-only file access. Cold context is the firewall. The judge sees the artifact and the original task — never the drafter's reasoning, prompts, or partial outputs. Bias from same-family alignment is real but small once the reasoning channel is severed. Gemini-judge and Codex provide the other-family cross-checks.

The current Claude Code session — the **orchestrator** — does not draft and does not judge. It only dispatches sub-agents and synthesizes the verdict. Three Claude entities are involved (orchestrator, drafter, judge), all isolated from each other.

## Subagents

**For ideas / outputs / decisions (primary use):**
- `assessor` — any "should we A or B" decision, build/buy/ship/kill, tradeoff calls
- `planner` — strategy docs, roadmaps, OKR drafts, experiment plans
- `reviewer` — generic critique of a written artifact (Slack drafts, doc reviews, copy)
- `red-team` — adversarial pressure-test of a position, plan, or argument
- `researcher` — "what does the evidence say about X" / fact-check a claim

**For code (secondary use):**
- `architect` — system design, APIs
- `implementer` — feature code, bug fixes
- `test-designer` — test suite design
- `shipper` — release notes

**Meta:**
- `router` — pick the right subagent for an ambiguous task

The underlying `the-llm-council` CLI's `critic` schema is code-shaped. For non-code use, Step 2's Gemini-judge call may fail synthesis validation — that's expected; the orchestrator reads `.drafts.gemini` from the CLI output directly, where Gemini's actual judge verdict lives unconstrained by schema. (Step 1's drafter no longer touches this CLI; it goes through the `council-drafter` sub-agent instead.)

## Execution

You (the orchestrator) must run these steps in order. Do not skip.

### Step 1 — Drafter (Mode A only — skip in Mode B)

**If Mode B** (user provided an artifact via `--files` or pasted in the task): SKIP this step entirely. The artifact-under-review is already in hand — go to Step 2. Note in the user-facing output that drafting was skipped because the user provided the artifact directly.

**If Mode A** (no artifact yet, council needs to produce one):

Dispatch the `council-drafter` sub-agent via the Task tool (`subagent_type: "council-drafter"`). The sub-agent runs as a fresh Claude Opus instance with a clean context, isolated from your (orchestrator's) reasoning.

Pass the sub-agent:
1. The **role** (the user's `$subagent` argument: `assessor`, `planner`, `reviewer`, `red-team`, `researcher`, `architect`, `implementer`, `test-designer`, or `shipper`)
2. The **task** verbatim
3. **Reference materials** — if the user passed `--files`, read each file with the Read tool and inline the FULL contents into the dispatch prompt. Do not summarize; the drafter must see the same source material the judges will see in Step 2.

The sub-agent returns the artifact as plain text. Capture it for Step 2.

If the user passed context files, **also remember those file paths** — Step 2 judges must see the same files to verify factual claims.

**Drafter failure handling:** If the `council-drafter` sub-agent returns an error, an empty response, or refuses the task, abort and report the failure to the user — drafting is the precondition for Mode A; we cannot proceed to Step 2 without an artifact. Recommend the user retry or switch to Mode B by providing their own artifact.

### Step 2 — Three judges in parallel

Build the shared judge prompt, then dispatch all three judges in the **same turn** (parallel tool calls in a single message). Each judge sees ONLY the artifact and the original task — never the drafter's reasoning, never the other judges' verdicts.

**Shared judge prompt template:**

```
You are an impartial judge grading the following artifact. You have not seen
the reasoning that produced it. Grade it cold.

Original task: $task

[REFERENCE MATERIALS — only if drafter received --files. Paste the FULL contents
of each file here, verbatim. Do not summarize. Judges must verify claims against
the same source material the drafter saw.]

Artifact (paste verbatim from Step 1's `council-drafter` sub-agent output — full text, no abbreviation):
<paste artifact here>

Return JSON only, with these keys:
- overall_score (integer 0-10)
- verdict ("ship" | "revise" | "reject")
- top_concern (one sentence — the single most important issue)
- strengths (array of strings)
- weaknesses (array of strings)

Do not wrap in markdown fences. Do not add commentary outside the JSON.
```

**CRITICAL:** Paste artifact and reference files VERBATIM. Summarizing biases the judges — they will fact-check against your summary instead of the real source, and judges who would have caught a real issue will instead "correct" you for hallucinating something that's actually in the file.

**ALSO CRITICAL:** Do not add your own framing, opinions, or "factual context notes" to any judge call (including the Claude-judge sub-agent). The orchestrator must not anchor the judges. If you have factual context, the judges can find it in the reference materials themselves.

Save this prompt to a temp file once (e.g. `/tmp/council-judge-prompt-$$.txt`) so all three judges receive identical input.

**Judge A — Gemini-judge** (Bash, fresh council process):

```bash
council run critic "$task" --providers gemini --json --input /tmp/council-judge-prompt-$$.txt
```

This is a separate council process from Step 1, so context does not leak.

**Judge B — Codex** (Bash, plugin companion script):

```bash
PLUGIN=$(ls -d "$HOME"/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs 2>/dev/null | sort -V | tail -1)
if [ -n "$PLUGIN" ]; then
  cat /tmp/council-judge-prompt-$$.txt | node "$PLUGIN" task
else
  echo '{"verdict":"unavailable","top_concern":"codex-plugin-cc not installed"}'
fi
```

The plugin's `task` mode uses managed Codex sessions — does not have the empty-response failure mode that `council run --providers codex` had.

**Judge C — Claude-judge** (Task tool, `council-judge` sub-agent):

Launch the `council-judge` sub-agent with the contents of the shared judge prompt. The sub-agent has its own system prompt rubric and returns JSON.

### Step 3 — Three-judge verdict

Parse each judge's JSON output (best-effort — strip code fences if present, fall back to "unavailable" if unparseable). Render the council output:

1. **Synthesized artifact** (from Step 1) — full text in scrollback for reference, not in the summary
2. **Judge table** — three rows (Gemini, Codex, Claude). Columns: overall score, verdict, top concern. Mark unavailable judges explicitly (e.g. `Codex: unavailable — empty response`).
3. **Divergence** — one or two sentences calling out where the judges disagree most. Disagreement IS the signal:
   - **3 judges, all converge** → strong signal in the consensus direction
   - **3 judges, 2 of 3 converge** → note the dissenter's concern, lean toward consensus
   - **3 judges, all diverge** → recommend revise, surface all three top concerns
   - **2 judges available, agree** → moderate signal in the consensus direction; flag the missing third
   - **2 judges available, split 1-1** → **no signal**; explicitly recommend re-run or escalate to user judgment, do NOT pick one verdict over the other
   - **1 judge available** → fallback only; report the single verdict but flag low confidence
4. **Recommendation** — your one-line take. Be honest if split. Do not paper over disagreement. If the council has degraded to 1-1 with no signal, say so explicitly — that is the correct output, not a forced verdict.

Keep the user-facing summary under ~30 lines. Raw artifact and individual judge JSON go in scrollback for reference.

### Cleanup

```bash
rm -f /tmp/council-judge-prompt-$$.txt
```

## Failure modes

- **Codex returns nothing or errors**: mark "Codex: unavailable" in the table and proceed with two judges. Do not block.
- **Plugin glob returns nothing**: codex plugin not installed — same fallback (mark unavailable, proceed).
- **Gemini critic call fails**: mark "Gemini-judge: unavailable" and proceed with available judges.
- **Claude-judge sub-agent unavailable**: extremely unlikely; if it happens, return whatever judges did respond and flag the gap.
- **All three judges fail**: surface the artifact alone, no verdict; tell the user the council failed and to re-run.

The council never blocks on a single judge. Two judges is still useful signal. One judge is a fallback. Zero judges means re-run.
