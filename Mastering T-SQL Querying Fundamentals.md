# Mastering T-SQL Querying Fundamentals - SQLBits
### Itzik Ben-Gan

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

## Cross Joins
Old SQL-89 standard only supported CROSS JOIN, using `SELECT * FROM t1, t2`.
SQL-92 changed this to support outer joins, so the old comma syntax was no longer sufficient. Thsi is when the JOIN and sub join keywords were intorduced. Comma is still supported but it will always be a CROSS JOIN.

Self cross joins are possible between multiple instances of the same table, this requires aliasing.

`SELECT t1.id, t1.name,
        t2.id, t2.name
FROM Employees t1
    CROSS JOIN Employees t2;`

Can be used to create sequences of values, such as a numbers table.

## Inner Joins
Implements two stages, a Cartesian join then a filter.
ALWAYS use the SQL-92 stadard:

`SELECT ...
FROM <table> t1
    INNER JOIN <table> t2
    ON t1.id = t2.id;`

## Composite Joins
These are based on joining on multiple columns. Check this if your primary keys are composites!

## Non-Equi Joins
Kinda the flip of a regular join. Basically instead of `t1.id = t2.id` use `t1.id < t2.id`.
This is a good way to return unique pairs.

## Multi-Join Queries
We process joins in written order, so this is important when we specify more than 1 join.
The optimiser will use cost estimates to re-order joins to be efficient, as long as it will resturn the correct result set. You can use WITH FORCE ORDER to enforce your join order and bypass the optmiser and force it to perform your steps in the order written.
When we get to multi-joins there is less option for the optmiser to re-order your joins for speed.

## Outer Joins
