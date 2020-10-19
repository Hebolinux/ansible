大纲
====
	1.什么是ansible
	2.ansible特性/优点
	3.ansible基础架构
	4.ansible安装与配置
	5.ansible inventory
	6.ansible ad-hoc
	7.ansible playbook
	8.变量 variables （优先级）
	9.判断和循环语句
	10.异常处理
	11.tag 标签
	12.handlers 触发器
	13.include 包含
	14.ansible jinja 模板
	15.ansible role 角色
	16.ansible galaxy
	17.ansible tower

### 一、什么是ansible
IT自动化的配置管理工具，主要体现在ansible集成了丰富的模块和功能组件。减少重复性工作和维护成本。

### 二、ansible特性/优点
	1.特性
		a. 批量执行远程命令
		b. 批量配置软件服务
		c. 实现软件开发功能
		d. 编排高级的IT任务
	2.优点
		a. 无代理模式
		b. 操作灵活，简单易用
		c. 安全，使用ssh协议进行通讯
		d. 可移植性高

### 三、ansible基础架构
	主机清单：inventory（对不同的主机进行逻辑上的分组）
				host：单个主机
				group：多个主机
	AD-Hoc：单条命令，通过不同的python module实现不同的操作
	Playbook：YAML语言编写，将多个AD-Hoc命令集中到一个文件中执行
	tcp/ssh：控制端与被控端之间的命令推送需要保证传输安全可靠

### 四、ansible的安装与配置
主要的重点在于配置文件的所在路径
#### 安装
```shell
Centos && Redhat
# yum install ansible -y
	#需要注意的是，Centos或RHEL安装ansible时需要EPEL源，可参考下面的链接

Ubuntu
$ sudo apt-get install -y ansible
```
[7版本的EPEL源](https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm)
[8版本的EPEL源](https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm)
#### 配置
```shell
$ ansible --version
ansible 2.9.6
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/hebo/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.8.2 (default, Jul 16 2020, 14:00:26) [GCC 9.3.0]
```
配置文件|作用
:-:|:-:
config file|ansible的主配置文件的路径
configured module|ansible模块查找路径
ansible python|ansible模块在python中的路径
executable location|ansible命令执行路径
host_key_checking|检查ssh认证，默认情况被注释
timeout|超时时间，主机较多时可适当延长超时时间
```diff
- 需要注意的是，config file的位置是有优先级的，配置文件的路径的不同导致优先级的不同，优先级高的路径优先被读取。如果有多个config file，则优先级高的config file生效。优先级从高到低如下：
	ANSIBLE_CONFIG（变量指定配置文件）
	ansible.cfg（当前目录下）
	.ansible.cfg（当前用户的家目录下）
	/etc/ansible/ansible.cfg
```
实际上，ansible.cfg中没有多少需要更改的配置，下面是一些常用配置选项的作用
选项|作用
:-:|:-:
inventory|指令hosts文件默认所在位置，可认为修改
forks|并发数，ansible默认一次推送命令到5台主机
remote_tmp|客户端执行ansible推送的命令时，需要通过python解析，解析时需要一个缓存目录，此选项指定缓存目录的位置
remote_port|远程端口

### 五、Ansible Inventory
Inventory文件中写入了被管理主机与主机组信息。默认文件路径位于/etc/ansible/hosts。可通过 -i 选项指定Inventory文件位置。连接被管理主机的方式有多种，但此处仅强调基于密钥连接

#### 基于密码
```shell
$ cat hosts

#主机+端口+密码
[webservers]
10.250.1.51 ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass='redhat'
10.250.1.52 ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass='redhat'

#域名+密码
[webservers]
web[1:2].example.com ansible_ssh_pass='redhat'

#域名+密码，此处为webservers设置vars变量
[webservers]
web[1:2].example.com
[webservers:vars]
ansible_ssh_pass='redhat'
```

