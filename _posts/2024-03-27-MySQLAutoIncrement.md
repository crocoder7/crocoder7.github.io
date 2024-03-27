---
title: "MySQL Auto-Inremented PK"

categories: 
  - MySQL
  - AutoIncrement
  - Index
last_modified_at: now
---
# Why use MySQL auto-incremented values as primary keys?.

It is quite a convention to use auto-incremented values as PK in MySQL. <br/>
What if we use UUID as a primary key? Would there be any problems? <br/>
Here are some pros and cons for using UUID as PK. <br/><br/>

### Pros
1. Like it's name, it's universally unique across the whole system unlike auto-incremented values are unique only within a single table.
2. Security wise, it would make harder for attackers to figure out PK. Many APIs contains PK as a part of it's URI. (e.g. /user/{userPK})

### Cons
1. Storage overhead due to bigger size of the column. (8bytes for bigint, 32bytes for varchar(32) UUID)
2. Index performance degradation due to bigger size of index. <br/>
   It would require extra I/O to retrieve extra page of data from DB. <br/>
   It would also use more memory in the buffer pool, leading to increased memory pressure and a lower cache hit ratio.
3. Join Queries would less efficient since NL joins use indexs for join query. Like mentioned above, require extra memory and I/O.
4. Index fragmentation would arise over time. <br/>
   It would require non-sequential access since the data in the index are not physically adjacent to each other(in SSD or HDD), thus increasing disk I/O.
<br/>

It would be better to use an auto-incremented value as the primary key (PK) for performance reasons. <br/> 
Also, the benefits of using UUID as the primary key are somewhat trivial compared to using an auto-incremented value.
