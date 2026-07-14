# Pokémon TCG AI Battle Challenge 競賽整理

> 整理日期：2026-07-14  
> 主要來源：Kaggle Simulation、Kaggle Strategy、Matsuo Institute `cabt Engine` 官方文件  
> 注意：競賽規則與時程仍可能更新，正式提交前應再次檢查 Kaggle 頁面。

---

## 1. 這個比賽要做什麼？

這場比賽不是一般 Kaggle 的「讀取表格資料、預測答案、上傳 `submission.csv`」。

你要製作的是一個能夠玩 **Pokémon Trading Card Game（寶可夢集換式卡牌遊戲）** 的 AI Agent，讓它：

1. 讀取目前的遊戲狀態。
2. 從模擬器提供的合法選項中選擇動作。
3. 使用你設計的 60 張牌組進行完整對戰。
4. 在 Kaggle 的自動對戰環境中與其他參賽者的 Agent 競爭。
5. 另外撰寫技術報告，說明 Agent 策略、牌組設計與實驗結果。

整體競賽分成兩個互相關聯的部分：

| 類別 | 主要任務 | 最後提交物 |
|---|---|---|
| **Simulation Category** | 提交實際會玩牌的 AI Agent，參加自動對戰排行榜 | 包含 `main.py`、`deck.csv` 的 `.tar.gz` |
| **Strategy Category** | 說明 Agent 的策略、技術方法、牌組概念與實驗結果 | Kaggle Writeup，最多 2,000 字 |

> 要參加 Strategy Category，必須同時參加 Simulation Competition。

---

# 2. 為什麼這是一個困難的 AI 問題？

Pokémon TCG 不是單純的分類或回歸問題，而是需要連續做決策的遊戲環境。

AI 必須處理：

- 對手手牌與牌庫順序等隱藏資訊。
- 抽牌、硬幣等隨機事件。
- 大量卡牌間的組合與連鎖效果。
- 當前回合的短期利益。
- 數個回合後的長期勝利條件。
- 不同對手、牌組與戰術造成的策略差異。
- 有限的運算與對戰時間。

因此它接近以下研究問題：

- Sequential Decision Making
- Imperfect Information Game
- Planning under Uncertainty
- Game AI
- Reinforcement Learning
- Search and Simulation
- Opponent Modeling
- Rule-based / Heuristic Agent

深度學習或強化學習不是提交格式上的必要條件；規則式、評分函數、搜尋、強化學習或混合方法都可以製作 Agent。真正重要的是 Agent 是否能穩定做出有效且合法的決策。

---

# 3. Simulation Category：實際對戰 Agent

## 3.1 你要完成的核心任務

你的程式會反覆收到目前的遊戲觀察資料，然後回傳要選擇的合法動作。

概念流程如下：

```text
cabt Engine 產生目前遊戲狀態
              │
              ▼
      Observation 傳給 Agent
              │
              ▼
 Agent 分析場面與合法選項
              │
              ▼
   回傳所選 Option 的索引
              │
              ▼
    模擬器執行動作並進入下一狀態
```

AI 可能需要決定：

- 起手時選擇哪隻 Pokémon 作為戰鬥 Pokémon。
- 哪些 Pokémon 放上備戰區。
- 要打出哪張 Trainer 卡。
- 是否使用 Pokémon 的特性。
- 能量要附給哪一隻 Pokémon。
- 是否進化。
- 是否撤退。
- 使用哪一個招式。
- 搜尋牌庫時要選擇哪些卡。
- 何時結束回合。
- 如何因應對手目前的場面與可能策略。

---

## 3.2 `Observation` 包含什麼？

根據 `cabt Engine` 官方文件，Observation 主要包含三個部分：

| 欄位 | 意義 |
|---|---|
| `logs` | 過去的行動與遊戲事件紀錄 |
| `current` | 目前的完整可觀察場面狀態 |
| `select` | 目前可選擇的合法選項 |

### `current`

`current` 可能包含：

