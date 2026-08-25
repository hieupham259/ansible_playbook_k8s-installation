# Các lớp Volume Snapshot (Volume Snapshot Classes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 11/16 · Kiểm chứng ở
[Lab 6b](labs/LAB-6B-SNAPSHOT-VA-VOLUME-NANG-CAO.md).

Bài rất ngắn và là bản sao của bài [96](96-storage-classes-vi.md) đặt lên snapshot. Đọc nó chủ
yếu để bắt hai chỗ **khác** StorageClass: cách chọn class mặc định, và việc đối tượng không sửa
được sau khi tạo. Cùng thuộc nhóm nợ lab với bài [99](99-volume-snapshots-vi.md), xem
[sổ nợ lab](labs/README.md#5-sổ-nợ-lab).

**Phải hiểu ở lần đọc này:**

- VolumeSnapshotClass với snapshot đóng vai trò như StorageClass với volume: mô tả "lớp" lưu
  trữ khi cấp phát một volume snapshot — mục *Giới thiệu*.
- Ba trường cốt lõi `driver`, `deletionPolicy`, `parameters`; **`driver` và `deletionPolicy`
  đều bắt buộc**; đối tượng **không cập nhật được sau khi đã tạo**; và không có sẵn CRD thì
  việc tạo VolumeSnapshotClass sẽ thất bại — mục *Tài nguyên VolumeSnapshotClass*, *Driver*,
  *DeletionPolicy*.
- Class mặc định là **mặc định theo từng CSI driver**: đánh dấu bằng annotation
  `snapshot.storage.kubernetes.io/is-default-class: "true"`; **nhiều class mặc định cùng tồn
  tại được** miễn mỗi cái gắn một driver riêng, nhưng **hai class mặc định cùng một driver thì
  việc tạo VolumeSnapshot thất bại** vì Kubernetes không quyết định được — mục *Các phụ thuộc
  của VolumeSnapshotClass*.
- Khi VolumeSnapshot không chỉ định class, Kubernetes chọn class mặc định có **CSI driver khớp
  với driver trong StorageClass của PVC** nguồn — cùng mục.
- `deletionPolicy` là `Retain` hay `Delete` quyết định số phận của snapshot bên dưới và của
  `VolumeSnapshotContent` khi VolumeSnapshot bị xóa — mục *DeletionPolicy*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ví dụ `driver: hostpath.csi.k8s.io` | driver mẫu của tài liệu CSI, không phải driver của lab | Lab 6a, khi chọn provisioner |
| *Tham số* | đặc thù từng `driver`, phải tra tài liệu driver | Lab 6b |

---

Tài liệu này mô tả khái niệm VolumeSnapshotClass trong Kubernetes. Bạn nên
làm quen trước với [volume snapshot](99-volume-snapshots-vi.md) và
[storage class](96-storage-classes-vi.md).

## Giới thiệu (Introduction)

Giống như StorageClass cung cấp cho quản trị viên một cách để mô tả các "lớp" (class)
lưu trữ mà họ cung cấp khi cấp phát (provision) một volume, VolumeSnapshotClass cung cấp
một cách để mô tả các "lớp" lưu trữ khi cấp phát một volume snapshot.

## Tài nguyên VolumeSnapshotClass (The VolumeSnapshotClass Resource)

Mỗi VolumeSnapshotClass chứa các trường `driver`, `deletionPolicy` và `parameters`,
được dùng khi một VolumeSnapshot thuộc lớp đó cần được
cấp phát động (dynamically provisioned).

Tên của một đối tượng VolumeSnapshotClass có ý nghĩa quan trọng, và là cách người dùng
yêu cầu một lớp cụ thể. Quản trị viên đặt tên và các tham số khác
của một lớp khi tạo các đối tượng VolumeSnapshotClass lần đầu, và các đối tượng này
không thể được cập nhật sau khi đã tạo.

> **Ghi chú:**
> Việc cài đặt các CRD là trách nhiệm của bản phân phối Kubernetes.
> Nếu không có sẵn các CRD cần thiết, việc tạo một VolumeSnapshotClass sẽ thất bại.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass
driver: hostpath.csi.k8s.io
deletionPolicy: Delete
parameters:
```

Quản trị viên có thể chỉ định một VolumeSnapshotClass mặc định cho các VolumeSnapshot
không yêu cầu gắn kết (bind) với một lớp cụ thể nào, bằng cách thêm annotation
`snapshot.storage.kubernetes.io/is-default-class: "true"`:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-hostpath-snapclass
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: hostpath.csi.k8s.io
deletionPolicy: Delete
parameters:
```

Nếu tồn tại nhiều CSI driver, có thể chỉ định một VolumeSnapshotClass mặc định
cho từng driver.

### Các phụ thuộc của VolumeSnapshotClass (VolumeSnapshotClass dependencies)

Khi bạn tạo một VolumeSnapshot mà không chỉ định VolumeSnapshotClass, Kubernetes
tự động chọn một VolumeSnapshotClass mặc định có CSI driver khớp với
CSI driver trong StorageClass của PVC.

Hành vi này cho phép nhiều đối tượng VolumeSnapshotClass mặc định cùng tồn tại trong một cluster,
miễn là mỗi đối tượng gắn với một CSI driver duy nhất.

Hãy luôn đảm bảo rằng chỉ có một VolumeSnapshotClass mặc định cho mỗi CSI driver. Nếu
nhiều đối tượng VolumeSnapshotClass mặc định được tạo cùng dùng một CSI driver,
việc tạo VolumeSnapshot sẽ thất bại vì Kubernetes không thể xác định nên dùng đối tượng nào.

### Driver

Các lớp volume snapshot có một driver xác định CSI volume plugin nào được
dùng để cấp phát các VolumeSnapshot. Trường này bắt buộc phải được chỉ định.

### DeletionPolicy

Các lớp volume snapshot có một [deletionPolicy](99-volume-snapshots-vi.md#delete).
Nó cho phép bạn cấu hình điều gì sẽ xảy ra với một VolumeSnapshotContent khi đối tượng
VolumeSnapshot gắn với nó bị xóa. deletionPolicy của một lớp volume snapshot có thể
là `Retain` hoặc `Delete`. Trường này bắt buộc phải được chỉ định.

Nếu deletionPolicy là `Delete`, snapshot lưu trữ bên dưới sẽ bị
xóa cùng với đối tượng VolumeSnapshotContent. Nếu deletionPolicy là `Retain`,
cả snapshot bên dưới lẫn VolumeSnapshotContent đều được giữ lại.

## Tham số (Parameters)

Các lớp volume snapshot có các tham số mô tả những volume snapshot thuộc
lớp volume snapshot đó. Các tham số được chấp nhận có thể khác nhau tùy theo
`driver`.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Cluster có hai VolumeSnapshotClass cùng đánh dấu mặc định. Khi nào điều đó chấp nhận được,
   khi nào nó làm hỏng việc tạo VolumeSnapshot? So sánh với cách StorageClass xử lý tình huống
   tương tự ở bài [96](96-storage-classes-vi.md).
2. Bạn tạo một VolumeSnapshot không đặt `volumeSnapshotClassName`. Kubernetes chọn class nào,
   và nó dựa vào thông tin gì để chọn?
3. Bạn cần đổi `deletionPolicy` của một VolumeSnapshotClass đang dùng từ `Delete` sang `Retain`.
   Làm thế nào?
4. Cluster lab của bạn hiện chưa cài các CRD của snapshot. Bạn `kubectl apply` một
   VolumeSnapshotClass. Kết quả?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Chấp nhận được khi mỗi class mặc định gắn với một CSI driver khác nhau** — bài nói rõ
   hành vi này cho phép nhiều VolumeSnapshotClass mặc định cùng tồn tại trong một cluster.
   **Hỏng khi hai class mặc định dùng chung một CSI driver**: lúc đó **việc tạo VolumeSnapshot
   sẽ thất bại** vì Kubernetes không xác định được nên dùng cái nào. Đây là chỗ trực giác từ
   StorageClass dẫn bạn đi sai: với StorageClass, nhiều class mặc định thì Kubernetes lặng lẽ
   **lấy cái tạo gần đây nhất**; với VolumeSnapshotClass thì nó **báo lỗi**.
2. Kubernetes **tự động chọn VolumeSnapshotClass mặc định có CSI driver khớp với CSI driver
   trong StorageClass của PVC** nguồn. Nói cách khác, quyết định không nằm ở snapshot mà nằm ở
   chỗ volume nguồn đang do driver nào quản lý.
3. **Không sửa được — phải tạo một VolumeSnapshotClass mới.** Bài nói thẳng: quản trị viên đặt
   tên và các tham số của một lớp khi tạo lần đầu, và **các đối tượng này không thể được cập
   nhật sau khi đã tạo**. Nếu class cũ đang là mặc định thì nhớ chuyển annotation mặc định
   sang class mới, và giữ đúng một class mặc định cho mỗi driver.
4. **Thất bại.** Ghi chú của bài nói rõ: việc cài các CRD là trách nhiệm của bản phân phối
   Kubernetes, và **nếu không có sẵn các CRD cần thiết thì việc tạo một VolumeSnapshotClass sẽ
   thất bại**. Đây là điều kiện đầu tiên phải kiểm tra ở Lab 6b, trước cả chuyện driver có hỗ
   trợ snapshot hay không.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
