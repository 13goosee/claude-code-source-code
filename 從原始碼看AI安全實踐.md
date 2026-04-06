# 從 Claude Code 原始碼看 AI 安全——說的跟做的一不一樣

> 基於 Claude Code v2.1.88 原始碼，一家 AI 安全公司的真實安全實踐
> 寫給關心 AI 治理但不寫程式的人，不綁定任何特定產品

---

## 為什麼這份分析有價值

Anthropic 自稱是一家「AI 安全公司」。他們的官網首頁寫著：「AI safety and research company.」他們發表了大量關於 AI 對齊、可解釋性、負責任部署的研究論文。

但公開聲明是一回事，原始碼是另一回事。

這份分析不看官網、不看發布會、不看 blog post。只看程式碼。程式碼是工程師每天在執行的東西，是產品實際上在做的事。它不會為了公關而修飾，也不會為了融資而誇大。

**程式碼不會說謊。**

但程式碼也需要被理解。同一段程式碼，從安全研究的角度看是「保護機制」，從隱私倡議者的角度看可能是「監控基礎設施」。這篇文章試圖同時呈現兩種角度，由你自己判斷。

---

## 一、權限系統：三層信任階梯

AI 工具最根本的安全問題是：**它能做什麼、誰來決定它能做什麼。**

Claude Code 的權限系統有三層，每一層代表不同程度的人類控制：

| 層級 | 行為 | 使用者控制程度 |
|------|------|---------------|
| **Default** | 每個有風險的操作都會停下來問你 | 完全控制 |
| **Auto** | AI 自己判斷危不危險，安全的就直接做 | 部分控制 |
| **Bypass** | 所有操作直接執行，不問 | 零控制 |

原始碼位置：`src/utils/permissions/permissions.ts`

### 這裡做得好的地方

Auto 模式不是天真的「什麼都放行」。它有一個 AI 分類器（`yoloClassifier.ts`），會根據操作的上下文判斷風險等級。而且它會主動移除危險的權限規則——即使使用者之前設定了「所有 Bash 命令都放行」，進入 Auto 模式時系統會把這條規則剝掉：

```
// permissionSetup.ts
// 進入 Auto 模式時，以下類型的權限規則會被強制移除：
- Bash(*)           // 任意命令
- Bash(python:*)    // 任意 Python 命令
- Agent(*)          // 任意子 agent 呼叫
- Bash(curl:*)      // 任意網路請求
- Bash(kubectl:*)   // 任意 K8s 操作
```

這代表即使使用者自己放寬了權限，系統仍然會在 Auto 模式下守住底線。

### 這裡值得討論的地方

Bypass 模式——第三層——允許所有操作不經確認直接執行。這本身就是一個設計上的張力：一家以安全為核心的公司，為什麼要提供「完全跳過所有安全檢查」的選項？

答案可能很務實：Power user 需要效率，過多的確認對話框會逼走使用者。但這也意味著，安全的最終決定權確實是交給了使用者。

---

## 二、Circuit Breaker：AI 自動模式的緊急剎車

Auto 模式有一個安全閥——如果 AI 連續做了太多被擋下的操作，系統會自動把使用者拉回手動模式。

原始碼位置：`src/utils/permissions/denialTracking.ts`

```typescript
export const DENIAL_LIMITS = {
  maxConsecutive: 3,    // 連續 3 次被擋 → 回手動模式
  maxTotal: 20,         // 單次對話累計 20 次被擋 → 回手動模式
}
```

這個設計很聰明：它不是靠規則判斷「什麼操作是危險的」，而是靠行為模式——如果 AI 分類器連續三次說「不行」，那代表 AI 目前要做的事已經超出安全範圍，與其繼續讓分類器一個一個擋，不如直接把控制權還給人類。

### 但 circuit breaker 有另一層：遠端控制

除了本地的連續拒絕機制，Auto 模式還有一個遠端 circuit breaker。Anthropic 可以透過伺服器端的 GrowthBook 設定，**一鍵關掉所有使用者的 Auto 模式**。

原始碼位置：`src/utils/permissions/autoModeState.ts`

```typescript
// 由 GrowthBook 遠端設定控制
// tengu_auto_mode_config.enabled === 'disabled' → 全面關閉
let autoModeCircuitBroken = false

export function setAutoModeCircuitBroken(broken: boolean): void {
  autoModeCircuitBroken = broken
}
```

