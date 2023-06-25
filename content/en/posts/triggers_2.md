---
title:  "Trigger Management in T-SQL - Part 2"
date:  2023-06-18
draft:  false
enableToc: true
enableTocContent: true
description: "I talk about AI and NLP in the context of my graduation project."
tags:
- misc
image: "images/db/trigger.jpg"
---

# Intro

In this part I will be talking about:

1. Trigger limitations
2. Trigger use cases

## Trigger limitation and use cases

Let's begin by discussing some of the advantages of using Triggers:

- Used for Database Integrity purposes
- We can enforce business rules and store them directly in the databases, this makes it easier to change and update the applications that are using the database, because the business logic is encapsulated inside the database itself
- Triggers give us control over which statements are allowed in a database
- Implementation of complex business logic triggered by a single event
- Auditing the database  and user activities

now let's see what the **Disadvantages** are:

- Triggers are not easy to manage in a centralized manner, because they are difficult to be detected or viewed
- they are invisible to a client application or when debugging code
- because of their complex code, we can't sometime trace their logic when troubleshooting
- They can affect the server and make it slower by overusing them

### Finding the server-level trigger

From all those pros and cons, we can conclude that we should document our triggers and make them as simpler as we can.
Because Triggers can be implemented on many levels (system, database, tables, etc) SQL Server gave us a way to view that information about triggers in one place

- [1] Getting system-level triggers

```sql
SELECT * FROM sys.server_triggers;
```

- [2] Getting the database and table triggers

```sql
SELECT * FROM sys.triggers
```

### Trigger type and definition

**The type of the trigger (database or table) can be determined from the 'parent_class_desc' column
** We can view triggers definition graphically using MS management studio:
head over to the Triggers folder and right-click on the trigger name and choose 'Script Trigger as -> CREATE TO -> New Query Edit Window'

** SQL system views are like virtual tables in the database, helping to reach the information that cannot be reached otherwise

```sql
SELECT definition
FROM sys.sql_modules
WHERE object_id = OBJECT_ID('PreventOrdersUpdate')
```

** We can also use the OBJECT_DEFINITION() function and pass it the id of the trigger

```sql
SELECT OBJECT_DEFINITION(OBJECT_ID('PreventOrdersUpdate')
```

** the last option we can use to use the 'sp_helptext' procedure, which uses a parameter called 'objname'

```sql
EXECUTE sp_helptext @objname = 'PreventOrdersUpdate'
```

but this option is not the most common one, it's rarely used

## Triggers best practice

- Make sure your database is well-documented

- keep your trigger logic simple
- avoid overusing triggers

### Time to practice

**Creating a report of existing triggers on the database**

```sql
-- Gather information about database triggers
SELECT name AS TriggerName,
    parent_class_desc AS TriggerType,
    create_date AS CreateDate,
    modify_date AS LastModifiedDate,
    is_disabled AS Disabled,
    is_instead_of_trigger AS InsteadOfTrigger
FROM sys.triggers
UNION ALL
SELECT name AS TriggerName,
    -- Get the column that contains the trigger type
    parent_class_desc AS TriggerType,
    create_date AS CreateDate,
    modify_date AS LastModifiedDate,
    is_disabled AS Disabled,
    0 AS InsteadOfTrigger
-- Gather information about server triggers
FROM sys.server_triggers
-- Order the results by the trigger name
ORDER BY TriggerName;
```

using OBJECT_DEFINITION

```sql
-- Gather information about database triggers
SELECT name AS TriggerName,
    parent_class_desc AS TriggerType,
    create_date AS CreateDate,
    modify_date AS LastModifiedDate,
    is_disabled AS Disabled,
    is_instead_of_trigger AS InsteadOfTrigger,
       -- Get the trigger definition by using a function
    OBJECT_DEFINITION (object_id)
FROM sys.triggers
UNION ALL
-- Gather information about server triggers
SELECT name AS TriggerName,
    parent_class_desc AS TriggerType,
    create_date AS CreateDate,
    modify_date AS LastModifiedDate,
    is_disabled AS Disabled,
    0 AS InsteadOfTrigger,
       -- Get the trigger definition by using a function
    OBJECT_DEFINITION (object_id)
FROM sys.server_triggers
ORDER BY TriggerName;
```

