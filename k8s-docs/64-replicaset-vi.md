# ReplicaSet

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/>
>
> Mục đích của ReplicaSet là duy trì một tập hợp ổn định các Pod bản sao (replica) luôn chạy
> tại bất kỳ thời điểm nào. Thông thường, bạn định nghĩa một Deployment và để Deployment đó
> tự động quản lý các ReplicaSet.

Mục đích của một ReplicaSet là duy trì một tập hợp ổn định các Pod bản sao (replica Pod)
luôn chạy tại bất kỳ thời điểm nào. Vì vậy, nó thường được dùng để đảm bảo tính sẵn sàng
(availability) của một số lượng Pod giống hệt nhau theo chỉ định.

## Cách một ReplicaSet hoạt động (How a ReplicaSet works) {#how-a-replicaset-works}

Một ReplicaSet được định nghĩa bằng các trường, bao gồm một selector chỉ định cách nhận diện
các Pod mà nó có thể thu nhận (acquire), một số lượng replica cho biết nó cần duy trì bao
nhiêu Pod, và một pod template chỉ định dữ liệu của các Pod mới mà nó cần tạo để đáp ứng
tiêu chí về số lượng replica. Sau đó, ReplicaSet thực hiện mục đích của mình bằng cách tạo
và xóa các Pod khi cần để đạt được số lượng mong muốn. Khi một ReplicaSet cần tạo các Pod
mới, nó sử dụng Pod template của mình.

