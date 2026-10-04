# Day 17：WBEM 總覽 — CIM 的通訊層

過去一週，我們深入探討了 CIM 的第三階段——Metamodel 與 MOF。從 Metamodel 的概念框架（Day 12），到 MOF 語法全解析（Day 13），再到真實 Schema 檔案的閱讀（Day 14），最後是 MOF Compiler 的實戰應用（Day 15）與 UML Profile（Day 16）。如果說這六天我們一直在研究 **CIM 的「模型」本身**，那麼從今天開始的第四階段，我們將把目光轉向 **CIM 的「通訊」**——也就是：當你有了 CIM Schema 定義的管理物件，你要**如何透過網路** 來查詢、建立、修改與刪除這些物件？

答案是 **WBEM**（Web-Based Enterprise Management）。

## CIM 定義「什麼」，WBEM 定義「怎麼傳」

在深入 WBEM 之前，我們必須先建立一個關鍵的認知：**CIM 是協定中立的（Protocol-Neutral）**。

回顧 Day 3 的核心設計哲學，CIM 從設計之初就刻意將「管理模型」與「傳輸機制」分離。這意味著 CIM Schema 本身不包含任何關於「如何存取這些物件」的資訊——它只定義物件的類別、屬性、關聯與方法，而不管這些物件是透過 SNMP、HTTP、SOAP 還是任何其他協定來存取[^dmtf-cim-page]。

這種分離正是 RFC 3444 中所區分的 **Information Model（資訊模型）** 與 **Data Model（資料模型）** 的關鍵差異：CIM Schema 屬於資訊模型層次，它獨立於任何具體的通訊協定與實作細節[^rfc3444]。

而 **WBEM** 的角色，正是填補 CIM 與實際通訊之間的空缺：

> **CIM**（資訊模型）：管理什麼？如何組織？
> **WBEM**（通訊框架）：怎麼存取？用什麼協定？如何編碼？

更具體地說，WBEM 是 DMTF 定義的一組管理技術與協定標準，它規範了 CIM 資源如何被**發現、存取與操作**。WBEM 本身不是單一協定，而是一個由三種通訊協定組成的家族[^dmtf-wbem]：

| 協定 | 文件編號 | 最新版本 | 發佈日期 | 風格 |
|------|---------|---------|---------|------|
| **CIM-XML** | DSP0200 + DSP0201 | 1.4.0 + 2.4.0 | 2013-08 | XML-RPC over HTTP |
| **WS-Management** | DSP0226 + DSP0227 + DSP0230 | 1.2.0 + 1.2.0 + 1.1.0 | 2014-09 | SOAP/Web Services |
| **CIM-RS** | DSP0210 + DSP0211 | 2.0.0 + 2.0.0 | 2015-03 | REST/JSON |

這三種協定使用了三種截然不同的通訊風格，但它們服務於同一個目的：讓用戶端能夠以標準化方式操作 CIM 管理物件。

---

## WBEM 的歷史：從「Web-Based Enterprise Management」開始

WBEM 的故事始於 **1996 年**，由 BMC Software、Cisco、Compaq、Intel 與 Microsoft 五家公司共同推動。當時的管理標準世界正處於 SNMP 稱霸但已顯露侷限的時代——SNMP 的資料模型（SMI）缺乏物件導向的關聯性與繼承能力，而 CMIP 雖然功能強大卻過於複雜難以大規模部署[^wikipedia-wbem]。

### 關鍵里程碑

| 年份 | 事件 |
|------|------|
| **1996** | BMC Software、Cisco、Compaq、Intel、Microsoft 共同發起 WBEM 倡議 |
| **1999** | CIM-XML 首版（DSP0200 v1.0.0、DSP0201 v2.0.0） |
| **2003-03** | DMTF 發佈 WBEM 三份核心規格「最終版」：CIM 規格、xmlCIM 編碼、CIM Operations over HTTP，標誌 WBEM 成為成熟標準[^xmlcover-cim] |
| **2008** | WS-Management v1.0.0 發佈，導入 SOAP 管理 |
| **2013-02** | WS-Management 被 ISO/IEC 採納為國際標準 ISO/IEC 17963:2013[^dmtf-iso] |
| **2013-02** | CIM-RS v1.0.0 發佈，導入 RESTful 管理 |
| **2014-2015** | 三個協定皆達到最新穩定版本：CIM-XML 1.4.0、WS-Management 1.2.0、CIM-RS 2.0.0 |

這三個協定的演進，恰好反映了網際網路技術的發展軌跡：1990 年代末的 XML-RPC 浪潮（CIM-XML）、2000 年代中期的 SOAP/Web Services 浪潮（WS-Management），以及 2010 年代以來的 REST/JSON 浪潮（CIM-RS）[^dmtf-wbem][^dmtf-wsman]。

