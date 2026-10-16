# Module: testing

**Path:** `src/api/v2/modules/testing/`
**Files:** `router.js`, `service.js`
**Base URL:** `/api/v2/:db/testing/...`
**Auth:** JWT required for all endpoints.

## Purpose

Internal QA test session tracking. Allows QA engineers to create named test sessions, record pass/fail results for individual test cases, and track overall progress. Admin can view all sessions across all users via `GET /admin/qa-results`.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/testing/sessions` | List current user's sessions with progress stats |
| POST | `/testing/sessions` | Create new session (`notes` optional) |
| GET | `/testing/sessions/:id` | Get session with all test results |
| DELETE | `/testing/sessions/:id` | Delete session |
| PATCH | `/testing/sessions/:id/results` | Upsert one test result |

## Result Schema

```json
{
  "planId": "smoke-tests",
  "testId": "T01",
  "status": "pass",
  "comment": "works as expected"
}
```

- `planId`: test plan identifier (e.g. feature name or test file)
- `testId`: individual test case ID within the plan
- `status`: `pass` | `fail` | `skip` | `blocked`

## Notes

- Not for end-user features — strictly internal QA tooling
- Test plans and test cases are defined in frontend code; this module only stores results
- `GET /admin/qa-results` (in `admin` module) provides a cross-user view for QA managers

## DB Tables (global public schema)

- `qa_test_sessions` — `id`, `user_email`, `notes`, `created_at`
- `qa_test_results` — `session_id`, `plan_id`, `test_id`, `status`, `comment`, `updated_at`
