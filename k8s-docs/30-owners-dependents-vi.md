# Đối tượng sở hữu và đối tượng phụ thuộc (Owners and Dependents)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1c](LO-TRINH-ADMIN.md#1c-vòng-đời-và-cơ-chế-nền-của-object),
bài 2/7 · Kiểm chứng ở [Lab 1c](labs/LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md).

Bài này đặt cạnh bài [18 — Label và Selector](18-labels-vi.md) mới đọc ở nhóm 1b. Cả hai đều
mô tả quan hệ giữa các object, nhưng dùng cho hai việc khác nhau — nắm được ranh giới đó là
mục tiêu chính.

**Phải hiểu ở lần đọc này:**

- **Label dùng để controller theo dõi một nhóm; owner reference dùng để xác định cái gì phải
  bị dọn khi chủ sở hữu bị xóa.** Cùng một Pod thường mang cả hai, phục vụ hai mục đích khác
  nhau.
- Owner reference nằm ở `metadata.ownerReferences` của **object phụ thuộc**, và gồm cả tên lẫn
  **UID** của chủ sở hữu — UID mới là thứ chống nhầm với một object trùng tên đời sau.
- Kubernetes tự đặt field này; bạn hiếm khi cần tự viết.
- `blockOwnerDeletion` quyết định object phụ thuộc có chặn được việc xóa chủ sở hữu hay không.
- **Owner reference giữa các namespace bị cấm theo thiết kế.** Chỉ định sai thì reference bị
  coi như không tồn tại và object phụ thuộc có thể bị xóa.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ví dụ Service tạo `EndpointSlice` | chưa học Service | giai đoạn 5 |
| Ví dụ ReplicaSet, Deployment, Job, CronJob | chưa học các workload controller | giai đoạn 4 |
| Admission controller kiểm soát quyền sửa `blockOwnerDeletion` | thuộc kiểm soát truy cập | giai đoạn 9 |
| Finalizer `foreground` và `orphan` | là cơ chế xóa theo tầng | bài [36](36-garbage-collection-vi.md) ngay sau |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Một controller đã dùng label để theo dõi nhóm Pod của mình. Vậy owner reference còn để làm
   gì nữa?
2. Owner reference nằm trên object nào — chủ sở hữu hay object phụ thuộc? Nó chứa những gì?
3. Vì sao owner reference phải mang **UID** chứ không chỉ tên?
4. Bạn tạo một owner reference trỏ từ một ConfigMap ở namespace `a` tới một object ở namespace
   `b`. Chuyện gì xảy ra?
5. `blockOwnerDeletion` dùng để làm gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Để xác định **cái gì phải bị dọn khi chủ sở hữu bị xóa**. Bài nói rõ: nếu bạn xóa một Job
   trong khi các Pod đang chạy, Kubernetes dùng **owner reference chứ không phải label** để
   biết Pod nào cần dọn. Label trả lời "nhóm này gồm những ai"; owner reference trả lời "ai
   sinh ra cái này".
2. Nằm trên **object phụ thuộc**, ở field `metadata.ownerReferences`. Nó chứa **tên và UID**
   của chủ sở hữu, và chủ sở hữu phải ở cùng namespace nếu là loại thuộc namespace.
3. Vì tên có thể được tái sử dụng — xóa object rồi tạo lại object trùng tên là hợp lệ (bài
   [17](17-names-vi.md)). Nếu chỉ có tên, object phụ thuộc cũ sẽ bám nhầm vào chủ sở hữu mới.
   UID là duy nhất theo từng lần xuất hiện nên loại được nhầm lẫn đó.
4. Owner reference cross-namespace **bị cấm theo thiết kế**. Reference đó bị coi như **không
   tồn tại**, và object phụ thuộc có thể bị xóa một khi tất cả chủ sở hữu được xác nhận là
   không tồn tại. Garbage collector còn ghi một Event cảnh báo với reason
   `OwnerRefInvalidNamespace`.
5. Nó quyết định object phụ thuộc đó có **chặn việc thu gom rác xóa chủ sở hữu** hay không.
   Kubernetes tự đặt thành `true` khi một controller đặt `ownerReferences`; bạn đặt tay được
   để kiểm soát phụ thuộc nào phải bị xóa xong trước khi chủ sở hữu ra đi.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
