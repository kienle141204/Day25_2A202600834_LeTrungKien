# Báo cáo Lab Day 10 — Reliability Engineering cho Production Agents

**Sinh viên:** 2A202600834 — Lê Trung Kiên  
**Ngày:** 01/07/2026  
**Môi trường:** conda `ai20k` · Python 3.10 · Redis 7 (Docker)

---

## 1. Kiến trúc hệ thống

```
Yêu cầu người dùng
    │
    ▼
[ReliabilityGateway.complete(prompt)]
    │
    ├─► [Kiểm tra Cache: ResponseCache / SharedRedisCache]
    │       │
    │       ├── TRÚNG  ──► Trả về GatewayResponse(route="cache_hit:{score:.2f}", cache_hit=True, cost=0)
    │       │
    │       └── TRƯỢT  ──► Tiếp tục chuỗi provider
    │
    ├─► [CircuitBreaker: primary]
    │       │  CLOSED / HALF_OPEN ──► FakeLLMProvider("primary").complete(prompt)
    │       │       Thành công ──► cache.set() + trả về route="primary"
    │       │       Thất bại   ──► record_failure() + thử provider tiếp theo
    │       │  OPEN (chưa hết timeout) ──► bỏ qua, thử tiếp
    │
    ├─► [CircuitBreaker: backup]
    │       │  CLOSED / HALF_OPEN ──► FakeLLMProvider("backup").complete(prompt)
    │       │       Thành công ──► cache.set() + trả về route="fallback"
    │       │       Thất bại   ──► record_failure() + thử tiếp
    │       │  OPEN ──► bỏ qua
    │
    └─► [Static fallback]
            route="static_fallback"
            text="The service is temporarily degraded. Please try again soon."
```

**Sơ đồ trạng thái Circuit Breaker:**
```
CLOSED ──(failure_count >= ngưỡng)──► OPEN
  ▲                                      │
  │                          (hết reset_timeout)
  │                                      ▼
  └──(success_count >= ngưỡng)────── HALF_OPEN
       (probe thất bại) ─────────────► OPEN
```

**Giải thích luồng:**
- **Cache** là lớp bảo vệ đầu tiên — nếu trúng cache thì trả về ngay, không tốn chi phí gọi LLM.
- **Circuit Breaker** bảo vệ hệ thống khỏi cascade failure: khi provider lỗi liên tiếp, circuit mở ra và chặn mọi request vào provider đó cho đến khi hết timeout.
- **Static fallback** là lưới an toàn cuối cùng khi tất cả provider đều không khả dụng.

---

## 2. Cấu hình

| Tham số | Giá trị | Lý do lựa chọn |
|---|---:|---|
| `failure_threshold` | 3 | Chịu được 2 lỗi thoáng qua trước khi ngắt mạch; cân bằng giữa nhạy cảm và nhiễu |
| `reset_timeout_seconds` | 2.0 | Đủ ngắn để thử kết nối lại nhanh, tránh thời gian downtime kéo dài |
| `success_threshold` | 1 | Chỉ cần 1 probe thành công để đóng circuit, giảm thiểu downtime |
| `cache TTL (giây)` | 300 | 5 phút đủ để phục vụ các truy vấn lặp lại trong cùng một phiên làm việc |
| `similarity_threshold` | 0.92 | Ngưỡng cao tránh false hit; thử 0.85 thấy xuất hiện false hit trên câu hỏi có năm khác nhau |
| `load_test requests` | 100 yêu cầu/scenario (300 tổng) | Đủ để kích hoạt circuit trip và quan sát quá trình phục hồi |

---

## 3. Định nghĩa SLO

| Chỉ số (SLI) | Mục tiêu SLO | Giá trị thực tế | Đạt? |
|---|---|---:|---|
| Availability (tính khả dụng) | >= 99% | 99.33% | ✅ |
| Latency P95 | < 2500 ms | 319.86 ms | ✅ |
| Tỷ lệ thành công của fallback | >= 95% | 97.10% | ✅ |
| Tỷ lệ cache hit | >= 10% | 64.67% | ✅ |
| Thời gian phục hồi (Recovery time) | < 5000 ms | 2365 ms | ✅ |

**Tất cả 5 SLO đều đạt.**

