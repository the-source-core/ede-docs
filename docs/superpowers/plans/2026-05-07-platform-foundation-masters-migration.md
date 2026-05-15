# Platform Foundation Masters Migration

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move domain-agnostic masters (timezone, state, city, UOM, partner) from `logistics.masters` into `foundation.base` as `res.*` models; update all FK references across `logistics.masters` and `logistics.pricing`; regenerate the Alembic migration.

**Architecture:** New model files (`geography.py`, `uom.py`, `partner.py`) added to `src/ede/foundation/base/models/`. Existing `res.country` enriched with iso_code_3, numeric_code, timezone FK. Seed CSV/XML files relocated from `src/domains/logistics/masters/data/` to `src/ede/foundation/base/data/` with model keys and `ref=` prefixes updated from `masters.*` to `base.*`. Logistics-specific models (`unlocode`, `facilities`, `equipment`, `products`, `trade.lane`) keep only logistics-specific FKs, repointed to `res.*`. A single new Alembic migration covers all table additions, enrichment columns, and drops.

**Tech Stack:** Python 3.10+, EDE framework (`api.model`, `fields.*`), Alembic, SQLAlchemy, XML data loader DSL.

---

## File Map

**New files (foundation.base):**
- `src/ede/foundation/base/models/geography.py` — `res.timezone`, `res.state`, `res.city`
- `src/ede/foundation/base/models/uom.py` — `res.uom.category`, `res.uom`
- `src/ede/foundation/base/models/partner.py` — `res.partner.role.master`, `res.partner`, `res.partner.role`, `res.partner.address`
- `src/ede/foundation/base/data/res.timezone.csv` — timezone seed (copied from logistics)
- `src/ede/foundation/base/data/res.country.csv` — **replace** with enriched version from logistics country CSV
- `src/ede/foundation/base/data/res.state.xml` — states seed (updated model keys + refs)
- `src/ede/foundation/base/data/res.country.timezones.xml` — country→timezone patches (updated)
- `src/ede/foundation/base/data/res.uom.category.csv` — UOM category seed (copied)
- `src/ede/foundation/base/data/res.uom.xml` — UOM units seed (updated model keys + refs)

**Modified files (foundation.base):**
- `src/ede/foundation/base/models/country.py` — enrich with iso_code_3, numeric_code, default_timezone_id, is_standard_locked, reverse O2M
- `src/ede/foundation/base/models/__init__.py` — add imports for geography, uom, partner
- `src/ede/foundation/base/__manifest__.py` — add new data files and views

**Modified files (logistics.masters models):**
- `src/domains/logistics/masters/models/geography.py` — remove Timezone/Country/State/City classes; keep UnlocodeMaster with FK refs updated to `res.*`
- `src/domains/logistics/masters/models/uom.py` — remove all models (file becomes empty/comment only)
- `src/domains/logistics/masters/models/parties.py` — remove all models (file becomes empty/comment only)
- `src/domains/logistics/masters/models/facilities.py` — update country/state/city/timezone/contact FKs → `res.*`
- `src/domains/logistics/masters/models/equipment.py` — update UOM FKs → `res.uom`
- `src/domains/logistics/masters/models/products.py` — update UOM + country FKs in `TradeLane` and `ProductMaster`
- `src/domains/logistics/masters/models/__init__.py` — remove `uom` and `parties` imports
- `src/domains/logistics/masters/__manifest__.py` — remove relocated seed files

**Modified files (logistics.pricing):**
- `src/domains/logistics/pricing/models/rate.py` — update `origin_country_id`, `destination_country_id` → `res.country`; `customer_id`, `vendor_id` → `res.partner`; `uom_id` on `RateLine` → `res.uom`
- `src/domains/logistics/pricing/models/margin_rule.py` — update `customer_id` → `res.partner`

---

## Task 1: Add `res.timezone`, `res.state`, `res.city` to foundation.base

**Files:**
- Create: `src/ede/foundation/base/models/geography.py`

- [ ] **Step 1: Create the file**

