# gh-csm — 開發規格 (SPEC)

GitHub Codespaces 額度監控與管理器的實作契約。使用者文件見 [README.md](README.md)。

本文件是需求討論與實機探針驗證的蒸餾結果：每一條「為什麼」都有實測依據，不是推測。

---

## 1. 專案定位

| 項目 | 決策 | 理由 |
|---|---|---|
| 形態 | gh extension，單一 bash 執行檔 `gh-csm` | 安裝＝clone repo，零打包、零 brew formula、darwin/linux 共用 |
| 命名 | `gh csm`（repo `gh-csm`） | `gh cs` 是 gh 內建 alias（= `gh codespace`），extension 不可覆蓋 core command/alias |
| 依賴 | `gh` + `jq`（+ bash 3.2 / coreutils） | 無 curl、無 python、無 node |
| 執行模型 | 一次性，跑完即退 | billing 資料本身就落後約 1 小時，常駐輪詢買不到新鮮度 |
| 資料源 | 官方 REST billing API | 網頁內部 `usage_table` 端點未公開、無版本承諾、需 session cookie，已排除 |
| 認證 | gh 既有的 OAuth token（`user` + `codespace` scope） | 腳本不儲存、不輸出 token；只以環境變數傳遞 |

四個交付檔：`gh-csm`（程式）、`README.md`（使用者）、`SPEC.md`（本文件，why）、
`CHANGELOG.md`（版本記錄，what）；另有 CI workflow（`.github/workflows/ci.yml`）。

---

## 2. API 契約

### 2.1 用量

```
GET https://api.github.com/users/{username}/settings/billing/usage?year=YYYY&month=M
```

- `month` 為 1–12，**UTC 月**（見 §5.2）。
- 需要 classic OAuth `user` scope。
- 缺 scope 的行為：`404 Not Found` 且 response header 帶 `X-Accepted-Oauth-Scopes: user`。
  實測對照：不存在的路徑回 404 但**沒有**該 header，因此 404 + header = 授權閘門，不是路徑錯誤。
- 只認 classic OAuth scope；fine-grained PAT 不可用（回應無 fine-grained permissions header）。
- `gh` 版本需支援 `gh auth status --json hosts`（2.40+）。

回應形態（單一 top-level key `usageItems`）：

```json
{
  "usageItems": [
    {
      "date": "2026-10-07T00:00:00Z",
      "product": "codespaces",
      "sku": "Codespaces compute 4-core",
      "quantity": 5.08799,
      "unitType": "Hours",
      "pricePerUnit": 0.36,
      "grossAmount": 1.8316764,
      "discountAmount": 1.8316764,
      "netAmount": 0.0,
      "repositoryName": "_w"
    },
    {
      "date": "2026-10-13T00:00:00Z",
      "product": "codespaces",
      "sku": "Codespaces storage",
      "quantity": 0.153216771,
      "unitType": "GigabyteHours",
      "pricePerUnit": 0.07,
      "grossAmount": 0.01072517,
      "discountAmount": 0.01072517,
      "netAmount": 0.0,
      "repositoryName": "_w"
    }
  ]
}
```

實測彙總（octocat，2026-10 UTC，21 筆）：

| SKU | unitType | Σ quantity |
|---|---|---|
| `Codespaces compute 2-core` | Hours | 7.76645 |
| `Codespaces compute 4-core` | Hours | 9.11165 |
| `Codespaces storage` | GigabyteHours | 5.397099746 |

→ compute 51.98 core-hrs（43% of 120）、storage 5.397 GB-h ÷ 744 h ≈ 0.0073 GB-month。
這組數字就是 `selftest` 的內嵌 fixture。

> prebuild 儲存：官方 SKU id 為 `codespaces_prebuild_storage`；本工具同時接受字串
> `Codespaces prebuild storage`，與 `Codespaces storage` 併入同一 GigabyteHours 池。
> 此字串由 id 推導、未經真實 prebuild 帳號驗證；若實際字串不同會落入未知 SKU 的
> warn（跳過而非算錯，見 §5.12）。

### 2.2 欄位語義

