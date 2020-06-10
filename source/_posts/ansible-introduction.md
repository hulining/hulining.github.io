---
title: Ansible 简单介绍
date: 2020/06/10
tags:
  - Ansible
categories:
  - Ansible
abbrlink: 
description: 简单介绍 Ansible 的基本部署,相关概念及相关模块的基本使用.简单介绍了 ansible-playbook 的使用方式
---

## 1 基本部署

ansible依赖于Python 2.6或更高的版本,paramiko,PyYAML及Jinja2

### 1.1 编译安装

```bash
# yum -y install python-jinja2 PyYAML python-paramiko python-babel python-crypto
# tar xf ansible-1.5.4.tar.gz
# cd ansible-1.5.4
# python setup.py build
# python setup.py install
# mkdir /etc/ansible
# cp -r examples/* /etc/ansible
```

### 1.2 rpm 包安装

```bash
# yum install epel-release
# yum install ansible
```

## 2 基本使用

ansible通过ssh实现配置管理,应用部署,任务执行等功能.因此,需要事先配置ansible端能基于密钥认证的方式联系各被管理节点

### 2.1 常用文件

```bash
# /etc/ansible/ansible.cfg 主配置文件
[defaults]
#inventory      = /etc/ansible/hosts    # 主机列表清单文件
#library        = /usr/share/my_modules/
#module_utils   = /usr/share/my_module_utils/
#remote_tmp     = ~/.ansible/tmp
#local_tmp      = ~/.ansible/tmp
#plugin_filters_cfg = /etc/ansible/plugin_filters.yml
#forks          = 5 # 默认并发数
#poll_interval  = 15
#sudo_user      = root  # 默认 sudo 用户
#ask_sudo_pass = True   # 是否需要询问sudo 用户密码
#ask_pass      = True
#transport      = smart
#remote_port    = 22    # 远程主机默认 ssh 端口
#module_lang    = C
#module_set_locale = False
host_key_checking = False # 是否检查主机的 host_key,如果设置为True,则需要在登录时输入 yes
log_path = /var/log/ansible.log # 日志文件

/etc/ansible/hosts          # Inventory
/usr/bin/ansible-doc        # 帮助文件
/usr/bin/ansible-playbook   # 指定运行任务文件
```

### 2.2 常用命令

```text
ansible-doc
    -l: 列出所有可用的模块
    -s <module_name>: 查看指定模块的使用帮助

ansible <host-pattern> [-m module_name] [-a args] [options]
    <host-pattern>  这次命令对哪些主机生效的
        all: /etc/ansible/hosts文件中所有主机
        group_name: /etc/ansible/hosts文件中定义的主机组
        ip
    -m module_name  要使用的模块
    -a args         模块特有的参数

  [options]
    -f forks        一次处理多少个主机
    -k, --ask-pass  指定连接密码
    -u REMOTE_USER, --user=REMOTE_USER  指定连接的远程用户
    -T TIMEOUT, --timeout=TIMEOUT   指定连接的超时时间
    -b, --become    指定执行命令的用户
    --become-user=USER   指定执行命令的用户,默认 root
    --become-method=BECOME_METHOD   指定变更用户的方式,如 sudo,su.默认 sudo
    -K, --ask-become-pass   指定改变用户的密码

    -l SUBSET, --limit=SUBSET 指定部分主机运行模块
    --list-hosts    列出匹配的主机列表

ansible-playbook [options] <playbook.yml> [playbook2 ...]
    运行Ansible playbools,在目标主机上执行已定义的任务
[options]
    --ask-sudo-pass: 询问sudo的密码
```

### 2.3 常用模块

模块执行是幂等的,这意味着多次执行是安全的,因为其结果均一致

- `command` 命令模块

```bash
# (默认模块)用于在远程主机执行命令;不能使用变量,管道等
# ansible all -a 'date'
```

- `archive` 打包模块

```bash
path    远程绝对路径,指定要打包的文件路径
exclude_path    排除的远程绝对路径
format  打包的方式,bz2, gz, tar, xz, zip,默认 gz
dest    打包后的文件路径
mode
```

- `copy` 复制文件模块(复制本地文件到远程主机的指定位置)

```bash
src     定义本地源文件路径
dest    定义远程目录文件路径(绝对路径)
owner   属主
group   属组
mode    权限

content 将内容直接输入到文件中

# ansible all -m copy -a 'src=/etc/fstab dest=/tmp/fstab.ansible owner=root mode=640'
```

