# Lab 25B — Report

**Họ tên / nhóm:** Chung Văn Duy - 2A202602854  **Commit:** a3f9ce7078c14d183ed1e95d311a2df59bc86855  **Điểm tự động (`make grade`):** 88.0 / 88

> Mọi nhận định phải có số. Dán số từ `reports/gameday/*.json`, `reports/workflows.json`, `reports/slo.json`.

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
| G4 | yours | 0.9995 | 5.614 | 1.015 | 0.002 | 2.066 | 0.0 | **PASS** |
| G5 | naive | 0.3699 | 5.359 | 17.030 | 16.030 | 2.124 | None | FAIL |
| G5 | yours | 0.8179 | 5.477 | 1.003 | 0.002 | 2.504 | None | **PASS** |
| G6 | naive | 0.5170 | 4.836 | 12.344 | 11.344 | 2.009 | 0.0 | FAIL |
| G6 | yours | 1.0000 | 5.969 | 1.051 | 0.017 | 2.227 | 0.0 | **PASS** |

## 2. Cấu hình client và lý do

| Tham số | Giá trị | Vì sao (dựa trên băng nào, số liệu nào) |
|---|---:|---|
| attempt_timeout_s | 8.0 | Băng G6: Provider `alpha-eu` bị treo hoàn toàn (hang). Nếu timeout 30s thì request ngậm hết deadline của user dẫn đến 109 lỗi deadline và availability tụt xuống 94.88%. Đặt 8.0s giúp cắt lỗ sớm và đủ thời gian failover sang `beta`. |
| max_attempts_per_provider | 2 | Băng G2 & G5: Cho phép tối đa 1 lần retry (tổng 2 attempts) trên cùng provider để cứu transient errors mà không làm bùng nổ retry storm (amplification ở G2 chỉ 1.041 so với 9.904 của naive). |
| backoff_base_s / cap_s | 0.25 / 4.0 | Băng G1 & G3: Giãn cách retry theo cấp số nhân có full jitter để tránh thundering herd khi provider bắt đầu hồi phục, cap 4.0s để không vượt deadline 30s. |
| retry_budget_ratio | 0.1 | Băng G3: Yêu cầu `retry_ratio <= 0.10`. Với ratio = 0.1, lượng retry bị khống chế cứng dưới 10% lưu lượng, đạt thực tế 0.034 (naive là 0.164). |
| breaker.window_size / min_calls | 20 / 10 | Băng G2 & G4: Cửa sổ 20 kết quả và tối thiểu 10 mẫu để tránh false trip vào ban đêm hoặc khi chỉ có vài request ngẫu nhiên thất bại, đồng thời phản ứng kịp trong vòng 3-5 giây khi sự cố xảy ra. |
| breaker.failure_rate_threshold | 0.5 | Băng G2: Khi alpha mất GPU dẫn đến 529 và timeout, tỷ lệ lỗi nhanh chóng vượt 50%, breaker lập tức chuyển OPEN để chuyển lưu lượng sang `alpha-eu`. |
| breaker.slow_call_s | 5.0 | Băng G4: Alpha vẫn trả lời không lỗi nhưng chậm thêm 5s (latency ~6.2s). Ngưỡng 5.0s giúp nhận diện cuộc gọi chậm như một tín hiệu suy thoái nghiêm trọng. |
| breaker.open_duration_s | 10.0 | Băng G2: Sau khi mở OPEN 10s, breaker chuyển sang HALF-OPEN thăm dò. 10s là khoảng thời gian đủ để hàng đợi của provider giải tỏa tải nghẽn. |
| breaker.half_open_max_probes | 2 | Băng G2: Chỉ cho phép 2 request thăm dò đồng thời ở HALF-OPEN để tránh hiện tượng thundering herd làm sập lại provider vừa mới hồi phục. |

Sơ đồ quyết định của `on_error` (ASCII hoặc bảng: loại lỗi → ghi breaker? → retry? → failover? → cooldown?):

