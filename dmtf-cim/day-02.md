# Day 2：管理標準的戰國時代 — CIM 誕生的背景

## 開場：1990 年代的管理協定大戰

想像一下，你身處 1995 年的一座大型機房。裡面有數十台來自不同品牌的伺服器、交換器和儲存設備。要監控這些設備的狀態，你得面對一個令人頭痛的問題：每一家廠商都有自己的一套管理方式——有的使用 SNMP 的 MIB（Management Information Base），有的只支援專屬的命令列介面，少數甚至需要你安裝一份特定的管理軟體才能讀取硬碟溫度或 CPU 使用率。要整合這些資訊，你必須撰寫大量的自訂轉換程式，而每當新增一台設備，這個噩夢又重新開始。

這個場景精準地描述了 1990 年代 IT 管理領域的核心困境：**管理資訊模型不互通**。CIM（Common Information Model）正是在這樣的大背景下誕生的。

## 管理標準的三路人馬

### SNMP — 網路的輕量級霸主

1988 年，IETF（Internet Engineering Task Force）發表了 SNMPv1（Simple Network Management Protocol），RFC 1065–1067，隨後於 1990 年以 RFC 1155–1157 正式取代[^snmp-rfc]。SNMP 的設計哲學是**保持簡單**——當時的設計者認為，由 OSI/IETF/NSF 官方推動的 HEMS/CMIS/CMIP 方案「在當時的計算平台上既不可實現，也可能不可行」[^snmp-wiki]。

SNMP 使用 **MIB（Management Information Base）** 來描述管理變數。MIB 本質上是一個以 ASN.1（Abstract Syntax Notation One）語法定義的樹狀結構，每個節點由 OID（Object Identifier）唯一識別。管理站（Management Station）透過 GetRequest、GetNextRequest、SetRequest 三種基本操作來查詢或修改設備狀態，設備則以 Response 或 Trap（未經請求的警報）回應。

SNMP 的優點顯而易見：簡單、輕量、容易實作，因此迅速成為 TCP/IP 網路的**事實上管理標準**。但它的簡單也帶來了根本性的限制：

- **缺乏物件導向模型**：MIB 本質上是平面的變數集合，沒有類別（class）、沒有繼承（inheritance）、沒有關聯（association）的概念
- **廠商 MIB 各自為政**：每一家設備廠商都在自己的 OID 樹下定義私有的 MIB，導致同一種管理概念在不同廠商的裝置上使用完全不同的 OID 與索引結構
- **索引一致性問題**：即使是標準化程度最高的 MIB-II（RFC 1213），其表格索引在不同平台之間也缺乏一致性，造成「當從多台可能使用不同表格索引方案的設備取得資訊時，會出現關聯性問題」[^snmp-wiki]

簡言之，SNMP 解決了「如何傳輸管理資料」的問題，卻沒有解決「管理資料應該長什麼樣子」的問題。

### CMIP — 厚重但強大的 OSI 標準

與 SNMP 同時代競爭的是 ISO/OSI 陣營的 **CMIP（Common Management Information Protocol）**，由 ITU-T X.711 / ISO/IEC 9596-1 定義[^cmip-wiki]。CMIP 是為了解決 SNMP 的諸多限制而設計的——它功能豐富得多：

- 使用 **GDMO（Guidelines for the Definition of Managed Objects）** 以物件導向的方式定義管理物件的行為
- 不僅可以「設值」（SNMP 的 Set），還可以定義任意類型的 **動作（Action）**
- 具備完善的授權、存取控制和安全日誌機制
- 使用 X.500 目錄的識別名稱（Distinguished Name）來唯一標識管理物件

然而，CMIP 的豐富功能也意味著**極高的複雜度與資源需求**。Wikipedia 明確指出：「在網際網路上，大多數 TCP/IP 設備支援的是 SNMP 而非 CMIP，這是因為 CMIP 代理與管理系統的複雜度和資源需求過高」[^cmip-wiki]。CMIP 最終主要存活在**電信領域**——TMN（Telecommunications Management Network）架構中，用於管理 ISDN、ATM、SDH/SONET 和 GSM 網路[^tmn-wiki]。

