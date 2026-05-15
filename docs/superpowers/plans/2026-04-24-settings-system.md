# Settings System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a pluggable, XML-driven runtime settings system where any module drops a `views/*_settings.xml` file and its settings automatically appear in the General Settings screen.

**Architecture:** A `SettingsRegistry` parses `<settings>` XML files at boot and attaches to `Env`; a `SettingsService` resolves values with an org → system → XML-default fallback chain; a REST API exposes schema+values; the frontend renders a schema-driven settings pane registered as a `client` ir.action.

**Tech Stack:** Python 3.10+, lxml-free XML parsing (stdlib `xml.etree.ElementTree`), `cryptography.fernet` for secret fields, React 19 + TypeScript + Tailwind CSS 4, existing `httpClient` pattern.

**Spec:** `docs/superpowers/specs/2026-04-24-settings-system-design.md`

---

## File Map

### New — Backend
| File | Responsibility |
|---|---|
| `src/ede/core/services/settings/__init__.py` | Package marker |
| `src/ede/core/services/settings/types.py` | `SettingsFieldDescriptor`, `SettingsSectionDescriptor`, `SettingsModuleDescriptor` dataclasses |
| `src/ede/core/services/settings/registry.py` | `SettingsRegistry` — scans manifest views, parses `<settings>` XML |
| `src/ede/core/services/settings/encryption.py` | `encrypt_value` / `decrypt_value` using Fernet |
| `src/ede/core/services/settings/service.py` | `SettingsService` — value resolution, set, audit, schema serialization |
| `src/ede/foundation/base/models/ir_config.py` | `ir.config` DomainModel (key-value store) |
| `src/ede/foundation/base/models/ir_config_log.py` | `ir.config.log` DomainModel (audit trail) |
| `src/ede/foundation/base/api/settings_routes.py` | Settings REST controller (GET/PATCH /api/settings) |
| `src/ede/foundation/base/views/base_settings.xml` | Base module settings declarations |
| `src/tests/settings/__init__.py` | Test package marker |
| `src/tests/settings/test_settings_registry.py` | Registry XML parsing tests |
| `src/tests/settings/test_encryption.py` | Encryption round-trip tests |
| `src/tests/settings/test_settings_service.py` | Service resolution chain + audit log tests |

### Modified — Backend
| File | Change |
|---|---|
| `src/ede/foundation/base/models/__init__.py` | Import `ir_config`, `ir_config_log` |
| `src/ede/foundation/base/api/__init__.py` | Import `settings_routes` |
| `src/ede/foundation/base/__manifest__.py` | Add `views/base_settings.xml` to `data` list |
| `src/ede/foundation/base/data/base_menus.xml` | Add `ir.action` + `ir.menu` for General Settings |
| `src/ede/core/bootstrap.py` | Build `SettingsRegistry`, add to `BootEnvironment` |
| `src/ede/core/types.py` (BootEnvironment) | Add `settings_registry` field |
| `src/ede/core/env.py` | Add `settings_registry` param + `get_setting()` + propagate in clone methods |
| `src/ede/cli/commands/server.py` | Pass `settings_registry` to `base_env` |
| `src/ede/cli/commands/gateway.py` | Pass `settings_registry` to `base_env` |
| `src/ede/core/services/presentation/view_registry.py` | Skip files whose root element is `<settings>` |

### New — Frontend
| File | Responsibility |
|---|---|
| `src/frontend/src/workspace/settings/types.ts` | `ModuleNavEntry`, `SettingsField`, `ModuleSettings` TS types |
| `src/frontend/src/workspace/services/SettingsApiService.ts` | `listModules`, `getModule`, `saveModule` API calls |
| `src/frontend/src/workspace/settings/SettingsPage.tsx` | Root component, registered as `"settings.general"` client action |
| `src/frontend/src/workspace/settings/SettingsNav.tsx` | Sidebar with category groups, dirty guard |
| `src/frontend/src/workspace/settings/SettingsPane.tsx` | Module pane — fetches schema+values, local form state, SaveBar |
| `src/frontend/src/workspace/settings/SettingsSection.tsx` | Section heading + field list |
| `src/frontend/src/workspace/settings/SettingsField.tsx` | Field renderer — label, scope badge, depends_on, delegates to widget |
| `src/frontend/src/workspace/settings/fields/SettingsToggle.tsx` | Boolean toggle widget |
| `src/frontend/src/workspace/settings/fields/SettingsTextInput.tsx` | Char + secret (password) widget |
| `src/frontend/src/workspace/settings/fields/SettingsNumberInput.tsx` | Integer number input |
| `src/frontend/src/workspace/settings/fields/SettingsSelect.tsx` | Selection dropdown |
| `src/frontend/src/workspace/settings/fields/SettingsMany2One.tsx` | Async searchable many2one picker |
| `src/frontend/src/workspace/settings/MigrationStub.tsx` | v1 stub for migration-required fields |

### Modified — Frontend
| File | Change |
|---|---|
| `src/frontend/src/workspace/components/action/ClientActionRegistry.tsx` | Register `"settings.general"` → `SettingsPage` |

---

## Task 1: Settings Service Package — Types + Registry

**Files:**
- Create: `src/ede/core/services/settings/__init__.py`
- Create: `src/ede/core/services/settings/types.py`
- Create: `src/ede/core/services/settings/registry.py`
- Create: `src/tests/settings/__init__.py`
- Create: `src/tests/settings/test_settings_registry.py`

- [ ] **Step 1: Write the failing tests**

Create `src/tests/settings/__init__.py` (empty), then:

```python
# src/tests/settings/test_settings_registry.py
"""Tests for SettingsRegistry XML parsing."""
from __future__ import annotations

import textwrap
import pytest

from ede.core.services.settings.registry import SettingsRegistry, SettingsParseError


# ── helpers ───────────────────────────────────────────────────────────────────

class _FakeApp:
    def __init__(self, app_root_dir: str, data_files: list[str]) -> None:
        self.app_root_dir = app_root_dir
        self.data_files = data_files


def _write_xml(tmp_path, filename: str, content: str) -> str:
    views_dir = tmp_path / "views"
    views_dir.mkdir(parents=True, exist_ok=True)
    path = views_dir / filename
    path.write_text(textwrap.dedent(content))
    return str(path)


# ── tests ─────────────────────────────────────────────────────────────────────

class TestSettingsRegistry:
    def test_parses_basic_settings_file(self, tmp_path):
        xml_path = _write_xml(tmp_path, "email_settings.xml", """
            <settings module="email" label="Email" icon="mail" category="Services" order="20">
              <section string="Features">
                <field name="email.notifications" type="boolean" scope="organization"
                       label="Enable Notifications" default="true"/>
              </section>
            </settings>
        """)
        app = _FakeApp(str(tmp_path), [xml_path])
        reg = SettingsRegistry()
        reg.load_from_app(app)

        mod = reg.get_module("email")
        assert mod is not None
        assert mod.label == "Email"
        assert mod.icon == "mail"
        assert mod.category == "Services"
        assert mod.order == 20
        assert len(mod.sections) == 1
        assert mod.sections[0].string == "Features"
        fld = mod.sections[0].fields[0]
        assert fld.name == "email.notifications"
        assert fld.field_type == "boolean"
        assert fld.scope == "organization"
        assert fld.default == "true"
        assert fld.secret is False
        assert fld.requires_migration is False

    def test_ignores_ede_view_xml_files(self, tmp_path):
        xml_path = _write_xml(tmp_path, "res_user_views.xml", """
            <ede version="1.0">
              <view id="res.user.list" model="res.user" type="list"></view>
            </ede>
        """)
        app = _FakeApp(str(tmp_path), [xml_path])
        reg = SettingsRegistry()
        reg.load_from_app(app)
        assert reg.list_modules() == []

    def test_ignores_files_outside_views_dir(self, tmp_path):
        data_dir = tmp_path / "data"
        data_dir.mkdir()
        xml_path = data_dir / "email_settings.xml"
        xml_path.write_text("<settings module='email' label='Email'></settings>")
        app = _FakeApp(str(tmp_path), [str(xml_path)])
        reg = SettingsRegistry()
        reg.load_from_app(app)
        assert reg.list_modules() == []

    def test_list_modules_sorted_by_order_then_key(self, tmp_path):
        for mod_key, order in [("z_mod", 50), ("a_mod", 10)]:
            _write_xml(tmp_path, f"{mod_key}_settings.xml", f"""
                <settings module="{mod_key}" label="{mod_key}" order="{order}">
                  <section string="S">
                    <field name="{mod_key}.f" type="boolean" scope="system" label="F"/>
                  </section>
                </settings>
            """)
        files = [
            str(tmp_path / "views" / "z_mod_settings.xml"),
            str(tmp_path / "views" / "a_mod_settings.xml"),
        ]
        reg = SettingsRegistry()
        reg.load_from_app(_FakeApp(str(tmp_path), files))
        modules = reg.list_modules()
        assert modules[0].module == "a_mod"
        assert modules[1].module == "z_mod"

    def test_selection_field_options_parsed(self, tmp_path):
        xml_path = _write_xml(tmp_path, "test_settings.xml", """
            <settings module="test" label="Test">
              <section string="S">
                <field name="test.signup" type="selection" scope="system" label="Signup Method">
                  <option value="open" label="Open Registration"/>
                  <option value="invite_only" label="Invite Only"/>
                </field>
              </section>
            </settings>
        """)
        reg = SettingsRegistry()
        reg.load_from_app(_FakeApp(str(tmp_path), [xml_path]))
        fld = reg.get_module("test").sections[0].fields[0]
        assert len(fld.options) == 2
        assert fld.options[0] == {"value": "open", "label": "Open Registration"}

    def test_raises_on_invalid_depends_on(self, tmp_path):
        xml_path = _write_xml(tmp_path, "bad_settings.xml", """
            <settings module="bad" label="Bad">
              <section string="S">
                <field name="bad.foo" type="char" scope="system" label="Foo"
                       depends_on="bad.nonexistent_boolean"/>
              </section>
            </settings>
        """)
        reg = SettingsRegistry()
        with pytest.raises(SettingsParseError, match="depends_on"):
            reg.load_from_app(_FakeApp(str(tmp_path), [xml_path]))

    def test_depends_on_passes_when_boolean_exists(self, tmp_path):
        xml_path = _write_xml(tmp_path, "ok_settings.xml", """
            <settings module="ok" label="OK">
              <section string="S">
                <field name="ok.enabled" type="boolean" scope="system" label="Enabled"/>
                <field name="ok.detail" type="char" scope="system" label="Detail"
                       depends_on="ok.enabled"/>
              </section>
            </settings>
        """)
        reg = SettingsRegistry()
        reg.load_from_app(_FakeApp(str(tmp_path), [xml_path]))
        fld = reg.get_module("ok").sections[0].fields[1]
        assert fld.depends_on == "ok.enabled"

    def test_get_field_by_key(self, tmp_path):
        xml_path = _write_xml(tmp_path, "x_settings.xml", """
            <settings module="x" label="X">
              <section string="S">
                <field name="x.secret_key" type="char" scope="organization"
                       label="API Key" secret="true"/>
              </section>
            </settings>
        """)
        reg = SettingsRegistry()
        reg.load_from_app(_FakeApp(str(tmp_path), [xml_path]))
        fld = reg.get_field("x.secret_key")
        assert fld is not None
        assert fld.secret is True
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /home/dharmang/personal/ede-frame/repository/ede-framework
pytest src/tests/settings/test_settings_registry.py -v 2>&1 | head -20
```

