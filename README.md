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
```

## Available Plugins

| Plugin | Source | What |
|---|---|---|
| [`claude-auto-dev`](https://github.com/EricCaiCai/claude-auto-dev) | `EricCaiCai/claude-auto-dev` | TDD 自動開發循環。Stop Hook 驅動 Red-Green-Refactor，quality-gate 為完成判斷的 source of truth。也提供 npm CLI scaffold。 |
| [`meeting-pipeline`](https://github.com/ericcai0814/claude-meeting-pipeline) | `ericcai0814/claude-meeting-pipeline` | 中文會議錄音 → 結構化紀錄 → 個人 issues。三段式：轉錄/紀錄/拆 issues。**macOS / Apple Silicon only**（mlx-whisper）。 |

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
