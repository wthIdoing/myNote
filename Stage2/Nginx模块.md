# Nginx模块

### 索引模块

索引模块可以向用户提供下载服务

默认路径为/etc/nginx/conf.d/

```nginx
server {
	listen 80;
	server_name download.oldboy.com;
	
	location / {		# 这里的/实际上就是/code的占位符
		root /code;
		charset utf-8;				# UTF-8字符集
		autoindex_localtime on;		# 本地时间
		autoindex_exact_size off;	# 不要确切地(exact)显示大小，以人类可读的方式显示文件大小 KB MB GB等
		autoindex on;				# 开启索引
	}

}
```

关于别名

> [!IMPORTANT]
>
> location块中，root与alias的区别

````nginx
server {
	listen 80;
	server_name download.oldboy.com;
	
	location /download {	# /可以看作下行/code的占位符，所以拼接起来的路径是/code/download
		root /code;	
		charset utf-8;				
		autoindex_localtime on;		
		autoindex_exact_size off;	
        autoindex on;
	}

}

server {
	listen 80;
	server_name download.oldboy.com;
	
	location /download {	# 如果这里使用的是别名alias，那么整个/download就是直接定位到/etc下
		alias /etc;	
		charset utf-8;				
		autoindex_localtime on;		
		autoindex_exact_size off;	
        autoindex on;
	}

}
````

### 访问控制

在location块中，我们可以决定哪些用户可以访问，哪些用户禁止访问。

> [!IMPORTANT]
>
> 生效的规则是从上往下进行匹配

```nginx
server {
        listen 80;
        server_name www.oldboy.com;

        location / {
        deny 10.0.0.41;			# 拒绝10.0.0.41访问
        allow all;			    # 允许其他所有IP访问
        root /code;
        index index.html;
        }
}

server {
        listen 80;
        server_name www.oldboy.com;

        location / {
        allow 10.0.0.41;			    # 只允许41访问
        deny all;						# 拒绝所有IP
        root /code;
        index index.html;
        }
}

# 允许多个不同的网段、然后拒绝所有
location / {
    deny  192.168.1.1;
    allow 192.168.1.0/24;
    allow 10.1.1.0/16;
    deny  all;
}
```

### 用户认证

```bash
auth_basic 用户验证模块
-b	# 允许在命令行直接输入密码，免交互
-c	# 创建文件
```

在/etc/nginx/路径下使用htpasswd命令生成密码文件

```bash
cd /etc/nginx
htpasswd -b -c auth_pass oldboy 123
			写在配置里的文件名称 用户名 密码
```

如果不使用-b选项，则是交互式输入密码

可以查看密码文件，密码是经过了加密的

```bash
cat /auth_pass
```

配置模块

```nginx
server {
	listen 80;
	server_name www.oldboy.com;

	location / {
    deny 10.0.0.41;
	allow all;
	root /code;
	index index.html;
	}

	location /download {
	auth_basic "hehe";				# 描述 必须有
	auth_basic_user_file auth_pass; # 指定密码文件的位置
    alias /code;
  	charset utf-8;
	autoindex_localtime on;
	autoindex_exact_size off;
        autoindex  on;
        }
}
```

### 状态模块

```nginx
server {
	listen 80;
	server_name www.oldboy.com;

	location / {
        deny 10.0.0.41;
	allow all;
	root /code;
	index index.html;
	}

	location /nginx_status {		# 这里是状态模块
	stub_status;
	}

}
```

使用浏览器访问www.oldboy.com/nginx_status则会显示

```bash
Active connections: 1 			# 当前TCP的连接数
server accepts handled requests
 1 1 1 
Reading: 0 Writing: 1 Waiting: 0 


Active connections: 1   # 当前TCP的连接数
accepts   				# 接受过多少次TCP连接
handled  				# 连接成功的数量
requests 				# 总的请求数
Reading: 0   # 读取头部信息数量   
Writing: 1   # 响应头部信息       
Waiting: 0   # 等待请求的数量      
```

### 连接数量限制

```nginx
#必须配置在HTTP层、不能配置到server层
limit_conn_zone $remote_addr zone=conn_zone:10m;
server   {
     #连接限制，限制同时最高1个连接
     limit_conn conn_zone 1;
}
```

### 请求数量限制

```nginx
#出现请求攻击行文
1.可以通过deny模块限制IP
2.如果IP经常变化使用limit_req_zone定义请求限制
[root@web01 conf.d]#cat z.conf
limit_conn_zone $remote_addr zone=conn_zone:10m;
limit_req_zone $binary_remote_addr zone=req_zone:10m rate=50r/s;
server {
	listen 80;
	server_name xbw.oldboy.com;
	limit_conn conn_zone 10;
	 limit_req zone=req_zone burst=10 nodelay;
	 limit_req_status  478;  # 自定义状态码
        root /code/xbw;
	index index.html;
}
```

### location匹配优先级

```bash
匹配符			匹配规则					优先级  
=			精确匹配						1
^~			以某个字符串开头				2
~			区分大小写的正则匹配				3
~*			不区分大小写的正则匹配				4
/			通用匹配，任何请求都会匹配到		5
```

```nginx
server {
        listen 80;
        server_name prioriry.oldboy.com
        devault_type text/html;
        location = / {
        return 200 "configuration A";
        }
        location / {
        return 200 "cinfiguration B";
        }
        location /documents/ {
        return 200 "configuration C";
        }    
        location ^~ /images/ {
        return 200 "configuration D";
        }
        location ~ /Documents/ {
        return 200 "configuration E";
        }
        location ~* \.(gif|jpg|jpeg){
        return 200 "configuration F";
        }

}

```

### 限速模块

```nginx
location /download {
    limit_rate_after 50M;		# 下载50M不限速
    limit_rate       50k;		# 然后以50k的速度下载
    auth_basic "hehe";
    auth_basic_user_file auth_pass;
    alias /code;
    charset utf-8;
    autoindex_localtime on;
    autoindex_exact_size off;
    autoindex  on;
}
```

