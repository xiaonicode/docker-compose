# docker-compose

> Docker Compose Builds the Basic Development Environment

## 环境准备

### 安装 Docker-CE

1. 卸载相关依赖

   ```bash
   sudo yum remove -y docker \
      docker-client \
      docker-client-latest \
      docker-common \
      docker-latest \
      docker-latest-logrotate \
      docker-logrotate \
      docker-engine
   ```

2. 安装必要的一些系统工具

   ```bash
   sudo yum install -y yum-utils device-mapper-persistent-data lvm2
   ```

3. 添加软件源信息

   ```bash
   sudo yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   ```

   ```bash
   sudo sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
   ```

4. 更新 yum 源

   ```bash
   sudo yum makecache fast
   ```

5. 安装 docker-ce

   ```bash
   sudo yum install -y docker-ce
   ```

6. 启动服务，并设置自动重启

   ```bash
   sudo systemctl start docker
   sudo systemctl enable docker
   ```

### 配置镜像加速

1. 创建级联目录

   ```bash
   sudo mkdir -p /etc/docker
   ```

2. 创建 `/etc/docker/daemon.json`，指定加速地址

   ```bash
   sudo tee /etc/docker/daemon.json <<-'EOF'
   {
     "registry-mirrors": ["https://${容器镜像加速地址}.mirror.aliyuncs.com"]
   }
   EOF
   ```

3. 重新加载配置文件

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```

### 卸载 Docker-CE

1. 卸载 `docker-ce`、`docker-ce-cli` 和 `containerd.io`

   ```bash
   sudo yum remove -y docker-ce docker-ce-cli containerd.io
   ```

2. 主机上的映像、容器、卷或自定义配置文件不会自动删除，删除所有映像、容器和卷

   ```bash
   sudo rm -rf /var/lib/docker
   sudo rm -rf /var/lib/containerd
   ```

### 安装 Docker-Compose

1. 切换到超级管理员

   ```bash
   su root
   ```

2. 下载二进制包

   ```bash
   # https://hub.fastgit.org 是 https://github.com 的代理镜像
   curl -L https://hub.fastgit.org/docker/compose/releases/download/v2.2.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
   ```

3. 赋予 `docker-compose` 可执行权限

   ```bash
   chmod +x /usr/local/bin/docker-compose
   ```

4. 创建软链接

   ```bash
   ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
   ```

### 卸载 Docker-Compose

1. 删除软链接

   ```bash
   sudo rm /usr/bin/docker-compose
   ```

2. 删除二进制包

   ```bash
   sudo rm /usr/local/bin/docker-compose
   ```

### 开启 TCP 远程连接

1. 编辑 `/lib/systemd/system/docker.service` 文件，在 `ExecStart` 行尾添加 `-H tcp://0.0.0.0:2375`，注意是空格分隔

   ```bash
   sudo vim /lib/systemd/system/docker.service
   ```

2. 重新加载配置文件

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```

3. 观察端口是否监听

   ```bash
   ss -lt | grep 2375
   ```