```python
# src/ede/foundation/base/models/geography.py
from __future__ import annotations

from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model("res.timezone", description="Timezone")
class Timezone(DomainModel):
    """IANA timezone reference. Used by contacts, facilities, calendar, and reporting."""

    code = fields.Char(max_length=50, unique=True, index=True, label="Code")
    name = fields.Char(max_length=200, required=True, label="Name")
    tz_name = fields.Char(max_length=100, label="TZ Name", help="IANA timezone name, e.g. Asia/Kolkata")
    utc_offset = fields.Char(max_length=10, label="UTC Offset", help="e.g. +05:30")
    is_standard_locked = fields.Boolean(default=False, label="Standard Locked")
    active = fields.Boolean(default=True, label="Active")


@api.model("res.state", description="State / Province")
class State(DomainModel):
    """State or province under a country. Used in addresses across all ERP domains."""

    country_id = fields.Reference(
        "res.country",
        on_delete="restrict",
        required=True,
        label="Country",
    )
    code = fields.Char(max_length=50, unique=True, index=True, label="Code")
    name = fields.Char(max_length=200, required=True, label="Name")
    standard_code = fields.Char(max_length=50, label="Standard Code")
    is_standard_locked = fields.Boolean(default=False, label="Standard Locked")
    active = fields.Boolean(default=True, label="Active")

    cities = fields.One2Many("res.city", "state_id", label="Cities")


@api.model("res.city", description="City")
class City(DomainModel):
    """City under a country and optional state. Used in CRM, HR, Finance addresses."""

    country_id = fields.Reference(
        "res.country",
        on_delete="restrict",
        required=True,
        label="Country",
    )
    state_id = fields.Reference(
        "res.state",
        on_delete="set null",
        label="State / Province",
    )
    code = fields.Char(max_length=50, unique=True, index=True, label="Code")
    name = fields.Char(max_length=200, required=True, label="Name")
    standard_code = fields.Char(max_length=50, label="Standard Code")
    is_standard_locked = fields.Boolean(default=False, label="Standard Locked")
    active = fields.Boolean(default=True, label="Active")
```

---

## Task 2: Enrich `res.country`

**Files:**
- Modify: `src/ede/foundation/base/models/country.py`

- [ ] **Step 1: Add enrichment fields to the Country model**

Replace the entire file content with:

```python
from __future__ import annotations

from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model("res.country", description="Country")
class Country(DomainModel):
    """
    ISO country reference record.

    iso_code_2    ISO 3166-1 alpha-2 (e.g. IN) — stored as `code` for backward compat
    iso_code_3    ISO 3166-1 alpha-3 (e.g. IND)
    numeric_code  ISO 3166-1 numeric (e.g. 356)
    """

    name = fields.Char(max_length=100, required=True, unique=True, label="Country Name")
    code = fields.Char(
        max_length=2,
        required=True,
        unique=True,
        index=True,
        label="ISO Alpha-2 Code",
        help="Two-letter ISO 3166-1 code, e.g. IN, US, GB.",
    )
    iso_code_3 = fields.Char(max_length=3, label="ISO Alpha-3", help="Three-letter ISO 3166-1 code, e.g. IND, USA.")
    numeric_code = fields.Char(max_length=10, label="Numeric Code", help="ISO 3166-1 numeric code, e.g. 356.")
    phone_code = fields.Integer(label="Calling Code", help="International dialling prefix without '+', e.g. 91.")
    currency_code = fields.Char(max_length=3, label="Currency Code", help="ISO 4217 code, e.g. INR, USD.")
    address_format = fields.Char(
        multi_line=True,
        label="Address Layout",
        help="Python %%-format string for rendering addresses.",
        default="%(street)s\n%(city)s %(state_code)s %(zip)s\n%(country_name)s",
    )
    default_timezone_id = fields.Reference(
        "res.timezone",
        on_delete="set null",
        label="Default Timezone",
    )
    is_standard_locked = fields.Boolean(default=False, label="Standard Locked")
    active = fields.Boolean(default=True, label="Active")

    states = fields.One2Many("res.state", "country_id", label="States")
    cities = fields.One2Many("res.city", "country_id", label="Cities")
```

---

## Task 3: Add `res.uom.category` and `res.uom` to foundation.base

**Files:**
- Create: `src/ede/foundation/base/models/uom.py`

- [ ] **Step 1: Create the file**

```python
# src/ede/foundation/base/models/uom.py
from __future__ import annotations

from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model("res.uom.category", description="UOM Category")
class UomCategory(DomainModel):
    """Measurement category grouping related units (weight, volume, dimension, quantity)."""

    code = fields.Char(max_length=50, unique=True, index=True, label="Code")
    name = fields.Char(max_length=200, required=True, label="Name")
    standard_code = fields.Char(max_length=50, label="Standard Code")
    is_standard_locked = fields.Boolean(default=False, label="Standard Locked")
    active = fields.Boolean(default=True, label="Active")

    uoms = fields.One2Many("res.uom", "category_id", label="Units of Measure")


@api.model("res.uom", description="Unit of Measure")
class Uom(DomainModel):
    """Unit of measure with conversion factor to base unit. Used by logistics, inventory, procurement, HR."""

    category_id = fields.Reference(
        "res.uom.category",
        on_delete="restrict",
        required=True,
        label="Category",
    )
    code = fields.Char(max_length=50, unique=True, index=True, label="Code")
    name = fields.Char(max_length=200, required=True, label="Name")
    symbol = fields.Char(max_length=20, label="Symbol", help="Display symbol, e.g. kg, m³")
    standard_code = fields.Char(max_length=50, label="Standard Code")
    conversion_factor_to_base = fields.Decimal(label="Conversion Factor to Base")
    is_base_unit = fields.Boolean(default=False, label="Is Base Unit")
    is_standard_locked = fields.Boolean(default=False, label="Standard Locked")
    active = fields.Boolean(default=True, label="Active")
```

