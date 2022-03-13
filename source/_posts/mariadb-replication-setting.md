---
title: MariaDB Replication|主從式架構設定
date: 2022-03-14 22:32:47
tags:
- 資料庫讀寫分離
- database
- MariaDB
- Google engine vm
- Replication
- Master Slave
categories: 資料庫
---

# 前言
上次說明完用gce架MariaDB，接下來要研究實作兩台database做讀寫分離的架構，[上一篇回顧](https://chiderlin.github.io/2022/02/19/Ubuntu18-04-MariaDB/)。
一開始設定讀寫分離架構成功，但遇到一個問題：如果寫資料庫閒置10幾分鐘都沒有被寫入之後，讀取的資料庫就停止同步了，要隔大概幾個小時讀的資料庫才會恢復同步....

正文會紀錄主從設定過程。

# 研究過程
1. 看MariaDB官方文件重新設定 -> 失敗
2. google找其他解決方案，設定config檔 -> 失敗
3. 設定 semi-sync -> 失敗
4. heartbeat設定 ＝> `成功`

# 開始之前
開好兩台mariaDB，正文開始設定兩台變成主從架構。

# 兩台database資料同步的原理？
#TODO: 寫資料同步的原理

# 正文：開始設定
## 1. 開啟主mariaDB資料庫配置檔案
`/etc/mysql/mariadb.conf.d/50-server.cnf檔案`

## 2. 在檔案新增
```shell
server-id=1 #server id每台要不同，不能重複
log-bin=master-bin #放log
log-bin-index=master-bin.index #放log
replicate-do-db=repl_test #你要同步的table名稱
```

## 3. 重啟服務: `server mysql restart`
## 4. 檢查配置，進入db查看master狀態:
``` shell
$sudo mysql -u root 
mysql[(none)]> SHOW MASTER STATUS;
```
(一開始設定好File名應該是這樣:master-bin.000001)，這樣表示有設定完成。
![master status](1.png)

## 5. 到從mariaDB資料庫配置檔案 /etc/mysql/mariadb.conf.d/50-server.cnf檔案
``` shell
server-id=2
replicate-do-db=repl_test
```
## 6. 一樣，重啟服務: `server mysql restart`
## 7. 建立兩個資料庫的關聯，在主資料庫建立一個操作主從同步資料庫的使用者
``` shell
mysql[(none)]> create user repl;
mysql[(none)]> GRANT REPLICATION SLAVE ON *.* TO 'repl(帳號)'@'從的ipxxx.xxx.xxx.xx(准許從ip可以連進來)' IDENTIFIED BY 'mysql(密碼)';
mysql[(none)]> flush privileges; #當前用戶訊息/權限設置從mysql內置庫中提取到內存裡 （更新權限功能）
```
## 8. 從資料庫執行：
``` shell
mysql[(none)]> change master to master_host='主xxx.xxx.xxx.xx',
master_port=3306,
master_user='repl',
master_password='mysql',
master_log_file='master-bin.000007', # 按照master status出來的資料來填寫(File)
master_log_pos=36974; # 按照master status出來的資料來填寫（Position)

mysql[(none)]> start slave;
# mysql[(none)]> stop slave; (補充：停止slave)

```
## 9. 檢視slave狀態`mysql[(none)]> show slave status \G;`
>\G表示換行顯示（美化顯示)
slave_IO_Running & slave_SQL_Running都YES就有連線了

![slave status](2.png)

## 10. 測試資料庫有沒有成功同步
> 記得要master資料庫寫入，slave只負責讀取，如果slave不小心寫入就會報錯

在master mariadb裡創database，slave也會同步了
``` shell
mysql[(none)]> create database testsplit;
```
---
# Heartbeat設定
為了解決資料庫閒置10幾分鐘都沒有被寫入之後，讀取的資料庫就會停止同步的問題，要設定heartbeat
``` shell

#TODO: 寫heartbeat設定部分

```
---
# 心得
遇到heartbeat這個坑，卡了兩天，主要是沒有聽過這個詞的話不會用這個關鍵字，也沒有查到相關的解法，後來跟主管討論被提出來，試著去設定發現真的是它的問題而我才得救了QQ，本來要找其他的實作方法(Galera Cluster)，也有設定成功，只是原理還需要再了解一下，下篇文章就寫吧。

---
# 資料來源:
[[教學][Ubuntu 架站] 如何在 Google Cloud Platform 架設 Ubuntu 伺服器](https://ui-code.com/archives/154)
[在 Ubuntu 18.04上安裝 MariaDB](https://medium.com/feveral%E7%9A%84%E7%A8%8B%E5%BC%8F%E7%AD%86%E8%A8%98/%E5%9C%A8-ubuntu-18-04%E4%B8%8A%E5%AE%89%E8%A3%9D-mariadb-a3084dae0304)
[worked=> How To Install MariaDB on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-install-mariadb-on-ubuntu-20-04)
[worked => 資料庫讀寫分離，主從同步實現方法](https://www.itread01.com/p/1323839.html)
[官方文件 - Setting Up Replication](https://mariadb.com/kb/en/setting-up-replication/#setting-up-a-replication-slave-with-mariabackup)
[worked=> What is heartbeat replication monitoring? [closed]](https://stackoverflow.com/questions/23798193/what-is-heartbeat-replication-monitoring)
[Actively monitoring replication connectivity with MySQL’s heartbeat](https://www.percona.com/blog/2011/12/29/actively-monitoring-replication-connectivity-with-mysqls-heartbeat/)
[MySQL主從復制配置心跳功能介紹](https://www.itread01.com/articles/1478099427.html)