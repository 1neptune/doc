

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


vol -f memory.dmp windows.info

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

# 确认系统版本、内核基址、DTB、符号表加载情况
vol -f memory.dmp windows.info
vol -f memory.dmp -r json windows.info > /tmp/windows/info.json

# 正常进程列表（基于活动进程链表）
vol -f memory.dmp windows.pslist
vol -f memory.dmp -r json windows.pslist > /tmp/windows/pslist.json

# 扫描隐藏/被摘链的进程（基于内存池扫描）
vol -f memory.dmp windows.psscan
vol -f memory.dmp -r json windows.psscan > /tmp/windows/psscan.json

# 查看进程树（父子关系）
vol -f memory.dmp windows.pstree
vol -f memory.dmp -r json windows.pstree > /tmp/windows/pstree.json

# 多维度进程交叉验证（链表+扫描+会话+句柄等）
vol -f memory.dmp windows.psxview
vol -f memory.dmp -r json windows.psxview > /tmp/windows/psxview.json

# 查看进程命令行参数
vol -f memory.dmp windows.cmdline
vol -f memory.dmp -r json windows.cmdline > /tmp/windows/cmdline.json

# 进程名伪装检测（EPROCESS vs PEB 对比）
vol -f memory.dmp windows.malware.pebmasquerade
vol -f memory.dmp -r json windows.malware.pebmasquerade > /tmp/windows/pebmasquerade.json


# 扫描网络对象（TCP/UDP 连接与监听）
vol -f memory.dmp windows.netscan
vol -f memory.dmp -r json windows.netscan > /tmp/windows/netscan.json

# 遍历网络跟踪结构（另一套数据结构，与 netscan 互补）
vol -f memory.dmp windows.netstat
vol -f memory.dmp -r json windows.netstat > /tmp/windows/netstat.json

# 扫描 RWX 内存区域和无文件注入
vol -f memory.dmp windows.malware.malfind
vol -f memory.dmp -r json windows.malware.malfind > /tmp/windows/malfind.json

# 可疑线程检测（起始地址不在已知模块内）
vol -f memory.dmp windows.malware.suspicious_threads
vol -f memory.dmp -r json windows.malware.suspicious_threads > /tmp/windows/suspicious_threads.json

# 进程幽灵化检测（DeletePending 位 / 文件对象为 0）
vol -f memory.dmp windows.malware.processghosting
vol -f memory.dmp -r json windows.malware.processghosting > /tmp/windows/processghosting.json

# DLL 加载对比（发现被摘链的 DLL）
vol -f memory.dmp windows.ldrmodules
vol -f memory.dmp -r json windows.ldrmodules > /tmp/windows/ldrmodules.json


# 系统服务调度表（检测 SSDT Hook）
vol -f memory.dmp windows.ssdt
vol -f memory.dmp -r json windows.ssdt > /tmp/windows/ssdt.json

# 内核回调例程（恶意驱动常注册回调）
vol -f memory.dmp windows.callbacks
vol -f memory.dmp -r json windows.callbacks > /tmp/windows/callbacks.json

# 内核模块对比（发现隐藏驱动）
vol -f memory.dmp windows.modules
vol -f memory.dmp -r json windows.modules > /tmp/windows/modules.json
vol -f memory.dmp windows.modscan
vol -f memory.dmp -r json windows.modscan > /tmp/windows/modscan.json

# 驱动扫描
vol -f memory.dmp windows.driverscan
vol -f memory.dmp -r json windows.driverscan > /tmp/windows/driverscan.json

# 服务差异对比（发现被 Rootkit 隐藏的服务）
vol -f memory.dmp windows.malware.svcdiff
vol -f memory.dmp -r json windows.malware.svcdiff > /tmp/windows/svcdiff.json

# 未挂钩系统调用检测（EDR 绕过技术）
vol -f memory.dmp windows.malware.unhooked_system_calls
vol -f memory.dmp -r json windows.malware.unhooked_system_calls > /tmp/windows/unhooked_syscalls.json


# 查看服务列表
vol -f memory.dmp windows.svclist
vol -f memory.dmp -r json windows.svclist > /tmp/windows/svclist.json

# 查看计划任务
vol -f memory.dmp windows.registry.scheduled_tasks
vol -f memory.dmp -r json windows.registry.scheduled_tasks > /tmp/windows/scheduled.json

# 用户辅助记录（程序执行次数、最后运行时间）
vol -f memory.dmp windows.registry.userassist
vol -f memory.dmp -r json windows.registry.userassist > /tmp/windows/userassist.json

# AmCache（应用程序执行信息 + SHA1 哈希）
vol -f memory.dmp windows.registry.amcache
vol -f memory.dmp -r json windows.registry.amcache > /tmp/windows/amcache.json

# Shimcache（程序执行痕迹，即使已删除）
vol -f memory.dmp windows.shimcachemem
vol -f memory.dmp -r json windows.shimcachemem > /tmp/windows/shimcache.json

# 注册表 hive 列表
vol -f memory.dmp windows.registry.hivelist
vol -f memory.dmp -r json windows.registry.hivelist > /tmp/windows/hivelist.json

# 自启动项检查（Run / RunOnce）
vol -f memory.dmp windows.registry.printkey --key "Software\Microsoft\Windows\CurrentVersion\Run"
vol -f memory.dmp -r json windows.registry.printkey --key "Software\Microsoft\Windows\CurrentVersion\Run" > /tmp/windows/run_key.json

# 命令历史（类似 Linux bash）
vol -f memory.dmp windows.cmdscan
vol -f memory.dmp -r json windows.cmdscan > /tmp/windows/cmdscan.json

# 文件对象扫描（含已删除文件）
vol -f memory.dmp windows.filescan
vol -f memory.dmp -r json windows.filescan > /tmp/windows/filescan.json

# MFT 记录扫描（含 ADS 备用数据流）
vol -f memory.dmp windows.mftscan.MFTScan
vol -f memory.dmp -r json windows.mftscan.MFTScan > /tmp/windows/mftscan.json

# 互斥体扫描（恶意软件家族指纹）
vol -f memory.dmp windows.mutantscan
vol -f memory.dmp -r json windows.mutantscan > /tmp/windows/mutantscan.json

# 句柄列表（异常文件/注册表/进程访问）
vol -f memory.dmp windows.handles
vol -f memory.dmp -r json windows.handles > /tmp/windows/handles.json

# 已卸载模块（临时加载后卸载的驱动）
vol -f memory.dmp windows.unloadedmodules
vol -f memory.dmp -r json windows.unloadedmodules > /tmp/windows/unloadedmodules.json


# 时间线聚合（自动运行多个时间相关插件，按时间排序）
vol -f memory.dmp timeliner.Timeliner
vol -f memory.dmp -r json timeliner.Timeliner > /tmp/windows/timeliner.json

# 内存字符串搜索（网络插件失效时的替代方案）
vol -f memory.dmp windows.strings --strings-file /tmp/windows/ip_list.txt

vol -f memory.dmp -r json windows.strings --strings-file /tmp/windows/ip_list.txt > /tmp/windows/strings_matches.json

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



# GitHub 符号表仓库
https://github.com/leludo84/vol3-linux-profiles


xz -d kernel-4.18.0-348.7.1.el8_5.x86_64.json.xz

# 输出kernel-4.18.0-348.7.1.el8_5.x86_64.json

mkdir -p ~/vol3/lib/python3.14/site-packages/volatility3/symbols/linux/

cp kernel-4.18.0-348.7.1.el8_5.x86_64.json ~/vol3/lib/python3.14/site-packages/volatility3/symbols/linux/

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
