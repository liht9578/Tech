``` toc
title: "## 知识目录"

```

###### 参考资料
- https://www.sjkjc.com/posts/install-mysql8-on-centos-stream-9/
- https://blog.csdn.net/u011868279/article/details/125512994
- https://blog.csdn.net/qq_23859799/article/details/85862821
- https://blog.csdn.net/michaelxguo/article/details/127676384
- https://blog.csdn.net/m0_74021233/article/details/135463757
- https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/install-redis-from-source/
###### Centos 9安装mysql8
``` shell
sudo dnf update -y
sudo dnf install -y wget

wget https://repo.mysql.com/mysql80-community-release-el9-1.noarch.rpm
sudo rpm -ivh mysql80-community-release-el9-1.noarch.rpm

sudo dnf config-manager --disable mysql57-community
sudo dnf config-manager --disable mysql56-community
sudo dnf config-manager --enable mysql80-community

sudo dnf install -y mysql-community-server --nogpgcheck

sudo systemctl start mysqld
sudo systemctl enable mysqld

sudo systemctl status mysqld

sudo grep 'temporary password' /var/log/mysqld.log

# 登录Mysql
mysql -u root -p

# 更改密码
ALTER USER 'root'@'localhost' IDENTIFIED BY 'new_password';

# 添加用户
create user 'tlc'@'%' identified by 'TlcUser@123';

# 用户授权
grant all privileges on *.* to 'tlc'@'%' with grant option;

# 刷新授权
flush privileges;

```
###### Centos 9安装openJDK 1.8
``` bash
# 安装命令
sudo dnf install -y java-1.8.0-openjdk

# 安装路径
/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.362.b09-4.el9.x86_64/jre/bin/java

# 别名
alias java8='/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.362.b09-4.el9.x86_64/jre/bin/java'

```
###### Centos 9安装redis
``` bash
# 下载redis
wget https://download.redis.io/redis-stable.tar.gz

# 解压缩并安装
tar zxvf redis-stable.tar.gz
make
make install

# 安装完成后的安装路径:/usr/local/bin/redis-server
# 配置文件路径：/etc/redis/redis.conf

# 修改配置文件
port 63790
bind 0.0.0.0
protected-mode no
daemonize yes
requirepass tlc@123

# 启动redis
sudo /usr/local/bin/redis-server /etc/redis/redis.conf

# 创建systemd服务文件
sudo vim /etc/systemd/system/local-redis.service

# 文件内容如下
[Unit]
Description=Local Redis
After=network.target
 
[Service]
Type=forking
ExecStart=/usr/local/bin/redis-server /etc/redis/redis.conf 
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true
 
[Install]
WantedBy=multi-user.target

# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 立即启动自定义 Nginx
sudo systemctl start local-redis

# 设置开机自启
sudo systemctl enable local-redis

# 验证是否生效
sudo systemctl status local-redis

```
###### Centos 9安装nginx并配置开机启动
``` shell
# 安装nginx
sudo dnf install nginx

# 安装文件路径：/usr/sbin/nginx
# 配置文件路径：/etc/nginx/nginx.conf

# 启动命令
sudo /usr/sbin/nginx -c /etc/nginx/nginx.conf

# 创建systemd服务文件
sudo vim /etc/systemd/system/custom-nginx.service

# 文件内容如下
# 注意：执行时使用root账号，所以启动命令不用sudo /usr/sbin/nginx
[Unit]
Description=Custom Nginx Server
After=network.target

[Service]
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -QUIT $MAINPID
Restart=always
Type=forking
PIDFile=/var/run/nginx_custom.pid

[Install]
WantedBy=multi-user.target

# 修改nginx.conf配置文件
user root;
pid /var/run/nginx_custom.pid;

# 启动时会报无法绑定8888的端口，[emerg] 23908#23908: bind() to 0.0.0.0:8888 failed (13: Permission denied)
# CentOS 9 默认启用了 **SELinux**，它可能禁止 Nginx 监听 `8888` 端口。
# 执行以下命令检查 SELinux 是否限制了端口：
sudo semanage port -l | grep http

# 如果 `8888` 端口 **不在允许的端口列表中**，需要手动添加：
sudo semanage port -a -t http_port_t -p tcp 8888

# 如果 `semanage` 命令不存在**，先安装：
sudo dnf install -y policycoreutils-python-utils

# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 立即启动自定义 Nginx
sudo systemctl start custom-nginx

# 设置开机自启
sudo systemctl enable custom-nginx

# 验证是否生效
sudo systemctl status custom-nginx

# 取消开机自启
sudo systemctl disable custom-nginx

# 立即停止服务
sudo systemctl stop custom-nginx

# 删除服务文件
sudo rm /etc/systemd/system/custom-nginx.service

# 重新加载 systemd
sudo systemctl daemon-reload

```
###### Centos 9开启启动nginx问题排查
``` shell
# 手动启动nginx没有问题，使用systemctl启动则会报错，错误日志为0.0.0.0:8801 failed (13: Permission denied)

# 解决方案：
# 如果系统启用了 SELinux，可能是它阻止了 Nginx 绑定端口 `8801`。检查 SELinux 状态
sestatus

# 如果 `SELinux status` 显示 `enabled`，可以临时禁用：
sudo setenforce 0
sudo systemctl stop custom-nginx.service
sudo systemctl start custom-nginx.service

# 如果这样能启动，说明是 SELinux 限制了端口绑定，执行：
sudo semanage port -a -t http_port_t -p tcp 8801
sudo semanage port -a -t http_port_t -p tcp 8802
sudo semanage port -a -t http_port_t -p tcp 9901
sudo semanage port -a -t http_port_t -p tcp 9902

# 然后恢复 SELinux：
sudo setenforce 1

# 检查 SELinux 是否阻止了 Nginx, 可以使用下面命令
sudo ausearch -m AVC -c nginx --raw | audit2why

# 如果存在type=AVC ... denied { name_connect } for pid=xxxxx comm="nginx" ...则说明有问题

# 使用下面命令，允许 Nginx 代理网络请求，然后重启nginx，则可以生效了
sudo setsebool -P httpd_can_network_connect on

```
