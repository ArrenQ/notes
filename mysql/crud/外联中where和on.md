外关联时需要注意where和on可能会导致查询结果的不同
如
```sql
SELECT e.ename, d.dname
FROM t_emp e 
LEFT JOIN t_dept d ON e.deptno = d.deptno
AND d.deptno = 10;

SELECT e.ename, d.dname
FROM t_emp e 
LEFT JOIN t_dept d ON e.deptno = d.deptno
WHERE d.deptno = 10;

```
这两个SQL 中，由于使用left join。以左表为基础，如果又表没有相关记录，则左表记录保留，右表列为null。
注意：条件d.deptno = 10 是右表的条件。
由于不是限制左表，因此如果是ON 条件的话，条件不满足不影响左表展示，这样会出现左表字段展示，右表字段为null
而如果使用where条件，则不满足的条件虽然在JOIN时会出现，但在where条件执行时被过滤，也就不会展示记录。

这也说明了联表查询时，join的动作比where先执行。
