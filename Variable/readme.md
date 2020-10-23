变量 
====
变量在整个playbook的项目中都是为了简化项目的维护，提高项目的可读性

### ansible如何定义和使用变量
1. 通过playbook文件中的play进行定义
2. 通过inventory主机清单进行变量定义
3. 通过执行playbook时使用-e参数指定变量

示例：使用vars关键字在play部分定义变量
```shell
- hosts: gui
  vars:
    - web_packages: httpd
    - ftp_packages: vsftpd

  tasks:
    - name: Installed RPM Packages
      yum:
        name: 
          - "{{ web_packages }}"
          - "{{ ftp_packages }}"
        state: present
```
```diff
- 注：在play中定义的变量仅在此playbook中生效
```

示例：通过定义一个变量文件vars_files，然后使用playbook进行调用
```shell
- hosts: gui
  vars_files: 
    - ./vars_public.yml    #对比上一个示例，将此处更改为调用存放变量的文件

  tasks:
    - name: Installed RPM Packages
      yum: 
        name: 
          - "{{ web_packages }}"
          - "{{ ftp_packages }}"
        state: present
```
```diff
- 调用存放变量的文件时，绝对路径或相对路径都可以使用，但建议使用相对路径
- 需要调用多个变量文件时可使用yaml语法中的列表格式，单个文件也可以写成vars_files: ./vars_public.yml的形式
```

---

示例：inventory定义变量
```shell
[szzt]
gui ansible_ssh_host=10.250.1.10
[szzt:vars]    #在组名后面指定变量
file_name=group_vars
```
在inventory主机清单中定义变量后在playbook中可以直接调用 <br />

示例：官方推荐在playbook项目目录下建立host_var和group_vars两个目录，分别在两个目录中存放与主机/主机组相关的变量
```shell
$ mkdir project && cd project    #创建一个项目目录
$ mkdir group_vars    #创建组变量目录
$ mkdir host_vars    #创建主机变量目录

$ cat group_vars/szzt    #在组目录下创建变量文件，此文件命名必须与inventory中的组名一致
web_packages: wget
ftp_packages: tree

$ cat test.yml 
- hosts: szzt
  tasks: 
    - name: Install Rpm Packages
      yum: 
        name: 
          - "{{ web_packages }}"
          - "{{ ftp_packages }}"
        state: present
```
```diff
- 系统在group_vars目录下提供了一个特殊文件，all，playbook中所有的组都可以使用此文件中的变量
- 主机变量与组变量的编写是一样的，主机变量文件命名也必须与inventory清单中的主机名一致
- 注：playbook首先会去找host_vars下的变量文件，如果host_vars下没有，则playbook会继续找本IP所归属的group_vars下的变量，如果也没有，则playbook会找到all文件，如果all文件也没有，则报错

- 在playbook中引用变量时要特别注意双引号
```

---

示例：通过执行playbook的时候，在命令行使用-e参数指定变量
```shell
- hosts: "{{ hosts }}"  #将play设定为一个变量，变量名可更改
  tasks: 
    - name: Install Rpm Packages
      yum: 
        name: 
          - "{{ web_packages }}"
          - "{{ ftp_packages }}"
        state: present

$ sudo ansible-playbook test.yml -e "hosts=gui"
```
### ansible变量优先级
根据上面的5中变量调用方式，其优先级从高到低为：
1. 命令行指定变量
2. playbook中的vars_file
3. playbook中的vars
4. playbook项目目录下的host_vars下的变量
5. playbook项目目录下的group_vars下的变量

### ansible的变量注册register
通常情况下，ansible的输出结果是定义的tasks是否成功，而要查看被控端主机的状态时（比如某个进程是否正常运行），则需要用到register关键词
。register会将单个tasks内执行的命令的结果保存到变量中，然后可以使用debug模块输出变量中的值。
```shell
- hosts: gui
  tasks: 
    - name: Check Httpd Server
      shell: ps aux | grep httpd
      register: check_httpd    #将shell模块执行的ps命令的结果保存到变量check_httpd中

    - name: Output Variables
      debug: 
        msg: "{{ check_httpd }}"    #使用debug模块的msg参数输出变量check_httpd所保存的所有内容
```
如果按照上例写法，则msg除了会输出ps命令执行的结果，还会输出ansible执行命令的状态。如果要简洁一些的内容，则可以在msg后面使用python中调用函数的语法形式
```shell
    - name: Output Variables
      debug: 
        msg: "{{ check_httpd.stdout_lines }}"    #设置仅输出stdout_lines中的内容
```

### ansible facts变量
facts用于采集被控端的状态指标，比如IP地址、主机名称、cpu信息等等，每次playbook执行前都会先采集被控端信息
```diff
- 默认情况的facts变量名都已经预先定义好了，只需要采集被控端的信息，然后传递至facts变量即可
```
```shell
$ sudo ansible gui -m setup    #查看被控端的变量信息
```
示例：获取facts变量中的主机名及其对应IP地址
```shell
- hosts: gui
  tasks: 
    - name: Output variables ansible facts
      debug: 
        msg: IP address "{{ ansible_fqdn }}" is "{{ ansible_eth0.ipv4.address }}"
```
此示例中的两个变量都是通过facts采集到的被控端上的信息。开启facts变量对于批量管理多台主机时比较便捷，但facts的采集时间相对来说也比较长。在确定不会用到facts中的变量时可以选择性关闭facts
```shell
- hosts: gui
  gather_facts: no    #在play中添加此选项即可关闭facts采集
  tasks:
    - name: Output variables ansible facts
      debug:
        msg: IP address "{{ ansible_fqdn }}" is "{{ ansible_eth0.ipv4.address }}"
```

### playbook示例
命名|作用
:-:|:-:
zabbix-agent-example.yml|使用facts变量根据被控端的主机名更改zabbix-agent的配置文件
memcached-example.yml|使用facts变量根据不同的内存生成不同的Memcached配置文件
hostname-example.yml|使用register变量更改远端主机名
nginx_php.yml|LNMP+可道云示例

[可道云软件下载地址](https://kodcloud.com/download/)
或者使用此命令下载
```shell
wget http://static.kodcloud.com/update/download/kodbox.1.13.zip
```