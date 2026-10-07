# Hướng dẫn khung "Cấu hình" trên dashboard — giải thích từng ô

Tài liệu này giải thích **từng ô** trong khung **Cấu hình** của dashboard, theo **đúng thứ tự và đúng tên** hiện trên màn hình. Mọi điều viết ở đây đã được đối chiếu với code của bot (`app/strategy.py`, `app/worker.py`, `app/engine.py`, `app/live.py`, `app/config.py`).

Quy ước trong tài liệu:
- **LONG = ĐÁNH TĂNG** (lệnh `Buy`, lời khi giá **lên**).
- **SHORT = ĐÁNH GIẢM** (lệnh `Sell`, lời khi giá **xuống**).
- Ví dụ tiền dùng tài khoản **100 USDT** cho dễ tính.

## Cách dùng khung Cấu hình
1. Sửa ô → bấm **"💾 Lưu cấu hình"**. Chưa bấm Lưu thì chưa có tác dụng.
2. Lần quét kế tiếp (tối đa khoảng 60 giây) sẽ dùng cấu hình mới.
3. Một bộ cấu hình dùng chung cho **PAPER, DEMO và LIVE**.
4. Vị thế **đang mở** không bị đảo chiều, không đổi đòn bẩy. Riêng chốt lời / cắt lỗ:
   - PAPER: số `tp_pct` / `max_loss_pct` mới áp dụng **ngay** cho cả vị thế đang mở.
   - DEMO/LIVE: TP/SL **đã đặt trên sàn** của vị thế đang mở giữ nguyên; lệnh mới mới dùng số mới. (Riêng bước "đóng dự phòng" của bot dùng số mới: nếu bạn hạ `tp_pct` xuống dưới mức lời hiện tại, bot sẽ tự đóng vị thế sau khoảng 15 giây.)

## Từ ngữ cần biết (mỗi từ 1 câu)
- **Ký quỹ (margin)**: số tiền bạn bỏ ra cho một lệnh.
- **Đòn bẩy (leverage) x10**: lệnh lớn gấp 10 lần ký quỹ → giá đi 1% thì lời/lỗ 10% ký quỹ.
- **Notional**: giá trị thật của lệnh = ký quỹ × đòn bẩy.
- **Thanh lý (liquidation)**: sàn tự đóng lệnh khi bạn lỗ gần hết ký quỹ; với x10 xảy ra khi giá đi ngược khoảng 9–10%.
- **EMA20 / EMA50**: đường giá trung bình của 20 / 50 cây nến gần nhất; giá nằm trên = đang có xu hướng tăng, nằm dưới = đang giảm.
- **Nến 15m / 1h**: mỗi cây nến là 15 phút / 1 giờ giá. Bot chỉ dùng nến **đã đóng**.
- **RSI**: chỉ số 0–100 đo giá đang "nóng" (cao) hay "nguội" (thấp).
- **Funding**: phí mà phe LONG và phe SHORT trả cho nhau mỗi vài giờ.
- **Spread**: chênh lệch giữa giá mua và giá bán tốt nhất; spread lớn = vào lệnh là lỗ ngay một chút.
- **Turnover 24h**: tổng tiền giao dịch của đồng coin trong 24 giờ; thấp = ít người mua bán.
- **TP (take profit)** = chốt lời; **SL (stop loss)** = cắt lỗ.

---

## 1. Bảng tóm tắt các ô tick

| Ô tick (tên trên form) | Bình thường (không đánh ngược) | Khi bật ĐÁNH NGƯỢC | Lý do ngắn |
|---|---|---|---|
| "ĐÁNH NGƯỢC tín hiệu…" | **Không nên tick** | — | Chỉ dùng khi bạn cố ý đánh ngược và đã chạy DEMO |
| "Bắt buộc EMA20 (15m)…" | **Nên tick** | **Tuỳ** | Giữ lệnh đúng chiều xu hướng ngắn; khi đánh ngược thì thành "đánh ngược xu hướng ngắn" |
| "Bắt buộc EMA50 (1h)…" | **Tuỳ** | **Không nên tick** | Bình thường: ít lệnh hơn nhưng thuận xu hướng lớn. Đánh ngược: bot đi ngược cả xu hướng 1 giờ |
| "Bắt buộc tín hiệu: …RSI… / phá đáy-đỉnh" | **Tuỳ** (người mới: không tick) | **Tuỳ** | Rất ít lệnh; cú phá đỉnh/đáy mạnh hay bị "Chống đuổi giá" chặn |
| "Bắt buộc volume nến tín hiệu > hệ số × TB20" | **Không nên tick** | **Không nên tick** | Lọc thêm, ít tác dụng với nến 15m, làm giảm số lệnh |
| "Chặn vào lệnh khi funding bất lợi…" | **Nên tick** | **Không nên tick** | Khi đánh ngược, ô này bảo vệ **nhầm phía** (xem mục 4) |
| "Bật chống đuổi giá…" | **Nên tick** | **Nên tick** | Tránh vào lệnh sau cú chạy quá mạnh |
| "Chỉ vào lệnh khi setup còn MỚI…" | **Nên tick** | **Nên tick** | Tránh vào mã đã chạy một chiều nhiều giờ |
| "Giả lập phí taker" | **Nên tick** | **Nên tick** | Chỉ ảnh hưởng PAPER, giúp kết quả giả lập sát thực tế |

---

## 2. Giải thích từng ô (theo thứ tự trên form)

### 2.1. Nhóm "Tài khoản & quản lý vốn"

**"Vốn paper ban đầu (USDT) — đổi sẽ hỏi reset tài khoản"** (`initial_capital`, mặc định **100**)
- Chỉ dùng cho **PAPER** (tiền giả lập). DEMO/LIVE dùng số dư thật trên sàn.
- Đổi số này, bot hỏi có reset tài khoản paper không (reset = xoá hết lịch sử paper).
- Khuyên: đặt gần bằng số tiền bạn định dùng thật, để kết quả giả lập dễ so sánh.

