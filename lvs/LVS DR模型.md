###1，环境
> VMWare10, CentOS6.3

###2，LVS DR网络规划
![LVS DR网络图](http://ww4.sinaimg.cn/large/6403b6a7gw1ep00fggxfmj20pn0b53zq.jpg)

> 所有机器都只需要一张网卡，给Director的eth0网卡起个别名eth0:1即VIP的值；给RealServer的lo网卡起个别名lo:0，地址指向VIP
> 
> VIP: 虚拟IP，终端用户在浏览器端输入VIP，Director通过lvs调度算法将请求转给Real Server
> 
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
> ip : 192.168.22.128
> mask: 255.255.255.0
> gateway: 192.168.22.2（VMware NAT的网关IP）
> 
> 网卡eth0别名: eth0:1  （别名是通过后面的脚本来配置）
> ip: 192.168.22.222                         (**VIP**)
> mask: 255.255.255.0
> 
> 设置Director路由转发
> net.ipv4.ip_forward=1

####3.3，ipvsadm命令、启动脚本
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

编写一个shell脚本，避免每次都输入命令：
####**director.sh**
<pre><code>
	#!/bin/bash
	#description: Config director
	VIP=192.168.22.222
	RIP1=192.168.22.131
	RIP2=192.168.22.132
	case "$1" in
	start)
	    #**配置别名保存vip, vip只跟自己通信**
	    ifconfig eth0:1 $VIP broadcast $VIP netmask 255.255.255.255 up
	    route add -host $VIP dev eth0:1                  #如果目标是vip，则通过eth0:1出去。
	    echo 1 > /proc/sys/net/ipv4/ip_forward      #打开路由转发
	    /sbin/iptables -F                                          #清空iptables，iptables与ipvsadm不能同时用
	    /sbin/iptables -Z                                          #重置iptables
	    /sbin/ipvsadm -C                                         #清空ipvsadm
	    ipvsadm -A -t $VIP:80 -s wrr
	    ipvsadm -a -t $VIP:80 -r $RIP1:80 -w 1 -g
	    ipvsadm -a -t $VIP:80 -r $RIP2:80 -w 1 -g
	    echo "ipvsadm start ok"
	    ;;
	stop)
	    ifconfig eth0:1 $VIP broadcast $VIP netmask 255.255.255.255 down
	    echo 0 > /proc/sys/net/ipv4/ip_forward
	    /sbin/ipvsadm -C
	    echo "ipvsadm stoped"
	    ;;
	status)
	    isrunning=`ipvsadm -L -n | grep $VIP`
	    if [ ! "$isrunning" ]; then
	        echo "ipvsadm is stoped..."
	    else
	        echo "ipvsadm is running..."
	        ipvsadm -L -n
	    fi
	    ;;
	*)
	   echo "Usage: $0 {start|stop|status}"
	   exit 1
	esac
	exit 0
</code></pre>

> **start**:  sh director.sh start
> **stop**:   sh director.sh stop
> **status**: sh director.sh status

###4，Real Server配置
####4.1, 配置网卡
这里只把Real Server 1的配置列出来，Real Server 2配置步骤与此相同。
> \# setup
> 
> setup之后最终效果如下:
> 网卡eth0: (使用VMware NAT模式连接网络) 
> ip: 192.168.22.131
> mask: 255.255.255.0
> gateway: 192.168.22.2（VMware NAT的网关IP）

####4.2, 安装web服务
安装nginx/apache/...，步骤省略
安装完后启动web服务
> 查看服务是否启动
> \# netstat -tnlp

####4.3, 启动脚本
用脚本完成对lo网卡的别名操作：
####**director.sh**
<pre><code>
	VIP=192.168.22.222
	case "$1" in
	start)
	        /sbin/ifconfig lo down
	        /sbin/ifconfig lo up
	        echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
	        echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
	        echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
	        echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
	        /sbin/ifconfig lo:0 $VIP broadcast $VIP netmask 255.255.255.255 up
	        /sbin/route add -host $VIP dev lo:0
	        echo "realserver start ok"
	        ;;
	stop)
	        /sbin/ifconfig lo:0 down
	        /sbin/route del $VIP >/dev/null 2>&1
	        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_ignore
	        echo 0 > /proc/sys/net/ipv4/conf/all/arp_ignore
	        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_announce
	        echo 0 > /proc/sys/net/ipv4/conf/all/arp_announce
	        echo "realserver stoped"
	        ;;
	status)
	        islothere=`/sbin/ifconfig lo:0 | grep $VIP`
	        isrothere=`netstat -rn | grep "lo" | grep $VIP`
	        if [ ! "$islothere" -o ! "$isrothere" ]; then
	            echo "LVS-DR real server stopped."
	        else
	            echo "LVS-DR real server running."
	        fi
	        ;;
	*)
	       echo "Usage: $0 {start|stop|status}"
	       exit 1
	esac
	exit 0
</code></pre>

> **start**:  sh realserver.sh start
> **stop**:   sh realserver.sh stop
> **status**: sh realserver.sh status

###5, 测试
至此LVS NAT环境搭建完成，我们来测试一下：
> 在浏览器中输入VIP来访问:
> http://192.168.22.222/index.html
> 
> 使用linux提供的并发测试命令来测试lvs调度情况:
> ab -c 100 -n 1000 http://192.168.22.222/index.html
>
> 可以修改lvs调度算法
> ipvsadm -E -t 192.168.22.222:80 -s wrr
> ipvsadm -E -t 192.168.22.222:80 -s rr
> 
> 查看lvs调度情况
> ipvsadm -L -n --stats|rate