---

## Task 4: Add `res.partner` family to foundation.base

**Files:**
- Create: `src/ede/foundation/base/models/partner.py`

- [ ] **Step 1: Create the file**

```python
# src/ede/foundation/base/models/partner.py
from __future__ import annotations

from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model("res.partner.role.master", description="Partner Role Master")
class PartnerRoleMaster(DomainModel):
    """Reusable role definitions assigned to partners (customer, vendor, carrier, agent, etc.)."""

    code = fields.Char(max_length=50, unique=True, index=True, label="Code")
    name = fields.Char(max_length=200, required=True, label="Role Name")
    standard_code = fields.Char(max_length=50, label="Standard Code")
    is_standard_locked = fields.Boolean(default=False, label="Standard Locked")
    active = fields.Boolean(default=True, label="Active")
    description = fields.Char(multi_line=True, label="Description")


@api.model("res.partner", description="Partner")
class Partner(DomainModel):
    """
    Universal party master for all ERP domains.

    A single real-world entity can hold many roles (customer, vendor, carrier, agent)
    via res.partner.role. Used by CRM, Finance, Procurement, HR, and Logistics.
    """

    member_type = fields.Enum(
        selection=[
            ("company",       "Company"),
            ("individual",    "Individual"),
            ("branch_entity", "Branch Entity"),
        ],
        label="Member Type",
    )
    code = fields.Char(max_length=50, unique=True, index=True, label="Code")
    name = fields.Char(max_length=200, required=True, label="Name")
    legal_name = fields.Char(max_length=300, label="Legal Name")
    standard_code = fields.Char(max_length=50, label="Standard Code")
    is_standard_locked = fields.Boolean(default=False, label="Standard Locked")

    parent_partner_id = fields.Reference(
        "res.partner",
        on_delete="set null",
        label="Parent Partner",
    )

    country_id  = fields.Reference("res.country",  on_delete="set null", label="Country")
    state_id    = fields.Reference("res.state",    on_delete="set null", label="State / Province")
    city_id     = fields.Reference("res.city",     on_delete="set null", label="City")
    timezone_id = fields.Reference("res.timezone", on_delete="set null", label="Timezone")

    email   = fields.Char(max_length=254, label="Email")
    phone   = fields.Char(max_length=30,  label="Phone")
    mobile  = fields.Char(max_length=30,  label="Mobile")
    website = fields.Char(max_length=256, label="Website")

    address_line_1  = fields.Char(max_length=300, label="Address Line 1")
    address_line_2  = fields.Char(max_length=300, label="Address Line 2")
    postal_code     = fields.Char(max_length=20,  label="Postal Code")

    tax_registration_no = fields.Char(max_length=100, label="Tax Registration No")
    registration_no     = fields.Char(max_length=100, label="Registration No")

    active = fields.Boolean(default=True, label="Active")
    description = fields.Char(multi_line=True, label="Description")

    role_ids       = fields.One2Many("res.partner.role",    "partner_id",        label="Roles")
    child_partners = fields.One2Many("res.partner",         "parent_partner_id", label="Child Partners")
    address_ids    = fields.One2Many("res.partner.address", "partner_id",        label="Addresses")


@api.model("res.partner.role", description="Partner Role")
class PartnerRole(DomainModel):
    """Role assignment: one partner may hold many roles."""

    partner_id = fields.Reference(
        "res.partner",
        on_delete="restrict",
        required=True,
        index=True,
        label="Partner",
    )
    role_id = fields.Reference(
        "res.partner.role.master",
        on_delete="restrict",
        required=True,
        label="Role",
    )
    active = fields.Boolean(default=True, label="Active")
    description = fields.Char(multi_line=True, label="Notes")


@api.model("res.partner.address", description="Partner Address")
class PartnerAddress(DomainModel):
    """
    A reusable address record linked to a partner.

    One partner can have multiple addresses of different types.
    unlocode_id is set only when the address represents a logistics node (port, ICD);
    that field is defined in logistics.masters which adds it via a One2Many reverse.
    """

    partner_id = fields.Reference(
        "res.partner",
        on_delete="restrict",
        required=True,
        index=True,
        label="Partner",
    )
    address_type = fields.Enum(
        selection=[
            ("billing",   "Billing"),
            ("pickup",    "Pickup"),
            ("delivery",  "Delivery"),
            ("office",    "Office"),
            ("warehouse", "Warehouse"),
            ("mailing",   "Mailing"),
        ],
        required=True,
        label="Address Type",
    )
    is_default = fields.Boolean(default=False, label="Default")

    address_line_1 = fields.Char(max_length=300, label="Address Line 1")
    address_line_2 = fields.Char(max_length=300, label="Address Line 2")
    postal_code    = fields.Char(max_length=20,  label="Postal Code")

    country_id = fields.Reference("res.country", on_delete="restrict", required=True, label="Country")
    state_id   = fields.Reference("res.state",   on_delete="set null",               label="State")
    city_id    = fields.Reference("res.city",    on_delete="set null",               label="City")

    latitude  = fields.Decimal(label="Latitude")
    longitude = fields.Decimal(label="Longitude")

    is_free_text_location = fields.Boolean(default=False, label="Custom Location")
    active = fields.Boolean(default=True, label="Active")
    notes  = fields.Char(multi_line=True, label="Notes")
```

