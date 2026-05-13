# Security review notes

Date: 2026-05-13

## Scope

Manual audit of the plugin source with focus on crashable paths and robustness against malformed or unexpected IPC data.

Reviewed files:
- `plugin/i3wm-window-title.c`
- `plugin/i3wm-window-title.h`

## Findings

### 1. NULL-pointer dereference from unchecked i3ipc return values (high)

The plugin used i3ipc APIs without checking returned pointers in several places:
- connection setup (`i3ipc_connection_new`)
- event subscriptions (`i3ipc_connection_subscribe`)
- tree/focused lookups (`i3ipc_connection_get_tree`, `i3ipc_con_find_focused`)
- event payload fields (`e`, `e->change`, `e->container`)

A failing IPC interaction or malformed/partial event could trigger a process crash (panel plugin DoS).

Status: **fixed** in current branch by adding guards before dereference and safe fallback UI text.

### 2. Unsafe unref on reconnect/free when connection is absent (medium)

Reconnect and free paths unconditionally called `g_object_unref(i3wmtp->conn)`.
If initialization failed before `conn` was set, this can dereference NULL and crash.

Status: **fixed** in current branch by checking `conn != NULL` before unref.

## Residual risk

- This plugin still trusts the semantics of library-owned strings from `i3ipc_con_get_name`. If the upstream library ever returns invalid UTF-8 or non-owned memory unexpectedly, GTK behavior depends on upstream contracts.
- No authentication layer exists for local i3 IPC by design; threat model remains local-user oriented.

## Recommendation

- Keep strict null-checking on every i3ipc object boundary.
- Consider defensive UTF-8 validation before rendering external strings to GUI labels.
- Add regression tests or fuzz-like harness around event callback handlers if a test framework is introduced.
