# Nginx中的rewrite重定向

## URL的完整表达（重要！）

**URL=协议://域名:端口+/路径/文件+?参数&参数**

​		     服务器信息+ $uri        + $args

### Nginx伪四层代理

修改LB中的nginx.conf配置

```bash
vim /etc/nginx/nginx.conf
```

```nginx
...
stream {
    upstream web01 {
        server 172.16.1.7:22;	# 这里使用的是四层代理所以没有DNS解析，不能使用/etc/hosts中的web01等域名进行解析   
    }
    upstream web02 {
        server 172.16.1.8:22
    }
    server {
        listen 5555;
        proxy_pass web01;
    }
    server {
        listen 5555;
        proxy_pass web02;
    }

}
# 因为使用的是四层转发，所以要在http块外进行配置
http {
    ...
}
```

检查并重载Nginx服务

```bash
nginx -t
systemctl reload nginx
```

在另一台服务器上连接LB服务器5555端口进行测试

```bash
ssh 10.0.0.5 -p 5555
```

会直接跳转到web01服务器

### rewrite中的last与break

先编辑一段用来测试跳转功能的.conf

```bash
vim /etc/nginx/conf.d/test.conf
```

```nginx
server {
    listen 80;
    server_name test.oldboy.com;
    root /code/test;

    location / {
        rewrite /1.html /2.html;	# 1.html->2.html
        rewrite /2.html /3.html;	# 2.html->3.html
    }

    location /2.html {
        rewrite /2.html /a.html;	# 2.html->a.html
    }

    location /3.html {
        rewrite /3.html /b.html;	# 3.html->b.html
    }

}
```

Nginx服务不会只在一个location中进行重定向，所以

1.html-->2.html-->3.html-->b.html

#### break的应用

```nginx
server {
    listen 80;
    server_name test.oldboy.com;
    root /code/test/;

    location / {
        rewrite /1.html /2.html break; # 直接停止重定向
        rewrite /2.html /3.html; 
    }

    location /2.html {
        rewrite /2.html /a.html;	
    }

    location /3.html {
        rewrite /3.html /b.html;
    }
}
```

break会停止所有location的重定向，所以

1.html-->2.html

结束

#### last的应用

```nginx
server {
    listen 80;
    server_name test.oldboy.com;
    root /code/test/;

    location / {
        rewrite /1.html /2.html last; # 停止在这个location块中的重定向
        rewrite /2.html /3.html; 		
    }

    location /2.html {
        rewrite /2.html /a.html;	# 最终2.html会被这个location匹配到，进而跳转至a.html
    }

    location /3.html {
        rewrite /3.html /b.html;
    }
}
```

last会跳出这个语句所在的重定向(可以理解为continue跳出本次循环？)，所以

1.html-->2.html-->a.html

#### rewrite中的其他跳转案例

```nginx
server {
    listen 80;
    server_name rewrite.oldboy.com;
    root /code/test;

    location ~ /(.*abc)$ {		# 正则表达式，匹配到任意以abc为结尾的URI资源
        rewrite ^(.*)$ /ccc/bbb/2.html;	# 会重定向	到/code/test/ccc/bbb/2.html
    }
}
```

```nginx
server {
    listen 80;
    server_name rewrite.oldboy.com;
    root /code/test;

    location ~ /2014/ccc/2.html {
        rewrite ^(.*)$ /2025/ccc/bbb/2.html;	# 当用户访问....com/2014/ccc/2.html会跳转到  任意字符/2025/ccc/bbb/2.html
    }
}
```

#### rewrite的后向引用

```nginx
server {
    listen 80;
    server_name rewrite.oldboy.com;
    root /code/test;

    location ~ /2014/ccc/2.html {
        rewrite ^/2014/(.*)$ /2025/$1;	# 这里使用到了与sed相似的后向引用，(/ccc/2.html)的内容被引用到了$1
    }
    #用户访问/2014/ccc/2.html实际上真实访问的是/2025/ccc/2.html
}
server {
    listen 80;
    server_name rewrite.oldboy.com;
    root /code/test;

    location / {
        rewrite ^/test/2014/(.*)/(.*)$ /2025/$1/$2;
    }
    #用户访问/test/2014/ccc/2.html实际上真实访问的是/2025/ccc/bbb/2.html
}
```

