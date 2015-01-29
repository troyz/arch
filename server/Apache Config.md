## Apache 配置 ##
### 1，目录访问权限 ###
输入用户名、密码后方可访问home目录下的文件：
##### httpd.conf #####
    <Directory "D:/software/wwwroot/home">
        Options -Indexes MultiViews                #-Indexes 禁止目录索引
        AllowOverride AuthConfig
        Order Deny,Allow
        Allow from all
        Options All
        AllowOverride All
    </Directory>
##### .htaccess #####
在home目录下放置`.htaccess`文件
<pre><code>authtype basic
authname "Please input the invitation code"        #输入框提示信息
authuserfile D:/software/wwwroot/home/.htpasswd    #指定用户名、密码文件存放位置
require valid-user
</code></pre>
### 2，URL重写 ###
    #/index.html转发到/home/index.html
    RewriteRule ^/(about|index).html$ /home/$1.html [L]
    #/resources, /shangjia 目录下所有的请求转发到/home/resources, /home/shangjia目录
    RewriteRule ^/(resources|shangjia|wojia)/(.+)$ /home/$1/$2 [L]
