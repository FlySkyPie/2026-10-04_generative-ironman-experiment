# Day 12：Metamodel 概論 — 什麼是「模型的模型」？

昨天我們完成第二階段（Day 7–11）的最後一站，以跨領域整合的方式看到了一台伺服器的完整 CIM 描述——從 `CIM_ComputerSystem` 到 `CIM_NetworkInterface`、`CIM_OperatingSystem`，再到 `CIM_Service` 與 `CIM_MetricService`。從今天開始，我們展開為期 5 天的 **第三階段：Metamodel 與 MOF**，從更高的抽象層次來理解 CIM 的「模型的模型」——也就是 CIM Metamodel。

## 什麼是「Metamodel」？

Metamodel（元模型）這個詞的字面意義是「關於模型的模型」。它與一般模型的關係，就像文法規則與用該文法寫出的句子之間的關係[^wiki-metamodel]：

| 層次 | 例子 | 說明 |
|------|------|------|
| **資料** | `"prod-web-03"`、`"00:1A:2B:3C:4D:5E"` | 具體的管理物件實例值 |
| **模型（Model）** | `CIM_ComputerSystem`、`CIM_LANEndpoint` | CIM Schema 中定義的類別、屬性、關聯 |
| **元模型（Metamodel）** | `class`、`property`、`association`、`qualifier` | 定義「類別」、「屬性」、「關聯」等概念的語言本身 |

用程式語言的類比來說：如果 CIM Schema 是 Java 或 Python 中的一個類別庫（class library），那 CIM Metamodel 就是 Java 或 Python 這門程式語言本身的語法規則與型別系統[^dmtf-cim-page]。

CIM Metamodel 的官方文件編號為 **DSP0004**，當前版本為 **3.0.1**（2014 年 8 月發佈）[^dsp0004-301]。DMTF 對其定位的定義如下：

> *「CIM Metamodel 定義了建構新的相容模型（conformant models）以及表示這些模型的 Schema 所需的語意。建模需求與環境經常不同且隨著時間變化，因此元模型進一步透過 Qualifier 機制來擴展其元素。」* [^dmtf-cim-page]

換句話說，CIM Metamodel 是 CIM 這個管理模型世界的「語法書」——它定義了什麼是一個合法的 CIM 類別、合法屬性有哪些約束、什麼樣的關聯是有效的。

---

## 五大 Meta-Schema 元素

CIM Metamodel 定義了五種核心的 meta-schema 元素（亦稱為「陳述型別／statement types」），它們是建構所有 CIM Schema 內容的基礎積木[^solaris-cim-terms][^cim-grammar]：

### 1. Class（類別）

Class 是 CIM 中最基本的結構元素，代表一個可管理的物件類型。每個 CIM Class 有以下特徵[^solaris-cim-terms]：

- **可擁有屬性（Properties）與方法（Methods）**
- **可繼承自另一類別**（單一繼承，Single Inheritance）
- **可被標記為特定種類**（如 Association、Indication），透過 Qualifier 來標示

```mof
// 一個典型的 CIM Class 定義語法
[Version ("2.0.0"),
 Description ("這是一個範例類別")]
class CIM_ExampleClass : CIM_ManagedElement {
    string ExampleProperty;
    uint32 ExampleMethod(uint16 Parameter);
};
```

`class` 關鍵字之後依序是類別名稱、選擇性的冒號 + 父類別名稱，以及大括號內部的類別特徵（Class Features）清單。

### 2. Property（屬性）

Property 是依附於一個 Class 的、有名稱且有資料型別的欄位，代表被管理物件的**狀態資訊**[^cim-grammar]：

```mof
uint32 TotalMemory;              // 純量屬性
string Description;              // 字串屬性
uint16 OperationalStatus[];      // 陣列屬性（以 [] 標記）
datetime TimeOfLastStateChange;  // 時間戳屬性
```

CIM 支援的資料型別包括[^cim-grammar]：

| 類別 | 型別 |
|------|------|
| **整數** | `uint8`、`sint8`、`uint16`、`sint16`、`uint32`、`sint32`、`uint64`、`sint64` |
| **浮點數** | `real32`、`real64` |
| **字元與字串** | `char16`、`string` |
| **布林** | `boolean` |
| **時間** | `datetime` |

