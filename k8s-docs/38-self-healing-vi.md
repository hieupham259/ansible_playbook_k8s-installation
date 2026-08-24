# Khả năng tự phục hồi của Kubernetes (Kubernetes Self-Healing)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/architecture/self-healing/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài 9/14 ·
Kiểm chứng ở Lab 4 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Trên kubernetes.io bài này nằm trong mục kiến trúc, tức là chỗ của **giai đoạn 1**. Lộ trình
cố ý **chuyển nó xuống đây** vì mọi gạch đầu dòng của nó đều là kết luận rút ra từ
[Deployment](63-deployment-vi.md), [ReplicaSet](64-replicaset-vi.md),
[StatefulSet](65-statefulset-vi.md) và [DaemonSet](66-daemonset-vi.md) — bốn bài bạn vừa
đọc. Đọc nó ở giai đoạn 1 thì chỉ học thuộc được bốn dòng; đọc ở đây thì nó là bài **tổng
kết**: mỗi dòng phải gọi tên được cơ chế đứng sau. Bài rất ngắn, hãy đọc chậm.

**Phải hiểu ở lần đọc này:**

- Bốn khả năng trong mục *Các khả năng tự phục hồi* nằm ở **bốn tầng khác nhau** và do **các
  thành phần khác nhau** lo: container, replica, lưu trữ, endpoint của Service.
- Ranh giới kubelet ↔ controller: **kubelet** bảo đảm container đang chạy và khởi động lại
  container hỏng **bên trong** Pod theo `restartPolicy`; **controller** tạo Pod **mới** khi
  cả Pod mất. Đây đúng là ranh giới bạn đã gặp ở Job và ReplicaSet.
- DaemonSet là ngoại lệ về **vị trí**: bài nói khi một Pod của DaemonSet lỗi, control plane
  tạo Pod thay thế để chạy **trên chính node đó**, không phải một node bất kỳ.
- Giới hạn ở mục *Những điểm cần cân nhắc*: Kubernetes khởi động lại được container nhưng
  **các vấn đề nằm ở bản thân ứng dụng phải được xử lý riêng**; persistent volume hỏng vẫn
  cần các bước khôi phục.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Khôi phục lưu trữ bền vững* và PersistentVolume controller | chưa học PV và PVC | giai đoạn 6 |
| *Cân bằng tải cho Service* và việc loại Pod khỏi endpoint | chưa học Service và EndpointSlice | giai đoạn 5 |
| Gạch cuối mục *Tiếp theo* — tự động co giãn node | thuộc vận hành vòng đời node | giai đoạn 12 |

---

Kubernetes được thiết kế với các khả năng tự phục hồi (self-healing) giúp duy trì sức
khỏe và tính khả dụng của các workload. Nó tự động thay thế các container bị lỗi, lên
lịch lại (reschedule) các workload khi node trở nên không khả dụng, và đảm bảo trạng
thái mong muốn của hệ thống luôn được duy trì.

## Các khả năng tự phục hồi (Self-Healing capabilities) {#self-healing-capabilities}