### Use cases for AFTER triggers (DML)

A common use for AFTER triggers is to store historical data in other tables (Having a history of changes performed on a table).

**Best practice**:

Keep an overview of the changes for the most important tables in your database
For example, let's say the customer on the Customers table changes his phone number, so we keep this change as well as the old phone number on the 'CustomersHistory' table

The previous table is obtained using an **AFTER** Trigger

```sql
CRETE TRIGGER CopyCusomtersToHistory
ON Customers
AFTER INSERT, UPDATE
AS
           INSERT INTO CustomersHistroy(Customer, ContractId, Address, PhoneNo)
          SELECT Customer, ContractID, Address, PhoneNo, GETDATE()
          FROM inserted;
```

### Table auditing using triggers

- Another major use of AFTER triggers is to audit changes occurring in the database

**Auditing** means: Tracking any changes that occur within the defined scope
usually, the scope of the audit is comprised of very important tables from the database
In the following query, we create a trigger called 'OrderAudit' that keeps track of any changes that occur to the 'Orders Table' it will fire for any DML statements,
inside the trigger we've two Boolean variables that will check the special tables "inserted" and "deleted" when one of the special tables contains data, the associated variables will be set to "true"
the combination of variables will tell us if the operation is an INSERT, UPDATE, or DELETE
these changes will be kept inside 'TablesAudit' Table

```sql
CREATE TRIGGER OrdersAudit
ON Orders
AFTER UPDATE, INSERT, DELETE
AS
        DECLARE @Insert BIT = 0, @Delete BIT = 0;
        IF EXISTS (SELECT *FROM inserted) SET @Insert = 1;
IF EXISTS (SELECT* FROM deleted( SET @Deleted = 1;

        INSERT INTO [TablesAudit] ([TableName], [EventType], [UserAccount], [EventDate])
        SELECT 'Orders' As [TableName]
        , CASE WHEN @Insert = 1 AND @Delete = 0 THEN 'INSERT'
                    WHEN @Insert = 1 AND @Delete = 1 THEN 'UPDATE'
                    WHEN @Insert = 0 AND @Delete = 1 THEN 'DELETE'
                    END AS [Event]
       , ORIGINAL_LOGIN()
       , GETDATE();

```

- Another use case is 'Notifying users':
which means we can send notifications to different users using triggers
most of the notifications will be about events happening in the database
In this query, the Sales department must be notified when new orders are placed
The stored procedure will be executed when an INSERT query happens

```sql
CREATE TRIGGER NewOrderNotification
ON Orders
AFTER INSERT
AS
                 EXECUTE SendNotifications @RecipientEmail = 'sales@freshfruit.com', 
                                  @EmailSubject = 'New Order Placed'
                                  @EmailBody = 'A new Order was just placed'
```

### time to practice

#### Copy Customer changes to the History table

```sql
CREATE TRIGGER CopyCustomersToHistory
ON Customers
AFTER INSERT, UPDATE
AS
             INSERT INTO CustomersHistory (CustomerID, Customer, ContractID, Address, PhoneNo, Email, ChangeDate)
            SELECT CustomerID, Customer, ContractID, ContractDate, Address, PhoneNo, Email, GETDATE()
          FROM inserted
```

#### Keep track of any modifications made to the contents of Orders

```sql
-- Add a trigger that tracks table changes
create trigger OrdersAudit
ON Orders
after INSERT, delete, update
AS
 DECLARE @Insert BIT = 0;
 DECLARE @Delete BIT = 0;
 IF EXISTS (SELECT * FROM inserted) SET @Insert = 1;
 IF EXISTS (SELECT * FROM deleted) SET @Delete = 1;
 INSERT INTO [TablesAudit] (TableName, EventType, UserAccount, EventDate)
 SELECT 'Orders' AS TableName
        ,CASE WHEN @Insert = 1 AND @Delete = 0 THEN 'INSERT'
     WHEN @Insert = 1 AND @Delete = 1 THEN 'UPDATE'
     WHEN @Insert = 0 AND @Delete = 1 THEN 'DELETE'
     END AS Event
     ,ORIGINAL_LOGIN() AS UserAccount
     ,GETDATE() AS EventDate;
```

