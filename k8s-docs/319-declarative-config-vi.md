# Quản lý object Kubernetes theo kiểu khai báo bằng file cấu hình (Declarative Management of Kubernetes Objects Using Configuration Files)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/>
>
> Tài liệu này trình bày cách tạo, cập nhật và xóa object Kubernetes bằng `kubectl apply`
> theo kiểu khai báo, cùng cơ chế apply tính toán patch từ file cấu hình, cấu hình thực tế
> (live configuration) và annotation `last-applied-configuration`.


Các object Kubernetes có thể được tạo, cập nhật và xóa bằng cách lưu nhiều file cấu hình
object trong một thư mục và dùng `kubectl apply` để tạo và cập nhật đệ quy các object đó khi
cần. Phương pháp này giữ lại các thay đổi đã ghi vào object đang chạy (live object) mà không
cần merge ngược các thay đổi đó vào file cấu hình. `kubectl diff` cũng cho bạn xem trước
những thay đổi mà `apply` sẽ thực hiện.

## Trước khi bạn bắt đầu (Before you begin)

Cài đặt [`kubectl`](185-tools-vi.md).

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Khuyến nghị chạy hướng dẫn này trên một cluster có ít nhất hai
node không đóng vai trò máy chủ control plane. Nếu chưa có cluster, bạn có thể tạo một
cluster bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy chạy `kubectl version`.

## Đánh đổi (Trade-offs)

Công cụ `kubectl` hỗ trợ ba kiểu quản lý object:

* Câu lệnh mệnh lệnh (imperative commands)
* Cấu hình object kiểu mệnh lệnh (imperative object configuration)
* Cấu hình object kiểu khai báo (declarative object configuration)

Xem [Quản lý object trong Kubernetes](27-object-management-vi.md)
để hiểu phần thảo luận về ưu và nhược điểm của từng kiểu quản lý object.

## Tổng quan (Overview)

Cấu hình object kiểu khai báo đòi hỏi bạn hiểu vững về định nghĩa và cấu hình object
Kubernetes. Hãy đọc và hoàn thành các tài liệu sau nếu bạn chưa làm:

* [Quản lý object Kubernetes bằng câu lệnh mệnh lệnh](320-imperative-command-vi.md)
* [Quản lý object Kubernetes kiểu mệnh lệnh bằng file cấu hình](321-imperative-config-vi.md)

Dưới đây là định nghĩa các thuật ngữ dùng trong tài liệu này:

- *file cấu hình object / file cấu hình (object configuration file / configuration file)*:
  Một file định nghĩa cấu hình cho một object Kubernetes. Chủ đề này trình bày cách truyền
  các file cấu hình cho `kubectl apply`. File cấu hình thường được lưu trong hệ thống quản
  lý mã nguồn, chẳng hạn Git.
- *cấu hình object thực tế / cấu hình thực tế (live object configuration / live
  configuration)*: Các giá trị cấu hình thực tế của một object, như cluster Kubernetes quan
  sát thấy. Chúng được giữ trong kho lưu trữ của cluster Kubernetes, thường là etcd.
- *người ghi cấu hình khai báo / người ghi khai báo (declarative configuration writer /
  declarative writer)*: Một người hoặc một thành phần phần mềm thực hiện cập nhật lên object
  đang chạy. Những người ghi được nhắc tới trong chủ đề này thay đổi các file cấu hình
  object và chạy `kubectl apply` để ghi các thay đổi đó.

## Cách tạo object (How to create objects)

Dùng `kubectl apply` để tạo tất cả các object — trừ những object đã tồn tại — được định
nghĩa bởi các file cấu hình trong một thư mục chỉ định:

```shell
kubectl apply -f <directory>
```

Lệnh này đặt annotation `kubectl.kubernetes.io/last-applied-configuration: '{...}'`
trên mỗi object. Annotation này chứa nội dung của file cấu hình object đã được dùng
để tạo object đó.

> **Ghi chú:** Thêm cờ `-R` để xử lý đệ quy các thư mục con.

Đây là một ví dụ về file cấu hình object:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
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

Chạy `kubectl diff` để in ra object sẽ được tạo:

```shell
kubectl diff -f https://k8s.io/examples/application/simple_deployment.yaml
```

