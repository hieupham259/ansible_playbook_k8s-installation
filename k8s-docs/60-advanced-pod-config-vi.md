# Cấu hình Pod nâng cao (Advanced Pod Configuration)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/advanced-pod-config/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài 11/11 · Kiểm chứng
ở Lab 3a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này là **trang giới thiệu, không phải trang dạy**: mỗi mục chỉ có một đoạn và một ví dụ YAML
rồi trỏ sang bài chuyên đề ở giai đoạn 7 và 9. Nhiệm vụ của lần đọc này là **nhận mặt các trường**
để khi gặp trong manifest thật bạn biết chúng thuộc cơ chế nào — chứ không phải nắm cú pháp đầy
đủ của node affinity hay seccomp.

**Phải hiểu ở lần đọc này:**

- PriorityClass là đối tượng ở **phạm vi cluster**, ánh xạ tên sang một số nguyên; bạn gán bằng
  `priorityClassName` và Kubernetes tự đặt `.spec.priority` — **bạn không đặt `.spec.priority`
  trực tiếp**. Khi Pod không lập lịch được vì thiếu tài nguyên, scheduler chiếm chỗ Pod có độ ưu
  tiên thấp hơn. Hai class có sẵn: `system-cluster-critical` và `system-node-critical`.
- RuntimeClass chọn container runtime cấp thấp cho Pod qua `runtimeClassName`. Nó cũng ở phạm vi
  cluster và chỉ **đại diện** cho một runtime có sẵn trên **một số hoặc tất cả** node — quản trị
  viên cluster mới là người cài và cấu hình runtime đó.
- `securityContext` có hai cấp: cấp Pod áp cho toàn bộ Pod hoặc đặt mặc định, cấp container chỉ
  áp cho một container. Chế độ đặc quyền **ghi đè nhiều thiết lập bảo mật khác** trong
  `securityContext`, nên tránh dùng khi còn cách khác.
- Bốn cơ chế tác động lập lịch và điểm khác nhau về **thứ mà chúng nhìn vào**: `nodeSelector` và
  node affinity nhìn **label của node**; pod affinity/anti-affinity nhìn **label của các Pod khác**
  đang chạy, kèm `topologyKey`; toleration cho phép Pod lên node **có taint** tương ứng.
- Pod overhead khai trong RuntimeClass bằng `overhead.podFixed`, và được **cộng thêm** vào request
  và limit của container để tính phần tài nguyên mà hạ tầng của Pod tiêu thụ.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cơ chế chiếm chỗ đứng sau PriorityClass | là chủ đề riêng của phần lập lịch | giai đoạn 7 — bài [141](141-pod-priority-preemption-vi.md) |
| Cấu hình runtime thật đứng sau RuntimeClass | lý thuyết đã đọc ở giai đoạn 2 | bài [43](43-runtime-class-vi.md) |
| *Các tùy chọn của security context* — capabilities, seccomp, SELinux, AppArmor | thuộc phần bảo mật | giai đoạn 9 — bài [127](127-linux-kernel-security-vi.md) |
| `windowsOptions.hostProcess` và các tùy chọn Windows | cluster lab chỉ có node Linux | giai đoạn 15 |
| Cú pháp đầy đủ của node affinity, pod affinity, taint và toleration | ở đây chỉ là ví dụ tối thiểu | giai đoạn 7 — bài [138](138-assign-pod-node-vi.md), [139](139-taint-and-toleration-vi.md), [140](140-topology-spread-constraints-vi.md) |
| *Pod overhead* — cách nó vào quota và eviction | cần requests/limits và QoS trước | giai đoạn 7 — bài [144](144-pod-overhead-vi.md) |

---

