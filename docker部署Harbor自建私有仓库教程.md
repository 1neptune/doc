##### 安装docker环境

```bash
# 测试环境
CentOS Linux release 7.9.2009 (Core)
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
  "dns": ["8.8.8.8", "223.5.5.5"],
  "insecure-registries": ["172.19.6.98"],
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me"
  ]
}
systemctl daemon-reload
systemctl restart docker
systemctl status docker
```

##### Harbor 自建私有仓库

```bash
more docker-compose.yml 
services:
  web-nginx:
    build: .
    image: 172.19.6.98/web/web-nginx:v2
    container_name: web-nginx
    ports:
      - "8090:80"
    restart: always
    networks:
      - nginxnet

networks:
  nginxnet:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.2.0/24
          gateway: 192.168.2.1more docker-compose.yml 
services:
  web-nginx:
    build: .
    image: 172.19.6.98/web/web-nginx:v2
    container_name: web-nginx
    ports:
      - "8090:80"
    restart: always
    networks:
      - nginxnet

networks:
  nginxnet:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.2.0/24
          gateway: 192.168.2.1# 下载与解压
https://github.com/goharbor/harbor/releases/
tar xvf harbor-offline-installer-v2.15.2.tgz
cd harbor
# 配置 harbor.yml
cp harbor.yml.tmpl harbor.yml
vim harbor.yml

配置项    修改说明
hostname    改成你服务器的 IP 或域名（如 172.19.6.98）
harbor_admin_password：    设置管理员密码（默认 Harbor12345）
https     暂时不用 HTTPS，整段注释掉（前面加 #）
password: Harbor@123456 设置数据库密码（默认 root123）
data_volume: /data/harbor

# 安装启动

# 生成配置文件
./prepare

# 执行安装
./install.sh

# 停止容器
docker compose down -v


# 客户端配置
vim /etc/docker/daemon.json
{
  "insecure-registries": ["172.19.6.98"]
}

systemctl restart docker


# 推送与拉取镜像

# 在 Harbor Web 界面创建一个项目 web

http://172.19.6.98/harbor/projects

admin/Harbor@123456

# Dockerfile 构建
mkdir web-nginx && cd web-nginx

cat > index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>My Custom Nginx</title>
</head>
<body>
    <h1>Hello from my custom nginx!</h1>
    <p>This is a custom index page.</p>
</body>
</html>
EOF

# 编写 Dockerfile

cat > Dockerfile <<'EOF'
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
EOF



# 通过Dockerfile构建Harbor格式镜像名
docker build -t 172.19.6.98/web/web-nginx:v1 .

cat > Dockerfile <<'EOF'
FROM 172.19.6.98/web/web-nginx:v1
COPY ./html /usr/share/nginx/html
EOF

docker build -t 172.19.6.98/web/web-nginx:v2 .
docker push 172.19.6.98/web/web-nginx:v2

docker pull 172.19.6.98/web/web-nginx:v2


# 通过commit构建Harbor格式镜像名
docker exec -it web-nginx bash
cat > /usr/share/nginx/html/test.html <<'EOF'
<!DOCTYPE html>
<html><body><h1>test</h1></body></html>
EOF
docker commit web-nginx 172.19.6.98/web/web-nginx:v2


# 登录
docker login 172.19.6.98

admin/Harbor@123456

# 凭证
cat /root/.docker/config.json

# 推送
docker push 172.19.6.98/web/web-nginx:v1

# 删除镜像测试
docker rmi 172.19.6.98/web/web-nginx:v1
#拉取
docker pull 172.19.6.98/web/web-nginx:v1


cat > docker-compose.yml <<'EOF'
services:
  web-nginx:
    build: .
    image: 172.19.6.98/web/web-nginx:v1
    container_name: web-nginx
    ports:
      - "8090:80"
    restart: always
EOF

# 分配指定地址
services:
  web-nginx:
    build: .
    image: 172.19.6.98/web/web-nginx:v2
    container_name: web-nginx
    ports:
      - "8090:80"
    restart: always
    networks:
      - nginxnet

networks:
  nginxnet:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.2.0/24
          gateway: 192.168.2.1
          
docker stop web-nginx
docker rm web-nginx

# 多版本部署
services:
  web-nginx-v1:
    image: 172.19.6.98/web/web-nginx:v1
    container_name: web-nginx-v1
    ports:
      - "8091:80"
    restart: always
    networks:
      nginxnet:
        ipv4_address: 192.168.1.10

  web-nginx-v2:
    image: 172.19.6.98/web/web-nginx:v2
    container_name: web-nginx-v2
    ports:
      - "8092:80"
    restart: always
    networks:
      nginxnet:
        ipv4_address: 192.168.1.11

networks:
  nginxnet:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.1.0/24
          gateway: 192.168.1.1
          


docker compose up -d
docker compose ps




# 查看每个容器具体网段
docker ps -q | xargs -I {} docker inspect {} \
  --format '{{.Name}} -> {{range $k,$v := .NetworkSettings.Networks}}{{$k}}:{{$v.IPAddress}} {{end}}'

 # 输出内容
/web-nginx -> web-nginx_default:172.21.0.2 
/nginx -> harbor_harbor:172.18.0.10 
/harbor-jobservice -> harbor_harbor:172.18.0.9 
/harbor-core -> harbor_harbor:172.18.0.8 
/harbor-db -> harbor_harbor:172.18.0.6 
/registry -> harbor_harbor:172.18.0.3 
/registryctl -> harbor_harbor:172.18.0.7 
/redis -> harbor_harbor:172.18.0.5 
/harbor-portal -> harbor_harbor:172.18.0.4 
/harbor-log -> harbor_harbor:172.18.0.2 
/ai-apm-web -> databuff-ai-apm:172.20.80.4 
/ai-apm-ingest -> databuff-ai-apm:172.20.80.3 
/ai-apm-doris-be -> databuff-ai-apm:172.20.80.5 
/ai-apm-doris-fe -> databuff-ai-apm:172.20.80.2 
```
