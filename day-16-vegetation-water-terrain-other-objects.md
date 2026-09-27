# Day 16｜植栽、水體、地形與其他第一層級物件

## 前言

昨天我們完成了運輸與基礎設施的 16 種物件。今天，我們要回頭看看 Phase 3 地圖上那些**還沒被深入探討**的第一層級城市物件——除了 Building、Bridge、Tunnel、Road、Railway 和 Transportation 系列之外，CityJSON 還有 **8 種**重要的第一層級物件：植被（SolitaryVegetationObject、PlantCover）、水體（WaterBody）、地形（TINRelief）、土地利用（LandUse）、城市家具（CityFurniture）、通用物件（GenericCityObject）和群組（CityObjectGroup）。

這 8 種物件雖然不像建築物那樣是城市模型的主角，但它們是讓城市模型**完整且真實**的關鍵——沒有樹木的城市是沙漠，沒有水體的地圖只是圖紙，沒有地形的 3D 模型就像漂浮在虛空中。

| 類型 | CityGML 模組 | 幾何限制 | 常見應用 |
|------|-------------|----------|----------|
| `SolitaryVegetationObject` | Vegetation | 任何幾何類型 | 獨立樹木、單株植物 |
| `PlantCover` | Vegetation | Solid, CompositeSolid, MultiSolid, MultiSurface, CompositeSurface | 森林、灌木叢、植被區 |
| `WaterBody` | WaterBody | MultiLineString, MultiSurface, CompositeSurface, Solid, CompositeSolid | 湖泊、河流、運河 |
| `TINRelief` | Relief | **CompositeSurface**（僅三角形） | 地形、地表 |
| `LandUse` | LandUse | MultiSurface, CompositeSurface | 土地使用分區 |
| `CityFurniture` | CityFurniture | 任何幾何類型 | 路燈、長椅、公車站牌 |
| `GenericCityObject` | Generics | MultiPoint, MultiLineString, MultiSurface, CompositeSurface, Solid, CompositeSolid | 不屬於任何類別的物件 |
| `CityObjectGroup` | CityObjectGroup | 任何幾何類型（可選） | 物件群組（街廓、社區） |

---

## 植栽與植被 — 城市的綠意

CityJSON 將植栽分為兩類：**SolitaryVegetationObject**（獨立植被）和 **PlantCover**（植被覆蓋區），兩者都源自 CityGML v3.0 的 Vegetation 模組[^citygml-vegetation]。

### SolitaryVegetationObject — 每一棵樹都是獨立的個體

這個類型用來表達**可以獨立標記的植物實體**——通常是重要的行道樹、古木、大型景觀樹等。它的幾何類型「完全不受限制」，你可以用 MultiPoint 單純標示位置、用 MultiLineString 表達樹幹加樹冠的骨架、用 MultiSurface 或 Solid 做出精細的 3D 樹木模型[^cityjson-spec]。

```json
{
  "type": "SolitaryVegetationObject",
  "attributes": {
    "trunkDiameter": 5.3,
    "crownDiameter": 11.0,
    "species": "Quercus robur"
  },
  "geometry": [{
    "type": "MultiPoint",
    "lod": "1",
    "boundaries": [1]
  }]
}
```

在真實資料集中，大量的 SolitaryVegetationObject 通常搭配**幾何模板 (Geometry Templates)** 使用[^cityjson-spec]。以 LoD3_Railway 資料集為例，超過 200 棵行道樹都共用同一個樹木模板，每一棵只記錄位置和簡單的變換矩陣，將檔案體積從數十 MB 縮小到數 KB[^lod3-railway]。

### PlantCover — 成片的植被

`PlantCover` 用來表達**大面積的植被覆蓋區域**——森林、灌木叢、草地、農田等。與 SolitaryVegetationObject 不同，它不追蹤個別植株，而是將整片區域作為一個地理物件來處理[^citygml-vegetation]。

