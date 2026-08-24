# Cấu hình Quota Memory và CPU cho một Namespace (Configure Memory and CPU Quotas for a Namespace)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/
>
> Định nghĩa giới hạn tổng tài nguyên memory và CPU cho một namespace.

Trang này hướng dẫn cách đặt hạn ngạch (quota) cho tổng lượng memory và CPU mà tất cả các Pod
đang chạy trong một namespace được phép sử dụng. Bạn chỉ định quota trong một đối tượng
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

Mỗi node trong cluster của bạn phải có ít nhất 1 GiB memory.

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được cô lập khỏi phần còn
lại của cluster.

```shell
kubectl create namespace quota-mem-cpu-example
```

## Tạo một ResourceQuota (Create a ResourceQuota)

Dưới đây là manifest cho một ResourceQuota mẫu:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

Tạo ResourceQuota:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-mem-cpu.yaml --namespace=quota-mem-cpu-example
```

Xem thông tin chi tiết về ResourceQuota:

```shell
kubectl get resourcequota mem-cpu-demo --namespace=quota-mem-cpu-example --output=yaml
```

ResourceQuota này đặt các yêu cầu sau lên namespace quota-mem-cpu-example:

* Với mọi Pod trong namespace, mỗi container phải có memory request, memory limit, cpu request
  và cpu limit.
* Tổng memory request của tất cả các Pod trong namespace đó không được vượt quá 1 GiB.
* Tổng memory limit của tất cả các Pod trong namespace đó không được vượt quá 2 GiB.
* Tổng CPU request của tất cả các Pod trong namespace đó không được vượt quá 1 cpu.
* Tổng CPU limit của tất cả các Pod trong namespace đó không được vượt quá 2 cpu.

Xem [ý nghĩa của CPU](110-manage-resources-containers-vi.md#meaning-of-cpu)
để hiểu Kubernetes định nghĩa "1 CPU" như thế nào.

## Tạo một Pod (Create a Pod)

Dưới đây là manifest cho một Pod mẫu:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo
spec:
  containers:
  - name: quota-mem-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
        cpu: "800m"
      requests:
        memory: "600Mi"
        cpu: "400m"
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-mem-cpu-pod.yaml --namespace=quota-mem-cpu-example
```

Xác minh rằng Pod đang chạy và container (duy nhất) của nó khỏe mạnh:

```shell
kubectl get pod quota-mem-cpu-demo --namespace=quota-mem-cpu-example
```

Một lần nữa, xem thông tin chi tiết về ResourceQuota:

```shell
kubectl get resourcequota mem-cpu-demo --namespace=quota-mem-cpu-example --output=yaml
```

Kết quả hiển thị quota cùng với lượng quota đã được sử dụng. Bạn có thể thấy các request và
limit về memory và CPU của Pod không vượt quá quota.

```
status:
  hard:
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.cpu: "1"
    requests.memory: 1Gi
  used:
    limits.cpu: 800m
    limits.memory: 800Mi
    requests.cpu: 400m
    requests.memory: 600Mi
```

Nếu bạn có công cụ `jq`, bạn cũng có thể truy vấn (bằng
[JSONPath](https://kubernetes.io/docs/reference/kubectl/jsonpath/)) để lấy riêng các giá trị
`used`, **và** in phần kết quả đó ra một cách dễ đọc. Ví dụ:

```shell
kubectl get resourcequota mem-cpu-demo --namespace=quota-mem-cpu-example -o jsonpath='{ .status.used }' | jq .
```

## Thử tạo Pod thứ hai (Attempt to create a second Pod)

Dưới đây là manifest cho Pod thứ hai:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo-2
spec:
  containers:
  - name: quota-mem-cpu-demo-2-ctr
    image: redis
    resources:
      limits:
        memory: "1Gi"
        cpu: "800m"
      requests:
        memory: "700Mi"
        cpu: "400m"
```

Trong manifest, bạn có thể thấy Pod này có memory request là 700 MiB. Lưu ý rằng tổng của
memory request đã dùng cộng với memory request mới này vượt quá quota memory request:
600 MiB + 700 MiB > 1 GiB.

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-mem-cpu-pod-2.yaml --namespace=quota-mem-cpu-example
```

Pod thứ hai không được tạo. Kết quả cho thấy việc tạo Pod thứ hai sẽ khiến tổng memory request
vượt quá quota memory request.

```
Error from server (Forbidden): error when creating "examples/admin/resource/quota-mem-cpu-pod-2.yaml":
pods "quota-mem-cpu-demo-2" is forbidden: exceeded quota: mem-cpu-demo,
requested: requests.memory=700Mi,used: requests.memory=600Mi, limited: requests.memory=1Gi
```

## Thảo luận (Discussion)

Như bạn đã thấy trong bài thực hành này, bạn có thể dùng ResourceQuota để giới hạn tổng memory
request của tất cả các Pod đang chạy trong một namespace. Bạn cũng có thể giới hạn tổng của
memory limit, cpu request và cpu limit.

Thay vì quản lý tổng mức sử dụng tài nguyên trong một namespace, có thể bạn muốn giới hạn từng
Pod riêng lẻ, hoặc các container trong những Pod đó. Để đạt được kiểu giới hạn như vậy, hãy dùng
[LimitRange](133-limit-range-vi.md).

> **Ghi chú:**
>
> Khi sử dụng [thay đổi kích thước Pod tại chỗ (in-place Pod resize)](289-resize-container-resources-vi.md),
> việc thực thi ResourceQuota được áp dụng lên các giá trị sau khi thay đổi. Nếu một lần thay
> đổi kích thước khiến namespace vượt quá giới hạn quota, thao tác đó bị từ chối và tài nguyên
> của Pod giữ nguyên không đổi.

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace quota-mem-cpu-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình Memory Request và Limit mặc định cho một Namespace](232-memory-default-namespace-vi.md)

* [Cấu hình CPU Request và Limit mặc định cho một Namespace](230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc Memory tối thiểu và tối đa cho một Namespace](231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](229-cpu-constraint-namespace-vi.md)

* [Cấu hình Quota số lượng Pod cho một Namespace](234-quota-pod-namespace-vi.md)

* [Cấu hình Quota cho các đối tượng API](252-quota-api-object-vi.md)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên Memory cho Container và Pod](264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho Container và Pod](263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và memory ở mức Pod](265-assign-pod-level-resources-vi.md)

* [Cấu hình Quality of Service cho Pod](288-quality-service-pod-vi.md)