- `cron` 计划任务

```bash
# 配置计划任务,其中job为必须参数
# 如果其它不写,默认为*
name    指定名称,最好指定,方便移除
minute  指定分钟
hour    指定小时
day     指定天
month   指定月份
weekday 表示周几
job     指定任务,必须
state   表示是添加还是删除
    present：创建计划任务
    absent：移除

# ansible webserver -m cron -a 'minute="10" job="/bin/echo hello" name="job_name"' # 创建任务,每小时的10分钟执行
# ansible webserver -m cron -a 'name="job_name" state=absent'  # 移除任务
```

- `file` 文件管理模块

```bash
path    指定远程主机上被管理的文件
group   指定文件属组
user    指定文件属组
mode    指定文件权限
state   表示添加/删除/创建链接文件
    absent: 递归删除文件/目录,并取消符号链接
    directory: 创建文件夹,相当于 mkdir -p <path>
    file: 返回文件的当前状态
    hard: 创建硬链接
    link: 创建软链接
    touch: 相当于 touch 命令

src     当state为hard或link时指定的链接文件位置
force   强制进行操作

# ansible all -m file -a 'path=/tmp/link_file src=/tmp/source_file state=link'
```

- `get_url` 从指定 url 下载文件

```bash
url     下载文件路径
dest    下载文件保存路径

mode    权限
timeout 连接超时时间

# ansible localhost -m get_url -a 'url=http://get_some_files dest=/tmp/ timeout=10 mode=+x'
```

- `group` 组管理模块

```bash
gid     gid
name    组名
state   状态,默认为present创建,使用absent删除
system  是否是系统组

# ansible webserver -m group -a 'name=mysql gid=306 system=yes'
# ansible webserver -m user -a 'name=mysql uid=306 system=yes group=mysql'
```

- `ini_file` 创建/修改 ini 文件中的配置

```bash
dest    ini 文件路径
section ini 文件内中部分的概念 "[]" 内内容
option  ini 文件配置
value   ini 文件配置值

# ansible localhost -m ini_file -a 'dest=/tmp/my.ini section=mysqlclient option=port value=3306'
```

- `lineinfile` 修改行格式的配置文件

```bash
path    指定配置文件路径
state   指定行的状态
regexp  指定行格式的正则表达式,使用正则表达式匹配需要修改的行
line    指定匹配行的字符串

backup  备份原有文件
create  如果指定文件不存在则创建

# ansible all -m lineinfile -a 'path=/tmp/sshd_config  regexp="^#UseDNS" line="UseDNS no"'
```

- `mount` 挂载管理模块

```bash
path    指定挂载路径
state   指定带挂载路径的状态
src     指定待挂载设备
opts    挂载的选项

backup  备份原有文件

# ansible all -m mount -a 'path=/mnt state=present fstype=ext4 src=/dev/sda2 opts="rw"'
```

- `service` 服务管理模块

```bash
name    指定服务名
enabled 是否开机自动启动
state   指定服务状态
    started     启动服务
    stoped      停止服务
    restarted   重启服务

# ansible webserver -m service -a 'name=httpd enabled=true state=started'
```

- `shell` 复杂命令模块

```bash
# 与command基本一致,支持管道,重定向等复杂命令
# ansible all -m shell -a 'echo magedu | passwd --stdin user1'
```

- `script` 脚本模块

```bash
# 使远程主机执行本地指定脚本
# ansible all -m script -a '/tmp/test.sh'
```

- `timezone` 设置时区

```bash
name    指定时区名称

# ansible all -m -a 'Asia/Shanghai'
```

- `user` 用户管理模块

```bash
name    用户名
uid     uid
state   状态,默认为present创建,使用absent删除
group   属于哪个组
groups  附加组
home    家目录
createhome  是否创建家目录
comment 注释信息
system  是否是系统用户

# ansible all -m user -a 'name="user1"'
# ansible all -m user -a 'name="user1" state=absent'
```

- `yum` yum管理模块

```bash
name    程序包名称(不指定版本就安装最新的版本latest)
state   present,latest表示安装,absent表示卸载

# ansible webserver -m yum -a 'name=httpd'
# ansible all -m yum -a 'name=httpd state=absent'
```

- `setup` 信息收集模块

