---
title: "Query Optimization: Tuning Indexes for better performance."
date: 2023-12-31
draft: false
enableToc: true
enableTocContent: true
description: Digging into Database indexing and how to tune them for better performance.
tags:
  - db
image: images/db/indexing.png
---

{{< featuredImage >}}

## Intro

Indexes (also called “keys” in MySQL) are data structures that storage engines use to find rows quickly. They also have several other beneficial properties that we’ll explore in this chapter.

Thus, they are not standardized: indexing works slightly differently in each engine, and not all engines support all types of indexes. Even when multiple engines support the same index type, they might implement it differently under the hood

In MySQL, a storage engine uses indexes in as in book's index you used to search for pages containing specific terms. It searches the index’s data structure for a value. When it finds a match, it can find the row that contains the match. Suppose you run the following query

```sql
SELECT first_name FROM sakila.actor WHERE actor_id = 5;
```

> [!info]
> An index contains values from one or more columns in a table. If you index more than one column, the column order is very important, because MySQL can only search efficiently on a leftmost prefix of the index

Creating an index on two columns is not the same as creating two separate single-column indexes, as you’ll see.

## Type of indexes

There are many types of indexes, each designed to perform well for different purposes.

We're going to cover:

- B-Tree Index
- Hash Index
- Bitmap Index

> [!info]
> Indexes are implemented in the storage engine layer, not the server layer.

MySQL used B-tree index by default on `CREATE TABLE` and other statements. However, storage engines might use different storage structures internally.

Storage engines use B-Tree indexes in various ways, which can affect performance. For instance, `MyISAM` uses a prefix compression technique that makes indexes smaller, but `InnoDB` leaves values uncompressed in its indexes. Also, `MyISAM` indexes refer to the indexed rows by their physical storage locations, but `InnoDB` refers to them by their primary key values which is the cluster key. [more on that later]. Each variation has benefits and drawbacks.

the general idea of of B-tree is that all the values are stored in order, and each leaf page is the same distance from the root.

