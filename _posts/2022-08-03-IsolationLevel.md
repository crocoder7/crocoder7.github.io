---
title: "DB Isolation Level"

categories: 
  - DB
last_modified_at: now
---
# What is isolation level?
* Isolation describes how changes applied by concurrent transactions are visible to each other.
* There are four types of isolation level
  <ol>
    <li>READ_UNCOMMITTED : It is possible for one transaction read a uncommited data that is modified by other transaction. It causes dirty read. It happens when data has been read by other transaction and rollbacked without commiting</li>
    <li>READ_COMMITTED : Only committed data can be seen. It can cause unrepeatable read.(data inconsistency) For example, when a column is read twice within a transaction, values can be different each other.</li>
    <li>REPEATABLE_READ : Data read is consistent within a single transaction. Even if other concurrent transaction changes a data within transaction, the data read within the transaction is consistent since it sees a undo log.(MVCC)</li>
    <li>SERIALIZABLE : It guarantees serial transaction. It puts shared lock on the row when read so that other transaction can not put exclusive lock to update that row. It can easily cause dead lock.</li>
  </ol>

* Reference
  * [Postgresql Docs](https://www.postgresql.org/docs/current/transaction-iso.html)

# Read-Modify-Write Cycle
* What happends when concurrent transactions try to change same data column?
* Data race can occur on the situation above.
* For example, when two transaction try to withdraw from the same bank account concurrently, second transaction can override the first transaction.

| Transaction A      | Transaction B |
| ----------- | ----------- |
| Begin| Begin|
| SELECT balance FROM account WHERE account_id = 1; (returns 10) ||
|| SELECT balance FROM account WHERE account_id = 1; (also returns 10)|
|UPDATE account SET balance = 5 WHERE account_id = 1; (returns 5)| |
||UPDATE account SET balance = 20 WHERE account_id = 1; (returns 20)|
| Commit| Commit|

* There are four solutions to this
  <ul>
  <li>Avoid read-modify-write cycle</li>
  <li>SELECT FOR UPDATE</li>
  <li>SERIALIZABLE isolation level</li>
  <li>Optimistic locking</li>
  </ul>

### Avoid read-modify-write cycle
* Instead of read-modify-write, just update.
  * UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
* However, it cannot be used since there can be complex conditions to apply in enterprise product.

### SELECT FOR UPDATE
* This query will put a lock on the row so that other transactions wait until the first transaction ends.
* It is recommended to use SERIALIZABLE transaction instead since protection is provided.

### SERIALIZABLE isolation level
* Other transaction try to change the same data would be rollbacked.

| Transaction A      | Transaction B |
| ----------- | ----------- |
| Begin      | Begin       |
| SELECT balance FROM account WHERE account_id = 1; (returns 10) ||
|| SELECT balance FROM account WHERE account_id = 1; (also returns 10)|
|UPDATE account SET balance = 5 WHERE account_id = 1; (returns 5)| |
||UPDATE account SET balance = 20 WHERE account_id = 1; (gets stuck here)|
| Commit   ||
||(could not serialize access due to concurrent update error)|
||(ROLLBACK)|

### Optimistic locking
* It verifies that the data has been changed by other transaction before commit.
* It follows steps below
  <ol>
  <li>Begin</li>
  <li>Modify</li>
  <li>Validate</li>
  <li>Commit/Rollback</li>
  </ol>
* For validate process, version column or last modified column would be needed.(often handled by ORM like Hibernate)

| Transaction A      | Transaction B |
| ----------- | ----------- |
| Begin| Begin|
| SELECT balance, version FROM account WHERE account_id = 1; (returns 10, 1) ||
|| SELECT balance, version FROM account WHERE account_id = 1; (also returns 10, 1)|
| Commit| Commit|
| Begin| Begin|
|UPDATE account SET balance = 5, version = 2 WHERE account_id = 1 AND version = 1; (returns 5)| |
||UPDATE account SET balance = 20 WHERE account_id = 1 AND version = 1; (matching zero rows)|
| Commit   ||
||(ROLLBACK)|

* References
  * [Blog](https://www.2ndquadrant.com/en/blog/postgresql-anti-patterns-read-modify-write-cycles/)
  * [Wikipedia](https://en.wikipedia.org/wiki/Optimistic_concurrency_control)
