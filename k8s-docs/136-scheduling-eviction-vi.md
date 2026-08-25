# Lập lịch, Preemption và Eviction (Scheduling, Preemption and Eviction)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 7 → nhóm [7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction), bài 1/13 ·
Kiểm chứng ở [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md).

Đây là **trang mục lục**, không phải bài học. Phần có nội dung thật chỉ khoảng mười dòng: đoạn
mở đầu định nghĩa ba từ khóa, và mục *Sự gián đoạn Pod*. Phần còn lại là hai danh sách link —
chúng chính là 12 bài kế tiếp của nhóm 7a cộng vài bài thuộc giai đoạn sau. Đọc để lấy bản đồ,
đừng cố hiểu từng mục trong danh sách.

**Phải hiểu ở lần đọc này:**

- Ba từ khóa tách bạch nhau: **lập lịch** là ghép Pod với Node để kubelet chạy được;
  **preemption** là chấm dứt Pod có Priority thấp hơn để Pod Priority cao hơn được lập lịch;
  **eviction** là chấm dứt một hoặc nhiều Pod đang nằm trên Node.
- Lập lịch xảy ra **trước khi** Pod chạy; preemption và eviction tác động vào Pod **đã có
  chỗ** trên node. Đó là ranh giới chia đôi cả nhóm bài này.
- Mục *Sự gián đoạn Pod*: gián đoạn **tự nguyện** do chủ ứng dụng hoặc quản trị viên chủ đích
  gây ra, gián đoạn **không tự nguyện** đến từ node cạn kiệt tài nguyên hoặc xóa nhầm.
- Trang gốc xếp Pod Priority/Preemption, node-pressure eviction và API-initiated eviction cùng
  vào mục *Sự gián đoạn Pod* — nghĩa là cả ba đều kết thúc bằng việc Pod bị chấm dứt.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Các link *Cấp phát tài nguyên động*, *Lập lịch PodGroup*, *Gang Scheduling*, *Lập lịch nhận biết topology*, *Preemption nhận biết workload* | thuộc lập lịch nâng cao, ngoài nhóm 7a | giai đoạn 13 |
| Link *Các tính năng do Node khai báo* | thuộc nhóm sau của cùng giai đoạn | nhóm [7b](00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên), bài [154](154-node-declared-features-vi.md) |
| Link *Descheduler* | công cụ ngoài Kubernetes core | không cần |

---

Trong Kubernetes, lập lịch (scheduling) là việc đảm bảo các Pod
được ghép cặp với các Node sao cho kubelet có thể chạy chúng. Preemption (chiếm chỗ)
là quá trình chấm dứt các Pod có Priority (độ ưu tiên) thấp hơn
để các Pod có Priority cao hơn có thể được lập lịch lên Node. Eviction (thu hồi) là quá trình
chấm dứt một hoặc nhiều Pod trên các Node.

## Lập lịch (Scheduling) {#scheduling}

