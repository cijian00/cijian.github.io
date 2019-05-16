---
layout: post
title: Polishombr - 二进制分析（未完）
tags: [工具]
comments: true
---

这款分析框架之前用过，简单记录下。

> polishombr项目Github地址


```
https://github.com/anssi-fr/polichombr/
```

#### 一、简介

> 1.dockerfile配置

```
FROM debian:jessie

MAINTAINER Tristan Pourcelot <tristan.pourcelot@ssi.gouv.fr>

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update
RUN apt-get upgrade -qq
RUN apt-get dist-upgrade -qq

RUN apt-get install -qqy git virtualenv ruby libffi-dev python-dev graphviz gcc libssl-dev

ADD https://github.com/anssi-fr/polichombr/tarball/master poli.tar.gz

RUN mv poli.tar.gz /opt/ && cd /opt/ && \
	tar xzf poli.tar.gz && mv ANSSI-FR-polichombr-* polichombr && \
	cd polichombr && \
	./install.sh
WORKDIR /opt/polichombr

RUN sed -i '/SQLALCHEMY_DATABASE_URI/c\SQLALCHEMY_DATABASE_URI = "sqlite:////opt/data/app.db"' config.py
RUN sed -i '/STORAGE_PATH/c\STORAGE_PATH = "/opt/data/storage"' config.py

ADD https://github.com/jjyg/metasm/tarball/master metasm.tar.gz
RUN tar xzf metasm.tar.gz && mv jjyg-metasm-*/* metasm && rm metasm.tar.gz

VOLUME "/opt/data/"
RUN cp /opt/polichombr/utils/db_create.py db_create.py
# RUN mv examples/db_create.py db_create.py

EXPOSE 5000
CMD flask/bin/python db_create.py && flask/bin/python run.py

```
```
以原Dockerfile创建polichombr会出现找不到examples/db_create.py的情况。
修改的dockerfile利用/opt/polichombr/utils/路径下的db_create.py即可。

RUN cp /opt/polichombr/utils/db_create.py db_create.py
# RUN mv examples/db_create.py db_create.py
```

> 2.docker运行

```
docker run -p 5000:5000 -v PATH_TO_YOUR_DATA/:/opt/data/ polichombr
```

```
基于docker的polichombr工具运行，-v指定一个宿主机目录（PATH_TO_YOUR_DATA），存储docker容器中/opt/data生成的文件。
```

#### 系统功能

> 1.功能模块

```
- 样本及文件存储
- 半自动化的恶意文件分析
- IDA PRO 集成
- 二进制代码在线解析
- Yara匹配
- 运用MACHOC模糊散列算法匹配二进制
```

> 2.主页面