| 欄位 | 語義 | 本工具怎麼用 |
|---|---|---|
| `quantity` | compute 是 **machine-hours**（機型小時，未乘核數）；storage 是 GigabyteHours | **唯一**用量來源 |
| `sku` | 人類可讀字串，如 `Codespaces compute 8-core`、`Codespaces storage` | 從字串抽核數，**不可**硬編 d2/d4 之類 id |
| `unitType` | `Hours` / `GigabyteHours` | 文件用途，程式不依賴 |
| `grossAmount` / `discountAmount` / `netAmount` | 金額 | **不用於計算** |

### 2.3 Codespace 清單

```
GET https://api.github.com/user/codespaces?per_page=100
```

需要 `codespace` scope。關鍵欄位：`name`、`state`、`machine.cpus`、`machine.display_name`、
`last_used_at`（RFC3339，可能帶 `Z` 或 `±HH:MM`）、`repository.full_name`。

`gh codespace list --json machineName` 只給 `basicLinux32gb` 這類機型 id，拿不到核數，
所以清單一律走 REST 取 `machine.cpus`。

---

## 3. 計算公式

```
cores(sku)        = 從 "Codespaces compute N-core" 取 N；不符此形狀 = 未知 SKU
compute_core_hours = Σ over compute items ( quantity × cores(sku) )
storage_gb_hours   = Σ over storage items ( quantity )
month_hours        = days_in_month(UTC year, UTC month) × 24      # 動態，非固定 720
storage_gb_month   = storage_gb_hours / month_hours
compute_pct        = floor(compute_core_hours / compute_limit × 100)
storage_pct        = floor(storage_gb_month  / storage_limit  × 100)
剩餘可跑小時(cores) = floor((compute_limit − compute_core_hours) / cores)   # 整數、向下取整
重置倒數           = 下一個 UTC 月初 00:00 − now(UTC)                     # 7d22h / 5h / 42m
```

- 額度預設 `GH_CSM_COMPUTE_LIMIT=120`、`GH_CSM_STORAGE_LIMIT=15`（GitHub Free）；Pro 為 180 / 20。
- 顏色門檻：`<70` 綠、`70–89` 黃、`≥90` 紅（以整數百分比判斷）。
- `storage_pct == 0` 但用量 > 0 → 顯示 `<1%`。
- 剩餘時數一律向下取整（34.01 → `34`），不虛報。
- 重置倒數格式：≥1 天 `7d22h`、<1 天 `5h`、<1 小時 `42m`。
- 未知 SKU：印出警告並跳過（無法判定核數時不猜），**不得靜默丟棄**。
- storage 加總同時接受 `Codespaces storage` 與 `Codespaces prebuild storage`（同一 GigabyteHours 池）。
- 2026-10 的 UTC 時數為 31 × 24 = 744，不是 720；`selftest` 以此為驗算基準。

---

## 4. 行為規格

### 4.1 CLI 介面

```
gh csm                                  # help（不做 quota fallback）
gh csm quota [--account <login> | --account=<login> | --all] [--month <YYYY-MM> | --month=<YYYY-MM>]
gh csm ls [gh codespace list args] [--account <login> | --account=<login>]
gh csm stop [-c <name>] [--account <login> | --account=<login>]
gh csm start [-c <name>] [snippet] [--account <login> | --account=<login>]
gh csm selftest
gh csm help | --help | --version
```

- 帳號預設 = gh 的 active 帳號；`--account` 指定；`--all` 逐一堆疊面板（僅 quota）。
- `--account` 空白與等號兩種形式皆可；`--all`、`-c` 只有空白形式（`--all=1`、`-c=x` 視為未知參數）。
- `--month` 僅 quota 適用（stop / start 收到直接報錯）：渲染指定 UTC 計費月，格式 `YYYY-MM`、
  月 01–12；晚於當前 UTC 月直接報錯並附目前可用上限，當前月允許，過去月份不設下限。
  空白與等號兩種形式皆可。
- `-c` 與 `--codespace` 同義（`stop` / `start`）。
- `ls` 是 `gh codespace list` 的 pass-through（§4.7）：只消費 csm 自己的 `--account`，其餘參數
  原序轉發；`--all` / `--month` 不適用（直接報錯）。
- `stop` / `start` 不支援 `--all`（一次一個帳號）。
- `stop` / `start` 不帶 `-c` 且 stdin 是 TTY：開共用互動選單 `pick_codespace`（§4.3.1）；
  `start` 選定後再提示指令（Enter = 互動 shell，§4.4）。
