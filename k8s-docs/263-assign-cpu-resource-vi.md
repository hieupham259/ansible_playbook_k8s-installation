# Gán tài nguyên CPU cho Container và Pod (Assign CPU Resources to Containers and Pods)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/>

Trang này hướng dẫn cách gán một *request* (yêu cầu) CPU và một *limit* (giới hạn) CPU cho
container. Container không thể sử dụng nhiều CPU hơn limit đã cấu hình. Miễn là hệ thống còn
thời gian CPU rảnh, container được đảm bảo nhận đủ lượng CPU mà nó yêu cầu.

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

Cluster của bạn phải có ít nhất 1 CPU sẵn sàng để sử dụng nhằm chạy các ví dụ trong tác vụ này.

Một vài bước trên trang này yêu cầu bạn chạy dịch vụ
[metrics-server](https://github.com/kubernetes-sigs/metrics-server) trong cluster. Nếu bạn đã có
metrics-server đang chạy, bạn có thể bỏ qua các bước đó.

Nếu bạn đang chạy minikube, hãy chạy lệnh sau để bật metrics-server:

```shell
minikube addons enable metrics-server
```

Để xem metrics-server (hoặc một trình cung cấp khác của resource metrics API, `metrics.k8s.io`)
có đang chạy hay không, hãy nhập lệnh sau:

```shell
kubectl get apiservices
```

Nếu resource metrics API khả dụng, kết quả sẽ bao gồm một tham chiếu đến `metrics.k8s.io`.

```
NAME
v1beta1.metrics.k8s.io
```

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi phần còn
lại của cluster.

```shell
kubectl create namespace cpu-example
```

## Chỉ định CPU request và CPU limit (Specify a CPU request and a CPU limit)

Để chỉ định CPU request cho một container, hãy thêm field `resources.requests.cpu` vào manifest
tài nguyên của container. Để chỉ định CPU limit, hãy thêm `resources.limits.cpu`.

Trong bài thực hành này, bạn tạo một Pod có một container. Container có request là 0.5 CPU và
limit là 1 CPU. Đây là file cấu hình của Pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo
  namespace: cpu-example
spec:
  containers:
  - name: cpu-demo-ctr
    image: vish/stress
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: "0.5"
    args:
    - -cpus
    - "2"
```

Mục `args` của file cấu hình cung cấp các đối số cho container khi nó khởi động. Đối số
`-cpus "2"` yêu cầu Container cố gắng sử dụng 2 CPU.

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/cpu-request-limit.yaml --namespace=cpu-example
```

Xác minh rằng Pod đang chạy:

```shell
kubectl get pod cpu-demo --namespace=cpu-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod cpu-demo --output=yaml --namespace=cpu-example
```

Kết quả cho thấy container duy nhất trong Pod có CPU request là 500 milliCPU và CPU limit là
1 CPU.

```yaml
resources:
  limits:
    cpu: "1"
  requests:
    cpu: 500m
```

Dùng `kubectl top` để lấy các số liệu (metrics) của Pod:

```shell
kubectl top pod cpu-demo --namespace=cpu-example
```

Kết quả ví dụ này cho thấy Pod đang sử dụng 974 milliCPU, thấp hơn một chút so với limit 1 CPU
được chỉ định trong cấu hình của Pod.

```
NAME                        CPU(cores)   MEMORY(bytes)
cpu-demo                    974m         <something>
```

Hãy nhớ rằng bằng cách đặt `-cpu "2"`, bạn đã cấu hình Container cố gắng sử dụng 2 CPU, nhưng
Container chỉ được phép sử dụng khoảng 1 CPU. Mức sử dụng CPU của container đang bị điều tiết
(throttle), vì container đang cố gắng sử dụng nhiều tài nguyên CPU hơn limit của nó.

> **Ghi chú:**
> Một cách giải thích khả dĩ khác cho việc mức sử dụng CPU thấp hơn 1.0 là Node có thể không có
> đủ tài nguyên CPU khả dụng. Hãy nhớ rằng điều kiện tiên quyết của bài thực hành này yêu cầu
> cluster của bạn có ít nhất 1 CPU sẵn sàng để sử dụng. Nếu Container của bạn chạy trên một Node
> chỉ có 1 CPU, Container không thể sử dụng nhiều hơn 1 CPU bất kể CPU limit được chỉ định cho
> Container là bao nhiêu.

## Đơn vị CPU (CPU units)

Tài nguyên CPU được đo bằng đơn vị *CPU*. Một CPU, trong Kubernetes, tương đương với:

* 1 AWS vCPU
* 1 GCP Core
* 1 Azure vCore
* 1 Hyperthread trên một bộ xử lý Intel bare-metal có Hyperthreading

Giá trị thập phân được cho phép. Một Container yêu cầu 0.5 CPU được đảm bảo lượng CPU bằng một
nửa so với Container yêu cầu 1 CPU. Bạn có thể dùng hậu tố m với nghĩa milli. Ví dụ 100m CPU,
100 milliCPU và 0.1 CPU đều là một. Không cho phép độ chính xác nhỏ hơn 1m.

CPU luôn được yêu cầu như một lượng tuyệt đối, không bao giờ là lượng tương đối; 0.1 là cùng một
lượng CPU trên máy một nhân, hai nhân hay 48 nhân.

Xóa Pod của bạn:

```shell
kubectl delete pod cpu-demo --namespace=cpu-example
```

## Chỉ định một CPU request quá lớn so với các Node của bạn (Specify a CPU request that is too big for your Nodes)

CPU request và limit gắn với Container, nhưng sẽ hữu ích khi coi Pod cũng có CPU request và
limit. CPU request của một Pod là tổng các CPU request của tất cả Container trong Pod. Tương tự,
CPU limit của một Pod là tổng các CPU limit của tất cả Container trong Pod.

Việc lập lịch (scheduling) Pod dựa trên request. Một Pod chỉ được lập lịch chạy trên một Node
nếu Node đó có đủ tài nguyên CPU khả dụng để đáp ứng CPU request của Pod.

Trong bài thực hành này, bạn tạo một Pod có CPU request lớn đến mức vượt quá dung lượng của bất
kỳ Node nào trong cluster. Đây là file cấu hình cho một Pod có một Container. Container yêu cầu
100 CPU, nhiều khả năng vượt quá dung lượng của bất kỳ Node nào trong cluster của bạn.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo-2
  namespace: cpu-example
spec:
  containers:
  - name: cpu-demo-ctr-2
    image: vish/stress
    resources:
      limits:
        cpu: "100"
      requests:
        cpu: "100"
    args:
    - -cpus
    - "2"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/cpu-request-limit-2.yaml --namespace=cpu-example
```

Xem trạng thái của Pod:

```shell
kubectl get pod cpu-demo-2 --namespace=cpu-example
```

Kết quả cho thấy trạng thái của Pod là Pending. Nghĩa là Pod chưa được lập lịch chạy trên bất
kỳ Node nào, và nó sẽ ở trạng thái Pending vô thời hạn:

```
NAME         READY     STATUS    RESTARTS   AGE
cpu-demo-2   0/1       Pending   0          7m
```

Xem thông tin chi tiết về Pod, bao gồm các sự kiện (event):

```shell
kubectl describe pod cpu-demo-2 --namespace=cpu-example
```

Kết quả cho thấy Container không thể được lập lịch vì các Node không đủ tài nguyên CPU:

```
Events:
  Reason                        Message
  ------                        -------
  FailedScheduling      No nodes are available that match all of the following predicates:: Insufficient cpu (3).
```

Xóa Pod của bạn:

```shell
kubectl delete pod cpu-demo-2 --namespace=cpu-example
```

## Nếu bạn không chỉ định CPU limit (If you do not specify a CPU limit)

Nếu bạn không chỉ định CPU limit cho một Container, thì một trong các tình huống sau sẽ xảy ra:

* Container không có giới hạn trên đối với lượng tài nguyên CPU nó có thể dùng. Container có thể
  sử dụng toàn bộ tài nguyên CPU khả dụng trên Node nơi nó đang chạy.

* Container đang chạy trong một namespace có CPU limit mặc định, và Container được tự động gán
  limit mặc định đó. Người quản trị cluster có thể dùng
  [LimitRange](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#limitrange-v1-core/)
  để chỉ định giá trị mặc định cho CPU limit.

## Nếu bạn chỉ định CPU limit nhưng không chỉ định CPU request (If you specify a CPU limit but do not specify a CPU request)

Nếu bạn chỉ định CPU limit cho một Container nhưng không chỉ định CPU request, Kubernetes sẽ tự
động gán một CPU request bằng với limit. Tương tự, nếu một Container chỉ định memory limit của
riêng nó nhưng không chỉ định memory request, Kubernetes sẽ tự động gán một memory request bằng
với limit.

## Động lực cho CPU request và limit (Motivation for CPU requests and limits)

Bằng cách cấu hình CPU request và limit cho các Container chạy trong cluster, bạn có thể sử dụng
hiệu quả tài nguyên CPU khả dụng trên các Node của cluster. Bằng cách giữ CPU request của Pod ở
mức thấp, bạn tăng cơ hội để Pod được lập lịch. Bằng cách có CPU limit lớn hơn CPU request, bạn
đạt được hai điều:

* Pod có thể có những đợt hoạt động bùng nổ (burst) tận dụng tài nguyên CPU đang rảnh.
* Lượng tài nguyên CPU mà Pod có thể dùng trong một đợt bùng nổ được giới hạn ở một mức hợp lý.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace cpu-example
```

## Tiếp theo (What's next)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên memory cho Container và Pod (Assign Memory Resources to Containers and Pods)](./264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU và memory ở cấp Pod (Assign Pod-level CPU and memory resources)](265-assign-pod-level-resources-vi.md)

* [Cấu hình Quality of Service cho Pod (Configure Quality of Service for Pods)](288-quality-service-pod-vi.md)

* [Thay đổi tài nguyên CPU và memory đã gán cho Container (Resize CPU and Memory Resources assigned to Containers)](289-resize-container-resources-vi.md)

* [Thay đổi tài nguyên CPU và memory ở cấp Pod (Resize Pod-level CPU and Memory Resources)](290-resize-pod-resources-vi.md)

### Dành cho người quản trị cluster (For cluster administrators)

* [Cấu hình memory request và limit mặc định cho một Namespace (Configure Default Memory Requests and Limits for a Namespace)](./232-memory-default-namespace-vi.md)

* [Cấu hình CPU request và limit mặc định cho một Namespace (Configure Default CPU Requests and Limits for a Namespace)](./230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc memory tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum Memory Constraints for a Namespace)](./231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum CPU Constraints for a Namespace)](./229-cpu-constraint-namespace-vi.md)

* [Cấu hình quota memory và CPU cho một Namespace (Configure Memory and CPU Quotas for a Namespace)](./233-quota-memory-cpu-namespace-vi.md)

* [Cấu hình quota Pod cho một Namespace (Configure a Pod Quota for a Namespace)](./234-quota-pod-namespace-vi.md)

* [Cấu hình quota cho các đối tượng API (Configure Quotas for API Objects)](./252-quota-api-object-vi.md)

* [Thay đổi tài nguyên CPU và memory đã gán cho Container (Resize CPU and Memory Resources assigned to Containers)](289-resize-container-resources-vi.md)
