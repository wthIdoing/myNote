# LNMP(Nginx+Mysql+PHP)架构的拆分

将学习过程中学习的Nginx+PHP+MySQL一体化服务的WEB服务器中，拆分出MySQL服务。

### DB01要做的事

首先需要准备一台专门负责数据库服务的服务器10.0.0.51(内网IP172.16.1.51)，名称db01（需要在其他服务器进行DNS解析）

在上面安装MySQL服务

因为MySQL已经闭源，所以使用MariaDB

#### 首先安装MariaDB，并启动服务(DB01服务器中)

```bash
yum -y install mariadb-server
systemctl start mariadb
systemctl enable mariadb
```

进入到MariaDB，并设置远程用户

> [!WARNING]
>
> 基于安全性考虑，MySQL数据库只允许本地的root用户进行登录，而拒绝root用户的远程登录
>
> 所以需要设置一个用于远程登录的用户

进入mysql

> [!NOTE]
>
> 刚安装的mariadb默认的root用户就是没有密码的，可以直接登录

```bash
mysql
```

#### 在MySQL中创建远程用户并授权

```mysql
grant all on *.* to remote_user@'%' identified by '123456';
flush privileges;
```

###### 创建用户语句

```mysql
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
```

- username：要创建的用户名
- host：指定用户登录时所用的主机，通配符%表示允许从任何主机登录
- password：用户登录的密码

###### 授权用户语句

```mysql
GRANT 权限 ON 对象名 TO username;
GRANT ALL ON *.* TO 'username'@'host';
```

- ALL 表示所有权限
- 第一个*表示数据库
- 第二个*表示数据表

#### 远程连接数据库测试

在另一台服务器上使用（随便一台服务器拿来测试）

```bash
mysql -hdb01 -uyellowsea -p123456
```

进行测试

### 数据库的迁移

#### 将WEB01上的数据库导出

```bash
mysqldump -uroot -p123456 -A > all.sql
# 将all.sql拷贝到db01服务器下的root路径
scp all.sql db01:/root
```

#### 将数据库导入至DB01

> [!IMPORTANT]
>
> 导入以后原先创建的用户不会被覆盖，包括未设置密码的root用户

DB01上进行的操作

```bash
mysql -uroot < all.sql
```

#### 修改WEB01本地与数据库相关的设置

首先需要关闭WEB01的数据库服务

WEB01上进行的操作

```bash
systemctl stop mariadb
systemctl disable mariadb
```

修改wordpress服务使用的数据库

```bash
vim /code/wordpress/wp-config.php
```

```php
 22 /** The name of the database for WordPress */
 23 define( 'DB_NAME', 'wordpress' );
 24 
 25 /** Database username */
 26 define( 'DB_USER', 'yellowsea' );
 27 
 28 /** Database password */
 29 define( 'DB_PASSWORD', '123456' );
 30 
 31 /** Database hostname */
 32 define( 'DB_HOST', 'db01' );
```

### 实现静态文件的一致性

在WordPress服务中，文章都被存放在数据库中，而上传的图片、视频等静态文件则额外存放在一个目录中。为了防止多台WEB服务器将各自的静态文件存放在各自的服务器中，需要将静态文件统一放在NFS服务器中。

#### NFS服务器要做的事

首先在NFS服务器中创建供多台WEB服务器使用的共享文件夹

```bash
mkdir -p /data/wp
```

设置属主属组（事先设置好了用户）

```bash
chown -R www.www /data/wp
```

##### 开放共享文件夹

配置NFS服务（以/data/wp文件夹举例）

```bash
/data/wp 172.16.1.0/24 (rw,sync,all_squash,anonuid=1000,anongid=1000)
```

重启NFS服务

```bash
systemctl restart nfs
```

查看配置文件是否正确（这些是自己配置的所有文件）

```bash
vim /var/lib/nfs/etab
/data/zh        172.16.1.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=1000,anongid=1000,sec=sys,rw,secure,root_squash,all_squash)
/data/wp        172.16.1.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=1000,anongid=1000,sec=sys,rw,secure,root_squash,all_squash)
/data   172.16.1.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=1000,anongid=1000,sec=sys,rw,secure,root_squash,all_squash)
```

#### WEB服务器要做的事

将服务中的静态文件拷贝至NFS服务器

```bash
scp -r /code/wordpress/wp-content/uploads/* nfs:/data/wp/
```

然后将NFS的共享文件夹挂载到WEB01

```bash
showmount -e nfs
# 返回结果
Export list for nfs:
/data/zh 172.16.1.0/24
/data/wp 172.16.1.0/24
/data    172.16.1.0/24
```

挂载文件夹

```bash
mount -t nfs nfs:/data/wp /code/wordpress/wp-content/uploads/
```

在WEB2上重复相同的步骤，就完成了将Nginx+PHP与MySQL的拆分。
