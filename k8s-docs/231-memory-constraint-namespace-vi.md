# Cấu hình ràng buộc bộ nhớ tối thiểu và tối đa cho một Namespace (Configure Minimum and Maximum Memory Constraints for a Namespace)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/>
>
> Định nghĩa một khoảng giá trị hợp lệ cho giới hạn tài nguyên bộ nhớ trong một namespace,
> để mọi Pod mới trong namespace đó đều nằm trong khoảng bạn cấu hình.

Trang này hướng dẫn cách đặt giá trị tối thiểu và tối đa cho lượng bộ nhớ mà các container
chạy trong một namespace được sử dụng. Bạn chỉ định các giá trị bộ nhớ tối thiểu và tối đa
trong một object
[LimitRange](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/limit-range-v1/).
Nếu một Pod không thỏa mãn các ràng buộc mà LimitRange áp đặt, nó sẽ không thể được tạo
trong namespace đó.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn phải có quyền tạo namespace trong cluster của mình.

Mỗi node trong cluster của bạn phải có ít nhất 1 GiB bộ nhớ khả dụng cho các Pod.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi
phần còn lại của cluster.

```shell
kubectl create namespace constraints-mem-example
```

## Tạo một LimitRange và một Pod (Create a LimitRange and a Pod)

Dưới đây là manifest ví dụ cho một LimitRange:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-min-max-demo-lr
spec:
  limits:
  - max:
      memory: 1Gi
    min:
      memory: 500Mi
    type: Container
```

Tạo LimitRange:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-constraints.yaml --namespace=constraints-mem-example
```

Xem thông tin chi tiết về LimitRange:

```shell
kubectl get limitrange mem-min-max-demo-lr --namespace=constraints-mem-example --output=yaml
```

Output hiển thị các ràng buộc bộ nhớ tối thiểu và tối đa đúng như mong đợi. Nhưng hãy để ý
rằng mặc dù bạn không chỉ định giá trị mặc định trong file cấu hình của LimitRange,
chúng vẫn được tạo tự động.

```
  limits:
  - default:
      memory: 1Gi
    defaultRequest:
      memory: 1Gi
    max:
      memory: 1Gi
    min:
      memory: 500Mi
    type: Container
```

Bây giờ, mỗi khi bạn định nghĩa một Pod trong namespace constraints-mem-example, Kubernetes
sẽ thực hiện các bước sau:

* Nếu bất kỳ container nào trong Pod đó không chỉ định memory request và limit của riêng nó,
  control plane sẽ gán memory request và limit mặc định cho container đó.

* Xác minh rằng mọi container trong Pod đó yêu cầu (request) ít nhất 500 MiB bộ nhớ.

* Xác minh rằng mọi container trong Pod đó yêu cầu không quá 1024 MiB (1 GiB) bộ nhớ.

Dưới đây là manifest cho một Pod có một container. Trong spec của Pod, container duy nhất
chỉ định memory request là 600 MiB và memory limit là 800 MiB. Các giá trị này thỏa mãn
ràng buộc bộ nhớ tối thiểu và tối đa mà LimitRange áp đặt.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-mem-demo
spec:
  containers:
  - name: constraints-mem-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
      requests:
        memory: "600Mi"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-constraints-pod.yaml --namespace=constraints-mem-example
```

Xác minh rằng Pod đang chạy và container của nó hoạt động bình thường:

```shell
kubectl get pod constraints-mem-demo --namespace=constraints-mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod constraints-mem-demo --output=yaml --namespace=constraints-mem-example
```

Output cho thấy container trong Pod đó có memory request là 600 MiB và memory limit là 800 MiB.
Các giá trị này thỏa mãn các ràng buộc mà LimitRange áp đặt cho namespace này:

```yaml
resources:
  limits:
     memory: 800Mi
  requests:
    memory: 600Mi
```

Xóa Pod của bạn:

```shell
kubectl delete pod constraints-mem-demo --namespace=constraints-mem-example
```

## Thử tạo một Pod vượt quá ràng buộc bộ nhớ tối đa (Attempt to create a Pod that exceeds the maximum memory constraint)

Dưới đây là manifest cho một Pod có một container. Container này chỉ định memory request
là 800 MiB và memory limit là 1.5 GiB.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-mem-demo-2
spec:
  containers:
  - name: constraints-mem-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "1.5Gi"
      requests:
        memory: "800Mi"
```

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-constraints-pod-2.yaml --namespace=constraints-mem-example
```

Output cho thấy Pod không được tạo, vì nó định nghĩa một container yêu cầu nhiều bộ nhớ hơn
mức cho phép:

```
Error from server (Forbidden): error when creating "examples/admin/resource/memory-constraints-pod-2.yaml":
pods "constraints-mem-demo-2" is forbidden: maximum memory usage per Container is 1Gi, but limit is 1536Mi.
```

## Thử tạo một Pod không đạt mức memory request tối thiểu (Attempt to create a Pod that does not meet the minimum memory request)

Dưới đây là manifest cho một Pod có một container. Container đó chỉ định memory request
là 100 MiB và memory limit là 800 MiB.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-mem-demo-3
spec:
  containers:
  - name: constraints-mem-demo-3-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
      requests:
        memory: "100Mi"
```

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-constraints-pod-3.yaml --namespace=constraints-mem-example
```

Output cho thấy Pod không được tạo, vì nó định nghĩa một container yêu cầu ít bộ nhớ hơn
mức tối thiểu bắt buộc:

```
Error from server (Forbidden): error when creating "examples/admin/resource/memory-constraints-pod-3.yaml":
pods "constraints-mem-demo-3" is forbidden: minimum memory usage per Container is 500Mi, but request is 100Mi.
```

## Tạo một Pod không chỉ định memory request hay limit nào (Create a Pod that does not specify any memory request or limit)

Dưới đây là manifest cho một Pod có một container. Container này không chỉ định memory request,
và cũng không chỉ định memory limit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constraints-mem-demo-4
spec:
  containers:
  - name: constraints-mem-demo-4-ctr
    image: nginx
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-constraints-pod-4.yaml --namespace=constraints-mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod constraints-mem-demo-4 --namespace=constraints-mem-example --output=yaml
```

