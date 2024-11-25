---
title: Solutions to the Lost Update Problem
date: 2024-11-25 12:00:00
---

The **Lost Update Problem** is a common issue in concurrent systems, where two transactions read the same data, modify it, and write it back to the database. The second transaction will overwrite the changes made by the first transaction, causing the first transaction's changes to be lost.

This is especially problematic in cases where money in a bank account is involved, as it can lead to inconsistencies in the account balance. A rather dramatic example of this is the [Flexcoin bankruptcy](https://www.reuters.com/article/technology/bitcoin-bank-flexcoin-shuts-after-hacking-theft-idUSBREA2329B/). 

<!--more-->

For example, in MySQL the default isolation level is `REPEATABLE READ`, which means that a transaction will read the same data multiple times and get the same result. However, this does not prevent the Lost Update Problem, as shown in this sequence diagram:

```plantuml
@startuml
Alice -> DB : set session transaction isolation level repeatable read;\nbegin;
Bob -> DB : set session transaction isolation level repeatable read;\nbegin;
Alice -> DB : select * from entry where id = 1;

note over Alice
  Result:
  | id | value |
  |  1 | 10    |
end note


Alice -> DB : update entry set value = value - 5 where id = 1;\ncommit;

note over Alice
  Result:
  | id | value |
  |  1 | 5    |
end note

Bob -> DB : select * from entry where id = 1;

note over Bob
    Bob still sees the initial value, since he is isolated from Alice's transaction:
    | id | value |
    |  1 | 10    |
end note


Bob -> DB : update entry set value = value - 10 where id = 1;\ncommit;

note over Bob
      Bob overwrote Alice's changes:
      | id | value |
      |  1 | 0    |
end note

@enduml
```

Fortunately, there are several ways to solve the Lost Update Problem:

## Isolation Level `SERIALIZABLE` 
 
`SERIALIZABLE` is the strictest isolation level, which prevents the Lost Update Problem by bailing out transactions that would cause conflicts:
```plantuml
@startuml
Alice -> DB : set session transaction isolation level serializable;\nbegin;
Bob -> DB : set session transaction isolation level serializable;\nbegin;
Alice -> DB : select * from entry where id = 1;

note over Alice
  Alice's transaction now holds a shared lock on the row.
  It's the same as `SELECT ... FOR SHARE`.
end note

Bob -> DB : select * from entry where id = 1;

note over Bob
  Bob can read the row.
end note

Alice -> DB : update entry set value = 'T1' where id = 1;

Bob -> DB : update entry set value = 'T2' where id = 1;

note over Bob
  Bob get' an error: [40001][1213] Deadlock found when trying to get lock; try restarting transaction
end note

@enduml
```
In MySQL, `SERIALIZABLE` is the same as `SELECT ... FOR SHARE`, which means reads are not blocked, but writes are. This can lead to deadlocks, as shown in the sequence diagram above.

## SELECT ... FOR UPDATE

`SELECT ... FOR UPDATE` is a way to lock rows for writing, which prevents other transactions from reading or writing the same rows. This is useful when you want to prevent the Lost Update Problem, but do not want to use the `SERIALIZABLE` isolation level:

```plantuml
@startuml
Alice -> DB : \nbegin;
Bob -> DB : set begin;
Alice -> DB : select * from entry where id = 1 for update;

note over Alice
Result:
| id | value |
|  1 | 10     |
end note

Bob -> DB : select * from entry where id = 1 for update;

note over Bob
  Bob's transaction is blocked until Alice's transaction is committed or rolled back.
end note

Alice -> DB : update entry set value = value - 5 where id = 1;

Alice -> DB : commit;

note over Alice
  Result:
  | id | value |
  |  1 | 5    |

  Bob's transaction is unblocked. He get's the same result as Alice.

end note

Bob -> DB : update entry set value = value - 5 where id = 1;

note over Bob
  Bob's transaction is commited

  Result:
  | id | value |
  |  1 | 0    |
end note

@enduml
```

## Optimistic Locking

This approach involves adding a version column to the table, which is incremented every time a row is updated. When a transaction updates a row, it checks whether the version has changed since it was read. If the version has changed, the transaction will fail and the changes will not be applied. This approach is useful when conflicts are rare, as it avoids the overhead of locking and blocking transactions:
```plantuml
@startuml
Alice -> DB : select * from entry where id = 1 and version = 1;
Bob -> DB : select * from entry where id = 1 and version = 1;

Alice -> DB : update entry set value = value - 5, version = 2 where id = 1 and version = 1;

note over Alice
  Result:
  | id | value | version |
  |  1 | 5     | 2       |
end note

Bob -> DB : update entry set value = value - 10, version = 2 where id = 1 and version = 1;

note over Bob
  Bob's transaction does not affected anything, since the version has changed:
  | id | value | version |
  |  1 | 5     | 2       |
end note

Bob -> DB : select ROW_COUNT();

note over Bob
  ROW_COUNT() returns 0, since the update did not affect any rows.
end note
```

As an alternative, you can use a `WHERE` clause with the old value, instead of a version column. This is useful when the table does not have a version column:

```plantuml
@startuml
Alice -> DB : select * from entry where id = 1;
Bob -> DB : select * from entry where id = 1;

note over Bob
    Both see the same value:
    | id | value |
    |  1 | 10    |
end note

Alice -> DB : update entry set value = value - 5 where id = 1 and value = 10;\n

note over Alice
    Alice's transaction succeeded:
    | id | value |
    |  1 | 5     |
end note

Bob -> DB : update entry set value = value - 10 where id = 1 and value = 10;\n

note over Bob
    Bob's transaction did not affect any rows, since the value has changed:
    | id | value |
    |  1 | 5     |
end note
 
Bob -> DB : select ROW_COUNT();

note over Bob
  ROW_COUNT() returns 0, since the update did not affect any rows.
end note 
@enduml
```

## Roll your own locking

If locking via the database is not enough for your use case, you can implement your own locking mechanism, e.g. with Redis or Memcached. 

## References

- [A beginner’s guide to database locking and the lost update phenomena](https://vladmihalcea.com/a-beginners-guide-to-database-locking-and-the-lost-update-phenomena/)
- [Isolation Levels in MySQL](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- [The race condition that led to Flexcoin bankruptcy](https://vladmihalcea.com/race-condition/)