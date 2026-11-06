# llama.cpp fork 同步協議（實戰版）

> 來源：`AGENTS.md` 的延伸實戰版。AGENTS.md 保持精簡；本檔記錄詳細檢查、
> 衝突解法與每次同步的踩雷。sync 前先用 `ctx_search(queries=[...], source="llama.cpp-fork-sync")` 撈本檔。

## 何時同步

依使用者明確要求才做（「同步上游 / sync」）。不要自己觸發。

## 同步前準備（rebase 前必做，省掉臨時補步驟）

```sh
# 0. 確認 upstream remote 存在（fork 必須指向 ggml-org/llama.cpp）
#    若 repo 原本只設 origin，這裡才會失敗 → 補上
git remote get-url upstream >/dev/null 2>&1 \
  || git remote add upstream https://github.com/ggml-org/llama.cpp.git

# fork 點 = origin/master 與 upstream/master 的 merge-base
# 不要用 fork commit 當基點，否則預判會漏判。下文 <fork點> 都用這個
FORK_BASE=$(git merge-base origin/master upstream/master)
```

## 同步前預判（省掉重複的 git 檢查）

rebase 前先用下面三條確認，通常可乾淨通過：

```sh
# 1. 上游有沒有改 AGENTS.md / build-cuda-windows.yml？（空 = 不會衝突）
git log <fork點>..upstream/master -- AGENTS.md .github/workflows/build-cuda-windows.yml

# 2. 上游有沒有「新增」workflow？（fork 刪除清單要蓋到全部）
#    用動態 fork 點，不要硬編碼 093a2f86c
git ls-tree -r --name-only <fork點> -- .github/workflows/    # fork 點
git ls-tree -r --name-only upstream/master -- .github/workflows/  # 現上游
# 用 comm -13 比較，右列多出來 = 新增、需手動刪

# 3. fork 已刪、上游仍有的 workflow，若上游也改過 → rebase 會 modify/delete 衝突
#    （需 git rm）。這是最容易漏判、也最花時間的一條，務必跑
git ls-tree -r --name-only upstream/master -- .github/workflows/ | sort > /tmp/up_wf
git ls-tree -r --name-only  origin/master  -- .github/workflows/ | sort > /tmp/for_wf
comm -23 /tmp/up_wf /tmp/for_wf | while read f; do
  git log "<fork點>..upstream/master" --format=%h -- "$f" | grep -q . \
    && echo "WARN modify/delete 風險: $f （上游有改，rebase 需 git rm）"
done
```

- 上游通常**不改** `AGENTS.md`，但**會 bump** `build-cuda-windows.yml` 的 CUDA
  版本（如 `0ecb159c9` bump 到 13.4.1）→ 該檔 content 衝突是預期的，保留 fork
  版（fork 鎖 12.4）。
- 上游的 `build-cuda-windows.yml` 對 matrix 每一列都帶 `defines: '-DGGML_CUDA_CCCL_VERSION=v3.4.3'`
  （上游 matrix 是 12.4/x64、13.4/x64、13.4/arm64）。**不跟**：fork 只有 12.4/x64 一列且不釘 CCCL 版本，
  用 CUDA Toolkit 自帶的 CCCL（決定與實證見 `.docs/build-notes.md`）。解衝突時不要把上游的 `defines:` 行帶回來。
- 預期衝突：(1) `build-cuda-windows.yml` content 衝突 → 保留 fork 版；
  (2) 刪除 upstream workflow 的 **delete/modify**，直接 `git rm`。
- **rebase 前務必備份**：`git branch backup/master-<date> master`。
- push 需 `--force-with-lease`（rebase 改寫 history，非 fast-forward）。

## 同步 procedure

```sh
git fetch upstream
GIT_EDITOR=true git rebase upstream/master   # 遇衝突停下來解
# 解完：git rm <倖存上游workflow> ; git rebase --continue
git push origin master --force-with-lease
```

### 衝突解法（依 AGENTS.md）

- `AGENTS.md` 衝突 → 保留 fork 版：`git checkout --theirs AGENTS.md`
  （rebase 時 `theirs` 才是 fork commit 版本）。
- `build-cuda-windows.yml` 衝突 → 保留 fork 版（單一 cuda job、CUDA 12.4/x64、
  無 hip、`GGML_CPU=ON`）。
