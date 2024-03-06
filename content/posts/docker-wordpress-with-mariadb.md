---
title: "Docker 搭建 WordPress"
tags: ["Docker", "WordPress", "MariaDB"]
date: 2023-01-17
update: 2023-01-31
draft: false
---

## 前言

大部分用 Docker 搭建 WordPress 的方法都是用 MySQL 数据库。

MairaDB 是 MySQL 的一个分支，主要由开源社区在维护，采用 GPL 授权许可。 MariaDB 的目的是完全兼容 MySQL，而且 MairaDB 的性能完全不输给 MySQL。

对于大多数应用程序和目的，尤其是 WordPress，用 MariaDB 替换 MySQL 基本上不存在兼容问题。

~~所以 WordPress 什么时候能原生支持 ProgreSQL~~

## 为什么开始使用 Docker 搭建 WordPress

**优点：**
- 搭建、升级方便，命令简单，不用自行配置环境。
- 迁移和备份操作方便。

**缺点：**
- 修改相关配置不够方便
- 需要学习 Docker 相关命令

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
$ mkdir -p /home/WordPress
$ cd /home/WordPress
```

然后用你习惯的编辑器创建 docker-compose.yml 文件并写入以下内容(请自行生成数据库密码)

```yml
version: '3'

services:
  db:
    restart: always
    image: mariadb
    container_name: wordpress_db
    networks:
      - wordpress_networks
    volumes:
      - ./db:/var/lib/mysql
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -P 3306 -p设置的数据库密码 | grep 'mysqld is alive' || exit 1"]
      interval: 15s
      retries: 4
    environment:
      - MARIADB_ROOT_PASSWORD=
      - MARIADB_DATABASE=WordPress
      - MARIADB_USER=wordpress
      - MARIADB_PASSWORD=
    logging:
      driver: "json-file"
      options:
        max-size: "10m"

  wordpress:
    restart: always
    image: wordpress:latest
    container_name: wordpress_service
    networks:
      - wordpress_networks
    ports:
      - "127.0.0.1:888:80"
    volumes:
      - ./html:/var/www/html
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80"]
      interval: 15s
      retries: 4
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: WordPress
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: 
    depends_on:
      - db

networks:
  wordpress_networks:
    external: false
```

保存退出后输入以下命令

```bash
$ docker-compose up -d
```

可以通过以下命令查看 Docker 容器是否正常

```shell
$ docker ps -a ｜ grep wordpress
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

Caddyfile 配置文件 (仅供参考，请将 blog.example.org 修改为你设置的域名)

```yml
blog.example.org {
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

	reverse_proxy localhost:888
}
```

然后检查 Caddy 配置

```bash
$ caddy validate --config /etc/caddy/Caddyfile
```

输出如下内容则表示配置正确：

```log
2023/01/17 08:48:27.552 INFO    using provided configuration    {"config_file": "/etc/caddy/Caddyfile", "config_adapter": ""}
2023/01/17 08:48:27.585 INFO    tls.cache.maintenance   started background certificate maintenance      {"cache": "0xc0002e4620"}
2023/01/17 08:48:27.586 INFO    http    enabling automatic HTTP->HTTPS redirects        {"server_name": "srv0"}
2023/01/17 08:48:27.591 INFO    tls.cache.maintenance   stopped background certificate maintenance      {"cache": "0xc0002e4620"}
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

等待一会儿后打开浏览器就可以查看 https://blog.example.org 即可看到 SSL 证书已经自动部署，然后就可以初始化你的 WordPress 站点了

### 修改 uploads.ini 配置以便上传主题

在 WordPress 目前下创建 `uploads.ini` 文件并写入以下配置

```ini
file_uploads = on
upload_max_filesize = 100M
post_max_size = 100M
```

{{< notice tip >}} 提示：使用 Cloudflare Free 计划并使用 CDN 无法上传超过 100M 大小的主题 ~~(话说有这么大的主题吗）~~ {{</ notice >}}

然后修改 docker-compose.yml，在 wordpress 容器 volumes 新增下列配置

```yml
- ./uploads.ini:/usr/local/etc/php/conf.d/uploads.ini
```

然后重启容器即可

```bash
$ docker-compose down && docker-compose up -d
```

- - -
## 参考资料

- [Caddy Server](https://caddyserver.com/docs/)
- [Docker Install](https://docs.docker.com/engine/install/debian/)
- [Wordpress Docker won't increase upload limit](https://stackoverflow.com/questions/42983276/wordpress-docker-wont-increase-upload-limit)