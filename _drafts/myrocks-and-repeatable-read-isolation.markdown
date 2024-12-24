---
layout: post
title: MyRocks and Repeatable Read Isolation
tags:
- myrocks
- database
---

MyRocks doesn't support gap locks. This means in some cases, the database would assume it has the lock, while it actually doesn't. For Repeatable Read, usually the db will take a shared lock, and establish a snapshot on the first read. Due to the lack of gap lock, sometimes the snapshot is not really a snapshot, and locks are effectively ignored.

**This means something that works with Read Committed isolation level on MyRocks can break with Repeatable Read. Yes; read the previous sentence again.**

According to [MySQL reference](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html),

<!--kg-card-begin: markdown-->

> REPEATABLE READ
> 
> This is the default isolation level for InnoDB. Consistent reads within the same transaction read the snapshot established by the first read. This means that if you issue several plain (nonlocking) SELECT statements within the same transaction, these SELECT statements are consistent also with respect to each other. See Section 15.7.2.3, “Consistent Nonlocking Reads”.
> 
> For locking reads (SELECT with FOR UPDATE or FOR SHARE), UPDATE, and DELETE statements, locking depends on whether the statement uses a unique index with a unique search condition, or a range-type search condition.
> 
> - 
> 
> For a unique index with a unique search condition, InnoDB locks only the index record found, not the gap before it.
> 
> - 
> 
> For other search conditions, InnoDB locks the index range scanned, using gap locks or next-key locks to block insertions by other sessions into the gaps covered by the range. For information about gap locks and next-key locks, see Section 15.7.1, “InnoDB Locking”.

<!--kg-card-end: markdown-->

It basically means for non-locking `SELECT`s it provides effectively snapshot isolation. What if I do add locks for my `SELECT`s? If the index is not unique, well, too bad, because MyRocks doesn't have gap locks, it simply ignores the lock.

According to [MySQL reference](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_consistent_read),

<!--kg-card-begin: markdown-->

> With REPEATABLE READ isolation level, the snapshot is based on the time when the first read operation is performed. With READ COMMITTED isolation level, the snapshot is reset to the time of each consistent read operation.

<!--kg-card-end: markdown-->
## Example

Here's a concrete example where things can go wrong. Say, I have a table with three columns, `husband_id, wife_id, is_married` . Each person can only be married to at most one other person. `is_married` columns allows soft deletes (divorce). Now we have an incoming request to set `husband(a)` and `wife(b)` married on MyRocks.

If you are on MySQL with InnoDB, you can have a simple query that looks like the following (assuming we have an index on `husband_id, is_married`, which is not unique):

<!--kg-card-begin: markdown-->

    START TRANSACTION;
    SELECT wife_id FROM marriage
    WHERE husband_id = husband_b
          AND is_married = 1;
          
    # check we get zero rows; otherwise abort
    
    INSERT ...
    COMMIT;

<!--kg-card-end: markdown-->

This query doesn't work on MyRocks due to the lack of gap lock. One can write a query that looks like the following on MyRocks instead (assuming we have a unique covering index on `husband_id, is_married, wife_id` :

<!--kg-card-begin: markdown-->

    
    # grab a mutex, so no other transactions can change the state
    SELECT wife_id FROM marriage
    WHERE husband_id = husband_a
          AND is_married = 1
          AND wife_id = 0
    FOR UPDATE;
    
    SELECT wife_id FROM marriage
    WHERE husband_id = husband_a
          AND is_married = 1;
          
    # check we get zero rows; otherwise abort  
    
    INSERT ...
    COMMIT;

<!--kg-card-end: markdown-->

It works around the gap lock issue by holding a mutex. This only works if all clients mutating the table coordinate together (know which mutex to hold). This query doesn't work with Repeatable Read on MyRocks however. According to the documentation for MySQL RR:

> [Consistent reads](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_consistent_read) within the same transaction read the [snapshot](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_snapshot) established by the first read.

The "first read" here is the locking mutex read, without gap lock. This means that the first locking read might establish a stale snapshot for the second read. It doesn't affect Read Committed because

> Each consistent read, even within the same transaction, sets and reads its own fresh snapshot.

## Why should I care

To avoid the following

<figure class="kg-card kg-image-card kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2021/05/5a2hto.jpg" class="kg-image" alt loading="lazy"><figcaption>RC = Read Committed, RR = Repeatable Read</figcaption></figure>