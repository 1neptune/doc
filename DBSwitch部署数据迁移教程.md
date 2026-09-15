##### 安装Docker

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

##### 部署DBSwitch

```bash
# DBSwitch
dbswitch工具提供源端数据库向目的端数据的迁移同步功能，包括全量和变化量方式。
结构迁移：
字段类型、主键信息、建表语句等的转换，并生成建表SQL语句。支持基于正则表达式转换的表名与字段名映射转换。

数据迁移：
基于JDBC的分批次读取源端数据库数据，并基于insert/copy方式将数据分批写入目的数据库。支持有主键表的"变化量"同步 （变化数据计算Change Data Calculate）功能。

Gitee地址：https://gitee.com/inrgihc/dbswitch

# 创建 docker-compose.yml 文件
mkdir -p /docker/dbswtich
vim  /docker/dbswtich/docker-compose.yml

services:
  dbswitch:
    image: registry.cn-hangzhou.aliyuncs.com/inrgihc/dbswitch:2.0.1
    container_name: dbswitch
    privileged: true
    restart: always
    environment:
      - TZ=Asia/Shanghai
      - DBTYPE=h2
    volumes:
      - /tmp:/tmp
    ports:
      - "9088:9088"
# 启动服务
cd /docker/dbswtich
docker compose up -d
# 查看容器状态
docker ps | grep dbswitch
# 确认端口监听
netstat -tunlp | grep 9088
# 实时查看日志
docker logs -f dbswitch
# 进入容器内部
docker exec -it dbswitch bash
# 重启服务
docker compose restart
# 停止服务
docker compose down
```

##### 访问Web控制台

```bash
浏览器访问：http://192.168.30.50:9088

默认账号：admin

默认密码：123456

修改密码: Db@123456
```

##### 创建数据源

<img src="file:///D:/marktext/images/2026-08-27-11-56-41-image.png" title="" alt="" width="660">

##### 创建迁移任务

![](D:/marktext/images/2026-08-27-12-00-05-image.png)

![](D:/marktext/images/2026-08-27-12-00-44-image.png)

![](D:/marktext/images/2026-08-27-15-21-22-image.png)
