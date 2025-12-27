# AnsiblePlaybook变量相关

## 定义变量

### 1.直接在play中定义

```yaml
# 第一层：Playbook 开始（列表项）
- name: 部署 Nginx 服务器             # 一个 Play 的开始
  hosts: web_servers                # 作用于哪些机器
  become: yes                        # 是否提权

  # 第二层：Play 的属性
  vars:
    http_port: 80

  # 第三层：任务列表
  tasks:
    - name: 安装 Nginx               # 这是一个具体的 Task
      yum:                           # 第四层：调用 yum 模块
        name: nginx                  # 模块参数
        state: present

    - name: 启动服务                  # 又一个 Task
      service:                       # 第四层：调用 service 模块
        name: nginx
        state: started
```

### 2.在文件中定义变量（也可以在多个文件中定义）

准备2个vars文件

```tex
vars1
pkg1: wget
pkg2: lrzsz
vars2
pkg3: bind-utils
pkg4: unzip
```

```yaml
- hosts: web01
  vars_files:	# vars_files固定写法
    - vars1		# 读取的这两个文件中的变量
    - vars2
  tasks:
    - name: Install wget&lrzsz
      ansible.builtin.yum:
        name: "{{ item }}"		# item循环固定写法
        state: present
      loop:
        - "{{ pkg1 }}"
        - "{{ pkg2 }}"
        - "{{ pkg3 }}"
        - "{{ pkg4 }}"
```

### 3.在主机清单中定义变量

/etc/ansible/hosts的主机清单中定义变量

```tex
nfs
[webs]
web[01:02]

[bak]
backup

[dbs]
db01 
db02

[lnmp:children]
webs
bak

[webs:vars]
pkg1=wget
pkg2=tree
```

```yaml
- hosts: webs
  tasks:
    - name: Install wget&lrzsz
      ansible.builtin.yum:
        name: "{{ item }}"
        state: present
      loop:
        - "{{ pkg1 }}"
        - "{{ pkg2 }}"
```

### 4.官方推荐定义变量方法

在当前运行playbook的目录下创建

#### group_vars：此目录中的文件用于给组定义变量

```bash
mkdir group_vars
touch group_vars/webs
```

webs中定义变量

```tex
pkg1: wget
pkg2: tree
```

也可以使用all为所有主机/组定义变量

```bash
touch group_vars/all
```

all中定义变量

```tex
pkg1: wget
pkg2: tree
```

#### host_vars：此目录中的文件用于给单台主机定义变量

```bash
mkdir host_vars
touch host_vars/nfs
```

在nfs中定义变量

```tex
pkg1: wget
pkg2: tree
```

此时只有主机清单中的nfs主机才能识别到这个变量

```yaml
- hosts: nfs		# 变量只对nfs主机生效
  tasks:
    - name: Install wget&tree
      ansible.builtin.yum:
        name: "{{ item }}"
        state: present
      loop:
        - "{{ pkg1 }}"
        - "{{ pkg2 }}"
```

## 变量注册

在执行某些命令时，我们看不到返回的结果是否表示执行成功，需要注册变量，将执行结果赋值给这个变量，再显示到本机

```yaml
- hosts: web01
  tasks:
    - name: list /root/ files
      command: 'ls -l /root/'
      register: list_file		# 注册了一个变量叫list_file

    - name: print list_files env
      debug:
        msg: "{{ list_file }}"	# 返回list_file携带的所以msg信息
```

```bash
TASK [print list_files env] *******************************************************************
ok: [web01] => {
    "msg": {
        "changed": true,
        "cmd": [
            "ls",
            "-l",
            "/root/"
        ],
        "delta": "0:00:00.005963",
        "end": "2025-12-26 17:42:09.867997",
        "failed": false,
        "msg": "",
        "rc": 0,
        "start": "2025-12-26 17:42:09.862034",
        "stderr": "",
        "stderr_lines": [],
        "stdout": "total 13644\n-rw-r--r-- 1 root root 13049663 Dec 16 15:51 apache-tomcat-9.0.113.tar.gz\ndrwxr-xr-x 3 root root       96 Dec 17 15:35 __MACOSX\ndrwxr-xr-x 4 root root       64 Aug 12  2019 tomcat-cluster-redis-session-manager\n-rw-r--r-- 1 root root   921429 Dec  8  2021 tomcat-cluster-redis-session-manager.zip",
        "stdout_lines": [
            "total 13644",
            "-rw-r--r-- 1 root root 13049663 Dec 16 15:51 apache-tomcat-9.0.113.tar.gz",
            "drwxr-xr-x 3 root root       96 Dec 17 15:35 __MACOSX",
            "drwxr-xr-x 4 root root       64 Aug 12  2019 tomcat-cluster-redis-session-manager",
            "-rw-r--r-- 1 root root   921429 Dec  8  2021 tomcat-cluster-redis-session-manager.zip"
        ]
    }
}
```

