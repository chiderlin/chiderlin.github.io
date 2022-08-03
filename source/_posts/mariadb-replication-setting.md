---
title: MariaDB Replication|主從式架構設定
date: 2022-03-27 14:32:47
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

上次說明完用 gce 架 MariaDB，接下來要研究實作兩台 database 做讀寫分離的架構，[上一篇回顧](https://chiderlin.github.io/2022/02/19/Ubuntu18-04-MariaDB/)。
一開始設定讀寫分離架構成功，但遇到一個問題：如果寫資料庫閒置 10 幾分鐘都沒有被寫入之後，讀取的資料庫就停止同步了，要隔大概幾個小時讀的資料庫才會恢復同步....

正文會紀錄主從設定過程。

# 開始之前

開好兩台 mariaDB，正文開始設定兩台變成主從架構。

# 兩台 database 資料同步的原理？

![資料同步原理圖](3.png)
(圖片來源：https://read01.com/zh-tw/BJ368O7.html#.Yj_cnjdBzMI)
簡單說：data 被寫到 binary log 裡面，slave 就會有一個線程去執行把資料寫到自己 slave 這邊的 log，達到複製資料的效果，此時使用者要讀取資料時就讀取 slave 的 database。

# 正文：開始設定

## 1. 開啟主 mariaDB 資料庫配置檔案

`/etc/mysql/mariadb.conf.d/50-server.cnf檔案`

## 2. 在檔案新增

```shell
server-id=1 #server id每台要不同，不能重複
log-bin=master-bin #放log
log-bin-index=master-bin.index #放log
#replicate-do-db=repl_test #不用 #你要同步的table名稱
```

## 3. 重啟服務: `sudo service mysql restart`

## 4. 檢查配置，進入 db 查看 master 狀態:

```shell
$sudo mysql -u root
MariaDB[(none)]> SHOW MASTER STATUS;
```

(一開始設定好 File 名應該是這樣:master-bin.000001)，這樣表示有設定完成。
![master status](1.png)

## 5. 到從 mariaDB 資料庫配置檔案 /etc/mysql/mariadb.conf.d/50-server.cnf 檔案

```shell
server-id=2
#replicate-do-db=repl_test 不用
```

## 6. 一樣，重啟服務: `sudo service mysql restart`

## 7. 建立兩個資料庫的關聯，在主資料庫建立一個操作主從同步資料庫的使用者

```shell
MariaDB[(none)]> create user repl;
MariaDB[(none)]> GRANT REPLICATION SLAVE ON *.* TO 'repl(帳號)'@'從的ipxxx.xxx.xxx.xx(准許從ip可以連進來)' IDENTIFIED BY 'mysql(密碼)';
MariaDB[(none)]> flush privileges; #當前用戶訊息/權限設置從mysql內置庫中提取到內存裡 （更新權限功能）
```

## 8. 從資料庫執行：

```shell
mysql[(none)]> change master to master_host='主xxx.xxx.xxx.xx',
master_port=3306,
master_user='repl',
master_password='mysql',
master_log_file='master-bin.000007', # 按照master status出來的資料來填寫(File)
master_log_pos=36974; # 按照master status出來的資料來填寫（Position)

MariaDB[(none)]> start slave;
# mysql[(none)]> stop slave; (補充：停止slave)

```

## 9. 檢視 slave 狀態`mysql[(none)]> show slave status \G;`

> \G 表示換行顯示（美化顯示)
> slave_IO_Running & slave_SQL_Running 都 YES 就有連線了

![slave status](2.png)

## 10. 測試資料庫有沒有成功同步

> 記得要 master 資料庫寫入，slave 只負責讀取，如果 slave 不小心寫入就會報錯

在 master mariadb 裡創 database，slave 也會同步了

```shell
MariaDB[(none)]> create database testsplit;
```

---

# 停止同步問題的研究過程

1. 看 MariaDB 官方文件重新設定 -> 失敗
2. google 找其他解決方案，設定 config 檔 -> 失敗
3. 設定 semi-sync -> 失敗
4. heartbeat 設定 ＝> `成功`

# Heartbeat 設定

## Heartbeat 設定之前，Heartbeat 是什麼

有兩個重要的東西需一起設定

1. MASTER_HEARTBEAT_PERIOD： master 沒有更新數據的期間，master 會定期發送心跳包給 slave，用來確保 master 是正常運作的，服務沒有掛掉。而設定 Heartbeat 就是設定多長時間發送一個心跳包。

2. **SLAVE_NET_TIMEOUT**: slave 在多久沒有收到任何數據後(包含 binlog or heartbeat)會認為是與 master 連線斷開了(timeout)而重新連線。而設定 SLAVE_NET_TIMEOUT 就是設定多長時間重新連線。
   `如果單純把slave_net_timeout設得很短而沒設定到HEARTBEAT，則會造成master沒有數據更新時，slave就頻繁重新連線。`

## 開始設定

原本的問題：資料庫閒置 10 幾分鐘都沒有被寫入之後，讀取的資料庫就會停止同步。

接下來會發現原本的 heartbeat 期間設定為 1800（秒），也就是 master30 分鐘才會有一個心跳包。
![圖四](4.png)

而 slave_net_timeout 原本是一小時(3600 秒)才會重新連線一次。
![圖六](6.png)

```shell

MariaDB[(none)]> show status like '%heartbeat%'; # 查看heartbeat期間

# 這邊要設定heartbeat前，如果有啟動slave的話要先停止！
MariaDB[(none)]> stop slave;
MariaDB[(none)]> CHANGE MASTER TO MASTER_HEARTBEAT_PERIOD=1 #改成你要多久跳一次（我設定每秒跳一次，依情況，每秒有時候會導致loading過重）

MariaDB[(none)]> show variables like 'slave_net_timeout';
MariaDB[(none)]> set global slave_net_timeout=60; #改成timeout 60秒就重新連線
```

![圖五](5.png)
![圖七](7.png)

這樣就完成了設定。

---

# 心得

遇到 heartbeat 這個坑，卡了兩天，主要是沒有聽過這個詞的話不會用這個關鍵字，也沒有查到相關的解法，後來跟主管討論被提出來，試著去設定發現真的是它的問題而我才得救了 QQ，本來要找其他的實作方法(Galera Cluster)，也有設定成功，只是原理還需要再了解一下，下篇文章就寫吧。

---

# 資料來源:

[[教學][Ubuntu 架站] 如何在 Google Cloud Platform 架設 Ubuntu 伺服器](https://ui-code.com/archives/154)
[在 Ubuntu 18.04上安裝 MariaDB](https://medium.com/feveral%E7%9A%84%E7%A8%8B%E5%BC%8F%E7%AD%86%E8%A8%98/%E5%9C%A8-ubuntu-18-04%E4%B8%8A%E5%AE%89%E8%A3%9D-mariadb-a3084dae0304)
[worked=> How To Install MariaDB on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-install-mariadb-on-ubuntu-20-04)
[worked => 資料庫讀寫分離，主從同步實現方法](https://www.itread01.com/p/1323839.html)
[官方文件 - Setting Up Replication](https://mariadb.com/kb/en/setting-up-replication/#setting-up-a-replication-slave-with-mariabackup)
[worked=> What is heartbeat replication monitoring? [closed]](https://stackoverflow.com/questions/23798193/what-is-heartbeat-replication-monitoring)
[Actively monitoring replication connectivity with MySQL’s heartbeat](https://www.percona.com/blog/2011/12/29/actively-monitoring-replication-connectivity-with-mysqls-heartbeat/)
[MySQL 主從復制配置心跳功能介紹](https://www.itread01.com/articles/1478099427.html)
[slave_net_timeout, MASTER_HEARTBEAT_PERIOD, MASTER_CONNECT_RETRY,以及 MASTER_RETRY_COUNT 设置和查看](https://blog.csdn.net/lanyang123456/article/details/104547838)
