# Prometheus + Grafana + Node_expoter + 邮件告警
## 一、前期环境准备 + 关闭防火墙 / 时间同步
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
## 二、安装 Node-exporter （被监控端采集）
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
## 三、安装 Prometheus 监控服务端
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
## 四、安装 Grafana 企业大屏展示
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
#### 配置数据源 → 选择 Prometheus  
##### 步骤 1：进入数据源配置页面
登录 Grafana 后，点击左侧菜单栏的 ⚙️ Configuration（配置） → Data sources（数据源）  
点击右上角的 Add data source（添加数据源）  
在列表中找到并选择 Prometheus  
##### 步骤 2：配置 Prometheus 连接信息
按以下关键配置填写：
| 配置项 |	填写内容	| 说明 |
Name	Prometheus-Linux	自定义名称，方便识别
URL	http://127.0.0.1:9090	Prometheus 服务地址（如果 Prometheus 不在本机，改成对应 IP + 端口）
Access	Server (default)	保持默认，由 Grafana 服务端访问 Prometheus
Scrape interval	保持默认或 15s	和 Prometheus 的 scrape_interval 保持一致
##### 步骤 3：测试连接并保存
拉到页面底部，点击 Save & test（保存并测试）  
出现 Data source is working 绿色提示，说明对接成功；  
如果报错，检查：Prometheus 是否正常运行、端口 9090 是否开放、URL 是否正确、防火墙 / 安全组是否放行。  
#### 导入 Linux 监控大屏模板（推荐模板号：1860）
这个模板是 Grafana 社区最通用的 node_exporter 监控模板，支持 CPU、内存、磁盘、网络等全维度监控。
##### 步骤 1：导入模板
点击左侧菜单栏的 Dashboards（仪表盘） → Import（导入）
在 Import via grafana.com 下方输入模板 ID：1860，点击 Load
也可以直接访问模板页面：Grafana Dashboard 1860，下载 JSON 文件后上传。
##### 步骤 2：配置导入参数
导入页面中，Prometheus 数据源 选择你刚才添加的 Prometheus-Linux
其他选项保持默认，点击 Import 完成导入。
##### 步骤 3：查看监控大屏
导入完成后，会自动跳转到 Linux 监控仪表盘，就能看到服务器的 CPU、内存、磁盘、网络、负载等指标数据了。
## 五、部署 Alertmanager + 企业邮件告警（核心亮点）
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

## 常见问题排查（必看）
### 一、仪表盘全是 “无数据”
检查 Prometheus 配置是否正确，node_exporter 的 scrape_configs 是否添加，且 Prometheus 能正常抓取 9100 端口；
访问 Prometheus UI：http://你的服务器IP:9090 → Status → Targets，看 linux-server 这个 job 的状态是否为 UP；
检查 Grafana 数据源配置是否正确，是否能正常连接 Prometheus。
部分面板无数据
确认 node_exporter 已安装并运行（端口 9100）；
部分磁盘、文件系统指标，需要 node_exporter 启动时开启对应采集器，默认一般是开启的，不用额外配置。
Grafana 访问不了
检查服务器安全组 / 防火墙是否开放 3000 端口；
检查 grafana-server 服务状态是否正常，日志是否报错：journalctl -u grafana-server -f。
地址填写：http://127.0.0.1:9090  
导入Linux 监控模版 ID：8919  
直接生成专业企业级 CPU / 内存 / 磁盘 / 负载大屏  
