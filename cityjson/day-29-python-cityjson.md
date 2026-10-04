# Day 29｜用 Python 處理 CityJSON — 從解析到實戰

## 前言

經過 28 天的學習，你已經了解 CityJSON 的格式結構、幾何核心、語意表面、外觀材質、工具生態，甚至走訪了全球的公開資料集。但到目前為止，我們大多透過命令列工具（cjio、citygml-tools）來操作資料。

今天，我們要進入**程式化處理**的世界——直接用 Python 來讀取、分析、轉換 CityJSON 資料。這才是釋放 3D 城市模型真正潛力的關鍵：**用程式批量處理數百萬棟建築物、自動計算屋頂面積、或將資料轉換為 3D Tiles 串流格式**。

---

## 方法一：使用內建 `json` 模組（零依賴）

最直接的方式是使用 Python 標準函式庫的 `json` 模組——CityJSON 本身就是合法的 JSON，不需要任何第三方套件就能讀取[^cityjson-spec]。

```python
import json

with open("zurich.city.json", "r") as f:
    data = json.load(f)

print(data["type"])       # "CityJSON"
print(data["version"])    # "2.0"
print(f"頂點數：{len(data['vertices'])}")
print(f"城市物件數：{len(data['CityObjects'])}")
```

### 解壓縮頂點座標

CityJSON 的頂點儲存為整數（經過 quantization），需搭配 `transform` 才能得到真實世界座標[^cityjson-transform]：

```python
def decompress_vertices(data):
    """將整數頂點還原為真實世界座標"""
    if "transform" not in data:
        return data["vertices"]

    scale = data["transform"]["scale"]
    translate = data["transform"]["translate"]
    return [
        [
            v[0] * scale[0] + translate[0],
            v[1] * scale[1] + translate[1],
            v[2] * scale[2] + translate[2],
        ]
        for v in data["vertices"]
    ]

V = decompress_vertices(data)
print(f"第一個頂點（真實座標）：{V[0]}")
```

---

## 方法二：使用 cjio Python API（推薦）

雖然 cjio 主要是命令列工具，但它也提供了完整的 Python API，位於 `cjio.cityjson` 模組，包含 `CityJSON` 類別與多種操作方法[^cjio-api]。

### 安裝

```bash
pip install 'cjio[export,reproject,validate]'
```

### 讀取 CityJSON 檔案

```python
from cjio import cityjson

with open("zurich.city.json", "r") as f:
    cm = cityjson.reader(file=f)

# 基本資訊
print(f"版本：{cm.get_version()}")
print(f"EPSG：{cm.get_epsg()}")
print(f"Bbox：{cm.get_bbox()}")
```

`cm.j` 就是原始的 JSON dict，你可以像操作一般 dict 一樣存取所有資料。

### 存取 CityObjects

```python
cos = cm.j["CityObjects"]

# 逐一列出所有物件
for co_id, co_obj in cos.items():
    print(f"{co_id}: {co_obj['type']}")
    if "attributes" in co_obj:
        print(f"  屬性：{list(co_obj['attributes'].keys())}")
    geom_count = len(co_obj.get("geometry", []))
    print(f"  幾何數量：{geom_count}")
```

---

## 實作 1：統計各類型物件數量

這是最基礎但也最實用的分析——了解資料集中有哪些城市物件、各有多少[^cityjson-cityobjects]：

```python
from collections import Counter
import json

def count_object_types(data):
    """統計 CityJSON 中各類物件的數量"""
    type_counts = Counter()

    for co_id, co_obj in data["CityObjects"].items():
        type_counts[co_obj["type"]] += 1

    return type_counts

with open("3dbag_tile.city.json", "r") as f:
    data = json.load(f)

counts = count_object_types(data)
total = sum(counts.values())

print(f"總物件數：{total}")
print(f"物件類型分布（前 10）：")
for obj_type, count in counts.most_common(10):
    print(f"  {obj_type}: {count}")
```

輸出範例：
```
總物件數：1,284
物件類型分布（前 10）：
  Building: 956
  BuildingPart: 218
  Road: 62
  SolitaryVegetationObject: 35
  CityFurniture: 13
```

---

