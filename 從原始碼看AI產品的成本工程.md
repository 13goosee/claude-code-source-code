# 從原始碼看 AI 產品的成本工程

> 基於 Claude Code v2.1.88 原始碼，拆解一家 AI 公司怎麼控制 token 成本
> 寫給想做 AI 產品但在意毛利率的人，不綁定任何特定產品

---

## 為什麼這篇比定價頁有用

每家 AI 公司的定價頁都告訴你一百萬 token 多少錢。沒有人告訴你：他們自己怎麼省 token。

這篇不是外部觀察。這是從一家 $3,800 億 AI 公司的**產品原始碼**裡，逐行讀出來的成本控制工程。他們的每一個設計決策——什麼時候壓縮、什麼時候用快取、什麼時候用便宜模型——都是在品質和成本之間的真實取捨。

對任何想做 AI 產品的人來說，**token 就是你的原料成本**。你的毛利率、單位經濟、能不能活過 Series A，都寫在你怎麼管理 token 裡。

---

## 一、Context 壓縮：對話太長怎麼辦？

### 問題的本質

AI 模型有 context window 上限（目前 Opus 4.6 是 1M tokens）。但更重要的是：**context 越長，每次 API call 越貴**。輸入 token 是按量計費的，一場 2 小時的 coding session 可能累積到幾十萬 token，每次你說一句話，這幾十萬 token 都要重新送一次。

### Anthropic 的三層壓縮架構

原始碼揭露了一個精密的三層壓縮系統，從輕到重：

**第一層：Micro-compact（工具結果清理）**

最輕量的壓縮。不叫 API，不花 token。只是把舊的工具輸出結果清空。

```typescript
// src/services/compact/microCompact.ts
const COMPACTABLE_TOOLS = new Set([
  'Read',        // 檔案讀取結果
  'Bash',        // 命令執行結果
  'Grep',        // 搜尋結果
  'Glob',        // 檔案搜尋結果
  'Edit',        // 編輯結果
  'Write',       // 寫入結果
  'WebFetch',    // 網頁抓取結果
  'WebSearch',   // 網頁搜尋結果
])
```

邏輯很簡單：你五分鐘前讀的檔案內容，模型已經「理解」過了，不需要在每次 API call 裡重複傳送。把舊的工具結果替換成 `[Old tool result content cleared]`，立刻省下大量 token。

但這裡有個精妙的設計：**保留最近的 N 個結果不清**。清太多，模型會失去工作記憶；清太少，省不了錢。這個 N 是透過遠端設定（GrowthBook）動態調整的，代表 Anthropic 還在持續優化這個平衡點。

更進階的版本叫 **Cached Micro-compact**，利用 Anthropic API 的 `cache_edits` 功能，不是修改本地訊息內容，而是告訴伺服器「幫我把這些工具結果從快取裡刪掉」。好處是不會破壞已有的 prompt cache（後面會詳細講），壞處是只有支援這個 API 的環境才能用。

還有一個 **Time-based Micro-compact**：如果使用者離開超過一段時間（快取肯定已經過期），系統會更積極地清理舊的工具結果，因為反正快取已經沒了，不如趁機瘦身。

**第二層：Auto-compact（自動摘要）**

當 token 用量逼近 context window 上限時觸發。系統啟動一個獨立的 AI 子任務，讓模型自己摘要整段對話。

```typescript
// src/services/compact/autoCompact.ts
const AUTOCOMPACT_BUFFER_TOKENS = 13_000
const MAX_OUTPUT_TOKENS_FOR_SUMMARY = 20_000
```

觸發閾值的計算：`有效 context window - 13,000 token buffer`。留 13K 的緩衝是因為——你不能在 100% 滿的時候才開始壓縮，那時候連壓縮的 API call 都塞不進去了。

壓縮 prompt 本身就是一件講究的事（`src/services/compact/prompt.ts`）。摘要要求包含 9 個章節：

1. 主要需求和意圖
2. 關鍵技術概念
3. 檔案和程式碼段落
4. 錯誤和修復
5. 問題解決過程
6. 所有使用者訊息（逐條保留）
7. 待辦任務
8. 當前工作
9. 下一步行動

注意第 6 點：**所有使用者訊息都要逐條保留**。這是一個產品決策——使用者說的話比 AI 回的話重要。使用者的意圖不能被「摘要」掉。

