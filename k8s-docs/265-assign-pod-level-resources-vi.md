# Gán tài nguyên CPU và memory ở cấp Pod (Assign Pod-level CPU and memory resources)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-pod-level-resources/

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.34 [beta]`

Trang này hướng dẫn cách chỉ định tài nguyên CPU và memory cho một Pod ở cấp Pod (pod-level),
bên cạnh việc khai báo tài nguyên ở cấp container (container-level). Một node Kubernetes
cấp phát (allocate) tài nguyên cho một Pod dựa trên các yêu cầu tài nguyên (resource requests)
của Pod đó. Các yêu cầu này có thể được định nghĩa ở cấp Pod hoặc riêng lẻ cho từng container
bên trong Pod. Khi cả hai cùng tồn tại, yêu cầu ở cấp Pod sẽ được ưu tiên.

Tương tự, mức sử dụng tài nguyên của một Pod bị ràng buộc bởi các giới hạn (limits), cũng có
thể được đặt ở cấp Pod hoặc riêng lẻ cho từng container bên trong Pod. Một lần nữa, giới hạn
ở cấp Pod được ưu tiên khi cả hai cùng tồn tại. Điều này cho phép quản lý tài nguyên một cách
linh hoạt, giúp bạn kiểm soát việc cấp phát tài nguyên ở cả cấp Pod lẫn cấp container.

Để chỉ định tài nguyên ở cấp Pod, bạn bắt buộc phải bật
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`PodLevelResources`.

Đối với tài nguyên cấp Pod (Pod Level Resources):

* Độ ưu tiên (Priority): Khi tài nguyên được chỉ định ở cả cấp Pod và cấp container,
  tài nguyên cấp Pod sẽ được ưu tiên.
* QoS: Tài nguyên cấp Pod được ưu tiên khi xác định lớp QoS (QoS class) của Pod.
* Điểm OOM (OOM Score): Việc tính toán điều chỉnh điểm OOM xét đến cả tài nguyên cấp Pod
  lẫn tài nguyên cấp container.
* Tính tương thích (Compatibility): Tài nguyên cấp Pod được thiết kế để tương thích với
  các tính năng hiện có.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.34 hoặc mới hơn. Để kiểm tra phiên bản,
nhập `kubectl version`.

[Feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`PodLevelResources` phải được bật cho control plane và cho tất cả các node trong cluster của bạn.

## Giới hạn (Limitations)

Đối với Kubernetes v1.36, tài nguyên cấp Pod có các giới hạn sau:

* **Loại tài nguyên (Resource Types):** Chỉ có CPU, memory và hugepages là các tài nguyên
  có thể được chỉ định ở cấp Pod.
* **Hệ điều hành (Operating System):** Tài nguyên cấp Pod không được hỗ trợ cho các Pod
  Windows.
* **Các trình quản lý tài nguyên (Resource Managers):** Topology Manager, Memory Manager và
  CPU Manager hỗ trợ tài nguyên cấp Pod khi
  [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
  `PodLevelResourceManagers` được bật. Xem
  [Các trình quản lý tài nguyên cấp Pod](74-resource-managers-vi.md)
  để biết thêm chi tiết. Nếu feature gate này không được bật, chúng sẽ không căn chỉnh
  (align) Pod và container dựa trên tài nguyên cấp Pod.
* **Thay đổi kích thước tại chỗ (In-Place Resize):**
  [Thay đổi kích thước tại chỗ (in-place resize)](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)
  đối với tài nguyên cấp Pod yêu cầu feature gate `InPlacePodLevelResourcesVerticalScaling`,
  hiện ở trạng thái alpha trong Kubernetes v1.36. Để biết thêm chi tiết, xem
  [Thay đổi kích thước tài nguyên CPU và Memory của Pod](https://kubernetes.io/docs/tasks/configure-pod-container/resize-pod-resources/).

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi
phần còn lại của cluster.

```shell
kubectl create namespace pod-resources-example
```

## Tạo một Pod với memory request và limit ở cấp Pod (Create a pod with memory requests and limits at pod-level)

Để chỉ định memory request cho một Pod ở cấp Pod, thêm trường `resources.requests.memory`
vào manifest spec của Pod. Để chỉ định memory limit, thêm trường `resources.limits.memory`.

Trong bài thực hành này, bạn tạo một Pod có một Container. Pod có memory request là 100 MiB
và memory limit là 200 MiB. Đây là file cấu hình cho Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: pod-resources-example
spec:
  resources:
    requests:
      memory: "100Mi"
    limits:
      memory: "200Mi"
  containers:
  - name: memory-demo-ctr
    image: nginx
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

Mục `args` trong manifest cung cấp các tham số cho container khi nó khởi động.
Các tham số `"--vm-bytes", "150M"` yêu cầu Container cố gắng cấp phát 150 MiB memory.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/pod-level-memory-request-limit.yaml --namespace=pod-resources-example
```