**"Đòn bẩy (isolated, tối đa 10x)"** (`leverage`, mặc định **5**)
- Mỗi lệnh dùng ký quỹ riêng (isolated): lệnh này thua không ăn vào tiền của lệnh khác.
- Ví dụ x5: giá đi 1% → lời/lỗ 5% ký quỹ. x10: giá đi 1% → 10% ký quỹ.
- Bot không cho đặt quá **10**.
- Khuyên: người mới **3–5**. Ở x10 với "Ngưỡng lỗ tối đa" 99%, giá đi ngược khoảng 8,5–9% là chạm cắt lỗ (ngay trước giá thanh lý).

**"Size % = KÝ QUỸ mỗi lệnh theo % equity"** (`size_pct`, mặc định **2**)
- Số này là **ký quỹ** (không phải giá trị lệnh). Ví dụ 100 USDT, size 2%, x5 → ký quỹ 2 USDT, lệnh trị giá 10 USDT.
- PAPER tính trên equity giả lập; DEMO/LIVE tính trên **số dư khả dụng thật**, rồi bot tự trừ bớt để đủ trả **phí mở + phí đóng** và chừa **1%** dự phòng. Ví dụ size 99%, x10 → ký quỹ thực tế ≈ **97,9%** số dư.
- Khuyên: **2–10%**. Đặt 99% nghĩa là **dồn cả tài khoản vào 1 lệnh**.

**"Số vị thế tối đa"** (`max_positions`, mặc định **5**)
- Số lệnh mở cùng lúc tối đa, LONG và SHORT tính chung.
- Khuyên: nếu size nhỏ (2–5%) thì 3–5. Nếu size lớn (≥ 50%) thì để **1** (xem mâu thuẫn ở mục 4).

**"Chốt lời: PnL chưa chốt ≥ % ký quỹ"** (`tp_pct`, mặc định **10**)
- Tính theo **% ký quỹ**, không phải % giá. Giá cần chạy = `tp_pct` ÷ đòn bẩy.
- Ví dụ: TP 8% ở x10 → giá chỉ cần chạy **0,8%** đúng chiều. TP 10% ở x5 → giá chạy 2%.
- Phí mở + đóng ở x10 ≈ **1,1% ký quỹ**, nên TP 8% gộp ≈ **6,9%** ròng.
- Khuyên: ít nhất gấp vài lần phí (≥ 5% ký quỹ ở x10).

**"Ngưỡng lỗ tối đa % ký quỹ (đóng + DỪNG bot)"** (`max_loss_pct`, mặc định **99**)
- Khi một lệnh lỗ tới mức này → bot đóng lệnh và **chuyển sang `STOPPED_MAX_LOSS`**: không mở lệnh mới cho tới khi bạn bấm "⟲ Reset trạng thái dừng" rồi "▶ Start".
- DEMO/LIVE: đây là **SL đặt sẵn trên sàn**. Nhưng sàn thanh lý trước khi lỗ tới 99%, nên bot **kéo SL về ngay trước giá thanh lý** (xem ô "SL đặt cách giá thanh lý…"). Ví dụ x10: đặt 99% hay 90% thì SL thật đều ở khoảng **-85% đến -90% ký quỹ** (giá đi ngược ~8,5–9%). Ở DEMO/LIVE, lệnh đóng do SL hoặc bị thanh lý đều làm bot dừng.
- PAPER không giả lập thanh lý: lệnh đóng đúng ở -99%.
- Khuyên: 40–70% ở x5. Số càng thấp thì cắt lỗ càng sớm (nhưng mỗi lần cắt lỗ bot đều dừng).

**"Cooldown sau khi đóng (giờ)"** (`cooldown_hours`, mặc định **4**)
- Sau khi đóng một lệnh ở mã nào (lời hay lỗ, LONG hay SHORT), bot **cấm** vào lại mã đó (cả hai chiều) trong số giờ này.
- Khuyên: 2–6 giờ. Đặt 0 thì bot có thể vào lại ngay mã vừa chốt.

### 2.2. Nhóm "Bộ lọc universe" (lọc danh sách coin)

**"Turnover 24h tối thiểu (USDT)"** (`min_turnover_24h`, mặc định **2.000.000**)
- Bỏ qua coin có tổng giao dịch 24h nhỏ hơn số này.
- Hạ thấp (ví dụ 1.000.000) → nhiều coin nhỏ, ít người mua bán, giá dễ giật mạnh (râu nến dài) → dễ chạm SL / thanh lý hơn.
- Khuyên: **2.000.000–10.000.000**.

**"Spread tối đa % ((ask-bid)/mid)"** (`max_spread_pct`, mặc định **0,3**)
- Bỏ qua coin có chênh lệch mua/bán lớn hơn số này.
- Vào lệnh market là mất khoảng nửa spread. Ví dụ spread 0,3% ở x10 → mất ngay ≈ **1,5% ký quỹ** — gần 1/5 của TP 8%.
- Khuyên: **0,1–0,2** nếu dùng đòn bẩy cao và TP nhỏ.

**"Tuổi niêm yết tối thiểu (giờ)"** (`min_listing_hours`, mặc định **24**)
- Bỏ qua coin mới lên sàn chưa đủ số giờ này (coin mới thường biến động điên cuồng).
- Khuyên: giữ **24** hoặc cao hơn.

**"Số symbol lấy nến mỗi lần quét (top-K)"** (`top_k`, mặc định **60**)
- Bot chỉ xét tín hiệu cho K coin có thanh khoản + biến động 24h cao nhất (sau khi lọc).
- Tăng → tìm được nhiều cơ hội hơn nhưng tốn nhiều request tới Bybit hơn.
- Khuyên: **40–100**.

