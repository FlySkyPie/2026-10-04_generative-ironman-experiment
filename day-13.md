# Day 13：Managed Object Format (MOF) 語法全解析

昨天我們建立了 CIM Metamodel 的概念框架——理解了五大 meta-schema 元素（Class、Property、Method、Association、Indication）以及 Qualifier 機制的設計哲學。如果說昨天是 Metamodel 的「**概念層**」，那麼今天就是它的「**語法層**」——我們將深入探討 **Managed Object Format (MOF)**，這是 CIM 用來實際書寫 Schema 的具體語言。

## MOF 的定位

MOF 是 CIM 的**正式語法（Concrete Syntax）**，定義於 DMTF 文件 **DSP0221 v3.0.0**（2012 年 12 月發佈）[^dsp0221-300]。它的定位可以從以下三種視角來理解：

| 視角 | 說明 |
|------|------|
| **對 CIM Metamodel 而言** | MOF 是 Metamodel 定義的「抽象語法（Abstract Syntax）」的**具體文字表現形式**[^dmtf-cim-page] |
| **對 Schema 開發者而言** | MOF 是書寫 CIM Class、Property、Association 的 **IDL（Interface Definition Language）風格語言** [^dsp0221-300] |
| **對工具而言** | MOF 是 **MOF Compiler**（如 OpenPegasus 的 `mofcomp`）的輸入格式，可編譯為 WMI Repository、Python 綁定或其他輸出 |

白話來說：**Metamodel 定義了「可以有什麼」，而 MOF 定義了「要怎麼寫」**。

MOF 的語法繼承了許多來自 **IDL（介面定義語言）** 與 **C++** 的風格特徵——以分號終止陳述句、以大括號分組、以雙斜線標記註解。這並非巧合：在 CIM 早期（1990 年代末），IDL 是描述分散式物件介面的主流語言，而 CIM 的設計者選擇了一種 IDL-like 的語法來降低學習曲線。

---

## MOF 檔案的基本結構

一個 MOF 檔案（`.mof`）由**一個或多個頂層產出（mofProduction）** 組成，它們按以下順序出現[^cim-grammar]：

```ebnf
mofSpecification ::= mofProduction+
mofProduction    ::= compilerDirective
                   | qualifierDcl
                   | classDcl
                   | instanceDcl
                   | '[' qualifierList (classDcl | instanceDcl)
```

典型的 MOF 檔案從上到下包含以下章節：

```
// 1. 命名空間宣告（選擇性）
instance of __namespace {
    name = "domainname";
};

// 2. Compiler Directive（#pragma 指令）
#pragma namespace("\\.root\\domain")
#pragma locale ("en_US")

// 3. Qualifier 宣告（必須在使用前宣告）
Qualifier Description : ToSubClass Amended;
Qualifier Values : ToSubClass Amended;

// 4. 類別定義（主要內容）
[Abstract, Version("2.8.1000"), Description("...")]
class CIM_ExampleClass : CIM_ManagedElement {
    [Key, Description("...")]
    string PropertyName;
};

// 5. Association 定義（以 Class 定義 + [Association] 實現）
[Association, Version("2.6.0")]
class CIM_ExampleAssociation : CIM_SomeBase {
    class CIM_ClassA REF RoleA;
    class CIM_ClassB REF RoleB;
};
```

---

## 註解

MOF 支援兩種註解風格，與 C++/Java 完全相同[^cim-grammar]：

```mof
// 這是單行註解，直到行末

/* 這是區塊註解，
   可以跨越多行，
   也可以嵌入單行中 */
```

註解在語法分析時被詞法分析器完全跳過（skip），不影響語意。

---

## Compiler Directive（#pragma 指令）

`#pragma` 是 MOF 的**編譯器指令**，出現在檔案前端，用來控制 MOF Compiler 的行為。根據 CIM v2.2.2 的 MOF 語法規則，共有 8 種 `#pragma` 指令[^cim-grammar]：