以上是返回的msg信息，不过我们可以通过层级关系获取指定的信息

### 层级获取信息

上文中，我们只想获取ls -l的执行结果，也就是stdout_lines中的信息，可以用以下方式来获取

```yaml
- hosts: web01
  tasks:
    - name: list /root/ files
      command: 'ls -l /root/'
      register: list_file

    - name: print list_files env
      debug:
        msg: "{{ list_file.stdout_lines }}"
```

可以看到只返回了这一条

```bash
TASK [print list_files env] *******************************************************************
ok: [web01] => {
    "msg": [
        "total 13644",
        "-rw-r--r-- 1 root root 13049663 Dec 16 15:51 apache-tomcat-9.0.113.tar.gz",
        "drwxr-xr-x 3 root root       96 Dec 17 15:35 __MACOSX",
        "drwxr-xr-x 4 root root       64 Aug 12  2019 tomcat-cluster-redis-session-manager",
        "-rw-r--r-- 1 root root   921429 Dec  8  2021 tomcat-cluster-redis-session-manager.zip"
    ]
}
```

### 层级定义变量

我们可以通过YAML语法的层级关系来定义变量，例如：

```yaml
lnmp:
  nginx:
    pkgs:
      pkg1: wget
      pkg2: tree
```

那么我们可以在playbook中使用层级定义的变量

```yaml
- hosts: nfs
  vars_files: lnmp
  tasks:
    - name: Install wget&tree
      ansible.builtin.yum:
        name: "{{ item }}"
        state: present
      loop:
        - "{{ lnmp.nginx.pkgs.pkg1 }}"
        - "{{ lnmp.nginx.pkgs.pkg2 }}"
```

不过假如变量名中带有**特殊字符**、**空格**、**连字符**等特殊字符，应该使用官方推荐的写法

```yaml
- hosts: nfs
  vars_files: lnmp
  tasks:
    - name: Install wget&tree
      ansible.builtin.yum:
        name: "{{ item }}"
        state: present
      loop:
        - "{{ lnmp['nginx']['pkgs']['pkg1'] }}"
        - "{{ lnmp['nginx']['pkgs']['pkg2'] }}"
```

携带特殊字符的变量名，例如：

```yaml
包含特殊字符的键名：lnmp['web-server']['php-version']
以数字开头的键名：lnmp['2nd-level']['key']
包含空格的键名：lnmp['my key']['sub key']
```

## Facts缓存

在执行Ansible-Playbook时，Ansible会首先收集目标主机的信息，这些信息可以用作**变量**进行调用

```tex
#Facts可以获取到客户端的信息赋值给相对的变量
ansible_processor_vcpus: cpu核
ansible_all_ipv4_addresses：仅显示ipv4的信息。
ansible_devices：仅显示磁盘设备信息。
ansible_distribution：显示是什么系统，例：centos,suse等。
ansible_distribution_major_version：显示是系统主版本。
ansible_distribution_version：仅显示系统版本。
ansible_machine：显示系统类型，例：32位，还是64位。
ansible_eth0：仅显示eth0的信息。
ansible_hostname：仅显示主机名。
ansible_kernel：仅显示内核版本。
ansible_lvm：显示lvm相关信息。
ansible_memtotal_mb：显示系统总内存。
ansible_memfree_mb：显示可用系统内存。
ansible_memory_mb：详细显示内存情况。
ansible_swaptotal_mb：显示总的swap内存。
ansible_swapfree_mb：显示swap内存的可用内存。
ansible_mounts：显示系统磁盘挂载情况。
ansible_processor：显示cpu个数(具体显示每个cpu的型号)。
ansible_processor_vcpus：显示cpu个数(只显示总的个数)。
```

可以在debug模块中调用msg输出对应的客户端信息

```yaml
- hosts: nfs
  tasks:
    - name: print os env
      debug:
        msg: "{{ ansible_distribution }}"		# 输出nfs主机的操作系统版本
```

```yaml
- hosts: nfs
  tasks:
    - name: print os env
      debug:
        msg: "{{ ansible_default_ipv4.address }}"	# 输出nfs主机的ipv4地址
```

