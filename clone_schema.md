# Clone Schema inside Database

## Function

```
The version in this page only clones tables. There's a more complete version, that copies sequences, tables, data, views & functions, in this posting.

CREATE OR REPLACE FUNCTION clone_schema(source_schema text, dest_schema text) RETURNS void AS
$BODY$
DECLARE 
  objeto text;
  buffer text;
BEGIN
    EXECUTE 'CREATE SCHEMA ' || dest_schema ;

    FOR objeto IN
        SELECT table_name::text FROM information_schema.tables WHERE table_schema = source_schema
    LOOP        
        buffer := dest_schema || '.' || objeto;
        EXECUTE 'CREATE TABLE ' || buffer || ' (LIKE ' || source_schema || '.' || objeto || ' INCLUDING CONSTRAINTS INCLUDING INDEXES INCLUDING DEFAULTS)';
        EXECUTE 'INSERT INTO ' || buffer || '(SELECT * FROM ' || source_schema || '.' || objeto || ')';
    END LOOP;

END;
$BODY$
LANGUAGE plpgsql VOLATILE;
Execution is simple:

SELECT clone_schema('old_schema','new_schema');
Worth noting is that this copies all tables in the schema, including the default values. Which, in the case of serial fields, will mean the default value points to the old schema:

    nextval('old_schema.sequence_name'::regclass)
If you want to have a new sequence for the new table in the new schema:

CREATE OR REPLACE FUNCTION clone_schema(source_schema text, dest_schema text) RETURNS void AS
$$

DECLARE
  object text;
  buffer text;
  default_ text;
  column_ text;
BEGIN
  EXECUTE 'CREATE SCHEMA ' || dest_schema ;
 
  -- TODO: Find a way to make this sequence's owner is the correct table.
  FOR object IN
    SELECT sequence_name::text FROM information_schema.SEQUENCES WHERE sequence_schema = source_schema
  LOOP
    EXECUTE 'CREATE SEQUENCE ' || dest_schema || '.' || object;
  END LOOP;
 
  FOR object IN
    SELECT table_name::text FROM information_schema.TABLES WHERE table_schema = source_schema
  LOOP
    buffer := dest_schema || '.' || object;
    EXECUTE 'CREATE TABLE ' || buffer || ' (LIKE ' || source_schema || '.' || object || ' INCLUDING CONSTRAINTS INCLUDING INDEXES INCLUDING DEFAULTS)';
   
    FOR column_, default_ IN
      SELECT column_name::text, replace(column_default::text, source_schema, dest_schema) FROM information_schema.COLUMNS where table_schema = dest_schema AND table_name = object AND column_default LIKE 'nextval(%' || source_schema || '%::regclass)'
    LOOP
      EXECUTE 'ALTER TABLE ' || buffer || ' ALTER COLUMN ' || column_ || ' SET DEFAULT ' || default_;
    END LOOP;
  END LOOP;
 
END;

$$ LANGUAGE plpgsql VOLATILE;
```

## Reference: https://wiki.postgresql.org/wiki/Clone_schema
