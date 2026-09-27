# Day 24｜CityJSONSeq (JSON Lines) — 串流處理大型城市模型

## 前言

在過去 23 天，我們處理的 CityJSON 檔案都是單一檔案、一次載入記憶體——這對一個街區（幾十到幾百棟建築物）來說完全沒問題。

但現實世界的 3D 城市模型往往是**全國級**的。荷蘭的 3DBAG 資料集涵蓋超過 **1000 萬棟**建築物[^3dbag]，如果把它們全部塞進一個 CityJSON 檔案，檔案大小可能達到數十 GB。這在實務上會遇到兩個核心問題：

1. **記憶體限制**——單一 CityJSON 檔案需要將所有頂點與物件一次載入記憶體，數十 GB 的檔案會輕易吃掉伺服器或筆電的所有記憶體。
2. **串流友善性**——如果你只要查詢特定區域的建築物，何必下載整個檔案？

今天要介紹的 **CityJSONSeq（JSON Lines 序列格式）** 正是為了解決這兩個問題而設計的。

---

## 為什麼傳統 JSON 無法串流？

先回想一個標準的 CityJSON 檔案結構：

```json
{
  "type": "CityJSON",
  "version": "2.0",
  "CityObjects": {
    "id-1": { ... },
    "id-2": { ... },
    ...
    "id-1M": { ... }
  },
  "vertices": [ ... ]
}
```

所有城市物件被包在一個巨大的 `CityObjects` 物件中。要解析這個檔案，你必須**先把整個檔案載入記憶體**，然後才能開始處理任何一個物件[^json-spec]。這在處理百萬建築時完全不可行。

這就是為什麼我們需要 **JSON Lines**（又稱 NDJSON，Newline Delimited JSON）[^jsonlines]——它將每一行獨立為一個完整的 JSON 物件，使得程式可以「逐行讀取、逐行處理」，永遠不需要將整個檔案載入記憶體。

---

## CityJSONSeq 格式

CityJSON 基於 JSON Lines 定義了自己的串流格式：**CityJSONSeq**[^cityjsonseq-specs]。它的檔案副檔名建議使用 **`.city.jsonl`**。

### 格式的核心規則

一個 CityJSONSeq 檔案遵循以下五條規則[^cityjsonseq-specs]：

1. 每個 JSON 物件必須符合 JSON 標準，採用 UTF-8 編碼。
2. 每個 JSON 物件後必須緊接一個換行字元（LF: `'\n'`）。
3. JSON 物件內部不得包含換行字元（即每行一個完整的 JSON 物件）。
4. **第一個 JSON 物件必須是 `"CityJSON"` 型別**——作為「表頭」（header），包含 CRS、transform、geometry templates 等 metadata，但 `"CityObjects"` 與 `"vertices"` 為空。
5. **其餘所有 JSON 物件必須是 `"CityJSONFeature"` 型別。**

用最簡單的方式呈現：

```json
{"type":"CityJSON","version":"2.0","transform":{"scale":[1,1,1],"translate":[0,0,0]},"metadata":{"referenceSystem":"https://www.opengis.net/def/crs/EPSG/0/7415"},"CityObjects":{},"vertices":[]}
{"type":"CityJSONFeature","id":"a","CityObjects":{...},"vertices":[...]}
{"type":"CityJSONFeature","id":"b","CityObjects":{...},"vertices":[...]}
{"type":"CityJSONFeature","id":"c","CityObjects":{...},"vertices":[...]}
```

第一行是 header（`"CityJSON"` 型別），後面每一列都是一筆獨立的 feature（`"CityJSONFeature"` 型別）。

每個 CityJSONFeature 都是一個 **獨立的、自包含的** 特徵——例如一棟建築物及其 children（BuildingPart、BuildingInstallation）[^cityjsonfeature-specs]。

---

## CityJSONFeature 物件結構

一個 CityJSONFeature 物件就是一個獨立的 CityJSON 特徵，具備以下成員[^cityjsonfeature-specs]：

