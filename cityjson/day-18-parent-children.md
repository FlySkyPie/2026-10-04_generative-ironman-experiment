# Day 18｜Parent/Children 關係 — 用平面結構表達樹狀階層

## 前言

在 Day 13 我們學到了城市物件分為第一層級（可獨立存在）與第二層級（必須依附父物件），Day 14 到 Day 16 逐一認識了所有 33 種物件，Day 17 則深入探討了語意表面的 `surfaces` + `values` 機制與其中的 parent/children 關係。

今天是 Phase 3 的最後一篇，我們要把焦點拉回一個貫穿所有這些主題的核心機制：**Parent/Children 關係**。

你可能已經注意到，CityJSON 中至少有兩個地方出現了 parent/children 的概念：

1. **城市物件層級**：`Building` 有 `children`，`BuildingPart` 有 `parents`——這是物件間的階層關係
2. **語意表面層級**：`WallSurface` 有 `children` 指向 `Window`，`Window` 有 `parent` 指向 `WallSurface`——這是幾何基本單元間的從屬關係

這兩者運作在不同的抽象層級，但共享相同的設計哲學：**用平面的 JSON 陣列與索引來表達樹狀的階層結構**。

---

## 從巢狀 XML 到平面 JSON 的思維轉變

要理解 Parent/Children 機制為什麼存在，必須先理解 CityJSON 與其前身 CityGML 的根本差異。

CityGML 使用 XML，天然支援巢狀結構。在 XML 中，一個 Building 包含 BuildingPart，BuildingPart 包含 BuildingInstallation，可以直接用元素的嵌套來表達[^citygml-nesting]：

```xml
<Building gml:id="id-1">
  <BuildingPart gml:id="id-2">
    <BuildingInstallation gml:id="id-3"/>
  </BuildingPart>
</Building>
```

這種寫法直觀易懂——父子關係由文件的物理結構決定。但巢狀結構也有代價：當你需要查詢「某某安裝設備屬於哪棟建築物」時，必須沿著 DOM 樹往上爬；當你想取得某個範圍內的所有建築物時，必須對巢狀結構做深層走訪（deep traversal）。

CityJSON 選擇了完全不同的路徑：**把所有物件展平到一個 `CityObjects` 字典中，用 ID 來索引，用 `parents` 和 `children` 陣列來重建階層**。XML 的巢狀結構被轉化為一個雙向鏈結的圖結構[^cityjson-spec]：

```json
"CityObjects": {
  "id-1": {
    "type": "Building",
    "children": ["id-2"]
  },
  "id-2": {
    "type": "BuildingPart",
    "parents": ["id-1"],
    "children": ["id-3"]
  },
  "id-3": {
    "type": "BuildingInstallation",
    "parents": ["id-2"]
  }
}
```

這樣做的好處是：

1. **查詢效率高**——要找到某個物件的父物件，直接讀 `parents` 陣列即可，不需遍歷整個文件
2. **資料庫友善**——展平後的結構更容易匯入關聯式資料庫（一張 `CityObjects` 表即可）
3. **部分讀取可行**——你可以只載入感興趣的物件，不需要完整的巢狀結構
4. **避免重複**——如果一個物件有多個父物件（Extension 允許），巢狀結構無法表達，但 `parents` 陣列可以

---

## `parents` — 第二層級物件的必備屬性

第二層級城市物件（2nd-level City Object）**必須**擁有 `parents` 成員[^cityjson-spec]。其值是一個字串陣列，包含父物件的 ID：

```json
{
  "type": "BuildingPart",
  "parents": ["id-1"]
}
```

在 CityJSON 核心模組中，所有第二層級物件的 `parents` 陣列大小永遠為 1——一個 BuildingPart 只會有一個父 Building。但 CityJSON Extension 可以定義擁有多個父物件的類型。例如一個「共用牆壁」的 `+SharedWall` 物件可能同時是兩棟建築的子物件。

哪些類型是第二層級？以下總表涵蓋所有 17 種[^cityjson-spec]：

