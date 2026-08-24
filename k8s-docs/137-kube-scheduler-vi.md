# Bộ lập lịch của Kubernetes (Kubernetes Scheduler)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), bài 2/13 ·
Kiểm chứng ở Lab 7a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài ngắn nhưng là **trục của cả nhóm 7a**. Mọi cơ chế ở các bài sau — nodeSelector, affinity,
taint, topology spread, preemption, bin packing — đều chỉ là cách can thiệp vào một trong hai
bước mà bài này mô tả. Đọc kỹ mục *Chọn node trong kube-scheduler*, còn lại đọc một lượt.

**Phải hiểu ở lần đọc này:**

- Scheduler chỉ theo dõi các Pod **chưa được gán Node**, và sản phẩm cuối cùng của nó là
  **binding** — thông báo quyết định cho API server. Nó không khởi chạy container.
- Hai bước theo đúng thứ tự: **lọc** (filtering) tìm tập node _khả thi_ (feasible), rồi **chấm
  điểm** (scoring) xếp hạng chính tập đó. Node ngoài tập khả thi không bao giờ được chấm điểm.
- Nếu danh sách sau bước lọc **rỗng**, Pod ở trạng thái chưa được lập lịch cho tới khi
  scheduler sắp đặt được nó — chứ không phải Pod bị lỗi.
- Nhiều node cùng điểm cao nhất thì kube-scheduler **chọn ngẫu nhiên** một trong số đó.
- API cho phép chỉ định thẳng node khi tạo Pod, nhưng bài nói rõ cách này **không phổ biến** và
  chỉ dùng cho trường hợp đặc biệt (chính là `nodeName` ở bài
  [138](138-assign-pod-node-vi.md)).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Scheduling Policies* — Predicate và Priority | là cách cấu hình thế hệ cũ | không cần |
| *Scheduling Profiles* và danh sách `QueueSort`, `Filter`, `Score`, `Bind`, `Reserve`, `Permit` | đây là tên các điểm mở rộng, có bài riêng | bài [147](147-scheduling-framework-vi.md) |
| Ý "bạn có thể tự viết scheduler và dùng thay thế" | chỉ cần biết là làm được | giai đoạn 14, bài [177](177-extend-kubernetes-vi.md) |
| Các link cuối bài về volume topology, storage capacity | đã đọc ở giai đoạn 6 | bài [103](103-storage-capacity-vi.md) |

---

Trong Kubernetes, _lập lịch_ (scheduling) là việc đảm bảo các Pod được ghép cặp
với các Node sao cho kubelet có thể chạy chúng.

## Tổng quan về lập lịch (Scheduling overview) {#scheduling}

Một bộ lập lịch (scheduler) theo dõi các Pod mới được tạo mà chưa được gán
Node nào. Với mỗi Pod mà scheduler phát hiện, scheduler chịu trách nhiệm
tìm Node tốt nhất để Pod đó chạy trên. Scheduler đưa ra quyết định
sắp đặt này dựa trên các nguyên tắc lập lịch được mô tả bên dưới.

Nếu bạn muốn hiểu vì sao các Pod được đặt lên một Node cụ thể,
hoặc nếu bạn đang dự định tự triển khai một scheduler tùy chỉnh, trang này
sẽ giúp bạn tìm hiểu về lập lịch.

## kube-scheduler

