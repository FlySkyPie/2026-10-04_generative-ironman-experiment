# Day 22｜Extensions 擴充機制 — 當標準不夠用時

## 前言

在過去 21 天，我們認識了 CityJSON 的核心規格——從幾何結構、城市物件、語意表面（Day 17）、到外觀材質（Day 19-20）、再到 Metadata（Day 21）。這些元素已經能夠描述絕大多數的 3D 城市模型場景。

但現實世界總是比標準複雜：

- 你需要為建築物加上**噪音模擬**的計算結果？
- 你想在 Building 上標記**碳足跡評估**資訊？
- 你的專案需要全新的城市物件類型（例如「充電樁」或「智慧路燈」）？

標準規格不可能涵蓋所有領域的需求。這就是 **Extensions（擴充機制）** 登場的時刻——它讓你在不破壞 CityJSON 核心相容性的前提下，自由擴充資料結構。

今天我們要來深入探討這個讓 CityJSON 保持「開放」的關鍵設計。

---

## 設計哲學 — 為什麼不要直接亂加屬性？

在 JSON 的世界裡，你可以隨時在任何物件上加入新屬性——這是 JSON 的「無模式（schema-less）」哲學[^cityjson-extensions]。但問題是：如果你只是隨手加了一個 `"color": "red"`，其他人寫的軟體該如何知道這個屬性的存在？它該是字串還是物件？是必填還是選填？

這就是 Extensions 要解決的核心問題：**提供一個正式的、可驗證的機制來定義擴充資料**。

Extensions 的設計目標是讓標準的 CityJSON 軟體（如 cjio、ninja.cityjson.org）**不需要修改就能處理擴充後的檔案**[^cityjson-extensions]。這與 CityGML 的 ADE（Application Domain Extension）形成對比——ADE 彈性更大，但通常需要為每個 ADE 撰寫專屬的解析程式。

### 四種擴充方式

Extensions 可以從 **四個維度** 擴充 CityJSON 核心模型[^cityjson-extensions]：

| 方式 | 說明 | 使用的 Extension 成員 |
|------|------|---------------------|
| **1. 根層級屬性** | 在檔案根層級加入新成員（如人口普查資料） | `extraRootProperties` |
| **2. 屬性擴充** | 在既有的 City Object 上加入新屬性 | `extraAttributes` |
| **3. 語意表面擴充** | 定義新的語意表面類型 | `extraSemanticSurfaces` |
| **4. 城市物件擴充** | 定義全新的 City Object 類型 | `extraCityObjects` |

---

## 宣告使用 Extension — `extensions` 根成員

在使用任何 Extension 之前，你需要在 CityJSON 檔案的根層級加入 `"extensions"` 成員[^cityjson-extensions]：

```json
{
  "type": "CityJSON",
  "version": "2.0",
  "extensions": {
    "noise": {
      "url": "https://cityjson.github.io/extensions/noise/2.0.0/noise.ext.json",
      "version": "2.0.0"
    },
    "solar-potential": {
      "url": "https://someurl.org/solar-potential.ext.json",
      "version": "0.8.1"
    }
  },
  "CityObjects": {},
  "vertices": []
}
```

每個 Extension 由一個**名稱**作為 key，值包含：

| 成員 | 說明 |
|------|------|
| `url` | Extension 檔案的公開 HTTP URL |
| `version` | 使用的 Extension 版本 |

> **注意**：Extension 檔案建議透過公開 URL 存取——這樣任何讀取檔案的人都可以下載 Extension 定義來驗證資料。

---

## Extension 檔案結構（`.ext.json`）

一個 Extension 檔案是一個合法的 JSON 物件，**必須**具備以下 9 個成員[^cityjson-extensions]：

| 成員 | 型態 | 說明 |
|------|------|------|
| `type` | 字串 | 固定為 `"CityJSONExtension"` |
| `name` | 字串 | Extension 的名稱 |
| `url` | 字串 | 此 Extension 檔案所在位置的 HTTP URL |
| `version` | 字串 | Extension 的版本 |
| `versionCityJSON` | 字串 | 相容的 CityJSON 版本（如 `"2.0"`） |
| `extraAttributes` | 物件 | 用 JSON Schema 定義的屬性擴充，或 `{}` |
| `extraCityObjects` | 物件 | 用 JSON Schema 定義的新城市物件，或 `{}` |
| `extraRootProperties` | 物件 | 用 JSON Schema 定義的根層級屬性，或 `{}` |
| `extraSemanticSurfaces` | 物件 | 用 JSON Schema 定義的新語意表面，或 `{}` |