### 2.3. Nhóm "Hướng giao dịch"

**"Hướng giao dịch (lọc theo hướng TÍN HIỆU)"** (`trade_direction`, mặc định **Cả hai (Long + Short)**)
- **Cả hai (Long + Short)**: nhận cả tín hiệu ĐÁNH TĂNG lẫn ĐÁNH GIẢM. Nếu một coin có cả hai thì bot chọn phía có điểm cao hơn.
- **Chỉ short**: chỉ nhận tín hiệu SHORT. **Chỉ long**: chỉ nhận tín hiệu LONG.
- Chú ý chữ **TÍN HIỆU**: ô này lọc **tín hiệu**, không lọc chiều lệnh. Khi bật ĐÁNH NGƯỢC, "Chỉ short" = chỉ vào lệnh **LONG** (xem mục 4).
- Khuyên: **Cả hai**.

**"ĐÁNH NGƯỢC tín hiệu (tín hiệu SHORT → vào LONG, tín hiệu LONG → vào SHORT)…"** (`invert_signals`, mặc định **không tick**)
- Bot vẫn tìm tín hiệu y như cũ (mọi bộ lọc, xếp hạng, cooldown tính theo **chiều tín hiệu**), chỉ **đảo chiều lệnh** lúc vào.
- Ví dụ: coin X vừa cắt xuống dưới EMA20 → tín hiệu SHORT → bot vào **LONG (ĐÁNH TĂNG)**. Tức là bot **cược tín hiệu sai**.
- TP/SL, thanh lý, lời/lỗ tính theo chiều lệnh thật.
- Khi bật: dashboard hiện dải vàng "⚠ Đang ĐÁNH NGƯỢC tín hiệu…". Trong bảng "Ứng viên", cột "Chiều lệnh" vẫn hiện **chiều tín hiệu**; cột kết quả ghi `OPENED LONG (ĐÁNH NGƯỢC)` = chiều lệnh thật.
- Lệnh đang mở **không** bị đảo khi bạn bật/tắt ô này.
- Khuyên: không tick, trừ khi bạn cố ý chơi kiểu đánh ngược và đã thử ở DEMO.

### 2.4. Nhóm "Điều kiện vào lệnh (bật/tắt — áp dụng đối xứng cho LONG và SHORT)"
Chỉ những ô **được tick** mới chặn lệnh. Ô không tick vẫn được tính và hiện mờ trong bảng ứng viên để bạn tham khảo.

**"Bắt buộc EMA20 (15m): short giá < EMA20, long giá > EMA20"** (`require_ema20_15m`, mặc định **tick**)
- Tín hiệu SHORT chỉ khi giá đang **dưới** EMA20 nến 15 phút; tín hiệu LONG chỉ khi giá **trên** EMA20.
- Đây là bộ lọc xu hướng ngắn hạn cơ bản nhất.
- Khuyên: **tick**.

**"Bắt buộc EMA50 (1h): short giá < EMA50, long giá > EMA50"** (`require_ema50_1h`, mặc định **không tick**)
- Thêm điều kiện xu hướng lớn hơn (nến 1 giờ). Tick cùng EMA20 = chỉ vào lệnh khi cả xu hướng 15 phút và 1 giờ cùng chiều.
- Ít lệnh hơn hẳn.
- Khuyên: tuỳ. **Không tick khi đánh ngược.**

**"Bắt buộc tín hiệu: short = RSI cắt xuống / phá đáy 20 nến; long = RSI cắt lên / phá đỉnh 20 nến"** (`require_signal`, mặc định **không tick**)
- SHORT cần **một trong hai**: (a) RSI vừa từ trên 60 rơi xuống dưới 55, hoặc (b) giá đóng nến dưới đáy thấp nhất của 20 nến trước ("phá đáy").
- LONG cần: (a) RSI vừa từ dưới 40 lên trên 45, hoặc (b) giá đóng nến trên đỉnh cao nhất 20 nến trước ("phá đỉnh").
- Rất ít lệnh. Cú phá đáy/đỉnh **mạnh** thường bị "Chống đuổi giá" chặn (mục 4).
- Khuyên: người mới **không tick**.

Bốn ô số RSI đi kèm (chỉ có tác dụng khi tick ô trên):
- **"SHORT: RSI phải từng > (trong lookback)"** (`rsi_upper`, mặc định **60**)
- **"SHORT: rồi RSI cắt xuống dưới"** (`rsi_lower`, mặc định **55**)
- **"LONG: RSI phải từng < (trong lookback)"** (`rsi_long_lower`, mặc định **40**)
- **"LONG: rồi RSI cắt lên trên"** (`rsi_long_upper`, mặc định **45**)
- **"RSI lookback (số nến 15m)"** (`rsi_lookback`, mặc định **6**): "từng >60 / <40" xét trong 6 nến gần nhất (1,5 giờ).
- Khuyên: giữ mặc định.

**"Bắt buộc volume nến tín hiệu > hệ số × TB20"** (`require_volume`, mặc định **không tick**)
- Chỉ vào lệnh khi nến 15m vừa đóng có khối lượng lớn hơn "hệ số × trung bình 20 nến trước".
- **"Hệ số volume (0 = tắt điều kiện volume)"** (`volume_mult`, mặc định **1,0**): ví dụ 1,5 = volume phải gấp 1,5 lần trung bình. Đặt **0** thì ô tick trên **mất tác dụng**.
- Khuyên: không tick.

