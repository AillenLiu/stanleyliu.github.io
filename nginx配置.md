#淘宝tengine配置
```
upstream www {
    ip_hash;
    server 114.119.6.145:8080;
    server 114.119.6.146:8080;
    check interval=3000 rise=2 fall=5 timeout=1000;
}
server {
    listen 80;
    server_name  www.haoeshang.com;
    index  index.html index.htm index.php index.jsp;

location ~ .*\.(ihtml|php|jsp|cgi)?$ {
    index  index.html index.htm index.php index.jsp;

    if ($http_cookie ~* "(.*)$") {
       set $www_cookie $1;
    }

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header HTTP_X_FORWARDED_FOR $remote_addr;
    proxy_pass http://www;
    error_log   /var/log/nginx/haoeshang.com_error.log;
    access_log /var/log/nginx/haoeshang.com_access.log main;
}

location ~ .*\.(html|htm|js|css|JPG|gif|jpg|jpeg|bmp|png|ico|swf|apk)$ {
    index  index.html index.htm index.php index.jsp;
    root /home/data/www;
    error_log   /var/log/nginx/haoeshang.com_error.log;
    access_log /var/log/nginx/haoeshang.com_access.log main;
}

}
```

## 一、 概述
1. 整体网络架构
nginx +keepalived+memcache+tomcat实现负载均衡双机互备和动静分离
2. 描述
	客户请求流量经过防火墙上时，防火墙检测本地规则和NAT记录并将流量发送到NAT本地地址10.100.10.48，当48及vip接受到请求的时候，会由45或者46（keepalived监控，如果一台服务器无法访问，自动切换到另一台）上的nginx进行判断是静态还是动态网页，如果是静态，将链接平均分配给本地nginx发布处理，动态网页则平均分配给50、51的8080端口（tomcat服务）处理。
3. 整个拓补

## 二、 环境说明
服务器：10.100.10.45安装keepaliced(心跳监控)、memcache(数据缓存)、nginx(负载均衡反向代理、静态处理)  
服务器：10.100.10.46安装keepaliced(心跳监控)、memcache(数据缓存)、nginx(负载均衡反向代理、静态处理)   
服务器：10.100.10.50安装tomcat(动态处理)   
服务器：10.100.10.51安装tomcat(动态处理)   
服务器：10.100.10.40安装solr搜索服务  
服务器：10.100.10.30安装oracle数据库服务
## 三、 10.100.10.45服务器配置（下简称45）
45服务安装软件有
	nginx：负责解析静态资源及向50.51转发动态资源实现50、51的负载均衡
	keepaliced:心跳监控，监测本机的nginx服务
	memcache：数据缓存服务
### (一) Nginx服务
1. 下载Tengine 
` wget http://tengine.taobao.org/download/tengine-2.1.0.tar.gz `
关于Tengine：Tengine是由淘宝网发起的Web服务器项目。它在Nginx的基础上，针对大访问量网站的需求，添加了很多高级功能和特性。Tengine的性能和稳定性已经在大型的网站如淘宝网，天猫商城等得到了很好的检验。它的最终目标是打造一个高效、稳定、安全、易用的Web平台
2. 安装需要的组件
` yum -y install gcc gcc-c++ openssl-devel zlib-devel pcre-devel autoconf automake ncurses `
3. 创建nginx用户
` useradd -s /sbin/nologin -M nginx `
4. 创建www用户
`useradd -s /sbin/nologin -M www`
5. 解压并编译安装
`tar zxvf tengine-2.1.0.tar.gz`
`cd tengine-2.1.0`
6. 顺序执行以下命令
` ./configure `
` make `
` make install `
7. 创建启动脚本，将nginx安装为Linux服务
通过vi命令创建启动脚本
` vi /etc/init.d/nginx `
8. 将以下内容拷贝到vi编辑器中（或者在window中新建nginx，将以下内容拷贝到文件中，然后上传到/etc/ init.d目录） 

