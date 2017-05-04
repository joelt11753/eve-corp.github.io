---
layout: post
title: SQL Operations - Flagging Records as Not Found
keywords: sql, etl, notfound, not found, exclude, merge, merging data, identifying missing records, missing
---

I came across an interesting scenario for one of my clients.  They have a system which scans database hardware, and then merges the metadata into another database for reporting.

When metadata is pulled, it goes into a harvesting database along with metadata such as which job it was part of, when it was ran, which servers were harvested, etc.  After the data is harvested, another process is responsible for updating a target database, which is used by alternate applications for reporting "current status".

For simplicity, we'll keep the operations limited to two tables in a single database: a `Scan` table, and a `Target` table.

`ObjectId` is shared identifier across both tables.

### Some Starting Historial Data

Assume that a previous harvest->merge cycle has already put this data into our tables.

Scan Data

ScanId | ObjectId | Name | Type
--- | --- | --- | ---
1 | 1 | `Persons` | Table
1 | 2 | `Name` | Column
1 | 3 | `Zip` | Column

Target Data

ObjectId | Name | Type | NotFound
--- | --- | --- | ---
1 | `Persons` | Table | NULL
2 | `Name` | Column | NULL
3 | `Zip` | Column | NULL

### Results from a new Scan after the column `Zip` was removed

Now assume that between the last run, and this run, somebody removed the `Zip` column.  Here's the state of our `Scan` table and the `Target` table is unchanged.  *Our task is to ensure that `Zip` is marked as `NotFound`*

Scan Data

ScanId | ObjectId | Name | Type
---  |--- | --- | ---
1 | 1 | `Persons` | Table
1 | 2 | `Name` | Column
1 | 3 | `Zip` | Column
2 | 1 | `Persons` | Table
2 | 2 | `Name` | Column

Take note that even though it wasn't in this scan, `Zip` is still in the table, with ObjectId = 1.

### How to identify `NotFound` Objects

Here's the data if you want to follow along

``` SQL
-- TABLES

DECLARE @TargetObjects TABLE (
    ObjectId INT NOT NULL,
    NAME VARCHAR(20) NOT NULL,
    TYPE VARCHAR(20) NOT NULL,
    NotFound BIT NULL
)

DECLARE @ScanObjects TABLE (
    ScanId INT NOT NULL,
    ObjectId INT NOT NULL,
    NAME VARCHAR(20) NOT NULL,
    TYPE VARCHAR(20) NOT NULL
)

-- DATA

INSERT INTO @TargetObjects
        ( ObjectId, Name, Type )
VALUES  ( 1, 'Persons', 'Table')
        , ( 2, 'Name', 'Column')
        , ( 3, 'Zip', 'Column')


INSERT INTO @ScanObjects
        ( ScanId, ObjectId, Name, Type )
VALUES  ( 1, 1, 'Persons', 'Table')
        , ( 1, 2, 'Name', 'Column')
        , ( 1, 3, 'Zip', 'Column')
        , ( 2, 1, 'Persons', 'Table')
        , ( 2, 2, 'Name', 'Column')

DECLARE @ScanId INT = 2
```

#### The Wrong Way

``` SQL
-- The Complete Wrong Way
SELECT 
    'TargetObjectId' = TargetObject.ObjectId
    , 'ScanObjectId' = ScanObject.ObjectId
    , 'ScanScanId' = ScanObject.ScanId
    , 'Name' = TargetObject.Name
    , 'Type' = TargetObject.Type
FROM @TargetObjects AS TargetObject
LEFT JOIN @ScanObjects AS ScanObject
    ON TargetObject.ObjectId = ScanObject.ObjectId
WHERE
    ScanObject.ObjectId IS NULL
    AND ScanObject.ScanId = @ScanId
```

The final result will give an **empty dataset**, which is not what we want.


TargetObjectId | ScanObjectId | ScanScanId | Name | Type
--- | ---| --- | --- | ---


#### Why the Wrong Way doesn't work

Let's look at the data without the `WHERE` clause to see what's going on.

