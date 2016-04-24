# ElephantStack
Column store for Postgres via arrays

## Description
Postgres does well at many "big data" challenges and can easily handle 10TB+ databases, but it does poorly when the number of individual rows gets very large (typically 100M+). Much of the reason for this is that Postgres has a minimum 23 byte overhead per row. In practical use, the per-row overhead is usually closer to 32 bytes or more (including surrogate key and index).

In contrast, Postgres arrays are very space efficient. There is no per-value overhead (except as required for type alignment), and ~16 bytes for the array itself. Storing 500 ints would take just over 2KB, whereas a table of 500 ints would require ~12KB additional row overhead (ignoring page overhead).

## The Problem
Creating a table that uses arrays instead of storing individual rows is easy, but interacting with it is difficult. Consider this example SQL:

```
UPDATE tablename SET value1 = 1 WHERE value2 = 42;
```

A column-store table for that might be

```
CREATE TABLE tablename_column_store(
  value1    int[]     NOT NULL
  , value2  int[]     NOT NULL
  ... other fields ...
);
```

To execute that update, you need to find every row where the value2 array contains 42 (ie: WHERE value2 @> array[42]). For each row that matches, you need to find the positions of every occurrance of 42 in the values2 array (perhaps using array_positions()), then set the value1 array to 1 at each of those positions.

Handling a query like

```
SELECT ... FROM tablename WHERE value1 BETWEEN 20 AND 30;
```

is even more difficult. You could recode the BETWEEN to something using ANY or ALL, but that's difficult and error-prone. It also doesn't index well.

## Goals
The goal for ElephantStack is to make these use cases easy to handle.

* Provide scalable _"column-store like"_ storage using arrays
** Support in-heap storage for data that is still mutating
** Support forced external storage for data that is static
** Support primary keys outside of partitioning
* Migrate data between storage mechanisms as necessary
* Support interfacing to a base table for highly volatile data
* Provide row-based access via writable views