```
#!/bin/sh
#
# nginx        Startup script for nginx
#
# chkconfig: - 85 15
# processname: nginx
# config: /etc/nginx/nginx.conf
# config: /etc/sysconfig/nginx
# pidfile: /var/run/nginx.pid
# description: nginx is an HTTP and reverse proxy server
#
### BEGIN INIT INFO
# Provides: nginx
# Required-Start: $local_fs $remote_fs $network
# Required-Stop: $local_fs $remote_fs $network
# Default-Start: 2 3 4 5
# Default-Stop: 0 1 6
# Short-Description: start and stop nginx
### END INIT INFO
 
# Source function library.
. /etc/rc.d/init.d/functions
 
if [ -f /etc/sysconfig/nginx ]; then
    . /etc/sysconfig/nginx
fi
 
prog=nginx
nginx=${NGINX-/usr/local/nginx/sbin/nginx}
conffile=${CONFFILE-/usr/local/nginx/conf/nginx.conf}
lockfile=${LOCKFILE-/var/lock/subsys/nginx}
pidfile=${PIDFILE-/var/run/nginx.pid}
SLEEPMSEC=100000
RETVAL=0
 
start() {
    echo -n $"Starting $prog: "
 
    daemon --pidfile=${pidfile} ${nginx} -c ${conffile}
    RETVAL=$?
    echo
    [ $RETVAL = 0 ] && touch ${lockfile}
    return $RETVAL
}
 
stop() {
    echo -n $"Stopping $prog: "
    killproc -p ${pidfile} ${prog}
    rm -rf /tmp/tcmalloc.*
    RETVAL=$?
    echo
    [ $RETVAL = 0 ] && rm -f ${lockfile} ${pidfile} /tmp/tcmalloc*
}
 
reload() {
    echo -n $"Reloading $prog: "
    killproc -p ${pidfile} ${prog} -HUP
    RETVAL=$?
    echo
}
 
upgrade() {
    oldbinpidfile=${pidfile}.oldbin
 
    configtest -q || return 6
    echo -n $"Staring new master $prog: "
    killproc -p ${pidfile} ${prog} -USR2
    RETVAL=$?
    echo
    /bin/usleep $SLEEPMSEC
    if [ -f ${oldbinpidfile} -a -f ${pidfile} ]; then
        echo -n $"Graceful shutdown of old $prog: "
        killproc -p ${oldbinpidfile} ${prog} -QUIT
        RETVAL=$?
        echo 
    else
        echo $"Upgrade failed!"
        return 1
    fi
}
 
configtest() {
    if [ "$#" -ne 0 ] ; then
        case "$1" in
            -q)
                FLAG=$1
                ;;
            *)
                ;;
        esac
        shift
    fi
    ${nginx} -t -c ${conffile} $FLAG
    RETVAL=$?
    return $RETVAL
}
 
rh_status() {
    status -p ${pidfile} ${nginx}
}
 
# See how we were called.
case "$1" in
    start)
        rh_status >/dev/null 2>&1 && exit 0
        start
        ;;
    stop)
        stop
        ;;
    status)
        rh_status
        RETVAL=$?
        ;;
    restart)
        configtest -q || exit $RETVAL
        stop
        start
        ;;
    upgrade)
        upgrade
        ;;
    condrestart|try-restart)
        if rh_status >/dev/null 2>&1; then
            stop
            start
        fi
        ;;
    force-reload|reload)
        reload
        ;;
    configtest)
        configtest
        ;;
    *)
        echo $"Usage: $prog {start|stop|restart|condrestart|try-restart|force-reload|upgrade|reload|status|help|configtest}"
        RETVAL=2
esac
 
exit $RETVAL
```
8. 修改文件权限为可执行 chmod +x /etc/init.d/nginx
9. Nginx配置文件修改 ，进行负载均衡配置
` vi  /usr/local/nginx/conf/nginx.conf  `
  将nginx.conf修改为以下内容
  
```
user www;
worker_processes     auto;
worker_cpu_affinity  auto;
worker_rlimit_nofile 65535;
error_log  /usr/local/nginx/error.log warn;
pid        /var/run/nginx.pid;
#google_perftools_profiles /tmp/tcmalloc;
 
events {
    use epoll;
    worker_connections  20480;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
 
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
 
    #access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    server_tokens   off;
 
    tcp_nodelay        on;
    tcp_nopush     on;
    keepalive_timeout  65;
    gzip  on;
    gzip_disable "msie6";
    gzip_http_version 1.1;
    gzip_vary on;
    gzip_comp_level 6;
    gzip_proxied any;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript text/x$
    gzip_buffers 16 8k;
    proxy_connect_timeout 600;
    proxy_read_timeout 600;
    proxy_send_timeout 600;
    proxy_buffers 4 32k;
    proxy_busy_buffers_size 64k;
    proxy_temp_file_write_size 64k;
 proxy_temp_path proxy_temp;
    client_max_body_size 10m;
    client_body_buffer_size 128k;
    include conf.d/*.conf;
}
```

9. B2B服务负载均衡配置  
` vi  /usr/local/nginx/conf/conf.d/yrguoshu.com.conf `

