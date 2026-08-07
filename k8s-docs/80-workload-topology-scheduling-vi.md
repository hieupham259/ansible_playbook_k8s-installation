# Lập lịch workload nhận biết topology (Topology-Aware Workload Scheduling)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/workload-api/topology-aware-scheduling/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](LO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 10/15 · Kiểm chứng ở Lab 13 (tùy chọn, chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Bài này ở trạng thái `alpha` từ v1.36 — **API có thể đổi giữa các phiên
bản**. Cluster lab 3 VM trên VMware **không có miền topology thật** (không rack, không zone,
không nhãn node tương ứng), nên ràng buộc topology ở đây không kiểm chứng được. Đọc để hiểu
khái niệm.

Trong lộ trình có **hai bài cùng tên** "Lập lịch workload nhận biết topology". Bài này là mặt
**API** — bạn khai báo gì trong PodGroup. Mặt **thuật toán và plugin** nằm ở
[bài 153](153-topology-aware-scheduling-vi.md), đọc sau.

**Phải hiểu ở lần đọc này:**

- Mục tiêu của TAS: đặt **tất cả Pod của một PodGroup cùng một miền topology** (rack, zone) để
  giảm độ trễ giao tiếp giữa các Pod và tránh workload bị phân mảnh trên hạ tầng.
- Với chính sách `gang`: TAS **mô phỏng việc gán cả nhóm cùng lúc**, đảm bảo ít nhất `minCount`
  Pod cùng nằm vừa trong một miền trước khi cam kết tài nguyên; không có phương án khả thi thì
  **toàn bộ PodGroup** trở thành unschedulable.
- Với chính sách `basic`: hành vi **có thể không nhất quán**, vì scheduler chỉ quan sát được
  một tập con Pod khi bước vào chu kỳ lập lịch; cách giảm nhẹ là dùng **scheduling gate** để
  trì hoãn cho tới khi mọi Pod của nhóm đã có mặt trong hàng đợi.
- Quy tắc với Pod đến sau: scheduler **buộc Pod mới vào đúng miền topology nơi các Pod hiện có
  đang cư trú**; miền đó thiếu dung lượng thì Pod mới nằm pending — **kể cả khi điều đó khiến
  số Pod được lập lịch tụt xuống dưới `minCount`**.
- Cấu hình API: `schedulingConstraints.topology` với `key` là **một nhãn node**; scheduler
  thực thi nghiêm ngặt việc mọi Pod nằm trên node có **cùng chính xác giá trị** của nhãn đó.
  Tính đến v1.36 chỉ được **một** ràng buộc topology cho mỗi PodGroup, và **TAS không kích
  hoạt preemption**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Thuật toán lập lịch theo placement mà `schedulingConstraints` được diễn giải trong đó | là cơ chế scheduler | [bài 151](151-podgroup-scheduling-vi.md) |