- **Khởi động lại ở cấp container (Container-level restarts):** Nếu một container bên
  trong Pod bị lỗi, Kubernetes sẽ khởi động lại nó dựa trên
  [`restartPolicy`](47-pod-lifecycle-vi.md#restart-policy).

- **Thay thế replica (Replica replacement):** Nếu một Pod trong
  [Deployment](63-deployment-vi.md) hoặc
  [StatefulSet](65-statefulset-vi.md)
  bị lỗi, Kubernetes sẽ tạo một Pod thay thế để duy trì số lượng replica đã chỉ định.
  Nếu một Pod thuộc [DaemonSet](66-daemonset-vi.md)
  bị lỗi, control plane sẽ tạo một Pod thay thế để chạy trên chính node đó.

- **Khôi phục lưu trữ bền vững (Persistent storage recovery):** Nếu một node đang chạy
  Pod có gắn PersistentVolume (PV) và node đó gặp sự cố, Kubernetes có thể gắn lại
  (reattach) volume vào một Pod mới trên một node khác.

- **Cân bằng tải cho Service (Load balancing for Services):** Nếu một Pod đứng sau một
  [Service](82-service-vi.md) bị lỗi,
  Kubernetes sẽ tự động loại bỏ nó khỏi các endpoint của Service để chỉ định tuyến lưu
  lượng (traffic) tới các Pod khỏe mạnh.

Dưới đây là một số thành phần chính cung cấp khả năng tự phục hồi của Kubernetes:

- **[kubelet](./22-architecture-vi.md#kubelet):** Đảm bảo các container đang chạy, và
  khởi động lại những container bị lỗi.

- **Các controller Deployment (thông qua ReplicaSet), ReplicaSet, StatefulSet và
  DaemonSet:** Duy trì số lượng Pod replica mong muốn.

- **PersistentVolume controller:** Quản lý việc gắn (attach) và tháo (detach) volume
  cho các workload có trạng thái (stateful).

## Những điểm cần cân nhắc (Considerations) {#considerations}

- **Lỗi lưu trữ (Storage Failures):** Nếu một persistent volume trở nên không khả dụng,
  có thể cần thực hiện các bước khôi phục.

- **Lỗi ứng dụng (Application Errors):** Kubernetes có thể khởi động lại container,
  nhưng các vấn đề nằm ở bản thân ứng dụng phải được xử lý riêng.

## Tiếp theo (What's next)

- Đọc thêm về [Pod](46-pods-vi.md)
- Tìm hiểu về [các Controller của Kubernetes](./25-controllers-vi.md)
- Khám phá [PersistentVolume](92-persistent-volumes-vi.md)
- Đọc về [tự động co giãn node (node autoscaling)](171-node-autoscaling-vi.md).
  Tự động co giãn node cũng cung cấp khả năng tự phục hồi nếu hoặc khi các node trong
  cluster của bạn gặp sự cố.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4:

1. Một container bên trong Pod bị lỗi. Ai khởi động lại nó, và theo trường nào? Nếu cả Pod
   biến mất thì ai lo, và nó làm gì khác đi?
2. **Câu bẫy.** Bạn xóa một Pod của DaemonSet đang chạy trên `k8s-worker2`. Pod thay thế có
   thể được lập lịch sang `k8s-worker1` không? Còn nếu đó là Pod của một Deployment 3 replica
   thì sao?
3. Bạn triển khai một ứng dụng có bug khiến nó thoát ngay sau khi khởi động. Kubernetes có
   "tự phục hồi" được không? Bài đặt ranh giới ở đâu?
4. Bài liệt kê Deployment trong danh sách các thành phần cung cấp khả năng tự phục hồi bằng
   cụm từ nào? Vì sao cách diễn đạt đó quan trọng sau khi bạn đã đọc bài
   [64](64-replicaset-vi.md) và [63](63-deployment-vi.md)?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **kubelet** khởi động lại container, dựa trên **`restartPolicy`** của Pod — bài xếp kubelet
   đầu tiên trong danh sách các thành phần, với vai trò "đảm bảo các container đang chạy, và
   khởi động lại những container bị lỗi". Nếu cả Pod mất thì việc đó thuộc về **các controller
   Deployment (thông qua ReplicaSet), ReplicaSet, StatefulSet và DaemonSet**, và chúng làm
   việc khác hẳn: chúng không sửa Pod cũ mà **tạo một Pod thay thế** để duy trì số lượng
   replica đã chỉ định.
2. **Với DaemonSet: không** — bài nói rõ "nếu một Pod thuộc DaemonSet bị lỗi, control plane sẽ
   tạo một Pod thay thế để chạy **trên chính node đó**". **Với Deployment: có** — Deployment
   chỉ cam kết giữ **số lượng** replica, không cam kết vị trí, nên Pod thay thế có thể rơi
   xuống bất kỳ worker nào. Trực giác "tự phục hồi nghĩa là dựng lại đúng chỗ cũ" chỉ đúng với
   DaemonSet, và đúng vì bản chất DaemonSet là tiện ích cấp node — thay thế sang node khác thì
   node cũ mất dịch vụ.
3. **Không.** Đây chính là mục *Những điểm cần cân nhắc*: "Kubernetes có thể khởi động lại
   container, nhưng các vấn đề nằm ở **bản thân ứng dụng** phải được xử lý riêng." Kubernetes
   sẽ khởi động lại container theo `restartPolicy` mãi mãi và bạn được một vòng lặp crash chứ
   không được một ứng dụng khỏe. Ranh giới tương tự áp dụng cho lưu trữ: nếu một persistent
   volume trở nên không khả dụng, có thể cần thực hiện các bước khôi phục thủ công.
4. Bằng cụm **"Các controller Deployment (thông qua ReplicaSet), ReplicaSet, StatefulSet và
   DaemonSet"**. Ba chữ **"thông qua ReplicaSet"** là điểm mấu chốt: Deployment không tự đếm
   và tạo Pod: nó điều khiển ReplicaSet, và chính ReplicaSet mới là thứ giữ số Pod khớp
   `.spec.replicas` và tạo lại Pod bị xóa. Sau bài 64 và 63 bạn đọc được câu đó như một mô tả
   cơ chế, chứ không phải một cách liệt kê cho đủ tên.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
