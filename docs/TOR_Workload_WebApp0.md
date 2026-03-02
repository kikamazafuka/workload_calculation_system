# Technical Specification (TOR)
# Web Application: Security Systems Maintenance Workload Calculator
**Version:** 1.0  
**Based on:** Шаблон_нагрузки_з_v_4_00.xlsx  
**Date:** 2025-02-21

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
The Excel template is large (2,935 rows × up to 47 columns per sheet), manual to update, error-prone in formula propagation, and difficult to share or version. Calculation results depend on normative time standards (нормативы) stored in a separate sheet — any change requires cascading formula updates.

### 2.2 Goals
- Replace the multi-sheet Excel model with a centralized, browser-accessible application.
- Allow engineers and managers to input/update equipment quantities per facility.
- Automatically recalculate workload metrics on save.
- Support import of existing XLSX data and export of results to XLSX/PDF.
- Provide a СВОД (summary) dashboard per division and per responsible engineer.
- Make normative time standards editable by administrators without formula changes.

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
| Р1 | Time norm for routine inspection (per unit, in minutes) |
| Р2 | Time norm for full maintenance (per unit, in minutes) |
| Нормативы | Reference table of time standards (minutes per equipment unit) |
| СВОД | Consolidated summary — aggregated totals per object or division |
| ИТОГО Числ | Final headcount coefficient: total minutes / (monthly working minutes) |
| ПЗВ | Подготовительно-заключительное время — prep/wrap-up time (fixed 20 min per visit) |
| ТМЦ | Товарно-материальные ценности — inventory/material assets |

---

## 4. Functional Requirements

### 4.1 Object Management (FR-01)
- CRUD operations for objects (facilities).
- Each object must have: `division`, `branch`, `name/address`, `responsible_engineer`.
- Objects are organized in a two-level hierarchy: **Division → Branch → Objects**.
- Support bulk import from XLSX (see §11).

### 4.2 Equipment Inventory per Object (FR-02)
Equipment is grouped into three categories. Each category maps to a sheet in the source file.

**4.2.1 Security Alarm Systems (ОС)**
| Field | Unit |
|---|---|
| Galaxy 512 (Galaxy Dimension GD-520) | pcs |
| Maestro (ППК ОП Maestro-1600) | pcs |
| серий А6, Аларм | pcs |
| А16-512 | pcs |
| Выносная панель управления ВПУ–А-16 | pcs |
| Устройство доступа | pcs |
| Шлейфы сигнализации | pcs |
| Каналы считывания | pcs |
| Извещатели, оповещатели | pcs |
| Galaxy 512 (контроллер АСПС и СО) | pcs |
| Расширитель | pcs |
| Контроллер системы | pcs |
| Системный блок ПЦН | pcs |
| ББП-20, ББП-3/12 (БРП 2401) | pcs |

**4.2.2 Fire Alarm Systems (ПС)**
| Field | Unit |
|---|---|
| серий А6, Аларм | pcs |
| А16-512 | pcs |
| СПИ УОО Молния | pcs |
| АМ200-Notifier | pcs |
| Шлейфы сигнализации | pcs |
| Каналы считывания | pcs |
| Извещатели, оповещатели | pcs |
| Таблички | pcs |
| Galaxy 512 (контроллер АСПС и СО) | pcs |
| Оракул | pcs |
| Танго ПУ/БП | pcs |
| Танго ПУ/ЗК | pcs |
| Расширитель | pcs |
| Адресный модуль | pcs |
| Адресный шлейфно-релейный модуль | pcs |
| Усилитель линии УЛТ | pcs |
| Адресные извещатели | pcs |
| Колонки | pcs |

**4.2.3 Video Surveillance (Видео)**
| Field | Unit |
|---|---|
| Видеокамеры | pcs |
| Микрофоны | pcs |
| Системный блок (видео сервер) | pcs |

### 4.3 Records & Administration Tasks (FR-03) — "Записи"
Quantities of service requests/tasks per object per reporting period:

| Task | Unit |
|---|---|
| Запросы в связи с отсутствием (нарушением) доступа | requests |
| Запросы в связи с проведением мониторинга записей | requests |
| Запросы по предоставлению записей системы видеонаблюдения | requests |
| Контроль процесса резервного копирования одной системы | instances |
| Администрирование систем безопасности филиала | instances |

### 4.4 Repairs (FR-04) — "Ремонт"
Count of repair operations per object per reporting period:

| Repair Type |
|---|
| Замена приемно-контрольного прибора серии А6 ОС |
| Замена приемно-контрольного прибора серии А6 ПС |
| Замена извещателя охранного оптико-электронного |
| Замена извещателя пожарного дымового |
| Замена шунтирующих и оконечного резисторов шлейфа ОС |
| Замена шунтирующих и оконечного резисторов шлейфа ПС |
| Замена блока бесперебойного питания ОС |
| Замена блока бесперебойного питания ПС |
| Замена аккумулятора ОС |
| Замена аккумулятора ПС |
| Замена блока питания видеосервера |
| Замена винчестера видеосервера |
| Замена основных составных частей видеосервера в комплексе |
| Переустановка программного обеспечения на видеосервере |
| Восстановление сигнала IP камеры |
| Восстановление сигнала аналоговой камеры |
| Акт о выполненных работах ОС |
| Дефектный акт ОС |
| Акт на списание ТМЦ из подотчета ОС |
| Акт о выполненных работах ПС |
| Дефектный акт ПС |
| Акт на списание ТМЦ из подотчета ПС |
| Акт о выполненных работах Видео |
| Дефектный акт Видео |
| Акт на списание ТМЦ из подотчета Видео |
| К-во ремонтов (total repair count — auto-calculated) |

> **Note:** `К-во ремонтов` is the sum of all individual repair counts for the object and must be computed automatically — it is not a user input.

### 4.5 Travel Data (FR-05) — "Дорога"
Per object:
- Transport type (free-text or dropdown: пешком / на машине / общественный транспорт)
- Distance, km (numeric)
- One-way travel time, minutes (numeric)
- Round-trip time, minutes (**auto-calculated** as `one_way_time × 2`)

### 4.6 Normative Standards Management (FR-06) — "Нормативы"
Administrators must be able to view and edit all normative time values without code changes. Each normative record has:
- Equipment/task name
- Category (ПС / ОС / Видео / Записи / Ремонт)
- R1 time value (minutes per unit — routine inspection)
- R2 time value (minutes per unit — full maintenance)

Full normative table (from source file):

**ПС (Fire Alarm):**
| Equipment | R1 (min) | R2 (min) |
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

**Видео (Video):**
| Equipment | R1 (min) | R2 (min) |
|---|---|---|
| Видеокамеры | 1.5 | 1.5 |
| Микрофоны | 0.5 | 0.5 |
| Системный блок (видео сервер) | 10 | 40 |

**ОС (Security Alarm):**
| Equipment | R1 (min) | R2 (min) |
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

**Записи (Video Archive Administration):**
| Task | R1 (min) |
|---|---|
| Запросы в связи с отсутствием (нарушением) доступа | 60 |
| Запросы в связи с проведением мониторинга записей | 180 |
| Запросы по предоставлению записей системы видеонаблюдения | 180 |
| Контроль процесса резервного копирования | 120 |
| Администрирование систем безопасности | 60 |

> **Note on "Записи":** The normative sheet (rows 20 and 73–77) contains two slightly different values for some tasks. The authoritative values are from rows 73–77 of the normative sheet (120 min for backup control, 180 min for records requests). These must be used in calculations. See §13 for clarification.

**Ремонт (Repairs) — fixed time per operation:**
| Repair Type | Time (min) |
|---|---|
| Замена ПКП серии А6 ОС | 150 |
| Замена ПКП серии А6 ПС | 150 |
| Замена извещателя охранного оптико-электронного | 15 |
| Замена извещателя пожарного дымового | 12 |
| Замена шунтирующих/оконечного резисторов ОС | 35 |
| Замена шунтирующих/оконечного резисторов ПС | 35 |
| Замена ББП ОС | 30 |
| Замена ББП ПС | 30 |
| Замена аккумулятора ОС | 5 |
| Замена аккумулятора ПС | 5 |
| Замена блока питания видеосервера | 20 |
| Замена винчестера видеосервера | 10 |
| Замена основных составных частей видеосервера | 30 |
| Переустановка ПО видеосервера | 90 |
| Восстановление сигнала IP камеры | 35 |
| Восстановление сигнала аналоговой камеры | 20 |
| Акт о выполненных работах ОС | 7 |
| Дефектный акт ОС | 60 |
| Акт на списание ТМЦ ОС | 20 |
| Акт о выполненных работах ПС | 7 |
| Дефектный акт ПС | 60 |
| Акт на списание ТМЦ ПС | 20 |
| Акт о выполненных работах Видео | 7 |
| Дефектный акт Видео | 60 |
| Акт на списание ТМЦ Видео | 20 |