Xác nhận rằng Pod đang chạy:

```shell
kubectl get pod memory-demo --namespace=pod-resources-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod memory-demo --output=yaml --namespace=pod-resources-example
```

Kết quả cho thấy Pod có memory request là 100 MiB và memory limit là 200 MiB.

```yaml
...
spec:
  containers:
  ...
  resources:
    requests:
      memory: 100Mi
    limits:
      memory: 200Mi
...
```

Chạy `kubectl top` để lấy các số liệu (metrics) của Pod:

```shell
kubectl top pod memory-demo --namespace=pod-resources-example
```

Kết quả cho thấy Pod đang sử dụng khoảng 162.900.000 byte memory, tức khoảng 150 MiB.
Con số này lớn hơn mức request 100 MiB của Pod, nhưng vẫn nằm trong mức limit 200 MiB
của Pod.

```
NAME                        CPU(cores)   MEMORY(bytes)
memory-demo                 <something>  162856960
```

## Tạo một Pod với CPU request và limit ở cấp Pod (Create a pod with CPU requests and limits at pod-level)

Để chỉ định CPU request cho một Pod, thêm trường `resources.requests.cpu` vào manifest spec
của Pod. Để chỉ định CPU limit, thêm trường `resources.limits.cpu`.

Trong bài thực hành này, bạn tạo một Pod có một container. Pod có request là 0.5 CPU và
limit là 1 CPU. Đây là file cấu hình cho Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo
  namespace: pod-resources-example
spec:
  resources:
    limits:
      cpu: "1"
    requests:
      cpu: "0.5"
  containers:
  - name: cpu-demo-ctr
    image: vish/stress
    args:
    - -cpus
    - "2"
```

Mục `args` của file cấu hình cung cấp các tham số cho container khi nó khởi động.
Tham số `-cpus "2"` yêu cầu Container cố gắng sử dụng 2 CPU.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/pod-level-cpu-request-limit.yaml --namespace=pod-resources-example
```

Xác nhận rằng Pod đang chạy:

```shell
kubectl get pod cpu-demo --namespace=pod-resources-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod cpu-demo --output=yaml --namespace=pod-resources-example
```

Kết quả cho thấy Pod có CPU request là 500 milliCPU và CPU limit là 1 CPU.

```yaml
spec:
  containers:
  ...
  resources:
    limits:
      cpu: "1"
    requests:
      cpu: 500m
```

Dùng `kubectl top` để lấy các số liệu của Pod:

```shell
kubectl top pod cpu-demo --namespace=pod-resources-example
```

Kết quả ví dụ này cho thấy Pod đang sử dụng 974 milliCPU, thấp hơn một chút so với
mức limit 1 CPU được chỉ định trong cấu hình của Pod.

```
NAME                        CPU(cores)   MEMORY(bytes)
cpu-demo                    974m         <something>
```

Hãy nhớ rằng khi đặt `-cpu "2"`, bạn đã cấu hình Container cố gắng sử dụng 2 CPU, nhưng
Container chỉ được phép sử dụng khoảng 1 CPU. Mức sử dụng CPU của container đang bị
điều tiết (throttle), vì container đang cố sử dụng nhiều tài nguyên CPU hơn mức CPU limit
của Pod.

