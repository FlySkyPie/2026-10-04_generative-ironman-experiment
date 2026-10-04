# Day 2：什麼是 CityJSON？3D 城市模型的新世代格式

> 如果 JSON 是後端工程師的共通語言，那麼 CityJSON 就是把這份共通語言延伸到三度空間的城市模型之中。

---

## 從 3D 城市模型與數位雙生的大趨勢談起

近年來，**數位雙生（Digital Twin）** 的概念從製造業延伸到了整個城市的尺度。從新加坡的 Virtual Singapore 到日本的 Plateau 計畫，世界各國紛紛投入建設城市級的 3D 數位模型[^virtual-singapore][^plateau]。

這些 3D 城市模型的應用範圍遠遠不只是「好看的 3D 地圖」：

- **能源分析**：估算建築屋頂的太陽能潛力、評估能源需求[^cityjson-apps]
- **防災模擬**：洪水淹沒範圍預測、地震損害評估、疏散路線規劃
- **都市規劃**：天際線可視性分析、噪音傳播預測、風場模擬
- **基礎設施**：無線基地台選址、無人機航線規劃
- **3D 地籍**：透過立體空間進行不動產登記與管理

然而，這些應用長期以來面臨一個共同瓶頸：**資料格式的入門門檻太高**。

## 傳統格式的困境：CityGML 的 XML 包袱

3D 城市模型的標準資料模型是 **CityGML**，由 OGC（Open Geospatial Consortium）制訂，自 2008 年即為國際標準[^citygml-ogc]。CityGML 定義了豐富的語意模型——建築物有屋頂、牆面、窗戶，道路有路面和標線，橋樑有橋面和橋墩——這些資訊對分析至關重要。

但 CityGML 的編碼採用的是 **GML（Geography Markup Language）**，一種基於 XML 的語法。這帶來了幾個實際問題：

1. **檔案肥大**：同一筆資料集，CityGML 的 XML 檔案比 CityJSON 平均大 7 倍[^cityjson-filesize]
2. **解析緩慢**：XML 的命名空間（namespace）解析、DOM 樹構建都需要大量計算資源
3. **開發不友善**：想處理 CityGML 的工程師，得先學習 XML Schema、GML 語法、以及 OGC 的一系列規範，學習曲線陡峭
4. **Web 整合困難**：相較於 JSON 在瀏覽器與 JavaScript 中的原生支援，XML 的處理需要額外套件

## CityJSON 的誕生與定位

CityJSON 正是在這樣的背景下誕生的。它由 **荷蘭台夫特理工大學（TU Delft）的 3D 地理資訊研究群** 主導開發，主要貢獻者包括 Hugo Ledoux、Balázs Dukai、Ken Arroyo Ohori、Kavisha Kumar 等人[^cityjson-about]。自 2017 年釋出首個測試版以來，歷經多次迭代，最終於 **2023 年 9 月成為 OGC 正式國際標準**（文件編號 20-072r5）[^cityjson-ogc-v2][^ogc-cityjson]。

CityJSON 的核心定位很明確：**JSON-based encoding for 3D city models**。它不是發明一個新的資料模型，而是為 CityGML 已經定義好的模型提供一個更現代、更輕量的編碼方式。

> 一句話總結：**CityGML 的資料模型 + JSON 的輕量**

這個定位意味著：
- **資料模型保持一致**：你在 CityJSON 中看到的 Building、Road、Bridge 等物件，與 CityGML 的定義完全相容
- **雙向轉換可行**：兩者之間可以無損轉換，官方工具 `citygml-tools` 即可做到一鍵轉換[^citygml-tools]
- **語意不流失**：所有屬性、語意表面（Semantic Surface）、層級細節（LoD）等資訊都保留

## CityJSON 的八大核心特點

### 1. JSON 編碼

CityJSON 使用純 JSON。這意味著任何一種程式語言都有成熟的解析器，開發者無需學習新工具即可開始處理 3D 城市資料。

### 2. 檔案輕量化

平均壓縮比約 **7 倍**，以蘇黎世資料集為例：CityGML 需 2,100 MB，CityJSON 僅 293 MB[^cityjson-filesize]。這直接影響了下載時間、儲存成本和後續處理效率。

### 3. 完整語意

