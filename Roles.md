Roles
=====

### 基本概述
Roles基于一个已知的文件结构，自动加载vars_files，tasks和handlers。roles比playbook的结构更加清晰有层次，在编写roles时最好能够将一个task拆分为一个文件，方便后续复用。

### Roles目录结构
roles官方目录结构，必须按如下定义。在每个目录中必须有main.yml文件。
```shell
/etc/ansible/roles$ tree
.
└── nfs              #角色名称 - 建议角色名称与要操作的服务名称一致
    ├── files 		 #存放文件 - 存放普通文件，使用copy模块调用
    ├── handlers	 #触发任务
    ├── meta 		 #依赖关系 - 允许不同的role之间的相互调用关系
    ├── tasks		 #具体任务
    ├── templates	 #模板文件 - 存放jinja2模板文件，使用template模块调用
    └── vars 		 #定义变量
```

### Roles依赖关系
roles允许在使用role的过程中引入其他的role。role依赖关系存储在role目录中的meta/main.yml文件中<br />
示例：安装wordpress依赖nginx和php
```shell
dependencies:
  - { role: nginx }
  - { role: php-fpm }
```
此例中wordpress的role会先执行nginx的role、然后执行php-fpm的role，最后执行wordpress本身的role

### Roles案例
Roles小技巧：
1. 创建roles目录结构，手动或使用命令ansible-galaxy init test创建test的role目录
2. 编写roles的功能，也就是tasks
3. 最后playbook引用roles编写好的tasks

示例：以安装memcached服务为例
```shell
$ tree    #查看role整体结构
.
├── memcached
│   ├── files
│   ├── handlers
│   │   └── main.yml
│   ├── tasks
│   │   └── main.yml
│   ├── templates
│   │   └── memcached.j2
│   └── vars
└── site.yml

$ cat memcached/tasks/main.yml    #查看tasks内容，此文件可使用include简化
- name: Install Memcached Server
  yum: 
    name: memcached
    state: present

- name: Configure Memcached Server
  template: 
    src: memcached.j2
    dest: /etc/sysconfig/memcached
  notify: Restart Memcached Server

- name: Started Memcached Server
  service: 
    name: memcached
    state: started
    enabled: yes

$ cat memcached/handlers/main.yml    #查看handlers触发内容
- name: Restart Memcached Server
  service: 
    name: memcached
    state: restarted

$ cat site.yml    #此文件对role来说相当于一个入口文件，整个memcached的role最后都是通过此playbook进行调用执行的
- hosts: gui
  roles: 
    - role: memcached
```

示例：简化后的tasks/main.yml文件
```shell
$ tree 
.
├── memcached
│   ├── files
│   ├── handlers
│   │   └── main.yml
│   ├── tasks
│   │   ├── config.yml
│   │   ├── install.yml
│   │   ├── main.yml
│   │   └── start.yml
│   ├── templates
│   │   └── memcached.j2
│   └── vars
└── site.yml

$ cat memcached/tasks/main.yml 
- include_tasks: install.yml
- include_tasks: config.yml
- include_tasks: start.yml
```

示例：playbook文件的写法。此示例与上面示例无关，仅作为playbook的写法扩展
```shell
- hosts: gui
  roles:
    - role: foo
      vars:
        message: "first"

    - { role: foo, vars: { message: "second" } }
      tags: 
      	- nginx
```
注：role的playbook语法里支持tags，可以通过role单独执行某个role<br />
示例：Nginx + PHP
```shell
$ tree nginx
nginx
├── handlers
│   └── main.yml
├── tasks
│   ├── config.yml
│   ├── install.yml
│   ├── main.yml
│   └── start.yml
└── templates
    └── nginx.conf.j2

$ tree php-fpm/
php-fpm/
├── handlers
│   └── main.yml
├── tasks
│   ├── config.yml
│   ├── install.yml
│   ├── main.yml
│   └── start.yml
├── templates
│   ├── php.ini.j2
│   └── www.conf.j2
└── vars
    └── main.yml

$ cat site.yml 
- hosts: gui
  roles: 
    - role: memcached
    - role: nginx
    - role: php-fpm

$ cat php-fpm/vars/main.yml 
packages: 
  - php
  - php-cli
  - php-fpm
  - php-pdo
  - php-gd
  - php-mbstring
```
仅展示关于vars的内容，其他内容前面都有

Ansible Galaxy
--------------
一个免费网站，存放共享的roles，从Galaxy下载roles角色是快速启动自动化项目的方式之一。Ansible提供了ansible-galaxy工具，可用init（初始化）、search（查找）、install（安装）、remove（移除）等操作
[Galaxy官网](galaxy.ansible.com)，从Galaxy下载的role默认会下载到~/.ansible/roles/目录下。