# Day 27｜從 CityGML 轉換到 CityJSON — 用 citygml-tools 打通資料管線

## 前言

如果你手上有一批 CityGML 檔案，但想要用 CityJSON 的生態系來處理，該怎麼辦？事實上，這是很多從業者最常面臨的場景——各國測繪機構、OpenStreetMap 社群和都市規劃單位累積了海量的 CityGML 資料，而 CityJSON 提供了一個更輕量、更開發者友善的途徑來分析這些資料[^cityjson-tutorial-conversion]。

為了解決這個需求，TU Delft 團隊開發了 **citygml-tools**——一套基於 Java 的命令列工具，可以實現 CityGML ↔ CityJSON 的雙向轉換[^citygml-tools-github]。今天我們就來深入學習這把關鍵鑰匙。

---

## citygml-tools 是什麼？

citygml-tools 是由 **citygml4j** 核心函式庫驅動的命令列工具，由 Claus Nagel 與 TU Delft 3D 地理資訊組開發維護[^citygml-tools-github]。它不是 Python 套件，而是以 Java 17 為基礎的獨立執行檔，支援：

| 功能 | 說明 |
|------|------|
| **CityGML → CityJSON** | 將 CityGML 1.0/2.0/3.0 轉為 CityJSON 1.0/1.1/2.0 |
| **CityJSON → CityGML** | 反向轉換，支援 LoD 對應策略 |
| **驗證** | 依 CityGML XML Schema 驗證原始檔 |
| **統計** | 分析 CityGML 內容的物件類型、數量與 LoD |
| **升級** | 將 CityGML 2.0/1.0 升級為 CityGML 3.0 |
| **合併** | 合併多個 CityGML 檔案（v2.4.0 新增） |
| **子集** | 依地理範圍或物件類型建立子集 |

截至 2026 年，最新穩定版本為 **v2.5.0**[^citygml-tools-releases]。

---

## 安裝方式

### 方式一：下載 ZIP 壓縮檔（建議）

citygml-tools 是 Java 應用程式，無法用 `pip install` 安裝。請前往 GitHub Releases 頁面下載最新版的 ZIP 壓縮檔[^citygml-tools-releases]：

```bash
# 1. 下載 v2.5.0 的 ZIP
# 手動下載或使用 wget
wget https://github.com/citygml4j/citygml-tools/releases/download/v2.5.0/citygml-tools-v2.5.0.zip

# 2. 解壓縮
unzip citygml-tools-v2.5.0.zip
cd citygml-tools-v2.5.0

# 3. 執行（需要 Java 17 以上）
java -version  # 確認版本
./citygml-tools --help
```

### 方式二：Docker

如果不想安裝 Java 環境，可以直接使用官方 Docker 映像[^citygml-tools-docker]：

```bash
docker pull citygml4j/citygml-tools:v2.5.0

docker run --rm -u 1000 \
  -v /path/to/your/data:/data \
  citygml4j/citygml-tools:latest \
  to-cityjson *.gml
```

### 方式三：從原始碼編譯

```bash
git clone https://github.com/citygml4j/citygml-tools.git
cd citygml-tools
./gradlew installDist
# 執行檔位於：citygml-tools/build/install/citygml-tools/
```

---

## 基本轉換流程

### 第一步：驗證原始 CityGML

轉換前建議先驗證來源檔案的合法性。citygml-tools 不具備容錯轉換能力——如果 CityGML 有結構性錯誤，轉換後的 CityJSON 可能缺少物件但看起來卻正常[^3d-geospatial-conversion]：

```bash
./citygml-tools validate /data/DenHaag_01.xml
```

### 第二步：查看統計資訊

了解來源檔案包含哪些內容，作為轉換後的比對基準：

```bash
./citygml-tools stats /data/DenHaag_01.xml
```

輸出範例：
```
Feature types:
  Building: 4,812 (LOD2)
  BuildingPart: 11,204
  BuildingInstallation: 892
  Road: 145
  SolitaryVegetationObject: 3,201
Appearances: yes (materials + textures)
Generic attributes: yes
CRS: EPSG:25832
```

### 第三步：執行轉換

