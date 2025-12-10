# Rsync基础

与cp命令不同的是，Rsync命令使用的是增量备份，即备份的时候不会将目标主机已经有的文件再次传输

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
vim /etc/syncd.conf
uid = rsync							#运行进程的用户
gid = rsync							#运行进程的用户组
port = 873							#默认端口
fake super = yes					#无需让rsync以root身份运行，允许接受文件的完整属性
use chroot = no						#禁锢推送的数据至某个目录，不允许跳出该目录
max connections = 200				#最大连接数
timeout = 600						#超时时间
ignore errors						#忽略错误信息
read only = false					#对备份数据可读写
list = false						#不允许查看模块信息
auth users = rsync_backup			#定义虚拟用户作为连接认证用户
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

   ```bash
   echo rsync_backup:123 > /etc/rsync.passwd
   chmod 600 /etc/rsync.pass					#修改权限，只有600权限才能正常工作
   ```

   

3. ###### 准备共享目录

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

### 客户端要做的事情（WEB01服务器举例）

将备份文件进行打包、生成校验文件、打包文件与校验码一起发送给服务端

清理7天的过期文件（额外）

将脚本文件统一放在/myscripts/目录下

##### 文件打包

```bash
[root@web01 ~]#vim /myscripts/backup.sh 
# 对文件进行备份
tar -zcvf /backup/web01/`hostname`_`hostname -I | awk '{print $2}'`_`date +%F`tar.gz -C /etc hosts passwd crontab
```



##### 生成校验文件

```bash
[root@web01 ~]#vim /myscripts/md5sum.sh 
# 生成备份文件的md5校验码
md5sum /backup/web01/`hostname`_`hostname -I | awk '{print $2}'`_`date +%F`.tar.gz > /backup/web01/md5.log
```



##### 发送给服务端

> [!NOTE]
>
> 其中，已经事先在/etc/hosts中对backup做过域名解析，所以这里直接使用backup(用户名)@backup(主机名)

```bash
[root@web01 ~]#vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.1.41 backup
```

> [!TIP]
>
> 其实这里才是发送给客户端
>

```bash
[root@web01 ~]#vim /myscripts/push.sh 
# 将备份文件和md5校验码推送至备份服务器
rsync -avz /backup/web01 backup@backup::web01 --password-file=/etc/rsync.passwd
```



##### 清理过期文件

```bash
[root@web01 ~]#vim /myscripts/7dsave.sh 
# 清理7天备份文件
find /backup/web01/ -type f -mtime +7 | xargs rm -rf
```



> [!IMPORTANT]
>
> 不要忘记写定时任务

##### 定时任务

```bash
[root@web01 ~]#crontab -e
# 备份文件
00 01 * * * sh /myscripts/backup.sh &> /dev/null
# 生成md5校验码
00 01 * * * sh /myscripts/md5.sh &> /dev/null
# 推送文件和md5校验码
00 01 * * * sh /myscripts/push.sh &> /dev/null
# 清理备份目录7天文件
00 00 */7 * * sh /myscripts/7dsave.sh &> /dev/null
```



### 服务端要做的事情

校验客户端的文件，如果校验失败则发送邮件给管理员

保留6个月的备份数据，其余的删除

##### md5校验脚本，对web01与nfs发送的文件进行md5校验(后续有继续加)

```bash
[root@backup ~]#vim /myscripts/md5.sh
# 对web01发送的文件进行校验
md5sum -c /backup/web01/md5.log
# 对nfs发送的文件进行校验
md5sum -c /backup/nfs/md5.log
//TODO
```

专门建立一个文件夹存放md5校验的结果

```bash
[root@backup ~]#mkdir -p /mylog/md5/
```



对客户端发送的备份文件用md5文件进行校验

```bash
[root@backup ~]#vim /myscripts/md5.sh 
# 对web01发送的文件进行校验,放到对应的文件夹(暂时)
md5sum -c /backup/web01/md5.log >> /mylog/md5/web01md5.log
# 对nfs发送的文件进行校验，放到对应的文件夹(暂时)
md5sum -c /backup/nfs/md5.log >> /mylog/md5/nfsmd5.log

[root@backup ~]#cat /mylog/md5/nfsmd5.log /mylog/md5/web01md5.log 
/backup/nfs/nfs_172.16.1.31_2025-11-27: OK
/backup/web01/web01_172.16.1.7_2025-11-27: OK

```

> [!NOTE]
>
> 可以看到是校验成功的⬆️

##### 清理超过6个月的备份文件

```bash
[root@backup ~]#vim /myscripts/6msave.sh
# 服务端保留6个月内的备份文件
find /backup/ -type f -mtime +`expr 6 \* 30` | xargs rm -rf
```

##### 将校验结果输出到屏幕或发送邮件给管理员(不会)

```java
//TODO
```

> [!IMPORTANT]
>
> 不要忘记写定时任务

##### 定时任务

```bash
[root@backup ~]#crontab -e
# 使用统一的脚本对所有服务器的备份文件进行校验
00 01 * * * sh /myscripts/md5.sh
# 对备份文件夹中超过6个月的文件进行清理
00 01 * * * sh /myscripts/6msave.sh
```









