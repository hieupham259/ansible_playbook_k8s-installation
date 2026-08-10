# Cấu hình memory request và limit mặc định cho một Namespace (Configure Default Memory Requests and Limits for a Namespace)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/>
>
> Định nghĩa giới hạn tài nguyên bộ nhớ mặc định cho một namespace, để mọi Pod mới
> trong namespace đó đều được cấu hình giới hạn tài nguyên bộ nhớ.

Trang này hướng dẫn cách cấu hình memory request và limit mặc định cho một namespace.

Một cluster Kubernetes có thể được chia thành nhiều namespace. Khi bạn đã có một namespace
với memory
[limit](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits)
mặc định, và sau đó bạn thử tạo một Pod có container không chỉ định memory limit của riêng nó,
thì control plane sẽ gán memory limit mặc định cho container đó.

Kubernetes gán memory request mặc định trong một số điều kiện nhất định sẽ được giải thích
ở phần sau của bài này.

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

Mỗi node trong cluster của bạn phải có ít nhất 2 GiB bộ nhớ.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi
phần còn lại của cluster.

```shell
kubectl create namespace default-mem-example
```

## Tạo một LimitRange và một Pod (Create a LimitRange and a Pod)

Dưới đây là manifest cho một LimitRange ví dụ. Manifest này chỉ định một memory request
mặc định và một memory limit mặc định.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```

Tạo LimitRange trong namespace default-mem-example:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults.yaml --namespace=default-mem-example
```

Bây giờ nếu bạn tạo một Pod trong namespace default-mem-example, và bất kỳ container nào
trong Pod đó không chỉ định giá trị memory request và memory limit của riêng nó,
thì control plane sẽ áp dụng các giá trị mặc định: memory request là 256MiB và
memory limit là 512MiB.

Dưới đây là manifest ví dụ cho một Pod có một container. Container này không chỉ định
memory request và limit.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo
spec:
  containers:
  - name: default-mem-demo-ctr
    image: nginx
```

Tạo Pod.

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults-pod.yaml --namespace=default-mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod default-mem-demo --output=yaml --namespace=default-mem-example
```

Output cho thấy container của Pod có memory request là 256 MiB và memory limit là 512 MiB.
Đây là các giá trị mặc định do LimitRange chỉ định.

```shell
containers:
- image: nginx
  imagePullPolicy: Always
  name: default-mem-demo-ctr
  resources:
    limits:
      memory: 512Mi
    requests:
      memory: 256Mi
```

Xóa Pod của bạn:

```shell
kubectl delete pod default-mem-demo --namespace=default-mem-example
```

## Điều gì xảy ra nếu bạn chỉ định limit của container mà không chỉ định request? (What if you specify a container's limit, but not its request?)

Dưới đây là manifest cho một Pod có một container. Container này chỉ định memory limit,
nhưng không chỉ định request:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo-2
spec:
  containers:
  - name: default-mem-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "1Gi"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults-pod-2.yaml --namespace=default-mem-example
```

Xem thông tin chi tiết về Pod:

```shell
kubectl get pod default-mem-demo-2 --output=yaml --namespace=default-mem-example
```

Output cho thấy memory request của container được đặt bằng với memory limit của nó.
Lưu ý rằng container không được gán giá trị memory request mặc định 256Mi.

```
resources:
  limits:
    memory: 1Gi
  requests:
    memory: 1Gi
```

## Điều gì xảy ra nếu bạn chỉ định request của container mà không chỉ định limit? (What if you specify a container's request, but not its limit?)

Dưới đây là manifest cho một Pod có một container. Container này chỉ định memory request,
nhưng không chỉ định limit:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo-3
spec:
  containers:
  - name: default-mem-demo-3-ctr
    image: nginx
    resources:
      requests:
        memory: "128Mi"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults-pod-3.yaml --namespace=default-mem-example
```

Xem specification của Pod:

```shell
kubectl get pod default-mem-demo-3 --output=yaml --namespace=default-mem-example
```

Output cho thấy memory request của container được đặt theo giá trị chỉ định trong manifest
của container. Container bị giới hạn sử dụng không quá 512MiB bộ nhớ, khớp với memory limit
mặc định của namespace này.

```
resources:
  limits:
    memory: 512Mi
  requests:
    memory: 128Mi
```

> **Ghi chú:**
>
> Một `LimitRange` **không** kiểm tra tính nhất quán của các giá trị mặc định mà nó áp dụng.
> Điều này có nghĩa là giá trị mặc định cho _limit_ do `LimitRange` đặt có thể nhỏ hơn giá trị
> _request_ được chỉ định cho container trong spec mà client gửi tới API server. Nếu điều đó
> xảy ra, Pod cuối cùng sẽ không thể được lập lịch (schedule).
> Xem [Ràng buộc đối với resource limit và request](https://kubernetes.io/docs/concepts/policy/limit-range/#constraints-on-resource-limits-and-requests)
> để biết thêm chi tiết.

## Động lực cho memory limit và request mặc định (Motivation for default memory limits and requests)

Nếu namespace của bạn đã cấu hình hạn ngạch tài nguyên (resource quota) cho bộ nhớ,
thì việc có sẵn một giá trị mặc định cho memory limit là rất hữu ích.
Dưới đây là ba trong số các ràng buộc mà một hạn ngạch tài nguyên áp đặt lên một namespace:

* Với mọi Pod chạy trong namespace, Pod và mỗi container của nó phải có memory limit.
  (Nếu bạn chỉ định memory limit cho mọi container trong một Pod, Kubernetes có thể suy ra
  memory limit ở cấp Pod bằng cách cộng các limit của các container lại).
* Memory limit áp dụng một lượng tài nguyên được đặt trước (resource reservation) trên node
  mà Pod đó được lập lịch. Tổng lượng bộ nhớ được đặt trước cho tất cả các Pod trong namespace
  không được vượt quá một giới hạn đã chỉ định.
* Tổng lượng bộ nhớ thực sự được sử dụng bởi tất cả các Pod trong namespace cũng không được
  vượt quá một giới hạn đã chỉ định.

Khi bạn thêm một LimitRange:

Nếu bất kỳ Pod nào trong namespace đó chứa một container không chỉ định memory limit của
riêng nó, control plane sẽ áp dụng memory limit mặc định cho container đó, và Pod có thể
được phép chạy trong một namespace đang bị giới hạn bởi một memory ResourceQuota.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace default-mem-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình CPU request và limit mặc định cho một Namespace](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/cpu-default-namespace/)

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
