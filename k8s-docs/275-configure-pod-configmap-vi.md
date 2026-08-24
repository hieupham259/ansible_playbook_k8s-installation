# Cấu hình một Pod để sử dụng ConfigMap (Configure a Pod to Use a ConfigMap)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/>

Nhiều ứng dụng dựa vào cấu hình được sử dụng trong quá trình khởi tạo ứng dụng hoặc lúc runtime.
Trong hầu hết các trường hợp, sẽ có nhu cầu điều chỉnh các giá trị được gán cho các tham số cấu
hình. ConfigMap là một cơ chế của Kubernetes cho phép bạn đưa (inject) dữ liệu cấu hình vào các
Pod của ứng dụng.

Khái niệm ConfigMap cho phép bạn tách rời các thành phần cấu hình khỏi nội dung image để giữ cho
ứng dụng đóng gói container có tính di động (portable). Ví dụ, bạn có thể tải về và chạy cùng một
container image để khởi động các container phục vụ cho mục đích phát triển cục bộ, kiểm thử hệ
thống, hoặc chạy một workload thực tế cho người dùng cuối.

Trang này cung cấp một loạt ví dụ sử dụng, minh họa cách tạo ConfigMap và cấu hình Pod bằng dữ
liệu được lưu trong ConfigMap.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy nhập `kubectl version`.

Bạn cần cài đặt sẵn công cụ `wget`. Nếu bạn có một công cụ khác như `curl` mà không có `wget`,
bạn sẽ cần điều chỉnh lại bước tải dữ liệu ví dụ.

## Tạo một ConfigMap (Create a ConfigMap) {#create-a-configmap}

Bạn có thể dùng `kubectl create configmap` hoặc một ConfigMap generator trong `kustomization.yaml`
để tạo một ConfigMap.

### Tạo ConfigMap bằng `kubectl create configmap` (Create a ConfigMap using `kubectl create configmap`)

