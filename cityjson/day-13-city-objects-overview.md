# Day 13｜城市物件總覽 — 第一層級與第二層級

## 前言

經過前幾天對幾何系統的深入探討（頂點變換、8 種幾何類型、邊界陣列、幾何模板、LoD），今天我們要進入 Phase 3 的第一篇：**城市物件 (City Objects)**。

如果把 CityJSON 檔案想像成一座城市的數位模型，那麼 `vertices` 是這座城市的「骨架」，`geometry` 是「皮膚」，而 **City Objects** 就是這座城市裡「有名字的事物」──建築物、道路、橋樑、樹木、地形……每一樣都是 City Object。

CityJSON 定義了多達 **33 種**城市物件類型，但它們並非全部平等。有些可以獨立存在，有些則必須依附在其他物件之上。這就是今天要探討的核心：**第一層級 (1st-level)** 與 **第二層級 (2nd-level)** 城市物件[^cityjson-spec]。

---

## 為什麼要有層級之分？

這要追溯到 CityJSON 的前身 **CityGML**。在 CityGML (XML-based) 中，物件之間的從屬關係是透過 XML 的巢狀結構來表達的：

```xml
<Building>
  <BuildingPart>
    <BuildingInstallation/>
  </BuildingPart>
</Building>
```

但在 CityJSON 中，所有物件都被「展平」(flattened) 到一個 `CityObjects` 字典中，不再有巢狀結構。因此，需要一套機制來還原這些從屬關係，這就是 `parents` 與 `children` 的由來[^cityjson-spec]。

---

## 第一層級城市物件 (1st-level)

第一層級城市物件代表**可以獨立存在**的實體，它們不需要父物件。這包含 16 種類型：

| 類型 | 中文描述 | 限制的幾何類型 |
|------|----------|----------------|
| `Building` | 建築物 | Solid, CompositeSolid, MultiSurface, CompositeSurface |
| `Bridge` | 橋樑 | Solid, CompositeSolid, MultiSurface, CompositeSurface |
| `Tunnel` | 隧道 | Solid, CompositeSolid, MultiSurface, CompositeSurface |
| `Road` | 道路 | 任何幾何類型 |
| `Railway` | 鐵路 | 任何幾何類型 |
| `Waterway` | 水路 | 任何幾何類型 |
| `TransportSquare` | 交通廣場（停車場、廣場） | 任何幾何類型 |
| `CityFurniture` | 城市家具（路燈、長椅、站牌） | 任何幾何類型 |
| `CityObjectGroup` | 城市物件群組 | 任何幾何類型 |
| `GenericCityObject` | 通用城市物件 | MultiPoint, MultiLineString, MultiSurface, CompositeSurface, Solid, CompositeSolid |
| `LandUse` | 土地利用 | MultiSurface, CompositeSurface |
| `OtherConstruction` | 其他構築物（電塔、圍欄、水箱） | 任何幾何類型 |
| `PlantCover` | 植被覆蓋區 | Solid, CompositeSolid, MultiSolid, MultiSurface, CompositeSurface |
| `SolitaryVegetationObject` | 單獨植被物件（獨立樹木） | 任何幾何類型 |
| `TINRelief` | 地形（不規則三角網） | CompositeSurface（僅三角形） |
| `WaterBody` | 水體（湖泊、河流） | MultiLineString, MultiSurface, CompositeSurface, Solid, CompositeSolid |

這些物件的最小合法表示非常簡單，只需要一個 `type` 欄位：

```json
{
  "type": "Building"
}
```

任何第一層級物件都可以這樣表達[^cityjson-spec]。

---

## 第二層級城市物件 (2nd-level)

第二層級城市物件**必須有父物件才能存在**，它們總共包含 17 種類型，分屬於三大模組：建築物、橋樑、隧道。

### 建築物家族（8 種第二層級）

