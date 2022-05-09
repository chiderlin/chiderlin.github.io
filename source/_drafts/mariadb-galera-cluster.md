---
title: MariaDB Galera Cluster|資料庫多主叢集架構設定
date: 2022-04-18 21:29:22
tags:
- 資料庫
- database
- Google engine vm
- Cluster
- 叢集
- 集群
- 多主

categories: 資料庫
---

# 前言
之前實作資料庫讀寫分離，也有研究一下資料庫叢集怎麼實作，也整理上來，如果想看讀寫分離[點我](https://chiderlin.github.io/2022/03/27/mariadb-replication-setting/)。

兩個方法各有優缺點，應該依專案的狀況選擇合適的技術做使用。


# 什麼是MariaDB Galera Cluster？多主叢集？
上一篇的資料庫讀寫分離是有一台db負責讀一台db負責寫，而多主叢集則是兩台同步都可以寫也可以讀
![架構圖](1.png)


# 開始之前

# 正文

# 心得
當時實作出來很有感，想著如果主從不同步的問題如果沒有解決的話就要改用這個～但後來有找到heart beat的設定，所以最後沒有使用這個Galera Cluster，但能解鎖這個也很不錯，之後多一個可以選擇的方式。

--
# 資料來源
[What is MariaDB Galera Cluster?](https://mariadb.com/kb/en/what-is-mariadb-galera-cluster/)
[How To Configure a Galera Cluster with MariaDB on Ubuntu 18.04 Servers](https://www.digitalocean.com/community/tutorials/how-to-configure-a-galera-cluster-with-mariadb-on-ubuntu-18-04-servers)
[Galera Cluster原理](https://segmentfault.com/a/1190000013652043)

[Galera Cluster: 一種新型的高一致性MySQL叢集架構](https://codertw.com/%E8%B3%87%E6%96%99%E5%BA%AB/120811/)

[Mariadb Galera cluster 的 auto increment 問題](https://ithelp.ithome.com.tw/questions/10192240?sc=pt)