
###读写分离
![RW Splitting](https://raw.githubusercontent.com/troyz/images/master/mysql/MySQL%20%E8%AF%BB%E5%86%99%E5%88%86%E7%A6%BB.png)

> #####在应用端处理
> 
> 1. Spring AbstractRoutingDataSource
> 2. 淘宝MyFox
> 3. [MySQL Replication Connection](http://dev.mysql.com/doc/connector-j/en/connector-j-master-slave-replication-connection.html)

> #####在数据库端处理
>
> 1. MySQL Proxy
> 2. Amoeba（不支持事务）

###水平拆分表
![MySQL水平拆分](https://raw.githubusercontent.com/troyz/images/master/mysql/MySQL%E6%8B%86%E5%88%86%E6%95%B0%E6%8D%AE%E8%A1%A8.png)