不只是幾何形狀，CityJSON 記錄了每個物件的類型、屬性、與其他物件的關係。知道某個面是「屋頂表面」還是「牆面表面」，對於太陽能分析或能源計算至關重要。

### 4. 多層級細節（LoD）

支援從 LoD0（地區輪廓）到 LoD3（含詳細建築外觀）的多種細節層級，同一物件可以同時擁有多個 LoD 的幾何資料。

### 5. 擴充機制 Extensions

當標準物件不夠用時，Extensions 允許自訂新的城市物件類型、屬性、語意表面。這相當於 CityGML 的 ADE（Application Domain Extension），但設計得更簡單。

### 6. 紋理與材質

支援材質（Material）與紋理（Texture），讓 3D 城市模型可以有逼真的外觀呈現。

### 7. 幾何模板（Geometry Templates）

透過模板與轉換矩陣（transformation matrix），可以大量複製重複的物件（如路燈、樹木），大幅節省空間。

### 8. 串流支援 CityJSONSeq

針對超大型資料集，CityJSONSeq（`.city.jsonl`）格式允許逐筆物件處理，不必一次載入整個檔案到記憶體[^cityjsonseq]。

## CityJSON 不是什麼

理解一個技術的「不是什麼」也很重要：

- **CityJSON 不是 CityGML 的取代品**。它是 CityGML 的另一種編碼方式。兩者可以共存，且有官方工具雙向轉換。
- **CityJSON 不是 BIM 格式**。雖然它支援建築物細節，但 IFC（Industry Foundation Classes）才是建築資訊模型的主流標準。
- **CityJSON 不是遊戲引擎格式**。雖然可以被匯入 Unity 或 Unreal，但它不是為即時渲染設計的。

## 小結

CityJSON 是一個專為開發者設計的 3D 城市模型格式。它保留了 CityGML 豐富的語意模型，但用 JSON 取代了 XML，讓檔案更小、開發更快、Web 整合更順暢。

明天，我們將深入比較 CityJSON 與 CityGML 的具體差異，從檔案大小到處理速度，用數據說話。

---

## 參考資料

[^cityjson-org]: CityJSON. (n.d.). CityJSON: A JSON-based encoding for 3D city models. Retrieved 2026-09-27, from https://www.cityjson.org/
[^cityjson-about]: CityJSON. (n.d.). About CityJSON. Retrieved 2026-09-27, from https://www.cityjson.org/about/
[^cityjson-filesize]: CityJSON. (n.d.). File size: CityJSON vs CityGML-XML: 7X compression. Retrieved 2026-09-27, from https://www.cityjson.org/filesize/
[^cityjson-apps]: CityJSON. (n.d.). Applications of 3D city models. Retrieved 2026-09-27, from https://www.cityjson.org/applications/
[^cityjsonseq]: CityJSON. (n.d.). CityJSON Sequences (CityJSONSeq). Retrieved 2026-09-27, from https://www.cityjson.org/cityjsonseq/
[^virtual-singapore]: Wikipedia. (n.d.). Virtual Singapore. Retrieved 2026-09-27, from https://en.wikipedia.org/wiki/Virtual_Singapore
[^plateau]: Murata. (n.d.). Examples of diverse applications of 3D urban models. Retrieved 2026-09-27, from https://article.murata.com/en-us/article/examples-of-diverse-applications-of-3d-urban-models
[^cityjson-ogc-v2]: 3D Geoinformation Group, TU Delft. (2023, October 23). CityJSON v2.0 now an OGC community standard. Retrieved 2026-09-27, from https://3d.bk.tudelft.nl/news/2023/10/23/cityjsonv2-ogc.html
[^ogc-cityjson]: Open Geospatial Consortium. (2023). OGC CityJSON Community Standard (20-072r5). Retrieved 2026-09-27, from https://docs.ogc.org/cs/20-072r5/20-072r5.html
[^citygml-ogc]: Open Geospatial Consortium. (n.d.). OGC CityGML. Retrieved 2026-09-27, from https://www.ogc.org/standards/citygml/
[^citygml-tools]: citygml4j. (n.d.). citygml-tools: Conversion tools for CityGML. Retrieved 2026-09-27, from https://github.com/citygml4j/citygml-tools