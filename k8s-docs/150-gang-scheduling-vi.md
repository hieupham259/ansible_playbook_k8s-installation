# Lập lịch theo nhóm (Gang Scheduling)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 11/15 · Kiểm chứng ở [Lab 13](labs/LAB-13-DRA.md).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Bài này ở trạng thái `alpha` từ v1.35, cần feature gate
`GenericWorkload` và API group `scheduling.k8s.io/v1alpha2` — **API có thể đổi giữa các phiên
bản**. Cluster lab 3 VM trên VMware không bật hai thứ đó; đọc để hiểu khái niệm.

Bài rất ngắn nhưng là **hạt nhân của cả giai đoạn**: nó nêu đúng một nguyên tắc — hoặc chạy
hết cả nhóm, hoặc không chạy gì. Đây là thứ khiến huấn luyện phân tán chạy được trên
Kubernetes, nên nếu chỉ đọc một bài trong nhóm PodGroup thì đọc bài này.

**Phải hiểu ở lần đọc này:**

- Định nghĩa: một nhóm Pod được lập lịch theo nguyên tắc **all-or-nothing**; cluster không
  chứa nổi cả nhóm (hoặc số Pod tối thiểu đã định) thì **không Pod nào được bind vào node**.
- Điều kiện rời pha `PreEnqueue` gồm **hai** vế: đối tượng PodGroup được tham chiếu đã tồn
  tại, **và** số Pod **đã được tạo** cho PodGroup ít nhất bằng `minCount`. Trước đó Pod không
  vào hàng đợi lập lịch hoạt động.
- Khi đủ quorum, scheduler dùng **chu trình lập lịch PodGroup** để ra **một quyết định duy
  nhất, nguyên tử** cho cả nhóm, thay vì n quyết định rời rạc.
- Plugin `GangScheduling` cài một điểm mở rộng **`Permit`**, được đánh giá cho mỗi Pod lập lịch
  được trong chu trình, để so số Pod đã sắp đặt thành công với `minCount`.
- Khi không đủ chỗ: các Pod chuyển sang **hàng đợi không thể lập lịch**, và bài nêu rõ mục
  đích — **để workload khác được lập lịch trong thời gian chờ**, thay vì giữ chỗ vô ích.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chu trình lập lịch PodGroup và quyết định nguyên tử diễn ra thế nào | là bài kế tiếp | [bài 151](151-podgroup-scheduling-vi.md) |