#### 错误跳转

```nginx
server {
    listen 80;
    server_name rewrite.oldboy.com;
    charset utf-8,gbk;
    root /code/test;
    error_page 403 404 500 501 502 @error_test;

    location @error_test {
        rewrite ^(.*)$ /404.html break;	# 当遇到以上异常状态码时就会重定向到指定页面，并且停止重定向
    }
}
```

#### args参数

```nginx
server {
    listen 80;
    server_name rewrite.oldboy.com;
    charset utf-8,gbk;
    root /code/test;
    index index.html;
    set $args "&showoffline=1";

    if ($remote_addr = 10.0.0.1) {
        rewrite ^(.*)$ http://rewrite.oldboy.com$1;	# 在输入rewrite.oldboy.com网址时，后面默认携带指定的args参数，也就是showoffline=1
    }
}
```

#### if相关语句

```nginx 
server {
    listen 80;
    server_name rewrite.oldboy.com;
    charset utf-8,gbk;
    root /code/test;
    index index.html;

    set $ip 0;	# 刚好这个变量名称叫ip

    if ($remote_addr = 10.0.0.1){
        set $ip 1;  # 如果客户端IP是10.0.0.1则重新赋值为1
    }

    if ($ip = 0){
        rewrite (.*) /wh.html break;
    }

}
```

## 自己的相关实验

```nginx
server {
    listen 80;
    server_name rewrite.oldboy.com;
    charset utf-8,gbk;
    root /code/test;

    #location /test {
    #rewrite ^(.*)$ http://www.baidu.com permanent;
    #return 301 http://www.baidu.com;
    #rewrite ^(.*)$ http://www.baidu.com redirect;
    #return 302 http://www.baidu.com;
    #}

    #location /abc/1.html {
    #       rewrite ^(.*)$ /ccc/bbb/2.html;
    #       #return 302 /ccc/bbb/2.html;

    #}

    #location ~ /(.*abc)$ {
    #       rewrite ^(.*)$ /ccc/bbb/2.html;
    #}

    #location ~ /2014/ccc/bbb/2.html {
    #       rewrite ^/2014/(.*)$ /$1;
    #}

    error_page 403 404 500 501 502 @error_test;     # 配置了异常状态跳转页面

    location @error_test {
        rewrite ^(.*)$ /404.html break;
    }

    #index index.html;
    #set $args "&showoffline=1";

    #if ($remote_addr = 10.0.0.1) {
    #       rewrite ^(.*)$ http://rewrite.oldboy.com$1;
    #}

    set $ip 0;		#  这里的$ip只是一个自定义的变量，并不是Nginx的内置变量

    if ($remote_addr = 10.0.0.1){
        set $ip 1;
    }

    if ($ip = 0){
        rewrite (.*) /wh.html break;
    }

}

```

## **return+状态码与rewrite重定向的区别（重要！）**

