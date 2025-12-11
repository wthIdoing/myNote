# LNMP架构总结（未完）

学习了LNMP架构并且将其拆分以后，一步到位的做法

## WEB服务器需要做的事情

### 下载Nginx服务，配置Nginx设置

配置Nginx仓库，并且下载Nginx服务

```bash
vim /etc/yum.repos.d/nginx.repo
[nginx-stable]		
name=nginx stable repo		
baseurl=http://nginx.org/packages/centos/7/$basearch/	
gpgcheck=0					
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key		
```

说文解字环节：

```bash
[nginx-stable]		# nginx稳定版
name=nginx stable repo		# 仓库名称
baseurl=http://nginx.org/packages/centos/7/$basearch/	#  下载地址，由于麒麟系统限制，centos后改为centos版本号
gpgcheck=0					# 校验数据包的完整性
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key		# 官网上的key值，gpgcheck为0，可以删除此行
```

#### 	安装Nginx

```bash
yum -y install nginx
```

#### 	查看版本

```bash
nginx -v
```

#### 	修改Nginx的配置文件

```nginx
vim /etc/nginx/nginx.conf
user  nginx;				# 启动nginx的用户，安装Nginx服务会自动创建该用户
worker_processes  auto;		# 进程数量（自动则与CPU核心数量有关）

error_log  /var/log/nginx/error.log notice;	# 错误日志的位置
pid        /var/run/nginx.pid;				# PID的位置


events {
    worker_connections  1024;				# 每个进程的最大连接数
}


http {										# http响应用户的请求
    include       /etc/nginx/mime.types;	# nginx服务支持的资源类型
    default_type  application/octet-stream;	# 如果请求的资源ngin不支持，默认为下载

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
        '$status $body_bytes_sent "$http_referer" '
        '"$http_user_agent" "$http_x_forwarded_for"';		# 日志格式

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;		# 文件高效传输
    #tcp_nopush     on;

    keepalive_timeout  65;	# 长连接的超时时间

    #gzip  on;				# 是否压缩资源传给用户（等级1-9）
    client_max_body_size 0;		# 1次请求默认上传1M文件，设置为0关闭限制,不设置可能会出现413错误
    include /etc/nginx/conf.d/*.conf;	# 将conf.d下所有的.conf结尾包含进当前配置
}
```

#### 	Nginx的日志格式

```nginx
log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';	

$remote_addr			# 客户端的IP地址
$remote_user			# 客户登录用户名
$[time_local]			# 本地时间
$request				# 请求服务器上具体的什么资源
$status					# 状态码
$body_bytes_sent		# 请求资源的大小
$http_referer			# 来源的页面
$http_user_agent		# 记录客户端的浏览器和操作系统的信息
$http_x_forwarded_for	# 在有均衡负载的情况下记录客户端的真实IP地址
```

#### 	检查Nginx配置语句是否正确

```bash
nginx -t
```

#### 	启动Nginx

```bash
systemctl start nginx
systemctl enable nginx
```

#### 	检查Nginx服务

查看有无应用占用80端口

```bash
netstat -tnulp
```

### 下载PHP服务，配置PHP设置

#### 	安装PHP

```bash
yum -y install php php-bcmath php-cli php-common php-devel php-embedded php-fpm php-gd php-intl php-mbstring php-mysqlnd php-opcache php-pdo php-process php-xml php-json
```

#### 	验证是否下载15个包

```bash
rpm -qa | grep php | wc -l
```

#### 	配置PHP服务，修改默认启动用户

```bash
ll /etc/php.ini /etc/php-fpm.d/www.conf
```

#### 	修改启动用户

创建一个统一的用户，避免代码执行时2个服务管辖的文件属主属组出现冲突

```bnash
useradd -s /sbin/nologin -M www
```

#### 	修改Nginx的启动用户

```bash
vim /etc/nginx/nginx.conf
```

```bash
2: user		www;
```

#### 	修改PHP的启动用户

```bash
vim /etc/php-fpm.d/www.conf
```

```bash
...
 24 user = www
 25 ; RPM: Keep a group allowed to write in log dir.
 26 group = www
...
```

#### 	修改PHP的监听方式

```bash
vim /etc/php-fpm.d/www.conf
```

```bash
38:listen = 127.0.0.1:9000
```

#### 	检查PHP配置语法是否正确

```bash
php-fpm -t
```

#### 	启动PHP服务

```bash
systemctl start php-fpm
systemctl enable php-fpm
```

#### 	检查PHP服务

```bash
netstat -tnulp		# 查看有无进程占用127.0.0.1:9000
```

