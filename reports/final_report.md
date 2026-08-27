# Báo cáo Reliability - Day 10

**Họ và tên:** Đặng Nguyên Giáp — **MSSV:** 2A202601486

## 1. Tổng quan kiến trúc

Gateway định tuyến mọi request qua chuỗi: kiểm tra cache trước tiên → gọi provider có bảo vệ bởi
circuit breaker → chỉ trả về thông báo suy giảm (degraded) tĩnh khi cache miss và tất cả provider
đều không khả dụng.

```
User Request
    |
    v
[Gateway.complete(prompt)]
    |
    v
[Kiểm tra cache: ResponseCache / SharedRedisCache]
    |-- similarity >= threshold & không phải false-hit --> HIT --> trả text đã cache (route="cache_hit:<score>")
    |
    v MISS (hoặc cache đang tắt)
[Circuit Breaker: primary] --(CLOSED/HALF_OPEN, cho phép)--> FakeLLMProvider("primary").complete()
    |  thành công --> cache.set() --> trả kết quả (route="primary")
    |  lỗi / CircuitOpenError (đang OPEN, chưa hết timeout) --> record_failure(), thử provider tiếp theo
    v
[Circuit Breaker: backup] --(CLOSED/HALF_OPEN, cho phép)--> FakeLLMProvider("backup").complete()
    |  thành công --> cache.set() --> trả kết quả (route="fallback")
    |  lỗi / CircuitOpenError --> record_failure(), hết provider để thử
    v
[Fallback tĩnh]
    trả về "The service is temporarily degraded. Please try again soon." (route="static_fallback")
```

Máy trạng thái circuit breaker (mỗi provider có một instance độc lập, lưu trong `gateway.breakers`):

```
        failure_count >= failure_threshold
   CLOSED ---------------------------------> OPEN
     ^                                          |
     | success_count >= success_threshold       | đã trôi qua reset_timeout_seconds
     |                                          v
   HALF_OPEN <-------------------------------- (cho phép 1 request "thăm dò")
     |
     +-- probe thất bại --> OPEN (reason="probe_failure")
```

## 2. Cấu hình

| Tham số | Giá trị | Lý do |
|---|---:|---|
| failure_threshold | 3 | 3 lần lỗi liên tiếp là đủ tín hiệu để ngừng gọi vào một provider đang suy giảm, mà không phản ứng thái quá với một lỗi ngẫu nhiên đơn lẻ; đáp ứng đúng yêu cầu "không retry storm". |
| reset_timeout_seconds | 2 | Đủ ngắn để một provider đã hồi phục được thăm dò (probe) nhanh chóng (recovery_time_ms quan sát được ≈ 2.2s, gần sát mốc này), đủ dài để tránh mở lại mạch ngay lập tức khi provider vẫn còn đang lỗi. |
| success_threshold | 1 | Chỉ cần 1 lần probe thành công ở trạng thái HALF_OPEN là đủ để tin tưởng lại `FakeLLMProvider`, vì các lỗi ở đây là ngẫu nhiên độc lập theo từng lần gọi (memoryless), không phải kiểu lỗi cần "khởi động lại từ từ" (warm-up) thật sự. |
| cache TTL (ttl_seconds) | 300 | 5 phút cân bằng giữa rủi ro dữ liệu cũ (staleness) và tỉ lệ cache hit, phù hợp với workload dạng FAQ mà câu trả lời không đổi liên tục theo từng request. |
| similarity_threshold | 0.92 | Đã thử 0.85 trước — bị false-hit với các câu hỏi chỉ khác năm ("refund policy 2024" vs "2026") vì n-gram cosine cho điểm ~0.87 (chỉ khác token năm 4 chữ số); 0.92 vẫn giữ được các cặp câu gần giống thật sự, đồng thời guardrail chống false-hit theo năm/số vẫn đóng vai trò lớp bảo vệ thứ hai. |
| load_test.requests | 100 (mỗi scenario, tổng 400 trên 4 scenario) | Đủ lớn để ước lượng P95/P99 ổn định và để circuit breaker có cơ hội trải qua nhiều lần chuyển trạng thái open/half-open/closed trong một lần chạy. |

## 3. Định nghĩa SLO

Các giá trị dưới đây là kết quả **gộp (combined)** từ cả 4 kịch bản chaos (`reports/metrics.json`)
— việc gộp này cố tình bao gồm cả kịch bản được thiết kế để thất bại (`both_degraded`), nên các SLI
gộp trông tệ hơn bất kỳ kịch bản "khoẻ mạnh" đơn lẻ nào. Xem mục 7 để có bảng chi tiết theo từng
scenario — đó mới là bức tranh trung thực hơn về sức khoẻ hệ thống kiểu production.