---

## Task 5: Wire foundation.base `__init__.py` and `__manifest__.py`

**Files:**
- Modify: `src/ede/foundation/base/models/__init__.py`
- Modify: `src/ede/foundation/base/__manifest__.py`

- [ ] **Step 1: Update `__init__.py`**

Add three new imports in dependency order (timezone before state/city before partner; uom before partner address):

```python
from . import ping
from . import ping_listener
from . import geography    # res.timezone → res.state → res.city (res.country ref'd but defined next)
from . import country      # res.country — enriched; references res.timezone (defined above)
from . import currency
from . import exchange_rate
from . import uom          # res.uom.category → res.uom
from . import partner      # res.partner.role.master → res.partner → res.partner.role → res.partner.address
from . import organization
from . import user
from . import data_reference
from . import action
from . import menu
from . import org_unit
from . import role
from . import permission
from . import role_binding
from . import access_grant
from . import notification_setting
from . import audit
from . import ir_config
from . import ir_config_log
from . import sequence
```

> **Note on import order:** `geography.py` defines `res.timezone` which `res.country` references. Import `geography` before `country`. `partner.py` references `res.country`, `res.state`, `res.city`, `res.timezone` — import after all four.

- [ ] **Step 2: Update `__manifest__.py` — add seed data entries**

Add to the `"data"` list (in load order — geography seed before country timezone patches):

```python
{
    "name": "Foundation Base",
    ...
    "data": [
        # ── Seed data ──────────────────────────────────────────────────────────
        "data/res.timezone.csv",          # NEW — load timezones before countries reference them
        "data/res.country.csv",           # EXISTING — now enriched with iso_code_3, is_standard_locked
        "data/res.country.timezones.xml", # NEW — patch country default_timezone_id after both loaded
        "data/res.uom.category.csv",      # NEW
        "data/res.uom.xml",               # NEW
        "data/base_setup.xml",
        # ── RBAC ───────────────────────────────────────────────────────────────
        "data/rbac_roles.xml",
        "data/ir.rbac.permission.csv",
        "data/rbac_seed.xml",
        # ── View DSL ───────────────────────────────────────────────────────────
        "views/res_organization_views.xml",
        "views/res_user_views.xml",
        "views/res_country_views.xml",
        "views/res_currency_views.xml",
        "views/res_exchange_rate_views.xml",
        "views/ir_sequence_views.xml",
        "views/ir_action_views.xml",
        "views/ir_menu_views.xml",
        "views/ir_data_reference_views.xml",
        "views/ir_rbac_views.xml",
        "views/base_settings.xml",
        # ── Navigation ─────────────────────────────────────────────────────────
        "data/base_menus.xml",
    ],
}
```

> **States seed note:** `res.state.xml` (13 000+ records) is intentionally omitted from foundation.base manifest for now — states are seeded at logistics domain startup since only logistics currently needs them. Add to foundation.base manifest when another domain needs state data.

---

## Task 6: Migrate seed data to foundation.base

**Files:**
- Create: `src/ede/foundation/base/data/res.timezone.csv`
- Replace: `src/ede/foundation/base/data/res.country.csv`
- Create: `src/ede/foundation/base/data/res.country.timezones.xml`
- Create: `src/ede/foundation/base/data/res.uom.category.csv`
- Create: `src/ede/foundation/base/data/res.uom.xml`

- [ ] **Step 1: Copy timezone CSV**

```bash
cp src/domains/logistics/masters/data/logistics.timezone.master.csv \
   src/ede/foundation/base/data/res.timezone.csv
```

The CSV format is already correct — no edits needed. The `id` values (`tz_africa_abidjan`, etc.) become `base.tz_africa_abidjan` when loaded by foundation.base.

- [ ] **Step 2: Replace `res.country.csv` with enriched logistics version**

Run:
```bash
cp src/domains/logistics/masters/data/logistics.country.master.csv \
   /tmp/logistics_country.csv
```