> **Ghi chú:** `diff` sử dụng [server-side dry-run](https://kubernetes.io/docs/reference/using-api/api-concepts/#dry-run),
> tính năng này cần được bật trên `kube-apiserver`.
>
> Vì `diff` thực hiện một request apply phía server ở chế độ dry-run,
> nó yêu cầu được cấp các quyền `PATCH`, `CREATE` và `UPDATE`.
> Xem [Dry-Run Authorization](https://kubernetes.io/docs/reference/using-api/api-concepts#dry-run-authorization)
> để biết chi tiết.

Tạo object bằng `kubectl apply`:

```shell
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml
```

In cấu hình thực tế bằng `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

Kết quả cho thấy annotation `kubectl.kubernetes.io/last-applied-configuration` đã được ghi
vào cấu hình thực tế, và nó khớp với file cấu hình:

```yaml
kind: Deployment
metadata:
  annotations:
    # ...
    # Đây là biểu diễn json của simple_deployment.yaml
    # Nó được kubectl apply ghi vào khi object được tạo
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.14.2","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
  # ...
spec:
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.14.2
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

## Cách cập nhật object (How to update objects)

Bạn cũng có thể dùng `kubectl apply` để cập nhật tất cả các object được định nghĩa trong
một thư mục, kể cả khi các object đó đã tồn tại. Cách tiếp cận này thực hiện những việc sau:

1. Đặt các field xuất hiện trong file cấu hình vào cấu hình thực tế.
2. Xóa khỏi cấu hình thực tế các field đã bị loại bỏ khỏi file cấu hình.

```shell
kubectl diff -f <directory>
kubectl apply -f <directory>
```

> **Ghi chú:** Thêm cờ `-R` để xử lý đệ quy các thư mục con.

Đây là một file cấu hình ví dụ:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
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

Tạo object bằng `kubectl apply`:

```shell
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml
```

> **Ghi chú:** Để minh họa, lệnh trên tham chiếu tới một file cấu hình đơn lẻ
> thay vì một thư mục.

In cấu hình thực tế bằng `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

Kết quả cho thấy annotation `kubectl.kubernetes.io/last-applied-configuration` đã được ghi
vào cấu hình thực tế, và nó khớp với file cấu hình:

```yaml
kind: Deployment
metadata:
  annotations:
    # ...
    # Đây là biểu diễn json của simple_deployment.yaml
    # Nó được kubectl apply ghi vào khi object được tạo
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.14.2","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
  # ...
spec:
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.14.2
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

Cập nhật trực tiếp field `replicas` trong cấu hình thực tế bằng `kubectl scale`.
Thao tác này không dùng `kubectl apply`:

```shell
kubectl scale deployment/nginx-deployment --replicas=2
```

In cấu hình thực tế bằng `kubectl get`:

```shell
kubectl get deployment nginx-deployment -o yaml
```

Kết quả cho thấy field `replicas` đã được đặt thành 2, và annotation
`last-applied-configuration` không chứa field `replicas`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # ...
    # lưu ý rằng annotation không chứa replicas
    # vì field này không được cập nhật thông qua apply
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.14.2","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
  # ...
spec:
  replicas: 2 # do scale ghi vào
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.14.2
        # ...
        name: nginx
        ports:
        - containerPort: 80
      # ...
```

Cập nhật file cấu hình `simple_deployment.yaml` để đổi image từ
`nginx:1.14.2` thành `nginx:1.16.1`, và xóa field `minReadySeconds`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
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
        image: nginx:1.16.1 # cập nhật image
        ports:
        - containerPort: 80
```

Apply các thay đổi đã thực hiện trên file cấu hình:

```shell
kubectl diff -f https://k8s.io/examples/application/update_deployment.yaml
kubectl apply -f https://k8s.io/examples/application/update_deployment.yaml
```

In cấu hình thực tế bằng `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/update_deployment.yaml -o yaml
```

Kết quả cho thấy các thay đổi sau trong cấu hình thực tế:

* Field `replicas` giữ nguyên giá trị 2 do `kubectl scale` đặt.
  Điều này khả thi vì field đó không có mặt trong file cấu hình.
* Field `image` đã được cập nhật từ `nginx:1.14.2` thành `nginx:1.16.1`.
* Annotation `last-applied-configuration` đã được cập nhật với image mới.
* Field `minReadySeconds` đã bị xóa.
* Annotation `last-applied-configuration` không còn chứa field `minReadySeconds`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # ...
    # Annotation chứa image đã cập nhật thành nginx 1.16.1,
    # nhưng không chứa replicas đã cập nhật thành 2
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.16.1","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
    # ...
spec:
  replicas: 2 # Do `kubectl scale` đặt. Bị `kubectl apply` bỏ qua.
  # minReadySeconds đã bị `kubectl apply` xóa
  # ...
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16.1 # Do `kubectl apply` đặt
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

> **Cảnh báo:** Việc trộn lẫn `kubectl apply` với các câu lệnh cấu hình object kiểu mệnh
> lệnh `create` và `replace` không được hỗ trợ. Lý do là `create` và `replace` không giữ lại
> annotation `kubectl.kubernetes.io/last-applied-configuration` mà `kubectl apply` dùng để
> tính toán các cập nhật.

## Cách xóa object (How to delete objects)

Có hai cách tiếp cận để xóa các object được quản lý bởi `kubectl apply`.

### Khuyến nghị: `kubectl delete -f <filename>` (Recommended)

Xóa object thủ công bằng câu lệnh mệnh lệnh là cách tiếp cận được khuyến nghị, vì nó tường
minh hơn về việc cái gì đang bị xóa, và ít khả năng dẫn tới việc người dùng xóa nhầm thứ gì
đó ngoài ý muốn:

```shell
kubectl delete -f <filename>
```

### Cách thay thế: `kubectl apply -f <directory> --prune` (Alternative)

Thay cho `kubectl delete`, bạn có thể dùng `kubectl apply` để xác định các object cần xóa
sau khi manifest của chúng đã bị loại bỏ khỏi một thư mục trên hệ thống file cục bộ.

Trong Kubernetes v1.36, có hai chế độ prune (dọn dẹp) khả dụng trong kubectl apply:

- Prune dựa trên allowlist: Chế độ này tồn tại từ kubectl v1.5 nhưng vẫn ở trạng thái alpha
  do các vấn đề về tính khả dụng, tính đúng đắn và hiệu năng trong thiết kế của nó. Chế độ
  dựa trên ApplySet được thiết kế để thay thế nó.
- Prune dựa trên ApplySet: Một _apply set_ là một object phía server (mặc định là một
  Secret) mà kubectl có thể dùng để theo dõi chính xác và hiệu quả các thành viên của tập
  hợp qua các lần thao tác **apply**. Chế độ này được giới thiệu ở trạng thái alpha trong
  kubectl v1.27 để thay thế cho prune dựa trên allowlist.

#### Allow list

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.5 [alpha]`

> **Cảnh báo:** Hãy cẩn trọng khi dùng `--prune` với `kubectl apply` ở chế độ allow list.
> Object nào bị prune phụ thuộc vào giá trị của các cờ `--prune-allowlist`, `--selector` và
> `--namespace`, và dựa vào việc khám phá động (dynamic discovery) các object trong phạm vi.
> Đặc biệt nếu giá trị các cờ thay đổi giữa các lần gọi lệnh, điều này có thể dẫn tới việc
> object bị xóa hoặc bị giữ lại ngoài dự kiến.

Để dùng prune dựa trên allowlist, thêm các cờ sau vào lệnh `kubectl apply` của bạn:

- `--prune`: Xóa các object đã apply trước đây mà không nằm trong tập được truyền cho lần
  gọi lệnh hiện tại.
- `--prune-allowlist`: Danh sách các group-version-kind (GVK) được xét để prune.
  Cờ này là tùy chọn nhưng rất nên dùng, vì giá trị mặc định của nó là một danh sách
  không đầy đủ gồm cả các kiểu thuộc phạm vi namespace lẫn phạm vi cluster, điều này
  có thể dẫn tới kết quả gây bất ngờ.
- `--selector/-l`: Dùng một label selector để giới hạn tập object được chọn để prune.
  Cờ này là tùy chọn nhưng rất nên dùng.
- `--all`: dùng thay cho `--selector/-l` để chọn tường minh tất cả các object đã apply
  trước đây thuộc các kiểu trong allowlist.

Prune dựa trên allowlist truy vấn API server để lấy tất cả các object thuộc các GVK trong
allowlist khớp với các label đã cho (nếu có), rồi cố gắng đối chiếu cấu hình object thực tế
trả về với các file manifest object. Nếu một object khớp với truy vấn, không có manifest
trong thư mục, và có annotation `kubectl.kubernetes.io/last-applied-configuration`,
thì object đó sẽ bị xóa.

```shell
kubectl apply -f <directory> --prune -l <labels> --prune-allowlist=<gvk-list>
```

> **Cảnh báo:** Apply kèm prune chỉ nên chạy trên thư mục gốc chứa các manifest object.
> Chạy trên thư mục con có thể khiến object bị xóa ngoài ý muốn nếu chúng đã được apply
> trước đó, mang các label đã cho (nếu có), và không xuất hiện trong thư mục con đó.

#### Apply set

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.27 [alpha]`

> **Thận trọng:** `kubectl apply --prune --applyset` đang ở trạng thái alpha, và các thay
> đổi không tương thích ngược có thể được đưa vào trong các bản phát hành tiếp theo.

Để dùng prune dựa trên ApplySet, đặt biến môi trường `KUBECTL_APPLYSET=true`,
và thêm các cờ sau vào lệnh `kubectl apply` của bạn:

- `--prune`: Xóa các object đã apply trước đây mà không nằm trong tập được truyền cho lần
  gọi lệnh hiện tại.
- `--applyset`: Tên của một object mà kubectl có thể dùng để theo dõi chính xác và hiệu quả
  các thành viên của tập hợp qua các lần thao tác `apply`.

```shell
KUBECTL_APPLYSET=true kubectl apply -f <directory> --prune --applyset=<name>
```

Theo mặc định, kiểu của object cha (parent) ApplySet được dùng là một Secret. Tuy nhiên,
cũng có thể dùng ConfigMap theo định dạng: `--applyset=configmaps/<name>`.
Khi dùng Secret hoặc ConfigMap, kubectl sẽ tạo object đó nếu nó chưa tồn tại.

Cũng có thể dùng custom resource làm object cha của ApplySet. Để bật khả năng này, gắn label
sau cho Custom Resource Definition (CRD) định nghĩa resource bạn muốn dùng:
`applyset.kubernetes.io/is-parent-type: true`. Sau đó, tạo object bạn muốn dùng làm cha của
ApplySet (kubectl không tự làm việc này cho custom resource). Cuối cùng, tham chiếu tới
object đó trong cờ applyset như sau: `--applyset=<resource>.<group>/<name>`
(ví dụ, `widgets.custom.example.com/widget-name`).

Với prune dựa trên ApplySet, kubectl thêm label `applyset.kubernetes.io/part-of=<parentID>`
vào mỗi object trong tập trước khi chúng được gửi lên server. Vì lý do hiệu năng, nó cũng
thu thập danh sách các kiểu resource và namespace mà tập hợp chứa, rồi thêm chúng dưới dạng
annotation trên object cha thực tế. Cuối cùng, khi kết thúc thao tác apply, nó truy vấn API
server để lấy các object thuộc các kiểu đó trong các namespace đó (hoặc trong phạm vi
cluster, tùy trường hợp) thuộc về tập hợp, như được xác định bởi label
`applyset.kubernetes.io/part-of=<parentID>`.

Lưu ý và hạn chế:

- Mỗi object chỉ có thể là thành viên của tối đa một tập hợp.
- Cờ `--namespace` là bắt buộc khi dùng bất kỳ object cha nào thuộc phạm vi namespace, kể cả
  Secret mặc định. Điều này nghĩa là các ApplySet trải trên nhiều namespace phải dùng một
  custom resource phạm vi cluster làm object cha.
- Để dùng prune dựa trên ApplySet một cách an toàn với nhiều thư mục, hãy dùng một tên
  ApplySet duy nhất cho mỗi thư mục.

## Cách xem một object (How to view an object)

Bạn có thể dùng `kubectl get` với `-o yaml` để xem cấu hình của một object đang chạy:

```shell
kubectl get -f <filename|url> -o yaml
```

## Cách apply tính toán khác biệt và merge các thay đổi (How apply calculates differences and merges changes)

> **Thận trọng:** Một *patch* là một thao tác cập nhật có phạm vi giới hạn ở các field cụ
> thể của một object thay vì toàn bộ object. Điều này cho phép chỉ cập nhật một tập field cụ
> thể trên một object mà không cần đọc object đó trước.

Khi `kubectl apply` cập nhật cấu hình thực tế cho một object, nó thực hiện điều đó bằng cách
gửi một patch request tới API server. Patch này định nghĩa các cập nhật giới hạn ở những
field cụ thể của cấu hình object thực tế. Lệnh `kubectl apply` tính toán patch request này
bằng cách dùng file cấu hình, cấu hình thực tế, và annotation `last-applied-configuration`
được lưu trong cấu hình thực tế.

### Tính toán merge patch (Merge patch calculation)

Lệnh `kubectl apply` ghi nội dung của file cấu hình vào annotation
`kubectl.kubernetes.io/last-applied-configuration`. Annotation này được dùng để nhận diện
các field đã bị loại khỏi file cấu hình và cần được xóa khỏi cấu hình thực tế. Sau đây là
các bước được dùng để tính toán field nào cần xóa hoặc cần đặt:

1. Tính các field cần xóa. Đó là các field có mặt trong `last-applied-configuration` nhưng
   không có trong file cấu hình.
2. Tính các field cần thêm hoặc đặt. Đó là các field có mặt trong file cấu hình mà giá trị
   không khớp với cấu hình thực tế.

Đây là một ví dụ. Giả sử đây là file cấu hình cho một object Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
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
        image: nginx:1.16.1 # cập nhật image
        ports:
        - containerPort: 80
```

Đồng thời, giả sử đây là cấu hình thực tế của cùng object Deployment đó:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # ...
    # lưu ý rằng annotation không chứa replicas
    # vì field này không được cập nhật thông qua apply
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.14.2","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
  # ...
spec:
  replicas: 2 # do scale ghi vào
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.14.2
        # ...
        name: nginx
        ports:
        - containerPort: 80
      # ...
```

Đây là các phép tính merge mà `kubectl apply` sẽ thực hiện:

1. Tính các field cần xóa bằng cách đọc giá trị từ `last-applied-configuration` và so sánh
   với giá trị trong file cấu hình.
   Xóa các field được đặt tường minh thành null trong file cấu hình object cục bộ, bất kể
   chúng có xuất hiện trong `last-applied-configuration` hay không.
   Trong ví dụ này, `minReadySeconds` xuất hiện trong annotation
   `last-applied-configuration` nhưng không xuất hiện trong file cấu hình.
   **Hành động:** Xóa `minReadySeconds` khỏi cấu hình thực tế.
2. Tính các field cần đặt bằng cách đọc giá trị từ file cấu hình và so sánh với giá trị
   trong cấu hình thực tế. Trong ví dụ này, giá trị của `image` trong file cấu hình không
   khớp với giá trị trong cấu hình thực tế. **Hành động:** Đặt giá trị của `image` trong
   cấu hình thực tế.
3. Đặt annotation `last-applied-configuration` cho khớp với nội dung của file cấu hình.
4. Gộp kết quả của các bước 1, 2, 3 thành một patch request duy nhất gửi tới API server.

Đây là cấu hình thực tế — kết quả của phép merge:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # ...
    # Annotation chứa image đã cập nhật thành nginx 1.16.1,
    # nhưng không chứa replicas đã cập nhật thành 2
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.16.1","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
    # ...
spec:
  selector:
    matchLabels:
      # ...
      app: nginx
  replicas: 2 # Do `kubectl scale` đặt. Bị `kubectl apply` bỏ qua.
  # minReadySeconds đã bị `kubectl apply` xóa
  # ...
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16.1 # Do `kubectl apply` đặt
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

### Các loại field khác nhau được merge như thế nào (How different types of fields are merged)

Cách một field cụ thể trong file cấu hình được merge với cấu hình thực tế phụ thuộc vào
loại của field đó. Có một số loại field:

- *primitive*: Field có kiểu string, integer hoặc boolean.
  Ví dụ, `image` và `replicas` là các field primitive. **Hành động:** Thay thế.

- *map*, còn gọi là *object*: Field có kiểu map hoặc một kiểu phức hợp chứa các subfield.
  Ví dụ, `labels`, `annotations`, `spec` và `metadata` đều là map.
  **Hành động:** Merge từng phần tử hoặc từng subfield.

- *list*: Field chứa một danh sách các phần tử có thể là kiểu primitive hoặc map.
  Ví dụ, `containers`, `ports` và `args` là các list. **Hành động:** Tùy trường hợp.

Khi `kubectl apply` cập nhật một field kiểu map hoặc list, thông thường nó không thay thế
toàn bộ field, mà cập nhật từng phần tử con riêng lẻ. Chẳng hạn, khi merge `spec` trên một
Deployment, toàn bộ `spec` không bị thay thế. Thay vào đó, các subfield của `spec`, như
`replicas`, được so sánh và merge.

### Merge thay đổi cho các field kiểu primitive (Merging changes to primitive fields)

Các field primitive được thay thế hoặc bị xóa.

> **Ghi chú:** Dấu `-` nghĩa là "không áp dụng" vì giá trị đó không được dùng đến.

| Field trong file cấu hình object | Field trong cấu hình object thực tế | Field trong last-applied-configuration | Hành động |
|----------------------------------|-------------------------------------|----------------------------------------|-----------|
| Có                               | Có                                  | -                                      | Đặt giá trị thực tế theo giá trị trong file cấu hình. |
| Có                               | Không                               | -                                      | Đặt giá trị thực tế theo cấu hình cục bộ. |
| Không                            | -                                   | Có                                     | Xóa khỏi cấu hình thực tế. |
| Không                            | -                                   | Không                                  | Không làm gì. Giữ giá trị thực tế. |

### Merge thay đổi cho các field kiểu map (Merging changes to map fields)

Các field biểu diễn map được merge bằng cách so sánh từng subfield hoặc từng phần tử của map:

> **Ghi chú:** Dấu `-` nghĩa là "không áp dụng" vì giá trị đó không được dùng đến.

| Key trong file cấu hình object | Key trong cấu hình object thực tế | Field trong last-applied-configuration | Hành động |
|--------------------------------|-----------------------------------|----------------------------------------|-----------|
| Có                             | Có                                | -                                      | So sánh giá trị các subfield. |
| Có                             | Không                             | -                                      | Đặt giá trị thực tế theo cấu hình cục bộ. |
| Không                          | -                                 | Có                                     | Xóa khỏi cấu hình thực tế. |
| Không                          | -                                 | Không                                  | Không làm gì. Giữ giá trị thực tế. |

### Merge thay đổi cho các field kiểu list (Merging changes for fields of type list)

Việc merge thay đổi cho một list dùng một trong ba chiến lược:

* Thay thế toàn bộ list nếu tất cả phần tử của nó là primitive.
* Merge từng phần tử riêng lẻ trong một list gồm các phần tử phức hợp.
* Merge một list gồm các phần tử primitive.

Việc chọn chiến lược được quyết định theo từng field.

#### Thay thế list nếu tất cả phần tử là primitive (Replace the list if all its elements are primitives)

Đối xử với list giống như một field primitive. Thay thế hoặc xóa toàn bộ list.
Cách này bảo toàn thứ tự.

**Ví dụ:** Dùng `kubectl apply` để cập nhật field `args` của một Container trong một Pod.
Thao tác này đặt giá trị của `args` trong cấu hình thực tế thành giá trị trong file cấu
hình. Mọi phần tử `args` đã được thêm vào cấu hình thực tế trước đó đều bị mất. Thứ tự các
phần tử `args` định nghĩa trong file cấu hình được giữ nguyên trong cấu hình thực tế.

```yaml
# giá trị trong last-applied-configuration
    args: ["a", "b"]

# giá trị trong file cấu hình
    args: ["a", "c"]

# cấu hình thực tế
    args: ["a", "b", "d"]

# kết quả sau khi merge
    args: ["a", "c"]
```

**Giải thích:** Phép merge đã dùng giá trị trong file cấu hình làm giá trị mới của list.

#### Merge từng phần tử của một list gồm các phần tử phức hợp (Merge individual elements of a list of complex elements)

Đối xử với list như một map, và coi một field cụ thể của mỗi phần tử như một key.
Thêm, xóa hoặc cập nhật từng phần tử riêng lẻ. Cách này không bảo toàn thứ tự.

Chiến lược merge này dùng một tag đặc biệt trên mỗi field gọi là `patchMergeKey`.
`patchMergeKey` được định nghĩa cho từng field trong mã nguồn Kubernetes:
[types.go](https://github.com/kubernetes/api/blob/d04500c8c3dda9c980b668c57abc2ca61efcf5c4/core/v1/types.go#L2747)
Khi merge một list các map, field được chỉ định làm `patchMergeKey` của một phần tử
được dùng như map key cho phần tử đó.

**Ví dụ:** Dùng `kubectl apply` để cập nhật field `containers` của một PodSpec.
Thao tác này merge list như thể nó là một map trong đó mỗi phần tử được định danh
bằng key là `name`.

```yaml
# giá trị trong last-applied-configuration
    containers:
    - name: nginx
      image: nginx:1.16
    - name: nginx-helper-a # key: nginx-helper-a; sẽ bị xóa trong kết quả
      image: helper:1.3
    - name: nginx-helper-b # key: nginx-helper-b; sẽ được giữ lại
      image: helper:1.3

# giá trị trong file cấu hình
    containers:
    - name: nginx
      image: nginx:1.16
    - name: nginx-helper-b
      image: helper:1.3
    - name: nginx-helper-c # key: nginx-helper-c; sẽ được thêm vào kết quả
      image: helper:1.3

# cấu hình thực tế
    containers:
    - name: nginx
      image: nginx:1.16
    - name: nginx-helper-a
      image: helper:1.3
    - name: nginx-helper-b
      image: helper:1.3
      args: ["run"] # Field sẽ được giữ lại
    - name: nginx-helper-d # key: nginx-helper-d; sẽ được giữ lại
      image: helper:1.3

# kết quả sau khi merge
    containers:
    - name: nginx
      image: nginx:1.16
      # Phần tử nginx-helper-a đã bị xóa
    - name: nginx-helper-b
      image: helper:1.3
      args: ["run"] # Field được giữ lại
    - name: nginx-helper-c # Phần tử được thêm vào
      image: helper:1.3
    - name: nginx-helper-d # Phần tử được bỏ qua (không đụng tới)
      image: helper:1.3
```

**Giải thích:**

- Container tên "nginx-helper-a" bị xóa vì không có container nào tên "nginx-helper-a"
  xuất hiện trong file cấu hình.
- Container tên "nginx-helper-b" giữ lại các thay đổi đối với `args` trong cấu hình thực
  tế. `kubectl apply` có thể nhận ra rằng "nginx-helper-b" trong cấu hình thực tế chính là
  "nginx-helper-b" trong file cấu hình, mặc dù các field của chúng có giá trị khác nhau
  (không có `args` trong file cấu hình). Đó là vì giá trị của field `patchMergeKey` (name)
  giống hệt nhau ở cả hai.
- Container tên "nginx-helper-c" được thêm vào vì không có container nào mang tên đó trong
  cấu hình thực tế, nhưng lại có một container mang tên đó trong file cấu hình.
- Container tên "nginx-helper-d" được giữ lại vì không có phần tử nào mang tên đó xuất hiện
  trong last-applied-configuration.

#### Merge một list gồm các phần tử primitive (Merge a list of primitive elements)

Kể từ Kubernetes 1.5, việc merge list gồm các phần tử primitive không được hỗ trợ.

> **Ghi chú:** Chiến lược nào trong các chiến lược trên được chọn cho một field cụ thể được
> điều khiển bởi tag `patchStrategy` trong
> [types.go](https://github.com/kubernetes/api/blob/d04500c8c3dda9c980b668c57abc2ca61efcf5c4/core/v1/types.go#L2748).
> Nếu không có `patchStrategy` nào được chỉ định cho một field kiểu list, thì list đó
> sẽ bị thay thế.

## Giá trị mặc định của field (Default field values)

API server đặt một số field nhất định về giá trị mặc định trong cấu hình thực tế nếu chúng
không được chỉ định khi object được tạo.

Đây là một file cấu hình cho một Deployment. File này không chỉ định `strategy`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
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

Tạo object bằng `kubectl apply`:

```shell
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml
```

In cấu hình thực tế bằng `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

Kết quả cho thấy API server đã đặt nhiều field về giá trị mặc định trong cấu hình thực tế.
Các field này không được chỉ định trong file cấu hình.

```yaml
apiVersion: apps/v1
kind: Deployment
# ...
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  replicas: 1 # do apiserver đặt mặc định
  strategy:
    rollingUpdate: # do apiserver đặt mặc định - suy ra từ strategy.type
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate # do apiserver đặt mặc định
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.14.2
        imagePullPolicy: IfNotPresent # do apiserver đặt mặc định
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP # do apiserver đặt mặc định
        resources: {} # do apiserver đặt mặc định
        terminationMessagePath: /dev/termination-log # do apiserver đặt mặc định
      dnsPolicy: ClusterFirst # do apiserver đặt mặc định
      restartPolicy: Always # do apiserver đặt mặc định
      securityContext: {} # do apiserver đặt mặc định
      terminationGracePeriodSeconds: 30 # do apiserver đặt mặc định
# ...
```

Trong một patch request, các field đã được đặt mặc định sẽ không được tính mặc định lại
(re-default) trừ khi chúng bị xóa tường minh như một phần của patch request. Điều này có thể
gây ra hành vi bất ngờ đối với các field được đặt mặc định dựa trên giá trị của các field
khác. Khi các field khác đó sau này thay đổi, các giá trị đã được suy ra mặc định từ chúng
sẽ không được cập nhật trừ khi chúng bị xóa tường minh.

Vì lý do đó, khuyến nghị rằng một số field nhất định do server đặt mặc định nên được định
nghĩa tường minh trong file cấu hình, ngay cả khi giá trị mong muốn trùng với giá trị mặc
định của server. Điều này giúp dễ dàng nhận ra các giá trị xung đột mà server sẽ không tính
mặc định lại.

**Ví dụ:**

```yaml
# last-applied-configuration
spec:
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

# file cấu hình
spec:
  strategy:
    type: Recreate # giá trị được cập nhật
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

# cấu hình thực tế
spec:
  strategy:
    type: RollingUpdate # giá trị được đặt mặc định
    rollingUpdate: # giá trị mặc định suy ra từ type
      maxSurge : 1
      maxUnavailable: 1
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

# kết quả sau khi merge - LỖI!
spec:
  strategy:
    type: Recreate # giá trị được cập nhật: không tương thích với rollingUpdate
    rollingUpdate: # giá trị được đặt mặc định: không tương thích với "type: Recreate"
      maxSurge : 1
      maxUnavailable: 1
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

**Giải thích:**

1. Người dùng tạo một Deployment mà không định nghĩa `strategy.type`.
2. Server đặt mặc định `strategy.type` thành `RollingUpdate` và đặt mặc định các giá trị
   `strategy.rollingUpdate`.
3. Người dùng đổi `strategy.type` thành `Recreate`. Các giá trị `strategy.rollingUpdate`
   vẫn giữ nguyên giá trị mặc định của chúng, dù server kỳ vọng chúng phải bị xóa.
   Nếu các giá trị `strategy.rollingUpdate` được định nghĩa ngay từ đầu trong file cấu hình,
   thì việc chúng cần bị xóa sẽ rõ ràng hơn.
4. Apply thất bại vì `strategy.rollingUpdate` không bị xóa. Field `strategy.rollingupdate`
   không thể được định nghĩa khi `strategy.type` là `Recreate`.

Khuyến nghị: Các field sau nên được định nghĩa tường minh trong file cấu hình object:

- Selector và label của PodTemplate trên các workload, như Deployment, StatefulSet, Job,
  DaemonSet, ReplicaSet và ReplicationController
- Chiến lược rollout của Deployment

### Cách xóa các field do server đặt mặc định hoặc do người ghi khác đặt (How to clear server-defaulted fields or fields set by other writers)

Các field không xuất hiện trong file cấu hình có thể bị xóa bằng cách đặt giá trị của chúng
thành `null` rồi apply file cấu hình. Với các field do server đặt mặc định, việc này kích
hoạt tính mặc định lại (re-defaulting) các giá trị đó.

## Cách chuyển quyền sở hữu một field giữa file cấu hình và người ghi mệnh lệnh trực tiếp (How to change ownership of a field between the configuration file and direct imperative writers)

Đây là các phương pháp duy nhất bạn nên dùng để thay đổi một field riêng lẻ của object:

- Dùng `kubectl apply`.
- Ghi trực tiếp vào cấu hình thực tế mà không sửa file cấu hình:
  ví dụ, dùng `kubectl scale`.

### Chuyển chủ sở hữu từ người ghi mệnh lệnh trực tiếp sang file cấu hình (Changing the owner from a direct imperative writer to a configuration file)

Thêm field đó vào file cấu hình. Với field này, ngừng mọi cập nhật trực tiếp vào cấu hình
thực tế mà không đi qua `kubectl apply`.

### Chuyển chủ sở hữu từ file cấu hình sang người ghi mệnh lệnh trực tiếp (Changing the owner from a configuration file to a direct imperative writer)

Kể từ Kubernetes 1.5, việc chuyển quyền sở hữu một field từ file cấu hình sang người ghi
mệnh lệnh đòi hỏi các bước thủ công:

- Xóa field đó khỏi file cấu hình.
- Xóa field đó khỏi annotation `kubectl.kubernetes.io/last-applied-configuration` trên
  object đang chạy.

## Thay đổi phương pháp quản lý (Changing management methods)

Các object Kubernetes chỉ nên được quản lý bằng một phương pháp tại một thời điểm.
Việc chuyển từ phương pháp này sang phương pháp khác là khả thi, nhưng là một quy trình
thủ công.

> **Ghi chú:** Dùng lệnh xóa mệnh lệnh (imperative deletion) cùng với quản lý khai báo
> là hoàn toàn ổn.

### Di chuyển từ quản lý bằng câu lệnh mệnh lệnh sang cấu hình object khai báo (Migrating from imperative command management to declarative object configuration)

Việc di chuyển từ quản lý bằng câu lệnh mệnh lệnh sang cấu hình object khai báo bao gồm
một số bước thủ công:

1. Xuất object đang chạy ra một file cấu hình cục bộ:

   ```shell
   kubectl get <kind>/<name> -o yaml > <kind>_<name>.yaml
   ```

1. Xóa thủ công field `status` khỏi file cấu hình.

   > **Ghi chú:** Bước này là tùy chọn, vì `kubectl apply` không cập nhật field status
   > ngay cả khi nó có mặt trong file cấu hình.

1. Đặt annotation `kubectl.kubernetes.io/last-applied-configuration` trên object:

   ```shell
   kubectl replace --save-config -f <kind>_<name>.yaml
   ```

1. Thay đổi quy trình làm việc để chỉ dùng duy nhất `kubectl apply` khi quản lý object đó.

### Di chuyển từ cấu hình object mệnh lệnh sang cấu hình object khai báo (Migrating from imperative object configuration to declarative object configuration)

1. Đặt annotation `kubectl.kubernetes.io/last-applied-configuration` trên object:

   ```shell
   kubectl replace --save-config -f <kind>_<name>.yaml
   ```

1. Thay đổi quy trình làm việc để chỉ dùng duy nhất `kubectl apply` khi quản lý object đó.

## Định nghĩa selector của controller và label của PodTemplate (Defining controller selectors and PodTemplate labels)

> **Cảnh báo:** Việc cập nhật selector trên các controller là điều rất không nên làm.

Cách tiếp cận được khuyến nghị là định nghĩa một label PodTemplate duy nhất, bất biến
(immutable), chỉ được dùng cho selector của controller và không mang ý nghĩa ngữ nghĩa nào
khác.

**Ví dụ:**

```yaml
selector:
  matchLabels:
      controller-selector: "apps/v1/deployment/nginx"
template:
  metadata:
    labels:
      controller-selector: "apps/v1/deployment/nginx"
```

## Tiếp theo (What's next)

* [Quản lý object Kubernetes bằng câu lệnh mệnh lệnh](320-imperative-command-vi.md)
* [Quản lý object Kubernetes kiểu mệnh lệnh bằng file cấu hình](321-imperative-config-vi.md)
* [Tài liệu tham khảo lệnh Kubectl](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/)
* [Tài liệu tham khảo API Kubernetes](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/)

