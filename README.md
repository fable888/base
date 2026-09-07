# Session bot Base (AMM) — Mode 3, chạy giấy

Luật tối cao: `CLAUDE.md`. Bản đồ đọc: `docs/DOC_MAP.md`. Stack khóa: TypeScript + viem (Node 20+, npm). README này chỉ nói hai việc: **lệnh thường dùng** và **chỉnh `config.toml` để chạy kèo Mode 3**.

## Bot này làm gì

- Chạy trên Base (`chain_id = 8453`), quét 3 sàn: Aerodrome Slipstream, Uniswap V3, Pancake V3. Aero cổ điển tắt.
- **Mode 3 duy nhất**: mua token **đã sống** — pool đã có giao dịch Swap thật trong cửa sổ gần nhất, có một vế WETH, LP phía WETH ≥ `min_lp_usd`. Kiểu bảng DexScreener. **Không** sniper pair mới đẻ (Mode 1/2 đã bỏ).
- Mỗi ứng viên đi qua 12 cửa lọc fail-closed (sim mua rồi bán, tax, impact, proxy, LP…). Rớt 1 cửa = bỏ, log `FILTER_FAIL:<cửa>`.
- Qua hết cửa ⇒ mua giấy đúng size `buy_weth_wei` (0,0028 WETH ≈ $7), ghi `BUY_PAPER`, giữ 1 vị thế trong `state/position.json`.
- Thoát: 0–60 phút đầu lãi ròng ≥ +5 % thì bán; sau 60 phút lãi ròng ≥ +3 % thì bán; **lỗ thì ôm** (`require_stop_loss = false`); đủ 24 giờ chưa bán ⇒ `STOPPED_HOLD`: dừng săn, giữ vị thế, không bán.
- `SCAN` khác `LIVE`: `live_* = false` chỉ cấm gửi tx thật, **không** tắt quét.

> **Cảnh báo.** Bot chỉ chạy **giấy** (`dry_run = true`, `allow_live = false`, `bot_armed = false`): RPC đọc thật, mua/bán là mô phỏng, không có `eth_sendRawTransaction`. Chỉ lật sang live khi chủ tự tay bật có chủ đích, sau GĐ6 theo `CLAUDE.md`. Claude Code không bao giờ lật 3 công tắc này.

## Cài & chạy lần đầu

```
npm install
copy .env.example .env      # PowerShell / cmd. Git Bash: cp .env.example .env
```

Điền vào `.env`: `CHAINSTACK_HTTP` và `CHAINSTACK_WS` (URL Chainstack Base kèm token). `PRIVATE_KEY` để trống khi chạy giấy. Không commit `.env`. `vps.json` giữ nguyên chữ `REPLACE`, URL thật chỉ nằm trong `.env`.

Chạy giấy 10 phút rồi tự dừng sạch:

```
npx tsx src/index.ts --seconds=600
```

Dòng đầu phải là `chain ok: eth_chainId=0x2105`. Log ghi vào `logs/bot.jsonl`.

## Lệnh thường dùng

Mỗi lệnh chạy từ thư mục gốc repo (bot đọc `config.toml`, `vps.json`, `.env` theo đường dẫn tương đối).

**Chạy giấy có hạn giờ** (giây; hết giờ tự gỡ WSS, xả log, thoát):

```
npx tsx src/index.ts --seconds=7200
```

**Chạy giấy không hạn giờ** (dừng bằng Ctrl-C):

```
npx tsx src/index.ts
```

`npm start` là cùng lệnh trên nhưng không truyền được `--seconds`, nên dùng thẳng `npx tsx`.

**Chỉ định file config khác**: chưa có flag `--config`. Bot luôn đọc `./config.toml` và `./vps.json` trong thư mục hiện tại. Muốn giữ nhiều kèo: chép thành `config.kèo-A.toml`, khi chạy thì copy đè lên `config.toml` rồi chạy.

**Build / kiểm kiểu / test**:

```
npm run typecheck     # tsc --noEmit
npm test              # vitest run — 32 file, 1673 test (2026-09-07)
```

Không có bước build riêng: `tsx` chạy thẳng TypeScript.

**Xem log** (`log_dir = "./logs"`, file `bot.jsonl`, mỗi dòng một JSON). Field `key` là nhãn ổn định để grep:

