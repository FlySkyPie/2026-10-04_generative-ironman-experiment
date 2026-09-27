# Day 4：CityJSON 的結構總覽 — 一次看懂 7 大成員

> 如果說 CityJSON 是一棟建築，那麼根層級的 9 個成員，就是這棟建築的鋼骨結構。今天我們先把藍圖攤開，之後再慢慢細看每個房間的細節。

---

## 從最小合法檔案說起

上一回我們用數據說明了 CityJSON 為什麼比 CityGML 輕量，今天我們直接解剖一個 CityJSON 檔案，看看它裡面長什麼樣子。

根據規格書[^cityjson-spec-v2]，一個最簡單的 CityJSON 檔案只需要 5 個成員：

```json
{
  "type": "CityJSON",
  "version": "2.0",
  "transform": {
    "scale": [1.0, 1.0, 1.0],
    "translate": [0.0, 0.0, 0.0]
  },
  "CityObjects": {},
  "vertices": []
}
```

這個檔案雖然沒有任何實際的城市物件（`CityObjects` 是空的），但它已經是一個**合法的** CityJSON 文件了。如果你把它丟進驗證器（`cjval`），它會告訴你：「這是一個正確的 CityJSON。」

## 完整但不一定有資料的 CityJSON 物件

如果我們把所有的標準成員都加上，即使它們都還是空的，會長這樣：

```json
{
  "type": "CityJSON",
  "version": "2.0",
  "extensions": {},
  "transform": {
    "scale": [1.0, 1.0, 1.0],
    "translate": [0.0, 0.0, 0.0]
  },
  "metadata": {},
  "CityObjects": {},
  "vertices": [],
  "appearance": {},
  "geometry-templates": {}
}
```

你可以把這個當作 CityJSON 的「空白模板」。真實的 CityJSON 檔案，就是在這個模板上填入實際的城市資料。

## 九大成員一覽

嚴格來說，CityJSON v2.0 規格書中定義的標準成員共有 9 個：5 個必要（Required）、4 個選用（Optional）。

| 成員 | 必要性 | 一句話用途 |
|------|--------|-----------|
| `type` | **必要** | 標記這是一個 CityJSON 檔案，值永遠是 `"CityJSON"` |
| `version` | **必要** | 宣告使用哪個版本（如 `"2.0"`） |
| `transform` | **必要** | 記錄整數座標與真實世界座標之間的換算公式 |
| `vertices` | **必要** | 所有 3D 頂點的集中營，幾何透過索引參考這裡 |
| `CityObjects` | **必要** | 整座城市的心臟——所有建築、道路、橋樑等物件的集合 |
| `metadata` | 選用 | 資料集的戶口名簿：CRS、範圍、建立者、日期等 |
| `appearance` | 選用 | 決定城市長什麼樣子：材質顏色和紋理貼圖 |
| `geometry-templates` | 選用 | 省空間的黑魔法：重複物件（樹、路燈）的模板 |
| `extensions` | 選用 | 標準不夠用時的擴充窗口 |

接下來我們逐一說明每個成員的定位。

---

### `type`：身分證

CityJSON 的 `type` 永遠是字串 `"CityJSON"`。這就像 JSON 的 `application/json` Content-Type 一樣，讓任何程式在讀取檔案時可以立即確認：「嗯，這是一個 CityJSON 格式的資料。[^cityjson-spec-v2]」

### `version`：版本聲明

格式為 `"X.Y"` 的字串，例如 `"2.0"`。注意它**不包含 patch 版本號**（也就是不會寫成 `"2.0.2"`），驗證時會自動使用該 minor 版本的最新 schema[^cityjson-spec-v2]。

### `transform`：座標壓縮的鑰匙

這可能是 CityJSON 最聰明的設計之一。為了讓檔案更小，所有頂點座標都以**整數**儲存，然後透過 `transform` 裡的 `scale` 和 `translate` 還原回真實座標[^transform-spec]：

```
v[0] = (vi[0] × scale[0]) + translate[0]
v[1] = (vi[1] × scale[1]) + translate[1]
v[2] = (vi[2] × scale[2]) + translate[2]
```

這套機制借用了 TopoJSON 的 **quantization（量化）** 概念[^topojson-spec]。例如 `scale: [0.001, 0.001, 0.001]` 表示保留 3 位小數（公釐級精確度），儲存時只需存 `[123456, 456789, 789012]` 這樣的整數，而非 `[123456.789, 456789.012, 789012.345]` 這樣的小數。

