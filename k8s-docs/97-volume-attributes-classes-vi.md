# Lớp thuộc tính Volume (Volume Attributes Classes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/volume-attributes-classes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](LO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 9/16 · Kiểm chứng ở
Lab 6b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Từ bài này trở đi là phần nâng cao của giai đoạn 6, và **tất cả đều phụ thuộc vào việc CSI
driver bạn cài ở Lab 6a có hỗ trợ hay không**. Đọc để biết cơ chế và biết cách kiểm tra điều
kiện, đừng kỳ vọng chạy được ngay. Bài rất ngắn; điều duy nhất đáng nhớ là VolumeAttributesClass
đổi được **sau khi** volume đã tồn tại — khác hẳn StorageClass.

**Phải hiểu ở lần đọc này:**

- VolumeAttributesClass mô tả các "lớp" lưu trữ **có thể thay đổi được (mutable)**, thường ứng
  với các mức chất lượng dịch vụ; Kubernetes không áp đặt ý nghĩa cho chúng — đoạn mở đầu.
- Điều kiện dùng được: **chỉ với lưu trữ qua Container Storage Interface**, và **chỉ khi CSI
  driver liên quan có triển khai API `ModifyVolume`** — đoạn mở đầu.
- Cái gì đổi được, cái gì không: **tên** class trong `volumeAttributesClassName` của PVC đổi
  được, nhưng **các tham số bên trong một class đã tồn tại là bất biến**. Muốn đổi tham số thì
  tạo class mới rồi trỏ PVC sang, đúng như ví dụ `silver` → `gold` — mục *API
  VolumeAttributesClass* và *Trình thay đổi kích thước*.
- Cùng một `driverName` phục vụ hai vai trò ở hai thời điểm: **provisioner** dùng khi PV thuộc
  lớp đó được cấp phát động, **resizer** dùng khi PV đang có bị chỉnh sửa — mục *Trình cấp
  phát* và *Trình thay đổi kích thước*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ví dụ `pd.csi.storage.gke.io`, `provisioned-iops`, `throughput` | tham số đặc thù GCE PD, lab không dùng | không cần |
| Chi tiết external-provisioner và external-resizer | thuộc phần triển khai CSI | Lab 6b |
| Giới hạn 512 tham số và 256 KiB trong mục *Tham số* | ngưỡng kỹ thuật, tra khi cần | không cần |

---

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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. StorageClass và VolumeAttributesClass khác nhau ở điểm căn bản nào?
2. Bạn cần nâng `iops` của lớp `silver` từ 3000 lên 4000 cho một PVC đang chạy. Sửa thẳng đối
   tượng `silver` được không? Quy trình đúng theo bài là gì?
3. Cùng một `driverName` được bài nhắc tới trong hai mục khác nhau. Hai vai trò đó là gì và
   xảy ra vào lúc nào?
4. Sau Lab 6a cluster lab của bạn sẽ có một provisioner đang chạy. Bạn cần kiểm tra hai điều
   gì trước khi kết luận VolumeAttributesClass dùng được ở đây?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **StorageClass mô tả lớp lưu trữ lúc cấp phát, VolumeAttributesClass mô tả các lớp *có thể
   thay đổi được* sau khi volume đã tồn tại.** Đó là lý do bài gọi chúng là "mutable class":
   bạn đổi mức chất lượng dịch vụ của một volume đang chạy bằng cách trỏ PVC sang một class
   khác, không cần tạo lại volume.
2. **Không sửa thẳng được.** Tên class trong PVC thì thay đổi được, nhưng **các tham số trong
   một lớp đã tồn tại là bất biến**. Quy trình đúng chính là ví dụ của bài: tạo một
   VolumeAttributesClass mới (`gold`) với tham số mong muốn, rồi cập nhật
   `volumeAttributesClassName` của PVC từ `silver` sang `gold` và apply.
3. **Provisioner và resizer.** Với vai trò provisioner, `driverName` xác định volume plugin nào
   cấp phát PV **khi PV thuộc lớp đó được cấp phát động**. Với vai trò resizer, cũng chính
   `driverName` đó xác định plugin nào **chỉnh sửa một PV đang có** khi bạn đổi class của PVC.
   Trường này bắt buộc phải chỉ định trong cả hai trường hợp.
4. Thứ nhất, **lưu trữ có đi qua Container Storage Interface không** — VolumeAttributesClass
   chỉ dùng được với CSI. Thứ hai, **CSI driver đó có triển khai API `ModifyVolume` không** —
   không có thì không đổi được thuộc tính của volume đang chạy. Thiếu một trong hai thì phần
   này chỉ dừng ở mức đọc, và đó chính là lý do bài được xếp kiểm chứng ở Lab 6b chứ không
   phải Lab 6a.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
