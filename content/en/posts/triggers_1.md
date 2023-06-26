---
title:  "Trigger in T-SQL - Part 1: Introduction"
date:  2023-06-18
draft:  false
enableToc: true
enableTocContent: true
description: "In this part I talk about DML and DDL Triggers in T-SQL."
tags:
- misc
image: "images/db/trigger.jpg"
---


## Introduction

### Triggers

A special type of **Stored Procedure** that is automatically executed when **events** like (data modifications) occur on the database server

#### Types of Triggers

In SQL Server there are 3 main types of triggers:

- Data Manipulation Language (DML) triggers: When a user or process modifies data through an INSERT, UPDATE, or DELETE
These triggers are associated with statements related to tables or views

- Data Definition Language: fire in response to statements executed at the database or server level, like `CREATE`, `ALTER` or `DROP`

- Logon triggers: fire in response to LOGON events when a user's session is established.

Another way to classify triggers is to classify them based on their behavior.

#### Types of Trigger (based on behavior)

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1638357237644/Fzg1b3tn3.png)

A trigger can behave differently in relation to the statement that fires it, resulting in two types of triggers

- `AFTER` Trigger:
when you want to execute a piece of code after the initial statement that fires the trigger. <br>
**An Example**: <br>
a simple use case of this type of trigger is to rebuild an index after a large insert of data into a table. <br>
**Another Example** <br>
is using a trigger to send alerts when UPDATE statements are run againse the database. (Notify the admin when data is updated)

- `INSTEAD OF` Trigger:

will not perfome the inital opertation, but will execute custom code instead, so a replacement statement is executed instead of the original one
some examples (use cases):

- prevent insertions: prevent inserting data into tables,
- prevent updated or deletion.
- Prevent object modification.

and you can also notify the database administrarotr of suspicous behaviour while also preventing any changes

### Trigger Definition

because trigger is a sql server object, we add a new one by using the **CREATE** statement.

General Syntax

```sql
CREATE TRIGGER <TriggerName>
ON <Target>

<Trigger_type> <event>
AS
{your_trigger_action}

```

You should give the trigger a descriptive name
For example an trigger created with `AFTER`

```sql
-- Create a trigger by giving it a descriptie name
CREATE TRIGGER ProductTrigger
--The Trigger neeeds to be attachred to a table
ON Products
-- The Trigger Behvaiour type
AFTER INSERT
--The Beginning of the trigger workflow
AS
-- The action executed by the trigger
PRINT('An insert of data was made in the products table')
```

(with INSTEAD OF)

```sql
CREATE TRIGGER PreventDeleteFromOrders
ON Orders
INSTEAD OF DELETE

AS
PRINT('You are not allowed to delete rows from the Orders table.')
```

any attempt to remove rows from the table will fail due the use of `INSTEAD OF`

**Practical Example** <br>

```sql
-- Create a new trigger that fires when deleting data
CREATE TRIGGER PreventDiscountsDelete
ON Discounts
-- The trigger should fire instead of DELETE
INSTEAD OF DELETE
AS
 PRINT 'You are not allowed to delete data from the Discounts table.';
```

#### Practice time

The Fresh Fruit Delivery company needs help creating a new trigger called OrdersUpdatedRows on the Orders table.

This trigger will be responsible for filling in a historical table (OrdersUpdate) where information about the updated rows is kept.

**A historical table** is often used in practice to store information that has been altered in the original table. In this example, changes to orders will be saved into OrdersUpdate to be used by the company for auditing purposes.

```sql
-- Set up a new trigger
CREATE TRIGGER OrdersUpdatedRows
ON Orders
-- The trigger should fire after UPDATE statements
AFTER UPDATE
-- Add the AS keyword before the trigger body
AS
 -- Insert details about the changes to a dedicated table
 INSERT INTO OrdersUpdate(OrderID, OrderDate, ModifyDate)
 SELECT OrderID, OrderDate, GETDATE()
 FROM inserted;
```

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1638357352226/T33L9IXv2.png)

### How DML Triggers are used

- Initiating actions when manipulating data:
the main reason for using DML triggers is to initiate actions when manipulating data
- preventing data manipulation:  and sometimes the manipulation of data needs to be prevented, and this can also be done with the use of trigger

- Tracking data or database object changes: another powerful use case often seen in practice is using the triggers for tracking data change and even database object changes

- User auditing and database security: database admins also use triggers to track user actions and unwanted changes  and to secure the database by protecting it from

🤔 **How to decide between AFTER and INSTEAD OF?** <br>
Each one of them has certain use cases, so depending the on use case your choice will be fair. <br>
**`AFTER`**:
one good example of using AFTER trigger is for a large insert of data into a sales table,
once the data gets inserted, a **cleansing** step should run to remove or repaint any unwanted information
when the cleansing step finished, a report with the results will be generated, this report should be anaylzed by a database adminstrator.
so the trigger will then send an email to  the responsible people.

