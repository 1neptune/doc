# 身份鉴别

## 身份标识、鉴别、复杂度、定期更换

### 1. 密码有效期策略

```bash
# 编辑 login.defs
vi /etc/login.defs

# 密码最长使用 180 天，超过后必须修改
PASS_MAX_DAYS 180

# 密码最短使用 1 天，防止用户当天反复改回旧密码
PASS_MIN_DAYS 1

# 密码过期前 28 天开始提醒用户
PASS_WARN_AGE 28


# 将已有用户密码最长使用期限设为 180 天
chage -M 180 <用户名>

# 将已有用户密码最短修改间隔设为 1 天
chage -m 1 <用户名>

# 将已有用户密码过期前提醒天数设为 28 天
chage -W 28 <用户名>

# 查看某个用户的密码策略
chage -l <用户名>
chage -l root
```

### 2. 密码复杂度

```bash
# 编辑 PAM 密码策略文件
vi /etc/pam.d/system-auth
vi /etc/pam.d/password-auth

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
password    requisite    pam_pwquality.so try_first_pass retry=5 local_users_only minlen=8 dcredit=-1 ucredit=-1 lcredit=-1 ocredit=-1 enforce_for_root

useradd -m test
passwd test
```

## 登录失败处理、超时退出

### 登录失败锁定

```bash
ls /lib64/security/ | grep faillock
# 启用 faillock
authconfig --enablefaillock --faillockargs="deny=5 unlock_time=300 fail_interval=300 even_deny_root root_unlock_time=300" --update
# 验证启用状态
grep -i faillock /etc/sysconfig/authconfig
faillock --user oracle
# 重置          
faillock --user oracle --reset 
```

### 登录超时自动退出

```bash
# 创建超时配置文件
echo 'readonly TMOUT=600' > /etc/profile.d/timeout.sh

# 导出 TMOUT 变量，600 秒无操作后自动退出
echo 'export TMOUT' >> /etc/profile.d/timeout.sh

# 立即加载配置
source /etc/profile.d/timeout.sh
```

### 远程管理防窃听

```bash
# 查看 SSH 服务状态
systemctl status sshd

# 确认 SSH 已启用
systemctl enable --now sshd

# 检查是否安装 telnet
rpm -qa | grep telnet

# 如有 telnet，停止并禁用
systemctl stop telnet.socket
systemctl disable telnet.socket
```

# 二、访问控制

## 重要文件权限

```bash
# 查看 /etc/passwd 权限
ls -l /etc/passwd

# 查看 /etc/shadow 权限
ls -l /etc/shadow

# 查看 /etc/group 权限
ls -l /etc/group

# 查看 /etc/crontab 权限
ls -l /etc/crontab

# /etc/passwd 设为 644
chmod 644 /etc/passwd

# /etc/shadow 设为 400
chmod 400 /etc/shadow

# /etc/group 设为 644
chmod 644 /etc/group

# /etc/crontab 设为 644
chmod 644 /etc/crontab
```

## 禁止 root 远程登录

```bash
vi /etc/ssh/sshd_config

# 禁止 root 用户通过 SSH 远程登录
PermitRootLogin no

# 检查语法
sshd -t

# 重启 SSH
systemctl restart sshd
```

## 禁用多余默认账户

```bash
# 查看所有可登录用户
grep -v '/sbin/nologin\|/bin/false' /etc/passwd

# 将 sync 用户 shell 改为 nologin
usermod -s /sbin/nologin sync

# 将 halt 用户 shell 改为 nologin
usermod -s /sbin/nologin halt

# 将 shutdown 用户 shell 改为 nologin
usermod -s /sbin/nologin shutdown

usermod -s /sbin/nologin test

userdel -r vncuser
```

## 最小权限与 sudo 配置

```bash
# /etc/pam.d/su 控制 su 命令的认证行为
vi /etc/pam.d/su

# 取消注释
auth required pam_wheel.so use_uid

# 将用户加入 wheel 组
usermod -aG wheel <用户名>

------
usermod -aG wheel payslip
```

## 配置 sudo

```bash
# 安全编辑 sudoers
visudo

# 允许 wheel 组执行所有命令
%wheel ALL=(ALL) ALL

# sudo 命令执行记录
grep sudo /var/log/secure
```

# 三、安全审计

## 启用审计服务

```bash
# 查看 auditd 状态
systemctl status auditd

ps -aux | grep syslog

ps -aux | grep rsyslog

ps -aux | grep audit

# 启用并启动 auditd
systemctl enable --now auditd

yum install -y audit audit-libs
```

