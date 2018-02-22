---
title: 在云服务器上搭建hexo
date: 2017-10-05 17:16:22
tags: hexo
---
之前用github托管过hexo服务，这次利用假期时间在自己的服务器上搭建了hexo服务，在经历过各种踩坑，查资料后总结出了这篇内容，若有不正确的内容还请大家多多包涵。

### 搭建环境介绍

本次搭建博客的服务器为阿里云ECS,操作系统为CENTOS 7.3

### 下载hexo框架

全局模式安装hexo
``` bash 
npm install hexo -g
```
执行初始化命令，初始化项目
``` bash
hexo init
```
生成静态页面
``` bash
hexo generator
```
获取简写为
``` bash
hexo g
```
然后在本地执行
```bash
hexo server
```
若这里报错出错，需要执行下面的命令下载hexo-server模块
```bash
npm install hexo-server
```
hexo服务默认监听在4000端口，这时我们使用localhost:4000即可访问到hexo生成的博客系统

### hexo基础命令
这里先介绍一些hexo的常用命令
```bash
    //新建文章
    hexo new "postName"
    //新建页面
    hexo new page "pageName"
    //生成静态页面至public目录
    hexo generate 
    //开启本地服务(默认端口4000)
    hexo server
    //部署hexo
    hexo deploy
    //查看帮助
    hexo help
    //查看Hexo的版本
    hexo version
```

### 服务器配置
这里先说一下自己服务器上搭建hexo的思路，在github上部署hexo服务是使用github上的静态站托管方式来托管hexo的静态站，这里我们使用自己的服务器采用nginx的方式托管静态站。

#### 1.配置服务器的node环境
这里我使用nvm部署nodejs服务，具体步骤如下。

1  在root用户或者权限较高的用户根目录下执行命令
```bash
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.2/install.sh | bash
```
2  重启窗口，可以输入nvm查看nvm是否安装成功。

3 安装某个特定版本的nodejs
```bash
nvm install v8.4.0
```
4 设置服务器使用该版本的nodejs
```bash
nvm use v8.4.0
```
5 设置默认的nodejs版本
```bash
nvm alias default v8.4.0
```
6 测试nodejs版本是否正确
```bash
node -v
```
至此，nodejs环境安装完毕。

### 2.搭建线上git服务
这里我们在自己的服务器上搭建git服务，目的是可以使用hexo deploy命令在更新代码的同时并在自己的平台线上部署 。

1 创建git用户
```bash
adduser git
```
2 赋予git用户权限
```bash
sudo visudo
```
在
```bash
root ALL=(ALL) ALL
```
下增加
```bash
git ALL=(ALL:ALL) ALL
```
之后保存退出

3 给git用户配置密码
```bash
sudo passwd git
```

4 禁用git用户的shell登陆权限

出于安全考虑，我们要让git用户不能通过shell登录。可以编辑/etc/passwd来实现
```bash
vi /etc/passwd
```
将
```bash
git:x:1001:1001::/home/git:/bin/bash
```
改成
```bash
git:x:1001:1001::/home/git:/usr/bin/git-shell
```
这样git用户可以通过ssh正常使用git,但是无法登录shell。

5 服务器端下载git

我们已经将git用户的内容全部配置完毕，接下来我们下载git,下载之前，我们先将本地的库进行更新。
```bash
sudo yum update
```
接下来下载git
```bash
sudo yum install git
```
6 创建git仓库

接下来创建一个git裸仓库，我选择的路径为/var/repo(注意，git用户需要有该路径下目录的读写权限)，在该路径下执行
```bash
git init --bare hexoBlog.git
```
并将改仓库的拥有者改为git
```bash
sudo chown -R git:git hexoBlog.git
```
7 修改git的hooks

进入到hexoBlog.git中的hooks中
```bash
cd ./hexoBlog.git/hooks
```
将其中的post-update.sample重命名为post-update。
```bash
mv ./post-update.sample ./post-update
```
这个钩子用于在将本地的代码推送到远程的这个库时会执行这个钩子中的内容，这里我们使用这个钩子在将我们的代码推送到该库时会自动将其下载到某个具体的文件夹下。
```bash
vi ./post-update
```
在其中增加内容
```bash
git --work-tree=/var/www/hexo --git-dir=/var/repo/hexoBlog.git checkout -f
```
增加内容后保存并退出，这样可以使在将代码上传到git的同时，可以将代码下载到/var/www/hexo文件夹下。这样关于git的配置
就到此为止。

### 3.搭建线上nginx服务
这里我们使用nginx托管我们的静态服务。

1 下载nginx

我们使用yum下载nginx
```bash
sudo yum install nginx
```
若可以输出nginx版本，则证明安装成功
```bash
nginx -v
```
2 配置nginx的配置文件

安装成功后，跳转到/etc/nginx目录下新建一个nginx配置文件，这里我建立的配置文件为blog-static.conf，具体的操作如下所示。
```bash
cd /etc/nginx
touch blog-static.conf
```
配置文件中的内容如下所示
```bash
server
{
    listen 80;
    server_name blog.littlebearbbb.cn;
    index index.html index.htm default.html default.htm;
    root /var/www/hexo;
    location ~ .*\.(ico|gif|jpg|jpeg|png|bmp|swf)$
    {
        access_log off;
        expires 1d;
    }

    location ~ .*/.(js|css|txt|xml)?$
    {
        access_log off;
        expires 12h;
    }

    location / {
        try_files $uri $uri/ =404;
    }

    access_log /var/wwwlogs/blog.log main;
```
这里需要注意的点是server_name为该静态网站绑定的域名,root为该网站根目录所在位置。
 
配置完毕后重启nginx服务，使其生效
```bash
sudo systemctl restart nginx.service
```
至此服务器相关的配置我们就全部完成了。

### 4.本地hexo博客上线流程
1 修改配置文件

修改根目录下的_config.yml配置文件，将deploy中的type改为git,repo改为git仓库的所在地址。

2 上线流程
```bash
#新建文章
hexo new "postName"
#生成静态文件
hexo g
#上传至服务器
hexo d
```










