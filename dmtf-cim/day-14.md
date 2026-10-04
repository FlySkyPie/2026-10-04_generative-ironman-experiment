# Day 14：MOF 實戰 — 讀懂真實 Schema 檔案

昨天我們詳細解析了 Managed Object Format（MOF）的完整語法，包括 `class`、`property`、`method`、`association` 等核心結構，以及 Qualifier 的宣告與使用方式。今天我們不再討論語法理論，而是**直接打開真實的 CIM Schema 檔案**，親手閱讀與解析 DMTF 官方發佈的 MOF 原始碼——因為理解 CIM 的最佳方式，就是直接閱讀它的 Schema 本身。

## 下載 CIM 2.56.0 MOF 並導覽目錄結構

截至 2026 年，CIM Schema 的最新穩定版本為 **v2.56.0**（2026 年 1 月 21 日發佈）[^dmtf-256-mofs]。你可以從以下位置下載：

| 下載 | 網址 | 大小 |
|------|------|------|
| **Final MOF** （穩定版） | `https://www.dmtf.org/sites/default/files/cim/cim_schema_v2560/cim_schema_2.56.0Final-MOFs.zip` | ~1.6 MB |
| **Experimental MOF** （實驗版） | `https://www.dmtf.org/sites/default/files/cim/cim_schema_v2560/cim_schema_2.56.0Experimental-MOFs.zip` | ~2.0 MB |
| **HTML 文件**（瀏覽用） | `http://schemas.dmtf.org/wbem/cim-html/2.55.0/` | 無需下載 |

下載並解壓縮 `Final-MOFs.zip` 後，會看到以下目錄結構[^dmtf-256-mofs]：

```
cim_schema_2.56.0Final-MOFs/
├── Application.mof
├── Application2.mof
├── AssetReport.mof
├── Business.mof
├── Campagne.mof
├── Capacity.mof
├── Collection.mof
├── Common.mof
├── Control.mof
├── Database.mof
├── Device.mof
├── DPCB.mof
├── Entity.mof
├── Event.mof
├── Landscape.mof
├── Material.mof
├── Measure.mof
├── Memory.mof
├── Network.mof
├── Physical.mof
├── Policy.mof
├── ProductInfo.mof
├── RecordUse.mof
├── Security.mof
├── Service.mof
├── Software.mof
├── SRM.mof
├── Storage.mof
├── System.mof
├── TokenRing.mof
├── User.mof
├── Video.mof
├── VMAN.mof
└── Wireline.mof
```

每個 `.mof` 檔案對應一個 **Schema 領域**（domain）。例如 `System.mof` 涵蓋電腦系統、作業系統、程序等類別；`Network.mof` 涵蓋網路介面與協定端點；`Device.mof` 涵蓋處理器、記憶體、儲存裝置等硬體元件。

> 注意：這些 MOF 檔案本身並不互相 `include` 或 `import`。CIM 採用的是**扁平式 Schema 設計**——所有 MOF 檔案在概念上屬於同一個全域命名空間。類別之間的跨檔案引用（如 `CIM_ComputerSystem` 定義在 `System.mof` 中，而被 `Device.mof` 中的關聯引用）是合法的，無需任何匯入宣告[^dmtf-cim-page]。

---

## 分析 System.mof 的真實片段

讓我們打開 **`System.mof`**，這是整個 CIM Schema 中最核心的檔案之一。以下是其前段的 Qualifier 宣告區[^microsoft-omi-system]：

```mof
// System.mof -- generated from the DMTF CIM V2.32.0 Schema
// Copyright (c) 2005 DMTF.  All rights reserved.

PRAGMA (MOF, "1.5");

    //  ===========================
    //  QUALIFIER DECLARATIONS
    //  ===========================

qualifier UMLPackagePath : string,
    scope (schema, class, association, indication) ,
    flavor (tosubclass, enableoverride, translatable) ;

qualifier MappingStrings : string,
    scope (property, method, parameter, indication) ,
    flavor (tosubclass, enableoverride, translatable) ;

qualifier ModelCorrespondence : string,
    scope (property, method, parameter, indication) ,
    flavor (tosubclass, enableoverride, translatable) ;

qualifier Propagated : string,
    scope (property) ,
    flavor (tosubclass, enableoverride, translatable) ;
```

**關鍵觀察**：