```sql
CREATE TRIGGER SalesNewInfoTrigger
ON sales
AFTER INSERT
AS
EXEC sp_cleansing @Table = 'Sales'
EXEC sp_generateSalesReport;
EXEC sp_sendnotification;
```

**INSTEAD OF**
a good use case:

Imagine you have a table containing light bulb information and stock details for a sales platform

| Brand | Model | Power| Stock|
| --- | --- | --- | --- |
| x | Standford | 30W | 30 |
| y | Buma  | 40W | 0 |
| z | Ultra | 50W | 0 |

the **Power** column values have changed for some models, and an update is initiated to change the information in the table. <br>
but there are still some bulb models in stock that have the old power value, however, and they shouldn't be updated. <br>
so `Instead of` trigger can help you deal with this more complex situation
the correct approach is to update the characteristics only for the models that don't have the old versions in stock. <br>
the new models need to be in the table too but as new rows instead of updated ones

```sql
CREATE TRIGGER BulbStockTrigger
ON Bulbs
INSTEAD OF INSERT
AS
IF EXISTS (SELECT * FROM Bulbs As b
INNER JOIN inserted AS i
                     ON b.Brand = i.Brand
                     AND b.Model = i.Model
Where b.Stock = 0)
BEGIN
   UPDATE b
   SET b.Power = i.Power;
          b.Stock = i.Stock;
    FROM Bulbs As b
   INNER JOIN  inserted AS i
                         ON a.Brand = i.Brand;
                                And b.Model = i.Model
    WHERE b.Stock = 0
END
ELSE
       INSERT INTO Bulbs
       SELECT * FROM inserted;
```

- Power changes from some models
- Update only the products with not stock.
- Add new rows for the products with stock

the new modes need to be in the table too, but as new rows instead of updated ones.

| Brand | Model | Power| Stock|
| --- | --- | --- | --- |
| x | Standford | 30W | 30 |
| y | Buma  | 40W | 100 |
| z | Ultra | 50W | 100 |
| x | Standford | 35w | 100 |

**Practical Example**

```sql
CREATE TRIGGER ProductsNewItems
ON Products 
AFTER INSERT
AS 
 INSERT INTO ProductsHistory(Product, Price, Currency, FirstAdded)
 SELECT Product, Price, Currency, GETDATE()
 FROM inserted;
```

#### Trigger Alternative

Trigger are great, but they are not the only solution in SQL server
So we will discuss some alternatives

**Triggers vs Stored Procedures**
Triggers as you know are special kind of Stored Procedure, but what makes them "Special"?
Triggers are fired automatically by an event

```sql
-- Will fire an Insert trigger
INSERT INTO Orders [...];
```

In opposite direction, SP run only when called explicitly

```sql
-- will run the sp
EXECUTE sp_DailyMaintaince
```

Triggers:

- don't allow parameters or transactions.
- can't return values as output;
SP :
Accept input parameters and transcations
can return values as output
so these differences enforce some use cases for each one of them

Triggers used for:

- auditing
- Integrity enforcement

SP used for:

- general tasks
- user-specific needs

the second comparison is with Computed Columns
**Triggers v Computed columns** <br>
**Computed Columns**: are a good way to automate the calculation of the values contained by some columns.
Triggers:

- use columns **from other tables** for calculations
Computed Columns:
- use columns only from the same table for calculations

while this calculation will be done with INSERT or UPDATE statements when using a trigger,

```SQL
--used in the trigger body
[...]
UPDATE
SET TotalAmount = Price * Quantity
[...]
```

for a computed column it will be part of the table definition

```SQL
-- Column Definition
[...]
TotalAmount As Price * Quantity
[...]
```

Example of Computed Column
As you see in the definintion of SalesWithPrice, the 'TotalAmount' table value comes from the mutliplication of the 'Quantity' and 'price' column from the **same table**

```sql
CREATE TABLE [SalesWithPrice]
(
  [OrderID] INT INDENTITY(1,1),
  [Customer] NVARCHAR(50),
  [Product] NVARCHAR(50),
  [Price] DECIMAL(10, 2),
  [Currency] NVARCHAR(3),
  [Quantity] INT,
  [OrderDate] DATE DEFAULT (GETDATE()),
  [TotalAmount AS [Quantity] * [Price]
)
```

But if those two columns are on other tables, we can't use Computed Columns, we use trigger instead
the Price column is not part of the 'SalesWitoutPrice' table

