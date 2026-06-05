# Oracle Attendance Machine (Stellar & zkbiotime) Data Synchronization Framework in Oracle APEX
 

This project provides an enterprise-grade Oracle PL/SQL synchronization framework for integrating attendance data from multiple biometric attendance platforms into Oracle Database.

Currently supported attendance systems:

* Stellar Attendance System
* ZKTeco ZKBioTime

The framework retrieves attendance transactions through REST APIs, processes JSON payloads, and stores attendance events into Oracle tables for HRMS, Payroll, Shift Management, and Attendance Analytics applications.

---

# System Architecture

```text
+-----------------------+
| Attendance Devices    |
| (Stellar / ZKTeco)    |
+-----------+-----------+
            |
            v
+-----------------------+
| Vendor REST APIs      |
+-----------+-----------+
            |
            v
+-----------------------+
| Oracle APEX           |
| APEX_WEB_SERVICE      |
+-----------+-----------+
            |
            v
+-----------------------+
| JSON Processing Layer |
| APEX_JSON             |
| JSON_TABLE            |
+-----------+-----------+
            |
            v
+-----------------------+
| STELLAR_RAW_DATA      |
+-----------+-----------+
            |
            v
+-----------------------+
| Attendance Engine     |
| HRMS / Payroll        |
+-----------------------+
```

---

# Key Features

## Multi-Vendor Attendance Support

Supports synchronization from:

* Stellar API
* ZKBioTime API

## Incremental Synchronization

The Stellar integration automatically retrieves only new attendance transactions using:

```sql
SELECT MAX(ACCESS_ID)
FROM STELLAR_RAW_DATA;
```

This reduces network traffic and improves synchronization performance.

---

# Database Structure

## Raw Attendance Table

```sql
CREATE TABLE STELLAR_RAW_DATA
(
    UNIT_ID           VARCHAR2(100),
    REGISTRATION_ID   VARCHAR2(100),
    EVENT_TIME        DATE,
    ACCESS_DATE       VARCHAR2(20),
    ACCESS_TIME       VARCHAR2(20),
    UNIT_NAME         VARCHAR2(200),
    ACCESS_ID         NUMBER,
    DEPARTMENT        VARCHAR2(200),
    USER_NAME         VARCHAR2(200),
    CARD              VARCHAR2(100),
    MACHINE_TYPE      VARCHAR2(30)
);
```

---

# Stellar Attendance Integration

## Synchronization Function

```plsql
FUNCTION STELLAR_DATA_SYNC_FUN
RETURN VARCHAR2;
```

## API Call

```plsql
vResult := APEX_WEB_SERVICE.MAKE_REST_REQUEST(
    p_url         => 'https://attendance-server/api/get_logs',
    p_http_method => 'GET',
    p_wallet_path => 'file:/u02/app/oracle/wallets/ssl_wallet'
);
```

## JSON Parsing

The Stellar API returns JSON data in the following format:

```json
{
  "log": [
    {
      "registration_id": "10303",
      "access_id": "19196763",
      "access_date": "2022-05-12",
      "access_time": "08:00:29"
    }
  ]
}
```

Oracle APEX JSON parser is used:

```plsql
APEX_JSON.PARSE(
    p_values => l_json_values,
    p_source => vResult
);
```

## Attendance Extraction

```plsql
l_registration_id :=
APEX_JSON.GET_VARCHAR2(
    p_path   => 'log[%d].registration_id',
    p0       => l_count,
    p_values => l_json_values
);
```

## Data Persistence

```plsql
INSERT INTO STELLAR_RAW_DATA
(
    UNIT_ID,
    REGISTRATION_ID,
    EVENT_TIME,
    ACCESS_ID,
    MACHINE_TYPE
)
VALUES
(
    l_unit_id,
    l_registration_id,
    l_event_time,
    l_access_id,
    'STELLAR'
);
```

---

# ZKBioTime Integration

## Synchronization Function

```plsql
FUNCTION ZKBIOTIME_DATA_SYNC_FUN
RETURN VARCHAR2;
```