實際例子：

```json
"transform": {
  "scale": [0.001, 0.001, 0.001],
  "translate": [442464.879, 5482614.692, 310.19]
}
```

如果有一個頂點儲存為 `[0, 0, 0]`，經過還原後會變成 `(442464.879, 5482614.692, 310.19)`。

### `vertices`：頂點集中營

這是一個陣列的陣列（array of arrays），每一筆都是 `[x, y, z]` 的整數座標[^cityjson-spec-v2]：

```json
"vertices": [
  [0, 0, 0],
  [100, 0, 0],
  [100, 100, 0],
  [0, 100, 0],
  [0, 0, 100]
]
```

CityJSON 的幾何在描述形狀時，並不會重複寫出座標值，而是透過**索引**來參考這個頂點陣列——0 代表第一個頂點、1 代表第二個，依此類推。這套索引機制源自 Wavefront OBJ 格式[^obj-format]，也是 CityJSON 能大幅縮小檔案體積的關鍵之一。

### `CityObjects`：城市的心臟

這是最重要的成員，也是一份城市模型中**真正的主角**。它的值是一個物件（key-value pairs），每一個 key 是城市物件的 ID，value 則是該物件的完整描述——包含其類型（Building、Road、Bridge 等）、屬性（attributes）、幾何形狀（geometry）、以及與其他物件的關聯[^cityjson-spec-v2]。

```json
"CityObjects": {
  "my-building-id-1": {
    "type": "Building",
    "attributes": {
      "storeysAboveGround": 5,
      "roofType": "gabled"
    },
    "geometry": [...],
    "children": ["my-building-part-1"]
  }
}
```

明後天我們會更深入探討這個成員，因為整座城市的靈魂都在這裡。

### `metadata`：資料集的戶口名簿

雖然選用，但一個好的資料集通常都會帶上 metadata。它記錄了[^metadata-spec]：

- **`geographicalExtent`**：資料集的 3D 範圍，格式為 `[minX, minY, minZ, maxX, maxY, maxZ]`
- **`identifier`**：資料集的唯一識別碼（建議使用 UUID）
- **`pointOfContact`**：聯絡人資訊（姓名、組織、Email 等）
- **`referenceDate`**：資料集的參考日期
- **`referenceSystem`**：座標參考系統（CRS），如 `https://www.opengis.net/def/crs/EPSG/0/4326`
- **`title`**：資料集的標題

```json
"metadata": {
  "geographicalExtent": [84710.1, 446846.0, -5.3, 84757.1, 446944.0, 40.9],
  "identifier": "eaeceeaa-3f66-429a-b81d-bbc6140b8c1c",
  "referenceSystem": "https://www.opengis.net/def/crs/EPSG/0/2355",
  "title": "Buildings in LoD2.3 of Chibougamau, Québec"
}
```

這些資訊遵循 ISO19115 國際標準[^iso19115]，確保與其他地理資訊系統相容。

### `appearance`：城市的美容師

如果資料集包含材質或紋理，就會出現在這裡。CityJSON 支援兩種外觀表現方式[^appearance-spec]：

- **Materials**：指定表面的顏色、光澤度、透明度等（遵循 X3D 標準）
- **Textures**：將圖片貼到表面上（遵循 COLLADA 標準）

外觀資料會與幾何表面建立連結，告訴渲染引擎「這個面是紅色的瓦片材質，那個面是玻璃材質」。

### `geometry-templates`：省空間的黑魔法

在一個城市模型中，有許多物體是重複出現的——同一款路燈、同一種行道樹。如果每一棵樹都儲存完整的幾何，檔案會非常龐大。

`geometry-templates` 允許你先定義一套「模板」，然後用 **GeometryInstance** 在指定位置「實例化」它[^geometry-templates-spec]：

```json
"geometry-templates": {
  "templates": {
    "我的路燈模板": {
      "type": "MultiSurface",
      "boundaries": [...]
    }
  }
}
```

每個實例只需要記錄：參考哪個模板、在哪個位置、旋轉角度和縮放比例——用一個 transformation matrix 搞定。這個概念相當於 CityGML 中的 Implicit Geometry[^citygml-implicit-geometry]。

### `extensions`：擴充窗口

當標準的城市物件類型不夠用時，Extensions 機制可以擴充 CityJSON 的能力。一個 Extension 可以做到 4 件事[^extensions-spec]：