### 3. Method（Method）

Method 是可對一個 Class 的實例**呼叫的操作**。每個 Method 有回傳型別與零個或多個參數[^cim-grammar]：

```mof
uint32 StartService();                                // 無參數
uint32 Reboot(string Reason, uint16 Timeout);         // 兩個參數
void ShowMetrics(uint16 Subject, uint16 Definition);  // void 回傳
```

Method 的參數有其獨立的型別與名稱，必要時也可以附加 Qualifier。

### 4. Association（關聯）

Association 是 CIM 中表達**類別之間語意關係**的機制。在 CIM Metamodel 中，Association **不是一個獨立的語法結構**——它是一個標記了 `[Association]` Qualifier 的 Class[^cim-grammar][^solaris-cim-terms]：

```mof
[Association, Version ("2.6.0")]
class CIM_BindsToLANEndpoint : CIM_BindsTo {
    [Override ("Antecedent")]
    CIM_LANEndpoint REF Antecedent;

    [Override ("Dependent")]
    CIM_ServiceAccessPoint REF Dependent;

    uint16 FrameType;
};
```

Association 的關鍵設計特徵是：
- 它使用 **Reference（REF）** 而非 Property 來指向參與的類別
- 它**不會改變參與類別的介面**——`CIM_LANEndpoint` 本身不需要知道它被哪些 Association 關聯
- 一個 Association 至少需要兩個 Reference

### 5. Indication（Indication）

Indication 代表一個**事件或通知**——它是一個被管理元素可以發出的非同步訊息。與 Association 類似，Indication 也是透過 Qualifier 來識別的類別[^cim-grammar][^solaris-cim-terms]：

```mof
[Indication, Version ("2.0.0")]
class CIM_AlertIndication : CIM_ManagedElement {
    uint16 AlertID;
    string AlertDescription;
    datetime AlertTime;
    uint16 Severity;
};
```

Indication 同樣可以有 Property、Method 與 Trigger（觸發建立 Indication 實例的系統事件）。它們是 CIM 處理警報、狀態變更等非同步通知的核心機制。

---

## Qualifier 機制 — 為模型元素附加 Metadata

> 有關 Qualifier 的語法與釋例將在明天 Day 13 的 MOF 語法全解析中深入探討，這裡先聚焦於概念層次的理解。

**Qualifier**（限定詞）是 CIM Metamodel 最強大的擴展性機制。它允許將各種 metadata（元資料）附加到類別、屬性、方法、關聯等元素上，而不需要修改 meta-schema 本身[^dmtf-cim-page][^cim-grammar]。

如果把 CIM Class 比喻為一個 HTML 元素（如 `<div>`），Qualifier 就像是 HTML 的屬性（如 `class="foo"`）——它不會改變元素的本質，但為元素附加了重要的語意資訊。

### Qualifier 的宣告語法

在 MOF 文件中，Qualifier 必須**先宣告後使用**，且宣告位於文件的前端：

```mof
qualifier <qualifierName> : <dataType> ,
    scope ( <metaElement> [, <metaElement>]* ) ,
    [ flavor ( <flavor> [, <flavor>]* ) ] ;
```

每個 Qualifier 宣告包含以下組成[^cim-grammar]：

| 元件 | 說明 |
|------|------|
| **Name** | Qualifier 的名稱（如 `Version`、`Description`、`Association`） |
| **Type** | 值的資料型別（`string`、`uint16`、`boolean`…） |
| **Scope** | 可套用的 meta-schema 元素（`class`、`property`、`method`、`parameter`、`schema` 或 `any`） |
| **Flavor** | 傳播規則（選擇性） |

### 五種 Flavor（傳播規則）

Flavor 定義了 Qualifier 如何在繼承鏈中傳播[^cim-grammar]：

| Flavor | 預設值 | 效果 |
|--------|--------|------|
| `enableoverride` | `true` | 子類別可以覆寫此 Qualifier 的值 |
| `disableoverride` | `false` | 子類別**不能**覆寫此 Qualifier 的值 |
| `tosubclass` | `true` | 子類別會繼承此 Qualifier |
| `restricted` | `false` | 此 Qualifier 只作用於當前元素，不傳播到子類別 |
| `translatable` | `false` | Qualifier 的值可以多語言化 |