| **模式**       | **rewrite 语句格式**                       | **触发机制**              | **最终结果 (对浏览器)**                                      | **F12 状态码**        | **适用场景**                                                 |
| -------------- | ------------------------------------------ | ------------------------- | ------------------------------------------------------------ | --------------------- | ------------------------------------------------------------ |
| **内部重写**   | `rewrite regex replacement **last**;`      | 显式强制内部跳转。        | URI 被修改，Nginx 重新匹配 `location` 并继续处理。           | **200 OK** (最终内容) | 最常用。用于内部路径转换和优化路由，用户无感知。             |
| **内部重写**   | `rewrite regex replacement **break**;`     | 显式停止 `rewrite` 模块。 | URI 被修改，Nginx 在当前 `location` 内继续处理其他指令。     | **200 OK** (最终内容) | 简单路径美化，通常用于 `location` 块内的最后一条 `rewrite` 规则。 |
| **内部重写**   | `rewrite regex **relative_uri**;` (无标志) | **默认行为（单次修改）**  | 行为等同于 `last`。URI 被修改，Nginx 重新匹配 `location`。   | **200 OK** (最终内容) | **只在** URI 在当前 `location` 内被修改 **不超过一次** 时成立。 |
| **外部重定向** | `rewrite regex replacement **redirect**;`  | 显式强制 302 重定向。     | Nginx 发送 302 响应，浏览器发起新请求。                      | **302 Found**         | 临时 URL 变更或跨域名/跨协议跳转，需要通知浏览器。           |
| **外部重定向** | `rewrite regex replacement **permanent**;` | 显式强制 301 重定向。     | Nginx 发送 301 响应，浏览器发起新请求并缓存跳转。            | **301 Moved**         | 永久 URL 变更，利于 SEO。                                    |
| **外部重定向** | `rewrite regex **full_url**;` (无标志)     | **默认行为（完整 URL）**  | 行为等同于 `redirect` (302)。                                | **302 Found**         | 目标是包含 `http://` 或 `https://` 的完整 URL。              |
| **外部重定向** | `rewrite regex **relative_uri**;` (无标志) | **默认行为（多次修改）**  | Nginx 在 `location` 内发现 **多次** URI 修改后，发送 302 响应。 | **302 Found**         | 内部重写逻辑出现循环或多步转换，Nginx 自动介入。             |

**使用return会返回状态码**

**![image-20251210205542838](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251210205542838.png)**

**使用rewrite不返回状态码**

**![image-20251210205547795](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251210205547795.png)**



| **rewrite 目标格式** | **URI 修改次数**        | **Nginx 判定**        | **Nginx 最终结果**               | **F12 状态码**        |
| -------------------- | ----------------------- | --------------------- | -------------------------------- | --------------------- |
| 相对 URI (`/path`)   | **1 次** (或 0 次)      | 单次内部重写          | `last` 行为，重新匹配 `location` | **200 OK** (最终内容) |
| 相对 URI (`/path`)   | **2 次及以上** (循环中) | 多次内部重写/潜在循环 | **302 外部重定向**               | **302 Found**         |
| 完整 URL (`http://`) | 任意次数                | 外部跳转请求          | **302 外部重定向**               | **302 Found**         |



## last与break的区别

| 特性                     | rewrite ... last                                             | rewrite ... break                                            |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **触发机制**             | 内部跳转(Internal Redirection)                               | 停止重写，继续执行(Stop Rewrites, Continue Execution)        |
| **Location匹配**         | 重新开始：Nginx用新的URI重新启动整个location块的匹配过程（从头开始）。 | 保持不变：Nginx停留在当前匹配到的location块中。              |
| 处理流程                 | 将请求导向另一个最匹配的新location块。                       | 在当前location块中继续执行后续指令（如proxy_pass，root等）。 |
| **当前Block内的rewrite** | 不执行任何后续的rewrite指令，直接跳转                        | 不执行任何后续的rewrite指令                                  |
| **URI结果**              | 最终由新匹配的location块决定。                               | 最终由当前location块中的其他指令决定。                       |
| **应用场景**             | 内部路由：将请求根据新的路径，路由到一个功能完全不同的location块进行处理。 | 路径美化/内部重写：在当前location内，将外部URL美化为内部程序上或文件路径。 |

## index与error_page语法差异

因为Nginx配置文件是受**指令(Directive)驱动**的，没有一种通用的语法，所以

#### ⚙️ 语法定义：

- **指令：** `index`
- **参数：** 接受 **1 到 N 个** 文件名作为参数。

#### ⚙️ 语法定义：

- **指令：** `error_page`
- **参数：** 接受 **1 到 M 个** 状态码参数，然后是 **1 个** URI（或 `location` 名）参数。

也就是将前面的状态码统一交给最后一个URI处理

| 指令       | 参数模式        | 含义                                                         |
| ---------- | --------------- | ------------------------------------------------------------ |
| index      | N个参数         | 所有参数都属于同一种类型（文件路径）。                       |
| error_page | M个参数+1个参数 | 前M个参数属于一种类型（状态码），最后1个参数属于另一这类型（处理动作）。 |