Output cho thấy container duy nhất của Pod có memory request là 1 GiB và memory limit là 1 GiB.
Làm thế nào container đó có được những giá trị này?

```
resources:
  limits:
    memory: 1Gi
  requests:
    memory: 1Gi
```

Vì Pod của bạn không định nghĩa memory request và limit nào cho container đó, cluster đã
áp dụng
[memory request và limit mặc định](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)
từ LimitRange.

Điều này có nghĩa là định nghĩa của Pod đó sẽ hiển thị các giá trị này. Bạn có thể kiểm tra
bằng `kubectl describe`:

```shell
# Tìm mục "Requests:" trong output
kubectl describe pod constraints-mem-demo-4 --namespace=constraints-mem-example
```

Tại thời điểm này, Pod của bạn có thể đang chạy hoặc không chạy. Hãy nhớ rằng điều kiện
tiên quyết của bài này là các Node của bạn có ít nhất 1 GiB bộ nhớ. Nếu mỗi Node của bạn chỉ
có 1 GiB bộ nhớ, thì không Node nào có đủ bộ nhớ cấp phát được (allocatable) để đáp ứng một
memory request 1 GiB. Nếu bạn đang dùng các Node có 2 GiB bộ nhớ, thì có lẽ bạn có đủ chỗ
để đáp ứng request 1 GiB đó.

Xóa Pod của bạn:

```shell
kubectl delete pod constraints-mem-demo-4 --namespace=constraints-mem-example
```

## Thực thi các ràng buộc bộ nhớ tối thiểu và tối đa (Enforcement of minimum and maximum memory constraints)

Các ràng buộc bộ nhớ tối đa và tối thiểu mà một LimitRange áp đặt lên một namespace chỉ được
thực thi khi một Pod được tạo hoặc cập nhật. Nếu bạn thay đổi LimitRange, điều đó không ảnh
hưởng đến các Pod đã được tạo trước đó.

> **Ghi chú:**
> Khi sử dụng [thay đổi kích thước Pod tại chỗ (in-place Pod resize)](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/),
> các ràng buộc bộ nhớ cũng được thực thi. Nếu một lần resize khiến các giá trị bộ nhớ của Pod
> vi phạm ràng buộc của LimitRange (vượt quá mức tối đa hoặc thấp hơn mức tối thiểu),
> lần resize đó sẽ bị từ chối và tài nguyên của Pod giữ nguyên các giá trị trước đó.

## Động lực cho ràng buộc bộ nhớ tối thiểu và tối đa (Motivation for minimum and maximum memory constraints)

Với vai trò quản trị viên cluster, bạn có thể muốn áp đặt các giới hạn lên lượng bộ nhớ mà
các Pod được sử dụng. Ví dụ:

* Mỗi Node trong cluster có 2 GiB bộ nhớ. Bạn không muốn chấp nhận bất kỳ Pod nào yêu cầu
  hơn 2 GiB bộ nhớ, vì không Node nào trong cluster có thể đáp ứng yêu cầu đó.

* Một cluster được dùng chung bởi bộ phận production và bộ phận development của bạn.
  Bạn muốn cho phép các workload production sử dụng tối đa 8 GiB bộ nhớ, nhưng muốn giới hạn
  các workload development ở mức 512 MiB. Bạn tạo các namespace riêng cho production và
  development, và áp dụng ràng buộc bộ nhớ cho từng namespace.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace constraints-mem-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình memory request và limit mặc định cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)

* [Cấu hình CPU request và limit mặc định cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)

* [Cấu hình hạn ngạch bộ nhớ và CPU cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)

* [Cấu hình hạn ngạch Pod cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-pod-namespace/)

* [Cấu hình hạn ngạch cho các API Object](https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên bộ nhớ cho Container và Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)

* [Gán tài nguyên CPU cho Container và Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)

* [Gán tài nguyên CPU và bộ nhớ ở cấp Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pod-level-resources/)

* [Cấu hình Quality of Service cho Pod](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
