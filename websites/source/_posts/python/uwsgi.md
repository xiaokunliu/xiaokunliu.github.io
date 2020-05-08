---
title: python之uwsgi配置
category: Python
date: 2019-12-10 23:21:11
tags: Python
---

<!-- more -->
> uwsg基础配置
```text
[uwsgi]
# 当前文件所处的文件夹
chdir=%d
project_name=%c
user=@(exec://whoami)
virtualenv=/home/keithl/workdir/python/pyenv/%(project_name)
# load a WSGI module
wsgi-file=wsgi_admin_handler.py
master=true

# set the socket listen queue size
listen=100
# 本机内网ip
lanip=@(exec://hostname -I|awk '{print $NF}')
# 本机外网ip
comboip=@(exec://hostname -I|awk '{print $1}')

# 设置端口
http=%(lanip):9000

# 设置进程数
processes=4

print=%(http)
pidfile=%(chdir)logs/uwsgi.pid
# supervisor transfrom process into daemon
vacuum=true
auto-procname=true
procname-prefix-spaced=%(user)_%(project_name)
# supervisorctl restart
touch-reload=/home/keithl/workdir/python/pyenv/cgi_reload_file/%(project_name)
#touch-reload=wsgi_handler.py
enable-threads=true
lazy-apps=true


# 日志
show-config=true
log-maxsize=0
logfile-chmod=644
logfile-chown=true


# daemonize=%(chdir)logs/uwsgi_default.log
logto=%(chdir)logs/uwsgi_default.log
req-logger=file:%(chdir)logs/uwsgi_req.log

exec-asap=mkdir -p %(chdir)logs
exec-asap=touch %(chdir)logs/uwsgi_default.log
exec-asap=chmod 644 %(chdir)logs/uwsgi_default.log
exec-asap=touch %(chdir)logs/uwsgi_req.log
exec-asap=chmod 644 %(chdir)logs/uwsgi_req.log

# 日志切割
cron = -30 -1 -1 -1 -1 find %(chdir)logs \( -name "uwsgi_req.log" -o -name "uwsgi_default.log" -o -name "default.log" -o -name "epay.log" -o -name "leazy-error-log" -o -name "leazy-info-log" -o -name "leazy-debug-log" -o -name "error.log" \) -size +100M -exec cp {} {}.`date +"%%Y-%%m-%%d_%%H:%%M:%%S"`.rotate.log \; -exec truncate {} --size 0 \;

# 压缩 每天凌晨2:56将logs目录下后缀是rotate.log的，且最后修改时间在1天以上的日志，用gzip压缩
cron = 56 2 -1 -1 -1 find %(chdir)logs -mtime +1 -name "*.rotate.log" -exec gzip {} \;

# 删除旧的日志 每天凌晨 3:56将太旧的备份文件删除，保留一个月吧,后缀是rotate.log.gz
cron = 56 3 -1 -1 -1 find %(chdir)logs -mtime +31 -name "*.rotate.log.gz" -exec rm {} \;
```
> 使用supervisord配置启动
```bash
[program:cclive]
directory=/home/cc/webcc/cclive
command=/home/cc/.virtualenvs/webdd/bin/uwsgi webdd_uwsgi.ini
user=cc
numprocs=1
stdout_logfile=/home/cc/webcc/cclive/logs/uwsgi_console.log
redirect_stderr=true
autostart=true
autorestart=true
startsecs=10
stopwaitsecs=600
killasgroup=true
priority=998
stopsignal=QUIT

[program:cclive_celery]
directory=/home/cc/webcc/cclive
command=/home/cc/.virtualenvs/webdd/bin/celery --app=webdd.celery:app worker --loglevel=INFO -Ofair --time-limit=120
user=cc
numprocs=1
stdout_logfile=/home/cc/webcc/cclive/logs/celery_console.log
redirect_stderr=true
autostart=true
autorestart=true
startsecs=10
stopwaitsecs=600
killasgroup=true
priority=998
```
> 使用celery配置django
```python
# celery.py
from __future__ import absolute_import

import os

from celery import Celery

from django.conf import settings

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'webdd.settings')

app = Celery('webdd')

# Using a string here means the worker will not have to
# pickle the object when using Windows.
app.config_from_object('django.conf:settings')
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)
```



