# Technical Specification (TOR)
# Web Application: Security Systems Maintenance Workload Calculator
**Version:** 2.2  
**Based on:** Шаблон_нагрузки_з_v_4_00.xlsx  
**Date:** 2026-02-23  
**Changelog v2.0:** Replaced hardcoded equipment tables with dynamic device catalog architecture (§4.2, §4.3, §4.5, §5, §6.3, §6.4, §7, §9, §10, §13).  
**Changelog v2.1:** Incorporated three architectural decisions: (1) Option C two-layer quantity model (physical + maintained); (2) System type restriction enforced at UI/API/DB levels; (3) Normatives managed per (device, system) pair. Updated §4.2, §4.3, §6.4, §7.3, §7.4, added C-17–C-20, added AC-11–AC-13.  
**Changelog v2.2:** Added Engineers Module — engineers as first-class entities, object-engineer assignments with equal workload split, engineer capacity tracking, overload detection, engineer dashboard, and coverage gap reporting. Updated §2, §3, §4 (FR-10, FR-11), §5, §6 (§6.12–6.14), §7, §9, §10, §12, §13 (C-21–C-25), §14 (AC-14–AC-18).

---

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [Business Context & Goals](#2-business-context--goals)
3. [Glossary](#3-glossary)
4. [Functional Requirements](#4-functional-requirements)
5. [Data Model](#5-data-model)
6. [Calculation Engine](#6-calculation-engine)
7. [User Interface Requirements](#7-user-interface-requirements)
8. [Non-Functional Requirements](#8-non-functional-requirements)
9. [Architecture Constraints](#9-architecture-constraints)
10. [API Design](#10-api-design)
11. [Migrations & Data Import](#11-migrations--data-import)
12. [Roles & Permissions](#12-roles--permissions)
13. [Structural Clarifications & Corrections](#13-structural-clarifications--corrections)
14. [Acceptance Criteria](#14-acceptance-criteria)

---

## 1. Project Overview

The system is a web-based replacement for the Excel workbook `Шаблон_нагрузки_з_v_4_00.xlsx`. It automates the calculation of **required maintenance staffing headcount** (нагрузка / численность) for technical maintenance (ТО) engineers responsible for servicing security and fire protection systems at bank branch facilities (objects).

The source workbook currently covers **~2,935 objects** belonging to regional divisions (подразделения) of Belarusbank.

---

## 2. Business Context & Goals

### 2.1 Problem Statement
The Excel template is large (2,935 rows × up to 47 columns per sheet), manual to update, error-prone in formula propagation, and difficult to share or version. The device catalog is hardcoded as fixed columns — adding a new device type requires schema changes and formula updates across multiple sheets.

### 2.2 Goals
- Replace the multi-sheet Excel model with a centralized, browser-accessible application.
- Allow engineers and managers to input/update equipment quantities per facility.
- Automatically recalculate workload metrics on save.
- Support a **dynamic device catalog** — admins can add new device types and their normatives without code changes.
- Support import of existing XLSX data and export of results to XLSX/PDF.
- Provide a СВОД (summary) dashboard per division and per responsible engineer.
- Track individual engineer workload, capacity utilisation, and overload status.
- Surface coverage gaps — objects with no engineer assigned.
- Make normative time standards editable by administrators without formula changes or redeployment.

---

## 3. Glossary

| Term | Definition |
|---|---|
| Объект (Object/Facility) | A physical location (bank branch, archive, garage, infokiosk, etc.) |
| Подразделение | Regional division (e.g., Брестское областное управление №100) |
| Филиал | Branch (sub-unit of a division; may equal the division) |
| Ответственные ТО | Responsible maintenance engineer(s) assigned to an object |
| ОС | Охранная сигнализация — security alarm system |
| ПС | Пожарная сигнализация — fire alarm system |
| Видео | Video surveillance system |
| Записи | Video archive records and administration tasks |
| Ремонт | Repairs — replacement of equipment components |
| Дорога | Travel — distance and time to reach a facility |
| Тип системы (System Type) | One of: ОС, ПС, Видео — the maintenance schedule context |
| Тип устройства (Device Type) | A named device in the admin-managed catalog |
| Контекст устройства | A (device type + system type) pair that carries R1/R2 normatives |
| Р1 | Minutes per unit for routine inspection — system-context-specific |
| Р2 | Minutes per unit for full maintenance — system-context-specific |
| Физическое количество | Physical quantity: how many units exist on site |
| Обслуживаемое количество | Maintained quantity: how many units are counted in a given system's calculation |
| СВОД | Consolidated summary — aggregated totals per object or division |
| ИТОГО Числ | Final headcount coefficient: total monthly minutes converted to FTE |
| ПЗВ | Подготовительно-заключительное время — fixed prep/wrap-up time (20 min per visit) |
| ТМЦ | Товарно-материальные ценности — inventory/material assets |
| Инженер (Engineer) | A maintenance technician who is also a system user (login account) |
| Нагрузка на инженера | Workload per engineer — sum of workload shares from all assigned objects |
| Доля объекта | An object's itogo_chislo_with_travel divided equally among its assigned engineers |
| Мощность (Capacity) | Maximum FTE capacity of an engineer, set by admin (e.g. 1.0 full-time, 0.5 half-time) |
| Коэффициент загрузки | Load ratio = engineer_total_load / capacity — measure of utilisation |
| Перегрузка (Overload) | Load ratio ≥ 1.0 — engineer is assigned more work than their capacity |
| Покрытие (Coverage gap) | An object that has no engineers assigned |

---

## 4. Functional Requirements

### 4.1 Object Management (FR-01)
- CRUD operations for objects (facilities).
- Each object must have: `division`, `branch`, `name/address`.
- Responsible engineers are managed via the `object_engineers` join table (§4.10), not as a free-text field on the object.
- Objects are organized in a two-level hierarchy: **Division → Branch → Objects**.
- Support bulk import from XLSX (see §11).

### 4.2 Device Catalog Management (FR-02)

The device catalog is the central, admin-managed registry of all equipment types. It is fully dynamic — adding a new device type requires no code changes or schema migrations.

#### 4.2.1 Device Types
A device type is a named piece of hardware. It carries **no normatives directly** — normatives are defined per (device type × system type) context.

Each device type record contains:
- `name` — display name (e.g., "Galaxy 512 (Galaxy Dimension GD-520)")
- `description` — optional free-text

Admins create device types and then define one or more **device-system contexts** for each.

#### 4.2.2 Device-System Contexts (Normatives)
A context record defines that a device is valid within a specific system type and specifies the R1/R2 normative values for that combination:

| Field | Description |
|---|---|
| `device_type` | Reference to the device type |
| `system_type` | ОС, ПС, or Видео |
| `r1_minutes` | Minutes per unit for routine inspection (Р1) |
| `r2_minutes` | Minutes per unit for full maintenance (Р2) |

**Rules:**
- The (device_type, system_type) pair must be unique — one normative row per device per system.
- A device can have contexts for one, two, or all three system types, independently configured.
- A device with no context for a given system type **cannot be assigned** to that system at any object. This restriction is enforced at three levels: (1) the UI hides invalid system types in the assignment dropdown, (2) the API rejects the request with HTTP 422 code `NO_CONTEXT_FOR_SYSTEM`, (3) the `object_system_assignments.context_id` FK prevents it at the database level.
- R1 and R2 are set per context independently — the same device may have different normatives under different systems.
- Admin explicitly controls which systems a device is valid for by creating or omitting context rows. There is no implicit "allow all systems" default — a newly created device type has no valid systems until the admin adds at least one context.

**Seed data from source XLSX — ПС contexts:**

| Device | R1 (min) | R2 (min) |
|---|---|---|
| серий А6, Аларм | 5 | 12 |
| А16-512 | 6 | 13 |
| СПИ УОО Молния | 2 | 3 |
| АМ200-Notifier | 5 | 10 |
| Шлейфы сигнализации | 0.06 | 0.7 |
| Каналы считывания | 0.02 | 1.5 |
| Извещатели, оповещатели | 0.3 | 4 |
| Таблички | 0.3 | 2 |
| Galaxy 512 (контроллер АСПС и СО) | 15 | 20 |
| Оракул | 3 | 15 |
| Танго ПУ/БП | 1 | 17 |
| Танго ПУ/ЗК | 1 | 19 |
| Расширитель | 0.03 | 3 |
| Адресный модуль | 0.03 | 1.8 |
| Адресный шлейфно-релейный модуль | 0.03 | 3 |
| Усилитель линии УЛТ | 0.03 | 1 |
| Адресные извещатели | 0.2 | 2.3 |
| Колонки | 0.4 | 3 |

**Seed data — ОС contexts:**

| Device | R1 (min) | R2 (min) |
|---|---|---|
| Galaxy 512 (Galaxy Dimension GD-520) | 7 | 20 |
| Maestro (ППК ОП Maestro-1600) | 5 | 10 |
| серий А6, Аларм | 5 | 8 |
| А16-512 | 5 | 10 |
| Выносная панель управления ВПУ–А-16 | 4.5 | 4.5 |
| Устройство доступа | 1 | 4 |
| Шлейфы сигнализации | 0.06 | 0.7 |
| Каналы считывания | 0.02 | 1.5 |
| Извещатели, оповещатели | 0.7 | 3 |
| Galaxy 512 (контроллер АСПС и СО) | 15 | 20 |
| Расширитель | 0.03 | 3 |
| Контроллер системы | 3 | 9.5 |
| Системный блок ПЦН | 10 | 40 |
| ББП-20, ББП-3/12 (БРП 2401) | 0.03 | 3 |

**Seed data — Видео contexts:**

| Device | R1 (min) | R2 (min) |
|---|---|---|
| Видеокамеры | 1.5 | 1.5 |
| Микрофоны | 0.5 | 0.5 |
| Системный блок (видео сервер) | 10 | 40 |

> Note: "серий А6, Аларм", "А16-512", "Расширитель", "Galaxy 512 (контроллер АСПС и СО)" appear in both ОС and ПС. They are **one device_type record each** with **two context rows** (different R1/R2 per system where applicable).

### 4.3 Object Equipment Inventory (FR-03)

Each object's equipment is recorded in two layers:

#### Layer 1 — Physical Inventory
What hardware physically exists on site, independent of which system it serves:
- Device type (from catalog)
- `quantity_physical` — how many units are installed

#### Layer 2 — System Assignments
Which maintenance systems a device is counted under, and with what quantity:
- Device type
- System type (must match one of the device's allowed system types)
- `quantity_maintained` — units counted in this system's workload calculation

**Key rules:**
- `quantity_physical` is stored once per (object, device_type) pair. It represents the hardware asset count — how many units are physically on site.
- `quantity_maintained` is stored once per (object, device_type, system_type) triple. It drives all workload calculations.
- A device can be assigned to multiple system types at the same object. Each assignment uses `quantity_maintained` independently with its own R1/R2 normatives and visit frequency. The physical quantity is not divided — it is reused across all system assignments.
- `quantity_maintained` may legitimately equal `quantity_physical` even when assigned to multiple systems (e.g., 1 physical Galaxy 512 box maintained under both ОС and ПС — `quantity_maintained = 1` in the ОС assignment AND `quantity_maintained = 1` in the ПС assignment). This reflects that the same hardware requires maintenance time counted separately under each system's schedule. This is intentional and not an error.
- `quantity_maintained` cannot exceed `quantity_physical` per individual assignment, but the sum of all `quantity_maintained` values across assignments for the same device at the same object can exceed `quantity_physical`. The UI shows a non-blocking informational warning when a single `quantity_maintained` value exceeds `quantity_physical` for that device at that object.
- Only device types with a valid `device_system_contexts` row for the chosen system type may be assigned to that system. This is enforced at UI, API, and DB levels.
- A device must first appear in the Physical Inventory (Section A of the Equipment tab) before it can be assigned to a system (Section B). The workflow is always: add to inventory → assign to systems.

### 4.4 Records & Administration Tasks (FR-04) — "Записи"
Quantities of service requests/tasks per object per 6-month planning period:

| Task | Unit | Normative (min/unit) |
|---|---|---|
| Запросы в связи с отсутствием (нарушением) доступа | requests | 60 |
| Запросы в связи с проведением мониторинга записей | requests | 180 |
| Запросы по предоставлению записей системы видеонаблюдения | requests | 180 |
| Контроль процесса резервного копирования одной системы | instances | 120 |
| Администрирование систем безопасности филиала | instances | 60 |

Records normatives are fixed constants, not device-system contexts. They have no R2 and no system type.

### 4.5 Repair Type Catalog & Object Repairs (FR-05) — "Ремонт"

Repair types are also admin-managed catalog entries (same extensibility principle as devices). Each repair type has a fixed time in minutes per operation. Admins can add new repair types without schema changes.

Count of repair operations per object per 6-month planning period is recorded per repair type.

**К-во ремонтов** (total repair count) = `SUM(all repair counts)` for the object — always computed, never a user input.

**Seed repair types from source XLSX:**

| Repair Type | Time (min) |
|---|---|
| Замена ПКП серии А6 ОС | 150 |
| Замена ПКП серии А6 ПС | 150 |
| Замена извещателя охранного оптико-электронного | 15 |
| Замена извещателя пожарного дымового | 12 |
| Замена шунтирующих/оконечного резисторов шлейфа ОС | 35 |
| Замена шунтирующих/оконечного резисторов шлейфа ПС | 35 |
| Замена блока бесперебойного питания ОС | 30 |
| Замена блока бесперебойного питания ПС | 30 |
| Замена аккумулятора ОС | 5 |
| Замена аккумулятора ПС | 5 |
| Замена блока питания видеосервера | 20 |
| Замена винчестера видеосервера | 10 |
| Замена основных составных частей видеосервера в комплексе | 30 |
| Переустановка ПО на видеосервере | 90 |
| Восстановление сигнала IP камеры | 35 |
| Восстановление сигнала аналоговой камеры | 20 |
| Акт о выполненных работах ОС | 7 |
| Дефектный акт ОС | 60 |
| Акт на списание ТМЦ из подотчета ОС | 20 |
| Акт о выполненных работах ПС | 7 |
| Дефектный акт ПС | 60 |
| Акт на списание ТМЦ из подотчета ПС | 20 |
| Акт о выполненных работах Видео | 7 |
| Дефектный акт Видео | 60 |
| Акт на списание ТМЦ из подотчета Видео | 20 |

### 4.6 Travel Data (FR-06) — "Дорога"
Per object:
- Transport type (dropdown: пешком / на машине / общественный транспорт)
- Distance, km (numeric, ≥ 0)
- One-way travel time, minutes (numeric, ≥ 0)
- Round-trip time — **auto-calculated** as `one_way_time × 2`, never user-editable

### 4.7 Consolidated Summary — СВОД (FR-07)
Per object, the СВОД shows:
- ПЗВ: fixed 20 min
- Travel: round-trip minutes
- Monthly average ТО time per system (ОС, ПС, Видео)
- Monthly average time for Записи
- Repair time without travel overhead (monthly)
- Repair time with travel overhead (monthly)
- **ИТОГО Числ (без дороги)** and **ИТОГО Числ (с дорогой)** — headcount coefficients
- Р1 and Р2 per-visit totals across all systems

### 4.8 Export (FR-08)
- Export СВОД to XLSX (matching original template column structure).
- Export СВОД to PDF.
- Export individual object inventory data to XLSX.

### 4.9 Dashboard (FR-09)
- Total headcount (ИТОГО Числ) per division.
- Number of objects per engineer with their load ratio.
- Objects with no system assignments (data gaps).
- Objects with no engineer assigned (coverage gaps).
- Top 10 objects by workload.

### 4.10 Engineer Management (FR-10)

Admins manage engineers through the User Management interface. An engineer is a `users` record with `role = 'engineer'` plus the following additional fields:

- `capacity_fte` — FTE capacity, decimal (e.g. `1.0` full-time, `0.5` half-time). Default: `1.0`.
- `home_division_id` — the division this engineer is primarily associated with, for display and filtering. Does **not** restrict which objects they can be assigned to.
- `employee_id` — optional internal HR identifier.

Admin operations:
- Create / edit / deactivate engineer accounts.
- Set or update `capacity_fte` at any time — triggers recalculation of that engineer's summary.
- View all engineers with their current load ratio and status.

### 4.11 Object-Engineer Assignments (FR-11)

An object can have zero, one, or many engineers assigned (joint responsibility without system-level split).

**Assignment rules:**
- Admins and editors (within their division) manage assignments via the Object Detail page or the Engineer Detail page.
- There is no system-type constraint on assignments — an engineer assigned to an object is responsible for all maintenance at that object.
- Assignments record `assigned_at` timestamp for audit purposes.
- An engineer can be assigned to objects in any division (cross-division allowed).
- Removing an assignment immediately triggers recalculation of that engineer's workload summary.

**Workload split:** When multiple engineers share an object, the object's `itogo_chislo_with_travel` is divided **equally** among all assigned engineers. The split ratio is `1 / COUNT(assigned engineers at object)` and is recomputed dynamically — not stored.

**Coverage gap:** An object with zero assigned engineers is flagged as a coverage gap in division reports and the dashboard.

---

## 5. Data Model

### 5.1 Entity Relationship Overview

```
Division ──< Branch ──< Object ──────────────────────────────┐
                           │                                  │
           ┌───────────────┼──────────────────┐    object_engineers (join)
           │               │                  │              │
    object_devices   object_system_      records_tasks       │
    (physical qty)   assignments         (Записи)       engineers/users
           │         (maintained qty)              (capacity_fte, home_division)
           │               │                                  │
           └───────┬───────┘                    engineer_summaries (computed)
             device_type_id ──> device_types
                   │                 │
             system_type ──> device_system_contexts
                             (R1/R2 normatives)
           │
    ┌──────┴──────┐
 object_       travel
 repairs
    │
 repair_type_id ──> repair_types

summaries (per-object computed cache)
engineer_summaries (per-engineer computed cache)
```

### 5.2 Tables

#### `divisions`
```sql
id          UUID         PK
name        VARCHAR(255) NOT NULL UNIQUE
created_at  TIMESTAMP
updated_at  TIMESTAMP
```

#### `branches`
```sql
id          UUID         PK
division_id UUID         FK → divisions.id NOT NULL
name        VARCHAR(255) NOT NULL
created_at  TIMESTAMP
updated_at  TIMESTAMP
UNIQUE(division_id, name)
```

#### `objects`
```sql
id                   UUID         PK
branch_id            UUID         FK → branches.id NOT NULL
name                 VARCHAR(500) NOT NULL   -- address / description
responsible_engineer VARCHAR(255)
import_seq_no        INTEGER                 -- original № from Excel
created_at           TIMESTAMP
updated_at           TIMESTAMP
```

---

#### `device_types`
```sql
id          UUID         PK
name        VARCHAR(255) NOT NULL UNIQUE
description TEXT
created_at  TIMESTAMP
updated_at  TIMESTAMP
```

One row per named device. "Galaxy 512 (контроллер АСПС и СО)" and "Galaxy 512 (Galaxy Dimension GD-520)" are two separate rows because they have distinct names and normatives.

#### `device_system_contexts`
```sql
id             UUID          PK
device_type_id UUID          FK → device_types.id NOT NULL
system_type    VARCHAR(10)   NOT NULL   -- 'ОС' | 'ПС' | 'Видео'
r1_minutes     DECIMAL(10,4) NOT NULL
r2_minutes     DECIMAL(10,4) NOT NULL
created_at     TIMESTAMP
updated_at     TIMESTAMP
UNIQUE(device_type_id, system_type)
```

This table is the normatives store. A (device, system_type) pair without a row here is not a valid assignment target. Enforced by FK from `object_system_assignments`.

---

#### `object_devices` — Physical Inventory
```sql
id                UUID          PK
object_id         UUID          FK → objects.id NOT NULL
device_type_id    UUID          FK → device_types.id NOT NULL
quantity_physical DECIMAL(10,2) NOT NULL DEFAULT 0
updated_at        TIMESTAMP
UNIQUE(object_id, device_type_id)
```

#### `object_system_assignments` — Maintenance Assignments
```sql
id                  UUID          PK
object_id           UUID          FK → objects.id NOT NULL
device_type_id      UUID          FK → device_types.id NOT NULL
system_type         VARCHAR(10)   NOT NULL   -- 'ОС' | 'ПС' | 'Видео'
quantity_maintained DECIMAL(10,2) NOT NULL DEFAULT 0
context_id          UUID          FK → device_system_contexts.id NOT NULL
                                  -- ON DELETE RESTRICT
updated_at          TIMESTAMP
UNIQUE(object_id, device_type_id, system_type)
```

`context_id` references the `device_system_contexts` row where `device_type_id` and `system_type` match. This FK guarantees the (device, system) combination is valid. `ON DELETE RESTRICT` prevents deleting a context that has active assignments.

---

#### `records_tasks`
```sql
id                  UUID          PK
object_id           UUID          FK → objects.id UNIQUE NOT NULL
access_requests     DECIMAL(10,2) NOT NULL DEFAULT 0
monitoring_requests DECIMAL(10,2) NOT NULL DEFAULT 0
footage_requests    DECIMAL(10,2) NOT NULL DEFAULT 0
backup_control      DECIMAL(10,2) NOT NULL DEFAULT 0
security_admin      DECIMAL(10,2) NOT NULL DEFAULT 0
updated_at          TIMESTAMP
```

#### `repair_types`
```sql
id           UUID          PK
name         VARCHAR(255)  NOT NULL UNIQUE
time_minutes DECIMAL(10,2) NOT NULL
created_at   TIMESTAMP
updated_at   TIMESTAMP
```

Admin-managed catalog. New entries added via UI without schema changes.

#### `object_repairs`
```sql
id             UUID    PK
object_id      UUID    FK → objects.id NOT NULL
repair_type_id UUID    FK → repair_types.id NOT NULL
                        -- ON DELETE RESTRICT
count          INTEGER NOT NULL DEFAULT 0
updated_at     TIMESTAMP
UNIQUE(object_id, repair_type_id)
```

#### `travel`
```sql
id               UUID          PK
object_id        UUID          FK → objects.id UNIQUE NOT NULL
transport_type   VARCHAR(100)
distance_km      DECIMAL(8,2)  NOT NULL DEFAULT 0
one_way_time_min DECIMAL(8,2)  NOT NULL DEFAULT 0
-- round_trip_min is always derived: one_way_time_min × 2. Never stored.
updated_at       TIMESTAMP
```

#### `app_config`
```sql
key         VARCHAR(100) PK
value       VARCHAR(255) NOT NULL
description TEXT
updated_at  TIMESTAMP
updated_by  UUID         FK → users.id
```

All named calculation constants (see §6.12). Changes to any key trigger bulk summary invalidation.

#### `summaries` — Computed Cache
```sql
id                         UUID           PK
object_id                  UUID           FK → objects.id UNIQUE NOT NULL

-- Per-system per-visit subtotals (→ СВОД cols R, S)
os_r1_per_visit            DECIMAL(10,4)
os_r2_per_visit            DECIMAL(10,4)
ps_r1_per_visit            DECIMAL(10,4)
ps_r2_per_visit            DECIMAL(10,4)
video_r1_per_visit         DECIMAL(10,4)
video_r2_per_visit         DECIMAL(10,4)
r1_per_visit_total         DECIMAL(10,4)   -- СВОД col R
r2_per_visit_total         DECIMAL(10,4)   -- СВОД col S

-- Monthly averages: (R1_annual + R2_annual) / 12
os_monthly_avg             DECIMAL(10,4)   -- СВОД col "Охрана"
ps_monthly_avg             DECIMAL(10,4)   -- СВОД col "Пожарная сигнализация"
video_monthly_avg          DECIMAL(10,4)   -- СВОД col "Видео"

-- Records (6-month total ÷ 6)
records_6months            DECIMAL(10,4)
records_monthly            DECIMAL(10,4)   -- СВОД col "Записи"

-- Repairs (6-month horizon ÷ 5)
total_repairs              INTEGER         -- К-во ремонтов
repair_work_6months        DECIMAL(10,4)
repair_travel_6months      DECIMAL(10,4)   -- total_repairs × round_trip_min
repair_pzv_6months         DECIMAL(10,4)   -- total_repairs × PZV_MINUTES
repair_no_travel_monthly   DECIMAL(10,4)   -- СВОД col "Ремонт без дороги"
repair_with_travel_monthly DECIMAL(10,4)   -- СВОД col "Ремонт с дорогой"

-- Travel
round_trip_min             DECIMAL(10,4)   -- one_way_time_min × 2
pzv_minutes                DECIMAL(10,4)   -- PZV_MINUTES at compute time

-- СВОД final totals
total_no_travel_min        DECIMAL(10,4)   -- СВОД col "ТО+ремонт(без дороги)+Дорога"
itogo_chislo_no_travel     DECIMAL(14,10)  -- СВОД col "ИТОГО Числ (без дороги)"
total_with_travel_min      DECIMAL(10,4)   -- СВОД col "ТО+ремонт(с дорогой)+Дорога"
itogo_chislo_with_travel   DECIMAL(14,10)  -- СВОД col "ИТОГО Числ (с дорогой)"

is_stale                   BOOLEAN         NOT NULL DEFAULT FALSE
computed_at                TIMESTAMP
```

#### `users`
```sql
id               UUID          PK
email            VARCHAR(255)  NOT NULL UNIQUE
name             VARCHAR(255)
role             VARCHAR(20)   NOT NULL DEFAULT 'viewer'
                               -- 'admin' | 'editor' | 'viewer' | 'engineer'
division_id      UUID          FK → divisions.id NULL
                               -- scope restriction for editors; NULL = all divisions
home_division_id UUID          FK → divisions.id NULL
                               -- display/filter home for engineers; no access restriction
capacity_fte     DECIMAL(4,2)  NOT NULL DEFAULT 1.0
                               -- meaningful only for role='engineer'; ignored for others
employee_id      VARCHAR(100)  NULL     -- optional HR identifier
password_hash    VARCHAR(255)
created_at       TIMESTAMP
updated_at       TIMESTAMP
```

> **Note:** `division_id` is the **access-control** scope used by editors.  
> `home_division_id` is the **display** home for engineers — they can be assigned to objects in any division regardless of this value.

#### `object_engineers` — Object-Engineer Assignments
```sql
id          UUID      PK
object_id   UUID      FK → objects.id NOT NULL
engineer_id UUID      FK → users.id   NOT NULL  -- must have role = 'engineer'
assigned_at TIMESTAMP NOT NULL DEFAULT now()
assigned_by UUID      FK → users.id   NULL       -- who created the assignment
UNIQUE(object_id, engineer_id)
```

`engineer_count` for workload splitting is computed at query time as  
`COUNT(*) WHERE object_id = :id` — never stored.

---

#### `engineer_summaries` — Per-Engineer Computed Cache
```sql
id                    UUID          PK
engineer_id           UUID          FK → users.id UNIQUE NOT NULL

-- Total load across all assigned objects
total_load            DECIMAL(14,10)  -- SUM of object shares
object_count          INTEGER         -- number of assigned objects

-- Breakdown by system component (engineer's share of each)
os_load               DECIMAL(14,10)
ps_load               DECIMAL(14,10)
video_load            DECIMAL(14,10)
records_load          DECIMAL(14,10)
repair_load           DECIMAL(14,10)

-- Capacity and status
capacity_fte          DECIMAL(4,2)    -- snapshot of capacity at compute time
load_ratio            DECIMAL(10,6)   -- total_load / capacity_fte
status                VARCHAR(20)     -- 'normal' | 'warning' | 'overloaded'

is_stale              BOOLEAN         NOT NULL DEFAULT FALSE
computed_at           TIMESTAMP
```

---

#### `audit_log`
```sql
id          UUID         PK
user_id     UUID         FK → users.id
table_name  VARCHAR(100)
record_id   UUID
action      VARCHAR(20)  -- 'CREATE'|'UPDATE'|'DELETE'
old_value   JSONB
new_value   JSONB
created_at  TIMESTAMP
```

Captures changes to `device_system_contexts`, `device_types`, `repair_types`, `app_config`, `object_engineers`, `users.capacity_fte`.

---

## 6. Calculation Engine

All calculations are performed **server-side only**. The `summaries` table is a precomputed cache. When source data changes, the affected summary is marked `is_stale = TRUE` synchronously; a background job recalculates it asynchronously.

### 6.1 Pipeline Overview

```
Stage 1 — Per-assignment contribution
  For each (object, device, system_type) assignment:
    r1_contrib = quantity_maintained × context.r1_minutes
    r2_contrib = quantity_maintained × context.r2_minutes

Stage 2 — Per-system per-visit subtotals
  For each system S ∈ {ОС, ПС, Видео}:
    R1_per_visit[S] = SUM(r1_contrib) for all assignments where system_type = S
    R2_per_visit[S] = SUM(r2_contrib) for all assignments where system_type = S

Stage 3 — Annual time (system-specific visit frequencies)
  R1_annual[S] = R1_per_visit[S] × config[S_R1_VISITS_PER_YEAR]
  R2_annual[S] = R2_per_visit[S] × config[S_R2_VISITS_PER_YEAR]

Stage 4 — Monthly average per system
  monthly_avg[S] = (R1_annual[S] + R2_annual[S]) / 12

Stage 5 — Records and repairs (§6.5, §6.6)

Stage 6 — СВОД aggregation and final headcount (§6.7, §6.8)
```

### 6.2 Maintenance Visit Frequencies

Visit frequency is tied to **system type**, not device type. Stored in `app_config`:

| Config Key | Value | Meaning |
|---|---|---|
| `OS_R1_VISITS_PER_YEAR` | 10 | ОС routine inspections/year |
| `OS_R2_VISITS_PER_YEAR` | 2 | ОС full maintenance visits/year |
| `PS_R1_VISITS_PER_YEAR` | 8 | ПС routine inspections/year |
| `PS_R2_VISITS_PER_YEAR` | 4 | ПС full maintenance visits/year |
| `VIDEO_R1_VISITS_PER_YEAR` | 10 | Видео routine inspections/year |
| `VIDEO_R2_VISITS_PER_YEAR` | 2 | Видео full maintenance visits/year |

### 6.3 Per-System Monthly Average

```sql
-- Pseudocode for system S:
R1_per_visit[S] = SUM(
    osa.quantity_maintained * dsc.r1_minutes
    FROM object_system_assignments osa
    JOIN device_system_contexts dsc ON dsc.id = osa.context_id
    WHERE osa.object_id = :object_id
    AND   osa.system_type = S
)

R2_per_visit[S] = SUM(
    osa.quantity_maintained * dsc.r2_minutes
    -- same joins and WHERE
)

R1_annual[S]   = R1_per_visit[S] * config[S + '_R1_VISITS_PER_YEAR']
R2_annual[S]   = R2_per_visit[S] * config[S + '_R2_VISITS_PER_YEAR']
monthly_avg[S] = (R1_annual[S] + R2_annual[S]) / 12
```

**Verified example — ОС, Object "Архив г. Брест":**
```
Assignments for ОС:
  серий А6, Аларм × 2:     R1 = 2×5=10,     R2 = 2×8=16
  Устройство доступа × 2:  R1 = 2×1=2,      R2 = 2×4=8
  Шлейфы × 12:             R1 = 12×0.06=0.72, R2 = 12×0.7=8.4
  Каналы × 2:              R1 = 2×0.02=0.04,  R2 = 2×1.5=3
  Извещатели × 21:         R1 = 21×0.7=14.7,  R2 = 21×3=63

R1_per_visit = 29.14,  R2_per_visit = 98.4
R1_annual    = 29.14 × 10 = 291.4
R2_annual    = 98.4  × 2  = 196.8
monthly_avg  = 488.2 / 12 = 40.683 min  ✓
```

**Verified example — ПС, Object "Архив г. Брест":**
```
R1_per_visit = 10.52,  R2_per_visit = 73.0
R1_annual    = 10.52 × 8 = 84.16
R2_annual    = 73.0  × 4 = 292.0
monthly_avg  = 365.0 / 12 = 30.417 min  ✓
```

### 6.4 Shared-Device Multi-System Example

**Scenario:** One Galaxy 512 (контроллер АСПС и СО) box physically installed at an object, responsible for both ОС and ПС functions.

**Database state:**
```
object_devices:
  { object_X, "Galaxy 512 контроллер", quantity_physical: 1 }
  -- One physical box on site

object_system_assignments:
  { object_X, "Galaxy 512 контроллер", system_type: ОС,
    quantity_maintained: 1, context_id: → dsc[Galaxy 512 контроллер, ОС] }
  { object_X, "Galaxy 512 контроллер", system_type: ПС,
    quantity_maintained: 1, context_id: → dsc[Galaxy 512 контроллер, ПС] }
  -- Same box counted in both systems' maintenance schedules
```

**Calculation contribution:**
```
ОС stage:
  R1_contrib = 1 × 15 = 15 min  (ОС context r1)
  R2_contrib = 1 × 20 = 20 min  (ОС context r2)

ПС stage:
  R1_contrib = 1 × 15 = 15 min  (ПС context r1 — same value here, but independent)
  R2_contrib = 1 × 20 = 20 min  (ПС context r2)

Both contributions feed into their respective system monthly_avg calculations
using ОС visit frequency (10R1+2R2) and ПС visit frequency (8R1+4R2) respectively.
```

**UI representation in the Equipment tab:**
```
Galaxy 512 (контроллер АСПС и СО)  [Physical: 1 unit]
  ├── ОС   qty_maintained: [1]   R1: 15 min   R2: 20 min   [Remove]
  └── ПС   qty_maintained: [1]   R1: 15 min   R2: 20 min   [Remove]
     [+ Assign to system ▾]  → dropdown: Видео (if context exists) only
```

No warning is shown: 1 maintained (per assignment) = 1 physical. The sum
across assignments is 2, but this is expected and does not trigger any alert.

### 6.5 Records Monthly Average

```
records_6months = SUM(task_quantity[j] × records_normative[j])  for all j
records_monthly = records_6months / 6
```

### 6.6 Repair Monthly Averages

```
total_repairs          = SUM(object_repairs.count)

repair_work_6months    = SUM(object_repairs.count × repair_types.time_minutes)

round_trip_min         = travel.one_way_time_min × 2

repair_travel_6months  = total_repairs × round_trip_min
repair_pzv_6months     = total_repairs × config[PZV_MINUTES]

repair_no_travel_monthly   = repair_work_6months / config[REPAIR_PRODUCTIVE_MONTHS]
repair_with_travel_monthly = (repair_work_6months
                               + repair_travel_6months
                               + repair_pzv_6months)
                             / config[REPAIR_PRODUCTIVE_MONTHS]
```

> Divisor is `REPAIR_PRODUCTIVE_MONTHS = 5`, not 6. See §13 C-12.

**Verified — Object "Архив г. Брест":**
```
repair_work_6months  = 361 min
total_repairs        = 8
round_trip_min       = 20
repair_travel_6months = 8 × 20 = 160
repair_pzv_6months    = 8 × 20 = 160

repair_no_travel_monthly   = 361 / 5 = 72.2  ✓
repair_with_travel_monthly = (361+160+160)/5 = 136.2  ✓
```

### 6.7 R1/R2 Per-Visit Reference Totals

```
r1_per_visit_total = R1_per_visit[ОС] + R1_per_visit[ПС] + R1_per_visit[Видео]
r2_per_visit_total = R2_per_visit[ОС] + R2_per_visit[ПС] + R2_per_visit[Видео]
```

Stored in `summaries` for СВОД columns R and S. Not used in headcount formula.

**Verified — Object "Архив г. Брест":**
```
r1_total = 29.14 + 10.52 + 0 = 39.66  ✓ (СВОД col R)
r2_total = 98.4  + 73.0  + 0 = 171.4  ✓ (СВОД col S)
```

### 6.8 СВОД Monthly Totals and Final Headcount

```
pzv = config[PZV_MINUTES]   -- 20

total_no_travel_min = pzv + round_trip_min
                    + os_monthly_avg + ps_monthly_avg + video_monthly_avg
                    + records_monthly + repair_no_travel_monthly

total_with_travel_min = pzv + round_trip_min
                      + os_monthly_avg + ps_monthly_avg + video_monthly_avg
                      + records_monthly + repair_with_travel_monthly

itogo_chislo_no_travel   = total_no_travel_min   / 60 / config[MONTHLY_HOURS_FUND]
                           × config[ABSENCE_COEFFICIENT]
itogo_chislo_with_travel = total_with_travel_min / 60 / config[MONTHLY_HOURS_FUND]
                           × config[ABSENCE_COEFFICIENT]
```

**Verified — Object "Архив г. Брест":**
```
total_no_travel   = 20+20+40.683+30.417+0+0+72.2  = 183.3 min  ✓
total_with_travel = 20+20+40.683+30.417+0+0+136.2 = 247.3 min  ✓
183.3 / 60 / 142.8 × 1.12 = 0.023961  ✓
247.3 / 60 / 142.8 × 1.12 = 0.032327  ✓
```

### 6.9 Division-Level Aggregation

```
division_headcount = SUM(itogo_chislo_with_travel)  for all objects in division
```

### 6.10 Cache Invalidation Rules

| Triggering Event | Action |
|---|---|
| `object_system_assignments` INSERT / UPDATE / DELETE | Mark this object's summary `is_stale = TRUE` |
| `object_devices` INSERT / UPDATE / DELETE | Mark this object's summary `is_stale = TRUE` |
| `records_tasks` UPDATE | Mark this object's summary `is_stale = TRUE` |
| `object_repairs` UPDATE | Mark this object's summary `is_stale = TRUE` |
| `travel` UPDATE | Mark this object's summary `is_stale = TRUE` |
| `device_system_contexts` UPDATE (r1 or r2) | Mark `is_stale = TRUE` for ALL objects with assignments referencing this context |
| `repair_types.time_minutes` UPDATE | Mark `is_stale = TRUE` for ALL objects with this repair type |
| `app_config` UPDATE (any calculation key) | Mark ALL summaries `is_stale = TRUE` |

Staleness flag is set **in the same DB transaction** as the data change. Background worker polls for stale records and recalculates in batches.

### 6.11 Application Configuration Constants

| Key | Default | Description |
|---|---|---|
| `MONTHLY_HOURS_FUND` | 142.8 | Monthly working hours per employee |
| `ABSENCE_COEFFICIENT` | 1.12 | Absence coefficient (коэффициент невыходов) |
| `PZV_MINUTES` | 20 | Prep/wrap-up time per visit (minutes) |
| `OS_R1_VISITS_PER_YEAR` | 10 | ОС routine visits/year |
| `OS_R2_VISITS_PER_YEAR` | 2 | ОС full maintenance visits/year |
| `PS_R1_VISITS_PER_YEAR` | 8 | ПС routine visits/year |
| `PS_R2_VISITS_PER_YEAR` | 4 | ПС full maintenance visits/year |
| `VIDEO_R1_VISITS_PER_YEAR` | 10 | Видео routine visits/year |
| `VIDEO_R2_VISITS_PER_YEAR` | 2 | Видео full maintenance visits/year |
| `REPAIR_PLANNING_MONTHS` | 6 | Repair planning horizon (months) |
| `REPAIR_PRODUCTIVE_MONTHS` | 5 | Divisor for repair monthly averaging |

All constants are stored in `app_config`, editable by admins at `/admin/config`. Never hardcoded in application logic.

### 6.12 Engineer Workload Calculation

Engineer workload is computed from cached `summaries` rows. It is recalculated whenever:
- An object summary changes for any object the engineer is assigned to.
- The set of engineers at an object changes (split ratio changes for all co-assigned engineers).
- `users.capacity_fte` changes for that engineer.
- `app_config[ENGINEER_WARNING_THRESHOLD]` changes (status may flip).

#### 6.12.1 Object Share per Engineer

For each object an engineer is assigned to:

```
engineer_count[object]   = COUNT(rows in object_engineers WHERE object_id = object)

object_share[engineer, object] = summaries.itogo_chislo_with_travel / engineer_count
```

The split is always equal regardless of when each engineer was assigned.

#### 6.12.2 System-Component Breakdown per Engineer

To support the breakdown by system type, apply the headcount formula to each system's monthly average *before* splitting:

```
-- For system component X (os, ps, video, records, repair):
component_coefficient[object, X] = monthly_avg_X / 60
                                   / config[MONTHLY_HOURS_FUND]
                                   × config[ABSENCE_COEFFICIENT]

engineer_component_share[engineer, object, X]
    = component_coefficient[object, X] / engineer_count[object]
```

Note: `os + ps + video + records + repair` components sum to approximately
`itogo_chislo_with_travel` (minor floating-point rounding acceptable).

#### 6.12.3 Engineer Total and Breakdown

```
engineer_total_load = SUM(object_share[engineer, object])
                      for all objects engineer is assigned to

engineer_os_load      = SUM(engineer_component_share[engineer, object, os])
engineer_ps_load      = SUM(engineer_component_share[engineer, object, ps])
engineer_video_load   = SUM(engineer_component_share[engineer, object, video])
engineer_records_load = SUM(engineer_component_share[engineer, object, records])
engineer_repair_load  = SUM(engineer_component_share[engineer, object, repair])
```

All results stored in `engineer_summaries`.

### 6.13 Capacity and Overload Status

```
load_ratio = engineer_total_load / users.capacity_fte

status:
  load_ratio < config[ENGINEER_WARNING_THRESHOLD]   → "normal"
  load_ratio >= config[ENGINEER_WARNING_THRESHOLD]
    AND load_ratio < 1.0                             → "warning"
  load_ratio >= 1.0                                  → "overloaded"
```

Two new `app_config` keys:

| Key | Default | Description |
|---|---|---|
| `ENGINEER_WARNING_THRESHOLD` | 0.9 | Load ratio at which status becomes "warning" |

The overload boundary is always exactly `1.0` (load = capacity) and is not configurable.

### 6.14 Engineer Cache Invalidation Rules

| Triggering Event | Affected Engineers | Action |
|---|---|---|
| `summaries.itogo_chislo_with_travel` updated for object X | All engineers assigned to object X | Mark their `engineer_summaries.is_stale = TRUE` |
| `object_engineers` INSERT (new assignment at object X) | All engineers now assigned to X (split ratio changed) | Mark all `is_stale = TRUE` |
| `object_engineers` DELETE (assignment removed at object X) | All remaining engineers at X + the removed engineer | Mark all `is_stale = TRUE` |
| `users.capacity_fte` UPDATE for engineer E | Engineer E only | Mark E's `engineer_summaries.is_stale = TRUE` |
| `app_config[ENGINEER_WARNING_THRESHOLD]` UPDATE | All engineers | Mark all `engineer_summaries.is_stale = TRUE` |

Staleness is set in the same transaction as the triggering change. Background worker recalculates stale engineer summaries after object summaries are current (object summaries must be fresh before engineer summaries can be computed).


---

## 7. User Interface Requirements

### 7.1 Pages / Views

| Route | View | Description |
|---|---|---|
| `/` | Dashboard | Headcount cards by division; top objects by workload |
| `/divisions` | Division List | List/search divisions |
| `/divisions/:id` | Division Detail | Objects with СВОД subtotals |
| `/objects` | Object List | Searchable, filterable table |
| `/objects/new` | Object Create | Create new object |
| `/objects/:id` | Object Detail | Tabbed detail view |
| `/objects/:id/edit` | Object Edit | Edit object metadata |
| `/svod` | СВОД | Full summary table with filters and export |
| `/catalog/devices` | Device Catalog | List / create / edit device types and contexts |
| `/catalog/devices/new` | New Device | Create device type and assign system contexts |
| `/catalog/devices/:id` | Device Detail | View/edit device and all system contexts |
| `/catalog/repairs` | Repair Types | List / create / edit repair type catalog |
| `/admin/config` | App Config | View/edit all calculation constants (admin only) |
| `/admin/users` | Users | User management (admin only) |
| `/import` | Import | XLSX import wizard |
| `/export` | Export | Export options |
| `/engineers` | Engineer List | All engineers with load ratio and status |
| `/engineers/:id` | Engineer Detail | Workload dashboard for one engineer |
| `/engineers/:id/edit` | Engineer Edit | Edit name, capacity, home division |

### 7.2 Object Detail Page — Tabs

1. **Оборудование** — Two-layer equipment view (physical inventory + system assignments)
2. **Записи** — Records/admin task counts
3. **Ремонт** — Repair operation counts (dynamic from repair_types catalog)
4. **Дорога** — Travel data
5. **Инженеры** — Assigned engineers list with their share of this object's workload
6. **СВОД** — Computed summary (read-only; shows stale indicator when `is_stale = TRUE`)

### 7.3 Equipment Tab — Two-Layer UI

**Section A — Physical Inventory**

A table of all device types at this object with their physical quantities:

| Device | Physical Qty | Actions |
|---|---|---|
| Galaxy 512 (контроллер АСПС и СО) | 1 | Edit / Remove |
| серий А6, Аларм | 2 | Edit / Remove |

"Add device" opens a searchable dropdown of all `device_types`. Adding a device here creates an `object_devices` row and makes it available for Section B.

**Section B — System Assignments**

A grouped view showing which systems each device is assigned to:

```
Galaxy 512 (контроллер АСПС и СО)  [Physical qty: 1]
  ├── ОС   qty_maintained: [1]   R1: 15 min  R2: 20 min   [Remove]
  └── ПС   qty_maintained: [1]   R1: 15 min  R2: 20 min   [Remove]
     [+ Assign to system ▾]  ← shows only Видео (if context exists) or empty

серий А6, Аларм  [Physical qty: 2]
  └── ОС   qty_maintained: [2]   R1: 5 min   R2: 8 min    [Remove]
     [+ Assign to system ▾]  ← shows ПС (context exists), Видео (no context → hidden)
```

**UX rules:**
- **Workflow order:** A device must be added to Section A (Physical Inventory) before it can appear in Section B (System Assignments). The "Assign to system" button in Section B is only available for devices already in the physical inventory.
- **R1/R2 are read-only in this view.** Values shown per assignment are pulled from `device_system_contexts` and cannot be edited here. A "Edit normatives →" link navigates to the Device Catalog page for that device.
- **System type dropdown** in "Assign to system" shows **only** system types for which a `device_system_contexts` row exists for that device. System types with no context are hidden entirely — not grayed out.
- `quantity_maintained` is editable inline per assignment row. Saving any value triggers an asynchronous summary recalculation; the СВОД tab shows a "Пересчитывается..." indicator.
- **Warning rule:** When `quantity_maintained > quantity_physical` for a single assignment row, display a yellow ⚠ icon and tooltip: **"Обслуживаемое количество (N) превышает физическое (M)"**. This is informational — it does not block saving.
- **Cascade on removal:** Removing a device from Section A (physical inventory) cascades to remove all its system assignments at this object. A confirmation dialog lists all affected assignments (e.g., "Это удалит назначения: ОС × 1, ПС × 1. Продолжить?") before proceeding.
- **Preventing orphaned assignments:** The API enforces that an `object_system_assignments` row cannot exist without a corresponding `object_devices` row for the same (object_id, device_type_id). Enforced at the application layer (not FK, since they are separate tables).

### 7.4 Engineers Tab (Object Detail)

Shows all engineers assigned to this object and their workload share:

```
Назначенные инженеры                              [+ Назначить инженера]

┌──────────────────────────┬──────────────┬───────────┬──────────┐
│ Инженер                  │ Доля объекта │ Загрузка  │          │
├──────────────────────────┼──────────────┼───────────┼──────────┤
│ Иванов Петр Сергеевич    │ 0.0161 FTE   │ ████░ 82% │ Снять    │
│ Сидорова Анна Николаевна │ 0.0161 FTE   │ ██░░░ 45% │ Снять    │
└──────────────────────────┴──────────────┴───────────┴──────────┘
  Split: 2 engineers → each gets 50% of object load (0.0323 / 2)
```

- "Доля объекта" = `itogo_chislo_with_travel / engineer_count` for this object.
- "Загрузка" = that engineer's total load ratio across ALL their objects (not just this one). Provides context — assigns may push an already-loaded engineer into overload.
- "+" button opens a searchable dropdown of all active engineers (not restricted by division).
- Removing an engineer triggers immediate recalculation of all remaining engineers at this object.

### 7.5 Engineer List Page (`/engineers`)

A table of all engineers with sortable columns:

```
┌──────────────────────────┬──────────────┬───────────┬───────────┬──────────────┬──────────┐
│ Инженер                  │ Подразделение│ Объектов  │ Нагрузка  │ Мощность     │ Статус   │
├──────────────────────────┼──────────────┼───────────┼───────────┼──────────────┼──────────┤
│ Иванов Петр              │ Брест №100   │ 47        │ 0.92 FTE  │ 1.0          │ ⚠ 92%    │
│ Сидорова Анна            │ Минск №200   │ 31        │ 1.08 FTE  │ 1.0          │ 🔴 108%  │
│ Козлов Дмитрий           │ Гродно №300  │ 12        │ 0.24 FTE  │ 0.5          │ ✅ 48%   │
└──────────────────────────┴──────────────┴───────────┴───────────┴──────────────┴──────────┘
```

Status icons: ✅ normal (< warning threshold) · ⚠ warning (≥ threshold, < 1.0) · 🔴 overloaded (≥ 1.0).

Filters: by home division, by status (normal / warning / overloaded), search by name.

### 7.6 Engineer Detail Page (`/engineers/:id`)

A personal workload dashboard with five sections:

**Section 1 — Summary cards**
```
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│ Нагрузка        │ Мощность        │ Загрузка        │ Объектов        │
│ 0.921 FTE       │ 1.0 FTE         │ 92.1%  ⚠        │ 47              │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

**Section 2 — Breakdown by system type** (horizontal bar or pie)
```
ОС      ████████████░░░ 0.41 FTE  (45%)
ПС      ██████░░░░░░░░░ 0.27 FTE  (29%)
Видео   ██░░░░░░░░░░░░░ 0.09 FTE  (10%)
Записи  █░░░░░░░░░░░░░░ 0.05 FTE  (5%)
Ремонт  ██░░░░░░░░░░░░░ 0.10 FTE  (11%)
```

**Section 3 — Assigned objects** (table, sorted by this engineer's share descending)
```
┌──────────────────────────────────────┬──────────────┬────────────────┬──────────────┐
│ Объект                               │ Доля инженера│ Всего на объект│ Кол-во инж.  │
├──────────────────────────────────────┼──────────────┼────────────────┼──────────────┤
│ ЦБУ г.Брест, ул.Ленина, 10           │ 0.032 FTE    │ 0.064 FTE      │ 2            │
│ Архив г.Брест, ул.Московская, 202Д   │ 0.024 FTE    │ 0.024 FTE      │ 1            │
│ ...                                  │ ...          │ ...            │ ...          │
└──────────────────────────────────────┴──────────────┴────────────────┴──────────────┘
```

**Section 4 — Assign / remove objects** (admin and editor only)
Button to assign additional objects; remove button per row.

**Section 5 — Stale indicator**
If `engineer_summaries.is_stale = TRUE`: banner "Данные пересчитываются..." across the page.

### 7.7 Division Detail — Coverage Gaps

On the Division Detail page (`/divisions/:id`), add a "Непокрытые объекты" section below the main СВОД table:

```
⚠ 12 объектов без назначенного инженера

┌─────────────────────────────────────────────────┬──────────────┐
│ Объект                                          │ Нагрузка     │
├─────────────────────────────────────────────────┼──────────────┤
│ Инфокиоск INF 00635 г.Брест ТЦ "Варшавский"     │ 0.008 FTE    │
│ ...                                             │ ...          │
└─────────────────────────────────────────────────┴──────────────┘
[Назначить инженеров →]  button linking to bulk-assign workflow
```

### 7.8 Device Catalog Page

**Device list:** searchable table of all device types, with columns showing valid systems (ОС ✓ / ПС ✓ / Видео —).

**Device detail / edit view:**

```
Device name:  [Galaxy 512 (контроллер АСПС и СО)         ]
Description:  [Combined fire and security controller      ]

System Contexts:
┌─────────────┬──────────────┬──────────────┬──────────┐
│ System Type │ R1 (min/unit)│ R2 (min/unit)│ Actions  │
├─────────────┼──────────────┼──────────────┼──────────┤
│ ОС          │ 15           │ 20           │ Edit  Del │
│ ПС          │ 15           │ 20           │ Edit  Del │
└─────────────┴──────────────┴──────────────┴──────────┘
[+ Add system context ▾]  ← shows only Видео (ОС and ПС already exist)
```

Deleting a context that has active `object_system_assignments` is **blocked** at two levels:
- **Application level:** Before deletion, the API queries for active assignments and returns HTTP 409 with a payload listing the count of affected objects and a sample list of up to 10 object names.
- **Database level:** `ON DELETE RESTRICT` FK on `object_system_assignments.context_id` prevents deletion even if the application-level check is bypassed.

The UI pre-checks on click and shows: **"Нельзя удалить: 42 объекта используют этот контекст. Сначала снимите назначения."**

### 7.9 СВОД Table Columns

| # | Column | Source |
|---|---|---|
| 1 | № | `object.import_seq_no` |
| 2 | Подразделение | `division.name` |
| 3 | Филиал | `branch.name` |
| 4 | Значение | `object.name` |
| 5 | Ответственные ТО | `object.responsible_engineer` |
| 6 | ПЗВ | `config[PZV_MINUTES]` |
| 7 | Дорога | `summaries.round_trip_min` |
| 8 | Пожарная сигнализация | `summaries.ps_monthly_avg` |
| 9 | Видео | `summaries.video_monthly_avg` |
| 10 | Охрана | `summaries.os_monthly_avg` |
| 11 | Записи | `summaries.records_monthly` |
| 12 | Ремонт без дороги | `summaries.repair_no_travel_monthly` |
| 13 | ТО+записи+ремонт(без дороги)+Дорога, мин | `summaries.total_no_travel_min` |
| 14 | ИТОГО Числ (без дороги) | `summaries.itogo_chislo_no_travel` |
| 15 | Ремонт с дорогой | `summaries.repair_with_travel_monthly` |
| 16 | ТО+записи+ремонт(с дорогой)+Дорога, мин | `summaries.total_with_travel_min` |
| 17 | ИТОГО Числ (с дорогой) | `summaries.itogo_chislo_with_travel` |
| 18 | Р1 на объекте всех систем | `summaries.r1_per_visit_total` |
| 19 | Р2 на объекте всех систем | `summaries.r2_per_visit_total` |

Stale rows (`is_stale = TRUE`) display a "Пересчитывается..." indicator in place of numeric values.

### 7.10 UI/UX Constraints
- All user-facing labels in **Russian**.
- СВОД table sortable by any column, filterable by division / branch / engineer.
- Repair tab is a dynamic list from `repair_types` — never hardcoded input fields.
- Zero values may display as blank (matching Excel behavior) but stored as 0.
- Inline СВОД editing not permitted.
- Engineer load ratio bars use colour coding matching status: green (normal) / amber (warning) / red (overloaded).
- Stale engineer summaries display "Пересчитывается..." placeholder values, never the last stale numbers.

---

## 8. Non-Functional Requirements

### 8.1 Performance
- СВОД page (100 rows, paginated) loads in under 3 seconds.
- Single object summary recalculation completes in under 200ms.
- Bulk recalculation of all 2,935 objects completes within 60 seconds (background job).
- Device catalog assignment dropdown returns results in under 300ms for up to 1,000 device types.
- Engineer detail page loads in under 2 seconds for an engineer with up to 500 assigned objects.
- Engineer list page (all engineers, paginated 50/page) loads in under 2 seconds.

### 8.2 Scalability
- Support up to 10,000 objects and 500 device types without architectural changes.
- Support up to 500 engineers and 50 concurrent users.

### 8.3 Security
- Authentication required for all routes except `/login`.
- HTTPS only in production.
- CSRF protection on all state-changing endpoints.
- Input validation and sanitization on all numeric fields.
- All changes to `device_system_contexts`, `device_types`, `repair_types`, `app_config`, `object_engineers`, and `users.capacity_fte` written to `audit_log`.

### 8.4 Data Integrity
- `ON DELETE RESTRICT` on `object_system_assignments.context_id → device_system_contexts.id`.
- `ON DELETE RESTRICT` on `object_repairs.repair_type_id → repair_types.id`.
- Deleting a `device_type` that has `object_devices` rows is blocked (application-level guard + DB constraint).
- Deleting a `users` record with role `engineer` that has active `object_engineers` rows is blocked until all assignments are removed.
- `is_stale = TRUE` is set in the **same transaction** as the data change — for both `summaries` and `engineer_summaries`.
- All NULL quantities treated as 0 in calculations.
- `engineer_summaries` recalculation must wait for all dependent `summaries` to be current (worker ordering constraint).

### 8.5 Availability
- Uptime target: 99% during business hours (08:00–20:00 local time, Mon–Fri).

### 8.6 Browser Support
- Chrome, Firefox, Edge — latest two major versions.
- Minimum resolution: 1280×768.

---

## 9. Architecture Constraints

### 9.1 Recommended Stack

| Layer | Recommendation |
|---|---|
| Frontend | React + TypeScript |
| State management | TanStack Query (server-state caching, stale polling) |
| Backend | Node.js (Fastify) or Python (FastAPI) |
| Database | PostgreSQL (FK constraints, JSONB audit, DECIMAL precision) |
| Background jobs | BullMQ (Node) or Celery (Python) |
| XLSX processing | SheetJS (JS) or openpyxl (Python) |
| Auth | JWT + refresh tokens |

### 9.2 Architectural Decisions

**AD-01: Equipment schema is fully dynamic.**
No `equipment_os`, `equipment_ps`, `equipment_video` tables exist. All equipment data lives in `object_devices` and `object_system_assignments`. Adding a new device type requires only inserting into `device_types` and `device_system_contexts` — zero migrations, zero code changes.

**AD-02: Normatives live on (device, system_type) context pairs, not on devices.**
`device_system_contexts` is the normatives store. R1/R2 values are meaningless without specifying both device and system context. There is no flat "normatives" table.

**AD-03: System type determines visit frequency, not device type.**
Visit frequency (R1/R2 visits per year) is read from `app_config` keyed by system type. Device properties have no influence on visit frequency.

**AD-04: Physical and maintained quantities are independent.**
`quantity_physical` tracks hardware assets. `quantity_maintained` drives calculations. The application must never auto-derive one from the other after import. They can and will legitimately differ.

**AD-05: Context deletion with active assignments is hard-blocked.**
`ON DELETE RESTRICT` FK from `object_system_assignments.context_id → device_system_contexts.id` prevents deletion at the database level. The API additionally returns a 409 with a list of affected objects before the user attempts deletion.

**AD-06: Repair types are a catalog, not hardcoded columns.**
`repair_types` + `object_repairs` replace the 25-column fixed repairs table. New repair operations are added via admin UI, no schema changes needed.

**AD-07: All calculations are server-side only.**
The frontend never computes workload values. It reads exclusively from `summaries`. This prevents calculation drift from UI bugs and ensures all users see consistent values.

**AD-08: Summaries use explicit staleness tracking.**
`is_stale` is set synchronously on write, cleared on successful recalculation. Background worker recalculates asynchronously. The UI reads `is_stale` and shows an indicator. This prevents serving stale data silently while maintaining write performance.

**AD-09: All calculation constants are read from `app_config` at compute time.**
The calculation service must reload config values for each recalculation batch, not cache them for the process lifetime. This ensures admin changes to constants take effect immediately in the next background job cycle.

**AD-10: Engineers are users, not a separate entity.**
An engineer is a `users` row with `role = 'engineer'`. There is no separate `engineers` table. `capacity_fte`, `home_division_id`, and `employee_id` are columns on `users`. This keeps authentication, role management, and engineer data in one place. The distinction between `division_id` (editor access scope) and `home_division_id` (engineer display home) must be explicitly maintained — they serve different purposes.

**AD-11: Engineer workload split ratio is always computed, never stored.**
`engineer_count` per object is a live COUNT query. Storing it would require updating it on every assignment change. Because the split is equal and the count is cheap to compute from `object_engineers`, it is calculated at summary computation time and not persisted.

**AD-12: Engineer summaries have a strict computation dependency on object summaries.**
The background worker must process stale object summaries before processing stale engineer summaries. An engineer summary computed from a stale object summary would silently propagate incorrect values. The worker queue must enforce this ordering — either via separate queues with priority, or by checking `summaries.is_stale = FALSE` for all assigned objects before computing an engineer summary.

**AD-13: Soft delete for engineers.**
Deactivated engineers (e.g., left the organisation) must not be hard-deleted if they have historical `object_engineers` assignments. Add `is_active BOOLEAN DEFAULT TRUE` to `users`. Inactive engineers are hidden from assignment dropdowns but their historical data is preserved. Their load still counts in division totals unless explicitly reassigned.

---

## 10. API Design

### 10.1 Base URL
```
/api/v1
```

### 10.2 Endpoints

#### Objects
```
GET    /objects                        List (paginated, filterable)
POST   /objects                        Create
GET    /objects/:id                    Get metadata
PUT    /objects/:id                    Update metadata
DELETE /objects/:id                    Delete
GET    /objects/:id/summary            Get computed summary
```

#### Physical Inventory
```
GET    /objects/:id/devices            List physical devices
POST   /objects/:id/devices            Add device {device_type_id, quantity_physical}
PUT    /objects/:id/devices/:dtid      Update physical quantity
DELETE /objects/:id/devices/:dtid      Remove (cascades system assignments)
```

#### System Assignments
```
GET    /objects/:id/assignments              List all assignments
POST   /objects/:id/assignments             Create {device_type_id, system_type, quantity_maintained}
PUT    /objects/:id/assignments/:aid         Update quantity_maintained
DELETE /objects/:id/assignments/:aid         Remove assignment
```

#### Records, Repairs, Travel
```
GET    /objects/:id/records            Get records task quantities
PUT    /objects/:id/records            Update (triggers recalc)
GET    /objects/:id/repairs            List repair counts per type
PUT    /objects/:id/repairs/:rtid      Set count for one repair type (triggers recalc)
GET    /objects/:id/travel             Get travel data
PUT    /objects/:id/travel             Update (triggers recalc)
```

#### Device Catalog
```
GET    /catalog/devices                List all device types
POST   /catalog/devices                Create device type
GET    /catalog/devices/:id            Get with all system contexts
PUT    /catalog/devices/:id            Update name/description
DELETE /catalog/devices/:id            Delete (blocked if object_devices rows exist)

GET    /catalog/devices/:id/contexts   List system contexts
POST   /catalog/devices/:id/contexts   Add context {system_type, r1_minutes, r2_minutes}
PUT    /catalog/devices/:id/contexts/:cid  Update r1/r2 (triggers bulk recalc for affected objects)
DELETE /catalog/devices/:id/contexts/:cid  Delete (blocked if active assignments; returns 409)
```

#### Repair Type Catalog
```
GET    /catalog/repairs                List all repair types
POST   /catalog/repairs                Create {name, time_minutes}
PUT    /catalog/repairs/:id            Update (triggers bulk recalc for affected objects)
DELETE /catalog/repairs/:id            Delete (blocked if active object_repairs; returns 409)
```

#### СВОД & Summary
```
GET    /svod                           All summaries (paginated, filterable)
POST   /svod/recalculate               Trigger full bulk recalculation (admin only)
GET    /svod/export/xlsx               Export to XLSX
GET    /svod/export/pdf                Export to PDF
```

#### App Configuration
```
GET    /admin/config                   List all config keys/values
PUT    /admin/config/:key              Update value (triggers bulk recalc; logged to audit_log)
GET    /admin/audit                    Audit log (admin only)
```

#### Engineers
```
GET    /engineers                         List all engineers (paginated, filterable)
POST   /engineers                         Create engineer (admin only)
GET    /engineers/:id                     Get engineer with summary
PUT    /engineers/:id                     Update name, capacity_fte, home_division (admin only)
DELETE /engineers/:id                     Deactivate (blocked if active assignments) (admin only)

GET    /engineers/:id/summary             Get engineer_summaries row
GET    /engineers/:id/objects             List all assigned objects with per-object shares
POST   /engineers/:id/objects             Assign object {object_id} to this engineer
DELETE /engineers/:id/objects/:oid        Remove object assignment
```

#### Object-Engineer Assignments (alternative entry point from object side)
```
GET    /objects/:id/engineers             List engineers assigned to this object
POST   /objects/:id/engineers             Assign engineer {engineer_id} to this object
DELETE /objects/:id/engineers/:eid        Remove engineer assignment
```

#### Coverage
```
GET    /coverage/gaps                     List all objects with zero assigned engineers
GET    /coverage/gaps?division_id=:did    Filtered by division
```

#### Import / Export / Auth
```
POST   /import/xlsx                    Upload XLSX; returns preview + validation report
POST   /import/xlsx/confirm            Execute confirmed import
POST   /auth/login
POST   /auth/logout
POST   /auth/refresh
GET    /auth/me
```

### 10.3 Response Format
```json
{ "data": { ... }, "meta": { "page": 1, "total": 2935, "per_page": 100 }, "error": null }
```
On error:
```json
{
  "data": null,
  "error": {
    "code": "CONTEXT_IN_USE",
    "message": "Cannot delete: 42 objects have active assignments using this context",
    "affected_count": 42
  }
}
```

---

## 11. Migrations & Data Import

### 11.1 Initial Data Import from XLSX

1. Parse `Нормативы` sheet → create `device_types` and `device_system_contexts` (seed data).
2. Parse repair normatives from `Нормативы` rows 23–24 → create `repair_types`.
3. Parse ОС, ПС, Видео, Записи, Ремонт, Дорога sheets. Match rows across sheets by `№` column.
4. For each row: create Division → Branch → Object (dedup divisions/branches by name).
5. For each non-zero equipment cell in ОС/ПС/Видео sheets:
   - Find `device_types` by column header name (create if not found, with warning).
   - Find `device_system_contexts` for (device, sheet's system type).
   - Create `object_devices` with `quantity_physical = cell_value`.
   - Create `object_system_assignments` with `quantity_maintained = cell_value`.
6. Populate `records_tasks` per object from Записи sheet.
7. Populate `object_repairs` per object from Ремонт sheet.
8. Populate `travel` per object from Дорога sheet.
9. Skip all "Расчет", `Лист1`, `Лист2` sheets.
10. Return validation report: objects created, warnings, skipped rows.
11. Queue full bulk recalculation.

> After import, `quantity_physical = quantity_maintained` for all records. Editors adjust `quantity_physical` manually if needed.

### 11.2 Import Validation Rules
- `№` must be numeric and unique.
- `Подразделение`, `Филиал`, `Значение` must not be empty.
- Equipment quantities must be non-negative or empty (treated as 0; zero values do not create assignment rows).
- If a column header doesn't match any `device_types` record: create a new device type, log a warning.
- If R1/R2 for a (device, system) pair already exists in DB and differs from XLSX: keep DB value, log a warning.

### 11.3 Database Migrations
Use a versioned migration tool (Flyway, Alembic, or knex migrate). All schema changes are versioned migration files. Never alter production schema manually.

---

## 12. Roles & Permissions

| Permission | Admin | Editor | Viewer | Engineer |
|---|---|---|---|---|
| View all objects / СВОД | ✅ | ✅ | ✅ | Own objects only |
| Edit object metadata | ✅ | ✅ (own div) | ❌ | ❌ |
| Edit equipment / assignments | ✅ | ✅ (own div) | ❌ | ❌ |
| Edit records / repairs / travel | ✅ | ✅ (own div) | ❌ | ❌ |
| Create / delete objects | ✅ | ❌ | ❌ | ❌ |
| Manage device catalog | ✅ | ❌ | ❌ | ❌ |
| Manage repair type catalog | ✅ | ❌ | ❌ | ❌ |
| Edit app configuration constants | ✅ | ❌ | ❌ | ❌ |
| Import XLSX | ✅ | ❌ | ❌ | ❌ |
| Export XLSX / PDF | ✅ | ✅ | ✅ | ✅ (own objects) |
| Trigger bulk recalculation | ✅ | ❌ | ❌ | ❌ |
| View audit log | ✅ | ❌ | ❌ | ❌ |
| Manage users / engineers | ✅ | ❌ | ❌ | ❌ |
| View own workload dashboard | ✅ | ✅ | ✅ | ✅ |
| Assign / remove engineers to objects | ✅ | ✅ (own div) | ❌ | ❌ |
| View engineer list and load ratios | ✅ | ✅ | ✅ | ✅ (own data only) |

**Editor scope:** `division_id` restricts all write operations to objects in their assigned division. Enforced at the API level.  
**Engineer scope:** Engineers can only view their own `engineer_summaries` and the objects they are assigned to. They cannot view other engineers' dashboards or unassigned objects.

---

## 13. Structural Clarifications & Corrections

### C-01: Duplicate Normative Values for "Записи"
`Нормативы` sheet has two sections for records tasks. Row 20 (60/180/20/3.15/60) is an older draft. **Use rows 73–77** as authoritative: 60/180/180/120/60 minutes.

### C-02: "Шлейфы сигнализации" R1 Discrepancy
Compact table (rows 3–4) shows R1=0.2 for ПС loops. Detailed table (rows 42, 65) shows R1=0.06 for both systems. **Use 0.06** as the seed value in `device_system_contexts`.

### C-03: "К-во ремонтов" Is Always Computed
Total repair count = `SUM(object_repairs.count)`. Never a user input. Read-only in UI.

### C-04: Round-Trip Travel Time Is Computed
`round_trip_min = one_way_time_min × 2`. Never stored. Never user-editable.

### C-05: "Расчет" Sheets Are Fully Derived
All "Расчет" sheets contain only references and formulas. They are not imported. Their logic is implemented in the calculation engine (§6).

### C-06: Р1/R1 Naming
Internal field names use `r1_minutes` / `r2_minutes` (Latin). UI labels display "Р1" / "Р2" (Cyrillic).

### C-07: Headcount Formula Constants Confirmed
Formula: `monthly_avg / 60 / 142.8 × 1.12`. Verified against ОС Расчет col AN. **142.8** = monthly working hours fund; **1.12** = absence coefficient. Both in `app_config`. No "7,647 min/month" divisor anywhere in the implementation.

### C-08: Empty Лист1 and Лист2
Not imported. Dashboard (FR-09) fulfills the same purpose as Лист1.

### C-09: Identical Division and Branch Names
Preserved as separate entities. No deduplication. UI shows both fields; they may be equal.

### C-10: Decimal Equipment Quantities
All equipment quantities use `DECIMAL(10,2)`. Decimal values (e.g., 2.2 шлейфа) are valid and accepted.

### C-11: Visit Frequencies Are Per-System Type
ОС: 10R1+2R2. ПС: 8R1+4R2. Видео: 10R1+2R2. Visit frequency is a property of the system type, not the device. Stored in `app_config`.

### C-12: Repair Monthly Divisor Is 5, Not 6
Repairs planned over 6 months but divided by **5** (productive months). Confirmed business rule. Do not change to 6 without explicit business owner approval.

### C-13: Repair PZV Is Separate from Visit PZV
PZV appears twice: once per maintenance visit (fixed cost in СВОД total), and once per repair operation (`repair_pzv_6months = total_repairs × PZV_MINUTES`). Both are additive and must not be collapsed.

### C-14: "Расчет" Sheets Define Structure, Not Storage
These sheets document the calculation pipeline. Developers must understand them but must not replicate their column structure as database tables.

### C-15: Same Device Name in Multiple Systems = One Device Type, Multiple Contexts
"серий А6, Аларм" in both ОС (R2=8) and ПС (R2=12): one `device_types` row, two `device_system_contexts` rows. The import must not create duplicate device_types for the same column header name.

### C-16: Physical vs. Maintained Quantity Divergence Is Expected
After import: `quantity_physical = quantity_maintained` for all records. Over time they may diverge (e.g., a new box installed but not yet assigned, or one box assigned to two systems). The application displays both values without forcing equality. A non-blocking warning shows when a single assignment's `quantity_maintained > quantity_physical`.

### C-17: Device Catalog System Restriction Is Explicit, Never Implicit
A newly created device type is assigned to **no systems** by default. The admin must explicitly create `device_system_contexts` rows to make a device available for assignment. There is no "inherit all systems" or "allow all" shortcut. This prevents accidental assignments using wrong or missing normatives.

### C-18: Orphaned Assignment Prevention
The system must prevent `object_system_assignments` rows from existing without a corresponding `object_devices` row at the same (object, device_type). This is enforced at the application service layer on every assignment create/update. It is not enforced by a direct FK because physical inventory and assignment are separate tables. The import process creates both rows together for every non-zero equipment cell.

### C-19: R1/R2 Are Never Overridden at the Assignment Level
`quantity_maintained` is the only user-editable field on `object_system_assignments`. There is no per-assignment override of R1/R2. Normatives are always read from `device_system_contexts` at calculation time. This ensures global consistency — changing a normative cascades to all objects using that context.

### C-21: Engineer Identity Is Unified with User Account
An engineer's login account and their professional data (capacity, home division) are the same record in `users`. There is no separate `engineers` table. Admins create an engineer by creating a user with `role = 'engineer'`. This means email, password, and engineer metadata are managed in one place.

### C-22: `responsible_engineer` VARCHAR Field Is Removed
The original Excel had a free-text "Ответственные ТО" column per object. In the web application this is replaced entirely by the `object_engineers` join table. The import process maps the text value to a matching `users.name` record (or creates a placeholder engineer account flagged for review). The VARCHAR field does not exist in the `objects` table.

### C-23: Equal Split Means Headcount Totals Are Consistent
With equal split, the sum of all engineers' `total_load` values equals the sum of all objects' `itogo_chislo_with_travel`. No workload is lost or double-counted at the division or system level.  
Proof: `SUM(object_share per engineer) = SUM(itogo / n × n) = SUM(itogo)`.

### C-24: Deactivated Engineers Retain Historical Data
Setting `users.is_active = FALSE` hides the engineer from dropdowns and the active list, but does not remove `object_engineers` rows or `engineer_summaries`. Division workload totals still reflect their assigned objects until an admin explicitly removes the assignments. This is a deliberate choice — unassigned workload must be visible, not silently dropped.

### C-25: Engineer Summary Depends on Object Summary Freshness
`engineer_summaries` must only be computed when all dependent `summaries` rows are fresh (`is_stale = FALSE`). If an object summary is stale, the engineer summary must also remain stale until the object is resolved. The background worker enforces this ordering. A partially-fresh engineer summary (some objects fresh, some stale) must not be written.

### C-20: New System Types Cannot Be Added Without Developer Involvement
The set of system types (ОС, ПС, Видео) is currently fixed. Visit frequency config keys, СВОД column structure, and calculation stages are all keyed to these three types. Adding a 4th system type (e.g., "СКУД" — access control) would require new `app_config` keys, a new СВОД column, and a new calculation stage. This is a developer task, not an admin task, and is out of scope for v1.

---

## 14. Acceptance Criteria

### AC-01: Data Completeness
Import of `Шаблон_нагрузки_з_v_4_00.xlsx` produces exactly 2,935 object records. All non-zero equipment values from ОС/ПС/Видео sheets are represented as `object_system_assignments` rows with corresponding `object_devices` rows.

### AC-02: Calculation Accuracy
All computed СВОД values match source XLSX "Расчет" sheet values within ±0.001. Verified fields: `os_monthly_avg`, `ps_monthly_avg`, `video_monthly_avg`, `records_monthly`, `repair_no_travel_monthly`, `repair_with_travel_monthly`, `total_no_travel_min`, `total_with_travel_min`, `itogo_chislo_no_travel`, `itogo_chislo_with_travel`, `r1_per_visit_total`, `r2_per_visit_total`.

### AC-03: Dynamic Normative Editability
After an admin updates `r1_minutes` or `r2_minutes` on any `device_system_contexts` row, all affected object summaries are marked stale and recalculated by the background job — without code deployment. UI shows stale indicator during recalculation.

### AC-04: New Device Type Usable Without Code Changes
Admin creates a device type, adds a system context (R1/R2), assigns it to an object, sets `quantity_maintained`. СВОД recalculates correctly. No code deployment required.

### AC-05: System Context Restriction Enforced
UI "Assign to system" dropdown shows only system types with a valid `device_system_contexts` row for the device. API rejects `POST /objects/:id/assignments` where (device_type_id, system_type) has no context row — returns HTTP 422 with code `NO_CONTEXT_FOR_SYSTEM`.

### AC-06: Context Deletion Blocked When In Use
`DELETE /catalog/devices/:id/contexts/:cid` returns HTTP 409 listing affected objects when any `object_system_assignments` references that context. Database FK `ON DELETE RESTRICT` also prevents bypass via direct SQL.

### AC-07: Computed Fields Are Read-Only
API ignores or rejects attempts to set `total_repairs`, `round_trip_min`, `is_stale`, or any `summaries` field directly. Returns HTTP 422 if attempted.

### AC-08: Role Enforcement
Editor assigned to Division A cannot read or write objects in Division B. `PUT /objects/:id` for a Division B object returns HTTP 403.

### AC-09: Export Fidelity
XLSX export of СВОД matches original template column structure and values within ±0.001.

### AC-10: Performance
СВОД page (first 100 rows, 2,935 objects imported) loads in under 3 seconds. Bulk recalculation of all objects completes in under 60 seconds.

### AC-11: Workflow Enforcement — Assign Before Physical Inventory Is Blocked
Attempting to `POST /objects/:id/assignments` for a device_type_id that has no corresponding `object_devices` row at that object returns HTTP 422 with code `DEVICE_NOT_IN_INVENTORY`.

### AC-12: Shared Device Multi-System Calculation Is Correct
An object with one Galaxy 512 (контроллер АСПС и СО) assigned to both ОС and ПС with `quantity_maintained = 1` each produces:
- `os_r1_per_visit` containing the 15 min contribution from this device
- `ps_r1_per_visit` containing the 15 min contribution from the same device
- Physical inventory shows 1 unit
- СВОД `Охрана` and `Пожарная сигнализация` columns both reflect this device's contribution independently

### AC-13: System Context Restriction UI Test
In the Equipment tab, for a device that has `device_system_contexts` only for ОС:
- "Assign to system" dropdown shows only "ОС"
- ПС and Видео are not present in the dropdown (not hidden/disabled — absent)
- Attempting `POST /objects/:id/assignments` with `system_type: "ПС"` returns HTTP 422 `NO_CONTEXT_FOR_SYSTEM`

### AC-14: Engineer Workload Equals Sum of Object Shares
For any engineer E assigned to objects O1…On, `engineer_summaries.total_load` must equal `SUM(summaries[Oi].itogo_chislo_with_travel / COUNT(engineers at Oi))` within ±0.000001.

### AC-15: Equal Split Consistency
The sum of `total_load` across all engineers assigned to a given object must equal that object's `itogo_chislo_with_travel` within ±0.000001.

### AC-16: Capacity and Status Logic
Given engineer with `capacity_fte = 0.8` and `total_load = 0.76`:
- `load_ratio = 0.76 / 0.8 = 0.95`
- With `ENGINEER_WARNING_THRESHOLD = 0.9`: status must be "warning"
Given `total_load = 0.84`: `load_ratio = 1.05` → status must be "overloaded"

### AC-17: Assignment Change Triggers All Co-Engineer Recalculation
When a third engineer is added to an object that previously had two, all three engineers' `engineer_summaries.is_stale` must be set to TRUE in the same transaction. After background recalculation, each engineer's share of that object must equal `itogo_chislo_with_travel / 3`.

### AC-18: Coverage Gap Reporting
`GET /coverage/gaps?division_id=X` returns all objects in division X that have zero rows in `object_engineers`. Verified against manual count from the object list.

---

*End of Technical Specification — Version 2.2*