Expected: `ModuleNotFoundError: No module named 'ede.core.services.settings'`

- [ ] **Step 3: Create the package and types**

```python
# src/ede/core/services/settings/__init__.py
# (empty)
```

```python
# src/ede/core/services/settings/types.py
from __future__ import annotations
from dataclasses import dataclass, field


@dataclass
class SettingsFieldDescriptor:
    name: str
    field_type: str          # boolean | integer | char | selection | many2one
    scope: str               # system | organization
    label: str
    default: str | None = None
    model: str | None = None
    domain: str | None = None
    secret: bool = False
    requires_migration: bool = False
    depends_on: str | None = None
    readonly: bool = False
    options: list[dict] = field(default_factory=list)


@dataclass
class SettingsSectionDescriptor:
    string: str
    fields: list[SettingsFieldDescriptor] = field(default_factory=list)


@dataclass
class SettingsModuleDescriptor:
    module: str
    label: str
    icon: str | None = None
    category: str = "General"
    order: int = 100
    sections: list[SettingsSectionDescriptor] = field(default_factory=list)
```

- [ ] **Step 4: Create the SettingsRegistry**

```python
# src/ede/core/services/settings/registry.py
from __future__ import annotations

import os
import xml.etree.ElementTree as ET
from typing import Any, Iterable

from .types import SettingsFieldDescriptor, SettingsSectionDescriptor, SettingsModuleDescriptor

_VALID_FIELD_TYPES = frozenset({"boolean", "integer", "char", "selection", "many2one"})
_VALID_SCOPES = frozenset({"system", "organization"})


class SettingsParseError(Exception):
    pass


class SettingsRegistry:
    def __init__(self) -> None:
        self._modules: dict[str, SettingsModuleDescriptor] = {}

    # ── public ────────────────────────────────────────────────────────────────

    def load_from_app(self, app_spec: Any) -> None:
        """Scan app manifest data_files for views/*.xml with <settings> root."""
        app_root_dir = os.path.abspath(str(app_spec.app_root_dir))
        views_dir = os.path.abspath(os.path.join(app_root_dir, "views")) + os.sep
        data_files: Iterable[str] = getattr(app_spec, "data_files", ()) or ()

        for raw_path in data_files:
            abs_path = os.path.abspath(str(raw_path))
            if not abs_path.endswith(".xml"):
                continue
            if not abs_path.startswith(views_dir):
                continue
            try:
                tree = ET.parse(abs_path)
                root = tree.getroot()
            except ET.ParseError:
                continue
            if root.tag != "settings":
                continue
            descriptor = self._parse_module(root, abs_path)
            self._modules[descriptor.module] = descriptor

    def get_module(self, module_key: str) -> SettingsModuleDescriptor | None:
        return self._modules.get(module_key)

    def list_modules(self) -> list[SettingsModuleDescriptor]:
        return sorted(self._modules.values(), key=lambda m: (m.order, m.module))

    def get_field(self, key: str) -> SettingsFieldDescriptor | None:
        for module in self._modules.values():
            for section in module.sections:
                for fld in section.fields:
                    if fld.name == key:
                        return fld
        return None

    def all_field_keys(self) -> set[str]:
        return {f.name for m in self._modules.values() for s in m.sections for f in s.fields}

    # ── private ───────────────────────────────────────────────────────────────

    def _parse_module(self, root: ET.Element, source: str) -> SettingsModuleDescriptor:
        module = root.attrib.get("module", "").strip()
        if not module:
            raise SettingsParseError(f"<settings> missing 'module' attribute in {source}")
        sections = [self._parse_section(el, source) for el in root.findall("section")]
        boolean_keys = {
            f.name for s in sections for f in s.fields if f.field_type == "boolean"
        }
        for section in sections:
            for fld in section.fields:
                if fld.depends_on and fld.depends_on not in boolean_keys:
                    raise SettingsParseError(
                        f"Field '{fld.name}' in module '{module}' has "
                        f"depends_on='{fld.depends_on}' which is not a boolean "
                        f"field in this module (in {source})"
                    )
        return SettingsModuleDescriptor(
            module=module,
            label=root.attrib.get("label", module),
            icon=root.attrib.get("icon"),
            category=root.attrib.get("category", "General"),
            order=int(root.attrib.get("order", "100")),
            sections=sections,
        )

    def _parse_section(self, el: ET.Element, source: str) -> SettingsSectionDescriptor:
        return SettingsSectionDescriptor(
            string=el.attrib.get("string", ""),
            fields=[self._parse_field(f, source) for f in el.findall("field")],
        )

    def _parse_field(self, el: ET.Element, source: str) -> SettingsFieldDescriptor:
        name = el.attrib.get("name", "").strip()
        if not name:
            raise SettingsParseError(f"<field> missing 'name' in {source}")
        field_type = el.attrib.get("type", "")
        if field_type not in _VALID_FIELD_TYPES:
            raise SettingsParseError(
                f"Unknown type '{field_type}' on field '{name}' in {source}"
            )
        scope = el.attrib.get("scope", "")
        if scope not in _VALID_SCOPES:
            raise SettingsParseError(
                f"Invalid scope '{scope}' on field '{name}' in {source}"
            )
        options = [
            {"value": opt.attrib.get("value", ""), "label": opt.attrib.get("label", "")}
            for opt in el.findall("option")
        ]
        return SettingsFieldDescriptor(
            name=name,
            field_type=field_type,
            scope=scope,
            label=el.attrib.get("label", name),
            default=el.attrib.get("default"),
            model=el.attrib.get("model"),
            domain=el.attrib.get("domain"),
            secret=el.attrib.get("secret", "false") == "true",
            requires_migration=el.attrib.get("requires_migration", "false") == "true",
            depends_on=el.attrib.get("depends_on"),
            readonly=el.attrib.get("readonly", "false") == "true",
            options=options,
        )
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
pytest src/tests/settings/test_settings_registry.py -v
```

Expected: All 8 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/ede/core/services/settings/ src/tests/settings/
git commit -m "$(cat <<'EOF'
[NEW] foundation: settings service package — types + SettingsRegistry XML parser

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Encryption Helper

**Files:**
- Create: `src/ede/core/services/settings/encryption.py`
- Create: `src/tests/settings/test_encryption.py`

- [ ] **Step 1: Write the failing tests**

```python
# src/tests/settings/test_encryption.py
"""Tests for Fernet-based settings encryption helper."""
from __future__ import annotations
import pytest
from ede.core.services.settings.encryption import encrypt_value, decrypt_value


class TestEncryption:
    def test_round_trip_returns_original_plaintext(self):
        token = encrypt_value("my-secret-value", "my-app-key")
        assert decrypt_value(token, "my-app-key") == "my-secret-value"

    def test_different_calls_produce_different_tokens(self):
        token1 = encrypt_value("same", "key")
        token2 = encrypt_value("same", "key")
        # Fernet tokens include a timestamp salt — same input → different ciphertext
        assert token1 != token2

    def test_decrypt_wrong_key_raises(self):
        from cryptography.fernet import InvalidToken
        token = encrypt_value("secret", "correct-key")
        with pytest.raises((InvalidToken, Exception)):
            decrypt_value(token, "wrong-key")

    def test_empty_string_round_trip(self):
        token = encrypt_value("", "key")
        assert decrypt_value(token, "key") == ""

    def test_unicode_round_trip(self):
        plaintext = "pässwörd-with-ünïcödé"
        token = encrypt_value(plaintext, "key")
        assert decrypt_value(token, "key") == plaintext
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
pytest src/tests/settings/test_encryption.py -v 2>&1 | head -15
```

Expected: `ImportError: cannot import name 'encrypt_value'`

- [ ] **Step 3: Verify cryptography is available**

```bash
python -c "from cryptography.fernet import Fernet; print('ok')"
```

If this fails, run: `pip install cryptography`. Also add to `pyproject.toml` under `[project.dependencies]`: `"cryptography>=42.0.0"`.

- [ ] **Step 4: Implement the encryption helper**

```python
# src/ede/core/services/settings/encryption.py
from __future__ import annotations

import base64
import hashlib


def _derive_fernet_key(secret_key: str) -> bytes:
    """Derive a 32-byte URL-safe base64-encoded key from an arbitrary string."""
    raw = hashlib.sha256(secret_key.encode("utf-8")).digest()
    return base64.urlsafe_b64encode(raw)


def encrypt_value(plaintext: str, secret_key: str) -> str:
    """Encrypt plaintext using Fernet. Returns base64 token string."""
    from cryptography.fernet import Fernet
    fernet = Fernet(_derive_fernet_key(secret_key))
    return fernet.encrypt(plaintext.encode("utf-8")).decode("utf-8")


def decrypt_value(token: str, secret_key: str) -> str:
    """Decrypt a Fernet token. Raises InvalidToken if key or token is wrong."""
    from cryptography.fernet import Fernet
    fernet = Fernet(_derive_fernet_key(secret_key))
    return fernet.decrypt(token.encode("utf-8")).decode("utf-8")
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
pytest src/tests/settings/test_encryption.py -v
```

