## REA 定义

* REA : Reverse Engineer Anything

Understand how software works. Rebuild the feature you need.  
理解软件如何工作。重建你需要的功能。

## 逆向环境

* node 26.13

* python 3.14

* ida-pro 9.3

* workbuddy 

* ida-pro-mcp：https://github.com/mrexodia/ida-pro-mcp.git

* REA agent ： https://github.com/morluto/rea

## 安装MCP

* workbuddy提示词：D:\REA\ida-pro-mcp-main.zip , 安装idalib-mcp 并验证可用性

## 安装REA agent 代理

* 官网：https://reatool.dev  

* github： https://github.com/morluto/rea

* workbuddy提示词： 使用npx rea-agents@latest setup安装REA , 并连接到这个编程助手，当前环境安装了idalib-mcp, 安装时启用idalib引擎，验证安装结果

* workbuddy 连接器 开启 REA mcp 

## REA 使用

* workbuddy提示词：使用 REA 分析 D:\REA\watchdog\sangforupdater.exe

* workbuddy提示词：根据 REA 的逆向分析，使用rust语言重写一个sangforupdater.exe ,生成后的文件保存在D:\REA\watchdog\rust 目录下
