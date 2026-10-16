# Học cùng mentor — Rancher, Helm, certificate và TLS

Bắt đầu từ [giáo án](rancher-mentor-study-plan.md). Thư mục này lưu giáo án và đầy đủ các bài
học đã hoàn tất; tài liệu giải thích cơ chế vẫn nằm trong
[cloudflare-rancher-docs](../cloudflare-rancher-docs/setup-rancher.md), còn lệnh/gate nằm trong
[runbook](../runbook-k8s-vmware.md#14-cài-rancher-2143--quản-lý-cụm).

## Cách lưu bài

Khi hoàn tất bài nào, mentor tự động tạo file cùng thư mục: `M01.md`, `M02.md`, …, `M12.md`,
rồi đổi tên file trong bảng bên dưới thành liên kết và cập nhật trạng thái. Đây là yêu cầu lưu
bài đã được người học đưa ra, không cần hỏi lại xác nhận ở cuối mỗi bài.

File bài giữ đầy đủ phần giảng, định nghĩa, ví dụ, sơ đồ, code/lệnh, nguồn tham chiếu và các lượt
hỏi–đáp theo thứ tự. Ghi rõ người nói; giữ nguyên câu trả lời của người học, cả điểm nhầm, nhận
xét sửa nhầm và các lượt trả lời lại. Kết thúc bằng checkpoint. Không thay hội thoại bằng một
đáp án mẫu/tóm tắt, không bịa nội dung chưa trao đổi và không tạo trước file bài chưa hoàn tất.
Xem [quy tắc chi tiết trong giáo án](rancher-mentor-study-plan.md#lưu-đầy-đủ-từng-bài-sau-khi-hoàn-tất).

## Tiến độ và danh sách bài

| Bài | Nội dung | Trạng thái | File khi hoàn tất |
| --- | --- | --- | --- |
| M01 | Helm: chart, values, manifest, release | Hoàn tất | [M01.md](M01.md) |
| M02 | Object, controller và CRD | Hoàn tất | [M02.md](M02.md) |
| M03 | Certificate, CA và key | Đang học phần certificate và cặp khóa | `M03.md` — chưa tạo |
| M04 | Ingress, Issuer, Certificate và hai Secret | Chưa học | `M04.md` — chưa tạo |
| M05 | Chuỗi xin và cấp certificate | Chưa học | `M05.md` — chưa tạo |
| M06 | Helm --wait và công việc bất đồng bộ | Chưa học | `M06.md` — chưa tạo |
| M07 | Readiness và gate §14.4 | Chưa học | `M07.md` — chưa tạo |
| M08 | Một phiên TLS: client và server | Chưa học | `M08.md` — chưa tạo |
| M09 | Handshake TLS 1.3 | Chưa học | `M09.md` — chưa tạo |
| M10 | Hai phiên TLS và hai hệ DNS trong lab | Chưa học | `M10.md` — chưa tạo |
| M11 | Origin URL, Origin Server Name và HTTP Host | Chưa học | `M11.md` — chưa tạo |
| M12 | Access, 302, 502 và khoanh vùng lỗi | Chưa học | `M12.md` — chưa tạo |

Câu 1/6 đã trả lời trước giáo án. Câu 2/6 chờ trả lời lại sau M07; câu 3/6 chờ sau M12.
Không dùng việc tạo thư mục hoặc chuyển giáo án để xác nhận hoàn tất bài học.

## Checkpoint đang học

- **Bài đã hoàn tất:** [M01 — Helm](M01.md), [M02 — Object, controller và CRD](M02.md).
  Các file giữ đầy đủ phần giảng, câu trả lời gốc và nhận xét.
- **Điều người học đã tự giải thích đúng ở M02:** cần controller hoạt động để thực hiện yêu cầu;
  phân biệt lỗi chấp nhận bản khai ở webhook với xử lý object đã lưu; ingress-shim là controller
  tạo object Certificate; cainjector là controller cập nhật dữ liệu CA trong cấu hình webhook.
- **Câu hỏi phụ M02 chưa được trả lời trực tiếp:** nguồn sinh annotation chỉ Issuer; Issuer là
  object cấu hình hay chương trình. Nội dung đã được giảng và lưu đầy đủ, sẽ củng cố ở M04;
  không ghi các câu này là đã được người học tự giải thích đúng.
- **Bài hiện tại:** M03 — Certificate, CA và key ở mức cần thiết.
- **Phần mở đầu:** certificate gắn hostname với public key (khóa công khai); private key là khóa
  bí mật tương ứng. Certificate được phép gửi cho client; private key phải được giữ kín.
- **Ví dụ:** khi Traefik đã nạp certificate Rancher đúng cấu hình lab từ Secret
  `tls-rancher-ingress`, certificate dành cho `rancher.hieupn.site` được trình cho client.
- **Câu kiểm tra M03 đang chờ trả lời:** “Giả sử Traefik đã nạp certificate Rancher đúng cấu hình
  lab. Trong hai thứ certificate và private key tương ứng, Traefik gửi thứ nào cho cloudflared,
  giữ kín thứ nào?”
- **Phần tiếp theo:** kiểm tra phân biệt certificate/private key, rồi tiếp tục vai trò khóa và CA.
- **Câu 2/6 và 3/6:** chưa trả lời/chấm lượt mới; giữ mốc sau M07 và M12.
- **Cluster thực tế:** chưa có output được cung cấp trong các bài này. Ví dụ là giả định để học,
  không phải kết quả kiểm tra cluster.
