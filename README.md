# OCSMS — Online College Society Management System

> **FAST-NUCES Peshawar** | Software Design & Architecture (SDA) — Task 1 & Task 2

A full-featured **JavaFX 21** desktop application for managing student societies, events, memberships, finance, attendance, and certificates — backed by a **Supabase (PostgreSQL)** cloud database.

---

## Table of Contents

- [Features](#features)
- [Technology Stack](#technology-stack)
- [Architecture](#architecture)
- [Role-Based Access Control](#role-based-access-control)
- [Project Structure](#project-structure)
- [Database Schema](#database-schema)
- [Prerequisites](#prerequisites)
- [Setup & Run](#setup--run)
- [Build Fat JAR](#build-fat-jar)
- [Default Credentials](#default-credentials)
- [Key Design Decisions](#key-design-decisions)

---

## Features

| Module | Highlights |
|---|---|
| **Authentication** | Login / Register; BCrypt-hashed passwords; role-aware routing |
| **Societies** | Create, browse, search, filter by category; auto-resize logo (200×200 px BICUBIC); archive/delete |
| **Membership** | Apply, approve, reject; duplicate-application guard; President views all approved members |
| **Events** | Create events, set capacity/date; register/deregister attendance |
| **Attendance** | Mark and view attendance per event |
| **Finance** | Treasurer allocates budgets per society; President views allocation and uploads bills/receipts; PDF budget report generation via PDFBox |
| **Certificates** | Issue certificates to event attendees; PDF export; student dashboard shows live certificate count |
| **Manage Presidents** | University Admin creates President accounts with admin-set passwords; Presidents can then login and create/manage their own society |
| **Dark UI** | Professional indigo/slate dark design system; no white-on-white; accessible color palette |

---

## Technology Stack

| Layer | Technology |
|---|---|
| Language | Java 17 (Eclipse Adoptium / Temurin) |
| UI Framework | JavaFX 21 (FXML + CSS) |
| Build Tool | Apache Maven 3.9.6 |
| Database | Supabase (PostgreSQL) via REST API |
| HTTP Client | OkHttp 4.12 |
| JSON | Gson 2.10.1 |
| PDF | Apache PDFBox 3.0.1 |
| Password Hashing | bcrypt (at.favre.lib 0.10.2) |
| Module System | JPMS (`module-info.java`) |

---

## Architecture

The project follows a **layered architecture** (MVC + Repository pattern):

```
┌─────────────────────────────────────┐
│           Presentation Layer        │
│   JavaFX FXML + Controllers (GUI)   │
└────────────────┬────────────────────┘
                 │
┌────────────────▼────────────────────┐
│            Service Layer            │
│  MembershipService · SocietyService │
│  EventService · NotificationService │
└────────────────┬────────────────────┘
                 │
┌────────────────▼────────────────────┐
│          Repository Layer           │
│  SocietyRepository · UserRepository │
│  EventRepository · FinanceRepository│
└────────────────┬────────────────────┘
                 │
┌────────────────▼────────────────────┐
│      Supabase REST API (OkHttp)     │
│         PostgreSQL Database         │
└─────────────────────────────────────┘
```

**Key utilities:**
- `SessionManager` — singleton holding the authenticated `User` object
- `SupabaseClient` — wraps OkHttp with Supabase URL + service-role key
- `LogoResizeUtil` — proportional BICUBIC image scaling padded to 200×200 px PNG
- `AlertUtil` — reusable JavaFX alert/confirm dialogs
- `PdfUtil` — PDFBox wrapper for certificate and budget PDF generation

---

## Role-Based Access Control

| Role | Registration | Can Do |
|---|---|---|
| **Student** | Self-register | Browse societies, apply for membership (once per society), register for events, view own certificates |
| **Society President** (`SOCIETY_ADMIN`) | Created by University Admin | Create & manage one society, upload logo, view approved members, view budget allocation, upload bills |
| **Treasurer** | Hard-coded credentials | Allocate budget to any society, view all finance entries, generate PDF report |
| **University Admin** | Hard-coded credentials | Full control: create/remove presidents, archive/delete societies, manage events, view all data |

> **Note:** Only students can self-register. Society Presidents are created exclusively by the University Admin (who also sets their initial password).

---

## Project Structure

```
ocsms/
├── pom.xml
└── src/
    └── main/
        ├── java/com/ocsms/
        │   ├── Main.java                        # JavaFX entry point
        │   ├── enums/                           # MembershipStatus, SocietyStatus, UserRole
        │   ├── model/                           # Domain entities
        │   │   ├── User.java / Student.java / SocietyAdmin.java
        │   │   ├── Treasurer.java / UniversityAdmin.java
        │   │   ├── Society.java / Event.java / Membership.java
        │   │   ├── Certificate.java / Announcement.java
        │   │   ├── BudgetAllocation.java / BudgetBill.java / FinanceEntry.java
        │   │   └── EventRegistration.java
        │   ├── repository/                      # Supabase REST calls
        │   │   ├── SupabaseClient.java
        │   │   ├── UserRepository.java
        │   │   ├── SocietyRepository.java
        │   │   └── EventRepository.java
        │   ├── service/                         # Business logic
        │   │   ├── MembershipService.java
        │   │   ├── SocietyService.java
        │   │   ├── EventService.java
        │   │   └── NotificationService.java
        │   ├── gui/controllers/                 # FXML controllers
        │   │   ├── LoginController.java
        │   │   ├── DashboardController.java
        │   │   ├── SocietyController.java
        │   │   ├── EventController.java
        │   │   ├── MembershipController.java
        │   │   ├── AttendanceController.java
        │   │   ├── FinanceController.java
        │   │   └── CertificateController.java
        │   └── util/
        │       ├── AlertUtil.java
        │       ├── SessionManager.java
        │       ├── LogoResizeUtil.java
        │       └── PdfUtil.java
        └── resources/
            ├── css/
            │   └── styles.css                   # Dark design system
            └── fxml/
                ├── login.fxml
                ├── dashboard.fxml
                ├── societies.fxml
                ├── events.fxml
                ├── membership.fxml
                ├── attendance.fxml
                ├── finance.fxml
                └── certificates.fxml
```

---

## Database Schema

The following tables are required in your Supabase project:

```sql
-- Users (students + society presidents)
CREATE TABLE ocsms_users (
    user_id      TEXT PRIMARY KEY,
    name         TEXT NOT NULL,
    roll_number  TEXT UNIQUE,
    email        TEXT UNIQUE,
    password     TEXT NOT NULL,     -- BCrypt hash
    role         TEXT NOT NULL,     -- STUDENT | SOCIETY_ADMIN | TREASURER | UNIVERSITY_ADMIN
    created_at   TIMESTAMPTZ DEFAULT now()
);

-- Societies
CREATE TABLE ocsms_societies (
    society_id   TEXT PRIMARY KEY,
    name         TEXT NOT NULL,
    category     TEXT,
    description  TEXT,
    member_limit INT DEFAULT 50,
    status       TEXT DEFAULT 'ACTIVE',
    president_id TEXT REFERENCES ocsms_users(user_id),
    logo_path    TEXT,
    created_at   TIMESTAMPTZ DEFAULT now()
);

-- Memberships
CREATE TABLE ocsms_memberships (
    membership_id TEXT PRIMARY KEY,
    student_id    TEXT REFERENCES ocsms_users(user_id),
    society_id    TEXT REFERENCES ocsms_societies(society_id),
    status        TEXT DEFAULT 'PENDING',   -- PENDING | APPROVED | REJECTED
    motivation    TEXT,
    applied_at    TIMESTAMPTZ DEFAULT now()
);

-- Events
CREATE TABLE ocsms_events (
    event_id     TEXT PRIMARY KEY,
    title        TEXT NOT NULL,
    description  TEXT,
    society_id   TEXT REFERENCES ocsms_societies(society_id),
    event_date   DATE,
    capacity     INT,
    created_at   TIMESTAMPTZ DEFAULT now()
);

-- Attendance
CREATE TABLE ocsms_attendance (
    attendance_id TEXT PRIMARY KEY,
    event_id      TEXT REFERENCES ocsms_events(event_id),
    student_id    TEXT REFERENCES ocsms_users(user_id),
    attended_at   TIMESTAMPTZ DEFAULT now()
);

-- Budget Allocations
CREATE TABLE ocsms_budget_allocations (
    allocation_id TEXT PRIMARY KEY,
    society_id    TEXT REFERENCES ocsms_societies(society_id),
    amount        NUMERIC(12,2),
    allocated_by  TEXT,
    note          TEXT,
    allocated_at  TIMESTAMPTZ DEFAULT now()
);

-- Budget Bills (uploaded by President)
CREATE TABLE ocsms_budget_bills (
    bill_id       TEXT PRIMARY KEY,
    allocation_id TEXT REFERENCES ocsms_budget_allocations(allocation_id),
    file_path     TEXT,
    description   TEXT,
    uploaded_at   TIMESTAMPTZ DEFAULT now()
);

-- Certificates
CREATE TABLE ocsms_certificates (
    cert_id           TEXT PRIMARY KEY,
    student_id        TEXT REFERENCES ocsms_users(user_id),
    event_id          TEXT REFERENCES ocsms_events(event_id),
    issued_date       DATE DEFAULT CURRENT_DATE,
    verification_code TEXT UNIQUE
);
```

> Run these SQL commands in the **Supabase SQL Editor** under your project dashboard.

---

## Prerequisites

| Requirement | Version |
|---|---|
| JDK | 17 (Eclipse Adoptium / Temurin recommended) |
| Apache Maven | 3.9.x |
| Internet | Required (Supabase REST API) |

---

## Setup & Run

### 1. Clone / Open the project

```
c:\Users\M Bilal\Desktop\SDA CODE\ocsms\
```

### 2. Configure Supabase credentials

Open `src/main/java/com/ocsms/repository/SupabaseClient.java` and set:

```java
private static final String SUPABASE_URL = "https://<your-project-ref>.supabase.co";
private static final String SERVICE_KEY  = "<your-service-role-key>";
```

### 3. Run the database setup SQL

Paste the schema from the [Database Schema](#database-schema) section into the **Supabase SQL Editor** and execute.

### 4. Build & launch

```powershell
# Set JAVA_HOME (adjust path if different)
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-17.0.18.8-hotspot"
$env:PATH      = "$env:JAVA_HOME\bin;C:\Users\M Bilal\Desktop\SDA CODE\apache-maven\apache-maven-3.9.6\bin;$env:PATH"

# Compile
mvn clean compile

# Run
mvn javafx:run
```

---

## Build Fat JAR

```powershell
mvn clean package
java -jar target/ocsms-1.0.0.jar
```

> **Note:** JavaFX modules must be on the module path. If running the JAR directly outside Maven, add `--module-path` and `--add-modules` flags for your local JavaFX SDK.

---

## Default Credentials

| Role | Roll No. / Username | Password |
|---|---|---|
| University Admin | `ADMIN-001` | `admin123` |
| Treasurer | `FIN-001` | `finance123` |
| Student | *(self-register)* | *(chosen at registration)* |
| Society President | *(created by Admin)* | *(set by Admin)* |

---

## Key Design Decisions

1. **Supabase over local DB** — REST-based cloud storage means no database installation required on client machines; OkHttp + Gson handle all communication.

2. **JPMS (module-info.java)** — Enforces explicit dependency declarations; `requires java.desktop` added for AWT/ImageIO used by `LogoResizeUtil`.

3. **BCrypt password hashing** — All user passwords are stored as BCrypt hashes (never plain text), including admin-set President passwords.

4. **Logo auto-resize** — `LogoResizeUtil` performs proportional BICUBIC scaling then pads to exactly 200×200 px on a transparent canvas, guaranteeing consistent branding regardless of original image dimensions.

5. **Async Tasks everywhere** — All Supabase I/O runs on background threads via `javafx.concurrent.Task`, with `Platform.runLater()` for UI updates, keeping the UI fully responsive.

6. **One President → One Society** — Enforced both in the UI (button disabled after first creation) and at the repository layer (`findByPresidentId` check).

7. **Duplicate membership guard** — Students cannot reapply to a society where they already have an ACTIVE or PENDING membership; enforced at both the service layer and the UI (button disabled + relabeled).

---

> **Developed for SDA coursework — FAST-NUCES Peshawar, 2025–2026**


> ## Team Members

| Name | GitHub |
|--------|--------|
| Muhammad Bilal | www.github.com/bilal-773 |
|Muhammad Muddasir | www.github.com/muddasir-io |
