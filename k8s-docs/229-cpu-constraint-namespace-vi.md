# Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum CPU Constraints for a Namespace)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/>
>
> Định nghĩa một khoảng giá trị CPU resource limit hợp lệ cho một namespace, để mọi Pod mới
> trong namespace đó nằm trong khoảng mà bạn cấu hình.

Trang này chỉ ra cách đặt giá trị tối thiểu và tối đa cho tài nguyên CPU mà các container và
Pod trong một namespace được sử dụng. Bạn chỉ định các giá trị CPU tối thiểu và tối đa trong
một object
[LimitRange](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/limit-range-v1/).
Nếu một Pod không đáp ứng các ràng buộc mà LimitRange áp đặt, nó không thể được tạo trong
namespace đó.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn phải có quyền tạo namespace trong cluster của mình.

Mỗi node trong cluster của bạn phải có ít nhất 1.0 CPU khả dụng cho các Pod.
Xem [ý nghĩa của CPU](110-manage-resources-containers-vi.md#meaning-of-cpu)
để biết Kubernetes hiểu "1 CPU" nghĩa là gì.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi phần còn
lại của cluster.

```shell
kubectl create namespace constraints-cpu-example
```

## Tạo một LimitRange và một Pod (Create a LimitRange and a Pod)

Đây là manifest cho một LimitRange mẫu:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-min-max-demo-lr
spec:
  limits:
  - max:
      cpu: "800m"
    min:
      cpu: "200m"
    type: Container
```

Tạo LimitRange:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-constraints.yaml --namespace=constraints-cpu-example
```

Xem thông tin chi tiết về LimitRange:

```shell
kubectl get limitrange cpu-min-max-demo-lr --output=yaml --namespace=constraints-cpu-example
```

Output cho thấy các ràng buộc CPU tối thiểu và tối đa đúng như mong đợi. Nhưng hãy để ý rằng
mặc dù bạn không chỉ định các giá trị mặc định trong file cấu hình của LimitRange, chúng vẫn
được tạo tự động.

```yaml
limits:
- default:
    cpu: 800m
  defaultRequest:
    cpu: 800m
  max:
    cpu: 800m
  min:
    cpu: 200m
  type: Container
```

Bây giờ, mỗi khi bạn tạo một Pod trong namespace constraints-cpu-example (hoặc một client
khác của Kubernetes API tạo một Pod tương đương), Kubernetes thực hiện các bước sau:

* Nếu bất kỳ container nào trong Pod đó không chỉ định CPU request và limit của riêng nó,
  control plane sẽ gán CPU request và limit mặc định cho container đó.

* Xác minh rằng mọi container trong Pod đó chỉ định một CPU request lớn hơn hoặc bằng 200
  millicpu.

* Xác minh rằng mọi container trong Pod đó chỉ định một CPU limit nhỏ hơn hoặc bằng 800
  millicpu.

> **Ghi chú:**
>
> Khi tạo một object `LimitRange`, bạn cũng có thể chỉ định giới hạn cho huge-pages hoặc GPU.
> Tuy nhiên, khi cả `default` và `defaultRequest` đều được chỉ định trên các tài nguyên này,
> hai giá trị đó phải bằng nhau.

Đây là manifest cho một Pod có một container. Manifest của container chỉ định CPU request là
500 millicpu và CPU limit là 800 millicpu. Các giá trị này thỏa mãn ràng buộc CPU tối thiểu
và tối đa mà LimitRange áp đặt cho namespace này.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo
spec:
  containers:
  - name: constraints-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        cpu: "800m"
      requests:
        cpu: "500m"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-constraints-pod.yaml --namespace=constraints-cpu-example
```

Xác minh rằng Pod đang chạy và container của nó khỏe mạnh:

```shell
kubectl get pod constraints-cpu-demo --namespace=constraints-cpu-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod constraints-cpu-demo --output=yaml --namespace=constraints-cpu-example
```

Output cho thấy container duy nhất của Pod có CPU request là 500 millicpu và CPU limit là 800
millicpu. Các giá trị này thỏa mãn các ràng buộc mà LimitRange áp đặt.

```yaml
resources:
  limits:
    cpu: 800m
  requests:
    cpu: 500m
```

## Xóa Pod (Delete the Pod)

```shell
kubectl delete pod constraints-cpu-demo --namespace=constraints-cpu-example
```

## Thử tạo một Pod vượt quá ràng buộc CPU tối đa (Attempt to create a Pod that exceeds the maximum CPU constraint)

Đây là manifest cho một Pod có một container. Container này chỉ định CPU request là 500
millicpu và CPU limit là 1.5 cpu.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo-2
spec:
  containers:
  - name: constraints-cpu-demo-2-ctr
    image: nginx
    resources:
      limits:
        cpu: "1.5"
      requests:
        cpu: "500m"
```

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-constraints-pod-2.yaml --namespace=constraints-cpu-example
```

Output cho thấy Pod không được tạo, vì nó định nghĩa một container không được chấp nhận.
Container đó không được chấp nhận vì nó chỉ định một CPU limit quá lớn:

```
Error from server (Forbidden): error when creating "examples/admin/resource/cpu-constraints-pod-2.yaml":
pods "constraints-cpu-demo-2" is forbidden: maximum cpu usage per Container is 800m, but limit is 1500m.
```

## Thử tạo một Pod không đạt mức CPU request tối thiểu (Attempt to create a Pod that does not meet the minimum CPU request)

Đây là manifest cho một Pod có một container. Container này chỉ định CPU request là 100
millicpu và CPU limit là 800 millicpu.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo-3
spec:
  containers:
  - name: constraints-cpu-demo-3-ctr
    image: nginx
    resources:
      limits:
        cpu: "800m"
      requests:
        cpu: "100m"
```

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-constraints-pod-3.yaml --namespace=constraints-cpu-example
```

Output cho thấy Pod không được tạo, vì nó định nghĩa một container không được chấp nhận.
Container đó không được chấp nhận vì nó chỉ định một CPU request thấp hơn mức tối thiểu đang
được thực thi:

```
Error from server (Forbidden): error when creating "examples/admin/resource/cpu-constraints-pod-3.yaml":
pods "constraints-cpu-demo-3" is forbidden: minimum cpu usage per Container is 200m, but request is 100m.
```

## Tạo một Pod không chỉ định bất kỳ CPU request hay limit nào (Create a Pod that does not specify any CPU request or limit)

Đây là manifest cho một Pod có một container. Container này không chỉ định CPU request, cũng
không chỉ định CPU limit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-cpu-demo-4
spec:
  containers:
  - name: constraints-cpu-demo-4-ctr
    image: vish/stress
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-constraints-pod-4.yaml --namespace=constraints-cpu-example
```

Xem thông tin chi tiết về Pod:

```
kubectl get pod constraints-cpu-demo-4 --namespace=constraints-cpu-example --output=yaml
```

Output cho thấy container duy nhất của Pod có CPU request là 800 millicpu và CPU limit là 800
millicpu. Làm thế nào container đó nhận được các giá trị này?

```yaml
resources:
  limits:
    cpu: 800m
  requests:
    cpu: 800m
```

Bởi vì container đó không chỉ định CPU request và limit của riêng nó, control plane đã áp
dụng
[CPU request và limit mặc định](230-cpu-default-namespace-vi.md)
từ LimitRange của namespace này.

Tại thời điểm này, Pod của bạn có thể đang chạy hoặc không. Hãy nhớ lại rằng một điều kiện
tiên quyết của tác vụ này là các Node của bạn phải có ít nhất 1 CPU khả dụng để sử dụng. Nếu
mỗi Node của bạn chỉ có 1 CPU, thì có thể không Node nào có đủ CPU cấp phát được (allocatable)
để đáp ứng một request 800 millicpu. Nếu bạn đang dùng các Node có 2 CPU, thì bạn có lẽ có đủ
CPU để đáp ứng request 800 millicpu.

Xóa Pod:

```
kubectl delete pod constraints-cpu-demo-4 --namespace=constraints-cpu-example
```

## Việc thực thi các ràng buộc CPU tối thiểu và tối đa (Enforcement of minimum and maximum CPU constraints)

Các ràng buộc CPU tối đa và tối thiểu mà một LimitRange áp đặt lên một namespace chỉ được
thực thi khi một Pod được tạo hoặc cập nhật. Nếu bạn thay đổi LimitRange, nó không ảnh hưởng
tới các Pod đã được tạo trước đó.

> **Ghi chú:**
>
> Khi sử dụng [thay đổi kích thước tài nguyên Pod tại chỗ (in-place Pod resize)](289-resize-container-resources-vi.md),
> các ràng buộc CPU cũng được thực thi. Nếu một lần resize làm cho các giá trị CPU của Pod vi
> phạm ràng buộc của LimitRange (vượt quá mức tối đa hoặc xuống dưới mức tối thiểu), lần
> resize đó sẽ bị từ chối và tài nguyên của Pod giữ nguyên các giá trị trước đó.

## Động cơ cho ràng buộc CPU tối thiểu và tối đa (Motivation for minimum and maximum CPU constraints)

Với vai trò người quản trị cluster, bạn có thể muốn áp đặt các hạn chế lên tài nguyên CPU mà
các Pod được sử dụng. Ví dụ:

* Mỗi Node trong cluster có 2 CPU. Bạn không muốn chấp nhận bất kỳ Pod nào yêu cầu nhiều hơn
  2 CPU, vì không Node nào trong cluster có thể đáp ứng request đó.

* Một cluster được chia sẻ giữa bộ phận production và bộ phận development của bạn. Bạn muốn
  cho phép các workload production tiêu thụ tới 3 CPU, nhưng muốn các workload development bị
  giới hạn ở 1 CPU. Bạn tạo các namespace riêng cho production và development, và áp dụng
  ràng buộc CPU cho từng namespace.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace constraints-cpu-example
```

## Tiếp theo (What's next)

### Dành cho người quản trị cluster (For cluster administrators)

* [Cấu hình memory request và limit mặc định cho một Namespace](232-memory-default-namespace-vi.md)

* [Cấu hình CPU request và limit mặc định cho một Namespace](230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc memory tối thiểu và tối đa cho một Namespace](231-memory-constraint-namespace-vi.md)

* [Cấu hình quota memory và CPU cho một Namespace](233-quota-memory-cpu-namespace-vi.md)

* [Cấu hình quota Pod cho một Namespace](234-quota-pod-namespace-vi.md)

* [Cấu hình quota cho các API object](252-quota-api-object-vi.md)

### Dành cho lập trình viên ứng dụng (For app developers)

* [Gán tài nguyên memory cho container và Pod](264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho container và Pod](263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và memory ở cấp Pod](265-assign-pod-level-resources-vi.md)

* [Cấu hình Quality of Service cho Pod](288-quality-service-pod-vi.md)
