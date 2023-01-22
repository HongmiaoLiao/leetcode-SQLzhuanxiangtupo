# (专项突破SQL-SQL进阶)

[leetcode页面](https://leetcode.cn/study-plan/sql/?progress=c22eg3t)

[toc]

## 汇总函数

### 1303 求团队人数

方法：先根据团队id分组查出团队id和团队人数，作为一个子查询表，用原表和此子查询表在团队id字段进行内连接，即可查询出员工id和团队人数

```SQL
select e1.employee_id, e2.team_size
from Employee e1 join (
    select team_id, count(*) team_size
    from Employee
    group by team_id
) e2 on e1.team_id = e2.team_id;
```

### 1308 不同性别每日分数统计

方法：一：连接法，将两份Scores表进行连接，连接条件为性别相等且第一份日期大于第二份日期。核心在于把求累和(cumsum)的问题转化为求相同性别, 日期小于等于当前日期的记录的分数之和。二：窗口函数，在窗口的每条记录动态应用聚合函数SUM，可以动态计算在指定的窗口内的累计分数。

求累积和的问题，往往能，一：用一个表分成两份进行连接，连接条件为第一份的一个字段大于等于另一份的相同字段，然后进行对这个字段分组聚合。二：使用窗口函数。

```SQL
select s1.gender, s1.day, sum(s2.score_points) total
from Scores s1, Scores s2
where s1.gender = s2.gender and s1.day >= s2.day
GROUP BY s1.gender, s1.day
ORDER BY s1.gender, s1.day;
```

```SQL
select 
    gender,
    day,
    sum(score_points) over (partition by gender order by day asc) total
from Scores;
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
    join Country C on C.country_code = substr(P.phone_number, 1, 3)
group by C.name 
having avg(tmp.duration) > (select avg(duration) from Calls);
```

## 排序函数

### 1077 项目员工Ⅲ

方法：将项目表内连接员工表后，利用rank()窗口函数对基于项目id分组的经验年数进行排名，以此作为子查询，再查询出排名为1的行。

```SQL
select project_id, employee_id
from (
    select 
        P.project_id, 
        P.employee_id,
        rank() over (partition by P.project_id order by E.experience_years desc) rk
    from Project P join Employee E on E.employee_id = P.employee_id
) tmp
where tmp.rk = 1;
```

### 1549 每件商品的最新订单

方法：使用rank()函数为订单表的日期附上排名作为子查询，选择出名词为1的行，然后和产品表进行连接，选择出产品名

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

### 1285 找到连续区间的开始和结束

方法：如果id和 id大小的排名 相减的值相等，说明这是一组连续的id，可以使用分组函数给id的大小进行排名，然后和id值进行相减，这个结果就是连续值的分组标识，然后以此作为子查询分组查询出最大和最小的id

row_number()：窗口函数，不重复的排名
查找连续值，都可以用  “值 - row_number()”作为连续值的一个分组标识

```SQL
select min(log_id) start_id, max(log_id) end_id
from (
    select
        log_id,
        log_id - row_number() over (order by log_id asc) group_id
    from Logs
) tmp
group by group_id
order by start_id;
```

### 1596 每位顾客最经常订购的商品

方法：先对订单表中的用户id和产品id进行分组，然后利用窗口函数，对分组进行计数和排序，以此作为子查询，和产品表连接，从而查出产品名称

```SQL
select tmp.customer_id, tmp.product_id, P.product_name
from (
    select 
        customer_id,
        product_id,
        rank() over (partition by customer_id order by count(*) desc) rk
    from Orders
    group by customer_id, product_id
) tmp join Products P on tmp.product_id = P.product_id and tmp.rk = 1;
```

### 178 分数排名

方法：一、直接使用内置的dense_rank()排名函数。二、计算不低与当前分数的不重复分数的个数，就是当前的排名。

注意，字段重命名中用到了函数，需要加上引号

```SQL
select
    score,
    dense_rank() over (order by score desc) 'rank'
from Scores;
```

```SQL
SELECT 
    a.score, 
    (
        SELECT Count(DISTINCT b.Score) 
        FROM Scores b 
        where b.Score >= a.Score
    )  'rank'
FROM Scores a ORDER BY a.score DESC;
```

### 177 第N高的薪水

方法：一、先将N设为N-1（因为limit中从0开始），然后进行分组（通过分组去重），根据薪水排序，最后用limit选择出第N个高的薪水（limit和offset字段后面只接受正整数或单一遍历，不能用表达式）。二、使用窗口函数计算出薪水的排名，然后作为子查询查出排名为N的薪水，如果没有查到的话，null是不会返回的，所以需要在外面再包一层带ifnull的选择。

```SQL
CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
  SET N := N - 1;
  RETURN (
      # Write your MySQL query statement below.
      select salary
      from Employee
      group by salary
      order by salary desc
      limit N, 1
  );
END
```

```SQL
CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
  RETURN (
      # Write your MySQL query statement below.
      select ifnull((
        select distinct salary
        from (
            select
                salary,
                dense_rank() over (order by salary desc) rk
            from Employee
        ) tmp
        where tmp.rk = N), 
        null
      )
  );
END
```

### 1951 查询具有最多共同关注者的所有两两结对组

方法：将一个表分成两份，在关注者上进行内连接，并令第一个表的id大于第二个的，这样不会重复。然后对连接的表根据两个id进行分组，再用窗口函数对组内的数量进行排名，最后外包一层选择查询出排名为1的两个id

```SQL
select user1_id, user2_id
from (
    select 
        r1.user_id user1_id, 
        r2.user_id user2_id, 
        rank() over (order by count(*) desc) rk
    from 
        Relations r1 join Relations r2 
        on r1.follower_id = r2.follower_id and r1.user_id < r2.user_id
    group by 
        r1.user_id, 
        r2.user_id
) tmp
where rk = 1;
```

### 1709 访问日期之间的最大的空档期

方法：先使用lead()窗口函数，按照时间排序把当前行时间和下一行的时间找到，作为子查询，再对用户id分组，查找最大的那个间隔

LEAD(col, offset, default)

col - 指你要操作的那一列
offset - 偏移几行，如果是1就是下1行，以此类推
default - 如果下一行不存在，用什么值填充

对于一张表的一行数据而言，在其之上的是Lag, 在其之下的是Lead

```SQL
select user_id, max(datediff(next_date, visit_date)) biggest_window
from (
    select
        user_id,
        visit_date,
        lead(visit_date, 1, "2021-1-1") over (partition by user_id order by visit_date asc) next_date
    from UserVisits
) tmp
group by user_id
order by user_id;
```

## 函数

### 1949 坚定的友谊

方法：1）建立好友表：使用union，将表复制一份，交换两列位置，然后和表拼接，这样就是一边是用户自己，另一边是用户好友。2）寻找共同好友，将好友表根据id进行自连接，这样就能拥有所有组合，然后进行分组查询，根据自连接的两个id进行分组，取分组内数量大于等于3的id。避免重复，条件使第一个id小于第二个id，为了确保选出的两个id本身是朋友，需要判断这两个id是否在好友表中

```SQL
with t as (
    (select user1_id id, user2_id friend_id from Friendship)
    union
    (select user2_id id, user1_id friend_id from Friendship)      
)

select 
    t1.id user1_id,
    t2.id user2_id,
    count(distinct t1.friend_id) common_friend
from t t1 join t t2 on t1.friend_id = t2.friend_id
where t1.id < t2.id 
    and (t1.id, t2.id) in (
        select user1_id, user2_id from Friendship
    )
group by t1.id, t2.id having count(*) >= 3;
```

### 1532 最近的三笔订单

方法：先在订单表中，使用窗口函数对用户id分组，对购买日期排序，作为子表，然后用用户表和这个表进行连接，再查询出排序小于等于3的行

```SQL
select C.name customer_name, O.customer_id, O.order_id, O.order_date
from Customers C join (
    select 
        customer_id, 
        order_id, 
        order_date,
        dense_rank() over (partition by customer_id order by order_date desc) rk
    from Orders
) O on C.customer_id = O.customer_id
where O.rk <= 3
order by customer_name asc, O.customer_id asc, O.order_date desc;
```

### 1126 查询活跃业务

方法：先按事件类型分组查询，查出每个事件的平均发生次数，然后用事件表和平均次数的子查询表进行连接，筛选出发生次数大于平均次数的行，最后对id分组查询，筛选出id出现次数大于2的

```SQL
select 
    e.business_id
from Events e join (
    select event_type, avg(occurences) avg_occ
    from Events
    group by event_type
) tmp on e.event_type = tmp.event_type
where e.occurences > tmp.avg_occ
group by e.business_id having count(*) >= 2;
```

### 1831 每天的最大交易

方法：先用窗口函数，在每天分组的窗口进行排名，注意这里需要用date函数取出同一日期忽略时分秒，以此作为子查询，查询出排名为1的交易

```SQL
select transaction_id
from (
    select 
        transaction_id,
        rank() over (partition by date(day) order by amount desc) rk
    from Transactions
) tmp where rk = 1
order by transaction_id;
```

## 递归、依赖、嵌套

### 1613 找到遗失的ID

方法：采用递归生成1到100的数作为一个表，然后和用户id进行左连接，找出连接后为空的id，同时限制id数值小于最大的用户id。

```SQL
with recursive a as (
    select 1 as num
    union
    select num+1 from a where num < 100
)
select num ids
from a left join Customers c on a.num = c.customer_id
where num < (select max(customer_id) from Customers) 
    and c.customer_id is null; 
```

### 1270 向公司CEO汇报工作的所有人

方法：一、间接关系有两级，可以两次使用子查询，用每一级查询的id作为子查询的条件，将多次查询的行进行联合，再包一层查询，剔除掉boss的id。二、由于boss的管理者id是自己，所以可以使用两次连接，将管理id和员工id作为连接条件，最后条件选出最上层管理者为boss且员工id不为boss的行

```SQL
select employee_id from (
    select employee_id from Employees where manager_id = 1
    union
    select employee_id from Employees where manager_id in (
        select employee_id from Employees where manager_id = 1
    ) 
    union
    select employee_id from Employees where manager_id in (
        select employee_id from Employees where manager_id in (
            select employee_id from Employees where manager_id = 1
        )
    )
) tmp where employee_id != 1;
```

```SQL
select e1.employee_id
from Employees e1 
    join Employees e2 on e1.manager_id = e2.employee_id
    join Employees e3 on e2.manager_id = e3.employee_id
where e1.employee_id != 1 and e3.manager_id = 1
```

### 1369 获取最近第二次的活动

方法：先使用窗口函数给每个人的活动按时间排名，作为子查询，再查出排名为2的活动，然后再查询只有一项活动的用户，两则进行联合。

```SQL
select username, activity, startDate, endDate
from (
    select 
        username,
        activity,
        startDate,
        endDate,
        rank() over (partition by username order by endDate desc) rk
    from UserActivity
) tmp where rk = 2
union
select username, activity, startDate, endDate
from UserActivity
group by username having count(*) = 1;
```

### 1412 查找成绩处于中游的学生

方法：先将成绩表和学生表进行连接，然后使用窗口函数，分别对成绩升序和降序排名，查询出第一名和最后一名，以此作为条件，再查询出连接表中不是第一名和最后一名的学生。

```SQL
select distinct S.student_id, S.student_name
from Exam E join Student S on E.student_id = S.student_id
where E.student_id not in (
    select student_id
    from (
        select 
            student_id,
            rank() over (partition by exam_id order by score asc) asc_rank,
            rank() over (partition by exam_id order by score desc) desc_rank
        from Exam
    ) T where asc_rank = 1 or desc_rank = 1
)
order by S.student_id;
```

### 1972 同一天的第一个电话和最后一个电话

方法：

```SQL
SELECT DISTINCT 
  u1 user_id
FROM (
  SELECT 
      u1, u2, dt
  FROM (
  SELECT # 2. 用户一天内通话行为的时间先后
      u1, u2, DATE(call_time) dt,
      ROW_NUMBER() OVER(PARTITION BY u1, DATE(call_time) ORDER BY call_time asc) rk_asc,
      ROW_NUMBER() OVER(PARTITION BY u1, DATE(call_time) ORDER BY call_time desc) rk_desc
    FROM (
      # 1. 列出用户所有的通话行为(呼出，或接听)
      SELECT caller_id u1, recipient_id u2, call_time FROM Calls
      UNION ALL 
      SELECT recipient_id u1, caller_id u2, call_time FROM Calls) t1) t2 
  WHERE rk_asc = 1 or rk_desc = 1 # 3. 筛选出用户一天内的第一通和最后一通通话
) t3
GROUP BY u1, dt # 4. 按用户日期聚合
HAVING COUNT(DISTINCT u2) = 1 # 5. 筛选出一天内，第一通和最后一通电话都为同一人的记录。
```

### 185 部门工资前三高的所有员工

方法：先利用窗口函数，根据每个部分分组按工资给出排名，以此作为子查询，和部门表连接查出部门id对应的部门名字，条件定为工资排名小于等于3

```SQL
select D.name Department, E.name Employee, E.salary Salary
from Department D join (
    select 
        departmentId,
        name,
        salary,
        dense_rank() over (partition by departmentId order by salary desc) rk
    from Employee
) E on D.id = E.departmentId
where E.rk <= 3;
```

### 1767 寻找没有被执行的任务对

方法：先用递归的方式，生成每个task_id的全部subtask_id。然后用这个表与Executed表进行左连接，连接的表中Executed部分为空的为没有执行的subtask_id;

```SQL
with recursive t(task_id, subtask_id) as (
    select task_id, subtasks_count from Tasks
    union
    select task_id, subtask_id-1 from t where subtask_id-1 > 0
)

select t.task_id, t.subtask_id
from t left join Executed E
    on t.task_id = E.task_id and t.subtask_id = E.subtask_id
where E.task_id is null;
```

### 1384 按年度列出销售总额

方法：不会，实在懒得看别人题解，先跳过了这题

```SQL

```

### 569 员工薪水的中位数

方法：先用窗口函数给每个公司员工薪水排名，统计每个公司员工个数，然后作为子查询，查询出排名大于等于员工个数一半且小于员工个数一半加一的薪水，这样在奇数个查询出一个中位数，偶数个查询出两个中位数。

```SQL
select id, company, salary
from (
    select 
        id, 
        company,
        salary,
        row_number() over (partition by company order by salary) rk,
        count(id) over (partition by company) cnt
    from Employee 
) t where rk >= cnt / 2 and rk <= cnt / 2 + 1;
```

### 571 给定数字的频率查询中位数

方法：使用sum over（order by）窗口函数对数字个数进行正序和逆序累积，当某一数字的正序和逆序累积均泰语整个序列的数字个数的一半时即为中位数

```SQL
select avg(t1.num) median
from (
    select 
        num,
        sum(frequency) over (order by num asc) asc_accumu,
        sum(frequency) over (order by num desc) desc_accumu
    from Numbers
) t1, (
    select sum(frequency) total
    from Numbers
) t2
where t1.asc_accumu >= t2.total/2 and t1.desc_accumu >= t2.total/2;
```

### 1225 报告系统状态的连续日期

方法：先将两个表联合，用窗口函数和subdate()来找到分组指标（因为每天执行一个任务，所以连续的任务用日期减排名是一样的），然后进行分组查询，查出组内最小和最大的值

对于给连续数值分组，可以用数值减排名（或行数）作为分组指标

```SQL
select period_state, min(exec_date) start_date, max(exec_date) end_date
from (
    select 
        period_state,
        exec_date,
        subdate(exec_date, row_number() over (partition by period_state order by exec_date)) diff
    from (
        select 'failed' period_state, fail_date exec_date from Failed
        union all
        select 'succeeded' period_state, success_date exec_date from Succeeded
    ) t1
) t2
where year(exec_date) = 2019
group by period_state, diff
order by start_date;
```

### 1454

方法：使用lead()窗口函数，计算按时间排序后，每一天和这天4行后的时间差，如果时间差为4，说明是连续的。以此作为子查询，查出时间差为4的即可

```SQL
select distinct T.id, A.name
from (
    select 
        id,
        login_date,
        datediff(lead(login_date, 4) over (partition by id order by login_date), login_date) diff
    from logins
    group by id, login_date
) T join Accounts A on T.id = A.id
where T.diff = 4
order by T.id;
```

### 678 学生地理信息报告

方法：一、使用窗口函数，对每个大洲的值进行分类并排序，以其顺序作为连接条件（此时需要数量最多的放在第一个，才能保证进行左连接时能够都覆盖到）.二、使用窗口函数根据大洲分组根据名字排序，作为子查询，再根据排序进行分组，使用case when选出三列，因为用group by对排名分组，所以选择时需要用min或max

```SQL
select America, Asia, Europe
from (
    select 
        name America,
        row_number() over (order by name) rn
    from Student
    where continent = "America"
) t1 left join (
    select 
        name Asia,
        row_number() over (order by name) rn
    from Student
    where continent = "Asia"
) t2 on t1.rn = t2.rn left join (
    select 
        name Europe,
        row_number() over (order by name) rn
    from Student
    where continent = "Europe"
) t3 on t1.rn = t3.rn;
```

```SQL
select
    max(case continent when "America" then name else null end) America,
    max(case continent when "Asia" then name else null end) Asia,
    max(case continent when "Europe" then name else null end) Europe
from (
    select 
        name,
        continent,
        row_number() over (partition by continent order by name) rn
    from Student
) t
group by rn;
```

### 2010 职员招聘人数Ⅱ

方法：使用两次sum() over 窗口函数，第一个用来确定Senior的id，第二个用来确定Junior的id。第一层（最内层）子查询筛选了薪水总和临近70000的Senior的id，第二层子查询，按经验和工资进行排序（也就是会先排已近选好的Senior，然后从升序的Junior工资中从低往高排）。用窗口函数算累计和，可以查出第一层选的id和第二层从低到高总和不超过70000的id。

sum(xxx) over (partition by groupname order by xx) 计算每个组的累计和

```SQL
select employee_id
from (
    select
        employee_id,
        70000 - sum(salary) over (order by experience, salary) last_salary2
    from (
        select
            employee_id,
            experience,
            salary,
            70000 - sum(salary) over (partition by experience order by salary) last_salary1
        from Candidates
    ) t1
    where t1.last_salary1 >= 0
) t2
where t2.last_salary2 >= 0
```
