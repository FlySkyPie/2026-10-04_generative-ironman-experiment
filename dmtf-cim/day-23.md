# Day 23：pywbem — Python 生態系的 CIM 工具

昨天我們深入了 OpenPegasus——CIM/WBEM 生態系中最完整的旗艦級開源實作，了解了其伺服器架構、Provider 框架與 CLI 工具群。今天——第五階段的第三篇——我們將把目光從 C++ 轉向 Python，認識 Python 生態系中操作 CIM 的核心工具：**pywbem** 與 **pywbemtools**。

如果說 OpenPegasus 是重型卡車——功能完整但部署與開發成本高，那麼 pywbem 就是靈活的轎車——輕量、易安裝、適合快速開發與自動化腳本。兩者相輔相成：pywbem 負責**客戶端**的查詢與操作，可連接到 OpenPegasus 或其他 WBEM 伺服器[^pywbem-intro]。

## 專案背景

pywbem 是一個純 Python 的 WBEM 客戶端與工具套件，遵循 DMTF 的 CIM/WBEM 標準（尤其是 DSP0200 與 DSP0201——CIM-XML 協定）[^pywbem-readme]。

| 屬性 | 值 |
|------|----|
| **最新穩定版** | 1.9.1（2026-08-31 發佈） |
| **開發版** | 1.10.0a1 |
| **Python 版本** | ≥ 3.9 |
| **授權** | LGPL-2.1-or-later |
| **GitHub Stars** | 43 |
| **主要維護者** | Andreas Maier、Karl Schopmeyer |
| **原始作者** | Tim Potter |

專案原始碼存放於 GitHub 的 `pywbem/pywbem`，並在 Read the Docs 有完整文件[^pywbem-github]。其專案組織下還包含 `pywbem/pywbemtools`（CLI 工具）、`pywbem/cimserver`（實驗性 WBEM 伺服器）、`pywbem/pyprov`（實驗性 Python Provider）等相關專案[^pywbemtools-github]。

## pywbem 的六大 Python API

pywbem 的核心是一個客戶端函式庫，但它提供的功能遠不止於此。以下六個 Python API 構成了 pywbem 的完整面貌[^pywbem-intro]：

### 1. WBEM 客戶端函式庫（WBEM Client Library）

最主要的 API，用於向 WBEM 伺服器發送 CIM 操作——包括 `EnumerateInstances`、`GetInstance`、`CreateInstance`、`DeleteInstance`、`InvokeMethod`、`ExecQuery` 等[^pywbem-client]。

```python
from pywbem import WBEMConnection, CIMError, Error

conn = WBEMConnection(
    'https://localhost',
    ('fred', 'blah'),
    default_namespace='root/cimv2',
    no_verification=True
)

try:
    instances = conn.EnumerateInstances('CIM_ComputerSystem')
    for inst in instances:
        print(f'Path: {inst.path}')
        print(inst.tomof())
except CIMError as err:
    print(f'CIMError: code={err.status_code_name}')
except Error as err:
    print(f'Error: {err}')
```

### 2. WBEM 伺服器封裝（WBEM Server Library）

封裝 WBEM 伺服器的功能查詢——品牌識別、Interop Namespace、Management Profiles 等[^pywbem-server]：

```python
from pywbem import WBEMServer, ValueMapping

server = WBEMServer(conn)
print(f'Brand: {server.brand}')
print(f'Version: {server.version}')
print(f'Interop namespace: {server.interop_ns}')

for inst in server.profiles:
    print(f'  {inst["RegisteredName"]} v{inst["RegisteredVersion"]}')
```

### 3. WBEM 指示監聽器（WBEM Indication Listener）

建立執行緒式的監聽器，用於接收 WBEM 伺服器發送的非同步事件通知（Indications）[^pywbem-intro]：

```python
from pywbem import WBEMListener

listener = WBEMListener('http://0.0.0.0:5999')
listener.add_callback(my_callback_function)
listener.start()
```

### 4. WBEM 訂閱管理器（WBEM Subscription Manager）

用於查詢與管理 WBEM 伺服器上的指示訂閱（Indication Subscriptions）[^pywbem-intro]。