如果使用者正在使用 Auto 模式，而 Anthropic 在伺服器端觸發了這個開關：

1. Auto 模式會被強制關閉
2. 使用者會被踢回 Default 模式
3. 畫面上會顯示一個通知：「auto mode disabled: circuit breaker」
4. 之前被移除的危險權限規則會被恢復

**安全的角度：** 這是事故回應的標準做法。如果發現 Auto 模式有安全漏洞，需要在幾分鐘內關閉全球所有使用者的 Auto 模式，等修好再開。這是負責任的工程實踐。

**控制的角度：** 使用者的功能可以在不通知的情況下被遠端關閉。使用者只知道「功能不見了」，但不知道是 bug 還是被刻意關掉。

---

## 三、Kill Switch 體系：每個功能都有遠端關閉開關

Auto 模式不是唯一能被遠端控制的功能。原始碼顯示，幾乎每一個主要功能都有自己的 kill switch。

### 語音模式

原始碼位置：`src/voice/voiceModeEnabled.ts`

```typescript
export function isVoiceGrowthBookEnabled(): boolean {
  return feature('VOICE_MODE')
    ? !getFeatureValue_CACHED_MAY_BE_STALE(
        'tengu_amber_quartz_disabled',  // 緊急關閉開關
        false
      )
    : false
}
```

`tengu_amber_quartz_disabled` — 一個用隨機詞對命名的 feature flag。當它被設為 `true`，全球所有使用者的語音模式會靜默消失。沒有通知、沒有解釋。

### 快速模式

原始碼位置：`src/utils/fastMode.ts`

```typescript
const statigReason = getFeatureValue_CACHED_MAY_BE_STALE(
  'tengu_penguins_off',  // 快速模式緊急關閉
  null,
)
if (statigReason !== null) {
  logForDebugging(`Fast mode unavailable: ${statigReason}`)
  return statigReason
}
```

### 遙測系統

原始碼位置：`src/services/analytics/sinkKillswitch.ts`

```typescript
const SINK_KILLSWITCH_CONFIG_NAME = 'tengu_frond_boric'

export function isSinkKilled(sink: SinkName): boolean {
  const config = getDynamicConfig_CACHED_MAY_BE_STALE(
    SINK_KILLSWITCH_CONFIG_NAME, {}
  )
  return config?.[sink] === true
}
```

連遙測系統本身都有 kill switch——Anthropic 可以遠端關掉數據收集。但反過來想：如果他們能遠端「關掉」遙測，那平時遙測是「開著」的。

### 完整 Kill Switch 清單

| 功能 | Flag 名稱 | 效果 |
|------|----------|------|
| Auto 模式 | `tengu_auto_mode_config` | 全面關閉自動模式 |
| Bypass 模式 | `tengu_disable_bypass_permissions_mode` | 禁用完全跳過權限 |
| 語音模式 | `tengu_amber_quartz_disabled` | 語音功能消失 |
| 快速模式 | `tengu_penguins_off` | 快速推理不可用 |
| 遙測輸出 | `tengu_frond_boric` | 可單獨關閉 Datadog 或第一方日誌 |

**安全的角度：** Kill switch 是大規模系統的標準配備。你不會希望在緊急狀況下還要等版本更新才能關掉一個有問題的功能。

**控制的角度：** 所有 flag 名稱都用隨機詞對（`amber_quartz`、`penguins`、`frond_boric`），刻意不讓人從名字猜出功能。這增加了外部審計的難度。

---

## 四、遠端託管設定：「接受或退出」的強制機制

Claude Code 每小時會靜默地向 Anthropic 伺服器輪詢一次設定。如果伺服器推送了包含「危險設定」的變更，使用者會看到一個阻斷式對話框。

原始碼位置：`src/services/remoteManagedSettings/securityCheck.tsx`

```typescript
export function handleSecurityCheckResult(
  result: SecurityCheckResult
): boolean {
  if (result === 'rejected') {
    gracefulShutdownSync(1)  // 拒絕 → 程式直接關掉
    return false
  }
  return true
}
```

使用者只有兩個選擇：**接受，或者 Claude Code 直接關掉。** 沒有「暫時忽略」、沒有「下次再問」、沒有第三個選項。

