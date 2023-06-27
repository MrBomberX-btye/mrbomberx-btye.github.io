---
title:  "Transaction Processing and Concurrency control in T-SQL"
date:  2023-06-18
draft:  false
enableToc: true
enableTocContent: true
description: "I talk about transactions in general and in T-SQL along with concurrency control"
tags:
- db
image: "images/db/transaction.png"
---

{{< featuredImage >}}

# Introduction

Transaction processing is not a new concept, when I was first introduced to it, I didn't get it back then,  because of the lack of practical examples and poorly long articles.

Recently I've taken a course on Datacamp which focuses heavily on that topic only,  using hands-on exercises and real-world scenarios.

So I decided to make a guide for myself and for others who are curious about this huge topic.

I hope you find this topic interesting and learn more about it, check the resources section for more information to dig in.

## Table of Content

1. Humble Intro to Transactions.
   - 1.1 Transaction Nature
   - 1.2 Transaction properties
2. Controlling Transactions
   - 2.1 Rolling back
   - 2.2 Savepoints
   - 2.3 Tracing Nested Transactions with @@TRANCOUNT
3. Handling Errors in Transactions
   - 3.1 spotting errors
   - 3.2 control the flow of transaction

## Transactions in SQL Server

**Transaction**: are _atomic_ unit of work that might include multiple activities that **query** and **modify** data, A one or more statements, all or none of the statements are executed.

Imagine we have a bank account database, we need to transfer $100 from account A to  account B
the procedure as came to your mind is

1. Subtract 100 from A
2. Add those 100 to B

so the operation here needs to be done as all statements, or not

**General Statement**

```sql
BEGIN {TRAN | TRANSACTION }
   [ { transcation_name | @tran_name_variable}
      [ WITH MARK ['description'] ]
  ]
[;]
```

We can optionally add a transaction name and WITH MARK, covering them is out of the scope right now!

```sql
 COMMIT [ {TRAN | TRANSACTION } [transcation_name | transc_name_variable ]]
[ WITH (DELAYED_DURABILITY = {OFF | ON } )][;]
```

> Once the commit is executed, the effect of the transaction can't be reversed!

`ROLLBACK` reverts the transaction to the beginning of it or a `savepoint` inside the transaction

```sql
ROLLBACK {TRAN | TRANSACTION }
[ {transcation_name | @tran_name_variable | savepoint_name | @savepoint_variable } [;]
```

we can define the boundaries (Beginning and end) of the transaction either:

**Explicitly** <br>

The start of a transaction is defined by BEGIN and the end either to be
COMMIT (in case you of success) or ROLLBACK if you need to undo changes

```sql
BEGIN TRAN
         --your query
COMMIT TRAN;
```

**Implicitly** <br>
MS SQL Server automatically commits the transaction at the end of each individual statement, in case you didn't specify this explicitly.

we can change this behavior by changing the session option [IMPLICIT_TRANSACTION] to ON, by doing so, we don't need to specify the beginning of transaction, but we need to specify the end of the transaction either by committing it or rollbacking it.

### Transaction properties

Transactions have four props: commonly known as ACID

- Atomicity
- Consistency
- Isolation
- Durability

#### ACID

In fact, knowing ACID properties is crucial to get a profound understanding of Transactions and their effects on the database state!

The safety guarantees provided by transactions are often described by the well-known
acronym ACID, which stands for Atomicity, Consistency, Isolation, and Durability.
It was coined in 1983 by Theo Härder and Andreas Reuter in an effort to
establish precise terminology for fault-tolerance mechanisms in databases.

##### **Atomicity**

ACID atomicity describes what happens if a client wants to make several
writes, but a fault occurs after some of the writes have been processed—for example,
a process crashes, a network connection is interrupted, a disk becomes full, or some
integrity constraint is violated.

If the writes are grouped together into an atomic
transaction and the transaction cannot be completed (committed) due to a fault, then
the transaction is aborted and the database must discard or undo any writes it has
made so far in that transaction.

Without atomicity, if an error occurs partway through making multiple changes, it’s
difficult to know which changes have taken effect and which haven’t. The application
could try again, but that risks making the same change twice, leading to duplicate or
incorrect data.

