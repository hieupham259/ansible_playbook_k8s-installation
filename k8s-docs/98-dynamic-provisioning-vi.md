# Cấp phát Volume động (Dynamic Volume Provisioning)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 5/16 · Kiểm chứng ở
[Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md).

Bài ngắn và gần như không có khái niệm mới sau bài [92](92-persistent-volumes-vi.md) và
[96](96-storage-classes-vi.md). Giá trị của nó nằm ở chỗ nói rõ **ranh giới trách nhiệm** giữa
quản trị viên và người dùng, và ở mục *Hành vi mặc định* — nơi nêu điều kiện thứ hai mà rất
nhiều người quên.

**Phải hiểu ở lần đọc này:**

- Cấp phát động giải quyết cái gì: bỏ hẳn bước admin gọi tay tới hệ thống lưu trữ rồi tạo đối
  tượng `PersistentVolume`; volume được tạo **khi người dùng tạo PVC** — đoạn mở đầu.
- Ranh giới trách nhiệm: **admin** tạo sẵn một hoặc nhiều StorageClass (mỗi class chỉ định
  provisioner và tham số, tên phải là DNS subdomain hợp lệ); **người dùng** chỉ chọn class qua
  trường `storageClassName` của PVC — mục *Bật cấp phát động*, *Sử dụng cấp phát động*.
- Xóa claim thì volume bị hủy — mục *Sử dụng cấp phát động*.
- Hành vi mặc định cần **hai** điều kiện cùng lúc: đánh dấu một StorageClass là mặc định bằng
  annotation `storageclass.kubernetes.io/is-default-class`, **và** bật admission controller
  `DefaultStorageClass` trên API server. Nhiều class mặc định thì Kubernetes lấy cái tạo gần
  đây nhất — mục *Hành vi mặc định*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hai ví dụ `provisioner: kubernetes.io/gce-pd` với `pd-standard` / `pd-ssd` | provisioner in-tree của GCE, lab không dùng | không cần |
| Annotation cũ `volume.beta.kubernetes.io/storage-class` | đã lỗi thời, chỉ còn gặp ở manifest cũ | không cần |
| *Nhận biết topology* | cần cluster nhiều zone; cơ chế thật nằm ở `volumeBindingMode` | bài [103](103-storage-capacity-vi.md) |

---

Cấp phát volume động (dynamic volume provisioning) cho phép các volume lưu trữ được tạo
theo nhu cầu (on-demand). Nếu không có cấp phát động, quản trị viên cluster phải tự tay
gọi tới nhà cung cấp cloud hoặc nhà cung cấp lưu trữ của họ để tạo các volume lưu trữ mới,
rồi tạo các [đối tượng `PersistentVolume`](92-persistent-volumes-vi.md)
để đại diện cho chúng trong Kubernetes. Tính năng cấp phát động loại bỏ nhu cầu
quản trị viên cluster phải cấp phát sẵn (pre-provision) lưu trữ. Thay vào đó, nó
tự động cấp phát lưu trữ khi người dùng tạo các
[đối tượng `PersistentVolumeClaim`](92-persistent-volumes-vi.md).

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

Để biết thêm chi tiết, xem khái niệm [Storage Classes](96-storage-classes-vi.md).

## Bật cấp phát động (Enabling Dynamic Provisioning)

Để bật cấp phát động, quản trị viên cluster cần tạo sẵn
một hoặc nhiều đối tượng StorageClass cho người dùng.
Các đối tượng StorageClass định nghĩa provisioner nào sẽ được dùng và các tham số nào
sẽ được truyền cho provisioner đó khi cấp phát động được kích hoạt.
Tên của một đối tượng StorageClass phải là một
[tên miền con DNS (DNS subdomain name)](17-names-vi.md#dns-subdomain-names) hợp lệ.

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
[Volume Binding Mode](96-storage-classes-vi.md#volume-binding-mode).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Trong quy trình cấp phát động, quản trị viên làm gì và người dùng làm gì? Ranh giới nằm ở
   trường nào của PVC?
2. Bạn đã đánh dấu một StorageClass là mặc định bằng annotation, nhưng PVC không đặt
   `storageClassName` vẫn không được cấp phát gì. Bài này nói bạn còn thiếu điều kiện nào?
3. Cluster có hai StorageClass cùng được đánh dấu mặc định. PVC không đặt `storageClassName`
   sẽ đi theo class nào?
4. Cluster lab của bạn **chưa có StorageClass và chưa có provisioner**, và bạn cần một volume
   bền vững ngay hôm nay, trước khi chạy Lab 6a. Theo bài này, cách còn lại là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Quản trị viên tạo sẵn các đối tượng StorageClass**, mỗi class chỉ định provisioner nào sẽ
   cấp phát và tham số nào được truyền cho provisioner đó. **Người dùng chỉ chọn class** bằng
   cách đưa tên class vào trường **`storageClassName`** của PersistentVolumeClaim. Đó chính là
   ranh giới: người dùng không phải biết lưu trữ được cấp phát ra sao, nhưng vẫn chọn được
   giữa nhiều "hương vị" lưu trữ.
2. Thiếu **admission controller `DefaultStorageClass` được bật trên API server**. Bài liệt kê
   hai việc phải làm cùng nhau: đánh dấu class là mặc định, *và* bảo đảm admission controller
   đó đang bật. Chính admission controller này mới là thứ tự động điền `storageClassName` vào
   PVC không khai gì; chỉ đặt annotation thôi thì không đủ.
3. **Class được tạo gần đây nhất.** Kubernetes không báo lỗi và cũng không chọn ngẫu nhiên —
   nó dùng StorageClass mặc định mới nhất.
4. Quay lại **cấp phát tĩnh**: tự tay gọi tới hệ thống lưu trữ để tạo volume, rồi tự tạo đối
   tượng `PersistentVolume` đại diện cho nó trong Kubernetes. Đó đúng là công việc mà cấp phát
   động sinh ra để loại bỏ, và cũng là lý do Lab 6a phải cài provisioner trước khi giai đoạn 6
   thực hành được.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
