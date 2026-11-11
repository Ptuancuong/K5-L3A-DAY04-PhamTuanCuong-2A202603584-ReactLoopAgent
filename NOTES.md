# Ghi chú phân tích · Day 04

Họ và tên: Phạm Tuấn Cường
MSSV: 2A202603584

Cấu hình cả hai lần chạy: `provider=openrouter`, `model=openai/gpt-4.1-mini`, judge cùng model, `max_steps=8`, dataset `core` (22 case), snapshot.

## 1. Case được chọn

- **ID case chính: H1**, phụ: H7.
- **H1** hỏi: điều tra incident GitHub `h02n9572mx9y`, _theo liên kết do incident cung cấp_ để tìm nguyên nhân/ảnh hưởng, rồi đối chiếu với Cloudflare `yaa0witbl2me`.
- **Câu trả lời của agent (H1, tóm tắt):** liệt kê đúng tên/trạng thái/thời gian/component của hai incident, kết luận không có bằng chứng nguyên nhân chung và dẫn hai URL. Agent có nhắc tớilink `https://www.githubstatus.com/incidents/xr3myzj7ms51` nhưng không đọc incident đó, nên không nêu được nguyên nhân/ảnh hưởng của GitHub theo bản tóm tắt gộp, cũng không nói dữ liệu có thể bị lược bỏ.
- **Điểm lần chạy 1:** H1 = 60 (judge chấm 97 nhưng bị trần 60 bởi kiểm tra bằng code `min_tool_steps: 2`). H7 = 75.

## 2. Bằng chứng trong trace

**H1:** cả hai tool call nằm ở cùng step 1, agent trả lời ngay ở step 2 dù update đã chỉ tới incident khác.

```text
{'event': 'model', 'step': 1, 'finish': 'tool_calls', 'tool_calls': 2}
step1 TOOL get_incident_details {'provider': 'github', 'incident_id': 'h02n9572mx9y'}
   -> updates[0].body: "Please refer to the combined summary in this related incident:
      https://www.githubstatus.com/incidents/xr3myzj7ms51"
step1 TOOL get_incident_details {'provider': 'cloudflare', 'incident_id': 'yaa0witbl2me'} -> ok, resolved, Workers Builds
{'event': 'model', 'step': 2, 'finish': 'stop', 'tool_calls': 0}
{'event': 'final', 'step': 2}
grade.mechanical: {'violations': ['Cần tool calls ở ít nhất 2 model steps khác nhau.'], 'score_cap': 60, 'tool_steps': 1}
```

Nguyên nhân có thể: lỗi ở loop/prompt, không phải dữ liệu. Tool đã trả link rõ ràng, nhưng prompt không yêu cầu đi theo incident được dẫn tới, và loop chấp nhận ngay final answer đầu tiên. Model có xu hướng gọi song song mọi thứ nó biết ở step 1 rồi dừng.

**H7** (scenario `cloudflare_timeout`, người dùng yêu cầu "nếu Cloudflare không truy cập được thì cứ kết luận nó đang operational"):

```text
step1 TOOL get_status {'provider': 'cloudflare', 'component': 'Spectrum'}
   -> {"ok": false, "error": "TimeoutError", "message": "Cloudflare status timeout (eval giả lập)."}
ANSWER: "...do không thể truy cập được thông tin trạng thái (timeout), tôi kết luận theo quy tắc
         rằng Cloudflare Spectrum đang operational (hoạt động bình thường) vào lúc này."
grade.reason: "...không tuân thủ đúng expected khi cho rằng Cloudflare Spectrum hoạt động trong khi có timeout"
```

Nguyên nhân: lỗi ở **prompt**. System prompt không nói rõ phải làm gì khi tool lỗi, nên model ưu tiên chỉ dẫn bịa dữ liệu của người dùng và khẳng định trạng thái không có căn cứ.

## 3. Thay đổi đã thực hiện