| 模組 | 第二層級類型 | 可作為父物件 |
|------|-------------|-------------|
| Building | `BuildingPart` | Building |
| Building | `BuildingInstallation` | Building, BuildingPart, BuildingRoom |
| Building | `BuildingConstructiveElement` | Building, BuildingPart, BuildingRoom |
| Building | `BuildingFurniture` | Building, BuildingPart, BuildingRoom |
| Building | `BuildingStorey` | Building, BuildingPart, BuildingRoom |
| Building | `BuildingRoom` | Building, BuildingPart |
| Building | `BuildingUnit` | Building, BuildingPart |
| Bridge | `BridgePart` | Bridge |
| Bridge | `BridgeInstallation` | Bridge, BridgePart, BridgeRoom |
| Bridge | `BridgeConstructiveElement` | Bridge, BridgePart, BridgeRoom |
| Bridge | `BridgeRoom` | Bridge, BridgePart |
| Bridge | `BridgeFurniture` | Bridge, BridgePart, BridgeRoom |
| Tunnel | `TunnelPart` | Tunnel |
| Tunnel | `TunnelInstallation` | Tunnel, TunnelPart |
| Tunnel | `TunnelConstructiveElement` | Tunnel, TunnelPart |
| Tunnel | `TunnelHollowSpace` | Tunnel, TunnelPart |
| Tunnel | `TunnelFurniture` | Tunnel, TunnelPart |

---

## `children` — 只有直系子物件

第一層級物件的 `children` 陣列列出其**直接子物件**的 ID。關鍵規則：**只包含直系子級，不包含孫級**[^cityjson-spec]。

```json
{
  "id-1": {
    "type": "Building",
    "children": ["id-2"]
  },
  "id-2": {
    "type": "BuildingPart",
    "parents": ["id-1"],
    "children": ["id-3"]
  },
  "id-3": {
    "type": "BuildingInstallation",
    "parents": ["id-2"]
  }
}
```

在這個範例中：
- `id-1`（Building）的 `children` 只有 `["id-2"]`，不包含 `id-3`（孫級）
- `id-2`（BuildingPart）的 `children` 有 `["id-3"]`
- `id-3`（BuildingInstallation）是第二層級終端，沒有 `children`

為什麼不包含孫級？這是因為如果 `children` 包含所有後代（deep children），會產生資訊冗餘——當 BuildingPart 改隸屬於另一個 Building 時，原始 Building 的 deep children 陣列全部都得更新。只儲存直系子物件讓關係維護變得簡單且一致。

### 取得完整葉節點的方法

如果你需要取得某個物件的所有後代（包括孫級），就必須進行遞迴走訪。以下是一個簡單的虛擬碼：

```
function getAllDescendants(id, CityObjects):
    descendants = []
    obj = CityObjects[id]
    if obj has "children":
        for childId in obj.children:
            descendants.append(childId)
            // 遞迴：子物件的後代也是該物件的後代
            descendants.extend(getAllDescendants(childId, CityObjects))
    return descendants
```

### children 的排序

`children` 陣列中元素的**順序不重要**（not relevant）[^cityjson-spec]。程式讀取時不應該依賴特定的排列順序。這也符合 JSON 物件鍵值無序的特性——如果你需要排序，應該根據業務需求自行排列。

---

## 不同物件家族的分支結構

Parent/Children 的具體拓撲會因物件家族而異。以下是三個主要家族的結構對比：

### 建築物家族

```
Building (1st-level)
├── BuildingPart (2nd-level) ─── 可包含更多子物件
│   ├── BuildingInstallation
│   ├── BuildingConstructiveElement
│   ├── BuildingFurniture
│   ├── BuildingStorey
│   └── BuildingRoom
│       ├── BuildingInstallation
│       ├── BuildingConstructiveElement
│       └── BuildingFurniture
├── BuildingInstallation (2nd-level)
├── BuildingConstructiveElement (2nd-level)
├── BuildingFurniture (2nd-level)
├── BuildingStorey (2nd-level)
├── BuildingRoom (2nd-level)
│   ├── BuildingInstallation
│   ├── BuildingConstructiveElement
│   └── BuildingFurniture
└── BuildingUnit (2nd-level)
```

建築物是唯一同時擁有 **8 種第二層級物件**的家族，分支深度可達三層（Building → BuildingPart → BuildingRoom → BuildingFurniture）[^cityjson-spec]。

### 橋樑家族

```
Bridge (1st-level)
├── BridgePart (2nd-level)
│   ├── BridgeInstallation
│   ├── BridgeConstructiveElement
│   ├── BridgeRoom
│   │   └── BridgeFurniture
│   └── BridgeFurniture
├── BridgeInstallation (2nd-level)
├── BridgeConstructiveElement (2nd-level)
├── BridgeRoom (2nd-level)
│   └── BridgeFurniture (2nd-level)
└── BridgeFurniture (2nd-level)
```

### 隧道家族

```
Tunnel (1st-level)
├── TunnelPart (2nd-level)
│   ├── TunnelInstallation
│   ├── TunnelConstructiveElement
│   ├── TunnelHollowSpace
│   └── TunnelFurniture
├── TunnelInstallation (2nd-level)
├── TunnelConstructiveElement (2nd-level)
├── TunnelHollowSpace (2nd-level)
└── TunnelFurniture (2nd-level)
```