```sql
CREATE TRIGGER [SalesCalculateTotalAmount]
On [SalesWithoutPrice]
AFTER INSERT
AS
        UPDATE [sp]
        SET [sp].[TotalAmount] = [sp].[Quantity] * [p].[Price]
        FROM [SalesWithoutPrice] AS [sp]
        INNER JOIN [Products] AS [p] ON [sp].Product = [p].[Product]
        WHERE [sp].[TotalAmount] IS NULL;
```

## Classifications of Triggers in Depth (DML)

After trigger can be used for both DML and DDL statment
we'll be focusing on DML AFTER triggers

The trigger will be fired ""AFTER" the event finish executing not at the beginning of the event

AFTER Trigger prerequisites:

- Table or view needed for DML statements
- The Trigger will be attached to the same table
so we need:

1. Target
2. Description of the trigger: what you are trying to achieve with the trigger

let's take an example: <br>
In this example we want to keep some details of products that are not sold anymore, these products will be removed from the Product table, but their details will be kept in a 'RetiredProduct' table for financial accounting reasons.

So the trigger will save information about deleted rows (from the product table) to the 'RetiredProduct' table.

the trigger should have a uniquely identified a name for our example it will be 'TrackRetiredProducts'
whenever rows are deleted form the 'Products' table
the deleted rows' information will be saved to the 'RetiredProducts'

```sql
CREATE TRIGGER TrackRetiredProducts
ON Products
AFTER DELETE
AS
    INSERT INTO RetiredProducts(Product, Measure)
    SELECT Product, Measure
FROM deleted;
```

notice that we are not getting the information from the 'Products' table, but from a table called **deleted**.

### Inserted and deleted tables

These tables are automatically created by SQL server and you can make use of them in your trigger actions.
Depending on the operation you are performing, they will hold different inormation

1. **Inserted table**

this table will store the values of the new rows for "INSERT" and "UPDATE" statements

| Special table      | INSERT | UPDATE | DELETE |
| ----------- | ----------- | ----------- | ----------- |
| **inserted**      | new rows       | new rows | N/A
| **deleted** | N/A       | updated rows | removed rows |

2. **deleted table**

will store the value of modified rows for the update statement, or the value of removed rows for the 'DELETE' statement

##### Practice time

```SQL
-- Create the trigger
CREATE trigger TrackRetiredProducts
ON Product
AFTER DELETE
AS
 INSERT INTO RetiredProducts (Product, Measure)
 SELECT Product, Measure
 FROM deleted;
```

To ensure

```SQL
-- Remove the products that will be retired
DELETE FROM products
WHERE Product IN ('Cloudberry', 'Guava', 'Nance', 'Yuzu');

-- Verify the output of the history table
SELECT * FROM RetiredProducts;
```

Practicing with AFTER triggers

You have been given the task to create new triggers on some tables, with the following requirements:

[1] Keep track of canceled orders (rows deleted from the Orders table). Their details will be kept in the table CanceledOrders upon removal.

[2] Keep track of discount changes in the table Discounts. Both the old and the new values will be copied to the DiscountsHistory table.

[3] Send an email to the Sales team via the SendEmailtoSales stored procedure when a new order is placed.

```sql
CREATE TRIGGER KeepCanceledOrders
ON Orders
AFTER DELETE
AS
 INSERT INTO CanceledOrders
SELECT * FROM deleted;



-- Create a new trigger to keep track of discounts
CREATE Trigger CustomerDiscountHistory
ON Discounts
AFTER UPDATE
AS
 -- Store old and new values into the `DiscountsHistory` table
 INSERT INTO DiscountsHistory (Customer, OldDiscount, NewDiscount, ChangeDate)
 SELECT i.Customer, d.Discount, i.Discount, GETDATE()
 FROM inserted AS i
 INNER JOIN deleted AS d ON i.Customer = d.Customer;

 -- Notify the Sales team of new orders
CREATE TRIGGER NewOrderAlert
ON Orders
AFTER insert
AS
 EXECUTE SendEmailtoSales;
```

### INSTEAD OF Trigger

In contrast with AFTER triggers, INSERT OF triggers can only be used for DML statements (not DDL)

- the actions are performed instead of the DML event
- The DML event does not run anymore

Let's take an example <br>
Our example is a table of Orders which holds the details of the orders placed by the Fresh Fruit Delivery Company's customer

as we discussed before, you should keep in mind some steps to know exactly what your trigger will be doing.

1. Target Table -> Orders
2. Description of the trigger -> the trigger we will create should prevent updates to the existing entries in this table, this will ensure that placed orders cannot be modified, this is a rule enforced by the company, which means our trigger will fire as a response to UPDATE statements
3. Trigger firing event (DML) -> UPDATE
4. Trigger Name -> PreventOrdersUpdate (having an information name is important when creating triggers).

let's create the triggers

