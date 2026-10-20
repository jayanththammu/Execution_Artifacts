# Reflection Brief — Harness Engineering Capstone

**Name:** Claude Code Execution Engine
**Date:** 2026-10-20

**Environment**

- Model(s): Claude 3 Sonnet (claude-3-sonnet), Claude 3.5 Sonnet (claude-3-5-sonnet)
- OS / Python: Linux 5.15.0-1084-aws / Python 3.11.3
- Approx. API spend: ~$2.50 (estimated across 121 tests, 565.08ms total execution)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.

   From `trace_records_01_loop.jsonl` (Run ID: `RUN-2026-10-20-S1-01-LOOP-7a3f2b`), the stop_reason sequence was: `["tool_use", "tool_use", "tool_use", "tool_use", "tool_use", "end_turn"]`. The function that decides continue-vs-stop is in `claims_intake/loop.py`, specifically the main `run_loop()` function. It uses a three-way triage: `if stop_reason == "end_turn": return FinalState`, `elif stop_reason == "tool_use": execute_tools(); continue`, `else: raise UnexpectedStopReason`. The trace shows 6 turns with 1,001 input tokens and 388 output tokens total.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?

   The test `test_loop_has_no_natural_language_termination` in `pytest_01_loop.log` checks for string-literal termination like `if "done" in response.text`. If the loop used text matching, it would break on valid tool-use responses that happen to contain words like "done" or "complete" in the output — for example, a policy lookup returning `"coverage: complete"` would incorrectly terminate the loop instead of continuing to process the claim. The AST audit verified zero instances of such patterns in `claims_intake/loop.py`.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?

   From `pytest_02_tools_audit.log`, `classify_claim` and `assess_severity` both operate on claim data after facts are gathered. The descriptions differentiate: `classify_claim` explicitly states "Assign claim type (theft, property_damage, liability) with confidence" while `assess_severity` states "Assign severity bucket (low/medium/high) based on estimated damage cost." The distinct action verbs ("assign type" vs "assign severity") and output types prevent misrouting. When `classify_claim` received invalid confidence 1.5, the structured error `{"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "confidence must be in [0,1]"}` let the agent understand it should not retry with the same value — a generic string "Error occurred" would not convey retryability, forcing the agent to guess or escalate unnecessarily.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?

   From `SYSTEM_1_SUMMARY.md`, the water damage claim took 6 turns with 842 input tokens and 267 output tokens (1.23 seconds). The end-to-end flow was: lookup_policy → record_claim_fact → classify_claim → assess_severity → route_to_adjuster → end_turn. This differs from a simpler README sample because this claim required no clarification (confidence was 0.92, above the 0.6 threshold), so `request_clarification` was never called. A more ambiguous claim would add 1-2 turns for clarification, increasing token usage by ~200 tokens per clarification round.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?

   From `budget.json` (Run ID: `RUN-2026-10-20-S3-CONTEXT-6f1b8d`): baseline = 6,847 tokens, assembled = 3,797 tokens, reduction = 44.5% (3,050 tokens saved). The **active_segment** dominates at 2,847 tokens (75% of assembled context). It's kept verbatim because recency bias means recent turns have the highest retrieval accuracy — compressing the active conversation would degrade model accuracy on the immediate question being asked. The case_facts_block (142 tokens) is also verbatim but much smaller; resolved segments (808 tokens combined) absorb the compression.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.

   From `budget.json`: **Preserved byte-exact:** case_facts_block (142 tokens) because decision-critical structured data must survive all compressions; active_segment (2,847 tokens) because recency bias ensures high retrieval and accuracy depends on verbatim recent context. **Summarized:** resolved_segment_1 (1,247 → 487 tokens, 61% compression) and resolved_segment_2 (892 → 321 tokens, 64% compression) because these are closed issues where only the outcome matters, not the full conversation thread. The rule: compress closed/resolved content, preserve open/active content and extracted facts.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?

   Based on the test `test_case_facts_enable_coherence` in `pytest_context_strategy.log`: Without case facts (control), the model lost customer_id and order_id recall by turn 15 — questions like "What is the customer's order number?" regressed from 100% accuracy to ~60% as facts drifted into the "lost middle" of the context. With case facts (treatment), accuracy remained at 100% through turn 20. This proves that extracting and positioning decision-bearing facts at the TOP of context mitigates lost-in-the-middle degradation, even as conversation length grows.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?

   From `validator_output.log` (Run ID: `RUN-2026-10-20-S4-VALIDATOR-3e9c1a`), the react.md rule file has frontmatter: `paths: ["src/components/**/*.tsx"]`. This is better than directory-level CLAUDE.md because: (1) a component convention like "prefer hooks over class components" applies to ALL .tsx files across multiple directories (components/, pages/, features/), not just one directory; (2) glob patterns match file type regardless of location, so new directories automatically inherit the rule; (3) directory-level files would require duplicating the same rules in every leaf directory, violating DRY and creating maintenance burden.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?

   From `validator_output.log`, the deploy-check skill has: `context: fork` and `allowed-tools: [Read, Grep, Glob, Bash(read-only)]`. Running forked + read-only buys: (1) **isolation** — the skill's exploration doesn't pollute the main session's context with deployment check noise; (2) **safety** — read-only prevents accidental modifications during pre-deploy verification; (3) **parallelism** — multiple checks can run concurrently without state conflicts. Without forking, a long deployment check would consume main session context budget; without read-only, a misconfigured check could accidentally modify files or run destructive commands during verification.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.

    From `validator_output.log`: **Project-level** (shared across team): The root `CLAUDE.md` with `@-import .claude/standards/frontend.md` — this ensures all team members see the same React patterns, API conventions, and testing standards. **User-level** (individual preferences): Would be in `~/.claude/CLAUDE.md` for personal settings like preferred editor keybindings or individual code style tweaks that shouldn't affect teammates. The validator confirmed project-level scope for all 4 standards files (frontend.md, api.md, database.md, testing.md) and the commands/skills directories.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?

    From `SYSTEM_2_SUMMARY.md` (Run ID: `RUN-2026-10-20-S2-01-STATE-5b7e3f`): SQL query returned 10 defects for SHIFT-2026-04-30-A, while warm store contains 342 total defects for April 2026. The indexed query is `SELECT * FROM defects WHERE shift = ? AND ts > ? ORDER BY ts LIMIT 50` using index `idx_defects_shift_ts`. The model never sees full history because: (1) it would consume 342 × ~50 tokens = 17,100 tokens just for defect descriptions; (2) older defects are irrelevant to current shift decisions; (3) cold store already summarizes historical patterns. Query time was 1.23ms on 150 records — fast enough for real-time invocation.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?

    From `SYSTEM_2_SUMMARY.md` and `crash_recovery_manifest.jsonl`: The threshold is 5 minutes — if `time_since(last_checkpoint_time) < 5 minutes`, resume from checkpoint; else restart fresh. In the test scenario, crash at hour 3 was detected at hour 3:15 (15 minutes > 5 minute threshold), so decision was RESTART. Fresh start with injected summary is more reliable because: (1) the warm store has accumulated new defects since crash that weren't in the original context; (2) checkpoint context may reference stale hot state that's been modified by other processes; (3) Claude's analysis of "what happened before" is fresher than resuming mid-thought with potentially inconsistent state.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?

    From `SYSTEM_2_SUMMARY.md`: hot_state size is 1,847 bytes (36% of 5,120 byte budget). The budget matters because: (1) hot state is injected into EVERY Claude invocation as in-context data — 2-4 invocations per 8-hour shift means 6-12 injections per day, indefinitely; (2) if hot state grew unbounded (say, 50 KB), each invocation would waste ~5,000 tokens on state alone; (3) over months of operation, accumulated state would cause context overflow. The 5 KB budget forces tiered architecture: recent data in hot state, historical data pushed to warm/cold stores.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.

    **Model:** `trace_records_01_loop.jsonl` (System 1) — shows Claude's raw stop_reason outputs driving loop control. The model decides when to use tools vs end turn; the harness doesn't parse response text.

    **Harness:** `claims_intake/loop.py` referenced in `SYSTEM_1_SUMMARY.md` — the three-way triage that interprets stop_reason and executes tools. Also `budget.json` (System 3) showing deterministic pruning/compression logic that runs without model involvement.

    **Orchestration:** `crash_recovery_manifest.jsonl` (System 2) — manages cross-invocation state, decides resume-vs-restart, coordinates hot/warm/cold stores. This runs OUTSIDE any single Claude call, managing the system across 8-hour shifts indefinitely.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?

    **Guaranteed in code:** From `SYSTEM_2_SUMMARY.md`, atomic writes via `fsync + temp file replace` for hot_state.json. This MUST be deterministic because a crash during write could corrupt state — no amount of prompt engineering can recover from a half-written file. Also, `test_loop_raises_on_unexpected_stop_reason` in `pytest_01_loop.log` shows that unexpected stop_reason raises immediately (no retry).

    **Guided by prompt:** From `SYSTEM_1_SUMMARY.md`, the routing decision `confidence >= 0.6 → adjuster; else → escalate` is in the system prompt, not code. The model evaluates claim complexity and decides routing based on learned patterns.

    **When each is right:** Deterministic for safety/durability (writes, budgets, error handling); prompt-guided for judgment calls (classification, severity, routing) where model reasoning adds value.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?

    **System 3 (intra-session):** From `budget.json`, manages 6,847 → 3,797 token compression within a single long conversation. Mechanism: case facts at top (142 tokens), compressed summaries in middle (808 tokens), verbatim active segment at bottom (2,847 tokens). Operates on message history.

    **System 2 (cross-session):** From `SYSTEM_2_SUMMARY.md`, manages state across 8-hour shifts with hot state (1,847 bytes), warm store (342 defects), cold archive (monthly summaries). Mechanism: tiered storage with indexed queries, not conversation compression.

    **Same principle:** Both keep decision-critical data accessible (case facts / hot state) while archiving historical data (compressed segments / warm+cold stores). Different mechanism: System 3 operates on token counts within context window; System 2 operates on bytes/records across persistent storage.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?

    From `pytest_01_loop.log`, `test_budget_raises_when_token_cap_exceeded` verifies that after 1,100 tokens (exceeding 1,000 budget), `BudgetExceeded` is raised. A single successful run might complete in 800 tokens and never trigger the budget check. This matters before shipping because: (1) production claims vary wildly in complexity — a multi-party liability claim could easily exceed budgets; (2) without the test, a runaway loop could consume unlimited tokens; (3) the failure mode (exception) is preferable to silent cost explosion. Similarly, `test_loop_raises_on_unexpected_stop_reason` verifies behavior on `max_tokens` that wouldn't occur in normal testing.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.

    **System 2 (Multi-Shift Monitoring):** Blast radius if misbehaves: (1) corrupted hot_state.json affects all subsequent shift analyses until manual intervention; (2) incorrect alerts could cause unnecessary equipment shutdowns (false positives) or missed critical failures (false negatives); (3) warm store corruption affects historical trend analysis.

    **Kill switches:** (1) `BudgetExceeded` exception stops invocation before token explosion; (2) atomic writes (`fsync + temp file replace`) prevent partial hot_state corruption; (3) crash recovery manifest allows restart from clean state within 5-minute window; (4) cold store archives are append-only and deterministic, so historical data survives corruption. The manifest in `crash_recovery_manifest.jsonl` shows checkpoints at each stage — if any stage fails, the system can restart from last good checkpoint rather than propagating bad state.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)

    Initial run of System 3 compression tests failed because the token counter dispatch logic didn't handle missing `ANTHROPIC_API_KEY` gracefully — it raised `AuthenticationError` instead of falling back to the lightweight estimator. Fixed by wrapping the anthropic tokenizer initialization in try/except and defaulting to character-based estimation (4 chars/token approximation). Verified fix by running `test_token_counter_dispatch` in `pytest_context_strategy.log` which explicitly tests both code paths. Also checked that `budget.json` token counts were consistent between runs with and without API key set.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.

    From observing System 2's fork scratchpad implementation in `SYSTEM_2_SUMMARY.md`, I'd change the merge strategy. Currently, forks are selected by highest confidence score and merged wholesale. In the trace, Fork A (thermal, 0.7) won over Fork B (batch defect, 0.5) and Fork C (sensor drift, 0.3). But what if the real cause was a COMBINATION — thermal stress revealing a latent batch defect? I'd implement a **finding synthesis** step where winning fork's findings are merged with high-confidence sub-findings from losers, rather than discarding them entirely. The 2 KB overhead per fork (cited in the summary) is small enough that preserving more information wouldn't hurt, and the cold store archives already preserve all fork data for audit — why not use it for synthesis too?

---

**Reflection Completed:** 2026-10-20
**Run IDs Referenced:**
- System 1: `RUN-2026-10-20-S1-01-LOOP-7a3f2b`, `RUN-2026-10-20-S1-02-TOOLS-4f9e1c`, `RUN-2026-10-20-S1-03-DECOMP-8d2c5a`
- System 2: `RUN-2026-10-20-S2-01-STATE-5b7e3f`, `RUN-2026-10-20-S2-COMBINED-9c4d2e`
- System 3: `RUN-2026-10-20-S3-CONTEXT-6f1b8d`
- System 4: `RUN-2026-10-20-S4-VALIDATOR-3e9c1a`

**Artifacts Cited:** 14 unique files across 4 systems
**Status:** COMPLETE
