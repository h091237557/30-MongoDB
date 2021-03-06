# 30-17之MongoDB的設計---正規與反正規化的戰爭
本篇文章將說要說如何設計`mongodb`的架構，讓你可以更快速的使用`mongodb`。

* 資料庫的正規化(文鄒鄒)。
* `mongodb`正規化與反正規化。
* 該選用那個方法呢 ? 


## ~ 正規化 ~
在開始討論`mongodb`架構時，有個東西要先講講，那就是『正規化』與『反正規化』，有使用過資料庫的應該都有聽過這名詞，不過這邊還是來解釋解釋，順到回憶一下。

首先什麼是正規化呢 ? 根據`wiki`的定義。
>Database normalization is the process of organizing the fields and tables of a relational database to minimize redundancy and dependency.

中文意思為。
> 資料庫正規化就是指將關聯式資料庫的欄位與表單進行讓『資料重複性與相依性』能夠降到最低的組織過程。

是的，真的很文鄒鄒，不過我們只要知道正規化的目的是解決資料的『重複性』與『相依性』這兩個點就夠囉，資料庫正規化有一些規則，每條規則都稱為『正規形式』，符合第一條規則就稱為『第一正規形式』，總共有不少條，但通常到『第三正規形式』就被視為最高級的正規形式，
下面來簡單的說明一下這幾條規則。

### 第一正規形式
以下條列為第一正規形式的規則，事實上重點還是在說『不要有重複群組』。

* 刪除各個資料表中的重複群組。
* 為每一組關聯的資料建立不同的資料表。
* 使用主索引鍵識別每一組關聯的資料。

我們假設資料為每個人的交易資料，下表為違反正規化的資料結構，因為它有重複的群組`Volume`，並且也缺少主索引鍵來識別每一組關聯的資料。

|  Name          | Date |Volume|
|:-------------:| -----:| -----:|
| Mark | 20160101| 10 , -20 |
| Jiro      | 20160102 | -20 , 30 |
| Ian      |   20160103 | 34 , -10 |

如果要符合第一正規形式大概要長的像降。

|TradeId|  Name          | Date |Volume|
|:-------------:|:-------------:| :-----:| :-----:|
|1| Mark | 20160101| 10  |
|2| Mark      | 20160101 | -20  |
|3| Jiro      |   20160102 | -20 |
|4| Jiro | 20160102 | 30 |
|5| Ian      | 20160103 | 34 |
|6| Ian      |   20160103 | -10 |


### 第二正規形式
以下條列為第二正規形式的規則，重點在於『去除相依性』。

* 為可套用於多筆記錄的多組值建立不同的資料表。
* 使用外部索引鍵，讓這些資料表產生關聯。

我們沿用第一正規形式的資料，但給他多加一筆欄位`age`，是的這個範例是違反第二正規形式，其中它的`Age`相依於`Name`，但是它不應該依賴資料表主索引鍵(`TradeId`)之外的索引鍵。

|TradeId(主鍵)|  Name(主鍵)          |Age| Date |Volume|
|:-------------:|:-------------:| :-----:|:-----:| :-----:|
|1| Mark |18| 20160101| 10  |
|2| Mark      |18| 20160101 | -20  |
|3| Jiro      |35|   20160102 | -20 |
|4| Jiro |35| 20160102 | 30 |
|5| Ian      |25| 20160103 | 34 |
|6| Ian      | 25|  20160103 | -10 |

所以上表如果要修改為符合第二正規形式的結構要拆分為兩個表`交易訂單資料`與`交易者資料`。

`交易訂單資料`

|TradeId(主鍵)|  UserId          | Date |Volume|
|:-------------:|:-------------:| :-----:|:-----:| :-----:|
|1| 001 | 20160101| 10  |
|2| 001      | 20160101 | -20  |
|3| 002      |   20160102 | -20 |
|4| 002 | 20160102 | 30 |
|5| 003      | 20160103 | 34 |
|6| 003      |  20160103 | -10 |

`交易者資料`

|  UserId          | Name |Age|
|:-------------:| :-----:| :-----:|
| 001 | Mark| 18 |
| 002      | Jiro | 35 |
| 003      |   Ian | 25 |

### 第三正規形式
* 刪除不依賴索引鍵的欄位。

下表為交易訂單的資料，但我們在這表上增加兩個欄位`Price`和`Total`，但實際上這已經違反了第三正規形式，`非主鍵欄位之間不能有依賴關係`，`Total`依賴了`Volume`與`Price`，所以如果要符合，就要把`Total`給砍囉。

`交易訂單資料`

|TradeId(主鍵)|  UserId          | Date |Volume|Price|Total|
|:-------------:|:-------------:| :-----:|:-----:| :-----:|:-----:| :-----:|
|1| 001 | 20160101| 10  | 20 | 200 |
|2| 001      | 20160101 | -20  |20 | -400|
|3| 002      |   20160102 | -20 | 30 | -600|
|4| 002 | 20160102 | 30 |10|300|
|5| 003      | 20160103 | 34 |10|340|
|6| 003      |  20160103 | -10 |20|-200|

##  MongoDB 中的正規化與反正規化