還有一個防呆設計：壓縮 prompt 的開頭用大寫加粗寫著 `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.`。為什麼？因為壓縮是用一個子 agent 跑的，這個子 agent 為了共享 prompt cache 繼承了父 agent 的所有工具。Sonnet 4.6 有 2.79% 的機率會在壓縮過程中嘗試呼叫工具，工具被拒絕後就沒有文字輸出，整個壓縮失敗。一行 prompt 修復了一個 2.79% 的失敗率。

**第三層：Session Memory Compact**

比 auto-compact 更激進——不只是摘要，是直接刪除舊訊息，只保留最近的。這是在 auto-compact 無法充分壓縮時的後備方案。

### 商業啟示

壓縮不只是技術問題。這三層架構對應三種成本/品質取捨：

| 壓縮層級 | 成本 | 品質損失 | 觸發時機 |
|---------|------|---------|---------|
| Micro-compact | 零（不叫 API） | 極低（只清工具輸出） | 每次 API call 前 |
| Auto-compact | 中（一次 API call） | 中等（摘要會丟細節） | 接近 context 上限 |
| Session Memory | 零（直接刪） | 較高（早期對話消失） | auto-compact 前的搶先嘗試 |

**如果你在做 AI 產品：** 不要等到 context window 爆了才想壓縮。學 Anthropic 的分層策略——能不花 token 壓的先壓，花 token 壓的晚點壓，壓不動的直接刪。

---

## 二、Prompt Cache：同樣的內容不要付兩次錢

### Cache 是 AI 產品最被低估的成本槓桿

Anthropic API 有一個 prompt caching 功能：如果你連續幾次 API call 的開頭部分（system prompt + 工具定義 + 早期對話）是一樣的，伺服器會快取這部分，下次只收 1/10 的價錢。

原始碼裡的定價清楚寫著（`src/utils/modelCost.ts`）：

| 項目 | Opus 4.6 正常 | Opus 4.6 快取讀取 |
|------|-------------|-----------------|
| 每百萬 input tokens | $5 | $0.50 |

**快取讀取只要正常價的 1/10。** 如果你的 system prompt + 工具定義有 5 萬 token（很常見），每次 API call 省 $0.225。一天跑 1,000 次就是省 $225。

### Anthropic 怎麼最大化 cache hit rate

原始碼裡有一個非常精密的快取斷點（cache breakpoint）策略（`src/services/api/claude.ts`）：

**規則一：每次 request 只放一個 `cache_control` 標記。**

```typescript
// Exactly one message-level cache_control marker per request. Mycro's
// turn-to-turn eviction frees local-attention KV pages at any cached
// prefix position NOT in cache_store_int_token_boundaries. With two
// markers the second-to-last position is protected and its locals
// survive an extra turn even though nothing will ever resume from
// there — with one marker they're freed immediately.
```

這段註解揭露了 Anthropic 內部推理系統（代號 Mycro）的記憶管理機制。放兩個標記會讓記憶體多留一個不需要的中間狀態。一個標記 = 最小記憶體佔用 = 伺服器能服務更多使用者 = 大家的快取都活更久。

**規則二：子 agent 的快取標記放在倒數第二則訊息。**

```typescript
const markerIndex = skipCacheWrite 
  ? messages.length - 2  // fork: 放在共享前綴的最後
  : messages.length - 1  // 正常: 放在最後一則訊息
```

為什麼？子 agent 是「用完即棄」的。如果把快取標記放在最後一則，子 agent 自己的尾巴會被寫入快取，佔用伺服器空間但永遠不會被重用。放在倒數第二則，標記落在父子 agent 共享的前綴上，寫入的快取可以被父 agent 的下一次 call 重用。

**規則三：1 小時的快取 TTL**

```typescript
function getCacheControl({ scope, querySource }) {
  return {
    type: 'ephemeral',
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
  }
}
```

預設的 prompt cache 只有 5 分鐘。但對於主要的對話迴圈，Anthropic 開了 1 小時的 TTL。使用者去泡杯咖啡回來，快取還在。這個 TTL 的選擇是透過 GrowthBook allowlist 控制的——不是所有場景都值得 1 小時快取（子 agent 可能只跑一次就不需要了）。

**規則四：快取感知的壓縮**

前面講的 micro-compact 有兩條路徑，其中一條（Cached MC）不直接修改訊息內容，而是用 API 的 `cache_edits` 告訴伺服器刪除特定的工具結果。**這保護了已有的 prompt cache。** 如果你直接改訊息，整個快取就失效了，下次 API call 要從頭花錢建快取。