Expected: All 5 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add src/ede/core/services/settings/encryption.py src/tests/settings/test_encryption.py
git commit -m "$(cat <<'EOF'
[ADD] foundation: Fernet encryption helper for secret settings values

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: ir.config + ir.config.log Models

**Files:**
- Create: `src/ede/foundation/base/models/ir_config.py`
- Create: `src/ede/foundation/base/models/ir_config_log.py`
- Modify: `src/ede/foundation/base/models/__init__.py`

- [ ] **Step 1: Create ir.config model**

```python
# src/ede/foundation/base/models/ir_config.py
from __future__ import annotations

from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model("ir.config", description="Runtime Configuration Entry")
class IrConfig(DomainModel):
    """
    Key-value store for runtime settings.

    key             Dotted key e.g. 'email.feature.notifications'
    value           Serialized plaintext value (null when encrypted_value is set)
    encrypted_value Fernet-encrypted blob for secret=true fields
    scope           'system' (global) or 'organization' (per-tenant)
    organization_id UUID string of res.organization for org-scoped rows; empty for system
    """

    key = fields.Char(max_length=255, required=True, index=True, label="Key")
    value = fields.Char(max_length=8000, required=False, label="Value")
    encrypted_value = fields.Char(max_length=8000, required=False, label="Encrypted Value")
    scope = fields.Enum(
        selection=[("system", "System"), ("organization", "Organization")],
        required=True,
        label="Scope",
    )
    # Stores res.organization.record_uuid string; empty string for system-scope rows.
    # Using Char avoids a hard FK dependency so ir.config can be queried standalone.
    organization_id = fields.Char(max_length=36, required=False, label="Organization ID")
```

- [ ] **Step 2: Create ir.config.log model**

```python
# src/ede/foundation/base/models/ir_config_log.py
from __future__ import annotations

from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model("ir.config.log", description="Settings Change Audit Log")
class IrConfigLog(DomainModel):
    """
    Immutable audit record for every ir.config change.
    created_at_utc (auto field) serves as the change timestamp.
    """

    key = fields.Char(max_length=255, required=True, index=True, label="Key")
    old_value = fields.Char(max_length=8000, required=False, label="Old Value")
    new_value = fields.Char(max_length=8000, required=False, label="New Value")
    scope = fields.Enum(
        selection=[("system", "System"), ("organization", "Organization")],
        required=True,
        label="Scope",
    )
    organization_id = fields.Char(max_length=36, required=False, label="Organization ID")
    changed_by = fields.Char(max_length=255, required=False, label="Changed By")
```

- [ ] **Step 3: Register models in __init__.py**

Open `src/ede/foundation/base/models/__init__.py` and add at the end:

```python
from . import ir_config       # ir.config — runtime key-value settings store
from . import ir_config_log   # ir.config.log — settings change audit trail
```

The file currently ends at the `audit` import. Add after it:

```python
# existing last lines:
from . import notification_setting
from . import audit    # ir.rbac.decision.log, ir.rbac.binding.change.log
# ADD these two lines:
from . import ir_config       # ir.config — runtime key-value settings store
from . import ir_config_log   # ir.config.log — settings change audit trail
```

- [ ] **Step 4: Generate migrations**

```bash
ede migrate generate -m "add ir.config and ir.config.log" --config ede.conf
```

Expected: Two new migration files in the base app's migration versions directory. Verify they contain `CREATE TABLE` statements for `ir_config` and `ir_config_log`.

- [ ] **Step 5: Apply migrations (system tenant)**

```bash
ede migrate upgrade --config ede.conf
```

Expected: `INFO: Running upgrade ... -> <hash> (add ir.config and ir.config.log)`

- [ ] **Step 6: Verify models are importable**

```bash
python -c "from ede.foundation.base.models.ir_config import IrConfig; from ede.foundation.base.models.ir_config_log import IrConfigLog; print('ok')"
```

Expected: `ok`

- [ ] **Step 7: Commit**

```bash
git add src/ede/foundation/base/models/ir_config.py \
        src/ede/foundation/base/models/ir_config_log.py \
        src/ede/foundation/base/models/__init__.py \
        src/ede/foundation/base/migrations/
git commit -m "$(cat <<'EOF'
[NEW] foundation.base: ir.config + ir.config.log models for runtime settings storage

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: SettingsService

**Files:**
- Create: `src/ede/core/services/settings/service.py`
- Create: `src/tests/settings/test_settings_service.py`

- [ ] **Step 1: Write the failing tests**

```python
# src/tests/settings/test_settings_service.py
"""Integration tests for SettingsService value resolution and persistence."""
from __future__ import annotations

from dataclasses import dataclass
from pathlib import Path
import pytest

from ede.core.adapters.persistence.sqlalchemy.generic_repo import _clear_metadata_cache
from ede.core.adapters.persistence.sqlalchemy.metadata_builder import SqlAlchemyMetadataBuilder
from ede.core.adapters.persistence.sqlalchemy.sqlalchemy_provider import SqlAlchemyPersistenceProvider
from ede.core.env import Env
from ede.core.kernel.model import DomainModel
from ede.core.registry import Registry
from ede.core.services.persistence.tenant_identity import PrefixTenantDatabaseIdentityResolver
from ede.core.tenancy.context import set_current_tenant_id
from ede.foundation.base.models.ir_config import IrConfig
from ede.foundation.base.models.ir_config_log import IrConfigLog
from ede.core.services.settings.types import (
    SettingsFieldDescriptor, SettingsSectionDescriptor, SettingsModuleDescriptor,
)
from ede.core.services.settings.registry import SettingsRegistry
from ede.core.services.settings.service import SettingsService, SettingsMigrationRequired


_TENANT = "test_settings_tenant"


@dataclass(frozen=True)
class _FakeSettings:
    DATABASE_ENGINE: str = "sqlite"
    DATABASE_NAME_PREFIX: str = "db_ede"
    SQLITE_TENANT_DATABASE_DIR: str = "/tmp/ede-test-settings"
    POSTGRES_HOST: str = "localhost"
    POSTGRES_PORT: int = 5432
    POSTGRES_USER: str = "postgres"
    POSTGRES_PASSWORD: str = "postgres"


def _bool_field(name: str, scope: str = "system", default: str = "false") -> SettingsFieldDescriptor:
    return SettingsFieldDescriptor(name=name, field_type="boolean", scope=scope, label=name, default=default)


def _char_field(name: str, scope: str = "system") -> SettingsFieldDescriptor:
    return SettingsFieldDescriptor(name=name, field_type="char", scope=scope, label=name)


def _secret_field(name: str) -> SettingsFieldDescriptor:
    return SettingsFieldDescriptor(name=name, field_type="char", scope="organization", label=name, secret=True)


def _migration_field(name: str) -> SettingsFieldDescriptor:
    return SettingsFieldDescriptor(
        name=name, field_type="many2one", scope="organization", label=name, requires_migration=True
    )


def _make_registry(*fields: SettingsFieldDescriptor) -> SettingsRegistry:
    reg = SettingsRegistry()
    # Manually inject descriptors (bypass XML parsing)
    from ede.core.services.settings.types import SettingsSectionDescriptor, SettingsModuleDescriptor
    mod = SettingsModuleDescriptor(
        module="test",
        label="Test",
        sections=[SettingsSectionDescriptor(string="S", fields=list(fields))],
    )
    reg._modules["test"] = mod
    return reg


def _build_env(tmp_path: Path) -> Env:
    _clear_metadata_cache()
    set_current_tenant_id(_TENANT)
    tenant_dir = tmp_path / "tenants"
    tenant_dir.mkdir(parents=True, exist_ok=True)
    settings = _FakeSettings(SQLITE_TENANT_DATABASE_DIR=str(tenant_dir))

    registry = Registry()
    registry.register_command_handlers(DomainModel)
    for model_cls in [IrConfig, IrConfigLog]:
        registry.register_model(model_cls)

    resolver = PrefixTenantDatabaseIdentityResolver.from_settings(settings)
    meta_result = SqlAlchemyMetadataBuilder().build(
        model_classes=[IrConfig, IrConfigLog],
        database_engine="sqlite",
        skip_audit_fks=True,
    )
    from sqlalchemy import create_engine as _ce
    identity = resolver.resolve(_TENANT)
    db_url = f"sqlite+pysqlite:///{tenant_dir}/{identity.database_name}.db"
    _ce(db_url).connect()  # ensure file exists
    meta_result.metadata.create_all(_ce(db_url))

    provider = SqlAlchemyPersistenceProvider.from_settings(
        settings=settings,
        tenant_database_identity_resolver=resolver,
        tenant_id_getter=lambda: _TENANT,
    )
    return Env(registry=registry, persistence=provider, tenant_id=_TENANT)


@pytest.fixture()
def env(tmp_path: Path):
    e = _build_env(tmp_path)
    yield e
    set_current_tenant_id(None)


class TestSettingsServiceGetValue:
    def test_returns_none_when_no_row_and_no_default(self, env):
        reg = _make_registry(_char_field("test.foo"))
        svc = SettingsService(reg)
        assert svc.get_value("test.foo", env) is None

    def test_returns_xml_default_when_no_row(self, env):
        reg = _make_registry(_bool_field("test.flag", default="true"))
        svc = SettingsService(reg)
        assert svc.get_value("test.flag", env) is True

    def test_returns_fallback_param_when_key_unknown(self, env):
        reg = SettingsRegistry()
        svc = SettingsService(reg)
        assert svc.get_value("unknown.key", env, default="fallback") == "fallback"

    def test_reads_system_scoped_row(self, env):
        reg = _make_registry(_char_field("test.color", scope="system"))
        svc = SettingsService(reg)
        svc.set_value("test.color", "blue", "system", None, env)
        assert svc.get_value("test.color", env) == "blue"

    def test_org_row_takes_precedence_over_system_row(self, env):
        reg = _make_registry(_char_field("test.lang", scope="organization"))
        svc = SettingsService(reg)
        svc.set_value("test.lang", "en", "system", None, env)
        svc.set_value("test.lang", "fr", "organization", _TENANT, env)
        env_with_principal = env.with_principal({"tenant_id": _TENANT})
        assert svc.get_value("test.lang", env_with_principal) == "fr"

    def test_falls_back_to_system_when_no_org_row(self, env):
        reg = _make_registry(_char_field("test.lang", scope="organization"))
        svc = SettingsService(reg)
        svc.set_value("test.lang", "en", "system", None, env)
        env_with_principal = env.with_principal({"tenant_id": _TENANT})
        assert svc.get_value("test.lang", env_with_principal) == "en"

    def test_boolean_deserializes_to_python_bool(self, env):
        reg = _make_registry(_bool_field("test.enabled"))
        svc = SettingsService(reg)
        svc.set_value("test.enabled", True, "system", None, env)
        result = svc.get_value("test.enabled", env)
        assert result is True
        assert isinstance(result, bool)

    def test_integer_deserializes_to_int(self, env):
        from ede.core.services.settings.types import SettingsFieldDescriptor
        fld = SettingsFieldDescriptor(name="test.count", field_type="integer", scope="system", label="Count")
        reg = _make_registry(fld)
        svc = SettingsService(reg)
        svc.set_value("test.count", 42, "system", None, env)
        assert svc.get_value("test.count", env) == 42


