###1，环境
> VMWare10, CentOS6.3

###2，LVS NAT网络规划
![LVS NAT网络图](http://ww3.sinaimg.cn/large/6403b6a7jw1eoymgazu84j20qt07v3z8.jpg)
可以看到Director机器有2个IP，也就是说需要2张网卡；Real Server只需要一个网卡。
> VIP: 虚拟IP，终端用户在浏览器端输入VIP，Director通过lvs调度算法将请求转给Real Server
> RIP: Real Server的IP

###3，Director配置
####3.1，lvs软件安装
> 检查内核是否加载了ipvs模块：
> \# lsmod | grep ip_vs
> 
> 如果没有加载，则手动加载：
> \# modprobe ip_vs
> 
> 安装ipvsadm
> sudo yum install ipvsadm

####3.2，配置网卡
> \# setup
> 
> setup之后最终效果如下：
> 
> 网卡eth0:                           (使用VMware NAT模式连接网络)
> ip : 192.168.22.128            (**VIP**)
> mask: 255.255.255.0
> gateway: 192.168.22.2（VMware NAT的网关IP）
> 
> 网卡eth1:                           (使用VMware vmnet2模式连接网络)
> ip: 10.0.0.1                         (**DIP**)
> mask: 255.255.255.0
> 
> 设置Director路由转发
> net.ipv4.ip_forward=1

####3.3，ipvsadm添加lvs规则
> ipvsadm命令
> 
> 1, 添加/编辑集群服务
> -A|E                   -t|u|f                vip:port               -s SCHEDULE_METHOD
> A|E: 添加|编辑
> -t|u|f: tcp|udp|firewall
> vip:port: 集群服务
> -s: 指定调度算法（默认wlc）wlc | wrr | rr
> 
> 2, 删除集群服务
> -D                    -t|u|f                vip:port
> 
> 3, 添加/编辑Real Server
> -a|e                  -t|u                  vip:port            -r rip:port         -g|i|m                   -w weight
> -a|e: 添加|修改realserver
> -t|u: tcp|udp
> vip:port 同上
> -r rip:port: 指定Real Server的ip及服务端口
> -g|i|m: 指定lvs模型, g: 默认gateway直接路由(director routing); i: internet turn隧道; m: nat地址伪装
> -w weight: 指定此Real Server的权重
> 
> 4, 删除realserver
> -d                  -t|u                  vip:port            -r rip:port
> 
> 5, 查看当前的规则
> -L      -n       --stats/rate
>
> 6, 清空规则
> -C
> 
> 7, 保证机器重启依然生效
> service ipvsadm save

针对我们上面的网络规划，我们需要这样配置
> ipvsadm -A -t 192.168.22.128:80 -s rr
> ipvsadm -a -t 192.168.22.128:80 -r 10.0.0.11:80 -m -w 1
> ipvsadm -a -t 192.168.22.128:80 -r 10.0.0.12:80 -m -w 1

###4，Real Server配置
####4.1, 配置网卡
这里只把Real Server 1的配置列出来，Real Server 2配置步骤与此相同。
> \# setup
> 
> setup之后最终效果如下:
> 网卡eth0: (使用VMware vmnet2模式连接网络) 
> ip: 10.0.0.11               (**RIP**)
> mask: 255.255.255.0
> gateway: 10.0.0.1  (即Director的**Dip**地址)

####4.2, 安装web服务
安装nginx/apache/...，步骤省略
安装完后启动web服务
> 查看服务是否启动
> \# netstat -tnlp

###5, 测试
至此LVS NAT环境搭建完成，我们来测试一下：
> 在浏览器中输入VIP来访问:
> http://192.168.22.128/index.html
> 
> 使用linux提供的并发测试命令来测试lvs调度情况:
> ab -c 100 -n 1000 http://192.168.22.128/index.html
>
> 可以修改lvs调度算法
> ipvsadm -E -t 192.168.22.128:80 -s wrr
> ipvsadm -E -t 192.168.22.128:80 -s rr
> 
> 查看lvs调度情况
> ipvsadm -L -n --stats|rate