- 上游 workflow（`check-vendor.yml`、`docker.yml`、…）delete/modify →
  `git rm` 維持刪除。
  - **modify/delete 衝突（`... deleted in <fork commit> and modified in HEAD`）是
    預期內**，不是 unexpected：代表上游也改過這個 fork 要刪的 workflow。
    直接 `git rm <檔>` 即可，不要當錯誤停下來排查。

## 同步後必查（push 前逐項）

1. `.github/workflows/` 下**只剩** `build-cuda-windows.yml`。若有上游 workflow
   倖存（delete/modify 被 3-way merge 解成保留修改），`git rm` 後
   `git rebase --continue`。
2. `git merge-base --is-ancestor upstream/master master` 通過。
3. `build-cuda-windows.yml` 為 fork 版：單一 `cuda` job、無 `hip`、CUDA 12.4/x64
   單一 matrix、`GGML_CPU=ON`。
4. `AGENTS.md` 為 fork 版（首行 `# AGENTS.md (fork 專用...`）。

## 實戰記錄（past sync log）
### 2026-11-06（sync to upstream 3ed7284f2）

- fork 點 `5ce06fc71`（上次 sync 點，tag `b12122`），上游 47 commits 至 `3ed7284f2`（tag `b12169`，range 內 34 個 tag）。
  上游 tip 是 `ci : bump CUDA 13 release builds to 13.5 (#31788)`。
- 三條預判：上游**未改** `AGENTS.md` / `.docs/`、**未新增** workflow（`comm -13` 空輸出），
  但**改過 5 個 fork 已刪的 workflow** → 預判 modify/delete：`build-cpu.yml`、`docker.yml`、
  `models-check.yml`、`release-publish.yml`、`release.yml`。`build-cuda-windows.yml` 有 version bump
  （13.x matrix 改 13.5）→ content 衝突預期，保留 fork 版（鎖 12.4）。
- rebase 37 個 fork commits：`build-cuda-windows.yml` content 衝突保留 fork 版；5 個 modify/delete
  全部 `git rm`（維持刪除）。一次迴圈 `git rm` 未合併檔案後 `git rebase --continue` 接完，無額外衝突。
