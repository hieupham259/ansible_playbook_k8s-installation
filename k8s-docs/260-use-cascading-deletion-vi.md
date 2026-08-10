# Sử dụng xóa theo tầng trong Cluster (Use Cascading Deletion in a Cluster)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/

Trang này hướng dẫn bạn cách chỉ định kiểu
[xóa theo tầng (cascading deletion)](36-garbage-collection-vi.md#cascading-deletion)
được sử dụng trong cluster của bạn trong quá trình thu gom rác (garbage collection).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò máy chủ control plane. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc sử dụng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn cũng cần [tạo một Deployment mẫu](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/#creating-and-exploring-an-nginx-deployment)
để thử nghiệm với các kiểu xóa theo tầng khác nhau. Bạn sẽ cần tạo lại
Deployment cho mỗi kiểu xóa.

## Kiểm tra owner reference trên các pod của bạn (Check owner references on your pods)

Kiểm tra rằng trường `ownerReferences` có mặt trên các pod của bạn:

```shell
kubectl get pods -l app=nginx --output=yaml
```

Kết quả đầu ra có một trường `ownerReferences` tương tự như sau:

```yaml
apiVersion: v1
    ...
    ownerReferences:
    - apiVersion: apps/v1
      blockOwnerDeletion: true
      controller: true
      kind: ReplicaSet
      name: nginx-deployment-6b474476c4
      uid: 4fdcd81c-bd5d-41f7-97af-3a3b759af9a7
    ...
```

## Sử dụng xóa theo tầng foreground (Use foreground cascading deletion) {#use-foreground-cascading-deletion}

Theo mặc định, Kubernetes sử dụng [xóa theo tầng background](36-garbage-collection-vi.md#background-deletion)
để xóa các đối tượng phụ thuộc (dependent) của một đối tượng. Bạn có thể chuyển sang xóa theo
tầng foreground bằng `kubectl` hoặc bằng Kubernetes API, tùy theo phiên bản Kubernetes
mà cluster của bạn đang chạy. Để kiểm tra phiên bản, hãy chạy `kubectl version`.

Bạn có thể xóa các đối tượng bằng xóa theo tầng foreground thông qua `kubectl` hoặc
Kubernetes API.

**Sử dụng kubectl**

Chạy lệnh sau:

```shell
kubectl delete deployment nginx-deployment --cascade=foreground
```

**Sử dụng Kubernetes API**

1. Khởi động một phiên proxy cục bộ:

   ```shell
   kubectl proxy --port=8080
   ```

1. Dùng `curl` để kích hoạt việc xóa:

   ```shell
   curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/deployments/nginx-deployment \
       -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
       -H "Content-Type: application/json"
   ```

   Kết quả đầu ra chứa một finalizer `foregroundDeletion`
   như sau:

   ```
   "kind": "Deployment",
   "apiVersion": "apps/v1",
   "metadata": {
       "name": "nginx-deployment",
       "namespace": "default",
       "uid": "d1ce1b02-cae8-4288-8a53-30e84d8fa505",
       "resourceVersion": "1363097",
       "creationTimestamp": "2021-07-08T20:24:37Z",
       "deletionTimestamp": "2021-07-08T20:27:39Z",
       "finalizers": [
         "foregroundDeletion"
       ]
       ...
   ```

## Sử dụng xóa theo tầng background (Use background cascading deletion) {#use-background-cascading-deletion}

1. [Tạo một Deployment mẫu](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/#creating-and-exploring-an-nginx-deployment).
1. Dùng `kubectl` hoặc Kubernetes API để xóa Deployment,
   tùy theo phiên bản Kubernetes mà cluster của bạn đang chạy. Để kiểm tra phiên bản, hãy chạy
   `kubectl version`.

Bạn có thể xóa các đối tượng bằng xóa theo tầng background thông qua `kubectl`
hoặc Kubernetes API.

Kubernetes sử dụng xóa theo tầng background theo mặc định, và vẫn làm như vậy
ngay cả khi bạn chạy các lệnh dưới đây mà không có cờ `--cascade` hay đối số
`propagationPolicy`.

**Sử dụng kubectl**

Chạy lệnh sau:

```shell
kubectl delete deployment nginx-deployment --cascade=background
```

**Sử dụng Kubernetes API**

1. Khởi động một phiên proxy cục bộ:

   ```shell
   kubectl proxy --port=8080
   ```

1. Dùng `curl` để kích hoạt việc xóa:

   ```shell
   curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/deployments/nginx-deployment \
       -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
       -H "Content-Type: application/json"
   ```

   Kết quả đầu ra tương tự như sau:

   ```
   "kind": "Status",
   "apiVersion": "v1",
   ...
   "status": "Success",
   "details": {
       "name": "nginx-deployment",
       "group": "apps",
       "kind": "deployments",
       "uid": "cc9eefb9-2d49-4445-b1c1-d261c9396456"
   }
   ```

## Xóa đối tượng chủ sở hữu và bỏ lại các đối tượng phụ thuộc (Delete owner objects and orphan dependents) {#set-orphan-deletion-policy}

Theo mặc định, khi bạn yêu cầu Kubernetes xóa một đối tượng, controller
cũng xóa các đối tượng phụ thuộc. Bạn có thể khiến Kubernetes *bỏ lại (orphan)* các đối tượng
phụ thuộc này bằng `kubectl` hoặc Kubernetes API, tùy theo phiên bản Kubernetes mà cluster
của bạn đang chạy. Để kiểm tra phiên bản, hãy chạy `kubectl version`.

**Sử dụng kubectl**

Chạy lệnh sau:

```shell
kubectl delete deployment nginx-deployment --cascade=orphan
```

**Sử dụng Kubernetes API**

1. Khởi động một phiên proxy cục bộ:

   ```shell
   kubectl proxy --port=8080
   ```

1. Dùng `curl` để kích hoạt việc xóa:

   ```shell
   curl -X DELETE localhost:8080/apis/apps/v1/namespaces/default/deployments/nginx-deployment \
       -d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
       -H "Content-Type: application/json"
   ```

   Kết quả đầu ra chứa `orphan` trong trường `finalizers`, tương tự như sau:

   ```
   "kind": "Deployment",
   "apiVersion": "apps/v1",
   "namespace": "default",
   "uid": "6f577034-42a0-479d-be21-78018c466f1f",
   "creationTimestamp": "2021-07-09T16:46:37Z",
   "deletionTimestamp": "2021-07-09T16:47:08Z",
   "deletionGracePeriodSeconds": 0,
   "finalizers": [
     "orphan"
   ],
   ...
   ```

Bạn có thể kiểm tra rằng các Pod do Deployment quản lý vẫn đang chạy:

```shell
kubectl get pods -l app=nginx
```

## Tiếp theo (What's next)

* Tìm hiểu về [chủ sở hữu và đối tượng phụ thuộc (owners and dependents)](30-owners-dependents-vi.md) trong Kubernetes.
* Tìm hiểu về [finalizer](29-finalizers-vi.md) trong Kubernetes.
* Tìm hiểu về [thu gom rác (garbage collection)](36-garbage-collection-vi.md).
