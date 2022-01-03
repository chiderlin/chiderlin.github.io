---
title: GCP-load balancer 負載平衡服務之proxy pass
date: 2021-12-28 16:45:14
tags:
- load balancer
- nginx
- google cloud
- 負載平衡
categories: 雲端
---

## 前言
goole的load balancer呢，給我的感覺是此load balancer非彼load balancer，因為呢，GCP的load balancer不限於處理流量的問題，還能處理proxy的問題，我的理解會是nginx，而google cloud也開設這樣的服務，所以是nginx的雲端版，而我這篇文章記錄的主要是proxy pass的部分，真正負載平衡的部分沒有特別設定，但gcp也會auto scale就是了。
經過公司這麼走一遭，關於導domain也讓我深有感想。

## GCP:load bablancer功能

## 設定

## proxy

## 資料參考：
[大架構的概念與程式設計－－（三）Load Balance（又叫分流、負載平衝）設計](https://akuma1.pixnet.net/blog/post/168326978-%E5%A4%A7%E6%9E%B6%E6%A7%8B%E7%9A%84%E6%A6%82%E5%BF%B5%E8%88%87%E7%A8%8B%E5%BC%8F%E8%A8%AD%E8%A8%88%EF%BC%8D%EF%BC%8D%EF%BC%88%E4%B8%89%EF%BC%89load-balance)
[如何透過 GCP HTTP(S) Load Balancing 來實現 HTTP Redirect HTTPS](https://blog.cloud-ace.tw/networking-website/load-balance/how-to-implement-http-redirect-https-through-gcp-https-load-balancing/)
[使用 Nginx 做 Load Balancer](https://blog.dtask.idv.tw/Nginx/2018-07-31/)
[Choosing the right load balancer in Google Cloud](https://medium.com/google-cloud/choosing-the-right-load-balancer-9ec909148a85)