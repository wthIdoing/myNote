# Ansible主机清单与Ad-Hoc的语法

Ansible可以通过主机清单实现多台服务器的自动化运维，而且可以保证幂等性，状态一致性

## Ansible的部署

在正式使用Ansible之前，需要安装Python，并且基于Python运行Ansible

这里使用10.0.0.81，m01作为主机，

```bash
wget https://www.python.org/ftp/python/3.8.16/Python-3.8.16.tgz
tar xf Python-3.8.16.tgz
cd Python-3.8.16
```

使用命令编译安装

```bash
./configure --enable-optimizations
make -j$(nproc)
make altinstall
```

下载并安装好Python3.8后，返回/root/使用阿里源的镜像安装ansible

```bash
cd
pip3.8 install ansible -i https://mirrors.aliyun.com/pypi/simple
mkdir /etc/ansible	# ansible不会创建这个目录，必须手动创建
vim /etc/ansible/ansible.cfg	# ansible不会创建配置文件，必须手动配置
```

```properties
[defaults]
host_key_checking = False	 # 控制 Ansible 是否检查远程主机的 SSH 密钥指纹
deprecation_warnings = False # 控制是否显示“弃用警告”
interpreter_python = /usr/bin/python3  # 指定使用的python3版本
[inventory]					 # 主机清单的位置默认/etc/ansible/hosts
[privilege_escalation]		 # sudo提权选项
[paramiko_connection]		 # 连接插件
[ssh_connection]			 # SSH远程连接插件
[persistent_connection]	     # SSH持久连接选项 默认选项
[accelerate]
[selinux]	
[colors]					 # 颜色选项 默认
[diff]						 # copy模块对比内容 默认
```

```tex
#重要配置
1.禁用 SSH 主机密钥检查 (host_key_checking = False)： 便于自动化，牺牲少量安全性。
2.禁用弃用警告 (deprecation_warnings = False)： 让输出更干净。
3.强制使用 Python 3 (interpreter_python = /usr/bin/python3)： 确保与现代系统的兼容性，这是一个非常重要的设置。
```

#### sshpass

sshpass可以让ansible通过ssh的用户名密码的方式来管理后端

```bash
yum -y install sshpass
```

## Ansible的主机清单

### Ansible-Inventory

##### 单台定义

```properties
10.0.0.7 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'
```

定义完了可以使用ping进行测试

```bash
ansible 10.0.0.7 -m ping
```

##### 定义别名

> [!TIP]
>
> 当然，如果你在/etc/hosts中进行了域名解析，是可以不用重复指定目标主机IP的

```bash
web01 ansible_ssh_host=10.0.0.7 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'
```

定义组

> [!IMPORTANT]
>
> 使用**中括号**定义组，没有分隔符，下面无论有多少回车，都会被定义在组中

```bash
[webs]
web01 ansible_ssh_host=10.0.0.7 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'
web02 ansible_ssh_host=10.0.0.8 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'
...中间无论有多少空行,下面的db01也被算在webs组中
db01 ansible_ssh_host=10.0.0.51 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'
```

假如一定要单独定义，可以写在最上方避免被包括进组

```bash
db01 ansible_ssh_host=10.0.0.51 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'
[webs]
web01 ansible_ssh_host=10.0.0.7 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'
web02 ansible_ssh_host=10.0.0.8 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'
```

使用all指代所有主机

```bash
ansible all -m ping
```

##### 使用序列进行定义

> [!NOTE]
>
> 这里需要事先在/etc/hosts中做好DNS解析

```bash
[webs]
web[01:02] ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'
```

##### 多组定义和子组

> [!NOTE]
>
> 此时，组lnmp就仅包含dbs与webs两个组

```properties
nfs ansible_ssh_host=10.0.0.31 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'

[dbs]
db01 ansible_ssh_host=10.0.0.51 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'

[webs]
web[01:02] ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'


[lnmp:children]
dbs
webs
```

##### 共同定义的变量

```properties
nfs ansible_ssh_host=10.0.0.31 ansible_ssh_user=root ansible_ssh_port=22 ansible_ssh_pass='oldboy123.com'

[dbs]
db01 ansible_ssh_host=10.0.0.51

[webs]
web[01:02]


[lnmp:children]
dbs
webs

[lnmp:vars]
ansible_ssh_user=root
ansible_ssh_port=22
ansible_ssh_pass='oldboy123.com'
# 也可以对所有的组进行定义
[all:vars]
ansible_ssh_user=root
ansible_ssh_port=22
ansible_ssh_pass='oldboy123.com'
```

可以用all对所有的主机进行测试

```bash
ansible all -m ping
```

## Ansible的Ad-Hoc与模块

目前学习的模块有以下8个

### file

​	**语法：**

```tex
file:
  path: file dir
  state:
  		touch		# 创建文件
  		directory	# 创建目录
  		absent		# 删除文件
  	owner:			# 定义属主
  	group:			# 定义属组
  	mode:			# 定义权限
```

- -m指定模块
- -a指定动作

```bash
ansible web01 -m file -a 'path=/root/a.txt state=touch owner=www group=www mode=0600'
```

### group

​	**语法：**

```tex
group:
  name:		# 指定组名
  gid:		# 指定组ID
```

```bash
ansible web01 -m group -a 'name=oldboy gid=777'
```

### user

​	**语法：**

