# Gán Pod vào Node bằng Node Affinity (Assign Pods to Nodes using Node Affinity)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/

Trang này hướng dẫn cách gán một Pod Kubernetes vào một node cụ thể bằng Node Affinity
trong một cluster Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.10 hoặc mới hơn. Để kiểm tra phiên bản,
nhập `kubectl version`.

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

    ```
    NAME      STATUS    ROLES    AGE     VERSION        LABELS
    worker0   Ready     <none>   1d      v1.13.0        ...,disktype=ssd,kubernetes.io/hostname=worker0
    worker1   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker1
    worker2   Ready     <none>   1d      v1.13.0        ...,kubernetes.io/hostname=worker2
    ```

    Trong kết quả trên, bạn có thể thấy node `worker0` có label `disktype=ssd`.

## Lên lịch một Pod bằng node affinity bắt buộc (Schedule a Pod using required node affinity)

Manifest này mô tả một Pod có node affinity dạng
`requiredDuringSchedulingIgnoredDuringExecution` với `disktype: ssd`.
Điều này có nghĩa là Pod sẽ chỉ được lên lịch (schedule) trên node có label `disktype=ssd`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd            
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

1. Áp dụng manifest để tạo một Pod được lên lịch vào node bạn đã chọn:

    ```shell
    kubectl apply -f https://k8s.io/examples/pods/pod-nginx-required-affinity.yaml
    ```

1. Xác nhận rằng Pod đang chạy trên node bạn đã chọn:

    ```shell
    kubectl get pods --output=wide
    ```

    Kết quả tương tự như sau:

    ```
    NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
    nginx    1/1       Running   0          13s    10.200.0.4   worker0
    ```

## Lên lịch một Pod bằng node affinity ưu tiên (Schedule a Pod using preferred node affinity)

Manifest này mô tả một Pod có node affinity dạng
`preferredDuringSchedulingIgnoredDuringExecution` với `disktype: ssd`.
Điều này có nghĩa là Pod sẽ ưu tiên node có label `disktype=ssd`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd          
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

1. Áp dụng manifest để tạo một Pod được lên lịch vào node bạn đã chọn:

    ```shell
    kubectl apply -f https://k8s.io/examples/pods/pod-nginx-preferred-affinity.yaml
    ```

1. Xác nhận rằng Pod đang chạy trên node bạn đã chọn:

    ```shell
    kubectl get pods --output=wide
    ```

    Kết quả tương tự như sau:

    ```
    NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
    nginx    1/1       Running   0          13s    10.200.0.4   worker0
    ```

## Tiếp theo (What's next)

Tìm hiểu thêm về
[Node Affinity](138-assign-pod-node-vi.md#node-affinity).
