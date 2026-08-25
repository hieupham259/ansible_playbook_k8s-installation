# Cấu hình Quota số lượng Pod cho một Namespace (Configure a Pod Quota for a Namespace)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-pod-namespace/
>
> Hạn chế số lượng Pod bạn có thể tạo trong một namespace.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 25 — Quản trị tài nguyên theo namespace](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace)
— **trang con của bài 2/7**, mục [Quản lý tài nguyên Memory, CPU và API](228-manage-resources-tasks-vi.md);
nó không có dòng riêng trong lộ trình. Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối [mục giai đoạn
25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace).

Đây là **trang con cuối** của bài 2/7, và là nửa còn lại của cặp ResourceQuota: bài
[233](233-quota-memory-cpu-namespace-vi.md) đo theo **lượng tài nguyên**, trang này đo theo **số
lượng object**. Checkpoint giai đoạn 25 gọi đích danh việc "đặt quota theo số lượng object và
chứng minh nó chặn", nên đây là bài phải làm được chứ không chỉ đọc.

Cái bẫy Deployment của trang này bạn đã dựng ở
[Lab 7b](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) phần B3, và quota theo số lượng object ở
phần B4. Đọc lại ở đây để lấy **đúng chỗ thông báo từ chối xuất hiện** — nó không nằm trên
terminal như ở bài [233](233-quota-memory-cpu-namespace-vi.md).

**Phải hiểu ở lần đọc này:**

- ResourceQuota `pod-demo` chỉ có một dòng `hard: pods: "2"` — quota này **không nói gì về CPU hay
  memory**, nó đếm object. Ngay sau khi tạo, `status.used` hiện `pods: "0"`.
- Deployment khai `replicas: 3` nhưng chỉ **hai Pod** được tạo. `kubectl apply` **không báo lỗi**;
  bằng chứng nằm trong `status` của chính Deployment: `availableReplicas: 2` và một `message` ghi
  `unable to create pods: ... is forbidden: exceeded quota`.
- Thông báo đó có **cùng cấu trúc ba mảnh** như ở bài [233](233-quota-memory-cpu-namespace-vi.md):
  `requested: pods=1, used: pods=2, limited: pods=2` — mảnh `requested` là **một** Pod, vì
  controller xin từng Pod một chứ không xin cả ba cùng lúc.
- Mục *Lựa chọn loại tài nguyên*: cùng cơ chế này dùng được để giới hạn **số lượng của loại object
  khác**, và bài nêu ví dụ CronJob. Đó là cầu nối sang bài
  [252](252-quota-api-object-vi.md) — bài 3/7 của giai đoạn 25.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cú pháp cụ thể để đặt quota cho các loại object khác ngoài Pod | mục *Lựa chọn loại tài nguyên* chỉ nêu đúng một câu về khả năng đó, không đưa manifest | bài [252](252-quota-api-object-vi.md), bài 3/7 của [giai đoạn 25](00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace) |
| Cơ chế Deployment tự tạo lại Pod và ý nghĩa các field khác trong `status` của nó | Deployment và ReplicaSet là kiến thức của phần workload; ở đây chỉ cần đọc `availableReplicas` và `message` | bài [63](63-deployment-vi.md) đã đọc ở [giai đoạn 4](00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller) |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình cấm minikube, kind và cluster dùng chung | cluster VM ba node của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Mục *Tiếp theo* — hai danh sách *Dành cho quản trị viên cluster* và *Dành cho nhà phát triển ứng dụng* | là con trỏ, không có nội dung mới | bài [252](252-quota-api-object-vi.md) là bài kế của giai đoạn 25; nhánh nhà phát triển đã đọc ở [giai đoạn 3c](00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) |

---

Trang này hướng dẫn cách đặt hạn ngạch (quota) cho tổng số Pod có thể chạy trong một Namespace.
Bạn chỉ định quota trong một đối tượng
[ResourceQuota](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/resource-quota-v1/).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

Bạn phải có quyền tạo namespace trong cluster của mình.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi phần còn
lại của cluster.

```shell
kubectl create namespace quota-pod-example
```

## Tạo một ResourceQuota (Create a ResourceQuota)

Dưới đây là manifest mẫu cho một ResourceQuota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-demo
spec:
  hard:
    pods: "2"
```

Tạo ResourceQuota:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-pod.yaml --namespace=quota-pod-example
```

Xem thông tin chi tiết về ResourceQuota:

```shell
kubectl get resourcequota pod-demo --namespace=quota-pod-example --output=yaml
```

Kết quả cho thấy namespace có quota là hai Pod, và hiện tại chưa có Pod nào; nghĩa là chưa phần
nào của quota được sử dụng.

```yaml
spec:
  hard:
    pods: "2"
status:
  hard:
    pods: "2"
  used:
    pods: "0"
```

Dưới đây là manifest mẫu cho một Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-quota-demo
spec:
  selector:
    matchLabels:
      purpose: quota-demo
  replicas: 3
  template:
    metadata:
      labels:
        purpose: quota-demo
    spec:
      containers:
      - name: pod-quota-demo
        image: nginx
```

Trong manifest đó, `replicas: 3` yêu cầu Kubernetes cố gắng tạo ba Pod mới, tất cả cùng chạy
một ứng dụng.

Tạo Deployment:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-pod-deployment.yaml --namespace=quota-pod-example
```

