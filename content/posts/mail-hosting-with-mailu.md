---
title: "Mailu 搭建邮局的不完全指南"
tags: ["Docker", "Mail"]
date: 2023-02-03
update: 2023-02-05
draft: true
---

[Mailu](https://mailu.io) 是一个开源的邮件服务，需要使用 Docker 安装。

## 系统要求

Mailu 基于 Docker，所以几乎可以在所有发行版上安装使用。请确保您的服务器有 1 GB 或更大的内存，并拥有独立的 IP 地址。

您需要确保您 vps 的 25 端口是开放的且未被占用，可以使用 Telnet 命令测试是否开放：

```bash
$ telnet smtp.aol.com 25
```
如果出现以下信息，则说明 25 端口是开放的：

```
Trying 87.248.97.31...
Connected to smtp.aol.g03.yahoodns.net.
Escape character is '^]'.
220 smtp.mail.yahoo.com ESMTP ready
```

否则说明您的 25 端口被服务商屏蔽，您需要向服务器申请开放 25 端口（建议在申请前查询 IP 是否被列入黑名单，被列入黑名单的 IP 发件容易进入垃圾箱）

本文使用的系统为 Debian11。

## 安装 Mailu

### 设置 DNS 记录

DNS 记录的设置分为两部分，此部分为搭建邮局前设置（请将域名替换为您自己的域名）

* 将域名 mail.sanae.im 设置 A / AAAA 记录，解析到服务器 IP
* 将 sanae.im 设置 MX 记录，优先级为 10，解析值为 mail.sanae.im

然后您需要在您的 VPS 服务商处设置 rDNS 记录，将 IP 解析到 mail.sanae.im（如果您的服务商不支持手动设置请发工单询问，无 rDNS 记录会增加发出的邮件进入垃圾箱的概率）

设置好相关 DNS 记录后就可以开始部署 Mailu 邮局服务了。

### 安装 Docker 和 Docker-Compose

这里使用了官方提供的一键脚本安装 Docker。如果你对此感到不放心，请通过[官网步骤](https://docs.docker.com/engine/install/debian/)自行安装。

```bash
$ bash <(curl -L https://get.docker.com/)
$ sudo curl -L "https://github.com/docker/compose/releases/download/latest/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
$ sudo chmod +x /usr/local/bin/docker-compose
```

### 生成 Mailu 配置文件

Mailu 官方提供了自动生成配置文件的网页，我们使用[这个网页](https://setup.mailu.io)来生成相关配置文件（编写本文时 Mailu 最新正式版本号为 v1.9）

#### Step1

第一步是 Docker 管理方式，请选择 Compose，Stack 模式已被弃用

![Step1](/img/Mailu/mailu1.png)

#### Step2

第二步需要设置路径和相关域名消息，请按照下图填写并替换为您自己的域名

* Mail storage path 请根据个人偏好填写（此处选择的目录为 `/home/Mailu`）
* Main mail domain and server display name 请填写您邮箱的后缀地址
* Postmaster local part 请填写您配置邮局的管理员邮箱地址（推荐填写 i / hi）
* Choose how you wish to handle security [TLS certificates](https://mailu.io/1.9/compose/setup.html#tls-certificates) 推荐选择 mail，如果您只想一键为 Mailu 配置 TLS 则可以选择 letsencrypt（此选项会在每次证书丢失重装后重新申请证书，请注意 Let's Encrypt 的速率限制）
* Authentication rate limit / Authentication rate limit per user / Outgoing message rate limit 请根据您的需求进行修改，不建议设置过高
* Opt-out of statistics 是否不上报相关统计数据请自行选择，从隐私角度考虑勾选以退出统计
* Website name 可根据您的偏好进行修改
* Linked Website URL 修改为 https://mail.sanae.im （请将域名替换为您自己的域名）
* Enable the admin UI (and path to the admin UI) 请勾选启用 admin UI（path 路径可根据个人偏好进行修改）

![Step2](/img/Mailu/mailu2.png)

#### Step3

第三步是选择网页邮箱的面板，此步骤可选择 none / Roundcube / Rainloop，请根据个人偏好选择。我选择的是 Roudcube。

下面的三个选项分别是杀毒服务、WebDAV 和邮件代收，请注意杀毒服务需要额外至少 1 GB 的内存，如果需要勾选先请确保您的服务器内存足够运行 `ClamAV`，其他两项请根据自己的偏好勾选。

![Step3](/img/Mailu/mailu3.png)

#### Step4

第四步需要配置 IP 与主机名

* IPv4 listen address 请填写您服务器对外的 IPv4 地址
* Enable IPv6 请谨慎勾选（如果您的服务器拥有公网 IPv6 且为 Docker 启用 IPv6 则推荐勾选以启用 IPv6）
* IPv6 listen address 如果您上一步选择了 Enable IPv6 请填写您服务器对外的 IPv6 地址
* Enable an internal DNS resolver (unbound) 建议勾选
* Public hostnames 填写 mail.sanae.im （请将域名替换为您自己的域名）

![Step4](/img/Mailu/mailu4.png)

#### Step5

第五步是选择数据库，此处选择最简单的 sqlit 即可。您也可以根据偏好选择 postgresql / mysql 

![Step5](/img/Mailu/mailu5.png)

### 修改 Mailu 配置文件

配置完成后点击 Setup Mailu，就可以得到自动生成好的配置文件了，请先将 docker-compose.yml 和 mailu.env 下载到本地进行修改

推荐添加为容器添加 `container_name` 配置便于后续使用 docker 相关命令

如果您在第二步 `Choose how you wish to handle security` 中选择的是 `letsencrypt` ，请不要修改 docker-compose.yml 中 front 部分的 ports 配置（此选项会占用 80 / 443 端口）。

此时可以将 docker-compose.yml 和 mailu.env 上传到服务器对应目录（目录为生成 Mailu 配置文件中第二步 Mail Storage path 填写的目录）

```bash
$ mkdir /home/Mailu
```

修改后的 docker-compose 配置文件如下（请根据自行生成的配置文件进行修改，此配置文件仅供参考）

```yml
version: '2.2'

services:

  # External dependencies
  redis:
    image: redis:7-alpine
    container_name: mailu_redis
    restart: always
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
    volumes:
      - "/home/Mailu/redis:/data"

  # Core services
  front:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}nginx:${MAILU_VERSION:-1.9}
    restart: always
    container_name: mailu_front
    env_file: mailu.env
    logging:
      driver: json-file
    ports:
      - "1.1.1.1:88:80"
      - "1.1.1.1:25:25"
      - "2606:4700:4700::1111:25:25"
      - "1.1.1.1:465:465"
      - "2606:4700:4700::1111:465:465"
      - "1.1.1.1:587:587"
      - "2606:4700:4700::1111:587:587"
      - "1.1.1.1:110:110"
      - "2606:4700:4700::1111:110:110"
      - "1.1.1.1:995:995"
      - "2606:4700:4700::1111:995:995"
      - "1.1.1.1:143:143"
      - "2606:4700:4700::1111:143:143"
      - "1.1.1.1:993:993"
      - "2606:4700:4700::1111:993:993"
    volumes:
      - "/home/Mailu/certs:/certs"
      - "/home/Mailu/overrides/nginx:/overrides:ro"

  admin:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}admin:${MAILU_VERSION:-1.9}
    restart: always
    container_name: mailu_admin
    env_file: mailu.env
    volumes:
      - "/home/Mailu/data:/data"
      - "/home/Mailu/dkim:/dkim"
    depends_on:
      - redis

  imap:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}dovecot:${MAILU_VERSION:-1.9}
    restart: always
    container_name: mailu_imap
    env_file: mailu.env
    volumes:
      - "/home/Mailu/mail:/mail"
      - "/home/Mailu/overrides/dovecot:/overrides:ro"
    depends_on:
      - front

  smtp:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}postfix:${MAILU_VERSION:-1.9}
    restart: always
    container_name: mailu_smtp
    env_file: mailu.env
    volumes:
      - "/home/Mailu/mailqueue:/queue"
      - "/home/Mailu/overrides/postfix:/overrides:ro"
    depends_on:
      - front

  antispam:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}rspamd:${MAILU_VERSION:-1.9}
    hostname: antispam
    container_name: mailu_antispam
    restart: always
    env_file: mailu.env
    volumes:
      - "/home/Mailu/filter:/var/lib/rspamd"
      - "/home/Mailu/overrides/rspamd:/etc/rspamd/override.d:ro"
    depends_on:
      - front

  # Optional services



  # Webmail
  webmail:
    image: ${DOCKER_ORG:-mailu}/${DOCKER_PREFIX:-}roundcube:${MAILU_VERSION:-1.9}
    restart: always
    container_name: mailu_webmail
    env_file: mailu.env
    volumes:
      - "/home/Mailu/webmail:/data"
      - "/home/Mailu/overrides/roundcube:/overrides:ro"
    depends_on:
      - imap
      
networks:
  default:
    enable_ipv6: true
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 192.168.203.0/24
        - subnet: fd56:c40b:eb8e:beef::/64
```
{{< notice note >}} 注意：编写本文是 Mailu 的最新版本号为 v1.9，如您使用的版本号高于此版本请优先参考官方文档 {{< /notice >}}

### 配置 TLS 

如果您在第二步 `Choose how you wish to handle security` 中选择的是 letsencrypt ，请跳过此部分内容。

#### 获取 TLS 证书

本文使用的是 Caddy 以配置反向代理和获取 TLS 证书，如需自定义编译 Caddy 推荐参考 [烧饼博客](https://u.sb/xcaddy/) 和 [双猫CC](https://catcat.cc/post/h9bti/)。

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
mail.example.org {
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
		Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" # HSTS
		Referrer-Policy strict-origin-when-cross-origin
		X-Permitted-Cross-Domain-Policies "none"
		X-Frame-Options SAMEORIGIN
		X-Content-Type-Options nosniff
		X-XSS-Protection "1; mode=block"
		-Server
	}

	reverse_proxy localhost:88 # 此处为 Mailu 的配置文件 docker-compose.yml 中 front 部分 ports 中 80 端口的映射端口
}
```

然后检查 Caddy 配置

```bash
$ caddy validate --config /etc/caddy/Caddyfile
```

输出类似如下内容（没有出现 Error）则表示配置正确：

```log
2023/01/31 04:37:47.552 INFO    using provided configuration    {"config_file": "/etc/caddy/Caddyfile", "config_adapter": ""}
2023/01/31 04:37:47.585 INFO    tls.cache.maintenance   started background certificate maintenance      {"cache": "0xc0002e4620"}
2023/01/31 04:37:47.586 INFO    http    enabling automatic HTTP->HTTPS redirects        {"server_name": "srv0"}
2023/01/31 04:37:47.591 INFO    tls.cache.maintenance   stopped background certificate maintenance      {"cache": "0xc0002e4620"}
Valid configuration
```

可使用以下命令美化你的 Caddyfile 文件

```bash
$ caddy fmt /etc/caddy/Caddyfile --overwrite
```

最后重启 Caddy

```bash
$ systemctl restart caddy
```

### 配置证书

Caddy 的默认证书目录为 `/var/lib/caddy/.local/share/caddy/certificates/` 

执行（此处使用绝对路径,请将路径替换为您的证书路径）

```bash
cp /var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/mail.sanae.im/mail.sanae.im.crt /home/Mailu/certs/cert.pem # 公钥
cp /var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/mail.sanae.im/mail.sanae.im.key /home/Mailu/certs/key.pem # 私钥
```

### 运行 Mailu 服务器

通过 SSH 连接上您的服务器，执行：

```bash
$ cd /home/Mailu
$ docker-compose -p mailu up -d
```

初次使用时会拉取多个 Docker 镜像，请耐心等待。可以通过以下命令查看 Docker 容器是否正常：

```shell
$ docker ps -a | grep mailu
```

若您本地已经拉取过 Mailu 相关镜像但不是[最新的版本](https://github.com/Mailu/Mailu/releases)，请执行：

```shell
docker-compose pull && docker-compose -p mailu up -d
```

如果 mail_front 容器的状态为 healthy 即为正常，此时可以为 Mailu 创建一个管理员账户：

```bash
$ docker-compose -p mailu exec admin flask mailu admin hi sanae.im 'PASSWORD'
```

此命令创建了一位用户名为 hi@sanae.im 密码为 PASSWORD 的管理员用户。请替换为您自己设置的值，为防止暴力破解推荐设置包含特殊字符的复杂密码（推荐使用 [1PassWord](https://1password.com/password-generator/) 提供的密码生成器生成安全复杂的密码）。

## 配置 Mailu

![login](/img/Mailu/mailu_login.png)

此时打开浏览器访问您设置的邮局服务地址就可以跳转到 Mailu 登录面板了，请输入上一步创建的管理员账户信息登录（可通过右上角切换成中文）。

![domain](/img/Mailu/mailu_domains.png)

点击左侧的邮件域，可管理所添加的邮箱域名。点击右上角的 New Domain （新域） 可添加新域名；在操作栏选择详细信息可进入域名详情页；在管理栏点击用户可查询当前此域名下创建的用户以及添加新用户。

![users](/img/Mailu/mailu_users.png)

在用户列表页可为用户设置前缀、密码、显示名称、说明、是否允许 IMAP / POP3，请根据个人需求进行配置，最后请点击保存。

![domain](/img/Mailu/mailu_domainconfig.png)

在域详细信息页，请先点击右上角的重新生成密钥再点击确定，即可生成 DKIM 记录，然后请在您的 DNS 服务商处根据提示添加相关 DNS 记录。全部添加完成后就完成了 Mailu 的搭建，应该可以正常收发信了。

注意：如果您勾选了 Enable IPv6，请将 spf 记录修改为 `v=spf1 ip4:your_ipv4 ip6:your_ipv6 -all`（允许 `your_ipv4` 和 `your_ipv6` 传出邮件，其他的硬拒绝）

## 邮件测试

使用 [Newsletters spam test by mail-tester](https://www.mail-tester.com/)，请使用您的邮箱地址向其发送一封测试邮件。如果您的配置无误且 IP 不在黑名单上，应该可以获得 10/10 分。

- - -
## 参考资料

- [Caddy Server](https://caddyserver.com/docs/)
- [Cloudflare DNS-DKIM-Record](https://www.cloudflare.com/learning/dns/dns-records/dns-dkim-record/)
- [Cloudflare DNS-DMARC-Record](https://www.cloudflare.com/learning/dns/dns-records/dns-dmarc-record/)
- [Cloudflare DNS-SPF-Record](https://www.cloudflare.com/learning/dns/dns-records/dns-spf-record/)
- [Docker Install](https://docs.docker.com/engine/install/debian/)
- [Mailu Docker-Compose Setup](https://mailu.io/1.9/compose/setup.html)
- [Mailu Using an external reverse proxy](https://mailu.io/1.9/reverse.html)