| 成員 | 必填 | 說明 |
|------|------|------|
| `type` | ✅ | 固定為 `"CityJSONFeature"` |
| `id` | ✅ | 字串，代表這個 feature 的識別碼（通常是 parent City Object 的 ID） |
| `CityObjects` | ✅ | 一個或多個 City Object 的集合（parent + 所有 children，遞迴包含） |
| `vertices` | ✅ | **僅限此 feature 的頂點**（整數座標），索引從 0 開始 |
| `appearance` | ❌ | 紋理與材質資訊（如有） |

### 一個完整的 CityJSONFeature 範例

```json
{
  "type": "CityJSONFeature",
  "id": "id-1",
  "CityObjects": {
    "id-1": {
      "type": "Building",
      "attributes": { "roofType": "gabled roof" },
      "children": ["mypart"],
      "geometry": [
        {
          "type": "Solid",
          "lod": "2.2",
          "boundaries": [ [ [0, 1, 2, 3, 4, 5] ] ]
        }
      ]
    },
    "mypart": {
      "type": "BuildingPart",
      "parents": ["id-1"],
      "children": ["mybalcony"],
      "geometry": [ ... ]
    },
    "mybalcony": {
      "type": "BuildingInstallation",
      "parents": ["mypart"],
      "geometry": [ ... ]
    }
  },
  "vertices": [
    [0, 0, 0], [10, 0, 0], [10, 10, 0], [0, 10, 0],
    [0, 0, 5], [10, 0, 5], [10, 10, 5], [0, 10, 5]
  ]
}
```

注意到 `vertices` 陣列的索引是**相對於這個 feature 本身**的。不同 feature 之間的頂點索引彼此獨立，沒有關聯。

---

## Header 物件的重要角色

CityJSONFeature 本身**不包含** transform、version、metadata、geometry-templates、extensions 等資訊[^cityjsonseq-specs]。這些資訊由第一行的 header CityJSON 物件提供。

因此，任何處理 CityJSONSeq 的程式都必須：

1. 先讀取 header，取得 transform 與 CRS 等全域資訊。
2. 對每個後續的 CityJSONFeature，使用 header 的 transform 將 local 整數頂點還原為真實世界座標。

公式仍然是熟悉的 `v[i] = (vi[i] * scale[i]) + translate[i]`[^transform-specs]。

### header 的結構

header 就是一個合法的 CityJSON 物件，差別只在於 `CityObjects` 和 `vertices` 必須為空：

```json
{
  "type": "CityJSON",
  "version": "2.0",
  "transform": {
    "scale": [0.001, 0.001, 0.001],
    "translate": [121000.0, 2500000.0, 0.0]
  },
  "metadata": {
    "referenceSystem": "https://www.opengis.net/def/crs/EPSG/0/7415",
    "geographicalExtent": [121000.0, 2500000.0, 0.0, 122000.0, 2501000.0, 100.0]
  },
  "CityObjects": {},
  "vertices": []
}
```

header 也可以包含 `"geometry-templates"`，讓所有 feature 共用相同的模板定義。

---

## 為什麼 CityJSONSeq 比單一檔案更省空間？

這聽起來可能違反直覺——把一個檔案拆成很多行，怎麼反而更小？

關鍵在於**去重複**。在標準 CityJSON 中，每個 City Object 的 vertices 都是全域的，頂點索引可能被多個幾何物件共用。而在 CityJSONSeq 中，每個 feature 的 vertices 都是**區域的**，feature 之間沒有共用頂點——這使得每個 feature 的資料更緊湊[^streaming-paper]。

根據 2024 年的研究論文，CityJSONSeq 不僅**比標準 CityJSON 節省約 10% 的磁碟空間**，在處理大型資料集時**處理時間降低一個數量級**，記憶體使用量也**大幅減少**[^streaming-paper]。

---

## 實戰：用 Python 串流讀取 CityJSONSeq