### 4.7 Consolidated Summary (СВОД) (FR-07)
The СВОД view shows per object:
- Prep/wrap-up time (ПЗВ): **fixed 20 min**
- Travel time (Дорога): round-trip minutes
- Time for ПС maintenance
- Time for Видео maintenance
- Time for ОС maintenance
- Time for Записи
- Repair time without travel
- Repair time with travel (includes travel per repair visit)
- **ИТОГО Числ (без дороги)** — headcount coefficient, repairs without travel component
- **ИТОГО Числ (с дорогой)** — headcount coefficient, repairs with travel component
- Р1 total time on object (all systems, routine)
- Р2 total time on object (all systems, full maintenance)

### 4.8 Export (FR-08)
- Export СВОД to XLSX (matching the original template structure).
- Export СВОД to PDF.
- Export individual object data sheets to XLSX.

### 4.9 Dashboard (FR-09)
Summary view grouped by **division** and by **responsible engineer**, showing:
- Total headcount (ИТОГО Числ) per division.
- Number of objects per engineer.
- Objects with no equipment data entered (data gaps).
- Top 10 objects by workload.

---

## 5. Data Model

### 5.1 Entity Relationship Overview

```
Division (1) ──< Branch (1) ──< Object (1)
                                   │
                   ┌───────────────┼───────────────┐
                   │               │               │
              EquipmentOS    EquipmentPS    EquipmentVideo
                   │               │               │
                   └───────────────┼───────────────┘
                                   │
                          ┌────────┴────────┐
                      Records           Repairs
                                   │
                               Travel
                                   │
                              Summary (computed)
```

### 5.2 Tables / Collections

#### `divisions`
```sql
id            UUID PK
name          VARCHAR(255) NOT NULL UNIQUE
created_at    TIMESTAMP
updated_at    TIMESTAMP
```

#### `branches`
```sql
id              UUID PK
division_id     UUID FK → divisions.id
name            VARCHAR(255) NOT NULL
created_at      TIMESTAMP
updated_at      TIMESTAMP
UNIQUE(division_id, name)
```

#### `objects`
```sql
id                   UUID PK
branch_id            UUID FK → branches.id
name                 VARCHAR(500) NOT NULL   -- address/description
responsible_engineer VARCHAR(255)
import_seq_no        INTEGER                 -- original row № from Excel
created_at           TIMESTAMP
updated_at           TIMESTAMP
```

#### `equipment_os` (Security Alarm)
```sql
id                        UUID PK
object_id                 UUID FK → objects.id UNIQUE
galaxy_512_gd520          DECIMAL(10,2) DEFAULT 0
maestro_ppk               DECIMAL(10,2) DEFAULT 0
a6_alarm                  DECIMAL(10,2) DEFAULT 0
a16_512                   DECIMAL(10,2) DEFAULT 0
vpu_a16                   DECIMAL(10,2) DEFAULT 0
access_device             DECIMAL(10,2) DEFAULT 0
signal_loops              DECIMAL(10,2) DEFAULT 0
reading_channels          DECIMAL(10,2) DEFAULT 0
detectors_sirens          DECIMAL(10,2) DEFAULT 0
galaxy_512_controller     DECIMAL(10,2) DEFAULT 0
expander                  DECIMAL(10,2) DEFAULT 0
system_controller         DECIMAL(10,2) DEFAULT 0
pcn_system_unit           DECIMAL(10,2) DEFAULT 0
bbp_brp                   DECIMAL(10,2) DEFAULT 0
updated_at                TIMESTAMP
```

#### `equipment_ps` (Fire Alarm)
```sql
id                        UUID PK
object_id                 UUID FK → objects.id UNIQUE
a6_alarm                  DECIMAL(10,2) DEFAULT 0
a16_512                   DECIMAL(10,2) DEFAULT 0
spi_molnia                DECIMAL(10,2) DEFAULT 0
am200_notifier            DECIMAL(10,2) DEFAULT 0
signal_loops              DECIMAL(10,2) DEFAULT 0
reading_channels          DECIMAL(10,2) DEFAULT 0
detectors_sirens          DECIMAL(10,2) DEFAULT 0
signs                     DECIMAL(10,2) DEFAULT 0
galaxy_512_controller     DECIMAL(10,2) DEFAULT 0
oracle                    DECIMAL(10,2) DEFAULT 0
tango_pu_bp               DECIMAL(10,2) DEFAULT 0
tango_pu_zk               DECIMAL(10,2) DEFAULT 0
expander                  DECIMAL(10,2) DEFAULT 0
address_module            DECIMAL(10,2) DEFAULT 0
address_relay_module      DECIMAL(10,2) DEFAULT 0
line_amplifier_ult        DECIMAL(10,2) DEFAULT 0
address_detectors         DECIMAL(10,2) DEFAULT 0
speakers                  DECIMAL(10,2) DEFAULT 0
updated_at                TIMESTAMP
```