```bash
# 每个被管理节点在接受并运行管理命令之前,会将自己主机相关信息,如操作系统版本,IP地址等报告给远程的 ansible 主机
# ansible all -m setup
```

- 更多模块参见[官方文档](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)

## 3 Ansible基础元素

### 3.1 Inventory

ansible 的主要用于批量主机操作,为了便捷地使用其中的部分主机,可以在 inventory file 中将其分组命名.默认的 inventory file 为`/etc/ansible/hosts`

inventory file 可以有多个,且也可以通过 Dynamic Inventory 来动态生成

#### 3.1.1 文件格式

inventory 文件遵循INI文件风格,中括号中的字符为组名,可以将同一个主机同时归并到多个不同的组中.此外,当如若目标主机使用了非默认的SSH端口,还可以在主机名称之后使用冒号加端口号来标明

ansible 中有两个默认的组,`all`和`ungrouped`

- `all` 表示所有主机组
- `ungrouped`表示不在自定义分组中的主机组

```bash
ntp.magedu.com # 该主机在ungrouped分组中

[webserver] # webserver组
www1.magedu.com:2222 # 指定端口为2222
www2.magedu.com

[dbserver]
db1.magedu.com
db2.magedu.com
db3.magedu.com

# 如果主机名遵循相似的命名模式,可使用列表的方式标识多个主机
[webserver]
www[01:50].example.com

[databases]
db-[a:f].example.com
```

#### 3.1.2 主机变量与组变量

```bash
# - 主机变量
[webserver]
www1.magedu.com http_port=80 maxRequestsPerChild=808
www2.magedu.com http_port=8080 maxRequestsPerChild=909

# - 组变量
[webserver]
www1.magedu.com
www2.magedu.com

[webserver:vars]
ntp_server=ntp.magedu.com
nfs_server=nfs.magedu.com
```

#### 3.1.3 组嵌套

```bash
[apache]
httpd1.magedu.com
httpd2.magedu.com

[nginx]
ngx1.magedu.com
ngx2.magedu.com

[webserver:children]
apache
nginx

[webserver:vars]
ntp_server=ntp.magedu.com
```

#### 3.1.4 inventory 参数

ansible 基于 ssh 连接 inventory 中指定的远程主机时,还可以通过参数指定其交互方式.这些参数如下所示

```text
ansible_ssh_port: 连接端口
ansible_ssh_user: 连接用户
ansible_ssh_pass: 连接用户密码
ansible_sudo_pass: 用户的sudo密码
ansible_connection: 连接方式,local, ssh or paramiko
ansible_ssh_private_key_file: ssh私钥文件
ansible_shell_type: 目标系统的shell类型
ansible_python_interpreter: 目标系统的python路径
```

### 3.2 变量

变量命名仅能由字母,数字和下划线组成,且只能以字母开头

#### 3.2.1 facts

facts 是由正在通信的远程目标主机发回的信息,这些信息被保存在 ansible 变量中.可使用 `setup` 模块对 facts 变量进行查看获取 `ansible hostname -m setup`

#### 3.2.2 通过命令行传递变量

在运行 playbook 的时候也可以使用 `--extra-vars` 传递一些变量供 playbook 使用,如 `ansible-playbook test.yml --extra-vars "hosts=www user=mageedu"`

#### 3.2.3 通过 roles 传递变量

当给一个主机应用角色的时候可以传递变量,然后在角色内使用这些变量.如

```yaml
- hosts: webservers
  roles:
  - common
  # 对foo_app_instance创建 dir,port 变量
  - { role: foo_app_instance, dir: '/web/htdocs/a.com',  port: 8080 }
```

#### 3.2.4 Inventory 文件中定义的变量

详见 3.1.2 主机变量与组变量 小节

#### 3.2.5 PlayBook 中使用 var 关键字定义的变量

```yaml
- hosts: webservers
  vars:
  - name: lilei
  - age: 18
  # ...
```

> 变量的优先级为: 命令行 `-e` -> playbook -> inventory 主机变量 -> `setup`

## 4 Playbook

playbook 是由一个或多个 "play" 组成的列表,play 的主要功能在于将事先归并为一组的主机装扮成事先通过 ansible 中的 task 定义好的角色.
从根本上来讲,所有 task 无非是调用 ansible 的一个 module,将多个 play 组织在一个 playbook 中,即可以让他们连同起来按事先编排的机制去完成某些任务.

