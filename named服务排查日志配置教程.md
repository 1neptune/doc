#### `named` 服务排查日志配置

```bash
tcp 0 0 192.168.3.112:53 0.0.0.0:* LISTEN 34991/named
tcp6 0 0 :::53 :::* LISTEN 34991/named
```

##### 修改 `/etc/named.conf` 中的 `logging` 配置，将日志路径改为 `named` 有权限写入的目录

```bash
logging {
    channel default_debug {
        file "data/named.run";
        severity dynamic;
    };
    channel query_log {
        file "/var/named/data/query.log" versions 7 size 100m;
        print-time yes;
        print-category yes;
        print-severity yes;
        severity info;
    };
    category queries {
        query_log;
    };
};
```

##### 验证并重启

```bash
named-checkconf /etc/named.conf
rndc reload
systemctl restart named
systemctl status named  
```

##### 为什么配置 named服务排查日志

* 排查异常或非法的域名请求

* 记录局域网内所有设备的域名访问行为

```bash
26-Aug-2026 10:58:29.959 queries: info: client @0x7fa09c1cac00 192.168.21.34#63977 (www.anywell.marcopolo.com.cn): query: www.anywell.marcopolo.com.cn IN A + (192.168.3.112)
26-Aug-2026 11:02:29.161 queries: info: client @0x7fa080026880 192.168.21.34#62750 (www.anywell.marcopolo.com.cn): query: www.anywell.marcopolo.com.cn IN TYPE65 + (192.168.3.112)
26-Aug-2026 11:02:30.191 queries: info: client @0x7fa09c1825e0 192.168.21.34#51281 (www.anywell.marcopolo.com.cn): query: www.anywell.marcopolo.com.cn IN A + (192.168.3.112)
26-Aug-2026 11:02:30.232 queries: info: client @0x7fa09400bb60 192.168.21.34#61912 (www.anywell.marcopolo.com.cn): query: www.anywell.marcopolo.com.cn IN A + (192.168.3.112)
26-Aug-2026 11:04:58.592 queries: info: client @0x7fa0840244f0 192.168.21.34#62137 (www.anywell.marcopolo.com.cn): query: www.anywell.marcopolo.com.cn IN A + (192.168.3.112)
26-Aug-2026 11:04:58.592 queries: info: client @0x7fa080026880 192.168.21.34#49349 (www.anywell.marcopolo.com.cn): query: www.anywell.marcopolo.com.cn IN TYPE65 + (192.168.3.112)
26-Aug-2026 11:04:58.629 queries: info: client @0x7fa09405b510 192.168.21.34#60969 (www.anywell.marcopolo.com.cn): query: www.anywell.marcopolo.com.cn IN A + (192.168.3.112)
```

| 字段              | 含义                             | 值                               |
| --------------- | ------------------------------ | ------------------------------- |
| 时间戳             | 查询发生的时间                        | 2026-08-26 10:58:29.959         |
| `client`        | 发起查询的客户端                       | `192.168.21.34#63977`（IP + 源端口） |
| 查询域名            | 请求解析的域名                        | `www.anywell.marcopolo.com.cn`  |
| `IN A`          | 查询 A 记录（IPv4 地址）               | 获取域名的 IPv4 地址                   |
| `IN TYPE65`     | 查询 HTTPS 记录（DNS over HTTPS 相关） | 较新的 DNS 记录类型                    |
| `192.168.3.112` | 接收查询的 DNS 服务器地址                | 你的 `named` 服务器 IP               |
