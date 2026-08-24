# Cấu hình Quota cho các đối tượng API (Configure Quotas for API Objects)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/

Trang này hướng dẫn cách cấu hình quota cho các đối tượng API, bao gồm
PersistentVolumeClaim và Service. Một quota giới hạn số lượng
đối tượng thuộc một loại cụ thể có thể được tạo trong một namespace.
Bạn chỉ định quota trong một đối tượng
[ResourceQuota](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#resourcequota-v1-core).

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

## Tạo một namespace (Create a namespace)

Tạo một namespace để các tài nguyên bạn tạo trong bài thực hành này được
cô lập khỏi phần còn lại của cluster.

```shell
kubectl create namespace quota-object-example
```

## Tạo một ResourceQuota (Create a ResourceQuota)

Dưới đây là file cấu hình cho một đối tượng ResourceQuota:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-quota-demo
spec:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
```

Tạo ResourceQuota:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-objects.yaml --namespace=quota-object-example
```

Xem thông tin chi tiết về ResourceQuota:

```shell
kubectl get resourcequota object-quota-demo --namespace=quota-object-example --output=yaml
```

Kết quả cho thấy trong namespace quota-object-example, có thể có tối đa
một PersistentVolumeClaim, tối đa hai Service loại LoadBalancer, và không có Service
loại NodePort nào.

```yaml
status:
  hard:
    persistentvolumeclaims: "1"
    services.loadbalancers: "2"
    services.nodeports: "0"
  used:
    persistentvolumeclaims: "0"
    services.loadbalancers: "0"
    services.nodeports: "0"
```

## Tạo một PersistentVolumeClaim (Create a PersistentVolumeClaim)

Dưới đây là file cấu hình cho một đối tượng PersistentVolumeClaim:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-quota-demo
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

Tạo PersistentVolumeClaim:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-objects-pvc.yaml --namespace=quota-object-example
```

Xác minh rằng PersistentVolumeClaim đã được tạo:

```shell
kubectl get persistentvolumeclaims --namespace=quota-object-example
```

Kết quả cho thấy PersistentVolumeClaim tồn tại và có trạng thái Pending:

```
NAME             STATUS
pvc-quota-demo   Pending
```

## Thử tạo PersistentVolumeClaim thứ hai (Attempt to create a second PersistentVolumeClaim)

Dưới đây là file cấu hình cho PersistentVolumeClaim thứ hai:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-quota-demo-2
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
```

Thử tạo PersistentVolumeClaim thứ hai:

```shell
kubectl apply -f https://k8s.io/examples/admin/resource/quota-objects-pvc-2.yaml --namespace=quota-object-example
```

Kết quả cho thấy PersistentVolumeClaim thứ hai không được tạo,
vì nó sẽ vượt quá quota của namespace.

```
persistentvolumeclaims "pvc-quota-demo-2" is forbidden:
exceeded quota: object-quota-demo, requested: persistentvolumeclaims=1,
used: persistentvolumeclaims=1, limited: persistentvolumeclaims=1
```

## Ghi chú (Notes)

Đây là các chuỗi được dùng để định danh những tài nguyên API có thể bị ràng buộc
bởi quota:

<table>
<tr><th>Chuỗi</th><th>Đối tượng API</th></tr>
<tr><td>"pods"</td><td>Pod</td></tr>
<tr><td>"services"</td><td>Service</td></tr>
<tr><td>"replicationcontrollers"</td><td>ReplicationController</td></tr>
<tr><td>"resourcequotas"</td><td>ResourceQuota</td></tr>
<tr><td>"secrets"</td><td>Secret</td></tr>
<tr><td>"configmaps"</td><td>ConfigMap</td></tr>
<tr><td>"persistentvolumeclaims"</td><td>PersistentVolumeClaim</td></tr>
<tr><td>"services.nodeports"</td><td>Service loại NodePort</td></tr>
<tr><td>"services.loadbalancers"</td><td>Service loại LoadBalancer</td></tr>
</table>

## Dọn dẹp (Clean up)

Xóa namespace của bạn:

```shell
kubectl delete namespace quota-object-example
```

## Tiếp theo (What's next)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Cấu hình Memory Request và Limit mặc định cho một Namespace](232-memory-default-namespace-vi.md)

* [Cấu hình CPU Request và Limit mặc định cho một Namespace](230-cpu-default-namespace-vi.md)

* [Cấu hình ràng buộc Memory tối thiểu và tối đa cho một Namespace](231-memory-constraint-namespace-vi.md)

* [Cấu hình ràng buộc CPU tối thiểu và tối đa cho một Namespace](229-cpu-constraint-namespace-vi.md)

* [Cấu hình Quota Memory và CPU cho một Namespace](233-quota-memory-cpu-namespace-vi.md)

* [Cấu hình Quota số lượng Pod cho một Namespace](234-quota-pod-namespace-vi.md)

### Dành cho nhà phát triển ứng dụng (For app developers)

* [Gán tài nguyên Memory cho Container và Pod](264-assign-memory-resource-vi.md)

* [Gán tài nguyên CPU cho Container và Pod](263-assign-cpu-resource-vi.md)

* [Gán tài nguyên CPU và memory ở mức Pod](265-assign-pod-level-resources-vi.md)

* [Cấu hình Quality of Service cho Pod](288-quality-service-pod-vi.md)
