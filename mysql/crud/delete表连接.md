```sql
DELETE e, d FROM t_emp e JOIN t_dept d
ON e.deptno = d.deptno AND d.dname='销售部';
```
sql的意思是删掉 名字为‘销售部’的部门，以及下面的所有员工。
如果是 Delete  e FROM.... 则只删除部门下的员工，部门本身的记录不删。
