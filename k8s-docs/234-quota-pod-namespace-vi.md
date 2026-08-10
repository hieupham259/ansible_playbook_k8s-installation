# Cấu hình Quota số lượng Pod cho một Namespace (Configure a Pod Quota for a Namespace)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-pod-namespace/
>
> Hạn chế số lượng Pod bạn có thể tạo trong một namespace.

Trang này hướng dẫn cách đặt hạn ngạch (quota) cho tổng số Pod có thể chạy trong một Namespace.
Bạn chỉ định quota trong một đối tượng
[ResourceQuota](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/resource-quota-v1/).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, nhập `kubectl version`.

Bạn phải có quyền tạo namespace trong cluster của mình.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi phần còn
lại của cluster.

```shell
kubectl create namespace quota-pod-example
```

## Tạo một ResourceQuota (Create a ResourceQuota)

Dưới đây là manifest mẫu cho một ResourceQuota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-demo
spec:
  hard:
    pods: "2"
```

Tạo ResourceQuota:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-pod.yaml --namespace=quota-pod-example
```

Xem thông tin chi tiết về ResourceQuota:

```shell
kubectl get resourcequota pod-demo --namespace=quota-pod-example --output=yaml
```

Kết quả cho thấy namespace có quota là hai Pod, và hiện tại chưa có Pod nào; nghĩa là chưa phần
nào của quota được sử dụng.

```yaml
spec:
  hard:
    pods: "2"
status:
  hard:
    pods: "2"
  used:
    pods: "0"
```

Dưới đây là manifest mẫu cho một Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-quota-demo
spec:
  selector:
    matchLabels:
      purpose: quota-demo
  replicas: 3
  template:
    metadata:
      labels:
        purpose: quota-demo
    spec:
      containers:
      - name: pod-quota-demo
        image: nginx
```

Trong manifest đó, `replicas: 3` yêu cầu Kubernetes cố gắng tạo ba Pod mới, tất cả cùng chạy
một ứng dụng.

Tạo Deployment:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-pod-deployment.yaml --namespace=quota-pod-example
```

Xem thông tin chi tiết về Deployment:

```shell
kubectl get deployment pod-quota-demo --namespace=quota-pod-example --output=yaml
```

Kết quả cho thấy mặc dù Deployment chỉ định ba replica, chỉ có hai Pod được tạo do quota mà bạn
đã định nghĩa trước đó:

```yaml
spec:
  ...
  replicas: 3
...
status:
  availableReplicas: 2
...
lastUpdateTime: 2021-04-02T20:57:05Z
    message: 'unable to create pods: pods "pod-quota-demo-1650323038-" is forbidden:
      exceeded quota: pod-demo, requested: pods=1, used: pods=2, limited: pods=2'
```

### Lựa chọn loại tài nguyên (Choice of resource)

Trong bài này bạn đã định nghĩa một ResourceQuota giới hạn tổng số Pod, nhưng bạn cũng có thể
giới hạn tổng số của các loại đối tượng khác. Ví dụ, bạn có thể quyết định giới hạn số lượng
CronJob được phép tồn tại trong một namespace.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace quota-pod-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình Memory Request và Limit mặc định cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)

* [Cấu hình CPU Request và Limit mặc định cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/)

* [Cấu hình ràng buộc Memory tối thiểu và tối đa cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)

* [Cấu hình Quota Memory và CPU cho một Namespace](233-quota-memory-cpu-namespace-vi.md)

* [Cấu hình Quota cho các đối tượng API](https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên Memory cho Container và Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)

* [Gán tài nguyên CPU cho Container và Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)

* [Gán tài nguyên CPU và memory ở mức Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pod-level-resources/)

* [Cấu hình Quality of Service cho Pod](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
