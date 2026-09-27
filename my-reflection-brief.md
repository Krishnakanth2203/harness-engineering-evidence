# Reflection Brief — Harness Engineering Capstone

**Name:** krishna kanth E
**Date:** September 27, 2026

**Environment**

- Model(s): claude-haiku-4-5-20251001 (System 1 run 20260925_045416); recorded-response client with no live inference spend (System 4)[cite: 2]
- OS / Python: Linux (Ubuntu 22.04), Python 3.13.0, pytest-9.1.1 (verified across all test runs)[cite: 2]
- Approx. API spend: ~$0.1157 USD for System 1's 8-claim test execution (per summary.md); System 2's LLM compression calls processed 12,334 + 347 tokens for refund and 11,475 + 503 tokens for subscription (per budget.json); System 4 operated purely offline via --recorded-response ($0.00 spend)[cite: 2].

---

## Part 1 — Per-system

### System 1 — Agentic loop

**1. Loop control.** In evidence/system1_agentic_loop/claim_01_kitchen_fire.jsonl (run 20260925_045416), the recorded stop sequence spans 3 turns: turn 1 stop_reason: tool_use (lookup_policy), turn 2 stop_reason: tool_use (record_claim_fact), and turn 3 stop_reason: end_turn. The execution flow is governed inside claims_intake/loop.py by run_claims_loop(). It inspects response.stop_reason directly: when stop_reason == "end_turn", it packages and returns FinalState, while stop_reason == "tool_use" appends the local tool results to messages and continues the cycle. No arbitrary turn counter or fragile regex parsing dictates termination.

**2. Anti-pattern.** In tests/test_antipatterns.py, the test test_no_integer_literal_iteration_cap_in_loop uses AST inspection to ensure loop.py contains no hardcoded range cap such as for _ in range(N) or while count < N. If a rigid cap (like range(3)) had been enforced, multi-step workflows like claim_02_stolen_bike and claim_03_water_damage—which require 4 to 5 turns to gather facts and clarify statements—would have been prematurely aborted midway through tool calls, preventing classify_claim and route_to_adjuster from executing.

**3. Tool design.** Both route_to_adjuster and escalate_to_human represent terminal dispatch actions with overlapping arguments (claim_summary vs. structured_summary). Misrouting is prevented because their tool schemas encode explicit, mutually exclusive confidence bounds: route_to_adjuster specifies confidence >= 0.6, whereas escalate_to_human explicitly triggers when confidence < 0.6 or policy ambiguities cannot be reconciled safely. When policy retrieval fails, _t_lookup_policy returns a structured diagnostic object ({"is_error": true, "error_category": "permanent", "is_retryable": false, "message": "..."}). Emitting is_retryable: false allows the agent to immediately ask the user for a valid policy identifier instead of spinning in redundant retry loops.

**4. Your numbers.** For claim_05_auto_collision in evidence/system1_agentic_loop/summary.md (run 20260925_045416), the run finished in 4 turns, utilizing 15,357 input tokens and 1,021 output tokens at an estimated cost of $0.0205 USD. Compared against claim_01_kitchen_fire (3 turns, 9,518 input / 583 output tokens, $0.0124 USD), the auto collision incident required an additional routing classification step and a larger schema payload, adding ~5,800 cumulative context tokens across the subsequent turn.

### System 2 — Context strategy

**5. The reduction.** From evidence/system2_context_strategy/budget.json (run 20260901-164727): baseline transcript tokens stood at 38,708, while assembled tokens dropped to 16,851, achieving a 56.47% context reduction[cite: 2]. Section counts: case_facts 204 tokens, resolved_refund 360 tokens, resolved_subscription 516 tokens, and active 15,789 tokens[cite: 2]. The active conversation dominates the assembled context (~93.7% of total)[cite: 2]. It is maintained byte-exact because it sits at the immediate boundary of the next model turn where recency bias is highest; any lossy compression of active state would risk catastrophic forgetting of immediate customer slots.

**6. Summarize vs preserve.** The boundary contract is enforced inside compressor.py::summarize_segment: if segment.status != "resolved", it rejects compression and throws a ValueError. In the numbers from budget.json, closed topics (resolved/refund and resolved/subscription) were compressed from 12,334 and 11,475 raw tokens down to 360 and 516 tokens respectively[cite: 2]. Conversely, the ongoing topic remained intact at 15,789 tokens, fulfilling the contracts verified in test_summarize_segment_refuses_to_compress_the_active_segment and test_active_segment_byte_exact[cite: 2].

