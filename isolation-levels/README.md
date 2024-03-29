# Overview
Transaction isolation levels are best explained by the
different racing conditions that can arise between separate transactions.
These conditions are called phenomenons.

## Phenomenons

To understand the phenomenons let's start by using a table as an example.
Created in mysql:
```mysql
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
```mysql
START TRANSACTION;
INSERT INTO account(id, amount) VALUES (1, 100);
DO SLEEP(15);
ROLLBACK;
```

```mysql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT * FROM account; -- dirty read happens
COMMIT;
```

### 2. Non-Repeatable Read

A transaction re-execute the same query but sees updated data.

First run:
```mysql
INSERT INTO account(id, amount) VALUES (2, 20);
```

Then run in parallel:
```mysql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT * FROM account;
DO SLEEP(15);
SELECT * FROM account;
COMMIT;
```

```mysql
START TRANSACTION;
UPDATE account
SET amount = 21
WHERE id = 2;
COMMIT;
```

### 3. Phantom Read

A transaction re-execute the same query but sees newly inserted data.

Run in parallel:
```mysql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT * FROM account;
DO SLEEP(15);
SELECT * FROM account;
COMMIT;
```

```mysql
START TRANSACTION;
INSERT INTO account(id, amount) VALUES (1, 100);
COMMIT;
```

### 4. Serialization Anomaly

Run in parallel:
```mysql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT amount INTO @my_amount FROM account a  WHERE id = 1;
DO SLEEP(15);
UPDATE account SET amount = @my_amount + 10 WHERE id = 1;
COMMIT;
```

```mysql
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
| Phantom Reads           | Possible         | Possible       | Prevented???    | Prevented    |
| Serialization Anomaly   | Possible         | Possible       | Possible        | Prevented    |

## Practical Examples
The following is a table with events for employees.
Every event contains a new state of an employee.

| Employee ID | Name           | Address | Status    | Timestamp  |
|-------------|----------------|---------|-----------|------------|
| 1           | Jane Unmarried | NY      | NULL      | 2024-03-19 |
| 1           | Jane Unmarried | CA      | NULL      | 2024-03-20 |
| 2           | John Doe       | CH      | PROCESSED | 2024-03-21 |
| 2           | John Doe       | TX      | NULL      | 2024-03-22 |

Requirements:
- our application sends employee to a third party service
- the service only supports create requests - updates are not supported
- create for existing employee returns an error
- updates will be supported in future

Solution:
- when the application starts it groups all employees 
- if there is any processed record for an employee -> ignore the rest
- if there are no processed and there are more than 1 record -> leave the most recent one

| Employee ID | Name           | Address | Status    | Timestamp  |
|-------------|----------------|---------|-----------|------------|
| 1           | Jane Unmarried | NY      | IGNORED   | 2024-03-19 |
| 1           | Jane Unmarried | CA      | NULL      | 2024-03-20 |
| 2           | John Doe       | CH      | PROCESSED | 2024-03-21 |
| 2           | John Doe       | TX      | IGNORED   | 2024-03-22 |

### Application Code Solution
```java
        // 1 DB query
        List<Employee> employees = fetchWithStatusNullOrStatus("PROCESSED");

        Set<String> processedIds = employees.stream()
                .filter(employee -> "PROCESSED".equals(employee.status))
                .map(Employee::getEmployeeId)
                .collect(Collectors.toSet());

        // remove already processed
        employees.removeIf(employee -> {
            if (!processedIds.contains(employee.getEmployeeId())) {
                return false;
            }
            
            if (!employee.getStatus().equals("PROCESSED")) {
                employee.setStatus("IGNORED");
                // 1 or 2 DB queries
                saveToDb(employee);    
            }
            return true;
        });

        Map<String, List<Employee>> groupings = employees.stream()
                .collect(Collectors.groupingBy(Employee::getEmployeeId));
        groupings.values().forEach(group -> group.sort(Comparator.comparing(Employee::getTimestamp)));
        groupings.forEach((employeeId, group) -> {
            if (group.size() == 1) {
                return;
            }
            Employee last = group.getLast();
            group.removeIf(employee -> {
                if (employee == last) {
                    return false;
                }
                employee.setStatus("IGNORED");
                // 1 or 2 DB queries
                saveToDb(employee);
                return true;
            });
        });

        List<Employee> latestRecords = groupings.values().stream()
                .flatMap(Collection::stream)
                .toList();
```

