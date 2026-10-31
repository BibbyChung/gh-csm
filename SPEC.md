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

三個交付檔：`gh-csm`（程式）、`README.md`（使用者）、`SPEC.md`（本文件）。

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

實測彙總（bb9755，2026-10 UTC，21 筆）：

| SKU | unitType | Σ quantity |
|---|---|---|
| `Codespaces compute 2-core` | Hours | 7.76645 |
| `Codespaces compute 4-core` | Hours | 9.11165 |
| `Codespaces storage` | GigabyteHours | 5.397099746 |

→ compute 51.98 core-hrs（43% of 120）、storage 5.397 GB-h ÷ 744 h ≈ 0.0073 GB-month。
這組數字就是 `selftest` 的內嵌 fixture。

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
剩餘可跑小時(cores) = (compute_limit − compute_core_hours) / cores
```

- 額度預設 `GH_CSM_COMPUTE_LIMIT=120`、`GH_CSM_STORAGE_LIMIT=15`（GitHub Free）；Pro 為 180 / 20。
- 顏色門檻：`<70` 綠、`70–89` 黃、`≥90` 紅（以整數百分比判斷）。
- `storage_pct == 0` 但用量 > 0 → 顯示 `<1%`。
- 未知 SKU：印出警告並跳過（無法判定核數時不猜），**不得靜默丟棄**。
- 2026-10 的 UTC 時數為 31 × 24 = 744，不是 720；`selftest` 以此為驗算基準。

---

## 4. 行為規格

### 4.1 CLI 介面

```
gh csm                                  # help（不做 dash fallback）
gh csm dash [--account <login> | --account=<login> | --all]
gh csm stop [-c <name>] [--account <login> | --account=<login>]
gh csm run -c <name> [snippet] [--account <login> | --account=<login>]
gh csm selftest
gh csm help | --help | --version
```

- 帳號預設 = gh 的 active 帳號；`--account` 指定；`--all` 逐一堆疊面板（僅 dash）。
- `--account` 空白與等號兩種形式皆可；`--all`、`-c` 只有空白形式（`--all=1`、`-c=x` 視為未知參數）。
- `stop` / `run` 不支援 `--all`（一次一個帳號）。
- 未知子命令 / 未知旗標：訊息 + usage，exit 1。

### 4.2 dash 面板（每帳號一個）

```
── <login> (active) ──────────────────────────────
計費月 2026-10 (UTC)  ·  本地 2026-11-01（計費月以 UTC 為準）
Compute  ████████░░░░░░░░░░░░  43%  已用 51.98 / 120 core-hrs
         → 2-core 機器還能跑 ~34.01 小時；4-core 機器還能跑 ~17.01 小時
Storage  ░░░░░░░░░░░░░░░░░░░░   <1%  已用 0.0073 / 15 GB-month（5.40 GB-h ÷ 744 h）
Codespaces
  NAME                                   STATE      MACHINE  LAST USED
  foul-hobgoblin-qvvqx795q4qjf449r       Shutdown   2-core   2026-10-30 23:26  (持續吃 storage)
建議
  gh codespace delete -c foul-hobgoblin-qvvqx795q4qjf449r    # Shutdown 仍計 storage，刪除才停損

