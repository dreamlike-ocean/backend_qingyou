## DQL查询

#### 基础查询

- 语法

```
SELECT
  查询列表
FROM
  表名;
```

- 起别名

```
# 起别名
SELECT 
  name AS my_name   # 第一种方式
FROM 
  students;  

SELECT 
  name my_name   # 第二种方式
FROM 
  students; 
```

- 去重

```
SELECT 
  DISTINCT name 
FROM 
  students;
```

- 字段拼接

```
MySQL中的'+'号只能作为运算符
SELECT 
  CONCAT(last_name,first_name) AS name
FROM
  students;
```

#### 条件查询

- 语法

```
SELECT
  查询列表
FROM
  表名
WHERE
  筛选条件;
```

- 逻辑运算符

```
&& 等价于AND 
|| 等价于OR 
 ！等价于NOT
# 从students表中选取id在10到20之间的学生的name(开区间)
SELECT
  name 
FROM 
  students
WHERE
  id > 10 AND id < 20;
```

#### 模糊查询

- like

```
通配符：% 任意多个字符
        _ 任意单个字符
转义字符： '\_'  '\%'
           '&_' ESCAPE '&' (&可为任意符号)
# 从students表中查询name字段中第三个为A，第五个为_，第六个为%的学生
SELECT
  *
FROM
  students
WHERE
  name LIKE '__A_\_\%%';
# 等价于 name LIKE '__A_$_$%%' ESCAPE '$';
```

- between a and b

```
# 从students表中查询id在100到120之间的学生信息(闭区间)
SELECT
  *
FROM
  students
WHERE
  id BETWEEN 100 ADN 120;
# 等价于 id >=100 AND id <=120;
1、BETWEEN AND是闭区间
2、a表达式必须<=b表达式
```

- in

```
# 从students表中查询id为60，70，80，90的学生信息
SELECT
  *
FROM
  students
WHERE
  id IN ('60','70','80','90');
# 等价于 id = '60' OR id = '70' OR id = '80' OR id = '90';
```

- is null

```
# 从students表中查询id不存在的学生信息
SELECT
  *
FROM
  students
WHERE
  id IS NULL;
# 等价于id <=> NULL
# <=>安全等于   <>安全不等于
# mysql中不能用id = NULL判断
```

#### 排序查询

- 语法

```
SELECT
  查询列表
FROM
  表名
(筛选条件)
ORDER BY 
  排序列表 （ASC|DESC）
# 默认ASC升序
排序列表可以是单个字段、多个字段、表达式、函数、别名等
```

#### 常见函数

##### 单行函数

- 字符函数

```
# 1、1ength获取参数值的字节个数
SELECT LENGTH('hello');
# 5


# 2、concat 拼接字符串
SELECT CONCAT(last_name,'_',first_name) 姓名 FROM students;


# 3、upper、lower 大小写转换
SELECT UPPER('hello');
# HELLO


# 4、substr\substring 字符串截取
# 注意所有SQL语句的索引都是从1

# 返回从第三个位置开始到字符串结尾的子字符串
SELECT SUBSTR('I love you',3);
# love you

# 返回从第三个位置开始长度为4的子字符串
SELECT SUBSTR('I love you',3，4);
# love

# 返回从倒数第五个位置开始长度为3的子字符串
SELECT SUBSTR('I love you',-3，3);
# you


# 5、instr 返回字符串第一次出现的索引，如果找不到则返回0
SELECT INSTR('I love you','love');
# 3


# 6、trim 去除前后空格
SELECT LENGTH('        hello        ');
# 5

# 去除其他字符
SELECT TRIM('a' FROM 'aaaaaaaaaaaaaahelloaaaaaaaaaaa');
# hello


# 7、lpad(rpad) 用指定字符实现左(右)填充指定长度
SELECT LPAD('hello'，10，'$');
# $$$$$hello

# 如果原长度超过指定长度，则从右开始截断
SELECT LPAD('hello',2,'$');
# he


# 8、replace 替换
SELECT REPLACE('I love you','you','she');
# I love she
```

- 数学函数

```
# 1、round 四舍五入
SELECT ROUND(1.65);
# 2


# 2、ceil(floor) 向上(向下)取整，返回>=(<=)该参数的最小整数
SELECT CEIL(1.02);
# 1


# 3、truncate 截断(保留小数点后多少位)
SELECT TRUNCATE(1.14514,1);
# 1.1


# 4、mod 取余
SELECT MOD(10,3);
#等同于SELECT 10%3;
# 1
```

