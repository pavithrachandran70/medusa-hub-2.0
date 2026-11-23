# What would go wrong

## Notes for the gate

### N1
- file: (not applicable)
- section: (not applicable)
- quote: (not applicable)
- severity: note
- problem: The spec documents reference a `www/` directory for the storefront (e.g., `www/src/lib/context/comparison-context.tsx`, `www/src/pages/compare.tsx`), but `repo-tree/medusa-appli` shows no `www/` directory exists in this repository. The only visible application entry point is `packages/admin/dashboard/src/app.tsx`, which is an admin dashboard, not a customer-facing storefront. The repo analysis confirms this is a monorepo with an admin dashboard but no visible storefront application. All story file paths assume a `www/` structure that does not exist. The comparison feature is described for customers, not administrators, so implementing it in the admin dashboard would contradict the requirements. Before implementation can proceed, clarify: (1) Does a separate storefront repository exist outside this monorepo? (2) Should a new storefront package be created in this monorepo? (3) Are the file paths in the documents incorrect?

VERDICT: PASS