**"Chặn vào lệnh khi funding bất lợi (an toàn, nên bật)"** (`require_funding`, mặc định **tick**)
- Không nhận tín hiệu SHORT khi funding thấp hơn **"Không SHORT nếu funding < (%)"** (`min_funding_rate_pct`, mặc định **-0,05%**).
- Không nhận tín hiệu LONG khi funding cao hơn **"Không LONG nếu funding > (%)"** (`max_funding_rate_pct`, mặc định **+0,05%**).
- Lý do: funding âm sâu = quá nhiều người SHORT, dễ bị giá giật lên; funding dương cao = quá nhiều người LONG, dễ bị xả.
- Ví dụ phí: funding 0,1% mỗi kỳ ở x10 = mất **1% ký quỹ** mỗi kỳ (thường 8 giờ một kỳ).
- Khuyên: **tick** khi không đánh ngược. **Không tick khi đánh ngược** (bảo vệ nhầm phía, mục 4).

### 2.5. Nhóm "Chống đuổi giá (không vào lệnh khi giá đã chạy quá xa)"

**"Bật chống đuổi giá: không LONG khi giá vừa tăng quá mạnh, không SHORT khi giá vừa giảm quá mạnh (nên bật)"** (`require_no_chase`, mặc định **tick**)
- Bỏ qua một phía nếu giá **đã chạy quá xa theo chiều đó**. Chỉ cần vướng **1** trong 4 kiểm tra dưới đây là bị chặn. Đặt một ngưỡng = 0 là tắt riêng kiểm tra đó; đặt cả 4 = 0 thì ô tick này **mất tác dụng**.
- **"Số nến 15m đã đóng để đo mức chạy gần đây (4 nến = 1 giờ)"** (`chase_lookback_bars`, mặc định **4**).
- **"Mức chạy tối đa trong khoảng trên (%)…"** (`max_move_pct_recent`, mặc định **3**): ví dụ coin tăng 4% trong 1 giờ → **không LONG**; giảm 4% → **không SHORT**.
- **"Giá cách EMA20 (15m) tối đa (%)…"** (`max_ema20_distance_pct`, mặc định **2**): giá cao hơn EMA20 quá 2% → không LONG; thấp hơn quá 2% → không SHORT.
- **"Biến động 24h tối đa (%)…"** (`max_change_24h_pct`, mặc định **25**): coin +30% trong 24h → không LONG; -30% → không SHORT.
- **"Nến 15m vừa đóng to bất thường…"** (`max_candle_body_mult`, mặc định **3**): nến xanh có thân > 3 lần trung bình 20 nến trước → không LONG; nến đỏ to như vậy → không SHORT.
- Khuyên: **tick**, giữ mặc định.

### 2.6. Nhóm "Ưu tiên tín hiệu MỚI & tránh mã vừa giao dịch"

**"Chỉ vào lệnh khi setup còn MỚI (vừa cắt EMA20 / vừa phá đáy-đỉnh)"** (`require_fresh_signal`, mặc định **tick**)
- "Tuổi setup" = số nến 15m kể từ khi **giá vừa cắt EMA20** sang phía đó **hoặc** **vừa phá đáy/đỉnh 20 nến** (lấy cái gần nhất).
- Chỉ vào khi tuổi ≤ **"Tuổi setup tối đa (số nến 15m đã đóng, 0 = nến vừa đóng)"** (`max_signal_age_bars`, mặc định **3** = trong khoảng 45 phút–1 giờ gần nhất).
- Ví dụ: coin nằm dưới EMA20 suốt 5 giờ mà không phá đáy mới → **không** được SHORT nữa.
- Khuyên: **tick**, tuổi **2–4**.

Các ô xếp hạng (quyết định coin nào được vào trước khi có nhiều coin đạt):
- **"Trọng số xếp hạng: độ mới"** (`rank_w_fresh`, mặc định **60**) — setup càng mới điểm càng cao.
- **"Trọng số xếp hạng: signal score"** (`rank_w_signal`, mặc định **25**) — tín hiệu càng "đẹp" điểm càng cao.
- **"Trọng số xếp hạng: turnover + |%24h|"** (`rank_w_market`, mặc định **15**) — coin càng sôi động điểm càng cao.
- **"Phạt mã đã giao dịch trong vòng (giờ, 0 = tắt)"** (`recent_trade_penalty_hours`, mặc định **24**).
- **"Điểm phạt mã vừa giao dịch (lớn = chỉ chọn khi không còn mã khác)"** (`recent_trade_penalty`, mặc định **1000**).
- Khác với cooldown: cooldown **cấm hẳn** trong 4 giờ; sau đó, tới hết 24 giờ, coin vẫn được vào nhưng xếp **cuối**.
- Khuyên: giữ mặc định.

### 2.7. Nhóm "Phí giả lập"

**"Giả lập phí taker"** (`simulate_fees`, mặc định **tick**)
- Chỉ ảnh hưởng **PAPER**: trừ phí như sàn thật. Bỏ tick thì kết quả paper đẹp hơn thực tế.
- **Không** ảnh hưởng DEMO/LIVE.
- Khuyên: **tick**.

**"Phí taker % mỗi chiều (DEMO/LIVE cũng dùng để chừa chỗ cho phí khi tính ký quỹ)"** (`taker_fee_pct`, mặc định **0,055**)
- PAPER: mức phí giả lập mỗi lần mở/đóng. DEMO/LIVE: bot dùng số này để **chừa tiền trả phí** khi tính ký quỹ.
- Ví dụ x10: phí mở + đóng ≈ 2 × 0,055% × 10 = **1,1% ký quỹ**.
- Khuyên: đặt **đúng mức phí taker thật** của tài khoản bạn (0,055% là mức tiêu chuẩn). **Không đặt thấp hơn** (xem mục 4).

### 2.8. Nhóm "Tiền thật (DEMO / LIVE) — giới hạn TUỲ CHỌN, 0 = TẮT (không giới hạn)"
Các ô này chỉ dùng cho DEMO/LIVE. **0 = không giới hạn gì cả.**

