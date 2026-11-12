# 認證與 env 檔

`tailnetctl.py` 從 env 檔（或 shell 環境變數）讀取設定。本技能預設使用：

```
~/.config/tailscale/tailnetctl.env
```

可用 `TAILNETCTL_ENV_FILE=/path/to/file` 覆寫；wrapper 會在 env 檔存在時把該路徑
以 `--env-file` 傳給工具。若使用者自己在指令中給 `--env-file`，以使用者為準。

只有 `join`／重新認證需要 auth key；`status`、`list`、`ssh`、`web` 不需要，
因此沒有 env 檔時仍可查狀態與操作既有連線。

## 建立

`$SKILL_DIR` 是本技能目錄。範本不含任何 secret，可安全複製：

```bash
mkdir -p ~/.config/tailscale
chmod 700 ~/.config/tailscale
install -m 600 "$SKILL_DIR/references/env.example" \
  ~/.config/tailscale/tailnetctl.env
```

填入 auth key。建議使用「key 檔案」而不是 inline key。**請使用者在自己的終端機
執行**，不要把 key 貼進對話；用隱藏輸入避免 key 留在 shell history。
交棒時務必補上：「完成後請在對話回覆『完成』，我不會自動偵測；失敗請貼錯誤訊息。」
並等待使用者回覆後再跑 `check_setup` 驗證。
若 key 檔已存在且有效就跳過以下指令：

```dotenv
TAILSCALE_AUTH_KEY_FILE=/home/<you>/.config/tailscale/authkey
TAILSCALE_REMOTE_SSH_USER=<remote-user>
```

```bash
install -m 600 /dev/null ~/.config/tailscale/authkey
printf 'Paste auth key: '; read -rs TSKEY; printf '\n'; printf '%s\n' "$TSKEY" > ~/.config/tailscale/authkey; unset TSKEY
```

也可請使用者用編輯器建立 `~/.config/tailscale/authkey`（內容只有 key）。

Auth key 建立位置：<https://console.tailscale.com/admin/settings/keys>

## 驗證

```bash
python3 "$SKILL_DIR/scripts/check_setup"
```

輸出 JSON，不會顯示 key。重點欄位：

- `env_exists`：env 檔是否存在。
- `env_mode_ok`：是否沒有 group/other 權限（應為 `true`）。
- `key_source` / `key_present`：key 來源與是否可用。
- `key_placeholder`：是否仍是 `REPLACE_ME`。
- `backend_state` / `online`：Tailscale 目前狀態。
- `remote_ssh_user`：`TAILSCALE_REMOTE_SSH_USER` 的值（非 secret，未設定為 `null`）。
- `current_user`：執行 `check_setup` 的目前使用者。
- `ssh_user_candidate`：SSH／web 前要確認的候選帳號；有設定 env 時等於
  `remote_ssh_user`，否則等於 `current_user`。
- `ssh_user_source`：候選帳號來源，`shell`、`env` 或 `current`。
- `ready`：是否可執行 `join`。

需要時也可用 `join --dry-run` 驗證 key 可解析（不會印出 key）。

`check_setup` 在 `ready: false` 時會以 exit 1 結束，但仍會輸出完整 JSON；
SSH／web 只需要 `ssh_user_candidate` 時可忽略 exit code。

## 欄位

| 欄位 | 必填 | 用途 |
|---|---|---|
| `TAILSCALE_AUTH_KEY` | `join` 時需要（二選一） | inline auth key；不可留在 placeholder。 |
| `TAILSCALE_AUTH_KEY_FILE` | 否（二選一，優先） | 只含 key 的檔案路徑；同一來源內優先於 inline key。 |
| `TAILSCALE_LOGIN_SERVER` | 否 | 自訂控制平面（如 Headscale）。 |
| `TAILSCALE_HOSTNAME` | 否 | 註冊到 tailnet 的 hostname。 |
| `TAILSCALE_ADVERTISE_TAGS` | 否 | 逗號分隔 tags，例如 `tag:dev,tag:linux`。 |
| `TAILSCALE_ACCEPT_DNS` | 否 | `true` 或 `false`。 |
| `TAILSCALE_REMOTE_SSH_USER` | 否 | SSH／web 連到 peer 時使用的遠端帳號。 |
| `TAILSCALE_REMOTE_WEB_PORT` | 否 | `web` 未指定 `--port` 時的遠端 port（預設 80）。 |

優先序：**CLI 參數 > shell 環境變數 > env 檔 > 預設值**。
同一來源內，`TAILSCALE_AUTH_KEY_FILE` 優先於 `TAILSCALE_AUTH_KEY`。

## 安全

- env 檔與 key 檔都必須 `0600`；`~/.config/tailscale` 建議 `0700`。
- 不要把 env 檔 commit 進任何 repo；本 skill repo 的 `.gitignore` 已排除 `*.env`。
- 不要用 `cat`、`grep`、`echo` 讀取 key；只讓 `tailnetctl.py` 讀。
- 不要把 key 放在 CLI 參數。
- auth key 可能過期或只能使用一次；失效時重新建立並更新 key 檔。
- `join` 成功後通常不需要再提供 key；只有重新認證或切換 tailnet 才需要。
