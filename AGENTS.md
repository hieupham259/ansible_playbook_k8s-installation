# Project instructions for Codex

## Runbook step requests

Có hai custom agent read-only cho runbook:

- `runbook_step_guide` tại `.codex/agents/runbook-step-guide.toml`: dùng khi người dùng yêu cầu
  giải thích có research hoặc đối chiếu tài liệu chính thức.
- `runbook_fast_step_guide` tại `.codex/agents/runbook-fast-step-guide.toml`: dùng khi người dùng
  yêu cầu đối chiếu nhanh gate và lấy nguyên văn step tiếp theo, không research.

Chỉ giao việc cho agent mà người dùng yêu cầu rõ ràng. Trong các trường hợp khác, main thread tự
đọc runbook, đối chiếu output và cung cấp step tiếp theo.

- Khi được yêu cầu sử dụng, agent phải làm việc ở chế độ read-only, chỉ trả về nội dung bám đúng
  runbook và main thread phải chờ agent hoàn thành rồi mới trả lời người dùng.
- Không tự thay đổi thứ tự, command, script, điều kiện PASS/STOP hoặc mở rộng sang bước nằm sau gate
  kế tiếp.
- Câu trả lời cuối phải nêu file/mục/dòng nguồn, nơi chạy, nguyên văn nội dung cần thực hiện, ý
  nghĩa từng command và gate cần gửi output lại.
- Nếu runbook thiếu, mâu thuẫn hoặc không khớp output thực tế, báo điểm chặn cho người dùng; không
  tự tạo bước thay thế. Chỉ sửa runbook khi người dùng yêu cầu sửa rõ ràng.

Không sử dụng hai agent runbook cho yêu cầu không liên quan hoặc khi người dùng không yêu cầu
agent tương ứng.