Sử dụng lệnh `kubectl create configmap` để tạo ConfigMap từ
[thư mục](#create-configmaps-from-directories), [file](#create-configmaps-from-files),
hoặc [giá trị literal](#create-configmaps-from-literal-values):

```shell
kubectl create configmap <map-name> <data-source>
```

trong đó \<map-name> là tên bạn muốn gán cho ConfigMap và \<data-source> là thư mục, file, hoặc
giá trị literal để lấy dữ liệu.
Tên của một object ConfigMap phải là một
[tên miền con DNS (DNS subdomain name)](17-names-vi.md#dns-subdomain-names)
hợp lệ.

Khi bạn tạo một ConfigMap dựa trên một file, key trong \<data-source> mặc định là basename (tên
file không kèm đường dẫn) của file đó, và giá trị mặc định là nội dung của file.

Bạn có thể dùng [`kubectl describe`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#describe)
hoặc [`kubectl get`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#get)
để truy xuất thông tin về một ConfigMap.

#### Tạo ConfigMap từ một thư mục (Create a ConfigMap from a directory) {#create-configmaps-from-directories}

Bạn có thể dùng `kubectl create configmap` để tạo một ConfigMap từ nhiều file trong cùng một thư
mục. Khi bạn tạo ConfigMap dựa trên một thư mục, kubectl xác định các file trong thư mục có tên
file là một key hợp lệ và đóng gói từng file đó vào ConfigMap mới. Mọi mục trong thư mục không
phải file thông thường sẽ bị bỏ qua (ví dụ: thư mục con, symlink, device, pipe, v.v.).

> **Ghi chú:**
> Mỗi tên file được dùng để tạo ConfigMap chỉ được chứa các ký tự được chấp nhận, gồm: chữ cái
> (`A` đến `Z` và `a` đến `z`), chữ số (`0` đến `9`), '-', '_', hoặc '.'.
> Nếu bạn dùng `kubectl create configmap` với một thư mục mà bất kỳ tên file nào chứa ký tự không
> được chấp nhận, lệnh `kubectl` có thể thất bại.
>
> Lệnh `kubectl` không in ra lỗi khi nó gặp một tên file không hợp lệ.

Tạo thư mục cục bộ:

```shell
mkdir -p configure-pod-container/configmap/
```

Bây giờ, tải về cấu hình mẫu và tạo ConfigMap:

```shell
# Tải các file mẫu vào thư mục `configure-pod-container/configmap/`
wget https://kubernetes.io/examples/configmap/game.properties -O configure-pod-container/configmap/game.properties
wget https://kubernetes.io/examples/configmap/ui.properties -O configure-pod-container/configmap/ui.properties

# Tạo ConfigMap
kubectl create configmap game-config --from-file=configure-pod-container/configmap/
```

Lệnh trên đóng gói từng file, trong trường hợp này là `game.properties` và `ui.properties` trong
thư mục `configure-pod-container/configmap/`, vào ConfigMap có tên game-config. Bạn có thể hiển
thị chi tiết của ConfigMap bằng lệnh sau:

```shell
kubectl describe configmaps game-config
```

Kết quả tương tự như sau:

```
Name:         game-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
ui.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
```

Các file `game.properties` và `ui.properties` trong thư mục `configure-pod-container/configmap/`
được thể hiện trong phần `data` của ConfigMap.

```shell
kubectl get configmaps game-config -o yaml
```

Kết quả tương tự như sau:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2022-02-18T18:52:05Z
  name: game-config
  namespace: default
  resourceVersion: "516"
  uid: b4952dc3-d670-11e5-8cd0-68f728db1985
data:
  game.properties: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
```

#### Tạo ConfigMap từ file (Create ConfigMaps from files) {#create-configmaps-from-files}

Bạn có thể dùng `kubectl create configmap` để tạo một ConfigMap từ một file riêng lẻ, hoặc từ
nhiều file.

Ví dụ,

```shell
kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties
```

sẽ tạo ra ConfigMap sau:

```shell
kubectl describe configmaps game-config-2
```

với kết quả tương tự như sau:

```
Name:         game-config-2
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
```

Bạn có thể truyền tham số `--from-file` nhiều lần để tạo một ConfigMap từ nhiều nguồn dữ liệu.

```shell
kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties --from-file=configure-pod-container/configmap/ui.properties
```

Bạn có thể hiển thị chi tiết của ConfigMap `game-config-2` bằng lệnh sau:

```shell
kubectl describe configmaps game-config-2
```

Kết quả tương tự như sau:

```
Name:         game-config-2
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
ui.properties:
----
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice
```

Sử dụng tùy chọn `--from-env-file` để tạo một ConfigMap từ một env-file, ví dụ:

```shell
# Env-file chứa một danh sách các biến môi trường.
# Các quy tắc cú pháp sau được áp dụng:
#   Mỗi dòng trong env-file phải ở định dạng VAR=VAL.
#   Các dòng bắt đầu bằng # (tức là comment) sẽ bị bỏ qua.
#   Các dòng trống sẽ bị bỏ qua.
#   Không có xử lý đặc biệt cho dấu nháy (tức là chúng sẽ trở thành một phần của giá trị trong ConfigMap).

# Tải các file mẫu vào thư mục `configure-pod-container/configmap/`
wget https://kubernetes.io/examples/configmap/game-env-file.properties -O configure-pod-container/configmap/game-env-file.properties
wget https://kubernetes.io/examples/configmap/ui-env-file.properties -O configure-pod-container/configmap/ui-env-file.properties

# Env-file `game-env-file.properties` có nội dung như dưới đây
cat configure-pod-container/configmap/game-env-file.properties
enemies=aliens
lives=3
allowed="true"

# This comment and the empty line above it are ignored
```

```shell
kubectl create configmap game-config-env-file \
       --from-env-file=configure-pod-container/configmap/game-env-file.properties
```

sẽ tạo ra một ConfigMap. Xem ConfigMap đó:

```shell
kubectl get configmap game-config-env-file -o yaml
```

kết quả tương tự như sau:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2019-12-27T18:36:28Z
  name: game-config-env-file
  namespace: default
  resourceVersion: "809965"
  uid: d9d1ca5b-eb34-11e7-887b-42010a8002b8
data:
  allowed: '"true"'
  enemies: aliens
  lives: "3"
```

Kể từ Kubernetes v1.23, `kubectl` hỗ trợ chỉ định tham số `--from-env-file` nhiều lần để tạo một
ConfigMap từ nhiều nguồn dữ liệu.

```shell
kubectl create configmap config-multi-env-files \
        --from-env-file=configure-pod-container/configmap/game-env-file.properties \
        --from-env-file=configure-pod-container/configmap/ui-env-file.properties
```

sẽ tạo ra ConfigMap sau:

```shell
kubectl get configmap config-multi-env-files -o yaml
```

với kết quả tương tự như sau:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2019-12-27T18:38:34Z
  name: config-multi-env-files
  namespace: default
  resourceVersion: "810136"
  uid: 252c4572-eb35-11e7-887b-42010a8002b8
data:
  allowed: '"true"'
  color: purple
  enemies: aliens
  how: fairlyNice
  lives: "3"
  textmode: "true"
```

#### Định nghĩa key sử dụng khi tạo ConfigMap từ file (Define the key to use when creating a ConfigMap from a file)

Bạn có thể định nghĩa một key khác với tên file để dùng trong phần `data` của ConfigMap khi sử
dụng tham số `--from-file`:

```shell
kubectl create configmap game-config-3 --from-file=<my-key-name>=<path-to-file>
```

trong đó `<my-key-name>` là key bạn muốn dùng trong ConfigMap và `<path-to-file>` là vị trí của
file nguồn dữ liệu mà bạn muốn key đó đại diện.

Ví dụ:

```shell
kubectl create configmap game-config-3 --from-file=game-special-key=configure-pod-container/configmap/game.properties
```

sẽ tạo ra ConfigMap sau:

```
kubectl get configmaps game-config-3 -o yaml
```

với kết quả tương tự như sau:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2022-02-18T18:54:22Z
  name: game-config-3
  namespace: default
  resourceVersion: "530"
  uid: 05f8da22-d671-11e5-8cd0-68f728db1985
data:
  game-special-key: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
```

#### Tạo ConfigMap từ giá trị literal (Create ConfigMaps from literal values) {#create-configmaps-from-literal-values}

Bạn có thể dùng `kubectl create configmap` với tham số `--from-literal` để định nghĩa một giá
trị literal (giá trị ghi trực tiếp) từ dòng lệnh:

```shell
kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
```

Bạn có thể truyền vào nhiều cặp key-value. Mỗi cặp được cung cấp trên dòng lệnh sẽ được thể hiện
thành một mục riêng trong phần `data` của ConfigMap.

```shell
kubectl get configmaps special-config -o yaml
```

Kết quả tương tự như sau:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2022-02-18T19:14:38Z
  name: special-config
  namespace: default
  resourceVersion: "651"
  uid: dadce046-d673-11e5-8cd0-68f728db1985
data:
  special.how: very
  special.type: charm
```

### Tạo ConfigMap từ generator (Create a ConfigMap from generator)

Bạn cũng có thể tạo một ConfigMap từ generator rồi apply nó để tạo object trên API server của
cluster. Bạn nên chỉ định các generator trong một file `kustomization.yaml` bên trong một thư mục.

#### Sinh ConfigMap từ file (Generate ConfigMaps from files)

Ví dụ, để sinh một ConfigMap từ file `configure-pod-container/configmap/game.properties`

```shell
# Tạo file kustomization.yaml với ConfigMapGenerator
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: game-config-4
  options:
    labels:
      game-config: config-4
  files:
  - configure-pod-container/configmap/game.properties
EOF
```

Apply thư mục kustomization để tạo object ConfigMap:

```shell
kubectl apply -k .
```

```
configmap/game-config-4-m9dm2f92bt created
```

Bạn có thể kiểm tra rằng ConfigMap đã được tạo như sau:

```shell
kubectl get configmap
```

```
NAME                       DATA   AGE
game-config-4-m9dm2f92bt   1      37s
```

và cả:

```shell
kubectl describe configmaps/game-config-4-m9dm2f92bt
```

```
Name:         game-config-4-m9dm2f92bt
Namespace:    default
Labels:       game-config=config-4
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","data":{"game.properties":"enemies=aliens\nlives=3\nenemies.cheat=true\nenemies.cheat.level=noGoodRotten\nsecret.code.p...

Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
Events:  <none>
```

Lưu ý rằng tên ConfigMap được sinh ra có một hậu tố được thêm vào bằng cách băm (hash) nội dung.
Điều này đảm bảo rằng một ConfigMap mới sẽ được sinh ra mỗi khi nội dung bị thay đổi.

#### Định nghĩa key sử dụng khi sinh ConfigMap từ file (Define the key to use when generating a ConfigMap from a file)

Bạn có thể định nghĩa một key khác với tên file để dùng trong ConfigMap generator.
Ví dụ, để sinh một ConfigMap từ file `configure-pod-container/configmap/game.properties` với key
`game-special-key`

```shell
# Tạo file kustomization.yaml với ConfigMapGenerator
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: game-config-5
  options:
    labels:
      game-config: config-5
  files:
  - game-special-key=configure-pod-container/configmap/game.properties
EOF
```

Apply thư mục kustomization để tạo object ConfigMap.

```shell
kubectl apply -k .
```

```
configmap/game-config-5-m67dt67794 created
```

#### Sinh ConfigMap từ literal (Generate ConfigMaps from literals)

Ví dụ này chỉ cho bạn cách tạo một `ConfigMap` từ hai cặp key/value literal:
`special.type=charm` và `special.how=very`, bằng Kustomize và kubectl. Để làm điều đó, bạn có
thể chỉ định generator cho `ConfigMap`. Tạo (hoặc thay thế) `kustomization.yaml` sao cho nó có
nội dung sau:

```yaml
---
# Nội dung kustomization.yaml để tạo một ConfigMap từ các literal
configMapGenerator:
- name: special-config-2
  literals:
  - special.how=very
  - special.type=charm
```

Apply thư mục kustomization để tạo object ConfigMap:

```shell
kubectl apply -k .
```

```
configmap/special-config-2-c92b5mmcf2 created
```

## Dọn dẹp giữa chừng (Interim cleanup)

Trước khi tiếp tục, hãy dọn dẹp một số ConfigMap bạn đã tạo:

```bash
kubectl delete configmap special-config
kubectl delete configmap env-config
kubectl delete configmap -l 'game-config in (config-4,config-5)'
```

Bây giờ bạn đã học cách định nghĩa ConfigMap, bạn có thể chuyển sang phần tiếp theo để học cách
sử dụng các object này với Pod.

---

## Định nghĩa biến môi trường cho container bằng dữ liệu ConfigMap (Define container environment variables using ConfigMap data)

### Định nghĩa một biến môi trường cho container với dữ liệu từ một ConfigMap duy nhất (Define a container environment variable with data from a single ConfigMap)

1. Định nghĩa một biến môi trường dưới dạng cặp key-value trong một ConfigMap:

   ```shell
   kubectl create configmap special-config --from-literal=special.how=very
   ```

2. Gán giá trị `special.how` được định nghĩa trong ConfigMap cho biến môi trường
   `SPECIAL_LEVEL_KEY` trong đặc tả (specification) của Pod.

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: dapi-test-pod
   spec:
     containers:
       - name: test-container
         image: registry.k8s.io/busybox:1.27.2
         command: [ "/bin/sh", "-c", "env" ]
         env:
           # Định nghĩa biến môi trường
           - name: SPECIAL_LEVEL_KEY
             valueFrom:
               configMapKeyRef:
                 # ConfigMap chứa giá trị bạn muốn gán cho SPECIAL_LEVEL_KEY
                 name: special-config
                 # Chỉ định key gắn với giá trị đó
                 key: special.how
     restartPolicy: Never
   ```

   Tạo Pod:

   ```shell
   kubectl create -f https://kubernetes.io/examples/pods/pod-single-configmap-env-variable.yaml
   ```

   Bây giờ, output của Pod sẽ bao gồm biến môi trường `SPECIAL_LEVEL_KEY=very`.

### Định nghĩa các biến môi trường cho container với dữ liệu từ nhiều ConfigMap (Define container environment variables with data from multiple ConfigMaps)

Giống như ví dụ trước, hãy tạo các ConfigMap trước.
Đây là manifest bạn sẽ dùng:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
```

* Tạo ConfigMap:

  ```shell
  kubectl create -f https://kubernetes.io/examples/configmap/configmaps.yaml
  ```

* Định nghĩa các biến môi trường trong đặc tả của Pod.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: dapi-test-pod
  spec:
    containers:
      - name: test-container
        image: registry.k8s.io/busybox:1.27.2
        command: [ "/bin/sh", "-c", "env" ]
        env:
          - name: SPECIAL_LEVEL_KEY
            valueFrom:
              configMapKeyRef:
                name: special-config
                key: special.how
          - name: LOG_LEVEL
            valueFrom:
              configMapKeyRef:
                name: env-config
                key: log_level
    restartPolicy: Never
  ```

  Tạo Pod:

  ```shell
  kubectl create -f https://kubernetes.io/examples/pods/pod-multiple-configmap-env-variable.yaml
  ```

  Bây giờ, output của Pod sẽ bao gồm các biến môi trường `SPECIAL_LEVEL_KEY=very` và
  `LOG_LEVEL=INFO`.

  Khi bạn đã sẵn sàng chuyển sang phần tiếp theo, hãy xóa Pod và ConfigMap đó:

  ```shell
  kubectl delete pod dapi-test-pod --now
  kubectl delete configmap special-config
  kubectl delete configmap env-config
  ```

## Cấu hình tất cả các cặp key-value trong ConfigMap thành biến môi trường của container (Configure all key-value pairs in a ConfigMap as container environment variables) {#configure-all-key-value-pairs-in-a-configmap-as-container-environment-variables}

* Tạo một ConfigMap chứa nhiều cặp key-value.

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: special-config
    namespace: default
  data:
    SPECIAL_LEVEL: very
    SPECIAL_TYPE: charm
  ```

  Tạo ConfigMap:

  ```shell
  kubectl create -f https://kubernetes.io/examples/configmap/configmap-multikeys.yaml
  ```

* Sử dụng `envFrom` để định nghĩa toàn bộ dữ liệu của ConfigMap thành các biến môi trường của
  container. Key trong ConfigMap sẽ trở thành tên biến môi trường trong Pod.

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: dapi-test-pod
  spec:
    containers:
      - name: test-container
        image: registry.k8s.io/busybox:1.27.2
        command: [ "/bin/sh", "-c", "env" ]
        envFrom:
        - configMapRef:
            name: special-config
    restartPolicy: Never
  ```

  Tạo Pod:

  ```shell
  kubectl create -f https://kubernetes.io/examples/pods/pod-configmap-envFrom.yaml
  ```

  Bây giờ, output của Pod sẽ bao gồm các biến môi trường `SPECIAL_LEVEL=very` và
  `SPECIAL_TYPE=charm`.

  Khi bạn đã sẵn sàng chuyển sang phần tiếp theo, hãy xóa Pod đó:

  ```shell
  kubectl delete pod dapi-test-pod --now
  ```

## Sử dụng biến môi trường định nghĩa từ ConfigMap trong lệnh của Pod (Use ConfigMap-defined environment variables in Pod commands)

Bạn có thể sử dụng các biến môi trường định nghĩa từ ConfigMap trong `command` và `args` của một
container bằng cú pháp thay thế `$(VAR_NAME)` của Kubernetes.

Ví dụ, manifest Pod sau:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox:1.27.2
      command: [ "/bin/echo", "$(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: SPECIAL_LEVEL
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: SPECIAL_TYPE
  restartPolicy: Never
```

Tạo Pod đó bằng cách chạy:

```shell
kubectl create -f https://kubernetes.io/examples/pods/pod-configmap-env-var-valueFrom.yaml
```

Pod đó tạo ra output sau từ container `test-container`:

```shell
kubectl logs dapi-test-pod
```

```
very charm
```

Khi bạn đã sẵn sàng chuyển sang phần tiếp theo, hãy xóa Pod đó:

```shell
kubectl delete pod dapi-test-pod --now
```

## Thêm dữ liệu ConfigMap vào một Volume (Add ConfigMap data to a Volume)

Như đã giải thích trong [Tạo ConfigMap từ file](#create-configmaps-from-files), khi bạn tạo một
ConfigMap bằng `--from-file`, tên file trở thành một key được lưu trong phần `data` của
ConfigMap. Nội dung file trở thành giá trị của key đó.

Các ví dụ trong phần này tham chiếu tới một ConfigMap tên là `special-config`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm
```

Tạo ConfigMap:

```shell
kubectl create -f https://kubernetes.io/examples/configmap/configmap-multikeys.yaml
```

### Nạp dữ liệu lưu trong ConfigMap vào một Volume (Populate a Volume with data stored in a ConfigMap)

Thêm tên ConfigMap vào phần `volumes` của đặc tả Pod.
Việc này thêm dữ liệu của ConfigMap vào thư mục được chỉ định bởi `volumeMounts.mountPath`
(trong trường hợp này là `/etc/config`). Phần `command` liệt kê các file trong thư mục với tên
khớp với các key trong ConfigMap.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox:1.27.2
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # Cung cấp tên của ConfigMap chứa các file bạn muốn
        # thêm vào container
        name: special-config
  restartPolicy: Never
```

Tạo Pod:

```shell
kubectl create -f https://kubernetes.io/examples/pods/pod-configmap-volume.yaml
```

Khi Pod chạy, lệnh `ls /etc/config/` tạo ra output dưới đây:

```
SPECIAL_LEVEL
SPECIAL_TYPE
```

Dữ liệu dạng văn bản được phơi ra dưới dạng file sử dụng bảng mã ký tự UTF-8. Để dùng bảng mã ký
tự khác, hãy dùng `binaryData`
(xem [object ConfigMap](108-configmap-vi.md#configmap-object)
để biết thêm chi tiết).

> **Ghi chú:**
> Nếu có bất kỳ file nào trong thư mục `/etc/config` của container image đó, việc mount volume
> sẽ khiến các file đó của image không thể truy cập được nữa.

Khi bạn đã sẵn sàng chuyển sang phần tiếp theo, hãy xóa Pod đó:

```shell
kubectl delete pod dapi-test-pod --now
```

### Thêm dữ liệu ConfigMap vào một path cụ thể trong Volume (Add ConfigMap data to a specific path in the Volume)

Sử dụng field `path` để chỉ định đường dẫn file mong muốn cho các mục cụ thể của ConfigMap.
Trong trường hợp này, mục `SPECIAL_LEVEL` sẽ được mount trong volume `config-volume` tại
`/etc/config/keys`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: registry.k8s.io/busybox:1.27.2
      command: [ "/bin/sh","-c","cat /etc/config/keys" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: SPECIAL_LEVEL
          path: keys
  restartPolicy: Never
```

Tạo Pod:

```shell
kubectl create -f https://kubernetes.io/examples/pods/pod-configmap-volume-specific-key.yaml
```

Khi Pod chạy, lệnh `cat /etc/config/keys` tạo ra output dưới đây:

```
very
```

> **Thận trọng:**
> Giống như trước, tất cả các file có sẵn trước đó trong thư mục `/etc/config/` sẽ bị xóa.

Xóa Pod đó:

```shell
kubectl delete pod dapi-test-pod --now
```

### Chiếu các key tới path và quyền file cụ thể (Project keys to specific paths and file permissions)

Bạn có thể chiếu (project) các key tới các path cụ thể. Tham khảo mục tương ứng trong hướng dẫn
[Secrets](334-distribute-credentials-secure-vi.md#project-secret-keys-to-specific-file-paths)
để biết cú pháp.
Bạn có thể đặt quyền POSIX (POSIX permissions) cho các key. Tham khảo mục tương ứng trong hướng
dẫn [Secrets](334-distribute-credentials-secure-vi.md#set-posix-permissions-for-secret-keys)
để biết cú pháp.

### Tham chiếu optional (Optional references)

Một tham chiếu ConfigMap có thể được đánh dấu là _optional_ (tùy chọn). Nếu ConfigMap không tồn
tại, volume được mount sẽ trống. Nếu ConfigMap tồn tại nhưng key được tham chiếu không tồn tại,
path đó sẽ không xuất hiện bên dưới điểm mount (mount point). Xem
[ConfigMap optional](#optional-configmaps) để biết thêm chi tiết.

### ConfigMap được mount sẽ tự động được cập nhật (Mounted ConfigMaps are updated automatically)

Khi một ConfigMap đang được mount bị cập nhật, nội dung được chiếu vào Pod rốt cuộc cũng sẽ được
cập nhật theo. Điều này cũng áp dụng cho trường hợp một ConfigMap được tham chiếu ở dạng
optional xuất hiện sau khi Pod đã khởi động.

Kubelet kiểm tra xem ConfigMap được mount có còn mới hay không trong mỗi chu kỳ đồng bộ định kỳ.
Tuy nhiên, nó dùng cache cục bộ dựa trên TTL để lấy giá trị hiện tại của ConfigMap. Kết quả là,
tổng độ trễ từ thời điểm ConfigMap được cập nhật đến thời điểm các key mới được chiếu vào Pod có
thể dài bằng chu kỳ đồng bộ của kubelet (mặc định là 1 phút) + TTL của cache ConfigMap trong
kubelet (mặc định là 1 phút). Bạn có thể kích hoạt việc làm mới ngay lập tức bằng cách cập nhật
một trong các annotation của Pod.

> **Ghi chú:**
> Một container sử dụng ConfigMap dưới dạng volume
> [subPath](91-volumes-vi.md#using-subpath) sẽ không nhận
> được các cập nhật của ConfigMap.

## Hiểu về ConfigMap và Pod (Understanding ConfigMaps and Pods)

Tài nguyên API ConfigMap lưu trữ dữ liệu cấu hình dưới dạng các cặp key-value. Dữ liệu có thể
được tiêu thụ trong các Pod hoặc cung cấp cấu hình cho các thành phần hệ thống như các
controller. ConfigMap tương tự như
[Secrets](109-secret-vi.md), nhưng cung cấp một phương
tiện để làm việc với các chuỗi không chứa thông tin nhạy cảm. Cả người dùng lẫn các thành phần
hệ thống đều có thể lưu dữ liệu cấu hình trong ConfigMap.

> **Ghi chú:**
> ConfigMap nên tham chiếu tới các file properties, chứ không thay thế chúng. Hãy hình dung
> ConfigMap như một thứ tương tự với thư mục `/etc` của Linux và nội dung bên trong nó. Ví dụ,
> nếu bạn tạo một [Volume Kubernetes](91-volumes-vi.md) từ
> một ConfigMap, mỗi mục dữ liệu trong ConfigMap được thể hiện bằng một file riêng lẻ trong
> volume đó.

Field `data` của ConfigMap chứa dữ liệu cấu hình. Như trong ví dụ dưới đây, dữ liệu này có thể
đơn giản (như các property riêng lẻ được định nghĩa bằng `--from-literal`) hoặc phức tạp (như
file cấu hình hoặc các khối JSON được định nghĩa bằng `--from-file`).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T19:14:38Z
  name: example-config
  namespace: default
data:
  # ví dụ về một property đơn giản được định nghĩa bằng --from-literal
  example.property.1: hello
  example.property.2: world
  # ví dụ về một property phức tạp được định nghĩa bằng --from-file
  example.property.file: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3
```

Khi `kubectl` tạo một ConfigMap từ dữ liệu đầu vào không phải ASCII hoặc UTF-8, công cụ này sẽ
đặt chúng vào field `binaryData` của ConfigMap thay vì `data`. Cả nguồn dữ liệu văn bản lẫn nhị
phân đều có thể được kết hợp trong cùng một ConfigMap.

Nếu bạn muốn xem các key trong `binaryData` (cùng giá trị của chúng) trong một ConfigMap, bạn có
thể chạy `kubectl get configmap -o jsonpath='{.binaryData}' <name>`.

Pod có thể nạp dữ liệu từ một ConfigMap sử dụng `data` hoặc `binaryData`.

## ConfigMap optional (Optional ConfigMaps) {#optional-configmaps}

Bạn có thể đánh dấu một tham chiếu tới ConfigMap là _optional_ trong đặc tả của Pod.
Nếu ConfigMap không tồn tại, phần cấu hình mà nó cung cấp dữ liệu trong Pod (ví dụ: biến môi
trường, volume được mount) sẽ trống.
Nếu ConfigMap tồn tại nhưng key được tham chiếu không tồn tại, dữ liệu cũng sẽ trống.

Ví dụ, đặc tả Pod sau đánh dấu một biến môi trường lấy từ ConfigMap là optional:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh", "-c", "env"]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: a-config
              key: akey
              optional: true # đánh dấu biến này là optional
  restartPolicy: Never
```

Nếu bạn chạy Pod này và không có ConfigMap nào tên là `a-config`, output sẽ trống.
Nếu bạn chạy Pod này và có một ConfigMap tên là `a-config` nhưng ConfigMap đó không có key tên
là `akey`, output cũng sẽ trống. Nếu bạn đặt một giá trị cho `akey` trong ConfigMap `a-config`,
Pod này sẽ in ra giá trị đó rồi kết thúc.

Bạn cũng có thể đánh dấu các volume và file được cung cấp bởi một ConfigMap là optional.
Kubernetes luôn tạo các đường dẫn mount cho volume, ngay cả khi ConfigMap hoặc key được tham
chiếu không tồn tại. Ví dụ, đặc tả Pod sau đánh dấu một volume tham chiếu tới ConfigMap là
optional:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: ["/bin/sh", "-c", "ls /etc/config"]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: no-config
        optional: true # đánh dấu ConfigMap nguồn là optional
  restartPolicy: Never
```

## Các hạn chế (Restrictions)

- Bạn phải tạo object `ConfigMap` trước khi tham chiếu tới nó trong đặc tả của một Pod. Hoặc
  cách khác, hãy đánh dấu tham chiếu ConfigMap là `optional` trong spec của Pod (xem
  [ConfigMap optional](#optional-configmaps)). Nếu bạn tham chiếu tới một ConfigMap không tồn
  tại và không đánh dấu tham chiếu đó là `optional`, Pod sẽ không khởi động được. Tương tự, các
  tham chiếu tới key không tồn tại trong ConfigMap cũng sẽ khiến Pod không khởi động được, trừ
  khi bạn đánh dấu các tham chiếu key đó là `optional`.

- Nếu bạn dùng `envFrom` để định nghĩa biến môi trường từ ConfigMap, các key bị coi là không
  hợp lệ sẽ bị bỏ qua. Pod vẫn được phép khởi động, nhưng các tên không hợp lệ sẽ được ghi lại
  trong event log (`InvalidVariableNames`). Thông điệp log liệt kê từng key bị bỏ qua. Ví dụ:

  ```shell
  kubectl get events
  ```

  Kết quả tương tự như sau:

  ```
  LASTSEEN FIRSTSEEN COUNT NAME          KIND  SUBOBJECT  TYPE      REASON                            SOURCE                MESSAGE
  0s       0s        1     dapi-test-pod Pod              Warning   InvalidEnvironmentVariableNames   {kubelet, 127.0.0.1}  Keys [1badkey, 2alsobad] from the EnvFrom configMap default/myconfig were skipped since they are considered invalid environment variable names.
  ```

- ConfigMap tồn tại trong một namespace cụ thể. Pod chỉ có thể tham chiếu tới các ConfigMap nằm
  trong cùng namespace với Pod đó.

- Bạn không thể dùng ConfigMap cho các static pod, vì kubelet không hỗ trợ điều này.

## Dọn dẹp (Cleaning up)

Xóa các ConfigMap và Pod mà bạn đã tạo:

```bash
kubectl delete configmaps/game-config configmaps/game-config-2 configmaps/game-config-3 \
               configmaps/game-config-env-file
kubectl delete pod dapi-test-pod --now

# Có thể bạn đã xóa nhóm tiếp theo này rồi
kubectl delete configmaps/special-config configmaps/env-config
kubectl delete configmap -l 'game-config in (config-4,config-5)'
```

Xóa file `kustomization.yaml` mà bạn đã dùng để sinh ConfigMap:

```bash
rm kustomization.yaml
```

Nếu bạn đã tạo thư mục `configure-pod-container` và không cần nó nữa, bạn cũng nên xóa nó, hoặc
chuyển nó vào thùng rác / nơi chứa file đã xóa.

```bash
rm -r configure-pod-container
```

## Tiếp theo (What's next)

* Theo dõi một ví dụ thực tế về
  [Cấu hình Redis bằng ConfigMap](https://kubernetes.io/docs/tutorials/configuration/configure-redis-using-configmap/).
* Theo dõi một ví dụ về
  [Cập nhật cấu hình thông qua ConfigMap](https://kubernetes.io/docs/tutorials/configuration/updating-configuration-via-a-configmap/).
