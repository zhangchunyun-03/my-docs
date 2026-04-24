# 阿里云ECS个人博客搭建
## 前置准备（先把阿里云试用 ECS 配置好）
### 1.ECS 必做配置
进入阿里云 ECS 实例 → 重置实例密码（root 账号密码，自己记牢）  
重置密码后 重启 ECS 实例  
复制保存：公网 IP、root 用户名、root 密码
### 2.放行阿里云安全组（企业最小权限原则）
ECS 控制台 → 网络与安全 → 安全组 → 配置入方向规则
| 协议 | 端口 | 授权对象 | 说明 |
|-----|------|---------|---------| 
| TCP | 22 | 本地电脑ip | 只允许自己 SSH 连接，防暴力破解 |
| TCP | 80 | 0.0.0.0/0 | 网站 HTTP 访问 |
| TCP | 443 | 0.0.0.0/0 | 只网站 HTTPS 访问 |
## 一、Xshell 连接阿里云 CentOS7.9 服务器
打开 Xshell → 文件 → 新建  
名称：自定义（blog - 阿里云运维机）  
协议：SSH  
主机：ECS公网IP  
端口号：22  
点击连接，输入用户名：root，再输入重置的 root 密码  
登录成功，进入命令行界面
## 二、企业级服务器基线初始化
### 2.1 修改主机名（企业规范）
```bash
hostnamectl set-hostname ops-blog-prod
## 2.2 配置时间同步（生产集群必须时间统一）
```bash
yum install -y chrony
systemctl start chronyd
systemctl enable chronyd
chronyc sources
### 2.3 更换阿里云 CentOS7 国内 YUM 源
```bash
yum install -y wget curl
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all && yum makecache
```
### 2.4 创建普通运维用户 + 授予 sudo 权限（禁止长期用 root 操作业务）
企业规范：日常运维不用 root，最小权限
```bash
useradd opsuser
-- 给 opsuser 这个用户设置密码
echo "Zcy20030812." | passwd opsuser --stdin
-- 把 opsuser 用户加入 wheel 管理员组
usermod -aG wheel opsuser
```
### 2.5 关闭系统防火墙 + SELinux（云服务器生产常规操作）
```bash
-- 通常在服务器内部网络环境、或者已经有上层安全组 / 云防火墙防护时，会临时关闭系统自带的 firewalld，避免端口被拦截。
systemctl stop firewalld
systemctl disable firewalld

setenforce 0
-- 直接修改 SELinux 配置文件，把强制模式改成禁用模式
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```
## 三、安装企业基础依赖环境
后续 Nginx、PHP、数据库、编译依赖全部一次性装好
```bash
yum install -y gcc gcc-c++ pcre-devel zlib-devel libxml2-devel libcurl-devel libpng-devel openssl-devel
```
## 四、分层部署企业 Web 架构 Nginx + PHP-FPM + MariaDB
Nginx 静态反向代理 → PHP-FPM 动态解析 → MariaDB 数据层，三层分离，中小企业标准生产架构
### 4.1 部署 Nginx 并设置开机自启
```bash 
yum install -y nginx
systemctl start nginx
systemctl enable nginx
```
### 4.2 部署 PHP7.4 + PHP-FPM（独立进程，和 Nginx 解耦）
```bash
yum install -y epel-release remi-release
yum install -y php74 php74-fpm php74-mysqlnd php74-gd php74-curl php74-xml

systemctl start php-fpm
systemctl enable php-fpm
```
### 4.3 部署 MariaDB 数据库 + 生产安全初始化
```bash
yum install -y mariadb-server mariadb
systemctl start mariadb
systemctl enable mariadb

-- 1.先获取 MySQL 初始临时密码,复制
grep 'temporary password' /var/log/mysqld.log
-- 2.用临时密码登录 MySQL
mysql -uroot -p
# 回车后，粘贴刚才获取的临时密码
```
修改 root 密码（必须符合复杂度要求：大小写 + 数字 + 特殊符号）
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY '你的新密码';
FLUSH PRIVILEGES;
exit;
```
```bash
-- 企业数据库安全加固（必做）
mysql_secure_installation
```
交互操作按企业规范选：  
设置 root 数据库强密码  
移除匿名用户 y  
禁止 root 远程登录 y  
删除测试库 y  
刷新权限 y  
## 五、创建企业规范目录结构（对标生产）
统一规划业务、日志、备份路径，不混乱乱放
```bash
-- 网站代码目录
mkdir -p /data/www/blog
-- 日志统一存放目录
mkdir -p /data/logs/nginx
-- 数据库/文件备份目录
mkdir -p /data/backup/mysql
```
## 六、企业 Git 代码上线流程（拒绝解压包上传）
模仿公司开发提交、运维拉取发布版本
### 6.1 安装 Git
```bash
yum install -y git
```
### 6.2 Git 拉取 WordPress 源码到业务目录
```bash
git clone https://gitee.com/liuxiaocai/wordpress.git /data/www/blog
```
### 6.3 权限最小化配置（生产安全规范）
```bash
chown -R nginx:nginx /data/www/blog
chmod -R 755 /data/www/blog
```
## 七、数据库生产级配置（不使用 root 建站）
### 7.1 进入数据库
```bash
mysql -uroot -p
```
### 7.2 创建独立业务库 + 独立业务账号（面试高频考点）
```sql
create database blog_prod default character set utf8mb4 collate utf8mb4_unicode_ci;
-- 仅允许本地应用连接，禁止外网
create user blog_app@localhost identified by 'Zcy20030812@';
-- 最小权限授权
grant all privileges on blog_prod.* to blog_app@localhost;
flush privileges;
exit;
```
记录信息：库名：blog_prod账号：blog_app密码：Zcy20030812@
## 八、手写 Nginx 虚拟主机配置（企业运维核心能力）
### 8.1 新建站点配置文件
```bash
vim /etc/nginx/conf.d/blog.conf
```
复制粘贴下面配置（适配 PHP-FPM）：
```nginx
server {
    listen 80;
    server_name 你的ECS公网IP;

    root /data/www/blog;
    index index.php index.html;

    access_log /data/logs/nginx/blog_access.log;
    error_log /data/logs/nginx/blog_error.log;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```
esc → 输入 :wq 保存退出
### 8.2 检测 Nginx 配置 + 重启生效
```bash
nginx -t
systemctl restart nginx
```
## 九、浏览器初始化 WordPress 博客
浏览器访问：http://ECS公网IP  
填写数据库信息：  
数据库名：blog_prod  
用户名：blog_app  
密码：Zcy20030812@  
数据库主机：localhost  
设置博客管理员账号密码，完成安装
## 十、阿里云免费 SSL 证书 + 全站 HTTPS（企业标配）
### 10.1 阿里云申请免费 SSL 证书
阿里云控制台搜索：SSL 证书
选择 免费证书 → 购买 → 选 免费 DVSSL
输入你的域名（没有域名就用 ECS 公网 IP 也能申请）
提交后，点 一键验证 → 等待 1 分钟签发
### 10.2 下载证书
签发成功 → 下载 → 选择 Nginx下载后得到两个文件：
xxx.pem
xxx.key
### 10.3 Xshell 上传证书到服务器
```bash
# 创建证书目录
mkdir -p /etc/nginx/ssl
```
把下载的两个文件上传到：
```plaintext
/etc/nginx/ssl/
```
### 10.4 替换 Nginx 企业级 HTTPS 配置
```bash
vim /etc/nginx/conf.d/blog.conf
```
全部删除原有内容，粘贴下面这套生产配置：
```nginx
server {
    listen 80;
    server_name 你的域名或公网IP;
    # 80端口强制跳转HTTPS（企业标准）
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name 你的域名或公网IP;

    # SSL证书路径
    ssl_certificate /etc/nginx/ssl/你的证书.pem;
    ssl_certificate_key /etc/nginx/ssl/你的证书.key;

    # 企业安全加密套件
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!MD5:!RC4:!DHE;

    # 网站目录
    root /data/www/blog;
    index index.php index.html;

    # 日志规范路径
    access_log /data/logs/nginx/blog_access.log;
    error_log /data/logs/nginx/blog_error.log;

    # WordPress伪静态
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # PHP解析
    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```
保存退出：
```plaintext
esc → :wq
```
### 10.5 重启 Nginx 生效
```bash
nginx -t
systemctl restart nginx
```
现在你的网站已经是企业标准 HTTPS 了！
## 十一、企业级自动备份（每天自动备份数据库 + 网站）
### 11.1 创建备份脚本
```bash
vim /data/backup/backup.sh
```
粘贴以下企业级备份脚本：
```bash
#!/bin/bash
DATE=$(date +%Y%m%d)
BACKUP_DIR="/data/backup"
DB_NAME="blog_prod"
DB_USER="blog_app"
DB_PASS="Blog@Prod2026"

# 备份数据库
mysqldump -u$DB_USER -p$DB_PASS $DB_NAME > $BACKUP_DIR/mysql/blog_$DATE.sql

# 备份网站文件
cd /data/www
zip -r $BACKUP_DIR/blog_$DATE.zip blog/

# 只保留最近7天备份
find $BACKUP_DIR -name "*.sql" -mtime +7 -delete
find $BACKUP_DIR -name "*.zip" -mtime +7 -delete
```
### 11.2 赋予执行权限
bash
运行
chmod +x /data/backup/backup.sh
### 11.3 加入定时任务（每天凌晨 2 点自动备份）
bash
运行
crontab -e
添加一行：
bash
运行
0 2 * * * /data/backup/backup.sh
企业自动化备份完成！
## 十二、企业级安全加固（求职必做）
### 12.1 禁止 root 远程登录（生产标准）
```bash
vim /etc/ssh/sshd_config
```
修改：
```plaintext
PermitRootLogin no
```
重启 SSH：
```bash
systemctl restart sshd
```
以后只能用 opsuser 登录，再 sudo 切换 root。
### 12.2 修改 SSH 默认端口（防暴力破解）
```bash
vim /etc/ssh/sshd_config
```
修改：
```plaintext
Port 2718
```
重启 SSH：
```bash
systemctl restart sshd
```
阿里云安全组必须放行 2718 端口
### 12.3 网站目录权限加固（最安全）
```bash
chown -R nginx:nginx /data/www/blog
find /data/www/blog -type d -exec chmod 755 {} \;
find /data/www/blog -type f -exec chmod 644 {} \;
chmod 600 /data/www/blog/wp-config.php
```
### 12.4 关闭数据库远程连接（企业必须）
```bash
vim /etc/my.cnf
```
添加：
```plaintext
bind-address = 127.0.0.1
```
重启数据库：
```bash
systemctl restart mariadb
```

## 遇到的问题解决
### 1.连接不上Xshell
#### 方法1： 临时最简单解决（先能连上再说）  
去阿里云安全组，先把 22 端口暂时改回 0.0.0.0/0 全网允许  
#### 方法2： 查出【阿里云眼里你真实的 IP】
你本地看到的公网 IP ≠ 阿里云服务器收到的真实来源 IP  
现在 Xshell 已经连上服务器，在服务器里执行：  
```bash
last -i
```
看这次登录记录里的 IP 地址,这个才是阿里云防火墙真实识别到的我的IP，不是电脑 cmd 查到的那个  
然后去服务器的安全组修改ssh远程登录的ip（ip/32）
### 2.执行 git clone 时，系统提示 Username for 'https://gitee.com':
说明：这个仓库要么是私有仓库，需要 Gitee 账号密码授权访问；要么是你当前的网络 / 仓库地址有问题，导致匿名访问失败。
#### 方法1：用 Gitee 账号授权下载
当提示 Username for 'https://gitee.com': 时，输入你的 Gitee 用户名  
回车后，再输入你的 Gitee 密码（或个人访问令牌）  
验证通过后，就能正常下载代码了  
#### 方法2：改用公开仓库的地址
如果这是一个公开仓库，大概率是地址写错了。你可以直接用浏览器打开 https://gitee.com/liuxiaocai/wordpress.git 看看：  
如果能直接访问、下载 ZIP 包，说明仓库是公开的，换用下面这种方式下载：  
```bash
# 直接下载 ZIP 包，不用 git
wget https://gitee.com/liuxiaocai/wordpress/repository/archive/master.zip -O wordpress.zip
unzip wordpress.zip -d /data/www/blog
```
如果浏览器里提示需要登录，说明这是私有仓库，只能用方法1授权访问。
#### 方法3：终止当前命令（ctrl + C）
既然这个地址要账号，咱们直接绕开它，用 wget 下载压缩包就行：
```bash
# 1. 先创建目标目录
mkdir -p /data/www/blog
# 2. 下载WordPress官方包（不用Gitee账号，直接从官网下）
wget https://cn.wordpress.org/latest-zh_CN.zip -O /tmp/wordpress.zip
# 3. 解压到目标目录
unzip /tmp/wordpress.zip -d /data/www/blog
# 4. 调整权限
chown -R nginx:nginx /data/www/blog
```
这样就能直接拿到完整的 WordPress 源码，不用注册任何账号。
### 3.被 WordPress 官网限流了（429）
用我的 GitHub 高速镜像
```bash
mkdir -p /data/www/blog
wget https://github.com/WordPress/WordPress/archive/refs/heads/master.zip -O /tmp/wp.zip
unzip /tmp/wp.zip -d /tmp
mv /tmp/WordPress-master/* /data/www/blog/
chown -R nginx:nginx /data/www/blog
```
第三步：验证是否下载成功
```bash
ls /data/www/blog
```
只要出现 wp-config.php 这些文件，就成功了！
### 4.没有域名，申请不了SSL证书
阿里云免费证书必须绑定已备案域名
#### 方法1：用真实域名（推荐，长期可用）
步骤 1：注册并解析域名  
去阿里云 / 腾讯网 / Namecheap 注册一个真实域名（如 你的域名.com）  
进入域名解析控制台，添加 2 条记录：  
| 主机记录 | 类型 |	记录值 |
|---------|------|---------|
| blog | A | 8.163.108.4 |
| www.blog | A | 8.163.108.4 |
保存，等待生效（通常 1-10 分钟）  
步骤 2：验证域名是否解析成功  
服务器上执行，能解析到你的 IP 就 OK
```bash
ping -c 2 blog.你的域名.com
```
步骤 3：重新申请证书
```bash
certbot --nginx -d blog.你的域名.com -d www.你的域名.com
```
#### 方法2：用服务器 IP 直接配 HTTPS（简单测试用）
如果你暂时没有域名，直接用 IP 配 HTTPS，不用域名解析  
步骤 1：停止并删除之前的配置
```bash
# 停止证书申请
Ctrl + C
# 删除自动生成的Nginx配置（如果有）
rm -rf /etc/nginx/conf.d/000-default.conf
```
步骤 2：手动生成自签证书（测试用）
```bash
mkdir -p /etc/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
```
填信息时直接回车默认即可
步骤 3：配置 Nginx（替换成你的博客配置）
创建 /etc/nginx/conf.d/blog-ssl.conf：
```nginx
server {
    listen 443 ssl;
    server_name 8.163.108.4; # 你的服务器IP

    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;

    root /data/www/blog;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}

# 可选：HTTP自动跳HTTPS
server {
    listen 80;
    server_name 8.163.108.4;
    return 301 https://$host$request_uri;
}
```
步骤 4：生效配置
```bash
nginx -t && systemctl restart nginx
```
浏览器访问：https://8.163.108.4（信任不安全提示即可）
