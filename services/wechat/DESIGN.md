# 微信服務號後端設計文檔

> 版本：v1　日期：2026-10-06

---

## 目標

在微信服務號（已認證，企業主體）上提供「人生決策問答」：用戶發一條消息，返回基於《高性價比人生指南》正文的回答。

---

## 已確認決策

| 項 | 決策 |
|----|------|
| 服務號類型 | 已認證企業服務號 |
| 回覆接口 | **待確認**（見下方「⚠️ 開放問題 1」） |
| LLM | 外部 API，默認 DeepSeek |
| 知識庫 | Cloudflare AI Search（書庫改動無需重新部署 Worker） |
| 代碼位置 | `services/wechat/` |
| SCF | 重用 `../Ruminer/ruminer-wechat-service/scf/wechat-proxy/`（需改轉發目標） |
| 域名 | atmaware.cn（已備案） |
| 用戶記憶 | v1 只保留近期對話輪次，閒置後清除，不做長期記憶 |
| 合規 | 生成式 AI 備案、PIPL 等由公司負責 |
| Skill 改動 | 需重新部署 Worker |

---

## ⚠️ 開放問題（開工前請確認）

### ~~問題 1：回覆接口~~（已確認）

用戶在**服務號私信**頁面發消息、收回覆，不做客服聊天窗口那種 UX。  
Ruminer 裡那條客服聊天窗口的交互模塊（`/cgi-bin/message/custom/send` 主動推送）**不採用**。

採用**方案 B 的結構**（5 秒內返回空串、後台異步生成）但回覆路徑走**被動回覆 + 微信重試窗口**：
- 第一次收到消息：投 DO 隊列，返回空串，開始推理。
- 微信在約 15 秒內重試最多 3 次；如果答案已好，用 MsgId 去重後返回被動 XML 回覆。
- 15 秒還沒好（複雜問題）：最後一次重試前返回「正在查詢中，請稍後發送任意消息獲取結果」，下一條消息觸發時返回緩存的答案。
- 這個兜底方案需要 DO 本地緩存上一條待發答案，用 `alarm` 在 5 分鐘後清除。

---

## 架構

```
用戶
  │ 微信消息（POST XML）
  ▼
SCF relay（騰訊雲函數，固定出口 IP）
  │ 轉發到 Worker（x-proxy-secret 校驗）
  ▼
Cloudflare Worker：wechat-callback
  │ 驗簽、去重、返回空串（<5s）
  │ waitUntil → DO
  ▼
Durable Object：LifeAgent（每 openid 一個實例）
  │ submitMessages(idempotencyKey=MsgId)
  │  ├─ AI Search 查書（env.BOOK_SEARCH）
  │  └─ DeepSeek API 生成回答
  │ onChatResponse → 方案 B: 客服接口
  │                → 方案 A: 緩存，等微信重試
  ▼
SCF relay（/cgi-bin/message/custom/send）
  │ x-proxy-secret
  ▼
微信服務端
```

### 組件職責

| 組件 | 說明 |
|------|------|
| SCF relay | ① 轉發 `/cgi-bin/*` 到 `api.weixin.qq.com`（固定出口 IP 滿足 access_token 白名單）；② 把微信 callback 轉發到 Worker（可選，直接走自定義域名更簡單） |
| Cloudflare Worker | HTTP 入口，驗簽、去重、5 秒內返回、把消息投入 DO |
| Durable Object | 每用戶一個，存對話歷史、調 AI Search 和 DeepSeek、通過客服接口回覆 |
| AI Search | 知識庫，書庫條目存在 built-in storage，CI 上傳更新 |
| KV | access_token 緩存（key: `wechat:access_token`）、消息去重（key: `wxmsg:<MsgId>`，TTL 600s） |

---

## 消息流（方案 B 展開）

```
1. 微信 POST XML → SCF → Worker
2. Worker 驗簽（SHA1：token + timestamp + nonce）
3. 去重：KV.get("wxmsg:<MsgId>")，存在則返回空串
4. KV.put("wxmsg:<MsgId>", "1", {expirationTtl: 600})
5. 返回 HTTP 200 空串（<1s，微信不再重試）
6. waitUntil(do.submitMessages(msgs, {idempotencyKey: MsgId}))
7. DO 推理：
   a. AI Search 查 5 條相關條目
   b. 緊急詞前置檢查（見下）
   c. DeepSeek 生成回答
8. onChatResponse → wechatSendText（via SCF）
9. 答案 >2000 字時截斷並附「完整回答：https://...」
```