```
                       ┌─────────────────────────┐
                       │   Nhận ProviderError    │
                       └────────────┬────────────┘
                                    │ classify(err)
                                    ▼
       ┌────────────────────────────┼───────────────────────────┐
       │ CLIENT_ERROR (400)         │ RATE_LIMITED (429)        │ QUOTA_EXHAUSTED (429)
       ▼                            ▼                           ▼
┌──────────────┐             ┌──────────────┐            ┌──────────────┐
│ Breaker:     │             │ Breaker:     │            │ Breaker:     │
│ release()    │             │ release()    │            │ release()    │
│              │             │              │            │              │
│ Cooldown:    │             │ Cooldown:    │            │ Cooldown:    │
│ Không        │             │ now +        │            │ now + 300s   │
│              │             │ retry_after  │            │ (dài hạn)    │
│ Action:      │             │              │            │              │
│ GiveUp       │             │ Action:      │            │ Action:      │
│ ("client_    │             │ Failover ngay│            │ Failover ngay│
│   error")    │             │ (spillover)  │            │              │
└──────────────┘             └──────────────┘            └──────────────┘
                                    │
       ┌────────────────────────────┴───────────────────────────┐
       │ OVERLOADED (503/529) / SERVER_ERROR (500) / TIMEOUT (0)
       ▼
┌───────────────────────────────────────────────────────────────┐
│ Breaker: on_failure(permit, now)                              │
│                                                               │
│ Điều kiện Retry cùng Provider:                                │
│   tries < max_attempts_per_provider (2)                       │
│   AND budget.try_acquire_retry(now)                           │
│   AND (now + backoff_delay + min_time_left <= deadline)       │
│                                                               │
│   ├─ THỎA MÃN  ──► Send(same_provider, delay=backoff_delay)   │
│   │                                                           │
│   └─ KHÔNG ĐỦ  ──► idx += 1, tries = 0,                       │
│                    _route(call, now, delay=0.0) [Failover]    │
└───────────────────────────────────────────────────────────────┘
```

## 3. Ba thí nghiệm (mỗi cái 3–5 câu + số)

1. **Retry budget là lớp phòng thủ cuối.** Khi breaker bị vô hiệu hóa (`min_calls: 1000000000`) và `max_attempts_per_provider: 5`, trên băng G2 khi giữ budget ta đạt amplification 1.383, retry_ratio 0.058, availability 99.87% và recovery 0.0s. Nhưng khi tắt budget, amplification tăng vọt lên 1.667 (vượt trần 1.6), retry_ratio lên 0.486, availability tụt xuống 96.91% và mất tới 12.0s để phục hồi. Trên băng G3 (lỗi 30% chưa đủ kích hoạt breaker 50%), tắt budget khiến retry_ratio tăng từ 0.068 lên 0.162 (vượt ngưỡng 0.10). Khi breaker hoạt động đúng, budget vẫn đóng vai trò sinh tử vì nó là chốt chặn tài nguyên toàn cục (global token bucket), ngăn chặn tình trạng hàng ngàn client cùng retry trong các kịch bản lỗi phân tán dưới ngưỡng kích hoạt breaker.
2. **`attempt_timeout_s: 30`.** Trên băng G6, timeout 30s khiến availability sụt giảm từ 99.92% xuống 94.88% (FAIL tiêu chí $\ge 95\%$), ghi nhận 109 lượt chết vì cạn kiệt deadline (`give_up_reasons={'deadline': 109}`). Tuy nhiên, `latency_p95_s` lại giảm nghịch lý từ 6.003s xuống 5.304s; đây là minh chứng điển hình của *survivorship bias* vì các cuộc gọi bị treo 30s đều thất bại nên bị loại khỏi tập tính p95. Trên băng G4 đối chứng, latency p95 hầu như không đổi (5.614s so với 5.683s) do alpha chỉ bị trễ 5s (tổng latency ~6.2s) nên không chạm trần timeout 8s hay 30s. Do đó, timeout "đúng" không được dựa vào deadline của user mà phải căn cứ vào p99 lúc bình thường cộng biên độ đủ để client kịp failover sang provider khác.
3. **Bỏ lọc tool calling khỏi `ladder()`.** Trên băng G6, việc bỏ lọc tool calling khiến hệ thống ghi nhận **268 lỗi `incompatible_routes`** và kéo availability tụt từ 99.92% xuống còn 88.76%. Đứng dưới góc độ một Product Manager: "Khi nhà cung cấp chính gặp sự cố, hệ thống đã điều hướng mù quáng các yêu cầu nghiệp vụ phức tạp sang mô hình giá rẻ C-mini không hỗ trợ gọi công cụ. Kết quả là 268 khách hàng bị gián đoạn giao dịch hoàn tiền, model sinh văn bản không thể xử lý tiếp, dẫn tới tỷ lệ hoàn tất nghiệp vụ sụt giảm hơn 11% và gây khiếu nại dịch vụ nghiêm trọng."

