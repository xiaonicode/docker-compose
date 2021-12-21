# HTTP 升级 HTTPS

## 替换 Alpine 镜像

1. 首先进入 `NGINX` 容器，进入后的用户身份默认为 `root`

   ```bash
   sudo docker exec -it nginx_server /bin/sh
   ```

2. 替换 `APK` 镜像

   ```bash
   # 备份镜像仓库
   cp -a /etc/apk/repositories /etc/apk/repositories.bak
   # 替换为华为云
   sed -i "s@https://dl-cdn.alpinelinux.org/@https://repo.huaweicloud.com/@g" /etc/apk/repositories
   # 更新索引
   apk update
   ```

## HTTPS 加密传输

1. 安装签发证书工具

   ```bash
   apk add --no-cache certbot-nginx
   ```

2. 申请证书

   ```bash
   certbot --nginx
   ```

3. 具体命令行交互过程

   ```bash
   Saving debug log to /var/log/letsencrypt/letsencrypt.log
   Enter email address (used for urgent renewal and security notices)
    (Enter 'c' to cancel): example@qq.com # 输入常用邮箱

   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   Please read the Terms of Service at
   https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
   agree in order to register with the ACME server. Do you agree?
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   (Y)es/(N)o: # y 输入 'y' 同意

   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   Would you be willing, once your first certificate is successfully issued, to
   share your email address with the Electronic Frontier Foundation, a founding
   partner of the Let's Encrypt project and the non-profit organization that
   develops Certbot? We'd like to send you email about our work encrypting the web,
   EFF news, campaigns, and ways to support digital freedom.
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   (Y)es/(N)o: y # 输入 'y' 同意
   Account registered.

   Which names would you like to activate HTTPS for?
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   1: example.com     # 注意：这些域名一定是真实购买的，且已做 ICP 备案
   2: www.example.com # 这里给出的是假域名
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   Select the appropriate numbers separated by commas and/or spaces, or leave input
   blank to select all options shown (Enter 'c' to cancel): # 直接按一下回车
   Requesting a certificate for example.com and www.example.com

   Successfully received certificate.
   Certificate is saved at: /etc/letsencrypt/live/example.com/fullchain.pem
   Key is saved at:         /etc/letsencrypt/live/example.com/privkey.pem
   This certificate expires on 2022-03-20.
   These files will be updated when the certificate renews.

   Deploying certificate
   Successfully deployed certificate for example.com to /etc/nginx/conf.d/wp.conf
   Successfully deployed certificate for www.example.com to /etc/nginx/conf.d/wp.conf
   Congratulations! You have successfully enabled HTTPS on https://example.com and https://www.example.com

   NEXT STEPS:
   - The certificate will need to be renewed before it expires. Certbot can automatically renew the certificate in the background, but you may need to take steps to enable that functionality. See https://certbot.org/renewal-setup for instructions.
   We were unable to subscribe you the EFF mailing list because your e-mail address appears to be invalid. You can try again later by visiting https://act.eff.org.

   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   If you like Certbot, please consider supporting our work by:
    * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
    * Donating to EFF:                    https://eff.org/donate-le
   - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
   ```

4. 浏览器直接输入 `http://example.com` 并访问，发现 `HTTP` 自动升级为 `HTTPS`