原始碼裡甚至有專門偵測快取失效的系統（`promptCacheBreakDetection.ts`），當快取讀取量突然下降時會記錄事件。一次壓縮操作造成的快取失效被記錄為誤報（false positive），佔所有快取失效事件的 20%——可見他們多認真在追蹤這件事。

### 商業啟示

**Prompt cache 是 AI 產品最大的成本優化機會。** 但它需要你理解一件反直覺的事：**改善品質（壓縮歷史對話）可能反而增加成本（破壞快取）**。Anthropic 花了大量工程投入來解決這個矛盾：用 cache_edits 做無損壓縮、用精確的斷點放置來最大化快取重用、用偵測系統來追蹤快取效率。

如果你在做 AI 產品，你的 system prompt 和工具定義應該盡可能穩定、放在 API call 的最前面、標記為可快取。任何會頻繁改動的內容（使用者設定、動態 context）應該放在快取前綴之後。

---

## 三、那個每天浪費 250,000 次 API Call 的 Bug

### 三行程式碼，一天 250K 次 API call

這可能是整份原始碼裡最有教育意義的一段：

```typescript
// src/services/compact/autoCompact.ts

// BQ 2026-03-10: 1,279 sessions had 50+ consecutive failures (up to 3,272)
// in a single session, wasting ~250K API calls/day globally.
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3
```

背景：auto-compact 會在 context 接近上限時觸發。如果壓縮失敗（比如 context 已經大到連壓縮 API call 本身都塞不進去），下一輪對話又會觸發壓縮……然後又失敗……無限循環。

原本的程式碼沒有失敗次數上限。結果：

- **1,279 個 session** 各自跑了 **50 次以上**的連續失敗壓縮（最多的跑了 3,272 次）
- 全球每天浪費 **約 250,000 次 API call**
- 每次 API call 都要傳送整個 context（因為 context 已經很大了才觸發壓縮）
- 這些 API call 100% 是無意義的——它們注定會失敗

修復方式：加一個 circuit breaker，連續失敗 3 次就停止嘗試。

```typescript
if (
  tracking?.consecutiveFailures !== undefined &&
  tracking.consecutiveFailures >= MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES
) {
  return { wasCompacted: false }
}
```

### 這個 bug 的成本影響

讓我們算一筆帳。假設每個失敗的壓縮 API call 平均傳送 200,000 input tokens（context 已經接近上限才觸發）：

- 250,000 次 × 200,000 tokens = 50,000,000,000 input tokens/天
- 以 Opus 4.6 的 $5/百萬 tokens 計算 = **$250,000/天**
- 即使大部分被 prompt cache 覆蓋（$0.50/百萬），仍然是 **$25,000/天**

這還沒算 output tokens（即使失敗也會有部分輸出）和伺服器資源消耗。

### 商業啟示

**一個缺少上限的迴圈，在規模化後會變成一場財務災難。** 這不是什麼高深的技術 bug——是一個 while loop 沒有 break condition。任何工程師都可能寫出這種程式碼。

但在 AI 產品裡，每次迴圈迭代都在燒錢。傳統軟體的無限迴圈消耗的是 CPU；AI 產品的無限迴圈消耗的是**真金白銀**。

教訓：**你的 AI 產品裡每一個自動觸發的 API call，都需要一個 circuit breaker。** 不只是壓縮——自動重試、自動修正、自動展開，任何會在失敗後重新觸發的流程，都需要最大次數限制。

---

## 四、快速模式的定價信號：速度值 6 倍

### 同模型，6 倍價差

原始碼裡硬編碼的定價（`src/utils/modelCost.ts`）：

```typescript
// Opus 4.6 普通模式：$5 input / $25 output per Mtok
export const COST_TIER_5_25 = {
  inputTokens: 5,
  outputTokens: 25,
  promptCacheWriteTokens: 6.25,
  promptCacheReadTokens: 0.5,
}

// Opus 4.6 快速模式：$30 input / $150 output per Mtok
export const COST_TIER_30_150 = {
  inputTokens: 30,
  outputTokens: 150,
  promptCacheWriteTokens: 37.5,
  promptCacheReadTokens: 3,
}
```

同一個模型（Opus 4.6），同樣的能力，同樣的回答品質。唯一的差別是推理速度。Input 價差 6 倍，output 價差 6 倍。

