# Day 23｜寫一個自己的 Extension — 碳足跡評估實戰

## 前言

昨天我們深入了解了 CityJSON Extension 的四種擴充方式與完整結構。你可能已經開始想像：「如果我要為建築物加上碳足跡評估的資訊，該怎麼做？」

今天我們就來**實際動手寫一個完整的 Extension**。我們會從情境設定出發，到撰寫 `.ext.json` 檔案、在 CityJSON 資料中使用、最後用 `cjval` 驗證——完整走過一遍 Extension 開發流程。

這個實戰演練會使用一個對應到 Day 22 情境的主題：**為 Building 加入「碳足跡評估」**。

---

## 情境設定

### 問題

你是一名建築永續顧問，正在協助市政府評估市中心 50 棟建築物的碳足跡。你需要為每棟建築記錄以下資訊[^epd-denmark]：

| 指標 | 說明 | 單位 |
|------|------|------|
| **營運碳排 (Operational Carbon)** | 建築物日常使用的能源碳排放 | kg CO₂e / m² / 年 |
| **蘊含碳排 (Embodied Carbon)** | 建材生產、運輸、興建、拆除的碳排放 | kg CO₂e / m² |
| **主要結構材料** | 建築主體使用的結構材料 | 列舉值（concrete / steel / timber / masonry） |
| **能源等級** | 建築能源效率等級 | A+ ~ G |
| **生命週期階段明細** | A1-A3 到 C3-C4 各階段的碳排放細項 | kg CO₂e |

### 為什麼不用直接加屬性？

當然，你可以直接在 Building 的 `attributes` 中加這些欄位——但在真實專案中，你需要：

- **確保資料一致性**——所有資料供應商都使用相同的欄位名稱與單位
- **可驗證**——其他人可以確認你的資料格式是否正確
- **可發現**——其他人知道你使用了哪些擴充屬性

這就是 Extension 的價值所在。

---

## Step 1：定義資料需求

首先，我們先決定要擴充什麼。這是一個單純的**屬性擴充**（Case 2）——在既有的 `Building` City Object 上加入新屬性。

我們不需要新的 City Object 類型，也不需要新的語意表面或根層級屬性。

在 Extension 中，對應的成員就是 `extraAttributes`。

---

## Step 2：撰寫 `.ext.json` 檔案

現在我們開始建立 `carbon-footprint.ext.json`。

### 基本結構

每個 Extension 檔案都必須包含 9 個必要成員[^cityjson-specs]：

```json
{
  "type": "CityJSONExtension",
  "name": "Carbon Footprint",
  "description": "Extension to model the carbon footprint assessment of buildings",
  "url": "https://example.com/extensions/carbon-footprint/1.0.0/carbon-footprint.ext.json",
  "version": "1.0.0",
  "versionCityJSON": "2.0",
  "extraRootProperties": {},
  "extraSemanticSurfaces": {},
  "extraCityObjects": {},
  "extraAttributes": {}
}
```

暫時先把不需要的擴充成員設為空物件 `{}`。

### 定義屬性 Schema

現在來填寫 `extraAttributes` 的核心部分。JSON Schema 是定義每個屬性型態與約束的語法[^json-schema]：

