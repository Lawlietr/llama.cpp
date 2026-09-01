# AGENTS.md (fork 專用, 取代上游版本)

此 repo 是 ggml-org/llama.cpp 的個人 fork，用途：按需與上游同步源碼，
並編譯 Windows x64 + CUDA 版本供自己使用。不會向上游提交 PR，
上游 AGENTS.md 的 contribution 規範在此不適用。

## 分支結構

- `master`：上游 master + 頂端一個 fork files commit。fork commit 內容：
  - 本檔案（取代上游 AGENTS.md）
  - `.github/workflows/build-cuda-windows.yml`（build 配置，dispatch + master push 觸發）
  - 刪除所有上游 workflow（`.github/workflows/` 下只留 `build-cuda-windows.yml`）
- 不要直接在此改源碼；要改 build 設定就改
  `.github/workflows/build-cuda-windows.yml`。

## 手動同步上游

```sh
git fetch upstream
git rebase upstream/master
git push
```

可能的衝突（只有 fork commit 動過的檔案會衝突）：

- `AGENTS.md`：rebase 衝突時保留 fork 版（rebase 時 `theirs` 才是
  fork commit 的版本）：`git checkout --theirs AGENTS.md`
- `.github/workflows/build-cuda-windows.yml`：若上游修改了同名 workflow，
  以上游版本為基礎重新套用 fork 的改動（`workflow_dispatch` + master
  push 觸發、
  單一 CUDA 12.4 x64 matrix、移除 hip job、ccache key 前綴
  `fork-windows-2022-`）。
- 上游 workflow 檔案（`check-vendor.yml`、`docker.yml` 等）：delete/modify
  衝突時直接 `git rm` 維持刪除。

## Build

- 觸發：master 的 push 自動觸發（含本地同步、GitHub web 同步），
  也可手動：Actions 頁面，或
  `gh workflow run build-cuda-windows.yml -R Lawlietr/llama.cpp`。
  注意：master 上任何 push 都會 build，實驗性 commit 先別直接 push 上去。
- 設定：Windows x64, CUDA 12.4（`windows-2022` runner）。
  CMake: `GGML_CUDA=ON`、`GGML_CUDA_FA_ALL_QUANTS=ON`、`GGML_RPC=ON`、
  `GGML_BACKEND_DL=ON`、`GGML_NATIVE=OFF`、`GGML_CPU=ON`（server 需要
  CPU 後端，關掉會報 `no CPU backend found`）。
- 完整 build（非單一 target）。artifact zip 含 `build\bin\Release` 的
  全部 `.exe` 與 `.dll`。必需產物：`llama-cli.exe`、`llama-server.exe`、
  `ggml-rpc-server.exe`（`GGML_RPC=ON` 不可移除）、`ggml-cuda.dll`。
  build 成功後自動發布到 Release
  `win-cuda-12.4-x64-{SHORT}`（含 commit hash 的動態 tag，每次 build 覆蓋，Releases 頁面下載，
  永久保留）。run 頁面的 Artifacts 也會有（90 天後失效）。
- 不編 ROCm/HIP。

## 維護注意事項

- 只執行被明確要求的事，不要自行延伸（建額外管線、下載產物到
  本地、建測試分支等）。要改動前先問。
- Artifacts 與 Release 是兩套機制：Artifacts 是 run 專屬暫存
  （90 天失效）；Release 永久保留，由 workflow 的
  `softprops/action-gh-release` 步驟發布到動態 tag
  `win-cuda-12.4-x64-{SHORT}`（含 commit hash，每次 build 覆蓋）。
- 用 REST API（`/actions/artifacts/{id}/zip`）下載 artifact 會多一層
  GitHub 自己的 zip 包裝；網頁 UI 下載沒有。
- Actions 的 `GITHUB_TOKEN` 無法 push 含 `.github/workflows/` 變更的
  commit（App token 沒有 workflows scope）；但本地 `gh` 的 user token
  可以，所以手動改 workflow 檔案沒問題。
- 曾嘗試每日自動同步管線，因 Actions runner 會劫持 git push 憑證
  （checkout 設 URL-specific credential helper，優先於 askpass / URL
  內嵌 token），且 fine-grained PAT 無法 push workflow 檔案，維護成本
  過高已捨棄。同步一律手動。
- `.github/workflows/` 下只留 `build-cuda-windows.yml`，其他上游 workflow
  已全部刪除（2025-08-31）。同步時若上游新增 workflow，需手動刪除，
  避免 push 觸發一堆無關 CI（如 `check-vendor` 卡在 queued 找不到
  self-hosted runner）。
- GitHub Actions 偶爾大規模不穩定，run 失敗且無明顯原因時，先看
  githubstatus.com 再排查自己的設定。
