---
name: tailnetctl
description: >
  Use this skill when the user wants to enable or connect Tailscale, check whether this machine is joined to a tailnet and online, list tailnet devices, SSH into a tailnet peer, or open an SSH port-forward (web tunnel) via tailnetctl.
  用於啟用 Tailscale、查詢是否已連上並加入 tailnet、列出 tailnet 裝置、以 SSH 連線到指定 peer，或建立 web tunnel。
  關鍵字：tailscale, tailnet, 啟用 Tailscale, 連線狀態, 加入 tailnet, 裝置清單, peer, SSH, web tunnel, tailnetctl, tailscale up, tailscale status.
---

# tailnetctl — Tailscale 連線與 tailnet 操作

本技能是 `tailnetctl.py`（vendored，未修改）的薄編排層。用自然語言完成：
啟用/加入 tailnet、查詢連線狀態、列出裝置、SSH 到 peer、開 web tunnel。

## 名詞與路徑

- **SKILL_DIR**：本技能所在目錄（host 提供）。以下指令用 `$SKILL_DIR` 代表。
- **wrapper**：`$SKILL_DIR/scripts/tailnetctl`。env 檔存在時自動帶入
  `--env-file`；`status`/`list`/`ssh`/`web` 在沒有 env 檔時仍可執行；
  `join` 缺 env 檔時會停下來顯示設定指引。
- **工具本體**：`$SKILL_DIR/scripts/tailnetctl.py`（原封不動的 vendored 版本）。
- **env 檔**：預設 `~/.config/tailscale/tailnetctl.env`（0600）；
  可用環境變數 `TAILNETCTL_ENV_FILE` 覆寫。
- **範本**：`$SKILL_DIR/references/env.example`（可安全複製，不含 secret）。
- **設定檢查**：`$SKILL_DIR/scripts/check_setup`，輸出 JSON，不顯示 key；
  另有非 secret 的 `ssh_user_candidate`／`ssh_user_source` 可用於 SSH 前確認帳號。
- **輔助腳本**：`ssh_peer`、`web_peer`，依 TTY 自動決定執行或只印指令。

## 前置條件

- Linux／macOS（Windows 用 WSL）；Python 3.8+。
- `tailscale` client 與 daemon；`ssh` client。
- env 檔只有 `join`／重新認證才需要；其他操作不需要 auth key。
- env 檔與 key 設定方式見 `references/env.md`，缺少時依下一節引導。
- 需要時 `sudo` 可執行（`join`、啟動 daemon 屬寫入操作）。

## 首次設定（env 檔不存在或 key 未就緒）

先跑檢查（唯讀、不顯示 key）：

```bash
python3 "$SKILL_DIR/scripts/check_setup"
```

若 `ready` 為 `false`：

1. 告訴使用者加入 tailnet 需要 Tailscale auth key，並提供建立網址：
   <https://console.tailscale.com/admin/settings/keys>
2. **先詢問**是否要由你建立 0600 的 env 範本；不要直接寫入。
3. 取得同意後建立目錄與範本（範本只有註解與 placeholder，沒有 secret）：
   ```bash
   mkdir -p ~/.config/tailscale && chmod 700 ~/.config/tailscale
   install -m 600 "$SKILL_DIR/references/env.example" \
     ~/.config/tailscale/tailnetctl.env
   ```
   若使用 `TAILNETCTL_ENV_FILE` 或 `--env-file`，改用該路徑。
4. **明確告訴使用者：「請在你自己的終端機執行以下指令」**，不要把 key 貼進對話，
   也不要用 `printf 'tskey-...'` 把 key 直接寫在命令列（會留在 shell history）。
   若 key 檔已存在且有效就跳過；否則用隱藏輸入建立 0600 key 檔：
   ```bash
   install -m 600 /dev/null ~/.config/tailscale/authkey
   printf 'Paste auth key: '; read -rs TSKEY; printf '\n'; printf '%s\n' "$TSKEY" > ~/.config/tailscale/authkey; unset TSKEY
   ```
   也可請使用者用編輯器建立
   `~/.config/tailscale/authkey`（內容只有 key）。
   並確認 env 檔的 `TAILSCALE_AUTH_KEY_FILE=/home/<you>/.config/tailscale/authkey`
   （或直接填 `TAILSCALE_AUTH_KEY=...`）。
5. 使用者確認完成後，再跑 `check_setup` 確認 `ready: true` 且 `env_mode_ok: true`；
   需要時再跑 `join --dry-run` 驗證 key 可解析。