```json
{
  "type": "PlantCover",
  "attributes": {
    "averageHeight": 11.05,
    "function": "deciduous forest"
  },
  "geometry": [{
    "type": "MultiSolid",
    "lod": "2",
    "boundaries": [
      [
        [ [[0, 3, 2, 1]], [[4, 5, 6, 7]], [[0, 1, 5, 4]], [[10, 13, 22, 31]] ]
      ],
      [
        [ [[5, 34, 31, 12]], [[44, 54, 62, 74]], [[111, 123, 922, 66]] ]
      ]
    ]
  }]
}
```

PlantCover 的幾何限制比 SolitaryVegetationObject 嚴格：只能使用 Solid、CompositeSolid、MultiSolid、MultiSurface 或 CompositeSurface 這五種——因為它必須表達一個**連續的空間範圍**，而不只是一個點或線[^cityjson-spec]。

---

## 水體 — 城市的藍色血脈

`WaterBody` 用來建模湖泊、河流、運河、水庫等所有類型的水體。它源自 CityGML v3.0 的 WaterBody 模組[^citygml-waterbody]。

```json
{
  "mygreatlake": {
    "type": "WaterBody",
    "attributes": {
      "usage": "leisure",
      "name": "Lake Black"
    },
    "geometry": [{
      "type": "Solid",
      "lod": "2",
      "boundaries": [
        [ [[0, 3, 2, 1]], [[4, 5, 6, 7]], [[0, 1, 5, 4]] ]
      ]
    }]
  }
}
```

WaterBody 的允許幾何類型很豐富：可以用 MultiLineString 代表河流中心線、用 MultiSurface 或 CompositeSurface 代表水面、用 Solid 或 CompositeSolid 代表完整的水體體積[^cityjson-spec]。

### 語意表面：WaterSurface 與 WaterGroundSurface

與 Road 的語意表面機制類似，WaterBody 也可以使用語意表面來區分水面的不同部分。在 CityGML 中，水體可以進一步分為 `WaterSurface`（水面）和 `WaterGroundSurface`（水底）——這些在 CityJSON 中是以**幾何的語意標註**來實現的[^cityjson-spec][^citygml-waterbody]。

```json
{
  "type": "WaterBody",
  "geometry": [{
    "type": "MultiSurface",
    "lod": "2",
    "boundaries": [
      [[0, 3, 2, 1]],
      [[4, 5, 6, 7]],
      [[1, 2, 6, 5]],
      [[2, 3, 7, 6]]
    ],
    "semantics": {
      "surfaces": [
        { "type": "WaterSurface" },
        { "type": "WaterGroundSurface" },
        { "type": "WaterGroundSurface" },
        { "type": "WaterGroundSurface" }
      ],
      "values": [0, 1, 1, 1]
    }
  }]
}
```

---

## 地形 — 讓建築有了立足之地

`TINRelief` 是 CityJSON 中唯一專門用來表達**地形**的物件類型，源自 CityGML v3.0 的 Relief 模組[^citygml-relief]。它的名稱 TIN 代表 **Triangulated Irregular Network**（不規則三角網），是 3D GIS 中最常見的地形表達方式。

```json
{
  "myterrain01": {
    "type": "TINRelief",
    "geographicalExtent": [
      84710.1, 446846.0, -5.3,
      84757.1, 446944.0, 40.9
    ],
    "geometry": [{
      "type": "CompositeSurface",
      "lod": "1",
      "boundaries": [
        [[0, 3, 2]], [[4, 5, 6]],
        [[1, 2, 6]], [[2, 3, 7]],
        [[3, 0, 4]]
      ]
    }]
  }
}
```

### TINRelief 的幾何限制

TINRelief 的幾何類型被嚴格限制為 **CompositeSurface** 一種[^cityjson-spec]。而且，雖然技術上任何 CompositeSurface 都可以用，但慣例是**每個 Surface 必須是三角形**（三個頂點，無內環）。這是 TIN 的本質要求——不規則三角網就是由三角形構成的。

值得注意的是，CityJSON 並**沒有**為 TIN 定義獨立的幾何類型——它純粹透過 CompositeSurface + 三角面片約定來表達。這與 CityJSON 的設計哲學一致：盡量重用既有的幾幾何類型，而不是創造新類型。

