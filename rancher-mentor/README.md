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
| M02 | Object, controller và CRD | Đang học phần object và controller | `M02.md` — chưa tạo |
| M03 | Certificate, CA và key | Chưa học | `M03.md` — chưa tạo |
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

- **Bài đã hoàn tất:** [M01 — Helm làm gì?](M01.md). Đã lưu đầy đủ phần giảng và hỏi–đáp,
  gồm câu trả lời gốc, nhận xét và các lượt sửa.
- **Điều người học đã tự giải thích đúng ở M01:** đầu vào của render; chỉ render thì chưa tạo Pod;
  dùng `helm template` để xem manifest trước khi cài; Kubernetes nhận manifest rồi tạo tài nguyên;
  thay Pod không tự tạo release Helm mới.
- **Điểm cần tiếp tục củng cố:** release là bản cài có tên do Helm quản lý; không đồng nhất
  release với manifest hay Pod.
- **Bài hiện tại:** M02 — object, controller và CRD.
- **Phần mở đầu:** object là bản ghi cấu hình/trạng thái trong Kubernetes; controller là chương
  trình quan sát và xử lý để trạng thái thực tiến gần trạng thái mong muốn.
- **Ví dụ:** object Deployment `rancher` ở namespace `cattle-system` khai `replicas: 1`;
  các controller liên quan phối hợp để duy trì số Pod mong muốn.
- **Câu kiểm tra M02 đang chờ trả lời:** “Giả sử trong namespace cattle-system, Deployment Rancher
  vẫn khai replicas: 1 nhưng các controller liên quan không hoạt động. Nếu Pod mất đi, chỉ
  riêng bản khai đó có tự tạo Pod bù không? Vì sao?”
- **Phần tiếp theo:** kiểm tra phân biệt object/controller, rồi tiếp tục M02 theo giáo án.
- **Câu 2/6 và 3/6:** chưa trả lời/chấm lượt mới; giữ mốc sau M07 và M12.
- **Cluster thực tế:** chưa có output được cung cấp trong các bài này; ví dụ trên là tình huống
  giả định để học, không phải kết quả kiểm tra cluster.