最簡單的轉換只需一個指令[^cityjson-tutorial-conversion]：

```bash
./citygml-tools to-cityjson /data/DenHaag_01.xml
```

這會在相同目錄下產生 `DenHaag_01.json`。如果需要更多控制，可以使用以下選項：

| 選項 | 說明 | 範例 |
|------|------|------|
| `--vertex-precision=N` | 頂點精確度（CRS 單位的小數位數） | 公制 CRS 設 `3`（毫米級） |
| `--template-precision=N` | 幾何模板頂點精確度 | `--template-precision=3` |
| `--json-lines` | 輸出為 CityJSONSeq (JSON Lines) | 適合超大檔案 |
| `--epsg=CODE` | 手動注入 EPSG 碼 | `--epsg=25832` |
| `--output=DIR` | 指定輸出目錄 | `--output=./cityjson/` |
| `--replace-templates` | 將模板幾何展開為實際座標 | 方便後續處理 |

**完整轉換範例：**

```bash
./citygml-tools to-cityjson \
  --vertex-precision=3 \
  --epsg=25832 \
  --output=./cityjson/ \
  /data/DenHaag_01.xml
```

### 第四步：驗證轉換結果

轉換完成後，用 cjio 或 cjval 檢查 CityJSON 的正確性：

```bash
# 用 cjio 看統計
cjio /data/DenHaag_01.json info --long

# 用 cjval 驗證語法
cjval /data/DenHaag_01.json
```

---

## 什麼被保留下來，什麼會遺失？

### ✅ 保留下來的內容

核心的 CityGML 資料模型幾乎全部保留[^citygml-tools-readme]：

- **所有特徵類型**——Building、Bridge、Tunnel、Road、Vegetation、WaterBody、LandUse、CityFurniture 等
- **物件階層與語意**——parent/children 關係、語意表面
- **所有幾何類型**——Solid、MultiSurface、CompositeSolid、TIN 等
- **隱含幾何**（Implicit Geometry）→ Geometry Templates
- **外觀材質**——紋理與材質
- **地址資訊**（v2.5.0 後屬性名稱標準化為小寫）
- **通用屬性**（Generic attributes）
- **CRS 資訊**（若可解析為 EPSG 碼）
- **LoD 0–3**（含 TU Delft 改良版字串格式）
- **所有 CityGML 命名空間**（Core、Building、Bridge、Tunnel、Transportation、Vegetation、WaterBody、LandUse、CityFurniture、Generics、Relief、Appearance）

### ❌ 不會被轉換的內容

| 項目 | 原因 |
|------|------|
| **LoD4（建築內部）** | CityJSON 不支援室內幾何 |
| **單一檔案中多個 CRS** | CityJSON 僅支援單一 CRS |
| **非 EPSG 的 CRS 識別碼** | CityJSON 需要 OGC URI 格式的 EPSG |
| **gml:id on 點/邊/面** | CityJSON 沒有低階幾何 ID 概念 |
| **Terrain Intersection Curve (TIC)** | CityJSON 不支援此類別 |
| **GeoreferencedTexture / TexCoordGen** | CityJSON 使用 UV 座標紋理 |
| **Raster Relief** | 僅支援 TIN Relief |
| **Section / Intersection 特徵** | CityJSON 不支援交通區段類別（v2.3.1 起略過） |
| **ADE 擴充屬性（無對應 Extension）** | 無對應 CityJSON Extension 時會無聲遺失 |
| **XML 註解、元素順序** | JSON 沒有對應概念 |

### 一個重要的警告：頂點精確度

`--vertex-precision` 的單位是 **CRS 單位**，不是公尺[^3d-geospatial-conversion]。這點極度重要：

- **公制 CRS（如 EPSG:25832）**：`--vertex-precision=3` 表示毫米級精確度，非常安全
- **地理 CRS（如 EPSG:4326，單位是度）**：`--vertex-precision=3` 表示約 100 公尺的網格，會嚴重破壞幾何！

如果你的資料是地理座標系統，請至少設為 7 位小數：
```bash
./citygml-tools to-cityjson --vertex-precision=7 /data/geographic_city.gml
```

---

## 大量批次轉換

