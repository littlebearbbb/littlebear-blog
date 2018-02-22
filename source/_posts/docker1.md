---
title: Node.js官方Docker化Demo
date: 2018-01-13 11:32:02
tags: docker
---
#### 简介
>该文为node.js官网提供的将nodejs构建在docker的简单翻译，可以作为docker最简单的入门，本文并没有docker安装的具体内容，安装docker请查看docker安装文档

#### 1.创建node app服务
package.json内容如下所示
```bash
{
  "name": "docker_web_app",
  "version": "1.0.0",
  "description": "Node.js on Docker",
  "author": "First Last <first.last@example.com>",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.16.1"
  }
}
```
app.js内容如下所示
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
#### 2.创建Dockerfile文件
在项目路径下创建Dockerfile文件,该文件为docker的配置文件
```bash
touch Dockerfile
```
Dockerfile中的具体内容如下所示
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

### 3.创建.dockerignore
该文件的作用类似于.gitignore文件，用于指明服务下的哪些文件不必要放在镜像中，该文件我的配置如下所示
```bash
node_modules
npm-debug.log
```

### 4.创建镜像
在你的项目目录下存在Dockerfile的前提下就可以使用docker build的方式来创建
```bash
### -t用于给镜像设置flag，用于方便查找镜像
$ docker build -t <your username>/node-web-app .
```

docker images命令用于列出当前存在的所有的镜像
```bash
docker images
```

### 5.启动镜像

```bash
### -d用于以后台模式启动镜像
### -p用于将本机的一个端口代理到容器中的一个端口
$ docker run -p 49160:8080 -d <your username>/node-web-app
```
打印容器日志输出:
```bash
# 获取容器的id
$ docker ps

# 打印目标id容器的日志输出
$ docker logs <container id>

# Example
Running on http://localhost:8080
```

如果你想进入到容器中，可以使用以下的命令
```bash
# Enter the container
$ docker exec -it <container id> /bin/bash
```

### 6.测试容器
为了测试容器的联通性，需要先获得容器映射的端口号
```bash
$ docker ps

# Example
ID            IMAGE                                COMMAND    ...   PORTS
ecce33b30ebf  <your username>/node-web-app:latest  npm start  ...   49160->8080
```
接下来使用curl来测试服务的联通性
```bash
$ curl -i localhost:49160

HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 12
ETag: W/"c-M6tWOb/Y57lesdjQuHeB1P/qTV0"
Date: Mon, 13 Nov 2017 20:53:59 GMT
Connection: keep-alive

Hello world
```
