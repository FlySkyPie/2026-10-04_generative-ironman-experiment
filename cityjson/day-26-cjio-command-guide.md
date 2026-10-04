# Day 26｜cjio 指令一把罩 — CityJSON 的瑞士刀

## 前言

過去幾天我們學到了很多處理大型 CityJSON 資料集的策略——地理分割、串流處理、CityJSONSeq。但這些操作的背後，幾乎都離不開同一個工具：**cjio**。

cjio（CityJSON/io）是 TU Delft 3D 地理資訊組開發的 Python 命令列工具，是 CityJSON 生態系中最核心、最通用的瑞士刀[^cjio-github]。它不只是一個「轉換器」——它可以用來檢視、分析、篩選、合併、轉換、驗證、清理 CityJSON 資料，而且所有的操作都可以**串聯成管線（pipeline）**，一次解析、多次操作、一次寫出。

今天我們從安裝開始，完整認識 cjio 的所有核心功能，最後再寫一個自動化腳本把所學串起來。

---

## 安裝

### pip 安裝

透過 PyPI 安裝是最直接的方式[^cjio-pypi]：

```bash
pip install cjio
```

這會安裝 cjio 本身以及基礎依賴。部分進階功能（匯出、三角化、座標轉換、驗證）需要額外的套件，可以用 extras 一次安裝：

```bash
pip install 'cjio[export,reproject,validate]'
```

這些 extras 分別會安裝：

| Extras | 啟用功能 | 依賴套件 |
|--------|----------|----------|
| `export` | OBJ/STL/GLB/B3DM 匯出 | `triangle`（三角化） |
| `reproject` | 座標系統轉換 | `pyproj` |
| `validate` | 驗證 | `cjvalpy` |

### Docker

如果你不想污染本機的 Python 環境，也可以用官方 Docker 映像檔[^cjio-docker]：

```bash
docker run --rm -v $(pwd):/data tudelft3d/cjio:latest cjio --help
```

### 確認安裝

```bash
cjio --version
```

輸出會包含 cjio 版本號與支援的 CityJSON 版本，例如：

```
cjio 0.10.1 (CityJSON v2.0)
```

---

## 核心觀念：管線（Pipeline）設計

在開始逐一介紹指令之前，必須先理解 cjio 最重要的設計哲學——**每個操作都是一個管線步驟**[^cjio-pipeline]。

```bash
cjio input.city.json [操作1] [操作2] ... [操作N] save output.city.json
```

解析器只會**讀取檔案一次**，然後依序執行每個操作，最後才寫出。這樣的設計帶來了幾個好處：

1. **節省 I/O**——不需要在中間步驟重複讀寫硬碟。
2. **減少記憶體**——整條管線共享同一個 in-memory 模型。
3. **可組合性**——每個操作都是獨立的 building block。

```bash
# 一個完整的管線範例
cjio old.city.json \
  upgrade \
  lod_filter 2.2 \
  subset --bbox 120000 2500000 120500 2500500 \
  vertices_clean \
  save cleaned.city.json
```

---

## 資訊查詢類

### `info` — 一眼看出資料集的摘要

`info` 是最常用的指令之一，它會輸出資料集的摘要資訊[^cjio-info]：

```bash
cjio myfile.city.json info
```

輸出範例：

```
CityJSON version: 2.0
EPSG: 7415
Bbox: [120000.0, 2500000.0, 0.0, 121000.0, 2501000.0, 50.0]
+ CityObjects:
  Building (25)
    - BuildingPart (12)
    - BuildingInstallation (8)
  Road (5)
  SolitaryVegetationObject (30)
+ materials: True
+ textures: False
```

加上 `--long` 可以看到更詳細的資訊[^cjio-info-long]：

```bash
cjio myfile.city.json info --long
```

額外輸出的資訊包括：

