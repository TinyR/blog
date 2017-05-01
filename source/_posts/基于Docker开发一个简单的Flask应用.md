---
title: 基于Docker开发一个简单的Flask应用
tags: [Docker,Web,Flask]
categories: [Docker]
date: 2017-04-30 22:23:19
---
&emsp;&emsp;五一宅在了住的地方，写篇博客记录下：怎么让一个Flask应用运行在Docker中。

## 简单的Flask应用

app.py

```Python
from flask import Flask
app = Flask(__name__)

@app.route('/home')
def hello_world():
    return 'Hello Flask'

if __name__ == '__main__':
    app.run(host='0.0.0.0')
```

## 生成requirements.txt文件

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
## 生成镜像
在项目目录下执行命令：
>docker build -t rna/flask .

rna/flask为镜像的名称。运行该命令后，会依次执行Dockerfile里的指令，最后生成镜像。可通过docker images查看所有镜像。

## 运行container
在项目目录下执行命令：
>docker run -d --rm -p 5000:5000 rna/flask

-p 5000:5000使得本机5000端口映射到container的5000端口。

![docker ps](https://github.com/TinyR/images/blob/master/Docker/%E5%9F%BA%E4%BA%8EDocker%E5%BC%80%E5%8F%91%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84Flask%E5%BA%94%E7%94%A8_docker_ps.PNG?raw=true)

## 访问
通过浏览器访问

![浏览器访问flask应用](https://github.com/TinyR/images/blob/master/Docker/%E5%9F%BA%E4%BA%8EDocker%E5%BC%80%E5%8F%91%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84Flask%E5%BA%94%E7%94%A8_%E8%AE%BF%E9%97%AE.PNG?raw=true)


## 参考资料

[Docker官方:Get Started, Part 2: Containers](https://docs.docker.com/get-started/part2/)

[Dockerize Simple Flask App](http://containertutorials.com/docker-compose/flask-simple-app.html)

[Dockerfile reference](https://docs.docker.com/engine/reference/builder/)