```
#设定负载均衡的服务器列表
upstream www {
    ip_hash;
    server 10.100.10.50:8080;
    server 10.100.10.51:8080;
    check interval=3000 rise=2 fall=5 timeout=1000;
}
server {
    listen 80;
    server_name  www.yrguoshu.com pic.yrguoshu.com;
    index  index.html index.htm index.php index.jsp;
    
location ~ .*\.(ihtml|php|jsp|cgi)?$ {
    index  index.html index.htm index.php index.jsp;
    
    if ($http_cookie ~* "(.*)$") {
       set $www_cookie $1;
    }
 
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header HTTP_X_FORWARDED_FOR $remote_addr;
    proxy_pass http://www;
    error_log   /var/log/nginx/yrguoshu.com_error.log;
    access_log /var/log/nginx/yrguoshu.com_access.log main;
}
location ~ .*\.(html|htm|js|css|JPG|gif|jpg|jpeg|bmp|png|ico|swf|apk)$ {
    index  index.html index.htm index.php index.jsp;
    root /home/data/www/Yurun-B2B-Mall;
    error_log   /var/log/nginx/yrguoshu.com_error.log;
    access_log /var/log/nginx/yrguoshu.com_access.log main;
}
}
#对weixin二级域名进行转发
server {
    listen       80;
    server_name  weixin.yrguoshu.com;
    index  index.html index.htm index.php index.jsp;
  
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header HTTP_X_FORWARDED_FOR $remote_addr;
    proxy_pass http://10.100.10.110:8080;
    error_log   /var/log/nginx/weixin.yrguoshu.com_error.log;
    access_log /var/log/nginx/weixin.yrguoshu.com_access.log main;
}
}
#对 funlohas二级域名进行转发
server {
    listen       80;
    server_name  funlohas.yrguoshu.com;
    index  index.html index.htm index.php index.jsp;
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header HTTP_X_FORWARDED_FOR $remote_addr;
    proxy_pass http://10.100.10.111:8080;
    error_log   /var/log/nginx/funlohas.yrguoshu.com_error.log;
    access_log /var/log/nginx/funlohas.yrguoshu.com_access.log main;
}
location = /favicon.ico {
 log_not_found off;
 access_log off;
}
}
```

10. B2C服务负载均衡配置

` vi  /usr/local/nginx/conf/conf.d/yrcailanzi.cn.conf `

```
upstream yrcailanzi {
    ip_hash;
server 10.100.10.50:8081;
    server 10.100.10.51:8081;
    check interval=3000 rise=2 fall=5 timeout=1000;
}
 
server {
    listen  80;
    server_name  yrcailanzi.cn;
    index  index.html index.htm index.php index.jsp;
    
location ~ .*\.(ihtml|php|jsp|cgi|sc|jhtml)?$ {
    index  index.html index.htm index.php index.jsp;
    
    if ($http_cookie ~* "(.*)$") {
       set $www_cookie $1;
    }
 
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header HTTP_X_FORWARDED_FOR $remote_addr;
    proxy_pass http://yrcailanzi;
    error_log   /var/log/nginx/yrcailanzi.cn_error.log;
    access_log /var/log/nginx/yrcailanzi.cn_access.log main;
}
location ~ .*\.(html|htm|js|css|JPG|gif|jpg|jpeg|bmp|png|ico|swf|apk)$ {
    index  index.html index.htm index.php index.jsp;
    root /home/data/www/yrcailanzi.cn;
    error_log   /var/log/nginx/yrcailanzi.cn_error.log;
    access_log /var/log/nginx/yrcailanzi.cn_access.log main;
}
}
```

11. Nginx维护信息  
部署路径：``` /usr/local/nginx ```  
主要配置文件：``` /usr/local/nginx/conf/conf.d/yrguoshu.com.conf ```  
主要配置文件：``` /usr/local/nginx/conf/conf.d/ yrcailanzi.cn.conf ```  
动态资源分发主机：  
10.100.10.50  
10.100.10.51  
静态资源发布路径：（注意后台上传相关静态资源路径在这里）  
[www.yrguoshu.com](www.yrguoshu.com)  静态资源路径: ` /home/data/www/Yurun-B2B-Mall `   
[www.yrcailanzi.cn](www.yrcailanzi.cn) 静态资源路径: ` /home/data/www/yrcailanzi.cn `  
日志保存路径：` /var/log/nginx/ `  
启动方式：``` service nginx {start|stop|restart|reload|status} ```

### (二) Keepalived服务
1. 下载keeplived
``` wget http://www.keepalived.org/software/keepalived-1.2.12.tar.gz ```  
``` tar zxvf keepalived-1.2.12.tar.gz ```  
``` cd keepalived-1.2.12 ```  
``` ./configure --sysconf=/usr/local/keepalived/etc --prefix=/usr/local/keepalived  --sharedstatedir=/usr/local/keepalived ```  
``` make ```  
``` make install ```
 
2. 修改keepalived配置文件  
``` vi /usr/local/keepalived/etc/keepalived/keepalived.conf ```   

