# NFS网络文件系统小结

> [!TIP]
>
> NFS(Network File System)可以将远程服务器挂载到本地，从而实现不同服务器的**数据一致性**和**共享存储**

## NFS服务器的搭建

##### 下载服务

首先，需要在作为NFS服务器的服务器上下载NFS服务

> [!CAUTION]
>
> nfs服务的全称是nfs-utils

```bash
yum -y install nfs-utils
```

##### 配置文件

> [!WARNING]
>
> 与其他服务不同，nfs默认的配置文件名称与服务名称差别比较大
>
> 这里的[exports](#/etc/exports与/var/lib/nfs/etab的不同)可以理解为一种对外的api接口

```bash
cat /etc/exports
```

应该配置的内容

```bash
/data 172.16.1.0/24(rw,sync,all_squash)
指定共享文件夹
向某个网段开放
rw：拥有读写权限
sync：写入到磁盘才会返回结果，安全但是写入较慢
all_squash：权限压缩，不管客户端访问的用户身份权限是什么，都被压缩成nfs的nobody身份权限
```

##### 创建对应数据

配置完成后，应该创建配置文件中指定的文件夹

> [!WARNING]
>
> 不要忘了将文件夹的属主属组改为对应的用户与用户组

```bash
mkdir /data
chown nobody.nobody /data
```

##### 启动服务

> [!CAUTION]
>
> 与本体全称不同的是，服务的名称是nfs

```bash
systemctl start nfs
systemctl enable nfs
```

##### 检查服务的配置文件是否正确

> [!WARNING]
>
> 这里的配置文件与手动配置的配置文件路径不同

```bash
cat /var/lib/nfs/etab
```

返回

```bash
/data 172.16.1.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=65534,anongid=65534,sec=sys,rw,secure,root_squash,all_squash)
```

## 客户端要做的事情

需要预先下载nfs-utils服务，否则没有showmount指令

```bash
yum -y install nfs-utils
```

使用showmount -e 查看目标服务器是否可以挂载

> [!NOTE]
>
> 这里的-e是exports的意思喔

```bash
showmount -e 172.16.1.31
或者如果有做DNS解析
showmount -e nfs
```

进行挂载

```bash
mount -t nfs 172.16.1.31:/data /mnt
或者事先做了DNS解析
mount -t nfs nfs:/data /mnt
mount -t 文件系统类型 IP:/指定共享的文件夹 /挂载目录
-t	指定文件系统类型
```



## 有什么疑惑？

#### 为什么包名是nfs-utils,服务名却是nfs

##### nfs软件包

​	nfs-utils是整个软件包，包括NFS服务端程序，也包括客户端工具，管理命令(showmount,exportfs)等

​	utils：utilities的缩写，意为**实用工具**

###### 	包含的命令

```bash
showmount

exportfs
```

##### nfs服务

​	nfs-server简写为nfs，是systemctl启动的服务名称，启动服务端的**核心功能**

#### /etc/exports与/var/lib/nfs/etab的不同

##### exports

​	exports是写配置的地方，配置可以很简略，因为系统有很多**默认值**，没写的选项系统会自动补全

##### etab

> [!NOTE]
>
> ​	tab，table的简写，也就是表格
>

​	写完配置文件并执行exportfs -r后，系统会解析exports，加上隐含的默认参数，生成最终完整的执行清单，这份清单就是etab。只有**etab中写的参数**才是nfs服务执行生效的。

#### 如果我想修改exports配置？

​	修改完成后，**不需要重启NFS服务**，应该使用exportfs -r刷新配置

​	也就是说共享规则由exportfs命令管理。它负责将配置信息写入到NFS内核的“执行清单”(etab)中。

#### sync与async的准确含义？

##### sync

​	nfs在收到客户端写入请求后，必须**等待数据被真正写入服务器磁盘**，才会向客户端返回“写入成功”的确认信息。

##### async

​	nfs在受到客户端写入请求后，数据**只需写入服务器的内存缓存**，直接向客户端返回“写入成功”的确认信息。服务器随后异步(Async)地将数据写入磁盘。

​	因为先写入内存，如果发生意外断电、系统崩溃或服务器重启，数据会**永久丢失**

#### all_squash的含义

all_squash表示所有**用户权限压缩（Squashing）**

- 在NFS中，特指权限的映射和降级
- root_squash(默认)：仅将客户端以root身份访问的用户映射为匿名用户**(只对root用户生效)**。其他普通用户的身份保持不变
- all_squash：将客户端所有用户的身份都映射为匿名用户

## 参数

| 参数名称          | 参数用途                         |
| ----------------- | -------------------------------- |
| rw                | read-write，可读写               |
| ro                | read-only，只读                  |
| sync              | 将数据写入到磁盘才返回           |
| async             | 将数据写入到内存缓存直接返回     |
| all_squash        | 全部用户权限压缩                 |
| root_squash(默认) | 只压缩root用户的权限             |
| no_root_squash    | 不会压缩root用户的权限           |
| anonuid           | anon: anonymous<br />匿名用户UID |
| anongid           | 匿名用户GID                      |

