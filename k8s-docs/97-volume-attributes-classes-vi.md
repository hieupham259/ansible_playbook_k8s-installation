# Lớp thuộc tính Volume (Volume Attributes Classes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/volume-attributes-classes/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [stable]`

Trang này giả định rằng bạn đã quen thuộc với [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/),
[volume](https://kubernetes.io/docs/concepts/storage/volumes/) và [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
trong Kubernetes.

VolumeAttributesClass cung cấp một cách để quản trị viên mô tả các "lớp" (class) lưu trữ
có thể thay đổi được (mutable) mà họ cung cấp. Các lớp khác nhau có thể tương ứng với các
mức chất lượng dịch vụ (quality-of-service) khác nhau.
Bản thân Kubernetes không áp đặt quan điểm về việc các lớp này đại diện cho điều gì.

Tính năng này đã đạt mức phổ biến rộng rãi (generally available - GA) kể từ phiên bản 1.34,
và người dùng có tùy chọn tắt nó đi.

Bạn cũng chỉ có thể dùng VolumeAttributesClass với lưu trữ được hỗ trợ bởi
Container Storage Interface, và chỉ khi CSI driver liên quan có triển khai API `ModifyVolume`.

## API VolumeAttributesClass (The VolumeAttributesClass API)

Mỗi VolumeAttributesClass chứa `driverName` và `parameters`, chúng được dùng khi một
PersistentVolume (PV) thuộc lớp đó cần được cấp phát động (dynamically provisioned)
hoặc chỉnh sửa.

Tên của một đối tượng VolumeAttributesClass có ý nghĩa quan trọng, và là cách để người dùng
yêu cầu một lớp cụ thể. Quản trị viên đặt tên và các tham số khác của một lớp khi tạo
các đối tượng VolumeAttributesClass lần đầu.
Trong khi tên của đối tượng VolumeAttributesClass trong một `PersistentVolumeClaim` có thể
thay đổi được, thì các tham số trong một lớp đã tồn tại là bất biến (immutable).

```yaml
apiVersion: storage.k8s.io/v1
kind: VolumeAttributesClass
metadata:
  name: silver
driverName: pd.csi.storage.gke.io
parameters:
  provisioned-iops: "3000"
  provisioned-throughput: "50" 
```

### Trình cấp phát (Provisioner)

Mỗi VolumeAttributesClass có một trình cấp phát (provisioner) xác định volume plugin nào
được dùng để cấp phát PV. Trường `driverName` phải được chỉ định.

Việc hỗ trợ tính năng VolumeAttributesClass được triển khai trong
[kubernetes-csi/external-provisioner](https://github.com/kubernetes-csi/external-provisioner).

Bạn không bị giới hạn ở việc chỉ định [kubernetes-csi/external-provisioner](https://github.com/kubernetes-csi/external-provisioner).
Bạn cũng có thể chạy và chỉ định các trình cấp phát bên ngoài (external provisioner),
là những chương trình độc lập tuân theo một đặc tả do Kubernetes định nghĩa.
Tác giả của các trình cấp phát bên ngoài có toàn quyền quyết định nơi đặt mã nguồn,
cách trình cấp phát được phân phối, cách nó cần được chạy, nó dùng volume plugin nào, v.v.

Để hiểu cách trình cấp phát hoạt động với VolumeAttributesClass, hãy tham khảo
[tài liệu CSI external-provisioner](https://kubernetes-csi.github.io/docs/external-provisioner.html).

### Trình thay đổi kích thước (Resizer)

Mỗi VolumeAttributesClass có một trình thay đổi kích thước (resizer) xác định volume plugin
nào được dùng để chỉnh sửa PV. Trường `driverName` phải được chỉ định.

Việc hỗ trợ tính năng chỉnh sửa volume cho VolumeAttributesClass được triển khai trong
[kubernetes-csi/external-resizer](https://github.com/kubernetes-csi/external-resizer).

Ví dụ, một PersistentVolumeClaim hiện có đang dùng một VolumeAttributesClass tên là silver:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pv-claim
spec:
  …
  volumeAttributesClassName: silver
  …
```

Một VolumeAttributesClass mới tên là gold đã có sẵn trong cluster:

```yaml
apiVersion: storage.k8s.io/v1
kind: VolumeAttributesClass
metadata:
  name: gold
driverName: pd.csi.storage.gke.io
parameters:
  iops: "4000"
  throughput: "60"
```

Người dùng cuối có thể cập nhật PVC với VolumeAttributesClass gold mới và apply:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pv-claim
spec:
  …
  volumeAttributesClassName: gold
  …
```

Để hiểu cách trình thay đổi kích thước hoạt động với VolumeAttributesClass, hãy tham khảo
[tài liệu CSI external-resizer](https://kubernetes-csi.github.io/docs/external-resizer.html).

## Tham số (Parameters)

Các VolumeAttributesClass có những tham số mô tả các volume thuộc về chúng. Các tham số
được chấp nhận có thể khác nhau tùy theo trình cấp phát hoặc trình thay đổi kích thước.
Ví dụ, giá trị `4000` cho tham số `iops`, và tham số `throughput` là đặc thù của GCE PD.
Khi một tham số bị bỏ qua, giá trị mặc định sẽ được dùng lúc cấp phát volume.
Nếu người dùng apply PVC với một VolumeAttributesClass khác mà bỏ qua các tham số,
giá trị mặc định của các tham số có thể được dùng tùy theo cách triển khai của CSI driver.
Vui lòng tham khảo tài liệu của CSI driver liên quan để biết thêm chi tiết.

Một VolumeAttributesClass có thể định nghĩa tối đa 512 tham số.
Tổng độ dài của đối tượng tham số, bao gồm cả các khóa (key) và giá trị (value),
không được vượt quá 256 KiB.
