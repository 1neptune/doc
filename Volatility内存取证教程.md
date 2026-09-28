##### 为什么需要内存取证

* **对抗无文件攻击** ：恶意代码不写入磁盘而是直接注入到合法进程的内存空间中执行；

* **捕获易失性证据** ：内存中保存加密密钥与凭据，当前活动的网络连接，正在运行的进程及其命令行参数，剪贴板内容；

* **应对反取证与数据销毁** ：攻击者常使用时间戳篡改、日志清除、内存擦除等手段，内存中可能仍保留着日志缓冲区、命令历史、注册表配置单元的未刷新版本；

* **与磁盘取证互补** :  磁盘取证告诉你攻击者留下了什么，内存取证告诉你攻击者正在做什么、用什么做、以及他们试图隐藏什么;



##### 内存取证的典型场景

* **应急响应与事件调查** :  当发现系统被入侵时，第一时间获取内存镜像，分析恶意进程、网络连接、注入代码，还原攻击者的操作痕迹；

* **恶意软件分析** ： 对于无文件恶意软件、内存驻留型恶意软件，内存取证是唯一能获取其完整样本和配置的手段；

* **凭证窃取检测** ： 检测攻击者是否使用了 Mimikatz 等工具从 `lsass.exe` 内存中提取了密码或哈希；

* **Rootkit 检测** ：内核级 Rootkit 会隐藏进程、文件和网络连接。内存取证可以直接分析内核结构，发现被篡改的系统调用表、隐藏的驱动对象；



##### 安全人员需要分析什么

* **进程分析**  ： 用于发现隐藏进程、异常父子关系和可疑命令行

```bash
Windows

# 列出活动进程链表（正常进程）
vol -f memory.dmp windows.pslist

# 扫描内存池中的进程对象（可发现被摘链的隐藏进程）
vol -f memory.dmp windows.psscan

# 以树状展示父子关系，快速发现异常（如 Word 启动 cmd.exe）
vol -f memory.dmp windows.pstree

# 查看进程完整命令行参数（识别编码的 PowerShell 等）
vol -f memory.dmp windows.cmdline


Linux

# 列出活动进程链表（正常进程）
vol -f memory.lime linux.pslist

# 以树状展示父子关系，快速发现异常（如 Web 服务启动的 Shell）
vol -f memory.lime linux.pstree

# 查看进程完整命令行参数（识别 Base64 编码、/tmp 下运行的程序等）
vol -f memory.lime linux.psaux

# 查看进程环境变量（发现 LD_PRELOAD 注入、自定义变量中的凭据）
vol -f memory.lime linux.envars
```



* **网络连接分析** : 用于定位 C2 通信、横向移动痕迹

```bash
Windows 

# 扫描内存中的网络连接对象
vol -f memory.dmp windows.netscan

# 遍历网络跟踪结构（另一套数据结构，与 netscan 互补）
vol -f memory.dmp windows.netstat


Linux

# 列出网络连接和监听 Socket（替代 netstat）
vol -f memory.lime linux.sockstat

# 扫描内存中的网络连接结构（与 sockstat 互补）
vol -f memory.lime linux.sockscan
```

* **恶意代码注入检测**： 用于发现无文件攻击、进程注入、Reflective DLL 注入

```bash
Windows

# 扫描 RWX 内存区域和无文件的内存注入（核心插件）
vol -f memory.dmp windows.malware.malfind

# 查看指定进程的 VAD 树，确认内存权限和私有属性
vol -f memory.dmp windows.vadinfo --pid <PID>

Linux

# 扫描进程内存中可疑的可执行区域（核心插件）
vol -f memory.lime linux.malfind
```



* **Rootkit 与内核 Hook 检测**: 用于发现内核级隐藏技术

```bash
Windows

# 检查 SSDT 是否被 Hook
vol -f memory.dmp windows.ssdt

# 对比模块列表（可能被 DKOM 操控）和驱动扫描（绕过 DKOM）
vol -f memory.dmp windows.modules
vol -f memory.dmp windows.driverscan

# 列出内核回调例程（恶意驱动常注册回调）
vol -f memory.dmp windows.callbacks


Linux


# 对比模块列表和内存扫描，发现隐藏的内核模块
vol -f memory.lime linux.check_modules

# 检查系统调用表是否被 Hook
vol -f memory.lime linux.check_syscall

# 检查中断描述符表（IDT）是否被篡改
vol -f memory.lime linux.check_idt

# 检查网络协议操作函数指针是否被 Hook（用于隐藏连接）
vol -f memory.lime linux.check_afinfo
```

* **凭证与敏感信息提取** ： 用于检测凭证窃取、评估泄露范围

```bash
Windows

# 提取 SAM 数据库中的本地账户 NTLM 哈希
vol -f memory.dmp windows.hashdump

# 提取 LSA secrets（服务账户密码、机器账户密码等）
vol -f memory.dmp windows.lsadump

# 提取缓存的域凭据（DCC2/MSCASH）
vol -f memory.dmp windows.cachedump

# 恢复 bash 命令历史（最有价值的取证线索之一）
vol -f memory.lime linux.bash

# 从环境变量中筛选疑似凭据（配合 grep）
vol -f memory.lime linux.envars | grep -iE "pass|key|secret|token"

# 在内存缓存中查找特定文件（如 /etc/shadow）
vol -f memory.lime linux.pagecache.Files --find /etc/shadow

# 从内存缓存中恢复 /etc/shadow 的内容
vol -f memory.lime linux.pagecache.InodePages --find /etc/shadow --dump
```



* **持久化机制分析** ： 用于发现攻击者维持访问的痕迹

```bash
Windows

# 列出加载的注册表 hive
vol -f memory.dmp windows.registry.hivelist

# 检查自启动项（Run/RunOnce）
vol -f memory.dmp windows.registry.printkey --key "Software\Microsoft\Windows\CurrentVersion\Run"

# 解码计划任务信息
vol -f memory.dmp windows.registry.scheduled_tasks

Linux 

# 列出进程打开的文件（发现指向 /tmp 或已删除文件的句柄）
vol -f memory.lime linux.lsof

# 列出挂载点（发现异常的 tmpfs 或远程挂载）
vol -f memory.lime linux.mountinfo
```

* **时间线还原** ： 用于按时间顺序聚合事件，还原攻击序列

```bash
Windows

# 运行所有与时间相关的插件，按时间排序输出
vol -f memory.dmp timeliner.Timeliner


Linux
# 获取系统启动时间
vol -f memory.lime linux.boottime
```



* **执行痕迹追踪**： 用于查看程序执行历史，即使文件已被删除

```bash
Windows 

# 查看 UserAssist 记录（用户执行过的程序、次数、最后运行时间）
vol -f memory.dmp windows.registry.userassist

# 读取 Shimcache（程序执行痕迹）
vol -f memory.dmp windows.shimcachemem

# 恢复命令历史（类似 Linux bash history）
vol -f memory.dmp windows.cmdscan

# 查看控制台缓冲区内容（可能包含攻击者执行的命令及输出）
vol -f memory.dmp windows.consoles


Linux 

# 列出所有进程内存映射的 ELF 文件（发现从临时目录加载的可执行文件）
vol -f memory.lime linux.elfs

# 查看指定进程的内存映射（类似 /proc/<pid>/maps）
vol -f memory.lime linux.proc.Maps --pid <PID>

# 查看内核日志缓冲区（可能包含攻击者的操作痕迹或错误信息）
vol -f memory.lime linux.kmsg
```



内存取证的核心价值在于：**它捕获的是系统运行时的“活”状态，这是磁盘取证无法替代的**



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