6. 回到原本請求，執行 `join` 並以 `status` 確認。

若 `check_setup` 顯示 `key_placeholder: true`，代表仍是 `REPLACE_ME`，重複步驟 4。
若 `env_mode_ok: false`，提醒 `chmod 600 <env 檔>`。

## 安全

- **不要** `cat`、`grep`、`echo` env 檔或 auth key；只能讓 `tailnetctl.py` 讀。
- **不要**把 key 放在命令列參數（會留在 shell history／process list）；
  引導使用者用隱藏輸入（`read -rs`）或編輯器寫入 key 檔。
- **不要**在非 TTY 環境用 `ssh -o BatchMode=yes` 反覆猜測遠端帳號；先確認 username，
  再產生指令給使用者。
- `join`、`join --force`、`--reset` 會改變網路狀態或重新認證；執行前先向使用者確認。
- `web` 會建立對外（依 `--bind`）的 SSH tunnel；預設只綁 `127.0.0.1`。
- SSH 沿用系統 `~/.ssh/config`、agent 與 `known_hosts`；不要自動關閉 host key 檢查。

---

## 工作流

### 1. 先查狀態（幾乎所有請求都從這步開始）

```bash
python3 "$SKILL_DIR/scripts/tailnetctl" status
```

需要結構化資料時：

```bash
python3 "$SKILL_DIR/scripts/tailnetctl" status --json
```

判讀：

- `BackendState == "Running"` 且 `Self.Online == true` → 已連上且在線。
- `BackendState == "NeedsLogin"` → 尚未加入 tailnet，需 `join`。
- 其他狀態或離線 → 未連線；`list` 的資料可能只是快取。
- `CurrentTailnet.Name` 是 tailnet 名稱；`MagicDNSSuffix` 是 MagicDNS 後綴。

回報時要明確回答兩件事：**是否已加入 tailnet**、**目前是否在線**。

### 2. 啟用／加入 tailnet

使用者說「啟用 Tailscale」、「連上 tailnet」時：

1. 先做步驟 1。若已 `Running` 且在線，直接回報，**不要**重認證。
2. 若尚未加入，先跑 `check_setup`。`ready: false` 或 env 檔不存在時，
   依「首次設定」引導；完成後再繼續。
3. 先預覽指令（auth key 會顯示為 `<redacted>`）：
   ```bash
   python3 "$SKILL_DIR/scripts/tailnetctl" join --dry-run
   ```
4. 正式加入：
   ```bash
   python3 "$SKILL_DIR/scripts/tailnetctl" join
   ```
5. 再次 `status` 確認 `Running` 且在線。

注意：

- 已 `Running` 但離線：先確認網路，不要自動 `--force`。
- 要重新認證或切換 tailnet 才用 `join --force`（先取得使用者同意）。
- 若預覽指令含 `sudo`，且 `sudo -n true` 失敗、目前又沒有 TTY，**不要**直接執行；
  把指令印給使用者，請他在終端機執行，或先 `sudo -v`。
- `join` 成功後會輪詢最多約 60 秒。

### 3. 列出 tailnet 裝置

```bash
python3 "$SKILL_DIR/scripts/tailnetctl" list --json
```

- 輸出是陣列，第一筆是 `self`（`self: true`），欄位含 `hostname`、`os`、
  `online`、`ip`、`dns_name`、`owner`、`tags`。
- 只看在線裝置：加 `--online-only`。
- 尚未連線或離線時，先提醒資料可能是快取。
- 呈現表格時標示 `(self)`；SSH 目標預設排除 `self`，除非使用者指定。

### 4. SSH 到 peer

1. 先用 `list --json` 確認裝置（hostname、IP 或序號）；不確定就讓使用者選。
2. **先確認遠端 username，再產生指令**：
   - 使用者已指定 `--remote-user` → 直接使用，不必再問。
   - 否則跑 `check_setup`，取 `ssh_user_candidate`（非 secret）。它的來源是
     `TAILSCALE_REMOTE_SSH_USER`（shell 環境變數或 env 檔）；未設定時就是
     `current_user`（ssh 預設的目前使用者）。即使 key 未就緒使 `check_setup`
     以 exit 1 結束，JSON 仍有效，SSH 流程只取這個欄位即可。
   - 問使用者：「要用 `<user>` 這個 username 連到 `<host>` 嗎？」