Then write `src/ede/foundation/base/data/res.country.csv` with the header from the logistics CSV but adding `currency_code` and `address_format` columns (both empty — existing data loader will skip blank values for optional fields):

```
id,code,name,iso_code_2,iso_code_3,is_standard_locked,active,phone_code,currency_code,address_format
```

The simplest approach: the logistics CSV already has `id, code, name, iso_code_2, iso_code_3, is_standard_locked, active, phone_code`. Add two trailing empty columns for `currency_code` and `address_format` using:

```bash
python3 - <<'EOF'
import csv, sys

src = "src/domains/logistics/masters/data/logistics.country.master.csv"
dst = "src/ede/foundation/base/data/res.country.csv"

with open(src) as fin, open(dst, "w", newline="") as fout:
    reader = csv.DictReader(fin)
    fieldnames = list(reader.fieldnames) + ["currency_code", "address_format"]
    writer = csv.DictWriter(fout, fieldnames=fieldnames)
    writer.writeheader()
    for row in reader:
        row["currency_code"] = ""
        row["address_format"] = ""
        writer.writerow(row)

print("Done — written", dst)
EOF
```

- [ ] **Step 3: Create `res.country.timezones.xml`**

```bash
cp src/domains/logistics/masters/data/geography_country_timezones.xml \
   src/ede/foundation/base/data/res.country.timezones.xml
```

Then do a global replace in the file:
```bash
sed -i \
  -e 's/model="logistics\.country\.master"/model="res.country"/g' \
  -e 's/ref="masters\.country_/ref="base.country_/g' \
  -e 's/ref="masters\.tz_/ref="base.tz_/g' \
  src/ede/foundation/base/data/res.country.timezones.xml
```

- [ ] **Step 4: Copy UOM category CSV**

```bash
cp src/domains/logistics/masters/data/logistics.uom.category.master.csv \
   src/ede/foundation/base/data/res.uom.category.csv
```

No format changes needed — the `id` values like `uom_cat_weight` become `base.uom_cat_weight`.

- [ ] **Step 5: Create `res.uom.xml`**

```bash
cp src/domains/logistics/masters/data/uom_units.xml \
   src/ede/foundation/base/data/res.uom.xml
```

Then update model keys and ref prefixes:
```bash
sed -i \
  -e 's/model="logistics\.uom\.master"/model="res.uom"/g' \
  -e 's/ref="masters\.uom_cat_/ref="base.uom_cat_/g' \
  -e 's/ id="logistics\.uom_/ id="base.uom_/g' \
  src/ede/foundation/base/data/res.uom.xml
```

---

## Task 7: Strip `logistics.masters/models/geography.py`

Remove Timezone, Country, State, City classes. Keep only `UnlocodeMaster`, updating its FK refs.

**Files:**
- Modify: `src/domains/logistics/masters/models/geography.py`

- [ ] **Step 1: Replace the file**

```python
# src/domains/logistics/masters/models/geography.py
from __future__ import annotations

from ede.core import api
from ede.core.kernel import fields
from ede.core.kernel.model import DomainModel


@api.model("logistics.unlocode.master", description="UN/LOCODE Master")
class UnlocodeMaster(DomainModel):
    """
    UN/LOCODE standardized global trade location.

    country_id and city_id reference res.country and res.city (platform models).
    UN/LOCODE itself (ports, airports, ICDs, terminals) is logistics-specific.
    """

    country_id = fields.Reference(
        "res.country",
        on_delete="restrict",
        required=True,
        label="Country",
    )
    city_id = fields.Reference(
        "res.city",
        on_delete="set null",
        label="City",
    )
    code = fields.Char(max_length=50, unique=True, index=True, label="Code")
    name = fields.Char(max_length=200, required=True, label="Name")
    unlocode = fields.Char(max_length=10, label="UN/LOCODE")
    function_code = fields.Char(max_length=20, label="Function Code")
    latitude = fields.Decimal(label="Latitude")
    longitude = fields.Decimal(label="Longitude")
    is_standard_locked = fields.Boolean(default=False, label="Standard Locked")
    active = fields.Boolean(default=True, label="Active")
    description = fields.Char(multi_line=True, label="Description")
```

---

## Task 8: Remove `logistics.masters/models/uom.py`

**Files:**
- Modify: `src/domains/logistics/masters/models/uom.py`

- [ ] **Step 1: Replace the file with a tombstone comment**

```python
# src/domains/logistics/masters/models/uom.py
# UOM models moved to foundation.base as res.uom.category and res.uom.
# See src/ede/foundation/base/models/uom.py
```

---

## Task 9: Remove `logistics.masters/models/parties.py`

**Files:**
- Modify: `src/domains/logistics/masters/models/parties.py`

- [ ] **Step 1: Replace the file with a tombstone comment**