## 實作 2：計算所有 RoofSurface 的總面積

這是一個真實的應用場景——都市太陽能潛力評估需要知道所有屋頂的可用面積。要計算屋頂面積，需要走訪每個 CityObject 的幾何、檢查語意表面類型為 `RoofSurface` 的 surface，然後計算其 3D 多邊形面積[^roof-surface-tutorial]。

### 3D 多邊形面積計算（Newell's Method）

一般 2D 多邊形面積公式只適用於 XY 平面投影，但屋頂是有斜度的（pitch）。我們需要使用 Newell's Method 計算真正的 3D 面積[^newell-method]：

```python
import numpy as np

def polygon_area_3d(pts):
    """使用 Newell's Method 計算 3D 多邊形面積"""
    n = np.zeros(3)
    for i in range(len(pts)):
        a, b = pts[i], pts[(i + 1) % len(pts)]
        n += np.cross(a, b)
    return 0.5 * np.linalg.norm(n)
```

### 完整屋頂面積計算

```python
def compute_total_roof_area(data, V):
    """計算 CityJSON 中所有 RoofSurface 的總面積"""
    total_area = 0.0
    roof_count = 0

    for co_id, co_obj in data["CityObjects"].items():
        for geom in co_obj.get("geometry", []):
            # 只處理有語意表面的幾何（LoD2 以上）
            semantics = geom.get("semantics")
            if semantics is None:
                continue

            surfaces = semantics["surfaces"]
            values = semantics["values"]
            boundaries = geom["boundaries"]
            geom_type = geom["type"]

            if geom_type == "MultiSurface":
                # boundaries[surface_index][ring][vertex_index]
                for surf_idx, surf in enumerate(boundaries):
                    if values[surf_idx] is None:
                        continue
                    surface_type = surfaces[values[surf_idx]]["type"]
                    if surface_type == "RoofSurface":
                        outer_ring = V[surf[0]]
                        area = polygon_area_3d(outer_ring)
                        total_area += area
                        roof_count += 1

            elif geom_type == "Solid":
                # boundaries[shell][face][ring][vertex_index]
                for shell in boundaries:
                    for face_idx, face in enumerate(shell):
                        si = values[0][face_idx] if values and values[0] else None
                        if si is not None and surfaces[si]["type"] == "RoofSurface":
                            outer_ring = V[face[0]]
                            area = polygon_area_3d(outer_ring)
                            total_area += area
                            roof_count += 1

    return total_area, roof_count

# 使用範例
with open("wijk.city.json", "r") as f:
    data = json.load(f)

V = decompress_vertices(data)
area, count = compute_total_roof_area(data, V)

print(f"RoofSurface 數量：{count}")
print(f"屋頂總面積：{area:.2f} 平方公尺")
print(f"平均每面面積：{area / count:.2f} 平方公尺" if count else "N/A")
```

### 使用 Shapely 計算 2D 足跡面積

如果你只需要 2D 投影面積（例如建築物足跡），可以利用 Shapely 函式庫簡化計算[^shapely-docs]：

```python
from shapely.geometry import Polygon

def compute_footprint_areas(data, V):
    """計算所有 Building 的 2D 足跡面積"""
    footprints = []

    for co_id, co_obj in data["CityObjects"].items():
        if co_obj["type"] != "Building":
            continue

        for geom in co_obj.get("geometry", []):
            semantics = geom.get("semantics")
            if semantics is None:
                continue

            surfaces = semantics["surfaces"]
            values = semantics["values"]
            boundaries = geom["boundaries"]

            for surf_idx, surf in enumerate(boundaries):
                si = values[surf_idx]
                if si is not None and surfaces[si]["type"] == "GroundSurface":
                    outer_ring = V[surf[0]]
                    poly = Polygon([(p[0], p[1]) for p in outer_ring])
                    footprints.append({
                        "id": co_id,
                        "area_m2": poly.area,
                        "num_vertices": len(outer_ring),
                    })
                    break  # 一個建築只取一個 GroundSurface

    return footprints

footprints = compute_footprint_areas(data, V)
total_footprint = sum(f["area_m2"] for f in footprints)
print(f"建築物數量：{len(footprints)}")
print(f"總足跡面積：{total_footprint:.2f} 平方公尺")
```

