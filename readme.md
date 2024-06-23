
# 写在前面

1. 开始本教程之前，需要确保已经有一台可以正常联网的服务器
2. 本教程所有命令皆以ubuntu 22.04为例
3. 推荐使用国外域名提供商和云服务器厂商
4. 如果使用的是国内云服务器厂商，请确保已经完成备案
5. 本教程使用jar文件部署，数据库使用的是MySQL

# 配置环境

## 安装JAVA

首先，我们需要先安装Java环境，目前Halo最低需要JRE 17的环境
使用SSH工具登录到自己的服务器之后，在终端窗口以此输入以下命令

```
sudo apt update && sudo apt upgrade #先更新系统包
sudo apt install openjdk-18-jdk #安装OpenJDK（开源的Java）
java -version #确认JAVA安装成功
```

  **若终端输出了java的版本信息，则证明安装成功**

## 安装MySQL

*Halo2同时支持  MySQL MariaDB 和 PostgreSQL*
*这里使用的是MySQL*

首先，使用下面的命令来安装MySQL服务器

```
sudo apt install mysql-server
```

安装完成后，数据库服务应该会自动启动，可以通过以下命令来进行确认

```
sudo systemctl status mysql
```

若能看到`active(running)`字样，则证明服务已启动

若服务未能正常运行，则可以使用以下命令来启动服务

```
sudo systemctl start mysql
sudo systemctl enable mysql 
```

**也可以通过`mysql --version`命令来验证安装结果**
**想要重启的话可以使用命令`sudo systemctl restart mysql`**

之后，就是启动`mysql-secure-installation`服务，一般来讲，它会自动启动，若没有自动启动，可以执行以下命令

```
sudo mysql_secure_installation
```

之后会弹一个提示，询问是否验证密码插件。它通过检查用户密码的强度来增强 MySQL Server 的安全性，允许用户仅设置强密码。按 Y 接受 VALIDATION 或按 RETURN 键跳过。这里按y来接受
示例：`Press y|Y for yes, any other key for No:`
接下来，会看到提示，来让你设置root密码。输入密码并按回车键，注意，为了安全，在控制台中不会显示键入的任何内容。
示例：

```
Please set the password for root here.
New passwrod:
re-enter new password:
```

接下来，会看到一个提示，询问你是否删除所有匿名用户，输入 Y 表示是。

## 为halo2创建一个数据库

首先使用`mysql -u root -p`命令登入到数据库

接下来使用以下的命令，来创建数据库和新建用户

```
-- 登录到MySQL后执行以下命令
CREATE USER 'user'@'localhost' IDENTIFIED BY 'password';
CREATE DATABASE halo;
GRANT ALL PRIVILEGES ON halo.* TO 'user'@'localhost';
FLUSH PRIVILEGES;

1. 将user替换为自己想设置的名称
2. 将password 替换为自己的密码，请使用强密码
3. 请记好用户名和密码
4. 新建的数据库名称为halo
```

# 安装halo2

*不推荐使用root用户来使用halo，若想直接使用root用户，请跳过第一步*

## 创建新系统用户

可以直接执行下面的命令

```
useradd -m halo #创建一个名为 halo 的用户
passwd halo #为 halo 用户创建密码
su - halo #登录到 halo 账户
```

## 正式开始安装halo

