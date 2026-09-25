---
rdq_version: 1
task: LINE 官方帳號群發與 GAS 雲端公告雙向同步
domain: dev
date: 2026-09-25
status: confirmed
telemetry:
  mode: lite
  rounds: 1
  questions: 4
  q4_adopted: 2
  revisions: 0
downstream: self
---

# RDQ 需求規格：LINE 官方帳號與 GAS 雲端公告同步

## 一句話任務
透過 LINE Messaging API 將管理員通知轉發為全體群發，並自動同步登錄至現有「雲端公告」Google 試算表資料庫。

## ✅ 已確認
- **存放目標**：直接寫入現有「雲端公告」家長通知中心試算表
- **帳號權限**：目前尚未開啟 Messaging API，提供建立與取金鑰指引
- **額度防呆**：納入每月免費 200 則訊息發送計算與防超額提示
- **圖文支援**：支援純文字與圖片公告同步轉換

## ❓ 假設（未確認，已採預設值，隨時可推翻）
- **關鍵架構**：採「管理員用個人 LINE 傳送公告給官方帳號 ➔ GAS 自動存入試算表並群發給全體家長」
- **身分驗證**：以管理員個人 LINE `userId` 驗證，防止非管理員觸發群發
- **訊息格式**：純文字轉 LINE Text 格式；圖片自動轉存 Drive 並轉 LINE Image 格式

## ➕ 已採納（象限Ⅳ）
- LINE 官方帳號免費額度計數防呆（每月 200 則提醒）
- 圖片訊息格式轉換支援

## ❌ 排除項（明確不做）
- 不走「LINE 官方後台手動群發 ➔ 監聽 Webhook」路線（LINE 官方架構未開放此事件）
- 暫不實作複雜多輪問答客服機器人

## 📋 一段式需求規格
在現有「**雲端公告**」（`/Users/tunyuan/opencode_0715/雲端公告/Code.gs`）擴充 **LINE Messaging API** 整合。實作 **doPost(e)** 作為 Webhook 端點，接收管理員於個人 LINE 傳送的公告指令；驗證管理員身分後，自動將通知存入 **通知試算表**，並調用 LINE **Broadcast API** 群發至所有好友，同時追蹤每月發送則數防呆，並支援圖文訊息推播。

## ✔ 驗收條件
- [x] 提供啟用 LINE Developers Messaging API 與取得 Channel Access Token 指引
- [ ] `Code.gs` 具備 Webhook 接收、管理員身分驗證、自動寫入試算表功能
- [ ] `Code.gs` 具備調用 Broadcast API 完成全體群發與額度防呆計算