| 指令 | 語法 | 說明 |
|------|------|------|
| `#pragma include` | `#pragma include("filename")` | 在該位置插入另一個檔案的內容（文字包含） |
| `#pragma namespace` | `#pragma namespace("path")` | 設定命名空間路徑，後續宣告都屬於此命名空間 |
| `#pragma locale` | `#pragma locale("ll_cc")` | 宣告此 MOF 檔案的自然語言區域（預設 `"en_US"`） |
| `#pragma instancelocale` | `#pragma instancelocale("ll_cc")` | 宣告實例（instance）使用的區域 |
| `#pragma nonlocal` | `#pragma nonlocal("type://handle")` | 指出實例的實際所在位置（用於跨命名空間引用） |
| `#pragma nonlocaltype` | `#pragma nonlocaltype("handle")` | 指出位置的型別 |
| `#pragma source` | `#pragma source("type://handle")` | 指出原始資料來源位置 |
| `#pragma sourcetype` | `#pragma sourcetype("handle")` | 指出原始資料來源的型別 |

在實際的 CIM Schema 檔案中，最常見的是前三者。以下是 CIM Schema 2.56.0 中 `CIM_System.mof` 的實際片段[^cim-system-mof]：

```mof
#pragma namespace("\\\\.root\\\\CIM\\\\Core\\\\System")
#pragma locale("en_US")

instance of __Namespace {
    Name = "System";
};
```

注意：命名空間路徑使用雙反斜線 `\\` 作為分隔符，根節點為 `\\.root`，例如 `\\.root\CIM\Core\System` 表示 CIM → Core → System 這個套件路徑。

---

## 資料型別系統

MOF 定義了 **14 種純量資料型別**[^cim-grammar]：

| 類別 | 型別 | 位元寬 | 說明 |
|------|------|--------|------|
| **無號整數** | `uint8` | 8 位元 | 0～255 |
| | `uint16` | 16 位元 | 0～65,535 |
| | `uint32` | 32 位元 | 0～4,294,967,295 |
| | `uint64` | 64 位元 | 0～2^64-1 |
| **有號整數** | `sint8` | 8 位元 | -128～127 |
| | `sint16` | 16 位元 | -32,768～32,767 |
| | `sint32` | 32 位元 | -2^31～2^31-1 |
| | `sint64` | 64 位元 | -2^63～2^63-1 |
| **浮點數** | `real32` | 32 位元 | IEEE 754 單精度 |
| | `real64` | 64 位元 | IEEE 754 雙精度 |
| **字元與字串** | `char16` | 16 位元 | Unicode 字元 |
| | `string` | 可變 | Unicode 字串 |
| **布林** | `boolean` | 1 位元 | `true` / `false` |
| **時間** | `datetime` | 64 位元 | 日期/時間值 |

此外，還有兩種特殊型別：

- **物件參考（Object Reference）**：語法為 `<ClassName> REF`，用於 Association 中指向參與的類別。例如 `CIM_System REF Antecedent;`。
- **陣列（Array）**：在型別名稱後加上 `[size]` 或 `[]` 表示可重複屬性。例如 `uint16 OperationalStatus[];`。

數值常數可寫為十進位、十六進位（`0x...`）或二進位（`...b`），皆可帶符號。

---

## Class 宣告語法

Class 是 MOF 中最核心的結構元素。其 EBNF 語法為[^cim-grammar]：

```ebnf
classDcl ::= 'class' className alias? superClass? '{' classFeature* '}' ';'
```

其中：
- **className**：以字母或底線開頭、後接字母/底線/數字的識別碼（**大小寫不敏感**，但書寫時的大小寫會保留）
- **alias**（選擇性）：`$IDENTIFIER` 格式的別名
- **superClass**（選擇性）：`:` + 父類別名稱，表示單一繼承
- **classFeature**：類別內的特徵（Property、Method、Reference）

### 沒有屬性的標記類別

最簡單的類別形式——沒有自己屬性的標記類別（Marker Class）：

```mof
[CIMVersion ("2.7.0"), Description ("一個資料庫系統標記類別")]
class CIM_DatabaseSystem : CIM_ApplicationSystem {
};
```

`CIM_DatabaseSystem` 沒有任何本地屬性——所有屬性皆繼承自 `CIM_ApplicationSystem`，它僅作為一個**標記節點**存在於類別層次中。

### 含屬性的典型類別

以下是 `CIM_Service` 在 MOF 中的實際宣告（簡化）[^cim-service-mof]：

```mof
[Abstract, Version ("2.45.0"),
 UMLPackagePath ("CIM::Core::Service"),
 Description ("Service 是一個代表可管理功能可用性的 LogicalElement")]
class CIM_Service : CIM_EnabledLogicalElement {

    [Key, Description ("建立此實例所用的類別名稱"),
     MaxLen (256)]
    string CreationClassName;

    [Key, Override ("Name"),
     Description ("作為企業環境中的唯一識別名稱"),
     MaxLen (256)]
    string Name;

    [Write, Description ("主要擁有者名稱")]
    string PrimaryOwnerName;

    [Description ("目前是否已啟動")]
    boolean Started;

    [Description ("啟動此服務"),
     IsCPU (true)]
    uint32 StartService ();

    [Description ("停止此服務"),
     IsCPU (true)]
    uint32 StopService ();
};
```

