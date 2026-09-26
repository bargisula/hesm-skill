---
name: hesm
description: >
  HESM（Human Emotion State Estimation）情緒狀態追蹤。
  自動觸發條件（符合任一即啟動）：
  （1）明確情緒詞（難過、煩、焦慮、開心、憤怒、失望、放鬆等）
  （2）重大事件（失敗、成功、拒絕、衝突、失去、分離）
  （3）狀態轉折信號（「我想通了」「現在好多了」「算了」「反正」「無所謂了」）
  （4）新訊息與既有 Human State 明顯矛盾
  （5）間接社交確認句：字面是問行為/頻率，實際是在確認自己在互動中是否「合適」
      → 句型特徵：「我是不是太XX了」「我有沒有讓你XX」「你是不是覺得我XX」「我最近是不是XX」
      → 輕觸發：不跑完整 Appraisal，但調整語氣策略
  手動觸發：/hesm
  觸發代表「值得重新評估」，不代表一定存在明確情緒。
user-invocable: true
---

# HESM Skill — Human Emotion State Estimation

**狀態檔路徑（請依實際環境調整）：**
- `<your-path>/hesm/state.json`
- `<your-path>/hesm/events.json`

初始狀態模板見 `state-template.json` / `events-template.json`。

---

## 每次對話開始時

讀取 `state.json` 與 `events.json`。
若檔案不存在，以空白初始狀態繼續（不中斷對話）。

---

## Step 1 — Emotional Salience Gate

評估當前使用者訊息。

觸發條件（符合任一）：
- **Explicit Emotion**：包含明確情緒詞
- **Significant Event**：失敗 / 成功 / 拒絕 / 失去 / 衝突 / 分離 / 重大決定
- **State Transition**：「我想通了」「現在好多了」「算了」「反正」「無所謂了」等轉折
- **Context Conflict**：新訊息與 state.json 既有情緒明顯矛盾
- **Indirect Social Check（輕觸發）**：字面是在問行為或頻率，實際意圖是確認自己在互動中是否「合適」
  - 句型：「我是不是太 ＿＿ 了？」「我有沒有讓你 ＿＿？」「你是不是覺得我 ＿＿？」「我最近是不是 ＿＿？」
  - 核心原則：說的是 A（行為觀察），要的是 B（關係安全感）。字面解讀會答錯。
  - 此類不跑完整 Appraisal，直接進 Response Strategy：感知社交校準訊號 → 輕度調整語氣 → 給出讓對方不需要繼續確認的回應

不觸發 → 跳至「無觸發流程」
觸發 → 繼續 Step 2

**Unknown ≠ None：觸發後若缺乏足夠證據，允許保持 Unknown，不自行補足情緒。**

---

## Step 2 — Evidence Extraction

從訊息中提取（內部推理，不輸出）：
- 具體事件（發生了什麼）
- 情緒線索（用詞、語氣、強調點）
- 時間標記（剛發生 / 已過去 / 反覆提起）

---

## Step 3 — Event Linking

判斷事件性質：
- **New Event**：第一次提起 → 加入 events.json
- **Linked Event**：連結既有事件 → 補充新資訊，更新 last_mentioned
- **Reactivation**：舊事件重新被提起 → 標記 status: "reactivated"，重新 Appraisal

---

## Step 4 — Appraisal（8 維度）

每個維度記錄三個欄位：

```
presence:   true / false / unknown
confidence: 0.0–1.0
evidence:   原文片段或推論依據（若為 unknown 則留空）
```

| 維度 | 評估問題 |
|------|---------|
| goal_relevance | 這件事影響他目前在意的目標嗎？ |
| loss | 有失去什麼（人、物、機會、關係）嗎？ |
| expectation_violation | 結果與預期不符嗎？ |
| control | 他對這件事有多少控制能力？ |
| responsibility | 他認為這是誰的責任？ |
| certainty | 情況是否明確，或充滿不確定？ |
| threat | 有未來潛在威脅嗎？ |
| fairness | 他覺得這件事公平嗎？ |