| SLI | Mục tiêu SLO | Giá trị thực tế (gộp) | Đạt? |
|---|---|---:|---|
| Availability | >= 99% | 84.5% | ❌ Không (bị kéo xuống bởi `both_degraded`) |
| Latency P95 | < 2500 ms | 315.82 ms | ✅ Đạt |
| Fallback success rate | >= 95% | 52.67% | ❌ Không (bị kéo xuống bởi `both_degraded`, kịch bản không có fallback khoẻ mạnh) |
| Cache hit rate | >= 10% | 55.25% | ✅ Đạt |
| Recovery time | < 5000 ms | 2234.68 ms | ✅ Đạt |

## 4. Metrics

Trích từ `reports/metrics.json` (chạy gộp, cache đang bật, 400 request trên toàn bộ 4 scenario):

| Metric | Giá trị |
|---|---:|
| total_requests | 400 |
| availability | 0.845 |
| error_rate | 0.155 |
| latency_p50_ms | 267.33 |
| latency_p95_ms | 315.82 |
| latency_p99_ms | 320.37 |
| fallback_success_rate | 0.5267 |
| cache_hit_rate | 0.5525 |
| circuit_open_count | 12 |
| recovery_time_ms | 2234.68 |
| estimated_cost | 0.048626 |
| estimated_cost_saved | 0.221 |

## 5. So sánh có cache vs không cache

Cùng config và scenario, `cache.enabled: true` vs `cache.enabled: false`
(`configs/default.yaml` vs `configs/no_cache.yaml`, xuất ra `reports/metrics.json` vs
`reports/metrics_no_cache.json`):

| Metric | Không cache | Có cache | Chênh lệch |
|---|---:|---:|---|
| availability | 0.7375 | 0.845 | +0.1075 (cache tránh được các lời gọi lẽ ra đã đụng phải provider đang lỗi) |
| latency_p50_ms | 273.25 | 267.33 | -5.92 ms (cache hit trả về gần như 0ms, kéo median xuống) |
| latency_p95_ms | 316.15 | 315.82 | ~gần như không đổi (P95 bị chi phối bởi độ trễ provider khi cache miss) |
| circuit_open_count | 26 | 12 | -14 (ít lời gọi provider thật hơn nên ít lỗi tích luỹ vào breaker hơn) |
| estimated_cost | 0.132756 | 0.048626 | -0.084130 (rẻ hơn ≈63%) |
| cache_hit_rate | 0 | 0.5525 | +0.5525 |
| estimated_cost_saved | 0 | 0.221 | +0.221 |

**Nhận xét:** cache ở đây không chỉ tối ưu độ trễ — vì nó chặn request trước khi tới chuỗi
provider, nó còn giảm số lỗi mà circuit breaker phải "chứng kiến", từ đó giảm `circuit_open_count`
và tăng availability tổng thể.

## 6. Redis shared cache

- **Vì sao cache in-memory không đủ cho triển khai nhiều instance:** `ResponseCache` giữ
  `self._entries` là một list Python cục bộ trong 1 process. Nếu gateway được scale ra N replica
  đứng sau load balancer, mỗi replica có cache riêng rỗng — hit rate thực tế vẫn thấp và cùng một
  câu hỏi bị trả lời lại N lần bởi chuỗi provider (vốn tốn kém và dễ lỗi) thay vì chỉ 1 lần.
- **`SharedRedisCache` giải quyết vấn đề này như thế nào:** mỗi entry được ghi vào Redis dưới dạng
  hash (`{prefix}{md5(query)}` → `{query, response}`) kèm TTL qua `EXPIRE`. Bất kỳ instance gateway
  nào trỏ tới cùng `redis_url` đều có thể `HGET`/`SCAN` cùng các key đó, nên cache được "làm nóng"
  bởi 1 instance sẽ hiển thị ngay lập tức cho tất cả các instance khác — không cần thêm cơ chế đồng
  bộ nào khác.

### Bằng chứng chia sẻ trạng thái (shared state)

Hai object `SharedRedisCache` độc lập trong Python (mô phỏng 2 instance gateway) cùng trỏ tới một
Redis:

```
>>> cache_a = SharedRedisCache("redis://localhost:6379/0", ttl_seconds=300, similarity_threshold=0.92)
>>> cache_a.set("What is the capital of France?", "Paris is the capital of France.")
>>> cache_b = SharedRedisCache("redis://localhost:6379/0", ttl_seconds=300, similarity_threshold=0.92)
>>> cache_b.get("What is the capital of France?")
('Paris is the capital of France.', 1.0)
```

`cache_b` chưa từng gọi `.set()` — nó đọc được giá trị do `cache_a` ghi vào, chứng minh trạng thái
được chia sẻ qua Redis chứ không nằm trong bộ nhớ riêng của từng process.

### Kết quả Redis CLI

```bash
$ docker compose exec redis redis-cli KEYS "rl:cache:*"
rl:cache:cfc5e751cb29

$ docker compose exec redis redis-cli HGETALL rl:cache:cfc5e751cb29
query
What is the capital of France?
response
Paris is the capital of France.
```

