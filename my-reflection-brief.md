# Reflection Brief — Harness Engineering Capstone

**Name:** Ankita Jadhav  
**Date:** 27 September 2026

## Environment

- **Model(s):** `claude-haiku-4-5-20251001` (recorded in `evidence/system1_agentic_loop/summary.md`)
- **OS / Python:** Linux / Python 3.13.0 (recorded in the pytest logs)
- **Approx. API spend:** $0.1090 USD for the recorded System 1 run (`evidence/system1_agentic_loop/summary.md`). The evidence does not record a combined spend for all four systems.

---

## Part 1 — Per-system

### System 1 — Agentic loop

**1. Loop control.**  
The trace in `evidence/system1_agentic_loop/claim_01_kitchen_fire.jsonl` shows the loop progressing through `tool_use`, `tool_use`, then `end_turn`. That demonstrates the core stop-reason contract: continue after `tool_use` and stop after `end_turn`. The submitted evidence does not contain the loop source file/function name, so I am not inventing one; the trace is the direct evidence of the behavior.

**2. Anti-pattern.**  
The System 1 suite reports all 29 tests passing in `evidence/system1_agentic_loop/pytest_S1.log`, including the anti-pattern suite. The important failure mode is a loop that continues after a terminal `end_turn` instead of returning. That would cause unnecessary extra model calls and could turn a completed claim into an unintended continuation.

**3. Tool design.**  
The trace demonstrates structured tool inputs: `lookup_policy` receives `{"policy_id":"POL-1001"}`, while `record_claim_fact` receives named fields such as `incident_date`, `incident_type`, and `scope_of_damage` in `claim_01_kitchen_fire.jsonl`. Structured inputs constrain what each tool receives and reduce ambiguity compared with sending an unstructured string. The submitted trace is the evidence for this behavior.

**4. Your numbers.**  
For `claim_01_kitchen_fire`, the trace contains 3 turns, with 9,498 input tokens and 576 output tokens; the run summary estimates $0.0124 for that claim. The overall System 1 run processed 8 fixtures and cost an estimated $0.1090, as recorded in `summary.md`. The actual claim therefore used a small number of turns but still accumulated substantial input context.

---

### System 2 — Context strategy

**5. The reduction.**  
`evidence/system2_context_strategy/budget.json` records 38,708 baseline tokens and 16,805 assembled tokens, a 56.59% reduction. The assembled context is dominated by the `active` section at 15,789 tokens, compared with 204 case-facts tokens, 369 resolved-refund tokens, and 461 resolved-subscription tokens. The active conversation is kept verbatim because it contains unresolved/current information that must not be rewritten as if it were settled facts.

**6. Summarize vs preserve.**  
The context strategy summarizes resolved material into compact sections while preserving the active segment separately. `budget.json` shows the resolved refund and subscription sections at 369 and 461 tokens, while the active section remains 15,789 tokens. This separates durable facts from still-changing conversation state and reduces the risk of a summary silently changing an unresolved fact.

**7. Facts block.**  
The normal evaluation passes all six questions in `evidence/system2_context_strategy/eval.jsonl`, including Q6, which returns the exact structured token `payment_update_status: in_progress`. The control evaluation in `eval_control.jsonl` fails Q6 because it loses the structured status token and instead says that no status token is present. This regression demonstrates why the durable facts block matters: exact structured facts must survive compression rather than being replaced by a paraphrase.

---

### System 3 — Claude Code config

**8. Path-scoped rules.**  
`evidence/system3_claude_config/evidence_rule_react.md` uses frontmatter paths `src/components/**/*` and `src/pages/**/*`. Path-scoped rules are better than putting every convention in a directory-level `CLAUDE.md` because the rule activates only for matching files. This keeps unrelated sessions and files from receiving irrelevant instructions.

**9. Forked skill.**  
`evidence/system3_claude_config/evidence_rule_skill.md` declares `context: fork` and uses a read-oriented allowed-tool set. The fork keeps verbose discovery output out of the parent session while returning a compact result, and the allowed tools prevent the skill from modifying or deploying anything. The evidence explicitly describes `/deploy-check` as read-only and forked.

