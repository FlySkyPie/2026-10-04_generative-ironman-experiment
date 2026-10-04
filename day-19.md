# Day 19：WS-Management — 企業級 SOAP 管理

昨天我們深入探討了 CIM-XML——WBEM 家族中年紀最大的成員，它以 XML-RPC over HTTP 的風格定義了 CIM 物件的查詢與操作。今天，我們將目光轉向 WBEM 家族的第二個協定——**WS-Management（Web Services for Management）**。

如果說 CIM-XML 是 WBEM 的「經典語言」，那麼 WS-Management 就是 WBEM 的「國際標準語言」。它採用了 **SOAP（Simple Object Access Protocol）** 與 **WSDL（Web Services Description Language）** 這套企業級 Web Services 技術堆疊，並最終在 2013 年被 ISO/IEC 採納為國際標準（ISO/IEC 17963:2013）[^dmtf-iso]。

## WS-Management 的定位與哲學

WS-Management 由 DMTF 於 2005 年接手標準化，其核心規格文件為 **DSP0226 v1.2.0**（2014-09-30 發佈），正式名稱為「Web Services for Management (WS-Management) Specification」[^dmtf-wsman]。

WS-Management 的設計哲學與 CIM-XML 有根本性的差異：

| 面向 | CIM-XML | WS-Management |
|------|---------|---------------|
| **通訊模式** | RPC（自訂 XML 格式） | SOAP（標準 SOAP Envelope） |
| **介面描述** | 隱含（由 Schema 定義） | 明確（WSDL 文件） |
| **狀態管理** | 無狀態（Stateless） | 有狀態（Subscription、Enumeration 支援） |
| **事件機制** | Export Messages（簡單） | WS-Eventing（完整訂閱管理） |
| **標準化路徑** | DMTF 專屬 | DMTF → ISO/IEC 17963:2013 |

CIM-XML 追求的是**簡單直接**——把 CIM 操作直接映射到 HTTP + XML。WS-Management 則追求的是**企業級標準化**——透過 SOAP/WSDL/WS-Addressing 這套完整的 Web Services 標準堆疊，讓管理介面具備與企業級商業應用相同層級的嚴謹性與互通性[^wikipedia-wsman]。

## 三類核心操作

WS-Management 定義了三大類操作，每一類都有各自的 WSDL Port Type 與 XML Schema：

### 1. Resource Access Operations（資源存取）

這是最基礎的操作，對應於 CRUD（Create、Read、Update、Delete）模式[^dmtf-wsman-resource]：

| WSDL Port Type | 操作 | HTTP 語意比擬 | 說明 |
|---------------|------|--------------|------|
| **Resource** | **Get** | GET | 讀取一個資源實例的所有屬性 |
| **Resource** | **Put** | PUT | 更新（或建立）一個資源實例 |
| **Resource** | **Delete** | DELETE | 刪除一個資源實例 |
| **ResourceFactory** | **Create** | POST | 建立一個新的資源實例 |

這些操作的 SOAP Action URI 遵循 `http://schemas.xmlsoap.org/ws/2004/09/transfer/{Operation}` 的模式。

Get 操作的 WSDL 定義（簡化）[^dsp8035-wsdl]：

```xml
<wsdl:portType name="Resource">
  <wsdl:operation name="Get">
    <wsdl:input message="tns:EmptyMessage"
                wsaw:Action="http://schemas.xmlsoap.org/ws/2004/09/transfer/Get"/>
    <wsdl:output message="tns:GetResponseMessage"
                 wsaw:Action="http://schemas.xmlsoap.org/ws/2004/09/transfer/GetResponse"/>
  </wsdl:operation>
  <!-- Put, Delete 類似結構 -->
</wsdl:portType>
```

### 2. Enumeration Operations（列舉操作）

用於列舉大量資料集合（如日誌、表格），採用「建立列舉會話 → 分批提取」的模式[^dmtf-wsman-enum]：

