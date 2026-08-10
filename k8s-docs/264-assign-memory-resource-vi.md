# Gán tài nguyên memory cho Container và Pod (Assign Memory Resources to Containers and Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/>

Trang này hướng dẫn cách gán một *request* (yêu cầu) memory và một *limit* (giới hạn) memory cho
Container. Container được đảm bảo có đủ lượng memory mà nó yêu cầu, nhưng không được phép sử
dụng nhiều memory hơn limit của nó.

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

Mỗi node trong cluster của bạn phải có ít nhất 300 MiB memory.

Một vài bước trên trang này yêu cầu bạn chạy dịch vụ
[metrics-server](https://github.com/kubernetes-sigs/metrics-server) trong cluster. Nếu bạn đã có
metrics-server đang chạy, bạn có thể bỏ qua các bước đó.

Nếu bạn đang chạy Minikube, hãy chạy lệnh sau để bật metrics-server:

```shell
minikube addons enable metrics-server
```

Để xem metrics-server có đang chạy hay không, hoặc một trình cung cấp khác của resource metrics
API (`metrics.k8s.io`), hãy chạy lệnh sau:

```shell
kubectl get apiservices
```

Nếu resource metrics API khả dụng, kết quả sẽ bao gồm một tham chiếu đến `metrics.k8s.io`.

```shell
NAME
v1beta1.metrics.k8s.io
```

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi phần còn
lại của cluster.

```shell
kubectl create namespace mem-example
```

## Chỉ định memory request và memory limit (Specify a memory request and a memory limit)

Để chỉ định memory request cho một container, hãy thêm field `resources.requests.memory` vào
manifest tài nguyên của container. Để chỉ định memory limit, hãy thêm `resources.limits.memory`.

Trong bài thực hành này, bạn tạo một Pod có một Container. Container có memory request là
100 MiB và memory limit là 200 MiB. Đây là file cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

Mục `args` trong file cấu hình cung cấp các đối số cho Container khi nó khởi động. Các đối số
`"--vm-bytes", "150M"` yêu cầu Container cố gắng cấp phát 150 MiB memory.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit.yaml --namespace=mem-example
```

Xác minh rằng Container của Pod đang chạy:

```shell
kubectl get pod memory-demo --namespace=mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod memory-demo --output=yaml --namespace=mem-example
```

Kết quả cho thấy Container duy nhất trong Pod có memory request là 100 MiB và memory limit là
200 MiB.

```yaml
...
resources:
  requests:
    memory: 100Mi
  limits:
    memory: 200Mi
...
```

Chạy `kubectl top` để lấy các số liệu (metrics) của pod:

```shell
kubectl top pod memory-demo --namespace=mem-example
```

Kết quả cho thấy Pod đang sử dụng khoảng 162.900.000 byte memory, tức khoảng 150 MiB. Con số này
lớn hơn request 100 MiB của Pod, nhưng vẫn nằm trong limit 200 MiB của Pod.

```
NAME                        CPU(cores)   MEMORY(bytes)
memory-demo                 <something>  162856960
```

Xóa Pod của bạn:

```shell
kubectl delete pod memory-demo --namespace=mem-example
```

## Vượt quá memory limit của Container (Exceed a Container's memory limit)

Một Container có thể vượt quá memory request của nó nếu Node còn memory khả dụng. Nhưng Container
không được phép sử dụng nhiều hơn memory limit của nó. Nếu Container cấp phát nhiều memory hơn
limit, Container trở thành ứng viên bị chấm dứt (terminate). Nếu Container tiếp tục tiêu thụ
memory vượt quá limit, Container sẽ bị chấm dứt. Nếu Container bị chấm dứt có thể được khởi động
lại, kubelet sẽ khởi động lại nó, giống như với bất kỳ loại lỗi runtime nào khác.

Trong bài thực hành này, bạn tạo một Pod cố gắng cấp phát nhiều memory hơn limit của nó. Đây là
file cấu hình cho một Pod có một Container với memory request là 50 MiB và memory limit là
100 MiB:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-2
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-2-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

Trong mục `args` của file cấu hình, bạn có thể thấy Container sẽ cố gắng cấp phát 250 MiB
memory, vượt xa limit 100 MiB.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit-2.yaml --namespace=mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod memory-demo-2 --namespace=mem-example
```

Tại thời điểm này, Container có thể đang chạy hoặc đã bị kill. Lặp lại lệnh trên cho đến khi
Container bị kill:

```shell
NAME            READY     STATUS      RESTARTS   AGE
memory-demo-2   0/1       OOMKilled   1          24s
```

Xem trạng thái Container ở mức chi tiết hơn:

```shell
kubectl get pod memory-demo-2 --output=yaml --namespace=mem-example
```

Kết quả cho thấy Container đã bị kill vì hết memory (OOM):

```yaml
lastState:
   terminated:
     containerID: 65183c1877aaec2e8427bc95609cc52677a454b56fcb24340dbd22917c23b10f
     exitCode: 137
     finishedAt: 2017-06-20T20:52:19Z
     reason: OOMKilled
     startedAt: null
```

Container trong bài thực hành này có thể được khởi động lại, nên kubelet sẽ khởi động lại nó.
Lặp lại lệnh này vài lần để thấy Container liên tục bị kill rồi được khởi động lại:

```shell
kubectl get pod memory-demo-2 --namespace=mem-example
```

Kết quả cho thấy Container bị kill, được khởi động lại, bị kill lần nữa, lại được khởi động lại,
và cứ thế tiếp diễn:

```
kubectl get pod memory-demo-2 --namespace=mem-example
NAME            READY     STATUS      RESTARTS   AGE
memory-demo-2   0/1       OOMKilled   1          37s
```
```

kubectl get pod memory-demo-2 --namespace=mem-example
NAME            READY     STATUS    RESTARTS   AGE
memory-demo-2   1/1       Running   2          40s
```

Xem thông tin chi tiết về lịch sử của Pod:

```
kubectl describe pod memory-demo-2 --namespace=mem-example
```

Kết quả cho thấy Container khởi động rồi thất bại lặp đi lặp lại:

```
... Normal  Created   Created container with id 66a3a20aa7980e61be4922780bf9d24d1a1d8b7395c09861225b0eba1b1f8511
... Warning BackOff   Back-off restarting failed container
```

Xem thông tin chi tiết về các Node của cluster:

```
kubectl describe nodes
```

Kết quả bao gồm một bản ghi về việc Container bị kill do tình trạng hết memory (out-of-memory):

```
Warning OOMKilling Memory cgroup out of memory: Kill process 4481 (stress) score 1994 or sacrifice child
```

Xóa Pod của bạn:

```shell
kubectl delete pod memory-demo-2 --namespace=mem-example
```

## Chỉ định một memory request quá lớn so với các Node của bạn (Specify a memory request that is too big for your Nodes)

Memory request và limit gắn với Container, nhưng sẽ hữu ích khi coi Pod cũng có memory request
và limit. Memory request của Pod là tổng các memory request của tất cả Container trong Pod.
Tương tự, memory limit của Pod là tổng các limit của tất cả Container trong Pod.

Việc lập lịch (scheduling) Pod dựa trên request. Một Pod chỉ được lập lịch chạy trên một Node
nếu Node đó có đủ memory khả dụng để đáp ứng memory request của Pod.

Trong bài thực hành này, bạn tạo một Pod có memory request lớn đến mức vượt quá dung lượng của
bất kỳ Node nào trong cluster. Đây là file cấu hình cho một Pod có một Container với request
1000 GiB memory, nhiều khả năng vượt quá dung lượng của bất kỳ Node nào trong cluster của bạn.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-3
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-3-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "1000Gi"
      limits:
        memory: "1000Gi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit-3.yaml --namespace=mem-example
```

Xem trạng thái của Pod:

```shell
kubectl get pod memory-demo-3 --namespace=mem-example
```

Kết quả cho thấy trạng thái của Pod là PENDING. Nghĩa là Pod không được lập lịch chạy trên bất
kỳ Node nào, và nó sẽ ở trạng thái PENDING vô thời hạn:

```
kubectl get pod memory-demo-3 --namespace=mem-example
NAME            READY     STATUS    RESTARTS   AGE
memory-demo-3   0/1       Pending   0          25s
```

Xem thông tin chi tiết về Pod, bao gồm các sự kiện (event):

```shell
kubectl describe pod memory-demo-3 --namespace=mem-example
```

Kết quả cho thấy Container không thể được lập lịch vì các Node không đủ memory:

```
Events:
  ...  Reason            Message
       ------            -------
  ...  FailedScheduling  No nodes are available that match all of the following predicates:: Insufficient memory (3).
```

## Đơn vị memory (Memory units)

Tài nguyên memory được đo bằng byte. Bạn có thể biểu diễn memory dưới dạng số nguyên thuần hoặc
số nguyên định điểm (fixed-point) kèm một trong các hậu tố sau: E, P, T, G, M, K, Ei, Pi, Ti,
Gi, Mi, Ki. Ví dụ, các giá trị sau biểu diễn xấp xỉ cùng một giá trị:

```
128974848, 129e6, 129M, 123Mi
```

Xóa Pod của bạn:

```shell
kubectl delete pod memory-demo-3 --namespace=mem-example
```

## Nếu bạn không chỉ định memory limit (If you do not specify a memory limit)

Nếu bạn không chỉ định memory limit cho một Container, một trong các tình huống sau sẽ xảy ra:

* Container không có giới hạn trên đối với lượng memory nó sử dụng. Container có thể sử dụng
  toàn bộ memory khả dụng trên Node nơi nó đang chạy, điều này đến lượt nó có thể kích hoạt OOM
  Killer. Hơn nữa, trong trường hợp xảy ra OOM Kill, container không có giới hạn tài nguyên sẽ
  có khả năng bị kill cao hơn.

* Container đang chạy trong một namespace có memory limit mặc định, và Container được tự động
  gán limit mặc định đó. Người quản trị cluster có thể dùng
  [LimitRange](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#limitrange-v1-core)
  để chỉ định giá trị mặc định cho memory limit.

## Động lực cho memory request và limit (Motivation for memory requests and limits)

Bằng cách cấu hình memory request và limit cho các Container chạy trong cluster, bạn có thể sử
dụng hiệu quả tài nguyên memory khả dụng trên các Node của cluster. Bằng cách giữ memory request
của Pod ở mức thấp, bạn tăng cơ hội để Pod được lập lịch. Bằng cách có memory limit lớn hơn
memory request, bạn đạt được hai điều:

* Pod có thể có những đợt hoạt động bùng nổ (burst) tận dụng lượng memory đang rảnh.
* Lượng memory mà Pod có thể dùng trong một đợt bùng nổ được giới hạn ở một mức hợp lý.

## Dọn dẹp (Clean up)

Xóa namespace của bạn. Thao tác này xóa tất cả các Pod bạn đã tạo cho tác vụ này:

```shell
kubectl delete namespace mem-example
```

## Tiếp theo (What's next)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên CPU cho Container và Pod (Assign CPU Resources to Containers and Pods)](./263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và memory ở cấp Pod (Assign Pod-level CPU and memory resources)](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pod-level-resources/)

* [Cấu hình Quality of Service cho Pod (Configure Quality of Service for Pods)](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)

* [Thay đổi tài nguyên CPU và memory đã gán cho Container (Resize CPU and Memory Resources assigned to Containers)](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)

### Dành cho người quản trị cluster (For cluster administrators)

* [Cấu hình memory request và limit mặc định cho một Namespace (Configure Default Memory Requests and Limits for a Namespace)](./232-memory-default-namespace-vi.md)

* [Cấu hình CPU request và limit mặc định cho một Namespace (Configure Default CPU Requests and Limits for a Namespace)](./230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc memory tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum Memory Constraints for a Namespace)](./231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum CPU Constraints for a Namespace)](./229-cpu-constraint-namespace-vi.md)

* [Cấu hình quota memory và CPU cho một Namespace (Configure Memory and CPU Quotas for a Namespace)](./233-quota-memory-cpu-namespace-vi.md)

* [Cấu hình quota Pod cho một Namespace (Configure a Pod Quota for a Namespace)](./234-quota-pod-namespace-vi.md)

* [Cấu hình quota cho các đối tượng API (Configure Quotas for API Objects)](./252-quota-api-object-vi.md)

* [Thay đổi tài nguyên CPU và memory đã gán cho Container (Resize CPU and Memory Resources assigned to Containers)](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)
