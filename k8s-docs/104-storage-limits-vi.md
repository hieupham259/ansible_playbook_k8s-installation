# Giới hạn volume theo từng Node (Node-specific Volume Limits)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/storage-limits/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 15/16 · Kiểm chứng ở
Lab 6b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này là ràng buộc lập lịch thứ hai đến từ lưu trữ: sau "node còn đủ dung lượng không" của
bài [103](103-storage-capacity-vi.md) là "node còn gắn thêm được volume không". Rất nhiều số
liệu trong bài là hằng số của các nhà cung cấp đám mây — không cần nhớ, chỉ cần biết chúng tồn
tại và Kubernetes tôn trọng chúng.

**Phải hiểu ở lần đọc này:**

- Vì sao cần: nhà cung cấp giới hạn số volume gắn được vào một Node, và nếu Kubernetes không
  tôn trọng giới hạn đó thì **Pod được lập lịch lên node sẽ bị kẹt trong lúc chờ volume được
  gắn** — đoạn mở đầu.
- Nguồn của con số giới hạn: với plugin in-tree, Kubernetes **tự xác định loại node** và áp số
  tối đa phù hợp; với CSI, **driver tự công bố qua `NodeGetInfo`** và kube-scheduler tôn trọng
  con số đó; với plugin in-tree đã di trú sang CSI, con số lấy từ **CSI driver** — mục *Giới
  hạn volume động*.
- Giới hạn có thể đổi lúc chạy: bật feature gate `MutableCSINodeAllocatableCount` cho
  `kube-apiserver` và `kubelet`, rồi driver đặt `nodeAllocatableUpdatePeriodSeconds` trong spec
  `CSIDriver` (**tối thiểu 10 giây**) để kubelet định kỳ gọi lại `NodeGetInfo` — mục *Số lượng
  volume cấp phát được của CSI Node có thể thay đổi*.
- Đường thoát khi hết chỗ: thao tác gắn thất bại với lỗi `ResourceExhausted` (mã gRPC 8) sẽ
  **kích hoạt cập nhật ngay lập tức**, và kubelet đánh dấu các Pod bị ảnh hưởng là `Failed` để
  controller tạo lại — tránh việc Pod kẹt vô thời hạn ở `ContainerCreating` — mục *Cập nhật
  định kỳ*.
- `spec.preventPodSchedulingIfMissing` trên đối tượng `CSIDriver` (alpha, cần feature gate
  `VolumeLimitScaling`) chặn Pod lên node chưa cài driver đó, và **chỉ áp dụng cho những Pod
  cần CSI volume tương ứng** — mục *Ngăn việc đặt Pod khi không có CSI driver*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Giới hạn mặc định của Kubernetes* và các con số theo loại instance EC2, GCE, Azure | lab không chạy trên nền tảng đám mây nào | không cần |
| *Giới hạn gắn CSI volume và cluster autoscaler* | cần cluster-autoscaler, chưa có trong lộ trình đến giai đoạn này | giai đoạn 12, bài [171](171-node-autoscaling-vi.md) |

---

Trang này mô tả số lượng volume tối đa có thể được gắn (attach) vào một
Node đối với các nhà cung cấp đám mây (cloud provider) khác nhau.

Các nhà cung cấp đám mây như Google, Amazon và Microsoft thường có giới
hạn về số lượng volume có thể gắn vào một Node. Việc Kubernetes tôn
trọng các giới hạn đó là rất quan trọng. Nếu không, các Pod được lập
lịch lên một Node có thể bị kẹt trong lúc chờ các volume được gắn vào.

## Giới hạn mặc định của Kubernetes (Kubernetes default limits)

Kubernetes scheduler có các giới hạn mặc định về số lượng volume có thể
gắn vào một Node:

