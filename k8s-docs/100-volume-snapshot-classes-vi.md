# Các lớp Volume Snapshot (Volume Snapshot Classes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/>

Tài liệu này mô tả khái niệm VolumeSnapshotClass trong Kubernetes. Bạn nên
làm quen trước với [volume snapshot](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) và
[storage class](https://kubernetes.io/docs/concepts/storage/storage-classes).

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

Các lớp volume snapshot có một [deletionPolicy](https://kubernetes.io/docs/concepts/storage/volume-snapshots/#delete).
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