- 日期函数

```
# 1、now 返回当前系统日期+时间
SELECT NOW();
# 2021-04-09 16:05:36


# 2、curdate 返回当前系统日期，不包含时间
SELECT CURDATE();
# 2021-04-09


# 3、curtime 返回当前系统时间，不包含日期
SELECT CURTIME();
# 16:05:36


# 4、获取指定的部分年、月、日、小时、分钟
SELECT YEAR(NOW());
# 2021
# 类似还有MONTH()、DAY()、HOUR()等


# 5、str_to_date 将字符通过指定格式转化为日期
SELECT STR_TO_DATE('1998-3-2','%Y-%c-%d');
# 1998-03-02


# 6、date_format 将日期转换为字符
SELECT DATE_FORMAT(NOW(),'%Y年%m月%d日');
# 2021年04月09日                                                                 
```

- 流程控制函数

```
# 1、if函数： 类似于三目运算符
SELECT IF(10<5,'大','小');
# 大


# 2、case函数： 
# 使用一：类似于switch case
CASE sid;
WHEN 10 THEN grade+10;
WHEN 20 THEN grade+20;
WHEN 30 THEN grade+30;
ELSE grade+2;
END

# 使用二：类似于 if else
CASE
WHEN sid>2000 THEN 'A'
WHEN sid>1000 THEN 'B'
WHEN sid>500  THEN 'C'
ELSE 'D'
END
```

- 其他函数

```
# version 查看版本
# datebase 查看数据库
# user 查看当前用户
```

##### 分组函数

```
功能：
用作统计使用，又称为聚合函数或统计函数或组合函数
分类：
sum 求和、 avg 平均值、 max 最大值、 min 最小值、 count 计算个数
特点：
1、sum、avg一般处理数值型
   max、min、count可以处理任何类型
2、以上分组函数都忽略null值
3、可以和distinct搭配实现去重
4、count(*)等价于count(常量),作用为统计表里全部数据的个数
5、和分组函数一同查询的字段要求是group by后的字段                                                                                                                                                                                                                                                                                                                                                                                                                              
```

#### 分组查询

- 语法

```
SELECT 分组函数，列(要求出现在group by的后面)
FROM 表
[WHERE 筛选条件]
GROUP BY 分组的列表
[ORDER BY 子句]
注意：查询列表必须要求是分组函数+group by后出现的字段
```

- 简单分组

```
# 案例1：查询一个公司每个工种的最高工资
SELECT MAX(salary),job_id
FROM employees
GROUP BY job_id;


# 案例2：查询每个位置上的部门个数
SELECT COUNT(*),location_id
FROM departments
GROUP BY location_id;
```

- 分组前的筛选

```
在分组前用where关键字
# 案例3：查询邮箱中包含a字符的，每个部门的平均工资
SELECT AVG(salary),department_id
FROM employees
WHERE email like '%a%'
GROUP BY department_id;


# 案例4：查询有奖金的每个领导的手下员工的最高工资
SELECT MAX(salary),manager_id
FROM employees
WHERE commission_pct IS NOT NULL
GROUP BY manager
```

- 分组后的筛选

```
在分组查询后用having关键词取代where
# 案例5：查询哪个部门的员工个数>2
SELECT COUNT(*),department_id
FROM employees
GROUP BY department_id
HAVING COUNT(*)>2;


# 案例6：查询每个工种有奖金的员工的最高工资>12000的工种编号和最高工资
SELECT MAX(salary),job_id
FROM employees
GROUP BU job_id
HAVING MAX(salary)>12000;


# 案例7：查询领导编号>102的每个领导手下的员工最低工资>5000的领导编号，以及其最低工资
SELECT MIN(salary),manager_id
FROM employees
WHERE manager_id>102
GROUP BY manager_id
HAVING MIN(salary)>5000
```

- 按表达式或函数分组

```
# 案例8：按员工姓名的长度分组，查询每一组的员工个数，筛选员工个数大于5的有哪些
SELECT COUNT(*)
FROM employees
GROUP BY LENGTH(last_name)
HAVING COUNT(*)>5;
```

- 按多个字段分组

```
# 案例9：查询每个部门每个工种的员工的平均工资
SELECT AVG(salary),department_id,job_id
FROM employees
GROUP BY department_id,job_id;
#将department_id和job_id一致的分成一组
```

- 添加排序

