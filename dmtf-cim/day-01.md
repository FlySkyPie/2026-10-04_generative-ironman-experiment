# Day 1：前言與系列大綱 — 從概念到實戰：DMTF CIM 共通資訊模型 30 天完全攻略

## 作者自我介紹

筆者是一名對系統管理標準與資訊模型有長期興趣的 IT 從業人員。在日常工作中，我發現許多管理工具雖然功能強大，但它們的資料模型往往各自為政——同一台伺服器在不同管理工具中被以截然不同的方式描述，導致整合成本居高不下。這個痛點驅使筆者開始探索 DMTF（Distributed Management Task Force）所制定的 CIM（Common Information Model，共通資訊模型），並驚喜地發現這是一個已持續發展超過 25 年、在各大作業系統中默默運作卻鮮為中文技術社群所知的標準。

## 為什麼選擇 CIM？

如果你曾經：

- 使用 `wmic` 或 PowerShell 查詢 Windows 系統資訊
- 透過 DMTF Redfish API 管理伺服器硬體
- 接觸過 Intel AMT（Active Management Technology）的頻外管理
- 好奇「管理資訊模型」與「資料模型」之間的差異

那麼你已經在與 CIM 的遺產或衍生標準互動。CIM 是上述所有技術的共同血脈——它定義了管理資訊的「語言」本身。

## 什麼是 DMTF CIM？

**CIM（Common Information Model，共通資訊模型）** 是由 DMTF 維護的一項開放標準，定義了 IT 環境中被管理元素（電腦系統、作業系統、網路、應用程式、儲存設備等）如何以一組共通的物件與關係來表示[^dmtf-cim-page]。其核心理念是：一旦管理資訊有了統一的描述方式，管理軟體就能夠撰寫一次、跨實作執行，無需昂貴的格式轉換[^wikipedia-cim]。

DMTF 本身成立於 1992 年，最初名為 **Desktop Management Task Force**，首個標準是 DMI（Desktop Management Interface）。1999 年，組織更名為 **Distributed Management Task Force**，反映其從桌管到分散式管理的擴張，同年 CIM 的工作正式展開[^wikipedia-dmtf]。如今 DMTF 的標準涵蓋 Redfish（資料中心 RESTful 管理）、SPDM（安全認證）、SMBIOS（系統韌體資訊）等廣泛領域，而 CIM 則是其中歷史最悠久的核心標準之一[^dmtf-about]。

CIM 的架構奠基於三個層次：

| 層次 | 說明 |
|------|------|
| **Core Model（核心模型）** | 最抽象的基礎類別，如 `CIM_ManagedElement`，是所有管理物件的起點 |
| **Common Model（共通模型）** | 標準的 IT 元素類別，如 `CIM_ComputerSystem`、`CIM_OperatingSystem`、`CIM_Process` |
| **Extension Schema（擴充綱要）** | 允許廠商在標準類別上衍生產品專屬的類別與屬性 |

這種三層設計使得 CIM 既能保證跨平台互通性，又能容納各家廠商的自訂需求[^dmtf-cim-page]。

## 30 天路線圖

本系列採用 **階段劃分法**，將 30 天分為 6 大階段：

```
Day 1      — 前言與系列大綱（本日）
─── 第一階段：CIM 基礎（Days 2–6）───
Day 2–6    — 認識 CIM：為什麼需要它？  誕生背景、核心哲學、類別體系
─── 第二階段：CIM Schema 架構（Days 7–11）───
Day 7–11   — 三層 Schema、核心領域、類別層次
─── 第三階段：Metamodel 與 MOF（Days 12–16）───
Day 12–16  — Meta Schema、MOF 語法、MOF Compiler
─── 第四階段：WBEM 通訊協定（Days 17–20）───
Day 17–20  — CIM-XML、WS-Management、CIM-RS
─── 第五階段：Profiles 與真實實作（Days 21–25）───
Day 21–25  — Management Profiles、OpenPegasus、pywbem、Windows WMI
─── 第六階段：應用、案例與展望（Days 26–29）───
Day 26–29  — DASH、SMASH、SMI-S、未來發展
Day 30     — 完賽心得與總結
```

