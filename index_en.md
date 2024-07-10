

# sp_Blitz

This documentation explains how to analyze and optimize indexes in an SQL Server database using the sp_Blitz tool.

## Requirements

- SQL Server 2012 or later

- SQL Server Management Studio (SSMS)

- sp_Blitz script or scripts

## Download

First, download the sp_Blitz script and load it into SQL Server.

- [Clone from GitHub](https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit)

- [Download as zip](https://downloads.brentozar.com/FirstResponderKit.zip)

## Running the sp_Blitz Script

Open the script you want to use from the downloaded FirstResponderKit folder.

- At this stage, make the connection security setting optional instead of mandatory to avoid certificate errors. You do not need to change the database name, just make the connection security setting optional.

- Once the connection is successfully made, execute the script that appears on the screen. It will be a long query; there is no need to worry.

- Open a new query window and execute the query you want to run.

- [Access all queries in the readme.md](https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit/tree/main#sp_blitz-overall-health-check)

## Identifying Indexes

To identify the existing indexes in a table, run the following query:

```sql

SELECT

    t.name AS TableName,

    i.name AS IndexName,

    i.index_id AS IndexID,

    ic.index_column_id AS IndexColumnID,

    col.name AS ColumnName

FROM

    sys.indexes i

INNER JOIN

    sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id

INNER JOIN

    sys.columns col ON ic.object_id = col.object_id AND ic.column_id = col.column_id

INNER JOIN

    sys.tables t ON i.object_id = t.object_id

WHERE

    t.name = '' -- enter the table name here.

ORDER BY

    t.name, i.index_id, ic.index_column_id;

```

## Analyzing Index Usage

To analyze the usage status of indexes, run the following query:

```sql

SELECT

    OBJECT_NAME(s.object_id) AS TableName,

    i.name AS IndexName,

    i.index_id AS IndexID,

    s.user_seeks,

    s.user_scans,

    s.user_lookups,

    s.user_updates

FROM

    sys.dm_db_index_usage_stats s

INNER JOIN

    sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id

WHERE

    OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1

    AND s.database_id = DB_ID('') -- enter the database name here.

    AND OBJECT_NAME(s.object_id) = '' -- enter the table name here

ORDER BY

    s.user_seeks DESC;

```

## Removing Unnecessary Indexes - Reconfiguring

This section will not be covered.

## Important Warnings

Specific index warnings detected by sp_Blitz in SQL Server and their meanings, along with solutions, are detailed below. Each warning's meaning, actions to be taken against these warnings, and the necessary SQL queries are included.

## Actions to Be Taken

### 1. Multiple Index Personalities: Duplicate Keys

#### Meaning

This warning indicates that there are multiple indexes with the same key columns on the same table. This suggests the presence of unnecessary indexes and resource waste.

#### Solution

**Identifying Indexes:**

```sql

SELECT

    t.name AS TableName,

    i.name AS IndexName,

    i.index_id AS IndexID,

    ic.index_column_id AS IndexColumnID,

    col.name AS ColumnName

FROM

    sys.indexes i

INNER JOIN

    sys.index_columns ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id

INNER JOIN

    sys.columns col ON ic.object_id = col.object_id AND ic.column_id = col.column_id

INNER JOIN

    sys.tables t ON i.object_id = t.object_id

WHERE

    t.name = '' -- enter the table name here.

ORDER BY

    t.name, i.index_id, ic.index_column_id;

```

**Removing Unnecessary Index:**

```sql

DROP INDEX IndexName ON TableName;

```

### 2. Multiple Index Personalities: Borderline Duplicate Keys

#### Meaning

This warning indicates that there are multiple indexes with similar but not identical key columns on the same table. These indexes may cause unnecessary performance degradation.

#### Solution

**Identifying and Analyzing Index Usage:**

```sql

SELECT

    OBJECT_NAME(s.object_id) AS TableName,

    i.name AS IndexName,

    i.index_id AS IndexID,

    s.user_seeks,

    s.user_scans,

    s.user_lookups,

    s.user_updates

FROM

    sys.dm_db_index_usage_stats s

INNER JOIN

    sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id

WHERE

    OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1

    AND OBJECT_NAME(s.object_id) = '' -- enter the table name here

ORDER BY

    s.user_seeks DESC;

```

**Removing Unnecessary Index:**

```sql

DROP INDEX IndexName ON TableName;

```

### 3. Indexaphobia: High Value Missing Index

#### Meaning

This warning indicates a missing index with high value. This suggests that a certain index is missing, which could significantly improve the performance of specific queries.

#### Solution

**Identifying Missing Index Recommendations:**

```sql

SELECT

    migs.avg_total_user_cost * migs.avg_user_impact * (migs.user_seeks + migs.user_scans) AS improvement_measure,

    OBJECT_NAME(mid.object_id) AS TableName,

    mid.equality_columns,

    mid.inequality_columns,

    mid.included_columns,

    migs.*

FROM

    sys.dm_db_missing_index_group_stats AS migs

INNER JOIN

    sys.dm_db_missing_index_groups AS mig ON migs.group_handle = mig.index_group_handle

INNER JOIN

    sys.dm_db_missing_index_details AS mid ON mig.index_handle = mid.index_handle

WHERE

    mid.database_id = DB_ID('') -- enter the database name here

ORDER BY

    improvement_measure DESC;

```

**Creating the Missing Index:**

```sql

CREATE INDEX IndexName ON TableName (Column1, Column2) INCLUDE (Column3, Column4);

```

### 4. Aggressive Indexes & Under-Indexing: Total Lock Wait Time > 5 Minutes (Row + Page)

#### Meaning

This warning indicates that the total lock wait time has exceeded 5 minutes. This situation can arise due to aggressive indexing or under-indexing.

#### Solution

**Identifying Lock Wait Times:**

```sql

SELECT

    request_session_id AS SPID,

    resource_type,

    resource_description,

    request_mode,

    request_status

FROM

    sys.dm_tran_locks

WHERE

    resource_type IN ('OBJECT', 'PAGE', 'RID', 'KEY')

ORDER BY

    request_session_id;

```

**Identifying Queries Causing Locks:**

```sql

SELECT

    blocking_session_id AS BlockingSessionID,

    session_id AS VictimSessionID,

    wait_type,

    wait_time,

    resource_description

FROM

    sys.dm_exec_requests

WHERE

    blocking_session_id <> 0;

```

**Removing the Index Causing Locks:**

```sql

DROP INDEX IndexName ON TableName;

```

### 5. Index Hoarder: NC Index with High Writes:Reads

#### Meaning

This warning indicates a non-clustered index with high write operations but relatively low read operations. These indexes can negatively impact write performance.

#### Solution

Analyze indexes with high write and low read ratios, and remove or optimize them if necessary.

**Analyzing Index Usage:**

```sql

SELECT

    OBJECT_NAME(s.object_id) AS TableName,

    i.name AS IndexName,

    i.index_id AS IndexID,

    s.user_seeks,

    s.user_scans,

    s.user_lookups,

    s.user_updates

FROM

    sys.dm_db_index_usage_stats s

INNER JOIN

    sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id

WHERE

    OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1

    AND OBJECT_NAME(s.object_id) = 'YourTableName'

ORDER BY

    s.user_updates DESC;

```

**Removing Unnecessary Index:**

```sql

DROP INDEX IndexName ON TableName;

```

### 6. Index Hoarder: Unused NC Index with High Writes

#### Meaning

This warning indicates a non-clustered index with high write operations but is unused. These indexes cause unnecessary resource consumption.

#### Solution

Identify and remove unused indexes with high write loads.

**Analyzing Index Usage:**

```sql

SELECT

    OBJECT_NAME(s.object_id) AS TableName,

    i.name AS IndexName,

    i.index_id AS IndexID,

    s.user_seeks,

    s.user_scans,

    s.user_lookups,

    s.user_updates

FROM

    sys.dm_db_index_usage_stats s

INNER JOIN

    sys.indexes i ON s.object_id = i.object_id AND s.index_id = i.index_id

WHERE

    OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1

    AND OBJECT_NAME(s.object_id) = '' -- enter the table name here

ORDER BY

    s.user_updates DESC;

```

**Removing Unnecessary Index:**

```sql

DROP INDEX IndexName ON TableName;

```

## Conclusion
 This documentation explains how to analyze and optimize indexes using sp_Blitz in SQL Server. Additionally, specific types of warnings detected by sp_Blitz and the actions to be taken against these warnings are detailed.

 **Author: Birkan Cemil ABACI**

## Resources
- [sys.dm_db_index_usage_stats Transact-SQL](https://docs.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-index-usage-stats-transact-sql)
- [sys.indexes Transact-SQL](https://docs.microsoft.com/en-us/sql/relational-databases/system-catalog-views/sys-indexes-transact-sql)
- [Brent Ozar's SQL Server First Responder Kit](https://github.com/BrentOzarULTD/SQL-Server-First-Responder-Kit)
 - BrentOzarULTD/SQL-Server-First-Responder-Kit:
   - sp_Blitz, sp_BlitzCache, sp_BlitzFirst, sp_BlitzIndex, and other SQL Server scripts for health checks and performance tuning.