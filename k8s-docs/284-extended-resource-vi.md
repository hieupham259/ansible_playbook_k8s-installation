# Gán Extended Resource cho một Container (Assign Extended Resources to a Container)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/extended-resource/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Trang này hướng dẫn cách gán các tài nguyên mở rộng (extended resource) cho một Container.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.36 hoặc mới hơn. Để kiểm tra phiên bản, hãy
nhập `kubectl version`.

Trước khi làm bài thực hành này, hãy làm bài thực hành trong
[Quảng bá Extended Resource cho một Node](./209-extended-resource-node-vi.md).
Bài đó sẽ cấu hình một trong các Node của bạn để quảng bá tài nguyên dongle.

## Gán một extended resource cho một Pod (Assign an extended resource to a Pod)

Để yêu cầu một extended resource, hãy đưa trường `resources.requests.<resource_name>` vào
manifest của container. `*.kubernetes.io/`. Tên extended resource hợp lệ có dạng
`example.com/foo`, trong đó `example.com` được thay bằng tên miền (domain) của tổ chức bạn
và `foo` là một tên tài nguyên mang tính mô tả.

Đây là file cấu hình cho một Pod có một Container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: extended-resource-demo
spec:
  containers:
  - name: extended-resource-demo-ctr
    image: nginx
    resources:
      requests:
        example.com/dongle: 3
      limits:
        example.com/dongle: 3
```

Trong file cấu hình, bạn có thể thấy Container yêu cầu 3 dongle.

Tạo một Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/extended-resource-pod.yaml
```

Xác minh rằng Pod đang chạy:

```shell
kubectl get pod extended-resource-demo
```

Mô tả (describe) Pod:

```shell
kubectl describe pod extended-resource-demo
```

Đầu ra cho thấy các yêu cầu về dongle:

```yaml
Limits:
  example.com/dongle: 3
Requests:
  example.com/dongle: 3
```

## Thử tạo một Pod thứ hai (Attempt to create a second Pod)

Đây là file cấu hình cho một Pod có một Container. Container này yêu cầu hai dongle.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: extended-resource-demo-2
spec:
  containers:
  - name: extended-resource-demo-2-ctr
    image: nginx
    resources:
      requests:
        example.com/dongle: 2
      limits:
        example.com/dongle: 2
```

Kubernetes sẽ không thể đáp ứng yêu cầu hai dongle, bởi vì Pod đầu tiên đã dùng ba trong
bốn dongle khả dụng.

Thử tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/extended-resource-pod-2.yaml
```

Mô tả Pod:

```shell
kubectl describe pod extended-resource-demo-2
```

Đầu ra cho thấy Pod không thể được lập lịch (schedule), bởi vì không có Node nào còn 2
dongle khả dụng:

```
Conditions:
  Type    Status
  PodScheduled  False
...
Events:
  ...
  ... Warning   FailedScheduling  pod (extended-resource-demo-2) failed to fit in any node
fit failure summary on nodes : Insufficient example.com/dongle (1)
```

Xem trạng thái của Pod:

```shell
kubectl get pod extended-resource-demo-2
```

Đầu ra cho thấy Pod đã được tạo, nhưng không được lập lịch để chạy trên một Node.
Nó có trạng thái Pending:

```yaml
NAME                       READY     STATUS    RESTARTS   AGE
extended-resource-demo-2   0/1       Pending   0          6m
```

## Dọn dẹp (Clean up)

Xóa các Pod mà bạn đã tạo cho bài thực hành này:

```shell
kubectl delete pod extended-resource-demo
kubectl delete pod extended-resource-demo-2
```

## Tiếp theo (What's next)

### Dành cho nhà phát triển ứng dụng (For application developers)

* [Gán tài nguyên bộ nhớ cho Container và Pod](./264-assign-memory-resource-vi.md)
* [Gán tài nguyên CPU cho Container và Pod](./263-assign-cpu-resource-vi.md)

### Dành cho quản trị viên cluster (For cluster administrators)

* [Quảng bá Extended Resource cho một Node](./209-extended-resource-node-vi.md)
