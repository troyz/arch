
[TOC]

###简介
使用mysql2redis可以非常便捷的将mysql中的数据导出到redis中去, 通常是需要一个select语句即可实现。

###软件安装
```
// 安装apr + apr-util
$ wget http://mirror.bit.edu.cn/apache//apr/apr-1.5.1.tar.gz
$ tar zxvf apr-1.5.1.tar.gz
$ ./configure --prefix=/usr/local/apr
$ make
$ make install

$ wget http://mirror.bit.edu.cn/apache//apr/apr-util-1.5.4.tar.gz
$ cd apr-util-1.5.4
$ ./configure --with-apr=/usr/local/apr
$ make
$ make install

// 安装hiredis, redis的C语言client
$ git clone https://github.com/redis/hiredis
$ cd hiredis
$ make
$ make install

$ git clone https://github.com/dawnbreaks/mysql2redis.git
$ cd mysql2redis
$ make
$ cp lib_mysqludf_redis_v2.so /usr/lib64/mysql/plugin/
```

### 注册mysql2redis UDF
```
CREATE FUNCTION redis_servers_set_v2 RETURNS int SONAME "lib_mysqludf_redis_v2.so";
CREATE FUNCTION redis_command_v2 RETURNS int SONAME "lib_mysqludf_redis_v2.so";
CREATE FUNCTION free_resources RETURNS int SONAME "lib_mysqludf_redis_v2.so";
```

> 如果出现以下异常信息，则关闭SELINUX
> ERROR 1126 (HY000): Can't open shared library 'lib_mysqludf_redis_v2.so' (errno: 2 libhiredis.so.0.12: failed to map segment from shared object: Permission denied)
> 
> 关闭selinux
> // 临时关闭 
> $ setenforce 0
> 
> //永久关闭 
> cat /etc/selinux/config
> SELINUX=disabled

###测试
```
// mysql
// 设置redis服务器地址
mysql> select redis_servers_set_v2("127.0.0.1",6379);
mysql> select * from user_info;
+----+------------+------+-------------------+---------+
| id | NAME       | age  | email             | addr    |
+----+------------+------+-------------------+---------+
|  1 | Troy Zhang |   30 | java-koma@163.com | ChengDu |
+----+------------+------+-------------------+---------+

// 将user_info表的数据更新到redis中
mysql> select redis_command_v2("hmset", concat("user_info:", id), 'name', name, 'age', age, 'email', email, 'addr', addr) from user_info;

// redis
127.0.0.1:6379> keys *
1) "user_info:1"
127.0.0.1:6379> hgetall user_info:1
1) "name"
2) "Troy Zhang"
3) "age"
4) "\x1e\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00"
5) "email"
6) "java-koma@163.com"
7) "addr"
8) "ChengDu"
```


###参考
> https://github.com/dawnbreaks/mysql2redis
> 
> 我的另外一篇文章：
> [Gearman + Nodejs + MySQL UDF异步实现 MySQL 到 Redis 的数据同步](http://www.cnblogs.com/java-koma/p/4280769.html) 