更有趣的是判斷邏輯：

```typescript
// src/utils/modelCost.ts
export function getOpus46CostTier(fastMode: boolean): ModelCosts {
  if (isFastModeEnabled() && fastMode) {
    return COST_TIER_30_150
  }
  return COST_TIER_5_25
}
```

快速模式不是用不同模型，是 API response 裡帶一個 `speed: 'fast'` 標記。同一個模型，伺服器端分配更多算力來加速推理。

### 完整的模型定價梯度

從原始碼整理出的完整定價表：

| 模型 | Input $/Mtok | Output $/Mtok | Cache Read $/Mtok | 定位 |
|------|-------------|--------------|-------------------|------|
| Haiku 3.5 | $0.80 | $4 | $0.08 | 最便宜，輕量任務 |
| Haiku 4.5 | $1 | $5 | $0.10 | 便宜，稍好品質 |
| Sonnet 4/4.5/4.6 | $3 | $15 | $0.30 | 中間，性價比最高 |
| Opus 4/4.1 | $15 | $75 | $1.50 | 貴，最強推理 |
| Opus 4.5/4.6 | $5 | $25 | $0.50 | 新一代，便宜又強 |
| Opus 4.6 Fast | $30 | $150 | $3 | 同品質，6x 速度 |

最值得注意的價格變動：**Opus 從 4.1 到 4.5 降了 3 倍**（$15→$5 input），但**快速模式把價格拉回 $30**。這意味著 Anthropic 認為下一代模型的成本可以大幅下降，但速度仍然是稀缺資源。

### 商業啟示

**速度是 AI 產品裡最不理性、最願意付錢的維度。** 使用者對「等 30 秒」和「等 5 秒」的付費意願差 6 倍。

這對 AI 產品的定價策略有三個啟示：

1. **免費版 vs 付費版可以是速度差異**，而不是功能差異。功能差異讓免費使用者覺得被閹割；速度差異讓免費使用者覺得「能用，但如果我付費就更快」
2. **Cache 是速度的隱性推手。** Cache read 的價格是正常價的 1/10，但更重要的是快取命中時伺服器不需要重新計算前綴——直接影響回應速度。省錢和加速是同一件事
3. **每一代新模型都會便宜，但速度永遠能溢價。** 你的商業模式不應該建立在「模型很貴」的假設上（它會越來越便宜），應該建立在「使用者願意為速度付錢」的假設上（這不會變）

---

## 五、Token 追蹤：你花了多少錢，精確到美分

### 追蹤系統的設計

Claude Code 在使用者每個 session 結束時顯示完整的成本報告（`src/cost-tracker.ts`）：

```
Total cost:            $1.23
Total duration (API):  3m 45s
Total duration (wall): 12m 30s
Total code changes:    42 lines added, 15 lines removed
Usage by model:
      claude-opus-4-6:  150,234 input, 12,345 output, 
                        130,000 cache read, 5,000 cache write ($0.98)
    claude-haiku-4-5:  45,678 input, 3,456 output, 
                        40,000 cache read, 1,000 cache write ($0.25)
```

幾個設計決策值得注意：

**分模型記帳。** 不是一個總數，是每個模型分開算。因為一個 session 裡可能用了多個模型（Opus 做主要推理、Haiku 做搜尋、Sonnet 做子任務），使用者需要知道錢花在哪裡。

**分 token 類型記帳。** Input、output、cache read、cache write 分開算。為什麼要這麼細？因為它們的價格不同。Cache read 只要 1/10 的價格，如果你把它跟正常 input 混在一起，使用者會以為他花了 10 倍的錢。

**跨 session 持久化。** 成本狀態會存到磁碟（`saveCurrentSessionCosts`），如果使用者的 session 被中斷或恢復，成本數據不會丟失。

```typescript
// src/cost-tracker.ts
export function addToTotalSessionCost(
  cost: number,
  usage: Usage,
  model: string,
): number {
  const modelUsage = addToTotalModelUsage(cost, usage, model)
  addToTotalCostState(cost, modelUsage, model)
  
  // 快速模式的追蹤特別標注 speed: 'fast'
  const attrs = isFastModeEnabled() && usage.speed === 'fast'
    ? { model, speed: 'fast' }
    : { model }
  
  getCostCounter()?.add(cost, attrs)
  getTokenCounter()?.add(usage.input_tokens, { ...attrs, type: 'input' })
  getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' })
  getTokenCounter()?.add(usage.cache_read_input_tokens ?? 0, { ...attrs, type: 'cacheRead' })
  getTokenCounter()?.add(usage.cache_creation_input_tokens ?? 0, { ...attrs, type: 'cacheCreation' })
  
  return totalCost
}
```

