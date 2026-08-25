# Cấp phát dư dung lượng Node cho Cluster (Overprovision Node Capacity For A Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/node-overprovisioning/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 16 — Vòng đời node](00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node), bài 4/4 · Kiểm chứng một phần
trên cluster lab: tạo PriorityClass âm và Deployment giữ chỗ, rồi chứng minh Pod thường **preempt**
được Pod giữ chỗ khi node hết dung lượng.

Bài này giả định cluster có **node autoscaler** — cluster lab ba VM cố định thì không có. Phần
"rút ngắn thời gian chờ node mới" vì vậy không tái hiện được; nhưng cơ chế cốt lõi —
PriorityClass âm + Pod giữ chỗ bị preempt — thì kiểm chứng được đầy đủ trên cluster lab. Bài
dựa lên [141](141-pod-priority-preemption-vi.md) đã đọc ở giai đoạn 7a.

**Phải hiểu ở lần đọc này:**

- Ý tưởng trung tâm: dự trữ dung lượng bằng **Pod thật đang chạy** mang PriorityClass giá trị
  **âm**. Chúng chiếm chỗ theo `requests`, nhưng vì độ ưu tiên thấp nhất nên là **ứng viên đầu
  tiên bị preempt** khi có Pod thường cần chỗ.
- Vì sao phải là **Pod giữ chỗ** chứ không phải chỉ "để trống node": scheduler chỉ nhìn
  `requests`; dung lượng để trống không có gì bảo đảm nó còn trống khi cần. Pod giữ chỗ biến
  dung lượng dự trữ thành thứ scheduler thấy được và thu hồi được tức thì.
- `globalDefault: false` là bắt buộc với PriorityClass âm này — nếu đặt `true` thì mọi Pod
  không khai báo priority trong cluster đều nhận độ ưu tiên âm.
- Tổng dung lượng dự trữ = `replicas × requests` của Pod giữ chỗ; điều chỉnh dự trữ bằng cách
  đổi `requests` hoặc `kubectl scale`. `podAntiAffinity` dạng `preferred` để trải các Pod giữ
  chỗ ra nhiều node.
- Bẫy vận hành: một số autoscaler (ví dụ Karpenter) coi affinity **preferred như hard rule**,
  nên số replica bạn đặt ở đây vô tình trở thành **số node tối thiểu** của cluster.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Hành vi riêng của Karpenter và các node autoscaler | cluster lab không có autoscaler | bài [171](171-node-autoscaling-vi.md) ở giai đoạn 12 |
| Con số dự trữ cụ thể nên đặt bao nhiêu | phụ thuộc kích thước cluster và workload thật | khi vận hành cluster thật, sau giai đoạn 23 (giám sát) |

---

Trang này hướng dẫn bạn cấu hình việc cấp phát dư (overprovisioning) Node trong cluster
Kubernetes của bạn. Cấp phát dư Node là một chiến lược chủ động dự trữ một phần tài nguyên
tính toán của cluster. Việc dự trữ này giúp giảm thời gian cần thiết để lập lịch (schedule)
các Pod mới trong các sự kiện mở rộng (scaling), nâng cao khả năng phản ứng của cluster trước
những đợt tăng đột biến về lưu lượng hoặc nhu cầu workload.

Bằng cách duy trì một lượng dung lượng chưa sử dụng, bạn đảm bảo rằng tài nguyên luôn sẵn sàng
ngay lập tức khi các Pod mới được tạo, tránh việc chúng rơi vào trạng thái pending trong khi
cluster đang mở rộng.

## Trước khi bạn bắt đầu (Before you begin)

- Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
  tiếp với cluster của bạn.
- Bạn nên có hiểu biết cơ bản về
  [Deployment](63-deployment-vi.md),
  độ ưu tiên (priority) của Pod,
  và PriorityClass.
- Cluster của bạn phải được thiết lập với một
  [autoscaler](171-node-autoscaling-vi.md)
  quản lý các node dựa trên nhu cầu.

## Tạo một PriorityClass (Create a PriorityClass)