```python
# src/domains/logistics/masters/models/parties.py
# Party/contact models moved to foundation.base as res.partner and related.
# See src/ede/foundation/base/models/partner.py
```

---

## Task 10: Update `logistics.masters/models/facilities.py`

All geography FKs → `res.*`. All contact FKs → `res.partner`.

**Files:**
- Modify: `src/domains/logistics/masters/models/facilities.py`

- [ ] **Step 1: Update FK references in `FacilityMaster`**

Change only the four geography FKs and three contact FKs (lines 83–107 in the original). The rest of the file stays identical.

Replace:
```python
    # Geography
    country_id = fields.Reference("logistics.country.master", on_delete="set null", label="Country")
    state_id = fields.Reference("logistics.state.master", on_delete="set null", label="State / Province")
    city_id = fields.Reference("logistics.city.master", on_delete="set null", label="City")
    unlocode_id = fields.Reference("logistics.unlocode.master", on_delete="set null", label="UN/LOCODE")
    timezone_id = fields.Reference("logistics.timezone.master", on_delete="set null", label="Timezone")

    # Parties
    owner_member_id = fields.Reference(
        "logistics.contact.member",
        on_delete="set null",
        label="Owner",
        help="Owning or controlling party.",
    )
    operator_member_id = fields.Reference(
        "logistics.contact.member",
        on_delete="set null",
        label="Operator",
        help="Operational party managing this facility.",
    )
    contact_member_id = fields.Reference(
        "logistics.contact.member",
        on_delete="set null",
        label="Primary Contact",
        help="Primary facility contact person or entity.",
    )
```

With:
```python
    # Geography
    country_id  = fields.Reference("res.country",             on_delete="set null", label="Country")
    state_id    = fields.Reference("res.state",               on_delete="set null", label="State / Province")
    city_id     = fields.Reference("res.city",                on_delete="set null", label="City")
    unlocode_id = fields.Reference("logistics.unlocode.master", on_delete="set null", label="UN/LOCODE")
    timezone_id = fields.Reference("res.timezone",            on_delete="set null", label="Timezone")

    # Parties
    owner_member_id = fields.Reference(
        "res.partner",
        on_delete="set null",
        label="Owner",
        help="Owning or controlling party.",
    )
    operator_member_id = fields.Reference(
        "res.partner",
        on_delete="set null",
        label="Operator",
        help="Operational party managing this facility.",
    )
    contact_member_id = fields.Reference(
        "res.partner",
        on_delete="set null",
        label="Primary Contact",
        help="Primary facility contact person or entity.",
    )
```

---

## Task 11: Update `logistics.masters/models/equipment.py`

UOM FK refs → `res.uom`.

**Files:**
- Modify: `src/domains/logistics/masters/models/equipment.py`

- [ ] **Step 1: Replace all four UOM FK refs in `EquipmentTypeMaster`**

Replace:
```python
    dimension_uom_id = fields.Reference("logistics.uom.master", on_delete="set null", label="Dimension UOM")
    weight_uom_id = fields.Reference("logistics.uom.master", on_delete="set null", label="Weight UOM")
    volume_uom_id = fields.Reference("logistics.uom.master", on_delete="set null", label="Volume UOM")
    area_uom_id = fields.Reference("logistics.uom.master", on_delete="set null", label="Area UOM")
```

With:
```python
    dimension_uom_id = fields.Reference("res.uom", on_delete="set null", label="Dimension UOM")
    weight_uom_id    = fields.Reference("res.uom", on_delete="set null", label="Weight UOM")
    volume_uom_id    = fields.Reference("res.uom", on_delete="set null", label="Volume UOM")
    area_uom_id      = fields.Reference("res.uom", on_delete="set null", label="Area UOM")
```

---

## Task 12: Update `logistics.masters/models/products.py`

`TradeLane` country FKs → `res.country`. `ProductMaster` UOM FK → `res.uom`.

**Files:**
- Modify: `src/domains/logistics/masters/models/products.py`

- [ ] **Step 1: Update `TradeLane` country FKs**

Replace:
```python
    origin_country_id = fields.Reference(
        "logistics.country.master",
        on_delete="restrict",
        required=True,
        label="Origin Country",
    )
    ...
    destination_country_id = fields.Reference(
        "logistics.country.master",
        on_delete="restrict",
        required=True,
        label="Destination Country",
    )
```

With:
```python
    origin_country_id = fields.Reference(
        "res.country",
        on_delete="restrict",
        required=True,
        label="Origin Country",
    )
    ...
    destination_country_id = fields.Reference(
        "res.country",
        on_delete="restrict",
        required=True,
        label="Destination Country",
    )
```

- [ ] **Step 2: Update `ProductMaster` UOM FK**

