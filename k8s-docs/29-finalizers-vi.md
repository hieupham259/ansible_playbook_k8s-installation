# Finalizers

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/finalizers/>

Finalizer là các key có namespace (namespaced key) báo cho Kubernetes chờ đến khi
các điều kiện cụ thể được thỏa mãn trước khi xóa hoàn toàn những resource
đã được đánh dấu xóa.
Finalizer cảnh báo các controller để chúng dọn dẹp những resource
mà object bị xóa từng sở hữu.

Khi bạn yêu cầu Kubernetes xóa một object có chỉ định finalizer,
Kubernetes API đánh dấu object đó chờ xóa bằng cách điền giá trị cho `.metadata.deletionTimestamp`,
và trả về status code `202` (HTTP "Accepted"). Object đích vẫn ở trạng thái đang kết thúc (terminating)
trong khi control plane, hoặc các thành phần khác, thực hiện các hành động
mà finalizer định nghĩa.
Sau khi các hành động này hoàn tất, controller xóa các finalizer liên quan
khỏi object đích. Khi field `metadata.finalizers` rỗng,
Kubernetes coi việc xóa đã hoàn tất và xóa object đó.

Bạn có thể dùng finalizer để kiểm soát việc thu gom rác (garbage collection)
của các object bằng cách cảnh báo controller
thực hiện các tác vụ dọn dẹp cụ thể trước khi xóa resource đích.

Finalizer thường không chỉ định code cần thực thi. Thay vào đó, chúng
thường là danh sách các key trên một resource cụ thể, tương tự như annotation.
Kubernetes tự động chỉ định một số finalizer, nhưng bạn cũng có thể
tự chỉ định finalizer của riêng mình.

## Cách finalizer hoạt động (How finalizers work)

Khi bạn tạo một resource bằng file manifest, bạn có thể chỉ định finalizer trong
field `metadata.finalizers`. Khi bạn cố gắng xóa resource đó, API server
xử lý yêu cầu xóa sẽ nhận thấy các giá trị trong field `finalizers`
và thực hiện những việc sau:

  * Sửa object để thêm field `metadata.deletionTimestamp` với
    thời điểm bạn bắt đầu việc xóa.
  * Ngăn không cho object bị gỡ bỏ cho đến khi tất cả các mục trong field `metadata.finalizers` của nó được xóa hết
  * Trả về status code `202` (HTTP "Accepted")

Controller quản lý finalizer đó nhận thấy object được cập nhật với
`metadata.deletionTimestamp`, cho biết việc xóa object đã được yêu cầu.
Sau đó controller cố gắng thỏa mãn các yêu cầu của những finalizer
được chỉ định cho resource đó. Mỗi khi một điều kiện finalizer được thỏa mãn,
controller xóa key đó khỏi field `finalizers` của resource. Khi field
`finalizers` rỗng, object có field `deletionTimestamp` đã được đặt
sẽ tự động bị xóa. Bạn cũng có thể dùng finalizer để ngăn việc xóa các resource không được quản lý (unmanaged resources).

Một ví dụ thường gặp về finalizer là `kubernetes.io/pv-protection`, dùng để ngăn
việc vô tình xóa các object `PersistentVolume`. Khi một object `PersistentVolume`
đang được một Pod sử dụng, Kubernetes thêm finalizer `pv-protection`. Nếu bạn
cố xóa `PersistentVolume` đó, nó chuyển sang trạng thái `Terminating`, nhưng
controller không thể xóa nó vì finalizer vẫn tồn tại. Khi Pod ngừng
sử dụng `PersistentVolume`, Kubernetes gỡ bỏ finalizer `pv-protection`,
và controller xóa volume đó.

> **Ghi chú:**
>
> * Khi bạn `DELETE` một object, Kubernetes thêm deletion timestamp cho object đó rồi
> ngay lập tức bắt đầu hạn chế các thay đổi đối với field `.metadata.finalizers` của object
> hiện đang chờ xóa. Bạn có thể gỡ bỏ các finalizer hiện có (xóa một mục khỏi danh sách `finalizers`)
> nhưng không thể thêm finalizer mới. Bạn cũng không thể sửa `deletionTimestamp` của một
> object một khi nó đã được đặt.
>
> * Sau khi việc xóa đã được yêu cầu, bạn không thể khôi phục object này. Cách duy nhất là xóa nó và tạo một object mới tương tự.

> **Ghi chú:** Tên finalizer tùy chỉnh **bắt buộc** phải là tên finalizer có định danh công khai đầy đủ (publicly qualified), ví dụ `example.com/finalizer-name`.
> Kubernetes cưỡng chế định dạng này; API server từ chối các thao tác ghi lên object nếu thay đổi đó không dùng tên finalizer đầy đủ định danh cho bất kỳ finalizer tùy chỉnh nào.

## Owner reference, label và finalizer (Owner references, labels, and finalizers) {#owners-labels-finalizers}

Giống như label, [owner reference](./30-owners-dependents-vi.md)
mô tả mối quan hệ giữa các object trong Kubernetes, nhưng được dùng cho
mục đích khác. Khi một controller quản lý các object
như Pod, nó dùng label để theo dõi thay đổi của các nhóm object liên quan. Ví
dụ, khi một Job tạo một hoặc
nhiều Pod, Job controller gắn label lên các Pod đó và theo dõi thay đổi của
bất kỳ Pod nào trong cluster có cùng label.

Job controller cũng thêm *owner reference* vào các Pod đó, trỏ đến
Job đã tạo ra chúng. Nếu bạn xóa Job trong khi các Pod này đang chạy,
Kubernetes dùng owner reference (chứ không phải label) để xác định những Pod nào trong
cluster cần được dọn dẹp.

Kubernetes cũng xử lý các finalizer khi nó nhận diện owner reference trên một
resource được nhắm đến để xóa.

Trong một số tình huống, finalizer có thể chặn việc xóa các object phụ thuộc (dependent objects),
khiến object chủ sở hữu (owner) được nhắm đến tồn tại
lâu hơn dự kiến mà không bị xóa hoàn toàn. Trong những tình huống này, bạn
nên kiểm tra finalizer và owner reference trên object chủ sở hữu và các object
phụ thuộc liên quan để tìm nguyên nhân.

> **Ghi chú:** Trong các trường hợp object bị kẹt ở trạng thái đang xóa, tránh gỡ bỏ
> finalizer thủ công để cho việc xóa tiếp tục. Finalizer thường được thêm
> vào resource vì một lý do nào đó, nên việc cưỡng chế gỡ bỏ chúng có thể dẫn đến sự cố trong
> cluster của bạn. Chỉ nên làm vậy khi đã hiểu rõ mục đích của finalizer
> và mục đích đó được hoàn thành theo cách khác (ví dụ, dọn dẹp thủ công
> một object phụ thuộc nào đó).

## Tiếp theo (What's next)

* Đọc [Using Finalizers to Control Deletion](https://kubernetes.io/blog/2021/05/14/using-finalizers-to-control-deletion/)
  trên blog của Kubernetes.
