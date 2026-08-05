# Cấp phát Volume động (Dynamic Volume Provisioning)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/>

Cấp phát volume động (dynamic volume provisioning) cho phép các volume lưu trữ được tạo
theo nhu cầu (on-demand). Nếu không có cấp phát động, quản trị viên cluster phải tự tay
gọi tới nhà cung cấp cloud hoặc nhà cung cấp lưu trữ của họ để tạo các volume lưu trữ mới,
rồi tạo các [đối tượng `PersistentVolume`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
để đại diện cho chúng trong Kubernetes. Tính năng cấp phát động loại bỏ nhu cầu
quản trị viên cluster phải cấp phát sẵn (pre-provision) lưu trữ. Thay vào đó, nó
tự động cấp phát lưu trữ khi người dùng tạo các
[đối tượng `PersistentVolumeClaim`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

## Bối cảnh (Background)

Việc triển khai cấp phát volume động dựa trên đối tượng API `StorageClass`
thuộc nhóm API `storage.k8s.io`. Quản trị viên cluster có thể định nghĩa bao nhiêu
đối tượng `StorageClass` tùy theo nhu cầu, mỗi đối tượng chỉ định một *volume plugin* (còn gọi là
*provisioner*) sẽ cấp phát volume, cùng tập tham số truyền cho
provisioner đó khi cấp phát.
Quản trị viên cluster có thể định nghĩa và cung cấp nhiều "hương vị" (flavor) lưu trữ khác nhau (từ
cùng một hoặc nhiều hệ thống lưu trữ khác nhau) trong một cluster, mỗi loại với một bộ
tham số tùy chỉnh. Thiết kế này cũng bảo đảm rằng người dùng cuối không phải lo lắng
về độ phức tạp và những chi tiết tinh tế của việc lưu trữ được cấp phát ra sao, nhưng vẫn
có khả năng lựa chọn giữa nhiều tùy chọn lưu trữ.

Để biết thêm chi tiết, xem khái niệm [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/).

## Bật cấp phát động (Enabling Dynamic Provisioning)

Để bật cấp phát động, quản trị viên cluster cần tạo sẵn
một hoặc nhiều đối tượng StorageClass cho người dùng.
Các đối tượng StorageClass định nghĩa provisioner nào sẽ được dùng và các tham số nào
sẽ được truyền cho provisioner đó khi cấp phát động được kích hoạt.
Tên của một đối tượng StorageClass phải là một
[tên miền con DNS (DNS subdomain name)](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names) hợp lệ.

Manifest sau đây tạo một storage class "slow" cấp phát các persistent disk
giống đĩa tiêu chuẩn (standard disk).

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

Manifest sau đây tạo một storage class "fast" cấp phát các persistent disk
giống SSD.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

## Sử dụng cấp phát động (Using Dynamic Provisioning)

Người dùng yêu cầu lưu trữ được cấp phát động bằng cách đưa một storage class vào
`PersistentVolumeClaim` của họ. Trước Kubernetes v1.6, việc này được thực hiện qua
annotation `volume.beta.kubernetes.io/storage-class`. Tuy nhiên, annotation này
đã bị loại bỏ dần (deprecated) kể từ v1.9. Giờ đây người dùng có thể và nên dùng
trường `storageClassName` của đối tượng `PersistentVolumeClaim`. Giá trị của
trường này phải khớp với tên của một `StorageClass` đã được quản trị viên
cấu hình (xem [Bật cấp phát động](#bật-cấp-phát-động-enabling-dynamic-provisioning)).

Ví dụ, để chọn storage class "fast", người dùng sẽ tạo
PersistentVolumeClaim sau:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast
  resources:
    requests:
      storage: 30Gi
```

Claim này dẫn tới việc một Persistent Disk giống SSD được cấp phát
tự động. Khi claim bị xóa, volume sẽ bị hủy.

## Hành vi mặc định (Defaulting Behavior)

Cấp phát động có thể được bật trên một cluster sao cho mọi claim đều được
cấp phát động nếu không có storage class nào được chỉ định. Quản trị viên cluster
có thể bật hành vi này bằng cách:

- Đánh dấu một đối tượng `StorageClass` là *mặc định (default)*.
- Bảo đảm rằng [admission controller `DefaultStorageClass`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass)
  được bật trên API server.

Quản trị viên có thể đánh dấu một `StorageClass` cụ thể là mặc định bằng cách thêm
[annotation `storageclass.kubernetes.io/is-default-class`](https://kubernetes.io/docs/reference/labels-annotations-taints/#storageclass-kubernetes-io-is-default-class) vào nó.
Khi một `StorageClass` mặc định tồn tại trong cluster và người dùng tạo một
`PersistentVolumeClaim` không chỉ định `storageClassName`, admission controller
`DefaultStorageClass` sẽ tự động thêm trường
`storageClassName` trỏ tới storage class mặc định.

Lưu ý rằng nếu bạn đặt annotation `storageclass.kubernetes.io/is-default-class`
thành true trên nhiều hơn một StorageClass trong cluster của bạn, và sau đó bạn
tạo một `PersistentVolumeClaim` không đặt `storageClassName`, Kubernetes
sẽ dùng StorageClass mặc định được tạo gần đây nhất.

## Nhận biết topology (Topology Awareness)

Trong các cluster [nhiều Zone (Multi-Zone)](https://kubernetes.io/docs/setup/best-practices/multiple-zones/), Pod có thể được phân bố trên nhiều
Zone trong một Region. Các backend lưu trữ chỉ nằm trong một Zone (Single-Zone) nên được cấp phát tại các Zone nơi
Pod được lập lịch (schedule). Điều này có thể đạt được bằng cách đặt
[Volume Binding Mode](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode).
