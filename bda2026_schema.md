# bda2026 資料庫說明

> 台灣股票市場研究型資料庫 — 財務數據 × 社群文本  
> 資料期間：2023-03-01 ～ 2025-02-28 ｜ MySQL 8.0+ ｜ utf8mb4

---

## 一、資料庫概覽

| 表格 | 說明 | 筆數 |
|------|------|------|
| `companies` | 上市/上櫃公司基本資料 | 1,932 家 |
| `stock_prices` | 每日收盤價（1,887 家公司） | 877,699 筆 |
| `income_stmt` | 損益表季報（2023Q1～2024Q4） | 14,968 筆 |
| `stock_text` | 財經文章與新聞（5 個平台） | 1,125,134 篇 |

### stock_text 來源分布

| 平台 | 篇數 |
|------|------|
| PTT Stock | 55,741 |
| Dcard 股票 | 335,122 |
| Mobile01 投資 | 208,502 |
| Yahoo新聞 股市匯市 | 27,570 |
| Yahoo股市 財經新聞 | 498,199 |

---

## 二、表格結構

### companies（公司基本資料）

```sql
CREATE TABLE companies (
    company_id   VARCHAR(10)  NOT NULL,  -- 股票代號（主鍵）
    company_name VARCHAR(100) NOT NULL,  -- 公司全名
    short_name   VARCHAR(45),            -- 公司簡稱
    PRIMARY KEY (company_id)
);
```

| 欄位 | 型態 | 說明 | 範例 |
|------|------|------|------|
| `company_id` 🔑 | VARCHAR(10) | 股票代號（主鍵） | 2330 |
| `company_name` | VARCHAR(100) | 公司全名 | 台灣積體電路製造股份有限公司 |
| `short_name` | VARCHAR(45) | 公司簡稱 | 台積電 |

---

### stock_prices（每日收盤價）

```sql
CREATE TABLE stock_prices (
    id            INT            AUTO_INCREMENT PRIMARY KEY,
    company_id    VARCHAR(10)    NOT NULL,   -- 股票代號
    trade_date    DATE           NOT NULL,   -- 交易日期
    closing_price DECIMAL(10,2),             -- 收盤價（元）
    UNIQUE KEY (company_id, trade_date)
);
```

| 欄位 | 型態 | 說明 | 範例 |
|------|------|------|------|
| `id` 🔑 | INT | 自動遞增主鍵 | 1 |
| `company_id` 🔗 | VARCHAR(10) | 股票代號（關聯 companies） | 2330 |
| `trade_date` | DATE | 交易日期 | 2024-01-02 |
| `closing_price` | DECIMAL(10,2) | 當日收盤價（元） | 593.00 |

> ⚠️ 只有交易日才有資料，週末與假日無記錄。

---

### income_stmt（損益表季報）

```sql
CREATE TABLE income_stmt (
    id                        INT            AUTO_INCREMENT PRIMARY KEY,
    company_id                VARCHAR(10)    NOT NULL,
    fiscal_year               SMALLINT       NOT NULL,
    fiscal_quarter            TINYINT        NOT NULL,  -- 1=Q1, 2=Q2, 3=Q3, 4=Q4
    period_start              DATE,
    period_end                DATE,
    revenue                   DECIMAL(20,0), -- 千元
    cost_of_revenue           DECIMAL(20,0), -- 千元
    operating_income          DECIMAL(20,0), -- 千元
    net_income                DECIMAL(20,0), -- 千元
    total_comprehensive_income DECIMAL(20,0),-- 千元
    eps                       DECIMAL(10,2)  -- 元/股
);
```

> ⚠️ 所有金額為**累計數字（YTD）**，單位**千元**（EPS 為元/股）。  
> Q2 單季營收 = Q2累計 − Q1累計

---

### stock_text（財經文章與新聞）

```sql
CREATE TABLE stock_text (
    no           INT           AUTO_INCREMENT PRIMARY KEY,
    id           VARCHAR(200),  -- 原始文章 ID
    p_type       VARCHAR(20),   -- 平台類型：bbs / forum / news
    s_name       VARCHAR(40),   -- 來源平台：Ptt / Dcard / Mobile01 / Yahoo股市
    s_area_name  VARCHAR(40),   -- 版面/頻道
    post_time    DATETIME,      -- 發文時間
    title        TEXT,          -- 文章標題
    author       TEXT,          -- 作者
    content      LONGTEXT,      -- 文章完整內容
    page_url     LONGTEXT,      -- 原始網頁連結
    content_type VARCHAR(20)    -- main / reply / NULL（PTT、Yahoo 無此欄位）
);
```

| 欄位 | 型態 | 說明 | 範例 |
|------|------|------|------|
| `no` 🔑 | INT | 主鍵 | 1 |
| `p_type` | VARCHAR(20) | 平台類型 | bbs / forum / news |
| `s_name` | VARCHAR(40) | 來源平台名稱 | Ptt / Dcard / Mobile01 / Yahoo股市 |
| `post_time` | DATETIME | 發文時間 | 2024-01-15 10:23:00 |
| `title` | TEXT | 文章標題 | [討論] 台積電明年展望如何？ |
| `content` | LONGTEXT | 文章完整內容 | （內文） |
| `content_type` | VARCHAR(20) | 主文/回文 | main / reply / NULL |

> ⚠️ Dcard 與 Mobile01 含回文（`content_type = 'reply'`）。  
> 只取主文請加：`WHERE content_type = 'main' OR content_type IS NULL`

---

## 三、表格關聯

```
companies (company_id)
    ├─── 1:多 ──→ stock_prices (company_id)
    └─── 1:多 ──→ income_stmt  (company_id)

stock_text ── 無直接外鍵，透過 title/content 文字比對關聯
```

---

## 四、常用查詢範例

### 篩選特定公司文章（關鍵字比對）
```sql
SELECT no, post_time, title, s_name
FROM stock_text
WHERE (title LIKE '%華城%' OR content LIKE '%華城%' OR title LIKE '%1519%')
  AND (content_type = 'main' OR content_type IS NULL)
ORDER BY post_time;
```

### 查詢特定公司股價
```sql
SELECT trade_date, closing_price
FROM stock_prices
WHERE company_id = '1519'
ORDER BY trade_date;
```

### 計算 n 日後漲跌幅
```sql
SELECT 
    a.trade_date AS D_date,
    a.closing_price AS price_D,
    b.closing_price AS price_Dn,
    ROUND((b.closing_price - a.closing_price) / a.closing_price * 100, 2) AS pct_change
FROM stock_prices a
JOIN stock_prices b 
    ON b.company_id = a.company_id
    AND b.trade_date = (
        SELECT trade_date FROM stock_prices
        WHERE company_id = a.company_id AND trade_date > a.trade_date
        ORDER BY trade_date LIMIT 1 OFFSET 2  -- n=3 交易日後
    )
WHERE a.company_id = '1519'
ORDER BY a.trade_date;
```

---

## 五、重要注意事項

1. **損益表為累計數字（YTD）**：Q2單季 = Q2累計 − Q1累計
2. **損益表金額單位為千元**：換算億元請除以 100,000
3. **stock_prices 只有交易日**：週末、假日無記錄
4. **stock_text 含回文**：Dcard/Mobile01 的 `content_type='reply'` 需過濾
5. **字元集為 utf8mb4**：連線時請指定 `charset='utf8mb4'`