``` 
! Configuration File for keepalived
 
global_defs {
notification_email {
ncpliuming@yurun.com
 
}
notification_email_from yurun.kp@qq.com
smtp_server 127.0.0.1
smtp_connect_timeout 30
router_id LVS_DEVEL
}
 
vrrp_script chk_nginx {
script "/etc/keepalived/check_nginx.sh"
interval 2
weight 2
}
vrrp_instance VI_1 {
state MASTER
interface eth0
virtual_router_id 51
priority 100
mcast_src_ip 10.100.10.45
advert_int 1
authentication {
auth_type PASS
auth_pass abcd
}
 
track_script {
chk_nginx
}
 
virtual_ipaddress {
10.100.10.48
}
 
notify_master "/etc/keepalived/clean_arp.sh  10.100.10.48"
}
}
```

3. 创建nginx进程监测脚本  
` vi /usr/local/keepalived/check_nginx.sh `  

```
#!/bin/bash
if [ "$(ps -ef | grep "nginx: master process"| grep -v grep )" == "" ]
then
 /usr/local/nginx/sbin/nginx
 sleep 5
 if [ "$(ps -ef | grep "nginx: master process"| grep -v grep )" == "" ]
 then
 killall keepalived
 fi
fi
```
4. 创建ip地址检查脚本（检测48虚拟ip是否被占用）
` vi /usr/local/keepalived/clean_arp.sh `  

``` 
#!/bin/sh
VIP=$1
GATEWAY=10.100.10.1
/sbin/arping -I bond0 -c 5 -s $VIP $GATEWAY &>/dev/null
```
5. 将Keepalived安装成Linux服务
依次执行下列命令
` #cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/rc.d/init.d/  `<br>
` #cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/  `<br>
` #mkdir /etc/keepalived `<br>
` #cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/ `<br>
` #cp /usr/local/keepalived/check_nginx.sh /etc/keepalived/ `<br>
` #cp /usr/local/keepalived/clean_arp.sh /etc/keepalived/ `<br>
` #cp /usr/local/keepalived/sbin/keepalived /usr/sbin/ `
设定执行权限
` chmod +x /etc/rc.d/init.d/keepalived `
通过以下命令启动、停止keepalived  
` service keepalived start/stop/restart `  
keepalived启动后，通过ip addr命令可以查看到10.100.10.48已经绑定到主机上。
 
### (三) Memcache服务
1. 安装  
安装libevent  ` yum install libevent libevent-devel -y `  
` wget http://www.memcached.org/files/memcached-1.4.20.tar.gz `  
` tar zxvf memcached-1.4.20.tar.gz `  
` cd memcached-1.4.20 `  
` ./configure `  
` make && make install `  
` chmod +x /usr/local/bin/memcached `  

2. 启动  
` /usr/local/bin/memcached -d -m 256 -u root -l 10.100.10.45 -p 11211 -c 2048 -P /tmp/memcached.pid `  
没错误提示的话，证明安装成功并且启动了Memcached服务了。
结束Memcached进程使用如下命令：
``` kill `cat /tmp/memcached.pid` ```  
## (四) Rsync文件同步服务
45作为主机，46作为客户机，当45目录中的文件变化时，46上的任务将文件同步备份至46服务器对应的目录中
1. 下载安装rsync
` wget http://rsync.samba.org/ftp/rsync/src/rsync-3.1.0.tar.gz `  
` tar zxvf rsync-3.1.0.tar.gz `  
` cd rsync-3.1.0 `  
` ./configure --prefix=/usr/local/rsync --with-rsyncd-conf=/usr/local/rsync/etc/rsyncd.conf `  
` make `  
` make install `
 
2. 创建rsync服务端配置文件
进入/usr/local/rsync 目录  
` mkdir etc `  
` mkdir pid `  
` mkdir  secrets `  
` mkdir socket `  
创建配置文件  
` vi /etc/rsyncd.conf `  
```
motd file = /usr/local/rsync/etc/rsyncd.motd
pid file = /usr/local/rsync/pid/rsyncd.pid 
port = 873
address = 10.100.10.45
socket options = /usr/local/rsync/socket/rsyncd.socket
uid = root
gid = root
use chroot = no
 
[Yurun-B2B-Mall]
comment = this is a Yurun-B2B-Mall reposity
path = /home/data/www/Yurun-B2B-Mall
use chroot = false
max connections = 0
log file = /usr/local/rsync/log/rsyncd.log
syslog facility = syslog
lock file = /usr/local/rsync/lock/rsyncd.lock
read only = true
write only = false
list = false
uid = root
gid = root
 
auth users = root
secrets file = /usr/local/rsync/secrets/rsyncd.secrets
strict modes = true
hosts allow = 10.100.10.46
hosts deny = *
ignore errors = true
ignore nonreadable = false
transfer logging = true
log format = %a-%b-%f-%m-%o-%P-%t-%u
timeout = 600
dont compress = true
 
 
[yrcailanzi.cn]
comment = this is a yrcailanzi.cn reposity
path = /home/data/www/yrcailanzi.cn
use chroot = false
max connections = 0
log file = /usr/local/rsync/log/rsyncd.log
syslog facility = syslog
lock file = /usr/local/rsync/lock/rsyncd.lock
read only = true
write only = false
list = false
uid = root
gid = root
 
auth users = root
secrets file = /usr/local/rsync/secrets/rsyncd.secrets
strict modes = true
hosts allow = 10.100.10.46
hosts deny = *
ignore errors = true
ignore nonreadable = false
transfer logging = true
log format = %a-%b-%f-%m-%o-%P-%t-%u
timeout = 600
dont compress = true
 ```
