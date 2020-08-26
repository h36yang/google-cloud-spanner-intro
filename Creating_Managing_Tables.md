# Creating and Managing Tables in Cloud Spanner

## Creating Interleaved Table
```sql
CREATE TABLE [child_table] (
    Col1 STRING(1024) NOT NULL,
    Col2 STRING(1024) NOT NULL,
    Col3 INT64
)
PRIMARY KEY (Col1, Col2), /* PK of parent table needs to be prefix of child table PK */
INTERLEAVE IN PARENT [parent_table] ON DELETE CASCADE
;
```

## Reads
### 2 Types of Reads
* **Strong Reads** (default) - reading app needs to see all writes so far
* **Stale Reads** - for apps that are tolerant of stale data but latency sensitive

### 3 Types of Read Operations
* **Read-Write Transaction** - if you need to write based on the value of one or more reads
* **Read-Only Transaction** - if you need to make multiple reads that require consistent view of data
* **Single Read Method** - single, standalone read call(s)

### Timestamp Bound
* Tells Spanner how to choose which data to return
* **3 Types of Timestamp Bounds**
  * **Strong** (default) - for strong reads
  * **Bounded** - for bounded stale reads
  * **Exact** - for exact stale reads

## Cloud Spanner Transactions
_A transaction in Cloud Spanner is a set of reads and writes that execute atomically at a single logical point in time across columns, rows, and tables in a database._

### 3 Types of Transactions
* **Locking Read-Write** - pessimistic locking; need to commit and might abort
  * Write that depends on values of reads
  * Multiple writes that need to be atomic
  * Conditional writes
* **Read-Only** - can read past timestamps; do not commit or acquire locks
  * Multiple reads at same timestamp
  * Multiple reads requiring consistency
* **Partitioned DML** - used for bulk updates, deletes, backfilling defaults and cleanup
  * Large scale UPDATE or DELETE
  * Cannot rollback
  * Spanner efficiently partitions key space
  * Efficiently executes without locking entire database

|DML|Partitioned DML|
|---:|:---|
|Regular transaction processing|Database-wide operations|
|Individual user transactions affecting small subset of database|Bulk updates/deletes, default values|
|Usually not idempotent|Must be idempotent|
|Can contain multiple statements|Single DML statement|
|Any statements, however complex|Must be **fully partitionable**|
|User creates transactions|Spanner creates transactions|

### Fully Partitionable Statement
_Expressible as a union of a set of statements, where each such statement accesses a single row of a table and no other table (no self-joins, multiple table statements)._

## Interacting via Python
* Install required package
```sh
pip install google-cloud-spanner
```
* Imports:
```python
from google.cloud import spanner
from google.cloud.spanner_v1 import param_types
```
* Initiate client and connect to instance/database:
```python
spanner_client = spanner.Client()
instance = spanner_client.instance("<INSTANCE_ID>")
database = instance.database("<DATABASE_ID>")
```
* Execute DDL commands:
```python
operation = database.update_ddl([
    "<COMMAND_1>",
    "<COMMAND_2>",
    ...
    "<COMMAND_n>",
]) # this is a synchronous call
print("Waiting for operation to complete...")
operation.result() # blocked until operation is finished
print("Operation completed.")
```
* Insert Data:
```python
with database.batch() as batch:
  batch.insert(
    table="<TABLE_NAME>",
    columns=("<COL_1>", "<COL_2>", ..., "<COL_n>"),
    values=[
      ("<VAL_1>", "<VAL_2>", ..., "<VAL_n>"),
      ("<VAL_1>", "<VAL_2>", ..., "<VAL_n>"),
      ("<VAL_1>", "<VAL_2>", ..., "<VAL_n>"),
      ...
    ]
  )
```
* Update Data:
```python
with database.batch() as batch:
  batch.update(
    table="<TABLE_NAME>",
    columns=("<COL_1>", "<COL_2>", ..., "<COL_n>"), # must contain key columns for Spanner to know which row(s) to update
    values=[
      ("<VAL_1>", "<VAL_2>", ..., "<VAL_n>"),
      ("<VAL_1>", "<VAL_2>", ..., "<VAL_n>"),
      ("<VAL_1>", "<VAL_2>", ..., "<VAL_n>"),
      ...
    ]
  )
```
* Delete Data:
```python
to_delete = spanner.KeySet(keys=[
    ["<KEY_1_VAL_1>", "<KEY_2_VAL_1>"],
    ["<KEY_1_VAL_2>", "<KEY_1_VAL_2>"],
    ...
    ["<KEY_1_VAL_n>", "<KEY_1_VAL_n>"],
  ])
with database.batch() as batch:
  batch.delete("<TABLE_NAME>", to_delete)
```
* Read Data:
  * Strong Read:
  ```python
  with database.snapshot() as snapshot:
    keyset = spanner.KeySet(all_=True)
    results = snapshot.read(
      table="<TABLE_NAME>",
      columns=("<COL_1>", "<COL_2>", ..., "<COL_n>"),
      keyset=keyset
    )

    for row in results:
      print("<COL_1>: {}, <COL_2>: {}, ... <COL_n>: {}".format(*row))
  ```
  * Stale Read:
  ```python
  import datetime
  staleness = datetime.timedelta(seconds=15)

  with database.snapshot(exact_staleness=staleness) as snapshot:
    ...
  ```
  * Read by Key Range:
  ```python
  with database.snapshot() as snapshot:
    keyrange = spanner.KeyRange(
      start_closed=["KEY_VAL_x"],
      end_closed=["KEY_VAL_y"]
    )
    # create keyset from keyrange
    keyset = spanner.KeySet(ranges=[keyrange])
    ...
  ```
  * Reads in Read-Only Transaction:
  ```python
  with database.snapshot(multi_use=True) as snapshot:
    # all reads in this snapshot will use the same timestamp across
    ...
  ```
