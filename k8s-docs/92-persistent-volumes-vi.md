# Volume bền vững (Persistent Volumes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/persistent-volumes/>
>
> Điều phối lưu trữ (Storage orchestration): Tự động mount hệ thống lưu trữ mà bạn chọn,
> dù đó là lưu trữ cục bộ (local storage), từ một nhà cung cấp cloud công cộng,
> hay một hệ thống lưu trữ mạng như iSCSI hoặc NFS.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 3/16 · Kiểm chứng ở
[Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md).

Đây là **bài xương sống của cả giai đoạn 6**. Bốn bài sau (StorageClass, cấp phát động,
snapshot, clone) đều chỉ là các nhánh mọc ra từ vòng đời PV/PVC mô tả ở đây. Bài dài, nhưng
phần lớn độ dài nằm ở các bảng liệt kê plugin và các mục ví dụ YAML; khung khái niệm thật sự
gói gọn trong mục *Vòng đời của một volume và claim* cộng hai mục *Persistent Volumes* và
*PersistentVolumeClaims*.

**Phải hiểu ở lần đọc này:**

- Vòng đời đầy đủ: cấp phát **tĩnh** (admin tạo sẵn PV) hay **động** (theo StorageClass) →
  ràng buộc → sử dụng → thu hồi. Ràng buộc PVC↔PV là **một-một và độc quyền** qua `claimRef`;
  không có volume khớp thì claim nằm unbound **vô thời hạn**. Ngoài ra PVC đang được Pod dùng
  thì việc xóa bị hoãn bằng finalizer — mục *Vòng đời của một volume và claim*.
- Bốn access mode và ý nghĩa thật của chúng: `ReadWriteOnce` là ràng buộc **theo node** chứ
  không phải theo Pod (nhiều Pod trên cùng node vẫn dùng chung được), chỉ `ReadWriteOncePod`
  mới giới hạn còn một Pod; access mode dùng để **khớp** PVC với PV chứ **không** cưỡng chế
  chống ghi — mục *Các chế độ truy cập*.
- Ba reclaim policy và khác biệt khi bạn xóa PVC: `Retain` giữ PV lại ở pha `Released` và phải
  dọn tay, `Delete` xóa cả đối tượng PV lẫn tài sản lưu trữ bên ngoài; PV cấp phát động **kế
  thừa** reclaim policy của StorageClass, mặc định là `Delete` — mục *Thu hồi*, *Chính sách thu
  hồi*, *Pha*.
- Mở rộng dung lượng: chỉ được khi `allowVolumeExpansion: true` trên storage class, thao tác là
  **sửa PVC** chứ không sửa PV, không bao giờ sinh PV mới, và **không thu nhỏ** được xuống dưới
  `.status.capacity` — mục *Mở rộng Persistent Volume Claim*.
- Ba trạng thái khác nhau của `storageClassName` trên PVC: đặt tên một class, đặt `""` (tự vô
  hiệu hóa cấp phát động, chỉ bind PV không class), và **bỏ trống** (có thể được gán class mặc
  định về sau) — mục *Lớp* của PVC và *Gán StorageClass mặc định hồi tố*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Recycle* và mẫu Pod tái chế | đã lỗi thời, chỉ còn `nfs` và `hostPath` hỗ trợ | không cần |
| *Finalizer bảo vệ việc xóa PersistentVolume* | chi tiết của external provisioner | Lab 6a, khi đã có provisioner thật |
| *Các loại Persistent Volume* | phần lớn là plugin cloud đã bị gỡ hoặc đã di trú sang CSI | không cần |
| *Node Affinity* và *Cập nhật node affinity* | chỉ cần cho volume `local`, và affinity chưa học | giai đoạn 7, bài [138](138-assign-pod-node-vi.md) |
| *Hỗ trợ raw block volume* | ứng dụng phải tự xử lý thiết bị khối thô | không cần |
| *Hỗ trợ Volume Snapshot…*, *Nhân bản volume*, *Volume populator và nguồn dữ liệu* | cần CSI driver hỗ trợ | bài [99](99-volume-snapshots-vi.md)–[102](102-volume-populators-vi.md), Lab 6b |
| *Theo dõi PVC không được sử dụng* | tính năng alpha, chưa bật trong cluster lab | không cần |
| *Viết cấu hình khả chuyển* | lời khuyên cho người đóng gói ứng dụng, không phải cho admin | không cần |

---

Tài liệu này mô tả _persistent volume_ (volume bền vững) trong Kubernetes. Bạn nên làm quen trước với
[volume](91-volumes-vi.md), [StorageClass](96-storage-classes-vi.md)
và [VolumeAttributesClass](97-volume-attributes-classes-vi.md).

## Giới thiệu (Introduction)

Quản lý lưu trữ là một bài toán tách biệt với quản lý các thực thể tính toán (compute instance).
Hệ thống con PersistentVolume cung cấp một API cho người dùng và quản trị viên,
trừu tượng hóa chi tiết về cách lưu trữ được cung cấp khỏi cách nó được sử dụng.
Để làm điều này, chúng ta có hai tài nguyên API mới: PersistentVolume và PersistentVolumeClaim.

Một _PersistentVolume_ (PV) là một phần lưu trữ trong cluster đã được quản trị viên
cấp phát (provision) sẵn, hoặc được cấp phát động (dynamically provisioned) bằng
[Storage Class](96-storage-classes-vi.md). Nó là một tài nguyên trong
cluster, giống như node cũng là một tài nguyên của cluster. PV là các volume plugin tương tự
Volume, nhưng có vòng đời độc lập với bất kỳ Pod riêng lẻ nào sử dụng PV đó.
Đối tượng API này lưu giữ các chi tiết triển khai của phần lưu trữ, có thể là
NFS, iSCSI, hay một hệ thống lưu trữ riêng của nhà cung cấp cloud.