| key | nghĩa |
|---|---|
| `SCAN_SWEEP` | một lượt quét Swap (số log, số pool sống, số pool đã xác minh) |
| `SCAN` | ứng viên đã xác minh on-chain, sắp vào cửa lọc |
| `FILTER_FAIL:<cửa>` | rớt cửa nào (`pair_no_weth`, `lp_usd`, `tax`, `impact`, `proxy_or_upgrade`, …) |
| `BUY_PAPER` | mua giấy xong, có `entry_weth` |
| `TP_PHASE1` / `TP_LATE` | bán giấy vì chốt lời trước / sau `phase1_sec` |
| `HOLD_LOSS` | đang lỗ, ôm (không bán) |
| `STOPPED_HOLD` | đủ `timeout_sec`, dừng săn, giữ vị thế |

Git Bash:

```
tail -f logs/bot.jsonl
grep '"key":"SCAN"' logs/bot.jsonl | tail
grep -o '"key":"FILTER_FAIL:[a-z_]*"' logs/bot.jsonl | sort | uniq -c | sort -rn
grep '"key":"BUY_PAPER"' logs/bot.jsonl
grep -E '"key":"(TP_PHASE1|TP_LATE|HOLD_LOSS|STOPPED_HOLD)"' logs/bot.jsonl | tail
grep -c eth_sendRawTransaction logs/bot.jsonl     # phải = 0 khi dry_run
```

PowerShell:

```
Get-Content logs\bot.jsonl -Wait -Tail 20
Select-String -Path logs\bot.jsonl -Pattern '"key":"BUY_PAPER"'
(Select-String -Path logs\bot.jsonl -Pattern '"key":"FILTER_FAIL:').Count
```

**Dừng bot**: Ctrl-C (SIGINT) hoặc để `--seconds` hết. Bot gỡ watcher, đóng WSS, xả log rồi in `dừng sạch: ticks=…`. Nếu còn ôm, nó in `CÒN ÔM …` và `state/position.json` vẫn trên đĩa; lần chạy `dry_run` tiếp theo tự xóa file đó (log `position.paper_cleared`) và về Idle. Nếu tiến trình cũ còn treo (cửa sổ khác), đóng cửa sổ đó trước khi chạy lại, vì hai tiến trình cùng ghi một `logs/bot.jsonl`.

**File điều khiển trong `state/`** (tạo file rỗng là đủ):

```
# Git Bash                       PowerShell
touch state/flatten.req          New-Item state\flatten.req -ItemType File   # bán giấy vị thế đang ôm nếu sim bán > 0
touch state/halt.lock            New-Item state\halt.lock -ItemType File     # Halted: không mở lệnh, không flatten cho tới khi xóa file
touch state/disarm.req           New-Item state\disarm.req -ItemType File    # bot_armed về false trong phiên
touch state/reset.req            New-Item state\reset.req -ItemType File     # chỉ khi STOPPED hoặc Idle trống; đang ôm ⇒ reset.blocked
```

**Chỉ soi universe, không mua** (chỉ đọc chain, không chạy runner). Tham số = số lượt quét, mỗi lượt ~80 giây:

```
npx tsx scripts/probe_universe.ts 4
npx tsx scripts/probe_symbol.ts 0x<token1> 0x<token2>
```

Flag scan-only cho bot chính: **chưa có**. Bot luôn mua giấy khi có ứng viên qua cửa. Muốn chỉ xem nó thấy gì, dùng `probe_universe.ts` ở trên.

**Kiểm tra RPC**: chưa có lệnh riêng. Chạy `npx tsx src/index.ts --seconds=5`, dòng `chain ok: eth_chainId=0x2105` là RPC HTTP sống; log có `rpc.block` là WSS sống.

**Kiểm tra số dư ví**: chưa có. Bot giấy không cần ví, không đọc `PRIVATE_KEY`.

**Bơm ứng viên tay** (test cửa lọc với một pool cụ thể, vẫn đi đủ 12 cửa):

```
npx tsx src/index.ts --seconds=120 --inject=venue=uni_v3,token=0x…,pool=0x…,factory=0x…,pool_block=NNN,fee=3000
```

`factory` phải là factory đã pin trong `DEX_REGISTRY.md`; Aero Slipstream dùng `tick_spacing=` thay `fee=`. Sai một trường là bot ném lỗi, không đoán.

## Chỉnh config để chạy kèo

