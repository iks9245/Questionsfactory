# Questionsfactory

Questionsfactory 是一個用來訓練「提問能力」的單頁 MVP。

這個項目相信一件事：在 AI 盛行的時代，答案會變得越來越容易取得，但真正決定思考品質的，仍然是問題本身。

## 項目目標

Questionsfactory 先訓練使用者判斷什麼是好問題，再逐步練習把普通問題升級成更清晰、更有行動價值的問題。

目前的學習循環是：

1. 判斷四個選項中哪一個問題品質最高
2. 看見好問題背後的評分維度
3. 練習改寫較弱的問題
4. 透過固定題庫或 LLM 生成題庫持續練習

## 核心維度

題目會圍繞六個問題品質維度：

- 目的：問題是否知道自己要解決什麼
- 情境：是否提供足夠背景
- 邊界：是否收斂範圍與限制
- 深度：是否能推動分析，而不是只要表面答案
- 行動：是否能導向下一步
- 假設：是否能暴露前提與盲點

## 功能

- 單一 `index.html`，不需要打包或後端
- 內建離線起始題庫
- 支援一次生成 10 至 30 條 LLM 題庫
- 支援雲端 LLM：
  - OpenAI Responses
  - Google Gemini
  - Anthropic Claude
  - OpenRouter
  - OpenAI-compatible endpoints
- 支援本機 LLM：
  - OMLX / MLX server
  - Local OpenAI-compatible servers
  - Ollama native API
- 本地分數校準，降低「答案越長分數越高」這類明顯破綻
- 使用瀏覽器 `localStorage` 保存練習進度

## 快速開始

直接用瀏覽器打開：

```text
index.html
```

如果想用本機 web server 測試：

```bash
python3 -m http.server 8765 --bind 127.0.0.1
```

然後打開：

```text
http://127.0.0.1:8765/index.html
```

## 接入 Local LLM

如果使用 OMLX / MLX server，可以在頁面中選擇：

```text
Provider: OMLX / MLX server
Endpoint: http://127.0.0.1:11123/v1
```

頁面會自動把 base URL：

```text
http://127.0.0.1:11123/v1
```

轉成 OpenAI-compatible chat completions 路徑：

```text
http://127.0.0.1:11123/v1/chat/completions
```

如果本機服務要求 API key，請填入 API key。若 `curl` 可以成功但瀏覽器失敗，通常是本機模型服務沒有開啟 CORS。

## 題庫品質

LLM 生成題目時，頁面會要求模型產生四個都合理的選項，而不是一眼就能看出高低的答案。

系統會盡量避免：

- 最佳答案永遠最長
- 分數差距過大
- 干擾選項明顯很弱
- 題目只是在考文字長度，而不是問題品質

如果模型生成的題目結構有效但不夠理想，頁面會保留可用題目並做本地分數校準，而不是直接把整批題庫丟掉。

## 隱私說明

API key 是在瀏覽器端輸入，並由瀏覽器直接送到你選擇的模型服務。

預設不會保存 API key。只有在勾選「記住 API key」時，key 才會以純文字存在這台瀏覽器的 `localStorage`。

如果要公開部署或給團隊使用，建議改成後端 proxy，由後端保管 API key。

## 目前狀態

這是一個 MVP，重點是先驗證「判斷問題品質」這個核心訓練循環。

後續可以加入：

- 使用者帳號與學習紀錄
- 共享題庫
- 模型評改改寫答案
- 團隊練習數據
- 更完整的問題力課程結構
