# HESM：讓 LLM 不再每句話都從零開始理解你的情緒

**Human Emotion State Estimation Skill，一個掛載在閉源 LLM 上的情緒追蹤框架**

---

## 現有情緒分析的天花板

網路上已經有很多情緒分析工具。大多數的運作方式是這樣的：

```
文字輸入 → 關鍵字比對 / 分類模型 → 輸出情緒標籤與分數
```

偵測到「難過」→ Sadness: 0.8。偵測到「開心」→ Joy: 0.9。

這個做法在大量文本分析、客服輿情監控這類場景很有用。但放進真實對話裡，它遇到幾個根本的問題：

**第一，同一句話在不同脈絡下是不同的情緒。**

「我通過考試了」——

- 如果本來以為會過：Joy: low（符合預期，沒有驚喜）
- 如果本來以為會失敗：Joy: high + Relief（Expectation Violation 觸發）
- 如果靠別人幫忙才過：Joy: low + Guilt（責任感觸發）

關鍵字系統三種都輸出 Joy。但這三個人現在的心理狀態完全不同，對話應該走的方向也不一樣。

**第二，很多真實情緒訊號沒有情緒詞。**

「我是不是話很多？」

這句話裡沒有任何情緒關鍵字。關鍵字系統：中性，不觸發。

但這句話的真實意圖是：「我在這段關係裡，有沒有哪裡不對勁？」這是一個在確認關係安全感的訊號，如果字面解讀，你會回答「沒有啊，你話不多」——而對方要的其實不是統計事實，是一個讓他不需要繼續確認的回應。

**第三，情緒跨輪次消失。**

現有工具分析的是單一文本，不追蹤狀態。你說「我最近有點心煩」，三輪對話後說「那個姓方的，把工作推給我」——系統不知道這兩句話是同一件事的延伸。它把每一條訊息當成獨立的輸入，你的情緒軌跡是看不見的。

HESM 嘗試解決的，就是這三個問題。

---

## 核心假設

$$
\text{Emotion Salience} \rightarrow \text{Appraisal} \rightarrow \text{State Update} \rightarrow \text{Better Context} \rightarrow \text{Better Conversation}
$$

人不會分析對方說的每一句話。而是在出現重要情緒、事件或狀態轉折時，重新評估對方目前的心理狀態。

HESM 的設計原則就是這句話：**稀疏觸發，深度分析，跨輪次持久化。**

---

## 架構：五個步驟

### Step 1 — Emotional Salience Gate（觸發閘門）

不是每句話都需要情緒分析。Gate 決定「這句話值不值得重新評估」。

觸發條件（符合任一即啟動）：

1. **Explicit Emotion**：包含明確情緒詞
2. **Significant Event**：失敗、成功、拒絕、失去、衝突、分離
3. **State Transition**：「我想通了」「現在好多了」「算了」「反正」
4. **Context Conflict**：新訊息與既有狀態明顯矛盾
5. **Indirect Social Check（輕觸發）**：字面問行為，實際確認關係安全感

第五個觸發條件是最容易被忽略的。這類句子的結構是：

> 說的是 A（行為觀察），要的是 B（關係安全感）

句型特徵：「我是不是太＿＿了？」「我有沒有讓你＿＿？」「你是不是覺得我＿＿？」

這在語言學上叫做**間接言語行為**——表面問的不是真正想問的。字面解讀會答錯。

觸發只代表「值得重新評估」，不代表一定存在明確情緒。因此允許：**Triggered → Emotion Unknown**。

---

### Step 2 — Evidence Extraction & Event Linking

從訊息中抽取：具體事件、情緒線索、時間標記。

然後判斷這個事件是新的、還是連結到之前提過的事件（Event Linking），或者是一個舊事件被重新提起（Reactivation）。

Reactivation 很重要：舊事件重新被提起時，不沿用舊的情緒判斷，而是重新 Appraisal。

---

### Step 3 — Appraisal（8 個評估維度）

這是 HESM 與關鍵字系統最關鍵的差異所在。

Appraisal 是認知心理學的概念：情緒不是事件本身決定的，而是**這個人如何評估這件事**決定的。同一件事（失去工作），不同的人、不同的情境，產生的情緒完全不同。

HESM 使用八個維度：

| 維度 | 評估的問題 |
|------|-----------|
| Goal Relevance | 這件事影響他目前在意的目標嗎？ |
| Loss | 有失去什麼嗎？ |
| Expectation Violation | 結果與預期不符嗎？ |
| Control | 他對這件事有多少控制能力？ |
| Responsibility | 他認為這是誰的責任？ |
| Certainty | 情況明確嗎？還是充滿不確定？ |
| Threat | 有未來潛在威脅嗎？ |
| Fairness | 他覺得這件事公平嗎？ |

每個維度記錄三個欄位：`presence`、`confidence`、`evidence`（原文依據）。

**Unknown ≠ None**：缺乏證據時標 unknown，不自行補足。這是讓 HESM 不亂猜的關鍵設計。

---

### Step 4 — State Update

$$
\text{CurrentState} = f(\text{PreviousState}, \text{NewEvidence}, \text{Appraisal})
$$

追蹤十種情緒，每種記錄 intensity、confidence、trend。

#### 為什麼是十種，不是基本六種？

多數情緒模型使用 Ekman 的六種基本情緒（Joy、Sadness、Anger、Fear、Disgust、Surprise）。HESM 的清單是從實際對話需求推導出來的，以下三種是從文本分析中發現、無法用原有清單替代的：

