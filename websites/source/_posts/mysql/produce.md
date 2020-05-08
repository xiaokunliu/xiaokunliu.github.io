---
title: mysql存储过程案例
category: MySQL技术
date: 2018-04-06 20:47:28
tags: MySQL技术
---

<!-- more -->

本文介绍关于在MySQL存储过程游标使用实例，包括简单游标使用与游标循环跳出等方法
> 例1、一个简单存储过程游标实例

```sql
DROP PROCEDURE IF EXISTS getUserInfo $$
CREATE PROCEDURE getUserInfo(in date_day datetime)
--
-- 实例
-- 存储过程名为：getUserInfo
-- 参数为：date_day日期格式:2008-03-08
--
BEGIN
declare _userName varchar(12); -- 用户名
declare _chinese int ; -- 语文
declare _math int ;    -- 数学
declare done int;
-- 定义游标 www.jbxue.com
DECLARE rs_cursor CURSOR FOR SELECT username,chinese,math from userInfo where datediff(createDate, date_day)=0;
DECLARE CONTINUE HANDLER FOR NOT FOUND SET done=1;
-- 获取昨天的日期
if date_day is null then
   set date_day = date_add(now(),interval -1 day);
end if;
open rs_cursor;
cursor_loop:loop
   FETCH rs_cursor into _userName, _chinese, _math; -- 取数据

   if done=1 then
    leave cursor_loop;
   end if;
   -- 更新表
   update infoSum set total=_chinese+_math where UserName=_userName;
end loop cursor_loop;
close rs_cursor;

    END$$
DELIMITER ;
```

> 例2、存储过程游标循环跳出现
在MySQL的存储过程中,游标操作时,需要执行一个conitnue的操作.众所周知,MySQL中的游标循环操作常用的有三种,LOOP,REPEAT,WHILE.三种循环,方式大同小异.以前从没用过,所以记下来,方便以后查阅.

1. REPEAT

```sql
REPEAT
    Statements;
  UNTIL expression
END REPEAT
demo
DECLARE num INT;
DECLARE my_string  VARCHAR(255);
REPEAT
SET  my_string =CONCAT(my_string,num,',');
SET  num = num +1;
  UNTIL num <5
END REPEAT;
```

2. WHILE

```sql
WHILE expression DO
    Statements;
END WHILE
demo
DECLARE num INT;
DECLARE my_string  VARCHAR(255);
SET num =1;
SET str ='';
  WHILE num  < span>10DO
SET  my_string =CONCAT(my_string,num,',');
SET  num = num +1;
END WHILE;
```

3. LOOP(这里面有非常重要的ITERATE,LEAVE)

```sql
DECLARE num  INT;
DECLARE str  VARCHAR(255);
SET num =1;
SET my_string ='';
  loop_label:  LOOP
IF  num <10THEN
      LEAVE  loop_label;
ENDIF;
SET  num = num +1;
IF(num mod3)THEN
      ITERATE  loop_label;
ELSE
SET  my_string =CONCAT(my_string,num,',');
ENDIF;
END LOOP;
# PS:可以这样理解ITERATE就是我们程序中常用的contiune,而ITERATE就是break.当然在MySQL存储过程,需要循环结构有个名称,其他都是一样的.
```

> 例3,mysql 存储过程中使用多游标
 
* 先创建一张表，插入一些测试数据：

