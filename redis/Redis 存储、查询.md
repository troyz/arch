
[TOC]
###数据存储
假设我们在MySQL数据库中有这样一张表：
```
mysql> desc user_info;
```
| Field | Type         | Null | Key | Default | Extra          |
|-------|--------------|------|-----|---------|----------------|
| id    | int(11)      | NO   | PRI | NULL    | auto_increment |
| NAME  | varchar(50)  | YES  |     | NULL    |                |
| age   | int(11)      | YES  |     | NULL    |                |
| email | varchar(50)  | YES  |     | NULL    |                |
| addr  | varchar(100) | YES  |     | NULL    |                |
|-------|--------------|------|-----|---------|----------------|

在redis中, 我们希望可以按照name,age,email(组合)查询, 那么我们的redis存储可以这样规划:
```
hmset user_info.id.{id} name {name} age {age} email {email} addr {addr}
```

###索引
```
sadd user_info.name.{name} {id1} {id2} {id3}
sadd user_info.age.{age} {id2} {id4} {id6}
sadd user_info.email.{email} {id1} {id3} {id9}
```

###查询

####keys模糊查询
```
//模糊查询name中含有keywords的记录
keys user_info.name.*keywords*
```
> [keys](http://redis.io/commands/keys) 支持正则匹配，但是Redis官网建议不要在prod环境使用这个命令，会降低性能。
> 
> Warning: consider KEYS as a command that should only be used in production environments with extreme care. It may ruin performance when it is executed against large databases. This command is intended for debugging and special operations, such as changing your keyspace layout. **Don't** use **KEYS** in your regular application code. If you're looking for a way to find keys in a subset of your keyspace, consider using **SCAN** or sets.

####sets查询
```
//按name查询
smembers user_info.name.{name}

//按age查询
smembers user_info.age.{age}

//按email查询
smembers user_info.email.{email}

//按name, age组合查询
sinter user_info.name.{name} user_info.age.{age}
```

> **注意**
> 上面的查询都是精确 = 查询


####scan查询
[SCAN ](http://redis.io/commands/scan)
: SCAN 迭代当前数据库的key集合.

[SSCAN](http://redis.io/commands/sscan)
: SSCAN 迭代指定Sets key集合的元素.

[ZSCAN](http://redis.io/commands/zscan)
: ZSCAN 迭代指定Sorted Set key集合的元素.

[HSCAN](http://redis.io/commands/hscan)
: HSCAN 迭代指定Hash key集合的field/value.

> 语法:
> **SCAN cursor [MATCH pattern] [COUNT count]**
>
> count 指定返回结果集的数量，默认为10，redis并**不**能保证每次返回的都是count个结果

SCAN是基于游标(cursor)的，每次只返回少量的结果集，所以需要不停迭代cursor，只到cursor返回0，

```
127.0.0.1:6379> scan 0 count 5
1) "10"                                    // 下一次迭代cursor的起始position
2) 1) "Key_6"
   2) "Key_1"
   3) "Key_4"
   4) "Key_10"
   5) "Key_5"
   6) "Key_7"                              // 我们指定count为5，但返回了6条数据
127.0.0.1:6379> scan 10 count 5
1) "11"                                    // 下一次迭代cursor的起始position
2) 1) "Key_8"
   2) "Key_3"
   3) "Key_11"
   4) "Key_9"
   5) "Key_2"
127.0.0.1:6379> scan 11 count 5
1) "0"                                    // 0表示迭代结束
2) 1) "Key_12"
```

> **Match**
> 这里需要注意下match执行的过程：
> 1, redis执行 scan x COUNT [count]
> 2, redis对上面的结果集进行match过滤，然后返回给client端
> 
> 这就意味着，如果你使用了match过滤，则绝大多数情况下，client端每次收到的结果集数量都小于你指定的[count]数量。

```
// match 正则匹配, 返回含有'1'的key集合
127.0.0.1:6379> scan 0 count 5  match *1*
1) "10"
2) 1) "Key_1"
   2) "Key_10"
127.0.0.1:6379> scan 10 count 5  match *1*
1) "11"
2) 1) "Key_11"
127.0.0.1:6379> scan 11 count 5  match *1*
1) "0"
2) 1) "Key_12"
```

```
// SCAN cursor [MATCH pattern] [COUNT count]
127.0.0.1:6379> sadd age.22 id_5 id_7 id_9 id_8 id_11 id_12 id_20
(integer) 7
127.0.0.1:6379> sscan age.22 0 count 5
1) "0"
2) 1) "id_7"
   2) "id_5"
   3) "id_11"
   4) "id_8"
   5) "id_9"
   6) "id_20"
   7) "id_12"
```

```
// HSCAN key cursor [MATCH pattern] [COUNT count]
127.0.0.1:6379> hmset user.id.1 name 'Troy Zhang' age 30 email 'java-koma@163.com' addr 'ChengDu, SiChuan, China' gender 'male' job 'Mobile Developer' phone '110' dept 'WEBOP CD'
OK
127.0.0.1:6379> hscan user.id.1 0
1) "0"
2)  1) "name"
    2) "Troy Zhang"
    3) "age"
    4) "30"
    5) "email"
    6) "java-koma@163.com"
    7) "addr"
    8) "ChengDu, SiChuan, China"
    9) "gender"
   10) "male"
   11) "job"
   12) "Mobile Developer"
   13) "phone"
   14) "110"
   15) "dept"
   16) "WEBOP CD"
```

> **Scan特点:**
> 1, Scan在redis server端是**无状态**的，任何时候你可以在中途中止迭代，而不需要通知redis server
> 2, 多个client可以同时迭代同一个cursor
> 
> **Scan缺点:**
> 1, count不能保证每次返回指定数量的结果集
> 2, 返回的key结果集有可能重复

----


> **总结**
> 在prod环境下, redis建议使用**scan**，而不要使用 **keys** 和 **smembers**
> 
> Since these commands allow for incremental iteration, returning only a small number of elements per call, they can be used in production without the downside of commands like **KEYS** or **SMEMBERS** that may block the server for a long time (even several seconds) when called against big collections of keys or elements.


