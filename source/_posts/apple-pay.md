---
title: Apple pay on web串接/解密紀錄
date: 2022-12-17 17:21:26
tags:
    - Nodejs
    - apple pay
    - apple pay on web
    - https
    - Payment Request API
categories: 第三方串接
---

## 前言

前端使用 Javascript
後端使用 Nodejs

> 注意：
>
> 1.  這篇文章是紀錄實作 Apple pay on web，且沒有透過第三方金流平台串接。
>
> -   apple pay 按鈕
> -   使用者付費
> -   解析 token
>
> 2.  apple pay on web 只有 Safari 瀏覽器可以使用
> 3.  apple pay 有兩種 API
>
> -   Apple Pay JS API (apple pay 原生的 API)
> -   (我用的) Payment Request API(遵照 W3C 準則而出的 API)

＊如果有和第三方金流平台配合例如：Tappay，他們的平台有簡化步驟，幫助大家更快速簡單串接 apple pay, google pay 等等的支付方式。他們也提供很完整的[文件解說](https://docs.tappaysdk.com/apple-pay/zh/portal.html#create-merchant-id)

即使我沒有使用第三方金流平台，他們的操作步驟還是有很多可以參考的地方。

在正文開始之前需要先準備好以下：（這邊操作可以參考 Tappay 文件，這邊就不會再紀錄）

1. 註冊 Apple ID
2. APPLE ID 加入 Developer Program（要付費)
3. 有了 Developer Program 之後
    - 申請 Merchant ID
    - 申請 Payment Processing Certificate：解密時會使用到！
    - 申請 Merchant identity Certificate：自己 Server 和 Apple Pay Server 認證時會用到！
      [官方文件](https://developer.apple.com/documentation/apple_pay_on_the_web/configuring_your_environment)
4. 準備好 Https 環境。檢查：
    - Https 環境
    - TLS 版本在 1.2 或以上
    - 你網站的 Cipher suites 有包含在 apple 官方文件裡面
      [官方文件](https://developer.apple.com/documentation/apple_pay_on_the_web/setting_up_your_server)
      [可以檢查環境的網站](https://www.ssllabs.com/ssltest/index.html)

## 正文

### 流程

![apple pay流程](1.png)

### .csr 到.cer 憑證的過程

-   Payment Processing Certificate
-   Merchant identity Certificate

都是以下流程從.csr 到.cer

1. 自己生成.csr 檔案及 Private Key.key
    - Private Key 自己保留好。
    - .csr 上傳到 Apple Pay 平台取得 .cer 憑證
      (Merchant identity Certificate => apple 產生 merchant_id.cer)
      (Payment Processing Certificate => apple 產生 applepay.cer)

＊如果用 Tappay，會直接從 Tappay 平台取得他們生成的.csr。

> .csr 是憑證簽署要求(Certificate Signing Request)

Merchant identity Certificate 生成.csr & private key.key
`openssl req -new -newkey rsa:2048 -nodes -keyout applepay.key -out applepay.csr -subj '/O=Subscribe Pro/C=US'`
openssl req -new -newkey rsa:2048 -nodes -keyout applepay.key -out applepay.csr -subj '/O=example.Ltd(公司名)/C=TW(國家)'

[有用的參考 Create Merchant Identity Certificate in Apple Developer Account](https://docs.subscribepro.com/integrations/magento-1/apple-pay/merchant-certificate/)

Payment Processing Certificate 生成.csr & private key.key
`openssl ecparam -out server.key -name prime256v1 -genke`
`openssl req -new -key server.key -out server.csr -sha256`
[有效參考 Apache: Create ECC CSR and Install ECC SSL Certificate](https://www.digicert.com/kb/ecc-csr-creation-ssl-installation-apache.htm)

### Apple Pay 按鈕

這邊有前端完整的 code 可以參考/實際操作/也有講解。可以自己玩玩。
[官方 Demo 參考：用 Safari 實際操作](https://applepaydemo.apple.com/payment-request-api)

畫面會是這樣
![apple demo](2.png)

### Apple Pay 付款認證階段

Server 驗證 Apple Pay Server 相關文件比較少，我在認證這邊卡了不少時間。
這邊只有提到要認證的 endpoint 是前端`onvalidatemerchant事件`返回的 event.validationURL 要傳到後端，由後端打 POST 到這個 validationURL 連帶 Payload 一起過去驗證。
[官方文件](https://developer.apple.com/documentation/apple_pay_on_the_web/apple_pay_js_api/requesting_an_apple_pay_payment_session)

#### 卡關部分：

-   Payload 部分我覺得範例寫的不是很清楚
-   然後我是使用 Axios，傳 Payload 方式不同
-   傳憑證的部分不知道怎麼傳
-   一開始不清楚上面的 Key 是什麼？

![官方Server驗證說明](3.png)

#### 解決：

1. 取得的.cer 憑證要轉成.pem 檔案，將.pem 放入 payload 傳送給 apple pay server
   `openssl x509 -inform der -in certificate.cer -out certificate.pem`

2. key 是產生.csr 也一起產生的 private key.key，因為.cer 裡面的資訊會包含 public key，所以提供 private key 來驗證你是這個 certificate 的 owner

3. 如果使用 Axios 的話就會是長這樣

    ```js
    const merchIdentityCert = fs.readFileSync(
        path.join(__dirname, '../', 'merchant_id.pem'),
        'utf-8'
    );
    const private_key = fs.readFileSync(
        path.join(__dirname, '../', 'applepay.key'),
        'utf-8'
    );
    const payload = {
        merchantIdentifier: 'merchant.com.xx.xx', // merchantID....
        displayName: 'example Ltd', // 公司名
        initiative: 'web', // 種類web
        initiativeContext: 'example.com', //你DOMAIN名字
    };
    const httpsAgent = new https.Agent({
        rejectUnauthorized: false,
        cert: merchIdentityCert, // 憑證放這
        key: private_key, // pk放這
    });

    const rs = await axios({
        url: authorizeUrl, // 從前端validationURL來的
        headers: {
            Accept: 'application/json',
            'Content-Type': 'application/txt;charset=UTF-8',
        },
        httpsAgent,
        method: 'post',
        data: JSON.stringify(payload),
    });
    ```

解決了以上的問題！
Apple Pay Server 認證成功會回傳 Payment session Object

```
{
    epochTimestamp: 1670136214313,
    expiresAt: 1670139814313,
    merchantSessionIdentifier: 'REDACTED',
    nonce: '757ae09c',
    merchantIdentifier: 'REDACTED',
    domainName: 'applepaydemo.apple.com',
    displayName: 'Apple Pay Demo',
    signature: 'REDACTED',
    operationalAnalyticsIdentifier:
        'Apple Pay Demo:A77873CD368A460BD5D3325AD76B01C16BB7F838CFFF654F9A993F4B6A9B4098',
    retries: 0,
    pspId: 'A77873CD368A460BD5D3325AD76B01C16BB7F838CFFF654F9A993F4B6A9B4098',
};
```

### Touch ID 付款

使用者 Touch ID 付款結果會透過`.show()`返回 promise 結果，也就是 Token

```
Payment Token: {
    "paymentData": {
        "version": "EC_v1",
        "data": "TQf1foe8j/aDJZAMO/9z0q1Ixslykn0isD5ky/jS4OHgDBByMSfbUxUZIJ0+HjUpftl1biXgBbAQ42sZwjZh2UXc7qhZLbvVZ94DthODEQMtsgS3Jd70aB86cdmf390jRdxP08oeXgY0hqG4h16CZ8r542Rs6viVu5ViSTbP42qcfRapsm09mVSOD65/VJTDkSToHCZVmkK/xpnc9Gn+1CwMXjFXcyXAQFUNhM48ldDSUHDbvFyZ/Ip7A8hJnvV0ZodRtDqNTehYO/Z2zcC0oGrmW9u/CRoYKTuh5p4H61VdejPenruyNulkw3BWqVnCFzdR9/ZQ9jlg5zY486GabpUwMc/TtR9Xdq2hz4FoIcT9FxNqxJ0dXGb9F9yGgfU2qarlVSyJsoPSQ0PC54pmstG/RaLnhYRvrXxzyireib0=",
        "signature": "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",
        "header": {
            "ephemeralPublicKey": "MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE9MQf5OWHgrinv0m9LAc4CoKckDkcDN37cp3k4TX5pArfMeVXzrzHSzdvppsKgdQEgpe/9qXQ9M0lJ1A8VpnpYg==",
            "publicKeyHash": "DUk3mLvfQgqZFWz9r45AsSC7Ew7mcDnbyChqPVfSG3c=",
            "transactionId": "97daac01e31dcb2373191b34532ebd0f7fef29797cbc2c9526fbf4aff3ca5625"
        }
    },
    "paymentMethod": {
        "displayName": "Visa xxxx",
        "network": "Visa",
        "type": "credit"
    },
    "transactionIdentifier": "97DAAC01E31DCB2373191B34532EBD0F7FEF29797CBC2C9526FBF4AFF3CA5625"
}
```

### 解密相關

以上取得的 Token 就是付款的加密資料，Apple Pay 也有提供解密的方式
解密這部分涉及密碼學相關 QQ。
官方說明裡也有對 Token 欄位的說明，其中 data 就是要解密的 Token
![token欄位說明](4.png)

解密步驟像文章提到的

```
1. Verify the signature
2. Determine which merchant public key was used
3. Restore the symmetric key
4. Use the symmetric key to decrypt the value of the data key
5. Confirm that this payment has not yet been credited
6. Verify the transaction details: currencyCode, transactionAmount, and applicationData
7. Use the decrypted payment data to process the payment
```

[官方解密步驟說明](https://developer.apple.com/documentation/passkit/apple_pay/payment_token_format_reference)
[參考：輕鬆解密 Apple pay payment token](https://medium.com/@kenchen_57904/%E8%BC%95%E9%AC%86%E8%A7%A3%E5%AF%86apple-pay-payment-token-6e7b900e8e0c)

如果自己解密以我的功力只能說會耗費非常之久，
找到有人有套件可以完成解密的繁雜步驟！
[套件相關](https://github.com/Kenblair1226/apple-pay-signature-and-decrypt)

有清楚寫下使用方法

```js
//取得到的token
const requestToken = {
    data: '<encryptedData>',
    version: 'EC_v1',
    signature: '<signature>',
    header: {
        ephemeralPublicKey: '<ephemeralPublicKey>',
        publicKeyHash: '<publicKeyHash>',
        transactionId: '<transactionId>',
    },
};

// import Payment Processing Certificate的pem檔案
const publicCert = fs.readFileSync(
    path.join(__dirname, './publicCert.pem'),
    'utf8'
);
// import Payment Processing Certificate的private key
const privateKey = fs.readFileSync(
    path.join(__dirname, './privateKey.pem'),
    'utf8'
);

const token = new applePayPaymentToken(requestToken);
const decryptedToken = token.decrypt(publicCert, privateKey);
decryptedToken
    .then((ret) => {
        console.log(ret);
    })
    .catch((err) => {
        console.error(err);
    });
```

#### Issue

下載套件時，發現開發者使用的其中一個套件已經沒有再維護了，所以下載會失敗。

#### 解決

於是我先把這個 repo 拉下來，試圖解除裡面的 bug。

1. 發現`fix:node-webcrypto-ossl`套件不再使用，我改成`@peculiar/webcrypto`套件

    載入方法也從`const crypto = new Crypto.Crypto()`
    改為`const crypto = new Crypto()`

2. 原本裡面的 AppleRootCA-G3.cer 可以沿用，如果過期可以自行到[APPLE PKI](https://www.apple.com/certificateauthority/)下載 AppleRootCA-G3.cer，一樣要附在程式，因為這個解密會需要去開啟 AppleRootCA-G3.cer 做 signature 驗證
   ![AppleRootCA-G3.cer](5.png)

解密完成就會是這些欄位
![解密出來的object欄位](6.png)

## 心得

一切都還只是在串接階段、測試環境，希望之後到正式環境階段沒差別。
研究+卡了好幾天，對憑證的申請/交換更了解，但解密方面，因為有找到相關套件，所以還沒有深入了解，還需要更多時間了解解密程序，還有密碼學相關知識。

之後 code 整理好，會放在這。

---

其他相關資料來源：
[Apple Pay on Web + Cybersource 串接筆記](https://blog.markkulab.net/2022/08/10/apple-pay-on-web-cybersource-integration/)

[关于 Apple Pay 接入和开发](https://zhuanlan.zhihu.com/p/45068888)
