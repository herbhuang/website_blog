---
title: 添加 Let's Encrypt SSL 证书泛域名支持
date: 2018-07-16 00:09:23
categories: website
---
今天在添加 CDN 的时候发现要在腾讯云上上传证书，检查后发现自己配置的泛域名证书并没有生效，把 lnmp 环境升级了一下，切换成了泛域名。
过程中出了一些小错，发现是 nginx 的 conf 文件配置错误，所以设置了一下。顺便温习了 vim 如何批量添加注释。
下一步打算将 HSTS 开启，并引入全站 CDN 以保护源站。

## 参考资料

- [Let'sEncrypt 免费通配符/泛域名SSL证书添加使用教程](https://lnmp.org/faq/letsencrypt-wildcard-ssl.html)

- [Let's Encrypt 官方网站](https://letsencrypt.org/)

- [维基百科：Let's Encrypt 项目](https://zh.wikipedia.org/wiki/Let%27s_Encrypt)