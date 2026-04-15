# Claude Code Prompt：股價漲跌預測 Pipeline（華城 1519）

請先做以下兩件事再開始寫程式：
1. 閱讀同目錄下的 `bda2026_schema.md` 以了解資料庫結構
2. 參考以下資料夾中的範例程式碼，了解本課程的實作風格與做法：
   `C:\Users\a3504\Desktop\學校與課業\課業\四下\人工智慧與商業分析\practice`
   （特別參考 example_3 統計TF/DF、example_4 計算相似文件、example_5/6 分類）

---

## 任務目標

建立一套模組化的股價漲跌預測 NLP pipeline，以台股財經文章預測特定股票 n 日後的漲跌方向。

---

## 專案參數（全部集中在每個模組最上方）

```python
# ── 可調整參數區 ──────────────────────────────────────
COMPANY_ID   = '1519'       # 股票代號
COMPANY_NAME = '華城'       # 用於關鍵字搜尋
N_DAYS       = 3            # 預測幾個交易日後的漲跌
SIGMA        = 0.03         # 漲跌門檻（3% = 0.03），低於此幅度不貼標籤
TOP_K_WORDS  = 200          # 建構向量空間使用的關鍵字數量
TEST_RATIO   = 0.2          # 測試資料比例

DB_CONFIG = {
    'host':     'localhost',
    'user':     'root',
    'password': 'your_password',  # 修改為實際密碼
    'database': 'bda2026',
    'charset':  'utf8mb4'
}
# ──────────────────────────────────────────────────────
```

---

## 模組規格

請建立以下獨立的 Jupyter Notebook（.ipynb），每個檔案**最上方**放參數區，每個主要方法標註 `# [METHOD] 目前方法：XXX | 可替換為：YYY`。

---

### step1_fetch_articles.ipynb — 篩選文章

**功能**：從 `stock_text` 撈出與目標公司相關的文章

**目前方法**：關鍵字 LIKE 比對（title 或 content 含公司名稱或股票代號）

**過濾條件**：
- 只取主文（`content_type = 'main' OR content_type IS NULL`）
- 文章時間需在 `stock_prices` 有資料的期間內

**輸出**：`articles_raw.csv`（欄位：no, post_time, title, content, s_name）

```python
# [METHOD] 目前方法：LIKE 關鍵字比對 | 可替換為：Aho-Corasick 多模式掃描
```

---

### step2_label.ipynb — 貼股價標籤

**功能**：對每篇文章，查找 D+n 個交易日後的收盤價變化，貼上 1（看漲）/ -1（看跌）/ 0（不出手）

**規則**：
- 漲幅 > SIGMA → label = 1
- 跌幅 > SIGMA → label = -1
- 介於中間 → label = 0（排除，不納入訓練/測試）

**注意**：n 日後需取「第 n 個交易日」（跳過週末假日，用 stock_prices 的實際交易日順序）

**輸出**：`articles_labeled.csv`（加入 label 欄位，label=0 的行也保留但標記）

```python
# [METHOD] 目前方法：固定 n 天後交易日 | 可替換為：事件窗口法（取最大漲跌）
```

---

### step3_vectorize.ipynb — 取詞建向量空間

**功能**：從看漲/看跌文章中取出具鑑別力的關鍵詞，將每篇文章轉為向量

**步驟**：
1. 對文章內容做 N-gram（2~4 gram 字元切割，不需斷詞套件）
2. 分別統計看漲文章和看跌文章的詞頻（TF）和文件頻率（DF）
3. 計算 TF-IDF × chi-square 指標，各取 TOP_K_WORDS/2 個鑑別詞
4. 合併為向量空間，每篇文章用詞頻轉為向量

**輸出**：`vectors.pkl`（含 feature_list, X, y）

```python
# [METHOD] 目前方法：N-gram 字元切割 + TF-IDF chi-square | 可替換為：jieba/ckip 斷詞、Word2Vec
```

---

### step4_classify.ipynb — 分類器訓練與評估

**功能**：用多種分類演算法訓練並評估，輸出 confusion matrix

**分類器**：NB、KNN（k=5）、SVM、Decision Tree、Random Forest

**資料切割**：依時間順序切割（前 80% 訓練，後 20% 測試），**不用隨機切割**

**輸出**：每個分類器的 confusion matrix、準確率

```python
# [METHOD] 目前方法：時序切割 80/20 | 可替換為：移動回測（step5）
# [METHOD] 目前方法：NB/KNN/SVM/DT/RF | 可替換為：投票集成、LLM 零樣本分類
```

---

### step5_evaluate.ipynb — 移動回測與自動評估

**功能**：模擬真實預測情境，逐日移動訓練窗口，計算出手率與準確率

**規則**：
- 每次取前 30 天資料訓練模型
- 預測當天文章對應 D+n 日漲跌
- 若當天看漲/看跌文章數差距太小（差距 < 20%）→ 不出手
- 若當天文章篇數 < 5 → 不出手

**輸出**：
- 逐日預測結果表
- 總出手率
- 總準確率（confusion matrix）

```python
# [METHOD] 目前方法：30天滾動窗口 | 可替換為：60天、週為單位
```

---

## 整體執行順序

```
step1 → step2 → step3 → step4 → step5
```

每個步驟讀取上一步的輸出檔案，確保可以單獨重跑任何一步。

---

## 輸出格式要求

最終在 step5 輸出：

```
===== 移動回測結果 =====
總文章天數：XXX 天
出手天數：XXX 天（出手率：XX.X%）

Confusion Matrix：
              預測漲    預測跌
真實漲    |   XX   |   XX   |
真實跌    |   XX   |   XX   |

總準確率：XX.X%
```
