---
title: Mysql + Grafana 监控爬虫程序
tags: [爬虫, 监控, Mysql, Grafana]
date: 2018-06-16 16:38:20
categories: 爬虫
permalink: crawler-monitor
keywords: 爬虫, JS, Mysql, Grafana
---

在使用爬虫爬取**大量**数据的时候，一般我们都会把程序挂在服务器上运行，然后就可以去干别的事情了。但是，我们还是有必要定时看一下程序运行情况的。虽然我们可以通过 log 信息来监控程序运行情况，但这往往不够直观。所以，今天我就讲讲如何使用 Mysql 和 Grafana 监控爬虫程序的**运行状况**，并**可视化**。

<!-- more -->

<br>

# 1. Grafana 简介



> Grafana 是一个数据可视化工具，它并不收集数据，但是可以从数据源（例如 Graphite、Mysql、InfluxDB等）中获取数据并可视化。

<br>

# 2. 运行 Grafana

Grafana 安装教程可以去网上搜，不多说。这里说的是另一种替代方法：使用已经安装好 Grafana 的 Docker 镜像（效果也是一样的）。如果不了解 Docker 的话，可以看下教程 [Docker — 从入门到实践](https://docker_practice.gitee.io/)

在这里，我们需要先安装好 Docker，并学会一些 Docker 基本命令，例如拉取镜像，容器的创建，容器的运行停止，镜像和容器的删除等等。

安装并运行 Docker 之后：

1. 在 [Docker hub](https://hub.docker.com/) 搜索一下包含 Grafana 的镜像，还是出现挺多个的。这里我选择了 [[kamon](https://hub.docker.com/u/kamon/)/[grafana_graphite](https://hub.docker.com/r/kamon/grafana_graphite/)
  ](https://hub.docker.com/r/kamon/grafana_graphite/) 

2. 在服务器上拉取镜像

```bash
docker pull kamon/grafana_graphite
```

3. 使用该镜像创建容器，并在后台运行
```bash
docker run \
  -d \
  -p 80:80 \
  -p 81:81 \
  -p 2003:2003 \
  -p 8125:8125/udp\
  -p 8126:8126\
  --name=grafana_graphite \
  kamon/grafana_graphite 
```
到这里，我们就得到了一个已经安装了 Grafana 的容器，根本就不用我们手动安装了O(∩_∩)O 哈哈~

4. 在浏览器中打开 http://your_server_ip:80/，登录（初始用户名和密码都是 admin），我们就可以看到 Grafana 的控制台了，还是挺酷炫的！

<br>

# 3. 编写爬虫程序并运行
略。（在这里，要将爬取到的 item 储存起来，例如插入 mysql 数据库）

<br>

# 4. 编写监控的脚本并运行
这里，我们要每隔一定时间查询爬取总量，并计算爬取速度。下面是一个例子：
代码分为两部分，首先是在我们存放 item 的数据库建立两个表，每个表有两个字段，一个是查询时间，另一个是 `item_total / item_min`。
```python
import pymysql as mdb
import time

# 存放爬取数据的数据库（这里我把统计的数据，存入了爬取数据所在的数据库）
DB_NAME = 'db_name'
TABLE_NAME1 = 'item_per_min'
TABLE_NAME2 = 'item_total'
host = 'your_server_ip'
user = 'your_user_name'
passwd = 'your_pwd'

# 爬取的 item 存放的表
item_table = 'item_table'

def create_table():
    use_db_str = 'use ' + DB_NAME
    create_table_str1 = "CREATE TABLE if not exists " + TABLE_NAME1 + """(
      `time` datetime NOT NULL,
      `speed` int NOT NULL DEFAULT '0'
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;"""

    create_table_str2 = "CREATE TABLE if not exists " + TABLE_NAME2 + """(
        `time` datetime NOT NULL,
        `total` int NOT NULL DEFAULT '0'   
        ) ENGINE=InnoDB DEFAULT CHARSET=utf8;"""

    # 数据库连接
    conn = mdb.connect(host, user, passwd)
    cursor = conn.cursor()
    try:
        cursor.execute(use_db_str)
        cursor.execute(create_table_str1)
        cursor.execute(create_table_str2)
        conn.commit()
    except Exception as e:
        print(e)
```

代码第二部分是主函数，每隔一分钟查询 items 数，并计算爬取速度，将得到的数据储存起来：

```python
if __name__ == '__main__':
    # 创建表
    create_table()
    conn = mdb.connect(host, user, passwd, DB_NAME)
    cursor = conn.cursor()

    before_item = 0
    while True:
        try:
            cursor.execute('SELECT count(*) FROM %s', item_table )
            result = cursor.fetchone()
            current_item = result[0]
            print(current_item)

            # 过去一分钟爬取量
            cursor.execute('insert into %s values (now(), %s)' % (TABLE_NAME1, current_item - before_item))
            # 爬取总量
            cursor.execute('insert into %s values (now(), %s)' % (TABLE_NAME2, current_item))

            conn.commit()
            before_item = current_item
        except Exception as e:
            print(e)

        time.sleep(60)
```
监控脚本写完后，就可以挂在服务器后台运行了

<br>

# 5. Grafana 配置

1. 配置数据源，这里命名为 `monitor_crawler`

   ​

<img src="/images/2018/06-09-1.png" align=center />



2. 新建 Dashboard，然后点击 Graph 图标创建图，接着点击 `Panel Title` -> `Edit`

   ​

<img src="/images/2018/06-09-2.png" align=center />



3. 选择我们刚才创建的数据源 `monitor_crawler`

   ​

<img src="/images/2018/06-09-3.png" align=center />



4. 按照 Grafana 提供的模板填写 sql 语句，这里查询了 `item_per_min` 表
```
SELECT
  UNIX_TIMESTAMP(time) as time_sec,
  speed as value,
  'items_min' as metric
FROM item_per_min
```

（注意我们选择的是 `Time series`）



<img src="/images/2018/06-09-4.png" align=center />



5. 可以选择绘图模式，一般是选 `Lines`

   ​


<img src="/images/2018/06-09-5.png" align=center />



# 6. 成果展示



<img src="/images/2018/06-09-6.png" align=center />




（其实 Grafana 还有很多很酷炫的设置，大家有兴趣可以去探索一下！）


# 总结

在安装好各种环境之后，Mysql + Grafana 监控爬虫程序的步骤：
> 1. 编写爬虫程序
> 2. 编写监控脚本，将爬取速度和爬取总量定时存进 Mysql 数据库
> 3. Grafana 新建数据源，连接对应的 Mysql 数据库
> 4. 创建新的 Dashboard，并在里面创建图表，图表数据源选择我们上一步新建的数据源