* Read-Write Transaction:
```python
# pass in transaction as parameter, instantiated by Spanner
def some_function(transaction):
  # regular CRUD syntax using transaction object
  transaction.insert(...)
  transaction.read(...)
  transaction.update(...)
  transaction.delete(...)

# pass the function to Spanner as callback
database.run_in_transaction(some_function)
```

## Secondary Index
* Create a secondary index:
```sql
CREATE INDEX <INDEX_NAME>
          ON <TABLE_NAME> (<COL_1>, <COL_2>, ...)
;
```
* Create a secondary index with nonkey columns:
```sql
CREATE INDEX <INDEX_NAME>
          ON <TABLE_NAME> (<COL_1>, <COL_2>, ...)
     STORING (<COL_x>, <COL_y>, ...)
;
```
* Use a secondary index:
  * Cloud Spanner does not automatically choose a secondary index if any non-index columns are queried
  * Option 1:
  ```python
  results = snapshot.execute_sql(
    "SELECT <COL_1>, <COL_2>, ..., <COL_n> "
    "FROM <TABLE_NAME>@{FORCE_INDEX = <INDEX_NAME>} "
    "WHERE <SOME_CONDITION>",
    params=params,
    param_types=param_types
  )
  ```
  * Option 2:
  ```python
  results = snapshot.read(
    table="<TABLE_NAME>",
    columns=("<COL_1>", "<COL_2>", ..., "<COL_n>"),
    keyset=keyset,
    index="<INDEX_NAME>"
  )
  ```

## Commit Timestamp
_The time when a transaction is committed in the database. Used to know when exactly a record changed (based on TrueTime technology)._

### How to use Commit Timestamp
* Set `allow_commit_timestamp` on a TIMESTAMP column
```sql
ALTER TABLE <TABLE_NAME>
  ADD COLUMN LAST_UPDATE TIMESTAMP OPTIONS(allow_commit_timestamp=true)
;
```
* Write timestamp as part of transaction
```python
with database.batch() as batch:
  batch.update(
    table="<TABLE_NAME>",
    columns=("<COL_1>", "<COL_2>", "LAST_UPDATE"),
    values=[
      ("<COL_1_VAL_1>", "<COL_2_VAL_1>", spanner.COMMIT_TIMESTAMP),
      ("<COL_1_VAL_2>", "<COL_2_VAL_2>", spanner.COMMIT_TIMESTAMP),
      ...
      ("<COL_1_VAL_n>", "<COL_2_VAL_n>", spanner.COMMIT_TIMESTAMP)
    ]
  )
```