- 雙方玩家狀態。
- 當前戰鬥 Pokémon。
- 備戰 Pokémon。
- 自己的手牌。
- 對手的手牌張數。
- 獎賞卡。
- 剩餘牌庫張數。
- 棄牌區。
- 場地卡。
- 回合數。
- 先後攻資訊。
- 本回合是否已使用 Supporter、能量或撤退。
- 中毒、灼傷、睡眠、麻痺、混亂等狀態。
- 對戰是否已產生結果。

重要的不完全資訊包括：

- 自己可以看到自己的手牌內容。
- 對手的手牌通常只顯示張數，不會直接顯示內容。
- 蓋牌狀態的獎賞卡不會直接提供卡牌內容。
- 牌庫的未抽牌順序未知。

### `select`

`select` 提供目前允許執行的選項。

Agent 不需要自行產生任意遊戲指令，而是從模擬器提供的合法 `option` 中，回傳所選選項的索引。

官方文件中的最小隨機 Agent 概念如下：

```python
import random

def agent(obs_dict: dict) -> list[int]:
    options = obs_dict["select"]["option"]
    max_count = obs_dict["select"]["maxCount"]

    return random.sample(
        list(range(len(options))),
        max_count
    )
```

這只是示範介面，不是有競爭力的策略。

實際實作時，需要確認每次選擇要求的數量、合法索引與回傳型別，避免 Agent 因格式錯誤而直接輸掉或無法執行。

---

## 3.3 你的 Agent 實際上要學什麼？

Agent 的核心不是判斷「這張牌是哪一類」，而是學習或設計一個決策函數：

\[
a_t = \pi(s_t, O_t, H_t)
\]

其中：

- \(s_t\)：目前可觀察的遊戲狀態。
- \(O_t\)：目前合法選項。
- \(H_t\)：歷史事件或對戰紀錄。
- \(a_t\)：Agent 最後選擇的動作。
- \(\pi\)：你設計的策略。

簡單 Agent 可能只依照當前場面選擇最高分動作。

進階 Agent 則可能考慮：

- 下一個狀態會變成什麼。
- 對手下一回合可能做什麼。
- 某個動作對幾個回合後的勝率有何影響。
- 資源現在應該使用，還是保留給後續回合。
- 某個選擇在不同牌組對戰中是否穩定。

---

# 4. 牌組 `deck.csv`

## 4.1 基本格式

`deck.csv` 需要列出一副 **60 張卡牌**的卡牌 ID。

官方文件的格式是每行一個卡牌 ID：

```csv
278
278
278
145
145
7
7
```

完整檔案需要有 60 行有效卡牌 ID。

卡牌 ID 可以透過 `cabt Engine` 的卡牌資料 API 取得，例如：

```python
all_card_data()
```

---

## 4.2 牌組不是和 Agent 分開評估的

牌組與 Agent 的策略必須互相配合。

例如：

- Agent 只會簡單判斷當回合最高攻擊力，可能適合較直接的快攻牌組。
- Agent 擅長多回合搜尋，可能可以操作較複雜的進化與資源控制牌組。
- Agent 不擅長判斷複雜卡牌文字時，牌組應避免過多高度情境式卡牌。
- Agent 擅長分析棄牌區與長期資源時，可以使用回收或資源循環戰術。

因此，研究重點不是單純找「最強牌組」，而是：

> 找到能被你的 Agent 穩定操作，並在多種對局中發揮效果的牌組。

---

# 5. Simulation 正式提交格式

正式提交是一個 `.tar.gz` 壓縮檔。

最低必要結構為：

```text
submission.tar.gz
├── main.py
└── deck.csv
```

注意事項：

1. `main.py` 必須位於壓縮檔的最上層，不能多包一層資料夾。
2. 壓縮檔內必須包含 `deck.csv`。
3. `deck.csv` 必須是一副合法的 60 張牌組。
4. 若 `main.py` 使用其他自訂模組、模型權重或資料檔，也必須一併打包，並確保 Kaggle 執行環境可以載入。
5. 不要只在自己的電腦測試；必須在接近 Kaggle 的環境中測試匯入、路徑、依賴與執行時間。

### 容易混淆的地方

