---
layout : post
title : Oracle - How to drop object if not exists
category: Oracle
tag: 
- Oracle
- Database
---

<p>
**The best and most efficient way is to catch the "table not found" exception: this avoids the overhead of checking if the table exists twice; and doesn't suffer from the problem that if the DROP fails for some other reason (that might be important) the exception is still raised to the caller:**
</p>

## Table

```sql

BEGIN
   EXECUTE IMMEDIATE 'DROP TABLE ' || table_name;
EXCEPTION
   WHEN OTHERS THEN
      IF SQLCODE != -942 THEN
         RAISE;
      END IF;
END;

```


**ADDENDUM**  For reference, here are the equivalent blocks for other object types:

## Sequence

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP SEQUENCE ' || sequence_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -2289 THEN
      RAISE;
    END IF;
END;
```

## View

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP VIEW ' || view_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -942 THEN
      RAISE;
    END IF;
END;
```

## Trigger

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP TRIGGER ' || trigger_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -4080 THEN
      RAISE;
    END IF;
END;
```

## Index

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP INDEX ' || index_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -1418 THEN
      RAISE;
    END IF;
END;
```

## Column

```sql
BEGIN
  EXECUTE IMMEDIATE 'ALTER TABLE ' || table_name
                || ' DROP COLUMN ' || column_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -904 AND SQLCODE != -942 THEN
      RAISE;
    END IF;
END;
```

## Database Link

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP DATABASE LINK ' || dblink_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -2024 THEN
      RAISE;
    END IF;
END;
```

## Materialized View

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP MATERIALIZED VIEW ' || mview_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -12003 THEN
      RAISE;
    END IF;
END;
```

## Type

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP TYPE ' || type_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -4043 THEN
      RAISE;
    END IF;
END;
```

## Constraint

```sql
BEGIN
  EXECUTE IMMEDIATE 'ALTER TABLE ' || table_name
            || ' DROP CONSTRAINT ' || constraint_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -2443 AND SQLCODE != -942 THEN
      RAISE;
    END IF;
END;
```

## Scheduler Job

```sql
BEGIN
  DBMS_SCHEDULER.drop_job(job_name);
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -27475 THEN
      RAISE;
    END IF;
END;
```

## User / Schema

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP USER ' || user_name;
  /* you may or may not want to add CASCADE */
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -1918 THEN
      RAISE;
    END IF;
END;
```

## Package

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP PACKAGE ' || package_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -4043 THEN
      RAISE;
    END IF;
END;
```

## Procedure

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP PROCEDURE ' || procedure_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -4043 THEN
      RAISE;
    END IF;
END;
```

## Function

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP FUNCTION ' || function_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -4043 THEN
      RAISE;
    END IF;
END;
```

## Tablespace

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP TABLESPACE' || tablespace_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -959 THEN
      RAISE;
    END IF;
END;
```

## Synonym

```sql
BEGIN
  EXECUTE IMMEDIATE 'DROP SYNONYM ' || synonym_name;
EXCEPTION
  WHEN OTHERS THEN
    IF SQLCODE != -1434 THEN
      RAISE;
    END IF;
END;
```
