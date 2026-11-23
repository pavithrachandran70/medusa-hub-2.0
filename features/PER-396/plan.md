# Plan — PER-396 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Product comparison feature adds UI for selecting and comparing up to 4 products side-by-side with persistence in browser storage; fits feature_with_architecture as N2 applies per the rejection feedback, requiring HLD and story breakdown.

## Steps

1. [ ] **analyst (requirements)** — Translate the ticket into Given/When/Then requirements with clear acceptance criteria for the comparison feature.
   - focus: Focus on the user flows: adding products to comparison from product cards and detail pages, viewing the comparison page, managing the comparison list, and browser storage persistence. Clarify the 4-product limit behavior and duplicate prevention logic.
2. [ ] **architect (hld)** — Design the component structure, state management approach, and localStorage schema before splitting into implementation stories.
   - focus: Define the high-level architecture: comparison state management solution (context/store), localStorage schema and sync strategy, component hierarchy (CompareButton, ComparisonPage, ComparisonList), routing structure, and integration points with existing product card and detail page components.
3. [ ] **scrum_master (breakdown)** — Break down the comparison feature into implementable stories for separate pickup.
   - focus: Split into logical stories such as: comparison state management setup, CompareButton integration in product cards, CompareButton integration in product detail pages, comparison page UI, localStorage persistence, and any necessary routing changes.
4. [ ] **designer (lld)** — Each story needs a concrete file plan showing which components to create or modify and how.
   - focus: For each story, specify the files to create or modify, component structure, props interfaces, state hooks, localStorage utility functions, and integration points with existing code.
5. [ ] **test_architect (tests)** — The acceptance criteria include specific behaviors that should be pinned as test cases per story.
   - focus: Map test cases 1:1 to acceptance criteria across stories: adding products from different entry points, enforcing the 4-product limit, preventing duplicates, displaying comparison data, removing products, clearing all, and verifying localStorage persistence.
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Cross-check that requirements, HLD, breakdown, and story LLDs agree on the component structure, state management, and localStorage schema.
   - focus: Verify consistency between the HLD's architecture decisions and each story's implementation details, ensuring the state management approach and localStorage schema are implemented uniformly across all stories.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — The ticket clearly states the scope (comparison of up to 4 products with specific display fields) and acceptance criteria; no product sign-off is needed.

## Planner notes

- attempt 1: loop story_with_design is ruled out: you answered that its not-when [N2] applies (the work splits into stories a developer would pick up separately → feature_with_architecture). Pick the loop that line points to, or the next loop down whose not-when lines do not apply
