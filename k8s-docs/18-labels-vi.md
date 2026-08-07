# Label và Selector (Labels and Selectors)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/>
>
> Label là các cặp key/value được gắn vào các object như Pod, dùng để chỉ định
> các thuộc tính nhận dạng có ý nghĩa với người dùng; selector cho phép chọn ra
> các tập con object dựa trên label.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1b](LO-TRINH-ADMIN.md#1b-làm-việc-với-object-và-kubectl),
bài 2/9 · Kiểm chứng ở Lab 1b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Đây là bài quan trọng nhất nhóm 1b.** Selector là cơ chế mà mọi controller và Service dùng
để tìm Pod — không nắm bài này thì giai đoạn 4 và 5 sẽ học vẹt.

**Phải hiểu ở lần đọc này:**

- Label **không đảm bảo duy nhất** — đây là khác biệt cốt lõi với name. Nhiều object cùng mang
  một label là chuyện bình thường và là mục đích thiết kế.
- Hai loại requirement: **dựa trên đẳng thức** (`=`, `==`, `!=`) và **dựa trên tập hợp**
  (`in`, `notin`, `exists`, `!`).
- Dấu phẩy là **AND**. Không có toán tử OR giữa các requirement — nhưng `in (a, b)` là OR trên
  *value* của cùng một key.
- Cái bẫy lớn nhất: `tier != frontend` **cũng chọn cả object không có key `tier`**. Tương tự
  với `notin`.
- `matchLabels` và `matchExpressions` là dạng selector trong manifest; `matchLabels` chỉ là
  cách viết gọn của `matchExpressions` với toán tử `In`.
- Ba lệnh phải thạo: `kubectl get -l`, `kubectl get -L`, `kubectl label`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| `nodeSelector` và ví dụ GPU `accelerator` | thuộc chủ đề lập lịch | giai đoạn 7 |
| *Service và ReplicationController* | chưa học Service | giai đoạn 5 |
| Ghi chú selector của hai ReplicaSet không được chồng lấn | chưa học ReplicaSet | giai đoạn 4 |
| Chuỗi truy vấn URL đã mã hóa trong mục *Lọc trong LIST và WATCH* | chi tiết cho người viết client | khi viết client |
| Ví dụ guestbook nhiều tầng | minh họa, không cần chạy | đọc lấy ý |

---

_Label_ (nhãn) là các cặp key/value được gắn vào các object như Pod.
Label được thiết kế để chỉ định các thuộc tính nhận dạng (identifying attributes) của object
có ý nghĩa và liên quan đến người dùng, nhưng không trực tiếp mang ngữ nghĩa
đối với hệ thống lõi. Label có thể được dùng để tổ chức và chọn ra các tập con
object. Label có thể được gắn vào object lúc tạo và sau đó
được thêm hoặc sửa đổi vào bất kỳ lúc nào. Mỗi object có thể có một tập các label key/value
được định nghĩa. Mỗi Key phải là duy nhất đối với một object cho trước.

```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

Label cho phép truy vấn (query) và theo dõi (watch) hiệu quả, và rất lý tưởng để dùng trong giao diện người dùng (UI)
và giao diện dòng lệnh (CLI). Những thông tin không dùng để nhận dạng nên được ghi lại bằng
[annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/).

## Động lực (Motivation)

Label cho phép người dùng ánh xạ cấu trúc tổ chức của riêng họ lên các object của hệ thống
theo cách ghép nối lỏng lẻo (loosely coupled), mà không yêu cầu client phải lưu trữ các ánh xạ này.

Việc triển khai service và các pipeline xử lý theo lô (batch processing) thường là những thực thể đa chiều
(ví dụ: nhiều phân vùng (partition) hoặc nhiều bản triển khai, nhiều nhánh phát hành (release track), nhiều tầng (tier),
nhiều micro-service trên mỗi tầng). Việc quản lý thường đòi hỏi các thao tác xuyên suốt (cross-cutting),
điều này phá vỡ tính đóng gói của các cách biểu diễn phân cấp cứng nhắc, đặc biệt là những
phân cấp cứng do hạ tầng quyết định thay vì do người dùng quyết định.

Ví dụ về label:

* `"release" : "stable"`, `"release" : "canary"`
* `"environment" : "dev"`, `"environment" : "qa"`, `"environment" : "production"`
* `"tier" : "frontend"`, `"tier" : "backend"`, `"tier" : "cache"`
* `"partition" : "customerA"`, `"partition" : "customerB"`
* `"track" : "daily"`, `"track" : "weekly"`

Đây là các ví dụ về
[những label thường dùng](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/);
bạn hoàn toàn có thể tự phát triển quy ước đặt tên của riêng mình.
Hãy nhớ rằng label Key phải là duy nhất đối với một object cho trước.

## Cú pháp và bộ ký tự (Syntax and character set)

_Label_ là các cặp key/value. Label key hợp lệ có hai phân đoạn (segment): một
prefix (tiền tố) tùy chọn và name (tên), phân tách bằng dấu gạch chéo (`/`). Phân đoạn name là bắt buộc và
phải có tối đa 63 ký tự, bắt đầu và kết thúc bằng một ký tự chữ-số
(`[a-z0-9A-Z]`), ở giữa có thể chứa dấu gạch ngang (`-`), gạch dưới (`_`), dấu chấm (`.`)
và các ký tự chữ-số. Prefix là tùy chọn. Nếu được chỉ định, prefix
phải là một DNS subdomain: một chuỗi các DNS label phân tách bằng dấu chấm (`.`),
tổng độ dài không quá 253 ký tự, theo sau là dấu gạch chéo (`/`).

Nếu prefix bị bỏ qua, label Key được coi là riêng tư đối với người dùng.
Các thành phần hệ thống tự động (ví dụ `kube-scheduler`, `kube-controller-manager`,
`kube-apiserver`, `kubectl`, hoặc các công cụ tự động hóa bên thứ ba khác) khi thêm label
vào object của người dùng cuối thì bắt buộc phải chỉ định prefix.

Các prefix `kubernetes.io/` và `k8s.io/` được
[dành riêng](https://kubernetes.io/docs/reference/labels-annotations-taints/) cho các thành phần lõi của Kubernetes.

Label value hợp lệ:

* phải có tối đa 63 ký tự (có thể để trống),
* nếu không trống, phải bắt đầu và kết thúc bằng một ký tự chữ-số (`[a-z0-9A-Z]`),
* ở giữa có thể chứa dấu gạch ngang (`-`), gạch dưới (`_`), dấu chấm (`.`) và các ký tự chữ-số.

Ví dụ, đây là manifest của một Pod có hai label
`environment: production` và `app: nginx`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

## Bộ chọn label (Label selectors) {#label-selectors}

Không giống như [name và UID](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/), label
không đảm bảo tính duy nhất. Nói chung, chúng ta kỳ vọng nhiều object cùng mang một (hoặc nhiều) label giống nhau.

Thông qua một _label selector_ (bộ chọn label), client/người dùng có thể xác định một tập các object.
Label selector là cơ chế gom nhóm (grouping primitive) cốt lõi trong Kubernetes.

API hiện hỗ trợ hai loại selector: _dựa trên đẳng thức_ (equality-based) và _dựa trên tập hợp_ (set-based).
Một label selector có thể gồm nhiều _yêu cầu_ (requirement) phân tách bằng dấu phẩy.
Trong trường hợp có nhiều yêu cầu, tất cả đều phải được thỏa mãn, do đó dấu phẩy
đóng vai trò như toán tử logic _AND_ (`&&`).

Ngữ nghĩa của selector rỗng hoặc không được chỉ định phụ thuộc vào ngữ cảnh,
và các kiểu API sử dụng selector nên tự tài liệu hóa tính hợp lệ cũng như ý nghĩa
của chúng.

> **Ghi chú:**
> Với một số kiểu API, chẳng hạn ReplicaSet, label selector của hai instance không được
> chồng lấn (overlap) trong cùng một namespace, nếu không controller có thể coi đó là các
> chỉ thị mâu thuẫn và không xác định được cần có bao nhiêu replica.

> **Thận trọng:**
> Với cả điều kiện dựa trên đẳng thức lẫn dựa trên tập hợp, không có toán tử logic _OR_ (`||`).
> Hãy đảm bảo các câu lệnh lọc của bạn được cấu trúc cho phù hợp.

### Yêu cầu _dựa trên đẳng thức_ (Equality-based requirement)

Yêu cầu _dựa trên đẳng thức_ (equality-based) hoặc _bất đẳng thức_ (inequality-based) cho phép lọc theo label key và value.
Các object khớp phải thỏa mãn tất cả các ràng buộc label được chỉ định, mặc dù chúng vẫn có thể
có thêm các label khác. Ba loại toán tử được chấp nhận: `=`, `==`, `!=`.
Hai toán tử đầu biểu diễn _đẳng thức_ (và là đồng nghĩa với nhau), còn toán tử cuối biểu diễn _bất đẳng thức_.
Ví dụ:

```
environment = production
tier != frontend
```

Câu lệnh đầu chọn tất cả các resource có key bằng `environment` và value bằng `production`.
Câu lệnh sau chọn tất cả các resource có key bằng `tier` và value khác `frontend`,
cùng với tất cả các resource không có label nào với key `tier`. Bạn có thể lọc các resource trong `production`
nhưng loại trừ `frontend` bằng toán tử dấu phẩy: `environment=production,tier!=frontend`

Một kịch bản sử dụng của yêu cầu label dựa trên đẳng thức là để Pod chỉ định
tiêu chí chọn node. Ví dụ, Pod mẫu bên dưới chọn các node có
label `accelerator` tồn tại và được đặt giá trị `nvidia-tesla-p100`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  containers:
    - name: cuda-test
      image: "registry.k8s.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-p100
```

### Yêu cầu _dựa trên tập hợp_ (Set-based requirement) {#set-based-requirement}

Yêu cầu label _dựa trên tập hợp_ (set-based) cho phép lọc key theo một tập các giá trị.
Ba loại toán tử được hỗ trợ: `in`, `notin` và `exists` (chỉ áp dụng cho định danh key).
Ví dụ:

```
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```

- Ví dụ thứ nhất chọn tất cả các resource có key bằng `environment` và value
  bằng `production` hoặc `qa`.
- Ví dụ thứ hai chọn tất cả các resource có key bằng `tier` và value khác
  `frontend` và `backend`, cùng với tất cả các resource không có label nào với key `tier`.
- Ví dụ thứ ba chọn tất cả các resource có chứa label với key `partition`;
  không kiểm tra value.
- Ví dụ thứ tư chọn tất cả các resource không có label với key `partition`;
  không kiểm tra value.

Tương tự, dấu phẩy đóng vai trò như toán tử _AND_. Do đó, việc lọc các resource
có key `partition` (bất kể value là gì) và có `environment` khác
`qa` có thể thực hiện bằng `partition,environment notin (qa)`.
Label selector _dựa trên tập hợp_ là dạng tổng quát của đẳng thức, vì
`environment=production` tương đương với `environment in (production)`;
tương tự cho `!=` và `notin`.

Yêu cầu _dựa trên tập hợp_ có thể trộn lẫn với yêu cầu _dựa trên đẳng thức_.
Ví dụ: `partition in (customerA, customerB),environment!=qa`.

## API

### Lọc trong LIST và WATCH (LIST and WATCH filtering)

Với các thao tác **list** và **watch**, bạn có thể chỉ định label selector để lọc tập các object
được trả về; bạn chỉ định bộ lọc bằng một tham số truy vấn (query parameter).
(Để tìm hiểu chi tiết về watch trong Kubernetes, hãy đọc
[phát hiện thay đổi hiệu quả](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)).
Cả hai loại yêu cầu đều được phép
(được trình bày ở đây như khi chúng xuất hiện trong chuỗi truy vấn URL):

* yêu cầu _dựa trên đẳng thức_: `?labelSelector=environment%3Dproduction,tier%3Dfrontend`
* yêu cầu _dựa trên tập hợp_: `?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29`

Cả hai kiểu label selector đều có thể được dùng để list hoặc watch các resource qua một REST client.
Ví dụ, khi làm việc với `apiserver` bằng `kubectl` và dùng kiểu _dựa trên đẳng thức_, bạn có thể viết:

```shell
kubectl get pods -l environment=production,tier=frontend
```

hoặc dùng yêu cầu _dựa trên tập hợp_:

```shell
kubectl get pods -l 'environment in (production),tier in (frontend)'
```

Như đã đề cập, yêu cầu _dựa trên tập hợp_ có khả năng biểu đạt cao hơn.
Chẳng hạn, chúng có thể thực hiện toán tử _OR_ trên các value:

```shell
kubectl get pods -l 'environment in (production, qa)'
```

hoặc giới hạn việc khớp phủ định (negative matching) qua toán tử _notin_:

```shell
kubectl get pods -l 'environment,environment notin (frontend)'
```

### Tham chiếu tập hợp trong các object API (Set references in API objects)

Một số object của Kubernetes, chẳng hạn [`services`](https://kubernetes.io/docs/concepts/services-networking/service/)
và [`replicationcontrollers`](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/),
cũng dùng label selector để chỉ định tập các resource khác, chẳng hạn
[pods](https://kubernetes.io/docs/concepts/workloads/pods/).

#### Service và ReplicationController

Tập các pod mà một `service` nhắm tới được định nghĩa bằng một label selector.
Tương tự, quần thể các pod mà một `replicationcontroller` cần
quản lý cũng được định nghĩa bằng một label selector.

Label selector cho cả hai object này được định nghĩa trong file `json` hoặc `yaml` dưới dạng map,
và chỉ hỗ trợ selector với yêu cầu _dựa trên đẳng thức_:

```json
"selector": {
    "component" : "redis",
}
```

hoặc

```yaml
selector:
  component: redis
```

Selector này (lần lượt ở định dạng `json` hoặc `yaml`) tương đương với
`component=redis` hoặc `component in (redis)`.

#### Các resource hỗ trợ yêu cầu dựa trên tập hợp (Resources that support set-based requirements)

Các resource mới hơn, chẳng hạn [`Job`](https://kubernetes.io/docs/concepts/workloads/controllers/job/),
[`Deployment`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/),
[`ReplicaSet`](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) và
[`DaemonSet`](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/),
cũng hỗ trợ yêu cầu _dựa trên tập hợp_.

```yaml
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - { key: tier, operator: In, values: [cache] }
    - { key: environment, operator: NotIn, values: [dev] }
```

`matchLabels` là một map các cặp `{key,value}`. Một cặp `{key,value}` đơn lẻ trong
map `matchLabels` tương đương với một phần tử của `matchExpressions`, trong đó trường `key`
là "key", `operator` là "In", và mảng `values` chỉ chứa "value".
`matchExpressions` là danh sách các yêu cầu chọn pod (pod selector requirement). Các toán tử hợp lệ gồm
In, NotIn, Exists và DoesNotExist. Tập values không được rỗng trong trường hợp
In và NotIn. Tất cả các yêu cầu, từ cả `matchLabels` lẫn `matchExpressions`,
được kết hợp bằng phép AND — tất cả đều phải được thỏa mãn thì mới khớp.

#### Chọn tập hợp node (Selecting sets of nodes)

Một trường hợp sử dụng của việc chọn theo label là để ràng buộc tập các node mà
một pod có thể được lập lịch (schedule) lên. Xem tài liệu về
[chọn node](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) để biết thêm thông tin.

## Sử dụng label hiệu quả (Using labels effectively)

Bạn có thể áp dụng một label duy nhất cho bất kỳ resource nào, nhưng đây không phải lúc nào cũng là
cách làm tốt nhất. Có nhiều kịch bản trong đó nên dùng nhiều label để
phân biệt các tập resource với nhau.

Chẳng hạn, các ứng dụng khác nhau sẽ dùng các giá trị khác nhau cho label `app`, nhưng một
ứng dụng nhiều tầng (multi-tier), như [ví dụ guestbook](https://github.com/kubernetes/examples/tree/master/web/guestbook/),
sẽ cần thêm cách phân biệt từng tầng.

Trong các ví dụ sau, label `app` được đưa vào cho tiện khi truy vấn thủ công
và dùng CLI đơn giản. Label `app.kubernetes.io/name` tuân theo quy ước gán nhãn
được Kubernetes khuyến nghị và phù hợp hơn cho công cụ và tự động hóa.

Frontend có thể mang các label sau:

```yaml
labels:
  app: guestbook
  app.kubernetes.io/name: guestbook
  tier: frontend
```

trong khi Redis master và replica sẽ có label `tier` khác nhau, và thậm chí có thể có thêm
label `role`:

```yaml
labels:
  app: guestbook
  app.kubernetes.io/name: guestbook
  tier: backend
  role: master
```

và

```yaml
labels:
  app: guestbook
  app.kubernetes.io/name: guestbook
  tier: backend
  role: replica
```

Các label cho phép cắt lát (slicing and dicing) các resource theo bất kỳ chiều nào được label chỉ định:

```shell
kubectl apply -f examples/guestbook/all-in-one/guestbook-all-in-one.yaml
kubectl get pods -Lapp -Ltier -Lrole
```

```none
NAME                           READY  STATUS    RESTARTS   AGE   APP         TIER       ROLE
guestbook-fe-4nlpb             1/1    Running   0          1m    guestbook   frontend   <none>
guestbook-fe-ght6d             1/1    Running   0          1m    guestbook   frontend   <none>
guestbook-fe-jpy62             1/1    Running   0          1m    guestbook   frontend   <none>
guestbook-redis-master-5pg3b   1/1    Running   0          1m    guestbook   backend    master
guestbook-redis-replica-2q2yf  1/1    Running   0          1m    guestbook   backend    replica
guestbook-redis-replica-qgazl  1/1    Running   0          1m    guestbook   backend    replica
my-nginx-divi2                 1/1    Running   0          29m   nginx       <none>     <none>
my-nginx-o0ef1                 1/1    Running   0          29m   nginx       <none>     <none>
```

```shell
kubectl get pods -lapp=guestbook,role=replica
```

```none
NAME                           READY  STATUS   RESTARTS  AGE
guestbook-redis-replica-2q2yf  1/1    Running  0         3m
guestbook-redis-replica-qgazl  1/1    Running  0         3m
```

## Cập nhật label (Updating labels)

Đôi khi bạn muốn gán lại nhãn (relabel) cho các pod và resource hiện có trước khi tạo
resource mới. Việc này có thể thực hiện bằng `kubectl label`.
Ví dụ, nếu bạn muốn gán nhãn tất cả các Pod NGINX của mình là tầng frontend, hãy chạy:

```shell
kubectl label pods -l app=nginx tier=fe
```

```none
pod/my-nginx-2035384211-j5fhi labeled
pod/my-nginx-2035384211-u2c7e labeled
pod/my-nginx-2035384211-u3t6x labeled
```

Lệnh này trước tiên lọc tất cả các pod có label "app=nginx", sau đó gán nhãn "tier=fe" cho chúng.
Để xem các pod bạn vừa gán nhãn, chạy:

```shell
kubectl get pods -l app=nginx -L tier
```

```none
NAME                        READY     STATUS    RESTARTS   AGE       TIER
my-nginx-2035384211-j5fhi   1/1       Running   0          23m       fe
my-nginx-2035384211-u2c7e   1/1       Running   0          23m       fe
my-nginx-2035384211-u3t6x   1/1       Running   0          23m       fe
```

Kết quả hiển thị tất cả các pod "app=nginx", kèm một cột label bổ sung là tier của pod
(được chỉ định bằng `-L` hoặc `--label-columns`).

Để biết thêm thông tin, xem [kubectl label](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#label).

## Tiếp theo (What's next)

- Tìm hiểu cách [thêm label vào node](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/#add-a-label-to-a-node)
- Tra cứu [các Label, Annotation và Taint phổ biến (Well-known labels, Annotations and Taints)](https://kubernetes.io/docs/reference/labels-annotations-taints/)
- Xem [các label được khuyến nghị (Recommended labels)](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)
- [Áp dụng Pod Security Standards bằng Label của Namespace](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)
- Đọc bài blog [Viết một Controller cho Pod Labels](https://kubernetes.io/blog/2021/06/21/writing-a-controller-for-pod-labels/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

1. Khác biệt cơ bản giữa **name** và **label** là gì? Vì sao không thể dùng name để làm việc
   mà label làm?
2. `kubectl get pods -l 'tier!=frontend'`. Một Pod hoàn toàn **không có** label key `tier` có
   nằm trong kết quả không?
3. Viết một selector chọn các Pod thuộc `production` hoặc `qa`, nhưng loại các Pod có
   `tier=frontend`.
4. Bài nói không có toán tử OR, nhưng `environment in (production, qa)` trông rất giống OR.
   Mâu thuẫn ở đâu?
5. `kubectl get pods -l tier=fe` và `kubectl get pods -L tier` khác nhau thế nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Name **duy nhất** và dùng để trỏ tới đúng **một** object; label **không duy nhất** và dùng
   để mô tả thuộc tính nhận dạng, cho phép chọn ra **một tập** object. Muốn nói "tất cả Pod
   của frontend ở production" thì name bó tay, vì bạn phải liệt kê từng tên và danh sách đó
   thay đổi liên tục.
2. **Có.** Requirement bất đẳng thức chọn cả các resource không mang key đó. Đây là chỗ dễ sai
   nhất: `!=` không có nghĩa là "có key này và giá trị khác".
3. `environment in (production,qa),tier notin (frontend)` — hoặc trộn hai kiểu:
   `environment in (production,qa),tier!=frontend`. Lưu ý cách viết thứ hai cũng lấy cả Pod
   không có key `tier`.
4. Không mâu thuẫn: OR bị cấm **giữa các requirement** (không thể viết "environment=production
   HOẶC tier=backend"). Còn `in (production, qa)` là tập giá trị **của cùng một key** — vẫn chỉ
   là một requirement.
5. `-l` **lọc**: chỉ trả về Pod khớp. `-L` **hiển thị**: giữ nguyên tập kết quả nhưng thêm một
   cột in ra giá trị label đó. Hai cờ hoàn toàn độc lập và thường dùng chung.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
