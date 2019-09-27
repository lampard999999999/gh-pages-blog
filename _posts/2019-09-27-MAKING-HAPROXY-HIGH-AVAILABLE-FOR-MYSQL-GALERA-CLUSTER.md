---
layout : post
title : MYSQL- MAKING HAPROXY HIGH AVAILABLE FOR MYSQL GALERA CLUSTER
category: MYSQL
tag: 
- Mysql
- Database
- Mariadb
---


After properly installing and testing a  [Galera Cluster](http://galeracluster.com/ "Galera Cluster")  we see that the set-up is not finished yet. It needs something in front of the Galera Cluster that balances the load over all nodes.  
So we install a load balancer in front of the Galera Cluster. Typically nowadays  [HAProxy](http://www.haproxy.org/ "HAProxy: The  again. The fail-over should happen automatically. For this we need a Virtual IP which should automatically fail-over.Reliable, High Performance TCP/HTTP Load Balancer")  is chosen for this purpose. But then we find, that the whole Galera Cluster is still not high available in case the load balancer fails or dies. So we need a second load balancer for high availability.  
But how should we properly fail-over when the HAProxy load balancer dies? For this purpose we put a Virtual IP (VIP) in front of the HAProxy load balancer pair. The Virtual IP is controlled and fail-overed with  [Keepalived](http://www.keepalived.org/ "VIP: Keepalived for Linux").

![haproxy_ha.png](https://www.fromdual.com/sites/default/files/haproxy_ha.png "HAProxy high available")

## INSTALLATION OF HAPROXY AND KEEPALIVED

First some preparations: For installing  `socat`  we need the repoforge repository:
```sh
shell> cd /download
shell> wget http://pkgs.repoforge.org/rpmforge-release/rpmforge-release-0.5.3-1.el6.rf.x86_64.rpm
shell> yum localinstall rpmforge-release-0.5.3-1.el6.rf.x86_64.rpm 
shell> yum update
shell> yum install socat
```
  

Then we can start installing HAProxy and Keepalived:
```sh
shell> yum install haproxy keepalived

shell> chkconfig haproxy on
shell> chkconfig keepalived on
```
  

We can check the installed HAProxy and Keepalived versions as follows:
```sh
shell> haproxy -v
HA-Proxy version 1.5.2 2014/07/12

shell> keepalived --version
Keepalived v1.2.13 (10/15,2014)
```
  

## CONFIGURATION OF HAPROXY

More details you can find in the  [HAProxy documentation](http://www.haproxy.org/download/1.5/doc/configuration.txt "HAProxy documentation").
```sh
shell> cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
shell> cat << _EOF >/etc/haproxy/haproxy.cfg
#
# /etc/haproxy/haproxy.cfg
#

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
  # to have these messages end up in /var/log/haproxy.log you will
  # need to:
  #
  # 1) configure syslog to accept network log events.  This is done
  #    by adding the '-r' option to the SYSLOGD_OPTIONS in
  #    /etc/sysconfig/syslog
  #
  # 2) configure local2 events to go to the /var/log/haproxy.log
  #   file. A line like the following can be added to
  #   /etc/sysconfig/syslog
  #
  #    local2.*                       /var/log/haproxy.log
  #
  log         127.0.0.1 local2

  chroot      /var/lib/haproxy
  pidfile     /var/run/haproxy.pid
  maxconn     1020   # See also: ulimit -n
  user        haproxy
  group       haproxy
  daemon

  # turn on stats unix socket
  stats socket /var/lib/haproxy/stats.sock mode 600 level admin
  stats timeout 2m

#---------------------------------------------------------------------
# common defaults that all the 'frontend' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
  mode    tcp
  log     global
  option  dontlognull
  option  redispatch
  retries                   3
  timeout queue             45s
  timeout connect           5s
  timeout client            1m
  timeout server            1m
  timeout check             10s
  maxconn                   1020

#---------------------------------------------------------------------
# HAProxy statistics backend
#---------------------------------------------------------------------
listen haproxy-monitoring *:80
  mode    http
  stats   enable
  stats   show-legends
  stats   refresh           5s
  stats   uri               /
  stats   realm             Haproxy\ Statistics
  stats   auth              monitor:AdMiN123
  stats   admin             if TRUE
  
frontend haproxy**1**   # change on 2nd HAProxy
  bind    *:3306
  default_backend           galera-cluster

backend galera-cluster
  balance roundrobin
  server  nodeA             192.168.1.61:3306 maxconn 151 check
  server  nodeB             192.168.1.62:3306 maxconn 151 check
  server  nodeC             192.168.1.63:3306 maxconn 151 check
_EOF
```
  

## STARTING AND TESTING HAPROXY

The HAProxy can be started as follows:
```sh
shell> service haproxy start
```
  

and then be checked either over the socket:
```sh
shell> socat /var/lib/haproxy/stats.sock readline
  prompt
  > show info
  > show stat
  > help
```
  

or over your favourite web browser entering the username and password (**monitor:AdMiN123**) specified in the configuration file above:

![haproxy_ui.png](https://www.fromdual.com/sites/default/files/haproxy_ui.png "HAProxy Monitoring Interface")

To check the application over the load balancer we can run the following command:
```sh
shell> mysql --user=app --password=secret --host=192.168.1.38 --port=3306 --exec="SELECT @@wsrep_node_name;"
+-------------------+
| @@wsrep_node_name |
+-------------------+
| Node C            |
+-------------------+
shell>  mysql --user=app --password=secret --host=192.168.1.38 --port=3306 --exec="SELECT @@wsrep_node_name;"
+-------------------+
| @@wsrep_node_name |
+-------------------+
| Node A            |
+-------------------+
shell>  mysql --user=app --password=secret --host=192.168.1.38 --port=3306 --exec="SELECT @@wsrep_node_name;"
+-------------------+
| @@wsrep_node_name |
+-------------------+
| Node B            |
+-------------------+
```
  

## CONFIGURATION A VIRTUAL IP (VIP) WITH KEEPALIVED

Now we have 2 HAProxy load balancers. But what happens if one of them fails. Then we do not want to reconfigure our application to work properly again. The fail-over should happen automatically. For this we need a Virtual IP which should automatically fail-over.

More details you can find in the  [Keepalived documentation](http://www.keepalived.org/documentation.html)  and the  [keepalived user guide](http://www.keepalived.org/pdf/UserGuide.pdf "Keepalived user guide").
```sh
shell> cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
cat << _EOF >/etc/keepalived/keepalived.conf
#
# /etc/keepalived/keepalived.conf
#

global_defs {

  notification_email {
    remote-dba@fromdual.com
    root@localhost
  }

  # Change email from on lb2:
  notification_email_from **lb1@haproxy1**
  router_id HAPROXY
}

vrrp_script chk_haproxy {
  script   "killall -0 haproxy"
  interval 2
  weight   2
}

vrrp_instance GALERA_VIP {

  interface         eth1
  virtual_router_id 42
  # Higher priority on other node
  priority          101   # **102**
  advert_int        1
  # notify "/usr/local/bin/VRRP-notification.sh"

  virtual_ipaddress {
    192.168.1.99/32 dev eth1
  }

  track_script {
    chk_haproxy
  }

  authentication {
    auth_type PASS
    auth_pass secret
  }
}
_EOF
```
  

## STARTING AND TESTING KEEPALIVED

To test the keepalived we can run the following command:
```sh
shell> keepalived -f /etc/keepalived/keepalived.conf --dont-fork --log-console --log-detail
^C
```
  

To finally start it the following command will serve:
```sh
shell> service keepalived start
```
  

To check the Virtual IP the following command will help:
```sh
shell> ip addr show eth1
```
  

And then we can check our application over the VIP:
```sh
shell> mysql --user=app --password=secret --host=192.168.1.99 --port=3306 --exec="SELECT @@wsrep_node_n
```
