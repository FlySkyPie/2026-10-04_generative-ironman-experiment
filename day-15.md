# Day 15：MOF Compiler — 從 MOF 到實作

昨天我們打開了真實的 CIM Schema 檔案，親自閱讀了 `System.mof` 中 `CIM_ComputerSystem` 的完整 MOF 定義，並追蹤了 `CIM_ComputerSystem` 到 `CIM_Process` 之間錯綜複雜的三層關聯鏈。如果說昨天是 **MOF 的閱讀理解**，那麼今天就是 **MOF 的自動化處理**——我們將深入探討 **MOF Compiler** 的架構與運作原理。

MOF Compiler 是 CIM 生態系中最關鍵的工具之一：它接收 MOF 語言的 Schema 檔案，剖析其內容，然後將結果**寫入目標 CIM Repository**，或者**生成另一種語言（C、Python 等）的資料結構**。它就像一個「橋樑」，讓抽象的 Schema 定義得以轉化為可被管理系統查詢與操作的具體物件。

## MOF Compiler 的編譯流程

無論是哪個實作，MOF Compiler 的基本流程都遵循以下六個階段[^dmtf-cim-page][^pywbem-compiler]：

```
MOF 文字輸入 (.mof)
        │
        ▼
┌─────────────────────┐
│  1. 詞法分析 (Lex)  │  將 MOF 原始文字切割為 tokens
│                      │  處理：字串跳脫、數字、關鍵字、識別碼
└──────────┬──────────┘
           │ tokens（記號串流）
           ▼
┌─────────────────────┐
│  2. 語法分析 (Yacc)  │  根據文法規則建構 AST（抽象語法樹）
│                      │  產出：CIMClass、CIMInstance、CIMQualifierDecl
└──────────┬──────────┘
           │ 結構化物件
           ▼
┌─────────────────────┐
│  3. 相依性解析       │  查閱參照類別、父類別、Qualifier 型別
│   (Resolution)       │  搜尋順序：記憶體快取 → 實體 Repository
└──────────┬──────────┘
           │ 已解析的完整定義
           ▼
┌─────────────────────┐
│  4. Repository 更新  │  將編譯結果寫入目標 Repository
│                      │  • 建立/修改類別 (addClass/modifyClass)
│                      │  • 建立/修改實例 (addInstance)
│                      │  • 建立/修改 Qualifier 宣告 (addQualifier)
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  5. 選擇性：閉包計算  │  計算類別的遞迴依賴關係
│   (Closure)          │  用於記憶體常駐 Repository (MRR) 生成
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  6. 輸出             │  • CIM Repository 更新（XML/BIN 格式）
│                      │  • WBEM 伺服器 Repository 更新
│                      │  • C 原始碼（*.cpp/*.h/*.mrr）生成
└─────────────────────┘
```

下面我們將逐一檢視三個主要的 MOF Compiler 實作，看看它們各自的設計取捨與適用場景。

---

## OpenPegasus MOF Compiler（cimmof）— C++ 實作

**OpenPegasus** 是 DMTF 官方最完整的 Open Source CIM 實作，其 MOF Compiler 位於原始碼樹的 `pegasus/src/Pegasus/Compiler/` 目錄下[^openpegasus-github]。

### 架構元件

| 元件 | 檔案 | 角色 |
|------|------|------|
| **詞法分析器 (Lexer)** | `cimmof.l` → `cimmof_lex.cpp` | Flex 產生；將 MOF 文字 token 化 |
| **語法分析器 (Parser)** | `cimmof.y` → `cimmof_tab.cpp` | Bison (YACC) 產生；LALR 文法分析 |
| **Parser 控制器** | `cimmofParser.h/.cpp` | 單例模式，協調整個編譯過程 |
| **Value Factory** | `valueFactory.h/.cpp` | 將 parsed 字串值轉換為型別化的 CIMValue |
| **宣告上下文** | `compilerDeclContext.h/.cpp` | 記憶體內查找，支援相依性解析 |
| **Repository 介面** | `cimmofRepositoryInterface.h/.cpp` | 統一的後端抽象層 |

