# Tự động dọn dẹp các Job đã hoàn thành (Automatic Cleanup for Finished Jobs)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/>
>
> Cơ chế time-to-live (TTL) để dọn dẹp các Job cũ đã chạy xong.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [stable]`

Khi Job của bạn đã chạy xong, việc giữ lại Job đó trong API (thay vì xóa Job ngay lập tức)
là rất hữu ích, để bạn có thể biết được Job đã thành công hay thất bại.

Controller TTL-after-finished của Kubernetes cung cấp một cơ chế TTL (time to live —
thời gian tồn tại) để giới hạn vòng đời của các object Job đã chạy xong.

## Dọn dẹp các Job đã hoàn thành (Cleanup for finished Jobs)

Controller TTL-after-finished chỉ được hỗ trợ cho Job. Bạn có thể dùng cơ chế này để tự động
dọn dẹp các Job đã hoàn thành (ở trạng thái `Complete` hoặc `Failed`) bằng cách chỉ định
trường `.spec.ttlSecondsAfterFinished` của Job, như trong
[ví dụ này](https://kubernetes.io/docs/concepts/workloads/controllers/job/#clean-up-finished-jobs-automatically).

Controller TTL-after-finished coi một Job là đủ điều kiện để được dọn dẹp sau khi Job đó
kết thúc được TTL giây. Bộ đếm thời gian bắt đầu chạy khi status condition của Job thay đổi
để cho thấy Job đang ở trạng thái `Complete` hoặc `Failed`; khi TTL hết hạn, Job đó trở nên
đủ điều kiện để bị xóa [theo tầng](./36-garbage-collection-vi.md#cascading-deletion)
(cascading). Khi controller TTL-after-finished dọn dẹp một Job, nó sẽ xóa Job theo tầng,
nghĩa là nó sẽ xóa cả các object phụ thuộc (dependent object) cùng với Job đó.

Kubernetes tôn trọng các bảo đảm về vòng đời object trên Job, chẳng hạn như chờ các
[finalizer](./29-finalizers-vi.md).

Bạn có thể đặt giá trị TTL (tính bằng giây) vào bất kỳ lúc nào. Dưới đây là một số ví dụ
về cách đặt trường `.spec.ttlSecondsAfterFinished` của một Job:

* Chỉ định trường này ngay trong manifest của Job, để Job có thể được tự động dọn dẹp một
  khoảng thời gian sau khi nó chạy xong.
* Đặt trường này theo cách thủ công cho các Job đã tồn tại và đã chạy xong, để chúng trở
  nên đủ điều kiện được dọn dẹp.
* Dùng một
  [mutating admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook)
  để đặt trường này một cách động (dynamic) tại thời điểm tạo Job. Quản trị viên cluster có
  thể dùng cách này để áp đặt một chính sách TTL cho các Job đã hoàn thành.
* Dùng một
  [mutating admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook)
  để đặt trường này một cách động sau khi Job đã chạy xong, và chọn các giá trị TTL khác
  nhau dựa trên trạng thái (status), các label của Job. Với trường hợp này, webhook cần
  phát hiện các thay đổi trong `.status` của Job và chỉ đặt TTL khi Job đang được đánh dấu
  là đã hoàn thành.
* Tự viết controller của riêng bạn để quản lý TTL dọn dẹp cho các Job khớp với một
  selector cụ thể.

## Những điểm cần lưu ý (Caveats)

### Cập nhật TTL cho các Job đã hoàn thành (Updating TTL for finished Jobs)

Bạn có thể sửa đổi khoảng thời gian TTL, ví dụ trường `.spec.ttlSecondsAfterFinished` của
Job, sau khi Job đã được tạo hoặc đã chạy xong. Nếu bạn kéo dài khoảng TTL sau khi khoảng
`ttlSecondsAfterFinished` hiện có đã hết hạn, Kubernetes không bảo đảm sẽ giữ lại Job đó,
ngay cả khi yêu cầu cập nhật để kéo dài TTL trả về phản hồi API thành công.

### Lệch thời gian (Time skew)

Vì controller TTL-after-finished dùng các timestamp được lưu trong các Job của Kubernetes
để xác định TTL đã hết hạn hay chưa, tính năng này nhạy cảm với hiện tượng lệch thời gian
(time skew) trong cluster của bạn, điều này có thể khiến control plane dọn dẹp các object
Job vào sai thời điểm.

Đồng hồ không phải lúc nào cũng chính xác, nhưng chênh lệch thường rất nhỏ. Hãy lưu ý rủi
ro này khi đặt một giá trị TTL khác không.

## Tiếp theo (What's next)

* Đọc [Tự động dọn dẹp Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/#clean-up-finished-jobs-automatically)

* Tham khảo [Kubernetes Enhancement Proposal](https://github.com/kubernetes/enhancements/blob/master/keps/sig-apps/592-ttl-after-finish/README.md)
  (KEP) về việc bổ sung cơ chế này.