`cabt Engine` Getting Started 文件使用：

```text
agent.py
deck.csv
```

作為本機最小測試範例。

但是 Kaggle Simulation 的正式提交要求是：

```text
main.py
deck.csv
```

所以：

- 本機測試文件中的示範名稱：`agent.py`
- Kaggle 正式提交入口：`main.py`

不要直接把本機範例原封不動打包而漏掉 `main.py`。

---

# 6. 如何在本機執行對戰？

官方文件提供的基本方式是透過 `kaggle_environments` 建立 `cabt` 環境。

概念範例如下：

```python
from kaggle_environments import make
from agent import agent

with open("deck.csv") as f:
    deck = [
        int(line)
        for line in f.readlines()
        if line.strip()
    ]

env = make(
    "cabt",
    configuration={
        "decks": [deck, deck]
    }
)

env.run([agent, agent])

with open("result.html", "w") as f:
    f.write(env.render(mode="html"))

print("Simulation finished.")
```

這個範例會：

1. 讀取牌組。
2. 建立兩個使用相同牌組的 Agent。
3. 執行一場自我對戰。
4. 將對戰結果輸出成 HTML。

實際研究時，建議改成大量對戰並記錄：

- 勝率。
- 先攻／後攻勝率。
- 平均回合數。
- 超時或非法動作次數。
- 不同對手牌組的 matchup。
- 特定動作的使用頻率。
- 勝利與失敗原因。
- Agent 每次決策的運算時間。

---

# 7. Simulation 如何評分？

Simulation 使用自動對戰 Ladder。

每個提交的 Agent 會：

1. 與 Ladder 中技能評分相近的 Agent 進行多場對戰。
2. 根據勝負調整技能評分。
3. 在持續對戰後逐漸形成排行榜。

因此分數不等同於固定測試資料上的一次性 accuracy，而是會受到以下因素影響：

- 對手池。
- 對戰場數。
- 對手實力。
- 牌組 matchup。
- 隨機抽牌。
- 先後攻。
- Agent 的穩定性。
- 非法輸出、錯誤或超時。
- Ladder 評分是否已經收斂。

Simulation 規則目前允許：

- 每日最多提交 5 次。
- 最多選擇 2 個 Final Submissions。

官方也說明，截止後對戰仍可能持續一段時間，以讓最終排行榜進一步校準與收斂。

---

# 8. Strategy Category：策略與技術報告

## 8.1 要提交什麼？

Strategy Category 要提交一篇 **Kaggle Writeup**。

它不是再交一份 `submission.csv`，也不是只貼程式碼，而是說明：

- 你如何設計 AI Agent。
- 為什麼採用這個方法。
- Agent 如何理解遊戲狀態。
- Agent 如何選擇動作。
- 牌組為什麼這樣設計。
- Agent 與牌組如何互相配合。
- 做過哪些實驗。
- 哪些方法成功、哪些方法失敗。
- Simulation 表現如何。
- 方法有哪些限制與可改進之處。

---

## 8.2 Writeup 基本要求

目前官方頁面列出的重點包括：

- 必須選擇 Competition Track。
- Writeup 不得超過 2,000 字。
- 超過字數可能受到扣分。
- Media Gallery 可以附加圖片或影片，但屬於 optional。
- 圖片、影片與 Pokémon 相關素材必須遵守競賽授權與使用規定。
- 最後必須完成正式提交，不能只停留在草稿狀態。

---

# 9. Strategy 評分比例

| 評分項目 | 比例 | 主要評估內容 |
|---|---:|---|
| **Model Score** | **70%** | 方法原創性、技術合理性、重複對戰穩定性，以及 Simulation 表現 |
| **Deck Score** | **20%** | 牌組概念是否清楚、是否符合預定策略、卡牌選擇是否合理 |
| **Report Score** | **10%** | 報告是否清楚、有邏輯，以及圖表、表格或其他視覺元素是否有效 |

## 9.1 Model Score 不只是看你用了什麼模型

雖然名稱叫做 Model Score，但不代表一定要使用深度神經網路。

評審真正關心的是：