```yaml
# playbook示例
- hosts: webservers
  vars:
    http_port: 80
    max_clients: 200
  remote_user: root
  tasks:
  - name: ensure apache is at the latest version
    yum: pkg=httpd state=latest
  - name: write the apache config file
    template: src=/srv/httpd.j2 dest=/etc/httpd.conf
    notify:
    - restart apache
  - name: ensure apache is running
    service: name=httpd state=started
  handlers:
    - name: restart apache
      service: name=httpd state=restarted
```

### 4.1 HostsUsers 主机与用户

ansible会使用 `<remote_user>` 定义的用户对 `<hosts>` 中的主机进行特定的Task操作.

`<hosts>` 用于指定要执行指定任务的主机,其可以是一个或多个由冒号分隔主机组;`<remote_user>`则用于指定远程主机上的执行任务的用户

```yaml
- hosts: webservers # 是一个由冒号分割的一个或多个主机组
  remote_user: root # 远程帐户
  sudo: yes # 支持sudo执行命令
  sudo_user: postgres # 支持sudo到不通用户身份执行命令
  become: yes # 是否支持切换用户
  become_user: postgres # 切换为指定用户
  become_method: su # 切换用户使用的方法

# 这里需要注意的是,我们在指定sudo密码时,需要使用 <ansible-playbook --ask-become-pass 或 -K> 指定密码
```

在每一个task中,也可以单独定义远程用户

```yaml
- hosts: webservers
  remote_user: root
  tasks:
    - service: name=nginx state=started
      remote_user: nginx
      sudo: yes
```

### 4.2 Task列表

每一个play中包含了一个 Task 列表,所有的 host 会获取到相同顺序(按照定义的顺序)的 Task 指令.一个 task 在其所对应的所有主机上执行完毕之后,下一个 task 才会执行

当运行PlayBook时,具有失败任务的主机将从整个 PlayBook 的调度轮询中删除.如果有失败,所有已执行任务都将回滚,所以只需要更正 PlayBook 并重新运行即可.且模块执行是幂等的,多次运行的结果一致,这意味着多次执行是安全的.

每个任务的目标都是执行一个带有非常具体参数的模块,常见的有 shell, serivce, script, copy 等

```yaml
tasks:
  - name: <task_name>
    <module_name>: key=value # 这是是在 ansible 中指定的模块名及其子选项的名称及参数
    ignore_errors: True # 使用ignore_errors忽略错误
# 或
tasks:
  - name: <task_name>
    <module_name>:
      key: value
```

### 4.3 Handlers 事件处理机制

用于当关注的资源发生变化时采取一定的操作

```yaml
# 当发生改动时,notify会在playbook的每一个task结束时被触发,notify只会被触发一次
# 在名为"write the apache config file"的Task执行完成后会启动notify机制,调用名为"restart apache"的handlers来进行下一步处理任务

- hosts: webservers
  remote_user: root
  tasks:
  - name: write the apache config file
    template: src=/srv/httpd.j2 dest=/etc/httpd.conf
    notify:
    - restart apache
  handlers:
    - name: restart apache
      service: name=httpd state=restarted
```

在Ansible2.2版本之后,`handlers` 可以使用 `listen`关键字监听普通主题,任务等

只需要tasks中notify的内容与 `handlers` 中 `listen` 的的内容一致即可

```yaml
handlers:
  - name: restart memcached
    service:
      name: memcached
      state: restarted
      listen: "restart web services"
  - name: restart apache
    service:
      name: apache
      state: restarted
      listen: "restart web services"

tasks:
    - name: restart everything
      command: echo "this task will restart the web services"
      notify: "restart web services"
```

## 5 高级用法

### 5.1 模版(template/Jinja2)

正如变量部分中已经引用的那样,Ansible使用Jinja2模板来启用动态表达式和访问变量.主要用于动态修改配置文件,动态配置相关参数等场景.

Jinja2中使用 `{{ var_name }}` 来引用变量,包括 3.2 变量 小节中介绍的所有方式定义的变量.

同时,Jinja2 中变量支持条件测试,循环引用,格式化输出等高级用法.

```yaml
# 使用 template 关键字指定模版文件,并将变量替换后的文件复制到远程主机上
- name: install httpd.conf
  template: src=httpd.conf dest=/etc/httpd/conf/httpd.conf
  notify:
  - restart httpd
```

### 5.2 条件测试