```
Building (1st)
├── BuildingPart
│   ├── BuildingInstallation
│   ├── BuildingConstructiveElement
│   ├── BuildingFurniture
│   ├── BuildingStorey
│   └── BuildingRoom
│       ├── BuildingInstallation
│       ├── BuildingConstructiveElement
│       └── BuildingFurniture
├── BuildingInstallation
├── BuildingConstructiveElement
├── BuildingFurniture
├── BuildingStorey
├── BuildingRoom
│   ├── BuildingInstallation
│   ├── BuildingConstructiveElement
│   └── BuildingFurniture
└── BuildingUnit
```

| 類型 | 可作為父物件 | 限制的幾何類型 |
|------|-------------|----------------|
| `BuildingPart` | Building | Solid, CompositeSolid, MultiSurface, CompositeSurface |
| `BuildingInstallation` | Building, BuildingPart, BuildingRoom | 任何幾何類型 |
| `BuildingConstructiveElement` | Building, BuildingPart, BuildingRoom | 任何幾何類型 |
| `BuildingFurniture` | Building, BuildingPart, BuildingRoom | 任何幾何類型 |
| `BuildingStorey` | Building, BuildingPart, BuildingRoom | Solid, CompositeSolid, MultiSurface, CompositeSurface |
| `BuildingRoom` | Building, BuildingPart | Solid, CompositeSolid, MultiSurface, CompositeSurface |
| `BuildingUnit` | Building, BuildingPart | Solid, CompositeSolid, MultiSurface, CompositeSurface |

### 橋樑家族（5 種第二層級）

```
Bridge (1st)
├── BridgePart
│   ├── BridgeInstallation
│   ├── BridgeConstructiveElement
│   ├── BridgeRoom
│   │   └── BridgeFurniture
│   └── BridgeFurniture
├── BridgeInstallation
├── BridgeConstructiveElement
├── BridgeRoom
│   └── BridgeFurniture
└── BridgeFurniture
```

| 類型 | 可作為父物件 | 限制的幾何類型 |
|------|-------------|----------------|
| `BridgePart` | Bridge | Solid, CompositeSolid, MultiSurface, CompositeSurface |
| `BridgeInstallation` | Bridge, BridgePart, BridgeRoom | 任何幾何類型 |
| `BridgeConstructiveElement` | Bridge, BridgePart, BridgeRoom | 任何幾何類型 |
| `BridgeRoom` | Bridge, BridgePart | 任何幾何類型 |
| `BridgeFurniture` | Bridge, BridgePart, BridgeRoom | 任何幾何類型 |

### 隧道家族（5 種第二層級）

```
Tunnel (1st)
├── TunnelPart
│   ├── TunnelInstallation
│   ├── TunnelConstructiveElement
│   ├── TunnelHollowSpace
│   └── TunnelFurniture
├── TunnelInstallation
├── TunnelConstructiveElement
├── TunnelHollowSpace
└── TunnelFurniture
```

| 類型 | 可作為父物件 | 限制的幾何類型 |
|------|-------------|----------------|
| `TunnelPart` | Tunnel | Solid, CompositeSolid, MultiSurface, CompositeSurface |
| `TunnelInstallation` | Tunnel, TunnelPart | 任何幾何類型 |
| `TunnelConstructiveElement` | Tunnel, TunnelPart | 任何幾何類型 |
| `TunnelHollowSpace` | Tunnel, TunnelPart | 任何幾何類型 |
| `TunnelFurniture` | Tunnel, TunnelPart | 任何幾何類型 |

第二層級物件的最小合法表示必須包含 `parents`：

```json
{
  "type": "BuildingPart",
  "parents": ["id-parent"]
}
```

任何第二層級物件都可以這樣表達[^cityjson-spec]。

---

## Children 與 Parents 機制

CityJSON 使用雙向鏈結來表達物件之間的從屬關係：

### 從父到子：`children`

第一層級物件的 `children` 陣列列出其**直接子物件**的 ID（不包含孫級）：

```json
"id-1": {
  "type": "Building",
  "children": ["id-2"],
  "geometry": [{...}]
}
```

- 只包含直接子級，不包含子級的子級（孫級）
- 一個物件可以有多種子類型（例如一個 Building 同時有 BuildingPart 和 BuildingInstallation）
- 子物件的順序不重要