Billing 約每小時結算一次，數字非即時。
```

規則：

1. 進度條寬度 20 格，`█` 已用 / `░` 未用。
2. 月份標題固定 UTC；`local 月 ≠ UTC 月` 時加註「計費月以 UTC 為準」。
3. `last_used_at` 轉本地時區顯示（RFC3339 → epoch → `date`，跨 darwin/GNU）。
4. Shutdown 狀態在清單標註「持續吃 storage」。
5. 行動建議：Shutdown 且有 storage 耗用 → `gh codespace delete`；其餘（執行中）→ `gh codespace stop`。
6. 顏色只在 stdout 是 TTY 且 `NO_COLOR` 未設時輸出。
7. 缺 `user` scope 的帳號：面板只顯示解鎖提示，不呼叫 billing API、不影響其他帳號。
8. 缺 `codespace` scope：仍顯示額度，清單顯示解鎖提示。
9. 頁尾固定一行結算延遲說明。

### 4.3 stop

- 無 `-c`：列出該帳號 codespaces（編號 / name / state / 核數 / last used），
  讀取編號 → 二次確認 `[y/N]` → `gh codespace stop -c <name>`；輸入 `q` 取消。
- 有 `-c`：直接停，不再確認。
- 已是 Shutdown 的項目拒絕停止，並提示改用 delete。
- 非 TTY 且無 `-c` → 直接報錯（避免在 CI 卡住）。
- 內部是 `gh codespace stop` 的薄包裝，不自己打 POST。

### 4.4 run

- `gh codespace ssh` 為底：Shutdown 的 codespace 由 gh 自動喚醒（原始碼
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

內嵌 bb9755 2026-10 真實帳單 fixture，驗證：

1. `sku_cores`：2 / 4 / 16 / 32-core 解析正確；未知 SKU 回非零。
2. fixture 聚合：compute 51.98 core-hrs、storage 5.40 GB-h。
3. 未知 SKU 觸發警告，且不計入 compute。
4. 51.98 / 120 = 43%。
5. 色階邊界：69 綠、70 黃、89 黃、90 紅。
6. 當月時數：2026-10 = 744、2026-02 = 672、2024-02 = 696。
7. storage 換算：5.397 / 744 ≈ 0.0073 GB-month。
8. 剩餘時數：(120 − 51.98)/2 ≈ 34.01、/4 ≈ 17.01。
9. RFC3339 `+08:00` 與 `Z` 表示同一瞬間（時區轉換正確）。
10. 進度條 0% / 100% 邊界。

任一失敗 → 印「期望 vs 實際」並 exit 1；全過 → exit 0。

---

## 5. 地雷清單

### 5.1 `gh cs` 是內建 alias

`gh cs` = `gh codespace`，gh 不允許 extension 覆蓋 core command/alias。
裝了 `gh-cs` 只能用 `gh extension exec cs` 呼叫，等於廢掉。故專案名 `gh-csm`，
執行檔名必須與 repo 同名且含前綴（`gh-csm`，不是 `csm`）。

### 5.2 計費月是 UTC

本地 11/1 凌晨時 UTC 仍是 10/31，查「11 月」會拿到空陣列——這不是 bug。
查詢鎖 UTC 月，顯示轉 local，交界期加註。

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

實測 shutdown 狀態每天仍有 +0.13~0.18 GB-h。所以「刪除優先於停止」，
且 dash 只對 Shutdown + 有 storage 耗用的機器給 delete 建議。

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

### 5.10 未來 release asset 命名

若之後發 release，asset 檔名**不要**長得像 `gh-csm-darwin-arm64`——gh 會誤判為
binary extension 而走不同安裝路徑。目前不需要 release（`gh extension install` 直接 clone repo）。

### 5.11 其他實作細節

- `/user/codespaces` 預設分頁 30 筆，帶 `per_page=100`。
- `gh codespace list --json machineName` 給的是機型 id，不是核數（見 §2.3）。
- `gh codespace ssh` 的參數解析是「第一個非旗標參數之後全部視為 command」，
  用 `--` 分隔最安全（snippet 以 `-` 開頭時也不會被當旗標）。
- 顏色需判斷 TTY，否則管線輸出會夾 ANSI escape。

---

## 6. 測試策略

| 層級 | 方式 |
|---|---|
| 離線 | `gh-csm selftest`：內嵌真實 fixture 驗公式、SKU 解析、色階、時區轉換 |
| 靜態 | `shellcheck gh-csm`（0 錯誤） |
| 實機唯讀 | `gh csm dash`（active 帳號）、`--all`、`--account`（缺 scope 帳號走解鎖提示） |
| 寫入操作 | `stop` / `run` 只驗參數解析與 scope 檢查，不在開發期實際執行 |

---

## 7. 未實作（刻意）

- `--watch` 常駐刷新（資料源本身延遲約 1 小時，收益低）。
- `ls` / `rm` / `ssh` 等原生子命令的再包裝（`gh codespace` 已有）。
- 額度方案自動偵測（API 查不出 Free/Pro，只能靠環境變數覆寫）。
- 跨 host（僅 `github.com`）。