#### `equipment_video`
```sql
id              UUID PK
object_id       UUID FK → objects.id UNIQUE
cameras         DECIMAL(10,2) DEFAULT 0
microphones     DECIMAL(10,2) DEFAULT 0
video_server    DECIMAL(10,2) DEFAULT 0
updated_at      TIMESTAMP
```

#### `records_tasks`
```sql
id                    UUID PK
object_id             UUID FK → objects.id UNIQUE
access_requests       DECIMAL(10,2) DEFAULT 0
monitoring_requests   DECIMAL(10,2) DEFAULT 0
footage_requests      DECIMAL(10,2) DEFAULT 0
backup_control        DECIMAL(10,2) DEFAULT 0
security_admin        DECIMAL(10,2) DEFAULT 0
updated_at            TIMESTAMP
```

#### `repairs`
```sql
id                         UUID PK
object_id                  UUID FK → objects.id UNIQUE
replace_pkp_a6_os          INTEGER DEFAULT 0
replace_pkp_a6_ps          INTEGER DEFAULT 0
replace_detector_security  INTEGER DEFAULT 0
replace_detector_fire      INTEGER DEFAULT 0
replace_resistors_os       INTEGER DEFAULT 0
replace_resistors_ps       INTEGER DEFAULT 0
replace_bbp_os             INTEGER DEFAULT 0
replace_bbp_ps             INTEGER DEFAULT 0
replace_battery_os         INTEGER DEFAULT 0
replace_battery_ps         INTEGER DEFAULT 0
replace_psu_video          INTEGER DEFAULT 0
replace_hdd_video          INTEGER DEFAULT 0
replace_components_video   INTEGER DEFAULT 0
reinstall_software_video   INTEGER DEFAULT 0
restore_ip_camera          INTEGER DEFAULT 0
restore_analog_camera      INTEGER DEFAULT 0
act_os                     INTEGER DEFAULT 0
defect_act_os              INTEGER DEFAULT 0
writeoff_act_os            INTEGER DEFAULT 0
act_ps                     INTEGER DEFAULT 0
defect_act_ps              INTEGER DEFAULT 0
writeoff_act_ps            INTEGER DEFAULT 0
act_video                  INTEGER DEFAULT 0
defect_act_video           INTEGER DEFAULT 0
writeoff_act_video         INTEGER DEFAULT 0
-- total_repairs computed: SUM of all above fields
updated_at                 TIMESTAMP
```

#### `travel`
```sql
id                  UUID PK
object_id           UUID FK → objects.id UNIQUE
transport_type      VARCHAR(100)
distance_km         DECIMAL(8,2) DEFAULT 0
one_way_time_min    DECIMAL(8,2) DEFAULT 0
-- round_trip_min computed: one_way_time_min * 2
updated_at          TIMESTAMP
```

#### `normatives`
```sql
id            UUID PK
name          VARCHAR(255) NOT NULL
category      ENUM('ПС','ОС','Видео','Записи','Ремонт') NOT NULL
field_key     VARCHAR(100) NOT NULL UNIQUE  -- maps to equipment table column
r1_minutes    DECIMAL(10,4) NOT NULL
r2_minutes    DECIMAL(10,4)               -- NULL for Записи/Ремонт (single rate)
updated_at    TIMESTAMP
```

#### `summaries` (computed, cached)
```sql
id                          UUID PK
object_id                   UUID FK → objects.id UNIQUE
pzv_minutes                 DECIMAL(10,4)  -- fixed 20
travel_minutes              DECIMAL(10,4)
ps_minutes                  DECIMAL(10,4)
video_minutes               DECIMAL(10,4)
os_minutes                  DECIMAL(10,4)
records_minutes             DECIMAL(10,4)
repair_no_travel_minutes    DECIMAL(10,4)
total_no_travel_minutes     DECIMAL(10,4)
itogo_chislo_no_travel      DECIMAL(12,6)
repair_with_travel_minutes  DECIMAL(10,4)
total_with_travel_minutes   DECIMAL(10,4)
itogo_chislo_with_travel    DECIMAL(12,6)
r1_total_minutes            DECIMAL(10,4)
r2_total_minutes            DECIMAL(10,4)
computed_at                 TIMESTAMP
```