### 真實案例

在 LoD3_Railway 資料集中，TINRelief 以 **LoD 3** 的精度出現，包含上千個三角形面片，並帶有材質（Material）資訊以輔助視覺化[^lod3-railway]。在 Den Haag 資料集中，建築物和地形被合併在同一個 CityJSON 檔案中，展示建築物如何「坐落在」地形之上[^denhaag-dataset]。

---

## 土地利用 — 城市的功能分區

`LandUse` 用來表達**土地的用途分區**——住宅區、商業區、工業區、農業區、公園綠地等，源自 CityGML v3.0 的 LandUse 模組[^citygml-landuse]。

```json
{
  "oneparcel": {
    "type": "LandUse",
    "attributes": {
      "function": "Industry and Business",
      "area-parcel": {
        "value": 437,
        "uom": "m2"
      }
    },
    "geometry": [{
      "type": "MultiSurface",
      "lod": "1",
      "boundaries": [
        [[0, 3, 2, 1]], [[4, 5, 6, 7]],
        [[0, 1, 5, 4]]
      ]
    }]
  }
}
```

LandUse 的幾何限制為 MultiSurface 或 CompositeSurface 兩種[^cityjson-spec]——因為土地利用通常是以**平面多邊形**來劃分的，不需要立體體積。你可以把每一個 LandUse 物件想像成一張疊在城市表面的彩色分區圖。

值得一提的是，LandUse 的 `attributes` 可以完全自訂。除了上面的巢狀結構（`area-parcel` 帶有 `value` 和 `uom` 單位）外，你也可以直接使用簡單的 `"function": "residential"` 標記[^cityjson-spec]。

---

## 城市家具 — 街道上的細節

`CityFurniture` 是最直觀的物件類型之一：**路燈、長椅、公車站牌、垃圾桶、標誌牌、自行車架、郵筒**……只要是城市街道上的設備都屬於這一類，源自 CityGML v3.0 的 CityFurniture 模組[^citygml-furniture]。

```json
{
  "mystopsign": {
    "type": "CityFurniture",
    "attributes": {
      "function": "bus stop"
    },
    "geometry": [{
      "type": "MultiSurface",
      "lod": "2",
      "boundaries": [
        [[0, 3, 2, 1]], [[4, 5, 6, 7]],
        [[0, 1, 5, 4]]
      ]
    }]
  }
}
```

CityFurniture 的幾何類型**完全不受限制**，因為城市家具有各種各樣的形狀——路燈桿用 MultiLineString、站牌用 MultiSurface、郵筒用 Solid[^cityjson-spec]。

在 LoD3_Railway 資料集中，CityFurniture 物件以 LoD 3 的細節呈現，並帶有材質資訊用於視覺化[^lod3-railway]。

---

## GenericCityObject — 當標準還不夠用

有時你會遇到一種城市物件，它 **不屬於** CityJSON 定義的任何一種類型——它既不是建築物、不是橋樑、不是植被、不是家具。這時就是 `GenericCityObject` 上場的時候了[^cityjson-spec][^citygml-generics]。

```json
{
  "whatisthat": {
    "type": "GenericCityObject",
    "attributes": {
      "class": "a big bucket of water",
      "usage": "it's not clear"
    },
    "geometry": [{
      "type": "CompositeSurface",
      "lod": "1",
      "boundaries": [
        [[0, 3, 2, 1]], [[4, 5, 6, 7]],
        [[0, 1, 5, 4]]
      ]
    }]
  }
}
```

GenericCityObject 的幾何限制比多數物件嚴格：只能使用 MultiPoint、MultiLineString、MultiSurface、CompositeSurface、Solid 或 CompositeSolid[^cityjson-spec]。這是因為通用物件不該使用過於特殊的幾何表達（如 MultiSolid），以維持最低限度的結構一致性。

---

## CityObjectGroup — 群組的力量