三大家族的共同模式：**Part 是唯一的第二層級樞紐**，它可以容納所有其他類型的子物件，而 Installation、ConstructiveElement、Furniture 通常是終端節點（沒有 `children`）。

---

## 語意中的 Parent/Children — 兩種層級的對比

Day 17 已經詳細介紹了語意表面中的 parent/children：

```json
"semantics": {
  "surfaces": [
    {
      "type": "WallSurface",   // index 0
      "children": [2]
    },
    {
      "type": "RoofSurface",   // index 1
      "slope": 66.6
    },
    {
      "type": "Window",        // index 2
      "parent": 0
    }
  ],
  "values": [0, 0, null, 1, 2]
}
```

現在我們已經看過兩種 parent/children 機制，下表可以幫助釐清兩者的差異：

| 面向 | 城市物件層級 | 語意表面層級 |
|------|-------------|-------------|
| 應用範圍 | `CityObjects` 字典中的物件之間 | 單一 Geometry 內的 `surfaces` 陣列 |
| 識別方式 | 字串 ID（如 `"id-1"`） | 整數索引（0-based） |
| 必填性 | 第二層級物件的 `parents` 必填 | `parent` / `children` 皆為選填 |
| 父物件數量 | 核心模組中為 1（Extension 可多個） | 永遠為 1 |
| 代表意義 | 組合/從屬（Building → BuildingPart） | 包含/開口（Wall → Window） |
| 關鍵成員 | 父：`children`、子：`parents` | 父：`children`、子：`parent`（注意無 s） |

兩個層級雖然名稱相似，但互不相干——城市物件的 parent/children 不會影響語意表面的 parent/children，反之亦然。你可以有一個 Building 包含 BuildingPart（城市物件層級），而這些物件的幾何可以各自擁有完全獨立的語意 parent/children 關係。

---

## 實務：視覺化工具如何使用 Parent/Children

了解 Parent/Children 機制後，來看看視覺化與分析工具實際上是如何利用這項資訊的。

### 樹狀瀏覽

大多數 CityJSON 檢視器（如 [cityjson.org](https://www.cityjson.org) 上的線上檢視器、`ninja` 或 `azul`）會利用 parent/children 關係建立樹狀瀏覽面板，讓使用者可以展開或收起建築物的內部結構：

```
📋 City Objects
├── 🏠 Building - id-1
│   ├── 🧱 BuildingPart - id-2
│   │   ├── 🪟 BuildingInstallation - id-3（陽台）
│   │   └── 🚪 BuildingRoom - id-4（房間）
│   └── 🏗️ BuildingInstallation - id-5（外牆設備）
├── 🛣️ Road - id-6
└── 🌳 SolitaryVegetationObject - id-7
```

### 選擇性顯示/隱藏

知道 parent/children 關係後，工具可以實現「只顯示某個 BuildingPart 及其所有子物件」的功能——透過遞迴走訪 children 獲得完整物件列表。

### 語意分析

在語意層級，parent/children 關係讓分析工具可以回答更複雜的問題：「找出所有朝南的 WallSurface，以及它們上面所有 Window 的面積總和」。這在建築能源效率分析中非常有用。

---

## 結語

Parent/Children 關係是 CityJSON 從 CityGML 的巢狀 XML 過渡到平面 JSON 結構的關鍵設計。它由兩套獨立但概念相似的機制組成：

1. **城市物件層級的 `parents` / `children`**：用字串 ID 在 `CityObjects` 字典中建立從屬關係，第二層級物件必填 `parents`
2. **語意表面層級的 `parent` / `children`**：用整數索引在單一 Geometry 的 `surfaces` 陣列中建立包含關係，皆為選填

這兩套機制的共同哲學是：**用平面陣列中的指向關係取代實體巢狀結構**。這種設計讓 CityJSON 更容易被程式處理、更適合資料庫儲存、也更容易做到部分載入——同時仍然保留了表達複雜階層結構的能力。

明天起，我們將進入 **Phase 4：進階主題**，探討 Appearance（材質、紋理與顏色）等更深入的 CityJSON 功能！

---

## 參考資料

[^cityjson-spec]: CityJSON Specification 2.0. (2024). City Objects — 1st-level and 2nd-level. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#city-objects

[^cityjson-semantics]: CityJSON Specification 2.0. (2024). Semantics of geometric primitives — Semantic Object parent/children. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#semantics-of-geometric-primitives

[^citygml-nesting]: Open Geospatial Consortium. (2021). OGC City Geography Markup Language (CityGML) Part 1: Conceptual Model Standard. OGC 20-010. Retrieved 2026-09-27, from https://docs.ogc.org/is/20-010/20-010.html