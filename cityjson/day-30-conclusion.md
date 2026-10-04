# Day 30｜完賽心得與總結 — 回顧 30 天的 CityJSON 旅程

## 前言

今天是 30 天系列的最後一天。回望第一天立下的路線圖，從 Phase 1 到 Phase 6，我們走過了一條完整的學習路徑——從「什麼是 CityJSON」一路深入到 Python 程式化處理。這篇文章既是對 30 天旅程的總回顧，也是對 CityJSON 生態系現狀與未來展望的總結。

---

## 30 天旅程回顧

### Phase 1：認識 CityJSON（Day 1-6）

系列以 **3D 城市模型與數位雙生的大趨勢** 開場[^day1]，說明了為什麼需要一個 JSON-based 的 3D 城市模型格式。Day 2 介紹了 CityJSON 的定位——OGC 標準家族中專為開發者設計的輕量格式[^day2]。Day 3 的 CityJSON vs CityGML 比較表揭示了關鍵數字：檔案小 7 倍、解析快 2 個數量級[^day3]。Day 4 拆解了根層級的 7 大成員，Day 5 手寫了第一個 CityJSON 檔案，Day 6 則搭建了完整的工具鏈環境[^day4][^day5][^day6]。

### Phase 2：幾何核心（Day 7-12）

Phase 2 深入了 CityJSON 的幾心核心。Day 7 的頂點系統與 Transform 揭露了來自 TopoJSON 的 quantization 壓縮機制[^day7]。Day 8 到 Day 10 逐步建立起對 8 種幾何類型（MultiPoint、MultiLineString、MultiSurface、Solid、CompositeSolid、MultiSolid、CompositeSurface、GeometryInstance）與邊界陣列巢狀深度的完整理解[^day8][^day9][^day10]。Day 11 的幾何模板是節省空間的殺手級功能，透過 transformation matrix 實例化重複物件（樹、路燈）[^day11]。Day 12 的 LoD 系統則是多層級細節的核心設計，支援字串格式如 `"2.2"`[^day12]。

### Phase 3：城市物件與語意（Day 13-18）

Phase 3 進入了語意層。Day 13 建立了城市物件的樹狀分類（第一層級 vs 第二層級）[^day13]。Day 14 深入了建築物家族的 8 種物件，Day 15 涵蓋了運輸與基礎設施的 Bridge 與 Tunnel 家族[^day14][^day15]。Day 16 覆蓋了植栽、水體、地形等較特殊的物件類型[^day16]。Day 17 的語意表面是精髓所在——它讓 3D 城市模型不只是幾何形狀，而是知道「這是牆還是屋頂」的智慧模型[^day17]。Day 18 的 Parent/Children 關係則說明了從巢狀 XML 到平面 JSON 的資料結構轉變[^day18]。

### Phase 4：進階功能（Day 19-23）

Day 19 與 Day 20 介紹了外觀材質與 UV 貼圖系統，支援 X3D 規格的 Material 與 COLLADA 標準的 Texture[^day19][^day20]。Day 21 的 Metadata 系統相容 ISO 19115 標準，讓資料具備座標、來源與身份[^day21]。Day 22 與 Day 23 則是 CityJSON 最靈活的設計——Extensions 擴充機制，透過 `+` 前綴與 `.ext.json` 檔案，讓開發者可以在標準不足時自行延伸[^day22][^day23]。

### Phase 5：實戰應用（Day 24-29）

Phase 5 從理論走向實戰。Day 24 的 CityJSONSeq 提供了以 JSON Lines 為基礎的串流方案，讓處理百萬建築層級的資料成為可能[^day24]。Day 25 討論了大型檔案處理策略（地理分割、串流、不要超過數百 MB）[^day25]。Day 26 的 cjio 指令指南是 Python CLI 的瑞士刀[^day26]。Day 27 的 CityGML → CityJSON 轉換展示了生態系的互通性[^day27]。Day 28 的世界資料集巡禮走訪了荷蘭 3DBAG（1000 萬建築物）、蘇黎世、紐約、蒙特婁、維也納等城市的開放資料[^day28]。Day 29 用 Python 實作了從解析、統計、屋頂面積計算到 glTF 轉換的完整管線[^day29]。

---

## 知識體系總整理

這 30 天所建立的知識體系可以歸納為以下層次：

