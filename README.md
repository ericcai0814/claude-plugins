# ericcai0814 claude-plugins marketplace

self-hosted [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) — 自家 plugin 的 catalog repo。

## Use

```bash
# 在 Claude Code CLI
/plugin marketplace add ericcai0814/claude-plugins

# 列出可裝的 plugins
/plugin marketplace list ericcai0814-claude-plugins

# 安裝
/plugin install claude-auto-dev
/plugin install meeting-pipeline
/plugin install ai-coding-workflow
```

## Plugins overview

| Plugin | Platform | Source |
|---|---|---|
| [`claude-auto-dev`](#claude-auto-dev) | 跨平台 | [`EricCaiCai/claude-auto-dev`](https://github.com/EricCaiCai/claude-auto-dev) |
| [`meeting-pipeline`](#meeting-pipeline) | macOS / Apple Silicon | [`ericcai0814/claude-meeting-pipeline`](https://github.com/ericcai0814/claude-meeting-pipeline) |
| [`ai-coding-workflow`](#ai-coding-workflow) | 跨平台 | [`ericcai0814/claude-ai-coding-workflow`](https://github.com/ericcai0814/claude-ai-coding-workflow) |

---

## `claude-auto-dev`

**TDD 自動開發循環。** Claude 跑 Red-Green-Refactor，Stop Hook 用 quality-gate 驗證完成。

### What it does

```
你: /auto-dev "實作用戶登入
              - JWT token 生成
              - Email + password 驗證
              - 錯誤訊息要友善"

Claude: [寫紅燈測試 → 實作 → 跑測試 → 重構 → touch .auto-dev/done]

Stop Hook 介入: 跑 build + test
  ✓ 通過 → 結束
  ✗ 失敗 → 把控制權還給 Claude，要求修正
```

### Why

傳統 TDD agent 用「逐項勾選 acceptance criteria」當完成判斷。這個 plugin 反過來：**checkpoint 退化為純紀錄，完成判斷只看 build / test exit code**。AC 顆粒度由 Claude 自由詮釋，Hook 不解析。

- Hook 不寫 checkpoint → 沒有 Claude / Hook 競爭條件
- Signal file（`.auto-dev/done`）作為 Claude 宣告完成的信號
- Quality-gate exit code 是唯一決策依據

### Triggers

`/auto-dev "task"`、「自動開發」、「自動 TDD」、「auto loop」、「幫我用 TDD 自動做完」。

### Quality-gate 自訂

預設偵測 `package.json` + `tsconfig.json` → 跑 tsc + npm test。其他語言請在你的 repo 加 `.auto-dev/config.json`：

```json
{ "build": "cargo check", "test": "cargo test" }
```

支援 Python（pytest）、Go（go test）、Rust、Deno 等任何回傳 exit code 的指令。

### 也可走 npm

除了 plugin 安裝，這個專案也發佈到 npm，可走 CLI scaffold 模式：

```bash
npx claude-auto-dev init        # 在 cwd 安裝 hook + skill
npx claude-auto-dev doctor      # 檢查安裝完整性
npx claude-auto-dev status      # 看 checkpoint
npx claude-auto-dev abort       # 安全中斷
```

詳細：[github.com/EricCaiCai/claude-auto-dev](https://github.com/EricCaiCai/claude-auto-dev)

---

## `meeting-pipeline`

**中文會議錄音 → 結構化紀錄 → 個人 issues。** 三段式整合工作流。

> ⚠️ **macOS / Apple Silicon only**（mlx-whisper 依賴 Apple MLX framework）。

### What it does

```
你: 整理 meetings/2026-05-20-sync.m4a 這場會議

Claude:
  [第 1 段] 背景跑 transcribe-zh → 產出 .txt / .srt / .vtt / .json
            → 驗證 hallucination（sort | uniq -c 找重複 > 50 次的行）

  [第 2 段] 選 template（教學場 vs 決議場）→ 讀逐字稿前 200 行樣本
            → 產出 meeting-notes.md（with `(?)` 標記不確定）
            → 給你校正人名與 owner

  [第 3 段] 若 repo 有 docs/agents/issue-tracker.md
            → 從校正後的 action items 挑 1-3 個拆 issue
```

整個流程是「**人機協作**」而非「**全自動**」——第 2 段必須由你確認 owner 才進第 3 段，避免 issue 拆給錯誤的人。

### Why

中文會議錄音用 LLM 整理常遇到三個問題：
1. **Whisper 對中英夾雜的轉錄品質不穩**——人名/技術名詞偏移率高（Cloud→Claude、Nebula→Naple）
2. **不同會議類型需求差很多**——training 場要抓「關鍵原則」、sync 場要抓「決議 + action items」，硬套同一份 template 紀錄會走味
3. **拆 issue 是 owner-sensitive 動作**——whisper 不告訴你誰說了什麼，全自動容易拆給錯誤的人

對應的設計：
- **anti-hallucination flags 內建在 `transcribe-zh`**（不需要每次自己記 mlx-whisper 參數）
- **兩種 template 分流**（`training-notes-template.md` / `notes-template.md`，附判斷表）
- **拆 issue 前強制 user 校正**（流程設計的強制 checkpoint）

### Triggers

「整理會議錄音」、「轉逐字稿」、「會議紀錄」、「meeting notes」、「整理這場會議」、「拆 action items」，或丟一個 `.m4a` / `.mp3` / `.wav` 給 Claude。

### Setup（一次性）

```bash
# 1. 安裝 mlx-whisper（首次跑會 download ~3GB large-v3 模型）
uv tool install mlx-whisper

# 2. 安裝 ffmpeg
brew install ffmpeg

# 3. 把 transcribe-zh 加到 PATH（plugin 安裝後在 ${CLAUDE_PLUGIN_ROOT}/bin/transcribe-zh）
cp ~/.claude/plugins/meeting-pipeline/bin/transcribe-zh ~/.local/bin/
```

### 兩種 template

| 場次性質 | Template |
|---|---|
| 講者單向傳播 + Demo + Q&A（教學 / 分享 / training） | `training-notes-template.md` |
| 多人對等討論、達成決議、分配任務（sync / planning / retro） | `notes-template.md` |
| 混合場 | training template，把決議併入「關鍵原則」段 |

詳細：[github.com/ericcai0814/claude-meeting-pipeline](https://github.com/ericcai0814/claude-meeting-pipeline)

---

## `ai-coding-workflow`

**團隊標準化 AI 輔助開發工作流程。** 6 個獨立 skill 依任務類型分流 + 4 階段標準流程。

### What it does

把「AI 輔助寫 code」拆成 6 個專業 skill，各自服務單一任務類型（避免 context 互相污染）：

| Skill | 觸發關鍵字 | 做什麼 |
|---|---|---|
| `planning` | 分析需求、建立計畫、技術選型 | 需求分析、架構設計 |
| `frontend` | 設計系統、建立元件、UI、狀態管理 | 元件開發、前端架構 |
| `backend` | API 設計、資料庫、認證、商業邏輯 | API/DB/middleware |
| `validation` | 驗證、測試、整合、E2E | 驗證框架、整合與 E2E 測試 |
| `review` | code review、安全審查、效能審查 | 程式碼/安全/效能審查 |
| `troubleshooting` | bug、錯誤、修復、debug | 踩坑案例、除錯流程 |
| `detect-context` | （由其他 skill 自動調用） | 技術棧/狀態/結構偵測 |

### 四階段流程

每個 skill 都遵循固定 Phase：

```
Phase 1: 任務理解        → 需求重述 + 假設清單 + 提問
Phase 2: 任務規劃        → 執行計畫（WAIT FOR CONFIRMATION）
Phase 3: 任務執行        → 按步驟執行、更新進度
Phase 4: 驗收與交付      → 70% MVP + 產出清單
```

Phase 2 等使用者確認才進 Phase 3——強制人工 checkpoint，避免規劃錯就一路寫到底。

### Hooks

| Hook | 階段 | 做什麼 |
|---|---|---|
| `sensitive-file-guard.js` | PreToolUse | 阻止讀取/修改 `.env`、credentials、keys |
| `markdown-lint.js` | PostToolUse | Markdown 格式檢查（front matter + 標題層級） |

### 與其他兩個 plugin 的關係

- `claude-auto-dev` 的 TDD loop 跟 `ai-coding-workflow` 的四階段不衝突——前者是「寫測試 → 寫實作」的 inner loop，後者是「需求 → 設計 → 實作 → 驗收」的 outer loop。可以同時掛載：用 `/planning` 拆需求出 acceptance criteria，再用 `/auto-dev` 自動跑 TDD。
- `meeting-pipeline` 是獨立 domain（中文會議錄音），跟 ai-coding-workflow 沒交集。

詳細：[github.com/ericcai0814/claude-ai-coding-workflow](https://github.com/ericcai0814/claude-ai-coding-workflow)

---

## Manifest

`.claude-plugin/marketplace.json` 是真正的 marketplace 入口。本 README 僅做說明。

## Why self-hosted

[Anthropic 的 community marketplace](https://claude.ai/settings/plugins/submit) 走 web 審核，曝光高但有 delay。self-hosted marketplace push 後立即可用，適合：

- 個人 / 團隊內部 plugin
- 還沒準備好走社群審核的 alpha plugin
- 想保留快速迭代節奏的場景

兩條路不互斥——個別 plugin 也可以同時提交到 community marketplace，那邊通過後曝光更廣。

## License

各 plugin 各自 LICENSE。本 marketplace catalog 採 MIT。