- 方法是否有清楚的策略邏輯。
- 方法是否具有原創性。
- 方法是否技術上合理。
- Agent 是否能在重複對戰中穩定運作。
- 是否有實驗支持你的設計。
- Simulation 的實際表現如何。

所以一個設計完整、測試充分的規則式或搜尋式 Agent，可能比一個缺乏穩定性與實驗分析的複雜模型更有說服力。

---

# 10. Strategy Writeup 建議結構

以下結構可以直接作為未來報告大綱。

## 10.1 Abstract

簡短回答：

- 你想解決什麼問題？
- 使用什麼方法？
- 使用什麼牌組？
- 最重要的結果是什麼？

## 10.2 Problem Definition

說明 Pokémon TCG Agent 面臨的問題：

- 不完全資訊。
- 隨機事件。
- 巨大的動作與卡牌組合空間。
- 長期資源規劃。
- 不同對手造成的策略差異。

## 10.3 Agent Architecture

說明 Agent 的組成，例如：

```text
Observation Parser
        │
        ▼
State Feature Extractor
        │
        ▼
Legal Action Evaluator
        │
        ├── Immediate reward
        ├── Resource value
        ├── Board development
        ├── Knockout probability
        └── Future-state value
        │
        ▼
Action Selector
```

## 10.4 State Representation

說明你使用哪些資訊：

- Active Pokémon。
- Bench 狀態。
- HP 與傷害。
- 能量配置。
- 手牌。
- 棄牌區。
- 剩餘牌庫。
- 獎賞卡數量。
- 對手場面。
- 回合數。
- 過去行動紀錄。

## 10.5 Action Selection

說明合法動作如何評分。

例如：

\[
Score(a)=
w_1 V_{\text{damage}}
+w_2 V_{\text{KO}}
+w_3 V_{\text{board}}
+w_4 V_{\text{resource}}
+w_5 V_{\text{future}}
-w_6 V_{\text{risk}}
\]

可能的評估項目：

- 是否能擊倒對手。
- 能造成多少傷害。
- 是否能完成進化。
- 是否改善下一回合攻擊條件。
- 是否消耗過多關鍵資源。
- 是否讓重要 Pokémon 暴露在危險中。
- 是否增加未來抽到有效牌的機率。

## 10.6 Deck Design

說明：

- 牌組核心勝利方式。
- 主要攻擊手。
- 輔助 Pokémon。
- Trainer 卡功能。
- 能量比例。
- 起手穩定性。
- Agent 為何能操作這副牌。
- 哪些牌因為 Agent 不容易正確使用而被移除。

## 10.7 Training or Optimization

依你的方法說明：

- 手工規則如何設計。
- 權重如何調整。
- 是否使用 grid search、Bayesian optimization 或 evolutionary search。
- 是否使用 imitation learning。
- 是否使用 reinforcement learning。
- reward 如何定義。
- 是否使用 self-play。
- 是否使用 MCTS 或有限深度搜尋。

## 10.8 Experiments

至少準備一個 baseline。

例如：

| Agent | 方法 | 對戰數 | 勝率 | 非法輸出率 | 平均決策時間 |
|---|---|---:|---:|---:|---:|
| Random | 隨機合法動作 | 1,000 | 20.1% | 0% | 1 ms |
| Rule V1 | 攻擊優先 | 1,000 | 38.4% | 0% | 2 ms |
| Rule V2 | 加入進化與能量規劃 | 1,000 | 52.7% | 0% | 4 ms |
| Search V1 | 兩步狀態搜尋 | 1,000 | 59.3% | 0% | 90 ms |

## 10.9 Ablation Study

逐一移除 Agent 中的模組，觀察性能變化：

| 版本 | 移除內容 | 勝率變化 |
|---|---|---:|
| Full Agent | 完整模型 | 60.2% |
| No KO Bonus | 移除擊倒獎勵 | -8.4% |
| No Future Value | 不考慮未來狀態 | -5.1% |
| No Resource Penalty | 不懲罰資源浪費 | -3.7% |

Ablation 可以幫助證明：

