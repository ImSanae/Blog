---
title: "Docker 搭建 NextCloud"
tags: ["Docker", "NextCloud", "ProgreSQL"]
date: 2023-01-17
update: 2023-01-31
draft: false
---

## 前言

Nextcloud is functionally similar to Dropbox, Office 365 or Google Drive. It can be hosted in the cloud or on-premises. It is scalable from home office solutions based on the low cost Raspberry Pi all the way through to full sized data centre solutions that support millions of users. -----[PrivacyTools](https://www.privacytools.io/encrypted-cloud-storage)

{{< notice tip >}} 官方 [Demo](https://try.nextcloud.com) {{< /notice >}}

**优势：**
- 100% 开源
- 全平台客户端支持
- 高速 (取决于连接速度上限)
- 私密 (自己全权管理)
- 稳定 (除非 VPS 服务商跑路)
- [全平台客户端支持](https://nextcloud.com/install/#install-clients)

**缺点：**
- 服务器硬盘容量一般不大 (可挂载外部储存以扩大储存空间)

## 准备工作

- 需要有一台 VPS，本文选择的系统是 Debian11
- 需要准备一个域名，建议长期持有，并配置好解析

### 安装 Docker 和 Docker-compose

这里第一步使用了官方提供的一键脚本安装 Docker。如果你对此感到不放心，请通过[官网步骤](https://docs.docker.com/engine/install/debian/)自行安装。

```bash
$ bash <(curl -L https://get.docker.com/)
$ sudo curl -L "https://github.com/docker/compose/releases/download/latest/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```
### 拉取镜像

```bash
$ mkdir -p /home/NextCloud && cd /home/NextCloud
```

然后用你习惯的编辑器创建 docker-compose.yml 文件并写入以下内容(请自行生成数据库密码)

```yml
version: '3.2'

services:
  db:
    restart: always
    image: postgres:14-alpine
    shm_size: 512mb
    container_name: nextcloud_db
    networks:
      - nextcloud_networks
    healthcheck:
      test: ['CMD', 'pg_isready', '-U', 'postgres']
      interval: 15s
      retries: 12
    volumes:
      - ./postgres:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=
      - POSTGRES_DB=NextCloud
    logging:
      driver: "json-file"
      options:
        max-size: "10m"

  redis:
    restart: always
    image: redis:latest
    container_name: nextcloud_redis
    networks:
      - nextcloud_networks
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 15s
      retries: 12
    volumes:
      - ./redis:/data
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
  
  app:
    restart: always
    image: nextcloud:latest
    container_name: nextcloud_app
    networks:
      - nextcloud_networks
    ports:
      - '127.0.0.1:8848:80'
    depends_on:
      - db
      - redis
    volumes:
      - ./nextcloud:/var/www/html
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_USER=nextcloud
      - POSTGRES_PASSWORD=
      - POSTGRES_DB=NextCloud

networks:
  nextcloud_networks:
    external: false
```

保存退出后输入以下命令

```bash
$ docker-compose up -d
```

可以通过以下命令查看 Docker 容器是否正常

```shell
$ docker ps -a | grep nextcloud
```

### 安装并配置 Caddy

在这一步之前，请记得在您的 DNS 解析服务商设置处添加 A / AAAA Record 解析到您的服务器 IP

* 安装 [Caddy](https://caddyserver.com/docs/install)

```bash
$ sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
$ curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
$ curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
$ sudo apt update
$ sudo apt install caddy
```

* 配置 Caddyfile

```bash
cd /etc/caddy
nano Caddyfile
```

Caddyfile 配置参考 (请将 cloud.example.org 修改为你设置的域名)

```yml
cloud.example.org {
	encode gzip zstd
	tls {
		protocols tls1.3
	}

	log {
		level info
		output file /var/log/caddy/blog.log {
			roll_size 10MB
			roll_keep 10
		}
	}

	header {
		Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
		Referrer-Policy strict-origin-when-cross-origin
		X-Permitted-Cross-Domain-Policies "none"
		X-Frame-Options SAMEORIGIN
		X-Content-Type-Options nosniff
		X-XSS-Protection "1; mode=block"
		-Server
	}

	redir /.well-known/carddav /remote.php/dav 301
	redir /.well-known/caldav /remote.php/dav 301

	@websockets {
		header Connection *Upgrade*
		header Upgrade websocket
	}
	reverse_proxy localhost:8848
}
```

然后检查 Caddy 配置

```bash
$ caddy validate --config /etc/caddy/Caddyfile
```

输出如下内容则表示配置正确：

```log
2023/01/17 05:17:19.552 INFO    using provided configuration    {"config_file": "/etc/caddy/Caddyfile", "config_adapter": ""}
2023/01/17 05:17:19.585 INFO    tls.cache.maintenance   started background certificate maintenance      {"cache": "0xc0002e4620"}
2023/01/17 05:17:19.586 INFO    http    enabling automatic HTTP->HTTPS redirects        {"server_name": "srv0"}
2023/01/17 05:17:19.591 INFO    tls.cache.maintenance   stopped background certificate maintenance      {"cache": "0xc0002e4620"}
Valid configuration
```

可使用以下命令美化你的 Caddyfile 文件

```bash
$ caddy validate --config /etc/caddy/Caddyfile
```

最后重启 Caddy

```bash
$ systemctl restart caddy
```

等待一会儿后打开浏览器就可以查看 https://cloud.example.org 即可看到 SSL 证书已经自动部署，然后就可以初始化你的 NextCloud 服务了

## 安全配置

{{< notice tip >}} 提示：本教程使用的 NextCloud 版本号为 v25，如果您使用的版本号与此版本不相符请优先参考官方文档 {{</ notice >}}

### 启用 Redis

在 `nextcloud/config/config.php` 添加以下内容

```php
  'memcache.distributed' => '\OC\Memcache\Redis',
  'redis' => [
      'host' => 'redis',
      'port' => 6379,
  ],
  'memcache.locking' => '\OC\Memcache\Redis',
```

然后重启容器即可
```bash
$ docker-compose down && docker-compose up -d
```

### 基本设置

点击右上角头像 -> 管理设置 -> 基本设置

#### 后台任务

请选择改用 Cron 模式

```bash
crontab -e
```

选择编辑器为nano或者你习惯的编辑器，随后输入：

```cron
*/5 * * * * docker exec -u www-data nextcloud_app php cron.php
```

#### 配置 SMTP 服务

请先在 个人 -> 个人信息 中填写绑定的电子邮箱，然后根据你的 SMTP 服务商提供的信息输入即可。然后点击下方的 测试并验证电子邮箱设置 ”发送电子邮件“，如果可以收到测试邮件说明配置正确

### 配置外部储存

点击右上角头像 -> 应用 -> 搜索 ` External storage support` 插件安装并启用。然后在转到 管理设置 -> 外部储存，根据你 Object Storage 提供的信息填入相应的配置中，点击保存左边出现绿色标志即为配置正确

### 安全与设置警告解决方法

config.php 目录地址: /home/NextCloud/nextcloud/config/config.php

{{< notice note >}} 注意：需要重新启用 NextCloud 容器才能生效 {{< /notice >}}

```bash
$ docker-compose down && docker-compose up -d
```

#### 你的实例正生成不安全的 URL

在 `config.php` 文件中添加以下内容

```
'overwrite.cli.url' => 'https://cloud.example.org',
```

#### 您的安装没有设置默认的电话区域

在 `config.php` 文件中添加以下内容

```
'default_phone_region' => '(任意国家代码)',
```

#### 你正在通过安全连接访问你的实例，然而你的实例正生成不安全的 URL。这很可能意味着你位于反向代理的后面，覆盖的配置变量没有正确设置。

在 `config.php` 文件中添加以下内容

```
  'trusted_proxies' => 
  array (
    0 => 'caddy',
  ),
```

- - -
## 参考资料

- [Caddy Server](https://caddyserver.com/docs/)
- [Docker Install](https://docs.docker.com/engine/install/debian/)
- [Nextcloud configuration/Memory caching](https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/caching_configuration.html)