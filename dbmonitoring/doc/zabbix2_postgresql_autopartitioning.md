Auto Partitioning with Zabbix 2.0 and Postgresql
================================================

> Taken from the [Zabbix Wiki](https://www.zabbix.org/wiki/Docs/howto/zabbix2_postgresql_autopartitioning)

Here is my take on Zabbix and Postgresql 9.x (auto) partitioning.

This approach:

* does not require you to prepare the database to partition it with zabbix
* does not require you to create/schedule a cron job for creating the tables in advance
* seems a bit simpler to implement than other solutions.

It will auto create partitions under the "partition" schema with the following name convention

  partitions.tablename_pYYYYMMDD ''' # for DAILY   partitions '''
  partitions.tablename_pYYYYMM   ''' # for MONTHLY partitions '''

Create the partitions schema
----------------------------

The partitioned tables will be created under the "partitions" schema, which you can create with:

<source lang="plsql">
-- Schema: partitions

-- DROP SCHEMA partitions;

CREATE SCHEMA partitions
  AUTHORIZATION zabbix;
</source>

Grant authorization to the PostgreSQL account used by Zabbix. See DBUser in ''zabbix_server.conf''.

Create the main function with the following code
------------------------------------------------

<source lang="plsql">
-- Function: trg_partition()

-- DROP FUNCTION trg_partition();

CREATE OR REPLACE FUNCTION trg_partition()
  RETURNS trigger AS
$BODY$
DECLARE
prefix text := 'partitions.';
timeformat text;
selector text;
_interval interval;
tablename text;
startdate text;
enddate text;
create_table_part text;
create_index_part text;
BEGIN

selector = TG_ARGV[0];

IF selector = 'day' THEN
timeformat := 'YYYY_MM_DD';
ELSIF selector = 'month' THEN
timeformat := 'YYYY_MM';
END IF;

_interval := '1 ' || selector;
tablename :=  TG_TABLE_NAME || '_p' || to_char(to_timestamp(NEW.clock), timeformat);

EXECUTE 'INSERT INTO ' || prefix || quote_ident(tablename) || ' SELECT ($1).*' USING NEW;
RETURN NULL;

EXCEPTION
WHEN undefined_table THEN

startdate := extract(epoch FROM date_trunc(selector, to_timestamp(NEW.clock)));
enddate := extract(epoch FROM date_trunc(selector, to_timestamp(NEW.clock) + _interval ));

create_table_part:= 'CREATE TABLE IF NOT EXISTS '|| prefix || quote_ident(tablename) || ' (CHECK ((clock >= ' || quote_literal(startdate) || ' AND clock < ' || quote_literal(enddate) || '))) INHERITS ('|| TG_TABLE_NAME || ')';
create_index_part:= 'CREATE INDEX '|| quote_ident(tablename) || '_1 on ' || prefix || quote_ident(tablename) || '(itemid,clock)';

EXECUTE create_table_part;
EXECUTE create_index_part;

--insert it again
EXECUTE 'INSERT INTO ' || prefix || quote_ident(tablename) || ' SELECT ($1).*' USING NEW;
RETURN NULL;

END;
$BODY$
  LANGUAGE plpgsql VOLATILE
  COST 100;
ALTER FUNCTION trg_partition()
  OWNER TO postgres;
</source>

Create a trigger for each (clock based) table you want to partition
-------------------------------------------------------------------

<source lang="plsql">
CREATE TRIGGER partition_trg BEFORE INSERT ON history           FOR EACH ROW EXECUTE PROCEDURE trg_partition('day');
CREATE TRIGGER partition_trg BEFORE INSERT ON history_uint      FOR EACH ROW EXECUTE PROCEDURE trg_partition('day');
CREATE TRIGGER partition_trg BEFORE INSERT ON history_str       FOR EACH ROW EXECUTE PROCEDURE trg_partition('day');
CREATE TRIGGER partition_trg BEFORE INSERT ON history_text      FOR EACH ROW EXECUTE PROCEDURE trg_partition('day');
CREATE TRIGGER partition_trg BEFORE INSERT ON history_log       FOR EACH ROW EXECUTE PROCEDURE trg_partition('day');
CREATE TRIGGER partition_trg BEFORE INSERT ON trends            FOR EACH ROW EXECUTE PROCEDURE trg_partition('month');
CREATE TRIGGER partition_trg BEFORE INSERT ON trends_uint       FOR EACH ROW EXECUTE PROCEDURE trg_partition('month');
</source>

Disable partitioning
--------------------

Should you want to remove the partitioning, just remove the partition_trg from each table, or run the following
<source lang="plsql">
DROP TRIGGER partition_trg ON history;
DROP TRIGGER partition_trg ON history_uint;
DROP TRIGGER partition_trg ON history_str;
DROP TRIGGER partition_trg ON history_text;
DROP TRIGGER partition_trg ON history_log;
DROP TRIGGER partition_trg ON trends;
DROP TRIGGER partition_trg ON trends_uint;
</source>

Remove unwanted old partitions
------------------------------

The following optional routine is to delete partitions older than the desired time.
Unfortunately it requires you to schedule it using "cron" or run it manually. (SEE BELOW)

<source lang="plsql">
-- Function: delete_partitions(interval, text)

-- DROP FUNCTION delete_partitions(interval, text);

CREATE OR REPLACE FUNCTION delete_partitions(intervaltodelete interval, tabletype text)
  RETURNS text AS
$BODY$
DECLARE
result record ;
prefix text := 'partitions.';
table_timestamp timestamp;
delete_before_date date;
tablename text;

BEGIN
    FOR result IN SELECT * FROM pg_tables WHERE schemaname = 'partitions' LOOP

        table_timestamp := to_timestamp(substring(result.tablename from '[0-9_]*$'), 'YYYY_MM_DD');
        delete_before_date := date_trunc('day', NOW() - intervalToDelete);
        tablename := result.tablename;

    -- Was it called properly?
        IF tabletype != 'month' AND tabletype != 'day' THEN
      RAISE EXCEPTION 'Please specify "month" or "day" instead of %', tabletype;
        END IF;


    --Check whether the table name has a day (YYYY_MM_DD) or month (YYYY_MM) format
        IF length(substring(result.tablename from '[0-9_]*$')) = 10 AND tabletype = 'month' THEN
            --This is a daily partition YYYY_MM_DD
            -- RAISE NOTICE 'Skipping table % when trying to delete "%" partitions (%)', result.tablename, tabletype, length(substring(result.tablename from '[0-9_]*$'));
            CONTINUE;
        ELSIF length(substring(result.tablename from '[0-9_]*$')) = 7 AND tabletype = 'day' THEN
            --this is a monthly partition
            --RAISE NOTICE 'Skipping table % when trying to delete "%" partitions (%)', result.tablename, tabletype, length(substring(result.tablename from '[0-9_]*$'));
            CONTINUE;
        ELSE
            --This is the correct table type. Go ahead and check if it needs to be deleted
      --RAISE NOTICE 'Checking table %', result.tablename;
        END IF;

  IF table_timestamp <= delete_before_date THEN
    RAISE NOTICE 'Deleting table %', quote_ident(tablename);
    EXECUTE 'DROP TABLE ' || prefix || quote_ident(tablename) || ';';
  END IF;
    END LOOP;
RETURN 'OK';

END;

$BODY$
  LANGUAGE plpgsql VOLATILE
  COST 100;
ALTER FUNCTION delete_partitions(interval, text)
  OWNER TO postgres;
</source>

You can then remove old partition using the following commands
<source lang="sql">
SELECT delete_partitions('7 days', 'day')
SELECT delete_partitions('11 months', 'month')
</source>