## API Call

```plsql
vResult := APEX_WEB_SERVICE.MAKE_REST_REQUEST(
    p_url         => 'https://cloudsolution.com/zkbiotime/data.php',
    p_http_method => 'GET',
    p_wallet_path => 'file:/u02/app/oracle/wallets/ssl_wallet'
);
```

---

## ZKBioTime JSON Format

```json
[
  {
    "id":"1",
    "emp_code":"5",
    "punch_time":"2022-07-21 18:42:59.000000",
    "terminal_sn":"CEDD202660094"
  }
]
```

---

## JSON_TABLE Processing

```plsql
SELECT
       j.id,
       j.emp_code,
       j.punch_time,
       j.terminal_sn
FROM JSON_TABLE
(
    vResult,
    '$[*]'
    COLUMNS
    (
        id          NUMBER       PATH '$.id',
        emp_code    VARCHAR2(20) PATH '$.emp_code',
        punch_time  VARCHAR2(50) PATH '$.punch_time',
        terminal_sn VARCHAR2(20) PATH '$.terminal_sn'
    )
) j;
```

---

## Attendance Storage

```plsql
INSERT INTO STELLAR_RAW_DATA
(
    UNIT_ID,
    REGISTRATION_ID,
    EVENT_TIME,
    ACCESS_ID,
    MACHINE_TYPE
)
VALUES
(
    rec.terminal_sn,
    rec.emp_code,
    TO_DATE(
        SUBSTR(rec.punch_time,1,19),
        'YYYY-MM-DD HH24:MI:SS'
    ),
    rec.id,
    'ZKTECO'
);
```

---

# Oracle Wallet Configuration

The integrations use Oracle Wallet for SSL communication.

Example:

```plsql
p_wallet_path => 'file:/u02/app/oracle/wallets/ssl_wallet'
```

Benefits:

* HTTPS Security
* SSL Certificate Validation
* Secure REST Communication

---

# Scheduling

Synchronization can be automated using Oracle Scheduler.

Example:

```plsql
BEGIN
    DBMS_SCHEDULER.CREATE_JOB(
        job_name        => 'ATTENDANCE_SYNC_JOB',
        job_type        => 'PLSQL_BLOCK',
        job_action      => '
        BEGIN
            STELLAR_DATA_SYNC_FUN;
            ZKBIOTIME_DATA_SYNC_FUN;
        END;',
        repeat_interval => 'FREQ=MINUTELY;INTERVAL=5',
        enabled         => TRUE
    );
END;
/
```

---

# Performance Recommendations

## Recommended Index

```sql
CREATE INDEX IDX_STELLAR_ACCESS_ID
ON STELLAR_RAW_DATA (ACCESS_ID);

CREATE INDEX IDX_STELLAR_EVENT_TIME
ON STELLAR_RAW_DATA (EVENT_TIME);

CREATE INDEX IDX_STELLAR_REG_ID
ON STELLAR_RAW_DATA (REGISTRATION_ID);
```

## Recommended Enhancements

* Bulk Insert Processing
* MERGE Instead of INSERT
* Attendance Deduplication
* Error Logging Table
* Retry Mechanism
* API Monitoring Dashboard

---

# Oracle APEX Integration

Attendance data can be consumed by:

* Interactive Reports
* Interactive Grids
* Attendance Dashboards
* Payroll Processing Modules
* Shift Management Systems
* Employee Self-Service Portals

---

# Business Benefits

* Real-Time Attendance Collection
* Centralized Attendance Repository
* Automated Payroll Integration
* Reduced Manual Processing
* Multi-Device Support
* Enterprise Scalability

---

# Author

Sanjay Sikder

Oracle APEX Professional Developer

Expertise:

* Oracle Database 19c / 21c
* Oracle APEX 22.x / 24.x
* REST API Integrations
* HRMS Solutions
* Payroll Systems
* Enterprise Attendance Management

---

# License

MIT License

This project is intended for Oracle Database and Oracle APEX developers implementing enterprise attendance synchronization solutions.
