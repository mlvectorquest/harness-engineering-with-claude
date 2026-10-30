# Reflection Brief — Harness Engineering Capstone

**Name:** Johann Venter
**Date:** 2026-10-30

Replace each `→` with your answer. **Every answer cites at least one artifact from your own runs** — a run ID, file path, token count, claim outcome, or test count. Uncited answers do not pass. 3–6 sentences each unless noted. Paste short artifact snippets where they help.

**Environment**

- Model(s): System 1: `claude-haiku-4-5-20251001` (from `summary.md`). System 2: no `ANTHROPIC_API_KEY`, so calls went through the `claude -p` CLI fallback. System 3: none (pure validator). System 4: none (offline, `--recorded-response`).
- OS / Python: macOS 26.6.2 (Darwin 25.6.0) / Python 3.13.2
- Approx. API spend: about $0.14 (System 1 run `20261014_135400`, "total estimated cost: $0.1376"). Systems 3 and 4 cost nothing; System 2 ran through the CLI, so I have no per-call dollar figure for it.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → In `traces/claim_02_stolen_bike.jsonl` the sequence is `tool_use, tool_use, tool_use, tool_use, end_turn` (turn 1 `lookup_policy`, turn 2 four `record_claim_fact` calls, turn 3 `classify_claim` + `assess_severity`, turn 4 `route_to_adjuster`, turn 5 a confirmation). The decision is made in `claims_intake/loop.py`, function `run()`: it continues only if `response.stop_reason == "tool_use"`, returns a `FinalState` on `"end_turn"`, and raises `UnexpectedStopReason` for anything else. No assistant text is inspected and there is no turn counter acting as the stop condition.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `test_no_integer_literal_iteration_cap_in_loop` statically (via AST) rejects `for _ in range(<int>)` or `while x < <int>` caps in `loop.py`. In my run, `claim_04_neighbor_injury` needed 7 turns (`summary.md`, 1 clarification), so a hard cap of, say, 5 turns would have cut it off before `route_to_adjuster` and left a claim that really was resolvable unrouted. Instead the safety net is a `Budget` (tokens and wall-clock) that raises `BudgetExceeded` rather than silently truncating.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `route_to_adjuster` and `escalate_to_human` both finish a claim and take a summary, so the descriptions split them by a numeric rule: route "when classification confidence is at least 0.6 and severity has been assessed", escalate "when confidence is below 0.6 even after clarification". Both are labelled `TERMINAL TOOL` and "call exactly once". Errors come back as `{"is_error": true, "error_category": "transient"|"permanent", "is_retryable": bool, ...}` (`_err()` in `tools.py`), so the agent can retry a transient failure and stop on a permanent one, which a plain string would not tell it. I did not see a tool error fire in my traces, so this is from the code, not from my run.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → `claim_02_stolen_bike` took 5 turns, 16,875 input / 744 output tokens, est. $0.0206 (`summary.md`). The exercise README expects "7 routed, 1 escalated" at about $0.05 total, and says `claim_03` routes after a clarification and `claim_06` escalates. My run was 4 routed and 4 `incomplete` at $0.1376 total, and `claim_03` and `claim_06` both ended `incomplete` with 0 clarifications. The difference is model behavior: in those claims Haiku 4.5 ended its turn (`stop_reason: end_turn`) before calling `route_to_adjuster` or `escalate_to_human`, so the loop correctly stopped but the claim was never finished (for example the `claim_01` trace has only 2 turns). Q19 covers this in more detail.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → Run `20261029-154424`: baseline 47,144 tokens, assembled 20,264, a 57.02% reduction (note the counts use a `len/3.8` heuristic because no API key was available). The `active` section dominates at 19,538 of the 20,264 tokens. It stays byte-exact because it is the issue being worked right now and the answers (failure code `AVS_MISMATCH`, new card last-4 `7782`) have to be exact.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Resolved threads are summarized and the live thread is not: `resolved_refund` is 238 tokens and `resolved_subscription` is 336 tokens, both written by the compression prompt into the compressible middle. The `active` issue (19,538) is kept verbatim at the bottom, and the case-facts block (149 tokens) is pinned at the top where compression cannot touch it. That placement is the rule in `assemble.py`: only the middle is compressible.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → With the facts block, `eval.jsonl` passed 6/6. In `eval_control.jsonl` (facts block removed, Q1 and Q6 only), Q6 failed: the answer paraphrased the status as "**Activ…**" instead of the exact token (expected fragment `in_prog…`). Q1 still passed unexpectedly, since the $22.14 refund survived in the summaries or active text. So the facts block is what protects exact structured values that compression would otherwise paraphrase, though not every fact depends on it.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → `.claude/rules/tests.md` has `paths:` with `"**/*.test.tsx"` and `"**/*.test.ts"`. Test files are co-located next to source files in many directories, so a directory-level CLAUDE.md would have to be copied into each one. The glob loads the test rules only when Claude touches a matching file, anywhere in the repo, and keeps them out of context otherwise. The suite passed 35 tests (`pytest_output.txt`) and the validator printed `OK`.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → `.claude/skills/deploy-check/SKILL.md` has `context: fork` and an `allowed-tools` list of `Read`, `Grep`, `Glob`, `Bash(git status:*)`, `Bash(git diff:*)`, `Bash(git log:*)`, `Bash(gh pr view:*)` and similar. Forked, the verbose check output stays in a separate session and only a single pass/fail summary returns to the main one. Read-only means the check cannot edit files or push, and that is enforced by the tool allowlist instead of by asking nicely in the prompt. Without the fork, the output floods the main context, and without the allowlist a pre-deploy check could change the thing it is checking.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator printed `OK` (`validator_output.txt`), which includes the `claude_md.py` checks that CLAUDE.md distinguishes the two scopes. Project-level is shared through git: this config's `.claude/standards/api.md` and `.claude/rules/*.md`. User-level lives in `~/.claude/` and is never shared: CLAUDE.md's own example is a personal commit-message style or a personal `/morning` command.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → My Shift C run printed `shift C: 17 new defects` (`shift_C_output.txt`), while `warm.sqlite` holds 40 defects in total. `defects_since` runs `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?`, served by `idx_defects_ts`, so filtering happens in SQL and not in Python or the model. Only those rows go into the prompt, so the model never sees the other 23 and the prompt does not grow as history does.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → `recovery.decide()` returns `"resume"` only if the manifest is incomplete and the last step is within `STALE_RESUME_THRESHOLD_MINUTES = 30` (about 1/16 of an 8-hour shift); otherwise `"fresh"`. A stale partial run may be working from a state that has since changed, so resuming can carry forward outdated assumptions. Starting fresh with the manifest's findings injected as a summary keeps the useful partial results and drops the stale working set. Forked investigations stay isolated in the same way: `fork.py` `fork_for_hypothesis()` copies `hot_state.json` into its own folder per hypothesis, each fork writes only to its own scratchpad, and `merge_findings()` appends the entries to the main scratchpad afterwards, so a bad fork cannot corrupt the main state (`test_merge_findings_appends_without_rewriting_existing` passes in `system4/pytest_output.txt`).

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → My evidence copy is 965 bytes (`hot_state_size.txt`), against `HOT_STATE_BYTE_BUDGET = 5,120` in `state.py`. The state goes into every prompt on every shift forever, so anything allowed to grow would eventually dominate the context and the cost. `write_atomic` raises `ValueError` over budget, and `_trim_to_budget` drops alerts first, so the cap is enforced in code and not by trusting the model to stay short.