**"Ký quỹ tối đa mỗi lệnh (USDT, 0 = không giới hạn)"** (`live_max_margin_usdt`, mặc định **0**)
- Ví dụ size 99% nhưng đặt 20 → mỗi lệnh chỉ dùng tối đa 20 USDT ký quỹ dù tài khoản có 500 USDT.
- Khuyên: **nên đặt** khi chạy LIVE.

**"Tổng ký quỹ tối đa của bot (USDT, 0 = không giới hạn)"** (`live_max_total_margin_usdt`, mặc định **0**)
- Tổng ký quỹ của mọi lệnh bot đang mở không vượt số này.
- Khuyên: nên đặt nếu `max_positions` > 1.

**"Lỗ tối đa trong ngày → dừng mở lệnh (USDT, 0 = tắt)"** (`daily_loss_limit_usdt`, mặc định **0**)
- Lỗ trong ngày (đã chốt + đang gồng, theo giờ Việt Nam) chạm −số này → bot chuyển `STOPPED_DAILY_LOSS`, dừng mở lệnh tới ngày hôm sau hoặc tới khi bạn bấm reset.
- Khuyên: **nên đặt** khi chạy LIVE.

**"SL đặt cách giá thanh lý ít nhất (% giá entry)"** (`sl_liq_buffer_pct`, mặc định **0,5**, cho phép 0,05–5)
- Nếu mức cắt lỗ bạn đặt nằm sau giá thanh lý, bot kéo SL về trước giá thanh lý một khoảng bằng số % giá vào lệnh này.
- Ví dụ LONG giá vào 100, x10, giá thanh lý ≈ 90,5 → SL ≈ 91,0 (lỗ ≈ 90% ký quỹ). Đặt 1,0 → SL ≈ 91,5 (lỗ ≈ 85%), cắt sớm hơn một chút, ít rủi ro bị thanh lý khi giá giật mạnh.
- Khuyên: **0,5–1,0**.

Các tham số chỉ báo (`ema_15m_period` = 20, `ema_1h_period` = 50, `rsi_period` = 14, `breakdown_lookback` = 20, `volume_lookback` = 20) **không có trên form**; chỉ sửa được qua API. Nên để nguyên.

---

## 3. Ô nào đi cùng ô nào (kết hợp tốt)

- **"Bắt buộc EMA20 (15m)" + "setup còn MỚI" + "chống đuổi giá"** (bộ mặc định): EMA20 cho biết chiều xu hướng ngắn, "setup còn MỚI" bắt đúng lúc xu hướng vừa bắt đầu, "chống đuổi giá" bỏ những cú đã chạy quá xa. Ba ô bổ sung cho nhau: vào **sớm** nhưng **không đuổi**.
- **"setup còn MỚI" + "chống đuổi giá"**: setup mới qua đường **cắt EMA20** thường giá còn gần EMA → dễ qua chống đuổi giá. Setup mới qua **phá đỉnh/đáy bằng một cây nến rất to** thì hay bị chặn — đó chính là điều bạn muốn (tránh mua đỉnh / bán đáy).
- **"Chặn vào lệnh khi funding bất lợi"** + mọi bộ lọc khác (khi **không** đánh ngược): luôn nên đi kèm, gần như không làm mất lệnh tốt.
- **Cooldown + phạt mã vừa giao dịch**: cooldown cấm vào lại 4 giờ, phạt điểm đẩy mã đó xuống cuối tới 24 giờ → bot không "nhai đi nhai lại" một coin.
- **`size_pct` nhỏ (2–10%) + `max_positions` 3–5 + `live_max_total_margin_usdt`**: chia rủi ro ra nhiều lệnh, có trần tổng.
- **`size_pct` lớn + `max_positions` = 1 + `live_max_margin_usdt` + `daily_loss_limit_usdt`**: nếu muốn dồn lệnh, ít nhất có trần tiền mỗi lệnh và trần lỗ mỗi ngày.
- **`leverage` cao + `max_spread_pct` thấp (0,1–0,2) + `min_turnover_24h` cao**: đòn bẩy cao thì phải chọn coin thanh khoản tốt, spread nhỏ, để phí vào lệnh không ăn hết TP.
- **ĐÁNH NGƯỢC + "chống đuổi giá"**: khi đánh ngược, chống đuổi giá chặn tín hiệu SHORT sau cú giảm mạnh → tức là **không LONG ngay sau cú sập mạnh** (không "bắt dao rơi"), và ngược lại không SHORT ngay sau cú tăng vọt. Nên giữ.

---

## 4. Ô nào tick cùng lúc sẽ MÂU THUẪN
Chỉ liệt kê những trường hợp **code thật sự tạo ra**.

**4.1. Nếu tick "ĐÁNH NGƯỢC" và "Bắt buộc EMA20 (15m)" / "Bắt buộc EMA50 (1h)" thì bot vào lệnh NGƯỢC xu hướng mà chính các ô EMA vừa xác nhận**, vì bộ lọc EMA được kiểm tra trên **chiều tín hiệu**, rồi chiều lệnh mới bị đảo.
- Ví dụ: tín hiệu SHORT cần giá < EMA20 (xu hướng ngắn đang giảm) → bot vào **LONG**. Tick thêm EMA50 1h → bot chỉ LONG khi **cả 15 phút lẫn 1 giờ đều đang giảm**, và chỉ SHORT khi cả hai đều đang tăng.
- Với EMA20, đó có thể là điều bạn muốn ("cược cú cắt EMA20 vừa xảy ra là giả"). Với EMA50 1h thì bot **đi ngược cả xu hướng lớn** — nếu xu hướng tiếp diễn ~9% (x10) là chạm SL. Khi đánh ngược: **không tick EMA50**.

**4.2. Nếu tick "ĐÁNH NGƯỢC" và chọn "Chỉ short" thì bot CHỈ vào lệnh LONG (và "Chỉ long" → chỉ SHORT)**, vì "Hướng giao dịch" lọc theo **tín hiệu**; bot lấy tín hiệu SHORT rồi đảo thành LONG. Ai muốn "chỉ ĐÁNH GIẢM" mà bật đánh ngược thì phải chọn **"Chỉ long"**.