创建认证文件
` vi /secrets/rsyncd.secrets `  
` root:abcd1234 `
 
设置文件权限
` chown root:root /usr/local/rsync/secrets/rsyncd.secrets `  
` chmod 600 /usr/local/rsync/secrets/rsyncd.secrets `  
` chmod +x /usr/local/rsync/bin/rsync `
3. 将服务加入到开机启动
` /usr/local/rsync/bin/rsync --daemon --config=/usr/local/rsync/etc/rsyncd.conf `  
## 四、 10.100.10.46服务器配置（下简称46）
46服务安装软件有  
	nginx：负责解析静态资源及向50.51转发动态资源实现50、51的负载均衡  
	keepaliced:心跳监控，监测本机的nginx服务  
	memcache：数据缓存服务  
### (一) Nginx服务
1. 下载Tengine 
` wget http://tengine.taobao.org/download/tengine-2.1.0.tar.gz `
关于Tengine：Tengine是由淘宝网发起的Web服务器项目。它在Nginx的基础上，针对大访问量网站的需求，添加了很多高级功能和特性。Tengine的性能和稳定性已经在大型的网站如淘宝网，天猫商城等得到了很好的检验。它的最终目标是打造一个高效、稳定、安全、易用的Web平台
2. 安装需要的组件
` yum -y install gcc gcc-c++ openssl-devel zlib-devel pcre-devel autoconf automake ncurses `
3. 创建nginx用户
` useradd -s /sbin/nologin -M nginx `
4. 创建www用户
` useradd -s /sbin/nologin -M www `
5. 解压并编译安装
` tar zxvf tengine-2.1.0.tar.gz `  
` cd tengine-2.1.0 `
顺序执行以下命令
` ./configure  `
` make `  
` make install `  
6. 创建启动脚本，将nginx安装为Linux服务
通过vi命令创建启动脚本
` vi /etc/init.d/nginx `
将以下内容拷贝到vi编辑器中（或者在window中新建nginx，将以下内容拷贝到文件中，然后上传到/etc/ init.d目录） 

```
#!/bin/sh
#
# nginx        Startup script for nginx
#
# chkconfig: - 85 15
# processname: nginx
# config: /etc/nginx/nginx.conf
# config: /etc/sysconfig/nginx
# pidfile: /var/run/nginx.pid
# description: nginx is an HTTP and reverse proxy server
#
### BEGIN INIT INFO
# Provides: nginx
# Required-Start: $local_fs $remote_fs $network
# Required-Stop: $local_fs $remote_fs $network
# Default-Start: 2 3 4 5
# Default-Stop: 0 1 6
# Short-Description: start and stop nginx
### END INIT INFO
 
# Source function library.
. /etc/rc.d/init.d/functions
 
if [ -f /etc/sysconfig/nginx ]; then
    . /etc/sysconfig/nginx
fi
 
prog=nginx
nginx=${NGINX-/usr/local/nginx/sbin/nginx}
conffile=${CONFFILE-/usr/local/nginx/conf/nginx.conf}
lockfile=${LOCKFILE-/var/lock/subsys/nginx}
pidfile=${PIDFILE-/var/run/nginx.pid}
SLEEPMSEC=100000
RETVAL=0
 
start() {
    echo -n $"Starting $prog: "
 
    daemon --pidfile=${pidfile} ${nginx} -c ${conffile}
    RETVAL=$?
    echo
    [ $RETVAL = 0 ] && touch ${lockfile}
    return $RETVAL
}
 
stop() {
    echo -n $"Stopping $prog: "
    killproc -p ${pidfile} ${prog}
    rm -rf /tmp/tcmalloc.*
    RETVAL=$?
    echo
    [ $RETVAL = 0 ] && rm -f ${lockfile} ${pidfile} /tmp/tcmalloc*
}
 
reload() {
    echo -n $"Reloading $prog: "
    killproc -p ${pidfile} ${prog} -HUP
    RETVAL=$?
    echo
}
 
upgrade() {
    oldbinpidfile=${pidfile}.oldbin
 
    configtest -q || return 6
    echo -n $"Staring new master $prog: "
    killproc -p ${pidfile} ${prog} -USR2
    RETVAL=$?
    echo
    /bin/usleep $SLEEPMSEC
    if [ -f ${oldbinpidfile} -a -f ${pidfile} ]; then
        echo -n $"Graceful shutdown of old $prog: "
        killproc -p ${oldbinpidfile} ${prog} -QUIT
        RETVAL=$?
        echo 
    else
        echo $"Upgrade failed!"
        return 1
    fi
}
 
configtest() {
    if [ "$#" -ne 0 ] ; then
        case "$1" in
            -q)
                FLAG=$1
                ;;
            *)
                ;;
        esac
        shift
    fi
    ${nginx} -t -c ${conffile} $FLAG
    RETVAL=$?
    return $RETVAL
}
 
rh_status() {
    status -p ${pidfile} ${nginx}
}
 
# See how we were called.
case "$1" in
    start)
        rh_status >/dev/null 2>&1 && exit 0
        start
        ;;
    stop)
        stop
        ;;
    status)
        rh_status
        RETVAL=$?
        ;;
    restart)
        configtest -q || exit $RETVAL
        stop
        start
        ;;
    upgrade)
        upgrade
        ;;
    condrestart|try-restart)
        if rh_status >/dev/null 2>&1; then
            stop
            start
        fi
        ;;
    force-reload|reload)
        reload
        ;;
    configtest)
        configtest
        ;;
    *)
        echo $"Usage: $prog {start|stop|restart|condrestart|try-restart|force-reload|upgrade|reload|status|help|configtest}"
        RETVAL=2
esac
 
exit $RETVAL
```  
修改文件权限为可执行
` chmod +x /etc/init.d/nginx `  
7. Nginx配置文件修改 ，进行负载均衡配置
` vi  /usr/local/nginx/conf/nginx.conf `  
   将nginx.conf修改为以下内容
