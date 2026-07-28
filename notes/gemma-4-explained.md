這篇文章在講什麼

這篇《A Visual Guide to Gemma 4》不是操作教學，也不是單純比 benchmark。它的重點是：用圖解方式，把 Gemma 4 為什麼能兼顧多模態、長前後文、效率與裝置端部署 這幾件事拆開講清楚。文章先講共通架構，再分別解釋 31B、26B A4B、E2B、E4B 這四種模型。

新手先記住這 6 件事

1. Gemma 4 有四個型號：E2B、E4B、31B、26B A4B。全部都支援文字與圖片輸入；小模型 E2B、E4B 另外還支援音訊。官方模型卡另外補充：E2B/E4B 的前後文長度是 128K，26B A4B/31B 是 256K。 
2. 它不是每一層都看完整段文字。Gemma 4 會把「只看附近內容的 local attention」和「看整體內容的 global attention」交錯使用，目的是省計算量，同時保留全局理解能力；而且 Gemma 4 特別把最後一層固定成 global attention。 
3. 它很在意記憶體效率。文章花很多篇幅解釋 Gemma 4 如何讓 global attention 更省，包括 Grouped Query Attention、K=V，以及 p-RoPE。白話說：能共用的資料就共用，能少存的就少存，能避免位置資訊干擾語意的就避免。 
4. 它真的不是只會看正方形圖片。Gemma 4 的 vision encoder 支援可變長寬比與可變解析度，會把圖片切成 patch，再用 2D RoPE 表示二維位置，最後再用 token budget 控制送進語言模型的影像資訊量。這是它多模態能力的重要基礎。 
5. 26B A4B 的關鍵是 MoE。它總共有 26B 參數，但推論時只會啟動約 4B 的 active parameters。文章的核心意思是：模型本體很大，但每次不需要全部一起上工，所以速度會比你直覺以為的快。 
6. E2B / E4B 的關鍵是 PLE（Per-Layer Embeddings）。它不是把所有能力都塞進主幹模型，而是把一部分資訊做成每層可查表的嵌入向量，讓模型在較小 RAM/VRAM 條件下也能維持不錯能力，所以特別適合手機、筆電、本機端部署。 

白話版整理

1. Dense 跟 MoE 到底差在哪

Dense model 可以想成：每次回答問題，整個主力團隊都一起上。

MoE（Mixture of Experts） 可以想成：公司裡有很多專家，但每次只叫幾位最適合的上場，其他人先坐著領乾薪，不是，先待命。Gemma 4 的 26B A4B 就是這種設計：總共有 128 個 experts，每次啟動 8 個，另外再加 1 個永遠在線的 shared expert。

2. 為什麼 attention 要分 local 和 global

因為 global attention 很貴。

如果每個 token 都要看完整串輸入，計算量和記憶體都會膨脹。Gemma 4 的做法是：大部分時間先看附近，偶爾再看全局，像開車時平常看前車和後照鏡，進交流道才抬頭看整張路網。這樣能兼顧速度與理解力。

3. K=V、GQA、p-RoPE 是在幹嘛

這三個名詞看起來像研究員故意不讓人下班，但本質都在做同一件事：讓長前後文不要太吃資源，也不要太容易失真。

GQA 是讓多個 query 共用較少的 key/value；K=V 是在 global attention 裡讓 key 和 value 合併，減少 cache；p-RoPE 則是只對部分維度加位置資訊，降低長序列時位置旋轉帶來的語意干擾。

4. 圖片是怎麼餵給 Gemma 4 的

Gemma 4 不是直接「看圖片」，而是先把圖片切成很多小塊 patch，再把每塊轉成嵌入向量。不同於老式做法把圖硬壓成正方形，Gemma 4 可以保留不同長寬比，並用 2D RoPE 分別表示寬度和高度位置。解析度則靠 soft token budget 控制，常見 budget 有 70、140、280、560、1120。budget 越高，保留的畫面細節越多，但運算也越重。

5. 音訊為什麼只有小模型有

文章提到 E2B、E4B 額外有 audio encoder，能處理語音辨識和語音翻譯。流程大致是：原始音訊先轉成 mel spectrogram，再切成片段、下採樣、經過 conformer encoder，最後投影到語言模型能理解的嵌入空間。這代表小模型不只省資源，還更適合做裝置端語音任務。

四個模型怎麼看

E2B / E4B

定位：輕量、裝置端、本機跑得動。

特色：Dense 架構、支援文字/圖片/音訊、用了 PLE。官方資料顯示它們的基礎推論記憶體需求大約是：E2B 為 9.6GB（BF16）或 3.2GB（Q4_0），E4B 為 15GB（BF16）或 5GB（Q4_0）。

26B A4B

定位：效能與速度平衡型。

特色：MoE，總參數 26B，但每次只動用約 4B active parameters；支援文字/圖片，前後文長度 256K。適合想要更強能力，但又不想直接上超大 dense 模型的人。

31B

定位：較傳統的大型 dense 主力模型。

特色：沒有 PLE，也不是 MoE，比較像「正統大模型」路線；支援文字/圖片，前後文長度 256K。若你想理解 Gemma 4 的核心架構，31B 可以視為最直觀的代表。

這篇文章真正想傳達的重點

不是「Gemma 4 很強」這種空話。

而是：

- 同一個模型家族，可以用不同架構去適配不同硬體場景。 
- 多模態不是把圖片或音訊硬接進去就算了，前面要有專門的 encoder 和投影機制。 
- 長前後文能力的代價很高，所以設計重點不是只把前後文拉長，而是如何不把記憶體和速度一起炸掉。 
- 小模型不代表只是縮水版；E2B/E4B 明顯是為裝置端做了特別工程化設計。 

給新手的最終結論

把 Gemma 4 想成一個家族，不是一顆模型。

這個家族用三條路線解決不同問題：

- E2B / E4B：我想在手機、筆電、本機端跑，還要支援語音。 
- 26B A4B：我想要更強能力，但不要每次都把整台機器燒成暖爐。 
- 31B：我就是要一顆比較直球的大型 dense 模型。 

而這篇文章最有價值的地方，是把背後設計講成人能讀的樣子。這在 AI 圈算稀有物種。

[https://newsletter.maartengrootendorst.com/p/a-visual-guide-to-gemma-4?fbclid=https://newsletter.maartengrootendorst.com/p/a-visual-guide-to-gemma-4?fbclid=IwdGRleAQ_EStleHRuA2FlbQIxMQBzcnRjBmFwcF9pZAo2NjI4NTY4Mzc5AAEeDF_V2ImEdqp0dQ4nphglw6A4VRM42MkAWsWMktFUtP4j1S1DXMdN0rYI1cI_aem_EXCKxuo61Hy3RW4OTvM03g](https://newsletter.maartengrootendorst.com/p/a-visual-guide-to-gemma-4?fbclid=https://newsletter.maartengrootendorst.com/p/a-visual-guide-to-gemma-4?fbclid=IwdGRleAQ_EStleHRuA2FlbQIxMQBzcnRjBmFwcF9pZAo2NjI4NTY4Mzc5AAEeDF_V2ImEdqp0dQ4nphglw6A4VRM42MkAWsWMktFUtP4j1S1DXMdN0rYI1cI_aem_EXCKxuo61Hy3RW4OTvM03g)

[#ai](#ai) [#ai/llm](#ai/llm) [#ai/llm/gemma](#ai/llm/gemma) [#ai/moe](#ai/moe) [#ai/multimodal](#ai/multimodal) [#ai/long-context](#ai/long-context) [#ai/on-device](#ai/on-device) [#技術筆記](#技術筆記) [#新手向](#新手向)