**4.3. Nếu tick "ĐÁNH NGƯỢC" và "Chặn vào lệnh khi funding bất lợi" thì bộ chặn bảo vệ NHẦM phía**, vì funding được kiểm tra trên chiều tín hiệu:
- Tín hiệu SHORT (→ lệnh LONG) chỉ bị chặn khi funding **âm sâu** — lúc đó LONG lại được **nhận** funding. Còn khi funding **dương cao** (LONG phải **trả** nhiều) thì không bị chặn.
- Tín hiệu LONG (→ lệnh SHORT) thì ngược lại.
- Kết quả: bot bỏ những lệnh được nhận funding và cho qua những lệnh phải trả. Khi đánh ngược: **không tick funding**.

**4.4. Nếu tick "Bắt buộc tín hiệu" (phá đáy/phá đỉnh) và "chống đuổi giá" thì các cú phá đỉnh/đáy MẠNH bị chặn**, vì cây nến phá đỉnh thường là nến to và giá vừa chạy nhanh. Đã thử bằng code: nến phá đỉnh +4% trong 1 giờ → điều kiện "tín hiệu" **đạt** nhưng bị chặn vì "+4,0% trong 1h, cách EMA20 +3,6%, nến 15m tăng to gấp …× TB20". Chỉ những cú phá **nhẹ nhàng** mới lọt (đường RSI vẫn có thể đạt). Đây là mâu thuẫn **một phần**; nếu muốn đánh breakout mạnh thì phải nới `max_move_pct_recent` / `max_candle_body_mult` / `max_ema20_distance_pct` — nhưng như vậy là chấp nhận đuổi giá.

**4.5. Nếu `size_pct` lớn (ví dụ 99) và `max_positions` > 1 thì thực tế vẫn chỉ có 1 lệnh lớn**, vì:
- PAPER: lệnh thứ 2 cần ký quỹ = 99% equity, nhưng tiền mặt chỉ còn ~1% → bị bỏ qua ("insufficient cash").
- DEMO/LIVE: lệnh thứ 2 tính 99% của **số dư còn lại** (~2%) → lệnh rất nhỏ (ví dụ tài khoản 1000 USDT → lệnh 2 chỉ ≈ 15 USDT ký quỹ) hoặc bị bỏ vì dưới mức tối thiểu của sàn.
- Muốn nhiều lệnh thì giảm `size_pct` (ví dụ 5 lệnh → mỗi lệnh ≤ 19%).

**4.6. Nếu `size_pct` gần 100 và `taker_fee_pct` thấp hơn phí thật (hoặc = 0) thì Bybit báo lỗi 110007 (không đủ số dư)**, vì bot chừa chỗ cho phí theo đúng số `taker_fee_pct` bạn nhập. Ví dụ 1000 USDT, size 99, x10:
- `taker_fee_pct` = 0,055 → ký quỹ 979,23 + phí ≈ 10,77 = 990 → vừa đủ (còn ~1% dự phòng).
- `taker_fee_pct` = 0 → ký quỹ 990 + phí thật ≈ 10,89 = **1000,89 > 1000** → sàn từ chối.
- Với 0,055% đúng, 110007 vẫn có thể xảy ra nếu tài khoản có lệnh/vị thế khác chiếm số dư hoặc giá nhảy mạnh lúc gửi lệnh. Khi gặp, bot tạm ngừng thử khoảng 1 phút rồi thử lại.

**4.7. Nếu đặt `max_loss_pct` cao (90–99) với đòn bẩy cao thì SL thật trên sàn KHÔNG ở mức bạn đặt**, vì sàn thanh lý trước, nên bot kéo SL về trước giá thanh lý (`sl_liq_buffer_pct`). Đã tính bằng code (LONG, giá vào 100, x10): `max_loss_pct` 99 hoặc 90 → SL ≈ **91,0–91,5** (lỗ ≈ **85–90% ký quỹ**); 80 → SL 92,0 (lỗ đúng 80%). Ở x5: 99 → SL ≈ 81,5 (lỗ ≈ 92,5%). Ở PAPER thì đóng đúng -99% (không có thanh lý) → **PAPER lỗ nặng hơn DEMO/LIVE** trong cùng tình huống.

**4.8. Nếu tick một ô nhưng ngưỡng của nó = 0 thì ô tick đó không có tác dụng**:
- "Bắt buộc volume…" + "Hệ số volume" = 0 → điều kiện volume tắt.
- "Bật chống đuổi giá…" + cả 4 ngưỡng = 0 → chống đuổi giá tắt.

### Không phải mâu thuẫn, nhưng cần biết
- **EMA20 + EMA50 + "setup còn MỚI" cùng tick** → không mâu thuẫn (ví dụ SHORT: giá vừa cắt xuống EMA20 trong lúc vẫn dưới EMA50 1h), nhưng **rất ít lệnh**; có khi cả buổi không có lệnh nào.
- **Bỏ tick TẤT CẢ điều kiện** (EMA20, EMA50, tín hiệu, volume, funding, chống đuổi giá, setup MỚI) → đã thử bằng code: **mọi coin đều "đạt" cả hai phía**; bot vào gần như bất kỳ coin nào đứng đầu bảng xếp hạng. Coi như đánh ngẫu nhiên.
- **Chỉ giữ "setup còn MỚI"** (bỏ EMA20): "mới" có thể đến từ **cú phá đỉnh/đáy** trong 3 nến dù giá **đã quay đầu** qua phía bên kia EMA20. Đã thử: cùng một coin, cả LONG lẫn SHORT đều đạt; bot chọn phía điểm cao hơn. Nhiều lệnh hơn, nhiễu hơn. Tick lại EMA20 thì trường hợp đó bị loại.
- **Các ô "Tiền thật" để 0** = **không có giới hạn nào**. Không lỗi, nhưng không có lưới an toàn.
- **`min_turnover_24h` thấp** = thêm nhiều coin nhỏ, giật mạnh. Không lỗi, nhưng rủi ro chạm SL cao hơn.
- **Cooldown**: không thấy mâu thuẫn với ô nào. Lưu ý: khi bot đang `STOPPED_MAX_LOSS` thì cooldown không còn ý nghĩa vì bot không mở lệnh mới.
- **Bỏ tick "Giả lập phí taker"** không làm DEMO/LIVE bớt phí; DEMO/LIVE luôn dùng `taker_fee_pct` để chừa phí.

