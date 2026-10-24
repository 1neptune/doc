# NFS服务器

```bash
# 安装依赖

# ubuntu
sudo apt update
sudo apt install nfs-kernel-server -y

# rocky
sudo yum update -y
sudo yum install nfs-utils -y


# 创建共享目录

sudo mkdir -p /data/share
sudo chmod 777 /data/share

# 配置 NFS 共享
sudo vim /etc/exports
# 允许所有访问
/data/share *(rw,sync,no_root_squash,no_subtree_check)
# 允许指定IP
# 协同服务器、全文检索服务器
/data/share 192.168.30.54(rw,sync,no_root_squash,no_subtree_check) 192.168.30.55(rw,sync,no_root_squash,no_subtree_check) 192.168.30.59(rw,sync,no_root_squash,no_subtree_check)


# 导出共享目录
sudo exportfs -a


# 启动NFS服务
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
sudo systemctl status nfs-server


```

# NFS客户端

```bash

# 安装依赖

# ubuntu
sudo apt update 
sudo apt install nfs-common -y

# rocky
sudo yum update -y
sudo yum install nfs-utils -y


# 创建挂载点：
sudo mkdir -p /data/share
vim /etc/hosts
192.168.30.59 file
sudo mount -t nfs -o sync,noac 192.168.30.59:/data/share /data/share
sudo mount -t nfs -o sync,noac file:/data/share /data/share
# 检查mount结果
df -h

vim /etc/fstab
# 设置开机自动挂载
192.168.30.59:/data/share /data/share nfs sync,noac 0 0
file:/data/share /data/share nfs sync,noac 0 0
```