Atomicity simplifies this problem: if a transaction was aborted, the
application can be sure that it didn’t change anything, so it can safely be retried.
The ability to abort a transaction on error and have all writes from that transaction
discarded is the defining feature of ACID atomicity.

Perhaps abortability would have
been a better term than atomicity, but we will stick with atomicity since that’s the
usual word.

##### **Consistency**

(invariants) that must always be true—for example, in an accounting system, credits
and debits across all accounts must always be balanced. If a transaction starts with a
database that is valid according to these invariants, and any writes during the transaction
preserve the validity, then you can be sure that the invariants are always satisfied.

However, this idea of consistency depends on the application’s notion of invariants,
and it’s the application’s responsibility to define its transactions correctly so that they
preserve consistency. This is not something that the database can guarantee: if you
write bad data that violates your invariants, the database can’t stop you. (Some specific
kinds of invariants can be checked by the database, for example using foreign
key constraints or uniqueness constraints. However, in general, the application
defines what data is valid or invalid—the database only stores it.)

Atomicity, isolation, and durability are properties of the database, whereas consistency
(in the ACID sense) is a property of the application. The application may rely
on the database’s atomicity and isolation properties in order to achieve consistency,
but it’s not up to the database alone. Thus, the letter C doesn’t really belong in ACID

##### **Isolation**

Isolation in the sense of ACID means that concurrently executing transactions are
isolated from each other: they cannot step on each other’s toes. The classic database
textbooks formalize isolation as serializability, which means that each transaction can
pretend that it is the only transaction running on the entire database. The database
ensures that when the transactions have been committed, the result is the same as if they
had run serially (one after another), even though in reality they may have run concurrently

##### **Durability**

The purpose of a database system is to provide a safe place where data can be stored
without fear of losing it. Durability is the promise that once a transaction has been committed
successfully, any data it has written will not be forgotten, even if there is a
hardware fault or the database crashes.

In a single-node database, durability typically means that the data has been written to
nonvolatile storage such as a hard drive or SSD. It usually also involves a write-ahead
log or similar, which allows recovery in the event that the data structures on disk are corrupted.

In a replicated database, durability
may mean that the data has been successfully copied to some number of nodes. In
order to provide a durability guarantee, a database must wait until these writes or
replications are complete before reporting a transaction as successfully committed.

### Putting it all together

Now let's see an example of a real-world transaction scenario

```sql
BEGIN TRAN;
UPDATE accounts SET current_balance = current_balance - 100 WHERE account_id = 1;
INSERT INTO transactions VALUES (1, -100, GETDATE());
UPDATE accounts SET current_balance = current_balance + 100 WHERE account_id = 5;
INSERT INTO transactions VALUES (5, 100, GETDATE());
COMMIT TRAN;
```

the second example uses rollback, which will revert the operation to the original state

```sql
BEGIN TRAN;
UPDATE accounts SET current_balance = current_balance - 100 WHERE account_id = 1;
INSERT INTO transactions VALUES (1, -100, GETDATE());
UPDATE accounts SET current_balance = current_balance + 100 WHERE account_id = 5;
INSERT INTO transactions VALUES (5, 100, GETDATE());
ROLLBACK TRAN;
```

third example with try... catch
we surrond transcation with try and catch

```sql
BEGIN TRY
   BEGIN TRAN;
      UPDATE accounts SET current_balance = current_balance - 100 WHERE account_id = 1;
      INSERT INTO transactions VALUES (1, -100, GETDATE());
      UPDATE accounts SET current_balance = current_balance + 100 WHERE account_id = 5;
      INSERT INTO transactions VALUES (5, 100, GETDATE());
   COMMIT TRAN;
END TRY
BEGIN CATCH
   ROLLBACK TRAN;
END CATCH
```

a fourth example of implicit transaction, which will cause only a three statements to be executed correctly.

```sql
UPDATE accounts SET current_balance = current_balance - 100 WHERE account_id = 1;
INSERT INTO transactions VALUES (1, -100, GETDATE());
UPDATE accounts SET current_balance = current_balance + 100 WHERE account_id = 5;
INSERT INTO transactions VALUES (500, 100, GETDATE()); -- ERROR!
```

