---
title: 用Goole build和Google run來部屬你的專案
date: 2021-12-30 19:18:44
tags:
- Google Cloud
- Google build
- Google run
- docker
- 部署
categories: 雲端
---

## 前言
一開始先接觸AWS的雲端服務，最近才開始學習使用GCP的服務，之前部署專案的經驗是使用AWS的EC2（對應GCP應該是Compute Engine），GCP我還沒自己開過機器，反而先接觸到了另一個服務，也就是Google run，對於想專心開發業務邏輯的工程師來說，可以更快速地處理部署，俗稱的Serverless。
>__Serverless?__
工程師不需要管理伺服器機器，而是讓google cloud來幫你處理大部分的事項，你只需要將你的專案交付給google cloud啟動你的服務。


## Serverless服務
1. Cloud function: 對應AWS的Lambda。Serverless for function
2. App Engine: Serverless for App
3. Google run: Serverless for container支援Kubernetes

## 部署前：
TODO 可以用一個簡單的express專案部署截圖上來
1. 部署前請先準備好你的專案
2. 開通好Google Cloud帳號

TODO（附圖）
3. google build跟GitHub連線，才能抓到你的專案
4. 在google run建立一個服務，裡面需要設定抓取github的哪個專案

## 準備部署：
在你的專案內準備兩個東西，都是google cloud要啟動你的服務時所需要的
1. google build需要的cloudbuild.yml
2. google Run需要的Dockerfile，因為google run是for container的～所以他會吃你的dockerfile包出docker image再啟動container服務。
而以下就會記錄要怎麼寫這兩個檔案囉

### cloudbubild.yml
TODO 寫yml範例
```yml
steps:
# name的名稱是google中的環境，會抓到對應的docker應用程式
# args裡面放docker的指令，這裡會形成docker image
- name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t','gcr.io/$PROJECT_ID/test-cloud/express-image:latest','.']

# 在google預設中$PROJEC_ID會自己抓到你的專案ID
- name: 'gcr.io/cloud-builders/docker'
  args: ['push','gcr.io/$PROJEC_ID/test-cloud/express-image:lastest']

- name: 'google/cloud/sdk'
  args: ['gcloud','run','deploy','cloud-run-hello','--image','gcr.io/$PROJECT_ID/test-cloud/express-image:lastest','--region','asia-east1']

images:
- 'gcr.io/$PROJECT_ID/test-cloud/express-image:latest'

```

### Dockerfile
TODO 寫dockerfile
```docker
```

## 部署成功
TODO（附圖）
會看到Ｖ圖示，且google run會直接給你一個domain，點進去就可以看到你的網站囉。

## 心得：
成功部署之後真的覺得對工程師來說很方便，前端工程師也可以痛個幾天入門，就可以快速部署上去，然後下一篇可以接續著記錄load balancer的服務，連貫的完成一個架站的過程。

## 資料參考：
[官方文件](https://cloud.google.com/run)
[Google Cloud Run详细介绍](https://www.servicemesher.com/blog/google-cloud-run-intro/)
[Cloud Build Deploying to Cloud Run](https://cloud.google.com/build/docs/deploying-builds/deploy-cloud-run)
[cloudbuild - bash指令寫法](https://cloud.google.com/build/docs/configuring-builds/run-bash-scripts)
[gcloud run deploy - 官方文件](https://cloud.google.com/sdk/gcloud/reference/run/deploy)
[bash指令區](https://tldp.org/LDP/abs/html/standard-options.html)