Trang này đề cập đến các chủ đề cấu hình Pod nâng cao, bao gồm [PriorityClass](#priorityclasses),
[RuntimeClass](#runtimeclasses), [security context](#security-context) bên trong Pod,
và giới thiệu các khía cạnh của [việc lập lịch](136-scheduling-eviction-vi.md#scheduling).

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

Để biết thêm thông tin, hãy xem [Độ ưu tiên và chiếm chỗ của Pod](141-pod-priority-preemption-vi.md).

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
> [_chế độ đặc quyền_ (privileged mode)](127-linux-kernel-security-vi.md#privileged-containers)
> trong các container Linux. Chế độ đặc quyền ghi đè nhiều thiết lập bảo mật khác trong `securityContext`.
> Tránh dùng thiết lập này trừ khi bạn không thể cấp các quyền tương đương bằng cách dùng các trường khác trong `securityContext`.
> Bạn có thể chạy các container Windows ở một chế độ đặc quyền tương tự bằng cách đặt cờ
> `windowsOptions.hostProcess` trên security context ở cấp Pod. Để biết chi tiết và hướng dẫn, hãy xem
> [Tạo một Windows HostProcess Pod](281-create-hostprocess-pod-vi.md).

Để biết thêm thông tin, hãy xem [Cấu hình Security Context cho Pod hoặc Container](291-security-context-vi.md).

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

Để biết thêm thông tin, hãy xem [Gán Pod cho Node](138-assign-pod-node-vi.md).

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

* Đọc về [Độ ưu tiên và chiếm chỗ của Pod](141-pod-priority-preemption-vi.md)
* Đọc về [RuntimeClass](./43-runtime-class-vi.md)
* Khám phá [Cấu hình Security Context cho Pod hoặc Container](291-security-context-vi.md)
* Tìm hiểu cách Kubernetes [gán Pod cho Node](138-assign-pod-node-vi.md)
* [Pod Overhead](144-pod-overhead-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Bạn viết `spec.priority: 10000` thẳng trong manifest Pod. Cách đó có đúng không, và cách đúng
   là gì?
2. `k8s-master` bị taint nên Pod thường không lên đó. Cơ chế nào trong bài cho phép một Pod vẫn
   được lập lịch lên node đó, và nó khác `nodeSelector` ở chỗ nào?
3. Node affinity và pod affinity dựa trên label của cái gì? Pod affinity còn cần thêm trường nào?
4. Bạn đặt `runtimeClassName: myclass` cho Pod nhưng chưa ai cài runtime đó. Theo bài, ai chịu
   trách nhiệm phần đó, và điều gì có thể khác nhau giữa các node?
5. Cùng một thiết lập bảo mật, đặt ở `securityContext` cấp Pod và cấp container thì khác nhau thế
   nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Sai — bạn không thể đặt `.spec.priority` trực tiếp.** Cách đúng là tạo một **PriorityClass**
   (đối tượng ở phạm vi cluster, ánh xạ tên sang giá trị số nguyên) rồi gán cho Pod bằng
   **`priorityClassName`**; Kubernetes sẽ tự đặt `.spec.priority` theo class đó. Số càng lớn thì
   độ ưu tiên càng cao, và độ ưu tiên chỉ có tác dụng khi Pod không lập lịch được vì thiếu tài
   nguyên — khi đó kube-scheduler chiếm chỗ các Pod có độ ưu tiên thấp hơn.
2. **Toleration** — nó cho phép Pod được lập lịch lên các node **có taint tương ứng**. Khác biệt
   với `nodeSelector` nằm ở bản chất của hai cơ chế: `nodeSelector` là **ràng buộc chọn node**,
   bạn nói Pod chỉ được lên node mang label nào; toleration **không chọn gì cả**, nó chỉ **gỡ bỏ
   một rào cản** do node dựng lên. Có toleration không có nghĩa Pod sẽ lên node đó, chỉ có nghĩa
   là nó không còn bị taint đó chặn.
3. **Node affinity dựa trên label của node** — ví dụ trong bài dùng
   `topology.kubernetes.io/zone`. **Pod affinity và anti-affinity dựa trên label của *các Pod
   khác*** đang chạy sẵn trên các node, tức là bạn ràng buộc vị trí của Pod này so với các Pod
   khác. Vì vậy pod affinity cần thêm **`topologyKey`**: phải nói rõ "cùng chỗ" nghĩa là cùng
   node, cùng zone, hay cùng đơn vị topology nào.
4. **Quản trị viên cluster** là người cài đặt và cấu hình các runtime cụ thể đứng sau RuntimeClass.
   Điều có thể khác nhau giữa các node chính là **runtime đó có mặt ở đâu**: quản trị viên có thể
   thiết lập cấu hình container runtime đặc biệt đó **trên tất cả các node, hoặc chỉ trên một số
   node** — RuntimeClass chỉ **đại diện** cho một runtime có sẵn trên một số hoặc tất cả node chứ
   không tự cài gì.
5. `securityContext` **cấp Pod áp dụng cho toàn bộ Pod**, và dùng cho hai mục đích: những khía
   cạnh bảo mật vốn thuộc về cả Pod, và những giá trị bạn muốn đặt làm **mặc định khi không có ghi
   đè ở cấp container**. `securityContext` **cấp container chỉ định cho đúng một container cụ
   thể**. Trong ví dụ của bài, cấp Pod đặt `runAsUser`, `runAsGroup`, `fsGroup`, còn cấp container
   đặt những thứ chỉ có nghĩa với riêng container đó như `allowPrivilegeEscalation`,
   `capabilities.drop` và `seccompProfile`.

</details>

Đây là bài cuối của nhóm **3a — Pod và vòng đời**. Trả lời được câu hỏi của cả mười một bài thì
bạn sẵn sàng vào Lab 3a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)); câu nào còn
hụt thì quay lại đúng bài tương ứng trước.
