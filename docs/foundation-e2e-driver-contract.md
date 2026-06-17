# E2E View-Driver Contract — `data-viewid`

> Status: 🟡 In progress — QA-Automation [Enhancement 05](../roadmap/foundation/qa-automation/enhancements/05-reusable-view-driver-engine.md).

The reusable e2e engine drives the standard React views (list, form, kanban,
search, chatter, notifications, dialog) through **one stable attribute contract**
instead of fragile selectors (placeholder text, `get_by_text` content, `.first`
/ `.last` ordinals, `nth-child`, synthetic-event dispatch). The frontend view
managers emit `data-viewid="…"` hooks; the Python view-driver engine targets
them. When the UI is revamped, only the attribute and its constant move —
**tests never change**.

This page is the single source of truth. It is mirrored verbatim by:

- **Frontend:** `data-viewid="…"` attributes in `src/frontend/src/managers/*`.
- **Python:** the constants in [`src/ede/foundation/qa_automation/ui/viewids.py`](../src/ede/foundation/qa_automation/ui/viewids.py).

## Naming

`data-viewid="ede-<surface>-<role>"` — lowercase, dash-separated, product-owned
(deliberately **not** `data-testid`, since these attributes ship in the production
webclient DOM and shouldn't read as test scaffolding). Record-bearing nodes also
carry a keyed `data-*` attribute (`data-record-id`, `data-field`, `data-stage`,
`data-message-id`, …) so a driver can address one row / card / field by identity.

Playwright's test-id attribute is registered to `data-viewid` once in the fixture
stack (`selectors.set_test_id_attribute("data-viewid")`), so authors may also use
`page.get_by_test_id("ede-list-row")` natively. The drivers themselves resolve via
CSS (`ui/viewids.vid(...)`), so they don't depend on that registration.

## The contract

| `data-viewid` | Keyed attrs | Emitted by (`managers/`) | Driven by |
|---|---|---|---|
| `ede-appswitcher-toggle` | — | `AppSwitcher.tsx` | `WebClient.open_app` |
| `ede-appswitcher-app` | `data-app-id` | `AppSwitcher.tsx` | `WebClient.open_app` (by visible name or `app_id`) |
| `ede-menu-item` | `data-menu-key` | `ModuleMenuManager.tsx` | `WebClient.open_menu` (by visible name or `menu_key`) |
| `ede-list` | — | `ListView.tsx` | `ListDriver.root` |
| `ede-action-create` | — | `ActionControlPanel.tsx` | `ListDriver.create` |
| `ede-list-row` | `data-record-id`, `data-record-name` | `ListView.tsx` | `ListDriver.row` / `open_record` |
| `ede-list-cell` | `data-field` | `ListView.tsx` | (cell-level assertions) |
| `ede-list-row-action` | `data-action` | `ListView.tsx` | `RowHandle.action` |
| `ede-search-input` | — | `SearchPanel.tsx` | `SearchDriver.type` / `search` |
| `ede-search-toggle` | — | `SearchPanel.tsx` | `SearchDriver.open_panel` |
| `ede-search-filter` | `data-filter-key` | `SearchPanel.tsx` | `SearchDriver.toggle_filter` |
| `ede-search-groupby` | `data-groupby-key` | `SearchPanel.tsx` | `SearchDriver.group_by` |
| `ede-search-chip` | `data-filter-key` | `SearchPanel.tsx` | `SearchDriver.remove_chip` |
| `ede-form` | — | `FormView.tsx` | `FormDriver.root` |
| `ede-field` | `data-field` | `FormSheet.tsx` | `FormDriver.set` / `get` |
| `ede-form-save` | — | `FormControlPanel.tsx` | `FormDriver.save` |
| `ede-form-discard` | — | `FormControlPanel.tsx` | `FormDriver.discard` |
| `ede-statusbar-action` | `data-transition` | `FormHeader.tsx` | `FormDriver.statusbar_action` |
| `ede-statusbar-stage` | `data-stage` | `FormHeader.tsx` | `FormDriver.current_stage` |
| `ede-kanban` | — | `KanbanView.tsx` | `KanbanDriver.root` |
| `ede-kanban-column` | `data-stage` | `KanbanView.tsx` | `KanbanDriver.column` / drag target |
| `ede-kanban-card` | `data-record-id` | `KanbanView.tsx` | `KanbanDriver.card` / `drag_to` |
| `ede-chatter-composer` | — | `RecordComposer.tsx` | `ChatterDriver.post` |
| `ede-chatter-send` | — | `RecordComposer.tsx` | `ChatterDriver.post` |
| `ede-chatter-message` | `data-message-id` | `RecordTimeline.tsx` | `ChatterDriver.messages` / `wait_for_message` |
| `ede-notification-bell` | — | `UserNotification.tsx` | `NotificationsDriver.open` |
| `ede-notification-item` | `data-notification-id` | `UserNotification.tsx` | `NotificationsDriver.items` / `click_item` |
| `ede-notification-mark-read` | — | `UserNotification.tsx` | `NotificationsDriver.mark_all_read` |
| `ede-dialog-confirm` | — | `ConfirmDialog.tsx` | `DialogDriver.confirm` |
| `ede-dialog-cancel` | — | `ConfirmDialog.tsx` | `DialogDriver.cancel` |

## Authoring a test

Hold one `WebClient` (the `web_client` fixture) and drive by intent:

```python
import pytest


@pytest.mark.e2e
@pytest.mark.qa_module("foundation.base")
@pytest.mark.brs("FND-BASE-01")
class TestCreateMember:
    def test_create(self, web_client):
        wc = web_client
        wc.open_app("Members")
        wc.list.create()
        wc.form.set("name", "Acme Industries").save()
        # Assert against the committed DB state — no time.sleep, no poll loop.
        wc.wait_for_orm("res.partner", [("name", "=", "Acme Industries")])
```

No CSS/text selector appears anywhere. If a flow needs a hook the contract
doesn't have yet, **add a `data-viewid`** to the relevant manager + a constant in
`viewids.py` + a row here — never reach into the DOM from the test.

## Adding a new hook

1. Add the attribute to the manager's JSX (`data-viewid="ede-<surface>-<role>"`,
   plus any keyed `data-*`).
2. Add the matching constant to `ui/viewids.py`.
3. Add the driver method (or extend an existing driver) under `ui/drivers/`.
4. Add a row to the table above.
5. `cd src/frontend && bun run build && bun run test` must stay green.

---

*Governed by the [`authoring-e2e-tests`](../.claude/skills/authoring-e2e-tests/SKILL.md) skill.*
