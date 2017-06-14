## centos7 部署 SpringBoot App

#### 安装 Vim

```
sudo yum install vim -y
```

#### 安装 Git

```
sudo yum install git -y
```

#### 安装 Java8

```
cd ~
wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u131-b11/d54c1d3a095b4ff2b6607d096fa80163/jdk-8u131-linux-x64.rpm?AuthParam=1497421784_8f51c8cfc0e264213d702755e7ed7e7f"

mv jdk-8u131-linux-x64.rpm\?AuthParam\=1497421784_8f51c8cfc0e264213d702755e7ed7e7f jdk-8u131-linux-x64.rpm
sudo yum localinstall jdk-8u131-linux-x64.rpm -y
rm jdk-8u131-linux-x64.rpm
```
#### 安装 Nginx (+ngx_pagespeed+ssl+http2)

```
sudo yum -y update  
sudo reboot

sudo yum -y install gcc-c++ pcre-devel zlib-devel make unzip

NPS_VERSION=1.12.34.2
NGINX_VERSION=1.12.0
wget http://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz
wget https://github.com/pagespeed/ngx_pagespeed/archive/v${NPS_VERSION}-beta.zip

tar -xvzf nginx-${NGINX_VERSION}.tar.gz
unzip v${NPS_VERSION}-beta.zip

cd ngx_pagespeed-${NPS_VERSION}-beta
psol_url=https://dl.google.com/dl/page-speed/psol/${NPS_VERSION}.tar.gz
[ -e scripts/format_binary_url.sh ] && psol_url=$(scripts/format_binary_url.sh PSOL_BINARY_URL)
wget ${psol_url}
tar -xzvf $(basename ${psol_url})
cd ..

cd ~
sudo yum install openssl-devel -y
cd nginx-${NGINX_VERSION}
./configure --add-module=$HOME/ngx_pagespeed-${NPS_VERSION}-beta --user=nobody --group=nobody --pid-path=/var/run/nginx.pid ${PS_NGX_EXTRA_FLAGS} --with-http_ssl_module --with-http_v2_module
sudo make
sudo make install

sudo ln -s /usr/local/nginx/conf/ /etc/nginx
sudo ln -s /usr/local/nginx/sbin/nginx /usr/sbin/nginx
```

接着创建 nginx 执行脚本，

```
vim /etc/init.d/nginx
```

复制内容

```
#!/bin/sh
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig:   - 85 15
# description:  NGINX is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: nginx
# config:      /etc/nginx/nginx.conf
# config:      /etc/sysconfig/nginx
# pidfile:     /var/run/nginx.pid

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0

nginx="/usr/sbin/nginx"
prog=$(basename $nginx)

NGINX_CONF_FILE="/etc/nginx/nginx.conf"

[ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx

lockfile=/var/lock/subsys/nginx

make_dirs() {
   # make required directories
   user=`$nginx -V 2>&1 | grep "configure arguments:.*--user=" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
   if [ -n "$user" ]; then
      if [ -z "`grep $user /etc/passwd`" ]; then
         useradd -M -s /bin/nologin $user
      fi
      options=`$nginx -V 2>&1 | grep 'configure arguments:'`
      for opt in $options; do
          if [ `echo $opt | grep '.*-temp-path'` ]; then
              value=`echo $opt | cut -d "=" -f 2`
              if [ ! -d "$value" ]; then
                  # echo "creating" $value
                  mkdir -p $value && chown -R $user $value
              fi
          fi
       done
    fi
}

start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    make_dirs
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}

stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}

restart() {
    configtest || return $?
    stop
    sleep 1
    start
}

reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
    RETVAL=$?
    echo
}

force_reload() {
    restart
}

configtest() {
  $nginx -t -c $NGINX_CONF_FILE
}

rh_status() {
    status $prog
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}

case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
            ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac
```

```
sudo chmod +x /etc/init.d/nginx

sudo service nginx start
sudo systemctl enable nginx

sudo mkdir -p /var/ngx_pagespeed_cache
sudo chown -R nobody:nobody /var/ngx_pagespeed_cache
```
#### 安装 Let's Encrypt 证书

```
cd ~
sudo yum install yum-utils -y
yum-config-manager --enable rhui-REGION-rhel-server-extras rhui-REGION-rhel-server-optional
sudo yum install certbot -y
certbot certonly --standalone -d doge.studio -d www.doge.studio
```
如果失败，需要开放 443 端口

```
systemctl stop firewalld
systemctl mask firewalld
sudo yum install iptables-services -y
systemctl enable iptables
systemctl start iptables
```
执行命令打开 iptables，

```
vim /etc/sysconfig/iptables
```

复制以下内容到 `--dport 22` 下面一行

```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
```
保存，重启 iptables

```
service iptables restart
```

再次执行

```
certbot certonly --standalone -d doge.studio -d www.doge.studio
```
之后就是 nginx 配置。