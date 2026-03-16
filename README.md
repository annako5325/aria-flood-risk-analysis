# Taiwan Flood Risk Analysis（ARIA）

本專案實作 **Automated Regional Impact Auditor (ARIA)**，  
利用地理空間分析方法評估台灣避難收容處所在洪水風險區域中的分布情形，並分析各行政區避難收容量是否足以支應可能的疏散需求。

本分析整合以下資料來源：

- 水利署河川圖資（River Polygon）
- 消防署避難收容處所資料（CSV）
- 鄉鎮市區行政界線
- 行政區人口統計資料

透過多級河川緩衝區分析與空間連接，判斷避難所的洪災風險等級，並進一步評估各行政區的避難容量缺口。

---

# 分析流程

## A. 資料載入與清理（Data Ingestion & Cleaning）

載入資料包含：

- 河川面 Shapefile（WRA）
- 避難收容處所 CSV（消防署）
- 鄉鎮市區行政界線
- 行政區人口統計資料

資料清理步驟：

- 移除經緯度為 0 或異常的避難所座標
- 透過鄉鎮界線進行空間過濾，移除海上或錯誤位置點位
- 將所有空間資料統一轉換為 **EPSG:3826（TWD97 / TM2）**

---

## B. 多級河川緩衝區分析（Multi-Buffer Risk Zoning）

建立三層河川警戒緩衝區：

| 風險等級 | 距離 |
|--------|------|
| 高風險 | 500 m |
| 中風險 | 1000 m |
| 低風險 | 2000 m |

利用 `geopandas.sjoin()` 進行空間連接，判斷避難所落在哪一層緩衝區內。

若避難所同時位於多個緩衝區，則採用 **最高風險等級** 作為最終分類。

---

## C. 收容量缺口分析（Capacity Gap Analysis）

以鄉鎮市區為單位進行統計，計算：

- 各行政區高 / 中 / 低風險避難所數量
- 各行政區避難所收容總量
- 安全區避難所收容量

並依據人口統計估算疏散需求。

本研究假設：


疏散比例 = 20% 行政區人口


收容量缺口定義為：


收容量缺口 = 需疏散人口 − 安全區收容量


依此找出 **避難收容量不足的行政區**，並產生 **風險最高 Top 10 行政區**。

---

## D. 視覺化（Visualization）

產生統計圖表：

**Top 10 高風險行政區**

- 風險避難所數量
- 風險區收容量 vs 安全區收容量

輸出圖檔：


risk_map.png


---

# 輸出成果

| 檔案 | 說明 |
|----|----|
| `ARIA.ipynb` | 完整分析 Notebook |
| `shelter_risk_audit.json` | 避難所風險清單 |
| `risk_map.png` | Top 10 高風險行政區統計圖 |
| `README.md` | 專案說明與 AI 診斷日誌 |

---

# AI 診斷日誌（AI Diagnostic Log）

在本專案開發過程中，曾遇到以下技術問題：

### 1. Jupyter Kernel 變數遺失

問題：

在 Notebook 執行過程中，部分變數（例如 `district_stats`）偶爾無法找到。

原因：

Jupyter Kernel 在某些情況下重新啟動，導致記憶體中的變數消失。

解決方式：

- 重新啟動 Kernel
- 執行 **Run All Cells** 確保完整流程重新執行

---

### 2. 人口資料型態錯誤

問題：

人口欄位 `P_CNT` 讀入時為字串型態，導致計算疏散人口時出現錯誤：


TypeError: can't multiply sequence by non-int of type 'float'


解決方式：

將資料轉換為數值型態：

```python
population_df["P_CNT"] = pd.to_numeric(population_df["P_CNT"], errors="coerce")
3. 避難所座標異常

問題：

部分避難所點位落在海上或行政區之外。

解決方式：

利用鄉鎮市區界線進行空間過濾，只保留位於台灣行政區範圍內的避難所。