class TestSettingsServiceSetValue:
    def test_raises_migration_required_for_migration_fields(self, env):
        reg = _make_registry(_migration_field("test.provider"))
        svc = SettingsService(reg)
        with pytest.raises(SettingsMigrationRequired):
            svc.set_value("test.provider", "some-uuid", "organization", _TENANT, env)

    def test_secret_field_stores_encrypted_not_plaintext(self, env):
        import os; os.environ["JWT_SECRET_KEY"] = "test-jwt-secret"
        reg = _make_registry(_secret_field("test.api_key"))
        svc = SettingsService(reg, jwt_secret_key="test-jwt-secret")
        svc.set_value("test.api_key", "plain-secret", "organization", _TENANT, env)

        rows = env.models["ir.config"].search([("key", "=", "test.api_key")], limit=1)
        assert rows[0].value is None
        assert rows[0].encrypted_value is not None
        assert rows[0].encrypted_value != "plain-secret"

    def test_secret_field_round_trips_correctly(self, env):
        reg = _make_registry(_secret_field("test.pwd"))
        svc = SettingsService(reg, jwt_secret_key="test-key")
        svc.set_value("test.pwd", "s3cr3t!", "organization", _TENANT, env)
        env_with_principal = env.with_principal({"tenant_id": _TENANT})
        assert svc.get_value("test.pwd", env_with_principal) == "s3cr3t!"

    def test_upserts_existing_row(self, env):
        reg = _make_registry(_char_field("test.color"))
        svc = SettingsService(reg)
        svc.set_value("test.color", "red", "system", None, env)
        svc.set_value("test.color", "blue", "system", None, env)
        rows = env.models["ir.config"].search([("key", "=", "test.color")])
        assert len(rows) == 1
        assert rows[0].value == "blue"


class TestSettingsServiceAuditLog:
    def test_audit_log_written_on_set(self, env):
        reg = _make_registry(_char_field("test.x"))
        svc = SettingsService(reg)
        svc.set_value("test.x", "hello", "system", None, env)
        logs = env.models["ir.config.log"].search([("key", "=", "test.x")])
        assert len(logs) == 1
        assert logs[0].new_value == "hello"
        assert logs[0].scope == "system"

    def test_audit_log_captures_old_value_on_update(self, env):
        reg = _make_registry(_char_field("test.y"))
        svc = SettingsService(reg)
        svc.set_value("test.y", "old", "system", None, env)
        svc.set_value("test.y", "new", "system", None, env)
        logs = env.models["ir.config.log"].search([("key", "=", "test.y")])
        assert len(logs) == 2
        second_log = sorted(logs, key=lambda r: r.created_at_utc)[-1]
        assert second_log.old_value == "old"
        assert second_log.new_value == "new"

    def test_secret_field_audit_logs_masked_values(self, env):
        reg = _make_registry(_secret_field("test.secret"))
        svc = SettingsService(reg, jwt_secret_key="k")
        svc.set_value("test.secret", "plaintext", "organization", _TENANT, env)
        logs = env.models["ir.config.log"].search([("key", "=", "test.secret")])
        assert logs[0].new_value == "••••••"
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
pytest src/tests/settings/test_settings_service.py -v 2>&1 | head -20
```

Expected: `ImportError: cannot import name 'SettingsService'`

- [ ] **Step 3: Implement SettingsService**

```python
# src/ede/core/services/settings/service.py
from __future__ import annotations

from typing import Any, Optional, TYPE_CHECKING

if TYPE_CHECKING:
    from ede.core.env import Env
    from .registry import SettingsRegistry


class SettingsMigrationRequired(Exception):
    def __init__(self, key: str) -> None:
        self.key = key
        super().__init__(f"Setting '{key}' requires a migration to change.")


class SettingsService:
    def __init__(
        self,
        registry: "SettingsRegistry",
        *,
        jwt_secret_key: str = "change-me-in-production",
    ) -> None:
        self._registry = registry
        self._jwt_secret_key = jwt_secret_key

    # ── public read ───────────────────────────────────────────────────────────

    def get_value(self, key: str, env: "Env", default: Any = None) -> Any:
        """Resolve: org-scoped DB row → system-scoped DB row → XML default → default."""
        field_desc = self._registry.get_field(key)
        org_id = self._current_org_id(env)

        # 1. Org-scoped row (only when principal carries org context)
        if org_id:
            rows = env.models["ir.config"].search(
                [("key", "=", key), ("scope", "=", "organization"),
                 ("organization_id", "=", org_id)],
                limit=1,
            )
            if rows:
                return self._deserialize(rows[0], field_desc)

        # 2. System-scoped row
        rows = env.models["ir.config"].search(
            [("key", "=", key), ("scope", "=", "system")],
            limit=1,
        )
        if rows:
            return self._deserialize(rows[0], field_desc)

        # 3. XML default
        if field_desc and field_desc.default is not None:
            return self._cast(field_desc.default, field_desc.field_type)

        return default

    def list_modules_nav(self, env: "Env") -> list[dict]:
        return [
            {
                "module": m.module,
                "label": m.label,
                "icon": m.icon,
                "category": m.category,
                "order": m.order,
            }
            for m in self._registry.list_modules()
        ]

    def get_module_schema_with_values(self, module_key: str, env: "Env") -> dict:
        mod = self._registry.get_module(module_key)
        if mod is None:
            return {}
        return {
            "module": mod.module,
            "label": mod.label,
            "icon": mod.icon,
            "sections": [
                {
                    "string": section.string,
                    "fields": [self._serialize_field_with_value(f, env) for f in section.fields],
                }
                for section in mod.sections
            ],
        }

    # ── public write ──────────────────────────────────────────────────────────

    def set_value(
        self,
        key: str,
        value: Any,
        scope: str,
        org_id: Optional[str],
        env: "Env",
    ) -> None:
        field_desc = self._registry.get_field(key)

        if field_desc and field_desc.requires_migration:
            raise SettingsMigrationRequired(key)

        is_secret = field_desc and field_desc.secret
        serialized = self._serialize_for_storage(value, field_desc)
        old_raw = self._get_raw_for_key(key, scope, org_id, env)

        # Upsert ir.config row
        org_id_str = org_id or ""
        domain = [("key", "=", key), ("scope", "=", scope), ("organization_id", "=", org_id_str)]
        rows = env.models["ir.config"].search(domain, limit=1)

        if is_secret:
            from .encryption import encrypt_value
            encrypted = encrypt_value(str(value), self._jwt_secret_key)
            write_data = {"value": None, "encrypted_value": encrypted, "scope": scope, "organization_id": org_id_str}
            if rows:
                rows[0].write(write_data)
            else:
                env.models["ir.config"].create({"key": key, **write_data})
        else:
            write_data = {"value": serialized, "encrypted_value": None, "scope": scope, "organization_id": org_id_str}
            if rows:
                rows[0].write(write_data)
            else:
                env.models["ir.config"].create({"key": key, **write_data})

        # Audit log
        changed_by = self._principal_id(env)
        env.models["ir.config.log"].create({
            "key": key,
            "old_value": "••••••" if is_secret else (old_raw or ""),
            "new_value": "••••••" if is_secret else (serialized or ""),
            "scope": scope,
            "organization_id": org_id_str,
            "changed_by": changed_by,
        })

    def save_module_values(self, module_key: str, values: dict, env: "Env") -> None:
        for key, value in values.items():
            field_desc = self._registry.get_field(key)
            if field_desc is None:
                continue
            scope = field_desc.scope
            org_id = self._current_org_id(env) if scope == "organization" else None
            self.set_value(key, value, scope, org_id, env)

    # ── private helpers ───────────────────────────────────────────────────────

    def _current_org_id(self, env: "Env") -> Optional[str]:
        if env.principal:
            return env.principal.get("tenant_id") or env.principal.get("organization_id")
        return env.tenant_id or None

    def _principal_id(self, env: "Env") -> str:
        if env.principal:
            return str(env.principal.get("sub") or env.principal.get("user_id") or "system")
        return "system"

    def _get_raw_for_key(
        self, key: str, scope: str, org_id: Optional[str], env: "Env"
    ) -> Optional[str]:
        org_id_str = org_id or ""
        rows = env.models["ir.config"].search(
            [("key", "=", key), ("scope", "=", scope), ("organization_id", "=", org_id_str)],
            limit=1,
        )
        if not rows:
            return None
        return rows[0].value

    def _deserialize(self, record: Any, field_desc: Any) -> Any:
        if field_desc and field_desc.secret and record.encrypted_value:
            from .encryption import decrypt_value
            return decrypt_value(record.encrypted_value, self._jwt_secret_key)
        return self._cast(record.value, field_desc.field_type if field_desc else "char")

    def _serialize_for_storage(self, value: Any, field_desc: Any) -> Optional[str]:
        if value is None:
            return None
        if field_desc and field_desc.field_type == "boolean":
            return "true" if value else "false"
        return str(value)

    def _cast(self, raw: Optional[str], field_type: str) -> Any:
        if raw is None:
            return None
        if field_type == "boolean":
            return raw.lower() in ("true", "1", "yes")
        if field_type == "integer":
            return int(raw)
        return raw

    def _serialize_field_with_value(self, field_desc: Any, env: "Env") -> dict:
        value = self.get_value(field_desc.name, env)
        display_value = None
        if field_desc.field_type == "many2one" and value and field_desc.model:
            try:
                results = env.models[field_desc.model].search(
                    [("record_uuid", "=", str(value))], limit=1
                )
                display_value = getattr(results[0], "name", None) if results else None
            except Exception:
                pass

        serialized_value = "••••••" if (field_desc.secret and value is not None) else value

        return {
            "name": field_desc.name,
            "type": field_desc.field_type,
            "scope": field_desc.scope,
            "label": field_desc.label,
            "requires_migration": field_desc.requires_migration,
            "secret": field_desc.secret,
            "depends_on": field_desc.depends_on,
            "readonly": field_desc.readonly,
            "options": field_desc.options,
            "model": field_desc.model,
            "domain": field_desc.domain,
            "value": serialized_value,
            "display_value": display_value,
        }
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
pytest src/tests/settings/test_settings_service.py -v
```

Expected: All tests PASS. (Fix any mismatches in search domain syntax if needed — check by running one test at a time.)

- [ ] **Step 5: Commit**

```bash
git add src/ede/core/services/settings/service.py src/tests/settings/test_settings_service.py
git commit -m "$(cat <<'EOF'
[NEW] foundation: SettingsService — value resolution, encryption, audit logging

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Env + BootEnvironment Integration + ViewRegistry Fix

