# 阿里云ECS个人博客搭建
## 前置准备（先把阿里云试用 ECS 配置好）
### 1、ECS 必做配置
进入阿里云 ECS 实例 → 重置实例密码（root 账号密码，自己记牢）
重置密码后 重启 ECS 实例
复制保存：公网 IP、root 用户名、root 密码
### 2、放行阿里云安全组（企业最小权限原则）
ECS 控制台 → 网络与安全 → 安全组 → 配置入方向规则
| 协议 | 端口 | 授权对象 | 说明 |
|-----|------|---------|---------| 
| TCP | 22 | 本地电脑ip | 只允许自己 SSH 连接，防暴力破解 |
| TCP | 80 | 0.0.0.0/0 | 网站 HTTP 访问 |
| TCP | 443 | 0.0.0.0/0 | 只网站 HTTPS 访问 |
## 第一步：Xshell 连接阿里云 CentOS7.9 服务器
打开 Xshell → 文件 → 新建
名称：自定义（blog - 阿里云运维机）
协议：SSH
主机：ECS公网IP
端口号：22
点击连接，输入用户名：root，再输入重置的 root 密码
登录成功，进入命令行界面
## 第二步：企业级服务器基线初始化
### 2.1 修改主机名（企业规范）
```bash
hostnamectl set-hostname ops-blog-prod
## 2.2 配置时间同步（生产集群必须时间统一）
```bash
yum install -y chrony
systemctl start chrony
systemctl enable chrony
chronyc sources
### 2.3 更换阿里云 CentOS7 国内 YUM 源
```bash
yum install -y wget curl
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
yum clean all && yum makecache
### 2.4 创建普通运维用户 + 授予 sudo 权限（禁止长期用 root 操作业务）
企业规范：日常运维不用 root，最小权限
```bash
useradd opsuser
echo "Zcy20030812." | passwd opsuser --stdin
usermod -aG wheel opsuser
### 2.5 关闭系统防火墙 + SELinux（云服务器生产常规操作）
```bash
systemctl stop firewalld
systemctl disable firewalld

setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
## 第三步：安装企业基础依赖环境
后续 Nginx、PHP、数据库、编译依赖全部一次性装好
bash /
yum install -y gcc gcc-c++ pcre-devel zlib-devel libxml2-devel libcurl-devel libpng-devel openssl-devel
## 第四步：分层部署企业 Web 架构 Nginx + PHP-FPM + MariaDB
Nginx 静态反向代理 → PHP-FPM 动态解析 → MariaDB 数据层，三层分离，中小企业标准生产架构
### 4.1 部署 Nginx 并设置开机自启
bash / 
yum install -y nginx
systemctl start nginx
systemctl enable nginx
### 4.2 部署 PHP7.4 + PHP-FPM（独立进程，和 Nginx 解耦）
bash /
yum install -y epel-release remi-release
yum install -y php74 php74-fpm php74-mysqlnd php74-gd php74-curl php74-xml

systemctl start php-fpm
systemctl enable php-fpm
### 4.3 部署 MariaDB 数据库 + 生产安全初始化
bash /
yum install -y mariadb-server mariadb
systemctl start mariadb
systemctl enable mariadb

企业数据库安全加固（必做）
bash /
mysql_secure_installation
交互操作按企业规范选：
设置 root 数据库强密码
移除匿名用户 y
禁止 root 远程登录 y
删除测试库 y
刷新权限 y
## 第五步：创建企业规范目录结构（对标生产）
统一规划业务、日志、备份路径，不混乱乱放
bash /
-- 网站代码目录
mkdir -p /data/www/blog
-- 日志统一存放目录
mkdir -p /data/logs/nginx
-- 数据库/文件备份目录
mkdir -p /data/backup/mysql
## 第六步：企业 Git 代码上线流程（拒绝解压包上传）
模仿公司开发提交、运维拉取发布版本
### 6.1 安装 Git
bash /
yum install -y git
### 6.2 Git 拉取 WordPress 源码到业务目录
bash /
git clone https://gitee.com/liuxiaocai/wordpress.git /data/www/blog
### 6.3 权限最小化配置（生产安全规范）
bash /
chown -R nginx:nginx /data/www/blog
chmod -R 755 /data/www/blog
## 第七步：数据库生产级配置（不使用 root 建站）
### 7.1 进入数据库
bash /
mysql -uroot -p
### 7.2 创建独立业务库 + 独立业务账号（面试高频考点）
sql /
create database blog_prod default character set utf8mb4 collate utf8mb4_unicode_ci;
-- 仅允许本地应用连接，禁止外网
create user blog_app@localhost identified by 'Blog@Prod2026';
-- 最小权限授权
grant all privileges on blog_prod.* to blog_app@localhost;
flush privileges;
exit;
记录信息：库名：blog_prod账号：blog_app密码：Zcy20030812@
## 第八步：手写 Nginx 虚拟主机配置（企业运维核心能力）
### 8.1 新建站点配置文件
bash /
vim /etc/nginx/conf.d/blog.conf
复制粘贴下面配置（适配 PHP-FPM）：

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
esc → 输入 :wq 保存退出
### 8.2 检测 Nginx 配置 + 重启生效
bash /
nginx -t
systemctl restart nginx
## 第九步：浏览器初始化 WordPress 博客
浏览器访问：http://ECS公网IP
填写数据库信息：
数据库名：blog_prod
用户名：blog_app
密码：Zcy20030812@
数据库主机：localhost
设置博客管理员账号密码，完成安装
