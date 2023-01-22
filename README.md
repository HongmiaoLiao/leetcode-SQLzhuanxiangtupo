# (专项突破SQL-SQL入门)

[leetcode页面](https://leetcode.cn/study-plan/sql/?progress=4oysn33)

[toc]

## 选择

### 595 大的国家

方法一：使用where和or

```SQL
select name, population, area from World where area >= 3000000 or population >= 25000000;
```

方法二：使用where子句和UNION
使用or会使索引失效，在数据量大的时候查找效率较低，通常建议使用union代替or

```SQL
select name, population, area from world where area >= 3000000
union
select name, population, area from world where population >= 25000000;
```

### 1757 可回收且低脂的产品

方法：使用where和and，sql等号是一个等号 =

```SQL
select product_id from Products where low_fats = 'Y' and recyclable = 'Y';
```

### 584 寻找用户推荐人

方法
tips：

* NULL不是值，所以不能对其使用谓词，如果使用谓词，结果是unknown。
* SQL的保留字中，很多都被归类为谓词一类，例如>,<>,= 等比较谓词，以及BETWEEN, LIKE, IN, IS NULL等。总结，谓词是一种特殊的函数，返回值是真值。(前面提到的诶个谓词，返回值都是true, false, unknown,SQL是三值逻辑，所以有三种真值）。
* 因为查询结果只会包含WHERE子句里的判断结果为true的行！不包含判断结果为false和unknown的行。且不仅是等号，对NULL使用其他比较谓词（比如> NULL），结果也都是unknown。

记忆：优先级：
AND:    false > unknown > true
OR:       true > unknown > false

方法一: 使用or或联合（联合需要加all），

```SQL
# select name from customer where referee_id != 2 or referee_id is null;
select name from customer where referee_id != 2
union all
select name from customer where referee_id is null;
```

方法二：使用not in

```SQL
select name from customer where id not in 
(select id from customer where referee_id = 2);
```

### 183 从不订购的客户

方法：使用not in语句

```SQL
select Name Customers from Customers C where C.Id not in
(select CustomerId from Orders);
```

## 排序&修改

### 1873 计算特殊奖金

方法一：使用if，条件也可以name like 'M%'

```SQL
select employee_id, if(left(name, 1) = 'M' or mod(employee_id, 2) = 0, 0, salary) as bonus
from Employees
order by employee_id;
```

方法二：使用CASE WHEN ... THEN ... else ..

```SQL
SELECT
    employee_id, 
    CASE WHEN 
        MOD(employee_id, 2) = 1 AND left(name, 1) != 'M' 
    THEN salary 
    ELSE 0 END
    AS bonus 
FROM Employees 
ORDER BY employee_id;
```

### 627 变更性别

方法一：使用if

```SQL
update Salary set sex = if(sex = 'f', 'm', 'f');
```

方法二：使用case when

```SQL
UPDATE salary
SET
    sex = CASE sex
        WHEN 'm' THEN 'f'
        ELSE 'm'
    END;
```

### 196 删除重复的电子邮箱

方法一：不能从select里面删除，需要再创建一个临时表，于是有

```SQL
delete from Person 
where id not in 
(select * from 
    (select min(id) from Person group by email)
as p);
```

方法二：使用自连接

```SQL
delete p1 from Person p1, Person p2 
where p1.email = p2.email && p1.id > p2.id;
```

## 字符串处理函数/正则

字符串拼接函数

* CONCAT(s1, s2,...,sn)，将字符串s1,s2...,sn拼接成一个字符串
* left(str, n)，从字符串左边截取n个字符
* substring(str, n)，截取从字符串第n个字符到结尾，substring(str,n,len) 截取从n开始的len个字符
* lower(str)全部变为小写，upper(str)全部变为大写
* group_concat()，和group by相关，将同一个分组的值连接起来，返回一个字符串结果

### 1667 修复表中的名字

方法：使用字符串拼接函数

```SQL
select user_id,  concat(upper(left(name, 1)), lower(substring(name, 2))) name
from Users
order by user_id;
```

### 1484 按日期分组销售产品

方法: 使用group by 配合group_concat函数，注意需要用distinct去重

```SQL
select sell_date, count(distinct product) num_sold, group_concat(distinct product) products
from Activities
group by sell_date
order by sell_date;
```

### 1527 患某种疾病的患者

方法一：使用or判断两种情况

```SQL
select * from Patients 
where conditions like 'DIAB1%' or conditions like '% DIAB1%';
```

方法二：使用正则表达式

```SQL
select * from Patients
where conditions regexp '^DIAB1|.*\\sDIAB1';
# \\s为空格
```

## 组合查询&指定选取

### 1965 丢失信息的雇员

方法：先将两个表的id进行并集，这个集合如果id只出现一次，说明在某一方缺失。只出现一次的id就是答案。

```SQL
select employee_id from
    (select employee_id from Employees
    union all
    select employee_id from Salaries
) as tmp
group by employee_id
having count(employee_id) = 1
order by employee_id;
```

### 1795 每个产品在不同商店的价格

方法一：行转列，使用union联合，

```SQL
select product_id, "store1" store, store1 price from Products where store1 is not null
union
select product_id, "store2" store, store2 price from Products where store2 is not null
union
select product_id, "store3" store, store3 price from Products where store3 is not null;
```

行转列，列转行，使用group by + if, 由于group by需要配合上聚合函数，所以对于只有一个值的分组，用sum,max,min都可以。是上面的逆操作

```SQL
SELECT 
  product_id,
  SUM(IF(store = 'store1', price, NULL)) store1,
  SUM(IF(store = 'store2', price, NULL)) store2,
  SUM(IF(store = 'store3', price, NULL)) store3 
FROM
  Products1 
GROUP BY product_id ;
```

### 608 树节点

方法一：使用两重if，先判断是否是根节点，再判断是否是叶子结点

```SQL
select id, 
    if(p_id is null, "Root", 
        if(id in (select p_id from tree), "Inner", "Leaf")) type 
from tree;
```

方法二：使用case when流程控制语句

```SQL
select id,
    case
        when id = (select id from tree where p_id is null)
            then "Root"
        when id in (select p_id from tree)
            then "Inner"
        else "Leaf"
    end type
from tree;
```

### 176 第二高的薪水

注意点：需要去重distinct，要考虑没有第k大的值需要返回null
方法一：使用order by + limit offset，再多一重select可以在没有第k大（offset k-1)时返回null

```SQL
select (
    select distinct salary 
    from Employee order 
    by salary desc 
    limit 1 offset 1
) SecondHighestSalary;
```

方法二：ifnull函数和LIMIT子句

```SQL
select ifnull(
        (select distinct salary 
        from Employee order 
        by salary desc 
        limit 1 offset 1), null)
    as SecondHighestSalary;
```

## 合并

### 175 组合两个表

方法：使用外连接outer join， 左边所有信息加上左边和右边的交集部分数据

```SQL
select firstName, lastName, city, state 
from Person 
left join Address on Person.personId = Address.PersonId;
```

### 1581 进店却未进行交易过的顾客

方法一：找到Visits表中visit_id不在Transactions表中的customer_id,说明这些人没有购物过，在进行分组查询计数

```SQL
select customer_id, count(customer_id) count_no_trans 
from Visits
where visit_id not in (
        select visit_id from Transactions
    )
group by customer_id;
```

方法二：使用外连接查询，Visits左连接Transactions，查询出连接后字段为null的行，就是在Transactions没有交易记录的行，即没有交易的客户，最后进行分组计算

```SQL
select customer_id, count(customer_id) count_no_trans 
from Visits
left join Transactions on Visits.visit_id = Transactions.visit_id
where amount is null
group by(customer_id);
```

### 1148 文章浏览Ⅰ

方法一：直接查询+distinct去重

```SQL
select distinct author_id id 
from Views 
where author_id = viewer_id 
order by id;
```

### 197 上升的温度

DATEDIFF(date1, date2)：返回起始时间date1和结束时间date2之间的天数, date1 - date2,

方法一：使用隐式内连接

```SQL
select w1.id from Weather w1, Weather w2 
where datediff(w1.recordDate, w2.recordDate) = 1 and w1.Temperature > w2.Temperature;
```

方法二：使用显式内连接

```SQL
select a.id 
from weather a 
join weather b on datediff(a.recordDate, b.recordDate) = 1
where a.Temperature > b.Temperature; 
```

### 607 销售员

方法：先对Company和Orders做内连接，找到买过RED的人员的id，在查询id不等于买过RED人员id的人

```SQL
select s.name from SalesPerson s
where s.sales_id not in (
    select o.sales_id
    from Orders o
    join Company c on
        o.com_id = c.com_id 
    where c.name = "RED"
);
```

## 计算函数

### 1141 查询近30天活跃用户数

注意点，一个用户可能在一天有多次访问，所以count user_id时需要加入distinct

方法一：使用datediff函数，然后分组计数

```SQL
select activity_date day, count(distinct user_id) active_users 
from Activity 
where datediff("2019-07-27", activity_date) between 0 and 29 
group by activity_date;
```

方法二：直接使用between

```SQL
select activity_date day, count(distinct user_id) active_users 
from Activity 
where activity_date between "2019-06-28" and "2019-07-27"
group by activity_date;
```

### 1693 每天的领导和合伙人

方法：分组去重统计

```SQL
select 
    date_id, 
    make_name, 
    count(distinct lead_id) unique_leads, 
    count(distinct partner_id) unique_partners
from DailySales
group by date_id, make_name;
```

### 1729 求关注者的数量

方法：分组统计

```SQL
select user_id, count(distinct follower_id) followers_count
from Followers
group by user_id;
```

### 586 订单最多的客户

方法一：使用分组将计算订单数，再对订单数进行降序排序，用limit 1取排序后第一个,这样对只有一个最多的答案有效

```SQL
select customer_number
from Orders
group by customer_number
order by count(*) desc
limit 1;
```

方法二：对于多行并列多的数据，可以先查询出最多的订单数，再分组进行匹配进行。

```SQL
select customer_number
from Orders
group by customer_number
having count(*) = (
    select count(customer_number) cnt
    from Orders
    group by customer_number
    order by cnt desc
    limit 1
);
```

### 511 游戏玩法分析 Ⅰ

方法：使用聚合函数对分组查询的分组进行操作

```SQL
select player_id, min(event_date) first_login
from Activity
group by player_id;
```

### 1890 2020年最后一次登录

方法：找到2020年最晚的一天即可,可以用year函数选出再2020年的行，然后再进行分组+max聚合函数

```SQL
select user_id, max(time_stamp) last_stamp
from Logins 
where year(time_stamp) = "2020"
group by user_id;
```

### 1741 查找每个员工花费的总时间

方法：使用分组查询对id和日期进行分组（即一人一天），使用聚合函数sum计算总时间

```SQL
select event_day "day", emp_id, sum(out_time - in_time) total_time
from Employees
group by emp_id, event_day
```

## 控制流

case when的两种语句

```SQL
# 语句一
CASE input_expression
    WHEN expression1 THEN result_expression1
    WHEN expression2 THEN result_expression2
    [...n]
    ELSE result_expression
    END
```

```SQL
# case语句二
CASE
    WHEN expression1 THEN result_expression1
    WHEN expression2 THEN result_expression2
    [...n]
    ELSE result_expression
    END
```

* COALESCE是一个函数， (expression_1, expression_2, ...,expression_n)依次参考各参数表达式，遇到非null值即停止并返回该值。如果所有的表达式都是空值，最终将返回一个空值。使用COALESCE在于大部分包含空值的表达式最终将返回空值。
* 外连接时要注意where和on的区别，on是在连接构造临时表时执行的，不管on中条件是否成立都会返回主表（也就是left join左边的表）的内容，where是在临时表形成后执行筛选作用的，不满足条件的整行都会被过滤掉

### 1393 股票的资本损益

方法：对股票名称进行分组后，使用聚合函数sum，使用if或when case将买的价格设定为负，卖的价格定位正，进行求和

```SQL
select stock_name, sum(
    case operation
    when "Buy" then -price
    else price
    end 
    ) capital_gain_loss
capital_gain_loss
from Stocks
group by stock_name;
```

```SQL
select stock_name, 
        sum(if(operation="Buy", -price, price))  capital_gain_loss
from Stocks
group by stock_name;
```

### 1407 排名靠前的旅行者

方法：利用左连接对两表拼接，然后使用分组+sum聚合函数进行统计，对统计还需要null的判断，可以用ifnull或coalesce函数
注意这里要group by id而不是name，因为一个名字可能不只一个id

```SQL
select u.name, ifnull(sum(r.distance),0) travelled_distance 
from Users u 
left join Rides r on  u.id = r.user_id
group by u.id
order by travelled_distance desc, name;
```

### 1158 市场分析 Ⅰ

方法一：使用左外连接进行拼接，在连接时使用on，能查询出为空的字段。然后使用分组加count聚合求出总数

```SQL
select u.user_id buyer_id, u.join_date, count(o.order_id) orders_in_2019
from Users u
left join Orders o on u.user_id = o.buyer_id and year(o.order_date) = "2019"
group by u.user_id;
```

方法二：先用分组+count对Orders表查询出购买者id和购买数量，此时是不包含为null的内容，以此作为临时表，和Users表作左连接。

```SQL
select u.user_id buyer_id, u.join_date, ifnull(t.cnt,0) orders_in_2019
from Users u
left join (
    select buyer_id, count(*) cnt
    from Orders
    where year(order_date) = "2019"
    group by buyer_id
) t
on u.user_id = t.buyer_id;
```

## 过滤

### 182 查找重复的电子邮箱

方法：使用group by + having子语句，用count聚合函数过滤掉出现次数小于1的

```SQL
select Email
from Person 
group by Email 
having count(*) >= 2;
```

### 1050 合作过至少三次的演员和导演

方法：直接group by对两个字段分组，再用having count进行计数过滤

```SQL
select actor_id, director_id 
from ActorDirector 
group by actor_id, director_id 
having count(*) >= 3;
```

### 1587 银行账户概要 Ⅱ

方法：将用户表和交易表进行左外连接，然后进行分组统计，用sum聚合函数过滤

```SQL
select u.name NAME, sum(t.amount) BALANCE 
from Users u 
    left join Transactions t 
    on u.account = t.account
group by u.account
having sum(t.amount) > 10000;
```

### 1084 销售分析 Ⅲ

方法：先进行左外连接，然后进行分组过滤
注意，having语句不能使用between，应该用聚合函数进行过滤

```SQL
select p.product_id, p.product_name
from Product p
    left join Sales s
    on p.product_id = s.product_id
group by p.product_id
having max(s.sale_date) <= "2019-03-31" and min(s.sale_date) >= "2019-01-01";
```
