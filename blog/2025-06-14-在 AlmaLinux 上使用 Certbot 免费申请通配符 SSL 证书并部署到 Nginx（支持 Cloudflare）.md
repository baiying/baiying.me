---
slug: certbot-apply-free-ssl-certificate-for-nginx-on-almalinux
title: 🛡 在 AlmaLinux 上使用 Certbot 免费申请通配符 SSL 证书并部署到 Nginx（支持 Cloudflare）
authors: baiying
tags: [certbot, free ssl certificate, nginx, almalinux, website]
---

# 🛡 在 AlmaLinux 上使用 Certbot 免费申请通配符 SSL 证书并部署到 Nginx（支持 Cloudflare）

在构建网站或部署应用时，为所有子域启用 HTTPS 是常见的需求。本文将详细介绍如何在 AlmaLinux 上使用 Certbot + DNS API 方式免费申请 Let's Encrypt 的通配符 SSL 证书，并在 Nginx 中部署使用。以 Cloudflare 为 DNS 服务商为例。

---

## 📌 一、环境准备

* 操作系统：AlmaLinux 8 或 9
* DNS 服务商：Cloudflare（支持 API）
* 域名：`example.com`（你自己的）
* 安装方式：使用 Certbot + Cloudflare 插件 + DNS-01 验证

---

## 📥 二、安装 Certbot 及 Cloudflare 插件

```bash
# 安装 epel-release 和 certbot
sudo dnf install epel-release -y
sudo dnf install certbot -y

# 安装 Cloudflare 插件
sudo dnf install python3-certbot-dns-cloudflare -y
```

---

## 🔐 三、创建 Cloudflare API Token

1. 登录 Cloudflare 控制台：[https://dash.cloudflare.com/](https://dash.cloudflare.com/)

2. 点击右上角头像 → **My Profile**

3. 在左侧菜单选择 **API Tokens**

4. 点击 **Create Token**

5. 选择模板：`Edit zone DNS`

6. 权限配置如下：

   | 权限        | 范围   |
   | --------- | ---- |
   | Zone.Zone | Read |
   | Zone.DNS  | Edit |

7. 限制区域：选择你的域名，如 `example.com`

8. 创建后复制 API Token，**只显示一次**

---

## 🗝 四、配置凭据文件

```bash
# 创建凭据文件
mkdir -p /root/.secrets
nano /root/.secrets/cloudflare.ini
```

内容如下：

```ini
dns_cloudflare_api_token = your_cloudflare_token_here
```

设置权限：

```bash
chmod 600 /root/.secrets/cloudflare.ini
```

---

## 📄 五、申请通配符 SSL 证书

```bash
certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  -d example.com -d "*.example.com"
```

如果成功，你会看到输出类似：

```
Congratulations! Your certificate and chain have been saved at:
/etc/letsencrypt/live/example.com/fullchain.pem
```

---

## 🌐 六、在 Nginx 中部署 SSL 证书

编辑你的 Nginx 配置：

```nginx
server {
    listen 443 ssl http2;
    server_name example.com *.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        root /var/www/html;
        index index.html index.htm;
    }
}

server {
    listen 80;
    server_name example.com *.example.com;
    return 301 https://$host$request_uri;
}
```

重载 Nginx：

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 🔁 七、设置自动续期及 Nginx 自动重载

测试续期命令：

```bash
certbot renew --dry-run
```

添加自动续期并重载 Nginx 的计划任务：

```bash
sudo crontab -e
```

添加：

```bash
0 3 * * * certbot renew --quiet --post-hook "systemctl reload nginx"
```

---

## 🧠 附录：常见问题 FAQ

### ❓ `--dns-cloudflare-credentials` 报错？

请确保你已安装插件：

```bash
sudo dnf install python3-certbot-dns-cloudflare -y
```

### ❓ 证书续期后不会自动生效？

加上 `--post-hook` 自动重载 Nginx，或手动 reload。

### ❓ 多个子域如何使用？

通配符证书支持任意子域，如 `app.example.com`、`api.example.com`，但 **不支持多级子域**（如 `a.b.example.com`）。

---

## ✅ 总结

| 步骤       | 工具                       |
| -------- | ------------------------ |
| 证书申请     | Certbot + DNS-01         |
| DNS 验证   | Cloudflare API           |
| 通配符支持    | `*.example.com`          |
| HTTPS 部署 | Nginx                    |
| 自动续期     | Certbot + cron/post-hook |

---

通过 Certbot 配合 DNS 插件，你可以高效、安全、免费地为你的整站（包括所有子域）配置 HTTPS，助力网站性能和安全性提升。

---

如你还使用了 Docker、Node.js、或自动化部署方案，也可以基于这个证书方案扩展，欢迎留言交流。