類別宣告中的關鍵規則：

1. **Qualifier 區塊**（`[...]`）可掛在類別或特徵上，以逗號分隔多個 Qualifier
2. **型別**在前、**名稱**在後，類似 C++ 變數宣告
3. 每個 Property 或 Method 以 `;` 終止
4. **Key** Qualifier 標記的屬性構成此類別的主鍵（通常與父類別的 Key 共同組成複合主鍵）
5. **Override("name")** 表示此屬性是繼承自父類別的同名屬性覆寫

---

## Method 宣告語法

Method 的 EBNF 語法為[^cim-grammar]：

```ebnf
methodAndPropertyDcl ::= dataType methodAndPropertyName
parameterDcl          ::= '(' parameterList? ')' ';'
parameter             ::= '[' qualifierList ']'? (dataType | objectRef) parameterName array?
```

Method 與 Property 的主要區別在於**參數列表**——有 `(...)` 的是 Method，沒有的是 Property：

```mof
// Property（無參數括號）
uint16 OperationalStatus[];

// Method（有參數括號）
uint32 Reboot (string Reason, uint16 Timeout);
```

每個 Method 參數也可以有自己的 Qualifier 區塊：

```mof
uint32 SetThreshold (
    [In] uint16 MetricType,
    [In, MinVal(0), MaxVal(100)] uint16 ThresholdValue
);
```

注意：Method 的參數在 MOF 語法中可以是 `In`、`Out`、`InOut` 等方向，但這些在 CIM Metamodel 中透過 Qualifier 來標記。

---

## Association 宣告語法

在 CIM Metamodel 中，Association **不是獨立的語法結構**——它是一個標記了 `[Association]` Qualifier 的 Class[^cim-grammar][^cim-computersystem-mof]。因此在 MOF 層面，Association 就是用 `class` 關鍵字宣告，只是加上 Association 標記。

現實中，Association 使用 **REF（Reference）** 語法來指向參與的類別：

```mof
[Association, Version ("2.6.0"),
 Description ("表示 CIM_ComputerSystem 與在它之上執行的 CIM_Process 之間的擁有關係")]
class CIM_SystemProcess : CIM_ManagedElement {

    [Key, Override ("Antecedent")]
    CIM_System REF Antecedent;
    //  ^^^^^^^^^^ ^^^^
    //  型別是 CIM_System，使用 REF 語法表示是參考而非嵌入值

    [Key, Override ("Dependent")]
    CIM_Process REF Dependent;

    uint16 ProcessPriority;  // Association 也可以有自己的屬性
};
```

REF 語法（`ClassName REF RoleName`）不同於 Property 語法（`DataType PropertyName`）——REF 宣告的是一個**指向其他 CIM 物件的參考**，而非一個嵌入的資料值[^cim-grammar]。

此 Association 的語意可理解為：**一個 `CIM_System` 擁有零個或多個 `CIM_Process`，而每個 `CIM_Process` 歸屬於一個或多個 `CIM_System`**。

---

## Qualifier 宣告與使用

### Qualifier 宣告

所有 Qualifier 必須在 MOF 檔案的前端**先宣告後使用**[^cim-grammar]：

```ebnf
qualifierDcl ::= 'qualifier' qualifierName qualifierType scope defaultFlavor? ';'
qualifierType ::= ':' dataType array? defaultValue?
scope ::= ',' 'scope' '(' metaElement (',' metaElement)* ')'
```

實際範例：

```mof
Qualifier Description : string, scope (any), flavor (ToSubClass, Amended);
Qualifier Version    : string, scope (class, property), flavor (ToSubClass);
Qualifier Key        : boolean, scope (property);
Qualifier Override   : string, scope (property, reference);
Qualifier MaxLen     : sint32, scope (property);
Qualifier Write      : boolean, scope (property);
Qualifier Abstract   : boolean, scope (class);
Qualifier Association : boolean, scope (class);
Qualifier Indication  : boolean, scope (class);
Qualifier Experimental : boolean, scope (class, property);
Qualifier Deprecated : string[], scope (class, property, reference, method);
Qualifier ValueMap   : string[], scope (property);
Qualifier Values     : string[], scope (property);
Qualifier UMLPackagePath : string, scope (class);
Qualifier IsCPU      : boolean, scope (method);
```