File duy nhất: `config.toml`. Bot đọc **một lần lúc khởi động**, thiếu field bắt buộc hoặc sai kiểu thì không lên (in danh sách lỗi). Sửa xong **phải dừng và chạy lại bot**. Số wei ghi dạng chuỗi `"…"`, không ghi số trần.

### A. Công tắc an toàn (làm trước)

| key | giấy (mặc định) | live | ghi chú |
|---|---|---|---|
| `dry_run` | `true` | `false` | `true` = không bao giờ gửi tx, `tx_hash = paper-no-broadcast` |
| `allow_live` | `false` | `true` | cửa 1 của cổng live mục 4 CLAUDE.md |
| `bot_armed` | `false` | `true` | `false` **không** chặn mua giấy; chỉ chặn gửi tx thật |

Checklist trước khi đụng 2 công tắc live:

1. `docs/STATE.md` có dòng `PROJECT LEAD chấp nhận bằng văn bản: GĐ6 paper OK, được mở GĐ7`. Chưa có = không bật.
2. Chạy giấy ≥ 2 giờ, `grep -c eth_sendRawTransaction logs/bot.jsonl` = 0, có ≥ 10 dòng `SCAN`/`FILTER_FAIL`.
3. `.env` có `PRIVATE_KEY` của ví riêng cho bot, ví có WETH ≥ `buy_weth_wei` và ETH gas ≥ `gas_reserve_eth_wei`.
4. Chỉ bật **một** `live_*` cho venue đầu tiên; calldata venue đó đã pin (GĐ7.2). Chưa pin = bot từ chối (`calldata_miss`).
5. Hiểu rằng 1 lệnh thua là phiên dừng (`net_pnl ≤ 0 ⇒ STOPPED`, reset tay).

### B. Chọn sân (discovery)

| key | đang dùng | tác dụng |
|---|---|---|
| `watch_living` | `true` | bật vòng quét Swap thật (Mode 3). `false` = không có nguồn ứng viên nào ⇒ 0 lệnh |
| `living_min_age_sec` | `0` | phải là số nguyên ≥ 0 để load. Vòng Mode 3 hiện **không** dùng sàn tuổi này (chỉ vòng cũ dùng), để 0 |
| `min_pool_age_sec` | `0` | `0` = cửa `age` PASS không đọc chain. > 0 = đòi block tạo pool, ứng viên Mode 3 không có ⇒ rớt hết |
| `min_lp_usd` | `10000` | LP **chỉ tính phía WETH** (số WETH trong pool × giá WETH/USD đọc từ pool WETH/USDC đã pin). Token vế kia không cộng |
| `scan_aero_classic` | `false` | giữ `false`. Không có code route Aero cổ điển |
| `live_aero_slip` / `live_uni_v3` / `live_pancake_v3` | `false` | chỉ quyết định venue được **gửi tx thật**. `false` **không** tắt quét venue đó |
| `denylist` | 40 address | token lớn / stable / wrapped bị bỏ ngay lúc quét để bot chỉ còn meme (xem mục E) |

Lưu ý: không có key `scan_uni_v3` / `scan_pancake_v3` / `scan_aero_slip` trong file hiện tại; vòng Mode 3 quét cả 3 venue đã pin.

Kèo mẫu:

- **Kèo chặt** (ít lệnh, pool to): `min_lp_usd = 50000` hoặc `100000`. Với 100000 thì gần như chỉ còn token đầu bảng (ICP, VIRTUAL, cbBTC…); nếu không muốn mua token lớn, kết hợp `denylist`.
- **Kèo hiện tại của chủ**: `min_lp_usd = 10000`, `min_pool_age_sec = 0`, `living_min_age_sec = 0`. Đúng lớp DexScreener 10 giờ – 16 ngày tuổi.
- **Không săn pair mới**: không bật lại Mode 1/2 (code đã gỡ khỏi `main`), `scan_aero_classic` giữ `false`.

### C. Size & phiên

