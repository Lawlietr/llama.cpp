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