* [Bộ lập lịch của Kubernetes (Kubernetes Scheduler)](137-kube-scheduler-vi.md)
* [Gán Pod cho Node (Assigning Pods to Nodes)](138-assign-pod-node-vi.md)
* [Pod Overhead](144-pod-overhead-vi.md)
* [Ràng buộc phân bố Pod theo topology (Pod Topology Spread Constraints)](140-topology-spread-constraints-vi.md)
* [Taint và Toleration (Taints and Tolerations)](139-taint-and-toleration-vi.md)
* [Scheduling Framework](147-scheduling-framework-vi.md)
* [Cấp phát tài nguyên động (Dynamic Resource Allocation)](149-dynamic-resource-allocation-vi.md)
* [Tinh chỉnh hiệu năng bộ lập lịch (Scheduler Performance Tuning)](146-scheduler-perf-tuning-vi.md)
* [Đóng gói tài nguyên cho các tài nguyên mở rộng (Resource Bin Packing for Extended Resources)](148-resource-bin-packing-vi.md)
* [Mức sẵn sàng lập lịch của Pod (Pod Scheduling Readiness)](145-pod-scheduling-readiness-vi.md)
* [Lập lịch PodGroup (PodGroup Scheduling)](151-podgroup-scheduling-vi.md)
* [Gang Scheduling](150-gang-scheduling-vi.md)
* [Lập lịch nhận biết topology (Topology-aware Scheduling)](153-topology-aware-scheduling-vi.md)
* [Preemption nhận biết workload (Workload-Aware preemption)](152-workload-aware-preemption-vi.md)
* [Descheduler](https://github.com/kubernetes-sigs/descheduler#descheduler-for-kubernetes)
* [Các tính năng do Node khai báo (Node Declared Features)](154-node-declared-features-vi.md)

## Sự gián đoạn Pod (Pod Disruption)

[Sự gián đoạn Pod (Pod disruption)](./53-disruptions-vi.md) là quá trình trong đó
các Pod trên các Node bị chấm dứt một cách tự nguyện (voluntary) hoặc không tự nguyện (involuntary).

Gián đoạn tự nguyện được khởi phát một cách có chủ đích bởi chủ sở hữu ứng dụng hoặc
quản trị viên cluster. Gián đoạn không tự nguyện là ngoài ý muốn và có thể bị kích hoạt bởi
những vấn đề không thể tránh khỏi như Node cạn kiệt tài nguyên (resources),
hoặc do việc xóa nhầm.

* [Độ ưu tiên và Preemption của Pod (Pod Priority and Preemption)](141-pod-priority-preemption-vi.md)
* [Eviction do áp lực node (Node-pressure Eviction)](142-node-pressure-eviction-vi.md)
* [Eviction khởi phát qua API (API-initiated Eviction)](143-api-eviction-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 7:

1. Ba từ lập lịch, preemption và eviction mô tả chuyện gì xảy ra với Pod, và mỗi chuyện xảy
   ra ở thời điểm nào so với lúc Pod bắt đầu chạy?
2. Preemption và eviction đều kết thúc bằng việc Pod bị chấm dứt. Vậy chúng có phải là hai
   tên gọi của cùng một việc không?
3. Bạn chạy `kubectl drain lab-k8s-worker2` để bảo trì máy. Việc các Pod rời node đó là gián đoạn
   tự nguyện hay không tự nguyện? Còn khi `lab-k8s-worker2` hết RAM và Pod bị chấm dứt thì sao?
4. Trang này xếp Pod Priority/Preemption vào mục nào, chứ không xếp vào mục *Lập lịch*? Điều
   đó gợi ý gì về bản chất của preemption?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Lập lịch** = ghép Pod với Node sao cho kubelet có thể chạy chúng — xảy ra **trước** khi
   Pod chạy. **Preemption** = chấm dứt các Pod có Priority thấp hơn **để** một Pod Priority cao
   hơn được lập lịch — tác động lên Pod đang chạy nhằm phục vụ một Pod chưa chạy. **Eviction**
   = chấm dứt một hoặc nhiều Pod trên các Node — tác động lên Pod đã có chỗ.
2. **Không.** Điểm phân biệt là **nguyên nhân**, không phải kết cục. Preemption luôn có một
   Pod đang chờ với Priority cao hơn đứng đằng sau và mục tiêu là **làm cho Pod đó lập lịch
   được**; eviction nói chung chỉ là việc chấm dứt Pod trên node, không cần có ai chờ chỗ.
   Trực giác "cùng chết là cùng một thứ" sai vì nó nhìn kết quả thay vì nhìn nguyên nhân.
3. `kubectl drain` là **gián đoạn tự nguyện** — bài định nghĩa gián đoạn tự nguyện là loại
   được khởi phát **có chủ đích** bởi chủ sở hữu ứng dụng hoặc quản trị viên cluster.
   `lab-k8s-worker2` hết RAM là **gián đoạn không tự nguyện**: bài nêu đúng ví dụ "Node cạn kiệt
   tài nguyên" là thứ ngoài ý muốn.
4. Trang xếp nó vào ***Sự gián đoạn Pod***, cùng chỗ với node-pressure eviction và
   API-initiated eviction. Nghĩa là dù preemption phục vụ mục tiêu lập lịch, **hệ quả của nó
   là một gián đoạn** đối với các Pod đang chạy — nên nó được nhìn cùng góc với eviction.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
