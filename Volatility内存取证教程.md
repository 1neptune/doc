

##### 安装volatility3

```bash
python3 -m venv ~/vol3

source ~/vol3/bin/activate

python -m ensurepip --upgrade

pip install volatility3

pip install yara-python pycryptodome

┌──(vol3)(root㉿kali)-[~]
└─# vol -h
Volatility 3 Framework 2.28.2
```

##### 内存取证

```bash
# VMware 虚拟机提取内存

vmss2core.exe -W8 win10.vmsn win10.vmem -> memory.dmp

# Windows  

# DumpIt 内存快照工具
https://www.magnetforensics.com/resources/magnet-dumpit-for-windows/

scp memory.dmp root@192.168.21.242:~/memory.lime

# 确认系统信息
vol -f memory.dmp windows.info

# 进程列表与隐藏进程对比

# 正常进程列表
vol -f memory.dmp windows.pslist

# 扫描隐藏/被摘链的进程
vol -f memory.dmp windows.psscan

# 查看进程树
vol -f memory.dmp windows.pstree

# 查看进程命令行
vol -f memory.dmp windows.cmdline

# 查看网络连接C2
vol -f memory.dmp windows.netscan

# 恶意代码注入检测
vol -f memory.dmp windows.malware.malfind

#注册表与持久化

# 查看服务列表
vol -f memory.dmp windows.svclist

# 查看计划任务
vol -f memory.dmp windows.registry.scheduled_tasks


# Linux

https://github.com/microsoft/avml/releases

wget https://github.com/microsoft/avml/releases/download/v0.20.0/avml

chmod +x avml

# 抓取内存快照到本地文件

./avml acquire memory.lime

# 压缩内存快照大小

./avml acquire --compress memory.lime.compressed

# 传输内存快照
scp memory.lime root@192.168.21.242:~/memory.lime


# 远程传输内存快照不落盘
nc -l -p 9000 > ~/memory.lime
./avml stream tcp 192.168.21.242:9000

# 确认系统信息
vol -f memory.lime banners

# 进程列表与树
vol -f memory.lime linux.pslist.PsList
vol -f memory.lime linux.pstree.PsTree

# 查看进程命令行参数
vol -f memory.lime linux.psaux.PsAux

# 网络连接C2
vol -f memory.lime linux.sockstat.Sockstat

# 恶意代码注入检测
vol -f memory.lime linux.malware.malfind.Malfind

# 恢复 bash 命令历史
vol -f memory.lime linux.bash.Bash
```
