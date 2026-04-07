# Design Decisions — Pending Review
**Created:** 2026-04-04  
**Status:** Awaiting owner decisions before implementation  
**Context:** Session reviewed `results_r_ts_20260404_070127.jsonl`, uncovered bugs and design gaps, proposed 4 fixes

---

## Session Summary (What Happened)

### Bugs Found and Fixed

**Bug 1 — Trace data loss (FIXED, not yet re-run)**  
`react_agent.py` was returning only the **last** assistant message as `final_answer`. All intermediate messages — including reflection blocks, reasoning between tool calls, and self-critique written mid-loop — were silently discarded.

- **Impact:** 49% of records in `results_r_ts_20260404_070127.jsonl` have `reflection: null` and `self_critique: null` even though the LLM likely wrote them in intermediate iterations
- **Fix applied:** `react_agent.py` now collects a `trace_parts` list across all iterations. `final_answer` = full concatenated trace with `[TOOL CALL]` / `[TOOL RESULT]` markers between segments
- **Files changed:** `agent/react_agent.py`

**Bug 2 — Extraction picked first match instead of last (FIXED)**  
`extract_prob_yes()`, `extract_selected_option()`, `extract_confidence_legacy()` used `re.search()` which returns the **first** match. With the new full trace, the first `P(Yes):` might appear in a tool result or intermediate reflection — not the final answer.

- **Impact:** Would have caused wrong probability extraction for new-format traces
- **Fix applied:** All three functions now use `findall()` + take the last match
- **Files changed:** `agent/evals.py`

**HTML viewer updated:**  
`eval/results_to_html.py` now parses `[TOOL CALL]`/`[TOOL RESULT]` markers into collapsible color-coded sections. Backward-compatible with old single-message format.

### Key Insight: What "Reflection" Actually Means in Current Code

The difference between "ReAct" and "ReAct + Reflection" is **only the system prompt text**. The code path (`react_agent.py`) is identical. No messages are injected, no tool calls are counted, no reflection is forced. The LLM is simply given a different prompt that says "reflect every 3 tool calls" and "write self-critique before concluding."

Whether the LLM actually follows this instruction is up to the LLM. In the `results_r_ts_20260404_070127.jsonl` run (which used the reflection prompt), 49% of records had no reflection in the captured output — partly due to Bug 1 (intermediate messages lost), and partly because the LLM sometimes doesn't comply.

---

## Decision 1 — Cross-Timepoint Memory (Sequential Mode)

### Current behavior
Each timepoint (t1, t2, ..., t5) for a question starts completely fresh. The agent at t4 has **no knowledge** of what it concluded at t1, t2, t3. It re-discovers evidence from scratch each time.

Note: The **Reddit database** at t4 naturally contains all content from t1–t3 (later cutoff_time = superset of earlier data). What's missing is the agent's own prior conclusions.

### Proposed change
Add a `--sequential` flag. When enabled, the agent at t4 receives a compressed summary of t1–t3:

```
Previous forecasts for this question:
  [2026-02-04] P(Yes)=0.30, Predicted=No, Tools=3
    Key evidence: No relevant Reddit posts found about DoorDash Q4 orders.
    Self-critique: Relying on absence of evidence.
  [2026-02-06] P(Yes)=0.25, Predicted=No, Tools=4
    Key evidence: Found 2024 Q1 report showing 620M orders, growth trend suggests >800M.
    Self-critique: Extrapolating from Q1 to Q4 is uncertain.
```

~100–150 tokens per prior timepoint. At t10, that's ~1,500 tokens of history — manageable within 32K context.

### Why this matters for the paper
- Mimics how a real forecaster works: update beliefs over time, don't start fresh every day
- Enables a new metric: **temporal learning curve** — does accuracy improve from t1 → tN?
- Enables measuring **belief revision** — does the agent appropriately update when new evidence appears?
- Creates a genuine paper novelty: no existing forecasting benchmark evaluates temporal belief revision in LLM agents

### Risks
- **Breaks statistical independence.** Each timepoint's accuracy depends on prior timepoints. Makes statistical analysis more complex (need paired tests, not i.i.d. assumptions)
- **Error propagation.** If t2 reaches a bad conclusion, t4 might inherit that error
- **Confounding.** If t4 is wrong, is it bad current reasoning or bad inherited history?

