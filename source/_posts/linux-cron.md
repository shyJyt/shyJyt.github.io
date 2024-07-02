---
title: 借助 cron service 实现定时任务
date: 2024-07-02 10:59:32
updated: 2024-07-02 10:59:32
categories:
- Linux
---

由于云平台的服务器经常会断网，得写个脚本定时登录。

### 登录脚本

```shell
#!/bin/bash

# cd to the position of shell file 
# 保证在不同路径下执行该脚本（因为使用了相对路径）的一致性
cd `dirname $0`

# cron 执行时使用的环境变量不包含 buaalogin，需绝对路径引用
# 由于已经切换到脚本所在目录，故相对路径生成的日志文件和脚本同目录
/usr/local/bin/buaalogin >> ./auto_login.log
```



### crontab 配置定时任务

```shell
# 修改任务列表
crontab -e

# 在文件末尾加上新任务
# 每 30 分钟执行一次 buaalogin
# 分 时 日 月 周几
# /a 表示每过 a 个单位时间执行
*/30 * * * * /home/buaa/LinkedOut_Deployment/auto_login/auto_login.sh

# 重启 cron 服务
sudo service cron restart

# 列出...
crontab -l
```

默认是不打印 cron 日志的，修改 `/etc/rsyslog.d/50-default.conf` ，将 `cron.*` 的注释取消，重启 rsyslog

```shell
sudo service rsyslog restart
```



### 安装 postfix 接收 cron 的报错信息

不安装邮件传输代理 (MTA) 会出现如下 info:

![image-20240630110236173](image-20240630110236173.png)

安装 postfix 后可看到具体报错，如由于路径 / 环境变量 导致的找不到命令 / 文件等等，在 `/var/spool/mail/username` 中。