| Dịch vụ đám mây | Số volume tối đa trên mỗi Node |
|---|---|
| [Amazon Elastic Block Store (EBS)](https://aws.amazon.com/ebs/) | 39 |
| [Google Persistent Disk](https://cloud.google.com/persistent-disk/) | 16 |
| [Microsoft Azure Disk Storage](https://azure.microsoft.com/en-us/services/storage/main-disks/) | 16 |

## Giới hạn volume động (Dynamic volume limits)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.17 [stable]`

Giới hạn volume động được hỗ trợ cho các loại volume sau.

- Amazon EBS
- Google Persistent Disk
- Azure Disk
- CSI

Với các volume được quản lý bởi các plugin volume in-tree, Kubernetes
tự động xác định loại Node và áp dụng số lượng volume tối đa phù hợp
cho node đó. Ví dụ:

* Trên
[Google Compute Engine](https://cloud.google.com/compute/),
có thể gắn tối đa 127 volume vào một node, [tùy theo loại
node](https://cloud.google.com/compute/docs/disks/#pdnumberlimits).

* Với các đĩa Amazon EBS trên các loại instance M5, C5, R5, T3 và Z1D,
Kubernetes chỉ cho phép gắn 25 volume vào một Node. Với các loại
instance khác trên
[Amazon Elastic Compute Cloud (EC2)](https://aws.amazon.com/ec2/),
Kubernetes cho phép gắn 39 volume vào một Node.

* Trên Azure, có thể gắn tối đa 64 đĩa vào một node, tùy theo loại node. Để biết thêm chi tiết, tham khảo [Sizes for virtual machines in Azure](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/sizes).

* Nếu một CSI storage driver công bố số lượng volume tối đa cho một Node (thông qua `NodeGetInfo`), kube-scheduler sẽ tôn trọng giới hạn đó.
Tham khảo [đặc tả CSI](https://github.com/container-storage-interface/spec/blob/master/spec.md#nodegetinfo) để biết chi tiết.

* Với các volume được quản lý bởi các plugin in-tree đã được di trú (migrate) sang một CSI driver, số lượng volume tối đa sẽ là con số do CSI driver báo cáo.

### Số lượng volume cấp phát được của CSI Node có thể thay đổi (Mutable CSI Node Allocatable Count)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Các CSI driver có thể điều chỉnh động số lượng volume tối đa có thể gắn
vào một Node ngay trong lúc chạy (runtime). Điều này nâng cao độ chính
xác khi lập lịch và giảm số lần lập lịch Pod thất bại do thay đổi về
mức độ sẵn có của tài nguyên.

Để dùng tính năng này, bạn phải bật feature gate
`MutableCSINodeAllocatableCount` trên các thành phần sau:

- `kube-apiserver`
- `kubelet`

#### Cập nhật định kỳ (Periodic Updates)

Khi được bật, các CSI driver có thể yêu cầu cập nhật định kỳ giới hạn
volume của chúng bằng cách đặt trường `nodeAllocatableUpdatePeriodSeconds`
trong đặc tả `CSIDriver`. Ví dụ:

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: hostpath.csi.k8s.io
spec:
  nodeAllocatableUpdatePeriodSeconds: 60
```

Kubelet sẽ định kỳ gọi endpoint `NodeGetInfo` của CSI driver tương ứng
để làm mới số lượng volume tối đa có thể gắn, theo chu kỳ được chỉ định
trong `nodeAllocatableUpdatePeriodSeconds`. Giá trị tối thiểu cho phép
của trường này là 10 giây.

Nếu một thao tác gắn volume thất bại với lỗi `ResourceExhausted` (mã
gRPC 8), Kubernetes kích hoạt cập nhật ngay lập tức số lượng volume cấp
phát được cho Node đó. Ngoài ra, kubelet đánh dấu các Pod bị ảnh hưởng
là Failed, cho phép các controller của chúng xử lý việc tạo lại. Điều
này ngăn các Pod bị kẹt vô thời hạn ở trạng thái `ContainerCreating`.

### Ngăn việc đặt Pod khi không có CSI driver (Preventing Pod placement without CSI driver)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [alpha]`

Nếu [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates#VolumeLimitScaling)
`VolumeLimitScaling` được bật và một CSI driver có đối tượng `CSIDriver`
tương ứng được cài đặt với `spec.preventPodSchedulingIfMissing` đặt là
true, thì scheduler sẽ ngăn việc đặt Pod lên các node chưa được cài đặt
CSI driver đó. Ví dụ:

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: hostpath.csi.k8s.io
spec:
  preventPodSchedulingIfMissing: true
```

Hạn chế này chỉ áp dụng cho các Pod cần CSI volume tương ứng.

### Giới hạn gắn CSI volume và cluster autoscaler (CSI volume attach limits and cluster autoscaler)

Nếu tùy chọn `--enable-csi-node-aware-scheduling` được bật trong
cluster-autoscaler, thì cluster-autoscaler có thể tính toán chính xác
số lượng node cần thiết để đáp ứng các Pod đang chờ (pending) cần CSI
volume.

Nếu bạn đang dùng cluster-autoscaler trong cluster Kubernetes của mình,
chúng tôi không khuyến nghị ngăn việc đặt Pod thông qua trường
`PreventPodSchedulingIfMissing`, trừ khi cluster-autoscaler cũng được
bật tùy chọn dòng lệnh `--enable-csi-node-aware-scheduling`. Lý do sâu
xa của hạn chế này trong khi tính năng `VolumeLimitScaling` vẫn ở giai
đoạn alpha là: việc ngăn đặt Pod có thể phá vỡ quá trình mô phỏng lập
lịch mà cluster-autoscaler thực hiện nếu cluster-autoscaler chưa nhận
biết được các giới hạn CSI volume. Chúng tôi kỳ vọng hạn chế này sẽ
biến mất khi `--enable-csi-node-aware-scheduling` được bật mặc định
trong cluster-autoscaler.

Tùy chọn dòng lệnh `--enable-csi-node-aware-scheduling` trong
cluster-autoscaler có thể được bật bất kể trạng thái của tính năng
`VolumeLimitScaling` trong Kubernetes. Chúng tôi khuyến nghị bật nó nếu
cluster của bạn đang dùng CSI volume và bạn gặp các vấn đề liên quan
đến việc quá nhiều Pod dồn vào một node khi một node mới được
cluster-autoscaler tạo ra, vì phiên bản hiện tại của cluster-autoscaler
không tính đúng số lượng node cần thiết để đáp ứng tất cả các Pod đang
chờ.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Hai worker của bạn chạy trên máy ảo, không có nhà cung cấp đám mây nào. Sau khi Lab 6a cài
   một CSI driver, ai quyết định số volume tối đa gắn được vào một node và Kubernetes biết con
   số đó bằng cách nào?
2. Một plugin in-tree đã được di trú sang CSI driver. Số volume tối đa lúc này lấy từ bảng mặc
   định của Kubernetes hay từ đâu?
3. Một Pod đang kẹt ở `ContainerCreating` vì node đã hết chỗ gắn volume. Với
   `MutableCSINodeAllocatableCount` được bật, Kubernetes phải chờ hết chu kỳ
   `nodeAllocatableUpdatePeriodSeconds` hay có đường khác?
4. Bật `preventPodSchedulingIfMissing: true` cho một CSIDriver có chặn **mọi** Pod lên các node
   chưa cài driver đó không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Chính CSI driver quyết định.** Nếu một CSI storage driver công bố số lượng volume tối đa
   cho một Node **thông qua `NodeGetInfo`**, kube-scheduler tôn trọng giới hạn đó. Không có
   cloud provider nào ở đây, nên các con số mặc định trong bảng đầu bài (EBS, GCE PD, Azure
   Disk) hoàn toàn không liên quan tới cluster của bạn.
2. **Từ CSI driver.** Bài nói rõ: với các volume được quản lý bởi plugin in-tree đã di trú sang
   một CSI driver, số volume tối đa sẽ là **con số do CSI driver báo cáo**, không phải con số
   Kubernetes tự suy theo loại node như với plugin in-tree chưa di trú.
3. **Không phải chờ.** Đây là chỗ dễ đoán sai: cập nhật định kỳ chỉ là một trong hai đường. Khi
   một thao tác gắn volume thất bại với lỗi **`ResourceExhausted` (mã gRPC 8)**, Kubernetes
   **kích hoạt cập nhật ngay lập tức** số lượng volume cấp phát được cho Node đó; đồng thời
   kubelet **đánh dấu các Pod bị ảnh hưởng là `Failed`** để controller của chúng xử lý việc tạo
   lại. Chính cơ chế này ngăn Pod kẹt vô thời hạn ở `ContainerCreating`. Nhân tiện, giá trị tối
   thiểu cho phép của `nodeAllocatableUpdatePeriodSeconds` là **10 giây**.
4. **Không.** Hạn chế này **chỉ áp dụng cho các Pod cần CSI volume tương ứng**. Pod không đụng
   tới volume của driver đó vẫn lập lịch bình thường lên node thiếu driver. Cũng nhớ đây là
   tính năng alpha và cần feature gate `VolumeLimitScaling`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
