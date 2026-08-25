# Cập nhật đối tượng API tại chỗ bằng kubectl patch (Update API Objects in Place Using kubectl patch)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/>
>
> Sử dụng kubectl patch để cập nhật các đối tượng API của Kubernetes tại chỗ (in place). Thực hiện một strategic merge patch hoặc một JSON merge patch.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 4 → nhóm [4a](00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout), bài 3/7 ·
Kiểm chứng ở [Lab 4a](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) phần B3.3 và B9.3.

Đọc bài này ngay sau bài [319](319-declarative-config-vi.md). Hai bài nói về hai đường **khác nhau**
để sửa một object đang chạy: 319 sửa qua **file cấu hình** rồi `apply`, còn bài này sửa **tại chỗ**
bằng `patch` mà không đụng tới file cấu hình — mục *Tóm tắt* nói thẳng điều đó ở câu cuối.

Bài dùng `tolerations` làm ví dụ cho một list bị thay thế. Taint và toleration thuộc
[giai đoạn 7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction); ở đây bạn chỉ cần thấy nó là **một
list khác** có hành vi merge khác `containers`, chưa cần biết nó dùng để làm gì.

**Phải hiểu ở lần đọc này:**

- Mục *Sử dụng strategic merge patch để cập nhật một Deployment*: patch mặc định **thêm** container
  `patch-demo-ctr-2` vào list `containers` chứ không thay thế list. Bằng chứng là tên Pod đổi và cột
  `READY` chuyển từ `1/1` sang `2/2`.
- Mục *Ghi chú về strategic merge patch*: cũng là strategic merge patch, nhưng list `tolerations`
  **bị thay thế**. Lý do nằm ở field chứ không ở loại patch — `Containers` mang tag
  `patchStrategy:"merge"` với `patchMergeKey:"name"`, còn `Tolerations` không có tag `patchStrategy`
  nên rơi về chiến lược mặc định là `replace`.
- Bảng ba giá trị của tham số `type` — `json`, `merge`, `strategic` — và **mặc định là `strategic`**.
  Với JSON merge patch (`--type merge`), muốn cập nhật một list thì phải ghi **cả list mới**, và list
  mới **thay thế hoàn toàn** list cũ: ví dụ của bài làm Pod tụt về `1/1` với đúng một container.
- Mục *retainKeys*: đặt `spec.strategy.type: Recreate` bị từ chối khi `spec.strategy.rollingUpdate`
  còn giá trị. Chỉ thị `$retainKeys` liệt kê các khóa cần giữ; **mọi khóa vắng mặt trong danh sách
  đó sẽ bị xóa** khi patch. Nó chỉ dùng được với field có `patchStrategy:"retainKeys"`, như
  `Strategy` của `DeploymentSpec`.
- Mục *Cập nhật số lượng replica của một đối tượng bằng `kubectl patch` với `--subresource`*:
  `--subresource=scale` patch thẳng vào subresource `scale`, và output báo
  `scale.autoscaling/nginx-deployment patched` chứ không phải tên Deployment. Ghi chú ngay sau đó:
  chỉ định `--subresource` cho resource không có subresource ấy thì API server trả **404 Not Found**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Các khối `types.go`, OpenApi spec và link API reference dùng để tra `patchStrategy` / `patchMergeKey` | là chỗ tra cứu khi cần biết một field cụ thể sẽ merge hay replace, không phải đoạn để đọc hiểu | tra khi viết patch ở [Lab 4a](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) phần B9.3 |
| Giá trị `type: json` (JSON Patch, RFC 6902) — chỉ xuất hiện trong bảng, bài không có ví dụ nào | bài này không thực hành nó; nó là loại patch thứ ba, khác cả `strategic` lẫn `merge` | [Lab 4a](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) phần B3.3, nơi JSON patch dùng phép `test` để tiền kiểm trước khi đổi `replicas` |
| Ghi chú cuối mục *Tóm tắt*: strategic merge patch không được hỗ trợ cho custom resource | custom resource và CRD chưa học | [giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) |
| Ví dụ `tolerations` với `key: dedicated` và `key: disktype` xét như một cơ chế lập lịch | ở đây nó chỉ đóng vai một list có hành vi merge khác `containers` | bài [139](139-taint-and-toleration-vi.md) ở [giai đoạn 7a](00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction) |

