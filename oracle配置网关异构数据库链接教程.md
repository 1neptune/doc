## **安装 Microsoft ODBC 驱动和 unixODBC**

```bash
su - root
# 导入 Microsoft GPG 密钥
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc

# 添加 Microsoft 软件源
sudo curl -o /etc/yum.repos.d/mssql-release.repo https://packages.microsoft.com/config/rhel/8/prod.repo

# 安装 Microsoft ODBC Driver 18 for SQL Server
sudo ACCEPT_EULA=Y yum install -y msodbcsql18

# 安装 unixODBC
sudo yum install -y unixODBC unixODBC-devel

# 验证驱动是否注册成功
odbcinst -q -d
# 输出中应能看到 [ODBC Driver 18 for SQL Server]
```

## **下载并解压网关介质**

```bash
su - oracle
cd /home/oracle
# 假设你已经把 LINUX.X64_193000_gateways.zip 上传到了这里
unzip LINUX.X64_193000_gateways.zip
# 解压后生成 /home/oracle/gateways 目录
```

## **创建独立网关环境脚本**

```bash
su - oracle
cat > /home/oracle/gw.sh << 'EOF'
#!/bin/bash
export ORACLE_BASE=/data/app/oracle
export ORACLE_HOME=/data/app/oracle/product/19.0.0/gateway
export PATH=$ORACLE_HOME/bin:/usr/local/bin:/bin:/usr/bin
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export TNS_ADMIN=$ORACLE_HOME/network/admin
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8
EOF
chmod +x /home/oracle/gw.sh
```

## 静默安装网关

### 编写响应文件

```bash
su - oracle
mkdir -p /home/oracle/rsp
cat > /home/oracle/rsp/tg.rsp << 'EOF'
oracle.install.responseFileVersion=/oracle/install/rspfmt_tginstall_response_schema_v19.0.0
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/data/app/oraInventory
ORACLE_HOME=/data/app/oracle/product/19.0.0/gateway
ORACLE_BASE=/data/app/oracle
oracle.install.tg.customComponents={oracle.rdbms.tg4msql:19.0.0.0.0,oracle.rdbms.hsodbc:19.0.0.0.0}
EOF
```

### 执行静默安装

```bash
export CV_ASSUME_DISTID=RHEL8
export LANG=en_US
cd /home/oracle/gateways
./runInstaller -silent -noconfig -responseFile /home/oracle/rsp/tg.rsp
su - root
/data/app/oracle/product/19.0.0/gateway/root.sh
exit
```

## 配置数据库链接

### 配置网关初始化文件

```bash
source /home/oracle/gw.sh
cat > $ORACLE_HOME/dg4msql/admin/initdg4msql_33.ora << 'EOF'
HS_FDS_CONNECT_INFO=192.168.3.3:1433//ifsdata_new
HS_FDS_TRACE_LEVEL=OFF
EOF
```

### 配置网关监听器 listener.ora

```bash
vim /etc/hosts
192.168.30.50 db

cat > $ORACLE_HOME/network/admin/listener.ora << 'EOF'
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = TCP)(HOST = db)(PORT = 1522))
    )
  )

SID_LIST_LISTENER =
  (SID_LIST =
    # 192.168.3.5
    (SID_DESC =
      (SID_NAME = dg4msql)
      (ORACLE_HOME = /data/app/oracle/product/19.0.0/gateway)
      (PROGRAM = dg4msql)
    )
    # 192.168.3.3
    (SID_DESC =
      (SID_NAME = dg4msql_33)
      (ORACLE_HOME = /data/app/oracle/product/19.0.0/gateway)
      (PROGRAM = dg4msql)
    )
  )
EOF


lsnrctl stop
lsnrctl start
lsnrctl status
```

### 数据库端配置tnsnames.ora

```bash
exit

su - oracle

cat >> $ORACLE_HOME/network/admin/tnsnames.ora << 'EOF'

GTWSQL_IFS =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = db)(PORT = 1522))
    (CONNECT_DATA = (SID = dg4msql))
    (HS = OK)
  )

GTWSQL_33 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = db)(PORT = 1522))
    (CONNECT_DATA = (SID = dg4msql_33))
    (HS = OK)
  )
EOF


tnsping GTWSQL_33
```

### 创建 DB Link 并测试

```bash
sqlplus / as sysdba

# 授权数据库链接权限
GRANT CREATE DATABASE LINK TO V3XUSER;

CREATE DATABASE LINK IFSDATA_LINK_33
  CONNECT TO "sa" IDENTIFIED BY "db_onlinemk_new2023"
  USING 'GTWSQL_33';


SELECT count(*) FROM IFS_Customer_PriceList_Link@IFSDATA_LINK_33;
```

## ![](D:/marktext/images/2026-11-14-09-35-13-image.png)

## 配置存储过程

### 创建数据库链接

![](D:/marktext/images/2026-11-14-09-38-07-image.png)

![](D:/marktext/images/2026-11-14-09-37-21-image.png)

### 转换数据库存储过程

