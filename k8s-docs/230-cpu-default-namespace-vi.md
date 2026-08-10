# Cấu hình CPU request và limit mặc định cho một Namespace (Configure Default CPU Requests and Limits for a Namespace)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/>
>
> Định nghĩa giới hạn tài nguyên CPU mặc định cho một namespace, để mọi Pod mới
> trong namespace đó đều được cấu hình giới hạn tài nguyên CPU.

Trang này hướng dẫn cách cấu hình CPU request và limit mặc định cho một namespace.

Một cluster Kubernetes có thể được chia thành nhiều namespace. Nếu bạn tạo một Pod trong một
namespace đã có CPU
[limit](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits)
mặc định, và bất kỳ container nào trong Pod đó không chỉ định CPU limit của riêng nó, thì
control plane sẽ gán CPU limit mặc định cho container đó.

Kubernetes gán CPU request mặc định, nhưng chỉ trong một số điều kiện nhất định sẽ được
giải thích ở phần sau của trang này.

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

Nếu bạn chưa quen với ý nghĩa của 1.0 CPU trong Kubernetes, hãy đọc
[ý nghĩa của CPU](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-cpu).

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi
phần còn lại của cluster.

```shell
kubectl create namespace default-cpu-example
```

## Tạo một LimitRange và một Pod (Create a LimitRange and a Pod)

Dưới đây là manifest cho một LimitRange ví dụ. Manifest này chỉ định một CPU request mặc định
và một CPU limit mặc định.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-limit-range
spec:
  limits:
  - default:
      cpu: 1
    defaultRequest:
      cpu: 0.5
    type: Container
```

Tạo LimitRange trong namespace default-cpu-example:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-defaults.yaml --namespace=default-cpu-example
```

Bây giờ nếu bạn tạo một Pod trong namespace default-cpu-example, và bất kỳ container nào
trong Pod đó không chỉ định giá trị CPU request và CPU limit của riêng nó, thì control plane
sẽ áp dụng các giá trị mặc định: CPU request là 0.5 và CPU limit mặc định là 1.

Dưới đây là manifest cho một Pod có một container. Container này không chỉ định
CPU request và limit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo
spec:
  containers:
  - name: default-cpu-demo-ctr
    image: nginx
```

Tạo Pod.

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-defaults-pod.yaml --namespace=default-cpu-example
```

Xem specification của Pod:

```shell
kubectl get pod default-cpu-demo --output=yaml --namespace=default-cpu-example
```

Output cho thấy container duy nhất của Pod có CPU request là 500m `cpu`
(bạn có thể đọc là "500 millicpu"), và CPU limit là 1 `cpu`.
Đây là các giá trị mặc định do LimitRange chỉ định.

```shell
containers:
- image: nginx
  imagePullPolicy: Always
  name: default-cpu-demo-ctr
  resources:
    limits:
      cpu: "1"
    requests:
      cpu: 500m
```

## Điều gì xảy ra nếu bạn chỉ định limit của container mà không chỉ định request? (What if you specify a container's limit, but not its request?)

Dưới đây là manifest cho một Pod có một container. Container này chỉ định CPU limit,
nhưng không chỉ định request:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo-2
spec:
  containers:
  - name: default-cpu-demo-2-ctr
    image: nginx
    resources:
      limits:
        cpu: "1"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-defaults-pod-2.yaml --namespace=default-cpu-example
```

Xem [specification](https://kubernetes.io/docs/concepts/overview/working-with-objects/#object-spec-and-status)
của Pod bạn vừa tạo:

```
kubectl get pod default-cpu-demo-2 --output=yaml --namespace=default-cpu-example
```

Output cho thấy CPU request của container được đặt bằng với CPU limit của nó.
Lưu ý rằng container không được gán giá trị CPU request mặc định 0.5 `cpu`:

```
resources:
  limits:
    cpu: "1"
  requests:
    cpu: "1"
```

## Điều gì xảy ra nếu bạn chỉ định request của container mà không chỉ định limit? (What if you specify a container's request, but not its limit?)

Dưới đây là manifest ví dụ cho một Pod có một container. Container này chỉ định CPU request,
nhưng không chỉ định limit:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-cpu-demo-3
spec:
  containers:
  - name: default-cpu-demo-3-ctr
    image: nginx
    resources:
      requests:
        cpu: "0.75"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/cpu-defaults-pod-3.yaml --namespace=default-cpu-example
```

Xem specification của Pod bạn vừa tạo:

```
kubectl get pod default-cpu-demo-3 --output=yaml --namespace=default-cpu-example
```

Output cho thấy CPU request của container được đặt theo giá trị bạn đã chỉ định lúc tạo Pod
(nói cách khác: nó khớp với manifest). Tuy nhiên, CPU limit của chính container đó được đặt
là 1 `cpu`, chính là CPU limit mặc định của namespace này.

```
resources:
  limits:
    cpu: "1"
  requests:
    cpu: 750m
```

## Động lực cho CPU limit và request mặc định (Motivation for default CPU limits and requests)

Nếu namespace của bạn đã cấu hình hạn ngạch tài nguyên (resource quota) cho CPU,
thì việc có sẵn một giá trị mặc định cho CPU limit là rất hữu ích.
Dưới đây là hai trong số các ràng buộc mà một hạn ngạch tài nguyên CPU áp đặt lên một namespace:

* Với mọi Pod chạy trong namespace, mỗi container của nó phải có CPU limit.
* CPU limit áp dụng một lượng tài nguyên được đặt trước (resource reservation) trên node
  mà Pod đó được lập lịch (schedule). Tổng lượng CPU được đặt trước cho tất cả các Pod
  trong namespace không được vượt quá một giới hạn đã chỉ định.

Khi bạn thêm một LimitRange:

Nếu bất kỳ Pod nào trong namespace đó chứa một container không chỉ định CPU limit của riêng nó,
control plane sẽ áp dụng CPU limit mặc định cho container đó, và Pod có thể được phép chạy
trong một namespace đang bị giới hạn bởi một CPU ResourceQuota.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace default-cpu-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình memory request và limit mặc định cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)

* [Cấu hình ràng buộc bộ nhớ tối thiểu và tối đa cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)

* [Cấu hình hạn ngạch bộ nhớ và CPU cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)

* [Cấu hình hạn ngạch Pod cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-pod-namespace/)

* [Cấu hình hạn ngạch cho các API Object](https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên bộ nhớ cho Container và Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)

* [Gán tài nguyên CPU cho Container và Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)

* [Gán tài nguyên CPU và bộ nhớ ở cấp Pod](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pod-level-resources/)

* [Cấu hình Quality of Service cho Pod](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