```bash
vi /etc/rsyslog.conf

# 记录 info 及以上日志
*.info;mail.none;authpriv.none;cron.none    /var/log/messages

# 认证日志
authpriv.*    /var/log/secure

# 邮件日志
mail.*    -/var/log/maillog

# cron 日志
cron.*    /var/log/cron

# 紧急日志
*.emerg    :omusrmsg:*

local7.*   /var/log/boot.log


# 日志转发到日志审计服务器
authpriv.*      @192.168.3.91
*.err   @192.168.3.91

# 重启生效
systemctl restart rsyslog
```

```bash
vi /etc/audit/auditd.conf

# 存储地址
log_file = /var/log/audit/audit.log

# 单个日志文件最大 100 MB
max_log_file = 100

# 最多保留 7 个轮转文件
num_logs = 7

service auditd restart


# 审计规则的配置文件

vi /etc/audit/rules.d/audit.rules



# 监控 /etc/passwd 变更
-w /etc/passwd -p wa -k passwd_changes



# 监控 /etc/sudoers 变更
-w /etc/sudoers -p wa -k sudoers_changes

# 监控 .env 变更
-w /data/payslip/.env -p wa -k env_changes
-w /data/yyzx/.env -p wa -k env_changes

# 监控 docker-compose.yml 变更
-w /data/payslip/docker-compose.yml -p wa -k docker_changes
-w /data/yyzx/docker-compose.yml -p wa -k docker_changes
```

## 审计记录检查

```bash
# 查看审计日志
ausearch -i

vipw -s
vipw -p
vipw -g
Usage: vipw [options]

Options:
  -g, --group                   edit group database
  -h, --help                    display this help message and exit
  -p, --passwd                  edit passwd database
  -q, --quiet                   quiet mode
  -R, --root CHROOT_DIR         directory to chroot into
  -s, --shadow                  edit shadow or gshadow database
# 查看指定 key 的审计记录
ausearch -k passwd_changes -i
```

## 日志权限与轮转

```bash
# 查看日志权限
ls -l /var/log/secure
ls -l /var/log/messages
ls -l /var/log/cron
ls -l /var/log/audit/audit.log

# secure 设为 644
chmod 644 /var/log/secure

# audit.log 设为 644
chmod 644 /var/log/audit/audit.log

# messages 设为 640
chmod 640 /var/log/messages

# cron 设为 640
chmod 640 /var/log/cron

vim /etc/logrotate.conf

# 每天轮转
daily

# 保留 180 份（约 6 个月）
rotate 180

# 文件不存在不报错
missingok

# 压缩旧日志
compress

# 验证
logrotate -d /etc/logrotate.conf
```

# 四、入侵防范

## 最小安装检查

```bash
# 查看已安装的包
rpm -qa

# 查看已安装的 yum 包
yum list installed

# 移除 yum 包
yum remove <包名>
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
firewall-cmd --permanent --list-rich-rules
firewall-cmd --list-ports
firewall-cmd --permanent --list-all

# 移除服务
firewall-cmd --permanent --remove-service=telnet
firewall-cmd --reload

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
vi /etc/ssh/sshd_config

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
# 查看安全补丁
yum updateinfo list security

# 安装安全更新
yum update --security

# 查看系统版本
cat /etc/redhat-release
```

# 五、恶意代码防范

```bash
# 检查是否安装 ClamAV
rpm -qa | grep clamav

# 安装 ClamAV
yum install -y epel-release
yum install -y clamav clamav-update

# 更新 ClamAV
yum update clamav clamav-update

# 卸载 ClamAV
yum remove clamav clamav-update

# 更新病毒库
clamscan --version
freshclam

# 扫描目录
clamscan -r /data

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
30 1 * * 1 /usr/bin/yum update -y clamav clamav-update >> /var/log/clamav/update.log 2>&1
0 2 * * * /usr/bin/clamscan -r /data -l /var/log/clamav/scan.log
```

# 六、可信验证

```bash
# 查看 TPM 设备
ls /dev/tpm*

# 查看 TPM 版本
cat /sys/class/tpm/tpm0/tpm_version_major

# 查看安全启动状态（CentOS 7 通常不支持 mokutil，视硬件而定）
mokutil --sb-state 2>/dev/null || echo "mokutil 不可用，跳过安全启动检查"

# 检查 UEFI 安全启动（备用方法）
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
