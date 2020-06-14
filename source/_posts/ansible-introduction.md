---
title: Ansible 用户指南
date: 2020/06/10
tags:
  - Ansible
categories:
  - Ansible
abbrlink: 
description: 本文主要介绍 Ansible 的基本部署,相关概念及相关模块的基本使用.介绍了 ansible-playbook 的使用方式.
---

## 基本部署

ansible依赖于Python 2.6或更高的版本,paramiko,PyYAML及Jinja2

### 编译安装

```bash
# yum -y install python-jinja2 PyYAML python-paramiko python-babel python-crypto
# tar xf ansible-1.5.4.tar.gz
# cd ansible-1.5.4
# python setup.py build
# python setup.py install
# mkdir /etc/ansible
# cp -r examples/* /etc/ansible
```

### rpm 包安装

```bash
# yum install epel-release
# yum install ansible
```

## 基本使用

ansible 通过 ssh 实现配置管理,应用部署,任务执行等功能.因此,需要事先配置 ansible 端能基于密钥认证的方式联系各被管理节点

### 配置文件

如下为 `/etc/ansible/ansible.cfg` 配置文件常用选项及含义

```ini
[defaults]
#inventory      = /etc/ansible/hosts    # 主机列表清单文件
#remote_tmp     = ~/.ansible/tmp    # 远程主机的工作目录,相关文件会被复制过来,等执行结束后删除
#local_tmp      = ~/.ansible/tmp    # 本地工作目录
#forks          = 5 # 默认并发数
#sudo_user      = root  # 默认 sudo 用户
#ask_sudo_pass = True   # 是否需要询问sudo 用户密码
#ask_pass      = True
#transport      = smart # 传输/连接模式
#remote_port    = 22    # 远程主机默认 ssh 端口
#roles_path    = /etc/ansible/roles  # 默认的 roles 目录
#timeout = 10 # 默认的 SSH 登录超时时间
log_path = /var/log/ansible.log # 日志文件
#deprecation_warnings = True # 是否输出选项/模块/功能过期信息

# 是否启用 SSH 连接管道,开启此选项可以减少在远程服务器上执行模块所需要的 SSH 连接数量,可以提升性能.
# 若开启,使用 sudo 时,需要在 `/etc/sudoers` 禁用 "requiretty"
pipelining = True

# 是否检查主机的 host_key,如果设置为 True,则需要在登录时输入 yes
host_key_checking = False

# facts 收集配置
# - smart 表示默认收集,但是如果已经收集了,则不再进行收集.此选项可将 facts 缓存到文件或 redis 中,而不用每次都进行收集
# - implicit 表示始终收集,除非使用 `gather_facts: False`
# - explicit 表示不进行收集,除非显示指定 `gather_facts: True`
#gathering = implicit

# facts 缓存配置,默认保存在内存中.也可设置为可持久化的配置(需要同时设置 `fact_caching_connection` 配置),如下
# - redis: 保存到 redis 中(需要安装 redis 连接工具 `pip install redis`)
# - jsonfile: 保存到 json 文件中
#fact_caching = memory

# 保存/读取缓存的位置
# redis: `host:port:database`
# jsonfile: `/path/to/jsonfile`
#fact_caching_connection = /tmp
```

### 常用命令

- `ansible-doc`: ansible 模块的帮助文档

```text
ansible-doc
    -l: 列出所有可用的模块
    -s <module_name>: 查看指定模块的使用帮助
```

- `ansible`: ansible 执行单个任务,