Một _PersistentVolumeClaim_ (PVC) là một yêu cầu lưu trữ của người dùng. Nó tương tự
như Pod. Pod tiêu thụ tài nguyên node còn PVC tiêu thụ tài nguyên PV. Pod có thể
yêu cầu mức tài nguyên cụ thể (CPU và bộ nhớ). Claim có thể yêu cầu kích thước
và các chế độ truy cập (access mode) cụ thể (ví dụ: chúng có thể được mount ở chế độ ReadWriteOnce,
ReadOnlyMany, ReadWriteMany, hoặc ReadWriteOncePod — xem [Các chế độ truy cập](#access-modes)).

Mặc dù PersistentVolumeClaim cho phép người dùng tiêu thụ tài nguyên lưu trữ một cách trừu tượng,
người dùng thường cần các PersistentVolume với những thuộc tính khác nhau — chẳng hạn
hiệu năng — cho những bài toán khác nhau. Quản trị viên cluster cần có khả năng
cung cấp nhiều loại PersistentVolume khác nhau không chỉ về kích thước và chế độ
truy cập, mà không để lộ cho người dùng chi tiết về cách các volume đó được triển khai.
Cho các nhu cầu này, có tài nguyên _StorageClass_.

Xem [hướng dẫn chi tiết kèm ví dụ hoạt động](https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage).

## Vòng đời của một volume và claim (Lifecycle of a volume and claim)

PV là tài nguyên trong cluster. PVC là yêu cầu đối với những tài nguyên đó và cũng đóng
vai trò như phiếu giữ chỗ (claim check) cho tài nguyên. Tương tác giữa PV và PVC tuân theo vòng đời sau:

### Cấp phát (Provisioning) {#provisioning}

Có hai cách để cấp phát PV: tĩnh (static) hoặc động (dynamic).

#### Tĩnh (Static)

Quản trị viên cluster tạo sẵn một số PV. Chúng mang chi tiết của phần lưu trữ
thực, sẵn sàng cho người dùng cluster sử dụng. Chúng tồn tại trong
Kubernetes API và có thể được tiêu thụ.

#### Động (Dynamic)

Khi không có PV tĩnh nào do quản trị viên tạo khớp với PersistentVolumeClaim của người dùng,
cluster có thể thử cấp phát động một volume dành riêng cho PVC đó.
Việc cấp phát này dựa trên StorageClass: PVC phải yêu cầu một
[storage class](96-storage-classes-vi.md) và
quản trị viên phải đã tạo và cấu hình class đó thì việc cấp phát động
mới diễn ra. Các claim yêu cầu class `""` thực chất tự vô hiệu hóa
cấp phát động cho chính chúng.

Để bật cấp phát lưu trữ động dựa trên storage class, quản trị viên cluster
cần bật [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass)
`DefaultStorageClass` trên API server. Có thể làm điều này, ví dụ, bằng cách đảm bảo `DefaultStorageClass` nằm
trong danh sách giá trị có thứ tự, phân tách bằng dấu phẩy của cờ `--enable-admission-plugins` của
thành phần API server. Để biết thêm về các cờ dòng lệnh của API server,
xem tài liệu [kube-apiserver](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/).

### Ràng buộc (Binding) {#binding}

Người dùng tạo — hoặc trong trường hợp cấp phát động thì đã tạo sẵn —
một PersistentVolumeClaim với một lượng lưu trữ cụ thể được yêu cầu cùng
các chế độ truy cập nhất định. Một vòng lặp điều khiển (control loop) trong control plane theo dõi các PVC mới, tìm
một PV khớp (nếu có), và ràng buộc (bind) chúng với nhau. Nếu một PV được cấp phát động
cho một PVC mới, vòng lặp sẽ luôn ràng buộc PV đó với PVC đó. Nếu không,
người dùng sẽ luôn nhận được ít nhất những gì họ yêu cầu, nhưng volume có thể
lớn hơn mức được yêu cầu. Một khi đã ràng buộc, các ràng buộc của PersistentVolumeClaim là độc quyền,
bất kể chúng được ràng buộc theo cách nào. Ràng buộc PVC với PV là ánh xạ một-một,
sử dụng ClaimRef — một ràng buộc hai chiều giữa PersistentVolume
và PersistentVolumeClaim.

Claim sẽ ở trạng thái chưa ràng buộc (unbound) vô thời hạn nếu không tồn tại volume khớp.
Claim sẽ được ràng buộc khi có volume khớp trở nên khả dụng. Ví dụ, một
cluster được cấp phát nhiều PV 50Gi sẽ không khớp với một PVC yêu cầu 100Gi.
PVC đó chỉ có thể được ràng buộc khi một PV 100Gi được thêm vào cluster.

### Sử dụng (Using)

Pod sử dụng claim như là volume. Cluster xem xét claim để tìm volume đã được
ràng buộc và mount volume đó cho Pod. Với các volume hỗ trợ nhiều
chế độ truy cập, người dùng chỉ định chế độ mong muốn khi sử dụng claim của họ
như một volume trong Pod.

Một khi người dùng có một claim và claim đó đã được ràng buộc, PV được ràng buộc thuộc về
người dùng chừng nào họ còn cần nó. Người dùng lên lịch Pod và truy cập các PV
đã claim bằng cách thêm mục `persistentVolumeClaim` vào khối `volumes` của Pod.
Xem [Dùng claim làm volume](#claims-as-volumes) để biết thêm chi tiết.

### Bảo vệ đối tượng lưu trữ đang được sử dụng (Storage Object in Use Protection)

Mục đích của tính năng Bảo vệ đối tượng lưu trữ đang được sử dụng là đảm bảo rằng
các PersistentVolumeClaim (PVC) đang được một Pod sử dụng và các PersistentVolume (PV)
đang ràng buộc với PVC không bị xóa khỏi hệ thống, vì điều đó có thể dẫn đến mất dữ liệu.

> **Ghi chú:**
> PVC được coi là đang được một Pod sử dụng khi tồn tại một đối tượng Pod đang dùng PVC đó.

Nếu người dùng xóa một PVC đang được một Pod sử dụng, PVC không bị xóa ngay lập tức.
Việc xóa PVC được hoãn lại cho đến khi PVC không còn được bất kỳ Pod nào sử dụng. Tương tự,
nếu quản trị viên xóa một PV đang ràng buộc với một PVC, PV không bị xóa ngay lập tức.
Việc xóa PV được hoãn lại cho đến khi PV không còn ràng buộc với PVC nào.

Bạn có thể thấy một PVC đang được bảo vệ khi trạng thái của PVC là `Terminating` và
danh sách `Finalizers` bao gồm `kubernetes.io/pvc-protection`:

```shell
kubectl describe pvc hostpath
Name:          hostpath
Namespace:     default
StorageClass:  example-hostpath
Status:        Terminating
Volume:
Labels:        <none>
Annotations:   volume.beta.kubernetes.io/storage-class=example-hostpath
               volume.beta.kubernetes.io/storage-provisioner=example.com/hostpath
Finalizers:    [kubernetes.io/pvc-protection]
...
```

Bạn cũng có thể thấy một PV đang được bảo vệ khi trạng thái của PV là `Terminating` và
danh sách `Finalizers` bao gồm `kubernetes.io/pv-protection`:

```shell
kubectl describe pv task-pv-volume
Name:            task-pv-volume
Labels:          type=local
Annotations:     <none>
Finalizers:      [kubernetes.io/pv-protection]
StorageClass:    standard
Status:          Terminating
Claim:
Reclaim Policy:  Delete
Access Modes:    RWO
Capacity:        1Gi
Message:
Source:
    Type:          HostPath (bare host directory volume)
    Path:          /tmp/data
    HostPathType:
Events:            <none>
```

### Thu hồi (Reclaiming) {#reclaiming}

Khi người dùng đã dùng xong volume của mình, họ có thể xóa đối tượng PVC khỏi
API, việc này cho phép thu hồi (reclaim) tài nguyên. Chính sách thu hồi (reclaim policy) của một PersistentVolume
cho cluster biết phải làm gì với volume sau khi nó được giải phóng khỏi claim.
Hiện tại, volume có thể được Giữ lại (Retained), Tái chế (Recycled), hoặc Xóa (Deleted).

#### Retain

Chính sách thu hồi `Retain` cho phép thu hồi tài nguyên một cách thủ công.
Khi PersistentVolumeClaim bị xóa, PersistentVolume vẫn tồn tại
và volume được coi là "đã giải phóng" (released). Nhưng nó chưa sẵn sàng cho
một claim khác vì dữ liệu của người claim trước đó vẫn còn trên volume.
Quản trị viên có thể thu hồi volume thủ công theo các bước sau.

1. Xóa PersistentVolume. Tài sản lưu trữ (storage asset) liên kết trong hạ tầng bên ngoài
   vẫn tồn tại sau khi PV bị xóa.
1. Tự dọn dẹp dữ liệu trên tài sản lưu trữ liên kết một cách phù hợp.
1. Tự xóa tài sản lưu trữ liên kết.

Nếu bạn muốn tái sử dụng cùng tài sản lưu trữ đó, hãy tạo một PersistentVolume mới với
cùng định nghĩa tài sản lưu trữ.

#### Delete

Với các volume plugin hỗ trợ chính sách thu hồi `Delete`, việc xóa sẽ loại bỏ
cả đối tượng PersistentVolume khỏi Kubernetes lẫn tài sản lưu trữ
liên kết trong hạ tầng bên ngoài. Các volume được cấp phát động
kế thừa [chính sách thu hồi của StorageClass của chúng](#reclaim-policy),
mặc định là `Delete`. Quản trị viên nên cấu hình StorageClass
theo kỳ vọng của người dùng; nếu không, PV phải được sửa (edit) hoặc
vá (patch) sau khi được tạo. Xem
[Thay đổi chính sách thu hồi của một PersistentVolume](194-change-pv-reclaim-policy-vi.md).

#### Recycle

> **Cảnh báo:**
> Chính sách thu hồi `Recycle` đã bị loại bỏ dần (deprecated). Thay vào đó, cách tiếp cận
> được khuyến nghị là sử dụng cấp phát động.

Nếu được volume plugin bên dưới hỗ trợ, chính sách thu hồi `Recycle` thực hiện
một thao tác xóa cơ bản (`rm -rf /thevolume/*`) trên volume và làm cho nó khả dụng trở lại cho một claim mới.

Tuy nhiên, quản trị viên có thể cấu hình một mẫu Pod tái chế (recycler Pod template) tùy chỉnh bằng
các tham số dòng lệnh của Kubernetes controller manager như mô tả trong
[tài liệu tham khảo](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/).
Mẫu Pod tái chế tùy chỉnh phải chứa phần khai báo `volumes`, như
trong ví dụ dưới đây:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pv-recycler
  namespace: default
spec:
  restartPolicy: Never
  volumes:
  - name: vol
    hostPath:
      path: /any/path/it/will/be/replaced
  containers:
  - name: pv-recycler
    image: "registry.k8s.io/busybox"
    command: ["/bin/sh", "-c", "test -e /scrub && rm -rf /scrub/..?* /scrub/.[!.]* /scrub/*  && test -z \"$(ls -A /scrub)\" || exit 1"]
    volumeMounts:
    - name: vol
      mountPath: /scrub
```

Tuy nhiên, đường dẫn cụ thể được chỉ định trong phần `volumes` của mẫu Pod tái chế
tùy chỉnh sẽ được thay thế bằng đường dẫn cụ thể của volume đang được tái chế.

### Finalizer bảo vệ việc xóa PersistentVolume (PersistentVolume deletion protection finalizer)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]`

Có thể thêm finalizer vào một PersistentVolume để đảm bảo rằng các PersistentVolume
có chính sách thu hồi `Delete` chỉ bị xóa sau khi phần lưu trữ nền (backing storage) đã bị xóa.

Finalizer `external-provisioner.volume.kubernetes.io/finalizer` (được giới thiệu
từ v1.31) được thêm vào cả các volume CSI được cấp phát động lẫn được cấp phát tĩnh.

Finalizer `kubernetes.io/pv-controller` (được giới thiệu từ v1.31) được thêm vào
các volume của plugin in-tree được cấp phát động và được bỏ qua đối với các volume
của plugin in-tree được cấp phát tĩnh.

Sau đây là ví dụ về một volume của plugin in-tree được cấp phát động:

```shell
kubectl describe pv pvc-74a498d6-3929-47e8-8c02-078c1ece4d78
Name:            pvc-74a498d6-3929-47e8-8c02-078c1ece4d78
Labels:          <none>
Annotations:     kubernetes.io/createdby: vsphere-volume-dynamic-provisioner
                 pv.kubernetes.io/bound-by-controller: yes
                 pv.kubernetes.io/provisioned-by: kubernetes.io/vsphere-volume
Finalizers:      [kubernetes.io/pv-protection kubernetes.io/pv-controller]
StorageClass:    vcp-sc
Status:          Bound
Claim:           default/vcp-pvc-1
Reclaim Policy:  Delete
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        1Gi
Node Affinity:   <none>
Message:
Source:
    Type:               vSphereVolume (a Persistent Disk resource in vSphere)
    VolumePath:         [vsanDatastore] d49c4a62-166f-ce12-c464-020077ba5d46/kubernetes-dynamic-pvc-74a498d6-3929-47e8-8c02-078c1ece4d78.vmdk
    FSType:             ext4
    StoragePolicyName:  vSAN Default Storage Policy
Events:                 <none>
```

Finalizer `external-provisioner.volume.kubernetes.io/finalizer` được thêm cho các volume CSI.
Sau đây là một ví dụ:

```shell
Name:            pvc-2f0bab97-85a8-4552-8044-eb8be45cf48d
Labels:          <none>
Annotations:     pv.kubernetes.io/provisioned-by: csi.vsphere.vmware.com
Finalizers:      [kubernetes.io/pv-protection external-provisioner.volume.kubernetes.io/finalizer]
StorageClass:    fast
Status:          Bound
Claim:           demo-app/nginx-logs
Reclaim Policy:  Delete
Access Modes:    RWO
VolumeMode:      Filesystem
Capacity:        200Mi
Node Affinity:   <none>
Message:
Source:
    Type:              CSI (a Container Storage Interface (CSI) volume source)
    Driver:            csi.vsphere.vmware.com
    FSType:            ext4
    VolumeHandle:      44830fa8-79b4-406b-8b58-621ba25353fd
    ReadOnly:          false
    VolumeAttributes:      storage.kubernetes.io/csiProvisionerIdentity=1648442357185-8081-csi.vsphere.vmware.com
                           type=vSphere CNS Block Volume
Events:                <none>
```

Khi cờ tính năng `CSIMigration{provider}` được bật cho một volume plugin in-tree cụ thể,
finalizer `kubernetes.io/pv-controller` được thay thế bằng
finalizer `external-provisioner.volume.kubernetes.io/finalizer`.

Các finalizer đảm bảo đối tượng PV chỉ bị loại bỏ sau khi volume đã bị xóa
khỏi backend lưu trữ, với điều kiện chính sách thu hồi của PV là `Delete`. Điều này
cũng đảm bảo volume được xóa khỏi backend lưu trữ bất kể thứ tự
xóa PV và PVC.

### Dành riêng một PersistentVolume (Reserving a PersistentVolume)

Control plane có thể [ràng buộc PersistentVolumeClaim với các PersistentVolume khớp](#binding)
trong cluster. Tuy nhiên, nếu bạn muốn một PVC ràng buộc với một PV cụ thể, bạn cần ràng buộc trước (pre-bind) chúng.

Bằng cách chỉ định một PersistentVolume trong PersistentVolumeClaim, bạn khai báo một ràng buộc
giữa PV và PVC cụ thể đó. Nếu PersistentVolume tồn tại và chưa dành riêng
cho PersistentVolumeClaim nào qua trường `claimRef` của nó, thì PersistentVolume và
PersistentVolumeClaim sẽ được ràng buộc với nhau.

Việc ràng buộc diễn ra bất kể một số tiêu chí khớp volume, bao gồm cả node affinity.
Control plane vẫn kiểm tra rằng [storage class](96-storage-classes-vi.md),
các chế độ truy cập, và kích thước lưu trữ được yêu cầu là hợp lệ.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: foo-pvc
  namespace: foo
spec:
  storageClassName: "" # Chuỗi rỗng phải được đặt tường minh, nếu không StorageClass mặc định sẽ được gán
  volumeName: foo-pv
  ...
```

Phương pháp này không đảm bảo bất kỳ đặc quyền ràng buộc nào đối với PersistentVolume.
Nếu các PersistentVolumeClaim khác có thể sử dụng PV mà bạn chỉ định, trước tiên bạn
cần dành riêng (reserve) volume lưu trữ đó. Chỉ định PersistentVolumeClaim liên quan
trong trường `claimRef` của PV để các PVC khác không thể ràng buộc với nó.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: foo-pv
spec:
  storageClassName: ""
  claimRef:
    name: foo-pvc
    namespace: foo
  ...
```

Điều này hữu ích nếu bạn muốn tiêu thụ các PersistentVolume có `persistentVolumeReclaimPolicy` được đặt
là `Retain`, bao gồm cả trường hợp bạn đang tái sử dụng một PV có sẵn.

### Mở rộng Persistent Volume Claim (Expanding Persistent Volumes Claims) {#expanding-persistent-volumes-claims}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Hỗ trợ mở rộng PersistentVolumeClaim (PVC) được bật theo mặc định. Bạn có thể mở rộng
các loại volume sau:

* csi (bao gồm một số loại volume đã được di trú sang CSI — CSI migrated)
* flexVolume (đã bị loại bỏ dần)
* portworxVolume (đã bị loại bỏ dần)

Bạn chỉ có thể mở rộng một PVC nếu trường `allowVolumeExpansion` của storage class của nó được đặt là true.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: example-vol-default
provisioner: vendor-name.example/magicstorage
parameters:
  resturl: "http://192.168.10.100:8080"
  restuser: ""
  secretNamespace: ""
  secretName: ""
allowVolumeExpansion: true
```

Để yêu cầu volume lớn hơn cho một PVC, hãy sửa đối tượng PVC và chỉ định kích thước
lớn hơn. Điều này kích hoạt việc mở rộng volume nằm sau PersistentVolume bên dưới. Một
PersistentVolume mới không bao giờ được tạo ra để đáp ứng claim. Thay vào đó, volume hiện có được thay đổi kích thước.

> **Cảnh báo:**
> Việc trực tiếp sửa kích thước của một PersistentVolume có thể ngăn cản việc tự động thay đổi kích thước của volume đó.
> Nếu bạn sửa dung lượng (capacity) của một PersistentVolume, rồi sau đó sửa `.spec` của
> PersistentVolumeClaim khớp với nó để làm cho kích thước của PersistentVolumeClaim bằng với PersistentVolume,
> thì sẽ không có việc thay đổi kích thước lưu trữ nào diễn ra.
> Control plane của Kubernetes sẽ thấy trạng thái mong muốn của cả hai tài nguyên khớp nhau,
> kết luận rằng kích thước volume nền đã được tăng thủ công
> và không cần thay đổi kích thước nữa.

#### Mở rộng volume CSI (CSI Volume expansion)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Hỗ trợ mở rộng volume CSI được bật theo mặc định nhưng cũng đòi hỏi
CSI driver cụ thể phải hỗ trợ mở rộng volume. Tham khảo tài liệu của
CSI driver cụ thể để biết thêm thông tin.

#### Thay đổi kích thước volume chứa file system (Resizing a volume containing a file system)

Bạn chỉ có thể thay đổi kích thước các volume chứa file system nếu file system đó là XFS, Ext3, hoặc Ext4.

Khi một volume chứa file system, file system chỉ được thay đổi kích thước khi một Pod mới đang sử dụng
PersistentVolumeClaim ở chế độ `ReadWrite`. Việc mở rộng file system được thực hiện hoặc khi Pod đang khởi động,
hoặc khi Pod đang chạy và file system bên dưới hỗ trợ mở rộng trực tuyến (online expansion).

FlexVolume (đã bị loại bỏ dần từ Kubernetes v1.23) cho phép thay đổi kích thước nếu driver được cấu hình với
khả năng `RequiresFSResize` là `true`. FlexVolume có thể được thay đổi kích thước khi Pod khởi động lại.

#### Thay đổi kích thước PersistentVolumeClaim đang được sử dụng (Resizing an in-use PersistentVolumeClaim)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Trong trường hợp này, bạn không cần xóa và tạo lại Pod hay deployment đang sử dụng PVC hiện có.
Bất kỳ PVC nào đang được sử dụng sẽ tự động trở nên khả dụng cho Pod của nó ngay khi file system của nó được mở rộng xong.
Tính năng này không có tác dụng với các PVC không được Pod hay deployment nào sử dụng. Bạn phải tạo một Pod
sử dụng PVC đó thì việc mở rộng mới có thể hoàn tất.

Tương tự các loại volume khác — volume FlexVolume cũng có thể được mở rộng khi đang được một Pod sử dụng.

> **Ghi chú:**
> Việc thay đổi kích thước FlexVolume chỉ khả thi khi driver bên dưới hỗ trợ thay đổi kích thước.

#### Khôi phục khi mở rộng volume thất bại (Recovering from Failure when Expanding Volumes)

Nếu người dùng chỉ định kích thước mới quá lớn để hệ thống lưu trữ bên dưới có thể đáp ứng,
việc mở rộng PVC sẽ được thử lại liên tục cho đến khi người dùng hoặc
quản trị viên cluster thực hiện một hành động nào đó. Điều này có thể không mong muốn, do đó
Kubernetes cung cấp các phương pháp sau để khôi phục khỏi những thất bại như vậy.

##### Thủ công với quyền truy cập của quản trị viên cluster (Manually with Cluster Administrator access)

Nếu việc mở rộng phần lưu trữ bên dưới thất bại, quản trị viên cluster có thể
khôi phục thủ công trạng thái của Persistent Volume Claim (PVC) và hủy các yêu cầu thay đổi kích thước.
Nếu không, các yêu cầu thay đổi kích thước sẽ được controller thử lại liên tục mà không cần
sự can thiệp của quản trị viên.

1. Đánh dấu PersistentVolume (PV) đang ràng buộc với PersistentVolumeClaim (PVC)
   bằng chính sách thu hồi `Retain`.
2. Xóa PVC. Vì PV có chính sách thu hồi `Retain` — chúng ta sẽ không mất dữ liệu nào
   khi tạo lại PVC.
3. Xóa mục `claimRef` khỏi spec của PV, để PVC mới có thể ràng buộc với nó.
   Việc này sẽ làm cho PV trở thành `Available`.
4. Tạo lại PVC với kích thước nhỏ hơn PV và đặt trường `volumeName` của
   PVC thành tên của PV. Việc này sẽ ràng buộc PVC mới với PV hiện có.
5. Đừng quên khôi phục lại chính sách thu hồi của PV.

##### Bằng cách yêu cầu mở rộng tới kích thước nhỏ hơn (By requesting expansion to smaller size)

Nếu việc mở rộng một PVC đã thất bại, bạn có thể thử lại việc mở rộng với
kích thước nhỏ hơn giá trị đã yêu cầu trước đó. Để yêu cầu một lần thử mở rộng mới với
kích thước đề xuất nhỏ hơn, hãy sửa `.spec.resources` của PVC đó và chọn một giá trị nhỏ hơn
giá trị bạn đã thử trước đó.
Điều này hữu ích nếu việc mở rộng tới giá trị cao hơn không thành công vì ràng buộc dung lượng.
Nếu điều đó đã xảy ra, hoặc bạn nghi ngờ nó có thể đã xảy ra, bạn có thể thử lại việc mở rộng bằng cách chỉ định
kích thước nằm trong giới hạn dung lượng của nhà cung cấp lưu trữ bên dưới. Bạn có thể theo dõi trạng thái của
thao tác thay đổi kích thước bằng cách quan sát `.status.allocatedResourceStatuses` và các sự kiện (event) trên PVC.

Lưu ý rằng,
mặc dù bạn có thể chỉ định lượng lưu trữ thấp hơn mức đã yêu cầu trước đó,
giá trị mới vẫn phải cao hơn `.status.capacity`.
Kubernetes không hỗ trợ thu nhỏ PVC xuống dưới kích thước hiện tại của nó.

## Các loại Persistent Volume (Types of Persistent Volumes)

Các loại PersistentVolume được triển khai dưới dạng plugin. Kubernetes hiện hỗ trợ các plugin sau:

* [`csi`](91-volumes-vi.md#csi) - Container Storage Interface (CSI)
* [`fc`](91-volumes-vi.md#fc) - lưu trữ Fibre Channel (FC)
* [`hostPath`](91-volumes-vi.md#hostpath) - volume HostPath
  (chỉ dành cho kiểm thử trên một node duy nhất; SẼ KHÔNG HOẠT ĐỘNG trong cluster nhiều node;
  hãy cân nhắc dùng volume `local` thay thế)
* [`iscsi`](91-volumes-vi.md#iscsi) - lưu trữ iSCSI (SCSI over IP)
* [`local`](91-volumes-vi.md#local) - các thiết bị lưu trữ cục bộ
  được mount trên node.
* [`nfs`](91-volumes-vi.md#nfs) - lưu trữ Network File System (NFS)

Các loại PersistentVolume sau đã bị loại bỏ dần nhưng vẫn khả dụng.
Nếu bạn đang sử dụng các loại volume này, ngoại trừ `flexVolume`, `cephfs` và `rbd`,
vui lòng cài đặt CSI driver tương ứng.

* [`awsElasticBlockStore`](https://kubernetes.io/docs/concepts/storage/volumes#awselasticblockstore) - AWS Elastic Block Store (EBS)
  (**di trú bật theo mặc định** từ v1.23)
* [`azureDisk`](https://kubernetes.io/docs/concepts/storage/volumes#azuredisk) - Azure Disk
  (**di trú bật theo mặc định** từ v1.23)
* [`azureFile`](https://kubernetes.io/docs/concepts/storage/volumes#azurefile) - Azure File
  (**di trú bật theo mặc định** từ v1.24)
* [`cinder`](https://kubernetes.io/docs/concepts/storage/volumes#cinder) - Cinder (lưu trữ khối OpenStack)
  (**di trú bật theo mặc định** từ v1.21)
* [`flexVolume`](91-volumes-vi.md#flexvolume) - FlexVolume
  (**bị loại bỏ dần** từ v1.23, không có kế hoạch di trú và không có kế hoạch gỡ bỏ hỗ trợ)
* [`gcePersistentDisk`](https://kubernetes.io/docs/concepts/storage/volumes#gcePersistentDisk) - GCE Persistent Disk
  (**di trú bật theo mặc định** từ v1.23)
* [`portworxVolume`](91-volumes-vi.md#portworxvolume) - volume Portworx
  (**di trú bật theo mặc định** từ v1.31)
* [`vsphereVolume`](https://kubernetes.io/docs/concepts/storage/volumes#vspherevolume) - volume vSphere VMDK
  (**di trú bật theo mặc định** từ v1.25)

Các phiên bản Kubernetes cũ hơn còn hỗ trợ các loại PersistentVolume in-tree sau:

* [`cephfs`](https://kubernetes.io/docs/concepts/storage/volumes#cephfs)
  (**không còn khả dụng** từ v1.31)
* `flocker` - lưu trữ Flocker.
  (**không còn khả dụng** từ v1.25)
* `glusterfs` - lưu trữ GlusterFS.
  (**không còn khả dụng** từ v1.26)
* `photonPersistentDisk` - đĩa bền vững của Photon controller.
  (**không còn khả dụng** từ v1.15)
* `quobyte` - volume Quobyte.
  (**không còn khả dụng** từ v1.25)
* [`rbd`](https://kubernetes.io/docs/concepts/storage/volumes#rbd) - volume Rados Block Device (RBD)
  (**không còn khả dụng** từ v1.31)
* `scaleIO` - volume ScaleIO.
  (**không còn khả dụng** từ v1.21)
* `storageos` - volume StorageOS.
  (**không còn khả dụng** từ v1.25)

## Persistent Volumes

Mỗi PV chứa một spec và status, tức là đặc tả và trạng thái của volume.
Tên của một đối tượng PersistentVolume phải là một
[tên miền con DNS hợp lệ](17-names-vi.md#dns-subdomain-names).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

> **Ghi chú:**
> Có thể cần các chương trình trợ giúp (helper program) liên quan đến loại volume để tiêu thụ
> một PersistentVolume trong cluster. Trong ví dụ này, PersistentVolume có
> loại NFS và cần chương trình trợ giúp /sbin/mount.nfs để hỗ trợ việc
> mount các filesystem NFS.

### Dung lượng (Capacity)

Nói chung, một PV sẽ có một dung lượng lưu trữ cụ thể. Dung lượng này được đặt bằng thuộc tính
`capacity` của PV, là một giá trị Quantity.

Hiện tại, kích thước lưu trữ là tài nguyên duy nhất có thể được đặt hoặc yêu cầu.
Các thuộc tính trong tương lai có thể bao gồm IOPS, thông lượng (throughput), v.v.

### Chế độ volume (Volume Mode) {#volume-mode}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [stable]`

Kubernetes hỗ trợ hai `volumeModes` cho PersistentVolume: `Filesystem` và `Block`.

`volumeMode` là một tham số API tùy chọn.
`Filesystem` là chế độ mặc định được dùng khi tham số `volumeMode` bị bỏ qua.

Một volume với `volumeMode: Filesystem` được *mount* vào Pod dưới dạng một thư mục. Nếu volume
được hỗ trợ bởi một thiết bị khối (block device) và thiết bị đó trống, Kubernetes tạo một filesystem
trên thiết bị trước khi mount nó lần đầu tiên.

Bạn có thể đặt giá trị của `volumeMode` là `Block` để dùng volume như một thiết bị khối thô (raw block device).
Volume như vậy được cung cấp cho Pod dưới dạng một thiết bị khối, không có bất kỳ filesystem nào trên đó.
Chế độ này hữu ích để cung cấp cho Pod cách nhanh nhất có thể để truy cập volume, không có
bất kỳ lớp filesystem nào giữa Pod và volume. Mặt khác, ứng dụng
chạy trong Pod phải biết cách xử lý một thiết bị khối thô.
Xem [Hỗ trợ raw block volume](#raw-block-volume-support)
để có ví dụ về cách sử dụng volume với `volumeMode: Block` trong một Pod.

### Các chế độ truy cập (Access Modes) {#access-modes}

Một PersistentVolume có thể được mount trên máy chủ theo bất kỳ cách nào mà nhà cung cấp
tài nguyên hỗ trợ. Như bảng dưới đây cho thấy, các nhà cung cấp sẽ có những khả năng khác nhau
và các chế độ truy cập của mỗi PV được đặt theo các chế độ cụ thể mà volume
đó hỗ trợ. Ví dụ, NFS có thể hỗ trợ nhiều client đọc/ghi, nhưng một PV
NFS cụ thể có thể được xuất (export) trên server ở dạng chỉ đọc. Mỗi PV có tập hợp
chế độ truy cập riêng mô tả các khả năng của PV cụ thể đó.

Các chế độ truy cập là:

`ReadWriteOnce`
: volume có thể được mount ở chế độ đọc-ghi bởi một node duy nhất. Chế độ truy cập ReadWriteOnce
  vẫn có thể cho phép nhiều pod truy cập (đọc hoặc ghi) volume đó khi các pod
  chạy trên cùng một node. Để truy cập bởi một pod duy nhất, hãy xem ReadWriteOncePod.

`ReadOnlyMany`
: volume có thể được mount ở chế độ chỉ đọc bởi nhiều node.

`ReadWriteMany`
: volume có thể được mount ở chế độ đọc-ghi bởi nhiều node.

 `ReadWriteOncePod`
: **TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.29 [stable]`
  volume có thể được mount ở chế độ đọc-ghi bởi một Pod duy nhất. Sử dụng chế độ truy cập
  ReadWriteOncePod nếu bạn muốn đảm bảo rằng chỉ một pod trong toàn bộ cluster có thể
  đọc hoặc ghi vào PVC đó.

> **Ghi chú:**
> Chế độ truy cập `ReadWriteOncePod` chỉ được hỗ trợ cho các volume
> CSI và Kubernetes phiên bản 1.22+. Để dùng tính năng này bạn cần cập nhật các
> [CSI sidecar](https://kubernetes-csi.github.io/docs/sidecar-containers.html)
> sau lên các phiên bản này hoặc mới hơn:
>
> * [csi-provisioner:v3.0.0+](https://github.com/kubernetes-csi/external-provisioner/releases/tag/v3.0.0)
> * [csi-attacher:v3.3.0+](https://github.com/kubernetes-csi/external-attacher/releases/tag/v3.3.0)
> * [csi-resizer:v1.3.0+](https://github.com/kubernetes-csi/external-resizer/releases/tag/v1.3.0)

Trong CLI, các chế độ truy cập được viết tắt là:

* RWO - ReadWriteOnce
* ROX - ReadOnlyMany
* RWX - ReadWriteMany
* RWOP - ReadWriteOncePod

> **Ghi chú:**
> Kubernetes sử dụng các chế độ truy cập của volume để khớp PersistentVolumeClaim với PersistentVolume.
> Trong một số trường hợp, các chế độ truy cập của volume cũng ràng buộc nơi PersistentVolume có thể được mount.
> Các chế độ truy cập của volume **không** áp đặt bảo vệ ghi (write protection) một khi phần lưu trữ đã được mount.
> Ngay cả khi các chế độ truy cập được chỉ định là ReadWriteOnce, ReadOnlyMany, hoặc ReadWriteMany,
> chúng không đặt bất kỳ ràng buộc nào lên volume. Ví dụ, ngay cả khi một PersistentVolume được
> tạo với ReadOnlyMany, không có gì đảm bảo rằng nó sẽ là chỉ đọc. Nếu các chế độ truy cập
> được chỉ định là ReadWriteOncePod, volume bị ràng buộc và chỉ có thể được mount trên một Pod duy nhất.

> __Quan trọng!__ Một volume chỉ có thể được mount bằng một chế độ truy cập tại một thời điểm,
> ngay cả khi nó hỗ trợ nhiều chế độ.

| Volume Plugin        | ReadWriteOnce          | ReadOnlyMany          | ReadWriteMany | ReadWriteOncePod       |
| :---                 | :---:                  | :---:                 | :---:         | -                      |
| AzureFile            | &#x2713;               | &#x2713;              | &#x2713;      | -                      |
| CephFS               | &#x2713;               | &#x2713;              | &#x2713;      | -                      |
| CSI                  | tùy thuộc driver       | tùy thuộc driver      | tùy thuộc driver | tùy thuộc driver    |
| FC                   | &#x2713;               | &#x2713;              | -             | -                      |
| FlexVolume           | &#x2713;               | &#x2713;              | tùy thuộc driver | -                   |
| HostPath             | &#x2713;               | -                     | -             | -                      |
| iSCSI                | &#x2713;               | &#x2713;              | -             | -                      |
| NFS                  | &#x2713;               | &#x2713;              | &#x2713;      | -                      |
| RBD                  | &#x2713;               | &#x2713;              | -             | -                      |
| VsphereVolume        | &#x2713;               | -                     | - (hoạt động khi các Pod nằm cùng chỗ) | - |
| PortworxVolume       | &#x2713;               | -                     | &#x2713;      | -                  | - |

### Lớp (Class)

Một PV có thể có một class, được chỉ định bằng cách đặt thuộc tính
`storageClassName` thành tên của một
[StorageClass](96-storage-classes-vi.md).
Một PV thuộc một class cụ thể chỉ có thể được ràng buộc với các PVC yêu cầu
class đó. Một PV không có `storageClassName` thì không có class và chỉ có thể được ràng buộc
với các PVC không yêu cầu class cụ thể nào.

Trước đây, annotation `volume.beta.kubernetes.io/storage-class` được dùng thay
cho thuộc tính `storageClassName`. Annotation này vẫn hoạt động; tuy nhiên,
nó sẽ bị loại bỏ hoàn toàn trong một bản phát hành Kubernetes tương lai.

### Chính sách thu hồi (Reclaim Policy) {#reclaim-policy}

Các chính sách thu hồi hiện tại là:

* Retain -- thu hồi thủ công
* Recycle -- xóa cơ bản (`rm -rf /thevolume/*`)
* Delete -- xóa volume

Với Kubernetes v1.36, chỉ các loại volume `nfs` và `hostPath` hỗ trợ tái chế (recycling).

### Tùy chọn mount (Mount Options)

Quản trị viên Kubernetes có thể chỉ định các tùy chọn mount bổ sung cho việc
mount một Persistent Volume trên node.

> **Ghi chú:**
> Không phải tất cả các loại Persistent Volume đều hỗ trợ tùy chọn mount.

Các loại volume sau hỗ trợ tùy chọn mount:

* `csi` (bao gồm các loại volume đã di trú sang CSI)
* `iscsi`
* `nfs`

Các tùy chọn mount không được kiểm tra tính hợp lệ (validate). Nếu một tùy chọn mount không hợp lệ, việc mount sẽ thất bại.

Trước đây, annotation `volume.beta.kubernetes.io/mount-options` được dùng thay
cho thuộc tính `mountOptions`. Annotation này vẫn hoạt động; tuy nhiên,
nó sẽ bị loại bỏ hoàn toàn trong một bản phát hành Kubernetes tương lai.

### Node Affinity

> **Ghi chú:**
> Với hầu hết các loại volume, bạn không cần đặt trường này.
> Bạn cần đặt nó một cách tường minh cho volume [local](91-volumes-vi.md#local).

Một PV có thể chỉ định node affinity để định nghĩa các ràng buộc giới hạn những node nào
có thể truy cập volume này. Các Pod sử dụng PV sẽ chỉ được lên lịch tới các node
được chọn bởi node affinity đó. Để chỉ định node affinity, đặt
`nodeAffinity` trong `.spec` của PV. Tài liệu tham khảo API
[PersistentVolume](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-v1/#PersistentVolumeSpec)
có thêm chi tiết về trường này.

#### Cập nhật node affinity (Updates to node affinity)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [alpha]`

Nếu [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `MutablePVNodeAffinity` được bật trong cluster của bạn,
trường `.spec.nodeAffinity` của một PersistentVolume là có thể thay đổi (mutable).
Điều này cho phép quản trị viên cluster hoặc storage controller bên ngoài cập nhật node affinity của một PersistentVolume khi dữ liệu được di chuyển,
mà không làm gián đoạn các pod đang chạy.

Khi cập nhật node affinity, bạn nên đảm bảo node affinity mới vẫn khớp với các node nơi volume hiện đang được sử dụng.
Với các pod vi phạm affinity mới, nếu pod đang chạy, nó có thể tiếp tục chạy. Nhưng Kubernetes không hỗ trợ cấu hình này.
Bạn nên sớm chấm dứt (terminate) các pod vi phạm.
Do việc lưu đệm trong bộ nhớ (in-memory caching), các pod được tạo sau khi cập nhật có thể vẫn được lên lịch theo node affinity cũ trong một khoảng thời gian ngắn.

Để dùng tính năng này, bạn nên bật feature gate `MutablePVNodeAffinity` trên các thành phần sau:

- `kube-apiserver`
- `kubelet`

### Pha (Phase)

Một PersistentVolume sẽ ở một trong các pha sau:

`Available`
: một tài nguyên còn trống, chưa được ràng buộc với claim nào

`Bound`
: volume đã được ràng buộc với một claim

`Released`
: claim đã bị xóa, nhưng tài nguyên lưu trữ liên kết chưa được cluster thu hồi

`Failed`
: volume đã thất bại trong quá trình thu hồi (tự động) của nó

Bạn có thể xem tên của PVC ràng buộc với PV bằng lệnh `kubectl describe persistentvolume <name>`.

#### Dấu thời gian chuyển pha (Phase transition timestamp)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.31 [stable]`

Trường `.status` của một PersistentVolume có thể bao gồm trường alpha `lastPhaseTransitionTime`. Trường này ghi lại
dấu thời gian của lần gần nhất volume chuyển pha. Với các volume mới được tạo,
pha được đặt là `Pending` và `lastPhaseTransitionTime` được đặt là
thời điểm hiện tại.

## PersistentVolumeClaims

Mỗi PVC chứa một spec và status, tức là đặc tả và trạng thái của claim.
Tên của một đối tượng PersistentVolumeClaim phải là một
[tên miền con DNS hợp lệ](17-names-vi.md#dns-subdomain-names).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```

### Các chế độ truy cập (Access Modes)

Claim sử dụng [cùng quy ước như volume](#access-modes) khi yêu cầu
lưu trữ với các chế độ truy cập cụ thể.

### Các chế độ volume (Volume Modes)

Claim sử dụng [cùng quy ước như volume](#volume-mode) để chỉ định
việc tiêu thụ volume dưới dạng filesystem hay thiết bị khối.

### Tên volume (Volume Name)

Claim có thể dùng trường `volumeName` để ràng buộc tường minh với một PersistentVolume cụ thể. Bạn cũng có thể
không đặt `volumeName`, biểu thị rằng bạn muốn Kubernetes thiết lập một PersistentVolume mới
khớp với claim đó.
Nếu PV được chỉ định đã ràng buộc với một PVC khác, việc ràng buộc sẽ bị kẹt
ở trạng thái pending.

### Tài nguyên (Resources)

Claim, giống như Pod, có thể yêu cầu một lượng cụ thể của một tài nguyên. Trong trường hợp này,
yêu cầu là cho lưu trữ. Cùng một
[mô hình tài nguyên](https://git.k8s.io/design-proposals-archive/scheduling/resources.md)
áp dụng cho cả volume và claim.

> **Ghi chú:**
> Với các volume `Filesystem`, yêu cầu lưu trữ chỉ kích thước volume "bên ngoài"
> (tức là kích thước được cấp phát từ backend lưu trữ).
> Điều này có nghĩa kích thước có thể ghi được có thể thấp hơn một chút với các nhà cung cấp
> xây dựng filesystem trên nền một thiết bị khối, do phần chi phí (overhead) của filesystem.
> Điều này đặc biệt rõ với XFS, nơi nhiều tính năng metadata được bật theo mặc định.

### Bộ chọn (Selector)

Claim có thể chỉ định một
[label selector](18-labels-vi.md#label-selectors)
để lọc thêm tập các volume.
Chỉ những volume có nhãn (label) khớp với selector mới có thể được ràng buộc với claim.
Selector có thể gồm hai trường:

* `matchLabels` - volume phải có một nhãn với giá trị này
* `matchExpressions` - danh sách các yêu cầu được tạo bằng cách chỉ định khóa (key), danh sách giá trị,
  và toán tử liên hệ giữa khóa và các giá trị.
  Các toán tử hợp lệ gồm `In`, `NotIn`, `Exists`, và `DoesNotExist`.

Tất cả các yêu cầu, từ cả `matchLabels` và `matchExpressions`, được
kết hợp với nhau bằng AND — tất cả đều phải được thỏa mãn thì mới khớp.

### Lớp (Class)

Một claim có thể yêu cầu một class cụ thể bằng cách chỉ định tên của một
[StorageClass](96-storage-classes-vi.md)
qua thuộc tính `storageClassName`.
Chỉ những PV thuộc class được yêu cầu — có cùng `storageClassName` với PVC —
mới có thể được ràng buộc với PVC đó.

PVC không nhất thiết phải yêu cầu một class. Một PVC với `storageClassName` được đặt
bằng `""` luôn được hiểu là yêu cầu một PV không có class, do đó nó
chỉ có thể được ràng buộc với các PV không có class (không có annotation hoặc annotation được đặt bằng `""`).
Một PVC không có `storageClassName` thì không hoàn toàn giống như vậy và được cluster
xử lý khác đi, tùy theo việc
[admission plugin `DefaultStorageClass`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#defaultstorageclass)
có được bật hay không.

* Nếu admission plugin được bật, quản trị viên có thể chỉ định một StorageClass mặc định.
  Tất cả PVC không có `storageClassName` chỉ có thể được ràng buộc với các PV thuộc class mặc định đó.
  Việc chỉ định StorageClass mặc định được thực hiện bằng cách đặt annotation
  `storageclass.kubernetes.io/is-default-class` bằng `true` trong một đối tượng StorageClass.
  Nếu quản trị viên không chỉ định class mặc định, cluster phản hồi việc tạo PVC
  như thể admission plugin bị tắt.
  Nếu có nhiều hơn một StorageClass mặc định được chỉ định, class mặc định mới nhất được dùng khi
  PVC được cấp phát động.
* Nếu admission plugin bị tắt, không có khái niệm StorageClass mặc định.
  Tất cả PVC có `storageClassName` được đặt bằng `""` chỉ có thể được ràng buộc với các PV
  cũng có `storageClassName` được đặt bằng `""`.
  Tuy nhiên, các PVC thiếu `storageClassName` có thể được cập nhật sau này khi StorageClass mặc định trở nên khả dụng.
  Nếu PVC được cập nhật, nó sẽ không còn ràng buộc với các PV có `storageClassName` cũng được đặt bằng `""`.

Xem [gán StorageClass mặc định hồi tố](#retroactive-default-storageclass-assignment) để biết thêm chi tiết.

Tùy phương thức cài đặt, một StorageClass mặc định có thể được triển khai
vào cluster Kubernetes bởi addon manager trong quá trình cài đặt.

Khi một PVC chỉ định `selector` bên cạnh việc yêu cầu một StorageClass,
các yêu cầu được kết hợp bằng AND: chỉ PV thuộc class được yêu cầu và có
các nhãn được yêu cầu mới có thể được ràng buộc với PVC.

> **Ghi chú:**
> Hiện tại, một PVC có `selector` không rỗng thì không thể được cấp phát động một PV cho nó.

Trước đây, annotation `volume.beta.kubernetes.io/storage-class` được dùng thay
cho thuộc tính `storageClassName`. Annotation này vẫn hoạt động; tuy nhiên,
nó sẽ không được hỗ trợ trong một bản phát hành Kubernetes tương lai.

#### Gán StorageClass mặc định hồi tố (Retroactive default StorageClass assignment) {#retroactive-default-storageclass-assignment}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.28 [stable]`

Bạn có thể tạo một PersistentVolumeClaim mà không chỉ định `storageClassName`
cho PVC mới, và bạn có thể làm vậy ngay cả khi không tồn tại StorageClass mặc định nào
trong cluster của bạn. Trong trường hợp này, PVC mới được tạo đúng như bạn định nghĩa, và
`storageClassName` của PVC đó vẫn không được đặt cho đến khi class mặc định trở nên khả dụng.

Khi một StorageClass mặc định trở nên khả dụng, control plane xác định mọi
PVC hiện có không có `storageClassName`. Với các PVC có giá trị `storageClassName` rỗng
hoặc không có khóa này, control plane sau đó
cập nhật các PVC đó để đặt `storageClassName` khớp với StorageClass mặc định mới.
Nếu bạn có một PVC hiện hữu với `storageClassName` là `""`, và bạn cấu hình
một StorageClass mặc định, thì PVC này sẽ không được cập nhật.

Để tiếp tục ràng buộc với các PV có `storageClassName` được đặt bằng `""`
(trong khi đang tồn tại một StorageClass mặc định), bạn cần đặt `storageClassName`
của PVC liên quan thành `""`.

Hành vi này giúp quản trị viên thay đổi StorageClass mặc định bằng cách gỡ bỏ
class cũ trước rồi tạo hoặc đặt một class khác. Khoảng thời gian ngắn khi
không có class mặc định khiến các PVC không có `storageClassName` được tạo lúc đó
không có class mặc định nào, nhưng nhờ việc gán StorageClass mặc định
hồi tố, cách thay đổi mặc định này là an toàn.

### Theo dõi PVC không được sử dụng (Unused PVC tracking)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Khi được bật, controller bảo vệ PVC sẽ thêm một
[condition](47-pod-lifecycle-vi.md#pod-conditions) `Unused` vào mỗi
PersistentVolumeClaim để chỉ ra liệu nó hiện có đang được tham chiếu bởi bất kỳ
Pod chưa kết thúc (non-terminal) nào hay không.

Condition này có hai trạng thái:

`Unused` với status `"True"` (lý do `NoPodsUsingPVC`)
: Không có Pod chưa kết thúc nào tham chiếu PVC này. `lastTransitionTime` ghi lại thời điểm
  PVC trở nên không được sử dụng.

`Unused` với status `"False"` (lý do `PodUsingPVC`)
: Ít nhất một Pod chưa kết thúc hiện đang tham chiếu PVC này.
  `lastTransitionTime` ghi lại thời điểm PVC bắt đầu được sử dụng.

Một Pod được coi là chưa kết thúc nếu pha của nó không phải `Succeeded` hoặc `Failed`.
Điều này có nghĩa một Pod ở trạng thái Pending (kể cả Pod chưa được lên lịch) cũng được tính
là đang sử dụng PVC.

`lastTransitionTime` của condition `Unused` có thể được quản trị viên cluster,
các công cụ giám sát, và các controller bên ngoài sử dụng để xác định các PVC
đã không được sử dụng trong thời gian dài. Ví dụ, để tìm tất cả PVC đã
không được sử dụng hơn 30 ngày, bạn có thể truy vấn các PVC mà condition `Unused`
có `status: "True"` và `lastTransitionTime` cũ hơn 30 ngày.

> **Ghi chú:**
> Khoảng thời gian không sử dụng mà condition này chỉ ra có thể ngắn hơn thời gian
> không sử dụng thực tế do độ trễ xử lý trong controller hoặc do
> tính năng được bật sau khi PVC đã ở trạng thái không sử dụng. Condition không được
> cập nhật khi PVC có `deletionTimestamp` được đặt (tức là các PVC đang bị xóa).

## Dùng claim làm volume (Claims As Volumes) {#claims-as-volumes}

Pod truy cập lưu trữ bằng cách dùng claim như một volume. Claim phải tồn tại trong
cùng namespace với Pod sử dụng claim đó. Cluster tìm claim trong
namespace của Pod và dùng nó để lấy PersistentVolume nằm sau claim.
Volume sau đó được mount vào máy chủ và vào trong Pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

### Lưu ý về namespace (A Note on Namespaces)

Các ràng buộc của PersistentVolume là độc quyền, và vì PersistentVolumeClaim là
các đối tượng thuộc namespace, việc mount claim với các chế độ "Many" (`ROX`, `RWX`) chỉ
khả thi bên trong một namespace.

### PersistentVolume kiểu `hostPath` (PersistentVolumes typed `hostPath`)

Một PersistentVolume kiểu `hostPath` dùng một tệp hoặc thư mục trên Node để mô phỏng
lưu trữ gắn qua mạng (network-attached storage). Xem
[ví dụ về volume kiểu `hostPath`](https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/#create-a-persistentvolume).

## Hỗ trợ raw block volume (Raw Block Volume Support) {#raw-block-volume-support}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [stable]`

Các volume plugin sau hỗ trợ raw block volume, bao gồm cả cấp phát động khi
áp dụng được:

* CSI (bao gồm một số loại volume đã di trú sang CSI)
* FC (Fibre Channel)
* iSCSI
* Local volume

### PersistentVolume sử dụng Raw Block Volume (PersistentVolume using a Raw Block Volume) {#persistent-volume-using-a-raw-block-volume}

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: block-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  persistentVolumeReclaimPolicy: Retain
  fc:
    targetWWNs: ["50060e801049cfd1"]
    lun: 0
    readOnly: false
```

### PersistentVolumeClaim yêu cầu Raw Block Volume (PersistentVolumeClaim requesting a Raw Block Volume) {#persistent-volume-claim-requesting-a-raw-block-volume}

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi
```

### Đặc tả Pod thêm đường dẫn Raw Block Device vào container (Pod specification adding Raw Block Device path in container)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-block-volume
spec:
  containers:
    - name: fc-container
      image: fedora:26
      command: ["/bin/sh", "-c"]
      args: [ "tail -f /dev/null" ]
      volumeDevices:
        - name: data
          devicePath: /dev/xvda
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: block-pvc
```

> **Ghi chú:**
> Khi thêm một thiết bị khối thô (raw block device) cho một Pod, bạn chỉ định đường dẫn thiết bị trong
> container thay vì đường dẫn mount.

### Ràng buộc Block Volume (Binding Block Volumes)

Nếu người dùng yêu cầu một raw block volume bằng cách chỉ định điều này qua trường `volumeMode`
trong spec của PersistentVolumeClaim, các quy tắc ràng buộc hơi khác so với
các bản phát hành trước vốn không coi chế độ này là một phần của spec.
Dưới đây là bảng các tổ hợp có thể mà người dùng và quản trị viên có thể chỉ định để
yêu cầu một thiết bị khối thô. Bảng cho biết volume sẽ được ràng buộc hay
không với từng tổ hợp: Ma trận ràng buộc volume cho các volume được cấp phát tĩnh:

| volumeMode của PV | volumeMode của PVC | Kết quả          |
| ------------------|:------------------:| ----------------:|
|   không chỉ định  | không chỉ định     | RÀNG BUỘC        |
|   không chỉ định  | Block              | KHÔNG RÀNG BUỘC  |
|   không chỉ định  | Filesystem         | RÀNG BUỘC        |
|   Block           | không chỉ định     | KHÔNG RÀNG BUỘC  |
|   Block           | Block              | RÀNG BUỘC        |
|   Block           | Filesystem         | KHÔNG RÀNG BUỘC  |
|   Filesystem      | Filesystem         | RÀNG BUỘC        |
|   Filesystem      | Block              | KHÔNG RÀNG BUỘC  |
|   Filesystem      | không chỉ định     | RÀNG BUỘC        |

> **Ghi chú:**
> Chỉ các volume được cấp phát tĩnh được hỗ trợ trong bản phát hành alpha. Quản trị viên
> nên cẩn trọng cân nhắc các giá trị này khi làm việc với các thiết bị khối thô.

## Hỗ trợ Volume Snapshot và khôi phục volume từ Snapshot (Volume Snapshot and Restore Volume from Snapshot Support) {#volume-snapshot-and-restore-volume-from-snapshot-support}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.20 [stable]`

Volume snapshot chỉ hỗ trợ các volume plugin CSI out-of-tree.
Để biết chi tiết, xem [Volume Snapshots](99-volume-snapshots-vi.md).
Các volume plugin in-tree đã bị loại bỏ dần. Bạn có thể đọc về các volume plugin
đã bị loại bỏ dần trong
[Volume Plugin FAQ](https://github.com/kubernetes/community/blob/main/sig-storage/volume-plugin-faq.md).

### Tạo PersistentVolumeClaim từ một Volume Snapshot (Create a PersistentVolumeClaim from a Volume Snapshot) {#create-persistent-volume-claim-from-volume-snapshot}

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restore-pvc
spec:
  storageClassName: csi-hostpath-sc
  dataSource:
    name: new-snapshot-test
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Nhân bản volume (Volume Cloning) {#volume-cloning}

[Nhân bản volume (Volume Cloning)](101-volume-pvc-datasource-vi.md)
chỉ khả dụng cho các volume plugin CSI.

### Tạo PersistentVolumeClaim từ một PVC có sẵn (Create PersistentVolumeClaim from an existing PVC) {#create-persistent-volume-claim-from-an-existing-pvc}

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cloned-pvc
spec:
  storageClassName: my-csi-plugin
  dataSource:
    name: existing-src-pvc-name
    kind: PersistentVolumeClaim
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## Volume populator và nguồn dữ liệu (Volume populators and data sources)

[Nhân bản volume](#volume-cloning) và
[khôi phục từ snapshot](#volume-snapshot-and-restore-volume-from-snapshot-support) điền sẵn dữ liệu
cho một volume mới từ một _nguồn dữ liệu_ (data source) tích hợp sẵn. _Volume populator_ mở rộng cơ chế này để
một PersistentVolumeClaim có thể được điền sẵn dữ liệu từ các loại nguồn khác (một custom resource),
được tham chiếu qua trường `dataSourceRef` của nó:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: populated-pvc
spec:
  dataSourceRef:
    name: example-name
    kind: ExampleDataSource
    apiGroup: example.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

Để biết chi tiết, bao gồm cả nguồn dữ liệu liên namespace (cross-namespace), xem
[Volume Populators and Data Sources](102-volume-populators-vi.md).

## Viết cấu hình khả chuyển (Writing Portable Configuration)

Nếu bạn viết các mẫu (template) cấu hình hoặc ví dụ chạy trên nhiều loại cluster khác nhau
và cần lưu trữ bền vững, khuyến nghị bạn dùng mẫu hình sau:

- Đưa các đối tượng PersistentVolumeClaim vào gói cấu hình của bạn (cùng với
  Deployment, ConfigMap, v.v.).
- Không đưa các đối tượng PersistentVolume vào cấu hình, vì người dùng khởi tạo
  cấu hình có thể không có quyền tạo PersistentVolume.
- Cho người dùng tùy chọn cung cấp tên storage class khi khởi tạo
  mẫu.
  - Nếu người dùng cung cấp tên storage class, hãy đặt giá trị đó vào
    trường `persistentVolumeClaim.storageClassName`.
    Điều này sẽ khiến PVC khớp với đúng storage
    class nếu cluster đã được quản trị viên bật StorageClass.
  - Nếu người dùng không cung cấp tên storage class, hãy để trường
    `persistentVolumeClaim.storageClassName` là nil. Điều này sẽ khiến một
    PV được tự động cấp phát cho người dùng với StorageClass mặc định
    trong cluster. Nhiều môi trường cluster có sẵn StorageClass mặc định,
    hoặc quản trị viên có thể tự tạo StorageClass mặc định của riêng mình.
- Trong công cụ của bạn, hãy theo dõi các PVC không được ràng buộc sau một khoảng thời gian
  và hiển thị điều này cho người dùng, vì nó có thể cho thấy cluster không có
  hỗ trợ lưu trữ động (khi đó người dùng nên tạo một PV khớp)
  hoặc cluster không có hệ thống lưu trữ (khi đó người dùng không thể triển khai
  cấu hình yêu cầu PVC).

## Tiếp theo (What's next)

* Tìm hiểu thêm về [Tạo một PersistentVolume](https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/#create-a-persistentvolume).
* Tìm hiểu thêm về [Tạo một PersistentVolumeClaim](https://kubernetes.io/docs/tutorials/configuration/configure-persistent-volume-storage/#create-a-persistentvolumeclaim).
* Đọc [tài liệu thiết kế Persistent Storage](https://git.k8s.io/design-proposals-archive/storage/persistent-storage.md).

### Tham khảo API (API references) {#reference}

Đọc về các API được mô tả trong trang này:

* [`PersistentVolume`](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-v1/)
* [`PersistentVolumeClaim`](https://kubernetes.io/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-claim-v1/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Cluster chỉ có các PV 50Gi đã cấp phát tĩnh. Bạn tạo một PVC xin 100Gi. Chuyện gì xảy ra,
   và trạng thái đó kéo dài bao lâu?
2. Một PV có `persistentVolumeReclaimPolicy: Retain`. Bạn xóa PVC đang bind với nó. PV chuyển
   sang pha nào, PVC khác dùng lại được ngay không? Câu trả lời khác thế nào nếu policy là
   `Delete`? Và PV được cấp phát động lấy policy từ đâu?
3. Hai Pod cùng mount một PVC có access mode `ReadWriteOnce`. Chúng chạy được không? Nếu bạn
   muốn chắc chắn chỉ đúng một Pod trong cả cluster ghi vào volume thì dùng gì?
4. PVC của bạn đang là 5Gi và bạn cần 10Gi. Sửa ở đâu, cần điều kiện gì, có PV mới được tạo
   không? Sau đó đổi ý muốn về 5Gi thì sao?
5. Cluster lab của bạn **chưa có StorageClass và chưa có provisioner**. Bạn `kubectl apply` một
   PVC không đặt `storageClassName`. Nó ở trạng thái nào? Sau khi Lab 6a cài xong một
   StorageClass mặc định thì điều gì tự động xảy ra với PVC đó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. PVC **ở trạng thái chưa ràng buộc (unbound) vô thời hạn**. Bài lấy đúng ví dụ này: một
   cluster có nhiều PV 50Gi sẽ không khớp với PVC xin 100Gi, và claim chỉ được bind khi một PV
   100Gi được thêm vào cluster. Vòng lặp điều khiển bảo đảm người dùng **luôn nhận được ít
   nhất** những gì họ yêu cầu, nên nó không ghép một PV nhỏ hơn cho đủ.
2. Với `Retain`: PV **vẫn tồn tại** và chuyển sang pha **`Released`** — nghĩa là claim đã bị
   xóa nhưng tài nguyên chưa được thu hồi. Nó **chưa sẵn sàng cho claim khác** vì dữ liệu của
   người dùng trước vẫn còn; admin phải tự xóa PV, tự dọn dữ liệu, tự xóa tài sản lưu trữ, rồi
   tạo lại PV mới nếu muốn tái sử dụng. Với `Delete`: **cả đối tượng PV lẫn tài sản lưu trữ
   bên ngoài đều bị xóa** — dữ liệu đi luôn. PV cấp phát động **kế thừa reclaim policy của
   StorageClass**, và mặc định của StorageClass là `Delete`; muốn khác thì phải cấu hình
   StorageClass từ đầu, nếu không phải sửa hoặc patch từng PV sau khi tạo.
3. **Chạy được, nhưng chỉ khi hai Pod nằm trên cùng một node.** Đây là chỗ dễ nhầm nhất:
   `ReadWriteOnce` nghĩa là volume mount đọc-ghi bởi **một node duy nhất**, không phải một Pod
   duy nhất; nhiều Pod trên cùng node vẫn cùng đọc/ghi được. Muốn đúng một Pod trong toàn
   cluster thì dùng **`ReadWriteOncePod`**. Cũng nhớ rằng access mode chỉ dùng để khớp PVC với
   PV và ràng buộc nơi mount, nó **không** áp đặt bảo vệ ghi: một PV khai `ReadOnlyMany` không
   vì thế mà trở thành chỉ đọc.
4. **Sửa `.spec.resources` của PVC** và xin kích thước lớn hơn. Điều kiện: storage class của
   PVC phải có **`allowVolumeExpansion: true`**. **Không có PV mới nào được tạo** — volume hiện
   có được thay đổi kích thước tại chỗ. Đừng sửa thẳng `capacity` của PV: control plane sẽ thấy
   trạng thái mong muốn của PV và PVC đã khớp, kết luận là không cần resize, và việc tự động mở
   rộng bị chặn lại. Còn về 5Gi: **không được**. Bạn có thể hạ giá trị yêu cầu xuống thấp hơn
   lần thử trước (hữu ích khi lần mở rộng trước thất bại vì hết dung lượng), nhưng giá trị mới
   vẫn phải **cao hơn `.status.capacity`**; Kubernetes không hỗ trợ thu nhỏ PVC.
5. PVC **được tạo đúng như bạn định nghĩa** và `storageClassName` **vẫn để trống**, còn bản
   thân PVC nằm chờ vì không có PV nào khớp. Đây là điểm khác biệt then chốt với `""`: bỏ trống
   không có nghĩa là "không class". Khi một StorageClass mặc định trở nên khả dụng, control
   plane rà các PVC hiện có không có `storageClassName` và **cập nhật chúng để trỏ tới class
   mặc định mới** — đó là *gán StorageClass mặc định hồi tố*. Nếu bạn đã đặt `storageClassName:
   ""` thì PVC sẽ **không** được cập nhật và tiếp tục chỉ bind được với PV không class.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
