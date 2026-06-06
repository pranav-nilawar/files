# 🎓 LearnHub — Enterprise Learning Platform

> Built on **IBM webMethods Integration Server 10.15** · Oracle DB · React 18

LearnHub is a full-stack enterprise learning platform that demonstrates real-world IBM webMethods IS integration patterns — JDBC Adapters, Pub/Sub Messaging, Flat File Processing, REST API Descriptor, REPEAT/EXIT Retry, Transaction Boundaries, and external service integration — all within a single coherent working application.

---

## 📋 Table of Contents

- [Project Overview](#project-overview)
- [Tech Stack](#tech-stack)
- [IS Package Structure](#is-package-structure)
- [Oracle Database Design](#oracle-database-design)
- [Flow Services](#flow-services)
- [REST API Endpoints](#rest-api-endpoints)
- [Pub/Sub Architecture](#pubsub-architecture)
- [Flat File Processing](#flat-file-processing)
- [REPEAT/EXIT Retry Pattern](#repeatexit-retry-pattern)
- [External Integrations](#external-integrations)
- [Frontend Architecture](#frontend-architecture)
- [Deployment Guide](#deployment-guide)
- [Configuration Reference](#configuration-reference)

---

## Project Overview

### What LearnHub Does

| Role | Capabilities |
|---|---|
| **Student** | Register · Login · Browse courses · Enroll/Unenroll · Track lesson progress · AI Chatbot help |
| **Admin** | View live stats · Manage users · Monitor enrollments · Add courses via flat file · Batch import CSV · Live event feed · Daily email reports |

### Key Technical Problems Solved

| Problem | Solution |
|---|---|
| Student count consistency | IS JDBC Update SQL adapters — NOT Oracle triggers |
| Email must not block enrollment | Fully async Pub/Sub — Adapter Notification → IS Trigger → SMTP |
| Progress update must be atomic | REPEAT/EXIT retry + `pub.art.transaction` boundary |
| Flat file course add | `pub.flatFile:convertToValues` → `pub.publish:publish` → Oracle via Pub/Sub |
| Oracle reserved words (`LEVEL`, `DESCRIPTION`) | Renamed to `course_level`, `course_desc` |
| File path security | `fileAccessControl.cnf` with `allowedWritePaths` |
| API key security | Gemini key stored in IS flow only — never sent to browser |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Integration Server | IBM webMethods IS 10.15 (localhost:5555) |
| Database | Oracle — schema: `SAGAPPLICATION` |
| Frontend | React 18 CDN + Babel (served from IS pub folder) |
| External AI | Google Gemini API via `pub.client:http` |
| Email | Gmail SMTP via `pub.client:smtp` (port 587, TLS) |
| DB Connection | JDBC Adapter — `jdbcConnection` |

---

## IS Package Structure

```
LearnHub_Project/
├── adapterNotification/
│   ├── enrollmentInsertNotification          ← polls learnhub_enrollments every 10s
│   └── enrollmentInsertNotificationPublishDocument
│
├── adapters/
│   ├── admin/                                ← 8 JDBC adapter services
│   │   ├── batchAddCourses                   INSERT → BATCH_IMPORT_LOG
│   │   ├── deleteUser                        DELETE → LEARNHUB_USERS
│   │   ├── getAllEnrollments                 SELECT → ENROLLMENT_DETAILS view
│   │   ├── getAllUsers                       SELECT → LEARNHUB_USERS
│   │   ├── getNotifications                 SELECT → LEARNHUB_ENROLLMENT_NOTIFICATION
│   │   ├── getStats                          SELECT → ADMIN_STATS view
│   │   ├── insertCourse                      INSERT → LEARNHUB_COURSES
│   │   └── insertNotification               INSERT → LEARNHUB_ENROLLMENT_NOTIFICATION
│   │
│   └── (root)/                               ← 13 JDBC adapter services
│       ├── checkEnrollment                   SELECT
│       ├── deleteEnrollment                  DELETE
│       ├── getAllCourses                      SELECT
│       ├── getAllProgress                     SELECT
│       ├── getEnrollmentsByUser             SELECT
│       ├── getProgress                       SELECT
│       ├── getUserByEmail                    SELECT
│       ├── insertEnrollment                  INSERT
│       ├── insertProgressEvent              MERGE  ← idempotent on retry
│       ├── insertUser                        INSERT
│       ├── updateProgress                    CUSTOM SQL
│       ├── updateCourseStudentCount         UPDATE SQL (Expression: students + 1)
│       └── decrementCourseStudentCount      UPDATE SQL (Expression: CASE WHEN students > 0 THEN students - 1 ELSE 0 END)
│
├── admin/                                    ← 10 flow services
│   ├── addCourseFromFlatFile
│   ├── batchAddCourses
│   ├── deleteUser
│   ├── getAdminEvents
│   ├── getAllEnrollments
│   ├── getAllUsers
│   ├── getStats
│   ├── onCourseAdd
│   ├── onEnrollmentChange                   → sends SMTP email
│   └── sendDailyReport                      → IS Scheduler at 09:00
│
├── auth/
│   ├── loginUser
│   └── registerUser
│
├── courses/
│   ├── getChatbotResponse                   → Gemini AI via pub.client:http
│   └── getCourses
│
├── documents/
│   └── courseDoc                            ← publishable document type
│
├── enrollments/                              ← 6 flow services
│   ├── enrollUser
│   ├── getAllProgress
│   ├── getEnrollments
│   ├── getProgress
│   ├── unenrollUser
│   └── updateProgress                       ← REPEAT/EXIT + transaction
│
├── flatFile/
│   ├── courseImportDictionary
│   └── courseImportSchema                   ← 10 fields, no id
│
├── restAPIs/
│   ├── restAPIDescriptor
│   ├── restResource
│   └── restResource_
│
└── triggers/
    ├── onCourseAdd                           ← subscribes to courseDoc
    └── onEnrollment                          ← subscribes to EnrollmentEvent
```

> **Total:** 21 JDBC Adapter services + 20 Flow services across 11 namespaces

---

## Oracle Database Design

### Schema: SAGAPPLICATION — 5 Tables

#### `learnhub_users`
| Column | Type | Notes |
|---|---|---|
| id | NUMBER PK | DEFAULT users_seq.NEXTVAL |
| name | VARCHAR2(100) | NOT NULL |
| email | VARCHAR2(150) | UNIQUE NOT NULL |
| password_hash | VARCHAR2(255) | Plain text — direct comparison |
| role | VARCHAR2(10) | DEFAULT 'student' |
| created_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |

#### `learnhub_courses`
| Column | Type | Notes |
|---|---|---|
| id | NUMBER PK | Assigned by courses_before_insert trigger |
| title | VARCHAR2(200) | |
| category | VARCHAR2(50) | |
| instructor | VARCHAR2(100) | |
| duration | VARCHAR2(20) | |
| lessons | NUMBER | |
| rating | NUMBER(3,1) | |
| students | NUMBER | Maintained by IS Update SQL adapters |
| course_level | VARCHAR2(20) | Renamed — LEVEL is Oracle reserved word |
| color | VARCHAR2(20) | |
| course_desc | VARCHAR2(500) | Renamed — DESCRIPTION is Oracle reserved word |

#### `learnhub_enrollments`
| Column | Type | Notes |
|---|---|---|
| id | NUMBER PK | DEFAULT enrollments_seq.NEXTVAL |
| user_id | NUMBER FK | → learnhub_users.id |
| course_id | NUMBER FK | → learnhub_courses.id |
| enrolled_at | TIMESTAMP | DEFAULT CURRENT_TIMESTAMP |
| progress | NUMBER | DEFAULT 0 |
| current_lesson | NUMBER | DEFAULT 0 |

#### `learnhub_enrollment_notification`
| Column | Type | Notes |
|---|---|---|
| id | NUMBER PK | GENERATED ALWAYS AS IDENTITY |
| user_id | NUMBER | |
| course_id | NUMBER | |
| enrollment_id | NUMBER | |
| event_type | VARCHAR2(50) | |
| processed | VARCHAR2(1) | DEFAULT 'N' |

#### `learnhub_progress_events`
| Column | Type | Notes |
|---|---|---|
| id | NUMBER PK | GENERATED ALWAYS AS IDENTITY |
| user_id | NUMBER | |
| course_id | NUMBER | |
| lesson_number | NUMBER | |
| progress_pct | NUMBER | |

### Sequences
```sql
users_seq        -- START WITH 1
enrollments_seq  -- START WITH 1
courses_seq      -- START WITH 9  (courses 1-8 are seeded)
```

### Oracle Trigger
```sql
-- courses_before_insert
-- Fires BEFORE INSERT on learnhub_courses
-- Assigns courses_seq.NEXTVAL to id
CREATE OR REPLACE TRIGGER courses_before_insert
BEFORE INSERT ON learnhub_courses
FOR EACH ROW
BEGIN
  :NEW.id := courses_seq.NEXTVAL;
END;
```

### Views
```sql
admin_stats        -- COUNT(*) for users, courses, enrollments
enrollment_details -- JOIN: learnhub_users + learnhub_courses + learnhub_enrollments
```

### ⚠️ Student Count — IS Adapters, NOT Oracle Triggers

```
updateCourseStudentCount:
  Table      : LEARNHUB_COURSES
  Column     : STUDENTS
  Expression : students + 1
  WHERE      : ID = ?  (courseId)
  Called by  : enrollUser (after insertEnrollment)

decrementCourseStudentCount:
  Table      : LEARNHUB_COURSES
  Column     : STUDENTS
  Expression : CASE WHEN students > 0 THEN students - 1 ELSE 0 END
  WHERE      : ID = ?  (courseId)
  Called by  : unenrollUser (after deleteEnrollment)
```

---

## Flow Services

### auth:loginUser
```
1. MAP  %body/email → email, %body/password → password
2. INVOKE adapters:getUserByEmail
3. pub.list:size → IF resultSize == 0 → return {status:"error", message:"User not found"}
4. MAP results[0]/PASSWORD_HASH → storedPassword
5. IF password != storedPassword → return {status:"error", message:"Invalid credentials"}
6. MAP → {status:"login_success", user:{ID, NAME, EMAIL, ROLE}}
```

### auth:registerUser
```
1. MAP  %body/name, email, password
2. INVOKE adapters:getUserByEmail → IF exists → return {status:"error", message:"Email already registered"}
3. INVOKE adapters:insertUser (NAME, EMAIL, PASSWORD_HASH)
4. INVOKE adapters:getUserByEmail → fetch new user
5. MAP → {status:"register_success", user:{ID, NAME, EMAIL}}
```

### enrollments:updateProgress — REPEAT/EXIT + Transaction
```
REPEAT count=3  label=retryLoop
  INVOKE pub.art.transaction:startTransaction  ← OUTSIDE TRY, inside REPEAT

  TRY:
    INVOKE pub.math:divideFloats   (lessonNumber / totalLessons → ratio)
    INVOKE pub.math:multiplyFloats (ratio × 100 → progressPct)
    INVOKE adapters:updateProgress (UPDATE learnhub_enrollments)
    INVOKE adapters:insertProgressEvent (MERGE — idempotent on retry)
    INVOKE pub.art.transaction:commitTransaction
    EXIT 'retryLoop'

  CATCH:
    INVOKE pub.flow:getLastError
    INVOKE pub.art.transaction:rollbackTransaction
    ← loop retries automatically up to 3 times
```

> **Transaction boundary REQUIRED** — 2 writes (UPDATE + MERGE) must be atomic.
> MERGE on `learnhub_progress_events` prevents duplicate audit rows if retry occurs.

### admin:addCourseFromFlatFile
```
1. MAP 10 fields from %body (title, category, instructor, duration, lessons,
                              rating, students, courseLevel, color, courseDesc)
2. INVOKE pub.string:concat → build csvLine
3. INVOKE pub.file:stringToFile → files/courses.csv
       ⚠️  Path must be in allowedWritePaths (fileAccessControl.cnf)
4. INVOKE pub.flatFile:convertToValues (courseImportSchema → ffValues)
5. MAP each field from ffValues/recordWithNoID[0] individually
       ⚠️  NOT whole ffValues — causes @record-id error
6. INVOKE pub.publish:publish (documents/courseDoc)
7. → triggers/onCourseAdd → admin:onCourseAdd → adapters/admin:insertCourse
8. → Oracle courses_before_insert trigger assigns id via courses_seq
```

### admin:batchAddCourses
```
1. MAP csvData (9 cols + auto-injected color at pos 9), batchId
2. INVOKE pub.file:stringToFile → files/batch_import.csv
3. INVOKE pub.flatFile:convertToValues
4. pub.list:size → totalRecords
5. LOOP over ffValues/recordWithNoID:
     INVOKE pub.publish:publish (courseDoc per row)
     INVOKE adapters/admin:batchAddCourses (audit log)
6. MAP {status:"batch_complete", totalImported, batchId}
```

> Frontend auto-injects color at position 9 from category mapping before sending 9-col CSV.

### admin:onEnrollmentChange — SMTP Email
```
TRY:
  INVOKE adapters:getUserByEmail (get student email)
  INVOKE pub.string:concat (build email body)
  INVOKE pub.client:smtp
    smtpServer : smtp.gmail.com
    smtpPort   : 587
    isSSL      : true
    login      : youremail@gmail.com
    password   : 16-char Gmail App Password
CATCH:
  INVOKE pub.flow:getLastError → log error
  ← Email failure is non-critical. Enrollment is NEVER rolled back.
```

---

## REST API Endpoints

**Base URL:** `http://localhost:5555/learnhub/api`
**Descriptor:** `restAPIs/restAPIDescriptor`

| Method | Endpoint | Flow Service |
|---|---|---|
| POST | /auth/register | auth:registerUser |
| POST | /auth/login | auth:loginUser |
| GET | /courses | courses:getCourses |
| POST | /enrollments/enroll | enrollments:enrollUser |
| POST | /enrollments/unenroll | enrollments:unenrollUser |
| GET | /enrollments/{userId} | enrollments:getEnrollments |
| GET | /admin/stats | admin:getStats |
| GET | /admin/users | admin:getAllUsers |
| GET | /admin/enrollments | admin:getAllEnrollments |
| DELETE | /admin/users/{userId} | admin:deleteUser |

---

## Pub/Sub Architecture

### Enrollment Notification Flow (Asynchronous)
```
enrollUser (sync ~200ms)
  └─ insertEnrollment → Oracle INSERT
  └─ updateCourseStudentCount → students + 1
  └─ returns "enrolled" to student immediately ✓

  [5–15 seconds later — fully decoupled]:
  enrollmentInsertNotification (polls every 10s)
    └─ detects INSERT on learnhub_enrollments
    └─ publishes EnrollmentEvent to IS_LOCAL_CONNECTION
    └─ triggers/onEnrollment fires
    └─ admin:onEnrollmentChange
    └─ pub.client:smtp → Gmail → student's inbox
```

### Course Add Flow (via Pub/Sub)
```
admin:addCourseFromFlatFile
  └─ pub.publish:publish (documents/courseDoc)
  └─ triggers/onCourseAdd fires
  └─ admin:onCourseAdd
  └─ adapters/admin:insertCourse → Oracle INSERT
  └─ courses_before_insert trigger assigns id
```

---

## Flat File Processing

### courseImportSchema — 10 Fields
```
Field 1 : title       (String)
Field 2 : category    (String)
Field 3 : instructor  (String)
Field 4 : duration    (String)
Field 5 : lessons     (String)
Field 6 : rating      (String)
Field 7 : students    (String)
Field 8 : courseLevel (String)
Field 9 : color       (String)  ← auto-injected by frontend from category map
Field 10: courseDesc  (String)
```

### fileAccessControl.cnf
```
# Location: C:\wMServiceDesigner\IntegrationServer\config\fileAccessControl.cnf
# ⚠️ Restart IS after any changes to this file

allowedWritePaths=C:\wMServiceDesigner\IntegrationServer\packages\LearnHub_Project\files\
allowedReadPaths=C:\wMServiceDesigner\IntegrationServer\packages\LearnHub_Project\files\
allowedDeletePaths=C:\wMServiceDesigner\IntegrationServer\packages\LearnHub_Project\files\
```

> File written to `files/courses.csv` (single course) and `files/batch_import.csv` (batch).
> `fileName` in flow must use forward slashes, no trailing slash.

---

## External Integrations

### Gemini AI Chatbot — courses:getChatbotResponse
```
1. MAP %body/question
2. pub.string:concat → build JSON body
   Prompt: "Answer in plain text only, no markdown. Question: " + question
3. pub.client:http
   URL    : https://generativelanguage.googleapis.com/v1beta/...?key=API_KEY
   Method : POST
   Header : Content-Type = application/json
4. pub.io:bytesToString (response bytes → string)
5. pub.json:jsonStringToDocument (parse response)
6. MAP candidates[0]/content/parts[0]/text → answer
```

> ⚠️ API key stored only inside the IS flow as a Set Value. Never sent to browser.
> Chatbot shown on home page only (💬 button).

### IS Scheduler — admin:sendDailyReport
```
Type      : Complex Repeating
All months, All weekdays
Hour      : 9
Minute    : 0
Service   : admin:sendDailyReport → pub.client:smtp
```

---

## Frontend Architecture

### Files
| File | Location | Role |
|---|---|---|
| `index.html` | IS pub folder | Student portal (React 18) |
| `admin.html` | IS pub folder | Admin panel (React 18) |

**URL:** `http://localhost:5555/LearnHub_Project/index.html`

Same-origin — served from IS itself → zero CORS issues.

### Auth Guard
```javascript
// admin.html — reads sessionStorage on load
const token = sessionStorage.getItem('user');
if (!token || JSON.parse(token).role !== 'admin') {
  window.location.href = '/LearnHub_Project/index.html';
}
```

### Role-Based Access
```
loginUser returns ROLE from learnhub_users.role
sessionStorage stores { id, name, email, role }
index.html  → Admin Panel link shown only if role === 'admin'
admin.html  → redirects to index.html if role !== 'admin'
```

### index.html Pages
| Page | Features |
|---|---|
| Home | Hero section · Stats · AI chatbot (💬 button) · Light/Dark mode |
| Courses | Load from Oracle via IS · Search · Filter by category · Sort · Enroll/Unenroll |
| My Learning | Real progress from Oracle · Lesson tracker · REPEAT/EXIT updateProgress |

### admin.html Tabs
| Tab | Features |
|---|---|
| Overview | Live stats from admin_stats view |
| Users | All users · Delete with confirm |
| Enrollments | Auto-refresh 10s · Progress bars · Completed badge |
| Courses | Add single (flat file) · Batch import (9-col CSV) |
| Live Events | Polls getAdminEvents every 5s · PROCESSED='N' only · Pause/Resume |

---

## Deployment Guide

### Prerequisites
- IBM webMethods IS 10.15 installed
- Oracle DB running (SAGAPPLICATION schema)
- `ojdbc8.jar` in `IntegrationServer/lib/jars/`

### Step-by-Step

**1. Start Oracle DB**
```
Windows Services → OracleServiceXE (or your service name) → Start
```

**2. Run Oracle SQL Scripts**
```sql
-- Run in SQL Developer — execute in this order:
-- 1. Create sequences (users_seq, enrollments_seq, courses_seq starting at 9)
-- 2. Create 5 tables
-- 3. Create courses_before_insert trigger
-- 4. Create admin_stats and enrollment_details views
-- 5. Insert 8 seed courses
-- 6. Insert admin user (role='admin')
-- 7. COMMIT
```

**3. Start IS**
```
C:\wMServiceDesigner\IntegrationServer\bin\startup.bat
```

**4. IS Admin Setup**
```
http://localhost:5555 (Administrator / manage)

Adapters → JDBC Adapter → jdbcConnection:
  Connection URL  : jdbc:oracle:thin:@localhost:1521:XE
  Username        : SAGAPPLICATION
  Transaction type: NO_TRANSACTION
  Auto-commit     : true
  → Enable connection

Packages → Management → LearnHub_Project → Reload
```

**5. Enable Triggers and Notifications**
```
IS Admin → Triggers:
  triggers/onCourseAdd    → Enable
  triggers/onEnrollment   → Enable

IS Admin → Adapter Notifications:
  enrollmentInsertNotification → Enable
```

**6. Create files/ folder**
```
mkdir C:\wMServiceDesigner\IntegrationServer\packages\LearnHub_Project\files\
```

**7. Update fileAccessControl.cnf**
```
C:\wMServiceDesigner\IntegrationServer\config\fileAccessControl.cnf
Add LearnHub_Project\files\ to all three allowedPaths
Restart IS after saving
```

**8. Deploy Frontend**
```
Copy index.html and admin.html to:
C:\wMServiceDesigner\IntegrationServer\packages\LearnHub_Project\pub\
```

**9. Set Gemini API Key**
```
Designer → courses:getChatbotResponse
→ pub.client:http step
→ URL field: append ?key=YOUR_GEMINI_API_KEY
```

**10. Configure Gmail SMTP**
```
Gmail → My Account → Security → App Passwords → Generate 16-char password
Designer → admin:onEnrollmentChange → pub.client:smtp step:
  smtpServer : smtp.gmail.com
  smtpPort   : 587
  login      : youremail@gmail.com
  password   : 16-char app password
```

---

## Configuration Reference

| Item | Value |
|---|---|
| IS Admin URL | http://localhost:5555 |
| IS Admin credentials | Administrator / manage |
| Student App URL | http://localhost:5555/LearnHub_Project/index.html |
| Admin Panel URL | http://localhost:5555/LearnHub_Project/admin.html |
| DB Schema | SAGAPPLICATION |
| JDBC Connection name | jdbcConnection |
| SMTP Server | smtp.gmail.com:587 |
| Scheduler | Complex Repeating, Hour=9, Min=0 |
| Adapter Notification polling | Every 10 seconds |
| Admin Live Events polling | Every 5 seconds |
| Password storage | Plain text — direct comparison in loginUser |
| Files path | packages/LearnHub_Project/files/ |

---

## IS Concepts Demonstrated

| Concept | Where Used |
|---|---|
| JDBC Adapter Services | 21 services (SELECT/INSERT/DELETE/MERGE/UPDATE SQL/CUSTOM SQL) |
| Adapter Notifications | enrollmentInsertNotification — polls every 10s |
| Publishable Document Types | courseDoc, enrollmentInsertNotificationPublishDocument |
| IS Triggers | onCourseAdd, onEnrollment |
| REPEAT / EXIT | updateProgress — count=3, label=retryLoop |
| Transaction Boundary | pub.art.transaction:startTransaction/commit/rollback |
| Flat File Schema | courseImportSchema — 10 fields, pub.flatFile:convertToValues |
| pub.client:http | Gemini AI — bytesToString + jsonStringToDocument |
| pub.client:smtp | Gmail — enrollment email + daily report |
| IS REST API Descriptor | restAPIs namespace, /learnhub/api base path |
| IS Scheduler | Complex Repeating at 09:00 daily |
| Java Service | utils:hashPassword — MessageDigest SHA-256 |
| pub.file:stringToFile | Write CSV to files/ folder |
| pub.math | divideFloats + multiplyFloats for progress calculation |
| pub.list:size | Check enrollment duplicates |
| pub.flow:return | Early exit on error conditions |

---

*LearnHub · IBM webMethods Integration Server 10.15 · Oracle SAGAPPLICATION · React 18*
