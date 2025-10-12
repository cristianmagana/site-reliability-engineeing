# Databases
>### CAP Theorem
<br>
The CAP theorem states that in the event of a network partition (failure) on a distributed database, it is possible to provide either consistency or availability—but not both. Network partitions are inevitable in distributed systems, so the theorem essentially forces you to choose between consistency and availability during network failures.
<br>
<br>

<b>Consistency</b> <br>
All the servers in the system will have the same data so users will get the same copy regardless of which server answers their request. Every read receives the most recent write or an error.

<b>Availability</b> <br>
The system will always respond to a request (even if it's not the latest data or consistent across the system). Every request receives a response, without guarantee that it contains the most recent write.

<b>Partition Tolerance</b> <br>
The system continues to operate as a whole even if individual servers fail or can't be reached. The system continues to function despite arbitrary network partitions between nodes.

<br>

<b>Real-World Example</b> <br>

Consider an e-commerce platform with servers in multiple data centers:

**CA System (Consistency + Availability)**: A traditional single-node relational database (like a standalone PostgreSQL or MySQL instance) provides both consistency and availability. All transactions are immediately consistent and the database is always available to serve requests. However, it sacrifices partition tolerance—if the single node fails or becomes unreachable, the entire system goes down. This is why CA systems are typically not truly distributed systems. In practice, most "CA" systems are actually single-server databases or systems within a single data center where network partitions are assumed not to occur.

**CP System (Consistency + Partition Tolerance)**: A banking transaction system prioritizes consistency. During a network partition between data centers, some nodes become unavailable (sacrificing availability) to ensure account balances are always accurate across all nodes. The system will reject requests or return errors rather than show potentially incorrect balances. You'd rather have the ATM say "service unavailable" than display an incorrect balance or allow overdrafts.

**AP System (Availability + Partition Tolerance)**: A shopping cart system prioritizes availability. During a network partition, customers can still add items to their cart on any available server (sacrificing strict consistency). Different data centers might temporarily show different cart contents, but they'll eventually synchronize when the partition is resolved. It's better to let customers continue shopping than to show error pages—a temporary inconsistency in cart items is acceptable.

<br>

<b>Why CAP Still Applies Today</b> <br>

The CAP theorem remains highly relevant in modern distributed systems:

- **Network partitions are inevitable**: Cloud infrastructure, microservices, multi-region deployments, and edge computing guarantee that network issues will occur regularly
- **Architectural trade-offs are required**: Every distributed database must make explicit choices about which guarantees to prioritize
- **Modern database examples**:
  - **Cassandra, DynamoDB, Riak** (AP): Favor availability and partition tolerance with eventual consistency
  - **MongoDB, HBase, Redis Cluster** (CP): Favor consistency and partition tolerance, becoming unavailable during partitions
  - **Google Spanner, CockroachDB**: Use sophisticated techniques (like atomic clocks and consensus protocols) to minimize trade-offs, but are still ultimately constrained by CAP during severe network failures

The theorem doesn't mean you can't have all three properties under normal conditions—it means you must choose which property to sacrifice when network partitions occur.

<br>


>### ACID
<br>

ACID is a set of properties that guarantee reliable processing of database transactions. These properties ensure data integrity and consistency even in the face of errors, power failures, or concurrent access. ACID compliance is a fundamental characteristic of traditional relational databases.

<br>
<br>

<b>Atomicity</b> <br>
An indivisible and irreducible series of database operations such that either all occurs, or nothing occurs. Transactions are "all-or-nothing"—if any part of the transaction fails, the entire transaction is rolled back and the database remains unchanged.

<b>Consistency</b> <br>
The database must transition from one valid state to another valid state, maintaining all defined rules, constraints, cascades, and triggers. This ensures that any transaction will bring the database from one consistent state to another, never leaving it in an invalid state that violates integrity constraints (such as foreign keys, unique constraints, or custom business rules).

<b>Isolation</b> <br>
Concurrent transactions execute independently without interfering with each other. The database ensures that concurrent execution of transactions leaves the database in the same state as if the transactions were executed sequentially. Isolation levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable) provide different trade-offs between consistency and performance.

<b>Durability</b> <br>
Once a transaction is committed, it will remain committed even in the case of system failures (power loss, crashes, or errors). This is typically achieved through write-ahead logging (WAL), where changes are written to a persistent log before being applied to the database.

