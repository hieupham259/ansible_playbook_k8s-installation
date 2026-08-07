# Sự gián đoạn (Disruptions)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/disruptions/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3b](LO-TRINH-ADMIN.md#3b-cấu-hình-và-tài-nguyên), bài 6/7 ·
Kiểm chứng ở Lab 3b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này viết cho **hai người đọc khác nhau** — chủ sở hữu ứng dụng và quản trị viên cluster —
và câu mở đầu nói rõ điều đó. Bạn đang học vai trò thứ hai, nên phần đáng giá nhất là ví dụ
drain ba node ở giữa bài: nó cho thấy chính xác vì sao một lệnh `kubectl drain` có thể treo.
Phần *Tách biệt vai trò Chủ sở hữu Cluster và Chủ sở hữu Ứng dụng* là chuyện tổ chức, không
phải cơ chế.

Bài nhắc nhiều controller (Deployment, StatefulSet) mà bạn chưa học ở giai đoạn 4. Lúc này chỉ
cần hiểu chúng là thứ giữ cho số replica luôn đúng như khai báo.

**Phải hiểu ở lần đọc này:**

- Ranh giới **tự nguyện / không tự nguyện** và cách phân loại của bài: kernel panic, mất node
  do phân mảnh mạng, và **eviction do node cạn kiệt tài nguyên** đều là *không tự nguyện*;
  drain node, xóa Pod, sửa pod template là *tự nguyện*.
- PDB chỉ chặn được gián đoạn tự nguyện **đi qua Eviction API**. Khối *Thận trọng* nói thẳng:
  xóa deployment hoặc xóa pod **bỏ qua** PDB. Còn gián đoạn không tự nguyện thì PDB không ngăn
  được nhưng **vẫn được tính vào ngân sách**.
- Số Pod "dự kiến" của PDB lấy từ `.spec.replicas` của workload resource sở hữu Pod, tìm ra
  qua `.metadata.ownerReferences`; nhóm Pod được chọn bằng **label selector**.
- `kubectl drain` gọi Eviction API và **thử lại các yêu cầu bị từ chối** cho tới khi xong hoặc
  hết timeout — nên nó có thể bị chặn rất lâu, đúng như ví dụ `pod-d` và `pod-e` ở mục
  *Ví dụ về PodDisruptionBudget*. Điều gì quyết định tốc độ đó được liệt kê ở cuối mục ấy.
- Hai ngoại lệ dễ sai: **rolling upgrade của workload resource không bị PDB giới hạn** (việc
  đó cấu hình trong spec của chính workload), và Pod bị trục xuất qua Eviction API vẫn được
  kết thúc êm, tôn trọng `terminationGracePeriodSeconds`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Các condition gián đoạn của Pod* — `DisruptionTarget` và các `reason` | mỗi `reason` thuộc một cơ chế chưa học | giai đoạn 7a, bài [141](141-pod-priority-preemption-vi.md) và [142](142-node-pressure-eviction-vi.md) |
| Dùng condition gián đoạn trong Pod failure policy của Job | chưa học Job | giai đoạn 4, bài [67](67-job-vi.md) |
| anti-affinity và trải ứng dụng theo zone | chưa học lập lịch | giai đoạn 7a, bài [138](138-assign-pod-node-vi.md) |
| `PriorityClass` như một nguồn gây gián đoạn | chưa học độ ưu tiên | giai đoạn 7a, bài [141](141-pod-priority-preemption-vi.md) |
| Unhealthy Pod Eviction Policy (`AlwaysAllow`) | là tùy chọn khi cấu hình PDB thật | CP1 — Vòng đời node |
| Thực hành cordon / drain / uncordon | cần quy trình bảo trì node | giai đoạn 12, bài [169](169-node-shutdown-vi.md) và CP1 |
| *Tách biệt vai trò Chủ sở hữu Cluster và Chủ sở hữu Ứng dụng* | là mô hình tổ chức, không phải cơ chế | giai đoạn 9, bài [122](122-multi-tenancy-vi.md) |

---

Hướng dẫn này dành cho các chủ sở hữu ứng dụng (application owner) muốn xây dựng
các ứng dụng có tính sẵn sàng cao (high availability), và do đó cần hiểu
những loại gián đoạn (disruption) nào có thể xảy ra với Pod.