在task后添加when子句即可使用条件测试,when语句支持Jinja2表达式语法.

```yaml
tasks:
- name: "shutdown Debian flavored systems"
  command: /sbin/shutdown -h now
  when: ansible_os_family == "Debian"
```

when语句中还可以使用 Jinja2 的大多[filter](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html).

例如要忽略此前某语句的错误并基于其结果(fail或success)运行后面指定的语句,可使用类似如下形式:

```yaml
tasks:
- command: /bin/false
  register: result
  ignore_errors: True
- command: /bin/something
  when: result|failed
- command: /bin/something_else
  when: result|success
- command: /bin/still/something_else
  when: result|skipped
```

更多条件测试用法,详见官方文档 [Tests](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tests.html) 小节

### 5.3 迭代

当有需要重复性执行的任务时,可以使用迭代机制.其使用格式为将需要迭代的内容定义为item变量引用,并通过`with_items`语句来指明迭代的元素列表即可

> 在 ansible 2.5 之后,引入了 `loop` 关键字,并作为官方推荐的迭代关键字选择

```yaml
- name: add several users
  user: name={{ item }} state=present groups=wheel
  with_items:
  - testuser1
  - testuser2

# 两种定义方式是等价的

- name: add user testuser1
  user: name=testuser1 state=present groups=wheel
- name: add user testuser2
  user: name=testuser2 state=present groups=wheel
```

`with_items`中可以使用元素还可为hashes(字典)

```yaml
- name: add several users
  user: name={{ item.name }} state=present groups={{ item.groups }}
  with_items:
    - { name: 'testuser1', groups: 'wheel' }
    - { name: 'testuser2', groups: 'root' }
```

关于迭代的高级功能,详见官方文档 [loop](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html) 小节

### 5.4 Tags 标签

ansible playbook 应用于有一个大型的 playbook,但仅仅运行指定的 tasks 的场景.在执行playbook时,可以使用 `--tags` 或 `--skip-tags` 运行或不运行带有指定标签的 tasks

```yaml
tasks:
- yum:
    name: "{{ item }}"
    state: present
  loop:
  - httpd
  - memcached
  tags:
  - packages

- template:
    src: templates/httpd.j2
    dest: /etc/httpd.conf
  tags:
  - config

- template:
    src: templates/memcached.j2
    dest: /etc/memcached.conf
  tags:
  - config # tags 支持复用
```

- `always` 标签默认每次都会执行,除非使用 `--skip-tags always` 特别指定跳过该tag标记的task
- `never`标签默认每次都不会执行,除非使用 `--tags never` 特别指定运行该tag标记的task

### 5.5 Roles

ansilbe自1.2版本引入的新特性,用于层次性,结构化地组织playbook

简单来讲,roles 就是通过分别将变量,文件,任务,模块及处理器放置于单独的目录中,并根据层次型结构自动装载它们.

```bash
# roles 的目录结构
├── site.yml        # 主文件,其中定义了主机及角色
└── roles           # roles目录,定义了角色目录
    ├── fooservers  # fooserver角色目录
    │   ├── files   # 静态文件位置
    │   ├── handlers # 至少有 main.yml 定义事件触发/处理机制
    │   ├── tasks   # 至少有 main.yml 定义task相关
    │   ├── templates # 动态模版文件位置
    │   └── vars    # 至少有 main.yml 定义变量
    └── webservers
        ├── files
        ├── handlers
        ├── tasks
        ├── templates
        └── vars
```

roles 下的目录架构遵循以下规则

- 目录名必须与角色名相同
- 角色目录下的目录有固定格式及作用
- 与 roles 同级的目录下,一般使用 site.yml 定义playbook.

以下是roles的一个使用示例:

```yaml
# site.yml
- hosts: <host_group>
  remote_user: <user>
  # ...
  roles:
  - webservers

# roles/fooservers/files/httpd.conf 保存静态文件

# roles/fooservers/tasks/main.yml
- name: install httpd service
  yum: name=httpd state=latest
- name: install httpd.conf
  copy: src=httpd.conf dest=/etc/httpd/conf/httpd.conf
  notify:
  - restart httpd
- name: start httpd service
  serivce: name=httpd state=started enabled=yes
  
# roles/fooservers/handlers/main.yml
- name: restart httpd
  service: name=httpd state=restarted

# roles/fooservers/vars/main.yml
var_name: var_value
```