> **實例**：如果 `Description` Qualifier 的 flavor 設置為 `(enableoverride, tosubclass)`，則 `CIM_ComputerSystem` 上定義的 `Description` 可以被子類別覆寫，但若一個 `CIM_ManagedSystemElement` 的 `[Deprecated]` 被設為 `disableoverride`，則所有子類別都無法取消這個廢棄狀態。

### 三種 Qualifier 類別

| 類別 | 來源 | 範例 |
|------|------|------|
| **Meta** | 由 CIM Metamodel 內建定義 | `Version`、`Description`、`UMLPackagePath` |
| **Standard** | 在 CIM Schema 規格中定義 | `Key`、`Override`、`ValueMap`、`Values`、`MappingStrings`、`Deprecated`、`MaxLen`、`Required` |
| **Optional** | 由使用者／供應商自訂擴展 | 略 |

### 常見的 Standard Qualifier 範例

以下是真實 CIM Schema 中頻繁出現的 Qualifier[^cim-grammar][^cim-computersystem]：

```mof
class CIM_ComputerSystem : CIM_System {
    [Key] string CreationClassName;
    //       ↑ 此屬性是此類別的 Key（主鍵）之一

    [ValueMap {"0", "1", "2", ..., "41"},
     Values {"Not Dedicated", "Unknown", "Other", ..., "Router"}]
    uint16 Dedicated;
    //       ↑ Dedicated 屬性只儲存 uint16 列舉值，
    //         但 Qualifier 提供了人類可讀的名稱對映

    [Deprecated {"CIM_ComputerSystem.OtherDedicatedInfo"}]
    string OtherDedicatedInfo;
    //       ↑ 此屬性已被廢棄，改用 OtherDedicatedInfo
};
```

---

## Metamodel 的版本演進：從 Infrastructure Spec 到 Metamodel

DSP0004 並非一開始就被稱為「Metamodel」。它經歷了一次重要的重新定位。以下為所有已發佈的 DSP0004 版本[^dsp0004-versions]：

| 版本 | 名稱 | 發佈日期 | 主要變化 |
|------|------|---------|---------|
| **2.3.0** | CIM Infrastructure Specification | 2005-10 | 最早的公開版本，定義 CIM 架構與概念 |
| **2.5.0** | CIM Infrastructure Specification | 2009-05 | 中期更新 |
| **2.6.0** | CIM Infrastructure Specification | 2010-03 | 進一步完善 |
| **2.7.0** | CIM Infrastructure Specification | 2012-05 | 最後一版 Infrastructure Specification 命名 |
| **2.8.0** | Common Information Model (CIM) Infrastructure | 2014-08 | 更名為「Infrastructure」，無「Specification」 |
| **3.0.0** | CIM Metamodel | 2013-01 | **重大重新定位**——從「基礎架構規格」重構為「元模型」 |
| **3.0.1** | Common Information Model (CIM) Metamodel | 2014-08 | 當前最新版本，對 v3.0.0 的小幅修正 |

### 從 Infrastructure 到 Metamodel 的跨越

這次重新定位不只是名稱的更動，而是概念層次的根本提升[^dmtf-cim-page][^dsp0004-301]：

**v2.x（基礎架構規格）** 的關注點：
- CIM 的整體架構介紹
- 命名規則與命名空間
- 與其他管理模型的對映方法
- CIM 的部署方式

**v3.x（元模型）** 的關注點：
- 明確定義建構合法 CIM 模型的「語言」本身
- 正式化 meta-schema 元素（Class、Property、Method、Association、Indication）
- 健全 Qualifier 的型別系統與傳播規則
- 強調「模型之模型」的抽象層次

有趣的是，v2.8.0（2014-08）的發佈日期**晚於** v3.0.0（2013-01），這意味著 DMTF 在發佈大規模重構的 v3.0.0 後，又為 v2.x 分支做了一次維護性更新（v2.8.0），然後才以 v3.0.1 作為統一的最新版本。這與許多標準化組織在過渡期維護雙軌的策略一致。

### 為何需要 Metamodel？

