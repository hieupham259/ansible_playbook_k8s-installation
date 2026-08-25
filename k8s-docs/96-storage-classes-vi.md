# Lớp lưu trữ (Storage Classes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/storage-classes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 4/16 · Kiểm chứng ở
[Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md).

Hơn một nửa bài là các mục theo từng nhà cung cấp lưu trữ (AWS, Azure, vSphere, Ceph,
Portworx). Đó là tài liệu tra cứu, không phải nội dung học. Bốn trường quyết định mọi thứ ở
lần đọc này là `provisioner`, `reclaimPolicy`, `volumeBindingMode` và annotation đánh dấu
class mặc định — tất cả đều xuất hiện trong ví dụ YAML đầu bài.

**Phải hiểu ở lần đọc này:**

- StorageClass là bản mô tả một "loại" lưu trữ do admin cung cấp, và `provisioner` là trường
  **bắt buộc**: nó xác định volume plugin nào sẽ cấp phát PV. Provisioner có thể là loại nội bộ
  hoặc một chương trình bên ngoài — mục *Đối tượng StorageClass*, *Provisioner*.
- `reclaimPolicy` của class quyết định reclaim policy của **mọi PV được class đó cấp phát
  động**; không khai thì mặc định là `Delete`. PV tạo thủ công giữ policy được gán lúc tạo —
  mục *Chính sách thu hồi*.
- `volumeBindingMode` quyết định **thời điểm** bind và cấp phát: `Immediate` (mặc định) làm
  ngay khi PVC được tạo, có thể sinh ra Pod không lập lịch được; `WaitForFirstConsumer` hoãn
  tới khi có Pod dùng PVC, để scheduler cân nhắc mọi ràng buộc của Pod — mục *Chế độ gắn kết
  volume*.
- StorageClass mặc định: đánh dấu bằng annotation `storageclass.kubernetes.io/is-default-class`;
  PVC không đặt `storageClassName` sẽ dùng nó; nhiều class mặc định thì Kubernetes lấy **cái
  được tạo gần đây nhất**; và cluster hoàn toàn có thể **không có** class mặc định nào — mục
  *StorageClass mặc định*.
- `allowVolumeExpansion: true` trên class là điều kiện để PVC mở rộng được, và chỉ **tăng**
  chứ không thu nhỏ — mục *Mở rộng volume*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bảng provisioner nội bộ và các mục *AWS EBS*, *AWS EFS*, *vSphere*, *Ceph RBD*, *Azure Disk*, *Azure File*, *Portworx volume* | lab không chạy trên các nền tảng đó, phần lớn đã lỗi thời | không cần |
| *NFS* và *Local* | là hai lựa chọn provisioner khả dĩ cho lab, nhưng chọn cái nào là việc của lab | Lab 6a |
| *Tùy chọn mount* | phụ thuộc từng volume plugin, không kiểm tra hợp lệ được trước | Lab 6a |
| *Các topology được phép* | cần cluster nhiều zone | không cần |
| *Tham số* | đặc thù từng `provisioner`, phải tra tài liệu của driver | Lab 6a |

---

Tài liệu này mô tả khái niệm StorageClass trong Kubernetes. Bạn nên
làm quen trước với [volume](91-volumes-vi.md) và
[persistent volume](92-persistent-volumes-vi.md).

Một StorageClass cung cấp cho quản trị viên một cách để mô tả các _lớp_ (class)
lưu trữ mà họ cung cấp. Các lớp khác nhau có thể tương ứng với các mức chất lượng dịch vụ (quality-of-service),
các chính sách sao lưu, hoặc các chính sách tùy ý do quản trị viên
cluster quyết định. Bản thân Kubernetes không có quan điểm áp đặt nào về việc các lớp
đại diện cho điều gì.

Khái niệm storage class của Kubernetes tương tự như "profiles" trong một số
thiết kế hệ thống lưu trữ khác.

## Đối tượng StorageClass (StorageClass objects)

Mỗi StorageClass chứa các trường `provisioner`, `parameters`, và
`reclaimPolicy`, được dùng khi một PersistentVolume thuộc lớp đó
cần được cấp phát động (dynamically provisioned) để đáp ứng một PersistentVolumeClaim (PVC).

Tên của một đối tượng StorageClass có ý nghĩa quan trọng, và là cách người dùng
yêu cầu một lớp cụ thể. Quản trị viên đặt tên và các tham số khác
của một lớp khi tạo các đối tượng StorageClass lần đầu tiên.