``` SQL
-- No Filter
SELECT 
    'TargetObjectId' = TargetObject.ObjectId
    , 'ScanObjectId' = ScanObject.ObjectId
    , 'ScanScanId' = ScanObject.ScanId
    , 'Name' = TargetObject.Name
    , 'Type' = TargetObject.Type
FROM @TargetObjects AS TargetObject
LEFT JOIN @ScanObjects AS ScanObject
    ON TargetObject.ObjectId = ScanObject.ObjectId
```

We get this:

TargetObjectId | ScanObjectId | ScanScanId | Name | Type
--- | ---| --- | --- | ---
1 | 1 | 1 | `Persons` | Table
2 | 2 | 1 | `Name` | Column
3 | 3 | 1 | `Zip` | Column
1 | 1 | 2 | `Persons` | Table
2 | 2 | 2 | `Name` | Column


Obviously there are no `NULL`, so `ScanObject.ObjectId IS NULL` will always return empty. Even we filter by `ScanId`, you can see how, unsurprisingly, still nothing is `NULL`, and our `Zip` column does not appear at all.


``` SQL
-- Filtering by ScanId in WHERE
SELECT 
    'TargetObjectId' = TargetObject.ObjectId
    , 'ScanObjectId' = ScanObject.ObjectId
    , 'ScanScanId' = ScanObject.ScanId
    , 'Name' = TargetObject.Name
    , 'Type' = TargetObject.Type
FROM @TargetObjects AS TargetObject
LEFT JOIN @ScanObjects AS ScanObject
    ON TargetObject.ObjectId = ScanObject.ObjectId
WHERE
    ScanObject.ScanId = @ScanId
```


TargetObjectId | ScanObjectId | ScanScanId | Name | Type
--- | ---| --- | --- | ---
1 | 1 | 2 | `Persons` | Table
2 | 2 | 2 | `Name` | Column


#### Right Way

*(Scroll down a little for the final answer.)*

The way to get what we want is to filter inside the `JOIN ON` clause, and not in the `WHERE`.

Here's what we want to on the opposite side of our "LEFT JOIN".

``` SQL
-- The other side of our "LEFT JOIN"
SELECT * 
    FROM @ScanObjects AS ScanObject
    WHERE ScanObject.ScanId = @ScanId
```

ScanId | ObjectId | Name | Type
--- | --- | --- | ---
2 | 1 | `Persons` | Table
2 | 2 | `Name` | Column

Notice how `ObjectId = 3` and `ScanId = 2` is missing.  This is what we want, because when we do our `LEFT JOIN`, the record from our Target will exist, and the Scan will not.

And here's the Final Answer:

``` SQL
-- The Complete Right Way
SELECT 
    'TargetObjectId' = TargetObject.ObjectId
    , 'ScanObjectId' = ScanObject.ObjectId
    , 'ScanScanId' = ScanObject.ScanId
    , 'Name' = TargetObject.Name
    , 'Type' = TargetObject.Type
FROM @TargetObjects AS TargetObject
LEFT JOIN @ScanObjects AS ScanObject
    ON TargetObject.ObjectId = ScanObject.ObjectId
        -- Do not move this filter into the WHERE clause.
        AND ScanObject.ScanId = @ScanId
WHERE
    ScanObject.ObjectId IS NULL
```

TargetObjectId | ScanObjectId | ScanScanId | Name | Type
--- | ---| --- | --- | ---
3 | `NULL` | `NULL` | `Zip` | Column

Hurray!

Then, it's a simple matter of marking the Target as "Not Found"

``` SQL
-- Flag missing records as NotFound
UPDATE  TargetObject
    SET NotFound = 1
FROM @TargetObjects AS TargetObject
LEFT JOIN @ScanObjects AS ScanObject
    ON TargetObject.ObjectId = ScanObject.ObjectId
        -- Do not move this filter into the WHERE clause.
        AND ScanObject.ScanId = @ScanId
WHERE
    ScanObject.ObjectId IS NULL
```

Checking our answer...

``` SQL
SELECT * FROM @TargetObjects
```

Looks good!

ObjectId | NAME | TYPE | NotFound
--- | --- | --- | ---
1 | `Persons` | Table | NULL
2 | `Name` | Column | NULL
3 | `Zip` | Column | **1**


*In a follow-up post, I may explore this concept using `MERGE`*