```sql
DROP TABLE IF EXISTS netingcn_proc_test;
CREATE TABLE `netingcn_proc_test` (
  `id` INTEGER(11) NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(20),
  `password` VARCHAR(20),
  PRIMARY KEY (`id`)
)ENGINE=InnoDB;

insert into netingcn_proc_test(name, password) values
('procedure1', 'pass1'),
('procedure2', 'pass2'),
('procedure3', 'pass3'),
('procedure4', 'pass4');

# 下面就是一个简单存储过程的例子：
drop procedure IF EXISTS test_proc;
delimiter //
create procedure test_proc()
begin
 -- 声明一个标志done， 用来判断游标是否遍历完成
 DECLARE done INT DEFAULT 0;
 -- 声明一个变量，用来存放从游标中提取的数据
 -- 特别注意这里的名字不能与由游标中使用的列明相同，否则得到的数据都是NULL
 DECLARE tname varchar(50) DEFAULT NULL;
 DECLARE tpass varchar(50) DEFAULT NULL;
 -- 声明游标对应的 SQL 语句
 DECLARE cur CURSOR FOR
  select name, password from netingcn_proc_test;
 -- 在游标循环到最后会将 done 设置为 1
 DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
 -- 执行查询
 open cur;
 -- 遍历游标每一行
 REPEAT
  -- 把一行的信息存放在对应的变量中
  FETCH cur INTO tname, tpass;
  if not done then
   -- 这里就可以使用 tname， tpass 对应的信息了
   select tname, tpass;
  end if;
  UNTIL done END REPEAT;
 CLOSE cur;
end
//
delimiter ;
```

* 执行存储过程(第一种情况)  

```sql 
call test_proc();

#  需要注意的是变量的声明、游标的声明和HANDLER声明的顺序不能搞错，必须是先声明变量，再申明游标，最后声明HANDLER。上述存储过程的例子中只使用了一个游标，那么如果要使用两个或者更多游标怎么办，其实很简单，可以这么说，一个怎么用两个就是怎么用的。例子如下：
drop procedure IF EXISTS test_proc_1;
delimiter //
create procedure test_proc_1()
begin
 DECLARE done INT DEFAULT 0;
 DECLARE tid int(11) DEFAULT 0;
 DECLARE tname varchar(50) DEFAULT NULL;
 DECLARE tpass varchar(50) DEFAULT NULL;
 DECLARE cur_1 CURSOR FOR
  select name, password from netingcn_proc_test;
 DECLARE cur_2 CURSOR FOR
  select id, name from netingcn_proc_test;
 DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
 open cur_1;
 REPEAT
  FETCH cur_1 INTO tname, tpass;
  if not done then
   select tname, tpass;
  end if;
  UNTIL done END REPEAT;
 CLOSE cur_1;
 -- 注意这里，一定要重置done的值为 0
 set done = 0;
 open cur_2;
 REPEAT
  FETCH cur_2 INTO tid, tname;
  if not done then
   select tid, tname;
  end if;
  UNTIL done END REPEAT;
 CLOSE cur_2;
end
//
delimiter ;
call test_proc_1();
```

* 执行存储过程(第二种情况)  
```sql
# 上述代码和第一个例子中基本一样，就是多了一个游标声明和遍历游标。这里需要注意的是，在遍历第二个游标前使用了set done = 0，因为当第一个游标遍历玩后其值被handler设置为1了，如果不用set把它设置为 0 ，那么第二个游标就不会遍历了。当然好习惯是在每个打开游标的操作前都用该语句，确保游标能真正遍历。当然还可以使用begin语句块嵌套的方式来处理多个游标,例如：
drop procedure IF EXISTS test_proc_2;
delimiter //
create procedure test_proc_2()
begin
 DECLARE done INT DEFAULT 0;
 DECLARE tname varchar(50) DEFAULT NULL;
 DECLARE tpass varchar(50) DEFAULT NULL;
 DECLARE cur_1 CURSOR FOR
  select name, password from netingcn_proc_test;
 DECLARE cur_2 CURSOR FOR
  select id, name from netingcn_proc_test;
 DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
 open cur_1;
 REPEAT
  FETCH cur_1 INTO tname, tpass;
  if not done then
   select tname, tpass;
  end if;
  UNTIL done END REPEAT;
 CLOSE cur_1;
 begin
  DECLARE done INT DEFAULT 0;
  DECLARE tid int(11) DEFAULT 0;
  DECLARE tname varchar(50) DEFAULT NULL;
  DECLARE cur_2 CURSOR FOR
   select id, name from netingcn_proc_test;
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
  open cur_2;
  REPEAT
   FETCH cur_2 INTO tid, tname;
   if not done then
    select tid, tname;
   end if;
   UNTIL done END REPEAT;
  CLOSE cur_2;
 end;
end
//
delimiter ;
call test_proc_2();
```
