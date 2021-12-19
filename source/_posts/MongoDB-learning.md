---
title: MongoDB 學習之旅-1 Mongoose
date: 2021-11-26 20:47:09
tags:
- database
- 資料庫
- NoSQL
- Mongoose
- MongoDB
categories: 資料庫
---

## 前言
之前自己做專案時都是使用MySQL搭配居多，公司是使用Node，也因為公司需求最近開始踏上MongoDB學習之旅，此篇文章我主要紀錄的是後面的小方法可以帶出來的功能，CRUD的部分就不特別寫。

## Node套件部分
以Node來說常見的套件我知道有兩個，而今天主要是來整理mongoose的一些用法。
1. [mongodb](https://docs.mongodb.com/drivers/node/current/quick-start/)
這個是官方提供的套件，有提供一整套蠻詳細的使用方法。最近也開始練習直接看官方文件～
2. [mongoose](https://mongoosejs.com/docs/index.html)
很多剛開始寫mongodb的人應該最多人是使用mongoose，我的感覺是ODM的方式讓他在管理上很淺顯易懂吧。
    > _ODM是什麼？=> Object Document Mapping
    跟MySQL的ORM是同樣的意思_ 

    >_於是又會問ORM是什麼？ => Object Relational Mapping
    簡單來說是： 把資料庫操作轉成抽象的程式碼物件的使用方式
    在google上面搜尋一下都有蠻清楚的解釋[可看](https://ithelp.ithome.com.tw/articles/10185085)

## 一些mongoose語法紀錄 
### .lean() ⇒

讓回傳回來的資料容量變小

```js
const schema = new mongoose.Schema({ name: String });
const MyModel = mongoose.model('Test', schema);

await MyModel.create({ name: 'test' });

// Module that estimates the size of an object in memory
const sizeof = require('object-sizeof');

const normalDoc = await MyModel.findOne();
// To enable the `lean` option for a query, use the `lean()` function.
const leanDoc = await MyModel.findOne().lean();

sizeof(normalDoc); // approximately 600
sizeof(leanDoc); // 36, more than 10x smaller!**
```

[來源](https://mongoosejs.com/docs/tutorials/lean.html)

### .exec() ⇒
其實用法有三種差異，我這邊用query Posts來測試
1. `Posts.findOne({name:'chi'})` => return a Query {}
如果要取用裡面資料會需要再使用`.then((doc)=>{})`把資料抓出來

2. `Posts.findOne({name:'chi'}).then()` => return a Promise {}
假議題，就是上面的用法用.then來接

3. `Posts.findOne({name:'chi'}).exec()` => return a Promise {}
也是回傳promise，那跟上面有什麼不同？
如果沒有用async/await，exec()返回來的資料也是要用.then把資料抓出來，他的優點在於其他地方：
    - 如果出現error更好的追蹤是在你程式碼哪一行出錯
    - 回傳的promise是fully-fledged promise
        > _什麼是fully-fledged promise?_
        _1.使用.exec()執行出來的程式碼是真的Promise，Promise所有的方法都有。
        2.而上面1程式碼是queries，然後可以使用.then()來接，但.then()接出來的資料不是真正的Promise，有些Promise原生的方法像catch, finally，這邊都不能使用。
        [來源](https://stackoverflow.com/questions/53470299/what-is-fully-fledged-promise)_




``` js

// 擷取測試結果的其中一小部分，這個程式碼有搭配express來query data
app.get('/test', async (req, res) => {
    const query = Posts.findOne({ name: 'chi' })
    // console.log(query) => return a Query {}

    const promise = Posts.findOne({ name: 'chi' }).exec()
    // console.log(promise) => return a Promise {<pending>}

    const then = Posts.findOne({ name: 'chi' }).then()
    // console.log(then) => return a Promise 假{<pending>}

    try {
        // 這三個正確，不會報錯
        assert.ok(!(query instanceof Promise))
        assert.ok(promise instanceof Promise)
        assert.ok(then instanceof Promise)
    } catch (e) {
        console.log(e)
    }
    return 'ok'
})

```
以上是為了測試他本身回傳回來的東西，但現在我們基本上都一定會搭配async/await來寫程式碼（對吧！！），如果是使用async/await都會成功回傳資料的，差別還是在於我剛列出來的那兩點，
**1 .exec()可以追蹤錯誤程式碼** => 是建議使用.exec()的喔
**2 .exec()回傳真正的promise**
```js

const query = await Posts.findOne({ name: 'chi' })
const promise = await Posts.findOne({ name: 'chi' }).exec()

// 這邊就沒有在列出.then()用法了，因為他是假議題

```

```json
{
    "_id": "6190c61ede7ec25305b1f563",
    "name": "chi",
    "title": "Post 1",
    "__v": 0
}

```
[官方解釋，我就是個翻譯仔](https://mongoosejs.com/docs/promises.html)



### .select() ⇒

選取特定欄位query出來，裡面有其他用法，可以傳遞參數的方式有很多，
所以用法上也很彈性，可以看到比較多不同的寫法。
直接貼官方code來偷懶。
```js
// 只選a,b欄位
query.select('a b');
query.select(['a', 'b']);
query.select({ a: 1, b: 1 });

// 不要c,d欄位其他都要
query.select('-c -d');

//原本foo select設定false
const schema = new Schema({
  foo: { type: String, select: false },
  bar: String
});
//這個語法改寫foo欄位的select:false，讓他顯示此欄位
// bar也會顯示，不會只顯示foo
query.select('+foo'); 

//自行定義1 or 0 ， 1顯示/0不顯示
query.select({ a: 1, b: 1 });
query.select({ c: 0, d: 0 });
```

```js
// 1 
//最新用法，就不用.select()，直接在第二個參數選取要的欄位
// query.select('a b');
Model.find({username : user.username}, 'first last', function (err, docs) {

});

// 如果要排除特定欄位則用 -
//query.select('-a -b');
Model.find({username : user.username}, '-first -last', function (err, docs) {

});

// 2
// select用法
Model.find({username : user.username})
.select('first last') // first, last欄位
.exec() // 執行query

//其他相等用法
// Equivalent syntaxes:
Model.select(['first', 'last']);
Model.select({ first: 1, last: 1 });
```


[官方](https://mongoosejs.com/docs/api/query.html#query_Query-select)

[其他來源](https://stackoverflow.com/questions/9548186/mongoose-use-of-select-method)



### .sort() ⇒

排序

```js
//由小排到大
Model.find().sort({_id:1})

//由大排到小
Model.find().sort({_id:-1})

//其他相等用法 asc(ascending), desc(descending), 1, -1
Model.sort({ field: 'asc', _id: -1 });
Model.sort('field -test'); // field小到大, -test大到小

```

[官方來源](https://mongoosejs.com/docs/api/query.html#query_Query-sort)
[其他來源](https://www.bmc.com/blogs/mongodb-sorting/)



## 創建Schema時可用expires來設定過期時間(TTL)

過期後，該筆資料自動刪除

```js
const mongoose = require('mongoose');

const schema = new mongoose.Schema({
  code: { type: String, index: true },
  created_at: { type: Date, default: Date.now, expires: 60 * 5 }, // 5分鐘後失效
});

module.exports = mongoose.model('sl2_captchas', schema);
```

[來源](https://www.itread01.com/content/1546830386.html)


## 紀錄
目前常碰到的是這幾個，之後如果有碰到別的用法將持續更新。