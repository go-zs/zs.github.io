---
title: 超实用压力测试工具
date: 2019-12-30 10:55:39
updated: 2020-01-13 16:55:03
tags:
---

ab全称为：ApacheBench，是Apache http性能测试工具，其设计意图是描绘当前所安装Apache的执行性能，当然，它也可以对其它类型的服务器进行压力测试。

<!-- more -->

## 安装
1. 手动下载，进入apache官网 http://httpd.apache.org/ 下载apache即可。
2. Linux命令行安装
```
# ubuntu
sudo apt-get install apache2-utils

# CentOS
yum -y install httpd-tools

# mac
mac自带了ab
```

## 基本使用
ab使用很简单，注意根目录的`url` 结尾必须是`/`

```
-c 并发数
-n 请求总数
ab -c 1000 -n 10000 https://baidu.com/
```

## 输出说明
ab输出结果指标比较多，一般关注吞吐率和平均等待时间即可。
```
Requests per second: 吞吐率
Time per request： 平均等待时间
Time per request: 服务器平均请求时间
```
![ab测试结题](ab测试/ab测试输出.jpg)

## ab添加请求头
-H 参数可以传多个header
```
-H "clientId:test-client" -H "token:93e6acff-2ef9-4c85-9d0b-c9948a8ee93b"
```


## ab POST 请求
将请求体写入 `post.txt` 中
-p post body内容，内容需要与编码相符
-T 指明编码形式

```
# urlencode 比如原data={"username":"zs"}, postdata要进行urlencode加密
ab -c 1000 -n 10000 -p post.txt -T application/x-www-form-urlencoded https://baidu.com

# json postdata {"username":"zs"}
ab -c 1000 -n 10000 -p post.txt -T application/json https://baidu.com


# form-data  boundary可以自定义，但必须与post data匹配
# ------ceshi\r\nContent-Disposition: form-data; name="accountEmail"\r\n\r\njiaowei@datagrand.com\r\n------WebKitFormBoundaryT5vjZWundP38bTD3\r\nContent-Disposition: form-data;     name="password"\r\n\r\njiaowei\r\n------ceshi--
ab -n 10000 -c 1000 -p post.txt -T 'multipart/form-data; boundary=---ceshi' "http://rpa-v5.datagrand.com/token?_allow_anonymous=true"

```
