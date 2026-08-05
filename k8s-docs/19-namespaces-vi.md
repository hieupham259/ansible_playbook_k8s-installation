# Namespaces

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/>
>
> Trong Kubernetes, namespace cung cấp cơ chế cô lập các nhóm resource bên trong
> một cluster duy nhất.

Trong Kubernetes, _namespace_ cung cấp một cơ chế để cô lập (isolate) các nhóm resource bên trong một cluster duy nhất. Tên của các resource phải là duy nhất trong một namespace, nhưng không cần duy nhất giữa các namespace. Phạm vi (scope) theo namespace chỉ áp dụng cho các object thuộc namespace _(ví dụ: Deployment, Service, v.v.)_ chứ không áp dụng cho các object ở phạm vi toàn cluster _(ví dụ: StorageClass, Node, PersistentVolume, v.v.)_.

## Khi nào nên dùng nhiều namespace (When to Use Multiple Namespaces)

Namespace được thiết kế để dùng trong các môi trường có nhiều người dùng trải rộng trên nhiều
team hoặc nhiều dự án. Với các cluster chỉ có từ vài đến vài chục người dùng, bạn hoàn toàn không
cần phải tạo hay bận tâm đến namespace. Hãy bắt đầu dùng namespace khi bạn
cần đến những tính năng mà chúng cung cấp.

Namespace cung cấp một phạm vi cho tên (scope for names). Tên của các resource phải là duy nhất trong một namespace,
nhưng không cần duy nhất giữa các namespace. Các namespace không thể lồng vào nhau và mỗi resource
Kubernetes chỉ có thể nằm trong một namespace.

Namespace là một cách để phân chia tài nguyên cluster giữa nhiều người dùng (thông qua [resource quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)).

Không nhất thiết phải dùng nhiều namespace để tách các resource chỉ hơi khác nhau,
chẳng hạn các phiên bản khác nhau của cùng một phần mềm: hãy dùng
[label](./18-labels-vi.md) để phân biệt
các resource trong cùng một namespace.

> **Ghi chú:**
> Với cluster production, hãy cân nhắc _không_ dùng namespace `default`. Thay vào đó, hãy tạo các namespace khác và sử dụng chúng.

## Các namespace ban đầu (Initial namespaces)

Kubernetes khởi đầu với bốn namespace ban đầu:

- `default`: Kubernetes bao gồm namespace này để bạn có thể bắt đầu sử dụng cluster mới của mình mà không cần tạo namespace trước.