#### `users`
```sql
id            UUID PK
email         VARCHAR(255) UNIQUE NOT NULL
name          VARCHAR(255)
role          ENUM('admin','editor','viewer') NOT NULL DEFAULT 'viewer'
division_id   UUID FK → divisions.id NULL  -- scope restriction for editors
password_hash VARCHAR(255)
created_at    TIMESTAMP
```

---

## 6. Calculation Engine

All calculations must be performed server-side. The `summaries` table is a cache invalidated on any change to equipment, records, repairs, travel, or normatives for that object.

### 6.1 Per-System Maintenance Time

For each equipment category, the maintenance time (in minutes) is:

```
system_time = SUM(equipment_quantity[i] × normative_r1[i])   for all equipment types i
```

This gives:
- `ps_minutes` — fire alarm ТО time
- `os_minutes` — security alarm ТО time
- `video_minutes` — video surveillance ТО time

### 6.2 Records / Administration Time

```
records_minutes = SUM(task_quantity[j] × normative_records[j])   for all tasks j
```

### 6.3 Repair Time (without travel)

```
repair_no_travel = SUM(repair_count[k] × normative_repair[k])   for all repair types k
```

### 6.4 Travel Time

```
round_trip_min = one_way_time_min × 2
```

Repair-with-travel adds one round trip per repair visit:
```
repair_with_travel = repair_no_travel + (total_repairs × round_trip_min)
```

where `total_repairs = SUM(all repair_count fields)`.

### 6.5 Monthly Total Time (two variants)

```
pzv = 20  (fixed)

total_no_travel_min = pzv + round_trip_min + ps_minutes + os_minutes
                    + video_minutes + records_minutes + repair_no_travel

total_with_travel_min = pzv + round_trip_min + ps_minutes + os_minutes
                      + video_minutes + records_minutes + repair_with_travel
```

### 6.6 Headcount Coefficient (ИТОГО Числ)

Monthly working time basis = **7,647 minutes/month**  
(derived from source: 247.3 min total → 0.0323 coefficient ≈ 247.3 / 7647)

```
itogo_chislo_no_travel   = total_no_travel_min / 7647
itogo_chislo_with_travel = total_with_travel_min / 7647
```

> **Important:** The exact working-time divisor (7,647 min/month) should be stored as an application-level configuration constant, editable by admins. Do not hardcode it in business logic.

### 6.7 R1 / R2 Object Totals

```
r1_total = ps_minutes(R1) + os_minutes(R1) + video_minutes(R1)
r2_total = ps_minutes(R2) + os_minutes(R2) + video_minutes(R2)
```

R1 and R2 calculations use separate normative values from the `normatives` table.

### 6.8 Division-Level Aggregation

```
division_headcount = SUM(itogo_chislo_with_travel)   for all objects in division
```

---

## 7. User Interface Requirements

### 7.1 Pages / Views

| Route | View | Description |
|---|---|---|
| `/` | Dashboard | Summary cards by division; top objects by workload |
| `/divisions` | Division List | List/search all divisions |
| `/divisions/:id` | Division Detail | All objects in division with СВОД table |
| `/objects` | Object List | Searchable, filterable table of all objects |
| `/objects/new` | Object Create | Form to create new object + equipment |
| `/objects/:id` | Object Detail | All data tabs for one object |
| `/objects/:id/edit` | Object Edit | Edit all data for one object |
| `/svod` | СВОД (Summary) | Full СВОД table with filters and export |
| `/normatives` | Normatives | View/edit time standards (admin only) |
| `/import` | Import | XLSX import wizard |
| `/export` | Export | Export options |
| `/users` | Users | User management (admin only) |

### 7.2 Object Detail Page — Tabs
The object detail page must have the following tabs, matching the Excel sheet structure:
1. **ОС** — Security alarm equipment quantities
2. **ПС** — Fire alarm equipment quantities
3. **Видео** — Video surveillance equipment quantities
4. **Записи** — Records/admin task counts
5. **Ремонт** — Repair operation counts
6. **Дорога** — Travel data
7. **СВОД** — Computed summary (read-only, auto-refreshed)

### 7.3 СВОД Table Columns
The СВОД table must reproduce the following columns:
1. №
2. Подразделение
3. Филиал
4. Значение (object name/address)
5. Ответственные ТО
6. ПЗВ (fixed 20 min)
7. Дорога (travel round-trip min)
8. Пожарная сигнализация (ps_minutes)
9. Видео (video_minutes)
10. Охрана (os_minutes)
11. Записи (records_minutes)
12. Ремонт без дороги
13. ТО + записи + ремонт (без дороги) + Дорога в месяц, мин
14. ИТОГО Числ (без дороги)
15. Ремонт с дорогой
16. ТО + записи + ремонт (с дорогой) + Дорога в месяц, мин
17. ИТОГО Числ (с дорогой)
18. Р1 на объекте всех систем
19. Р2 на объекте всех систем