###  Mongodb 正規化範例
上面的章節中我們已經大致上了解正規化是什麼意思，這邊再簡單的用`mongodb`的結構來複習一次，正規化就是將資料分散到多個不同的`collection`，不同`collection`之間可以相互引用資料，所以找資料時，只要`join`(在關聯式時)就好，但`mongodb`沒`join`……，只能多次搜尋了。

下面運用簡單的交易資訊來建立`mongodb`正規化的結構。

```
交易資料的 collection

{ "tradeId" : 1 , "userId" : "001", "date" : 20160101 , "volume" : 10}
{ "tradeId" : 2 , "userId" : "002", "date" : 20160102 , "volume" : 20}
{ "tradeId" : 3 , "userId" : "003", "date" : 20160103 , "volume" : 30}
```
然後另一個`collection`為交易者的資料。

```
// 交易者的 collection

{ "userId" : "001" , "name" : "mark" ,"age" : 20}
{ "userId" : "002" , "name" : "jiro" ,"age" : 35}
{ "userId" : "003" , "name" : "ian" ,"age" : 25}
```
但這種有什麼缺點呢 ? 上面有提到搜尋時，需要搜尋兩次才能找到全部的資料，假設我們要尋找交易單號為`1`的交易者資訊，就需要搜尋兩次才能找到，如下。

```
db.trades.find({ "tradeId" : 1 },function(err,trade){
	var userId = trade.userId;
	
	db.users.find({"userId" : userId},function(err,user){
		XXXXXXX
	});
})
```
那優點是啥 ? 就是更新時很方便，如果`mark`這位交易者的年齡打錯了，要修改只要針對`user collection`的進行修改。

```
db.user.udpate({"name" : "mark"},{"$set" : { "age" : 18} })

```

### MongoDB 反正規化範例

而反正規化則相反，將每個`document`中所需要資料都建立成子文件形式，如果有資料要更新，那麼所有的`document`都需要進行更新，但在搜尋時，只需要尋找一次，就可以得到所有數據，理論上來說應該是比正規化速度還快。

以下將以交易資料來說明`mongodb`的反正規化資料結構。

```
交易資料的 collection

{ 
  "tradeId" : 1 , "date" : 20160101 , "volume" : 10 ,
  "user" : {
  	"name" : "mark",
  	"age" : 20
  }
}
{ 
  "tradeId" : 2 , "date" : 20160102 , "volume" : 20 ,
  "user" : {
	"name" : "jiro",
	"age" : 35  
  }
}
{ 
  "tradeId" : 3 , "date" : 20160103 , "volume" : 30 , 
  "user" : {
  	"name" : "Ian",
  	"age" : 25
  }
}
```
這樣要尋找交易單為`1`的交易人資訊非常的簡單，如下。

```
db.trades.find({ "tradeId" : 1})

```
但如果要更新交易人的資訊，那就比較麻煩了，因為它就要尋找所有交易單裡的交易人進行修改囉，這樣當然比`正規化`的慢多囉。

### 何時使用正規化 ? 何時使用反正規化

我們根據`MongoDB`權威指南所列出的原則來看看。

|  反正規化          | 正規化 |
|:-------------:| :-----:|
| 子文件較小 | 子文件較大|
| 資料不太常改變      | 資料很常改變 |
| 最終資料一致即可      |   中間階段的資料必須一致 |
| `document`資料小幅增加 | `document`資料大幅增加|
| 資料通常需要執行二次搜尋才能獲得      | 資料通常不包含在結果中 |
| 要求快速搜尋      |   要求快速寫入 |

簡單的說明一下上述的原則

* 子文檔大小 : 合理，子文件太大時有可能會超過`mongodb`的限制，每個`document`大小16MB這項限制，拆開來存成另一個`document`，就可以避免這問題。
* 資料改變頻率 : 就如同上面的例子，反正規化更新比較麻煩也廢時，正規更新較快。
* 資料一致性 : 也就是說反正規化可能會發生某筆交易單的交易人`mark`的年紀和另一筆交易單的`mark`年紀不一致的狀況，因為還沒全部更新完，就有人搜尋了。
* 資料增加幅度 : 似乎與第一項子文檔大小相似的原理。

## ~ 結語 ~
本篇文章簡單的說明完正規化與反正規化的問題，基本上要選擇用那種來設計你的`MongoDB`，還是要看看你的需求才能決定，如果真的難以決定，只要記好一個原則，依使用率最高的功能來進行設計，嗯……個人的想法。

`P.S` 又快`g`的感覺 ~ 各位`+u^17` ~  

## ~ 參考資料 ~

* [https://www.mongodb.com/blog/post/6-rules-of-thumb-for-mongodb-schema-design-part-1](https://www.mongodb.com/blog/post/6-rules-of-thumb-for-mongodb-schema-design-part-1)
* [https://blog.toright.com/posts/4483/mongodb-schema-%E8%A8%AD%E8%A8%88%E6%8C%87%E5%8D%97.html](https://blog.toright.com/posts/4483/mongodb-schema-%E8%A8%AD%E8%A8%88%E6%8C%87%E5%8D%97.html)
* [https://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E5%BA%93%E8%A7%84%E8%8C%83%E5%8C%96](https://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E5%BA%93%E8%A7%84%E8%8C%83%E5%8C%96)
