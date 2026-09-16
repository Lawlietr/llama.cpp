# llama.cpp fork build / 操作備註

> 非每次 session 都需要的操作細節；AGENTS.md 只放不變量。需要時直接讀本檔，
> 或 `ctx_search(source="llama.cpp-fork-notes")` 搜尋。

## Build 產物內容

- artifact zip 含 `build\bin\Release` 的全部 `.exe` 與 `.dll`。
- 必需產物：`llama-cli.exe`、`llama-server.exe`、`ggml-rpc-server.exe`
  （對應 `GGML_RPC=ON`，不可移除）、`ggml-cuda.dll`。

## Artifacts vs Release（兩套機制）

- Artifacts：run 專屬暫存，90 天失效。用 REST API
  （`/actions/artifacts/{id}/zip`）下載會多一層 GitHub 自己的 zip 包裝；
  網頁 UI 下載沒有。
- Release：永久保留。由 workflow 的 `softprops/action-gh-release` 步驟發布到
  動態 tag `win-cuda-12.4-x64-{SHORT}`（含 commit hash，每次 build 覆蓋）。

## 為何同步一律手動（決策記錄）

- Actions 的 `GITHUB_TOKEN` 無法 push 含 `.github/workflows/` 變更的 commit
  （App token 沒有 workflows scope）；本地 `gh` 的 user token 可以，
  所以手動改 workflow 檔案沒問題。
- 曾嘗試每日自動同步管線：Actions runner 會劫持 git push 憑證
  （checkout 設 URL-specific credential helper，優先於 askpass / URL
  內嵌 token），且 fine-grained PAT 無法 push workflow 檔案，維護成本
  過高已捨棄（2025-08-31）。
- 結論：同步一律手動。上游新增 workflow 需手動刪除，留下會讓 push
  觸發一堆無關 CI（如 `check-vendor` 卡在 queued 找不到 self-hosted runner）。
