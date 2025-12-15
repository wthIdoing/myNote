# HTTPS证书

## HTTPS加密流程

```tex
流程(面试题)
1、浏览器发起往服务器的443端口发起请求，请求携带了浏览器支持的加密算法和哈希算法。
2、服务器收到请求，选择浏览器支持的加密算法和哈希算法。
3、服务器下将数字证书返回给浏览器，这里的数字证书可以是向某个可靠机构申请的，也可以是自制的。
4、浏览器进入数字证书认证环节，这一部分是浏览器内置的TLS完成的：
4.1 首先浏览器会从内置的证书列表中索引，找到服务器下发证书对应的机构，如果没有找到，此时就会提示用户该证书是不是由权威机构颁发，是不可信任的。如果查到了对应的机构，则取出该机构颁发的公钥。
4.2 用机构的证书公钥解密得到证书的内容和证书签名，内容包括网站的网址、网站的公钥、证书的有效期等。浏览器会先验证证书签名的合法性（验证过程类似上面Bob和Susan的通信）。签名通过后，浏览器验证证书记录的网址是否和当前网址是一致的，不一致会提示用户。如果网址一致会检查证书有效期，证书过期了也会提示用户。这些都通过认证时，浏览器就可以安全使用证书中的网站公钥了。
4.3 浏览器生成一个随机数R，并使用网站公钥对R进行加密。
5、浏览器将加密的R传送给服务器。
6、服务器用自己的私钥解密得到R。
7、服务器以R为密钥使用了对称加密算法加密网页内容并传输给浏览器。
8、浏览器以R为密钥使用之前约定好的解密算法获取网页内容。
```

![image-20251214173021050](C:\Users\yellowsea\AppData\Roaming\Typora\typora-user-images\image-20251214173021050.png)

## 模拟网站篡改

### WEB01要做的事情

准备相应的配置文件与页面

```bash
vim /etc/nginx/conf.d/test.conf
```

```nginx
server {
    listen 80;
    server_name test.oldboy.com;
    root /code/test;
    index index.html;
    charset utf-8;
}
```

```bash
vim /code/test/index.html
```

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>我是title</title>
    </head>
    <body>
        <article>
            <header>
                <h1>我是妹妹</h1>
                <p>创建时间：<time pubdate="pubdate">2025/5/20</time></p>
            </header>
            <p>
                <b>Aticle</b>第一次用h5写文章,好他*的紧张...
            </p>
            <footer>
                <p><small>版权所有！</small></p>
            </footer>
        </article>
    </body>
</html>
```

### WEB02要做的事情

配置文件用来劫持WEB01

```bash
vim /etc/nginx/conf.d/jc.conf
```

```nginx
upstream jiechi {
    server 10.0.0.7:80;
}

server {
    listen 80;
    server_name test.olcboy.com;

    location / {
        proxy_pass http://jiechi;
        proxy_set_header Host $http_host;
        sub_filter '<h1>我是妹妹' '<h1>我是哥哥';
        sub_filter '<small>版权所有' ' <small>开源';
    }
}
[root@web02 conf.d]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@web02 conf.d]# systemctl restart nginx
```

由于WEB01使用的是HTTP协议，所有内容都以**明文**传输，所以会被劫持

## 保护域名

### 假证书

创建存放证书的路径

```bash
mkdir -p /etc/nginx/ssl_key
```

创建证书

```bash
cd /etc/nginx/ssl_key
openssl genrsa -idea -out server.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
......................................+++++
.............................................+++++
e is 65537 (0x010001)
Enter pass phrase for server.key:			# 输入密码 1234
Verifying - Enter pass phrase for server.key: # 输入1234
```

生成自签证书内容自定义填写

```bash
openssl req -days 36500 -x509 \		# 这里是输入
> -sha256 -nodes -newkey rsa:2048 -keyout server.key -out server.crt
Generating a RSA private key		# 这里是生成内容
.............................................................................................................................................+++++
...........+++++
writing new private key to 'server.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN	# 这里需要手动输入
State or Province Name (full name) [Some-State]:BJ
Locality Name (eg, city) []:SH
Organization Name (eg, company) [Internet Widgits Pty Ltd]:oldboy
Organizational Unit Name (eg, section) []:oldboy
Common Name (e.g. server FQDN or YOUR name) []:oldboy
Email Address []:oldboy123@qq.com
```

配置证书

```bash
vim /etc/nginx/conf.d/test.oldboy.conf
```

```nginx
server {
    listen 443 ssl;
    server_name test.oldboy.com;
    root /code/wordpress;
    ssl_certificate ssl_key/server.crt;		# 工作路径在/etc/nginx/ 这里使用相对路径
    ssl_certificate_key ssl_key/server.key;
    location / {
        index index.php index.html;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_keep_conn on;   		# 开启长连接
        fastcgi_connect_timeout 60s;	# 超时时间
        include fastcgi_params;
    }
}

