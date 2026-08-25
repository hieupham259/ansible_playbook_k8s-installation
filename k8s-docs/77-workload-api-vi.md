# Workload API

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/workload-api/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 7/15 · Kiểm chứng ở [Lab 13](labs/LAB-13-DRA.md).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Bài này ở trạng thái `alpha` từ v1.36, thuộc API group
`scheduling.k8s.io/v1alpha2` và cần feature gate `GenericWorkload` — **API có thể đổi giữa các
phiên bản**. Cluster lab 3 VM trên VMware không bật những thứ đó; đọc để hiểu khái niệm.

Đừng nhầm `Workload` (danh từ riêng, một kind API) với "workload" nghĩa chung mà bạn đã dùng
suốt từ giai đoạn 4. Đây là một đối tượng cụ thể, và nó **không** quản lý Pod — việc đó vẫn
thuộc về các workload controller như Job.

**Phải hiểu ở lần đọc này:**

- Workload là **mẫu chính sách tĩnh, tồn tại lâu dài**: nó chỉ định các nhóm Pod cần được lập
  lịch ra sao, và **không** theo dõi trạng thái runtime.
- Toàn bộ `spec` của Workload **bất biến** sau khi tạo: không sửa, không thêm, không xóa mục
  trong `podGroupTemplates`.
- Mỗi mục `podGroupTemplates` phải có `name` duy nhất (để `spec.podGroupTemplateRef` của
  PodGroup trỏ tới) và một chính sách lập lịch; tối đa **8** template trong một Workload.
- `controllerRef` chỉ phục vụ khả năng quan sát và công cụ hỗ trợ — bài nói thẳng: **dữ liệu
  này không được dùng để lập lịch**.
- Con đường tích hợp sẵn ở mục *Gang scheduling với Job*: bật `WorkloadWithJob`, Job dạng
  indexed chạy song song có `.spec.parallelism` bằng `.spec.completions` thì Job controller tự
  tạo Workload và PodGroup, với `minCount` đặt bằng parallelism.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `priorityClassName` và `disruptionMode` trong template | phụ thuộc gate `WorkloadAwarePreemption` | [bài 78](78-workload-disruption-priority-vi.md) |