<br>

<b>Real-World Example: Bank Transfer</b> <br>

Consider transferring $100 from Account A to Account B:

```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A';
  UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B';
COMMIT;
```

**Atomicity**: If the debit from Account A succeeds but the credit to Account B fails (due to a crash or error), the entire transaction is rolled back. You'll never have $100 disappear from the system or appear out of nowhere.

**Consistency**: Before and after the transaction, the total money in the system remains the same. Database constraints (like balance cannot be negative, or account must exist) are enforced. If Account B doesn't exist, the transaction fails and rolls back.

**Isolation**: If another transaction is checking the total balance of both accounts simultaneously, it will either see the state before the transfer (both original balances) or after the transfer (both updated balances)—never a state where money was debited but not yet credited.

**Durability**: Once the COMMIT succeeds and you receive confirmation, even if the server crashes one millisecond later, the transfer is permanently recorded. When the system restarts, both accounts will reflect the transfer.

<br>

<b>ACID in Modern Systems</b> <br>

ACID properties remain critical for applications requiring strong data integrity:

- **Traditional RDBMS** (PostgreSQL, MySQL, Oracle, SQL Server): Full ACID compliance with configurable isolation levels
- **NewSQL databases** (Google Spanner, CockroachDB, VoltDB): Provide ACID guarantees even in distributed environments
- **NoSQL trade-offs**: Many NoSQL databases sacrifice full ACID properties for scalability and performance:
  - **MongoDB**: ACID transactions within a single document; multi-document ACID added in version 4.0+
  - **Cassandra**: Eventually consistent by default; lightweight transactions available with performance cost
  - **DynamoDB**: ACID transactions available but with throughput limitations
- **Event sourcing systems**: Use append-only logs to achieve durability and enable temporal queries

**When ACID matters most**: Financial transactions, inventory management, order processing, medical records, any system where data corruption or loss is unacceptable.

**Trade-offs**: ACID compliance can reduce throughput and increase latency, especially in distributed systems. This is why many modern applications choose BASE (Basically Available, Soft state, Eventual consistency) for scenarios where eventual consistency is acceptable.

<br>

>### Normalization
<br>

Normalization entails organizing the columns and tables of a database to ensure that their dependencies are properly enforced by database integrity constraints.

<b>1NF</b> <br>
In simple terms, a single cell cannot hold multiple values. If a table contains a composite or multi-valued attribute, it violates the First Normal Form.  

<b>2NF</b> <br>
The first condition in the 2nd NF is that the table has to be in 1st NF. The table also should not contain partial dependency.

<b>3NF</b> <br>
The same rule applies as before i.e, the table has to be in 2NF before proceeding to 3NF. The other condition is there should be no transitive dependency for non-prime attributes. That means non-prime attributes (which doesn’t form a candidate key) should not be dependent on other non-prime attributes in a given table.

<b>Boyce Codd Normal Form (BCNF)</b> <br>
In BCNF if every functional dependency A → B, then A has to be the Super Key of that particular table.

<br>

> ### Relational Databases (SQL)
<br>
Known for storing data in spreadsheet-like tables that have their columns and data types strictly defined. The tables can have relationships between each other and the data is queried with SQL (Structured Query Language)

<br>
<br>

|  |  |
| --------        | ------ |
| Schema          | Fixed Schema I.e. Tables and Columns are preferred |
| Storage         |  Table (Row -> Entity, Column -> Attributes) |
| RDBMS           | Oracle, IBM DB2, MSSQL, MySQL |
| Query           | SQL | 
| Scalability     |  In common situation suitable for Vertical scaling |
| ACID Compliance | Compliant | 


<br>

> ### NoSQL
<br>

NoSQL databases (aka "not only SQL") are non-tabular databases designed to handle large volumes of data, high user loads, and flexible data models. They emerged to address the scalability and flexibility limitations of traditional relational databases. NoSQL databases typically follow the BASE model rather than ACID, trading strong consistency for availability and partition tolerance.

<br>
<br>

<b>NoSQL Database Types</b> <br>

**Document Stores** (MongoDB, CouchDB): Store data as JSON-like documents, allowing nested structures and flexible schemas. Ideal for content management, user profiles, and catalogs.

