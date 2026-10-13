# Lab 25B — Report

**Họ tên / nhóm:** Đặng Quang Hưng (2A202602719) · **Commit:** (working copy) · **Điểm tự động (`make grade`):** 84.0 / 88 (+ 4 điểm SLO hidden)

> Mọi nhận định có số liệu cụ thể trích xuất từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`, `reports/cache/*.json`.

---

## 1. Kết quả Game Day (naive vs của bạn)

| Băng | Client | availability | p95 (s) | amplification | retry_ratio | cost/turn | recovery (s) | PASS? |
|---|---|---:|---:|---:|---:|---:|---:|:---:|
| G1 | naive | 0.7194 | 7.157 | 7.738 | 6.738 | 2.163 | 4.0 | FAIL |
| G1 | yours | 1.0000 | 4.776 | 1.062 | 0.002 | 2.101 | 0.0 | **PASS** |
| G2 | naive | 0.4560 | 15.034 | 9.904 | 8.904 | 4.627 | 155.0 | FAIL |
| G2 | yours | 1.0000 | 10.920 | 1.041 | 0.007 | 2.119 | 0.0 | **PASS** |
| G3 | naive | 1.0000 | 4.768 | 1.164 | 0.164 | 2.328 | 0.0 | FAIL |
| G3 | yours | 1.0000 | 4.660 | 1.060 | 0.034 | 2.145 | 0.0 | **PASS** |
| G4 | naive | 1.0000 | 13.764 | 1.004 | 0.004 | 2.008 | 0.0 | FAIL |
| G4 | yours | 1.0000 | 5.562 | 1.015 | 0.002 | 2.072 | 0.0 | **PASS** |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | null | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | null | **PASS** |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.961 | 1.055 | 0.019 | 2.189 | 0.0 | **PASS** |

---

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| `attempt_timeout_s` | 8.0 | Băng G6 & G4: Phải đủ ngắn để khi provider treo/chậm bất thường (alpha-eu ở G6), client timeout sớm và còn đủ thời gian trong turn deadline 30s để failover sang provider khác (beta). |
| `max_attempts_per_provider` | 2 | Băng G1, G2, G3: Chỉ cho phép tối đa 1 retry trên cùng provider (tổng 2 attempts), tránh retry storm làm nghẹt hàng đợi của provider đang quá tải. |
| `backoff_base_s` / `cap_s` | 0.25 / 4.0 | Băng G2, G3: Full jitter với base 0.25s giúp phân tán tải retry, cap 4.0s để không làm trễ quá nhiều thời gian của turn (deadline 30s). |
| `retry_budget_ratio` | 0.1 | Băng G3 (`retry_ratio` ≤ 0.10): Giới hạn lượng retry không vượt quá 10% tổng số request gửi đầu tiên, ngăn chặn hoàn toàn hiện tượng khuếch đại retry khi diện rộng lỗi. |
| `breaker.window_size` / `min_calls` | 20 / 10 | Băng G2, G3, G5: Cửa sổ trượt 20 kết quả gần nhất, cần ít nhất 10 calls để tính tỷ lệ lỗi, tránh ngắt mạch oan vì 1-2 lỗi ngẫu nhiên trong giờ thấp điểm. |
| `breaker.failure_rate_threshold` | 0.5 | Băng G2, G3: Khi tỷ lệ lỗi ≥ 50% trong cửa sổ trượt, mở breaker ngay lập tức để chuyển hướng lưu lượng sang provider kế tiếp. |
| `breaker.slow_call_s` | 5.0 | Băng G4 (`latency_p95_s` ≤ 10s): Bắt các call chậm (alpha bị chậm thêm 5s dù không có lỗi HTTP), giúp breaker nhận diện provider xuống cấp. |
| `breaker.slow_rate_threshold` | 0.6 | Băng G4: Nếu 60% call thành công nhưng chậm ≥ 5s, breaker mở với lý do `slow_call_rate`, bảo đảm p95 toàn hệ thống chỉ đạt 5.56s. |
| `breaker.open_duration_s` | 10.0 | Băng G2: Thời gian chờ 10s trước khi cho phép probe ở HALF-OPEN, đủ cho provider giải phóng hàng đợi mà không kéo dài thời gian cô lập quá lâu. |
| `breaker.half_open_max_probes` | 2 | Băng G2: Chỉ cho phép tối đa 2 probe đồng thời vào provider đang hồi phục, triệt tiêu thundering herd. |

### Sơ đồ quyết định của `on_error`:

```
                    +-----------------------+
                    |  Lỗi từ Provider     |
                    +-----------+-----------+
                                |
                                v
                    +-----------------------+
                    | classify(err)         |
                    +-----------+-----------+
                                |
         +----------------------+----------------------+
         |                      |                      |
         v                      v                      v
   CLIENT_ERROR          QUOTA_EXHAUSTED          RATE_LIMITED
   (400, 401, 422)       (429 insufficient)       (429 rate_limit)
         |                      |                      |
   Release permit         Release permit         Release permit
         |                      |                      |
   GiveUp("client_error") Set cooldown 300s      Set cooldown retry_after
   (Không retry,                |                      |
    không failover)       Failover ngay          Failover ngay (delay=0)
                                |                      |
                                +----------+-----------+
                                           |
                                           v
                        +------------------------------------+
                        | OVERLOADED / SERVER_ERROR / TIMEOUT|
                        | (529, 503, 500, 502, timeout)      |
                        +------------------+-----------------+
                                           |
                                  on_failure(breaker)
                                           |
                             tries < max_attempts (2)
                                AND budget allows
                                AND fits deadline?
                                   /        \
                            (Yes) /          \ (No)
                                 v            v
                           Retry SAME    Failover sang
                           provider      provider kế tiếp
                           với backoff   trong ladder (delay=0)
```

---

## 3. Ba thí nghiệm

### 1. Retry budget là lớp phòng thủ cuối
- Khi giả lập breaker bị cấu hình sai (`min_calls: 1000000000`, `max_attempts_per_provider: 5`):
  - **(a) Khi có retry budget (ratio 0.1):**
    - Băng G2: `amplification = 1.380`, `retry_ratio = 0.058`, `recovery_s = 0.0s`, `availability = 0.9987`.
    - Băng G3: `amplification = 1.136`, `retry_ratio = 0.068`, `recovery_s = 0.0s`, `availability = 1.0000`.
  - **(b) Khi tắt retry budget:**
    - Băng G2: `amplification = 1.675`, `retry_ratio = 0.495` (tăng gấp 8.5 lần!), `recovery_s = 25.0s` (hệ thống bị kẹt phục hồi chậm), `availability = 0.9666`.
    - Băng G3: `retry_ratio = 0.159` (vượt ngưỡng cho phép 0.10).
- **Kết luận:** Khi breaker hoạt động đúng, nó cách ly provider nhanh chóng. Nhưng trong thực tế, breaker có thể cấu hình sai ngưỡng hoặc lỗi chỉ xảy ra rải rác dưới ngưỡng trip. Khi đó, retry budget là "dây an toàn" toán học cứng chặn đứng retry storm, không cho phép tổng lượng retry vượt quá tỷ lệ an toàn của hệ thống. Do đó bắt buộc phải có cả hai.

### 2. `attempt_timeout_s: 30`
- Khi đặt `attempt_timeout_s = 30.0`:
  - Băng G6: `availability` tụt từ `1.0` xuống `0.9488` (FAIL tiêu chí ≥ 0.95), do một provider bị treo (`alpha-eu`) giữ kết nối tới 30s khiến request hết kiệt turn deadline của user và không còn cơ hội failover sang `beta`.
  - Băng G4: `latency_p95_s = 5.608s`.
- **Kết luận:** Timeout "đúng" không được bằng turn deadline tổng (30s). Nó phải dựa trên p99 latency thông thường của provider (1.2–1.8s) cộng thêm một biên độ an toàn (ví dụ 8.0s), đảm bảo 1 attempt timeout vẫn để lại tối thiểu 15–20s cho các bước failover trong fallback ladder.

### 3. Bỏ lọc tool calling khỏi `ladder()`
- Khi bỏ lọc capability trong `ladder()` ở băng G6:
  - Xuất hiện `incompatible_routes = 258`.
  - `availability` tụt thảm hại từ `1.0` xuống `0.8918`.
- **Góc nhìn Product Manager:** 258 khách hàng yêu cầu hoàn tiền bị hệ thống đẩy sang mô hình giá rẻ `gamma-mini` (không có khả năng gọi tool truy vấn đơn và thanh toán). Hậu quả là agent hoặc báo lỗi ngớ ngẩn hoặc trả lời sai lệch, khách hàng không nhận được tiền hoàn và đánh giá dịch vụ CSKH 1 sao. Fallback ladder bắt buộc phải tôn trọng capability contract.

---

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0000 | 81 | 128 | 81 | 0 | 1,574,200 đ | 1.123 |
| yours | 1.0000 | 66 | 0 | 0 | 0 | 0 đ | 0.303 |

### Trả lời câu hỏi:
- **Vì sao chỉ idempotency key là chưa đủ?** Idempotency key chỉ bảo đảm server thanh toán không trừ tiền 2 lần khi nhận cùng 1 key. Nhưng nếu không có checkpoint, khi worker bị crash ở bước gửi email hoặc tóm tắt, worker mới khởi động lại sẽ phải chạy lại từ bước 1 (`classify_request`, `lookup_order`), gây lãng phí chi phí gọi LLM (`wasted_llm_per_crash` cao).
- **Vì sao chỉ checkpoint là chưa đủ?** Nếu worker gọi payment thành công nhưng bị rớt mạng ngay trước khi nhận response hoặc bị crash trước khi kịp ghi checkpoint, worker mới sẽ không thấy checkpoint và gọi lại payment. Không có idempotency key, cổng thanh toán sẽ trừ tiền lần thứ hai (thực hiện thanh toán trùng).
- **Điều gì xảy ra nếu ghi checkpoint *trước* khi gọi payment?** Nếu ghi checkpoint `issue_refund` trước khi gọi cổng thanh toán, khi worker bị crash đúng lúc đang gửi request (hoặc trước khi request tới nơi), worker mới đọc thấy checkpoint đã tồn tại nên bỏ qua bước hoàn tiền. Kết quả là khách hàng **bị mất tiền hoàn** (`lost_refunds > 0`), vi phạm nghiêm trọng cam kết tài chính.

---

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | paged (start: 3481) | paged (`fast_burn_2pct_1h`, start: 3507) | 27 min |
| provider_outage | page | paged (start: 8760) | paged (`fast_burn_2pct_1h`, start: 8764) | 4 min |
| deploy_blip | none | **paged (false alarm)** | **none (không page)** | — |
| slow_leak | ticket | missed (0 ticket) | ticketed (`budget_leak_10pct_3d`) | 1337 min |
| degraded_evening | page | paged (start: 30214) | paged (`slow_burn_5pct_6h`, start: 30291) | 291 min |
| night_spike | none | **paged (false alarm)** | **none (không page)** | — |
| small_bump | none | **paged (false alarm)** | **none (không page)** | — |

### Error budget 30 ngày:
- Tổng request: 25,901,755
- Allowed bad (SLO 99.5%): 129,508.8
- Thực tế bad: 55,618
- Availability đạt được: **99.7853%**
- **Budget đã tiêu:** **42.95%**, **Budget còn lại:** **57.05%**.
- **Có đóng băng release không?** **Không** (`freeze_releases: False`), vì ngân sách lỗi vẫn còn hơn 57%.
- **Vì sao `small_bump` (20 phút ở 12% lỗi) không nên đánh thức ai lúc 11 giờ trưa?**
  Mặc dù tỷ lệ lỗi 12% trong 20 phút nhìn có vẻ cao, nhưng tổng số lỗi phát sinh chỉ tiêu thụ một phần rất nhỏ (< 0.5%) của ngân sách lỗi 30 ngày và tự kết thúc nhanh chóng. Đánh thức kỹ sư on-call cho những sự cố ngắn hạn tự hồi phục tạo ra báo động giả (alert fatigue), làm giảm năng suất và phân tâm khỏi các sự cố suy thoái thực sự.

---

## 6. Postmortem — Băng G2 với client naive (blameless)

![G2 Overload Storm](gameday/G2_overload_storm.png)

### Tóm tắt (3 câu):
Vào t=120s, deployment chính `alpha` mất 75% năng lực xử lý trong 3 phút do sự cố phần cứng hạ tầng. Client naive (v1) với 3 tầng retry chồng chéo không có jitter, không circuit breaker và không fallback đã dồn dập gửi hàng nghìn retry vào cụm đang nghẽn. Hệ thống rơi vào trạng thái hỏng hóc siêu ổn định (metastable failure), kéo dài thời gian ngừng trệ thêm 155 giây sau khi provider đã hoàn toàn hồi phục, khiến tỷ lệ thành công của user chỉ còn 45.6%.

### Ảnh hưởng:
- **1,285 lượt user thất bại** (chỉ có 1,077 / 2,362 turns thành công, availability 45.6%).
- Sự cố kéo dài tổng cộng 335 giây (180s thời gian provider lỗi + 155s thời gian hệ thống kẹt phục hồi).
- Tỷ lệ lỗi 54.4% trong gần 6 phút đã ngốn sạch hơn 120% error budget của cả tháng.

### Timeline (giây ảo — trích từ `naive_G2_overload_storm_timeseries.csv`):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 0–119 | Hoạt động bình thường | Traffic ổn định 6–10 turns/s, `attempts_alpha` tương đương `calls_ok`. |
| 120 | Alpha mất 75% năng lực | Năng lực phục vụ sụt giảm, hàng đợi alpha bắt đầu đầy. |
| 136 | Retry storm bùng phát | `attempts_alpha` tăng vọt từ 53 lên 82, 122, rồi chạm đỉnh 254 req/s tại t=143. |
| 140–299 | Hệ thống sập toàn diện | `calls_ok` rớt về 0. Lỗi 529 và 429 tràn ngập, hàng đợi nghẽn cứng. |
| 300 | Alpha phục hồi 100% phần cứng | Năng lực 18 slot được khôi phục, nhưng `calls_ok` vẫn là 0! |
| 301–454 | Metastable failure | Server vẫn đang miệt mài xử lý hàng nghìn request cũ đã hết hạn 30s deadline; request mới vào tiếp tục bị timeout hoặc 429. |
| 455 | Hệ thống bắt đầu phục hồi | Hàng đợi cũ được dọn sạch, `calls_ok` bắt đầu tăng trở lại 4–9 calls/s. |
| 458 | Phục hồi hoàn toàn | `recovery_s = 155.0s`. |

### Trigger vs Root cause:
- **Trigger:** Sự sụt giảm 75% GPU tạm thời của provider `alpha` tại t=120.
- **Root cause:** Kiến trúc client naive có 3 tầng retry độc lập chồng lên nhau (app × gateway × SDK) tạo ra hệ số khuếch đại lên tới 9.9 lần (`amplification = 9.904`), không có circuit breaker ngắt tải, và không có fallback sang các cụm khác (`alpha-eu`, `beta`).
- **Cơ chế metastable failure:** Sau khi alpha khôi phục năng lực ở t=300, hàng nghìn retry tồn đọng vẫn nằm trong hàng đợi FIFO của server. Server tiêu tốn 100% CPU/GPU xử lý các request mà phía client đã hết kiên nhẫn (bỏ cuộc sau 30s deadline). Công sức xử lý bị lãng phí hoàn toàn, tạo thành vòng lặp tự duy trì (work-amplifying loop) kéo dài sự cố thêm 155s.

### Điều gì giúp / Điều gì làm tệ hơn:
- **Làm tệ hơn:** Delay cố định không có jitter khiến các request retry đập vào cùng thời điểm; không tôn trọng header `retry-after` (vi phạm 28,381 lần); không có retry budget.
- **Điều giúp khắc phục (trong ResilientClient):** Circuit breaker phát hiện failure rate ngắt mạch chuyển sang `alpha-eu` và `beta`; retry budget giới hạn retry ≤ 10%; và backoff full jitter.

### Action items:

| # | Hành động | Loại | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Thay thế NaiveClient bằng ResilientClient (Circuit breaker + Fallback ladder + Token-bucket budget) | Prevent | Core Platform Team | Chạy Game Day G2 đạt recovery = 0s, availability = 100% |
| 2 | Cấu hình multi-window burn-rate alerting theo SRE Workbook | Detect | SRE On-call Team | Mô phỏng sli_30d phát hiện outage trong < 5 phút, 0 false alarm |
| 3 | Tích hợp timeout propagation và deadline context hủy request phía server khi client drop | Mitigate | Backend Lead | Kiểm tra hàng đợi server rỗng ngay khi client hủy kết nối |

---

## 7. Một điểm yếu còn lại

- **Điểm yếu:** Khi chuyển hướng tải sang provider `beta` (như ở G6 hoặc khi alpha sập), chi phí trên mỗi lượt user tăng đáng kể từ 2.10 lên 2.50+ (`cost_per_call` của beta là 1.4 so với 1.0 của alpha). Ngoài ra, nếu có sự cố mạng toàn cầu khiến cả 4 provider cùng phản hồi chậm (degraded latency diện rộng), fallback ladder vẫn phải thử lần lượt qua các provider, dẫn tới p95 tổng của turn có thể chạm sát trần 30s của người dùng.
- **Cách khắc phục:** 
  1. Triển khai **Hedging / Speculative Execution có kiểm soát**: nếu provider chính không phản hồi sau p95 thông thường (ví dụ 2.5s), phát một request song song sang provider dự phòng mà không hủy request đầu, lấy kết quả của bên về trước.
  2. Bổ sung cơ chế **Cost-aware Traffic Shaping**: khi failover sang model đắt tiền (`beta`), tự động điều chỉnh max tokens hoặc áp dụng semantic cache mạnh hơn để kiểm soát ngân sách.

---

## 8. Cache (Part G)

Số liệu từ `reports/cache/*.json` (`make cache`):

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8373 | 0.8424 | 0.000 | 0 | 0 | 0 | 1.386 | FAIL |
| cached (của bạn) | 0.9124 | 0.9645 | 0.652 | 157 | 0 | 3 | 0.915 | **PASS** |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| `similarity_threshold` | 0.90 | Ngăn chặn trả lời nhầm giữa các câu hỏi gần giống (như đổi size vs đổi màu, hoàn tiền vs thời gian hoàn). Tại 0.80 sinh ra 157 lỗi `other_question`; tại 0.90 đạt 0 lỗi mà vẫn giữ hit ratio 65.2%. |
| `ttl_s` | 60 | Khống chế số câu trả lời lỗi thời khi chính sách đổi không có webhook (t=480). Tại TTL 300s bị 95 câu outdated; tại TTL 60s chỉ còn 3 câu (đạt chuẩn ≤ 15). |
| `max_stale_s` | 300 | Đảm bảo tính năng stale-if-error cứu vãn sự cố sập toàn bộ provider trong 120s (t=240–360). Nếu đặt 0s, `faq_availability` tụt từ 96.45% xuống 87.33%. |

### Thí nghiệm:

1. **Ngưỡng similarity:**
   - `th = 0.80`: `cache_hit_ratio = 0.666`, `wrong_answers = 157` (toàn bộ do `other_question: 157`), `cost = 0.965`, `faq_avail = 0.8733`. Các câu hỏi khác nhau nhưng gần nghĩa (như "phí ship nội thành" vs "phí ship đi tỉnh", cosine ~0.86) bị cache nhầm lẫn.
   - `th = 0.90`: `cache_hit_ratio = 0.652`, `wrong_answers = 0`, `cost = 0.915`, `faq_avail = 0.9645`.
   - `th = 0.97`: `cache_hit_ratio = 0.581`, `wrong_answers = 0`, `cost = 0.964`, `faq_avail = 0.9640`. Ngưỡng quá khắt khe làm giảm tỷ lệ hit từ 65.2% xuống 58.1%, tăng chi phí gọi provider.
2. **TTL:**
   - `ttl_s = 30`: `outdated_answers = 2`, `cache_hit_ratio = 0.592`.
   - `ttl_s = 60`: `outdated_answers = 3`, `cache_hit_ratio = 0.652`.
   - `ttl_s = 300`: `outdated_answers = 95` (FAIL nặng vì trần là 15), `cache_hit_ratio = 0.731`.
   - **Giải thích:** Khi đối tác vận chuyển âm thầm thay đổi bảng phí ship ở t=480 mà không gửi webhook, hệ thống hoàn toàn không biết để chủ động invalidate cache. Do đó, chỉ có cơ chế hết hạn tự nhiên (TTL) mới buộc client gọi lại provider để cập nhật câu trả lời mới.
3. **Stale-if-error:**
   - Khi đặt `max_stale_s: 0`: `faq_availability` giảm mạnh từ `0.9645` xuống `0.8733` (FAIL tiêu chí ≥ 0.93) do trong 120s mọi provider sập (t=240–360), cache không được phép phục vụ dữ liệu cũ và phải GiveUp.

### Trả lời câu hỏi:
- **Vì sao câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?") không được cache, kể cả khi cache theo từng user?**
  Dữ liệu cá nhân (trạng thái đơn hàng, số tiền hoàn, điểm thưởng) mang tính biến động cao theo thời gian thực (ví dụ đơn vừa chuyển sang trạng thái "đang giao"). Cache câu trả lời cá nhân sẽ cung cấp thông tin sai lệch. Đặc biệt nguy hiểm hơn, nếu cache không cô lập tuyệt đối, câu hỏi của user A có thể trả lời lộ thông tin nhạy cảm của user B (vi phạm bảo mật và quyền riêng tư nghiêm trọng).
- **Trong lúc mọi provider sập, dashboard vẫn xanh nhờ cache. Bạn sẽ alert dựa trên số nào để on-call vẫn biết có sự cố?**
  Cần xây dựng alert dựa trên **Upstream Provider Error Rate / Circuit Breaker State** hoặc **Stale-Served Metric** (`stale_served > 0` hoặc `provider_call_failures > threshold`). Không thể chỉ nhìn vào `user_turn_availability` ở tầng biên, mà phải giám sát trực tiếp sức khỏe của các provider hạ tầng.
