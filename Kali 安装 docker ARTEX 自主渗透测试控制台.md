# 卸载可能存在的旧版本

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-doc podman-docker containerd runc | cut -f1)
```

# 安装依赖并添加 Docker 的 GPG 密钥

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

# 添加 Docker 仓库

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian trixie stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

# 安装 Docker 及 Compose 插件

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

# 启动 Docker 并设置开机自启

```bash
systemctl enable --now docker
docker --version      
Docker version 29.9.0, build f415da8
docker compose version
Docker Compose version v5.6.0
```

# 安装ARTEX 自主渗透测试控制台

```bash
git clone https://github.com/mhtsec/ARTEX.git
cd ARTEX
./install.sh
```


