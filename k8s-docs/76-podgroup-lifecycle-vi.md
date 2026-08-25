# Vòng đời của PodGroup (PodGroup Lifecycle)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/podgroup-api/lifecycle/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 6/15 · Kiểm chứng ở [Lab 13](labs/LAB-13-DRA.md).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Bài này ở trạng thái `alpha` từ v1.36 — **API có thể đổi giữa các phiên
bản** — và cluster lab 3 VM trên VMware không bật API group cần thiết, nên không dựng lại được
các tình huống mô tả ở đây. Đọc để hiểu khái niệm.

Bài ngắn, đọc tiếp ngay sau [bài 75](75-podgroup-api-vi.md). Phần đáng nhớ nhất không phải
phần vòng đời mà là mục *Giới hạn* ở cuối: năm ràng buộc quyết định thứ gì sửa được và thứ gì
phải tạo lại.

**Phải hiểu ở lần đọc này:**

- PodGroup thuộc sở hữu của workload controller đã tạo nó qua `ownerReferences` tiêu chuẩn,
  nên bị **thu gom tự động** khi đối tượng sở hữu bị xóa.
- **Thứ tự tạo bắt buộc** là Workload → PodGroup → Pod. API server **từ chối** PodGroup có
  `podGroupTemplateRef` trỏ tới Workload không tồn tại hoặc đang bị xóa; còn Pod trỏ tới
  PodGroup chưa có thì chỉ nằm pending và được đưa vào hàng đợi khi PodGroup xuất hiện.
- **Bảo vệ khi xóa**: một finalizer chặn việc xóa PodGroup cho tới khi mọi Pod tham chiếu nó
  đạt pha kết thúc (`Succeeded` hoặc `Failed`).
- Hai kiểu quản lý: **do controller quản lý** (Job tự đặt `podGroupName` cho từng Pod, tương
  tự cách DaemonSet đặt node affinity) và **do người dùng quản lý** (bạn tự tạo PodGroup và tự
  đặt tên trong Pod template).
- Năm mục trong *Giới hạn*, đặc biệt: mọi Pod trong nhóm phải cùng `.spec.schedulerName` — nếu
  lệch thì **cả nhóm** bị từ chối; `minCount` và `spec.schedulingGroup` đều bất biến; tối đa 8
  PodGroupTemplate trong một Workload.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Vì sao Job là workload controller duy nhất tạo PodGroup | cần cấu trúc Workload | [bài 77](77-workload-api-vi.md) |
