# llama.cpp fork 同步協議（實戰版）

> 來源：`AGENTS.md` 的延伸實戰版。AGENTS.md 保持精簡；本檔記錄詳細檢查、
> 衝突解法與每次同步的踩雷。sync 前先用 `ctx_search(queries=[...], source="llama.cpp-fork-sync")` 撈本檔。

## 何時同步

依使用者明確要求才做（「同步上游 / sync」）。不要自己觸發。

## 同步前預判（省掉重複的 git 檢查）

rebase 前先用下面兩條確認，通常可乾淨通過：

```sh
# 1. 上游有沒有改 AGENTS.md / build-cuda-windows.yml？（空 = 不會衝突）
git log <fork點>..upstream/master -- AGENTS.md .github/workflows/build-cuda-windows.yml

# 2. 上游有沒有「新增」workflow？（fork 刪除清單要蓋到全部）
git ls-tree -r --name-only 093a2f86c -- .github/workflows/   # fork 點
git ls-tree -r --name-only upstream/master -- .github/workflows/  # 現上游
# 用 comm -13 比較，右列多出來 = 新增、需手動刪
```

- 上游通常**不改** `AGENTS.md` 與 `build-cuda-windows.yml` → rebase 乾淨套上
  （檔名與上游相同會並存，取 fork 版即可）。
- 唯一預期衝突：刪除 upstream workflow 的 **delete/modify**，直接 `git rm`。
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