**7. Facts block.** In eval.jsonl, all 6 evaluation queries passed (6/6)[cite: 2]. In eval_control.jsonl where the extracted case facts header was removed, Question 6 ("What is the structured status of the payment method update issue?") degraded from PASS to FAIL Q6 (expected FAIL) with the model responding that no structured status record existed[cite: 2]. Questions 1 through 5 passed in both variants[cite: 2]. This proves that while general narrative questions can be retrieved from raw active text, precise structured status tokens require a deterministic, uncompressed case facts header to prevent extraction failure.

### System 3 — Claude Code config

**8. Path-scoped rules.** In evidence/system3_claude_config/evidence_rule_tests.md, the frontmatter specifies:
paths:
  - "**/*.test.tsx"
  - "**/*.test.ts"
This is far superior to a directory-level CLAUDE.md because test files in modern monorepos are co-located across src/components/, src/pages/, and src/api/. Rather than duplicating test rules across multiple subdirectories or bloating the root config, glob-scoped matching activates conventions only when a test buffer is touched. This was verified by test_ac_02_06_test_file_matches_react_and_tests, which confirms a test file under src/components/ simultaneously inherits React component and test runner rules[cite: 2].

**9. Forked skill.** In evidence/system3_claude_config/evidence_rule_skill.md (.claude/skills/deploy-check/SKILL.md), the frontmatter defines:
context: fork
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash(git status:*)
  - Bash(git diff:*)
  - Bash(git log:*)
  - Bash(git rev-parse:*)
  - Bash(git ls-files:*)
  - Bash(gh pr view:*)
  - Bash(gh pr checks:*)
Running forked delegates verbose command output (such as git diff outputs and status tables) to an isolated subagent shell, returning only the final validation verdict to the parent session. The read-only tool allowlist ensures that the skill structurally cannot invoke git commit, git push, or write operations during a deployment check. Without context: fork, hundreds of ephemeral log lines would pollute the primary session context, causing unnecessary token consumption and context drift.