Scope 參數指定此 Qualifier 可以掛在哪些 meta-element 上：`class`、`property`、`method`、`parameter`、`reference`、`schema`，或 `any`（所有元素）。

### Flavors（傳播規則）

Flavor 在 Qualifier 宣告中定義，控制 Qualifier 在繼承鏈中的傳播行為[^cim-grammar]：

```ebnf
defaultFlavor ::= ',' 'flavor' '(' flavor (',' flavor)* ')'
flavor        ::= 'enableoverride' | 'disableoverride' | 'tosubclass' | 'restricted' | 'translatable'
```

| Flavor | 預設值 | 效果 |
|--------|--------|------|
| `enableoverride` | `true` | 子類別可以覆寫此 Qualifier 的值 |
| `disableoverride` | `false` | 子類別**不能**覆寫此 Qualifier 的值 |
| `tosubclass` | `true` | 子類別會繼承此 Qualifier 的值 |
| `restricted` | `false` | 只作用於當前元素，不傳播到子類別 |
| `translatable` | `false` | 值可以多語言化（用於 Description 等） |

一個重要的限制：在**同一個命名空間內**，Flavor 的設定不能改變（以避免矛盾）。不同命名空間可以使用不同的 Flavor 設定。

### Qualifier 的使用

Qualifier 在類別、Property、Method 上使用 `[QualifierName, ...]` 的語法：

```mof
[Experimental, Version ("2.12.0"),
 UMLPackagePath ("CIM::Database::Sql")]
class CIM_SqlTable : CIM_SqlObject {

    [Key, Description ("Table ID"), MaxLen (64)]
    string TableID;

    [ValueMap ({"2", "3", "4"}),
     Values ({"SYSTEM_GENERATED", "USER_GENERATED", "DERIVED_SELF_REF"})]
    uint16 ReferenceType;

    [Description ("是否可 INSERT")]
    boolean insertable;
};
```

每個 Qualifier 可以帶參數（`QualifierName (value)`）或不帶參數（此時預設為 `true`）。

### Property 上常見的 Qualifier 組合

| Qualifier 組合 | 用途 |
|---------------|------|
| `[Key]` | 主鍵屬性 |
| `[Key, MaxLen(256), Description("...")]` | 字串主鍵 |
| `[Write]` | 可寫入的屬性（預設唯讀） |
| `[Override("parentProp")]` | 覆寫繼承的屬性 |
| `[Propagated("Class.prop")]` | 傳播父類別的主鍵屬性 |
| `[Required]` | 必須有值 |
| `[Deprecated({"alternative"})]` | 已廢棄，附帶替代路徑 |
| `[ValueMap({"0","1"}), Values({"Off","On"})]` | 列舉值對映 |
| `[MinVal(0), MaxVal(100)]` | 數值範圍限制 |

---

## Instance 宣告

MOF 支援宣告**預定義的實例**[^cim-grammar]：

```ebnf
instanceDcl ::= 'instance' 'of' className alias? '{' 
                 ( '[' qualifierList ']'? valueInitializer )+ 
               '}' ';'
```

最常見的是 `__namespace` 與 `__Namespace` 實例：

```mof
instance of __namespace {
    name = "ExampleDomain";
};

instance of __Namespace {
    Name = "Hardware";
};
```

也可以用來建立某個 CIM Class 的預設實例：

```mof
instance of CIM_ComputerSystem {
    CreationClassName = "CIM_ComputerSystem";
    Name = "default-system";
};
```

---

## 完整範例：從零寫一個 CIM_ManagedElement 的子類別

現在讓我們將所學整合為一個完整的 MOF 檔案——為一家虛構的「先進散熱管理系統」建立 CIM 擴展：

