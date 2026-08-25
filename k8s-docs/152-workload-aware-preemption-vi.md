# Preemption nhận biết workload (Workload-Aware Preemption)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/workload-aware-preemption/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 13/15 · Kiểm chứng ở [Lab 13](labs/LAB-13-DRA.md).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Bài này ở trạng thái `alpha` từ v1.36 và cần **cả hai** feature gate
`GenericWorkload` và `GangScheduling` cùng API group `scheduling.k8s.io/v1alpha2` — **API có
thể đổi giữa các phiên bản**. Cluster lab 3 VM trên VMware không bật những thứ đó; đọc để hiểu
khái niệm.

Bài rất ngắn và chỉ có nghĩa nếu bạn đã nắm preemption mặc định của Pod ở giai đoạn 7. Cách
đọc hiệu quả nhất là **đặt cạnh nhau**: cùng một nguyên tắc, ba khác biệt.

**Phải hiểu ở lần đọc này:**

- Phạm vi áp dụng: cơ chế này **chỉ được dùng trong quá trình lập lịch PodGroup** và **thay
  thế** preemption mặc định cho các Pod thuộc PodGroup đó.
- Khác biệt 1 — **đơn vị preemptor**: scheduler coi cả PodGroup là **một bên chiếm chỗ duy
  nhất**, thay vì đánh giá từng Pod của nhóm một cách cô lập.
- Khác biệt 2 — **miền toàn cluster**: thay vì xét preemption theo từng node, scheduler xét cả
  cluster như một miền và chọn tập nạn nhân **trải trên nhiều node**.
- Khác biệt 3 — **thứ bậc tầm quan trọng của nạn nhân**, theo đúng thứ tự: độ ưu tiên → loại
  workload (PodGroup quan trọng hơn Pod đơn lẻ **cùng độ ưu tiên**) → kích thước nhóm (nhóm
  nhiều thành viên hơn thì quan trọng hơn) → thời điểm khởi động (khởi động sớm hơn thì quan
  trọng hơn).
- Ranh giới ở khối *Ghi chú*: khi lập lịch **một Pod đơn lẻ**, preemption mặc định được áp
  dụng, và tính đến 1.36 nó **không tôn trọng** `priority` hay `disruptionMode` của PodGroup mà
  Pod nạn nhân thuộc về.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Preemption mặc định: PriorityClass, chọn nạn nhân theo từng node | là nền của bài này | giai đoạn 7 — [bài 141](141-pod-priority-preemption-vi.md) |
| Cách bật các feature gate và API group alpha trong cluster thật | lab không chạy được | khi công việc thực sự cần |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Preemption (chiếm chỗ) nhận biết workload giới thiệu một cơ chế preemption được thiết kế riêng cho các PodGroup.
Khi một PodGroup không thể được lập lịch, scheduler sử dụng một logic preemption cố gắng
làm cho việc lập lịch PodGroup này trở nên khả thi. Cách tiếp cận này chỉ được dùng trong quá trình lập lịch PodGroup
và thay thế cơ chế preemption mặc định cho các pod thuộc một PodGroup nhất định.

Khi tính năng này được bật, scheduler coi PodGroup như một đơn vị preemptor (bên chiếm chỗ) duy nhất,
thay vì đánh giá từng pod riêng lẻ của PodGroup một cách cô lập. Để nhường chỗ cho các pod đang chờ trong nhóm,
nó tìm kiếm các nạn nhân (victim) trên toàn bộ cluster,
và biết cách đối xử cũng như preempt các PodGroup khác với vai trò nạn nhân theo các chế độ gián đoạn (disruption mode) của chúng.