| 層次 | 核心能力 | 對應天數 |
|------|----------|----------|
| **理解 What** | 知道 CityJSON 是什麼、為何存在、與 CityGML 的關係 | Day 1-3 |
| **認識結構** | 看懂 7 大根層級成員、最小合法檔案 | Day 4-5 |
| **操作工具** | 使用 cjio、cjval、ninja、azul 等工具 | Day 6 |
| **幾何能力** | 理解頂點系統、8 種幾何類型、LoD、幾何模板 | Day 7-12 |
| **語意能力** | 辨識城市物件類型、語意表面、階層關係 | Day 13-18 |
| **進階功能** | 操作材質紋理、Metadata、Extensions 擴充 | Day 19-23 |
| **實戰應用** | Python 程式化處理、串流、轉換、視覺化 | Day 24-29 |

---

## CityJSON 生態系現狀（截至 2026 年 9 月）

### 標準版本

CityJSON 目前最新版本為 **v2.0.2**（2026 年 3 月 27 日發布），是 **OGC 官方社群標準**（文件編號 20-072r5）[^cityjson-v202]。v2.0.2 的主要更新包括：允許 `CityJSONFeature` 擁有額外屬性（便於 OGC API-Features 整合）、以 `"const"` 取代單值 `"enum"`、擴充版本支援 `X.Y.Z` 格式、以及指向新的 Extensions Registry[^cityjson-release][^cityjson-ogc]。

### 最大新聞：FlatCityBuf

2025 年 7 月發表的 **FlatCityBuf** 是 CityJSON 生態系近年最大的突破——一種以 Google FlatBuffers 為基礎的雲端優化二進位格式[^flatcitybuf-announce]。到 2026 年 8 月，它已發展為四種獨立實作（Rust、C++、Python、TypeScript），全部通過共享的一致性測試套件[^flatcitybuf-update]。

最令人驚豔的是它的 **瀏覽器檢視器**（flatcitybuf-prototype.hideba.me）——荷蘭全國 3DBAG（**68GB、1070 萬棟建築**）的單一 `.fcb` 檔案直接從雲端儲存查詢，無伺服器、無 WebAssembly，純粹透過 HTTP range requests 達成即時瀏覽[^flatcitybuf-viewer]。

### 產業採用

CityJSON 的產業採用正在快速擴張[^cityjson-adoption]：

- **荷蘭 3DBAG**：全球第一個全國性 3D 建築物資料集，約 1000 萬棟 LoD2 建築物
- **新加坡**：開放 3D 城市模型以 CityJSON 格式發布
- **Smart Data Models**：CityJSON v2.0.2 已轉譯為 NGSI-LD/NGSI-v2 標準，共 33 種核心城市物件類別
- **軟體生態系**：GitHub 上超過 36 個相關倉儲，支援 FME、Blender、Rhino 3D/Grasshopper、Revit、QGIS、ParaView、xeokit 等平台

### GitHub 正在討論中的未來功能

從 CityJSON 規格的 GitHub issues 可以看到社群正在規劃中的功能[^cityjson-issues]：

| Issue | 主題 |
|-------|------|
| #222 | 從 JSON Schema 自動產生 TypeScript/C++/Python 型別定義 |
| #221 | LoD 以陣列形式指定 |
| #204 | 所有城市物件都支援 extra attributes |
| #203 | 語意表面也支援額外的擴充屬性 |
| #181 | 複合座標參考系統（Compound CRS）支援 |
| #206 | v3.0 的 `pointOfContact` 角色改名 |

### v3.0 展望

GitHub 上已經有 `v3` 標籤的 issue 在討論中[^cityjson-v3]，顯示 CityJSON 團隊正在為 **v3.0 做準備**。雖然目前還沒有正式的時間表，但從社群的討論方向來看，v3.0 可能聚焦於：更完善的型別系統、更彈性的 LoD 表示方式、以及對大型資料集的更佳支援。

---

## 給未來學習者的建議

經過 30 天的系統學習，以下是給想踏入 CityJSON 領域的開發者的建議：

### 學習路徑

1. **從 Day 1-6 開始**：建立對 CityJSON 的基本認識，安裝工具鏈（cjio、ninja viewer），親手寫一個最小檔案
2. **掌握幾何核心（Day 7-12）**：頂點系統與 transform 是理解所有後續內容的基礎
3. **取得真實資料（Day 28）**：直接從 3DBAG 下載荷蘭的建築資料來玩，真實資料比手寫範例更有感覺
4. **用 Python 串接（Day 29）**：寫一個簡單的統計腳本，把所學轉化為可重複使用的程式碼

### 避坑指南

