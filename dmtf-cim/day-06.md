# Day 6：CIM Schema 的版本演化 — 從 v2.0（1999）到 v2.56.0（2026）

前五天我們從 CIM 的誕生背景（Day 2）、設計哲學（Day 3）、核心類別（Day 4）、關聯與繼承（Day 5）一路深入。今天我們將視角拉高，從 **版本演化的宏觀圖景** 來審視 CIM 如何走過 27 年的持續發展歷程——一個從 1999 年 v2.0 到 2026 年 v2.56.0、跨越 56 個版本的長青標準。

## 從 v2.0（1999）到 v2.56.0（2026）：56 個版本的旅程

CIM Schema 的第一個正式版本是 1999 年發布的 **v2.0**[^wikipedia-cim]。DMTF 官方在 2.55.0 與 2.56.0 的釋出公告中明確指出，這些分別是「自 CIM 2.0 以來的第五十五次與第五十六次更新」[^dmtf-256-news][^dmtf-255-news]——換言之，每一版 Schema 更新都被視為對 v2.x 主版本的一次增量改進。

截至 2026 年，DMTF 官方網站列出的可追溯版本歷史始於 v2.21.0（2010 年 4 月 21 日）[^dmtf-schemas]：

| 版本 | 發布日期 | 間隔 |
|------|---------|------|
| 2.21.0 | 2010-04-21 | — |
| 2.22.1 | 2009-10-25 | （修正版） |
| 2.23.0 | 2009-10-25 | — |
| 2.24.1 | 2010-01-20 | — |
| 2.25.0 | 2010-03-31 | ~2 個月 |
| 2.26.0 | 2010-07-21 | ~4 個月 |
| 2.27.0 | 2010-11-15 | ~4 個月 |
| 2.28.0 | 2011-02-02 | ~2.5 個月 |
| 2.29.1 | 2011-06-03 | ~4 個月 |
| 2.30.0 | 2011-09-27 | ~4 個月 |
| 2.31.0 | 2011-12-22 | ~3 個月 |
| 2.31.1 | 2012-04-04 | ~3.5 個月（微幅修正） |
| 2.32.0 | 2012-04-17 | 13 天後 |
| 2.33.0 | 2012-06-28 | ~2 個月 |
| 2.34.0 | 2012-09-06 | ~2 個月 |
| 2.35.1 | 2013-04-06 | ~7 個月 |
| 2.36.0 | 2013-04-06 | 同日（平行發布） |
| 2.37.0 | 2013-06-06 | ~2 個月 |
| 2.38.0 | 2013-08-25 | ~2.5 個月 |
| 2.39.0 | 2013-12-14 | ~3.5 個月 |
| 2.40.0 | 2014-02-24 | ~2 個月 |
| 2.41.0 | 2014-06-20 | ~4 個月 |
| 2.42.0 | 2014-09-29 | ~3 個月 |
| 2.43.0 | 2015-01-27 | ~4 個月 |
| 2.44.0 | 2015-04-30 | ~3 個月 |
| 2.44.1 | 2015-06-22 | ~2 個月（微幅修正） |
| 2.45.0 | 2016-01-11 | ~7 個月 |
| 2.46.0 | 2016-05-25 | ~4.5 個月 |
| 2.47.0 | 2016-10-04 | ~4 個月 |
| 2.48.0 | 2016-12-14 | ~2.5 個月 |
| 2.49.0 | 2017-06-14 | ~6 個月 |
| 2.50.0 | 2018-01-04 | ~6.5 個月 |
| 2.51.0 | 2018-06-12 | ~5 個月 |
| 2.52.0 | 2019-05-25 | ~11.5 個月 |
| 2.53.0 | 2020-03-04 | ~9 個月 |
| 2.54.0 | 2020-10-26 | ~8 個月 |
| 2.54.1 | 2022-03-08 | ~16.5 個月 |
| 2.55.0 | 2024-02-29 | ~24 個月 |
| **2.56.0** | **2026-01-21** | **~23 個月** |

從這張表的節奏變化可以看出：CIM Schema 在 **2010–2014 年間** 大約每年發布 4–6 個版本（約 2–3 個月一版），節奏相當快；進入 **2015 年以後** 發布頻率逐漸放緩至一年大約 1–2 版；而 **2020 年後** 則穩定在每兩年一個主要版本（2.54.0 → 2.55.0 → 2.56.0）。

值得注意的是，有些版本採用了 x.y.z 的三段式編號（如 2.22.1、2.24.1、2.29.1、2.31.1、2.35.1、2.44.1、2.54.1），這些 patch 級版本通常用於較小幅度的修正或一致性調整，而非引入大量新功能。

## 版本編號慣例與命名模式

CIM Schema 的版本編號有一個有趣的歷史特點：**早期版本曾出現過 x.y.z 三段式編號**（例如 v2.22.1、v2.44.1），這些通常是微幅修正版（bug fix / alignment）。近期的版本則一律使用兩段式編號（2.55.0、2.56.0）。

