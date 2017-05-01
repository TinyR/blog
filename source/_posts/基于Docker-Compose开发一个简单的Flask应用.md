---
title: 基于Docker Compose开发一个简单的Flask应用
tags: [Docker,Web,Flask]
categories: [Docker]
date: 2017-05-01 18:47:23
---
&emsp;&emsp;前一篇介绍了如何使一个简单的Flask应用运行在Docker中，一个简单的应用，一个容器运行。一个项目往往还需要Mysql,Redis等等组件，web应用，Mysql，redis运行在各自的容器中。Docker Compose能够帮助定义并且运行有多个container的应用程序。下面将会开发一个简单的Flask应用，使用到了redis，使用Docker Compose运行这个应用。
## 简单的Flask应用
app.py

```python

from flask import Flask
import redis

app = Flask(__name__)
redis = redis.StrictRedis(host='redis', port=6379, db=0)
@app.route('/home')
def hello_flask():
    count = redis.incr('count')
    return 'Hello Flask! I have been seen {} times.\n'.format(count)

if __name__ == '__main__':
    app.run(host='0.0.0.0')

```
注意：host为redis。

## requirements.txt文件

运行命令：

>pip freeze > requirements.txt

在项目目录下就会生成requirements.txt文件，记录项目依赖。

## Dockerfile文件
在项目目录下新建Dockerfile文件。
```bash
#使用python3.5作为base image
FROM python:3.5

#设置container的工作目录
WORKDIR /app

#复制falsk应用到/app目录下
ADD . /app

#下载依赖包
RUN pip3 install -r requirements.txt

#使得container的5000端口对外暴露
EXPOSE 5000

#当container启动的时候执行该命令
CMD ["python", "app.py"]
```

## docker-compose.yml文件
在项目目录下新建docker-compose.yml文件。
```
version: '2'
services:
  web:
    build: .
    ports:
     - "5000:5000"
    volumes:
     - .:/app
  redis:
    image: "redis:alpine"
```
docker-compose.yml文件定义了两个services，一个是web,一个是redis。volumes表示将当前目录挂载到container的/app目录下，这样修改当前目录下的代码，再次启动时便会生效。

## 启动应用
在项目目录下执行:
>docker-compose up -d

运行命令：
>docker-compose ps

![docker-compose ps](https://github.com/TinyR/images/blob/master/Docker/%E5%9F%BA%E4%BA%8EDocker%20Compose%E5%BC%80%E5%8F%91%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84Flask%E5%BA%94%E7%94%A8_ps.PNG?raw=true)

## 浏览器访问
通过浏览器访问可看到:

![浏览器访问flask应用](https://github.com/TinyR/images/blob/master/Docker/%E5%9F%BA%E4%BA%8EDocker%20Compose%E5%BC%80%E5%8F%91%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84Flask%E5%BA%94%E7%94%A8_result.PNG?raw=true)






## 参考资料

[Docker官方:Get started with Docker Compose](https://docs.docker.com/compose/gettingstarted/)