### 三種 Repository 後端

`cimmofRepositoryInterface` 提供了統一的介面，對接三種不同的後端[^cimmof-parser]：

| 後端 | 類別 | 用途 |
|------|------|------|
| **本地 Repository** | `cimmofRepository` | 包裝 `CIMRepository`，直接讀寫檔案系統中的 XML/BIN Repository |
| **Client 連線** | `cimmofClient` | 透過 `CIMClient` 連接到執行中的 Pegasus WBEM 伺服器 |
| **MRR 生成器** | `cimmofMRR` | 生成 C 原始碼，將 Schema 嵌入為靜態資料結構 |

### MRR（Memory Resident Repository）

MRR 是 OpenPegasus 中一項獨特的設計。不同於傳統的「編譯 → 寫入 Repository 檔案 → 執行時期載入」流程，MRR 直接將 Schema 的內容**編譯為 C 原始碼**，使其成為可執行檔的一部分[^cimmof-mrr]：

```
MOF → cimmofMRR → *.cpp + *.h + *.mrr
                 → 編譯器 → 靜態連結進入 WBEM Server 可執行檔
```

MRR 生成的 C 原始碼包含：
- `MRRQualifierDecl` 結構（Qualifier 宣告）
- `MRRProperty`、`MRRReference`、`MRRMethod`、`MRRParameter` 結構
- `MRRClass` 結構（包含其特徵陣列）
- 頂層的 `MRRNameSpace` 結構（整合所有類別與 Qualifier）

這意味著 WBEM Server 可以在**完全不讀取任何檔案**的情況下，直接在記憶體中查詢 Schema 定義——對於嵌入式或高效能場景極有價值。

### 閉包計算（Closure）

`Closure()` 函數是 MRR 模式下的一個關鍵步驟。它計算一個類別的**遞迴依賴閉包**（Transitive Closure）[^cimmof-closure]：

> 一個類別的閉包包含所有滿足以下任意條件的類別：
> 1. 反身性：類別本身
> 2. 父類別（Superclass）
> 3. 參考屬性（型別為 Reference 的 Property）
> 4. 參考參數（型別為 Reference 的 Method 參數）
> 5. EmbeddedInstance Qualifier 中命名的類別
> 6. 對 1 到 5 中發現的類別遞迴應用

這確保了 MRR 生成的 C 原始碼包含所有必要的依賴類別，不會在執行時期出現未定義的符號。

---

## pywbem mof_compiler — Python 生態系的 CIM 工具

**pywbem** 是現代化的純 Python CIM/WBEM 用戶端堆疊。其 `mof_compiler` 是使用 **PLY（Python Lex-Yacc）** 實作的 MOF Compiler，目前仍在活躍維護中[^pywbem-github][^pywbem-compiler]。

### 安裝與設定

pywbem 是純 Python 套件，不需任何 OS 層級的前置依賴[^pywbem-install]：

```bash
pip install pywbem
```

安裝完成後，`mof_compiler` 指令列工具與 `pywbem` Python 套件即可使用。

### 指令列使用

最簡潔的編譯指令：

```bash
mof_compiler CIM_Schema_2.45.mof -s https://localhost -n root/cimv2 -u sheldon -p 42
```

| 選項 | 說明 |
|------|------|
| `moffile ...` | 一個或多個 MOF 檔案 |
| `-s, --server url` | WBEM 伺服器 URL（如 `https://host:5989`） |
| `-n, --namespace ns` | 目標命名空間（預設 `root/cimv2`） |
| `-u, --user user` | 認證使用者名稱 |
| `-p, --password pwd` | 密碼（省略時互動提示） |
| `-I, --include dir` | MOF 包含目錄（可多次指定） |
| `-d, --dry-run` | 僅檢查語法，不修改伺服器 |
| `-r, --remove` | 移除 MOF 檔案中定義的元素 |
| `-v, --verbose` | 輸出更多訊息 |