Bắt đầu bằng việc định nghĩa một PriorityClass cho các Pod giữ chỗ (placeholder). Trước tiên,
tạo một PriorityClass với giá trị độ ưu tiên âm, mà lát nữa bạn sẽ gán cho các Pod giữ chỗ.
Sau đó, bạn sẽ thiết lập một Deployment sử dụng PriorityClass này.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: placeholder # các Pod này đại diện cho dung lượng giữ chỗ
value: -1000
globalDefault: false
description: "Negative priority for placeholder pods to enable overprovisioning."
```

Sau đó tạo PriorityClass:

```shell
kubectl apply -f https://k8s.io/examples/priorityclass/low-priority-class.yaml
```

Tiếp theo, bạn sẽ định nghĩa một Deployment sử dụng PriorityClass có độ ưu tiên âm này và chạy
một container tối giản. Khi bạn thêm nó vào cluster, Kubernetes chạy các Pod giữ chỗ đó để dự
trữ dung lượng. Bất cứ khi nào xảy ra thiếu hụt dung lượng, control plane sẽ chọn một trong các
Pod giữ chỗ này làm ứng viên đầu tiên để preempt (chiếm chỗ).

## Chạy các Pod yêu cầu dung lượng node (Run Pods that request node capacity)

Xem lại manifest mẫu:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: capacity-reservation
  # Bạn nên quyết định sẽ triển khai vào namespace nào
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: capacity-placeholder
  template:
    metadata:
      labels:
        app.kubernetes.io/name: capacity-placeholder
      annotations:
        kubernetes.io/description: "Capacity reservation"
    spec:
      priorityClassName: placeholder
      affinity: # Cố gắng đặt các Pod dự phòng này trên các node khác nhau
                # nếu có thể
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app.kubernetes.io/name: capacity-placeholder
              topologyKey: topology.kubernetes.io/hostname
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.6
        resources:
          requests:
            cpu: "50m"
            memory: "512Mi"
          limits:
            memory: "512Mi"
```

### Chọn một namespace cho các Pod giữ chỗ (Pick a namespace for the placeholder pods)

Bạn nên chọn, hoặc tạo, một namespace mà các Pod giữ chỗ sẽ được đưa vào.

### Tạo Deployment giữ chỗ (Create the placeholder deployment)

Tạo một Deployment dựa trên manifest đó:

```shell
# Thay tên namespace "example"
kubectl --namespace example apply -f https://k8s.io/examples/deployments/deployment-with-capacity-reservation.yaml
```

## Điều chỉnh resource request của Pod giữ chỗ (Adjust placeholder resource requests)

Cấu hình resource request và limit cho các Pod giữ chỗ để xác định lượng tài nguyên cấp phát dư
mà bạn muốn duy trì. Việc dự trữ này đảm bảo một lượng CPU và memory cụ thể luôn được giữ sẵn
cho các Pod mới.

Để chỉnh sửa Deployment, sửa mục `resources` trong file manifest của Deployment
để đặt request và limit phù hợp. Bạn có thể tải file đó về máy rồi chỉnh sửa
bằng trình soạn thảo văn bản mà bạn thích.

Bạn cũng có thể chỉnh sửa Deployment bằng kubectl:

```shell
kubectl edit deployment capacity-reservation
```

Ví dụ, để dự trữ tổng cộng 0.5 CPU và 1GiB memory trên 5 Pod giữ chỗ,
hãy định nghĩa resource request và limit cho một Pod giữ chỗ như sau:

```yaml
  resources:
    requests:
      cpu: "100m"
      memory: "200Mi"
    limits:
      cpu: "100m"
```

## Đặt số lượng replica mong muốn (Set the desired replica count)

### Tính tổng tài nguyên được dự trữ (Calculate the total reserved resources)

Ví dụ, với 5 replica, mỗi replica dự trữ 0.1 CPU và 200MiB memory:  
Tổng CPU được dự trữ: 5 × 0.1 = 0.5 (trong đặc tả Pod, bạn sẽ viết số lượng là `500m`)  
Tổng memory được dự trữ: 5 × 200MiB = 1GiB (trong đặc tả Pod, bạn sẽ viết `1 Gi`)  

Để mở rộng Deployment, điều chỉnh số lượng replica dựa trên kích thước cluster và workload dự kiến:

```shell
kubectl scale deployment capacity-reservation --replicas=5
```

Xác minh việc mở rộng:

```shell
kubectl get deployment capacity-reservation
```

Kết quả sẽ phản ánh số lượng replica đã được cập nhật:

```none
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
capacity-reservation   5/5     5            5           2m
```