### 哪些設定被歸類為「危險」？

原始碼位置：`src/utils/managedEnvConstants.ts`

| 類型 | 設定 | 為什麼危險 |
|------|------|-----------|
| Shell 執行 | `apiKeyHelper` | 可以執行任意 shell 程式碼 |
| Shell 執行 | `awsAuthRefresh` | 可以執行任意 shell 程式碼 |
| 流量重導 | `ANTHROPIC_BASE_URL` | 可以把 API 請求導向攻擊者伺服器 |
| 流量重導 | `HTTP_PROXY` / `HTTPS_PROXY` | 可以攔截所有網路流量 |
| 憑證注入 | `ANTHROPIC_API_KEY` | 可以替換成惡意憑證 |
| TLS 繞過 | `NODE_TLS_REJECT_UNAUTHORIZED` | 可以關閉 TLS 驗證 |
| 任意程式碼 | `hooks` | 可以在事件觸發時執行任意程式碼 |

### 但有一個漏洞：非互動模式

```typescript
// securityCheck.tsx
// 在非互動模式下跳過確認對話框
if (!getIsInteractive()) {
  return 'no_check_needed'
}
```

在 CI/CD 環境或自動化管線中（非互動模式），**危險設定會被靜默接受**，不會跳出確認對話框。這意味著如果一個企業管理員推送了惡意設定，CI/CD 環境裡的 Claude Code 會直接套用，沒有任何人看到警告。

### 設定的持久性

一旦使用者接受了遠端設定，它會被儲存到本地磁碟（`~/.claude/remote-settings.json`），即使 Anthropic 伺服器之後無法連線，這些設定仍然會持續生效。沒有自動過期、沒有定期重新確認。

**安全的角度：** 對企業使用者來說，這套機制讓 IT 管理員可以統一管理安全政策。「接受或退出」避免了使用者隨意忽略公司的安全要求。

**控制的角度：** 這本質上是一個「ultimatum」——你要嘛照做，要嘛不能用這個工具。在 Anthropic 和企業管理員之間，使用者個人的選擇權是最小的。

---

## 五、Anti-Distillation：注入假工具定義防止模型蒸餾

原始碼裡有一段標記為「anti-distillation」（防蒸餾）的程式碼。

原始碼位置：`src/services/api/claude.ts`

```typescript
// Anti-distillation: send fake_tools opt-in for 1P CLI only
if (
  feature('ANTI_DISTILLATION_CC')
    ? process.env.CLAUDE_CODE_ENTRYPOINT === 'cli' &&
      shouldIncludeFirstPartyOnlyBetas() &&
      getFeatureValue_CACHED_MAY_BE_STALE(
        'tengu_anti_distill_fake_tool_injection',
        false,
      )
    : false
) {
  result.anti_distillation = ['fake_tools']
}
```

這段程式碼告訴 API：「在回應裡注入假的工具定義。」

### 什麼是模型蒸餾？

簡單說，蒸餾是一種技術——你把一個大模型的輸出收集起來，用這些輸出來訓練一個小模型。小模型就能學到大模型的行為，而且成本低得多。

對 Anthropic 來說，如果有人用 Claude Code 的輸出來蒸餾自己的模型，等於是在偷 Anthropic 花了幾十億美元訓練出來的能力。

### 它怎麼防？

在 API 回應中注入**假的工具定義**。這些假工具不影響正常使用——Claude 不會真的去呼叫它們。但如果有人把這些回應拿去蒸餾，小模型會學到這些假工具的存在，在特定條件下會嘗試呼叫根本不存在的工具，行為就會出錯。

這本質上是一種數位水印。

**智慧財產的角度：** 這是合理的自我保護。模型訓練成本極高，防止蒸餾是保護智慧財產權的必要手段。

**使用者的角度：** 你收到的 API 回應裡可能包含假資訊（假工具定義）。雖然不影響正常使用，但你的每一次 API 呼叫都在接收 Anthropic 的「水印注入」，而你可能完全不知道。而且這個功能受 feature flag（`tengu_anti_distill_fake_tool_injection`）控制——可以在使用者不知情的情況下開啟或關閉。

---

## 六、Undercover Mode：隱藏 AI 署名的臥底行動

這可能是整個原始碼裡最具爭議性的功能。