```json
{
  "type": "CityJSONExtension",
  "name": "Carbon Footprint",
  "description": "Extension to model the carbon footprint assessment of buildings",
  "url": "https://example.com/extensions/carbon-footprint/1.0.0/carbon-footprint.ext.json",
  "version": "1.0.0",
  "versionCityJSON": "2.0",
  "extraRootProperties": {},
  "extraSemanticSurfaces": {},
  "extraCityObjects": {},
  "extraAttributes": {
    "Building": {
      "+co2-operational": {
        "type": "number",
        "description": "Operational carbon emissions (kg CO2e/m2/year)",
        "minimum": 0.0
      },
      "+co2-embodied": {
        "type": "number",
        "description": "Embodied carbon emissions (kg CO2e/m2)",
        "minimum": 0.0
      },
      "+co2-structure-material": {
        "type": "string",
        "description": "Main structural material",
        "enum": [ "concrete", "steel", "timber", "masonry", "mixed" ]
      },
      "+co2-energy-rating": {
        "type": "string",
        "description": "Energy efficiency rating",
        "enum": [ "A+", "A", "B", "C", "D", "E", "F", "G" ]
      },
      "+co2-lifecycle-phases": {
        "type": "array",
        "description": "Carbon emissions per lifecycle phase (kg CO2e)",
        "items": {
          "type": "object",
          "properties": {
            "phase": {
              "type": "string",
              "enum": [ "A1-A3", "A4-A5", "B4-B5", "B6", "C3-C4" ]
            },
            "description": { "type": "string" },
            "value": { "type": "number", "minimum": 0.0 }
          },
          "required": [ "phase", "value" ],
          "additionalProperties": false
        },
        "minItems": 1
      }
    }
  }
}
```

讓我們逐一看看這些 Schema 關鍵字的作用：

| JSON Schema 關鍵字 | 效果 | 在我們的 Extension 中 |
|-------------------|------|---------------------|
| `"type": "number"` | 限制必須為數值 | `+co2-operational`、`+co2-embodied` |
| `"minimum": 0.0` | 限制最小值 | 碳排放不可能為負數 |
| `"enum": [...]` | 限制只能從指定的值中選取 | 結構材料、能源等級、生命週期階段 |
| `"required": [...]` | 標記必填欄位 | `phase` 與 `value` 必須同時提供 |
| `"additionalProperties": false` | 禁止額外未定義屬性 | 防止打錯欄位名稱 |
| `"minItems": 1` | 陣列至少要有 1 個元素 | 至少需有一個生命週期階段 |

### 完整的 Extension 檔案

將以上內容組合起來，就是一個**完整可用的 `.ext.json` 檔案**[^shed-extension]：

```json
{
  "type": "CityJSONExtension",
  "name": "Carbon Footprint",
  "description": "Extension to model the carbon footprint assessment of buildings",
  "url": "https://example.com/extensions/carbon-footprint/1.0.0/carbon-footprint.ext.json",
  "version": "1.0.0",
  "versionCityJSON": "2.0",
  "extraRootProperties": {},
  "extraSemanticSurfaces": {},
  "extraCityObjects": {},
  "extraAttributes": {
    "Building": {
      "+co2-operational": {
        "type": "number",
        "description": "Operational carbon emissions (kg CO2e/m2/year)",
        "minimum": 0.0
      },
      "+co2-embodied": {
        "type": "number",
        "description": "Embodied carbon emissions (kg CO2e/m2)",
        "minimum": 0.0
      },
      "+co2-structure-material": {
        "type": "string",
        "description": "Main structural material",
        "enum": [ "concrete", "steel", "timber", "masonry", "mixed" ]
      },
      "+co2-energy-rating": {
        "type": "string",
        "description": "Energy efficiency rating",
        "enum": [ "A+", "A", "B", "C", "D", "E", "F", "G" ]
      },
      "+co2-lifecycle-phases": {
        "type": "array",
        "description": "Carbon emissions per lifecycle phase (kg CO2e)",
        "items": {
          "type": "object",
          "properties": {
            "phase": {
              "type": "string",
              "enum": [ "A1-A3", "A4-A5", "B4-B5", "B6", "C3-C4" ]
            },
            "description": { "type": "string" },
            "value": { "type": "number", "minimum": 0.0 }
          },
          "required": [ "phase", "value" ],
          "additionalProperties": false
        },
        "minItems": 1
      }
    }
  }
}
```

---

## Step 3：在 CityJSON 檔案中使用 Extension

有了 Extension 檔案，我們就可以在 CityJSON 資料中使用了。

### 第一步：宣告使用 Extension

在 CityJSON 的根層級加入 `extensions` 成員，指向我們的 Extension：

