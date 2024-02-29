---
layout: craft
title: x-csrf-token_note
date: 2023-09-06 15:49:23
tags:
---

# 前言

重新研究當初爬蟲 worldGym 時，需要在 request 帶入 x-csrf-token 才能爬到資料，想探究 x-csrf-token，主要想解決怎麼直接利用程式取得 x-csrf-token 並完成爬蟲功能，而不使用手工取得。

# 研究過程

x-csrf-token 和記錄在 cookie session 是有相關的，x-csrf-token 估計有加密從在 cookie 中，當我把 Application > Cookies > laravel_session 刪除後 reload 網頁，x-csrf-token 就會產生一個新的。

有另一個 XSRF-TOKEN 刪除後沒什麼關聯。

# 思考方向

-   是否可以第一次 fetch 到網站，取得到 x-csrf-token，第二次 fetch 就可以附上 x-csrf-token 爬取到資料。

-   是否可以取得到 cookie 的資料，解密 cookie 資料(應該包含 x-csrf-token)，第二次 fetch 就可以附上 x-csrf-token 爬取到資料。
