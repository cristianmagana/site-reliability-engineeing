# SQL

>### CAPs Theorem
<br>
Distributed data stores that claims, in the event of a network failure on a distributed database, it is possible to provide either consistency or availability—but not both.
<br>
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

> ### Scaling Considerations
<br>
https://levelup.gitconnected.com/database-concerns-in-large-system-design-3f84b6331ff9

<br>

Transaction workloads require a normalized design while analytical workloads require a denormalized design. 

A 3NF assures data consistency and accuracy but performance may be reduced due to the multiple joins involved.
<br>

<b> </b> <br>
<b> </b> <br>
<b> </b> <br>
<b> </b> <br>
<b> </b> <br>