```
user www;
worker_processes     auto;
worker_cpu_affinity  auto;
worker_rlimit_nofile 65535;
error_log  /usr/local/nginx/error.log warn;
pid        /var/run/nginx.pid;
#google_perftools_profiles /tmp/tcmalloc;
 
events {
    use epoll;
    worker_connections  20480;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
 
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
 
    #access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    server_tokens   off;
 
    tcp_nodelay        on;
    tcp_nopush     on;
    keepalive_timeout  65;
    gzip  on;
    gzip_disable "msie6";
    gzip_http_version 1.1;
    gzip_vary on;
    gzip_comp_level 6;
    gzip_proxied any;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript text/x$
    gzip_buffers 16 8k;
    proxy_connect_timeout 600;
    proxy_read_timeout 600;
    proxy_send_timeout 600;
    proxy_buffers 4 32k;
    proxy_busy_buffers_size 64k;
    proxy_temp_file_write_size 64k;
 proxy_temp_path proxy_temp;
    client_max_body_size 10m;
    client_body_buffer_size 128k;
    include conf.d/*.conf;
}
```  
8. B2B服务负载均衡配置
` vi  /usr/local/nginx/conf/conf.d/yrguoshu.com.conf `  
#设定负载均衡的服务器列表

```
upstream www {
    ip_hash;
    server 10.100.10.50:8080;
    server 10.100.10.51:8080;
    check interval=3000 rise=2 fall=5 timeout=1000;
}
server {
    listen 80;
    server_name  www.yrguoshu.com pic.yrguoshu.com;
    index  index.html index.htm index.php index.jsp;
    
location ~ .*\.(ihtml|php|jsp|cgi)?$ {
    index  index.html index.htm index.php index.jsp;
    
    if ($http_cookie ~* "(.*)$") {
       set $www_cookie $1;
    }
 
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header HTTP_X_FORWARDED_FOR $remote_addr;
    proxy_pass http://www;
    error_log   /var/log/nginx/yrguoshu.com_error.log;
    access_log /var/log/nginx/yrguoshu.com_access.log main;
}
location ~ .*\.(html|htm|js|css|JPG|gif|jpg|jpeg|bmp|png|ico|swf|apk)$ {
    index  index.html index.htm index.php index.jsp;
    root /home/data/www/Yurun-B2B-Mall;
    error_log   /var/log/nginx/yrguoshu.com_error.log;
    access_log /var/log/nginx/yrguoshu.com_access.log main;
}
}
#对weixin二级域名进行转发
server {
    listen       80;
    server_name  weixin.yrguoshu.com;
    index  index.html index.htm index.php index.jsp;
  
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header HTTP_X_FORWARDED_FOR $remote_addr;
    proxy_pass http://10.100.10.110:8080;
    error_log   /var/log/nginx/weixin.yrguoshu.com_error.log;
    access_log /var/log/nginx/weixin.yrguoshu.com_access.log main;
}
}
#对 funlohas二级域名进行转发
server {
    listen       80;
    server_name  funlohas.yrguoshu.com;
    index  index.html index.htm index.php index.jsp;
location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header HTTP_X_FORWARDED_FOR $remote_addr;
    proxy_pass http://10.100.10.111:8080;
    error_log   /var/log/nginx/funlohas.yrguoshu.com_error.log;
    access_log /var/log/nginx/funlohas.yrguoshu.com_access.log main;
}
location = /favicon.ico {
 log_not_found off;
 access_log off;
}
}
```