- 哪一個設計真正有效。
- 改善不是單純來自運氣。
- Agent 的各個模組各自貢獻多少。

## 10.10 Failure Analysis

不要只報告成功結果，也應分析：

- 哪些牌組特別難打。
- 哪些場面容易做錯。
- 是否經常過度消耗資源。
- 是否無法理解某類卡牌效果。
- 搜尋是否因分支太多而變慢。
- RL 是否只記住特定對手。
- Agent 是否在先攻或後攻明顯失衡。

## 10.11 Conclusion

總結：

- 最有效的方法。
- 最重要的實驗發現。
- 牌組與 Agent 的配合方式。
- 目前限制。
- 後續改進方向。

---

# 11. 你最後需要交什麼？

## 11.1 Simulation Checklist

- [ ] `main.py`
- [ ] `deck.csv`
- [ ] `deck.csv` 包含合法的 60 張卡牌 ID
- [ ] `main.py` 位於壓縮檔最上層
- [ ] 所有自訂依賴檔案皆已打包
- [ ] 可以在 Kaggle 相容環境成功 import
- [ ] Agent 每次只回傳合法選項索引
- [ ] 回傳數量符合該次選擇要求
- [ ] Agent 不會因空值、特殊狀態或遊戲結束而崩潰
- [ ] 已進行大量自我對戰
- [ ] 已檢查超時、錯誤與非法動作
- [ ] 最後輸出為 `.tar.gz`

## 11.2 Strategy Checklist

- [ ] 已參加 Simulation Competition
- [ ] 已提交可執行的 Simulation Agent
- [ ] 已選擇 Strategy Competition Track
- [ ] Writeup 不超過 2,000 字
- [ ] 清楚說明 Agent 策略
- [ ] 清楚說明牌組概念
- [ ] 提供 baseline
- [ ] 提供實驗結果
- [ ] 提供重複對戰的穩定性分析
- [ ] 提供 ablation 或版本比較
- [ ] 說明失敗案例與限制
- [ ] 圖表與媒體符合 Pokémon 素材授權規定
- [ ] 已正式 Submit，而不是只儲存 Draft

---

# 12. 建議的實作順序

## 階段 1：先完成可執行版本

1. 下載官方資料與 Starter Notebook。
2. 跑通 `cabt Engine`。
3. 使用隨機 Agent 完成一場完整對戰。
4. 理解 Observation 與 legal options。
5. 正確讀取 60 張 `deck.csv`。
6. 成功產生 `result.html`。
7. 成功打包第一份 `submission.tar.gz`。

目標不是先拿高分，而是先確保：

> Agent 能合法、穩定且完整地打完一場比賽。

## 階段 2：建立 Rule-based Baseline

加入最基本的決策優先順序：

1. 可以擊倒對手時優先擊倒。
2. 可以有效攻擊時優先攻擊。
3. 優先完成核心 Pokémon 的進化。
4. 優先附能量給即將攻擊的 Pokémon。
5. 維持備戰區的後續攻擊手。
6. 避免浪費關鍵 Trainer 卡。
7. 沒有其他有效行動時結束回合。

## 階段 3：加入狀態評分函數

建立可比較的 state value：

```text
場面分數 =
    獎賞卡優勢
  + 有效攻擊手數量
  + 場上總 HP
  + 可用能量
  + 手牌品質
  + 下一回合攻擊可能性
  - 即將被擊倒風險
  - 關鍵資源損失
```

## 階段 4：加入搜尋或學習

再逐步嘗試：

- One-step lookahead。
- Two-step lookahead。
- Beam search。
- Monte Carlo simulation。
- MCTS。
- Supervised learning。
- Imitation learning。
- Reinforcement learning。
- Self-play。
- Hybrid rule + learned value model。

## 階段 5：建立實驗紀錄

每次修改都要記錄：

- Git commit。
- Agent 版本。
- 牌組版本。
- 對戰對手。
- 隨機種子。
- 對戰場數。
- 勝率。
- 決策時間。
- Crash 數量。
- 主要修改。
- 是否上傳 Kaggle。
- Ladder 分數變化。

