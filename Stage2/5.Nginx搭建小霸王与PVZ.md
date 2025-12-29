# Nginx搭建小霸王与PVZ

首先去Nginx官网，使用配置官网的仓库

### 配置仓库

> [!TIP]
>
> https://nginx.org/

```bash
vim /etc/yum.repos.d/nginx.repo
```



```properties
[nginx-stable]		# nginx稳定版
name=nginx stable repo		# 仓库名称
baseurl=http://nginx.org/packages/centos/7/$basearch/	#  下载地址，由于麒麟系统限制，centos后改为centos版本号
gpgcheck=0					# 校验数据包的完整性
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key		# 官网上的key值，gpgcheck为0，可以删除此行
```

### 安装Nginx

```bash
yum -y install nginx
```

使用 nginx -v查看版本 

```bash
nginx -v
```

查看nginx的配置文件

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

    include /etc/nginx/conf.d/*.conf;	# 将conf.d下所有的.conf结尾包含进当前配置
}
```

日志格式说明

```bash
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

nginx server配置文件

> [!TIP]
>
> /etc/nginx/nginx.conf中，include /etc/nginx/conf.d/*.conf;指定的文件，需要写在conf.d目录中

> [!WARNING]
>
> 不要忘记每句末尾要加分号！

```nginx
vim /etc/nginx/conf.d/fc_sim.conf
server {
    listen 80;						# 监听80端口
    server_name fs.oldboy.com;		# 根据域名相应

    location / {					# 网址最后省略的/，例如10.0.0.7/
        root /code/fc;					# 指定/访问哪个目录
        index index.html index.htm		# 指定访问的首页
    }
}
```

配置完成后，需要验证语法是否正确

```bash
nginx -t
```

全部工作完成后，启动nginx并配置开机自启

```bash 
systemctl start nginx
systemctl enable nginx
```

检查端口，查看nginx服务是否工作

```bash
netstat -tunlp
```

### Nginx指定的目录

在fc_sim.conf中，我们制定了有用户进行请求时，访问的代码路径，那么我们需要将代码放在那个路径

```bash
mkdir -p /code/fc
```

上传代码后，将文件解压到/code/fc目录下

![image-20251203212809195](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251203212809195.png)