### Recommended approach
Run **both** modes as a controlled ablation:

| Condition | Description | What it tests |
|-----------|-------------|---------------|
| Independent (current, default) | Each timepoint fresh | Baseline forecasting ability |
| Sequential (`--sequential`) | Carry forward compressed summaries | Temporal belief revision |

The comparison itself is a paper contribution.

### Decision needed
- [ ] Add sequential mode? (Yes / No / Later)
- [ ] If yes: what to include in summary? (conclusion only / conclusion + evidence / conclusion + evidence + self-critique)

---

## Decision 2 — Consistent Final Reflection and Self-Critique

### Current behavior
- **ReAct (base):** No reflection or self-critique instructions
- **ReAct + Reflection:** Prompt says "reflect every 3 tool calls" + "write self-critique before final answer"
- **Zero-shot:** No reflection or self-critique

Result: only ~51% of ReAct+Reflection records have extractable reflection/self-critique (Bug 1 + LLM non-compliance combined). ReAct and ZS records never have it.

### Options

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A. Keep as-is** | Only ReAct+Reflection has reflection prompt | Clean ablation between modes | Inconsistent data, can't compare self-critique quality across modes |
| **B. Final-only everywhere** | Move self-critique + final reflection to base prompt (all modes get it) | Consistent data across all conditions; doesn't micro-manage the loop | Blurs the ablation — what exactly is being compared? |
| **C. Final-only in base, intermediate in Reflection addon** | Base prompt requires final Reflection + Self-critique. Reflection addon ALSO adds intermediate reflection every 3 tool calls | Clean ablation (intermediate reflection = the treatment variable). Consistent final output across all conditions | Slightly more complex prompt design |

### Recommended: Option C

This gives 3 clean experimental conditions:

| Condition | Intermediate reflection (every 3 tools) | Final reflection + self-critique |
|-----------|------------------------------------------|----------------------------------|
| Zero-shot | N/A (no tools) | Yes |
| ReAct | No | Yes |
| ReAct + Reflection | Yes | Yes |

The ablation variable is clear: **does intermediate structured reflection during evidence gathering improve final prediction quality?** All conditions produce the same final output format, making comparison straightforward.

### What "final reflection" means concretely
Before writing `Reasoning: / Final Answer: / Selected Option: / P(Yes):`, the LLM must write:

```
Reflection:
  Evidence for Yes: ...
  Evidence for No: ...
  Gaps / uncertainties: ...
  Current P(Yes): 0.X

Self-critique: <strongest counter-argument; note if confusing absence-of-evidence with evidence-of-absence>
```

This is a prompt instruction, not an injected message. The LLM may still not comply — but with the instruction in the base prompt, compliance should be higher.

### Decision needed
- [ ] Which option? (A / B / C)
- [ ] Should zero-shot also have final reflection? (Yes = richer data / No = cleaner baseline)

---

## Decision 3 — Expose All Search Capabilities

### Current behavior
The `search_database` tool is hardcoded to `engine="hybrid"` and does not expose the `Month` or `Authors` filters that the Text2SQL server supports.

### What's missing

| Server capability | Currently exposed? | Proposed |
|-------------------|--------------------|----------|
| `--vector` (pure pgvector semantic search) | No — hardcoded to hybrid | Add `engine` param: "hybrid" (default) or "vector" |
| `--hybrid` (semantic + full-text) | Yes (hardcoded) | Keep as default |
| `Month` filter (e.g., "2024-02") | No | Add `month` param |
| `Authors` filter (comma-separated) | No | Add `authors` param |

### Why expose them
- **Vector search:** Hybrid mode uses keyword matching which can miss semantically related but differently-worded content. Pure vector search may find relevant posts that keyword search misses. The LLM should be able to choose.
- **Month filter:** More precise than `start_time + cutoff_time` range. Useful when the LLM wants posts from a specific month.
- **Authors filter:** Allows "search for semantic content but only from this specific author." Currently the LLM can only do author lookup (SQL) OR semantic search — not both together.

### Risk
More parameters = more tokens in tool schemas (~150 extra tokens). Negligible impact on context.

### Impact on existing results
**None.** This only adds options. Old results were produced with hybrid-only and fewer filters. New results will have more flexibility. The two are still comparable because the LLM *can* choose to use only hybrid with no extra filters — it's additive.