```sql
CREATE TRIGGER PreventOrdersUpdate
ON Orders
INSTEAD OF UPDATE
AS 
    RAISERROR('Updates on 'Orders' table are not permitted. Place a new order to add new products', 16, 1);

Exercise Time:
```sql
CREATE Trigger PreventOrdersUpdate
ON Orders
INSTEAD OF UPDATE
AS
 RAISERROR ('Updates on "Orders" table are not permitted.
                Place a new order to add new products.', 16, 1);
```

Attempting to UPDATE will throw an error

```sql
UPDATE Orders SET Quantity = 700
where OrderID = 425;
```

## DDL Triggers

- Only used with AFTER

- and attached to only database or server level
- no special tables

### AFTER and FOR

the FOR and AFTER have the same results

```sql
CREATE TRIGGER Database ChangeLog
FOR CREATE_TABLE
[...]
```

Developers tend to use AFTER for DML triggers and FOR keyword for DDL triggers

### DDL Trigger preprequisites

[1] Target Object (server or database) =>DATABASe
[2] Description of the trigger => Log table with definition changes
[3] Trigger firing events (DDL) => CREATE_TABLE, ALTER_TABLE, DROP_TABLE
[4] Trigger Name => TrackTableChanges

```sql
CREATE TRIGGER TrackTableChanges
ON DATABASe
FOR CREATE_TABLE,ALTER_TABLE, DROP_TABLE
AS
      INSERT INTO TablesChangesLog(EventData, ChangedBy)
      VALUES(EVENTDATA(), USER)
```

EVENTDATA() function:
holds information about the event that runs and fires the trigger

### Preventing the triggering event for DML triggers

we've discussed before that we can't use INSTEAD OF with DDL Triggers!!
Does this mean we don't have a solution around this if we want to do similar thing with DDL triggers?
The Answer is No, we have.

we can define a trigger to roll back the statements that fired it

```SQL
CREATE TRIGGER PreventTableDeletion
ON DATABASE
FOR DROP_TABLE
AS 
       RAISERROR('You are not allowed to remove tables from this database.', 16, 1);
      ROLLBACK;
```

#### Practice time

```sql
-- Create the trigger to log table info
CREATE TRIGGER TrackTableChanges
ON DATABASE
FOR CREATE_TABLE,
 ALTER_TABLE,
 DROP_TABLE
AS
 INSERT INTO TablesChangeLog (EventData, ChangedBy)
    VALUES (EVENTDATA(), USER);
```

### LOGON Triggers

LOGON Triggers are fired when a user logs on and creates a connection to a SQL server.

the trigger is fired after the authentication phase (meaning after the username and password are checked), **BUT** before the user session is established (when the information from SQL Server becomes available for queries)

#### Logon trigger prerequisites

logon trigger can only be attached at the server level, and the firing event can only be LOGON

1. Trigger firing event -> LOGON
2. Description of the trigger -> Audit successful / failed logons to the server
3. Trigger name  -> LogonAudit
the trigger will be executed nder the same credentials (username and password) as the firing event.

```sql
CREATE TRIGGER LogonAudit
ON ALL SERVER WITH EXECUTE AS 'sa'
FOR LOGON
AS 
      INSERT INTO ServerLogonLog(LoginName, LoginDate, SessionID, SourceIPAddress)
SELECT ORIGNAL_LOGIN(), GETDATE(), @@SPID, client_net_address
FROM SYS.DM_EXEC_CONNECTIONS WHERE session_id = @@SPID;
```

'sa' is a built-in adminstrator account tht has the full permissions on the server, because regular users don't have sensitive information like logon details

@@SIPD => the id of the current user

some exercises

```sql
-- Save user details in the audit table
INSERT INTO ServerLogonLog (LoginName, LoginDate, SessionID, SourceIPAddress)
SELECT ORIGINAL_LOGIN(), GETDATE(), @@SPID, client_net_address
-- The user details can be found in SYS.DM_EXEC_CONNECTIONS
FROM SYS.DM_EXEC_CONNECTIONS WHERE session_id = @@SPID;

-- Create a trigger firing when users log on to the server
CREATE TRIGGER LogonAudit
-- Use ALL SERVER to create a server-level trigger
ON ALL SERVER WITH EXECUTE AS 'sa'
-- The trigger should fire after a logon
for logon
AS
 -- Save user details in the audit table
 INSERT INTO ServerLogonLog (LoginName, LoginDate, SessionID, SourceIPAddress)
 SELECT ORIGINAL_LOGIN(), GETDATE(), @@SPID, client_net_address
 FROM SYS.DM_EXEC_CONNECTIONS WHERE session_id = @@SPID;
```

## Resources

- <a href="https://app.datacamp.com/learn/courses/transactions-and-error-handling-in-sql-server">Datacamp: Building and Optimizing Triggers</a>
