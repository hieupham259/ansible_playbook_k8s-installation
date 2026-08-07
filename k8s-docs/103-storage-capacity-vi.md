# Dung lượng lưu trữ (Storage Capacity)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/storage-capacity/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](LO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 14/16 · Kiểm chứng ở
Lab 6b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này là chỗ lưu trữ gặp lập lịch. Nó trả lời câu hỏi bạn đã đặt ra ở bài
[96](96-storage-classes-vi.md): khi dùng `WaitForFirstConsumer`, scheduler lấy đâu ra thông tin
để biết node nào còn chỗ. Đây cũng là bài bắc cầu sang giai đoạn 7, nên đọc phần *Lập lịch* thật
kỹ và bỏ qua chi tiết triển khai.

**Phải hiểu ở lần đọc này:**

- Vấn đề gốc: dung lượng khác nhau theo node — lưu trữ mạng có thể không tới được mọi node,
  hoặc vốn là lưu trữ cục bộ. Không theo dõi dung lượng thì scheduler chọn node không đủ chỗ và
  phải lập lịch lại nhiều lần — đoạn mở đầu.
- Hai phần mở rộng API: đối tượng **`CSIStorageCapacity`** do CSI driver tạo trong namespace của
  driver, mỗi đối tượng mang thông tin dung lượng cho **một storage class** và định nghĩa những
  node nào truy cập được; và trường **`CSIDriverSpec.StorageCapacity`** bật cơ chế này lên —
  mục *API*.
- **Ba điều kiện đồng thời** thì scheduler mới xét dung lượng: Pod dùng volume **chưa được
  tạo**, StorageClass tham chiếu CSI driver và dùng `volumeBindingMode: WaitForFirstConsumer`,
  và `CSIDriver` của driver đó có `StorageCapacity: true`. Với `Immediate` thì **driver quyết
  định nơi tạo volume trước**, scheduler chỉ chạy theo sau — mục *Lập lịch*.
- Phép kiểm tra rất đơn giản: so kích thước volume với dung lượng liệt kê trong các đối tượng
  `CSIStorageCapacity` có topology bao gồm node đó — mục *Lập lịch*.
- Quyết định node chỉ là **tạm thời**: driver được yêu cầu tạo volume kèm gợi ý node; tạo không
  được thì lựa chọn node bị đặt lại và scheduler thử lại. Theo dõi dung lượng **tăng xác suất
  thành công ngay lần đầu chứ không bảo đảm**, vì scheduler quyết định trên thông tin có thể đã
  lỗi thời — mục *Lập lịch lại* và *Hạn chế*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Trước khi bạn bắt đầu* | chỉ nói cần một CSI driver có hỗ trợ theo dõi dung lượng | Lab 6a, khi chọn provisioner |
| Đoạn về volume tạm thời CSI trong mục *Lập lịch* | loại volume này không dùng trong lab | không cần |
| Tình huống thất bại vĩnh viễn với Pod nhiều volume trong *Hạn chế* | hiếm gặp và phải can thiệp tay | không cần |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. StorageClass của bạn để `volumeBindingMode: Immediate`. Scheduler có dùng thông tin
   `CSIStorageCapacity` không? Vậy ai quyết định volume nằm ở đâu, và Pod được lập lịch ra sao?
2. Hai worker của bạn có dung lượng đĩa dành cho lưu trữ khác nhau. Ba điều kiện nào phải đủ
   thì scheduler mới loại bỏ node không đủ chỗ khi chọn nơi chạy Pod?
3. Scheduler đã chọn `k8s-worker2` dựa trên `CSIStorageCapacity`, nhưng driver báo không tạo
   được volume ở đó. Chuyện gì xảy ra tiếp?
4. Bật theo dõi dung lượng lưu trữ có bảo đảm Pod luôn được lập lịch thành công không? Vì sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Thông tin dung lượng chỉ được dùng khi StorageClass ở chế độ
   `WaitForFirstConsumer`. Với `Immediate`, **storage driver quyết định nơi tạo volume, độc lập
   với các Pod sẽ dùng volume đó**; sau khi volume đã tồn tại, scheduler chỉ còn cách lập lịch
   Pod lên các node mà volume đang khả dụng. Nói cách khác thứ tự bị đảo ngược: lưu trữ chọn
   trước, tính toán chạy theo.
2. Ba điều kiện, và phải đủ **cả ba**: Pod dùng một **volume chưa được tạo**; volume đó dùng
   một **StorageClass tham chiếu tới một CSI driver và đặt `volumeBindingMode:
   WaitForFirstConsumer`**; và đối tượng **`CSIDriver` của driver đó có `StorageCapacity` đặt
   là `true`**. Khi đó scheduler chỉ xét những node có đủ dung lượng, bằng một phép so sánh rất
   đơn giản giữa kích thước volume và dung lượng ghi trong các đối tượng `CSIStorageCapacity`
   có topology bao gồm node đó.
3. **Lựa chọn node bị đặt lại và scheduler thử tìm node khác cho Pod.** Bài nói rõ quyết định
   node cho volume `WaitForFirstConsumer` chỉ là **tạm thời**: bước tiếp theo là driver được
   yêu cầu tạo volume kèm gợi ý rằng volume cần khả dụng trên node đã chọn, và vì thông tin
   dung lượng có thể đã lỗi thời nên việc tạo hoàn toàn có thể thất bại.
4. **Không bảo đảm.** Nó chỉ **làm tăng khả năng lập lịch thành công ngay lần đầu**, vì
   scheduler buộc phải quyết định dựa trên thông tin có thể đã lỗi thời. Thông thường cơ chế
   thử lại — giống hệt khi lập lịch mà không có thông tin dung lượng — sẽ xử lý các lần thất
   bại. Đây là chỗ dễ hiểu nhầm: bật tính năng này không biến việc cấp phát thành một giao dịch
   chắc chắn, nó chỉ giảm số vòng thử.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
