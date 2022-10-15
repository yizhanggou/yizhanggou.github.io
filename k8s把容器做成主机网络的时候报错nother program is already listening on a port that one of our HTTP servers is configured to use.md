---
title: "k8s把容器做成主机网络的时候报错nother program is already listening on a port that one of our HTTP servers is configured to use"
author: 一张狗
lastmod: 2019-07-06 07:35:10
date: 2018-11-01 15:11:48
tags: []
---


因为某些原因所以项目组需要把整个系统的所有容器都做成主机网络：

1. 防拷贝需要使用主机网络的信息，所以这些容器需要挂载主机网络；
2. 1.6以下的容器如果使用主机网络那就不能通过kube-dns来做路由，所以这几个容器的下游也需要主机网络；

在apply成主机网络的时候，除了第一个能成功，其他的都起不来，提示：

another program is already listening on a port that one of our HTTP servers is configured to use

尝试了网上的解决方案

<div class="codecolorer-container text default" style="overflow:auto;white-space:nowrap;width:435px;"><div class="text codecolorer">unlink /<span class="hljs-keyword">var</span>/run/supervisor.sock</div></div>但是不行；

然后发现配置文件有这么个东西
```
; Sample supervisor config file. 
[unix_http_server] 
file=/tmp/supervisor.sock ; (the path to the socket file) ;
chmod=0700 ; socket file mode (default 0700) ;
chown=nobody:nogroup ; socket file uid:gid owner ;
username=user ; (default is no username (open server)) ;
password=123 ; (default is no password (open server)) 
[inet_http_server] ; inet (TCP) server disabled by default
port=127.0.0.1:9001 ; (ip_address:port specifier, *:port for all iface)
username=user ; (default is no username (open server))<
password=123 ; (default is no password (open server))
[supervisord] logfile=/tmp/supervisord.log ; (main log file;default $CWD/supervisord.log) 
logfile_maxbytes=50MB ; (max main logfile bytes b4 rotation;default 50MB) 
logfile_backups=10 ; (num of main logfile rotation backups;default 10) 
loglevel=debug ; (log level;default info; others: debug,warn,trace) 
pidfile=/tmp/supervisord.pid ; (supervisord pidfile;default supervisord.pid) 
nodaemon=true ; (start in foreground if true;default false) 
minfds=1024 ; (min. avail startup file descriptors;default 1024) 
minprocs=200 ; (min. avail process descriptors;default 200) ;
umask=022 ; (process file creation umask;default 022) ;
user=chrism ; (default is current user, required if root) ;
identifier=supervisor ; (supervisord identifier, default is 'supervisor') ;
directory=/tmp ; (default is not to cd during start) ;
nocleanup=true ; (don't clean up tempfiles at start;default false) ;
childlogdir=/tmp ; ('AUTO' child log dir, default $TEMP) ;
environment=KEY=value ; (key value pairs to add to environment) ;
strip_ansi=false ; (strip ansi escape codes in logs; def. false) 
; the below section must remain in the config file for RPC ; (supervisorctl/web interface) to work, additional interfaces may be ; added by defining them in separate rpcinterface: sections 
[rpcinterface:supervisor] supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface [supervisorctl] serverurl=unix:///tmp/supervisor.sock ; use a unix:// URL for a unix socket serverurl=http://127.0.0.1: ; 
use an http:// url to specify an inet socket username=chris ; should be same as http_username if set 
password=123 ; should be same as http_password if set ;
prompt=mysupervisor ; cmd line prompt (default "supervisor") ;
history_file=~/.sc_history ; use readline history if available ; The below sample program section shows all possible program subsection values, ; create one or more 'real' program: sections to be able to control them under ; supervisor.
```
就是说，supervisord会绑定9001监听客户端的通信，所以如果都挂主机网络的话会冲突；

果断删除掉，就可以了。


