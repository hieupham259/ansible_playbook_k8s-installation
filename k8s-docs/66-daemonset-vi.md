# DaemonSet

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/>
>
> DaemonSet định nghĩa các Pod cung cấp những tiện ích cục bộ trên node (node-local
> facilities). Đó có thể là các thành phần thiết yếu cho hoạt động của cluster, chẳng hạn
> một công cụ hỗ trợ mạng, hoặc là một phần của add-on.

Một _DaemonSet_ đảm bảo rằng tất cả (hoặc một số) Node chạy một bản sao của một Pod. Khi
các node được thêm vào cluster, Pod cũng được thêm vào các node đó. Khi các node bị gỡ
khỏi cluster, những Pod này sẽ bị thu gom rác (garbage collected). Xóa một DaemonSet sẽ
dọn dẹp các Pod mà nó đã tạo.

Một số trường hợp sử dụng điển hình của DaemonSet là:

- chạy một daemon lưu trữ của cluster trên mọi node
- chạy một daemon thu thập log trên mọi node
- chạy một daemon giám sát node trên mọi node

Trong trường hợp đơn giản, một DaemonSet bao phủ tất cả các node sẽ được dùng cho mỗi loại
daemon. Một cấu hình phức tạp hơn có thể dùng nhiều DaemonSet cho cùng một loại daemon,
nhưng với các flag khác nhau và/hoặc yêu cầu memory và cpu khác nhau cho các loại phần
cứng khác nhau.

## Viết một Spec cho DaemonSet (Writing a DaemonSet Spec)

### Tạo một DaemonSet (Create a DaemonSet)

Bạn có thể mô tả một DaemonSet trong một file YAML. Ví dụ, file `daemonset.yaml` dưới đây
mô tả một DaemonSet chạy Docker image fluentd-elasticsearch:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # các toleration này để daemonset có thể chạy được trên các control plane node
      # hãy xóa chúng nếu control plane node của bạn không nên chạy pod
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v5.0.1
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      # có thể nên đặt một priority class cao để đảm bảo Pod của DaemonSet
      # chiếm chỗ (preempt) các Pod đang chạy
      # priorityClassName: important
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

Tạo một DaemonSet dựa trên file YAML:

```
kubectl apply -f https://k8s.io/examples/controllers/daemonset.yaml
```

### Các trường bắt buộc (Required Fields)

