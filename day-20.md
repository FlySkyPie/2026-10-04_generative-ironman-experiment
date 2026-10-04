# Day 20：CIM-RS — 現代化 RESTful 管理 API

昨天我們探討了 WS-Management——以 SOAP/Web Services 為基礎的企業級管理協定。如果說 CIM-XML 是 WBEM 的「經典語言」、WS-Management 是「國際標準語言」，那麼今天的主角 **CIM-RS（CIM RESTful Services）** 就是 WBEM 家族的「現代語言」。

CIM-RS 是 WBEM 三協定中最年輕的成員。它的核心規格 **DSP0210（CIM-RS Protocol）** 首版於 **2013 年 2 月** 發佈，最新版本 **2.0.0** 於 **2015 年 3 月 10 日** 標準化。與之配套的 **DSP0211（CIM-RS Payload Representation in JSON）** 也在同日達到 v2.0.0[^dmtf-dsp0210][^dmtf-dsp0211]。

CIM-RS 的誕生反映了管理標準世界的一次重要轉向：從 XML-RPC 風格（CIM-XML）與 SOAP 風格（WS-Management），轉向 **RESTful 架構風格**，並選擇 **JSON** 作為主要資料交換格式[^dmtf-wbem]。

## CIM-RS 的設計哲學

CIM-RS 與前兩支協定的根本差異在於它對 REST 原則的堅持：

| 面向 | CIM-XML | WS-Management | CIM-RS |
|------|---------|---------------|--------|
| **通訊模式** | RPC（自訂 XML 格式） | SOAP（標準 SOAP Envelope） | **REST（Representational State Transfer）** |
| **HTTP 方法** | 僅 POST（所有語意在 XML 中） | 僅 POST（SOAP over HTTP） | **GET / POST / PUT / DELETE** |
| **酬載格式** | XML（自訂 DTD） | XML（SOAP + XML Schema） | **JSON（DSP0211）或 XML** |
| **資源定址** | URI Path | URI Path | **URI Path（標準 REST 風格）** |
| **狀態管理** | 無狀態（Stateless） | 有狀態（Subscription、Enumeration） | **無狀態（Stateless）** |
| **學習曲線** | 中 | 高 | **低** |
| **防火牆友善** | 中（需自訂埠 5988/5989） | 高（標準 HTTP/HTTPS） | **高（標準 HTTP/HTTPS）** |

CIM-RS 的設計目標很明確：**讓任何熟悉 HTTP 與 REST 的開發者都能立即上手**，無需學習自訂 XML 格式（CIM-XML）或 SOAP/WSDL 複雜堆疊（WS-Management）[^dmtf-stackmap]。

## 規格家族：DSP0210 與 DSP0211

CIM-RS 由兩份核心規格構成，加上一系列先導文件（Informational Documents）[^dmtf-wbem]：

| 文件編號 | 名稱 | 最新版本 | 發佈日期 | 狀態 |
|---------|------|---------|---------|------|
| **DSP0210** | CIM-RS Protocol Specification | 2.0.0 | 2015-03-10 | Standard |
| **DSP0211** | CIM-RS Payload Representation in JSON | 2.0.0 | 2015-03-10 | Standard |
| DSP-IS0201 | CIM Operations Over RESTful Services | 1.0.0 | 2011-05-16 | Informational |
| DSP-IS0202 | CIM-RS Binding to JSON | 1.0.0 | 2011-05-16 | Informational |
| DSP-IS0203 | CIM-RS White Paper | 1.0.0 | 2011-11-07 | Informational |

值得注意的是，CIM-RS 的標準化歷程始於三份 Informational 文件。DSP-IS0201 於 2011 年先以白皮書形式提出「CIM Operations Over RESTful Services」的概念，經過兩年醞釀後才正式以 DSP0210 標準化。DSP-IS0202 與 DSP-IS0203 則分別驗證了 JSON 綁定可行性與整體架構[^dmtf-published-docs]。

### 版本演進

DSP0210 的版本歷史[^dmtf-dsp0210]：

| 版本 | 發佈日期 | 說明 |
|------|---------|------|
| **2.0.0** | **2015-03-10** | **最新標準版本**，顯著改進 RESTful 操作定義 |
| 1.0.1 | 2014-02-11 | 小幅修正 |
| 1.0.0 | 2013-02-04 | 首版標準 |

DSP0211 的版本歷史[^dmtf-dsp0211]：