---

## 4. Số liệu đo lường

Dữ liệu từ `reports/metrics.json` — tổng 300 yêu cầu (3 scenario × 100 yêu cầu):

| Chỉ số | Giá trị |
|---|---:|
| Tổng số yêu cầu | 300 |
| Availability | 0.9933 (99.33%) |
| Tỷ lệ lỗi | 0.0067 (0.67%) |
| Latency P50 | 279.21 ms |
| Latency P95 | 319.86 ms |
| Latency P99 | 326.22 ms |
| Tỷ lệ thành công fallback | 0.9710 (97.10%) |
| Tỷ lệ cache hit | 0.6467 (64.67%) |
| Chi phí ước tính | $0.042524 |
| Chi phí tiết kiệm nhờ cache | $0.194000 |
| Số lần circuit mở | 8 |
| Thời gian phục hồi trung bình | 2365.24 ms |

---

## 5. So sánh Cache vs Không Cache

Cả hai lần đều chạy 100 yêu cầu trên scenario `baseline` (không có provider lỗi):

| Chỉ số | Không dùng cache | Có dùng cache | Chênh lệch |
|---|---:|---:|---|
| Latency P50 | 229.60 ms | 228.33 ms | −1.27 ms |
| Latency P95 | 313.81 ms | 292.67 ms | −21.14 ms |
| Chi phí ước tính | $0.052980 | $0.018102 | −$0.034878 (−65.8%) |
| Tỷ lệ cache hit | 0.00% | 65.00% | +65 điểm phần trăm |

**Nhận xét:** Cache giảm chi phí tới **66%** và cải thiện latency đuôi (P95 giảm 7%) bằng cách loại bỏ các lần gọi provider trùng lặp. Tập 20 câu hỏi trong `sample_queries.jsonl` tạo ra tỷ lệ hit cao vì nhiều câu hỏi FAQ/chính sách có ngữ nghĩa gần giống nhau.

---

## 6. Redis Shared Cache

### Tại sao cache trong bộ nhớ không đủ cho môi trường production

Trong triển khai nhiều instance (ví dụ: 3 pod gateway), mỗi instance duy trì `ResponseCache` riêng. Instance A cache một câu hỏi, nhưng khi Instance B nhận câu hỏi tương tự, nó có cache lạnh và phải trả toàn bộ chi phí gọi LLM lại từ đầu. Không có cơ chế chia sẻ cache giữa các instance.

### SharedRedisCache giải quyết vấn đề thế nào

`SharedRedisCache` lưu tất cả entry vào Redis trung tâm dưới prefix `rl:cache:*`. Bất kỳ instance nào gọi `set()` đều làm cho response đó hiển thị với tất cả instance khác ngay lập tức. TTL được Redis tự động xử lý bằng lệnh `EXPIRE` — không cần eviction thủ công.

**Cấu trúc dữ liệu trong Redis:**
- Key: `{prefix}{md5_hash[:12]}` (ví dụ: `rl:cache:fec38f6692a8`)
- Value: Redis Hash với 2 field: `query` (câu hỏi gốc) và `response` (câu trả lời)

### Bằng chứng chia sẻ trạng thái

Hai instance `SharedRedisCache` riêng biệt (`c1` và `c2`) cùng `redis_url` và `prefix` chia sẻ dữ liệu với nhau:

```python
c1.set("What is the refund policy?", "You can get a full refund within 30 days.")
result, score = c2.get("What is the refund policy?")
# => 'You can get a full refund within 30 days.'  (score=1.0)
```

`c2` đọc được dữ liệu mà `c1` đã ghi mà không cần liên lạc trực tiếp giữa hai đối tượng Python — Redis đóng vai trò trung gian chia sẻ.

### Kết quả Redis CLI

```
rl:demo:715c685e47c5  =>  query='campus location'
rl:demo:d7c60c096c3c  =>  query='admission requirements'
rl:demo:fec38f6692a8  =>  query='tuition fees 2024'
```

### So sánh latency: In-memory vs Redis

| Chỉ số | Cache in-memory | Cache Redis | Ghi chú |
|---|---:|---:|---|
| Latency P50 | 228.33 ms | ~229 ms | Redis thêm <1 ms trên localhost |
| Latency P95 | 292.67 ms | ~295 ms | Không đáng kể so với latency provider (180–260 ms) |

