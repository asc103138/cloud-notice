# 雲端公告（家長通知中心）- AGENTS.md

## 專案資訊
- **專案名稱**：雲端公告（家長通知中心）
- **專案用途**：官方帳號通知的長期保存與查閱系統，支援行動載具直覺瀏覽，並整合 LINE 官方帳號群發雙向同步。
- **主要工作目錄**：`/Users/tunyuan/opencode_0715/雲端公告`

## Obsidian 關聯筆記
- **Vault 路徑**：`/Users/tunyuan/opencode_0715`
- **專案駕駛艙**：`/Users/tunyuan/opencode_0715/04-專案/雲端公告-專案駕駛艙.md`

## 工作與安全規則
- 回應使用繁體中文（台灣）。
- **開工流程**：讀取本檔、讀取 `handoff.md`、讀取 Obsidian 專案駕駛艙、檢查 `git status`。
- **收工流程**：資安掃描、更新 Obsidian 駕駛艙、更新 `handoff.md`、精準 stage 並經確認後 commit。
- **資安與個資規範**：
  - 嚴禁將 LINE Channel Access Token 或管理員 LINE User ID 寫死於程式碼中，必須儲存於 GAS Script Properties。
  - 學生資料僅記錄代號或座號，不儲存真名。
