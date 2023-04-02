# Databases
>### CAP Theorem
<br>
Distributed data stores that claims, in the event of a network failure on a distributed database, it is possible to provide either consistency or availability—but not both.
<br>
<br>

<b>Consistency</b> <br>
 All the servers in the system will have the same data so users will get the same copy regardless of which server answers their request.

<b>Availability</b> <br>
 The system will always respond to a request (even if it's not the latest data or consistent across the system or just a message saying the system isn't working).

<b>Partition Tolerance</b> <br>
 The system continues to operate as a whole even if individual servers fail or can't be reached.

<br>


>### ACID 
<br>

<b>Atomicity</b> <br> 
An indivisible and irreducible series of database operations such that either all occurs, or nothing occurs.

<b>Consistency</b> <br> 
Understood as after a successful write, update or delete of a Record, any read request immediately receives the latest value of the Record.

<b>Isolation</b> <br>
Any reads or writes performed on the database will not be impacted by other reads and writes of separate transactions occurring on the same database. A global order is created with each transaction queueing up in line to ensure that the transactions complete in their entirety before another one begins.

<b>Durability</b> <br>
Durability ensures that changes made to the database (transactions) that are successfully committed will survive permanently, even in the case of system failures (e.g.; change logs).

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
NoSQL databases (aka "not only SQL") are non-tabular databases and store data differently than relational tables. NoSQL databases come in a variety of types based on their data model. The main types are document, key-value, wide-column, and graph. They provide flexible schemas and scale easily with large amounts of data and high user loads.
<br>
    
    <br>

|  |  |
| --------        | ------ |
| Schema          | Dynamic schema I.e. Can store data for entity and its attributes dynamically |
| Storage         | Key Value, Document, Graph, Wide-column |
| Query           | unSQL | 
| Scalability     | In common situation suitable for Horizontal scaling |
| ACID Compliance | Compromises ACID properties | 

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