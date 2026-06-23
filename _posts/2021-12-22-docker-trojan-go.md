---
layout: post
title: "Docker 搭建 Trojan-Go 服务"
date: 2021-12-22 00:00:00 +0800
categories: DevOps
tags: [Docker, Trojan-Go, 实践]
---
# docker搭建trojan-go服务

Tags: devOps, 实践

- 参考文章
    - [Trojan-go 帮助文档](https://p4gefau1t.github.io/trojan-go/basic/config/)（内含trojan-go原理）
    - [Qv2ray Trojan-go安装配置](https://www.osuix.com/2021/01/12/qv2ray-trojan-go-%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE-for-win10-macos/)
    - [Qv2ray参数设置](https://qv2ray.net/lang/zh/manual/general.html)

# Step1. 准备服务器

需要一台非大陆的，且未被GWF封禁的服务器，已经准备好的跳过此步骤。

以下以购买[搬瓦工vps](https://bandwagonhost.com/aff.php?aff=68731)为例：

1. 搬瓦工购买地址：[搬瓦工(https://bandwagonhost.com](https://bandwagonhost.com/aff.php?aff=68731))，网上有优惠券的兑换码，可以搜一下
2. 搬瓦工还有个地址[搬瓦工(https://bwh81.net)](https://bwh81.net/)，上面访问不了可访问这个，还不清楚有什么区别![](markdowns/pics/devOps/docker_trojan-go/docker_trojan-go_vps_step1.png)
    
3. 购买后vps的ip和ssh端口、初始密码，会通过邮件方式通知，也可以登录web端查看或修改![](markdowns/pics/devOps/docker_trojan-go/docker_trojan-go_vps_step2.png) ![](markdowns/pics/devOps/docker_trojan-go/docker_trojan-go_vps_step3.png)
    
4. 开防火墙和开放端口
    
    搬瓦工新购买的vps防火墙服务是没有启用的
    
    ```bash
    systemctl status firewalld # 查看防火墙状态
    systemctl enable firewalld # 设置开机启动防火墙
    systemctl start firewalld  # 启动防火墙
    
    firewall-cmd --list-ports # 查看开放的端口
    # 开放端口，需要开放80、443, 和你的ssh连接的端口
    firewall-cmd --add-port=80/tcp --zone=public --permanent
    firewall-cmd --add-port=443/tcp --zone=public --permanent
    firewall-cmd --add-port=port/tcp --zone=public --permanent
    
    firewall-cmd --reload # 重启防火墙
    firewall-cmd --list-ports # 确认端口是否有开放
    ```
    

# Step2. 准备域名

可使用免费域名（没用过），可选[华为域名](https://www.huaweicloud.com/product/domain.html#section-4)，新用户1块1年）

将域名解析到上一步的服务器上。因为是国外的服务器，域名可以不需要备案

# Step3. 准备证书

这里准备了两个网站，freessl.org的xxx.com包含了www解析，而OHTTPS不包含，在申请证书时需注意，是需要包含后面trojan-go绑定的域名的（比如绑定的www.abc.com，使用OHTTPS时就需要明确关联www.abc.com，而不能是abc.com）
## freessl.org

1. 为域名准备证ssl证书，可在[freessl.org](https://freessl.org/)申请，跟着步骤点NEXT就可以了，若最后验证选的DNS验证，需要在域名上添加一条CANME解析![](markdowns/pics/devOps/docker_trojan-go/docker_trojan-go_cert.png)
    

2. 等待生效，直到freessl.org上的“验证”成功

## OHTTPS
去这里申请也可以，支持自动更新证书[OHTTPS - 免费HTTPS证书、自动更新、自动部署](https://ohttps.com)
OHTTPS的需要明确指定关联域名
# Step4. 拉取镜像

1. ssh连上服务器，安装docker
2. 拉取以下docker镜像
    
    ```bash
    # 添加repo 
    # yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    # 安装docker-ce
    # yum install docker-ce -y
    # systemctl start docker
    docker pull python # 用于搭建django服务，trojan-go代理的fallback使用
    docker pull nginx  # 用于代理整个服务器的请求
    docker pull teddysun/trojan-go  # 主程序
    docker pull wbsu2003/lskypro # 图床服务，可以使用
    docker pull mysql:5.7  # 为了搭建lskypro图床，当前lskypro支持的是mysql5.7
    ```
    

# Step5. 创建自定义docker网络

默认的bridge网络，各容器间不能通过hostname互相访问，需要用自定义的网络

```bash
# 创建名为trojan-network的网络
docker network create trojan-network
```

# Step6. 创建django服务

1. 创建适配django的镜像，容器内打算使用8000端口

```bash
# step1
mkdir django && cd django
# step2
cat > Dockerfile << EOF 
FROM docker.io/python:latest
EXPOSE 8000
RUN pip install --upgrade pip && pip install django
WORKDIR ~
EOF
# step3
docker image build -f Dockerfile -t django:v0.1 ./
# step4
docker image ls # 验证django:v0.1镜像是否有成功生成
```

1. 启动django服务

```bash
# step1
docker run -dit --name=django -h django --network trojan-network django:v0.1
# step2
docker exec -it django /bin/bash
# step3
pip install --upgrade pip
pip install django
# step4
mkdir -p ~/django && cd ~/django
django-admin startproject mydjango
cd mydjango
python manage.py migrate
nohup python manage.py runserver 0.0.0.0:8000 &
# step5 退出django
exit
```

# Step7. 启动mysql服务

```bash
# step1  以下my-secret-pw是你自己的mysql密码
docker run -d --name=mysql -h mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw \
--network trojan-network mysql:5.7
# step2 初始化一个数据库
docker exec -it mysql /bin/bash
mysql -u root -p # 回车输入密码
create database lskypro;
# step6 退出mysql
exit
```

# Step8. 启动lskypro服务

```bash
# step1
docker run -d --name=lskypro -h lskypro --network trojan-network wbsu2003/lskypro
```

# Step9. 启动nginx服务

```bash
# step1
docker run -d --name=nginx -h nginx --network trojan-network -p 80:80 -p 443:443 nginx
docker exec -it nginx /bin/bash
# step2 安装vim，因为没有映射数据卷，我们要在容器内编辑配置文件
apt-get update
apt-get install vim
# step3 修改配置文件
vim /etc/nginx/nginx.conf

# 配置文件添加下面的内容(有缩进的部分)
		http { # http是默认有的
				...
				server {
		        # 意思是将80端口的流量代理到兰空图床 
		        listen 80;
		        server_name {你的域名};
		        location / {
		            proxy_pass http://lskypro:80; # 因为都是在trojan-network网络下，可直接通过主机名访问
		        }
		    }
		}
		
		# 配置https代理，将443端口的流量代理到trojan-go
		# 大概意思就是nginx不校验ssl，由服务自己校验
		stream {
		    upstream app {
		        server trojan-go:443;
		    }
		
		    map $ssl_preread_server_name $upstream {
		        default app;
		    }
		
		    server {
		        listen 443;
		        ssl_preread on;
		        proxy_pass $upstream;
		    }
		}

# step4 校验配置并重启服务
nginx -t -c /etc/nginx/nginx.conf
# 如果配置检查提示无法解析trojan-go,需要在/etc/hosts 中增加一个映射
# trojan-go的容器地址  trojan-go

nginx -s reload

# step5 退出nginx容器
exit
```

# Step10. 配置lskypro

配置了nginx，现在可以连接图床进行配置，浏览器输入`http://你的域名` 正常将出现兰空图床的配置界面![](markdowns/pics/devOps/docker_trojan-go/docker_trojan-go_lsky_step1.png)


是的“数据库连接地址”配`mysql`就好了，因为服务在同一个网络下，可通过主机名访问

# Step11. 启动trojan-go服务

终于到主角了

## 11.1 编辑trojan-go配置

```bash
# step1 新建配置文件
mkdir -p /etc/trojan-go/ && cd /etc/trojan-go/
# step2 拷贝证书文件到/etc/trojan-go/目录
# step3 编辑配置文件
vim config.json 

# 完整的配置如下，需要变更的字段有:
# remote_addr: 你的域名
# password: 客户端连接时的密码
# cert: 申请的证书的公钥, .pem格式的
# key： 申请的证书的私钥, .key格式的
{
    "run_type": "server",
    "local_addr": "0.0.0.0",
    "local_port": 443,
    "remote_addr": "{你的域名}",
    "remote_port": 80,
    "password": [
        "password"
    ],
    "log_level": 0,
    "log_file": "/etc/trojan-go/trojan-go.log",
    "disable_http_check": false,
    "udp_timeout": 60,
    "ssl": {
        "verify": true,
        "verify_hostname": true,
        "cert": "/etc/trojan-go/xxx.pem",
        "key": "/etc/trojan-go/xxx.key",
        "key_password": "",
        "cipher": "",
        "curves": "",
        "prefer_server_cipher": false,
        "sni": "",
        "alpn": [
            "http/1.1"
        ],
        "session_ticket": true,
        "reuse_session": true,
        "plain_http_response": "",
        "fallback_addr": "django",
        "fallback_port": 8000,
        "fingerprint": ""
    },
    "tcp": {
        "no_delay": true,
        "keep_alive": true,
        "prefer_ipv4": false
    },
    "mux": {
        "enabled": false,
        "concurrency": 8,
        "idle_timeout": 60
    },
    "router": {
        "enabled": false,
        "bypass": [],
        "proxy": [],
        "block": [],
        "default_policy": "proxy",
        "domain_strategy": "as_is",
        "geoip": "$PROGRAM_DIR$/geoip.dat",
        "geosite": "$PROGRAM_DIR$/geosite.dat"
    },
    "websocket": {
        "enabled": false,
        "path": "",
        "host": ""
    },
    "shadowsocks": {
        "enabled": false,
        "method": "AES-128-GCM",
        "password": ""
    },
    "transport_plugin": {
        "enabled": false,
        "type": "",
        "command": "",
        "option": "",
        "arg": [],
        "env": []
    },
    "forward_proxy": {
        "enabled": false,
        "proxy_addr": "",
        "proxy_port": 0,
        "username": "",
        "password": ""
    },
    "mysql": {
        "enabled": false,
        "server_addr": "localhost",
        "server_port": 3306,
        "database": "",
        "username": "",
        "password": "",
        "check_rate": 60
    },
    "api": {
        "enabled": false,
        "api_addr": "",
        "api_port": 0,
        "ssl": {
            "enabled": false,
            "key": "",
            "cert": "",
            "verify_client": false,
            "client_cert": []
        }
    }
}
```

## 11.2 启动服务

```bash

docker run -d --restart=always --network trojan-network --name trojan-go -h trojan-go \
 -v /etc/trojan-go:/etc/trojan-go teddysun/trojan-go

# 如果服务的容器一直在重启，可以检查/etc/trojan-go/trojan-go.log 日志查看
```

# Step12. 校验服务是否正常

你可以通过使用浏览器访问你的域名`https://your-domain-name.com`来验证。

- 如果工作正常，你的浏览器会显示一个正常的HTTPS保护的Web页面，页面内容与服务器本机80端口上的页面一致。我们的是兰空图床
- 你还可以使用`http://your-domain-name.com:443`验证`fallback_port`工作是否正常。我们的也是兰空图床，不过是https的

# Step13. 客户端，最后一步

[客户端](https://github.com/p4gefau1t/trojan-go)有很多，github上有相关介绍 

![](markdowns/pics/devOps/docker_trojan-go/docker_trojan-go_client.png)

如果使用Qv2ray可参考这个 [Qv2ray Trojan-go安装配置](https://www.osuix.com/2021/01/12/qv2ray-trojan-go-%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE-for-win10-macos/)， 很详细

# 作者总结

学到的新知识：

- docker 容器可通过主机名互相访问，但需要在同一个自定义网络中
- nginx的https代理，如果nginx不校验ssl，配置应该创建stream节点
    
    ```bash
    stream {
    	  upstream app {
    		    server trojan-go:443;
    		}
    		
    		map $ssl_preread_server_name $upstream {
    		    default app;
        }
    		
    	  server {
    		    listen 443;
    		    ssl_preread on;
    		    proxy_pass $upstream;
    		}
    }
    ```