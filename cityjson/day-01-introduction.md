# Day 1：前言、動機與 30 天路線圖

> 系列名稱：**CityJSON 實戰 30 天 — 從入門到數位雙生的 JSON 3D 城市模型**

---

## 前言

3D 城市模型與數位雙生（Digital Twin）技術近年發展迅速，從都市規劃、能源分析、防災模擬到自駕車測試，應用場景不斷擴大。然而，這個領域的入門門檻一直存在：傳統的 CityGML 格式以 XML 編碼，檔案龐大、解析耗時、學習曲線陡峭。

CityJSON 的出現改變了這個局面。它以 JSON 編碼，是 OGC（Open Geospatial Consortium）國際標準格式[^cityjson-org]，讓熟悉 JSON 的開發者能快速上手 3D 城市資料處理。

本系列用 30 天的時間，帶領對 3D 城市模型有興趣的開發者，從零開始掌握 CityJSON。

## 為什麼選擇 CityJSON？

CityJSON 繼承了 CityGML 完整的資料模型（Data Model），卻使用了 JSON——這個全世界開發者都熟悉的格式。這意味著：

- **檔案更小**：比 CityGML 小約 6–7 倍，以蘇黎世（Zürich）資料集為例，CityGML 為 2,100 MB，CityJSON 僅 293 MB[^cityjson-filesize]
- **解析更快**：快約 2 個數量級（約 100 倍），因為 `json.loads()` 遠比 XML namespace 解析來得快[^spatialworkflow]
- **開發更友善**：任何會 JSON 的開發者都能立刻上手
- **標準化**：隸屬於 OGC 標準家族，v1.0 於 2021 年成為 OGC 社群標準，v2.0 於 2023 年晉升為正式標準[^cityjson-ogc-v1][^cityjson-ogc-v2]

用一句話總結：**CityGML 的資料模型 + JSON 的輕量**。

## 為什麼選這個主題？

選擇 CityJSON 作為鐵人賽主題，有幾個客觀原因：

**第一，繁體中文的 3D 城市模型教學資源極度稀缺。** 相關文件大多以英文為主，對台灣開發者來說門檻較高。一套完整的繁體中文教學系列，有助於降低入門障礙。

**第二，CityJSON 本質上就是為開發者設計的格式。** 它將複雜的城市模型簡化為開發者友善的 JSON 結構，讓習慣 JSON 生態系的工程師能快速上手。這種設計理念本身就是值得推廣的技術資產。

**第三，技術寫作是檢驗理解深度的有效方式。** 連續 30 天每天深入研究一個主題並整理輸出，有助於系統性地建構知識體系。

## 系列涵蓋範圍

這 30 天的內容涵蓋以下五大領域：

1. **認識 CityJSON** — 為什麼需要用這個格式、格式的基礎結構
2. **幾何核心** — 頂點系統、幾何類型、多層級細節（LoD）
3. **城市物件與語意** — 建築物、道路、橋樑等物件及其語意表面
4. **進階功能** — 材質紋理、Metadata、擴充機制
5. **實戰應用** — 工具鏈使用、串流處理、程式實作

## 目標讀者

這個系列是為以下讀者設計的：

- 對 3D GIS、數位雙生有興趣的開發者
- 會基礎 JSON、想學 3D 城市資料處理的軟體工程師
- 已經用過 CityGML 但想找更輕量替代方案的地理資訊從業者
- 對智慧城市、3D 都市建模感興趣的學生

## 30 天路線圖

```
Phase 1｜認識 CityJSON（Day 1-6）   — 為什麼、是什麼、怎麼開始
Phase 2｜幾何核心（Day 7-12）       — 頂點、變換、幾何類型、LoD
Phase 3｜城市物件與語意（Day 13-18） — Building、Road、Semantics
Phase 4｜進階功能（Day 19-23）      — Appearance、Metadata、Extensions
Phase 5｜實戰應用（Day 24-29）      — 串流、工具鏈、Python 實作
Phase 6｜完賽總結（Day 30）         — 回顧與下一步
```

### Phase 1：認識 CityJSON（Day 2-6）

- Day 2：什麼是 CityJSON？
- Day 3：CityJSON vs CityGML
- Day 4：CityJSON 結構總覽
- Day 5：Hello CityJSON
- Day 6：開發環境與工具鏈

### Phase 2：幾何核心（Day 7-12）

- Day 7：頂點系統與 Transform
- Day 8：基礎幾何類型
- Day 9：實體幾何
- Day 10：邊界陣列深度解析
- Day 11：幾何模板
- Day 12：多層級細節（LoD）

### Phase 3：城市物件與語意（Day 13-18）

- Day 13：城市物件總覽
- Day 14：建築物家族
- Day 15：運輸與基礎設施
- Day 16：植栽、水體、地形
- Day 17：語意表面
- Day 18：Parent/Children 關係

### Phase 4：進階功能（Day 19-23）

- Day 19：外觀材質
- Day 20：UV 座標與紋理貼圖
- Day 21：Metadata 與 CRS
- Day 22：Extensions 擴充機制
- Day 23：寫一個自己的 Extension

### Phase 5：實戰應用（Day 24-29）

- Day 24：CityJSONSeq 串流
- Day 25：大型檔案處理策略
- Day 26：cjio 指令一把罩
- Day 27：從 CityGML 轉換到 CityJSON
- Day 28：全球城市資料集巡禮
- Day 29：用 Python 處理 CityJSON

### Day 30：完賽心得與總結

## 讀者預期收穫

30 天後，你將能夠：

1. 讀懂 CityJSON 檔案的完整結構
2. 使用現有工具操作 CityJSON 資料
3. 理解 3D 城市模型的核心概念
4. 用 Python 自行處理 CityJSON 資料
5. 知道在哪裡找到真實的 CityJSON 資料集

## 寫作風格說明

本系列每篇文章的結構如下：

1. **破題** — 一句話讓你知道本篇重點
2. **理論說明** — 深入淺出的概念講解
3. **程式碼/JSON 範例** — 以真實範例為基礎改編
4. **小結** — 重點整理與下一篇預告

文章中的所有 JSON 範例皆改編自 [CityJSON 官方規格](https://github.com/cityjson/specs)[^cityjson-specs]，確保內容正確且與標準一致。

---

明天開始，正式進入 CityJSON 的世界。

## 參考資料

[^cityjson-org]: CityJSON. (n.d.). CityJSON: A JSON-based encoding for 3D city models. Retrieved 2026-09-27, from https://www.cityjson.org/
[^cityjson-filesize]: CityJSON. (n.d.). File size: CityJSON vs CityGML-XML: 7X compression. Retrieved 2026-09-27, from https://www.cityjson.org/filesize/
[^spatialworkflow]: Spatial Workflow. (2026, September 15). CityJSON and CityGML explained. Retrieved 2026-09-27, from https://www.spatialworkflow.io/cityjson-and-citygml-explained/
[^cityjson-ogc-v1]: CityJSON. (2021, August 16). CityJSON v1.0 is now an OGC standard! Retrieved 2026-09-27, from https://www.cityjson.org/news/2021/08/16/hello-ogc/
[^cityjson-ogc-v2]: 3D Geoinformation Group, TU Delft. (2023, October 23). CityJSON v2.0 now an OGC community standard. Retrieved 2026-09-27, from https://3d.bk.tudelft.nl/news/2023/10/23/cityjsonv2-ogc.html
[^cityjson-specs]: CityJSON. (n.d.). CityJSON specifications (GitHub repository). Retrieved 2026-09-27, from https://github.com/cityjson/specs