# Municipal HRMS: Expanded Specification Pack

This document provides expanded deliverables requested:
- Full OpenAPI-style specification (core modules).
- Database migration scripts (PostgreSQL) aligned to the ERD.
- Wireframe mockups (text-based) for key user flows.
- Production-grade data governance and security policy.

---

## System Overview

The municipal HRMS supports 500–2,000 employees across departments (administration, public works, fire/police, etc.) with a secure, mobile-responsive experience. It enforces RBAC across roles, integrates with identity providers, and applies civil service compliance rules with auditable workflows.

```mermaid
flowchart LR
  subgraph Users
    A[Employees\n(Admin, Public Works, Fire/Police)]
    M[Mobile Devices]
  end

  A -->|HTTPS| FE[React Web App]
  M -->|HTTPS| FE

  FE -->|API| BE[Backend Services\n(Node.js + Django)]
  BE -->|RBAC/AuthZ| IAM[Identity & Access\n(SSO/MFA/Directory)]
  BE -->|Data Access| DB[(PostgreSQL)]

  BE -->|Audit Logs| LOG[Immutable Audit Store]
  BE -->|Compliance Rules| GOV[Policy Engine\n(Civil Service Rules)]

  classDef secure fill:#e6f7ff,stroke:#1890ff,stroke-width:1px;
  classDef data fill:#fff7e6,stroke:#fa8c16,stroke-width:1px;
  classDef policy fill:#f6ffed,stroke:#52c41a,stroke-width:1px;

  class FE,BE secure;
  class DB data;
  class GOV policy;
```

**Tech Stack**
- Frontend: React (responsive UI for desktop and mobile)
- Backend: Node.js + Django (API services, business logic, integrations)
- Database: PostgreSQL (transactional HR data)

**Key Principles**
- Secure by default: TLS, MFA, least-privilege RBAC, audit logging
- Mobile-responsive UX for field and desk users
- Role-based access control aligned to municipal job classes
- Compliance with civil service laws, retention rules, and auditability

## 1. Expanded OpenAPI Specification (Core Modules)

**Base URL:** `/api/v1`  
**Auth:** OAuth2 + JWT (Bearer)

### 1.1 Authentication

```
POST /auth/login
Request:
{
  "username": "string",
  "password": "string"
}
Response 200:
{
  "token": "jwt",
  "expires_in": 3600,
  "roles": ["HR", "MANAGER"]
}
```

```
GET /auth/me
Response 200:
{
  "id": "uuid",
  "name": "string",
  "email": "string",
  "roles": ["HR"]
}
```

### 1.2 Employees

```
GET /employees
Query Params:
  departmentId, status, search
Response 200:
[
  {
    "id": "uuid",
    "first_name": "string",
    "last_name": "string",
    "email": "string",
    "status": "active"
  }
]
```

```
POST /employees
Request:
{
  "first_name": "string",
  "last_name": "string",
  "email": "string",
  "hire_date": "YYYY-MM-DD",
  "department_id": "uuid",
  "role_id": "uuid"
}
Response 201:
{
  "id": "uuid",
  "first_name": "string",
  "last_name": "string"
}
```

```
PUT /employees/{id}
Request:
{
  "status": "terminated",
  "termination_date": "YYYY-MM-DD"
}
```

```
DELETE /employees/{id}
Response 204
```

### 1.3 Recruitment

```
POST /jobs
Request:
{
  "title": "string",
  "department_id": "uuid",
  "description": "string",
  "open_date": "YYYY-MM-DD",
  "close_date": "YYYY-MM-DD"
}
```

```
GET /jobs
Query Params: status, departmentId
```

```
POST /applications
Request:
{
  "job_id": "uuid",
  "candidate_name": "string",
  "email": "string",
  "resume_url": "string"
}
```

```
PUT /applications/{id}/rank
Request:
{
  "exam_score": 92,
  "ranking": 1
}
```

### 1.4 Time & Attendance

```
POST /attendance/clock-in
Request:
{
  "employee_id": "uuid",
  "timestamp": "ISO8601",
  "lat": 0,
  "lng": 0
}
```

```
POST /attendance/clock-out
Request:
{
  "employee_id": "uuid",
  "timestamp": "ISO8601"
}
```

```
POST /leave-requests
Request:
{
  "employee_id": "uuid",
  "leave_type": "vacation",
  "start_date": "YYYY-MM-DD",
  "end_date": "YYYY-MM-DD"
}
```

```
PUT /leave-requests/{id}/approve
Request:
{
  "approved_by": "uuid",
  "status": "approved"
}
```

### 1.5 Payroll & Benefits

```
POST /payroll/sync
Request:
{
  "period_start": "YYYY-MM-DD",
  "period_end": "YYYY-MM-DD"
}
```

```
GET /payroll/{employeeId}
```

```
POST /benefits/enrollments
Request:
{
  "employee_id": "uuid",
  "plan_id": "uuid",
  "effective_date": "YYYY-MM-DD"
}
```

