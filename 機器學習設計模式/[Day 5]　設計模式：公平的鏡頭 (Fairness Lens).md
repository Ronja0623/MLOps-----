# 摘要
　　公平的鏡頭設計模式強調利用預處理和後續資料處理技術來確保模型預測的公平性。偏差可能來自於資料分布、表示方式或演算法選擇。解決方案包括在訓練前和訓練後對資料和模型進行分析，如使用 What-If Tool、Captum 和 AIF360 等工具。
# 公平的鏡頭
　　公平是一個非常難被明確定義的形容詞，或許我們會說公平就是沒有偏見，但偏見是什麼？應該由誰來定義？由於公平並非放諸四海皆準，因此目前也沒有讓模型公平的統一解決方案。
　　Fairness Lens 設計模式的核心在於利用**預處理**和**後續資料處理**技術來確保模型的預測是否公平，主要評估的方式是判斷資料與預測的不平衡是否自然。這裡對於「自然」的定義是指是否能夠反映欲應用的現實世界，還是因為資料蒐集的手段、資料蒐集的場域、模型採用的演算法等原因而產生了我們所觀察到的不平衡。今天將著重處理不自然的偏差，而自然、且不會對不同族群造成不利影響的偏差會在明天接續討論。
## 偏差
### 資料集
#### 問題偏差 (problematic bias)
　　描述因為訓練資料分布存在偏差，進而影響模型的狀態。通常在以人為中心的資料集中，在涉及身分的特徵上，這種類型的偏差會特別明顯。
#### 報告偏差 (reporting bias)
　　描述資料表示某些族群的方式存在偏差，儘管資料集看似有平衡地表示關於身分的特徵。舉例來說，如果某道料理在網路上的評價上較為負面，後面即使獲得了正面的評價，也可能受大量負面評論的資料影響，而也一起被歸類為負評。但這並非是因為資料蒐集的偏差造成的，因為資料集的分布符合現實，只是訓練結果因此沒有正確表現出現實的情況。
#### 隱性偏差 (implicit bias)／代理偏差 (proxy bias)
　　前面兩個都是可以藉由移除資料集中的偏差解決，然而還存在一種問題，是因為模型在學習過程中無意地捕捉到與目標變數相關但並非直接相關的特徵，這些特徵可能間接代表了敏感特徵，如性別、種族或社會經濟地位。這種偏差較難察覺，因為它不是直接反映在資料的顯性特徵上，而是通過間接方式影響模型的預測。
　　舉例來說，在設計一個求職推薦系統時，如果歷史資料中存在職業與性別的偏見（例如，工程師多為男性，護理師多為女性），那麼模型可能會無意中學習到這種性別偏見，即使我們已經將性別這個特徵移除。
#### 實驗者偏差 (experimenter bias)
　　這種偏差描述的是因為資料的標註方式而引入的偏差，通常是因為標註者的主觀意見、期望或習慣影響了標註結果。舉例來說，在做關於仇恨言論的標註時，因為每個人對於仇恨言論的定義不同，就可能因此引入不同的標註風格。
### 演算法
　　目標函數、演算法等也可能引入偏差。舉例來說，我們採用了不合適的加權交叉熵 (weighted cross-entropy) 作為損失函數來處理類別不平衡的問題，並對較少見的類別賦予更高的權重，但這樣的做法可能會導致模型過度關注某些類別，忽略其他類別的重要性，從而引入新的偏差。此外，不同的演算法在處理特定問題時有各自的偏好和局限性，像決策樹演算法可能對於高度非線性的資料表現不佳，而神經網絡可能容易過度擬合。
## 解決方案

>在訓練模型之前就找出具有有害偏差的資料區域，並通過公平的鏡頭來評估訓練出的模型。

