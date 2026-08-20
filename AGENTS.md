# Project instructions for Codex

## Runbook step requests

Chỉ giao việc đọc và trích runbook cho custom agent `runbook_step_guide` tại
`.codex/agents/runbook-step-guide.toml` khi người dùng yêu cầu rõ ràng sử dụng agent này. Trong
các trường hợp khác, main thread tự đọc runbook, đối chiếu output và cung cấp step tiếp theo.

- Khi được yêu cầu sử dụng, agent phải làm việc ở chế độ read-only, chỉ trả về nội dung bám đúng
  runbook và main thread phải chờ agent hoàn thành rồi mới trả lời người dùng.
- Không tự thay đổi thứ tự, command, script, điều kiện PASS/STOP hoặc mở rộng sang bước nằm sau gate
  kế tiếp.
- Câu trả lời cuối phải nêu file/mục/dòng nguồn, nơi chạy, nguyên văn nội dung cần thực hiện, ý
  nghĩa từng command và gate cần gửi output lại.
- Nếu runbook thiếu, mâu thuẫn hoặc không khớp output thực tế, báo điểm chặn cho người dùng; không
  tự tạo bước thay thế. Chỉ sửa runbook khi người dùng yêu cầu sửa rõ ràng.

Không sử dụng `runbook_step_guide` cho yêu cầu không liên quan hoặc khi người dùng không yêu cầu
agent này.
