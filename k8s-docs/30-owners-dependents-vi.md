# Đối tượng sở hữu và đối tượng phụ thuộc (Owners and Dependents)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/>

Trong Kubernetes, một số object là *chủ sở hữu (owner)* của các object khác.
Ví dụ, một ReplicaSet là chủ sở hữu
của một tập các Pod. Những object được sở hữu này là *đối tượng phụ thuộc (dependent)* của chủ sở hữu của chúng.

Quan hệ sở hữu khác với cơ chế [label và selector](./18-labels-vi.md)
mà một số resource cũng sử dụng. Ví dụ, hãy xét một Service
tạo ra các object `EndpointSlice`. Service dùng label để cho phép control plane
xác định những object `EndpointSlice` nào được dùng cho Service đó. Bên cạnh
label, mỗi `EndpointSlice` được quản lý thay mặt cho một Service còn có
một owner reference. Owner reference giúp các thành phần khác nhau của Kubernetes tránh
can thiệp vào những object mà chúng không kiểm soát.

## Owner reference trong đặc tả object (Owner references in object specifications)

Các object phụ thuộc có field `metadata.ownerReferences` tham chiếu đến
object chủ sở hữu của chúng. Một owner reference hợp lệ bao gồm tên object và một UID
trong cùng namespace với object phụ thuộc. Kubernetes tự động đặt giá trị cho
field này đối với những object là phụ thuộc của các object khác như
ReplicaSet, DaemonSet, Deployment, Job và CronJob, và ReplicationController.
Bạn cũng có thể tự cấu hình các mối quan hệ này bằng cách thay đổi giá trị của
field này. Tuy nhiên, thường thì bạn không cần làm vậy và có thể để Kubernetes
tự động quản lý các mối quan hệ đó.

Các object phụ thuộc cũng có field `ownerReferences.blockOwnerDeletion` nhận
giá trị boolean và kiểm soát việc các phụ thuộc cụ thể có thể chặn việc thu gom rác
(garbage collection) xóa object chủ sở hữu của chúng hay không. Kubernetes tự động đặt
field này thành `true` nếu một controller
(ví dụ, Deployment controller) đặt giá trị cho field
`metadata.ownerReferences`. Bạn cũng có thể đặt giá trị cho field
`blockOwnerDeletion` thủ công để kiểm soát những phụ thuộc nào được chặn việc thu gom
rác.

Một admission controller của Kubernetes kiểm soát quyền của người dùng khi thay đổi field này cho
các resource phụ thuộc, dựa trên quyền xóa (delete) đối với chủ sở hữu. Cơ chế kiểm soát này
ngăn người dùng không được phép trì hoãn việc xóa object chủ sở hữu.

> **Ghi chú:**
> Owner reference giữa các namespace (cross-namespace) bị cấm theo thiết kế.
> Các phụ thuộc thuộc phạm vi namespace có thể chỉ định chủ sở hữu ở phạm vi cluster hoặc phạm vi namespace.
> Một chủ sở hữu thuộc phạm vi namespace **bắt buộc** phải tồn tại trong cùng namespace với phụ thuộc.
> Nếu không, owner reference bị coi như không tồn tại, và object phụ thuộc
> sẽ bị xóa một khi tất cả chủ sở hữu được xác nhận là không tồn tại.
>
> Các phụ thuộc thuộc phạm vi cluster chỉ có thể chỉ định chủ sở hữu thuộc phạm vi cluster.
> Từ v1.20 trở đi, nếu một phụ thuộc phạm vi cluster chỉ định một kind thuộc phạm vi namespace làm chủ sở hữu,
> nó bị coi là có owner reference không thể phân giải được, và không thể được thu gom rác.
>
> Từ v1.20 trở đi, nếu garbage collector phát hiện một `ownerReference` cross-namespace không hợp lệ,
> hoặc một phụ thuộc phạm vi cluster có `ownerReference` tham chiếu đến một kind thuộc phạm vi namespace, một Event cảnh báo
> với reason là `OwnerRefInvalidNamespace` và `involvedObject` là phụ thuộc không hợp lệ đó sẽ được ghi nhận.
> Bạn có thể kiểm tra loại Event này bằng cách chạy
> `kubectl get events -A --field-selector=reason=OwnerRefInvalidNamespace`.

## Quan hệ sở hữu và finalizer (Ownership and finalizers)

Khi bạn yêu cầu Kubernetes xóa một resource, API server cho phép
controller quản lý xử lý mọi [quy tắc finalizer](./29-finalizers-vi.md)
của resource đó. Finalizer
ngăn việc vô tình xóa những resource mà cluster của bạn có thể vẫn cần để hoạt động
đúng. Ví dụ, nếu bạn cố xóa một [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) vẫn đang
được một Pod sử dụng, việc xóa không diễn ra ngay lập tức vì
`PersistentVolume` có finalizer `kubernetes.io/pv-protection` trên nó.
Thay vào đó, [volume](https://kubernetes.io/docs/concepts/storage/volumes/) vẫn ở trạng thái `Terminating` cho đến khi Kubernetes gỡ bỏ
finalizer, điều này chỉ xảy ra sau khi `PersistentVolume` không còn
được gắn (bound) với Pod nào.

Kubernetes cũng thêm finalizer vào resource chủ sở hữu khi bạn dùng
[xóa theo tầng kiểu foreground hoặc orphan (foreground or orphan cascading deletion)](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#cascading-deletion).
Với xóa kiểu foreground, Kubernetes thêm finalizer `foreground` để
controller phải xóa những resource phụ thuộc có
`ownerReferences.blockOwnerDeletion=true` trước khi xóa chủ sở hữu. Nếu bạn
chỉ định chính sách xóa kiểu orphan, Kubernetes thêm finalizer `orphan` để
controller bỏ qua các resource phụ thuộc sau khi xóa object chủ sở
hữu.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [finalizer trong Kubernetes](./29-finalizers-vi.md).
* Tìm hiểu về [thu gom rác (garbage collection)](https://kubernetes.io/docs/concepts/architecture/garbage-collection).
* Đọc tài liệu tham khảo API cho [metadata của object](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/object-meta/#System).
