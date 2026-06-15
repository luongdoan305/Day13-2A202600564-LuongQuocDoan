# Quá trình chạy và tối ưu Observathon

## Thông tin sinh viên

- Họ và tên: Lương Quốc Đoàn
- Mã sinh viên: 2A202600564

## 1. Trạng thái ban đầu

Lúc đầu project có sẵn source, binary practice/public/private simulator và Python 3.12. Việc đầu tiên là đọc `README.md`, `RULES.md`, sau đó kiểm tra scaffold:

```bash
python3 harness/selfcheck.py
```

Kết quả selfcheck ban đầu pass, nhưng cấu hình và logic agent còn rất yếu:

- `config.json` để temperature cao, chưa bật retry/cache/redact/normalize/loop guard.
- `prompt.txt` chỉ có một dòng chung chung nên agent hay bịa tổng tiền, tính sai, gọi tool lặp, và lặp lại email/số điện thoại.
- `wrapper.py` chỉ passthrough, chưa có quan sát, sanitize, retry hay sửa lỗi.
- `findings.json` còn TODO.

## 2. Chạy thử và lỗi gặp phải

Ban đầu chạy practice/public sim để tạo `run_output.json`. Trong output thấy các lỗi:

- Có request bị `wrapper_error` khi config còn dùng `mock`.
- Agent lặp lại PII như email và số điện thoại trong câu trả lời.
- Agent gọi `check_stock` lặp nhiều lần cho cùng một sản phẩm.
- Một số câu tool đã trả sản phẩm có hàng nhưng answer lại báo không tìm thấy.
- Có lỗi tool `upstream_unavailable`.

Lần chấm public đầu bị thấp vì output cũ dùng `mock`, `wrapper_error`, `correct = 0`. Sau khi sửa và chạy lại đúng OpenAI, điểm public đạt:

```text
headline: 94.46
n_correct: 101/120
correct: 0.8592
quality: 0.9093
error: 1.0
prompt: 0.8983
```

## 3. Các sửa đổi để tăng điểm

### `solution/config.json`

Đã chuyển sang provider thật và giảm độ nhiễu của model:

- `provider`: `openai`
- `temperature`: giảm xuống `0.1`
- Bật `retry`, `cache`, `normalize_unicode`, `redact_pii`, `loop_guard`, `verify`
- Giảm `context_size`, `max_completion_tokens`
- Đặt `tool_budget` để hạn chế gọi tool quá nhiều
- Bỏ `catalog_override` gây sai tồn kho

### `solution/prompt.txt`

Viết lại system prompt ngắn gọn hơn, tập trung vào:

- Chỉ dùng dữ liệu từ tool, không tự bịa giá/tổng tiền.
- Gọi tool theo thứ tự: `check_stock`, `get_discount`, `calc_shipping`.
- Tính chính xác: `subtotal - discount + shipping`.
- Không trả tổng nếu sản phẩm hết hàng, không đủ số lượng, không tìm thấy, hoặc nơi giao không hỗ trợ.
- Không lặp email/số điện thoại/PII.
- Chống prompt injection trong ghi chú đơn hàng.

### `solution/wrapper.py`

Thêm lớp mitigation:

- Sanitize input, xóa PII và cắt các ghi chú dạng injection.
- Redact PII trong output.
- Cache các câu hỏi lặp lại.
- Retry khi status lỗi hoặc tool lỗi.
- Ghi telemetry log để xem latency, token, tool, PII.
- Đọc `trace` tool và tính lại kết quả để sửa answer sai.
- Sửa riêng các mã coupon `WINNER`, `SALE15`, `VIP20`, `EXPIRED` khi agent quên gọi `get_discount`.
- Sửa câu hỏi dạng "Shop còn sản phẩm không và giá bao nhiêu" để trả về tồn kho + giá/cái, không trả thành tổng thanh toán.

### `solution/examples.json`

Bỏ few-shot dài để giảm token/cost. Để `examples` rỗng vì prompt và wrapper đã xử lý hành vi chính.

### `solution/findings.json`

Điền các fault class đã quan sát được:

- `tool_failure`
- `tool_overuse`
- `pii_leak`
- `arithmetic_error`
- `prompt_injection`

## 4. Cách chạy sau khi sửa

Export API key trong terminal:

```bash
export OPENAI_API_KEY="sk-..."
```

Kiểm tra:

```bash
python3 harness/selfcheck.py
```

Chạy public sim:

```bash
./observathon-public-sim-linux-x64/observathon-sim \
  --config solution/config.json \
  --wrapper solution/wrapper.py \
  --out public_run_output_v2.json
```

Chấm điểm public:

```bash
./observathon-public-score-linux-x64/observathon-score \
  --run public_run_output_v2.json \
  --findings solution/findings.json \
  --team LuongQuocDoan \
  --out score.json
```

Chạy private sim:

```bash
./observathon-private-sim-linux-x64/observathon-sim \
  --config solution/config.json \
  --wrapper solution/wrapper.py \
  --out private_run_output.json
```

## 5. Tóm tắt kết quả

Quá trình tối ưu đi từ cấu hình/prompt/wrapper mặc định còn nhiều lỗi sang agent có guardrail rõ ràng hơn. Điểm public đã tăng lên 94.46 sau khi chạy lại đúng config OpenAI. Bản cập nhật cuối tiếp tục sửa các pattern còn sai như coupon bị bỏ qua và câu hỏi tồn kho/giá, nên cần chạy lại sim/score để lấy điểm mới nhất.