- 不帶 `-c` 且非 TTY：明確報錯，不在 CI 卡住。
- 未知子命令 / 未知旗標：訊息 + usage，exit 1。

### 4.2 quota 面板（每帳號一個）

```
── octocat · 2026-10 UTC · resets in 7d22h ────
Compute  ████████░░░░░░░░░░░░  43%   51.98/120 core-hrs · 2-core 34h · 4-core 17h left
Storage  ░░░░░░░░░░░░░░░░░░░░   <1%   0.01/15 GB-month
```

`--all`：每帳號一個面板、空行分隔，`(active)` 只在此模式出現；缺 `user` scope 的帳號壓成單行提示。

```
── octocat (active) · 2026-10 UTC · resets in 7d22h ────
Compute  ████████░░░░░░░░░░░░  43%   51.98/120 core-hrs · 2-core 34h · 4-core 17h left
Storage  ░░░░░░░░░░░░░░░░░░░░   <1%   0.01/15 GB-month

── hubot · 2026-10 UTC · resets in 7d22h ────
  ⚠ missing user scope — unlock: gh auth switch --user hubot && gh auth refresh -h github.com -s user,codespace && gh auth switch --user octocat
```

範例中 octocat 為 active，故解鎖提示為三步（切回原 active）；目標帳號本身是 active 時僅需 refresh 單步（見規則 7）。

規則：

1. header 固定 `── <login>[ (active)] · <YYYY-MM> UTC[ · resets in <countdown>] ────`；
   `resets in …` 只在查詢當前 UTC 月時出現，歷史月份省略（倒數只對當前計費月有意義）。
2. 進度條寬度 20 格，`█` 已用 / `░` 未用；綠 `<70`、黃 `70–89`、紅 `≥90`。
3. Compute 行：`<used>/<limit> core-hrs`（2 位小數）＋`2-core <N>h · 4-core <M>h left`；
   時數整數向下取整，`2-core … left` 整段以粗體＋進度條同色標示（非 TTY / `NO_COLOR` 時退化為純文字）。
4. Storage 行恆顯示（無顯示門檻）；`storage_pct == 0` 且用量 > 0 → `<1%`；數值 2 位小數；
   不再顯示 GB-h ÷ 小時的換算細節。storage 除以查詢月份（`--month`）的實際時數。
5. quota 不含 codespace 清單與行動建議（理由見 §9.1），**只呼叫 billing API**；
   `fetch_codespaces` 僅供 `stop` / `start` 的選單使用。
6. 顏色只在 stdout 是 TTY 且 `NO_COLOR` 未設時輸出。
7. 缺 `user` scope 的帳號：只顯示單行解鎖提示，不呼叫 billing API、不影響其他帳號。
   目標帳號是 active → `unlock: gh auth refresh -h github.com -s user,codespace`；
   非 active → 三步 `gh auth switch --user <target> && gh auth refresh -h github.com -s user,codespace
   && gh auth switch --user <目前 active>`（切回原 active，與 README 一致）。
8. billing 失敗：stderr 單行錯誤＋面板單行提示，不中斷其他帳號；該帳號計為失敗。
9. 無頁尾結算延遲註記；期待管理（約每小時結算一次）由 README 承接。
10. exit code：所有要求的帳號都成功渲染 = 0；任一帳號讀不到（缺 `user` scope / billing 失敗）= 1；
    `--all` 照印全部面板後仍回 1。參數 / 認證錯誤維持 `die` = 1。

### 4.3 stop

- 無 `-c`：呼叫共用選單 `pick_codespace`（§4.3.1）列出該帳號 codespaces
  （編號 / name / state / 核數 / last used），讀取編號 → 二次確認 `[y/N]` →
  `gh codespace stop -c <name>`；輸入 `q` 取消。
- 有 `-c`：直接停，不再確認。
- 已是 Shutdown 的項目拒絕停止，並提示改用 delete。
- 非 TTY 且無 `-c` → 直接報錯（避免在 CI 卡住）。
- 內部是 `gh codespace stop` 的薄包裝，不自己打 POST。

#### 4.3.1 `pick_codespace`（stop / start 共用）

- 輸入：`fetch_codespaces` 的 TSV（name / state / cpus / last_used_at / repository）；
  第二參數為動詞，組出提示 `Select which codespace to <verb> (q to cancel): `。