| WSDL Port Type | 操作 | 說明 |
|---------------|------|------|
| **DataSource** | **Enumerate** | 開始一個列舉序列，回傳 `EnumerationContext`（列舉上下文） |
| **DataSource** | **Pull** | 從活躍的列舉序列中提取資料（支援 `MaxTime`、`MaxElements`、`MaxCharacters` 三種節流參數） |
| **DataSource** | **Renew** | 延長列舉序列的過期時間 |
| **DataSource** | **GetStatus** | 查詢列舉序列的狀態與過期時間 |
| **DataSource** | **Release** | 明確關閉列舉序列 |
| **EnumEndEndpoint** | **EnumerationEnd** | （非同步通知）當列舉序列意外終止時發送 |

這種「Enumerate → Pull」的兩階段模式解決了 CIM-XML 的 `EnumerateInstances` 在面對大量資料時的回應大小問題。用戶端可以控制每次提取的資料量，避免單一回應過大。

### 3. Notification Operations（事件通知操作）

用於事件訂閱與非同步通知，採用 WS-Eventing 模式[^dmtf-wsman-event]：

| WSDL Port Type | 操作 | 說明 |
|---------------|------|------|
| **EventSource** | **Subscribe** | 建立一個事件訂閱；指定 Delivery 模式、Expires 時間、過濾條件 |
| **SubscriptionManager** | **Renew** | 延長訂閱的存活時間 |
| **SubscriptionManager** | **GetStatus** | 查詢訂閱的狀態 |
| **SubscriptionManager** | **Unsubscribe** | 取消訂閱 |
| **EndToEndpoint** | **SubscriptionEnd** | （非同步通知）當訂閱因傳送失敗或來源終止而結束時發送 |

這比 CIM-XML 的 Export Messages 更加完整——WS-Management 提供了完整的訂閱生命週期管理，包括過期時間、重新續約、明確取消等功能。

### Identify 操作：探索能力的敲門

除了上述三類操作外，WS-Management 還定義了一個特殊的 **Identify** 操作[^wsman-identity]：

```xml
<wsmid:IdentifyRequest>
  <!-- 空的請求 -->
</wsmid:IdentifyRequest>

<wsmid:IdentifyResponse>
  <wsmid:ProtocolVersion>1.2</wsmid:ProtocolVersion>
  <wsmid:ProductVendor>Microsoft</wsmid:ProductVendor>
  <wsmid:ProductVersion>WinRM 3.0</wsmid:ProductVersion>
  <wsmid:InitiativeSupport>
    <wsmid:InitiativeName>DASH</wsmid:InitiativeName>
    <wsmid:InitiativeVersion>1.2</wsmid:InitiativeVersion>
  </wsmid:InitiativeSupport>
  <wsmid:SecurityProfiles>http://schemas.dmtf.org/wbem/wsman/1/security/tls</wsmid:SecurityProfiles>
</wsmid:IdentifyResponse>
```

這個操作在 DASH/SMASH 的兩階段發現機制中扮演了關鍵角色——Phase 1 透過 RMCP 廣播探測 WS-Management 能力，Phase 2 就是用 Identify 操作確認具體的 DASH/SMASH 支援版本、安全設定與能力清單[^wiki-dash]。

---

## DSP0227：WS-Management CIM Binding

CIM 是協定中立的資訊模型，而 WS-Management 是具體的通訊協定——這兩者之間的橋樑，正是 **DSP0227（WS-Management CIM Binding Specification）**[^dmtf-dsp0227]。

### CIM 類別 → WS-Management 資源

每個 CIM 類別與每個 CIM 實例都被定址為一個 WS-Management 資源（Resource），透過 URI 來識別：

| CIM 概念 | WS-Management 資源 URI | 範例 |
|---------|----------------------|------|
| 命名空間 | `{namespace path}` | `root/cimv2` |
| 類別 | `{namespace}:{ClassName}` | `root/cimv2:CIM_ComputerSystem` |
| 實例 | `{namespace}:{ClassName}.Key=Value` | `root/cimv2:CIM_ComputerSystem.Name="svr-01"` |

