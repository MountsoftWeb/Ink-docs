---
title: 07-云服务器安装 Gitlab
date: 2022-07-19
tags:
  - Development-docs
categories:
  - Development-docs
---

## Gitlab

MountsoftWeb 开发在 github 上，如果有自己的内部开发，再放到公网上就不合适，所以要搭建一个内部的仓库，进行项目管理，更重要的是习惯使用 git 工具进行项目管理。

### 服务器安装 Gitlab

云服务器准备安装一个 Gitlab，练练手，记录这个过程。

```
安装 git 依赖
yum -y install policycoreutils openssh-server openssh-clients postfix

下载 gitlab 镜像
wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/gitlab-ce-13.1.2-ce.0.el7.x86_64.rpm

安装 gitlab
rpm -ivh gitlab-ce-13.1.2-ce.0.el7.x86_64.rpm

这里有的人会遇到问题（我自己在安装的时候也遇到这样的问题，摸索之后得到解决）：安装得时候会提示rpm: Header V4 DSA/SHA1 Signature, key ID 442df0f8: NOKEY

解决：在语句后面加上 --force --nodeps
rpm -ivh gitlab-ce-13.1.2-ce.0.el7.x86_64.rpm --force --nodeps

```

安装完成之后，需要修改配置文件

```
vim /etc/gitlab/gitlab.rb
在gitlab.rb文件中找到 external_ur 部位

#修改访问URL
#格式：external_url 'http://ip:端口'
external_url 'http://自己的ip:8081'
#配置时区(可以不用配置)
gitlab_rails['time_zone'] = 'Asia/Shanghai'
在gitlab.rb文件中找到 unicorn['port']=8080 部位 ，将其前面的注释去掉并将8080改成之前设置的8081.
```

防火墙开启，打开端口

```
firewall-cmd --zone=public --add-port=8081/tcp --permanent

防火墙命令
systemctl start firewalld   // 开启
firewall-cmd --reload       // 重启
firewall-cmd --query-port=8081/tcp  //  查看端口是否开启

重置 gitlab 修改配置生效
gitlab-ctl reconfigure
```

gitlab 常用命令

```
gitlab-ctl start      # 启动所有 gitlab 组件；
gitlab-ctl stop       # 停止所有 gitlab 组件；
gitlab-ctl restart    # 重启所有 gitlab 组件；
gitlab-ctl status     # 查看服务状态；
gitlab-ctl reconfigure        # 刷新配置文件；
vim /etc/gitlab/gitlab.rb     # 修改默认的配置文件；
gitlab-rake gitlab:check SANITIZE=true --trace    # 检查gitlab；
gitlab-ctl tail        # 查看日志；
```

### docker 安装 gitlab

可以使用 docker 进行环境搭建，还是推荐 docker 进行服务部署，主要是省事，省事，省事，重要的事情说三遍。没有那么多配置规则，做好目录挂载就可。

```
docker pull gitlab/gitlab-ce:15.0.0-ce.0

生成挂载目录
mkdir -p /gitlab/etc/gitlab
mkdir -p /gitlab/var/log
mkdir -p /gitlab/var/opt

docker run
-d
-p 8943:443
-p 8990:80
-p 8922:22
--restart always
--name gitlab
-v /da1/apps/gitlab/etc:/etc/gitlab
-v /da1/apps/gitlab/log:/var/log/gitlab
-v /da1/apps/gitlab/data:/var/opt/gitlab
--privileged=true
gitlab-ce

--privileged=true         #让容器获取宿主机root权限


```

gitlab 配置

```
vim /home/gitlab/etc/gitlab/gitlab.rb

# 配置http协议所使用的访问地址,不加端口号默认为80
external_url 'http://192.168.1.1'

# 配置ssh协议所使用的访问地址和端口
gitlab_rails['gitlab_ssh_host'] = '192.168.1.1'
gitlab_rails['gitlab_shell_ssh_port'] = 222 # 此端口是run时22端口映射的222端口

:wq #保存配置文件并退出

# 重启gitlab容器
 docker restart-dev gitlab-dev

# 进入容器内部
docker exec -it gitlab /bin/bash

# 进入控制台
gitlab-rails console -e production

# 查询id为1的用户，id为1的用户是超级管理员
user = User.where(id:1).first
# 修改密码为 123456
user.password='123456'
# 保存
user.save!
# 退出
exit;

docker exec -it gitlab-dev grep ‘Password:’ /etc/gitlab/initial_root_password

# 初始化用户密码
gitlab-rake "gitlab:password:reset[root]"
```

### 数据迁移

目前服务器配置低，后续服务增加，配置跟不上，就需要进行数据迁移，需要具备这个技能，不可被动。

需要保证服务器 gitlab 版本与目的服务器一致。在源服务器执行以下命令。

```
#执行如下命令开始备份

gitlab-rake gitlab:backup:create

# 备份文件在，备份文件从gitlab.rb中查找

ll /var/opt/gitlab/backups

1659017650_2022_07_28_15.2.0_gitlab_backup.tar
```

通过 scp 命令传输到目的服务器。

```
#停止相关数据连接服务，如果是新环境，其实不必要重启

gitlab-ctl stop

#修改权限，如果是从本服务器恢复可以不修改

chmod 777 data/gitlab/backup/1659017650_2022_07_28_15.2.0_gitlab_backup.tar

#从1659017650_2022_07_28_15.2.0编号备份中恢复

sudo gitlab-rake gitlab:backup:restore BACKUP=1659017650_2022_07_28_15.2.0

#按照提示输入两次yes并回车 # 启动

sudo gitlab-ctl start
```

#### 数据定时备份

通过上述操作步骤，就可以完成 gitlab 数据迁移，这样之后服务器变更，也能恢复数据。做数据迁移，不单单是在服务器迁移在使用该方法，在日常开发中要有数据忧患意识。即使不迁移数据，我们也要做好数据备份操作。

```
# 系统定时备份任务

crontab -e # 每天2点定时备份

0 3 * * * gitlab-rake gitlab:backup:create

# 重启crontab

systemctl restart crond

# 定时清理备份 # gitLab已经支持，配置gitlab来实现自动清理功能

vim etc/gitlab/gitlab.rb

# 将backup_keep_time的配置取消注释，根据需要设置自动清理多少天前的备份，设置备份保留7天（7*3600*24=604800） gitlab_rails['backup_keep_time'] = 604800

#重新加载gitlab的配置文件

gitlab-ctl reconfigure
```

代码仓库地址: [MountsoftWeb](https://github.com/mountsoftweb/)

欢迎大家点击查看，觉着有用的话帮忙点个 star ，一起进步，成长！
