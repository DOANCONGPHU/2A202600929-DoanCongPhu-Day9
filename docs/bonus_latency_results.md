# Bài tập cộng điểm: Giảm latency Stage 5

## Cách đo

- Chạy đầy đủ 5 service của Stage 5.
- Dùng cùng câu hỏi mặc định trong `test_client.py`.
- Chạy E2E 3 lần cho mỗi phiên bản.
- Đo từ lúc client bắt đầu kết nối tới khi nhận được final response.
- Model sử dụng khi đo: `gpt-4o-mini`.

`test_client.py` đã được bổ sung dòng `TOTAL LATENCY` để có thể demo lại:

```bash
uv run python test_client.py
```

## Kết quả baseline

| Lần chạy | Latency |
|---|---:|
| 1 | 70.77 giây |
| 2 | 56.07 giây |
| 3 | 50.36 giây |
| **Trung bình** | **59.07 giây** |
| **Median** | **56.07 giây** |

## Phương án giảm latency

Hệ thống ban đầu có các LLM call tuần tự không cần thiết:

1. Law Agent dùng LLM chỉ để quyết định hai boolean `needs_tax` và
   `needs_compliance`.
2. Customer Agent dùng LLM để chọn tool, mặc dù system prompt yêu cầu luôn
   delegate mọi câu hỏi pháp lý, sau đó dùng thêm một LLM call để diễn đạt lại
   kết quả của Law Agent.

Các thay đổi đã áp dụng:

- Thay LLM routing trong `law_agent/graph.py` bằng deterministic keyword routing.
- Thêm fast path trong `customer_agent/agent_executor.py`: Customer Agent
  discover và delegate trực tiếp tới Law Agent.
- Giữ nguyên Tax và Compliance chạy song song.
- Giữ nguyên các LLM call tạo nội dung chuyên môn tại Law, Tax, Compliance và
  bước aggregate.

Luồng sau tối ưu:

```text
Client -> Customer Agent -> Registry -> Law Agent
                                  -> analyze_law
                                  -> local keyword routing
                                  -> Tax + Compliance in parallel
                                  -> aggregate
       <- final response <---------
```

## Kết quả sau tối ưu

| Lần chạy | Latency |
|---|---:|
| 1 | 46.31 giây |
| 2 | 39.97 giây |
| 3 | 44.13 giây |
| **Trung bình** | **43.47 giây** |
| **Median** | **44.13 giây** |

## So sánh

| Chỉ số | Trước | Sau | Cải thiện |
|---|---:|---:|---:|
| Trung bình | 59.07 giây | 43.47 giây | **15.60 giây / 26.4%** |
| Median | 56.07 giây | 44.13 giây | **11.94 giây / 21.3%** |

Latency vẫn dao động do thời gian phản hồi từ LLM provider và độ dài output,
nhưng cả trung bình và median đều giảm rõ ràng.

## Trade-off

- Keyword routing nhanh và ổn định nhưng có thể bỏ sót cách diễn đạt không chứa
  từ khóa đã định nghĩa.
- Customer fast path phù hợp vì service này chỉ là entry point cho câu hỏi pháp
  lý. Nếu sau này Customer Agent hỗ trợ nhiều loại yêu cầu như billing hoặc
  support, cần thêm router nhẹ trước khi delegate.