### 操作對應

DSP0227 定義了 CIM 操作與 WS-Management 操作之間的對應關係[^pywbem-operations]：

| CIM 操作 | WS-Management 操作 | 說明 |
|---------|-------------------|------|
| `GetInstance` | **Get** | 讀取實例的所有屬性 |
| `ModifyInstance` | **Put** | 修改指定的屬性值 |
| `CreateInstance` | **Create** | 建立新實例 |
| `DeleteInstance` | **Delete** | 刪除實例 |
| `EnumerateInstances` | **Enumerate + Pull** | 列舉所有實例（兩階段） |
| `Associators` / `References` | **Get**（透過 CIM Binding 延伸） | 關聯遍歷 |
| `Invoke`（自訂方法） | **Invoke** | 呼叫 CIM Schema 定義的方法 |

### CIM Binding Schema（DSP8019）

DSP0227 的具體 XML 型別定義於 **DSP8019（CIM Binding Schema）**，其命名空間為 `http://schemas.dmtf.org/wbem/wsman/1/cimbinding.xsd`[^dsp8019-xsd]。

這個 Schema 定義了幾個重要的擴充元素：

- **`PolymorphismMode`** — 列舉控制多型行為的三種模式：`IncludeSubClassProperties`（包含子類別屬性）、`ExcludeSubClassProperties`（排除子類別屬性）、`None`（無多型）
- **`AssociatedInstances`** — 用於查詢關聯物件的複雜型別，支援依關聯類別、角色、結果類別進行過濾
- **`CIMException`** — SOAP Fault 的子代碼，用於傳遞 CIM 特定的錯誤資訊
- **`Capability_FaultIncludesCIMError`** — 能力宣告，表示伺服端能在錯誤回應中附帶 `CIM_Error` 物件

這份 Schema 自 1.0.0（2009-10）歷經 1.1.0（2010-02）與 1.2.0（2011-03）兩個版本的演進，逐步加入了 `IsCIM_Error` 屬性以及 Class Metadata Path 的能力宣告。

---

## DSP0230：WS-CIM Mapping

如果說 DSP0227 解決了「CIM 如何透過 WS-Management 傳輸」的問題，那麼 **DSP0230（WS-CIM Mapping Specification）** 解決的就是更底層的問題：「CIM 的類別、屬性、方法、關聯、繼承……這些抽象概念，要如何精確地翻譯成 XML Schema 與 WSDL？」[^dmtf-dsp0230]。

### 映射演算法

DSP0230 定義了一套完整的映射規則：

1. **CIM 類別 → XSD complexType**：每個 CIM 類別對映到一個 XML Schema 的 `complexType`，類別名稱即為型別名稱。

2. **CIM 繼承 → XSD extension**：CIM 的類別繼承層次透過 XML Schema 的 `xs:extension` 機制來對映——子類別擴充（extend）父類別的 complexType。

3. **CIM 屬性 → XSD 元素**：每個屬性成為 child element，使用 WS-CIM Common Schema（DSP8004）定義的擴充資料型別[^common-xsd]：

| CIM 資料型別 | WS-CIM XSD 型別 |
|-------------|----------------|
| `string` | `cim:cimString`（繼承 `xs:string`） |
| `unsigned16` | `cim:cimUnsignedShort`（繼承 `xs:unsignedShort`） |
| `unsigned32` | `cim:cimUnsignedInt`（繼承 `xs:unsignedInt`） |
| `boolean` | `cim:cimBoolean`（繼承 `xs:boolean`） |
| `datetime` | `cim:cimDateTime`（五選一結構） |
| `reference` | `cim:cimReference` |
| `real32` | `cim:cimFloat`（繼承 `xs:float`） |

4. **CIM 方法 → WSDL 操作**：CIM 類別中定義的方法對映到 WSDL 操作，輸入與輸出訊息包含方法的參數與回傳值。

