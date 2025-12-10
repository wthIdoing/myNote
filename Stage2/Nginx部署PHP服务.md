# Nginx部署PHP服务

服务需要安装MySQL和PHP系列的组件

### MariaDB数据库的安装与配置

##### 安装MariaDB数据库

```bash
yum -y install mariadb-server
```

##### 启动MariaDB服务

```bash
systemctl start mariadb
syatemctl enable mariadb
```

##### 配置密码

```bash
mysqladmin password '123456'
```

##### 登录MySQL数据库

```bash
mysql -uroot -p123456
```

### PHP的安装与配置

#####  安装PHP

```bash
yum -y install php php-bcmath php-cli php-common php-devel php-embedded php-fpm php-gd php-intl php-mbstring php-mysqlnd php-opcache php-pdo php-process php-xml php-json
```

##### 验证是否安装成功15个包

```bash
rpm -qa | grep php | wc -l
```

##### 配置PHP服务，修改默认启动用户

```bash
ll /etc/php.ini /etc/php-fpm.d/www.conf
```

##### 修改启动用户

这里将Nginx与PHP服务的用户统一为www，于是直接创建一个新的虚拟用户

```bash
useradd -s /sbin/nologin -M www
```

###### 修改Nginx启动用户

```bash
vim /etc/nginx/nginx.conf
2:user	www;		# 将第二行的默认用户改为www
```



###### 修改PHP启动用户

```bash
vim /etc/php-fpm.d/www.conf
...
 24 user = www
 25 ; RPM: Keep a group allowed to write in log dir.
 26 group = www
...
```



##### 修改监听方式

```bash
cat /etc/php-fpm.d/www.conf -n

38:listen = 127.0.0.1:9000		# 修改38行监听端口的设置
```

##### 启动PHP服务

```bash
systemctl start php-fpm
systemctl enable php-fpm
```

##### 检查PHP服务

```bash
netstat -tnulp		# 然后查看有无进程占用127.0.0.1:9000端口
php-fpm -t
返回结果：NOTICE: configuration file /etc/php-fpm.conf test is successful
```

### 配置Nginx连接PHP

##### 在Nginx的conf.d路径中写入PHP相关的配置文件，文件名称自取

```nginx
vim /etc/nginx/conf.c/php.conf
server {
        listen 80;
        server_name php.oldboy.com;
        root /code;

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

#### 写一个PHP文件测试是否可以正常解析

```php
vim /codeindex.php
<?php
		phpinfo();
?>
nginx -t
systemctl restart nginx
```

重启Nginx后，在主机配置NDS解析，再访问php.oldboy.com/index.php

##### 配置PHP连接MySQL数据库

```php
vim /code/mysql.php
<?php
	$servername = "localhost";
	$username = "root";
	$password = "123456";
	
	// 创建连接
	$conn = mysqli_connect($servername, $username, $password);
	
	// 检测连接
	if(!$conn){
		die("Connection failed: " . mlysqli_connect_error());
	}
	echo "成功与MySQL数据库建立连接"
?>
```

### 以WordPress举例，部署业务

```nginx
vim /etc/nginx/conf.d/wp.conf
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

需要创建代码目录

```bash
mkdir -p /code/wordpress
```

在/code/wordpress中下载代码

```bash
cd /code/wordpress/
wget https://cn.wordpress.org/wordpress-6.0-zh_CN.tar.gz
tar -xf wordpress-6.0-zh_CN.tar.gz
```

> [!WARNING]
>
> 另：需要将代码源目录的属主属组改为Nginx与PHP的共同属主属组，否则会出现权限冲突的情况

```bash
chown -R www.www /code/wordpress/
```

做个DNS解析并在浏览器上访问www.wp.com

10.0.0.7	www.wp.com

后续在使用WordPress时，需要在MySQL数据库中创建一个数据库

```bash
mysql -uroot -p123456 -e"create database wordpress;"
```



```mysql
mysql -uroot -p123456
create database wordpress;
show databases;		# 可以看到创建的数据库wordpress
```

#### 413错误的出现

> [!CAUTION]
>
> 在使用wordpress时，上传文件出现413Request Entity Too Large

文档中是这么写的：

| Syntax:  | `**client_max_body_size** *size*;` |
| :------- | ---------------------------------- |
| Default: | `client_max_body_size 1m;`         |
| Context: | `http`, `server`, `location`       |

Sets the maximum allowed size of the client request body. If the size in a request exceeds the configured value, the 413 (Request Entity Too Large) error is returned to the client. Please be aware that browsers cannot correctly display this error. Setting `*size*` to 0 disables checking of client request body size.

Nginx服务**单次**请求**默认允许上传1M**大小的文件，如果超过这个大小则返回413错误。

但是将client_max_body_size设置为0可以关闭这个设置。

```bash
vim /etc/nginx/nginx.conf

...
client_max_body_size 20M;		# 加上这一条

    include /etc/nginx/conf.d/*.conf;
}
```

