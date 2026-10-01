# 0001 容器引擎：CI 用 Docker，本機可用 rootless Podman

- 狀態：Accepted
- 日期：2026-10-01

## 背景

規格要求本機與 CI 使用相同的容器環境定義。開發機是共用的實驗室伺服器（Ubuntu 22.04），使用者沒有 sudo 權限，也沒有安裝 Docker。實測可用的條件：

- 已安裝 `podman` 3.4.4，`/etc/subuid` 有使用者的設定，`newuidmap` 存在。
- rootless `podman run docker.io/library/ubuntu:22.04` 可正常執行。

GitHub-hosted 的 `ubuntu-*` runner 內建 Docker。

## 決策

1. image 一律用標準 Dockerfile 語法定義，放在 `docker/`。
2. GitHub Actions 使用 Docker。
3. 本機的 `scripts/ci_local.sh` 讀取 `CONTAINER_ENGINE`（`docker`／`podman`）；未設定時依序偵測 docker、podman。
4. UID 對應方式：Docker 用 `--user "$(id -u):$(id -g)"`，rootless Podman 用 `--userns=keep-id`。
5. 不使用只有單一引擎支援的 Dockerfile 語法；BuildKit 專屬功能（例如 `RUN --mount=type=cache`）在確認 Podman 3.4.4 支援前不採用。

## 取捨

- 好處：沒有管理員權限也能在本機重現 CI；rootless 執行本身也更安全。
- 代價：兩種引擎的行為不完全相同（UID mapping、build cache、image 名稱前綴 `localhost/`）。若本機與 CI 結果不一致，以 CI（Docker）為準，並把差異記錄到 `docs/runbook.md`。
- 實驗室伺服器是共用資源：image 與 build cache 存放在使用者的 home 下，要注意磁碟配額，定期執行 `podman system prune`。

## 未採用的方案

- 請管理員安裝 Docker 並把使用者加進 docker group：這等同給予 root 權限，不適合共用伺服器。
- 本機完全不使用容器、只在 CI 上跑：違反「本機可重現 CI」的目標，而且每次除錯都要 push 上去，回饋迴圈太慢。