```
# 案例10：查询每个部门每个工种的员工的平均工资，并且按平均工资降序显示
SELECT AVG(salary),department_id,job_id
FROM employees
GROUP BY department_id,job_id
ORDER BY AVG(salary) DESC;
```

#### 连接查询/多表查询

```
当查询的字段来涉及个表时，就需要用到连接查询
按年代分类：
  sql92标准：只支持内连接
  sql99标准：支持内连接+外连接（左外+右外）+交叉连接
按功能分类：
  内连接：
    等值连接
    非等值连接
    自连接
  外连接：
    左外连接
    右外连接
    全外连接
  交叉连接
```

##### sql92标准

###### 内连接

- 等值连接

```
# 案例1：查询女生名和对应的男生名
SELECT girl_name,boy_name
FROM boys,girls
WHERE girls.boyfriend_id=boys.id;
# 用表名限定字段
# 表的先后顺序没有要求


# 案例2：查询员工名和对应的部门名
SELECT name,department_name
FROM employees,departments
WHERE employees.department_id=departments.id;


# 案例3：查询员工名，工种号，工种名
SELECT name,e.job_id,job_name
FROM employees e,jobs j
WHERE e.job_id=j.id;
# 为表起别名，起了别名后不能再用原名


# 添加筛选
# 案例4：查询有奖金的员工名、部门名
SELECT name,department_name
FROM employees e, departments d
WHERE e.department_id=d.id
AND e.commission_pct IS NOT NULL


# 案例5：查询位置名中第二个字符为o的部门名和位置名
SELECT department_name, location_name
FROM departments d, locations l
WHERE d.location_id=l.location_name
AND l.location_name LIKE '_o%'


# 添加分组
# 案例6：查询每个城市的部门个数和城市名
SELECT COUNT(*),city
FROM departments d, citys c
WHERE d.city_id=c.id;
GROUP BY city;        


# 案例7：查询有奖金的每个部门的部门名和部门的领导编号和该部门的最低工资
SELECT department_name, d.manager_id, MIN(salary)
FROM departments d, employees e
WHERE e.department_id=d.id
AND commission_pct IS NOT NULL
GROUP BY department_name, d.manager_id;


# 添加排序
# 案例8：查询每个工种的工种名和员工的个数，并且按员工个数降序
SELECT job_name, COUNT(*)
FROM jobs j, employees e
WHERE e.job_id=j.id
GROUP BY job_name
ORDER BY COUNT(*) DESC


# 实现三表连接
# 案例9：查询员工名、部门名和所在的城市
SELECT name，department_name, city
FROM employees e, departments d, citys c
WHERE e.department_id=d.id
AND d.city_id=c.id
```

- 非等值连接

```
# 案例1：查询员工的工资和工资级别
SELECT salary, grade_level
FROM employees e, job_grades g
WHERE salary BETWEEN g.lowest_sal AND g.highest_sal;
```

- 自连接

```
把一张表当作多张表来用
# 案例1：查询员工名和上级的名称
SELECT e.name, m.name
FROM employees e, employees m
WHERE e.manager_id=m.employee_id;
```

##### sql99标准(推荐)

- 语法

```
SELECT 查询列表
FROM 表1 别名 
[连接类型] JOIN 表2 别名 []
ON 连续条件
WHERE 筛选条件
关键字：
1、using
用法类似与on
using(id)等价于on a.id=b.id
连接类型：
inner 内连接
left[outer] 左外连接
right[outer] 右外连接
full[outer] 全外连接 (mysql不支持)
cross 交叉连接
```

###### 内连接

```
特点：
1、inner可以省略
2、筛选条件放在where后面，连接条件放在on后面
3、查询多表的交集部分
```

- 语法

```
SELECT 查询列表
FROM 表1 别名
INNER JOIN 表2 别名
ON 连接条件
```

- 等值连接

```
# 案例1：查询员工名、部门名
SELECT name, department_name
FROM employees e
INNER JOIN departments d
ON e.department=d.id;


# 添加筛选
# 案例2：查询名字中包含e的员工名和工种名
SELECT name, job_name
FROM employees e
INNER JOIN jobs j
ON e.job_id=j.id
WHERE name LIKE '%e%';


# 添加分组
# 案例3：查询部门个数>3的城市名和部门个数
SELECT city_name, COUNT(*)
FROM departments d
INNNER JOIN citys c
ON d.city_id=c.city_name     
GROUP BY c.city_name;


# 添加排序
# 案例4：查询员工个数>3的部门名和员工个数，并按个数降序
SELECT department_name, e.COUNT(*)
FROM department d
INNER JOIN employees e
ON e.department_id=d.id
HAVING e.COUNT(*)>3
GROUP BY department_name
ORDER BY e.COUNT(*) DESC;


# 三表连接
# 案例5：查询员工名、部门名、工种名并按部门名降序
SELECT name, department_name, job_name
FROM employees e
INNER JOIN departments d
ON e.department_i=d.id
INNER JOIN jobs j
ON e.job_id=j.id 
ORDER BY department_name DESC;                      
```

