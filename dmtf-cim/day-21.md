# Day 21：Management Profiles — 為場景量身定制的 CIM 子集

昨天我們完成了第四階段「WBEM 通訊協定」的最後一篇——CIM-RS，為三種通訊協定的探討畫下句號。今天開始的第五階段，我們將視角從「模型」與「通訊」拉到更高一層，探討 **Management Profiles（管理輪廓）**——這是 CIM 從理論模型走向實際落地的最後一塊拼圖。

如果你還記得 Day 17 的 DMTF Stack Map，Architecture 的第二層是 WBEM 通訊協定，第四層是 DASH/SMASH 等垂直應用（Initiatives），而夾在兩者之間的第三層就是 **Management Profiles**[^dmtf-stackmap]。

## Profiles 的角色：從「通用」到「專用」

CIM Schema 是一套龐大而全面的管理模型，涵蓋伺服器、網路、儲存、安全性、軟體、指標等數百個類別。這個廣度的代價是：**對於任何一個具體的管理場景，CIM 都太過龐大了**。

舉例來說，如果你只是想管理一台桌上型電腦的電力狀態，你不需要了解整個 `CIM_Network`、`CIM_Storage`、`CIM_Database` 領域——你只需要 `CIM_PowerSupply`、`CIM_Chassis`、`CIM_Sensor` 等少數幾個類別以及它們之間的關聯。

Management Profiles 正是為此而生的機制。DMTF 將其定義為[^dmtf-profiles]：

> A Profile defines the CIM model and associated behavior for a particular management domain (including the CIM classes, associations, indications, methods and properties). This provides a unified way of describing the management domain in CIM, which contributes to interoperability and helps with ease of use.
>
>（輪廓為特定管理領域定義了 CIM 模型與相關行為——包括 CIM 的類別、關聯、指標、方法與屬性。這提供了一種統一的方式來描述 CIM 中的管理領域，有助於提升互通性與易用性。）

每個 Profile 對裝置實作者來說是一份藍圖，對管理應用開發者來說是一份 API 規格書——雙方只須遵守同一份 Profile，即可確保互通。

## Profile 的基本結構

根據 DMTF 的 **DSP1001（Management Profile Usage Guide）** v1.2.0（2014-07-31），一份標準的 Management Profile 主要由以下部分構成[^dmtf-dsp1001]：

### Central Class（中心類別）

每個 Profile 必定有一個 **Center Class**，它是這個管理領域的入口點與聚焦點。例如「Power State Management Profile」的 Center Class 是 `CIM_PowerSupply`——所有與電源狀態管理相關的操作都圍繞這個類別展開。

### Scoping Class（範圍類別）與關聯

為了將 Center Class 的實例放置在正確的管理脈絡中，Profile 會定義一個 **Scope Class**。Scoping Class 提供實例存在的生命週期範圍與環境脈絡——通常是一個更高層次的類別（如 `CIM_ComputerSystem`、`CIM_ServiceProcessor`），其生命週期決定了 Profile 實例的生滅。

Center Class 與 Scoping Class 之間的關聯（Association）是 Profile 的骨架，它定義了：

- 一個 Scoping Class 可以有多少個 Center Class 實例（基數）
- 雙方的存取路徑與導航方向
- 生命週期依賴關係（級聯刪除、範圍邊界）

### 屬性、方法、指標與列舉值

Profile 從 CIM Schema 中選取具體的類別時，並非採用全部屬性。相反地，它會精確列出[^dmtf-profiles]：

| 項目 | 說明 |
|------|------|
| **必備屬性（Mandatory Properties）** | 實作此 Profile 的裝置必須支援的屬性 |
| **選用屬性（Optional Properties）** | 可以但不須要支援的屬性 |
| **固定值（Fixed Values）** | 某些屬性被固定為特定的列舉值或常數，可減少實作差異 |
| **方法（Methods）** | 此 Profile 支援的 CIM 方法（Extrinsic Methods） |
| **指標（Indications）** | 此 Profile 可能發送的事件通知 |
| **型別限制** | 對屬性型別、數值範圍、字串長度的進一步約束 |

這種精確選取機制是 Profile 的核心價值——它消除了 CIM Schema 的模糊性，讓實作者知道什麼是一定要實作的、什麼是可選擇的。

## DSP1001：了解 Profile 的指南

要深入了解 Profile 的結構，最重要的文件是 **DSP1001（Management Profile Usage Guide）**[^dmtf-dsp1001]：

| 版本 | 發佈日期 | 狀態 |
|------|---------|------|
| **1.2.0** | **2014-07-31** | **Standard（最新）** |
| 1.1.1 | 2014-02-11 | Standard |
| 1.1.0 | 2011-06-30 | Standard |
| 1.0.1 | 2009-08-05 | Standard |