9. Nginx维护信息
部署路径：` /usr/local/nginx `  
主要配置文件：` /usr/local/nginx/conf/conf.d/yrguoshu.com.conf `  
主要配置文件：` /usr/local/nginx/conf/conf.d/ yrcailanzi.cn.conf `  
动态资源分发主机：  
10.100.10.50  
10.100.10.51  
静态资源发布路径：（注意后台上传相关静态资源路径在这里）  
www.yrguoshu.com静态资源路径:/home/data/www/Yurun-B2B-Mall  
www.yrcailanzi.cn静态资源路径:/home/data/www/yrcailanzi.cn   
日志保存路径：` /var/log/nginx/ `   
启动方式：` service nginx {start|stop|restart|reload|status} `  
 
### (二) Keepalived服务
1. 下载keeplived
` wget http://www.keepalived.org/software/keepalived-1.2.12.tar.gz `  
` tar zxvf keepalived-1.2.12.tar.gz `  
` cd keepalived-1.2.12 `  
` ./configure --sysconf=/usr/local/keepalived/etc --prefix=/usr/local/keepalived  --sharedstatedir=/usr/local/keepalived `  
` make `  
` make install `  
 
2. 修改keepalived配置文件
` vi /usr/local/keepalived/etc/keepalived/keepalived.conf `  

```
! Configuration File for keepalived
 
global_defs {
notification_email {
ncpliuming@yurun.com
 
}
notification_email_from yurun.kp@qq.com
smtp_server 127.0.0.1
smtp_connect_timeout 30
router_id LVS_DEVEL
}
 
vrrp_script chk_nginx {
script "/etc/keepalived/check_nginx.sh"
interval 2
weight 2
}
vrrp_instance VI_1 {
state MASTER
interface eth0
virtual_router_id 61
priority 100
mcast_src_ip 10.100.10.46
advert_int 1
authentication {
auth_type PASS
auth_pass abcd
}
 
track_script {
chk_nginx
}
 
virtual_ipaddress {
10.100.10.48
}
 
notify_master "/etc/keepalived/clean_arp.sh  10.100.10.48"
}
}
 ```
 
3. 创建nginx进程监测脚本
` vi /usr/local/keepalived/check_nginx.sh `  
```
#!/bin/bash
if [ "$(ps -ef | grep "nginx: master process"| grep -v grep )" == "" ]
then
 /usr/local/nginx/sbin/nginx
 sleep 5
 if [ "$(ps -ef | grep "nginx: master process"| grep -v grep )" == "" ]
 then
 killall keepalived
 fi
fi
```
 
4. 创建ip地址检查脚本（检测48虚拟ip是否被占用）
` vi /usr/local/keepalived/clean_arp.sh `  
```
#!/bin/sh
VIP=$1
GATEWAY=10.100.10.1
/sbin/arping -I bond0 -c 5 -s $VIP $GATEWAY &>/dev/null
```  
5. 将Keepalived安装成Linux服务
依次执行下列命令
` #cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/rc.d/init.d/ `  
` #cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/ `  
` #mkdir /etc/keepalived `  
` #cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/ `  
` #cp /usr/local/keepalived/check_nginx.sh /etc/keepalived/ `  
` #cp /usr/local/keepalived/clean_arp.sh /etc/keepalived/ `  
` #cp /usr/local/keepalived/sbin/keepalived /usr/sbin/ `
设定执行权限
` chmod +x /etc/rc.d/init.d/keepalived `  
通过以下命令启动、停止keepalived  
` service keepalived start/stop/restart `  
keepalived启动后，通过ip addr命令可以查看到10.100.10.48已经绑定到主机上。
 
### (三) Memcache服务
1. 安装
安装libevent   ` yum install libevent libevent-devel -y `  
` wget http://www.memcached.org/files/memcached-1.4.20.tar.gz ` 
` tar zxvf memcached-1.4.20.tar.gz `  
` cd memcached-1.4.20 `  
` ./configure `  
` make && make install `  
` chmod +x /usr/local/bin/memcached `  
2. 启动
` /usr/local/bin/memcached -d -m 256 -u root -l 10.100.10.46 -p 11211 -c 2048 -P /tmp/memcached.pid `  
没错误提示的话，证明安装成功并且启动了Memcached服务了。  
结束Memcached进程使用如下命令：  
` kill `cat /tmp/memcached.pid` `  
###(四) Rsync文件同步服务
45作为主机，46作为客户机，当45目录中的文件变化时，46上的任务将文件同步备份至46服务器对应的目录中
1. 下载安装rsync
` wget http://rsync.samba.org/ftp/rsync/src/rsync-3.1.0.tar.gz `   
` tar zxvf rsync-3.1.0.tar.gz `  
` cd rsync-3.1.0 `  
` ./configure --prefix=/usr/local/rsync --with-rsyncd-conf=/usr/local/rsync/etc/rsyncd.conf `  
` make `  
` make install `  
 
