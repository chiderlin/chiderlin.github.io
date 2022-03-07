---
title: Node的buffer和stream概念加一點google storage實作
date: 2021-12-07 21:21:25
tags: 
- Nodejs
- buffer
- stream
- file
- Google storage
- GCS
- Google cloud
categories: Nodejs
---

## 前言 
雖然實作過檔案之間的傳送或編寫/覆蓋新檔案，但發現自己對底層實作的原理完全沒有概念，加上這次在研究GCS的實作，因此有了這篇文章的誕生。學習程式之後察覺，每個部分都可以越挖越深，越看越細。

## Stream - 流
什麼是Stream流？ I/O的傳輸=>資料輸入輸出，有資料從一端流向另一端的感覺。HTTP請求、檔案傳輸都會使用到Stream的概念。
有四種不同的流：
1. Readable streams 創造讀取的流
2. Writable streams 創造寫入的流
3. Duplex streams 可讀可寫流
4. Transform streams 基於Duplex，在讀寫過程可以調整數據的流
在實作上我們常用的fs套件就是base on 原生的stream再包裝，所以在fs裡面也可以看到很多stream的使用方法。

## Buffer - 緩衝區
二進制緩衝區，和String可以互相做轉換。Stream的傳輸會將數據儲存到內部的緩衝區(buffer)中，buffer容量滿時再流向另一端，這麼做可以減少stream直接對磁碟頻繁的操作。而緩衝器的容量大小也可以藉由`highWaterMark`來做設定。

---

**附上fs操作檔案的實作**
``` js
const fs = require('fs');
const read = fs.readFileSync('./test1.txt'); // return buffer type => <Buffer 71 77 65 65 72 74 79 79>

const readstream = fs.createReadStream('./test1.txt');// return stream type 
// 兩種方式都可以用來傳輸或寫入新檔案，只是用法上的不同
```

> _如果使用`createreadstream`做傳輸的話，會使用pipe()來做搭配，資料在流動的時候需要有一個管道讓它進行傳輸_
而有趣的是，read & write的用法是有對應的
``` js

const readable = fs.createReadStream('test1.txt');
const writeable = fs.createWriteStream('test2.txt')

// 讀取檔案再把內容傳到新檔案
readable
.on('error',(err)=>{
    // 處理過程出現錯誤會觸發此事件
})
.on('data',(chunk)=>{ // 2
    console.log(chunk) //數據在這裡被轉成buffer
    //再由buffer把資料傳出去
})
.on('end',()=>{ // 3
    // 已經沒有數據需要解析
    console.log('there will be no more data.')
})


writeable
.on('pipe',(stream)=>{ // 1
    console.log(stream) // 接收stream時觸發事件 
})
.on('finish',()=>{ // 4
    console.log('數據寫入新檔案完成會觸發')
})

readable.pipe(writeable); //用管道進行資料傳送，傳送到可寫流，也就是test2.txt的檔案
```

以上的執行順序會是
1. writeable打開pipe管道，開始接收readable的stream
2. readable的data事件觸發，開始將數據轉換成buffer
3. readable的end事件觸發，數據已全數轉換完畢
4. writeable的finish事件觸發，數據全數完成寫入新檔案