| Nội dung thật của `basic` và `gang` | ở đây mới là tên chính sách | [bài 79](79-workload-policies-vi.md) |
| Ràng buộc topology cho nhóm | là một mặt riêng của cùng API | [bài 80](80-workload-topology-scheduling-vi.md) |
| Thuật toán khiến gang scheduling hoạt động | thuộc scheduler, không phải API | [bài 150](150-gang-scheduling-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Tài nguyên API `Workload` định nghĩa các yêu cầu lập lịch (scheduling) và cấu trúc của một
ứng dụng gồm nhiều Pod. Trong khi các workload controller như [Job](67-job-vi.md)
quản lý trạng thái lúc chạy (runtime state) của ứng dụng, `Workload` chỉ định cách các nhóm `Pod`
cần được lập lịch. Job controller là controller tích hợp sẵn duy nhất tạo ra các đối tượng
[PodGroup](75-podgroup-api-vi.md) từ
`PodGroupTemplates` của `Workload` tại thời điểm chạy.

## Workload là gì? (What is a Workload?)

Tài nguyên API Workload thuộc nhóm API (API group) `scheduling.k8s.io/v1alpha2`,
và cluster của bạn phải bật API group đó, cũng như bật
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`GenericWorkload`, trước khi bạn có thể sử dụng API này.

`Workload` là một mẫu chính sách (policy template) tĩnh, tồn tại lâu dài. Nó định nghĩa
những chính sách lập lịch nào cần được áp dụng cho các nhóm Pod, nhưng bản thân nó không theo dõi
trạng thái lúc chạy. Trạng thái lập lịch lúc chạy được duy trì bởi các đối tượng
[PodGroup](75-podgroup-api-vi.md),
do các controller tạo ra từ `PodGroupTemplates` của `Workload`.

## Cấu trúc API (API structure)

Một `Workload` gồm hai trường: một danh sách `PodGroupTemplates` và một tham chiếu controller
(tùy chọn). Toàn bộ spec của `Workload` là bất biến (immutable) sau khi tạo: bạn không thể
sửa đổi các template hiện có, thêm template mới, hay xóa template khỏi `podGroupTemplates`.

### PodGroupTemplates

Danh sách `spec.podGroupTemplates` định nghĩa các thành phần riêng biệt của workload.
Ví dụ, một job học máy (machine learning) có thể có một template `driver` và một template `worker`.

Mỗi mục trong `podGroupTemplates` phải có:
1. Một `name` duy nhất, được dùng để tham chiếu đến template trong `spec.podGroupTemplateRef` của `PodGroup`.
2. Một [chính sách lập lịch](./79-workload-policies-vi.md) (`basic` hoặc `gang`).

Nếu [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) [`WorkloadAwarePreemption`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#WorkloadAwarePreemption) được bật, mỗi mục trong `podGroups` cũng có thể có [độ ưu tiên và chế độ gián đoạn](./78-workload-disruption-priority-vi.md).

Số lượng PodGroupTemplate tối đa trong một Workload là 8.

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: Workload
metadata:
  name: training-job-workload
  namespace: some-ns
spec:
  controllerRef:
    apiGroup: batch
    kind: Job
    name: training-job
  podGroupTemplates:
  - name: workers
    schedulingPolicy:
      gang:
        # Gang chỉ có thể được lập lịch nếu 4 pod chạy được cùng lúc
        minCount: 4
    priorityClassName: high-priority # Chỉ áp dụng khi bật feature gate WorkloadAwarePreemption
    disruptionMode: PodGroup # Chỉ áp dụng khi bật feature gate WorkloadAwarePreemption
```

Khi một workload controller tạo `PodGroup` từ một trong các template này, nó sao chép
`schedulingPolicy` vào spec riêng của `PodGroup`. Các thay đổi đối với `Workload` chỉ ảnh hưởng
đến những `PodGroup` mới được tạo, không ảnh hưởng đến các `PodGroup` hiện có.

### Tham chiếu đến đối tượng điều khiển workload (Referencing a workload controlling object)

Trường `controllerRef` liên kết Workload trở lại đối tượng cấp cao cụ thể định nghĩa ứng dụng,
chẳng hạn như một [Job](67-job-vi.md) hoặc một CRD tùy chỉnh.
Điều này hữu ích cho khả năng quan sát (observability) và các công cụ hỗ trợ.
Dữ liệu này không được dùng để lập lịch hay quản lý Workload.

## Gang scheduling với Job (Gang scheduling with Jobs)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Khi feature gate
[`WorkloadWithJob`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
được bật, [Job](67-job-vi.md) controller
tự động tạo các đối tượng Workload và PodGroup cho các Job dạng indexed chạy song song có
`.spec.parallelism` bằng `.spec.completions`. `minCount` của chính sách gang
được đặt bằng parallelism của Job, do đó tất cả các Pod phải có thể được lập lịch cùng nhau
trước khi bất kỳ Pod nào trong số đó được gán (bind) vào node.

Đây là con đường tích hợp sẵn để sử dụng gang scheduling với Job.
Bạn không cần tự tạo các đối tượng Workload hay PodGroup vì Job controller
xử lý việc này một cách tự động. Các workload controller khác (chẳng hạn
JobSet) có thể tự quản lý các đối tượng Workload và PodGroup của riêng chúng một cách độc lập.

## Tiếp theo (What's next)

* Tìm hiểu về [các chính sách lập lịch PodGroup](./79-workload-policies-vi.md).
* Xem cách các PodGroup được tạo từ Workload trong phần tổng quan [PodGroup API](75-podgroup-api-vi.md).
* Đọc về cách các Pod tham chiếu đến PodGroup của chúng thông qua trường [nhóm lập lịch (scheduling group)](./59-scheduling-group-vi.md).
* Tìm hiểu về [Lập lịch workload nhận biết topology](./80-workload-topology-scheduling-vi.md).
* Hiểu thuật toán [gang scheduling](150-gang-scheduling-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Workload giữ gì, PodGroup giữ gì, và Pod nào đang chạy thì do đối tượng nào quản lý?
2. Bạn sửa chính sách trong một PodGroupTemplate của Workload. Các PodGroup đang chạy có đổi
   theo không? Còn `controllerRef` — nó có ảnh hưởng đến quyết định lập lịch không?
3. Bạn muốn dùng gang scheduling cho một Job mà không tự tạo đối tượng Workload hay PodGroup
   nào. Job phải thỏa những điều kiện nào, và `minCount` khi đó bằng bao nhiêu?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Workload giữ chính sách lập lịch** dưới dạng các PodGroupTemplate — một mẫu chính sách
   tĩnh, tồn tại lâu dài, không theo dõi trạng thái runtime. **PodGroup giữ trạng thái lập lịch
   runtime** cho từng thực thể, được controller tạo ra từ template. Còn **Pod đang chạy vẫn do
   workload controller quản lý** (ví dụ Job controller) — Workload chỉ định *cách lập lịch*,
   không quản lý vòng đời Pod.
2. **Không, PodGroup đang tồn tại không đổi.** Khi controller tạo PodGroup, nó **sao chép**
   `schedulingPolicy` vào spec riêng của PodGroup, và bài nói rõ thay đổi ở Workload **chỉ ảnh
   hưởng đến những PodGroup mới được tạo**. Trên thực tế bạn còn không sửa được: **toàn bộ spec
   của Workload là bất biến sau khi tạo**. Với `controllerRef`: **không ảnh hưởng gì tới lập
   lịch** — bài ghi thẳng rằng dữ liệu này chỉ dùng cho khả năng quan sát và công cụ hỗ trợ.
3. Cần bật feature gate **`WorkloadWithJob`**, và Job phải là **dạng indexed chạy song song có
   `.spec.parallelism` bằng `.spec.completions`**. Khi đó Job controller tự tạo cả Workload lẫn
   PodGroup, và **`minCount` của chính sách gang được đặt bằng parallelism của Job** — tức là
   tất cả Pod phải lập lịch được cùng nhau trước khi bất kỳ Pod nào được bind. Đây là con
   đường tích hợp sẵn; các controller khác như JobSet thì tự quản lý Workload và PodGroup của
   riêng chúng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