**語法檢查模式**（不連線伺服器）：

```bash
mof_compiler mymodel.mof -s https://myserver -n root/cimv2 -d
```

**移除先前編譯的元素**：

```bash
mof_compiler mymodel.mof -s https://myserver -n root/cimv2 -r
```

### Python API 使用

`MOFCompiler` 類別提供了完整的程式化控制：

```python
from pywbem import WBEMConnection
from pywbem._mof_compiler import MOFCompiler, MOFWBEMConnection

# 選項 A：針對真實 WBEM 伺服器編譯
conn = WBEMConnection('https://myserver:5989',
                      ('username', 'password'),
                      default_namespace='root/cimv2')
mofcomp = MOFCompiler(conn,
                      search_paths=['/path/to/schema'])

# 選項 B：記憶體內 Repository（不需伺服器）
local_repo = MOFWBEMConnection()
mofcomp = MOFCompiler(local_repo,
                      search_paths=['/path/to/schema'])
```

**核心方法**[^pywbem-mofcompiler-api]：

```python
# 編譯 MOF 檔案
mofcomp.compile_file('my_classes.mof', 'root/cimv2')

# 編譯 MOF 字串
mof_str = '''
    Qualifier Key : boolean = false,
        Scope(property, reference),
        Flavor(DisableOverride, ToSubclass);

    [Key]
    class CIM_Widget {
        [Key] string InstanceID;
        uint32 Quantity;
    };

    instance of CIM_Widget {
        InstanceID = "widget1";
        Quantity = 5;
    };
'''
mofcomp.compile_string(mof_str, 'root/cimv2')

# 查詢編譯後的類別
ccs = local_repo.GetClass('CIM_Widget', namespace='root/cimv2',
                           LocalOnly=False, IncludeQualifiers=True)
print(ccs.properties['Quantity'].type)  # 'uint32'

# 取得實例
from pywbem._cim_obj import CIMInstanceName
inst = local_repo.GetInstance(
    CIMInstanceName('CIM_Widget',
                    {'InstanceID': 'widget1'}))
print(inst['Quantity'])  # 5
```

### pywbem 的 MOF 語法限制

pywbem 的 MOF Compiler 支援 DSP0004，但有以下已知限制[^pywbem-compiler]：

1. 某些 MOF 語法 tokens（如 `indication`）不能用作屬性名稱
2. 帶有別名的實例必須初始化 Key 屬性（或具備非 NULL 的預設值）
3. 別名必須在使用前定義
4. 命名空間 pragma 僅接受 CIM 命名空間名稱，不接受完整路徑

---

## Windows mofcomp.exe — 從 WMI 到 MI

Day 24 我們將詳細探討 CIM 在 Windows 的實作，但這裡先做一個關鍵的前瞻說明：

在 CIM 的 Windows 生態系中，**並不存在一個名為 `mofcomp.exe` 的獨立工具**。Windows 上的 MOF Compiler 就是 OpenPegasus 專案產生的 **`cimmof.exe`**[^openpegasus-installer]。OpenPegasus 的 InnoSetup 安裝腳本明確使用 `cimmof.exe` 在安裝期間將 Schema MOF 檔案編譯為 CIM Repository：

| 工具 | 說明 |
|------|------|
| `cimmof.exe` | OpenPegasus MOF Compiler（Windows 版本） |
| `cimmofl.exe` | 本地端 MOF Compiler 變體 |
| `mof_compiler.bat` | pywbem 提供的 Windows 批次包裝 |

pywbem 則提供 `mof_compiler.bat` 作為 Windows 上的啟動腳本[^pywbem-mof-compiler-bat]：

```batch
@setlocal enableextensions
@python %~d0%~p0%~n0 %*
@endlocal
@exit /b %errorlevel%
```

---

## 實作練習：編譯 MOF 並查詢產生物件

讓我們用 pywbem 的 **FakedWBEMConnection**（記憶體內模擬 WBEM 伺服器）來完成一個完整的端到端演練[^pywbem-mock]：

