# Nhóm lập lịch (Scheduling Group)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/scheduling-group/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 4/15 · Kiểm chứng ở Lab 13 (tùy chọn, chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Bài này ở trạng thái `alpha` từ v1.35 và phụ thuộc feature gate
`GenericWorkload` — **API có thể đổi giữa các phiên bản**. Cluster lab 3 VM trên VMware chạy
cấu hình mặc định nên không bật gate này; đọc để hiểu khái niệm, không phải để thao tác.

Bài rất ngắn và là **cửa vào của cả nhóm PodGroup/Workload** (bài 4 đến bài 14 của giai đoạn).
Nó chỉ trả lời một câu hỏi: Pod nói với scheduler rằng nó thuộc về một nhóm bằng cách nào.
Mọi thứ về nhóm đó — chính sách, vòng đời, thuật toán — nằm ở các bài sau.

**Phải hiểu ở lần đọc này:**

- Trường `spec.schedulingGroup.podGroupName` liên kết Pod tới một PodGroup **trong cùng
  namespace**, theo tên.
- Trường này **bất biến**: đã đặt thì Pod không chuyển sang PodGroup khác được.
- Hành vi thực tế **không do Pod quyết định mà do chính sách của PodGroup**: `basic` thì mỗi
  Pod vẫn được lập lịch độc lập theo hành vi tiêu chuẩn, việc phân nhóm chỉ đóng vai trò như
  một label ở cấp nhóm; `gang` thì Pod đi vào vòng đời "tất cả hoặc không có gì" quanh
  `minCount`.
- Pod trỏ tới PodGroup **chưa tồn tại** thì nằm pending, và scheduler tự xem xét lại ngay khi
  PodGroup được tạo — đúng với cả hai chính sách, vì scheduler cần PodGroup mới biết chính sách.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Chi tiết hai chính sách `basic` và `gang` | ở đây mới chỉ là tên gọi | [bài 79](79-workload-policies-vi.md) |
| PodGroup trông ra sao và lấy chính sách từ đâu | chưa học đối tượng đó | [bài 75](75-podgroup-api-vi.md) và [bài 77](77-workload-api-vi.md) |
| Cơ chế "tất cả hoặc không có gì" chạy thế nào trong scheduler | là thuật toán, không phải API | [bài 150](150-gang-scheduling-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [alpha]`

Bạn có thể liên kết một `Pod` với một [PodGroup](75-podgroup-api-vi.md)
để cho biết `Pod` đó thuộc về một nhóm các `Pod` được lập lịch cùng nhau. Điều này cho phép
scheduler áp dụng các chính sách ở cấp nhóm, chẳng hạn như gang scheduling, thay vì
xử lý từng `Pod` một cách độc lập.

## Chỉ định nhóm lập lịch (Specifying a scheduling group)

Khi feature gate [`GenericWorkload`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#GenericWorkload)
được bật,
bạn có thể đặt trường `spec.schedulingGroup` trong manifest của `Pod`. Trường này thiết lập
một liên kết theo tên đến một đối tượng `PodGroup` cụ thể trong cùng namespace.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: worker-0
  namespace: some-ns
spec:
  schedulingGroup:
    podGroupName: training-worker-0
  containers:
  - name: ml-worker
    image: training:v1
```

Trường `schedulingGroup` là bất biến (immutable). Một khi đã được đặt, `Pod` không thể
được chuyển sang một `PodGroup` khác.

## Hành vi (Behavior)

Khi bạn đặt `spec.schedulingGroup`, scheduler sẽ tra cứu
[PodGroup](75-podgroup-api-vi.md) được tham chiếu và áp dụng
[chính sách lập lịch](79-workload-policies-vi.md) được định nghĩa trong đó:

* Nếu `PodGroup` dùng chính sách `basic`, mỗi `Pod` được lập lịch độc lập theo
  hành vi tiêu chuẩn của Kubernetes. Việc phân nhóm chỉ được dùng như một label ở cấp nhóm.
* Nếu `PodGroup` dùng chính sách `gang`, `Pod` sẽ đi vào vòng đời lập lịch kiểu
  "tất cả hoặc không có gì" (all-or-nothing). Scheduler cố gắng sắp đặt đồng thời ít nhất
  `minCount` `Pod` trong nhóm; không `Pod` nào được gắn (bind) vào node trừ khi
  đạt được số lượng tối thiểu này.

## Tham chiếu đến PodGroup chưa tồn tại (Missing PodGroup reference)

Nếu một `Pod` tham chiếu đến một `PodGroup` chưa tồn tại, `Pod` đó sẽ ở trạng thái pending.
Scheduler sẽ tự động xem xét lại `Pod` này ngay khi `PodGroup` được tạo.

Điều này áp dụng bất kể chính sách cuối cùng là `basic` hay `gang`,
vì scheduler cần có `PodGroup` để xác định được chính sách.

## Tiếp theo (What's next)

* Tìm hiểu về [PodGroup API](75-podgroup-api-vi.md) và vòng đời của nó.
* Đọc về [các chính sách lập lịch của PodGroup](79-workload-policies-vi.md).
* Hiểu về thuật toán [gang scheduling](150-gang-scheduling-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Đặt `spec.schedulingGroup.podGroupName` cho một Pod thì Pod đó có chắc chắn được lập lịch
   theo kiểu "tất cả hoặc không có gì" không?
2. Pod tham chiếu một PodGroup chưa tồn tại thì chuyện gì xảy ra, và vì sao hành vi đó giống
   nhau bất kể chính sách cuối cùng là `basic` hay `gang`?
3. Bài nhấn mạnh `schedulingGroup` là trường bất biến. Điều đó chặn đúng thao tác nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Trường này chỉ tạo một liên kết theo tên tới một PodGroup trong cùng namespace;
   **hành vi lập lịch do chính sách của PodGroup quyết định**. Nếu PodGroup dùng chính sách
   `basic`, mỗi Pod vẫn được lập lịch **độc lập theo hành vi tiêu chuẩn của Kubernetes** và
   việc phân nhóm chỉ được dùng như một label ở cấp nhóm. Chỉ khi PodGroup dùng chính sách
   `gang` thì Pod mới đi vào vòng đời all-or-nothing quanh `minCount`, và khi đó **không Pod
   nào được bind vào node trừ khi đạt số lượng tối thiểu**.
2. Pod **ở trạng thái pending**, và **scheduler tự động xem xét lại Pod ngay khi PodGroup được
   tạo** — không cần tạo lại Pod. Hành vi này áp dụng cho cả hai chính sách vì lý do rất trực
   tiếp: **scheduler cần có PodGroup thì mới xác định được chính sách**. Không có đối tượng
   đó, nó không biết nên xử lý Pod theo `basic` hay `gang`.
3. Nó chặn việc **chuyển một Pod sang PodGroup khác**. Một khi `spec.schedulingGroup` đã được
   đặt, liên kết nhóm của Pod là cố định trong suốt vòng đời của Pod đó.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