### 從子到父：`parents`

第二層級物件**必須**有 `parents` 陣列，指向其父物件：

```json
"id-2": {
  "type": "BuildingPart",
  "parents": ["id-1"],
  "children": ["id-3"],
  "geometry": [{...}]
}
```

- 在 CityJSON 核心模組中，第二層級物件的 `parents` 陣列大小永遠為 1（只有一個父物件）
- 擴充 (Extension) 可以定義擁有多個父物件的物件

### 完整範例

```json
"CityObjects": {
  "id-1": {
    "type": "Building",
    "geographicalExtent": [84710.1, 446846.0, -5.3, 84757.1, 446944.0, 40.9],
    "children": ["id-2"],
    "geometry": [{...}]
  },
  "id-2": {
    "type": "BuildingPart",
    "parents": ["id-1"],
    "children": ["id-3"],
    "geometry": [{...}]
  },
  "id-3": {
    "type": "BuildingInstallation",
    "parents": ["id-2"],
    "geometry": [{...}]
  },
  "id-4": {
    "type": "LandUse",
    "geometry": [{...}]
  }
}
```

可以看到：
- `id-1` (Building) 是獨立的，它引用 `id-2` 作為子物件
- `id-2` (BuildingPart) 有父物件 `id-1`，同時又引用 `id-3` 作為子物件
- `id-3` (BuildingInstallation) 是最底層的子物件，只有父物件 `id-2`
- `id-4` (LandUse) 是完全獨立的第一層級物件，沒有父子關係

這形成了一個**鏈結串列**式（而非巢狀樹狀）的結構──展平後透過 ID 關聯還原階層[^cityjson-spec]。

---

## 交通運輸的特殊案例

`Road`、`Railway`、`Waterway`、`TransportSquare` 這四種交通物件有一個特殊之處：它們的子物件**類型與父物件相同**。例如一條道路可以被切分為多個路段：

```json
"ma_rue": {
  "type": "Road",
  "children": ["sect1", "sect2"],
  "geometry": [...]
},
"sect1": {
  "type": "Road",
  "parents": ["ma_rue"],
  "geometry": [...]
}
```

由於子物件和父物件都是 `Road`（同一類型），它們不被視為獨立的第二層級類型，而是同一類型之間的父子關係[^cityjson-spec]。

---

## 今日總結

| 概念 | 重點 |
|------|------|
| **第一層級 (1st-level)** | 可獨立存在，無需父物件，共 16 種 |
| **第二層級 (2nd-level)** | 必須有父物件，使用 `parents` 陣列，共 17 種 |
| **`children`** | 父物件指向直接子物件的 ID 陣列 |
| **`parents`** | 子物件指向父物件的 ID 陣列（核心模組固定為 1 個） |
| **交通運輸** | 子物件與父物件為同一類型（如 Road → Road） |
| **最小表達** | 第一層級只需 `type`；第二層級需要 `type` + `parents` |

明天我們將深入探討最常見也最豐富的**建築物家族**，看看 Building、BuildingPart、BuildingRoom 等 8 種建築相關物件如何組合出完整的數位建築模型。

---

## 參考資料

[^cityjson-spec]: CityJSON Specification 2.0.2. (2024). City Objects — 1st-level and 2nd-level. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#city-objects

[^citygml-building]: Open Geospatial Consortium. (2022). OGC CityGML v3.0.0 — Building module. Retrieved 2026-09-27, from https://docs.ogc.org/is/20-010/20-010.html#toc45

[^citygml-bridge]: Open Geospatial Consortium. (2022). OGC CityGML v3.0.0 — Bridge module. Retrieved 2026-09-27, from https://docs.ogc.org/is/20-010/20-010.html#toc44

[^citygml-tunnel]: Open Geospatial Consortium. (2022). OGC CityGML v3.0.0 — Tunnel module. Retrieved 2026-09-27, from https://docs.ogc.org/is/20-010/20-010.html#toc46