1. **`PRAGMA (MOF, "1.5")`** — 告訴編譯器此檔案遵循 MOF 語法 1.5 版
2. **Qualifier 必須在使用前宣告** — 這裡宣告了四個在 System.mof 中頻繁使用的 Qualifier
3. **`scope (schema, class, association, indication)`** — 表明 `UMLPackagePath` 可套用於 schema、class、association 和 indication 四種 meta-schema 元素
4. **`flavor` 中的 `tosubclass` 與 `enableoverride`** — 這些 Qualifier 會被子類別繼承且可被子類別覆寫

---

## `CIM_ComputerSystem` 的完整 MOF 定義

`CIM_ComputerSystem` 是 CIM Schema 中最重要、最複雜的類別之一。以下為其在 System.mof 中的真實定義[^microsoft-omi-computersystem]：

```mof
// System.mof 中的 CIM_ComputerSystem 定義

   [Version ( "2.28.0" ),
    UMLPackagePath ( "CIM::System::SystemElements" ),
    Description (
       "A class derived from System that is a special collection of "
       "ManagedSystemElements. This collection is related to the "
       "providing of compute capabilities and MAY serve as an "
       "aggregation point to associate one or more of the following "
       "elements: FileSystem, OperatingSystem, Processor and Memory "
       "(Volatile and/or NonVolatile Storage)." )]
class CIM_ComputerSystem : CIM_System {

   // ====== Key Properties ======

   [Override ( "NameFormat" ),
    Description (
       "The ComputerSystem object and its derivatives are Top "
       "Level Objects of CIM. They provide the scope for "
       "numerous components. Having unique System keys is "
       "required. The NameFormat property identifies how the "
       "ComputerSystem Name is generated." ),
    ValueMap { "Other", "IP", "Dial", "HID", "NWA", "HWA", "X25",
       "ISDN", "IPX", "DCC", "ICD", "E.164", "SNA", "OID/OSI",
       "WWN", "NAA", "UUID" }]
   string NameFormat;

   // ====== Enumerated Properties ======

   [Description (
       "Enumeration indicating the purpose(s) to which the "
       "ComputerSystem is dedicated. A value of 0 (Not Dedicated) "
       "indicates it is not dedicated to any purpose." ),
    ValueMap { "0", "1", "2", "3", "4", "5", "6", "7", "8", "9",
       "10", "11", "12", "13", "14", "15", "16", "17", "18",
       "19", "20", "21", "22", "23", "24", "25", "26", "27",
       "28", "29", "30", "31", "32", "33", "34", "35", "36",
       "37", "38", "..", "32568..65535" },
    Values { "Not Dedicated", "Unknown", "Other", "Storage",
       "Router", "Switch", "Layer 3 Switch",
       "Central Office Switch", "Hub", "Access Server",
       "Firewall", "Print", "I/O", "Web Caching", "Management",
       "Block Server", "File Server", "Mobile User Device",
       "Mobile Phone", "Video", "Game Console" },
    ArrayType ( "Indexed" ),
    MappingStrings { "MIB.IETF|MIB-II.sysServices",
       "FC-GS.INCITS-T11 | Platform | PlatformType" },
    ModelCorrespondence {
       "CIM_ComputerSystem.OtherDedicatedDescriptions" }]
   uint16 Dedicated[];

   // ====== Method ======

   [Deprecated { "CIM_PowerManagementService.SetPowerState" },
    Description (
       "Sets the power state of the computer. This method is "
       "deprecated in favor of PowerManagementService." )]
   uint32 SetPowerState(
         [IN,
          Description ( "The desired state for the ComputerSystem." ),
          ValueMap { "1", "2", "3", "4", "5", "6", "7", "8" },
          Values { "Full Power", "Power Save - Low Power Mode",
             "Power Save - Standby", "Power Save - Other",
             "Power Cycle" , "Power Off", "Hibernate", "Soft Off" }]
      uint32 PowerState,
         [IN, Description (
             "Time indicates when the power state should be set." )]
      datetime Time);

};
```

### 值得注意的設計細節

1. **`Version` 與 `UMLPackagePath`** — 每個主要類別都有版本追蹤與 UML 套件路徑，讓 Schema 管理人員可以清楚識別每個類別的歸屬與版本
2. **`Override` 的使用** — `NameFormat` 使用了 `[Override ( "NameFormat" )]`，表示它繼承自父類別（`CIM_System`）的同一屬性，此處重新定義了其語意與列舉值
3. **`ValueMap` + `Values` 模式** — 屬性實際上儲存的是 `uint16` 數值，但 `Values` 提供了人類可讀的文字對映；`ArrayType ( "Indexed" )` 表示這是一個索引式陣列，可同時選取多個值
4. **`MappingStrings`** — 提供了對其他管理標準（如 MIB-II、FC-GS）的對映路徑，實現跨標準互通
5. **`Deprecated` 方法** — `SetPowerState` 被標記為廢棄，改由 `CIM_PowerManagementService` 取代，體現 Schema 的優雅演化策略

