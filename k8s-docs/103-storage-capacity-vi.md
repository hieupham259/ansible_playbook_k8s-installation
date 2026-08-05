# Dung lượng lưu trữ (Storage Capacity)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/storage-capacity/>

Dung lượng lưu trữ là hữu hạn và có thể khác nhau tùy theo node mà Pod
chạy trên đó: kho lưu trữ gắn qua mạng (network-attached storage) có thể
không truy cập được từ mọi node, hoặc kho lưu trữ vốn dĩ là cục bộ của
một node ngay từ đầu.

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Trang này mô tả cách Kubernetes theo dõi dung lượng lưu trữ và cách
scheduler dùng thông tin đó để [lập lịch Pod](https://kubernetes.io/docs/concepts/scheduling-eviction/)
lên các node có quyền truy cập đủ dung lượng lưu trữ cho những volume
còn thiếu. Nếu không có tính năng theo dõi dung lượng lưu trữ, scheduler
có thể chọn một node không đủ dung lượng để cấp phát (provision) volume,
và sẽ cần nhiều lần lập lịch lại.

## Trước khi bạn bắt đầu (Before you begin)

Kubernetes v1.36 bao gồm hỗ trợ API ở cấp cluster cho việc theo dõi
dung lượng lưu trữ. Để dùng tính năng này, bạn cũng phải sử dụng một
CSI driver hỗ trợ theo dõi dung lượng. Hãy tham khảo tài liệu của các
CSI driver mà bạn dùng để biết tính năng này có sẵn hay không và, nếu
có, cách sử dụng nó. Nếu bạn không chạy Kubernetes v1.36, hãy xem
tài liệu của phiên bản Kubernetes đó.

## API

Có hai phần mở rộng API cho tính năng này:
- Các đối tượng [CSIStorageCapacity](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/csi-storage-capacity-v1/):
  chúng được CSI driver tạo ra trong namespace nơi driver được cài đặt.
  Mỗi đối tượng chứa thông tin dung lượng cho một storage class và định
  nghĩa những node nào có quyền truy cập vào kho lưu trữ đó.
- [Trường `CSIDriverSpec.StorageCapacity`](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/csi-driver-v1/#CSIDriverSpec):
  khi được đặt là `true`, Kubernetes scheduler sẽ xem xét dung lượng
  lưu trữ đối với các volume dùng CSI driver đó.

## Lập lịch (Scheduling)

Thông tin dung lượng lưu trữ được Kubernetes scheduler sử dụng nếu:
- một Pod dùng một volume chưa được tạo,
- volume đó dùng một StorageClass tham chiếu tới một CSI driver và
  dùng [chế độ gắn kết volume (volume binding mode)](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode)
  `WaitForFirstConsumer`, và
- đối tượng `CSIDriver` của driver đó có `StorageCapacity` được đặt là
  true.

Trong trường hợp đó, scheduler chỉ xem xét cho Pod những node có đủ
dung lượng lưu trữ khả dụng. Phép kiểm tra này rất đơn giản, chỉ so
sánh kích thước volume với dung lượng được liệt kê trong các đối tượng
`CSIStorageCapacity` có topology bao gồm node đó.

Với các volume dùng chế độ gắn kết `Immediate`, storage driver quyết
định nơi tạo volume, độc lập với các Pod sẽ dùng volume đó. Sau khi
volume được tạo, scheduler sẽ lập lịch Pod lên các node mà volume đang
khả dụng.

Với [volume tạm thời CSI (CSI ephemeral volumes)](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/#csi-ephemeral-volumes),
việc lập lịch luôn diễn ra mà không xem xét dung lượng lưu trữ. Điều
này dựa trên giả định rằng loại volume này chỉ được dùng bởi các CSI
driver đặc biệt vốn cục bộ trên một node và không cần tài nguyên đáng
kể ở đó.

## Lập lịch lại (Rescheduling)

Khi một node đã được chọn cho Pod có các volume `WaitForFirstConsumer`,
quyết định đó vẫn chỉ là tạm thời. Bước tiếp theo là CSI storage driver
được yêu cầu tạo volume kèm theo gợi ý rằng volume này cần khả dụng
trên node đã chọn.

Vì Kubernetes có thể đã chọn node dựa trên thông tin dung lượng đã lỗi
thời, nên có khả năng volume thực tế không thể được tạo. Khi đó lựa
chọn node bị đặt lại và Kubernetes scheduler thử lại việc tìm node cho
Pod.

## Hạn chế (Limitations)

Theo dõi dung lượng lưu trữ làm tăng khả năng lập lịch thành công ngay
lần đầu, nhưng không thể đảm bảo điều đó vì scheduler phải quyết định
dựa trên thông tin có thể đã lỗi thời. Thông thường, cơ chế thử lại
giống như khi lập lịch không có thông tin dung lượng lưu trữ sẽ xử lý
các lần lập lịch thất bại.

Một tình huống mà việc lập lịch có thể thất bại vĩnh viễn là khi Pod
dùng nhiều volume: một volume có thể đã được tạo trong một phân đoạn
topology (topology segment) mà sau đó không còn đủ dung lượng cho
volume khác. Cần can thiệp thủ công để khôi phục, ví dụ bằng cách tăng
dung lượng hoặc xóa volume đã được tạo.

## Tiếp theo (What's next)

 - Để biết thêm thông tin về thiết kế, xem
[KEP Storage Capacity Constraints for Pod Scheduling](https://github.com/kubernetes/enhancements/blob/master/keps/sig-storage/1472-storage-capacity-tracking/README.md).
