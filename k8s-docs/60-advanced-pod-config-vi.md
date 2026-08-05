# Cấu hình Pod nâng cao (Advanced Pod Configuration)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/advanced-pod-config/>

Trang này đề cập đến các chủ đề cấu hình Pod nâng cao, bao gồm [PriorityClass](#priorityclasses),
[RuntimeClass](#runtimeclasses), [security context](#security-context) bên trong Pod,
và giới thiệu các khía cạnh của [việc lập lịch](https://kubernetes.io/docs/concepts/scheduling-eviction/#scheduling).

## PriorityClasses

_PriorityClass_ cho phép bạn thiết lập mức độ quan trọng của các Pod so với các Pod khác.
Nếu bạn gán một priority class cho một Pod, Kubernetes sẽ đặt trường `.spec.priority` cho Pod đó
dựa trên PriorityClass mà bạn đã chỉ định (bạn không thể đặt `.spec.priority` trực tiếp).
Nếu hoặc khi một Pod không thể được lập lịch, và vấn đề là do thiếu tài nguyên, kube-scheduler
sẽ cố gắng chiếm chỗ (preempt) các Pod có độ ưu tiên thấp hơn,
nhằm giúp việc lập lịch cho Pod có độ ưu tiên cao hơn trở nên khả thi.

PriorityClass là một đối tượng API ở phạm vi cluster (cluster-scoped), ánh xạ tên của một
priority class sang một giá trị độ ưu tiên dạng số nguyên. Số càng lớn thì độ ưu tiên càng cao.

### Định nghĩa một PriorityClass (Defining a PriorityClass)

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 10000
globalDefault: false
description: "Priority class for high-priority workloads"
```

### Chỉ định độ ưu tiên của Pod bằng PriorityClass (Specify pod priority using a PriorityClass)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  priorityClassName: high-priority
```

### Các PriorityClass có sẵn (Built-in PriorityClasses)

Kubernetes cung cấp sẵn hai PriorityClass:
- `system-cluster-critical`: Dành cho các thành phần hệ thống thiết yếu đối với cluster
- `system-node-critical`: Dành cho các thành phần hệ thống thiết yếu đối với từng node riêng lẻ. Đây là độ ưu tiên cao nhất mà một Pod có thể có trong Kubernetes.

Để biết thêm thông tin, hãy xem [Độ ưu tiên và chiếm chỗ của Pod](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/).

## RuntimeClasses

_RuntimeClass_ cho phép bạn chỉ định container runtime cấp thấp cho một Pod. Nó hữu ích khi bạn
muốn chỉ định các container runtime khác nhau cho các loại Pod khác nhau, chẳng hạn khi bạn cần
các mức độ cô lập (isolation) hoặc các tính năng runtime khác nhau.

### Pod ví dụ (Example Pod) {#runtimeclass-pod-example}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: myclass
  containers:
  - name: mycontainer
    image: nginx
```

[RuntimeClass](./43-runtime-class-vi.md) là một đối tượng ở phạm vi cluster, đại diện cho một
container runtime có sẵn trên một số hoặc tất cả các node của bạn.

Quản trị viên cluster là người cài đặt và cấu hình các runtime cụ thể đứng sau RuntimeClass.

Họ có thể thiết lập cấu hình container runtime đặc biệt đó trên tất cả các node, hoặc có thể
chỉ trên một số node.

Để biết thêm thông tin, hãy xem tài liệu về [RuntimeClass](./43-runtime-class-vi.md).

## Cấu hình security context ở cấp Pod và cấp container (Pod and container level security context configuration) {#security-context}

Trường `Security context` trong đặc tả (specification) của Pod cung cấp khả năng kiểm soát
chi tiết đối với các thiết lập bảo mật cho Pod và các container.

### `securityContext` cho toàn bộ Pod (Pod-wide `securityContext`) {#pod-level-security-context}

Một số khía cạnh bảo mật áp dụng cho toàn bộ Pod; với những khía cạnh khác,
bạn có thể muốn đặt một giá trị mặc định mà không có ghi đè nào ở cấp container.

Dưới đây là một ví dụ về việc dùng `securityContext` ở cấp Pod:

#### Pod ví dụ (Example Pod) {#pod-level-security-context-example}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:  # Áp dụng cho toàn bộ Pod
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: sec-ctx-demo
    image: registry.k8s.io/e2e-test-images/agnhost:2.45
    command: ["sh", "-c", "sleep 1h"]
```

### Security context ở cấp container (Container-level security context) {#container-level-security-context}

Bạn có thể chỉ định security context chỉ cho một container cụ thể.
Dưới đây là một ví dụ:

#### Pod ví dụ (Example Pod) {#container-level-security-context-example}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-2
spec:
  containers:
  - name: sec-ctx-demo-2
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
      seccompProfile:
        type: RuntimeDefault
```

### Các tùy chọn của security context (Security context options)

- **User ID và Group ID**: Kiểm soát container chạy dưới user/group nào
- **Capabilities**: Thêm hoặc loại bỏ các Linux capability
- **Seccomp Profiles**: Thiết lập các profile điện toán bảo mật (secure computing)
- **SELinux Options**: Cấu hình ngữ cảnh SELinux
- **AppArmor**: Cấu hình các profile AppArmor để kiểm soát truy cập bổ sung
- **Windows Options**: Cấu hình các thiết lập bảo mật dành riêng cho Windows

> **Thận trọng:**
> Bạn cũng có thể dùng `securityContext` của Pod để cho phép
> [_chế độ đặc quyền_ (privileged mode)](https://kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/#privileged-containers)
> trong các container Linux. Chế độ đặc quyền ghi đè nhiều thiết lập bảo mật khác trong `securityContext`.
> Tránh dùng thiết lập này trừ khi bạn không thể cấp các quyền tương đương bằng cách dùng các trường khác trong `securityContext`.
> Bạn có thể chạy các container Windows ở một chế độ đặc quyền tương tự bằng cách đặt cờ
> `windowsOptions.hostProcess` trên security context ở cấp Pod. Để biết chi tiết và hướng dẫn, hãy xem
> [Tạo một Windows HostProcess Pod](https://kubernetes.io/docs/tasks/configure-pod-container/create-hostprocess-pod/).

Để biết thêm thông tin, hãy xem [Cấu hình Security Context cho Pod hoặc Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/).

## Tác động đến quyết định lập lịch Pod (Influencing Pod scheduling decisions) {#scheduling}

Kubernetes cung cấp nhiều cơ chế để kiểm soát việc các Pod của bạn được lập lịch lên node nào.

### Node selector (Node selectors)

Dạng ràng buộc chọn node đơn giản nhất:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    disktype: ssd
```

### Node affinity

Node affinity cho phép bạn chỉ định các quy tắc ràng buộc việc Pod của bạn có thể được lập lịch
lên những node nào. Dưới đây là ví dụ về một Pod ưu tiên chạy trên các node được gắn label
thuộc một châu lục cụ thể, lựa chọn dựa trên giá trị của label
[`topology.kubernetes.io/zone`](https://kubernetes.io/docs/reference/labels-annotations-taints/#topologykubernetesiozone).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
  containers:
  - name: with-node-affinity
    image: registry.k8s.io/pause:3.8
```

### Pod affinity và anti-affinity (Pod affinity and anti-affinity)

Bên cạnh node affinity, bạn cũng có thể ràng buộc việc một Pod có thể được lập lịch lên những node
nào dựa trên label của _các Pod khác_ đang chạy sẵn trên các node. Pod affinity cho phép bạn chỉ
định các quy tắc về vị trí mà một Pod nên được đặt so với các Pod khác.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - database
        topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: registry.k8s.io/pause:3.8
```

### Toleration (Tolerations)

_Toleration_ cho phép Pod được lập lịch lên các node có taint tương ứng:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: myapp
    image: nginx
  tolerations:
  - key: "key"
    operator: "Equal"
    value: "value"
    effect: "NoSchedule"
```

Để biết thêm thông tin, hãy xem [Gán Pod cho Node](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/).

## Pod overhead

Pod overhead cho phép bạn tính đến phần tài nguyên mà hạ tầng của Pod tiêu thụ, cộng thêm
vào các request và limit của container.

```yaml
---
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kvisor-runtime
handler: kvisor-runtime
overhead:
  podFixed:
    memory: "2Gi"
    cpu: "500m"
---
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  runtimeClassName: kvisor-runtime
  containers:
  - name: myapp
    image: nginx
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

## Tiếp theo (What's next)

* Đọc về [Độ ưu tiên và chiếm chỗ của Pod](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
* Đọc về [RuntimeClass](./43-runtime-class-vi.md)
* Khám phá [Cấu hình Security Context cho Pod hoặc Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
* Tìm hiểu cách Kubernetes [gán Pod cho Node](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
* [Pod Overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