每個版本都以 ZIP 壓縮檔形式發布，檔案命名模式也經歷了演化：

| 時期 | 命名慣例 | 範例 |
|------|---------|------|
| v2.21.0–v2.36.0 | `cim_schema_v2.xx.0-Final-MOFs.zip` | `cim_schema_v2.21.0-Final-MOFs.zip` |
| v2.37.0+ | `cim_schema_2.xx.0-Final-MOFs.zip` | `cim_schema_2.56.0-Final-MOFs.zip` |

從 v2.37.0（2013 年 6 月）開始，檔名中的 `v` 前綴被移除，此慣例沿用至今。

## 每個 Release 的發布內容

CIM Schema 的每一次正式發布，不僅包含模型定義本身，還附帶多種格式的產出檔案，以滿足不同使用場景的需求[^cim-schema-2560]。以 2.56.0 為例，釋出檔涵蓋以下內容：

| 壓縮檔 | 內容 | 用途 |
|--------|------|------|
| **MOFs**（Final + Experimental） | `.mof` 格式的完整 Schema 定義 | — **MOF Compiler**（如 pywbem `mof_compiler`、OpenPegasus、Windows `mofcomp.exe`）的原始輸入 |
| **XML Classes**（單一檔案版） | 將全部類別合併為一個 XML 檔 | — 快速檢索整個 Schema 的 XML 結構 |
| **XML Classes**（個別檔案版） | 每個類別一個獨立的 XML 檔 | — 針對特定類別的精準查詢與比對 |
| **XSD Classes**（個別檔案版） | XSD（XML Schema Definition） | — XML 工具鏈中的驗證與資料綁定 |
| **Annotated XSD**（自 v2.40.0 起） | 附帶額外註釋的 XSD | — 更豐富的文件化資料交換[^cim-schema-2400] |
| **HTML DOC** | 轉換為 HTML 格式的線上文件 | — 人機可讀的文件查找（DMTF 官方亦提供線上版） |
| **Visio（.vsd）** | UML 類別圖的 Microsoft Visio 格式 | — 使用 Visio 進行圖形化建模與修改 |
| **PDF（UML 圖）** | 各模型區域的 UML 類別圖 PDF | — 無需專用工具的圖形化瀏覽 |
| **Diff 檔** | 跨版本的差異比較 HTML | — 快速掌握版本間變更 |

這 9 類釋出檔分為 **Final** 和 **Experimental** 兩套，各自獨立壓縮。

CIM Schema 的模型區域（Model Area）則按以下領域分類，涵蓋從核心到應用的各個面向[^cim-schema-2560]：

| 模型區域 | 包含的主要類別 |
|---------|--------------|
| `CIM_Application` | 應用程式、服務、軟體元件 |
| `CIM_Core` | `CIM_ManagedElement`、`CIM_Collection`、`CIM_Product` |
| `CIM_Database` | 資料庫系統、資料表、SQL 物件 |
| `CIM_Device` | 處理器、記憶體、磁碟、感應器 |
| `CIM_Event` | 事件記錄、警報、日誌 |
| `CIM_Interop` | 互通性（WBEM 相關） |
| `CIM_IPsecPolicy` | 網路安全政策 |
| `CIM_Metrics` | 計量、統計、臨界值 |
| `CIM_Network` | 協定端點、LAN、Wi-Fi |
| `CIM_Physical` | 機箱、插槽、卡、實體記憶體 |
| `CIM_Policy` | 政策規則與條件 |
| `CIM_Security` | 認證、存取控制 |
| `CIM_Support` | 支援類別（BIOS、開機、診斷） |
| `CIM_System` | 電腦系統、作業系統、程序 |
| `CIM_User` | 使用者、組織 |

## Final vs Experimental 雙軌制

CIM Schema 從 v2.21.0 開始採用了 **Final / Experimental 雙軌制**，這是它得以在 27 年間兼顧**穩定性**與**創新速度**的核心機制[^cim-schema-2560]：

```
                ┌──────────────────┐
                │  Experimental     │ ← 新提案 × 最少 1 次發布
                │  （可向後不相容） │
                └────────┬─────────┘
                         ↓ 獲得至少 2 個獨立實作
                ┌──────────────────┐
                │     Final        │ ← 成熟穩定
                │  （保證向後相容）  │
                └──────────────────┘
```

| 屬性 | Final | Experimental |
|------|-------|-------------|
| **穩定性** | 保證向後相容：不得以向後不相容的方式變更 | **可能以向後不相容的方式變更** |
| **晉升條件** | 至少 **2 個獨立實作** 存在且累積足夠的實作經驗 | 經 CIM Forum 核准即可加入 |
| **檔案位置** | `cim_schema_X.XX.X-Final-*.zip` | `cim_schema_X.XX.X-Experimental-*.zip` |
| **同一版本中** | Final 是 Experimental 的穩定子集 | Experimental 包含 Final + 實驗中專案 |