> **Ghi chú:**
> Một số autoscaler, đặc biệt là [Karpenter](171-node-autoscaling-vi.md#karpenter),
> coi các quy tắc affinity dạng preferred như là quy tắc bắt buộc (hard rule) khi cân nhắc
> việc mở rộng node. Nếu bạn dùng Karpenter hoặc một autoscaler node khác sử dụng cùng
> heuristic này, số lượng replica mà bạn đặt ở đây cũng đồng thời đặt số lượng node tối thiểu
> cho cluster của bạn.

## Tiếp theo (What's next)

- Tìm hiểu thêm về [PriorityClass](141-pod-priority-preemption-vi.md#priorityclass) và cách
  chúng ảnh hưởng đến việc lập lịch Pod.
- Khám phá [tự động mở rộng node](171-node-autoscaling-vi.md) để điều chỉnh động kích thước
  cluster dựa trên nhu cầu workload.
- Hiểu về [preemption của Pod](141-pod-priority-preemption-vi.md), một
  cơ chế then chốt để Kubernetes xử lý tranh chấp tài nguyên. Cùng trang đó cũng đề cập tới
  _eviction_ (thu hồi), vốn ít liên quan hơn tới cách tiếp cận dùng Pod giữ chỗ, nhưng cũng là
  một cơ chế để Kubernetes phản ứng khi tài nguyên bị tranh chấp.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 16:

1. Vì sao phải dự trữ dung lượng bằng **Pod giữ chỗ đang chạy**, thay vì đơn giản là không
   triển khai gì lên một phần node và coi đó là dung lượng dự phòng?
2. **Câu bẫy.** PriorityClass của Pod giữ chỗ có `value: -1000`. Giá trị âm này khiến Pod giữ
   chỗ **khó được lập lịch hơn** Pod thường — đúng hay sai? Nó thực sự quyết định điều gì?
3. Trên cluster lab ba VM không có autoscaler, phần nào của bài **vẫn kiểm chứng được** và phần
   nào **không**? Nói rõ vì sao.
4. Bạn muốn dự trữ tổng cộng 500m CPU và 1GiB memory. Với 5 replica Pod giữ chỗ thì `requests`
   của mỗi Pod là bao nhiêu, và vì sao `globalDefault` của PriorityClass phải là `false`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì scheduler ra quyết định dựa trên **`requests` của các Pod đang tồn tại**, không dựa trên
   ý định của con người. Dung lượng "để trống" không được cluster ghi nhận, nên bất kỳ workload
   nào cũng có thể chiếm mất bất cứ lúc nào. Pod giữ chỗ biến phần dự trữ thành thứ scheduler
   **thấy được** (nó chiếm requests thật) và đồng thời **thu hồi được ngay lập tức** (nó bị
   preempt trước tiên) — đó là điều mà chỗ trống thụ động không làm được.
2. **Sai.** Độ ưu tiên âm **không** ngăn Pod giữ chỗ được lập lịch — khi còn dung lượng, chúng
   vẫn chạy bình thường. Cái nó quyết định là **thứ tự bị preempt**: khi cluster thiếu chỗ,
   control plane chọn Pod có độ ưu tiên thấp nhất làm nạn nhân đầu tiên, và Pod giữ chỗ luôn
   là nhóm đó. Trực giác "ưu tiên thấp = khó lên" sai ở chỗ nó lẫn giữa *quyền được lập lịch*
   và *thứ tự bị đuổi*.
3. **Kiểm chứng được:** tạo PriorityClass âm, chạy Deployment giữ chỗ, làm cluster hết chỗ rồi
   quan sát Pod giữ chỗ bị **preempt** để nhường chỗ cho Pod thường — toàn bộ cơ chế
   PriorityClass/preemption nằm trong Kubernetes, không cần autoscaler. **Không kiểm chứng
   được:** lợi ích thật sự của kỹ thuật này — rút ngắn thời gian chờ khi autoscaler cấp node
   mới — vì cluster lab có ba VM cố định, không co giãn.
4. Mỗi Pod giữ chỗ đặt `requests` là **`cpu: 100m`** và **`memory: 200Mi`** (5 × 100m = 500m;
   5 × 200Mi = 1GiB). `globalDefault` phải là `false` vì PriorityClass đánh dấu `globalDefault`
   sẽ áp cho **mọi Pod không khai báo `priorityClassName`** trong cả cluster — đặt `true` ở đây
   nghĩa là toàn bộ workload bình thường bỗng mang độ ưu tiên −1000 và thành ứng viên bị
   preempt đầu tiên.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Đây là bài cuối của [Giai đoạn 16 — Vòng đời node](00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node) — làm xong
thì chuyển sang [Giai đoạn 17 — Nâng cấp cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster).