- 輸出：全域 `PICK_NAME`、`PICK_STATE`（選定的 name / state）。
- `q`：印 `Cancelled`，回 1（呼叫端視為正常結束，不再動作）。
- 非數字輸入或編號超界：`die`（英文訊息）。
- 列表渲染只此一份，`stop` 與 `start` 不得各自複製。

### 4.4 start

- 無 `-c` 且 stdin 是 TTY：`pick_codespace` 選定後提示
  `Command to run (Enter for interactive shell): `；輸入非空 = snippet（跑完退出），
  Enter = 互動 shell。
- 無 `-c` 且非 TTY：`start requires -c <name> in non-interactive mode`。
- `gh codespace ssh` 為底：選到 Shutdown 的 codespace 是合法操作，gh 會自動喚醒（原始碼
  `waitUntilCodespaceConnectionReady` 會對 Shutdown 狀態呼叫 `StartCodespace`，上限 5 分鐘）。
- 有 snippet：`gh codespace ssh -c <name> -- "<snippet>"`，執行完退出。
- 無 snippet：`gh codespace ssh -c <name>` 進互動 shell。
- 多個位置參數會以空白串接成單一 snippet。

### 4.5 多帳號

- 列舉：`gh auth status --json hosts`（gh 無專用 list API）。
- 每個帳號的 gh 呼叫：`GH_TOKEN=$(gh auth token --user <login>)`，
  並以 `env -u GITHUB_TOKEN` 確保不被環境變數蓋掉。
- token 只在該次子行程的環境變數中出現，不寫入檔案、不 echo。
- 帳號清單與 scopes 只抓一次（`AUTH_JSON` 快取）。

### 4.6 selftest（零網路）

內嵌 octocat 2026-10 真實帳單 fixture，驗證：

1. `sku_cores`：2 / 4 / 16 / 32-core 解析正確；未知 SKU 回非零。
2. fixture 聚合：compute 51.98 core-hrs、storage 5.40 GB-h。
3. 未知 SKU 觸發警告，且不計入 compute。
4. 51.98 / 120 = 43%。
5. 色階邊界：69 綠、70 黃、89 黃、90 紅。
6. 當月時數：2026-10 = 744、2026-02 = 672、2024-02 = 696。
7. storage 換算：5.397 / 744 ≈ 0.0073 GB-month。
8. 剩餘時數（向下取整）：(120 − 51.98)/2 = 34、/4 = 17；額度用完 = 0。
9. RFC3339 `+08:00` 與 `Z` 表示同一瞬間（時區轉換正確）。
10. 進度條 0% / 100% 邊界。
11. 重置倒數：固定 now epoch + 2026-10 → `7d22h`。
12. 重置倒數跨年：2026-12 → 2027-01-01 前 5h → `5h`；不足 1 小時 → `42m`。
13. prebuild storage 與一般 storage 同池計入，且不觸發未知 SKU 警告。
14. `--month` 驗證：合法（2024-02、當前 UTC 月）、格式錯、`00`、`13`、未來月
    （錯誤訊息含 `latest available`）；`set_billing_period` 覆寫可離線測試。
15. `ls` 轉發：subshell 內 stub `gh_as` / `load_auth` / `require_scope`，驗 `--account` 被消費、
    其餘參數原序轉發、無參數不炸（空 `REST` 護欄）、`--month` 與 `--all`（真的
    `resolve_login`）被拒。
16. 版本漂移守門：腳本自身目錄的 `CHANGELOG.md` 含 `## [<VERSION>]` 段落。不檢查 git tag
    （CI 是 shallow clone，沒有 tags）。

任一失敗 → 印「期望 vs 實際」並 exit 1；全過 → exit 0。
輸出（標題、案例名稱、`selftest: <n> passed, 0 failed`）為英文，與其他使用者可見輸出一致。
案例數為 43 項＋2 項 locale 回歸錨（其中 comma-locale 錨在未安裝逗號小數 locale 的機器會跳過）。

### 4.7 ls

- `gh csm ls` 是 `gh codespace list` 的薄包裝：csm 只消費自己的 `--account`（含等號形式），
  其餘參數原序留在 `REST` 轉發給 gh。