### Decision needed
- [ ] Add all three? (Yes / No / Partial — specify which)

---

## Decision 4 — Information Gain Metric (Prior vs Posterior)

### The problem this solves
Current metrics (Brier score, accuracy, MCC) measure whether the agent got the right answer. They don't measure **whether retrieval actually helped.** If the agent's final P(Yes) = 0.3 and its zero-shot prior was also 0.3, then all the tool calls added nothing.

### Proposed approach
Before the ReAct loop, make one zero-shot LLM call to get the agent's **prior**:

```
Step 1: Zero-shot call → prior_prob_yes = 0.4
Step 2: Full ReAct loop with tools → agent_prob_yes = 0.2
Step 3: information_gain = |0.2 - 0.4| = 0.2 (retrieval shifted belief by 20pp)
```

Store `prior_prob_yes` in the result record. The eval suite can then compute:

- **Mean information gain:** How much does retrieval shift beliefs on average?
- **Helpful retrieval rate:** % of timepoints where retrieval moved probability toward ground truth
- **Harmful retrieval rate:** % where retrieval moved probability away from ground truth
- **No-effect rate:** % where retrieval didn't change the probability meaningfully

### Why this matters for the paper
This directly answers the core research question: *"Does giving the LLM a retrieval database improve calibration over zero-shot?"* Instead of comparing two separate runs (ZS vs ReAct), you measure the improvement **within each prediction** — a much more powerful paired comparison.

### Cost
One extra zero-shot LLM call per timepoint (~8.7s based on Run 3 latency). For 939 questions × 10 timepoints = ~22 GPU-hours extra. Significant but not prohibitive.

### Alternative: use existing ZS results as the prior
If you run ZS and ReAct on the **same questions and timepoints**, you can compute information gain by matching records. This avoids the extra LLM call but requires careful record matching and assumes the ZS run represents the agent's true prior (it does, since ZS = no tools).

### Decision needed
- [ ] Add prior measurement? (Yes — inline / Yes — use ZS run as proxy / No / Later)
- [ ] Add `--no_prior` flag to skip when not needed? (Yes / No)

---

## Priority Assessment

| Change | Impact on paper | Implementation effort | Risk | Suggested priority |
|--------|----------------|----------------------|------|-------------------|
| Fix 1 (trace capture) | **Critical** — without it, half the reflection data is lost | **Done** | Low (already implemented) | Already done |
| Fix 2 (last-match extraction) | **Critical** — prevents wrong data extraction | **Done** | Low (already implemented) | Already done |
| Decision 1 (sequential mode) | **High** — novel contribution, new metric | Medium (1-2 hours) | Medium (context window, error propagation) | Run after independent baseline |
| Decision 2 (consistent reflection) | **High** — enables clean cross-condition comparison | Low (prompt changes only) | Low | Do before next experiment run |
| Decision 3 (expose all tools) | **Medium** — more capability, but may not change results much | Low (30 min) | Very low | Do before next run |
| Decision 4 (information gain) | **High** — directly measures retrieval value | Low-Medium | Low | Can use ZS proxy initially |

### Suggested execution order (after decisions are made)
1. Decision 2 (prompt changes) + Decision 3 (tool exposure) — quick, do before any new runs
2. Re-run experiments with fixed trace capture + updated prompts (Runs 4, 5)
3. Decision 4 (information gain) — compute from Run 4 vs Run 3 comparison
4. Decision 1 (sequential mode) — implement and run as a separate ablation after baseline runs

---

## Files Changed This Session

| File | Change | Status |
|------|--------|--------|
| `agent/react_agent.py` | Full trace capture (trace_parts + [TOOL CALL]/[TOOL RESULT] markers) | Changed, not yet tested in live run |
| `agent/evals.py` | Last-match extraction for prob_yes, selected_option, confidence_legacy | Changed, unit-tested |
| `eval/results_to_html.py` | New trace-aware HTML viewer with collapsible tool/thinking sections | Created, tested on old results |

### Not yet changed (pending decisions above)
- `agent/prompts.py` — waiting on Decision 2
- `agent/tools.py` — waiting on Decision 3
- `agent/evals.py` (sequential mode) — waiting on Decision 1
