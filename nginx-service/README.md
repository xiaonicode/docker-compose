# 配置 HTTPS 证书

## 文件准备

1. `Windows` 系统以管理员权限，利用 [SwitchHosts](https://swh.app/zh/) 工具修改 `C:\Windows\System32\drivers\etc\hosts` 文件，配置域名映射

   ```text
   # 虚拟机 IP 地址 ← 自定义域名
   192.168.146.100 sctcmall.com www.sctcmall.com
   ```

2. 修改 `nginx-service/conf/nginx.conf` 配置文件，配置上游服务器

   ```conf
   #gzip  on;
   upstream sctcmall {
       server 192.168.146.1:8888; # IP：通过 Windows 的 ipconfig 命令查看以太网适配器 VMware Network Adapter VMnet8 的 IPv4 地址得到，以后要替换为真正的 IP
   }
   ```

3. 新增 `nginx-service/conf/conf.d/sctcmall.conf` 配置文件，填入以下内容

   ```conf
   server {
       if ($host = sctcmall.com) {
           return 301 https://$host$request_uri;
       }

       if ($host = www.sctcmall.com) {
           return 301 https://$host$request_uri;
       }

       listen       80;
       listen  [::]:80 ipv6only=on;
       server_name  sctcmall.com www.sctcmall.com;
       return 404;
   }

   server {
       listen           443 ssl;
       listen      [::]:443 ssl ipv6only=on;
       server_name sctcmall.com www.sctcmall.com;

       keepalive_timeout   70;              # 启用保持连接，以通过一个连接发送多个请求
       ssl_session_cache   shared:SSL:10m;  # 重用 SSL 会话参数以避免并行和后续连接的 SSL 握手
       ssl_session_timeout 10m;             # SSL 会话存活时间

       ssl_certificate     /ssl/server.crt; # SSL 证书
       ssl_certificate_key /ssl/server.key; # SSL 私钥

       location / {
           proxy_pass http://sctcmall; # 配置反向代理
       }
   }
   ```

## 生成证书

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
   > Country Name (2 letter code) [XX]:CN ← 国家代号，中国输入CN
   >
   > State or Province Name (full name) []:SiChuan ← 省的全名，拼音
   >
   > Locality Name (eg, city) [Default City]:Chengdu ← 市的全名，拼音
   >
   > Organization Name (eg, company) [Default Company Ltd]:SCTC Corp. ← 公司英文名
   >
   > Organizational Unit Name (eg, section) []: ← 可以不输入
   >
   > Common Name (eg, your name or your server's hostname) []: ← 此时不输入
   >
   > Email Address []:admin@sctcmall.com ← 电子邮箱，可随意填
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

## 重启 Nginx

1. 重启容器服务

   ```bash
   sudo docker-compose restart nginx-server
   ```

2. 测试通过 `sctcmall.com` 访问
