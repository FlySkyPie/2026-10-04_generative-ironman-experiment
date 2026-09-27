# Day 17｜語意表面 — 不只是幾何，還知道這是牆還是屋頂

## 前言

過去幾天我們認識了 CityJSON 的所有 33 種城市物件與它們的幾何類型。但只靠幾何，3D 城市模型就像一張沒有圖例的地圖——你知道那裡有一棟建築，但不知道哪些面是屋頂、哪些面是牆、窗戶在哪裡、門在哪裡。

**語意表面 (Semantic Surfaces)** 就是解決這個問題的機制。它為幾何的每一個基本單元（面、點、線）貼上「標籤」，告訴電腦：「這個三角形是屋頂」、「這個四邊形是窗戶」、「這些點代表一棵樹的位置」。

為什麼這很重要？試想以下場景：

- **太陽能潛力分析**：需要找出所有朝南的 RoofSurface，計算面積與坡度
- **能源效率評估**：需要計算 Window 面積與 WallSurface 面積的比例（窗牆比）
- **都市洪水模擬**：需要區分 GroundSurface 與高於地面的表面
- **室內導航**：需要知道 Door 的位置、FloorSurface 在哪裡

沒有語意表面，這些分析全部得靠人工猜測或複雜的幾何運算來推斷。有了語意表面，一切變得簡單且明確。

---

## Semantic Object — 語意的原子單位

在 CityJSON 中，每一個語意標籤都是一個 **Semantic Object**（語意物件）[^cityjson-semantics]。它是一個 JSON 物件，代表一個幾何基本單元的意義——對於表面和體積幾何（如 MultiSurface、Solid），基本單元是「面」；對於 MultiPoint 或 MultiLineString，基本單元則是「點」或「線段」[^cityjson-spec]。

```json
{
  "type": "RoofSurface",
  "slope": 16.4,
  "children": [2, 37],
  "solar-potential": 5
}
```

一個 Semantic Object 有幾個重要成員：

| 成員 | 必填 | 型態 | 說明 |
|------|------|------|------|
| `type` | ✅ | 字串 | 語意類型名稱（如 `"RoofSurface"`） |
| `parent` | ❌ | 整數 | 指向同幾何中另一個 Semantic Object 的索引（0-based） |
| `children` | ❌ | 整數陣列 | 指向同幾何中子 Semantic Object 的索引陣列 |
| 其他 | ❌ | 任意 | 自訂屬性（如坡度、太陽能潛力） |

語意類型的值必須是 CityJSON 定義的標準類型，或以 `+` 開頭的自訂類型（遵循 Extension 規則）[^cityjson-extensions]。

---

## Building 的 11 種語意類型

Building 家族是語意表面最豐富的應用場景。`Building`、`BuildingPart`、`BuildingRoom`、`BuildingStorey`、`BuildingUnit` 和 `BuildingInstallation` 共用的 11 種語意類型如下[^cityjson-semantics]：

| 語意類型 | 說明 | 應用場景 |
|----------|------|----------|
| `RoofSurface` | 屋頂表面 | 太陽能潛力、屋頂面積計算 |
| `GroundSurface` | 地面接觸面 | 洪水模擬、接地分析 |
| `WallSurface` | 外牆表面 | 能源效率、立面分析 |
| `ClosureSurface` | 封閉表面 | 地下空間封閉、體積計算 |
| `OuterCeilingSurface` | 外部天花板 | 懸挑結構、陽台底部分析 |
| `OuterFloorSurface` | 外部地板 | 露台、外露樓板分析 |
| `Window` | 窗戶 | 窗牆比、採光分析 |
| `Door` | 門 | 室內導航、逃生路線 |
| `InteriorWallSurface` | 內牆表面 | 室內格局、空間分割 |
| `CeilingSurface` | 室內天花板 | 室內設計、樓高分析 |
| `FloorSurface` | 室內地板 | 室內導航、面積計算 |

