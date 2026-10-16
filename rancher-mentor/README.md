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
| M01 | Helm: chart, values, manifest, release | Đang học | `M01.md` — chưa tạo |
| M02 | Object, controller và CRD | Chưa học | `M02.md` — chưa tạo |
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

- **Bài hiện tại:** M01.
- **Phần vừa học:** chart Rancher + `rancher-values.yaml` → Helm render → manifest YAML;
  khi cài đặt, Helm gửi manifest đã render tới Kubernetes.
- **Người học đã xác định đúng:** hai đầu vào là chart Rancher và file values.
- **Điểm mentor vừa sửa, chưa kiểm tra lại:** render do Helm thực hiện trước khi gửi bản khai
  tới Kubernetes; không phải gửi vào Kubernetes để Kubernetes render.
- **Câu kiểm tra đang chờ trả lời:** “Giả sử chưa cài Rancher, bạn chỉ dùng Helm để render chart
  với values, chưa gửi bản khai tới Kubernetes. Lúc này cluster đã có Pod Rancher mới chưa?
  Vì sao?”
- **Phần tiếp theo:** kiểm tra lại điểm trên, sau đó tiếp tục M01 về mục đích của
  `helm template` / `helm install`, release và vai trò của Kubernetes trong việc chạy Pod.
- **Trạng thái:** chưa hoàn tất M01, chưa tạo `M01.md`; chưa chấm lượt mới câu 2/6 hoặc 3/6.

Các lượt giảng và hỏi–đáp trước checkpoint vẫn cần được giữ nguyên để đưa vào `M01.md` khi
hoàn tất bài. Khi nối tiếp ở chat mới, dùng lịch sử hội thoại cùng checkpoint; nếu thiếu lịch
sử thì nêu rõ phần thiếu, không tự dựng lại câu trả lời.