這些資料未來可以直接變成 Strategy Writeup 的實驗章節。

---

# 13. 推薦的專案資料夾結構

```text
pokemon-tcg-agent/
├── submission/
│   ├── main.py
│   ├── deck.csv
│   ├── agent/
│   │   ├── parser.py
│   │   ├── evaluator.py
│   │   ├── policy.py
│   │   └── utils.py
│   └── models/
│       └── value_model.bin
│
├── experiments/
│   ├── run_matches.py
│   ├── evaluate_matchups.py
│   ├── ablation.py
│   └── configs/
│
├── decks/
│   ├── deck_v1.csv
│   ├── deck_v2.csv
│   └── deck_notes.md
│
├── results/
│   ├── match_results.csv
│   ├── matchup_summary.csv
│   └── plots/
│
├── notebooks/
├── tests/
├── README.md
└── build_submission.py
```

正式打包時，只把執行 Agent 必要的檔案放入 `.tar.gz`。

---

# 14. 重要時程

Kaggle 官方截止時間皆以 **UTC 11:59 PM** 為準。下表同時換算為台灣時間（UTC+8）。

| 項目 | Kaggle 日期（UTC） | 台灣時間 |
|---|---|---|
| Simulation 開始 | 2026-06-16 11:00 UTC | 2026-06-16 19:00 |
| Simulation 加入／組隊截止 | 2026-08-09 23:59 UTC | 2026-08-10 07:59 |
| Simulation 最終提交截止 | 2026-08-16 23:59 UTC | 2026-08-17 07:59 |
| Strategy 加入／組隊截止 | 2026-09-06 23:59 UTC | 2026-09-07 07:59 |
| Strategy Writeup 截止 | 2026-09-13 23:59 UTC | 2026-09-14 07:59 |

> 不建議等到台灣時間早上 07:59 才提交。應至少提前一天完成最終版本，以防止 Kaggle 執行、打包或提交失敗。

---

# 15. 獎項概況

Strategy Main Track 的總獎金為：

```text
US$240,000
```

共有 8 組 Finalists，每組可獲得：

```text
US$30,000
```

Finalists 也可能受邀參加後續的實體決賽活動；實際資格、地點與要求應以官方後續通知為準。

Simulation Category 的主要功能是建立實際競爭排行榜，並作為 Strategy 評估中的實際性能證據。

---

# 16. 最簡單的總結

這場 Kaggle 要你完成一個完整的遊戲 AI 系統：

```text
卡牌資料理解
      +
60 張牌組設計
      +
遊戲狀態解析
      +
合法動作選擇
      +
長短期策略規劃
      +
大量模擬對戰
      +
實驗與技術報告
```

最後需要提交兩項主要成果：

```text
Simulation：
submission.tar.gz
├── main.py
└── deck.csv
```

以及：

```text
Strategy：
一篇最多 2,000 字的 Kaggle Writeup
```

最適合的起步方式不是立刻訓練複雜 RL，而是：

1. 跑通官方環境。
2. 提交能完整對戰的隨機 Agent。
3. 建立 Rule-based baseline。
4. 設計牌組與狀態評分函數。
5. 大量對戰並記錄結果。
6. 再決定是否加入搜尋、RL 或其他學習方法。

---

# 18. 官方來源

1. Kaggle — PTCG AI Battle Challenge Simulation  
   <https://www.kaggle.com/competitions/pokemon-tcg-ai-battle>

2. Kaggle — PTCG AI Battle Challenge Simulation Rules  
   <https://www.kaggle.com/competitions/pokemon-tcg-ai-battle/rules>

3. Kaggle — PTCG AI Battle Challenge Strategy  
   <https://www.kaggle.com/competitions/pokemon-tcg-ai-battle-challenge-strategy>

4. Kaggle — PTCG AI Battle Challenge Strategy Rules  
   <https://www.kaggle.com/competitions/pokemon-tcg-ai-battle-challenge-strategy/rules>

5. Matsuo Institute — `cabt Engine` Documentation  
   <https://matsuoinstitute.github.io/cabt/>