| key | đang dùng | tác dụng |
|---|---|---|
| `buy_weth_wei` | `"2800000000000000"` | **KHÓA — đừng đổi.** Vốn gốc phiên = size mọi lệnh. Không compound |
| `buy_denom` | `"weth"` | nhãn; code hiện chỉ mua bằng WETH |
| `max_open_positions` | `1` | phải = 1, khác là không load |
| `cooldown_sec` | `30` | sau khi đóng lệnh, nghỉ N giây mới mở lệnh mới. Không có dòng log riêng: trong lúc nghỉ chỉ thấy `SCAN` mà không có `BUY_PAPER` |
| `max_trades_per_session` | `5` | đủ 5 lệnh ⇒ STOPPED dù lệnh cuối đang lãi |
| `max_session_loss_pct` | `0.50` | lỗ dồn ≥ 50 % × `buy_weth_wei` ⇒ STOPPED |
| `session_usd`, `size_pct` | `100`, `1.0` | nhãn, không đổi size |

Kèo mẫu:

- **1 con, nghỉ 30 s, tối đa 5 lệnh/phiên** (đang dùng): giữ nguyên.
- **Muốn ít lệnh hơn**: `max_trades_per_session = 1`. Sau lệnh đầu bot về STOPPED, muốn chạy tiếp thì tạo `state/reset.req` rồi chạy lại.

### D. Khi nào bán / khi nào ôm

Thứ tự quyết định mỗi nhịp (`src/exit.ts`): flatten → kill (LP tụt, sim bán fail, và RPC không đọc được 2 nhịp **nếu** `kill_unread_while_hold = true`) → SL (nếu bật) → trailing hòa vốn → TP sớm → TP muộn → timeout.

| key | đang dùng | tác dụng |
|---|---|---|
| `phase1_sec` | `3600` | mốc chia TP sớm / TP muộn |
| `tp_net_pct` | `0.05` | t < `phase1_sec` và lãi ròng ≥ 5 % ⇒ bán (`TP_PHASE1`) |
| `late_tp_net_pct` | `0.03` | `phase1_sec` ≤ t < `timeout_sec` và lãi ròng ≥ 3 % ⇒ bán (`TP_LATE`). Phải < `tp_net_pct` |
| `require_stop_loss` | `false` | `false` = nhánh SL không bao giờ bán, lỗ ôm (`HOLD_LOSS`) |
| `sl_pct` | `-0.20` | chỉ có tác dụng khi `require_stop_loss = true`. Phải < 0 để load |
| `timeout_sec` | `86400` | đủ 24 h ⇒ `STOPPED_HOLD`: dừng săn, **không bán**, giữ `position.json` |
| `arm_trail_pct` | `0.05` | từng chạm +5 % thì bật sàn hòa vốn |
| `breakeven_net_pct` | `0.00` | đã bật sàn mà tụt về ≤ 0 % ⇒ bán (`trail_breakeven`) |
| `lp_drop_kill` | `0.40` | LP pool tụt 40 % so với lúc mua ⇒ bán gấp (kill) |
| `kill_unread_while_hold` | `false` | `false` = RPC đọc không được tax/LP/gas bao nhiêu nhịp cũng chỉ ôm (log `hold.unread`), không bán. `true` = 2 nhịp unread liên tiếp ⇒ bán (hành vi cũ). Thiếu key ⇒ `true` |

Lãi ròng = sim bán hết − gas ước lượng − `entry_weth`, nên vừa mua xong luôn thấy `HOLD_LOSS` âm nhẹ (phí + gas). Đó là bình thường.

Kèo chủ (mặc định file): 60 phút đầu +5 % out; sau đó +3 % out; lỗ không bán; RPC đọc lỗi cũng không bán (`kill_unread_while_hold = false`); 24 h ngừng săn, giữ vị thế. Muốn bot tự tắt đúng 24 h: chạy với `--seconds=86400`.

Vẫn còn 3 kiểu bán khẩn không tắt được bằng config: LP pool tụt ≥ `lp_drop_kill` (rug), sim bán revert, tín hiệu dump lớn (`hard_dump`). Đây là bảo vệ khỏi rug, không phải cắt lỗ theo giá.

Kèo thử nghiệm (tùy chọn, **không** đổi mặc định file):

- **Siết TP**: `phase1_sec = 1800`, `tp_net_pct = 0.03`. Khi đó `late_tp_net_pct` phải nhỏ hơn 0.03 (ví dụ `0.02`), không thì không load.
- **Bật SL −20 %**: `require_stop_loss = true`. Đổi hành vi: lỗ ròng ≤ −20 % là bán và phiên STOPPED (`net_pnl ≤ 0`). Không còn "lỗ ôm".
- Nới TP cao hơn mặc định: không khuyến khích, trái quyết định chủ 2026-09-07.