---

## Part 2 — Synthesis

*Graded on connecting two or more systems. Cite a named file/artifact from each.*

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: Haiku 4.5 choosing `classify_claim` and `route_to_adjuster` in `claim_02_stolen_bike.jsonl`, because it makes the judgment calls and nothing deterministic could read the claim text.
    → Harness: `claims_intake/loop.py` `run()` (and System 2's `assemble.py`), which decide what the model sees and when it stops.
    → Orchestration: `shift_monitor/pipeline.py` with `hot_state.json` and `shift_scratchpad.jsonl`, which carry state across independent sessions (shift C's 17 defects).

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → Guaranteed in code: `HotState.write_atomic` refuses a payload over 5,120 bytes (mine is 965), and `loop.py` stops on `end_turn`. Guided by prompt: the tool descriptions tell the model to call `route_to_adjuster` or `escalate_to_human` before finishing, and in my run `claim_01`, `claim_03`, `claim_06` and `claim_07` ended `incomplete` because it did not (`summary.md`). Use code for anything that must always hold (limits, safety, state integrity), and prompts for judgment where a wrong answer is recoverable, with the harness catching the failure.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 cut a 47,144-token transcript to 20,264 (57.02%, `budget.json`) by summarizing resolved threads (238 and 336 tokens) and keeping the active one exact. System 4 keeps the model's view small across sessions: `hot_state.json` is 965 bytes of a 5,120 budget, and only 17 of 40 stored defects reach the prompt. The shared principle is that the harness decides what the model sees and keeps the rest outside the window. System 2 does it by compressing inside one conversation, and System 4 does it by externalizing state to disk and SQL between conversations.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → `tests/test_antipatterns.py` statically checks that `loop.py` has no integer-literal turn cap and no string-membership control flow, which a run that happens to succeed would never show. My `claim_02` finished in 5 turns, so a hidden cap of 5 would have looked fine, and it would only fail on a longer claim like `claim_04` (7 turns). It matters before shipping because that failure shows up only on rare inputs, in production.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → System 1: the agent can only call the seven tools in `TOOL_SCHEMAS`, and its only side effects are appending to the `queues/*.jsonl` files and the trace, so the worst case is a wrongly routed or unrouted claim and not a changed policy or payment. The kill switch is `Budget` in `budget.py`, which raises `BudgetExceeded` on a token or wall-clock limit, plus `UnexpectedStopReason` for unknown stop reasons. In my run the real misbehavior was mild: four claims ended `incomplete` and nothing was wrongly routed.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → The System 1 run (`20261014_135400`) did not match the README: 4 routed and 4 `incomplete`, with no `escalations.jsonl`, including `claim_06`, which should have escalated. In the `claim_01` trace the model returned `end_turn` on turn 2 after recording facts, before any terminal tool call. I did not fix it, and decided to submit the run as it is. I did confirm the loop behaved correctly by marking those claims `incomplete` instead of faking a route. Separately, with no API key, System 2's `budget.json` records a heuristic token counter. Also, my System 1 suite has 29 tests (`collected 29 items` in `system1/pytest_output.txt`), which matches the exercise README's "29 tests" even though the assignment text says 30.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would add a harness-level guard to the System 1 loop: when `stop_reason == "end_turn"` arrives with no terminal tool called, re-prompt once, and if that fails, escalate to a human automatically. Today the loop in `loop.py` returns on `end_turn` and the claim becomes `incomplete` (4 of 8 in `summary.md`), which silently leaves a claim in limbo. The prompt already tells the model to finish with a terminal tool and it was not enough, so this guarantee belongs in code, as the byte budget in System 4 does.
