# 透過AzureML訓練客製化模型及部署IoT應用的開發流程：以PCB-AoI個案為例


【撰寫時間：2023/4/30 writen by Markov Chen】

2023 AI EXPO向我們展示大模型的時代已經來臨，Winner-Take-All的趨勢商業結構改變了整個產業的商業價值與定位，許多業者也重新開始思考高階人才對於公司而言是甚麼樣的存在？從工作價值、職責規劃到適任條件，該如何面對新時代的職場生態是每個人都須認清的課題。

資訊科技從過去 **[材料能源]>[電腦]>[通訊技術]>[伺服器]>[巨量數據]** 發展到現在的 **[AI/區塊鏈]** ，每個主題在其所處的時代都會有其結構性的關鍵工作。而經歷了近百年研究的AI也成為了現代所有研發量能的核心，無論是化學、電機、機械、電子或資訊或是其它相關領域，過去在工程上難以分析處理的數學、統計問題都在這項技術的幫助下迎刃而解，藉此衍伸出了更高水準的 **"產品/服務"** 品質，因此業界也開始出現了一種專門為不同主題提供AI解決方案的新工作 -*「資料科學家」*。

近幾個月迅速崛起的**生成式AI**對於產品端的半導體及電子相關產業而言也無疑是一項激勵，在自動化製造、知識管理、客戶服務或人機協同上都增強了對AI的依賴。然而，這些內容對硬體設備的規格與要求也急遽攀升(例如：NVIDIA在一顆V100 GPU的情況下，要訓練出1024 * 1024的[StyleGAN3](https://nvlabs-fi-cdn.nvidia.com/stylegan3/stylegan3-paper.pdf)影像生成模型大約就需要92年的時間)。

考慮到擁有龐大事業體系的企業，資料儲存空間與訓練的計算成本將會是AI導入的關鍵。除了那些本身就擁有關鍵資源的網路巨擘，我們很難想像每間企業都能夠擁有自己獨立的大模型訓練伺服器(就像是要求他們通通都要有google規模等級的機房)。因此，這些處於結構洞的關鍵企業開始紛紛扮演起領頭羊的角色，透過**雲端**向企業們提供這些AI應用的開發資源，諸如資料中心、Foundation模型、GPU運算、線上API、開發環境工具...等，希望能夠藉由充分發揮集中管理、專業化分工的優勢，使各個產業更好地發揮其本職上所具備的商業價值。

對於大部分"非資深"的一般資料科學家，我們該如何適應這個新型態的產業環境？許多強而有力且新穎的模型在這些平台基本上都是開源的，該受到關注的更應該是在於 (一)"透過Domain-Knowledge釐清這些框架如何導入應用的方法論"以及 (二)"站在巨人的肩膀上，善用開發者社群組織所提供的資源"。未來在Edge和Cloud的混合環境下工作將會成為這個領域的新生態，因為這絕對是驅動大模型最有效的開發途徑，而這篇文章將會透過一個PCB-AoI的個案實例展示AI應用的開發流程，使大家能更快速地熟悉所述的工作型態。

## 目錄

- [相關資源介紹](#)
    - [PCB-AoI公開資料集](#)
    - [開發者社群](#)
    - [雲端平台的選擇](#)
    - [AzureML](#)

- [實作流程](#)
    - [選擇硬體與環境](#)
    - [流程實驗](#)
        - 
## 開發環境介紹
### PCB-AoI公開資料集
表面貼焊技術(Surface-mount technology, SMT)對於電子封裝而言是非常重要的製程，各式各樣的晶片、元件會先被擺放在對應的位置，再透過自動化設備將pin腳焊接到電路載板的端口上。而焊接的好壞對於這塊PCB面板的處理效能、穩定性與防水性都是會有非常顯著的影響，例如：焊錫太厚會使電子傳輸速度變慢而導致Lag；焊錫太少則可能會產生接觸不良或是滲入雜質而損毀元件。因此在製程的中/後段便會透過AoI儀器檢查**出現瑕疵的區域(如下圖所示)**，以此使封裝設備能自動地識別出問題的所在，以此對其進行維護和補救。

![](https://i.imgur.com/fXKB5cH.jpg)

雖然這類問題有許多更高效的數學解法，並且也同時具有極高的精度，但是考慮到生成式AI的擴展性，神經網路在AoI領域的研究仍如火如荼的進行著。因此，我們將以**未來導入AI於這類檢測產品的情境**作為背景，假設PCB-AoI公開資料集的資料就是我們在生產線上實際蒐集到的數據，並實際對這項專案所要求的功能實施研發工作。


### 開發者社群
通過Survey相關的期刊文獻，我們可以發現已經有許多學者為這類影像辨識技術提出新穎的模型架構，例如：You only look once(YoLo)、Vision Transformer(ViT)，而這些著名的模型通常在官方Github或Paper with code上都會有相關的釋例，我們只需要針對它們的輸入/輸出進行微調就可以直接開始建模。

* **獲取神經網路的設計圖**
而這些資源大部分都是採用`Pytorch`、`Tensorflow`的框架來進行建構與訓練，定義網路的元件與連結方式，並透過客製化的資料集計算損失&執行反向傳播。雖然這兩種框架都算是主流，但是Tensorflow在不同版本之間的函數工具間還是經常會出現相容性的問題(儘管官方提供的教學資源很豐富)；而Pytorch不僅在相容性上的表現較好，並且在運算速度方面更勝一籌，因此業界也較常把`Pytorch`當作AI開發的首選框架。

* **將訓練好的模型封裝成ONNX格式**
而這些框架都有各自不同的存取方式，例如Pytorch的`.pth`、Tensorflow的`.pb`或`.h5`，它們不僅完整的保存了網路的結構，也包含了許多個別框架底下的相關資訊。但是在導入實務應用時，我們其實只會需要正向傳播的功能，並且也同時需要給其他開發者提供統一的呼叫介面，因此訓練好的模型通常會封裝成**ONNX**的形式，只儲存與正向傳播有關的內容。

* **將ONNX部署於邊緣智慧裝置**
邊緣運算裝置需要搭載`ONNX runtime`、`TensorRT`或`OpenVINO`引擎才能對ONNX進行推論，而這個引擎所處理的內容與CPU/GPU運算也相對單純，所以會比直接呼叫訓練框架提供的格式更有效率，也更能夠滿足邊緣設備的運算資源。

### 雲端平台的選擇

有時我們所設計的方法論中，可能也包含一些Funodation模型的應用，直接對輸入樣本中的通則性知識進行擷取/嵌入，幫助我們減少了上億級別的訓練開銷。但是針對個別企業的商業應用，利用內部資料集做**fine-tuning**的建模工作同樣也是不小的負擔，有時候也會包含多個深度模型的交互運用。

而在電腦資源有限的情況下，就可以考慮使用以下幾個雲端平台來建模，包括：Azure的Machine Learning Studio、AWS的SageMaker、Google的Colab，它們皆是透過jupyter介面直接連線到雲端供應商提供的私有虛擬機，如此一來不僅能夠很方便地切換不同類型的主機設備與環境，還可以直接使用在市面上較難取得的**GPU**來做AI運算。以下便列出了幾個大型的商用雲端平台，並比較了他們所支援的GPU種類：
![](https://i.imgur.com/CJoI8uZ.png)
![](https://i.imgur.com/xJ3jZdA.png)
*[擷取自NVIDIA官方網站]*

### AzureML 
而在各大平台的使用體驗上，個人是偏好使用Azure的Machine Learning Studio。當然，有時候日常也會搭配其他平台來使用，只是我認為Azure所提供的介面對於機器學習的「整個開發生命週期」而言是最直觀的。就以其管理頁面來說，從設備與執行環境、資料儲存與管理、模型版本控制、到訓練過程的監測都可以在同個畫面完成，這在團隊級的開發中無疑能帶來更高的溝通效率，也可以減少很多開發時程與軟/硬體配置更換的問題，所以這篇文章的實作會以AzureML作為主要的示範對象。

![](https://i.imgur.com/hEAFTb5.png)

[Machine Learning Studio]>[<專案名稱> Workspace]>[Launch Studio]

## 實作流程
### 選擇硬體與環境
在開始實作一個專案前，我們首先要考慮的就是硬體支不支援我們所設計的實驗方案(例如：資料量有多少？、模型訓練需要使用PGU嗎？、訓練時程要花費多久？、正/反向傳播所會占用的記憶體空間有大？...等)。藉此我們可以在上方Studio主畫面中的`Compute`選擇&創建不同規格機型，當我們想更換一部設備進行實驗時只需要簡單做個設定即可，而不需要把實驗資料進行大規模的搬遷。
![](https://i.imgur.com/85hsOgr.png)<br>
另外，這些電腦設備上的執行環境同樣也可以在`Environments`中做管理，只需要設定一次就可以在不同的設備上使用，當發現環境配置有錯時也只要修改Dockerfile的script再一鍵rebuild即可，省去許多過去在實驗室時每換一部設備都要重裝環境的麻煩。
![](https://i.imgur.com/OKu0E5Q.png)<br>
再來是`Data`的資料管理，有過實務開發經驗的人肯定對資料管理很有感，因為客製化的模型難就難在**問題的定義與資料範圍** ，這考驗的不僅僅是資料科學家的Domain-Knowledge，有時候也只有在實驗完成後才能得知資料的好壞，所以在更新ETL程序後的資料可以在workspace儲存空間中進行分門別類，也可以同時在雲端進行版本控管，避免每個人手中雖然同樣都有叫train_data的資料，但內容物完全不一致的問題。

![](https://i.imgur.com/tjLWhHw.png)

### 實驗流程