| 版本 | 發佈日期 | 說明 |
|------|---------|------|
| **2.0.0** | **2015-03-10** | **最新標準版本**，完善 JSON 酬載編碼規則 |
| 1.0.1 | 2014-02-11 | 小幅修正 |
| 1.0.0 | 2013-02-04 | 首版標準 |

DSP0210 與 DSP0211 一直保持著同步的版本號——這並非巧合，而是因為兩份規格在設計上緊密耦合：協定定義「如何操作」，JSON 酬載定義「如何表達」。

---

## HTTP 方法與 CIM 操作的對映

CIM-RS 的核心創舉在於將標準 HTTP 方法直接對映到 CIM 操作。讓 RESTful CRUD 模式與管理操作無縫銜接[^wiki-cim]：

| HTTP 方法 | CIM-RS 操作 | 等同的 CIM-XML 操作 | URI 目標 |
|-----------|-------------|---------------------|---------|
| **GET** | **Retrieve** | `EnumerateInstances`、`GetInstance` | 類別 URI（列舉）或實例 URI（讀取） |
| **POST** | **Create / Invoke** | `CreateInstance`、`Invoke`（自訂方法） | 類別 URI（建立）或實例 URI（方法呼叫） |
| **PUT** | **Modify / Update** | `ModifyInstance`、`SetProperty` | 實例 URI（更新屬性） |
| **DELETE** | **Delete** | `DeleteInstance` | 實例 URI（刪除） |

### URI 定址模式

CIM-RS 使用 WBEM URI Mapping（DSP0207）作為物件定位的基礎，並採用 RESTful 風格的 URI 模式[^dmtf-uri-mapping]：

```
/{namespace}/{classname}                     — 指向某個類別的所有實例
/{namespace}/{classname}/{instanceId}        — 指向某個特定的實例
/{namespace}/{classname}/{instanceId}/{property} — 指向某個實例的特定屬性
```

實際範例：

```
/cim/CIM_ComputerSystem                       — 所有電腦系統實例
/cim/CIM_ComputerSystem/svr-01                — 名為 svr-01 的電腦系統
/cim/CIM_ComputerSystem/svr-01/OperationalStatus — svr-01 的 OperationalStatus 屬性
```

這裡的命名空間前綴（如 `cim`）由 DSP0207 規範，類別名稱直接來自 CIM Schema，實例 ID 為該類別的 Key Property（通常是 Name）的值。

### 完整請求/回應範例

以下展示 CIM-RS 在 JSON 下的四種基本操作。

**GET：列舉所有 `CIM_LogicalDisk` 實例**

```http
GET /cim/CIM_LogicalDisk HTTP/1.1
Host: wbem-server.example.com
Accept: application/json
```

**回應**：

```json
{
  "CIM_LogicalDisk": [
    { "Name": "C:", "DeviceID": "disk_0_c", "OperationalStatus": 2, "FileSystemType": "NTFS" },
    { "Name": "D:", "DeviceID": "disk_0_d", "OperationalStatus": 2, "FileSystemType": "NTFS" }
  ]
}
```

---

**GET：查詢特定 `CIM_ComputerSystem` 實例**

```http
GET /cim/CIM_ComputerSystem/svr-01 HTTP/1.1
Host: wbem-server.example.com
Accept: application/json
```

**回應**（HTTP 200 OK）：

```json
{
  "Name": "svr-01",
  "ElementName": "Production Web Server",
  "Description": "Main application server",
  "OperationalStatus": 2,
  "HealthState": 5,
  "ChassisType": 23,
  "CreationClassName": "CIM_ComputerSystem",
  "ResetCount": 3
}
```

---

**PUT：修改實例屬性**

```http
PUT /cim/CIM_ComputerSystem/svr-01 HTTP/1.1
Host: wbem-server.example.com
Content-Type: application/json

{
  "Description": "Updated: Main application server (v2)",
  "ResetCount": 0
}
```

**回應**：HTTP 200 OK（成功更新），或 HTTP 204 No Content（伺服端選擇不回傳內容）。

---

**POST：建立新實例**

```http
POST /cim/CIM_ComputerSystem HTTP/1.1
Host: wbem-server.example.com
Content-Type: application/json

{
  "Name": "svr-02",
  "ElementName": "Backup Server",
  "ChassisType": 23
}
```

**回應**：HTTP 201 Created，`Location` 標頭指向新資源 URI：`/cim/CIM_ComputerSystem/svr-02`

---

**DELETE：刪除實例**

```http
DELETE /cim/CIM_ComputerSystem/svr-02 HTTP/1.1
Host: wbem-server.example.com
```