---

## 從 `CIM_ComputerSystem` 到 `CIM_Process` 的關聯鏈

在 Day 12 的 Q&A 中，我們已觸及「ComputerSystem 如何獲知其上的 Process」這個問題。答案是：**二者之間並無直接的 Association**。實際上是透過 **三層的關聯與 Key Propagation 機制** 來實現的。

### 第一層：ComputerSystem ↔ OperatingSystem

`CIM_InstalledOS` 是一個 Association class，連結 `CIM_ComputerSystem` 與 `CIM_OperatingSystem`[^microsoft-omi-installedos]：

```mof
[Association, Aggregation, Composition, Version ( "2.7.0" ),
 UMLPackagePath ( "CIM::System::OperatingSystem" ),
 Description (
    "A link between the ComputerSystem and the OperatingSystem(s) "
    "installed or loaded on it." )]
class CIM_InstalledOS : CIM_SystemComponent {

   [Aggregate, Override ( "GroupComponent" ),
    Min ( 1 ), Max ( 1 ),
    Description ( "The ComputerSystem." )]
   CIM_ComputerSystem REF GroupComponent;

   [Override ( "PartComponent" ), Weak,
    Description ( "The OperatingSystem installed on the ComputerSystem." )]
   CIM_OperatingSystem REF PartComponent;

   [Description (
       "Boolean indicating that the OperatingSystem is the "
       "default OS for the ComputerSystem." ),
    MappingStrings { "MIF.DMTF|Operating System|001.4" }]
   boolean PrimaryOS;
};
```

關鍵點：
- `Min ( 1 ), Max ( 1 )` 表示一個 ComputerSystem **必然** 關聯到至少一個 OS（而且最多一個作為 Group）
- `Weak` 表示 OS 的生命週期依賴於其所屬的 ComputerSystem
- 這是一個 **Composition**（複合）關聯——移除 ComputerSystem 時，其所屬的 OS 也將被移除

### 第二層：OperatingSystem ↔ Process 的 Key Propagation

`CIM_Process` 與 `CIM_OperatingSystem` 之間**同樣沒有直接的 Association**。其關係是透過 **6 個 Key Properties** 中的 4 個 Propagated Properties 來實現的[^microsoft-omi-process]：

```mof
class CIM_Process : CIM_EnabledLogicalElement {

   // --- Propagated from CIM_ComputerSystem (via OS) ---
   [Key, Description ( "The scoping ComputerSystem's CreationClassName." ),
    MaxLen ( 256 ),
    Propagated ( "CIM_OperatingSystem.CSCreationClassName" )]
   string CSCreationClassName;

   [Key, Description ( "The scoping ComputerSystem's Name." ),
    MaxLen ( 256 ),
    Propagated ( "CIM_OperatingSystem.CSName" )]
   string CSName;

   // --- Propagated from CIM_OperatingSystem ---
   [Key, Description ( "The scoping OperatingSystem's CreationClassName." ),
    MaxLen ( 256 ),
    Propagated ( "CIM_OperatingSystem.CreationClassName" )]
   string OSCreationClassName;

   [Key, Description ( "The scoping OperatingSystem's Name." ),
    MaxLen ( 256 ),
    Propagated ( "CIM_OperatingSystem.Name" )]
   string OSName;

   // --- Local Key Properties ---
   [Key, Description (
       "CreationClassName indicates the name of the class or the "
       "subclass used in the creation of an instance." ),
    MaxLen ( 256 )]
   string CreationClassName;

   [Key, Description (
       "A string used to identify the Process. A Process ID is a "
       "kind of Process Handle." ),
    MaxLen ( 256 ),
    MappingStrings { "MIF.DMTF|Process Information|001.1" }]
   string Handle;

   // ... 其他屬性（ExecutionState、CreationDate、KernelModeTime 等）...
};
```

### Propagation 的完整流程

```
CIM_ComputerSystem                  CIM_OperatingSystem                CIM_Process
┌──────────────────────┐           ┌──────────────────────┐           ┌──────────────────────┐
│ CreationClassName    │──prop──▶  │ CSCreationClassName  │──prop──▶  │ CSCreationClassName  │
│ Name                 │──prop──▶  │ CSName               │──prop──▶  │ CSName               │
│                      │           │ CreationClassName    │──prop──▶  │ OSCreationClassName  │
│                      │           │ Name                 │──prop──▶  │ OSName               │
└──────────────────────┘           └──────────────────────┘           │ CreationClassName    │
                                                                      │ Handle (Process ID)  │
                                                                      └──────────────────────┘
```

