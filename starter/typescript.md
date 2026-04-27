# Typescript

## Commands

- After code changes (not documentation changes): `npm run check` (get full output, no tail). Fix all errors, warnings, and infos before committing.
- Note: `npm run check` does not run tests.
- Never run: `npm run dev`, `npm run build`, `npm test` (full suite) — these start long-running processes or run too much.
- Run a single test file with: `npx tsx ../../node_modules/vitest/dist/cli.js --run test/specific.test.ts`. Run from the package root, not the repo root.
- If you create or modify a test file, run *that file only* using the command above and iterate until it passes. Never run the full suite to verify a single-file change.
- Otherwise, only run tests when the user asks.