**回應**：HTTP 200 OK 或 HTTP 204 No Content。

---

**POST：呼叫自訂方法（Invoke）**

```http
POST /cim/CIM_ComputerSystem/svr-01/Reboot HTTP/1.1
Host: wbem-server.example.com
Content-Type: application/json

{
  "ForceReboot": true,
  "DelaySeconds": 60
}
```

**回應**：

```json
{
  "result": "Operation scheduled",
  "estimatedCompletion": "2026-10-03T14:01:00Z"
}
```

這種「POST MethodCall」的模式，是 CIM-RS 保留的唯一 RPC 風格痕跡——它對應於 CIM 類別中定義的 Extrinsic Methods。

---

## DSP0211：JSON 酬載格式

DSP0211 定義了 CIM 物件的 JSON 表示法。它與 DSP0210 緊密合作，後者定義如何操作，前者定義操作什麼。其核心編碼規則[^dmtf-dsp0211]：

### 基本編碼規則

1. **屬性名稱** — 直接對應 CIM Schema 中的 Property 名稱，大小寫敏感。例如 `OperationalStatus`、`ElementName`、`HealthState`。

2. **屬性值** — 使用 JSON 原生型別：`string`（字串）、`number`（數值）、`boolean`（布林值）、`null`（空值）。列舉值以數值表示。

3. **Key 屬性** — 當屬性是該類別的識別鍵（Key Property）時，在請求/回應中會用來定位實例。例如 `CIM_ComputerSystem` 的 Key 是 `Name`。

4. **關聯（Association）** — 以 JSON 物件或陣列表示，包含 `@ref` 指向被引用實例的 URI：

```json
{
  "Name": "svr-01",
  "RunningProcesses": [
    { "@ref": "/cim/CIM_Process/pid-1234" },
    { "@ref": "/cim/CIM_Process/pid-5678" }
  ]
}
```

5. **多型（Polymorphism）** — 透過 `@type` 屬性記錄實際類別：

```json
{
  "@type": "CIM_LogicalDisk",
  "Name": "C:",
  "FileSystemType": "NTFS",
  "VolumeSizeInBytes": 536870912000
}
```

### 錯誤回應格式

當操作失敗時，CIM-RS 使用標準 HTTP 狀態碼並在 JSON 主體中附帶錯誤細節：

```json
{
  "CIM_Error": {
    "ErrorCode": 4,
    "ErrorDescription": "InstanceNotFound",
    "ErrorMessage": "CIM_ComputerSystem with Name 'svr-99' not found."
  }
}
```

常見的 HTTP 狀態碼與對應語意：

| HTTP 狀態碼 | 語意 |
|------------|------|
| **200 OK** | GET/PUT/DELETE 成功，回應主體包含結果 |
| **201 Created** | POST 建立成功，Location 標頭含新資源 URI |
| **204 No Content** | 操作成功但無回應主體 |
| **400 Bad Request** | 請求格式錯誤或參數不正確 |
| **404 Not Found** | 指定的資源不存在 |
| **405 Method Not Allowed** | 該 URI 不支援此 HTTP 方法 |
| **500 Internal Server Error** | 伺服端內部錯誤 |

---

## 過濾查詢：CQL 與 Filter Query Language

CIM-RS 支援過濾查詢，允許用戶端在列舉時指定條件來縮小結果範圍。這項能力來自兩個規格[^dmtf-wbem]：

- **DSP0202（CIM Query Language，CQL）** — 宣告式查詢語言，類似 SQL 風格的過濾條件
- **DSP0212（Filter Query Language）** — 用於 URI 查詢參數的輕量級過濾語法

在 CIM-RS 中，過濾條件透過查詢參數傳遞：

```http
GET /cim/CIM_ComputerSystem?filter=OperationalStatus=2&healthState>=5 HTTP/1.1
Host: wbem-server.example.com
Accept: application/json
```

這讓用戶端可以在網路層級進行過濾，減少不必要的資料傳輸——這在管理大量裝置（如資料中心）時尤其重要。

---

## 三協定的使用場景與取捨

經過四天的深入探索，現在我們可以給出一個全面的比較：