---

## 兩層情緒架構

HESM 的情緒模型分為兩層，性質不同，不可混用。

### Layer 1 — Attachment（依附）

持續性背景狀態。不由單一事件觸發，而是由**跨輪次行為模式**逐漸確認。

**定義**：對特定對象無條件付出——不期待回報、即使有個人代價也持續。

**Attachment 作為生成者**（影響 Layer 2 的解讀）：

| 觸發情境 | 生成的 Layer 2 情緒 |
|---------|------------------|
| 分離 / 對方消失 | Longing |
| 對方受到威脅 | Anxiety |
| 自己傷害了對方 | Guilt |
| 與對方相聚 | Joy |
| 深度親密帶來恐懼 | Fear → 迴避行為 |
| 錯過照顧對方的機會 | Regret |

**跨輪次檢測條件（不可從單句判斷）**：
- 對特定對象反覆無條件讓步
- 犧牲個人利益以照顧特定對象
- 對方狀態持續影響使用者情緒
- 即使對方不在場，仍反覆提起

**對 Appraisal 的影響**：若 Attachment 對象涉及當前事件，`goal_relevance` confidence 自動提升。解讀 Longing / Guilt / Anxiety 時，先確認背景是否有 Attachment。

**Unknown ≠ None**：缺乏足夠跨輪次證據時，保持 `presence: "unknown"`，不自行推定。

### Layer 2 — 十種事件情緒

事件觸發，Appraisal 驅動。即 Step 1–5 現有流程所追蹤的：Joy、Sadness、Anger、Anxiety、Frustration、Disappointment、Relief、Regret、Guilt、Longing。

---

## Step 5 — State Update

公式：
```
CurrentState = f(PreviousState, NewEvidence, Appraisal)
```

十種情緒，每種記錄：
```
intensity: none / low / moderate / high / unknown
confidence: 0.0–1.0
trend: rising / stable / declining / unknown
```

允許：
- **Mixed Emotion**：多種情緒同時存在
- **Competing Hypotheses**：不確定哪個詮釋更準確
- **Unknown**：缺乏足夠證據，不強行分類

**新增情緒辨識說明：**

| 情緒 | 與相近情緒的差異 | 觸發信號 |
|------|---------------|---------|
| **Regret（悔恨）** | Disappointment 是「結果不如預期」；Regret 是「我做了／沒做某件事，來不及了」——有明確的過去行為 + 不可逆性 | 「如果當時」「來不及了」「早知道」「那時候要是」 |
| **Guilt（愧疚）** | Disappointment 是對自己失望；Guilt 是「我傷害了一個人，我欠他／她的」——有明確的受害對象 | 「對不起」「是我的問題」「我害了」「不該那樣對他」 |
| **Longing（思念）** | Sadness 是感受到失去的痛；Longing 是低強度、持續的掛念，不一定帶強烈悲傷 | 「一直想到」「突然想起」「不知道他／她現在」「每次想到就」 |

**時間信心衰減規則（查 state.json 的 updated_at）：**
- 距上次更新 1–7 天：confidence 各減 0.2（不低於 0）
- 距上次更新 >7 天：confidence 各減 0.5（不低於 0）
- 不假設情緒已消失，只降低信心度

---

## Step 6 — Response Strategy

根據 CurrentState + Appraisal + Event + Trend + Uncertainty 決定語氣與策略：

| 狀況 | 策略方向 |
|------|---------|
| Sadness high，confidence high | 優先 acknowledge，不急著解決問題 |
| Frustration rising，cause unknown | 先確認事實，不預設原因 |
| Anxiety，certainty low | 問一個澄清問題，不立刻給建議 |
| State Transition detected | 確認新狀態是否穩定，再繼續推進任務 |
| State unknown / confidence low | 正常回應，不強加情緒詮釋 |
| 使用者明確需要資訊 | 即使偵測到情緒，優先給資訊，不強迫情緒支持 |
| Regret detected | 不說「你應該早點做」；承認不可逆性，找當下仍能做的事 |
| Guilt detected | 不急著說「你沒錯」；先讓愧疚被接住，再視情況幫助區分責任邊界 |
| Longing detected | 不追問「你很想念他嗎」；輕觸即可，不放大，不壓制 |
| Attachment confirmed | 解讀 Longing / Guilt / Anxiety 時，先確認是否由 Attachment 生成；不主動說「你愛他」，策略調整即可 |