[kube-scheduler](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
là scheduler mặc định của Kubernetes và chạy như một phần của control plane.
kube-scheduler được thiết kế sao cho, nếu bạn muốn và cần, bạn có thể
viết thành phần lập lịch của riêng mình và dùng nó thay thế.

Kube-scheduler chọn một node tối ưu để chạy các pod mới được tạo hoặc chưa
được lập lịch (unscheduled). Vì các container trong pod — và bản thân các pod —
có thể có những yêu cầu khác nhau, scheduler lọc bỏ những node
không đáp ứng các nhu cầu lập lịch cụ thể của Pod. Ngoài ra, API cho phép
bạn chỉ định node cho một Pod ngay khi tạo Pod, nhưng cách này không phổ biến
và chỉ được dùng trong các trường hợp đặc biệt.

Trong một cluster, các Node đáp ứng các yêu cầu lập lịch của một Pod
được gọi là các node _khả thi_ (feasible). Nếu không có node nào phù hợp, pod
sẽ ở trạng thái chưa được lập lịch cho đến khi scheduler có thể sắp đặt được nó.

Scheduler tìm các Node khả thi cho một Pod, sau đó chạy một tập các
hàm để chấm điểm các Node khả thi này và chọn Node có điểm cao nhất
trong số đó để chạy Pod. Scheduler sau đó thông báo cho
API server về quyết định này trong một quá trình gọi là _binding_ (gắn kết).

Các yếu tố cần được xét đến khi ra quyết định lập lịch bao gồm
yêu cầu tài nguyên riêng lẻ và tổng hợp, các ràng buộc về phần cứng / phần mềm /
chính sách, các đặc tả affinity và anti-affinity, tính cục bộ của
dữ liệu (data locality), sự can nhiễu giữa các workload, v.v.

### Chọn node trong kube-scheduler (Node selection in kube-scheduler) {#kube-scheduler-implementation}

kube-scheduler chọn node cho pod qua một thao tác gồm 2 bước:

1. Lọc (Filtering)
1. Chấm điểm (Scoring)

Bước _lọc_ tìm ra tập các Node khả thi để lập lịch Pod.
Ví dụ, bộ lọc PodFitsResources kiểm tra xem một Node ứng viên
có đủ tài nguyên khả dụng để đáp ứng các yêu cầu tài nguyên (resource request)
cụ thể của Pod hay không. Sau bước này, danh sách node chứa những Node
phù hợp; thường sẽ có nhiều hơn một Node. Nếu danh sách rỗng, Pod đó
(tạm thời) chưa thể được lập lịch.

Trong bước _chấm điểm_, scheduler xếp hạng các node còn lại để chọn
nơi đặt Pod phù hợp nhất. Scheduler gán một điểm số cho mỗi Node
đã vượt qua bước lọc, dựa trên các quy tắc chấm điểm đang có hiệu lực.

Cuối cùng, kube-scheduler gán Pod cho Node có thứ hạng cao nhất.
Nếu có nhiều hơn một node với điểm số bằng nhau, kube-scheduler chọn
ngẫu nhiên một trong số đó.

Có hai cách được hỗ trợ để cấu hình hành vi lọc và chấm điểm
của scheduler:

1. [Scheduling Policies](https://kubernetes.io/docs/reference/scheduling/policies) cho phép bạn cấu hình các _Predicate_ cho bước lọc và các _Priority_ cho bước chấm điểm.
1. [Scheduling Profiles](https://kubernetes.io/docs/reference/scheduling/config/#profiles) cho phép bạn cấu hình các Plugin hiện thực các giai đoạn lập lịch khác nhau, bao gồm: `QueueSort`, `Filter`, `Score`, `Bind`, `Reserve`, `Permit`, và các giai đoạn khác. Bạn cũng có thể cấu hình kube-scheduler để chạy các profile khác nhau.

## Tiếp theo (What's next)

* Đọc về [tinh chỉnh hiệu năng bộ lập lịch (scheduler performance tuning)](146-scheduler-perf-tuning-vi.md)
* Đọc về [ràng buộc phân bố Pod theo topology (Pod topology spread constraints)](140-topology-spread-constraints-vi.md)
* Đọc [tài liệu tham khảo](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/) của kube-scheduler
* Đọc tài liệu tham khảo [cấu hình kube-scheduler (v1)](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/)
* Tìm hiểu về [cấu hình nhiều scheduler](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)
* Tìm hiểu về [các chính sách quản lý topology (topology management policies)](259-topology-manager-vi.md)
* Tìm hiểu về [Pod Overhead](144-pod-overhead-vi.md)
* Tìm hiểu về việc lập lịch cho các Pod sử dụng volume trong:
  * [Hỗ trợ Volume Topology](96-storage-classes-vi.md#volume-binding-mode)
  * [Theo dõi dung lượng lưu trữ (Storage Capacity Tracking)](103-storage-capacity-vi.md)
  * [Giới hạn Volume theo từng Node (Node-specific Volume Limits)](104-storage-limits-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Kể hai bước của kube-scheduler theo đúng thứ tự, và cho biết mỗi bước nhận vào gì, trả ra
   gì.
2. Sau khi chọn được node, kube-scheduler có tự khởi chạy container trên node đó không? Nếu
   không thì nó làm gì, và ai chạy container?
3. Trên cluster lab, mỗi worker `k8s-worker1` và `k8s-worker2` có 2 vCPU. Bạn tạo một Pod xin
   `requests` 4 CPU. Bước nào của scheduler loại hai worker đó, và Pod kết thúc ở trạng thái
   nào?
4. Cả hai worker đều khả thi và được chấm điểm bằng nhau. Pod lên node nào? Nếu bạn tạo lại
   Pod đó vài lần, kết quả có lặp lại giống nhau không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Lọc (filtering) rồi chấm điểm (scoring).** Lọc nhận toàn bộ node của cluster và trả ra
   **tập node khả thi** — ví dụ bộ lọc `PodFitsResources` kiểm tra node ứng viên có đủ tài
   nguyên khả dụng cho resource request của Pod hay không. Chấm điểm nhận **chính tập khả thi
   đó** và trả ra một điểm số cho mỗi node; scheduler gán Pod cho node có thứ hạng cao nhất.
2. **Không.** Scheduler **thông báo cho API server** về quyết định — quá trình đó gọi là
   **binding**. Việc chạy container là của **kubelet** trên node được gán.
3. Bước **lọc** loại cả hai, vì không node nào đủ tài nguyên khả dụng đáp ứng request. Danh
   sách node khả thi **rỗng**, nên Pod **ở trạng thái chưa được lập lịch** cho đến khi
   scheduler sắp đặt được nó. Đây không phải lỗi của Pod — bước chấm điểm thậm chí không chạy
   vì không có node nào đi tới đó.
4. **Không đoán trước được: kube-scheduler chọn ngẫu nhiên** một trong các node có điểm bằng
   nhau. Vì vậy chạy lại nhiều lần bạn có thể thấy Pod rơi vào worker khác nhau — đó là hành
   vi đúng, không phải dấu hiệu cấu hình sai.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