- **stream-through 是不可協商的紅線**：不捕捉 stdout（無 `$(...)`）、不接管線、不套自訂 pager、
  不重排表格、不改顏色與 TTY 判斷；`gh codespace list` 的 exit code 原樣成為本子命令的 exit code。
  旗標與欄位因此不在本文件複刻（避免 gh 版本漂移造成文件過時），`--json` 等上游輸出直接穿透。
- 身分：`load_auth` → `resolve_login`（預設 active 帳號）→ `require_scope <login> codespace` →
  `gh_as <login> codespace list …`，輸出即該帳號的 `gh codespace list`。
- `--all` 不適用：沿用 `resolve_login` 的共用拒絕（`--all does not apply to this subcommand`）。
- `--month` 不適用：比照 stop / start 的 `--month only applies to quota`。
- `-h` / `--help` 由 `parse_account_flags` 攔截，印 csm 自己的 usage，不是
  `gh codespace list --help`；要查上游旗標請直接跑 `gh codespace list --help`。
- 無參數時走無參數分支；必須以 `[ "${#REST[@]}" -gt 0 ]` 護欄判斷，bash 3.2 + `set -u` 下
  展開空陣列會 unbound。

---

## 5. 地雷清單

### 5.1 `gh cs` 是內建 alias

`gh cs` = `gh codespace`，gh 不允許 extension 覆蓋 core command/alias。
裝了 `gh-cs` 只能用 `gh extension exec cs` 呼叫，等於廢掉。故專案名 `gh-csm`，
執行檔名必須與 repo 同名且含前綴（`gh-csm`，不是 `csm`）。

### 5.2 計費月是 UTC

本地 11/1 凌晨時 UTC 仍是 10/31，查「11 月」會拿到空陣列——這不是 bug。
查詢鎖 UTC 月；quota 的 header 與重置倒數也全走 UTC，所以不再另註本地交界。
只有 `stop` / `start` 選單的 last used 轉成本地時間。

### 5.3 `netAmount == 0` 過濾會漏抓超額

免費額度內每筆 `discountAmount == grossAmount`、`netAmount == 0`，看起來很適合當
「免費用量」的 filter。但超出額度後 `netAmount > 0`，該筆會被濾掉 → 用量少算、警示失效。
**一律用 `quantity` 加總。**

### 5.4 storage 的金額欄位不能當美元

storage 的 `grossAmount` 有 ÷730 未換算的怪癖（quantity 0.406576944 GB-h 對應
grossAmount 0.028460376，即 ×0.07/730）。金額僅供參考，計算只用 `quantity`。

### 5.5 codespace 內的 `GITHUB_TOKEN` 會蓋掉登入

`GITHUB_TOKEN` 的優先序高於 `GH_TOKEN` 與 gh 儲存的憑證。在 codespace 內執行本工具，
會變成以該 repo-scoped token 查詢，症狀是 403 或「沒有 active 帳號」。
處理：程式啟動時就 `unset GITHUB_TOKEN`（並提示一次），所有 gh 呼叫再用 `env -u GITHUB_TOKEN` 保險。

### 5.6 Shutdown 的 codespace 仍持續計 storage

實測 shutdown 狀態每天仍有 +0.13~0.18 GB-h。所以「刪除優先於停止」。
quota 不再列清單（§9.1）：要停用 `gh csm stop`，要真的止血用 `gh codespace delete`。

### 5.7 `gh auth refresh` 沒有 `--user`

inactive 帳號要 `gh auth switch --user <login>` → `gh auth refresh -h github.com -s user,codespace`
→ 切回原帳號。`gh auth token` 才有 `--user`，所以讀取階段可以全自動，只有解鎖要人工一次。

### 5.8 billing 端點只認 classic OAuth

fine-grained PAT 不支援；不要提供 PAT 路徑，少一條要維護的認證。

### 5.9 bash 3.2（macOS 系統 bash）

禁用：`declare -A`、`mapfile`、`readarray`、`${var,,}`、`${var^^}`、`&>>`、`local -n`。
本檔另外踩到一個實機 bug：**`$var` 緊貼非 ASCII 字元會被 bash 3.2 吃進變數名**
（`"$sku」"` → `sku」: unbound variable`），所以中文字串前的變數一律寫 `${var}`。
`"${arr[@]}"` 在 bash 3.2 + `set -u` 對空陣列會報 unbound，故陣列只以索引存取。

