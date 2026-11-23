# Plan — PER-395 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: quick_fix
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Display Order ID in the Inventory Item Reservations table: a UI update with backend query changes in one repo. Fits quick_fix because it modifies existing code to display an already-available field without changing any interface or requiring a file plan.

## Steps

1. [ ] **analyst (inline_story)** — Define what changes are needed: query modification to include Order ID, table column addition, navigation setup, and fallback display.
   - focus: Write one inline story specifying: (1) the Admin API query change to retrieve Order ID from reservations, (2) the table component update to display it as a clickable link, (3) navigation to order details page, (4) display '-' when no Order ID exists, (5) test cases for both scenarios.
2. [ ] **critic (consistency_check)** — Verify the inline story covers all acceptance criteria consistently.
3. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
4. [ ] **dev** _(auto)_ — Loop delivery.
5. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
6. [ ] **open_pr** _(auto)_ — Loop delivery.
7. [ ] **pr_review** _(auto)_ — Loop delivery.
8. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Scope and acceptance criteria are clearly stated in the ticket; no product ambiguity exists.

## Planner notes

- attempt 1: loop story_with_design is ruled out: you answered that its not-when [N2] applies (the work splits into stories a developer would pick up separately → feature_with_architecture). Pick the loop that line points to, or the next loop down whose not-when lines do not apply