## Uses cases for INSTEAD OF Trigger (DML)

- Preventing certain operations from happening
- Enforcing data integrity
- Control the database statements

### An example of a trigger that prevents and notifies admin

Let's say we don't want the regular users to make certain operations on the database tables [like updating or deleting] and when they make so, we send the admin a notification

```sql
CREATE TRIGGER PreventCustomerRemoval
ON Customers
INSTEAD OF DELETE
AS 
        
      DECLARE @EmailBodyText NAVCHAR(50) = (SELECT 'User "' + ORIGINAL_LOGIN() + ' " tried to remove a customer from the database. ');

            RAISEERROR('Customer entries are no subject to removal.', 16, 1);
             EXECUTE SendNotification @RecipentEmail = 'admin@freshfruit.com'
                                                          , @EmailSubject = 'Suspicous database behaviour
                                              , @EmailBody = @EmailBodyText;
```

### Triggers with conditional logic

Triggers are not just limited to the prevention of operations, we can also use it to decide whether or not some of the operations should succeed

![](https://i.ibb.co/C5s7Xnb/1.png)

here we prevent inserting any new orders when there is not sufficient stock of the product

```sql
CREATE TRIGGER ConfirmStock
ON Orders
INSTEAD OF INSERT
AS
IF EXISTS(SELECT * FROM Product AS p 
INNER JOIN inserted AS i ON i.Product = p.Product
WHERE p.Quantity < i.Quantity)
RAISERROR('You cannot place orders when there is no product stock', 16, 1);
ELSE 
INSERT INTO dbo.Orders(Customer, Product, Quantity, OrderDate, TotalAmount)
SELECT Customer, Product, Quantity, OrderDate, TotalAmount FROM Inserted;

```

### time for practice

```sql
-- Create a new trigger to confirm stock before ordering
CREATE TRIGGER ConfirmStock
ON Orders
INSTEAD OF INSERT
AS
 IF EXISTS (SELECT *
      FROM Products AS p
      INNER JOIN inserted AS i ON i.Product = p.Product
      WHERE p.Quantity < i.Quantity)
 BEGIN
  RAISERROR("You cannot place orders when there is no stock for the order's product.", 16, 1);
 END
 ELSE
 BEGIN
  INSERT INTO Orders (OrderID, Customer, Product, Price, Currency, Quantity, WithDiscount, Discount, OrderDate, TotalAmount, Dispatched)
  SELECT OrderID, Customer, Product, Price, Currency, Quantity, WithDiscount, Discount, OrderDate, TotalAmount, Dispatched FROM inserted;
 END;
```

### Use cases for DDL Triggers

As you know DDL Triggers can be created at different levels (Database level, Server level)
[1] Database Level
A trigger created at the database level can respond to statements related to tables, view interactions, and index management, as well as more specific statements to do with permissions management or statistics

- `CREATE_TABLE, ALTER_TABLE, DROP_TABLE`
- `CREATE_VIEW, ALTER_VIEW, DROP_VIEW`
- `CREATE_INDEX, ALTER_INDEX, DROP_INDEx`

[2] Server Level

at the server level, the trigger used can respond to database management and controlling server permissions and the use of credentials

- `CREATE_DATEBASE, ALTER_DATABASe, DROP_DATEBASE`
- `GRANT_SERVER, DENY_SERVER, REVOKE_SERVER`
- `CREATE_CREDENTIAL, ALTER_CREDENTIAL, DROP_CREDENTIAL`

check the online documentation for the full list of DDL

### Databasse auditing

we can keep track of the changes at the database level

```SQL
CREATE TRIGGER DatabaseAudit
ON DATABASE
FOR DDL_TABLE_VIEW_EVENTS
AS
INSERT INTO [DatabaseAudit] ([EventType], [Database], [Object], [UserAccount]. [Query], [EventTime])
SELECT 
EVENTDATA().value('/EVENT_INSTANCE/EventType)[1]', 'NAVCHAR(50)
)

```
