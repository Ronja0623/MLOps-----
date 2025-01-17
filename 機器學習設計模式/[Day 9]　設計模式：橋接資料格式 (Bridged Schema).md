# 摘要
　　當資料格式或特徵發生變更時，舊資料可能不再直接適用於新模型。針對這種情況，可以透過橋接方法來過渡，將舊資料中的類別以新資料的比例進行機率補值或靜態補值，例如根據新類別的分佈，對舊資料進行重新編碼。在訓練模型時，驗證集與測試集應只包含新資料，並透過隨機抽樣確定合適的數量，以確保評估結果的穩定性；訓練集則應盡量減少舊資料的使用，並測試加入不同量舊資料後的影響。
# 橋接資料格式
　　在 [[Day 8]　建構 ML 系統的挑戰 — 資料漂移]] 我們提到資料的格式、特徵值的分布等發生變更的時候，因為新資料的累積需要時間，所以我們可以嘗試橋接兩種格式，藉此渡過新資料還沒有累積到足夠的量的時間。
## 情境
　　當我們在訓練一個銷售應用程式時，假設輸入之一是支付類型，過去我們只能使用「現金」和「卡片」，現在多了「金融卡」、「信用卡」、「悠遊卡」。
## 可以簡單地對兩組特徵做聯集就好了嗎？
　　為什麼不建立過去的特徵列表和現在的特徵的列表的聯集，然後再進行編碼呢？這樣應該就不會有問題了吧？
　　問題是新的資料不可能會再出現「卡片」這個特徵，因為輸入已經改變了，卡片的分類變得更細，也就是說新的分類只會有四類：「現金」、「金融卡」、「信用卡」、「悠遊卡」。這麼做的話，之前蒐集的關於「卡片」的資料都是無效的。
## 解決方案
### 一、舊資料只有兩種類型，新資料有四種，這件事可以怎麼處理？
　　我們可以知道原本的卡片會包含「金融卡」、「信用卡」、「悠遊卡」，只是沒有詳細紀錄確切的資訊，因此我們可以嘗試**補入新的值**。
#### 機率補值
　　首先觀察新的訓練資料，知道這三種在資料中的所佔比例，然後採用這種比例對舊資料隨機補入新值。舉例來說，假設`金融卡：信用卡：悠遊卡＝7：2：1`，那麼我們可以隨機抽 0.7 被標註為「卡片」的舊資料，將它更標記為「金融卡」。
　　但是問題是，舊資料並不一定會和新的資料具有相同的比例。舉例來說，假設舊資料蒐集自千禧年前，當時連悠遊卡公司的前身都還沒有被北捷收購，就不要說資料裡會出現以悠遊卡支付的交易資訊了。
#### 靜態方法
　　類別變數通常使用 one-hot 編碼（相關的權衡與折衷我們曾在 [[[Day 1]　邁向 MLOps 之旅]] 的「這系列文章會包含哪些內容？」簡單提及，我們也在 [[[Day 12]　設計模式：被雜湊的特徵 (Hashed Feature)]] 接續討論 one-hot 編碼的問題），如果我們在這個情境中採用 one-hot 編碼來處理相關問題，那麼我們在新資料裡的所有「卡片」資料中看見的 one-hot 編碼平均值將會是 [0, 0.7, 0.2, 0.1] ，第一個 0 是「現金」類別。
　　我們可以將這個 [0, 0.7, 0.2, 0.1] 做為舊資料中「卡片」類別的新編碼。
#### 其他靜態補值
- 特徵是數值
	- 呈常態分布：平均值
	- 呈傾斜、或有大量異常值：中位數
- 特徵是類別
	- 可排序：中位值
	- 不可排序：模數
- 特徵是布林：特徵是 true 的機率
### 機率補值 vs. 靜態方法
　　如果採用機率補值，就會需要多寫一個 function 來為每筆資料賦值；而假如採用靜態方法，一行程式就夠了。
###  二、如何進行資料分割，以建立訓練、驗證和測試集？
#### 1. 驗證集和測試集不可以包含舊資料
　　首先，我們要先知道驗證集和測試集的目的是什麼：驗證集是在訓練途中能夠「偷偷看一下」模型訓練的方向是否正確，而測試集是在模型訓練結束後用來檢視模型在面對「從未看過」的資料時的表現如何。總結來說，在這個設計模式中，這兩個資料集都不應該包含任何一筆舊資料，因為我們都知道我們現在要處理這個模型就是因為舊資料不會再出現了。
#### 2. 驗證集應該包含多少新資料？
　　然後我們要拿現在已經被佈署到生產環境的舊模型出來，然後將現有的新資料取出來做為測試集，每取一個數字做為測試集的數量，就反覆隨機抽取定量資料，計算對舊模型來說，新資料的量要取多少才能讓評估指標趨於一致。
　　舉例來說，如果要在 1000 ~ 10000 筆資料之間評估，在評估 1500 是不是一個合適的數值時，假設我們的指標是準確度，就反覆做二三十次，這二三十次間反覆隨機從新資料中隨機抽取 1500 筆資料，然後計算這二三次的準確度結果產生的標準差。
　　做這件事的目的是為了確認至少驗證集的資料要大於多少，才能讓模型的訓練結果趨於穩定。
#### 3. 訓練集應該包含多少新資料？多少舊資料？
　　訓練集將在扣除驗證集需要的數量之後，使用全部的新資料做為訓練集；如果資料量許可，可以取扣除驗證集與測試集需要的新資料之後剩餘的全部新資料做為訓練集。
　　至於舊資料，我們前面提過，做橋接本就是做為過渡的一種權衡，因此我們應該盡可能減少使用舊資料做為訓練資料。可以嘗試做一系列測試，以推估在加入多少舊資料之後，模型在應對新資料的表現將開始下滑。
#### 補充
　　在評估舊模型和新模型時，驗證資料集非常重要，因為新的資料可能還來不及展現足夠的價值。
# 參考資料
- [悠遊卡 | wikipedia](https://zh.wikipedia.org/zh-tw/%E6%82%A0%E9%81%8A%E5%8D%A1)
- Machine Learning Design Patterns: Solutions to Common Challenges in Data Preparation, Model Building, and MLOps CH1, CH6 Design Pattern:  Bridged Schema