resulting in an **inconsistent** state

### Exercises

```sql

BEGIN TRY  
 Begin TRAN;
  UPDATE accounts SET current_balance = current_balance - 100 WHERE account_id = 1;
  INSERT INTO transactions VALUES (1, -100, GETDATE());
        
  UPDATE accounts SET current_balance = current_balance + 100 WHERE account_id = 5;
  INSERT INTO transactions VALUES (5, 100, GETDATE());
 Commit TRAN;
END TRY
BEGIN CATCH  
 rollback TRAN;
END CATCH
```

```sql
BEGIN TRY  
 -- Begin the transaction
 BEGIN tran;
  UPDATE accounts SET current_balance = current_balance - 100 WHERE account_id = 1;
  INSERT INTO transactions VALUES (1, -100, GETDATE());
        
  UPDATE accounts SET current_balance = current_balance + 100 WHERE account_id = 5;
        -- Correct it
  INSERT INTO transactions VALUES (500, 100, GETDATE());
    -- Commit the transaction
 Commit TRAN;    
END TRY
BEGIN CATCH  
 SELECT 'Rolling back the transaction';
    -- Rollback the transaction
 Rollback tran;
END CATCH
```

using `@@ROWCOUNT` to control when to rollback the transcation

```sql
-- Begin the transaction
begin tran; 
 UPDATE accounts set current_balance = current_balance + 100
  WHERE current_balance < 5000;
 -- Check number of affected rows
 IF @@ROWCOUNT > 200 
  BEGIN 
         -- Rollback the transaction
    Rollback tran; 
   SELECT 'More accounts than expected. Rolling back'; 
  END
 ELSE
  BEGIN 
         -- Commit the transaction
   commit tran
   ; 
   SELECT 'Updates commited'; 
  END
```

### @TRANCOUNT and savepoints

@@TRANCOUNT returns the **number of BEGIN TRAN** statements that are active in your current connection
Returns:

- 0 -> no open transactions
- greater than 0 -> open transaction

It's modified by:

- BEGIN TRAN -> (which increases @@TRANCOUNT by 1) @@TRANCOUNT + 1
- COMMIT TRAN -> @@TRANCOUNT - 1
- ROLLBACK TRAN -> @@TRANCOUNT = 0 (except with savepoint_name)

an example of @@TRANCOUNT in nested transaction

```sql
SELECT @@TRANCOUNT AS '@@TRANCOUNT value'; -- 0
BEGIN TRAN;
    SELECT @@TRANCOUNT AS '@@TRANCOUNT value'; -- 1
    DELETE  transcations;
    BEGIN TRAN;
          SELECT @@TRANSCOUNT AS '@@TRANCOUNT value'; -- 2
          DELTE accounts;
    COMMIT TRAN; -- (2-1 = 1)
    SELECT @@TRANSCOUNT AS '@@TRANSCOUNT value'; -- 1
ROLLBACk TRAN; -- (0)
SELECT @@TRANCOUNT AS '@@TRANCOUNT value'; -- 0
```

| @@TRASCOUNT value |
| -- |
| 0|

@@TRANCOUNT in a TRY..CATCH construct

```sql
BEGIN TRY
   BEGIN TRANS;
      UPDATE account SET current_balance = current_balance - 100 WHERE account_id = 1;
      INSERT INTO transactions VALUES (1, -100, GETDATE())

      UPDATE accounts SET current_balance = current_balance + 100 WHERE account_id = 5;
      INSERT INTO transcations VALUES (5, 100, GETDATE());

   IF (@@TRANCOUNT > 0)
      COMMIT TRAN;
END TRY
BEGIN CATCH
   IF (@@TRANCOUNT > 0)
      ROLLBACK TRAN;
END CATCH
```

Savepoints are:

- Markers within a transcations
- Allow to rollback to the savepoints

```sql
SAVE {TRAN | TRANSACTION} {savepoint_name | @savepoint_variable}
[;]
```

let's see an example

