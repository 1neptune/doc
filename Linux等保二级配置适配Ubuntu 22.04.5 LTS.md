# 身份鉴别

## 身份标识、鉴别、复杂度、定期更换

### 1. 密码有效期策略

```bash

sudo vi /etc/login.defs

PASS_MAX_DAYS 180
PASS_MIN_DAYS 1
PASS_WARN_AGE 28


sudo chage -M 180 <用户名>
sudo chage -m 1 <用户名>
sudo chage -W 28 <用户名>
sudo chage -l <用户名>

# 查看某个用户的密码策略
chage -l <用户名>
chage -l root
```

### 2. 密码复杂度

```bash
sudo apt install libpam-pwquality

sudo vi /etc/pam.d/common-password

# 在 password 段配置密码质量要求：
# try_first_pass：先使用用户输入的密码尝试
# retry=5：密码不符合要求时允许重试 5 次
# local_users_only：只对本地用户生效
# minlen=8：密码最小长度 8 位
# dcredit=-1：至少包含 1 个数字
# ucredit=-1：至少包含 1 个大写字母
# lcredit=-1：至少包含 1 个小写字母
# ocredit=-1：至少包含 1 个特殊字符
# enforce_for_root：root 修改密码时也强制检查
password    requisite    pam_pwquality.so retry=5 minlen=8 dcredit=-1 ucredit=-1 lcredit=-1 ocredit=-1 enforce_for_root

useradd -m test
passwd test
```

## 登录失败处理、超时退出

### 登录失败锁定

```bash
sudo vi /etc/security/faillock.conf
deny = 5
unlock_time = 300
fail_interval = 300
even_deny_root
root_unlock_time = 300

sudo vi /etc/pam.d/common-auth
# here are the per-package modules (the "Primary" block)
auth    requisite                       pam_faillock.so preauth
auth    [success=1 default=ignore]      pam_unix.so nullok
auth    [default=die]                   pam_faillock.so authfail
auth    sufficient                      pam_faillock.so authsucc

sudo vi /etc/pam.d/common-account
# 放第一行
# here are the per-package modules (the "Primary" block)
account required                        pam_faillock.so 

sudo vi /etc/ssh/sshd_config
UsePAM yes
KbdInteractiveAuthentication yes
sudo systemctl restart ssh

grep faillock /etc/pam.d/common-auth
auth    required    pam_faillock.so preauth
auth    [default=die]    pam_faillock.so authfail
auth    sufficient    pam_faillock.so authsucc


faillock --user <用户名>
sudo faillock --user <用户名> --reset
```

### 登录超时自动退出

```bash
echo 'readonly TMOUT=600' | sudo tee /etc/profile.d/timeout.sh
echo 'export TMOUT' | sudo tee -a /etc/profile.d/timeout.sh
source /etc/profile.d/timeout.sh
```

### 远程管理防窃听

```bash

systemctl status ssh
sudo systemctl enable --now ssh


dpkg -l | grep telnet

sudo apt remove telnet
```

# 二、访问控制

## 重要文件权限

```bash
ls -l /etc/passwd /etc/shadow /etc/group /etc/crontab

sudo chmod 644 /etc/passwd
sudo chmod 400 /etc/shadow
sudo chmod 644 /etc/group
sudo chmod 644 /etc/crontab
```

## 禁止 root 远程登录

```bash
sudo vi /etc/ssh/sshd_config


PermitRootLogin no

sudo sshd -t
sudo systemctl restart ssh
```

## 禁用多余默认账户

```bash
grep -v '/usr/sbin/nologin\|/bin/false' /etc/passwd

sudo usermod -s /usr/sbin/nologin sync
sudo usermod -s /usr/sbin/nologin halt
sudo usermod -s /usr/sbin/nologin shutdown

usermod -s /sbin/nologin test
userdel -r  test
```

## 最小权限与 sudo 配置

```bash
sudo usermod -aG sudo <用户名>

sudo -l -U <用户名>
```

## 配置 sudo

```bash
sudo grep sudo /var/log/auth.log
```

# 三、安全审计

## 启用审计服务

```bash
sudo apt update
sudo apt install -y auditd audispd-plugins


sudo systemctl enable --now auditd

sudo auditctl -s
```

```bash
sudo vi /etc/rsyslog.conf

sudo vi /etc/rsyslog.d/50-default.conf

authpriv.*    @<日志服务器IP>
*.err        @<日志服务器IP>

sudo systemctl restart rsyslog
```