```mof
// ===================================================================
// CIM_ThermalSolution.mof
// 為 ACME Advanced Cooling Platform 提供的 CIM Extension Schema
// 遵循 DSP0221 v3.0.0 語法
// 基底版本：CIM 2.56.0
// ===================================================================

#pragma namespace("\\\\.root\\\\ACME\\\\Thermal")
#pragma locale("en_US")

instance of __Namespace {
    Name = "Thermal";
};

// ===================================================================
// Qualifier 宣告（先宣告後使用）
// ===================================================================
Qualifier Description : string,
    scope (class, property, reference, method),
    flavor (ToSubClass, Amended);

Qualifier Version : string,
    scope (class),
    flavor (ToSubClass);

Qualifier UMLPackagePath : string,
    scope (class),
    flavor (ToSubClass);

Qualifier Key : boolean,
    scope (property),
    flavor (ToSubClass);

Qualifier Write : boolean,
    scope (property);

Qualifier MaxLen : sint32,
    scope (property);

Qualifier ValueMap : string[],
    scope (property),
    flavor (ToSubClass, Amended);

Qualifier Values : string[],
    scope (property),
    flavor (ToSubClass, Amended);

Qualifier MinVal : sint32,
    scope (property);

Qualifier MaxVal : sint32,
    scope (property);

Qualifier Override : string,
    scope (property, reference);

Qualifier Deprecated : string[],
    scope (class, property);

Qualifier Experimental : boolean,
    scope (class, property);

// ===================================================================
// CIM_ThermalSensor — 散熱感應器
// ===================================================================
[Version ("1.0.0"),
 UMLPackagePath ("ACME::Thermal::Sensor"),
 Description ("ACME Advanced Cooling Platform 的智慧散熱感應器，"
              "支援多點溫度監控與即時回饋")]
class CIM_ThermalSensor : CIM_Sensor {

    [Key, Description ("散熱感應器的唯一識別碼"),
     MaxLen (64)]
    string SensorID;

    [Description ("目前溫度讀數（攝氏）"),
     MinVal (-50), MaxVal (150)]
    sint32 CurrentTemperature;

    [Write, Description ("溫度警報臨界值（攝氏）")]
    sint32 AlarmThreshold;

    [ValueMap ({"0", "1", "2", "3"}),
     Values ({"Unknown", "Normal", "Warning", "Critical"})]
    uint16 ThermalStatus;

    [Description ("冷卻設備控制狀態")]
    boolean CoolingActive;

    [Description ("取得目前溫度讀數")]
    sint32 ReadTemperature ();

    [Description ("設定溫度臨界值"),
     IsCPU (true)]
    uint32 SetThreshold (
        [In] sint32 ThresholdValue
    );
};

// ===================================================================
// CIM_ThermalCoolingUnit — 冷卻設備
// ===================================================================
[Version ("1.0.0"),
 UMLPackagePath ("ACME::Thermal::Cooling"),
 Description ("ACME 散熱管理平台中的一組冷卻設備，"
              "可為一個或多個 ThermalSensor 提供冷卻服務")]
class CIM_ThermalCoolingUnit : CIM_ManagedElement {

    [Key, Description ("冷卻設備名稱"),
     MaxLen (128)]
    string UnitName;

    [Description ("目前冷卻功率（百分比 0–100）"),
     MinVal (0), MaxVal (100)]
    uint16 PowerLevel;

    [Description ("設備目前是否在運作")]
    boolean IsRunning;

    [Description ("啟動冷卻設備")]
    uint32 StartUnit ();

    [Description ("停止冷卻設備")]
    uint32 StopUnit ();
};

// ===================================================================
// Association — ThermalSensorsAssociated — 感應器與冷卻設備的關聯
// ===================================================================
[Association, Version ("1.0.0"),
 UMLPackagePath ("ACME::Thermal::Cooling"),
 Description ("關聯 ThermalSensor 與其服務的 ThermalCoolingUnit")]
class CIM_ThermalSensorsAssociated : CIM_ManagedElement {

    [Key, Override ("Antecedent")]
    CIM_ThermalCoolingUnit REF Antecedent;

    [Key, Override ("Dependent")]
    CIM_ThermalSensor REF Dependent;

    [Description ("在此組合中的優先順序")]
    uint16 Priority;
};
```

讓我們逐段解讀這個 MOF 範例：

1. **命名空間宣告**：使用 `#pragma namespace` 設定路徑為 `\\.root\ACME\Thermal`，與 CIM Core Schema 的 `\\.root\CIM\Core\*` 保持一致的風格。

2. **Qualifier 宣告**：雖然 Qualifier 一般由 CIM Metamodel 或 CIM Schema 宣告，但獨立的 Extension Schema 可以（也應該）再次宣告它們以確保 MOF Compiler 能正確處理。DMTF 官方 Schema 檔案就採用相同的做法。

