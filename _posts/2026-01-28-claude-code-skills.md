---
title: Claude Code Skills 與 MCP，擴展 AI 能力的兩種方式
author: Cooper
date: 2026-01-28 11:00:00 +0800
categories: [AI Tools]
tags: [ai, claude, mcp, skills, anthropic]
image:
  path: /claude-code-skills/banner.jpg
---

最近這段時間，Claude Code 簡直成了開發者圈子裡的當紅炸子雞。作為 Anthropic 推出的命令行工具，它不僅能寫代碼，還能直接在你的終端裡執行指令、運行測試，甚至幫你修復 Bug。不過，用久了你可能會發現，雖然 Claude 本身很聰明，但有時候它還是會顯得有些「手短」，比如它不知道你公司的特定代碼規範，或者沒辦法直接讀取你 Slack 裡的訊息。

為了讓 Claude 變得更強大，我們有兩種主要的擴展方式：**Skills** 和 **MCP (Model Context Protocol)**。這兩者雖然都能增強 Claude 的能力，但背後的邏輯和適用場景卻大不相同。今天我就來跟大家聊聊這兩者的差異，以及在什麼情況下該選哪一個。

## Skills 是什麼？

簡單來說，Skills 就是一堆 Markdown 格式的指令文件。如果你用過一些 AI Agent 框架，你可以把它想像成是一種「領域知識注入」。

在 Claude Code 中，Skills 通常存放在 `~/.claude/skills/` 或者專案根目錄下的 `.claude/skills/` 文件夾裡。當你啟動 Claude Code 時，它會自動讀取這些 Markdown 文件，並將其中的內容作為系統提示詞（System Prompt）的一部分。

### Skills 的運作機制

Skills 並不是真正的「代碼」，它更像是給 Claude 的「操作手冊」。例如，你可以寫一個關於 Git 操作的 Skill，告訴 Claude 在提交代碼時必須遵循 Conventional Commits 規範，並且在 commit 之前要先跑一遍測試。

```markdown
# git-master Skill

你是一個 Git 專家。在執行任何 Git 操作之前，請確保：
1. 檢查當前分支狀態。
2. 提交訊息必須符合 `type(scope): description` 格式。
3. 如果是修復 Bug，必須引用相關的 Issue 編號。
```

當 Claude 讀到這個 Skill 後，它在處理 Git 相關任務時就會自動遵守這些規則。

{: .prompt-info }
Skills 的優勢在於它非常輕量。你不需要寫任何 Python 或 Node.js 代碼，只需要寫好 Markdown 說明文件，Claude 就能理解並執行。

## MCP 是什麼？

MCP 全稱是 **Model Context Protocol**，這是 Anthropic 推出的一套開放標準。如果說 Skills 是「操作手冊」，那麼 MCP 就是「外部插件」或「工具箱」。

MCP 採用了客戶端-服務器（Client-Server）架構。Claude 作為客戶端，通過 JSON-RPC 協議與運行在本地或遠端的 MCP 服務器通信。這意味著 MCP 可以做 Skills 做不到的事情：**執行真實的邏輯、訪問實時數據、調用外部 API**。

### MCP 的運作機制

當你配置了一個 MCP 服務器（例如 GitHub MCP），Claude 就不再只是「知道」怎麼操作 GitHub，而是擁有了「調用 GitHub API」的能力。它可以列出 PR、創建 Issue、甚至讀取特定倉庫的內容。

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "your_token_here"
      }
    }
  }
}
```

{: .prompt-warning }
MCP 需要運行一個獨立的進程（服務器），因此它的配置比 Skills 複雜一些，且需要消耗額外的系統資源。

## 兩者差異比較

雖然兩者都能擴展能力，但我們可以從以下幾個維度來對比：

| 維度 | Skills | MCP |
| :--- | :--- | :--- |
| **本質** | 靜態指令 (Markdown) | 動態工具 (Server) |
| **複雜度** | 極低，寫字就行 | 中到高，需要編程或配置 |
| **實時性** | 依賴模型內置知識 | 可獲取實時數據 |
| **外部訪問** | 無法直接訪問 API | 可自由訪問網絡與系統資源 |
| **適用場景** | 規範定義、流程指導、特定領域知識 | 數據集成、自動化操作、複雜邏輯執行 |

### 什麼時候該用 Skills？

如果你只是想規範 Claude 的行為，或者教它一些專案特有的術語和流程，Skills 是最佳選擇。比如：
- 定義專案的代碼風格。
- 提供特定庫的 API 使用範例。
- 設定複雜任務的標準作業程序 (SOP)。

### 什麼時候該用 MCP？

如果你需要 Claude 與外界互動，或者處理需要精確計算、實時查詢的任務，那就必須用 MCP。比如：
- 讀取數據庫中的數據。
- 發送 Slack 訊息或郵件。
- 搜索 Google 或訪問特定的網頁內容。

## 推薦的擴展工具

在實際開發中，我推薦大家嘗試以下幾個非常實用的 Skills 和 MCP。

### 推薦的 Skills (來自 oh-my-opencode)

1. **git-master**: 這是我最常用的 Skill。它能確保 Claude 寫出完美的 Commit Message，並且在重構或合併分支時不會亂搞。
2. **frontend-ui-ux**: 當你需要 Claude 幫你寫前端組件時，這個 Skill 能讓它更注重設計細節，而不僅僅是把功能寫出來。
3. **playwright**: 專門用於瀏覽器自動化測試。它提供了豐富的指令集，讓 Claude 能更精準地編寫和調試測試腳本。

### 推薦的 MCP 服務器

1. **filesystem**: 這是最基礎但也最強大的 MCP。它允許 Claude 以更結構化的方式讀寫文件，處理大型專案時比內置的文件操作更可靠。
2. **GitHub**: 讓 Claude 成為你的 GitHub 助手，處理 PR 和 Issue 變得易如反掌。
3. **Slack**: 如果你的團隊使用 Slack 溝通，這個 MCP 可以讓 Claude 直接讀取討論上下文，甚至幫你總結會議記錄。

## 結語

總結來說，Skills 和 MCP 並不是競爭關係，而是互補關係。Skills 負責「大腦」的思維邏輯和規範，而 MCP 則負責延伸「雙手」的觸及範圍。

對於大多數開發者來說，我建議先從編寫簡單的 Skills 開始，把你的開發習慣和專案規範寫進去。當你發現 Claude 沒辦法獲取某些必要的外部數據時，再考慮引入 MCP。

Claude Code 的強大之處就在於這種極高的可擴展性。通過合理配置這兩者，你完全可以打造出一個專屬於你的、最強大的 AI 編程助手。

---

[1]: https://modelcontextprotocol.io/
[2]: https://www.anthropic.com/news/model-context-protocol
[3]: https://github.com/modelcontextprotocol/servers
[4]: https://docs.anthropic.com/en/docs/agents-and-tools/claude-code/overview