一個**最小但完整**的 Extension 檔案如下[^cityjson-extensions]：

```json
{
  "type": "CityJSONExtension",
  "name": "noise",
  "description": "Extension to model the noise",
  "url": "https://cityjson.github.io/extensions/noise/2.0.0/noise.ext.json",
  "version": "2.0.0",
  "versionCityJSON": "2.0",
  "extraAttributes": {},
  "extraCityObjects": {},
  "extraRootProperties": {},
  "extraSemanticSurfaces": {}
}
```

---

## `+` 前綴慣例 — 一眼看出哪些是擴充

CityJSON 要求所有由 Extension 定義的元素名稱**必須以 `"+"` 開頭**[^cityjson-extensions]：

| 元素類型 | 範例 | 說明 |
|----------|------|------|
| 根屬性 | `"+census"` | Extension 定義的根層級新屬性 |
| 屬性 | `"+colour"`、`"+area-parcel"` | 在 City Object 的 `attributes` 中加入 |
| 語意表面 | `"+ThermalSurface"` | 新的語意表面類型 |
| 城市物件 | `"+NoiseBuilding"` | 新的 City Object 類型 |

為什麼要這樣設計？因為 `"+"` 讓**任何解析器在讀取檔案時，可以立刻知道哪些資料是 Core 標準的一部分、哪些由 Extension 定義**——不需要載入 Extension 檔案就能做出這個判斷。

---

## Case 1：根層級屬性（Extra Root Properties）

最簡單的擴充方式——在 CityJSON 檔案的根層級加入一個新的 top-level 成員[^cityjson-extensions]。

想像我們要為一個街區儲存人口普查資料。首先在 Extension 中定義：

```json
"extraRootProperties": {
  "+census": {
    "type": "object",
    "properties": {
      "percent_men": {
        "type": "number",
        "minimum": 0.0,
        "maximum": 100.0
      },
      "percent_women": {
        "type": "number",
        "minimum": 0.0,
        "maximum": 100.0
      }
    }
  }
}
```

然後在 CityJSON 檔案中使用：

```json
{
  "type": "CityJSON",
  "version": "2.0",
  "extensions": {
    "census": {
      "url": "https://someurl.org/census.ext.json",
      "version": "0.7.2"
    }
  },
  "CityObjects": { ... },
  "vertices": [ ... ],
  "+census": {
    "percent_men": 49.5,
    "percent_women": 51.5
  }
}
```

---

## Case 2：屬性擴充（Extra Attributes）

最常見的擴充方式——在既有的 City Object 上加入新屬性[^cityjson-extensions]。

### 簡單型別屬性

假設我們想要為建築物加上 RGBA 顏色值：

```json
// 在 CityJSON 檔案中的 Building 物件
{
  "type": "Building",
  "attributes": {
    "storeysAboveGround": 2,
    "+colour": {
      "rgba": [255, 255, 255, 1]
    }
  },
  "geometry": [ ... ]
}
```

在 Extension 中定義對應的 Schema：

```json
"extraAttributes": {
  "Building": {
    "+colour": {
      "type": "object",
      "properties": {
        "rgba": {
          "type": "array",
          "items": { "type": "number" },
          "minItems": 4,
          "maxItems": 4
        }
      },
      "required": [ "rgba" ],
      "additionalProperties": false
    }
  }
}
```

### 附單位的屬性

更實用的範例——為建築物加上「地塊面積」，並標記單位（UoM）：

```json
"extraAttributes": {
  "Building": {
    "+area-parcel": {
      "type": "object",
      "properties": {
        "value": { "type": "number" },
        "uom": { "type": "string", "enum": [ "m2", "feet2" ] }
      },
      "required": [ "value", "uom" ],
      "additionalProperties": false
    }
  }
}
```

使用時：

```json
{
  "type": "Building",
  "attributes": {
    "storeysAboveGround": 2,
    "+area-parcel": {
      "value": 437,
      "uom": "m2"
    }
  },
  "geometry": [ ... ]
}
```

### 多型別屬性支援

Extension 的 `extraAttributes` 可以同時為多種 City Object 定義屬性——例如同時為 `Building` 和 `BuildingPart` 定義相同的噪音屬性[^noise-extension]：