現在我們來實際動手寫一個 Python 串流處理的範例。這段程式展示了 CityJSONSeq 最核心的優勢——**逐行讀取、逐行處理，完全不佔用大量記憶體**。

### 基本串流讀取器

```python
import json


def stream_cityjsonseq(path: str):
    """讀取 .city.jsonl 檔案，逐行產生 header 與每個 feature。"""
    with open(path, "r") as f:
        # --- 第 1 行：header ---
        line = f.readline()
        if not line:
            raise ValueError("空檔案——第 1 行應為 CityJSON header")

        header = json.loads(line)
        if header.get("type") != "CityJSON":
            raise ValueError(f"第 1 行必須為 'CityJSON' 型別，"
                             f"實際為 '{header.get('type')}'")

        yield ("header", header)

        # --- 第 2 行起：CityJSONFeature ---
        for line_no, line in enumerate(f, start=2):
            line = line.strip()
            if not line:
                continue  # 跳過空行

            feature = json.loads(line)
            if feature.get("type") != "CityJSONFeature":
                print(f"⚠ 第 {line_no} 行跳過：非預期型別 "
                      f"'{feature.get('type')}'")
                continue

            yield ("feature", feature)
```

### 使用範例：統計建築物數量

```python
# 統計 dataset 中所有建築物的數量
building_count = 0
transform = None

for obj_type, obj in stream_cityjsonseq("3dbag_b2.city.jsonl"):
    if obj_type == "header":
        transform = obj.get("transform", {})
        print(f"Header 載入完成 | CRS: "
              f"{obj['metadata'].get('referenceSystem', 'unknown')}")
        print(f"Transform: scale={transform.get('scale')}, "
              f"translate={transform.get('translate')}")
    else:
        # 檢查這個 feature 中是否有 Building 型別的物件
        for co_id, co_obj in obj["CityObjects"].items():
            if co_obj["type"] == "Building":
                building_count += 1

print(f"\n統計結果：共 {building_count} 棟建築物")
```

即使 dataset 包含 1000 萬棟建築物，這段程式的記憶體使用量也只會停留在 **KB 等級**——每次只處理一行 JSON。

### 還原真實世界座標

要將 feature 的 local 整數頂點轉換為真實世界座標，使用 header 的 transform：

```python
import json
import numpy as np


def read_cityjsonseq_georeferenced(path: str):
    """讀取 CityJSONSeq 並還原真實世界座標。"""
    with open(path, "r") as f:
        header = json.loads(f.readline())
        if header.get("type") != "CityJSON":
            raise ValueError("第 1 行必須為 CityJSON header")

        scale = np.array(header["transform"]["scale"], dtype=np.float64)
        translate = np.array(header["transform"]["translate"], dtype=np.float64)

        for line in f:
            line = line.strip()
            if not line:
                continue

            feature = json.loads(line)
            if feature.get("type") != "CityJSONFeature":
                continue

            # 將所有頂點從整數轉換為真實世界座標
            vertices_local = np.array(feature["vertices"], dtype=np.float64)
            vertices_world = vertices_local * scale + translate

            # 將轉換後的座標附加回 feature
            feature["_vertices_world"] = vertices_world

            yield feature

# 使用範例
for i, feature in enumerate(read_cityjsonseq_georeferenced("montreal.city.jsonl")):
    if i >= 5:
        break  # 只印前 5 筆
    print(f"Feature id={feature['id']}, "
          f"頂點數={len(feature['vertices'])}, "
          f"第一個頂點世界座標={feature['_vertices_world'][0]}")
```

### 串流篩選特定類型

由於串流處理的特性，我們可以在不載入整個資料集的情況下，即時篩選出需要的資料：

