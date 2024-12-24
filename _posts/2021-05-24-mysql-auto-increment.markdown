---
layout: post
title: How `AUTO_INCREMENT` works on MySQL
date: '2021-05-24 23:00:41'
tags:
- mysql
---

You have created tables with `AUTO_INCREMENT`. Something like

<!--kg-card-begin: markdown-->

    CREATE TABLE user (
      user_id INT(11) NOT NULL AUTO_INCREMENT,
      ...
    )

<!--kg-card-end: markdown-->

MySQL manages `user_id` for you automatically. You get user\_id 1, 2, 3, 4, ... But how does it work?

InnoDB, one of the most common storage engine MySQL uses, supports three different lock modes for AUTO\_INCREMENT – traditional, consecutive and interleaved.

The lock modes are named this way because it has something to do with SBR – statement based replication. If the mysql primary inserted two rows with user\_id 1 and 2, you would want the replica to have 1 and 2 be assosicated with the correct user row instead of interleaved.

## Traditional lock mode

This is the most straightforward one. It acquires a table-level lock for inserts. The lock is held until the end of the insert statement you are running. Since transactions will be replicated in the same order (or as if in the same order) as they are executed, "traditional" lock mode works for SBR setup. But as you can tell, it's also slow.

## Consecutive lock mode

This is the default InnoDB lock mode until MySQL 8.0. It still uses table-level lock for inserts but not always. For example, if MySQL knows the number of rows will be inserted (e.g. insert one row), it can just allocate ids to the transaction and move on. Practically it means MySQL only needs to hold a lock for the duration of the allocation (not until the end of the insert statement). What if MySQL allocates the ids to some transaction and it later got rolled back? These ids are simply lost. MySQL doesn't guarantee that there are no gaps in an auto\_increment column.

## Interleaved lock mode

This is the default for MySQL 8.0. There are no locks whatsoever. Think of it as an `std::atomic` essentially. Of course this is the fastest option. As a result, it's not safe to run interleaved lock mode with SBR. Otherwise it's possible for user id 1 to have user id 2's user name on the replica.

Reference: [https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html#innodb-auto-increment-lock-modes](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html#innodb-auto-increment-lock-modes)

