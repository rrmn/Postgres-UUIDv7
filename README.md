# Postgres-UUIDv7
Add a UUIDv7 function to Postgres

## Background

Time-sortable UUIDs are awesome. 


## Create function 

```
CREATE OR REPLACE FUNCTION generate_uuid7()
RETURNS uuid AS $$
DECLARE
    unixts bigint;
    msec bigint;
    seq bigint;
    rand bytea;
    uuid_str text;
BEGIN
    -- Get UNIX timestamp in milliseconds
    unixts := (EXTRACT(epoch FROM clock_timestamp()) * 1000)::bigint;

    -- Generate random sequence counter (12 bits) and random data (62 bits)
    seq := (get_byte(gen_random_bytes(2), 0) << 8) | get_byte(gen_random_bytes(2), 1);
    rand := gen_random_bytes(8); -- 8 bytes = 64 bits, will use 62 bits

    -- Millisecond precision (12 bits)
    msec := unixts & 4095; -- Lower 12 bits of unixts

    -- Shift unixts to get first 36 bits for timestamp
    unixts := unixts >> 12;

    -- Combine parts into UUID string
    uuid_str := lpad(to_hex(unixts), 9, '0') ||             -- 36 bits for unixts
                lpad(to_hex(msec), 3, '0') ||               -- 12 bits for msec
                '7' ||                                      -- 4 bits for 'uuid version 7'
                lpad(to_hex(seq), 3, '0') ||                -- 12 bits for sequence
                lpad(to_hex((get_byte(rand, 0) & 63) | 128), 2, '0') || -- Variant bits (2 bits) and first 6 bits of random data
                encode(substring(rand from 2), 'hex');      -- Remaining random data (56 bits)

    -- Format string to UUID spec
    uuid_str := substr(uuid_str, 1, 8) || '-' ||
                substr(uuid_str, 9, 4) || '-' ||
                substr(uuid_str, 13, 4) || '-' ||
                substr(uuid_str, 17, 4) || '-' ||
                substr(uuid_str, 21, 12);

    RETURN uuid(uuid_str);
END;
$$ LANGUAGE plpgsql;

```

## Use function 

```
SELECT generate_uuid7();
```
