---
title: GPT、Claude、Gemini、MiniMax、GLM 怎麼分工？從 oh-my-opencode 的代理設定來看
author: Cooper
date: 2026-02-05 09:00:00 +0800
categories: [AI Tools]
tags: [ai, opencode, claude, gpt, gemini, minimax, glm]
image:
  path: /oh-my-opencode-models/banner.svg
---

## 前言

現在還在問「哪個模型最強」，老實說有點像在問哪把起子最強一樣。你真正在做事的時候，問的不是這個。

我自己最近比較有感的做法，反而是看一套實戰框架到底怎麼分工。因為代理場景裡，模型不是拿來參加排行榜的，是拿來放在不同位置上做不同事的。

所以這篇我不想寫成那種全模型點名簿。我直接借 `oh-my-opencode`（現在 repo 名稱已改成 `oh-my-openagent`）裡的代理設定和 fallback 鏈來看，`GPT`、`Claude`、`Gemini`、`MiniMax`、`GLM` 到底各自比較像哪種角色。[1][2]

## 先把最重要的一句話講掉

如果只看目前這套代理框架的配置思路，我自己的理解會是這樣：

- `Claude`：適合當主 orchestrator，長任務、規範感、穩定度高
- `GPT`：適合深度執行、重推理、實作與驗證
- `Gemini`：很適合搜尋、前端/視覺、輕量任務
- `MiniMax`：速度快、成本友善，適合 utility 與檢索類任務
- `GLM`：在這套配置裡常拿來當 fallback 或平衡型主力，尤其在多代理環境裡很有存在感

如果你是想找一個全能王，看到這裡差不多就可以先停了。真正在多代理場景裡，通常還是**分工比單挑更合理**。

## 這套框架看模型的方式，其實很務實

我去看 `oh-my-opencode / oh-my-openagent` 現在的 README、installation guide 和 model requirement 設定，裡面其實把這件事講得很明白。[1][2][3]

### 1. Sisyphus 這種主代理，偏向 Claude / GLM

在 model fallback 裡，主 orchestrator 常優先走 `claude-opus-4-6`，再來才是 `GLM` 這類模型。[3]

這個訊號很明顯。主代理不是只負責回答問題，它還要拆任務、分派背景代理、收流程。這種位置吃的不是單點爆發，而是整體穩定度。

### 2. Hephaestus 這類深度執行代理，偏向 GPT

框架裡很明顯有「GPT 適合 Hephaestus 深度工作」的思路，甚至還直接寫出 `Do NOT use Sisyphus with GPT. For GPT models, always use Hephaestus.` 這種提醒。[4]

這句話我覺得很有意思，因為它其實在說：

> GPT 很強，但更適合拿來做深挖與執行，而不是當總指揮。

### 3. Gemini 常出現在搜尋、前端、輕量探索

installation guide 裡提到 `Gemini 3 Pro` 對 visual/frontend 任務很有優勢，`Gemini 3 Flash` 則偏向快、輕、適合 doc search 和 light tasks。[2]

這跟我自己的感覺也滿接近。Gemini 不見得每題都最穩，可是在要速度、要大範圍閱讀的時候，常常很好用。

### 4. MiniMax 很明顯被當成高性價比工具人

在設定裡，`MiniMax M2.5` 常被放在 `explore`、`librarian` 這類輕量任務的 fallback 鏈裡。[3]

講白一點，這代表它很適合：

- 搜
- 查
- 跑輕任務
- 降成本

如果你每天很多小任務，這類模型很有存在感。

### 5. GLM 很像是一個很實際的平衡位

GLM 在這套框架裡出現頻率也不低，尤其在 `z.ai` / `opencode` 來源下，常被用成主力 fallback、甚至部分主代理選項。[2][3]

它很像那種不一定最亮眼，可是放在整體流程裡就是很耐用的角色。

## 如果你是新手，我會怎麼記這幾家

下面這段我故意不用太學術的講法，直接講人話。

### Claude：像專案經理型工程師

它不是每個地方都最激進，但在長任務、規則感、流程穩定這種地方，通常表現不錯。

如果你要的是：

- 把需求拆清楚
- 按規矩做
- 不要一下太跳

那 Claude 很常會讓人比較安心。

### GPT：像很能做事的重工型工程師

我會把它放在需要深挖、要真的下刀修改、要做驗證的地方。

尤其在 coding assistant 這類執行任務上，它的存在感還是很強。

### Gemini：像資訊量很大的快手型助手

適合快速閱讀、文件整理、視覺與前端類任務。

如果你常做的是探索、查資料、前端嘗試，它常常會很順。

### MiniMax：像便宜又俐落的日常工具人

不用每次都派大將上場。很多探索、整理、搜索，MiniMax 這種速度快、成本友善的模型就夠了。

### GLM：像很實用的平衡型替補主力

有時候它不是最出風頭的，但在多代理架構裡，這種能撐住整體流程的模型很重要。

## 那如果今天只能先選一個呢

可以選，但我得先講，**這本來就是退而求其次的做法。**

如果真的只能選一個，我會看你的主要工作是什麼：

- 偏長任務 orchestration：先看 `Claude`
- 偏深度實作：先看 `GPT`
- 偏查資料 / 視覺 / 前端：先看 `Gemini`
- 偏成本 / 快速小任務：先看 `MiniMax`
- 偏平衡與替補可得性：可以看 `GLM`

如果你只是想先照這篇跑一輪最小分工，我自己會先這樣配：

- 主代理：`Claude`
- 深度執行：`GPT`
- 搜尋 / 輕量探索：`Gemini` 或 `MiniMax`
- 平價 fallback：`GLM`

先這樣配，你就已經比「一個模型硬打全部」正常很多了。

## 我自己最在意的，其實一直都不是單挑

這也是我想寫這篇的原因。

現在太多文章都在比 benchmark，可是真的用起來，你很少只叫一個模型做完整套流程。更多時候是：

- 一個負責規劃
- 一個負責搜尋
- 一個負責重工
- 一個負責便宜快速地補洞

如果你用的是代理型框架，這個思維通常比排行榜重要得多。

如果你真的有在用 `OpenCode`、`Claude Code` 或各種 agent harness，最後大概都會慢慢接受一件事：

> 模型不是拿來選邊站，是拿來安排位置。

這也是我看 `oh-my-opencode / oh-my-openagent` 最大的收穫之一。它不是要你選邊站，而是在提醒你：不同模型有不同脾氣，你要把它放在適合的位置。

如果你是剛開始玩多模型，我反而建議你先不要追求「最強」，先追求「分工合理」。這樣通常比較省錢，也比較不容易翻車。

## 參考資料

[1]: https://github.com/code-yeongyu/oh-my-opencode
[2]: https://github.com/code-yeongyu/oh-my-openagent
[3]: https://github.com/code-yeongyu/oh-my-opencode/blob/dev/docs/guide/installation.md
[4]: https://github.com/code-yeongyu/oh-my-opencode/blob/dev/src/hooks/no-sisyphus-gpt/hook.ts
