# Mastering T-SQL Querying Fundamentals - SQLBits
### Itzik Ben-Gan

http://tsql.solidq.com/SourceCodes/Mastering%20T-SQL%20Querying%20Fundamentals.txt

# Processing Order

1. FROM
2. WHERE
3. GROUP BY
4. HAVING
5. SELECT

    a. \<select list>

    b. DISTINCT
6. ORDER BY
7. OFFSET / FETCH / TOP

## 1. FROM
### JOIN
3 steps:
1. Cartesian product ( a cross join)
2. Apply ON clause
3. Add outer rows

### Others
There are also APPLY, PIVOT and UNPIVOT

## 2. WHERE
Only rows for which the WHERE predicate is TRUE are returned (not UNKNOWNs)
Column aliases cannot be used here as we've not evaluated select yet.
Aggregate functions cannot be used here as GROUP BY has not been evaluated yet.

## 3. GROUP BY
A group is created for each unique value in the GROUP BY list.

## 4. HAVING
HAVING predicate is evaluated for each group. Only groups for which the predicate is TRUE are returned.

## 5. SELECT
There are two algorithms that process groups, it can use stringaggregate or hashaggregate. If the optimiser chooses one or the other it can affect the order results are returned in.
Results are sets so no order is guaranteed! If we need order we need to specify an ORDER BY!

## 6. ORDER BY
CAN refer to column aliases in SELECT.
The results of an ORDER BY query is called a *result cursor* as it's order, so not a set! So we can't use the output of an ordered select as another set in a query, as a query is a RELATION.
This is why you can't use ORDER BY in a view, because then your result from the view is not a relation. This isn't true when using a TOP statement as the ORDER BY is becoming part of the filter. In this case though the order of the results is not guaranteed as we're returning a set. THIS IS IMPORTANT!

## 7. TOP / FETCH-OFFSET

## Misc
SQL Server has 3 logical states! TRUE, FALSE and UNKNOWN.
The unknown result is given when comparing against a NULL.
In most cases any result that returns an UNKNOWN is discarded. This is important when you consider the above.

This is also important in a check constraint as these check for the presence of a FALSE, rather than a NOT TRUE.

Tables are equivalent to sets, and sets do not have an order. We only apply order at step 6 (ORDER BY).
We can't use an alias from the SELECT clause in the FROM statement as the FROM clause is computed first.
SQL is based on multi-set theory where sets can contain duplicates, hence the DISTINCT clause.
The image of the order (LQP Diagram) is in Itzik's articles on sqlmag.com

# Joins
A  join is a table operator from the FROM clause. Takes place between two tables, left and right.
If using an INNER JOIN it doesn't amtter if you filter in the ON or WHERE clauses.

## Cross Joins
Old SQL-89 standard only supported CROSS JOIN, using `SELECT * FROM t1, t2`.
SQL-92 changed this to support outer joins, so the old comma syntax was no longer sufficient. Thsi is when the JOIN and sub join keywords were intorduced. Comma is still supported but it will always be a CROSS JOIN.

Self cross joins are possible between multiple instances of the same table, this requires aliasing.

```
SELECT t1.id, t1.name,
        t2.id, t2.name
FROM Employees t1
    CROSS JOIN Employees t2;
```

Can be used to create sequences of values, such as a numbers table.

## Inner Joins
Implements two stages, a Cartesian join then a filter.
ALWAYS use the SQL-92 stadard:

```
SELECT ...
FROM <table> t1
    INNER JOIN <table> t2
    ON t1.id = t2.id;
```

## Composite Joins
These are based on joining on multiple columns. Check this if your primary keys are composites!

## Non-Equi Joins
Kinda the flip of a regular join. Basically instead of `t1.id = t2.id` use `t1.id < t2.id`.
This is a good way to return unique pairs.

## Multi-Join Queries
We process joins in written order, so this is important when we specify more than 1 join.
The optimiser will use cost estimates to re-order joins to be efficient, as long as it will return the correct result set. You can use WITH FORCE ORDER to enforce your join order and bypass the optmiser and force it to perform your steps in the order written.
When we get to multi-joins there is less option for the optmiser to re-order your joins for speed.

## Outer Joins
One (or both) tables are marked as preserved. NULLs are used in the non-preserved columns in outer rows.
Can be LEFT, RIGHT or FULL.

### Including missing values
Use an auxiliary table or function to create a list of values then outer join to your fact table:
(see slides p 25)

### Left Outer Join syntax
Because of the join order, if you're doing outer joins on multiple tables, place the second join in brackets, and leave the last ON out to make it clear which order things are being processed in.

### Using COUNT aggregate with outer joins
If you're using COUNT in an outer join, specify the column name to count rather than * otherwise it will count NULLs

# Subqueries

## Self contained Subqueries
Subquery independent of outer query.
EG order with maximum orderID
```
SELECT orderid, orcerdate, empid, custid
FROM Sales.Orders
WHERE orderid = (
                    SELECT MAX(O.orderid)
                    FROM Sales.Orders AS O
                );
```

## Scalar Subqueries
If a subquery returns an empty set, the results is converted to NULL
Will break if returns more than one value.
```
SELECT orderid
FROM Sales.Orders
WHERE empid = (
                SELECT E.empid
                FROM HR>Employees AS E
                WHERE E.lastname LIKE N'B%'
                );
```

