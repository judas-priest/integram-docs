# ADR-013: Module owns its own DDL

**Status:** Accepted
**Date:** 2026-05-30

## Context

`bootstrap-pg-schemas.js` creates tables for all modules when a workspace is first created. Individual modules also have `ensureTables()` / `ensureAgentRegistryTable()` that create the same tables via `CREATE TABLE IF NOT EXISTS` at first access (lazy init).

When bootstrap and module DDLs diverge (different columns, types, constraints), `CREATE TABLE IF NOT EXISTS` sees the bootstrap-created table and skips creation. The module then operates against a schema it doesn't expect — missing columns cause 500 errors at runtime.

This happened with `_v2_agent_registry`: bootstrap created it with 8 columns, service.js expected 18. Incremental `ADD COLUMN IF NOT EXISTS` migrations were added to bridge the gap, but they grew fragile and incomplete.

## Decision

**Each module is the single source of truth for its own tables.**

- `bootstrap-pg-schemas.js` creates only the workspace schema (`CREATE SCHEMA IF NOT EXISTS`) and tables that have no module-level `ensureTables()`.
- If a module has `ensureTables()` with `CREATE TABLE IF NOT EXISTS`, bootstrap MUST NOT create that table.
- No incremental `ADD COLUMN IF NOT EXISTS` migrations — the `CREATE TABLE` statement contains the full schema. If the table already exists with the correct schema, it's a no-op. If it doesn't exist, it's created correctly.
- Legacy tables with wrong schemas are fixed manually (DROP + let module recreate), not through migration chains.

## Consequences

- New workspaces get correct tables from module's `ensureTables()` on first access.
- No schema drift between bootstrap and modules.
- Adding a column to a module's table = one change in one file (module's `ensureTables()`).
- Existing workspaces with legacy tables require a one-time manual fix (DROP old table, module recreates on next access). This is acceptable because agent_registry/agent_tasks had no critical data in most workspaces (only kval had agents).