### 5. MOF 編譯器（MOF Compiler API）

以程式方式將 MOF 檔案或字串編譯到 CIM Repository 中（詳見下一節）。

### 6. Mock WBEM 伺服器（Mock WBEM Server）

在本機建立一個記憶體中的模擬 WBEM 伺服器，不需要真實的 WBEM 伺服器即可進行原型開發與測試[^pywbem-mock]：

```python
from pywbem_mock import FakedWBEMConnection, DMTF_MOF

conn = FakedWBEMConnection(default_namespace='root/cimv2')
conn.build_repository_from_mof(DMTF_MOF)

conn.add_cim_obj(conn.GetClass('CIM_ComputerSystem'), 'root/cimv2')
inst = conn.EnumerateInstances('CIM_ComputerSystem')
print(inst[0].tomof())
```

## MOF Compiler：從 Schema 到可操作物件

MOF Compiler 是 pywbem 中最具生產力的工具之一。它以 **DSP0004（CIM Metamodel）** 定義的 MOF 語法檔案為輸入，將其編譯到 CIM Repository 中——無論是遠端的 WBEM 伺服器，還是本機的 Mock Repository[^pywbem-compiler]。

### 命令列形式（`mof_compiler`）

安裝 pywbem 後即可使用：

```bash
# 基本語法檢查（不修改儲存庫）
mof_compiler CIM_Schema_2.56.mof -d

# 編譯到 WBEM 伺服器
mof_compiler CIM_Schema_2.56.mof \
    -s https://localhost \
    -n root/cimv2 \
    -u user -p secret \
    --no-verify-cert

# 移除已編譯的元素
mof_compiler CIM_Schema_2.56.mof \
    -s https://localhost -n root/cimv2 \
    -u user -p secret -r
```

| 選項 | 用途 |
|------|------|
| `-s, --server` | WBEM 伺服器 URL |
| `-n, --namespace` | 目標 CIM Namespace（預設 `root/cimv2`） |
| `-u, -p` | 認證 |
| `-d, --dry-run` | 只檢查語法，不修改儲存庫 |
| `-r, --remove` | 移除模式 |
| `-I, --include` | MOF 包含目錄 |

### Python API 形式

```python
from pywbem import MOFCompiler, WBEMConnection

conn = WBEMConnection('https://localhost', ('user', 'secret'),
                      default_namespace='root/cimv2',
                      no_verification=True)
compiler = MOFCompiler(conn, verbose=True)

# 編譯 MOF 檔案
compiler.compile_file('my_model.mof', 'root/cimv2')

# 編譯 MOF 字串
compiler.compile_string('''
    instance of CIM_ComputerSystem {
        Name = "MySystem";
        SystemDescription = "A test system";
    };
''', 'root/cimv2')

# 查詢依賴的 MOF 檔案
path = compiler.find_mof('CIM_ComputerSystem')

# 出錯時回滾
compiler.rollback()
```

`MOFCompiler` 的 `compile_file()` 支援 `pragma include` 指令（對應的 MOF 包含檔案），並會自動在 `search_paths` 中搜尋依賴類別的 MOF 定義（如父類別、限定子型別的定義）[^pywbem-compiler]。

## pywbemcli：命令列的 WBEM 客戶端

`pywbemcli` 是 `pywbemtools` 套件提供的命令列工具，支援一次性命令模式與互動式（REPL）模式[^pywbemcli-docs]。

### 安裝

```bash
pip install pywbemtools
# 或使用 pipx（建議，可全域使用）
pipx install pywbemtools
```

安裝完成後即可使用 `pywbemcli` 與 `pywbemlistener` 兩項指令。

### 連線管理

建立持久連線定義，避免每次重複輸入伺服器位址與認證：

```bash
# 連線到真實 WBEM 伺服器
pywbemcli -s https://localhost --no-verify -u user -p secret \
    connection save my-server

# 使用 MOF 檔案建立本機 Mock 伺服器
pywbemcli -m tests/unit/simple_assoc_mock_model.mof \
    connection save assoc1
```

### 查詢 CIM Class

