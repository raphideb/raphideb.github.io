---
title: "Oracle on Kubernetes"
type: "docs"
weight: 4
description: "Oracle Database 23c Free Edition deployment"
---

## Oracle Database Operator

After deploying Oracle Database using the scripts from <a href="https://github.com/raphideb/kube" target="_blank">github.com/raphideb/kube</a>, you have a fully usable Oracle Database 23c Free Edition running in Kubernetes, managed by the Oracle Database Operator.

The deployment includes:

- **Oracle Database Operator** - Kubernetes-native Oracle management
- **Oracle 23c Free Edition** - Latest Oracle database features
- **Single Instance Database** - Suitable for development and testing
- **Grafana integration** - Custom monitoring dashboard with Oracle exporter

## Connecting to Oracle

### Using orasql Script (Recommended)

The repository includes a helper script for easy access:

```bash
./orasql
```

Output shows available databases:

```
Usage: orasql <database-name> [namespace]

Available databases:
oracle23    Healthy   FREE
myapp-db    Healthy   FREE
```

Connect to a database:

```bash
./orasql oracle23
```

This automatically finds the pod and connects with SQL*Plus.

### Manual Connection

Get the pod name:

```bash
kubectl get singleinstancedatabase -n oracle
kubectl get pods -n oracle
```

Connect with kubectl exec:

```bash
kubectl exec -it oracle23-abcde -n oracle -- sqlplus sys/YourPassword_123@FREE as sysdba
```

### Port Forwarding

Forward Oracle port to localhost:

```bash
kubectl port-forward -n oracle pod/oracle23-abcde 1521:1521
```

Connect from another terminal with SQL*Plus or SQL Developer:

```
Host: localhost
Port: 1521
SID: FREE
User: sys
Password: YourPassword_123
Role: SYSDBA
```

## Deploying Multiple Instances

### Deploy Additional Database

The repository includes a deployment script:

```bash
./deploy_oracle.sh myapp-db FREE MySecurePass_456
```

This creates a new Oracle instance with:

- Database name: myapp-db
- SID: FREE (required for Free Edition)
- Password: MySecurePass_456
- Grafana monitoring enabled

Verify deployment:

```bash
kubectl get singleinstancedatabase -n oracle
kubectl get pods -n oracle
```

### Connect to New Instance

```bash
./orasql myapp-db
```

## Basic Database Operations

### Create User and Schema

Connect to the database:

```bash
./orasql oracle23

CREATE USER myapp IDENTIFIED BY AppPassword_123;
GRANT CONNECT, RESOURCE TO myapp;
GRANT CREATE SESSION TO myapp;
GRANT UNLIMITED TABLESPACE TO myapp;
```

Connect as new user:

```sql
CONNECT myapp/AppPassword_123@FREE
```

### Create Tables

```sql
CREATE TABLE employees (
  id NUMBER PRIMARY KEY,
  first_name VARCHAR2(50),
  last_name VARCHAR2(50),
  email VARCHAR2(100) UNIQUE,
  hire_date DATE DEFAULT SYSDATE
);

CREATE SEQUENCE emp_seq START WITH 1 INCREMENT BY 1;

CREATE OR REPLACE TRIGGER emp_bir
BEFORE INSERT ON employees
FOR EACH ROW
BEGIN
  :new.id := emp_seq.NEXTVAL;
END;
/
```

### Insert Data

```sql
INSERT INTO employees (first_name, last_name, email)
VALUES ('Alice', 'Smith', 'alice@example.com');

INSERT INTO employees (first_name, last_name, email)
VALUES ('Bob', 'Jones', 'bob@example.com');

COMMIT;

SELECT * FROM employees;
```

## Oracle 23c New Features

### JSON Relational Duality

Create a duality view for JSON and SQL access:

```sql
CREATE TABLE customers (
  customer_id NUMBER PRIMARY KEY,
  name VARCHAR2(100),
  email VARCHAR2(100)
);

CREATE TABLE orders (
  order_id NUMBER PRIMARY KEY,
  customer_id NUMBER REFERENCES customers(customer_id),
  order_date DATE,
  total NUMBER(10,2)
);

-- Create JSON Relational Duality View
CREATE JSON RELATIONAL DUALITY VIEW customer_orders_dv AS
customers @insert @update @delete
{
  customer_id: customer_id,
  name: name,
  email: email,
  orders: orders @insert @update @delete
  [
    {
      order_id: order_id,
      order_date: order_date,
      total: total
    }
  ]
};
```

Query as JSON:

```sql
SELECT json_serialize(data) FROM customer_orders_dv;
```

### SQL Domains

Define reusable domains:

```sql
CREATE DOMAIN email_domain AS VARCHAR2(100)
  CONSTRAINT email_check CHECK (VALUE LIKE '%@%.%');

CREATE TABLE users (
  user_id NUMBER PRIMARY KEY,
  username VARCHAR2(50),
  user_email email_domain
);

-- This will fail validation
INSERT INTO users VALUES (1, 'test', 'invalid-email');

-- This will succeed
INSERT INTO users VALUES (1, 'test', 'user@example.com');
```

## Performance Tuning

### View Execution Plans

