# 均衡负载(LB)的实现与Redis缓存保持会话一致性

均衡负载分为四层均衡负载与七层均衡负载，这里的层数参考的OSI模型

###### 四层与七层的区别

- 四层在传输层进行转发 IP+端口
- 七层在应用层进行转发，使用的是各种协议

| 特性         | 四层代理(L4)              | 七层代理(L7)                     |
| ------------ | ------------------------- | -------------------------------- |
| **工作层级** | 传输层(Layer 4)           | 应用层(Layer 7)                  |
| **转发依据** | IP地址、端口号            | URL、HTTP Header、Cookie、内容等 |
| **性能**     | 高，处理速度快            | 较低，需要解析应用层报文         |
| **功能**     | 简单、透明                | 强大、只能、可定制               |
| **实现方式** | 报文转发(NAT/DR)          | 代理模式（完全代理）             |
| **典型应用** | 游戏、数据库、简单TCP服务 | Web服务(HTTP/HTTPS)              |

但是使用七层代理时，负载均衡服务器向后端服务器发送消息的端口会受到65535的上限，所以在高并发场景下，使用仅负责**报文转发**的四层代理

LVS：四种网络模式

| L4代理模式  | 转发方式                         | 是否解决端口耗尽？                            | 适用场景                                         |
| ----------- | -------------------------------- | --------------------------------------------- | ------------------------------------------------ |
| **NAT模式** | 负载均衡器**替换**源IP和目标IP   | 否，因为它需要进行SNATCH，仍受65535端口限制。 | 后端服务器与LB在不同网络、或不方便配置DR模式时。 |
| **DR模式**  | 负载均衡器**只替换**MAC地址      | 是，因为相应直接回给客户端，LB不参与连接。    | 高并发、高性能场景。                             |
| **TUN模式** | 负载均衡器将数据包**封装**后转发 | 是，与DR类似，但配置更复杂。                  | 后端服务器与LB跨域广域网时。                     |

- 还有一个FULL NAT

## Nginx实现负载均衡

在web01与web02上分别配置相同的静态页面

```bash
vim /etc/nginx/conf.d/www.oldboy.conf
```

```nginx
server {
	listen 80;
	server_name www.oldboy.com;
	root /code/test;
	index index.html;
}
```

检查并重载
```bash
nginx -t
systemctl reload nginx
```

创建对应的页面

```bash
mkdir -p /code/test
echo web01... > /code/test/index.html
```

使用LB(172.16.1.5)服务器配置转发

```bash
vim /etc/nginx/lb.conf
```

```nginx
upstream webs {
	server web01;	# 已经做了DNS解析
	server web02;
}
server {
	listen 80;
	server_name www.oldboy.com;

	location / {
	proxy_pass http://webs;
    # 配置遇到以下状态码则自动访问下一台server
    proxy_next_upstream error timeout http_500 http_502 http_503 http_504;
	proxy_set_header Host $http_host;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_http_version 1.1;

	proxy_connect_timeout 30;
	proxy_send_timeout 60;
	proxy_read_timeout 60;

	proxy_buffering on;
	proxy_buffer_size 32k;
	proxy_buffers 4 128k;
	}
}
```

检查并重载
```bash
nginx -t
systemctl reload nginx
```

## Nginx的调度算法

- 轮询（默认）
- weight加权轮询    weight值越大，分配的访问概率越高
- IP_HASH        根据访问IP的HASH值进行分配，固定的IP访问固定的后端服务器，但是**违背均衡负载**的初衷
- URL_HASH    根据访问的URL的HASH值进行分配，同样的URL定向到同一个后端服务器
- LEAST_CONN最少链接数   哪台服务器链接数少就给哪台服务器分发

加权轮询与服务状态的实例

```nginx
upstream webs {
	server 10.0.0.7 weight=5;	# web01处理5个请求 然后web02处理1个请求
	server 10.0.0.8;
    server 10.0.0.9 backup;
}
```

###### Nginx后端服务状态 

- down  表示不参与调度
- backup  当其他服务器挂掉以后，backup的服务器才会参与调度

## Nginx均衡负载健康检查

### LB服务器上要做的事

