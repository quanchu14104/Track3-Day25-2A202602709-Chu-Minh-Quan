# Lab 25B — Report

**Họ tên / nhóm:** Chu Minh Quan (MSSV: 2A202602709)  **Commit:** `b3c1dc2`  **Điểm tự động (`make grade`):** 84.0 / 88 *(100% test & public benchmark pass; 4 điểm còn lại thuộc dataset ẩn chấm bởi giảng viên)*

> Mọi nhận định có số liệu đối chiếu từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json` và `reports/cache/*.json`.

---

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| **G1** | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| **G1** | yours | **1.0000** | **4.786** | **1.064** | **0.001** | **2.101** | **0.0** | **PASS** |
| **G2** | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| **G2** | yours | **1.0000** | **9.789** | **1.032** | **0.003** | **2.100** | **0.0** | **PASS** |
| **G3** | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| **G3** | yours | **1.0000** | **4.714** | **1.036** | **0.016** | **2.110** | **0.0** | **PASS** |
| **G4** | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| **G4** | yours | **1.0000** | **5.536** | **1.016** | **0.003** | **2.091** | **0.0** | **PASS** |
| **G5** | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | null | FAIL |
| **G5** | yours | **0.8108** | **5.578** | **1.003** | **0.002** | **2.521** | null | **PASS** |
| **G6** | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| **G6** | yours | **1.0000** | **6.369** | **1.058** | **0.022** | **2.185** | **0.0** | **PASS** |

---

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | 8.0 s | Dựa trên **G6** và **G4**. Turn deadline là 30 s. Ở G6 khi `alpha-eu` bị treo, nếu timeout quá dài (ví dụ 30 s), toàn bộ budget thời gian của lượt user bị tiêu sạch ở attempt đầu và không thể failover (availability giảm xuống 94.88% < 95%). Giá trị 8.0 s đủ rộng cho median 1.2–1.8 s của các model nhưng kịp thời giải phóng call để failover sang `beta`. |
| `max_attempts_per_provider` | 2 | Dựa trên **G2** và **G3**. Giới hạn đúng 1 lần retry trên cùng provider (tối đa 2 attempts). Bản naive có 3 tầng retry chồng chéo (36 attempts) khiến amplification vọt lên 9.9 ở G2 và 17.0 ở G5. Nếu provider gặp lỗi transient 2 lần liên tiếp, failover sang provider tiếp theo hiệu quả hơn nhiều so với việc tiếp tục dồn tải vào node đang nghẽn. |
| `backoff_base_s` / `cap_s` | 0.25 s / 4.0 s | Dựa trên **G1** và **G3**. Base 0.25 s cho phép retry nhanh với các lỗi mạng chớp nhoáng, trong khi cap 4.0 s kết hợp Full Jitter dàn đều request, tránh hiện tượng các client retry đồng loạt vào cùng một thời điểm. |
| `retry_budget_ratio` | 0.1 | Dựa trên **G3** (`retry_ratio <= 0.10`) và **G2**. Token bucket nạp 10% cho mỗi request mới, khống chế số lượng retry không bao giờ vượt quá 10% tổng lưu lượng, triệt tiêu nguy cơ hình thành retry storm. |
| `breaker.window_size` / `min_calls` | 20 / 10 | Dựa trên **G2**, **G3**, **G5**. Cần tối thiểu 10 sample để tỷ lệ lỗi có ý nghĩa thống kê, tránh mở mạch do vài lỗi ngẫu nhiên. Cửa sổ trượt 20 giúp breaker phản ứng nhanh khi tỷ lệ lỗi tăng vọt và xóa bỏ lịch sử cũ nhanh chóng khi hệ thống hồi phục. |
| `breaker.failure_rate_threshold` | 0.5 | Dựa trên **G2** và **G3**. Khi tỷ lệ lỗi vượt quá 50%, khả năng provider đó phục vụ thành công tiếp theo là rất thấp; circuit trip sang OPEN giúp failover ngay sang `alpha-eu` và cho `alpha` cơ hội giải tỏa hàng đợi. |
| `breaker.slow_call_s` | 5.0 s | Dựa trên **G4**. Provider median là 1.2–1.5 s. Ở G4, `alpha` không trả lỗi HTTP nào nhưng bị chậm thêm 5 s (thành ~6.5 s). Bản naive có p95 vọt lên 13.764 s (> 10 s threshold). Nhờ `slow_call_s = 5.0 s`, breaker nhận diện sự suy thoái hiệu năng và trip mở mạch, giữ p95 ở 5.536 s. |
| `breaker.slow_rate_threshold` | 0.6 | Dựa trên **G4**. Khi >= 60% request trong cửa sổ bị chậm >= 5 s, breaker xác nhận hạ tầng provider đang bị nghẽn nghiêm trọng và chuyển sang OPEN. |
| `breaker.open_duration_s` | 10.0 s | Dựa trên **G2**. Cho provider 10 giây nghỉ ngơi để đào thải các request cũ quá hạn trước khi thử thăm dò lại ở HALF-OPEN. |
| `breaker.half_open_max_probes` | 2 | Dựa trên **G2** và unit test. Giới hạn tối đa 2 request probe đồng thời ở HALF-OPEN, ngăn chặn thundering herd làm sập lại provider vừa mới ngóc đầu dậy. |

### Sơ đồ quyết định của `on_error`

```
                                  [ ProviderError ]
                                          │
                                 classify(err)
                                          │
            ┌──────────────────┬──────────┴──────────┬────────────────────────┐
            ▼                  ▼                     ▼                        ▼
     [ CLIENT_ERROR ]   [ QUOTA_EXHAUSTED ]   [ RATE_LIMITED ]        [ OVERLOADED / ]
      (400, 401, 422)    (429 no quota)        (429 + retry-after)    [ SERVER_ERROR /]
            │                  │                     │                [ TIMEOUT       ]
      Breaker: release   Breaker: release      Breaker: release       (529, 503, 500, 0)
            │                  │                     │                        │
        GiveUp                 │                     │                Breaker: on_failure
    ("client_error")    Cooldown provider:    Cooldown provider:              │
    (Không retry,       now + 300 s           now + retry_after       Kiểm tra điều kiện:
     không failover)           │                     │                - tries < max_attempts (2)
                               └──────────┬──────────┘                - budget.try_acquire() == True
                                          │                           - delay + timeout <= deadline
                                      Failover                                │
                                  (sang node kế tiếp                  ┌───────┴───────┐
                                   trong ladder)                      ▼               ▼
                                                                   [ YES ]         [ NO ]
                                                                      │               │
                                                                 Retry SAME       Failover
                                                                 provider với   (chuyển sang
                                                                  full jitter    node kế tiếp)
```

---

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.**  
   Khi giả lập breaker bị vô hiệu hóa (`min_calls: 1_000_000_000`) và tăng `max_attempts_per_provider: 5`:
   - Ở kịch bản **(a) giữ retry budget**: G2 đạt `amplification = 1.383`, `retry_ratio = 0.058`, `recovery_s = 0.0 s`, `availability = 0.9987`; G3 đạt `amplification = 1.136`, `retry_ratio = 0.068`.
   - Ở kịch bản **(b) tắt retry budget**: G2 bị khuếch đại mạnh lên `amplification = 1.672`, `retry_ratio = 0.491` (tăng gấp **8.5 lần**), `recovery_s = 30.0 s`, `availability = 0.9649`; G3 có `retry_ratio = 0.162`.
   - *Kết luận*: Khi breaker hoạt động đúng, nó bảo vệ provider bằng cách cắt mạch sớm. Tuy nhiên, nếu breaker gặp lỗi cấu hình, misconfiguration hoặc cửa sổ trượt chưa đủ mẫu (dưới `min_calls`), retry budget hoạt động như chốt an toàn định lượng cuối cùng (rate-limiting retries) khống chế không cho phép client tạo ra retry storm vượt quá 10% lưu lượng. Do đó, một hệ thống resilient bắt buộc phải có cả hai tầng phòng thủ độc lập.

2. **`attempt_timeout_s: 30`.**  
   Khi nâng attempt timeout lên 30 s:
   - Trên **G4** (slow region): availability vẫn giữ **1.0000**, p95 đạt 5.683 s, `give_up_reasons: {}` (không có cuộc gọi nào bỏ cuộc vì breaker slow-call kịp thời phát hiện và trip mở mạch sau 5 s).
   - Trên **G6** (provider outage, `alpha-eu` treo vô hạn): availability sụt giảm nghiêm trọng từ **1.0000 xuống 0.9488** (vi phạm ngưỡng `>= 0.95`). Xem xét lý do từ bỏ, hệ thống ghi nhận `give_up_reasons: {'deadline': 109, 'no_provider_available': 13}` — có tới **109 cuộc gọi thất bại vì chạm deadline 30 s** trước khi có cơ hội failover.
   - *Cơ chế*: Khi một provider bị treo vô hạn (blackhole), timeout 30 s ngốn trọn vẹn toàn bộ 30 s deadline của lượt người dùng. CallContext hết hạn tại attempt đầu tiên, triệt tiêu cơ hội failover sang `beta`. Timeout "đúng" không được đặt bằng turn deadline mà phải dựa trên phân phối độ trễ bình thường của provider (ví dụ $p99 + \text{buffer}$, khoảng 4–6× median service time, tức 8 s) để đảm bảo client kịp failover ít nhất 1–2 bậc trước khi chạm deadline.

3. **Bỏ lọc tool calling khỏi `ladder()`.**  
   Khi bỏ lọc `supports_tools` khỏi `ladder()` trên băng **G6**:
   - Số lượng `incompatible_routes` tăng vọt từ **0 lên 258** cuộc gọi không tương thích, kéo availability tụt từ **1.0000 xuống 0.8918** (thất bại).
   - *Góc nhìn Product Manager*: Việc gửi request có yêu cầu gọi tool sang model `gamma-mini` (vốn không hỗ trợ tool calling) tương đương với việc chuyển tiếp 258 khách hàng đang cần hoàn tiền đến một nhân viên hoàn toàn không có quyền hạn truy cập hệ thống ngân hàng. Model chỉ có thể trả lời bừa hoặc báo lỗi cú pháp, làm đứt gãy 258 quy trình nghiệp vụ và trực tiếp phá vỡ trải nghiệm khách hàng cũng như uy tín thương hiệu.

---

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid (cents) | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,574,200 | 1.123 |
| yours | **1.0000** | **66** | **0** | **0** | **0** | **0** | **0.303** |

### Phân tích Idempotency Key & Checkpoint

- **Vì sao chỉ idempotency key là chưa đủ?**  
  Nếu chỉ có Idempotency Key mà không có Checkpoint: khi worker bị crash (ví dụ ở bước 4 `send_email`), worker mới khởi động lại từ đầu mà không có bộ nhớ về tiến trình. Nó sẽ phải thực thi lại toàn bộ các bước LLM phía trước (`classify_request`, `lookup_order`), gây lãng phí chi phí token LLM lớn (`wasted_llm_per_crash > 1.1`) và tăng độ trễ xử lý.
- **Vì sao chỉ checkpoint là chưa đủ?**  
  Nếu chỉ có Checkpoint mà không có Idempotency Key: khi một tác vụ side-effect (như trừ tiền) gặp `ServiceTimeout` (ngân hàng đã trừ tiền nhưng mất kết nối trả về) hoặc worker chết ngay sau khi thanh toán nhưng trước khi kịp ghi checkpoint, worker mới khởi động lại sẽ thực hiện lại bước thanh toán với mã mới, dẫn đến khách hàng bị trừ tiền/hoàn tiền 2 lần (`duplicate_refunds`).
- **Điều gì xảy ra nếu ghi checkpoint TRƯỚC khi gọi payment?**  
  Nếu ghi checkpoint trước rồi worker bị crash hoặc sập mạng ngay trước khi lệnh thanh toán kịp tới cổng thanh toán: worker phục hồi sau đó đọc checkpoint thấy bước `issue_refund` đã "thành công" nên sẽ bỏ qua bước này và đi thẳng sang bước gửi email thông báo hoàn tất. Khách hàng nhận được email thông báo đã nhận tiền nhưng thực tế tài khoản không nhận được xu nào -> Gây ra **lost refunds** (mất tiền hoàn), là lỗi nghiêm trọng nhất trong hệ thống tài chính.

---

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| `checkout_bug` | page | page (delay 1m) | page | 27 min |
| `provider_outage` | page | page (delay 0m) | page | 4 min |
| `deploy_blip` | none | **page (false page)** | **none** | — |
| `slow_leak` | ticket | **none (missed)** | **ticket** | 1337 min |
| `degraded_evening` | page | page (delay 214m) | page | 291 min |
| `night_spike` | none | **page (false page)** | **none** | — |
| `small_bump` | none | **page (false page)** | **none** | — |

- **Error budget 30 ngày:**  
  - Tổng số requests: `25,901,755`  
  - Lỗi tối đa cho phép (`1 - 0.995`): `129,508.8`  
  - Số lỗi thực tế: `55,618`  
  - Availability đạt được: **0.997853 (99.785%)**  
  - Ngân sách lỗi đã tiêu: **42.95%**, ngân sách còn lại: **57.05%**  
  - **Quyết định:** **KHÔNG** đóng băng release (`freeze_releases: False`) vì hệ thống vẫn còn 57.05% error budget.
- **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?**  
  Vì `small_bump` chỉ tiêu tốn chưa đầy 0.2% error budget tháng (20 phút × 12% lỗi trên tổng số 43,200 phút). Sự cố ngắn hạn này không đe dọa đến cam kết SLO 30 ngày (99.5%). Việc gửi page cảnh báo âm thanh đánh thức kỹ sư on-call cho những blip tự phục hồi sẽ gây ra **alert fatigue** (kiệt sức vì cảnh báo sai), làm giảm sự nhạy bén và tập trung của đội ngũ khi có thảm họa thực sự xảy ra.

---

## 6. Postmortem — Băng G2 với client naive (blameless)

*Biểu đồ phân tích: `reports/gameday/G2_overload_storm.png` | Dữ liệu gốc: `reports/gameday/naive_G2_overload_storm_timeseries.csv`*

### Tóm tắt (3 câu)
Trong khoảng thời gian từ t=120 s đến t=300 s, provider chính `alpha` bị suy giảm đột ngột 75% năng lực GPU, dẫn đến ứ đọng hàng đợi và trả về mã lỗi 529. Bản client v1 `NaiveClient` với 3 tầng retry không jitter và không có circuit breaker đã kích hoạt một retry storm dữ dội, đẩy số lượng request attempt lên tới 163 attempts/s. Hậu quả là ngay cả khi provider đã khôi phục 100% năng lực vào lúc t=300 s, hệ thống vẫn tiếp tục tê liệt thêm 155 giây do rơi vào trạng thái metastable failure (hàng đợi ngập tràn request rác và tải retry duy trì vượt quá công suất xử lý).

### Ảnh hưởng
- **1,285 lượt user thất bại** (chỉ có 1,077/2,362 turns thành công, availability sụt giảm còn **45.60%**).
- Sự cố provider kéo dài 180 s (3 phút), nhưng hệ thống bị sập kéo dài tới **335 s** (~5.6 phút).
- Sự cố này đốt cháy hơn **120% error budget của toàn bộ tháng** chỉ trong chưa đầy 6 phút.

### Timeline (giây ảo — trích xuất từ timeseries CSV)

| t (s) | Sự kiện | Bằng chứng số liệu từ CSV |
|---:|---|---|
| **0 – 119** | Trạng thái bình thường | `done = 7–10/s`, `ok = 7–10/s` (tỷ lệ thành công 100%), `attempts_alpha = 5–9/s`. |
| **120** | `alpha` mất 75% năng lực GPU | Hàng đợi bắt đầu dồn ứ, `attempts_alpha = 5/s`, các request bắt đầu chờ lâu hơn. |
| **130** | Hàng đợi server chạm ngưỡng 10 s | Server bắt đầu trả về lỗi 529 `overloaded_error` và timeout. |
| **150** | Bùng nổ Retry Storm | Client naive retry liên tục 3 tầng; `attempts_alpha` tăng vọt lên **163 attempts/s**, `calls_ok = 0` (tỷ lệ thành công 0.00%). |
| **180** | Đỉnh điểm quá tải | `attempts_alpha = 138/s`, server ngập trong 3,194 lỗi 529 và 2,737 lỗi timeout, availability chạm đáy 0%. |
| **300** | `alpha` hồi phục 100% GPU | Server có đủ GPU nhưng hệ thống **vẫn không phục hồi**: `attempts_alpha = 115/s`, `calls_ok = 0` (0% success). |
| **301 – 450** | Vòng lặp Metastable Failure | Server bận rộn xử lý các request cũ mà client đã timeout bỏ cuộc (zombie work); trong khi đó client vẫn tiếp tục bắn retry với tốc độ 50–100 attempts/s giữ tải cao hơn công suất xử lý. |
| **455** | Hàng đợi được xả sạch | Sau 155 giây kể từ khi khôi phục GPU, hàng đợi xả hết rác: `done = 5, ok = 4` (tỷ lệ thành công đạt 80%), `attempts_alpha` giảm về 13/s. |
| **500** | Trở lại trạng thái ổn định | `done = 10, ok = 10` (100% success), `attempts_alpha = 9/s`, hệ thống bình thường. |

### Trigger vs Root Cause
- **Trigger (Ngòi nổ):** Sự cố hạ tầng phần cứng khiến provider `alpha` bị mất 75% GPU trong 3 phút (t=120 đến t=300).
- **Root Cause (Nguyên nhân gốc rễ):** Thiết kế client `NaiveClient` thiếu khả năng chịu tải: cấu hình 3 tầng retry lồng nhau (tối đa 36 attempts/call), khoảng thời gian delay cố định không có jitter ngẫu nhiên, không có retry budget và không có circuit breaker. Khi gặp lỗi, client tự động nhân bản tải lên gấp **9.9 lần** (34,475 attempts cho 3,481 logical calls).
- **Cơ chế Metastable Failure:** Khi hàng đợi của server bị đầy, client hết kiên nhẫn và từ bỏ (client timeout), nhưng server vẫn tiếp tục xử lý các request cũ này theo hàng đợi FIFO (công sức bị lãng phí hoàn toàn). Cùng lúc đó, các đợt retry dồn dập đổ vào giữ cho hàng đợi luôn ở trạng thái đầy ứ. Tải tự sinh ra từ retry vượt quá năng lực phục hồi của provider, duy trì vòng lặp lỗi ngay cả khi ngòi nổ ban đầu đã biến mất.

### Điều gì giúp / Điều gì làm tệ hơn
- **Làm tệ hơn:** Retry nhiều tầng không có jitter; không có circuit breaker cắt tải; client timeout nhưng server không hỗ trợ cancel request đang xếp hàng.
- **Giúp hồi phục:** Lưu lượng người dùng giảm tự nhiên về đêm/cuối scenario, cho phép server xả hết các request tồn đọng.

### Action Items

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai `ResilientClient` tích hợp Circuit Breaker sliding-window (trip khi lỗi >= 50% hoặc chậm >= 60%) và Retry Budget (ratio 0.1) | Mitigate | SRE Team | Băng G2 đạt `amplification <= 1.6` và `recovery <= 30 s` trong CI test suite. |
| 2 | Loại bỏ hoàn toàn retry ở tầng Application và Gateway; chỉ giữ 1 tầng retry tại SDK với Full Jitter Exponential Backoff và tuân thủ `retry-after` | Prevent | Platform Team | Test unit `test_backoff_has_jitter` pass và 0 vi phạm retry-after trên mọi kịch bản. |
| 3 | Xây dựng Fallback Ladder tự động chuyển hướng thông minh sang `alpha-eu` và `beta` khi provider chính bị mở mạch | Prevent | Core Architecture | Băng G2 đạt availability >= 95% và G6 đạt 0 lỗi incompatible routes. |
| 4 | Thiết lập Multi-window Multi-burn-rate Alerting (page 14.4x trong 1h/5m) để phát hiện và cảnh báo sự cố overload trong vòng dưới 5 phút | Detect | Monitoring Team | Kiểm tra trên dữ liệu SLI 30 ngày đạt 0 false pages và phát hiện đúng 100% sự cố lớn. |

---

## 7. Một điểm yếu còn lại

- **Điểm yếu:**  
  Client hiện tại sử dụng **Fallback Ladder tĩnh** (`alpha -> alpha-eu -> beta -> gamma-mini`). Khi `alpha` gặp sự cố (như trong G6 hay G2), toàn bộ lưu lượng bị dồn tức thì sang `alpha-eu` và `beta`. `beta` có chi phí cao hơn 40% (1.4 vs 1.0) và giới hạn rate limit chỉ là 16 rps (thấp hơn 20 rps của `alpha`). Nếu toàn bộ người dùng ồ ạt chuyển sang `beta`, `beta` sẽ nhanh chóng bị nghẹt rate limit, gây ra hiện tượng **cascade failure** (đổ vỡ dây chuyền). Ngoài ra, nếu tất cả các provider đều bị suy thoái mạng cùng lúc, việc duyệt tuần tự qua ladder sẽ khiến latency p95 tăng cao.
- **Phương án khắc phục:**  
  1. Thay thế ladder tĩnh bằng **Dynamic Weighted Routing** kết hợp thuật toán **P2C (Power of Two Choices)** và **EWMA Latency**: Chọn ngẫu nhiên 2 provider có hỗ trợ tool và gửi request tới node có inflight request và EWMA latency thấp hơn.
  2. Áp dụng **Adaptive Concurrency Limiting** (như thuật toán AIMD của TCP / Netflix Concurrency Limits): Client tự động điều chỉnh số lượng kết nối đồng thời tối đa tới từng provider dựa trên Little's Law và thời gian phản hồi thực tế, chủ động từ chối mềm (shed load) hoặc phục vụ câu trả lời dự phòng trước khi làm nghẹt hàng đợi provider.

---

## 8. Cache (Part G)

*Số liệu trích xuất từ `reports/cache/*.json` (`make cache` trên băng C1).*

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| `resilient` (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| `cached` (của bạn) | **0.9124** | **0.9645** | **0.592** | **192** | **0** | **2** | **0.955** | **PASS** |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| `similarity_threshold` | 0.90 | Với ngưỡng starter 0.80, cache trả lời sai tới **181 câu hỏi** (`other_question: 181`) vì bắt nhầm các câu hỏi khác nhau. Ngưỡng 0.90 loại bỏ hoàn toàn câu trả lời sai (`wrong_answers = 0`) trong khi vẫn duy trì tỷ lệ cache hit cao (**59.2%** > 50%). Nếu tăng lên 0.97, tỷ lệ hit tụt xuống 49.9% (dưới chuẩn 50%). |
| `ttl_s` | 30 s | Ở t=480 s, bảng phí ship thay đổi âm thầm không có webhook. Với starter TTL 600 s hoặc 300 s, số câu trả lời lỗi thời lên tới **80–95 câu** (vượt trần 15). Giảm TTL xuống 30 s giới hạn số câu trả lời cũ chỉ còn **2 câu** (đạt chuẩn `<= 15`) mà vẫn duy trì hit ratio 59.2%. |
| `max_stale_s` | 300 s | Sự cố toàn bộ provider sập ở C1 kéo dài 120 s (t=240 đến 360 s). Giá trị starter `max_stale_s: 0` khiến `faq_availability` chỉ đạt 85.29% (thất bại). Đặt 300 s cho phép phục vụ **192 câu trả lời stale an toàn**, kéo `faq_availability` lên **96.45%** (vượt yêu cầu >= 93%). |

### Thí nghiệm cache

1. **Ngưỡng similarity:**  
   - Chạy với `similarity_threshold = 0.80`: `cache_hit_ratio = 0.610`, `wrong_answers = 181` (`other_question: 181`), `cost/turn = 1.015`.  
   - Chạy với `similarity_threshold = 0.90` (của chúng tôi): `cache_hit_ratio = 0.592`, `wrong_answers = 0`, `cost/turn = 0.955`.  
   - Chạy với `similarity_threshold = 0.97`: `cache_hit_ratio = 0.499` (FAIL do `< 0.5`), `wrong_answers = 0`, `cost/turn = 1.018`.  
   - *Phân tích*: Ở 0.80, cache ghép nhầm các câu hỏi có embedding gần nhau nhưng ngữ nghĩa nghiệp vụ hoàn toàn khác nhau (ví dụ: câu hỏi về "thời gian nhận tiền hoàn" bị trả lời nhầm bằng "chính sách điều kiện hoàn tiền", hoặc "phí ship nội thành" bị nhầm thành "phí ship liên tỉnh"). Ngưỡng 0.90 là điểm cân bằng tối ưu giữa độ chính xác tuyệt đối và tỷ lệ trúng cache.

2. **TTL:**  
   - Chạy với `ttl_s = 30 s`: `outdated_answers = 2`, `cache_hit_ratio = 0.592`, `faq_availability = 0.9645`.  
   - Chạy với `ttl_s = 60 s`: `outdated_answers = 3`, `cache_hit_ratio = 0.652`, `faq_availability = 0.9645`.  
   - Chạy với `ttl_s = 300 s`: `outdated_answers = 95` (FAIL nặng vì trần cho phép là `<= 15`).  
   - *Phân tích*: Khi nhà vận chuyển thay đổi bảng giá lúc t=480 s mà không có webhook báo trước (silent change), cache layer hoàn toàn không nhận được bất kỳ sự kiện invalidation chủ động nào. Do đó, thời gian hết hạn tự nhiên (TTL) là cơ chế phòng thủ duy nhất để ép buộc cache phải loại bỏ dữ liệu cũ và truy vấn câu trả lời mới từ provider.

3. **Stale-if-error:**  
   - Chạy với `max_stale_s = 0`: `faq_availability = 0.8529` (FAIL vì `< 0.93`), `stale_served = 0`.  
   - Chạy với `max_stale_s = 300`: `faq_availability = 0.9645` (PASS), `stale_served = 192`.  
   - *Phân tích*: Khi toàn bộ provider bị sập từ t=240 đến t=360, các mục cache đã hết hạn TTL sẽ không thể revalidate. Nếu không có `stale-if-error`, client buộc phải trả về lỗi `GiveUp`. Bằng cách cho phép phục vụ câu trả lời cũ đã qua hạn trong lúc sự cố, client đã cứu được 192 lượt người dùng hỏi các câu hỏi FAQ phổ biến.

### Trả lời câu hỏi nghiệp vụ & vận hành

- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") KHÔNG được cache, kể cả khi cache theo từng user?**  
  1. *Rủi ro rò rỉ dữ liệu cá nhân (Data Leakage)*: Nếu xảy ra lỗi phân đoạn user hoặc độ tương đồng embedding cao giữa hai câu hỏi của hai user khác nhau, hệ thống có thể trả nhầm thông tin đơn hàng nhạy cảm (địa chỉ, số điện thoại, giá trị đơn) của người này cho người khác.  
  2. *Dữ liệu mang tính thời điểm cao (Stale Real-time State)*: Trạng thái đơn hàng biến động liên tục theo thời gian thực (đang lấy hàng -> đang giao -> đã giao). Cache câu trả lời cá nhân sẽ trả lời thông tin cũ đã sai lệch, gây hoang mang và khiếu nại từ khách hàng.  
  3. *Hiệu quả thấp (Low Cache Hit)*: Mỗi mã đơn hàng là duy nhất (high cardinality), xác suất cùng một user hỏi lặp lại chính xác cùng một đơn trong vài phút là cực kỳ thấp, không đem lại giá trị tối ưu chi phí.

- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**  
  Nếu chỉ đo `user_facing_availability` ở tầng cache ngoài cùng, dashboard sẽ "xanh giả tạo" và che giấu sự cố hạ tầng ngầm. Để phát hiện sự cố, hệ thống giám sát cần thiết lập cảnh báo dựa trên:
  1. **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_ratio` hoặc `stale_served_count > 0`)**: Số lượng câu trả lời stale tăng vọt là bằng chứng chắc chắn upstream provider đã sập và client đang phải dùng phao cứu sinh cuối cùng.
  2. **Upstream Provider Error Rate (đo tại client trước tầng cache)**: Cảnh báo trực tiếp khi tỷ lệ 5xx / timeout của các LLM provider vượt ngưỡng (ví dụ > 5%).
  3. **Tỷ lệ thất bại của các Cache Miss (`cache_miss_error_rate > 10%`)**: Các câu hỏi không có trong cache hoặc câu hỏi có tool calling đều bị lỗi khi provider sập. Giám sát tỷ lệ lỗi của luồng này sẽ phản ánh chân thực tình trạng sức khỏe của backend.