1. 在 root 層級新增成員
2. 為現有的 City Object 加上自訂屬性
3. 定義新的語意物件（Semantic Object）
4. 定義新的 City Object 類型

```json
"extensions": {
  "我的擴充": {
    "url": "https://example.com/my-extension.json",
    "version": "1.0"
  }
}
```

與 CityGML 的 ADE（Application Domain Extension）相比，CityJSON 的 Extensions 更簡潔且對軟體友善——遵循 6 條簡單規則即可讓標準的 CityJSON 解析器正確讀取[^extensions-spec]。

## 一個實際的小範例

讓我們把這些成員拼在一起，產生一個有實際內容的簡單 CityJSON：

```json
{
  "type": "CityJSON",
  "version": "2.0",
  "transform": {
    "scale": [1.0, 1.0, 1.0],
    "translate": [0.0, 0.0, 0.0]
  },
  "vertices": [
    [0, 0, 0], [10, 0, 0], [10, 10, 0], [0, 10, 0],
    [0, 0, 10], [10, 0, 10], [10, 10, 10], [0, 10, 10]
  ],
  "CityObjects": {
    "building-1": {
      "type": "Building",
      "attributes": { "name": "Tutorial Building" },
      "geometry": [
        {
          "type": "Solid",
          "lod": "2",
          "boundaries": [[[0, 1, 2, 3], [4, 5, 6, 7], [0, 1, 5, 4],
                          [1, 2, 6, 5], [2, 3, 7, 6], [3, 0, 4, 7]]]
        }
      ]
    }
  },
  "metadata": {
    "title": "教學用簡單建築物",
    "referenceSystem": "https://www.opengis.net/def/crs/EPSG/0/4326"
  }
}
```

這個檔案定義了一棟 LoD2 的立方體建築物，8 個頂點，6 個面。明天我們會從這個小小的範例出發，逐步建構出更真實的 CityJSON 資料集。

## 小結

今天我們了解了 CityJSON 的根層級骨架：

| 成員 | 必要？ | 功能定位 |
|------|--------|---------|
| `type` | ✅ | 格式身分證 |
| `version` | ✅ | 版本聲明 |
| `transform` | ✅ | 座標壓縮鑰匙 |
| `vertices` | ✅ | 頂點集中營 |
| `CityObjects` | ✅ | 城市的心臟 |
| `metadata` | — | 資料戶口名簿 |
| `appearance` | — | 材質與紋理 |
| `geometry-templates` | — | 重複物件模板 |
| `extensions` | — | 擴充窗口 |

5 個必要、4 個選用——掌握了這 9 個成員，你就掌握了一座 3D 城市模型的骨架。

明天，我們將用手寫一個完整的 CityJSON 檔案，從繪製建築物的每個面開始，學習如何定義幾何形狀，並用線上檢視器看到成果。

---

## 參考資料

[^cityjson-spec-v2]: CityJSON. (n.d.). CityJSON v2.0.2 Specification: The CityJSON Object. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/

[^transform-spec]: CityJSON. (n.d.). CityJSON v2.0.2 Specification: Transform Object. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#transform-object

[^topojson-spec]: TopoJSON. (n.d.). TopoJSON Format Specification: Transforms. Retrieved 2026-09-27, from https://github.com/topojson/topojson-specification/blob/master/README.md#212-transforms

[^obj-format]: Wikipedia. (n.d.). Wavefront .obj file format. Retrieved 2026-09-27, from https://en.wikipedia.org/wiki/Wavefront_.obj_file

[^metadata-spec]: CityJSON. (n.d.). CityJSON v2.0.2 Specification: Metadata. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#metadata

[^iso19115]: International Organization for Standardization. (2003). ISO 19115:2003 Geographic information — Metadata. Retrieved 2026-09-27, from https://www.iso.org/standard/53798.html

[^appearance-spec]: CityJSON. (n.d.). CityJSON v2.0.2 Specification: Appearance Object. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#appearance-object

[^geometry-templates-spec]: CityJSON. (n.d.). CityJSON v2.0.2 Specification: Geometry Templates. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#geometry-templates

[^citygml-implicit-geometry]: Kolbe, T. H., Gröger, G., Plümer, L. (2005). CityGML: Interoperable Access to 3D City Models. In: *Geo-information for Disaster Management* (pp. 883–899). Springer.

[^extensions-spec]: CityJSON. (n.d.). CityJSON v2.0.2 Specification: Extensions. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#extensions