3. **`CIM_ThermalSensor`** 繼承自 `CIM_Sensor`，新增了 `CurrentTemperature`、`AlarmThreshold`、`ThermalStatus` 與 `CoolingActive` 等散熱專用屬性，以及 `ReadTemperature()` 和 `SetThreshold()` 兩個方法。

4. **`CIM_ThermalCoolingUnit`** 從 `CIM_ManagedElement` 直接繼承，代表一組冷卻設備。

5. **`CIM_ThermalSensorsAssociated`** 使用 Association 模式，透過 `REF` 語法建立 `CIM_ThermalCoolingUnit` 與 `CIM_ThermalSensor` 之間的多對多關係。`Priority` 屬性定義了在該組合中的優先級。

這個範例雖然是虛構的，但完全遵循了 CIM Schema 的真實寫法——你可以將它提交給 MOF Compiler（如 OpenPegasus 的 `mofcomp`）來產生 CIM Repository 的資料結構[^openpegasus-mofcomp]。

---

## MOF 語法的設計哲學

回顧整個 MOF 語法的設計，我們可以總結出三個核心原則：

1. **IDL 風格的熟悉感**——選擇 C++/IDL 風格的語法，讓 1990 年代晚期到 2000 年代初期的系統工程師能快速上手。花括號分組、分號終止、雙斜線註解，都是當時的主流語法特徵。

2. **Qualifier 為中心的可擴展性**——MOF 中幾乎所有元素（class、property、method、parameter、reference）都可以附加 Qualifier，這使得 Metamodel 不需要頻繁修改就能支援新的 Schema 需求。正如 Day 12 所討論的：Qualifier 是 CIM 可擴展性的關鍵。

3. **語言與工具的雙重定位**——MOF 不僅是供人閱讀的語言（它確實比 XML 或 XSD 更易讀），更是供 MOF Compiler 處理的機器輸入。這決定了它的語法必須嚴謹、無歧義、且可被 LL(1) 文法解析[^cim-grammar]。

---

## 小結

| 概念 | 對應語法 | |
|------|----------|--|
| Metamodel 的五大元素 | Class → `class`、Property → `type name`、Method → `type name(…)`、Association → `[Association] class …`、Indication → `[Indication] class …` |
| Qualifier 系統 | `[Qualifier(value), …]` 附加在任何元素上 |
| 型別系統 | 14 種純量型別 + REF + Array |
| Compiler Directive | `#pragma namespace/locale/include/…` |
| 繼承 | `class Child : Parent { … }` |
| 主鍵 | `[Key]` + 複合 Key 機制 |
| 註解 | `//` 和 `/* */` |

明天（Day 14）我們將把這些語法知識轉化為實戰能力——下載真實的 CIM Schema 2.56.0 MOF ZIP、分析 `CIM_System.mof` 的真實片段、並深入剖析 `CIM_ComputerSystem` 的完整 MOF 定義與 Association 的實際寫法。

---

## 參考資料

[^dmtf-cim-page]: DMTF. (n.d.). CIM Standards Overview. Retrieved 2026-10-03, from https://www.dmtf.org/standards/cim

[^dsp0221-300]: DMTF. (2012). Managed Object Format Specification — DSP0221 v3.0.0. Retrieved 2026-10-03, from https://www.dmtf.org/sites/default/files/standards/documents/DSP0221_3.0.0.pdf

[^cim-grammar]: DMTF / Pedro Assis. (n.d.). CIM v2.2.2 Grammar Definition (cim22Grammar.g). Retrieved 2026-10-03, from https://raw.githubusercontent.com/slebok/zoo/master/zoo/meta/mof/cim/assis/fetched/cim22Grammar.g

[^cim-system-mof]: DMTF. (2026). CIM Schema v2.56.0 — CIM_System.mof. Retrieved 2026-10-03, from https://schemas.dmtf.org/wbem/cim-html/2.56.0/

[^cim-service-mof]: DMTF. (n.d.). CIM_Service Class (v2.45.0+). Retrieved 2026-10-03, from https://schemas.dmtf.org/wbem/cim-html/2.55.0/CIM_Service.html

[^cim-computersystem-mof]: DMTF. (n.d.). CIM_ComputerSystem Class (v2.55.0). Retrieved 2026-10-03, from https://schemas.dmtf.org/wbem/cim-html/2.55.0/CIM_ComputerSystem.html

[^openpegasus-mofcomp]: OpenPegasus. (n.d.). OpenPegasus WBEM Services — MOF Compiler. Retrieved 2026-10-03, from https://sourceforge.net/projects/openpegasus/