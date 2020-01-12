---
title: github加速
date: 2020-01-12 21:12:54
tags:
---

国内访问GitHub的速度实在感人，经常clone不下来，push不上去，大大影响了程序员的工作效率。


## https 修改hosts


GitHub在国内访问速度慢的问题原因有很多，有很大一部分原因是是GitHub的CDN加速的域名遭到dns污染。
这个可以通过修改`hosts`文件解决。

```
// vim /etc/hosts  
// add 
192.30.253.112 http://github.com
151.101.184.133 http://assets-cdn.github.com
151.101.185.194 http://github.global.ssl.fastly.net
```

## ssh 设置代理 ~/.ssh/config
当然，修改`hosts`只能提升`https`方式，我们通常是用公钥访问，走的是`ssh`协议，这样访问还是很慢。如果有梯子就可以提升`ssh`协议的访问速度。


```shell
// vim ~/.ssh/config
// add
Host github.com
User git
ProxyCommand /usr/bin/nc -x 127.0.0.1:1080 %h %p  # 1080 是shdowsocks代理端口
IdentityFile ~/.ssh/id_rsa
```