5. **CIM 命名空間 → XSD targetNamespace**：CIM 命名空間（如 `root/cimv2`）對映為 URI 形式的 `targetNamespace`。

### 多型物件的 XML 表示

假設我們有一個 `CIM_LogicalDisk` 實例（繼承自 `CIM_StorageDrive`），在 WS-CIM XML 中的表示可能如下：

```xml
<CIM_LogicalDisk xsi:type="CIM_LogicalDisk"
                 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                 xmlns:cim="http://schemas.dmtf.org/wbem/wscim/1/common">
  <cim:Name cim:Key="true">C:</cim:Name>
  <cim:DeviceID>disk_0_c</cim:DeviceID>
  <!-- 從父類別繼承的屬性 -->
  <cim:OperationalStatus>2</cim:OperationalStatus>  
  <!-- 子類別專屬屬性 -->
  <cim:FileSystemType>NTFS</cim:FileSystemType>
  <cim:VolumeSizeInBytes>536870912000</cim:VolumeSizeInBytes>
</CIM_LogicalDisk>
```

注意 `cim:Key="true"` 標記了屬性的識別角色，`xsi:type` 則記錄了實際的類別以支援多型。

---

## DASH/SMASH 為何選擇 WS-Management？

現在我們已經理解了 WS-Management 的完整技術架構，可以回答一個有趣的問題：**為什麼 DASH（桌上型管理架構）與 SMASH（伺服器管理架構）都選擇 WS-Management 作為通訊協定，而不是 CIM-XML？**

原因可以歸納為以下幾點[^dmtf-wsman][^wiki-dash]：

| 原因 | 說明 |
|------|------|
| **既有標準基礎** | SOAP/WSDL/WS-Addressing 是 2000 年代後期的企業標準，不需重新發明車輪 |
| **ISO 標準化** | WS-Management 是唯一被 ISO/IEC 17963:2013 採納的 WBEM 協定——對於需要遵循國際標準的企業環境至關重要 |
| **安全分離** | WS-Management 將協定與傳輸/加密分離（Decoupled），可以獨立於 TLS 版本而演進——對頻外（Out-of-Band）管理的安全需求尤為重要 |
| **事件訂閱完整** | 完整的 WS-Eventing 機制，支援 Subscribe/Renew/Unsubscribe/SubscriptionEnd 的完整生命週期 |
| **擴充性** | 透過 WSDL 與 XML Schema 的型別繼承機制，DASH 與 SMASH 可以自然擴充 CIM Profile |
| **防火牆友善** | 使用標準 HTTP/HTTPS 埠（80/443）或 IANA 分配的埠（5985/5986），不需開通特殊埠 |
| **大規模管理** | Enumerate/Pull 的兩階段列舉模式，適合資料中心成千臺伺服器的大規模管理場景 |

有趣的是，CIM-XML 雖然在 Linux（OpenPegasus）與 Python（pywbem）生態系中最為活躍，但 DASH/SMASH 所針對的企業硬體管理場景，恰恰需要 WS-Management 所提供的那些企業級特性——ISO 標準化、安全分離、完整事件管理。而 **Intel AMT**（Active Management Technology）與 **Microsoft WinRM**（Windows Remote Management）分別代表了 DASH 與企業伺服器管理領域中最具代表性的 WS-Management 實作[^wikipedia-winrm]。

---

## 小結

1. **WS-Management（DSP0226 v1.2.0）** 是 WBEM 家族中以 SOAP/WSDL 為基礎的通訊協定，也是唯一被 ISO/IEC 17963:2013 採納為國際標準的 WBEM 協定。
2. **三類核心操作** 涵蓋了管理通訊的全部需求：Resource Access（CRUD）、Enumeration（大量資料列舉）、Notification（事件訂閱管理）。
3. **DSP0227（CIM Binding）** 定義了 CIM 類別、屬性、方法與 WS-Management 操作之間的對應關係，使 CIM 模型能夠透過 WS-Management 的 SOAP 訊息來傳輸。
4. **DSP0230（WS-CIM Mapping）** 定義了從 CIM Metamodel 到 XML Schema 與 WSDL 的映射演算法，使 CIM 的物件導向概念（類別、繼承、屬性、方法）能夠精確翻譯為 Web Services 的型別系統。
5. **DASH 與 SMASH** 選擇 WS-Management 而非 CIM-XML，主要是因為其 ISO 標準地位、安全分離設計、完整事件管理與大規模管理能力。

