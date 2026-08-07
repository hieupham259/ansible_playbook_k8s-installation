# Workload (Workloads)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/>
>
> Tìm hiểu về Pod — đối tượng tính toán nhỏ nhất có thể triển khai trong Kubernetes —
> và các tầng trừu tượng cấp cao hơn giúp bạn chạy chúng.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3a](LO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài 1/11 · Kiểm chứng
ở Lab 3a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài mở nhóm, rất ngắn và chỉ là **bản đồ**. Nó liệt kê các tài nguyên workload mà bạn sẽ học
chi tiết ở giai đoạn 4. Ở lần đọc này chỉ cần nắm vai trò của từng loại và lý do vì sao bạn
không nên quản Pod bằng tay.

**Phải hiểu ở lần đọc này:**

- Workload là ứng dụng chạy trên Kubernetes, và dù nó gồm bao nhiêu thành phần thì nó luôn chạy
  **bên trong một tập Pod**. Pod là thứ duy nhất thực sự chạy.
- Lỗi node là **chung cuộc** đối với Pod trên node đó: Kubernetes không hồi sinh Pod cũ, bạn phải
  có Pod mới, kể cả khi node sau đó khỏe lại.
- Vì vậy bạn dùng **tài nguyên workload**: chúng cấu hình controller giữ đúng số lượng Pod đúng
  loại đang chạy, khớp với trạng thái bạn đã chỉ định.
- Phân vai bốn nhóm tài nguyên có sẵn: Deployment/ReplicaSet cho ứng dụng phi trạng thái (Pod
  hoán đổi được cho nhau), StatefulSet cho workload có theo dõi trạng thái, DaemonSet cho mô hình
  mỗi node một Pod, Job/CronJob cho tác vụ chạy đến khi hoàn tất — một lần hoặc theo lịch.
- Khi lõi không có hành vi bạn cần, hệ sinh thái mở rộng bằng custom resource definition.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết từng controller trong danh sách | ở đây chỉ là bản đồ; mỗi loại có bài riêng | giai đoạn 4 |
| `PersistentVolume` trong mô tả StatefulSet | chưa học lưu trữ | giai đoạn 6 |
| *Sắp đặt workload*: Workload API, `PodGroupTemplates`, `spec.schedulingGroup`, gang scheduling | tính năng alpha, cần scheduler nâng cao | giai đoạn 13 — bài [77](77-workload-api-vi.md) và [150](150-gang-scheduling-vi.md) |
| Custom resource definition | thuộc phần mở rộng Kubernetes | giai đoạn 14 — bài [179](179-custom-resources-vi.md) |
| Mục *Tiếp theo* nhắc Service và Ingress | chưa học mạng | giai đoạn 5 |

---

Workload là một ứng dụng chạy trên Kubernetes.
Dù workload của bạn là một thành phần đơn lẻ hay nhiều thành phần phối hợp với nhau,
trên Kubernetes bạn chạy nó bên trong một tập các [_Pod_](./46-pods-vi.md).
Trong Kubernetes, một Pod đại diện cho một tập gồm một hoặc nhiều container
đang chạy trên cluster của bạn.

Các Pod trong Kubernetes có [vòng đời được định nghĩa rõ ràng](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/).
Ví dụ, khi một Pod đang chạy trong cluster của bạn thì một lỗi nghiêm trọng trên
node nơi Pod đó đang chạy có nghĩa là tất cả các Pod trên node đó đều thất bại.
Kubernetes coi mức độ thất bại này là chung cuộc: bạn sẽ cần tạo một Pod mới
để khôi phục, kể cả khi node đó sau này khỏe mạnh trở lại.

Tuy nhiên, để mọi việc dễ dàng hơn đáng kể, bạn không cần quản lý trực tiếp từng Pod.
Thay vào đó, bạn có thể dùng các _tài nguyên workload_ (workload resources) quản lý
một tập Pod thay cho bạn. Các tài nguyên này cấu hình các controller
đảm bảo đúng số lượng Pod thuộc đúng loại đang chạy, khớp với trạng thái
mà bạn đã chỉ định.

Kubernetes cung cấp sẵn một số tài nguyên workload:

* [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) và [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
  (thay thế tài nguyên cũ ReplicationController).
  Deployment phù hợp để quản lý một workload ứng dụng phi trạng thái (stateless) trên cluster của bạn,
  trong đó mọi Pod thuộc Deployment đều có thể hoán đổi cho nhau và có thể được thay thế khi cần.
* [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) cho phép bạn
  chạy một hoặc nhiều Pod liên quan có theo dõi trạng thái theo cách nào đó. Ví dụ, nếu workload
  của bạn ghi dữ liệu lâu dài, bạn có thể chạy một StatefulSet ánh xạ mỗi Pod với một
  [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). Mã của bạn, chạy trong các
  Pod của StatefulSet đó, có thể sao chép (replicate) dữ liệu sang các Pod khác trong cùng StatefulSet
  để cải thiện khả năng chống chịu tổng thể.
* [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) định nghĩa các Pod cung cấp
  những tiện ích cục bộ cho node.
  Mỗi khi bạn thêm vào cluster một node khớp với đặc tả (specification) trong một DaemonSet,
  control plane sẽ lập lịch (schedule) một Pod của DaemonSet đó lên node mới.
  Mỗi Pod trong một DaemonSet thực hiện công việc tương tự như một system daemon trên máy chủ
  Unix / POSIX cổ điển. Một DaemonSet có thể là thành phần nền tảng cho hoạt động của cluster,
  chẳng hạn một plugin để chạy [mạng của cluster (cluster networking)](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-network-model),
  nó có thể giúp bạn quản lý node,
  hoặc có thể cung cấp hành vi tùy chọn giúp nâng cao nền tảng container mà bạn đang vận hành.
* [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) và
  [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) cung cấp những cách khác nhau để
  định nghĩa các tác vụ chạy đến khi hoàn tất rồi dừng lại.
  Bạn có thể dùng [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) để
  định nghĩa một tác vụ chạy đến khi hoàn tất, chỉ một lần. Bạn có thể dùng
  [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) để chạy
  cùng một Job nhiều lần theo lịch.

Trong hệ sinh thái Kubernetes rộng hơn, bạn có thể tìm thấy các tài nguyên workload của bên thứ ba
cung cấp các hành vi bổ sung. Bằng cách dùng một
[custom resource definition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/),
bạn có thể thêm một tài nguyên workload của bên thứ ba nếu bạn muốn một hành vi cụ thể
không thuộc phần lõi của Kubernetes. Ví dụ, nếu bạn muốn chạy một nhóm Pod cho ứng dụng của mình nhưng
dừng công việc trừ khi _tất cả_ các Pod đều khả dụng (có lẽ cho một tác vụ phân tán thông lượng cao nào đó),
thì bạn có thể triển khai hoặc cài đặt một phần mở rộng (extension) có cung cấp tính năng đó.

## Sắp đặt workload (Workload placement)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [alpha]`

Trong khi các tài nguyên workload tiêu chuẩn (như Deployment và Job) quản lý vòng đời của các Pod,
bạn có thể có những yêu cầu lập lịch phức tạp, trong đó các nhóm Pod phải được đối xử như một đơn vị duy nhất.

[Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/) cho phép bạn định nghĩa các `PodGroupTemplates` để nhóm các Pod lại và áp dụng các chính sách lập lịch nâng cao cho chúng,
chẳng hạn [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).
Các controller tạo ra các đối tượng [PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/) từ các template này lúc chạy (runtime),
và các `Pod` tham chiếu tới `PodGroup` của chúng qua
trường `spec.schedulingGroup`. Điều này đặc biệt hữu ích cho các workload xử lý theo lô (batch processing)
và học máy (machine learning), nơi yêu cầu sắp đặt kiểu "tất cả hoặc không gì cả" (all-or-nothing).

## Tiếp theo (What's next)

Bên cạnh việc đọc về từng loại API (API kind) dành cho quản lý workload, bạn có thể đọc cách
thực hiện các tác vụ cụ thể:

* [Chạy một ứng dụng phi trạng thái bằng Deployment](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
* Chạy một ứng dụng có trạng thái, dưới dạng [một thực thể đơn lẻ](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/)
  hoặc dưới dạng [một tập được nhân bản](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/)
* [Chạy các tác vụ tự động với CronJob](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/)

Để tìm hiểu về các cơ chế của Kubernetes cho việc tách mã nguồn khỏi cấu hình,
hãy xem [Configuration](https://kubernetes.io/docs/concepts/configuration/).

Có hai khái niệm hỗ trợ cung cấp bối cảnh về cách Kubernetes quản lý Pod
cho các ứng dụng:
* [Garbage collection](./36-garbage-collection-vi.md) dọn dẹp các đối tượng
  khỏi cluster của bạn sau khi _tài nguyên sở hữu_ (owning resource) của chúng đã bị gỡ bỏ.
* [Controller _time-to-live after finished_](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
  gỡ bỏ các Job khi một khoảng thời gian định trước đã trôi qua kể từ lúc chúng hoàn tất.

Khi ứng dụng của bạn đã chạy, bạn có thể muốn đưa nó ra internet dưới dạng
một [Service](https://kubernetes.io/docs/concepts/services-networking/service/) hoặc, chỉ với ứng dụng web,
dùng một [Ingress](./11-ingress-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Một Pod đang chạy trên `k8s-worker2` và node đó gặp lỗi nghiêm trọng. Sau khi worker2 khỏe
   lại, chính Pod đó có chạy tiếp không? Bài dùng từ gì để mô tả mức độ của lỗi này?
2. Bạn cần mỗi node chạy đúng một Pod thu log. Dùng một Deployment với số replica bằng số node
   có tương đương DaemonSet không?
3. Tài nguyên workload cho bạn thêm điều gì so với việc tự tạo từng Pod?
4. Job và CronJob khác nhau ở đâu, theo đúng cách bài mô tả?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Bài nói Kubernetes coi mức độ thất bại này là **chung cuộc**: khi node lỗi thì mọi
   Pod trên node đó đều thất bại, và bạn **cần tạo một Pod mới** để khôi phục, kể cả khi node
   sau này khỏe mạnh trở lại.
2. **Không tương đương.** DaemonSet gắn với node, không gắn với con số: mỗi khi bạn thêm vào
   cluster một node khớp với đặc tả trong DaemonSet, control plane **lập lịch một Pod của
   DaemonSet đó lên node mới**. Một Deployment chỉ giữ đúng số replica bạn khai, không có bảo
   đảm nào về việc trải đều mỗi node một Pod, và không tự bám theo node mới thêm vào.
3. Tài nguyên workload **cấu hình các controller** đảm bảo **đúng số lượng Pod thuộc đúng loại
   đang chạy, khớp với trạng thái mà bạn đã chỉ định**. Nói cách khác nó biến việc "tạo Pod" —
   một thao tác một lần — thành một trạng thái mong muốn được duy trì liên tục.
4. **Job định nghĩa một tác vụ chạy đến khi hoàn tất, chỉ một lần; CronJob chạy cùng một Job
   nhiều lần theo lịch.** Cả hai đều thuộc nhóm "chạy đến khi hoàn tất rồi dừng lại", khác với
   Deployment/StatefulSet/DaemonSet vốn giữ Pod chạy liên tục.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