- **vertices_total**——頂點總數
- **transform/scale 與 translate**——壓縮參數
- **geom primitives**——使用的幾何類型（如 `["Solid", "MultiSurface"]`）
- **LoD**——涵蓋的細節層級（如 `["1.3", "2.2"]`）
- **semantics surfaces**——語意表面類型（如 `["RoofSurface", "WallSurface", "GroundSurface"]`）
- **attributes**——所有 City Object 的屬性清單（如 `["roofType", "measuredHeight"]`）

### `metadata_get` — 讀取 metadata

如果你只想知道 metadata 資訊：

```bash
cjio myfile.city.json metadata_get
```

---

## 資料篩選類

### `subset` — 多種方式創建子集

`subset` 是 cjio 中最靈活的指令之一，支援五種篩選方式，還可以組合使用[^cjio-subset]：

```bash
# 1️⃣ 依地理範圍（bounding box）
cjio myfile.city.json subset --bbox 104607 490148 104703 490257 save out.city.json

# 2️⃣ 依半徑（3D 球體）
cjio myfile.city.json subset --radius 500 121000 2500500 50 save out.city.json

# 3️⃣ 依物件型別
cjio myfile.city.json subset --cotype Building --cotype BuildingPart save out.city.json

# 4️⃣ 依物件 ID
cjio myfile.city.json subset --id building_001 --id building_002 save out.city.json

# 5️⃣ 隨機取樣
cjio myfile.city.json subset --random 100 save sample.city.json
```

**組合篩選**——多個條件可以同時使用（`--random` 除外，其他條件會共同作用）：

```bash
# 在某個範圍內，只保留 Building
cjio myfile.city.json subset \
  --bbox 120000 2500000 120500 2500500 \
  --cotype Building \
  save buildings_in_area.city.json
```

**反向篩選**——加上 `--exclude` 會把選中的物件**刪除**，而不是保留：

```bash
# 刪除所有 BuildingPart
cjio myfile.city.json subset --cotype BuildingPart --exclude save no_parts.city.json
```

### `lod_filter` — 依細節層級篩選

只保留特定 LoD 的幾何資料：

```bash
cjio myfile.city.json lod_filter 2.2 save lod2_only.city.json
```

所有不包含該 LoD 的 City Object 會留下但 geometry 變為空陣列。

---

## 資料修改類

### `vertices_clean` — 清理頂點

移除**重複頂點**（duplicate vertices）和**孤立頂點**（orphan vertices——未被任何幾何參照的頂點）[^cjio-vertices]：

```bash
cjio myfile.city.json vertices_clean save cleaned.city.json
```

這在經過多次 `subset`、`merge` 後特別有用——那些被刪除的 City Object 留下的孤立頂點會白白佔用檔案空間。

### `attribute_remove` / `attribute_rename` — 屬性管理

```bash
# 移除某個屬性
cjio myfile.city.json attribute_remove roofType save out.city.json

# 重新命名屬性
cjio myfile.city.json attribute_rename oldName newName save out.city.json
```

### `materials_remove` / `textures_remove` — 移除外觀資料

如果你不需要材質或紋理資訊，可以移除它們來縮小檔案：

```bash
cjio myfile.city.json materials_remove save out.city.json
cjio myfile.city.json textures_remove save out.city.json
```

### `triangulate` — 三角化所有幾何

將所有多邊形面（polygons）轉換為三角形（triangles）[^cjio-triangulate]：

```bash
cjio myfile.city.json triangulate save triangulated.city.json
```

如果遇到自相交或不規則多邊形，可以加上 `--sloppy` 使用 mapbox-earcut 演算法（更寬容，但可能產生較不規則的三角形）：

```bash
cjio myfile.city.json triangulate --sloppy save triangulated.city.json
```

---

## 合併與升級

### `merge` — 合併多個 CityJSON 檔案

將一個或多個 CityJSON 檔案合併到目前的模型中[^cjio-merge]：

```bash
cjio base.city.json merge '/data/tiles/*.city.json' save merged.city.json
```

合併時 cjio 會自動處理：