此制度的目的是：**讓新概念先在 Experimental 中接受考驗（孵化），成熟後再晉升為 Final**，同時確保 Final 的使用者（如管理平台實作者）永遠不需要擔心 Schema 突然改變。

## 版本 2.56.0 亮點

2026 年 1 月 21 日發布的 **CIM Schema 2.56.0** 是截至 2026 年 10 月的最新版本[^dmtf-256-news][^cim-256-relnotes]。DMTF 官方以「automated build」方式發布——所有變更請求均以程式化方式套用，而非手動編輯。

### Final Schema 的關鍵變更

| 類別 | 變更請求 | 說明 |
|------|---------|------|
| `Device/CIM_Processor` | CR0230 | 為 `UpgradeMethod` 新增 **7 種新處理器插槽類型** |
| `Physical/CIM_PhysicalMemory` | CR0242 | 為 `MemoryTechnology` 新增 **MRDIMM**（多重排名 DIMM） |
| `Physical/CIM_PhysicalMemory` | — | 新增 `MemoryFormFactor` 屬性，**棄用**原有的 `FormFactor` |
| `Physical/CIM_PhysicalMemory` | — | 為 `MemoryType` 新增 MRDIMM |
| `Physical/CIM_PhysicalMemory` | — | 為 `MemoryFormFactor` 新增對應列舉值 |

其中 **MRDIMM（Multirank DIMM）** 是記憶體技術的重要更新：傳統 DIMM 只有單列（Single Rank）記憶體，MRDIMM 允許在單一 DIMM 上整合多個記憶體列（Rank），以滿足高效能伺服器對記憶體密度與頻寬不斷成長的需求。

**新增的 7 種處理器插槽**（CR0230）則反映了 x86 與 ARM 伺服器處理器實體封裝的演進趨勢。

### 聯盟貢獻模式

2.56.0 版 Schema 的貢獻來自 DMTF 及其**聯盟合作夥伴**[^cim-256-relnotes]：
- **SNIA（Storage Networking Industry Association）** — 儲存管理模型
- **OGF（Open Grid Forum）** — 網格與分散式計算管理
- **The Green Grid** — 節能管理模型
- **Printer Work Group（IEEE-ISTO 旗下）** — 列印裝置管理

這種跨聯盟的協作模式，使 CIM 能持續吸納來自不同產業領域的管理需求，維持其「共同資訊模型」的初衷。

## 歲月中沉澱的智慧

回顧 CIM Schema 的版本演化，有幾個值得的觀察：

1. **一致性高於新奇**：27 年間，CIM 從未做過 breaking change（破壞性變更）——所有 Final 類別的向後相容承諾一直維持至今。這在 IT 標準中極為罕見。

2. **增量演進，非革命**：每次更新的幅度都不大（幾個類別增減屬性、修改列舉值），但持續了 56 次。這種「緩慢但穩定」的節奏讓所有實作方有充裕時間跟進。

3. **從「版本頻繁」到「兩年一版」的成熟化**：2010–2014 年間每年 4–6 版的快速迭代，反映了 Schema 仍在快速擴展期；近年的兩年一版則說明了 CIM 已進入穩定維護期。

4. **檔案的多元化是包容性的表現**：MOF 給工具、XML/XSD 給整合、PDF 給文件、Visio 給模型設計師——同一份 Schema 以多種格式發布，降低了不同使用者的進入門檻。

明天（Day 7）我們將進入 **第二階段：CIM Schema 架構**，從 Core Schema 開始，深入探討 `CIM_ManagedElement` 到 `CIM_ManagedSystemElement` 的共同 DNA，以及 `OperationalStatus`、`HealthState` 等屬性背後的語意標準化。

## 參考資料

[^wikipedia-cim]: Wikipedia. (2025). Common Information Model (computing). Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Common_Information_Model_(computing)

[^dmtf-schemas]: DMTF. (n.d.). CIM Schemas — Version History. Retrieved 2026-10-03, from https://www.dmtf.org/standards/cim/schemas

[^dmtf-256-news]: DMTF. (2026). DMTF Releases CIM 2.56.0. Retrieved 2026-10-03, from https://www.dmtf.org/content/dmtf-releases-cim-256-0

[^dmtf-255-news]: DMTF. (2024). DMTF Releases CIM 2.55.0. Retrieved 2026-10-03, from https://www.dmtf.org/content/dmtf-releases-cim-255

[^cim-schema-2560]: DMTF. (2026). CIM Schema v2.56.0 Release Page. Retrieved 2026-10-03, from https://www.dmtf.org/standards/cim/cim_schema_v2560

[^cim-256-relnotes]: DMTF. (2026). CIM Schema 2.56.0 Release Notes. Retrieved 2026-10-03, from https://www.dmtf.org/sites/default/files/cim/cim_schema_v2560/releasenotes_0.html

[^cim-schema-2400]: DMTF. (2014). CIM Schema v2.40.0 Release Page. Retrieved 2026-10-03, from https://www.dmtf.org/standards/cim/cim_schema_v2400