```python
import pywbem
import pywbem_mock
from pywbem._cim_obj import CIMInstanceName

# 建立模擬 WBEM 伺服器
conn = pywbem_mock.FakedWBEMConnection(
    default_namespace='root/cimv2')

# 定義 MOF 內容
mof_content = '''
    Qualifier Key : boolean = false,
        Scope(property, reference),
        Flavor(DisableOverride, ToSubclass);

    Qualifier Description : string = null,
        Scope(any),
        Flavor(EnableOverride, ToSubclass, Translatable);

    [Description ("代表一台可管理的伺服器")]
    class CIM_MyServer {
        [Key, Description ("唯一識別碼")]
        string InstanceID;

        [Description ("伺服器名稱")]
        string Name;

        [Description ("CPU 數量")]
        uint16 NumCPUs;

        [Description ("記憶體大小 (MB)")]
        uint32 TotalMemoryMB;
    };

    instance of CIM_MyServer as $S1 {
        InstanceID = "svr-001";
        Name = "prod-web-01";
        NumCPUs = 16;
        TotalMemoryMB = 65536;
    };

    instance of CIM_MyServer as $S2 {
        InstanceID = "svr-002";
        Name = "prod-db-01";
        NumCPUs = 32;
        TotalMemoryMB = 262144;
    };
'''

# 編譯 MOF 字串
conn.compile_mof_string(mof_content)

# 查詢所有實例
instances = conn.EnumerateInstances('CIM_MyServer')
for inst in instances:
    print(f"路徑: {inst.path}")
    print(inst.tomof())
    print("---")

# 取得特定實例
inst = conn.GetInstance(
    CIMInstanceName('CIM_MyServer',
                    {'InstanceID': 'svr-001'}))
print(f"伺服器名稱: {inst['Name']}")
print(f"CPU 數量:   {inst['NumCPUs']}")
print(f"記憶體:     {inst['TotalMemoryMB']} MB")
```

**執行結果**：

```
路徑: root/cimv2:CIM_MyServer.InstanceID="svr-001",Name="prod-web-01"
instance of CIM_MyServer {
   InstanceID = "svr-001";
   Name = "prod-web-01";
   NumCPUs = 16;
   TotalMemoryMB = 65536;
};

---
路徑: root/cimv2:CIM_MyServer.InstanceID="svr-002",Name="prod-db-01"
instance of CIM_MyServer {
   InstanceID = "svr-002";
   Name = "prod-db-01";
   NumCPUs = 32;
   TotalMemoryMB = 262144;
};

---
伺服器名稱: prod-web-01
CPU 數量:   16
記憶體:     65536 MB
```

這個練習展示了 MOF Compiler 的核心工作流程：

1. **詞法分析**與**語法分析**：將 MOF 文字轉化為 `CIMClass` / `CIMInstance` 等結構化物件
2. **Repository 更新**：將解析後的類別與實例寫入目標 Repository（此處為記憶體模擬）
3. **查詢操作**：完成編譯後，即可使用標準的 `EnumerateInstances`、`GetInstance` 等 WBEM 操作來查詢與操作 CIM 物件

---

## 三種 MOF Compiler 的比較

| 面向 | OpenPegasus cimmof | pywbem mof_compiler | Windows 工具 |
|------|--------------------|--------------------|-------------|
| **語言** | C++（Bison/Flex） | 純 Python（PLY） | C++（cimmof.exe） |
| **維護狀態** | 休眠（最後 CVS 版本 2.14.1） | 活躍維護 | 同 OpenPegasus |
| **安裝方式** | 原始碼編譯 | `pip install pywbem` | OpenPegasus Installer |
| **Repository 後端** | 本地、Client、MRR | WBEMConnection / MOFWBEMConnection | 本地 / Client |
| **特殊功能** | MRR（C 原始碼生成） | 記憶體內模擬（mock） | 與 WMI/MI 整合 |
| **適用場景** | 高效能 WBEM Server | Python 自動化、測試、教育 | Windows 管理環境 |

