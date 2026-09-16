# AGENTS.md (fork 專用, 取代上游版本)

此 repo 是 ggml-org/llama.cpp 的個人 fork，用途：按需與上游同步源碼，
並編譯 Windows x64 + CUDA 版本供自己使用。不會向上游提交 PR，
上游 AGENTS.md 的 contribution 規範在此不適用。

## 分支結構

- `master`：上游 master + 頂端一層 fork commits。fork 自有的檔案：
  - 本檔案（取代上游 AGENTS.md）
  - `.github/workflows/build-cuda-windows.yml`（build 配置，dispatch + master push 觸發）
  - 刪除所有上游 workflow

## 硬規則

- 不要直接改源碼；build 設定改 workflow 檔（build 配置的唯一 source of truth）。
- `.github/workflows/` 下只留 `build-cuda-windows.yml`；上游新增 workflow 就刪除
  （避免 push 觸發無關 CI）。
- master 上任何 push 都會觸發 build，實驗性 commit 先別直接 push。
- 只執行被明確要求的事，不要自行延伸；要改動前先問。

## 同步上游（依明確要求）

- 詳細檢查清單與實戰記錄先
  `ctx_search(queries=["fork sync 衝突解法"], source="llama.cpp-fork-sync")`
  （`.docs/sync-protocol.md`），避免重複推導。
- rebase 前務必備份：`git branch backup/master-<date> master`。
- `git fetch upstream && GIT_EDITOR=true git rebase upstream/master`
- 預期衝突：`build-cuda-windows.yml` content 衝突（上游 bump CUDA 版本）
  → 保留 fork 版（`git checkout --theirs`）；上游 workflow delete/modify → `git rm`。
- 同步後必查 4 項（見 sync-protocol.md）再
  `git push origin master --force-with-lease`（rebase 改寫 history，必須）。

## Build 不變量

- Windows x64、CUDA 12.4 鎖版（上游 bump 不跟）、不編 ROCm/HIP。
- 完整 build（非單一 target）。必需產物：`llama-cli.exe`、`llama-server.exe`、
  `ggml-rpc-server.exe`、`ggml-cuda.dll`。
- `GGML_CPU=ON` 必要（server 需要 CPU 後端，關掉會報
  `no CPU backend found`）。
- 成功自動發布 Release `win-cuda-12.4-x64-{SHORT}`（含 commit hash 的動態 tag，
  每次 build 覆蓋，永久保留）。
- 手動觸發：`gh workflow run build-cuda-windows.yml -R Lawlietr/llama.cpp`。
- 配置細節以 workflow 檔為準，本檔不複製 flag 清單。

## Misc

- run 失敗且無明顯原因 → 先看 githubstatus.com 再排查自己的設定。
- 操作細節（artifact/Release 機制、決策記錄）：`.docs/build-notes.md` /
  `ctx_search(source="llama.cpp-fork-notes")`。
