```sql
UPDATE t_emp SET sal = 10000
WHERE deptno = 
(SELECT  deptno FROM t_dept WHERE dname='SALES');

UPDATE t_emp e JOIN t_dept d ON e.deptno = d.deptno
AND d.dname = 'SALES'
SET e.sal = 10000, d.dname='销售部';
```

- 第一个sql因为把子查询写在where子句中，会出现相关子查询，相关子查询的坑可以查看[子查询的坑](子查询的坑.md)
- 第二个sql解决子查询的坑，同时还能一条语句修改两张表。
