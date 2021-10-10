---
title: "PostgreSQL利用Foreign Table通过tds_fdw插件访问Microsoft SQL Server数据库表数据"
date: 2021-10-10T23:12:54+08:00
draft: true
tags: ["PostgreSQL","MSSQL"]
categories: ["PostgreSQL","DB"]
---

### PostgreSQL Foreign Table 简介

PostgreSQL Foreign Table 外部表支持从自身数据以外的数据源获取数据，例如文件、其他数据库。常见的有oracle_fdw,mysql_fdw,file_fdw。本文是使用tds_fdw插件转发了Microsoft SQL Server数据库中的数据，提供了只读访问权限，PostgresSQL此次部署在Windows Server平台。

官网介绍 [PostgreSQL: Documentation: 9.3: Foreign Data](https://www.postgresql.org/docs/9.3/ddl-foreign-data.html)

### 环境准备

本次PostgreSQL与Microsoft SQL Server部署在同一台服务器上
PostgreSQL版本为 PostgreSQL 12.8, compiled by Visual C++ build 1914, 64-bit
端口 5432

Microsoft SQL Server版本为 Microsoft SQL Server 2016 (RTM) - 13.0.1601.5 (X64)   Apr 29 2016 23:23:58   Copyright (c) Microsoft Corporation  Developer Edition (64-bit) on Windows 10 Enterprise LTSC 2019 6.3 \<X64\> (Build 17763: ) (Hypervisor)  
端口 1433

### tds_fdw 插件介绍与安装

GitHub地址 [TDS Foreign data wrapper](https://github.com/tds-fdw/tds_fdw)
作者提供了PostgreSQL通过Tabular Data Stream (TDS)协议访问Sybase databases 和 Microsoft SQL Server插件，不过根据作者介绍，现版本还不支持写入(经过测试确实如此)以及不支持下推

且作者只提供了Linux和MacOS平台下的编译方式，没有提供Windows平台的，某个issue也提出了这个问题，其中有人帮忙打包好了对应版本的DLL文件：
[Installation on Windows · Issue #53 · tds-fdw/tds_fdw · GitHub](https://github.com/tds-fdw/tds_fdw/issues/53)

其中可以找到PostgreSQL 11 和 12 版本对应的编译后的文件，
[tds_fdw-pg_edb_11_winx64.zip](https://github.com/yyjdelete/temp_storage/files/4384803/tds_fdw-pg_edb_11_winx64.zip)
[tds_fdw-pg_edb_12_winx64.zip](https://github.com/yyjdelete/temp_storage/files/4605716/tds_fdw-pg_edb_12_winx64.zip)

编译后的文件中有三个文件夹bin，lib 和 share, 对应了PostgreSQL安装路径下的三个文件夹，将其复制进去即可。

具体方式及明细参见 [issuecomment-604196621](https://github.com/tds-fdw/tds_fdw/issues/53#issuecomment-604196621)

### PostgreSQL 配置

因为是新安装的PostgreSQL，所以需要配置一下其他设置
Wiwndows下PostgreSQL配置文件位于: C:\Program Files\PostgreSQL\12\data\postgresql.conf

其中
listen_addresses = '*' 允许外部IP访问
port = 5432  端口号默认是5432
shared_buffers = 128MB 缓存最大值，因为数据量较小，所以这里设置了128MB

### tds_fdw加载以及Foreign Table配置

在安装了对应插件文件，重启PostgreSQL服务后在SQL查询界面：

``` SQL

#启用tds_fdw插件
CREATE EXTENSION tds_fdw;

#配置Microsoft SQL Server服务器地址
CREATE SERVER nc_sqlserver    
      FOREIGN DATA WRAPPER tds_fdw
      OPTIONS(servername 'localhost',port '1433',DATABASE 'Test',language 'Simplified Chinese',character_set 'UTF-8');

#配置可以访问Microsoft SQL Server要复制的数据库的用户以及密码
CREATE  USER MAPPING FOR postgres
         SERVER nc_sqlserver
         OPTIONS(username 'sa',password 'xxx');

#将整个库链接到PostgreSQL，链接特定表的方式参照tds_fdw GitHub README
IMPORT FOREIGN SCHEMA dbo from server nc_sqlserver into public;

```

之后通过pgAdmin就可以看到对应库中的Foreign Tables出现了Microsoft SQL Server中的表,即可按照普通表格查询数据