除了 Building 家族，CityJSON 也為其他物件類型定義了語意類型：

- **WaterBody**：`WaterSurface`、`WaterGroundSurface`、`WaterClosureSurface`
- **Transportation**（Road、Railway、TransportSquare）：`TrafficArea`、`AuxiliaryTrafficArea`、`TransportationMarking`、`TransportationHole`

---

## `surfaces` 與 `values` — 語意映射的核心機制

理論很簡單：把語意標籤貼到幾何上。但實作上有一個挑戰：**一個大型建築可能被三角化成幾百個三角形，其中數十個三角形都屬於同一面牆**——如果每一個三角形都要重複宣告「這是 WallSurface」，不僅浪費空間，也難以維護。

CityJSON 的解法來自一個經典的設計模式：**宣告一次，重複引用**。整個機制由兩個陣列組成[^cityjson-semantics]：

### `surfaces` — 語意定義陣列

`surfaces` 是一個 Semantic Object 的陣列，類似「圖例」或「查詢表」。每一個元素定義一種語意類型及其屬性。

### `values` — 索引映射陣列

`values` 是一個整數（或 `null`）的巢狀陣列，與幾何的 `boundaries` 結構對應。每一個值指向 `surfaces` 陣列中的一個索引（0-based），告訴電腦「這個基本單元對應哪種語意」。`null` 表示該基本單元沒有語意。

### MultiSurface 範例

以一個最典型的建築物外殼為例——5 個面分別是兩面牆、一個帶有洞口的牆面（無語意）、一個屋頂和一面落地窗：

```json
{
  "type": "MultiSurface",
  "lod": "2",
  "boundaries": [
    [[0, 3, 2, 1]],
    [[4, 5, 6, 7]],
    [[0, 1, 5, 4]],
    [[0, 2, 3, 8]],
    [[10, 12, 23, 48]]
  ],
  "semantics": {
    "surfaces": [
      {
        "type": "WallSurface",
        "slope": 33.4,
        "children": [2]
      },
      {
        "type": "RoofSurface",
        "slope": 66.6
      },
      {
        "type": "+PatioDoor",
        "parent": 0,
        "colour": "blue"
      }
    ],
    "values": [0, 0, null, 1, 2]
  }
}
```

解讀方式：

| `values` 索引 | 值 | 對應的 `surfaces` | 意義 |
|-------------|-----|-------------------|------|
| `values[0] = 0` | 牆面 | `surfaces[0]` = `WallSurface` | 第一個面是外牆 |
| `values[1] = 0` | 牆面 | `surfaces[0]` = `WallSurface` | 第二個面也是外牆 |
| `values[2] = null` | 無 | — | 第三個面沒有語意（可能是有洞口的牆面） |
| `values[3] = 1` | 屋頂 | `surfaces[1]` = `RoofSurface` | 第四個面是屋頂 |
| `values[4] = 2` | 落地門 | `surfaces[2]` = `+PatioDoor` | 第五個面是落地門（自訂類型） |

這個範例同時展示了自訂語意類型（`+PatioDoor`）——只要以 `+` 前綴，你就可以定義標準未涵蓋的語意[^cityjson-spec]。

### CompositeSolid 範例

對於體積幾何（Solid、CompositeSolid、MultiSolid），`values` 的巢狀深度需要增加一層來對應 shell 結構：

```json
{
  "type": "CompositeSolid",
  "lod": "2.2",
  "boundaries": [
    [
      [
        [[0, 3, 2, 1, 22]], [[4, 5, 6, 7]],
        [[0, 1, 5, 4]], [[1, 2, 6, 5]]
      ]
    ],
    [
      [
        [[666, 667, 668]], [[74, 75, 76]], [[880, 881, 885]]
      ]
    ]
  ],
  "semantics": {
    "surfaces": [
      { "type": "RoofSurface" },
      { "type": "WallSurface" }
    ],
    "values": [
      [
        [0, 1, 1, null]
      ],
      [
        [null, null, null]
      ]
    ]
  }
}
```

