---
name: council
description: Run a multi-LLM council with separation of powers — three independent cold-context judges (Gemini, Codex, Claude) critique an idea, draft, plan, decision, or output. Optionally with a Gemini drafter producing the artifact first.
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
- **Gemini** (via `GOOGLE_API_KEY`) — produces the artifact

**Judges (parallel, cold context, each blind to the others and to the drafter's reasoning):**
- **Gemini-judge** — separate `council run critic` process. Same model family as drafter, but a fresh context that sees only the artifact. The drafter does not get to judge its own work.
- **Codex** — invoked via the `codex-plugin-cc` companion script at `~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs`. This is a managed-session invocation — different code path than `council run --providers codex`, which historically returned empty responses.
- **Claude-judge** — fresh Claude sub-agent via the `council-judge` agent (read-only, clean context).

Claude is NOT a council member. Claude is the **orchestrator**. The Claude-judge sub-agent is Claude's peer voice with a clean context window so it can grade without anchoring to the orchestrator's reasoning.

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

The underlying council CLI's `critic`/`architect`/etc. JSON schemas are code-shaped. For non-code use, this causes synthesis to fail validation — that's expected; Step 1 instructs the orchestrator to read `.drafts.gemini` directly, where Gemini's actual output lives unconstrained by schema.

## Execution

You (the orchestrator) must run these steps in order. Do not skip.

### Step 1 — Drafter (Mode A only — skip in Mode B)

**If Mode B** (user provided an artifact via `--files` or pasted in the task): SKIP this step entirely. The artifact-under-review is already in hand — go to Step 2. Note in the user-facing output that drafting was skipped because the user provided the artifact directly.

**If Mode A** (no artifact yet, council needs to produce one):

```bash
council run $subagent "$task" --providers gemini --json
```

Capture the full JSON output. **Extract the artifact from `.drafts.gemini`** — not `.output` or `.synthesis`. The council CLI's synthesis pass enforces a per-subagent schema (`architect`, `critic`, etc.) that often does not match what Gemini actually produces, causing `success: false` with `validation_errors`. The DRAFT under `.drafts.gemini` is always present and contains the substantive content. The synthesis failure is expected and should be ignored.

If the user passed context files to the drafter (via `--files`), **remember those file paths** — Step 2 judges must see the same files to verify factual claims.

**Discard** Gemini's self-critique from this call — we want cold judges in Step 2, not the drafter's own grade.

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

Artifact (paste verbatim from Step 1's `.drafts.gemini` — full text, no abbreviation):
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