```sql
EXPLAIN PLAN FOR
SELECT * FROM employees WHERE email = 'alice@example.com';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

### Create Index

```sql
CREATE INDEX idx_emp_email ON employees(email);
```

### Gather Statistics

```sql
EXEC DBMS_STATS.GATHER_TABLE_STATS('MYAPP', 'EMPLOYEES');
```

### Check Index Usage

```sql
SELECT index_name, table_name, uniqueness
FROM user_indexes
WHERE table_name = 'EMPLOYEES';
```

## Monitoring and Troubleshooting

### Check Database Status

```bash
kubectl get singleinstancedatabase -n oracle
```

Output shows health status:

```
NAME        STATUS    SID
oracle23    Healthy   FREE
myapp-db    Healthy   FREE
```

### View Pod Logs

```bash
kubectl logs -f oracle23-abcde -n oracle
```

### View Events

```bash
kubectl get events -n oracle --sort-by='.lastTimestamp'
```

### Resource Usage

```bash
kubectl top pods -n oracle
```

### Check Database Uptime

```bash
./orasql oracle23

SELECT instance_name, status, startup_time
FROM v$instance;
```

### View Active Sessions

```sql
SELECT username, status, osuser, machine
FROM v$session
WHERE username IS NOT NULL
ORDER BY username;
```

### Check Tablespace Usage

```sql
SELECT tablespace_name,
       ROUND(used_space * 8192 / 1024 / 1024, 2) AS used_mb,
       ROUND(tablespace_size * 8192 / 1024 / 1024, 2) AS size_mb,
       ROUND(used_percent, 2) AS used_percent
FROM dba_tablespace_usage_metrics
ORDER BY used_percent DESC;
```

## Grafana Monitoring

### Access Dashboard

Open Grafana:

http://\<your-host-ip\>:30000

Import the custom Oracle dashboard from the repository:

```
OracleDB_Grafana.json
```

This modified dashboard allows selecting databases by name instead of IP address.

### Metrics Available

- Active sessions
- Tablespace usage
- Database availability
- SQL execution statistics
- Buffer cache hit ratio
- Memory usage (PGA/SGA)
- Wait events

## Backup and Recovery

### Export Data

Export a schema:

```bash
kubectl exec oracle23-abcde -n oracle -- expdp system/YourPassword_123@FREE \
  schemas=MYAPP \
  directory=DATA_PUMP_DIR \
  dumpfile=myapp_backup.dmp \
  logfile=myapp_backup.log
```

### Import Data

Import to another database:

```bash
kubectl exec myapp-db-fghij -n oracle -- impdp system/MySecurePass_456@FREE \
  schemas=MYAPP \
  directory=DATA_PUMP_DIR \
  dumpfile=myapp_backup.dmp \
  logfile=myapp_restore.log \
  remap_schema=MYAPP:NEWAPP
```

### RMAN Backup

Connect as SYSDBA:

```bash
./orasql oracle23

RMAN TARGET /

BACKUP DATABASE PLUS ARCHIVELOG;
LIST BACKUP;
```

## PL/SQL Development

### Create Stored Procedure

```sql
CREATE OR REPLACE PROCEDURE add_employee (
  p_first_name VARCHAR2,
  p_last_name VARCHAR2,
  p_email VARCHAR2
) AS
BEGIN
  INSERT INTO employees (first_name, last_name, email)
  VALUES (p_first_name, p_last_name, p_email);
  COMMIT;
EXCEPTION
  WHEN OTHERS THEN
    ROLLBACK;
    RAISE;
END;
/
```

### Execute Procedure

```sql
EXEC add_employee('Charlie', 'Brown', 'charlie@example.com');
```

### Create Function

```sql
CREATE OR REPLACE FUNCTION get_employee_count
RETURN NUMBER
IS
  v_count NUMBER;
BEGIN
  SELECT COUNT(*) INTO v_count FROM employees;
  RETURN v_count;
END;
/
```

### Use Function

```sql
SELECT get_employee_count() AS total_employees FROM dual;
```

## Cleanup

### Delete Single Database

```bash
./del_oracle.sh
```

Select the database to delete from the list.

### Manual Deletion

```bash
kubectl delete singleinstancedatabase oracle23 -n oracle
```

This deletes the pod and service but preserves PVC (data).

### Delete Everything Including Data

```bash
kubectl delete singleinstancedatabase oracle23 -n oracle
kubectl delete pvc -n oracle oracle23-pvc
```

## Useful SQL Queries

### Database Size

```sql
SELECT SUM(bytes)/1024/1024 AS size_mb
FROM dba_data_files;
```

### Top SQL by Executions

```sql
SELECT sql_id, executions, elapsed_time/1000000 AS elapsed_sec
FROM v$sql
ORDER BY executions DESC
FETCH FIRST 10 ROWS ONLY;
```

### Table Sizes

```sql
SELECT segment_name, bytes/1024/1024 AS size_mb
FROM user_segments
WHERE segment_type = 'TABLE'
ORDER BY bytes DESC
FETCH FIRST 10 ROWS ONLY;
```

### Invalid Objects

```sql
SELECT object_name, object_type, status
FROM user_objects
WHERE status = 'INVALID';
```

### Locked Objects

```sql
SELECT l.session_id, o.object_name, l.locked_mode
FROM v$locked_object l
JOIN dba_objects o ON l.object_id = o.object_id;
```

## Free Edition Limitations

Oracle Database 23c Free Edition has these limitations:

- Maximum 2GB RAM for database
- Maximum 2 CPU threads
- Maximum 12GB user data
- Single instance only (no RAC)

Perfect for development, testing, and learning Oracle features in Kubernetes.

## Network Policies

Restrict access to Oracle using Calico network policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: oracle-access
  namespace: oracle
spec:
  podSelector:
    matchLabels:
      app: oracle23
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: myapp
    ports:
    - protocol: TCP
      port: 1521
```

Apply it:

```bash
kubectl apply -f oracle-policy.yaml
```

Now only pods labeled with `app: myapp` can connect to Oracle.

## Getting Started

For installation instructions, deployment scripts, and configuration details, see the README files in the <a href="https://github.com/raphideb/kube" target="_blank">script repository</a>.

