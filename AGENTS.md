# Project instructions for Codex

## Runbook step requests

Khi người dùng yêu cầu cung cấp step tiếp theo, tiếp tục một runbook, đưa lại step, xác định tiến
độ trong runbook hoặc giải thích command/script thuộc một step, phải giao việc đọc và trích
runbook cho custom agent `runbook_step_guide` tại
`.codex/agents/runbook-step-guide.toml`.

- Agent phải làm việc ở chế độ read-only và chỉ trả về nội dung bám đúng runbook.
- Chờ agent hoàn thành rồi mới trả lời người dùng.
- Không tự thay đổi thứ tự, command, script, điều kiện PASS/STOP hoặc mở rộng sang bước nằm sau gate
  kế tiếp.
- Câu trả lời cuối phải nêu file/mục/dòng nguồn, nơi chạy, nguyên văn nội dung cần thực hiện, ý
  nghĩa từng command và gate cần gửi output lại.
- Nếu agent phát hiện runbook thiếu, mâu thuẫn hoặc không khớp output thực tế, báo điểm chặn cho
  người dùng; không tự tạo bước thay thế. Chỉ sửa runbook khi người dùng yêu cầu sửa rõ ràng.

Các yêu cầu không liên quan đến việc cung cấp hoặc giải thích step trong runbook không cần giao
cho agent này.
