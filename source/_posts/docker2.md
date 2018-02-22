---
title: jenkins+docker实现项目持续部署
date: 2018-01-27 16:42:32
tags: docker
---

#### 1.安装jenkins
在阿里云上安装jenkins的docker镜像
```bash
##在dockerhub 上拉取最新的jenkins镜像
docker pull jenkins:latest
```
在阿里云上使用docker命令开启jenkins服务
```bash
#参数
#d:后台运行 p:端口号映射 v:券挂载，这里的映射关系为将docker镜像内的/var/jenkins_home挂载到服务器上的/var/jenkins_node上
sudo docker run -d --name jenkins_node -p 49002:8080 -v /var/jenkins_node:/var/jenkins_home jenkins:latest
```
启动jenkins有可能会因为没有/var/jenkins_node的目录权限而启动失败，若遇到这种情况，可以修改该目录的权限
```bash
chmod 777 /var/jenkins_node 
```



#### 2.部署node服务

这里使用测试node服务,app.js中的内容如下所示
```bash
'use strict';

const express = require('express');

// Constants
const PORT = 8080;
const HOST = '0.0.0.0';

// App
const app = express();
app.get('/', (req, res) => {
  res.send('Hello world\n');
});

app.listen(PORT, HOST);
console.log(`Running on http://${HOST}:${PORT}`);
```

Dockerfile中的内容如下所示
```bash
#获取node镜像源
FROM node:carbon

#创建应用目录
WORKDIR /usr/src/app

#将package.json和package-lock.json拷贝至应用目录
COPY package*.json ./

#执行npm install命令
RUN npm install

#将nodejs服务下的文件全部拷贝到应用目录下
COPY . .

#向外部暴露出应用所使用的8080端口
EXPOSE 8080

#执行npm start启动服务
CMD [ "npm", "start" ]
```

.dockerignore中的内容如下所示

```bash
node_modules
npm-debug.log
```

这里是将该项目托管到码云上进行管理

#### 3.配置jenkins服务
当jenkins启动后，需要对jenkins进行配置，因为我们配置jenkins的端口为49002，这里我们可以在浏览器上使用[hostip]:49002的方式进行启动，第一次启动会进入下图所示页面  
![image](/img/jenkins/jenkins2.png)  
我们需要在输入框中输入密钥，系统提示我们需要在/var/jenkins_home/secrets/initialAdminPassword位置获取密钥，由于我们使用了券挂载的方式，可以在卷挂载的位置获密钥，使用下方命令打印密钥
```bash
##cat命令用于输出文件中的内容
cat /var/jenkins_node/secrets/initialAdminPassword
```
将输出结果拷贝到文本框中点击continue完成验证内容。  
选择安装推荐的插件  
![image](/img/jenkins/jenkins3.png)  
安装中  
![image](/img/jenkins/jenkins4.png)  
安装后需要创建jenkins登录账号
![image](/img/jenkins/jenkins5.png)

进入到系统中我们选择系统管理下的管理插件

![image](/img/jenkins/jenkins1.png)

选择可选插件选项卡，可以根据ssh进行过滤，查询到Publish Over SSH勾选进行后点击直接安装按钮  
![image](/img/jenkins/jenkins6.png)  
等待安装。。。  
![image](/img/jenkins/jenkins7.png)  
安装完毕后返回首页，选择系统管理下的系统设置，下滑至Publish over SSH部分，点击增加按钮  
![image](/img/jenkins/jenkins8.png)  
之后在name处填写为该server起的名称，在hostname处填写服务器的ip地址,username处填写服务器的用户名，在上方的Passphrase处填写该用户登录服务器的密码,填写完毕后点击下方的Test Configuration的按钮，测试通过后点击左下方的保存，对配置进行保存。  
![image](/img/jenkins/jenkins9.png)  
配置后回到新建页面，点击创建一个新任务，选择构建一个自由风格的软件项目  
![image](/img/jenkins/jenkins10.png)  
创建后在源码管理部分选择使用GIT进行管理，Repository URL这里我是选择使用http形式的url地址。若使用该方式的url，需要创建Credentials,这里我们点击add创建。  
![image](/img/jenkins/jenkins11.png)  
点击添加后进入添加认证的界面，在该界面设置登录到git上的用户名和密码，确认无误后点击添加按钮。   
![image](/img/jenkins/jenkins12.png)   
之后在Credentials下拉选单中选择已经添加好的认证。
之后配置构建，点击增加构建步骤按钮，点击其中的Send files or execute commands over SSH  
![image](/img/jenkins/jenkins13.png)  
在构建面板中SSH Server的Name部分的下拉选框我们选择我们已经配置好的ssh server。	Exec command 部分配置如下所示的command命令
```bash
sudo docker stop docker-test || true && sudo docker
rm docker-test || true && cd /var/jenkins_node/workspace/docker-test && sudo docker build -t docker-test . && sudo docker run --name docker-test -p 8080:8080 -d docker-test
```
该命令的含义为若存在当前服务的容器先进行删除，然后跳转到git默认拉取的目录下再进行容器的构建，之后保存设定。  
![image](/img/jenkins/jenkins14.png)  
之后返回主面板，点击主面板最右侧的开始按钮进行部署。  
![image](/img/jenkins/jenkins15.png)


>注意：在部署时可能会报出：sudo: no tty present and no askpass program specified，这种错误是由于设定的用户在在使用sudo命令时需要输入密码导致的，解决该问题需要登录到该服务器上在root用户下执行vi /etc/sudoers，添加一行免密码设置:[用户名] ALL = NOPASSWD: ALL。

文章引用于[一步一步打造jenkins+docker+nodejs项目的自动部署环境](https://www.jianshu.com/p/052a2401595a) 



 



 