# Preemption nhận biết workload (Workload-Aware Preemption)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/workload-aware-preemption/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Preemption (chiếm chỗ) nhận biết workload giới thiệu một cơ chế preemption được thiết kế riêng cho các PodGroup.
Khi một PodGroup không thể được lập lịch, scheduler sử dụng một logic preemption cố gắng
làm cho việc lập lịch PodGroup này trở nên khả thi. Cách tiếp cận này chỉ được dùng trong quá trình lập lịch PodGroup
và thay thế cơ chế preemption mặc định cho các pod thuộc một PodGroup nhất định.

Khi tính năng này được bật, scheduler coi PodGroup như một đơn vị preemptor (bên chiếm chỗ) duy nhất,
thay vì đánh giá từng pod riêng lẻ của PodGroup một cách cô lập. Để nhường chỗ cho các pod đang chờ trong nhóm,
nó tìm kiếm các nạn nhân (victim) trên toàn bộ cluster,
và biết cách đối xử cũng như preempt các PodGroup khác với vai trò nạn nhân theo các chế độ gián đoạn (disruption mode) của chúng.

Tính năng này phụ thuộc vào [Gang Scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)
và [Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/).
Hãy đảm bảo các feature gate [`GenericWorkload`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#GenericWorkload)
và [`GangScheduling`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#GangScheduling)
cùng nhóm API (API group) `scheduling.k8s.io/v1alpha2` đã được bật trong cluster.

## Cách hoạt động (How it works)

Quá trình preemption nhận biết workload tuân theo cùng các nguyên tắc
như [preemption mặc định](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#preemption)
với một vài khác biệt:

1. Miền toàn cluster (cluster-wide domain): Thay vì đánh giá preemption theo từng node một,
   scheduler đánh giá toàn bộ cluster như một miền duy nhất.
   Nó chọn ra một tập các nạn nhân trải trên nhiều node có thể bị loại bỏ
   để tạo đủ chỗ cho PodGroup preemptor được lập lịch.

2. Thứ bậc tầm quan trọng của nạn nhân (victim importance hierarchy): Scheduler quyết định những đơn vị preemption nào
   (các pod riêng lẻ hoặc các PodGroup) quan trọng hơn và cần được miễn trừ khỏi preemption
   dựa trên một thứ bậc nghiêm ngặt:
   * Độ ưu tiên (priority): Đơn vị có độ ưu tiên cao hơn luôn quan trọng hơn.
   * Loại workload: PodGroup được coi là quan trọng hơn các Pod riêng lẻ có cùng độ ưu tiên.
   * Kích thước nhóm (với PodGroup): Nếu cả hai đơn vị đều là PodGroup,
     đơn vị có nhiều thành viên hơn (kích thước lớn hơn) được coi là quan trọng hơn.
   * Thời điểm khởi động (start time): Đơn vị khởi động sớm hơn thì quan trọng hơn.

3. Độ ưu tiên và sự gián đoạn của pod group: Scheduler xem xét
   [độ ưu tiên và chế độ gián đoạn](https://kubernetes.io/docs/concepts/workloads/workload-api/disruption-and-priority/) cụ thể của một PodGroup
   để đánh giá liệu các pod của nó có thể bị preempt hay không và bị preempt như thế nào trong các sự kiện preemption.

> **Ghi chú:**
> Khi lập lịch một Pod đơn lẻ, cơ chế preemption mặc định cho pod được áp dụng.
> Tính đến 1.36, khi scheduler thực hiện preemption mặc định cho một Pod đơn lẻ
> và nó cố gắng preempt một Pod thuộc về một PodGroup, nó **không**
> tôn trọng các trường `priority` hoặc `disruptionMode` của PodGroup đó.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [Độ ưu tiên và Sự gián đoạn của PodGroup](https://kubernetes.io/docs/concepts/workloads/workload-api/disruption-and-priority/).
* Tìm hiểu về [Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/).
* Đọc thêm về [Gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).