还可以通过变量进行字符拼接

```yaml
- hosts: nfs
  tasks:
    - name: create hostname_ip dir
      ansible.builtin.file:
        path: /root/{{ ansible_hostname }}_{{ ansible_default_ipv4.address }}		# 用于拼接主机名+IP
        state: directory
```

## jinja2模板

### backup为例

可以先定义变量，再使用模板将变量填入文件中预先写好的占位符

以backup为例，先写rsyncd.conf.j2模板

```jinja2
uid = {{ user }}
gid = {{ user }}
port = {{ port }}
fake super = yes
use chroot = no
max connections = 200
timeout = 600
ignore errors = yes
read only = false
list = false
auth users = backup
secrets file = /etc/rsync.passwd

[backup]
path = {{ dir}}

[nfs]
path = {{ nfs }}

[web01]
path = {{ web01 }}

[web02]
path = {{ web02 }}

[data]
path = {{ data }}
```

写对应的yml文件

```yaml
- hosts: backup
  vars: 
    - user: www
    - port: 873
  tasks:
    - name: define dirs		# 定义data路径
      ansible.builtin.set_fact:
        dir: /backup
        data: /data

    - name : define subpath		# 使用定义好的data变量定义子路径
      ansible.builtin.set_fact:
        nfs: "{{ dir }}/nfs"
        web01: "{{ dir }}/web01"
        web02: "{{ dir }}/web02"

    - name: Install rsync Server
      ansible.builtin.yum:
        name: rsync
        state: present

    - name: Configure rsync Server
      ansible.builtin.template:
        src: /root/ansible/rsyncd.conf.j2
        dest: /etc/rsyncd.conf

    - name: Create User "{{ user }}"
      ansible.builtin.user:
        name: "{{ user }}"
        shell: /sbin/nologin
        create_home: false
        state: present

    - name: Configure passwd file
      ansible.builtin.copy:
        content: "rsync_backup:123"
        dest: /etc/rsync.passwd
        mode: 0600

    - name: Create directories
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: "{{ user }}"
        group: "{{ user }}"
      loop:
        - "{{ dir }}"
        - "{{ data }}"
        - "{{ nfs }}"
        - "{{ web01 }}"
        - "{{ web02 }}"

    - name: Start rsync Server
      ansible.builtin.systemd:
        name: rsyncd
        state: started
        enabled: true
```

### nfs为例

写exports.j2模板

```jinja2
{{ data }} {{ip}}(rw,sync,all_squash,anonuid={{ uid }},anongid={{ gid }})
{{ wp }} {{ip}}(rw,sync,all_squash,anonuid={{ uid }},anongid={{ gid }})
{{ zh }} {{ip}}(rw,sync,all_squash,anonuid={{ uid }},anongid={{ gid }})
{{ zrlog }} {{ip}}(rw,sync,all_squash,anonuid={{ uid }},anongid={{ gid }})
```

写对应的yml文件

```yaml
- hosts: nfs
  vars:
    - ip: 172.16.1.0/24
    - user: www
  tasks:
    - name: set data path
      ansible.builtin.set_fact: 
        data: /data

    - name: set subpath
      ansible.builtin.set_fact:
        wp: "{{ data }}/wp"
        zh: "{{ data }}/zh"
        zrlog: "{{ data }}/zrlog"

    - name: Install nfs Server
      ansible.builtin.yum:
        name: nfs-utils
        state: present

    - name: Create User "{{ user }}"
      ansible.builtin.user:
        name: "{{ user }}"
        shell: /sbin/nologin
        create_home: false
        state: present

    - name: Get user information
      ansible.builtin.getent:		# 使用getent获取系统数据库passwd中的user
        database: passwd
        key: "{{ user }}"

    - name: Set UID and GID facts
      ansible.builtin.set_fact:			# 获得user后使用层级信息定义uid和gid
        uid: "{{ getent_passwd[user][1] }}"
        gid: "{{ getent_passwd[user][2] }}"

    - name: Configure nfs Server
      ansible.builtin.template:
        src: /root/ansible/exports.j2
        dest: /etc/exports

    - name: Create "{{ dir }}"
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        owner: "{{ user }}"
        group: "{{ user }}"
      loop:
        - "{{ data }}"
        - "{{ wp }}"
        - "{{ zh }}"
        - "{{ zrlog }}"

    - name: Start nfs Server
      ansible.builtin.systemd:
        name: nfs
        state: started
        enabled: true
```





