# Overview

Transaction isolation levels are best explained by the
different racing conditions that can arise between separate transactions.
These conditions are called phenomenons:

## Phenomenons
### 1. Dirty Reads
Transaction reads data written by a concurrent uncommitted transaction. 
If the uncommitted transaction is rolled back, the reading transaction has read "dirty" data that never actually existed.

**Example:**

Transaction 1 starts and updates a row in the database but does not commit yet.
Transaction 2 reads the updated row before Transaction 1 commits.
If Transaction 1 is rolled back, Transaction 2 has read data that was never committed, i.e., a dirty read.
