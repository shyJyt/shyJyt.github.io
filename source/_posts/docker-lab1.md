---
title: docker lab1
date: 2024-04-16 12:06:14
updated: 2024-04-16 12:06:14
categories:
- Docker
---

### 1. 数据持久化

#### 1.1 bind mount

##### 1.1.1 创建

在当前目录下分别创建 log, data, conf 目录，分别持久化 mysql 的日志、数据以及配置文件，通过 `-v` 选项建立与容器内对应目录的映射。

```shell
docker run -it -p 8888:3306 \
-v $PWD/log:/var/log/mysql \
-v $PWD/data:/var/lib/mysql \
-v $PWD/conf:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql
```

##### 1.1.2 写入数据

```shell
docker exec -it b10d0778405d mysql -u root -p
```

使用 mysql 命令行客户端，创建 test 数据库。

![image-20240416104319917](image-20240416104319917.png)

##### 1.1.3 销毁

```shell
docker stop <container-id>
docker rm <container-id>
```

##### 1.1.4 重新创建

同 1.1.1 

##### 1.1.5 重新读取

上面几步的截图如下，可以看到基于干净的 mysql 镜像重新创建的容器仍然可以查询到先前创建的 test 数据库。

![image-20240416105003258](image-20240416105003258.png)

#### 1.2 volumes

##### 1.2.1 创建

```shell
docker run -d \
-v mysql_data:/var/lib/mysql \
-v mysql_log:/var/log/mysql \
-v mysql_conf:/etc/mysql/conf.d \
-e MYSQL_ROOT_PASSWORD=root \
-d --name mysql01 mysql
```

由 docker 管理程序自动生成三个命名 volume .

![image-20240416114259192](image-20240416114259192.png)

##### 1.2.2 写入数据

```shell
docker exec -it mysql01 mysql -u root -p
```

同样地，使用 mysql 命令行客户端，创建 test 数据库。

![image-20240416104319917](image-20240416104319917.png)

##### 1.2.3 销毁

```shell
docker stop mysql01
docker rm mysql01
```

##### 1.2.4 重新创建

同 1.2.1 

##### 1.2.5 重新读取

上面几步的截图如下，可以看到基于干净的 mysql 镜像重新创建的容器仍然可以查询到先前创建的 test 数据库。

![image-20240416114738123](image-20240416114738123.png)

![image-20240416114754027](image-20240416114754027.png)

#### 1.3 方案对比

- **性能：**bind mount 方式，docker 容器直接访问 host 的目录或文件，性能更佳。
- **权限：**bind mount 方式，docker 容器对于该 host 目录可能会引入权限问题。 如果容器仅需要只读访问权限，最好是显式设定只读方式。
- **移植性：**
  - volume 方式，如果 host 中目录为空，docker 先将容器中的对应目录复制到 host 下， 然后再进行挂载操作。
  - bind mount方式，挂载之前没有复制操作，容器要依赖 host 主机的一个绝对路径, 使得容器的移植性变差。

### 2. 构建 Nginx 的 Ubuntu 镜像

#### 2.1 docker commit 

##### 2.1.1 更新 apt  并换源

这里使用的是阿里云的镜像站，由于使用的是干净的 Ubuntu22 镜像，无法通过常用的 vim, nano 等命令行工具创建或编辑文件，故先在宿主机创建该源文件，借助 docker cp 在宿主机和容器间传递文件。

```shell
docker cp ./sources.list <container-id>:/etc/apt/
```

`sources.list` 内容如下，注释掉 `deb-src` 部分以提升 apt update 速度。

```
deb http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
# deb-src http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
# deb-src http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
# deb-src http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse

# deb http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
# deb-src http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse

deb http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
# deb-src http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
```

更新 apt

```shell
apt update
```

##### 2.1.2 安装 nginx

安装 nginx

```shell
apt install -y nginx
```

##### 2.1.3 修改 nginx 主页文件

nginx 的默认配置文件会将 `/var/www/html/index.nginx-debian.html` 作为主页，修改其内容包含学号。

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>==========================</h1>
<h1>My Student Number: 21371337</h1>
<h1>==========================</h1>
</body>
</html>
```

同样使用 docker cp 传进容器内

```shell
docker cp ./index.nginx-debian.html <container-id>:/var/www/html/index.nginx-debian.html
```

##### 2.1.4 构建新镜像

注意修改启动命令为 nginx -g daemon off ，防止容器后台模式启动后立刻退出。

```shell
docker commit -c 'CMD ["nginx", "-g", "daemon off;"]' <container-id> ubuntu-nginx:dockercommit
```

##### 2.1.5 本地测试

![image-20240416083934662](image-20240416083934662.png)

##### 2.1.6 登录并提交到镜像库

```shell
docker login --username=21371337 scs.buaa.edu.cn:8081

docker image tag ubuntu-nginx:dockercommit scs.buaa.edu.cn:8081/personal-21371337/ubuntu-nginx:dockercommit

docker image push scs.buaa.edu.cn:8081/personal-21371337/ubuntu-nginx:dockercommit
```

#### 2.2 Dockerfile

##### 2.2.1 Dockerfile

```dockerfile
FROM ubuntu

COPY sources.list /etc/apt/sources.list

RUN apt update

RUN apt install -y nginx

COPY index.nginx-debian.html /var/www/html/index.nginx-debian.html

CMD ["nginx", "-g", "daemon off;"]
```

##### 2.2.2 构建镜像

![image-20240416091403047](image-20240416091403047.png)

##### 2.2.3 本地测试

![image-20240416090805808](image-20240416090805808.png)

##### 2.2.4 上传

和 2.1.6 同理。