```json
{
  "type": "CityJSON",
  "version": "2.0",
  "extensions": {
    "carbon-footprint": {
      "url": "https://example.com/extensions/carbon-footprint/1.0.0/carbon-footprint.ext.json",
      "version": "1.0.0"
    }
  },
  "CityObjects": { },
  "vertices": []
}
```

### 第二步：為 Building 填入碳足跡資料

現在我們可以在 Building 物件的 `attributes` 中，使用 `+` 前綴的擴充屬性：

```json
{
  "type": "CityJSON",
  "version": "2.0",
  "extensions": {
    "carbon-footprint": {
      "url": "https://example.com/extensions/carbon-footprint/1.0.0/carbon-footprint.ext.json",
      "version": "1.0.0"
    }
  },
  "CityObjects": {
    "building-gov-office": {
      "type": "Building",
      "attributes": {
        "storeysAboveGround": 8,
        "storeysBelowGround": 2,
        "measuredHeight": 34.5,
        "+co2-operational": 28.3,
        "+co2-embodied": 452.0,
        "+co2-structure-material": "concrete",
        "+co2-energy-rating": "B",
        "+co2-lifecycle-phases": [
          { "phase": "A1-A3", "description": "材料生產", "value": 285.0 },
          { "phase": "A4-A5", "description": "運輸與施工", "value": 52.0 },
          { "phase": "B4-B5", "description": "修繕與更換", "value": 38.0 },
          { "phase": "B6", "description": "營運能耗", "value": 28.3 },
          { "phase": "C3-C4", "description": "廢棄處理", "value": 77.0 }
        ]
      },
      "geometry": [ ]
    },
    "building-library": {
      "type": "Building",
      "attributes": {
        "storeysAboveGround": 3,
        "measuredHeight": 15.2,
        "+co2-operational": 45.1,
        "+co2-embodied": 320.0,
        "+co2-structure-material": "timber",
        "+co2-energy-rating": "D",
        "+co2-lifecycle-phases": [
          { "phase": "A1-A3", "description": "材料生產", "value": 195.0 },
          { "phase": "B6", "description": "營運能耗", "value": 45.1 },
          { "phase": "C3-C4", "description": "廢棄處理", "value": 80.0 }
        ]
      },
      "geometry": [ ]
    }
  },
  "vertices": []
}
```

注意這裡的 `+` 前綴——任何讀取這個檔案的軟體，不需要載入 Extension 就能知道哪些是核心屬性、哪些是擴充屬性。

---

## Step 4：用 `cjval` 驗證

### 驗證 Extension 檔案本身

先用 `cjvalext` 確認 `.ext.json` 檔案格式正確[^cjval]：

```bash
cjvalext carbon-footprint.ext.json
```

如果輸出 `✓ Valid extension`，表示 Extension 檔案的結構是合法的。