```sql
BEGIN TRAN
   SAVE TRAN savepoint1
   INSERT INTO customers VALUES ('Yousef', 'Meska', 'yousefmeska123@gmail.com', '01211212797')

   SAVE TRAN savepoint2
   INSERT INTO customers VALUES ('Omar', 'Meska', 'omarmeska123@gmail.com', '01011010676')

   ROLLBACK TRAN savepoint1
   ROLLBACK TRAN savepoint2

   SAVE TRAN savepoint3
   INSERT INTO customers VALUES ('ahmed', 'meska', 'ahmedmeska123@gmail.com', '01212')
COMMIT TRAN
```

only the last insert will took place

:notebook: note:
> savepoints don't affect the value of @@TRANSCOUNT

### Examples

```sql
BEGIN TRY
 -- Begin the transaction
 begin tran;
     -- Correct the mistake
  UPDATE accounts SET current_balance = current_balance + 200
   WHERE account_id = 10;
     -- Check if there is a transaction
  IF @@trancount > 0     
      -- Commit the transaction
   commit tran;
     
 SELECT * FROM accounts
     WHERE account_id = 10;      
END TRY
BEGIN CATCH  
    SELECT 'Rolling back the transaction'; 
    -- Check if there is a transaction
    IF @@trancount > 0    
     -- Rollback the transaction
        rollback tran;
END CATCH
```

```sql
BEGIN TRAN;
 -- Mark savepoint1
  SAVE TRAN savepoint1;
 INSERT INTO customers VALUES ('Mark', 'Davis', 'markdavis@mail.com', '555909090');

 -- Mark savepoint2
    SAVE TRAN savepoint2;
 INSERT INTO customers VALUES ('Zack', 'Roberts', 'zackroberts@mail.com', '555919191');

 -- Rollback savepoint2
 Rollback tran savepoint2    -- Rollback savepoint1
rollback tran savepoint2
 -- Mark savepoint3
 save tran savepoint3
 INSERT INTO customers VALUES ('Jeremy', 'Johnsson', 'jeremyjohnsson@mail.com', '555929292');
-- Commit the transaction
COMMIT TRAN;
```

## Controlling Errors of Transcations (XACT_ABORT & XACT_STATE)

`XACT_ABORT` specified where the current transction will be automatically rolled back when an error occrus

It can be set to on or off.

```sql
SET XACT_ABORT { ON | OFF}
```

If an error occurs under the default setting which is by default `OFF`, the transcation can automatically be rolled back or not, depending on the error, if the transcations is not rolled back, **it remains open**

Setting it to ON will ensure the transaction will be rolled back when an error occurs and abort the transaction

```sql
SET XACT_ABORT ON
```

Let's see an examples

```sql
SET XACT_ABORT OFF
BEGIN TRAN
   INSERT INTO customers VALUES ('yousef', 'meska', 'yousefmeska123@gmail.com', '1545')
   INSERT INTO customers VALUES ('omar', 'meska', 'omarmeska.com', '1546') -- ERROR!
COMMIT TRAN
```

The last statement generates an error of violating the unique key 'unique_email'

```
Violation of UNIQUE KEY 'unique_email'
```

If we checked the customer's table we'll see the first statement has been executed despite an error found on the transaction

| customer_id | first_name | last_name | email | phone |
| --- | ---- | --- | --- | --- |
| 14 | yousef | meska | yousefmeska123.com | 1545 |

Now If we turned XACT_ABORT to ON, the transaction will be rolled back and aborted

| customer_id | first_name | last_name | email | phone |
| --- | ---- | --- | --- | --- |

### XACT_ABORT with RAISEERROR and THROW statement

```sql
SET XACT_ABORT ON

BEGIN TRAN
 INSERT INTO Users ('yousef', 'meska', 'yousefmeska123@gmail.com', '011000000')
    RAISERROR('An Error occured', 16 ,1);
    INSERT INTO Users ('omar', 'meska', 'omarmeska123@gmail.com', '012000000')
 COMMIT TRAN
 
SELECT * FROM Users WHERE first_name IN ('yousef', 'omar')

-- What's the output ?


SET XACT_ABORT ON

BEGIN TRAN
 INSERT INTO Users ('yousef', 'meska', 'yousefmeska123@gmail.com', '011000000')
 THROW 55000, 'An Error occured', 1;
 INSERT INTO Users ('omar', 'meska', 'omarmeska123@gmail.com', '012000000')
COMMIT TRAN
 
SELECT * FROM Users WHERE first_name IN ('yousef', 'omar')


-- What's the output? and why?

# Answer

1. Setting XACT_ABORT to ON, will rollback the transaction if an error occured and abort the execution
However, using `RAISERROR()` will not affect `XACT_ABORT`.
So the output will be an error which will be shown to the user and the transcation will take place.

2. So Microsoft recommend using `THROW` instead because it will affect XAACT_ABORT and the transaction will be rolled back in addition to the error message that will be shown to the user
```