---

## 小結

今天我們深入探討了 MOF Compiler 的完整架構與三種主要實作：

- **OpenPegasus cimmof**（C++）是最完整的參考實作，其 MRR 功能可將 Schema 編譯為 C 原始碼進行靜態連結，適合高效能與嵌入式場景
- **pywbem mof_compiler**（Python）是當前最活躍的實作，提供便利的指令列工具與靈活的 Python API，尤其適合自動化腳本、測試開發與學習目的
- **Windows cimmof.exe** 源自 OpenPegasus，為 Windows 管理環境提供 MOF 編譯能力

無論哪個實作，MOF Compiler 的核心價值始終如一：**將人類可讀的 Schema 定義轉化為機器可操作的管理物件**。明天，我們將從編譯器這個「文字化」的世界，轉向圖形化的建模語言——**UML Profile for CIM**，了解如何以 UML 圖形來設計 CIM Schema，以及如何從 UML 逆向生成 MOF 定義。

[^dmtf-cim-page]: DMTF. (n.d.). CIM — Common Information Model. Retrieved 2026-10-03, from https://www.dmtf.org/standards/cim
[^pywbem-compiler]: pywbem. (n.d.). MOF Compiler — pywbem Documentation. Retrieved 2026-10-03, from https://pywbem.readthedocs.io/en/latest/compiler.html
[^openpegasus-github]: OpenPegasus. (n.d.). OpenPegasus — Open Source WBEM/CIM Implementation. Retrieved 2026-10-03, from https://github.com/OpenPegasus/OpenPegasus
[^cimmof-parser]: OpenPegasus. (n.d.). cimmofParser.h — Compiler Architecture Overview. Retrieved 2026-10-03, from https://raw.githubusercontent.com/OpenPegasus/OpenPegasus/main/pegasus/src/Pegasus/Compiler/cimmofParser.h
[^cimmof-mrr]: OpenPegasus. (n.d.). cimmofMRR.cpp — Memory Resident Repository Generator. Retrieved 2026-10-03, from https://raw.githubusercontent.com/OpenPegasus/OpenPegasus/main/pegasus/src/Pegasus/Compiler/cimmofMRR.cpp
[^cimmof-closure]: OpenPegasus. (n.d.). Closure.h — Class Dependency Closure. Retrieved 2026-10-03, from https://raw.githubusercontent.com/OpenPegasus/OpenPegasus/main/pegasus/src/Pegasus/Compiler/Closure.h
[^pywbem-github]: pywbem. (n.d.). pywbem — Python WBEM Client and MOF Compiler. Retrieved 2026-10-03, from https://github.com/pywbem/pywbem
[^pywbem-install]: pywbem. (n.d.). Installation — pywbem Documentation. Retrieved 2026-10-03, from https://pywbem.readthedocs.io/en/latest/intro.html#installation
[^pywbem-mofcompiler-api]: pywbem. (n.d.). _mof_compiler.py — MOF Compiler Source Code. Retrieved 2026-10-03, from https://raw.githubusercontent.com/pywbem/pywbem/master/pywbem/_mof_compiler.py
[^pywbem-mock]: pywbem. (n.d.). Mock WBEM Server — pywbem Documentation. Retrieved 2026-10-03, from https://pywbem.readthedocs.io/en/latest/mockwbemserver.html
[^pywbem-mof-compiler-bat]: pywbem. (n.d.). mof_compiler.bat — Windows Wrapper. Retrieved 2026-10-03, from https://raw.githubusercontent.com/pywbem/pywbem/master/mof_compiler.bat
[^openpegasus-installer]: OpenPegasus. (n.d.). OpenPegasus.iss — InnoSetup Installer Script. Retrieved 2026-10-03, from https://raw.githubusercontent.com/OpenPegasus/OpenPegasus/main/pegasus/installs/InnoSetup/OpenPegasus.iss