這份指南並非定義任何一個具體的 Profile，而是教導 **如何撰寫 Profile**。它包含：

1. **Profile 文件的規範章節結構** — 每一份 Profile 文件都應包含的章節（Scope、Normative References、Terms and Definitions、Profile Definition、Center Class Specification、Scoping Class Specification、Compliance 等）
2. **Profile 元素的標記規則** — 如何用表格與清單列出必備/選用屬性、固定值、列舉限制
3. **Profile 命名與編號規則** — DSP 編號的分配方式與文件命名慣例
4. **Profile 間的依賴關係** — 一個 Profile 可以基於另一個 Profile 擴充，形成層次關係

這讓 Profile 的撰寫成為一種 **半結構化的過程**——只要遵循 DSP1001 的範本，任何人都可以為新的管理領域撰寫標準化的 Profile。

## DSP1034：Simple Identity Management Profile

讓我們以 **DSP1034（Simple Identity Management Profile）** v1.1.0（2012-12-13）為例，實際看到 Management Profile 的樣貌[^dmtf-dsp1034]：

| 屬性 | 值 |
|------|----|
| **Center Class** | `CIM_UserIdentity`（使用者身分） |
| **Scope Class** | `CIM_Service`（管理服務為範圍邊界） |
| **必備屬性** | `Name`、`UserPassword`、`Enabled`、`UserId` |
| **選用屬性** | `UserCertificate`、`Passphrase`、`BiometricData` |
| **主要方法** | `AuthenticateUser`（驗證使用者身分） |
| **安全考量** | 密碼傳輸必須使用 TLS 加密，不得以明文傳送 |

這個 Profile 的目的非常單純：管理裝置上的使用者帳號資訊——使用者名稱、密碼、啟用/停用狀態、憑證等。任何實作此 Profile 的裝置，管理應用都可以用統一的方式查詢使用者清單、驗證使用者身分、修改密碼，無論該裝置是伺服器、路由器還是儲存陣列。

從這個例子可以看出 Profile 的威力：**同一套 CIM Schema 的類別，在不同的 Profile 中被選取與限制，就能描述截然不同的管理領域**。

## DSP0800–DSP0845 Profile 系列一覽

如果 DSP1034 是 CIM Schema 層次的 Profile，那麼 **DSP0800–DSP0845** 系列就是同一組 Profile 的 **協定綁定版本**——它們定義了這些 Profile 如何透過 **SM CLP（Server Management Command Line Protocol）** 來操作[^dmtf-dsp0800]。

SM CLP 是 SMASH（Systems Management Architecture for Server Hardware）所定義的命令列管理協定，標準化為 **ANSI INCITS 438-2008**[^dmtf-smash]。DSP0800–DSP0845 系列的工作，就是把 CIM Schema 層次的 Profile 操作——例如「讀取電源狀態」、「設定開機順序」——映射為具體的 SM CLP 命令列語法。

| DSP # | Profile 名稱 | 用途 |
|-------|-------------|------|
| **DSP0800** | Base Server Profile | 基本伺服器管理 |
| **DSP0803** | SM CLP Admin Domain Profile | 管理領域操作 |
| **DSP0804** | Modular System Profile | 模組化系統管理 |
| **DSP0805** | Sensors Profile | 感應器讀取（溫度、電壓等） |
| **DSP0808** | CPU Profile | 處理器管理 |
| **DSP0809** | System Memory Profile | 系統記憶體管理 |
| **DSP0810** | Record Log Profile | 紀錄日誌存取 |
| **DSP0811** | Simple Identity Management Profile | 使用者身分管理（對應 DSP1034） |
| **DSP0812** | Physical Asset Profile | 硬體資產清點 |
| **DSP0813** | Boot Control Profile | 開機順序控制 |
| **DSP0814** | Fan Profile | 風扇管理 |
| **DSP0815** | Ethernet Port Profile | 乙太網路埠管理 |
| **DSP0822** | Power Supply Profile | 電源供應管理 |
| **DSP0823** | Power State Management Profile | 電源狀態管理 |
| **DSP0824** | Service Processor Profile | 服務處理器管理 |
| **DSP0826** | Software Inventory Profile | 軟體庫存查詢 |
| **DSP0827** | Software Update Profile | 軟體更新操作 |
| **DSP0834** | Computer System Profile | 電腦系統管理 |
| **DSP0835** | Indicator LED Profile | LED 指示燈控制 |

以上全部為 v1.0.0，於 2009 年發佈。

