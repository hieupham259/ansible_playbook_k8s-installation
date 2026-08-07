# Quản lý Workload (Workload Management)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/controllers/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 4](LO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài 1/14 ·
Kiểm chứng ở Lab 4 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là **trang mục lục** của cả giai đoạn 4, không phải bài học. Nhiệm vụ của nó là cho bạn
biết có bao nhiêu loại controller và mỗi loại sinh ra để giải bài toán nào. Đọc mất khoảng
mười phút; mọi chi tiết cơ chế đều nằm ở các bài sau.

**Phải hiểu ở lần đọc này:**

- Vì sao có tầng workload object: bạn khai báo một object **cao hơn Pod**, rồi control plane
  tự tạo và xóa Pod thay bạn dựa trên spec đó — thay vì bạn tự trông từng Pod một.
- Deployment (và gián tiếp là ReplicaSet) dành cho workload **stateless**, nơi mọi Pod hoán
  đổi được cho nhau và thay thế Pod nào cũng như nhau.
- StatefulSet dành cho các Pod **dựa vào định danh riêng biệt** — đây đúng là giả định ngược
  với Deployment; dùng nhiều nhất để gắn mỗi Pod với một khối lưu trữ bền vững của riêng nó.
- DaemonSet dành cho tiện ích **cấp node** (driver lưu trữ, plugin mạng), chạy trên mọi node
  hoặc chỉ một tập con node.
- Job và CronJob dành cho tác vụ **chạy đến khi hoàn tất rồi dừng**: Job là tác vụ một lần,
  CronJob lặp lại theo lịch.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Đoạn StatefulSet — liên kết Pod với PersistentVolume | chưa học lưu trữ | giai đoạn 6 |
| Đoạn DaemonSet — plugin cho phép node truy cập mạng của cluster | chưa học mạng và CNI | giai đoạn 5 |
| Mục *Các chủ đề khác* — dọn dẹp Job và ReplicationController | chỉ là hai dòng mục lục | bài [68](68-ttlafterfinished-vi.md) và [70](70-replicationcontroller-vi.md), cuối giai đoạn 4 |

---

Kubernetes cung cấp một số API tích hợp sẵn để quản lý các workload
và các thành phần của những workload đó theo cách khai báo (declarative).

Xét cho cùng, ứng dụng của bạn chạy dưới dạng các container bên trong Pod;
tuy nhiên, việc quản lý từng Pod riêng lẻ sẽ tốn rất nhiều công sức. Ví dụ, nếu một Pod
thất bại, có lẽ bạn muốn chạy một Pod mới để thay thế nó. Kubernetes có thể làm điều đó cho bạn.

Bạn dùng Kubernetes API để tạo một đối tượng (object) workload đại diện cho một mức trừu tượng
cao hơn so với Pod, sau đó control plane của Kubernetes tự động quản lý các đối tượng Pod
thay cho bạn, dựa trên đặc tả (specification) của đối tượng workload mà bạn đã định nghĩa.

Các API tích hợp sẵn để quản lý workload gồm:

[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) (và gián tiếp là [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)),
cách phổ biến nhất để chạy một ứng dụng trên cluster của bạn.
Deployment phù hợp để quản lý một workload ứng dụng phi trạng thái (stateless) trên cluster của bạn,
trong đó mọi Pod thuộc Deployment đều có thể hoán đổi cho nhau và có thể được thay thế khi cần.
(Deployment là sự thay thế cho API ReplicationController cũ).

[StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) cho phép bạn
quản lý một hoặc nhiều Pod — tất cả chạy cùng một mã ứng dụng — trong đó các Pod dựa vào việc
có một định danh (identity) riêng biệt. Điều này khác với Deployment, nơi các Pod được kỳ vọng
là có thể hoán đổi cho nhau.
Cách dùng phổ biến nhất của StatefulSet là để có thể tạo liên kết giữa các Pod của nó và
bộ lưu trữ bền vững (persistent storage) của chúng. Ví dụ, bạn có thể chạy một StatefulSet
gắn mỗi Pod với một [PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).
Nếu một trong các Pod của StatefulSet thất bại, Kubernetes tạo một Pod thay thế được kết nối
tới cùng PersistentVolume đó.

[DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) định nghĩa các Pod
cung cấp những tiện ích cục bộ cho một node cụ thể;
ví dụ, một driver cho phép các container trên node đó truy cập một hệ thống lưu trữ. Bạn dùng DaemonSet
khi driver đó, hoặc một dịch vụ cấp node khác, phải chạy trên node mà nó hữu ích.
Mỗi Pod trong một DaemonSet đóng vai trò tương tự một system daemon trên máy chủ Unix / POSIX
cổ điển.
Một DaemonSet có thể là thành phần nền tảng cho hoạt động của cluster của bạn,
chẳng hạn một plugin cho phép node đó truy cập
[mạng của cluster (cluster networking)](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-network-model),
nó có thể giúp bạn quản lý node,
hoặc có thể cung cấp những tiện ích ít thiết yếu hơn giúp nâng cao nền tảng container mà bạn đang vận hành.
Bạn có thể chạy DaemonSet (và các pod của chúng) trên mọi node trong cluster, hoặc chỉ trên một
tập con các node (ví dụ, chỉ cài driver tăng tốc GPU trên những node có gắn GPU).

Bạn có thể dùng [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) và / hoặc
[CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) để
định nghĩa các tác vụ chạy đến khi hoàn tất rồi dừng lại. Một Job đại diện cho một tác vụ chạy
một lần, trong khi mỗi CronJob lặp lại theo lịch.

Các chủ đề khác trong mục này:

- [Tự động dọn dẹp các Job đã hoàn tất (Automatic Cleanup for Finished Jobs)](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
- [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4:

1. Bài phân biệt Deployment với StatefulSet bằng đúng một giả định về các Pod. Giả định đó
   là gì, và vì sao nó khiến bạn không thể thay StatefulSet bằng Deployment?
2. Trên cluster lab của bạn (1 control plane `k8s-master` + 2 worker), Flannel cần có mặt
   trên **mọi** node. Theo cách bài mô tả các loại controller, Flannel thuộc loại nào, và vì
   sao Deployment không làm được việc đó?
3. Job và CronJob khác nhau ở điểm nào? Điểm chung nào tách cả hai ra khỏi Deployment,
   StatefulSet và DaemonSet?
4. Bài nói bạn được lợi gì khi khai báo một workload object thay vì tự tạo từng Pod?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Giả định là **Pod có hoán đổi được cho nhau hay không**. Deployment giả định **mọi Pod
   hoán đổi cho nhau và có thể được thay thế khi cần** — mất Pod nào cũng như nhau.
   StatefulSet dành cho các Pod **dựa vào việc có một định danh riêng biệt**, nên một Pod
   không thể thay bằng một Pod bất kỳ khác. Bài nói thêm: cách dùng phổ biến nhất của
   StatefulSet là tạo liên kết giữa mỗi Pod và bộ lưu trữ bền vững của riêng nó, và khi một
   Pod hỏng thì Pod thay thế được nối tới **đúng PersistentVolume cũ**.
2. **DaemonSet.** Bài nói rõ DaemonSet định nghĩa các Pod cung cấp tiện ích cục bộ cho một
   node cụ thể, và nêu đúng ví dụ này: "một plugin cho phép node đó truy cập mạng của
   cluster". Deployment chỉ bảo đảm **tổng số** replica, không bảo đảm **vị trí** — ba
   replica của một Deployment hoàn toàn có thể cùng nằm trên một worker, để node còn lại
   không có Pod mạng nào.
3. Khác nhau: **Job là tác vụ chạy một lần, CronJob lặp lại theo lịch**. Điểm chung tách
   chúng khỏi ba loại kia: chúng định nghĩa các tác vụ **chạy đến khi hoàn tất rồi dừng
   lại**, trong khi Deployment, StatefulSet và DaemonSet quản lý các Pod được kỳ vọng chạy
   mãi.
4. **Việc quản lý từng Pod riêng lẻ tốn rất nhiều công sức.** Bài lấy ví dụ trực tiếp: nếu
   một Pod thất bại, bạn muốn có Pod mới thay thế — và Kubernetes làm việc đó cho bạn. Bạn
   tạo một object đại diện cho **mức trừu tượng cao hơn Pod**, rồi control plane tự quản lý
   các object Pod thay bạn dựa trên spec bạn đã định nghĩa.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
