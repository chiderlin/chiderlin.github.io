---
title: JWT＋Refresh token觀念及實作驗證
date: 2021-11-25 22:36:44
tags:
- jwt
- Jsonwebtoken
- Refresh token
- Authentication
- Nodejs
categories: 網頁開發
---

### JWT ( JSON Web Token )組成
訪問資源的令牌，常用在身份驗證，驗證成功才可以獲取資訊，由明碼組成，可以在 jwt.io 查詢JWT裡存放的資料是什麼，所以不能放機敏資料，組成分成三個部分，分別是Header, Payload, Signature。

長這樣：Header.Payload.Signature 都轉為Base64編碼
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6IjEyMzQ1Njc4OTAiLCJuYW1lIjoiQ2hpIiwiaWF0IjoxNjM2NzkzOTY3LCJleHAiOjE2MzY3OTQyNTV9.2Ev3rSQwgIYJXVhv7gC5FrLcJdHE2kNliIYvAUd86ZU
```
- Header : 定義 Token 種類及雜湊演算法的地方。
    alg : 雜湊演算法使用HS256、type : 定義 Token 種類為JWT。

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9部分：
{
  "alg": "HS256",
  "typ": "JWT"
}
```


- Payload : 存放用戶相關訊息，通常放id、姓名等等，不可放機敏資料。可以另外設定 exp ( Token到期時間)、iat 設定 Token 簽發時間。

```
eyJpZCI6IjEyMzQ1Njc4OTAiLCJuYW1lIjoiQ2hpIiwiaWF0IjoxNjM2NzkzOTY3LCJleHAiOjE2MzY3OTQyNTV9部分：
{
  "id": "1234567890",
  "name": "Chi",
  "iat": 1636793967, => Token簽發時間，Unix時間(點我進去時間轉換網站）
  "exp": 1636794255 => Token到期時間，Unix時間(點我進去時間轉換網站）
}
```
<br/>

- Signature : 數位簽章，由Header(Base64編碼後)+Paylaod(Base64編碼後)+密鑰一起透過Header定義的HS256雜湊演算法加密產生的。密鑰在伺服器拿到Token之後會進行解碼，確認資料正確、secret正確，以驗證身份。
```
2Ev3rSQwgIYJXVhv7gC5FrLcJdHE2kNliIYvAUd86ZU部分：
HMACSH256(base64UrlEncode(header)+'.'+base54UrlEncode(payload),
secret)
```
<br/>

### JWT工作流程
![](workflow.png)

### 加上Refresh Token
機制符合OAuth2.0規範，JWT＋Refresh Token的機制也可以減少存取Redis的次數。Token分成：Access Token, Refresh Token。
- Access Token: 用來存取資料的Token ，時效較短(ex: 30 mins)。
- Refresh Token: 用來取得Access Token ，時效較長(ex:30天)。如果Refresh Token也過期了，用戶就必須重新登入了。
<br/>

### 加入Refresh Token則管理流程變更如下：
資料庫已經備好一些簡單的假資料方便實作，建立了三個表分別是access_tokens(token資料), posts(文章資料), users(使用者資料)，這邊著重在jwt的demo，有很多東西都先簡化處理。

1. 這邊我輸入name當作登入，後端驗證完取得access token及refresh token回傳給前端
![](1.png)
<br/>

2. 將access token 放到Bear token ， 送出get ‘/posts’的請求，可取得該使用者po過的文
![](2.png)
<br/>

3. access token過期之後把登入取得的refresh token 打 post ‘/token’ API，後端驗證可取得新的access token回傳給client端
![](3.png)
<br/>

4. 新的access token 即可再用Bear token再次取得該使用者posts的資料
![](4.png)


5. 登出=> 清除db裡的refresh token，搭配網頁的話，cookie也清除


>_Refresh Token用途，_
1. 表明Client端已經從用戶那裡獲得訪問數據許可，因此可以再次請求新訪問令牌。
2. 與長期Access Token相比，加入Refresh Token助於增加整個安全流程。
3. Refresh Token提升用戶體驗，不需一直做重新登入的動作。

### 實作JWT + Refresh Token ，附上程式碼
- index.js檔案引入套件
```js
//載入套件
require('dotenv').config()
const express = require('express');
const app = express();
const jwt = require('jsonwebtoken')
app.use(express.json());


// 連線資料庫
const mongoose = require('mongoose');
mongoose.connect(process.env.MONGO_URI)

// 載入model
const Posts = require('./model/post');
const User = require('./model/user')
const AccessToken = require('./model/access_token');

// 載入驗證middleware
const authenticateToken = require('./auth')
// 載入model


app.listen(3000)
```

- auth.js middleware驗證檔
```js
const jwt = require('jsonwebtoken');
require('dotenv').config()
module.exports = async (req, res, next) => {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];
    if (!token) return res.sendStatus(401) // 沒有token
    try {
        const payload = await jwt.verify(token, process.env.ACCESS_TOKEN_SECRET);
        req.user = payload;
        req.token = token;
        res.set('Authorization', `Bearer ${token}`)
        res.cookie('jwt', token)
    } catch (e) {

        // token沒有權限或者過期
        if (e.name === 'TokenExpiredError') return res.sendStatus(403)
    }


    next()
}


```