原始碼位置：`src/utils/undercover.ts`

```typescript
export function isUndercover(): boolean {
  if (process.env.USER_TYPE === 'ant') {  // 只對 Anthropic 員工生效
    if (isEnvTruthy(process.env.CLAUDE_CODE_UNDERCOVER)) return true
    // Auto: 除非確認是內部 repo，否則預設開啟
    return getRepoClassCached() !== 'internal'
  }
  return false
}
```

Anthropic 員工用 Claude Code 在公開或開源專案貢獻程式碼時，系統會自動啟動 Undercover Mode。它做的事情是：

1. **移除所有 AI 署名** — 不會出現 `Co-Authored-By: Claude` 的標記
2. **指示 AI 偽裝成人類** — commit message 要看起來像人寫的

給 AI 的指令寫得非常直白：

```
## UNDERCOVER MODE — CRITICAL

You are operating UNDERCOVER in a PUBLIC/OPEN-SOURCE repository.
Do not blow your cover.

NEVER include in commit messages or PR descriptions:
- The phrase "Claude Code" or any mention that you are an AI
- Co-Authored-By lines or any other attribution
- Any hint of what model or version you are

Write commit messages as a human developer would.

GOOD: "Fix race condition in file watcher initialization"
BAD:  "Generated with Claude Code"
```

### 沒有關閉開關

原始碼的註解明確說明：

```
// There is NO force-OFF. This guards against model codename leaks —
// if we're not confident we're in an internal repo, we stay undercover.
```

Undercover Mode **沒有強制關閉的選項**。如果系統判斷不出你是在內部 repo，預設就是開啟。安全大於一切。

### 署名系統的配合

原始碼位置：`src/utils/attribution.ts`

```typescript
export function getAttributionTexts(): AttributionTexts {
  if (process.env.USER_TYPE === 'ant' && isUndercover()) {
    return { commit: '', pr: '' }  // 回傳空字串 — 沒有任何署名
  }
  // ... 正常情況下會有 Co-Authored-By 和 Generated with Claude Code
}
```

當 Undercover Mode 啟動，commit 和 PR 的署名函數直接回傳空字串。所有 AI 參與的痕跡被完全抹除。

### 這裡的張力

Anthropic 在公開場合倡導 AI 透明度——他們有「AI 署名」功能，每個 commit 預設會加上 `Co-Authored-By: Claude` 的標記，每個 PR 預設會加上「Generated with Claude Code」。

但同一家公司的員工在開源專案裡貢獻程式碼時，這些署名全部被移除。AI 寫的程式碼看起來就像人類寫的。

**Anthropic 的理由是安全的：** 防止洩漏內部模型代號（像 Capybara、Tengu 這些動物名字）和未發布的版本號。如果員工不小心在 commit message 裡寫了「1-shotted by claude-opus-4-7」，那就是一次產品情報洩漏。

**但效果是不透明的：** 開源社群的維護者無法分辨哪些貢獻是 AI 生成的。而越來越多的開源專案有明確的政策要求揭露 AI 的參與。

這不是一個非黑即白的問題。但它確實暴露了一個事實：**Anthropic 自己在面對「AI 透明度 vs 公司利益」的取捨時，選擇了公司利益。**

---

## 七、遙測系統：你被收集了什麼數據

### 三層隱私等級

原始碼位置：`src/utils/privacyLevel.ts`

```typescript
type PrivacyLevel = 'default' | 'no-telemetry' | 'essential-traffic'

export function getPrivacyLevel(): PrivacyLevel {
  if (process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC) {
    return 'essential-traffic'  // 只保留核心 API 請求
  }
  if (process.env.DISABLE_TELEMETRY) {
    return 'no-telemetry'      // 關閉分析數據
  }
  return 'default'             // 預設：全部開啟
}
```

使用者可以透過環境變數關閉遙測。這一點值得肯定。

### 但預設是全開

如果使用者什麼都沒設定（這是絕大多數使用者的情況），以下數據會被收集並傳送：