---

Trang này hướng dẫn cách dùng `kubectl patch` để cập nhật một đối tượng API tại chỗ. Các bài
thực hành trong trang này minh họa một strategic merge patch và một JSON merge patch.

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

## Sử dụng strategic merge patch để cập nhật một Deployment (Use a strategic merge patch to update a Deployment)

Đây là file cấu hình cho một Deployment có hai replica. Mỗi replica là một Pod có một
container:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: patch-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: patch-demo-ctr
        image: nginx
      tolerations:
      - effect: NoSchedule
        key: dedicated
        value: test-team
```

Tạo Deployment:

```shell
kubectl apply -f https://k8s.io/examples/application/deployment-patch.yaml
```

Xem các Pod gắn với Deployment của bạn:

```shell
kubectl get pods
```

Output cho thấy Deployment có hai Pod. Giá trị `1/1` cho biết mỗi Pod có một container:

```
NAME                        READY     STATUS    RESTARTS   AGE
patch-demo-28633765-670qr   1/1       Running   0          23s
patch-demo-28633765-j5qs3   1/1       Running   0          23s
```

Hãy ghi lại tên của các Pod đang chạy. Lát nữa, bạn sẽ thấy các Pod này bị chấm dứt
(terminate) và được thay thế bằng các Pod mới.

Tại thời điểm này, mỗi Pod có một Container chạy image nginx. Bây giờ giả sử bạn muốn mỗi Pod
có hai container: một container chạy nginx và một container chạy redis.

Tạo một file tên là `patch-file.yaml` với nội dung sau:

```yaml
spec:
  template:
    spec:
      containers:
      - name: patch-demo-ctr-2
        image: redis
```

Patch Deployment của bạn:

```shell
kubectl patch deployment patch-demo --patch-file patch-file.yaml
```

Xem Deployment đã được patch:

```shell
kubectl get deployment patch-demo --output yaml
```

Output cho thấy PodSpec trong Deployment có hai Container:

```yaml
containers:
- image: redis
  imagePullPolicy: Always
  name: patch-demo-ctr-2
  ...
- image: nginx
  imagePullPolicy: Always
  name: patch-demo-ctr
  ...
```

Xem các Pod gắn với Deployment đã được patch:

```shell
kubectl get pods
```

Output cho thấy các Pod đang chạy có tên khác với các Pod chạy trước đó. Deployment đã chấm
dứt các Pod cũ và tạo hai Pod mới tuân theo spec đã cập nhật của Deployment. Giá trị `2/2`
cho biết mỗi Pod có hai Container:

```
NAME                          READY     STATUS    RESTARTS   AGE
patch-demo-1081991389-2wrn5   2/2       Running   0          1m
patch-demo-1081991389-jmg7b   2/2       Running   0          1m
```

Xem kỹ hơn một trong các Pod patch-demo:

```shell
kubectl get pod <your-pod-name> --output yaml
```

Output cho thấy Pod có hai Container: một container chạy nginx và một container chạy redis:

```
containers:
- image: redis
  ...
- image: nginx
  ...
```

### Ghi chú về strategic merge patch (Notes on the strategic merge patch)

Thao tác patch bạn thực hiện trong bài thực hành trên được gọi là *strategic merge patch*
(vá trộn theo chiến lược). Lưu ý rằng bản patch đã không thay thế danh sách `containers`.
Thay vào đó, nó thêm một Container mới vào danh sách. Nói cách khác, danh sách trong bản patch
được trộn (merge) với danh sách hiện có. Điều này không phải lúc nào cũng xảy ra khi bạn dùng
strategic merge patch trên một danh sách. Trong một số trường hợp, danh sách bị thay thế chứ
không được trộn.

Với strategic merge patch, một danh sách hoặc bị thay thế hoặc được trộn tùy theo chiến lược
patch (patch strategy) của nó. Chiến lược patch được chỉ định bởi giá trị của khóa
`patchStrategy` trong một field tag ở mã nguồn Kubernetes. Ví dụ, trường `Containers` của
struct `PodSpec` có `patchStrategy` là `merge`:

```go
type PodSpec struct {
  ...
  Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" ...`
  ...
}
```

Bạn cũng có thể thấy chiến lược patch trong
[OpenApi spec](https://raw.githubusercontent.com/kubernetes/kubernetes/master/api/openapi-spec/swagger.json):

```yaml
"io.k8s.api.core.v1.PodSpec": {
    ...,
    "containers": {
        "description": "List of containers belonging to the pod.  ...."
    },
    "x-kubernetes-patch-merge-key": "name",
    "x-kubernetes-patch-strategy": "merge"
}
```

Và bạn có thể thấy chiến lược patch trong
[tài liệu Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podspec-v1-core).

Tạo một file tên là `patch-file-tolerations.yaml` với nội dung sau:

```yaml
spec:
  template:
    spec:
      tolerations:
      - effect: NoSchedule
        key: disktype
        value: ssd