## 4. Workflow hoàn tiền

| Executor | completion | crashes | duplicate refunds | duplicate emails | lost refunds | overpaid | wasted LLM / crash |
|---|---:|---:|---:|---:|---:|---:|---:|
| naive | 1.0 | 81 | 128 | 81 | 0 | 1,496,700 ₫ | 1.123 |
| yours | 1.0 | 66 | 0 | 0 | 0 | 0 ₫ | 0.303 |

**Trả lời:**
- Chỉ có idempotency key là **chưa đủ** vì khi worker bị crash, worker mới khởi động lại sẽ không biết phiên làm việc đã chạy đến bước nào nếu không có checkpoint, dẫn đến việc phải tính toán lại từ đầu các bước LLM tốn kém (`wasted_llm_per_crash` cao).
- Chỉ có checkpoint cũng **chưa đủ** vì nếu worker bị crash ngay sau khi gọi cổng thanh toán thành công nhưng chưa kịp lưu checkpoint, worker mới sẽ gọi lại API thanh toán; nếu không có idempotency key, cổng thanh toán sẽ trừ tiền lần thứ hai gây duplicate refund.
- Nếu ghi checkpoint **trước** khi gọi payment: Trong trường hợp payment bị lỗi thực sự hoặc worker chết ngay trước khi lệnh thanh toán tới được ngân hàng, checkpoint đã ghi nhận thành công giả mạo. Khi worker mới hồi phục, nó thấy checkpoint tồn tại nên bỏ qua bước hoàn tiền, dẫn đến hậu quả nghiêm trọng nhất: **khách hàng bị mất tiền hoàn (lost refunds)**.

## 5. Alerting & error budget

| Sự cố | Mong đợi | Naive alerter | Burn-rate alerter | Độ trễ phát hiện |
|---|---|---|---|---:|
| checkout_bug | page | page | page | 27 min |
| provider_outage | page | page | page | 4 min |
| deploy_blip | none | page (FALSE) | none | N/A |
| slow_leak | ticket | none (MISSED) | ticket | 1337 min |
| degraded_evening | page | page | page | 291 min |
| night_spike | none | page (FALSE) | none | N/A |
| small_bump | none | page (FALSE) | none | N/A |

**Error budget 30 ngày:** Đã tiêu **42.95%**, còn lại **57.05%**. Quyết định: **KHÔNG đóng băng release** (`freeze_releases: False`) vì ngân sách lỗi còn hơn 57%.
Sự cố `small_bump` (20 phút ở 12% lỗi) **không** nên đánh thức on-call lúc 11 giờ trưa vì tổng mức tiêu hao error budget của nó chỉ khoảng 0.28% ngân sách tháng. Đây là một sự cố ngắn tự kết thúc, không đe dọa SLO tổng thể; việc page sẽ gây mệt mỏi cảnh báo (alert fatigue) và làm gián đoạn công việc của kỹ sư mà không mang lại giá trị vận hành.

## 6. Postmortem — Băng G2 với client naive (blameless)

*Dùng `make plot` (hình `reports/gameday/G2_overload_storm.png`) và CSV `reports/gameday/naive_G2_overload_storm_timeseries.csv`.*