- **Transform 不可忽略**：很多人直接讀取 `vertices` 卻忘了套用 `transform`，得到的座標是錯的
- **邊界陣列的深度**：每種幾何類型的 boundary array 深度不同，寫程式走訪時要特別注意
- **語意 values 的對應**：Solid 的 values 是二維陣列（shell → face），MultiSurface 是一維陣列（surface），搞混會出錯
- **物件不一定要有幾何**：某些城市物件（如 CityObjectGroup）可能沒有幾何資料

---

## 推薦資源

以下資源是這 30 天內容的核心來源，也是未來深入研究的最佳起點：

| 資源 | 說明 | 網址 |
|------|------|------|
| **CityJSON 官方網站** | 規格、新聞、軟體清單 | https://www.cityjson.org/ |
| **官方規格 v2.0.2** | 完整的標準文件 | https://www.cityjson.org/specs/2.0.2/ |
| **CityJSON GitHub** | 原始碼、issue 討論 | https://github.com/cityjson/specs |
| **Ninja Viewer** | 線上瀏覽器檢視器 | https://ninja.cityjson.org/ |
| **cjio 文件** | Python CLI 操作手冊 | https://cjio.readthedocs.io/ |
| **FlatCityBuf** | 雲端優化二進位格式 | https://www.cityjson.org/flatcitybuf/ |
| **3DBAG** | 荷蘭全國建築物資料集 | https://3dbag.nl/ |
| **TU Delft 3D 地理資訊團隊** | CityJSON 背後的開發團隊 | https://3d.bk.tudelft.nl/ |
| **Extensions Registry** | 官方擴充功能註冊表 | https://github.com/cityjson/extensions |
| **OGC CityJSON 標準** | OGC 正式標準頁面 | https://www.ogc.org/standards/cityjson/ |

---

## 心態轉變與技術成長

30 天前，CityJSON 對我來說是一個「聽過但沒用過」的格式。30 天後，這份學習透過輸出內化為知識體系。

這個過程中有幾個關鍵的心態轉變：

**從「讀規格」到「用規格」**：規格文件（`.bs` 格式的原始規格）一開始讀起來很吃力，但當你需要寫檔案、做轉換、除錯時，規格變成了解決問題的最佳夥伴。這 30 天系列的所有內容都直接來自官方規格與相關工具文件[^cityjson-specs]。

**從「JSON 格式」到「3D 城市資料架構」**：CityJSON 最吸引人的地方在於它用開發者熟悉的 JSON 包裝了一個嚴謹的 3D 城市資料模型。這 30 天學到的不只是一個格式，而是 **3D 城市資料該如何被結構化、儲存、傳輸與處理** 的一套完整方法論。

**從「消費者」到「參與者」**：CityJSON 是一個開放原始碼、開放標準的社群專案。學習的最終階段不只是使用它，而是有能力參與——提出 issue、貢獻 extension、改進工具。

---

## 未來的下一步

CityJSON 的發展方向非常明確：

1. **雲端原生**：FlatCityBuf 的出現讓 68GB 的全國級資料集可以在瀏覽器中即時查詢，這對數位雙生基礎設施是革命性的
2. **標準化深化**：從 OGC 社群標準到 Smart Data Models 整合，CityJSON 正在進入更多產業標準體系
3. **生態系擴張**：36+ 個 GitHub 倉儲、跨語言實作（Rust、C++、Python、TypeScript）、與主要 AEC/GIS 平台的整合，生態系的自我強化正在加速
4. **v3.0 醞釀中**：從社群的討論來看，v3.0 將會是一次重要的躍進

---

## 感謝

30 天不中斷的寫作是一場耐力賽。感謝 iThome 鐵人賽提供這個舞台，感謝 TU Delft 3D Geoinformation 團隊（Hugo Ledoux 與他的夥伴們）創造了 CityJSON 這樣優雅的格式，感謝所有在途中點開文章的讀者。

如果你從這個系列中學到了東西、找到了靈感、或產生了疑問——都歡迎在 CityJSON 的 GitHub 上提出 issue 或討論。開放標準的美麗之處就在於：**每個人不只是在學習，而是可以參與塑造這個技術的未來。**

---

## 參考資料

[^day1]: Day 1：前言、動機與 30 天路線圖。Retrieved 2026-09-27, from https://www.cityjson.org/

[^day2]: Day 2：什麼是 CityJSON？3D 城市模型的新世代格式。Retrieved 2026-09-27, from https://www.cityjson.org/

[^day3]: Day 3：CityJSON vs CityGML — 為什麼需要另一個格式？Retrieved 2026-09-27, from https://www.cityjson.org/filesize/