```

Patch Deployment của bạn:

```shell
kubectl patch deployment patch-demo --patch-file patch-file-tolerations.yaml
```

Xem Deployment đã được patch:

```shell
kubectl get deployment patch-demo --output yaml
```

Output cho thấy PodSpec trong Deployment chỉ có một Toleration:

```yaml
tolerations:
- effect: NoSchedule
  key: disktype
  value: ssd
```

Lưu ý rằng danh sách `tolerations` trong PodSpec đã bị thay thế, không được trộn. Đó là vì
trường Tolerations của PodSpec không có khóa `patchStrategy` trong field tag của nó. Vì vậy
strategic merge patch dùng chiến lược patch mặc định, tức là `replace` (thay thế).

```go
type PodSpec struct {
  ...
  Tolerations []Toleration `json:"tolerations,omitempty" protobuf:"bytes,22,opt,name=tolerations"`
  ...
}
```

## Sử dụng JSON merge patch để cập nhật một Deployment (Use a JSON merge patch to update a Deployment)

Strategic merge patch khác với
[JSON merge patch](https://tools.ietf.org/html/rfc7386).
Với JSON merge patch, nếu bạn muốn cập nhật một danh sách, bạn phải chỉ định toàn bộ danh
sách mới. Và danh sách mới thay thế hoàn toàn danh sách hiện có.

Lệnh `kubectl patch` có tham số `type` mà bạn có thể đặt thành một trong các giá trị sau:

| Giá trị tham số | Kiểu merge |
|---|---|
| json | [JSON Patch, RFC 6902](https://tools.ietf.org/html/rfc6902) |
| merge | [JSON Merge Patch, RFC 7386](https://tools.ietf.org/html/rfc7386) |
| strategic | Strategic merge patch |

Để so sánh JSON patch và JSON merge patch, xem
[JSON Patch and JSON Merge Patch](https://erosb.github.io/post/json-patch-vs-merge-patch/).

Giá trị mặc định của tham số `type` là `strategic`. Vì vậy trong bài thực hành trước, bạn đã
thực hiện một strategic merge patch.

Tiếp theo, hãy thực hiện một JSON merge patch trên cùng Deployment đó. Tạo một file tên là
`patch-file-2.yaml` với nội dung sau:

```yaml
spec:
  template:
    spec:
      containers:
      - name: patch-demo-ctr-3
        image: gcr.io/google-samples/hello-app:2.0
```

Trong lệnh patch, đặt `type` thành `merge`:

```shell
kubectl patch deployment patch-demo --type merge --patch-file patch-file-2.yaml
```

Xem Deployment đã được patch:

```shell
kubectl get deployment patch-demo --output yaml
```

Danh sách `containers` mà bạn chỉ định trong bản patch chỉ có một Container.
Output cho thấy danh sách gồm một Container của bạn đã thay thế danh sách `containers` hiện
có.

```yaml
spec:
  containers:
  - image: gcr.io/google-samples/hello-app:2.0
    ...
    name: patch-demo-ctr-3
```

Liệt kê các Pod đang chạy:

```shell
kubectl get pods
```

Trong output, bạn có thể thấy các Pod hiện có đã bị chấm dứt và các Pod mới được tạo. Giá trị
`1/1` cho biết mỗi Pod mới chỉ chạy một Container.

```shell
NAME                          READY     STATUS    RESTARTS   AGE
patch-demo-1307768864-69308   1/1       Running   0          1m
patch-demo-1307768864-c86dc   1/1       Running   0          1m
```

## Sử dụng strategic merge patch để cập nhật một Deployment với chiến lược retainKeys (Use strategic merge patch to update a Deployment using the retainKeys strategy)

Đây là file cấu hình cho một Deployment dùng chiến lược `RollingUpdate`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: retainkeys-demo
spec:
  selector:
    matchLabels:
      app: nginx
  strategy:
    rollingUpdate:
      maxSurge: 30%
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: retainkeys-demo-ctr
        image: nginx
```