### 5.10 發佈模型與 release asset 命名

定案：`main` 分支即發佈通道，`CHANGELOG.md` 記錄版本、輕量 tag 當版本錨點；**不發** GitHub
Release、不提供 asset（決策理由見 §9.7）。

- 安裝＝clone `main` HEAD、升級＝pull `main`；tag 不參與安裝升級，只供人閱讀、compare
  與 `gh extension install --pin <tag>` 釘版本。
- 分工：`CHANGELOG.md` 記「改了什麼」（使用者看）；本文件 §9 記「為什麼」（維護者看）。
  同一件事不在兩處各寫一份。
- tag 打在 `VERSION` bump 的同一個 commit 上，一律輕量 tag（`git tag vx.y.z`，不加 `-a` / `-m`）：
  版本說明的載體是 CHANGELOG.md，tag 只負責指位置。
- 平台後綴 asset 禁令維持：若未來真要發 asset，檔名**不要**長得像 `gh-csm-darwin-arm64`——
  gh 會誤判為 binary extension 而走不同安裝路徑。

### 5.11 其他實作細節

- `/user/codespaces` 預設分頁 30 筆，帶 `per_page=100`。
- `gh codespace list --json machineName` 給的是機型 id，不是核數（見 §2.3）。
- `gh codespace ssh` 的參數解析是「第一個非旗標參數之後全部視為 command」，
  用 `--` 分隔最安全（snippet 以 `-` 開頭時也不會被當旗標）。
- 顏色需判斷 TTY，否則管線輸出會夾 ANSI escape。
- 小數點：數值 awk 呼叫點逐處前綴 `LC_ALL=C`，輸出格式化與輸入解析都釘。gawk 在逗號小數
  locale（如 `de_DE`）會讓 `printf "%.2f"` 印出 `51,98`，也會把小數字串讀錯；C locale 固定
  點號。**不設全域 locale**：行程環境會經 `gh`→`ssh` 子行程繼承，macOS ssh `SendEnv LANG LC_*`
  轉發到遠端 sshd（`AcceptEnv LC_*`），遠端 shell 以 C locale 啟動會讓 codespace 內中文變亂碼。

### 5.12 prebuild storage 字串未經實機驗證

官方 SKU id 是 `codespaces_prebuild_storage`，本工具接受對應的人類可讀字串
`Codespaces prebuild storage`。若 GitHub 實際回傳不同標籤，會落入未知 SKU 的 warn 並跳過——
少算但不誤算（安全方向失敗）。有真實 prebuild 用量時應以 API 回應校正此字串。

---

## 6. 測試策略

| 層級 | 方式 |
|---|---|
| 離線 | `gh-csm selftest`：內嵌真實 fixture 驗公式、SKU 解析、色階、時區轉換、重置倒數、月份驗證、`ls` 參數轉發與拒絕 |
| 靜態 | `shellcheck gh-csm`（0 錯誤） |
| CI | `.github/workflows/ci.yml`：Ubuntu（bash 5.x）＋ macOS（`/bin/bash` 3.2）跑 shellcheck + selftest，零網路零 secrets |
| 實機唯讀 | `gh csm quota`（active 帳號）、`--all`、`--account`（缺 scope 帳號走解鎖提示）、`--month`（歷史月）、`gh csm ls`（含 `--account`） |
| 寫入操作 | `stop` / `start` 只驗參數解析、選單渲染與 scope 檢查，不在開發期實際執行 |

---

## 7. 未實作（刻意）

- `--watch` 常駐刷新（資料源本身延遲約 1 小時，收益低）。
- `rm` / `ssh` 等原生子命令的再包裝（`gh codespace` 已有；理由見 §9.6）。
- 額度方案自動偵測（API 查不出 Free/Pro，只能靠環境變數覆寫）。
- 跨 host（僅 `github.com`）。

---

## 8. 檔案結構與拆檔觸發線

現況：單一執行檔 `gh-csm`（約 800 行）。拆檔不是目的，是規模的代價；未觸發前所有東西
留在單檔，避免為了「看起來整齊」付出跨檔追蹤與 bootstrap 成本。

**觸發條件（任一成立才拆）**：

