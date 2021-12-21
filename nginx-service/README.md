# HTTPS 加密传输

## 前提条件

1. `Windows` 系统以管理员权限，利用 [SwitchHosts](https://swh.app/zh/) 工具修改 `C:\Windows\System32\drivers\etc\hosts` 文件，配置域名映射

   ```text
   # 虚拟机 IP 地址 ← 自定义域名
   192.168.146.100 example.com www.example.com
   ```

2. 修改 `nginx-service/conf/nginx.conf` 配置文件，配置上游服务器

   ```conf
   #gzip  on;
   upstream example {
       server 192.168.146.1:8888; # IP：通过 Windows 的 ipconfig 命令查看以太网适配器 VMware Network Adapter VMnet8 的 IPv4 地址得到，以后要替换为真正的 IP
   }
   ```

## 方式一：`openssl`

### 文件准备

1. 新增 `nginx-service/conf/conf.d/redirect.conf` 配置文件

   ```conf
   server {
       listen       80;
       server_name  example.com www.example.com;

       if ($host = example.com) {
           return 301 https://$host$request_uri;
       }

       if ($host = www.example.com) {
           return 301 https://$host$request_uri;
       }

       return 404;
   }
   ```

2. 新增 `nginx-service/conf/conf.d/example.conf` 配置文件

   ```conf
   server {
       listen       443 ssl;
       server_name  example.com;

       ssl_certificate      /ssl/server.crt; # SSL 证书
       ssl_certificate_key  /ssl/server.key; # SSL 私钥

       ssl_session_cache    shared:SSL:10m;  # 重用 SSL 会话参数以避免并行和后续连接的 SSL 握手
       ssl_session_timeout  10m;             # SSL 会话存活时间

       ssl_ciphers  HIGH:!aNULL:!MD5;
       ssl_prefer_server_ciphers  on;

       location / {
           proxy_pass http://example; # 配置反向代理
       }
   }
   ```

3. 新增 `nginx-service/conf/conf.d/example-www.conf` 配置文件

   ```conf
   server {
       listen       443 ssl;
       server_name  www.example.com;

       ssl_certificate      /ssl/server.crt;
       ssl_certificate_key  /ssl/server.key;

       ssl_session_cache    shared:SSL:10m;
       ssl_session_timeout  10m;

       ssl_ciphers  HIGH:!aNULL:!MD5;
       ssl_prefer_server_ciphers  on;

       location / {
           proxy_pass http://example;
       }
   }
   ```

### 生成证书

1. 在 `nginx-service` 目录下创建并进入 `ssl` 子目录

2. 创建服务器证书的密钥文件 `server.key`，并且记住两次输入的密码

   ```bash
   sudo openssl genrsa -des3 -out server.key 2048
   ```

3. 创建服务器证书的申请文件 `server.csr`

   ```bash
   sudo openssl req -new -key server.key -out server.csr
   ```
   > Enter pass phrase for server.key: ← 输入前面创建的密码
   >
   > Country Name (2 letter code) [XX]:CN ← 国家代号，中国输入 CN
   >
   > State or Province Name (full name) []:Sichuan ← 省的全名，拼音
   >
   > Locality Name (eg, city) [Default City]:Chengdu ← 市的全名，拼音
   >
   > Organization Name (eg, company) [Default Company Ltd]: ← 可以不输入
   >
   > Organizational Unit Name (eg, section) []: ← 可以不输入
   >
   > Common Name (eg, your name or your server's hostname) []: ← 此时不输入
   >
   > Email Address []:admin@example.com ← 电子邮箱，可随意填
   >
   > A challenge password []: ← 可以不输入
   >
   > An optional company name []: ← 可以不输入

4. 备份一份服务器密钥文件

   ```bash
   sudo cp server.key server.key.bak
   ```

5. 去除文件口令

   ```bash
   sudo openssl rsa -in server.key.bak -out server.key
   ```

6. 生成证书文件 `server.crt`

   ```bash
   sudo openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
   ```

### 重启 Nginx

1. 重启容器服务

   ```bash
   sudo docker-compose restart nginx-server
   ```

2. 测试通过 `example.com` 访问，发现浏览器提示 `HTTPS` 不安全，因为 `CA` 根目录证书不受信任

## 方式二：`let's encrypt`

### 文件准备

1. 新增 `nginx-service/conf/conf.d/example.conf` 配置文件

   ```conf
   server {
       listen       80;
       server_name  example.com www.example.com;

       location / {
           proxy_pass http://example; # 配置反向代理
       }
   }
   ```

2. 新增 `nginx-service/conf/conf.d/example-www.conf` 配置文件

   ```conf
   server {
       listen       80;
       server_name  www.example.com;

       location / {
           proxy_pass http://example; # 配置反向代理
       }
   }
   ```

### 替换 Alpine 镜像

1. 首先进入 `Nginx` 容器，进入后的用户身份默认为 `root`

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

### 配置 HTTPS 证书

1. 安装签发证书工具

   ```bash
   apk add --no-cache certbot-nginx
   ```

2. 申请证书

   ```bash
   certbot --nginx
   ```

3. 具体命令行交互过程（存在自定义域名无法被 `DNS` 解析的问题，将导致后续教程步骤无法进行）

   ```bash
   Saving debug log to /var/log/letsencrypt/letsencrypt.log
   Enter email address (used for urgent renewal and security notices)
    (Enter 'c' to cancel): admin@example.com # 输入常用邮箱

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
   2: www.example.com # 这里给出的是自定义域名
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

### 重启 Nginx

1. 重启容器服务

   ```bash
   sudo docker-compose restart nginx-server
   ```

2. 浏览器直接输入 `example.com` 并访问，发现 `HTTP` 自动升级为 `HTTPS`