這裡 `values` 的結構對應 `boundaries` 的巢狀層級：每一個 Solid 的 shell 內，每個 face 對應一個語意索引。第二個 Solid 全部為 `null`，代表該實體的所有面都沒有語意資訊。

---

## Parent/Children 關係 — 窗戶屬於哪面牆？

在 CityJSON 中，窗戶和門「屬於」哪一面牆或屋頂，不是用巢狀結構來表達，而是透過 Semantic Object 之間的 **parent/children 指標** 來連結[^cityjson-semantics]。

### 語意中的父子機制

一個 Semantic Object 可以宣告：

- **`parent`**（整數）：指向同 `surfaces` 陣列中的另一個索引。**只能有一個 parent**——扇窗戶只會屬於一面牆。
- **`children`**（整數陣列）：指向同 `surfaces` 陣列中的子物件索引。一個牆面可以有多個子物件（多扇窗戶）。

這是一種**雙向鏈結**：父說「這些是我的孩子」，子說「這是我的父親」。

```json
{
  "surfaces": [
    {
      "type": "WallSurface",    // index 0
      "slope": 33.4,
      "children": [2]           // 這面牆有一個孩子（index 2 的 Window）
    },
    {
      "type": "RoofSurface",    // index 1
      "slope": 66.6
    },
    {
      "type": "Window",          // index 2
      "parent": 0,              // 這扇窗屬於 index 0 的 WallSurface
      "type-glass": "HR++"      // 自訂屬性：玻璃類型
    }
  ],
  "values": [0, 0, null, 1, 2]
}
```

在這個範例中：

- `WallSurface`（index 0）宣告 `children: [2]`，表示它包含 index 2 的開口
- `Window`（index 2）宣告 `parent: 0`，確認它屬於 index 0 的 WallSurface
- `Window` 同時帶有自訂屬性 `type-glass`，記錄了玻璃的類型（HR++ 節能玻璃）

### 為什麼不用巢狀結構？

如果你熟悉 XML 的巢狀思維，可能會想問：「為什麼不直接把 Window 放在 WallSurface 裡面？」這是因為 **CityJSON 的語意陣列是平面的**——所有 Semantic Object 都在同一層 `surfaces` 陣列中，用索引互相參照。這種設計來自 JSON 的資料特性：巢狀會增加走訪的複雜度，而平面陣列 + 索引的方式更容易被程式處理和查詢[^cityjson-spec]。

### 多個窗戶的範例

一面牆開了三扇窗：

```json
{
  "surfaces": [
    {
      "type": "WallSurface",
      "children": [1, 2, 3]
    },
    { "type": "Window", "parent": 0, "width": 1.2, "height": 1.5 },
    { "type": "Window", "parent": 0, "width": 0.8, "height": 1.5 },
    { "type": "Window", "parent": 0, "width": 1.2, "height": 1.5 }
  ]
}
```

三扇窗的 `parent` 都指向 index 0（同一面牆），該牆的 `children` 陣列則依序列出三扇窗的索引。視覺化工具可以利用這個結構來正確著色或選擇性隱藏開口。

---

## 語意表面的應用場景

語意表面讓 3D 城市模型從「漂亮的幾何模型」進化成「可分析的資料模型」。以下是幾個關鍵應用：

### 太陽能潛力分析

這是語意表面最經典的應用之一。透過找出所有 `RoofSurface` 類型的表面，分析其坡度（`slope`）和方位角，結合氣象資料計算太陽輻射量，就可以自動找出最適合安裝太陽能板的屋頂[^solar-potential]。

```json
{
  "type": "RoofSurface",
  "slope": 35.0,
  "solar-potential": 8.5
}
```

### 建築能源效評估

藉由計算 `Window` 面積與 `WallSurface` 面積的比例（窗牆比），可以評估建築物的能源效率。`+HRplusplus`（HR++ 玻璃）等自訂屬性可以進一步提供隔熱性能資訊。