**Tóm tắt (3 câu):**
Vào lúc $t=120$, provider `alpha` bị suy giảm 75% năng lực xử lý GPU trong vòng 3 phút, dẫn đến hàng đợi bị ứ đọng và phát sinh lỗi quá tải 529. Phiên bản `NaiveClient` với cơ chế retry 3 tầng không jitter và không có Circuit Breaker đã khuếch đại lưu lượng lên gấp hơn 30 lần, tạo ra một cơn bão retry storm lên tới 254 attempts/s. Hậu quả là hệ thống rơi vào trạng thái lỗi siêu bền (metastable failure), tiếp tục sập thêm 155 giây sau khi GPU của provider đã hồi phục hoàn toàn, khiến 54.4% lượt người dùng thất bại.

**Ảnh hưởng:**
Có **1,285 / 2,362 lượt người dùng thất bại** (tỷ lệ thành công chỉ đạt 45.60%). Sự cố kéo dài tổng cộng 335 giây ảo (từ $t=120$ đến $t=455$). Sự cố này tiêu tốn tới **1,285 / 11.81 = 10,880% error budget của tháng** (vượt hơn 108 lần toàn bộ ngân sách lỗi cho phép của cả tháng).

**Timeline** (giây ảo — lấy từ CSV `naive_G2_overload_storm_timeseries.csv`):

| t | Sự kiện | Bằng chứng (số) |
|---:|---|---|
| 119 | Hệ thống vận hành bình thường trước sự cố | `calls_ok = 7`, `attempts_alpha = 9`, latency ổn định |
| 120 | Alpha mất 75% năng lực GPU, hàng đợi bắt đầu nghẽn | `calls_ok = 7`, `attempts_alpha = 5`, bắt đầu xuất hiện 529 |
| 122 | Tỷ lệ thành công chạm đáy do hàng đợi vượt 10s | `calls_ok = 0`, `calls_done = 0`, xuất hiện client timeout |
| 136 | Retry storm 3 tầng bùng nổ dữ dội | `attempts_alpha = 53`, `calls_ok = 3`, khuếch đại gấp 6 lần |
| 143 | Tải retry dội vào alpha đạt đỉnh lịch sử | `attempts_alpha = 254 req/s`, `calls_ok = 0`, 100% thất bại |
| 300 | Alpha phục hồi 100% năng lực GPU | `attempts_alpha = 115`, `calls_ok = 0`, hệ thống vẫn tê liệt |
| 350 | Alpha tiếp tục nghẽn vì lượng retry cũ tích tụ | `attempts_alpha = 17`, `calls_ok = 0`, hàng đợi toàn request chết |
| 453 | Hàng đợi request chết cạn kiệt, hệ thống bắt đầu phục hồi | `calls_ok = 4`, `calls_done = 8`, `attempts_alpha = 7` |
| 459 | Hệ thống phục hồi hoàn toàn sau 155s kể từ khi GPU hồi phục | `calls_ok = 10`, `calls_done = 14`, `attempts_alpha = 16` |

**Trigger vs root cause:**
- **Trigger:** Provider `alpha` bị mất 75% số lượng GPU trong 3 phút (từ $t=120$ đến $t=300$).
- **Root cause:** Kiến trúc client thiếu khả năng chịu lỗi: 3 tầng retry độc lập nhân tải lên 36 lần, retry với chu kỳ cố định không jitter, không có Retry Budget và không có Circuit Breaker.
- **Cơ chế Metastable Failure:** Sau $t=300$, provider đã có lại 100% GPU nhưng hệ thống vẫn sập vì hàng đợi FIFO chứa đầy các request cũ mà client đã timeout từ bỏ. Provider vẫn phải tiêu tốn 100% GPU để xử lý các request chết này. Trong khi đó, các tầng client tiếp tục dội thêm retry mới vào hàng đợi, tạo thành một vòng lặp tự duy trì (self-sustaining feedback loop) giữ tải luôn vượt quá năng lực xử lý của server.

