# Quản lý Workload (Managing Workloads)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/management/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 4](LO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài 10/14 ·
Kiểm chứng ở Lab 4 (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là bài **vận hành**, không phải bài khái niệm: nó trả lời câu hỏi "biết controller rồi
thì gõ gì hằng ngày". Câu mở đầu bài giả định bạn đã expose ứng dụng qua một Service — bạn
chưa học Service, và không sao cả: mọi thứ trong bài trừ mục canary đều chạy được mà không
cần Service. Bài trùng nội dung một phần với [Deployment](63-deployment-vi.md); ở đây chỉ
cần rút ra các thói quen tổ chức manifest và bộ lệnh cập nhật.

**Phải hiểu ở lần đọc này:**

- Gộp nhiều tài nguyên vào một file bằng `---`, và **thứ tự trong manifest là thứ tự tạo** —
  vì thế nên khai báo Service trước Deployment. `kubectl apply` nhận nhiều tham số `-f`, và
  nhận cả URL.
- Thao tác hàng loạt: lọc bằng `-l`/`--selector` theo label thay vì liệt kê tên; và
  `--recursive`/`-R` để xử lý thư mục con — **mặc định thao tác dừng ở cấp thư mục đầu tiên**
  và báo lỗi nếu cấp đó không có manifest nào.
- Bốn cách sửa tài nguyên và ranh giới giữa chúng: `kubectl apply` (khai báo, chỉ đổi thứ bạn
  chỉ định, không ghi đè thay đổi tự động), `kubectl edit` (đúng bằng get → sửa → apply),
  `kubectl patch` (JSON patch, JSON merge patch, strategic merge patch), và
  `kubectl replace --force` — cái cuối là **cập nhật gây gián đoạn** vì nó xóa rồi tạo lại.
- Quản lý rollout bằng `kubectl rollout status` với `--timeout` khi cần chờ và `--watch=false`
  khi chỉ muốn xem trạng thái; rollout dùng được với **DaemonSet, Deployment và StatefulSet**.
- Canary thủ công: hai bản phát hành khác nhau ở **một label riêng** (bài dùng `track` với giá
  trị `stable` và `canary`), còn các label kia giữ nguyên; **tỷ lệ lưu lượng do số replica
  quyết định** — 3 và 1 trong ví dụ là tỷ lệ 3:1.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ví dụ mở đầu dùng Service `type: LoadBalancer` | cluster lab không có load balancer bên ngoài | giai đoạn 5 |
| Đoạn `selector:` của Service frontend trong *Triển khai canary* | cách Service chọn Pod theo tập con label | giai đoạn 5 |
| Lệnh `kubectl autoscale deployment/my-nginx --min=1 --max=3` trong *Scale ứng dụng của bạn* | chính bài đã ghi: cần sẵn một nguồn metric | bài [72](72-horizontal-pod-autoscale-vi.md) — [nợ lab](labs/README.md#5-sổ-nợ-lab) trả ở Lab 11b |
| *Công cụ bên ngoài* — Helm và Kustomize | công cụ bên thứ ba, không nằm trong lộ trình | không cần |
| Link *server-side apply* trong mục *kubectl apply* | cơ chế quản lý quyền sở hữu field | không cần |

---

Bạn đã triển khai ứng dụng của mình và công bố (expose) nó qua một Service. Giờ thì sao? Kubernetes cung cấp
một số công cụ giúp bạn quản lý việc triển khai ứng dụng, bao gồm scale và cập nhật.

## Tổ chức cấu hình tài nguyên (Organizing resource configurations)

Nhiều ứng dụng yêu cầu tạo nhiều tài nguyên, chẳng hạn một Deployment đi kèm một Service.
Việc quản lý nhiều tài nguyên có thể được đơn giản hóa bằng cách gộp chúng vào cùng một file
(phân tách bằng `---` trong YAML). Ví dụ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-svc
  labels:
    app: nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
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

Nhiều tài nguyên có thể được tạo theo cùng cách như một tài nguyên đơn lẻ:

```shell
kubectl apply -f https://k8s.io/examples/application/nginx-app.yaml
```

```none
service/my-nginx-svc created
deployment.apps/my-nginx created
```

Các tài nguyên sẽ được tạo theo thứ tự chúng xuất hiện trong manifest. Vì vậy, tốt nhất là
khai báo Service trước, vì điều đó đảm bảo scheduler có thể phân tán các pod gắn với
Service ngay khi chúng được các controller (chẳng hạn Deployment) tạo ra.

`kubectl apply` cũng chấp nhận nhiều tham số `-f`:

```shell
kubectl apply -f https://k8s.io/examples/application/nginx/nginx-svc.yaml \
  -f https://k8s.io/examples/application/nginx/nginx-deployment.yaml
```

Một thực hành được khuyến nghị là đặt các tài nguyên liên quan đến cùng một microservice hoặc cùng một
tầng (tier) ứng dụng vào cùng một file, và gộp tất cả các file gắn với ứng dụng của bạn vào cùng một
thư mục. Nếu các tầng của ứng dụng liên kết với nhau qua DNS, bạn có thể triển khai tất cả
các thành phần của stack cùng một lúc.

Một URL cũng có thể được chỉ định làm nguồn cấu hình, rất tiện khi triển khai trực tiếp từ
các manifest trong hệ thống quản lý mã nguồn (source control) của bạn:

```shell
kubectl apply -f https://k8s.io/examples/application/nginx/nginx-deployment.yaml
```

```none
deployment.apps/my-nginx created
```

Nếu bạn cần định nghĩa thêm manifest, chẳng hạn thêm một ConfigMap, bạn cũng có thể làm điều đó.

### Công cụ bên ngoài (External tools)

Mục này chỉ liệt kê những công cụ phổ biến nhất dùng để quản lý workload trên Kubernetes. Để xem
danh sách đầy đủ hơn, hãy xem
[Application definition and image build](https://landscape.cncf.io/guide#app-definition-and-development--application-definition-image-build)
trong CNCF Landscape.

#### Helm {#external-tool-helm}

> **Ghi chú:** Mục này liên kết đến một dự án hoặc sản phẩm bên thứ ba không thuộc Kubernetes.
> Các tác giả của dự án Kubernetes không chịu trách nhiệm về dự án đó. Để biết thêm chi tiết,
> hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content).

[Helm](https://helm.sh/) là công cụ quản lý các gói (package) tài nguyên Kubernetes
đã được cấu hình sẵn. Các gói này được gọi là _Helm chart_.

#### Kustomize {#external-tool-kustomize}

[Kustomize](https://kustomize.io/) duyệt qua một manifest Kubernetes để thêm, xóa hoặc cập nhật
các tùy chọn cấu hình. Nó có sẵn ở cả dạng binary độc lập lẫn dạng
[tính năng gốc (native feature)](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
của kubectl.

## Thao tác hàng loạt trong kubectl (Bulk operations in kubectl)

Tạo tài nguyên không phải là thao tác duy nhất mà `kubectl` có thể thực hiện hàng loạt. Nó còn có thể
trích xuất tên tài nguyên từ các file cấu hình để thực hiện những thao tác khác, đặc biệt là để
xóa chính các tài nguyên bạn đã tạo:

```shell
kubectl delete -f https://k8s.io/examples/application/nginx-app.yaml
```

```none
deployment.apps "my-nginx" deleted
service "my-nginx-svc" deleted
```

Với trường hợp hai tài nguyên, bạn có thể chỉ định cả hai tài nguyên trên dòng lệnh bằng cú pháp
resource/name:

```shell
kubectl delete deployments/my-nginx services/my-nginx-svc
```

Với số lượng tài nguyên lớn hơn, bạn sẽ thấy dễ dàng hơn khi chỉ định selector (truy vấn theo label)
bằng `-l` hoặc `--selector`, để lọc tài nguyên theo label của chúng:

```shell
kubectl delete deployment,services -l app=nginx
```

```none
deployment.apps "my-nginx" deleted
service "my-nginx-svc" deleted
```

### Nối chuỗi lệnh và lọc (Chaining and filtering)

Vì `kubectl` xuất ra tên tài nguyên theo đúng cú pháp mà nó chấp nhận, bạn có thể nối chuỗi
các thao tác bằng `$()` hoặc `xargs`:

```shell
kubectl get $(kubectl create -f docs/concepts/cluster-administration/nginx/ -o name | grep service/ )
kubectl create -f docs/concepts/cluster-administration/nginx/ -o name | grep service/ | xargs -i kubectl get '{}'
```

Kết quả có thể tương tự như sau:

```none
NAME           TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)      AGE
my-nginx-svc   LoadBalancer   10.0.0.208   <pending>     80/TCP       0s
```

Với các lệnh trên, đầu tiên bạn tạo các tài nguyên trong `docs/concepts/cluster-administration/nginx/`
và in ra các tài nguyên đã tạo với định dạng output `-o name` (in mỗi tài nguyên dưới dạng resource/name).
Sau đó bạn `grep` để lấy riêng Service, rồi in nó ra bằng
[`kubectl get`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_get/).

### Thao tác đệ quy trên file cục bộ (Recursive operations on local files)

Nếu bạn tổ chức tài nguyên của mình trong nhiều thư mục con bên trong một thư mục
nhất định, bạn cũng có thể thực hiện các thao tác một cách đệ quy trên các thư mục con đó, bằng cách
chỉ định `--recursive` hoặc `-R` cùng với tham số `--filename`/`-f`.

Ví dụ, giả sử có một thư mục `project/k8s/development` chứa tất cả các
manifest cần cho môi trường phát triển (development), được tổ chức theo loại tài nguyên:

```none
project/k8s/development
├── configmap
│   └── my-configmap.yaml
├── deployment
│   └── my-deployment.yaml
└── pvc
    └── my-pvc.yaml
```

Mặc định, khi thực hiện một thao tác hàng loạt trên `project/k8s/development`, thao tác sẽ dừng
ở cấp đầu tiên của thư mục và không xử lý thư mục con nào. Nếu bạn thử tạo các tài nguyên trong
thư mục này bằng lệnh sau, chúng ta sẽ gặp lỗi:

```shell
kubectl apply -f project/k8s/development
```

```none
error: you must provide one or more resources by argument or filename (.json|.yaml|.yml|stdin)
```

Thay vào đó, hãy chỉ định tham số dòng lệnh `--recursive` hoặc `-R` cùng với tham số `--filename`/`-f`:

```shell
kubectl apply -f project/k8s/development --recursive
```

```none
configmap/my-config created
deployment.apps/my-deployment created
persistentvolumeclaim/my-pvc created
```

Tham số `--recursive` hoạt động với mọi thao tác chấp nhận tham số `--filename`/`-f`, chẳng hạn:
`kubectl create`, `kubectl get`, `kubectl delete`, `kubectl describe`, hay thậm chí `kubectl rollout`.

Tham số `--recursive` cũng hoạt động khi có nhiều tham số `-f` được cung cấp:

```shell
kubectl apply -f project/k8s/namespaces -f project/k8s/development --recursive
```

```none
namespace/development created
namespace/staging created
configmap/my-config created
deployment.apps/my-deployment created
persistentvolumeclaim/my-pvc created
```

Nếu bạn muốn tìm hiểu thêm về `kubectl`, hãy đọc
[Công cụ dòng lệnh (kubectl)](https://kubernetes.io/docs/reference/kubectl/).

## Cập nhật ứng dụng mà không gây gián đoạn (Updating your application without an outage)

Đến một lúc nào đó, bạn rồi sẽ cần cập nhật ứng dụng đã triển khai, thường bằng cách chỉ định
một image mới hoặc một image tag mới. `kubectl` hỗ trợ một số thao tác cập nhật, mỗi thao tác
phù hợp với những tình huống khác nhau.

Bạn có thể chạy nhiều bản sao của ứng dụng, và dùng _rollout_ để chuyển dần lưu lượng sang
các Pod mới khỏe mạnh. Cuối cùng, tất cả các Pod đang chạy sẽ mang phần mềm mới.

Phần này của trang hướng dẫn bạn cách tạo và cập nhật ứng dụng bằng Deployment.

Giả sử bạn đang chạy phiên bản 1.14.2 của nginx:

```shell
kubectl create deployment my-nginx --image=nginx:1.14.2
```

```none
deployment.apps/my-nginx created
```

Đảm bảo rằng có 1 replica:

```shell
kubectl scale --replicas 1 deployments/my-nginx --subresource='scale' --type='merge' -p '{"spec":{"replicas": 1}}'
```

```none
deployment.apps/my-nginx scaled
```

và cho phép Kubernetes thêm các replica tạm thời trong quá trình rollout, bằng cách đặt
_mức surge tối đa_ (surge maximum) là 100%:

```shell
kubectl patch --type='merge' -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge": "100%" }}}}'
```

```none
deployment.apps/my-nginx patched
```

Để cập nhật lên phiên bản 1.16.1, đổi `.spec.template.spec.containers[0].image` từ `nginx:1.14.2`
thành `nginx:1.16.1` bằng `kubectl edit`:

```shell
kubectl edit deployment/my-nginx
# Sửa manifest để dùng container image mới hơn, sau đó lưu thay đổi của bạn
```

Vậy là xong! Deployment sẽ cập nhật ứng dụng nginx đã triển khai một cách khai báo (declarative)
và từng bước ở phía sau. Nó đảm bảo rằng chỉ một số lượng nhất định replica cũ có thể ngừng hoạt động
trong khi chúng đang được cập nhật, và chỉ một số lượng nhất định replica mới có thể được tạo vượt
trên số lượng pod mong muốn. Để tìm hiểu chi tiết hơn về cách điều này diễn ra,
hãy xem [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

Bạn có thể dùng rollout với DaemonSet, Deployment hoặc StatefulSet.

### Quản lý rollout (Managing rollouts)

Bạn có thể dùng [`kubectl rollout`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)
để quản lý việc cập nhật từng bước (progressive update) của một ứng dụng hiện có.

Ví dụ:

```shell
kubectl apply -f my-deployment.yaml

# chờ rollout hoàn tất
kubectl rollout status deployment/my-deployment --timeout 10m # timeout 10 phút
```

hoặc

```shell
kubectl apply -f backing-stateful-component.yaml

# không chờ rollout hoàn tất, chỉ kiểm tra trạng thái
kubectl rollout status statefulsets/backing-stateful-component --watch=false
```

Bạn cũng có thể tạm dừng (pause), tiếp tục (resume) hoặc hủy (cancel) một rollout.
Hãy xem [`kubectl rollout`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)
để tìm hiểu thêm.

## Triển khai canary (Canary deployments)

Một tình huống khác cần đến nhiều label là để phân biệt các lần triển khai của những bản phát hành
(release) hoặc cấu hình khác nhau của cùng một thành phần. Một thực hành phổ biến là triển khai một
bản *canary* của bản phát hành ứng dụng mới (chỉ định qua image tag trong pod template) chạy song song
với bản phát hành trước đó, để bản phát hành mới có thể nhận lưu lượng production thật trước khi được
triển khai hoàn toàn.

Ví dụ, bạn có thể dùng label `track` để phân biệt các bản phát hành khác nhau.

Bản phát hành chính, ổn định sẽ có label `track` với giá trị `stable`:

```none
name: frontend
replicas: 3
...
labels:
   app: guestbook
   tier: frontend
   track: stable
...
image: gb-frontend:v3
```

và sau đó bạn có thể tạo một bản phát hành mới của guestbook frontend mang label `track`
với giá trị khác (tức là `canary`), để hai tập pod không chồng lấn lên nhau:

```none
name: frontend-canary
replicas: 1
...
labels:
   app: guestbook
   tier: frontend
   track: canary
...
image: gb-frontend:v4
```

Service frontend sẽ bao phủ cả hai tập replica bằng cách chọn tập con chung của các label
của chúng (tức là bỏ qua label `track`), nhờ đó lưu lượng sẽ được chuyển hướng đến cả hai
ứng dụng:

```yaml
selector:
   app: guestbook
   tier: frontend
```

Bạn có thể tinh chỉnh số lượng replica của bản phát hành stable và canary để xác định tỷ lệ
lưu lượng production thật mà mỗi bản phát hành sẽ nhận (trong trường hợp này là 3:1).
Khi đã tự tin, bạn có thể cập nhật track stable lên bản phát hành ứng dụng mới và gỡ bỏ
bản canary.

## Cập nhật annotation (Updating annotations)

Đôi khi bạn muốn gắn annotation vào tài nguyên. Annotation là metadata tùy ý, không dùng để
định danh, để các API client như công cụ hoặc thư viện truy xuất.
Việc này có thể thực hiện bằng `kubectl annotate`. Ví dụ:

```shell
kubectl annotate pods my-nginx-v4-9gw19 description='my frontend running nginx'
kubectl get pods my-nginx-v4-9gw19 -o yaml
```

```shell
apiVersion: v1
kind: pod
metadata:
  annotations:
    description: my frontend running nginx
...
```

Để biết thêm thông tin, hãy xem [annotation](./20-annotations-vi.md)
và [kubectl annotate](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_annotate/).

## Scale ứng dụng của bạn (Scaling your application)

Khi tải trên ứng dụng của bạn tăng hoặc giảm, hãy dùng `kubectl` để scale ứng dụng.
Ví dụ, để giảm số lượng replica của nginx từ 3 xuống 1, hãy chạy:

```shell
kubectl scale deployment/my-nginx --replicas=1
```

```none
deployment.apps/my-nginx scaled
```

Bây giờ bạn chỉ còn một pod do deployment quản lý.

```shell
kubectl get pods -l app=my-nginx
```

```none
NAME                        READY     STATUS    RESTARTS   AGE
my-nginx-2035384211-j5fhi   1/1       Running   0          30m
```

Để hệ thống tự động chọn số lượng replica nginx theo nhu cầu,
trong khoảng từ 1 đến 3, hãy chạy:

```shell
# Việc này yêu cầu đã có sẵn một nguồn metric cho container và Pod
kubectl autoscale deployment/my-nginx --min=1 --max=3
```

```none
horizontalpodautoscaler.autoscaling/my-nginx autoscaled
```

Bây giờ các replica nginx của bạn sẽ được tự động scale lên và xuống theo nhu cầu.

Để biết thêm thông tin, hãy xem tài liệu
[kubectl scale](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_scale/),
[kubectl autoscale](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_autoscale/) và
[horizontal pod autoscaler](https://kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/).

## Cập nhật tài nguyên tại chỗ (In-place updates of resources)

Đôi khi cần thực hiện những cập nhật nhỏ, hẹp và không gây gián đoạn trên các tài nguyên bạn đã tạo.

### kubectl apply

Bạn nên duy trì một tập các file cấu hình trong hệ thống quản lý mã nguồn
(xem [configuration as code](https://martinfowler.com/bliki/InfrastructureAsCode.html)),
để chúng có thể được bảo trì và quản lý phiên bản cùng với mã nguồn của các tài nguyên mà chúng cấu hình.
Sau đó, bạn có thể dùng [`kubectl apply`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_apply/)
để đẩy các thay đổi cấu hình của bạn lên cluster.

Lệnh này sẽ so sánh phiên bản cấu hình bạn đang đẩy lên với phiên bản trước đó và áp dụng
những thay đổi bạn đã thực hiện, mà không ghi đè bất kỳ thay đổi tự động nào trên các thuộc tính
mà bạn không chỉ định.

```shell
kubectl apply -f https://k8s.io/examples/application/nginx/nginx-deployment.yaml
```

```none
deployment.apps/my-nginx configured
```

Để tìm hiểu thêm về cơ chế bên dưới, hãy đọc
[server-side apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/).

### kubectl edit

Ngoài ra, bạn cũng có thể cập nhật tài nguyên bằng
[`kubectl edit`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/):

```shell
kubectl edit deployment/my-nginx
```

Cách này tương đương với việc đầu tiên `get` tài nguyên, chỉnh sửa nó trong trình soạn thảo văn bản,
rồi `apply` tài nguyên với phiên bản đã cập nhật:

```shell
kubectl get deployment my-nginx -o yaml > /tmp/nginx.yaml
vi /tmp/nginx.yaml
# chỉnh sửa, sau đó lưu file

kubectl apply -f /tmp/nginx.yaml
deployment.apps/my-nginx configured

rm /tmp/nginx.yaml
```

Cách này cho phép bạn thực hiện những thay đổi lớn hơn một cách dễ dàng hơn. Lưu ý rằng bạn có thể
chỉ định trình soạn thảo qua biến môi trường `EDITOR` hoặc `KUBE_EDITOR`.

Để biết thêm thông tin, hãy xem
[kubectl edit](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_edit/).

### kubectl patch

Bạn có thể dùng [`kubectl patch`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_patch/)
để cập nhật các đối tượng API tại chỗ. Lệnh con này hỗ trợ JSON patch,
JSON merge patch, và strategic merge patch.

Xem
[Update API Objects in Place Using kubectl patch](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/)
để biết thêm chi tiết.

## Cập nhật gây gián đoạn (Disruptive updates)

Trong một số trường hợp, bạn có thể cần cập nhật các trường tài nguyên không thể cập nhật sau khi
đã khởi tạo, hoặc bạn muốn thực hiện ngay lập tức một thay đổi đệ quy, chẳng hạn để sửa các pod hỏng
do một Deployment tạo ra. Để thay đổi những trường như vậy, hãy dùng `replace --force`; lệnh này xóa
và tạo lại tài nguyên. Trong trường hợp này, bạn có thể sửa trực tiếp file cấu hình gốc của mình:

```shell
kubectl replace -f https://k8s.io/examples/application/nginx/nginx-deployment.yaml --force
```

```none
deployment.apps/my-nginx deleted
deployment.apps/my-nginx replaced
```

## Tiếp theo (What's next)

- Tìm hiểu [cách dùng `kubectl` để khảo sát (introspect) và debug ứng dụng](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 4:

1. Trong một file gộp cả Service và Deployment, vì sao bài khuyên khai báo Service trước?
2. `kubectl apply -f project/k8s/development` trả về
   `error: you must provide one or more resources by argument or filename`, dù thư mục đó có
   đủ manifest trong các thư mục con. Vì sao, và sửa thế nào?
3. **Câu bẫy.** `kubectl apply` và `kubectl replace --force` khác nhau ở đâu về mặt gián đoạn
   dịch vụ, và khi nào bạn buộc phải dùng cái sau?
4. Bạn muốn khoảng 25% lưu lượng production đi vào bản mới theo cách canary thủ công trong
   bài. Bạn chỉnh cái gì, label nào phải khác nhau và label nào phải giữ giống nhau?
5. Trên `k8s-master`, bạn viết một script apply Deployment rồi phải **dừng chờ** tới khi
   rollout xong, tối đa 10 phút. Lệnh nào? Nếu chỉ muốn xem trạng thái mà không chờ thì sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Vì **các tài nguyên được tạo theo đúng thứ tự chúng xuất hiện trong manifest**. Bài nêu lý
   do cụ thể: khai báo Service trước "đảm bảo scheduler có thể phân tán các pod gắn với
   Service ngay khi chúng được các controller (chẳng hạn Deployment) tạo ra". Làm ngược lại
   thì Pod đã được lập lịch xong trước khi Service tồn tại.
2. Vì **mặc định, thao tác hàng loạt dừng ở cấp đầu tiên của thư mục và không xử lý thư mục
   con nào** — cấp đầu tiên chỉ có thư mục, không có file manifest, nên `kubectl` báo không
   tìm thấy tài nguyên. Sửa bằng cách thêm **`--recursive` hoặc `-R`** cùng tham số
   `--filename`/`-f`. Tham số này hoạt động với mọi thao tác nhận `-f` — `create`, `get`,
   `delete`, `describe`, kể cả `rollout` — và hoạt động cả khi bạn truyền nhiều `-f`.
3. `kubectl apply` **so sánh phiên bản cấu hình bạn đẩy lên với phiên bản trước đó và chỉ áp
   dụng phần thay đổi**, không ghi đè các thuộc tính bạn không chỉ định — tài nguyên vẫn sống,
   Deployment tự lo rollout êm. `kubectl replace --force` thì **xóa và tạo lại tài nguyên** —
   bài in ra đúng hai dòng `deleted` rồi `replaced`. Trực giác "force chỉ là apply mạnh tay
   hơn" là sai: đây là cập nhật **gây gián đoạn**. Bạn buộc phải dùng nó khi cần sửa các
   trường **không thể cập nhật sau khi đã khởi tạo**, hoặc khi muốn ép ngay một thay đổi đệ
   quy, chẳng hạn để sửa các Pod hỏng do một Deployment tạo ra.
4. Bạn **chỉnh số replica của hai bản phát hành** — bài dùng 3 cho `stable` và 1 cho `canary`
   để được tỷ lệ 3:1, tức khoảng 25% cho bản mới. Label **phải khác nhau** là label phân biệt
   bản phát hành, ở đây là `track` (`stable` với `canary`), và hai Deployment cũng mang tên
   khác nhau (`frontend`, `frontend-canary`). Các label **phải giữ giống nhau** là phần chung
   mà Service dùng để bao phủ cả hai tập Pod — trong ví dụ là `app: guestbook` và
   `tier: frontend`. Khi đã tự tin, bạn cập nhật track `stable` lên bản mới rồi gỡ bản canary.
5. `kubectl rollout status deployment/my-deployment --timeout 10m`. Nếu chỉ muốn xem trạng
   thái mà không chờ, thêm `--watch=false` — bài đưa đúng ví dụ đó cho một StatefulSet. Ngoài
   ra bạn còn có thể tạm dừng, tiếp tục hoặc hủy một rollout bằng `kubectl rollout`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