**Files:**
- Modify: `src/ede/core/types.py` (BootEnvironment)
- Modify: `src/ede/core/bootstrap.py`
- Modify: `src/ede/core/env.py`
- Modify: `src/ede/cli/commands/server.py`
- Modify: `src/ede/cli/commands/gateway.py`
- Modify: `src/ede/core/services/presentation/view_registry.py`

- [ ] **Step 1: Add settings_registry to BootEnvironment**

Open `src/ede/core/bootstrap.py`. Find the `BootEnvironment` class definition (around line 15):

```python
@dataclass(frozen=True)
class BootEnvironment:
    registry: Registry
    apps: List[AppMeta]
    http_routes: HttpRouteRegistry
```

Add the `settings_registry` field with a default of `None` so existing callers don't break:

```python
@dataclass(frozen=True)
class BootEnvironment:
    registry: Registry
    apps: List[AppMeta]
    http_routes: HttpRouteRegistry
    settings_registry: Optional[Any] = None  # SettingsRegistry, Optional to avoid circular import
```

Add `Optional` to the imports at the top of `bootstrap.py`:
```python
from typing import List, Optional, Any
```

- [ ] **Step 2: Build SettingsRegistry in bootstrap_environment()**

In `src/ede/core/bootstrap.py`, find the `bootstrap_environment()` function. Before the `return BootEnvironment(...)` line, add:

```python
    from ede.core.services.settings.registry import SettingsRegistry
    settings_registry = SettingsRegistry()
    for app in apps:
        try:
            settings_registry.load_from_app(app)
        except Exception:
            pass  # Non-fatal: missing or malformed settings files don't crash boot

    return BootEnvironment(
        registry=registry,
        apps=apps,
        http_routes=loader.http_routes,
        settings_registry=settings_registry,
    )
```

- [ ] **Step 3: Add settings_registry to Env**

Open `src/ede/core/env.py`. Add to the `__init__` signature (after `web_push_registry`):

```python
    def __init__(
        self,
        registry: Registry,
        *,
        # ... existing params ...
        web_push_registry: Optional[Any] = None,
        settings_registry: Optional[Any] = None,   # ADD THIS
    ) -> None:
```

In `__init__` body, after `self.web_push_registry = web_push_registry`:
```python
        self._settings_registry: Optional[Any] = settings_registry
```

Add these two methods before the `run_hooks` method:

```python
    @property
    def settings_registry(self) -> Optional[Any]:
        """SettingsRegistry instance built at boot. None in test envs without it."""
        return self._settings_registry

    def get_setting(self, key: str, default: Any = None) -> Any:
        """Resolve a runtime setting by key (org → system → XML default → default)."""
        if self._settings_registry is None:
            return default
        from ede.core.services.settings.service import SettingsService
        from ede import runtime
        jwt_key = getattr(getattr(runtime, "settings", None), "JWT_SECRET_KEY", "change-me-in-production")
        return SettingsService(self._settings_registry, jwt_secret_key=jwt_key).get_value(key, self, default)
```

- [ ] **Step 4: Propagate settings_registry in clone methods**

In `env.py`, each of the three clone methods (`with_tenant_id`, `with_principal`, `with_remote_addr`) creates a new `Env(...)`. Add `settings_registry=self._settings_registry` to each. For example:

```python
    def with_tenant_id(self, tenant_id: str) -> "Env":
        return Env(
            self.registry,
            event_queue=self._event_queue,
            persistence=self.persistence,
            correlation_id=self.correlation_id,
            causation_id=self.causation_id,
            tenant_id=tenant_id,
            tenant_host_suffixes=list(self.tenant_host_suffixes),
            tenant_id_header_name=self.tenant_id_header_name,
            principal=self.principal,
            remote_addr=self.remote_addr,
            web_push_registry=self.web_push_registry,
            settings_registry=self._settings_registry,   # ADD
        )
```

Apply the same `settings_registry=self._settings_registry` addition to `with_principal` and `with_remote_addr`.

- [ ] **Step 5: Pass settings_registry in server.py**

Open `src/ede/cli/commands/server.py`. Find the `base_env = Env(...)` block (around line 116). Add `settings_registry` to the call:

```python
    base_env = Env(
        registry,
        event_queue=event_queue,
        persistence=None,
        tenant_host_suffixes=list(foundation_settings_module.TENANT_HOST_SUFFIXES),
        tenant_id_header_name=foundation_settings_module.TENANT_ID_HEADER_NAME,
        tenant_id=default_tenant_id,
        settings_registry=environment.settings_registry,   # ADD
    )
```

The `environment` variable is the `BootEnvironment` returned by `bootstrap_environment()`. Find it by looking for the line `environment = bootstrap_environment(config=config)` or similar. The variable name might differ — grep for `BootEnvironment` in the function to find it.

- [ ] **Step 6: Pass settings_registry in gateway.py**

Open `src/ede/cli/commands/gateway.py`. Find `base_env = Env(...)` (around line 255). Apply the same addition:

```python
        settings_registry=environment.settings_registry,  # ADD
```

- [ ] **Step 7: Fix ViewRegistry to skip <settings> root files**

Open `src/ede/core/services/presentation/view_registry.py`. Find the `_collect_view_file_paths_from_app` method. Inside the loop where view files are collected, add a skip check for settings files:

```python
    def _collect_view_file_paths_from_app(self, *, app_spec: Any) -> List[str]:
        app_root_dir = os.path.abspath(str(app_spec.app_root_dir))
        views_dir = os.path.abspath(os.path.join(app_root_dir, "views")) + os.sep
        data_files: Iterable[str] = getattr(app_spec, "data_files", ()) or ()
        absolute_data_files = [os.path.abspath(str(p)) for p in data_files]

        view_files: List[str] = []
        for absolute_path in absolute_data_files:
            if not absolute_path.endswith(".xml"):
                continue
            if not absolute_path.startswith(views_dir):
                continue
            if self._is_settings_file(absolute_path):   # ADD skip check
                continue
            view_files.append(absolute_path)

        return sorted(view_files)

    @staticmethod
    def _is_settings_file(file_path: str) -> bool:      # ADD this method
        """Return True if the XML file has a <settings> root (not a view DSL file)."""
        import xml.etree.ElementTree as ET
        try:
            tree = ET.parse(file_path)
            return tree.getroot().tag == "settings"
        except Exception:
            return False
```

Make sure `Iterable` is imported in `view_registry.py` (it likely already is from `typing`).

- [ ] **Step 8: Smoke-test the server boots**

```bash
ede serve --config ede.conf &
sleep 3
curl -s http://localhost:8000/api/v1/health | python -m json.tool
kill %1
```

Expected: `{"status": "ok"}` with no errors in server output.

- [ ] **Step 9: Commit**

```bash
git add src/ede/core/bootstrap.py \
        src/ede/core/env.py \
        src/ede/cli/commands/server.py \
        src/ede/cli/commands/gateway.py \
        src/ede/core/services/presentation/view_registry.py
git commit -m "$(cat <<'EOF'
[IMP] core: integrate SettingsRegistry into boot + Env; ViewRegistry skips <settings> files

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Settings API Controller + Base Module Settings XML + Menu

**Files:**
- Create: `src/ede/foundation/base/api/settings_routes.py`
- Create: `src/ede/foundation/base/views/base_settings.xml`
- Modify: `src/ede/foundation/base/api/__init__.py`
- Modify: `src/ede/foundation/base/__manifest__.py`
- Modify: `src/ede/foundation/base/data/base_menus.xml`

- [ ] **Step 1: Create the settings API controller**

```python
# src/ede/foundation/base/api/settings_routes.py
from __future__ import annotations

import logging
from typing import Any, Dict

from ede.core import api
from ede.core.services.http.controller import RouteController

logger = logging.getLogger(__name__)


@api.route_config(prefix="/api/settings", tags=["foundation.settings"])
class SettingsController(RouteController):
    """
    Runtime settings API.

    GET  /api/settings           → list of module nav entries
    GET  /api/settings/{module}  → schema + current values for one module
    PATCH /api/settings/{module} → save values for one module
    """

    @api.route("", methods=["GET"], auth="user")
    def list_modules(self) -> list[dict]:
        """Return all registered settings modules for sidebar nav."""
        svc = self._make_service()
        if svc is None:
            return []
        return svc.list_modules_nav(self.env)

    @api.route("/{module_key}", methods=["GET"], auth="user")
    def get_module(self, module_key: str) -> Dict[str, Any]:
        """Return schema + resolved values for one settings module."""
        svc = self._make_service()
        if svc is None:
            return {"error": "settings_registry not available"}
        result = svc.get_module_schema_with_values(module_key, self.env)
        if not result:
            return {"error": f"Module '{module_key}' not found"}
        return result

    @api.route("/{module_key}", methods=["PATCH"], auth="user")
    def update_module(self, module_key: str, body: Dict[str, Any] = None) -> Dict[str, Any]:
        """
        Save settings values for one module.

        Body: { "values": { "key.name": <value>, ... } }

        Returns updated schema+values on success.
        Returns 409 { "error": "migration_required", "key": "..." } if a
        requires_migration field is included in values.
        """
        from ede.core.services.settings.service import SettingsMigrationRequired
        svc = self._make_service()
        if svc is None:
            return {"error": "settings_registry not available"}

        body = body or {}
        values: dict = body.get("values") or {}

        try:
            svc.save_module_values(module_key, values, self.env)
        except SettingsMigrationRequired as exc:
            # Return 409-style response; the FastAPI adapter will send 200
            # but the client checks for "error": "migration_required"
            return {"error": "migration_required", "key": exc.key}

        return svc.get_module_schema_with_values(module_key, self.env)

    # ── private ───────────────────────────────────────────────────────────────

    def _make_service(self):
        reg = getattr(self.env, "settings_registry", None)
        if reg is None:
            return None
        from ede.core.services.settings.service import SettingsService
        from ede import runtime
        jwt_key = getattr(getattr(runtime, "settings", None), "JWT_SECRET_KEY", "change-me-in-production")
        return SettingsService(reg, jwt_secret_key=jwt_key)
