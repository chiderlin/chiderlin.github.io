---
title: MariaDB slave/master備份+升級
date: 2022-7-29 14:23:33
tags:
    - 資料庫
    - database
    - Google engine vm
    - MariaDB
    - Master Slave
    - Ubuntu
    - upgrade
categories: 資料庫
---

# 前言

要用到 DATA TYPE JSON 時發現目前 mariaDB 版本不足，原本 10.1.48，
要到 10.2.3 才可以儲存 JSON 資料型態，快速記錄一下如何升級。

# 正文

我發現我有地方搞混了，來補充一下資料，
備份有兩種方法複製資料

1. mysqldump (mysql or mariaDB 都適用)
2. mariaDB 的 mariabackup

`---1,2 擇一方式備份即可---`

## 資料庫備份＆重新下載 mariaDB

```shell

儲存備份的資料
1. 用mysqldump來備份 => 匯出一sql文檔
$ mysqldump -uroot -p --all-databases > backup.sql


2. 用mariabackup來備份 => 匯出一大包/backup folder
#sudo mariabackup --backup -u<username> -p<password> --target-dir=/path/to/save/backup
$ sudo mariabackup --backup -uroot -proot --target-dir=/home/avery_yc/backup/

要恢復備份前要先準備它，讓它變成可以restore到DB
$ sudo mariabackup --prepare --target-dir=/home/avery_yc/backup/


--------重新下載mariadb的部分----------
停止原本在運作的mariadb
$ sudo systemctl stop mariadb

刪除原本的mariadb
$ sudo apt-get remove mariadb-server

重新下載mariadb，指定版本10.5
$ sudo apt install wget
$ wget https://downloads.mariadb.com/MariaDB/mariadb_repo_setup
$ chmod +x mariadb_repo_setup
$ sudo ./mariadb_repo_setup --mariadb-server-version="mariadb-10.5"

$ sudo apt update
$ sudo apt install mariadb-server mariadb-backup
$ sudo mariadb-upgrade

此時查看mariadb狀態已經是啟動的了 ⇒ $ sudo systemctl status mariadb

如果還沒啟動執行 => $ sudo systemctl start mariadb
---------重新下載mariadb的部分-----------

1. mysqldump還原備份的資料
$ mysql -uroot -p < backup.sql


2. mariabackup還原備份的資料
$ mariabackup --copy-back \
   --target-dir=/home/avery_yc/backup/

更改/backup權限
$ chown -R mysql:mysql /var/lib/mysql/

```

為什麼要更改權限？？[here](https://stackoverflow.com/questions/8329080/chown-mysqlmysql-data-tmp-command)
=> 大概感覺：mysql 需要自己的 user ID(mysql) & Group ID(mysql) 且會創造自己的目錄，如果使用別人的 user ID & Group ID mysql 會 error

![active](1.png)

成功升級成 10.5 了
![查看版本](2.png)

我先升級 slave 再 master
都是一樣的步驟，升級成功之後也檢查是否還有 slave & master 的關係。

## 檢查

`show slave status \G`
slave GOOD
![查看salve狀況](3.png)

`show master status`
master GOOD
![查看master狀況](4.png)

# 心得

其實實作起來蠻快的，但在升級前還是要有一顆大膽的心，因為是已經在運作的 database，所以操作時小心翼翼，還好 survey 後一次就成功了，熱騰騰的記錄下來。

---

# 資料來源

["chown mysql:mysql /data/tmp" command [closed]](https://stackoverflow.com/questions/8329080/chown-mysqlmysql-data-tmp-command)
[[Mariabackup] : Full Backup and Restore with Mariabackup](https://mariadb.com/kb/en/full-backup-and-restore-with-mariabackup/)
[How to Upgrade MariaDB in Ubuntu 18.04 LTS](https://www.liquidweb.com/kb/upgrade-mariadb-ubuntu-18-04/)
[[Mysqldump] : 備份&還原資料庫 – 指令範例](https://code.yidas.com/mysqldump/)
[sudo mysqldump : Permission denied](https://askubuntu.com/questions/378339/sudo-mysqldump-permission-denied)
[查詢已安裝的 MySQL / MariaDB 版本](https://www.ltsplus.com/mysql/check-mysql-version)
