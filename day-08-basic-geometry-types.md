# Day 8：基礎幾何類型 — MultiPoint、MultiLineString、MultiSurface、CompositeSurface

> Day 7 我們學會了頂點與 Transform 的整數壓縮魔法，今天終於要來看這些頂點如何組成真正的幾何。CityJSON 定義了 8 種幾何類型，今天先從最基礎的四種開始。

---

## 先看總覽：8 種幾何類型

CityJSON 的幾何物件都承襲自 CityGML 的線性平面幾何（linear and planar primitives）設計哲學——不支援曲線或參數曲面，所有幾何都由**直線段和平面**構成[^cityjson-specs-geometry]。

這 8 種幾何類型可分成兩大類：

| 類別 | 幾何類型 | 說明 |
|------|---------|------|
| **基礎點線面** | `MultiPoint` | 多個獨立點 |
| | `MultiLineString` | 多條折線 |
| | `MultiSurface` | 多個獨立平面（可含洞） |
| | `CompositeSurface` | 多個相連平面（共享邊界） |
| **實體幾何** | `Solid` | 單一立體（外殼 + 內殼） |
| | `CompositeSolid` | 多個相連的 Solid |
| | `MultiSolid` | 多個不相連的 Solid |
| **模板** | `GeometryInstance` | 指向幾何模板的實例 |

今天先攻前四種，實體幾種留給 Day 9。

---

## 共同的結構骨架

所有幾何物件都長這樣[^cityjson-specs-geometry]：

```json
{
  "type": "<幾何類型>",
  "lod": "<LoD 字串>",
  "boundaries": [/* 巢狀陣列 */]
}
```

三個必備成員：
- **`type`** — 上面 8 種之一
- **`lod`** — 層級細節（Level-of-Detail），可以是單一字元 `"1"` 或 TU Delft 改良格式如 `"2.1"`（Day 12 詳細說明）
- **`boundaries`** — 頂點索引的巢狀陣列，深度取決於幾何類型

可選成員：`semantics`、`material`、`texture`，後面幾天會陸續介紹。

---

## 深度規則（Depth Rules）— 看懂 boundary 陣列的關鍵

幾何類型決定了 `boundaries` 陣列的**巢狀深度**[^cityjson-specs-boundaries]：

| 幾何類型 | boundaries 深度 | 陣列結構 |
|----------|---------------|----------|
| `MultiPoint` | 1 | `[v1, v2, ...]` |
| `MultiLineString` | 2 | `[[v1, v2], [v3, v4, v5], ...]` |
| `MultiSurface` | 3 | `[[[外環], [內環], ...], ...]` |
| `CompositeSurface` | 3 | 同上 |
| `Solid` | 4 | `[[[[外環], ...], ...], ...]` |
| `MultiSolid` / `CompositeSolid` | 5 | `[[[[[外環], ...], ...], ...], ...]` |

這個深度規則是 CityJSON 最容易被忽略卻最重要的設計。看懂深度，你就看懂了一半的 CityJSON 幾何。

---

## MultiPoint — 最簡單的幾何

MultiPoint 就是一群 3D 點，`boundaries` 只是一個一維陣列，裡面放頂點索引[^cityjson-specs-geometry]：

```json
{
  "type": "MultiPoint",
  "lod": "1",
  "boundaries": [2, 44, 0, 7]
}
```

這個幾何等於四個點：vertices[2]、vertices[44]、vertices[0]、vertices[7]。

**應用場景**：
- **建築地址點**（address location）— 標記建築物大門位置
- **單棵樹木**（SolitaryVegetationObject）— 樹木的中心點
- **家具**（BuildingFurniture、CityFurniture）— 物件的定位點

因為只儲存點，沒有面積或體積，MultiPoint 常用於**定位和索引**，而非完整建模。

---

## MultiLineString — 線條的集合

MultiLineString 是一組折線（polyline），每條折線由一串頂點構成。`boundaries` 深度為 2[^cityjson-specs-geometry]：

```json
{
  "type": "MultiLineString",
  "lod": "1",
  "boundaries": [
    [2, 3, 5],
    [77, 55, 212]
  ]
}
```

這裡有兩條折線：第一條經過 vertices[2]→[3]→[5]，第二條經過 [77]→[55]→[212]。

**應用場景**：
- **道路中心線** — 公路、街道的軸線
- **鐵路軌道**（Railway）— 軌道線路
- **水道**（WaterBody）— 河流、運河的中心線
- **管線網路** — 水管、瓦斯管、電纜路徑

MultiLineString 在都市基礎設施建模中很常見，因為許多設施本質上就是線性特徵。

---

## MultiSurface — 最常用的幾何類型

如果 CityJSON 幾何只能學一種，那就是 MultiSurface。它是**最常見、最通用的幾何類型**，用於建築物外牆、屋頂、地面、土地界線等。`boundaries` 深度為 3[^cityjson-specs-geometry]。

### 基本範例（無洞）

三面獨立的矩形牆面：

```json
{
  "type": "MultiSurface",
  "lod": "2",
  "boundaries": [
    [[0, 3, 2, 1]],
    [[4, 5, 6, 7]],
    [[0, 1, 5, 4]]
  ]
}
```

每個 surface 由一個陣列描述，陣列內的第一個子陣列是**外環（exterior ring）**——也就是 polygon 的輪廓。

### 含洞的 Surface

有時候牆上開了窗戶，需要「挖洞」——這就是內環（interior ring）的用途[^cityjson-geom-arrays]：

