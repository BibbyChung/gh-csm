# gh-csm

GitHub Codespaces 額度監控與管理器，包成 gh extension：一個指令看這個月還剩多少
compute / storage 額度，順手停掉或喚醒 codespace。

```
$ gh csm dash
── bb9755 · 2026-10 UTC · 離重置 7d22h ────
Compute  ████████░░░░░░░░░░░░  43%   51.98/120 core-hrs · 剩 2-core 34h · 4-core 17h
Storage  ░░░░░░░░░░░░░░░░░░░░   <1%   0.01/15 GB-month
```

`--all` 每個帳號一個面板，`(active)` 標在 header；沒解鎖 `user` scope 的帳號只留一行提示：

```
$ gh csm dash --all
── bb9755 (active) · 2026-10 UTC · 離重置 7d22h ────
Compute  ████████░░░░░░░░░░░░  43%   51.98/120 core-hrs · 剩 2-core 34h · 4-core 17h
Storage  ░░░░░░░░░░░░░░░░░░░░   <1%   0.01/15 GB-month

── BibbyChung · 2026-10 UTC · 離重置 7d22h ────
  ⚠ 缺 user scope，解鎖：gh auth switch --user BibbyChung && gh auth refresh -h github.com -s user,codespace
```

## 功能

- **dash**：三行看完——compute / storage 進度條（綠 <70%、黃 70–89%、紅 ≥90%），
  依剩餘 core-hours 換算 2-core / 4-core 還能跑幾小時（整數、向下取整，不虛報），
  header 帶距離額度重置的倒數（`7d22h` / `5h` / `42m`）。
  清單與操作交給 `stop`，dash 只打一次 billing API。
- **stop**：不帶參數是互動選單（編號選、再確認）；`-c <name>` 直接停。
- **run**：喚醒 codespace 後執行 snippet，或省略 snippet 直接進互動 shell
  （stopped 的 codespace 會自動喚醒）。
- **多帳號**：預設看 gh 的 active 帳號，`--account <login>` 指定，`--all` 全部堆疊。
- **selftest**：零網路的離線驗算，用真實帳單 fixture 檢查公式與色階。

## 需求

- `gh`（2.40+，已 `gh auth login`）
- `jq`
- bash 3.2 以上（macOS 內建的 `/bin/bash` 就是 3.2.57，沒問題）

## 安裝

```bash
gh extension install <owner>/gh-csm
```

之後就能用 `gh csm ...`。升級：

```bash
gh extension upgrade gh-csm
```

開發時（在 repo 目錄內，建立 symlink，改完即生效）：

```bash
gh extension install .
```

## 一次性解鎖 scope

billing 端點只認 classic OAuth 的 `user` scope，`codespace` scope 則是列出／停止／
進入 codespace 用。每個帳號解鎖一次即可：

```bash
gh auth refresh -h github.com -s user,codespace
```

**多帳號**：`gh auth refresh` 沒有 `--user`，inactive 帳號要先切過去再切回來。

```bash
gh auth switch --user other-account
gh auth refresh -h github.com -s user,codespace
gh auth switch --user your-main-account
```

沒解鎖的帳號不會讓整個 dashboard 爆掉——該帳號面板只留一行解鎖指令，
其他帳號照常顯示。

## 使用方式

```bash
gh csm                       # help
gh csm dash                  # 額度面板（一定要明寫 dash，沒有預設子命令）
gh csm dash --all            # 所有已登入帳號，面板垂直堆疊
gh csm dash --account bb9755 # 指定帳號

gh csm stop                  # 互動選單挑一個 codespace 停掉
gh csm stop -c my-codespace  # 直接停

gh csm run -c my-codespace 'echo gogogo...'   # 喚醒 + 執行，跑完退出
gh csm run -c my-codespace                    # 喚醒 + 進互動 shell

gh csm selftest              # 離線自檢
```

### 額度上限

預設是 GitHub Free 的 120 core-hours + 15 GB-month。Pro（180 / 20）或其他方案
用環境變數覆寫：

```bash
GH_CSM_COMPUTE_LIMIT=180 GH_CSM_STORAGE_LIMIT=20 gh csm dash
```

要固定下來就寫進 shell rc：

```bash
export GH_CSM_COMPUTE_LIMIT=180
export GH_CSM_STORAGE_LIMIT=20
```

API 查不出帳號方案別，所以只能自己設定。

## 限制（誠實揭露）

- **數字有延遲**：GitHub billing 約每小時結算一次，剛停掉的 codespace 不會馬上反映。
- **計費月是 UTC**：dash 的月份與重置倒數都走 UTC。本地時間月底／月初可能與 UTC 落在不同月，
  屬正常現象；`stop` 選單的 last used 才轉本地時區。
- **Shutdown 仍吃 storage**：停止只停 compute，儲存空間繼續計費。想真的止血要刪除。
- **刪除不影響已產生的帳**：計費紀錄不會因為刪掉 codespace 而消失。
- **classic OAuth only**：billing 端點不接受 fine-grained PAT，只能用 `gh auth refresh` 解鎖。
- **在 codespace 內執行**：環境內建的 `GITHUB_TOKEN` 會蓋掉你的 gh 登入，本工具會自動忽略它
  並提示一次；若仍有 403，手動 `unset GITHUB_TOKEN` 再跑。
- **只支援 github.com**，不含 GitHub Enterprise Server。
- `stop` / `run` 需要 `codespace` scope；`dash` 的額度區需要 `user` scope。

## 開發

```bash
shellcheck gh-csm
/bin/bash gh-csm selftest
```

設計決策、API 契約、公式與地雷清單見 [SPEC.md](SPEC.md)。