This figure shows an abstraction of a B-tree index used by InnoDB storage engine
![](https://i.imgur.com/4ULPTTm.png)

Leaf pages are special, because they have pointers to the indexed data instead of pointers to other pages. (Different storage engines have different types of “pointers” to the data.)

Because B-tree store the indexes columns in order, they're useful for searchhing for ranges of data.

Let's take an example of a `composite index` [more on that later]

```sql
CREATE TABLE People ( 
 last_name varchar(50) not null,
 first_name varchar(50) not null, 
 dob date not null, 
 gender enum('m', 'f')not null, 
 key(last_name, first_name, dob) 
);
```

The index here will contain the values from the last_name, first_name, and dob columns for every row in the table
Here's how the index arranges the data it stores
![](https://i.imgur.com/JblrQr0.png)

Notice that the index sorts the values according to the order of the columns given in the index in the CREATE TABLE statement

> It's way more important to know when to use B-tree and when not to!

There're some types of queries that can use a B-tree index, which are the ones that can be used for **lookups** by

- full key value
- key range
- key prefix

> [!danger]
> They're useful only if the lookup uses a leftmost prefix of the index.

Taking the previous example in considerations, let's look when the index will be used and when it not would be!

**Match the full value key**
Specifying values for all columns in the index.

```sql
SELECT first_name FROM People WHERE last_name = 'Allen' AND first_name = 'Cuba' AND dob = '1960-01-01'
```

**Match a leftmost prefix**
Find all people with the last name Allen, this will use only the first column in the index

```sql
SELECT first_name FROM People WHERE last_name = 'Allen'
```

**Match a column prefix**
Matching on the first part of a column's value, for example, finding all people whose last names begin with J

```sql
SELECT first_name FROM People WHERE last_name LIKE 'J%'
```

**Match a range of values**
You an find people whose last names are between Allen and Meska

```sql
SELECT first_name FROM People WHERE last_name BETWEEN 'Allen' AND 'Meska';
```

**Match one part exactly and match a range on another part**
This index can help you find everyone whose last name is Allen and whose first name starts with the letter K (Kim, Karl, etc.). This is an exact match on last_ name and a range query on first_name

```sql
SELECT first_name FROM People WHERE last_name = 'Allen' AND first_name LIKE 'K%'
```

**Match index-only queries**
B-Tree indexes can normally support index-only queries, which are queries that access only the index, not the row storage. We discuss this optimization later in “Covering Indexes"

> [!info]
> Another important benefit from using B-tree is finding values in sorted order as well lookups, Because B-tree's nodes are sorted.

So B-tree index would be helpful for `ORDER BY`  and `GROUP BY` clauses

### Drawbacks and limitations of using B-tree index

They're not useful if the lookup does not **<u>start</u>**<u></u> from the leftmost side of the indexed colum

```sql
SELECT first_name FROM People WHERE first_name = 'Yousef';
```

The ubove quey will not use the index.
Also you can't find people whose last name **ends** with a particular letter, for examaple the below query will not use the index.

```sql
SELECT first_name FROM People WHERE last_name LIKE '%J';

```

The storage engine can’t optimize accesses with any columns to the right of the first range condition. For example, if your query is

```sql
WHERE last_name="Smith" AND first_name LIKE 'J%' AND dob='1976-12-23'
```

the index access will use only the first two columns in the index, because the LIKE is a range condition (the server can use the rest of the columns for other purposes, though). For a column that has a limited number of values, you can often work around this by specifying equality conditions instead of range conditions. We show detailed examples of this in the indexing case study later in this chapter.

You can't skip columns either, MySQL will choose only the first column from the index which is `last_name`

```sql
WHERE last_name = 'Smith' AND dob = '1976-12-23'
```

> [!info]
> MySQL stops at the first range!

Some of these limitations are not inherent to B-Tree indexes, but are a result of how the MySQL query optimizer and storage engines use indexes. Some of them might be removed in the future

Now you know why we said the column order is extremely important: these limitations are all related to column ordering. For optimal performance, you might need to create indexes with the same columns in different orders to satisfy your queries.

if the information you're searching for in the index itself, it's called "inline query", like
EXPLAIN ANALYZE SELECT id from employees where id = 1; # (Id where is primary key (btree))
It will show you that Heap Fetches are 0
opposing to this query SELECT name from employees where id = 2 (name has no index on it )
So, First thing will happen, the engine will go to the page using the id (on the index) which containthe information we need about name (which is stored on disk [another read]) If we do the query above again, it will result in much smaller time, because caching
Let's look at another query

```sql
SELECT id from employee where name = 'Zsh';
```

this will result in Sequential scan which cost alot of time. [Fulltable scan].
Although mySql does something intelligent somehow by applying worker threads so it can dsequential scan in parallel
If we now make an index on name

```sql
CREATE INDEX employee_idx on employees(name)
```

$ Bitmap index created
Now let's seaerch

```sql
SELECT id,name FROM employees Where name = 'Yousef'; (index will be used)
SELECT id,name FROM employees WHERE name like '%You%' (index will not be used) # 
```

becausactually this expression is not a single value, we've alot of possabilites.

## Benefits of Indexes

## Indexing Strategies For High Performance

There are many ways to choose and use indexes effectively, because there are many special-case optimizations and specialized behaviors. Determining what to use when and evaluating the performance implications of your choices are skills you’ll learn over time

In this section we're going to cover the following:

- Isolating the Column
- Prefix indexes and Index Selectivity
- MultiColumns Index
- Choosing Good Columns Order

### Isolating the columns

We commonly see queries that defeat indexes or prevent MySQL from using the available indexes. MySQL generally can’t use indexes on columns unless the columns are isolated in the query. “Isolating” the column means it should not be part of an expression or be inside a function in the query. For example, here’s a query that can’t use the index on actor_id:

```sql
SELECT actor_id FROM sakila.actor WHERE actor_id + 1 = 5
```

A human can easily see that the WHERE clause is equivalent to actor_id = 4, but MySQL can’t solve the equation for actor_id. It’s up to you to do this. You should get in the habit of simplifying your WHERE criteria, so the indexed column is alone on one side of the comparison operator.

### Prefix Indexes and Index Selectivity

Sometimes you need to index very long character columns, which makes your indexes large and slow. One strategy is to simulate a hash index, as we showed earlier in this chapter. But sometimes that isn’t good enough.

What can you do? You can often save space and get good performance by indexing the first few characters instead of the whole value. This makes your indexes use less space, but it also makes them less selective. Index selectivity is the ratio of the number of distinct indexed values (the cardinality) to the total number of rows in the table (#T), and ranges from 1/#T to 1. A highly selective index is good because it lets MySQL filter out more rows when it looks for matches. A unique index has a selectivity of 1, which is as good as it gets.

A prefix of the column is often selective enough to give good performance. If you’re indexing BLOB or TEXT columns, or very long VARCHAR columns, you must define prefix indexes, because MySQL disallows indexing their full length. The trick is to choose a prefix that’s long enough to give good selectivity, but short enough to save space. The prefix should be long enough to make the index nearly as useful as it would be if you’d indexed the whole column. In other words, you’d like the prefix’s cardinality to be close to the full column’s cardinality.

To determine a good prefix length, find the most frequent values and compare that list to a list of the most frequent prefixes.

Let's assume we've example datasets of cities, let's find the most frequently occurring  cities

```sql
SELECT COUNT(*) AS cnt, city
FROM sakila.city_demo GROUP BY ORDER BY cnt DESC LIMIT 10
```

| cnt | city           |
| --- | -------------- |
| 65  | London         |
| 49  | Hiroshima      |
| 48  | Teboksary      |
| 48  | Pak Kret       |
| 48  | Yaound         |
| 47  | Tel Aviv-Jaffa |
| 47  | Shimoga        |
| 45  | Cabuyao        |
| 45  | Callao         |
| 45  | Bislig         |

Notice that there are roughly 45 to 65 occurrences of each value. Now we find the most frequently occurring city name prefixes, beginning with three-letter prefixes:

```sql
SELECT COUNT(*) AS cnt, LEFT(city, 3) AS pref -> FROM sakila.city_demo GROUP BY pref ORDER BY cnt DESC LIMIT 10;
```

| cnt | pref |
| --- | ---- |
| 483 | San  |
| 195 | Cha  |
| 177 | Tan  |
| 167 | Sou  |
| 163 | al-  |
| 163 | Sal  |
| 146 | Shi  |
| 136 | Hal  |
| 130 | Val  |
| 129 | Bat  |

There are many more occurrences of each prefix, so there are many fewer unique prefixes than unique full-length city names. The idea is to increase the prefix length until the prefix becomes nearly as selective as the full length of the column. A little experimentation shows that 7 is a good value

| cnt | pref    |
| --- | ------- |
| 70  | Santiag |
| 68  | San Fel |
| 65  | London  |
| 61  | Valle d |
| 49  | Hiroshi |
| 48  | Teboksa |
| 48  | Pak Kre |
| 48  | Yaound  |
| 47  | Tel Avi |
| 47  | Shimoga |

Another way to calculate a good prefix length is by computing the full column’s selectivity and trying to make the prefix’s selectivity close to that value. Here’s how to find the full column’s selectivity

By calculating the total distinct values over to the number of values in the table

```sql
SELECT COUNT(DISTINCT city)/COUNT(*) FROM sakila.city_demo;
```

The prefix will be about as good, on average (there’s a caveat here, though), if we target a selectivity near .031. It’s possible to evaluate many different lengths in one query, which is useful on very large tables. Here’s how to find the selectivity of several prefix lengths in one query:

```sql
SELECT COUNT(DISTINCT LEFT(city, 3))/COUNT(*) AS sel3, 
COUNT(DISTINCT LEFT(city, 4))/COUNT(*) AS sel4, 
COUNT(DISTINCT LEFT(city, 5))/COUNT(*) AS sel5,
COUNT(DISTINCT LEFT(city, 6))/COUNT(*) AS sel6,  
COUNT(DISTINCT LEFT(city, 7))/COUNT(*) AS sel7
FROM sakila.city_demo;
```

This query shows that increasing the prefix length results in successively smaller improvements as it approaches seven characters. It’s not a good idea to look only at average selectivity. The caveat is that the worstcase selectivity matters, too. The average selectivity might make you think a four- or five-character prefix is good enough, but if your data is very uneven, that could be a trap. If you look at the number of occurrences of the most common city name prefixes using a value of 4, you’ll see the unevenness clearly:

```sql
SELECT COUNT(*) AS cnt, LEFT(city, 4) AS pref 
FROM sakila.city_demo GROUP BY pref ORDER BY cnt DESC LIMIT 5;
```

| cnt | pref |
| --- | ---- |
| 205 | San  |
| 200 | Sant |
| 135 | Sout |
| 104 | Chan |
| 91  | Toul |

With four characters, the most frequent prefixes occur quite a bit more often than the most frequent full-length values. That is, the selectivity on those values is lower than the average selectivity. If you have a more realistic dataset than this randomly generated sample, you’re likely to see this effect even more. For example, building a four-character prefix index on real-world city names will give terrible selectivity on cities that begin with “San” and “New,” of which there are many

Now that we’ve found a good value for our sample data, here’s how to create a prefix index on the column:

```sql
ALTER TABLE sakila.city_demo ADD KEY (city(7))
```

Prefix indexes can be a great way to make indexes smaller and faster, but they have downsides too: MySQL cannot use prefix indexes for ORDER BY or GROUP BY queries, nor can it use them as covering indexes. A common case we’ve found to benefit from prefix indexes is when long hexadecimal identifiers are used. We discussed more efficient techniques of storing such identifiers in the previous chapter, but what if you’re using a packaged solution that you can’t modify? We see this frequently with vBulletin and other applications that use MySQL to store website sessions, keyed on long hex strings. Adding an index on the first eight characters or so often boosts performance significantly, in a way that’s completely transparent to the application.

### Multi-Column Indexs

Multicolumn indexes are often very poorly understood. Common mistakes are to index many or all of the columns separately, or to index columns in the wrong order. We’ll discuss column order in the next section. The first mistake, indexing many columns separately, has a distinctive signature in `SHOW CREATE TABLE`

```sql
CREATE TABLE t ( c1 INT, c2 INT, c3 INT, KEY(c1), KEY(c2), KEY(c3) )
```

> [!info]
> This strategy of indexing often results when people give vague but authoritativesounding advice such as “create indexes on columns that appear in the WHERE clause.” This advice is very wrong. It will result in one-star indexes at best. These indexes can be many orders of magnitude slower than truly optimal indexes. Sometimes when you can’t design a three-star index, it’s much better to ignore the WHERE clause and pay attention to optimal row order or create a covering index instead

Individual indexes on lots of columns won’t help MySQL improve performance for most queries. MySQL 5.0 and newer can cope a little with such poorly indexed tables by using a strategy known as **index merge**, which permits a query to make limited use of multiple indexes from a single table to locate desired rows. Earlier versions of MySQL could use only a single index, so when no single index was good enough to help, MySQL often chose a table scan

For example, the film_actor table has an index on film_id and an index on actor_id, but neither is a good choice for both WHERE conditions in this query:

```sql
SELECT film_id, actor_id FROM sakila.film_actor
WHERE actor_id = 1 OR film_id = 1;
```

In older MySQL versions, that query would produce a table scan unless you wrote it as the UNION of two queries:

```sql
SELECT film_id, actor_id FROM sakila.film_actor WHERE actor_id = 1 UNION ALL 
SELECT film_id, actor_id FROM sakila.film_actor WHERE film_id = 1 AND actor_id <> 1;
```

In MySQL 5.0 and newer, however, the query can use both indexes, scanning them simultaneously and merging the results

There are three variations on the algorithm: union for OR conditions, intersection for AND conditions, and unions of intersections for combinations of the two. The following query uses a union of two index scans, as you can see by examining the Extra column:

```sql
EXPLAIN SELECT film_id, actor_id FROM sakila.film_actor WHERE actor_id = 1 OR film_id = 1
```

```
*************************** 1. row *************************** id: 1 select_type: SIMPLE table: film_actor type: index_merge possible_keys: PRIMARY,idx_fk_film_id key: PRIMARY,idx_fk_film_id key_len: 2,2 ref: NULL rows: 29 Extra: **Using union(PRIMARY,idx_fk_film_id); Using where**
```

Something to keep in mind that index merge is somehow a costly operation after all.
So It might be an indication of a poorly indexed table if you see "index merge" in the `EXPLAIN`:

- When the server intersects indexes (usually for AND conditions), it usually means that you need a single index with all the relevant columns, not multiple indexes that have to be combined.
- When the server unions indexes (usually for OR conditions), sometimes the algorithm’s buffering, sorting, and merging operations use lots of CPU and memory resources. This is especially true if not all of the indexes are very selective, so the scans return lots of rows to the merge operation.
When you see an index merge in EXPLAIN, you should examine the query and table structure to see if this is really the best you can get. You can disable index merges with the optimizer_switch option or variable. You can also use IGNORE INDEX.

### Choosing a Good Columns Order

There is an old rule of thumb for choosing column order: place the most selective columns first in the index. How useful is this suggestion? It can be helpful in some cases, but it’s usually much less important than avoiding random I/O and sorting, all things considered. (Specific cases vary, so there’s no one-size-fits-all rule. That alone should tell you that this rule of thumb is probably less important than you think.)

> Placing the most selective columns first can be a good idea when there is no sorting or grouping to consider, and thus the purpose of the index is only to optimize WHERE lookups. In such cases, it might indeed work well to design the index so that it filters out rows as quickly as possible, so it’s more selective for queries that specify only a prefix of the index in the WHERE clause. However, this depends not only on the selectivity (overall cardinality) of the columns, but also on the actual values you use to look up rows—the distribution of values. This is the same type of consideration we explored for choosing a good prefix length. You might actually need to choose the column order such that it’s as selective as possible for the queries that you’ll run most

let's take an example query and diagnose it better

```sql
SELECT * FROM payment WHERE staff_id = 2 AND customer_id = 584;
```

should we create an index on (staff_id, customer_id) or should we reverse the column order ?

> We can run some quick queries to help examine the **distribution of values** in the table to determine which columns has a higher selectivity.

Let's count the cardinality of each predicate in the `WHERE` clause

```sql
SELECT SUM(staff_id = 2), SUM(customer_id = 584) FROM payment;
```

```
*************************** 1. row *************************** SUM(staff_id = 2): 7992 SUM(customer_id = 584): 30
```

According to the rule of thumb, we should place customer_id first in the index, because the predicate matches fewer rows in the table. We can then run the query again to see how selective staff_id is within the range of rows selected by this specific customer ID

```sql
SELECT SUM(staff_id = 2) FROM payment WHERE customer_id = 584;
```

```
SUM(staff_id = 2): 17
```

But if you don’t have specific samples to run, it might be better to use the old rule of thumb, which is to look at the cardinality across the board, not just for one query:

```sql
SELECT COUNT(DISTINCT staff_id)/COUNT(*) AS staff_id_selectivity, 
COUNT(DISTINCT customer_id)/COUNT(*) AS customer_id_selectivity,
COUNT(*) 
FROM payment\G
```

```
staff_id_selectivity: 0.0001 customer_id_selectivity: 0.0373 COUNT(*): 16049
```

customer_id has higher selectivity, so again the answer is to put that column first in the index:

```sql
ALTER TABLE payment ADD KEY(customer_id, staff_id);
```

In the end, although the rule of thumb about selectivity and cardinality is interesting to explore, other factors—such as sorting, grouping, and the presence of range conditions in the query’s WHERE clause—can make a much bigger difference to query performance

## Clustered Index vs Non-Clustered Index

![](https://i.imgur.com/hsFXyrH.png)

![](https://i.imgur.com/NjLLYou.png)

### Clustered Index

Clustered indexes aren’t a separate type of index. Rather, they’re an approach to data storage. The exact details vary between implementations, but InnoDB’s clustered in-dexes actually store a B-Tree index and the rows together in the same structure.

When a table has a clustered index, its rows are actually stored in the index’s leaf pages.

ou can have only one clustered index per table, because you can’tstore the rows in two places at once

Because storage engines are responsible for implementing indexes, not all storage en-gines support clustered indexes, But our focus here on `InnoDB`
![](https://i.imgur.com/8X2TrZD.png)

Notice how records are laid out in a clustered index. Notice that the leafpages contain full rows but the node pages contain only the indexed columns in the figure above.
Clustered Index (only 1 on the table, mostly the primary key)

Some database servers let you choose which index to cluster, but none of MySQL’s built-in storage engines does at the time of this writing. InnoDB clusters the data by the primary key.

If you don’t define a primary key, InnoDB will try to use a unique nonnullable indexinstead. If there’s no such index, InnoDB will define a hidden primary key for you andthen cluster on that.

Clustering can help performance but it can also has negative and serious problems.

What are the advatages of Clustering data ?

- You can keep related data close together. For example, when implementing amailbox, you can cluster by user_id, so you can retrieve all of a single user’s mes-sages by fetching only a few pages from disk. If you didn’t use clustering, eachmessage might require its own disk I/O.
- Data access is fast. A clustered index holds both the index and the data togetherin one B-Tree, so retrieving rows from a clustered index is normally faster than acomparable lookup in a nonclustered index.
- Queries that use covering indexes can use the primary key values contained at theleaf node.

What are the disadvantages ?

- clustering gives the largest improvement for I/O-bound workloads. If the data fitsin memory the order in which it’s accessed doesn’t really matter, so clusteringdoesn’t give much benefit
- Insert speeds depend heavily on insertion order. Inserting rows in primary keyorder is the fastest way to load data into an InnoDB table. It might be a good idea to reorganize the table with OPTIMIZE TABLE after loading a lot of data if you didn’tload the rows in primary key order.
- pdating the clustered index columns is expensive, because it forces InnoDB tomove each updated row to a new location.
- ables built upon clustered indexes are subject to page splits when new rows areinserted, or when a row’s primary key is updated such that the row must be moved.A page split happens when a row’s key value dictates that the row must be placedinto a page that is full of data. The storage engine must split the page into two toaccommodate the row. Page splits can cause a table to use more space on disk.
- lustered tables can be slower for full table scans, especially if rows are less denselypacked or stored nonsequentially because of page splits
- econdary index accesses require two index lookups instead of one
The last point can be a bit confusing. Why would a secondary index require two indexlookups? The answer lies in the nature of the “row pointers” the secondary index stores.Remember, a leaf node doesn’t store a pointer to the referenced row’s physical location;rather, it stores the row’s primary key values

That means that to find a row from a secondary index, the storage engine first finds theleaf node in the secondary index and then uses the primary key values stored there tonavigate the primary key and find the row. That’s double work: two B-Tree navigationsinstead of one.

### Comparison of InnoDB and MyISAM data layout

The differences between clustered and nonclustered data layouts, and the correspond-ing differences between primary and secondary indexes, can be confusing and surpris-ing. Let’s see how InnoDB and MyISAM lay out the following table

```sql
CREATE TABLE layout_test ( col1 int NOT NULL, col2 int NOT NULL, PRIMARY KEY(col1), KEY(col2));
```

suppose we've populated the table with pk from 1 to 10,000 inserted in random order and then optimized with `OPTIMIZE TABLE` In other words, the data is arrangedoptimally on disk, but the rows might be in a random order. The values for col2 arerandomly assigned between 1 and 100, so there are lots of duplicates.

> Nonclustered index designs aren’t always able to provide single-operation row lookups, by the way. Whena row changes it might not fit in its original location anymore, so you might end up with fragmented rowsor “forwarding addresses” in the table, both of which would result in more work to find the row.

#### MyISAM's data layout

MyI-SAM stores the rows on disk in the order in which they were inserted
![](https://i.imgur.com/tyOooS9.png)

Because the rows are fixed-size, MyISAM can find any row by seeking the required number of bytes from the beginning of the table. (MyISAM doesn’t always use “row numbers,” as we’ve shown it uses different strategies depending on whether the rows are fixed-size or variable-size.)

Each leaf node in the index can simply contain the row number.

#### InnoDB's data layout

#### Inserting rows in primary key order with InnoDB

![](https://i.imgur.com/69yQyJ1.png)

```SQL
/** Clustered index **/
ALTER TABLE dbo.MyTable
 ADD CONSTRAINT PK_MyTable PRIMARY KEY(Id);

/**Nonclustered index**/
ALTER TABLE dbo.MyTable
ADD CONSTRAINT UC_MyTable_Id UNIQUE(Id);
```

---

![](https://i.imgur.com/fmPsCIk.png)

![](https://i.imgur.com/bP6t5C5.png)

Non-clustered index
suppose we've created a non-clustered index on the name column
![](https://i.imgur.com/JpDfCFc.png)

![](https://i.imgur.com/jRIGhug.png)

Row locators contain:

- Cluster key (employee_id)
- actual row [name]

Both clustered index and non-clustered index are working together to find the data!

What are the steps used to find the actual row ?
![](https://i.imgur.com/ZacibzY.png)

1. SQL servers used non-clustered index on the name column to quicly find the employee entry on the index
2. Clustered index (employee_id) is used to find the actual row!

> In sql server we read the execution plan from top to bottom and from right to left!

![](https://i.imgur.com/gnjyi2X.png)

When a query navigate through the clustered index tree to the base table data, this is called **Clustered index seek**
The clustered index contains the base table data itself, this is why you can create one clustere index

![](https://i.imgur.com/mYNFF3C.png)

**non-clustered index** is seperate from the base data, the base data could exist instead as clustered index
the data directly available may be limited because usually non-clustered indexes only include a subset of columns from the table

if values from columns not in the index are requested, the query may navigate back to the base data using the references mentioned earlier
If the query optimizer decides that doing that is too expensive, it may fall back on scaning the base data instead of using the index

Non-clustered index begin seperate from the base data brings about a couple of important features

**Filtered indexes** only contains rows that meet a user-defind predicate, and to create these you use `WHERE` clause on the index definition, so clustered index can't be used because it has to contain all the data on the table.

```sql
CREATE NONCLUSTERED INDEX IX_PhoneBook_NCI 
ON dbo.PhoneBook(LastName, FirstName)
WHERE (LastName >= 'Yousef');
```

Summary:
A clusterd index is a way of representing data as a whole
A non-clustered index is a physically seperate structure that refernces the base data and can different sort order