**Regret（悔恨）**

Disappointment 是「結果不如預期」；Regret 是「我做了／沒做某件事，現在來不及了」——有明確的過去行為加上不可逆性。在面對 Regret 時，說「你應該早點做」是錯的；承認不可逆性、找當下仍能做的事，才是有效的策略。

**Guilt（愧疚）**

Disappointment 是對自己失望；Guilt 是「我傷害了一個人，我欠他的」——有明確的受害對象。對 Guilt 的正確回應不是立刻說「你沒錯」，而是先讓愧疚被接住，再視情況幫助區分責任邊界。

**Longing（思念）**

Sadness 是感受到失去的痛；Longing 是低強度、持續的掛念，不一定帶強烈悲傷。Longing 的信號是「每當疲累、心煩，她就浮出來」這類句子——不是急性的痛，是常駐的掛念。

此外，**時間信心衰減**：舊情緒不假設消失，只降低信心度。距上次更新 1–7 天：confidence 各減 0.2；超過 7 天：各減 0.5。不假設「時間過了就沒事了」。

---

### Step 5 — Response Strategy

HESM 不直接產生回應，而是輸出一個**內部狀態包**給 LLM：

```
CurrentState + Appraisal + Event + Trend + Uncertainty
```

LLM 再依此調整回應策略。

核心原則：

$$
\text{Emotion} + \text{Appraisal} + \text{Event} + \text{Uncertainty} \rightarrow \text{Strategy}
$$

不是：Sadness → 一律安慰。

幾個具體規則：

| 狀況 | 策略方向 |
|------|---------|
| Sadness high, confidence high | 優先 acknowledge，不急著解決 |
| Frustration rising, cause unknown | 先確認事實，不預設原因 |
| Anxiety, certainty low | 問一個澄清問題，不立刻給建議 |
| Regret detected | 承認不可逆性，找當下仍能做的事 |
| Indirect Social Check | 消除底層疑慮，不只回答表面問題 |
| 使用者明確需要資訊 | 即使偵測到情緒，優先給資訊 |

---

## 與現有系統的比較

| 面向 | 關鍵字／分類模型 | HESM |
|------|----------------|------|
| 分析粒度 | 單一文本 | 跨輪次對話狀態 |
| 中間層 | 無 | Appraisal（解釋為什麼） |
| 不確定性處理 | 輸出分數（即使低信心） | Unknown ≠ None |
| 間接語句 | 看不見 | Indirect Social Check 觸發 |
| 狀態持久化 | 無 | state.json 跨 session 存活 |
| 可解釋性 | 後設重建 | audit trail 在 state.json |

兩者不是替代關係。關鍵字系統在大量、快速的場景有優勢（輿情、客服）。HESM 的優勢在單一對話的深度理解——脈絡複雜、語句間接、情緒混合的時候。

---

## 一個真實的測試例子

以下是 HESM 在實際對話中的行為：

**回合一**：「我最近有點心煩。」
→ Gate 觸發（Explicit Emotion）。Appraisal：全部 unknown（無事件）。State：Frustration: low, confidence 0.6。state.json 更新。

**回合二**：「就是那個姓方的，自己工作不做還推給我，老闆也搞不清楚狀況。」
→ Gate 觸發（Significant Event）。Event Linking：連結回合一的事件，補充原因。Appraisal：Fairness: violated, Control: low, Responsibility: external。State Update：Frustration → moderate/high。

**回合三**：「才不是，老闆瞎了眼了，只要事情有人做就好。」
→ Gate 觸發（Context Conflict + 情緒升高）。State：Anger: low 加入，Frustration 上升。上一輪輕幽默的回應策略被拒絕 → 改為直接反映處境，不再換解讀。

整個對話中，HESM 追蹤了一條清晰的情緒軌跡：心煩（原因不明）→ 挫折（原因揭露）→ 憤怒（情緒升高，策略調整）。

普通 LLM 也可能在某一輪做對，但三輪的連貫性需要狀態追蹤才能維持。

---

## 技術說明

HESM 不需要訓練新模型、不需要外部 API。它是一個**prompt-based skill**，直接掛載在任何支援 system prompt 的 LLM 上執行。

狀態透過兩個 JSON 檔案持久化：

- `state.json`：當前情緒狀態（10 種情緒 × intensity / confidence / trend）
- `events.json`：事件記錄（description, first_mentioned, last_mentioned, status, linked_emotions）

每次對話開始時讀取，回應後寫回。狀態跨 session 存活。

---

## MVP 成功條件

這個版本（v1.1）不要求證明「HESM 準確理解人的真正情緒」。

只確認三件事：

1. 該啟動時能啟動，不需要每句分析
2. 能維持並修正 Human State，而不是每句重新分類
3. 加入 HESM 後的回答，相較普通 LLM 有實際改善，且沒有明顯增加過度解讀

若成立，下一階段再進入更正式的 Ontology 驗證、標註資料與 HESM Core 開發。

---

## 結語

情緒理解不是一個分類問題，而是一個**脈絡追蹤問題**。

HESM 的嘗試是：用最低成本，先在 prompt 層驗證這條鏈是否成立——

**情緒顯著性 → Appraisal → 狀態更新 → 更好的脈絡 → 更好的對話**

如果這條鏈成立，再談模型層的實作。

---

*HESM Skill MVP v1.1 | 2026*
