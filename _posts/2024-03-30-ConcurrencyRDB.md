---
title: "Concurrency control in RDB"

categories: 
  - RDB
  - Concurrency
  - MySQL
  - Lock
last_modified_at: now
---
# Concurrency control in RDB
### Optimistic lock vs Pessimistic lock

There are two types of locks that exist to solve concurrency problems in RDB.

The first one is pessimistic locking. It uses a lock provided by the RDB. If a clause such as 'SELECT FOR UPDATE' is used, the RDB puts an exclusive lock on selected rows so that other queries cannot access them. <br/>
Since it uses the lock provided by the database, there are possibilities of encountering deadlock situations.

On the other hand, optimistic locking is done at the application level rather than the database level. It uses the version of the row and employs it in the update clause.

```
UPDATE optimistic
SET name = newName AND version = 2
WHERE id = 1 AND version = 1
```

If two clients update at the same time, one client would encounter an error since it could not find the version in the WHERE clause. A version column is not necessary; the entire row information can be used for the update condition.

```
UPDATE optimistic
SET name = newName 
WHERE id = 1 AND name = originalName
```

Optimistic locking requires additional action when one thread fails, so it should be used when repeating the action is not a significant problem. <br/>
This is due to the assumption underlying optimistic locking, which assumes that conflicts are rare. Thus, it is suitable when conflicts are infrequent.

| Optimistic  | Pessimistic |
| ------------- | ------------- |
| ![OpRes50](/assets/images/OpResTime50.png)  | ![PeRes50](/assets/images/PeResTime50.png)  |


As you can see above, the response time is better for using optimistic locking in load tests with 50 threads. The maximum response time for optimistic locking is about 13 ms compared to 17 ms for pessimistic locking.

Therefore, it shows better performance when conflicts are rare, as it minimizes the overhead of managing locks and avoids blocking.

How about in the case of higher loads? The results differ as the thread count increases to 500. Optimistic locking starts to emit errors, resulting in only 370 successful samples out of 500. <br/>
The throughput of optimistic locking is 123 per second, which is lower than pessimistic locking's 166 per second. Most of the response times are under 14 ms for pessimistic locking (even though it lags much at first), while many response times for optimistic locking are over 15 ms.

| Optimistic  | Pessimistic |
| ------------- | ------------- |
| ![Op500](/assets/images/Op500.png)  | ![Pe500](/assets/images/Pe500.png)  |
| ![OpRes500](/assets/images/OpResTime500.png)  | ![PeRes500](/assets/images/PeResTime500.png)  |

In real life, users who receive errors would retry, or retry logic would be implemented in the application, causing higher loads, which leads to slower response times and lower TPS.

| Threads  | Result |
| ------------- | ------------- |
| 500  | ![Op500](/assets/images/Op500.png)  |
| 2000  | ![Op2000](/assets/images/Op2000.png)  |
| 3000  | ![Op3000](/assets/images/Op3000.png)  |

As seen from the results, TPS peaks at 2000 threads (160 per second) and declines at 3000 threads (102 per second), which is even lower than at 500 threads (123 per second).

Optimistic locking could be a good option if you can predict that it would rarely fail, with relatively lower traffic, and retrying is not a significant issue.

Pessimistic locking could be used when traffic is relatively higher, or you do not want users to retry. <br/><br/>

This article was created by Crocoder7. It is not to be copied without permission.