### 1.6 Performance & Training

```
POST /performance-reviews
Request:
{
  "employee_id": "uuid",
  "cycle": "2024",
  "rating": "meets_expectations"
}
```

```
POST /training-records
Request:
{
  "employee_id": "uuid",
  "course_name": "OSHA 10",
  "completed_on": "YYYY-MM-DD",
  "expires_on": "YYYY-MM-DD"
}
```

### 1.7 Compliance & Audit

```
GET /audit-logs
Query Params: entity, entity_id, actor_id, date_from, date_to
```

```
GET /reports/eeo-1
Query Params: year
```

```
GET /reports/turnover
Query Params: year
```

---

## 2. Database Migration Scripts (PostgreSQL)

**Filename convention:** `YYYYMMDDHHMMSS_create_tables.sql`

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE departments (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  budget_code TEXT NOT NULL
);

CREATE TABLE roles (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  title TEXT NOT NULL,
  pay_grade TEXT NOT NULL
);

CREATE TABLE employees (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  first_name TEXT NOT NULL,
  last_name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL,
  hire_date DATE NOT NULL,
  termination_date DATE,
  status TEXT NOT NULL DEFAULT 'active',
  union_code TEXT,
  department_id UUID REFERENCES departments(id),
  role_id UUID REFERENCES roles(id)
);

CREATE TABLE employee_docs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  employee_id UUID REFERENCES employees(id),
  doc_type TEXT NOT NULL,
  storage_url TEXT NOT NULL,
  signed BOOLEAN DEFAULT false,
  created_at TIMESTAMP DEFAULT now()
);

CREATE TABLE attendance_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  employee_id UUID REFERENCES employees(id),
  clock_in TIMESTAMP,
  clock_out TIMESTAMP,
  lat NUMERIC,
  lng NUMERIC
);

CREATE TABLE leave_requests (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  employee_id UUID REFERENCES employees(id),
  leave_type TEXT NOT NULL,
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending'
);

CREATE TABLE payroll_records (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  employee_id UUID REFERENCES employees(id),
  gross_pay NUMERIC(12,2),
  deductions NUMERIC(12,2),
  net_pay NUMERIC(12,2),
  period_start DATE,
  period_end DATE
);

CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  actor_id UUID REFERENCES employees(id),
  action TEXT NOT NULL,
  entity TEXT NOT NULL,
  entity_id UUID,
  created_at TIMESTAMP DEFAULT now()
);
```

---

## 3. Wireframe Mockups (Text-Based)

### 3.1 Employee Dashboard
```
+--------------------------------------------------+
| Logo | Search | Notifications | User Profile     |
+--------------------------------------------------+
| Sidebar:                                       |
| - Dashboard                                    |
| - Employees                                    |
| - Recruitment                                  |
| - Attendance                                   |
| - Payroll                                      |
| - Reports                                      |
+--------------------------------------------------+
| Main Panel:                                    |
|  KPI Cards: Headcount | Turnover | Vacancies     |
|  Org Chart Preview                             |
|  Alerts: Certification Expiry / Payroll Due    |
+--------------------------------------------------+
```

### 3.2 Employee Profile
```
+----------------- Employee Profile -----------------+
| Name | Role | Department | Status                  |
| Tabs: Profile | Documents | Benefits | Performance |
| Profile Info + Edit Button                         |
| Emergency Contacts                                |
| Skills Matrix                                     |
+----------------------------------------------------+
```

### 3.3 Leave Request Flow
```
[Calendar View]   [Leave Balance Panel]
- Select dates
- Choose leave type
- Submit
```

### 3.4 Recruitment Pipeline
```
Pipeline Columns:
Applied | Screening | Interview | Offer | Hired
Candidate Cards with score + status
```

---

## 4. Data Governance & Security Policy

### 4.1 Data Classification
- **Public:** job postings, anonymized reports.
- **Restricted:** employee records, payroll, performance reviews.
- **Confidential:** union agreements, disciplinary records.

### 4.2 Access Control
- RBAC enforced at API + database layer.
- MFA required for HR/Payroll/Admin roles.
- Least privilege access by department.

### 4.3 Audit & Compliance
- Immutable audit logs stored in separate schema.
- FOIA export pipeline with redaction for sensitive data.
- Retention schedules aligned to municipal regulations.

### 4.4 Encryption & Storage
- AES-256 at rest for DB + documents.
- TLS 1.2+ for all API traffic.
- Automatic backups with multi-region replication.

### 4.5 Data Sovereignty
- Hosting restricted to government-approved region.
- On-prem or sovereign cloud option with vendor exit strategy.

---

## 5. Expanded Deliverables (Optional Next Steps)

- Generate full OpenAPI YAML/JSON specification file.
- Produce migration files split by module with indexes.
- Add UI design kit in Figma or similar.
- Create compliance-ready SOC2/ISO templates.
