---
layout: post
title: "当FastAPI遇上Nginx Unit"
date: 2022-07-30 12:22:14 +0800
categories: 原创
---

Python素来以慢而闻名。正所谓，“都用Python了，还在乎什么性能”。Django、Flask无不以简单易用而著称，但却距离高性能很远。

Python自3.4版本加入了asyncio库以来，异步编程的生态得到了长足发展。Django基金会发布的ASGI协议，更是将异步HTTP服务器和上层的路由设施解耦开来。由此，高性能的C语言编写的服务器便得到了大施拳脚的空间。

本次介绍的就是利用高性能的路由框架FastAPI和Nginx Unit服务器，组合成为一个高性能的服务端框架， 让你的业务逻辑跑得飞快。

[FastAPI](https://fastapi.tiangolo.com/)，是一个高性能的路由框架。主要负责将解析好的HTTP请求交给合适的代码逻辑来处理。

[Nginx Unit](https://unit.nginx.org)，是一个由Nginx团队维护使用C语言编写的高性能HTTP服务器，可以做反向代理、静态代理，也可以做应用服务器。作为应用服务器，Nginx Unit支持Go、Java、JS等多门语言。对于Python语言，Nginx Unit支持WSGI、ASGI两种协议。在Python的使用场景下，Nginx Unit主要负责处理TCP网络连接，以及HTTP报文的接收、解析和发送。

话不多说，直接看代码。

### 代码结构

这里我把所有的代码、配置都放在同一个目录下`fastapi-unit-demo`。

```
➜ tree

.
├── config
│   └── config.json                            # Nginx Unit的应用配置
├── docker-compose.yml                         # 启动入口
├── dockerfile                                 # 构建自己的镜像
├── logs
│   └── unit.log                               # 空文件，应用启动后，log会输出在这里
├── requirements.txt                           # 应用的依赖
├── src
│   └── demo.py                                # demo的入口
├── state                                      # 空文件夹，交给Nginx Unit使用
└── tsinghua.mirror                            # 构建镜像使用的清华大学mirror

4 directories, 7 files
```

#### Python代码

```
➜ cat src/demo.py

from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello, World!"}
```

#### Nginx Unit配置文件

```
➜ cat config/config.json

{
    "listeners":{
        "*:80":{
            "pass":"applications/fastapi"    # 这一行关联的是下方的具体配置
        }
    },
    "applications": {
        "fastapi": {
            "type": "python 3.10",           # 和镜像自带的Python版本保持一致
            "path": "/www",                  # 代码的路径
            "module": "demo",                # 启动的入口
            "callable": "app"                # ASGI app的变量名
        }
    }
}
```

#### 构建镜像

```
➜ cat dockerfile

FROM nginx/unit:1.27.0-python3.10

COPY requirements.txt /config/requirements.txt
COPY tsinghua.mirror /etc/apt/sources.list

RUN apt update && apt install -y python3-pip                                    \
    && pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple \  # 配置pypi镜像，按具体情况可以去掉
    && pip3 install -r /config/requirements.txt                                 \
    && apt remove -y python3-pip                                                \
    && apt autoremove --purge -y                                                \
    && rm -rf /var/lib/apt/lists/* /etc/apt/sources.list.d/*.list
```

#### docker-compose配置

```
➜ cat docker-compose.yml

version: "3.3"
services:
  app:
    build:
      context: .
      dockerfile: dockerfile
    volumes:
      - ./config:/docker-entrypoint.d/
      - ./logs/unit.log:/var/log/unit.log
      - ./state:/var/lib/unit
      - ./src:/www
    ports:
      - 8080:80
```

#### 启动demo看看

在代码根目录下执行命令：
```
➜ docker-compose up
```

启动完成后，打开浏览器，访问`http://localhost:8080`,就可以看到响应了`{"message":"Hello, World!"}`。

### 性能一瞥

使用wrk简单的测试一下hello-world性能。CPU是AMD Ryzen 5800X，系统是Debian 11，内核版本5.10。

> 注意，Nginx Unit服务器默认使用1个线程处理请求，所以Python代码理应跑在单进程模式中。

```
➜  wrk git:(master) ✗ ./wrk -t2 -c20 -d10s --latency http://localhost:8080

Running 10s test @ http://localhost:8080
  2 threads and 20 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.38ms   56.89us   3.43ms   96.40%
    Req/Sec     7.29k   133.32     7.64k    83.00%
  Latency Distribution
     50%    1.38ms
     75%    1.38ms
     90%    1.39ms
     99%    1.51ms
  145182 requests in 10.00s, 21.60MB read
Requests/sec:  14517.95
Transfer/sec:      2.16MB
```

对香农计划的未来成果简直更期待了呢~
