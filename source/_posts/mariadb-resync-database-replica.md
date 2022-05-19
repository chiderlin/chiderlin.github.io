---
title: MariaDB slave/master replica斷掉，讓資料重新同步
date: 2022-05-18 00:00:04
tags:
    - database
    - MariaDB
    - 資料庫
    - Google engine vm
    - Ubuntu
    - Master Slave
    - re-sync
categories: 資料庫
---

# 前言

最近發很多資料庫設定的文章，好像快要變成記錄機器設定的部落格了 XD，
當下遇到問題，解決之後紀錄起來，感覺寫在這邊會比較完整，之後找紀錄也知道可以統一來這邊巡巡～ 也是快速紀錄的一篇。

[問題] 因為 master 磁碟滿了所以 master/slave 連線也斷掉了，
[解決] => 本文重點

1. slave 資料要同步回來
2. slave 重新連線 master
   [其他補充]
   master 因為 bin-log 檔案太多導致磁碟滿了，原先設定 bin-log 10 天清除，改成 3 天清除
   也可以加開磁碟空間。

---

# 正文

1. dump sql 檔案

    ```shell
        sudo mysqldump mysql -uroot -p > database.sql
    ```

2. master server 停機

    ```shell
        sudo service mysqld stop
    ```

3. 用 ssh ui 介面下載 master database.sql 到本機

4. slave ssh ui 介面上傳 database.sql (會到:~位置)

5. slave server 停止 replica

    ```shell
        shell mysql> stop slave
    ```

6. slave db 匯入 dump sql 檔案
    ```shell
        /home/path: $sudo mysqldump mysql -uroot -p < database.sql
    ```
7. master server 重新啟動
   為了看 bin-log 目前位置:
    ```shell
        $sudo service mysqld restart
    ```
8. master server 看 master 狀態
    ```shell
        mysql> show master status; 這指令應該看過無數次
    ```
    取得目前 bin-log 位置跟檔名。

9.slave server 重新設定

    ```shell
        重新設定追蹤的 master bin-log 位置/檔名
        mysql> reset slave;
        mysql>CHANGE MASTER TO MASTER_LOG_FILE='master-bin.000002', MASTER_LOG_POS=343;
    ```

10. slave server 重啟 slave
    ```shell
        mysql>`start slave;`
        mysql> `show slave status\G;` 這指令應該看過無數次
    ```

**已重新連線完成，到 db 裡面查看，數字同步完成**

### 補充：

原本要用 scp 遠端複製兩台 gcp instance，但 gcp 的設定卡很久一直沒有找到解法
scp error: permission denied.
gcloud error: request had insufficient authentication scopes.
最後決定直接先用 UI 介面直接先下載到本機...又 UI upload 到另一台 remote server...

---

# 資料來源

[[NOT WORK]将文件传输到 Linux 虚拟机](https://cloud.google.com/compute/docs/instances/transfer-files)
[How to re-sync the Mysql DB if Master and slave have different database incase of Mysql replication?](https://stackoverflow.com/questions/2366018/how-to-re-sync-the-mysql-db-if-master-and-slave-have-different-database-incase-o)
[How to Start, Stop, and Restart MySQL Server](https://www.hivelocity.net/kb/how-to-start-stop-and-restart-mysql-server/)
[How to Transfer a MySQL Database between Two Servers using SCP](https://medium.com/@hello.renzladroma/how-to-transfer-a-mysql-database-between-two-servers-using-scp-ea474a58cc5b)
[mysql 锁库与解锁 FLUSH TABLES WITH READ LOCK 和 UNLOCK TABLES](https://blog.csdn.net/a13568hki/article/details/107936023)