Tính năng này phụ thuộc vào [Gang Scheduling](150-gang-scheduling-vi.md)
và [Workload API](77-workload-api-vi.md).
Hãy đảm bảo các feature gate [`GenericWorkload`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#GenericWorkload)
và [`GangScheduling`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#GangScheduling)
cùng nhóm API (API group) `scheduling.k8s.io/v1alpha2` đã được bật trong cluster.

## Cách hoạt động (How it works)

Quá trình preemption nhận biết workload tuân theo cùng các nguyên tắc
như [preemption mặc định](141-pod-priority-preemption-vi.md#preemption)
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
   [độ ưu tiên và chế độ gián đoạn](78-workload-disruption-priority-vi.md) cụ thể của một PodGroup
   để đánh giá liệu các pod của nó có thể bị preempt hay không và bị preempt như thế nào trong các sự kiện preemption.

> **Ghi chú:**
> Khi lập lịch một Pod đơn lẻ, cơ chế preemption mặc định cho pod được áp dụng.
> Tính đến 1.36, khi scheduler thực hiện preemption mặc định cho một Pod đơn lẻ
> và nó cố gắng preempt một Pod thuộc về một PodGroup, nó **không**
> tôn trọng các trường `priority` hoặc `disruptionMode` của PodGroup đó.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [Độ ưu tiên và Sự gián đoạn của PodGroup](78-workload-disruption-priority-vi.md).
* Tìm hiểu về [Workload API](77-workload-api-vi.md).
* Đọc thêm về [Gang scheduling](150-gang-scheduling-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Preemption nhận biết workload khác preemption mặc định ở hai điểm cơ bản nào — ai là bên
   chiếm chỗ, và nạn nhân được tìm ở phạm vi nào?
2. Hai PodGroup cùng độ ưu tiên, một nhóm 8 Pod khởi động lúc 10:00, một nhóm 4 Pod khởi động
   lúc 09:00. Scheduler coi nhóm nào quan trọng hơn, và nó xét theo thứ tự tiêu chí nào?
3. Một Pod đơn lẻ có độ ưu tiên rất cao cần chỗ, và ứng viên nạn nhân là một Pod thuộc PodGroup
   đã đặt `disruptionMode: PodGroup`. Scheduler có tôn trọng chế độ đó không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Bên chiếm chỗ là cả PodGroup, không phải từng Pod**: scheduler coi PodGroup như một đơn vị
   preemptor duy nhất thay vì đánh giá từng Pod của nhóm một cách cô lập. **Nạn nhân được tìm
   trên toàn cluster**: thay vì đánh giá preemption theo từng node một, scheduler xem cả cluster
   như một miền duy nhất và chọn một tập nạn nhân **trải trên nhiều node** sao cho đủ chỗ cho
   nhóm preemptor. Điều này hợp lý vì nhóm cần chỗ ở nhiều node cùng lúc, nên dọn chỗ trên một
   node không giải quyết được gì.
2. **Nhóm 8 Pod quan trọng hơn**, dù nó khởi động muộn hơn. Thứ bậc được xét theo đúng thứ tự:
   (1) **độ ưu tiên** — ở đây bằng nhau nên chưa phân định; (2) **loại workload** — cả hai đều
   là PodGroup nên vẫn hòa; (3) **kích thước nhóm** — nhóm nhiều thành viên hơn được coi là
   quan trọng hơn, và tiêu chí này đã phân định xong; (4) **thời điểm khởi động** — không được
   xét tới. Nếu bạn trả lời theo "chạy trước thì quý hơn", hãy nhớ `start time` là tiêu chí
   **cuối cùng**, chỉ dùng khi cả ba tiêu chí trước đều hòa.
3. **Không.** Khối *Ghi chú* nói rõ: khi lập lịch một Pod đơn lẻ, **cơ chế preemption mặc định
   cho Pod được áp dụng**, và tính đến 1.36, khi nó preempt một Pod thuộc về một PodGroup, nó
   **không tôn trọng** các trường `priority` hay `disruptionMode` của PodGroup đó. Nói cách
   khác, `disruptionMode: PodGroup` chỉ có hiệu lực khi bên chiếm chỗ cũng đi qua đường
   workload-aware preemption; nó không phải một lớp bảo vệ tuyệt đối cho cả nhóm.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