### 命名的由來

「Web-Based Enterprise Management」這個名稱在 1996 年提出時，「Web-Based」指的是利用**網際網路技術**（HTTP、HTML、XML）來進行企業管理，而不是指透過瀏覽器操作。這在當時是一個前瞻性的視角——因為 1996 年連 HTTP/1.1 都尚未成為標準（RFC 2616 在 1999 年才發佈）。

有趣的是，WBEM 這個名稱雖然至今仍在使用，但隨著時間推移，它實際上已經成為 CIM 通訊層的總稱，而非特指某一個具體協定。

---

## DMTF Stack Map：CIM ↔ Profiles ↔ WBEM ↔ Initiatives

要理解 WBEM 在 DMTF 架構中的位置，最好的方式就是看 DMTF 的 **Stack Map（技術堆疊圖）**[^dmtf-stackmap]：

```
┌─────────────────────────────────────────────────────────┐
│                   DMTF Initiatives                        │
│  DASH（桌管）／ SMASH（伺服器）／ SMI-S（儲存）           │
│  VMAN（虛擬化）／ CDM（診斷模型）                         │
├─────────────────────────────────────────────────────────┤
│                   Management Profiles                     │
│  為特定場景挑選 CIM 類別的子集（DSP0232、DSP1001 等）    │
├─────────────────────────────────────────────────────────┤
│               WBEM Communication Protocols                │
│  CIM-XML (DSP0200) │ WS-Management (DSP0226) │ CIM-RS   │
│  + 發現、過濾、URI 對映等支援規格                         │
├─────────────────────────────────────────────────────────┤
│               CIM（Common Information Model）              │
│  CIM Metamodel (DSP0004) → CIM Schema (v2.56.0)          │
│  以 MOF / UML / XML Schema 定義的物件模型                │
└─────────────────────────────────────────────────────────┘
```

這個堆疊的每一層都有明確的職責[^dmtf-wbem]：

### 第一層：CIM（Common Information Model）

這是整個架構的**基石**。CIM Metamodel（DSP0004 v3.0.1）定義了建構管理物件的「語法規則」，而 CIM Schema（v2.56.0）則實際定義了數百個管理類別——從 `CIM_ComputerSystem` 到 `CIM_Process`，從 `CIM_NetworkInterface` 到 `CIM_PhysicalPackage`。

### 第二層：WBEM Communication Protocols

這是我們本週的主題。WBEM 三協定的共同目標是提供 CIM 模型的標準化存取，但它們各自採用了不同的通訊風格。此外，WBEM 還包含一系列支援規格：

| 文件編號 | 名稱 | 最新版本 |
|---------|------|---------|
| DSP0205 | WBEM Discovery Using SLP | 1.0.1 |
| DSP0206 | WBEM SLP Template | 2.0.1 |
| DSP0207 | WBEM URI Mapping | 1.0.1 |
| DSP0212 | Filter Query Language | 1.0.1 |
| DSP0202 | CIM Query Language（CQL） | 1.0.0 |

這些規格共同構成了 WBEM 的完整生態系。

### 第三層：Management Profiles

Profiles 是 CIM 的「裁剪版」。因為 CIM Schema 涵蓋了非常廣泛的領域（系統、網路、儲存、安全、政策……），而一個具體的管理場景（例如「管理一台桌上型電腦」）通常只需要其中的一小部分。Profile 的功能就是從 CIM Schema 中選取適當的類別，定義它們之間的約束條件，從而形成一個針對特定場景的精簡模型。

### 第四層：Initiatives

Initiatives 是 DMTF 將 CIM 與 WBEM 應用於特定垂直領域的完整方案。例如：
- **DASH**（Desktop and mobile Architecture for System Hardware）— 桌上型與行動裝置管理
- **SMASH**（Systems Management Architecture for Server Hardware）— 伺服器管理
- **SMI-S**（Storage Management Initiative Specification）— 企業儲存管理（由 SNIA 主導）
- **VMAN**（Virtualization Management）— 虛擬化管理
- **CDM**（Common Diagnostic Model）— 共同診斷模型

我們將在第六階段（Day 26-29）逐一深入探討這些 Initiatives。

### 關鍵設計原則：橫向獨立性

這個堆疊的一個重要設計原則是 **各層之間可以獨立演進**。CIM Schema 可以在不影響 WBEM 協定的情況下發佈新版本；WBEM 協定可以在不修改 CIM Schema 的情況下增加新的傳輸方式；Initiatives 可以針對特定場景定義自己的 Profile 而無需變更 CIM 核心。

