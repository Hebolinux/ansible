Jinja2
======

基本介绍
-------
Jinja2是Python的全功能模板引擎，Ansible通常会使用Jinja2模板来修改被控端配置文件。例如10台主机都安装httpd服务，但是要求每个服务器端口不一。Ansible允许jinja2模板中使用条件判断和循环，但jinjia判断与循环语法不允许在playbook中使用。

Ansible Jinja2基本使用
---------------------
1. jinja模板基本语法
	1. 要想在配置文件中使用jinja2,playbook中的tasks必须使用template模块
	2. 模板配置文件里面使用变量，比如{{ PORT }}或使用{{ facts变量 }}
2. jinja模板逻辑关系
	{% for i in EXPR %}...{% endfor %}作为循环表达式
	{% if EXPR %}...{% elif EXPR %}...{% endif %}作为条件判断
	{# COMMENT #}表示注释

示例：jinja基本语法
```shell
- hosts: gui
  tasks:
    - name: Copy Template File /etc/motd
      template: 
        src: ./motd.j2
        dest: /etc/motd

$ cat motd.j2 
Welcome to SZZT Compute Service

This System Hostname: {{ ansible_hostname }}
This System Total Memory is: {{ ansible_memtotal_mb }} MB
This System Free Memory is: {{ ansible_memfree_mb }} MB
```

示例：jinja判断语句
```shell
{% if ansible_fqdn == "Centos7" %}
echo "123"    #这里不会作为命令执行，而是作为文件内容保存到被控端
{% elif ansible_fqdn == "Ubuntu" %}
echo "456"
{% else %}
echo "789"
{% endif %}
```

示例：jinja循环语句
```shell
{% for i in range(1,10) %}
	server 172.16.1.{{ i }};
{% endfor %}
```

示例：jinja渲染nginx_proxy配置文件
```shell
upstream {{ server_name }} {    #server_name是在playbook中定义的主机名变量
{% for i in range(1,10) %}
	server 10.250.1.{{ i }}:{{ http_port }} weight=2;    #IP地址循环，指定端口
{% endfor %}
}

server {
	listen {{ http_port }};
	server_name {{ server_name }};
	localtion / {
		proxy_pass http://{{ server_name }};    #nginx负载均衡文件中的资源池upstream设置的与server_name同名
		include proxy_params;
	}
}
```

ansible使用jinja2的判断表达式渲染出keepalived的Master和Slave的配置文件，并推送至被控端。可以通过三种方式实现
```diff
+ Inventory中的host_vars根据不同主机设置不同的变量
+ Playbook使用when判断主机名称，然后分发不同的配置文件
+ 使用jinja2的方式渲染出不同的配置文件
```
示例：根据不同的主机设置不同的变量配置keepalived
```shell
global_defs{
	router_id {{ ansible_fqdn }}
}

vrrp_instance VI_1{
	state {{ state }}    #state和priority变量可以在host_vars中根据不同的IP设置不同的值
	priority {{ priority }}
	interface eth0
	virtual_router_id 50
	advert_int 1
	authentication {
		auth_type PASS
		auth_pass 1111
	}
	virtual_ipaddress {	
		10.250.1.100
	}
}
```
示例：when判断主机名后发布不同配置的keepalived
```shell
- hosts: gui
  tasks: 
    - name: Copy Keepalived
      template: 
        src: ./keepalived-master.conf.j2
        dest: /tmp/keepalived.conf
      when: ( ansible_hostname == "centos7" )

# cat keepalived-master.conf.j2 
global_defs{
	router_id {{ ansible_fqdn }}
}

vrrp_instance VI_1{
	state MASTER
	priority 150
	...
```
示例：jinja渲染keepalived配置
```shell
global_defs{
        router_id {{ ansible_fqdn }}
}

vrrp_instance VI_1{
{% if ansible_hostname == "Centos7" %}
        state MASTER
        priority 150
{% else %}
        state BACKUP
        priority 100
{% endif %}
#--------------相同点----------------
        interface eth0
        virtual_router_id 50
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass 1111
        }
        virtual_ipaddress {
                10.250.1.100
        }
}
```
此例使用jinja2的判断语句对不同主机渲染不同配置