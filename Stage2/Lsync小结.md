# Lsync小结

lsync可以实时监控并推送对应目录中更新的文件

### Lsync的安装

```bash
yum -y install lsyncd
```

### Lsync的配置

> [!NOTE]
>
> 需要在NFS服务器上开启lsync服务，实时同步至backup服务器

```bash
cat /etc/lsyncd.conf
```

##### 以下是配置文件

> [!WARNING]
>
> 格式必须正确，否则服务启动失败

```properties
settings {
	logfile = "/var/log/lsyncd/lsyncd.log",
	satusFile = "/var/log/lsyncd/lsyncd.status",
	maxProcesses = 2,		# 最大连接数
	nodaemon = false,		# 开启守护进程
}
sync {
	default.rsync,
	source = "/data/",
    target = "backup@backup::data",		# 主机名@地址：：模块名
	delete = true,		# 开启全量同步
	delay = 1,		# 延迟1秒推送
	rsync = {
		binary = "user/bin/rsync",		# 二进制命令文件
		password_file = "/etc/rysnc.passwd",	# 指定密码文件
		archive = true,		# 相当于-a，保留文件属性
		compress = true,		# 相当于-z，压缩
	}
}
```

##### 启动lsync服务

```bash
systemctl start lsyncd
systemctl enable lsyncd
```

## rsync服务与NFS服务用户冲突问题

为了保证rsync推送服务与NFS文件共享服务使用的目录**权限不冲突**，应该**统一用户**