#### 下载MariaDB（不是mariadb-server），需要使用mysql语句进行远程连接

```bash
yum -y install mariadb		# mariadb-server是服务端提供的数据库服务
```

## 数据库服务器需要做的事情

### 配置MySQL数据库

#### 	安装MariaDB并启动服务

```bash
yum -y install mariadb-server
systemctl start mariadb
systemctl enable mariadb
```

### 	在MySQL数据库中创建用于远程连接的用户并授权

```bash
mysql -uroot		# 默认是没有密码的
```

```mysql
grant all on *.* to remote_user@'%' identified by 'password';	# 创建用户设置密码并授权
flush privileges;		# 刷新系统权限相关表
```

## NFS服务器需要做的事情

#### 	下载服务

```bash
yum -y install nfs-utils
```

#### 	配置文件

同样地，这里如果需要配置rsync服务，需要将服务用户统一为www，所以可以事先创建一个www用户

```bash
useradd -s /sbin/nologin -M www
```

指定对外开放的共享文件夹

```bash
vim /etc/exports
```


```bash
/data 172.16.1.0/24(rw,sync,all_squash,anonuid=1000,anongid=1000)
```

#### 创建文件夹并修改属主属组

```bash
mkdir -p /data
chown -R www.www /data
```

#### 启动服务

```bash
systemctl start nfs
systemctl enable nfs
```

#### 检查服务的配置文件是否正确

```bash
cat /var/lib/nfs/etab
# 返回结果
/data 172.16.1.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=65534,anongid=65534,sec=sys,rw,secure,root_squash,all_squash)
```

# 这里将以WordPress为例，示范如何部署服务

先写Nginx配置


```bash
vim /etc/nginx/conf.d/wp.conf
```

```nginx
server {
    listen 80;
    server_name www.wp.com;
    root /code/wordpress;

    location / {
        index index.php index.html;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

}
```

创建代码目录

```bash
mkdir -p /code/wordpress
```

在/code/wordpress中下载代码，如果不是下载代码，则需要将代码文件上传至服务器

```bash
cd /code/
wget https://cn.wordpress.org/wordpress-6.0-zh_CN.tar.gz
tar -xf wordpress-6.0-zh_CN.tar.gz
```

修改代码文件夹的属主属组

```bash
chown -R www.www /code
```

配置好wp.conf后应该检查语法是否正确并重启Nginx服务

```bash
nginx -t
systemctl restart nginx
```

做好DNS解析后，访问www.wp.com

![image-20251208214320519](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251208214320519.png)

![image-20251208214949367](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251208214949367.png)

这里可以先使用mysql语句测试数据库是否可以远程连接

```bash
mysql -hdb01 -uremote_user -p123456		# 这里我创建的用户是remote_user，翻译过来就是远程用户
```

因为服务需要事先创建一个数据库，所以我们需要去DB01的数据库中创建一个

```mysql
create database wordpress;
show databases;		# 查看所有数据库
```

```mysql
# 可以看到返回结果
MariaDB [(none)]> create database wordpress;
Query OK, 1 row affected (0.000 sec)

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| wordpress          |
+--------------------+
4 rows in set (0.000 sec)

```

接下来是运行安装程序，点击开始就好

![image-20251208215252318](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251208215252318.png)

此时WEB01尚未挂载共享文件，也没有创建静态数据，可以看到程序文件夹中并没有upload文件夹

![image-20251208215633713](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251208215633713.png)

只有在网页上要发布文章的时候才会自动创建一个upload文件夹（这里是上网页放入了图片还没发）

![image-20251208215825703](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251208215825703.png)

![image-20251208215748171](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251208215748171.png)

没测试直接上传了……

![image-20251208220030408](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251208220030408.png)

测试下一张

![image-20251208220106564](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251208220106564.png)

这里并没有发布，只是上传图片，就已经将图片上传至程序文件夹中了

#### 那么我们应该将这些数据scp至NFS共享文件夹，再挂载共享文件夹至WEB01

同步数据

```bash
scp -R /code/wordpress/wp-content/uploads/* nfs:/data/wp
```

挂载共享文件夹

```bash
mount -t nfs nfs:/data/wp /code/wordpress/wp-content/uploads/
df -h	# 查看是否挂载
```

![image-20251208220442573](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251208220442573.png)

可以看到已经挂载成功

```java
//TODO		接下来部署WEB02查看是否共用共享文件夹
```

将DNS解析指向WEB02

![image-20251209085045116](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251209085045116.png)

![image-20251209085313165](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251209085313165.png)

测试成功
