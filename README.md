# Online Examination System

A web-based examination platform built with **HTML, CSS, JavaScript, Core Java, JDBC and MySQL**. The application follows a three-tier architecture and supports teacher-side exam creation, student-side exam attempts, automatic evaluation, result storage, analytics and leaderboards.

## Table of Contents

- [Features](#features)
- [Technology Stack](#technology-stack)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Database Design](#database-design)
- [Database Features](#database-features)
- [API Endpoints](#api-endpoints)
- [Prerequisites](#prerequisites)
- [Setup and Installation](#setup-and-installation)
- [Run the Application](#run-the-application)
- [Demo Accounts](#demo-accounts)
- [Analytical Queries](#analytical-queries)
- [Security Notes](#security-notes)
- [Known Limitations](#known-limitations)

## Features

### Student

- Sign in through the login page.
- Browse examinations by subject and difficulty.
- View exam questions and multiple-choice options.
- Select, change or clear answers while attempting an exam.
- Submit responses and view the calculated result.
- Review submitted answers and results.
- View leaderboard rankings.
- Record events such as tab switching through the cheating-log endpoint.

### Teacher

- Sign in through the teacher login flow.
- View available subjects and examinations.
- Add subjects.
- Create examinations with a title, description, subject, duration, difficulty, pass mark and questions/options.
- View exam analytics, including attempts, average score, maximum score and pass count.

### Database and Analytics

- Store users, subjects, exams, questions, options, student responses, results, audit records and cheating logs.
- Automatically calculate and update results through database triggers.
- Apply negative marking to incorrect answers.
- Maintain audit information when examination questions are updated.
- Generate exam analytics through a stored procedure.
- Generate student rankings through a SQL view and the `RANK()` window function.
- Use JDBC transactions for related database operations.

## Technology Stack

| Layer | Technology |
|---|---|
| Frontend | HTML5, CSS3, JavaScript |
| Backend | Java 17, built-in `com.sun.net.httpserver.HttpServer` |
| Database | MySQL, InnoDB storage engine |
| Database connectivity | JDBC, MySQL Connector/J |
| JSON handling | `org.json` library |
| SQL features | Constraints, indexes, triggers, stored procedure, view, window function |

The backend uses Core Java and JDBC; it does not use Spring Boot or an ORM framework.

## Architecture

The project uses a three-tier architecture:

```text
+------------------------------------+
| Presentation Layer                 |
| HTML, CSS and JavaScript pages     |
| Login, Teacher Dashboard, Exam UI, |
| Leaderboard                        |
+----------------+-------------------+
                 | HTTP / JSON
                 v
+------------------------------------+
| Application Layer                  |
| Java HTTP Server                   |
| Server.java                        |
| ExamDAO.java                       |
| AnalyticsDAO.java                  |
| DBConnection.java                  |
+----------------+-------------------+
                 | JDBC / SQL
                 v
+------------------------------------+
| Data Layer                         |
| MySQL: online_exam_db              |
| Tables, constraints, triggers,     |
| stored procedure and view          |
+------------------------------------+
```

**Request flow:** The browser sends an HTTP request to a Java API endpoint. `Server.java` handles the request and calls the relevant DAO method. The DAO communicates with MySQL through JDBC. The backend returns the result as JSON for the frontend to display.

## Project Structure

```text
OnlineExamSystem/
├── backend/
│   ├── lib/
│   │   ├── json-20240303.jar
│   │   └── mysql-connector-j-26.10.0/
│   │       └── mysql-connector-j-26.10.0.jar
│   ├── out/                         # Compiled Java classes
│   └── src/
│       ├── api/
│       │   └── Server.java
│       ├── config/
│       │   └── DBConnection.java
│       └── dao/
│           ├── ExamDAO.java
│           └── AnalyticsDAO.java
├── database/
│   ├── schema.sql
│   ├── seed.sql
│   ├── upgrade_subjects.sql
│   └── advanced_dbms.sql
├── frontend/
│   ├── login.html
│   ├── teacher-dashboard.html
│   ├── student-exam.html
│   └── leaderboard.html
├── Launch-Website.bat
└── README.md
```

## Database Design

The database is named **`online_exam_db`** and contains nine core tables:

| Table | Purpose |
|---|---|
| `Users` | Stores student and teacher account information and roles. |
| `Subjects` | Stores examination subjects. |
| `Exams` | Stores exam details, subject, duration, difficulty and pass criteria. |
| `Questions` | Stores questions belonging to an examination. |
| `Options` | Stores multiple-choice options and identifies correct options. |
| `Student_Responses` | Stores the option selected by a student for each exam question. |
| `Results` | Stores the calculated score, total marks and pass status. |
| `Audit_Logs` | Stores records of tracked database changes. |
| `Cheating_Logs` | Stores examination-related events recorded for monitoring. |

The schema uses primary keys, foreign keys, unique constraints, check constraints, indexes and InnoDB. The `Student_Responses` table has a unique key on `(student_id, exam_id, question_id)` so a student's response for a question can be updated rather than duplicated. `Results` has a unique key on `(student_id, exam_id)`.

## Database Features

### Automatic evaluation and negative marking

The database triggers `after_response_insert` and `after_response_update` recalculate a student's accumulated score and update the corresponding `Results` record. The marking rule in the project is:

| Response | Score contribution |
|---|---:|
| Correct answer | Marks assigned to the question |
| Incorrect answer | -0.25 |
| Unanswered question | 0 |

The triggers use `INSERT ... ON DUPLICATE KEY UPDATE` to insert or update the student's result. The Java submission flow also contains result-recalculation logic in `ExamDAO.submitExam()`.

### Audit logging

The `after_question_update` trigger inserts an audit record when a question is updated. It records the question identifier and the old and new question text/marks values in `Audit_Logs`.

### Stored procedure: `GetExamAnalytics`

The stored procedure accepts an exam ID and returns:

- `total_attempts`
- `avg_score`
- `max_score`
- `pass_count`

Example:

```sql
CALL GetExamAnalytics(1);
```

`AnalyticsDAO.getAnalytics(int examId)` invokes this procedure through JDBC using `CallableStatement`.

### View: `Student_Leaderboard`

The view joins results with user and examination information and calculates a rank for each student within each exam using:

```sql
RANK() OVER (
    PARTITION BY r.exam_id
    ORDER BY r.score DESC
)
```

Example:

```sql
SELECT * FROM Student_Leaderboard;
```

`RANK()` assigns the same rank to tied scores and leaves a gap after a tie. The view selects display-related user information and does not expose the `password_hash` column.

## API Endpoints

The Java HTTP server runs on port **8080** and registers the following API contexts:

| Endpoint | Purpose |
|---|---|
| `/api/login` | Authenticates a user. |
| `/api/subjects` | Retrieves subjects and supports adding a subject. |
| `/api/exams` | Retrieves examinations. |
| `/api/questions` | Retrieves questions for an exam. |
| `/api/submit` | Submits exam responses and returns the result. |
| `/api/create-exam` | Creates an examination with questions and options. |
| `/api/log-cheating` | Records an examination-related event. |
| `/api/analytics` | Retrieves analytics for an exam. |
| `/api/leaderboard` | Retrieves exam-specific or global leaderboard data. |
| `/api/review` | Retrieves a student's exam review. |

Request parameters and JSON payloads depend on the endpoint and are handled by the corresponding handler in `backend/src/api/Server.java`.

## Prerequisites

- Java Development Kit (JDK) 17.
- MySQL Server (the included Windows launcher checks/starts the `MySQL80` service).
- MySQL Workbench or another MySQL client is recommended for importing and inspecting SQL scripts.
- A browser such as Chrome, Edge or Firefox.

## Setup and Installation

### 1. Create the database and tables

Open MySQL Workbench or a MySQL terminal and execute the scripts in this order from the project root:

```sql
SOURCE database/schema.sql;
SOURCE database/seed.sql;
SOURCE database/advanced_dbms.sql;
```

Alternatively, open each SQL file in MySQL Workbench and execute it in the same order.

- `schema.sql` creates the database and the nine tables.
- `seed.sql` inserts sample users, subjects and a sample DBMS examination with questions/options. Run this seed script once on a fresh database to avoid adding duplicate sample exam/question records.
- `advanced_dbms.sql` creates the response triggers, question audit trigger, `GetExamAnalytics` procedure and `Student_Leaderboard` view.
- `upgrade_subjects.sql` is an upgrade script for an older schema that does not yet have the Subjects-related structure. It is not required after creating a fresh database with the supplied `schema.sql`.

### 2. Configure database credentials

Open:

```text
backend/src/config/DBConnection.java
```

The current source uses the following local connection settings:

- Host: `localhost`
- Port: `3306`
- Database: `online_exam_db`
- Username: `root`
- Password: `root`

Update the username and password to match your local MySQL configuration before running the backend. Do not use hard-coded development credentials in a production deployment; use environment variables or protected configuration and a least-privilege database account.

### 3. Confirm Java and MySQL

Check Java:

```bash
java -version
```

Check that MySQL is running and that the `online_exam_db` database has been created successfully.

## Run the Application

### Windows — launcher script

From the project folder, double-click `Launch-Website.bat` or run it from Command Prompt:

```bat
Launch-Website.bat
```

The launcher attempts to start the `MySQL80` Windows service, opens the browser at `http://localhost:8080`, changes into the backend directory and starts the Java server using the included compiled classes and JAR dependencies.

If your MySQL service has a different Windows service name, start MySQL manually before running the launcher.

### Manual backend startup

From the project root, run:

**Windows:**

```bat
cd backend
java -cp "out;lib\mysql-connector-j-26.10.0\mysql-connector-j-26.10.0.jar;lib\json-20240303.jar" api.Server
```

**macOS / Linux:**

```bash
cd backend
java -cp "out:lib/mysql-connector-j-26.10.0/mysql-connector-j-26.10.0.jar:lib/json-20240303.jar" api.Server
```

The server prints a startup message when it is listening on port `8080`.

Open the application:

```text
http://localhost:8080
```

The root path serves `login.html`. The other frontend pages can be opened through the same server, for example:

- `http://localhost:8080/teacher-dashboard.html`
- `http://localhost:8080/student-exam.html`
- `http://localhost:8080/leaderboard.html`

Keep the Java server process running while using the application. Press `Ctrl+C` in the terminal to stop it.

## Demo Accounts

The supplied `seed.sql` inserts these sample accounts:

| Role | Email | Password |
|---|---|---|
| Teacher | `teacher@test.com` | `pass123` |
| Student | `student@test.com` | `pass123` |

These are development/demo credentials only. They are stored using the seed script's SHA-256 expression and must not be reused for real accounts or production deployment.

## Analytical Queries

Useful SQL commands for inspecting the database features:

```sql
USE online_exam_db;

-- Inspect the tables
SHOW TABLES;

-- Inspect stored procedures and triggers
SHOW PROCEDURE STATUS WHERE Db = 'online_exam_db';
SHOW TRIGGERS;

-- Run analytics for exam 1
CALL GetExamAnalytics(1);

-- Inspect the leaderboard
SELECT *
FROM Student_Leaderboard
ORDER BY exam_title, rank_position;

-- Inspect results and audit records
SELECT * FROM Results;
SELECT * FROM Audit_Logs;
```

## Security Notes

- The supplied `DBConnection.java` contains hard-coded local database credentials. Change these for your environment and avoid committing real credentials.
- The login query uses MySQL `SHA2(..., 256)`. For production systems, use a dedicated password-hashing algorithm such as Argon2id or bcrypt with per-user salts and appropriate work factors.
- Enforce role-based authorization and ownership checks on the server; hiding frontend controls is not sufficient authorization.
- Validate exam timing, submission windows and all user-supplied data on the server.
- Use HTTPS and restrict CORS appropriately before exposing the application beyond local development.

## Known Limitations

- The supplied project is configured as a local development application and uses a simple Java HTTP server.
- The launcher is Windows-specific; macOS and Linux users can start the backend using the manual command above.
- The project should be strengthened with production-grade authentication, authorization, configuration management, structured logging, automated integration tests, pagination and deployment/backup documentation before production use.

---

**Project:** Online Examination System  
**Architecture:** Three-tier (HTML/CSS/JavaScript + Core Java/JDBC + MySQL)  
**Default local URL:** `http://localhost:8080`