**10. Scope.**  
The project-level example is `evidence/system3_claude_config/CLAUDE.md`, which says the shared conventions live in the repository and are loaded for the team's Claude sessions. The same file distinguishes user-level configuration under `~/.claude/` and says it is not versioned or shared. Thus project-level configuration is team/repository scope, while user-level configuration is personal scope.

---

### System 4 — Orchestration

**11. Push work down.**  
The S4 test log includes `test_defects_since_uses_index_and_does_not_load_full_table` and `test_defects_since_respects_limit`. These tests show that history lookup is pushed into the indexed warm-store query rather than loading the whole table into the model context. The model therefore receives the selected result needed for the shift rather than the entire warm history.

**12. Crash recovery.**  
`evidence/system4_orchestration/pytest_S4.log` shows the recovery truth-table tests passing, including the 30-minute threshold test and tests for incomplete/complete state. The test suite explicitly records `test_threshold_constant_is_30_minutes` as passed. A fresh start can therefore be safer than resuming when the saved state is incomplete or stale beyond the recovery threshold.

**13. Small state.**  
`evidence/system4_orchestration/hot_state_size.txt` reports that `data/hot_state.json` is 643 bytes. A small hot state matters because this system runs once per shift indefinitely: repeatedly carrying a large state forward would increase storage and context costs and make every future invocation more expensive. The 643-byte result demonstrates the intended small-state design.

---

## Part 2 — Synthesis

**14. Three layers.**

- **Model:** `evidence/system1_agentic_loop/claim_01_kitchen_fire.jsonl` — the model produces stop reasons and structured tool calls that drive the agent loop.
- **Harness:** `evidence/system3_claude_config/CLAUDE.md` and the path-scoped rule files — they define project conventions and when those conventions load.
- **Orchestration:** `evidence/system4_orchestration/shift_run_output.txt` and `scratchpad_line.jsonl` — they show a shift-level run using persisted state and a scratchpad.

Together these separate model behavior, development-time constraints, and cross-invocation state/control.

**15. Deterministic vs prompt.**  
A deterministic behavior is atomic state persistence: `pytest_S4.log` records `test_hotstate_atomic_write` as passed, so the write contract is enforced in code rather than left to model interpretation. A prompt-guided behavior is the construction of the rich shift prompt, which is explicitly tested by `test_build_rich_prompt_stays_under_4000_chars`. Deterministic rules are appropriate when correctness must be guaranteed; prompts are appropriate when the model needs reasoning guidance within those hard boundaries.

**16. Context, two faces.**  
System 2 manages context within a long conversation by reducing 38,708 baseline tokens to 16,805 assembled tokens, a 56.59% reduction, while preserving structured facts and the active segment (`budget.json`). System 4 manages context across shifts by keeping durable state outside the model context; its hot state is only 643 bytes and the scratchpad contains a single structured finding. The shared principle is to keep the model's working context small while retaining durable information outside the immediate prompt.

**17. Reliability you can't see in one run.**  
The S4 test `test_mid_write_read_reveals_prior_complete_lines` guarantees that a partially written scratchpad does not erase previously completed lines. A single successful shift run would not expose this crash/mid-write case because it requires an interrupted or incomplete write. The passing test matters because recovery behavior must be correct even when the normal run looks successful.

**18. Blast radius.**  
For System 4, the immediate blast radius of a bad run is the shift-level state and the single model invocation: the test `test_run_shift_invokes_client_exactly_once_and_writes_state` constrains the invocation and state-write boundary. The evidence does not show a dedicated application-level kill-switch flag, so I would not claim one exists. Operationally, stopping the shift runner before the next invocation is the containment point visible from the submitted evidence.

---

## Part 3 — Honest assessment

**19. What broke.**  
No functional failure remained in the final recorded validation: System 1 reports 29 passed, System 2 reports 30 passed, System 3 reports 35 passed with validator output `OK`, and System 4 reports 33 passed. The evidence therefore supports saying that the final environment was checked successfully rather than inventing a failed test that is not recorded in the submission. These results are captured in the four `pytest_*.log` files and `validator_output.txt`.

**20. What I'd change.**  
I would add an explicit, versioned operational disable/kill-switch state to the orchestration layer. The submitted System 4 evidence demonstrates strong controls around atomic state, recovery, one invocation, and forked scratchpads, but it does not expose a dedicated disable flag. Making that control explicit would give operators a deterministic way to stop future shift invocations without relying on model behavior.
