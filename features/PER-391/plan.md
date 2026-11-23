# Plan — PER-391 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

A cache invalidation fix in one repo that updates the batch inventory location levels mutation to clear the query cache after a successful update, ensuring the UI reflects the new quantity immediately.

## Steps

1. [x] **analyst (inline_story)** — Turn the ticket into a testable inline story with clear Given/When/Then and tech notes for the cache invalidation pattern.
   - focus: Focus on the cache invalidation mechanism: which query key to invalidate, when to trigger it (after mutation success), and the existing cache patterns in the codebase.
2. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
3. [ ] **dev** _(auto)_ — Loop delivery.
4. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
5. [ ] **open_pr** _(auto)_ — Loop delivery.
6. [ ] **pr_review** _(auto)_ — Loop delivery.
7. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope is clear: invalidate the inventory item levels query cache after the batch update mutation succeeds.
- **architect** — No new components, interfaces, or contracts; this is a cache invalidation fix inside existing mutation logic.
- **scrum_master** — One implementable story; no breakdown needed.
- **designer** — The developer can code this from the inline story; cache invalidation follows an established pattern and does not need a file-by-file plan.
- **test_architect** — The ticket already states the regression test requirement in AC6; the analyst will capture it in the inline story.
- **critic** — Only one spec document (the inline story); no cross-artifact consistency to check.