- 回到index.js=> 登入部分＆refresh token取得新access token程式碼
```js
//自己包一個產生access token的function，demo所以15秒就過期
const generateAccessToken = (payload) => jwt.sign(payload, process.env.ACCESS_TOKEN_SECRET, { expiresIn: '15s' });

// 登入
app.post('/login', async (req, res) => {
    // 取得前端的json資料（使用者名稱）
    const username = req.body.username;
    const user = { name: username }
    // 和db驗證是否有此使用者
    const _f = await User.findOne({ name:user.name })
    if (!username) return res.json({ error: true, msg: 'provide username' })
    if (!_f) return res.json({ error: true, msg: 'user not exist' })
    // 驗證ok產生access_token, refresh_token
    const accessToken = generateAccessToken(user)
    const refreshToken = jwt.sign(user, process.env.REFRESH_TOKEN_SECRET) // refresh token沒有設定期限，登出清除refresh token
    // 把access_token, refresh_token存到AccessToken表
    const newToken = await new AccessToken({
        user_id: _f._id.toString(),
        access_token: accessToken,
        refresh_token: refreshToken
    });
    await newToken.save();
    //回傳前端
    return res.send({ accessToken, refreshToken })

});

// access token過期時，用refresh token換新access token
app.post('/token', async (req, res) => {
    // 原本access token過期，呼叫/token api
    const refreshToken = req.body.token;
    if (!refreshToken) return res.sendStatus(401);

    // db驗證refresh token
    const find_r = await AccessToken.findOne({ refresh_token:refreshToken });
    if (!find_r) return res.sendStatus(403);
    try {
        const payload = await jwt.verify(refreshToken, process.env.REFRESH_TOKEN_SECRET);
        // 裡面有iat跟name，只需要name，重新產生iat
        const new_access_token = await generateAccessToken({ name: payload.name });
        const nowtime = Math.floor((new Date()).getTime() / 1000);
        // 更新AccessToken表的access token
        await AccessToken.updateOne(
            { refresh_token: refreshToken },
            { access_token: new_access_token, refresh_at: nowtime });
        return res.json({ new_access_token });

    } catch (e) {
        console.log(e.name)
        console.log(e, message)
        return res.sendStatus(403);
    }

});
```


- 登出程式碼
```js
app.delete('/logout', async (req, res) => {
    let token;
    if(!req.token) { // 我這邊直接用postman 帶上bear token，會在這邊取得token
        const authHeader = req.headers['authorization'];
        token = authHeader && authHeader.split(' ')[1];
        console.log(token)
    } else { // 有前端頁面的話，可從這裡取得資料
        token = req.token
    }
    if (!token) return res.sendStatus(401) // 沒有token

    // db驗證是否有此refresh token
    const oldAccessToken = await AccessToken.findOne({ refresh_token: token })
    if (!oldAccessToken) return;
    await AccessToken.deleteOne({ refresh_token: oldAccessToken.refresh_token })

    // 有前端頁面的話，清除cookie
    res.clearCookie('jwt');
    return res.sendStatus(204)
})
```

### 心得
剛開始接觸的時候一知半解，也是透過多次摸索慢慢提升自己對JWT的理解，希望能更靈活運用在專案上，以後回來再看一次JWT可能都還會有新的感悟，有一些基礎的概念，但實際運用的方式有很多種，相信還有很多可以優化的地方，至於如何能靈活運用還是要再造化。

---
資料來源：

[[Web] 以 axios 實踐前端 refresh token 機制](https://www.dotblogs.com.tw/wasichris/2020/10/25/223728)
[为什么 OAuth 里除了 Access Token 之外，还需要 Refresh Token？](https://segmentfault.com/a/1190000040410624)
[結合 JWT 與 Refresh Token 達到黑名單失效機制](https://tec.xenby.com/44-%E7%B5%90%E5%90%88-jwt-%E8%88%87-refresh-token-%E9%81%94%E5%88%B0%E9%BB%91%E5%90%8D%E5%96%AE%E5%A4%B1%E6%95%88%E6%A9%9F%E5%88%B6)
[是誰在敲打我窗？什麼是 JWT ？](https://5xruby.tw/posts/what-is-jwt)
[[筆記] 透過 JWT 實作驗證機制](https://medium.com/%E9%BA%A5%E5%85%8B%E7%9A%84%E5%8D%8A%E8%B7%AF%E5%87%BA%E5%AE%B6%E7%AD%86%E8%A8%98/%E7%AD%86%E8%A8%98-%E9%80%8F%E9%81%8E-jwt-%E5%AF%A6%E4%BD%9C%E9%A9%97%E8%AD%89%E6%A9%9F%E5%88%B6-2e64d72594f8)