**顧問模型（advisor）的成本也算。** 如果 API response 裡包含了 advisor 模型的用量（Anthropic 內部用於某些進階推理的輔助模型），也會一起算進去。使用者看到的是真實的總成本，不是被低估的數字。

### 商業啟示

**成本透明是信任的基礎。** Anthropic 選擇在每個 session 結束時直接告訴使用者花了多少錢。這看起來像是「把使用者嚇跑」，但其實是建立信任——如果使用者不知道他花了多少錢，他對整個產品的信任度會降低。

如果你的 AI 產品按用量計費，不要藏著計費細節。把成本拆開給使用者看：多少花在推理上、多少花在搜尋上、多少被快取省下來了。**使用者不是怕花錢，是怕不知道錢花在哪裡。**

---

## 六、模型選擇策略：什麼時候用貴的，什麼時候用便宜的

### 不是每個任務都需要最聰明的模型

Claude Code 的原始碼裡，不同的子 agent 使用不同的模型：

```typescript
// src/tools/AgentTool/built-in/exploreAgent.ts
// 外部使用者用 Haiku（便宜快速），內部員工用父 agent 的模型
model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku'

// src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts
// 問答查詢用 Haiku——不需要複雜推理
model: 'haiku'

// src/tools/AgentTool/built-in/statuslineSetup.ts
// 設定 UI 用 Sonnet——中等複雜度
model: 'sonnet'
```

翻譯成商業語言：

| 任務類型 | 模型選擇 | 原因 | 成本差異 |
|---------|---------|------|---------|
| 搜尋檔案、找程式碼 | Haiku（$1/Mtok） | 不需推理，只需快速掃描 | 最便宜 |
| 查文件、回答問題 | Haiku（$1/Mtok） | 資訊檢索，不需創造 | 最便宜 |
| 設定 UI、簡單操作 | Sonnet（$3/Mtok） | 需要一些理解但不複雜 | 中等 |
| 核心推理、寫程式 | Opus（$5/Mtok） | 需要最強的推理能力 | 較貴 |

Explore agent（搜尋 agent）用 Haiku 而不是 Opus，**每次搜尋的成本差 5 倍**。考慮到 Anthropic 的數據顯示**每週有 3,400 萬次以上的 Explore agent 調用**（原始碼註解裡的真實數字），這個模型選擇節省的成本是天文數字。

### 商業啟示

**模型選擇不是一個技術決策，是一個產品經濟學決策。** 你的產品裡不同的功能路徑應該用不同的模型：

- 搜尋、分類、簡單抽取 → 最便宜的模型
- 摘要、改寫、中等推理 → 中等模型
- 核心的創造性任務 → 最貴的模型

這跟傳統軟體的 CPU 優化邏輯完全不同。傳統軟體裡你不會說「這個 API endpoint 用 2 核，那個用 8 核」。但在 AI 產品裡，**每個功能路徑的「算力」是直接反映在帳單上的**。

---

## 七、Agent 套娃的成本控制

### 問題：一個 agent 叫另一個 agent，token 消耗怎麼控制？

當主 agent 呼叫一個子 agent（比如搜尋檔案），子 agent 需要自己的 context。如果子 agent 完整繼承父 agent 的 context，一個 10 萬 token 的對話叫一個子 agent，子 agent 的第一次 API call 就要花 10 萬 token 的 input 費用。子 agent 再叫子 agent？20 萬 token。層層套娃，成本指數爆炸。

### Anthropic 的四個控制手段

**手段一：精簡子 agent 的初始 context**

```typescript
// src/tools/AgentTool/runAgent.ts

// 唯讀 agent（Explore, Plan）不需要 CLAUDE.md 裡的 commit/PR/lint 規則
// 省 ~5-15 Gtok/week across 34M+ Explore spawns
const shouldOmitClaudeMd = agentDefinition.omitClaudeMd && ...

// 也省掉 git status（可能有 40KB）
// 省 ~1-3 Gtok/week fleet-wide
const { gitStatus: _omittedGitStatus, ...systemContextNoGit } = ...
```

