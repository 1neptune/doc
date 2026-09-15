##### 配置主机名

```bash
hostnamectl set-hostname databuff
vi /etc/hosts
# 添加: 172.19.6.98 databuff
# 查看主机名
hostname
# 输出: databuff
```

##### 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
```

##### 配置yum源

```bash
# 配置yum源

mkdir -p /etc/yum.repos.d/backup
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup/

# Centos7 
cat > /etc/yum.repos.d/CentOS-vault.repo << 'EOF'
[base]
name=CentOS-7 - Base - Vault
baseurl=https://mirrors.aliyun.com/centos-vault/7.9.2009/os/$basearch/
gpgcheck=0
enabled=1

[updates]
name=CentOS-7 - Updates - Vault
baseurl=https://mirrors.aliyun.com/centos-vault/7.9.2009/updates/$basearch/
gpgcheck=0
enabled=1

[extras]
name=CentOS-7 - Extras - Vault
baseurl=https://mirrors.aliyun.com/centos-vault/7.9.2009/extras/$basearch/
gpgcheck=0
enabled=1

[centosplus]
name=CentOS-7 - Plus - Vault
baseurl=https://mirrors.aliyun.com/centos-vault/7.9.2009/centosplus/$basearch/
gpgcheck=0
enabled=0
EOF

yum clean all
yum makecache
```

##### 安装docker环境

```bash
# 安装docker 
yum install -y yum-utils
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
systemctl start docker
systemctl enable docker
docker version
# 输出 Docker Engine Version: 26.1.3
docker compose version
# 输出 Docker Compose version: v2.27.0
systemctl status docker

# 配置daemon.json（配置 网络下载docker hub 资源）
vim /etc/docker/daemon.json
{
  "dns": [ "8.8.8.8"],
  "registry-mirrors": [
    "https://docker.1panel.live",
    "https://docker.xuanyuan.me",
    "https://docker.1ms.run",
    "https://docker.m.daocloud.io"
  ],
  "bip": "192.168.1.1/24"
} 

systemctl daemon-reload
systemctl restart docker
systemctl status docker
```

##### 安装DataBuff环境

```bash
# 设置安装目录变量
vim ~/.bashrc
export APM_INSTALL_DIR=/data/databuff-ai-apm
source ~/.bashrc

# 安装
curl -fsSL https://databuff.ai/databuff/ai-apm-install.sh | bash

# 默认安装目录

默认路径: /opt/databuff-ai-apm
自定义安装路径： /data/databuff-ai-apm

databuff-ai-apm/
├── docker-compose.yml    # 主栈：Doris FE/BE + ingest + web
├── start.sh / stop.sh    # 推荐启停方式
├── env.sh                # 镜像版本（与 deploy/env.sh 一致）
├── data/                 # Doris 持久化（fe-meta、be-storage）
├── scripts/              # init-doris、pull-images、runtime 等
└── sql/                  # databuff.sql（首次启动导入）


### 输出
[install] 启动服务 ... 完成

========================================================
 安装完成
========================================================

  Web UI
    http://172.19.6.98:27403
  账号
    admin / Databuff@123
  Ingest
    http://172.19.6.98:4318/v1/traces

  安装目录
    /data/databuff-ai-apm
  启动
    cd /data/databuff-ai-apm && ./start.sh
  停止
    cd /data/databuff-ai-apm && ./stop.sh
  Demo造数
    curl -fsSL https://databuff.ai/databuff/ai-apm-demo-install.sh | bash

========================================================

# 升级
yum install -y python3
curl -fsSL https://databuff.ai/databuff/ai-apm-update.sh | bash
# 输出
========================================================
 DataBuff AI APM  在线升级
 保留 data/，不执行全新安装
========================================================

[update] 刷新本机 update.sh 与 scripts/ ...

========================================================
 DataBuff AI APM  升级 0.1.7 → 0.1.9
========================================================

[update] (1/6) 检查环境
[update] (1/6) 检查环境 ... 完成
[update] (2/6) 停止服务
[update] (2/6) 停止服务 ... 完成
[update] (3/6) 备份 data/
[update] (3/6) 备份 data/（默认跳过；需要时加 --backup） ... 跳过
[update] (4/6) 更新部署文件
[update] (4/6) 更新部署文件 ... 完成
[update] (5/6) 加载镜像
[pull-images] 加载离线镜像 (arch=amd64, apm=0.1.9, doris=4.1.1)
[pull-images] APM stack (ingest+web+demo)
[pull-images]   下载 ai-apm-stack-0.1.9-amd64.tar.gz ...
[pull-images]   ai-apm-stack-0.1.9-amd64.tar.gz: 19M / 622M (3%, 10s)
[pull-images]   ai-apm-stack-0.1.9-amd64.tar.gz: 102M / 622M (16%, 60s)

# 启停与重启
cd /opt/databuff-ai-apm
./start.sh    # 首次自动初始化 Doris 并导入表结构
./stop.sh     # 停止全部容器

# 重启单服务
docker compose restart ai-apm-web
docker compose restart ai-apm-ingest

# 卸载
cd /opt/databuff-ai-apm
./stop.sh
cd ..
rm -rf /opt/databuff-ai-apm

# 模型配置 

配置管理-模型配置
```

##### DataBuff 使用

###### OpenTelemetry Injector

```bash
# 检查被监测主机是否支持eBPF
ls /sys/kernel/btf/vmlinux

# 安装 Injector 及 Agent

# RHEL / CentOS / Rocky Linux (x86_64)
wget https://github.com/open-telemetry/opentelemetry-injector/releases/download/v0.9.0/opentelemetry-injector-0.9.0-1.x86_64.rpm
rpm -i opentelemetry-injector-0.9.0-1.x86_64.rpm


# 配置 OTLP 导出地址
vim /etc/opentelemetry/injector/default_env.conf

OTEL_SERVICE_NAME=A8V11
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_LOGS_EXPORTER=otlp
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_EXPORTER_OTLP_ENDPOINT=http://172.19.6.98:4317
OTEL_PROPAGATORS=tracecontext,baggage
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=test

# 启用 Injector
# 全局启用（需 root）对主机上所有支持的进程生效
echo /usr/lib/opentelemetry/libotelinject.so >> /etc/ld.so.preload

# 修改后需重启目标应用
# reboot
```

##### 排查案例

###### 功能接口存在慢响应

![](D:/marktext/images/2026-09-11-10-00-56-3bd20450a66d257327eed75fc01299a3.png)

###### databuff 查询对应功能接口

![](D:/marktext/images/2026-09-11-10-01-28-f1ae70fe8f0fc3d78d5a250e67c13044.png)

###### 查询对应慢SQL

![](D:/marktext/images/2026-09-10-17-16-43-image.png)