---

## 5. Ba cấu hình mẫu

Ví dụ số tiền dựa trên tài khoản **200 USDT**. Đổi theo tài khoản của bạn. **Luôn chạy DEMO trước.**

### 5.1. "An toàn" (người mới, ưu tiên giữ tiền)
- [ ] ĐÁNH NGƯỢC tín hiệu — **không tick**
- Hướng giao dịch: **Cả hai (Long + Short)**
- [x] Bắt buộc EMA20 (15m)
- [x] Bắt buộc EMA50 (1h)
- [ ] Bắt buộc tín hiệu (RSI / phá đáy-đỉnh)
- [ ] Bắt buộc volume
- [x] Chặn vào lệnh khi funding bất lợi
- [x] Bật chống đuổi giá (giữ mặc định 4 nến / 3% / 2% / 25% / 3)
- [x] Chỉ vào lệnh khi setup còn MỚI — tuổi tối đa **3**
- [x] Giả lập phí taker — phí **0,055**
- Đòn bẩy **3** · Size **5%** · Số vị thế tối đa **3** · Chốt lời **10%** (giá chạy ~3,3%) · Lỗ tối đa **40%** (giá ngược ~13%, không bị kẹp vì thanh lý ở x3 xa hơn nhiều) · Cooldown **4**
- Turnover tối thiểu **10.000.000** · Spread tối đa **0,15** · Tuổi niêm yết **48** · Top-K **60**
- Ký quỹ tối đa mỗi lệnh **10** · Tổng ký quỹ tối đa **30** · Lỗ tối đa trong ngày **15** · SL cách thanh lý **1,0**
- Kết quả: ít lệnh, thuận xu hướng cả 15 phút và 1 giờ, mỗi lệnh thua mất ≈ 2% tài khoản.

### 5.2. "Cân bằng" (gần mặc định)
- [ ] ĐÁNH NGƯỢC tín hiệu — **không tick**
- Hướng giao dịch: **Cả hai (Long + Short)**
- [x] Bắt buộc EMA20 (15m)
- [ ] Bắt buộc EMA50 (1h)
- [ ] Bắt buộc tín hiệu
- [ ] Bắt buộc volume
- [x] Chặn vào lệnh khi funding bất lợi
- [x] Bật chống đuổi giá (mặc định)
- [x] Chỉ vào lệnh khi setup còn MỚI — tuổi tối đa **3**
- [x] Giả lập phí taker — phí **0,055**
- Đòn bẩy **5** · Size **10%** · Số vị thế tối đa **3** · Chốt lời **10%** (giá chạy 2%) · Lỗ tối đa **60%** (giá ngược 12%, không bị kẹp) · Cooldown **4**
- Turnover tối thiểu **3.000.000** · Spread tối đa **0,2** · Tuổi niêm yết **24** · Top-K **60**
- Ký quỹ tối đa mỗi lệnh **20** · Tổng ký quỹ tối đa **60** · Lỗ tối đa trong ngày **30** · SL cách thanh lý **0,5**
- Kết quả: số lệnh vừa phải; mỗi lệnh thua mất ≈ 6% tài khoản.

### 5.3. "Đánh ngược tín hiệu" (kiểu bạn đang chơi, nhưng bớt rủi ro)
- [x] ĐÁNH NGƯỢC tín hiệu
- Hướng giao dịch: **Cả hai (Long + Short)**
- [x] Bắt buộc EMA20 (15m) — giữ để logic nhất quán: "cược cú cắt EMA20 vừa xảy ra là giả"
- [ ] Bắt buộc EMA50 (1h) — **bỏ**, tránh đi ngược xu hướng 1 giờ (mục 4.1)
- [ ] Bắt buộc tín hiệu
- [ ] Bắt buộc volume
- [ ] Chặn vào lệnh khi funding bất lợi — **bỏ**, vì khi đánh ngược nó bảo vệ nhầm phía (mục 4.3)
- [x] Bật chống đuổi giá (mặc định) — chặn LONG ngay sau cú sập mạnh, SHORT ngay sau cú tăng vọt
- [x] Chỉ vào lệnh khi setup còn MỚI — tuổi tối đa **3**
- [x] Giả lập phí taker — phí **0,055**
- Đòn bẩy **5** · Size **30%** · Số vị thế tối đa **1** · Chốt lời **8%** (giá chạy 1,6%) · Lỗ tối đa **60%** (giá ngược 12%, không bị kẹp) · Cooldown **4**
- Turnover tối thiểu **5.000.000** · Spread tối đa **0,15** · Tuổi niêm yết **24** · Top-K **60**
- Ký quỹ tối đa mỗi lệnh **60** · Tổng ký quỹ tối đa **60** · Lỗ tối đa trong ngày **40** · SL cách thanh lý **0,5**
- Kết quả: một lệnh thua mất ≈ 18% tài khoản (thay vì ≈ 85–90% như cấu hình hiện tại), một lệnh thắng ≈ +2% tài khoản. Vẫn cần tỉ lệ thắng rất cao (≈ 90%) mới có lời — hãy kiểm chứng ở DEMO.

---