### 7.4 UI/UX Constraints
- All user-facing labels must be in **Russian**.
- Equipment field names must match the original Excel headers exactly (for traceability).
- The СВОД table must be sortable by any column and filterable by division/branch/engineer.
- Input fields for quantities must only accept non-negative numbers; integer fields for repairs, decimal for equipment quantities.
- Cells with zero values may display as blank (matching Excel behavior) but must store 0.
- Inline editing of СВОД is not permitted; changes must go through the Object Detail form.

---

## 8. Non-Functional Requirements

### 8.1 Performance
- СВОД page for all 2,935 objects must load in under 3 seconds (server-side pagination allowed: 100 rows/page default).
- Recalculation of a single object's summary must complete in under 200ms.
- Bulk recalculation of all objects must complete within 60 seconds (background job acceptable).

### 8.2 Scalability
- System must support up to 10,000 objects without architectural changes.
- Support up to 50 concurrent users.

### 8.3 Security
- Authentication required for all routes except `/login`.
- HTTPS only in production.
- CSRF protection on all state-changing endpoints.
- Input validation and sanitization on all numeric fields.
- Normative edits must be logged in an audit trail (who changed what and when).

### 8.4 Data Integrity
- All `NULL` values for equipment quantities must be treated as `0` in calculations.
- `summaries` table must be invalidated (and recomputed) whenever any source data for that object changes.
- Normative changes must trigger bulk recalculation of all summaries (background queue).

### 8.5 Accessibility
- Web interface must be usable in Chrome, Firefox, Edge (latest two versions each).
- Minimum target resolution: 1280×768.

### 8.6 Availability
- Uptime target: 99% during business hours (08:00–20:00 local time, Mon–Fri).

---

## 9. Architecture Constraints

### 9.1 Recommended Stack
| Layer | Recommendation | Rationale |
|---|---|---|
| Frontend | React + TypeScript | Component reuse for tab-based object form |
| State management | React Query / TanStack Query | Server-state caching; summaries invalidation |
| Backend | Node.js (Express or Fastify) **or** Python (FastAPI) | REST API; background jobs |
| Database | PostgreSQL | Decimal precision; UUID support; JSON if needed |
| Background jobs | BullMQ (Node) or Celery (Python) | Async normative recalculation |
| XLSX processing | SheetJS (JS) or openpyxl (Python) | Import/export matching source template |
| Auth | JWT + refresh tokens or session-based | Stateless API support |
| Deployment | Docker Compose (dev) / Kubernetes or VPS (prod) | |

### 9.2 Architectural Decisions

**AD-01: Computed columns must not be stored as raw user inputs.**  
`round_trip_min`, `total_repairs`, and all columns in `summaries` are always derived. They must never be directly editable by users.

**AD-02: Normatives are a first-class configuration entity.**  
Time standards must live in the database (`normatives` table), not hardcoded. The application must work correctly after admin edits to normatives without any code deployment.

**AD-03: Calculations are server-side only.**  
No workload calculations are performed in the browser. The frontend only displays values returned by the API. This ensures consistency and prevents drift from frontend bugs.

**AD-04: The СВОД is a materialized view (cached).**  
Given 2,935+ rows and complex aggregations, `summaries` is precomputed and stored. It is invalidated on write and recalculated asynchronously. The UI should show a "stale" indicator if `computed_at < updated_at` on any source table for that object.

**AD-05: Monthly working time constant is configurable.**  
The divisor `7647` minutes/month (see §6.6) must be stored as a named application configuration value (e.g., `MONTHLY_WORKING_MINUTES`), editable by admins in the UI.

**AD-06: Equipment fields use DECIMAL, not INTEGER.**  
Some equipment entries in the source data contain decimal quantities (e.g., `2.2` signal loops). All equipment quantity fields must use `DECIMAL(10,2)`.

**AD-07: Branch and Division may be identical.**  
In the source data, `Подразделение` and `Филиал` are often the same string. The schema keeps them as separate entities but allows them to share the same name. The import process must not deduplicate them as one.

**AD-08: Separate R1 and R2 calculation paths.**  
R1 (routine inspection) and R2 (full maintenance) use different normative values. Both totals must be calculated and stored independently. The primary workload metric (ИТОГО Числ) uses R1 values for maintenance time, but R2 totals are displayed for reference.

---

## 10. API Design

### 10.1 Base URL
```
/api/v1
```

