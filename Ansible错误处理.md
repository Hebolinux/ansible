Ansible错误处理
==============
正常情况下playbook执行过程中出错时，ansible会立即终止后续的tasks，但多数情况下执行过程中并不希望因为某一个tasks的错误而终止所有步骤，这就需要跳过tasks的报错继续向下执行。

ignore_errors错误忽略
--------------------
```shell
- hosts: gui
  tasks:
    - name: Command
      command: 
      	cmd: /bin/false
      ignore_errors: yes

    - name: Create File
      file:
        path: /tmp/ttt
        state: touch
```
ignore_errors也常用于判断语句，比如判断http服务是否启动，如果服务没有启动，则ansible判断此tasks错误，后续的tasks就无法再继续执行，类似此类情况就需要用到ignore_errors。

changed_when错误处理
-------------------
1. 强制调用handlers。handlers只会在playbook的最后执行，但如果触发notify后，后续的tasks出现了错误，那handlers也不会触发。强制调用handlers时，只要触发了notify则handlers就一定会执行
2. 关闭changed的状态changed_when，前提是确定该tasks不会对被控端做任何的修改和变更
3. 使用changed_when检查tasks任务返回的结果

示例：强制调用handlers
```shell
- hosts: gui
  force_handlers: yes

  tasks:
    - name: Touch File
      file: 
        path: /tmp/bgx_handlers
        state: touch
      notify: Restart Httpd Server

    - name: Install Packages
      yum:
        name: sb
        state: latest

  handlers:
    - name: Restart Httpd Server
      service:
        name: httpd
        state: restarted
```

示例：关闭changed的状态changed_when
```shell
- hosts: gui
  tasks:
    - name: Install Httpd Server
      yum:
        name: httpd
        state: present

    - name: Service Httpd Server
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Check Httpd Server
      shell: 
        cmd: ps aux | grep httpd
      register: check_httpd
      changed_when: false

    - name: OutPut Variables
      debug: 
        msg: "{{ check_httpd.stdout_lines }}"
```

示例：使用changed_when检查tasks任务返回的结果
```shell
- hosts: gui
  tasks:

    - name: Install Nginx Server
      yum:
        name: nginx
        state: present

    - name: Configure Nginx Server
      copy: 
        src: ./nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Restart Nginx Server

    - name: Check Nginx Configure Status
      command: 
        cmd: /usr/sbin/nginx -t 
      register: check_nginx
      changed_when: 
        - ( check_nginx.stdout.find('successful') )
        - false

#    - name: Debug
#      debug: 
#        msg: "{{ check_nginx }}"

    - name: Service Nginx Server
      service:
        name: nginx
        state: started

  handlers:
    - name: Restart Nginx Server 
      service:
        name: nginx
        state: restarted
```
changed_when会检查nginx的配置文件的格式输出是否为successful，如果不是successful则会报错，ansible会停止往下执行tasks。