### 洪水模擬

`GroundSurface` 標記了建築物與地面接觸的表面，`WaterSurface` 與 `WaterGroundSurface` 則標記了水體的語意——這些資訊對都市洪水模擬至關重要，可以幫助計算水流路徑與淹水範圍。

### 室內導航與避難

結合 `Door`、`FloorSurface`、`CeilingSurface` 和 `InteriorWallSurface`，可以建立建築物內部的可通行路徑網路，用於緊急避難路線規劃或室內導航應用。

---

## 語意值的深度規則對照

由於 `values` 陣列的巢狀深度與 `boundaries` 相關，以下是對照表[^cityjson-spec]：

| 幾何類型 | `boundaries` 深度 | `values` 深度 | 範例值 |
|----------|------------------|--------------|--------|
| MultiPoint | 2 | 1 | `[0, 1, null]` |
| MultiLineString | 3 | 1 | `[0, 1, null]` |
| MultiSurface | 3 | 1 | `[0, 0, null, 1]` |
| CompositeSurface | 3 | 1 | `[0, 0, null]` |
| Solid | 4 | 2 | `[[0, 1, null]]` |
| MultiSolid | 4 | 2 | `[[0, 1, null]]` |
| CompositeSolid | 4 | 2 | `[[0, 1, null]]` |

規則很簡單：**從 `boundaries` 的深度減 2**，就是 `values` 的深度。唯一例外是 `MultiPoint` 和 `MultiLineString`，它們的 `values` 深度固定為 1（因為基本單元是點或線段，而不是面）[^cityjson-semantics]。

---

## 語意 vs. Attributes：怎麼分工？

你可能會疑惑：「語意表面的屬性和 City Object 的 `attributes` 有什麼不同？」

這是一個很好的問題，兩者的分工很清晰：

| 面向 | City Object `attributes` | Semantic Object 屬性 |
|------|------------------------|---------------------|
| 範圍 | 整個物件 | 單一幾何基本單元 |
| 範例 | 建築物高度、地址、建造年份 | 該面屋頂的坡度、該扇窗的玻璃類型 |
| 粒度 | 粗粒度 | 細粒度 |

簡單來說：**attributes 是整棟建築的資料，語意屬性是單一面板的資料**。

---

## 結語

語意表面是 CityJSON 從「3D 幾何格式」進化到「3D 城市資訊模型」的關鍵設計。透過 `surfaces` + `values` 的映射機制，它以極簡潔的方式為每一個幾何基本單元賦予意義——無論是 Building 的 11 種語意類型、WaterBody 的 3 種水體語意，還是以 `+` 前綴擴充的自訂語意。

更重要的是，**語意表面不僅是標籤，還可以攜帶屬性**——屋頂的坡度、窗戶的玻璃類型、牆面的太陽能潛力值——這些屬性讓 3D 城市模型從視覺化工具升級為真正的分析平台。

明天，我們將深入探討另一個關鍵機制：**Parent/Children 關係**——從平面 JSON 結構理解樹狀的城市物件階層，以及語意表面中 parent/children 與城市物件層級中的 parent/children 有什麼不同。

---

## 參考資料

[^cityjson-spec]: CityJSON Specification 2.0. (2024). Semantics of geometric primitives. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#semantics-of-geometric-primitives

[^cityjson-semantics]: CityJSON Specification 2.0. (2024). Semantics of geometric primitives — Semantic Object structure. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#semantics-of-geometric-primitives

[^cityjson-extensions]: CityJSON Specification 2.0. (2024). Extensions — Custom semantic types. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#extensions

[^solar-potential]: Biljecki, F., Stoter, J., & Ledoux, H. (2015). An improved LOD specification for 3D building models using CityJSON. *Computers, Environment and Urban Systems*, 59, 25-37. Retrieved 2026-09-27, from https://doi.org/10.1016/j.compenvurbsys.2016.04.005