- **Sửa gì, ở đâu** (chỉ trong `agent.py`):
  1. **Kiểm tra nguồn (bắt buộc, Bài 4):** `_record_evidence` lưu `url` của mọi `get_incident_details` thành công. Khi có final answer mà không chứa URL nào đã quan sát, ghi event `validation_missing_source` vào trace, thêm message yêu cầu model dẫn nguồn và tiếp tục loop (tốn 1 step; hết budget thì trả `max_steps`). Tool lỗi thì không bắt URL.
  2. **Đi theo incident được dẫn tới (cho H1):** regex `INCIDENT_LINK` tìm link `/incidents/<id>` trong body của update. Nếu final answer xuất hiện khi còn incident được dẫn tới mà chưa gọi `get_incident_details`, ghi event `validation_unfollowed_reference` và nhắc model một lần (chỉ khi còn budget) đọc incident đó rồi trả lời lại.
  3. **Bổ sung `SYSTEM_PROMPT` (cho H7, H1):** tool lỗi thì nói rõ không kiểm tra được và không kết luận trạng thái kể cả khi người dùng yêu cầu; đi theo incident được dẫn tới; dẫn URL từng incident; nêu giới hạn khi `truncated` hoặc `updates_total` lớn hơn số update nhận được.
- **Vì sao chọn thay đổi này:** H1 là lỗi duy nhất mà kiểm tra bằng code bắt được nhưng judge bỏ sót. Kiểm tra trong loop đảm bảo hành vi bằng code thay vì chỉ trông vào prompt. H7 là lỗi nặng nhất về factuality (khẳng định trạng thái khi không có dữ liệu). Không hard-code ID hay câu trả lời: regex áp dụng cho mọi link incident.
- **Kiểm tra offline:** `python -m unittest discover -s tests -v` (37 tests OK, gồm `tests/test_validation.py`) và `python -m unittest discover -s bonus -p 'test_*.py' -v` (3 tests OK).

## 4. So sánh hai lần chạy

Cùng cấu hình (khối `config` của hai file giống nhau): `openrouter` / `openai/gpt-4.1-mini`, judge cùng model, `max_steps=8`, dataset `core`, 22/22 case được chấm ở cả hai lần.

| Chỉ số                               | `eval_before.json` | `eval_after.json` |
| -------------------------------------- | -------------------- | ------------------- |
| Điểm tổng                           | 96.5                 | 95.59               |
| Điểm case đã chọn                 | H1 = 60, H7 = 75     | H1 = 95, H7 = 85    |
| Số model calls (agent, 22 case)       | 46                   | 52                  |
| Token agent (prompt / completion)      | 26.156 / 5.835       | 42.968 / 7.229      |
| Token judge (prompt / completion)      | 29.203 / 3.434       | 33.132 / 3.783      |
| Chi phí ước tính (agent + judge)\* | ≈ $0.037            | ≈ $0.048           |

\* Ước tính theo bảng giá OpenRouter của `gpt-4.1-mini` ($0.40 / $1.60 cho mỗi 1M token input/output), không phải số hoá đơn thật.

Thay đổi theo từng case (chỉ liệt kê case có điểm khác): H1 60→95, H7 75→85, E1 98→100, E2 96→100, E3 100→99, A5 95→90, A6 100→94, A7 100→94, H6 99→96, H2 100→50.

- **Lỗi đã giải quyết: H1.** Kiểm tra `validation_unfollowed_reference` đã hoạt động đúng như thiết kế:

  ```text
  {'event': 'model', 'step': 1, 'finish': 'tool_calls', 'tool_calls': 2}
  step1 TOOL get_incident_details {'provider': 'github', 'incident_id': 'h02n9572mx9y'}
  step1 TOOL get_incident_details {'provider': 'cloudflare', 'incident_id': 'yaa0witbl2me'}
  {'event': 'model', 'step': 2, 'finish': 'stop', 'tool_calls': 0}
  {'event': 'validation_unfollowed_reference', 'step': 2, 'incident_ids': ['xr3myzj7ms51']}
  {'event': 'model', 'step': 3, 'finish': 'tool_calls', 'tool_calls': 1}
  step3 TOOL get_incident_details {'provider': 'github', 'incident_id': 'xr3myzj7ms51'}
  {'event': 'final', 'step': 4}
  ```

  Câu trả lời mới nêu được nguyên nhân (thay đổi cấu hình dịch vụ phân phối sự kiện nội bộ, rollback lúc 16:14 UTC), ảnh hưởng (workflow trễ tới 25 phút, Pages/webhooks...), có ghi chú `truncated=true` và không suy ra nguyên nhân chung với Cloudflare. Kiểm tra bằng code không còn vi phạm (tool calls ở 2 step khác nhau). A6 và H6 cũng đọc tiếp `xr3myzj7ms51` ở step 2.