這兩行程式碼每週省下 **6-18 Giga tokens**（6-18 × 10⁹ tokens）。以 Haiku 的 $1/Mtok 計算，每週省 **$6,000-$18,000**。

為什麼 CLAUDE.md 和 git status 可以省？因為 Explore agent 只是幫你找檔案，它不需要知道你的 commit 規則和 git 狀態。父 agent 會解讀子 agent 的結果。

**手段二：省掉一次性子 agent 的 metadata trailer**

```typescript
// src/tools/AgentTool/AgentTool.tsx

// One-shot built-ins (Explore, Plan) are never continued via SendMessage
// — the agentId hint and <usage> block are dead weight (~135 chars ×
// 34M Explore runs/week ≈ 1-2 Gtok/week)
if (data.agentType && ONE_SHOT_BUILTIN_AGENT_TYPES.has(data.agentType)) {
  return { content: contentOrMarker }  // 省掉 agentId 和 usage trailer
}
```

每個子 agent 的回覆本來會帶一個 `agentId: xxx (use SendMessage...)` 的尾巴和 `<usage>` 統計。對 Explore 和 Plan 這種一次性的子 agent，這些 metadata 永遠不會被使用。135 個字元 × 3,400 萬次/週 = 又省了 **1-2 Gtok/week**。

**手段三：子 agent 的快取前綴共享**

前面講的 cache breakpoint 策略裡有一個細節：子 agent 把快取標記放在倒數第二則訊息（共享前綴的末尾），而不是最後一則（子 agent 自己的內容）。這樣子 agent 的 API call 可以重用父 agent 的快取，不需要重新計算共享的 system prompt 和工具定義。

**手段四：maxTurns 限制**

```typescript
// src/tools/AgentTool/forkSubagent.ts
maxTurns: 200  // fork agent 最多跑 200 回合

// 而 Explore agent 則有更嚴格的限制
// （由 agent 定義的 maxTurns 控制）
```

每個子 agent 有最大回合數。到了上限就強制停止，避免子 agent 陷入無限迴圈（跟前面 250K bug 同樣的教訓）。

### 商業啟示

**Agent 套娃是 AI 產品最容易失控的成本來源。** 你的 AI 產品如果有「自主」功能（AI 決定什麼時候呼叫子任務），你需要：

1. **最小 context 原則**：子 agent 只拿到它需要的 context，不是全部
2. **快取共享**：子 agent 盡可能重用父 agent 的快取
3. **最大回合數**：每個自動觸發的流程都要有上限
4. **便宜模型優先**：不需要複雜推理的子任務用最便宜的模型

一個缺少這些控制的 AI agent 系統，成本曲線是指數的，不是線性的。

---

## 八、autoDream 的成本設計：背景記憶為什麼不是每次都跑

### 觸發條件的商業邏輯

autoDream 是 Claude Code 的記憶整合系統——在背景把多次對話的記憶整理、去重、更新。但它不是每次對話都跑，有嚴格的觸發條件：

```typescript
// src/services/autoDream/autoDream.ts

const DEFAULTS: AutoDreamConfig = {
  minHours: 24,    // 距離上次整合至少 24 小時
  minSessions: 5,  // 至少 5 個新的對話 session
}
```

為什麼是 24 小時 + 5 次對話？

**成本考量：** 一次 autoDream 要啟動一個完整的子 agent，讀取多個 session 的 transcript，然後呼叫多次 API 做分析和整合。這個過程可能消耗數千到數萬 token。如果每次對話都跑，成本會跟主要對話一樣高。

**品質考量：** 記憶整合需要足夠的新素材。如果只有 1-2 個短對話，整合出來的東西沒有價值。5 個 session 是一個合理的下限——足夠產生有意義的 pattern 和洞察。

**Gate 順序的設計（cheapest first）：**

```typescript
// Gate order (cheapest first):
//   1. Time: hours since lastConsolidatedAt >= minHours (one stat)
//   2. Sessions: transcript count with mtime > lastConsolidatedAt >= minSessions
//   3. Lock: no other process mid-consolidation
```

先檢查時間（只需要讀一個時間戳，成本幾乎為零），再掃描 session 數量（需要列目錄），最後取鎖。**每一道閘的成本都比下一道低**，能擋住的盡早擋，避免浪費。