```python
import json


def filter_by_type(path: str, target_types: set):
    """串流篩選：只產生包含指定 City Object 型別的 feature。"""
    with open(path, "r") as f:
        # header 直接傳遞
        header = json.loads(f.readline())
        yield header

        for line in f:
            line = line.strip()
            if not line:
                continue

            feature = json.loads(line)
            # 檢查這個 feature 中有沒有目標型別
            if any(
                obj["type"] in target_types
                for obj in feature["CityObjects"].values()
            ):
                yield feature


# 只保留 Buildings
with open("only_buildings.city.jsonl", "w") as out:
    first = True
    for obj in filter_by_type("big_city.city.jsonl", {"Building", "BuildingPart"}):
        out.write(json.dumps(obj, separators=(",", ":")) + "\n")
```

---

## 工具支援

CityJSON 生態系中有多種工具可以處理 CityJSONSeq 格式。

### cjseq（Rust 專用工具）

**cjseq** 是一個用 Rust 撰寫的專用工具，專為 CityJSONSeq 設計[^cjseq]：

**CityJSON → CityJSONSeq：**
```bash
cjseq cat myfile.city.json > myfile.city.jsonl
```

**從 stdin 讀取：**
```bash
cat myfile.city.json | cjseq cat
```

**CityJSONSeq → CityJSON：**
```bash
cat data/*.city.jsonl | cjseq collect > hugefile.city.json
```

**串流篩選：**
```bash
# 依地理範圍篩選
cat myfile.city.jsonl | cjseq filter --bbox 85007 446179 85168 446290

# 依物件類型篩選
cat myfile.city.jsonl | cjseq filter --cotype Building

# 隨機取樣 10 筆
cat myfile.city.jsonl | cjseq filter --random 10

# 半徑搜尋（以某點為中心 500 公尺內）
cat myfile.city.jsonl | cjseq filter --radius 500 610 50.0
```

**排除特定類型：**
```bash
cat myfile.city.jsonl | cjseq filter --exclude BuildingPart > no_parts.city.jsonl
```

### cjio（Python 工具）

**cjio** 自 v0.8 起也支援 CityJSONSeq[^cjio]：

**讀取串流（stdin）：**
```bash
cat mystream.city.jsonl | cjio --suppress_msg stdin remove_materials save stdout
```

**輸出串流：**
```bash
cjio --suppress_msg myfile.city.json remove_materials export jsonl stdout | less
```

**儲存為檔案：**
```bash
cat myfile.city.json | cjio --suppress_msg stdin crs_reproject 7415 export jsonl output.city.jsonl
```

### 比較：cjseq vs cjio

| 功能 | cjseq（Rust） | cjio（Python） |
|------|-------------|---------------|
| 語言 | Rust | Python |
| 定位 | 專為 CityJSONSeq 設計 | 通用 CityJSON 處理 |
| 轉為 CityJSONSeq | `cjseq cat` | `cjio ... export jsonl` |
| 轉回 CityJSON | `cjseq collect` | `cjio stdin ... save` |
| 串流篩選 | 內建 `filter` | 用 `subset` + pipe |
| 效能 | 極快 | 較慢 |

### 驗證

CityJSONSeq 檔案也可以驗證[^cjval]：

```bash
cjseq cat -f myfile.city.json | cjval --verbose
```

