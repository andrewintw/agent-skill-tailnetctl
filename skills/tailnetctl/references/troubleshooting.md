# 疑難排解

| 症狀 | 可能原因 | 處理 |
|---|---|---|
| `找不到 tailscale` | 未安裝 client | Linux：`curl -fsSL https://tailscale.com/install.sh \| sh`；其他平台見 tailscale.com/download |
| `Cannot connect to Tailscale daemon` / 連不上 daemon | `tailscaled` 未啟動或目前使用者無權限 | 讓工具嘗試啟動；必要時先啟動服務，或加 `--sudo` |
| `join` 卡住或要求密碼 | 需要 sudo，但非互動環境無法輸入 | 先 `join --dry-run`，把含 `sudo` 的指令交給使用者在終端機執行；或先 `sudo -v` |
| `BackendState=NeedsLogin` | 尚未加入 tailnet | 執行 `join` |
| `BackendState` 非 `Running` | 尚未連線或被停止 | 先 `join`；仍失敗就看 daemon／網路 |
| `Running` 但 `Online=no` | 本機離線或網路問題 | 檢查網路；不要自動 `--force`，先問使用者 |
| `list` 資料看起來過時 | 未連線／離線時顯示快取 | 先跑 `status`，確認 `Running` 且在線 |
| `TAILSCALE_AUTH_KEY` 仍是 `REPLACE_ME` | key 未設定 | 更新 env 檔或 `TAILSCALE_AUTH_KEY_FILE` |
| auth key 被拒 / 過期 | key 失效、單次使用已用過 | 重新建立 auth key 並更新 |
| `警告: ... 權限過寬` | env 檔可被 group/other 讀取 | `chmod 600 ~/.config/tailscale/tailnetctl.env` |
| SSH 沒有反應或卡住 | 非 TTY 環境執行互動式 ssh | 改用 `scripts/ssh_peer`；非 TTY 會印出指令 |
| 一直詢問遠端使用者 | 非 TTY 無法回答提示 | 先以 `check_setup` 的 `ssh_user_candidate` 確認帳號，再用 `--remote-user` |
| 在對話視窗無法完成 SSH | agent 非 TTY，無法交棒互動式 ssh | 把 `ssh -- <user>@<host>` 完整指令給使用者在終端機執行，並要求完成後回覆結果 |
| 使用者貼完指令後以為 agent 會自動偵測 | 交棒時沒有要求回報完成 | 每次交棒都明確要求「完成後在對話回覆『完成』」，等待回覆再以 `check_setup`／`status` 驗證 |
| `BatchMode ... Permission denied` | 用 BatchMode 反覆猜帳號 | 不要猜；先確認 username，或請使用者用密碼／金鑰登入 |
| auth key 留在 shell history | 用 `printf` 直接把 key 寫在命令列 | 改用 `read -rs` 隱藏輸入或編輯器寫入 key 檔 |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | 節點重建、host key 變更 | 確認是預期變更後 `ssh-keygen -R <hostname>` 與 `ssh-keygen -R <ip>` |
| Headscale／自架控制平面連不上 | 未指定 login server | 設 `TAILSCALE_LOGIN_SERVER` |
| `web` 的 port 已被占用 | 本機 port 衝突 | 用 `--local-port` 換 port |
| `web` 被其他機器連入 | `--bind` 非 loopback | 改用 `127.0.0.1`（預設） |
| `Address already in use` / `ExitOnForwardFailure` | 轉送埠無法綁定 | 換 `--local-port`，或先停掉佔用者 |

## 除錯

- 加 `--debug` 可看到實際執行的指令（auth key 會遮蔽）。
- `status --json` 是原始 Tailscale status JSON，適合進階排查。
- `join --dry-run` 只印出指令，不執行。
- 工具結束碼：`0` 成功、`1` 錯誤、`130` 使用者在提示時取消。

## 不要做的事

- 不要 `cat` / `grep` env 檔或 key。
- 不要在未確認的情況下 `join --force`（可能中斷目前連線或切換 tailnet）。
- 不要自動關閉 SSH host key 檢查。
- 不要在非 TTY 硬跑互動式 `ssh` 或 `web`，也不要用 BatchMode 反覆猜帳號。
- 不要把 auth key 用 `printf` 直接寫在命令列（會留在 shell history）。