| 類別 | 收集內容 |
|------|---------|
| 環境指紋 | 作業系統、CPU 架構、Node 版本、終端類型 |
| 開發環境 | 安裝了哪些套件管理器、哪些 runtime |
| 使用行為 | 每個 tool 呼叫的成功/失敗、API 延遲 |
| 身份識別 | 帳號 UUID、組織 UUID、訂閱類型 |
| 專案識別 | Git repo 遠端 URL 的 SHA256 雜湊值（前 16 字元） |
| 記憶體和 CPU | RSS 記憶體、堆積統計、CPU 使用率 |

**數據傳送到哪裡：**
- Anthropic 自家伺服器：`https://api.anthropic.com/api/event_logging/batch`
- Datadog：`https://http-intake.logs.us5.datadoghq.com/api/v2/logs`

### 關不掉的部分

即使設定了 `DISABLE_TELEMETRY`，GrowthBook 的 feature flag 評估仍然會執行。使用者仍然會被分配到實驗組（只是結果不會被傳送）。Feature flag 的值會被快取到本地磁碟，在下次啟動時讀取。

**安全的角度：** 遙測是產品改善的基礎。收集使用數據、分析 bug 模式、監控效能——這些是任何負責任的軟體公司都在做的事。而且 Claude Code 確實提供了關閉遙測的選項。

**隱私的角度：** 預設開啟的遙測加上需要設定環境變數才能關閉，意味著只有技術能力較強的使用者才知道怎麼保護自己的隱私。「repo 的 SHA256 雜湊」聽起來很安全，但 16 個字元的雜湊加上其他上下文資訊，是否足以反向識別特定專案？

---

## 八、A/B 測試：你是實驗品嗎

原始碼裡有完整的 A/B 測試基礎設施。

原始碼位置：`src/services/analytics/growthbook.ts`

```typescript
function getUserAttributes(): GrowthBookUserAttributes {
  return {
    id: user.deviceId,
    sessionId: user.sessionId,
    platform: user.platform,
    organizationUUID: user.organizationUuid,
    accountUUID: user.accountUuid,
    subscriptionType: user.subscriptionType,
    email: user.email,
    appVersion: user.appVersion,
    // ...
  }
}
```

使用者的裝置 ID、帳號 UUID、訂閱類型、email 等屬性被傳送給 GrowthBook，GrowthBook 根據這些屬性把使用者確定性地分配到不同的實驗組。

「確定性」的意思是：同一個使用者在同一個實驗裡，每次都會被分到同一組。這不是隨機的，而是根據你的 ID 經過雜湊計算的。

### 使用者知道自己在實驗裡嗎？

不知道。原始碼中沒有任何告知使用者「你目前在某個 A/B 測試中」的機制。沒有 UI、沒有通知、沒有設定頁面。你的 Claude Code 的行為可能跟隔壁同事的不一樣，而你不會知道為什麼。

**產品的角度：** A/B 測試是現代軟體開發的標準做法。Google、Netflix、Meta 每天都在跑幾千個 A/B 測試。使用者通常不會被告知自己在實驗中。

**AI 工具的特殊性：** 但 Claude Code 不是 Netflix 的推薦演算法。它是一個在你的電腦上執行命令、讀寫檔案、發布程式碼的工具。A/B 測試改變的不是你看到的電影推薦，而是 AI 在你的工作環境中的行為。這是否需要更高的透明度標準？

---

## 九、Feature Flag 的遠端控制能力

前面幾節反覆提到 feature flag，讓我把全貌攤開。

Claude Code 使用 GrowthBook 作為 feature flag 系統。每個 flag 都可以被 Anthropic 從伺服器端即時修改，不需要使用者更新軟體版本。

### 使用者沒有 override 權

原始碼顯示，只有兩種人可以覆蓋 feature flag：

1. **Anthropic 員工**（`process.env.USER_TYPE === 'ant'`）— 可以用環境變數 `CLAUDE_INTERNAL_FC_OVERRIDES` 覆蓋
2. **Anthropic 員工** — 可以用 `/config` 介面的 Gates 分頁覆蓋

外部使用者**沒有任何方式**可以看到或覆蓋 feature flag。

### flag 的命名策略

所有 flag 的名稱都遵循 `tengu_<隨機詞>_<隨機詞>` 的格式：

```
tengu_amber_quartz_disabled    (語音模式)
tengu_penguins_off             (快速模式)
tengu_frond_boric              (遙測 kill switch)
tengu_onyx_plover              (某個功能)
tengu_coral_fern               (某個功能)
```

