# Tự động co giãn DNS Service trong một cluster (Autoscale the DNS Service in a Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/dns-horizontal-autoscaling/>
>
> Trang này hướng dẫn cách bật và cấu hình tính năng tự động co giãn (autoscaling) cho DNS
> service trong cluster Kubernetes của bạn.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối — CP6 DNS, CNI và kube-proxy](00-ALO-TRINH-ADMIN.md#cp6--dns-cni-và-kube-proxy),
bài 4/7 · thực hành trực tiếp trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

Đừng nhầm bài này với HPA (Horizontal Pod Autoscaler) đã học ở bài
[72](72-horizontal-pod-autoscale-vi.md): công cụ ở đây là `cluster-proportional-autoscaler`,
co giãn theo **kích thước cluster** (số node, số core) chứ không theo mức tiêu thụ tài nguyên,
và không cần metrics-server.

**Phải hiểu ở lần đọc này:**

- `kube-dns-autoscaler` là một Deployment triển khai **tách biệt** với DNS service; Pod của nó
  poll API server lấy số node và số core rồi tự tính số replica mong muốn cho DNS backend —
  khác về bản chất với HPA vốn dựa trên metric tiêu thụ.
- Cách xác định **scale target**: tìm Deployment DNS qua label `k8s-app=kube-dns` (CoreDNS
  vẫn giữ label này để tương thích với các cluster từng dùng kube-dns), kết quả có dạng
  `Deployment/<tên-deployment>`.
- Công thức tính số replica ở chế độ `linear`:
  `replicas = max( ceil( cores × 1/coresPerReplica ) , ceil( nodes × 1/nodesPerReplica ) )`
  — node nhiều core thì `coresPerReplica` chiếm ưu thế, node ít core thì `nodesPerReplica`
  chiếm ưu thế.
- Tinh chỉnh tham số bằng cách sửa ConfigMap `kube-dns-autoscaler`; autoscaler tự làm mới bảng
  tham số mỗi chu kỳ poll, **không cần build lại hay khởi động lại** Pod autoscaler.
- Ba cách tắt autoscaling và điều kiện áp dụng: scale Deployment autoscaler về 0 (luôn dùng
  được), xóa Deployment (chỉ khi không có gì tự tạo lại nó), xóa file manifest trên node
  master (chỉ khi cluster dùng Addon Manager).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Cách 3: xóa file manifest khỏi node master* | chỉ áp dụng cho cluster do Addon Manager (đã deprecated) quản lý; cluster kubeadm của Lab 00 không dùng cơ chế này | ngoài phạm vi lộ trình — chỉ cần khi vận hành cluster cũ dùng Addon Manager |
| Chế độ điều khiển *ladder* và các mẫu co giãn khác | bài chỉ nhắc tên, không trình bày | tài liệu [cluster-proportional-autoscaler](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler) trên GitHub khi bạn cần mẫu co giãn theo bậc |

---

Trang này hướng dẫn cách bật và cấu hình tính năng tự động co giãn (autoscaling) cho DNS
service trong cluster Kubernetes của bạn.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
  tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
  không đóng vai trò host của control plane. Nếu bạn chưa có cluster, bạn có thể tạo một
  cluster bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng
  một trong các sân chơi (playground) Kubernetes sau:

  * [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  * [KodeKloud](https://kodekloud.com/public-playgrounds)

  Để kiểm tra phiên bản, nhập `kubectl version`.

* Hướng dẫn này giả định các node của bạn dùng kiến trúc CPU AMD64 hoặc Intel 64.

* Hãy chắc chắn rằng [Kubernetes DNS](10-dns-pod-service-vi.md) đã được bật.

## Xác định xem DNS horizontal autoscaling đã được bật hay chưa (Determine whether DNS horizontal autoscaling is already enabled) {#determining-whether-dns-horizontal-autoscaling-is-already-enabled}

Liệt kê các Deployment trong namespace kube-system của cluster:

```shell
kubectl get deployment --namespace=kube-system
```

Output sẽ tương tự thế này:

```
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
...
kube-dns-autoscaler    1/1     1            1           ...
...
```

Nếu bạn thấy "kube-dns-autoscaler" trong output, nghĩa là DNS horizontal autoscaling đã được
bật, và bạn có thể bỏ qua tới mục
[Tinh chỉnh các tham số autoscaling](#tuning-autoscaling-parameters).

## Lấy tên Deployment DNS của bạn (Get the name of your DNS Deployment) {#find-scaling-target}

Liệt kê các deployment DNS trong namespace kube-system của cluster:

```shell
kubectl get deployment -l k8s-app=kube-dns --namespace=kube-system
```

Output sẽ tương tự thế này:

```
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
...
coredns   2/2     2            2           ...
...
```

Nếu bạn không thấy Deployment nào cho DNS service, bạn cũng có thể tìm theo tên:

```shell
kubectl get deployment --namespace=kube-system
```

và tìm một deployment có tên `coredns` hoặc `kube-dns`.

Scale target (đích co giãn) của bạn là

```
Deployment/<tên-deployment-của-bạn>
```

trong đó `<tên-deployment-của-bạn>` là tên Deployment DNS của bạn. Ví dụ, nếu Deployment cho
DNS của bạn tên là coredns, thì scale target là Deployment/coredns.

> **Ghi chú:** CoreDNS là DNS service mặc định của Kubernetes. CoreDNS đặt label
> `k8s-app=kube-dns` để có thể hoạt động trong các cluster vốn ban đầu dùng kube-dns.

## Bật DNS horizontal autoscaling (Enable DNS horizontal autoscaling) {#enablng-dns-horizontal-autoscaling}

Trong phần này, bạn tạo một Deployment mới. Các Pod trong Deployment chạy một container dựa
trên image `cluster-proportional-autoscaler-amd64`.

Tạo một file tên `dns-horizontal-autoscaler.yaml` với nội dung sau:

```yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: kube-dns-autoscaler
  namespace: kube-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:kube-dns-autoscaler
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["list", "watch"]
  - apiGroups: [""]
    resources: ["replicationcontrollers/scale"]
    verbs: ["get", "update"]
  - apiGroups: ["apps"]
    resources: ["deployments/scale", "replicasets/scale"]
    verbs: ["get", "update"]
# Xóa rule configmaps khi issue dưới đây được sửa:
# kubernetes-incubator/cluster-proportional-autoscaler#16
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "create"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: system:kube-dns-autoscaler
subjects:
  - kind: ServiceAccount
    name: kube-dns-autoscaler
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: system:kube-dns-autoscaler
  apiGroup: rbac.authorization.k8s.io

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-dns-autoscaler
  namespace: kube-system
  labels:
    k8s-app: kube-dns-autoscaler
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    matchLabels:
      k8s-app: kube-dns-autoscaler
  template:
    metadata:
      labels:
        k8s-app: kube-dns-autoscaler
    spec:
      priorityClassName: system-cluster-critical
      securityContext:
        seccompProfile:
          type: RuntimeDefault
        supplementalGroups: [ 65534 ]
        fsGroup: 65534
      nodeSelector:
        kubernetes.io/os: linux
      containers:
      - name: autoscaler
        image: registry.k8s.io/cpa/cluster-proportional-autoscaler:1.8.4
        resources:
            requests:
                cpu: "20m"
                memory: "10Mi"
        command:
          - /cluster-proportional-autoscaler
          - --namespace=kube-system
          - --configmap=kube-dns-autoscaler
          # Nên giữ target đồng bộ với cluster/addons/dns/kube-dns.yaml.base
          - --target=<SCALE_TARGET>
          # Khi cluster dùng các node lớn (nhiều core), "coresPerReplica" nên chiếm ưu thế.
          # Nếu dùng các node nhỏ, "nodesPerReplica" nên chiếm ưu thế.
          - --default-params={"linear":{"coresPerReplica":256,"nodesPerReplica":16,"preventSinglePointFailure":true,"includeUnschedulableNodes":true}}
          - --logtostderr=true
          - --v=2
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
      serviceAccountName: kube-dns-autoscaler
```

Trong file này, thay `<SCALE_TARGET>` bằng scale target của bạn.

Đi tới thư mục chứa file cấu hình và nhập lệnh sau để tạo Deployment:

```shell
kubectl apply -f dns-horizontal-autoscaler.yaml
```

Output của một lệnh chạy thành công là:

```
deployment.apps/kube-dns-autoscaler created
```

DNS horizontal autoscaling giờ đã được bật.

## Tinh chỉnh các tham số DNS autoscaling (Tune DNS autoscaling parameters) {#tuning-autoscaling-parameters}

Xác nhận ConfigMap kube-dns-autoscaler tồn tại:

```shell
kubectl get configmap --namespace=kube-system
```

Output sẽ tương tự thế này:

```
NAME                  DATA      AGE
...
kube-dns-autoscaler   1         ...
...
```

Sửa dữ liệu trong ConfigMap:

```shell
kubectl edit configmap kube-dns-autoscaler --namespace=kube-system
```

Tìm dòng sau:

```yaml
linear: '{"coresPerReplica":256,"min":1,"nodesPerReplica":16}'
```

Sửa các trường theo nhu cầu của bạn. Trường "min" cho biết số lượng DNS backend tối thiểu. Số
lượng backend thực tế được tính bằng phương trình:

```
replicas = max( ceil( cores × 1/coresPerReplica ) , ceil( nodes × 1/nodesPerReplica ) )
```

Lưu ý rằng giá trị của cả `coresPerReplica` lẫn `nodesPerReplica` đều là số thực (float).

Ý tưởng ở đây là: khi cluster dùng các node có nhiều core, `coresPerReplica` sẽ chiếm ưu thế.
Khi cluster dùng các node có ít core hơn, `nodesPerReplica` sẽ chiếm ưu thế.

Còn có các mẫu co giãn (scaling pattern) khác được hỗ trợ. Xem chi tiết tại
[cluster-proportional-autoscaler](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler).

## Tắt DNS horizontal autoscaling (Disable DNS horizontal autoscaling)

Có một vài lựa chọn để tinh chỉnh DNS horizontal autoscaling. Dùng lựa chọn nào phụ thuộc vào
các điều kiện khác nhau.

### Cách 1: Scale deployment kube-dns-autoscaler xuống 0 replica (Option 1: Scale down the kube-dns-autoscaler deployment to 0 replicas)

Cách này áp dụng được cho mọi tình huống. Nhập lệnh sau:

```shell
kubectl scale deployment --replicas=0 kube-dns-autoscaler --namespace=kube-system
```

Output là:

```
deployment.apps/kube-dns-autoscaler scaled
```

Xác nhận số replica bằng không:

```shell
kubectl get rs --namespace=kube-system
```

Output hiển thị 0 ở các cột DESIRED và CURRENT:

```
NAME                                  DESIRED   CURRENT   READY   AGE
...
kube-dns-autoscaler-6b59789fc8        0         0         0       ...
...
```

### Cách 2: Xóa deployment kube-dns-autoscaler (Option 2: Delete the kube-dns-autoscaler deployment)

Cách này áp dụng được nếu kube-dns-autoscaler nằm dưới quyền kiểm soát của chính bạn, nghĩa
là sẽ không có ai tạo lại nó:

```shell
kubectl delete deployment kube-dns-autoscaler --namespace=kube-system
```

Output là:

```
deployment.apps "kube-dns-autoscaler" deleted
```

### Cách 3: Xóa file manifest của kube-dns-autoscaler khỏi node master (Option 3: Delete the kube-dns-autoscaler manifest file from the master node)

Cách này áp dụng được nếu kube-dns-autoscaler nằm dưới quyền kiểm soát của
[Addon Manager](https://git.k8s.io/kubernetes/cluster/addons/README.md) (đã deprecated), và
bạn có quyền ghi trên node master.

Đăng nhập vào node master và xóa file manifest tương ứng. Đường dẫn phổ biến cho
kube-dns-autoscaler là:

```
/etc/kubernetes/addons/dns-horizontal-autoscaler/dns-horizontal-autoscaler.yaml
```

Sau khi file manifest bị xóa, Addon Manager sẽ xóa Deployment kube-dns-autoscaler.

## Tìm hiểu cách DNS horizontal autoscaling hoạt động (Understanding how DNS horizontal autoscaling works)

* Ứng dụng cluster-proportional-autoscaler được triển khai tách biệt với DNS service.

* Một Pod autoscaler chạy một client thực hiện poll (truy vấn định kỳ) Kubernetes API server
  để lấy số lượng node và core trong cluster.

* Số replica mong muốn được tính toán và áp dụng cho các DNS backend dựa trên số node và core
  hiện có thể lập lịch (schedulable) cùng các tham số co giãn đã cho.

* Các tham số co giãn và điểm dữ liệu được cung cấp cho autoscaler qua một ConfigMap, và nó
  làm mới bảng tham số của mình sau mỗi chu kỳ poll để luôn cập nhật với các tham số co giãn
  mong muốn mới nhất.

* Bạn được phép thay đổi các tham số co giãn mà không cần build lại hay khởi động lại Pod
  autoscaler.

* Autoscaler cung cấp một giao diện controller hỗ trợ hai mẫu điều khiển: *linear* (tuyến
  tính) và *ladder* (bậc thang).

## Tiếp theo (What's next)

* Đọc về [Đảm bảo lập lịch cho các Add-On Pod quan trọng (Guaranteed Scheduling For Critical Add-On Pods)](210-guaranteed-scheduling-critical-addon-pods-vi.md).
* Tìm hiểu thêm về
  [cách hiện thực cluster-proportional-autoscaler](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu dưới đây mà không nhìn lại bài là đủ cho lần đọc ở checkpoint này.

1. `kube-dns-autoscaler` quyết định số replica của CoreDNS dựa trên những con số nào, và điều
   đó khác gì với cách HPA quyết định số replica?
2. Cluster lab của bạn có 3 node (1 master + 2 worker), mỗi node 2 core. Với tham số
   `linear: '{"coresPerReplica":256,"min":1,"nodesPerReplica":16}'`, autoscaler sẽ đặt bao
   nhiêu replica cho CoreDNS? Nêu cách tính.
3. Bạn vừa sửa `nodesPerReplica` trong ConfigMap `kube-dns-autoscaler`. Có cần khởi động lại
   Pod autoscaler để thay đổi có hiệu lực không? Vì sao?
4. Bạn muốn tạm tắt DNS autoscaling trên một cluster mà bạn không chắc có hệ thống nào khác sẽ
   tự tạo lại Deployment autoscaler hay không. Trong ba cách tắt, cách nào an toàn cho tình
   huống này và vì sao hai cách còn lại không phù hợp?
5. Cluster của bạn chạy CoreDNS chứ không phải kube-dns, vậy tại sao lệnh
   `kubectl get deployment -l k8s-app=kube-dns --namespace=kube-system` vẫn tìm ra nó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Autoscaler poll API server lấy **số node và số core có thể lập lịch trong cluster**, rồi
   tính số replica mong muốn theo các tham số co giãn trong ConfigMap. Khác biệt cốt lõi với
   HPA: **co giãn theo kích thước cluster, không theo mức tiêu thụ tài nguyên** (CPU/memory)
   của workload — nên nó không cần metrics-server.
2. Áp công thức `replicas = max( ceil( cores × 1/coresPerReplica ) , ceil( nodes ×
   1/nodesPerReplica ) )`: cores = 6 → ceil(6/256) = 1; nodes = 3 → ceil(3/16) = 1. Kết quả
   **max(1, 1) = 1 replica** (thỏa `min` = 1).
3. **Không cần.** Bài nói rõ autoscaler làm mới bảng tham số của nó sau **mỗi chu kỳ poll** để
   cập nhật tham số mới nhất; thay đổi tham số co giãn được phép thực hiện mà không cần build
   lại hay khởi động lại Pod autoscaler.
4. Dùng **Cách 1 — scale Deployment autoscaler về 0 replica**, vì bài khẳng định cách này
   "áp dụng được cho mọi tình huống". Cách 2 (xóa Deployment) chỉ phù hợp khi **không có ai
   tạo lại nó** — điều bạn đang không chắc. Cách 3 chỉ áp dụng khi autoscaler do Addon Manager
   (đã deprecated) quản lý và bạn có quyền ghi trên node master.
5. Vì **CoreDNS chủ động đặt label `k8s-app=kube-dns`** để có thể hoạt động trong các cluster
   vốn ban đầu dùng kube-dns. Đây là điểm dễ nhầm: label mô tả vai trò DNS mặc định chứ không
   phản ánh tên phần mềm đang chạy.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
