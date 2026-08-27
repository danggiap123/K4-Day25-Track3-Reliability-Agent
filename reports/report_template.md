# Báo cáo Reliability - Day 10

**Họ và tên:** Đặng Nguyên Giáp — **MSSV:** 2A202601486

## 1. Tổng quan kiến trúc

Mô tả gateway, circuit breaker, chuỗi fallback, và các lớp cache của bạn.
Kèm theo một sơ đồ đơn giản (dạng text/ASCII cũng được):

```
User Request
    |
    v
[Gateway] ---> [Kiểm tra cache] ---> HIT? trả kết quả đã cache
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? bỏ qua)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? bỏ qua)
    v
[Thông báo fallback tĩnh]
```

## 2. Cấu hình

| Tham số | Giá trị | Lý do |
|---|---:|---|
| failure_threshold | TODO | TODO |
| reset_timeout_seconds | TODO | TODO |
| success_threshold | TODO | TODO |
| cache TTL | TODO | TODO |
| similarity_threshold | TODO | TODO |
| load_test requests | TODO | TODO |

## 3. Định nghĩa SLO

Định nghĩa mục tiêu SLO và cho biết hệ thống của bạn có đạt được không:

| SLI | Mục tiêu SLO | Giá trị thực tế | Đạt? |
|---|---|---:|---|
| Availability | >= 99% | TODO | TODO |
| Latency P95 | < 2500 ms | TODO | TODO |
| Fallback success rate | >= 95% | TODO | TODO |
| Cache hit rate | >= 10% | TODO | TODO |
| Recovery time | < 5000 ms | TODO | TODO |

## 4. Metrics

Dán hoặc tóm tắt nội dung từ `reports/metrics.json`.

| Metric | Giá trị |
|---|---:|
| availability | TODO |
| error_rate | TODO |
| latency_p50_ms | TODO |
| latency_p95_ms | TODO |
| latency_p99_ms | TODO |
| fallback_success_rate | TODO |
| cache_hit_rate | TODO |
| estimated_cost_saved | TODO |
| circuit_open_count | TODO |
| recovery_time_ms | TODO |

## 5. So sánh có cache vs không cache

Chạy simulation với cache bật và tắt. Điền đầy đủ cả hai cột:

| Metric | Không cache | Có cache | Chênh lệch |
|---|---:|---:|---|
| latency_p50_ms | TODO | TODO | TODO |
| latency_p95_ms | TODO | TODO | TODO |
| estimated_cost | TODO | TODO | TODO |
| cache_hit_rate | 0 | TODO | TODO |

## 6. Redis shared cache

Giải thích vì sao cache dùng chung quan trọng với production:

- Vì sao cache in-memory không đủ cho triển khai nhiều instance: TODO
- `SharedRedisCache` giải quyết vấn đề này như thế nào: TODO

### Bằng chứng chia sẻ trạng thái (shared state)

Chứng minh hai instance cache riêng biệt có thể thấy cùng một dữ liệu:

```
# Dán kết quả test hoặc kết quả chạy script cho thấy trạng thái được chia sẻ
TODO
```

### Kết quả Redis CLI

```bash
# docker compose exec redis redis-cli KEYS "rl:cache:*"
TODO
```

### So sánh độ trễ: cache in-memory vs Redis (tuỳ chọn thêm)

| Metric | Cache in-memory | Cache Redis | Ghi chú |
|---|---:|---:|---|
| latency_p50_ms | TODO | TODO | |
| latency_p95_ms | TODO | TODO | |

## 7. Các kịch bản chaos

| Scenario | Hành vi kỳ vọng | Hành vi quan sát được | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Toàn bộ traffic fallback sang backup, circuit mở | TODO | TODO |
| primary_flaky_50 | Circuit dao động, xen kẽ giữa primary và fallback | TODO | TODO |
| all_healthy | Toàn bộ request qua primary, không circuit nào mở | TODO | TODO |
| (kịch bản tự thêm của bạn) | TODO | TODO | TODO |

## 8. Phân tích lỗi còn tồn tại

Giải thích một điểm yếu còn tồn tại và cách bạn sẽ khắc phục trước khi đưa vào production.

- Điều gì vẫn có thể sai sót?
- Bạn sẽ thay đổi gì? (ví dụ: lưu trạng thái circuit vào Redis, giới hạn tốc độ theo từng user, SLO về chất lượng)

## 9. Bước tiếp theo

Liệt kê 2-3 cải tiến cụ thể bạn sẽ thực hiện:

1. TODO
2. TODO
3. TODO