這種鬆散耦合（Loose Coupling）的設計，使得 CIM 能在將近 30 年的時間裡保持高度穩定性——CIM 的核心類別從 1999 年到今天幾乎沒有重大不相容變更[^dmtf-cim-page]。

---

## 三種協定的宏觀比較

在我們後續三天分別深入 CIM-XML、WS-Management 與 CIM-RS 之前，這裡先給出一個宏觀的比較表：

| 面向 | CIM-XML | WS-Management | CIM-RS |
|------|---------|---------------|--------|
| **通訊模式** | RPC（遠端程序呼叫） | SOAP（Simple Object Access Protocol） | REST（Representational State Transfer） |
| **酬載編碼** | XML（自訂 DTD） | XML（SOAP Envelope） | JSON（DSP0211）或 XML |
| **預設埠** | 5988（HTTP）／5989（HTTPS） | 80/443（傳統），5985/5986（WinRM） | 80/443 |
| **ISO 標準** | 否 | 是（ISO/IEC 17963:2013） | 否 |
| **主要實作** | OpenPegasus、pywbem、SBLIM | Microsoft WinRM、Intel AMT | 較少 |
| **成熟度** | 最成熟（1999～） | 成熟（2008～） | 較新（2013～） |
| **學習曲線** | 中 | 高（SOAP 複雜性） | 低（RESTful） |
| **防火牆友善** | 中（需自訂埠） | 高（標準 HTTP/HTTPS） | 高（標準 HTTP/HTTPS） |

有趣的是，CIM-XML 雖然最早，但至今仍在 Linux 與 Python 的 CIM 生態系中最為活躍；WS-Management 雖然是 ISO 標準，但其主要實作幾乎全部來自 Microsoft 的 WinRM；CIM-RS 最現代化，但由於 DMTF 在 2015 年之後將重心轉向 Initiatives（特別是 DASH 與 SMASH），其生態系相對較小[^dmtf-wbem]。

---

## 小結

1. **CIM 是協定中立的資訊模型**，它定義「管理什麼」，而 **WBEM 是通訊框架**，它定義「怎麼存取」。
2. **WBEM 有三種通訊協定**：CIM-XML（XML-RPC 風格，1999）、WS-Management（SOAP 風格，2008）、CIM-RS（RESTful 風格，2013）。
3. **DMTF Stack Map** 呈現了四層架構：CIM → WBEM → Profiles → Initiatives，各層之間鬆散耦合、獨立演進。
4. **WBEM 的歷史反映了網際網路技術的演進**：從 XML-RPC 到 SOAP 再到 REST/JSON。
5. **三種協定各有取捨**：CIM-XML 最成熟、WS-Management 是 ISO 標準且廣泛部署（WinRM）、CIM-RS 最現代。

明天開始，我們將逐一深入探索這三種協定的細節——從 CIM-XML 的 HTTP 訊息格式與 XML 編碼開始。

[^dmtf-cim-page]: Distributed Management Task Force. (n.d.). CIM Home. Retrieved 2026-10-03, from https://www.dmtf.org/standards/cim
[^rfc3444]: Pras, A., & Schoenwaelder, J. (2003, January). On the Difference between Information Models and Data Models (RFC 3444). IETF. Retrieved 2026-10-03, from https://www.rfc-editor.org/info/rfc3444
[^dmtf-wbem]: Distributed Management Task Force. (n.d.). WBEM Standards — Web-Based Enterprise Management. Retrieved 2026-10-03, from https://www.dmtf.org/standards/wbem
[^dmtf-stackmap]: Distributed Management Task Force. (n.d.). DMTF Technologies Stack Map. Retrieved 2026-10-03, from https://www.dmtf.org/standards/stackmap
[^dmtf-wsman]: Distributed Management Task Force. (n.d.). WS-Management. Retrieved 2026-10-03, from https://www.dmtf.org/standards/wsman
[^dmtf-iso]: Distributed Management Task Force. (2013, February 26). DMTF Standards Gain Global Recognition — WS-Management adopted as ISO/IEC 17963:2013. Retrieved 2026-10-03, from https://www.dmtf.org/content/dmtf-standards-gain-global-recognition
[^wikipedia-wbem]: Wikipedia. (n.d.). Web-Based Enterprise Management. Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Web-Based_Enterprise_Management
[^xmlcover-cim]: OASIS XML Cover Pages. (2003). DMTF Publishes Final Release of Web-Based Enterprise Management (WBEM) Specifications. Retrieved 2026-10-03, from https://xml.coverpages.org/xmlCIM.html