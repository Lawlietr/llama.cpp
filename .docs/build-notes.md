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

## CUDA 的 CCCL 版本：我們不釘版本（2026-11-05 決定）

- 上游從 `767767850`（2026-10-01，`CUDA: Make CCCL configurable + pin it to 3.4.3 for CI jobs`）開始有
  `GGML_CUDA_CCCL_VERSION`（`ggml/CMakeLists.txt` 的 `CACHE STRING`）。值是 CCCL 的 git tag；
  **留空代表用安裝的 CUDA Toolkit 自帶的 CCCL**。指定時 `ggml/src/ggml-cuda/CMakeLists.txt` 會用
  FetchContent 從 GitHub 抓該 tag 並 `target_link_libraries(ggml-cuda PRIVATE CCCL::CCCL)`。
- 上游 CI 對每一列 matrix 都釘 `-DGGML_CUDA_CCCL_VERSION=v3.4.3`，註解是
  「CTK >= 13.5 bundles CCCL >= 3.5; omit GGML_CUDA_CCCL_VERSION for those versions」。
  上游的 matrix 是 12.4/x64、13.4/x64、13.4/arm64。
- **fork 的決定：不釘。** 我們的 workflow 沒有 `GGML_CUDA_CCCL_VERSION`，也就是走 CUDA 12.4 自帶的 CCCL。
  實證：該開關問世（2026-10-01）之後的每次 sync 與 build 都成功，包括 2026-11-04 那次完整 build。
- 若哪天 CUDA 12.4 自帶的 CCCL 編不過（例如上游開始用只有新版才有的 CCCL API）：在 matrix 那列加回
  `defines: '-DGGML_CUDA_CCCL_VERSION=v3.4.3'`，並在 cmake 行末帶上 `${{ matrix.defines }}`。
  這會讓 configure 階段多一次抓 CCCL 的網路動作（runner 連 GitHub）。
- 舊名 `GGML_CUDA_CUB_3DOT2` 是上面那個 commit 改名前的寫法，上游現在已經不用（`149f5641b` 與
  `5ce06fc71` 都 0 處命中）。它留在我們 workflow 裡只是 CMake 的 unused variable 警告，
  2026-11-05 已從 workflow 移除。

## 手動觸發注意事項

- `gh workflow run`（workflow_dispatch）跑的是 **origin** 上 default branch
  的 workflow 定義，不是本地工作副本。本地領先 origin 時（例如改了
  workflow 還沒 push），先 push——push 本身就會自動觸發 build；直接
  dispatch 只會測到舊配置。