### E. Cửa lọc token

12 cửa, chạy theo thứ tự `FILTER_GATES` trong `src/filter/types.ts`: `size`, `age`, `proxy_or_upgrade`, `trading`, `owner_hostile`, `limits`, `lp_usd`, `sim`, `tax`, `impact`, `limits_size`, `fee_tier`. Rớt cửa nào, log `FILTER_FAIL:<tên cửa>` hoặc `FILTER_FAIL:<reason>` (ví dụ `tax_unread`, `pair_no_weth`).

| key | đang dùng | cửa |
|---|---|---|
| `max_buy_tax` / `max_sell_tax` | `0.005` | `tax`: thuế 2 chiều ≤ 0,5 %. Không đo được (`tax_unread`) = bỏ |
| `max_buy_impact` | `0.03` | `impact`: trượt giá mua size `buy_weth_wei` < 3 % |
| `max_buy_slippage` / `max_sell_slippage` / `slippage_bps` | `0.04` / `0.06` / `400` | cap slippage lúc gửi tx thật (GĐ7). Giấy chưa dùng, giữ để load |
| `skip_proxy_token` | `true` | `proxy_or_upgrade`: EIP-1967 / EIP-1167 / có `upgradeTo` ⇒ bỏ |
| `require_trading_enabled` | `false` | `trading`: `false` = PASS luôn, log `disabled_by_config` |
| `require_owner_check` | `false` | `owner_hostile`: `false` = PASS luôn, kill owner lúc ôm cũng tắt |
| `min_lp_usd` | `10000` | `lp_usd` (xem mục B) |
| `denylist` | 40 address | bỏ token ngay lúc quét, trước mọi cửa (log `FILTER_FAIL:denylisted`). File ship đã chặn ICP, ZEN, VIRTUAL, cbBTC, USDT, AERO, HYPE, SOL, LINK, AAVE… (2026-09-07) |

Kèo hiện tại: tax 0,5 %, impact 3 %, skip proxy ON, trading/owner check OFF. Không nới tax/impact.

Thêm một token vào danh sách chặn (ví dụ ZEN, token lớn không phải meme). Thấy bot mua giấy con nào không muốn thì lấy address từ dòng `BUY_PAPER`, kiểm symbol bằng `scripts/probe_symbol.ts`, rồi thêm một dòng:

```toml
denylist = [
  "0xf43eB8De897Fbc7F2502483B2Bef7Bb9EA179229",   # ZEN — lấy address từ dòng BUY_PAPER / SCAN trong log, symbol kiểm bằng scripts/probe_symbol.ts
]
```

Phải là address 40 hex có `0x`, không phân biệt hoa thường. Ghi sai kiểu là bot không load.

### F. Gas / WS / log

| key | đang dùng | tác dụng |
|---|---|---|
| `gas_reserve_eth_wei` | `"400000000000000"` | ETH gas phải giữ lại trong ví (live). Phải > 0 |
| `buy_max_gas_eth_wei` / `flatten_max_gas_eth_wei` | `"400000000000000"` | trần gas 1 lệnh mua / 1 lệnh flatten (live) |
| `tx_timeout_sec` | `30` | chờ tx live |
| `ws_silence_sec` | `20` | WSS im N giây ⇒ reconnect. Không HALT trừ `halt_on_ws_dead = true` |
| `log_dir` | `"./logs"` | thư mục log, file `bot.jsonl`, xoay theo `log_max_mb` / `log_keep_files` |
| `hold_log_sec` / `dup_log_sec` | `15` / `30` | bóp log lặp cùng `(event, trace_id)` |
| `log_level` | `"debug"` | **code hiện không đọc key này.** Log luôn ghi đủ; muốn bớt thì grep theo `key` |

Khi 0 lệnh: đọc `SCAN_SWEEP` (có `pools_active` > 0 chưa?), rồi `grep -o '"key":"FILTER_FAIL:[a-z_]*"' | sort | uniq -c` xem rớt cửa nào nhiều nhất.

## 3 profile copy-dùng

Chỉ ghi key **đổi**, các key khác giữ nguyên file.

**1) GIẤY — soi kèo (mặc định an toàn, đúng file hiện tại)**