Replace:
```python
    default_uom_id = fields.Reference(
        "logistics.uom.master",
        on_delete="set null",
        label="Default UOM",
        help="Default unit of measure for pricing this charge on a quote line. "
             "Applicable when product_type is charge or accessorial.",
    )
```

With:
```python
    default_uom_id = fields.Reference(
        "res.uom",
        on_delete="set null",
        label="Default UOM",
        help="Default unit of measure for pricing this charge on a quote line. "
             "Applicable when product_type is charge or accessorial.",
    )
```

---

## Task 13: Update `logistics.masters/__init__.py` and `__manifest__.py`

**Files:**
- Modify: `src/domains/logistics/masters/models/__init__.py`
- Modify: `src/domains/logistics/masters/__manifest__.py`

- [ ] **Step 1: Update `models/__init__.py`** — remove `uom` and `parties` imports

```python
from . import geography    # logistics.unlocode.master only (timezone/country/state/city moved to foundation.base)
from . import equipment    # transport_mode → category → subcategory → identifier_type → equipment_type
from . import facilities   # facility_type → zone_type → chl_type → facility → zone
from . import operational  # equipment_status → condition → usage_status → movement_event_type
from . import security     # seal_type → seal_change_reason
from . import maintenance  # maintenance_type → status → inspection_type → inspection_result_status
from . import documents    # document_type → reference_context_type
from . import products     # product_master (freight_service/charge/accessorial) → commodity_master
```

- [ ] **Step 2: Update `__manifest__.py`** — remove relocated seed data, keep logistics-specific data

```python
{
    "name": "Global Logistics Masters",
    "summary": "Master data foundation for global logistics and equipment control.",
    "description": """
Global Logistics Masters provides governed reference data for logistics
and equipment control operations. Geography, UOM, and party masters are
platform-level and live in foundation.base (res.*). This module covers
UN/LOCODE trade locations, transport modes, equipment, facilities,
operational states, products, and commodities.
""",
    "author": "THE_BLACK_BOX",
    "category": "Logistics",
    "version": "1.0.0",
    "depends": [],
    "data": [
        # ── View DSL ──────────────────────────────────────────────────────────────
        "views/geography_views.xml",
        "views/facilities_views.xml",
        "views/equipment_views.xml",
        "views/operational_views.xml",
        "views/security_views.xml",
        "views/maintenance_views.xml",
        "views/documents_views.xml",
        "views/products_views.xml",
        # ── RBAC roles ────────────────────────────────────────────────────────────
        "data/logistics_roles.xml",
        # ── Seed data ─────────────────────────────────────────────────────────────
        "data/ir.rbac.permission.csv",
        # ── Navigation ────────────────────────────────────────────────────────────
        "data/logistics_menus.xml",
    ],
}
```

> **Removed from manifest:** `views/parties_views.xml`, `views/uom_views.xml`, `data/logistics.timezone.master.csv`, `data/logistics.country.master.csv`, `data/geography_country_timezones.xml`, `data/geography_states.xml`, `data/logistics.uom.category.master.csv`, `data/uom_units.xml`, `data/members_menus.xml`

---

## Task 14: Update `logistics.pricing/models/rate.py`

`pricing.rate`: country and party FKs → `res.*`. `pricing.rate.line`: UOM FK → `res.uom`.

**Files:**
- Modify: `src/domains/logistics/pricing/models/rate.py`

- [ ] **Step 1: Update `Rate.origin_country_id` and `destination_country_id`**

Replace:
```python
    origin_country_id = fields.Reference(
        "logistics.country.master",
        on_delete="set null",
        label="Origin Country",
        help="For country-level rate definitions when specific port is not required.",
    )
    destination_country_id = fields.Reference(
        "logistics.country.master",
        on_delete="set null",
        label="Destination Country",
    )
```

With:
```python
    origin_country_id = fields.Reference(
        "res.country",
        on_delete="set null",
        label="Origin Country",
        help="For country-level rate definitions when specific port is not required.",
    )
    destination_country_id = fields.Reference(
        "res.country",
        on_delete="set null",
        label="Destination Country",
    )
```

- [ ] **Step 2: Update `Rate.customer_id` and `vendor_id`**

Replace:
```python
    customer_id = fields.Reference(
        "logistics.contact.member",
        on_delete="set null",
        label="Customer",
        help="Set for customer-specific sell rates. Null = general tariff (all customers).",
    )
    vendor_id = fields.Reference(
        "logistics.contact.member",
        on_delete="set null",
        label="Vendor / Carrier",
        help="Set for buy rates — the carrier, airline, transporter, or agent.",
    )
```

With:
```python
    customer_id = fields.Reference(
        "res.partner",
        on_delete="set null",
        label="Customer",
        help="Set for customer-specific sell rates. Null = general tariff (all customers).",
    )
    vendor_id = fields.Reference(
        "res.partner",
        on_delete="set null",
        label="Vendor / Carrier",
        help="Set for buy rates — the carrier, airline, transporter, or agent.",
    )
```