1. **座標解壓縮**——每個 tile 可能使用不同的 `transform`，cjio 會先將所有頂點還原為真實世界座標再合併。
2. **頂點陣列拼接**——多個檔案的頂點陣列會串接在一起。
3. **索引偏移**——幾何 boundary 陣列中的頂點索引會自動加上適當的偏移量。
4. **Appearance 與 Template 索引**——材質、紋理、幾何模板的索引也會一併調整。

### `upgrade` — 升級 CityJSON 版本

將舊版的 CityJSON 檔案升級到最新版本（目前為 v2.0）[^cjio-upgrade]：

```bash
cjio old_v1_1.city.json upgrade save upgraded.city.json
```

cjio 支援逐步升級（如 v1.0 → v1.1 → v2.0），並可指定升級後的座標精度：

```bash
cjio old.city.json upgrade --digit 2 save upgraded.city.json
```

---

## 座標系統操作

### `crs_assign` — 指派座標參考系統

如果檔案缺少 CRS 資訊，可以用這個指令**指派**一個 EPSG 碼（注意：這個指令**不會轉換**座標值，只會設定 metadata 中的 CRS）[^cjio-crs]：

```bash
cjio myfile.city.json crs_assign 7415 save out.city.json
```

### `crs_reproject` — 座標系統轉換

真正的座標轉換——將所有頂點從目前的 CRS 轉換到目標 EPSG[^cjio-crs]：

```bash
cjio myfile.city.json crs_reproject 4979 save out.city.json
```

這個指令需要安裝 `pyproj`（`pip install 'cjio[reproject]'`）且原始檔案必須已經有定義 CRS 的 EPSG。轉換時也可以指定精度：

```bash
cjio myfile.city.json crs_reproject --digit 7 4979 save out.city.json
```

### `crs_translate` — 座標平移

將所有頂點平移某個向量——預設會將最小座標移至原點，讓座標值變小：

```bash
cjio myfile.city.json crs_translate save out.city.json
```

也可以指定自訂平移量：

```bash
cjio myfile.city.json crs_translate --minxyz -100 -25 -1 save out.city.json
```

注意：平移後 CRS/EPSG 會被設為 `None`，因為座標值已經不再是原 CRS 下的真實座標。

---

## 格式匯出

### `export` — 轉換為其他格式

cjio 支援將 CityJSON 匯出為五種格式[^cjio-export]：

```bash
# OBJ（Wavefront，支援紋理）
cjio myfile.city.json export obj myfile.obj

# STL（僅幾何，無紋理）
cjio myfile.city.json export stl myfile.stl

# GLB（Binary glTF 2.0）
cjio myfile.city.json export glb myfile.glb

# B3DM（Batched 3D Model for 3D Tiles）
cjio myfile.city.json export b3dm myfile.b3dm

# JSONL（CityJSONSeq，串流格式）
cjio myfile.city.json export jsonl myfile.city.jsonl
```

| 格式 | 紋理支援 | 適合場景 |
|------|----------|----------|
| **OBJ** | ✅（含 .mtl） | 3D 編輯軟體（Blender、MeshLab） |
| **STL** | ❌ | 3D 列印（僅幾何） |
| **GLB** | ❌（僅基本 PBR） | Web 瀏覽器、CesiumJS |
| **B3DM** | ❌ | 3D Tiles 管線 |
| **JSONL** | ✅（CityJSONSeq） | 串流處理、大規模分析 |

匯出時也可以使用 `--sloppy` 三角化：

```bash
cjio myfile.city.json export --sloppy obj myfile.obj
```

### `save` — 儲存為 CityJSON

```bash
# 預設儲存（compact JSON）
cjio myfile.city.json save output.city.json

# 美化輸出（indented JSON）
cjio myfile.city.json save --indent output.city.json

# 複製紋理到新目錄
cjio myfile.city.json save --textures /new/texture/dir output.city.json
```

---

## 紋理管理

### `textures_locate` 與 `textures_update`

處理紋理檔案路徑：