| Các điểm mở rộng `PreEnqueue`, `Permit` của scheduling framework | nền đã học | giai đoạn 7 — [bài 147](147-scheduling-framework-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [alpha]`

Gang scheduling (lập lịch theo nhóm) đảm bảo một nhóm Pod được lập lịch theo nguyên tắc "tất cả hoặc không gì cả" (all-or-nothing).
Nếu cluster không thể chứa toàn bộ nhóm (hoặc một số lượng Pod tối thiểu được định nghĩa),
thì không Pod nào được gắn kết (bind) vào node.

Tính năng này phụ thuộc vào [PodGroup API](75-podgroup-api-vi.md).
Hãy đảm bảo feature gate [`GenericWorkload`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#GenericWorkload)
và nhóm API (API group) `scheduling.k8s.io/v1alpha2`
đã được bật trong cluster.

## Cách hoạt động (How it works)

Khi plugin `GangScheduling` được bật, scheduler thay đổi vòng đời của các Pod thuộc về
một [PodGroup](75-podgroup-api-vi.md) có
[chính sách lập lịch (scheduling policy)](79-workload-policies-vi.md) kiểu `gang`.
Quá trình này diễn ra theo các bước sau cho mỗi PodGroup:

1. Scheduler giữ các Pod ở pha `PreEnqueue` cho đến khi:
   * Đối tượng PodGroup được tham chiếu tồn tại.
   * Số lượng `Pod` đã được tạo cho `PodGroup` ít nhất bằng `minCount`.

   Các `Pod` không đi vào hàng đợi lập lịch hoạt động (active scheduling queue) cho đến khi cả hai điều kiện được thỏa mãn.

2. Khi đã đạt số lượng tối thiểu (quorum), scheduler cố gắng tìm vị trí sắp đặt cho tất cả các Pod trong nhóm.
   Nó tận dụng chu trình [lập lịch PodGroup](151-podgroup-scheduling-vi.md) để đưa ra một quyết định
   lập lịch duy nhất, có tính nguyên tử (atomic). Plugin `GangScheduling` triển khai một điểm mở rộng `Permit` được đánh giá cho mỗi
   Pod có thể lập lịch trong chu trình. Điểm mở rộng này được dùng để xác định liệu ràng buộc `minCount` có được thỏa mãn hay không,
   bằng cách so sánh số Pod đã được sắp đặt thành công với giá trị `minCount`.

3. Nếu scheduler tìm được vị trí sắp đặt hợp lệ cho ít nhất `minCount` Pod,
   nó cho phép những Pod đã được sắp đặt thành công đó được bind vào các node đã gán cho chúng.
   Nếu không tìm được đủ vị trí sắp đặt để thỏa mãn yêu cầu `minCount`, thì không Pod nào được lập lịch.
   Thay vào đó, chúng được chuyển sang hàng đợi không thể lập lịch (unschedulable queue) để chờ tài nguyên cluster được giải phóng,
   cho phép các workload khác được lập lịch trong thời gian chờ đó.

## Tiếp theo (What's next)

* Tìm hiểu về [PodGroup API](75-podgroup-api-vi.md) và [vòng đời](76-podgroup-lifecycle-vi.md) của nó.
* Đọc về [các chính sách lập lịch PodGroup](79-workload-policies-vi.md).
* Đọc về [lập lịch PodGroup](151-podgroup-scheduling-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Gang scheduling khác lập lịch từng Pod ở điểm nào? Vì sao cách lập lịch từng Pod lại gây
   rắc rối với một job huấn luyện phân tán cần 4 worker chạy đồng thời?
2. Hai điều kiện nào phải cùng đúng thì các Pod của một PodGroup mới rời pha `PreEnqueue`?
   Điều kiện thứ hai đếm Pod theo tiêu chí gì?
3. `minCount: 4` nhưng cluster chỉ còn chỗ cho 3 Pod. Ba Pod đã tìm được node có được bind
   không? Điều gì xảy ra với chúng, và vì sao bài coi đó là lựa chọn tốt?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Lập lịch từng Pod ra **n quyết định độc lập**; gang scheduling ra **một quyết định duy nhất,
   nguyên tử cho cả nhóm**, và nếu cluster không chứa nổi cả nhóm (hoặc số Pod tối thiểu đã
   định) thì **không Pod nào được bind vào node**. Với job huấn luyện phân tán, lập lịch từng
   Pod cho phép 3/4 worker khởi động và **chiếm giữ tài nguyên trong khi không thể tiến triển**
   — chúng chờ worker thứ tư mãi không có chỗ. Kết quả là tài nguyên bị giam mà công việc
   không chạy.
2. Thứ nhất: **đối tượng PodGroup được tham chiếu phải tồn tại**. Thứ hai: **số Pod đã được tạo
   cho PodGroup ít nhất bằng `minCount`**. Chú ý tiêu chí đếm — đây là **số Pod đã được tạo**,
   không phải số Pod đã lập lịch được. Đó là điều kiện quorum để bắt đầu; việc có đủ chỗ hay
   không mới được kiểm ở bước sau, qua điểm mở rộng `Permit`.
3. **Không Pod nào được bind.** Cả nhóm được chuyển sang **hàng đợi không thể lập lịch** để chờ
   tài nguyên cluster được giải phóng. Bài nêu rõ lợi ích: làm vậy **cho phép các workload khác
   được lập lịch trong thời gian chờ đó** — thay vì để ba Pod nửa vời giữ chỗ mà không sinh ra
   kết quả nào.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
