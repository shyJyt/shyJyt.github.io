---
title: 非容器化部署 Vue + Django 
date: 2024-07-02 10:54:28
updated: 2024-07-02 10:54:28
categories:
- Linux
---

### 前端

build 成 dist 上传到服务器即可

### 后端

1、项目文件上传，创建虚拟环境，安装对应包

注意 uwsgi 需要用 conda install ，如果还有问题，采用以下命令安装。

```shell
anaconda search -t conda uwsgi

anaconda show conda-forge/uwsgi

conda install --channel https://conda.anaconda.org/conda-forge uwsgi 
```

2、uwsgi：配置 uwsgi.ini

```shell
[uwsgi]
socket = 127.0.0.1:8000
chdir = /home/buaa/LinkedOut_Backend
wsgi-file = /home/buaa/LinkedOut_Backend/LinkedOut_Backend/wsgi.py
master = true
enable-threads = true
processes = 8
buffer-size = 65536
vacuum = true
daemonize = /home/buaa/LinkedOut_Deployment/backend/uwsgi/uwsgi.log
virtualenv = /home/buaa/miniconda3/envs/LinkedOut
uwsgi_read_timeout = 600
threads = 2
chmod-socket = 664
pidfile = /home/buaa/LinkedOut_Deployment/backend/uwsgi/uwsgi.pid
```

3、nginx：配置 web.conf

注意配置请求体的大小限制 `client_body_buffer_size 10m; client_max_body_size 20m;` 以避免 403 （请求体过大）错误，默认是 1M。

```json
server {
        listen 80;
        server_name  ip;

        location / {
                root /home/buaa/dist;
                index index.html index.htm;
                try_files $uri $uri/ /index.html;
        }

        location /api {
                include /etc/nginx/uwsgi_params;
                uwsgi_pass 127.0.0.1:8000;
                client_body_buffer_size 10m;
                client_max_body_size 20m;
        }

        location /ws {
                proxy_pass http://127.0.0.1:8001;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_read_timeout 360s;
        }

        error_page 497 https://$host$uri?$args;
}
```

### 部署脚本（半自动

因为暂未找到在 bash 脚本中启动 conda 虚拟环境的方法，故仍需手动执行命令。

```shell
cd /home/buaa/LinkedOut_Backend
echo $(pwd)

# conda
conda activate LinkedOut

# stop previous service
sudo pkill -f uwsgi -9
sudo pkill -f dephne -9

# get latest version
git checkout dev
git pull

# migrate
python manage.py makemigrations
python manage.py migrate

# start uwsgi
cd /home/buaa/LinkedOut_Deployment/backend
echo $(pwd)
uwsgi --ini ./uwsgi/uwsgi.ini
```

daphne 默认占用前台，要么 tmux 新开窗口，要么采用 supervisor 后台跑。

```shell
# activate conda venv
conda activate LinkedOut

sudo pkill -f dephne -9

# start daphne
cd /home/buaa/LinkedOut_Backend
daphne LinkedOut_Backend.asgi:application -p 8001
```

最后重启 nginx 即可。

```shell
sudo nginx -s reload
```

### 附：常用命令

**nginx**

```shell
# 查看状态
sudo systemctl status nginx
# 启动
sudo systemctl start nginx
# 重启
sudo nginx -s reload
# 停止
sudo nginx -s stop
```

**uwsgi**

```shell
# 注意不要用 sudo，不然有日志里面可以看到有 warning，但是这样就必须给 ubuntu 足够权限
# 可能是上传时用的 root 用户，所以文件所有者就给了 root，此时 ubuntu 再操作就没权限了
sudo chmod -R 777 ./*
uwsgi ini --uwsgi.ini

uwsgi --stop uwsgi.pid
sudo pkill -f uwsgi -9

ps aux | grep uwsgi
```

**daphne**

```shell
sudo pkill -f dephne -9

cd /home/buaa/LinkedOut_Backend
daphne LinkedOut_Backend.asgi:application -p 8001
```

**mysql**

可能默认用户不是 root，去配置文件里面看看。而且默认配置 `bind 127.0.0.1`，注释掉才可允许远程连接。

```shell
mysql -u root -p
```

**redis**

同上 mysql

```shell
# 查看状态
sudo systemctl status redis-server
# 重启
sudo systemctl restart redis-server
# 登录
redis-cli
auth 123456
```
