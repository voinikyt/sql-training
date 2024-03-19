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
| 2           | John Doe       | CH      | NULL      | 2024-03-22 |

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
| 2           | John Doe       | CH      | IGNORED   | 2024-03-22 |

### Application Code Solution
```java
List<Employee> employees = fetchWithStatusNullOrStatus("PROCESSED");

Set<String> processedIds = employees.stream()
        .filter(employee -> "PROCESSED".equals(employee.status))
        .map(Employee::getId)
        .collect(Collectors.toSet());

// remove already processed
        employees.removeIf(employee -> {
        if (!processedIds.contains(employee.id)) {
        return false;
        }
        employee.setStatus("IGNORED");
saveToDb(employee);
            return true;
                    });

Map<String, List<Employee>> groupings = employees.stream()
        .collect(Collectors.groupingBy(Employee::getId));
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
saveToDb(employee);
                return true;
                        });
                        });

List<Employee> latestRecords = groupings.values().stream()
        .flatMap(Collection::stream)
        .toList();

        System.out.println(latestRecords);
    }
```

What if we have 100s of employees, and we must ignore almost all of them.
That means a query 2 queries for each of them (select for update);

### Cheeky solution running queries

## Concurrency Behaviour 