甚至還有一個 **scan throttle**：即使時間閘通過了但 session 閘沒通過，10 分鐘內不會重新掃描 session 目錄。因為掃描目錄本身也有 I/O 成本。

```typescript
const SESSION_SCAN_INTERVAL_MS = 10 * 60 * 1000  // 10 分鐘掃描節流
```

### 完成後的日誌

```typescript
logEvent('tengu_auto_dream_completed', {
  cache_read: result.totalUsage.cache_read_input_tokens,
  cache_created: result.totalUsage.cache_creation_input_tokens,
  output: result.totalUsage.output_tokens,
  sessions_reviewed: sessionIds.length,
})
```

Anthropic 在追蹤每次 dream 的 cache 使用狀況。如果 cache_read 很高，代表記憶整合在重用已有的快取（好的）；如果 cache_created 很高，代表它在建立大量新快取（貴的）。這些數據用來調整 dream 的觸發頻率和記憶讀取策略。

### 商業啟示

**背景 AI 任務的觸發條件設計，本質上是一個 ROI 計算。** 每次觸發都有成本（token 消耗），每次觸發都有收益（更好的記憶 = 更好的使用者體驗）。觸發太頻繁，成本超過收益；觸發太少，使用者感受不到記憶在進步。

Anthropic 的做法值得學習：

1. **最便宜的檢查先做**（時間 → 數量 → 鎖）
2. **設定合理的觸發閾值**（24h + 5 sessions）
3. **追蹤每次觸發的成本**（cache read/write/output tokens）
4. **用遠端設定動態調整閾值**（GrowthBook）

---

## 九、工具輸入截斷：遙測也要省

### 不只是 API call，遙測也是成本

大多數人只想到 API call 的 token 成本。但 Anthropic 的原始碼提醒我們：**遙測（telemetry）也是成本**。

每次工具呼叫的輸入參數都會被記錄到 telemetry 系統。如果一個 `Read` 工具呼叫讀了一個 50KB 的檔案，而你把完整的檔案路徑和內容都記到遙測裡……

```typescript
// src/services/analytics/metadata.ts

const TOOL_INPUT_STRING_TRUNCATE_AT = 512   // 超過 512 字元的字串
const TOOL_INPUT_STRING_TRUNCATE_TO = 128   // 截斷到 128 字元
const TOOL_INPUT_MAX_JSON_CHARS = 4 * 1024  // 整個 JSON 最多 4KB
const TOOL_INPUT_MAX_COLLECTION_ITEMS = 20  // 陣列最多 20 項
const TOOL_INPUT_MAX_DEPTH = 2              // 巢狀最多 2 層
```

一個五層式的截斷系統：

1. **字串截斷**：超過 512 字元的字串，只保留前 128 字元 + 一個 `[1234 chars]` 標記
2. **JSON 上限**：整個序列化結果超過 4KB 就截斷
3. **陣列裁切**：超過 20 項的陣列只保留前 20 項
4. **深度限制**：超過 2 層的巢狀物件替換為 `<nested>`
5. **內部 key 過濾**：底線開頭的內部標記（如 `_simulatedSedEdit`）不記錄

這些截斷的目的不只是省儲存空間——遙測數據最終會被上傳到分析系統，每一位元都是傳輸成本和儲存成本。

### 商業啟示

**成本控制不只是 API call。** 你的 AI 產品的成本結構裡，可能有一大塊是你沒注意到的：

- 遙測/日誌的儲存和傳輸
- 每次請求的 JSON 序列化開銷
- 錯誤追蹤系統裡的大型 payload
- 長時間 session 的記憶體消耗

Anthropic 在遙測這種「看起來不花錢」的地方都做了五層截斷。你的產品有嗎？

---

## 十、ToolSearch 的延遲載入：不把菜單全印出來

### 問題：40+ 個工具的 schema 有多大？

Claude Code 有 40 多個內建工具，加上使用者的 MCP 工具可能超過 100 個。每個工具的定義（名稱 + 描述 + 輸入 schema）可能是 500-2,000 tokens。100 個工具 = 50,000-200,000 tokens。

**這些工具定義在每次 API call 都要傳送。**

如果你有一個帶 100 個 MCP 工具的設定，光工具定義就佔了 context window 的 10-20%。這不只是成本問題——這是功能問題，因為留給實際對話的空間被壓縮了。

### ToolSearch 的解法：按需載入