```bash
CREATE OR REPLACE PROCEDURE A8_Agreement_Sys_Compare_price
(
    p_cust_no            IN  VARCHAR2,   --客户编码
    -- p_part_no         IN  VARCHAR2,   --产品编码
    p_part_Name          IN  VARCHAR2,   --产品名称
    p_grade              IN  VARCHAR2,   --等级
    p_qty_min_price_control IN FLOAT,    --最低价格数量控制,即XX平米以上才算最低价

    p_Cust_min_price     OUT FLOAT,      --集团客户最低价
    p_Grp_min_price      OUT FLOAT,      --事业部最低价
    -- p_qty_avg_discount_contral IN FLOAT, --平均折扣数量控制,即XX平米以上才算折扣
    p_Cust_avg_discount  OUT VARCHAR2,   --客户平均折扣
    p_Grp_avg_discount   OUT VARCHAR2,   --事业部平均折扣,
    p_Current_price      OUT FLOAT,      --现行价格
    p_ConfigMsg          OUT VARCHAR2    --配置信息
)
AS
    v_Cust_min_price    FLOAT := 0;
    v_Grp_min_price     FLOAT := 0;
    v_Cust_avg_discount VARCHAR2(50) := ' ';
    v_Grp_avg_discount  VARCHAR2(50) := ' ';
    v_Current_price     FLOAT := 0;
    v_ConfigMsg         VARCHAR2(100) := ' ';
BEGIN
    -- SQL Server 中的 exec [192.168.3.3].[ifsdata_new].[dbo].A8_Agreement_Sys_Compare_price
    "dbo"."A8_Agreement_Sys_Compare_price"@IFSDATA_LINK_33(
        p_cust_no,
        p_part_Name,
        p_grade,
        p_qty_min_price_control,
        v_Cust_min_price,
        v_Grp_min_price,
        v_Cust_avg_discount,
        v_Grp_avg_discount,
        v_Current_price,
        v_ConfigMsg
    );
    p_Cust_min_price    := v_Cust_min_price;
    p_Grp_min_price     := v_Grp_min_price;
    p_Cust_avg_discount := v_Cust_avg_discount;
    p_Grp_avg_discount  := v_Grp_avg_discount;
    p_Current_price     := v_Current_price;
    p_ConfigMsg         := v_ConfigMsg;
END A8_Agreement_Sys_Compare_price;
/
```

### ![](D:/marktext/images/2026-11-14-09-42-25-image.png)

![](D:/marktext/images/2026-11-14-09-43-03-image.png)

### 存储过程测试用例

```bash
DECLARE
    v_cust_no               VARCHAR2(50) := '0001';
    v_part_Name             VARCHAR2(100) := 'CSN27K004AS-M';
    v_grade                 VARCHAR2(10) := 'A';
    v_qty_min_price_control FLOAT := 47.52;

    v_Cust_min_price        FLOAT;
    v_Grp_min_price         FLOAT;
    v_Cust_avg_discount     VARCHAR2(50);
    v_Grp_avg_discount      VARCHAR2(50);
    v_Current_price         FLOAT;
    v_ConfigMsg             VARCHAR2(100);
BEGIN
    A8_Agreement_Sys_Compare_price(
        p_cust_no               => v_cust_no,
        p_part_Name             => v_part_Name,
        p_grade                 => v_grade,
        p_qty_min_price_control => v_qty_min_price_control,
        p_Cust_min_price        => v_Cust_min_price,
        p_Grp_min_price         => v_Grp_min_price,
        p_Cust_avg_discount     => v_Cust_avg_discount,
        p_Grp_avg_discount      => v_Grp_avg_discount,
        p_Current_price         => v_Current_price,
        p_ConfigMsg             => v_ConfigMsg
    );

    DBMS_OUTPUT.PUT_LINE('=================================');
    DBMS_OUTPUT.PUT_LINE('调用成功！返回结果如下：');
    DBMS_OUTPUT.PUT_LINE('集团客户最低价 (Cust_min_price): ' || NVL(TO_CHAR(v_Cust_min_price), 'NULL'));
    DBMS_OUTPUT.PUT_LINE('事业部最低价 (Grp_min_price): ' || NVL(TO_CHAR(v_Grp_min_price), 'NULL'));
    DBMS_OUTPUT.PUT_LINE('客户平均折扣 (Cust_avg_discount): ' || NVL(v_Cust_avg_discount, 'NULL'));
    DBMS_OUTPUT.PUT_LINE('事业部平均折扣 (Grp_avg_discount): ' || NVL(v_Grp_avg_discount, 'NULL'));
    DBMS_OUTPUT.PUT_LINE('现行价格 (Current_price): ' || NVL(TO_CHAR(v_Current_price), 'NULL'));
    DBMS_OUTPUT.PUT_LINE('配置信息 (ConfigMsg): ' || NVL(v_ConfigMsg, 'NULL'));
    DBMS_OUTPUT.PUT_LINE('=================================');

EXCEPTION
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('=================================');
        DBMS_OUTPUT.PUT_LINE('调用失败！错误信息如下：');
        DBMS_OUTPUT.PUT_LINE(SQLERRM);
        DBMS_OUTPUT.PUT_LINE('=================================');
END;
```

Oracle 执行结果：

![](D:/marktext/images/2026-11-14-10-04-07-image.png)

SQLServer执行结果：

![](D:/marktext/images/2026-11-14-10-08-45-image.png)
