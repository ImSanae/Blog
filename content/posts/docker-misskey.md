---
title: "Docker 搭建 Misskey 实例"
tags: ["Docker", "Misskey", "Fediverse"]
date: 2023-06-08  
draft: false
---

{{< notice tip >}} 本文中使用的 Misskey 版本号为 v13 {{< /notice >}}

## 前言

本文是笔者写的自建 Misskey ~~踩坑记录~~指南。之前搭建过 Mastodon 但是 Mastodon 的配置和服务器要求过高，出于经济和实用方面考虑而放弃了 Mastodon。之后在某位朋友的推荐下了解到了 Misskey，同样是基于 [ActivityPub](https://en.wikipedia.org/wiki/ActivityPub) 协议属于茫茫 [Fediverse](https://en.wikipedia.org/wiki/Fediverse) 宇宙中的一员，在 Misskey 实例上的用户也可以关注茫茫宇宙中的其他人，Misskey 相比来说比较轻量且 UI 设计可爱。

不过 Misskey 的文档翻译不及时有的严重滞后甚至不少内容只有日文版本，而且 [Issue](https://github.com/misskey-dev/misskey/issues) 基本上都是日文。

## 准备工作

- 需要有一台 VPS，本文选择的系统是 Debian11
- 需要一个域名（之前未参与过联邦宇宙，否则后续实例间的通讯可能会出现签名问题）

### 安装 Docker 和 Docker-compose

这里第一步使用了官方提供的一键脚本安装 Docker。如果你对此感到不放心，请通过[官网步骤](https://docs.docker.com/engine/install/debian/)自行安装。

{{< notice note >}} 注意：官方的安装命令未包含 `docker-compose` 软件包。

`sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose docker-compose-plugin` 输入命令以安装所有需要的软件包。 {{< /notice >}}

```bash
$ bash <(curl -L https://get.docker.com/)
$ sudo curl -L "https://github.com/docker/compose/releases/download/latest/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```

### 获取 Misskey

{{< notice note >}} 鉴于大部分人没有修改源代码的需求，我们不推荐使用 Git 拉取源码自行 Build 镜像。

我们选择直接使用 Build 好的镜像进行搭建。
{{< /notice >}}

```bash
$ mkdir -p /home/Misskey && cd /home/Misskey
```

用你习惯的编辑器创建 docker-compose.yml 文件并写入以下内容

```yml
version: "3"

services:
  web:
    image: misskey/misskey:latest
    container_name: misskey
    restart: always
    links:
      - db
      - redis
    depends_on:
      - db
      - redis
    networks:
      - misskey
    ports:
      - "127.0.0.1:3000:3000" # 如果遇到端口被占用请将前面的 3000 改为其他端口
    volumes:
      - ./.config:/misskey/.config:ro
      - ./files:/misskey/files # 如果您使用外部储存媒体文件可以将此行删去

  redis:
    image: redis:alpine
    container_name: misskey_redis
    restart: always
    networks:
      - misskey
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
    volumes:
      - ./redis:/data

  db:
    image: postgres:15-alpine
    container_name: misskey_db
    restart: always
    networks:
      - misskey
    env_file:
      - .config/docker.env
    volumes:
      - ./postgres15:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD', 'pg_isready', '-U', 'postgres']

networks:
  misskey:
    external: false
```

### 编辑配置

请在 Misskey 目录下进行操作

```bash
$ mkdir -p .config && cd .config
$ wget https://github.com/misskey-dev/misskey/raw/develop/.config/docker_example.yml && cp docker_example.yml default.yml
$ wget https://github.com/misskey-dev/misskey/raw/develop/.config/docker_example.env && cp docker_example.env docker.env
```

下一步是编辑具体的配置信息，请使用你喜欢或熟悉的文本编辑器实现

#### default.yml

这个是站点配置文件，参考样例可见[此链接](https://github.com/misskey-dev/misskey/blob/develop/.config/docker_example.yml)，开发组贴心地在每一项之前都提供了介绍。

首先必须要修改的是 URL，请改成自己实例的链接。

{{< notice warning >}} 警告：与其他实例的通讯依赖此配置，一旦实例启动后请不要修改此配置，以免造成不可挽回的错误。

{{< /notice >}}

第二项是 [port](https://github.com/misskey-dev/misskey/blob/develop/.config/docker_example.yml#L31) ，此配置保持默认即可。

![port](/img/Misskey/docker_port.png)

下面是 PostgreSQL 和 Redis 依赖服务的配置，开发组已经将连接的 host 和 port 配置好了，您只需要设置 PostgreSQL 的数据库名、用户名和密码即可。

![PostgreSQL](/img/Misskey/docker_PostgreSQL.png)

{{< notice warning >}}
此部分配置必须与 docker.env 下的保持一致，否则 Misskey 将无法正确启动
{{< /notice >}}

#### docker.env

```env
# 数据库密码
POSTGRES_PASSWORD=ncV42GH17GPpvLbJLNqi9r0RKiJCfNJgy8toqiy1dreavGYNWhPTmjGgEqeJC5G8
# 数据库用户
POSTGRES_USER=misskey
# 数据库名
POSTGRES_DB=Misskey
```

### 启动服务

```bash 
$ docker-compose pull && docker-compose up -d
```

可以通过以下命令查看 Docker 容器是否正常

```shell
$ docker ps -a | grep misskey
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

Caddyfile 配置参考 (请将 misskey.example 修改为你实际使用的域名)

```yml
misskey.example {
        reverse_proxy localhost:3000
        encode zstd gzip
}
```

配置好后，重载 Caddy 配置，等待 SSL 证书自动签发完成后，打开浏览器输入实例域名就可以看到你搭建好的 Misskey 实例啦。

{{< notice note >}}
如果您使用 Cloudflare CDN 作为代理，请在 Cloudflare Dash 域名下的规则页面 -> Configuration Rules 创建一条规则为 Misskey 实例关闭 Auto Minify，否则 Misskey 将无法工作
{{< /notice >}}

## 写在后面

Misskey 或者说整个 Fediverse 宇宙十分广阔，愿我们能在茫茫宇宙中相遇。

- - -
## 参考资料

- [Caddy Server](https://caddyserver.com/docs/)
- [Docker Install](https://docs.docker.com/engine/install/debian/)
- [Create Misskey instance with Docker Compose](https://misskey-hub.net/en/docs/install/docker.html)