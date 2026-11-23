# Plan — PER-383 (v1)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Customer wishlist feature with add/view/remove operations, requiring new API endpoints, database schema for wishlist storage, and UI components across product and wishlist pages; feature_with_architecture fits because it adds new REST endpoints for wishlist CRUD and a new database table to persist wishlist items.

## Steps

1. [ ] **analyst (requirements)** — Define clear requirements for wishlist operations, authentication constraints, and UI behaviors.
   - focus: Focus on the customer-wishlist relationship, persistence requirements, and the add/view/remove operations with their edge cases (duplicates, empty states).
2. [ ] **architect (hld)** — Design the wishlist API endpoints, database schema, and integration with existing customer and product modules.
   - focus: Define REST endpoints for wishlist CRUD operations, wishlist_item schema with customer_id and product_id, and how it integrates with the customer module for authentication.
3. [ ] **scrum_master (breakdown)** — Break down into implementable stories: backend (API + DB), frontend (product listing integration), frontend (wishlist page), and tests.
   - focus: Separate backend API and schema from UI components; product listing wishlist button is one story, dedicated wishlist page is another.
4. [ ] **designer (lld)** — Provide file-level implementation plans for each story covering API routes, database migrations, service layer, and UI components.
   - focus: Detail the wishlist module structure, API route handlers, database entity definitions, and React component files for the wishlist icon and wishlist page.
5. [ ] **test_architect (tests)** — Map acceptance criteria to test cases covering API operations, authentication, duplicate handling, and empty states.
   - focus: Cover API tests for CRUD operations, integration tests for customer-wishlist association, and UI tests for add/remove actions and empty state display.
6. [ ] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [ ] **critic (consistency_check)** — Ensure consistency between HLD API contracts, story-level schemas, and test coverage for all acceptance criteria.
8. [ ] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [ ] **dev** _(auto)_ — Loop delivery.
10. [ ] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [ ] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Ticket clearly defines scope (authenticated customers only), acceptance criteria (10 specific behaviors), and boundaries (no sharing or public wishlists mentioned).

## Planner notes

- attempt 1: loop feature_with_architecture is ruled out: you answered that its not-when [N1] applies (one story's worth of work inside one component, and every interface stays as it is → quick_fix or story_with_design). Pick the loop that line points to, or the next loop down whose not-when lines do not apply
