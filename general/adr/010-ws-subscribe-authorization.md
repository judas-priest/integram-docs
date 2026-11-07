# ADR-010: WebSocket Subscribe-Level Authorization

**Date:** 2026-05-02
**Status:** Accepted

## Context

The WS server authenticated connections via JWT but had no per-channel access control. Any authenticated workspace user could subscribe to any objects type or document, regardless of their v2 grants or document sharing settings.

Additionally, ops handlers (`handleObjectsOp`, `handleDocumentsOp`) did not verify that the sender was subscribed to the channel, allowing identity leakage to authorized subscribers.

## Decision

Authorize at subscribe time, not at broadcast time. This is the industry standard (Laravel Echo, Pusher).

- **Objects channel**: check `_v2_grants` loaded at auth time (`ws._user.grants`) for the typeId using `canSubscribeObjects()`
- **Documents channel**: call `checkAccess()` from documents/service.js for the documentId — mirrors existing REST access control
- **Ops handlers**: verify sender is subscribed before relaying

## Consequences

- One DB call per subscribe for documents channel (lightweight, called once)
- No broadcast filtering needed — subscribers are pre-authorized
- Legacy workspaces without `_v2_grants` rows fall back to allow (no regression)
- Future channels must implement a subscribe gate — this ADR is the precedent
- `canSubscribeObjects` is exported from ws.js to allow unit testing without mocking
