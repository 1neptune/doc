##### Windows 配置 OpenSSH

```bash

Subsystem	sftp	internal-sftp

ChrootDirectory %u
AllowGroups Administrators

Match User A8
ChrootDirectory D:\UFSeeyon\A8\Enterprise\base\upload
ForceCommand internal-sftp
AllowTcpForwarding no
AllowAgentForwarding no
X11Forwarding no
PermitTTY no


```



##### Linux 使用 sftp  传输文件



```bash
# 1. 连接
sftp -P 38022 A8@192.168.30.46

# 2. 输入密码后进入 sftp>

# 3. 此时你的远程根目录就是 upload 目录
sftp> pwd
# 输出 /   实际对应D:\UFSeeyon\A8\Enterprise\base\upload

# 4. 切换到本地 /data/upload
sftp> lcd /data/upload

# 5. 查看本地文件
sftp> lls

# 6. 上传所有文件 (递归)
sftp> put -r *
```

  