### DMI — DMTF 的第一份標準

1992 年，**Desktop Management Task Force**（桌管管理任務小組）成立，由多家 PC 平台廠商（後期理事包括 Broadcom、Cisco、Dell、HPE、Intel、Lenovo 等）組成[^dmtf-wiki]。該組織的第一份標準是 **DMI（Desktop Management Interface）**，始於 1994 年，2.0 版於 1998 年 6 月 24 日發布[^dmi-wiki]。

DMI 的目標是讓桌上型電腦——即 PC——的硬體元件有標準化的管理資訊。它的核心概念是 **MIF（Management Information Format）**，一種用於描述靜態系統資料（型號 ID、序號、記憶體位址、連接埠位址）的格式。不同於 SNMP 的網路中心視角，DMI 聚焦於**單一電腦的內部元件**[^dmi-wiki]。

有趣的是，DMI 可以與 SNMP 共存：當 SNMP 查詢到達時，DMI 可以用其 MIF 資料填寫出 SNMP 的 MIB，作為一個代理（Proxy Agent）代表整個 LAN 區段的 DMI 設備回應[^dmi-wiki]。這種靈活性是早期管理標準的典型特徵——各標準各自解決一部分問題，卻缺乏統一的框架。

## 三路的困境：為什麼一個也不夠？

SNMP、CMIP 和 DMI 各自在某個面向解決了管理問題，但共同面臨一個根本性挑戰：**缺乏一個通協定中立的、物件導向的、可擴充的資訊模型**。

| 面向 | SNMP | CMIP | DMI |
|------|------|------|-----|
| 模型方式 | 樹狀 MIB（變數） | GDMO 物件 | MIF（靜態資料） |
| 通訊方式 | UDP（簡單） | 完整 OSI 堆疊 | 記憶體常駐代理 |
| 物件導向 | ✗ | ✓ | ✗（有限制） |
| 跨廠商互通 | ✗（眾多私有 MIB） | 有限（僅電信領域） | ✗（PC 限） |
| 部署規模 | 最大（TCP/IP 網路） | 極小（電信核心） | 中等（桌上型 PC） |

這些問題的根源在於：**沒有任何一份標準先釐清「資訊模型」與「資料模型」之間的區別**。

## 關鍵文獻：RFC 3444 — 資訊模型與資料模型之辨

2000 年 12 月，第八屆 IRTF 網路管理研究小組（NMRG）工作坊在德州奧斯汀舉行，來自 IETF、DMTF、ITU 和學術界的專家齊聚一堂。會議的產出便是 2003 年 1 月發布的 **RFC 3444**，題為 **"On the Difference between Information Models and Data Models"**[^rfc3444]。

RFC 3444（A. Pras & J. Schoenwaelder, 2003）提出了以下核心區別[^rfc3444]：

| 概念 | 定義 | 目的對象 |
|------|------|---------|
| **資訊模型（IM）** | 在**概念層級**定義受管物件，**獨立於具體實作或傳輸協定** | 設計者與操作者 |
| **資料模型（DM）** | 在**較低抽象層級**定義，包含實作與協定專屬細節（如 OID 指派、索引結構） | 實作者 |

RFC 3444 進一步指出，IETF 的 MIB 模組、ITU-T 的 GDMO 模組、以及 **DMTF 的 CIM Schema**（使用 MOF 語法）在分類上都屬於資料模型（DM），而非資訊模型（IM）[^rfc3444]。

不過，RFC 3444 也特別指出一個例外：**CIM Schema 因為支援 UML 圖形化表示法，可說比 IETF MIB 模組更接近資訊模型**。這點正是 CIM 設計哲學的關鍵所在——它誕生之初就意識到，一個好的管理模型必須同時服務兩個層次。