明天，我們將探索 WBEM 家族中最年輕的成員——**CIM-RS（RESTful 管理 API）**，看看 DMTF 如何將現代 RESTful 設計原則應用到管理通訊中。

[^dmtf-iso]: Distributed Management Task Force. (2013, February 26). DMTF Standards Gain Global Recognition — WS-Management adopted as ISO/IEC 17963:2013. Retrieved 2026-10-03, from https://www.dmtf.org/content/dmtf-standards-gain-global-recognition
[^dmtf-wsman]: Distributed Management Task Force. (n.d.). WS-Management Standards. Retrieved 2026-10-03, from https://www.dmtf.org/standards/wsman
[^wikipedia-wsman]: Wikipedia. (n.d.). WS-Management. Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/WS-Management
[^dmtf-wsman-resource]: Distributed Management Task Force. (n.d.). DSP8035 — Resource Access Operations WSDL. Retrieved 2026-10-03, from http://schemas.dmtf.org/wbem/wsman/1/dsp8035_1.0.wsdl
[^dsp8035-wsdl]: Distributed Management Task Force. (n.d.). Resource Operations Port Type WSDL. Retrieved 2026-10-03, from http://schemas.dmtf.org/wbem/wsman/1/dsp8035_1.0.wsdl
[^dmtf-wsman-enum]: Distributed Management Task Force. (n.d.). DSP8037 — Enumeration Operations WSDL. Retrieved 2026-10-03, from http://schemas.dmtf.org/wbem/wsman/1/dsp8037_1.0.wsdl
[^dmtf-wsman-event]: Distributed Management Task Force. (n.d.). DSP8036 — Notification Operations WSDL. Retrieved 2026-10-03, from http://schemas.dmtf.org/wbem/wsman/1/dsp8036_1.0.wsdl
[^wsman-identity]: Distributed Management Task Force. (n.d.). WS-Management Identity XML Schema. Retrieved 2026-10-03, from http://schemas.dmtf.org/wbem/wsman/identity/1/wsmanidentity.xsd
[^wiki-dash]: Wikipedia. (n.d.). Desktop and mobile Architecture for System Hardware (DASH). Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Desktop_and_mobile_Architecture_for_System_Hardware
[^dmtf-dsp0227]: Distributed Management Task Force. (n.d.). DSP0227 — WS-Management CIM Binding Specification. Retrieved 2026-10-03, from https://www.dmtf.org/dsp/DSP0227
[^dsp8019-xsd]: Distributed Management Task Force. (n.d.). DSP8019 — CIM Binding Schema v1.2.0. Retrieved 2026-10-03, from http://schemas.dmtf.org/wbem/wsman/1/dsp8019.xsd
[^pywbem-operations]: pywbem Development Team. (n.d.). pywbem — WBEM Operations. Retrieved 2026-10-03, from https://pywbem.readthedocs.io/en/latest/client/operations.html
[^dmtf-dsp0230]: Distributed Management Task Force. (n.d.). DSP0230 — WS-CIM Mapping Specification. Retrieved 2026-10-03, from https://www.dmtf.org/dsp/DSP0230
[^common-xsd]: Distributed Management Task Force. (n.d.). DSP8004 — WS-CIM Common Schema. Retrieved 2026-10-03, from http://schemas.dmtf.org/wbem/wscim/1/common.xsd
[^wikipedia-winrm]: Wikipedia. (n.d.). Windows Remote Management (WinRM). Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Windows_Remote_Management