```typescript
// src/utils/toolSearch.ts

// 當 MCP 工具定義超過 context window 的 10% 時，自動啟用延遲載入
const DEFAULT_AUTO_TOOL_SEARCH_PERCENTAGE = 10 // 10%
```

工具被分成兩類：

1. **始終載入的核心工具**：Read, Edit, Write, Bash, Grep, Glob 等——高頻使用，schema 不大
2. **延遲載入的工具**：MCP 工具、低頻內建工具——用 `defer_loading: true` 標記，只傳名稱和一行描述

當模型需要一個延遲載入的工具時，它先呼叫 ToolSearchTool 搜尋（用工具名稱或功能關鍵字），搜尋結果回傳 `tool_reference` 區塊，API 在伺服器端把完整的 schema 展開給模型看。

```typescript
// 搜尋結果使用 tool_reference 區塊
return {
  type: 'tool_result',
  tool_use_id: toolUseID,
  content: content.matches.map(name => ({
    type: 'tool_reference',
    tool_name: name,
  })),
}
```

這代表 100 個延遲載入的工具，在 prompt 裡只佔 100 行的名稱列表（約 2,000-3,000 tokens），而不是完整 schema 的 50,000-200,000 tokens。

**省了 95% 以上的工具定義 token。**

### Haiku 不支援

```typescript
// Haiku 模型不支援 tool_reference
const DEFAULT_UNSUPPORTED_MODEL_PATTERNS = ['haiku']
```

這是一個有趣的取捨：最便宜的模型反而要載入所有工具定義（因為它不支援延遲載入的 API feature）。這意味著在 Haiku 上用很多 MCP 工具，每次 call 的 overhead 會很大。

### 商業啟示

**功能擴展和成本控制是一對天然矛盾。** 你的 AI 產品支援越多功能（工具、外掛、API），每次 API call 的 overhead 越大。Anthropic 用延遲載入解決了這個矛盾——使用者可以裝 100 個 MCP 工具，但只有真正用到的才會消耗 token。

這對做 AI 平台（允許外掛）的創業者特別重要：**你的外掛系統需要一個延遲載入機制**，否則外掛越多、API call 越貴、使用者體驗越差。

---

## 總結：AI 產品的成本工程不是省錢，是建立可持續的單位經濟

把所有的成本控制手段整理成一張表：

| 手段 | 節省幅度 | 實現複雜度 | 品質影響 |
|------|---------|-----------|---------|
| Prompt cache | 90%（cache hit 部分） | 中等 | 零 |
| Micro-compact | 10-30%（清舊工具結果） | 低 | 極低 |
| Auto-compact | 50-70%（摘要壓縮） | 高 | 中等 |
| 模型分層 | 3-15x（Haiku vs Opus） | 低 | 取決於任務 |
| 工具延遲載入 | 95%（工具定義部分） | 中等 | 零 |
| 子 agent context 精簡 | 5-15 Gtok/week | 低 | 零 |
| Circuit breaker | 250K calls/day → 0 | 極低（3 行程式碼） | 零 |
| 遙測截斷 | 儲存和傳輸成本 | 低 | N/A |
| 背景任務觸發控制 | 減少無謂觸發 | 低 | 零 |

最後給想做 AI 產品的人三個核心原則：

**原則一：Cache 是你最好的朋友。** 能快取的東西一定要快取。System prompt、工具定義、早期對話——這些在每次 API call 裡佔的比例可能超過 80%。如果你能把 cache hit rate 從 0% 提到 80%，你的 input cost 直接降 72%。

**原則二：每個自動觸發的流程都需要上限和 circuit breaker。** 這不是過度工程，是生存必需。在 AI 產品裡，一個沒有 break condition 的迴圈，一天可以燒掉一個工程師一年的薪水。

**原則三：不是每個 token 都值同樣的錢。** 搜尋用便宜模型、推理用貴模型、快取用最便宜的價格——分層定價不只是 API 提供商的事，也是你作為 AI 產品開發者要做的架構決策。

程式碼不會說謊。Anthropic 的原始碼裡藏著一個清楚的訊息：**AI 產品的護城河不在模型的能力，在你控制成本的能力。** 能力會被追上，但你的成本工程——你的壓縮策略、快取架構、模型分層、circuit breaker——這些是在規模化之後才會顯現差距的東西。

當你的競爭對手還在比誰的 prompt 更長的時候，你應該在比誰的 token 花得更聰明。