**Điều gì giúp / điều gì làm tệ hơn:**
- *Làm tệ hơn:* Client timeout nhưng server vẫn tiếp tục xử lý công việc vô ích; delay cố định khiến hàng ngàn retry ập đến cùng một thời điểm; không có cơ chế failover sang `alpha-eu` hay `beta`.
- *Giúp ích:* Simulator có ngưỡng giới hạn hàng đợi 10s (shed load bằng lỗi 529), nếu không hệ thống sẽ bị treo vĩnh viễn không bao giờ phục hồi.

**Action items** (cụ thể, có owner, đo được):

| # | Hành động | Loại (prevent / detect / mitigate) | Owner | Cách kiểm chứng |
|---|---|---|---|---|
| 1 | Triển khai ResilientClient tích hợp Circuit Breaker sliding-window và Retry Budget 10% | Prevent | Core Agent Team | Chạy CI Game Day băng G2 đạt `amplification <= 1.1` và `recovery_s <= 5s` |
| 2 | Thiết lập hệ thống giám sát Multi-window Burn-rate Alerting phát hiện quá tải trong 4 phút | Detect | SRE Team | Kiểm tra trên dữ liệu SLI 30 ngày: phát hiện G2 trong 4 phút, 0 false page |
| 3 | Cấu hình Fallback Ladder tự động failover sang `alpha-eu` và `beta` khi alpha gặp lỗi | Mitigate | Infrastructure Team | Availability trên băng G2 đạt $\ge 99.5\%$ khi alpha mất 75% năng lực |

## 7. Một điểm yếu còn lại

**Điểm yếu:** Chi phí vận hành gia tăng đột biến khi failover sang `beta` trong sự cố kéo dài. Deployment `beta` có chi phí cao hơn 40% (1.4 so với 1.0) và độ trễ trung bình chậm hơn (~1.8s so với 1.2s). Khi `alpha` gặp sự cố diện rộng, toàn bộ traffic đổ dồn sang `beta` không chỉ làm tăng chi phí mà còn dễ gây quá tải dây chuyền (cascading failure) sang tenant của `beta` (như trong băng G6 khi beta bị bóp rate limit xuống 9 rps).
**Hướng khắc phục:**
1. Áp dụng cơ chế **load shedding có phân cấp ưu tiên**: Khi phải chạy trên `beta`, chỉ ưu tiên phục vụ các khách hàng VIP hoặc các giao dịch hoàn tiền có giá trị lớn; các yêu cầu thông thường sẽ nhận câu trả lời suy thoái tĩnh.
2. Tối ưu hóa định tuyến lai (Hybrid Routing): Tận dụng mô hình siêu rẻ `gamma-mini` (chi phí 0.2) cho bước `chat` không cần tool calling, giúp bù trừ chi phí gia tăng của bước `tool` trên `beta`.

## 8. Cache (Part G)

Số từ `reports/cache/*.json` (`make cache`).

| Client | availability | faq_availability | cache_hit_ratio | stale_served | wrong_answers | outdated_answers | cost/turn | PASS? |
|---|---:|---:|---:|---:|---:|---:|---:|:---:|
| resilient (không cache) | 0.8390 | 0.8453 | 0.000 | 0 | 0 | 0 | 1.384 | FAIL |
| cached (của bạn) | 0.9117 | 0.9634 | 0.653 | 159 | 0 | 3 | 0.915 | **PASS** |

