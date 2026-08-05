# Phiên bản tương thích cho các thành phần Control Plane của Kubernetes (Compatibility Version For Kubernetes Control Plane Components)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/cluster-administration/compatibility-version/

Kể từ bản phát hành v1.32, chúng tôi đã đưa vào các tùy chọn có thể cấu hình về tương thích phiên bản (version compatibility) và giả lập (emulation) cho các thành phần control plane của Kubernetes, nhằm giúp việc nâng cấp an toàn hơn bằng cách cung cấp nhiều quyền kiểm soát hơn và tăng độ chi tiết của các bước mà quản trị viên cluster có thể thực hiện.

## Phiên bản giả lập (Emulated Version)

Tùy chọn giả lập được đặt bằng flag `--emulated-version` của các thành phần control plane. Nó cho phép thành phần đó giả lập hành vi (các API, tính năng, ...) của một phiên bản Kubernetes cũ hơn.

Khi được sử dụng, các khả năng (capability) sẵn có sẽ khớp với phiên bản được giả lập:
* Bất kỳ khả năng nào có trong binary version nhưng được giới thiệu sau emulation version sẽ không khả dụng.
* Bất kỳ khả năng nào bị loại bỏ sau emulation version sẽ vẫn khả dụng.

Điều này cho phép một binary của một bản phát hành Kubernetes cụ thể giả lập hành vi của một phiên bản trước đó với độ trung thực đủ cao, đến mức khả năng tương tác (interoperability) với các thành phần hệ thống khác có thể được định nghĩa theo phiên bản được giả lập.

Giá trị `--emulated-version` phải <= `binaryVersion`. Xem thông điệp trợ giúp (help message) của flag `--emulated-version` để biết dải phiên bản giả lập được hỗ trợ.