或是直接上傳到 [validator.cityjson.org](https://validator.cityjson.org) 進行線上驗證。

---

## 真實案例：荷蘭 3DBAG

荷蘭的 **3DBAG** 資料集是 CityJSONSeq 最經典的應用案例[^3dbag]。這個資料集涵蓋全荷蘭約 **1000 萬棟建築物**，提供 LoD 1.2、2.2 與 3.0 的 3D 模型。

3DBAG 團隊提供 CityJSONSeq 格式的資料下載，每個省份一個 `.city.jsonl` 檔案。使用者可以：

1. **串流下載**指定區域的資料，無需一次下載整個國家。
2. **即時篩選**特定類型或區域的建築物。
3. **低記憶體處理**——即使在 8GB 記憶體的筆電上也能處理全國資料。

```bash
# 下載 Utrecht 省的資料，只保留 LoD 2.2 的建築物
curl -O https://data.3dbag.nl/cityjsonseq/utrecht.city.jsonl.gz
gunzip utrecht.city.jsonl.gz
cat utrecht.city.jsonl | cjseq filter --cotype Building > utrecht_buildings.city.jsonl
```

---

## 設計限制與注意事項

使用 CityJSONSeq 時有幾個限制需要留意：

1. **頂點是區域的**——每個 feature 的頂點索引從 0 開始，無法跨 feature 共用頂點。這意味著 CityJSONSeq 不適合需要全域頂點分析的情境。

2. **沒有跨 feature 的幾何參考**——如果兩棟建築物共用一面牆（鄰居關係），在 CityJSONSeq 中無法表達這個共用關係，因為每個 feature 的幾何都是獨立的。

3. **處理前需先讀取 header**——如果不先讀取 header，就無法正確將座標還原為真實世界座標。這在分散式處理中需要額外注意（例如，每個 worker 節點都需要 header 資訊）。

4. **不支援多行 JSON**——每個 feature 必須在一行內完成。這對非常複雜的幾何物件可能不太友善，但可以透過提高 `"coordinatePrecision"` 來控制頂點數。

---

## 結語

CityJSONSeq 是處理大規模 3D 城市模型的關鍵技術。它將「一次載入、全量處理」的傳統 JSON 思維，轉變為「逐行讀取、串流處理」的現代資料處理模式。

今天我們學到了：

1. **問題**——標準 CityJSON 檔案在數百萬建築規模下會遇到記憶體瓶頸。
2. **解決方案**——CityJSONSeq 基於 JSON Lines 實現串流處理，header 存放全域資訊，每個 feature 獨立存在。
3. **Python 實作**——只需 `json.loads()` 逐行解析即可，不需要任何特殊函式庫。
4. **工具生態**——cjseq（Rust）提供高效能的轉換與篩選，cjio（Python）提供串流輸出支援。
5. **真實案例**——荷蘭 3DBAG 資料集證明了 CityJSONSeq 在國家級資料處理中的實用性。

明天我們將深入探討**大型檔案處理策略**——包括地理分割（tiling）、cjio 的進階操作，以及如何整合 CityJSONSeq 與分割策略來處理真實世界的百萬級資料集。

---

## 參考資料

[^3dbag]: 3DBAG. (n.d.). 3DBAG — 3D BAG van heel Nederland. Retrieved 2026-09-27, from https://3dbag.nl

[^json-spec]: Bray, T. (2017). RFC 8259: The JavaScript Object Notation (JSON) Data Interchange Format. Internet Engineering Task Force. Retrieved 2026-09-27, from https://datatracker.ietf.org/doc/html/rfc8259

[^jsonlines]: JSON Lines. (n.d.). JSON Lines — JSON-based format for streaming. Retrieved 2026-09-27, from https://jsonlines.org

[^cityjsonseq-specs]: CityJSON. (2024). CityJSON Specification 2.0.2 — §7.2 Text sequences and streaming with CityJSONFeature. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#cityjsonfeature-object

[^cityjsonfeature-specs]: CityJSON. (2024). CityJSON Specification 2.0.2 — CityJSONFeature Object. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#cityjsonfeature-object

[^transform-specs]: CityJSON. (2024). CityJSON Specification 2.0.2 — §5 The Transform Object. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#transform-object

[^streaming-paper]: Ledoux, H., Stavropoulou, G., & Dukai, B. (2024). Streaming CityJSON datasets. arXiv:2407.00017. Retrieved 2026-09-27, from https://arxiv.org/abs/2407.00017

[^cjseq]: CityJSON. (2024). cjseq — Rust tool to process CityJSONSeq datasets. Retrieved 2026-09-27, from https://github.com/cityjson/cjseq

[^cjio]: CityJSON. (2024). cjio — CityJSON Command-line Interface (Python). Retrieved 2026-09-27, from https://github.com/cityjson/cjio

[^cjval]: CityJSON. (2024). cjval — CityJSON Validator. Retrieved 2026-09-27, from https://github.com/cityjson/cjval