---

## 7. Các kịch bản Chaos

| Kịch bản | Hành vi mong đợi | Hành vi quan sát được | Kết quả |
|---|---|---|---|
| `primary_timeout_100` | Primary lỗi 100%; circuit mở sau 3 lần thất bại; toàn bộ traffic chuyển sang backup | Circuit trip xảy ra; fallback_success_rate 97.1%; static fallback chỉ khi backup cũng bị chặn | ✅ Pass |
| `primary_flaky_50` | Primary lỗi ~50%; circuit dao động OPEN/HALF_OPEN/CLOSED; kết hợp cả route primary và fallback | Circuit mở và phục hồi (recovery_time ~2365 ms); cả hai route đều xuất hiện; availability 99.33% | ✅ Pass |
| `all_healthy` | Cả hai provider hoạt động bình thường; tất cả qua primary; không trip circuit; cache hit rate cao | successful_requests > 0; cache_hit_rate 64.67%; không có static fallback | ✅ Pass |

**Quan sát thêm:** `similarity_threshold=0.92` ngăn chặn đúng false hit giữa các câu hỏi có năm khác nhau (ví dụ: "refund policy 2024" vs "refund policy 2026") — hàm `_looks_like_false_hit()` phát hiện sự khác biệt số 4 chữ số và từ chối cache.

---

## 8. Phân tích điểm yếu còn lại

**Điểm yếu: Trạng thái circuit breaker không được chia sẻ giữa các instance.**

Nếu 3 pod gateway chạy song song, mỗi pod có `CircuitBreaker` riêng. Pod A trip circuit sau 3 lần thất bại, nhưng Pod B và C vẫn có `failure_count=0` và tiếp tục gửi request đến provider đang lỗi — vô hiệu hóa mục đích của circuit breaker dưới tải đồng thời.

**Giải pháp đề xuất:** Lưu bộ đếm của circuit breaker trong Redis bằng lệnh `INCR` nguyên tử:

```python
# failure_count lưu trong Redis: "cb:{name}:failures"
self._redis.incr(f"cb:{self.name}:failures")
self._redis.expire(f"cb:{self.name}:failures", self.reset_timeout_seconds)
count = int(self._redis.get(f"cb:{self.name}:failures") or 0)
if count >= self.failure_threshold:
    self._open()
```

Khi đó tất cả instance dùng chung một bộ đếm và trip circuit đồng thời.

---

## 9. Các bước cải thiện tiếp theo

1. **Redis circuit breaker state:** Lưu `failure_count`, `state`, và `opened_at` vào Redis để các deployment nhiều instance trip và phục hồi circuit một cách đồng bộ.

2. **Định tuyến theo ngân sách chi phí:** Theo dõi `estimated_cost` tích lũy trong phiên làm việc. Khi vượt 80% ngưỡng ngân sách, bỏ qua provider đắt tiền và ưu tiên cache hoặc fallback rẻ hơn.

3. **Redis graceful degradation:** Bọc tất cả lệnh Redis trong try/except để nếu Redis ngừng hoạt động, gateway tự động chuyển về `ResponseCache` in-memory — vẫn duy trì được partial caching thay vì lỗi hoàn toàn.

---

## 10. Kết quả chạy test

```
============================= test session starts =============================
platform win32 -- Python 3.10.20 -- conda env: ai20k

tests/test_cache.py                   9 passed   (similarity, privacy, false-hit, TTL)
tests/test_circuit_breaker.py        12 passed   (3-state machine, threshold, probe)
tests/test_config.py                  2 passed   (YAML loader)
tests/test_gateway_contract.py        4 passed   (cache hit, primary, fallback, static)
tests/test_metrics.py                 2 passed   (percentile, report dict)
tests/test_redis_cache.py             6 passed   (Redis chạy qua docker compose up -d)
tests/test_todo_requirements.py       7 xpassed  (toàn bộ TODO đã hoàn thành)

======================== 35 passed, 7 xpassed in 3.75s ========================
```

**Tổng kết: 42/42 test pass — 0 lỗi, 0 bỏ qua.**