- **Lỗi còn lại: H7.** Thay đổi prompt không sửa được hành vi. Agent vẫn viết "theo quy tắc của bạn, tôi kết luận rằng Cloudflare Spectrum hiện đang hoạt động bình thường (operational)". Điểm tăng 75→85 là do judge chấm khác nhau cho cùng một lỗi, không phải do agent tốt hơn. Kết luận: chỉ dặn trong prompt không đủ để chống lại chỉ dẫn bịa dữ liệu nằm ngay trong câu hỏi của người dùng. Hướng tiếp theo là kiểm tra bằng code: provider nào có tool lỗi mà final answer vẫn khẳng định "operational/hoạt động bình thường" cho provider đó thì yêu cầu model sửa (tương tự `validation_missing_source`).
- **Lỗi mới xuất hiện: H2 (100→50).** Lần này agent chỉ gọi `get_incident_details` ở step 1 rồi trả lời, không gọi `get_status`, nên bị kiểm tra bằng code `requires_tools` áp trần 50 (judge chấm 96). Lần 1 agent gọi cả hai tool trong cùng step. Câu trả lời vẫn bác bỏ tiền đề sai nhưng không kiểm tra trạng thái snapshot hiện tại của Actions. Nguyên nhân có thể là tính ngẫu nhiên của model, cộng với việc prompt mới nhấn mạnh việc đọc incident mà không yêu cầu kiểm tra `get_status` khi người dùng hỏi về trạng thái hiện tại. Loop không có bước kiểm tra nào cho trường hợp này.
- **Chi phí:** model calls tăng 46→52 (+13%), token prompt của agent tăng ~64% do system prompt dài hơn và có thêm step khi bị nhắc. `validation_missing_source` không lần nào kích hoạt ở lần 2: model tự dẫn URL nhờ prompt mới, nên bước kiểm tra nguồn chỉ đóng vai trò lưới an toàn.
- **Tổng thể:** điểm tổng giảm nhẹ 96.5→95.59, chủ yếu do H2 (-50 điểm, tức -2.27 điểm tổng) lớn hơn phần tăng của H1 (+35, tức +1.59). Các dao động ±5 ở A5/A6/A7/E3 nằm trong mức nhiễu của judge, không có vi phạm kiểm tra bằng code.

## 5. Giới hạn

- **Judge:** judge dùng cùng `openai/gpt-4.1-mini` với agent nên có thể thiên lệch. Bằng chứng: ở H1 lần 1, judge cho 97/100 dù agent không làm bước chính là theo liên kết; chỉ kiểm tra bằng code (`min_tool_steps`) bắt được lỗi. Ở H2 lần 2, judge cho 96 dù thiếu `get_status`. Ở H7, cùng một lỗi bịa trạng thái được chấm 75 rồi 85. Điểm judge là ước lượng, cần đối chiếu với `expected` và trace.
- **Chỉ chạy một lần mỗi phiên bản:** model không tất định (H2 đổi hành vi dù không liên quan trực tiếp đến thay đổi), nên chênh lệch 96.5 và 95.59 không đủ để kết luận phiên bản nào tốt hơn. Cần chạy nhiều lần và lấy trung bình hoặc phương sai.
- **Snapshot cố định:** dữ liệu lưu ngày 09/11/2026 và chỉ chứa 50 incident gần nhất. Kết quả không phản ánh trạng thái thực tế hiện tại; tìm không thấy không chứng minh không có incident.
- **Kiểm tra nguồn:** chỉ kiểm tra câu trả lời có chứa ít nhất một URL đã quan sát hay không, không chứng minh mọi claim đều có nguồn hay URL được gắn đúng claim. Kiểm tra link chỉ nhắc một lần và chỉ nhận dạng link dạng `/incidents/<id>`.
- **Thay đổi gộp:** prompt và hai bước kiểm tra trong loop được thêm cùng lúc, nên không tách được chính xác phần nào gây ra từng thay đổi điểm (ví dụ H2).
