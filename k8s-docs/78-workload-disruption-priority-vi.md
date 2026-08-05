# Gián đoạn và độ ưu tiên của Pod Group (Pod Group Disruption and Priority)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/workload-api/disruption-and-priority/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

PodGroup có thể khai báo một chế độ gián đoạn (disruption mode). Chế độ này quy định cách
scheduler có thể làm gián đoạn một PodGroup đang chạy, ví dụ để nhường chỗ cho
một PodGroup có độ ưu tiên cao hơn. PodGroup cũng có một độ ưu tiên (priority),
ghi đè độ ưu tiên của từng pod riêng lẻ trong nhóm
đối với các sự kiện [chiếm chỗ nhận biết workload (workload-aware preemption)](https://kubernetes.io/docs/concepts/scheduling-eviction/workload-aware-preemption/).

## Các loại chế độ gián đoạn (Disruption mode types)

> **Ghi chú:**
> Tính đến 1.36, các trường `priority` hoặc `disruptionMode` của PodGroup chỉ được tôn trọng
> bởi [workload-aware preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/workload-aware-preemption/).
> Trong giai đoạn lập lịch pod, scheduler không xét đến
> các trường `priority` hoặc `disruptionMode` của PodGroup.

API hỗ trợ hai chế độ gián đoạn: `Pod` và `PodGroup`.
Chế độ mặc định là `Pod`.

### Pod

Chế độ `Pod` chỉ thị cho scheduler coi tất cả các Pod trong nhóm là những thực thể riêng biệt,
cho phép làm gián đoạn độc lập một pod đơn lẻ trong PodGroup.

### PodGroup

Chế độ `PodGroup` nhấn mạnh ngữ nghĩa "tất cả hoặc không gì cả" (all-or-nothing) đối với việc gián đoạn.
Nó chỉ thị cho scheduler rằng tất cả các pod trong PodGroup phải bị gián đoạn cùng nhau.

## Độ ưu tiên của pod group (Pod group priority)

PodGroup sử dụng cùng khái niệm [PriorityClass](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#priorityclass) như các Pod đơn lẻ.
Sau khi bạn đã tạo một hoặc nhiều PriorityClass,
bạn có thể tạo một PodGroup chỉ định tên của một trong các PriorityClass đó trong spec của nó.
Admission controller về priority sử dụng trường `priorityClassName` và điền giá trị số nguyên của độ ưu tiên.
Nếu không tìm thấy priority class, PodGroup sẽ bị từ chối.
Khi `priorityClassName` không được đặt cho một PodGroup, Kubernetes sẽ tìm một giá trị mặc định (một PriorityClass có `globalDefault` được đặt là true).
Nếu không có PriorityClass nào có `globalDefault` được đặt là true, PodGroup không có `priorityClassName` sẽ có độ ưu tiên bằng không.

Độ ưu tiên của PodGroup là độ ưu tiên có tính quyết định (authoritative) cho tất cả các pod trong nhóm trong các sự kiện [workload-aware preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/workload-aware-preemption/), ngay cả khi độ ưu tiên của từng pod riêng lẻ tạo nên PodGroup này khác nhau.

YAML sau đây là một ví dụ về cấu hình PodGroup sử dụng PriorityClass `high-priority`,
tương ứng với giá trị độ ưu tiên số nguyên 1000000.
Admission controller về priority kiểm tra spec và phân giải độ ưu tiên của PodGroup thành 1000000.

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  namespace: ns-1
  name: job-1
spec:
  priorityClassName: high-priority
```

## Tiếp theo (What's next)

* Đọc về thuật toán [Workload-Aware Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/workload-aware-preemption/).
* Tìm hiểu về [Workload API](./77-workload-api-vi.md).