#### 基于密钥
```shell
# ssh-keygen	#密钥操作自行百度
# ssh-copy-id -i ~/.ssh/id_rsa.pub 10.250.1.51
# ssh-copy-id -i ~/.ssh/id_rsa.pub 10.250.1.52
# cat hosts
[webservers]
10.250.1.51
10.250.1.52

#别名+地址+端口
[webservers]
web1 ansible_ssh_host=10.250.1.51 ansible_ssh_port=22
web2 ansible_ssh_host=10.250.1.52
```
	#注：若当前下发公钥的用户不是root，则需要在目标主机地址前面指定root账户

#### 主机组
```shell
# cat hosts
[lbservers]
10.250.1.51
10.250.1.52

[webservers]
10.250.1.53
10.250.1.54

[servers:children]	#定义servers组，包含lbservers和webservers
lbservers
webservers
```

### 六、AD-Hoc
AD-Hoc类似单个的shell命令，执行完就结束。AD-Hoc命令格式如下：
命令格式|ansible|webservers|-m|command|-a|'df-h'
:-:|:-:|:-:|:-:|:-:|:-:|:-:
格式说明|命令|主机名称|指定模块|模块名称|模块动作|具体命令

使用AD-Hoc执行一次远程命令，注意观察返回结果的颜色
- 绿色：被管理端主机没有修改
- 黄色：被管理端主机发现变更
- 红色：故障
```diff
- 任何ansible模块的用法都可以通过ansible-doc命令查看示例语法
```
ansible的模块有很多，下面仅提及较常用的模块
功能|模块
:-:|:-:
命令|command(默认)  shell  scripts
安装|yum  yum_repository
配置|copy  file  grt_url lineinfile
启动|service  systemd
用户|user  group
任务|cron
挂载|mount
防火墙|firewall  selinux

#### ping
```shell
# ansible all -m ping	#测试主机联通性
```

#### command & shell
对比：
	command模块不支持管道符号
	默认不指定-m模块时，使用command模块
```shell
# ansible web -m command -a 'ps -aux | grep nginx'

# ansible web -m shell -a 'ps -aux | grep nginx' 
```

#### yum
```shell
# ansible-doc yum	#查看EXAMPLES示例
# ansible web -m yum -a 'name=httpd state=latest'	#安装最新的httpd服务，如果已安装就更新软件包

# ansible web -m yum -a 'name=httpd state=latest enablerepo=epel'	#指定yum源安装软件包

# ansible web -m yum -a 'name=https://mirrors.aliyun.com/zabbix/zabbix/4.2/rhel/7/x86_64/zabbix-agent-4.2.3-2.el7.x86_64.rpm state=latest'	#通过URL安装软件包

# ansible web -m yum -a 'name="*" state=latest exclude=kernel*,foo*'	#更新所有软件包，排除与kernel和foo有关的

# ansible web -m yum -a 'name=httpd state=absent'	#卸载httpd软件包
```

#### copy
```shell
# ansible gui -m copy -a 'src=/root/a/httpd.conf dest=/etc/httpd/conf/httpd.conf owner=root group=root mode=644'	#将本地修改后的配置文件推送到远端

# ansible gui -m copy -a 'src=/root/a/httpd.conf dest=/etc/httpd/conf/httpd.conf owner=root group=root mode=644 backup=yes'	#backup只会在文件有改动后才会备份

# ansible gui -m copy -a "content=HttpServer dest=/var/www/html/index.html"	#直接将content的内容复制到远端文件中
```

#### get_url
使用get_url时是将链接文件下载到主机组内的所有主机，除非手动指定单个主机
```shell
 # ansible gui -m get_url -a 'url=https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm dest=/etc/yum.repos.d/'	#下载文件到指定目录

# md5sum epel-release-latest-7.noarch.rpm	#使用ansible验证md5之前要先计算出文件的md5值
# ansible gui -m get_url -a 'url=https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm dest=/etc/yum.repos.d/ checksum=312e447861ec8dc90f2eef5d85778059'	#下载链接时验证md5值
```

#### file
```shell
# ansible gui -m file -a 'path=/var/www/html/index.html state=touch owner=apache group=apache mode=644'	#创建文件

# ansible gui -m file -a 'path=/var/www/html/dd state=directory owner=apache group=apache mode=755'	#创建目录

# ansible gui -m file -a 'path=/var/www/html/dd owner=root group=root mode=755'	#更改目录权限

# ansible gui -m file -a 'path=/var/www/html/dd owner=root group=root recurse=yes'	#更改目录及目录下的递归授权
```