```bash
# 查看目前所有紋理檔案的路徑
cjio input.city.json textures_locate

# 更新紋理目錄路徑
cjio input.city.json textures_update /new/textures/location save out.city.json

# 使用相對路徑
cjio input.city.json textures_update ./textures --relative save out.city.json
```

---

## 實戰一：完整的資料清理管線

假設你剛從某個來源下載了一批 CityJSON v1.1 的資料，想要把它們清理成乾淨的 v2.0 格式，只保留建築物和 LoD 2.2 的幾何，然後輸出統計資訊。用 cjio 一條管線就夠了：

```bash
cjio raw_data.city.json \
  upgrade \
  lod_filter 2.2 \
  subset --cotype Building --cotype BuildingPart \
  vertices_clean \
  save --indent clean_buildings.city.json
```

如果同時想要知道輸出結果的摘要：

```bash
cjio clean_buildings.city.json info --long
```

---

## 實戰二：自動化 Python 腳本

cjio 不只是 CLI 工具，它也提供了完整的 Python API。以下是一個實戰腳本：讀取目錄中所有的 CityJSON 檔案，為每個檔案產生 GeoJSON 邊界框索引[^cjio-api-example]：

```python
from cjio import cityjson
import glob
import json
import os
from shapely.geometry import box, mapping


def path_leaf(path: str) -> str:
    """從完整路徑中取出檔名。"""
    head, tail = os.path.split(path)
    return tail or os.path.basename(head)


# 掃描目錄中所有 .city.json 檔案
files = glob.glob("./*.city.json")

features = []
epsg = None

for f in files:
    with open(f, "r") as cj_file:
        cm = cityjson.reader(file=cj_file)

    theinfo = json.loads(cm.get_info())
    bbox = theinfo["bbox"]

    # 建立 2D bounding box 幾何
    polygon = box(bbox[0], bbox[1], bbox[3], bbox[4])
    features.append({
        "properties": {"name": path_leaf(f)},
        "geometry": mapping(polygon),
    })

    if epsg is None and "epsg" in theinfo:
        epsg = theinfo["epsg"]

# 寫出 GeoJSON
geojson = {
    "type": "FeatureCollection",
    "features": features,
    "crs": {
        "type": "name",
        "properties": {
            "name": f"EPSG:{epsg}"
        }
    }
}

with open("grid.geojson", "w") as out:
    json.dump(geojson, out, indent=2)

print(f"✅ 已產生 grid.geojson，包含 {len(features)} 個 tiles")
```

這段腳本可以用 Docker 執行：

```bash
docker run --rm -v /path/to/data:/data tudelft3d/cjio:latest python gridder.py
```

---

## 實戰三：批次轉換與統計

假設你有一個包含 100 個 CityJSON tiles 的目錄，你想要批次分析它們的內容：

```bash
# 批次統計所有 tiles 的建築物數量
for f in /data/tiles/*.city.json; do
  echo "=== $(basename $f) ==="
  cjio "$f" subset --cotype Building save /dev/null 2>&1 | grep "Building"
done
```

或是用 cjio 的串流能力，將所有 tiles 合併後輸出統計：

```bash
cjio /data/tiles/tile_001.city.json \
  merge '/data/tiles/tile_*.city.json' \
  info \
  save /dev/null
```

> 注意：`save /dev/null` 在這裡只是為了完成管線（cjio 的管線最後一定要有 `save` 或 `export` 等終端操作），真正的輸出來自 `info`。

---

## cjio 常用指令速查表

