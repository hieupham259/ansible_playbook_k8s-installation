# Gián đoạn và độ ưu tiên của Pod Group (Pod Group Disruption and Priority)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/workload-api/disruption-and-priority/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 8/15 · Kiểm chứng ở Lab 13 (tùy chọn, chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Bài này ở trạng thái `alpha` từ v1.36 — **API có thể đổi giữa các phiên
bản** — và cluster lab 3 VM trên VMware không bật API group cần thiết nên không dựng được tình
huống preemption theo nhóm. Đọc để hiểu khái niệm.

Bài rất ngắn nhưng chứa một **ranh giới dễ hiểu sai**, nằm ngay trong khối *Ghi chú* đầu mục
*Các loại chế độ gián đoạn*: tính đến 1.36, hai trường mà bài này giới thiệu **chỉ** có tác
dụng trong preemption nhận biết workload, không tác dụng trong pha lập lịch pod.

**Phải hiểu ở lần đọc này:**

- Hai chế độ gián đoạn: `Pod` (mặc định — coi mỗi Pod là thực thể riêng, gián đoạn từng Pod
  độc lập) và `PodGroup` (tất cả Pod trong nhóm phải bị gián đoạn cùng nhau).
- PodGroup dùng **đúng khái niệm PriorityClass** như Pod: admission controller đọc
  `priorityClassName` và điền giá trị số nguyên.
- Ba nhánh khi phân giải độ ưu tiên: không tìm thấy priority class → **PodGroup bị từ chối**;
  không đặt `priorityClassName` → tìm PriorityClass có `globalDefault: true`; không có cái nào
  → độ ưu tiên bằng **không**.
- Độ ưu tiên của PodGroup là giá trị **có tính quyết định cho tất cả Pod trong nhóm** trong sự
  kiện workload-aware preemption, kể cả khi từng Pod có độ ưu tiên khác nhau.
- Ranh giới: `priority` và `disruptionMode` của PodGroup **chỉ được workload-aware preemption
  tôn trọng**; trong giai đoạn lập lịch pod, scheduler không xét tới chúng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Thuật toán preemption nhận biết workload dùng hai trường này ra sao | là bài riêng ở cuối nhóm | [bài 152](152-workload-aware-preemption-vi.md) |
| PriorityClass, `globalDefault`, preemption mặc định của Pod | nền đã học | giai đoạn 7 — [bài 141](141-pod-priority-preemption-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

PodGroup có thể khai báo một chế độ gián đoạn (disruption mode). Chế độ này quy định cách
scheduler có thể làm gián đoạn một PodGroup đang chạy, ví dụ để nhường chỗ cho
một PodGroup có độ ưu tiên cao hơn. PodGroup cũng có một độ ưu tiên (priority),
ghi đè độ ưu tiên của từng pod riêng lẻ trong nhóm
đối với các sự kiện [chiếm chỗ nhận biết workload (workload-aware preemption)](152-workload-aware-preemption-vi.md).

## Các loại chế độ gián đoạn (Disruption mode types)

> **Ghi chú:**
> Tính đến 1.36, các trường `priority` hoặc `disruptionMode` của PodGroup chỉ được tôn trọng
> bởi [workload-aware preemption](152-workload-aware-preemption-vi.md).
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

PodGroup sử dụng cùng khái niệm [PriorityClass](141-pod-priority-preemption-vi.md#priorityclass) như các Pod đơn lẻ.
Sau khi bạn đã tạo một hoặc nhiều PriorityClass,
bạn có thể tạo một PodGroup chỉ định tên của một trong các PriorityClass đó trong spec của nó.
Admission controller về priority sử dụng trường `priorityClassName` và điền giá trị số nguyên của độ ưu tiên.
Nếu không tìm thấy priority class, PodGroup sẽ bị từ chối.
Khi `priorityClassName` không được đặt cho một PodGroup, Kubernetes sẽ tìm một giá trị mặc định (một PriorityClass có `globalDefault` được đặt là true).
Nếu không có PriorityClass nào có `globalDefault` được đặt là true, PodGroup không có `priorityClassName` sẽ có độ ưu tiên bằng không.

Độ ưu tiên của PodGroup là độ ưu tiên có tính quyết định (authoritative) cho tất cả các pod trong nhóm trong các sự kiện [workload-aware preemption](152-workload-aware-preemption-vi.md), ngay cả khi độ ưu tiên của từng pod riêng lẻ tạo nên PodGroup này khác nhau.

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

* Đọc về thuật toán [Workload-Aware Preemption](152-workload-aware-preemption-vi.md).
* Tìm hiểu về [Workload API](./77-workload-api-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Bạn gán `priorityClassName: high-priority` cho một PodGroup đang chờ. Nhóm đó có được
   scheduler ưu tiên xếp trước các nhóm khác trong hàng đợi không?
2. `disruptionMode: Pod` khác `disruptionMode: PodGroup` ở điểm nào, và cái nào là mặc định?
3. Một PodGroup có độ ưu tiên 1000000, nhưng các Pod bên trong lại có PriorityClass khác nhau.
   Trong một sự kiện workload-aware preemption, giá trị nào có tính quyết định?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Đây là bẫy chính của bài. Khối *Ghi chú* nói rõ: tính đến 1.36, các trường
   `priority` và `disruptionMode` của PodGroup **chỉ được workload-aware preemption tôn
   trọng**, còn **trong giai đoạn lập lịch pod, scheduler không xét đến chúng**. Nói cách khác,
   `priorityClassName` trên PodGroup quyết định *ai bị đá khi có tranh chấp*, không quyết định
   *ai được xếp trước trong hàng đợi*. Trực giác quen thuộc từ PriorityClass của Pod không áp
   dụng nguyên vẹn ở đây.
2. **`Pod` là mặc định**: scheduler coi tất cả Pod trong nhóm là những thực thể riêng biệt và
   được phép **làm gián đoạn độc lập một Pod đơn lẻ** trong PodGroup. **`PodGroup`** áp ngữ
   nghĩa **"tất cả hoặc không gì cả" cho việc gián đoạn**: tất cả Pod trong nhóm phải bị gián
   đoạn cùng nhau. Đây là mặt đối xứng của gang scheduling ở phía lập lịch.
3. **Độ ưu tiên của PodGroup**, tức 1000000. Bài nói thẳng: độ ưu tiên của PodGroup là độ ưu
   tiên **có tính quyết định (authoritative) cho tất cả các Pod trong nhóm** trong các sự kiện
   workload-aware preemption, **ngay cả khi độ ưu tiên của từng Pod khác nhau** — nó ghi đè độ
   ưu tiên riêng của từng Pod cho loại sự kiện đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
