# Xây dựng một DaemonSet cơ bản (Building a Basic DaemonSet)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/manage-daemon/create-daemon-set/>
>
> Trang này minh họa cách xây dựng một DaemonSet cơ bản để chạy một Pod trên mọi node
> trong cluster Kubernetes.

Trang này minh họa cách xây dựng một DaemonSet cơ bản chạy một Pod trên mọi node trong một
cluster Kubernetes. Bài viết trình bày một tình huống sử dụng đơn giản: mount một file từ
host, ghi log nội dung của file đó bằng [init container](50-init-containers-vi.md), và tận
dụng một pause container.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Một cluster Kubernetes có ít nhất hai node (một control plane node và một worker node) để
minh họa hành vi của DaemonSet.

## Định nghĩa DaemonSet (Define the DaemonSet)

Trong bài thực hành này, bạn tạo một DaemonSet cơ bản để đảm bảo rằng bản sao của một Pod
được lập lịch (schedule) trên mọi node. Pod này sẽ dùng một init container để đọc và ghi log
nội dung của `/etc/machine-id` từ host, còn container chính sẽ là một container `pause`, có
nhiệm vụ giữ cho Pod tiếp tục chạy.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: example-daemonset
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: example
  template:
    metadata:
      labels:
        app.kubernetes.io/name: example
    spec:
      containers:
      - name: pause
        image: registry.k8s.io/pause
      initContainers:
      - name: log-machine-id
        image: busybox:1.37
        command: ['sh', '-c', 'cat /etc/machine-id > /var/log/machine-id.log']
        volumeMounts:
        - name: machine-id
          mountPath: /etc/machine-id
          readOnly: true
        - name: log-dir
          mountPath: /var/log
      volumes:
      - name: machine-id
        hostPath:
          path: /etc/machine-id
          type: File
      - name: log-dir
        hostPath:
          path: /var/log
```

1. Tạo một DaemonSet dựa trên manifest (YAML) ở trên:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/basic-daemonset.yaml
   ```

1. Sau khi áp dụng, bạn có thể xác minh rằng DaemonSet đang chạy một Pod trên mọi node trong
   cluster:

   ```shell
   kubectl get pods -o wide
   ```

   Kết quả sẽ liệt kê mỗi node một Pod, tương tự như sau:

   ```
   NAME                                READY   STATUS    RESTARTS   AGE    IP       NODE
   example-daemonset-xxxxx             1/1     Running   0          5m     x.x.x.x  node-1
   example-daemonset-yyyyy             1/1     Running   0          5m     x.x.x.x  node-2
   ```

1. Bạn có thể kiểm tra nội dung của file `/etc/machine-id` đã được ghi log bằng cách xem thư
   mục log được mount từ host:

   ```shell
   kubectl exec <pod-name> -- cat /var/log/machine-id.log
   ```

   Trong đó `<pod-name>` là tên của một trong các Pod của bạn.

## Dọn dẹp (Cleaning up)

Để xóa DaemonSet, hãy chạy lệnh sau:

```shell
kubectl delete --cascade=foreground --ignore-not-found --now daemonsets/example-daemonset
```

Ví dụ DaemonSet đơn giản này giới thiệu những thành phần chủ chốt như init container và
volume dạng host path, và bạn có thể mở rộng chúng cho các tình huống sử dụng nâng cao hơn.
Để biết thêm chi tiết, hãy tham khảo [DaemonSet](66-daemonset-vi.md).

## Tiếp theo (What's next)

* Xem [Thực hiện rolling update trên một DaemonSet](388-update-daemon-set-vi.md)
* Xem [Tạo một DaemonSet để nhận (adopt) các Pod DaemonSet đã tồn tại](66-daemonset-vi.md)