注意這裡的模式：每個 Profile 的名稱都對應一個特定的管理領域，且這些領域有很強的可組合性（Composability）——一台伺服器可以同時實作 Base Server Profile、CPU Profile、Memory Profile、Power Supply Profile、Boot Control Profile……它們彼此獨立又互補，共同構成完整的伺服器管理能力。

### Profile 的組合與發現

管理應用如何知道一台裝置支援哪些 Profile？答案是 **DSP1033（Profile Registration Profile）** v1.1.0（2014-05-25）。這是一個「關於 Profile 的 Profile」——它定義了 Profile 的註冊機制，讓裝置可以自我宣告它支援的 Profile 清單[^dmtf-dsp1033]。

管理用戶端可以查詢 `CIM_RegisteredProfile` 類別的實例，得知裝置支援哪些 Profile，然後再針對性地呼叫對應 Profile 的操作。這正是 DASH/SMASH 中「Phase 1（RMCP 發現）→ Phase 2（Identify 確認能力）→ Phase 3（Profile 操作）」這套發現流程的資料來源。

## 如何閱讀一份 Profile 文件

綜合以上討論，閱讀一份 DMTF Management Profile 文件的建議步驟如下：

1. **確認 Profile 編號與版本** — 確認文件的 DSP 編號（如 DSP1034）與版本號，以及它所參照的 CIM Schema 版本
2. **找出 Central Class** — Profile 定義的第一個表格通常就是 Central Class 的規格，包括類別名稱與繼承路徑
3. **找出 Scoping Class** — 確定 Profile 的範圍邊界，以及 Central Class 與 Scoping Class 的關聯型別與基數
4. **辨識必備 vs 選用屬性** — 表格中的 `Mandatory` 與 `Optional` 欄位會清楚標示哪些屬性必須實作
5. **理解固定值與列舉限制** — Profile 會對某些屬性指定固定值或限制列舉範圍，這是確保互通性的關鍵
6. **檢視方法與指標** — 如果有定義 Extrinsic Methods 或 Indications，通常列在 Profile 文件的最後部分
7. **查看依賴關係** — 有些 Profile 基於其他 Profile 建構，文件會列出其依賴的 Profile 編號

## 小結

1. **Management Profiles 是 CIM Schema 的場景專用子集**，它從數百個類別中精準選取特定管理領域所需的類別、屬性、方法與值。
2. **每個 Profile 由 Center Class 與 Scope Class 構成**，Center Class 是操作焦點，Scope Class 提供生命週期範圍。
3. **DSP1001 是撰寫 Profile 的方法指南**，DSP1033 則讓裝置可以自我宣告支援的 Profile 清單。
4. **DSP1034（Simple Identity Management Profile）** 是 Profile 的典型範例，用於管理使用者身分認證。
5. **DSP0800–DSP0845 系列**是 SM CLP 協定綁定版本的 Profile 定義，構成 SMASH 伺服器管理的基礎。
6. **Profile 的可組合性**使得一台裝置可以同時實作多個 Profile，互不衝突又互相補充。

明天我們將進入第五階段的第二篇——**OpenPegasus**，探討 CIM 在開源世界的旗艦級實作。

[^dmtf-stackmap]: Distributed Management Task Force. (n.d.). DMTF Technologies Stack Map. Retrieved 2026-10-03, from https://www.dmtf.org/standards/stackmap
[^dmtf-profiles]: Distributed Management Task Force. (n.d.). Management Profiles. Retrieved 2026-10-03, from https://www.dmtf.org/standards/profiles
[^dmtf-dsp1001]: Distributed Management Task Force. (2014, July 31). DSP1001 — Management Profile Usage Guide (v1.2.0). Retrieved 2026-10-03, from https://www.dmtf.org/dsp/DSP1001
[^dmtf-dsp1034]: Distributed Management Task Force. (2012, December 13). DSP1034 — Simple Identity Management Profile (v1.1.0). Retrieved 2026-10-03, from https://www.dmtf.org/dsp/DSP1034
[^dmtf-dsp0800]: Distributed Management Task Force. (2009). DSP0800 — Base Server Profile SM CLP Command Mapping Specification (v1.0.0). Retrieved 2026-10-03, from https://www.dmtf.org/dsp/DSP0800
[^dmtf-smash]: Distributed Management Task Force. (n.d.). SMASH — Systems Management Architecture for Server Hardware. Retrieved 2026-10-03, from https://www.dmtf.org/standards/smash
[^dmtf-dsp1033]: Distributed Management Task Force. (2014, May 25). DSP1033 — Profile Registration Profile (v1.1.0). Retrieved 2026-10-03, from https://www.dmtf.org/dsp/DSP1033