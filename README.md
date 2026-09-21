# Buy the Dip on Bitcoin（比特幣相對低點抄底）

本文件說明目前已在運作的比特幣相對低點抄底流程：只在 1 小時 K 線收盤並確認相對低點之後，於樞紐低點掛出限價買單。此處只記錄既有行為，不描述尚未實作的策略。

## Goal

- **標的**：`BTCUSD_PERP`（幣安幣本位永續合約，coin-margined perpetual）
- **週期**：只用已收盤的 1 小時（1h）K 線；當根尚未收盤的 K 線不觸發下單
- **動作**：相對低點確認後，在樞紐低點（pivot low）掛出 **LIMIT BUY**，數量 **1 張合約**（名目價值 100 USD），時效 **GTC**（Good Till Cancel，成交或取消前持續有效）

## Relative-low rules (both used; either triggers one order)

3 根與 5 根兩套規則同時使用。任一規則成立，就針對該樞紐低點下一筆訂單。

同一個樞紐低點即使兩套規則都符合，也只下一筆訂單。

### 3-bar

以連續三根已收盤的 1h K 線為一組，中間那根為候選樞紐：

- 中間 K 線的最低價低於左右兩根的最低價
- 左右兩根的最高價都高於中間 K 線的最高價
- 右側 K 線收盤後才確認；確認前不下單

### 5-bar

以連續五根已收盤的 1h K 線為一組，第 3 根為候選樞紐：

- 第 3 根的最低價是這五根之中的最低價
- 相鄰 K 線的最高價不需要高於中間 K 線的最高價
- 第 5 根 K 線收盤後才確認；確認前不下單

## Order / risk behavior

- **保留所有未成交的限價買單**：出現新的相對低點時，不取消較早掛出、尚未成交的買單
- **不做歷史回補**：只對上線之後才確認的訊號下實盤單。範例上線時點為台北時間 2026-09-15 22:10；此時間之前已確認的訊號不再補單
- **實盤 API 路徑**：幣安統一帳戶／Portfolio Margin 的幣本位（CM）介面，`POST /papi/v1/cm/order`。這不是傳統 `dapi` 的 Enable Futures 路徑
- **紙本帳本可與實盤分開**：紙上記帳僅供對照，與實盤委託分開記錄

## Hourly reporting

每小時回報一次，內容包含：

- 最新一根已收盤 1h K 線的開高低收（OHLC），時間以台北時間表示
- 新確認的訊號；若已送出實盤單，一併附上 `orderId` 與狀態
- 目前仍掛著、尚未成交的買單清單
- 帳戶變動：成交、取消、狀態變更

查詢可輪詢 `openOrders`、`allOrders`、`userTrades`。

## Architecture (current)

公開 1h K 線 → 偵測 3 根／5 根相對低點 → 可選的紙本狀態；實盤單以本機 `.env` 金鑰經 MSI 送出，呼叫 papi CM API。

- 行情來源是公開的 1 小時 K 線
- 訊號偵測同時套用 3 根與 5 根規則
- 紙本狀態為可選，與實盤分開
- 實盤下單讀取本機 `.env`，經 MSI 呼叫 `POST /papi/v1/cm/order`

### 金鑰與權限

- 密鑰只放在本機 `.env`，不要提交到版本庫
- 建議權限：讀取，以及統一帳戶／合約交易
- 不要開啟提現權限
- IP 白名單必須與實際出口 IP 一致。家用 IP 會變動；白名單不符時，介面可能回傳 `-2015`

## .env example (placeholders only)

以下僅為佔位符。請在本機 `.env` 填入自己的金鑰，不要把真實金鑰寫進版本庫。

```env
BINANCE_API_KEY=your_api_key_here
BINANCE_SECRET=your_api_secret_here
MODE=your_mode_here
SYMBOL=BTCUSD_PERP
CONTRACT_QTY=1
CONTRACT_SIZE_USD=100
```

| 變數 | 說明 |
| --- | --- |
| `BINANCE_API_KEY` | 幣安 API Key（佔位） |
| `BINANCE_SECRET` | 幣安 API Secret（佔位） |
| `MODE` | 運行模式 |
| `SYMBOL` | 交易標的，`BTCUSD_PERP` |
| `CONTRACT_QTY` | 每次限價買單張數，`1` |
| `CONTRACT_SIZE_USD` | 單張名目價值（USD），`100` |

## Known limitations / TODO

- 槓桿上限，以及未成交買單的最大掛單數，尚未參數化
- 「買突破」尚未實作
- 家用動態 IP 與交易所 IP 白名單不一致時需要手動更新，不符時容易出現 `-2015`
- 尚未實作出場、停損、停利
- 本文件與流程只說明既有自動化行為，不構成投資建議

## Links

- 儲存庫：<https://github.com/cequ3108/Buy-the-dip-on-Bitcoin>
