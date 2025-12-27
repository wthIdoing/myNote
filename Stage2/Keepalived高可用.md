# Keepalived高可用

Keepalived基于VRRP（虚拟路由协议）实现，解决了负载均衡单点故障的问题

#### 那么什么是VRRP协议？（OSI七层模型中的网络层协议）

VRRP能够在**不改变组网**的情况下，采用将多台路由设备组成一个**虚拟路由器**，通过配置虚拟路由器的IP地址为默认网关，实现默认网关的备份。当网关设备发生故障时，VRRP机制能够**选举新的网关设备**承担数据流量，从而保障网络的**可靠通信**。

#### Keepalived高可用

VRRP只关心网卡和协议进程是否存活，而应用服务挂掉并不关心

Keepalived可以监控本地的**服务进程**、文件或运行**自定义脚本**，提高了VRRP的可用性

### 如何使用Keepalived

先准备一台LB02（第二台负载均衡服务器）

配置Nginx仓库

```bash
scp web01:/etc/yum.repos.d/nginx.repo /etc/yum.repos.d/
```

安装Nginx

```bash
yum -y install nginx
```

将LB01服务器的Nginx配置文件同步到LB02

```bash
rsync -avz web01:/etc/nginx/ /etc/nginx/
```

启动Nginx

```bash
systemctl start nginx
systemctl enable nginx
```

#### 分别在两台服务器上部署Keepalived

```bash
yum -y install keepalived
```

配置keepalived

```bash
vim /etc/keepalived/keepalived.conf
```

```properties
global_defs {                   # 全局配置
    router_id lb02              # 标识身份->名称
}

vrrp_instance VI_1 {
    state BACKUP                # 标识角色状态，主机用MASTER
    interface ens33              # 网卡绑定接口
    virtual_router_id 50        # 虚拟路由id
    priority 100                # 优先级  100票
    nopreempt					# 非抢占模式
    advert_int 1                # 监测间隔时间 秒
    authentication {            # 认证
        auth_type PASS          # 认证方式
        auth_pass 1111          # 认证密码
    }
    virtual_ipaddress {         
        10.0.0.3                # 虚拟的VIP地址
    }
}
```

> [!TIP]
>
> 非抢占模式：即主机挂了在备份机工作时，主机恢复了，也不会进行工作

###  Keepalived的脑裂

在 Keepalived 脑裂时，两台主机都因为收不到对方的 VRRP 通告报文，错误地认为对方宕机了，因此都将自己的状态切换为 **Master**，并且都尝试接管和宣告同一个**虚拟 IP (VIP)**。这时候应该**优先关闭低优先级节点（备份机）**，以防出现超出预期设计的情况。

现在我们在LB02上编写脚本

```bash
mkdir -p /servers/scripts/
```

```bash
vim /servers/scripts/check_nginx.sh
```

```bash
#!/bin/sh
NG=`ps -C nginx --no-header|wc -l`		# 设定变量，NG是筛选有无Nginx进程
if [ $NG -eq 0 ]
    then
    #如果nginx不存在则尝试重启nginx
    systemctl restart nginx
    #等待1秒
    sleep 1
    #在重新检查nginx是否存在
    NG=`ps -C nginx --no-header|wc -l`
    if [ $NG -eq 0 ]
        then
        #如果$NG变量为0说明nginx还是没有启动、只能杀死keepalived
        systemctl stop keepalived
        fi
fi
```

脚本需要Keepalived服务启动，所以需要x执行权限

```bash
chmod +x check_nginx.sh
```

最终的keepalived配置文件

```properties
global_defs {
    router_id lb02
}
vrrp_script check_nginx {		# Keepalived内部脚本名称
    script "/servers/scripts/check_nginx.sh"	# 脚本的位置
    interval 5		#  间隔5秒执行一次
}

vrrp_instance VI_1 {
	state BACKUP
	interface ens33
	virtual_router_id 50
	priority 150
	#nopreempt		# 不启用抢占模式
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	virtual_ipaddress {
		10.0.0.3
	}
	track_script {
		check_nginx		# 调用check_nginx脚本
	}
}
```

