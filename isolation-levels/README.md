# Overview
Transaction isolation levels are best explained by the
different racing conditions that can arise between separate transactions.
These conditions are called phenomenons.

## Phenomenons

To understand the phenomenons let's start by using a table as an example.
Created in mysql:
```sql
CREATE TABLE IF NOT EXISTS transactions
(
    id     INT PRIMARY KEY,
    amount NUMERIC NOT NULL
);
```

### 1. Dirty Reads
Transaction reads data written by a concurrent uncommitted transaction.\
If the uncommitted transaction is rolled back, the reading transaction has read "dirty" data that never actually existed.

**Example:**\
Runs first:\
```sql
START TRANSACTION;
INSERT INTO transactions(id, amount) VALUES (1, 100);
DO SLEEP(15);
ROLLBACK;
```
And then:\
```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT * FROM transactions; -- dirty read happens
COMMIT;
```

The solutions is to increase the isolation level:
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT * FROM transactions; -- dirty read not possible
COMMIT;
```