### 10.2 Endpoints

#### Objects
```
GET    /objects                  List objects (paginated, filterable)
POST   /objects                  Create object
GET    /objects/:id              Get object with all equipment data
PUT    /objects/:id              Update object metadata
DELETE /objects/:id              Delete object
GET    /objects/:id/summary      Get computed summary for object
```

#### Equipment
```
GET    /objects/:id/os           Get OS equipment
PUT    /objects/:id/os           Update OS equipment (triggers recalculation)
GET    /objects/:id/ps           Get PS equipment
PUT    /objects/:id/ps           Update PS equipment
GET    /objects/:id/video        Get Video equipment
PUT    /objects/:id/video        Update Video equipment
GET    /objects/:id/records      Get records/admin tasks
PUT    /objects/:id/records      Update records tasks
GET    /objects/:id/repairs      Get repairs
PUT    /objects/:id/repairs      Update repairs
GET    /objects/:id/travel       Get travel data
PUT    /objects/:id/travel       Update travel data
```

#### Summary / СВОД
```
GET    /svod                     Get all summaries (paginated, filterable)
POST   /svod/recalculate         Trigger bulk recalculation (admin)
GET    /svod/export/xlsx         Export СВОД to XLSX
GET    /svod/export/pdf          Export СВОД to PDF
```

#### Normatives
```
GET    /normatives               List all normatives
GET    /normatives/:id           Get single normative
PUT    /normatives/:id           Update normative value (admin; triggers bulk recalc)
```

#### Divisions & Branches
```
GET    /divisions                List divisions
GET    /divisions/:id            Division detail with aggregate headcount
GET    /branches                 List branches
```

#### Import
```
POST   /import/xlsx              Upload XLSX file; returns preview + validation report
POST   /import/xlsx/confirm      Confirm and execute import
```

#### Auth
```
POST   /auth/login
POST   /auth/logout
POST   /auth/refresh
GET    /auth/me
```

### 10.3 Response Format
All endpoints return JSON:
```json
{
  "data": { ... },
  "meta": { "page": 1, "total": 2935, "per_page": 100 },
  "error": null
}
```

On error:
```json
{
  "data": null,
  "error": { "code": "VALIDATION_ERROR", "message": "...", "fields": { ... } }
}
```

---

## 11. Migrations & Data Import

### 11.1 Initial Data Import from XLSX
The import module must:
1. Accept the source XLSX file.
2. Parse all sheets: ОС, ПС, Видео, Записи, Ремонт, Дорога.
3. Match rows across sheets by the `№` (sequence number) column — this is the cross-sheet key.
4. Create or update divisions, branches, objects, and all equipment records.
5. Skip "Расчет" sheets entirely (they are derived data).
6. Skip `Лист1` and `Лист2` (auxiliary/empty).
7. Import normatives from the `Нормативы` sheet.
8. Return a validation report: rows parsed, rows with errors, fields with empty values.
9. Recalculate all summaries after import.

### 11.2 Import Validation Rules
- `№` must be numeric and unique within the import.
- `Подразделение` and `Филиал` must not be empty.
- `Значение` (object name/address) must not be empty.
- Equipment quantities must be non-negative numbers or empty (treated as 0).
- Distances and times must be non-negative.
- Log (but do not fail on) decimal values in integer-expected fields.

### 11.3 Database Migrations
Use a migration tool (e.g., Flyway, Liquibase, Alembic, or `knex migrate`). All schema changes must be versioned. Never alter production schema manually.

---

## 12. Roles & Permissions

| Permission | Admin | Editor | Viewer |
|---|---|---|---|
| View all data | ✅ | ✅ | ✅ |
| Edit object data | ✅ | ✅ (own division) | ❌ |
| Create/delete objects | ✅ | ❌ | ❌ |
| Edit normatives | ✅ | ❌ | ❌ |
| Import XLSX | ✅ | ❌ | ❌ |
| Export XLSX/PDF | ✅ | ✅ | ✅ |
| Manage users | ✅ | ❌ | ❌ |
| View audit log | ✅ | ❌ | ❌ |

**Editor scope:** An editor's `division_id` field restricts them to only editing objects belonging to their assigned division.

---

## 13. Structural Clarifications & Corrections

The following discrepancies and ambiguities were identified in the source XLSX and must be resolved in the web application:

### C-01: Duplicate Normative Values for "Записи"
The `Нормативы` sheet contains two sections with values for records/administration tasks:
- **Row 20** lists: 60, 180, 20, 3.15, 60 (minutes)
- **Rows 73–77** list: 60, 180, 180, 120, 60 (minutes)