## 6. Cấu hình hiện tại của bạn

| Ô | Giá trị của bạn | Nhận xét |
|---|---|---|
| ĐÁNH NGƯỢC tín hiệu | tick | OK nếu cố ý. Nhớ: cột "Chiều lệnh" trong bảng ứng viên là chiều **tín hiệu** |
| Hướng giao dịch | Cả hai | OK |
| Bắt buộc EMA20 (15m) | tick | Với đánh ngược: bot LONG khi giá dưới EMA20, SHORT khi giá trên EMA20 |
| Bắt buộc EMA50 (1h) | tick | ⚠ **Mâu thuẫn 4.1**: bot đi ngược cả xu hướng 1 giờ; cùng với "setup MỚI" → rất ít lệnh |
| Bắt buộc tín hiệu | không tick | OK |
| Bắt buộc volume | không tick | OK |
| Chặn funding bất lợi | không tick | Đúng khi đánh ngược (tránh mâu thuẫn 4.3). Bù lại không có gì chặn lệnh phải trả funding cao: funding 0,1% ở x10 = −1% ký quỹ mỗi kỳ |
| Bật chống đuổi giá | tick | Tốt, giữ |
| Setup còn MỚI, tuổi 3 | tick | Tốt, giữ |
| Size **99%**, đòn bẩy **10**, tối đa **1** vị thế | | Ký quỹ ≈ 97,9% số dư mỗi lệnh. `max_positions` = 1 là đúng với size 99 (tránh mâu thuẫn 4.5) |
| Chốt lời **8%** | | = giá chạy **0,8%**. Trừ phí ≈ 1,1% → ròng ≈ **+6,9% ký quỹ** |
| Lỗ tối đa **99%** | | ⚠ **4.7**: SL thật trên sàn ≈ **-85…-90% ký quỹ** (giá ngược ~8,5–9%). 99 hay 90 đều như nhau ở x10 |
| Cooldown **4** | | OK |
| Turnover tối thiểu **1.000.000** | | Thấp hơn mặc định → nhiều coin nhỏ, giật mạnh, dễ chạm SL. Khuyên ≥ 3.000.000 |
| Các ô "Tiền thật" = **0** | | Không có giới hạn nào. Khuyên đặt ít nhất "Ký quỹ tối đa mỗi lệnh" và "Lỗ tối đa trong ngày" |
| SL cách thanh lý **0,5** | | OK |
| Phí taker **0,055** | | OK, **đừng hạ** (mâu thuẫn 4.6 → lỗi 110007) |

**Con số quan trọng nhất với cấu hình hiện tại** (tài khoản 100 USDT):
- Mỗi lệnh: ký quỹ ≈ 97,9 USDT, giá trị lệnh ≈ 979 USDT.
- Thắng (giá chạy đúng 0,8%): ≈ **+6,8 USDT**.
- Thua (chạm SL, giá ngược ~8,5–9%): ≈ **−84 đến −89 USDT**, và bot **dừng** (`STOPPED_MAX_LOSS`).
- Cần khoảng **13 lệnh thắng liên tiếp để bù 1 lệnh thua** → tỉ lệ thắng phải trên **≈ 93%** mới hoà vốn.
- Spread tối đa 0,3% ở x10 có thể ăn ≈ 1,5% ký quỹ ngay khi vào — gần 1/5 khoản lời mục tiêu.

**Nếu bạn bỏ tick tất cả, chỉ giữ "Chỉ vào lệnh khi setup còn MỚI"** (cộng "Bật chống đuổi giá" — **nên giữ**):
- Bỏ EMA50: **hết mâu thuẫn 4.1** với xu hướng 1 giờ — nên làm.
- Bỏ EMA20: số lệnh **tăng rõ**. Một coin đạt khi vừa cắt EMA20 **hoặc** vừa phá đỉnh/đáy 20 nến trong 3 nến gần nhất. Nhưng "vừa phá đỉnh" vẫn tính dù giá **đã rơi lại** dưới EMA20; khi đó cả LONG lẫn SHORT có thể cùng đạt và bot chọn theo điểm → lệnh **nhiễu hơn**, logic đánh ngược kém rõ ràng (có lúc bot lại đi **cùng** chiều giá vừa chạy).
- Funding vẫn không tick là đúng (vì đang đánh ngược).
- Giữ "Bật chống đuổi giá": đây là lớp chặn duy nhất còn lại giúp bot không LONG ngay sau cú sập mạnh / SHORT sau cú tăng vọt.
- Với size 99% x10 và nhiều lệnh hơn, bạn **gặp lệnh thua (≈ −85–90% tài khoản) sớm hơn**.
- **Gợi ý**: bỏ EMA50, **giữ EMA20**, giữ chống đuổi giá + setup MỚI; giảm size/đòn bẩy như mẫu 5.3; đặt "Ký quỹ tối đa mỗi lệnh" và "Lỗ tối đa trong ngày". Chạy DEMO vài ngày trước.

---

## 7. ⛔ Nhắc nhở rủi ro
- Ở **LIVE** bot dùng **TIỀN THẬT**. Bạn có thể mất toàn bộ số tiền trong tài khoản.
- Với **size 99% và đòn bẩy x10**, **một lần chạm cắt lỗ** (giá đi ngược khoảng 8,5–9%) **mất khoảng 85–90% tài khoản**. Trượt giá, giá giật mạnh hoặc sàn bảo trì có thể làm lỗ nhiều hơn.
- TP nhỏ (0,8% giá) + SL xa (~9% giá) = thắng ít, thua đậm: cần thắng **trên ~93%** số lệnh mới không lỗ.
- Các giới hạn tiền thật mặc định **TẮT (= 0)** — bạn phải tự đặt.
- Kết quả PAPER/DEMO **không đảm bảo** kết quả LIVE. Chỉ dùng số tiền bạn chấp nhận mất hết.