- 非等值连接

```
# 案例1：查询员工的工资级别
SELECT salary, grade_level
FROM employees e
JOIN job_grades g
ON e.salary BETWEEN g.lowest_sal AND g.highest_sal;
```

- 自连接

```
# 案例1：查询员工的名字、上级的名字
SELECT e.name, m.name
FROM employees e
JOIN employees m
ON e.manager_id=m.id;
```

###### 外连接

```
特点：
1、外连接的查询结果为主表中的所有记录，与副表中符合连接条件的记录。
2、左外连接，left左边的是主表
   右外连接，right右边的是主表
```

- 左(右)外连接

```
# 案例1 ：查询男朋友不在男生表的女生
# 左外
SELECT b.name, g.name
FROM girls g
LEFT OUTER JOIN boys b
ON g.boyfriend_id=b.id
[WHERE b.id IS NULL] 
# 只显示没有男朋友的女生

# 右外
SELECT b.name, g.name
FROM boys b
RIGHT OUTER JOIN girls g
ON g.boyfriend_id=b.id
```

- 全外连接(mysql不支持)

###### 交叉连接

```
查询结果是笛卡尔乘积
```

#### 子查询

```
含义：出现在其他语句中的select语句，称为子查询或内查询
      外部的查询语句称为主查询或外查询
分类：
按结果集的行列数不同：
  标量子查询（结果集只有一行一列）
  列子查询（结果集只有一列多行）
  行子查询（结果集只有一行多列）
  表子查询（一般情况）
按子查询出现的位置：
  select后：仅支持标量子查询
  from后：仅支持表子查询
  where或having后：标量子查询、列(行)子查询
  exists后（相关子查询）：表子查询
```

##### where或having后面

```
特点：
1、子查询放在小括号内
2、子查询一般放在条件的右侧
3、标量子查询，一般搭配着单行操作符使用 <  >  >=  <=  <>  =
4、列子查询，一般搭配多行操作符使用  in、any/some、all
```

###### 标量子查询

```
# 案例1：谁的工资比Abel高
SELECT *
FROM employees
WHERE salary>(
    SELECT salary
    FROM employees 
    WHERE name='Abel'
);


# 案例2：返回job_id与141号员工相同，salary 比143号员工多的员工 姓名，job_id和工资
SELECT name, job_id, salary
FROM employees e
WHERE e.job_id=(
    SELECT job_id
    FROM employees
    WHERE employee_id=141
)
AND salary>(
    SELECT salary
    FROM employees
    WHERE employee_id=143
);
```

###### 列子查询

```
特点：
1、返回多行
2、需要配合多行比较操作符来使用：
in/not in
any/some
all
# 案例1：返回loaction_id是1400或1700的部门中的所有员工姓名
select name
from employees
where department_id in(
    select distinct department_id
    from department
    where location_id in(1400,1700)
);


# 案例2：返回其它工种中比job_id为`IT_PROG`工种所有员工的工资都要低的员工的员工号、姓名
select id,name
from employees
where salary < all(  # min
    select distinct salary 
    from employees
    where job_id = 'IT_PROG'
) and job_id != 'IT_PROG';
```

###### 行子查询

```
特点：返回一行多列
# 案例1：查询员工编号最少并且工资最高的员工信息
select *
from employees
where (employee_id,salary)=(
    select min(employee_id),max(salary)
    from employees
);
```

##### select后面

```
特点：只支持一行一列（标量子查询）
# 案例1：查询每个部门的员工的个数
select e.*,(
    select count(*)
    from employees e
    where e.department_id=d.department_id
) 个数
from department d;
```

##### from后面

```
特点：将select返回的结果当作表（数据源）来使用
必须起别名
# 案例1、查询每个部门的平均工资的工资等级
select ag_dep.*, grade.level
from(
   select avg(salary) ag ,department_id
   from employees
   group by department_id
) ag_dep
inner join job_grades g
on ag_dep.ag between lowest_sal and highest_sal;
```