---

## 實作 3：將 CityJSON 轉換為 glTF / 3D Tiles

為了在 Web 瀏覽器或遊戲引擎中視覺化 3D 城市模型，最常見的做法是轉換為 **glTF**（GL Transmission Format）或 **3D Tiles**（Cesium 的串流格式）。

### 方法一：使用 cjio CLI 匯出 GLB

最簡單的方式是直接使用 cjio 的命令列匯出功能[^cjio-export]：

```bash
# 匯出為單一 GLB 檔案
cjio zurich.city.json export glb zurich.glb

# 匯出到目錄（自動以檔名命名）
cjio zurich.city.json export glb /output/dir/
```

### 方法二：使用 cjio Python API 匯出 GLB

```python
from cjio import cityjson

with open("model.city.json", "r") as f:
    cm = cityjson.reader(file=f)

# 匯出為 GLB（二進位 glTF）
glb_bytes = cm.export2glb(do_triangulate=True)
with open("model.glb", "wb") as out:
    out.write(glb_bytes)

# 匯出為 B3DM（Batched 3D Model，用於 3D Tiles）
b3dm_bytes = cm.export2b3dm()
with open("model.b3dm", "wb") as out:
    out.write(b3dm_bytes)
```

> **注意**：`export2glb()` 需要安裝 `triangle` 套件（在 `cjio[export]` 中會自動安裝）。匯出時會自動將 z-up 轉換為 y-up（glTF 慣例），並保留物件 ID 作為網格名稱。

### 方法三：使用 tyler 建立 3D Tiles（大規模資料集）

對於大型資料集（如全國 3DBAG），**tyler** 是更專業的選擇——它是一個 Rust 寫的命令列工具，由 3DGI 開發，專門將 CityJSON 轉換為 3D Tiles v1.1[^tyler-github]：

```bash
# 安裝（Docker）
docker pull 3dgi/tyler:0.4.1-beta1

# 將 CityJSON 目錄轉為 3D Tiles
docker run --rm -v /data:/data 3dgi/tyler:0.4.1-beta1 \
  tyler /data/cityjson/ \
  --output /data/3dtiles/ \
  --object-type Building \
  --object-type BuildingPart \
  --3dtiles-implicit
```

tyler 支援的功能包括：
- 隱式（implicit）與顯式（explicit）3D Tiles 兩種模式
- 每個 CityObject 的屬性嵌入 `EXT_mesh_features` 擴充
- 按物件類型上色（e.g. `--color-building-part #FF0000`）
- 輸出格式：3D Tiles（GLB content）、OBJ、CityJSON、CityJSONSeq

---

## 進階應用：完整處理管線

將所學串聯起來，寫一個完整的 Python 腳本——讀取 CityJSON → 統計物件類型 → 計算屋頂面積 → 輸出 GeoJSON 與 GLB：

```python
#!/usr/bin/env python3
"""CityJSON 分析管線"""
import json
import numpy as np
from collections import Counter

def decompress_vertices(data):
    if "transform" not in data:
        return np.array(data["vertices"], dtype=np.float64)
    s = data["transform"]["scale"]
    t = data["transform"]["translate"]
    return np.array(data["vertices"], dtype=np.float64) * s + t

def polygon_area_3d(pts):
    n = np.zeros(3)
    for i in range(len(pts)):
        n += np.cross(pts[i], pts[(i + 1) % len(pts)])
    return 0.5 * np.linalg.norm(n)

def analyze_cityjson(path):
    with open(path, "r") as f:
        data = json.load(f)

    V = decompress_vertices(data)
    cos = data["CityObjects"]

    # 1. 物件類型統計
    type_counts = Counter(co["type"] for co in cos.values())
    print(f"=== {path} ===")
    print(f"版本：{data['version']}")
    print(f"頂點數：{len(data['vertices'])}")
    print(f"物件總數：{sum(type_counts.values())}")
    print(f"物件類型分布：")
    for t, c in type_counts.most_common():
        print(f"  {t}: {c}")

    # 2. 屋頂面積計算
    total_roof = 0.0
    roof_count = 0
    for co_obj in cos.values():
        for geom in co_obj.get("geometry", []):
            sem = geom.get("semantics")
            if sem is None:
                continue
            surfaces = sem["surfaces"]
            values = sem["values"]
            for surf_idx, surf in enumerate(geom["boundaries"]):
                si = values[surf_idx] if geom["type"] == "MultiSurface" else values[0][surf_idx]
                if si is not None and surfaces[si]["type"] == "RoofSurface":
                    area = polygon_area_3d(V[surf[0]])
                    total_roof += area
                    roof_count += 1

    print(f"RoofSurface 數量：{roof_count}")
    print(f"屋頂總面積：{total_roof:.2f} m²")
    return data, V

if __name__ == "__main__":
    analyze_cityjson("zurich.city.json")
```