`CityObjectGroup` 是 CityJSON 中最特別的物件之一——它存在的目的不是代表一個實體，而是**把一群物件組織在一起**[^cityjson-spec][^citygml-group]。

```json
{
  "my-neighbourhood": {
    "type": "CityObjectGroup",
    "attributes": {
      "location": "Chibougamau Sud"
    },
    "children": ["building1", "building3"],
    "children_roles": ["residential building", "voting location"],
    "geometry": [{
      "type": "MultiSurface",
      "lod": "2",
      "boundaries": [ [[2, 41, 5, 77]] ]
    }]
  }
}
```

CityObjectGroup 擁有幾個獨特之處：

1. **`children` 是必填欄位**：不像其他物件的 `children` 是可選的，CityObjectGroup 必須指定它包含了哪些物件。
2. **`children_roles` 可選**：用來描述每個子物件在群組中的角色，長度必須與 `children` 相同。
3. **可以有自己的幾何**：群組本身可以有一個幾何（例如街廓的邊界多邊形），但不是必須。
4. **群組可以包含群組**：因為 CityObjectGroup 本身也是一個城市物件，它可以成為另一個群組的子物件，形成**巢狀群組**結構[^cityjson-spec]。

在 LoD3_Railway 資料集中，一群 SolitaryVegetationObject（樹木）被組織成一個 CityObjectGroup，每棵樹的 `parents` 陣列都指向這個群組的 ID，形成了**雙向的從屬關係**[^lod3-railway]。

---

## 一個完整場景：城市公園

為了展示這些物件如何協同工作，以下是一個城市公園的片段：

```json
{
  "CityObjects": {
    "central-park": {
      "type": "CityObjectGroup",
      "children": ["creek", "big-oak", "pond"],
      "children_roles": ["water", "tree", "water"],
      "geometry": [{
        "type": "MultiSurface",
        "lod": "1",
        "boundaries": [ [[0, 1, 2, 3, 4]] ]
      }]
    },
    "creek": {
      "type": "WaterBody",
      "attributes": { "name": "Park Creek" },
      "geometry": [{
        "type": "MultiLineString",
        "lod": "1",
        "boundaries": [ [ [10, 11, 12, 13] ] ]
      }]
    },
    "big-oak": {
      "type": "SolitaryVegetationObject",
      "attributes": {
        "species": "Quercus rubra",
        "trunkDiameter": 8.2
      },
      "geometry": [{
        "type": "MultiPoint",
        "lod": "1",
        "boundaries": [5]
      }]
    },
    "pond": {
      "type": "WaterBody",
      "attributes": { "usage": "ornamental" },
      "geometry": [{
        "type": "MultiSurface",
        "lod": "2",
        "boundaries": [ [[8, 9, 10, 7, 6]] ]
      }]
    },
    "park-table": {
      "type": "CityFurniture",
      "attributes": { "function": "picnic table" },
      "geometry": [{
        "type": "MultiPoint",
        "lod": "1",
        "boundaries": [15]
      }]
    }
  }
}
```

這個範例雖然簡化，但展示了不同物件類型如何在同一個場景中協同工作，以及 CityObjectGroup 如何將這些物件組織成一個邏輯單元。

---

## 結語

今天我們認識了 CityJSON 中 8 種重要的第一層級城市物件——從自然環境（植栽、水體、地形）到人為設施（土地利用、城市家具），再到靈活的擴充機制（通用物件、群組）。至此，CityJSON 的 33 種城市物件已經全部涵蓋完畢。

加上前兩天的 Building、Bridge、Tunnel、Transportation 家族，你現在對 CityJSON 的城市物件體系已經有了完整的認識：