## Multi-valued Subqueries
Things like IN a subquery set, also the reverse NOT IN.

## Correlated Subqueries
Where inner query correlated to outer table.
```
SELECT custid, orderid, orderdate, empid
FROM Sales.Order AS O1
WHERE orderid = (
    SELECT MAX(O2.orderid)
    FROM Sales.Orders AS O2
    Where O2.custid = O1.custid
    );
```

## EXISTS
Will accept a subquery as input. Returns true if at least one row, false if empty set. Never returns UNKNOWN.
Preferable to IN as presents less opportunities for issues to arise with NULLs and is more natural language.

If using a LEFT OUTER JOIN with a WHERE it is faster to use an NOT EXISTS or EXISTS instead as there are specific T-SQL shortcuts to detect the Semi-join and only return the top 1 row when using EXISTS.

## Troubles with NOT IN and NULLs
Customers from Spain who placed no orders:
```
SELECT custid, companyname
FROM Sales.Customers AS C
WHERE custid NOT IN (
    SELECT O.custid
    FROM Sales.Orders AS O
    );
```
Works fine of there are no NULLs. If subquery returns a single NULL then the query will return an empty set as there will be a comparison to UNKNOWN (from comparing to NULL).
Can fix this by adding a line to eliminate NULLs or just use NOT EXISTS instead. EXISTS will optimize better:
```
SELECT custid, companyname
FROM Sales.Customers AS C
WHERE NOT EXISTS (
    SELECT *
    FROM Sales.Orders AS O
    WHERE O.custid = C.custid
    );
```
An excellent practice is to always use table aliases in subqueries to prevent any confusion.

# Table Expressions

## Derived tables
Example:
```
SELECT *
FROM (
    SELECT custid, companyname
    FROM Sales.Customers
    WHERE country = N'USA'
    ) AS USACusts;
```
These can accept parameters and can be nested. Nesting sucks to follow, and multiple references to overlapping columns means it's often better to use a CTE.

## Common Table Expressions
Example:
```
WITH USACusts AS
(
    SELECT custid, companyname
    FROM Sales.Customers
    WHERE country = N'USA
)
SELECT * FROM USACusts;
```
These can use inline or external aliasing, and supports multiple CTEs:
```
WITH C1 AS
(
    SELECT YEAR(orderdate) AS orderyear, custid
    FROM Sales.Orders
),
C2 AS
(
    SELECT orderyear, COUNT(DISTINCT custid) as numcusts
    FROM C1
    GROUP BY orderyear
)
SELECT orderyear, numcusts
FROM C2
WHERE numcusts > 70;
```
Because we name the CTE to start with, we can use multiple copies of the CTE in the FROM statement, so it can be compared to itself.

# Views
Views are basically table expressions that can be re-used.
Also inline table values functions, which are basically views that accept parameters.
`CREATE FUNCTION \<blah> (@parm as int) RETURNS TABLE AS RETRUN \<queryusingparm>`

# Apply
Similar to a join, but looks at two inputs a set. Has two forms; CROSS APPLY and OUTER APPLY.
CROSS APPLY works similar to a cross join. Returns only rows for the left table where the right table expression is an empty set.
OUTER APPLY adds rows from the left table for values on the right which would return an empty set.

# Set operators
Set operators work on query results with compatible schema.
Operators are UNION (ALL), INTERSECT and EXCEPT. INTERSECT precedes UNION and EXCEPT in execution order.

## UNION
UNION ALL returns a result with all rows from both input sets.
UNION returns a result set with distinct rows from both input sets, it's implicitly distinct.
NULLS compared to NULLs resolve as TRUE.
UNION ALL is usually recommended for performance, even if UNION may be appropriate.

## INTERSECT
Returns only distinct rows that appear in both sets.

## EXCEPT
Returns distinct rows that appear in the first set and not in the second.

# Recent T-SQL Additions
* DROP IF EXISTS and CREATE OR ALTER (2016sp1) are self explanatory!
* TRIM (vNext) trails leading and trailing spaces.
* STRING_SPLIT (2016) splits a string into a set based on a delimiter. (NOTE: Cardinality estimator guesses 50 rows every time)
* STRING_AGG (vNext) creates ordered sets (pulls multiple row values into a string to return as a column)
* DATEDIFF_BIG (2016) returns a BIGINT datediff (for bigger values)
* AT TIME ZONE (2016) self explanatory!
* Temporal Tables (2016) Keep previous versions of data in a history table. The table must have a primary key, you specify SYSTEM_VERSIONING = ON and name a table to track the history as well as two DATETIME2 rows specially created as the transaction start and end times and the clause PERIOD FOR SYSTEM_TIME specifying these columns. **NOTE:** The new columns for this feature can be marked as [HIDDEN] so they won't show up in a SELECT \*. The history table is obfuscated away, it won't appear in objex at table level. To query a temporal table you use the FOR SYSTEM_TIME AS OF \<datetime> clause to see the value at a certain time. You can also do FROM \<start> TO \<end>, BETWEEN \<start> AND \<end> and CONTAINED IN(\<start>, \<end>).