#配置将用户访问http请求强制跳转https
server {
    listen 80;
    server_name test.oldboy.com;
    return 302 https://$server_name$request_uri;
    # rewrite ^(.*)$ https://$server_name$request_uri redirect;
}
```

### 真证书

```bash
[root@web01 ssl_key]# unzip 20022642_test.linuxnc.com_nginx.zip 
#将之前的做备份
[root@web01 ssl_key]# mv server.crt server.crt.bak
[root@web01 ssl_key]# mv server.key server.key.bak

#将下载的公钥和私钥进行改名(和配置文件一致)
[root@web01 ssl_key]# mv test.linuxnc.com.pem server.crt
[root@web01 ssl_key]# mv test.linuxnc.com.key server.key
```

配置证书

```bash
vim /etc/nginx/conf.d/test.conf
```

```nginx
server {
    listen 443 ssl;
    server_name test.linuxnc.com;	# 域名要与申请的证书的域名一致
    root /code/wordpress;
    ssl_certificate ssl_key/server.crt;
    ssl_certificate_key ssl_key/server.key;
    location / {
        index index.php index.html;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_keep_conn on;   		# 开启长连接
        fastcgi_connect_timeout 60s;	# 超时时间
        include fastcgi_params;
    }
}

#配置将用户访问http请求强制跳转https
server {
    listen 80;
    server_name test.linuxnc.com;
    return 302 https://$server_name$request_uri;
}
```

## 负载均衡+HTTPS

配置了均衡负载与HTTPS以后，WEB服务器不能直接接收HTTP协议的请求，否则会不断302重定向，最后浏览器显示连接已拒绝(ERR_CONNECTION_REFUSED)。

需要通过LB服务器与一台WEB服务器进行安装WordPress，否则会因为会话不一致导致安装失败

- 如果您没有在 LB 上配置**严格的会话粘滞 (Sticky Session)** 策略，LB 可能会将您安装过程中的第一个请求发送给 **WEB01**，而将第二个请求发送给 **WEB02**。

- 如果请求在两台服务器之间切换，**WEB01** 上的 PHP 进程可能完成了数据库创建，但当请求切换到 **WEB02** 时，**WEB02** 可能会因为文件（例如 `wp-config.php`）尚未完全生成或 PHP 会话丢失而中断安装，导致安装失败或数据不完整。

所以在安装的时候需要在LB服务器中屏蔽其中一台服务器进行安装，安装完成后再开启负载均衡

## WEB服务器要做的事

### WEB01和WEB02的配置

加上了HTTPS后，接受HTTP请求会不断返回302

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
        fastcgi_keep_conn on;                   # 开启长连接
        fastcgi_connect_timeout 60s;    # 超时时间
        include fastcgi_params;
        fastcgi_param HTTPS on;		# 添加此参数
    }
}
```

### 将WordPress放在NFS服务器上，避免会话不一致的问题

#### NFS服务器要做的事情

```bash
mkdir -p /data/wp
cd /data/wp
wget https://cn.wordpress.org/wordpress-6.0-zh_CN.tar.gz
tar -xf wordpress-6.0-zh_CN.tar.gz 
mv /data/wp/wordpress/* /data/wp/	# 把wordpress中的文件移动到共享目录中
```

WEB服务器上做的

将NFS共享目录挂载到wp.conf配置的指定目录

```bash
showmount -e nfs
mount -t nfs nfs:/data/wp /code/wordpress
```

### LB服务器的配置

> [!WARNING]
>
> 安装WordPress时，需要**屏蔽其他服务器**，通过**一台服务器进行安装**

```bash
vim /etc/nginx/conf.d/wp.conf
```

```nginx
upstream web {
    server 172.16.1.7:80;
    server 172.16.1.8:80;		# 如果要安装WordPress，需要注释调其中一个条，将其中一个服务器屏蔽掉
}
server {
    listen 443 ssl;
    server_name www.wp.com;
    ssl_certificate ssl_key/server.crt;		# Nginx的默认工作路径在/etc/nginx/，所以这里的路径是/etc/nginx/ssl_key……
    ssl_certificate_key ssl_key/server.key;
    include lv_env;

    location / {
        proxy_pass http://web;
    }

}

#配置将用户访问http请求强制跳转https
server {
    listen 80;
    server_name www.wp.com;
    return 302 https://$server_name$request_uri;
    # rewrite ^(.*)$ https://$server_name$request_uri;
}
```

```bash
nginx -t
systemctl reload nginx
```