citygml-tools 原生支援萬用字元（glob pattern）批次處理[^citygml-tools-readme]：

```bash
# 轉換目錄中所有 GML 檔案
./citygml-tools to-cityjson --output=./output_dir /data/tiles/*.gml

# 指定多個檔案
./citygml-tools to-cityjson file1.gml file2.gml file3.gml

# 使用 Docker 批次處理
docker run --rm -u 1000 \
  -v /data:/data \
  citygml4j/citygml-tools:latest \
  to-cityjson /data/*.gml
```

### 批次處理的注意事項

1. **單執行緒**——轉換是 XML 解析繫結的單執行緒操作。建議對每個圖磚（tile）啟動獨立的 citygml-tools 行程來平行處理。
2. **記憶體管理**——整個模型會載入記憶體。對於大型檔案，需調整 JVM heap：
   ```bash
   export JAVA_OPTS="-Xmx16g"
   ./citygml-tools to-cityjson /data/huge_city.gml
   ```
3. **轉換時間**——每 100 MB 的 CityGML 約需 1–3 分鐘。建議先對小區域測試，確認設定正確後再執行完整批次。

### 合併多個 CityGML 檔案

v2.4.0 後支援合併功能[^citygml-tools-releases]：

```bash
./citygml-tools merge --output=merged.gml tile1.gml tile2.gml tile3.gml
```

---

## 反向轉換：CityJSON → CityGML

### 基本用法

CityJSON 同樣可以轉回 CityGML[^cityjson-tutorial-conversion]：

```bash
./citygml-tools from-cityjson /data/denhaag.city.json
```

### LoD 對應策略

CityJSON 使用小數版 LoD（如 `2.2`），而 CityGML 使用整數 LoD（如 `2`）。citygml-tools 提供三種對應模式[^citygml-tools-readme]：

```bash
# maximum（預設）：取最高的 CityJSON LoD
./citygml-tools from-cityjson --lod-mode=maximum file.city.json

# minimum：取最低的 CityJSON LoD
./citygml-tools from-cityjson --lod-mode=minimum file.city.json

# 明確對應：指定 CityGML LoD = CityJSON LoD
./citygml-tools from-cityjson --lod-mapping 2=2.2 file.city.json
```

### 其他選項

```bash
# 不要將未知 Extension 類型對應為 GenericCityObject
./citygml-tools from-cityjson --no-map-unknown-types file.city.json

# 覆蓋輸出 CityGML 的 srsName
./citygml-tools from-cityjson --crs-name="urn:adv:crs:ETRS89_UTM32*DE_DHHN2016_NH" file.city.json
```

### 雙向驗證（Round-tripping）

一個實務上極度推薦的做法是 **雙向驗證**[^3d-geospatial-conversion]：

```bash
# 1. CityGML → CityJSON
./citygml-tools to-cityjson original.gml

# 2. CityJSON → CityGML
./citygml-tools from-cityjson original.json

# 3. 比較原始與回圈的 CityGML 統計
./citygml-tools stats original.gml
./citygml-tools stats original_roundtrip.gml
```

如果統計結果相差不大，就代表轉換是可靠的。

---

## ADE 與 CityJSON Extension 的支援

### 問題核心：ADE 不會自動轉換

CityGML 的 Application Domain Extension（ADE）是擴充機制的最大痛點。

citygml-tools **無法通用地處理 ADE**[^citygml-tools-issue-41]——它需要一個實作該 ADE 的 Java JAR 模組。如果轉換時沒有載入對應的 Extension JAR，ADE 屬性會**無聲地遺失**（僅輸出警告）：

```
WARNING: Non-CityGML content is skipped unless a matching ADE extension has been loaded
```

### 載入 ADE 的方式

```bash
# 透過 CLI 選項指定擴充目錄
./citygml-tools --extensions=/path/to/extensions/dir to-cityjson file.gml

# 或將 JAR 放在 citygml-tools 目錄下的 extensions/ 資料夾
```

### CityJSON Extension 的對應策略

如果你的 CityGML 含有 ADE 屬性，有幾種處理方式：

