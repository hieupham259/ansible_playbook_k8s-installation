# Lập lịch PodGroup (PodGroup Scheduling)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/scheduling-eviction/podgroup-scheduling/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 13](LO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao),
bài 12/15 · Kiểm chứng ở Lab 13 (tùy chọn, chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Giai đoạn 13 không bắt buộc với admin mới.** Phần lớn giai đoạn này là tính năng alpha/beta
hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Chỉ đọc khi đã vững giai đoạn 1–12 hoặc khi
công việc thực sự cần. Bài này ở trạng thái `alpha` từ v1.36 — **API và tên các điểm mở rộng
có thể đổi giữa các phiên bản**. Cluster lab 3 VM trên VMware không bật API group cần thiết,
nên chu trình mô tả ở đây không quan sát được. Đọc để hiểu khái niệm.

Đây là bài **cơ chế** đứng dưới [bài 150](150-gang-scheduling-vi.md): gang scheduling nói
*luật*, bài này nói *chu trình thực thi luật đó*. Hai mục đáng dừng lại là *Chu trình lập lịch
PodGroup* và *Hạn chế* — mục sau cho biết thuật toán này **không** vạn năng.

**Phải hiểu ở lần đọc này:**

- Vấn đề bài này giải: scheduler tiêu chuẩn đánh giá Pod **tuần tự**, nên hai workload cạnh
  tranh có thể mỗi bên lập lịch được một phần, **chiếm dụng dung lượng cluster mà không bên
  nào đủ tài nguyên để khởi động hoàn chỉnh** — deadlock tài nguyên.
- Bốn bước của chu trình: gom mọi Pod cùng nhóm đang trong hàng đợi và **sắp xếp tất định** →
  chụp **một snapshot trạng thái cluster** dùng cho cả chu trình → tìm placement → **quyết
  định nguyên tử** cho cả nhóm. Thất bại thì tất cả Pod về hàng đợi và áp dụng backoff tiêu
  chuẩn.
- Thuật toán mặc định vẫn dựa trên lập lịch từng Pod: filter/score cho từng Pod, Pod vừa chỗ
  thì được **giả định và giữ chỗ** trên node cho tới hết thuật toán; sau đó điểm mở rộng
  `Permit` kiểm tra tiêu chí cấp nhóm.
- Mục *Hạn chế* — phần dễ bỏ qua nhất: vì thứ tự xử lý là tất định, thuật toán **chỉ được kỳ
  vọng tìm ra placement với nhóm Pod đồng nhất và không có phụ thuộc liên Pod**. Nhóm không
  đồng nhất, hoặc có affinity/anti-affinity/topology spread, thì **không đảm bảo**. Ngoài ra
  mọi Pod trong nhóm phải cùng `.spec.schedulerName`, kiểm tra trước khi chu trình bắt đầu.
- Hai condition và ý nghĩa của reason: `PodGroupScheduled` (`Scheduled`, `Unschedulable`,
  `SchedulerError`) và `DisruptionTarget` (`PreemptionByScheduler`).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ba pha của *Thuật toán lập lịch theo placement* | mới là khung, plugin cụ thể ở bài sau | [bài 153](153-topology-aware-scheduling-vi.md) |
| `DisruptionTarget` và việc preempt cả một PodGroup | là bài kế tiếp | [bài 152](152-workload-aware-preemption-vi.md) |
| Điểm mở rộng `PostFilter`, `Permit`, pha filter/score | nền đã học | giai đoạn 7 — [bài 147](147-scheduling-framework-vi.md) |
| Ràng buộc phân tán topology, affinity/anti-affinity | nền đã học | giai đoạn 7 — [bài 138](138-assign-pod-node-vi.md), [bài 140](140-topology-spread-constraints-vi.md) |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Bộ lập lịch (scheduler) tiêu chuẩn của Kubernetes đánh giá các Pod một cách tuần tự. Khi nhiều workload,
chẳng hạn các job huấn luyện máy học (machine learning training job), được gửi lên đồng thời,
việc đánh giá tuần tự này có thể dẫn đến deadlock tài nguyên.
Ví dụ, hai workload cạnh tranh nhau có thể mỗi bên chỉ lập lịch được một phần các Pod của mình,
chiếm dụng dung lượng cluster nhưng không bên nào có đủ tài nguyên để khởi động hoàn chỉnh.

Chu trình lập lịch PodGroup đánh giá một nhóm Pod như một đơn vị duy nhất.
Scheduler cố gắng tìm vị trí sắp đặt (placement) cho tất cả các Pod trong nhóm cùng một lúc.
Nếu không tìm được đủ tài nguyên để thỏa mãn yêu cầu của toàn bộ nhóm, không Pod nào được bind.

Ngoài ra, việc coi cả nhóm như một thực thể thống nhất tạo nên một kiến trúc nền tảng
giúp đơn giản hóa việc triển khai các tính năng lập lịch theo nhóm khác.

Tính năng này phụ thuộc vào [Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/).
Hãy đảm bảo feature gate [`GenericWorkload`](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/#GenericWorkload)
và nhóm API (API group) `scheduling.k8s.io/v1alpha1`
đã được bật trong cluster.

## Chu trình lập lịch PodGroup (PodGroup scheduling cycle)

Để hỗ trợ lập lịch một nhóm Pod cùng nhau, kube-scheduler sử dụng **chu trình lập lịch PodGroup**.
Thay vì xử lý từng Pod riêng lẻ và giữ chúng lại tại một cổng `WaitOnPermit`,
scheduler đánh giá tập thể toàn bộ nhóm các Pod đang chờ thuộc về một PodGroup cụ thể.
Thay vì thực thi các chu trình lập lịch riêng cho từng Pod,
nó đánh giá tính khả thi cho cả nhóm rồi sau đó chuyển thẳng sang pha binding.

Khi scheduler lấy ra (pop) một Pod thuộc về một PodGroup, nó truy xuất tất cả các Pod khác đang trong hàng đợi thuộc nhóm đó.
Sau đó nó sắp xếp chúng một cách tất định (deterministic) dựa trên độ ưu tiên và thời điểm chúng được scheduler quan sát thấy lần đầu,
rồi khởi động chu trình lập lịch PodGroup như sau:

1. **Chụp snapshot trạng thái cluster:** Khi scheduler bắt đầu đánh giá một PodGroup,
   nó chụp một snapshot duy nhất của trạng thái cluster, tồn tại trong suốt thời gian của chu trình.
   Điều này đảm bảo việc đánh giá luôn nhất quán cho cả nhóm và ngăn ngừa race condition với các sự kiện khác.

2. **Tìm các vị trí sắp đặt khả thi:** Scheduler chạy [thuật toán lập lịch PodGroup](#podgroup-scheduling-algorithm)
   để tìm các vị trí sắp đặt hợp lệ trên Node cho các Pod trong nhóm.

3. **Quyết định nguyên tử (atomic):** Tùy theo kết quả của thuật toán, quyết định lập lịch
   được áp dụng một cách nguyên tử cho toàn bộ PodGroup.

   * **Thành công:** Nếu scheduler tìm thấy đủ tài nguyên và các vị trí sắp đặt hợp lệ cho các Pod
     (ví dụ: thỏa mãn ràng buộc `minCount` cho [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)),
     những Pod đó đi thẳng vào chu trình binding với các node đã được chọn cho chúng.
     Các Pod còn lại không thể lập lịch được trả về hàng đợi lập lịch để chờ tài nguyên khả dụng,
     nhờ đó chúng có thể gia nhập cùng các Pod đã được lập lịch.

     Hơn nữa, nếu các Pod mới được thêm vào một PodGroup sau khi những Pod khác đã được lập lịch,
     chu trình sẽ đánh giá các Pod mới trong khi vẫn tính đến những Pod hiện có.

   * **Thất bại:** Nếu scheduler không thể tìm đủ tài nguyên để làm cho PodGroup trở nên khả thi
     (ví dụ: không đáp ứng được ràng buộc `minCount`), toàn bộ PodGroup bị coi là không thể lập lịch (unschedulable).
     Không Pod nào được bind; thay vào đó, tất cả được trả về hàng đợi lập lịch.
     Logic backoff lập lịch tiêu chuẩn được áp dụng, cho phép PodGroup được thử lại sau.

Bằng cách dùng cách tiếp cận một chu trình duy nhất này, scheduler tránh được các điểm nghẽn kém hiệu quả,
nơi các nhóm mới chỉ được lập lịch một phần chiếm giữ dung lượng cluster trong khi chờ vô thời hạn để phần còn lại của nhóm có chỗ.

## Thuật toán lập lịch PodGroup (PodGroup scheduling algorithm) {#podgroup-scheduling-algorithm}

Thuật toán lập lịch PodGroup mặc định dựa chủ yếu vào thuật toán lập lịch cơ sở theo từng Pod.
Nó lặp qua các Pod và thực hiện những bước sau cho mỗi Pod:

1. Tìm một node khả thi bằng các pha lọc (filtering) và chấm điểm (scoring) tiêu chuẩn cho từng Pod.

   * Nếu Pod vừa chỗ, nó được tạm thời giả định (assumed) và giữ chỗ (reserved) trên node được chọn cho đến khi thuật toán lập lịch kết thúc.
   * Nếu Pod không thể vừa, scheduler thử preemption (chiếm chỗ) bằng cách chạy điểm mở rộng `PostFilter`.

2. Kiểm tra xem các Pod có thể lập lịch có đáp ứng các tiêu chí lập lịch của nhóm hay không
   (ví dụ: `minCount` cho [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/)) thông qua điểm mở rộng `Permit`.
   Nếu điểm mở rộng này trả về trạng thái `Success` cho bất kỳ Pod nào, PodGroup được xem là khả thi.
   Nếu thuật toán xử lý hết tất cả các Pod mà không đạt được trạng thái `Success`, PodGroup bị coi là không thể lập lịch.

## Thuật toán lập lịch theo placement (Placement scheduling algorithm) {#placement-scheduling-algorithm}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [alpha]`

Thuật toán lập lịch theo placement là một thuật toán lập lịch PodGroup thay thế, sử dụng các
[plugin lập lịch](https://kubernetes.io/docs/reference/scheduling/config/#scheduling-plugins) để tìm vị trí sắp đặt
tối ưu cho PodGroup đang được xem xét. Người dùng có thể điều chỉnh thuật toán theo nhu cầu cụ thể của mình
bằng cách sử dụng và cấu hình các plugin.

Thuật toán tiến hành qua ba pha chính cho một PodGroup nhất định:

### Pha 1: Sinh các placement ứng viên (Candidate placement generation)

Sinh ra các *placement* ứng viên (các tập con của node mà về mặt lý thuyết là khả thi cho việc gán PodGroup),
ví dụ dựa trên các ràng buộc lập lịch của PodGroup (có thể được định nghĩa
trong đối tượng PodGroup).

Pha này thực thi dưới dạng điểm mở rộng: `PlacementGeneratePlugin`.

### Pha 2: Lọc ở mức Pod và kiểm tra tính khả thi (Pod-level filtering and feasibility check)

Xác thực từng placement được đề xuất, bằng cách chạy thuật toán lập lịch PodGroup mặc định, để xem
số lượng Pod cần thiết từ PodGroup có thể vừa chỗ hay không. Nếu vừa, placement đó được đánh dấu là khả thi.

### Pha 3: Chấm điểm và lựa chọn placement (Placement scoring and selection)

Chấm điểm tất cả các placement khả thi để chọn ra miền (domain) tối ưu cho PodGroup.

Pha này thực thi dưới dạng điểm mở rộng: `PlacementScorePlugin`.

### Hạn chế (Limitations)

Thuật toán lập lịch PodGroup dựa trên một cách sắp xếp Pod cụ thể và có thể không tìm được một placement hợp lệ
mà lẽ ra có thể được phát hiện nếu xử lý các Pod của nhóm theo một thứ tự khác. Cụ thể:

* Với các nhóm Pod **đồng nhất (homogeneous)** cơ bản (tức là những nhóm mà tất cả Pod có yêu cầu lập lịch giống hệt nhau
  và không có phụ thuộc liên Pod như affinity, anti-affinity hay ràng buộc phân tán topology (topology spread constraints)),
  thuật toán được kỳ vọng sẽ tìm được một placement nếu placement đó tồn tại.

* Với các nhóm Pod **không đồng nhất (heterogeneous)**, việc tìm được một placement hợp lệ không được đảm bảo.

* Với các nhóm Pod có **phụ thuộc liên Pod (inter-Pod dependencies)**, việc tìm được một placement hợp lệ không được đảm bảo.

Ngoài các trường hợp trên, với những trường hợp liên quan đến **phụ thuộc nội nhóm (intra-group dependencies)**
(ví dụ: khi khả năng lập lịch của một Pod phụ thuộc vào một thành viên khác của nhóm thông qua inter-Pod affinity),
thuật toán này có thể không tìm được placement bất kể trạng thái cluster ra sao, do thứ tự xử lý tất định của nó.

Để đảm bảo hành vi nhất quán trong suốt toàn bộ chu trình, thuật toán yêu cầu tất cả các Pod thuộc về cùng một PodGroup
phải có cùng `.spec.schedulerName`. Yêu cầu này được xác thực trước khi chu trình bắt đầu,
và PodGroup bị từ chối nếu ràng buộc không được đáp ứng.

## Các condition của PodGroup (PodGroup conditions)

Sau khi một chu trình lập lịch PodGroup hoàn tất, scheduler cập nhật các condition trên
`status.conditions` của PodGroup:

* `PodGroupScheduled`: cho biết PodGroup đã được lập lịch thành công hay chưa.
* `DisruptionTarget`: chỉ ra rằng PodGroup sắp bị chấm dứt do một sự gián đoạn (disruption)
  chẳng hạn như preemption.

### `PodGroupScheduled`

Khi chu trình lập lịch thành công, condition được đặt thành `True` với lý do (reason)
`Scheduled`. Với các PodGroup dùng chính sách `gang`, điều này có nghĩa là ít nhất `minCount` Pod
đã được sắp đặt.

Khi lập lịch thất bại, condition được đặt thành `False` với một trong các lý do sau:

* `Unschedulable` — nhóm không thể được sắp đặt do các ràng buộc tài nguyên,
  các quy tắc affinity hoặc anti-affinity, hoặc không đủ dung lượng cho gang.
* `SchedulerError` — lập lịch thất bại vì lỗi nội bộ của scheduler
  (ví dụ: khi phân tích các ràng buộc lập lịch như `nodeAffinity`).

### `DisruptionTarget`

Khi scheduler preempt một PodGroup để nhường chỗ cho các PodGroup hoặc Pod
có độ ưu tiên cao hơn, nó đặt condition này thành `True` với lý do `PreemptionByScheduler`.

Bạn có thể kiểm tra các condition bằng:

```shell
kubectl get podgroup <name> -o jsonpath='{.status.conditions}'
```

## Tiếp theo (What's next)

* Tìm hiểu về [Workload API](https://kubernetes.io/docs/concepts/workloads/workload-api/).
* Xem cách [tham chiếu một Workload](https://kubernetes.io/docs/concepts/workloads/pods/workload-reference/) trong một Pod.
* Đọc về [gang scheduling](https://kubernetes.io/docs/concepts/scheduling-eviction/gang-scheduling/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho một giai đoạn tùy chọn:

1. Vì sao đánh giá Pod tuần tự lại dẫn tới deadlock tài nguyên khi hai job lớn được gửi lên
   cùng lúc? Việc chụp một snapshot trạng thái cluster cho cả chu trình giải quyết chuyện gì?
2. Nếu tồn tại một cách sắp đặt hợp lệ cho cả nhóm, chu trình lập lịch PodGroup có chắc chắn
   tìm ra nó không?
3. `PodGroupScheduled: False` với reason `Unschedulable` khác với reason `SchedulerError` ở
   chỗ nào? Đọc được điều gì khác nhau về nguyên nhân?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì mỗi Pod được xét riêng, **hai workload cạnh tranh có thể mỗi bên chỉ lập lịch được một
   phần các Pod của mình, chiếm dụng dung lượng cluster nhưng không bên nào có đủ tài nguyên để
   khởi động hoàn chỉnh**. Chu trình PodGroup đổi đơn vị đánh giá từ Pod sang nhóm: nếu không
   đủ tài nguyên cho cả nhóm thì **không Pod nào được bind**. Snapshot phục vụ mục đích khác:
   nó **giữ trạng thái cluster cố định trong suốt chu trình**, để việc đánh giá **nhất quán cho
   cả nhóm** và **không bị race condition với các sự kiện khác** xen vào giữa chừng.
2. **Không chắc chắn.** Đây là chỗ dễ tưởng bở nhất. Thuật toán **dựa trên một cách sắp xếp Pod
   cụ thể (tất định)**, nên nó **chỉ được kỳ vọng tìm ra placement với các nhóm Pod đồng nhất**
   — tất cả Pod có yêu cầu lập lịch giống hệt nhau và không có phụ thuộc liên Pod. Với nhóm
   **không đồng nhất** hoặc có **phụ thuộc liên Pod** (affinity, anti-affinity, topology spread),
   việc tìm được placement hợp lệ **không được đảm bảo**; riêng với phụ thuộc *nội nhóm* — khả
   năng lập lịch của một Pod phụ thuộc vào một thành viên khác qua inter-Pod affinity — thuật
   toán có thể **không bao giờ** tìm ra placement, bất kể trạng thái cluster.
3. **`Unschedulable`** nghĩa là quyết định đúng nhưng **cluster không đáp ứng được**: hết tài
   nguyên, vướng quy tắc affinity/anti-affinity, hoặc không đủ dung lượng cho gang. Đây là tình
   huống bình thường, nhóm sẽ được thử lại theo backoff. **`SchedulerError`** nghĩa là **lỗi
   nội bộ của scheduler** — ví dụ khi phân tích các ràng buộc lập lịch như `nodeAffinity`. Cái
   đầu bảo bạn nhìn vào dung lượng cluster; cái sau bảo bạn nhìn vào manifest và log scheduler.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
