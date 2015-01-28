MySQL优化
===============
[TOC]
1,定位慢查询
--------------------------
增、删、改10%，查询90%

>**数据库引擎**
>
> - **MyISAM**：不支持事务，用于只读程序提高性能
> - **InnoDB**：支持ACID事务、行级锁、并发
> - Berkeley DB：支持事务

<pre><code>//设置命令结束符
DELIMITER $$

//创建表
CREATE TABLE emp
(
    empno MEDIUMINT UNSIGNED NOT NULL DEFAULT 0,
    ename VARCHAR(20) NOT NULL DEFAULT '',
    job VARCHAR(9) NOT NULL DEFAULT '',
    mgr MEDIUMINT UNSIGNED NOT NULL DEFAULT 0,
    hiredate DATETIME NOT NULL,
    sal DECIMAL(7,2) NOT NULL,
    comm DECIMAL(7,2) NOT NULL,
    deptno MEDIUMINT UNSIGNED NOT NULL DEFAULT 0
)ENGINE=MYISAM DEFAULT CHARSET=utf8;

//获取随机字符串的存储函数
CREATE FUNCTION rand_string(len INT) 
RETURNS VARCHAR(255)
BEGIN
    DECLARE char_str VARCHAR(100);
    DECLARE return_str VARCHAR(255);
    DECLARE i INT;
    SET char_str = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    SET return_str = '';
    SET i = 0;
    WHILE i &lt len DO
	SET return_str = CONCAT(return_str, SUBSTRING(char_str, FLOOR(1+RAND()*62), 1));
	SET i = i + 1;
    END WHILE;
    RETURN return_str;
END

//获取随机4位整数的存储函数
CREATE FUNCTION rand_int()
RETURNS INT(4)
BEGIN
    DECLARE i INT DEFAULT 0;
    SET i = FLOOR(1 + RAND() * 10000);
    RETURN i;
END$$

//批量插入数据的存储过程
CREATE PROCEDURE insert_emp(IN _start INT(10), IN _max_num INT(10))
BEGIN
    DECLARE i INT DEFAULT 0;
    SET i = 0;
    SET autocommit=0; //设置事务不自动提交，但如果是MyISAM（不支持事务）数据库则无效。
    WHILE i &lt _max_num DO
	SET i = i + 1;
	INSERT INTO emp VALUES(_start + i, rand_string(6), 'salesman', 0001, CURDATE(), 2000, 400, rand_int());
    END WHILE;
    COMMIT;
END$$

//调用存储过程插入5000000条数据
call insert_emp(1, 5000000)$$
</code></pre>

### MySQL命令
|  SQL命令      |   含义  |
| :------------- | :-----|
| show [session \| global] status like 'com_insert%' | 显示执行了多少条insert语句 |
| show [session \| global] status like 'com_select%' | 显示执行了多少条select语句 |
| show status like 'uptime' | 显示数据库启动多长时间了 |
| show [session \| global] status like 'slow_queries' | 显示慢查询（默认情况下大于10s是慢查询） |
| show [session \| global] variables like 'long_query_time' | 显示慢查询的时间，默认10s |
| SET [session \| global] long_query_time=0.5; | 设置慢查询时间为0.5s （注意，session关闭后会恢复到之前的默认值） |

###定位慢查询
开启慢查询日志，一旦开启后，日志文件的位置在my.ini中找<kbd>datadir</kbd>，默认情况下是不会记录慢查询的。
>  1，关闭当前MySQL服务
>			net stop mysql

> 2，启动MySQL并开启慢查询[^startup]
> 			mysqld --defaults-file=my.ini  --long_query_time=0.5  --slow-query-log --log-output=FILE

> 3, 关闭安全模式
>          mysqladmin -uroot -p** shut down

或者，如果不想重启数据库，可以在MySQL中执行以下命令，但是对旧的session无效，对新的session才有效。
> SET GLOBAL long_query_time=0.5;
SET GLOBAL slow_query_log='ON';
SET GLOBAL log_output='FILE';
SET GLOBAL log_slow_admin_statements='ON';

### 索引
> 1，创建索引
> 
> create index 索引名 on 表名 (字段名);

> 2，查看索引
> 
> show indexes from 表名;

> 3，全文索引
> 
> create table article (
> title varchar(200), 
> body TEXT, 
> **FULLTEXT**(title, body) 
> ) engine=MyISAM charset=utf8;
> 
> 查询全文索引
> 
> select * from article where match(title, body) against('keywords');

> 查看匹配相关度
> 
> select match(title, body) against('keywords') from article;
> 
> **Note: ** 
> 
> FULLTEXT类型只在MyISAM引擎中有，InnoDB中没有，MySQL默认使用InnoDB
> 
> 匹配度超过%50（一半的表数据）的关键字，是不会做全文索引的，这个查询关键字keywords叫停止词（这个特点告诉我们，要做全文索引必须是海量数据）
> 
> FULLTEXT不支持中文，需要用<kbd>sphinx</kbd>技术去处理中文

> 4，唯一索引
> 
> create table stu(
> stuid int primary key,
> name varchar(10) **unique**,
> email varchar(50));
> 
> create **unique** index 索引名 on 表名 (字段名)
> 
> **Note:**
> 
> 主键与唯一索引的区别：
> 
> 主键索引不能重复、不能为空
> 
> 唯一索引不能重复、可以为空

### explain分析SQL语句
> explain select * from 表名 where 条件 \G; 
> 
> 创建复合索引（按照字段1查询时索引被使用，按字段2查询时索引不会被使用）
> 
> alter table 表名 add index 索引名 (字段1, 字段2)
> 
> **Note:**
> 
> 如果用主键去查询，自动会使用主键索引
> 
> 如果创建的是复合索引，只有左边的可以用，右边的不管用。（不要用组合索引）
> 
> 在模糊查询时，如果like '%keywords'索引不被使用；like 'keywords%'索引被使用
> 
> 在条件语句中使用or，or两边的字段都必须要有索引，有一个没有，索引就无法使用。
> 
> 如果字段是字符型的，查询时必须有''引起来，索引才能使用。

###选择MySQL存储引擎
**MyISAM**

: 不支持事务处理

: 查询和添加效率高

: 碎片多

: 论坛

> 碎片整理
> 
> optimize table 表名
 
**InnoDB**

: 支持事务

: 保存比较重要的信息
  
**Memory**

: 数据频繁更改，而且不需要在数据库中永久保存

: Sesion入库


[^startup]: MySQL5.6启动参数

http://dev.mysql.com/doc/refman/5.6/en/server-options.html

http://dev.mysql.com/doc/refman/5.6/en/log-destinations.html

http://dev.mysql.com/doc/refman/5.6/en/mysqld-option-tables.html