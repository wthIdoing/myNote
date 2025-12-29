# Rsync基础

与cp命令不同的是，Rsync命令使用的是**增量备份**，即备份的时候不会将目标主机已经有的文件再次传输

可以减少带宽压力，提高备份速度

## Rsync语法

```bash
rsync [options] source destination:directory
参数:
-a					#archive	归档模式，保留所有文件属性
-v					#verbose	显示详细传输信息
-z					#compross传输时压缩数据
--exclude=			#不要的文件
--bwlimit=			#限制的带宽	bw:bandwidth
--delete 			#完全同步，目标主机有额外的文件也一并删除
--password-file=	#指定客户端的密码文件路径
```

##### rsync的安装

```bash
yum -y install rsync
```



##### 本地命令模式

```bash
sync -avz 源目录/文件 目标目录
```



> [!NOTE]
>
> 使用远程命令模式时，需要输入主机密码

##### 远程命令模式

```bash
rsync -avz 源目录/文件 目标主机的用户@目标主机:目录
```



##### 守护进程模式

> [!WARNING]
>
> 由于在公网使用远程命令模式可能有**泄露**目标主机用户密码的风险，需要使用守护进程模式

```bash
rsync -avz 源目录/文件 rsync用户@目标主机::模块
```

##### 这是守护进程模式的配置文件，在使用守护进程模式同步文件时用到

> [!TIP]
>
> 在守护进程模式中，指定的Rsync用户、密码、模块等需要在**服务端**配置文件中进行配置

### rsync文件配置

 进行rsyncd.conf文件的配置

```bash
vim /etc/rsyncd.conf
uid = www							#运行进程的用户，这里避免与nfs服务用户冲突，统一为www用户与用户组
gid = www							#运行进程的用户组
port = 873							#默认端口
fake super = yes					#无需让rsync以root身份运行，允许接受文件的完整属性
use chroot = no						#禁锢推送的数据至某个目录，不允许跳出该目录
max connections = 200				#最大连接数
timeout = 600						#超时时间
ignore errors = yes					#忽略错误信息
read only = false					#对备份数据可读写
list = false						#不允许查看模块信息
auth users = backup			#定义虚拟用户作为连接认证用户
secrets file = /etc/rsync.passwd	#定义rsync服务用户连接认证密码文件路径

[backup]							#定义模块信息
comment = commit					#模块注释信息
path = /backup						#定义接收备份数据的目录
```

> [!IMPORTANT]
>
> 在守护进程模式中，配置文件中指定的用户、模块目录等需要在**服务端**中进行配置
>
> **客户端**中只需要配置服务端用户的密码（**只需要指定密码**，而不需要指定用户名）

1. ###### 创建虚拟用户

   ```bash
   groupadd rsync
   useradd -s /sbin/nologin -M rsync
   ```

   

2. ###### 准备密码文件

   > [!CAUTION]
   >
   > ****密码文件权限**只有600**才能正常工作，无论是**客户端**还是**服务端**
   >
   > **服务端的**密码文件的属主属组**必须为root**

   ```bash
   echo rsync_backup:123 > /etc/rsync.passwd
   chmod 600 /etc/rsync.passwd					#修改权限，只有600权限才能正常工作
   ```

   

3. ###### 准备共享目录

   > [!CAUTION]
   >
   > 共享目录必须更改属组属主
   >
   > 权限可以不用改

   ```bsah
   mkdir /backup/web01
   chown rsync.rsync /backup/web01		# 修改属主属组
   ```

##### 启动服务

```bash
systemctl start rsyncd		# 启动服务
systemctl enable rsyncd		# 开机自启
```

##### 查看服务器是否运行873端口

```bash
netstat -tunlp
-t	显示TCP传输协议的连线情况
-u	显示UDP传输协议的连线情况
-n	numeric 直接显示IP地址，而不通过域名服务器
-l	listening显示监控中的服务器的socket
-p	programd显示正在使用socket的程序识别码和程序名称
```

# 小结

应该将备份服务器作为服务端运行守护进程，其余客户端只需要配置服务端的密码，并且在客户端上使用守护进程执行推送与拉取，不应该在服务端上进行推送与拉取。



# 为备份文件写脚本、定期备份与校验

> [!CAUTION]
>
> 客户端的备份文件与校验码**路径**要与服务端的**相同**
>

### 客户端要做的事情（这里的脚本是通用脚本）

#### 将所有要做的事按步骤写入脚本

```bash
vim /servers/scripts/backup.sh
#!/bin/bash
#此脚本用于备份、校验、推送和清理7天文件
# 定义变量
HOST=`hostname`
IP=`hostname -I | awk '{pring $2}'`
TIME=`date +%F`
DIR=${HOST}_${IP}_${TIME}
# 创建打包目录和校验文件目录
mkdir -p /backup/$HOST/
# 打包文件到以上目录
tar -zcvf /backup/$HOST/$DIR.tar.gz -C /etc hosts passwd
# 用md5文件进行校验
md5sum /backup/$HOST/$DIR.tar.gz > /backup/$HOST/$DIR.tar.gz.md5
# 推送到backup
rsync -avz /backup/$HOST/ backup@backup::$HOST --password-file=/etc/rsync.passwd 
# 清理7天文件
find /backup/$HOST/ -mtime +7 | xargs rm -rf

```



> [!NOTE]
>
> 其中，已经事先在/etc/hosts中对backup做过域名解析，所以这里直接使用backup(用户名)@backup(主机名)

```bash
[root@web01 ~]#vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.1.41 backup
```



> [!IMPORTANT]
>
> 不要忘记写定时任务

##### 定时任务

```bash
[root@web01 ~]#crontab -e
# 备份、推送、清理七天文件
00 01 * * * sh /servers/sciprts/backup.sh &> /dev/null
```



### 服务端要做的事情

校验客户端的文件，如果校验失败则发送邮件给管理员

保留6个月的备份数据，其余的删除

##### 进行MD5校验，并发送邮件，清理超过6个月的文件

```bash
vim /servers/scripts/md5check.sh
#!/bin/bash
# 定义校验文件存放路径
RESULT=/servers/log/md5/`date +%F`.log
# 创建存放md5校验文件的文件夹
mkdir -p /servers/log/md5/
# 用于md5校验备份的文件
find /backup/ -name "*.md5" | xargs md5sum -c > $RESULT
# 将校验结果发送给邮箱
mail -s check_result 1329488086@qq.com < $RESULT
# 服务端保留6个月内的备份文件
find /backup/ -type f -mtime +`expr 6 \* 30` | xargs rm -rf
```

##### 安装邮件

```bash
yum -y install mailx sendmail
```

##### 配置邮件服务

```bash
vim /etc/mail.rc
set from=要发送邮件的邮箱
set smtp=smtp.163.com	#需要先在163邮箱中开启smtp服务
set smtp-auth-user=要发送邮件的邮箱
set smtp-auth-password=开启smtp服务后给的授权码
set smtp-auth=login
```

##### 测试发送邮件

```bash
echo 发送的内容 | mail -s "发送的标题" 目标邮箱
或者
mail -s "发送的标题" 目标邮箱 < /希望发送的内容的文件
```



> [!IMPORTANT]
>
> 不要忘记写定时任务

##### 定时任务

```bash
[root@backup ~]#crontab -e
# 使用统一的脚本对所有服务器的备份文件进行校验，并清理超过6个月的文件
00 01 * * * sh /servers/scripts/md5check.sh &> /dev/null
```









