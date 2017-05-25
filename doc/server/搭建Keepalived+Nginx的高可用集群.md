====== 搭建Keepalived+Nginx的高可用集群 ======

环境: 
  * 虚拟机软件:VirtualBox
  * 宿主机系统:Windows 10
  * 虚拟机系统：CentOS-6.8-i386（CentOS-6.8-i386-bin-DVD1.iso）
  * 虚拟机内存：1G
  * nginx IP:192.168.3.26、192.168.3.27
  * API服务IP：192.168.3.28、192.168.3.29
  * 虚拟IP：192.168.3.30
源码文件:
  * keepalived-1.2.17.tar.gz(下载：{{ :keepalived-1.2.17.tar.gz |}})
  * nginx-1.9.15.tar.gz（下载：{{ :nginx-1.9.15.tar.gz |}}）
成品配置（下载：{{ :keepalived.zip |}}）：
  * keepalived.conf 
  * nginx_pid.sh

==== 安装Linux ====

  * 其他发行版本也行，不区分32、64位，本例中为节省系统资源选用32位
  * 无需桌面程序，SSH接入
  * 须root账户或sudoers
  * 固定IP

==== 安装Nginx ====

//编译环境准备//
  yum -y install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
//解压nginx源码//
  
  tar -xvf nginx-1.9.15.tar.gz
  
//执行编译、安装//
  
  cd nginx-1.9.15
  ./configure --prefix=/opt/nginx --sbin-path=/usr/bin/nginx
  make && make install

//启停nginx//
  
  nginx  ##启动
  nginx -s stop ##停止
  nginx -s reload ##重新加载
  /opt/nginx/conf  ##配置文件
  
//编辑index.html//

  vi /opt/nginx/html/index.html  ##增加标记如：Welcome to nginx!This server is nginx1
  
__**相同操作在192.168.3.27机器上执行**__

==== 安装Keepalived ====

//解压keepalived源码//
  
  tar -xvf keepalived-1.2.17.tar.gz
  mv keepalived-1.2.17 /usr/local/keepalived
  
//执行编译、安装//
  
  cd /usr/local/keepalived
  ./configure --prefix=/usr/local/keepalived
  make && make install
  mkdir /usr/local/keepalived/script/

//配置keepalived MASTER&&BACKUP//

**  注意注释配置，需要根据实际情况修改！**
  
  vi /usr/local/keepalived/etc/keepalived/keepalived.conf
  
  ! Configuration File for keepalived
  
  global_defs {
     notification_email {
       zhangjingtaosleep@163.com
     }
     notification_email_from 610039879@qq.com
     smtp_server smtp.qq.com
     smtp_connect_timeout 30
     router_id LVS_DEVEL
  }
  
  vrrp_script chk_http_port {
      script "/usr/local/keepalived/script/nginx_pid.sh" ####检测nginx状态的脚本路径
      interval 2
      weight -20
  }
  
  vrrp_instance VI_1 {
      state MASTER ##主MASTER副BACKUP
      interface **eth0** ##ifconfig看网卡名称
      virtual_router_id 51
      priority 100 ##主100 副90
      advert_int 1
      mcast_src_ip **192.168.3.26** ##本机IP
      authentication {
          auth_type PASS
          auth_pass 1111
      }
      track_script {
          chk_http_port
      }
      virtual_ipaddress {
          **192.168.3.30/24** ##虚拟IP
      }
  }
  
//配置为系统服务//

  cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/rc.d/init.d/       
  cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/      
  mkdir /etc/keepalived       
  cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/       
  cp /usr/local/keepalived/sbin/keepalived /usr/sbin/  
  
  
//编辑nginx_pid脚本检查nginx状态//
  
  vi /usr/local/keepalived/script/nginx_pid.sh
  
  #!/bin/bash
  if [ "$(ps -ef | grep "nginx: master process"| grep -v grep )" == "" ]
  then
   /usr/bin/nginx ##本例中nginx配置在/usr/bin下
   sleep 5
   if [ "$(ps -ef | grep "nginx: master process"| grep -v grep )" == "" ]
   then
   killall keepalived
   fi
  fi
  
  ##增加执行权限
  chmod 777 /usr/local/keepalived/script/nginx_pid.sh
  
脚本逻辑：检查nginx实例是否存活，如果没有尝试重启，5s后再次检查，如果没有杀掉本机的keepalived（释放VIP占用，BACKUP机会自动抢占VIP，实现IP漂移）

__**相同操作在192.168.3.27机器上执行 注意MASTER和BACKUP的区别**__


//启动服务//

  ##启动服务命令
  keepalived
  ##查看日志
  tail -f /var/log/messages 
  ##正常日志如下：
  Mar  5 01:18:33 nginx1 Keepalived_vrrp[29143]: VRRP_Instance(VI_1) Entering MASTER STATE
  
分别启动两个机器的keepalived后，登录master检查

  * ip -a

{{http://wiki.onlysleep.net/dokuwiki/lib/exe/fetch.php?cache=&media=qq截图20170305145630.png|}}

  * nginx在nginx_pid脚本中被启动

{{http://wiki.onlysleep.net/dokuwiki/lib/exe/fetch.php?cache=&media=qq截图20170305151444.png|}}


//验证服务//

  * 服务正常时截图/MASTER

{{http://wiki.onlysleep.net/dokuwiki/lib/exe/fetch.php?cache=&media=qq截图20170305150431.png|}}

  * 主服务异常时截图

{{http://wiki.onlysleep.net/dokuwiki/lib/exe/fetch.php?cache=&media=qq截图20170305150310.png|}}  
  
  
//最终实现//
  - A,B服务正常时，A nginx挂掉，尝试重启，成功继续A服务失败kill keepalived，VIP自动跳转到B服务
  - A服务恢复后，抢回B服务上的VIP，继续提供服务



==== 安装APIServer ====

//环境准备//
   - 安装jdk(vi /etc/profile)
   - 启动API程序(java -jar api_pack-1.0.0-SNAPSHOT.jar)

//配置nginx负载均衡//

  vi /opt/nginx/conf/nginx.conf
  http {
    include       mime.types;
    default_type  application/octet-stream;
    
    access_log  logs/access.log  main;
  
    sendfile        on;
    keepalive_timeout  65;
    upstream api_server{
       server 192.168.3.28:8080;
       server 192.168.3.29:8080;
    }
  
    #gzip  on;
  
    server {
        listen       80;
        server_name  api_server;
  
        #charset koi8-r;
  
        #access_log  logs/host.access.log  main;
  
        location / {
            root   html;
            index  index.html index.htm;
            proxy_pass http://api_server;
            proxy_set_header X_Real-IP $remote_addr;
            client_max_body_size 100m;
        }
        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
  }
  

//重启keepalived生效//

  killall keepalived
  keepalived

//验证//
  http://192.168.3.30/example/1

{{http://wiki.onlysleep.net/dokuwiki/lib/exe/fetch.php?cache=&media=1.png|}}

{{http://wiki.onlysleep.net/dokuwiki/lib/exe/fetch.php?cache=&media=2.png|}}

//补充//

  ##CentOS默认开启了iptables服务，需要开启指定端口支持服务访问
  vi /etc/sysconfig/iptables
  ##增加一行
  -A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 8080 -j ACCEPT
  ##重启服务
  service iptables restart
  
==EOF==
 --- //[[zhangJingtaosleep@163.com|张京涛]] 2017/03/06 22:09//