| Các plugin TAS và cách chỉnh trọng số | thuộc cấu hình scheduler | [bài 153](153-topology-aware-scheduling-vi.md) |
| Scheduling gate | nền đã học | giai đoạn 7 — [bài 145](145-pod-scheduling-readiness-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

*Lập lịch nhận biết topology* (Topology-Aware Scheduling — TAS) là một tính năng của Workload API
giúp tối ưu việc sắp đặt (placement) các pod trong cluster.

TAS đảm bảo rằng tất cả các pod trong một PodGroup được đặt cùng nhau (co-located) vào một miền topology
(topology domain) cụ thể, chẳng hạn như một rack máy chủ hoặc một zone duy nhất. Điều này giảm thiểu
độ trễ giao tiếp giữa các pod và ngăn workload bị phân mảnh trên hạ tầng cluster.

## Lập lịch nhận biết topology với chính sách gang scheduling (Topology-aware scheduling with gang scheduling policy)

Khi áp dụng cho các PodGroup có chính sách lập lịch `gang`, TAS mô phỏng khả năng gán
(*placement*) toàn bộ nhóm pod cùng một lúc. Nó đảm bảo rằng ít nhất `minCount` pod
đã chỉ định có thể cùng nằm vừa trong cùng một miền topology trước khi cam kết tài nguyên.
Nếu không tìm thấy phương án sắp đặt khả thi nào, toàn bộ PodGroup trở thành không thể lập lịch (unschedulable).

Đây là cách tiếp cận được khuyến nghị cho các workload như huấn luyện AI và ML phân tán, vốn
yêu cầu nghiêm ngặt về khoảng cách gần để giảm thiểu độ trễ giao tiếp giữa các pod.

Nếu các pod mới được thêm vào PodGroup trong khi một số pod đã được lập lịch (ví dụ, khi các pod
bị tạo lại), scheduler sẽ buộc tất cả các pod mới đến phải nằm trên đúng miền topology
nơi các pod hiện có đang cư trú. Nếu miền cụ thể đó thiếu dung lượng
cho các pod mới, các pod này sẽ ở trạng thái pending — ngay cả khi điều đó có nghĩa là tại thời điểm này
có ít hơn `minCount` pod được lập lịch.

> **Ghi chú:**
> Tính đến v1.36, Lập lịch nhận biết topology không kích hoạt việc chiếm chỗ (preemption) đối với workload hay pod.
> Nếu không thể tìm thấy phương án sắp đặt khả thi nào mà không kích hoạt preemption, PodGroup trở thành không thể lập lịch.

## Lập lịch nhận biết topology với chính sách basic (Topology-aware scheduling with basic scheduling policy)

Việc dùng TAS với chính sách lập lịch `basic` có thể thể hiện hành vi không nhất quán. Scheduler có thể
chỉ quan sát được một tập con các pod khi bước vào chu kỳ lập lịch PodGroup — do đó tính khả thi
của việc sắp đặt chỉ được đánh giá cho các pod quan sát được, thay vì toàn bộ PodGroup. Để giảm nhẹ
một phần hạn chế này, bạn có thể dùng scheduling gate để trì hoãn việc lập lịch PodGroup cho đến khi
tất cả các pod trong PodGroup đều có mặt trong hàng đợi lập lịch.

Nếu không tìm thấy phương án sắp đặt khả thi cho toàn bộ PodGroup, có thể chỉ một tập con các pod được lập lịch,
và chúng được đảm bảo đáp ứng các ràng buộc lập lịch.

Nếu các pod mới được thêm vào PodGroup trong khi một số pod đã được lập lịch, scheduler sẽ hành xử
giống như trường hợp chính sách `gang` — buộc các pod mới vào cùng miền topology, trừ khi
không đủ dung lượng (trong trường hợp đó các pod mới sẽ ở trạng thái pending).

## Cấu hình API: các ràng buộc lập lịch (API configuration: scheduling constraints)

Mọi PodGroup (hoặc PodGroupTemplate) có thể tùy chọn khai báo trường `schedulingConstraints`,
trường này được diễn giải bởi [thuật toán lập lịch PodGroup dựa trên placement](https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/#placement-scheduling-algorithm).
Nếu các ràng buộc được định nghĩa trong PodGroupTemplate, chúng sẽ được sao chép sang các PodGroup tham chiếu đến template đó.

Tính đến Kubernetes v1.36, API hỗ trợ các ràng buộc topology.

> **Ghi chú:**
> Tính đến Kubernetes v1.36, bạn chỉ có thể chỉ định một ràng buộc topology duy nhất trong mỗi PodGroup.

### Ràng buộc topology (Topology constraint)

Để định nghĩa một ràng buộc topology cho PodGroup, bạn cần đặt một `key` tương ứng với
một nhãn (label) node của Kubernetes, đại diện cho miền topology đích (ví dụ, một rack hoặc một zone).
Scheduler thực thi nghiêm ngặt việc tất cả các pod trong PodGroup được đặt lên những node có
cùng chính xác giá trị cho nhãn đã chỉ định này.

Dưới đây là ví dụ về một PodGroup được cấu hình với ràng buộc topology:

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  name: example-podgroup
spec:
  schedulingPolicy:
    gang:
      minCount: 4
  schedulingConstraints:
    topology:
      - key: topology.example.com/rack
```

## Tiếp theo (What's next)

* Tìm hiểu về [các chính sách của pod group](./79-workload-policies-vi.md).
* Tìm hiểu về [các plugin liên quan đến Lập lịch nhận biết topology](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-aware-scheduling/)
* Đọc về thuật toán [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Một PodGroup dùng `gang` kèm ràng buộc topology không tìm được miền nào chứa đủ `minCount`
   Pod, nhưng cluster vẫn còn Pod ưu tiên thấp có thể bị đá đi. Scheduler có preempt để dọn
   chỗ không?
2. Vì sao dùng TAS với chính sách `basic` lại có thể cho kết quả không nhất quán, và bài gợi ý
   cách giảm nhẹ nào?
3. Vài Pod của một nhóm bị tạo lại trong khi các Pod còn lại đã chạy, nhưng miền topology cũ
   đã hết chỗ. Scheduler đặt Pod mới sang miền còn trống chứ?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Khối *Ghi chú* nói rõ: tính đến v1.36, **TAS không kích hoạt preemption đối với
   workload hay Pod**, và nếu không tìm được phương án sắp đặt khả thi mà không cần preemption
   thì **PodGroup trở thành unschedulable**. Trực giác "ưu tiên cao thì cứ đá ưu tiên thấp"
   không áp dụng ở đây — đó là cơ chế của một bài khác, không phải của TAS.
2. Vì scheduler **có thể chỉ quan sát được một tập con các Pod** khi bước vào chu kỳ lập lịch
   PodGroup, nên **tính khả thi của việc sắp đặt chỉ được đánh giá cho các Pod quan sát được**,
   không phải cho toàn bộ nhóm. Hệ quả: có thể chỉ một tập con Pod được lập lịch (dù các Pod
   đó vẫn được đảm bảo thỏa mãn ràng buộc). Cách giảm nhẹ mà bài đưa ra là **dùng scheduling
   gate để trì hoãn việc lập lịch PodGroup cho tới khi tất cả Pod của nhóm có mặt trong hàng
   đợi**. Với `gang` thì vấn đề này không đặt ra vì cả nhóm được mô phỏng cùng lúc.
3. **Không.** Scheduler **buộc tất cả Pod mới đến phải nằm trên đúng miền topology nơi các Pod
   hiện có đang cư trú**. Nếu miền đó thiếu dung lượng, các Pod mới **ở trạng thái pending** —
   và bài nhấn mạnh: kể cả khi điều đó có nghĩa là tại thời điểm này **có ít hơn `minCount` Pod
   được lập lịch**. Hành vi này giống nhau cho cả `gang` lẫn `basic`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