## $agrs的作用

#### 1. 传递状态和控制参数 (最常见用途)

这类参数告诉服务器（或应用）：**“我想看什么”** 或 **“你想如何处理我的请求”**。

- **分页/排序：** `?page=3&sort=time`
- **过滤/搜索：** `?category=tech&search=Nginx`
- **显示模式：** `?view=mobile` 或您在 Nginx 例子中看到的 `&showoffline=1`（可能用于显示离线内容）。
- **API 请求：** `?apikey=xyz123`

#### 2. 标识用户会话或安全信息

虽然不安全，但在某些简单的或历史遗留系统中，查询字符串可能用于：

- **会话跟踪：** `?sessionid=abcde`
- **令牌/凭证：** `?token=fghijk` (例如一些邮件链接的验证或密码重置链接)。
  - *（在现代 Web 开发中，这些信息通常通过 **Cookies** 或 **HTTP Header** 传递以提高安全性。）*

#### 3. 动态内容生成

对于服务器端的动态语言（如 Java 的 Servlet/Spring、PHP、Node.js 等）：

- **Java 开发示例：** 当您在 Java 应用中通过 `HttpServletRequest.getParameter("key")` 来获取数据时，这些数据就是从 `$args` 中解析出来的。例如：
  - 访问 `http://app.com/product?id=123`
  - Java 代码通过 `request.getParameter("id")` 得到值 `123`，然后根据这个 ID 去数据库查询对应的商品信息并展示。

### 🚀 总结：Nginx 与 `$args` 的关系

对于 Nginx 来说：

1. Nginx **自己不处理** `$args` 携带的业务逻辑（比如它不会去判断 `id=123` 应该对应哪个商品）。
2. Nginx 的职责是 **接收** `$args`，并将其原封不动地 **传递** 给后端的应用服务器（如 Tomcat, Jetty, 或 PHP-FPM），让后端程序去执行具体的业务逻辑。

### 使用if语句控制用户访问

```nginx
set $ip 0;	# 刚好这个变量名称叫ip

if ($remote_addr = 10.0.0.1){
    set $ip 1;  # 如果客户端IP是10.0.0.1则重新赋值为1
}

if ($ip = 0){
    rewrite (.*) /wh.html break;
}
```

可以看到同样是访问rewrite.oldboy.com，本地(10.0.0.1)可以访问到首页，而其他IP都会进行跳转

![image-20251210215231784](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251210215231784.png)

### Nginx内置变量


```nginx
$args           #这个变量等于请求行中的参数。
$content_length #请求头中的Content-length字段。
$content_type   #请求头中的Content-Type字段。
$document_root  #当前请求在root指令中指定的值。
$host           #请求主机头字段，否则为服务器名称。
$http_user_agent #客户端agent信息
$http_cookie    #客户端cookie信息
$limit_rate      #这个变量可以限制连接速率。
$request_body_file #客户端请求主体信息的临时文件名。
$request_method     #客户端请求的动作，通常为GET或POST。
$remote_addr     #客户端的IP地址。
$remote_port     #客户端的端口。
$remote_user    #已经经过Auth Basic Module验证的用户名。
$request_filename #当前请求的文件路径，由root或alias指令与URI请求生成。
$query_string   #与$args相同。
$scheme         #HTTP方法（如http，https）。
$server_protocol                 #请求使用的协议，通常是HTTP/1.0或HTTP/1.1。
$server_addr        #服务器地址，在完成一次系统调用后可以确定这个值。
$server_name     #服务器名称。
$server_port    #请求到达服务器的端口号。
$request_uri     #包含请求参数的原始URI，不包含主机名，如：”/foo/bar.php?arg=baz”。
$uri            #不带请求参数的当前URI，$uri不包含主机名，如”/foo/bar.html”。
$document_uri   #与$uri相同。

$X-Forwarded-For	#HTTP的请求端真实的IP，只有在通过了HTTP 代理或者负载均衡服务器时才会添加该项。标准格式如下：X-Forwarded-For: client1, proxy1, proxy2
```