### 緊急詞前置檢查

在步驟 7a 之前先做關鍵詞匹配，不調 AI，直接返回：

| 觸發詞 | 立即回覆 |
|--------|---------|
| 急救、心臟驟停、大出血、火災、溺水、中毒 | 請立即撥打 120 / 119。（第 13 節有現場急救步驟） |
| 自殺、活不下去、不想活 | 全國心理援助熱線：12356（24 小時）。 |

---

## AI Search 知識庫

### 數據結構

每個書庫條目上傳為一個文件，key 格式：`book/S{section}-{index}.md`

文件正文就是原始 Markdown 條目文本（`### N. 標題` 到下一個 `### ` 之間的全部內容）。

自定義元數據（5 個字段上限）：

| 字段 | 類型 | 說明 |
|------|------|------|
| `section` | number | 節號（1–34） |
| `grade` | text | 證據等級：`A`、`B`、`C` |
| `outcome` | text | 口徑：`死亡率`、`金钱`、`时间`、`自由` |
| `level` | text | 收益量級：`大`、`中`、`小` |
| `ratio` | text | 性價比檔：`极高`、`高`、`一般` |

性價比檔在 CI 上傳時按 `index.html` 里的 `COST_W`/`e.ratio` 邏輯預計算好，Agent 不需要在運行時計算。

### 檢索參數

```ts
instance.search({
  messages: [{ role: "user", content: userQuery }],
  ai_search_options: {
    retrieval: {
      retrieval_type: "hybrid",  // 語義 + BM25，對中文條目效果更好
      max_num_results: 5,
      keyword_match_mode: "or",  // 關鍵詞任一匹配，召回更多
    },
    reranking: { enabled: true, model: "@cf/baai/bge-reranker-base" },
  },
})
```

**月度配額**：AI Search 每月包含 1000 次語義查詢和 1000 次全文查詢（hybrid 各消耗一次，即每次問答消耗一對）。超出後 hybrid 查詢 $0.75 / 千次，全文 $0.10 / 千次。如果 v1 問答量少（月均數百次），Free 配額夠用；達到數千次時考慮限流或降為純 keyword 查詢節省費用。

### CI 上傳管道

`.github/workflows/book.yml` 在 `main` 上觸發後，追加一個 job：

```yaml
- name: Upload to AI Search
  run: node tools/aisearch/upload.mjs
  env:
    CF_ACCOUNT_ID: ${{ secrets.CF_ACCOUNT_ID }}
    CF_API_TOKEN: ${{ secrets.CF_API_TOKEN }}
    AI_SEARCH_INSTANCE: howtolivebetter-book
```

`tools/aisearch/upload.mjs` 做的事：
1. 解析 `book/*.md` 拿到所有條目（復用 `tools/lib/book.mjs` 的解析邏輯）
2. 對每個條目按 `COST_W`/`e.ratio` 算出 `ratio`
3. 調 AI Search Items API 上傳；key 已存在就覆蓋（upload 是 upsert）
4. 刪除書庫中已不存在的舊條目

built-in storage 的文件上傳後立即索引，不需要 sync job，書庫更新不需要重新部署 Worker。

---

## 用戶身份與對話記憶

- DO 名稱：`openid` 的 HMAC-SHA256（用 `WECHAT_APP_SECRET` 作 key），避免裸 openid 出現在 DO 名稱裡
- 每次對話最多保留最近 10 輪（20 條消息）
- 閒置超過 24 小時後清除歷史（DO alarm 設置）
- v1 不做跨輪次的長期用戶畫像

```ts
// ponytail: 10 輪硬截斷，如果需要更長上下文改這個常量
const MAX_TURNS = 10;
```

---

## SCF 改動

重用 `../Ruminer/ruminer-wechat-service/scf/wechat-proxy/index.js`，只改一行：把硬編碼的轉發目標改成環境變量：

```js
// 原來：
const TARGET = 'https://api.ruminer.app/wechat/callback';
// 改為：
const TARGET = process.env.WECHAT_WORKER_URL; // e.g. https://wechat.atmaware.cn/callback
```

`/cgi-bin/*` 的轉發邏輯不動。

微信 callback 入站：優先讓微信直接回調 Worker 的自定義域名（`wechat.atmaware.cn`），SCF 轉發只在調試或備用時用。

---

## Workers Free 限制與風險