```

- [ ] **Step 2: Register controller in base API __init__.py**

Open `src/ede/foundation/base/api/__init__.py`. Add the import:

```python
from . import routes
from . import crud_routes
from . import bootstrap
from . import rbac_routes
from . import settings_routes   # ADD
```

- [ ] **Step 3: Create base module settings XML**

```xml
<!-- src/ede/foundation/base/views/base_settings.xml -->
<settings module="base" label="General" icon="settings" category="Company" order="10">

  <section string="Company Policies">
    <field name="base.signup_method"
           type="selection"
           scope="system"
           label="User Registration Method"
           default="invite_only">
      <option value="invite_only" label="Invite Only"/>
      <option value="open" label="Open Registration"/>
      <option value="disabled" label="Disabled"/>
    </field>
  </section>

  <section string="Features">
    <field name="base.feature.multi_org"
           type="boolean"
           scope="system"
           label="Enable Multi-Organization Support"
           default="false"/>
  </section>

</settings>
```

- [ ] **Step 4: Update base manifest to include settings XML**

Open `src/ede/foundation/base/__manifest__.py`. In the `"data"` list, add the settings XML after the existing view DSL files:

```python
        # ── View DSL ───────────────────────────────────────────────────────────
        "views/res_organization_views.xml",
        "views/res_user_views.xml",
        "views/res_country_views.xml",
        "views/ir_action_views.xml",
        "views/ir_menu_views.xml",
        "views/ir_data_reference_views.xml",
        "views/ir_rbac_views.xml",
        # ── Settings DSL ───────────────────────────────────────────────────────
        "views/base_settings.xml",      # ADD THIS LINE
        # ── Navigation structure ───────────────────────────────────────────────
        "data/base_menus.xml",
```

- [ ] **Step 5: Add General Settings ir.action + ir.menu to base_menus.xml**

Open `src/ede/foundation/base/data/base_menus.xml`. Find the section with `<!-- Actions -->` near the top (around line 33). Add a new action for General Settings after the existing actions but before the menu records:

```xml
    <!-- General Settings — client action opens SettingsPage React component -->
    <record id="base.action_general_settings" model="ir.action">
      <field name="name">General Settings</field>
      <field name="path">general-settings</field>
      <field name="action_type">client</field>
      <field name="client_component">settings.general</field>
    </record>
```

Then find `base.menu_cat_general_settings` (around line 137). Add a menu item linking to this action inside the General Settings category:

```xml
    <!-- General Settings menu leaf — points to the client settings action -->
    <record id="base.menu_general_settings" model="ir.menu">
      <field name="name">General Settings</field>
      <field name="parent_id" ref="base.menu_cat_general_settings"/>
      <field name="action_id" ref="base.action_general_settings"/>
      <field name="sequence">1</field>
    </record>
```

- [ ] **Step 6: Apply data changes via migration upgrade**

```bash
ede migrate upgrade --config ede.conf
```

- [ ] **Step 7: Smoke-test the API endpoints**

Start the server and test:

```bash
ede serve --config ede.conf &
sleep 3

# Login first to get a token
TOKEN=$(curl -s -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"admin"}' | python -c "import sys,json; print(json.load(sys.stdin).get('access_token',''))")

# Test list modules
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/settings | python -m json.tool

# Test get module
curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/settings/base | python -m json.tool

kill %1
```

Expected: Module list includes `{"module": "base", "label": "General", ...}`. Module detail shows sections with fields.

- [ ] **Step 8: Commit**

```bash
git add src/ede/foundation/base/api/settings_routes.py \
        src/ede/foundation/base/api/__init__.py \
        src/ede/foundation/base/views/base_settings.xml \
        src/ede/foundation/base/__manifest__.py \
        src/ede/foundation/base/data/base_menus.xml
git commit -m "$(cat <<'EOF'
[NEW] foundation.base: settings API controller + base_settings.xml + General Settings menu entry

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Frontend — SettingsPage + API Service + ClientActionRegistry

**Files:**
- Create: `src/frontend/src/workspace/settings/types.ts`
- Create: `src/frontend/src/workspace/services/SettingsApiService.ts`
- Create: `src/frontend/src/workspace/settings/SettingsPage.tsx`
- Modify: `src/frontend/src/workspace/components/action/ClientActionRegistry.tsx`

- [ ] **Step 1: Define TypeScript types**

```typescript
// src/frontend/src/workspace/settings/types.ts
export interface ModuleNavEntry {
    module: string
    label: string
    icon: string | null
    category: string
    order: number
}

export interface SettingsFieldDef {
    name: string
    type: "boolean" | "integer" | "char" | "selection" | "many2one"
    scope: "system" | "organization"
    label: string
    requires_migration: boolean
    secret: boolean
    depends_on: string | null
    readonly: boolean
    options: { value: string; label: string }[]
    model: string | null
    domain: string | null
    value: unknown
    display_value: string | null
}

export interface SettingsSectionDef {
    string: string
    fields: SettingsFieldDef[]
}

export interface ModuleSettings {
    module: string
    label: string
    icon: string | null
    sections: SettingsSectionDef[]
}
```

- [ ] **Step 2: Create the settings API service**

```typescript
// src/frontend/src/workspace/services/SettingsApiService.ts
import { httpClient } from "@/services/api/HttpClient"
import type { ModuleNavEntry, ModuleSettings } from "../settings/types"

export async function listSettingsModules(): Promise<ModuleNavEntry[]> {
    return httpClient.get<ModuleNavEntry[]>("/api/settings")
}

export async function getModuleSettings(moduleKey: string): Promise<ModuleSettings> {
    return httpClient.get<ModuleSettings>(`/api/settings/${moduleKey}`)
}

export async function saveModuleSettings(
    moduleKey: string,
    values: Record<string, unknown>,
): Promise<ModuleSettings> {
    return httpClient.patch<ModuleSettings>(`/api/settings/${moduleKey}`, { values })
}
```

Check how `httpClient` is defined in `src/frontend/src/services/api/HttpClient.ts` — it likely has a `.get()` and `.post()` method. If `.patch()` does not exist, add it following the same pattern as `.post()` but with `method: "PATCH"`.

- [ ] **Step 3: Create the SettingsPage skeleton**

```tsx
// src/frontend/src/workspace/settings/SettingsPage.tsx
import { useEffect, useState } from "react"
import type { ActionDef } from "../../types"
import type { ModuleNavEntry } from "./types"
import { listSettingsModules } from "../services/SettingsApiService"
import { SettingsNav } from "./SettingsNav"
import { SettingsPane } from "./SettingsPane"

interface Props {
    action: ActionDef
}

export function SettingsPage({ action }: Props) {
    const [modules, setModules] = useState<ModuleNavEntry[]>([])
    const [activeModule, setActiveModule] = useState<string | null>(null)
    const [isDirty, setIsDirty] = useState(false)

    useEffect(() => {
        listSettingsModules().then((mods) => {
            setModules(mods)
            if (mods.length > 0 && !activeModule) {
                setActiveModule(mods[0].module)
            }
        })
    }, [])

    const handleNavSelect = (moduleKey: string) => {
        if (isDirty) {
            const ok = window.confirm("You have unsaved changes. Discard them?")
            if (!ok) return
            setIsDirty(false)
        }
        setActiveModule(moduleKey)
    }

    return (
        <div className="flex h-full bg-surface overflow-hidden">
            <SettingsNav
                modules={modules}
                activeModule={activeModule}
                onSelect={handleNavSelect}
            />
            <div className="flex-1 overflow-y-auto">
                {activeModule ? (
                    <SettingsPane
                        key={activeModule}
                        moduleKey={activeModule}
                        onDirtyChange={setIsDirty}
                    />
                ) : (
                    <div className="flex items-center justify-center h-full text-sm text-text-subtle">
                        Select a settings module from the sidebar.
                    </div>
                )}
            </div>
        </div>
    )
}
```

- [ ] **Step 4: Register SettingsPage in ClientActionRegistry**

Open `src/frontend/src/workspace/components/action/ClientActionRegistry.tsx`. Add the import and registration:

```tsx
import type { ActionDef } from "../../types"
import { StorageDriveView } from "../../views/client/StorageDriveView"
import { SettingsPage } from "../../settings/SettingsPage"   // ADD

type ClientComponent = React.ComponentType<{ action: ActionDef }>

const REGISTRY: Record<string, ClientComponent> = {
    "storage.drive": StorageDriveView,
    "settings.general": SettingsPage,   // ADD
}
```

- [ ] **Step 5: Verify TypeScript compiles**

```bash
cd src/frontend && bun run typecheck 2>&1 | head -20
```

If `typecheck` script doesn't exist: `bunx tsc --noEmit 2>&1 | head -20`

Fix any import errors (usually missing httpClient.patch — add it if needed).

- [ ] **Step 6: Start dev server and verify Settings page loads**

```bash
cd src/frontend && bun run dev &
# In another terminal, start the backend:
cd /home/dharmang/personal/ede-frame/repository/ede-framework && ede serve --config ede.conf &
sleep 5
```

Navigate to the app, log in, find "General Settings" in the Settings menu. The page should render with the sidebar and a loading state. Check browser console for errors.

