## Testing

### Running tests

```bash
# Backend (vitest)
cd backend && npm test          # watch mode
cd backend && npm run test:run  # single run
cd backend && npm run test:coverage

# Frontend (vitest)
cd frontend && npm test         # watch mode
cd frontend && npm run test     # single run (vitest run)
```

### Test file locations
- Backend: `*.test.js` or `__tests__/*.test.js` next to the file under test
- Frontend: `*.test.js` next to the file under test

### What to test
- **Backend utils and services** — pure functions, SQL builders, business logic. No HTTP layer.
- **Frontend utils and composables** — format helpers, state logic, pure functions.
- **Do not test** router handlers directly, Vue components (no component tests currently), or third-party integrations.

### Writing tests
- One `describe` block per file/module under test.
- Test behavior, not implementation — assert outputs and side effects, not internal calls.
- Use real inputs and expected outputs, not mocks of the thing being tested.
- If a function is hard to test, it's a design problem — simplify the function, don't mock around it.

### E2E tests
Playwright-based E2E scripts live in `docs/e2e/`. Not part of CI — run manually for feature verification.