你可能會問：「CIM Schema 不是已經定義了所有管理物件嗎？為什麼還需要一個 Metamodel？」

答案在於 CIM 的**可擴展性（Extensibility）**。想像你是某家儲存陣列供應商的工程師。CIM Schema 已經定義了 `CIM_StorageDrive` 與 `CIM_BlockStorageService`，但你的產品有一個創新的快取技術，需要在 `CIM_StorageDrive` 上新增一些管理屬性。你不能修改 DMTF 的標準 Schema——但你可以建立自己的 Extension Schema，而 Metamodel 保證了你的擴展是「合法」的 CIM，可以被標準的 CIM Object Manager 理解。

此外，Metamodel 還提供了將 CIM 對映到其他管理模型的標準方法——例如將 CIM Class 對映到 SNMP MIB 的 `OBJECT-TYPE`，或對映到 DMTF MIF。這一切都需要一個形式化的語言定義作為橋樑。

### 一個重要的澄清

在 CIM 整體標準體系中，CIM Metamodel（DSP0004）只是三大部分之一[^dmtf-cim-page]：

| 元件 | 內容 | 版本 |
|------|------|------|
| **CIM Metamodel（DSP0004）** | 建構模型的語言（meta-schema、Qualifier、MOF 語法） | 3.0.1 |
| **CIM Specification** | 各管理 Profile 與對映標準的集合 | 多個 DSP |
| **CIM Schema** | 實際的類別定義（Core + Common + Extension） | 2.56.0（2026-01） |

三者的關係類似於：**Metamodel** 規定了 CIM 文件的語法規則，**CIM Specification** 提供了各領域的管理需求檔案，而 **CIM Schema** 則是我們這個系列前面 11 天一直在深入探討的實際類別定義。Schema 是用 Metamodel 這門語言寫出來的。

---

## 小結

今天我們建立了 CIM Metamodel 的概念框架：

- **Metamodel 是「模型的模型」**，它定義了 CIM Schema 的建構語言，而非管理物件本身
- 五大 **meta-schema 元素**（Class、Property、Method、Association、Indication）構成了所有 CIM Schema 的基礎積木
- **Qualifier 機制**提供了強大的 metadata 附加能力，允許 Schema 在不修改 Metamodel 的前提下持續擴展
- DSP0004 從 v2.x 的 **Infrastructure Specification** 演進為 v3.x 的 **Metamodel**，反映了 DMTF 對形式模型語言的重視
- Metamodel、CIM Specification 與 CIM Schema 構成了 CIM 標準的三層體系

明天（Day 13）我們將在此基礎上，深入探討 **MOF（Managed Object Format）語法**——也就是 Metamodel 定義的具體語言如何書寫——並從零開始建立一個 `CIM_ManagedElement` 的子類別。

## 參考資料

[^dmtf-cim-page]: DMTF. (n.d.). CIM Standards Overview. Retrieved 2026-10-03, from https://www.dmtf.org/standards/cim

[^dsp0004-301]: DMTF. (2014). Common Information Model (CIM) Metamodel — DSP0004 v3.0.1. Retrieved 2026-10-03, from https://www.dmtf.org/dsp/DSP0004

[^dsp0004-versions]: DMTF. (n.d.). DSP0004 All Published Versions. Retrieved 2026-10-03, from https://www.dmtf.org/dsp/DSP0004

[^cim-grammar]: DMTF. (n.d.). CIM v2.2.2 Grammar Definition (cim22Grammar.g). Retrieved 2026-10-03, from https://raw.githubusercontent.com/slebok/zoo/master/zoo/meta/mof/cim/assis/fetched/cim22Grammar.g

[^wiki-metamodel]: Wikipedia. (n.d.). Metamodeling. Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Metamodeling

[^solaris-cim-terms]: Oracle. (n.d.). Solaris WBEM Services Administrator's Guide — Appendix A: CIM Terms and Concepts. Retrieved 2026-10-03, from https://docs.oracle.com/cd/E19455-01/806-6468/6jfdjss9h/index.html

[^cim-computersystem]: DMTF. (n.d.). CIM_ComputerSystem Class (v2.55.0). Retrieved 2026-10-03, from https://schemas.dmtf.org/wbem/cim-html/2.55.0/CIM_ComputerSystem.html