```json
{
  "type": "MultiSurface",
  "lod": "2.1",
  "boundaries": [
    [
      [1, 2, 3, 4, 5],
      [6, 8, 7],
      [9, 12, 11, 10]
    ],
    [[14, 15, 16]]
  ]
}
```

第一個 surface 有三個 ring：`[1,2,3,4,5]` 是外環，`[6,8,7]` 和 `[9,12,11,10]` 是內環（兩個洞）。第二個 surface 只有一個外環，沒有洞。

### 真實範例：橋樑

一座箱型橋樑的六個面[^cityjson-specs-geometry]：

```json
{
  "type": "MultiSurface",
  "lod": "2",
  "boundaries": [
    [[0, 3, 2, 1]],
    [[4, 5, 6, 7]],
    [[0, 1, 5, 4]],
    [[1, 2, 6, 5]],
    [[2, 3, 7, 6]],
    [[3, 0, 4, 7]]
  ]
}
```

注意：雖然六個面看起來像一個盒子，但 MultiSurface 裡的每個 surface 都是**獨立**的，並不會自動檢查它們是否形成封閉空間。要表達封閉體積，需要用明天要介紹的 Solid。

**誰可以用 MultiSurface？** 幾乎所有城市物件都可以——Building、Road、Railway、LandUse、PlantCover、Bridge、Tunnel、CityFurniture……多到列不完[^cityjson-specs-objects]。

---

## CompositeSurface — 有連通性要求的 MultiSurface

CompositeSurface 的**資料結構和 MultiSurface 完全一樣**，都是深度 3 的巢狀陣列。關鍵差異在語意上[^cityjson-specs-geometry]：

> CompositeSurface 中的每個 surface 必須**共享邊界**（topologically connected），形成一個連續的面殼。

```json
{
  "type": "CompositeSurface",
  "lod": "1.3",
  "boundaries": [
    [[0, 3, 2, 1]],
    [[4, 5, 6, 7]],
    [[0, 1, 5, 4]]
  ]
}
```

### 最重要的應用：地形網格（TINRelief）

TINRelief（三角網不規則地形）**只能用 CompositeSurface**[^cityjson-specs-geometry]：

```json
{
  "type": "CompositeSurface",
  "lod": "1",
  "boundaries": [
    [[0, 3, 2]],
    [[4, 5, 6]],
    [[1, 2, 6]],
    [[2, 3, 7]],
    [[3, 0, 4]]
  ]
}
```

地形網格由連續的三角形構成，每個三角形和鄰近三角形共享邊緣——這正是 CompositeSurface 的典型情境。雖然規格書說「實務上任何 CompositeSurface 都可以用於地形，不一定要三角形」，但三角網格是最常見的形式[^cityjson-specs-geometry]。

### MultiSurface vs CompositeSurface

| 比較 | MultiSurface | CompositeSurface |
|------|-------------|-----------------|
| 面之間關係 | 獨立，可不接觸 | 相連，共享邊緣 |
| boundaries 深度 | 3 | 3 |
| JSON 結構 | 完全相同 | 完全相同 |
| 驗證工具 | 不檢查連通性 | val3dity 會檢查 |
| 主要用途 | 建築外牆、屋頂、土地 | 地形網格、連續面殼 |

一句話記法：**MultiSurface 是「一盤散沙」，CompositeSurface 是「拼圖」**。

---

## 從幾何到物件的限制

不是所有城市物件都可以用任意的幾何類型。規格書對每個物件類型都有明確的**允許幾何清單**[^cityjson-specs-objects]。例如：

- **Building** 只能用 `MultiSurface`、`CompositeSurface`、`Solid`、`MultiSolid`——不能用 `MultiPoint` 或 `MultiLineString`
- **Road** 同上（加上 `CompositeSolid`）
- **SolitaryVegetationObject** 可以用全部 7 種（含 `MultiPoint`）
- **TINRelief** 只能用 `CompositeSurface`

這些限制確保了資料的語意正確性——你不會把一條路誤標成一串點。

---

## 小結

今天我們學到了四種基礎幾何類型：

- **MultiPoint**（深度 1）— 點集合，用於定位
- **MultiLineString**（深度 2）— 線集合，用於道路、管線
- **MultiSurface**（深度 3）— 面集合，最常用，各面獨立
- **CompositeSurface**（深度 3）— 相連的面集合，用於地形網格

核心重點是 **boundaries 的深度規則**——每一層陣列對應一種幾何層次，從點（1）到線（2）到面（3）到體（4/5）。理解這個深度規則，讀懂任何 CityJSON 檔案都不再是問題。

明天進軍實體幾何：Solid、CompositeSolid、MultiSolid！

---

## 參考資料

[^cityjson-specs-geometry]: Ledoux, H., & Dukai, B. (2023). CityJSON Specifications 2.0.0: §3 Geometry Objects. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#geometry-objects

[^cityjson-specs-boundaries]: Ledoux, H., & Dukai, B. (2023). CityJSON Specifications 2.0.0: §3.2 Arrays to represent boundaries. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#arrays-to-represent-boundaries

[^cityjson-specs-objects]: Ledoux, H., & Dukai, B. (2023). CityJSON Specifications 2.0.0: §2 City Objects. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#city-objects

[^cityjson-geom-arrays]: CityJSON. (n.d.). Geometric primitives and depth of arrays. Retrieved 2026-09-27, from https://www.cityjson.org/dev/geom-arrays/

[^val3dity]: Ledoux, H. (2022). val3dity: Validation of 3D city models. Retrieved 2026-09-27, from https://github.com/tudelft3d/val3dity