Tài liệu này cũng dành cho các quản trị viên cluster muốn thực hiện các thao tác
tự động trên cluster, như nâng cấp và tự động co giãn (autoscaling) cluster.

## Gián đoạn tự nguyện và không tự nguyện (Voluntary and involuntary disruptions) {#voluntary-and-involuntary-disruptions}

Pod không biến mất cho đến khi có ai đó (một người hoặc một controller) hủy chúng,
hoặc khi xảy ra một lỗi phần cứng hay lỗi phần mềm hệ thống không thể tránh khỏi.

Chúng ta gọi những trường hợp không thể tránh khỏi này là *gián đoạn không tự nguyện*
(involuntary disruption) đối với một ứng dụng. Ví dụ:

- lỗi phần cứng của máy vật lý đang chạy node
- quản trị viên cluster xóa nhầm VM (instance)
- lỗi của nhà cung cấp cloud hoặc hypervisor làm VM biến mất
- kernel panic
- node biến mất khỏi cluster do phân mảnh mạng cluster (network partition)
- Pod bị trục xuất (eviction) do node bị
  [cạn kiệt tài nguyên](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/).

Ngoại trừ tình huống cạn kiệt tài nguyên, tất cả các tình huống này hẳn đều
quen thuộc với hầu hết người dùng; chúng không phải là đặc thù
của Kubernetes.

Chúng ta gọi các trường hợp còn lại là *gián đoạn tự nguyện* (voluntary disruption).
Chúng bao gồm cả các hành động do chủ sở hữu ứng dụng khởi xướng lẫn các hành động
do quản trị viên cluster khởi xướng. Các hành động điển hình của chủ sở hữu ứng dụng gồm:

- xóa deployment hoặc controller khác đang quản lý pod
- cập nhật pod template của một deployment gây ra việc khởi động lại
- trực tiếp xóa một pod (ví dụ do vô tình)

Các hành động của quản trị viên cluster gồm:

- [Drain một node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/) để sửa chữa hoặc nâng cấp.
- Drain một node khỏi cluster để thu nhỏ cluster (tìm hiểu về
  [Tự động co giãn node (Node Autoscaling)](https://kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)).
- Gỡ một pod khỏi node để nhường chỗ cho thứ khác vừa vặn trên node đó.

Các hành động này có thể được thực hiện trực tiếp bởi quản trị viên cluster, bởi
cơ chế tự động hóa do quản trị viên cluster vận hành, hoặc bởi nhà cung cấp dịch vụ
lưu trữ (hosting) cluster của bạn.

Hãy hỏi quản trị viên cluster của bạn, hoặc tham khảo tài liệu của nhà cung cấp cloud
hay bản phân phối, để xác định xem có nguồn gián đoạn tự nguyện nào được bật cho
cluster của bạn hay không. Nếu không có nguồn nào được bật, bạn có thể bỏ qua việc
tạo Pod Disruption Budget.

> **Thận trọng:**
> Không phải mọi gián đoạn tự nguyện đều bị ràng buộc bởi Pod Disruption Budget. Ví dụ,
> việc xóa deployment hoặc pod sẽ bỏ qua Pod Disruption Budget.

## Đối phó với gián đoạn (Dealing with disruptions) {#dealing-with-disruptions}

Dưới đây là một số cách để giảm thiểu gián đoạn không tự nguyện:

- Đảm bảo pod của bạn [yêu cầu (request) các tài nguyên](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource) mà nó cần.
- Nhân bản (replicate) ứng dụng nếu bạn cần tính sẵn sàng cao hơn. (Tìm hiểu về việc chạy các ứng dụng nhân bản
  [không trạng thái (stateless)](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
  và [có trạng thái (stateful)](https://kubernetes.io/docs/tasks/run-application/run-replicated-stateful-application/).)
- Để có tính sẵn sàng cao hơn nữa khi chạy các ứng dụng nhân bản,
  hãy trải các ứng dụng ra nhiều rack (dùng
  [anti-affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity))
  hoặc nhiều zone (nếu dùng
  [cluster đa zone](https://kubernetes.io/docs/setup/multiple-zones)).

Tần suất của các gián đoạn tự nguyện thay đổi tùy trường hợp. Trên một cluster Kubernetes
cơ bản, không có gián đoạn tự nguyện tự động nào (chỉ có các gián đoạn do người dùng
kích hoạt). Tuy nhiên, quản trị viên cluster hoặc nhà cung cấp hosting của bạn có thể chạy
thêm một số dịch vụ gây ra gián đoạn tự nguyện. Ví dụ, việc triển khai cuốn chiếu
(rolling out) các bản cập nhật phần mềm node có thể gây ra gián đoạn tự nguyện. Ngoài ra,
một số cách hiện thực tính năng tự động co giãn cluster (node) có thể gây ra gián đoạn
tự nguyện nhằm chống phân mảnh và nén gọn các node.
Quản trị viên cluster hoặc nhà cung cấp hosting của bạn nên có tài liệu mô tả mức độ
gián đoạn tự nguyện có thể xảy ra, nếu có. Một số tùy chọn cấu hình nhất định, chẳng hạn
[dùng PriorityClass](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
trong spec của pod, cũng có thể gây ra gián đoạn tự nguyện (và không tự nguyện).

## Ngân sách gián đoạn Pod (Pod disruption budgets) {#pod-disruption-budgets}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [stable]`

Kubernetes cung cấp các tính năng giúp bạn chạy các ứng dụng có tính sẵn sàng cao
ngay cả khi bạn thường xuyên tạo ra các gián đoạn tự nguyện.

Với tư cách chủ sở hữu ứng dụng, bạn có thể tạo một PodDisruptionBudget (PDB) cho mỗi
ứng dụng. Một PDB giới hạn số lượng Pod của một ứng dụng nhân bản có thể ngừng hoạt động
đồng thời do các gián đoạn tự nguyện. Ví dụ, một ứng dụng dựa trên cơ chế đồng thuận
đa số (quorum) muốn đảm bảo rằng số lượng replica đang chạy không bao giờ xuống dưới
số lượng cần thiết để đạt quorum. Một web front end có thể muốn đảm bảo rằng số lượng
replica đang phục vụ tải không bao giờ giảm xuống dưới một tỷ lệ phần trăm nhất định
của tổng số.

Người quản lý cluster và nhà cung cấp hosting nên dùng các công cụ
tôn trọng PodDisruptionBudget bằng cách gọi [Eviction API](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/#eviction-api)
thay vì trực tiếp xóa pod hay deployment.

Ví dụ, lệnh con `kubectl drain` cho phép bạn đánh dấu một node là sắp ngừng
phục vụ. Khi bạn chạy `kubectl drain`, công cụ này cố trục xuất (evict) tất cả các Pod trên
node mà bạn đang đưa ra khỏi vòng phục vụ. Yêu cầu trục xuất mà `kubectl` gửi
thay cho bạn có thể tạm thời bị từ chối, vì vậy công cụ sẽ định kỳ thử lại tất cả các
yêu cầu thất bại cho đến khi tất cả Pod trên node đích bị kết thúc (terminate), hoặc
cho đến khi đạt tới thời gian chờ (timeout) có thể cấu hình được.

Một PDB chỉ định số lượng replica mà một ứng dụng có thể chấp nhận có, so với số lượng
mà nó được dự kiến có. Ví dụ, một Deployment có `.spec.replicas: 5` được
kỳ vọng có 5 pod tại bất kỳ thời điểm nào. Nếu PDB của nó cho phép có 4 pod tại một thời điểm,
thì Eviction API sẽ cho phép gián đoạn tự nguyện một pod (nhưng không phải hai) tại một thời điểm.

Nhóm các pod hợp thành ứng dụng được chỉ định bằng một label selector, giống với
selector được dùng bởi controller của ứng dụng đó (deployment, stateful-set, v.v.).

Số lượng pod "dự kiến" được tính từ `.spec.replicas` của workload resource
đang quản lý các pod đó. Control plane phát hiện workload resource sở hữu pod bằng cách
xem xét `.metadata.ownerReferences` của Pod.

[Gián đoạn không tự nguyện](#voluntary-and-involuntary-disruptions) không thể bị PDB
ngăn chặn; tuy nhiên chúng vẫn được tính vào ngân sách (budget).

Các Pod bị xóa hoặc không khả dụng do một đợt nâng cấp cuốn chiếu (rolling upgrade) của
ứng dụng vẫn được tính vào ngân sách gián đoạn, nhưng các workload resource (như Deployment
và StatefulSet) không bị PDB giới hạn khi thực hiện nâng cấp cuốn chiếu. Thay vào đó,
việc xử lý lỗi trong quá trình cập nhật ứng dụng được cấu hình trong spec của
workload resource cụ thể.

Bạn nên đặt [Unhealthy Pod Eviction Policy](https://kubernetes.io/docs/tasks/run-application/configure-pdb/#unhealthy-pod-eviction-policy)
là `AlwaysAllow` cho các PodDisruptionBudget của mình để hỗ trợ trục xuất các ứng dụng
hoạt động sai trong quá trình drain node.
Hành vi mặc định là chờ các pod của ứng dụng trở nên [khỏe mạnh (healthy)](https://kubernetes.io/docs/tasks/run-application/configure-pdb/#healthiness-of-a-pod)
trước khi quá trình drain có thể tiếp tục.

Khi một pod bị trục xuất bằng eviction API, nó được
[kết thúc](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination) một cách êm thấm (gracefully),
tôn trọng thiết lập `terminationGracePeriodSeconds` trong [PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podspec-v1-core) của nó.

## Ví dụ về PodDisruptionBudget (PodDisruptionBudget example) {#pdb-example}

Xét một cluster có 3 node, từ `node-1` đến `node-3`.
Cluster đang chạy một vài ứng dụng. Một trong số đó ban đầu có 3 replica gọi là
`pod-a`, `pod-b`, và `pod-c`. Một pod khác không liên quan và không có PDB, gọi là `pod-x`,
cũng được hiển thị. Ban đầu, các pod được bố trí như sau:

|       node-1         |       node-2        |       node-3       |
|:--------------------:|:-------------------:|:------------------:|
| pod-a  *available*   | pod-b *available*   | pod-c *available*  |
| pod-x  *available*   |                     |                    |

Cả 3 pod đều thuộc một deployment, và chúng cùng nhau có một PDB yêu cầu
phải có ít nhất 2 trong 3 pod khả dụng tại mọi thời điểm.

Ví dụ, giả sử quản trị viên cluster muốn khởi động lại máy vào một phiên bản kernel mới
để sửa một lỗi trong kernel. Quản trị viên cluster trước tiên thử drain `node-1` bằng
lệnh `kubectl drain`. Công cụ này cố trục xuất `pod-a` và `pod-x`. Việc này thành công
ngay lập tức. Cả hai pod đồng thời chuyển sang trạng thái `terminating`.
Điều này đưa cluster vào trạng thái sau:

|   node-1 *draining*  |       node-2        |       node-3       |
|:--------------------:|:-------------------:|:------------------:|
| pod-a  *terminating* | pod-b *available*   | pod-c *available*  |
| pod-x  *terminating* |                     |                    |

Deployment nhận thấy một trong các pod đang kết thúc, nên nó tạo một pod thay thế
gọi là `pod-d`. Vì `node-1` đã bị cordon, pod này được đặt lên một node khác. Một thứ
gì đó cũng đã tạo `pod-y` để thay thế cho `pod-x`.

(Lưu ý: với StatefulSet, `pod-a` — vốn sẽ có tên kiểu như `pod-0` — cần phải
kết thúc hoàn toàn trước khi pod thay thế của nó, cũng có tên `pod-0` nhưng có
UID khác, có thể được tạo. Ngoài điểm đó ra, ví dụ này cũng áp dụng cho StatefulSet.)

Bây giờ cluster ở trạng thái này:

|   node-1 *draining*  |       node-2        |       node-3       |
|:--------------------:|:-------------------:|:------------------:|
| pod-a  *terminating* | pod-b *available*   | pod-c *available*  |
| pod-x  *terminating* | pod-d *starting*    | pod-y              |

Đến một thời điểm nào đó, các pod kết thúc, và cluster trông như sau:

|    node-1 *drained*  |       node-2        |       node-3       |
|:--------------------:|:-------------------:|:------------------:|
|                      | pod-b *available*   | pod-c *available*  |
|                      | pod-d *starting*    | pod-y              |

Tại thời điểm này, nếu một quản trị viên cluster thiếu kiên nhẫn thử drain `node-2` hoặc
`node-3`, lệnh drain sẽ bị chặn (block), vì deployment chỉ có 2 pod khả dụng,
và PDB của nó yêu cầu ít nhất 2. Sau một khoảng thời gian, `pod-d` trở nên khả dụng.

Trạng thái cluster bây giờ trông như sau:

|    node-1 *drained*  |       node-2        |       node-3       |
|:--------------------:|:-------------------:|:------------------:|
|                      | pod-b *available*   | pod-c *available*  |
|                      | pod-d *available*   | pod-y              |

Bây giờ, quản trị viên cluster thử drain `node-2`.
Lệnh drain sẽ cố trục xuất hai pod theo một thứ tự nào đó, giả sử
`pod-b` trước rồi đến `pod-d`. Nó sẽ trục xuất được `pod-b`.
Nhưng khi cố trục xuất `pod-d`, nó sẽ bị từ chối vì làm vậy sẽ chỉ còn lại
một pod khả dụng cho deployment.

Deployment tạo một pod thay thế cho `pod-b` gọi là `pod-e`.
Vì không đủ tài nguyên trong cluster để lập lịch (schedule) cho
`pod-e`, quá trình drain lại bị chặn tiếp. Cluster có thể rơi vào
trạng thái sau:

|    node-1 *drained*  |       node-2        |       node-3       | *không có node*    |
|:--------------------:|:-------------------:|:------------------:|:------------------:|
|                      | pod-b *terminating* | pod-c *available*  | pod-e *pending*    |
|                      | pod-d *available*   | pod-y              |                    |

Tại thời điểm này, quản trị viên cluster cần thêm một node trở lại
cluster để tiếp tục việc nâng cấp.

Bạn có thể thấy cách Kubernetes điều chỉnh tốc độ mà các gián đoạn
có thể xảy ra, dựa theo:

- ứng dụng cần bao nhiêu replica
- mất bao lâu để tắt một instance một cách êm thấm (graceful shutdown)
- mất bao lâu để một instance mới khởi động
- loại controller
- năng lực tài nguyên của cluster

## Các condition gián đoạn của Pod (Pod disruption conditions) {#pod-disruption-conditions}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.31 [stable]`

Một [condition](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-conditions) `DisruptionTarget`
dành riêng được thêm vào Pod để chỉ ra rằng
Pod sắp bị xóa do một sự gián đoạn (disruption).
Trường `reason` của condition này còn chỉ ra thêm
một trong các lý do sau cho việc kết thúc Pod:

`PreemptionByScheduler`
: Pod sắp bị scheduler chiếm chỗ (preempt) để nhường chỗ cho một Pod mới có độ ưu tiên cao hơn. Để biết thêm thông tin, xem [Chiếm chỗ theo độ ưu tiên của Pod](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/).

`DeletionByTaintManager`
: Pod sắp bị Taint Manager (một phần của node lifecycle controller bên trong `kube-controller-manager`) xóa do có taint `NoExecute` mà Pod không dung thứ (tolerate); xem cơ chế trục xuất dựa trên taint.

`EvictionByEvictionAPI`
: Pod đã được đánh dấu để trục xuất bằng Kubernetes API.

`DeletionByPodGC`
: Pod đang gắn với một node không còn tồn tại, sắp bị xóa bởi [cơ chế thu gom rác Pod (Pod garbage collection)](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-garbage-collection).

`TerminationByKubelet`
: Pod đã bị kubelet kết thúc, do trục xuất vì áp lực tài nguyên node (node pressure eviction),
  do [tắt node êm thấm (graceful node shutdown)](https://kubernetes.io/docs/concepts/architecture/nodes/#graceful-node-shutdown),
  hoặc do bị chiếm chỗ để nhường cho [các pod quan trọng của hệ thống](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/).

Trong tất cả các kịch bản gián đoạn khác, như trục xuất do vượt quá
[giới hạn tài nguyên container của Pod](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/),
Pod không nhận condition `DisruptionTarget` vì các gián đoạn đó nhiều khả năng
do chính Pod gây ra và sẽ tái diễn nếu thử lại.

> **Ghi chú:**
> Một sự gián đoạn Pod có thể bị ngắt giữa chừng. Control plane có thể thử
> tiếp tục gián đoạn cùng Pod đó, nhưng điều này không được đảm bảo. Kết quả là
> condition `DisruptionTarget` có thể được thêm vào một Pod, nhưng sau đó Pod đó
> có thể thực tế không bị xóa. Trong tình huống như vậy, sau một khoảng thời gian,
> condition gián đoạn của Pod sẽ được xóa bỏ.

Cùng với việc dọn dẹp các pod, bộ thu gom rác Pod (PodGC) cũng sẽ đánh dấu chúng là
failed nếu chúng đang ở một phase chưa kết thúc (non-terminal)
(xem thêm [Pod garbage collection](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-garbage-collection)).

Khi dùng Job (hoặc CronJob), bạn có thể muốn dùng các condition gián đoạn Pod này
như một phần của [chính sách xử lý Pod lỗi (Pod failure policy)](https://kubernetes.io/docs/concepts/workloads/controllers/job#pod-failure-policy) của Job.

## Tách biệt vai trò Chủ sở hữu Cluster và Chủ sở hữu Ứng dụng (Separating Cluster Owner and Application Owner Roles) {#separating-cluster-owner-and-application-owner-roles}

Thông thường, sẽ hữu ích khi xem Người quản lý Cluster
và Chủ sở hữu Ứng dụng là các vai trò tách biệt với hiểu biết hạn chế
về nhau. Sự phân tách trách nhiệm này
có thể hợp lý trong các kịch bản sau:

- khi có nhiều nhóm ứng dụng dùng chung một cluster Kubernetes, và
  có sự chuyên môn hóa vai trò một cách tự nhiên
- khi các công cụ hoặc dịch vụ bên thứ ba được dùng để tự động hóa việc quản lý cluster

Pod Disruption Budget hỗ trợ sự phân tách vai trò này bằng cách cung cấp một
giao diện giữa các vai trò.

Nếu tổ chức của bạn không có sự phân tách trách nhiệm như vậy,
có thể bạn không cần dùng Pod Disruption Budget.

## Cách thực hiện các hành động gây gián đoạn trên Cluster (How to perform Disruptive Actions on your Cluster) {#how-to-perform-disruptive-actions-on-your-cluster}

Nếu bạn là Quản trị viên Cluster và cần thực hiện một hành động gây gián đoạn trên tất cả
các node trong cluster, chẳng hạn nâng cấp phần mềm node hoặc phần mềm hệ thống, dưới đây
là một số lựa chọn:

- Chấp nhận thời gian ngừng hoạt động (downtime) trong quá trình nâng cấp.
- Chuyển đổi dự phòng (failover) sang một cluster bản sao hoàn chỉnh khác.
   - Không có downtime, nhưng có thể tốn kém, cả về số node nhân đôi
     lẫn công sức con người để điều phối việc chuyển đổi.
- Viết các ứng dụng chịu được gián đoạn và dùng PDB.
   - Không có downtime.
   - Nhân đôi tài nguyên ở mức tối thiểu.
   - Cho phép tự động hóa việc quản trị cluster nhiều hơn.
   - Viết ứng dụng chịu được gián đoạn là việc không đơn giản, nhưng phần công sức
     bỏ ra để chịu được gián đoạn tự nguyện phần lớn trùng lặp với công sức hỗ trợ
     autoscaling và chịu được gián đoạn không tự nguyện.

## Tiếp theo (What's next)

* Làm theo các bước để bảo vệ ứng dụng của bạn bằng cách
  [cấu hình một Pod Disruption Budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/).

* Tìm hiểu thêm về [drain node](https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)

* Tìm hiểu về [cập nhật một deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#updating-a-deployment),
  bao gồm các bước để duy trì tính khả dụng của nó trong quá trình rollout.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Một ứng dụng 3 replica có PDB yêu cầu luôn còn ít nhất 2 pod khả dụng. Đồng nghiệp chạy
   `kubectl delete deployment` lên chính ứng dụng đó. PDB có chặn không?
2. `k8s-worker2` bị kernel panic và biến mất khỏi cluster. Đó là loại gián đoạn nào, PDB có
   bảo vệ được không, và nó có ảnh hưởng gì tới các gián đoạn tự nguyện sau đó?
3. Cluster lab của bạn chỉ có hai worker. Một ứng dụng 3 replica có PDB "ít nhất 2 pod khả
   dụng", các pod nằm rải trên cả hai worker. Bạn chạy `kubectl drain k8s-worker2`. Những gì
   quyết định lệnh này chạy xong hay treo lại?
4. Số pod "dự kiến" mà PDB dùng để tính ngân sách lấy từ đâu? Control plane biết pod nào
   thuộc workload nào bằng cách nào?
5. Một Deployment 3 replica có PDB "ít nhất 2 pod khả dụng" đang chạy rolling update, và
   trong lúc đó có 1 pod tạm thời không khả dụng. Rollout có bị PDB chặn không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không chặn.** Đây là bẫy kinh điển: PDB *trông như* một hàng rào bảo vệ chung cho ứng
   dụng, nhưng nó chỉ ràng buộc những gì đi qua **Eviction API**. Khối *Thận trọng* nói thẳng:
   "Không phải mọi gián đoạn tự nguyện đều bị ràng buộc bởi Pod Disruption Budget. Ví dụ, việc
   xóa deployment hoặc pod sẽ bỏ qua Pod Disruption Budget." Vì vậy bài khuyến nghị người quản
   lý cluster và nhà cung cấp hosting **dùng công cụ tôn trọng PDB bằng cách gọi Eviction API**
   thay vì xóa trực tiếp. PDB bảo vệ khỏi tự động hóa cư xử đúng mực, không bảo vệ khỏi
   `kubectl delete`.
2. **Gián đoạn không tự nguyện** — kernel panic nằm đúng trong danh sách ví dụ ở đầu bài, cùng
   với lỗi phần cứng và mất node do phân mảnh mạng. **PDB không ngăn được** chúng: "Gián đoạn
   không tự nguyện không thể bị PDB ngăn chặn; **tuy nhiên chúng vẫn được tính vào ngân sách**."
   Hệ quả trực tiếp: sau sự cố đó, ngân sách đã bị tiêu hết, nên một thao tác drain hay nâng
   cấp *tự nguyện* ngay sau đó sẽ bị từ chối cho tới khi ứng dụng phục hồi đủ replica.
3. `kubectl drain` gọi Eviction API cho từng pod trên node; yêu cầu bị từ chối sẽ được
   **thử lại định kỳ** cho tới khi mọi pod trên node kết thúc hoặc **hết timeout cấu hình
   được**. Nó chỉ đi tiếp khi ứng dụng vẫn còn ≥ 2 pod khả dụng — nghĩa là pod thay thế phải
   được tạo và trở nên khả dụng trên `k8s-worker1`. Nếu worker còn lại **không đủ tài nguyên**
   để lập lịch pod thay thế, pod đó nằm `Pending` và drain bị chặn tiếp — đúng tình huống
   `pod-e` trong ví dụ của bài, và lối thoát ở đó là **thêm node**. Cuối mục ví dụ liệt kê
   đủ những gì điều tiết tốc độ này: số replica cần, thời gian tắt êm, thời gian một instance
   mới khởi động, loại controller, và năng lực tài nguyên của cluster.
4. Lấy từ **`.spec.replicas` của workload resource đang quản lý các pod đó**. Control plane
   phát hiện workload nào sở hữu một pod bằng cách xem **`.metadata.ownerReferences`** của
   Pod. Còn nhóm pod hợp thành ứng dụng thì được chỉ định bằng **label selector**, giống
   selector mà controller của ứng dụng dùng.
5. **Không.** Các pod bị xóa hoặc không khả dụng trong một đợt nâng cấp cuốn chiếu **vẫn được
   tính vào ngân sách gián đoạn**, nhưng bản thân workload resource (Deployment, StatefulSet)
   **không bị PDB giới hạn khi thực hiện nâng cấp cuốn chiếu**. Việc kiểm soát mức không khả
   dụng trong lúc rollout được cấu hình trong **spec của chính workload**, không phải trong PDB.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
