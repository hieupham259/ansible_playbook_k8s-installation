# Thay đổi StorageClass mặc định (Change the default StorageClass)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/>
>
> Trang này hướng dẫn cách thay đổi Storage Class mặc định được dùng để cấp phát (provision)
> volume cho những PersistentVolumeClaim không có yêu cầu đặc biệt.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 26 — Vận hành lưu trữ](00-ALO-TRINH-ADMIN.md#giai-đoạn-26--vận-hành-lưu-trữ),
bài 1/4 · Giai đoạn này của Phần II không có lab riêng: thực hành ngay trên cluster lab, sau khi đã có
StorageClass và dynamic provisioner từ [Lab 6a](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md).

Đây là trang task dạng runbook, rất ngắn. Lý thuyết về default StorageClass bạn đã đọc ở
[bài 96](96-storage-classes-vi.md); trang này chỉ bổ sung thao tác `kubectl patch` và các bẫy
khi đổi mặc định.

**Phải hiểu ở lần đọc này:**

- Default StorageClass được nhận diện bằng annotation
  `storageclass.kubernetes.io/is-default-class` đặt là `true`; **mọi giá trị khác hoặc thiếu
  annotation đều được hiểu là `false`**, và cột NAME của `kubectl get storageclass` hiện thêm
  `(default)`.
- Đổi mặc định là hai lệnh `kubectl patch` annotation: đặt `false` cho class cũ, đặt `true` cho
  class mới — không cần tạo lại StorageClass.
- Có thể tồn tại **nhiều StorageClass cùng được đánh dấu mặc định**; khi đó PVC không khai báo
  `storageClassName` sẽ dùng default StorageClass **được tạo gần đây nhất**.
- PVC chỉ định sẵn `volumeName` sẽ **treo ở pending** nếu `storageClassName` của volume tĩnh
  không khớp với `StorageClass` trên PVC.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Đoạn về addon manager tự tạo lại default StorageClass khi bị xóa | chỉ gặp trên cluster dựng sẵn của nhà cung cấp; cluster kubeadm của lab không có addon manager kiểu này | không cần |

---

Trang này hướng dẫn cách thay đổi Storage Class mặc định được dùng để cấp phát volume cho các
PersistentVolumeClaim không có yêu cầu đặc biệt.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

## Vì sao phải thay đổi storage class mặc định? (Why change the default storage class?)

Tùy vào phương thức cài đặt, cluster Kubernetes của bạn có thể được triển khai sẵn với một
StorageClass được đánh dấu là mặc định. StorageClass mặc định này sau đó được dùng để cấp phát
động (dynamically provision) lưu trữ cho những PersistentVolumeClaim không yêu cầu một storage
class cụ thể nào. Xem
[tài liệu về PersistentVolumeClaim](92-persistent-volumes-vi.md#persistentvolumeclaims)
để biết chi tiết.

StorageClass mặc định được cài sẵn có thể không phù hợp với workload mà bạn dự kiến; ví dụ, nó
có thể cấp phát loại lưu trữ quá đắt. Nếu rơi vào trường hợp đó, bạn có thể đổi StorageClass
mặc định hoặc vô hiệu hóa nó hoàn toàn để tránh việc cấp phát lưu trữ động.

Việc xóa StorageClass mặc định có thể không có tác dụng, vì nó có thể được trình quản lý addon
(addon manager) chạy trong cluster của bạn tự động tạo lại. Hãy tham khảo tài liệu của bản cài
đặt bạn đang dùng để biết chi tiết về addon manager và cách tắt từng addon.

## Thay đổi StorageClass mặc định (Changing the default StorageClass)

1. Liệt kê các StorageClass trong cluster của bạn:

   ```bash
   kubectl get storageclass
   ```

   Kết quả tương tự như sau:

   ```bash
   NAME                 PROVISIONER               AGE
   standard (default)   kubernetes.io/gce-pd      1d
   gold                 kubernetes.io/gce-pd      1d
   ```

   StorageClass mặc định được đánh dấu bằng `(default)`.

1. Đánh dấu StorageClass mặc định thành không-mặc-định:

   StorageClass mặc định có một annotation `storageclass.kubernetes.io/is-default-class` được
   đặt là `true`. Bất kỳ giá trị nào khác, hoặc việc thiếu annotation này, đều được hiểu là
   `false`.

   Để đánh dấu một StorageClass là không-mặc-định, bạn cần đổi giá trị của annotation đó thành
   `false`:

   ```bash
   kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
   ```

   trong đó `standard` là tên StorageClass mà bạn chọn.

1. Đánh dấu một StorageClass là mặc định:

   Tương tự bước trước, bạn cần thêm/đặt annotation
   `storageclass.kubernetes.io/is-default-class=true`.

   ```bash
   kubectl patch storageclass gold -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
   ```

   Lưu ý rằng bạn có thể có nhiều `StorageClass` cùng được đánh dấu là mặc định. Nếu có nhiều
   hơn một `StorageClass` được đánh dấu mặc định, một `PersistentVolumeClaim` không khai báo
   `storageClassName` tường minh sẽ được tạo bằng `StorageClass` mặc định được tạo gần đây
   nhất. Khi một `PersistentVolumeClaim` được tạo với `volumeName` chỉ định sẵn, nó sẽ nằm ở
   trạng thái pending nếu `storageClassName` của volume tĩnh đó không khớp với `StorageClass`
   trên `PersistentVolumeClaim`.

1. Kiểm tra lại rằng StorageClass bạn chọn đã là mặc định:

   ```bash
   kubectl get storageclass
   ```

   Kết quả tương tự như sau:

   ```bash
   NAME             PROVISIONER               AGE
   standard         kubernetes.io/gce-pd      1d
   gold (default)   kubernetes.io/gce-pd      1d
   ```

## Tiếp theo (What's next)

* Tìm hiểu thêm về [PersistentVolume](92-persistent-volumes-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở checkpoint giai đoạn 26:

1. Trên cluster lab của bạn, bạn patch annotation `is-default-class=true` cho một StorageClass
   thứ hai nhưng quên đặt `false` cho class cũ. PVC mới không khai báo `storageClassName` sẽ
   được tạo bằng class nào?
2. Xóa hẳn default StorageClass có phải là cách chắc chắn để tắt cấp phát lưu trữ động không?
   Vì sao trang này khuyên cách khác?
3. Một StorageClass mang annotation `storageclass.kubernetes.io/is-default-class: "false"` khác
   gì với một StorageClass hoàn toàn không có annotation đó?
4. Bạn tạo một PVC có `volumeName` trỏ tới một PV tĩnh, nhưng `storageClassName` của PV đó
   không khớp với `StorageClass` trên PVC. Chuyện gì xảy ra?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **StorageClass mặc định được tạo gần đây nhất.** Bài nói rõ: được phép có nhiều
   `StorageClass` mặc định cùng lúc, và khi đó PVC không khai báo `storageClassName` sẽ dùng
   default **mới được tạo gần nhất** — không phải class cũ, cũng không phải lỗi. Đây là chỗ dễ
   nhầm: cluster không từ chối trạng thái "hai default", nó chỉ chọn theo thời điểm tạo.
2. **Không chắc chắn.** Trên các cluster có addon manager, default StorageClass bị xóa **có thể
   được tự động tạo lại**. Cách trang này hướng dẫn là **bỏ đánh dấu mặc định** bằng cách đặt
   annotation `is-default-class` về `false` — trạng thái đó không bị addon manager "sửa lại",
   và vẫn giữ được StorageClass cho ai cần dùng tường minh.
3. **Không khác gì nhau về hiệu lực.** Quy tắc của bài: chỉ giá trị `true` mới làm một
   StorageClass thành mặc định; **mọi giá trị khác hoặc việc thiếu annotation đều được hiểu là
   `false`**.
4. **PVC nằm ở trạng thái pending.** Khi `volumeName` được chỉ định sẵn, việc bind chỉ xảy ra
   nếu `storageClassName` của volume tĩnh khớp với `StorageClass` trên PVC; không khớp thì PVC
   treo pending chứ không tự rơi về default StorageClass.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