#### lineinfile
此模块仅作为扩展模块，常用在修改文件内容。有兴趣百度一下这个模块的选项的作用。
```shell
# ansible gui -m lineinfile -a "dest=/etc/exports line='hello world'"	#向/etc/exports文件的最后一行插入文本

# ansible-doc lineinfile
```

#### service
```shell
# ansible gui -m service -a 'name=httpd state=started enabled=yes'	#启动httpd服务并设置为开机自启
```
service模块的选项比较单一，用表格的方式列出
选项|作用
:-:|:-:
started|启动
restarted|重启
stopped|停止
reloaded|重载

#### group
```shell
# ansible gui -m group -a 'name=news gid=9999 state=present'	#创建一个新组并设置gid

# ansible gui -m group -a 'name=http gid=8888 system=true'	#创建系统组

# ansible gui -m group -a 'name=news state=absent'	#删除组
```

#### user
```shell
# ansible gui -m user -a 'name=joh uid=1040 group=adm'	#创建用户并指定uid,gid

# ansible gui -m user -a 'name=joh shell=/sbin/nologin groups=bin,sys'	#禁止账户登录并追加两个附属组

# ansible localhost -m debug -a "msg={{ '123' | password_hash('sha512','salt') }}"	#生成加密后的hash字符串

# ansible gui -m user -a 'name=jsm password=$6$salt$jkHSO0tOjmLW0S1NFlw5veSIDRAVsiQQMTrkOKy4xdCCLPNIsHhZkIRlzfzIvKyXeGdOfCBoW1wJZPLyQ9Qx/1 create_home=yes'	#创建用户设置密码，create_home默认是开启的

# ansible gui -m user -a 'name=jsm state=absent remove=yes'	#remove表示彻底删除，等同于userdel -r

# ansible gui -m user -a 'name=http generate_ssh_key=yes ssh_key_bits=2048 ssh_key_file=.ssh/id_rsa'	#创建ssh秘钥对，指定秘钥强度2048，指定秘钥存放路径
```
指定单个组和多个组时需要注意group与groups的使用

#### cron
```shell
# ansible gui -m cron -a 'name=job1 job="ls > /dev/null"'	#每分钟执行一次job，name表示这个定时任务的注释

# ansible gui -m cron -a 'name=job2 minute=0 hour=5,2 job="ls > /dev/null"'	#凌晨5点和2点执行job

# ansible gui -m cron -a 'name=job2 minute=0 hour=5,2 job="ls > /dev/null" disabled=yes'	#关闭定时任务
```
关闭定时任务的命令需要与配置定时任务时的命令一样，仅在后续添加一个disabled选项

#### mount
mount需要注意几个state参数
选项|作用
:-:|:-:
mounted|永久挂载，立即生效
present|永久挂载，写入fstab文件中，但不会立即生效
unmounted|临时卸载，不会删除fstab文件中的条目
absent|永久卸载，立即生效
```shell
# ansible gui -m mount -a 'src=/dev/sr0 path=/mnt/cdrom fstype=iso9660 opts=defaults state=mounted'	#永久挂载光盘文件

# ansible gui -m mount -a 'src=/dev/sr0 path=/mnt/cdrom fstype=iso9660 opts=defaults state=unmounted'	#临时卸载
```

#### firewalld && selinux
```shell
# ansible gui -m selinux -a 'state=disabled'	#关闭selinux

# ansible gui -m firewalld -a 'service=http permanent=yes state=enabled'	#永久放行http服务，但不会立即生效

# ansible gui -m firewalld -a 'zone=public port=8080/tcp permanent=yes state=enabled'	#放行8080端口，zone默认为public

# ansible gui -m firewalld -a 'zone=public port=8080-8090/tcp permanent=yes immediate=yes state=enabled'	#永久放行8080~8090端口，immediate选项表示立即生效
```