3. 使用者確認後，用輔助腳本產生指令：
   ```bash
   python3 "$SKILL_DIR/scripts/ssh_peer" --peer <host> --remote-user <user> [--ssh-port <port>]
   ```
   若使用者要用 ssh 預設帳號，才省略 `--remote-user`。
4. 若使用者要指定 ssh 選項：
   ```bash
   python3 "$SKILL_DIR/scripts/ssh_peer" --peer <host> --remote-user <user> --ssh-arg='-o StrictHostKeyChecking=accept-new'
   ```
5. 非 TTY（agent 工具呼叫）時，把印出的 `ssh -- <user>@<host>` 貼給使用者，
   並明確說「請在終端機執行」。**不要**在對話視窗嘗試互動式 ssh，也不要
   用 BatchMode 反覆試帳號；必要時只做 `tailscale ping` 確認可達。

遠端使用者優先序：`--remote-user` > `TAILSCALE_REMOTE_SSH_USER` > 目前使用者
（`current_user`／`whoami`）。

### 5. Web tunnel（SSH 端口轉送）

```bash
python3 "$SKILL_DIR/scripts/web_peer" --peer <host> --port <remote_port> [--local-port <local_port>]
```

- 預設綁 `127.0.0.1:8080`，連到 peer 的指定 port。
- 遠端 username 規則與 SSH 相同：先用 `check_setup` 的 `ssh_user_candidate` 確認，
  需要時加 `--remote-user`；非 TTY 時把完整指令交給使用者在終端機執行。
- TTY 時前景執行，按 Ctrl-C 結束；非 TTY 時只印出等價的 `ssh -N -L ...` 指令。
- 非 loopback 的 `--bind` 會讓其他機器連入，執行前提醒使用者。

### 6. 其他

- 未給子指令時，工具會顯示完整 help 與範例。
- 全域選項（放在子指令前）：`--env-file`、`--sudo`、`--no-sudo`、
  `--no-start-daemon`、`--debug`。
- 結束碼：`0` 成功、`1` 錯誤（`CtlError`）、`130` 使用者在提示時取消。

---

## 指令速查

| 需求 | 指令 |
|---|---|
| 設定檢查 | `python3 "$SKILL_DIR/scripts/check_setup"` |
| 連線摘要 | `python3 "$SKILL_DIR/scripts/tailnetctl" status` |
| 原始狀態 | `... status --json` |
| 加入 tailnet | `... join`（先 `join --dry-run`） |
| 裝置清單 | `... list` / `... list --json` / `... list --online-only` |
| SSH | `python3 "$SKILL_DIR/scripts/ssh_peer" --peer HOST ...` |
| Web tunnel | `python3 "$SKILL_DIR/scripts/web_peer" --peer HOST --port N ...` |

---

## 失敗排除

先看 `references/troubleshooting.md`。常見：

- 找不到 env 檔或 key：依「首次設定」引導，不要要求使用者在聊天貼 key。
- 找不到 `tailscale`：依平台安裝，工具會顯示安裝指引。
- daemon 未啟動：工具會嘗試啟動；失敗時可能需要 `--sudo` 或先啟動服務。
- `NeedsLogin`：執行 `join`。
- `REPLACE_ME` 或 key 無效：更新 env 檔的 auth key。
- env 檔權限過寬：`chmod 600`。
- 非 TTY 下卡在 sudo／遠端使用者提示：改用 `--dry-run` 印指令給使用者，
  並先用 `check_setup` 的 `ssh_user_candidate` 確認 username。
- SSH host key 變更：確認是預期變更後，`ssh-keygen -R <host>`。

---

## Definition of Done

- 已先查 `status`，並明確回答「是否已加入 tailnet」「是否在線」。
- 需要 `join` 時，已先 `--dry-run` 並在必要時取得使用者同意。
- 裝置清單有標示 `(self)`，SSH 目標沒有誤選自己。
- SSH／web 指令產生前已確認 username（`--remote-user` > `TAILSCALE_REMOTE_SSH_USER`
  > 目前使用者），且最終指令有明確標示「請在終端機執行」。
- 非 TTY 的 SSH／web 只輸出可執行指令，未強行開啟互動 session，也未用 BatchMode
  反覆猜帳號。
- 若缺少 env／key，已用 `check_setup` 與「首次設定」引導完成，且明確請使用者在
  終端機輸入 key、未要求使用者在對話中貼 key。
- 未讀取或印出 auth key，未修改 vendored `tailnetctl.py`。
- 未自動 commit 或變更使用者未同意的網路設定。