1. **尋找現有 Extension JAR**——確認該 ADE 是否已有 citygml4j 的實作模組
2. **自行定義 CityJSON Extension**——撰寫 `.ext.json` 定義檔（詳見 Day 22、Day 23）
3. **接受遺失**——如果 ADE 資料對你的應用不重要，可以忽略
4. **聯絡開發者**——在 GitHub 上開 issue 請求支援

---

## 實戰：完整的轉換管線

以下是一個從 CityGML 到乾淨 CityJSON 的完整流程：

```bash
#!/bin/bash
# 完整轉換腳本

INPUT_DIR="/data/citygml_tiles"
OUTPUT_DIR="/data/cityjson_output"
mkdir -p "$OUTPUT_DIR"

# 1. 先查看其中一個檔案的統計
./citygml-tools stats "$INPUT_DIR/tile_001.gml"

# 2. 測試一個檔案
./citygml-tools to-cityjson \
  --vertex-precision=3 \
  --output="$OUTPUT_DIR" \
  "$INPUT_DIR/tile_001.gml"

# 3. 用 cjio 驗證
cjio "$OUTPUT_DIR/tile_001.json" info --long

# 4. 如果測試通過，批次處理全部
for f in "$INPUT_DIR"/*.gml; do
  echo "Converting: $f"
  ./citygml-tools to-cityjson \
    --vertex-precision=3 \
    --output="$OUTPUT_DIR" \
    "$f"
done

# 5. 最終統計
echo "=== Conversion Summary ==="
echo "Input: $(find "$INPUT_DIR" -name '*.gml' | wc -l) files"
echo "Output:"
find "$OUTPUT_DIR" -name '*.json' -exec ls -lh {} \;
```

---

## 小結

今天我們學習了如何用 citygml-tools 打通 CityGML 與 CityJSON 之間的資料管線。以下是重點回顧：

- **citygml-tools** 是 Java 命令列工具，不是 Python 套件，需 Java 17 以上
- **核心轉換**只需一條指令：`./citygml-tools to-cityjson file.gml`
- **頂點精確度**的單位是 CRS 單位而非公尺——地理座標系統需設為 7 位以上
- **批次轉換**支援萬用字元，建議為每個圖磚啟動獨立行程以平行處理
- **ADE 屬性**在無對應 Extension JAR 時會無聲遺失，需特別留意
- **雙向驗證**（round-tripping）是確認轉換品質的最佳做法
- 轉換後的 CityJSON 檔案大小約為 CityGML 的 **十分之一**

明天，我們將踏上全球城市資料集的巡禮，看看各國開放的 CityJSON 資料集有哪些亮點！

---

## 參考資料

[^cityjson-tutorial-conversion]: CityJSON. (n.d.). Converting to/from CityGML files. Retrieved 2026-09-27, from https://www.cityjson.org/tutorials/conversion/

[^citygml-tools-github]: citygml4j. (2026). citygml-tools: Command-line tools for CityGML and CityJSON. GitHub. Retrieved 2026-09-27, from https://github.com/citygml4j/citygml-tools

[^citygml-tools-releases]: citygml4j. (2026). citygml-tools Releases. GitHub. Retrieved 2026-09-27, from https://github.com/citygml4j/citygml-tools/releases

[^citygml-tools-docker]: citygml4j. (2026). citygml-tools on Docker Hub. Retrieved 2026-09-27, from https://hub.docker.com/r/citygml4j/citygml-tools

[^citygml-tools-readme]: citygml4j. (2026). citygml-tools README. GitHub. Retrieved 2026-09-27, from https://github.com/citygml4j/citygml-tools#readme

[^3d-geospatial-conversion]: 3D Geospatial. (n.d.). Converting CityGML to CityJSON with citygml-tools. Retrieved 2026-09-27, from https://www.3d-geospatial.com/3d-geospatial-fundamentals-for-digital-twins/citygml-and-cityjson-processing/converting-citygml-to-cityjson-with-citygml-tools/

[^citygml-tools-issue-41]: citygml4j. (2022). How to load EnergyADE extensions [GitHub Issue #41]. Retrieved 2026-09-27, from https://github.com/citygml4j/citygml-tools/issues/41