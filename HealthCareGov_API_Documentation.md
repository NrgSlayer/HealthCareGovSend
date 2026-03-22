# HealthCareGov — Backend API Documentation

**Version:** 1.0.0 | **Spring Boot:** 3.3.5 | **Java:** 17

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Tech Stack](#2-tech-stack)
3. [Architecture](#3-architecture)
4. [Implemented Features](#4-implemented-features)
5. [Authentication & Authorization](#5-authentication--authorization)
6. [Running the Project Locally](#6-running-the-project-locally)
7. [API Reference](#7-api-reference)
   - [1. User Management](#71-user-management)
   - [2. Patient Registration & Profile](#72-patient-registration--profile)
   - [3. Hospital Management](#73-hospital-management)
   - [4. Resource Management](#74-resource-management)
   - [5. Analytics & Reports](#75-analytics--reports)
   - [6. Doctor Schedule](#76-doctor-schedule)
   - [7. Appointments](#77-appointments)
   - [8. Treatment & Medical Records](#78-treatment--medical-records)
   - [9. Compliance](#79-compliance)
   - [10. Audit](#710-audit)
   - [11. Notifications](#711-notifications)
   - [12. Program Dashboard & Reports](#712-program-dashboard--reports)
8. [Error Responses](#8-error-responses)
9. [Validation Reference](#9-validation-reference)

---

## 1. Project Overview

**HealthCareGov** is a REST API backend for a public healthcare and hospital management system. It enables:

- Citizens to register, book appointments, and access their medical history
- Doctors to manage schedules, record treatments, and update patient records
- Hospital Administrators to manage facilities, resources, and user accounts
- Compliance Officers to track policy adherence and generate audit records
- Government Auditors to review compliance findings
- Program Managers to monitor healthcare program performance through dashboards and reports

The system enforces strict role-based access control using JWT tokens. All passwords are stored using BCrypt hashing.

---

## 2. Tech Stack

| Layer | Technology |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.3.5 |
| Security | Spring Security 6 + JWT (jjwt 0.12.6) |
| Persistence | Spring Data JPA + Hibernate |
| Database | MySQL 8 |
| Validation | Jakarta Bean Validation |
| Boilerplate | Lombok |
| Build | Maven |
| Password Hashing | BCrypt |

---

## 3. Architecture

```
Client Request
     │
     ▼
JwtAuthenticationFilter  ←── validates Bearer token, sets SecurityContext
     │
     ▼
SecurityFilterChain  ←── enforces URL-level role rules
     │
     ▼
Controller  ←── receives DTOs, delegates to Service
     │
     ▼
Service  ←── business logic, ownership checks, calls Repository
     │
     ▼
Repository (JPA)  ←── database operations
     │
     ▼
MySQL Database
```

**Key design rules:**
- Controllers never return entity objects — only DTOs
- All business logic and ownership verification live in the Service layer
- `SecurityUtils` reads the authenticated caller from `SecurityContext` for ownership checks
- `AuditLogService` uses `REQUIRES_NEW` propagation to always persist audit entries independently

---

## 4. Implemented Features

| Backlog ID | Feature | Role |
|---|---|---|
| USER-001 | User Login (JWT issued on success) | All |
| USER-002 | User Logout (audit logged) | All |
| USER-003 | Role-based access control | All |
| PAT-001 | Patient registration | Public |
| PAT-002 | Upload identity / health documents | Patient, Admin |
| PAT-003 | View and update patient profile | Patient, Doctor, Admin |
| PAT-004 | Book appointment | Patient |
| PAT-005 | Cancel appointment | Patient |
| PAT-006 | View medical history and summary | Patient, Doctor, Admin |
| PAT-007 | Appointment confirmation/cancellation notifications | System |
| DOC-001 | Doctor views their appointment schedule | Doctor, Admin |
| DOC-002 | Doctor sets and edits availability slots | Doctor |
| DOC-003 | Doctor records treatment details | Doctor |
| DOC-004 | Doctor/Admin updates patient record | Doctor, Admin |
| DOC-005 | Doctor receives treatment update notifications | System |
| HADM-001 | Add hospital | Admin |
| HADM-002 | View and search hospitals | Authenticated |
| HADM-003 | Edit hospital | Admin |
| HADM-004 | Add resources to hospitals | Admin |
| HADM-005 | Update/delete resources | Admin |
| HADM-006 | View resources by hospital | Authenticated |
| HADM-007 | Resource monitoring dashboard | Admin, Manager |
| HADM-008 | Generate hospital analytics reports | Admin, Manager |
| HADM-009 | User account lifecycle management | Admin |
| HADM-010 | View all appointments and reassign | Admin |
| HADM-011 | Patient check-in (arrival timestamped in AuditLog) | Admin |
| COMP-001 | Create and update compliance records | Compliance |
| COMP-002 | Search and filter compliance records | Compliance, Admin |
| COMP-003 | View audit logs (read-only) | Compliance, Admin, Auditor |
| COMP-004 | Compliance deadline alerts (notification) | System |
| AUDT-001 | Create and update audits | Auditor |
| AUDT-002 | Audit dashboard | Auditor, Admin |
| PGM-001 | Program performance dashboard | Admin, Manager |
| PGM-002 | Generate scoped program reports | Admin, Manager |

---

## 5. Authentication & Authorization

### How it works

```
1. Register:  POST /api/users/register  (or /api/patients/register)
2. Login:     POST /api/users/login     → returns { "token": "eyJ..." }
3. Use token: Add header to all requests:
              Authorization: Bearer eyJ...
```

### JWT token contents

| Claim | Value |
|---|---|
| `sub` | User email |
| `role` | User role (e.g. `Admin`, `Doctor`) |
| `iat` | Issued at (Unix timestamp) |
| `exp` | Expiry (24 hours after issue) |

### Registration rules

| Role | How to register | Initial status |
|---|---|---|
| **Admin** | `POST /api/users/register` with `"role": "Admin"` | **Active** immediately |
| Doctor | `POST /api/users/register` with `"role": "Doctor"` | Pending (admin must approve) |
| Compliance | `POST /api/users/register` with `"role": "Compliance"` | Pending |
| Auditor | `POST /api/users/register` with `"role": "Auditor"` | Pending |
| Manager | `POST /api/users/register` with `"role": "Manager"` | Pending |
| **Patient** | `POST /api/patients/register` (different endpoint) | Pending |

Pending accounts receive a `400` with message _"Account is pending approval"_ on login until an Admin calls `PUT /api/users/{id}/status` with `{ "status": "Active" }`.

### Role → Endpoint access matrix

| Endpoint | Admin | Doctor | Patient | Compliance | Auditor | Manager |
|---|---|---|---|---|---|---|
| `POST /api/users/register` | ✅ | ✅ | — | ✅ | ✅ | ✅ |
| `POST /api/users/login` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `GET /api/users` | ✅ | | | | | |
| `PUT /api/users/{id}/status` | ✅ | | | | | |
| `POST /api/patients/register` | Public | | | | | |
| `GET /api/patients/{id}` | ✅ | ✅ | Own only | | | |
| `PUT /api/patients/{id}` | ✅ | | Own only | | | |
| `POST /api/patients/documents` | ✅ | | Own only | | | |
| `GET /api/patients/{id}/history` | ✅ | ✅ | Own only | | | |
| `GET /api/patients/{id}/summary` | ✅ | ✅ | Own only | | | |
| `POST /api/hospitals` | ✅ | | | | | |
| `GET /api/hospitals/**` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `PUT/DELETE /api/hospitals/*` | ✅ | | | | | |
| `POST /api/resources` | ✅ | | | | | |
| `GET /api/resources/**` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `PUT/DELETE /api/resources/*` | ✅ | | | | | |
| `GET /api/analytics/**` | ✅ | | | | | ✅ |
| `POST /api/schedules` | | ✅ | | | | |
| `PUT /api/schedules/*` | | ✅ | | | | |
| `GET /api/schedules/**` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `POST /api/appointments/book` | | | ✅ | | | |
| `PUT /api/appointments/cancel` | | | Own only | | | |
| `GET /api/appointments/doctor/*` | ✅ | ✅ | | | | |
| `GET /api/appointments` | ✅ | | | | | |
| `PUT /api/appointments/*/reassign` | ✅ | | | | | |
| `PUT /api/appointments/*/checkin` | ✅ | | | | | |
| `POST /api/treatments` | | ✅ | | | | |
| `PUT /api/treatments/patients/*` | ✅ | ✅ | | | | |
| `POST /api/compliance` | | | | ✅ | | |
| `PUT /api/compliance/*` | | | | ✅ | | |
| `GET /api/compliance` | ✅ | | | ✅ | | |
| `GET /api/compliance/audit-logs` | ✅ | | | ✅ | ✅ | |
| `POST /api/audits` | | | | | ✅ | |
| `PUT /api/audits/*` | | | | | ✅ | |
| `GET /api/audits/**` | ✅ | | | | ✅ | |
| `GET /api/notifications/{userId}` | ✅ | Own only | Own only | Own only | Own only | Own only |
| `GET /api/program/dashboard` | ✅ | | | | | ✅ |
| `POST /api/program/reports` | ✅ | | | | | ✅ |

> **Ownership rules:** Where "Own only" is shown, the PATIENT/DOCTOR/etc. can only access their own data. Admin can always access anyone's data. These checks are enforced in the service layer, not just by URL rules.

---

## 6. Running the Project Locally

### Prerequisites

- Java 17+
- Maven 3.6+
- MySQL 8+

### Steps

```bash
# 1. Create the database
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS healthcaregov;"

# 2. Configure credentials (if different from defaults)
# Edit: src/main/resources/application.properties
spring.datasource.username=root
spring.datasource.password=root

# 3. Build and run
cd healthcaregov-mvp
./mvnw spring-boot:run

# Server starts on: http://localhost:9090
```

Spring Boot will auto-create all 14 database tables on first run (`ddl-auto=update`). No manual schema is needed.

### Configuration properties

| Property | Default | Description |
|---|---|---|
| `server.port` | `9090` | HTTP port |
| `spring.datasource.url` | `jdbc:mysql://localhost:3306/healthcaregov` | MySQL URL |
| `spring.datasource.username` | `root` | MySQL username |
| `spring.datasource.password` | `root` | MySQL password |
| `app.jwt.secret` | (see file) | JWT signing secret — **change in production** |
| `app.jwt.expiration-ms` | `86400000` | Token lifetime (24 hours) |

### First-run sequence

```
1.  POST /api/users/register         { role: "Admin" }   → Active immediately
2.  POST /api/users/register         { role: "Doctor" }  → Pending
3.  POST /api/users/login            admin credentials   → get JWT token
4.  PUT  /api/users/{doctorId}/status { "status": "Active" }  (use admin token)
5.  POST /api/patients/register      patient details
6.  PUT  /api/users/{patientId}/status { "status": "Active" } (use admin token)
7.  POST /api/users/login            doctor credentials  → get doctor JWT
8.  POST /api/hospitals              hospital details    (use admin token)
9.  POST /api/schedules              slot details        (use doctor token)
10. POST /api/users/login            patient credentials → get patient JWT
11. POST /api/appointments/book      booking details     (use patient token)
```

---

## 7. API Reference

All endpoints are prefixed with `http://localhost:9090`.

For authenticated endpoints, include: `Authorization: Bearer <token>`

---

### 7.1 User Management

#### Register User
```
POST /api/users/register
Auth: None (public)
```

**Request body:**
```json
{
  "name":     "Dr. Priya Sharma",
  "role":     "Doctor",
  "email":    "doctor@hcg.com",
  "phone":    "9000000002",
  "password": "securepass123"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| name | string | ✅ | |
| role | string | ✅ | Admin / Doctor / Compliance / Auditor / Manager |
| email | string | ✅ | Must be a valid email, unique |
| phone | string | ✅ | |
| password | string | ✅ | Stored as BCrypt hash |

**Response (201):**
```json
{
  "userID": 2,
  "name": "Dr. Priya Sharma",
  "role": "Doctor",
  "email": "doctor@hcg.com",
  "phone": "9000000002",
  "status": "Pending"
}
```

---

#### Login
```
POST /api/users/login
Auth: None (public)
```

**Request body:**
```json
{
  "email":    "admin@hcg.com",
  "password": "admin123"
}
```

**Response (200):**
```json
{
  "userID":    1,
  "patientID": null,
  "name":      "System Admin",
  "role":      "Admin",
  "email":     "admin@hcg.com",
  "status":    "Active",
  "token":     "eyJhbGciOiJIUzI1NiJ9...",
  "message":   "Login successful."
}
```

> `patientID` is non-null only when `role` is `Patient`.
> The `token` must be used as `Authorization: Bearer <token>` on all subsequent requests.

---

#### Logout
```
POST /api/users/logout/{userId}
Auth: Any authenticated user (own userId only; Admin may logout anyone)
```

**Response (200):** `"Logged out successfully."`

---

#### List Users (HADM-009)
```
GET /api/users?status=Pending
Auth: Admin
```

| Query param | Required | Description |
|---|---|---|
| status | No | Filter by status: Pending / Active / Inactive / Rejected |

**Response (200):** Array of `UserResponse`

---

#### Update User Status (HADM-009)
```
PUT /api/users/{userId}/status?adminId={adminId}
Auth: Admin
```

**Request body:**
```json
{ "status": "Active" }
```

Allowed values: `Active` (approve/reactivate) | `Inactive` (deactivate) | `Rejected`

**Response (200):** Updated `UserResponse`

---

### 7.2 Patient Registration & Profile

#### Register Patient (PAT-001)
```
POST /api/patients/register
Auth: None (public)
```

**Request body:**
```json
{
  "name":        "Ravi Kumar",
  "dob":         "1990-05-15",
  "gender":      "Male",
  "address":     "42 Gandhi Nagar, Chennai",
  "contactInfo": "9876543210",
  "email":       "ravi@email.com",
  "password":    "pass1234"
}
```

| Field | Type | Required | Validation |
|---|---|---|---|
| name | string | ✅ | |
| dob | string | ✅ | Format: `yyyy-MM-dd` |
| gender | string | ✅ | |
| address | string | ✅ | |
| contactInfo | string | ✅ | Exactly 10 digits |
| email | string | ✅ | Valid email, unique |
| password | string | ✅ | |

Creates both a `User` account (status=Pending) and a `Patient` profile linked to it. Patient appears in `GET /api/users?status=Pending` until approved by Admin.

**Response (201):** `PatientResponse`

---

#### View Patient Profile (PAT-003)
```
GET /api/patients/{patientId}
Auth: Patient (own only), Doctor, Admin
```

**Response (200):**
```json
{
  "patientID":   1,
  "userID":      5,
  "name":        "Ravi Kumar",
  "dob":         "1990-05-15",
  "gender":      "Male",
  "address":     "42 Gandhi Nagar, Chennai",
  "contactInfo": "9876543210",
  "status":      "Active"
}
```

---

#### Update Patient Profile (PAT-003)
```
PUT /api/patients/{patientId}
Auth: Patient (own only), Admin
```

**Request body:**
```json
{
  "address":     "99 New Street, Chennai",
  "contactInfo": "9000011111"
}
```

Only `address` and `contactInfo` can be updated by patients.

---

#### Upload Document (PAT-002)
```
POST /api/patients/documents
Auth: Patient (own only), Admin
```

**Request body:**
```json
{
  "patientID": 1,
  "docType":   "IDProof",
  "fileURI":   "/uploads/patient1/aadhar.pdf"
}
```

| Field | Allowed values |
|---|---|
| docType | `IDProof` / `HealthCard` |

**Response (201):** `DocumentResponse` with `verificationStatus: "Pending"`

---

#### View Treatment History (PAT-006)
```
GET /api/patients/{patientId}/history
Auth: Patient (own only), Doctor, Admin
```

**Response (200):** Array of `TreatmentResponse`

---

#### View Medical Summary (PAT-006)
```
GET /api/patients/{patientId}/summary
Auth: Patient (own only), Doctor, Admin
```

**Response (200):** `MedicalSummaryResponse`

---

### 7.3 Hospital Management

#### Add Hospital (HADM-001)
```
POST /api/hospitals
Auth: Admin
```

**Request body:**
```json
{
  "name":     "City General Hospital",
  "location": "Chennai",
  "capacity": 500,
  "status":   "Active"
}
```

| Field | Validation |
|---|---|
| capacity | 1 – 10,000 |

**Response (201):** `HospitalResponse`

---

#### List All Hospitals (HADM-002)
```
GET /api/hospitals
Auth: Any authenticated user
```

---

#### Get Hospital by ID (HADM-002)
```
GET /api/hospitals/{id}
Auth: Any authenticated user
```

---

#### Search Hospitals (HADM-002)
```
GET /api/hospitals/search?query=Chennai
Auth: Any authenticated user
```

Searches both `name` and `location` fields (case-insensitive).

---

#### Edit Hospital (HADM-003)
```
PUT /api/hospitals/{id}
Auth: Admin
```

---

#### Delete Hospital (HADM-003)
```
DELETE /api/hospitals/{id}
Auth: Admin
Response: 204 No Content
```

---

### 7.4 Resource Management

#### Add Resource (HADM-004)
```
POST /api/resources
Auth: Admin
```

**Request body:**
```json
{
  "hospitalID": 1,
  "type":       "Beds",
  "quantity":   200,
  "status":     "Available"
}
```

| Field | Allowed values | Validation |
|---|---|---|
| type | `Beds` / `Equipment` / `Staff` | |
| quantity | integer | 1 – 100,000 |

**Response (201):** `ResourceResponse`

---

#### List Resources (HADM-006)
```
GET /api/resources?hospitalId=1
Auth: Any authenticated user
```

`hospitalId` is optional. If omitted, returns all resources.

---

#### Get Resource by ID
```
GET /api/resources/{id}
Auth: Any authenticated user
```

---

#### Update Resource (HADM-005)
```
PUT /api/resources/{id}
Auth: Admin
```

---

#### Delete Resource (HADM-005)
```
DELETE /api/resources/{id}
Auth: Admin
Response: 204 No Content
```

---

### 7.5 Analytics & Reports

#### Hospital Analytics Dashboard (HADM-007)
```
GET /api/analytics/hospitals
Auth: Admin, Manager
```

**Response (200):**
```json
{
  "totalHospitals": 3,
  "totalBeds":      600,
  "totalEquipment": 120,
  "totalStaff":     300,
  "totalCapacity":  1500
}
```

---

#### Hospital Capacity Report (HADM-008)
```
GET /api/analytics/reports/hospital-capacity
Auth: Admin, Manager
```

---

#### Resource Availability Report (HADM-008)
```
GET /api/analytics/reports/resource-availability
Auth: Admin, Manager
```

---

#### Resource Distribution Report (HADM-008)
```
GET /api/analytics/reports/resource-distribution
Auth: Admin, Manager
```

---

### 7.6 Doctor Schedule

#### Add Availability Slot (DOC-002)
```
POST /api/schedules
Auth: Doctor
```

**Request body:**
```json
{
  "doctorId":      2,
  "hospitalId":    1,
  "availableDate": "2025-12-25",
  "timeSlot":      "10:00:00"
}
```

| Field | Format |
|---|---|
| availableDate | `yyyy-MM-dd` |
| timeSlot | `HH:mm:ss` |

Duplicate slots (same doctorId + date + timeSlot) are rejected with `400`.

**Response (201):** `ScheduleResponse` with `status: "Available"`

---

#### Edit Availability Slot (DOC-002)
```
PUT /api/schedules/{scheduleId}
Auth: Doctor
```

Cannot edit a slot with `status: "Booked"`.

---

#### View Doctor's Slots (DOC-001, DOC-002)
```
GET /api/schedules/doctor/{doctorId}
Auth: Any authenticated user
```

**Response (200):** Array of `ScheduleResponse`

---

### 7.7 Appointments

#### Book Appointment (PAT-004)
```
POST /api/appointments/book
Auth: Patient
```

**Request body:**
```json
{
  "patientID":  1,
  "doctorID":   2,
  "hospitalID": 1,
  "date":       "2025-12-25",
  "time":       "10:00:00"
}
```

- A schedule slot must exist for the doctor on this date/time with `status: "Available"`
- On success: slot → `Booked`, appointment → `Confirmed`, patient receives a notification

**Response (201):** `AppointmentResponse`

---

#### Cancel Appointment (PAT-005)
```
PUT /api/appointments/cancel
Auth: Patient (own appointments only)
```

**Request body:**
```json
{ "appointmentID": 1 }
```

- On success: slot reverts to `Available`, appointment → `Cancelled`, patient notified

---

#### Doctor's Appointment List (DOC-001)
```
GET /api/appointments/doctor/{doctorId}
Auth: Doctor, Admin
```

---

#### All Appointments (HADM-010)
```
GET /api/appointments
Auth: Admin
```

---

#### Reassign Appointment (HADM-010)
```
PUT /api/appointments/{appointmentId}/reassign?adminId={adminId}
Auth: Admin
```

**Request body:**
```json
{ "newDoctorId": 3 }
```

---

#### Check-In Patient (HADM-011)
```
PUT /api/appointments/{appointmentId}/checkin?adminId={adminId}
Auth: Admin
```

Changes appointment status to `Arrived`. Arrival timestamp recorded in AuditLog.

---

### 7.8 Treatment & Medical Records

#### Record Treatment (DOC-003)
```
POST /api/treatments
Auth: Doctor
```

**Request body:**
```json
{
  "patientId":      1,
  "doctorId":       2,
  "diagnosis":      "Hypertension Stage 1",
  "prescription":   "Amlodipine 5mg once daily",
  "treatmentNotes": "Reduce salt intake. Exercise daily.",
  "status":         "Active"
}
```

| Field | Allowed values |
|---|---|
| status | `Active` / `Completed` |

Also updates the patient's rolling `MedicalRecord` and notifies the doctor.

**Response (201):** `TreatmentResponse`

---

#### Update Patient Medical Record (DOC-004)
```
PUT /api/treatments/patients/{patientId}
Auth: Doctor, Admin
```

**Request body:**
```json
{
  "updaterId":   2,
  "name":        "Ravi Kumar",
  "contactInfo": "9000011111",
  "status":      "Active"
}
```

- Blocked if patient status is `Finalized`
- Only Doctor or Admin roles are permitted

---

### 7.9 Compliance

#### Create Compliance Record (COMP-001)
```
POST /api/compliance?officerId={officerId}
Auth: Compliance Officer
```

**Request body:**
```json
{
  "entityId": 1,
  "type":     "Appointment",
  "result":   "Pass",
  "notes":    "All documentation verified."
}
```

| Field | Allowed values |
|---|---|
| type | `Appointment` / `Treatment` / `Hospital` |
| result | `Pass` / `Fail` |

Also notifies the compliance officer and writes to AuditLog.

**Response (201):** `ComplianceResponse`

---

#### Update Compliance Record (COMP-001)
```
PUT /api/compliance/{id}?officerId={officerId}
Auth: Compliance Officer
```

---

#### Search Compliance Records (COMP-002)
```
GET /api/compliance?type=Appointment&result=Pass&startDate=2025-01-01&endDate=2025-12-31
Auth: Compliance Officer, Admin
```

| Query param | Required | Format |
|---|---|---|
| type | No | Appointment / Treatment / Hospital |
| result | No | Pass / Fail |
| entityId | No | Integer |
| startDate | No | `yyyy-MM-dd` |
| endDate | No | `yyyy-MM-dd` |

---

#### View Audit Logs (COMP-003)
```
GET /api/compliance/audit-logs
Auth: Compliance Officer, Admin, Auditor
```

Read-only. Logs cannot be deleted.

**Response (200):** Array of `AuditLogResponse`

---

### 7.10 Audit

#### Create Audit (AUDT-001)
```
POST /api/audits
Auth: Auditor
```

**Request body:**
```json
{
  "officerId": 4,
  "scope":     "Q4 Hospital Compliance Review",
  "findings":  null,
  "status":    null
}
```

New audits always start with `status: "Scheduled"` regardless of the `status` field sent.

**Response (201):** `AuditResponse`

---

#### Update Audit (AUDT-001)
```
PUT /api/audits/{auditId}?requesterId={requesterId}
Auth: Auditor
```

**Status flow (strictly enforced):**
```
Scheduled → In-Progress → Completed
```

Once `Completed`, only users with `Auditor` role can make further updates.

---

#### Audit Dashboard (AUDT-002)
```
GET /api/audits?status=Scheduled
Auth: Auditor, Admin
```

`status` filter is optional.

---

#### Get Audit Details (AUDT-002)
```
GET /api/audits/{auditId}
Auth: Auditor, Admin
```

---

### 7.11 Notifications

#### Get Notifications
```
GET /api/notifications/{userId}
Auth: Any authenticated user (own userId only; Admin may access any)
```

**Response (200):**
```json
[
  {
    "notificationID": 1,
    "userId":         5,
    "entityId":       1,
    "message":        "Your appointment with Dr. Priya on 2025-12-25 at 10:00 is confirmed.",
    "category":       "Appointment",
    "status":         "Unread",
    "createdDate":    "2025-12-20T09:30:00"
  }
]
```

| Category | Triggered when |
|---|---|
| `Appointment` | Appointment confirmed or cancelled (→ patient) |
| `Treatment` | Treatment recorded (→ doctor) |
| `Compliance` | Compliance record created (→ officer) |

---

### 7.12 Program Dashboard & Reports

#### Program Dashboard (PGM-001)
```
GET /api/program/dashboard?startDate=2025-01-01&endDate=2025-12-31
Auth: Admin, Manager
```

Date range is optional and filters appointment counts.

**Response (200):**
```json
{
  "totalAppointments":    150,
  "confirmedAppointments": 120,
  "cancelledAppointments": 30,
  "totalTreatments":       95,
  "activeTreatments":      40,
  "completedTreatments":   55,
  "totalHospitals":         3,
  "totalCapacity":       1500,
  "totalComplianceRecords": 25,
  "passedCompliance":       20,
  "failedCompliance":        5
}
```

---

#### Generate Report (PGM-002)
```
POST /api/program/reports
Auth: Admin, Manager
```

**Request body:**
```json
{
  "hospitalId": 1,
  "scope":      "Hospital"
}
```

| scope value | Metrics included |
|---|---|
| `Appointment` | confirmed, cancelled counts for the hospital |
| `Treatment` | active, completed counts |
| `Hospital` | capacity, beds, equipment, staff |
| `Compliance` | passed, failed counts |

**Response (201):** `ReportResponse` with `metrics` as a JSON string

---

## 8. Error Responses

All errors return a consistent JSON structure:

```json
{
  "status":    400,
  "message":   "Human-readable description of the problem",
  "timestamp": 1700000000000
}
```

| HTTP Status | When |
|---|---|
| `400 Bad Request` | Validation failure, business rule violation, ownership check failed |
| `401 Unauthorized` | No token or invalid/expired token |
| `403 Forbidden` | Valid token but insufficient role |
| `404 Not Found` | Entity not found |
| `409 Conflict` | Duplicate entry (email, schedule slot) or FK constraint on delete |
| `500 Internal Server Error` | Unexpected server error |

---

## 9. Validation Reference

| Field | Rule |
|---|---|
| Patient `contactInfo` | Exactly 10 digits (`\d{10}`) |
| Patient `dob` | Format `yyyy-MM-dd` |
| Document `docType` | `IDProof` or `HealthCard` |
| Schedule / Appointment `time` | Format `HH:mm:ss` |
| Schedule `availableDate` | Format `yyyy-MM-dd` |
| Hospital `capacity` | Integer, 1–10,000 |
| Resource `quantity` | Integer, 1–100,000 |
| Compliance `type` | `Appointment`, `Treatment`, or `Hospital` |
| Report `scope` | `Appointment`, `Treatment`, `Hospital`, or `Compliance` |
| User status (HADM-009) | `Active`, `Inactive`, or `Rejected` |
| Audit status flow | `Scheduled` → `In-Progress` → `Completed` (no skipping, no reverting) |
| User role (register) | `Admin`, `Doctor`, `Compliance`, `Auditor`, `Manager` (not `Patient`) |