2. 创建rsync客户端配置文件
进入` /usr/local/rsync ` 目录
` mkdir etc `  
` mkdir pid `  
` mkdir  secrets `  
` mkdir socket `  
创建认证文件   
` vi /secrets/rsyncd.secrets `  
abcd1234  
创建同步脚本  
` vi /etc/rsyncd.sh `  

```
#!/bin/bash
/usr/local/rsync/bin/rsync -vzrtopg --progress --delete --password-file=/usr/local/rsync/secrets/rsyncd.secrets root@10.100.10.45::Yurun-B2B-Mall /home/data/www/Yurun-B2B-Mall
/usr/local/rsync/bin/rsync -vzrtopg --progress --delete --password-file=/usr/local/rsync/secrets/rsyncd.secrets root@10.100.10.45::yrcailanzi.cn /home/data/www/yrcailanzi.cn
```  
设置文件权限
` chown root:root /usr/local/rsync/secrets/rsyncd.secrets `  
` chmod 600 /usr/local/rsync/secrets/rsyncd.secrets `  
` chmod +x /usr/local/rsync/bin/rsync `  
` chmod +x  /usr/local/rsync/etc/rsyncd.sh `  
3. 创建定时任务
开打系统计划配置
` crontab –e `  
编辑如下内容（5分钟执行一次同步）
` */5 * * * * sh /usr/local/rsync/etc/rsyncd.sh `  
 
##五、 10.100.10.50服务器配置（下简称50）
50服务器安装java及tomcat，负责动态资源的解析
###(一) Java安装
通过命令` java –version `，查看主机是否安装有java，如果没有安装通过命令
` yum  install  java ` 选择7.0版本安装
###(二) 安装配置tomcat
1. 安装tomcat(系统已经有jdk)  
` wget http://mirrors.cnnic.cn/apache/tomcat/tomcat-6/v6.0.41/bin/apache-tomcat-6.0.41.tar.gz `  
` tar zxvf apache-tomcat-6.0.41.tar.gz `  
` mv apache-tomcat-6.0.41/ /usr/local/tomcat/ `
  
2. 启动/停止tomcat  
` /usr/local/tomcat/bin/startup.sh `     启动  
` /usr/local/ tomcat/bin/shutdown.sh `   停止

3. B2B tomcat虚拟主机配置  
 ```
      <Host name="www.yrguoshu.com"  appBase=""
            unpackWARs="true" autoDeploy="true">
            <Context path="" docBase="/home/data/www/Yurun-B2B-Mall" reloadable="true" debug="0" />
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="yrguoshu_access_log." suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
```

4. Tomcat维护信息  
www.yrguoshu.com
WEB服务
tomcat部署路径：` /usr/local/tomcat `  
tomcat启动脚本：` /usr/local/tomcat/bin/startup.sh `  
tomcat停止脚本：` /usr/local/tomcat/bin/startup.sh `  
项目发布路径：` /home/data/www/Yurun-B2B-Mall/ `  
tomcat端口：8080  
****
# 六、 10.100.10.51服务器配置（下简称51）
51服务器安装java及tomcat，负责动态资源的解析
## (一) Java安装
通过命令` java –version `，查看主机是否安装有java，如果没有安装通过命令
` yum  install  java ` 选择7.0版本安装
## (二) 安装配置tomcat
1. 安装tomcat(系统已经有jdk)  
` wget http://mirrors.cnnic.cn/apache/tomcat/tomcat-6/v6.0.41/bin/apache-tomcat-6.0.41.tar.gz `  
` tar zxvf apache-tomcat-6.0.41.tar.gz `  
` mv apache-tomcat-6.0.41/ /usr/local/tomcat/ `

2. 启动/停止tomcat 
` /usr/local/tomcat/bin/startup.sh `    启动  
` /usr/local/ tomcat/bin/shutdown.sh `  停止

3. B2B tomcat虚拟主机配置  
```
      <Host name="www.yrguoshu.com"  appBase=""
            unpackWARs="true" autoDeploy="true">
            <Context path="" docBase="/home/data/www/Yurun-B2B-Mall" reloadable="true" debug="0" />
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="yrguoshu_access_log." suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
```
4. Tomcat维护信息
[www.yrguoshu.com](www.yrguoshu.com)
WEB服务  
tomcat部署路径：` /usr/local/tomcat `  
tomcat启动脚本：` /usr/local/tomcat/bin/startup.sh `  
tomcat停止脚本：` /usr/local/tomcat/bin/startup.sh `  
项目发布路径：` /home/data/www/Yurun-B2B-Mall/ `  
tomcat端口：8080