[^day4]: Day 4：CityJSON 的結構總覽。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#the-cityjson-object

[^day5]: Day 5：Hello CityJSON。Retrieved 2026-09-27, from https://ninja.cityjson.org/

[^day6]: Day 6：開發環境與工具鏈。Retrieved 2026-09-27, from https://www.cityjson.org/software/

[^day7]: Day 7：頂點系統與 Transform。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#the-transform-object

[^day8]: Day 8：基礎幾何類型。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#geometry-objects

[^day9]: Day 9：實體幾何。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#solid-geometry

[^day10]: Day 10：幾何邊界陣列深度解析。Retrieved 2026-09-27, from https://www.cityjson.org/tutorials/geom-arrays/

[^day11]: Day 11：幾何模板。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#geometry-templates

[^day12]: Day 12：多層級細節 (LoD)。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#levels-of-detail

[^day13]: Day 13：城市物件總覽。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#city-objects

[^day14]: Day 14：建築物家族。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#building-objects

[^day15]: Day 15：運輸與基礎設施。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#transportation-objects

[^day16]: Day 16：植栽、水體、地形。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#vegetation-objects

[^day17]: Day 17：語意表面。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#semantics

[^day18]: Day 18：Parent/Children 關係。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#city-objects

[^day19]: Day 19：外觀材質。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#appearance

[^day20]: Day 20：UV 座標與紋理貼圖實戰。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#texture

[^day21]: Day 21：Metadata 與 CRS。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#metadata

[^day22]: Day 22：Extensions 擴充機制。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#extensions

[^day23]: Day 23：寫一個自己的 Extension。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#extensions

[^day24]: Day 24：CityJSONSeq (JSON Lines) 串流處理。Retrieved 2026-09-27, from https://www.cityjson.org/news/2024/09/16/cityjsonseq/

[^day25]: Day 25：大型檔案處理策略。Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#handling-large-files

[^day26]: Day 26：cjio 指令一把罩。Retrieved 2026-09-27, from https://cjio.readthedocs.io/

[^day27]: Day 27：從 CityGML 轉換到 CityJSON。Retrieved 2026-09-27, from https://www.cityjson.org/tutorials/conversion/

[^day28]: Day 28：全球城市資料集巡禮。Retrieved 2026-09-27, from https://www.cityjson.org/datasets/

[^day29]: Day 29：用 Python 處理 CityJSON。Retrieved 2026-09-27, from https://cjio.readthedocs.io/en/stable/api_tutorial_basics.html

[^cityjson-v202]: CityJSON. (2026, March 27). CityJSON Specification v2.0.2. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/

[^cityjson-release]: CityJSON. (2026, March 27). CityJSON v2.0.2 Release. Retrieved 2026-09-27, from https://github.com/cityjson/specs/releases/tag/2.0.2

[^cityjson-ogc]: OGC. (2023, November 6). CityJSON v2.0 OGC Community Standard (20-072r5). Retrieved 2026-09-27, from https://docs.ogc.org/cs/20-072r5/20-072r5.html

[^flatcitybuf-announce]: 3D Geoinformation Group, TU Delft. (2025, July 17). FlatCityBuf: A new cloud-optimised CityJSON. Retrieved 2026-09-27, from https://www.cityjson.org/news/2025/07/17/flatcitybuf/

[^flatcitybuf-update]: CityJSON. (2026, August 10). FlatCityBuf now speaks C++, Python, TypeScript—and runs in your browser. Retrieved 2026-09-27, from https://www.cityjson.org/news/2026/08/10/flatcitybuf-updates/

[^flatcitybuf-viewer]: CityJSON. (n.d.). FlatCityBuf Web Viewer. Retrieved 2026-09-27, from https://www.cityjson.org/flatcitybuf/viewer/

[^cityjson-adoption]: CityJSON. (n.d.). Datasets & Software. Retrieved 2026-09-27, from https://www.cityjson.org/datasets/

[^cityjson-issues]: CityJSON. (n.d.). CityJSON Specs GitHub Issues. Retrieved 2026-09-27, from https://github.com/cityjson/specs/issues

[^cityjson-v3]: CityJSON. (n.d.). CityJSON v3.0 Planning Issue (v3 label). Retrieved 2026-09-27, from https://github.com/cityjson/specs/issues?q=is%3Aissue+label%3Av3

[^cityjson-specs]: CityJSON. (n.d.). CityJSON Specifications (GitHub). Retrieved 2026-09-27, from https://github.com/cityjson/specs