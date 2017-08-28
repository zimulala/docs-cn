# Aggregate (GROUP BY) Functions

## Aggregate (GROUP BY) Function Descriptions

本小结描述当前 TiDB 支持的内建聚合函数

| Name	                                                                                                        | Description                                       |
|:--------------------------------------------------------------------------------------------------------------|:--------------------------------------------------|
| [`COUNT()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_count)                   | 返回当前 group 里的数据的条数                     |
| [`COUNT(DISTINCT)`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_count-distinct)  | 返回当前 group 里不同数据的条数                   |
| [`SUM()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_sum)                       | 放回当前 group 里的和                             |
| [`AVG()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_avg)                       | 返回当前 group 里这一列的均值                     |
| [`MAX()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_max)                       | 返回当前 group 里这一列的最大值                   |
| [`MIN()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_min)                       | 返回当前 group 里这一列的最小值                   |
| [`GROUP_CONCAT()`](https://dev.mysql.com/doc/refman/5.7/en/group-by-functions.html#function_group-concat)     | 返回当前 group 里这一列所有字符串拼接在一起的值   |

- 除非特别说明，所有的聚合函数都会忽略 `NULL` 值。
- 如果在使用聚合函数的时候没有使用 `GROUP BY` 子句，这等效于将所有的数据视为一个 group 并在这个 group 上做聚合。详细请参考：[TiDB 对 GROUP BY 的处理](#tidb-handling-of-group-by)。

## GROUP BY 修饰符

TiDB 当前不支持任何 group by 修饰符，我们会在将来实现这一特性，详细请参考：[#4250](https://github.com/pingcap/tidb/issues/4250)。

## <span id="tidb-handling-of-group-by">TiDB 对 GROUP BY 的处理</span>

TiDB 在 group by 上的行为和 MySQL 处于非 `[`ONLY_FULL_GROUP_BY`](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_only_full_group_by)` sql mode 下的行为一样：允许 select、having、order by 中出现对非聚合列的引用，
TiDB performs equivalent to MySQL with sql mode [`ONLY_FULL_GROUP_BY`](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html#sqlmode_only_full_group_by) being disabled: permits the `SELECT` list, `HAVING` condition, or `ORDER BY` list to refer to non-aggregated columns even if the columns are not functionally dependent on `GROUP BY` columns.

For example, this query is illegal in MySQL 5.7.5 with `ONLY_FULL_GROUP_BY` enabled because the non-aggregated column "b" in the `SELECT` list does not appear in the `GROUP BY`:

```sql
drop table if exists t;
create table t(a bigint, b bigint, c bigint);
insert into t values(1, 2, 3), (2, 2, 3), (3, 2, 3);
select a, b, sum(c) from t group by a;
```

The preceding query is legal in TiDB. TiDB does not support sql mode `ONLY_FULL_GROUP_BY` currently, we'll do it in the future, for more inmormation, see [#4248](https://github.com/pingcap/tidb/issues/4248).

Suppose that we execute the following query, expecting the results to be ordered by "c":
```sql
drop table if exists t;
create table t(a bigint, b bigint, c bigint);
insert into t values(1, 2, 1), (1, 2, 2), (1, 3, 1), (1, 3, 2);
select distinct a, b from t order by c;
```

To order the result, duplicates must be eliminated first. But to do so, which row should we keep? This choice influences the retained value of "c", which in turn influences ordering and makes it arbitrary as well.

In MySQL, a query that has `DISTINCT` and `ORDER BY` is rejected as invalid if any `ORDER BY` expression does not satisfy at least one of these conditions:
- The expression is equal to one in the `SELECT` list
- All columns referenced by the expression and belonging to the query's selected tables are elements of the `SELECT` list

But in TiDB, the above query is legal, for more information see [#4254](https://github.com/pingcap/tidb/issues/4254).

Another TiDB extension to standard SQL permits references in the `HAVING` clause to aliased expressions in the `SELECT` list. For example, the following query returns "name" values that occur only once in table "orders":
```sql
select name, count(name) from orders
group by name
having count(name) = 1;
```

The TiDB extension permits the use of an alias in the `HAVING` clause for the aggregated column:
```sql
select name, count(name) as c from orders
group by name
having c = 1;
```

Standard SQL permits only column expressions in `GROUP BY` clauses, so a statement such as this is invalid because "FLOOR(value/100)" is a noncolumn expression:
```sql
select id, floor(value/100)
from tbl_name
group by id, floor(value/100);
```

TiDB extends standard SQL to permit noncolumn expressions in `GROUP BY` clauses and considers the preceding statement valid.

Standard SQL also does not permit aliases in `GROUP BY` clauses. TiDB extends standard SQL to permit aliases, so another way to write the query is as follows:
```sql
select id, floor(value/100) as val
from tbl_name
group by id, val;
```

## Detection of Functional Dependence

TiDB does not support sql mode `ONLY_FULL_GROUP_BY` and detection of functional dependence, we'll do it in the future, for more information see [#4248](https://github.com/pingcap/tidb/issues/4248).
