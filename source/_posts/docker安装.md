---
title: CentOS安装docker教程
date: 2019-11-09 19:54:48
updated: 2019-11-09 19:54:48
tags:
---

Docker 分为 CE 和 EE 两大版本。CE 即社区版（免费，支持周期 7 个月），EE 即企业版，强调安全，付费使用，支持周期 24 个月。这里只介绍 Docker CE 在 CentOS下的安装。

<!-- more -->


## 安装docker

1. yum安装docker依赖包
    ```shell
    sudo yum install -y yum-utils device-mapper-persistent-data lvm2
    ```

2. 配置 docker-ce 仓库
    ```shell
    sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    ```
3. 安装 docker-ce
   ```shell
   sudo yum install docker-ce
   ```

4. 配置docker开机启动并启动docker服务
    ```shell
    sudo systemctl enable docker.service
    sudo systemctl start docker.service
    ```

5. 此时docker已经启动，我们需要将当前用户添加至 `docker`用户组，这样就可以免`sudo`
    ```shell
    sudo usermod -aG docker $(whoami)
    ```

6. 使用执行完成后执行 `cat /etc/group` 检查下创建是否有效

7. 退出当前用户重新登录以便权限配置生效，或者重启 `docker-daemon`

    ```shell
    sudo systemctl restart docker
    ```

8. 如果出现提示`dial unix /var/run/docker.sock: connect: permission denied` ，说明需要对当前用户授权

    ```shell
    sudo chmod a+rw /var/run/docker.sock
    ```

## 安装 docker-compose

1. 安装社区企业Linux附加包 `epel`
   ```shell
   sudo yum install -y epel-release 
   ```

2. 安装 `pip`
   ```shell
   sudo yum install -y python-pip
   ```
4. 更新`python`环境
   ```shell
   sudo yum upgrade python*
   ```

5. 通过`pip`安装docker-compose, 安装成功后`docker-compose version`查看版本信息
   ```
   pip install docker-compose
   ```