---

## 常用函式庫總覽

| 函式庫 | 用途 | 安裝指令 |
|--------|------|----------|
| **cjio** | CityJSON 讀取、分析、轉換、驗證（Python API + CLI） | `pip install cjio` |
| **cjvalpy** | CityJSON 驗證器核心（cjio validate 的後端） | `pip install cjvalpy` |
| **shapely** | 幾何操作（面積計算、空間查詢） | `pip install shapely` |
| **numpy** | 數值運算（頂點計算、Newell's Method） | `pip install numpy` |
| **pyproj** | 座標系統轉換（reprojection） | `pip install pyproj` |
| **triangle** | 多邊形三角化（glTF/GLB 匯出需要） | `pip install triangle` |
| **tyler** | CityJSON → 3D Tiles（Rust CLI，非 Python） | `docker pull 3dgi/tyler` |

---

## 小結

今天我們學會了三種用 Python 處理 CityJSON 的核心技能：

| 實作 | 技術 | 應用場景 |
|------|------|----------|
| **統計物件類型** | `json.load()` + `Counter` | 資料集品質檢查、內容摘要 |
| **計算屋頂面積** | Newell's Method + 語意表面走訪 | 太陽能評估、建材估算 |
| **轉換為 glTF/3D Tiles** | cjio API / tyler | Web 視覺化、Cesium 整合 |

CityJSON 作為純 JSON 格式的最大優勢在此展現：**你不需要學習專屬的 SDK 或 API**，Python 的標準函式庫就能上手。而 cjio Python API 則提供了更高層的抽象，讓處理 3D 城市模型變得像操作 Python dict 一樣直覺。

明天是系列的最後一天——我們會回顧這 30 天的旅程，總結 CityJSON 的學習路徑，並展望未來。

---

## 參考資料

[^cityjson-spec]: CityJSON Specification v2.0. (2024). The CityJSON Object. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#the-cityjson-object

[^cityjson-transform]: CityJSON Specification v2.0. (2024). Transform Object. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#the-transform-object

[^cityjson-cityobjects]: CityJSON Specification v2.0. (2024). City Objects. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.0/#city-objects

[^cjio-api]: cjio. (2025). cjio Python API Tutorial — Basics. Retrieved 2026-09-27, from https://cjio.readthedocs.io/en/stable/api_tutorial_basics.html

[^cjio-export]: cjio. (2025). Export to glTF / GLB. Retrieved 2026-09-27, from https://cjio.readthedocs.io/en/stable/index.html#generating-binary-gltf

[^roof-surface-tutorial]: 3D Geospatial. (2025). Extracting LOD2 Roof Surfaces from CityJSON. Retrieved 2026-09-27, from https://www.3d-geospatial.com/3d-geospatial-fundamentals-for-digital-twins/citygml-and-cityjson-processing/extracting-lod2-roof-surfaces-from-cityjson/

[^newell-method]: Newell, M. (1975). A method for the calculation of the area of a polygon on a sphere. Proceedings of SIGGRAPH 1975.

[^shapely-docs]: Shapely. (2025). Shapely Documentation — Polygon. Retrieved 2026-09-27, from https://shapely.readthedocs.io/en/stable/reference/shapely.Polygon.html

[^tyler-github]: 3DGI. (2025). tyler — Create 3D Tiles from CityJSON. Retrieved 2026-09-27, from https://github.com/3DGI/tyler