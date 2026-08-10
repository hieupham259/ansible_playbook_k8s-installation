# Chia sẻ một Cluster bằng Namespace (Share a Cluster with Namespaces)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/namespaces/

Trang này trình bày cách xem, làm việc bên trong và xóa các namespace. Trang cũng trình bày cách
dùng namespace của Kubernetes để phân chia cluster của bạn.

## Trước khi bạn bắt đầu (Before you begin)

* Có sẵn một [cluster Kubernetes](https://kubernetes.io/docs/setup/).
* Bạn có hiểu biết cơ bản về Pod, Service và Deployment trong Kubernetes.

## Xem các namespace (Viewing namespaces)

Liệt kê các namespace hiện có trong một cluster bằng:

```shell
kubectl get namespaces
```
```console
NAME              STATUS   AGE
default           Active   11d
kube-node-lease   Active   11d
kube-public       Active   11d
kube-system       Active   11d
```

Kubernetes khởi đầu với bốn namespace ban đầu:

* `default` Namespace mặc định cho các đối tượng không thuộc namespace nào khác
* `kube-node-lease` Namespace này chứa các đối tượng
  [Lease](https://kubernetes.io/docs/concepts/architecture/leases/) gắn với từng node. Node
  lease cho phép kubelet gửi các
  [heartbeat](https://kubernetes.io/docs/concepts/architecture/nodes/#heartbeats) để control
  plane có thể phát hiện node bị lỗi.
* `kube-public` Namespace này được tạo tự động và mọi người dùng đều đọc được (kể cả những
  người chưa được xác thực). Namespace này chủ yếu được dành riêng cho việc sử dụng của cluster,
  trong trường hợp một số tài nguyên cần được hiển thị và đọc được công khai trong toàn bộ
  cluster. Khía cạnh công khai của namespace này chỉ là một quy ước, không phải một yêu cầu
  bắt buộc.
* `kube-system` Namespace cho các đối tượng do hệ thống Kubernetes tạo ra

Bạn cũng có thể lấy thông tin tóm tắt của một namespace cụ thể bằng:

```shell
kubectl get namespaces <name>
```

Hoặc bạn có thể lấy thông tin chi tiết với:

```shell
kubectl describe namespaces <name>
```
```console
Name:           default
Labels:         <none>
Annotations:    <none>
Status:         Active

No resource quota.

Resource Limits
 Type       Resource    Min Max Default
 ----               --------    --- --- ---
 Container          cpu         -   -   100m
```

Lưu ý rằng các chi tiết này hiển thị cả resource quota (nếu có) lẫn các resource limit range.

Resource quota theo dõi mức sử dụng tổng hợp các tài nguyên trong Namespace và cho phép người
vận hành cluster định nghĩa các giới hạn sử dụng tài nguyên *cứng* (Hard) mà một Namespace được
phép tiêu thụ.

Một limit range định nghĩa các ràng buộc min/max về lượng tài nguyên mà một thực thể đơn lẻ có
thể tiêu thụ trong một Namespace.

Xem [Admission control: Limit Range](https://git.k8s.io/design-proposals-archive/resource-management/admission_control_limit_range.md)

Một namespace có thể ở một trong hai giai đoạn (phase):

* `Active` namespace đang được sử dụng
* `Terminating` namespace đang được xóa và không thể dùng cho các đối tượng mới

Để biết thêm chi tiết, xem
[Namespace](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/namespace-v1/)
trong tài liệu tham chiếu API.

## Tạo một namespace mới (Creating a new namespace)

> **Ghi chú:**
> Tránh tạo namespace có tiền tố `kube-`, vì tiền tố này được dành riêng cho các namespace
> hệ thống của Kubernetes.

Tạo một file YAML mới tên là `my-namespace.yaml` với nội dung:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <insert-namespace-name-here>
```

Sau đó chạy:

```shell
kubectl create -f ./my-namespace.yaml
```

Cách khác, bạn có thể tạo namespace bằng lệnh dưới đây:

```shell
kubectl create namespace <insert-namespace-name-here>
```

Tên namespace của bạn phải là một
[nhãn DNS (DNS label)](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#dns-label-names)
hợp lệ.

Có một trường tùy chọn là `finalizers`, cho phép các bên quan sát (observables) dọn sạch tài
nguyên mỗi khi namespace bị xóa. Hãy nhớ rằng nếu bạn chỉ định một finalizer không tồn tại,
namespace vẫn sẽ được tạo nhưng sẽ bị kẹt ở trạng thái `Terminating` nếu người dùng cố xóa nó.

Bạn có thể tìm thêm thông tin về `finalizers` trong
[tài liệu thiết kế](https://git.k8s.io/design-proposals-archive/architecture/namespaces.md#finalizers)
của namespace.

## Xóa một namespace (Deleting a namespace)

Xóa một namespace bằng

```shell
kubectl delete namespaces <insert-some-namespace-name>
```

> **Cảnh báo:**
> Thao tác này xóa _mọi thứ_ bên trong namespace!

Việc xóa này diễn ra bất đồng bộ (asynchronous), vì vậy trong một khoảng thời gian bạn sẽ thấy
namespace ở trạng thái `Terminating`.

## Phân chia cluster của bạn bằng namespace của Kubernetes (Subdividing your cluster using Kubernetes namespaces)

Theo mặc định, một cluster Kubernetes sẽ khởi tạo một namespace default khi cấp phát (provision)
cluster để chứa tập hợp mặc định các Pod, Service và Deployment mà cluster sử dụng.

Giả sử bạn có một cluster mới, bạn có thể xem các namespace hiện có bằng cách làm như sau:

```shell
kubectl get namespaces
```
```console
NAME      STATUS    AGE
default   Active    13m
```

### Tạo các namespace mới (Create new namespaces)

Trong bài tập này, bạn tạo thêm hai namespace Kubernetes để chứa nội dung của bạn.

Trong một kịch bản mà một tổ chức đang dùng chung một cluster Kubernetes cho cả mục đích phát
triển (development) và sản xuất (production):

- Đội phát triển muốn duy trì một không gian trong cluster nơi họ có thể xem danh sách các Pod,
  Service và Deployment mà họ dùng để xây dựng và chạy ứng dụng của mình. Trong không gian này,
  các tài nguyên Kubernetes được tạo ra và mất đi liên tục, và các hạn chế về việc ai được hay
  không được sửa đổi tài nguyên được nới lỏng để hỗ trợ phát triển linh hoạt (agile).

- Đội vận hành muốn duy trì một không gian trong cluster nơi họ có thể áp đặt các quy trình
  nghiêm ngặt về việc ai được hay không được thao tác trên tập hợp các Pod, Service và
  Deployment đang chạy trang production.

Một khuôn mẫu mà tổ chức này có thể áp dụng là phân vùng cluster Kubernetes thành hai namespace:
`development` và `production`. Hãy tạo hai namespace mới để chứa phần việc của bạn.

Tạo namespace `development` bằng kubectl:

```shell
kubectl create -f https://k8s.io/examples/admin/namespace-dev.json
```

Tạo namespace `production` bằng kubectl:

```shell
kubectl create -f https://k8s.io/examples/admin/namespace-prod.json
```

Để chắc chắn mọi thứ đã đúng, liệt kê tất cả các namespace trong cluster.

```shell
kubectl get namespaces --show-labels
```

```console
NAME          STATUS    AGE       LABELS
default       Active    32m       <none>
development   Active    29s       name=development
production    Active    23s       name=production
```

### Tạo pod trong mỗi namespace (Create pods in each namespace)

Một namespace của Kubernetes cung cấp phạm vi (scope) cho các Pod, Service và Deployment trong
cluster. Người dùng tương tác với một namespace sẽ không nhìn thấy nội dung trong namespace
khác. Để minh họa điều này, hãy tạo một Deployment và các Pod trong namespace `development`.

```shell
kubectl create deployment snowflake \
  --image=registry.k8s.io/serve_hostname \
  -n=development --replicas=2
```

Bạn vừa tạo một deployment với số replica là 2, chạy pod tên là `snowflake` với một container cơ
bản phục vụ trả về hostname.

```shell
kubectl get deployment -n=development
```
```console
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
snowflake    2/2     2            2           2m
```

```shell
kubectl get pods -l app=snowflake -n=development
```
```console
NAME                         READY     STATUS    RESTARTS   AGE
snowflake-3968820950-9dgr8   1/1       Running   0          2m
snowflake-3968820950-vgc4n   1/1       Running   0          2m
```

Điều này cho thấy các nhà phát triển có thể làm những gì họ muốn mà không phải lo lắng về việc
ảnh hưởng tới nội dung trong namespace `production`.

Chuyển sang namespace `production` và xem cách tài nguyên trong một namespace bị ẩn khỏi
namespace kia. Namespace `production` lúc này còn trống, và các lệnh sau sẽ không trả về gì.

```shell
kubectl get deployment -n=production
kubectl get pods -n=production
```

Tạo một số pod trong namespace `production`.

```shell
kubectl create deployment cattle --image=registry.k8s.io/serve_hostname -n=production
kubectl scale deployment cattle --replicas=5 -n=production

kubectl get deployment -n=production
```

```console
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
cattle       5/5     5            5           10s
```

```shell
kubectl get pods -l app=cattle -n=production
```
```console
NAME                      READY     STATUS    RESTARTS   AGE
cattle-2263376956-41xy6   1/1       Running   0          34s
cattle-2263376956-kw466   1/1       Running   0          34s
cattle-2263376956-n4v97   1/1       Running   0          34s
cattle-2263376956-p5p3i   1/1       Running   0          34s
cattle-2263376956-sxpth   1/1       Running   0          34s
```

Đến đây, hẳn đã rõ ràng rằng các tài nguyên mà người dùng tạo trong một namespace bị ẩn khỏi
namespace kia.

Khi khả năng hỗ trợ chính sách (policy) trong Kubernetes phát triển thêm, kịch bản này sẽ được
mở rộng để cho thấy cách bạn có thể cung cấp các quy tắc ủy quyền (authorization) khác nhau cho
từng namespace.

## Hiểu động lực của việc dùng namespace (Understanding the motivation for using namespaces)

Một cluster đơn lẻ nên có khả năng đáp ứng nhu cầu của nhiều người dùng hoặc nhiều nhóm người
dùng (từ đây trong tài liệu này gọi là một _cộng đồng người dùng_).

Các _namespace_ của Kubernetes giúp các dự án, các đội nhóm hoặc các khách hàng khác nhau chia
sẻ chung một cluster Kubernetes.

Nó làm được điều đó bằng cách cung cấp:

1. Một phạm vi cho các [tên (names)](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/).
1. Một cơ chế để gắn việc ủy quyền và chính sách vào một phần con của cluster.

Việc dùng nhiều namespace là tùy chọn.

Mỗi cộng đồng người dùng muốn có thể làm việc tách biệt với các cộng đồng khác. Mỗi cộng đồng
người dùng có riêng:

1. các tài nguyên (pod, service, replication controller, v.v.)
1. các chính sách (ai được hay không được thực hiện hành động trong cộng đồng của họ)
1. các ràng buộc (cộng đồng này được cấp chừng này quota, v.v.)

Người vận hành cluster có thể tạo một Namespace cho mỗi cộng đồng người dùng riêng biệt.

Namespace cung cấp một phạm vi duy nhất cho:

1. các tài nguyên có tên (để tránh xung đột tên cơ bản)
1. việc ủy thác quyền quản lý cho những người dùng được tin cậy
1. khả năng giới hạn mức tiêu thụ tài nguyên của cộng đồng

Các trường hợp sử dụng bao gồm:

1. Với vai trò người vận hành cluster, tôi muốn hỗ trợ nhiều cộng đồng người dùng trên một
   cluster duy nhất.
1. Với vai trò người vận hành cluster, tôi muốn ủy thác quyền quản lý các phân vùng của cluster
   cho những người dùng được tin cậy trong các cộng đồng đó.
1. Với vai trò người vận hành cluster, tôi muốn giới hạn lượng tài nguyên mỗi cộng đồng có thể
   tiêu thụ nhằm hạn chế ảnh hưởng tới các cộng đồng khác đang dùng cluster.
1. Với vai trò người dùng cluster, tôi muốn tương tác với các tài nguyên liên quan tới cộng
   đồng người dùng của mình một cách tách biệt khỏi những gì các cộng đồng người dùng khác đang
   làm trên cluster.

## Hiểu về namespace và DNS (Understanding namespaces and DNS)

Khi bạn tạo một [Service](https://kubernetes.io/docs/concepts/services-networking/service/), nó
tạo một [bản ghi DNS (DNS entry)](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
tương ứng. Bản ghi này có dạng `<service-name>.<namespace-name>.svc.cluster.local`, nghĩa là
nếu một container chỉ dùng `<service-name>`, nó sẽ phân giải tới service nằm cùng namespace.
Điều này hữu ích khi dùng cùng một cấu hình cho nhiều namespace như Development, Staging và
Production. Nếu bạn muốn truy cập xuyên namespace, bạn cần dùng tên miền đầy đủ (fully qualified
domain name — FQDN).

## Tiếp theo (What's next)

* Tìm hiểu thêm về [thiết lập namespace ưa dùng](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#setting-the-namespace-preference).
* Tìm hiểu thêm về [thiết lập namespace cho một request](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/#setting-the-namespace-for-a-request)
* Xem [thiết kế namespace](https://git.k8s.io/design-proposals-archive/architecture/namespaces.md).
