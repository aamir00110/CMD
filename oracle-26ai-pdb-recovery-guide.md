# Oracle 26AI Pluggable Database Recovery Guide
## Complete Reference for All Recovery Types

---

## Table of Contents
1. [Prerequisites & Setup](#prerequisites--setup)
2. [Full Database Recovery](#full-database-recovery)
3. [Point-in-Time Recovery (PITR)](#point-in-time-recovery-pitr)
4. [Datafile Recovery](#datafile-recovery)
5. [Tablespace Recovery](#tablespace-recovery)
6. [Archivelog Mode Setup](#archivelog-mode-setup)
7. [RMAN Backup Strategy](#rman-backup-strategy)
8. [Common Recovery Scenarios](#common-recovery-scenarios)
9. [Monitoring & Verification](#monitoring--verification)

---

## Prerequisites & Setup

### Required Background
- Oracle 26AI installed and running
- RMAN (Recovery Manager) configured
- Archivelog mode enabled (for point-in-time recovery)
- Sufficient disk space for backups and recovery
- Appropriate database and OS user privileges

### Key Files & Locations
```
$ORACLE_BASE/diag/rdbms/<db_name>/<pdb_name>/trace/     # Alert logs
$ORACLE_HOME/dbs/                                        # Initialization files
/u01/oradata/<db_name>/                                 # Data files
/u01/archive_logs/                                       # Archive logs
```

### Database Connection Setup
```sql
-- Connect to PDB as SYSDBA
sqlplus / as sysdba

-- Set container to PDB (if using multitenant)
ALTER SESSION SET CONTAINER=pdb_name;

-- Verify database mode
SELECT NAME, OPEN_MODE, ARCHIVELOG FROM V$DATABASE;

-- Check backup status
SELECT * FROM V$BACKUP;
```

---

## Full Database Recovery

### Scenario: Complete database loss or corruption

#### Step 1: Determine Failure Point

```sql
-- Connect to PDB
SQLPLUS / AS SYSDBA

-- Check current status
SELECT NAME, OPEN_MODE, ARCHIVELOG, PROTECTION_MODE 
FROM V$DATABASE;

-- Find last archived log
SELECT SEQUENCE#, FIRST_TIME, NEXT_TIME 
FROM V$ARCHIVED_LOG 
ORDER BY SEQUENCE# DESC 
FETCH FIRST 1 ROW ONLY;

-- Check for redo logs
SELECT GROUP#, STATUS, MEMBERS, ARCHIVED 
FROM V$LOG 
ORDER BY GROUP#;
```

#### Step 2: Take Database Offline

```sql
-- Shutdown database immediately
SHUTDOWN ABORT;

-- Verify shutdown
SELECT INSTANCE_NAME, STATUS FROM V$INSTANCE;
```

#### Step 3: RMAN Full Recovery

```bash
# Start RMAN session
rman target / catalog rman_user/password@catalog_db

# Connect to PDB specifically
RMAN> CONNECT TARGET sys/password@pdb_name
RMAN> CONNECT CATALOG rman_user/password@catalog_db
```

##### Option A: Restore & Recover with RMAN

```sql
RMAN> STARTUP MOUNT;

-- Restore all datafiles from backup
RMAN> RESTORE DATABASE;

-- Recover database to current point
RMAN> RECOVER DATABASE;

-- Open database with reset logs (after recovery)
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

##### Option B: Recover to Specific SCN

```sql
RMAN> STARTUP MOUNT;

RMAN> SET UNTIL SCN = 12345678;

RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;

RMAN> ALTER DATABASE OPEN RESETLOGS;
```

#### Step 4: Verify Recovery

```sql
-- Verify database is open
SELECT NAME, OPEN_MODE, DATABASE_ROLE, PROTECTION_MODE
FROM V$DATABASE;

-- Check for errors
SELECT NAME, VALUE 
FROM V$PARAMETER 
WHERE NAME LIKE '%audit%';

-- Verify datafiles
SELECT FILE#, NAME, STATUS, ENABLED
FROM V$DATAFILE
ORDER BY FILE#;

-- Check archive logs
SELECT COUNT(*) AS total_archive_logs
FROM V$ARCHIVED_LOG;

-- Run data integrity check
DBMS_REPAIR.CHECK_OBJECT(
  schema_name => 'USERS',
  object_name => 'table_name',
  object_type => dbms_repair.table_object
);
```

---

## Point-in-Time Recovery (PITR)

### Scenario: Recover database to a specific moment before data loss

#### Step 1: Identify Target Recovery Time

```sql
-- Find exact SCN at time of interest
SELECT NAME, TIME_DP, CHANGE# 
FROM V$LOG_HISTORY 
WHERE TRUNC(TIME_DP) = TRUNC(SYSDATE)
ORDER BY CHANGE# DESC;

-- Or use log sequence number
SELECT SEQUENCE#, FIRST_TIME, NEXT_TIME, ARCHIVED
FROM V$LOG
ORDER BY SEQUENCE#;

-- Check if logs are available
SELECT NAME, COMPLETION_TIME, BLOCKS, BLOCK_SIZE
FROM V$ARCHIVED_LOG
WHERE COMPLETION_TIME BETWEEN 
  TO_DATE('2024-01-15 10:00:00', 'YYYY-MM-DD HH24:MI:SS') 
  AND TO_DATE('2024-01-15 14:00:00', 'YYYY-MM-DD HH24:MI:SS')
ORDER BY SEQUENCE#;
```

#### Step 2: RMAN PITR Setup

```bash
rman target / catalog rman_user/password@catalog_db
```

##### Option A: Recovery Using Timestamp

```sql
RMAN> STARTUP MOUNT;

-- Set recovery target to specific time
RMAN> SET UNTIL TIME "to_date('2024-01-15 10:30:00','YYYY-MM-DD HH24:MI:SS')";

RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;

RMAN> ALTER DATABASE OPEN RESETLOGS;
```

##### Option B: Recovery Using SCN

```sql
RMAN> STARTUP MOUNT;

-- Find correct SCN
RMAN> SET UNTIL SCN = 98765432;

RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;

RMAN> ALTER DATABASE OPEN RESETLOGS;
```

##### Option C: Recovery Using Log Sequence Number

```sql
RMAN> STARTUP MOUNT;

RMAN> SET UNTIL SEQUENCE 5432 THREAD 1;

RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;

RMAN> ALTER DATABASE OPEN RESETLOGS;
```

#### Step 3: Validate PITR Data

```sql
-- Check recovery time
SELECT CURRENT_TIMESTAMP FROM DUAL;

-- Verify restored data
SELECT COUNT(*) FROM critical_table;

-- Compare row counts if backup available
SELECT TABLE_NAME, NUM_ROWS
FROM USER_TABLES
WHERE TABLE_NAME = 'CRITICAL_TABLE';

-- Check for recent changes
SELECT * FROM AUDIT_LOG
WHERE CREATION_TIME > SYSDATE - 1
ORDER BY CREATION_TIME DESC;
```

---

## Datafile Recovery

### Scenario: Single or multiple datafiles are corrupted or missing

#### Step 1: Identify Failed Datafiles

```sql
-- Check datafile status
SELECT FILE#, NAME, STATUS, BYTES
FROM V$DATAFILE
WHERE STATUS != 'AVAILABLE';

-- Check alert logs
SELECT * FROM V$ALERT_TYPES;

-- Query tablespace health
SELECT TABLESPACE_NAME, STATUS, EXTENT_MANAGEMENT
FROM DBA_TABLESPACES;
```

#### Step 2: Recovery Process

##### Option A: Recover While Database Online (RMAN)

```bash
rman target /
```

```sql
-- Database remains open during recovery
RMAN> RECOVER DATAFILE 4, 7, 9;
```

##### Option B: Recover With Database Offline

```sql
-- Mount database
SQLPLUS / AS SYSDBA
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;

-- Restore specific datafiles
RMAN> RESTORE DATAFILE 4, 7;
RMAN> RECOVER DATAFILE 4, 7;

-- Open database
RMAN> ALTER DATABASE OPEN;
```

##### Option C: Recover Entire Tablespace

```sql
RMAN> RECOVER TABLESPACE users_ts;

-- Or multiple tablespaces
RMAN> RECOVER TABLESPACE users_ts, temp_ts, undo_ts;
```

#### Step 3: Verify Datafile Recovery

```sql
-- Check datafile status
SELECT FILE#, NAME, STATUS, BYTES/1024/1024 AS SIZE_MB
FROM V$DATAFILE
ORDER BY FILE#;

-- Verify tablespace consistency
ANALYZE TABLE table_name COMPUTE STATISTICS;

-- Check for bad blocks
SELECT FILE#, BLOCK#, CREATION_TIME
FROM V$DATABASE_BLOCK_CORRUPTION;

-- Run integrity check
BEGIN
  DBMS_REPAIR.CHECK_OBJECT(
    schema_name => 'SCHEMA_NAME',
    object_name => 'TABLE_NAME',
    object_type => DBMS_REPAIR.TABLE_OBJECT
  );
END;
/
```

---

## Tablespace Recovery

### Scenario: Specific tablespace needs recovery without full database recovery

#### Step 1: Identify Tablespace Issues

```sql
-- List all tablespaces
SELECT TABLESPACE_NAME, STATUS, EXTENT_MANAGEMENT, SEGMENT_SPACE_MANAGEMENT
FROM DBA_TABLESPACES
ORDER BY TABLESPACE_NAME;

-- Check tablespace datafiles
SELECT FILE#, FILE_NAME, TABLESPACE_NAME, STATUS, BYTES/1024/1024 AS SIZE_MB
FROM DBA_DATA_FILES
WHERE TABLESPACE_NAME = 'USERS'
ORDER BY FILE#;

-- Check for offline datafiles
SELECT NAME, STATUS
FROM V$DATAFILE
WHERE STATUS = 'OFFLINE';
```

#### Step 2: Offline Tablespace

```sql
-- Take tablespace offline (immediate = no checkpoint)
ALTER TABLESPACE users OFFLINE IMMEDIATE;

-- Verify offline status
SELECT TABLESPACE_NAME, STATUS
FROM DBA_TABLESPACES
WHERE TABLESPACE_NAME = 'USERS';
```

#### Step 3: RMAN Tablespace Recovery

```bash
rman target /
```

```sql
-- Restore tablespace
RMAN> RESTORE TABLESPACE users;

-- Recover tablespace
RMAN> RECOVER TABLESPACE users;
```

#### Step 4: Bring Tablespace Online

```sql
-- Online tablespace
ALTER TABLESPACE users ONLINE;

-- Verify status
SELECT TABLESPACE_NAME, STATUS, EXTENT_MANAGEMENT
FROM DBA_TABLESPACES
WHERE TABLESPACE_NAME = 'USERS';

-- Check available free space
SELECT TABLESPACE_NAME, SUM(BYTES)/1024/1024 AS FREE_MB
FROM DBA_FREE_SPACE
WHERE TABLESPACE_NAME = 'USERS'
GROUP BY TABLESPACE_NAME;
```

#### Step 5: Validation

```sql
-- Verify table data
SELECT COUNT(*) FROM table_in_recovered_ts;

-- Check segment information
SELECT SEGMENT_NAME, SEGMENT_TYPE, BYTES/1024/1024 AS SIZE_MB
FROM DBA_SEGMENTS
WHERE TABLESPACE_NAME = 'USERS'
ORDER BY BYTES DESC;
```

---

## Archivelog Mode Setup

### Enable Archivelog Mode (Required for PITR)

#### Step 1: Check Current Mode

```sql
-- Connect as SYSDBA
SQLPLUS / AS SYSDBA

-- Check archivelog status
SELECT NAME, LOG_MODE, ARCHIVELOG FROM V$DATABASE;

-- Check redo log configuration
SELECT GROUP#, THREAD#, SEQUENCE#, ARCHIVED, STATUS
FROM V$LOG
ORDER BY GROUP#;
```

#### Step 2: Enable Archivelog Mode (if disabled)

```sql
-- Shutdown database
SHUTDOWN IMMEDIATE;

-- Startup in mount mode
STARTUP MOUNT;

-- Enable archivelog
ALTER DATABASE ARCHIVELOG;

-- Open database
ALTER DATABASE OPEN;

-- Verify
SELECT NAME, LOG_MODE, ARCHIVELOG FROM V$DATABASE;
```

#### Step 3: Configure Archive Destination

```sql
-- Set archive log destination
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1='LOCATION=/u01/archive_logs VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=prod_db' SCOPE=BOTH;

-- Set archive format
ALTER SYSTEM SET LOG_ARCHIVE_FORMAT='%t_%s_%r.dbf' SCOPE=BOTH;

-- Enable archiving
ALTER SYSTEM SET ARCHIVE_LAG_TARGET=600 SCOPE=BOTH;

-- Verify settings
SHOW PARAMETER LOG_ARCHIVE;
```

#### Step 4: Monitor Archive Process

```sql
-- Check archive status
SELECT * FROM V$ARCHIVE_PROCESSES;

-- Check archived logs
SELECT SEQUENCE#, FIRST_TIME, NEXT_TIME, ARCHIVED, DELETED, STATUS
FROM V$LOG
ORDER BY SEQUENCE# DESC;

-- Monitor archive destination
SELECT NAME, DESTINATION, TRANSMIT_MODE, AFFIRM, DB_UNIQUE_NAME, STATUS
FROM V$ARCHIVE_DEST
WHERE DESTINATION IS NOT NULL;
```

---

## RMAN Backup Strategy

### Configure RMAN for Reliable Recovery

#### Step 1: RMAN Configuration

```bash
# Connect to RMAN
rman target /
```

```sql
-- Set backup location
CONFIGURE CHANNEL DEVICE TYPE DISK FORMAT '/u01/rman_backup/%d_%I_%T_%s.bkp';

-- Set default device type
CONFIGURE DEFAULT DEVICE TYPE TO DISK;

-- Set retention policy
CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 30 DAYS;

-- Enable controlfile autobackup
CONFIGURE CONTROLFILE AUTOBACKUP ON;
CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '/u01/rman_backup/ctl_%d_%T.bkp';

-- Show configuration
SHOW ALL;
```

#### Step 2: Full Database Backup

```sql
RMAN> RUN {
  ALLOCATE CHANNEL c1 DEVICE TYPE DISK;
  ALLOCATE CHANNEL c2 DEVICE TYPE DISK;
  
  -- Full database backup
  BACKUP DATABASE PLUS ARCHIVELOG;
  
  -- Backup controlfile
  BACKUP CURRENT CONTROLFILE FORMAT '/u01/rman_backup/control_%T.bkp';
  
  RELEASE CHANNEL c1;
  RELEASE CHANNEL c2;
}
```

#### Step 3: Incremental Backup Strategy

```sql
-- Level 0 (full backup)
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE;

-- Level 1 (since last level 0)
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE;

-- Level 1 cumulative (since last level 0)
RMAN> BACKUP INCREMENTAL LEVEL 1 CUMULATIVE DATABASE;
```

#### Step 4: Verify Backups

```sql
-- List all backup pieces
RMAN> LIST BACKUP SUMMARY;

-- Check backup details
RMAN> LIST BACKUP OF DATABASE;

-- Validate backup
RMAN> VALIDATE BACKUP;

-- Restore validation (test without actual restore)
RMAN> RESTORE DATABASE VALIDATE;
```

---

## Common Recovery Scenarios

### Scenario 1: Accidental Data Deletion

```sql
-- Find deletion time from audit logs
SELECT * FROM DBA_AUDIT_TRAIL
WHERE ACTION_NAME = 'DELETE'
AND TIMESTAMP >= TRUNC(SYSDATE)
ORDER BY TIMESTAMP DESC;

-- Connect to RMAN
rman target /

-- Recover to time just before deletion
RMAN> SET UNTIL TIME "to_date('2024-01-15 10:45:00','YYYY-MM-DD HH24:MI:SS')";
RMAN> RESTORE DATABASE;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

### Scenario 2: Corrupted Redo Logs

```sql
-- Identify bad redo logs
SELECT GROUP#, STATUS, ARCHIVED FROM V$LOG;

-- Clear bad log
ALTER DATABASE CLEAR LOGFILE GROUP 2;

-- Re-enable logging
ALTER DATABASE OPEN;

-- Check status
SELECT GROUP#, STATUS FROM V$LOG;
```

### Scenario 3: Lost Control File

```bash
# Stop database
SHUTDOWN ABORT;

# Copy backup controlfile from RMAN backup location
cp /u01/rman_backup/control_*.bkp /u01/oradata/prod_db/control01.ctl

# Restore from RMAN
rman target /
```

```sql
RMAN> STARTUP NOMOUNT;
RMAN> RESTORE CONTROLFILE FROM AUTOBACKUP;
RMAN> ALTER DATABASE MOUNT;
RMAN> RECOVER DATABASE;
RMAN> ALTER DATABASE OPEN RESETLOGS;
```

### Scenario 4: Tempfile Loss

```sql
-- Check tempfile status
SELECT FILE#, NAME, STATUS FROM V$TEMPFILE;

-- Drop and recreate tempfile
ALTER TABLESPACE TEMP DROP TEMPFILE '/u01/oradata/prod_db/temp01.dbf';

ALTER TABLESPACE TEMP ADD TEMPFILE '/u01/oradata/prod_db/temp01.dbf' 
SIZE 100M AUTOEXTEND ON NEXT 100M MAXSIZE UNLIMITED;

-- Verify
SELECT FILE#, NAME, STATUS FROM V$TEMPFILE;
```

### Scenario 5: Undo Tablespace Corruption

```sql
-- Check undo status
SELECT TABLESPACE_NAME, STATUS FROM DBA_TABLESPACES
WHERE TABLESPACE_NAME LIKE 'UNDO%';

-- Take offline
ALTER TABLESPACE undotbs1 OFFLINE;

-- Recover from RMAN
RMAN> RESTORE TABLESPACE undotbs1;
RMAN> RECOVER TABLESPACE undotbs1;

-- Bring online
ALTER TABLESPACE undotbs1 ONLINE;
```

---

## Monitoring & Verification

### Post-Recovery Validation

#### 1. Database Status Check

```sql
-- Verify database mode
SELECT NAME, OPEN_MODE, LOG_MODE, DATABASE_ROLE FROM V$DATABASE;

-- Check instance status
SELECT INSTANCE_NAME, STATUS, ARCHIVER FROM V$INSTANCE;

-- Verify protection mode (for Data Guard)
SELECT PROTECTION_MODE, STANDBY_BECAME_PRIMARY_SCN FROM V$DATABASE;
```

#### 2. Data Integrity Verification

```sql
-- Check for invalid objects
SELECT OBJECT_TYPE, COUNT(*) AS INVALID_COUNT
FROM DBA_OBJECTS
WHERE STATUS = 'INVALID'
GROUP BY OBJECT_TYPE;

-- Compile invalid objects
ALTER SESSION SET CURRENT_SCHEMA = schema_name;
BEGIN
  FOR obj IN (SELECT OBJECT_NAME FROM DBA_OBJECTS WHERE STATUS = 'INVALID') LOOP
    EXECUTE IMMEDIATE 'ALTER ' || obj.OBJECT_NAME || ' COMPILE';
  END LOOP;
END;
/

-- Run index validation
ANALYZE INDEX index_name VALIDATE STRUCTURE;

-- Check table consistency
ANALYZE TABLE table_name COMPUTE STATISTICS;
```

#### 3. Backup Verification

```sql
-- List recent backups
SELECT BACKUP_TYPE, COUNT(*) AS COUNT, MAX(COMPLETION_TIME) AS LATEST
FROM V$BACKUP_SET_DETAILS
GROUP BY BACKUP_TYPE
ORDER BY MAX(COMPLETION_TIME) DESC;

-- Check backup summary
RMAN> REPORT SCHEMA;

-- List outdated backups
RMAN> LIST OBSOLETE;

-- Delete obsolete backups
RMAN> DELETE OBSOLETE;
```

#### 4. Performance Verification

```sql
-- Check redo log performance
SELECT GROUP#, MB_PER_SEC, PHYSICAL_WRITES 
FROM V$SYSSTAT 
WHERE NAME LIKE 'redo%';

-- Monitor redo generation rate
SELECT NAME, VALUE 
FROM V$SYSSTAT 
WHERE NAME IN ('redo log space requests', 'redo log space wait time');

-- Check recovery performance
SELECT NAME, VALUE 
FROM V$SYSSTAT 
WHERE NAME LIKE 'physical%';
```

#### 5. Archive Log Status

```sql
-- Archive log generation
SELECT TRUNC(FIRST_TIME) AS DAY, COUNT(*) AS LOGS_PER_DAY
FROM V$ARCHIVED_LOG
GROUP BY TRUNC(FIRST_TIME)
ORDER BY TRUNC(FIRST_TIME) DESC;

-- Archive destination status
SELECT DEST_NAME, DESTINATION, STATUS, ERROR
FROM V$ARCHIVE_DEST_STATUS
WHERE STATUS != 'INACTIVE';
```

---

## Troubleshooting Common Issues

### Issue: "ORACLE not available" during recovery

```bash
# Ensure ORACLE_SID and ORACLE_HOME are set
export ORACLE_SID=pdb_name
export ORACLE_HOME=/u01/oracle/product/26.0
export PATH=$ORACLE_HOME/bin:$PATH

# Test connection
sqlplus / as sysdba
```

### Issue: Archive logs not found

```sql
-- Check available archive logs
SELECT COUNT(*) FROM V$ARCHIVED_LOG;

-- Check archive destination
SHOW PARAMETER LOG_ARCHIVE_DEST;

-- Manually register archive logs
RMAN> CATALOG DATAFILECOPY '/u01/archive_logs/archive_1234.dbf';
```

### Issue: Control file is read-only

```bash
# Check file permissions
ls -la /u01/oradata/prod_db/control*.ctl

# Change permissions if needed
chmod 660 /u01/oradata/prod_db/control*.ctl

# Verify ownership
chown oracle:dba /u01/oradata/prod_db/control*.ctl
```

### Issue: Recovery stuck or slow

```sql
-- Check recovery progress
SELECT * FROM V$SESSION_RECOVERY;

-- Check redo logs being applied
SELECT * FROM V$LOGSTDBY_LOG;

-- Monitor I/O during recovery
SELECT * FROM V$IOSTAT_BY_FILETYPE;
```

---

## Checklist: Before & After Recovery

### Before Recovery
- [ ] Identify root cause of failure
- [ ] Verify backup integrity
- [ ] Ensure sufficient disk space
- [ ] Notify stakeholders
- [ ] Document current state
- [ ] Test recovery in non-production first
- [ ] Set maintenance window
- [ ] Back up current (corrupted) database

### After Recovery
- [ ] Verify database is OPEN
- [ ] Check for invalid objects
- [ ] Run integrity checks
- [ ] Validate data
- [ ] Verify redo logs
- [ ] Check alert logs for errors
- [ ] Re-enable archive mode (if applicable)
- [ ] Run full backup immediately
- [ ] Update documentation
- [ ] Notify stakeholders of completion

---

## Quick Reference Commands

```sql
-- Database Status
SELECT NAME, OPEN_MODE, LOG_MODE FROM V$DATABASE;

-- Datafile Status
SELECT FILE#, NAME, STATUS FROM V$DATAFILE ORDER BY FILE#;

-- Tablespace Status
SELECT TABLESPACE_NAME, STATUS FROM DBA_TABLESPACES;

-- Archive Logs
SELECT SEQUENCE#, FIRST_TIME, ARCHIVED FROM V$LOG ORDER BY SEQUENCE#;

-- Recovery Status
SELECT * FROM V$SESSION WHERE COMMAND = 37;

-- Backup Status
RMAN> LIST BACKUP SUMMARY;

-- Invalid Objects
SELECT OBJECT_TYPE, OBJECT_NAME, STATUS FROM DBA_OBJECTS WHERE STATUS = 'INVALID';
```

---

## Additional Resources

- Oracle 26AI Documentation: https://docs.oracle.com
- RMAN Best Practices: Oracle Database Backup and Recovery Guide
- Data Guard Setup: Oracle Data Guard Concepts and Administration
- Performance Tuning: Oracle Database Performance Tuning Guide

---

**Last Updated:** January 2024
**Oracle Version:** Oracle 26AI
**Document Status:** Comprehensive Reference Guide
