# Báo cáo cuối: Day 10 Reliability

## 1. Tóm tắt kiến trúc

```text
Yêu cầu người dùng
    |
    v
[ReliabilityGateway]
    |
    +--> [ResponseCache / SharedRedisCache] -- hit --> trả response từ cache
    |
    v miss
[CircuitBreaker: primary] -- closed/half-open --> primary provider
    |
    v fail/open
[CircuitBreaker: backup] ---------------------> backup provider
    |
    v fail/open
[Static fallback response]
```

Gateway kiểm tra cache trước để giảm latency và chi phí mô phỏng. Nếu cache miss, mỗi lần gọi provider đều đi qua circuit breaker. Khi primary provider lỗi hoặc circuit đang mở, request được chuyển sang backup provider. Nếu tất cả provider đều lỗi, gateway trả về một response tĩnh ở trạng thái degraded thay vì để hệ thống crash.

## 2. Cấu hình

| Thiết lập | Giá trị | Lý do |
|---|---:|---|
| failure_threshold | 3 | Mở circuit sau nhiều lỗi liên tiếp, nhưng vẫn chịu được lỗi provider ngẫu nhiên. |
| reset_timeout_seconds | 2 | Cho provider thời gian hồi phục trước khi thử probe ở trạng thái half-open. |
| success_threshold | 1 | Một probe thành công là đủ cho lab local dùng fake provider. |
| cache backend | memory | Lần chạy mặc định dùng in-memory cache; Redis backend được kiểm thử riêng. |
| cache TTL | 300 | Giữ các query lặp lại đủ lâu để tái sử dụng, nhưng không giữ cache quá lâu. |
| similarity_threshold | 0.92 | Ngưỡng tương đồng tương đối chặt để giảm semantic false hit. |
| load_test requests | 100 mỗi scenario | 3 scenario tạo tổng cộng 300 request mô phỏng. |

## 3. Định nghĩa SLO

| SLI | Mục tiêu SLO | Giá trị thực tế | Đạt? |
|---|---|---:|---|
| Availability | >= 99% | 99.00% | Có |
| Latency P95 | < 2500 ms | 318.23 ms | Có |
| Fallback success rate | >= 95% | 95.59% | Có |
| Cache hit rate | >= 10% | 63.33% | Có |
| Recovery time | < 5000 ms | N/A | Không có recovery event trong cached run |

## 4. Metrics

| Metric | Giá trị |
|---|---:|
| total_requests | 300 |
| availability | 0.99 |
| error_rate | 0.01 |
| latency_p50_ms | 271.6 |
| latency_p95_ms | 318.23 |
| latency_p99_ms | 319.62 |
| fallback_success_rate | 0.9559 |
| cache_hit_rate | 0.6333 |
| circuit_open_count | 8 |
| recovery_time_ms | null |
| estimated_cost | 0.046704 |
| estimated_cost_saved | 0.19 |

## 5. So sánh khi bật và tắt cache

| Metric | Không dùng cache | Có dùng cache | Chênh lệch |
|---|---:|---:|---:|
| availability | 0.9733 | 0.99 | +0.0167 |
| latency_p50_ms | 275.48 | 271.6 | -3.88 ms |
| latency_p95_ms | 314.86 | 318.23 | +3.37 ms |
| estimated_cost | 0.122692 | 0.046704 | -0.075988 |
| cache_hit_rate | 0.0 | 0.6333 | +0.6333 |
| circuit_open_count | 22 | 8 | -14 |

Cache làm giảm đáng kể số lần gọi provider và chi phí mô phỏng. Chênh lệch P95 latency không quá lớn vì cached responses được trộn với provider calls, và latency của provider vẫn chi phối các request không hit cache.

## 6. Redis shared cache

In-memory cache không đủ cho triển khai production nhiều instance, vì mỗi gateway process có cache riêng. Nếu người dùng được route sang instance khác, instance đó sẽ không thấy response đã được tính ở instance trước.

`SharedRedisCache` giải quyết vấn đề này bằng cách lưu cache entry trong Redis hash và đặt TTL. Nhiều gateway instance dùng chung Redis URL và prefix có thể chia sẻ response với nhau.

Bằng chứng shared state:

```text
instance_2_value=shared report response
score=1.0
keys=['rl:evidence:853c11f9f307']
```

Kết quả test Redis:

```text
pytest tests/test_redis_cache.py -q
6 passed
```

## 7. Chaos scenarios

| Scenario | Hành vi kỳ vọng | Hành vi quan sát được | Kết quả |
|---|---|---|---|
| primary_timeout_100 | Primary luôn lỗi; traffic fallback sang backup hoặc static fallback. | Scenario hoàn thành thành công. | Pass |
| primary_flaky_50 | Circuit mở khi primary lỗi lặp lại; backup xử lý nhiều request. | Scenario hoàn thành thành công. | Pass |
| all_healthy | Primary xử lý phần lớn request không hit cache. | Scenario hoàn thành thành công. | Pass |

## 8. Phân tích điểm yếu còn lại

Một điểm yếu còn lại là trạng thái circuit breaker vẫn nằm trong từng process. Trong triển khai nhiều instance thực tế, một gateway instance có thể đã mở circuit, nhưng instance khác vẫn tiếp tục gửi traffic tới provider đang lỗi.

Trước khi đưa vào production, tôi sẽ lưu counter và trạng thái circuit breaker trong Redis bằng atomic operations, hoặc dùng một health signal chung theo từng provider. Tôi cũng sẽ thêm cảnh báo SLO theo từng route và ghi lại fallback reason theo provider để debug incident dễ hơn.

## 9. Kiểm tra cuối

```text
pytest -q
35 passed, 7 xpassed
```

Artifacts đã tạo:

- `reports/metrics.json`
- `reports/metrics_no_cache.json`
- `reports/metrics.csv`
- `reports/final_report.md`