## DMTF 的轉型與 CIM 的誕生（1999）

DMTF 在 1999 年做出了一個重大決定：將組織名稱從 **Desktop** Management Task Force 改為 **Distributed** Management Task Force[^dmtf-wiki]。名稱的改變反映了組織野心——從桌上型電腦的管理，擴展到分散式 IT 環境的所有元素。同一年，**CIM v1.0 正式發布**[^cim-wiki]（在 2009 年版本的系列大綱中曾稱為 1997 年，但根據 Wikipedia 與 RFC 3444 中引用的 CIM v2.2 早在 1999 年 6 月即已存在的證據，1999 年為正確年份[^rfc3444]）。

CIM 的核心理念可以用一句話總結：

> **不只定義「資料長什麼樣子」，更要定義「管理概念之間的關係」。**

這是 SNMP MIB 模型中完全缺乏的維度。CIM 使用物件導向的三層結構來回應 1990 年代的三大困境[^dmtf-cim]：

1. **Core Model（核心模型）**：提供最抽象的基礎類別（如 `CIM_ManagedElement`），解決沒有共同起點的問題
2. **Common Model（共通模型）**：定義標準 IT 元素（`CIM_ComputerSystem`、`CIM_OperatingSystem`、`CIM_Process`），解決跨廠商命名不一致的問題
3. **Extension Schema（擴充綱要）**：允許廠商衍生產品的專屬類別，同時維持與標準模型的可互通性

## 小結：從戰國到統一

1990 年代的管理標準生態是一個典型的「戰國時代」——SNMP 在網路層面稱霸，CMIP 在電信領域屹立不搖，DMI 在桌上型電腦站穩腳步，但它們之間缺乏共同的語言。

CIM 並非這些標準的取代者，而是它們的**共同基礎**。它定義了管理資訊的「通用語法」，使得：

- SNMP 的 MIB 可以從 CIM Schema 自動衍生（透過 CIM-to-SNMP 對映）
- CMIP 的 GDMO 定義可以與 CIM 類別建立對應關係
- WBEM 三協定（CIM-XML、WS-Management、CIM-RS）則提供了 CIM 的實際通訊層

這個願景在 1999 年 CIM v1.0 中首次實現，並經過 27 年持續演進至今。

明天（Day 3）我們將深入探討 CIM 的核心設計哲學——它的物件導向模型如何運作，以及三層架構（Core → Common → Extension）如何實現互通性與擴充性之間的平衡。

## 參考資料

[^snmp-rfc]: IETF. (1990). RFC 1157 — A Simple Network Management Protocol (SNMP). Retrieved 2026-10-03, from https://datatracker.ietf.org/doc/html/rfc1157
[^snmp-wiki]: Wikipedia. (2026). Simple Network Management Protocol. Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Simple_Network_Management_Protocol
[^cmip-wiki]: Wikipedia. (2026). Common Management Information Protocol. Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Common_Management_Information_Protocol
[^tmn-wiki]: Wikipedia. (2026). Telecommunications Management Network. Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Telecommunications_Management_Network
[^dmi-wiki]: Wikipedia. (2025). Desktop Management Interface. Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Desktop_Management_Interface
[^dmtf-wiki]: Wikipedia. (2026). Distributed Management Task Force. Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Distributed_Management_Task_Force
[^cim-wiki]: Wikipedia. (2025). Common Information Model (computing). Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Common_Information_Model_(computing)
[^rfc3444]: Pras, A. & Schoenwaelder, J. (2003). RFC 3444 — On the Difference between Information Models and Data Models. IETF. Retrieved 2026-10-03, from https://datatracker.ietf.org/doc/html/rfc3444
[^dmtf-cim]: DMTF. (n.d.). CIM — Common Information Model. Retrieved 2026-10-03, from https://www.dmtf.org/standards/cim