Playbook语句 
===========
此章包括判断语句when、循环语句with_items、触发器handlers、标签tags和包含语句include，这些语句可以使playbook的功能更加完善和强大。
### 判断语句when
在ansible的被控端下的主机操作系统如果不一样，那么判断语句将会非常有用。比如Centos和Ubuntu的Apache软件包名称就不一样，想要为两个系统都安装上http服务就需要先判断其操作系统。
```shell
- hosts: szzt
  tasks:
    - name: Instal Httpd Server
      yum:
        name: httpd
        state: present
      when: ( ansible_distribution == "CentOS")

    - name: Install Apache2 Server
      yum:
        name: apache2
        state: present
      when: ( ansible_distribution == "Ubuntu")
```
```diff
补充：find ./ -type f -maxdepth 1    #查看1级目录下的所有文件
```
在when语句中有一种类似shell通配符的语法，它能够仅对符合匹配的主机进行操作，其他主机全部跳过。这需要用到固定语法‘is match("")’，注意括号中的引号必须存在，都则ansible执行会报错
```shell
- hosts: szzt
  tasks:
    - name: Create YUM Repository
      yum_repository:
        name: ansible
        description: ansible_test
        baseurl: http://mirrors.aliyun.com/repo/Centos-7.repo
        enabled: yes
        gpgcheck: no
      when: ( ansible_fqdn is match("min")) or
      		( ansible_fqdn is match("tencent"))
```
此例中的when判断语句表示只有主机名以min开头的主机才执行yum_repository操作，相对的，既然有‘is match("")’语句，那也有其反义语句‘is not match("")’，上述语法之外，when语句中还使用了逻辑判断语句or，与之相反的也可以使用and，and语句通常用于同时匹配主机名和IP地址的情况
```shell
#通过命令执行的结果进行判断
- hosts: all
  tasks:
    - name: Check Httpd Server
      command:
        cmd: systemctl is-active httpd
      ignore_errors: yes
      register: check_httpd

#   - name: Debug
#     debug: 
#       msg: "{{ check_httpd }}"

   - name: Httpd Restart
     service:
       name: httpd
       state: restarted
     when: check_httpd.rc == 0
```
此例中when判断首先会确认上一条命令的执行状态码，状态码正常时才会执行下一命令

### 循环语句with_items
循环语句可以减少重复性的工作，在playbook中常见的重启多个服务，使用循环语法实现时非常简洁<br />
示例：重启多个服务
```shell
- hosts: gui
  tasks: 
    - name: Service Nginx Server
      service: 
        name: "{{ item }}"    #使用item是固定语法
        state: restarted
      with_items: 
        - nginx
        - php-fpm
```
示例：安装多个软件包
```shell
- hosts: gui
  tasks: 
    - name: Install Packages
      yum: 
        name: "{{ pack }}"
        state: present
      vars: 
        pack: 
          - httpd
          - mariadb-server
```
示例：使用变量字典循环批量创建用户
```shell
- hosts: gui
  tasks: 
    - name: Add Users
      user: 
        name: "{{ item.name }}"
        groups: "{{ item.groups }}"
        state: present
      with_items: 
        - { name: 'test1', groups: 'bin' }    #列表内以字典的方式存储元素
        - { name: 'test2', groups: 'root' }   #创建用户时组必须是已经存在的
```
调用变量时直接写item时表示取items中的一整行，以python的语法格式仅调用列表中的某个元素<br />
示例：使用变量字典循环批量拷贝文件
```shell
- hosts: gui
  tasks: 
    - name: Copy Configure
      copy: 
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
      with_items: 
        - { src: '/home/hebo/git/ansible/project/rsyncd.conf.j2', dest: '/root/rsyncd.conf'}
        - { src: '/home/hebo/git/ansible/project/rsync.pass.j2', dest: '/root/rsync.pass'}
```

### handlers触发器
hadlers执行流程：notify 监控 --> 通知 --> handlers触发
```shell
- hosts: gui
  tasks: 
    - name: Configure Kod
      copy:
        src: /home/hebo/git/ansible/project/kod.conf.j2    #此文件从被控端拷贝后修改
        dest: /etc/nginx/conf.d/kod.conf
      notify: Service Nginx Restart    #监控上面配置文件是否有修改

  handlers: 
    - name: Service Nginx Restart    #必须与notify的name一致
      service: 
        name: nginx
        state: restarted
```
注意事项：
1. 无论多少个tasks通知了相同的handlers，handlers仅会在所有tasks结束后运行一次。如果playbook中间出现了错误，handlers也不会执行
2. 只有tasks发生了改变才会通知handlers，没有改变则不会触发handlers
3. 不能使用handlers代替tasks，因为handlers是一个特殊的tasks

### tags标签
正常情况下，playbook是从头执行到尾，如果只想执行playbook中某一个tasks，则需要用到tags标签。tags的三种使用场景：
```diff
+ 对一个tasks指定一个tags标签
+ 对一个tasks指定多个tags标签
+ 对多个tasks指定一个tags标签
```
示例：为playbook中的某个tasks设置标签然后单独执行此标签
```shell
- hosts: gui
  tasks:
    - name: Install NFS Server
      yum:
        name: nfs-utils
        state: present
      tags: install_nfs

    - name: Service NFS Server
      service: 
        name: nfs-server
        state: started
        enabled: yes
      tags: start_nfs-server

$ sudo ansible-playbook test.yml -t "install_nfs"
$ sudo ansible-playbook test.yml --skip-tags "install_nfs"
```
“-t”选项表示仅执行playbook中标记过此标签的tasks，“--skip-tags”选项表示执行playbook时跳过标记过此标签的tasks

### include文件复用
当多个playbook中都需要执行同一个步骤时，比如重启nginx服务，可以将重启nginx服务的tasks单独作为一个文件，所有的playbook都引用此文件。
示例：单独的引用文件
```shell
- name: Restart Httpd Service
  service: 
    name: httpd
    state: restarted
```
注：此文件中只有一个tasks，没有play的任何信息<br />
示例：项目的playbook内容
```shell
- hosts: gui
  tasks:
    - name: A Project comand
      command:
        cmd: echo "A"

    - name: Restart Httpd Service
      include_tasks: restart_httpd.yml
```
注：include_tasks也可更改为include，但建议使用include_tasks。<br />
除了include操作，还有一个import_playbook操作。include与import_playbook的区别在于include只会引用tasks，而import_playbook是引用完整的playbook文件。
示例：import_playbook操作
```shell
- import_playbook: ./test1.yml
- import_playbook: ./test2.yml
```
