# Lập lịch theo nhóm (Gang Scheduling)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [alpha]`

Gang scheduling (lập lịch theo nhóm) đảm bảo một nhóm Pod được lập lịch theo nguyên tắc "tất cả hoặc không gì cả" (all-or-nothing).
Nếu cluster không thể chứa toàn bộ nhóm (hoặc một số lượng Pod tối thiểu được định nghĩa),
thì không Pod nào được gắn kết (bind) vào node.

Tính năng này phụ thuộc vào [PodGroup API](https://kubernetes.io/docs/concepts/workloads/podgroup-api/).
Hãy đảm bảo feature gate [`GenericWorkload`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#GenericWorkload)
và nhóm API (API group) `scheduling.k8s.io/v1alpha2`
đã được bật trong cluster.

## Cách hoạt động (How it works)

Khi plugin `GangScheduling` được bật, scheduler thay đổi vòng đời của các Pod thuộc về
một [PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/) có
[chính sách lập lịch (scheduling policy)](https://kubernetes.io/docs/concepts/workloads/workload-api/policies/) kiểu `gang`.
Quá trình này diễn ra theo các bước sau cho mỗi PodGroup:

1. Scheduler giữ các Pod ở pha `PreEnqueue` cho đến khi:
   * Đối tượng PodGroup được tham chiếu tồn tại.
   * Số lượng `Pod` đã được tạo cho `PodGroup` ít nhất bằng `minCount`.

   Các `Pod` không đi vào hàng đợi lập lịch hoạt động (active scheduling queue) cho đến khi cả hai điều kiện được thỏa mãn.

2. Khi đã đạt số lượng tối thiểu (quorum), scheduler cố gắng tìm vị trí sắp đặt cho tất cả các Pod trong nhóm.
   Nó tận dụng chu trình [lập lịch PodGroup](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/) để đưa ra một quyết định
   lập lịch duy nhất, có tính nguyên tử (atomic). Plugin `GangScheduling` triển khai một điểm mở rộng `Permit` được đánh giá cho mỗi
   Pod có thể lập lịch trong chu trình. Điểm mở rộng này được dùng để xác định liệu ràng buộc `minCount` có được thỏa mãn hay không,
   bằng cách so sánh số Pod đã được sắp đặt thành công với giá trị `minCount`.

3. Nếu scheduler tìm được vị trí sắp đặt hợp lệ cho ít nhất `minCount` Pod,
   nó cho phép những Pod đã được sắp đặt thành công đó được bind vào các node đã gán cho chúng.
   Nếu không tìm được đủ vị trí sắp đặt để thỏa mãn yêu cầu `minCount`, thì không Pod nào được lập lịch.
   Thay vào đó, chúng được chuyển sang hàng đợi không thể lập lịch (unschedulable queue) để chờ tài nguyên cluster được giải phóng,
   cho phép các workload khác được lập lịch trong thời gian chờ đó.

## Tiếp theo (What's next)

* Tìm hiểu về [PodGroup API](https://kubernetes.io/docs/concepts/workloads/podgroup-api/) và [vòng đời](https://kubernetes.io/docs/concepts/workloads/podgroup-api/lifecycle/) của nó.
* Đọc về [các chính sách lập lịch PodGroup](https://kubernetes.io/docs/concepts/workloads/workload-api/policies/).
* Đọc về [lập lịch PodGroup](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/).