```tex
user:
  name:				# 定义用户名
  uid:				# 定义uid
  group:			# 定义gid
  state:
    present			# 创建用户（在场）（默认）
    absent			# 删除用户（不在场）
  remove:
  	yes				# 是否删除home目录 等价于userdel -r
  shell:
    /sbin/nologin	# 虚拟用户
    /bin/bash		# 可登录（默认）
  create_home:
  	true			# 创建home目录（默认）
  	false			# 不创建home目录
```

创建指定uid、gid的虚拟用户

> [!TIP]
>
> 需要先创建名为oldboy的组

```bash
ansible webs -m user -a 'name=oldboy uid=777 group=oldboy state-present shell=/sbin/nologin create_home=false'
```

删除oldboy并且不保留home目录

```bash
ansible webs -m user -a 'name=oldboy state=absent remove=yes'
```

### systemd

​	**语法：**

```tex
systemd:
  name:			# 定义服务名称
  state:		
    started		# 启动服务
    stopped		# 停止服务
    restarted	# 重启服务
    reloaded	# 重载服务
  enabled:
    yes			# 开机自启
    no			# 关闭开机自启
```

停止Nginx服务

```bash
ansible webs -m systemd -a 'name=nginx state=stopped'
```

启动Nginx服务并设定为开机自启

```bash
ansible webs -m systemd -a 'name=nginx state=started enabled=yes'
```

### yum

​	**语法：**

```tex
yum:
  name:
    wget				# 服务名称
    xxx.rpm				# 本地rpm包（这里要写本地rpm包的绝对路径）
  state:
    present				# 安装
    absent				# 卸载
  download_only: true	# 只下载安装包
  download_dir: /opt	# 指定下载目录
```

安装wget

```bash
ansible webs -m yum -a 'name=wget state=present'
```

将安装包下载到指定位置

> [!CAUTION]
>
> 已经安装了服务的主机不会下载安装包

```bash
ansible webs -m yum -a 'name=wget download_only=true download_dir=/opt'
```

安装指定位置的安装包

```bash
ansible webs -m yum -a 'name=/opt/wget-1.20.3-6.ky10.x86_64.rpm state=present'
```

### command

**不使用command的原因**

#### ① 失去“幂等性” (Idempotency)

这是运维自动化的灵魂。

- **使用 `file` 模块：** `file: path=/tmp/test state=directory`。执行 100 次，只有第一次会创建目录，剩下 99 次 Ansible 都会检查目录已存在，不做任何修改。
- **使用 `command` 模块：** `command: mkdir /tmp/test`。执行第 2 次时，系统会报错 `File exists`，导致你的整个自动化剧本（Playbook）中断退出。

#### ② 无法识别“变更状态” (Changed)

Ansible 的精髓在于它能告诉你哪些机器发生了真正的改变。

- 专门模块会根据实际情况返回 `changed`（改了）或 `ok`（没动）。
- `command` 模块只要运行了命令，永远返回 `changed`，这让你根本无法判断服务器的真实状态是否发生了预期之外的波动。

#### ③ 缺乏安全性与跨平台性

- 专门模块会自动处理不同系统的差异（比如 `package` 模块在 CentOS 用 yum，在 Ubuntu 用 apt）。
- `command` 模块是硬编码的。你在 CentOS 上写的 `yum install` 命令，放到 Ubuntu 上就会直接崩溃。

​	**语法：**

```bash
ansible webs -m command -a 'df -h'
```

### copy

​	**语法：**

```tex
copy:
  src:		# 源文件（来自本机）
  dest:		# 目标路径（指定主机的目标路径）
  owner:	# 属主
  group:	# 属组
  mode:		# 权限，比如0644
  backup:	# 拷贝前是否需要备份（拷贝完毕会保留一份备份文件）
  content	# 将content后的内容写入到目标文件（会与src产生冲突）
```

拷贝文件

```bash
ansible webs -m copy -a 'src=a.txt dest=/root/'
```

拷贝前备份目标文件

```bash
ansible webs -m copy -a 'src=a.txt dest=/root/ backup=yes'
```

使用content生成文件并写入内容

> [!NOTE]
>
> 这里是直接在/root下生成了pass，并向里面写入了rsync_backup:123，而**不是**进行文件的拷贝

```bash
ansible webs -m copy -a 'content="rsync_backup:123" dest=/root/pass mode=0600'
```

### cron

> [!IMPORTANT]
>
> 这里是写入**用户**的定时任务，可以使用crontab -l查看，并且使用crontab -e进行编辑。与系统定时任务**不同**，用户定时任务默认路径只有usr/bin:/bin，所以部分命令可能不会执行

​	**语法：**

```tex
cron:
  name:		# 在生成的定时任务上添加注释
  state:	# 指定创建还是删除
  minute:	# 1-59分钟
  hour:		# 0-23小时
  job:		# 具体执行的命令
```

创建时间同步的任务

```bash
ansible webs -m cron -a 'name="时间同步" minute=* hour=* job="ntpdate ntp1.aliyun.com &> /dev/null"'
```

删纯时间同步任务

```bash
ansible webs -m cron -a 'name="时间同步" state=absent'
```

> [!WARNING]
>
> 上述命令中的state我虽然理解为状态，比如present（在场）与absent（不在场），还有file模块中的touch与directory，但是实际上已经touch的文件并不能更改为directory状态，所以还是理解为创建的命令吧
