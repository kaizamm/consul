---
title: consul+docker+register+ocnsul-template实现服务自动发现+主动注册+auto_scale.md
date: 2017.11.17
---
### 环境准备
client agent:node1 192.168.0.11
server agent:node2 192.168.0.12
#### consul client
node1 192.168.0.11
+ /etc/default/consul :
```
 CONSUL_FLAGS="-data-dir=/opt/consul/data -config-dir=/opt/consul/conf -pid-file=/opt/consul/run/consul.pid -client=0.0.0.0 -bind=192.168.0.11 -node=consul-node-1 -retry-join '192.168.0.12'"
```
+  /usr/lib/systemd/system/consul.service
```
[Unit]
Description=Consul service discovery agent
Requires=network-online.target
After=network-online.target
[Service]
User=consul
Group=consul
EnvironmentFile=-/etc/default/consul
Environment=GOMAXPROCS=2
Restart=on-failure
ExecStartPre=[ -f "/opt/consul/run/consul.pid" ] && /usr/bin/rm -f /opt/consul/run/consul.pid
ExecStart=/usr/bin/consul agent $CONSUL_FLAGS
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGTERM
TimeoutStopSec=5
[Install]
WantedBy=multi-user.target
```
+ /usr/bin/consul
+ /opt/consul/{data,run,conf}
+ useradd consul

####  consul server
192.168.0.12 node2
+ /etc/default/consul
```
CONSUL_FLAGS="-server -ui -bootstrap-expect=1 -data-dir=/opt/consul/data -config-dir=/opt/consul/conf -pid-file=/opt/consul/run/consul.pid -client=0.0.0.0 -bind=192.168.0.12 -node=consul-server"
```
+ /usr/lib/systemd/system/consul.service
```
[Unit]
Description=Consul service discovery agent
Requires=network-online.target
After=network-online.target
[Service]
User=consul
Group=consul
EnvironmentFile=-/etc/default/consul
Environment=GOMAXPROCS=2
Restart=on-failure
ExecStartPre=[ -f "/opt/consul/run/consul.pid" ] && /usr/bin/rm -f /opt/consul/run/consul.pid
ExecStart=/usr/bin/consul agent $CONSUL_FLAGS
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGTERM
TimeoutStopSec=5
[Install]
WantedBy=multi-user.target
```
+ /usr/bin/consul
+ /opt/consul/{data,run,conf}
+ useradd consul

#### docker环境
搭建简单web。https://hub.docker.com/r/crccheck/hello-world/
```
$ docker run -d -P crccheck/hello-world
$ curl localhost:$port  #port为自动暴露的宿主机端口
Hello World
```

### registrator
client端装registator，结合docker API 实现服务自动发现、注册及注销，consul注册地址请注意填写agent地址，请勿直接注册进catalog。https://github.com/kaizamm/registrator
```
$ docker run -d \
    --name=registrator \
    --net=host \
    --volume=/var/run/docker.sock:/tmp/docker.sock \
    gliderlabs/registrator:latest \
      consul://192.168.0.11:8500
```

### 几个常用API
#### deregister
```
{
  "Datacenter": "dc1",
  "Node": "foobar"
}
{
  "Datacenter": "dc1",
  "Node": "foobar",
  "CheckID": "service:redis1"
}
{
  "Datacenter": "dc1",
  "Node": "foobar",
  "ServiceID": "redis1"
}
```
如：
```
#删除这个该Node的及上面的所有service/heath check/
curl -X PUT -d '{"Datacenter":"dc1","Node":"consul-node-1"}' 172.30.33.4:8500/v1/catalog/deregister
```

### consul template

+ download地址: https://releases.hashicorp.com/consul-template/0.19.4/
+ github地址：https://github.com/kaizamm/consul-template

```
consul-template -template "in.tpl:out.txt"   #根据in.tpl的模版文件生成out.txt，consult-template以daemon方式运行
consul-template -template "in.tpl:out.txt" -once   #非daemon方式
 consul-template -template "in.tpl:out.conf:service nginx -s reload"  #第三个参数为要执行的动作
```
根据in.tpl生成out.conf
```
#in.tpl
upstream hello-world {
  {{ range service "hello-world"}}
  server {{ .Address }}:{{ .Port }}; {{ end }}
}

server {
  listen 8080;
  location /{
    proxy_pass http://hello-world;
  }
}
```
运行以上consul-template，查看out.conf
```
upstream hello-world {

  server 192.168.0.12:7777;
  server 192.168.0.12:8888;
}

server {
  listen 8080;
  location /{
    proxy_pass http://hello-world;
  }
}
```
到此为止，结合consul agent + consul template + consul registrator 就能实现服务发现与自动扩容

### 附supervisor(centos 6可用该方法)
对于redhat7可以采用systemctl对服务控制，对于centos6,则可以考虑用附supervisor对其进程进行管理
http://supervisord.org/installing.html
#### pip install
```
wget https://bootstrap.pypa.io/get-pip.py --no-check-cetificate
python get-pip.py install
```

如：
```
#cat /etc/supervisor/django.conf
[program:django]
directory = /usr/local/cmdb ;
command = python manage.py runserver 0.0.0.0:7990 ;
stdout_logfile = /opt/logs/supervisor/django_out.log ;
stderr_logfile = /opt/logs/supervisor/django_err.log ;
autostart = true ;
startsecs = 5 ;
autorestart = true ;
startretries = 3 ;
user = root ;
```
