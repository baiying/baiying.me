---
slug: Guide-to-Applying-for-SSL-Certificates-on-macOS
title: macOS 申请 SSL 证书及相关问题解决指南
authors: baiying
tags: [certbot, free ssl certificate, nginx, almalinux, website]
---

在网站安全配置中，SSL 证书是保障数据传输安全的重要环节。本文将针对在 macOS 系统下申请 SSL 证书时遇到的常见问题，提供详细的解决方案。

<!-- truncate -->

一、域名提供商无 API 服务时申请证书的方法



如果域名提供商未提供 API 服务，我们可以采用手动 DNS 验证的方式申请通配符 SSL 证书，具体步骤如下：




1.  **安装 Certbot**：通过 Homebrew 安装 Certbot，在终端执行命令`brew install certbot`。


2.  **执行手动申请命令**：在终端运行`certbot certonly --manual --preferred-challenges=dns --server ``https://acme-v02.api.letsencrypt.org/directory`` -d '``example.com``' -d '*.``example.com``'` ，其中`example.com`替换为你的域名。


3.  **添加 TXT 记录**：Certbot 运行后，会要求在域名的 DNS 配置里添加一条 TXT 记录，格式类似`_acme-challenge.example.com``. IN TXT "随机字符串"`。登录域名提供商的管理界面，手动添加该记录。


4.  **等待 DNS 记录生效**：添加完 TXT 记录后，通常需要等待 5 - 30 分钟，让 DNS 记录在全球范围内完成传播。可使用命令`dig ``_acme-challenge.example.com`` TXT +short`验证记录是否生效，当命令返回添加的 TXT 记录值时，即可继续下一步操作。


5.  **完成验证**：确认 TXT 记录生效后，按`Enter`键继续完成验证过程，Certbot 会尝试读取 TXT 记录，证明你对该域名拥有控制权。


6.  **证书生成**：验证通过后，证书会被保存到`/etc/letsencrypt/live/``example.com/`目录中。


### 注意要点&#xA;



*   **记录时效性**：每次申请证书时生成的 TXT 记录都是临时的，仅在当前申请过程中有效。如果申请失败，需要重新生成 TXT 记录。


*   **手动续期**：由于无法通过 API 自动更新 DNS 记录，在证书到期需要续期时，需要重复上述手动操作步骤。


*   **有效期**：Let’s Encrypt 证书的有效期为 90 天，建议在到期前 30 天进行续期操作。


### 替代方案&#xA;

如果手动操作过于繁琐，可考虑以下方案：




*   **临时转移域名**：将域名暂时转移到支持 API 的 DNS 提供商（如 Cloudflare），完成证书申请后再转回。


*   **使用 DNS 解析服务**：添加一个支持 API 的 DNS 解析服务（如 Cloudflare），同时保留原域名提供商，然后使用该服务的 API 进行验证。


二、证书目录访问权限问题解决



申请到的证书通常存放在`/etc/letsencrypt/live/``rubymall.com`目录下，但有时会出现使用 sudo 也无法进入该目录的情况，可参考以下解决方法：


### 1. 检查目录权限&#xA;



*   **查看目录权限**：在终端执行`ls -la /etc/letsencrypt/live/`，正常情况下，该目录的所有者应为`root`，所属组为`wheel`，权限为`drwx------`（仅 root 可读写执行）。


*   **确认当前用户权限**：执行`who am i`记录当前用户名（如`yourusername`），然后执行`groups yourusername`查看用户是否属于`wheel`组。若不在`wheel`组中，需先将用户添加到该组（需 root 权限），执行命令`sudo dscl . -append /Groups/wheel GroupMembership yourusername`，之后重启终端使权限生效。


### 2. 使用 sudo 正确访问目录&#xA;



*   **通过 sudo 查看目录内容**：执行`sudo ls -la /etc/letsencrypt/live/``rubymall.com/`，若提示`Permission denied`，可能是因为`sudo`未加载正确的环境变量，可尝试执行`sudo -i`，输入密码后进入 root shell，此时可直接访问系统目录，再执行`ls -la /etc/letsencrypt/live/``rubymall.com/` 。


*   **复制证书文件到普通目录**：若需要操作证书文件（如复制到网站服务器目录），可在 root shell 中执行`cp -r /etc/letsencrypt/live/``rubymall.com/`` /Users/yourusername/certificates/` ，然后退出 root shell：`exit`，此时可在`/Users/yourusername/certificates/`目录中访问证书文件。


### 3. 权限问题的终极解决方案&#xA;

如果上述方法仍无效，可能是目录的**访问控制列表（ACL）** 或**系统完整性保护（SIP）** 导致：




*   **检查 ACL 设置**：执行`sudo ls -la@ /etc/letsencrypt/live/`，若存在非标准 ACL 规则，可重置为默认，执行命令`sudo chmod -R 700 /etc/letsencrypt/live/`和`sudo chown -R root:wheel /etc/letsencrypt/live/`。


*   **关闭 SIP（仅作为最后手段）**：重启 Mac，按住`Command+R`进入恢复模式；打开终端，执行`csrutil disable`；重启系统后尝试访问目录，完成后再次进入恢复模式启用 SIP，执行`csrutil enable`。**注意**：关闭 SIP 会降低系统安全性，仅在必要时使用。


### 4. 证书使用建议&#xA;

即使无法直接访问系统目录，仍可通过以下方式使用证书：




*   在 Web 服务器配置中直接指定证书路径（需 root 权限），例如 Nginx 配置：




```
ssl\_certificate /etc/letsencrypt/live/rubymall.com/fullchain.pem;


ssl\_certificate\_key /etc/letsencrypt/live/rubymall.com/privkey.pem;
```



*   若需在普通用户程序中使用证书，按前文方法复制到用户目录后再引用。


通过以上方法，基本可以解决在 macOS 系统下申请 SSL 证书及访问证书目录时遇到的问题。在实际操作过程中，若遇到其他异常情况，欢迎随时交流探讨。


> （注：文档部分内容可能由 AI 生成）
>