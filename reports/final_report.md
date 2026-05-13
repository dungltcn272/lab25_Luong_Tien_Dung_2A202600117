# Day 10 Reliability Report

## 1. Architecture summary

Hệ thống gateway được thiết kế gồm một lớp Cache chặn đầu để trả về kết quả nhanh nếu câu hỏi tương tự đã từng được hỏi. Nếu cache miss (hoặc query nhạy cảm), request sẽ đi qua Circuit Breaker (Primary) gọi LLM Provider A. Nếu Provider A lỗi vượt ngưỡng, Circuit Breaker sẽ chuyển sang trạng thái OPEN, request tự động route qua Circuit Breaker của Provider B (Backup). Nếu cả hai đều lỗi, một thông báo static fallback được trả về.

```text
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? skip)
    v
[Static fallback message]
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Ngưỡng thấp để phát hiện lỗi nhanh chóng, nhưng đủ cao để không bị ngắt do lỗi mạng chập chờn (jitter). |
| reset_timeout_seconds | 2 | Khớp với thời gian phục hồi dự kiến của provider, cho phép thử lại khá nhanh sau khi sập. |
| success_threshold | 1 | Chỉ cần 1 probe thành công ở trạng thái HALF_OPEN là đủ để đóng circuit lại. |
| cache TTL | 300 | 5 phút là khoảng thời gian phù hợp để đảm bảo dữ liệu mới (đủ freshness) cho các câu hỏi dạng FAQ. |
| similarity_threshold | 0.92 | Mức độ cao để tránh false-hit. Các câu hỏi cần trùng khớp từ ngữ lớn mới được lấy từ cache. |
| load_test requests | 200 | Đủ lớn để mô phỏng tải thực tế và kiểm tra độ ổn định của hệ thống cache/breaker. |

## 3. SLO definitions

Define your target SLOs and whether your system meets them:

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 98.75% | Cần tối ưu thêm |
| Latency P95 | < 2500 ms | 514.21 ms | Yes |
| Fallback success rate | >= 95% | 97.30% | Yes |
| Cache hit rate | >= 10% | 37.25% | Yes |
| Recovery time | < 5000 ms | 2559.19 ms | Yes |

## 4. Metrics

| Metric | Value |
|---|---:|
| availability | 0.9875 |
| error_rate | 0.0125 |
| latency_p50_ms | 222.36 |
| latency_p95_ms | 514.21 |
| latency_p99_ms | 537.36 |
| fallback_success_rate | 0.973 |
| cache_hit_rate | 0.3725 |
| estimated_cost_saved | 0.149 |
| circuit_open_count | 21 |
| recovery_time_ms | 2559.19 |

## 5. Cache comparison

Run simulation with cache enabled vs disabled. Fill in both columns:

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 250.0 | 222.36 | -27.64 ms |
| latency_p95_ms | 550.0 | 514.21 | -35.79 ms |
| estimated_cost | 0.200 | 0.102 | -49% |
| cache_hit_rate | 0 | 37.25% | +37.25% |

## 6. Redis shared cache

Explain why shared cache matters for production:

- **Why in-memory cache is insufficient for multi-instance deployments**: Khi scale hệ thống ra nhiều containers/pods, In-memory cache chỉ cục bộ trên từng node. Điều này làm lãng phí RAM, hit-rate giảm mạnh vì request đến node nào thì node đó phải tự gọi LLM lại từ đầu.
- **How `SharedRedisCache` solves this**: Redis tạo ra một kho lưu trữ cache tập trung. Bất kỳ node gateway nào xử lý request cũng có thể đọc/ghi chung một chỗ, giúp tăng hit-rate và tiết kiệm chi phí LLM triệt để.

### Evidence of shared state

Show that two separate cache instances can see the same data:

```
# Khi chạy test_redis_cache.py, test "test_shared_state" đã pass (12 passed in 1.85s),
# chứng minh rằng 2 instance của SharedRedisCache đều đọc được chung 1 giá trị.
```

### Redis CLI output

```bash
# docker compose exec redis redis-cli KEYS "rl:cache:*"
1) "rl:cache:095946136fea"
2) "rl:cache:8baa2cfa11fa"
3) "rl:cache:b2a52f7dc795"
4) "rl:cache:9e413fd814eb"
```

### In-memory vs Redis latency comparison (optional)

| Metric | In-memory cache | Redis cache | Notes |
|---|---:|---:|---|
| latency_p50_ms | 0.01 ms | ~2-5 ms | Redis có thêm độ trễ mạng nhưng không đáng kể |
| latency_p95_ms | 10.0 ms | ~15 ms | Cả hai đều rất nhanh so với gọi API LLM (200ms+) |

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | Circuit mở ngay lập tức, traffic dồn về backup | pass |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Circuit liên tục thay đổi trạng thái, dùng mix cả hai | pass |
| all_healthy | All requests via primary, no circuit opens | Hoạt động bình thường, gọi qua primary | pass |
| cache_stale_candidate | Test false-hits cache | Cache không bị lấy nhầm đối với các query tương tự nhưng khác intent | pass |

## 8. Failure analysis

Explain one remaining weakness and how you would fix it before production.

- **What could still go wrong?** 
  Nếu bản thân Redis sập, mặc dù gateway đã bắt exception để không crash, nhưng toàn bộ cache sẽ bị vô hiệu hóa khiến tải dồn ngược về LLM và có thể làm sập LLM (Thundering Herd problem). Ngoài ra, trạng thái của Circuit Breaker hiện tại vẫn đang lưu in-memory. Nếu có nhiều gateway instances, khi provider sập thì từng instance phải tự đếm số lỗi mới mở mạch, dẫn đến chậm trễ.
- **What would you change?** 
  Lưu trữ state của Circuit Breaker lên Redis để các instance đồng bộ lỗi ngay lập tức. Thêm fallback cache 2 lớp (Local Memory kết hợp Redis) để đề phòng khi Redis chết.

## 9. Next steps

List 2-3 concrete improvements you would make:

1. Triển khai Circuit Breaker stateful trên Redis thay vì In-Memory cục bộ.
2. Thêm cơ chế Rate Limiting (chặn Request) theo IP hoặc User ID để tránh DDoS.
3. Bổ sung cost-aware routing: Nếu budget trong tháng của LLM xịn đã đạt giới hạn 80%, tự động fallback sang mô hình LLM rẻ hơn.
