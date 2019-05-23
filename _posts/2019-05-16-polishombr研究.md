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

![主界面](https://cijian00.github.io/img/polichombr/1.png)



> 显示历史上传文件的MD5值，以此为链接。

```
-- Home（主界面）
-- Search（搜索界面）
# 三种检索方式。搜索文件hash、Machoc哈希表、文本信息（摘要，函数名称，字符串，文件名，分析结果）。
-- Families（群组）
# 将样本进行分类
-- signature
# 用于Yara匹配
```

>3.文件上传

![文件上传](https://cijian00.github.io/img/polichombr/2.png)

```
- Sensibility：TLP协议
-- TLP White  - 无限制
根据标准版权规则，WHITE信息可以自由分发，不受任何限制。
-- TLP Green - 社区范围
此类信息可以在特定社区内广泛传播。但是，这些信息可能不会在互联网上公开发布或公开发布，也不会在社区外发布。
-- TLP Amber - 限量发行
收件人可以与其组织内的其他人共享AMBER信息，但仅限于“需要知道”的基础上。可以期望发起者指定该共享的预期限制。
-- TLP Red - 仅限指定收件人
例如，在会议的背景下，RED信息仅限于出席会议的人员。在大多数情况下，RED信息将通过口头或亲自传递。
# 选择项多了一个TLP BLACK，【禁止分享】。
```

>4.分析界面

![分析界面](https://cijian00.github.io/img/polichombr/3.png)

```
- 样本元数据信息
-- 文件类型
-- 程序创建时间
-- 文件大小
-- MD5
-- SHA1
-- SHA256
-- 分析状态
-- 文件名称
-- 所属用户
-- 所属群组
- 摘要
- Machoc 签名
```

>5.二进制静态分析

![静态分析](https://cijian00.github.io/img/polichombr/4.png)

```
-	判断是否加壳
--	敏感函数调用
--	调用的API名称
--	函数起始地址、栈顶地址
--	入口点函数调用树
-	加密数据
-	PE文件

# .dll出现的位置地址，栈地址，函数地址；
```

>6.代码解析

- 基于函数地址信息生成程序流程图

![代码解析](https://cijian00.github.io/img/polichombr/5.png)

- 在文件存储目录/data/storage下生成.svg文件，用以展示程序流程图。

![代码解析](https://cijian00.github.io/img/polichombr/6.png)

- .bin文件以样本文件的SHA256值命名。

>7.API

```
import argparse

from poliapi.mainapi import SampleModule, FamilyModule

def send_sample(sample="", family=None, tlp=None, api_key=""):
    """
        Send a sample using the SampleModule API
    """
    sapi = SampleModule(api_key=api_key)
    if family is not None:
        fapi = FamilyModule(api_key=api_key)
        answer = fapi.get_family(family)
        if answer["family"] is None:
            print("[!] Error : the family does not exist...")
            return False
    sid = sapi.send_sample(sample, tlp)

    print("Uploaded sample ID : %d" % sid)
    if family is not None:
        answer = sapi.assign_to_family(sid, family)
        if not answer['result']:
            print("Cannot affect sample to family")
            return False
    return True

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Send sample via the API")
    parser.add_argument('api_key', type=str, help="Your API key")
    parser.add_argument('samples', help='the samples files', nargs='+')
    parser.add_argument('--family', help='associated family')
    parser.add_argument('--tlp', type=int,
                        help="The TLP level,\
                              can be from 1 to 5,\
                              1=TLPWHITE / 5=TLPBLACK")
    args = parser.parse_args()
    for sample in args.samples:
send_sample(sample, args.family, args.tlp, args.api_key)
```
- 测试
```
curl -XGET http://IP/api/1.0/samples/1/names/ --cookie "remember_token=……;session= "
```

![API](https://cijian00.github.io/img/polichombr/7.png)