- `kube-node-lease`: Namespace này chứa các object [Lease](https://kubernetes.io/docs/concepts/architecture/leases/) gắn với từng node. Node lease cho phép kubelet gửi [heartbeat](https://kubernetes.io/docs/concepts/architecture/nodes/#node-heartbeats) để control plane có thể phát hiện node bị lỗi.

- `kube-public`: Namespace này có thể được đọc bởi *tất cả* các client (kể cả những client chưa xác thực). Namespace này chủ yếu được dành cho mục đích sử dụng của cluster, trong trường hợp một số resource cần được hiển thị và đọc công khai trên toàn cluster. Khía cạnh công khai của namespace này chỉ là một quy ước, không phải là một yêu cầu bắt buộc.

- `kube-system`: Namespace dành cho các object do hệ thống Kubernetes tạo ra.

## Làm việc với Namespace (Working with Namespaces)

Việc tạo và xóa namespace được mô tả trong
[tài liệu Hướng dẫn quản trị về namespace](https://kubernetes.io/docs/tasks/administer-cluster/namespaces).

> **Ghi chú:**
> Tránh tạo namespace có prefix `kube-`, vì prefix này được dành riêng cho các namespace hệ thống của Kubernetes.

### Xem các namespace (Viewing namespaces)

Bạn có thể liệt kê các namespace hiện có trong cluster bằng lệnh:

```shell
kubectl get namespace
```
```
NAME              STATUS   AGE
default           Active   1d
kube-node-lease   Active   1d
kube-public       Active   1d
kube-system       Active   1d
```

### Đặt namespace cho một request (Setting the namespace for a request)

Để đặt namespace cho request hiện tại, dùng cờ (flag) `--namespace`.

Ví dụ:

```shell
kubectl run nginx --image=nginx --namespace=<insert-namespace-name-here>
kubectl get pods --namespace=<insert-namespace-name-here>
```

### Đặt namespace mặc định ưu tiên (Setting the namespace preference)

Bạn có thể lưu vĩnh viễn namespace cho tất cả các lệnh kubectl tiếp theo trong
context đó.

```shell
kubectl config set-context --current --namespace=<insert-namespace-name-here>
# Kiểm tra lại
kubectl config view --minify | grep namespace:
```

## Namespace và DNS (Namespaces and DNS)

Khi bạn tạo một [Service](https://kubernetes.io/docs/concepts/services-networking/service/),
nó sẽ tạo một [bản ghi DNS](./10-dns-pod-service-vi.md) tương ứng.
Bản ghi này có dạng `<service-name>.<namespace-name>.svc.cluster.local`, nghĩa là
nếu một container chỉ dùng `<service-name>`, nó sẽ phân giải (resolve) tới service
cục bộ trong namespace đó. Điều này hữu ích khi dùng cùng một cấu hình trên
nhiều namespace, chẳng hạn Development, Staging và Production. Nếu bạn muốn truy cập
xuyên namespace, bạn cần dùng tên miền đầy đủ (fully qualified domain name — FQDN).

Do đó, tất cả tên namespace phải là
[DNS label hợp lệ theo RFC 1123](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-label-names).

> **Cảnh báo:**
> Bằng cách tạo các namespace trùng tên với [các tên miền cấp cao nhất (top-level domain) công cộng](https://data.iana.org/TLD/tlds-alpha-by-domain.txt), các Service trong những
> namespace này có thể có tên DNS ngắn trùng lặp với các bản ghi DNS công cộng.
> Workload từ bất kỳ namespace nào khi thực hiện tra cứu DNS mà không có [dấu chấm ở cuối (trailing dot)](https://datatracker.ietf.org/doc/html/rfc1034#page-8) sẽ
> bị chuyển hướng tới các service đó, được ưu tiên hơn DNS công cộng.
>
> Để giảm thiểu rủi ro này, hãy giới hạn quyền tạo namespace cho những người dùng đáng tin cậy. Nếu
> cần, bạn cũng có thể cấu hình thêm các cơ chế kiểm soát bảo mật bên thứ ba, chẳng hạn
> [admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/),
> để chặn việc tạo bất kỳ namespace nào trùng tên với [các TLD công cộng](https://data.iana.org/TLD/tlds-alpha-by-domain.txt).

## Không phải object nào cũng nằm trong namespace (Not all objects are in a namespace)

Hầu hết các resource của Kubernetes (ví dụ: Pod, Service, Deployment và các loại khác) đều nằm trong một namespace nào đó. Tuy nhiên, bản thân resource namespace lại không nằm trong một namespace. Và các resource cấp thấp, chẳng hạn
[Node](https://kubernetes.io/docs/concepts/architecture/nodes/) và
[PersistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/), không nằm trong bất kỳ namespace nào.

Để xem những resource Kubernetes nào nằm trong và không nằm trong namespace:

```shell
# Nằm trong namespace
kubectl api-resources --namespaced=true

# Không nằm trong namespace
kubectl api-resources --namespaced=false
```

## Gán nhãn tự động (Automatic labelling)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.22 [stable]`

Control plane của Kubernetes đặt một label bất biến (immutable)
`kubernetes.io/metadata.name` trên tất cả các namespace.
Giá trị của label này là tên của namespace.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [tạo một namespace mới](https://kubernetes.io/docs/tasks/administer-cluster/namespaces/#creating-a-new-namespace).
* Tìm hiểu thêm về [xóa một namespace](https://kubernetes.io/docs/tasks/administer-cluster/namespaces/#deleting-a-namespace).
