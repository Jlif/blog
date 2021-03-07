---
layout: post
title: 使用 python 脚本下载图片
date: '2018-05-26'
categories: 瞎折腾
abbrlink: 835d5244
---

## 前言

好久没更新博客了，最近来一波更新吧。

咳咳，最近得到了一批纯洁图片的 url 地址，起因是有人用爬虫爬取纯洁网站的图片。刚好很久没折腾了，于是我想着使用 python 脚本，把它们都下载下来。
<!-- more -->

## 第一个简单的版本：单进程下载

话不多说，直接上代码。

```python
import sqlite3
import urllib.request

conn = sqlite3.connect('seePicture.db')
cursor = conn.cursor()
cursor.execute('select * from picture')
# 执行查询语句
values = cursor.fetchall()
# select 语句执行通过 fetchall（）拿到结果集，是一个 list，每个元素是 tuple
cursor.close()
conn.close()

imgName = 0
li = []
head = 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/66.0.3359.181 Safari/537.36'

for t in values:
    url = t[3]
    li.append(url)

def get_content(pic_url, headers):
    req = urllib.request.Request(pic_url)
    req.add_header("User-Agent", headers)

    content = urllib.request.urlopen(req).read()
    return content

for url in li:
    try:
        f = open('D:\\Temp\\pic\\' + str(imgName) + '.jpg', 'wb')
        c = get_content(url, head)
        f.write(c)
        print(url + ' ok')
        f.close()
    except Exception as e:
        print(url + ' error')
        print(e)
    imgName += 1

print("All Done!")
```

简单解释一下，这个直接就是从 sqlite 中取出所有数据，因为本身数据量就不大，我看了下就一千多条记录，可以一次性全部取出。将取出的记录中的第 4 个字段，也就是图片地址那个字段取出来，组成一个新的 list。然后就是对这个 list 进行遍历，将它下载保存到`D:\Temp\pic`中。

刚开始我直接下载的时候我看报错信息是`HTTP Error 403: Forbidden`，于是我加上了请求头`User-Agent`，然后就能正常请求下载了。

## 第二个版本：使用多进程，提高下载速度

第一个版本里面，使用的是本地的 sqlite 数据库，其实就是一个几十 k 的文件，图片太少了。

然后我要到了数据库，看了下将近两百万条，这个应该够看了。跟上面相比只是连接的数据库不一样了，其他的都一样。

但是这样存在一个问题，就是下载太慢了，因为毕竟是单线程下载。然后我在网上看了好几份代码，结合了一下，同时修改了代码结构，整出一个多进程的版本，代码如下：

```python
import requests
import pymysql
from multiprocessing import Pool
import datetime

class Spider:

    @staticmethod
    def get_list():
        db = pymysql.connect(host="192.168.1.1", user="root", password="root", db="db", port=3306)
        # 使用 cursor() 方法获取操作游标
        cur = db.cursor()
        # 编写 sql 查询语句  user 对应我的表名
        sql = "SELECT * FROM picture WHERE id > 0 LIMIT 4000"
        try:
            cur.execute(sql)  # 执行 sql 语句
            results = cur.fetchall()  # 获取查询的所有记录
        finally:
            db.close()  # 关闭连接
        url_list = []
        for t in results:
            url = t[2]
            url_list.append(url)
        return url_list

    @staticmethod
    def get_pic(pic_url, file_name):
        try:
            resource = requests.get(pic_url)
            with open(file_name, 'wb') as file:
                file.write(resource.content)
                print(pic_url + " ok")
        except Exception as e:
            print(pic_url + " error")
            print(e)

if __name__ == "__main__":
    spider = Spider()
    urls = spider.get_list()
    p = Pool(40)
    d1 = datetime.datetime.now()
    j = 0

    for i in urls:
        p.apply_async(spider.get_pic, args=(i, 'D:\\Temp\\pic\\' + str(j) + '.jpg'))
        j += 1

    p.close()
    p.join()
    d2 = datetime.datetime.now()
    print(d2 - d1)
    print("All Done!")
```

上述代码主要是使用了多进程来执行任务，从而加快执行的速度。使用上述代码之后，我发现下载图片的速度相比原来大大提高，一下子就能下载好多张。

## 第三个版本：直接命令行启动

有了上述版本之后，其实就已经能正常使用了，但是由于我是通过 ide，也就是 PyCharm 直接启动的，如果日常使用的话，会十分不方便。然后我就对它改造了一下，使得这个脚本直接可以在命令行里面启动，启动的时候给个参数就行。

代码如下：

```python
import requests
import pymysql
from multiprocessing import Pool
import datetime
import sys

class Spider:

    @staticmethod
    def get_list(num):
        db = pymysql.connect(host="192.168.1.1", user="root", password="root", db="db", port=3306)
        # 使用 cursor() 方法获取操作游标
        cur = db.cursor()
        # 编写 sql 查询语句  user 对应我的表名
        sql = 'SELECT * FROM picture WHERE id > ' + num + ' LIMIT 4000'
        try:
            cur.execute(sql)  # 执行 sql 语句
            results = cur.fetchall()  # 获取查询的所有记录
        finally:
            db.close()  # 关闭连接
        url_list = []
        for t in results:
            url = t[2]
            url_list.append(url)
        return url_list

    @staticmethod
    def get_pic(pic_url, file_name):
        try:
            resource = requests.get(pic_url)
            with open(file_name, 'wb') as file:
                file.write(resource.content)
                print(pic_url + " ok")
        except Exception as e:
            print(pic_url + " error")
            print(e)

if __name__ == "__main__":
    arg = sys.argv[1]
    print('传入的参数为：' + arg)

    j = int(arg) + 1

    spider = Spider()
    urls = spider.get_list(str(j))
    p = Pool(40)
    d1 = datetime.datetime.now()

    for i in urls:
        p.apply_async(spider.get_pic, args=(i, 'D:\\Temp\\pic\\' + str(j) + '.jpg'))
        j += 1

    p.close()
    p.join()
    d2 = datetime.datetime.now()
    print(d2 - d1)
    print("All Done!")
```

主要是使用了`sys.argv[1]`这个变量，`sys.argv[0]`默认就是文件名，`sys.argv[1]`就是输入的第一个参数，另外就是注意一下哪里该用整数，哪里该用字符串。

在命令行环境下，假设这个 python 脚本名字为`spy.py`，启动的时候输入：

```shell
python spy.py 0
```

即可开始运行脚本。

> 注意：直接运行脚本的时候还要注意一个细节，就是引入的库是全局存在的，还是当前项目所引入的，不然可能会报`ModuleNotFoundError: No module named 'XXX'`异常。