| 分類 | 物件數量 | 包含 |
|------|---------|------|
| 建築物家族 | 8 種 | Building, BuildingPart, BuildingInstallation, BuildingConstructiveElement, BuildingFurniture, BuildingStorey, BuildingRoom, BuildingUnit |
| 橋樑家族 | 6 種 | Bridge, BridgePart, BridgeInstallation, BridgeConstructiveElement, BridgeRoom, BridgeFurniture |
| 隧道家族 | 6 種 | Tunnel, TunnelPart, TunnelInstallation, TunnelConstructiveElement, TunnelHollowSpace, TunnelFurniture |
| Transportation | 4 種 | Road, Railway, Waterway, TransportSquare |
| 自然環境 | 4 種 | SolitaryVegetationObject, PlantCover, WaterBody, TINRelief |
| 其他 | 5 種 | LandUse, CityFurniture, GenericCityObject, CityObjectGroup, OtherConstruction, TINRelief |

明天，我們將進入 Phase 3 的下一個主題：**語意表面 (Semantic Surfaces)**——讓幾何不只是形狀，還能告訴我們「這是牆」、「這是屋頂」。

---

## 參考資料

[^cityjson-spec]: CityJSON Specification 2.0. (2024). City Objects — Various 1st-level City Objects. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#city-objects

[^citygml-vegetation]: Open Geospatial Consortium. (2021). OGC City Geography Markup Language (CityGML) Part 1: Conceptual Model Standard — Vegetation module (Clause 7.14). OGC Document 20-010, Version 3.0.0. Retrieved 2026-09-27, from https://docs.ogc.org/is/20-010/20-010.html#toc40

[^citygml-waterbody]: Open Geospatial Consortium. (2021). OGC City Geography Markup Language (CityGML) Part 1: Conceptual Model Standard — WaterBody module (Clause 7.15). OGC Document 20-010, Version 3.0.0. Retrieved 2026-09-27, from https://docs.ogc.org/is/20-010/20-010.html#toc42

[^citygml-relief]: Open Geospatial Consortium. (2021). OGC City Geography Markup Language (CityGML) Part 1: Conceptual Model Standard — Relief module (Clause 7.12). OGC Document 20-010, Version 3.0.0. Retrieved 2026-09-27, from https://docs.ogc.org/is/20-010/20-010.html#toc38

[^citygml-landuse]: Open Geospatial Consortium. (2021). OGC City Geography Markup Language (CityGML) Part 1: Conceptual Model Standard — LandUse module (Clause 7.10). OGC Document 20-010, Version 3.0.0. Retrieved 2026-09-27, from https://docs.ogc.org/is/20-010/20-010.html#toc36

[^citygml-furniture]: Open Geospatial Consortium. (2021). OGC City Geography Markup Language (CityGML) Part 1: Conceptual Model Standard — CityFurniture module (Clause 7.6). OGC Document 20-010, Version 3.0.0. Retrieved 2026-09-27, from https://docs.ogc.org/is/20-010/20-010.html#toc32

[^citygml-group]: Open Geospatial Consortium. (2021). OGC City Geography Markup Language (CityGML) Part 1: Conceptual Model Standard — CityObjectGroup module (Clause 7.7). OGC Document 20-010, Version 3.0.0. Retrieved 2026-09-27, from https://docs.ogc.org/is/20-010/20-010.html#toc33

[^citygml-generics]: Open Geospatial Consortium. (2021). OGC City Geography Markup Language (CityGML) Part 1: Conceptual Model Standard — Generics module (Clause 7.9). OGC Document 20-010, Version 3.0.0. Retrieved 2026-09-27, from https://docs.ogc.org/is/20-010/20-010.html#toc35

[^lod3-railway]: TU Delft 3D Geoinformation. (2024). LoD3_Railway.city.json — CityJSON dataset with railway, vegetation, terrain, water, tunnels. Retrieved 2026-09-27, from https://3d.bk.tudelft.nl/opendata/cityjson/3dcities/v2.0/LoD3_Railway.city.json

[^denhaag-dataset]: TU Delft 3D Geoinformation. (2024). DenHaag_01.city.json — Buildings (LoD2) and Terrain merged. Retrieved 2026-09-27, from https://3d.bk.tudelft.nl/opendata/cityjson/3dcities/v2.0/DenHaag_01.city.json

[^cityjson-spec-attributes]: CityJSON Specification 2.0. (2024). City Objects — Attributes for all City Objects. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#city-objects