首先下载编译过程中会用到的命令

```bash
yum install -y gcc glibc gcc-c++ pcre-devel openssl-devel patch
```

下载与**安装过**的Nginx版本一致的Nginx源码

```bash
wget http://nginx.org/download/nginx-1.26.1.tar.gz
```

下载nginx_upstream_check第三方模块并上传至home目录

```bash
unzip nginx_upstream_check_module-master.zip
tar -xf nginx-1.26.1.tar.gz
```

进入Nginx目录，选择版本号最相近的打补丁

```bash
cd nginx-1.26.1
patch -p1 < ../nginx_upstream_check_module-master/check_1.20.1+.patch
./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --add-module=/root/nginx_upstream_check_module-master --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Wp,-D_FORTIFY_SOURCE=2 -fexceptions -fstack-protector-strong --param=ssp-buffer-size=4 -grecord-gcc-switches -m64 -mtune=generic -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie'
make 
make install
```

### 🔧 1. `patch` 命令

- **作用：** 应用补丁（Patch）。
- **解释：** 在软件开发中，**补丁** 文件（通常以 `.patch` 结尾）包含了对原始源代码文件所做的修改记录。第三方模块的开发者通常会提供一个补丁文件，用于修改 Nginx **核心** 源代码，使其能够识别和调用新的模块。
- **流程：** `patch` 命令会读取这个补丁文件，并自动修改（“打补丁”）您本地的 Nginx 源代码目录中的对应文件。

------

### ⚙️ 2. `./configure` 命令

- **作用：** 配置（Configuration）编译环境。
- **解释：** 这个脚本（通常是一个可执行文件）会检查您的系统环境，确定所有必要的 **依赖项**（如编译器、库文件等）是否就绪。更重要的是，对于 Nginx 来说，您会在这里指定您想要编译进 Nginx 的 **模块**。
  - 例如，在编译 Nginx 时，您会使用参数如 `--add-module=/path/to/third_party_module` 来告诉 `./configure` 脚本将您的第三方模块包含进来。
- **结果：** 成功执行后，它会生成一个或多个 **Makefile** 文件，这些文件包含了告诉 `make` 命令如何编译程序的详细指令。

------

### 🔨 3. `make` 命令

- **作用：** 编译（Compilation）源代码。
- **解释：** `make` 命令读取上一步生成的 **Makefile** 文件，并根据其中的指示，调用编译器（如 GCC）将所有的 **源代码**（包括 Nginx 核心和您的第三方模块）文件转换成计算机可以直接执行的 **二进制** 代码。
- **流程：** 这是整个过程中耗时最长的一步，它将源代码“构建”成最终的可执行程序。

------

### ✅ 4. `make install` 命令

- **作用：** 安装（Installation）编译好的程序。
- **解释：** 这个命令同样依赖于 **Makefile** 文件中的指令。它的作用是将上一步 `make` 编译生成的 **二进制可执行文件**、库文件和配置文件等，复制到系统或指定目录中（例如，Nginx 的可执行文件可能会被复制到 `/usr/local/nginx/sbin/`）。
- **结果：** 执行完成后，新的、包含第三方模块功能的 Nginx 就成功安装并可以使用了。

------

### 🚀 总结流程

从用户的角度来看，整个过程可以概括为：

1. **准备**：`patch`（修改原始代码）
2. **规划**：`./configure`（告诉系统和程序要怎么编译）
3. **建造**：`make`（执行编译，生成程序）
4. **部署**：`make install`（放置程序到正确的位置）

```bash
vim /etc/nginx/conf.d/lb.conf
```

```nginx
upstream webs {
    server 172.16.1.7:80 max_fails=2 fail_timeout=10s;
    server 172.16.1.8:80 max_fails=2 fail_timeout=10s;
    check interval=3000 rise=2 fall=3 timeout=1000 type=tcp;  
    #interval  检测间隔时间，单位为毫秒
    #rise      表示请求2次正常，标记此后端的状态为up
    #fall      表示请求3次失败，标记此后端的状态为down
    #type      类型为tcp
    #timeout   超时时间，单位为毫秒
}
        server_name www.oldboy.com;
        proxy_next_upstream error timeout http_500 http_502 http_503 http_504;

        location / {
        proxy_pass http://webs;
        proxy_set_header Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;

        proxy_connect_timeout 30;
        proxy_send_timeout 60;
        proxy_read_timeout 60;

        proxy_buffering on;
        proxy_buffer_size 32k;
        proxy_buffers 4 128k;
        }

      location /check {	# 当访问www.oldboy.com/check url时则返回状态模块对应的信息
        check_status;
      }

}
```