*这里是照搬官方文档*
[官方文档链接](https://docs.halo.run/getting-started/install/jar-file)

创建存放运行包的目录

```
mkdir ~/app && cd ~/app
```

下载运行包

```
wget https://dl.halo.run/release/halo-2.16.0.jar -O halo.jar
#写教程时，最新版本只有2.16.0，若有更新版本，请使用更新版本
```

创建 工作目录

```
mkdir ~/.halo2 && cd ~/.halo2
```

创建配置文件

```
vim application.yaml
```

将以下内容复制到 application.yaml 中，根据下面的配置说明进行配置。

```
server:
  # 运行端口
  port: 8090
spring:
  # 数据库配置，支持 MySQL、MariaDB、PostgreSQL、H2 Database，具体配置方式可以参考下面的数据库配置
  r2dbc:
    url: r2dbc:pool:mysql://localhost:3306/halo #若前面创建数据时，没有修改配置，这里可以不用改，否则将3306改为你自己修改的数据库端口号，将halo修改为自己更改的数据库名称
    username: user #修改为自己创建的数据用户名称
    password: password #修改为自己创建的数据用户密码
  sql:
    init:
      mode: always
      # 需要配合 r2dbc 的配置进行改动
      platform: mysql
halo:
  caches:
    page:
      # 是否禁用页面缓存
      disabled: true
  # 工作目录位置
  work-dir: ${user.home}/.halo2
  # 外部访问地址
  external-url: http://localhost:8090 #若已经有域名，请将这里改为你自己的域名
  # 附件映射配置，通常用于迁移场景
  attachment:
    resource-mappings:
      - pathPattern: /upload/**
        locations:
          - migrate-from-1.x
```

配置写完之后，先按`ESC`,再按`shift+i`，再输入`wq`并按回车按键完成保存。

测试运行Halo
`cd ~/app && java -jar halo.jar --spring.config.additional-location=optional:file:$HOME/.halo2/`
打开浏览器，输入<http://你的服务器ip:8090，来测试halo是否正常运行网页能否正常打开>
*⚠️注意：需要在服务器防火墙中允许tcp协议8090端口入站*

若访问正常，可以使用`ctrl+c`来停止服务

### 作为服务运行

下面将介绍如何将 Halo 作为服务运行，以实现在关闭 ssh 连接后，Halo 仍然可以正常运行。

1. 退出 halo 账户，登录到 root 账户，如果当前就是 root 账户，请略过此步骤。
2. 创建 halo.service 文件

```
vim /etc/systemd/system/halo.service
```

将以下内容复制到 halo.service 中，根据下面的配置说明进行配置。

```
[Unit]
Description=Halo Service
Documentation=https://docs.halo.run
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=USER
ExecStart=/usr/bin/java -server -Xms256m -Xmx256m -jar JAR_PATH --spring.config.additional-location=optional:file:/home/halo/.halo2/
ExecScom=/bin/kill -s QUIT $MAINPID
Restart=always
StandOutput=syslog

StandError=inherit

[Install]
WantedBy=multi-user.target
```

| 参数 | 含义 |
|----|----|
|JAR_PATH：| Halo 运行包的绝对路径，例如 /home/halo/app/halo.jar，注意：此路径不支持 ~ 符号。|
|USER：| 运行 Halo 的系统用户，如果有按照上方教程创建新的用户来运行 Halo，修改为你创建的用户名称即可。反之请删除 User=USER。|

*⚠️请确保 /usr/bin/java 是正确无误的。建议将 ExecStart 中的命令复制出来运行一下，保证命令有效。*

重新加载 systemd

```
systemctl daemon-reload
```

运行服务

```
systemctl start halo
```

在系统启动时启动服务

```
systemctl enable halo
```

最后，你可以通过下面的命令查看服务日志：

```
journalctl -n 20 -u halo
```

### 使用`screen`来保持halo运行

screen和systemd二选一就行
如果使用screen，请确保服务器已经安装screen
若使用screen，请以此使用下面命令

```
screen -R halo
cd ~/app && java -jar halo.jar --spring.config.additional-location=optional:file:$HOME/.halo2/
```

# 配置Nginx,并部署ssl

安装nginx

```
sudo apt install nginx
```

启动nginx服务

```
sudo systemctl start nginx
sudo systemctl enable nginx
```

使用vim编辑器打开配置文件

```
sudo nano /etc/nginx/nginx.conf
```

复制粘贴下面的代码到配置文件里

```
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
 worker_connections 768;
 # multi_accept on;
}

http {

 ##
 # Basic Settings
 ##

 sendfile on;
 tcp_nopush on;
 types_hash_max_size 2048;
 # server_tokens off;

 # server_names_hash_bucket_size 64;
 # server_name_in_redirect off;

 include /etc/nginx/mime.types;
 default_type application/octet-stream;

 ##
 # SSL Settings
 ##

 ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
 ssl_prefer_server_ciphers on;

 ##
 # Logging Settings
 ##

 access_log /var/log/nginx/access.log;
 error_log /var/log/nginx/error.log;

 ##
 # Gzip Settings
 ##

 gzip on;

 # gzip_vary on;
 # gzip_proxied any;
 # gzip_comp_level 6;
 # gzip_buffers 16 8k;
 # gzip_http_version 1.1;
 # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

 ##
 # Virtual Host Configs
 ##

 include /etc/nginx/conf.d/*.conf;

 include /etc/nginx/sites-enabled/*;

 upstream halo {
   server 127.0.0.1:8090; #这里的端口号改为你自己设置的halo服务端口号
 } 

 server {
  listen 443 ssl;
  listen [::]:443 ssl;
   server_name www.yourdomain.com yourdomain.com;
   client_max_body_size 1024m;

      #填写证书文件绝对路径
      ssl_certificate /etc/nginx/cert/www.yourdomain.com.pem;
      #填写证书私钥文件绝对路径
      ssl_certificate_key /etc/nginx/cert/www.yourdomain.com.key;

      ssl_session_cache shared:SSL:1m;
      ssl_session_timeout 5m;

   location / {
     proxy_pass http://halo;
    proxy_set_header HOST $host;
     proxy_set_header X-Forwarded-Proto $scheme;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}

#这里是将所有的http链接重定向到https链接
 server {
     listen 80;
     #填写证书绑定的域名
     server_name yourdomain.com;
     #将所有HTTP请求通过rewrite指令重定向到HTTPS。
     return 301 https://$server_name$request_uri;
}

}

```

**⏰ ssl证书可以去你购买域名的地方去申请免费ssl证书**
**⚠️要将服务器的防火墙的80和443端口打开**
