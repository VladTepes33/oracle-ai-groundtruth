# Example Q&A — RAC + Data Guard Switchover

> This example shows the type of response you can expect from the assistant
> when the project includes official Oracle PDFs and the knowledge base.

---

## Question

```
How do I perform a planned switchover from a 3-node RAC primary to a 
2-node RAC physical standby using Data Guard Broker? 
What are the pre and post-checks?
```

---

## Expected Response

### Pre-Switchover Checklist

Before initiating the switchover, verify the following on the **primary**:

**1. Apply lag must be zero (or near-zero)**
```sql
SELECT name, value, unit FROM v$dataguard_stats 
WHERE name IN ('apply lag', 'transport lag');
```

**2. No archive log gaps**
```sql
SELECT * FROM v$archive_gap;
-- Must return 0 rows
```

**3. Switchover status must be ready**
```sql
SELECT switchover_status FROM v$database;
-- Expected: 'TO STANDBY' or 'SESSIONS ACTIVE'
```

**4. Validate with Broker before proceeding**
```bash
dgmgrl sys/password@ORCL_PRI
DGMGRL> VALIDATE DATABASE FOR SWITCHOVER 'ORCL_STB';
```

**5. Notify application teams** — connections will be briefly interrupted.
All active sessions will receive TCP RST during VIP failover (~30 seconds).

---

### Switchover Procedure (via Broker)

```bash
dgmgrl sys/password@ORCL_PRI

DGMGRL> SHOW CONFIGURATION;
-- Confirm: Configuration Status = SUCCESS

DGMGRL> SWITCHOVER TO 'ORCL_STB';
-- Broker handles:
--   1. Suspending redo transport
--   2. Flushing remaining redo to standby
--   3. Converting primary → standby
--   4. Converting standby → primary
--   5. Restarting services on both sides
```

Broker estimated time: **2–5 minutes** for a typical RAC+DG switchover.

---

### Post-Switchover Checklist

**1. Confirm new roles**
```sql
-- On new primary (former standby):
SELECT db_unique_name, database_role, open_mode FROM v$database;
-- Expected: ORCL_STB | PRIMARY | READ WRITE

-- On new standby (former primary):
SELECT db_unique_name, database_role, open_mode FROM v$database;
-- Expected: ORCL_PRI | PHYSICAL STANDBY | READ ONLY WITH APPLY
```

**2. Verify Broker configuration**
```bash
DGMGRL> SHOW CONFIGURATION;
-- Expected: Configuration Status = SUCCESS (no WARNING)
```

**3. Verify redo apply is running on new standby**
```sql
SELECT process, status, sequence# 
FROM v$managed_standby 
WHERE process LIKE 'MRP%';
-- Expected: MRP0 | APPLYING_LOG
```

**4. Verify SCAN and VIP listeners on new primary**
```bash
srvctl status scan
srvctl status scan_listener
srvctl status nodeapps
```

**5. Verify application services started on new primary**
```bash
srvctl status service -db ORCL_STB
-- Services with ROLE=PRIMARY should now be ONLINE
```

**6. Test application connectivity**
```bash
tnsping ORCL_STB    # should resolve to new primary SCAN
sqlplus appuser/password@ORCL_SVC
```

---

📚 Références
```
┌──────────────────────────────────────────────────────────────────────────┐
│ [1]  Oracle Data Guard Broker 19c — Chapter 5                            │
│      "Switchover and Failover Operations" — p. 5-3 to 5-12              │
│ [2]  Oracle Data Guard Concepts and Administration 19c — Chapter 9       │
│      "Using SQL Apply to Upgrade the Oracle Database" — p. 9-1           │
│      (switchover pre-checks section)                                     │
│ [3]  Oracle RAC Administration and Deployment Guide 19c — Chapter 6      │
│      "Administering Database Instances and Cluster Databases" — p. 6-18  │
│ [KB] Knowledge Base oracle19c_ha — Part 3.5 (Switchover procedures)      │
│ [KB] Knowledge Base oracle19c_ha — Part 4.3 (SRVCTL for RAC+DG)         │
└──────────────────────────────────────────────────────────────────────────┘
```

---

> **Note**: This is an illustrative example. Actual page numbers depend on the
> specific PDF version uploaded to your Claude Project.