### 逐階段說明

**第一階段：CIM 基礎（Days 2–6）** — 我們將回到 1990 年代管理協定百家爭鳴的時代，理解 CIM 為何誕生、它的物件導向設計哲學、以及核心類別體系如何建構。這階段的目標是建立 CIM 的基本語彙。

**第二階段：CIM Schema 架構（Days 7–11）** — 深入 CIM Schema 的各個領域模組，包括 System、Device、Network、Application、Event、Security、Policy、Physical 等。讀者將理解 CIM 如何以統一的方式描述從處理器到資料庫的所有管理元素。

**第三階段：Metamodel 與 MOF（Days 12–16）** — 這是最「硬核」的階段。我們將探討「模型的模型」（Metamodel），學習 MOF（Managed Object Format）語法，實際閱讀真實的 Schema 檔案，並了解 MOF Compiler 如何將 Schema 編譯為可操作物件。

**第四階段：WBEM 通訊協定（Days 17–20）** — 模型定義好了，如何透過網路傳輸與操作？WBEM 三協定家族——CIM-XML（基於 HTTP+XML）、WS-Management（基於 SOAP）、CIM-RS（基於 RESTful JSON）——各自適合不同的場景。

**第五階段：Profiles 與真實實作（Days 21–25）** — 從紙上標準到真實世界。我們將探討 Management Profile 的結構，並橫跨 OpenPegasus（C++）、pywbem（Python）、Windows WMI/MI、Linux SBLIM、Solaris WBEM 等多個平台。

**第六階段：應用、案例與展望（Days 26–29）** — CIM 在 DASH（桌管）、SMASH（伺服器管理）、SMI-S（企業儲存）中的應用。最後展望 CIM 在 AIOps 和邊際運算時代的未來角色。

## 讀者預期收穫

完成本系列後，你將能夠：

- **理解 CIM 全貌** — 知道 CIM 三層 Schema 如何分工、Core/Common/Extension 各自的角色
- **讀寫 MOF 語法** — 能夠閱讀真實的 `.mof` 檔案，甚至自行定義簡單的 CIM 類別
- **掌握 WBEM 三協定** — 理解 CIM-XML、WS-Management、CIM-RS 的設計取捨與適用場景
- **操作實作平台** — 在 pywbem、OpenPegasus 或 Windows WMI 中查詢與操作 CIM 物件
- **連接其他標準** — 理解 DASH、SMASH、SMI-S 與 CIM 的關係，以及 CIM 在 Redfish 等新興標準中的間接影響

## 本系列的特點

- **中文圈首個完整介紹 DMTF CIM 的系列文** — 絕大多數 CIM 資料僅有英文規格文件，本系列旨在填補此一繁體中文知識空白
- **理論與實戰並重** — 不是規格文件的逐字翻譯，而是搭配實際操作（pywbem CLI、OpenPegasus cimcli、Windows PowerShell）
- **多平台比較視角** — 橫跨 Windows WMI/MI、Linux SBLIM/OpenPegasus、Solaris WBEM Services

## 期待你的追蹤

明天開始，我們將正式進入第一階段，從 1990 年代管理標準的「戰國時代」講起，看看 CIM 究竟解決了什麼問題。如果對系統管理標準、資訊模型設計或 open source 管理工具感興趣，歡迎每日追蹤。

## 參考資料

[^dmtf-cim-page]: DMTF. (n.d.). CIM — Common Information Model. Retrieved 2026-10-03, from https://www.dmtf.org/standards/cim
[^wikipedia-cim]: Wikipedia. (2025). Common Information Model (computing). Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Common_Information_Model_(computing)
[^wikipedia-dmtf]: Wikipedia. (2025). Distributed Management Task Force. Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Distributed_Management_Task_Force
[^dmtf-about]: DMTF. (n.d.). About DMTF. Retrieved 2026-10-03, from https://www.dmtf.org/about