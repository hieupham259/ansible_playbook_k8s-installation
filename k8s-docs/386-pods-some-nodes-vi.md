# Chỉ chạy Pod trên một số Node nhất định (Running Pods on Only Some Nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-daemon/pods-some-nodes/>
>
> Hướng dẫn dùng label của node và `nodeSelector` để một DaemonSet chỉ đặt Pod lên một tập node
> nhất định thay vì mọi node trong cluster.

Trang này minh họa cách bạn có thể chỉ chạy Pod trên một số Node nhất định trong khuôn khổ một
DaemonSet.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Chỉ chạy Pod trên một số Node nhất định (Running Pods on only some Nodes)

Hãy hình dung bạn muốn chạy một DaemonSet, nhưng bạn chỉ cần chạy các daemon pod đó trên những
node có ổ lưu trữ thể rắn (SSD) cục bộ. Chẳng hạn, Pod có thể cung cấp dịch vụ cache cho node, và
cache chỉ hữu ích khi có sẵn bộ lưu trữ cục bộ độ trễ thấp.

### Bước 1: Thêm label cho các node của bạn (Step 1: Add labels to your nodes)

Thêm label `ssd=true` vào những node có ổ SSD.

```shell
kubectl label nodes example-node-1 example-node-2 ssd=true
```

### Bước 2: Tạo manifest (Step 2: Create the manifest)

Hãy tạo một DaemonSet để cấp phát (provision) các daemon pod chỉ trên những node được gắn label
SSD.

Tiếp theo, hãy dùng `nodeSelector` để đảm bảo DaemonSet chỉ chạy Pod trên các node có label `ssd`
được đặt bằng `"true"`.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-driver
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: ssd-driver-pod
  template:
    metadata:
      labels:
        app: ssd-driver-pod
    spec:
      nodeSelector:
        ssd: "true"
      containers:
        - name: example-container
          image: example-image
```

### Bước 3: Tạo DaemonSet (Step 3: Create the DaemonSet)

Tạo DaemonSet từ manifest bằng cách dùng `kubectl create` hoặc `kubectl apply`.

Bây giờ hãy gắn label `ssd=true` cho một node khác.

```shell
kubectl label nodes example-node-3 ssd=true
```

Việc gắn label cho node sẽ tự động kích hoạt control plane (cụ thể là DaemonSet controller) chạy
một daemon pod mới trên node đó.

```shell
kubectl get pods -o wide
```

Kết quả trả về sẽ tương tự như sau:

```
NAME                              READY     STATUS    RESTARTS   AGE    IP      NODE
<daemonset-name><some-hash-01>    1/1       Running   0          13s    .....   example-node-1
<daemonset-name><some-hash-02>    1/1       Running   0          13s    .....   example-node-2
<daemonset-name><some-hash-03>    1/1       Running   0          5s     .....   example-node-3
```