- [ ] **Step 7: Commit**

```bash
git add src/frontend/src/workspace/settings/types.ts \
        src/frontend/src/workspace/services/SettingsApiService.ts \
        src/frontend/src/workspace/settings/SettingsPage.tsx \
        src/frontend/src/workspace/components/action/ClientActionRegistry.tsx \
        src/frontend/src/services/api/HttpClient.ts  # if modified for .patch()
git commit -m "$(cat <<'EOF'
[NEW] frontend: SettingsPage client action + API service + ClientActionRegistry registration

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Frontend — SettingsNav + SettingsPane + Field Widgets + SaveBar

**Files:**
- Create: `src/frontend/src/workspace/settings/SettingsNav.tsx`
- Create: `src/frontend/src/workspace/settings/SettingsPane.tsx`
- Create: `src/frontend/src/workspace/settings/SettingsSection.tsx`
- Create: `src/frontend/src/workspace/settings/SettingsField.tsx`
- Create: `src/frontend/src/workspace/settings/fields/SettingsToggle.tsx`
- Create: `src/frontend/src/workspace/settings/fields/SettingsTextInput.tsx`
- Create: `src/frontend/src/workspace/settings/fields/SettingsNumberInput.tsx`
- Create: `src/frontend/src/workspace/settings/fields/SettingsSelect.tsx`
- Create: `src/frontend/src/workspace/settings/fields/SettingsMany2One.tsx`

- [ ] **Step 1: Create SettingsNav (sidebar)**

```tsx
// src/frontend/src/workspace/settings/SettingsNav.tsx
import type { ModuleNavEntry } from "./types"

interface Props {
    modules: ModuleNavEntry[]
    activeModule: string | null
    onSelect: (moduleKey: string) => void
}

export function SettingsNav({ modules, activeModule, onSelect }: Props) {
    // Group modules by category, preserving order within each group
    const grouped = modules.reduce<Record<string, ModuleNavEntry[]>>((acc, mod) => {
        const cat = mod.category || "General"
        if (!acc[cat]) acc[cat] = []
        acc[cat].push(mod)
        return acc
    }, {})

    return (
        <aside className="w-56 shrink-0 border-r border-border bg-surface-subtle h-full overflow-y-auto py-4">
            {Object.entries(grouped).map(([category, mods]) => (
                <div key={category} className="mb-4">
                    <p className="px-4 pb-1 text-xs font-semibold uppercase tracking-wider text-text-subtlest">
                        {category}
                    </p>
                    {mods.map((mod) => (
                        <button
                            key={mod.module}
                            onClick={() => onSelect(mod.module)}
                            className={[
                                "w-full text-left px-4 py-2 text-sm transition-colors",
                                activeModule === mod.module
                                    ? "bg-primary/10 text-primary font-medium"
                                    : "text-text hover:bg-surface-muted",
                            ].join(" ")}
                        >
                            {mod.label}
                        </button>
                    ))}
                </div>
            ))}
        </aside>
    )
}
```

- [ ] **Step 2: Create field widgets**

```tsx
// src/frontend/src/workspace/settings/fields/SettingsToggle.tsx
interface Props {
    value: boolean
    onChange: (v: boolean) => void
    disabled?: boolean
}
export function SettingsToggle({ value, onChange, disabled }: Props) {
    return (
        <button
            role="switch"
            aria-checked={value}
            onClick={() => !disabled && onChange(!value)}
            className={[
                "relative inline-flex h-5 w-9 items-center rounded-full transition-colors focus:outline-none",
                value ? "bg-primary" : "bg-border",
                disabled ? "opacity-50 cursor-not-allowed" : "cursor-pointer",
            ].join(" ")}
        >
            <span
                className={[
                    "inline-block h-4 w-4 transform rounded-full bg-white shadow transition-transform",
                    value ? "translate-x-4" : "translate-x-0.5",
                ].join(" ")}
            />
        </button>
    )
}
```

```tsx
// src/frontend/src/workspace/settings/fields/SettingsTextInput.tsx
interface Props {
    value: string
    onChange: (v: string) => void
    secret?: boolean
    disabled?: boolean
    placeholder?: string
}
export function SettingsTextInput({ value, onChange, secret, disabled, placeholder }: Props) {
    return (
        <input
            type={secret ? "password" : "text"}
            value={typeof value === "string" ? value : ""}
            onChange={(e) => onChange(e.target.value)}
            disabled={disabled}
            placeholder={placeholder}
            className="w-64 rounded-md border border-border bg-surface px-3 py-1.5 text-sm text-text focus:border-primary focus:outline-none disabled:opacity-50"
        />
    )
}
```

```tsx
// src/frontend/src/workspace/settings/fields/SettingsNumberInput.tsx
interface Props {
    value: number | ""
    onChange: (v: number) => void
    disabled?: boolean
}
export function SettingsNumberInput({ value, onChange, disabled }: Props) {
    return (
        <input
            type="number"
            value={value === "" ? "" : value}
            onChange={(e) => {
                const n = parseInt(e.target.value, 10)
                if (!isNaN(n)) onChange(n)
            }}
            disabled={disabled}
            className="w-32 rounded-md border border-border bg-surface px-3 py-1.5 text-sm text-text focus:border-primary focus:outline-none disabled:opacity-50"
        />
    )
}
```

```tsx
// src/frontend/src/workspace/settings/fields/SettingsSelect.tsx
interface Option { value: string; label: string }
interface Props {
    value: string
    options: Option[]
    onChange: (v: string) => void
    disabled?: boolean
}
export function SettingsSelect({ value, options, onChange, disabled }: Props) {
    return (
        <select
            value={value ?? ""}
            onChange={(e) => onChange(e.target.value)}
            disabled={disabled}
            className="w-64 rounded-md border border-border bg-surface px-3 py-1.5 text-sm text-text focus:border-primary focus:outline-none disabled:opacity-50"
        >
            {options.map((opt) => (
                <option key={opt.value} value={opt.value}>{opt.label}</option>
            ))}
        </select>
    )
}
```

```tsx
// src/frontend/src/workspace/settings/fields/SettingsMany2One.tsx
import { useState, useEffect } from "react"
import { httpClient } from "@/services/api/HttpClient"

interface Props {
    value: string | null
    displayValue: string | null
    model: string
    domain?: string | null
    onChange: (uuid: string, label: string) => void
    disabled?: boolean
}

interface SearchResult { record_uuid: string; name: string }

export function SettingsMany2One({ value, displayValue, model, domain, onChange, disabled }: Props) {
    const [options, setOptions] = useState<SearchResult[]>([])

    useEffect(() => {
        const modelUrl = model.replace(/\./g, "__")
        const domainParam = domain ? `?domain=${encodeURIComponent(domain)}` : ""
        httpClient
            .get<{ records: SearchResult[] }>(`/api/v1/${modelUrl}/search${domainParam}`)
            .then((res) => setOptions(res.records ?? []))
            .catch(() => {/* non-fatal */})
    }, [model, domain])

    return (
        <select
            value={value ?? ""}
            onChange={(e) => {
                const opt = options.find((o) => o.record_uuid === e.target.value)
                if (opt) onChange(opt.record_uuid, opt.name)
            }}
            disabled={disabled}
            className="w-64 rounded-md border border-border bg-surface px-3 py-1.5 text-sm text-text focus:border-primary focus:outline-none disabled:opacity-50"
        >
            <option value="">— Select —</option>
            {options.map((opt) => (
                <option key={opt.record_uuid} value={opt.record_uuid}>{opt.name}</option>
            ))}
        </select>
    )
}
```

- [ ] **Step 3: Create SettingsField (label + scope badge + widget dispatcher)**

```tsx
// src/frontend/src/workspace/settings/SettingsField.tsx
import type { SettingsFieldDef } from "./types"
import { SettingsToggle } from "./fields/SettingsToggle"
import { SettingsTextInput } from "./fields/SettingsTextInput"
import { SettingsNumberInput } from "./fields/SettingsNumberInput"
import { SettingsSelect } from "./fields/SettingsSelect"
import { SettingsMany2One } from "./fields/SettingsMany2One"
import { MigrationStub } from "./MigrationStub"

interface Props {
    field: SettingsFieldDef
    formValue: unknown
    allValues: Record<string, unknown>
    onChange: (key: string, value: unknown) => void
}

export function SettingsField({ field, formValue, allValues, onChange }: Props) {
    // depends_on: hide when referenced boolean is false
    if (field.depends_on) {
        const parentValue = allValues[field.depends_on]
        if (!parentValue) return null
    }

    const disabled = field.readonly

    const widget = (() => {
        if (field.requires_migration) {
            return <MigrationStub field={field} />
        }
        switch (field.type) {
            case "boolean":
                return (
                    <SettingsToggle
                        value={Boolean(formValue)}
                        onChange={(v) => onChange(field.name, v)}
                        disabled={disabled}
                    />
                )
            case "integer":
                return (
                    <SettingsNumberInput
                        value={typeof formValue === "number" ? formValue : ""}
                        onChange={(v) => onChange(field.name, v)}
                        disabled={disabled}
                    />
                )
            case "selection":
                return (
                    <SettingsSelect
                        value={typeof formValue === "string" ? formValue : ""}
                        options={field.options}
                        onChange={(v) => onChange(field.name, v)}
                        disabled={disabled}
                    />
                )
            case "many2one":
                return (
                    <SettingsMany2One
                        value={typeof formValue === "string" ? formValue : null}
                        displayValue={field.display_value}
                        model={field.model ?? ""}
                        domain={field.domain}
                        onChange={(uuid) => onChange(field.name, uuid)}
                        disabled={disabled}
                    />
                )
            default: // char
                return (
                    <SettingsTextInput
                        value={typeof formValue === "string" ? formValue : (formValue === "••••••" ? "" : "")}
                        onChange={(v) => onChange(field.name, v)}
                        secret={field.secret}
                        disabled={disabled}
                        placeholder={field.secret ? "••••••" : undefined}
                    />
                )
        }
    })()

    return (
        <div className="flex items-center justify-between py-3 border-b border-border/40 last:border-0">
            <div className="flex items-center gap-2 min-w-0">
                <span className="text-sm text-text truncate">{field.label}</span>
                <span
                    className={[
                        "shrink-0 text-[10px] font-medium px-1.5 py-0.5 rounded",
                        field.scope === "system"
                            ? "bg-amber-100 text-amber-700 dark:bg-amber-900/30 dark:text-amber-400"
                            : "bg-blue-100 text-blue-700 dark:bg-blue-900/30 dark:text-blue-400",
                    ].join(" ")}
                >
                    {field.scope === "system" ? "System" : "Company"}
                </span>
            </div>
            <div className="shrink-0 ml-4">{widget}</div>
        </div>
    )
}
```

- [ ] **Step 4: Create SettingsSection**

```tsx
// src/frontend/src/workspace/settings/SettingsSection.tsx
import type { SettingsSectionDef } from "./types"
import { SettingsField } from "./SettingsField"

