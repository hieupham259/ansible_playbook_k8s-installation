# Quản lý object Kubernetes theo kiểu khai báo bằng Kustomize (Declarative Management of Kubernetes Objects Using Kustomize)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/>
>
> Kustomize là một công cụ độc lập để tùy biến các object Kubernetes thông qua một file
> kustomization; từ kubectl 1.14, bạn có thể dùng trực tiếp `kubectl apply -k` để quản lý
> object bằng file kustomization.


[Kustomize](https://github.com/kubernetes-sigs/kustomize) là một công cụ độc lập để tùy biến
các object Kubernetes thông qua một
[file kustomization](https://kubectl.docs.kubernetes.io/references/kustomize/glossary/#kustomization).

Từ phiên bản 1.14, kubectl cũng hỗ trợ quản lý object Kubernetes bằng file kustomization.
Để xem các tài nguyên (resource) trong một thư mục chứa file kustomization, hãy chạy lệnh sau:

```shell
kubectl kustomize <kustomization_directory>
```

Để apply các tài nguyên đó, hãy chạy `kubectl apply` với cờ `--kustomize` hoặc `-k`:

```shell
kubectl apply -k <kustomization_directory>
```

## Trước khi bạn bắt đầu (Before you begin)

Cài đặt [`kubectl`](185-tools-vi.md).

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình
để giao tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít
nhất hai node không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có
thể tạo một cluster bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy chạy `kubectl version`.

## Tổng quan về Kustomize (Overview of Kustomize)

Kustomize là một công cụ để tùy biến cấu hình Kubernetes. Nó có các tính năng sau để quản lý
file cấu hình ứng dụng:

* sinh tài nguyên từ các nguồn khác
* thiết lập các trường xuyên suốt (cross-cutting field) cho tài nguyên
* kết hợp (compose) và tùy biến các tập hợp tài nguyên

### Sinh tài nguyên (Generating Resources)

ConfigMap và Secret lưu dữ liệu cấu hình hoặc dữ liệu nhạy cảm được các object Kubernetes
khác, chẳng hạn Pod, sử dụng. Nguồn chân lý (source of truth) của ConfigMap hoặc Secret
thường nằm ngoài cluster, ví dụ một file `.properties` hoặc một file khóa SSH.
Kustomize có `secretGenerator` và `configMapGenerator`, dùng để sinh Secret và ConfigMap
từ file hoặc từ các giá trị literal.

#### configMapGenerator

Để sinh một ConfigMap từ file, hãy thêm một mục vào danh sách `files` trong `configMapGenerator`.
Đây là ví dụ sinh một ConfigMap với một mục dữ liệu lấy từ file `.properties`:

```shell
# Tạo file application.properties
cat <<EOF >application.properties
FOO=Bar
EOF

cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-1
  files:
  - application.properties
EOF
```

Có thể kiểm tra ConfigMap được sinh ra bằng lệnh sau:

```shell
kubectl kustomize ./
```

ConfigMap được sinh ra là:

```yaml
apiVersion: v1
data:
  application.properties: |
    FOO=Bar
kind: ConfigMap
metadata:
  name: example-configmap-1-8mbdf7882g
```

Để sinh một ConfigMap từ một env file, hãy thêm một mục vào danh sách `envs` trong `configMapGenerator`.
Đây là ví dụ sinh một ConfigMap với một mục dữ liệu lấy từ file `.env`:

```shell
# Tạo file .env
cat <<EOF >.env
FOO=Bar
EOF

cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-1
  envs:
  - .env
EOF
```

Có thể kiểm tra ConfigMap được sinh ra bằng lệnh sau:

```shell
kubectl kustomize ./
```

ConfigMap được sinh ra là:

```yaml
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  name: example-configmap-1-42cfbf598f
```

> **Ghi chú:**
> Mỗi biến trong file `.env` trở thành một key riêng biệt trong ConfigMap mà bạn sinh ra.
> Điều này khác với ví dụ trước, vốn nhúng cả một file tên `application.properties`
> (cùng toàn bộ các mục trong nó) làm giá trị cho một key duy nhất.

ConfigMap cũng có thể được sinh từ các cặp key-value literal. Để sinh một ConfigMap từ một
cặp key-value literal, hãy thêm một mục vào danh sách `literals` trong configMapGenerator.
Đây là ví dụ sinh một ConfigMap với một mục dữ liệu lấy từ một cặp key-value:

```shell
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-2
  literals:
  - FOO=Bar
EOF
```

Có thể kiểm tra ConfigMap được sinh ra bằng lệnh sau:

```shell
kubectl kustomize ./
```

ConfigMap được sinh ra là:

```yaml
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  name: example-configmap-2-g2hdhfc6tk
```

Để dùng một ConfigMap được sinh ra trong một Deployment, hãy tham chiếu nó bằng tên của
configMapGenerator. Kustomize sẽ tự động thay tên này bằng tên đã được sinh ra.

Đây là ví dụ một deployment sử dụng ConfigMap được sinh ra:

```yaml
# Tạo file application.properties
cat <<EOF >application.properties
FOO=Bar
EOF

cat <<EOF >deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: example-configmap-1
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
configMapGenerator:
- name: example-configmap-1
  files:
  - application.properties
EOF
```

Sinh ConfigMap và Deployment:

```shell
kubectl kustomize ./
```

Deployment được sinh ra sẽ tham chiếu tới ConfigMap được sinh ra theo tên:

```yaml
apiVersion: v1
data:
  application.properties: |
    FOO=Bar
kind: ConfigMap
metadata:
  name: example-configmap-1-g4hk9g2ff8
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: my-app
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: my-app
        name: app
        volumeMounts:
        - mountPath: /config
          name: config
      volumes:
      - configMap:
          name: example-configmap-1-g4hk9g2ff8
        name: config
```

#### secretGenerator

Bạn có thể sinh Secret từ file hoặc từ các cặp key-value literal.
Để sinh một Secret từ file, hãy thêm một mục vào danh sách `files` trong `secretGenerator`.
Đây là ví dụ sinh một Secret với một mục dữ liệu lấy từ một file:

```shell
# Tạo file password.txt
cat <<EOF >./password.txt
username=admin
password=secret
EOF

cat <<EOF >./kustomization.yaml
secretGenerator:
- name: example-secret-1
  files:
  - password.txt
EOF
```

Secret được sinh ra như sau:

```yaml
apiVersion: v1
data:
  password.txt: dXNlcm5hbWU9YWRtaW4KcGFzc3dvcmQ9c2VjcmV0Cg==
kind: Secret
metadata:
  name: example-secret-1-t2kt65hgtb
type: Opaque
```

Để sinh một Secret từ một cặp key-value literal, hãy thêm một mục vào danh sách `literals`
trong `secretGenerator`. Đây là ví dụ sinh một Secret với một mục dữ liệu lấy từ một cặp
key-value:

```shell
cat <<EOF >./kustomization.yaml
secretGenerator:
- name: example-secret-2
  literals:
  - username=admin
  - password=secret
EOF
```

Secret được sinh ra như sau:

```yaml
apiVersion: v1
data:
  password: c2VjcmV0
  username: YWRtaW4=
kind: Secret
metadata:
  name: example-secret-2-t52t6g96d8
type: Opaque
```

Giống ConfigMap, Secret được sinh ra có thể được dùng trong Deployment bằng cách tham chiếu
tên của secretGenerator:

```shell
# Tạo file password.txt
cat <<EOF >./password.txt
username=admin
password=secret
EOF

cat <<EOF >deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app
        volumeMounts:
        - name: password
          mountPath: /secrets
      volumes:
      - name: password
        secret:
          secretName: example-secret-1
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
secretGenerator:
- name: example-secret-1
  files:
  - password.txt
EOF
```

#### generatorOptions

Các ConfigMap và Secret được sinh ra có một hậu tố hash nội dung (content hash suffix) gắn
vào tên. Điều này bảo đảm rằng một ConfigMap hoặc Secret mới sẽ được sinh ra khi nội dung
thay đổi. Để tắt hành vi thêm hậu tố, bạn có thể dùng `generatorOptions`. Bên cạnh đó, cũng
có thể chỉ định các tùy chọn xuyên suốt cho các ConfigMap và Secret được sinh ra.

```shell
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-3
  literals:
  - FOO=Bar
generatorOptions:
  disableNameSuffixHash: true
  labels:
    type: generated
  annotations:
    note: generated
EOF
```

Chạy `kubectl kustomize ./` để xem ConfigMap được sinh ra:

```yaml
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  annotations:
    note: generated
  labels:
    type: generated
  name: example-configmap-3
```

### Thiết lập các trường xuyên suốt (Setting cross-cutting fields)

Việc thiết lập các trường xuyên suốt cho tất cả tài nguyên Kubernetes trong một dự án là
khá phổ biến. Một số trường hợp sử dụng của việc thiết lập trường xuyên suốt:

* đặt cùng một namespace cho tất cả tài nguyên
* thêm cùng một tiền tố (prefix) hoặc hậu tố (suffix) vào tên
* thêm cùng một tập label
* thêm cùng một tập annotation

Đây là một ví dụ:

```shell
# Tạo file deployment.yaml
cat <<EOF >./deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        image: nginx
EOF

cat <<EOF >./kustomization.yaml
namespace: my-namespace
namePrefix: dev-
nameSuffix: "-001"
labels:
  - pairs:
      app: bingo
    includeSelectors: true 
commonAnnotations:
  oncallPager: 800-555-1212
resources:
- deployment.yaml
EOF
```

Chạy `kubectl kustomize ./` để thấy tất cả các trường đó đều đã được thiết lập trong tài
nguyên Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    oncallPager: 800-555-1212
  labels:
    app: bingo
  name: dev-nginx-deployment-001
  namespace: my-namespace
spec:
  selector:
    matchLabels:
      app: bingo
  template:
    metadata:
      annotations:
        oncallPager: 800-555-1212
      labels:
        app: bingo
    spec:
      containers:
      - image: nginx
        name: nginx
```

### Kết hợp và tùy biến tài nguyên (Composing and Customizing Resources)

Việc kết hợp một tập tài nguyên trong một dự án và quản lý chúng trong cùng một file hoặc
thư mục là điều phổ biến. Kustomize cho phép kết hợp tài nguyên từ các file khác nhau và áp
dụng patch hoặc các tùy biến khác lên chúng.

#### Kết hợp (Composing)

Kustomize hỗ trợ kết hợp các tài nguyên khác nhau. Trường `resources` trong file
`kustomization.yaml` định nghĩa danh sách các tài nguyên đưa vào một cấu hình. Hãy đặt đường
dẫn tới file cấu hình của một tài nguyên vào danh sách `resources`.
Đây là ví dụ một ứng dụng NGINX gồm một Deployment và một Service:

```shell
# Tạo file deployment.yaml
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Tạo file service.yaml
cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF

# Tạo file kustomization.yaml kết hợp cả hai
cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```

Các tài nguyên từ `kubectl kustomize ./` chứa cả object Deployment lẫn object Service.

#### Tùy biến (Customizing)

Có thể dùng patch để áp dụng các tùy biến khác nhau lên tài nguyên. Kustomize hỗ trợ các cơ
chế patch khác nhau thông qua `StrategicMerge` và `Json6902` bằng trường `patches`. `patches`
có thể là một file hoặc một chuỗi inline, nhắm tới một hoặc nhiều tài nguyên.

Trường `patches` chứa một danh sách các patch được áp dụng theo thứ tự khai báo. Đích của
patch (patch target) chọn tài nguyên theo `group`, `version`, `kind`, `name`, `namespace`,
`labelSelector` và `annotationSelector`.

Nên dùng các patch nhỏ, mỗi patch làm một việc. Ví dụ, tạo một patch để tăng số replica của
deployment và một patch khác để đặt giới hạn bộ nhớ (memory limit). Tài nguyên đích được so
khớp bằng các trường `group`, `version`, `kind` và `name` lấy từ file patch.

```shell
# Tạo file deployment.yaml
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Tạo patch increase_replicas.yaml
cat <<EOF > increase_replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
EOF

# Tạo một patch khác set_memory.yaml
cat <<EOF > set_memory.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  template:
    spec:
      containers:
      - name: my-nginx
        resources:
          limits:
            memory: 512Mi
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
patches:
  - path: increase_replicas.yaml
  - path: set_memory.yaml
EOF
```

Chạy `kubectl kustomize ./` để xem Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: 512Mi
```

Không phải tài nguyên hay trường nào cũng hỗ trợ patch `strategicMerge`. Để hỗ trợ sửa đổi
các trường bất kỳ trên các tài nguyên bất kỳ, Kustomize cho phép áp dụng
[JSON patch](https://tools.ietf.org/html/rfc6902) thông qua `Json6902`.
Để tìm đúng tài nguyên cho một patch `Json6902`, bắt buộc phải chỉ định trường `target`
trong `kustomization.yaml`.

Ví dụ, việc tăng số replica của một object Deployment cũng có thể thực hiện bằng patch
`Json6902`. Tài nguyên đích được so khớp bằng `group`, `version`, `kind` và `name` lấy từ
trường `target`.

```shell
# Tạo file deployment.yaml
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Tạo một json patch
cat <<EOF > patch.yaml
- op: replace
  path: /spec/replicas
  value: 3
EOF

# Tạo file kustomization.yaml
cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml

patches:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: my-nginx
  path: patch.yaml
EOF
```

Chạy `kubectl kustomize ./` để thấy trường `replicas` đã được cập nhật:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80
```

Ngoài patch, Kustomize còn cho phép tùy biến image của container hoặc bơm (inject) giá trị
trường từ các object khác vào container mà không cần tạo patch. Ví dụ, bạn có thể thay đổi
image dùng bên trong container bằng cách chỉ định image mới trong trường `images` của
`kustomization.yaml`.

```shell
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
images:
- name: nginx
  newName: my.image.registry/nginx
  newTag: "1.4.0"
EOF
```

Chạy `kubectl kustomize ./` để thấy image đang dùng đã được cập nhật:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: my.image.registry/nginx:1.4.0
        name: my-nginx
        ports:
        - containerPort: 80
```

Đôi khi ứng dụng chạy trong một Pod có thể cần dùng giá trị cấu hình từ các object khác.
Ví dụ, một Pod của object Deployment cần đọc tên Service tương ứng từ biến môi trường (Env)
hoặc như một tham số dòng lệnh. Vì tên Service có thể thay đổi khi `namePrefix` hoặc
`nameSuffix` được thêm vào file `kustomization.yaml`, việc hard-code tên Service trong tham
số dòng lệnh là không nên. Cho nhu cầu này, Kustomize có thể bơm tên Service vào container
thông qua `replacements`.

```shell
# Tạo file deployment.yaml (đặt dấu nháy quanh ký hiệu phân tách here doc)
cat <<'EOF' > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        command: ["start", "--host", "MY_SERVICE_NAME_PLACEHOLDER"]
EOF

# Tạo file service.yaml
cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF

cat <<EOF >./kustomization.yaml
namePrefix: dev-
nameSuffix: "-001"

resources:
- deployment.yaml
- service.yaml

replacements:
- source:
    kind: Service
    name: my-nginx
    fieldPath: metadata.name
  targets:
  - select:
      kind: Deployment
      name: my-nginx
    fieldPaths:
    - spec.template.spec.containers.0.command.2
EOF
```

Chạy `kubectl kustomize ./` để thấy tên Service được bơm vào container là `dev-my-nginx-001`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-my-nginx-001
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - command:
        - start
        - --host
        - dev-my-nginx-001
        image: nginx
        name: my-nginx
```

## Base và Overlay (Bases and Overlays)

Kustomize có khái niệm **base** và **overlay**. Một **base** là một thư mục có
`kustomization.yaml`, chứa một tập tài nguyên cùng các tùy biến đi kèm. Base có thể là thư
mục cục bộ hoặc thư mục từ một repo từ xa, miễn là bên trong có file `kustomization.yaml`.
Một **overlay** là một thư mục có `kustomization.yaml` tham chiếu tới các thư mục
kustomization khác như là `bases` của nó. Một **base** không biết gì về overlay và có thể
được dùng trong nhiều overlay.

File `kustomization.yaml` trong thư mục **overlay** có thể tham chiếu nhiều `bases`, gộp tất
cả tài nguyên định nghĩa trong các base đó thành một cấu hình thống nhất. Ngoài ra, nó có
thể áp dụng các tùy biến lên trên các tài nguyên này để đáp ứng những yêu cầu riêng.

Đây là ví dụ một base:

```shell
# Tạo thư mục chứa base
mkdir base
# Tạo file base/deployment.yaml
cat <<EOF > base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
EOF

# Tạo file base/service.yaml
cat <<EOF > base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF
# Tạo file base/kustomization.yaml
cat <<EOF > base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```

Base này có thể được dùng trong nhiều overlay. Bạn có thể thêm `namePrefix` khác nhau hoặc
các trường xuyên suốt khác trong từng overlay. Đây là hai overlay cùng dùng một base.

```shell
mkdir dev
cat <<EOF > dev/kustomization.yaml
resources:
- ../base
namePrefix: dev-
EOF

mkdir prod
cat <<EOF > prod/kustomization.yaml
resources:
- ../base
namePrefix: prod-
EOF
```

## Cách apply/xem/xóa object bằng Kustomize (How to apply/view/delete objects using Kustomize)

Dùng `--kustomize` hoặc `-k` trong các lệnh `kubectl` để nhận diện các tài nguyên được quản
lý bởi `kustomization.yaml`. Lưu ý rằng `-k` phải trỏ tới một thư mục kustomization, ví dụ

```shell
kubectl apply -k <kustomization directory>/
```

Với `kustomization.yaml` sau,

```shell
# Tạo file deployment.yaml
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Tạo file kustomization.yaml
cat <<EOF >./kustomization.yaml
namePrefix: dev-
labels:
  - pairs:
      app: my-nginx
    includeSelectors: true 
resources:
- deployment.yaml
EOF
```

Chạy lệnh sau để apply object Deployment `dev-my-nginx`:

```shell
> kubectl apply -k ./
deployment.apps/dev-my-nginx created
```

Chạy một trong các lệnh sau để xem object Deployment `dev-my-nginx`:

```shell
kubectl get -k ./
```

```shell
kubectl describe -k ./
```

Chạy lệnh sau để so sánh object Deployment `dev-my-nginx` với trạng thái mà cluster sẽ có
nếu manifest được apply:

```shell
kubectl diff -k ./
```

Chạy lệnh sau để xóa object Deployment `dev-my-nginx`:

```shell
> kubectl delete -k ./
deployment.apps "dev-my-nginx" deleted
```

## Danh sách tính năng của Kustomize (Kustomize Feature List)

| Trường | Kiểu | Giải thích |
|-------|------|-------------|
| bases | []string | Mỗi mục trong danh sách này phải trỏ tới một thư mục chứa file kustomization.yaml |
| commonAnnotations | map[string]string | các annotation thêm vào tất cả tài nguyên |
| commonLabels | map[string]string | các label thêm vào tất cả tài nguyên và selector |
| configMapGenerator | [][ConfigMapArgs](https://github.com/kubernetes-sigs/kustomize/blob/master/api/types/configmapargs.go#L7) | Mỗi mục trong danh sách này sinh ra một ConfigMap |
| configurations | []string | Mỗi mục trong danh sách này phải trỏ tới một file chứa [cấu hình transformer của Kustomize](https://github.com/kubernetes-sigs/kustomize/tree/master/examples/transformerconfigs) |
| crds | []string | Mỗi mục trong danh sách này phải trỏ tới một file định nghĩa OpenAPI cho các kiểu Kubernetes |
| generatorOptions | [GeneratorOptions](https://github.com/kubernetes-sigs/kustomize/blob/master/api/types/generatoroptions.go#L7) | Thay đổi hành vi của tất cả generator ConfigMap và Secret |
| images | [][Image](https://github.com/kubernetes-sigs/kustomize/blob/master/api/types/image.go#L8) | Mỗi mục dùng để sửa tên, tag và/hoặc digest của một image mà không cần tạo patch |
| labels | map[string]string | Thêm label mà không tự động chèn selector tương ứng |
| namePrefix | string | giá trị của trường này được thêm vào trước tên của tất cả tài nguyên |
| nameSuffix | string | giá trị của trường này được thêm vào sau tên của tất cả tài nguyên |
| patchesJson6902 | [][Patch](https://github.com/kubernetes-sigs/kustomize/blob/master/api/types/patch.go#L10) | Mỗi mục trong danh sách này phải trỏ tới một object Kubernetes và một Json Patch |
| patchesStrategicMerge | []string | Mỗi mục trong danh sách này phải trỏ tới một strategic merge patch của một object Kubernetes |
| replacements | [][Replacements](https://github.com/kubernetes-sigs/kustomize/blob/master/api/types/replacement.go#L15) | sao chép giá trị từ trường của một tài nguyên vào bất kỳ số lượng đích được chỉ định nào. |
| resources | []string | Mỗi mục trong danh sách này phải trỏ tới một file cấu hình tài nguyên đang tồn tại |
| secretGenerator | [][SecretArgs](https://github.com/kubernetes-sigs/kustomize/blob/master/api/types/secretargs.go#L7) | Mỗi mục trong danh sách này sinh ra một Secret |
| vars | [][Var](https://github.com/kubernetes-sigs/kustomize/blob/master/api/types/var.go#L19) | Mỗi mục dùng để bắt (capture) văn bản từ trường của một tài nguyên |

## Tiếp theo (What's next)

* [Kustomize](https://github.com/kubernetes-sigs/kustomize)
* [Kubectl Book](https://kubectl.docs.kubernetes.io)
* [Tài liệu tham khảo lệnh Kubectl](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/)
* [Tài liệu tham khảo Kubernetes API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/)

