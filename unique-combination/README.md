# Use case

```sql
CREATE TABLE invoice
(
    id         serial PRIMARY KEY,
    invoice_id TEXT NOT NULL,
    client_id  TEXT NOT NULL,
    name       TEXT NOT NULL
)
```

Invoice and client combination should be unique.\
But there could be multiple lines with the same combination.

| ID | Invoice ID | Client ID | Client Name       |
|----|------------|-----------|-------------------|
| 1  | INV 1      | CLI 1     | Unmarried Happily |
| 2  | INV 1      | CLI 1     | Marries So So     |
| 2  | INV 2      | CLI 2     | Forever Young     |
| 4  | ~~INV 1~~  | ~~CLI 3~~ | ~~Asian Bot~~     |

How can we prevent inserting an invoice with different client combination.

# Naive solution with Code

3 queries

```java
Optional<Invoice> fromDb = invoiceRepository.findByInvoiceIdAndClientNotEquals(inv.getInvoiceId(), inv.getClientId());
if (fromDb.isPresent()) {
    log.error("There is already invoice for different client. You fail.");
    return;
}
invoiceRepository.save(inv);
```

or expressed as SQL

```sql
DO
$$
    DECLARE
        v_exists BOOLEAN;
    BEGIN
        SELECT EXISTS (SELECT 1
                       FROM invoice
                       WHERE invoice_id = 'INV 1'
                         AND client_id != 'CLI 1')
        INTO v_exists;

        PERFORM PG_SLEEP(10);

        IF v_exists THEN
            RAISE NOTICE 'There is already an invoice for a different client. You fail.';
        ELSE
            INSERT INTO invoice (invoice_id, client_id, name)
            VALUES ('INV 1', 'CLI 1', 'random name');
        END IF;
    END
$$;
```

```sql
DO
$$
    DECLARE
        v_exists BOOLEAN;
    BEGIN
        SELECT EXISTS (SELECT 1
                       FROM invoice
                       WHERE invoice_id = 'INV 1'
                         AND client_id != 'CLI 2')
        INTO v_exists;

        PERFORM PG_SLEEP(10);

        IF v_exists THEN
            RAISE NOTICE 'There is already an invoice for a different client. You fail.';
        ELSE
            INSERT INTO invoice (invoice_id, client_id, name)
            VALUES ('INV 1', 'CLI 2', 'random name 2');
        END IF;
    END
$$;
```

# Solution with Postgres Creating a lot of Contention

```sql
begin;
SET TRANSACTION ISOLATION LEVEL serializable;
DO $$
DECLARE
    v_exists BOOLEAN;
BEGIN
    SELECT EXISTS (
        SELECT 1 
        FROM invoice
        WHERE invoice_id = 'INV 1' AND client_id != 'CLI 1'
    ) INTO v_exists;

	perform pg_sleep(10);
   
    IF v_exists THEN
        RAISE NOTICE 'There is already an invoice for a different client. You fail.';
    ELSE
        INSERT INTO invoice (invoice_id, client_id, name) 
        VALUES ('INV 1', 'CLI 1', 'random name');
    END IF;
END $$;
commit;
```

```sql
begin;
SET TRANSACTION ISOLATION LEVEL serializable ;
DO $$
DECLARE
    v_exists BOOLEAN;
BEGIN
    SELECT EXISTS (
        SELECT 1 
        FROM invoice
        WHERE invoice_id = 'INV 1' AND client_id != 'CLI 2'
    ) INTO v_exists;

   	perform pg_sleep(10);
   
    IF v_exists THEN
        RAISE NOTICE 'There is already an invoice for a different client. You fail.';
    ELSE
        INSERT INTO invoice (invoice_id, client_id, name) 
        VALUES ('INV 1', 'CLI 2', 'random name 2');
    END IF;
END $$;
commit;
```

```shell
Error occurred during SQL script execution

Reason:
SQL Error [40001]: ERROR: could not serialize access due to read/write dependencies among transactions
  Detail: Reason code: Canceled on identification as a pivot, during write.
  Hint: The transaction might succeed if retried.
  Where: SQL statement "INSERT INTO invoice (invoice_id, client_id, name) 
        VALUES ('INV 1', 'CLI 2', 'random name 2')"
PL/pgSQL function inline_code_block line 16 at SQL statement
```

# PostgresSQL Solution without Contention

```sql
CREATE TABLE invoice_client
(
    invoice_id text NOT NULL,
    client_id  text NOT NULL,
    PRIMARY KEY (invoice_id)
)
```

```sql
begin;
SET TRANSACTION ISOLATION LEVEL read committed;
DO $$
DECLARE
     v_client_id TEXT;
BEGIN
	INSERT INTO invoice (invoice_id, client_id, name) 
        VALUES ('INV 1', 'CLI 1', 'random name');
   
    perform pg_sleep(8);   
       
	INSERT INTO invoice_client (invoice_id, client_id)
    VALUES ('INV 1', 'CLI 1')
    ON CONFLICT (invoice_id)
    DO UPDATE SET client_id = invoice_client.client_id
    RETURNING client_id INTO v_client_id;      
       
    IF v_client_id <> 'CLI 1' THEN
        RAISE EXCEPTION 'There is already an invoice for a different client. You fail.';
    END IF;
END $$;
commit;
```

```sql
begin;
SET TRANSACTION ISOLATION LEVEL read committed;
DO $$
DECLARE
     v_client_id TEXT;
BEGIN
	INSERT INTO invoice (invoice_id, client_id, name) 
        VALUES ('INV 1', 'CLI 2', 'random name');
   
    perform pg_sleep(10);   
       
	INSERT INTO invoice_client (invoice_id, client_id)
    VALUES ('INV 1', 'CLI 2')
    ON CONFLICT (invoice_id)
    DO UPDATE SET client_id = invoice_client.client_id
    RETURNING client_id INTO v_client_id;       	
   	       
    IF v_client_id <> 'CLI 2' THEN
        RAISE EXCEPTION 'There is already an invoice for a different client. You fail.';	  
    END IF;
END $$;
commit;
```

## MySql Translation

```mysql
DELIMITER $$

CREATE PROCEDURE process_invoice()
BEGIN
    DECLARE v_client_id VARCHAR(255);
    
    -- Attempt to insert the new invoice
    INSERT INTO invoice (invoice_id, client_id, name) 
    VALUES ('INV 1', 'CLI 2', 'random name');
    
    -- MySQL does not have a built-in sleep function in the SQL layer, 
    -- but you can use SELECT SLEEP(10) if you really need to pause.
    SELECT SLEEP(10);

    -- Attempt to insert into the mapping table or update if there's a conflict
    INSERT INTO invoice_to_client_mapping (invoice_id, client_id)
    VALUES ('INV 1', 'CLI 2')
    ON DUPLICATE KEY UPDATE client_id = VALUES(client_id);
    
    -- After attempting the insert/update, check if the client_id matches
    SELECT client_id INTO v_client_id FROM invoice_to_client_mapping WHERE invoice_id = 'INV 1';
    
    IF v_client_id <> 'CLI 2' THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'There is already an invoice for a different client. You fail.';
    END IF;
END$$

DELIMITER ;
```

```mysql
START TRANSACTION;
CALL process_invoice();
COMMIT;
```