**10. Scope.** The validator output confirmed a clean hierarchy with OK (exit code 0)[cite: 2]. Project-level scope includes files checked into version control such as root CLAUDE.md and .claude/rules/*.md (react.md, api.md, tests.md), which are automatically loaded into every team member's workspace[cite: 2]. User-level scope consists of private machine configurations such as ~/.claude/skills/deploy-check-strict/ or local developer preferences; these remain strictly local, are never committed to git, and allow personal customization without affecting teammates[cite: 2].

### System 4 — Orchestration

**11. Push work down.** In shift_run_output.txt, the shift run reported shift C: 0 new defects against a warm tier populated with 40 baseline defect records (fixtures/defects.json)[cite: 2]. The indexed SQL query is WarmStore.defects_since() inside shift_monitor/warm.py: SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?[cite: 2]. Because gather_new_defects delegates directly to this SQL call without in-memory Python filtering (as verified by test_gather_new_defects_has_no_python_side_filtering), filtering occurs entirely inside the SQLite index before records reach the prompt, ensuring prompt size stays bounded under 4,000 characters regardless of warm database size[cite: 2].

**12. Crash recovery.** In recovery.py, the constant STALE_RESUME_THRESHOLD_MINUTES is set to 30[cite: 2]. The function recovery_decide() returns "resume" only if the execution manifest contains incomplete tasks and elapsed downtime is <= 30 minutes; otherwise, it forces "fresh"[cite: 2]. The cutoff is verified by test_recovery_decide_truth_table[30-False-resume] and [31-False-fresh][cite: 2]. A fresh restart with an injected summary is preferable after 30 minutes because intermediate machine state has likely drifted; resuming a stale run risks reasoning over outdated parameters, whereas a fresh run with an injected summary preserves verified conclusions while discarding corrupted transient steps.

**13. Small state.** In evidence/system4_orchestration/hot_state_size.txt, data/hot_state.json occupies exactly 643 bytes, well below the 5 KB storage threshold[cite: 2]. This is critical because shift_monitor executes across every physical shift indefinitely[cite: 2]. The dataclass explicitly caps historical tracking at 20 defect hashes (test_hotstate_rejects_more_than_20_hashes)[cite: 2]. Without this cap, state size would expand linearly over time, ballooning prompt costs and degrading model response times. Enforcing a constant state size ensures steady-state operation for months without performance degradation.

---

## Part 2 — Synthesis

**14. Three layers.**
- Model: claims_intake/tools.py (TOOL_SCHEMAS) provides the parameter contracts and enum constraints that Claude reasons over during tool dispatch.
- Harness: claims_intake/loop.py (run_claims_loop()) handles client interaction, intercepts tool calls, verifies stop_reason, and injects observation results into the context window.
- Orchestration: shift_monitor/recovery.py and manifest.py coordinate multi-shift execution, managing SQLite indexing, state file persistence, and automated crash recovery across distinct sessions[cite: 2].

**15. Deterministic vs prompt.** Deterministic behaviors enforced in code include the strict 20-hash cap on HotState (test_hotstate_rejects_more_than_20_hashes), atomic file writes (test_hotstate_atomic_write), and read-only tool constraints in deploy-check[cite: 2]. Prompt-guided behavior is used for semantic triage, such as deciding whether a claim description indicates water damage or porch structural wear. Code enforcement is required for hard safety constraints, file integrity, and bounded memory budgets. Prompt guidance is suited for interpreting ambiguous natural language and edge-case classification where rigid logic is impractical.

**16. Context, two faces.** System 2 manages context intra-session within a single conversation window: it dynamically compresses resolved dialogue threads down to 360/516 tokens while preserving the active thread byte-exact at 15,789 tokens (reducing the overall transcript from 38,708 to 16,851 tokens)[cite: 2]. System 4 manages context cross-session across independent processes: rather than summarizing conversation turns, it queries an indexed SQLite database (WHERE ts > ?) to ingest only recent incident deltas, serializing a tiny 643-byte state snapshot between runs[cite: 2]. Both maintain strict token efficiency, but System 2 operates via in-memory LLM summarization, whereas System 4 offloads memory to an indexed disk store.

**17. Reliability you can't see in one run.** In System 4, test_mid_write_read_reveals_prior_complete_lines tests resilience against unexpected power cuts or aborts midway through a log write[cite: 2]. A standard single run only executes along the happy path and cannot reveal if an interrupted write would leave a corrupted file that breaks subsequent shift restarts. Verifying atomic flush semantics (os.fsync) and boundary truth tables (test_recovery_decide_truth_table) under test automation is essential to ensure long-term stability in unattended production environments[cite: 2].

**18. Blast radius.** In System 3, the deploy-check skill restricts capabilities using an explicit allowlist: allowed-tools: [Read, Grep, Glob, Bash(git status|diff|log|rev-parse|ls-files:*), Bash(gh pr view|checks:*)][cite: 2]. Because write tools (Write, Edit) and mutating bash commands (git push, git merge) are omitted, the subagent cannot alter code, push broken branches, or overwrite repository files even if the model hallucinates[cite: 2]. The kill switch is the YAML configuration itself: because tool boundaries are enforced by Claude Code prior to invocation, removing permissions from the skill file immediately revokes access without needing application restarts.

---

## Part 3 — Honest assessment

**19. What broke.** During initial test execution for System 1, running claims_intake.run produced a client instantiation failure (TypeError: Client.__init__() got an unexpected keyword argument 'proxies') caused by an unpinned transitive upgrade to httpx>=0.28.0. This was resolved by explicitly pinning dependencies in the virtual environment:
pip install "httpx==0.27.2" "httpcore==1.0.5" "anthropic==0.39.0"
When moving to System 2, using anthropic==0.39.0 broke token counting (AttributeError: 'Messages' object has no attribute 'count_tokens'), requiring anthropic==0.69.0 to be restored for that specific virtual environment[cite: 2]. This confirmed that environment isolation across each system's virtual environment is essential.

**20. What you'd change.** I would introduce explicit dependency lockfiles (uv.lock or pinned poetry.lock) within each system's directory rather than relying on loose version ranges in pyproject.toml. Floating dependencies for httpx and anthropic resulted in hours spent diagnosing conflicting keyword arguments and missing token-counting methods, a failure mode that deterministic dependency locking completely avoids.