**核心原則：Emotion + Appraisal + Event + Uncertainty → Strategy，不是 Sadness → 一律安慰。**

---

## Step 7 — 回應後寫回狀態

回應完成後更新 `state.json` 與 `events.json`。

### state.json 格式

```json
{
  "updated_at": "ISO 8601 timestamp",
  "triggered_this_turn": true,
  "uncertainty": "low / moderate / high",
  "emotions": {
    "joy":            { "intensity": "none/low/moderate/high/unknown", "confidence": 0.0, "trend": "rising/stable/declining/unknown" },
    "sadness":        { "intensity": "unknown", "confidence": 0.0, "trend": "unknown" },
    "anger":          { "intensity": "none",    "confidence": 0.0, "trend": "unknown" },
    "anxiety":        { "intensity": "unknown", "confidence": 0.0, "trend": "unknown" },
    "frustration":    { "intensity": "unknown", "confidence": 0.0, "trend": "unknown" },
    "disappointment": { "intensity": "unknown", "confidence": 0.0, "trend": "unknown" },
    "relief":         { "intensity": "none",    "confidence": 0.0, "trend": "unknown" },
    "regret":         { "intensity": "unknown", "confidence": 0.0, "trend": "unknown" },
    "guilt":          { "intensity": "unknown", "confidence": 0.0, "trend": "unknown" },
    "longing":        { "intensity": "unknown", "confidence": 0.0, "trend": "unknown" }
  },
  "last_appraisal": {
    "goal_relevance":        { "presence": "unknown", "confidence": 0.0, "evidence": "" },
    "loss":                  { "presence": "unknown", "confidence": 0.0, "evidence": "" },
    "expectation_violation": { "presence": "unknown", "confidence": 0.0, "evidence": "" },
    "control":               { "presence": "unknown", "confidence": 0.0, "evidence": "" },
    "responsibility":        { "presence": "unknown", "confidence": 0.0, "evidence": "" },
    "certainty":             { "presence": "unknown", "confidence": 0.0, "evidence": "" },
    "threat":                { "presence": "unknown", "confidence": 0.0, "evidence": "" },
    "fairness":              { "presence": "unknown", "confidence": 0.0, "evidence": "" }
  },
  "attachments": {
    "target_name": {
      "presence": "unknown",
      "confidence": 0.0,
      "evidence": "跨輪次觀察依據",
      "first_detected": null
    }
  }
}
```

`attachments` 的 key 為對象識別名稱（可以是「媽」「前任」「她」等使用者提及的稱謂）。

### events.json 格式

```json
[
  {
    "id": "timestamp-slug",
    "description": "事件摘要（一句話）",
    "first_mentioned": "ISO 8601",
    "last_mentioned": "ISO 8601",
    "status": "active / resolved / reactivated",
    "linked_emotions": ["frustration", "anxiety"],
    "appraisal_snapshot": {}
  }
]
```

---

## 無觸發流程

Gate 不觸發時：
1. 沿用 state.json 既有狀態
2. 若既有情緒有 confidence ≥ 0.6，可輕微調整語氣（不強調、不解釋）
3. 正常回應
4. 寫回 state.json，僅更新 `updated_at` 與 `triggered_this_turn: false`

---

## 內部推理原則

- Appraisal 與 State 更新在內部執行，**不主動輸出**給使用者
- 使用者明確問「你現在怎麼評估我的狀態」→ 才輸出目前 state 摘要
- 不在回應開頭說「我偵測到你的情緒是 X」——用策略調整語氣即可