即使沒有明確的 Association 關聯，只要 `CIM_Process` 的 `CSCreationClassName` + `CSName` 與某個 `CIM_ComputerSystem` 的 `CreationClassName` + `Name` 匹配，就可以在查詢時建立起隱含的關聯路徑。

這種設計稱為 **Key Propagation**（金鑰傳播），是 CIM 在保持靈活性的同時確保資料完整性的核心技術——它讓不同 Schema 領域中的類別無需顯式關聯即可相互連結，同時又提供了足夠的金鑰資訊來還原這些連結的語意。

---

## 實戰：閱讀一份 MOF 檔案的檢查清單

當你第一次打開一個陌生的 `.mof` 檔案時，可以按以下順序閱讀，逐步建立理解：

1. **看檔案頭** — 確認 `PRAGMA` 版本（1.4 或 1.5）與版權資訊
2. **看 Qualifier 宣告** — 了解此檔案中使用了哪些自訂 Qualifier
3. **掃描 `class` 宣告** — 列出所有在此檔案中定義的主要類別
4. **檢查繼承層次** — 每個 `class X : Y` 中的冒號後的父類別
5. **尋找 Association** — 標記了 `[Association]` 的類別，理解其 `REF` 指向
6. **關注 Key Properties** — 標記了 `[Key]` 的屬性，它們建構了類別實例的識別
7. **留意 Propagated** — 標記了 `[Propagated]` 的屬性，它們指向了金鑰的來源
8. **查看 ValueMap/Values** — 確認列舉型屬性的合法值範圍
9. **尋找 Deprecated** — 了解哪些部分被標記為廢棄，與取代者的關聯
10. **對照 MOF → HTML** — 在 `http://schemas.dmtf.org/wbem/cim-html/2.55.0/` 上以 HTML 格式瀏覽同一類別，可獲得更友善的閱讀體驗

---

## 小結

今天我們親手打開了 DMTF 官方 Schema 的原始 MOF 檔案，從下載目錄結構、到剖析 `System.mof` 的 Qualifier 宣告、再到完整解讀 `CIM_ComputerSystem` 的定義，最後一路追蹤到 `CIM_Process` 的 Key Propagation 路徑。最關鍵的收穫是：**Association 不是 CIM 中唯一建立關聯的方式**——Key Propagation 透過在類別間傳播金鑰屬性，實現了不同領域之間的隱含關聯，讓 Schema 更加靈活且易於擴展。

明天 Day 15，我們將探討 **MOF Compiler**——如何將這些 `.mof` 檔案編譯成真實可操作的物件，包括 OpenPegasus 的 `mofcomp`、pywbem 的 `mof_compiler`，以及 Windows 的 `mofcomp.exe`。

[^dmtf-256-mofs]: DMTF. (2026). CIM Schema 2.56.0 Downloads. Retrieved 2026-10-03, from https://www.dmtf.org/standards/cim/cim_schema_v2560
[^dmtf-cim-page]: DMTF. (n.d.). DMTF CIM Standards. Retrieved 2026-10-03, from https://www.dmtf.org/standards/cim
[^microsoft-omi-system]: Microsoft. (n.d.). OMI Project — System.mof (CIM 2.32.0). Retrieved 2026-10-03, from https://raw.githubusercontent.com/microsoft/omi/master/Unix/share/omischema/CIM-2.32.0/System/CIM_System.mof
[^microsoft-omi-computersystem]: Microsoft. (n.d.). OMI Project — CIM_ComputerSystem.mof (CIM 2.32.0). Retrieved 2026-10-03, from https://raw.githubusercontent.com/microsoft/omi/master/Unix/share/omischema/CIM-2.32.0/System/CIM_ComputerSystem.mof
[^microsoft-omi-installedos]: Microsoft. (n.d.). OMI Project — CIM_InstalledOS.mof (CIM 2.32.0). Retrieved 2026-10-03, from https://raw.githubusercontent.com/microsoft/omi/master/Unix/share/omischema/CIM-2.32.0/System/CIM_InstalledOS.mof
[^microsoft-omi-process]: Microsoft. (n.d.). OMI Project — CIM_Process.mof (CIM 2.32.0). Retrieved 2026-10-03, from https://raw.githubusercontent.com/microsoft/omi/master/Unix/share/omischema/CIM-2.32.0/System/CIM_Process.mof