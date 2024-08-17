[176. 第二高的薪水](https://leetcode-cn.com/problems/second-highest-salary/)

编写一个 SQL 查询，获取 Employee 表中第二高的薪水（Salary） 。
```
+----+--------+
| Id | Salary |
+----+--------+
| 1  | 100    |
| 2  | 200    |
| 3  | 300    |
+----+--------+
```
例如上述 Employee 表，SQL查询应该返回 200 作为第二高的薪水。如果不存在第二高的薪水，那么查询应返回 null。
```
+---------------------+
| SecondHighestSalary |
+---------------------+
| 200                 |
+---------------------+
```
```SQL
# 使用limit，limit a offset b, 返回a条数据，b是偏移量，从0开始，
# 从b开始返回a条数据
# limit b, a
select
(
    select distinct Salary
    from Employee
    order by Salary desc
    limit 1 offset 1
) as SecondHighestSalary
```

## JOINS

![sql_join](https://gitee.com/Krains/FigureBed/raw/master/img/sql_join.png)

#### [175. 组合两个表](https://leetcode-cn.com/problems/combine-two-tables/)

表1: `Person`

```
+-------------+---------+
| 列名         | 类型     |
+-------------+---------+
| PersonId    | int     |
| FirstName   | varchar |
| LastName    | varchar |
+-------------+---------+
PersonId 是上表主键
```

表2: `Address`

```
+-------------+---------+
| 列名         | 类型    |
+-------------+---------+
| AddressId   | int     |
| PersonId    | int     |
| City        | varchar |
| State       | varchar |
+-------------+---------+
AddressId 是上表主键 
```

编写一个 SQL 查询，满足条件：无论 person 是否有地址信息，都需要基于上述两表提供 person 的以下信息：

```
FirstName, LastName, City, State
```

左连接，将左表的数据都显示出来，右表只保留公共的部分，没有的用null替代

```sql
select
    FirstName, LastName, City, State
from 
    Person as A
left join
    Address as B
on
    A.PersonId = B.PersonId
```

#### [181. 超过经理收入的员工](https://leetcode-cn.com/problems/employees-earning-more-than-their-managers/)

`Employee` 表包含所有员工，他们的经理也属于员工。每个员工都有一个 Id，此外还有一列对应员工的经理的 Id。

```
+----+-------+--------+-----------+
| Id | Name  | Salary | ManagerId |
+----+-------+--------+-----------+
| 1  | Joe   | 70000  | 3         |
| 2  | Henry | 80000  | 4         |
| 3  | Sam   | 60000  | NULL      |
| 4  | Max   | 90000  | NULL      |
+----+-------+--------+-----------+
```

给定 `Employee` 表，编写一个 SQL 查询，该查询可以获取收入超过他们经理的员工的姓名。在上面的表格中，Joe 是唯一一个收入超过他的经理的员工。

```
+----------+
| Employee |
+----------+
| Joe      |
+----------+
```

```sql
# Write your MySQL query statement below

# 连接时a.ManagerId = b.Id时a表放前面
# 反过来b表放前面，这是为什么？怎么判断的？
# 内连接
-- select
--     a.Name as Employee
-- from
--     Employee as a 
-- join
--     Employee as b 
-- on
--     a.ManagerId = b.Id
-- where
--     a.Salary > b.Salary;

# 笛卡尔积
select 
    a.Name as Employee
from   
    Employee as a,
    Employee as b 
where   
    a.ManagerId = b.Id
and
    a.Salary > b.Salary;
```

#### [182. 查找重复的电子邮箱](https://leetcode-cn.com/problems/duplicate-emails/)

编写一个 SQL 查询，查找 `Person` 表中所有重复的电子邮箱。

**示例：**

```
+----+---------+
| Id | Email   |
+----+---------+
| 1  | a@b.com |
| 2  | c@d.com |
| 3  | a@b.com |
+----+---------+
```

根据以上输入，你的查询应返回以下结果：

```
+---------+
| Email   |
+---------+
| a@b.com |
+---------+
```

```sql

# Write your MySQL query statement below

# 子查询，圆括号()，给表取别名
select
    Email
from
    (select
            Email, count(Email) as num
        from
            Person
        group by
            Email
    )as temp_table
where num >= 2;

# 使用having
-- select 
--     Email
-- from
--     Person
-- group by
--     Email
-- having count(Email) >=2;
```

#### [183. 从不订购的客户](https://leetcode-cn.com/problems/customers-who-never-order/)

某网站包含两个表，`Customers` 表和 `Orders` 表。编写一个 SQL 查询，找出所有从不订购任何东西的客户。

`Customers` 表：

```
+----+-------+
| Id | Name  |
+----+-------+
| 1  | Joe   |
| 2  | Henry |
| 3  | Sam   |
| 4  | Max   |
+----+-------+
```

`Orders` 表：

```
+----+------------+
| Id | CustomerId |
+----+------------+
| 1  | 3          |
| 2  | 1          |
+----+------------+
```

例如给定上述表格，你的查询应返回：

```
+-----------+
| Customers |
+-----------+
| Henry     |
| Max       |
+-----------+
```

```sql
# Write your MySQL query statement below

# left join, 保留左边的全部数据，右表没有则填充null
-- select
--     a.Name as Customers
-- from
--     Customers as a
-- left join
--     Orders as b 
-- on
--     a.Id = b.CustomerId
-- where
--     b.Id is null;

# 使用子查询和 not in (里面应该是记录而不是表，加了别名就成表了)
select  
    Name as Customers
from
    Customers
where Id not in(
    select  
        distinct CustomerId
    from
        Orders
);
```

