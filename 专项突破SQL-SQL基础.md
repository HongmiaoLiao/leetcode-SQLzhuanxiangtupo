# (专项突破SQL-SQL基础)

[leetcode页面](https://leetcode.cn/study-plan/sql/?progress=y6ua4oj)

[toc]

## 数值处理函数

### 1699 两人之间的通话次数

方法：该题的难点在于如何合并两行记录，可以用两个if函数，将选择的字段进行调转，分为小的在前，大的在后的两个新字段，然后进行分组查询。

（注意：子查询需要将查询作为(as)一个表，不能少了这个as tmp临时表的命名。

```SQL
select person1, person2, count(*) call_count, sum(duration) total_duration
from (
    select
        if(from_id > to_id, to_id, from_id) person1,
        if(from_id > to_id, from_id, to_id) person2,
        duration
    from Calls
    ) as tmp
group by person1, person2
```

### 1251 平均售价

方法：将两个表关联，计算某个时期的销售总额，并记下那个时期的销售量，作为子表，然后对这个子表进行分组查询，用一个产品总的销售额除总的销售量计算出平均售价。

Round(num, 2) 将一个数保留2位小数

```SQL
select product_id, Round(sum(total_price) / sum(units),2) average_price from
    (select p.product_id product_id, p.price * u.units total_price, u.units units
    from Prices p
    join UnitsSold u on p.product_id = u.product_id
    where u.purchase_date between p.start_date and end_date
    ) tmp 
group by product_id
order by product_id
```

### 1571 仓库经历

方法：先将两个表根据产品id进行连接，然后根据仓库名进行分组计算

```SQL
select WAREHOUSE_NAME, sum(product_volume) VOLUME
from
    (select
        w.name WAREHOUSE_NAME, 
        w.units * p.Width * p.Length * p.Height product_volume
    from Warehouse w
    join Products p on w.product_id = p.product_id
    ) tmp
group by WAREHOUSE_NAME
```

```SQL
# 以上写法子查询多余了，可以少一个子查询
select
    w.name WAREHOUSE_NAME, 
    sum(w.units * p.Width * p.Length * p.Height) VOLUME
from Warehouse w
join Products p on w.product_id = p.product_id
group by WAREHOUSE_NAME
```

### 1445 苹果和桔子

方法一：将其分成两个表，再进行连接查询

方法二：对日期进行分组，如果是苹果，sold_num值设为正, 否则为负，然后通过sum计算

```SQL
select a.sale_date sale_date, a.sold_num - o.sold_num diff 
from
(select sale_date, sold_num from Sales where fruit = "apples") a 
join
(select sale_date, sold_num from Sales where fruit = "oranges") o
on a.sale_date = o.sale_date
```

```SQL
select sale_date, sum(if(fruit = "apples", sold_num, -sold_num)) diff
from Sales
group by sale_date;
```

### 1193 每月交易Ⅰ

方法：使用DATE_FORMAT函数对日期输出为按年月格式，然后按年月、国家分组，再对分组后的内容计算，其中，对分组中某个条件的计算，可以用IF函数加一层判别。

DATE_FORMAT(date, format)：用于以不同的格式显示日期/时间数据。date参数是合法的日期，format规定日期/时间的输出格式。

```SQL
select 
    DATE_FORMAT(trans_date, '%Y-%m') month, 
    country,
    count(*) trans_count,
    count(if(state = "approved", 1, NULL)) approved_count,
    sum(amount) trans_total_amount,
    sum(if(state = "approved", amount, 0)) approved_total_amount
from Transactions
group by month, country;
```

### 1633 各赛事的用户注册率

方法：先计算出用户的总数，然后对注册表进行分组查询，统计每个分组占用户总数的百分比

```SQL
select contest_id, Round(count(*) / (select count(*) from Users) * 100, 2) percentage
from Register
group by contest_id
order by percentage desc, contest_id asc;
```

### 1173 即时事务配送Ⅰ

方法：分别计算即时订单数量和总的订单数量，两者相除后乘100，取两位小数即可

```SQL
select Round(
    (select count(*) from Delivery where order_date = customer_pref_delivery_date) 
    / (select count(*) from Delivery) * 100
    , 2
) immediate_percentage;
```

### 1211 查询结果的质量和占比

方法：分组查询，对每个字段进行对应计算

```SQL
select 
    query_name, 
    round(avg(rating / position), 2) quality,
    round(count(if(rating < 3, 1, NULL)) / count(*) * 100, 2) poor_query_percentage
from Queries
group by query_name;
```

## 连接

### 1607 没有卖出的买家

方法：一、先查出有在2020年卖出的，然后使用not in查出没有在2020年卖出的。二、用Seller表连接在2020年有卖出的order表，连接后seller_id为NULL的话说明没有在2020年卖出。

取指定年份写法有：
YEAR(date) = 2020
LEFT(date) = '2020'
SUBSTR(date, 1, 4) = '2020'
DATE_FORMAT(date, '%Y') = 2020
date between '2020-01-1' and '2020-12-31'
date > '2019-12-31' and date < '2021-01-01'

```SQL
select seller_name from Seller
where seller_id not in (
    select seller_id from orders where YEAR(sale_date) = 2020
)
order by seller_name
```

```SQL
select s.seller_name
from Seller s 
left join Orders o
on s.seller_id = o.seller_id and YEAR(O.sale_date) = 2020
where o.seller_id IS NULL
order by s.seller_name;
```

### 619 只出现一次的最大数字

方法：一、进行分组查询，用if将数量超过1个的置为NULL，然后再排序取第一个元素。二、先使用子查询找出仅出现一次的数字，然后使用MAX函数找出最大一个

```SQL
select if(count(*) > 1, NULL, num) num
from MyNumbers 
group by num
order by num desc
limit 1;
```

```SQL
select max(num) num 
from (
    select num from MyNumbers
    group by num having count(*) = 1
) tmp;
```

### 1112 每位学生的最高成绩

方法：一、使用窗口函数对每个分组（窗口）取出排名，然后再取第一名。二、两次分组，先按student_id找出乘积最高的列，再按student_id和grade分组找出course_id最小的一个

窗口函数（MySQL8.0之后支持）：在满足某些条件的记录集合上执行的特殊函数，对于每条记录都要在此窗口内执行函数。

* rank()：阶梯排序，如果前两个是并列的1，接下来是第三名
* dense_rank()：连续排序，如果前两个并列1，接下来是第二名
* row_number()：不会出现重复的排序

<窗口函数> over (partition by <分组列名> order by <排序列名>)

```SQL
select student_id, course_id, grade
from (
    select 
        student_id, 
        course_id,
        grade,
        rank() over (partition by student_id order by grade desc, course_id asc) rk
    from Enrollments
) tmp
where rk = 1
order by student_id;
```

```SQL
select student_id, min(course_id) course_id, grade
from Enrollments
where (student_id, grade) in (
    select student_id, max(grade) grade 
    from Enrollments 
    group by student_id
)
group by student_id, grade
order by student_id asc;
```

### 1398 购买了产品A和产品B却没有购买产品C的顾客

方法：在Orders表中对customer_id分组，having中用count或sum判断三个条件

```SQL
select c.customer_id customer_id, c.customer_name customer_name
from Customers c join Orders o on  c.customer_id = o.customer_id
group by customer_id
having 
    count(if(product_name = 'A', 1, NULL)) > 0
    and count(if(product_name = 'B', 1, NULL)) > 0
    and count(if(product_name = 'C', 1, NULL)) = 0
```

### 1440 计算布尔表达式的值

方法：使用两次连接取得对应的值，然后使用case when对三个操作符分支进行判断。

```SQL
select left_operand, operator, right_operand, (
    case 
        when e.operator = '=' and v1.value = v2.value then 'true'
        when e.operator = '>' and v1.value > v2.value then 'true'
        when e.operator = '<' and v1.value < v2.value then 'true'
    else 'false'
    end
) value
from Expressions e 
    join Variables v1 on e.left_operand = v1.name
    join Variables v2 on e.right_operand = v2.name
```

### 1264

方法：先从关系表中用联合查询找出两列中用户1的朋友，然后用找出的朋友id和Likes表进行连接，即可找出朋友们喜欢的页面，最后找出用户1喜欢的页面用not in条件排除掉，即可的到答案

```SQL
select distinct page_id recommended_page from (
    select user2_id friend_id from Friendship where user1_id = 1 
    union 
    select user1_id friend_id from Friendship where user2_id = 1 
) tmp join Likes on friend_id = user_id 
where page_id not in (
    select page_id from Likes where user_id = 1
);
```

### 570 至少有5名直接下属的经历

方法：先用分组选出个数大于等于5的managerId，然后再选出id在这个managerId里的名字

上面一题和这一题可以看出，join on和in 用法常常可以相互替换，比如id in managerId 和 join ... on id = managerId

```SQL
select name from Employee where id in (
    select managerId from Employee 
    group by managerId having count(*) > 4 
);
```

### 1303 求团队人数

方法：先根据team_id进行分组计数作为一个子查询，然后再用原来的表在team_id进行连接，即可给每个用户附上分组的计数

```SQL
select employee_id, team_size from Employee join (
    select team_id, count(*) team_size 
    from Employee 
    group by team_id
) tmp on Employee.team_id = tmp.team_id;
```

### 1280 学生们参加各科测试的次数

方法：注意不参加的学科也要记为0，所以不能忽略Subjects表。先对Students和Subjects求笛卡尔积，再连接Examinations表，然后进行分组查询即可

CROSS JOIN：笛卡尔积

```SQL
select S.student_id, S.student_name, Sb.subject_name, count(E.subject_name) attended_exams
from Students S
    cross join Subjects Sb
    left join Examinations E on E.student_id = S.student_id and E.subject_name = Sb.subject_name
group by S.student_id, Sb.subject_name
order by S.student_id, Sb.subject_name;
```

### 1501 可以放心投资的国家

方法：先用union取Calls的两列将其变为一列，然后和Person和Country表进行连接，最后进行分组查询

```SQL
select distinct C.name country from (
    (select caller_id id, duration from Calls)
    union
    (select callee_id id, duration from Calls)
) tmp 
    join Person P on P.id = tmp.id 
    join Country C on substr(P.phone_number, 1, 3) = country_code
group by C.name
having avg(tmp.duration) > (select avg(duration) from Calls)
```

### 184 部门工资最高的员工

方法：现在部门内查询最高工资，因为可能同时多个有最高工资，而部门内只能查一个最高，所以然后把员工和部门连接，然后用IN语句查询部门名字和工资的关系。

```SQL
select D.name Department, E.name Employee, E.salary Salary
from Employee E
    join Department D on E.departmentId = D.id
where (E.departmentId, E.salary) in (
    select departmentId, max(salary) Salary 
    from Employee
    group by departmentId
);
```

### 580 统计各专业学生人数

方法：先将部门和学生表进行连接，然后对部分进行分组查询。注意count(*)只能统计非空的字段，如果为空无法记为0，该有sum + if将空的设为0

```SQL
select D.dept_name dept_name, sum(if(S.dept_id is null, 0, 1)) student_number
from Department D
    left join Student S on D.dept_id = S.dept_id
group by D.dept_id
order by student_number desc, dept_name asc;
```

### 1294 不同国家的天气类型

方法：用天气表连接国家表，where条件选出有11月份信息的行，进行分组，然后用case when语句对分组平均值进行查询。

```SQL
select 
    C.country_name, (
    case 
        when avg(W.weather_state) <= 15 then "Cold"
        when avg(W.weather_state) >= 25 then "Hot"
    else
        "Warm"
    end
    ) weather_type
from Weather W
    join Countries C on W.country_id = C.country_id
where date_format(W.day, '%Y-%m') = '2019-11'
group by C.country_id;
```

### 626 换座位

方法：用if语句，如果id是偶数，减1；如果id是奇数，加1。当总数为奇数时，再套一层if，只有在id是奇数，且id不等于座位总数时，id加1.

```SQL
select 
    if(id % 2 = 0, 
        id - 1,
        if(id = (select count(*) from Seat), id, id+1)
    ) id, student
from Seat
order by id;
```

### 1783 大满贯数量

方法：将Championships表的各个字段查询并用union all联合成一列，再和Players表进行连接，最后分组查询

union和union all区别：union会将重复的行合并，而union all不会合并重复行

```SQL
select P.player_id, P.player_name, count(id) grand_slams_count 
from (
    (select Wimbledon id from Championships)
    union all
    (select Fr_open  id from Championships)
    union all
    (select US_open  id from Championships)
    union all
    (select Au_open  id from Championships)
) tmp join Players P on tmp.id = P.player_id
group by id
```

### 1164 指定日期的产品价格

方法：

1. 找出所有产品；
2. 找到2019-08-16前所有改动的产品的最新价格
   1. 使用分组max函数找到产品最新修改的时间。使用where查询限制时间小于等于 2019-08-16
   2. 使用 where in子查询，根据 product_id 和 change_date 找到对应的价格：
3. 上面两步已经找到了所有的产品和已经修改过价格的产品。使用 left join 得到所有产品的最新价格，如果没有设置为 10。

```SQL
select t1.product_id, ifnull(t2.new_price,10) price
from (
    select distinct product_id from Products
) t1 left join (
    select product_id, new_price
    from Products
    where (product_id, change_date) in (
        select product_id, max(change_date)
        from Products
        where change_date <= "2019-08-16"
        group by product_id
    )
) t2 on t1.product_id = t2.product_id;
```

## 不等式连接

### 603 连续空余座位

方法：使用自连接，连接后结果是一个笛卡尔积，然后进行筛选，筛选出座位号绝对值相差1，且同时为free的座位id，最后需要对id进行去重

```SQL
select distinct c1.seat_id
from Cinema c1 join Cinema c2
    on abs(c1.seat_id - c2.seat_id) = 1 and c1.free = 1 and c2.free = 1 
order by c1.seat_id

```

### 1731 每位经理的下属员工数量

方法：先用分组选出经理的id、汇报人数以及员工平均年龄，然后和这个表进行内连接，获得经理的名字。

```SQL
select E1.employee_id, E1.name, tmp.reports_count, round(tmp.average_age, 0) average_age
from Employees E1 join (
    select reports_to employee_id, count(*) reports_count, avg(age) average_age
    from Employees
    group by reports_to
) tmp on E1.employee_id = tmp.employee_id
order by E1.employee_id;
```

### 1747 应该被禁止的Leetflex账户

方法：自连接，再加上一系列条件即可，条件是id相等，ip不等，t1登录时间在t2登入和退出时间之间

```SQL
select distinct t1.account_id
from LogInfo t1, LogInfo t2
where t1.account_id = t2.account_id
    and t1.ip_address != t2.ip_address
    and t1.login >= t2.login
    and t1.login <= t2.logout;
```

### 181 超过经理收入的员工

方法：使用自连接，将雇员id和经理id进行连接，然后查询出雇员收入大于经理收入的员工。

内连接有两种写法

```SQL
from t1, t2 where t1.xx = t2.xx ....
from t1 join t2 on t1.xx = t2.xx ....

```

JOIN通常更有效

```SQL
select E1.name Employee
from Employee E1 
    join Employee E2 on E1.managerId = E2.id and E1.salary > E2.salary;
```

```SQL
select E1.name Employee
from Employee E1, Employee E2
where E1.managerId = E2.id and E1.salary > E2.salary;
```

### 1459 矩形面积

方法：采用自连接，能构成矩形的条件是两个点的x和y坐标不相等，再加上前一个id大于后一个id的条件避免重复。

```SQL
select P1.id p1, P2.id p2, abs((P1.x_value - P2.x_value) * (P1.y_value - P2.y_value)) area
from Points P1, Points P2
where P1.x_value != P2.x_value and P1.y_value != P2.y_value and P1.id < P2.id
order by area desc, p1, p2;
```

### 180 连续出现的次数

方法：采用自连接，取三份表，条件为三个表的id连续且数值相等，最后需要去重

```SQL
select distinct L1.Num ConsecutiveNums
from Logs L1, Logs L2, Logs L3
where 
    L1.Id = L2.Id + 1 and L2.Id = L3.Id + 1
    and L1.Num = L2.Num and L1.Num = L3.Num;
```

### 1988 找出每所学习的最低分数要求

方法：题目意思就是要找出capacity大于student_count的最小分数。将这个条件作为Schools左连接Exam的条件，然后对school_id进行分组查询，用ifnull将最小值不存在的置为-1

```SQL
select S.school_id, ifnull(min(E.score), -1) score
from Schools S left join Exam E on S.capacity >= E.student_count
group by S.school_id;
```

## 子查询

### 1549 每件商品的最新订单

方法：一、在订单表先分组选出最大的日期，这样只能选出一条，以此作为子查询外面再包一层选择where ... in，即可选择出重复的最大日期，然后和产品表进行左连接选择出产品名称。 二、使用rank()函数为订单表的日期附上排名作为子查询，选择出名词为1的行，然后和产品表进行连接，选择出产品名

```SQL
select P.product_name, O.product_id, O.order_id, O.order_date 
from Orders O
    left join Products P on O.product_id = P.product_id
where (O.product_id, O.order_date) in (
    select product_id, max(order_date) order_date 
    from Orders
    group by product_id
    )
order by P.product_name, O.product_id, O.order_id;
```

```SQL
select P.product_name, O.product_id, O.order_id, O.order_date
from Products P join (
    select 
        product_id,
        order_id,
        order_date,
        rank() over (partition by product_id order by order_date desc) rk
    from Orders
    ) O on O.product_id = P.product_id
where O.rk = 1
order by P.product_name, O.product_id, O.order_id;
```

### 1321 餐厅营业额的变化增长

方法：先用一个子查询查出每天的营业总额，然后在这个子查询上使用窗口函数，查出每7天窗口的营业总额。最后以此营业总额作为子查询，计算平均值，注意需要用条件选出第7天以及之后的数据，才能完整覆盖7天

窗口函数写法：

```SQL
[你要的操作] OVER ( PARTITION BY  <用于分组的列名>
                    ORDER BY <按序叠加的列名>
                    ROWS <窗口滑动的数据范围> )
```

对于<滑动窗口的数据范围>用来限定[你要的操作] 所运用的数据的范围, 可用如下

```SQL
当前行 - current row
之前的行 - preceding
之后的行 - following
无界限 - unbounded
表示从前面的起点 - unbounded preceding
表示到后面的终点 - unbounded following

比如取当前行和前五行：ROWS between 5 preceding and current row --共6行
```

```SQL
select distinct visited_on, sum_amount amount, round(sum_amount/7, 2) average_amount
from (
    select visited_on,sum(amount) over (order by visited_on rows 6 preceding) sum_amount
    from (
        select visited_on, sum(amount) amount
        from Customer
        group by visited_on
    ) tmp1
) tmp2
where datediff(visited_on, (select min(visited_on) from Customer)) >= 6;
```

### 1045 买下所有产品的客户

方法：根据用户id进行分组，判断分组内的数目是否等于产品表所有产品的数量。注意一点分组count和product查询的count都需要distinct去重

```SQL
select customer_id
from Customer
group by customer_id
having count(distinct product_key) = (select count(distinct product_key) from Product);
```

### 1341 电影评分

方法：两个分别查询，再用union联合，分组后按统计函数排序，取第一个即可

分组后函数可以用在排序处

```SQL
(
    select U.name results 
    from MovieRating M join Users U on M.user_id = U.user_id
    group by M.user_id
    order by count(M.user_id) desc, U.name asc
    limit 1
)
union
(
    select M.title results
    from MovieRating MR join Movies M on MR.movie_id = M.movie_id
    where date_format(MR.created_at, '%Y-%m') = '2020-02'
    group by MR.movie_id
    order by avg(MR.rating) desc, M.title asc
    limit 1
)
```

### 1867

方法：先分组查出每个订单的平均数量，以此作为子查询查出最大的平均数量。然后再对订单id进行分组查询，查询出最大的数量大于最大平均数量的id

```SQL
# Write your MySQL query statement below
select order_id
from OrdersDetails 
    group by order_id
    having max(quantity) >(
        select max(a) 
        from (
            select avg(quantity) a
            from OrdersDetails
            group by order_id
        ) tmp
    )
```

### 550 游戏玩法分析IV

方法：先分组查出每个用户首次的登录时间，然后再和原表进行连接，连接条件为用户id相同且日期相差1天，这样连接部分的表如果不满足条件对应的字段为null，所以可以对用于对连接表的字段进行计算。

```SQL
select round(avg(a.player_id is not null), 2) fraction
from (
    select player_id, min(event_date) event_date
    from Activity
    group by player_id
) b left join Activity a
    on datediff(a.event_date, b.event_date) = 1 and a.player_id = b.player_id;

select round(count(b.player_id) / count(distinct a.player_id), 2) fraction
from Activity a left join (
    select player_id, min(event_date) event_date
    from Activity
    group by player_id
) b on datediff(a.event_date, b.event_date) = 1 and a.player_id = b.player_id;
```

### 262 行程和用户

方法：将Trips表的client_id和driver_id与Users表进行两次连接，连接条件中剔除Users表中被禁的行。然后对日期进行分组查询。

```SQL
select t.request_at Day, round(avg(t.status != "completed"), 2) 'Cancellation Rate'
from Trips t
    join Users u1 on t.client_id = u1.users_id and u1.banned = "NO"
    join Users u2 on t.driver_id = u2.users_id and u2.banned = "NO"
where t.request_at between "2013-10-01" and "2013-10-03"
group by t.request_at;
```