Tạo deployment:

```shell
kubectl apply -f https://k8s.io/examples/application/deployment-retainkeys.yaml
```

Tại thời điểm này, deployment đã được tạo và đang dùng chiến lược `RollingUpdate`.

Tạo một file tên là `patch-file-no-retainkeys.yaml` với nội dung sau:

```yaml
spec:
  strategy:
    type: Recreate
```

Patch Deployment của bạn:

```shell
kubectl patch deployment retainkeys-demo --type strategic --patch-file patch-file-no-retainkeys.yaml
```

Trong output, bạn có thể thấy rằng không thể đặt `type` thành `Recreate` khi
`spec.strategy.rollingUpdate` đã có giá trị được định nghĩa:

```
The Deployment "retainkeys-demo" is invalid: spec.strategy.rollingUpdate: Forbidden: may not be specified when strategy `type` is 'Recreate'
```

Cách để xóa giá trị của `spec.strategy.rollingUpdate` khi cập nhật giá trị cho `type` là dùng
chiến lược `retainKeys` cho strategic merge.

Tạo một file khác tên là `patch-file-retainkeys.yaml` với nội dung sau:

```yaml
spec:
  strategy:
    $retainKeys:
    - type
    type: Recreate
```

Với bản patch này, chúng ta chỉ ra rằng chúng ta chỉ muốn giữ lại khóa `type` của đối tượng
`strategy`. Do đó, `rollingUpdate` sẽ bị xóa trong quá trình patch.

Patch Deployment của bạn một lần nữa với bản patch mới này:

```shell
kubectl patch deployment retainkeys-demo --type strategic --patch-file patch-file-retainkeys.yaml
```

Kiểm tra nội dung của Deployment:

```shell
kubectl get deployment retainkeys-demo --output yaml
```

Output cho thấy đối tượng strategy trong Deployment không còn chứa khóa `rollingUpdate` nữa:

```yaml
spec:
  strategy:
    type: Recreate
  template:
```

### Ghi chú về strategic merge patch dùng chiến lược retainKeys (Notes on the strategic merge patch using the retainKeys strategy)

Thao tác patch bạn thực hiện trong bài thực hành trên được gọi là *strategic merge patch với
chiến lược retainKeys*. Phương pháp này đưa vào một chỉ thị (directive) mới `$retainKeys` với
các đặc điểm sau:

- Nó chứa một danh sách các chuỗi.
- Tất cả các trường cần được giữ lại phải có mặt trong danh sách `$retainKeys`.
- Các trường có mặt sẽ được trộn với đối tượng đang chạy (live object).
- Tất cả các trường bị thiếu sẽ bị xóa khi patch.
- Tất cả các trường trong danh sách `$retainKeys` phải là tập cha (superset) hoặc trùng với
  các trường có trong bản patch.

Chiến lược `retainKeys` không hoạt động với mọi đối tượng. Nó chỉ hoạt động khi giá trị của
khóa `patchStrategy` trong field tag ở mã nguồn Kubernetes có chứa `retainKeys`. Ví dụ, trường
`Strategy` của struct `DeploymentSpec` có `patchStrategy` là `retainKeys`:

```go
type DeploymentSpec struct {
  ...
  // +patchStrategy=retainKeys
  Strategy DeploymentStrategy `json:"strategy,omitempty" patchStrategy:"retainKeys" ...`
  ...
}
```