| Nội dung thật của `basic` và `gang` | ở đây chỉ là tên chính sách | [bài 79](79-workload-policies-vi.md) |
| Ai sinh ra condition `PodGroupScheduled` và khi nào | thuộc chu trình lập lịch | [bài 151](151-podgroup-scheduling-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Một [PodGroup](75-podgroup-api-vi.md) được lập lịch
như một đơn vị và được bảo vệ khỏi việc bị xóa sớm trong khi các Pod của nó vẫn đang chạy.

## Quyền sở hữu và vòng đời (Ownership and lifecycle)

Các `PodGroup` được sở hữu bởi workload controller đã tạo ra chúng (ví dụ, một Job)
thông qua cơ chế `ownerReferences` tiêu chuẩn. Khi đối tượng sở hữu bị xóa, các `PodGroup`
sẽ tự động được thu gom (garbage collected).

Tên của `PodGroup` phải là duy nhất trong một namespace và phải là
[DNS subdomain](17-names-vi.md#dns-subdomain-names) hợp lệ.

## Thứ tự tạo (Creation ordering)

Các controller phải tạo các đối tượng theo thứ tự sau:

1. `Workload` — mẫu chính sách lập lịch.
2. `PodGroup` — thực thể (instance) runtime.
3. `Pods` — với `spec.schedulingGroup.podGroupName` trỏ đến `PodGroup`.

Nếu một `PodGroup` chứa `podGroupTemplateRef` trỏ đến một `Workload` không
tồn tại (hoặc đang bị xóa), API server sẽ từ chối yêu cầu tạo `PodGroup` đó.
`Workload` được tham chiếu phải tồn tại trước thì `PodGroup` mới có thể được tạo.

Nếu một `Pod` tham chiếu đến một `PodGroup` chưa tồn tại, `Pod` đó sẽ ở trạng thái pending.
Scheduler sẽ tự động đưa `Pod` vào hàng đợi lập lịch ngay khi `PodGroup` được tạo.

## Bảo vệ khi xóa (Deletion protection)

Một `PodGroup` không thể bị xóa hoàn toàn trong khi bất kỳ Pod nào của nó vẫn đang chạy.
Một finalizer chuyên dụng bảo đảm rằng việc xóa bị chặn cho đến khi tất cả các `Pod`
tham chiếu đến `PodGroup` đã đạt đến pha kết thúc (`Succeeded` hoặc `Failed`).

## PodGroup do controller quản lý và do người dùng quản lý (Controller-managed and user-managed PodGroups)

Trong hầu hết các trường hợp, các workload controller (ví dụ, Job) tự động tạo các `PodGroup`
(controller-managed — do controller quản lý). Controller xác định `podGroupName` cho mỗi Pod
tại thời điểm tạo, tương tự cách một `DaemonSet` đặt node affinity cho từng Pod.

Nếu bạn cần kiểm soát nhiều hơn việc đặt tên và vòng đời, bạn có thể tạo trực tiếp các đối tượng `PodGroup` và tự đặt
`spec.schedulingGroup.podGroupName` trong các Pod template của mình
(user-managed — do người dùng quản lý). Cách này cho bạn toàn quyền kiểm soát việc tạo và đặt tên `PodGroup`.

## Giới hạn (Limitations) {#limitations}

* Tất cả các Pod trong một `PodGroup` phải dùng cùng một `.spec.schedulerName`.
  Nếu phát hiện sự không khớp, scheduler sẽ từ chối tất cả các Pod trong nhóm với trạng thái không thể lập lịch (unschedulable).
* Trường `spec.schedulingPolicy.gang.minCount` trên một PodGroup là bất biến (immutable).
  Một khi đã tạo, bạn không thể thay đổi số lượng Pod tối thiểu phải có thể lập lịch được để nhóm được chấp nhận (admit).
* Trường `spec.schedulingGroup` trên một Pod là bất biến.
  Một khi đã được đặt, một Pod không thể chuyển sang một PodGroup khác.
* Số lượng `PodGroupTemplate` tối đa trong một `Workload` là 8.
* Condition `PodGroupScheduled` chỉ phản ánh kết quả của lần thử lập lịch
  ban đầu. Một khi condition này được đặt thành `True`, scheduler sẽ không cập nhật nó
  nếu sau đó các Pod bị lỗi, bị trục xuất (evict), hoặc ngừng chạy.

## Tiếp theo (What's next)

* Tìm hiểu tổng quan và cấu trúc của [PodGroup API](75-podgroup-api-vi.md).
* Tìm hiểu về [Workload API](77-workload-api-vi.md) — API cung cấp các `PodGroupTemplate`.
* Xem cách các Pod tham chiếu đến PodGroup của chúng thông qua trường [scheduling group](59-scheduling-group-vi.md).
* Hiểu về thuật toán [gang scheduling](150-gang-scheduling-vi.md).
* Đọc [các chính sách lập lịch của PodGroup](79-workload-policies-vi.md) để biết chi tiết về `basic` và `gang`.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Một PodGroup có 4 Pod, trong đó 3 Pod dùng scheduler mặc định còn 1 Pod đặt
   `.spec.schedulerName` khác. Chuyện gì xảy ra — chỉ Pod lệch bị treo, hay nhiều hơn thế?
2. Bạn `kubectl delete podgroup` trong khi các Pod của nó vẫn đang chạy. Đối tượng biến mất
   ngay không, và cơ chế nào quyết định điều đó?
3. Một PodGroup đang tồn tại có `minCount: 4`. Bạn muốn đổi thành 6 vì job cần thêm worker.
   Làm được bằng `kubectl edit` không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Cả nhóm hỏng, không chỉ Pod lệch.** Mục *Giới hạn* nói rõ: tất cả Pod trong một PodGroup
   phải dùng cùng một `.spec.schedulerName`, và khi phát hiện không khớp, **scheduler từ chối
   tất cả các Pod trong nhóm** với trạng thái unschedulable. Trực giác "một Pod cấu hình sai
   thì chỉ Pod đó chịu" sai ở đây vì PodGroup là một đơn vị lập lịch: không thể có hai
   scheduler cùng ra quyết định cho một quyết định chung.
2. **Không biến mất ngay.** Một **finalizer chuyên dụng** chặn việc xóa cho tới khi **tất cả
   các Pod tham chiếu PodGroup đã đạt pha kết thúc** — `Succeeded` hoặc `Failed`. Đây là "bảo
   vệ khi xóa": PodGroup không thể bị xóa hoàn toàn trong khi Pod của nó còn chạy. Lưu ý đây
   là cơ chế khác với `ownerReferences`: cái sau lo việc thu gom PodGroup khi *đối tượng sở
   hữu* (ví dụ Job) bị xóa.
3. **Không.** `spec.schedulingPolicy.gang.minCount` là **bất biến**: một khi đã tạo, bạn không
   thể thay đổi số Pod tối thiểu phải lập lịch được để nhóm được chấp nhận. Muốn con số khác
   thì phải có một PodGroup khác. Cùng họ với ràng buộc này là `spec.schedulingGroup` của Pod
   — cũng bất biến, nên Pod không chuyển nhóm được.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
