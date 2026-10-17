# Học cùng mentor — Rancher, Helm, certificate và TLS

Bắt đầu từ [giáo án](rancher-mentor-study-plan.md). Thư mục này lưu giáo án và đầy đủ các bài
học đã hoàn tất; tài liệu giải thích cơ chế vẫn nằm trong
[cloudflare-rancher-docs](../cloudflare-rancher-docs/setup-rancher.md), còn lệnh/gate nằm trong
[runbook](../runbook-k8s-vmware.md#14-cài-rancher-2143--quản-lý-cụm).
Mục tiêu là hiểu toàn bộ cơ chế trong setup-rancher.md: M01–M06 là chặng nền, sau đó học
Câu 4/6–6/6 trực tiếp trong tài liệu và đối chiếu những phần còn thiếu.

## Cách lưu bài

Khi hoàn tất bài nền nào, mentor tự động tạo file cùng thư mục: `M01.md`, `M02.md`, …, `M06.md`,
rồi đổi tên file trong bảng bên dưới thành liên kết và cập nhật trạng thái. Đây là yêu cầu lưu
bài đã được người học đưa ra, không cần hỏi lại xác nhận ở cuối mỗi bài. Câu 4/6–6/6 sau M06
lưu theo cùng quy tắc trong `C04.md`, `C05.md`, `C06.md`; mã C là câu tình huống, không phải
thêm bài M07–M12. Chỉ tạo file khi đã có trao đổi thực tế và hoàn tất phần tương ứng.

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

Giáo án hiện có **6 bài**. M05 mới gộp M05–M07 cũ; M06 mới gộp M08–M12 cũ. Nội dung cũ
trở thành các mục kiến thức bên trong hai tình huống, không phải các bài độc lập cần học xong
trước khi được trả lời. Các mã/mốc cũ trong M01–M04 là lịch sử đã lưu; giữ nguyên các file đó.

| Bài | Nội dung | Trạng thái | File khi hoàn tất |
| --- | --- | --- | --- |
| M01 | Helm: chart, values, manifest, release | Hoàn tất | [M01.md](M01.md) |
| M02 | Object, controller và CRD | Hoàn tất | [M02.md](M02.md) |
| M03 | Certificate, CA và key | Hoàn tất kiến thức; bản lưu còn thiếu một số lời giảng đầu bài | [M03.md](M03.md) |
| M04 | Ingress, Issuer, Certificate và hai Secret | Hoàn tất | [M04.md](M04.md) |
| M05 | Helm, cấp certificate và gate | Chưa học; dùng Câu 2/6 ngay từ đầu bài | `M05.md` — chưa tạo |
| M06 | TLS, origin và khoanh vùng lỗi | Chưa học; dùng Câu 3/6 ngay từ đầu bài | `M06.md` — chưa tạo |

Câu 1/6 đã trả lời trước giáo án. Câu 2/6 được dùng ngay từ đầu M05; Câu 3/6 được dùng ngay từ đầu M06.
Không dùng việc tạo thư mục hoặc chuyển giáo án để xác nhận hoàn tất bài học.

### Chặng áp dụng sau M06

Hoàn tất M06 chưa phải đã hiểu hết setup-rancher.md. Tiếp tục các tình huống dưới đây, bổ sung
kiến thức ngay trong lúc giải quyết; sau đó dùng [bảng đối chiếu toàn bộ tài liệu](rancher-mentor-study-plan.md#đối-chiếu-toàn-bộ-setup-ranchermd)
để tìm và xử lý phần chưa thực sự học.

| Mốc | Nội dung | Trạng thái | File khi hoàn tất |
| --- | --- | --- | --- |
| [Câu 4/6](../cloudflare-rancher-docs/setup-rancher.md#câu-46--vì-sao-phải-cài-và-kiểm-tra-theo-thứ-tự-này) | Chuẩn bị, backup, thứ tự §14, values/render, bootstrap và các gate | Chưa học/chưa trả lời; chưa có điểm | `C04.md` — chưa tạo |
| [Câu 5/6](../cloudflare-rancher-docs/setup-rancher.md#câu-56--browser-dùng-được-thì-client-và-cluster-khác-đã-dùng-được-chưa) | Server URL, client/agent, quyền/trust, local và downstream | Chưa học/chưa trả lời; chưa có điểm | `C05.md` — chưa tạo |
| [Câu 6/6](../cloudflare-rancher-docs/setup-rancher.md#câu-66--trang-web-và-node-đều-khỏe-vì-sao-rancher-vẫn-lỗi) | Dashboard/API, SelfUser/Steve, sự cố sau reboot và condition local | Chưa học/chưa trả lời; chưa có điểm | `C06.md` — chưa tạo |
| Đối chiếu bao phủ | Các phần chính, từng bước §14 và 15 cơ chế trong giáo án | Chưa hoàn tất | Ghi checkpoint và bổ sung vào file Mxx/Cxx liên quan |

Điểm 0–3 của từng câu và mức bao phủ tài liệu được theo dõi riêng. Chỉ xác nhận đã học toàn bộ
khi các phần còn thiếu đã được giải thích và có căn cứ từ trao đổi; xem [tiêu chí hoàn tất](rancher-mentor-study-plan.md#khi-nào-hoàn-tất-toàn-bộ-tài-liệu).

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
- **Bài tiếp theo:** M05 — Helm, cấp certificate và gate, học cùng Câu 2/6 ngay từ đầu. Gộp phần chuỗi cấp phát, phạm vi `--wait` và gate §14.4 thành kiến thức bổ sung trong tình huống. M05 và M06 đều chưa bắt đầu; việc gộp bài không xác nhận đã học xong các phần đó.
- **Câu kiểm tra M04 đang chờ:** không còn; điểm cuối đã được người học tự chốt.
- **Điểm từ M02 chưa có câu trả lời riêng trực tiếp trong phiên M04:** annotation chỉ Issuer đến từ template chart khi Helm render. Không ghi câu này là đã được người học tự trả lời trong phiên.
- **Câu 2/6:** chưa trả lời lại/chấm lượt mới; dùng ngay từ đầu M05 và giữ nguyên lịch sử/điểm cũ.
- **Câu 3/6:** chưa trả lời; dùng ngay từ đầu M06.
- **Sau M06:** tiếp tục Câu 4/6–6/6 trực tiếp trong setup-rancher.md; cả ba chưa học/chưa trả lời, chưa có điểm.
- **Bao phủ toàn bộ tài liệu:** chưa hoàn tất; dùng bảng đối chiếu để ghi các mục đã tự giải thích và còn thiếu. Hoàn tất chặng nền hoặc đọc đáp án chưa xác nhận đã hiểu toàn bộ.
- **Cluster thực tế:** chưa có output được người học cung cấp trong các bài này; ví dụ là giả định để học.
