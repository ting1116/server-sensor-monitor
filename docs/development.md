# 開發環境

本文件說明如何在本機建立開發環境，以及本機環境與 CI 基準環境的關係。產品行為的規則請見 [spec.md](spec.md)。

## 1 兩層環境

| | 本機開發環境（conda） | 基準環境（容器） |
|---|---|---|
| 用途 | 日常撰寫、編譯、測試、除錯 | 確認變更在 CI 相同的環境中通過 |
| 回饋速度 | 快，修改後數秒內重新編譯 | 較慢，需要 build image |
| 是否為合併標準 | 否 | 是，CI gate 以此為準（spec 第 9、10 節） |
| 開始使用 | M1 | M4 |

本機環境的目標是**盡量接近**基準環境，讓大部分問題在本機就能發現；但它不是基準環境本身。兩者結果不一致時，以容器（CI）為準。

## 2 前置需求

- Linux x86_64，建議 Ubuntu 22.04（與基準環境相同）
- 系統已安裝 GCC 11 系列的 `g++`
- `git`
- conda（Anaconda、Miniconda 或 Miniforge 皆可）

不需要 sudo 權限。

## 3 建立環境

```bash
conda env create -f environment.yml
conda activate ssm
```

之後若 `environment.yml` 有變更：

```bash
conda env update -n ssm -f environment.yml --prune
```

`environment.yml` 只使用 `conda-forge` channel，並以 `nodefaults` 排除 defaults channel。即使個人的 conda 設定中有其他 channel（例如 bioconda），也不會影響此環境的套件來源。

## 4 驗證環境

```bash
conda activate ssm
which g++ cmake ninja cppcheck python
g++ --version | head -1
cmake --version | head -1
python --version
```

預期結果：

| 項目 | 預期 |
|---|---|
| `which g++` | `/usr/bin/g++`（**不是** conda 環境內的路徑） |
| `g++` | 11.x |
| `cmake` | 3.22.x |
| `python` | 3.10.x |
| 其他工具 | 位於 conda 環境的 `bin/` 之下 |

## 5 為什麼編譯器用系統的，而不是 conda 的

- 基準環境是 Ubuntu 22.04 + GCC 11（spec 第 9 節）。系統 g++ 版本與之相同，conda 提供的 GCC 版本則不同。
- 使用系統 g++ 時，執行檔連結的是系統的 `libstdc++`，與使用者實際執行時的環境一致。
- 因此 `environment.yml` **刻意不包含** `gxx`、`compilers` 等編譯器套件，不要自行加入。

確認執行檔連結的是系統函式庫：

```bash
ldd build/server-sensor-monitor | grep libstdc++
# 預期：/lib/x86_64-linux-gnu/libstdc++.so.6
```

## 6 版本對照與已知差異

| 工具 | 本機（conda） | 基準環境 | 說明 |
|---|---|---|---|
| g++ | 11.4（系統） | GCC 11 系列 | 一致 |
| Python | 3.10 | 3.10 | 一致；開發機的系統 Python 可能較新，撰寫 validator 時不可使用 3.11 以後的語法 |
| CMake | 3.22 | 3.22 | 刻意固定。conda 預設會安裝 4.x，可能讓只有新版支援的寫法在本機通過、在 CI 失敗 |
| cppcheck | 2.21 | **待定（M4）** | 不同版本偵測到的問題不同。M4 決定容器內版本後，`environment.yml` 跟著調整 |
| Ninja、GDB、gh | 未固定 | 不適用或未固定 | 版本差異不影響建置結果 |

**已知限制**：在 M4 之前，本機 cppcheck 的結果不能代表 CI 的結果。

## 7 GitHub CLI

第一次使用時登入：

```bash
gh auth login
```

依提示選擇 GitHub.com、HTTPS，並以瀏覽器完成驗證。登入資訊存放在使用者 home 目錄下，不會進入 repository。

## 8 容器環境（M4 起）

M4 完成後，此節補充：

- 如何以 `scripts/ci_local.sh` 在本機執行與 CI 相同的檢查
- `CONTAINER_ENGINE` 的設定方式（Docker 或 rootless Podman），見 [decisions/0001-container-engine.md](decisions/0001-container-engine.md)
- image 與 cache 的磁碟清理方式

## 9 常見問題

**`which g++` 指向 conda 環境內的路徑**
環境中被裝進了編譯器套件。用 `conda list -n ssm | grep -E 'gxx|gcc'` 確認，移除後重新驗證。

**`cmake --version` 顯示 4.x**
環境沒有依 `environment.yml` 更新。執行第 3 節的 `conda env update` 指令。

**在本機通過，CI 失敗**
先對照第 6 節的版本差異。M4 之後，在開 PR 前執行 `scripts/ci_local.sh`，在本機重現 CI。