##### exists后面（相关子查询）

```
exists语法：
exists(完整的查询语句)
结果：1或0
# 案例1：查询有员工的部门名
select department_name
from departments d
where exists(
    select *
    from employees e
    where d.department_id=e.department_id
);
```

#### 分页查询

```
特点：
limit语句放在查询语句的最后

如果要显示的页数为pages，每页的条目数为size
则可以写为limit (page-1)*size,size;
```

- 语法

```
select 查询列表
from 表1
[join type join 表2
on 连接条件
where 筛选条件
group by 分组字段
having 分组后的筛选
order by 排序的字段]
limit 分页语句;
limit后分页条件有两种：
  1、limit后跟着两个数字（只有一个数字时默认第一个数字为0）：
    limit 1,3;
    表示查询3条数据，跳过1条数据（即查询2，3，4）
  2、limit和offset关键字组合使用：
    limit 5 offset 4;
    表示查询5条数据，跳过4条（即查询5，6，7，8，9）
```

- 案例

```
# 案例1：查询前五条员工信息
select * from employees limit 0,5;
select * from employees limit 5;


# 案例2：查询第11条到第25条
select * from employees limit 10,15;


# 案例3：查询有奖金的前十名员工的信息
select * 
from employees 
where commission_pct is not null
order by salary desc
limit 10;
```

#### 联合查询

```
应用场景：
当要查询的结果来自多个表，且多个表没有直接的连接关系，但是查询的信息一致
```

- 语法

```
查询语句1
union
查询语句2
union
···
```

## DML增删改

#### 插入语句

```
1、插入的值的类型要与列的类型一致或兼容
2、为nullable的列插入null的方法：
   ①列名对应的值为null
   ②不写列名和值
3、列的顺序可以调换
4、列名的个数必须与值的个数保持一致
5、可以省略列名，默认所有列，并且列的顺序和表中列的顺序一致
```

- 语法1

```
insert into 表名 (列名) value(值);
```

- 语法2

```
insert into 表名
set 列名1=值1, 列名2=值2;
两种插入方式的比较：
1、方式一支持插入多行，方式二不支持
2、方式一支持子查询，方式二不支持
```

#### 修改语句

##### 单表修改

```mysql
update 表名
set 字段=值,...
where 筛选条件
```

##### 联表修改

```mysql
update 表1 别名
inner|left|right join 表2 别名
on 连接条件
set 字段=值,...
where 筛选条件
```

#### 删除语句

##### 单表删除

```mysql
delete from 表名 where 筛选条件
# 不指定where则s
```

##### 联表删除

```mysql
delete 别名
from 表1名 别名
inner|left|right join 表2 别名
on 连接条件
where 筛选条件
```

##### truncate清空表中的数据

```mysql
truncate [table] 表名;
# 关键词table可以省略
```

区别：

1、truncate清空整个表，属于DDL语句，不能回滚

2、delete删除不会清除自增的数据

## DDL数据定义语句

#### 库

##### 创建库

```mysql
create database 库名;
# 如果库已经存在会报错
create database if not exists 库名;
# 如果不存在则创建，存在不会报错
```

##### 修改库字符集

```mysql
alter database 库名 character set 字符集名;
```

##### 删除库

```mysql
drop database 库名;
# 如果库不存在会报错
drop database if exists 库名;
# 如果存在则创建，不存在不会报错
```

#### 表

##### 创建表

```mysql
create table 表名(
	字段名 字段的类型 [(长度) 约束],
	字段名 字段的类型 [(长度) 约束],
	...
	字段名 字段的类型 [(长度) 约束]
);
```

##### 修改表

```mysql
# 修改字段名 字段的类型 [(长度) 约束]
alter table 表名 change [column] 原字段名 [新字段名] 字段类型;
# 修改字段类型
alter table 表名 modify [column] 字段名 新字段类型 [新约束];
# 添加字段
alter table 表名 add 字段名 字段的类型 [(长度) 约束];
# 删除字段
alter table 表名 drop [column] 字段名;
# 修改表名
alter table 表名 rename to 新表名;
```

##### 删除表

```mysql
drop table [if exists] book_author;
```

##### 复制表

```mysql
# 复制结构
create table 新表名 like 原表名;
# 复制结构+全部数据
create table 新表名 select * from 旧表名;
# 复制部分结构+部分数据
create table 新表名 select [字段名] from 旧表名 [条件语句];
```

