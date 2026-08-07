# PodGroup API

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/podgroup-api/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](LO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 5/15 · Kiểm chứng ở Lab 13 (tùy chọn, chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Bài này ở trạng thái `alpha` từ v1.36, nằm trong API group
`scheduling.k8s.io/v1alpha2` và cần feature gate `GenericWorkload` — **API có thể đổi giữa các
phiên bản**. Cluster lab 3 VM trên VMware chạy cấu hình mặc định nên hai thứ đó đều tắt: các
lệnh `kubectl get podgroups` trong bài sẽ không chạy được. Đọc để hiểu khái niệm.

Điều dễ lẫn nhất ở bài này là **hai đối tượng na ná nhau**: Workload và PodGroup. Giữ chặt một
câu: Workload là *mẫu chính sách* tồn tại lâu dài, PodGroup là *thực thể runtime*.

**Phải hiểu ở lần đọc này:**

- PodGroup là đơn vị lập lịch **khép kín**: nó mang chính sách lập lịch chi phối việc sắp đặt
  và ghi lại trạng thái runtime của quyết định lập lịch đó.
- `spec.schedulingPolicy` (`basic` hoặc `gang`) được **sao chép** từ PodGroupTemplate của
  Workload tại thời điểm tạo; với PodGroup độc lập thì bạn đặt trực tiếp.
- Vì sao tách Workload và PodGroup: Workload là định nghĩa chính sách dùng chung, còn cập nhật
  status của từng PodGroup **không tranh chấp** trên đối tượng dùng chung đó.
- Chuỗi ba bước ở mục *Cách các thành phần khớp với nhau*: controller tạo Workload → tạo
  PodGroup từ một PodGroupTemplate → tạo Pod trỏ về PodGroup qua
  `spec.schedulingGroup.podGroupName`. **Job là workload controller tích hợp sẵn duy nhất**
  tuân theo mẫu này.
- Ranh giới của status: condition `PodGroupScheduled` **chỉ phản ánh quyết định lập lịch ban
  đầu**, scheduler không cập nhật nó khi Pod lỗi hay bị trục xuất về sau.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Quyền sở hữu*, thứ tự tạo, bảo vệ khi xóa | là nội dung của bài kế tiếp | [bài 76](76-podgroup-lifecycle-vi.md) |
| `spec.podGroupTemplateRef` và cấu trúc Workload | chưa học đối tượng Workload | [bài 77](77-workload-api-vi.md) |
| Chi tiết `basic` và `gang` | ở đây mới chỉ là tên chính sách | [bài 79](79-workload-policies-vi.md) |
| Danh sách đầy đủ condition và reason | sinh ra bởi chu trình lập lịch | [bài 151](151-podgroup-scheduling-vi.md) |
| *Yêu cầu thiết bị DRA cho một PodGroup* | tính năng alpha riêng, cần thiết bị thật | khi công việc thực sự cần |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

PodGroup là một đối tượng runtime đại diện cho một nhóm các Pod được lập lịch cùng nhau
như một đơn vị duy nhất. Trong khi [Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/)
định nghĩa các mẫu (template) chính sách lập lịch, PodGroup là đối tượng runtime tương ứng,
mang theo cả chính sách lẫn trạng thái lập lịch cho một thực thể (instance) cụ thể của nhóm đó.

## PodGroup là gì? (What is a PodGroup?)

Tài nguyên PodGroup API là một phần của API group `scheduling.k8s.io/v1alpha2`,
và cluster của bạn phải bật API group đó, cũng như
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `GenericWorkload`,
trước khi bạn có thể sử dụng API này.

PodGroup là một đơn vị lập lịch khép kín. Nó định nghĩa nhóm các Pod cần được lập lịch
cùng nhau, mang theo chính sách lập lịch chi phối việc sắp đặt (placement), và ghi lại
trạng thái runtime của quyết định lập lịch đó.

## Cấu trúc API (API structure)

Một PodGroup gồm phần `spec` định nghĩa hành vi lập lịch mong muốn và
phần `status` phản ánh trạng thái lập lịch hiện tại.

### Chính sách lập lịch (Scheduling policy)

Mỗi PodGroup mang một [chính sách lập lịch](https://kubernetes.io/docs/concepts/workloads/workload-api/policies/)
(`basic` hoặc `gang`) trong `spec.schedulingPolicy`. Khi một workload controller tạo
PodGroup, chính sách này được sao chép từ PodGroupTemplate của Workload tại thời điểm tạo.
Với các PodGroup độc lập (standalone), bạn đặt chính sách này trực tiếp.

```yaml
spec:
  schedulingPolicy:
    gang:
      minCount: 4
```

### Tham chiếu template (Template reference)

Trường tùy chọn `spec.podGroupTemplateRef` liên kết PodGroup ngược về PodGroupTemplate
trong Workload mà từ đó nó được tạo ra. Điều này hữu ích cho khả năng quan sát (observability) và các công cụ hỗ trợ.

```yaml
spec:
  podGroupTemplateRef:
    workload:
      workloadName: training-policy
      podGroupTemplateName: worker
```

### Yêu cầu thiết bị DRA cho một PodGroup (Requesting DRA devices for a PodGroup)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Các thiết bị (device) khả dụng thông qua cơ chế cấp phát tài nguyên động
(Dynamic Resource Allocation — DRA)
có thể được một PodGroup yêu cầu thông qua trường `spec.resourceClaims` của nó:

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  name: training-group
  namespace: some-ns
spec:
  ...
  resourceClaims:
  - name: pg-claim
    resourceClaimName: my-pg-claim
  - name: pg-claim-template
    resourceClaimTemplateName: my-pg-template
```

Các ResourceClaim gắn với PodGroup có thể được chia sẻ bởi tất cả các Pod thuộc
nhóm đó. Vì chỉ cần một tham chiếu đến PodGroup trong `status.reservedFor` của
ResourceClaim thay vì từng Pod riêng lẻ, nên bất kỳ số lượng Pod nào trong cùng một
PodGroup đều có thể dùng chung một ResourceClaim. Các ResourceClaim cũng có thể được
sinh ra từ các ResourceClaimTemplate cho mỗi PodGroup, cho phép các thiết bị được
cấp phát cho mỗi ResourceClaim được sinh ra đó được chia sẻ bởi các Pod trong từng PodGroup.

Để biết thêm chi tiết và một ví dụ đầy đủ hơn, xem
[tài liệu DRA](https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/#workload-resource-claims).

### Trạng thái (Status)

Scheduler cập nhật `status.conditions` để báo cáo liệu nhóm đã được lập lịch
thành công hay chưa. Condition chính là `PodGroupScheduled`, có giá trị `True`
khi tất cả các Pod bắt buộc đã được sắp đặt xong và `False` khi việc lập lịch thất bại.

> **Ghi chú:**
> Condition `PodGroupScheduled` chỉ phản ánh quyết định lập lịch ban đầu.
> Scheduler không cập nhật nó nếu sau đó các Pod bị lỗi hoặc bị trục xuất (evict). Xem
> [Giới hạn](https://kubernetes.io/docs/concepts/workloads/podgroup-api/lifecycle/#limitations)
> để biết chi tiết.

Xem trang [Vòng đời của PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/lifecycle/#podgroup-status)
để biết danh sách đầy đủ các condition và lý do (reason).

## Tạo một PodGroup (Creating a PodGroup)

Tài nguyên PodGroup API là một phần của API group `scheduling.k8s.io/v1alpha2`
(và cluster của bạn phải bật API group đó, cũng như
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `GenericWorkload`,
trước khi bạn có thể sử dụng API này).

Manifest sau đây tạo một PodGroup với chính sách gang scheduling, yêu cầu
ít nhất 4 Pod phải có thể lập lịch được đồng thời:

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  name: training-worker-0
  namespace: default
spec:
  schedulingPolicy:
    gang:
      minCount: 4
```

Bạn có thể xem các PodGroup trong cluster của mình:

```shell
kubectl get podgroups
```

Để xem status đầy đủ bao gồm các condition lập lịch:

```shell
kubectl describe podgroup training-worker-0
```

## Cách các thành phần khớp với nhau (How it fits together)

Mối quan hệ giữa controller, Workload, PodGroup và Pod tuân theo mẫu sau:

1. Workload controller tạo một Workload định nghĩa các PodGroupTemplate cùng với các chính sách lập lịch.
2. Với mỗi thực thể runtime, controller tạo một PodGroup từ một trong các PodGroupTemplate của Workload.
3. Controller tạo các Pod tham chiếu đến PodGroup
   thông qua trường `spec.schedulingGroup.podGroupName`.

Controller của [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/) hiện là
workload controller tích hợp sẵn duy nhất tuân theo mẫu này.
Các controller tùy chỉnh có thể triển khai cùng luồng xử lý này cho các loại workload của riêng chúng.

```yaml
apiVersion: scheduling.k8s.io/v1alpha2
kind: Workload
metadata:
  name: training-policy
spec:
  podGroupTemplates:
  - name: worker
    schedulingPolicy:
      gang:
        minCount: 4
---
apiVersion: scheduling.k8s.io/v1alpha2
kind: PodGroup
metadata:
  name: training-worker-0
spec:
  podGroupTemplateRef:
    workload:
      workloadName: training-policy
      podGroupTemplateName: worker
  schedulingPolicy:
    gang:
      minCount: 4
---
apiVersion: v1
kind: Pod
metadata:
  name: worker-0
spec:
  schedulingGroup:
    podGroupName: training-worker-0
  containers:
  - name: ml-worker
    image: training:v1
```

Workload đóng vai trò là một định nghĩa chính sách tồn tại lâu dài, trong khi các PodGroup
xử lý trạng thái runtime tạm thời cho từng thực thể. Sự tách biệt này có nghĩa là
các cập nhật status của từng PodGroup riêng lẻ không tranh chấp trên đối tượng Workload dùng chung.

## Tiếp theo (What's next)

* Tìm hiểu chi tiết về [vòng đời của PodGroup](https://kubernetes.io/docs/concepts/workloads/podgroup-api/lifecycle/).
* Đọc về [Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/) — API cung cấp các PodGroupTemplate.
* Xem cách các Pod tham chiếu đến PodGroup của chúng thông qua trường [scheduling group](https://kubernetes.io/docs/concepts/workloads/pods/scheduling-group/).
* Hiểu về thuật toán [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Workload và PodGroup, cái nào mang chính sách và cái nào mang trạng thái runtime? Bài nêu
   lợi ích gì của việc tách hai đối tượng này?
2. Bạn thấy một PodGroup có condition `PodGroupScheduled: True`. Kết luận được rằng nhóm đó
   hiện đang chạy đủ Pod không?
3. Trên cluster lab mặc định của bạn, `kubectl get podgroups` sẽ không trả về gì hữu ích. Hai
   thứ nào phải được bật trước thì API này mới dùng được?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Workload mang chính sách, PodGroup mang trạng thái runtime.** Workload đóng vai trò định
   nghĩa chính sách **tĩnh, tồn tại lâu dài** qua các PodGroupTemplate; PodGroup là **thực thể
   runtime** cho từng lần chạy, mang bản sao của chính sách cộng với trạng thái lập lịch. Lợi
   ích bài nêu rất cụ thể: **các cập nhật status của từng PodGroup riêng lẻ không tranh chấp
   trên đối tượng Workload dùng chung**. Hệ quả kèm theo là PodGroup tự chứa — chính sách được
   sao chép vào nó lúc tạo, chứ không phải tra ngược về Workload mỗi lần lập lịch.
2. **Không.** Đây đúng là chỗ dễ nhầm nhất của bài: `PodGroupScheduled` **chỉ phản ánh quyết
   định lập lịch ban đầu**. Một khi được đặt thành `True`, **scheduler sẽ không cập nhật lại
   nó** nếu sau đó các Pod bị lỗi, bị trục xuất hay ngừng chạy. Nó trả lời câu hỏi "nhóm này
   đã từng được sắp đặt thành công chưa", không phải "nhóm này có đang khỏe không".
3. **API group `scheduling.k8s.io/v1alpha2`** và **feature gate `GenericWorkload`**. Bài nhắc
   điều kiện này hai lần — ở mục *PodGroup là gì?* và ở mục *Tạo một PodGroup* — vì thiếu một
   trong hai thì đối tượng PodGroup không tồn tại trong cluster.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