Giống như mọi cấu hình Kubernetes khác, một DaemonSet cần các trường `apiVersion`, `kind`,
và `metadata`. Để biết thông tin chung về cách làm việc với các file cấu hình, xem
[chạy ứng dụng stateless](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
và [quản lý object bằng kubectl](./27-object-management-vi.md).

Tên của một object DaemonSet phải là một
[tên DNS subdomain](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)
hợp lệ.

Một DaemonSet cũng cần một phần
[`.spec`](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status).

### Pod Template

`.spec.template` là một trong các trường bắt buộc trong `.spec`.

`.spec.template` là một [pod template](https://kubernetes.io/docs/concepts/workloads/pods/#pod-templates).
Nó có schema giống hệt một Pod, ngoại trừ việc nó được lồng bên trong và không có
`apiVersion` hay `kind`.

Bên cạnh các trường bắt buộc của một Pod, một Pod template trong DaemonSet phải chỉ định
các label phù hợp (xem [bộ chọn pod](#pod-selector)).

Một Pod Template trong DaemonSet phải có
[`RestartPolicy`](./47-pod-lifecycle-vi.md#restart-policy) bằng `Always`, hoặc không chỉ
định — khi đó giá trị mặc định là `Always`.

### Bộ chọn Pod (Pod Selector) {#pod-selector}

Trường `.spec.selector` là một bộ chọn pod (pod selector). Nó hoạt động giống như
`.spec.selector` của một [Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/).

Bạn phải chỉ định một pod selector khớp với các label của `.spec.template`.
Ngoài ra, một khi DaemonSet đã được tạo, `.spec.selector` của nó không thể bị thay đổi
(mutate). Việc thay đổi pod selector có thể dẫn đến việc vô tình bỏ rơi (orphan) các Pod,
và điều này từng được nhận thấy là gây bối rối cho người dùng.

`.spec.selector` là một object gồm hai trường:

* `matchLabels` - hoạt động giống như `.spec.selector` của một
  [ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/).
* `matchExpressions` - cho phép xây dựng các selector phức tạp hơn bằng cách chỉ định
  key, danh sách các giá trị, và một toán tử liên hệ giữa key và các giá trị đó.

Khi cả hai được chỉ định, kết quả là phép AND của chúng.

`.spec.selector` phải khớp với `.spec.template.metadata.labels`.
Cấu hình có hai trường này không khớp nhau sẽ bị API từ chối.

### Chạy Pod trên một số Node được chọn (Running Pods on select Nodes)

Nếu bạn chỉ định `.spec.template.spec.nodeSelector`, thì DaemonSet controller sẽ tạo Pod
trên các node khớp với [node selector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
đó. Tương tự, nếu bạn chỉ định `.spec.template.spec.affinity`, thì DaemonSet controller
sẽ tạo Pod trên các node khớp với
[node affinity](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
đó. Nếu bạn không chỉ định cái nào cả, thì DaemonSet controller sẽ tạo Pod trên tất cả
các node.

## Cách các Daemon Pod được lập lịch (How Daemon Pods are scheduled)

Một DaemonSet có thể được dùng để đảm bảo rằng tất cả các node đủ điều kiện đều chạy một
bản sao của một Pod. DaemonSet controller tạo một Pod cho mỗi node đủ điều kiện và thêm
trường `spec.affinity.nodeAffinity` của Pod để khớp với host mục tiêu. Sau khi Pod được
tạo, scheduler mặc định thường sẽ tiếp quản và gắn (bind) Pod vào host mục tiêu bằng cách
đặt trường `.spec.nodeName`. Nếu Pod mới không thể vừa trên node, scheduler mặc định có
thể chiếm chỗ (preempt) — trục xuất (evict) — một số Pod hiện có dựa trên
[độ ưu tiên (priority)](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#pod-priority)
của Pod mới.

> **Ghi chú:**
> Nếu việc pod của DaemonSet phải chạy trên từng node là quan trọng, thường nên đặt
> `.spec.template.spec.priorityClassName` của DaemonSet thành một
> [PriorityClass](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#priorityclass)
> có độ ưu tiên cao hơn để đảm bảo việc trục xuất này diễn ra.

Người dùng có thể chỉ định một scheduler khác cho các Pod của DaemonSet, bằng cách đặt
trường `.spec.template.spec.schedulerName` của DaemonSet.

Node affinity gốc được chỉ định tại trường `.spec.template.spec.affinity.nodeAffinity`
(nếu có) sẽ được DaemonSet controller xem xét khi đánh giá các node đủ điều kiện, nhưng
trên Pod được tạo ra, nó bị thay thế bằng node affinity khớp với tên của node đủ điều
kiện.

```yaml
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchFields:
      - key: metadata.name
        operator: In
        values:
        - target-host-name
```

### Taint và toleration (Taints and tolerations)

DaemonSet controller tự động thêm một tập các toleration vào các Pod của DaemonSet:

*Bảng: Các toleration cho pod của DaemonSet*

| Key của toleration | Effect | Chi tiết |
| ------------------- | ------ | -------- |
| [`node.kubernetes.io/not-ready`](https://kubernetes.io/docs/reference/labels-annotations-taints/#node-kubernetes-io-not-ready) | `NoExecute` | Pod của DaemonSet có thể được lập lịch lên các node không khỏe mạnh hoặc chưa sẵn sàng nhận Pod. Mọi Pod DaemonSet đang chạy trên các node như vậy sẽ không bị trục xuất. |
| [`node.kubernetes.io/unreachable`](https://kubernetes.io/docs/reference/labels-annotations-taints/#node-kubernetes-io-unreachable) | `NoExecute` | Pod của DaemonSet có thể được lập lịch lên các node không thể kết nối tới được từ node controller. Mọi Pod DaemonSet đang chạy trên các node như vậy sẽ không bị trục xuất. |
| [`node.kubernetes.io/disk-pressure`](https://kubernetes.io/docs/reference/labels-annotations-taints/#node-kubernetes-io-disk-pressure) | `NoSchedule` | Pod của DaemonSet có thể được lập lịch lên các node đang gặp vấn đề áp lực đĩa (disk pressure). |
| [`node.kubernetes.io/memory-pressure`](https://kubernetes.io/docs/reference/labels-annotations-taints/#node-kubernetes-io-memory-pressure) | `NoSchedule` | Pod của DaemonSet có thể được lập lịch lên các node đang gặp vấn đề áp lực bộ nhớ (memory pressure). |
| [`node.kubernetes.io/pid-pressure`](https://kubernetes.io/docs/reference/labels-annotations-taints/#node-kubernetes-io-pid-pressure) | `NoSchedule` | Pod của DaemonSet có thể được lập lịch lên các node đang gặp vấn đề áp lực tiến trình (process pressure). |
| [`node.kubernetes.io/unschedulable`](https://kubernetes.io/docs/reference/labels-annotations-taints/#node-kubernetes-io-unschedulable) | `NoSchedule` | Pod của DaemonSet có thể được lập lịch lên các node không thể lập lịch (unschedulable). |
| [`node.kubernetes.io/network-unavailable`](https://kubernetes.io/docs/reference/labels-annotations-taints/#node-kubernetes-io-network-unavailable) | `NoSchedule` | **Chỉ được thêm cho các Pod DaemonSet yêu cầu host networking**, tức các Pod có `spec.hostNetwork: true`. Những Pod DaemonSet như vậy có thể được lập lịch lên các node có mạng không khả dụng. |

Bạn cũng có thể thêm các toleration của riêng mình vào các Pod của một DaemonSet, bằng
cách định nghĩa chúng trong Pod template của DaemonSet.

Vì DaemonSet controller tự động đặt toleration
`node.kubernetes.io/unschedulable:NoSchedule`, Kubernetes có thể chạy Pod của DaemonSet
trên các node được đánh dấu là _unschedulable_ (không thể lập lịch).

Nếu bạn dùng một DaemonSet để cung cấp một chức năng quan trọng ở cấp node, chẳng hạn
[mạng của cluster](https://kubernetes.io/docs/concepts/cluster-administration/networking/),
thì việc Kubernetes đặt Pod của DaemonSet lên các node trước khi chúng sẵn sàng (ready)
là hữu ích. Ví dụ, nếu không có toleration đặc biệt đó, bạn có thể rơi vào tình trạng bế
tắc (deadlock): node không được đánh dấu là ready vì network plugin chưa chạy trên đó,
trong khi network plugin lại không chạy trên node đó vì node chưa ready.

## Giao tiếp với các Daemon Pod (Communicating with Daemon Pods)

Một số mẫu (pattern) khả dĩ để giao tiếp với các Pod trong một DaemonSet là:

- **Push**: Các Pod trong DaemonSet được cấu hình để gửi cập nhật tới một service khác,
  chẳng hạn một cơ sở dữ liệu thống kê. Chúng không có client.
- **NodeIP và Known Port**: Các Pod trong DaemonSet có thể dùng `hostPort`, để pod có thể
  truy cập được thông qua IP của node.
  Client biết danh sách IP của các node bằng cách nào đó, và biết port theo quy ước.
- **DNS**: Tạo một [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
  với cùng pod selector, rồi khám phá các DaemonSet thông qua resource `endpoints`
  hoặc truy xuất nhiều bản ghi A từ DNS.
- **Service**: Tạo một service với cùng Pod selector, và dùng service này để truy cập một
  daemon trên một node ngẫu nhiên. Dùng
  [Service Internal Traffic Policy](https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/)
  để giới hạn trong các pod trên cùng node.

## Cập nhật một DaemonSet (Updating a DaemonSet)

Nếu label của node bị thay đổi, DaemonSet sẽ nhanh chóng thêm Pod vào các node mới khớp
và xóa Pod khỏi các node mới không còn khớp.

Bạn có thể sửa đổi các Pod mà một DaemonSet tạo ra. Tuy nhiên, Pod không cho phép cập
nhật tất cả các trường. Ngoài ra, DaemonSet controller sẽ dùng template gốc vào lần tiếp
theo một node (kể cả node trùng tên) được tạo.

Bạn có thể xóa một DaemonSet. Nếu bạn chỉ định `--cascade=orphan` với `kubectl`, thì các
Pod sẽ được để lại trên các node. Nếu sau đó bạn tạo một DaemonSet mới với cùng selector,
DaemonSet mới sẽ nhận nuôi (adopt) các Pod hiện có. Nếu có Pod nào cần thay thế, DaemonSet
sẽ thay thế chúng theo `updateStrategy` của nó.

Bạn có thể [thực hiện rolling update](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/)
trên một DaemonSet.

## Các lựa chọn thay thế cho DaemonSet (Alternatives to DaemonSet)

### Script khởi tạo (Init scripts)

Hoàn toàn có thể chạy các tiến trình daemon bằng cách khởi động chúng trực tiếp trên node
(ví dụ dùng `init`, `upstartd`, hay `systemd`). Cách này hoàn toàn ổn. Tuy nhiên, có một
số lợi ích khi chạy các tiến trình như vậy thông qua một DaemonSet:

- Khả năng giám sát và quản lý log cho các daemon theo cùng cách như với ứng dụng.
- Cùng ngôn ngữ cấu hình và công cụ (ví dụ Pod template, `kubectl`) cho cả daemon lẫn
  ứng dụng.
- Chạy daemon trong container với giới hạn tài nguyên (resource limit) làm tăng mức độ
  cách ly giữa daemon và các container ứng dụng. Tuy nhiên, điều này cũng có thể đạt được
  bằng cách chạy daemon trong một container nhưng không nằm trong một Pod.

### Pod trần (Bare Pods)

Có thể trực tiếp tạo các Pod có chỉ định một node cụ thể để chạy. Tuy nhiên, một DaemonSet
sẽ thay thế các Pod bị xóa hoặc bị chấm dứt vì bất kỳ lý do gì, chẳng hạn khi node gặp sự
cố hoặc khi việc bảo trì node gây gián đoạn, như nâng cấp kernel. Vì lý do này, bạn nên
dùng DaemonSet thay vì tạo các Pod riêng lẻ.

### Static Pod (Static Pods)

Có thể tạo Pod bằng cách ghi một file vào một thư mục nhất định được Kubelet theo dõi.
Chúng được gọi là [static pod](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/).
Khác với DaemonSet, static Pod không thể được quản lý bằng kubectl hay các client khác
của Kubernetes API. Static Pod không phụ thuộc vào apiserver, điều này khiến chúng hữu
ích trong các trường hợp khởi tạo (bootstrap) cluster. Ngoài ra, static Pod có thể bị
loại bỏ (deprecated) trong tương lai.

### Deployment (Deployments)

DaemonSet tương tự [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
ở chỗ cả hai đều tạo Pod, và các Pod đó có các tiến trình không được kỳ vọng sẽ tự kết
thúc (ví dụ web server, storage server).

Dùng Deployment cho các service stateless, như các frontend, nơi việc tăng giảm số lượng
replica và tung ra (roll out) các bản cập nhật quan trọng hơn việc kiểm soát chính xác
Pod chạy trên host nào. Dùng DaemonSet khi điều quan trọng là một bản sao của Pod phải
luôn chạy trên tất cả hoặc một số host nhất định, nếu DaemonSet cung cấp chức năng cấp
node cho phép các Pod khác chạy đúng trên node cụ thể đó.

Ví dụ, [các network plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
thường bao gồm một thành phần chạy dưới dạng DaemonSet. Thành phần DaemonSet này đảm bảo
node nơi nó đang chạy có mạng cluster hoạt động bình thường.

## Tiếp theo (What's next)

* Tìm hiểu về [Pod](./46-pods-vi.md):
  * Tìm hiểu về [static Pod](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/),
    hữu ích để chạy các thành phần control plane của Kubernetes.
* Tìm hiểu cách sử dụng DaemonSet:
  * [Thực hiện rolling update trên một DaemonSet](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/).
  * [Thực hiện rollback trên một DaemonSet](https://kubernetes.io/docs/tasks/manage-daemon/rollback-daemon-set/)
    (ví dụ khi một lần roll out không diễn ra như bạn mong đợi).
* Hiểu [cách Kubernetes gán Pod cho các Node](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/).
* Tìm hiểu về [device plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
  và [các add-on](https://kubernetes.io/docs/concepts/cluster-administration/addons/),
  vốn thường chạy dưới dạng DaemonSet.
* `DaemonSet` là một resource cấp cao nhất (top-level) trong Kubernetes REST API.
  Đọc định nghĩa object
  [DaemonSet](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/daemon-set-v1/)
  để hiểu API dành cho daemon set.
