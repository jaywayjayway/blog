

title: uwsgi+nginx简洁配置
date: 2016-10-22 19:44:10
tags:
-  nginx

categories: devops
---

网上找了一圈的nginx+uwsgi的配置，各种尝试，各种失败。千呼万唤始出来，终于可以正常协同工作了。在此记录下。

`uwsgi`一定要用`pip`安装版本的，路径为`/usr/bin/uwsgi`，使用`yum`安装版本的可能要进坑

以下是`uwsgi`的配置文件
``` shell 
## uwsgi.ini ###
[uwsgi]
# the base directory (full path)
chdir           = /usr/local/src/alert
module          = run     ##  启动flask应用的启动文件
enable-threads  = true
callable        =  app    ##  启动文件的应用
# master
master          = true
# maximum number of worker processes
processes       = 4
# the socket (use the full path to be safe
socket          = /var/run/api.sock
#socket          = 127.0.0.1:2222
# with appropriate permissions
chmod-socket    = 664      ##socket文件的权限
# clear environment on exit
vacuum          = true
uid             = daemon
gid             = daemon
# Run in the background as a daemon
daemonize       = /var/log/uwsgi/api.log

```
`python`应用的启动文件(我这里是一个`flask`应用)
```shell
from alert import app
if __name__ == "__main__":
    app.run(debug=True, host='0.0.0.0',port=2222)
```


启动姿势
```shell

uwsgi的命令行启动：

/usr/bin/uwsgi --socket 0.0.0.0:2222 --protocol=http --module run  --callable app —threads  4

daemon启动

/usr/bin/uwsgi —ini  uwsgi.ini

nginx的配置
```
`nginx`的配置
```shell
upstream uwsgi {
    server unix:///var/run/api.sock;
}

server {
    listen 5000;
    charset     utf-8;
    access_log /data/logs/weblog/api_access.log;
    error_log /data/logs/weblog/api_error.log;

    location / {
        uwsgi_pass   uwsgi;  # 指向uwsgi 所应用的内部地址,所有请求将转发给uwsgi 处理
        include     uwsgi_params; # the uwsgi_params file you installed
    }
}
```

