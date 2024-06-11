# ansible部署LAMP

## 部署WEB应用

创建相关工作目录：

```bash
[ansibl]# mkdir httpd/{tasks,handlers,files,meta,templates} -p
```

编写的主任务文件：

- 安装httpd、php等软件
- 创建 web文档目录
- 上传应用源码并解压
- 根据变量文件中定义的数据库信息，修改应用配置文件
- 配置httpd虚拟主机

```
vim httpd/tasks/main.yaml
```

```
- name: install httpd package
  yum:
    name: "{{ item }}"
    state: present
  loop:
    - httpd
    - php
    - php-mysqlnd
    - php-gd

- name: create root dir
  file:
    path: "{{ webroot }}"
    state: directory
    owner: apache
    mode: 755
  ignore_errors: True

- name: Copy app
  unarchive:
    src: messageBoard.tar
    dest:  "{{ webroot }}"

- name: Change db config
  replace:
    path: "{{ webroot }}/config.php"
    regexp: "{{ item.src }}"
    replace: "{{ item.dest }}"
  loop:
    - { src: "172.31.200.41", dest: "{{groups.mysql[0]}}" }
    - { src: "message", dest: "{{app_mysql_database}}" }
    - { src: "user1", dest: "{{app_mysql_user}}" }
    - { src: "ROOT@123", dest: "{{app_mysql_passwd}}" }


- name: install configure file
  template:
    src: vhost.conf.j2
    dest: /etc/httpd/conf.d/vhost.conf
  notify:
    - restart httpd

- name: Start httpd
  systemd:
    name: httpd
    state: started
    enabled: yes
```

httpd虚拟主机的配置文件

```
vim  httpd/templates/vhost.conf.j2 
```

```
{% if http_port != 80 %}
Listen {{ http_port }}
{% endif %}
<VirtualHost *:{{ http_port }}>
    DocumentRoot "{{ webroot }}"
    ServerName {{ ansible_hostname }}
    ErrorLog "/var/log/httpd/{{ server_name }}-error_log"
    CustomLog "/var/log/httpd/{{ server_name }}-access_log" common
    <Directory "{{ webroot }}">
        require all  granted
    </Directory>
</VirtualHost>
```

编写handlers文件：

```yaml
[ansibl]# vim httpd/handlers/main.yml 
```

```
- name: restart httpd
  systemd:
    name: httpd
    state: restarted
```

变量文件如下：

```yaml
[ansibl]# vim group_vars/all 
```

```
rootpasswd: Admin@123
app_mysql_database: app
app_mysql_user: app
app_mysql_passwd: App@1234
app_mysql_host: 192.168.100.%

server_name: lamp.com
webroot: /webroot/
http_port: 80
```

playbook文件如下：

```yaml
[ansibl]# vim lamp.yaml 

- hosts: webserver
  roles:
    - role: httpd
  tags: httpd
```

测试运行playbook文件：

```bash
[ansibl]# ansible-playbook -t httpd lamp.yaml
```



## 数据库部署

建立mysql相关目录结构：

```bash
[ansibl]# mkdir mysql/{tasks,handlers,templates,files,meta} -p
```

编写mysql的安装任务：

- 安装MySQL
- 修改MySQL密码
- 导入数据
- 创建应用用户并授权到应用数据库

```
[ansibl]# vim mysql/tasks/main.yml 
```

```
- name: Copy mysql rpm package
  unarchive:
    src: mysql-5.7.40-1.el7.x86_64.rpm-bundle.tar
    dest:  /tmp/

- name: install MySQL
  yum:
    name:
      - /tmp/mysql-community-client-5.7.40-1.el7.x86_64.rpm
      - /tmp/mysql-community-common-5.7.40-1.el7.x86_64.rpm
      - /tmp/mysql-community-devel-5.7.40-1.el7.x86_64.rpm
      - /tmp/mysql-community-embedded-5.7.40-1.el7.x86_64.rpm
      - /tmp/mysql-community-embedded-compat-5.7.40-1.el7.x86_64.rpm
      - /tmp/mysql-community-embedded-devel-5.7.40-1.el7.x86_64.rpm
      - /tmp/mysql-community-libs-5.7.40-1.el7.x86_64.rpm
      - /tmp/mysql-community-libs-compat-5.7.40-1.el7.x86_64.rpm
      - /tmp/mysql-community-server-5.7.40-1.el7.x86_64.rpm
      - /tmp/mysql-community-test-5.7.40-1.el7.x86_64.rpm
      - MySQL-python
    state: installed
  register: result

- name: Include task list in play only if the condition is true
  ansible.builtin.include: "changepasswd.yml"
  when: result is change

- name: import a database
  mysql_db:
    login_host: "localhost"
    login_user: "root"
    login_password: "{{ rootpasswd }}"
    login_unix_socket: "/var/lib/mysql/mysql.sock"
    login_port: "3306"
    name: "{{app_mysql_database}}"
    target: "{{ webroot }}/message.sql"
    state: "import"
  ignore_errors: yes

- name: create mysql user
  no_log: true
  mysql_user:
    login_host: "localhost"
    login_port: 3306
    login_user: root
    login_password: "{{ rootpasswd }}"
    login_unix_socket: "/var/lib/mysql/mysql.sock"
    name: "{{ app_mysql_user }}"
    password: "{{ app_mysql_passwd }}"
    host: "{{ app_mysql_host }}"
    priv: "{{ app_mysql_database }}.*:ALL"
    state: present

- name: 复制配置文件
  template:
    src: my.cnf.j2
    dest: /etc/my.cnf
    backup: yes
  notify:
    - reload mysql
```

第一次启动MySQL时，需要先启动MySQL，随后修改root用户密码

```
[ansibl]# vim mysql/tasks/changepasswd.yml 
```

```
- name: reload mysql
  systemd:
    name: mysqld
    enabled: yes
    state: started

- name: change root passwd
  shell: |
    p=`grep 'password is generated for root@localhost' /var/log/mysqld.log | cut -d ' ' -f 11`
    mysqladmin -uroot -p"$p" password "{{rootpasswd}}"
```

MySQL配置文件，使用模板

```
[ansible]# vim mysql/templates/my.cnf.j2
```

```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock

symbolic-links=0

log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```

编写handlers文件：

```yaml
[ansibl]# vim mysql/handlers/main.yml 
```

```
- name: reload mysql
  systemd:
    name: mysqld
    enabled: yes
    state: restarted
    daemon_reload: yes
```

playbook文件如下：

```yaml
[ansibl]# vim  lamp.yaml

- hosts: mysql
  roles: 
    - role: mysql
  tags: mysql
```

运行palybook：

```bash
[ansibl]# ansible-playbook -t mysql webcluster.yaml
```





最终文件结构如下：

```
.
├── ansible.cfg
├── group_vars
│   └── all
├── hosts
├── httpd
│   ├── files
│   │   └── messageBoard.tar
│   ├── handlers
│   │   └── main.yml
│   ├── meta
│   ├── tasks
│   │   └── main.yaml
│   └── templates
│       └── vhost.conf.j2
├── lamp.yaml
└── mysql
    ├── files
    │   └── mysql-5.7.40-1.el7.x86_64.rpm-bundle.tar
    ├── handlers
    │   └── main.yml
    ├── meta
    ├── tasks
    │   ├── changepasswd.yml
    │   └── main.yml
    └── templates
        └── my.cnf.j2

13 directories, 13 files
```

整体执行任务：

```
[root@centos7-ansible ansible-lamp]# ansible-playbook lamp.yaml
```

