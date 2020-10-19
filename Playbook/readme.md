Playbook
=========
基本概述
-------
playbook本身只是一个用yaml语法编辑的文本文件，此文件必须以yml或yaml为后缀结尾。playbook由两部分组成：play和task，play定义的是主机的角色，task定义的是具体执行的任务。playbook是由一个或多个play组成，一个play可以包含多个task任务。<br />
task在整个playbook文件中占据了绝大部分，而play就可以简单的看作是"hosts"指定的主机或主机组，上述内容写一个playbook文件后就很好理解了。

### Playbook与AD-Hoc的关系
1. playbook是对AD-Hoc的一种编排方式
2. playbook可以持久运行
3. playbook适合复杂任务
4. playbook能控制任务执行的先后顺序

### Playbook书写格式
语法|描述
:-:|:-:
缩进|YAML固定缩进风格表示层级结构，每个缩进建议使用两个空格，不能使用tab
冒号|除了以冒号结尾，其他所有冒号后面所有必须有空格
短横线|表示列表项，使用一个短横线加一个空格。多个项使用同样的缩进级别作为同一列表。

### 启动http服务playbook示例
下面示例中Mount模块会用到Centos-7本地镜像，所以需要先连接镜像
```shell
# cat /home/hebo/learn_ansible/http.yml
- hosts: gui							#此处即为play，指定主机

  tasks:								#以下内容都是task，是具体的执行内容
    - name: Mount CDROM					#playbook中name仅表示对操作的注释
      mount:							#启用挂载模块，下面都是挂载模块下的选项
        path: /mnt/cdrom
        src: /dev/sr0
        fstype: iso9660
        opts: defaults
        state: mounted

    - name: Install Httpd Server
      yum:
        name: httpd
        state: present

    - name: Create Http Group
      group:
        name: http
        state: present

    - name: Create Http User
      user:
        name: http
        state: present
        group: http
        shell: /sbin/nologin
        system: true

    - name: Configure Httpd Server
      copy:
        content: "It's work"
        dest: /var/www/html/index.html
        owner: http
        group: http
        mode: 644

    - name: Service Httpd Server
      service:
        service: httpd
        state: started
        enabled: yes

    - name: Down Selinux
      selinux:
        state: disabled

    - name: Configure Firewalld Server
      firewalld:
        service: http
        state: enabled
        permanent: yes
        immediate: yes
        zone: public
```

### 其他Playbook示例
名称|作用
:-:|:-:
nfs.yml|安装nfs服务
lamp-kod.yml|安装可道云云盘
