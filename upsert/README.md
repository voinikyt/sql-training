# Concurrency Issues

```sql
CREATE TABLE client
(
    id      text PRIMARY KEY,
    name    text NOT NULL,
    address text,
    email   text
);

INSERT INTO public.client (id, "name", address, email)
VALUES ('client_1', 'old name', 'old address', 'old@mail.com');
```

## Process A - inserts/updates name and address preserving email

3 queries.
```kotlin
Client fromDb = clientRepository.findById("client_1");
Client client = Client("client_1", "old name", "old address", fromDb.?getEmail());
clientRepository.save(fromDb);
```

## Process B - updates email
```kotlin
Client fromDb = clientRepository.findById("client_1");
fromDb.setEmail("new@mail.com");
clientRepository.save(fromDb);
```

# The problem

1. It's very slow having 3 queries per record. What if you have to import thousands of records.
2. Concurrency issues

```sql
DO $$
    DECLARE
        v_client RECORD;
        v_id TEXT := 'client_1';
        v_name TEXT := 'new name';
        v_address TEXT := 'new address';
    BEGIN
        SELECT * INTO v_client FROM client WHERE id = v_id;

        perform pg_sleep(15);

        IF v_client IS NOT NULL THEN
            UPDATE client
            SET name = v_name,
                address = v_address,
                email = v_client.email
            WHERE id = v_client.id;
        ELSE
            INSERT INTO client (id, name, address, email)
            VALUES (v_id, v_name, v_address, null);
        END IF;
    END $$;
```

```sql
DO $$
DECLARE
    v_client RECORD;
    v_id TEXT := 'client_1';
    v_email TEXT := 'new@mail.com';
BEGIN
    SELECT * INTO v_client FROM client WHERE id = v_id;

   	perform pg_sleep(10);
   
    UPDATE client 
    SET name = v_client.name, 
        address = v_client.address, 
        email = v_email
    WHERE id = v_client.id;
END $$;
```

# A Solution
```sql

```