```json
"extraAttributes": {
  "Building": {
    "+noise-buildingLDenMin": { "$ref": "#/definitions/measure" },
    "+noise-buildingLDenMax": { "$ref": "#/definitions/measure" }
  },
  "BuildingPart": {
    "+noise-buildingLDenMin": { "$ref": "#/definitions/measure" },
    "+noise-buildingLDenMax": { "$ref": "#/definitions/measure" }
  }
}
```

這種做法在官方 Noise Extension 中被大量使用——它為 Building 和 BuildingPart 各定義了 **11 個**與噪音模擬相關的屬性。

---

## Case 3：語意表面擴充（Extra Semantic Surfaces）

除了 Core 標準定義的 11 種語意表面（Day 17），Extensions 也可以定義新的語意類型[^cityjson-extensions]：

```json
"extraSemanticSurfaces": {
  "+ThermalSurface": {
    "type": "object",
    "properties": {
      "type": { "enum": [ "+ThermalSurface" ] },
      "azimuth": { "type": "number" }
    },
    "required": [ "type", "azimuth" ],
    "additionalProperties": false
  }
}
```

這使得分析軟體可以標記建築物的特定表面為「熱輻射面」，並附帶方位角資訊——這些都是標準語意無法涵蓋的。

---

## Case 4：城市物件擴充（Extra City Objects）

當既有的 City Object 類型無法滿足需求時，你可以建立**全新的 City Object 類型**[^cityjson-extensions]。

### 建立 1st-Level 物件

假設我們需要一個 `+NoiseBuilding` 物件，它繼承 `_AbstractBuilding` 的行為但儲存噪音相關屬性：

```json
"extraCityObjects": {
  "+NoiseBuilding": {
    "allOf": [
      { "$ref": "cityobjects.json#/_AbstractBuilding" },
      {
        "properties": {
          "type": { "enum": [ "+NoiseBuilding" ] },
          "attributes": {
            "properties": {
              "buildingLDenMin": { "type": "number" }
            }
          }
        },
        "required": [ "type" ]
      }
    ]
  }
}
```

### 建立 2nd-Level 物件

對於需要有 parent 的次級物件（如 `+NoiseBuildingPart`），需要明確將 `"parents"` 設為必填：

```json
"extraCityObjects": {
  "+NoiseBuildingPart": {
    "allOf": [
      { "$ref": "cityobjects.json#/_AbstractBuilding" },
      {
        "properties": {
          "type": { "enum": [ "+NoiseBuildingPart" ] },
          "attributes": {
            "properties": {
              "buildingLDenMin": { "type": "number" }
            }
          }
        },
        "required": [ "type", "parents" ]
      }
    ]
  }
}
```

> **關於 JSON Schema 繼承**：JSON Schema 本身不支援繼承，所以擴充 City Object 的唯一方式是**複製父物件的 Schema 後再擴充它**，並給予新的名稱（如 `"+NoiseBuilding"`）[^cityjson-extensions]。

---

## 新增 City Objects 的 6 條規則

當你建立新的 City Object 時，必須遵守以下規則，以確保所有標準 CityJSON 軟體**不需要修改就能正確處理**[^cityjson-extensions]：

| # | 規則 | 說明 |
|---|------|------|
| 1 | **名稱以 `+` 開頭** | 例如 `"+NoiseBuilding"`，讓所有解析器立刻識別 |
| 2 | **必須有 `type` 成員** | 符合 CityJSON 的基本規範 |
| 3 | **不得直接附加子類型到既有物件** | 你不能在 `Building` 下加 `"+Balcony"`——應該建立新的 `"+FancyBuilding"` 來包含 `"+Balcony"` |
| 4 | **所有幾何資料必須在 `geometry` 中** | 不能藏在巢狀結構的其他地方 |
| 5 | **幾何邊界必須是 Core 的 8 種類型** | 同樣，外觀和模板也必須遵循核心規格 |
| 6 | **可重用 Core 的類型定義** | 例如 `"Solid"` 和語意表面都可以直接參考使用 |

規則 1 與規則 5 尤其重要——它們確保 Viewer 和分析工具不需要任何修改就能正確顯示和處理新物件。

---

## Extensions Registry — 9 個官方與社群擴充

CityJSON 官方在 GitHub 上維護了一個 **Extensions Registry**[^extensions-registry]，收錄了以下 9 個可用擴充：

