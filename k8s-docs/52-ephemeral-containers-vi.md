# Các container tạm thời (Ephemeral Containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.25 [stable]`

Trang này cung cấp cái nhìn tổng quan về ephemeral container (container tạm thời): một
loại container đặc biệt chạy tạm thời trong một Pod có sẵn nhằm thực hiện các hành động
do người dùng khởi xướng, chẳng hạn như xử lý sự cố (troubleshooting). Bạn dùng
ephemeral container để kiểm tra (inspect) các dịch vụ chứ không phải để xây dựng ứng
dụng.

## Hiểu về ephemeral container (Understanding ephemeral containers)

Pod là khối xây dựng nền tảng của các ứng dụng Kubernetes. Vì Pod được thiết kế để có thể
vứt bỏ và thay thế được (disposable and replaceable), bạn không thể thêm một container
vào Pod sau khi nó đã được tạo. Thay vào đó, bạn thường xóa và thay thế các Pod theo cách
có kiểm soát bằng các deployment.

Tuy nhiên, đôi khi vẫn cần kiểm tra trạng thái của một Pod có sẵn, ví dụ để xử lý một lỗi
khó tái hiện. Trong những trường hợp này, bạn có thể chạy một ephemeral container trong
một Pod có sẵn để kiểm tra trạng thái của nó và chạy các lệnh tùy ý.

### Ephemeral container là gì? (What is an ephemeral container?)

Ephemeral container khác các container khác ở chỗ chúng thiếu các đảm bảo về tài nguyên
hay việc thực thi, và chúng sẽ không bao giờ được tự động khởi động lại, do đó chúng
không phù hợp để xây dựng ứng dụng. Ephemeral container được mô tả bằng cùng
`ContainerSpec` như các container thông thường, nhưng nhiều trường không tương thích và
không được phép sử dụng cho ephemeral container.

- Ephemeral container không được có port, vì vậy các trường như `ports`,
  `livenessProbe`, `readinessProbe` không được phép sử dụng.
- Việc phân bổ tài nguyên của Pod là bất biến (immutable), vì vậy không được phép đặt
  `resources`.
- Để xem danh sách đầy đủ các trường được phép, hãy xem
  [tài liệu tham khảo về EphemeralContainer](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#ephemeralcontainer-v1-core).

Ephemeral container được tạo thông qua một handler đặc biệt tên là `ephemeralcontainers`
trong API thay vì được thêm trực tiếp vào `pod.spec`, do đó không thể thêm một ephemeral
container bằng `kubectl edit`.

Giống như các container thông thường, bạn không thể thay đổi hoặc xóa một ephemeral
container sau khi đã thêm nó vào một Pod.

> **Ghi chú:**
> Ephemeral container không được hỗ trợ bởi
> [static pod](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/).

## Công dụng của ephemeral container (Uses for ephemeral containers)

Ephemeral container hữu ích cho việc xử lý sự cố theo cách tương tác khi `kubectl exec`
là không đủ, do container đã bị crash hoặc container image không bao gồm các tiện ích gỡ
lỗi (debugging utilities).

Đặc biệt, [distroless image](https://github.com/GoogleContainerTools/distroless) cho phép
bạn triển khai các container image tối giản nhằm giảm bề mặt tấn công (attack surface)
và mức độ phơi nhiễm trước lỗi cũng như lỗ hổng bảo mật. Vì distroless image không bao
gồm shell hay bất kỳ tiện ích gỡ lỗi nào, rất khó để xử lý sự cố với distroless image
nếu chỉ dùng `kubectl exec`.

Khi sử dụng ephemeral container, sẽ hữu ích nếu bật
[chia sẻ process namespace](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/)
để bạn có thể xem các tiến trình (process) trong những container khác.

## Tiếp theo (What's next)

* Tìm hiểu cách [gỡ lỗi pod bằng ephemeral container](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container).