Với vai trò quản trị viên, bạn có thể chỉ định một StorageClass mặc định áp dụng cho mọi PVC
không yêu cầu một lớp cụ thể. Để biết thêm chi tiết, xem
[khái niệm PersistentVolumeClaim](92-persistent-volumes-vi.md#persistentvolumeclaims).

Đây là một ví dụ về StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: low-latency
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: csi-driver.example-vendor.example
reclaimPolicy: Retain # giá trị mặc định là Delete
allowVolumeExpansion: true
mountOptions:
  - discard # điều này có thể bật UNMAP / TRIM ở tầng lưu trữ khối (block storage)
volumeBindingMode: WaitForFirstConsumer
parameters:
  guaranteedReadWriteLatency: "true" # tùy nhà cung cấp (provider-specific)
```

## StorageClass mặc định (Default StorageClass)

Bạn có thể đánh dấu một StorageClass là mặc định cho cluster của mình.
Để biết hướng dẫn thiết lập StorageClass mặc định, xem
[Thay đổi StorageClass mặc định](192-change-default-storage-class-vi.md).

Khi một PVC không chỉ định `storageClassName`, StorageClass mặc định sẽ được
sử dụng.

Nếu bạn đặt annotation
[`storageclass.kubernetes.io/is-default-class`](https://kubernetes.io/docs/reference/labels-annotations-taints/#storageclass-kubernetes-io-is-default-class)
thành true trên nhiều hơn một StorageClass trong cluster của bạn, và sau đó bạn
tạo một PersistentVolumeClaim không đặt `storageClassName`, Kubernetes
sẽ dùng StorageClass mặc định được tạo gần đây nhất.

> **Ghi chú:**
> Bạn nên cố gắng chỉ có một StorageClass trong cluster được
> đánh dấu là mặc định. Lý do Kubernetes cho phép bạn có
> nhiều StorageClass mặc định là để cho phép việc chuyển đổi diễn ra liền mạch.

Bạn có thể tạo một PersistentVolumeClaim mà không chỉ định `storageClassName`
cho PVC mới, và bạn có thể làm vậy ngay cả khi không tồn tại StorageClass mặc định nào
trong cluster. Trong trường hợp này, PVC mới được tạo đúng như bạn định nghĩa, và
`storageClassName` của PVC đó vẫn không được đặt cho đến khi có một giá trị mặc định khả dụng.

Bạn có thể có một cluster không có bất kỳ StorageClass mặc định nào. Nếu bạn không đánh dấu
StorageClass nào là mặc định (và cũng chưa có cái nào được thiết lập sẵn cho bạn bởi, ví dụ, một nhà cung cấp đám mây),
thì Kubernetes không thể áp dụng cơ chế đặt mặc định đó cho các PersistentVolumeClaim cần
đến nó.

Nếu hoặc khi một StorageClass mặc định trở nên khả dụng, control plane sẽ xác định các
PVC hiện có không có `storageClassName`. Với các PVC có giá trị `storageClassName`
rỗng hoặc không có khóa này, control plane sau đó
cập nhật các PVC đó để đặt `storageClassName` khớp với StorageClass mặc định mới.
Nếu bạn có một PVC hiện hữu mà `storageClassName` là `""`, và bạn cấu hình
một StorageClass mặc định, thì PVC này sẽ không bị cập nhật.

Để tiếp tục gắn kết (bind) với các PV có `storageClassName` đặt là `""`
(trong khi đang tồn tại một StorageClass mặc định), bạn cần đặt `storageClassName`
của PVC liên quan thành `""`.

## Provisioner

Mỗi StorageClass có một provisioner xác định volume plugin nào được dùng
để cấp phát các PV. Trường này phải được chỉ định.

| Volume Plugin        | Provisioner nội bộ |            Ví dụ cấu hình             |
| :------------------- | :------------------: | :-----------------------------------: |
| AzureFile            |       &#x2713;       |       [Azure File](#azure-file)       |
| CephFS               |          -           |                   -                   |
| FC                   |          -           |                   -                   |
| FlexVolume           |          -           |                   -                   |
| iSCSI                |          -           |                   -                   |
| Local                |          -           |            [Local](#local)            |
| NFS                  |          -           |              [NFS](#nfs)              |
| PortworxVolume       |       &#x2713;       |  [Portworx Volume](#portworx-volume)  |
| RBD                  |          -           |         [Ceph RBD](#ceph-rbd)         |
| VsphereVolume        |       &#x2713;       |          [vSphere](#vsphere)          |

Bạn không bị giới hạn trong việc chỉ định các provisioner "nội bộ"
được liệt kê ở đây (có tên bắt đầu bằng tiền tố "kubernetes.io" và được phân phối
cùng với Kubernetes). Bạn cũng có thể chạy và chỉ định các provisioner bên ngoài (external provisioner),
là các chương trình độc lập tuân theo một [đặc tả](https://git.k8s.io/design-proposals-archive/storage/volume-provisioning.md)
do Kubernetes định nghĩa. Tác giả của các provisioner bên ngoài có toàn quyền quyết định
nơi đặt mã nguồn của họ, cách provisioner được phân phối, cách nó cần được
chạy, volume plugin nào nó sử dụng (bao gồm cả Flex), v.v. Repository
[kubernetes-sigs/sig-storage-lib-external-provisioner](https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner)
chứa một thư viện để viết các provisioner bên ngoài, hiện thực phần lớn
đặc tả nói trên. Một số provisioner bên ngoài được liệt kê trong repository
[kubernetes-sigs/sig-storage-lib-external-provisioner](https://github.com/kubernetes-sigs/sig-storage-lib-external-provisioner).

Ví dụ, NFS không cung cấp provisioner nội bộ, nhưng có thể dùng một provisioner
bên ngoài. Cũng có những trường hợp các nhà cung cấp lưu trữ bên thứ 3
cung cấp provisioner bên ngoài của riêng họ.

## Chính sách thu hồi (Reclaim policy) {#reclaim-policy}

Các PersistentVolume được tạo động bởi một StorageClass sẽ có
[chính sách thu hồi (reclaim policy)](92-persistent-volumes-vi.md#reclaiming)
được chỉ định trong trường `reclaimPolicy` của lớp đó, có thể là
`Delete` hoặc `Retain`. Nếu không có `reclaimPolicy` nào được chỉ định khi một
đối tượng StorageClass được tạo, giá trị mặc định sẽ là `Delete`.

Các PersistentVolume được tạo thủ công và được quản lý thông qua một StorageClass sẽ có
bất kỳ chính sách thu hồi nào được gán cho chúng lúc tạo.

## Mở rộng volume (Volume expansion) {#allow-volume-expansion}

Các PersistentVolume có thể được cấu hình để có thể mở rộng. Điều này cho phép bạn thay đổi
kích thước volume bằng cách chỉnh sửa đối tượng PVC tương ứng, yêu cầu một lượng
lưu trữ mới lớn hơn.

Các loại volume sau đây hỗ trợ mở rộng volume, khi StorageClass
bên dưới có trường `allowVolumeExpansion` đặt là true.

*Bảng các loại volume và phiên bản Kubernetes mà chúng yêu cầu*

| Loại volume          | Phiên bản Kubernetes yêu cầu để mở rộng volume   |
| :------------------- | :----------------------------------------------- |
| Azure File           | 1.11                                             |
| CSI                  | 1.24                                             |
| FlexVolume           | 1.13                                             |
| Portworx             | 1.11                                             |
| rbd                  | 1.11                                             |

> **Ghi chú:**
> Bạn chỉ có thể dùng tính năng mở rộng volume để tăng kích thước một Volume, không thể thu nhỏ nó.

## Tùy chọn mount (Mount options)

Các PersistentVolume được tạo động bởi một StorageClass sẽ có các
tùy chọn mount được chỉ định trong trường `mountOptions` của lớp đó.

Nếu volume plugin không hỗ trợ tùy chọn mount nhưng tùy chọn mount lại được
chỉ định, việc cấp phát sẽ thất bại. Tùy chọn mount **không** được kiểm tra tính hợp lệ ở cả
lớp lẫn PV. Nếu một tùy chọn mount không hợp lệ, việc mount PV sẽ thất bại.

## Chế độ gắn kết volume (Volume binding mode) {#volume-binding-mode}

Trường `volumeBindingMode` kiểm soát thời điểm
[gắn kết volume và cấp phát động](92-persistent-volumes-vi.md#provisioning)
diễn ra. Khi không được đặt, chế độ `Immediate` được dùng theo mặc định.

Chế độ `Immediate` cho biết việc gắn kết volume và cấp phát
động diễn ra ngay khi PersistentVolumeClaim được tạo. Với các backend lưu trữ
bị ràng buộc theo topology và không thể truy cập toàn cục từ mọi Node
trong cluster, các PersistentVolume sẽ được gắn kết hoặc cấp phát mà không biết gì về các yêu cầu
lập lịch của Pod. Điều này có thể dẫn đến các Pod không thể lập lịch được.

Quản trị viên cluster có thể giải quyết vấn đề này bằng cách chỉ định chế độ `WaitForFirstConsumer`,
chế độ này sẽ trì hoãn việc gắn kết và cấp phát một PersistentVolume cho đến khi một Pod dùng PersistentVolumeClaim đó được tạo.
Các PersistentVolume sẽ được chọn hoặc cấp phát phù hợp với topology được
chỉ định bởi các ràng buộc lập lịch của Pod. Chúng bao gồm, nhưng không giới hạn ở, [yêu cầu
tài nguyên](110-manage-resources-containers-vi.md),
[node selector](138-assign-pod-node-vi.md#nodeselector),
[pod affinity và
anti-affinity](138-assign-pod-node-vi.md#affinity-and-anti-affinity),
và [taint và toleration](139-taint-and-toleration-vi.md).

Các plugin sau hỗ trợ `WaitForFirstConsumer` với cấp phát động:

- Các volume CSI, với điều kiện driver CSI cụ thể hỗ trợ điều này

Các plugin sau hỗ trợ `WaitForFirstConsumer` với gắn kết PersistentVolume được tạo sẵn:

- Các volume CSI, với điều kiện driver CSI cụ thể hỗ trợ điều này
- [`local`](#local)

> **Ghi chú:**
> Nếu bạn chọn dùng `WaitForFirstConsumer`, đừng dùng `nodeName` trong spec của Pod
> để chỉ định node affinity.
> Nếu `nodeName` được dùng trong trường hợp này, bộ lập lịch sẽ bị bỏ qua và PVC sẽ đứng yên ở trạng thái `pending`.
>
> Thay vào đó, bạn có thể dùng node selector cho `kubernetes.io/hostname`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  nodeSelector:
    kubernetes.io/hostname: kube-01
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

## Các topology được phép (Allowed topologies)

Khi người vận hành cluster chỉ định chế độ gắn kết volume `WaitForFirstConsumer`, trong phần lớn các tình huống
không còn cần thiết phải giới hạn việc cấp phát vào các topology cụ thể nữa. Tuy nhiên,
nếu vẫn cần, có thể chỉ định `allowedTopologies`.

Ví dụ này minh họa cách giới hạn topology của các volume được cấp phát vào các
zone cụ thể và nên được dùng thay thế cho các tham số `zone` và `zones` đối với các
plugin được hỗ trợ.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner:  example.com/example
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: topology.kubernetes.io/zone
    values:
    - us-central-1a
    - us-central-1b
```

## Tham số (Parameters)

Các StorageClass có các tham số mô tả những volume thuộc về storage
class đó. Các tham số khác nhau có thể được chấp nhận tùy theo `provisioner`.
Khi một tham số bị bỏ qua, một giá trị mặc định nào đó sẽ được sử dụng.

Có thể định nghĩa tối đa 512 tham số cho một StorageClass.
Tổng chiều dài của đối tượng tham số, bao gồm cả các khóa và giá trị của nó, không được
vượt quá 256 KiB.

### AWS EBS

Kubernetes v1.36 không bao gồm loại volume `awsElasticBlockStore`.

Driver lưu trữ in-tree AWSElasticBlockStore đã bị ngừng hỗ trợ (deprecated) từ bản phát hành Kubernetes v1.19
và sau đó bị gỡ bỏ hoàn toàn trong bản phát hành v1.27.

Dự án Kubernetes khuyến nghị bạn dùng driver lưu trữ out-of-tree
[AWS EBS](https://github.com/kubernetes-sigs/aws-ebs-csi-driver) thay thế.

Đây là một ví dụ StorageClass cho driver CSI AWS EBS:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  csi.storage.k8s.io/fstype: xfs
  type: io1
  iopsPerGB: "50"
  encrypted: "true"
  tagSpecification_1: "key1=value1"
  tagSpecification_2: "key2=value2"
allowedTopologies:
- matchLabelExpressions:
  - key: topology.ebs.csi.aws.com/zone
    values:
    - us-east-2c
```

`tagSpecification`: Các tag có tiền tố này được áp dụng cho các volume EBS được cấp phát động.

### AWS EFS

Để cấu hình lưu trữ AWS EFS, bạn có thể dùng driver out-of-tree [AWS_EFS_CSI_DRIVER](https://github.com/kubernetes-sigs/aws-efs-csi-driver).

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-92107410
  directoryPerms: "700"
```

- `provisioningMode`: Loại volume sẽ được cấp phát bởi Amazon EFS. Hiện tại, chỉ hỗ trợ cấp phát dựa trên access point (`efs-ap`).
- `fileSystemId`: File system mà access point được tạo bên trong nó.
- `directoryPerms`: Quyền thư mục của thư mục gốc được tạo bởi access point.

Để biết thêm chi tiết, tham khảo tài liệu [AWS_EFS_CSI_Driver Dynamic Provisioning](https://github.com/kubernetes-sigs/aws-efs-csi-driver/blob/master/examples/kubernetes/dynamic_provisioning/README.md).


### NFS

Để cấu hình lưu trữ NFS, bạn có thể dùng driver in-tree hoặc
[driver CSI NFS cho Kubernetes](https://github.com/kubernetes-csi/csi-driver-nfs#readme)
(khuyến nghị).

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: example-nfs
provisioner: example.com/external-nfs
parameters:
  server: nfs-server.example.com
  path: /share
  readOnly: "false"
```

- `server`: Server là hostname hoặc địa chỉ IP của NFS server.
- `path`: Đường dẫn được export bởi NFS server.
- `readOnly`: Cờ cho biết liệu lưu trữ có được mount ở chế độ chỉ đọc hay không (mặc định là false).

Kubernetes không bao gồm provisioner NFS nội bộ.
Bạn cần dùng một provisioner bên ngoài để tạo StorageClass cho NFS.
Dưới đây là một số ví dụ:

- [NFS Ganesha server and external provisioner](https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner)
- [NFS subdir external provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner)

### vSphere

Có hai loại provisioner cho các storage class vSphere:

- [Provisioner CSI](#vsphere-provisioner-csi): `csi.vsphere.vmware.com`
- [Provisioner vCP](#vcp-provisioner): `kubernetes.io/vsphere-volume`

Các provisioner in-tree đã [bị ngừng hỗ trợ (deprecated)](https://kubernetes.io/blog/2019/12/09/kubernetes-1-17-feature-csi-migration-beta/#why-are-we-migrating-in-tree-plugins-to-csi).
Để biết thêm thông tin về provisioner CSI, xem
[Kubernetes vSphere CSI Driver](https://vsphere-csi-driver.sigs.k8s.io/) và
[vSphereVolume CSI migration](https://kubernetes.io/docs/concepts/storage/volumes#vsphere-csi-migration).

#### CSI Provisioner {#vsphere-provisioner-csi}

Provisioner StorageClass vSphere CSI hoạt động với các cluster Tanzu Kubernetes.
Để xem một ví dụ, tham khảo [repository vSphere CSI](https://github.com/kubernetes-sigs/vsphere-csi-driver/blob/master/example/vanilla-k8s-RWM-filesystem-volumes/example-sc.yaml).

#### vCP Provisioner

Các ví dụ sau đây dùng provisioner StorageClass của VMware Cloud Provider (vCP).

1. Tạo một StorageClass với định dạng đĩa do người dùng chỉ định.

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: fast
   provisioner: kubernetes.io/vsphere-volume
   parameters:
     diskformat: zeroedthick
   ```

   `diskformat`: `thin`, `zeroedthick` và `eagerzeroedthick`. Mặc định: `"thin"`.

2. Tạo một StorageClass với định dạng đĩa trên một datastore do người dùng chỉ định.

   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: fast
   provisioner: kubernetes.io/vsphere-volume
   parameters:
     diskformat: zeroedthick
     datastore: VSANDatastore
   ```

   `datastore`: Người dùng cũng có thể chỉ định datastore trong StorageClass.
   Volume sẽ được tạo trên datastore được chỉ định trong StorageClass,
   trong trường hợp này là `VSANDatastore`. Trường này là tùy chọn. Nếu
   datastore không được chỉ định, thì volume sẽ được tạo trên datastore
   được chỉ định trong file cấu hình vSphere dùng để khởi tạo vSphere Cloud
   Provider.

3. Quản lý Storage Policy bên trong kubernetes

   - Dùng chính sách SPBM vCenter có sẵn

     Một trong những tính năng quan trọng nhất của vSphere cho việc Quản lý Lưu trữ là
     quản lý dựa trên chính sách. Storage Policy Based Management (SPBM) là một
     framework chính sách lưu trữ cung cấp một mặt phẳng điều khiển hợp nhất duy nhất
     trải rộng trên nhiều dịch vụ dữ liệu và giải pháp lưu trữ. SPBM giúp
     quản trị viên vSphere vượt qua các thách thức về cấp phát lưu trữ từ trước,
     chẳng hạn như hoạch định dung lượng, các mức dịch vụ khác biệt và quản lý
     phần dung lượng dự phòng (capacity headroom).

     Các chính sách SPBM có thể được chỉ định trong StorageClass bằng tham số
     `storagePolicyName`.

   - Hỗ trợ chính sách Virtual SAN bên trong Kubernetes

     Quản trị viên Vsphere Infrastructure (VI) sẽ có khả năng chỉ định các
     Virtual SAN Storage Capabilities tùy chỉnh trong quá trình cấp phát volume động. Giờ đây
     bạn có thể định nghĩa các yêu cầu lưu trữ, chẳng hạn như hiệu năng và tính sẵn sàng,
     dưới dạng các storage capability trong quá trình cấp phát volume động.
     Các yêu cầu storage capability được chuyển đổi thành một chính sách Virtual SAN,
     sau đó được đẩy xuống lớp Virtual SAN khi một
     persistent volume (đĩa ảo) được tạo. Đĩa ảo được
     phân tán trên datastore Virtual SAN để đáp ứng các yêu cầu.

     Bạn có thể xem [Storage Policy Based Management for dynamic provisioning of volumes](https://github.com/vmware-archive/vsphere-storage-for-kubernetes/blob/fa4c8b8ad46a85b6555d715dd9d27ff69839df53/documentation/policy-based-mgmt.md)
     để biết thêm chi tiết về cách dùng chính sách lưu trữ cho việc quản lý
     persistent volume.

### Ceph RBD (deprecated) {#ceph-rbd}

> **Ghi chú:**
> **TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.28 [deprecated]`
>
> Provisioner nội bộ này của Ceph RBD đã bị ngừng hỗ trợ (deprecated). Vui lòng dùng
> [driver CSI CephFS RBD](https://github.com/ceph/ceph-csi).

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
provisioner: kubernetes.io/rbd # Provisioner này đã bị ngừng hỗ trợ (deprecated)
parameters:
  monitors: 198.19.254.105:6789
  adminId: kube
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: kube
  userId: kube
  userSecretName: ceph-secret-user
  userSecretNamespace: default
  fsType: ext4
  imageFormat: "2"
  imageFeatures: "layering"
```

- `monitors`: Các Ceph monitor, phân tách bằng dấu phẩy. Tham số này là bắt buộc.
- `adminId`: Ceph client ID có khả năng tạo image trong pool.
  Mặc định là "admin".
- `adminSecretName`: Tên Secret cho `adminId`. Tham số này là bắt buộc.
  Secret được cung cấp phải có type "kubernetes.io/rbd".
- `adminSecretNamespace`: Namespace cho `adminSecretName`. Mặc định là "default".
- `pool`: Pool Ceph RBD. Mặc định là "rbd".
- `userId`: Ceph client ID được dùng để map image RBD. Mặc định giống
  với `adminId`.
- `userSecretName`: Tên của Ceph Secret cho `userId` dùng để map image RBD. Nó
  phải tồn tại trong cùng namespace với các PVC. Tham số này là bắt buộc.
  Secret được cung cấp phải có type "kubernetes.io/rbd", ví dụ được tạo theo cách
  sau:

  ```shell
  kubectl create secret generic ceph-secret --type="kubernetes.io/rbd" \
    --from-literal=key='QVFEQ1pMdFhPUnQrSmhBQUFYaERWNHJsZ3BsMmNjcDR6RFZST0E9PQ==' \
    --namespace=kube-system
  ```

- `userSecretNamespace`: Namespace cho `userSecretName`.
- `fsType`: fsType được kubernetes hỗ trợ. Mặc định: `"ext4"`.
- `imageFormat`: Định dạng image Ceph RBD, "1" hoặc "2". Mặc định là "2".
- `imageFeatures`: Tham số này là tùy chọn và chỉ nên được dùng nếu bạn
  đặt `imageFormat` là "2". Các tính năng hiện được hỗ trợ chỉ có `layering`.
  Mặc định là "", và không có tính năng nào được bật.

### Azure Disk

Kubernetes v1.36 không bao gồm loại volume `azureDisk`.

Driver lưu trữ in-tree `azureDisk` đã bị ngừng hỗ trợ (deprecated) từ bản phát hành Kubernetes v1.19
và sau đó bị gỡ bỏ hoàn toàn trong bản phát hành v1.27.

Dự án Kubernetes khuyến nghị bạn dùng driver lưu trữ bên thứ ba
[Azure Disk](https://github.com/kubernetes-sigs/azuredisk-csi-driver) thay thế.

### Azure File (deprecated) {#azure-file}

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
parameters:
  skuName: Standard_LRS
  location: eastus
  storageAccount: azure_storage_account_name # giá trị ví dụ
```

- `skuName`: Bậc SKU của tài khoản lưu trữ Azure. Mặc định là rỗng.
- `location`: Vị trí (location) của tài khoản lưu trữ Azure. Mặc định là rỗng.
- `storageAccount`: Tên tài khoản lưu trữ Azure. Mặc định là rỗng. Nếu một tài khoản
  lưu trữ không được cung cấp, tất cả các tài khoản lưu trữ liên kết với resource
  group sẽ được tìm kiếm để tìm một tài khoản khớp với `skuName` và `location`. Nếu một
  tài khoản lưu trữ được cung cấp, nó phải nằm trong cùng resource group với
  cluster, và `skuName` cùng `location` sẽ bị bỏ qua.
- `secretNamespace`: namespace của secret chứa Azure Storage
  Account Name và Key. Mặc định giống với namespace của Pod.
- `secretName`: tên của secret chứa Azure Storage Account Name và
  Key. Mặc định là `azure-storage-account-<accountName>-secret`
- `readOnly`: cờ cho biết liệu lưu trữ có được mount ở chế độ chỉ đọc hay không.
  Mặc định là false, nghĩa là mount đọc/ghi. Thiết lập này cũng ảnh hưởng đến
  thiết lập `ReadOnly` trong VolumeMounts.

Trong quá trình cấp phát lưu trữ, một secret có tên theo `secretName` được tạo cho
thông tin xác thực để mount. Nếu cluster đã bật cả
[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) lẫn
[Controller Roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#controller-roles),
hãy thêm quyền `create` đối với tài nguyên `secret` cho clusterrole
`system:controller:persistent-volume-binder`.

Trong bối cảnh đa người thuê (multi-tenancy), rất khuyến nghị đặt giá trị cho
`secretNamespace` một cách tường minh, nếu không thông tin xác thực của tài khoản lưu trữ có thể
bị các người dùng khác đọc được.

### Portworx volume (deprecated) {#portworx-volume}

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: portworx-io-priority-high
provisioner: kubernetes.io/portworx-volume # Provisioner này đã bị ngừng hỗ trợ (deprecated)
parameters:
  repl: "1"
  snap_interval: "70"
  priority_io: "high"
```

- `fs`: filesystem sẽ được tạo: `none/xfs/ext4` (mặc định: `ext4`).
- `block_size`: kích thước block tính bằng Kbyte (mặc định: `32`).
- `repl`: số bản sao (replica) đồng bộ được cung cấp dưới dạng
  hệ số sao chép `1..3` (mặc định: `1`). Ở đây cần một chuỗi, tức là
  `"1"` chứ không phải `1`.
- `priority_io`: quyết định volume sẽ được tạo từ lưu trữ hiệu năng
  cao hơn hay có mức ưu tiên thấp hơn `high/medium/low` (mặc định: `low`).
- `snap_interval`: khoảng thời gian đồng hồ/thời gian tính bằng phút để kích hoạt snapshot.
  Các snapshot là gia tăng (incremental) dựa trên khác biệt so với snapshot trước đó, 0
  tắt snapshot (mặc định: `0`). Ở đây cần một chuỗi, tức là
  `"70"` chứ không phải `70`.
- `aggregation_level`: chỉ định số phần (chunk) mà volume sẽ được
  phân tán vào, 0 cho biết một volume không gộp (non-aggregated) (mặc định: `0`). Ở đây cần
  một chuỗi, tức là `"0"` chứ không phải `0`
- `ephemeral`: chỉ định volume nên được dọn dẹp sau khi unmount
  hay nên là bền vững. Trường hợp dùng `emptyDir` có thể đặt giá trị này là true và
  trường hợp dùng `persistent volumes` chẳng hạn cho các cơ sở dữ liệu như Cassandra nên đặt
  là false, `true/false` (mặc định `false`). Ở đây cần một chuỗi, tức là
  `"true"` chứ không phải `true`.

### Local

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner # cho biết StorageClass này không hỗ trợ cấp phát tự động
volumeBindingMode: WaitForFirstConsumer
```

Các volume local không hỗ trợ cấp phát động trong Kubernetes v1.36;
tuy nhiên vẫn nên tạo một StorageClass để trì hoãn việc gắn kết volume cho đến khi một Pod thực sự
được lập lịch lên node phù hợp. Điều này được chỉ định bởi chế độ gắn kết
volume `WaitForFirstConsumer`.

Việc trì hoãn gắn kết volume cho phép bộ lập lịch xem xét tất cả các
ràng buộc lập lịch của Pod khi chọn một PersistentVolume phù hợp cho một
PersistentVolumeClaim.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Cluster lab của bạn có hai worker và chỉ `lab-k8s-worker2` có đĩa dữ liệu dành cho lưu trữ.
   StorageClass để nguyên `volumeBindingMode` mặc định. Bạn tạo PVC trước rồi mới tạo Pod.
   Rủi ro là gì, và `WaitForFirstConsumer` sửa được chính xác điều gì?
2. StorageClass của bạn không khai `reclaimPolicy`. Xóa PVC thì dữ liệu còn không? Muốn giữ
   lại thì sửa ở đâu, và việc sửa đó có tác dụng với các PV đã được cấp phát trước đó không?
3. Cluster có hai StorageClass cùng đặt annotation mặc định là true. Bạn tạo một PVC không đặt
   `storageClassName`. Class nào được dùng, và bài khuyên gì?
4. `allowVolumeExpansion: true` cho phép bạn làm gì và **không** cho phép làm gì? Trường này
   đặt ở đâu — trên PVC hay trên StorageClass?
5. Cluster lab hiện **chưa có StorageClass và chưa có provisioner**. Nếu bạn chỉ `kubectl apply`
   một đối tượng StorageClass mà không làm gì thêm, cluster đã cấp phát được volume chưa?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Với `Immediate` — chế độ **mặc định khi bạn không khai `volumeBindingMode`** — việc gắn kết
   và cấp phát diễn ra **ngay khi PVC được tạo**, tức là trước khi Kubernetes biết Pod nào sẽ
   dùng nó. Với backend bị ràng buộc theo topology và không truy cập được từ mọi node, PV có
   thể bị cấp phát ở nơi Pod không lập lịch tới được, và **Pod trở thành không lập lịch được**.
   `WaitForFirstConsumer` **hoãn cả việc bind lẫn việc cấp phát cho tới khi có Pod dùng PVC
   đó**, nhờ vậy PV được chọn hoặc tạo phù hợp với các ràng buộc lập lịch của Pod: yêu cầu tài
   nguyên, node selector, pod affinity/anti-affinity, taint và toleration. Lưu ý cái bẫy kèm
   theo: đã dùng `WaitForFirstConsumer` thì **đừng dùng `nodeName`** trong spec Pod, vì làm vậy
   là bỏ qua scheduler và PVC sẽ đứng yên ở `pending`; hãy dùng node selector
   `kubernetes.io/hostname`.
2. **Dữ liệu mất.** Không khai `reclaimPolicy` thì giá trị mặc định là **`Delete`**, và PV cấp
   phát động sẽ mang policy đó. Muốn giữ thì đặt `reclaimPolicy: Retain` **trên StorageClass**.
   Việc sửa đó **chỉ có tác dụng với các PV được cấp phát sau đó**: PV nhận reclaim policy của
   class tại thời điểm nó được cấp phát, còn PV tạo thủ công thì giữ nguyên policy được gán lúc
   tạo. Các PV đã tồn tại phải sửa hoặc patch từng cái.
3. **Class được tạo gần đây nhất.** Bài nói rõ Kubernetes cho phép nhiều class mặc định chỉ để
   việc chuyển đổi diễn ra liền mạch, và **khuyên bạn cố gắng chỉ giữ đúng một** class được
   đánh dấu mặc định.
4. Nó cho phép **tăng** kích thước một volume bằng cách sửa PVC để xin dung lượng lớn hơn. Nó
   **không** cho phép thu nhỏ volume. Và trường này nằm **trên StorageClass**, không phải trên
   PVC — PVC chỉ mở rộng được nếu storage class bên dưới nó đã bật cờ này.
5. **Chưa.** `provisioner` là trường bắt buộc và nó chỉ *khai báo* volume plugin nào sẽ cấp
   phát PV; bản thân đối tượng StorageClass không cấp phát gì cả. Phải có một provisioner —
   nội bộ hoặc một external provisioner đang chạy trong cluster — thì mới có thứ đứng ra tạo
   volume. Đó chính là việc Lab 6a phải làm trước khi bất kỳ PVC nào của bạn được bind.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
