# Ảnh chụp nhanh Volume (Volume Snapshots)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/volume-snapshots/>

Trong Kubernetes, một _VolumeSnapshot_ đại diện cho một ảnh chụp nhanh (snapshot) của một volume
trên hệ thống lưu trữ. Tài liệu này giả định rằng bạn đã quen thuộc với
[persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) trong Kubernetes.

## Giới thiệu (Introduction)

Tương tự như cách các tài nguyên API `PersistentVolume` và `PersistentVolumeClaim` được
dùng để cấp phát volume cho người dùng và quản trị viên, các tài nguyên API `VolumeSnapshotContent`
và `VolumeSnapshot` được cung cấp để tạo các snapshot của volume cho
người dùng và quản trị viên.

Một `VolumeSnapshotContent` là một snapshot được chụp từ một volume trong cluster mà
quản trị viên đã cấp phát. Nó là một tài nguyên trong cluster, giống như
PersistentVolume là một tài nguyên của cluster.

Một `VolumeSnapshot` là một yêu cầu tạo snapshot của một volume do người dùng đưa ra. Nó tương tự
như một PersistentVolumeClaim.

`VolumeSnapshotClass` cho phép bạn chỉ định các thuộc tính khác nhau thuộc về một
`VolumeSnapshot`. Các thuộc tính này có thể khác nhau giữa các snapshot được chụp từ cùng một
volume trên hệ thống lưu trữ, và do đó không thể được diễn đạt bằng cùng
`StorageClass` của một `PersistentVolumeClaim`.

Snapshot của volume cung cấp cho người dùng Kubernetes một cách chuẩn hóa để sao chép nội dung
của một volume tại một thời điểm cụ thể mà không cần tạo một volume hoàn toàn mới. Chức năng
này cho phép, chẳng hạn, quản trị viên cơ sở dữ liệu sao lưu (backup) cơ sở dữ liệu trước khi
thực hiện các thao tác chỉnh sửa hoặc xóa.

Người dùng cần lưu ý những điểm sau khi sử dụng tính năng này:

- Các đối tượng API `VolumeSnapshot`, `VolumeSnapshotContent`, và `VolumeSnapshotClass`
  là các CRDs, không phải
  là một phần của API lõi (core API).
- Hỗ trợ `VolumeSnapshot` chỉ khả dụng với các CSI driver.
- Trong quy trình triển khai `VolumeSnapshot`, nhóm Kubernetes cung cấp
  một snapshot controller để triển khai vào control plane, và một container trợ giúp
  dạng sidecar gọi là csi-snapshotter để triển khai cùng với CSI driver.
  Snapshot controller theo dõi các đối tượng `VolumeSnapshot` và `VolumeSnapshotContent`
  và chịu trách nhiệm tạo và xóa đối tượng `VolumeSnapshotContent`.
  Sidecar csi-snapshotter theo dõi các đối tượng `VolumeSnapshotContent` và kích hoạt
  các thao tác `CreateSnapshot` và `DeleteSnapshot` trên một endpoint CSI.
- Ngoài ra còn có một validating webhook server cung cấp việc kiểm tra chặt chẽ hơn trên
  các đối tượng snapshot. Thành phần này nên được các bản phân phối (distro) Kubernetes cài đặt cùng với
  snapshot controller và các CRD, chứ không phải các CSI driver. Nó nên được cài đặt trong mọi
  cluster Kubernetes có bật tính năng snapshot.