- 檔案超過約 1500 行。
- 出現新子系統：`--watch` 常駐刷新、組態檔（如 `~/.config/gh-csm/`）、
  JSON／其他機器可讀輸出格式。

註：`ls` 轉發 gh 的輸出（含 `--json`）不算新增 JSON 子系統——csm 不解析也不產生該格式，
只是把子行程的 stdout 原樣穿透，不觸發拆檔。

**拆法**：`lib/` 下按子系統切檔（`auth` / `billing` / `panel` / `selftest`），
機械式 `git mv` 搬移後在 `gh-csm` 補路徑 bootstrap（`source "$(dirname "$0")/lib/*.sh"`），
不改邏輯、不趁機重構；行為測試（`selftest`）必須在搬移前後輸出相同。

---

## 9. 設計決策記錄

### 9.1 面板極簡化（0.2.0）

使用者實跑後回饋「資訊太多」，0.2.0 把面板收成三行：header（帳號／UTC 月／離重置倒數）
＋ compute 一行 ＋ storage 一行。

- **移除**：codespace 清單、行動建議、「建議」區塊、頁尾結算延遲註記、獨立計費月行、
  本地／UTC 交界註記、GB-h ÷ 小時的換算細節。
- **職責搬遷**：清單與「挑一個停掉」的操作本來就是 `stop` 的互動選單在做，面板不再重複。
- **副作用（正面）**：面板只剩一次 billing API 呼叫，不再需要 `codespace` scope；
  `fetch_codespaces` 保留給 `stop`。
- **期待管理**：「約每小時結算一次」從輸出移到 README，需要時再查。

### 9.2 介面英文化與共用選單（0.2.1）

公開發布前把使用者可見輸出與 README 轉英文；SPEC 與程式碼註解維持中文（內部文件與維護者語言）。

- **範圍**：usage / help、`die` / `warn` 訊息、面板、`stop` / `start` 選單與確認、
  selftest 輸出、`--version`、README。程式碼註解與本文件不動。
- **面板版面不變，只換字串**：header 為 `resets in <countdown>`，compute 行為
  `2-core <N>h · 4-core <M>h left`；粗體＋狀態色改為覆蓋整個 `2-core … left` 片段（含 `left`）。
- **`start` 無 `-c`**：與 `stop` 共用 `pick_codespace`（§4.3.1），選定後提示
  `Command to run (Enter for interactive shell): `；Enter = 互動 shell。
  非 TTY 直接報錯，行為對映 `stop`。
- **不做 i18n 框架**：單一語言、單一檔案，字串直接寫死；真要加語言時再抽成字串表。

### 9.3 子命令改名 `quota` / `start`（0.3.0）

公開發布前一次定案，避免安裝者學一套名字後再改一輪（churn）。

- **`quota`**：面板講的就是額度，名字自解釋；`--account` / `--all` 仍是它的旗標。
- **`start`**：與 `stop` 對稱（start / stop），語意也比「喚醒並執行」的舊動詞直接。
- **不做舊名 alias**：兩套名字並存等於永久背兩份介面與測試；舊名一律 unknown subcommand。
- **同步範圍**：usage / help / 錯誤訊息 / picker 動詞 / README / 本文件。
  `Command to run`、`Waking … and running:` 是一般英文，不是子命令名，維持原樣。

### 9.4 首次公開發布（1.0.0）

