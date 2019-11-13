---
title: frp内网穿透教程
date: 2019-11-10 15:38:57
tags:
---

移动互联网应用的开发在很多场景下需要我们的内网服务能够在公网被访问，通常需要用到内网穿透工具。FRP就是这样一款开源、免费、跨平台的内网穿透工具。FRP 全名：Fast Reverse Proxy。FRP 是一个使用 Go 语言开发的高性能的反向代理应用，可以帮助您轻松地进行内网穿透，对外网提供服务。FRP 支持 TCP、UDP、HTTP、HTTPS等协议类型，并且支持 Web 服务根据域名进行路由转发。

<!-- more -->

## 准备工作

1. 公网服务器
   我们需要一台公网服务器作为中转节点。推荐选择国外云商选AWS的Amazon EC2的T2实例，T2实例主要考虑到一些客户无需满负荷运转CPU，但偶尔可能也会有突发性高CPU性能需求, 开发场景下这种服务器性价比非常高。国内阿里云也有这种突发性能的型号可以选择。

2. 下载frp软件
   frp在[github](https://github.com/fatedier/frp)上开源，打开软件的[release页面](https://github.com/fatedier/frp/releases), 下载对应平台的软件并解压


## 通过ssh访问
1. 将`frps`和`frps.ini`放到公网服务器上, 修改`frps.ini`配置如下，其中         `bind_port`是`frp`服务所监听端口
    ```
    [common]
    bind_port = 7000
    ```

2. 启动服务, 启动成功会打印出`Start frps success`
   ```shell
   ./frps -c ./frps.ini 
   ```

3. 在本地客户端修改`frpc.ini`文件，将`server_port`修改为公网服务器ip
   ```shell
    [common]
    server_addr = x.x.x.x # 公网ip
    server_port = 7000

    [ssh]
    type = tcp
    local_ip = 127.0.0.1
    local_port = 22
    remote_port = 6000
   ```

4. 启动客户端
   ```shell
   ./frpc -c ./frpc.ini
   ```

## web服务代理
经常我们希望内网的web服务能够在公网被访问，而本机通常是没有公网ip的，我们需要配置下frp的http访问端口

1. 修改 `frps.ini`, `vhost_http_port`为公网http服务访问端口
    ```
    [common]
    bind_port = 7000
    vhost_http_port = 8088
    ```

2. 修改`frpc.ini`, `local_port`为本机服务所在端口，而`custom_domains`为解析到公网ip端口8088上的域名，也可以通过 `nginx`做一层反向代理
    ```shell
    [common]
    server_addr = x.x.x.x
    server_port = 7000

    [web]
    type = http
    local_port = 80
    custom_domains = www.xxx.com
    ```

3. 启动服务和客户端，浏览器访问 `http://www.xxx.com:8088` 即可访问到处于内网机器上的 web 服务