| 面向 | CIM-XML | WS-Management | CIM-RS |
|------|---------|---------------|--------|
| **最佳適用場景** | Linux 生態系、OpenPegasus 環境 | Windows 環境（WinRM）、需要 ISO 標準 | 現代 RESTful 架構、JSON 生態系 |
| **生態系濃度** | Linux（SBLIM、OpenPegasus、pywbem） | Windows（WinRM、Intel AMT） | 較小（雖被規範但實作有限） |
| **主要實作** | OpenPegasus、pywbem、SBLIM | Microsoft WinRM、Intel AMT | DASH/SMASH 中的參考實作 |
| **互通性標準** | DMTF 專屬 | ISO/IEC 17963:2013 | DMTF 專屬（但 RESTful 原則本身高度標準化） |
| **開發者親和性** | 中等 | 低（需 SOAP/WSDL 知識） | **高**（只需 HTTP + JSON） |
| **事件/通知機制** | Export Messages（基本） | WS-Eventing（完整訂閱管理） | 依賴標準 HTTP 機制（Webhooks 等） |
| **防火牆穿透** | 需開通 5988/5989 | 80/443 標準埠 | 80/443 標準埠 |

一個耐人尋味的觀察：雖然 CIM-RS 在技術上最現代化，其生態系卻不如 CIM-XML 與 WS-Management 豐富。原因是 DMTF 在 2015 年之後將其標準化重心從「通訊協定」轉向「垂直應用領域」（Initiatives）——特別是 DASH 與 SMASH[^dmtf-stackmap]。這些 Initiatives 多數基於 WS-Management（因為後者是 ISO 標準且已在企業環境廣泛部署），而非 CIM-RS。

然而，CIM-RS 的設計理念——**以 RESTful 原則管理 CIM 資源**——從未消失。事實上，DMTF 後來推出的 **Redfish** 規範（基於 RESTful 設計）採用了非常類似 CIM-RS 的哲學，只是將其應用於資料中心管理這個更具商業吸引力的領域[^dmtf-redfish]。

---

## 小結

1. **CIM-RS 是 WBEM 家族中最現代化的協定**，以 RESTful 架構風格取代了 CIM-XML 的 RPC 風格與 WS-Management 的 SOAP 風格。

2. **DSP0210 v2.0.0（2015-03-10）與 DSP0211 v2.0.0 構成 CIM-RS 的核心**，分別規範通訊協定與 JSON 酬載格式。

3. **HTTP 方法直接對映 CIM 操作**：GET 用於列舉/讀取、PUT 用於更新、POST 用於建立與方法呼叫、DELETE 用於刪除。

4. **JSON 是 CIM-RS 的主要酬載格式**，支援原生型別、關聯引用（`@ref`）、多型（`@type`）以及標準化錯誤格式。

5. **CIM-RS 的生態系較小**，但其 RESTful 設計理念在 DMTF 後續的 Redfish 規範中得到了延續與發揚。

到目前為止，我們已經完成了第四階段「WBEM 通訊協定」的全部四天內容（Day 17–20）。明天開始的第五階段，我們將進入 **Profiles 與真實實作**——探討 CIM 如何透過 Management Profiles 為特定場景量身定做，以及 OpenPegasus、pywbem、WMI 等真實世界的 CIM 實作。

[^dmtf-dsp0210]: Distributed Management Task Force. (2015, March 10). DSP0210 — CIM-RS Protocol Specification (v2.0.0). Retrieved 2026-10-03, from https://www.dmtf.org/dsp/DSP0210
[^dmtf-dsp0211]: Distributed Management Task Force. (2015, March 10). DSP0211 — CIM-RS Payload Representation in JSON (v2.0.0). Retrieved 2026-10-03, from https://www.dmtf.org/dsp/DSP0211
[^dmtf-wbem]: Distributed Management Task Force. (n.d.). WBEM Standards — Web-Based Enterprise Management. Retrieved 2026-10-03, from https://www.dmtf.org/standards/wbem
[^dmtf-stackmap]: Distributed Management Task Force. (n.d.). DMTF Technologies Stack Map. Retrieved 2026-10-03, from https://www.dmtf.org/standards/stackmap
[^dmtf-published-docs]: Distributed Management Task Force. (n.d.). All DMTF Standard Publications. Retrieved 2026-10-03, from https://www.dmtf.org/standards/published_documents
[^wiki-cim]: Wikipedia. (n.d.). Common Information Model (computing). Retrieved 2026-10-03, from https://en.wikipedia.org/wiki/Common_Information_Model_(computing)
[^dmtf-uri-mapping]: Distributed Management Task Force. (n.d.). DSP0207 — WBEM URI Mapping (v1.0.1). Retrieved 2026-10-03, from https://www.dmtf.org/standards/wbem
[^dmtf-redfish]: Distributed Management Task Force. (n.d.). Redfish — Redfish RESTful API. Retrieved 2026-10-03, from https://www.dmtf.org/standards/redfish