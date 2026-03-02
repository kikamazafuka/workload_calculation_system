# Technical Specification (TOR)
# Web Application: Security Systems Maintenance Workload Calculator
**Version:** 2.7  
**Based on:** Шаблон_нагрузки_з_v_4_00.xlsx  
**Date:** 2026-02-23  
**Changelog v2.0:** Replaced hardcoded equipment tables with dynamic device catalog architecture (§4.2, §4.3, §4.5, §5, §6.3, §6.4, §7, §9, §10, §13).  
**Changelog v2.1:** Incorporated three architectural decisions: (1) Option C two-layer quantity model (physical + maintained); (2) System type restriction enforced at UI/API/DB levels; (3) Normatives managed per (device, system) pair. Updated §4.2, §4.3, §6.4, §7.3, §7.4, added C-17–C-20, added AC-11–AC-13.  
**Changelog v2.2:** Added Engineers Module — engineers as first-class entities, object-engineer assignments with equal workload split, engineer capacity tracking, overload detection, engineer dashboard, and coverage gap reporting. Updated §2, §3, §4 (FR-10, FR-11), §5, §6 (§6.12–6.14), §7, §9, §10, §12, §13 (C-21–C-25), §14 (AC-14–AC-18).  
**Changelog v2.3:** Resolved 22 open questions. Key changes: (1) R2-includes-R1 clarification; (2) planning periods; (3) on-demand recalculation; (4) home division = travel reference; (5) placeholder accounts on import; (6) engineer data entry rights; (7) hard delete for PoC; (8) СВОД export scope. Updated §4 (FR-12), §5, §7, §9, §10, §11, §12, §13 (C-26–C-32), §14 (AC-19–AC-21).  
**Changelog v2.4:** Resolved 22 open questions. Key changes: (1) R2-includes-R1 clarification — Excel uses additive model, TOR calculation confirmed correct; (2) planning periods as tracked entities for repairs and records; (3) recalculation is on-demand only; (4) home division = travel reference point for engineer; (5) import creates placeholder accounts; (6) engineers can edit own objects; (7) object deletion is soft-archive for PoC; (8) СВОД export scope clarified. Updated §4 (FR-12 Periods), §5, §7, §9, §10, §11, §12, §13 (C-26–C-32), §14 (AC-19–AC-21).  
**Changelog v2.5:** Technology stack finalised. Updated §9.1 (Java/Spring Boot), §15.8 (PoC stack), §18 (Datadog/log4j2/Actuator). Added §19 Testing Strategy, §20 CI/CD Pipeline. Introduced Redis scope (AD-17), DTO layer constraint (AD-18), rate limiting (§8.3), Liquibase migration structure (§11.3), Jacoco coverage thresholds.  
**Changelog v2.6:** Corrected repair calculation engine. К-во ремонтов = COUNT of distinct repair types with non-zero counts (not SUM of quantities). repair_travel and repair_pzv use 3-tier threshold formula (≤5→0, ≤10→kvo×rate, >10→10×rate). Verified against all non-zero repair rows in source XLSX: zero mismatches. Updated §4.5, §6.6, §6.11 (new config keys), §13 (C-36, C-37), §14 (AC-22).  
**Changelog v2.7:** Dependency sweep after v2.6 repair formula corrections. Fixed 8 locations: (1) §5 summaries column comments; (2) §6.1 pipeline Stage 5 expanded with kvo and effective_trips; (3) §6.10 — DELETE added to object_repairs invalidation trigger; (4) C-03 rewritten — was wrong SUM definition, now correct COUNT definition with cross-ref to C-36; (5) C-13 rewritten — old flat formula replaced with threshold-aware description; (6) AC-07 — total_repairs definition clarified; (7) AC-22 moved from orphaned position after §20 into §14 with 3-band structure; (8) §19.2 — RepairCalculationTest added with 8 boundary cases covering all threshold bands.

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
15. [PoC Scope](#15-poc-scope)
16. [Aggregation Rules](#16-aggregation-rules)
17. [Concurrency & Locking](#17-concurrency--locking)
18. [Observability & Monitoring](#18-observability--monitoring)
19. [Testing Strategy](#19-testing-strategy)
20. [CI/CD Pipeline](#20-cicd-pipeline)

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
| Период (Planning period) | A 6-month window to which repair counts and records tasks belong (e.g. H1 2025, H2 2025) |

---

## 4. Functional Requirements

### 4.1 Object Management (FR-01)
- CRUD operations for objects (facilities).
- Each object must have: `division`, `branch`, `name/address`.
- Responsible engineers are managed via the `object_engineers` join table (§4.10), not as a free-text field on the object.
- Objects are organized in a two-level hierarchy: **Division → Branch → Objects**.
- Support bulk import from XLSX (see §11).
- Deleted objects are **permanently removed** (hard delete) for PoC. Soft-archive / decommission status is a post-MVP consideration (see §13 C-30).

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

Records tasks are **irregular events** — they do not occur on a fixed schedule. Quantities are entered for a specific 6-month planning period (see FR-12). The monthly average `records_monthly = records_6months / 6` represents a smoothed load estimate, not a guaranteed monthly occurrence.

### 4.5 Repair Type Catalog & Object Repairs (FR-05) — "Ремонт"

Repair types are also admin-managed catalog entries (same extensibility principle as devices). Each repair type has a fixed time in minutes per operation. Admins can add new repair types without schema changes.

Count of repair operations per object per 6-month planning period is recorded per repair type.

**К-во ремонтов** (total repair count) = `COUNT(object_repairs rows WHERE count > 0 AND period_id = current_period)` for the object — always computed, never a user input. It counts **distinct repair types that were performed at least once**, not the sum of quantities.

> **Example:** An object with "Замена аккумулятора ОС × 3" and "Замена извещателя × 5" has К-во ремонтов = **2** (two distinct repair types performed), not 8 (sum of quantities). This is the value used in the travel/PZV threshold formula.

Repair counts belong to a specific 6-month planning period (see FR-12). The system tracks which period each set of repair counts belongs to, enabling comparison across periods.

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

### 4.12 Planning Periods (FR-12)

Repair counts and records tasks are tied to a specific **6-month planning period**. A period covers either H1 (January–June) or H2 (July–December) of a calendar year.

**Period entity fields:**
- `name` — display name, e.g. "H1 2025", "H2 2025"
- `start_date` / `end_date` — ISO dates
- `is_active` — boolean; exactly one period is active at any time (the "current" period)
- `created_at`

**Rules:**
- Only one period can be `is_active = TRUE` at a time. Setting a new period active automatically deactivates the previous one.
- All data entry for repairs and records (via the UI or API) defaults to the currently active period.
- Admins can create future periods in advance and switch the active period when ready.
- Past periods are read-only — their repair counts and records data cannot be edited once the period is no longer active.
- СВОД calculations use the **active period's** data by default. Users can select any past period for historical comparison.
- When no period is active (e.g. gap between periods), data entry for repairs and records is blocked with a warning.

**Data model impact:** `object_repairs` and `records_tasks` both gain a `period_id` FK → `periods.id`. See §5.

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
object_id           UUID          FK → objects.id NOT NULL
period_id           UUID          FK → periods.id NOT NULL
access_requests     DECIMAL(10,2) NOT NULL DEFAULT 0
monitoring_requests DECIMAL(10,2) NOT NULL DEFAULT 0
footage_requests    DECIMAL(10,2) NOT NULL DEFAULT 0
backup_control      DECIMAL(10,2) NOT NULL DEFAULT 0
security_admin      DECIMAL(10,2) NOT NULL DEFAULT 0
updated_at          TIMESTAMP
UNIQUE(object_id, period_id)
```

One row per (object, period) pair. Past-period rows are read-only once the period is deactivated.

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
period_id      UUID    FK → periods.id NOT NULL
count          INTEGER NOT NULL DEFAULT 0
updated_at     TIMESTAMP
UNIQUE(object_id, repair_type_id, period_id)
```

One row per (object, repair_type, period). Past-period rows are read-only once the period is deactivated.

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

#### `periods` — Planning Periods
```sql
id          UUID         PK
name        VARCHAR(50)  NOT NULL UNIQUE   -- e.g. "H1 2025"
start_date  DATE         NOT NULL
end_date    DATE         NOT NULL
is_active   BOOLEAN      NOT NULL DEFAULT FALSE
created_at  TIMESTAMP
updated_at  TIMESTAMP
CHECK (end_date > start_date)
```

Only one row may have `is_active = TRUE` at a time. Enforced by a partial unique index:
```sql
CREATE UNIQUE INDEX one_active_period ON periods (is_active) WHERE is_active = TRUE;
```

---

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

> Summaries are computed from the **active period's** records and repairs data by default. The `period_id` used for the latest computation is stored for traceability.

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
total_repairs              INTEGER         -- kvo = COUNT(repair_types WHERE count > 0 in period)
repair_work_6months        DECIMAL(10,4)   -- SUM(count × time_minutes) for performed types
repair_travel_6months      DECIMAL(10,4)   -- effective_trips × round_trip_min  (threshold formula §6.6)
repair_pzv_6months         DECIMAL(10,4)   -- effective_trips × PZV_MINUTES     (threshold formula §6.6)
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

period_id                  UUID            FK → periods.id  -- period used for this computation
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
  records_monthly = records_6months / 6  (§6.5)

  For repairs (§6.6):
    repair_work_6months = SUM(count × time_minutes)
    kvo = COUNT(repair types with count > 0)          ← distinct type count, NOT sum of quantities
    effective_trips = threshold(kvo, ZERO_THRESHOLD, CAP)  ← 0 | kvo | CAP
    repair_travel_6months = effective_trips × round_trip_min
    repair_pzv_6months    = effective_trips × PZV_MINUTES

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

#### Step 1 — Work time

```
repair_work_6months = SUM(object_repairs.count × repair_types.time_minutes)
                      WHERE object_repairs.count > 0
                      AND   object_repairs.period_id = current_period
```

#### Step 2 — К-во ремонтов (kvo)

```
kvo = COUNT(object_repairs rows WHERE count > 0 AND period_id = current_period)
```

`kvo` counts **distinct repair types performed at least once** in the period — not the sum of quantities. This is the value tested against thresholds in Steps 3 and 4.

> **Example:** "Замена аккумулятора × 3" and "Замена извещателя × 5" → `kvo = 2`, not 8.

#### Step 3 — Threshold for effective trip count

```
effective_trips =
    kvo <= config[REPAIR_TRAVEL_ZERO_THRESHOLD]:  0
    kvo <= config[REPAIR_TRAVEL_CAP]:             kvo
    kvo >  config[REPAIR_TRAVEL_CAP]:             config[REPAIR_TRAVEL_CAP]
```

New config constants:

| Key | Default | Description |
|---|---|---|
| `REPAIR_TRAVEL_ZERO_THRESHOLD` | 5 | kvo ≤ this value → zero travel and PZV overhead |
| `REPAIR_TRAVEL_CAP` | 10 | kvo above this → cap effective_trips at this value |

#### Step 4 — Travel and PZV overhead

```
round_trip_min        = travel.one_way_time_min × 2

repair_travel_6months = effective_trips × round_trip_min
repair_pzv_6months    = effective_trips × config[PZV_MINUTES]
```

Both use `effective_trips` — not raw `kvo`, not `SUM(count)`.

#### Step 5 — Monthly averages

```
repair_no_travel_monthly   = repair_work_6months
                             / config[REPAIR_PRODUCTIVE_MONTHS]

repair_with_travel_monthly = (repair_work_6months
                               + repair_travel_6months
                               + repair_pzv_6months)
                             / config[REPAIR_PRODUCTIVE_MONTHS]
```

> Divisor is `REPAIR_PRODUCTIVE_MONTHS = 5`. See §13 C-12.

**Verified — Object "Архив г.Брест" (kvo = 8, within 5–10 band):**
```
Repair types with count > 0:
  Замена извещателя пожарного дымового      × 1  →  12 min
  Замена шунт/оконечного резистора ОС       × 3  → 105 min
  Замена шунт/оконечного резистора ПС       × 3  → 105 min
  Замена аккумулятора ОС                    × 1  →   5 min
  Акт о выполненных работах ОС              × 1  →   7 min
  Дефектный акт ОС                          × 1  →  60 min
  Акт о выполненных работах ПС              × 1  →   7 min
  Дефектный акт ПС                          × 1  →  60 min

repair_work_6months = 361 min
kvo = 8  (8 distinct types with count > 0)
round_trip_min = 20

5 < kvo=8 ≤ 10  →  effective_trips = 8
repair_travel_6months = 8 × 20 = 160  ✓
repair_pzv_6months    = 8 × 20 = 160  ✓

repair_no_travel_monthly   = 361 / 5 = 72.2  ✓
repair_with_travel_monthly = (361 + 160 + 160) / 5 = 136.2  ✓
```

**Verified — High repair count (kvo = 17, cap applies):**
```
kvo = 17 > 10  →  effective_trips = 10
round_trip_min = 20
repair_travel_6months = 10 × 20 = 200  ✓
repair_pzv_6months    = 10 × 20 = 200  ✓
```

**Verified — Low repair count (kvo = 1, zero threshold applies):**
```
kvo = 1 ≤ 5  →  effective_trips = 0
repair_travel_6months = 0  ✓
repair_pzv_6months    = 0  ✓
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

Summaries are marked stale automatically on data change, but **recalculation is triggered on-demand by admins only** — not automatically. The background worker runs only when explicitly triggered via `POST /svod/recalculate` (see §10). This simplifies operations and gives admins control over when calculations are refreshed (e.g. after a bulk data entry session).

| Triggering Event | Staleness Action (immediate, same transaction) |
|---|---|
| `object_system_assignments` INSERT / UPDATE / DELETE | Mark this object's `summaries.is_stale = TRUE` |
| `object_devices` INSERT / UPDATE / DELETE | Mark this object's `summaries.is_stale = TRUE` |
| `records_tasks` INSERT / UPDATE | Mark this object's `summaries.is_stale = TRUE` |
| `object_repairs` INSERT / UPDATE / DELETE | Mark this object's `summaries.is_stale = TRUE` (any change to repair counts or the set of performed types changes kvo and repair_work_6months) |
| `travel` UPDATE | Mark this object's `summaries.is_stale = TRUE` |
| `device_system_contexts` UPDATE (r1 or r2) | Mark `is_stale = TRUE` for ALL objects with assignments using this context |
| `repair_types.time_minutes` UPDATE | Mark `is_stale = TRUE` for ALL objects with this repair type |
| `app_config` UPDATE (any calculation key) | Mark ALL `summaries.is_stale = TRUE` |
| `periods.is_active` changed (period switch) | Mark ALL `summaries.is_stale = TRUE` |
| Any `summaries.is_stale` set TRUE | Mark all engineers assigned to that object: `engineer_summaries.is_stale = TRUE` |

**Recalculation trigger:** Admin clicks "Пересчитать" in the UI or calls `POST /svod/recalculate`. The background worker then processes all stale summaries in dependency order: object summaries first, then engineer summaries.

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
| `REPAIR_TRAVEL_ZERO_THRESHOLD` | 5 | kvo ≤ this → zero travel and PZV overhead for repairs |
| `REPAIR_TRAVEL_CAP` | 10 | kvo above this → cap effective_trips at this value |
| `ENGINEER_WARNING_THRESHOLD` | 0.9 | Load ratio at which engineer status becomes "warning" |

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

Staleness is set in the same transaction as the triggering change. Actual recalculation is on-demand (triggered by admin). Background worker processes engineer summaries only after all dependent object summaries are fresh.


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
| `/svod` | СВОД | Full summary table with period selector, filters, and export |
| `/catalog/devices` | Device Catalog | List / create / edit device types and contexts |
| `/catalog/devices/new` | New Device | Create device type and assign system contexts |
| `/catalog/devices/:id` | Device Detail | View/edit device and all system contexts |
| `/catalog/repairs` | Repair Types | List / create / edit repair type catalog |
| `/admin/config` | App Config | View/edit all calculation constants (admin only) |
| `/admin/periods` | Planning Periods | Create / activate / deactivate planning periods (admin only) |
| `/admin/users` | Users | User management (admin only) |
| `/import` | Import | XLSX import wizard |
| `/export` | Export | Export options |
| `/engineers` | Engineer List | All engineers with load ratio and status |
| `/engineers/:id` | Engineer Detail | Workload dashboard for one engineer |
| `/engineers/:id/edit` | Engineer Edit | Edit name, capacity, home division |

### 7.2 Object Detail Page — Tabs

1. **Оборудование** — Two-layer equipment view (physical inventory + system assignments)
2. **Записи** — Records/admin task counts *(scoped to active period)*
3. **Ремонт** — Repair operation counts *(scoped to active period)*
4. **Дорога** — Travel data
5. **Инженеры** — Assigned engineers list with their share of this object's workload
6. **СВОД** — Computed summary (read-only; shows stale indicator when `is_stale = TRUE`; period shown in header)

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
- **Travel review prompt:** When a new engineer is assigned to an object (or the last engineer is removed and a new one added), the UI displays a non-blocking banner: **"Проверьте данные о маршруте — время в пути может отличаться для нового инженера"**. Travel time is stored per object and reflects the distance from the assigned engineer's home division office. When the responsible engineer changes, travel data should be reviewed and updated manually.

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

Stale rows (`is_stale = TRUE`) display a "Данные устарели — нажмите Пересчитать" indicator in place of numeric values.

**Period selector:** A dropdown above the СВОД table allows selecting which planning period's data to view. Defaults to the active period. When a non-active period is selected, the table shows historical data for that period (read-only). The "Пересчитать" button is disabled for past periods.

### 7.10 UI/UX Constraints
- All user-facing labels in **Russian**.
- СВОД table sortable by any column, filterable by division / branch / engineer.
- Repair tab is a dynamic list from `repair_types` — never hardcoded input fields.
- Zero values may display as blank (matching Excel behavior) but stored as 0.
- Inline СВОД editing not permitted.
- Engineer load ratio bars use colour coding matching status: green (normal) / amber (warning) / red (overloaded).
- Stale engineer summaries display "Пересчитывается..." placeholder values, never the last stale numbers.
- **Data entry by engineers:** Engineers can enter and edit Записи and Ремонт data for their assigned objects in the active period only. They cannot edit Оборудование, Нормативы, or Дорога. This matches their role as field operators who report what happened (repairs done, records requests handled) without modifying the equipment inventory or normatives.
- **Period lock:** Once a period is deactivated, all its Записи and Ремонт data becomes read-only for all roles including admin. Only a new period activation can unlock data entry.

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
- Authentication required for all routes except `/login` and `/actuator/health`.
- HTTPS (TLS) only in production, enforced at Nginx reverse proxy level.
- JWT-based stateless authentication; tokens validated on every request by Spring Security filter chain.
- CSRF protection: not required for stateless JWT APIs (no session cookies). Ensure `SameSite=Strict` on any cookies used.
- Input validation via Spring Validation (Jakarta Bean Validation) on all DTOs; numeric fields validated for range and non-null.
- All changes to `device_system_contexts`, `device_types`, `repair_types`, `app_config`, `object_engineers`, and `users.capacity_fte` written to `audit_log`.
- **Rate limiting:** Bucket4j applied at the Spring filter level. Default limits: 100 requests/minute per IP for unauthenticated endpoints; 300 requests/minute per user-id for authenticated endpoints. Admin-configurable via environment variables. Exceeding limit returns HTTP 429 with `Retry-After` header.
- **Sensitive endpoint protection:** `/actuator/*` endpoints (except `/actuator/health`) are accessible only from internal network (Nginx blocks external access to `/actuator`).

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

### 9.1 Technology Stack

#### Backend

| Component | Choice | Notes |
|---|---|---|
| Runtime | Java 21 (LTS) | Long-term support; virtual threads available for async if needed |
| Framework | Spring Boot 3.x | Auto-configuration, production-ready defaults |
| REST | Spring Web (MVC) | Standard REST controllers |
| Persistence | Spring Data JPA (Hibernate) | ORM over PostgreSQL; DECIMAL precision preserved via `BigDecimal` |
| Security | Spring Security + JWT | Stateless JWT auth; refresh tokens in MVP |
| Validation | Spring Validation (Jakarta) | Bean validation on DTOs; Zod mirrors on frontend |
| Metrics | Spring Boot Actuator | Exposes `/actuator/health`, `/actuator/metrics`, `/actuator/prometheus` |
| Build | Maven | Dependency management; multi-module layout for PoC→MVP |
| DTO mapping | MapStruct | Compile-time DTO↔entity mapping; no reflection overhead |
| Boilerplate | Lombok | `@Data`, `@Builder`, `@RequiredArgsConstructor` — optional per class |
| Logging | log4j2 | JSON layout for structured logging (see §18); SLF4J as facade |
| XLSX processing | Apache POI | Industry-standard Java XLSX read/write; replaces openpyxl |

#### Database & Migrations

| Component | Choice | Notes |
|---|---|---|
| Database | PostgreSQL | FK constraints, `DECIMAL` precision, `JSONB` audit column |
| Migrations | Liquibase | Versioned changelogs in `src/main/resources/db/changelog`; runs on startup |

#### Caching & Async Processing

| Component | Choice | Notes |
|---|---|---|
| Cache + Queue | Redis | Dual use: (1) background job queue for recalculation; (2) optional response cache. See AD-17 for scope. |
| Job integration | Spring Data Redis / Redisson | Redis-backed queue; Spring `@Async` or Redisson `RQueue` for job dispatch |

#### Frontend

| Component | Choice | Notes |
|---|---|---|
| Framework | React 18 + TypeScript | Strict mode; functional components only |
| Build | Vite | Fast HMR; production build with tree-shaking |
| Routing | React Router v6 | File-based route structure |
| Server state | TanStack Query | Caching, background refetch, stale-while-revalidate |
| Client state | Zustand (preferred) or Redux Toolkit | Zustand for PoC (lighter); Redux Toolkit if state complexity grows in MVP |
| UI components | Material UI (MUI) v5+ | Consistent design system; MUI DataGrid for СВОД table |
| HTTP | Axios | Interceptors for JWT injection and 401 handling |
| Forms | React Hook Form + Zod | Client-side schema validation mirrors server-side Bean Validation |

#### Testing

| Tool | Scope |
|---|---|
| JUnit 5 | Unit tests — services, calculation engine, mappers |
| Mockito | Mock dependencies in unit tests |
| Testcontainers (PostgreSQL) | Integration tests — repository layer against real DB |
| RestAssured | API integration tests — full HTTP round-trip |
| WireMock | External API mocking (future integrations) |
| Jacoco | Code coverage reports; minimum threshold defined in §19 |

#### Infrastructure

| Component | Choice | Notes |
|---|---|---|
| Containerisation | Docker | One image per service |
| Local orchestration | Docker Compose | 5 containers: backend, frontend, PostgreSQL, Redis, Nginx |
| Reverse proxy | Nginx | TLS termination, static frontend serving, API proxy |
| CI/CD | GitHub Actions | See §20 |
| APM / Monitoring | Datadog | See §18 |
| Rate limiting | Bucket4j | Token-bucket algorithm; applied per-IP and per-user-id at API gateway layer |

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
The calculation service must reload config values for each recalculation batch, not cache them for the process lifetime. This ensures admin changes to constants take effect immediately in the next triggered recalculation.

**AD-14: Recalculation is on-demand, not event-driven.**
The system marks summaries stale automatically on data change, but does not auto-trigger recalculation. Recalculation runs only when an admin explicitly triggers it. This is the correct model for PoC — it avoids cascading background load during bulk data entry and gives operators control over when they see updated numbers. Post-MVP may move to automatic triggers.

**AD-15: Period is the unit of data versioning for operational data.**
`object_repairs` and `records_tasks` are period-scoped. All other data (equipment, travel, assignments) is current-state only — no period versioning. This means СВОД can be computed for any historical period by using that period's repair/records rows with current equipment and normatives. Historical equipment states are not tracked in PoC.

**AD-16: Travel time is per-object, set by whoever enters data.**
Travel data reflects the distance from the responsible engineer's home division base to the object. It is a manually entered field — the system does not compute it. When engineer assignment changes, the travel field must be manually reviewed. The home division of the engineer is the reference, but no automatic distance calculation is implemented.

**AD-17: Redis scope is job queue first, cache second.**
Redis serves two purposes. Primary: the recalculation job queue — stale summary IDs are pushed to a Redis list; the background worker pops and processes them. Secondary (MVP only): response caching for aggregation endpoints (§16.7) if query latency becomes measurable under load. For PoC, Redis is used for the job queue only. Aggregation queries run live against PostgreSQL (on-the-fly per §16 decision). No business data is stored in Redis — it is a transient layer only. If Redis is unavailable, the system degrades gracefully: recalculation is blocked but all read/write operations against PostgreSQL continue.

**AD-18: All API responses go through DTOs — never raw JPA entities.**
MapStruct compile-time mappers translate between JPA entities and DTOs for all request/response cycles. JPA entities are never serialised directly to JSON. This prevents accidental exposure of lazy-loaded relations, internal fields (`password_hash`, `is_stale` internals), and database-level annotations leaking into the API contract. The DTO layer is the API contract. Changing internal entity structure does not break the API as long as mappers are updated.

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
GET    /svod/export/xlsx               Export to XLSX (active period by default; ?period_id= for historical)
GET    /svod/export/pdf                Export to PDF (active period by default; ?period_id= for historical)
```

#### App Configuration
```
GET    /admin/config                   List all config keys/values
PUT    /admin/config/:key              Update value (marks all summaries stale; logged to audit_log)
GET    /admin/audit                    Audit log (admin only)
```

#### Planning Periods
```
GET    /admin/periods                  List all periods
POST   /admin/periods                  Create period {name, start_date, end_date}
GET    /admin/periods/:id              Get period details
PUT    /admin/periods/:id/activate     Set as active period (deactivates current active)
GET    /admin/periods/active           Get the currently active period
```

#### Recalculation (on-demand)
```
POST   /svod/recalculate               Trigger full recalculation of all stale summaries (admin only)
POST   /svod/recalculate/:object_id    Trigger recalculation for one object (admin only)
GET    /svod/recalculate/status        Check background job status {total_stale, processed, remaining}
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

#### Aggregations (on-the-fly, no cache)
```
GET    /aggregations/company              Company-wide required FTE and breakdown
GET    /aggregations/divisions            All divisions summary list
GET    /aggregations/divisions/:id        Single division detail (§16.7 response shape)
GET    /aggregations/branches             All branches summary list
GET    /aggregations/branches/:id         Single branch detail
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

> **Engineer import:** The `Ответственные ТО` column contains plain name strings (e.g. "Александр Н Соловей"). The import attempts to match each name to an existing `users` record by `users.name` (case-insensitive). If a match is found, the engineer is assigned to the object. If no match is found, a **placeholder engineer account** is created with `role = 'engineer'`, `is_active = FALSE`, and a flag `requires_activation = TRUE`. A post-import report lists all placeholder accounts created, prompting admins to set passwords and activate them. Objects with unresolved engineers are not left unassigned — the placeholder account serves as the assignment holder until activated.

### 11.2 Import Validation Rules
- `№` must be numeric and unique.
- `Подразделение`, `Филиал`, `Значение` must not be empty.
- Equipment quantities must be non-negative or empty (treated as 0; zero values do not create assignment rows).
- If a column header doesn't match any `device_types` record: create a new device type, log a warning.
- If R1/R2 for a (device, system) pair already exists in DB and differs from XLSX: keep DB value, log a warning.
- Engineer name matching is case-insensitive and trims whitespace. Partial matches (e.g. "А. Соловей" vs "Александр Соловей") are not attempted — only exact full-name matches. Non-matching names create placeholder accounts.
- Import assigns all repairs and records to the **active period** at the time of import. If no period is active, import is blocked until an admin activates a period.

### 11.3 Database Migrations
Use **Liquibase** for all schema versioning. Changelogs live in `src/main/resources/db/changelog/`. Structure:

```
db/changelog/
  db.changelog-master.xml        ← root changelog, includes all others
  changes/
    v1.0.0-initial-schema.xml    ← PoC schema (divisions, objects, summaries, …)
    v1.1.0-periods.xml           ← MVP: planning periods
    v1.2.0-rbac.xml              ← MVP: roles, division scoping
    v1.3.0-physical-inventory.xml← MVP: object_devices table
    …
```

Liquibase runs automatically on application startup via Spring Boot auto-configuration. **Never alter production schema manually.** All changes go through versioned changesets with `author`, `id`, and rollback instructions where applicable.

Seed data (device types, system contexts, repair types) is loaded as a Liquibase changeset — not application-level code — so it runs exactly once and is versioned alongside schema changes.

---

## 12. Roles & Permissions

| Permission | Admin | Editor | Viewer | Engineer |
|---|---|---|---|---|
| View all objects / СВОД | ✅ | ✅ | ✅ | Own objects only |
| Edit object metadata | ✅ | ✅ (own div) | ❌ | ❌ |
| Edit equipment / assignments | ✅ | ✅ (own div) | ❌ | ❌ |
| Edit records / repairs (active period) | ✅ | ✅ (own div) | ❌ | ✅ (own objects) |
| Edit travel data | ✅ | ✅ (own div) | ❌ | ❌ |
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
**Period lock:** Записи and Ремонт data is read-only for all roles once a period is deactivated. Only admin can create and activate a new period to enable data entry again.

---

## 13. Structural Clarifications & Corrections

### C-01: Duplicate Normative Values for "Записи"
`Нормативы` sheet has two sections for records tasks. Row 20 (60/180/20/3.15/60) is an older draft. **Use rows 73–77** as authoritative: 60/180/180/120/60 minutes.

### C-02: "Шлейфы сигнализации" R1 Discrepancy
Compact table (rows 3–4) shows R1=0.2 for ПС loops. Detailed table (rows 42, 65) shows R1=0.06 for both systems. **Use 0.06** as the seed value in `device_system_contexts`.

### C-03: "К-во ремонтов" Is Always Computed — Counts Distinct Types, Not Sum of Quantities
`К-во ремонтов` = `COUNT(object_repairs rows WHERE count > 0 AND period_id = current_period)`. It counts how many distinct repair types were performed at least once in the period — not the total number of individual repair operations and not the sum of `count` values. An object with "Замена аккумулятора × 3" and "Замена извещателя × 5" has `К-во ремонтов = 2`, not 8. This value is referred to as `kvo` in the calculation engine (§6.6) and is used as the threshold input for `effective_trips`. It is always computed server-side; never a user input; read-only in all UI views. See also C-36.

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

### C-13: Repair PZV Is Separate from Visit PZV, and Uses the Threshold Formula
PZV appears in two places in the calculation:
1. **Per maintenance visit** — `pzv = config[PZV_MINUTES]` (20 min), a fixed cost added once in the СВОД total regardless of equipment count.
2. **Per repair trip** — `repair_pzv_6months = effective_trips × config[PZV_MINUTES]`, where `effective_trips` is determined by the 3-tier threshold on `kvo` (§6.6 Step 3).

Both are additive in the final `total_with_travel_min` formula and must not be collapsed. The second PZV cost is zero when `kvo ≤ REPAIR_TRAVEL_ZERO_THRESHOLD` (≤ 5), meaning objects with few repair types pay no repair PZV overhead at all.

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

### C-26: R2 Is Deeper Than R1 But Counted Additively in the Formula
R2 maintenance is a full inspection that covers the scope of R1 work. However, the Excel workbook treats R1 and R2 visit budgets as additive: `annual = R1_per_visit × R1_visits + R2_per_visit × R2_visits`. This is confirmed by analysis of the ОС Расчет sheet. The application must replicate this additive model exactly. Substituting R2 for R1 on shared visit days (i.e. `R1 × (R1_visits − R2_visits) + R2 × R2_visits`) gives a different and incorrect result.

### C-27: Travel Time Is Per-Object, Relative to Engineer's Home Division
The `Дорога` sheet stores one travel time per object. This time represents the distance from the responsible engineer's home division office to the object. It is entered manually — not computed from coordinates. When the responsible engineer changes (different home division), the travel time field must be reviewed and updated by an editor or admin. The system prompts for review but does not block saving.

### C-28: Recalculation Is On-Demand — Not Automatic
The system marks summaries stale immediately when data changes, but does not automatically trigger recalculation. Recalculation runs only when an admin explicitly presses "Пересчитать". This is intentional for PoC — it prevents background calculation load during bulk data entry sessions. Automatic triggering is a post-MVP enhancement.

### C-29: Planning Periods Lock Operational Data
Once a period is deactivated, its `object_repairs` and `records_tasks` rows become read-only for all roles. This ensures historical data integrity — past period calculations remain reproducible. Equipment, normatives, and travel data are not period-scoped in PoC; only repair counts and records tasks are.

### C-30: Object Deletion Is Hard Delete for PoC
Deleted objects are permanently removed. There is no soft-archive or decommission status in PoC. Post-MVP should introduce `is_active = FALSE` on objects to handle closed branches while preserving historical calculation data.

### C-31: Import Creates Placeholder Accounts for Unresolved Engineers
When an engineer name from the `Ответственные ТО` column does not match any existing `users.name`, the import creates a placeholder account (`role = 'engineer'`, `is_active = FALSE`, `requires_activation = TRUE`). The object is assigned to this placeholder. The post-import report lists all placeholder accounts. Admins activate them by setting a password. This prevents coverage gaps from appearing due to import name mismatches.

### C-32: `responsible_engineer` VARCHAR Field Does Not Exist in the Schema
The original Excel free-text field is replaced by `object_engineers` join rows. After import, the text is resolved to user accounts (or placeholders). No VARCHAR field is retained on the `objects` table. The СВОД export populates the "Ответственные ТО" column by joining `users.name` through `object_engineers`.

### C-36: К-во ремонтов Is COUNT of Distinct Repair Types, Not SUM of Quantities
`К-во ремонтов` (total repair count) = `COUNT(object_repairs rows WHERE count > 0)` for the current period. It counts how many distinct repair types were performed at least once — not the total number of individual repair operations. An object with "Замена аккумулятора × 3" and "Замена извещателя × 5" has `kvo = 2`, not 8. This is the value used in the repair travel/PZV threshold formula. The field is always computed; it is never user-entered. Verified against the Ремонт Расчет sheet: object "Архив г.Брест" has 8 distinct repair types → `kvo = 8`.

### C-37: Repair Travel and PZV Use a 3-Tier Threshold on kvo
Both `repair_travel_6months` and `repair_pzv_6months` apply the same threshold logic:
- `kvo ≤ 5` → zero overhead (too few repairs to justify extra trips)
- `5 < kvo ≤ 10` → overhead = `kvo × rate` (one trip/PZV per repair type)
- `kvo > 10` → overhead capped at `10 × rate`

The two formulas differ only in the rate used: travel uses `round_trip_min`, PZV uses `config[PZV_MINUTES]`. Neither uses the raw sum of repair quantities. The threshold values (5 and 10) are stored in `app_config` as `REPAIR_TRAVEL_ZERO_THRESHOLD` and `REPAIR_TRAVEL_CAP`. Verified across all non-zero repair rows in source XLSX: zero mismatches.

### C-33: Aggregations Are Never Cached in the Database
Branch, division, and company-wide required FTE values are computed by live SQL aggregation over `summaries` at query time. No `branch_summaries`, `division_summaries`, or `company_summary` tables exist. With ~2,935 objects, a SUM over `summaries.itogo_chislo_with_travel` grouped by `division_id` completes in milliseconds with a proper index. Caching these aggregations would add invalidation complexity with no meaningful performance benefit.

### C-34: Staffing Need Is Required FTE Only — No Capacity Comparison at Branch/Division Level
`branch_load` and `division_load` express how many full-time engineers are theoretically required to cover all objects. They do not compare against available engineer capacity. Capacity comparison (load ratio, overload status) is defined only at the individual engineer level (§6.13). Division managers see required FTE and must use their own knowledge of assigned headcount to assess adequacy.

### C-35: Import Is the Highest-Priority MVP Item
The PoC requires manual data entry. With 2,935 objects, manual entry is a demo-only shortcut — not a viable production workflow. XLSX import (M-01) must be the first item delivered in MVP, before any other MVP feature, as it is the precondition for real users adopting the system.

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
API ignores or rejects attempts to set `total_repairs`, `round_trip_min`, `is_stale`, or any `summaries` field directly. Returns HTTP 422 if attempted. Note: `total_repairs` is the stored snapshot of `kvo` (COUNT of distinct repair types with count > 0) at the time of last calculation — it is a computed output, not an input.

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

### AC-19: Period Data Isolation
Repair counts entered in period H1 2025 must not appear in H2 2025 calculations. After switching the active period, `GET /objects/:id/repairs` returns 0 counts for repair types with no H2 2025 rows, even if H1 2025 rows exist for the same types.

### AC-20: On-Demand Recalculation Only
After updating a normative (`PUT /catalog/devices/:id/contexts/:cid`), all affected summaries must be marked `is_stale = TRUE` but values in the СВОД must remain unchanged (showing stale indicator) until `POST /svod/recalculate` is called. Auto-recalculation must not occur.

### AC-21: Import Engineer Resolution
Import of `Шаблон_нагрузки_з_v_4_00.xlsx` must: (a) match engineer names to existing `users.name` records, (b) create placeholder accounts for unresolved names, (c) return a report listing matched engineers, created placeholders, and objects with no engineer name in the source. No object must be left without an `object_engineers` row after import (all get either a matched or placeholder account).

### AC-22: Repair Threshold Formula — Three-Band Correctness
All three threshold bands must be verified in the integration test suite (`CalculationServiceTest`):

**Band A — mid-range (5 < kvo ≤ 10):** For the reference object with kvo = 8, `round_trip_min = 20`, `PZV = 20`, `repair_work_6months = 361`:
- `effective_trips = 8`
- `repair_travel_6months = 8 × 20 = 160`
- `repair_pzv_6months    = 8 × 20 = 160`
- `repair_with_travel_monthly = (361 + 160 + 160) / 5 = 136.2 ±0.001`

**Band B — zero threshold (kvo ≤ 5):** For an object with kvo = 3:
- `effective_trips = 0`
- `repair_travel_6months = 0`, `repair_pzv_6months = 0`

**Band C — cap (kvo > 10):** For an object with kvo = 17, `round_trip_min = 20`:
- `effective_trips = 10` (capped)
- `repair_travel_6months = 10 × 20 = 200`
- `repair_pzv_6months    = 10 × 20 = 200`

## 15. PoC Scope

### 15.1 Purpose

The PoC has one goal: **demonstrate that the web application correctly calculates ИТОГО Числ per object and per engineer**, verifiable by manual spot-checking against the source Excel file. It is the minimum build that proves the core value proposition to stakeholders and justifies full MVP investment.

PoC is **not** a stripped-down MVP. It is a focused validator. Some simplifications will be reversed in MVP — these are explicitly documented below so developers build with the migration in mind.

---

### 15.2 PoC Scope — What Is Built

#### Core (non-negotiable for PoC)

| Feature | Notes |
|---|---|
| **Manual data entry** | Create objects, enter equipment, repairs, records, travel via UI. All 2,935 objects from source file entered manually (import is MVP). |
| **Calculation engine** | Full calculation: ОС, ПС, Видео, Записи, Ремонт, Дорога → ИТОГО Числ. All formulas from §6. |
| **СВОД table** | Paginated table matching all 19 source columns. |
| **СВОД export to XLSX** | Export matching original template structure. Stakeholders verify spot-checked rows against source file. |
| **Object detail page** | View and edit equipment quantities, repairs, records, travel. |
| **Engineer module** | Engineers as users, object-engineer assignments, workload split, capacity, load ratio, overload status, engineer dashboard. Full §4.10–4.11, §6.12–6.14, §7.4–7.7. |
| **Aggregation layer** | Branch, division, company-wide required FTE totals. All computed on the fly (§16). |
| **Basic dashboard** | Required FTE by division, top objects by workload, coverage gaps. |
| **Authentication** | Login / logout. Single role — all authenticated users can read and write. |
| **Seed normatives** | Device catalog and repair types loaded as migration seed data. Values from source XLSX. |
| **Basic observability** | Structured logging, health endpoint, job failure alerting (§18 PoC tier). |

---

### 15.3 PoC Simplifications (Intentional Deviations from Full TOR)

Each simplification is labelled, explained, and marked with its reversal milestone.

**S-01: Flat equipment model — no physical inventory layer**

Full TOR: two layers — `object_devices` (physical qty) + `object_system_assignments` (maintained qty).

PoC: one table `equipment_entries` — `(object_id, device_type_id, system_type, quantity)`. Physical qty is implicitly equal to maintained qty.

*Reversed in:* M-03 (MVP)

**S-02: Synchronous recalculation on save — no staleness tracking**

Full TOR: `is_stale` flag + on-demand background job triggered by admin.

PoC: recalculates object summary synchronously when any of its data is saved (~5ms per object). No background worker, no `is_stale` column, no admin trigger button.

*Reversed in:* M-06 (MVP)

**S-03: Static normatives — no admin UI for catalog**

Full TOR: admin-editable device types, system contexts, and repair types via UI.

PoC: normatives loaded as migration seed data. No UI to add, edit, or delete device types, contexts, or repair types. Calculation uses seed values directly.

*Reversed in:* M-04, M-05 (MVP)

**S-04: Single role — no RBAC**

Full TOR: admin, editor, engineer, viewer roles with division scoping.

PoC: one role — authenticated user. Login/logout only. Engineers log in as regular users; their engineer-specific fields (capacity_fte) are set directly in the database for PoC.

*Reversed in:* M-02 (MVP)

**S-05: No planning periods**

Full TOR: `periods` table; repairs and records are period-scoped.

PoC: one `object_repairs` row per (object, repair_type), one `records_tasks` row per object — no period FK.

*Reversed in:* M-01 (MVP)

**S-06: XLSX import is manual — import via UI is MVP**

Full TOR: bulk XLSX import of source file.

PoC: all data entered manually through the UI. For demo purposes, a representative subset of objects (~20–50 from different divisions) is entered, not all 2,935. Full import is the first MVP milestone.

*Reversed in:* M-01 (MVP) — this is the highest-priority MVP item since 2,935 rows of manual entry is not viable for production.

**S-07: No audit log**

Full TOR: changes to normatives, config, and assignments are audited.

PoC: no audit log.

*Reversed in:* M-08 (MVP)

**S-08: No PDF export**

PoC exports СВОД to XLSX only.

*Reversed in:* M-09 (post-MVP)

**S-09: No concurrency/locking**

Full TOR: optimistic locking with `updated_at` version check (§17).

PoC: last-write-wins. Acceptable with 1–2 demo users.

*Reversed in:* M-11 (MVP)

---

### 15.4 PoC Data Model

Simplified schema — replaces the full §5 schema for PoC only. Designed as a strict subset: no columns are dropped when migrating to MVP, only new tables and columns are added.

```sql
-- Org structure
divisions  (id, name)
branches   (id, division_id, name)
objects    (id, branch_id, name, responsible_engineer_text VARCHAR, import_seq_no)
-- Note: responsible_engineer_text is a display-only label for PoC.
-- Replaced by object_engineers join table in MVP (S-04 reversal).

-- Device catalog — seed data, read-only in PoC
device_types            (id, name)
device_system_contexts  (id, device_type_id, system_type, r1_minutes, r2_minutes)
repair_types            (id, name, time_minutes)

-- Equipment — flat model (S-01)
equipment_entries  (id, object_id, device_type_id, system_type, quantity DECIMAL(10,2))
UNIQUE(object_id, device_type_id, system_type)

-- Operational data — no period FK (S-05)
records_tasks  (id, object_id, access_requests, monitoring_requests,
                footage_requests, backup_control, security_admin)
UNIQUE(object_id)

object_repairs  (id, object_id, repair_type_id, count INTEGER)
UNIQUE(object_id, repair_type_id)

travel  (id, object_id, transport_type, distance_km, one_way_time_min)
UNIQUE(object_id)

-- Computed cache — synchronous, no is_stale (S-02)
summaries  (id, object_id,
            os_r1_per_visit, os_r2_per_visit,
            ps_r1_per_visit, ps_r2_per_visit,
            video_r1_per_visit, video_r2_per_visit,
            r1_per_visit_total, r2_per_visit_total,
            os_monthly_avg, ps_monthly_avg, video_monthly_avg,
            records_monthly,
            total_repairs,   -- COUNT of distinct repair types with count > 0
            repair_no_travel_monthly, repair_with_travel_monthly,
            round_trip_min, pzv_minutes,
            total_no_travel_min, itogo_chislo_no_travel,
            total_with_travel_min, itogo_chislo_with_travel,
            computed_at)
UNIQUE(object_id)

-- Engineer module — full (S-04 partial: no role column yet)
users  (id, email, name, password_hash,
        capacity_fte DECIMAL(4,2) DEFAULT 1.0,
        home_division_id UUID FK → divisions.id,
        is_active BOOLEAN DEFAULT TRUE,
        created_at)
-- No role column for PoC; all users treated as admins

object_engineers  (id, object_id, engineer_id, assigned_at)
UNIQUE(object_id, engineer_id)

engineer_summaries  (id, engineer_id,
                     total_load, object_count,
                     os_load, ps_load, video_load, records_load, repair_load,
                     capacity_fte, load_ratio, status,
                     computed_at)
UNIQUE(engineer_id)

-- Config constants — hardcoded in application for PoC (S-03)
-- Moved to app_config table in MVP (M-10)
```

---

### 15.5 PoC UI Routes

| Route | View |
|---|---|
| `/login` | Login page |
| `/` | Dashboard — required FTE by division, top 10 objects, coverage gaps |
| `/objects` | Object list — searchable, filterable, shows ИТОГО Числ |
| `/objects/new` | Create object |
| `/objects/:id` | Object detail — 5 tabs: Оборудование, Записи, Ремонт, Дорога, Инженеры |
| `/objects/:id/svod` | СВОД tab (computed summary, read-only) |
| `/engineers` | Engineer list — load ratio, status, object count |
| `/engineers/:id` | Engineer detail — workload dashboard |
| `/svod` | Full СВОД table — paginated, filterable |
| `/svod/export` | Trigger XLSX export |

No catalog management pages, no admin pages, no periods page.

---

### 15.6 PoC Acceptance Criteria

| ID | Criterion |
|---|---|
| **PAC-01** | For a manually entered object matching "Архив г.Брест, ул.Московская, 202Д" with the correct equipment quantities, `itogo_chislo_with_travel = 0.032327 ±0.000001`. |
| **PAC-02** | For an engineer assigned as the sole responsible engineer for that object with `capacity_fte = 1.0`, `engineer_total_load = 0.032327 ±0.000001` and `status = "normal"`. |
| **PAC-03** | СВОД XLSX export, when opened in Excel, matches manually verified reference values within ±0.001. |
| **PAC-04** | Editing any equipment quantity in the UI and saving immediately updates the СВОД tab and the engineer's load ratio without page refresh. |
| **PAC-05** | СВОД table loads first 100 rows in under 3 seconds. |
| **PAC-06** | Unauthenticated requests to any route redirect to `/login`. |
| **PAC-07** | When a second engineer is assigned to the reference object, both engineers' `total_load` updates to `0.032327 / 2 = 0.016163 ±0.000001`. |
| **PAC-08** | Division dashboard shows correct required FTE total = SUM of `itogo_chislo_with_travel` for all objects in that division. |
| **PAC-09** | Health endpoint `GET /health` returns HTTP 200 with `{"status": "ok"}` and database connectivity check. |

---

### 15.7 PoC → MVP Migration Path

Items are ordered by dependency. M-01 is the highest priority because manual entry of 2,935 objects is not viable for production use.

| ID | Item | Depends on | Notes |
|---|---|---|---|
| M-01 | XLSX import (FR bulk) | — | Highest priority. Resolves S-06. Includes engineer name resolution and placeholder accounts. |
| M-02 | RBAC — admin, editor, engineer, viewer roles | M-01 | Resolves S-04. Division scoping for editors. |
| M-03 | Physical inventory layer (`object_devices`) | — | Resolves S-01. Split equipment UI into two sections. |
| M-04 | Device catalog management UI | M-02 | Resolves S-03 (devices). Admin-only. |
| M-05 | Repair type catalog management UI | M-02 | Resolves S-03 (repairs). Admin-only. |
| M-06 | Staleness tracking + on-demand recalculation | M-02 | Resolves S-02. Background worker, `is_stale`, admin trigger. |
| M-07 | Planning periods (FR-12) | M-01, M-06 | Resolves S-05. Period selector in UI, period lock. |
| M-08 | Audit log | M-02 | Resolves S-07. |
| M-09 | Concurrency / optimistic locking | M-06 | Resolves S-09. `updated_at` version check on writes. |
| M-10 | `app_config` table with admin UI | M-02 | Move hardcoded constants to DB. |
| M-11 | PDF export | — | Resolves S-08. Post-MVP. |

---

### 15.8 PoC Technology Stack

The PoC uses the **full production stack** defined in §9.1 — no throwaway stack, no language switch between PoC and MVP. This prevents a rewrite at MVP transition and means every line of PoC code is production-eligible.

#### Backend — PoC Configuration

| Component | PoC setting | Notes |
|---|---|---|
| Java 21 + Spring Boot 3.x | Full stack | Same as MVP |
| Spring Data JPA | PoC simplified schema (§15.4) | Entities match PoC tables exactly; MVP entities added incrementally |
| Liquibase | `v1.0.0-initial-schema.xml` + seed data changeset | PoC schema + all seed normatives in one migration |
| Spring Security + JWT | Single role, no refresh tokens | Refresh tokens added in M-02 (MVP RBAC) |
| Apache POI | XLSX export only | Import (POI read) added in M-01 (MVP) |
| Redis | Job queue for recalculation | Even synchronous PoC recalc uses Redis queue stub for forward-compatibility |
| log4j2 | JSON layout from day one | Structured logging is non-negotiable even in PoC |
| Spring Actuator | `/actuator/health` exposed | Used for Docker Compose healthcheck; `/actuator/metrics` enabled for Datadog |
| Bucket4j | Basic rate limiting on `/api/**` | Prevents accidental hammering during demo |

#### Frontend — PoC Configuration

| Component | PoC setting | Notes |
|---|---|---|
| React 18 + TypeScript + Vite | Full setup | No shortcuts; strict TypeScript from start |
| Material UI | MUI DataGrid for СВОД table | Pagination and sorting built-in; handles 2,935 rows |
| TanStack Query | Server state for all API calls | Stale-while-revalidate; automatic refetch after mutations |
| Zustand | Minimal client state (auth token, current user) | No Redux for PoC |
| React Hook Form + Zod | All data-entry forms | Equipment, repairs, records, travel, engineer assignment |
| Axios | HTTP client with JWT interceptor | 401 → redirect to login |

#### Infrastructure — PoC Docker Compose

```yaml
services:
  backend:   # Spring Boot JAR, port 8080
  frontend:  # Nginx serving Vite build, port 3000
  postgres:  # PostgreSQL 15, port 5432
  redis:     # Redis 7, port 6379
  nginx:     # Reverse proxy: / → frontend, /api → backend
             # TLS termination with self-signed cert for PoC
```

**Schema continuity guarantee:** The PoC Liquibase changelog (`v1.0.0`) defines a strict subset of the full MVP schema. MVP migrations (`v1.1.0` onwards) add tables and columns — they never drop or rename PoC columns. PoC data survives migration intact.

### 15.9 Architecture: Monolith for PoC and MVP

The system is built as a **single monolithic service** for both PoC and MVP. A separate microservice for engineer workload is not justified because:

- Engineer workload is a simple aggregation (`SUM` over already-cached `summaries` rows) — no independent data source, no separate scaling requirement.
- A service boundary would require inter-service auth, network calls, and separate deployment — significant complexity with no benefit at this scale.
- The monolith can be decomposed later if the engineer module grows to include scheduling, mobile access, or push notifications.

Post-MVP consideration: if the engineer dashboard needs real-time push updates or a mobile client, extract the engineer notifications concern into a lightweight event service. The calculation core stays in the monolith.


## 16. Aggregation Rules

All branch, division, and company-wide metrics are **computed on the fly** from the `summaries` table. No aggregation results are stored in the database — queries run against fresh `summaries` rows every time. This is viable because the data volume is bounded (~2,935 objects) and the queries are simple SUM aggregations.

---

### 16.1 Required FTE — Definition

At every level, the metric is **required FTE** — the total maintenance workload expressed as full-time equivalent headcount needed to cover all assigned objects:

```
required_fte = SUM(summaries.itogo_chislo_with_travel)
               for all objects in scope
```

The `itogo_chislo_with_travel` variant is used throughout the aggregation chain (not `no_travel`). This is the operationally correct figure — it includes time engineers spend travelling to objects.

There is **no capacity comparison** at branch or division level. Required FTE is reported as an absolute number. Capacity comparison lives only at the individual engineer level (§6.13).

---

### 16.2 Aggregation Chain

```
object_load   = summaries.itogo_chislo_with_travel          (per object)

branch_load   = SUM(object_load)
                WHERE objects.branch_id = branch.id          (per branch)

division_load = SUM(object_load)
                WHERE branches.division_id = division.id     (per division)
              = SUM(branch_load) for all branches in division

company_load  = SUM(object_load) for all objects             (company-wide)
              = SUM(division_load) for all divisions
```

All three levels are derived from the same `summaries` table. The chain is consistent by construction: `company_load = SUM(division_loads) = SUM(branch_loads) = SUM(object_loads)`.

---

### 16.3 System-Component Breakdown at Each Level

The aggregation chain applies identically to each system component:

```
branch_os_load      = SUM(summaries.os_monthly_avg / 60 / MONTHLY_HOURS_FUND × ABSENCE_COEFFICIENT)
branch_ps_load      = SUM(summaries.ps_monthly_avg / 60 / MONTHLY_HOURS_FUND × ABSENCE_COEFFICIENT)
branch_video_load   = SUM(summaries.video_monthly_avg / 60 / MONTHLY_HOURS_FUND × ABSENCE_COEFFICIENT)
branch_records_load = SUM(summaries.records_monthly / 60 / MONTHLY_HOURS_FUND × ABSENCE_COEFFICIENT)
branch_repair_load  = SUM(summaries.repair_with_travel_monthly / 60 / MONTHLY_HOURS_FUND × ABSENCE_COEFFICIENT)

branch_load = branch_os_load + branch_ps_load + branch_video_load
            + branch_records_load + branch_repair_load
            + SUM(pzv + round_trip) contribution
```

In practice it is simpler and equivalent to aggregate `itogo_chislo_with_travel` directly and decompose using the component ratios from individual summaries for display purposes.

---

### 16.4 Staffing Need per Branch and Division

```
staffing_need_branch   = CEIL(branch_load × 10) / 10
                         -- rounded up to nearest 0.1 FTE for reporting

staffing_need_division = CEIL(division_load × 10) / 10

staffing_need_company  = CEIL(company_load × 10) / 10
```

> **Note:** The rounding rule (`CEIL` to nearest 0.1) is a reporting convention. The underlying `branch_load` and `division_load` values are stored and compared with full precision. Only the displayed staffing need figure is rounded.

---

### 16.5 Coverage Metrics per Branch and Division

```
objects_total_branch          = COUNT(objects) WHERE branch_id = branch.id
objects_with_engineer_branch  = COUNT(DISTINCT object_id FROM object_engineers
                                       WHERE object_id IN objects of branch)
coverage_gap_count_branch     = objects_total_branch - objects_with_engineer_branch

uncovered_load_branch         = SUM(itogo_chislo_with_travel)
                                 for objects with NO rows in object_engineers
                                 and branch_id = branch.id
```

`uncovered_load_branch` is the required FTE that currently has no engineer assigned — this is the figure management uses to justify new hires.

The same formulas apply at division level by summing across all branches in the division.

---

### 16.6 Overload Summary per Branch and Division

```
engineers_overloaded_branch   = COUNT(users)
                                 WHERE role = 'engineer'
                                 AND home_division_id maps to this branch's division
                                 AND engineer_summaries.status = 'overloaded'

engineers_warning_branch      = COUNT(users) ... WHERE status = 'warning'
```

> Note: overload is attributed to the engineer's `home_division_id`, not to where their objects are located. An engineer may service objects in multiple divisions — their overload status is reported under their home division.

---

### 16.7 API Aggregation Endpoints

Aggregation queries are exposed as dedicated read-only endpoints (see §10). They are not cached — they run live against `summaries`:

```
GET /aggregations/company              Company-wide totals
GET /aggregations/divisions            All divisions with load, staffing_need, gaps
GET /aggregations/divisions/:id        Single division detail
GET /aggregations/branches             All branches with load, staffing_need, gaps
GET /aggregations/branches/:id         Single branch detail
```

Response shape for division:
```json
{
  "division_id": "...",
  "division_name": "Брестское областное управление №100",
  "object_count": 312,
  "required_fte": 4.2831,
  "staffing_need": 4.3,
  "uncovered_load": 0.124,
  "coverage_gap_count": 8,
  "engineers_total": 4,
  "engineers_overloaded": 1,
  "engineers_warning": 0,
  "breakdown": {
    "os":      1.121,
    "ps":      0.983,
    "video":   0.441,
    "records": 0.217,
    "repair":  1.521
  }
}
```

---

### 16.8 PoC vs MVP Scope for Aggregation

All aggregation rules defined in this section are **in scope for PoC**. Aggregation queries are on-the-fly SQL — no new tables, no caching mechanism, no additional migrations beyond what the PoC schema already provides. The aggregation endpoints and dashboard widgets are part of the PoC deliverable.


## 17. Concurrency & Locking

**Scope: MVP.** The PoC uses last-write-wins (S-09). This section defines the strategy that replaces S-09 in M-11.

---

### 17.1 Risk Scenarios

With editors across multiple divisions and admins performing bulk operations simultaneously, uncontrolled concurrent writes produce silent data loss:

- Two editors open the same object's equipment tab simultaneously. Both make changes. The second save overwrites the first with no error.
- An admin updates a normative while a background recalculation job is mid-flight. The job writes a summary computed from the old normative after the new normative is saved.
- A bulk XLSX import runs while an editor is updating an object that appears in the import. The import overwrites the editor's in-flight change.

---

### 17.2 Strategy: Optimistic Locking via `updated_at` Version

All writable tables carry an `updated_at TIMESTAMP` column. On every write, the client sends the `updated_at` value it last read. The server checks that the row's current `updated_at` matches before applying the update:

```sql
UPDATE equipment_entries
SET    quantity = :new_qty, updated_at = now()
WHERE  id = :id
AND    updated_at = :client_version;   -- optimistic lock check

-- If 0 rows affected → conflict detected → return HTTP 409
```

This requires no database-level locks and adds no latency on the happy path.

---

### 17.3 Conflict Response

When a conflict is detected (0 rows updated), the API returns:

```json
HTTP 409 Conflict
{
  "error": {
    "code": "EDIT_CONFLICT",
    "message": "Данные изменены другим пользователем. Перезагрузите страницу.",
    "current_updated_at": "2025-06-15T14:23:11Z"
  }
}
```

The UI catches 409 responses and displays a non-blocking alert:
> **"Данные были изменены другим пользователем. Ваши изменения не сохранены. Перезагрузить страницу?"**

No automatic merge is attempted. The user must reload to get the latest state and re-enter their changes if needed.

---

### 17.4 Tables Covered by Optimistic Locking

| Table | `updated_at` column | Locking applied |
|---|---|---|
| `objects` | ✅ | On metadata updates |
| `equipment_entries` / `object_system_assignments` | ✅ | On quantity changes |
| `records_tasks` | ✅ | On task count updates |
| `object_repairs` | ✅ | On repair count updates |
| `travel` | ✅ | On travel data updates |
| `object_engineers` | ✅ | On assignment changes |
| `device_system_contexts` | ✅ | On normative edits (admin only) |
| `repair_types` | ✅ | On time_minutes edits (admin only) |
| `app_config` | ✅ | On constant changes (admin only) |
| `summaries` / `engineer_summaries` | ❌ | Written only by background worker; no concurrent user writes |

---

### 17.5 Transaction Boundaries

The following operations must execute within a single database transaction:

| Operation | What must be atomic |
|---|---|
| Save equipment + trigger recalc | UPDATE equipment_entries + UPDATE summaries (if sync) |
| Assign engineer to object | INSERT object_engineers + mark engineer_summaries stale |
| Remove engineer from object | DELETE object_engineers + mark all co-engineers' summaries stale |
| Activate a planning period | UPDATE periods SET is_active=FALSE (all) + UPDATE periods SET is_active=TRUE (new) |
| Bulk XLSX import | All object/equipment/repair/records INSERTs + summary computation — committed together or rolled back entirely |
| Normative update | UPDATE device_system_contexts + mark all affected summaries stale |

Transactions must not span HTTP requests. Long-running operations (bulk import, bulk recalculation) run inside a single database transaction per batch, not per row.

---

### 17.6 Import Concurrency

The XLSX import (MVP) locks the affected objects for the duration of the import batch using `SELECT ... FOR UPDATE` on the `objects` rows being imported. Any concurrent edit to those objects returns HTTP 409 to the editor attempting the edit. The import holds the lock for a maximum of 30 seconds per batch; if exceeded, the import batch is rolled back and re-queued.

---

### 17.7 Background Worker Isolation

The recalculation worker (MVP) reads `summaries.is_stale = TRUE` rows and updates them. To prevent two worker instances from computing the same object simultaneously:

```sql
-- Worker claims a batch atomically
UPDATE summaries
SET    is_stale = 'PROCESSING'   -- intermediate status
WHERE  is_stale = TRUE
LIMIT  100
RETURNING id, object_id;
```

If the worker crashes mid-batch, a watchdog resets `PROCESSING → TRUE` after a configurable timeout (default: 5 minutes). This prevents summaries from being stuck in `PROCESSING` state permanently.


## 18. Observability & Monitoring

Requirements are tiered: **PoC** covers the minimum needed to operate the demo confidently; **MVP** adds production-grade monitoring.

---

### 18.1 PoC Tier (Required for PoC Delivery)

#### Structured Logging — log4j2

Configure log4j2 with `JsonTemplateLayout` (or `JsonLayout`) so all log output is newline-delimited JSON to stdout. Docker captures stdout; Datadog Agent tails container logs.

`log4j2-spring.xml` minimum configuration:

```xml
<JsonTemplateLayout eventTemplateUri="classpath:LogstashJsonEventLayoutV1.json"/>
```

Minimum fields on every log line:

```json
{
  "timestamp": "2025-06-15T14:23:11.432Z",
  "level": "INFO",
  "logger": "com.company.workload.service.CalculationService",
  "thread": "http-nio-8080-exec-3",
  "request_id": "c3d8e2a1",
  "user_id": "uuid-or-null",
  "message": "Object summary recalculated",
  "object_id": "uuid",
  "duration_ms": 4,
  "service": "workload-api",
  "version": "1.0.0",
  "environment": "poc"
}
```

`request_id` is injected by a Spring `OncePerRequestFilter` that generates a UUID and stores it in MDC. `user_id` is set from the JWT principal in the same filter.

Log levels: `DEBUG` (dev only, disabled in all other environments), `INFO` (normal operations), `WARN` (recoverable issues — e.g. stale summary detected), `ERROR` (exceptions, failed operations, job failures).

**No plaintext logs permitted in any environment** — including PoC. log4j2 `ConsoleAppender` with JSON layout is configured from day one.

#### Health Endpoint — Spring Actuator

Spring Boot Actuator provides the health endpoint out of the box:

```
GET /actuator/health
```

Auto-configured `HealthIndicators` check:
- PostgreSQL datasource connectivity (`DataSourceHealthIndicator`)
- Redis connectivity (`RedisHealthIndicator`)
- Disk space (`DiskSpaceHealthIndicator`)

Returns HTTP 200 `{"status": "UP"}` when all healthy; HTTP 503 when any component is DOWN.

`/actuator/health` is accessible publicly (used by Nginx health check and Docker Compose `healthcheck`). All other `/actuator/**` endpoints are restricted to internal network only (Nginx rule: deny external access to `/actuator` except `/actuator/health`).

Docker Compose healthcheck:
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
  interval: 30s
  timeout: 5s
  retries: 3
```

#### Datadog Integration — PoC

Install the Datadog Agent as a Docker Compose sidecar:

```yaml
datadog-agent:
  image: datadog/agent:latest
  environment:
    DD_API_KEY: ${DD_API_KEY}
    DD_LOGS_ENABLED: "true"
    DD_APM_ENABLED: "true"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - /proc:/host/proc:ro
    - /sys/fs/cgroup:/host/sys/fs/cgroup:ro
```

Label backend container for Datadog log collection:
```yaml
backend:
  labels:
    com.datadoghq.ad.logs: '[{"source":"java","service":"workload-api"}]'
```

Spring Actuator exports metrics to Datadog via Micrometer:
```yaml
# application.yml
management:
  metrics:
    export:
      datadog:
        api-key: ${DD_API_KEY}
        step: 30s
```

For PoC, this provides: log aggregation, JVM metrics (heap, GC, threads), HTTP request metrics, and basic APM tracing — with zero code instrumentation.

#### Error Alerting — PoC

Datadog alert on `status:error` log events from service `workload-api`. Notification via email (configure in Datadog UI). No on-call paging for PoC. 7-day log retention.

---

### 18.2 MVP Tier (Required Before Production Launch)

#### Application Performance Monitoring (APM)

Datadog APM is the single APM platform (chosen in §9.1). The Datadog Agent (introduced in PoC) is configured for full APM tracing in MVP via the Datadog Java APM agent (`dd-java-agent.jar`). Instrument:

- Every HTTP request: method, route, status code, duration, user_id
- Every database query: query type, table, duration, row count
- Every background job execution: job type, object count, total duration, success/failure
- Every calculation engine run: object_id, duration_ms, result value

#### Performance Thresholds and Alerting

| Metric | Warning threshold | Critical threshold |
|---|---|---|
| HTTP request p95 latency | > 1s | > 3s |
| HTTP error rate (5xx) | > 1% over 5 min | > 5% over 5 min |
| Background job duration | > 30s | > 90s |
| DB query p95 latency | > 200ms | > 500ms |
| DB connection pool exhaustion | > 80% | > 95% |
| Disk usage | > 70% | > 90% |

Alerts route to an admin notification channel (email or Slack webhook, configured via environment variable).

#### Database Monitoring

- Slow query log: capture all queries exceeding 500ms
- Connection pool metrics: active, idle, waiting connections
- Table bloat: monitor `summaries` and `engineer_summaries` for excessive dead tuples (auto-vacuum tuning)
- Index usage: verify the `one_active_period` partial index and UNIQUE indexes are used by query planner

#### Background Job Failure Alerting

The recalculation worker (MVP) must emit a structured `ERROR` log and trigger an alert if:
- A batch fails after 3 retry attempts
- Any object summary remains in `PROCESSING` status for > 5 minutes (watchdog timeout)
- The stale summary queue length exceeds 500 objects (backlog alert)

#### Log Retention

| Environment | Retention | Storage |
|---|---|---|
| PoC | 7 days | Local file rotation |
| MVP Staging | 14 days | Log aggregator |
| MVP Production | 90 days | Log aggregator (compliance minimum) |

#### Structured Logging Extensions for MVP

In addition to PoC fields, MVP log lines include:

```json
{
  "service": "workload-api",
  "version": "1.2.0",
  "environment": "production",
  "trace_id": "otel-trace-id",
  "span_id": "otel-span-id"
}
```

This enables distributed tracing if a second service is added (e.g., a notification service post-MVP).

---

### 18.3 Key Runbook Items (PoC)

Minimal operational runbook for PoC deployment:

| Scenario | Action |
|---|---|
| Application not responding | `docker compose restart api` — check `/health` |
| Database connection errors | Check PostgreSQL container logs; verify connection pool settings |
| Calculation produces wrong values | Check seed normative data via `/admin/config`; compare with source XLSX values in §4.2 |
| XLSX export fails | Check Datadog logs for `ApachePOI` or `XSSFWorkbook` exception; verify column mapping in §7.9 |
| All summaries show stale | Trigger `POST /svod/recalculate` as admin; monitor job logs |


## 19. Testing Strategy

### 19.1 Test Pyramid

```
         /\
        /  \   E2E (manual PoC, Playwright post-MVP)
       /----\
      /      \  Integration — RestAssured + Testcontainers
     /--------\
    /          \  Unit — JUnit 5 + Mockito
   /____________\
```

### 19.2 Unit Tests (JUnit 5 + Mockito)

**Scope:** Services, calculation engine, MapStruct mappers, Zod schemas (frontend).

**Mandatory coverage targets (enforced by Jacoco):**

| Package | Line coverage | Branch coverage |
|---|---|---|
| `*.service.calculation.*` | ≥ 95% | ≥ 90% |
| `*.service.*` (other) | ≥ 80% | ≥ 75% |
| `*.mapper.*` | ≥ 80% | — |
| Overall | ≥ 75% | ≥ 70% |

The calculation engine (§6) receives the highest coverage requirement because a calculation error is the primary business risk.

**Key unit test cases:**
- `CalculationServiceTest`: verify ОС, ПС, Видео monthly averages, repair monthly averages, and ИТОГО Числ against the verified reference values from §6.3 and §6.8.
- `RepairCalculationTest`: verify all three threshold bands explicitly (AC-22):
  - `kvo = 3` → `effective_trips = 0`, travel and PZV both zero
  - `kvo = 8` → `effective_trips = 8`, `repair_with_travel_monthly = 136.2`
  - `kvo = 17` → `effective_trips = 10` (cap), `repair_travel = 10 × round_trip_min`
  - `kvo = 5` → boundary: `effective_trips = 0` (at threshold, not above)
  - `kvo = 6` → boundary: `effective_trips = 6` (just above threshold)
  - `kvo = 10` → boundary: `effective_trips = 10` (at cap, not above)
  - `kvo = 11` → boundary: `effective_trips = 10` (one above cap, still capped)
  - `kvo = 0` → `effective_trips = 0`, `repair_work_6months = 0`
- `EngineerWorkloadServiceTest`: verify workload split for 1, 2, and 3 co-engineers; verify load_ratio and status transitions.
- `AggregationServiceTest`: verify branch_load = SUM(object_loads) for test dataset.

**Jacoco Maven configuration:**
```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <executions>
    <execution>
      <id>check</id>
      <goals><goal>check</goal></goals>
      <configuration>
        <rules>
          <rule>
            <element>PACKAGE</element>
            <limits>
              <limit>
                <counter>LINE</counter>
                <value>COVEREDRATIO</value>
                <minimum>0.75</minimum>
              </limit>
            </limits>
          </rule>
        </rules>
      </configuration>
    </execution>
  </executions>
</plugin>
```

Build fails if coverage drops below threshold. Coverage reports published to GitHub Actions artifacts on every PR.

### 19.3 Integration Tests (Testcontainers + RestAssured)

**Scope:** Repository layer, full HTTP API round-trips, Liquibase migration correctness.

**Testcontainers:** Each integration test class spins up a real PostgreSQL 15 container (and Redis container where needed). Spring `@SpringBootTest` with `webEnvironment = RANDOM_PORT`. Liquibase migrations run automatically on test DB startup — this validates that migration scripts are correct.

**RestAssured test examples:**

```java
// Calculation accuracy integration test
given()
  .auth().oauth2(adminToken)
  .body(referenceObjectEquipmentPayload)
.when()
  .post("/api/v1/objects/{id}/equipment", objectId)
.then()
  .statusCode(200);

given()
  .auth().oauth2(adminToken)
.when()
  .get("/api/v1/objects/{id}/summary", objectId)
.then()
  .statusCode(200)
  .body("data.itogo_chislo_with_travel",
        closeTo(0.032327, 0.000001));  // PAC-01 verified value
```

**Key integration tests:**
- Full calculation pipeline for reference object (PAC-01 value).
- Period isolation: repairs in period A do not appear in period B summary.
- Optimistic lock conflict: simultaneous PUT returns 409 for second writer (MVP).
- Role enforcement: editor cannot write to another division's objects (MVP).
- Liquibase: all migrations apply cleanly to empty DB; rollbacks apply where defined.

### 19.4 WireMock (External API Mocking)

WireMock is configured for any future external integrations (e.g., Datadog API calls in tests, external HR system for engineer import). Not used in PoC — no external API calls in PoC scope. Stub stubs are set up as placeholders in `src/test/resources/wiremock/`.

### 19.5 Frontend Testing

| Tool | Scope |
|---|---|
| Vitest | Unit tests for Zod schemas and utility functions |
| React Testing Library | Component tests for form validation behaviour |
| Manual testing | СВОД table rendering, export download, engineer dashboard |

Playwright E2E tests are post-MVP.

### 19.6 Test Execution in CI

See §20. All unit and integration tests run on every pull request. Build is blocked if:
- Any test fails
- Jacoco coverage drops below threshold
- Liquibase migration fails on clean DB


## 20. CI/CD Pipeline

### 20.1 Platform

GitHub Actions. All workflows defined in `.github/workflows/`.

### 20.2 Workflow: Pull Request (`pr.yml`)

Triggered on: every pull request to `main` or `develop`.

```
Steps:
1. Checkout
2. Set up Java 21 (Temurin)
3. Maven: compile
4. Maven: unit tests (JUnit 5 + Mockito)
5. Maven: integration tests (Testcontainers — requires Docker-in-Docker runner)
6. Jacoco: generate coverage report
7. Jacoco: enforce coverage threshold (build fails if below minimum)
8. Upload coverage report to GitHub Actions artifacts
9. Frontend: npm ci
10. Frontend: TypeScript type check (tsc --noEmit)
11. Frontend: Vitest unit tests
12. Post PR status check (pass/fail)
```

**No Docker build on PRs** — keeps PR feedback fast (target: under 5 minutes).

### 20.3 Workflow: Main Branch (`main.yml`)

Triggered on: push to `main` (after PR merge).

```
Steps:
1–11. All PR steps (full test suite)
12. Docker: build backend image  (tag: git SHA + latest)
13. Docker: build frontend image (tag: git SHA + latest)
14. Docker: push both images to container registry
15. Deploy: SSH to on-premise server, pull new images, docker compose up -d
16. Health check: poll /actuator/health until UP (timeout 60s)
17. Notify: post deployment status to Slack/email
```

### 20.4 Workflow: Nightly (`nightly.yml`)

Triggered on: cron `0 2 * * *` (02:00 UTC daily).

```
Steps:
1. Full test suite (same as PR)
2. OWASP Dependency Check (CVE scan on Maven dependencies)
3. Publish CVE report to GitHub Actions artifacts
4. Alert on new HIGH/CRITICAL CVEs
```

### 20.5 Environment Variables & Secrets

All sensitive values stored as GitHub Actions Secrets:

| Secret | Used by |
|---|---|
| `DD_API_KEY` | Datadog Agent in Docker Compose |
| `POSTGRES_PASSWORD` | PostgreSQL container |
| `JWT_SECRET` | Spring Security JWT signing |
| `REGISTRY_USERNAME` / `REGISTRY_TOKEN` | Docker image push |
| `DEPLOY_SSH_KEY` | SSH to on-premise server |
| `DEPLOY_HOST` | On-premise server address |

Non-sensitive config (port numbers, app version) stored as GitHub Actions Variables (not Secrets).

### 20.6 Container Registry

On-premise Docker registry or GitHub Container Registry (`ghcr.io`). Image naming:

```
ghcr.io/{org}/workload-backend:{git-sha}
ghcr.io/{org}/workload-backend:latest
ghcr.io/{org}/workload-frontend:{git-sha}
ghcr.io/{org}/workload-frontend:latest
```

`latest` tag updated only on `main` branch pushes, not on PRs.

### 20.7 Deployment Docker Compose (Production)

```yaml
version: "3.9"
services:
  backend:
    image: ghcr.io/{org}/workload-backend:latest
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/workload
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
      SPRING_REDIS_HOST: redis
      JWT_SECRET: ${JWT_SECRET}
      DD_API_KEY: ${DD_API_KEY}
    depends_on:
      postgres: { condition: service_healthy }
      redis:    { condition: service_healthy }
    healthcheck:
      test: ["CMD","curl","-f","http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  frontend:
    image: ghcr.io/{org}/workload-frontend:latest

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: workload
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD","pg_isready","-U","postgres"]

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD","redis-cli","ping"]

  nginx:
    image: nginx:alpine
    ports: ["443:443", "80:80"]
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/nginx/certs:ro
    depends_on: [backend, frontend]

  datadog-agent:
    image: datadog/agent:latest
    environment:
      DD_API_KEY: ${DD_API_KEY}
      DD_LOGS_ENABLED: "true"
      DD_APM_ENABLED: "true"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /proc:/host/proc:ro
      - /sys/fs/cgroup:/host/sys/fs/cgroup:ro

volumes:
  pgdata:
```

### 20.8 Rollback Procedure

If the health check in step 16 of `main.yml` fails:

```
1. GitHub Actions marks deployment as failed
2. SSH to server: docker compose up -d --scale backend=0
3. Pull previous image tag (git SHA of last good commit)
4. docker compose up -d
5. Verify /actuator/health returns UP
6. Create GitHub Issue with deployment failure label
```

No automated rollback in PoC — manual procedure above. Automated rollback (blue-green or rolling) is post-MVP.


---

*End of Technical Specification — Version 2.7*