- Các CSI driver có thể đã hoặc chưa triển khai chức năng snapshot của volume.
  Các CSI driver đã cung cấp hỗ trợ cho snapshot của volume nhiều khả năng sẽ dùng
  csi-snapshotter. Xem [tài liệu CSI Driver](https://kubernetes-csi.github.io/docs/) để biết chi tiết.
- Việc cài đặt các CRD và snapshot controller là trách nhiệm của bản phân phối Kubernetes.

Với các trường hợp sử dụng nâng cao, chẳng hạn tạo snapshot theo nhóm (group snapshot) cho nhiều volume, xem tài liệu bên ngoài
[CSI Volume Group Snapshot](https://kubernetes-csi.github.io/docs/group-snapshot-restore-feature.html).

## Vòng đời của một volume snapshot và volume snapshot content (Lifecycle of a volume snapshot and volume snapshot content)

`VolumeSnapshotContent` là các tài nguyên trong cluster. `VolumeSnapshot` là các yêu cầu
đối với những tài nguyên đó. Sự tương tác giữa `VolumeSnapshotContent` và `VolumeSnapshot`
tuân theo vòng đời sau:

### Cấp phát Volume Snapshot (Provisioning Volume Snapshot)

Có hai cách snapshot có thể được cấp phát: cấp phát sẵn (pre-provisioned) hoặc cấp phát động (dynamically provisioned).

#### Cấp phát sẵn (Pre-provisioned) {#static}

Quản trị viên cluster tạo một số `VolumeSnapshotContent`. Chúng mang thông tin chi tiết
của snapshot volume thực trên hệ thống lưu trữ, sẵn sàng cho người dùng cluster sử dụng.
Chúng tồn tại trong Kubernetes API và sẵn sàng để được tiêu thụ.

#### Động (Dynamic)

Thay vì dùng một snapshot có sẵn từ trước, bạn có thể yêu cầu một snapshot được chụp
động từ một PersistentVolumeClaim. [VolumeSnapshotClass](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/)
chỉ định các tham số đặc thù của nhà cung cấp lưu trữ sẽ được dùng khi chụp snapshot.

### Ràng buộc (Binding)

Snapshot controller xử lý việc ràng buộc một đối tượng `VolumeSnapshot` với một
đối tượng `VolumeSnapshotContent` phù hợp, trong cả hai kịch bản cấp phát sẵn và cấp phát động.
Sự ràng buộc là một ánh xạ một-một.

Trong trường hợp ràng buộc kiểu cấp phát sẵn, VolumeSnapshot sẽ vẫn ở trạng thái chưa ràng buộc cho tới khi
đối tượng VolumeSnapshotContent được yêu cầu được tạo ra.

### Bảo vệ PersistentVolumeClaim đang làm nguồn Snapshot (Persistent Volume Claim as Snapshot Source Protection)

Mục đích của cơ chế bảo vệ này là bảo đảm rằng các đối tượng API
PersistentVolumeClaim
đang được sử dụng không bị xóa khỏi hệ thống trong khi một snapshot đang được chụp từ nó
(vì điều này có thể dẫn tới mất dữ liệu).

Trong khi một snapshot của PersistentVolumeClaim đang được chụp, PersistentVolumeClaim đó
đang được sử dụng (in-use). Nếu bạn xóa một đối tượng API PersistentVolumeClaim đang được sử dụng
làm nguồn của snapshot, đối tượng PersistentVolumeClaim đó không bị xóa ngay lập tức. Thay vào đó,
việc xóa đối tượng PersistentVolumeClaim bị hoãn lại cho tới khi snapshot ở trạng thái readyToUse hoặc bị hủy bỏ.

### Xóa (Delete)

Việc xóa được kích hoạt bằng cách xóa đối tượng `VolumeSnapshot`, và `DeletionPolicy`
sẽ được tuân theo. Nếu `DeletionPolicy` là `Delete`, thì snapshot lưu trữ bên dưới
sẽ bị xóa cùng với đối tượng `VolumeSnapshotContent`. Nếu `DeletionPolicy` là
`Retain`, thì cả snapshot bên dưới và `VolumeSnapshotContent` đều được giữ lại.

## Các VolumeSnapshot (VolumeSnapshots)

Mỗi VolumeSnapshot chứa một spec và một status.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: new-snapshot-test
spec:
  volumeSnapshotClassName: csi-hostpath-snapclass
  source:
    persistentVolumeClaimName: pvc-test
```

`persistentVolumeClaimName` là tên của PersistentVolumeClaim làm nguồn dữ liệu
cho snapshot. Trường này là bắt buộc khi cấp phát động một snapshot.

Một volume snapshot có thể yêu cầu một class cụ thể bằng cách chỉ định tên của một
[VolumeSnapshotClass](https://kubernetes.io/docs/concepts/storage/volume-snapshot-classes/)
qua thuộc tính `volumeSnapshotClassName`. Nếu không đặt gì, thì
class mặc định sẽ được dùng nếu có.

Với các snapshot cấp phát sẵn, bạn cần chỉ định một `volumeSnapshotContentName`
làm nguồn cho snapshot như trong ví dụ sau. Trường nguồn
`volumeSnapshotContentName` là bắt buộc đối với các snapshot cấp phát sẵn.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: test-snapshot
spec:
  source:
    volumeSnapshotContentName: test-content
```

## Các VolumeSnapshotContent (Volume Snapshot Contents)

Mỗi VolumeSnapshotContent chứa một spec và status. Trong cấp phát động,
snapshot common controller tạo ra các đối tượng `VolumeSnapshotContent`. Dưới đây là một ví dụ:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: snapcontent-72d9a349-aacd-42d2-a240-d775650d2455
spec:
  deletionPolicy: Delete
  driver: hostpath.csi.k8s.io
  source:
    volumeHandle: ee0cfb94-f8d4-11e9-b2d8-0242ac110002
  sourceVolumeMode: Filesystem
  volumeSnapshotClassName: csi-hostpath-snapclass
  volumeSnapshotRef:
    name: new-snapshot-test
    namespace: default
    uid: 72d9a349-aacd-42d2-a240-d775650d2455
```

`volumeHandle` là định danh duy nhất của volume được tạo trên backend
lưu trữ và được CSI driver trả về trong quá trình tạo volume. Trường này
là bắt buộc khi cấp phát động một snapshot.
Nó chỉ định nguồn volume của snapshot.

Với các snapshot cấp phát sẵn, bạn (với vai trò quản trị viên cluster) chịu trách nhiệm
tạo đối tượng `VolumeSnapshotContent` như sau.

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: new-snapshot-content-test
spec:
  deletionPolicy: Delete
  driver: hostpath.csi.k8s.io
  source:
    snapshotHandle: 7bdd0de3-aaeb-11e8-9aae-0242ac110002
  sourceVolumeMode: Filesystem
  volumeSnapshotRef:
    name: new-snapshot-test
    namespace: default
```

`snapshotHandle` là định danh duy nhất của volume snapshot được tạo trên
backend lưu trữ. Trường này là bắt buộc đối với các snapshot cấp phát sẵn.
Nó chỉ định CSI snapshot id trên hệ thống lưu trữ mà
`VolumeSnapshotContent` này đại diện.

`sourceVolumeMode` là chế độ (mode) của volume mà snapshot được chụp từ đó. Giá trị
của trường `sourceVolumeMode` có thể là `Filesystem` hoặc `Block`. Nếu
chế độ của volume nguồn không được chỉ định, Kubernetes coi snapshot như thể
chế độ của volume nguồn là không xác định.

`volumeSnapshotRef` là tham chiếu tới `VolumeSnapshot` tương ứng. Lưu ý rằng
khi `VolumeSnapshotContent` đang được tạo dưới dạng một snapshot cấp phát sẵn,
`VolumeSnapshot` được tham chiếu trong `volumeSnapshotRef` có thể chưa tồn tại.

## Chuyển đổi volume mode của một Snapshot (Converting the volume mode of a Snapshot) {#convert-volume-mode}

Nếu API `VolumeSnapshots` được cài đặt trên cluster của bạn hỗ trợ trường `sourceVolumeMode`,
thì API này có khả năng ngăn người dùng không được phép chuyển đổi
chế độ của một volume.

Để kiểm tra xem cluster của bạn có khả năng dùng tính năng này hay không, chạy lệnh sau:

```yaml
$ kubectl get crd volumesnapshotcontent -o yaml
```

Nếu bạn muốn cho phép người dùng tạo một `PersistentVolumeClaim` từ một
`VolumeSnapshot` có sẵn, nhưng với volume mode khác với nguồn, annotation
`snapshot.storage.kubernetes.io/allow-volume-mode-change: "true"` cần được thêm vào
`VolumeSnapshotContent` tương ứng với `VolumeSnapshot` đó.

Với các snapshot cấp phát sẵn, `spec.sourceVolumeMode` cần được
quản trị viên cluster điền vào.

Một ví dụ tài nguyên `VolumeSnapshotContent` có bật tính năng này sẽ trông như sau:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: new-snapshot-content-test
  annotations:
    - snapshot.storage.kubernetes.io/allow-volume-mode-change: "true"
spec:
  deletionPolicy: Delete
  driver: hostpath.csi.k8s.io
  source:
    snapshotHandle: 7bdd0de3-aaeb-11e8-9aae-0242ac110002
  sourceVolumeMode: Filesystem
  volumeSnapshotRef:
    name: new-snapshot-test
    namespace: default
```

## Cấp phát Volume từ Snapshot (Provisioning Volumes from Snapshots)

Bạn có thể cấp phát một volume mới, được nạp sẵn dữ liệu từ một snapshot, bằng cách dùng
trường _dataSource_ trong đối tượng `PersistentVolumeClaim`.

Để biết thêm chi tiết, xem
[Volume Snapshot và khôi phục Volume từ Snapshot](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#volume-snapshot-and-restore-volume-from-snapshot-support).