## Tạo một Pod với request và limit ở cả cấp Pod lẫn cấp container (Create a pod with resource requests and limits at both pod-level and container-level)

Để gán tài nguyên CPU và memory cho một Pod, bạn có thể chỉ định chúng ở cả cấp Pod lẫn
cấp container. Thêm trường `resources` vào spec của Pod để định nghĩa tài nguyên cho toàn bộ
Pod. Ngoài ra, thêm trường `resources` bên trong phần khai báo container trong manifest của
Pod để đặt các yêu cầu tài nguyên riêng cho từng container.

Trong bài thực hành này, bạn sẽ tạo một Pod có hai container để tìm hiểu sự tương tác giữa
khai báo tài nguyên cấp Pod và cấp container. Bản thân Pod sẽ có CPU request và limit được
định nghĩa, trong khi chỉ một trong hai container có request và limit tài nguyên tường minh
của riêng nó. Container còn lại sẽ thừa hưởng các ràng buộc tài nguyên từ thiết lập cấp Pod.
Đây là file cấu hình cho Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources-demo
  namespace: pod-resources-example
spec:
  resources:
    limits:
      cpu: "1"
      memory: "200Mi"
    requests:
      cpu: "1"
      memory: "100Mi"
  containers:
  - name: pod-resources-demo-ctr-1
    image: nginx
    resources:
      limits:
        cpu: "0.5"
        memory: "100Mi"
      requests:
        cpu: "0.5"
        memory: "50Mi"
  - name: pod-resources-demo-ctr-2
    image: fedora
    command:
    - sleep
    - inf
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/pod-level-resources.yaml --namespace=pod-resources-example
```

Xác nhận rằng Container của Pod đang chạy:

```shell
kubectl get pod pod-resources-demo --namespace=pod-resources-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod pod-resources-demo --output=yaml --namespace=pod-resources-example
```

Kết quả cho thấy một container trong Pod có memory request là 50 MiB và CPU request là
0.5 core, với memory limit là 100 MiB và CPU limit là 0.5 core. Bản thân Pod có memory
request là 100 MiB và CPU request là 1 core, cùng memory limit là 200 MiB và CPU limit
là 1 core.

```yaml
...
  containers:
  -
    name: pod-resources-demo-ctr-1
    resources:
      limits:
        cpu: 500m
        memory: 100Mi
      requests:
        cpu: 500m
        memory: 50Mi
...
  -
    name: pod-resources-demo-ctr-2
    resources: {}
...
  resources:
    limits:
      cpu: "1"
      memory: 200Mi
    requests:
      cpu: "1"
      memory: 100Mi
...
```

Vì request và limit ở cấp Pod đã được chỉ định, mức đảm bảo request cho cả hai container
trong Pod sẽ bằng 1 core CPU và 100Mi memory. Ngoài ra, cả hai container cộng lại sẽ không
thể sử dụng nhiều tài nguyên hơn mức được chỉ định trong limit cấp Pod, đảm bảo tổng cộng
chúng không thể vượt quá 200 MiB memory và 1 core CPU.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace pod-resources-example
```

## Tiếp theo (What's next)

### Dành cho nhà phát triển ứng dụng (For application developers)

* [Gán tài nguyên Memory cho Container và Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)

* [Gán tài nguyên CPU cho Container và Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình Memory Request và Limit mặc định cho một Namespace](232-memory-default-namespace-vi.md)

* [Cấu hình CPU Request và Limit mặc định cho một Namespace](230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc Memory tối thiểu và tối đa cho một Namespace](231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](229-cpu-constraint-namespace-vi.md)

* [Cấu hình Quota Memory và CPU cho một Namespace](233-quota-memory-cpu-namespace-vi.md)

* [Các trình quản lý tài nguyên cấp Pod](74-resource-managers-vi.md)