Xem thông tin chi tiết về Deployment:

```shell
kubectl get deployment pod-quota-demo --namespace=quota-pod-example --output=yaml
```

Kết quả cho thấy mặc dù Deployment chỉ định ba replica, chỉ có hai Pod được tạo do quota mà bạn
đã định nghĩa trước đó:

```yaml
spec:
  ...
  replicas: 3
...
status:
  availableReplicas: 2
...
lastUpdateTime: 2021-04-02T20:57:05Z
    message: 'unable to create pods: pods "pod-quota-demo-1650323038-" is forbidden:
      exceeded quota: pod-demo, requested: pods=1, used: pods=2, limited: pods=2'
```

### Lựa chọn loại tài nguyên (Choice of resource)

Trong bài này bạn đã định nghĩa một ResourceQuota giới hạn tổng số Pod, nhưng bạn cũng có thể
giới hạn tổng số của các loại đối tượng khác. Ví dụ, bạn có thể quyết định giới hạn số lượng
CronJob được phép tồn tại trong một namespace.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace quota-pod-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình Memory Request và Limit mặc định cho một Namespace](232-memory-default-namespace-vi.md)

* [Cấu hình CPU Request và Limit mặc định cho một Namespace](230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc Memory tối thiểu và tối đa cho một Namespace](231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](229-cpu-constraint-namespace-vi.md)

* [Cấu hình Quota Memory và CPU cho một Namespace](233-quota-memory-cpu-namespace-vi.md)

* [Cấu hình Quota cho các đối tượng API](252-quota-api-object-vi.md)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên Memory cho Container và Pod](264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho Container và Pod](263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và memory ở mức Pod](265-assign-pod-level-resources-vi.md)

* [Cấu hình Quality of Service cho Pod](288-quality-service-pod-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 25:

1. Trên cluster lab, `lab-k8s-worker1` và `lab-k8s-worker2` còn thừa CPU lẫn RAM. Bạn tạo một
   namespace với `hard: pods: "2"` rồi apply Deployment `replicas: 3`. Bạn nhận được bao nhiêu
   Pod, và vì sao chỗ trống trên hai worker **không** cứu được replica thứ ba?
2. **Câu bẫy.** Deployment không đạt đủ replica, nên dễ kết luận "lệnh `kubectl apply` đã thất
   bại". Thực tế cái gì đã thành công, cái gì bị từ chối, và bạn phải nhìn vào đâu mới thấy lý do?
3. Thông báo ghi `requested: pods=1, used: pods=2, limited: pods=2`. Vì sao `requested` là **1**
   chứ không phải 3?
4. Mục *Lựa chọn loại tài nguyên* mở rộng cơ chế này ra khỏi Pod. Nó nói gì, và điều đó làm rõ
   thêm khác biệt nào giữa quota của trang này với quota memory/CPU ở bài
   [233](233-quota-memory-cpu-namespace-vi.md)?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Đúng hai Pod** — `status` của Deployment hiện `availableReplicas: 2` dù `replicas: 3`. Chỗ
   trống trên node không cứu được gì vì **ResourceQuota không đo dung lượng node**: quota này đếm
   **số object Pod trong namespace**, và trần là 2. Đây là ranh giới đáng nhớ — quota chặn ở khâu
   **API server chấp nhận object**, còn dung lượng node chỉ có tiếng nói ở khâu lập lịch, tức sau
   đó.
2. **Việc tạo Deployment thành công** — object Deployment được ghi vào cluster bình thường và
   `kubectl apply` không báo lỗi. Cái bị từ chối là **request tạo Pod thứ ba**, do controller của
   Deployment phát ra, chứ không phải lệnh của bạn. Vì vậy lý do không hiện trên terminal mà nằm
   trong **`status` của Deployment**: `availableReplicas: 2` và `message` ghi `unable to create
   pods: ... is forbidden: exceeded quota`. Đây là khác biệt so với bài
   [233](233-quota-memory-cpu-namespace-vi.md), nơi bạn tự tạo Pod nên nhận `Forbidden` ngay.
3. Vì controller **xin từng Pod một**, không xin gộp cả ba. Ở lần bị chặn, namespace đã dùng
   `pods=2` (bằng trần `limited: pods=2`) và request đang xét chỉ xin thêm **một** Pod nữa —
   `requested: pods=1`. Con số 3 là mong muốn ghi trong `spec.replicas` của Deployment, không phải
   nội dung của một request gửi tới API server.
4. Nó nói bạn **cũng có thể giới hạn tổng số của các loại đối tượng khác**, và nêu ví dụ giới hạn
   số **CronJob** được tồn tại trong một namespace. Điều đó làm rõ rằng ResourceQuota có **hai
   cách đo**: đo **lượng tài nguyên tiêu thụ** như bài 233 (`requests.memory`, `limits.cpu`…), và
   đo **số lượng object tồn tại** như trang này — cách thứ hai chặn được cả những object **không
   tiêu thụ CPU hay memory nào đáng kể**. Chi tiết cú pháp cho loại object khác nằm ở bài
   [252](252-quota-api-object-vi.md).

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
