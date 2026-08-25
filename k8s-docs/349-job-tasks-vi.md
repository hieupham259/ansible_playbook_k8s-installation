# Chạy Job (Run Jobs)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/job/>
>
> Chạy Job bằng xử lý song song (parallel processing).

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 4 — Workload controller](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller)
→ [4b. StatefulSet, DaemonSet, Job và autoscaling](00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling),
bài 3/7 của dòng **Thực hành** · Kiểm chứng ở
[Lab 4b — StatefulSet, DaemonSet và Job](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md),
**phần B6**: lab dùng trang này như bảng chọn và chạy ba mẫu mà nó liệt kê — Indexed Job ở
**B6.3**, CronJob ở **B9**, khai triển template ở **B10**.

Đây là **trang mục lục**, đọc trong vài phút. Không có bước nào để làm theo.

**Phải hiểu ở lần đọc này:**

- Toàn bộ nội dung trang là **một danh sách link** tới các trang con của mục tác vụ *Run Jobs* —
  không có thao tác nào để thực hiện.
- Danh sách đó gom đúng một chủ đề: **chạy Job bằng xử lý song song**. Nó gồm CronJob theo lịch
  ([350](350-automated-tasks-cron-jobs-vi.md)), hàng đợi công việc thô
  ([351](351-coarse-parallel-work-queue-vi.md)) và mịn ([352](352-fine-parallel-work-queue-vi.md)),
  Indexed Job ([353](353-indexed-parallel-processing-vi.md)), Job có giao tiếp Pod-với-Pod
  ([354](354-job-pod-to-pod-communication-vi.md)), khai triển template
  ([355](355-parallel-processing-expansion-vi.md)) và Pod failure policy
  ([383](383-pod-failure-policy-vi.md)).
- Dùng nó như **bảng chọn mẫu**: gặp một bài toán chạy theo lô thì mở đúng trang mẫu tương ứng,
  thay vì tự chế cách tổ chức Job.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Link *xử lý song song mịn với hàng đợi công việc* (bài [352](352-fine-parallel-work-queue-vi.md)) | [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) xếp bài này vào nhóm không kiểm chứng được trên cluster baseline: nó cần một Service Redis và một image worker tự build | dòng **Thực hành** của [giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) |
| Link *Job với giao tiếp Pod-với-Pod* (bài [354](354-job-pod-to-pod-communication-vi.md)) | lộ trình xếp bài này ở giai đoạn sau, sau khi đã học Service | dòng **Thực hành** của [giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng) |
| Link *xử lý các lỗi Pod có thể và không thể thử lại bằng Pod failure policy* (bài [383](383-pod-failure-policy-vi.md)) | lộ trình xếp mục Pod failure policy của bài [67](67-job-vi.md) là đọc lướt ở lần đọc đầu | [giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng](00-ALO-TRINH-ADMIN.md#giai-đoạn-29--daemonset-job-nâng-cao-và-thiết-bị-chuyên-dụng) |

---

Đây là trang mục lục của mục tác vụ (task) *Run Jobs* trong tài liệu Kubernetes: các tác vụ
hướng dẫn chạy Job bằng xử lý song song.

---

Các trang trong mục này:

* [Chạy các tác vụ tự động với CronJob (Running Automated Tasks with a CronJob)](./350-automated-tasks-cron-jobs-vi.md)

* [Xử lý song song thô với hàng đợi công việc (Coarse Parallel Processing Using a Work Queue)](351-coarse-parallel-work-queue-vi.md)

* [Xử lý song song mịn với hàng đợi công việc (Fine Parallel Processing Using a Work Queue)](352-fine-parallel-work-queue-vi.md)

* [Indexed Job cho xử lý song song với phân công công việc tĩnh (Indexed Job for Parallel Processing with Static Work Assignment)](353-indexed-parallel-processing-vi.md)

* [Job với giao tiếp Pod-với-Pod (Job with Pod-to-Pod Communication)](354-job-pod-to-pod-communication-vi.md)

* [Xử lý song song bằng cách khai triển template (Parallel Processing using Expansions)](355-parallel-processing-expansion-vi.md)

* [Xử lý các lỗi Pod có thể và không thể thử lại bằng Pod failure policy (Handling retriable and non-retriable pod failures with Pod failure policy)](383-pod-failure-policy-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở nhóm 4b:

1. Trang này cho bạn thứ mà bài [67](67-job-vi.md) không cho. Đó là thứ gì? Trả lời bằng vai trò
   của trang, không bằng nội dung Job.
2. **Câu bẫy.** Trang nằm trong nhánh `/docs/tasks/`, tức nhánh thực hành. Vậy bạn làm theo nó
   trên cluster lab bằng cách nào?
3. Trong danh sách trang con, những trang nào là các mẫu mà nhóm 4b thực sự chạy trên cluster lab,
   và những trang nào lộ trình để lại cho giai đoạn sau?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Nó cho **bảng chọn mẫu**: một danh sách các cách tổ chức Job cho xử lý song song, mỗi cách một
   trang riêng. Bài [67](67-job-vi.md) dạy bản thân đối tượng Job; trang này chỉ trả lời câu hỏi
   **bài toán của tôi hợp với mẫu nào**, và dẫn bạn tới đúng trang mẫu đó.
2. **Không làm theo được, vì nó không có bước nào.** Đây là chỗ dễ hụt hẫng: cùng nằm trong nhánh
   thực hành nhưng **toàn bộ nội dung là danh sách link**. Muốn thực hành thì mở một trang con, ví
   dụ [350](350-automated-tasks-cron-jobs-vi.md), [353](353-indexed-parallel-processing-vi.md) hoặc
   [355](355-parallel-processing-expansion-vi.md).
3. Chạy thật ở nhóm 4b: **CronJob ([350](350-automated-tasks-cron-jobs-vi.md))**, **Indexed Job
   ([353](353-indexed-parallel-processing-vi.md))** và **khai triển template
   ([355](355-parallel-processing-expansion-vi.md))** — tương ứng các phần B9, B6.3 và B10 của
   [Lab 4b](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md). Để lại giai đoạn sau: **hàng đợi công
   việc mịn ([352](352-fine-parallel-work-queue-vi.md))** và **Job giao tiếp Pod-với-Pod
   ([354](354-job-pod-to-pod-communication-vi.md))** ở giai đoạn 5, **Pod failure policy
   ([383](383-pod-failure-policy-vi.md))** ở giai đoạn 29. Còn **hàng đợi công việc thô
   ([351](351-coarse-parallel-work-queue-vi.md))** thì đọc ở nhóm 4b để nắm mẫu, nhưng lab chỉ
   dựng được **hình dạng** của nó ở phần B6.4.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
