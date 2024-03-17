# Overview
Transaction isolation levels are best explained by the
different racing conditions that can arise between separate transactions.
These conditions are called phenomenons.

## Phenomenons

To understand the phenomenons let's start by using a table as an example.
Created in mysql:
```sql
CREATE TABLE IF NOT EXISTS account
(
    id     INT PRIMARY KEY,
    amount NUMERIC NOT NULL
);
```

### 1. Dirty Reads
Transaction reads data written by a concurrent uncommitted transaction.\
If the uncommitted transaction is rolled back, the reading transaction has read "dirty" data that never actually existed.

**Example:**\
Run in parallel the below 2 scripts:
```sql
START TRANSACTION;
INSERT INTO account(id, amount) VALUES (1, 100);
DO SLEEP(15);
ROLLBACK;
```

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT * FROM account; -- dirty read happens
COMMIT;
```

### 2. Non-Repeatable Read

A transaction re-execute the same query but sees updated data.

First run:
```sql
INSERT INTO account(id, amount) VALUES (2, 20);
```

Then run in parallel:
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT * FROM account;
DO SLEEP(15);
SELECT * FROM account;
COMMIT;
```

```sql
START TRANSACTION;
UPDATE account
SET amount = 21
WHERE id = 2;
COMMIT;
```

### 3. Phantom Read

A transaction re-execute the same query but sees newly inserted data.

Run in parallel:
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT * FROM account;
DO SLEEP(15);
SELECT * FROM account;
COMMIT;
```

```sql
START TRANSACTION;
INSERT INTO account(id, amount) VALUES (1, 100);
COMMIT;
```

### 4. Serialization Anomaly

Run in parallel:
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT amount INTO @my_amount FROM account a  WHERE id = 1;
DO SLEEP(15);
UPDATE account SET amount = @my_amount + 10 WHERE id = 1;
COMMIT;
```

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT amount INTO @my_amount FROM account a  WHERE id = 1;
DO SLEEP(10);
UPDATE account SET amount = @my_amount + 15 WHERE id = 1;
COMMIT;
```

## Isolation Levels

To solve the above phenomenons one must increase the isolation level:

| Phenomenon              | Read Uncommitted | Read Committed | Repeatable Read | Serializable |
|-------------------------|------------------|----------------|-----------------|--------------|
| Dirty Reads             | Possible         | Prevented      | Prevented       | Prevented    |
| Non-Repeatable Reads    | Possible         | Possible       | Prevented       | Prevented    |
| Phantom Reads           | Possible         | Possible       | Possible        | Prevented    |
| Serialization Anomaly   | Possible         | Possible       | Possible        | Prevented    |