| Tham số `cache:` | Giá trị | Vì sao (số liệu nào) |
|---|---:|---|
| similarity_threshold | 0.90 | Với ngưỡng starter 0.80, hệ thống bị 157 lỗi `other_question` (trả nhầm câu hỏi gần nghĩa). Nâng lên 0.90 đưa `wrong_answers` về đúng 0 mà vẫn duy trì hit ratio cao 65.3% (vượt xa tiêu chuẩn 50%). |
| ttl_s | 60 | Tại $t=480$, bảng phí ship thay đổi ngầm không có webhook. Nếu để TTL starter 600s, có tới 95 câu trả lời lỗi thời bị phục vụ. Hạ TTL xuống 60s giúp `outdated_answers` chỉ còn 3 (đạt chuẩn $\le 15$). |
| max_stale_s | 600 | Tại $t=240..360$, toàn bộ provider sập trong 2 phút. Nếu để `max_stale_s: 0`, `faq_availability` chỉ đạt 87.09%. Cấu hình 600s giúp stale-if-error cứu 159 lượt, nâng `faq_availability` lên 96.34% (đạt chuẩn $\ge 93\%$). |

Thí nghiệm (mỗi cái 2–4 câu + số):

1. **Ngưỡng.** Chạy C1 với `similarity_threshold = 0.80` cho hit ratio 66.6%, nhưng sinh ra **157 lỗi `wrong_answers`** (`other_question: 157`), cost/turn là 0.965. Với ngưỡng chuẩn **0.90**, `wrong_answers = 0`, hit ratio 65.3% và cost giảm còn 0.915. Khi tăng lên **0.97**, `wrong_answers = 0` nhưng hit ratio tụt xuống 58.1% làm cost/turn tăng lên 0.964 do phải gọi provider nhiều hơn. Ở ngưỡng 0.80, câu hỏi "Thời gian hoàn tiền mất bao lâu?" bị trả lời nhầm bằng câu trả lời của "Chính sách hoàn tiền gồm những gì?".
2. **TTL.** Chạy C1 với `ttl_s = 30s / 60s / 300s` ghi nhận số lượng `outdated_answers` lần lượt là **2 / 3 / 95**. Khi nhà vận chuyển đổi giá cước ở $t=480$ mà không gửi webhook, cache không nhận được tín hiệu để chủ động xóa dữ liệu. Do đó, webhook hoàn toàn bất lực trước các cập nhật ngầm, và TTL ngắn (60s) là tuyến phòng thủ duy nhất buộc cache phải định kỳ làm mới từ provider.
3. **Stale-if-error.** Chạy với `max_stale_s: 0` khiến `stale_served = 0` và kéo `faq_availability` tụt xuống **87.09%** (vi phạm tiêu chí $\ge 93\%$). Khi bật `max_stale_s: 600s`, hệ thống đã phục vụ thành công **159 câu trả lời cũ**, giúp `faq_availability` đạt tới **96.34%** trong giai đoạn mọi provider đều sập.

**Trả lời:**
- Câu hỏi cá nhân ("Đơn #48213 của tôi đang ở đâu?", "Điểm tích lũy của tôi là bao nhiêu?") **không** được cache kể cả theo từng `user_id` vì trạng thái đơn hàng và điểm thưởng thay đổi liên tục theo thời gian thực (đơn chuyển từ đang giao sang đã giao, điểm bị trừ khi đổi quà). Cache câu trả lời cá nhân sẽ dẫn đến việc trả lời sai trạng thái mới nhất của đơn hàng và tiềm ẩn nguy cơ rò rỉ dữ liệu nhạy cảm nếu có lỗi va chạm key giữa các phiên làm việc.
- Trong lúc mọi provider sập nhưng dashboard vẫn xanh nhờ phục vụ câu trả lời cache, hệ thống cần kích hoạt cảnh báo dựa trên các chỉ số sau để on-call lập tức phát hiện:
  1. **Tỷ lệ phục vụ dữ liệu cũ (`stale_served_ratio` hoặc `stale_rate`):** Khi tỷ lệ stale vượt quá 5% lưu lượng.
  2. **Tỷ lệ lỗi backend provider thô (`upstream_provider_5xx_rate`):** Đo đạc trực tiếp tại gateway tầng network trước cache, phát hiện provider trả về 503/529.
  3. **Lưu lượng cache miss gặp lỗi (`cache_miss_failure_rate`):** Các câu hỏi cá nhân hoặc câu hỏi mới bị GiveUp, phản ánh chính xác sự cố của hạ tầng bên dưới.