還有其他的`事件`可以到[官方文件](https://nodejs.org/api/stream.html)看看

---
>因為使用方法很多，這裡我是想自己記錄另外一些使用方法，
_自行將檔案轉成buffer再透過可寫流寫入檔案，
也就是使用上面的readFileSync方法_
``` js 
// 1
const fs = require('fs');
const readbuffer = fs.readFileSync('./test1.txt');
const writeable = fs.createWriteStream();
writeable.write(readbuffer,()=>{
    console.log('write successfully')
});

// 2
const fs = require('fs');
const readbuffer = fs.readFileSync('./test1.txt');
const writeable = fs.createWriteStream();
writeable
.on('finish',()=>{
    console.log('完成')
})
.end(readbuffer,()=>{
    // 會在finish事件觸發完後執行最後一個寫入的動作，因此也可以直接在這邊寫入buffer
    // 寫完便會關閉流
    console.log('最後一個想寫入的東西，寫入完成')
})

```


## Google Storage實作檔案傳輸
google storage，google提供的雲端空間，用來存放檔案或圖片，和AWS的S3是相同功能，實務上用來將網頁上傳的檔案跟圖片存放到這邊來，成功之後就可以返回一個url提供我們下載自己上傳的檔案&呈現在自己的網頁上。

1. 必須先npm `@google-cloud/storage`才能連接google storage空間
2. 必須先到GCP取得憑證，利用憑證才能進入google storage空間
3. 提供GCS bucket name => 也就是你在google storage中開創的空間名稱
**完成以上才會開始寫程式**
_*這邊沒有寫太多GCS說明，主要注重在傳輸檔案的部分_

``` js

const { keyFilename } = require('./config'); //存放我的設定檔，裡面有憑證檔的路徑
const { Storage } = require('@google-cloud/storage');
const storage = new Storage({ keyFilename }); //提供憑證來初始化

let bucket = storage.bucket('your_bucket_name') //連線到google storage 你創的空間

/*
* @param {String} buffer - 要傳送的檔案的buffer
* @destination {String} destination - 到目的地（雲端）後檔案的名稱
**/
const uploadFromBuffer = (buffer, destination) => new Promise((resolve, reject)=>{
    const file = bucket.file(destination) //先創造檔案物件
    const ws = file.createWriteStream(); //創造可寫流
    ws
    .on('error',(err)=>{
        reject(err);
    })
    .on('finish',async()=>{
        resolve(await file.publicUrl()); // 會回傳檔案的url
    })
    .end(buffer); //寫入資料囉
    
});

```
會看到`@google-cloud/storage`的使用也有`createReadStream`, `createWriteStream`的方法，表示他們也有使用stream再包裝～


## 前後端上傳檔案實作
以上是單純上傳到google storage的程式碼，這部分通常都會寫在後端，現在想實作從前端串接後端API再上傳至GCS上，前端到後端API的這段我也研究了一些時間，原因是因為跟postman的不熟悉？以及傳送檔案這樣的實物和傳送JSON感覺上不同而導致在操作上會有很多疑惑，許多不確定經過一試再試的精神才總算完成了！！

### 1. 前端使用 FormData   
之前也有實作過傳送檔案的方法，當時使用的方式是google搜尋後比較常見的方法，是使用FormData在前端抓取到資料，而他的實作順序是
1. 使用者上傳檔案，前端接收到data訊息
2. 打後端API傳送formData，檔案被傳送到專案的資料夾內，檔案相關資料用Multer接收
3. 後端API用stream傳送資料到GCS上
完成前到後的流程，如果想看相關教學可以看[這裡](https://medium.com/%E9%BA%A5%E5%85%8B%E7%9A%84%E5%8D%8A%E8%B7%AF%E5%87%BA%E5%AE%B6%E7%AD%86%E8%A8%98/%E7%AD%86%E8%A8%98-%E4%BD%BF%E7%94%A8-multer-%E5%AF%A6%E4%BD%9C%E5%A4%A7%E9%A0%AD%E8%B2%BC%E4%B8%8A%E5%82%B3-ee5bf1683113)


### 2. 前端直接傳送buffer到後端
用buffer傳到後端是這次新的學習，這邊紀錄一下。
後端的Buffer在前端是概括了ArrayBuffer、TypeBuffer、viewData，
ArrayBuffer是一大塊內存，只能唯讀，若想對ArrayBuffer暫存器做操作，要使用TypeBuffer的種類才可以，而TypeBuffer種類有Uint8array, Uint16array...等等，而此次前端的資料流就使用了Uint8array來傳送到Server端。

1. 前端使用file API - FileReader操作，將檔案讀取成ArrayBuffer格式
2. fetch給server時 body的data要將ArrayBuffer轉成Uint8array才可傳送
3. server接收到Uint8array後再轉成Buffer才可傳至gcs


upload.html
```html

<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>

<body>
    <input type="file" accept="" id="file-uploader">
    <input type="button" value="案 下去" id="btn">
    <script>
        const fileUploader = document.getElementById('file-uploader');
        const btn = document.getElementById('btn')
        let file;
        let filename;
        fileUploader.addEventListener('change', handleFileUpload);
        btn.addEventListener('click', upload);
        async function upload() {
            try {
                const beforeUploadCheck = await beforeUpload(file);
                if (!beforeUploadCheck.isValid) throw beforeUploadCheck.errorMessages;

                const arrayBuffer = await getArrayBuffer(file);
                await uploadFileAJAX(arrayBuffer, filename);
                console.log("File Uploaded Success");

            } catch (err) {
                console.log(err)
            }
        }
        async function handleFileUpload(e) {
            file = e.target.files[0]
            filename = e.target.files[0].name
            console.log(file)
            console.log(filename)
            if (!file) return;
        }

        function beforeUpload(fileobj) {
            return new Promise((resolve) => {
                const validFileTypes = ["image/jpeg", "image/png", "xlsx"];
                const isValidFileType = validFileTypes.includes(fileobj.type)
                let errorMessages = [];

                if (!isValidFileType) errorMessages.push('You can only upload JPG or PNG file!');

                const isValidFileSize = fileobj.size / 1024 / 1024 < 2;
                if (!isValidFileSize) errorMessages.push('Image must smaller than 2MB!');
                resolve({
                    isValid: isValidFileType && isValidFileSize,
                    errorMessages: errorMessages.join("\n")
                });
            });
        };


        function getArrayBuffer(fileobj) {
            return new Promise((resolve, reject) => {
                const reader = new FileReader()
                // get ArrayBuffer when fileReader on load
                reader.addEventListener('load', () => {
                    console.log(reader)
                    resolve(reader.result)
                });
                reader.addEventListener('error', () => {
                    reject('error occurred in getArrayBuffer');
                });
                // read the blob object ad ArrayBuffer
                reader.readAsArrayBuffer(fileobj)
            });
        };

        function uploadFileAJAX(arrayBuffer, filename) {
            return fetch("/upload/buffer", {
                headers: {
                    "content-type": "application/json",
                },
                method: "POST",
                body: JSON.stringify({
                    uint8array: new Uint8Array(arrayBuffer), //**** important
                    filename: `test/${filename}`

                })
            })
                .then(res => {
                    return res.json()
                })
            .then(data => console.log(data))
            .catch(err => console.log(err))
        };
    </script>
</body>

</html>

```

express接收部分
```js
app.post('/upload/buffer', async (req, res) => {
    const _uint8array = req.body.uint8array // 原本傳進來是json
    const _filename = req.body.filename
    const uint8array = Uint8Array.from(Object.values(_uint8array)) //把value抓出來變成array
    const buffer = Buffer.from(uint8array) // array才可以轉成Buffer
    const g_url = await GCS.uploadFromBuffer(buffer, _filename) //呼叫上面gcs寫的uploadFromBuffer function上傳到google storage
    return res.json({ status: "OK", data: g_url }) //成功
})

```



[參考這篇的範例程式碼](https://pjchender.dev/webapis/webapis-file-input/)

## 心得：
邊整理資料邊整理思緒，想了一遍流程後思路也更清晰一些，但寫文章關於架構的整理，想著要怎麼寫才能清楚一點紀錄整個過程還是挺花時間的就是。以上就是近日來了解的部分，若說要研究肯定還可以再看更多細節QQ，只能多看看文件，相信會跟它們熟一點～以上


資料來源：
[认识node核心模块--从Buffer、Stream到fs](https://segmentfault.com/a/1190000011968267)
[瞭解Node.js中的Stream](https://tw511.com/a/01/23038.html)
[Node.js流](https://tw511.com/6/73/2292.html)
[Streams and Buffers in Node.js](https://medium.com/developers-arena/streams-and-buffers-in-nodejs-30ff53edd50f)
[官方文件](https://nodejs.org/api/stream.html#stream)
[官方文件中文](https://www.nodeapp.cn/stream.html#stream_writable_end_chunk_encoding_callback)
[google storage SDK文件](https://googleapis.dev/nodejs/storage/latest/Bucket.html)
[Node.js 大檔案上傳](https://hackmd.io/@c36ICNyhQE6-iTXKxoIocg/ByLUTZzhv)
[[WebAPIs] 檔案上傳 Input File, File Upload, and FileList](https://pjchender.dev/webapis/webapis-file-input/)
[nodejs 处理文件上传(express)](https://blog.csdn.net/m0_37263637/article/details/104560862)