The values differ for "Запросы по предоставлению записей" (20 vs 180 min) and "Контроль резервного копирования" (3.15 vs 120 min). **Resolution:** Use rows 73–77 as the authoritative source. Row 20 appears to be an older/draft version and should be ignored.

### C-02: "Шлейфы сигнализации" R1 Discrepancy
In the structured normative table (rows 37+), `Шлейфы сигнализации` for both ОС and ПС shows R1 = `0.06`. However, the compact table (rows 3–4) shows `0.2` for ПС шлейфы. **Resolution:** Use `0.06` from the detailed table (rows 42, 65) as the authoritative value.

### C-03: "К-во ремонтов" Is Computed
The column "К-во ремонтов" (repair count) in the Ремонт sheet is the SUM of all individual repair quantities. In the web application, this must be a computed/derived field — never a user input. It must be recalculated on every save of the repairs form.

### C-04: Round-Trip Travel Time Is Computed
"Время в пути в обе стороны в МИНУТАХ" = `one_way_time × 2`. This is always derived from the one-way time and must not be a separate user input.

### C-05: "Расчет" Sheets Are Derived
All sheets suffixed "Расчет" (ОС Расчет, ПС Расчет, Видео Расчет, Записи Расчет, Ремонт Расчет) contain equipment quantities multiplied by normative values. In the web app, these calculations are performed by the server; there are no "Расчет" views. Their content is reflected in the СВОД and individual summary fields.

### C-06: Р1 vs R1 Naming
The source document uses both "Р1" (Cyrillic) and "R1" (Latin) in different places. Internally, the application must use a consistent field name (`r1_minutes`, `r2_minutes`). The UI must display "Р1" and "Р2" (Cyrillic) to match user expectations.

### C-07: Monthly Working Minutes Constant
The headcount divisor is not explicitly stated in the source file but is reverse-engineered from the data:  
`247.3 min / 0.03232... coefficient ≈ 7,647 minutes/month`.  
This corresponds to approximately **127.45 working hours/month** (roughly 22 working days × 5.8 hours, possibly reflecting partial-day field work). This value must be confirmed with the business owner before go-live and stored as the `MONTHLY_WORKING_MINUTES` configuration constant.

### C-08: Empty Лист1 and Лист2
`Лист2` is completely empty. `Лист1` contains a pivot table (likely auto-generated by Excel from СВОД). Neither sheet needs to be replicated in the web app. The dashboard fulfills the same purpose as `Лист1`.

### C-09: Identical Division and Branch Names
In the majority of rows, `Подразделение` and `Филиал` hold the same string value. The import must preserve both as separate data points. The UI should show both fields but allow them to be equal. Do not attempt to normalize or deduplicate them.

### C-10: Decimal Quantities in Integer-Seeming Fields
Signal loops (`Шлейфы сигнализации`) occasionally appear with decimal values in the "Расчет" sheets (e.g., `2.2`). This is likely a calculation artifact from earlier normative versions. The web app must store all equipment quantities as `DECIMAL(10,2)` and accept decimal input without validation errors.

---

## 14. Acceptance Criteria

### AC-01: Data Completeness
Import of `Шаблон_нагрузки_з_v_4_00.xlsx` must produce exactly 2,935 object records with no data loss from the ОС, ПС, Видео, Записи, Ремонт, and Дорога sheets.

### AC-02: Calculation Accuracy
For any given object, the computed СВОД values (ps_minutes, os_minutes, video_minutes, records_minutes, repair times, ИТОГО Числ) must match the values in the source XLSX "Расчет" sheets within a tolerance of ±0.001.

### AC-03: Normative Editability
After an admin changes a normative value, the СВОД values for all affected objects must update (via background recalculation) without any code deployment. The UI must show a "recalculating" indicator during this process.

### AC-04: Export Fidelity
XLSX export of СВОД must produce a file whose column structure and values match the original template's СВОД sheet within ±0.001 for all numeric values.

### AC-05: Role Enforcement
An editor assigned to Division A must be unable to edit data for any object in Division B (verified by API-level tests).

### AC-06: Computed Fields Cannot Be Manually Set
API endpoints for `PUT /objects/:id/repairs`, `PUT /objects/:id/travel`, and `PUT /svod` must reject any attempt to directly set computed fields (`total_repairs`, `round_trip_min`, `summaries.*`). These fields must be ignored if included in the request body.

### AC-07: Performance
Loading the СВОД page (first 100 rows) with all 2,935 objects imported must complete in under 3 seconds under normal load.

---

*End of Technical Specification*