```text
ansible <host-pattern> [-m module_name] [-a args] [options]
  <host-pattern>  这次命令对主机清单中哪些主机生效,以下均支持正则表达式匹配,以 `~` 开头
    `all` 或 `*`: 清单文件中所有主机
    `host`: 单个主机,使用 `,` 或 `:` 分割可指定多个主机
    `group`: 单个主机组,使用 `,` 或 `:` 分割可指定多个主机组
    `group[index]`: 表示主机组中指定索引的主机,与 Python 中列表使用方式相同
    `group_1:!group_2`: 使用 `!` 表示在主机组 group_1 中,而不在主机组 group_2 中
    `group_1:&group_2`: 使用 `&` 表示 group_1,group_2 主机组中共同包含的主机

    -m module_name  要使用的模块
    -a args         模块特有的参数

  [options]
    --ask-vault-pass    询问加密后的密码
    --vault-password-file   加密后的密码文件
    -e EXTRA_VARS, --extra-vars EXTRA_VARS  传入 `key=value` 或 json 格式的变量.或包含变量的 YAML/JSON 文件,需要在指定的文件名前加上 `@` 符号
    -f FORKS, --forks FORKS     一次处理多少个主机,默认为 5
    -i INVENTORY, --inventory INVENTORY     指定主机清单文件,默认为 `/etc/ansible/hosts`
    -k, --ask-pass  指定连接密码
    -u REMOTE_USER, --user=REMOTE_USER  指定连接的远程用户,默认为当前用户
    -T TIMEOUT, --timeout=TIMEOUT   指定连接的超时时间,默认为 10
    -b, --become    指定执行命令的用户
    -K, --ask-become-pass   指定改变用户的密码
    --become-method=BECOME_METHOD   指定变更用户的方式,可使用 `ansible-doc -t become -l` 查看.默认 sudo
    --become-user=USER   指定执行命令的用户,默认 root

    -l SUBSET, --limit=SUBSET 指定部分主机运行模块
    --list-hosts    列出匹配的主机列表
    --check     以检查模式运行,不做任何修改,尝试预测发生的改变
```

- `ansible-playbook`: 运行 Ansible playbools,在目标主机上执行已定义的任务

```text
ansible-playbook [options] <playbook.yml> [playbook2 ...]

  [options] 支持以上 ansible 的列出的所有选项
  --list-tags       列出 Playbook 中所有 Tag
  --list-tasks      列出 Playbook 中所有 Tasks
  --skip-tags SKIP_TAGS     执行 Playbook 时,跳过带有指定 Tag 的 Task
  -t TAGS, --tags TAGS      只执行指定 Tag 的 Task
  --vault-id VAULT_IDS      指定要使用的 vault id
  --vault-password-file     指定 vault 密码文件
```