What if we have 100s of employees, and we must ignore 90 of them and process 10 of them:
* 180 queries - select for update
* 20 for the successful ones

Also let's say that we have 100s of processed employees.
* we'll fetch a lot of data in memory

### Solution running queries - without being taught on isolation levels

#### PostgresSQL
```mysql
CREATE TABLE employee
(
    id           SERIAL PRIMARY KEY,
    employee_id  TEXT   NOT NULL,
    name         TEXT   NOT NULL,
    address      TEXT   NOT NULL,
    status       TEXT,
    last_updated TIMESTAMP NOT NULL
);
```

```mysql
INSERT INTO employee (employee_id, name, address, status, last_updated)
VALUES ('1', 'Jane Unmarried', 'NY', NULL, '2024-03-19'),
       ('1', 'Jane Unmarried', 'CA', NULL, '2024-03-20'),
       ('2', 'John Doe', 'CH', 'PROCESSED', '2024-03-21'),
       ('2', 'John Doe', 'TX', NULL, '2024-03-22');
```

Run in parallel:
```mysql
BEGIN;

SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Ignore processed
UPDATE employee e1
SET status = 'IGNORED'
FROM (
    SELECT employee_id
    FROM employee
    WHERE status = 'PROCESSED'
) e2 
WHERE e1.employee_id = e2.employee_id AND e1.status IS NULL;

-- Ignore older
WITH max_updated AS (
    SELECT employee_id, MAX(last_updated) AS last_updated
    FROM employee
    WHERE status IS NULL
    GROUP BY employee_id
)
UPDATE employee e1
SET status = 'IGNORED'
FROM max_updated e2
WHERE e1.employee_id = e2.employee_id AND e1.status IS NULL AND e1.last_updated != e2.last_updated;

select pg_sleep(15);

SELECT * FROM employee WHERE status IS NULL;

COMMIT;

```

```mysql
INSERT INTO employee (employee_id, name, address, status, last_updated)
VALUES ('1', 'Jane Unmarried', 'NY', NULL, '2024-04-01'),
       ('2', 'John Doe', 'TX', NULL, '2024-04-02');
```

##### Solution
```mysql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

#### MySQL

```mysql
CREATE TABLE employee
(
    id           INT AUTO_INCREMENT PRIMARY KEY,
    employee_id  TEXT      NOT NULL,
    name         TEXT      NOT NULL,
    address      TEXT      NOT NULL,
    status       TEXT,
    last_updated TIMESTAMP NOT NULL
);
```

```mysql
INSERT INTO employee (employee_id, name, address, status, last_updated)
VALUES ('1', 'Jane Unmarried', 'NY', NULL, '2024-03-19'),
       ('1', 'Jane Unmarried', 'CA', NULL, '2024-03-20'),
       ('2', 'John Doe', 'CH', 'PROCESSED', '2024-03-21'),
       ('2', 'John Doe', 'TX', NULL, '2024-03-22');
```

Now run in parallel:
```mysql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;

-- ignore processed
UPDATE employee e1
JOIN
(
    SELECT employee_id
    FROM employee
    WHERE status = 'PROCESSED'
) e2 
ON e1.employee_id = e2.employee_id
SET e1.status = 'IGNORED' WHERE e1.status IS NULL;

-- ignore older
UPDATE employee e1
JOIN
(
     SELECT employee_id, MAX(last_updated) AS last_updated
     FROM employee
     WHERE status IS NULL
     GROUP BY employee_id
 ) e2 
ON e1.employee_id = e2.employee_id
SET e1.status = 'IGNORED'
WHERE e1.status IS NULL AND e1.last_updated != e2.last_updated;

DO SLEEP(15);

SELECT * FROM employee WHERE status IS NULL;

COMMIT;
```

```mysql
INSERT INTO employee (employee_id, name, address, status, last_updated)
VALUES ('1', 'Jane Unmarried', 'NY', NULL, '2024-04-01'),
       ('2', 'John Doe', 'TX', NULL, '2024-04-02');
```

##### Solution 
```mysql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```