### So sánh độ trễ: cache in-memory vs Redis (tuỳ chọn thêm)

| Metric | Cache in-memory | Cache Redis | Ghi chú |
|---|---:|---:|---|
| latency_p50_ms | 238.6 | chưa load-test riêng | Redis thêm một round-trip mạng nội bộ cho mỗi lần tra cache (~dưới 1ms trên localhost) so với việc quét list trong process; với quy mô workload này (100 entry) sự khác biệt là không đáng kể so với độ trễ provider mô phỏng (180-260ms). |
| latency_p95_ms | 318.44 | chưa load-test riêng | Lý do tương tự — bị chi phối bởi độ trễ provider, không phải backend cache. |

## 7. Các kịch bản chaos

| Scenario | Hành vi kỳ vọng | Hành vi quan sát được | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Primary lỗi 100%, toàn bộ traffic phải fallback sang backup | availability=0.98, fallback_success_rate=0.957, circuit_open_count=6, static_fallbacks=2 — backup hấp thụ gần như toàn bộ traffic, breaker của primary dao động open/half-open đúng như kỳ vọng | ✅ Pass |
| primary_flaky_50 | Primary lỗi chập chờn 50%, circuit phải dao động giữa gọi thành công và fallback | availability=1.0, fallback_success_rate=1.0, circuit_open_count=0 — với `failure_threshold=3` và tỉ lệ lỗi chỉ ~50%, xác suất 3 lần lỗi liên tiếp khá hiếm nên breaker hầu như không bị trip; backup đã bao phủ hết mọi lỗi xảy ra | ✅ Pass |
| all_healthy | Cả hai provider khoẻ mạnh (override fail rate xuống 2%), toàn bộ traffic qua primary, không circuit nào mở | availability=1.0, circuit_open_count=0, cache_hit_rate=0.67 | ✅ Pass |
| both_degraded (tự thêm) | Primary lỗi 70% / backup lỗi 60% — kỳ vọng có response fallback tĩnh và availability thấp khi cả hai breaker cùng mở | availability=0.0, static_fallbacks=100, circuit_open_count=2 — cả hai provider lỗi đủ nhiều để trip breaker và mọi request đều rơi vào thông báo fallback tĩnh | ✅ Pass |

`both_degraded` là kịch bản tự thêm ngoài 3 kịch bản gốc trong config khởi tạo
(`configs/default.yaml`), nhằm kiểm chứng nhánh "mọi thứ đều sập" và xác nhận gateway suy giảm một
cách "graceful" (trả thông báo tĩnh, không crash) thay vì văng exception.

## 8. Phân tích lỗi còn tồn tại

**Điểm yếu còn tồn tại:** các SLI ở mục 3 (chạy gộp) trông khá tệ (availability 84.5%, fallback
success 52.67%) chỉ vì `both_degraded` bị gộp chung vào cùng một số liệu tổng — một kịch bản hoạt
động tệ có thể "che khuất" ba kịch bản khoẻ mạnh khác trên một dashboard tổng quan duy nhất. Trong
hệ thống production thực tế, một kỹ sư trực (on-call) chỉ nhìn vào con số availability gộp có thể
phản ứng thái quá (hoặc phản ứng không đủ, nếu số liệu gộp tình cờ trông ổn) vì thiếu breakdown
theo scenario hoặc theo provider.

**Đề xuất khắc phục:** xuất metrics kèm nhãn (label) theo `scenario` (hoặc trong production là
`route`/`provider`) thay vì chỉ một số liệu tổng duy nhất — ví dụ dùng counter/histogram Prometheus
theo từng provider và trạng thái circuit, để có thể "cắt lát" (slice) availability theo provider và
theo phân khúc traffic. Đây chính là lý do `RunMetrics.scenarios` và `transition_log` theo từng
provider đã sẵn có trong codebase; bước tiếp theo là xuất chúng thành time series có nhãn thay vì
chỉ một dòng JSON/CSV được làm phẳng (flatten) duy nhất.

## 9. Bước tiếp theo

1. Thêm cost-aware routing (stretch goal): khi `estimated_cost` tích luỹ vượt một ngưỡng ngân sách,
   chuyển sang provider `backup` rẻ hơn hoặc chế độ chỉ dùng cache, thay vì luôn thử `primary`
   trước.
2. Lưu bộ đếm của circuit breaker vào Redis (`INCR`/`EXPIRE`) để trạng thái breaker được chia sẻ
   giữa các replica gateway giống như cache đang làm — hiện tại breaker của mỗi replica độc lập
   nhau, nên một replica vẫn có thể đang dồn dập gọi vào một provider đã chết trong khi replica
   khác đã kịp mở mạch (trip) rồi.
3. Xuất metrics dưới dạng Prometheus series có nhãn (theo scenario/provider) thay vì một file JSON
   được làm phẳng duy nhất, đúng như phân tích ở mục 8, để dashboard tổng quan không che khuất các
   lỗi cục bộ.