- 上游本版**無版本 bump**（`bump version` / `GGML_VERSION` grep 無命中），不影響 build 配置。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`（1 個檔）；`upstream/master` 為 ancestor；
  `build-cuda-windows.yml` 單一 matrix 僅 `cuda: '12.4'` + `arch: x64`、`hip|rocm` 0 命中、
  `-DGGML_CPU=ON` 在第 65 行；`AGENTS.md` 為 fork 版。`AGENTS.md` / `.docs/*` 與同步前 `git diff --stat` 空輸出。
- push `3119f6daa`（`--force-with-lease`）。備份：`backup/master-pre-sync-20261106`（`cb8450583`）。
- Actions run 正常觸發（run `42194342918`，head_sha 對得上新 HEAD，`in_progress`）。

### 2026-11-05（sync to upstream 5ce06fc71）
- fork 點 `149f5641b`（上次 sync 點，tag `b12098`），上游 24 commits 至 `5ce06fc71`（tag `b12122`，range 內 15 個 tag）。
  上游 tip 是 `CUDA: fuse NVFP4 gate/up matmuls and GLU in MMQ (#31296)`。
- 三條預判：上游**未改** `AGENTS.md` / `build-cuda-windows.yml` / `.docs/`；**未新增** workflow（兩邊都 63 個，
  `comm -13` 空輸出）；但上游**改過 3 個 fork 已刪的 workflow** → 預判 modify/delete：`build-cmake-pkg.yml`、
  `build-sanitize.yml`、`ci-self-hosted-cpu.yml`。rebase 實際就是這 3 處，沒有額外衝突。
- rebase 35 個 fork commits：停在 `711e79c58`（`Remove upstream workflows, keep only build-cuda-windows.yml`）解 2 檔，
  停在 `2121a2447` 解 1 檔，其餘一次通過。全部 `git rm`（維持刪除）。
- **本次操作失誤（記錄以免重犯）**：解完衝突後，我在 shell 迴圈裡重複叫 `git rebase upstream/master`，
  而不是 `git rebase --continue`。git 只印 `rm -fr .git/rebase-merge and run me again` 後退出、**不做任何事**，
  迴圈空轉 11 輪；repo 停在衝突中途（HEAD detached 在 `ca6940f0b`，也就是尚未套用「刪 workflow」那個 commit 的位置），
  沒有任何損害，改用 `git rebase --continue` 就接回去了。
  正確寫法：先判斷 `.git/rebase-merge` 或 `.git/rebase-apply` 是否存在，存在就只叫 `git rebase --continue`，
  絕不再叫 `git rebase <branch>`。
- 上游本版**有 version bump**：llama.cpp `0.9.0`（`cd959bae9`）、ggml `0.29.0`（`cca14f64f`；`ggml/CMakeLists.txt`
  本版唯一改動是 `GGML_VERSION_MINOR 28 -> 29`）→ 不影響 build 配置，CUDA 照舊鎖 12.4。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`（1 個檔）；`upstream/master` 為 ancestor；
  `build-cuda-windows.yml` / `AGENTS.md` / `.docs/*` 與同步前 `git diff --stat` 空輸出；workflow 內容為單一 `cuda` job、
  matrix 僅 `cuda: '12.4'` + `arch: x64`、`hip|rocm` 0 命中、`-DGGML_CPU=ON` 在第 66 行。
  整體規模：111 files changed, 5269 insertions(+), 938 deletions(-)。
- 產物驗證方式更正：`Pack artifacts` 是 `7z a ... build\bin\Release\*.exe build\bin\Release\*.dll`，
  `ggml-cuda.dll` 靠 glob 進 zip，**不會**在 workflow 以檔名出現 → 用「workflow 裡有沒有那個檔名」驗產物會誤判。
- 開關查證：`LLAMA_BUILD_BORINGSSL` 定義在 `vendor/cpp-httplib/CMakeLists.txt`（不在 `ggml/CMakeLists.txt`），
  只 grep ggml 那邊會誤判成不存在；上次同步點與現在都是 25 處，未變。
- 已移除的死參數：`matrix.defines` 的 `-DGGML_CUDA_CUB_3DOT2=ON`（連同 CMake 行的 `${{ matrix.defines }}`）。
  這個名字在上游 `149f5641b` 與 `5ce06fc71` 都 0 處命中，是上游 `767767850`（2026-10-01，
  `CUDA: Make CCCL configurable + pin it to 3.4.3 for CI jobs`）改名後的殘留，我們是從 fork 時的舊上游
  workflow 抄來的。移除後行為不變：那個 commit 在上次同步點之前就在，而 11-04 那次成功的 build 本來就沒釘 CCCL。
  上游的替代品與我們的決定見 `.docs/build-notes.md`。
- push `--force-with-lease`。備份：`backup/master-pre-sync-20261105`（`f9ae2f8e1`）。


### 2026-11-04（sync to upstream 149f5641b）

- fork 點 `2cbca389f`（上次 sync 點），上游 35 commits（至 `149f5641b`，tags 到 b12090）。
  三條預判：上游**未改** `AGENTS.md` / `build-cuda-windows.yml` / `.docs/`（pre-check
  `git diff --stat` 空輸出）、**未改**任何既有 workflow（無 modify/delete 風險）、
  但**新增** 1 個 workflow `pr-video-demo-label.yml`（pre-check `--name-status` 唯一的 `A`）。
- rebase 33/33 一次通過，零衝突（本輪 fork commits 少、上游未碰重疊檔案）。
- 新增的上游 workflow 於 rebase 後 `git rm`，補一個 commit
  `sync: remove upstream pr-video-demo-label.yml (added after fork base)`。
- 上游本版無版本 bump（`bump version` grep 無命中），不影響 build 配置。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`；`upstream/master` 為 ancestor；
  `build-cuda-windows.yml` / `AGENTS.md` / `.docs/*` 與同步前 `git diff --stat` 空輸出；
  workflow 內容：單一 `cuda` job、matrix 僅 `cuda: '12.4'` + `arch: x64`、無 hip/rocm、
  `GGML_CPU=ON`。6 個 CMake 開關在 `ggml/CMakeLists.txt` 全存在。
- push（`--force-with-lease`）。備份：`backup/master-pre-sync-20261104`。

### 2026-11-03（sync to upstream 2cbca389f）

- fork 點 `785322d17`（上次 sync 點），上游 30 commits（`1c5ed702c`…`2cbca389f`）。
  三條預判：上游**未改** `AGENTS.md` / `build-cuda-windows.yml`、**未新增** workflow
  （fork 點與現上游的 `.github/workflows/` 清單完全相同）、無 modify/delete 風險。全部命中
  —— rebase 32/32 一次通過，無衝突。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`；`upstream/master` 為 ancestor；
  `build-cuda-windows.yml` / `AGENTS.md` / `.docs/*` 與同步前 `git diff --stat` 空輸出（fork
  修改完整保留）；workflow 內容：單一 `cuda` job、matrix `include` 僅 `cuda: '12.4'` + `arch: x64`、
  無 hip/rocm、`GGML_CPU=ON`。
- push `f2713786c`（`--force-with-lease`）。備份：`backup/master-pre-sync-20261103`。
- run #41574694841 in_progress，jobs 只有一個 `windows-cuda (12.4, x64)` → 符合「actions 只有一個編譯工作」。

### 2026-11-01（sync to upstream 785322d17）

- fork 點 `a91eab67d`（上次 sync 點），上游 3 commits（`a91eab67d`…`785322d17`）。
  三條预判：上游**未改** `AGENTS.md` / `build-cuda-windows.yml`、**未新增** workflow、
  無 modify/delete 風險。全部命中 —— rebase 一次通過（31/31），無衝突。
- 三個 upstream commit 只碰 `ggml/src/ggml-hexagon`、`ggml/src/ggml-sycl/CMakeLists.txt`、
  `requirements/*.txt`，均與 fork 修改無關。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`；`upstream/master` 為 ancestor；
  `build-cuda-windows.yml` / `AGENTS.md` 與同步前 blob identical。workflow 內容：單一 `cuda` job、
  matrix 僅 `cuda: '12.4'` + `arch: x64`、無 hip/rocm、`GGML_CPU=ON`。
- push `94cf780ae`（`--force-with-lease`）。備份：`backup/master-pre-sync-20261101-fresh`。

### 2026-11-01（sync to upstream a91eab67d）

- fork 點 `3c809f443`（上次 sync 點），上游 26 commits（`e0f142d5d`…`a91eab67d`），
  fork 端 29 commits。三條預判：上游**未改** `AGENTS.md` / `build-cuda-windows.yml` /
  `.docs/`（第一條空輸出）；**新增** 1 個 workflow（`build-apple-daily.yml`，來自
  `e216f4ec8` ci : extract most apple jobs into daily workflow）→ rebase 後手動 `git rm`；
  第三條預警命中 3 個 modify/delete 風險（`behavior-check.yml`、`build-apple.yml`、
  `build-apple-daily.yml`）。
- rebase 停在 `678a7394a`（Remove upstream workflows）：1 個 modify/delete 衝突
  → `git rm .github/workflows/build-apple.yml`。
- rebase 停在 `d8ae149fb`（23/29，remove behavior-check.yml）：1 個 modify/delete 衝突
  → `git rm .github/workflows/behavior-check.yml`。其後一次通過。
- **教訓**：解衝突後要用 `git rebase --continue`，**不要**再跑一次 `git rebase
  upstream/master`。迴圈裡誤跑 40 次；結果只是重複對同一個 commit 做 3-way merge
  （`msgnum` 沒前進、`git rm` 的暫存刪除被保留），沒有破壞 history，檢查
  `.git/rebase-merge/msgnum`/`end` 與 `HEAD` 就能確認。改寫成外層只跑一次 rebase、
  之後一律 `--continue`。
- 上游本版含 `cmake : embed an app manifest in all Windows exes`（`d63fbfe79`）與
  `common : drop the Windows argv reconstruction`（`aa588a346`），都是 Windows 相關但
  不影響 build 配置。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`（`build-apple-daily.yml` 已刪）；
  `upstream/master` 為 ancestor；`build-cuda-windows.yml` / `AGENTS.md` / `.docs/*` 與同步前
  `git diff --stat` 空輸出（fork 修改完整保留）；workflow 內容：單一 `cuda` job、matrix
  `include` 僅 `cuda: '12.4'` + `arch: x64`、無 hip/rocm、`GGML_CPU=ON`
  （`ggml/CMakeLists.txt:189`）。
- 額外檢查：workflow 用到的 6 個 CMake 開關（`GGML_BACKEND_DL`/`GGML_NATIVE`/`GGML_CPU`/
  `GGML_CUDA`/`GGML_CUDA_FA_ALL_QUANTS`/`GGML_RPC`）在上游新版 `ggml/CMakeLists.txt`
  全存在未改名 → 不需調 flag。
- push（`--force-with-lease`）。備份：`backup/master-pre-sync-20261101`。

### 2026-10-29（sync to upstream 2e0d86beb）

- fork 點 `8ca4060b7`（上次 sync 點），上游 30 commits（`6886628cb`…`2e0d86beb`），
  fork 端 27 commits。三條預判：上游**未改** `AGENTS.md` / `build-cuda-windows.yml`、
  **未新增** workflow；第三條預警命中 —— 上游 `ff0d396e3`（ci s390x Ubuntu 26.04）
  改了 3 個 fork 已刪的 workflow（`build-ibm.yml`、`release.yml`、`release-publish.yml`）。
- rebase 停在 `81e6113ac`（7/27）：2 個 modify/delete 衝突 → `git rm
  .github/workflows/build-ibm.yml .github/workflows/release.yml`。
- rebase 停在 `3765ad2a7`（19/27）：1 個 modify/delete 衝突 →
  `git rm .github/workflows/release-publish.yml`（注意同一 commit 也改 `AGENTS.md` /
  `.docs/sync-protocol.md`，`git rm` 只處理 workflow 那一個路徑，其餘照原樣帶上）。
  其後 commits 一次通過。
- 上游本版有 `llama.cpp : bump version to 0.8.0`（`8642541b2`）與 `ggml : bump v.28.0`
  （`ec941ee85`），不影響 build 配置。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`；`upstream/master` 為 ancestor；
  `build-cuda-windows.yml` / `AGENTS.md` / `.docs/*` 與同步前 `git diff --stat` 空輸出
  （blob identical）；workflow 內容：單一 `cuda` job、matrix 僅 `cuda: '12.4'` + `arch: x64`、
  無 hip/rocm、`GGML_CPU=ON`（`ggml/CMakeLists.txt:189` 仍是 option）。
- 額外檢查：workflow 用到的 6 個 CMake 開關（`GGML_BACKEND_DL`/`GGML_NATIVE`/`GGML_CPU`/
  `GGML_CUDA`/`GGML_CUDA_FA_ALL_QUANTS`/`GGML_RPC`）在上游新版 `ggml/CMakeLists.txt` 全存在，
  未改名 → 不需調 flag。
- push（`--force-with-lease`）。備份：`backup/master-pre-sync-20261029`。

### 2026-10-29（sync to upstream 3c809f443）

- fork 點 `2e0d86bebc`（上次 sync 點），上游 28 commits（`7ac5727d6`…`3c809f443`）。
  三條预判：上游**未改** `AGENTS.md` / `build-cuda-windows.yml`（`git log $FORK_BASE..upstream/master -- <path>` 空）、**新增** 多個 workflow（ai-issues.yml、bench.yml.disabled、build-cuda-ubuntu.yml 等，fork 刪除 commit 蓋不到 → rebase 後需 `git rm`）、第 3 條預警 fork 已刪 workflow 的 modify/delete 風險。
- rebase 停在 commit `b64b9a3f6`（sync: remove upstream behavior-check.yml）：1 個 modify/delete 衝突 → `git rm .github/workflows/behavior-check.yml` 維持刪除，其後 fork commits 一次通過。
- rebase 完成後 `.github/workflows/` 只剩 `build-cuda-windows.yml`（fork 的刪除 commit 蓋到了所有新增/既有 upstream workflow）。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`；`upstream/master` 為 ancestor；`build-cuda-windows.yml` / `AGENTS.md` 與同步前 blob identical（fork 修改完整保留）。workflow 內容：單一 `cuda` job、matrix `include` 僅 `cuda: '12.4'` + `arch: x64`、無 hip/rocm、`GGML_CPU=ON`。
- push `9da4f334d`（`--force-with-lease`）。備份：`backup/master-pre-sync-20261029-fresh`。

### 2026-10-28（sync to upstream 8ca4060b7）

- fork 點 `7aca0d2ea`（上次 sync 點），上游 33 commits（`1a97e387d`…`8ca4060b7`）。
  三條預判：上游**未改** `AGENTS.md` / `build-cuda-windows.yml`、**未新增** workflow、
  第 3 條預警 2 個 fork 已刪、上游也改過的 workflow modify/delete 風險。
- rebase 停在 commit `42268b6e0`（刪除 upstream workflows，7/26）：1 個 modify/delete
  衝突 → `git rm .github/workflows/release.yml`，其後刪除多個 upstream workflow 的 commit
  一次通過。
- rebase 停在 commit `aaf9dd28b`（sync: remove upstream ci-self-hosted workflows，18/26）：
  1 個 modify/delete 衝突 → `git rm .github/workflows/ci-self-hosted-webgpu.yml` 維持刪除，
  其後 fork commits 一次通過。
- rebase 完成後 `.github/workflows/` 只剩 `build-cuda-windows.yml`（fork 的刪除 commit
  蓋到了所有新增/既有 upstream workflow，含上游在 fork 點之後新增的 behavior-check.yml）。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`；`upstream/master` 為 ancestor；
  `build-cuda-windows.yml` / `AGENTS.md` 與同步前 blob identical（`git diff --stat backup/... master -- <those paths>` 空輸出）。workflow 內容：單一 `cuda` job、matrix `include` 僅 `cuda: '12.4'` + `arch: x64`、無 hip/rocm、`GGML_CPU=ON`。
- push（`--force-with-lease`）。備份：`backup/master-pre-sync-20261028`。

### 2026-02-14（sync to upstream 4c9233c03）

- fork 點 `093a2f86c`。上游自 fork 點後**未改** `AGENTS.md` /
  `build-cuda-windows.yml`，**未新增** workflow → 預判全綠。
- rebase 在 commit `b088117ad`（刪除 upstream workflow）停：4 個 delete/modify
  衝突 → `git rm`：
  - `.github/workflows/build-ibm.yml`
  - `.github/workflows/build-self-hosted.yml`
  - `.github/workflows/fusion.yml`
  - `.github/workflows/release.yml`
- `build-cuda-windows.yml` 乾淨套上（上游未改）。
- 結果：`.github/workflows/` 只剩 `build-cuda-windows.yml`；上游為 master
  ancestor；fork commits 完整保留在頂端。
- 備份：`backup/master-pre-sync-20260214`。

### 2026-09-16（sync to upstream 930e2fa59）

- fork 點 `4c9233c03`。上游**未改** `AGENTS.md`；**改了**
  `build-cuda-windows.yml`（`0ecb159c9` bump Windows CUDA build 到 13.4.1）；
  **未新增** workflow。
- rebase 停在 commit `dec21e3cf`（1/12）：`build-cuda-windows.yml` content
  衝突 → `git checkout --theirs` 保留 fork 版（fork 鎖 12.4）。
- rebase 停在 commit `23e649aeb`（5/12）：4 個 delete/modify 衝突 → `git rm`：
  - `.github/workflows/build-cuda-ubuntu.yml`
  - `.github/workflows/build-self-hosted.yml`
  - `.github/workflows/release.yml`
  - `.github/workflows/server-self-hosted.yml`
- 同步後必查 4 項全過。備份：`backup/master-pre-sync-20260916`。
- push `2baaff2e5`（`--force-with-lease`）；觸發的 run 只有 1 個
  `fork-windows-cuda`。

### 2026-10-27（sync to upstream 7aca0d2ea）

- fork 點 `d854d98ba`（上次 sync 點），上游 24 commits（`48c36110a`…`7aca0d2ea`）。
  三條預判：上游**未改** `AGENTS.md` / `build-cuda-windows.yml`、**新增** workflow
  `.github/workflows/behavior-check.yml`（`4244ad7ac` ci : add windows behavior check
  #31191）→ 第 3 條預判如實預警 fork 已刪的 workflow modify/delete 風險。
- rebase 停在 commit `7db847a8a`（刪除 upstream workflows，7/25）：1 個 modify/delete
  衝突 → `git rm .github/workflows/build-apple.yml`，其後刪除多個 upstream workflow
  的 commit 一次通過。
- rebase 停在 commit `14d9047f1`（sync: remove upstream behavior-check.yml，23/25）：
  1 個 modify/delete 衝突 → `git rm .github/workflows/behavior-check.yml` 維持刪除，
  其後 2 個 fork commits 一次通過。
- rebase 完成後 `.github/workflows/` 只剩 `build-cuda-windows.yml`（fork 的刪除 commit
  蓋到了所有新增/既有 upstream workflow，含上游在 fork 點之後新增的 behavior-check.yml）。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`；`upstream/master` 為 ancestor；
  `build-cuda-windows.yml` / `AGENTS.md` 與同步前 blob identical（fork 修改完整保留）。
  workflow 內容：單一 `cuda` job、matrix `include` 僅 `cuda: '12.4'` + `arch: x64`、
  無 hip/rocm、`GGML_CPU=ON`。
- push `c6605cded`（`--force-with-lease`）。備份：`backup/master-pre-sync-20261027`。

### 2026-10-26（sync to upstream d854d98ba）

- fork 點 `7790a9231`（上次 sync 點），上游 9 commits。三條預判：上游**未改**
  `AGENTS.md` / `build-cuda-windows.yml`、**未新增** workflow、**未改** workflow
  → 預判全綠，無 workflow delete/modify 風險。
- rebase 24/24 一次通過，無衝突。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`；`upstream/master` 為
  ancestor；`build-cuda-windows.yml` / `AGENTS.md` 與同步前 blob identical（fork
  修改完整保留）。jobs 下僅單一 `cuda` job → 符合「actions 只有一個編譯工作」。
- push `c25dafb46`（`--force-with-lease`）。備份：`backup/master-pre-sync-20261026`。

### 2026-10-25（sync to upstream 7790a9231）

- fork 點 `36e36e63a`（上次 sync 點），上游 40 commits。三條預判：上游**未改**
  `AGENTS.md` / `build-cuda-windows.yml`、**未新增** workflow；第 3 條預警 7 個
  fork 已刪、上游也改過的 workflow modify/delete 風險。
- rebase 停在 commit `692e38bec`（刪除 upstream workflows，8/23）：3 個
  modify/delete 衝突 → `git rm`：
  - `.github/workflows/build-apple.yml`
  - `.github/workflows/build-cmake-pkg.yml`
  - `.github/workflows/build-sanitize.yml`
- rebase 停在 commit `a96efd911`（sync: remove upstream ci-self-hosted workflows，
  17/23）：4 個 modify/delete 衝突 → `git rm`：
  - `.github/workflows/ci-self-hosted-cpu.yml`
  - `.github/workflows/ci-self-hosted-metal.yml`
  - `.github/workflows/ci-self-hosted-vulkan.yml`
  - `.github/workflows/ci-self-hosted-webgpu.yml`
- 其後 6 個 fork commits 一次通過。rebase 完成後 `.github/workflows/` 只剩
  `build-cuda-windows.yml`（fork 的刪除 commit 蓋到了所有新增/既有 upstream workflow）。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`；`upstream/master` 為
  ancestor；`build-cuda-windows.yml` / `AGENTS.md` 與同步前 blob identical（fork
  修改完整保留）。workflow 內容：單一 `cuda` job、matrix `include` 僅 `cuda: '12.4'`
  + `arch: x64`、無 hip/rocm、`GGML_CPU=ON`。
- push `d35ca6105`（`--force-with-lease`）。備份：`backup/master-pre-sync-20261025`。
- run `fork-windows-cuda` in_progress（headSha `d35ca6105`），**jobs 只有一個**
  `windows-cuda (12.4, x64)` → 符合「actions 只有一個編譯工作」。

### 2026-10-24（sync to upstream 36e36e63a）

- fork 點 `8baabd53b`，上游 55 commits，fork 端 22 commits。預判：上游**未改**
  `AGENTS.md` / `build-cuda-windows.yml`；**新增** workflow `.github/workflows/
  behavior-check.yml`（`f00c9083b` ci : add behavior check workflow #31036，同一個
  commit 也改了 `build-cpu.yml`）→ 第 3 條預判如實預警 `build-cpu.yml` 的
  modify/delete 風險。
- rebase 停在 commit `0c70c61aa`（刪除 upstream workflows，8/22）：**唯一衝突**
  = `.github/workflows/build-cpu.yml` UD（modify/delete）→ `git rm` 維持刪除，
  其後 14 個 fork commits 一次通過。
- rebase 完成後 `.github/workflows/` 出現 `behavior-check.yml`（上游在 fork 點之後
  新增，fork 的刪除 commit 蓋不到）→ 另補一個刪除 commit
  `sync: remove upstream behavior-check.yml (added after fork base)`。**這是常態
  步驟，不是踩雷**：每逢上游新增 workflow 就要補這一刀。
- 必查 4 項全過：workflows 只剩 `build-cuda-windows.yml`；`upstream/master` 為
  ancestor；`build-cuda-windows.yml` / `AGENTS.md` / `.docs/*` 與同步前 blob 逐一
  比對為 identical（`git diff --stat backup/... master -- <those paths>` 空輸出）。
  workflow 內容再確認：單一 `cuda` job、matrix `include` 僅 `cuda: '12.4'` + `arch: x64`、
  無 hip/rocm、`-DGGML_CPU=ON`。
- push `bf2946f7b`（`--force-with-lease`，23 個 fork commits 在上游之上）。
  備份：`backup/master-pre-sync-20261024`。
- run #64 `fork-windows-cuda` in_progress，headSha `bf2946f7b`，**jobs 只有一個**
  `windows-cuda (12.4, x64)` → 符合「actions 只有一個編譯工作」。
- `gh workflow list -R Lawlietr/llama.cpp` 會顯示第二筆
  `.github/workflows/pat-classic-wf-test.yml`（id 344535503，state active）：這是
  2026-08-28 測 PAT 權限时留下的**孤兒註冊**，該路徑不在 master tree（`git ls-tree`
  查無），最後兩筆 run 發生在 `_pat_classic_test` 分支且 0s 失敗，之後從未再跑。
  → 不會觸發額外 build，不必處理（要清乾淨只能在 Actions UI 裡停用/移除註冊）。

### 2026-10-21（sync to upstream 8baabd53b）

- fork 點 `4a45de4f2`，上游 11 commits。三條預判全綠：上游**未改** `AGENTS.md` /
  `build-cuda-windows.yml`、**未新增** workflow、無 modify/delete 風險。
- rebase 21 個 fork commits 一次通過，**零衝突**。
- 必查 4 項全過；`.github/workflows/` 只剩 `build-cuda-windows.yml`（單一 `cuda` job、
  CUDA 12.4/x64、`GGML_CPU=ON`），`AGENTS.md` 為 fork 版。
- push `741dc22dc`（`--force-with-lease`）。備份：`backup/master-pre-sync-20261021`。
- run #63 `fork-windows-cuda` 針對新 sha `741dc22` **in_progress**（build 正常觸發）。
- **注意 `gh` 預設 repo**：`gh repo view` 回 `ggml-org/llama.cpp`（auth context 指向上游），
  所以 `gh run list` 沒加 `-R` 時會抓到上游的 run（`CI (sycl)`、`Server`、`CI (wasi)`…，
  headSha 為上游 commit），`-c <fork sha>` 也抓不到自己的 run。**查自己 fork 的 run 一定要
  加 `-R Lawlietr/llama.cpp`**。

### 2026-10-09（sync to upstream de7fa0a3c）

- fork 點 `9c2e0e491`，上游 24 commits。三條預判全綠：上游**未改** `AGENTS.md` /
  `build-cuda-windows.yml`、**未新增** workflow、無 modify/delete 風險。
- rebase 19 個 fork commits 一次通過，**零衝突**（歷來最乾淨的一次）。
- 必查 4 項全過；`build-cuda-windows.yml` 與 `AGENTS.md`、`.docs/*` 的 blob 與同步前
  逐一 `git rev-parse` 比對，全部 `SAME`。
- push `9d61fadd2`（`--force-with-lease`）。備份：`backup/master-pre-sync-20261009`。
- **踩雷：push 沒有產生 run。** 排查順序與結論：
  1. `git ls-remote` / repo `pushed_at` 有更新 → push 確實送達 GitHub。
  2. `actions/runs?status=queued|in_progress` = 0；新 sha 的 `check-runs total=0`
     → 不是「run 失敗」，是**根本沒建立 run**。
  3. `actions/permissions` 回 `enabled:true`（這是 allowed_actions policy，
     **不代表** Actions 可用）；`gh workflow run` → **HTTP 422
     "Actions has been disabled for this repository"**。
  4. githubstatus.com 全部 operational → 不是平台故障。
  5. 正解：Actions 頁面旗標（匿名也看得到）
     *"Workflows aren't being run on this fork because of its GitHub Actions
     usage. A repository maintainer can re-enable them."*
     → GitHub 因 usage 自動**暫停 fork 的 workflows**，需在 Actions 頁面點
     re-enable 手動恢復；`PUT /actions/workflows/{id}/enable` 與
     `DELETE /actions/workflows/{id}` 在此狀態下都會 404 / 無效，API 解不了。
- 教訓：**push 後務必確認 run 真的存在**（`runs?per_page=1` 的 `head_sha` 要對得上），
  不要只看 push 成功。近期 build 常達 1–2 小時（#47/#49/#51 皆 >1h），
  容易觸發這種 usage-based 暫停。