interface Props {
    section: SettingsSectionDef
    formValues: Record<string, unknown>
    onChange: (key: string, value: unknown) => void
}

export function SettingsSection({ section, formValues, onChange }: Props) {
    return (
        <div className="mb-6">
            {section.string && (
                <h3 className="text-xs font-semibold uppercase tracking-wider text-text-subtle mb-2">
                    {section.string}
                </h3>
            )}
            <div className="rounded-lg border border-border bg-surface px-4">
                {section.fields.map((field) => (
                    <SettingsField
                        key={field.name}
                        field={field}
                        formValue={formValues[field.name]}
                        allValues={formValues}
                        onChange={onChange}
                    />
                ))}
            </div>
        </div>
    )
}
```

- [ ] **Step 5: Create SettingsPane (module pane with form state and SaveBar)**

```tsx
// src/frontend/src/workspace/settings/SettingsPane.tsx
import { useEffect, useState } from "react"
import type { ModuleSettings } from "./types"
import { getModuleSettings, saveModuleSettings } from "../services/SettingsApiService"
import { SettingsSection } from "./SettingsSection"

interface Props {
    moduleKey: string
    onDirtyChange: (dirty: boolean) => void
}

export function SettingsPane({ moduleKey, onDirtyChange }: Props) {
    const [schema, setSchema] = useState<ModuleSettings | null>(null)
    const [loadedValues, setLoadedValues] = useState<Record<string, unknown>>({})
    const [formValues, setFormValues] = useState<Record<string, unknown>>({})
    const [saving, setSaving] = useState(false)
    const [error, setError] = useState<string | null>(null)

    // Compute initial values map from schema
    const extractValues = (mod: ModuleSettings): Record<string, unknown> =>
        Object.fromEntries(
            mod.sections.flatMap((s) => s.fields.map((f) => [f.name, f.value]))
        )

    useEffect(() => {
        setSchema(null)
        setError(null)
        getModuleSettings(moduleKey)
            .then((mod) => {
                setSchema(mod)
                const vals = extractValues(mod)
                setLoadedValues(vals)
                setFormValues(vals)
            })
            .catch(() => setError("Failed to load settings."))
    }, [moduleKey])

    const isDirty = JSON.stringify(formValues) !== JSON.stringify(loadedValues)

    useEffect(() => {
        onDirtyChange(isDirty)
    }, [isDirty])

    const handleChange = (key: string, value: unknown) => {
        setFormValues((prev) => ({ ...prev, [key]: value }))
    }

    const handleDiscard = () => {
        setFormValues({ ...loadedValues })
        onDirtyChange(false)
    }

    const handleSave = async () => {
        if (!schema) return
        setSaving(true)
        setError(null)
        try {
            const updated = await saveModuleSettings(moduleKey, formValues)
            const vals = extractValues(updated)
            setLoadedValues(vals)
            setFormValues(vals)
            setSchema(updated)
            onDirtyChange(false)
        } catch (e) {
            setError("Failed to save settings. Please try again.")
        } finally {
            setSaving(false)
        }
    }

    if (error && !schema) {
        return <div className="p-6 text-sm text-error">{error}</div>
    }
    if (!schema) {
        return <div className="p-6 text-sm text-text-subtle">Loading…</div>
    }

    return (
        <div className="max-w-2xl mx-auto p-6">
            <h2 className="text-lg font-semibold text-text mb-6">{schema.label}</h2>

            {schema.sections.map((section) => (
                <SettingsSection
                    key={section.string}
                    section={section}
                    formValues={formValues}
                    onChange={handleChange}
                />
            ))}

            {error && <p className="text-sm text-error mb-3">{error}</p>}

            {isDirty && (
                <div className="sticky bottom-0 flex items-center justify-end gap-3 border-t border-border bg-surface py-3 mt-6">
                    <button
                        onClick={handleDiscard}
                        disabled={saving}
                        className="px-4 py-2 text-sm text-text-subtle hover:text-text transition-colors disabled:opacity-50"
                    >
                        Discard
                    </button>
                    <button
                        onClick={handleSave}
                        disabled={saving}
                        className="px-4 py-2 text-sm font-medium rounded-md bg-primary text-white hover:bg-primary/90 transition-colors disabled:opacity-50"
                    >
                        {saving ? "Saving…" : "Save Changes"}
                    </button>
                </div>
            )}
        </div>
    )
}
```

- [ ] **Step 6: Build and verify TypeScript**

```bash
cd src/frontend && bunx tsc --noEmit 2>&1 | head -30
```

Fix any type errors. Common issues:
- `httpClient.patch` not defined — check `HttpClient.ts` and add if missing
- Import path issues — use relative paths from `settings/` to `services/`

- [ ] **Step 7: Visual test in browser**

With dev server and backend running:
1. Navigate to Settings > General Settings
2. Verify sidebar shows "Company" category with "General" item
3. Click General — pane loads with "Company Policies" section
4. Toggle a boolean field — SaveBar should appear at bottom
5. Click Discard — SaveBar disappears, field resets
6. Change a value and Save — verify no error and values persist on refresh

- [ ] **Step 8: Commit**

```bash
git add src/frontend/src/workspace/settings/
git commit -m "$(cat <<'EOF'
[NEW] frontend: Settings pane components — Nav, Pane, Section, Field, widgets, SaveBar

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Frontend — MigrationStub

**Files:**
- Create: `src/frontend/src/workspace/settings/MigrationStub.tsx`

- [ ] **Step 1: Create MigrationStub component**

```tsx
// src/frontend/src/workspace/settings/MigrationStub.tsx
import { useState } from "react"
import type { SettingsFieldDef } from "./types"

interface Props {
    field: SettingsFieldDef
}

/**
 * v1 stub for requires_migration=true fields.
 * Shows the current value read-only with a "Change Provider" button
 * that opens an inline confirmation. In v1, saving updates the value
 * directly without data migration. Real migration wizard is deferred to v2.
 */
export function MigrationStub({ field }: Props) {
    const [showConfirm, setShowConfirm] = useState(false)

    const currentLabel = field.display_value ?? (field.value ? String(field.value) : "Not configured")

    return (
        <div className="flex items-center gap-3">
            <span className="text-sm text-text-subtle">{currentLabel}</span>
            <button
                onClick={() => setShowConfirm(true)}
                className="text-xs font-medium text-primary hover:underline"
            >
                Change Provider →
            </button>
            {showConfirm && (
                <div className="fixed inset-0 bg-black/40 flex items-center justify-center z-50">
                    <div className="bg-surface rounded-xl shadow-xl p-6 max-w-sm w-full mx-4">
                        <h3 className="text-base font-semibold text-text mb-2">Change Provider</h3>
                        <p className="text-sm text-text-subtle mb-4">
                            Changing this service provider may require migrating existing data.
                            Full migration wizard is coming in a future release.
                            For now, this will update the setting immediately.
                        </p>
                        <div className="flex justify-end gap-3">
                            <button
                                onClick={() => setShowConfirm(false)}
                                className="px-3 py-1.5 text-sm text-text-subtle hover:text-text"
                            >
                                Cancel
                            </button>
                            <button
                                onClick={() => setShowConfirm(false)}
                                className="px-3 py-1.5 text-sm font-medium rounded-md bg-primary text-white hover:bg-primary/90"
                            >
                                Understood
                            </button>
                        </div>
                    </div>
                </div>
            )}
        </div>
    )
}
```

- [ ] **Step 2: Verify TypeScript**

```bash
cd src/frontend && bunx tsc --noEmit 2>&1 | grep -i error | head -10
```

Expected: No new errors.

- [ ] **Step 3: Build production bundle**

```bash
cd src/frontend && bun run build 2>&1 | tail -10
```

Expected: Build succeeds with no errors.

- [ ] **Step 4: Run test suite**

```bash
cd /home/dharmang/personal/ede-frame/repository/ede-framework && ./run_tests.sh
```

Expected: All existing tests still pass. New settings tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/frontend/src/workspace/settings/MigrationStub.tsx
git commit -m "$(cat <<'EOF'
[NEW] frontend: MigrationStub v1 placeholder for migration-required settings fields

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
EOF
)"
```

---

## Verification Checklist

Run these after all tasks complete:

```bash
# 1. Full test suite
./run_tests.sh

# 2. Settings-specific tests
pytest src/tests/settings/ -v

# 3. Production frontend build
cd src/frontend && bun run build

# 4. End-to-end smoke test
ede serve --config ede.conf &
sleep 3
TOKEN=$(curl -s -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"admin"}' | python -c "import sys,json; d=json.load(sys.stdin); print(d.get('access_token',''))")

# Module list
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/settings | python -m json.tool

# Base module schema
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8000/api/settings/base | python -m json.tool

# Save a setting
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:8000/api/settings/base \
  -d '{"values":{"base.signup_method":"open"}}' | python -m json.tool

# Verify audit log was written
curl -s -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/v1/ir.config.log/search?limit=1" | python -m json.tool

kill %1
```

Expected at each step: valid JSON response with no errors.

---

## Self-Review Notes

- `SettingsService.get_value()` uses `env.principal.get("tenant_id")` as org_id — valid for v1 single-org-per-tenant setup
- `ir.config.organization_id` is `Char(36)` not a FK Reference — deliberate: avoids coupling, service validates
- `ViewRegistry._is_settings_file()` adds one XML parse per views/ file at boot — acceptable cost (~1ms per file)
- `httpClient.patch()` may not exist in the frontend — check `HttpClient.ts` and add if missing (Task 7)
- Migration for `ir.config` and `ir.config.log` must be run before testing the API; Task 3 covers this