更多命令及参数,详见官方文档 - [Working with command line tools](https://docs.ansible.com/ansible/latest/user_guide/command_line_tools.html).

### 常用模块

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

- `debug` 执行时打印信息模块

```bash
msg     要打印的信息
var     要打印的变量名,与 msg 互斥
```

- `fail` 自定义失败信息模块

```bash
msg     自定义的失败信息

# 一般与 when 一起使用
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

更多模块及其参数,详见官方文档 - [Module Index](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html)

## Inventory 主机清单

ansible 的主要用于批量主机操作,为了便捷地使用其中的部分主机,可以在 inventory file 中将其分组命名.默认的 inventory file 为`/etc/ansible/hosts`

inventory file 可以有多个,且也可以通过 Dynamic Inventory 来动态生成

### 文件格式

inventory 文件遵循INI文件风格,中括号中的字符为组名,可以将同一个主机同时归并到多个不同的组中.此外,当如若目标主机使用了非默认的SSH端口,还可以在主机名称之后使用冒号加端口号来标明.示例如下:

```text
ungrouped_host

[group1]
host1
host2:222
```

### 主机和组

ansible 中有两个默认的组,`all` 和 `ungrouped`

- `all` 表示所有主机组
- `ungrouped` 表示不在自定义分组中的主机组

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

### 组嵌套

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

### 主机变量与组变量

```bash
# - 主机变量
[webserver]
www1.magedu.com http_port=80 maxRequestsPerChild=808
www2.magedu.com http_port=8080 maxRequestsPerChild=909

[webserver]
www1.magedu.com
www2.magedu.com

# - 组变量
[webserver:vars]
ntp_server=ntp.magedu.com
nfs_server=nfs.magedu.com
```

除了在主机清单文件中定义变量外,ansible 还支持在如下位置定义变量

- 在 `/etc/ansible/group_vars/` 目录下以 '.yml', '.yaml' 或 '.json' 格式定义组变量
- 在 `/etc/ansible/host_vars/` 目录下以 '.yml', '.yaml' 或 '.json' 格式定义主机变量

> 主机清单中变量优先级从低到高为 `all group -> parent group -> child group -> host`

### inventory 参数

ansible 连接主机时,可指定连接主机的相关参数.

- `ansible_connection`: 连接主机方式,可选为 `smart`, `ssh` 或 `paramiko`.默认是 `smart`
- `ansible_host`: 连接主机名
- `ansible_port`: 连接端口,默认 22
- `ansible_user`: 连接用户
- `ansible_password`: 连接密码(不要以明文方式存储,使用 vault 进行存储,详见 [Variables and Vaults](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html#best-practices-for-variables-and-vaults)

当连接方式为 `ssh` 时,可指定如下 SSH 参数:

- `ansible_ssh_private_key_file`: ssh 私钥文件
- `ansible_ssh_pipelining`: 是否使用 ssh pipelining

权限升级相关参数如下:

- `ansible_become`: 允许强制特权升级
- `ansible_become_user`: 使用指定用户执行模块
- `ansible_become_method`: 使用指定方法变为指定用户
- `ansible_become_password`: 变为指定用户过程中需要的用户密码

远程主机相关参数如下:

- `ansible_shell_type`: 目标系统的 shell 类型
- `ansible_python_interpreter`: 目标系统的python路径

#### No-SSH 连接参数

- `local`: 仅在设备本地执行模块

## Playbook

### 简介

Playbook 是由一个或多个 "play"(剧本) 组成的列表,剧本的主要功能在于将事先归并为一组的主机装扮成事先通过 ansible 中的 task 定义好的角色.

从根本上来讲,所有 task 无非是调用 ansible 的一个 module,将多个剧本组织在一个 Playbook 中,即可以让他们连同起来按事先编排的机制去完成某些任务.

Playbook 使用 yaml 编写.示例如下

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

#### 基本元素

##### 主机与用户

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

# 这里需要注意的是,我们在指定sudo密码时,需要使用 `ansible-playbook --ask-become-pass 或 -K` 指定密码
```

在每一个 task 中,也可以单独定义远程用户

```yaml
- hosts: webservers
  remote_user: root
  tasks:
    - service: name=nginx state=started
      remote_user: nginx
      sudo: yes
```

##### Task 列表

每一个play中包含了一个 Task 列表,所有的 host 会获取到相同顺序(按照定义的顺序)的 Task 指令.

> 一个 Task 在其所对应的所有主机上执行完毕之后,下一个 Task 才会执行

当运行 Playbook 时,具有失败任务的主机将从整个 Playbook 的调度轮询中删除.如果有失败,所有已执行任务都将回滚,所以只需要更正 Playbook 并重新运行即可.且模块执行是幂等的,多次运行的结果一致,这意味着多次执行是安全的.

每个任务的目标都是执行一个带有非常具体参数的模块,常见的有 `shell, serivce, script, copy` 等

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

#### Handlers 事件处理机制

Handler 用于当关注的资源发生变化时采取一定的操作.

当带有 `notify` 关键字的 Task 执行前后状态发生改动时,`notify` 关键字指定的 handlers 将被被触发,且只会被触发一次.

```yaml
# 如在名为 "write the apache config file" 的 Task 执行完成后会启动 notify 机制,触发名为 "restart apache" 的 handlers 来进行下一步处理任务
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

在 Ansible 2.2 版本之后,handlers 可以使用 `listen` 关键字监听普通主题,任务等.只需要 tasks中 `notify` 的内容与 handlers 中 `listen` 的的内容一致即可.(可一次性触发多个 handlers)

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

### 创建可复用的 Playbook

#### `include` 和 `import`

`include` 和 `import`(在 Ansible 2.4 版中添加)允许用户将大的剧本拆分成较小的剧本文件.使用 `include` 和 `import` 使得剧本文件分工更加明确,重用性更高.

- 所有 `include*` 语句均在执行 Playbook 时进行处理
- 所有 `import*` 语句均在解析 Playbook 时进行处理,就好像它本身就是定义在那里一样.

示例如下:

```yaml
# 支持使用 import 或 include 直接导入 playbook 或 tasks
- import_playbook: common_playbook.yml
- import_playbook: common_playbook.yml
- import_tasks: common_tasks.yml
- include_tasks: common_tasks.yml
```

#### Role 角色

Role 是 ansilbe 1.2 版本引入的新特性,用于层次性,结构化地组织 Playbook

简单来讲,roles 就是通过分别将变量,文件,任务,模块及处理器放置于单独的目录中,并根据层次型结构自动装载它们.

Ansible 将通过以下方式搜索我们定义的 Roles

- Playbook 同级的 `roles/` 目录下
- 默认的 `/etc/ansible/roles` 目录下

Role 的结构大概如下

```bash
# roles 的目录结构
├── site.yml        # `ansible-playbook` 命令执行的主文件,其中定义了主机及其角色
└── roles           # roles 目录,定义了角色目录
    ├── fooservers  # fooserver 角色目录,目录名即为角色名
    │   ├── files   # 静态文件位置,可以包括 copy 或 script 中定义的文件
    │   ├── handlers # 至少有 main.yml 定义事件触发/处理机制
    │   ├── tasks   # 至少有 main.yml 定义task相关
    │   ├── templates # 动态模版文件位置
    |   |—— defaults # 定义默认变量
    |   |—— vars # 自定义变量位置
    │   └── meta    # 定义该角色的元数据信息
    └── webservers
        ├── files
        ├── handlers
        ├── tasks
        ├── templates
        └── vars
```

以下是 roles 的一个使用示例:

```yaml
# site.yml
- hosts: <host_pattern>
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

### 变量

变量命名仅能由字母,数字和下划线组成,且只能以字母开头

#### 变量定义

以下是变量定义的几种方式

##### 在 Inventory 文件中定义变量

详见[主机变量与组变量](#主机变量与组变量)小节

##### 在 PlayBook 中使用 var 关键字定义的变量

```yaml
# site.yml
- hosts: webservers
  vars:
  - http_port: 80
  # ...
```

##### facts 变量发现

facts 是由正在通信的远程目标主机发回的信息,这些信息被保存在 `ansible_facts` 变量中.可使用 `setup` 模块对 facts 变量进行查看获取 `ansible hostname -m setup`

假如您不需要任何有关主机的 facts 变量,则可以在 Playbook 中禁用 facts 变量收集.如下

```yaml
- hosts: example
  gather_facts: no
```

另外,可在[配置文件](#配置文件)中查看有关变量缓存的相关配置.

##### 注册变量

可在 Task 中运行命令并通过 `register` 关键字将该命令的结果注册为变量,供以后使用(多用于[条件判断](#条件判断)中).

如下为注册变量使用示例

```yaml
- hosts: web_servers
  tasks:
     - shell: /usr/bin/foo
       register: foo_result
       ignore_errors: True
     - shell: /usr/bin/bar
       when: foo_result.rc == 5
```

可通过[注册变量结果](#注册变量结果) 查询可用的变量结果.

##### 通过命令行传递变量

在运行 Playbook 的时候也可以使用 `--extra-vars` 传递一些变量供 Playbook 使用.

`--extra-vars` 参数支持如下格式

- 键值对,如 `key=value`
- json
- json 或 yaml 文件,需要在文件名前添加 `@`

```bash
ansible-playbook release.yml --extra-vars "version=1.23.45 other_variable=foo"
ansible-playbook release.yml --extra-vars '{"version":"1.23.45","other_variable":"foo"}'
ansible-playbook release.yml --extra-vars "@some_file.json"
ansible-playbook release.yml --extra-vars "@some_file.yml"
```

##### 通过 roles 传递变量

当给一个主机应用角色的时候可以传递变量,然后在角色内使用这些变量.如

```yaml
- hosts: webservers
  roles:
  - common
  # 对foo_app_instance创建 dir,port 变量
  - { role: foo_app_instance, dir: '/web/htdocs/a.com',  port: 8080 }
```

#### 变量使用

##### 通过 Jinja2 及其 filter 使用

定义变量后,可使用 [Jinja2](https://jinja.palletsprojects.com/) (或[中文文档地址](http://docs.jinkan.org/docs/jinja2/)) 模版系统在 Playbook 中使用它们.

如下是一个简单示例:

```yaml
- hosts: app_servers
  vars:
    app_path: "{{ base_path }}/22"
```

更多使用方式,参见[模版(Jinja2)](#模版(Jinja2))部分.

Jinja2 filter 可以在模版表达式中转换变量的值.Jinja2 包含了许多[内置过滤器](http://jinja.pocoo.org/docs/templates/#builtin-filters),同时 Ansible 提供了更多过滤器,参见官方文档 - [Filters](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html)

#### 变量的优先级

以下为常见变量的优先级(从低到高,高优先级会覆盖低优先级):

1. 环境变量
2. role default: 角色中定义的默认变量
3. inventory file or script group vars: 主机清单中定义的主机组变量
4. inventory group_vars/all: 主机清单目录下的 `group_vars/all`
5. playbook group_vars/all: playbook 目录下的 `group_vars/all`
6. inventory group_vars/*: 主机清单目录下的 `group_vars/*`
7. playbook group_vars/*: playbook 目录下的 `group_vars/*`
8. inventory file or script host vars: 主机清单中定义的主机变量
9. inventory host_vars/*: 主机清单目录下的 `host_vars/*`
10. playbook host_vars/*: playbook 目录下的 `host_vars/*`
11. host facts / cached set_facts: 主机 facts 或 facts 缓存中的变量
12. play vars: 在 play 中定义的全局变量,如 `roles/x/tasks/main.yml` 中全局 `vars`
13. play vars_prompt: 在 play 中定义的全局变量,如 `roles/x/tasks/main.yml` 中全局 `vars_prompt`
14. play vars_files: 在 play 中定义的全局变量,如 `roles/x/tasks/main.yml` 中全局 `vars_files`
15. role vars (defined in role/vars/main.yml): 定义在 `roles/x/vars/main.yml` 的变量
16. block vars (only for tasks in block): 定义在 `roles/x/tasks/main.yml` 单个 `block` 中的变量,仅用于此 `block`
17. task vars (only for the task): 定义在 `roles/x/tasks/main.yml` 单个 `task` 中的变量,仅用于此 `task`
18. include_vars
19. set_facts / registered vars: 在 Task 中注册的变量
20. role (and include_role) params: 通过 playbook 传入的参数
21. include params
22. extra vars: 通过 `--extra-vars` 命令行参数传入的变量

### 模版(Jinja2)

正如[变量使用](#变量使用)部分中已经介绍的那样,Ansible 使用 Jinja2 模板来启用动态表达式和访问变量.主要用于动态修改配置文件,动态配置相关参数等场景.

Jinja2 中使用 {% raw %}`{{ var_name }}`{% endraw %} 来引用变量,包括 [变量定义](#变量定义) 小节中介绍的所有方式定义的变量.

```yaml
# 使用 template 关键字指定模版文件,并将变量替换后的文件复制到远程主机上
- name: install httpd.conf
  template: src=httpd.conf.j2 dest=/etc/httpd/conf/httpd.conf
  notify:
  - restart httpd
```

同时,Jinja2 中变量支持过滤(Filter),条件测试(Tests),循环引用(Lookups)等高级用法.

#### Filters

详见官方文档 - [Filters](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html)

#### 条件测试

##### 字符串

支持使用 `match`, `search`, `regex` 关键字将字符串与子字符串或正则表达式进行匹配:

```yaml
vars:
  url: "http://example.com/users/foo/resources/bar"

tasks:
  - debug:
    msg: "matched pattern 1"
    when: url is match("http://example.com/users/.*/resources/.*")

  - debug:
    msg: "matched pattern 2"
    when: url is search("/users/.*/resources/.*")

  - debug:
    msg: "matched pattern 3"
    when: url is search("/users/")

  - debug:
    msg: "matched pattern 4"
    when: url is regex("example.com/\w+/foo")
```

##### 版本比较

支持使用 `version`(ansible 2.5 之后版本,之前使用 `version_compare`) 对版本号进行检查.如:

```yaml
{{ ansible_facts['distribution_version'] is version('12.04', '>=') }}
```

如吓是版本比较支持的操作符:

```text
<, lt, <=, le, >, gt, >=, ge, ==, =, eq, !=, <>, ne
```

##### 子集

支持使用 `subset`, `superset` 查看一个集合是否是另一个集合的子集或包含另一个子集.如:

```yaml
vars:
  a: [1,2,3,4,5]
  b: [2,3]
tasks:
  - debug:
      msg: "A includes B"
    when: a is superset(b)

  - debug:
      msg: "B is included in A"
    when: b is subset(a)
```

##### all/any 判断

支持对列表进行 `all, any` 判断(所有值均为 True,或任意值为 True)

```yaml
vars:
  mylist:
    - 1
    - "{{ 3 == 3 }}"
    - True
  myotherlist:
    - False
    - True
tasks:
  - debug:
      msg: "all are true!"
    when: mylist is all
  - debug:
      msg: "at least one is true"
    when: myotherlist is any
```

##### 路径判断

支持对指定路径进行判断

```yaml
- debug:
    msg: "path is a directory"
  # 路径是 目录,文件,链接,是否存在,挂载点
  when: mypath is directory | file | link | exists |mount

- debug:
    msg: "path is {{ (mypath is abs)|ternary('absolute','relative')}}"

- debug:
    msg: "path is the same file as path2"
  when: mypath is same_file(path2)
```

##### 结果判断

支持对结果进行判断

```yaml
tasks:
  - shell: /usr/bin/foo
    register: result
    ignore_errors: True
  - debug:
      msg: "it failed"
    # 结果是 失败,改变(用于触发),成功,跳过
    when: result is failed | changed | succeeded | success | skipped
```

##### 变量定义判断

支持对指定变量是否定义进行判断

```yaml
tasks:
    - shell: echo "I've got '{{ foo }}' and am not afraid to use it!"
      # 判断变量 foo 定义或没有定义
      when: foo is defined | undefined
```

#### 查找与迭代

当有需要重复性执行的任务时,可以使用迭代机制.其使用格式为将需要迭代的内容定义为变量,并通过 `with_<lookup>` 语句来指明迭代的元素列表即可.`with_<lookup>` 中支持 hashes(字典) 元素

```yaml
- name: add several users
  user: name={{ item }} state=present groups=wheel
  with_items:
  - testuser1
  - testuser2

- name: add several users
  user: name={{ item.name }} state=present groups={{ item.groups }}
  with_items:
    - { name: 'testuser1', groups: 'wheel' }
    - { name: 'testuser2', groups: 'root' }
```

> 在 ansible 2.5 之后,引入了 `loop` 关键字,并作为官方推荐的迭代关键字选择.

关于迭代的高级功能,详见官方文档 - [Loops](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html),或后面的章节 [循环与迭代 Loops](#循环与迭代-Loops)

### `when` 条件判断

`when` 条件判断句用于在特定条件下做某些操作.如

- 执行一个 Task
- 导入一个 Task 文件
- 将角色分配给主机

`when` 条件判断句支持使用 [Filters](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html) 或 [Tests](https://docs.ansible.com/ansible/latest/user_guide/playbooks_tests.html) [条件测试](#条件测试)作为条件判断的依据,当结果为 True 时,执行某些 Tasks.

`when` 条件判断句支持 `and`, `or` 或 `not` 关键字作为条件子句间逻辑关系的判断.同时支持以列表形式指定多个条件子句.列表形式子句间的逻辑关系为 `and`.

示例如下:

```yaml
- hosts: webservers
  roles:
     - role: debian_stock_config
       when: ansible_facts['os_family'] == 'Debian'
# 多条件子句
tasks:
  - name: "shut down CentOS 6 and Debian 7 systems"
    command: /sbin/shutdown -t now
    when: (ansible_facts['distribution'] == "CentOS" and ansible_facts['distribution_major_version'] == "6") or
          (ansible_facts['distribution'] == "Debian" and ansible_facts['distribution_major_version'] == "7")
# 列表形式指定多个条件子句
tasks:
  - name: "shut down CentOS 6 systems"
    command: /sbin/shutdown -t now
    when:
      - ansible_facts['distribution'] == "CentOS"
      - ansible_facts['distribution_major_version'] == "6"
# 使用 Tests 条件测试中结果判断,是否 import 或 include 这个 tasks
- import_tasks: other_tasks.yml # note "import"
  when: x is not defined
```

Ansible 提供了一些常用的条件判断 facts,如下

- `ansible_facts['distribution']`: 系统发行版,可能的值为

```text
Alpine, Altlinux, Amazon, Archlinux, ClearLinux, Coreos, CentOS, Debian, Fedora, Gentoo,
Mandriva, NA, OpenWrt, OracleLinux, RedHat, Slackware, SMGL, SUSE, Ubuntu, VMwareESX
```

- `ansible_facts['distribution_major_version']`: 系统发行版主版本号
- `ansible_facts['os_family']`: 系统发行版类型,可能值为

```text
AIX, Alpine, Altlinux, Archlinux, Darwin, Debian, FreeBSD, Gentoo,
HP-UX, Mandrake, RedHat, SGML, Slackware, Solaris, Suse, Windows
```

### 循环与迭代 Loops

对于需要重复执行的任务,Ansible 提供了两个用于创建循环的关键字 `loop` 和 `with_<lookup>`.

> `loop` 关键字是在 2.5 之后的版本中引入的,而且在以后的版本会优化相关语法.并作为官方推荐的迭代方式;`with_<lookup>` 后续版本也会继续支持.

#### `loop` vs `with_*`

- `with_<lookup>` 关键字依赖于 [Lookup Plugins](https://docs.ansible.com/ansible/latest/plugins/lookup.html#lookup-plugins)
- `loop` 关键字等价于 `with_list`,是简单循环最好的选择
- `loop` 关键字不支持字符串作为输入,参见 [loop 中 query 与 lookup 函数比较](#loop-中-query-与-lookup-函数比较)或官方文档 - [Ensuring list input for loop: query vs. lookup](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#query-vs-lookup) 或.
- 一般来说,[后文](#with_X-转换为-loop)(官方文档 - [Migrating from with_X to loop](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html#migrating-from-with-x-to-loop))中包含的 `with_*` 均可以转换为 `loop`.
- 对于 `with_items` 带有列表嵌套的场景,需要对 `loop` 内容使用 `flatten(1)` 过滤器来得到相同的结果.如 `with_items: [1, [2, 3], 4]` 可转换为 `loop: "{{ [1, [2,3] ,4] | flatten(1) }}"`

#### loop 中 query 与 lookup 函数比较

- `lookup` 函数默认返回逗号分割的字符串.可以使用 `wantlist=True` 参数强制该函数返回列表.
- `query` 函数总是返回列表,当使用 `loop` 关键字时,可以提供一个更简单的接口.

```yaml
loop: "{{ query('inventory_hostnames', 'all') }}"
loop: "{{ lookup('inventory_hostnames', 'all', wantlist=True) }}"
```

#### with_X 转换为 loop

##### with_list

`with_list` 可直接转换为 `loop`

```yaml
- name: with_list
  debug:
    msg: "{{ item }}"
  with_list:
  # loop:
    - one
    - two
```

#### with_item

`with_items` 可以转换为 `loop` 和 `flatten` 过滤器

```yaml
- name: with_items
  debug:
    msg: "{{ item }}"
  with_items: "{{ items }}"
  # loop: "{{ items|flatten(levels=1) }}"
```

#### with_indexed_items

`with_indexed_items` 可以转换为 `loop`, `flatten` 过滤器和 `loop_control.index_var`.

```yaml
- name: with_indexed_items
  debug:
    msg: "{{ item.0 }} - {{ item.1 }}"
  with_indexed_items: "{{ items }}"

- name: with_indexed_items -> loop
  debug:
    msg: "{{ index }} - {{ item }}"
  loop: "{{ items|flatten(levels=1) }}"
  loop_control:
    index_var: index
```

#### with_flattened

`with_flattened` 可以转换为 `loop` 和 `flatten` 过滤器

```yaml
- name: with_flattened
  debug:
    msg: "{{ item }}"
  with_flattened: "{{ items }}"
  # loop: "{{ items|flatten }}"
```

#### with_fileglob

`with_fileglob` 可以转换为 `loop` 和 `lookup` 或 `query` 函数

```yaml
- name: with_flattened
  debug:
    msg: "{{ item }}"
  with_fileglob: '*.txt'
  # loop: "{{ query('fileglob', '*.txt') }}"
  # loop: "{{ lookup('fileglob', '*.txt', wantlist=True) }}"
```

### Blocks

Blocks 允许对 Task 进行逻辑分组并进行错误处理.除循环外,单个任务的大多数内容均可用于 Block.

示例如下:

```yaml
tasks:
  - name: Install, configure, and start Apache
    block:
      - name: install httpd and memcached
        yum:
          name:
          - httpd
          - memcached
          state: present

      - name: apply the foo config template
        template:
          src: templates/src.j2
          dest: /etc/foo.conf
      - name: start service bar and enable it
        service:
          name: bar
          state: started
          enabled: True
    when: ansible_facts['distribution'] == 'CentOS'
    become: true
    become_user: root
    ignore_errors: yes
```

除了单个任务的大多数常用内容外,Block 还支持如下关键字:

- `rescue`: 用于错误处理,相当于 `try...catch` 语句
- `always`: 此关键字定义的内容总是执行,相当于 `finally` 语句

```yaml
- name: Attempt and graceful roll back demo
  block:
    - debug:
        msg: 'I execute normally'
    - name: i force a failure
      command: /bin/false
    - debug:
        msg: 'I never execute, due to the above task failing, :-('
  rescue:
    - debug:
        msg: 'I caught an error'
    - name: i force a failure in middle of recovery! >:-)
      command: /bin/false
    - debug:
        msg: 'I also never execute :-('
  always:
    - debug:
        msg: "This always executes"
```

### 高级功能

#### Tags 标签

ansible playbook 应用于有一个大型的 playbook,但仅仅运行指定的 tasks 的场景.在执行 playbook 时,可以使用 `--tags` 或 `--skip-tags` 运行或不运行带有指定标签的 tasks

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

- `always` 标签默认每次都会执行,除非使用 `--skip-tags always` 特别指定跳过该 tag 标记的 task
- `never`标签默认每次都不会执行,除非使用 `--tags never` 特别指定运行该 tag 标记的 task

#### 其它常用关键字

- `debugger`: 显示 debug 信息,可选项为 `always | never | on_failed | on_unreachable | on_skipped`.

```yaml
- name: Execute a command
  command: false
  debugger: on_failed
```

- `serial`: 一次性多少个主机并行执行任务,多用于滚动更新.支持数字,百分比以及它们的列表.若指定为列表,则每次执行的主机数等于列表中元素.

```yaml
- name: test play
  hosts: webservers
  serial: 2
  gather_facts: False

  tasks:
    - name: task one
      command: hostname
    - name: task two
      command: hostname
```

假设有 4 台主机,执行过程将如下所示

```yaml
PLAY [webservers] ****************************************

TASK [task one] ******************************************
changed: [web2]
changed: [web1]

TASK [task two] ******************************************
changed: [web1]
changed: [web2]

PLAY [webservers] ****************************************

TASK [task one] ******************************************
changed: [web3]
changed: [web4]

TASK [task two] ******************************************
changed: [web3]
changed: [web4]

PLAY RECAP ***********************************************
web1      : ok=2    changed=2    unreachable=0    failed=0
web2      : ok=2    changed=2    unreachable=0    failed=0
web3      : ok=2    changed=2    unreachable=0    failed=0
web4      : ok=2    changed=2    unreachable=0    failed=0
```

- `run_once`: 尝试仅在主机列表的第一台主机上运行一次,然后将运行结果和 facts 应用到其他主机上.
- `ignore_errors`: 尝试忽略执行过程中的错误
- `failed_when`: 定义 playbook 执行失败的条件.当该关键字指定的条件判断句为 True 时,playbook 执行失败并退出执行.等价于 `fail` 模块与 `when` 条件判断使用.
- `any_errors_fatal`: 只要有错误发生,则终止整个 Playbook 的执行
- `vars_prompt`, `prompt`: 由 `vars_prompt` 包含的多个 `prompt` 列表用于交互式定义变量.
  - `private`: 是否定义为私有变量.
  - `default`: 为交互式变量设置默认值
  - `encrypt`: 设置加密方式.可选值为参见[官方文档](https://docs.ansible.com/ansible/latest/user_guide/playbooks_prompts.html#prompts)
  - `confirm`: 是否需要确认,再次输入
  - `salt_size`: 加密过程中加盐大小
  - `unsafe`: 输入变量中可能包含导致创建模版失败的特殊字符(如 `%`).使用该参数进行转义

```yaml
- hosts: all
  vars_prompt:
    - name: username
      prompt: "What is your username?"
      default: "mike"
      private: no
    - name: password
      prompt: "What is your password?"
      unsafe: yes
      private: yes
      encrypt: "sha512_crypt"
      confirm: yes
      salt_size: 7

  tasks:
    - debug:
        msg: 'Logging in as {{ username }}'
```

- `connection`: 指定连接的主机.多用于指定为 `local`.

更多关键字,详见官方文档 - [Playbook Keywords](https://docs.ansible.com/ansible/latest/reference_appendices/playbooks_keywords.html).
