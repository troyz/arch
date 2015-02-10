
[TOC]
###1, 环境
> CentOS, MySQL, Redis, Nodejs

###2, [Redis](http://redis.io/)简介
Redis是一个开源的K-V内存数据库，它的key可以是string/set/hash/list/...，因为是基于内存的，所在访问速度相当快。

###3, [Gearman](http://gearman.org/)简介
Gearman是一个开源的Map/Reduce分布式计算框架，具有丰富的client sdk，而且它支持MySQL UDF。
####Gearman工作图
![Gearman工作图](http://ww2.sinaimg.cn/large/6403b6a7gw1ep2g6r50qpj20db08egn2.jpg)
####Gearman调用流程
![Gearman调用流程](http://ww2.sinaimg.cn/large/6403b6a7gw1ep2g7o94ozj20da03zt9k.jpg)
####Gearman集群
从图中可以看出貌似Gearman的集群功能比较弱，Job Server之间没啥关系好像。
![Gearman集群](http://ww3.sinaimg.cn/large/6403b6a7gw1ep2g8gv52qj20au04yjs2.jpg)

###4, MySQL - Redis配合使用方案
![MySQL - Redis配合使用方案](http://ww2.sinaimg.cn/large/6403b6a7gw1ep2ge1u4s8j20np0f4js6.jpg)

首先我们以MySQL数据为主，将insert/update/delete交给MySQL，而select交给redis；
当有数据发生变化时，通过MySQL Trigger实时异步调用Gearman的UDF提交一个job给Job Server，当job执行的时候会去更新redis；从而保证redis与MySQL中的数据是同步的。

###4, 软件安装
####安装gearman
```
//安装依赖
$ yum install -y boost-devel gperf libevent-devel libuuid-devel

//下载gearman
$ wget https://launchpad.net/gearmand/1.2/1.1.12/+download/gearmand-1.1.12.tar.gz
$ tar zxvf gearmand-1.1.12.tar.gz

//编译安装，指定mysqlclient的链接路径
$ ./configure LDFLAGS=-L/usr/lib64/mysql/
$ make
$ make install

//启动gearmand服务端
$ /usr/local/sbin/gearmand -l /var/log/gearmand.log -d

//查看是否安装成功，查看gearman版本
$ /usr/local/sbin/gearmand -V
```
####gearman client
[gearman client](http://gearman.org/download/#client__worker_apis)有很多种：PHP, Perl, Python, Ruby, Nodejs, Go, Java, C#...，我们选择[GearmaNode](https://github.com/veny/GearmaNode)(Nodejs库)作为实验用的gearman client

Nodejs的安装省略

> GearmaNode的安装及测试

**安装GearmaNode**
```
//安装GearmaNode
$ npm install gearmanode
```

**启动Gearman Job Server**
```
//确保gearman的job server已经启动, job server默认监听4730端口
$ netstat -nplt | grep 4730

//如果没有，则启动job server（-d表示启动, -l指定log的位置）
$ /usr/local/sbin/gearmand -l /var/log/gearmand.log -d
```

**worker.js (Gearman Job Worker)**
```
var gearmanode = require('gearmanode');
var worker = gearmanode.worker();
//对job client传递过来的字符串执行reverse倒序操作
worker.addFunction('reverse', function (job) {
    job.sendWorkData(job.payload); // mirror input as partial result
    job.workComplete(job.payload.toString().split("").reverse().join(""));
});
```

```
//启动job worker
$ node worker.js
```

**client.js (Gearman Job Client)**
```
var gearmanode = require('gearmanode');
var client = gearmanode.client();
//提交job
var job = client.submitJob('reverse', 'hello world!');
job.on('workData', function(data) {
    console.log('WORK_DATA >>> ' + data);
});
//结果回调函数
job.on('complete', function() {
    console.log('RESULT >>> ' + job.response);
    client.close();
});
```

```
//执行client.js
$ node client.js
//查看控制台打印输出
```
WORK_DATA >>> hello world!
RESULT >>> !dlrow olleh

可以看到gearman安装成功，运行正常。

###5, MySQL UDF + Trigger同步数据到Gearman

####安装lib_mysqludf_json
lib_mysqludf_json可以把MySQL表的数据以json数据格式输出
```
//下载lib_mysqludf_json
$ git clone https://github.com/mysqludf/lib_mysqludf_json.git
$ cd lib_mysqludf_json

//编译
$ gcc $(mysql_config --cflags) -shared -fPIC -o lib_mysqludf_json.so lib_mysqludf_json.c

//拷贝lib_mysqludf_json.so到MySQL的plugin目录
//可以登陆MySQL，输入命令"show variables like '%plugin%'"查看plugin位置
$ cp lib_mysqludf_json.so /usr/lib64/mysql/plugin/

//演示lib_mysqludf_json功能
$ mysql -uname -hhost -ppwd
//首先注册UDF函数
mysql> CREATE FUNCTION json_object RETURNS STRING 
       SONAME "lib_mysqludf_json.so";
//json_array|json_members|json_values函数注册方式与json_object一样.
mysql> use test;
mysql> select * from user_list;
+------+----------+
| NAME | PASSWORD |
+------+----------+
| troy | pwd      |
+------+----------+
mysql> select json_object(name,password) as user from user_list;
+----------------------------------+
| user                             |
+----------------------------------+
| {"name":"troy","password":"pwd"} |
+----------------------------------+
```

####安装gearman-mysql-udf
```
//下载
$ wget https://launchpad.net/gearman-mysql-udf/trunk/0.6/+download/gearman-mysql-udf-0.6.tar.gz
$ tar zxvf gearman-mysql-udf-0.6.tar.gz
$ cd gearman-mysql-udf-0.6

//安装libgearman-devel
$ yum install libgearman-devel

//编译安装
$ ./configure --with-mysql=/usr/bin/mysql_config --libdir=/usr/lib64/mysql/plugin/
$ make && make install

//登录MySQL注册UDF函数
mysql> CREATE FUNCTION gman_do_background RETURNS STRING
       SONAME "libgearman_mysql_udf.so";
mysql> CREATE FUNCTION gman_servers_set RETURNS STRING
       SONAME "libgearman_mysql_udf.so";
//函数gman_do|gman_do_high|gman_do_low|gman_do_high_background|gman_do_low_background|gman_sum注册方式类似，请参考gearman-mysql-udf-0.6/README

//指定gearman job server地址
mysql> SELECT gman_servers_set('127.0.0.1:4730');
```

> 如果出现异常信息：
> ERROR 1126 (HY000): Can't open shared library 'libgearman_mysql_udf.so' (errno: 11 libgearman.so.8: cannot open shared object file: No such file or directory) 
> 表示系统找不到 libgearman.so 文件，一般so都在/usr/local/lib目录下，修改配置文件/etc/ld.so.conf，将/usr/local/lib目录加入进去即可:

```
$ cat /etc/ld.so.conf
include ld.so.conf.d/*.conf
/usr/local/lib
$ /sbin/ldconfig -v | grep gearman*
```

####MySQL Trigger调用Gearman UDF实现同步
```
DELIMITER $$
CREATE TRIGGER user_list_data_to_redis AFTER UPDATE ON user_list
  FOR EACH ROW BEGIN
    SET @ret=gman_do_background('syncToRedis', json_object(NEW.name as 'name', NEW.password as 'password')); 
  END$$
DELIMITER ;
```

####Gearman Nodejs Worker将数据异步复制到redis

[redis](http://redis.io/download)的安装步骤省略
启动redis-server
```
$ redis-server
```


安装redis的Nodejs client:
```
npm install hiredis redis
```

**redis-worker.js** (Gearman Nodejs worker)
```
var redis = require("redis");
var gearmanode = require('gearmanode');
var worker = gearmanode.worker();

//添加gearman函数syncToRedis
//当MySQL表记录更改时，此函数会被调用
worker.addFunction('syncToRedis', function (job) {
    job.sendWorkData(job.payload);
    console.log("-------job.payload: " + job.payload.toString());
    //将字符串转换成json object, 然后调用更新redis
    updateRedis(eval('(' + job.payload.toString() + ')'));
    job.workComplete("Successed!");
});

//些函数只是简单的将MySQL表中的一行的记录按单个字段更新到redis中。可根据实际情况自行扩展
function updateRedis(json)
{
    var client = redis.createClient();
    client.on("error", function (err) {
        console.log("Error " + err);
    });
    for(var key in json)
    {
        client.set(key, json[key], redis.print);
    }
    client.quit();
}
```

启动worder:
```
node redis-worker.js
```

更新MySQL
```
mysql> select * from user_list;
+------+----------+
| NAME | PASSWORD |
+------+----------+
| troy | jane     |
+------+----------+
mysql> update user_list set name='hello', password='world';
```

redis-worker控制台输出：
```
-------job.payload: {"name":"hello","password":"world"}
```

查看redis数据
```
$ redis-cli
127.0.0.1:6379> keys *
1) "password"
2) "name"
127.0.0.1:6379> get name
"hello"
127.0.0.1:6379> get password
"world"
```

可以看到只要MySQL中user_list表的数据被update了，redis中的数据也会被更新。


###6, 参考资料
[Gearman+PHP+MySQL UDF的组合异步实现MySQL到Redis的数据复制](http://avnpc.com/pages/mysql-replication-to-redis-by-gearman)
