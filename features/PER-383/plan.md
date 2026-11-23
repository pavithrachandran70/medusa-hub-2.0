# Plan — PER-383 (v2)

- made from: ticket
- source: LLM planner
- needs plan review: yes
- track hint: standard
- loop: feature_with_architecture
- delivery: delivery
- PR fixes run automatically: pr_fix_minor, pr_fix_moderate (other PR comments go to a person)

Add wishlist feature for authenticated customers to save, view and manage products across sessions; needs database schema, API endpoints and UI components, so feature_with_architecture fits.

## Steps

1. [x] **analyst (requirements)** — Define requirements for wishlist persistence, API contracts and UI behavior.
   - focus: Focus on authentication requirements, data persistence across sessions, and idempotency for duplicate adds.
2. [x] **architect (hld)** — Design database schema for wishlist items, REST/GraphQL API endpoints, and integration with existing customer and product modules.
   - focus: Define wishlist table schema with customer and product foreign keys, API endpoints for add/remove/list operations, and authentication middleware requirements.
3. [x] **scrum_master (breakdown)** — Break down into backend (schema, API), frontend (UI components, state management), and testing stories.
   - focus: Split into: 1) database migration and API layer, 2) UI components and integration, 3) tests covering all acceptance criteria.
4. [x] **designer (lld)** — Provide file-level implementation plans for database models, API routes, frontend components and state management.
   - focus: Detail migration files, service layer methods, React components for wishlist icon/page, and API client calls.
5. [x] **test_architect (tests)** — Map test cases to acceptance criteria covering API operations, authentication, idempotency, persistence and UI states.
   - focus: Include integration tests for API endpoints, unit tests for idempotency logic, and E2E tests for UI flows including empty state.
6. [x] **merge_pack (pack)** _(auto)_ — Loop feature_with_architecture: Deterministic merged_plan.md from ordered specs and story summaries.
7. [x] **critic (consistency_check)** — Verify consistency between database schema in HLD, API contracts in specs, and frontend integration across requirements, breakdown and stories.
8. [x] **architect_gate (review)** _(auto)_ — Human approval of specs and this plan before Dev.
9. [x] **dev** _(auto)_ — Loop delivery.
10. [x] **pre_pr_qa** _(auto)_ — Loop delivery.
11. [x] **open_pr** _(auto)_ — Loop delivery.
12. [ ] **pr_review** _(auto)_ — Loop delivery.
13. [ ] **merge** _(auto)_ — Loop delivery.

## Skipped roles

- **tpm_gate** — Ticket clearly defines scope (authenticated customer wishlist) and acceptance criteria (10 specific testable requirements).

## Planner notes

- attempt 1: loop feature_with_architecture is ruled out: you answered that its not-when [N1] applies (one story's worth of work inside one component, and every interface stays as it is → quick_fix or story_with_design). Pick the loop that line points to, or the next loop down whose not-when lines do not apply
