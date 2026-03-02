# Workload Calculation System --- Technical Specification (TOR)

Version: 3.0\
Status: Final\
Format: Master Development Specification

------------------------------------------------------------------------

## 1. Purpose

The system is a web application for automated calculation of technical
workload, labor time, and staffing requirements for engineers
maintaining technical systems across multiple branches and objects.

------------------------------------------------------------------------

## 2. Core Business Goals

-   automate workload calculations
-   centralize data storage
-   manage engineers and assignments
-   calculate staffing needs
-   generate reports
-   provide analytics dashboards

------------------------------------------------------------------------

## 3. System Hierarchy

    Branch
     └── Object
           └── System
                 └── Device
                       └── Calculation

Rules: - One branch → many objects - One object → many systems - One
system → many devices

------------------------------------------------------------------------

## 4. User Roles

### Administrator

Full system access.

### Engineer

Work with assigned objects and data entry.

### Manager

Analytics, reports, workforce monitoring.

------------------------------------------------------------------------

## 5. Branch Module

Fields: - id - name - code - status

Rules: - branch may contain unlimited objects - branch cannot be deleted
if objects exist

------------------------------------------------------------------------

## 6. Object Module

Fields: - id - name - address - branch_id (required) - responsible
engineer - status - notes

Validation: - object must belong to branch

------------------------------------------------------------------------

## 7. Systems Module

Each object may contain multiple systems.

Default system types: - Security Alarm - Fire Alarm - Video
Surveillance - Access Control - Logs - Repairs - Travel

Admin can add system types.

------------------------------------------------------------------------

## 8. Device-Level Structure (Mandatory)

Each system must contain devices.

Device fields: - device type - quantity - normative time - coefficient -
calculated time

Formula:

    device_time = quantity × normative × coefficient

------------------------------------------------------------------------

## 9. Normatives Module

Editable reference table.

Fields: - device type - system type - time value - coefficient - active
flag

System must store change history.

------------------------------------------------------------------------

## 10. Calculations

### System

    system_time = sum(device_times)

### Object

    object_time = sum(system_times) + travel + preparation

### Branch

    branch_time = sum(object_times)

### Staffing

    required_staff = total_minutes ÷ monthly_norm

------------------------------------------------------------------------

## 11. Travel Module

Fields: - distance - time per trip - trips per month

    travel_total = trips × time_per_trip

------------------------------------------------------------------------

## 12. Repair Module

Fields: - repair type - quantity - prep time - execution time - travel
time

------------------------------------------------------------------------

## 13. Engineer Module

Engineer fields: - id - name - branch - monthly_norm -
max_load_percent - status

------------------------------------------------------------------------

### Assignment Table

Fields: - engineer_id - object_id - role - start_date - end_date

Rules: - engineer can have many objects - object can have many
engineers - history must be stored

------------------------------------------------------------------------

### Engineer Workload

    engineer_load = sum(assigned object loads)

### Utilization

    utilization = engineer_load ÷ monthly_norm × 100

Status thresholds:

  Load       Status
  ---------- -------------
  0--60      underloaded
  60--100    normal
  100--120   overloaded
  \>120      critical

------------------------------------------------------------------------

## 14. Dashboards

### Global

-   total workload
-   required engineers
-   branch comparison

### Branch

-   object count
-   total workload
-   staff demand

### Engineer

-   assigned objects
-   utilization
-   load graph

------------------------------------------------------------------------

## 15. Reports

System must generate:

-   branch workload
-   object workload
-   engineer workload
-   device breakdown

Export: - XLSX - PDF - CSV

------------------------------------------------------------------------

## 16. Validation Rules

-   numeric ≥ 0
-   quantity integer
-   duplicate devices forbidden
-   norm must exist
-   inactive engineer cannot be assigned

------------------------------------------------------------------------

## 17. Performance Requirements

-   calculation ≤ 2 sec
-   page load ≤ 1.5 sec
-   support 10k objects per branch
-   support 100 systems per object
-   support 500 devices per object
-   support 1000 users

------------------------------------------------------------------------

## 18. Security

Required: - HTTPS - role permissions - password hashing - session
timeout - audit logs

------------------------------------------------------------------------

## 19. Database Core Tables

-   Users
-   Branches
-   Objects
-   Systems
-   Devices
-   Normatives
-   Engineers
-   Assignments
-   Calculations
-   AuditLogs

------------------------------------------------------------------------

## 20. API Endpoints

    GET /objects
    POST /objects
    PUT /objects/{id}
    DELETE /objects/{id}

    GET /branches
    POST /branches

    GET /systems/{object}
    POST /systems

    GET /devices/{system}
    POST /devices

    GET /engineers
    POST /engineers

    GET /calculations/{object}

------------------------------------------------------------------------

## 21. Deployment

Must support: - dev environment - staging - production - CI/CD pipeline

------------------------------------------------------------------------

## 22. Acceptance Criteria

System accepted only if:

-   calculations match Excel results
-   workload updates instantly
-   engineer utilization correct
-   reports generate
-   permissions enforced
-   100 concurrent users stable

------------------------------------------------------------------------

## 23. Future Extensions

Optional modules: - mobile app - predictive analytics - HR integration -
AI workload planner

------------------------------------------------------------------------

# END OF SPECIFICATION