| 類別 | 指令 | 用途 |
|------|------|------|
| **查詢** | `info` | 輸出資料集摘要 |
| | `info --long` | 詳細資訊（頂點數、幾何類型、LoD） |
| | `metadata_get` | 顯示 metadata |
| **篩選** | `subset --bbox` | 依地理範圍篩選 |
| | `subset --radius` | 依半徑篩選 |
| | `subset --cotype` | 依物件型別篩選 |
| | `subset --id` | 依 ID 篩選 |
| | `subset --random` | 隨機取樣 |
| | `lod_filter` | 依 LoD 篩選 |
| **修改** | `vertices_clean` | 移除重複／孤立頂點 |
| | `attribute_remove/rename` | 屬性管理 |
| | `materials_remove` | 移除材質 |
| | `textures_remove` | 移除紋理 |
| | `triangulate` | 三角化 |
| **合併** | `merge` | 合併多個檔案 |
| **升級** | `upgrade` | 升級 CityJSON 版本 |
| **座標** | `crs_assign` | 指派 CRS |
| | `crs_reproject` | 轉換 CRS |
| | `crs_translate` | 平移座標 |
| **匯出** | `export obj/stl/glb/b3dm/jsonl` | 匯出為其他格式 |
| | `save` | 儲存為 CityJSON |
| **紋理** | `textures_locate` | 查詢紋理路徑 |
| | `textures_update` | 更新紋理路徑 |

---

## 結語

今天我們完整認識了 cjio——CityJSON 生態系中最重要的工具。從資訊查詢、資料篩選、格式轉換、座標操作，到自動化腳本，cjio 幾乎涵蓋了所有你可能需要的日常操作。

cjio 的設計哲學——**管線化、可組合、一次解析多次操作**——讓它不僅是一個命令列工具，更是一個強大的資料處理框架。不論你是要清理資料、批次轉換、還是建立自動化管線，cjio 都能勝任。

明天我們會探討另一個重要的實戰主題：**如何將 CityGML 資料轉換為 CityJSON**，以及轉換過程中需要注意的各種細節。

---

## 參考資料

[^cjio-github]: TU Delft 3D Geoinformation. (n.d.). cjio — CityJSON/io. GitHub. Retrieved 2026-09-27, from https://github.com/cityjson/cjio

[^cjio-pypi]: cjio. (n.d.). cjio on PyPI. Retrieved 2026-09-27, from https://pypi.org/project/cjio/

[^cjio-docker]: cjio. (n.d.). Docker images for cjio. Docker Hub. Retrieved 2026-09-27, from https://hub.docker.com/r/tudelft3d/cjio

[^cjio-pipeline]: cjio. (n.d.). cjio documentation — Pipelines of operators. Retrieved 2026-09-27, from https://cjio.readthedocs.io/en/stable/includeme.html#pipelines-of-operators

[^cjio-info]: cjio. (n.d.). cjio CLI reference — info. Retrieved 2026-09-27, from https://cjio.readthedocs.io/en/stable/

[^cjio-info-long]: DeepWiki. (n.d.). cjio — Data management commands. Retrieved 2026-09-27, from https://deepwiki.com/cityjson/cjio/3.2-data-management-commands

[^cjio-subset]: cjio. (n.d.). cjio CLI reference — subset. Retrieved 2026-09-27, from https://cjio.readthedocs.io/en/stable/

[^cjio-vertices]: cjio. (n.d.). cjio CLI reference — vertices_clean. Retrieved 2026-09-27, from https://cjio.readthedocs.io/en/stable/

[^cjio-triangulate]: cjio. (n.d.). cjio CLI reference — triangulate. Retrieved 2026-09-27, from https://cjio.readthedocs.io/en/stable/

[^cjio-merge]: cjio. (n.d.). cjio CLI reference — merge. Retrieved 2026-09-27, from https://cjio.readthedocs.io/en/stable/

[^cjio-upgrade]: cjio. (n.d.). cjio CLI reference — upgrade. Retrieved 2026-09-27, from https://cjio.readthedocs.io/en/stable/

[^cjio-crs]: cjio. (n.d.). cjio CLI reference — CRS commands. Retrieved 2026-09-27, from https://cjio.readthedocs.io/en/stable/

[^cjio-export]: DeepWiki. (n.d.). cjio — Export and conversion commands. Retrieved 2026-09-27, from https://deepwiki.com/cityjson/cjio/3.3-export-and-conversion-commands

[^cjio-api-example]: cjio. (n.d.). cjio documentation — Example: gridder.py. Retrieved 2026-09-27, from https://cityjson.github.io/cjio/includeme.html