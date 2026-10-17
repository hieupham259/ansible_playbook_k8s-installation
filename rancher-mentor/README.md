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

## Đối chiếu vị trí trong diagram

Sau mỗi câu trả lời được đánh giá đúng hoặc khi hoàn tất một phần/bài, mentor chỉ rõ **phần,
tên diagram, nhãn node/mũi tên và ý nghĩa của đoạn vừa hiểu**. **Khi nhắc một diagram, phải vẽ
ngay bằng Mermaid trong câu trả lời; không gửi link thay cho hình hoặc bắt người học mở trang khác.** Dùng
[Nền tảng](../cloudflare-rancher-docs/setup-rancher.md#nền-tảng--helm-cert-manager-và-rancher-thực-chất-là-gì)
và [chuỗi cấp certificate](../cloudflare-rancher-docs/setup-rancher.md#sơ-đồ-tuần-tự--chuỗi-cấp-certificate-chạy-ngầm-sau-helm-install)
để đối chiếu. Xem [bảng hướng dẫn trong giáo án](rancher-mentor-study-plan.md#đối-chiếu-diagram-sau-mỗi-phần-đã-hiểu).

Các lần đối chiếu và Mermaid thực sự đã đưa trong hội thoại sẽ được giữ trong file bài và
checkpoint. Nếu vẽ trích đoạn/rút gọn hoặc đổi vai client/server cho lab, phải nói rõ. Nếu diagram
không có node riêng cho chi tiết đang học, giải thích ngay và nói rõ giới hạn; minh họa mới phải
được phân biệt với sơ đồ gốc. Đây là tiến độ hiểu bài, không phải trạng thái cluster thực tế.

## Tiến độ và danh sách bài

| Bài | Nội dung | Trạng thái | File khi hoàn tất |
| --- | --- | --- | --- |
| M01 | Helm: chart, values, manifest, release | Hoàn tất | [M01.md](M01.md) |
| M02 | Object, controller và CRD | Hoàn tất | [M02.md](M02.md) |
| M03 | Certificate, CA và key | Hoàn tất kiến thức; bản lưu còn thiếu một số lời giảng đầu bài | [M03.md](M03.md) |
| M04 | Ingress, Issuer, Certificate và hai Secret | Hoàn tất | [M04.md](M04.md) |
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

- **Bài đã hoàn tất:** [M01 — Helm](M01.md), [M02 — Object, controller và CRD](M02.md), [M03 — Certificate, CA và key](M03.md), [M04 — Ingress, Issuer, Certificate và hai Secret](M04.md).
- **Giới hạn bản lưu M03:** còn thiếu nguyên văn nhiều phản hồi Mentor và một số Mermaid đầu bài. File giữ nguyên những lượt còn truy xuất được, đánh dấu chỗ thiếu và chép phần lời giảng trong ảnh 2; không dựng hội thoại. Trạng thái hoàn tất kiến thức dựa trên các câu trả lời cuối bài, không có nghĩa bản chép đã đủ toàn văn.
- **Điều người học đã tự giải thích đúng ở M03:** chữ ký trên certificate dùng private key CA từ Secret `tls-rancher` và có từ trước kết nối; CertificateVerify chứng minh bên kia giữ private key tương ứng với public key trong certificate; HTTPS vẫn mã hóa khi `noTLSVerify: true`, nhưng kết nối thành công chưa chứng minh xác minh đúng Rancher theo CA/hostname.
- **Câu trả lời chốt M03 nguyên văn:** “dữ liệu được mã hoá nhưng việc kết nối thành công không đủ chứng minh `cloudflared` xác minh đúng Rancher theo CA và hostname”.
- **Điểm đã được sửa trong M03:** người tạo chữ ký CA khác người gửi certificate; CertificateVerify là bản tin mang chữ ký handshake, không phải certificate thứ hai. Giữ các câu trả lời nhầm ban đầu trong file bài, không thay bằng đáp án đã sửa.
- **Vị trí đối chiếu cuối M03:** phần “Controller nào cấp certificate và ai gửi hai bản tin TLS?” của `setup-rancher.md`, bước 3/bước 8 và bước 10 cùng ghi chú `noTLSVerify`. Minh họa bổ sung tách kiểm chữ ký handshake khỏi kiểm CA/hostname được lưu ở lượt chốt M03.
- **Bài vừa hoàn tất:** [M04 — Ingress, Issuer, Certificate và hai Secret](M04.md), ngày 18/10/2026 (giờ Việt Nam).
- **Giới hạn bản lưu M04:** phần trước phiên này chỉ có checkpoint người học cung cấp; không dựng lại hội thoại cũ. File giữ phần giảng, các lượt hỏi–đáp, nhận xét/sửa nhầm, Mermaid và bản review 10 bước trong phiên 17–18/10/2026.
- **Điều người học đã tự giải thích đúng ở M04:** Certificate tồn tại chưa chứng minh Secret có certificate đã ký; certificates-issuing ghi kết quả; CSR nằm trong CertificateRequest; Issuer tham chiếu CA bằng spec.ca.secretName; tls-rancher giữ dữ liệu CA dùng để ký, tls-rancher-ingress giữ certificate/key endpoint cho Traefik; có thể cấp lại bằng cùng CA còn hợp lệ; khi cần certificate mới, cert-manager tạo CertificateRequest mới cho lần xin cấp đó.
- **Câu trả lời chốt M04 nguyên văn:** “tức là vì Cần certificate mới nên cần dùng `CertificateRequest`mới để cấp lại certificate”. Mentor đã xác nhận đây là việc cấp/gia hạn chạy nền, không xảy ra cho mỗi kết nối TLS hoặc mỗi HTTP request mới.
- **Điểm đã được sửa trong M04:** Certificate object khác nơi lưu certificate đã ký và khác bản tin TLS Certificate; CertificateRequest đã hoàn tất khác với certificate đã cấp hết hạn; quy tắc một lần cấp thuộc API/controller của cert-manager. Giữ nguyên các câu trả lời nhầm, các phản biện và những giải thích Mentor đã đính chính trong file bài.
- **Vị trí vừa đối chiếu ở M04:** các minh họa mới về yêu cầu → controller → Secret kết quả, Issuer → Secret CA, hai Secret có hai mục đích, và Certificate → key-manager/request-manager → CertificateRequest. Đoạn 10 bước trong setup-rancher.md đã được người học review và yêu cầu cập nhật ở commit 2684e67; không coi các minh họa là trạng thái cluster thực tế.
- **Bài tiếp theo:** M05 — hệ thống lại chuỗi xin và cấp certificate, ai tạo gì và ai đọc gì tới lúc Traefik nạp dữ liệu. Chưa bắt đầu bài M05; các phần tra cứu để sửa tài liệu không tự xác nhận hoàn tất M05.
- **Câu kiểm tra M04 đang chờ:** không còn; điểm cuối đã được người học tự chốt.
- **Điểm từ M02 chưa có câu trả lời riêng trực tiếp trong phiên M04:** annotation chỉ Issuer đến từ template chart khi Helm render. Không ghi câu này là đã được người học tự trả lời trong phiên.
- **Câu 2/6:** chưa trả lời lại/chấm lượt mới; giữ mốc sau M07.
- **Câu 3/6:** chưa trả lời; giữ mốc sau M12.
- **Cluster thực tế:** chưa có output được người học cung cấp trong các bài này; ví dụ là giả định để học.