**Key-Value Stores** (Redis, DynamoDB, Riak): Simple key-value pairs offering extremely fast reads/writes. Perfect for caching, session management, and real-time analytics.

**Wide-Column Stores** (Cassandra, HBase, ScyllaDB): Organize data in columns rather than rows, optimized for analytical queries across massive datasets. Used for time-series data, IoT sensors, and event logging.

**Graph Databases** (Neo4j, Amazon Neptune, ArangoDB): Store data as nodes and relationships, optimized for traversing connections. Excellent for social networks, recommendation engines, and fraud detection.

<br>

|  |  |
| --------        | ------ |
| Schema          | Dynamic schema I.e. Can store data for entity and its attributes dynamically |
| Storage         | Key Value, Document, Graph, Wide-column |
| Query           | Database-specific APIs, some support SQL-like languages |
| Scalability     | In common situation suitable for Horizontal scaling |
| ACID Compliance | Compromises ACID properties, follows BASE model |

<br>

> ### BASE
<br>

BASE is an alternative to ACID that prioritizes availability and scalability over strict consistency. It's the consistency model typically used by NoSQL databases and distributed systems. BASE stands for Basically Available, Soft state, and Eventual consistency.

<br>
<br>

<b>Basically Available</b> <br>
The system guarantees availability by allowing partial failures without complete system shutdown. Rather than enforcing strict consistency, the system responds to requests even if some nodes are down or data is temporarily inconsistent. Availability is prioritized over consistency.

<b>Soft State</b> <br>
The state of the system may change over time, even without new input, due to eventual consistency. Data replicas may be temporarily inconsistent, and the system doesn't guarantee the state will be identical across all nodes at any given moment. The system's state is "soft" rather than "hard" (strictly defined).

<b>Eventual Consistency</b> <br>
Given enough time without new updates, all replicas will eventually converge to the same value. The system doesn't guarantee immediate consistency but promises that inconsistencies are temporary. Updates propagate through the system asynchronously, and clients may see stale data temporarily.

<br>

<b>Real-World Example: Social Media Feed</b> <br>

Consider a like counter on a popular social media post:

**Basically Available**: When you like a post, the system immediately responds with success even if not all global replicas have been updated. Users in different regions might see slightly different like counts, but the system remains operational.

**Soft State**: The like count might temporarily show different values to different users as updates propagate across data centers. User A in New York sees 1,523 likes while User B in Tokyo sees 1,519 likes for the same post at the same moment.

**Eventual Consistency**: After a short period (typically milliseconds to seconds), all replicas synchronize and converge to the same like count. Eventually, both User A and User B will see the same accurate count (e.g., 1,527 likes).

This is acceptable because:
- It's more important that users can always interact with content (availability)
- A temporarily incorrect like count doesn't impact user experience significantly
- The system can handle millions of concurrent operations by distributing load

<br>

<b>BASE in Modern Systems</b> <br>

BASE is widely adopted in systems where availability and scalability outweigh the need for immediate consistency:

- **Social media platforms** (Facebook, Twitter, Instagram): User feeds, like counts, follower counts
- **E-commerce** (Amazon): Product views, wish lists, shopping cart recommendations
- **Content delivery**: Video streaming services, news feeds, blog posts
- **Analytics systems**: Click tracking, user behavior monitoring, real-time dashboards
- **Distributed caches** (Redis, Memcached): Session data, API response caching

**ACID vs BASE Comparison**:
- **ACID**: Strong consistency, immediate accuracy, potential availability loss, suitable for financial transactions
- **BASE**: High availability, eventual accuracy, partition tolerant, suitable for user-facing features where temporary inconsistency is acceptable

**When BASE is appropriate**: User engagement metrics, social features, content distribution, real-time analytics, caching layers, any scenario where temporary inconsistency won't cause business or financial harm.

**Trade-offs**: BASE systems require careful design to handle conflicts when the same data is updated in multiple locations. Techniques like conflict-free replicated data types (CRDTs), vector clocks, and last-write-wins strategies help resolve conflicts.

<br>

> ### Scaling Considerations
<br>

Transaction workloads require a normalized design while analytical workloads require a denormalized design. 

A 3NF assures data consistency and accuracy but performance may be reduced due to the multiple joins involved.
<br>

<b> </b> <br>
<b> </b> <br>
<b> </b> <br>
<b> </b> <br>
<b> </b> <br>