```bash
# 列出所有 CIM 類別
pywbemcli -n my-server class enumerate

# 以樹狀結構展示類別階層
pywbemcli -n my-server class tree

# 檢視特定類別的定義
pywbemcli -n my-server class get CIM_ComputerSystem --show-attributes

# 查詢特定類別的 Association 與 References
pywbemcli -n my-server class associators CIM_ComputerSystem
```

### 操作 CIM Instance

```bash
# 列舉所有 CIM_ComputerSystem 實例
pywbemcli -n my-server instance enumerate CIM_ComputerSystem

# 取得特定實例
pywbemcli -n my-server instance get \
    'root/cimv2:CIM_ComputerSystem.Name="localhost"'

# 建立新實例
pywbemcli -n my-server instance create \
    'root/cimv2:CIM_ComputerSystem.Name="svr-03"'

# 修改屬性
pywbemcli -n my-server instance modify \
    'root/cimv2:CIM_ComputerSystem.Name="localhost"' \
    Description="Production server"

# 查詢關聯的實例
pywbemcli -n my-server instance associators \
    'root/cimv2:CIM_ComputerSystem.Name="localhost"'
```

### 互動式 REPL 模式

```bash
pywbemcli -n my-server
pywbemcli> server brand
pywbemcli> server interop
pywbemcli> server profiles --organization DMTF
pywbemcli> class tree
pywbemcli> enum CIM_ComputerSystem
pywbemcli> :q   # 離開
```

REPL 模式支援 tab-completion 與指令歷史，是探索 CIM Repository 最直觀的方式。

## 完整範例：自動化查詢 CIM Repository

結合以上 API，這是一個完整的 Python 程式範例，用於連接到 WBEM 伺服器、查詢系統資訊、以及列出所有可用的 Management Profiles[^pywbem-examples]：

```python
#!/usr/bin/env python3
"""
pywbem CIM Repository 查詢範例
"""
import sys
from pywbem import WBEMConnection, WBEMServer, ValueMapping, CIMError

SERVER_URL = 'https://localhost'
USERNAME = 'admin'
PASSWORD = 'secret'
NAMESPACE = 'root/cimv2'

# 建立連線
conn = WBEMConnection(SERVER_URL, (USERNAME, PASSWORD),
                      default_namespace=NAMESPACE,
                      no_verification=True)

try:
    # 伺服器資訊
    server = WBEMServer(conn)
    print(f'WBEM 伺服器品牌: {server.brand}')
    print(f'Interop 命名空間: {server.interop_ns}')
    print()

    # 列出所有 Management Profiles
    print('支援的 Management Profiles:')
    org_vm = ValueMapping.for_property(
        server, server.interop_ns,
        'CIM_RegisteredProfile', 'RegisteredOrganization'
    )
    for inst in server.profiles:
        org = org_vm.tovalues(inst['RegisteredOrganization'])
        name = inst['RegisteredName']
        vers = inst['RegisteredVersion']
        print(f'  {org} {name} v{vers}')
    print()

    # 查詢電腦系統
    print('CIM_ComputerSystem 實例:')
    for inst in conn.EnumerateInstances('CIM_ComputerSystem'):
        print(inst.tomof())
    print()

    # ExecQuery 範例
    print('ExecQuery 結果:')
    result = conn.ExecQuery(
        NAMESPACE,
        'SELECT * FROM CIM_ComputerSystem WHERE Name LIKE "server%"'
    )
    for item in result:
        print(f'  {item["Name"]}: {item.get("Description", "N/A")}')

except CIMError as err:
    print(f'CIM 錯誤: {err.status_code_name}: {err.status_description}')
    sys.exit(1)
except Exception as err:
    print(f'錯誤: {err}')
    sys.exit(1)
```

## Pull Operations：處理大規模查詢

CIM Repository 可能包含數千甚至數萬個實例（例如大型資料中心的所有伺服器）。傳統的 `EnumerateInstances` 一次回傳全部結果，可能造成記憶體問題。pywbem 支援 DMTF 標準的 **Pull Operations**——將一次回應拆分為多次小批次[^pywbem-concepts]：

