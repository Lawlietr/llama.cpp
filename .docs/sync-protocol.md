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