```toml
dry_run = true
allow_live = false
bot_armed = false
watch_living = true
min_lp_usd = 10000
min_pool_age_sec = 0
max_trades_per_session = 5
cooldown_sec = 30
require_stop_loss = false
kill_unread_while_hold = false
# denylist: giữ 40 dòng sẵn trong file (token lớn), thêm dòng khi thấy bot mua con không phải meme
```

**2) GIẤY — siết LP $50k, 1 lệnh/phiên**

```toml
dry_run = true
allow_live = false
bot_armed = false
min_lp_usd = 50000
max_trades_per_session = 1
cooldown_sec = 30
# denylist: giữ 40 dòng sẵn trong file, không để [] (nếu không lệnh duy nhất sẽ rơi vào ICP/ZEN/cbBTC)
```

**3) LIVE — chỉ khung, không khuyến khích**

```toml
# ĐỪNG điền cho tới khi đủ 8 mục dưới. Claude Code không được điền hộ.
dry_run = false
allow_live = true
bot_armed = true
live_aero_slip = false     # bật ĐÚNG MỘT venue đã pin calldata
live_uni_v3 = false
live_pancake_v3 = false
gas_reserve_eth_wei = "…"  # > 0
buy_max_gas_eth_wei = "…"  # > 0
flatten_max_gas_eth_wei = "…"
```

Checklist 8 mục (mục 4 + mục 9 CLAUDE.md):

1. Văn bản chủ ĐẠT GĐ6 và dòng mở GĐ7 trong `docs/STATE.md`.
2. GĐ7.1 signer + `live_gate_ok`, GĐ7.2 calldata venue đó pin từ tx thật, GĐ7.3 executor — cả ba ĐẠT.
3. `eth_chainId = 0x2105`, `vps.json` load OK, WSS ra `rpc.block`.
4. Không có `state/halt.lock`.
5. 3 trần gas > 0, ETH gas riêng, WETH ≥ `buy_weth_wei`.
6. Chỉ một `live_* = true`.
7. Phiên giấy ≥ 2 h trước đó: 0 `eth_sendRawTransaction`, ≥ 10 candidate log, flatten thử OK.
8. `PRIVATE_TX_URL` rỗng ⇒ tx đi public mempool, chấp nhận rủi ro sandwich.

## Một phiên mẫu

1. Sửa `config.toml` theo profile 1 (hoặc chỉ thêm `denylist`). Đảm bảo `dry_run = true`.
2. Xóa vị thế giấy cũ nếu có: bot tự làm lúc khởi động (`position.paper_cleared`), không cần tay.
3. Chạy `npx tsx src/index.ts --seconds=7200`.
4. Thấy `chain ok: eth_chainId=0x2105` và `bot.start ghi xong`. Nếu in `MISSING: CHAINSTACK_HTTP` thì `.env` chưa điền.
5. Mở cửa sổ thứ hai: `tail -f logs/bot.jsonl` (hoặc `Get-Content logs\bot.jsonl -Wait -Tail 20`).
6. Sau ~1–2 phút phải có `SCAN_SWEEP` với `pools_active` > 0, rồi các dòng `SCAN` (ứng viên có `lp_weth_wei`, `swaps_in_window`).
7. Với mỗi ứng viên: `FILTER_FAIL:<cửa>` hoặc `BUY_PAPER`. Mở `state/position.json` để xem token, pool, `entry_weth = 2800000000000000`.
8. Đang ôm: mỗi nhịp có `exit.decided` với `key` là `HOLD_LOSS` (lỗ, ôm) hoặc `HOLD` (lãi chưa đủ). Bot không quét thêm khi đang ôm.
9. Lãi ròng ≥ +5 % trong 60 phút đầu ⇒ `TP_PHASE1` rồi `position.close`, `pnl.closed`; sau 60 phút ≥ +3 % ⇒ `TP_LATE`. Lãi ⇒ SCANNING tiếp sau `cooldown_sec`. Lỗ ⇒ STOPPED.
10. Đủ 24 h chưa bán ⇒ `STOPPED_HOLD`, bot dừng săn nhưng giữ `position.json`.
11. Muốn thoát tay: tạo `state/flatten.req` (bán giấy nếu sim bán > 0) hoặc Ctrl-C.
12. Sau khi dừng: `grep -c eth_sendRawTransaction logs/bot.jsonl` phải là 0.

## Lỗi hay gặp khi chỉnh kèo