| 項目 | 限制 | 影響 |
|------|------|------|
| Worker CPU | 10 ms / 請求 | 入口 Worker 只做驗簽和投隊列，不跑推理，安全 |
| DO CPU | 同上，但 `waitUntil` 延長到 30s | DeepSeek 調用在 DO 裡，IO 等待不計入 CPU，預計 OK |
| DO subrequest | 50 次 / 請求 | 一次問答：1 次 AI Search + 1 次 DeepSeek + 1 次客服接口 = 3 次，安全 |
| Worker 請求數 | 10 萬 / 天 | 對應 ~7 問答/分鐘，超出升級 Paid（$5/月） |
| DO 請求數 | 10 萬 / 天 | 同上 |
| DO 存儲 | 1 GB / 實例 | 每用戶 10 輪歷史幾 KB，不是瓶頸 |
| KV 寫入 | 100 萬 / 天（Free）| 每條消息 2 次 KV 寫入，10 萬問答 / 天無壓力 |

**主要風險**：DO 的 10 ms CPU 限制在每次請求的同步路徑上。`submitMessages` 是異步接受（返回後繼續在後台跑），實際推理不在同步路徑，應該沒問題。上線前需壓測確認 DO 同步初始化路徑（讀歷史、驗簽）是否在 10 ms 內。

---

## Skill 適配說明

`skills/life-decision-guide/SKILL.md` 的第 1–3 步目前假設有 `git` 和 `grep`。Worker 環境裡這些不可用，改為：

- 第 1 步：調 AI Search 工具（`env.BOOK_SEARCH.search(...)`）
- 第 3 步：把 search 返回的 `chunks[].text` 直接當條目正文

第 4、5 步的排序和答案結構邏輯保持不變。

**Skill 改動需要重新部署 Worker**，可以在 Skill 文件裡加一段 `## Worker 模式` 說明，這樣同一份 Skill 既能給 Claude Code 用，也能給 Worker 看。

---

## Secrets 和配置

以下均通過 `wrangler secret put` 設置，不進代碼庫：

| 名稱 | 說明 |
|------|------|
| `WECHAT_TOKEN` | 微信服務器配置的 token（驗簽用） |
| `WECHAT_AES_KEY` | 消息加解密 key（啟用加密模式時） |
| `WECHAT_APP_ID` | 服務號 AppID |
| `WECHAT_APP_SECRET` | 服務號 AppSecret（也用作 openid HMAC key） |
| `SCF_WECHAT_PROXY_URL` | SCF 函數 URL |
| `SCF_PROXY_SECRET` | SCF `x-proxy-secret` 校驗頭 |
| `DEEPSEEK_API_KEY` | DeepSeek API key |

`wrangler.toml` 裡的 bindings（非 secret）：

```toml
[[kv_namespaces]]
binding = "KV"
id = "..."

[[ai_search]]
binding = "BOOK_SEARCH"
instance_name = "howtolivebetter-book"
remote = true  # 本地開發用

[[durable_objects.bindings]]
name = "LIFE_AGENT"
class_name = "LifeAgent"
```

---

## 日誌與隱私

- 日誌不記錄消息正文
- openid 記為 HMAC 後的 DO 名稱，原始 openid 不落日誌
- access_token 不落日誌
- DeepSeek 如果走 Cloudflare AI Gateway，可在 Gateway 做請求日誌（只記元數據）

---

## 每用戶限流（建議）

```ts
// ponytail: 每用戶每天 20 次，KV 計數，升規模時換成 Rate Limiting binding
const DAILY_LIMIT = 20;
```

超出限制返回提示：「今天的問答次數已用完，明天再來。」

---

## 文件結構（規劃）

```
services/wechat/
  DESIGN.md               ← 本文件
  wrangler.toml
  src/
    index.ts              ← Worker 入口（驗簽、去重、投 DO）
    agent.ts              ← LifeAgent Durable Object
    wechat.ts             ← 微信消息解析、XML 構建、客服接口發送
    token.ts              ← access_token 管理（KV 緩存）
    search.ts             ← AI Search 工具封裝
    safety.ts             ← 緊急詞前置檢查
  tools/aisearch/
    upload.mjs            ← CI 用：解析 book/，上傳到 AI Search
package.json
```

---

## 待辦（實現前）

1. **確認回覆接口**（開放問題 1），決定走方案 A 還是方案 B
2. SCF 環境變量改動並重新部署
3. 創建 AI Search instance `howtolivebetter-book`，定義 5 個元數據字段
4. 寫 `tools/aisearch/upload.mjs`，跑一次初始上傳
5. 決定是否讓微信 callback 直達 Worker（繞過 SCF 入站轉發）
6. 壓測 DO 同步路徑的 CPU 耗時
