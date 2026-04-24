# Prometheus + Grafana + Node_expoter + 邮件告警
## 前期环境准备 + 关闭防火墙 / 时间同步
### 1. 安装基础依赖
```bash
yum install -y wget curl vim net-tools ntpdate unzip
```
### 2. 时间强制同步（监控集群必须时间一致）
```bash
ntpdate ntp.aliyun.com
timedatectl set-timezone Asia/Shanghai
```
### 3. 阿里云安全组提前放行端口
```plaintext
9090（Prometheus）
3000（Grafana面板）
9100（Node-exporter）
```
## 第一步：安装 Node-exporter （被监控端采集）
作用：采集服务器硬件、系统指标
```bash
# 下载二进制包（企业生产统一二进制部署，不用yum）
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz

tar zxf node_exporter-1.8.2.linux-amd64.tar.gz
mv node_exporter-1.8.2.linux-amd64 /usr/local/node_exporter

# 配置系统开机自启systemd（企业规范）
cat > /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=node_exporter
After=network.target

[Service]
ExecStart=/usr/local/node_exporter/node_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```
启动并开机自启：
```bash
systemctl daemon-reload
systemctl start node_exporter
systemctl enable node_exporter
```
测试：ip:9100 能访问就是采集正常
## 第二步：安装 Prometheus 监控服务端
### 1. 二进制安装（企业生产标准）
```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-amd64.tar.gz

tar zxf prometheus-2.53.0.linux-amd64.tar.gz
mv prometheus-2.53.0.linux-amd64 /usr/local/prometheus
```
### 2. 修改配置文件，加入本机监控 + 后续告警规则
```bash
vim /usr/local/prometheus/prometheus.yml
```
替换为下面内容：
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'linux-server'
    static_configs:
    - targets: ['你的ECS公网IP:9100']
```
### 3. 配置 systemd 开机自启
```bash
cat > /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=prometheus
After=network.target

[Service]
ExecStart=/usr/local/prometheus/prometheus \
--config.file=/usr/local/prometheus/prometheus.yml \
--storage.tsdb.path=/usr/local/prometheus/data
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```
启动：
```bash
systemctl daemon-reload
systemctl start prometheus
systemctl enable prometheus
```
访问：ip:9090 Prometheus 后台正常
## 第三步：安装 Grafana 企业大屏展示
### 1. 安装 Grafana
```bash
wget https://dl.grafana.com/oss/release/grafana-10.2.3-1.x86_64.rpm
yum install -y grafana-10.2.3-1.x86_64.rpm
```
### 2. 启动 + 开机自启
```bash
systemctl start grafana-server
systemctl enable grafana-server
```
访问：ip:3000默认账号密码：admin / admin
### 3. Grafana 对接 Prometheus + 导入 Linux 大屏模板
配置数据源 → 选择 Prometheus
地址填写：http://127.0.0.1:9090
导入Linux 监控模版 ID：8919
直接生成专业企业级 CPU / 内存 / 磁盘 / 负载大屏
## 第四步：部署 Alertmanager + 企业邮件告警（核心亮点）
实现：CPU 过高、磁盘快满、服务器挂掉自动发你邮箱
### 1. 安装 Alertmanager
```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz

tar zxf alertmanager-0.27.0.linux-amd64.tar.gz
mv alertmanager-0.27.0.linux-amd64 /usr/local/alertmanager
```
### 2. 配置邮箱告警（用 QQ 邮箱 / 163 邮箱）
```bash
vim /usr/local/alertmanager/alertmanager.yml
```
粘贴（改成自己的接收邮箱 + 发件邮箱授权码）：
```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 3h
  receiver: 'email-notify'

receivers:
- name: 'email-notify'
  email_configs:
  - to: 接收告警的邮箱@qq.com
    from: 发送邮箱@qq.com
    smtp_server: smtp.qq.com:465
    smtp_auth_username: 发送邮箱@qq.com
    smtp_auth_password: 你的邮箱授权码
    smtp_require_tls: false
```
### 3. 配置 Prometheus 告警规则 + 触发阈值
CPU 使用率超过 80%、磁盘使用率超过 85%、服务器宕机 立刻告警
### 4. 启动 Alertmanager
```bash
nohup /usr/local/alertmanager/alertmanager --storage.path=/usr/local/alertmanager/data >/dev/null 2>&1 &
```