## Nginx会话保持

#### WEB01上部署phpMyAdmin业务

```bash
vim /etc/nginx/conf.d/admin.conf
```

```nginx
server {
        listen 80;
        server_name www.admin.com;
        root /code/admin;

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

验证并重启nginx

```bash
nginx -t
systemctl restart nginx
```

创建代码目录并上传代码

```bash
mkdir /code/admin
cd /code/admin
unzip phpMyAdmin-5.2.3-all-languages.zip
mv phpMyAdmin-5.2.3-all-languages* .
```

将配置文件拷贝为生效的配置文件

```bash
cp config.sample.inc.php config.inc.php		# 中间有 sample不生效
```

修改数据指向172.16.1.51

```bash
vim /config.inc.php
```

```php
$cfg['Servers'][$i]['host'] = '172.16.1.51';
```

修改存储会话目录的属主属组为Nginx和PHP的启动用户

```bash
chown -R www.www /var/lib/php/session
```

#### WEB02部署相同的phpMyAdmin服务

```bash
scp web01:/etc/nginx/conf.d/admin.conf /etc/nginx/conf.d/
scp -r web01:/code/admin /code/
```

#### 接入负载均衡

##### 在LB服务器上要做的事

```bash
vim /etc/nginx/conf.d/admin.conf
```

```nginx
upstream admin {
    server 172.16.1.7:80;
    server 172.16.1.8:80;
}
server {
	listen 80;
	server_name www.admin.com;

	location / {
	proxy_pass http://admin;
	proxy_set_header Host $http_host;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_http_version 1.1;

	proxy_connect_timeout 30;
	proxy_send_timeout 60;
	proxy_read_timeout 60;

	proxy_buffering on;
	proxy_buffer_size 32k;
	proxy_buffers 4 128k;
	}
}
```

检查并重启Nginx服务

```bash
nginx -t
systemctl restart nginx
```



## 完整的会话丢失过程描述

假设您有两台后端服务器：**Server A** 和 **Server B**，它们位于一个 **负载均衡器 (LB)** 之后，并且它们之间**没有共享会话信息**。

### 步骤 1: 访问首页 (获取登录页面)

1. **浏览器 (Client) 发起请求:** 首次访问 `phpMyAdmin` 的 URL。
2. **负载均衡器 (LB) 转发:** LB 接收请求，并根据其算法（如轮询）将其转发给 **Server A**。
3. **Server A 响应:** Server A 返回登录页面的 HTML 内容。此时，Server A **通常不会**设置会话 Cookie，因为它只是一个静态页面或未涉及状态的操作。
4. **结果:** 浏览器显示登录页面，**目前浏览器中没有有效的 phpMyAdmin 会话 Cookie。**

### 步骤 2: 登录请求 (会话的建立与设置)

1. **浏览器发起请求:** 您输入用户名和密码，点击“登录”按钮，浏览器向 LB 发送一个 **POST** 请求，携带登录数据。
2. **负载均衡器 (LB) 转发:** LB 接收请求，再次根据算法（如轮询），它将请求转发给 **Server B**。
3. **Server B 处理登录:**
   - Server B 接收到登录数据，进行身份验证，**验证成功**。
   - Server B 在其**本地的会话存储**中创建一个新的用户会话（Session），保存用户的登录状态和权限信息。
   - Server B 为此会话生成一个唯一的 **Session ID**（例如 `S_ID_B`）。
   - Server B 在响应头中添加 `Set-Cookie: PHPSESSID=S_ID_B`（或其他名称），将其发送给浏览器。
4. **结果:** Server B 成功处理登录，并指示浏览器跳转到主页（通常是 302 重定向），同时**浏览器保存了 Server B 设置的 Session Cookie**（`S_ID_B`）。

### 步骤 3: 登录后跳转/访问主页 (会话的丢失)

1. **浏览器发起请求:** 浏览器根据 Server B 的重定向指示，或直接访问主页 URL。此时，浏览器会自动携带已保存的 **Session Cookie** (`S_ID_B`) 发起请求。
2. **负载均衡器 (LB) 转发:** LB 接收到带有 `S_ID_B` 的请求，并根据算法，这次将请求转发给 **Server A**。
3. **Server A 验证失败 (会话丢失):**
   - Server A 收到请求和 Cookie `S_ID_B`。
   - Server A 拿着这个 Session ID (`S_ID_B`) 去查找**自己本地的会话存储**。
   - 因为这个会话是由 **Server B** 创建的，**Server A 的存储中没有 `S_ID_B` 对应的用户状态信息**。
   - Server A 无法识别这个会话，认为用户处于**未登录状态**。
   - Server A 强制返回错误信息或**跳转回登录页面**。
4. **结果:** 浏览器重新显示登录页面，尽管您刚刚输入了正确的账户和密码。

### 步骤 4: 循环往复

如果您再次尝试登录（或刷新页面），浏览器将继续带着旧的 Session Cookie (`S_ID_B`，直到它过期或被新的覆盖) 发起请求。LB 再次随机转发。

- **如果转发到 Server A:** Server A 依旧不认识 `S_ID_B`，要求重新登录。
- **如果偶然转发到 Server B:** Server B **可能**会识别 `S_ID_B`，短暂显示主页，但一旦下次请求又被转发到 Server A，会话又会丢失。

**正是这种服务器之间“互相不认识对方创建的会话”的现象，导致了页面持续刷新回登录状态。**

## Redis部署

为了防止会话丢失，应该将不同服务器的session的cookie存放在同一台服务器的Redis缓存中

### 在Redis服务器上要做的事

##### 1.安装Redis

```bash
yum -y install redis
```

##### 2.修改监听IP

```bash
vim /etc/redis.conf
87:bind 127.0.0.1 172.16.1.51	# 这里是临时拿MySQL服务器代替以下，所以主机号是51
```

##### 3.设置Redis密码

```bash
vim /etc/redis.conf
1044:requirepass 123456
```

##### 4.启动Redis

```bash
systemctl start redis
systemctl enable redis
```

### WEB01服务器上要做的事

#### 配置PHP session指向Redis

```bash
vim /etc/php.ini
```

```bash
1222 session.save_handler = redis
1255 session.save_path = "tcp://172.16.1.51:6379?auth=123456"
```

##### 注释www.conf的倒数第3和4行

```bash
vim /etc/php-fpm.d/www.conf
;php_value[session.save_handler] = files		        # 这行
;php_value[session.save_path]    = /var/lib/php/session # 这行
php_value[soap.wsdl_cache_dir]  = /var/lib/php/wsdlcache
;php_value[opcache.file_cache]  = /var/lib/php/opcache
```

#### 编译PHP连接Redis的插件

##### 1.下载Redis源码包

```bash
wget https://pecl.php.net/get/redis-5.3.7.tgz
tar -zxvf redis-5.3.7.tgz
```

##### 2.配置Redis并编译安装

```bash
cd redis-5.3.7/
phpize
./configure
make && make install
```

##### 3.开启Redis插件功能，配置文件增加以下内容 

```bash
vim /etc/php.ini
1357:extension=redis.so
```

##### 4.检查并重启PHP服务

```bash
php-fpm -t
systemctl restart php-fpm
```

### WEB02要做的事

##### 1.将WEB01配置好的文件拷贝过来

```bash
scp web01:/etc/php.ini /etc
scp web01:/etc/php-fpm.d/www.conf /etc/php-fpm.d/
```

##### 2.下载、配置、编译并安装Redis插件

```bash
wget https://pecl.php.net/get/redis-5.3.7.tgz
tar -zxvf redis-5.3.7.tgz
cd redis-5.3.7

phpize
./configure
make
sudo make install
```

##### 3.检测并重启PHP服务

```bash
php-fpm -t
systemctl restart php-fpm
```