你也可以用任何 JSON Schema 驗證器來檢查——只要對照官方的 [extension.schema.json](https://raw.githubusercontent.com/cityjson/cjval/main/schemas/extensions/20/extension.schema.json)[^ext-schema] 即可。

### 驗證 CityJSON 檔案中的 Extension

驗證使用了 Extension 的 CityJSON 檔案[^cjval]：

```bash
cjval mycity-with-carbon.city.json
```

`cjval` 會自動從 CityJSON 檔案中宣告的 `extensions` URL 下載 Extension 定義，然後檢查每個 Building 物件的 `+co2-*` 屬性是否符合我們定義的 Schema。

如果你在開發階段不想從遠端 URL 載入（可能尚未部署），可以用 `-e` 指定本地 Extension 檔案：

```bash
cjval mycity-with-carbon.city.json -e ./carbon-footprint.ext.json
```

### 常見驗證錯誤

| 錯誤情境 | 驗證結果 | 原因 |
|----------|----------|------|
| 能源等級寫成 `"A+"` | ✅ 通過 | 在 `enum` 中 |
| 能源等級寫成 `"A++"` | ❌ 錯誤 | `"A++"` 不在 `enum` 清單中 |
| 結構材料寫成 `"brick"` | ❌ 錯誤 | `"brick"` 不在 `enum` 中（可考慮加入） |
| 生命週期少寫 `value` | ❌ 錯誤 | `value` 在 `required` 中 |
| 碳排放設為 `-5.0` | ❌ 錯誤 | `minimum: 0.0` |

---

## 進階：加入 Extension 定義的進階型別

如果我們的碳足跡模型中，`+co2-operational` 的單位也需要像昨天看到的 `+area-parcel` 那樣使用 `value + uom` 模式，我們可以使用物件型別：

```json
"+co2-operational": {
  "type": "object",
  "properties": {
    "value": { "type": "number", "minimum": 0.0 },
    "uom": { "type": "string", "enum": ["kgCO2e/m2/year"] }
  },
  "required": ["value", "uom"],
  "additionalProperties": false
}
```

對應的資料會變成：

```json
"+co2-operational": {
  "value": 28.3,
  "uom": "kgCO2e/m2/year"
}
```

這樣做的好處是**單位與數值一起傳遞**，不會有單位混淆的問題[^cityjson-specs]。

---

## 如何發布你的 Extension

開發完成後，你有兩個選擇：

### 選項 A：提交流官方 Registry

1. 將 Extension 放到 GitHub 上
2. 向 [cityjson/extensions](https://github.com/cityjson/extensions) 提交 Pull Request[^extensions-registry]
3. 遵循規定的目錄結構：

```
extensions/
  carbon-footprint/
    1.0.0/
      carbon-footprint.ext.json
      extension.toml
      README.md
      examples/
      LICENSE.txt
```

### 選項 B：自行託管

只要 Extension 檔案可以透過公開 HTTP URL 存取即可，不需要註冊。對內部專案或實驗性擴充來說，這是最快的方式。

---

## 結語

今天我們從零開始，走完整個 Extension 開發流程：

1. **定義需求** — 決定要擴充什麼、需要哪些資料欄位
2. **撰寫 Schema** — 使用 JSON Schema 為每個屬性定義型態、範圍與約束
3. **在資料中使用** — 以 `+` 前綴在 City Object 的 `attributes` 中填入資料
4. **驗證** — 使用 `cjvalext` 驗證 Extension 檔案、`cjval` 驗證資料

這看起來步驟不少，但重點在於：**一旦定義好 Extension，你的資料就變成了具有型別檢查的「結構化擴充」**，而不是隨手亂加的 JSON 屬性。

明天開始我們將進入 Phase 5——實戰應用篇。首先來看看 CityJSON 如何處理**超大規模的城市模型**。

---

## 參考資料

[^cityjson-specs]: CityJSON Specification 2.0.2. (2024). §8 Extensions. Retrieved 2026-09-27, from https://www.cityjson.org/specs/2.0.2/#extensions

[^json-schema]: JSON Schema. (n.d.). JSON Schema Specification. Retrieved 2026-09-27, from https://json-schema.org/specification

[^shed-extension]: Hugo Ledoux. (2024). Shed Extension (0.1.1) — Template Extension Example. Retrieved 2026-09-27, from https://github.com/cityjson/extensions/tree/main/extensions/shed/0.1.1

[^cjval]: CityJSON. (2024). cjval — CityJSON Validator. Retrieved 2026-09-27, from https://github.com/cityjson/cjval

[^ext-schema]: CityJSON. (2024). Extension Schema (v2.0). Retrieved 2026-09-27, from https://raw.githubusercontent.com/cityjson/cjval/main/schemas/extensions/20/extension.schema.json

[^extensions-registry]: CityJSON. (2024). CityJSON Extensions Registry. Retrieved 2026-09-27, from https://github.com/cityjson/extensions

[^epd-denmark]: EN 15978. (2011). Sustainability of construction works — Assessment of environmental performance of buildings. European Committee for Standardization.