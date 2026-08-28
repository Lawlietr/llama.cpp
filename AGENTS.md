# AGENTS.md (fork 專用, 取代上游版本)

此 repo 是 ggml-org/llama.cpp 的個人 fork，用途：每天跟上游同步源碼，
並編譯 Windows x64 + CUDA 版本供自己使用。不會向上游提交 PR，
上游 AGENTS.md 的 contribution 規範在此不適用。

## 分支結構

- `master`：上游 master 的鏡像 + 一個 fork files commit（本檔案、
  `.github/workflows/fork-daily-sync.yml`、`fork/` 目錄、刪除
  `.github/workflows/build-cann.yml`）。不要直接在此改源碼。
- `ci-win-cuda-faq`：build 分支，由 `fork-daily-sync.yml` 每天從 master
  重新產生並 force-push（= master + 用 `fork/build-cuda-windows.yml`
  覆蓋 `.github/workflows/build-cuda-windows.yml`）。
  不要在此分支放任何手動 commit，會被下次 sync 覆蓋。
- `fork/build-cuda-windows.yml`：Windows CUDA build 配置的單一來源。
  要改 build 參數（CMake flags、CUDA 版本、產出物）就改這個檔案。

## 每日流程（fork-daily-sync.yml）

- 排程：05:00 GMT+8（cron `0 21 * * *` UTC），schedule 只能跑在 default
  branch（master），所以 sync workflow 必須放 master。
- 步驟：
  1. `git rebase -X ours upstream/master`：把 fork files commit 重放到
     最新上游之上，push master。
  2. 以新 master 建立 `ci-win-cuda-faq`，覆蓋 build workflow，force-push。
  3. 觸發 `build-cuda-windows.yml`（dispatch input `trigger_build=false`
     可只做同步不編譯）。
- 手動觸發：Actions 頁面，或 `gh workflow run fork-daily-sync.yml`
  （預設在 master 跑）。

## Build 需求（目前）

- Windows x64, CUDA 12.4（`windows-2022` runner）
- CMake: `GGML_CUDA=ON`、`GGML_CUDA_FA_ALL_QUANTS=ON`、`GGML_RPC=ON`、
  `GGML_BACKEND_DL=ON`、`GGML_NATIVE=OFF`、`GGML_CPU=OFF`
- 完整 build（非單一 target），artifact zip 含 `build\bin\Release` 的
  全部 `.exe` 與 `.dll`（llama-cli / llama-server / ggml-rpc-server /
  ggml-cuda.dll 等），在 run 頁面下載，預設保留 90 天。
- 不編 ROCm/HIP（上游同 workflow 的 hip job 已移除）。

## 憑證：FORK_SYNC_TOKEN（必要）

GITHUB_TOKEN 無法 push 含 `.github/workflows/` 變更的 commit（GitHub 不允許
GitHub App token 取得 `workflows` scope，這是官方確認的行為），所以
sync workflow 的所有 push 與 build 觸發都改用 repo secret `FORK_SYNC_TOKEN`。

push 認證採 `GIT_ASKPASS`：`actions/checkout` 會把 GITHUB_TOKEN 嵌入
origin remote URL，而 git 對 URL 內嵌 token 的優先級高於 askpass /
credential helper，任何從 origin URL 派生的 push 都會用回 GITHUB_TOKEN。
因此 workflow 用乾淨的 `fork` remote URL + askpass 腳本推送。

建立方式（一勞永逸）：
1. https://github.com/settings/tokens?type=classic 建立 classic PAT
   - Scopes: `repo`、`workflow`（兩者都要勾）
   - 有效期建議設長（如 1 年，到期限流）
   - 注意：不能用 fine-grained PAT。fine-grained token 即使有
     `Actions: Read and write` 也無法 push 含 `.github/workflows/` 變更的
     commit（git 層的 workflow scope 檢查只認 classic token 的 `workflow`
     scope，已實測驗證）。
   - classic token 是帳號級權限（非單一 repo），遺失時先到
     settings/tokens 吊銷。
2. `gh secret set FORK_SYNC_TOKEN` （貼上 token）或 Actions 頁面手動設定

若該 secret 遺失或到期，sync 會掛在 push 步驟，重建並重設即可。

## 維護注意事項

- GitHub Actions 偶爾大規模不穩定，run 失敗且無明顯原因時，先看
  githubstatus.com 再排查自己的設定。
- 若上游修改 `AGENTS.md` 內容，rebase 會以 fork 版為準（`-X ours`），
  屬預期行為。若上游修改了 `build-cann.yml`，rebase 會遇到
  delete/modify 衝突而失敗，需手動處理：master 上重新 `git rm` 該檔
  並 commit，再重跑 sync。
- 本 fork 的 Actions 有時不會註冊純 dispatch 的 workflow 到 index
  （API dispatch 回 404）。build workflow 第一次跑過之後就正常；
  sync workflow 已內建 fallback。
- `build-cann.yml`（昇騰 NPU）已從 fork 移除：它在 push 時觸發且必然
  失敗，與本 fork 需求無關。