### XACT_STATE

```sql
XACT_STATE()
```

It doesn't take any parameters.
It returns

- `0` -> no open transaction
- `1` -> Open and committable transaction
- `-1` -> Open and uncommitable transactions (Doomed transaction)

When a transaction is **Doomed** that's means

- You can't commit
- You can't rollback to a savepoint
- You can only rollback the full transaction
- You can't make any changes but you can read data

Let's see an examples
In this example, the transaction will becommitted if there's no error between the TRY block, if there's an error, the catch will handle it by determining the state of the transaction
And the state of the transaction will remain open and committable because we set `XACT_ABORT` to OFF
if the transaction is committable then the transaction will be committed
if not it will be rolled backs

```sql
SET XACT_ABORT OFF -- if there's an error the transaction will be open if not rolled back
BEGIN TRY
   BEGIN TRAN
      INSERT INTO customers VALUES ('x', 'y', 'x@gmail.com')
      INSERT INTO customers VALUES ('z', 'u', 'z@gmail.com') -- ERROR!
   COMMIT TRAN
END TRY
BEGIN CATCH
   IF XACT_STATE() = -1
      ROLLBACK
   IF XACT_STATE = 1
      COMMIT
   SELECT ERROR_MESSAGE() As error_messag
END CATCH
```

Only the first statement will be committed

| customer_id | first_name | last_name | email |
| --- | ---- | --- | --- |
| 15 | x | y | <x@gmail.com> |

Let's see what happens when we need to make the transaction uncommittable.

```sql
SET XACT_ABORT ON -- the transaction will remain open but uncommittable
BEGIN TRY
   BEGIN TRAN
      INSERT INTO customers VALUES ('x', 'y', 'x@gmail.com')
      INSERT INTO customers VALUES ('z', 'u', 'z@gmail.com') -- ERROR!
   COMMIT TRAN
END TRY
BEGIN CATCH
   IF XACT_STATE() = -1
      ROLLBACK
   IF XACT_STATE = 1
      COMMIT
   SELECT ERROR_MESSAGE() As error_messag
END CATCH
```

| customer_id | first_name | last_name | email |
| --- | ---- | --- | --- |

The transaction has been rolled back

### Exercises

```sql
-- Use the appropriate setting
SET XACT_ABORT off;
-- Begin the transaction
Begin transaction; 
 UPDATE accounts set current_balance = current_balance - current_balance * 0.01 / 100
  WHERE current_balance > 5000000;
 IF @@ROWCOUNT <= 10 
     -- Throw the error
  Throw 55000, 'Not enough wealthy customers!', 1;
 ELSE  
     -- Commit the transaction
  Commit transaction; 
```

```sql
-- Use the appropriate setting
SET XACT_ABORT on;
BEGIN TRY
 BEGIN TRAN;
  INSERT INTO customers VALUES ('Mark', 'Davis', 'markdavis@mail.com', '555909090');
  INSERT INTO customers VALUES ('Dylan', 'Smith', 'dylansmith@mail.com', '555888999');
 COMMIT TRAN;
END TRY
BEGIN CATCH
 -- Check if there is an open transaction
 IF XACT_STATE() <> 0
     -- Rollback the transaction
  Rollback tran;
    -- Select the message of the error
    SELECT error_message() AS Error_message;
END CATCH
```

## Resources

- [ ] Datacamp, Transaction and Error Handling in SQL server interactive course
- [ ] Ch. 7 Transaction processing, Designing Data-Intensive Applications
- [ ] Microsoft T-SQL Documentation