| Extension | 版本 | 開發者 | 說明 |
|-----------|------|--------|------|
| **noise** | 2.0.0 | Hugo Ledoux | 噪音排放模擬 |
| **energy-space-heating** | 1.1.1 | Özge Tufan | 建築空間供暖需求計算 |
| **quality** | 1.0.1 | Grigory Ilizirov, Sagi Dalyot | 資料品質建模 |
| **dynamizer** | 2.0.0 | Imane Jeddoub | CityGML 3.0 Dynamizer 概念 |
| **lcc** | 0.3.0 | Stelios Vitalis | 線性單元複合體拓撲 |
| **perception** | 0.6.0 | Binyu Lei | 建築物視覺感知 |
| **val3dity** | 0.3.0 | Ravi Peters | 儲存 val3dity 驗證報告 |
| **multiroofs** | 0.1.0 | Hugo Ledoux | MultiRoofs EU 專案 |
| **shed** | 0.1.1 | Hugo Ledoux | 範例 Extension（學習用模板） |

所有 Extension 的 schema 檔案遵循統一的 URL 模式：
```
https://cityjson.github.io/extensions/{name}/{version}/{name}.ext.json
```

除了官方 Registry，你也可以自行託管 Extension 檔案，只要有公開 HTTP URL 即可。

### 圖形化建構工具

CityJSON 官方提供了一個 **Extension Builder** 線上工具[^extensions-builder]：

```
https://www.cityjson.org/extensions/builder/
```

它可以引導你透過 GUI 建立 `.ext.json` 檔案，特別適合初學者快速上手。

---

## 實際案例 — Noise Extension 架構

讓我們以 Noise Extension 的真實架構為例，看看一個完整的 Extension 如何運作。

Noise Extension 定義了[^noise-extension]：

| 擴充項目 | 數量 | 說明 |
|----------|------|------|
| `extraAttributes`（Building） | 11 個屬性 | 建築噪音屬性（LDen、LNight、反射修正等） |
| `extraAttributes`（BuildingPart） | 11 個屬性 | 同上，針對 BuildingPart |
| `extraCityObjects` | 3 個新物件 | `+NoiseRoadSegment`、`+NoiseRailwaySegment`、`+NoiseCityFurnitureSegment` |

其中一個 `+NoiseRoadSegment` 的屬性多達 **23 個**——包括不同時間段的車流量（`mDay`、`mEvening`、`mNight`）、車速（`speedDayPkw`、`speedDayLkw`）、道路坡度等。

```json
// CityJSON 檔案中使用 Noise Extension 的範例
{
  "type": "CityJSON",
  "version": "2.0",
  "extensions": {
    "noise": {
      "url": "https://cityjson.github.io/extensions/noise/2.0.0/noise.ext.json",
      "version": "2.0.0"
    }
  },
  "CityObjects": {
    "building-01": {
      "type": "Building",
      "attributes": {
        "storeysAboveGround": 5,
        "+noise-buildingLDenEq": { "value": 65.3, "uom": "dB" },
        "+noise-buildingLNightEq": { "value": 52.1, "uom": "dB" },
        "+noise-buildingReflection": "bothSides"
      },
      "geometry": [ ... ]
    }
  },
  "vertices": [ ... ]
}
```

這個案例展示了 Extension 如何讓一個**領域特定的複雜資料模型**（噪音模擬）與標準的 3D 城市模型無縫整合。

---

## 結語

今天我們了解了 CityJSON 的 Extension 擴充機制——從四種擴充方式（根屬性、屬性、語意表面、新物件）到 `+` 前綴慣例，再到 6 條規則和官方 Registry。

Extensions 是 CityJSON 的核心設計哲學之一：**在保持核心簡潔的同時，不限制領域特定的資料需求**。這與其競爭對手 CityGML 的 ADE 機制形成鮮明對比——ADE 更靈活但也更容易「破壞」標準軟體的相容性。

明天，我們將實際動手——**寫一個自己的 Extension**，把這些知識應用到一個具體的例子上。

---

## 參考資料

[^cityjson-extensions]: CityJSON Specification 2.0.2. (2024). §8 Extensions. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#extensions

[^noise-extension]: Hugo Ledoux. (2024). CityJSON Noise Extension (2.0.0). Retrieved 2026-09-27, from https://cityjson.github.io/extensions/noise/2.0.0/noise.ext.json

[^extensions-registry]: CityJSON. (2024). CityJSON Extensions Registry. Retrieved 2026-09-27, from https://github.com/cityjson/extensions

[^extensions-builder]: CityJSON. (2024). CityJSON Extension Builder. Retrieved 2026-09-27, from https://www.cityjson.org/extensions/builder/

[^extensions-page]: CityJSON. (2024). Extensions — cityjson.org. Retrieved 2026-09-27, from https://www.cityjson.org/extensions/