Một ReplicaSet được liên kết với các Pod của nó thông qua trường
[metadata.ownerReferences](./36-garbage-collection-vi.md#owners-dependents)
của Pod, trường này chỉ định resource nào đang sở hữu object hiện tại. Tất cả các Pod được
một ReplicaSet thu nhận đều mang thông tin định danh của ReplicaSet sở hữu chúng trong
trường ownerReferences. Chính nhờ liên kết này mà ReplicaSet biết được trạng thái của các
Pod nó đang duy trì và lập kế hoạch tương ứng.

Một ReplicaSet nhận diện các Pod mới cần thu nhận bằng selector của nó. Nếu có một Pod
không có OwnerReference, hoặc OwnerReference không phải là một controller, và Pod đó khớp
với selector của một ReplicaSet, thì nó sẽ ngay lập tức bị ReplicaSet đó thu nhận.

## Khi nào nên dùng ReplicaSet (When to use a ReplicaSet)

Một ReplicaSet đảm bảo rằng một số lượng pod replica được chỉ định luôn chạy tại bất kỳ
thời điểm nào. Tuy nhiên, Deployment là một khái niệm cấp cao hơn, quản lý các ReplicaSet
và cung cấp cơ chế cập nhật khai báo (declarative update) cho các Pod cùng nhiều tính năng
hữu ích khác. Do đó, chúng tôi khuyến nghị dùng Deployment thay vì trực tiếp dùng
ReplicaSet, trừ khi bạn cần cơ chế điều phối cập nhật tùy biến hoặc hoàn toàn không cần
cập nhật.

Điều này thực ra có nghĩa là có thể bạn sẽ không bao giờ cần thao tác trực tiếp với các
object ReplicaSet: hãy dùng Deployment và định nghĩa ứng dụng của bạn trong phần spec.

## Ví dụ (Example)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # thay đổi số replica tùy theo trường hợp của bạn
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5
```

Lưu manifest này vào file `frontend.yaml` và gửi (submit) nó lên một cluster Kubernetes
sẽ tạo ra ReplicaSet đã định nghĩa cùng các Pod mà nó quản lý.

```shell
kubectl apply -f https://kubernetes.io/examples/controllers/frontend.yaml
```

Sau đó bạn có thể xem các ReplicaSet hiện đang được triển khai:

```shell
kubectl get rs
```

Và thấy ReplicaSet frontend mà bạn đã tạo:

```
NAME       DESIRED   CURRENT   READY   AGE
frontend   3         3         3       6s
```

Bạn cũng có thể kiểm tra trạng thái của ReplicaSet:

```shell
kubectl describe rs/frontend
```

Và bạn sẽ thấy output tương tự như:

```
Name:         frontend
Namespace:    default
Selector:     tier=frontend
Labels:       app=guestbook
              tier=frontend
Annotations:  <none>
Replicas:     3 current / 3 desired
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  tier=frontend
  Containers:
   php-redis:
    Image:        us-docker.pkg.dev/google-samples/containers/gke/gb-frontend:v5
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  13s   replicaset-controller  Created pod: frontend-gbgfx
  Normal  SuccessfulCreate  13s   replicaset-controller  Created pod: frontend-rwz57
  Normal  SuccessfulCreate  13s   replicaset-controller  Created pod: frontend-wkl7w
```

Và cuối cùng bạn có thể kiểm tra các Pod đã được khởi chạy:

```shell
kubectl get pods
```

Bạn sẽ thấy thông tin Pod tương tự như:

```
NAME             READY   STATUS    RESTARTS   AGE
frontend-gbgfx   1/1     Running   0          10m
frontend-rwz57   1/1     Running   0          10m
frontend-wkl7w   1/1     Running   0          10m
```

Bạn cũng có thể xác nhận rằng owner reference của các pod này được đặt trỏ tới ReplicaSet
frontend. Để làm điều đó, lấy yaml của một trong các Pod đang chạy:

```shell
kubectl get pods frontend-gbgfx -o yaml
```

Output sẽ trông tương tự như dưới đây, với thông tin của ReplicaSet frontend được đặt
trong trường ownerReferences của metadata:

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2024-02-28T22:30:44Z"
  generateName: frontend-
  labels:
    tier: frontend
  name: frontend-gbgfx
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: frontend
    uid: e129deca-f864-481b-bb16-b27abfd92292
...
```

## Thu nhận các Pod không theo template (Non-Template Pod acquisitions)

Mặc dù bạn hoàn toàn có thể tạo các Pod trần (bare Pod) mà không gặp vấn đề gì, chúng tôi
đặc biệt khuyến nghị bạn đảm bảo rằng các Pod trần đó không mang label khớp với selector
của một trong các ReplicaSet của bạn. Lý do là vì một ReplicaSet không chỉ giới hạn ở việc
sở hữu các Pod được chỉ định bởi template của nó — nó có thể thu nhận các Pod khác theo
cách đã mô tả ở các phần trước.

Lấy ví dụ ReplicaSet frontend ở trên, và các Pod được chỉ định trong manifest sau:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  labels:
    tier: frontend
spec:
  containers:
  - name: hello1
    image: gcr.io/google-samples/hello-app:2.0

---

apiVersion: v1
kind: Pod
metadata:
  name: pod2
  labels:
    tier: frontend
spec:
  containers:
  - name: hello2
    image: gcr.io/google-samples/hello-app:1.0
```

Vì các Pod đó không có một Controller (hay bất kỳ object nào) làm owner reference và khớp
với selector của ReplicaSet frontend, chúng sẽ ngay lập tức bị ReplicaSet này thu nhận.

Giả sử bạn tạo các Pod này sau khi ReplicaSet frontend đã được triển khai và đã thiết lập
xong các Pod replica ban đầu để đáp ứng yêu cầu về số lượng replica của nó:

```shell
kubectl apply -f https://kubernetes.io/examples/pods/pod-rs.yaml
```

Các Pod mới sẽ bị ReplicaSet thu nhận, rồi ngay lập tức bị chấm dứt (terminate) vì khi đó
ReplicaSet vượt quá số lượng mong muốn.

Liệt kê các Pod:

```shell
kubectl get pods
```

Output cho thấy các Pod mới hoặc đã bị chấm dứt, hoặc đang trong quá trình bị chấm dứt:

```
NAME             READY   STATUS        RESTARTS   AGE
frontend-b2zdv   1/1     Running       0          10m
frontend-vcmts   1/1     Running       0          10m
frontend-wtsmm   1/1     Running       0          10m
pod1             0/1     Terminating   0          1s
pod2             0/1     Terminating   0          1s
```

Nếu bạn tạo các Pod trước:

```shell
kubectl apply -f https://kubernetes.io/examples/pods/pod-rs.yaml
```

Rồi sau đó mới tạo ReplicaSet:

```shell
kubectl apply -f https://kubernetes.io/examples/controllers/frontend.yaml
```

Bạn sẽ thấy rằng ReplicaSet đã thu nhận các Pod này và chỉ tạo thêm các Pod mới theo spec
của nó cho đến khi tổng số Pod mới và Pod ban đầu khớp với số lượng mong muốn. Khi liệt kê
các Pod:

```shell
kubectl get pods
```

Output sẽ cho thấy:
```
NAME             READY   STATUS    RESTARTS   AGE
frontend-hmmj2   1/1     Running   0          9s
pod1             1/1     Running   0          36s
pod2             1/1     Running   0          36s
```

Theo cách này, một ReplicaSet có thể sở hữu một tập các Pod không đồng nhất.

## Viết một manifest ReplicaSet (Writing a ReplicaSet manifest)

Giống như mọi object API Kubernetes khác, một ReplicaSet cần các trường `apiVersion`,
`kind`, và `metadata`. Với ReplicaSet, `kind` luôn là ReplicaSet.

Khi control plane tạo các Pod mới cho một ReplicaSet, `.metadata.name` của ReplicaSet là
một phần cơ sở để đặt tên cho các Pod đó. Tên của một ReplicaSet phải là một giá trị
[DNS subdomain](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)
hợp lệ, nhưng điều này có thể tạo ra kết quả không mong đợi đối với hostname của Pod. Để
có độ tương thích tốt nhất, tên nên tuân theo các quy tắc chặt chẽ hơn của một
[DNS label](./17-names-vi.md#dns-label-names).

Một ReplicaSet cũng cần một
[phần `.spec`](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status).

### Pod Template

`.spec.template` là một [pod template](https://kubernetes.io/docs/concepts/workloads/pods/#pod-templates),
và cũng bắt buộc phải có sẵn các label. Trong ví dụ `frontend.yaml` của chúng ta, có một
label: `tier: frontend`. Hãy cẩn thận đừng để trùng lặp với selector của các controller
khác, kẻo chúng cố gắng nhận nuôi (adopt) Pod này.

Đối với trường [chính sách khởi động lại](./47-pod-lifecycle-vi.md#restart-policy)
(restart policy) của template, `.spec.template.spec.restartPolicy`, giá trị duy nhất được
phép là `Always`, và đó cũng là giá trị mặc định.

### Bộ chọn Pod (Pod Selector)

Trường `.spec.selector` là một [label selector](./18-labels-vi.md). Như đã thảo luận
[ở trên](#how-a-replicaset-works), đây là các label được dùng để nhận diện các Pod tiềm
năng cần thu nhận. Trong ví dụ `frontend.yaml`, selector là:

```yaml
matchLabels:
  tier: frontend
```

Trong ReplicaSet, `.spec.template.metadata.labels` phải khớp với `spec.selector`, nếu
không nó sẽ bị API từ chối.

> **Ghi chú:**
> Với 2 ReplicaSet cùng chỉ định một `.spec.selector` nhưng khác nhau ở các trường
> `.spec.template.metadata.labels` và `.spec.template.spec`, mỗi ReplicaSet sẽ bỏ qua các
> Pod do ReplicaSet kia tạo ra.

### Replicas

Bạn có thể chỉ định số lượng Pod cần chạy đồng thời bằng cách đặt `.spec.replicas`.
ReplicaSet sẽ tạo/xóa các Pod của nó để khớp với con số này.

Nếu bạn không chỉ định `.spec.replicas`, giá trị mặc định là 1.

## Làm việc với ReplicaSet (Working with ReplicaSets)

### Xóa một ReplicaSet và các Pod của nó (Deleting a ReplicaSet and its Pods)

Để xóa một ReplicaSet cùng tất cả các Pod của nó, dùng
[`kubectl delete`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete).
[Trình thu gom rác (Garbage collector)](./36-garbage-collection-vi.md) sẽ tự động xóa tất
cả các Pod phụ thuộc theo mặc định.

Khi dùng REST API hoặc thư viện `client-go`, bạn phải đặt `propagationPolicy` là
`Background` hoặc `Foreground` trong tùy chọn `-d`. Ví dụ:

```shell
kubectl proxy --port=8080
curl -X DELETE  'localhost:8080/apis/apps/v1/namespaces/default/replicasets/frontend' \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
  -H "Content-Type: application/json"
```

### Chỉ xóa ReplicaSet (Deleting just a ReplicaSet)

Bạn có thể xóa một ReplicaSet mà không ảnh hưởng đến bất kỳ Pod nào của nó bằng
[`kubectl delete`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#delete)
với tùy chọn `--cascade=orphan`.
Khi dùng REST API hoặc thư viện `client-go`, bạn phải đặt `propagationPolicy` là `Orphan`.
Ví dụ:

```shell
kubectl proxy --port=8080
curl -X DELETE  'localhost:8080/apis/apps/v1/namespaces/default/replicasets/frontend' \
  -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
  -H "Content-Type: application/json"
```

Sau khi bản gốc đã bị xóa, bạn có thể tạo một ReplicaSet mới để thay thế. Miễn là
`.spec.selector` cũ và mới giống nhau, ReplicaSet mới sẽ nhận nuôi các Pod cũ. Tuy nhiên,
nó sẽ không nỗ lực làm cho các Pod hiện có khớp với một pod template mới, khác trước. Để
cập nhật các Pod sang spec mới theo cách có kiểm soát, hãy dùng
[Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#creating-a-deployment),
vì ReplicaSet không trực tiếp hỗ trợ rolling update.

### Các Pod đang kết thúc (Terminating Pods)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`

Bạn có thể bật tính năng này bằng cách đặt
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`DeploymentReplicaSetTerminatingReplicas`
trên [API server](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/)
và trên [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)

Các Pod chuyển sang trạng thái đang kết thúc (terminating) do bị xóa hoặc do thu hẹp quy
mô (scale down) có thể mất nhiều thời gian để kết thúc, và có thể tiêu tốn thêm tài nguyên
trong khoảng thời gian đó. Hệ quả là tổng số pod có thể tạm thời vượt quá `.spec.replicas`.
Các pod đang kết thúc có thể được theo dõi qua trường `.status.terminatingReplicas` của
ReplicaSet.

### Cô lập Pod khỏi một ReplicaSet (Isolating Pods from a ReplicaSet)

Bạn có thể tách các Pod ra khỏi một ReplicaSet bằng cách thay đổi label của chúng. Kỹ
thuật này có thể được dùng để đưa Pod ra khỏi service nhằm phục vụ gỡ lỗi (debug), khôi
phục dữ liệu, v.v. Các Pod bị tách ra theo cách này sẽ được thay thế tự động (giả định
rằng số lượng replica không bị thay đổi).

### Thay đổi quy mô một ReplicaSet (Scaling a ReplicaSet)

Một ReplicaSet có thể dễ dàng được mở rộng (scale up) hoặc thu hẹp (scale down) chỉ bằng
cách cập nhật trường `.spec.replicas`. ReplicaSet controller đảm bảo rằng số lượng Pod
mong muốn có label selector khớp luôn khả dụng và hoạt động.

Khi thu hẹp quy mô, ReplicaSet controller chọn các pod sẽ bị xóa bằng cách sắp xếp các pod
hiện có để ưu tiên thu hẹp các pod dựa trên thuật toán tổng quát sau:

1. Các pod Pending (và không thể lập lịch — unschedulable) bị thu hẹp trước
2. Nếu annotation `controller.kubernetes.io/pod-deletion-cost` được đặt, thì pod có giá
   trị thấp hơn sẽ được chọn trước.
3. Các pod trên node có nhiều replica hơn được chọn trước các pod trên node có ít replica
   hơn.
4. Nếu thời điểm tạo của các pod khác nhau, pod được tạo gần đây hơn được chọn trước pod
   cũ hơn (thời điểm tạo được phân nhóm theo thang log nguyên).

Nếu tất cả các tiêu chí trên đều như nhau, việc lựa chọn là ngẫu nhiên.

### Chi phí xóa Pod (Pod deletion cost)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.22 [beta]`

Bằng annotation
[`controller.kubernetes.io/pod-deletion-cost`](https://kubernetes.io/docs/reference/labels-annotations-taints/#pod-deletion-cost),
người dùng có thể đặt mức ưu tiên về việc pod nào sẽ bị xóa trước khi thu hẹp quy mô một
ReplicaSet.

Annotation này cần được đặt trên pod, phạm vi giá trị là [-2147483648, 2147483647]. Nó
biểu thị chi phí của việc xóa một pod so với các pod khác thuộc cùng ReplicaSet. Các pod
có chi phí xóa thấp hơn được ưu tiên xóa trước các pod có chi phí xóa cao hơn.

Giá trị ngầm định của annotation này đối với các pod không đặt nó là 0; giá trị âm được
phép. Các giá trị không hợp lệ sẽ bị API server từ chối.

Tính năng này ở trạng thái beta và được bật mặc định. Bạn có thể tắt nó bằng
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`PodDeletionCost` trên cả kube-apiserver và kube-controller-manager.

> **Ghi chú:**
> - Cơ chế này chỉ được tôn trọng ở mức nỗ lực tốt nhất (best-effort), nên nó không đưa ra
>   bất kỳ đảm bảo nào về thứ tự xóa pod.
> - Người dùng nên tránh cập nhật annotation này thường xuyên, chẳng hạn cập nhật dựa theo
>   giá trị của một metric, vì làm vậy sẽ sinh ra một lượng đáng kể các cập nhật pod trên
>   apiserver.

#### Ví dụ tình huống sử dụng (Example Use Case)

Các pod khác nhau của một ứng dụng có thể có mức độ sử dụng (utilization) khác nhau. Khi
thu hẹp quy mô, ứng dụng có thể muốn xóa các pod có mức sử dụng thấp hơn trước. Để tránh
cập nhật pod thường xuyên, ứng dụng nên cập nhật `controller.kubernetes.io/pod-deletion-cost`
một lần trước khi thực hiện thu hẹp quy mô (đặt annotation thành giá trị tỉ lệ với mức sử
dụng của pod). Cách này hiệu quả nếu bản thân ứng dụng kiểm soát việc thu hẹp quy mô; ví
dụ, driver pod của một triển khai Spark.

### ReplicaSet làm mục tiêu cho Horizontal Pod Autoscaler (ReplicaSet as a Horizontal Pod Autoscaler Target)

Một ReplicaSet cũng có thể là mục tiêu của
[Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/).
Nghĩa là một ReplicaSet có thể được tự động thay đổi quy mô (auto-scale) bởi một HPA. Dưới
đây là một ví dụ HPA nhắm tới ReplicaSet mà chúng ta đã tạo ở ví dụ trước.

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: ReplicaSet
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

Lưu manifest này vào file `hpa-rs.yaml` và gửi nó lên một cluster Kubernetes sẽ tạo ra HPA
đã định nghĩa, tự động thay đổi quy mô ReplicaSet mục tiêu dựa trên mức sử dụng CPU của
các Pod được nhân bản.

```shell
kubectl apply -f https://k8s.io/examples/controllers/hpa-rs.yaml
```

Hoặc bạn có thể dùng lệnh `kubectl autoscale` để đạt cùng kết quả (và cách này dễ hơn!)

```shell
kubectl autoscale rs frontend --max=10 --min=3 --cpu=50%
```

## Các lựa chọn thay thế cho ReplicaSet (Alternatives to ReplicaSet)

### Deployment (khuyến nghị) (Deployment (recommended))

[`Deployment`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) là
một object có thể sở hữu các ReplicaSet, cập nhật chúng và các Pod của chúng thông qua
rolling update kiểu khai báo, phía server. Mặc dù ReplicaSet có thể được dùng độc lập,
ngày nay chúng chủ yếu được Deployment dùng như một cơ chế để điều phối việc tạo, xóa và
cập nhật Pod. Khi dùng Deployment, bạn không phải lo lắng về việc quản lý các ReplicaSet
mà chúng tạo ra. Deployment sở hữu và quản lý các ReplicaSet của nó. Vì vậy, khuyến nghị
là dùng Deployment khi bạn muốn có ReplicaSet.

### Pod trần (Bare Pods)

Khác với trường hợp người dùng trực tiếp tạo Pod, một ReplicaSet sẽ thay thế các Pod bị
xóa hoặc bị chấm dứt vì bất kỳ lý do gì, chẳng hạn khi node gặp sự cố hoặc khi việc bảo
trì node gây gián đoạn, như nâng cấp kernel. Vì lý do này, chúng tôi khuyến nghị bạn dùng
ReplicaSet ngay cả khi ứng dụng của bạn chỉ cần một Pod duy nhất. Hãy hình dung nó tương
tự một trình giám sát tiến trình (process supervisor), chỉ khác là nó giám sát nhiều Pod
trên nhiều node thay vì các tiến trình riêng lẻ trên một node duy nhất. Một ReplicaSet ủy
quyền việc khởi động lại container cục bộ cho một agent nào đó trên node, chẳng hạn
Kubelet.

### Job

Dùng [`Job`](https://kubernetes.io/docs/concepts/workloads/controllers/job/) thay cho
ReplicaSet đối với các Pod được kỳ vọng sẽ tự kết thúc (nghĩa là các batch job).

### DaemonSet

Dùng [`DaemonSet`](./66-daemonset-vi.md) thay cho ReplicaSet đối với các Pod cung cấp
chức năng ở cấp máy (machine-level), chẳng hạn giám sát máy hoặc ghi log của máy. Các Pod
này có vòng đời gắn với vòng đời của máy: Pod cần chạy trên máy trước khi các Pod khác
khởi động, và an toàn để chấm dứt khi máy đã sẵn sàng để được khởi động lại/tắt.

### ReplicationController

ReplicaSet là thế hệ kế nhiệm của
[ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/).
Cả hai phục vụ cùng một mục đích và hành xử tương tự nhau, ngoại trừ việc
ReplicationController không hỗ trợ các yêu cầu selector dựa trên tập hợp (set-based) như
mô tả trong [hướng dẫn sử dụng label](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors).
Vì vậy, ReplicaSet được ưa chuộng hơn ReplicationController.

## Tiếp theo (What's next)

* Tìm hiểu về [Pod](./46-pods-vi.md).
* Tìm hiểu về [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).
* [Chạy một ứng dụng stateless bằng Deployment](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/),
  vốn dựa trên ReplicaSet để hoạt động.
* `ReplicaSet` là một resource cấp cao nhất (top-level) trong Kubernetes REST API.
  Đọc định nghĩa object
  [ReplicaSet](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/replica-set-v1/)
  để hiểu API dành cho replica set.
* Đọc về [PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)
  và cách bạn có thể dùng nó để quản lý tính sẵn sàng của ứng dụng trong các giai đoạn
  gián đoạn (disruption).
