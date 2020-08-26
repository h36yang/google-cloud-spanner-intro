# Getting Started with Cloud Spanner
Learning notes from "Creating and Administering Google Cloud Spanner Instances" course

## What is Google Cloud Spanner
_A global, horizontally scaling, strongly consistent relational database service built on proprietary technology._

## Cloud Spanner vs. Other GCP Technologies
* **Cloud SQL** - regional RDBMS with limited scaling
* **BigQuery** - data warehousing (OLAP not OLTP)
* **BigTable** - NoSQL wide column store
* **Datastore** - NoSQL document database
* **Cloud Storage Buckets** - blob storage

|Cloud Spanner|Cloud SQL|
|---:|:---|
|Relational (ACID++ Support)|Relational (ACID Support)|
|Regional or Global|Regional Only|
|Scales Horizontally|Scales Vertically|
|Scales to Any Size, IOPS|Smaller, Lower IOPS|
|Google Proprietary|MySQL and PostgreSQL|
|Relatively Expensive|Relatively Cheap|

|Cloud Spanner|BigQuery|
|---:|:---|
|Access using SQL|Access using SQL|
|Scales to Petabytes|Scales to Petabytes|
|Horizontal scaling with replication|Horizontal scaling with replication|
|Strong Consistency|Eventual Consistency|
|Strong ACID Support|Not quite ACID|
|Requires Cluster Creation|Serverless|
|Keys indexed, can add secondary indexes|No indexes, no provisioning|
|Transactional Processing (OLTP)|Analytical Processing (OLAP)|

## Horizontal Scaling and Strong Consistency
_Cloud Spanner is virtually the only horizontally scaling RDBMS to offer strong consistency._
* **TrueTime** - highly available, distributed clock for applications on the GCP
* **External Consistency** - system behaves like all transactions were executed sequentially on one server

## Instances and Nodes
* Instance Configuration
  * Cannot be changed after creation
  * Regional Instances (99.99% availability):
    * All data in same region
    * 3 Read-Write replicas in different zones
  * Multi-Regional Instances (99.999% availability):
    * Data replicated across regions
    * Complex replication scheme
* Node Count - 2TB/node
  * Recommended minimum of 3 nodes for production
  * Can change node count after creation
  * Nodes are _not the same as replicas_

## Schema and Data Model
_Cloud Spanner uses interleaved parent-child tables to store records **together**. This ensures fast retrieval if data is commonly accessed together._
* Can define interleaved tables 7 levels deep
* Spanner identifies hot spots and splits on the key of the parent table
* Additional splits are created automatically as needed
* Load based splitting
* Splits are replicated individually

## Replication
* Spanner is built on a replicated filesystem
  * Spanner writes database mutations to file
  * Filesystem takes care of replication, recovery
* In addition, Spanner further replicates data
  * Spanner database replicas are called _splits_
  * Splits depend on data semantics
  * Splitting semantics are defined using parent-child relationships
    * Interleave rows from different tables and group into splits
    * User can control splits by choice of primary keys and parent-child relationships
  * Sets of splits stored and replicated using **Paxos** (standard consensus algorithm)
    * Every split has a designated leader, which is responsible for handling writes to that split

### 3 Types of Replicas
* **Read-Write** - can become leader and can vote
  * Maintain full copy of data
  * Serve reads too
  * Can vote on whether to commit write
  * Participate in leadership election
  * Single-region instances contain only these
* **Read-Only** - cannot become leader or vote
  * Only used in multi-region instances
  * Maintain full copy of data
  * Replicated from Read-Write replicas
  * Cannot vote on commits
  * Cannot become leader
  * Serve both _strong reads_ and _stale reads_
  * Stale reads can be served without round-trip to leader
  * Strong reads must incur delay of round-trip to leader
* **Witness** - cannot become leader but can vote
  * Only used in multi-region instances
  * Do not maintain full copy of data
  * Vote to commit writes
  * Vote in leadership election
  * Cannot become leader

## Best Practices
**Primary Key Design**
* Choose carefully to avoid hotspots
* Avoid monotonically increasing/decreasing values, e.g. auto-increment keys

**Data Loading**
* Avoid writing rows in primary key order
* Partition ordered key ranges then write in batches
  * Allows well-distributed writes

**Limit Row Size**
* Row should be 4GB or less. This includes:
  * Top-level row
  * All interleaved child rows
  * Index rows

**Interleaved Tables**
* Parent and child table rows will be physically co-located
* Create hierarchy of interleaved tables
* Reduce hotspotting
* Do not create non-interleaved indexes on monotonically changing data
  * Fine to use interleaved index on such column

# Deep Dives
[Creating and Managing Tables in Cloud Spanner](Creating_Managing_Tables.md)

[Integrating Cloud Spanner with Other Google Cloud Services](Integrating_Other_Services.md)