| Triệu chứng | Key liên quan | Lệnh check |
|---|---|---|
| 0 `BUY_PAPER` cả giờ | `watch_living`, `min_lp_usd`, `min_pool_age_sec` | `grep -c '"key":"SCAN"' logs/bot.jsonl`; = 0 ⇒ xem `SCAN_SWEEP` có `pools_active`; > 0 ⇒ `grep -o '"key":"FILTER_FAIL:[a-z_]*"' logs/bot.jsonl \| sort \| uniq -c` |
| Ứng viên toàn pool rác | `min_lp_usd` (phía WETH), `denylist` | `grep '"key":"SCAN"' logs/bot.jsonl \| grep -o '"lp_weth_wei":"[0-9]*"'` — LP WETH phải cỡ ≥ 2 WETH (≈ $10k) |
| Nghĩ `bot_armed = false` chặn mua giấy | `bot_armed` | Không chặn. `grep '"key":"BUY_PAPER"' logs/bot.jsonl` vẫn ra dòng khi `bot_armed = false` |
| Tưởng `live_* = false` tắt quét | `live_*` | Không tắt. `grep '"event":"scan.sweep"' logs/bot.jsonl \| tail -1` vẫn có `verified_now` |
| LP "đúng $10k trên DexScreener" mà vẫn rớt `lp_usd` | `min_lp_usd` | DexScreener cộng cả 2 vế; bot chỉ tính vế WETH. Xem `lp_weth_wei` trong dòng `SCAN` |
| `FILTER_FAIL:tax` / `impact` hàng loạt | `max_buy_tax`, `max_sell_tax`, `max_buy_impact` | `grep -E '"key":"FILTER_FAIL:(tax|impact)' logs/bot.jsonl \| tail -3` rồi đọc field `detail`. `tax_unread` = node không sim được, không phải token xấu. Không nới ngưỡng |
| `age` rớt hết | `min_pool_age_sec` | phải = 0 cho Mode 3. `grep '"key":"FILTER_FAIL:age"' logs/bot.jsonl` |
| WSS chết / reconnect liên tục | `ws_silence_sec`, `halt_on_ws_dead` | Nhìn console: dòng `ws_silence: action=reconnect attempt=N`. Trong log: `grep -c '"event":"rpc.block"' logs/bot.jsonl` phải tăng đều (~1 dòng / 2 s). `CHAINSTACK_WS` trong `.env` phải là `wss://` |
| Bot không lên, in danh sách lỗi config | field thiếu / sai kiểu | đọc dòng lỗi; số wei phải là chuỗi `"…"`; `max_open_positions` phải 1; `tp_net_pct` > `late_tp_net_pct` |

## Không làm

- Không đổi `buy_weth_wei` trong code hay hướng dẫn. Số của chủ, chỉ chủ đổi trong `config.toml`.
- Không bật lại Mode 1/2 (quét pair mới), không bật `scan_aero_classic`.
- Không gửi tx khi `dry_run = true`. Không lật `dry_run` / `allow_live` / `bot_armed` trong chat.
- Không nới `max_buy_tax` / `max_sell_tax` / `max_buy_impact` để "cho có lệnh".
- Không in `.env`, URL Chainstack, `PRIVATE_KEY` ra chat/log/commit.

## Nguồn của README này (2026-09-07)

Lệnh đã **chạy thử** trên máy này khi viết:

- `npm run typecheck` (0 lỗi), `npm test` (32 file, 1673 test pass).
- `npx tsx src/index.ts --seconds=45` — lên `chain ok`, `SCAN_SWEEP`, `SCAN`, `BUY_PAPER`, `HOLD_LOSS`, dừng sạch, 0 `eth_sendRawTransaction`.
- Các lệnh `grep` / `Select-String` / `Get-Content -Tail` trên `logs/bot.jsonl`.
- `npx tsx scripts/probe_universe.ts 4` (phiên trước, BAOCAO46).

Lệnh chỉ **đọc từ source**, chưa chạy trong phiên viết README: `--inject=`, `--paper-fill=`, các file `state/*.req` / `halt.lock`, `scripts/probe_symbol.ts`, `scripts/probe_active.ts`, `scripts/probe_swaps.ts`. Không có flag `--config`, `--scan-only`, `--check-rpc`, `--balance`, `log_level` không được code đọc — đã ghi rõ ở từng mục.
