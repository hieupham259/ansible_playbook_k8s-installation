# Gán Pod vào Node (Assign Pods to Nodes)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/

Trang này hướng dẫn cách gán một Pod Kubernetes vào một node cụ thể trong một cluster
Kubernetes.

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

## Thêm label cho một node (Add a label to a node)

1. Liệt kê các node trong cluster của bạn, kèm theo các label của chúng:

    ```shell
    kubectl get nodes --show-labels
    ```

    Kết quả tương tự như sau:

    ```shell
    NAME      STATUS    ROLES    AGE     VERSION        LABELS
    worker0   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker0
    worker1   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker1
    worker2   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker2
    ```

1. Chọn một trong các node của bạn và thêm label cho nó:

    ```shell
    kubectl label nodes <your-node-name> disktype=ssd
    ```

    trong đó `<your-node-name>` là tên của node bạn đã chọn.

1. Xác nhận rằng node bạn chọn đã có label `disktype=ssd`:

    ```shell
    kubectl get nodes --show-labels
    ```

    Kết quả tương tự như sau:

    ```shell
    NAME      STATUS    ROLES    AGE     VERSION        LABELS
    worker0   Ready     <none>   1d      v1.13.0        ...,disktype=ssd,kubernetes.io/hostname=worker0
    worker1   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker1
    worker2   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker2
    ```

    Trong kết quả trên, bạn có thể thấy node `worker0` có label `disktype=ssd`.

## Tạo một Pod được lên lịch vào node bạn đã chọn (Create a pod that gets scheduled to your chosen node)

File cấu hình Pod này mô tả một Pod có node selector `disktype: ssd`. Điều này có nghĩa là
Pod sẽ được lên lịch (schedule) trên một node có label `disktype=ssd`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

1. Dùng file cấu hình để tạo một Pod sẽ được lên lịch vào node bạn đã chọn:

    ```shell
    kubectl apply -f https://k8s.io/examples/pods/pod-nginx.yaml
    ```

1. Xác nhận rằng Pod đang chạy trên node bạn đã chọn:

    ```shell
    kubectl get pods --output=wide
    ```

    Kết quả tương tự như sau:

    ```shell
    NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
    nginx    1/1       Running   0          13s    10.200.0.4   worker0
    ```

## Tạo một Pod được lên lịch vào một node cụ thể (Create a pod that gets scheduled to specific node)

Bạn cũng có thể lên lịch một Pod vào đúng một node cụ thể bằng cách đặt `nodeName`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: foo-node # lên lịch Pod vào một node cụ thể
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

Dùng file cấu hình để tạo một Pod sẽ chỉ được lên lịch vào node `foo-node`.

## Tiếp theo (What's next)

* Tìm hiểu thêm về [label và selector](18-labels-vi.md).
* Tìm hiểu thêm về [node](23-nodes-vi.md).
