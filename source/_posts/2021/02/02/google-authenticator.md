---
title: 身份认证器简化使用
date: 2021-02-02 13:27:16
tags:
---

很多网站登录都接入了谷歌身份验证器，手机验证虽然很安全，但是也很麻烦，如果能在电脑上获取就能大大简化登录。

<!-- more -->

谷歌身份验证器最早是谷歌为了减少 Gmail 邮箱遭受恶意攻击而推出的两步验证方式，后来被很多网站支持。使用谷歌身份验证器，可以避免比如不久前发生的 Coinbase 短信二次验证导致的安全事故。开启谷歌身份验证之后，登录账户，除了输入用户名和密码，还需要输入谷歌验证器上的动态密码。

谷歌验证器上的动态密码，也称为一次性密码，密码按照时间或使用次数不断动态变化（默认 30 秒变更一次）。它和很多银行发行的动态口令卡类似，可以断网使用，只不过前者是谷歌推出的一个 App，后者是专门的一个硬件。

## Chrome插件

基本上互联网圈都是用的`Chrome`，我们只需要下载`身份认证器`这个插件即可。

打开`Chrome`商店，搜索`身份认证器`

![](https://pic.hupai.pro/img/20210202133142.png)

安装即可。

安装完成后，在账号绑定界面点击扫码即可。

![](https://pic.hupai.pro/img/20210202133252.png)


## 脚本

我们在绑定时看到的二维码，实际上是这样的一串内容,`xx@gmail.com`表示账号名，`DDD`表示的是加密密钥。

```
otpauth://totp/xx@gmail.com?secret=DDD
```

客户端每30秒使用密钥『DDD』和时间戳通过一种『算法』生成一个6位数字的一次性密码，如『684060』。

这个算法叫`OTP`。

1. OTP
OTP(One-Time Password)译为一次性密码，也称动态口令。是使用密码技术实现的在客户端和服务器之间通过共享秘密的一种认证技术，是一种强认证技术，是增强目前静态口令认证的一种非常方便技术手段，是一种重要的双因素认证技术。

1.1 OTP的认证原理
动态口令的基本认证原理是在认证双方共享密钥，也称种子密钥，并使用的同一个种子密钥对某一个事件计数、或时间值、或者是异步挑战数进行密码算法计算，使用的算法有对称算法、HASH、HMAC，之后比较计算值是否一致进行认证。可以做到一次一个动态口令，使用后作废，口令长度通常为6-8个数字，使用方便，与通常的静态口令认证方式类似.

1.3 OTP的实现方式
时间同步(TOTP)
事件同步(HOTP)
挑战/应答(OCRA)

Google身份认证器使用的是`HOTP(HMAC-base On-Time Password)`译为基于HMAC的一次性密码，也称事件同步的动态密码。

如之前的插件就是`javascript`语言的实现。[插件地址](https://github.com/Authenticator-Extension/Authenticator)

`oath-toolkit`这个工具也实现了`OTP`相关算法，它的[官网](https://www.nongnu.org/oath-toolkit/)

`mac`下安装非常简单，`brew install oath-toolkit`。

安装完成后，执行命令`oathtool -b --totp 'private_key'`，注意这个`private_key`是可以通过手机扫绑定时提供的二维码拿到的。

```
# 5YTCB6SOBTYDCOG为密钥
$ oathtool --totp -b 5YTCB6SOBTYDCOG
624309
```

可以在`~/.bashrc`里添加`alias pd='echo "$(oathtool --totp -b 5YTCB6SOBTYDCOG)\c" | pbcopy'`

然后就可以通过运行`pd`，直接将生成的`code`写入系统粘贴板。


