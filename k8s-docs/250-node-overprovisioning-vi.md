# Cấp phát dư dung lượng Node cho Cluster (Overprovision Node Capacity For A Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/node-overprovisioning/

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