```bash
sudo vi /etc/audit/auditd.conf
log_file = /var/log/audit/audit.log
max_log_file = 100
num_logs = 7

sudo vi /etc/audit/rules.d/99-custom.rules
-w /etc/passwd -p wa -k passwd_changes
-w /etc/sudoers -p wa -k sudoers_changes
-w /data/payslip/.env -p wa -k env_changes
-w /data/yyzx/.env -p wa -k env_changes
-w /data/payslip/docker-compose.yml -p wa -k docker_changes
-w /data/yyzx/docker-compose.yml -p wa -k docker_changes


sudo augenrules --load
sudo systemctl restart auditd
```

## 审计记录检查

```bash
sudo ausearch -i
sudo ausearch -k passwd_changes -i
```

## 日志权限与轮转

```bash
sudo chmod 640 /var/log/auth.log
sudo chmod 644 /var/log/audit/audit.log
sudo chmod 640 /var/log/syslog


sudo vi /etc/logrotate.conf
daily
rotate 180
missingok
compress
sudo logrotate -d /etc/logrotate.conf
```

# 四、入侵防范

## 最小安装检查

```bash
dpkg -l
apt list --installed
sudo apt remove <包名>
sudo apt autoremove
```

## 关闭多余服务与端口

```bash
# 查看开机自启服务
systemctl list-unit-files | grep enabled

# 查看服务状态
systemctl status <服务名>.service

# 停止多余服务
systemctl stop <服务名>.service

# 禁用多余服务
systemctl disable <服务名>.service

# 查看监听端口
ss -tulnp

# 查看防火墙规则
sudo ufw status numbered
sudo ufw allow ssh
# 移除服务
sudo ufw delete <编号>
sudo ufw delete allow <端口/服务>

ipset list
iptables -L INPUT -n --line-numbers
iptables -L FORWARD -n --line-numbers
iptables -L DOCKER -n --line-numbers
iptables -L DOCKER-USER -n --line-numbers

# 移除规则
iptables -D INPUT 2
iptables -D DOCKER-USER 2

# 保存规则
service iptables save

# 重启服务
systemctl restart iptables
```

## 限制管理终端

```bash
sudo vi /etc/ssh/sshd_config

# 只允许指定用户 SSH 登录
AllowUsers payslip

# 只允许指定IP SSH 登录
AllowUsers *@192.168.3.19

# 只允许 payslip 从 192.168.3.19 登录
AllowUsers payslip@192.168.3.19

# 限制 SSH 最大认证尝试次数
MaxAuthTries 5

# 禁止空密码登录
PermitEmptyPasswords no


systemctl restart sshd
```

## 漏洞修补

```bash

sudo apt update
sudo apt list --upgradable
# 仅列出安全更新
apt list --upgradable 2>/dev/null | grep -i security

sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
lsb_release -a
```

# 五、恶意代码防范

```bash
# 检查是否安装 ClamAV
rpm -qa | grep clamav

# 安装 ClamAV
sudo apt update
sudo apt install -y clamav clamav-daemon

sudo systemctl stop clamav-freshclam
# 更新病毒库
clamscan --version
sudo freshclam
sudo systemctl start clamav-freshclam
sudo systemctl enable clamav-freshclam


# 扫描目录
clamscan -r /data

sudo clamscan -r /data -l /var/log/clamav/scan.log

# 定期升级扫描
mkdir -p /var/log/clamav

vi /etc/logrotate.d/clamav
/var/log/clamav/*.log {
    daily
    rotate 180
    create 0640 root root
    dateext
    dateformat -%Y%m%d
    missingok
    compress
    delaycompress
    notifempty
    copytruncate
}

logrotate -d /etc/logrotate.d/clamav

# 计划任务
crontab -e

0 1 * * * /usr/bin/freshclam --quiet
0 2 * * * /usr/bin/clamscan -r /data -l /var/log/clamav/scan.log
```

# 六、可信验证

```bash
# 查看 TPM 设备
ls /dev/tpm*

cat /sys/class/tpm/tpm0/tpm_version_major 2>/dev/null || echo "未检测到 TPM 设备"
mokutil --sb-state 2>/dev/null || echo "mokutil 不可用"
dmesg | grep -i "secure boot" || echo "系统未启用安全启动或内核不支持"
```

# 七、数据备份恢复

## 本地备份

```bash
# 创建备份目录
mkdir -p /backup/payslip

# 备份 /data/payslip
tar -czvf /backup/payslip/payslip_$(date +%F).tar.gz /data/payslip

# 查看备份
ls -lh /backup/payslip/

# 恢复测试
mkdir -p /data/paysliptest
tar -xzvf /backup/payslip/payslip_$(date +%F).tar.gz -C /data/paysliptest
```

## 异地备份

```bash
# rsync 同步到异地
rsync -avz --delete /data/payslip/ root@<异地IP>:/backup/payslip/

# 定时任务
crontab -e

# 每天凌晨 2 点同步
0 2 * * * rsync -avz --delete /data/payslip/ root@<异地IP>:/backup/payslip/
```
