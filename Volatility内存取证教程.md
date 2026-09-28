

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

scp memory.dmp root@192.168.21.242:~/memory.dmp

# 确认系统信息
vol -f memory.dmp windows.info > /tmp/windows/info.txt

# 输出
┌──(vol3)(root㉿kali)-[~]
└─# vol -f memory.dmp windows.info   
Volatility 3 Framework 2.28.2
Progress:  100.00               PDB scanning finished                                                                                              
Variable        Value

Kernel Base     0xf80641800000
DTB     0x1aa000
Symbols file:///root/vol3/lib/python3.14/site-packages/volatility3/symbols/windows/ntkrnlmp.pdb/F57E740B088E5056E8AF0772F1CC5BEB-1.json.xz
Is64Bit True
IsPAE   False
layer_name      0 WindowsIntel32e
memory_layer    1 WindowsCrashDump64Layer
base_layer      2 FileLayer
KdVersionBlock  0xf8064240f400
Major/Minor     15.19041
MachineType     34404
KeNumberProcessors      4
SystemTime      2026-09-28 02:13:48+00:00
NtSystemRoot    C:\Windows
NtProductType   NtProductWinNt
NtMajorVersion  10
NtMinorVersion  0
PE MajorOperatingSystemVersion  10
PE MinorOperatingSystemVersion  0
PE Machine      34404
PE TimeDateStamp        Sat Feb  2 23:04:03 1985

# 进程列表与隐藏进程对比

# 正常进程列表
vol -f memory.dmp windows.pslist

vol -f memory.dmp -r json windows.pslist > /tmp/windows/pslist.json

# 扫描隐藏/被摘链的进程
vol -f memory.dmp windows.psscan

vol -f memory.dmp -r json windows.psscan > /tmp/windows/psscan.json

# 查看进程树
vol -f memory.dmp windows.pstree

vol -f memory.dmp -r json windows.pstree > /tmp/windows/pstree.json


# 查看进程命令行
vol -f memory.dmp windows.cmdline

vol -f memory.dmp -r json windows.cmdline > /tmp/windows/cmdline.json

# 查看网络连接C2
vol -f memory.dmp windows.netscan

vol -f memory.dmp -r json windows.netscan > /tmp/windows/netscan.json

# 恶意代码注入检测
vol -f memory.dmp windows.malware.malfind

vol -f memory.dmp -r json windows.malware.malfind > /tmp/windows/malfind.json

#注册表与持久化

# 查看服务列表
vol -f memory.dmp windows.svclist

vol -f memory.dmp -r json windows.svclist > /tmp/windows/svclist.json

# 查看计划任务
vol -f memory.dmp windows.registry.scheduled_tasks

vol -f memory.dmp -r json windows.registry.scheduled_tasks > /tmp/windows/scheduled.json

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
vol -f memory.lime banners > /tmp/linux/banners.txt

# 进程列表

vol -f memory.lime linux.pslist.PsList
vol -f memory.lime -r json linux.pslist.PsList > /tmp/linux/pslist.json

# 进程树
vol -f memory.lime linux.pstree.PsTree
vol -f memory.lime -r json linux.pstree.PsTree > /tmp/linux/pstree.json

# 查看进程命令行参数
vol -f memory.lime linux.psaux.PsAux
vol -f memory.lime -r json linux.psaux.PsAux > /tmp/linux/psaux.json

# 网络连接C2
vol -f memory.lime linux.sockstat.Sockstat
vol -f memory.lime -r json linux.sockstat.Sockstat > /tmp/linux/sockstat.json

# 恶意代码注入检测
vol -f memory.lime linux.malware.malfind.Malfind
vol -f memory.lime -r json linux.malware.malfind.Malfind > /tmp/linux/malfind.json

# 恢复 bash 命令历史
vol -f memory.lime linux.bash.Bash
vol -f memory.lime -r json linux.bash.Bash > /tmp/linux/bash.json
```