Bạn cũng có thể thấy chiến lược `retainKeys` trong
[OpenApi spec](https://raw.githubusercontent.com/kubernetes/kubernetes/master/api/openapi-spec/swagger.json):

```yaml
"io.k8s.api.apps.v1.DeploymentSpec": {
    ...,
    "strategy": {
        "$ref": "#/definitions/io.k8s.api.apps.v1.DeploymentStrategy",
        "description": "The deployment strategy to use to replace existing pods with new ones.",
        "x-kubernetes-patch-strategy": "retainKeys"
    },
    ....
}
```

Và bạn có thể thấy chiến lược `retainKeys` trong
[tài liệu Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#deploymentspec-v1-apps).

### Các dạng khác của lệnh kubectl patch (Alternate forms of the kubectl patch command)

Lệnh `kubectl patch` chấp nhận YAML hoặc JSON. Nó có thể nhận bản patch dưới dạng một file
hoặc trực tiếp trên dòng lệnh.

Tạo một file tên là `patch-file.json` với nội dung sau:

```json
{
   "spec": {
      "template": {
         "spec": {
            "containers": [
               {
                  "name": "patch-demo-ctr-2",
                  "image": "redis"
               }
            ]
         }
      }
   }
}
```

Các lệnh sau là tương đương nhau:

```shell
kubectl patch deployment patch-demo --patch-file patch-file.yaml
kubectl patch deployment patch-demo --patch 'spec:\n template:\n  spec:\n   containers:\n   - name: patch-demo-ctr-2\n     image: redis'

kubectl patch deployment patch-demo --patch-file patch-file.json
kubectl patch deployment patch-demo --patch '{"spec": {"template": {"spec": {"containers": [{"name": "patch-demo-ctr-2","image": "redis"}]}}}}'
```

### Cập nhật số lượng replica của một đối tượng bằng `kubectl patch` với `--subresource` (Update an object's replica count using `kubectl patch` with `--subresource`) {#scale-kubectl-patch}

Flag `--subresource=[subresource-name]` được dùng với các lệnh kubectl như get, patch, edit,
apply và replace để lấy và cập nhật các subresource `status`, `scale` và `resize` của các
resource mà bạn chỉ định. Bạn có thể chỉ định một subresource cho bất kỳ resource nào của
Kubernetes API (dựng sẵn và CR) có subresource `status`, `scale` hoặc `resize`.

Ví dụ, một Deployment có subresource `status` và subresource `scale`, nên bạn có thể dùng
`kubectl` để lấy hoặc chỉnh sửa riêng subresource `status` của một Deployment.

Đây là manifest cho một Deployment có hai replica:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # chỉ định deployment chạy 2 pod khớp với template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

Tạo Deployment:

```shell
kubectl apply -f https://k8s.io/examples/application/deployment.yaml
```

Xem các Pod gắn với Deployment của bạn:

```shell
kubectl get pods -l app=nginx
```

Trong output, bạn có thể thấy Deployment có hai Pod. Ví dụ:

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7fb96c846b-22567   1/1     Running   0          47s
nginx-deployment-7fb96c846b-mlgns   1/1     Running   0          47s
```

Bây giờ, patch Deployment đó với flag `--subresource=[subresource-name]`:

```shell
kubectl patch deployment nginx-deployment --subresource='scale' --type='merge' -p '{"spec":{"replicas":3}}'
```

Output là:

```shell
scale.autoscaling/nginx-deployment patched
```

Xem các Pod gắn với Deployment đã được patch:

```shell
kubectl get pods -l app=nginx
```

Trong output, bạn có thể thấy một pod mới được tạo, nên bây giờ bạn có 3 pod đang chạy.

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7fb96c846b-22567   1/1     Running   0          107s
nginx-deployment-7fb96c846b-lxfr2   1/1     Running   0          14s
nginx-deployment-7fb96c846b-mlgns   1/1     Running   0          107s
```

Xem Deployment đã được patch:

```shell
kubectl get deployment nginx-deployment -o yaml
```

```yaml
...
spec:
  replicas: 3
  ...
status:
  ...
  availableReplicas: 3
  readyReplicas: 3
  replicas: 3
```

> **Ghi chú:**
> Nếu bạn chạy `kubectl patch` và chỉ định flag `--subresource` cho một resource không hỗ trợ
> subresource cụ thể đó, API server sẽ trả về lỗi 404 Not Found.

## Tóm tắt (Summary)

Trong bài thực hành này, bạn đã dùng `kubectl patch` để thay đổi cấu hình đang chạy (live
configuration) của một đối tượng Deployment. Bạn không thay đổi file cấu hình mà bạn đã dùng
ban đầu để tạo đối tượng Deployment. Các lệnh khác để cập nhật đối tượng API bao gồm
[kubectl annotate](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#annotate),
[kubectl edit](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#edit),
[kubectl replace](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#replace),
[kubectl scale](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#scale),
và
[kubectl apply](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#apply).

> **Ghi chú:**
> Strategic merge patch không được hỗ trợ cho custom resource.

## Tiếp theo (What's next)

* [Quản lý đối tượng Kubernetes](27-object-management-vi.md)
* [Quản lý đối tượng Kubernetes bằng lệnh mệnh lệnh (imperative commands)](320-imperative-command-vi.md)
* [Quản lý mệnh lệnh đối tượng Kubernetes bằng file cấu hình](321-imperative-config-vi.md)
* [Quản lý khai báo đối tượng Kubernetes bằng file cấu hình](319-declarative-config-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4:

1. Vẫn `kubectl patch deployment patch-demo --patch-file <file>`, không đổi cờ `type`. Patch vào
   `containers` thì **thêm** một container, patch vào `tolerations` thì **thay thế** cả list. Vì sao
   cùng một loại patch lại cho hai hành vi khác nhau?
2. **Câu bẫy.** Deployment đang có hai container. Bạn chạy patch với `--type merge` và một list
   `containers` chỉ gồm một phần tử mang tên mới. Sau đó mỗi Pod có mấy container, và vì sao trực
   giác "merge nghĩa là trộn" lại sai ở đây?
3. Trên `lab-k8s-master`, một Deployment đang chạy với `strategy.rollingUpdate.maxSurge: 30%`. Bạn
   patch `spec.strategy.type: Recreate` bằng strategic merge patch thường. API server trả lời thế
   nào, và bạn phải sửa bản patch ra sao để nó đi qua?
4. `kubectl patch deployment nginx-deployment --subresource='scale' --type='merge' -p '{"spec":{"replicas":3}}'`
   in ra dòng gì, và chuyện gì xảy ra nếu bạn dùng `--subresource=scale` cho một resource không có
   subresource đó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì hành vi merge được quyết định **theo từng field**, không phải theo loại patch. Chiến lược
   được chỉ định bởi khóa `patchStrategy` trong field tag ở mã nguồn Kubernetes: `Containers` của
   `PodSpec` có **`patchStrategy:"merge"` với `patchMergeKey:"name"`** nên list được trộn và mỗi
   container được nhận diện bằng `name`; `Tolerations` **không có khóa `patchStrategy`** nên
   strategic merge patch dùng chiến lược mặc định là **`replace`**, thay cả list.
2. **Mỗi Pod chỉ còn một container** — bài cho thấy cột `READY` tụt về `1/1`. Trực giác sai vì
   `--type merge` không phải là "strategic merge patch"; nó là **JSON merge patch (RFC 7386)**, một
   thứ khác hẳn. Với JSON merge patch, muốn cập nhật một list thì phải chỉ định **toàn bộ danh sách
   mới**, và danh sách mới **thay thế hoàn toàn** danh sách hiện có. Cái trộn theo `patchMergeKey`
   là `strategic` — đúng giá trị **mặc định** của tham số `type`, tức loại patch bạn dùng khi không
   ghi cờ nào.
3. API server **từ chối**, báo Deployment không hợp lệ tại `spec.strategy.rollingUpdate` với lý do
   `Forbidden` — không thể đặt `type` thành `Recreate` khi `spec.strategy.rollingUpdate` đã có giá
   trị được định nghĩa. Cách sửa là dùng chiến lược
   **`retainKeys`**: thêm `$retainKeys: [type]` vào bản patch cùng với `type: Recreate`. Khai báo đó
   nói rằng chỉ giữ khóa `type` của object `strategy`, nên `rollingUpdate` **bị xóa trong quá trình
   patch**. Nó chạy được vì `Strategy` của `DeploymentSpec` có `patchStrategy:"retainKeys"`.
4. In ra **`scale.autoscaling/nginx-deployment patched`** — tên của **subresource** được patch, không
   phải `deployment.apps/nginx-deployment`. Kết quả là Deployment lên 3 Pod. Nếu resource không hỗ
   trợ subresource được chỉ định, API server trả về lỗi **404 Not Found**. `--subresource` dùng được
   với `get`, `patch`, `edit`, `apply` và `replace`, cho các subresource `status`, `scale` và `resize`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
