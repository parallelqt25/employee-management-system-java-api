# Employee Leave & Overtime Management API — Spring Boot (MariaDB) Take‑Home Assessment

**Goal:** Build a production‑grade **Java 17+ / Spring Boot 3.x** API for **Employee Leave** and **Overtime** management with robust policy controls, approvals, balances, ledgers, and reporting — backed by **MariaDB**.

- Use **Java 17+**, **Spring Boot 3.x**, **Spring Web**, **Spring Validation**, **Spring Data JPA**, **MariaDB**.
- Provide **precise, descriptive naming** for all identifiers (e.g., `leaveBalanceLedger`, `overtimePolicyTier`, `employeePrimaryEmailLower`).
- Publish a **public GitHub repository** with this **`README.md`** and a **Postman collection**.
- Favor **deterministic rules** and **idempotent writes** for creation/approval flows.
- Store all instants in **UTC**; apply **organization timezone** for business-day logic.

**Suggested timebox:** 6–10 hours. If you run over, submit as‑is with notes on trade‑offs.

---

## Table of Contents

1. [Functional Requirements](#functional-requirements)  
2. [Non‑Functional Requirements](#non-functional-requirements)  
3. [Tech Constraints](#tech-constraints)  
4. [Relational Data Model](#relational-data-model)  
5. [Full MariaDB DDL](#full-mariadb-ddl)  
6. [API Specification](#api-specification)  
7. [Computation Rules](#computation-rules)  
8. [Pagination, Sorting, Filtering](#pagination-sorting-filtering)  
9. [Validation, Idempotency, Concurrency](#validation-idempotency-concurrency)  
10. [Error Model](#error-model)  
11. [Caching & Performance](#caching--performance)  
12. [Authentication Using JWT (Optional)](#authentication-using-jwt-optional)  
13. [Reports (Optional)](#reports-optional)  
14. [Project Structure](#project-structure)  
15. [Configuration & Setup](#configuration--setup)  
16. [Build & Run](#build--run)  
17. [Dependencies (POM excerpt)](#dependencies-pom-excerpt)  
18. [Postman Collection](#postman-collection)  
19. [Quick cURL Tests](#quick-curl-tests)  
20. [Quality Bar & Rubric](#quality-bar--rubric)  
21. [DTO Signatures (Reference)](#dto-signatures-reference)  
22. [Service Outline (Pseudocode)](#service-outline-pseudocode)  
23. [Submission Instructions](#submission-instructions)

---

## Functional Requirements

### A) Employees & Organization
- Manage **organizations** (timezone, week start) and **employees** with manager chain.
- Each employee has: code, email (unique within org), names, employment type, FTE, default holiday calendar, and default work schedule.
- Search employees with pagination and filters.

### B) Holiday Calendars & Work Schedules
- CRUD **holiday calendars** and **holidays** (recurring annual allowed).
- CRUD **work schedules** (per‑day working hours; JSON schema accepted), bound to timezone.

### C) Leave Types & Policies
- CRUD **leave types** (e.g., Annual Leave, Sick Leave, Unpaid Leave, TOIL/Comp Time).  
- Configure **leave policies**: accrual frequency (monthly/annual/etc.), proration, carryover limit & expiry, maximum balance, negative balance limit, half‑day support.

### D) Leave Requests & Approvals
- Employee submits **leave requests** with start/end instants, units (hours/days), reason, and optional attachments (text URL placeholder ok).
- Supports **half‑day** and **partial‑day** if `unit=HOURS`.
- Check **blackout windows** and **holiday overlap**.
- **Multi‑step approvals**: ordered approvers; each step decides; request finalizes when all required steps approve; any rejection closes the request.
- On approval: **deduct** from `leaveBalanceLedger` with an atomic transaction; on rejection/cancel, ledger is not charged or is reversed if previously charged.

### E) Leave Balances & Ledger
- Maintain **`leave_balance_ledger`** for each leave type and employee: accruals, usages, adjustments, carryovers, expiries.
- Maintain **`leave_balance_summaries`** for fast reads; keep consistent in transactions.
- Admin can make **manual adjustments** with notes (credit/debit).

### F) Overtime Policies & Entries
- CRUD **overtime policies**: thresholds (daily/weekly), tiered multipliers (1.5x, 2x), weekly caps, **comp time** rules (TOIL multiplier) and payout option.
- Employees submit **overtime entries** with date/time or hours; choose **comp time** or **cash payout** (mutually exclusive).
- Multi‑step approvals for overtime similar to leave.
- On approval:  
  - If **comp time**: credit **TOIL** (a leave type) in `leave_balance_ledger` with multiplier.  
  - If **payout**: record a payout ledger note; cash payment processing itself is **out of scope** (flag stored).

### G) Accrual Job
- Admin endpoint to **run accruals** for a date (e.g., monthly close). Generates ledger accrual entries respecting policy and proration; updates summaries.

### H) Reports (Optional)
- Period summaries for **leave usage**, **overtime hours**, **balances**, **policy breaches**.

---

## Non‑Functional Requirements
- **Precise, descriptive naming** (e.g., `overtimePolicyDailyThresholdHours`, `leaveRequestIdempotencyKey`).  
- Java 17, Spring Boot 3.x; **Bean Validation** for all inputs.  
- Layers: **controller → service → repository** with DTO mapping.  
- **Transactional integrity**: approvals and ledgers must commit atomically.  
- **UTC storage**, org timezone for business rules.  
- **Structured logging** & meaningful error codes.  
- **Unit tests** for policy math; **integration tests** with MariaDB (**Testcontainers**).

---

## Tech Constraints
- Build: **Maven** (or Gradle).  
- DB: **MariaDB 10.6+** (InnoDB, utf8mb4).  
- ORM: **Spring Data JPA** (Hibernate).  
- Optional cache: **Spring Cache + Caffeine**.  
- Optional JWT: `spring-boot-starter-oauth2-resource-server` or `jjwt`.

---

## Relational Data Model

Key entities:

- **organizations** → **employees** (manager link)  
- **holiday_calendars** → **holidays**  
- **work_schedules**  
- **leave_types**, **leave_type_policies**  
- **leave_blackout_windows**  
- **leave_requests** → **leave_approval_steps**  
- **leave_balance_ledger**, **leave_balance_summaries**  
- **overtime_policies**  
- **overtime_entries** → **overtime_approval_steps**  
- **idempotency_keys** (shared infra)  
- **app_users** (optional JWT)

---

## Full MariaDB DDL

> Put this file at `src/main/resources/db/migration/V1__baseline.sql` (Flyway), or run manually.

```sql
SET NAMES utf8mb4;
SET sql_mode = 'STRICT_ALL_TABLES';

-- ============================================================================
-- TABLE: organizations
-- ============================================================================
CREATE TABLE IF NOT EXISTS organizations (
  org_id                    BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_name                  VARCHAR(200)    NOT NULL,
  org_code                  VARCHAR(100)    NOT NULL,
  org_code_lower            VARCHAR(100)
    GENERATED ALWAYS AS (LOWER(TRIM(org_code))) PERSISTENT,
  default_timezone          VARCHAR(64)     NOT NULL,
  week_start_day            TINYINT         NOT NULL DEFAULT 1, -- 1=Monday
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_organizations PRIMARY KEY (org_id),
  UNIQUE KEY uk_org_code_lower (org_code_lower)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: employees
-- ============================================================================
CREATE TABLE IF NOT EXISTS employees (
  employee_id               BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_id                    BIGINT UNSIGNED NOT NULL,
  employee_code             VARCHAR(100)    NOT NULL,
  employee_code_lower       VARCHAR(100)
    GENERATED ALWAYS AS (LOWER(TRIM(employee_code))) PERSISTENT,
  primary_email             VARCHAR(320)    NOT NULL,
  primary_email_lower       VARCHAR(320)
    GENERATED ALWAYS AS (LOWER(TRIM(primary_email))) PERSISTENT,
  first_name                VARCHAR(100)    NOT NULL,
  last_name                 VARCHAR(100)    NOT NULL,
  manager_employee_id       BIGINT UNSIGNED NULL,
  employment_type           ENUM('FULL_TIME','PART_TIME','CONTRACTOR','INTERN') NOT NULL DEFAULT 'FULL_TIME',
  fte                       DECIMAL(5,2)    NOT NULL DEFAULT 1.00,
  hire_date                 DATE            NOT NULL,
  termination_date          DATE            NULL,
  default_holiday_calendar_id BIGINT UNSIGNED NULL,
  default_work_schedule_id  BIGINT UNSIGNED NULL,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_employees PRIMARY KEY (employee_id),
  CONSTRAINT fk_emp_org FOREIGN KEY (org_id) REFERENCES organizations(org_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_emp_manager FOREIGN KEY (manager_employee_id) REFERENCES employees(employee_id)
    ON DELETE SET NULL ON UPDATE CASCADE,
  UNIQUE KEY uk_emp_code (org_id, employee_code_lower),
  UNIQUE KEY uk_emp_email (org_id, primary_email_lower),
  KEY idx_emp_org (org_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: holiday_calendars & holidays
-- ============================================================================
CREATE TABLE IF NOT EXISTS holiday_calendars (
  holiday_calendar_id       BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_id                    BIGINT UNSIGNED NOT NULL,
  calendar_name             VARCHAR(150)    NOT NULL,
  is_default                TINYINT(1)      NOT NULL DEFAULT 0,
  timezone                  VARCHAR(64)     NULL,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_holiday_calendars PRIMARY KEY (holiday_calendar_id),
  CONSTRAINT fk_hc_org FOREIGN KEY (org_id) REFERENCES organizations(org_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  UNIQUE KEY uk_hc_name (org_id, calendar_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS holidays (
  holiday_id                BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  holiday_calendar_id       BIGINT UNSIGNED NOT NULL,
  holiday_date              DATE            NOT NULL,
  holiday_name              VARCHAR(150)    NOT NULL,
  is_recurring_annual       TINYINT(1)      NOT NULL DEFAULT 0,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_holidays PRIMARY KEY (holiday_id),
  CONSTRAINT fk_holiday_calendar FOREIGN KEY (holiday_calendar_id) REFERENCES holiday_calendars(holiday_calendar_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  UNIQUE KEY uk_holiday_unique (holiday_calendar_id, holiday_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: work_schedules (JSON payload describing weekly hours)
-- ============================================================================
CREATE TABLE IF NOT EXISTS work_schedules (
  work_schedule_id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_id                    BIGINT UNSIGNED NOT NULL,
  schedule_name             VARCHAR(150)    NOT NULL,
  timezone                  VARCHAR(64)     NOT NULL,
  schedule_json             JSON            NOT NULL,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_work_schedules PRIMARY KEY (work_schedule_id),
  CONSTRAINT fk_ws_org FOREIGN KEY (org_id) REFERENCES organizations(org_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  UNIQUE KEY uk_ws_name (org_id, schedule_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: leave_types & leave_type_policies
-- ============================================================================
CREATE TABLE IF NOT EXISTS leave_types (
  leave_type_id             BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_id                    BIGINT UNSIGNED NOT NULL,
  leave_type_code           VARCHAR(50)     NOT NULL,
  leave_type_code_lower     VARCHAR(50)
    GENERATED ALWAYS AS (LOWER(TRIM(leave_type_code))) PERSISTENT,
  leave_type_name           VARCHAR(150)    NOT NULL,
  unit                      ENUM('HOURS','DAYS') NOT NULL,
  requires_approval         TINYINT(1)      NOT NULL DEFAULT 1,
  allow_negative_balance    TINYINT(1)      NOT NULL DEFAULT 0,
  allow_half_day            TINYINT(1)      NOT NULL DEFAULT 1,
  is_active                 TINYINT(1)      NOT NULL DEFAULT 1,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_leave_types PRIMARY KEY (leave_type_id),
  CONSTRAINT fk_lt_org FOREIGN KEY (org_id) REFERENCES organizations(org_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  UNIQUE KEY uk_lt_code (org_id, leave_type_code_lower)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS leave_type_policies (
  leave_policy_id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_id                    BIGINT UNSIGNED NOT NULL,
  leave_type_id             BIGINT UNSIGNED NOT NULL,
  accrual_frequency         ENUM('NONE','DAILY','WEEKLY','BIWEEKLY','MONTHLY','ANNUAL') NOT NULL DEFAULT 'ANNUAL',
  accrual_amount            DECIMAL(10,4)   NOT NULL DEFAULT 0.0000,
  accrual_proration         ENUM('NONE','CALENDAR_DAYS','WORKING_DAYS') NOT NULL DEFAULT 'NONE',
  accrual_start_basis       ENUM('HIRE_DATE','CALENDAR_YEAR','CUSTOM') NOT NULL DEFAULT 'HIRE_DATE',
  accrual_custom_start_month TINYINT        NULL, -- 1..12 when CUSTOM
  carryover_limit           DECIMAL(10,4)   NULL,
  carryover_expiry_month_day CHAR(5)        NULL, -- 'MM-DD'
  max_balance               DECIMAL(10,4)   NULL,
  negative_balance_limit    DECIMAL(10,4)   NULL,
  is_active                 TINYINT(1)      NOT NULL DEFAULT 1,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_leave_policies PRIMARY KEY (leave_policy_id),
  CONSTRAINT fk_ltp_org FOREIGN KEY (org_id) REFERENCES organizations(org_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_ltp_type FOREIGN KEY (leave_type_id) REFERENCES leave_types(leave_type_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  KEY idx_ltp_active (org_id, leave_type_id, is_active)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: leave_blackout_windows
-- ============================================================================
CREATE TABLE IF NOT EXISTS leave_blackout_windows (
  blackout_id               BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_id                    BIGINT UNSIGNED NOT NULL,
  start_datetime_utc        DATETIME(6)     NOT NULL,
  end_datetime_utc          DATETIME(6)     NOT NULL,
  reason                    VARCHAR(255)    NOT NULL,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_blackouts PRIMARY KEY (blackout_id),
  CONSTRAINT fk_blackout_org FOREIGN KEY (org_id) REFERENCES organizations(org_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  KEY idx_blackout_range (org_id, start_datetime_utc, end_datetime_utc)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: leave_requests & leave_approval_steps
-- ============================================================================
CREATE TABLE IF NOT EXISTS leave_requests (
  leave_request_id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_id                    BIGINT UNSIGNED NOT NULL,
  employee_id               BIGINT UNSIGNED NOT NULL,
  leave_type_id             BIGINT UNSIGNED NOT NULL,
  start_datetime_utc        DATETIME(6)     NOT NULL,
  end_datetime_utc          DATETIME(6)     NOT NULL,
  requested_duration_hours  DECIMAL(10,4)   NOT NULL,
  requested_duration_unit   ENUM('HOURS','DAYS') NOT NULL,
  status                    ENUM('DRAFT','PENDING','APPROVED','REJECTED','CANCELLED') NOT NULL DEFAULT 'DRAFT',
  current_approval_step     INT             NOT NULL DEFAULT 1,
  reason                    TEXT            NULL,
  cancel_reason             TEXT            NULL,
  idempotency_key           CHAR(36)        NULL,
  created_by_employee_id    BIGINT UNSIGNED NOT NULL,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_leave_requests PRIMARY KEY (leave_request_id),
  CONSTRAINT fk_lr_org FOREIGN KEY (org_id) REFERENCES organizations(org_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_lr_emp FOREIGN KEY (employee_id) REFERENCES employees(employee_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_lr_type FOREIGN KEY (leave_type_id) REFERENCES leave_types(leave_type_id)
    ON DELETE RESTRICT ON UPDATE CASCADE,
  UNIQUE KEY uk_lr_idem (org_id, created_by_employee_id, idempotency_key),
  KEY idx_lr_emp (employee_id, status, start_datetime_utc)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS leave_approval_steps (
  leave_approval_step_id    BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  leave_request_id          BIGINT UNSIGNED NOT NULL,
  sequence_number           INT             NOT NULL,
  approver_employee_id      BIGINT UNSIGNED NOT NULL,
  status                    ENUM('PENDING','APPROVED','REJECTED','SKIPPED') NOT NULL DEFAULT 'PENDING',
  decision_comment          VARCHAR(500)    NULL,
  decided_at                DATETIME(6)     NULL,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_leave_approval_steps PRIMARY KEY (leave_approval_step_id),
  CONSTRAINT fk_las_lr FOREIGN KEY (leave_request_id) REFERENCES leave_requests(leave_request_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_las_approver FOREIGN KEY (approver_employee_id) REFERENCES employees(employee_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  UNIQUE KEY uk_las_seq (leave_request_id, sequence_number)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: leave_balance_ledger & leave_balance_summaries
-- ============================================================================
CREATE TABLE IF NOT EXISTS leave_balance_ledger (
  ledger_id                 BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_id                    BIGINT UNSIGNED NOT NULL,
  employee_id               BIGINT UNSIGNED NOT NULL,
  leave_type_id             BIGINT UNSIGNED NOT NULL,
  event_type                ENUM('ACCRUAL','USAGE','ADJUSTMENT','CARRYOVER_IN','CARRYOVER_EXPIRE','ENCASHMENT','OVERTIME_TO_TOIL') NOT NULL,
  quantity                  DECIMAL(10,4)   NOT NULL, -- +credit, -debit
  balance_after             DECIMAL(10,4)   NULL,
  effective_at              DATETIME(6)     NOT NULL,
  reference_leave_request_id BIGINT UNSIGNED NULL,
  reference_overtime_entry_id BIGINT UNSIGNED NULL,
  notes                     VARCHAR(255)    NULL,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_leave_ledger PRIMARY KEY (ledger_id),
  CONSTRAINT fk_ll_org FOREIGN KEY (org_id) REFERENCES organizations(org_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_ll_emp FOREIGN KEY (employee_id) REFERENCES employees(employee_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_ll_type FOREIGN KEY (leave_type_id) REFERENCES leave_types(leave_type_id)
    ON DELETE RESTRICT ON UPDATE CASCADE,
  CONSTRAINT fk_ll_lr FOREIGN KEY (reference_leave_request_id) REFERENCES leave_requests(leave_request_id)
    ON DELETE SET NULL ON UPDATE CASCADE,
  KEY idx_ll_emp_type (employee_id, leave_type_id, effective_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS leave_balance_summaries (
  org_id                    BIGINT UNSIGNED NOT NULL,
  employee_id               BIGINT UNSIGNED NOT NULL,
  leave_type_id             BIGINT UNSIGNED NOT NULL,
  balance                   DECIMAL(10,4)   NOT NULL,
  last_updated_at           DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_leave_balance_summaries PRIMARY KEY (employee_id, leave_type_id),
  CONSTRAINT fk_lbs_org FOREIGN KEY (org_id) REFERENCES organizations(org_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_lbs_emp FOREIGN KEY (employee_id) REFERENCES employees(employee_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_lbs_type FOREIGN KEY (leave_type_id) REFERENCES leave_types(leave_type_id)
    ON DELETE RESTRICT ON UPDATE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: overtime_policies
-- ============================================================================
CREATE TABLE IF NOT EXISTS overtime_policies (
  overtime_policy_id        BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_id                    BIGINT UNSIGNED NOT NULL,
  policy_name               VARCHAR(150)    NOT NULL,
  daily_threshold_hours     DECIMAL(6,2)    NOT NULL DEFAULT 8.00,
  daily_tier1_hours         DECIMAL(6,2)    NOT NULL DEFAULT 2.00,
  daily_tier1_multiplier    DECIMAL(6,3)    NOT NULL DEFAULT 1.500,
  daily_tier2_multiplier    DECIMAL(6,3)    NOT NULL DEFAULT 2.000,
  weekly_threshold_hours    DECIMAL(6,2)    NOT NULL DEFAULT 40.00,
  weekly_max_overtime_hours DECIMAL(6,2)    NULL,
  allow_comp_time           TINYINT(1)      NOT NULL DEFAULT 0,
  comp_time_multiplier      DECIMAL(6,3)    NULL,
  is_active                 TINYINT(1)      NOT NULL DEFAULT 1,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_overtime_policies PRIMARY KEY (overtime_policy_id),
  CONSTRAINT fk_op_org FOREIGN KEY (org_id) REFERENCES organizations(org_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  UNIQUE KEY uk_policy_name (org_id, policy_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: overtime_entries & overtime_approval_steps
-- ============================================================================
CREATE TABLE IF NOT EXISTS overtime_entries (
  overtime_entry_id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  org_id                    BIGINT UNSIGNED NOT NULL,
  employee_id               BIGINT UNSIGNED NOT NULL,
  work_date                 DATE            NOT NULL,
  start_datetime_utc        DATETIME(6)     NULL,
  end_datetime_utc          DATETIME(6)     NULL,
  reported_hours            DECIMAL(10,4)   NOT NULL,
  source                    ENUM('MANUAL','CLOCK','IMPORT') NOT NULL DEFAULT 'MANUAL',
  selected_comp_time        TINYINT(1)      NOT NULL DEFAULT 0,
  selected_cash_payout      TINYINT(1)      NOT NULL DEFAULT 1,
  overtime_policy_id        BIGINT UNSIGNED NULL,
  status                    ENUM('DRAFT','SUBMITTED','APPROVED','REJECTED','CANCELLED') NOT NULL DEFAULT 'DRAFT',
  idempotency_key           CHAR(36)        NULL,
  reason                    VARCHAR(255)    NULL,
  created_by_employee_id    BIGINT UNSIGNED NOT NULL,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_overtime_entries PRIMARY KEY (overtime_entry_id),
  CONSTRAINT fk_oe_org FOREIGN KEY (org_id) REFERENCES organizations(org_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_oe_emp FOREIGN KEY (employee_id) REFERENCES employees(employee_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_oe_policy FOREIGN KEY (overtime_policy_id) REFERENCES overtime_policies(overtime_policy_id)
    ON DELETE SET NULL ON UPDATE CASCADE,
  UNIQUE KEY uk_oe_idem (org_id, created_by_employee_id, idempotency_key),
  KEY idx_oe_emp_date (employee_id, work_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE IF NOT EXISTS overtime_approval_steps (
  ot_approval_step_id       BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  overtime_entry_id         BIGINT UNSIGNED NOT NULL,
  sequence_number           INT             NOT NULL,
  approver_employee_id      BIGINT UNSIGNED NOT NULL,
  status                    ENUM('PENDING','APPROVED','REJECTED','SKIPPED') NOT NULL DEFAULT 'PENDING',
  decision_comment          VARCHAR(500)    NULL,
  decided_at                DATETIME(6)     NULL,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_ot_approval_steps PRIMARY KEY (ot_approval_step_id),
  CONSTRAINT fk_oas_oe FOREIGN KEY (overtime_entry_id) REFERENCES overtime_entries(overtime_entry_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  CONSTRAINT fk_oas_approver FOREIGN KEY (approver_employee_id) REFERENCES employees(employee_id)
    ON DELETE CASCADE ON UPDATE CASCADE,
  UNIQUE KEY uk_oas_seq (overtime_entry_id, sequence_number)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: idempotency_keys (shared infra)
-- ============================================================================
CREATE TABLE IF NOT EXISTS idempotency_keys (
  idempotency_key_id        BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  owner_scope               VARCHAR(100)    NOT NULL, -- e.g., 'leave_request:create'
  owner_key                 VARCHAR(200)    NOT NULL, -- e.g., 'org:123|emp:456'
  idempotency_key           CHAR(36)        NOT NULL,
  request_hash              CHAR(64)        NOT NULL,
  response_code             INT             NOT NULL,
  response_body             LONGBLOB        NULL,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_idempotency_keys PRIMARY KEY (idempotency_key_id),
  UNIQUE KEY uk_idem_tuple (owner_scope, owner_key, idempotency_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================================
-- TABLE: app_users (optional JWT)
-- ============================================================================
CREATE TABLE IF NOT EXISTS app_users (
  user_id                   BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  username                  VARCHAR(100)    NOT NULL,
  username_lower            VARCHAR(100)
    GENERATED ALWAYS AS (LOWER(TRIM(username))) PERSISTENT,
  password_hash             VARCHAR(100)    NOT NULL, -- BCrypt 60 chars
  display_name              VARCHAR(150)    NULL,
  role                      VARCHAR(50)     NOT NULL DEFAULT 'ADMIN',
  is_active                 TINYINT(1)      NOT NULL DEFAULT 1,
  created_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
  updated_at                DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  CONSTRAINT pk_app_users PRIMARY KEY (user_id),
  UNIQUE KEY uk_app_users_username_lower (username_lower)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## API Specification

**Base URL:** `http://localhost:8080/api`

> **Conventions**
> - Pagination via `page` (1‑based), `pageSize` (≤100).  
> - Sorting via `sortBy` and `sortDirection` (`asc|desc`).  
> - All times are ISO‑8601; **store UTC**, interpret using org timezone.  
> - Idempotency for writes: `Idempotency-Key: <uuid>` header.

### Health
```
GET /health
→ 200 { "status": "ok", "service": "leave-overtime-api", "ts": "<iso>" }
```

### Organizations
```http
POST /organizations
{ "orgName": "Acme Inc", "orgCode": "ACME", "defaultTimezone": "America/Los_Angeles", "weekStartDay": 1 }
→ 201 { orgId, ... }

GET /organizations/{orgId}
→ 200 { ... }
```

### Employees
```http
POST /organizations/{orgId}/employees
{
  "employeeCode": "E001",
  "primaryEmail": "jane.doe@acme.com",
  "firstName": "Jane",
  "lastName": "Doe",
  "managerEmployeeId": null,
  "employmentType": "FULL_TIME",
  "fte": 1.0,
  "hireDate": "2025-01-15",
  "defaultHolidayCalendarId": null,
  "defaultWorkScheduleId": null
}
→ 201 { employeeId, ... }

GET /organizations/{orgId}/employees?page=1&pageSize=20&q=doe
→ 200 { page, pageSize, totalItems, totalPages, items: [...] }
```

### Holiday Calendars & Holidays
```http
POST /organizations/{orgId}/holiday-calendars
{ "calendarName": "US-2025", "timezone": "America/Los_Angeles", "isDefault": true }
→ 201 { holidayCalendarId, ... }

POST /holiday-calendars/{holidayCalendarId}/holidays
{ "holidayDate": "2025-07-04", "holidayName": "Independence Day", "isRecurringAnnual": true }
→ 201 { holidayId, ... }
```

### Work Schedules
```http
POST /organizations/{orgId}/work-schedules
{
  "scheduleName": "Standard-9-5",
  "timezone": "America/Los_Angeles",
  "scheduleJson": {
    "mon": {"start":"09:00","end":"17:00","breakMins":60},
    "tue": {"start":"09:00","end":"17:00","breakMins":60},
    "wed": {"start":"09:00","end":"17:00","breakMins":60},
    "thu": {"start":"09:00","end":"17:00","breakMins":60},
    "fri": {"start":"09:00","end":"17:00","breakMins":60}
  }
}
→ 201 { workScheduleId, ... }
```

### Leave Types & Policies
```http
POST /organizations/{orgId}/leave-types
{
  "leaveTypeCode": "ANNUAL",
  "leaveTypeName": "Annual Leave",
  "unit": "DAYS",
  "requiresApproval": true,
  "allowNegativeBalance": false,
  "allowHalfDay": true
}
→ 201 { leaveTypeId, ... }

POST /organizations/{orgId}/leave-types/{leaveTypeId}/policies
{
  "accrualFrequency": "MONTHLY",
  "accrualAmount": 1.75,
  "accrualProration": "WORKING_DAYS",
  "accrualStartBasis": "HIRE_DATE",
  "carryoverLimit": 10,
  "carryoverExpiryMonthDay": "03-31",
  "maxBalance": 30,
  "negativeBalanceLimit": 0,
  "isActive": true
}
→ 201 { leavePolicyId, ... }
```

### Leave Blackout Windows
```http
POST /organizations/{orgId}/leave-blackouts
{
  "startDatetimeUtc": "2025-11-25T00:00:00Z",
  "endDatetimeUtc":   "2025-12-01T00:00:00Z",
  "reason": "Peak period"
}
→ 201 { blackoutId, ... }
```

### Leave Requests
```http
POST /organizations/{orgId}/leave-requests
Headers: Idempotency-Key: b5d5a9bb-...

{
  "employeeId": 101,
  "leaveTypeId": 10,
  "startDatetimeUtc": "2025-02-10T16:00:00Z",
  "endDatetimeUtc":   "2025-02-12T16:00:00Z",
  "requestedDurationUnit": "DAYS",
  "requestedDurationHours": 16.0,
  "reason": "Family trip"
}
→ 201 { leaveRequestId, status: "DRAFT", currentApprovalStep: 1, ... }

POST /leave-requests/{leaveRequestId}/submit
→ 200 { status: "PENDING" }

POST /leave-requests/{leaveRequestId}/approve
{ "approvalStepId": 555, "comment": "Approved" }
→ 200 { status: "PENDING" | "APPROVED" }  // advances step or finalizes

POST /leave-requests/{leaveRequestId}/reject
{ "approvalStepId": 555, "comment": "Not feasible" }
→ 200 { status: "REJECTED" }

POST /leave-requests/{leaveRequestId}/cancel
{ "cancelReason": "Change of plans" }
→ 200 { status: "CANCELLED" }

GET /organizations/{orgId}/leave-requests?employeeId=101&status=PENDING&page=1&pageSize=20
→ 200 { page, pageSize, totalItems, totalPages, items:[...] }
```

### Leave Balances
```http
GET /organizations/{orgId}/employees/{employeeId}/leave-balances
→ 200 [
  { "leaveTypeId": 10, "leaveTypeCode": "ANNUAL", "balance": 12.5, "unit": "DAYS" },
  { "leaveTypeId": 11, "leaveTypeCode": "SICK",   "balance": 4.0,  "unit": "DAYS" }
]

POST /organizations/{orgId}/leave-balances/adjust
{
  "employeeId": 101, "leaveTypeId": 10,
  "quantity": 2.0, "notes": "Year-end award"
}
→ 200 { "newBalance": 14.5 }
```

### Accrual Jobs
```http
POST /organizations/{orgId}/accruals/run?asOfDate=2025-12-31
→ 202 { "jobId": "acc-20251231", "summary": { "employeesProcessed": 42 } }
```

### Overtime Policies
```http
POST /organizations/{orgId}/overtime-policies
{
  "policyName": "US-Standard",
  "dailyThresholdHours": 8.0,
  "dailyTier1Hours": 2.0,
  "dailyTier1Multiplier": 1.5,
  "dailyTier2Multiplier": 2.0,
  "weeklyThresholdHours": 40.0,
  "weeklyMaxOvertimeHours": 20.0,
  "allowCompTime": true,
  "compTimeMultiplier": 1.0
}
→ 201 { overtimePolicyId, ... }
```

### Overtime Entries & Approvals
```http
POST /organizations/{orgId}/overtime-entries
Headers: Idempotency-Key: a2d3c0f1-...

{
  "employeeId": 101,
  "workDate": "2025-02-14",
  "startDatetimeUtc": "2025-02-14T02:00:00Z",
  "endDatetimeUtc":   "2025-02-14T05:00:00Z",
  "reportedHours": 3.0,
  "selectedCompTime": true,
  "selectedCashPayout": false,
  "overtimePolicyId": 5,
  "reason": "Release cutover"
}
→ 201 { overtimeEntryId, status: "DRAFT" }

POST /overtime-entries/{overtimeEntryId}/submit
→ 200 { status: "SUBMITTED" }

POST /overtime-entries/{overtimeEntryId}/approve
{ "approvalStepId": 77, "comment": "OK" }
→ 200 { status: "APPROVED" }

POST /overtime-entries/{overtimeEntryId}/reject
{ "approvalStepId": 77, "comment": "Out of window" }
→ 200 { status: "REJECTED" }
```

---

## Computation Rules

### Leave Duration
- For `unit=DAYS`: calculate using **org work schedule** and **holidays** (exclude weekends/holidays).  
- For partial days, respect `allow_half_day`; otherwise round up to nearest whole day.  
- For `unit=HOURS`: compute raw hours from start/end times minus breaks; clamp to schedule.

### Accruals
- Based on policy frequency: add `accrualAmount` per period; prorate by **WORKING_DAYS** or **CALENDAR_DAYS** as configured.  
- On carryover date (`carryoverExpiryMonthDay`), move excess to `CARRYOVER_EXPIRE` with negative ledger entries.

### Overtime
- Per entry: split hours into **Tier1** up to `dailyTier1Hours` at `tier1Multiplier` and **Tier2** remainder at `tier2Multiplier`.  
- Weekly aggregation: if weekly total exceeds `weeklyThresholdHours`, mark as overtime; enforce `weeklyMaxOvertimeHours` cap.  
- **Comp time**: credit TOIL (`OVERTIME_TO_TOIL`) = `overtimeHours * compTimeMultiplier` (usually 1.0).

---

## Pagination, Sorting, Filtering

- Default `page=1`, `pageSize=20` (max 100).  
- Sortable fields: createdAt, updatedAt, startDatetimeUtc, endDatetimeUtc, status, employeeId.  
- Filter by `status`, `employeeId`, `leaveTypeId`, `dateFrom`, `dateTo` (overlap).

---

## Validation, Idempotency, Concurrency

- **Validation:** Bean Validation (e.g., `@Email`, `@Positive`, `@PastOrPresent`). Reject illogical ranges (end before start).  
- **Idempotency:** Accept `Idempotency-Key` header for **create/submit** flows; persist responses in `idempotency_keys`.  
- **Concurrency:** Use optimistic locking (`@Version`) or ETag for mutation endpoints; re-read balances within the same transaction; prevent negative balances if disallowed.

---

## Error Model

All errors return JSON:

```json
{
  "errorCode": "VALIDATION_ERROR | NOT_FOUND | CONFLICT | UNAUTHORIZED | FORBIDDEN | INTERNAL_ERROR",
  "errorMessage": "Human-readable detail",
  "details": { "field": "leaveTypeId" }
}
```
- 400: validation
- 401/403: auth (if using JWT)
- 404: not found
- 409: uniqueness or business rule conflict (e.g., blackout)
- 422: cannot fulfill (e.g., negative balance not allowed)
- 500: unexpected

---

## Caching & Performance

- Cache **active leave types**, **policies**, **holiday calendars** for 60s (Spring Cache + Caffeine).  
- Expose `X-Cache: HIT|MISS` headers where appropriate.  
- Use **indexes** from DDL for scanning queries; paginate large lists.

---

## Authentication Using JWT (Optional)

- `POST /auth/login` with demo user (env or `app_users`) → JWT.  
- Protect mutating routes.  
- Env: `JWT_SECRET`, demo credentials or seeded user (BCrypt).

---

## Reports (Optional)

- `GET /organizations/{orgId}/reports/leave-usage?from=...&to=...`  
- `GET /organizations/{orgId}/reports/overtime-summary?from=...&to=...`  
- `GET /organizations/{orgId}/reports/leave-balance-snapshot?asOf=...`

---

## Project Structure

```text
.
├─ src/main/java/com/example/leaveovertime/
│  ├─ LeaveOvertimeApplication.java
│  ├─ api/
│  │  ├─ OrganizationController.java
│  │  ├─ EmployeeController.java
│  │  ├─ HolidayController.java
│  │  ├─ WorkScheduleController.java
│  │  ├─ LeaveTypeController.java
│  │  ├─ LeaveRequestController.java
│  │  ├─ AccrualController.java
│  │  ├─ OvertimePolicyController.java
│  │  ├─ OvertimeEntryController.java
│  │  └─ AuthController.java           // optional
│  ├─ service/
│  │  ├─ LeaveComputationService.java
│  │  ├─ LeaveLedgerService.java
│  │  ├─ LeaveWorkflowService.java
│  │  ├─ OvertimeComputationService.java
│  │  ├─ OvertimeWorkflowService.java
│  │  └─ PolicyCacheService.java
│  ├─ repository/  // Spring Data JPA repos
│  ├─ domain/      // JPA entities
│  ├─ dto/         // request/response DTOs
│  ├─ config/
│  │  ├─ CacheConfig.java
│  │  └─ SecurityConfig.java           // optional
│  ├─ util/
│  │  ├─ DateTimeUtils.java
│  │  └─ OverlapUtils.java
│  └─ middleware/
│     └─ GlobalExceptionHandler.java
├─ src/main/resources/
│  ├─ application.yml
│  └─ db/migration/V1__baseline.sql
├─ postman/Leave-Overtime.postman_collection.json
├─ pom.xml
└─ README.md
```

---

## Configuration & Setup

**MariaDB — create DB and user**

```sql
CREATE DATABASE IF NOT EXISTS leave_overtime
  DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'leave_overtime_user'@'%' IDENTIFIED BY 'changeme';
GRANT ALL PRIVILEGES ON leave_overtime.* TO 'leave_overtime_user'@'%';
FLUSH PRIVILEGES;
```

**`application.yml` (example)**

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mariadb://localhost:3306/leave_overtime?useUnicode=true&characterEncoding=utf8
    username: leave_overtime_user
    password: changeme
    driver-class-name: org.mariadb.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: validate           # Flyway manages schema
    properties:
      hibernate:
        format_sql: true
        jdbc:
          time_zone: UTC
    open-in-view: false
  flyway:
    enabled: true
    baseline-on-migrate: true

spring:
  cache:
    cache-names: policyCache,holidayCache
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=60s

app:
  keywordCacheTtlSeconds: 60
```

---

## Dependencies (POM excerpt)

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>org.mariadb.jdbc</groupId>
    <artifactId>mariadb-java-client</artifactId>
    <version>3.4.0</version>
  </dependency>
  <!-- Optional cache -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
  </dependency>
  <dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
  </dependency>
  <!-- Optional JWT stack -->
  <!--
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
  </dependency>
  -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mariadb</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

---

## Postman Collection

Include `postman/Leave-Overtime.postman_collection.json` with folders:

1. **Health** — `GET /api/health`  
2. **Organizations & Employees** — create/get/list  
3. **Holidays & Work Schedules** — CRUD  
4. **Leave Types & Policies** — CRUD  
5. **Leave Requests** — create/submit/approve/reject/cancel  
6. **Overtime Policies & Entries** — create/submit/approve/reject  
7. **Accruals** — run accruals

**Example Postman tests**
```javascript
pm.test("Status is 200", () => pm.response.to.have.status(200));
pm.test("Has items or page", () => {
  const b = pm.response.json();
  pm.expect(b.page || Array.isArray(b)).to.be.ok;
});
```

---

## Submission Instructions

1. Create a **public GitHub repository** (e.g., `leave-overtime-api-springboot`).  
2. Implement the solution following this spec.  
3. Include:
   - This **`README.md`**,
   - A **Postman collection** in `/postman`,
   - **Flyway migration** at `src/main/resources/db/migration/V1__baseline.sql`,
   - Minimal sample JSON payloads in `/samples`.
4. Verify locally:
   - `./mvnw spring-boot:run` boots the server,
   - `GET /api/health` returns `200`.
5. Share the **repo URL** and (optional) a brief note on trade‑offs.