```python
# Iterator 形式的 pull operation
# 自動在傳統模式與 pull 模式之間切換
for inst in conn.IterEnumerateInstances('CIM_ComputerSystem'):
    process_instance(inst)  # 逐個處理，記憶體恆定
```

如果伺服器支援 Pull Operations，`IterEnumerateInstances` 會使用 Open/Pull/Close 三階段的協議；若不支援，則自動退回傳統的 `EnumerateInstances`。

## 與 OpenPegasus 的比較

| 面向 | pywbem | OpenPegasus |
|------|--------|-------------|
| **角色** | 純客戶端 | 伺服器 + Provider 框架 |
| **部署** | `pip install` 即可 | 需原始碼編譯、環境變數設定 |
| **語言** | Python | C++ |
| **Mock 測試** | 內建 `FakedWBEMConnection` | 需完整的 Provider 開發 |
| **伺服器功能** | 無（需連接到 WBEM 伺服器） | 完整 WBEM 伺服器 + CIMOM |
| **適用場景** | 快速腳本、自動化、測試 | 生產部署、Provider 開發 |

兩者並非競爭關係，而是**互補**：OpenPegasus 作為伺服端提供 CIM Object Manager 與 Provider Framework，pywbem 作為客戶端提供靈活的查詢與操作介面。

## 小結

1. **pywbem** 是一個純 Python 的 WBEM 客戶端函式庫，提供六大 Python API（WBEM 客戶端、伺服器封裝、事件監聽、訂閱管理、MOF 編譯、Mock 伺服器）。
2. **MOF Compiler** 可以將 MOF 檔案編譯到 CIM Repository，支援命令列與 Python API 兩種形式。
3. **pywbemcli** 是 `pywbemtools` 提供的命令列 WBEM 客戶端，支援一次性命令與互動式 REPL 模式，是探索 CIM 最便捷的方式。
4. **Pull Operations** 讓大規模 CIM Repository 的查詢變得記憶體友善，適合處理資料中心等級的資料量。
5. pywbem 與 OpenPegasus 互補——前者是輕量客戶端，後者是完整伺服器。

明天我們將進入第五階段的第四篇——**CIM 在 Windows 的實作：WMI 與 MI**。

[^pywbem-intro]: pywbem. (2026). Introduction — pywbem documentation. Retrieved 2026-10-03, from https://pywbem.readthedocs.io/en/latest/intro.html
[^pywbem-readme]: pywbem. (2026). pywbem GitHub Repository. Retrieved 2026-10-03, from https://github.com/pywbem/pywbem
[^pywbem-github]: pywbem. (2026). pywbem Project Homepage. Retrieved 2026-10-03, from https://pywbem.github.io/
[^pywbemtools-github]: pywbem. (2026). pywbemtools GitHub Repository. Retrieved 2026-10-03, from https://github.com/pywbem/pywbemtools
[^pywbem-client]: pywbem. (2026). WBEM Client Library — pywbem documentation. Retrieved 2026-10-03, from https://pywbem.readthedocs.io/en/latest/client.html
[^pywbem-server]: pywbem. (2026). WBEM Server Functions — pywbem documentation. Retrieved 2026-10-03, from https://pywbem.readthedocs.io/en/latest/client/operations.html
[^pywbem-mock]: pywbem. (2026). Mock WBEM Server — pywbem documentation. Retrieved 2026-10-03, from https://pywbem.readthedocs.io/en/latest/mock.html
[^pywbem-compiler]: pywbem. (2026). MOF Compiler — pywbem documentation. Retrieved 2026-10-03, from https://pywbem.readthedocs.io/en/latest/compiler.html
[^pywbemcli-docs]: pywbem. (2026). pywbemcli — pywbemtools documentation. Retrieved 2026-10-03, from https://pywbemtools.readthedocs.io/en/stable/pywbemcli/index.html
[^pywbem-examples]: pywbem. (2026). pywbem Examples Directory. Retrieved 2026-10-03, from https://github.com/pywbem/pywbem/tree/master/examples/
[^pywbem-concepts]: pywbem. (2026). Concepts — pywbem documentation (Pull Operations). Retrieved 2026-10-03, from https://pywbem.readthedocs.io/en/latest/concepts.html