- [ ] **Step 3: Update `RateLine.uom_id`**

Replace:
```python
    uom_id = fields.Reference(
        "logistics.uom.master",
        on_delete="set null",
        label="UOM",
        help="Per shipment, per container, per kg, per CBM, W/M, per trip, per day.",
    )
```

With:
```python
    uom_id = fields.Reference(
        "res.uom",
        on_delete="set null",
        label="UOM",
        help="Per shipment, per container, per kg, per CBM, W/M, per trip, per day.",
    )
```

- [ ] **Step 4: Update the docstring reference in `Rate`**

Replace (in the class docstring):
```
      - logistics.contact.member for all party types (customer, vendor, carrier, agent)
```

With:
```
      - res.partner for all party types (customer, vendor, carrier, agent)
```

---

## Task 15: Update `logistics.pricing/models/margin_rule.py`

`MarginRule.customer_id` → `res.partner`.

**Files:**
- Modify: `src/domains/logistics/pricing/models/margin_rule.py`

- [ ] **Step 1: Update `customer_id` FK**

Replace:
```python
    customer_id    = fields.Reference("logistics.contact.member", on_delete="set null", label="Customer")
```

With:
```python
    customer_id    = fields.Reference("res.partner", on_delete="set null", label="Customer")
```

---

## Task 16: Run tests

- [ ] **Step 1: Run the full test suite**

```bash
./run_tests.sh
```

Expected: all existing tests pass. The migration adds new models and removes old ones — no existing test touches the removed models. If any test fails, check the error message; likely cause is a missing import or a ref to a removed model key.

---

## Task 17: Generate Alembic migration and verify

- [ ] **Step 1: Generate the migration**

```bash
ede migrate generate -m "platform-geography-uom-partner-in-foundation-base" --config ede.conf
```

Expected output: a new file created in `src/ede/foundation/base/migrations/versions/` (or shared versions folder depending on alembic config) with `down_revision = '078d47db5c26'`.

- [ ] **Step 2: Verify migration content**

Open the generated file and confirm it contains:

**CREATE TABLE operations (new res.* tables):**
- `res_timezone`
- `res_state` (FK → `res_country.record_uuid`)
- `res_city` (FK → `res_country.record_uuid`, `res_state.record_uuid`)
- `res_uom_category`
- `res_uom` (FK → `res_uom_category.record_uuid`)
- `res_partner_role_master`
- `res_partner` (self-ref FK → `res_partner.record_uuid` + FKs to res.country/state/city/timezone)
- `res_partner_role` (FKs → `res_partner`, `res_partner_role_master`)
- `res_partner_address` (FKs → `res_partner`, `res_country`, `res_state`, `res_city`)

**ALTER TABLE operations (enriched res_country columns):**
- `ADD COLUMN iso_code_3`
- `ADD COLUMN numeric_code`
- `ADD COLUMN default_timezone_id` (FK → `res_timezone.record_uuid`)
- `ADD COLUMN is_standard_locked`
- `ADD COLUMN active`

**DROP TABLE operations (removed logistics.masters tables):**
- `logistics_timezone_master`
- `logistics_country_master`
- `logistics_state_master`
- `logistics_city_master`
- `logistics_uom_category_master`
- `logistics_uom_master`
- `logistics_contact_role_master`
- `logistics_contact_member`
- `logistics_contact_member_role`
- `logistics_contact_address`

**ALTER TABLE operations (FK column retargets in remaining logistics tables):**
- `logistics_facility_master`: drop old FK constraints on country/state/city/timezone/owner/operator/contact; add new FKs pointing to `res_country`, `res_state`, `res_city`, `res_timezone`, `res_partner`
- `logistics_equipment_type_master`: drop old UOM FK constraints; add new FKs to `res_uom`
- `logistics_product_master`: drop old UOM FK; add new FK to `res_uom`
- `logistics_trade_lane`: drop old country FK constraints; add new FKs to `res_country`
- `logistics_unlocode_master`: drop old country/city FK constraints; add new FKs to `res_country`, `res_city`
- `pricing_rate`: drop old country/partner FK constraints; add new FKs to `res_country`, `res_partner`
- `pricing_rate_line`: drop old UOM FK; add new FK to `res_uom`
- `pricing_margin_rule`: drop old partner FK; add new FK to `res_partner`

- [ ] **Step 3: Apply the migration (dev environment)**

```bash
ede migrate upgrade --config ede.conf
```

Expected: `INFO alembic: Running upgrade ... -> <new_rev>, platform-geography-uom-partner-in-foundation-base`

- [ ] **Step 4: Run the server to verify boot**

```bash
ede serve --config ede.conf --with-worker
```

Expected: server starts without import errors or missing model errors.