首次公開發布於 [BibbyChung/gh-csm](https://github.com/BibbyChung/gh-csm)，tag `v1.0.0`。

- **公開起點即 1.0.0**：命令面（`quota` / `stop` / `start`）、輸出格式與 API 契約已定案，
  安裝者不必再跟一次改名；0.1.0–0.3.0 是發布前的內部迭代，不另發佈。
- **首發範圍**：`quota`（單次 billing 呼叫、三行面板）、`stop` / `start`（共用 picker）、
  多帳號（`--account` / `--all`）、離線 `selftest`。
- **授權**：MIT，見 [LICENSE](LICENSE)。

### 9.5 1.1.0：exit code、歷史月份、prebuild storage

- **quota exit code 語意化**：全部要求的帳號成功渲染 = 0；任一帳號讀不到（缺 `user` scope /
  billing 失敗）= 1，`--all` 印完所有面板後彙總。原本永遠 0，腳本與 CI 無法判斷成敗。
- **`--month YYYY-MM`**：只對 quota 生效；未來月拒絕（錯誤附當前可用上限），當前月允許；
  非當前月省略 `resets in …`，因為倒數只描述當前計費月。`set_billing_period` 接受覆寫參數，
  月份驗證與 storage 換算可在 selftest 離線測試。
- **prebuild storage 同池**：`Codespaces prebuild storage` 併入 storage 加總；字串由 SKU id
  推導、未經實機驗證（§5.12），不認得就 warn，不會算錯。
- **解鎖提示補切回**：非 active 帳號的三步提示包含切回目前 active，照做不再改掉使用者全域
  gh 登入；active 帳號維持單行 refresh。
- **CI**：Ubuntu + macOS 雙平台跑 shellcheck 與 selftest，覆蓋 bash 5.x 與 3.2。

### 9.6 1.2.0：新增 `ls`（`gh codespace list` 的薄包裝）

- **為何加**：§9.1 刻意把 codespace 清單移出 quota 面板，`stop` 的選單又要求 TTY；多帳號環境
  原本得先 `gh auth switch` 才看得到別的帳號有哪些 codespace。`ls` 補上這個缺口（唯讀、零計算），
  quota 仍維持單次 billing 呼叫。
- **為何只包 `ls`，不包 `rm` / `ssh`**：`ls` 唯讀且輸出可完整穿透，包裝只多出「選身分」的價值；
  `rm` 是破壞性操作、`ssh` 是互動 session，包一層只多一個要維護與測試的介面，原生
  `gh codespace rm` / `ssh` 已足夠。§7 保留這兩項為刻意未實作。
- **pass-through 契約**：csm 不複刻上游旗標與欄位（避免 gh 版本漂移讓文件與行為不一致），
  參數原序轉發、stdout 與 exit code 原樣穿透；`--all` / `--month` 不適用（§4.7）。

### 9.7 版本記錄：CHANGELOG + 輕量 tag

- **為何是 changelog + tag 兩件套**：使用者要知道「這版多了什麼」，維護者要知道「為什麼這樣
  設計」，兩者壽命不同。CHANGELOG.md 以 Keep a Changelog 1.1.0 格式只寫 what（面向使用者）；
  設計理由留在 §9，後續版本可直接引用，不必在 changelog 重述一遍。
- **為何不發 GitHub Release**：gh script extension 的安裝與升級只認 repo 的 `main` HEAD
  （install＝clone、upgrade＝pull），tag 與 release 對功能零影響。Release 只是多一層要同步的
  介面，asset 還得繞 §5.10 的 binary extension 誤判，收益為零。
- **tag 的價值**：版本錨點——GitHub tags 頁可跳到對應 commit、compare 連結能固定區間、
  `gh extension install --pin <tag>` 可釘住版本；這三件事都不需要 Release。
- **為何不用 annotated tag**：版本說明的載體是 CHANGELOG.md；annotated tag 自帶 message 等於
  同一份說明存兩處，遲早不同步。輕量 tag 只負責指位置，`git tag vx.y.z <commit>` 即可。
  既有 `v1.0.0` 是早期建立的 annotated tag，保留不動；1.1.0 起一律輕量。

### 9.8 1.2.1：locale 收斂到 awk 呼叫點

- **全域 export 改逐點前綴**：移除開頭的 `export LC_ALL=C`；9 個數值 helper（`fdiv`、`fgt`、
  `fclose`、`fmt2`、`fmt4`、`pct`、`remaining_hours`、`sum_compute`、`sum_quantity`）的 awk
  呼叫一律改為 `LC_ALL=C awk …`。純字串 awk（`is_positive_number`、`epoch_from_iso`、
  `accounts_inline`）不釘，維持 locale 透傳。
- **污染鏈**：腳本 export → `gh codespace ssh` 子行程繼承 → macOS ssh `SendEnv LANG LC_*` →
  遠端 sshd `AcceptEnv LC_*` → 遠端 shell 以 C locale 啟動 → codespace 內中文亂碼。
- **保護不變**：逗號小數 locale 下格式化仍輸出點號、輸入解析仍以點號讀取；selftest 新增
  逗號 locale 回歸錨（找不到 locale 則跳過）與「腳本不得全域 export」靜態斷言。
- **版本**：1.2.1（原 1.1.1，未發佈即併入）。
