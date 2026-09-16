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

## ccache cache 策略（2026-09-16 build 時間差異調查＋修復）

### 調查

- 每次 build 都是全新 runner 上完整編譯 ~692 個 object；ccache 是唯一
  去重層（命中 = 毫秒返回，未命中 = 真編）。
- build 時間差異根因：ccache step 未設 `max-size`，action 預設 **500MB** →
  本地 cache 長期 99% 滿＋build 中持續 LRU 驱逐（單 run 193 cleanups）→
  命中率在 31%–55% 之間飄 → 同源碼 build 時間在 32 min 與 2.5 h 之間擺動。
  命中率差距恰好落在最慢的 CUDA `.cu` TU（188 個，nvcc 單檔幾十秒）。
- 數據：09-12 165/518 hits（2h49m）；09-15 207/669 hits（2h25m）；
  09-16 369/669 hits（32m）。每次 run 排隊時間皆 0（created==started），
  與 runner 資源鬆緊無關。

### 修復

- workflow ccache step 加 `max-size: 4GB`（2026-09-16）。完整 object
  cache ~1GB，4GB 留 4 倍餘裕；job 結尾的 `ccache-clear` 步驟會刪舊的
  timestamped entries，repo 10GB cache 上限不受影響。
- 預期：修復後首次測試 build 仍 ~30 min（restore 的舊 entry 只有
  ~55% 的 object），但該 run 會存下完整 cache；第二次 build 起命中率
  應達 90%+，預期 < 15 min。
- 觀察方式：job 的 `CCache Statistics` summary（Hits/Misses %）與
  `Post ccache` step 的 `Cache size (GB)`（不應再接近上限）。

### 進階選項（未做）

- ccache 4.13.6 支援 http/redis remote storage（`secondary_storage`），
  可跨 repo/機器共享 cache；需自架服務。目前單機使用，本地 4GB＋
  actions/cache 已足夠。