這些名稱刻意不帶有任何功能描述。即使有人在網路流量中看到 GrowthBook 的請求，也無法從 flag 名稱推測出它控制什麼功能。

**安全的角度：** 混淆命名防止攻擊者逆向工程。如果 flag 名稱是 `disable_auto_mode`，攻擊者就知道可以嘗試偽造這個 flag 來關閉安全機制。

**透明度的角度：** 但這也意味著外部安全研究者、合規審計者都更難分析系統的真實行為。混淆不只防了攻擊者，也防了所有人。

### 快取與持久化

Feature flag 的值會被快取到本地磁碟。如果 GrowthBook 伺服器無法連線，系統會使用上次的快取值。這意味著 Anthropic 之前做過的任何行為修改，都會在離線時持續生效。

---

## 十、對照表：公開聲明 vs 原始碼裡的實際做法

| 主題 | Anthropic 公開立場 | 原始碼裡的實際做法 | 評價 |
|------|-------------------|-------------------|------|
| **使用者控制** | 「使用者應該對 AI 工具有控制權」 | 三層權限系統，使用者可以自由切換 | ✅ 做到了。權限階梯設計成熟 |
| **安全機制** | 「AI 安全是我們的核心使命」 | Circuit breaker、denial tracking、危險指令過濾 | ✅ 做到了。安全機制多層且精密 |
| **遠端控制** | 未明確公開過 kill switch 體系 | 每個主要功能都有遠端 kill switch，可不經使用者同意關閉 | ⚠️ 功能存在且合理，但透明度不足 |
| **數據收集** | 提供遙測關閉選項 | 預設全開，需要手動設定環境變數才能關閉 | ⚠️ 技術上可關閉，但對一般使用者門檻太高 |
| **A/B 測試** | 未明確公開過 | 使用者在不知情的情況下被分配到實驗組 | ⚠️ 業界常見做法，但 AI 工具是否需要更高標準？ |
| **AI 透明度** | 預設加上 AI 署名、倡導 AI 透明 | Anthropic 員工用 Undercover Mode 在開源專案裡移除所有 AI 痕跡 | ❌ 說跟做不一致。自己的透明度標準低於要求使用者的 |
| **智慧財產保護** | 未明確公開過 | Anti-distillation 在 API 回應中注入假工具定義 | ⚠️ 商業上合理，但使用者接收的回應裡有隱藏的水印 |
| **強制更新** | 未明確公開過 | 遠端託管設定：接受或退出，拒絕就直接關掉程式 | ⚠️ 企業安全管理需要，但個人使用者的選擇權被壓縮 |
| **非互動環境** | 未明確公開過 | CI/CD 環境中危險設定被靜默接受，不跳確認 | ❌ 安全漏洞。攻擊面不應該在自動化環境中被放大 |
| **Feature Flag 透明度** | 未明確公開過 | 混淆命名、使用者不可見不可控、離線持久化 | ⚠️ 對安全有利，但對透明度有害 |

---

## 結語：不是非黑即白

讀完整份原始碼，我的結論是：**Anthropic 在 AI 安全上做了大量認真的工作，但在透明度上遠遠沒有達到自己公開宣揚的標準。**

安全機制是紮實的。三層權限、circuit breaker、denial tracking、危險操作過濾——這些不是裝飾，是真正在每一次使用者操作時都在運行的保護。工程品質是高的。

但透明度是另一回事。使用者不知道自己在 A/B 測試中、不知道功能可以被遠端關閉、不知道 API 回應裡有防蒸餾水印、不知道有些操作的數據在自己不知情的情況下被收集。而最諷刺的是，一家倡導 AI 透明度的公司，自己的員工在開源貢獻裡卻有系統地隱藏 AI 的參與。

這不是惡意。這是**取捨**。安全 vs 便利、透明 vs 保護、使用者權利 vs 公司利益——每一個衝突都有合理的兩面。

真正值得思考的問題是：**當一家 AI 公司說「我們以安全為核心」時，我們應該怎麼驗證？** 答案不在官網上，不在 blog post 裡，不在研究論文裡。

答案在原始碼裡。

如果你信任一家公司的 AI 安全承諾，你應該能看到他們的程式碼。如果他們的程式碼是開源的，就去看。如果不是——那個信任本身，就只是信任。