　　兩件要做的事情：在訓練之前仔細觀察**哪些**族群會被模型影響，以及那些群體會**怎麼**被模型影響。
　　書中使用 [What-If Tool](https://pair-code.github.io/what-if-tool/) 來展示兩種類型的分析技術，並在後面的小節介紹了  [Fairness Indicators](https://www.tensorflow.org/tfx/guide/fairness_indicators)，其中 Fairness Indicators 和 tensorflow 進行了相當良好的整合，其他能達成類似功能的工具還有 [AIF360 (AI Fairness 360)](https://aif360.res.ibm.com/)。這種涉及模型的解釋技術稱為可解釋的 AI (Explainable AI, XAI)，進一步的 XAI 技術學習地圖可以參考[這篇文章](https://ithelp.ithome.com.tw/m/articles/10318532)。
### 問題定義
　　進一步分析前面提到的偏差問題，總結來說，我們可以將它分成：
- 資料偏差
	- 資料分布偏差：問題偏差
	- 資料表示偏差：報告偏差、隱性偏差／代理偏差、實驗者偏差
- 演算法偏差
### 方法一：將資料以標籤類別分割
- 資料分布偏差：關注是否有哪些類別、哪些特徵的資料不足。
- 資料表示偏差：關注是否存在某個特徵被歸類於特定類別的機率較高，並且我們能夠知道在我們的模型關注的「世界」與這種模式並不相符。例如前面提到的案例，女性應該被推薦去當護理師，男性應該被推薦成為工程師，這便與我們對世界的認知並不相符。不過這種透過價值觀來評斷資料表示是否存在偏差的手段也將引入新的偏差，但這點在此先暫時不提。
### 方法二：將資料以特徵分割，觀察各特徵的平衡

### 最後，但在訓練之前
　　如果我們已經竭盡所能消除了資料收集偏差，卻還是發現資料存在不平衡的問題，就可以思考此時適合如何平衡資料集，這類自然的偏差我們將在明天繼續討論。
### 訓練之後
　　儘管已經對資料進行了嚴格的審視，但偏差仍可能進入模型當中，主要的原因包含前面提到的演算法所造成的偏差，以及我們沒有發現的資料偏差，因此我們應該謹慎地評估模型，也就是對前面提到的「某些群體會**怎麼**被模型影響」這個問題進一步分析。我們要評估的是這種影響是否是有害的，即這種影響是否反映了真實情況，以及這種影響是否符合我們的期待（這裡的意思可以理解為是否符合我們在開發專案前定義的目標）。
　　這時採用的分析技術稱為後模型分析 (Post-Model Analysis)，或者事後分析 (Post hoc Explanations)。除了可以使用書中提到的 [What-If Tool](https://pair-code.github.io/what-if-tool/) ，還可以參考 [Captum](https://captum.ai/)和前面提過的 [AIF360 (AI Fairness 360)](https://aif360.res.ibm.com/)，其中 Captum 和 pytorch 進行了相當良好的整合，對於自定義的模型解釋來說相當有幫助。
## 其他方案：直接定義規則
- 資料蒐集時：自行定義標籤值、特徵值，或直接摘除 (ablation) 、修改部分資訊。
- 訓練模型後：在產生預測結果後，直接定義不允許和允許出現在使用者面前的內容，在結果抵達使用者眼前之前先進行篩選。常用於生成式模型，畢竟我們難以控制模型產生的內容。
# 補充
　　今天的文章沒有談到太多 What-If Tool 的實作，進一步的資訊可以參考[這篇文章](https://pair-code.github.io/what-if-tool/ai-fairness.html)，或者參考[作者提供的程式碼](https://github.com/GoogleCloudPlatform/ml-design-patterns/blob/master/07_responsible_ai/fairness.ipynb)。此外，也來不及提到太多如何「自動」評估公平性的問題，而是手動處理、人工判別，關於將公平性整合進模型的進一步討論會在下週進行。
　　其它處理偏差的技術還可以參考 Model Cards，相關資訊可以參考[這篇文章](https://arxiv.org/abs/1810.03993)，它的做法是提供一個框架回報模型的能力與限制，嘗試提供模型什麼時候適合提供某些細節的資訊，來提高模型的透明度。這項技術可以和前面提到的**解決方案：直接定義規則**進行整合，相關工具可以參考 Tensorflow 的 [Model Card Tookit](https://www.tensorflow.org/responsible_ai/model_card_toolkit/guide?hl=zh-tw) 。
## 公平性與可解釋性
　　儘管我們今天討論的問題是公平性，也介紹了許多可解釋工具，似乎正將二者混為一談，不過需要注意二者的概念仍然存在差異。解釋技術是處理公平性問題的一種方法，而且它也適用於處理更多其他問題；相應地，也存在許多其他方法可以處理公平性的問題，而非只有解釋技術一種。
# 參考資料
- Machine Learning Design Patterns: Solutions to Common Challenges in Data Preparation, Model Building, and MLOps CH7 Design Pattern 30: Fairness Lens
- [[Day 2] 從黑盒到透明化：XAI技術的發展之路](https://ithelp.ithome.com.tw/m/articles/10318532)