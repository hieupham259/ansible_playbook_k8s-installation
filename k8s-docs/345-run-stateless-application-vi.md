# Chạy một ứng dụng Stateless bằng Deployment (Run a Stateless Application Using a Deployment)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/

Trang này hướng dẫn cách chạy một ứng dụng bằng đối tượng Deployment của Kubernetes.

## Mục tiêu (Objectives)

- Tạo một deployment nginx.
- Dùng kubectl để liệt kê thông tin về deployment.
- Cập nhật deployment.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Kubernetes server của bạn phải ở phiên bản v1.9 hoặc mới hơn. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

## Tạo và khám phá một deployment nginx (Creating and exploring an nginx deployment) {#creating-and-exploring-an-nginx-deployment}

Bạn có thể chạy một ứng dụng bằng cách tạo một đối tượng Deployment của Kubernetes, và bạn
có thể mô tả một Deployment trong file YAML. Ví dụ, file YAML sau mô tả một Deployment
chạy Docker image nginx:1.14.2:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # yêu cầu deployment chạy 2 pod khớp với template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

1. Tạo một Deployment dựa trên file YAML:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/deployment.yaml
   ```

1. Hiển thị thông tin về Deployment:

   ```shell
   kubectl describe deployment nginx-deployment
   ```

   Kết quả tương tự như sau:

   ```
   Name:     nginx-deployment
   Namespace:    default
   CreationTimestamp:  Tue, 30 Aug 2016 18:11:37 -0700
   Labels:     app=nginx
   Annotations:    deployment.kubernetes.io/revision=1
   Selector:   app=nginx
   Replicas:   2 desired | 2 updated | 2 total | 2 available | 0 unavailable
   StrategyType:   RollingUpdate
   MinReadySeconds:  0
   RollingUpdateStrategy:  1 max unavailable, 1 max surge
   Pod Template:
     Labels:       app=nginx
     Containers:
       nginx:
       Image:              nginx:1.14.2
       Port:               80/TCP
       Environment:        <none>
       Mounts:             <none>
     Volumes:              <none>
   Conditions:
     Type          Status  Reason
     ----          ------  ------
     Available     True    MinimumReplicasAvailable
     Progressing   True    NewReplicaSetAvailable
   OldReplicaSets:   <none>
   NewReplicaSet:    nginx-deployment-1771418926 (2/2 replicas created)
   No events.
   ```

1. Liệt kê các Pod do deployment tạo ra:

   ```shell
   kubectl get pods -l app=nginx
   ```

   Kết quả tương tự như sau:

   ```
   NAME                                READY     STATUS    RESTARTS   AGE
   nginx-deployment-1771418926-7o5ns   1/1       Running   0          16h
   nginx-deployment-1771418926-r18az   1/1       Running   0          16h
   ```

1. Hiển thị thông tin về một Pod:

   ```shell
   kubectl describe pod <pod-name>
   ```

   trong đó `<pod-name>` là tên của một trong các Pod của bạn.

## Cập nhật deployment (Updating the deployment)

Bạn có thể cập nhật deployment bằng cách apply một file YAML mới. File YAML sau
chỉ định rằng deployment cần được cập nhật để dùng nginx 1.16.1.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1 # Cập nhật phiên bản nginx từ 1.14.2 lên 1.16.1
        ports:
        - containerPort: 80
```

1. Apply file YAML mới:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/deployment-update.yaml
   ```

1. Quan sát deployment tạo các pod với tên mới và xóa các pod cũ:

   ```shell
   kubectl get pods -l app=nginx
   ```

## Mở rộng quy mô ứng dụng bằng cách tăng số replica (Scaling the application by increasing the replica count)

Bạn có thể tăng số lượng Pod trong Deployment của mình bằng cách apply một file YAML
mới. File YAML sau đặt `replicas` thành 4, tức là chỉ định Deployment cần có
bốn Pod:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 4 # Cập nhật replicas từ 2 lên 4
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.16.1
        ports:
        - containerPort: 80
```

1. Apply file YAML mới:

   ```shell
   kubectl apply -f https://k8s.io/examples/application/deployment-scale.yaml
   ```

1. Xác minh rằng Deployment có bốn Pod:

   ```shell
   kubectl get pods -l app=nginx
   ```

   Kết quả tương tự như sau:

   ```
   NAME                               READY     STATUS    RESTARTS   AGE
   nginx-deployment-148880595-4zdqq   1/1       Running   0          25s
   nginx-deployment-148880595-6zgi1   1/1       Running   0          25s
   nginx-deployment-148880595-fxcez   1/1       Running   0          2m
   nginx-deployment-148880595-rwovn   1/1       Running   0          2m
   ```

Để biết quy trình mở rộng quy mô chi tiết, bao gồm thu hẹp quy mô (scale down) và
đưa về không (scale to zero), hãy xem
[Scale một Deployment thủ công](346-scale-deployment-vi.md).

## Xóa một deployment (Deleting a deployment)

Xóa deployment theo tên:

```shell
kubectl delete deployment nginx-deployment
```

## ReplicationControllers -- cách làm cũ (ReplicationControllers -- the Old Way)

Cách được khuyến nghị để tạo một ứng dụng có nhiều bản sao (replicated application) là dùng
Deployment, và Deployment sẽ dùng ReplicaSet bên dưới. Trước khi Deployment và ReplicaSet
được thêm vào Kubernetes, các ứng dụng có nhiều bản sao được cấu hình bằng
[ReplicationController](70-replicationcontroller-vi.md).

## Tiếp theo (What's next)

- Tìm hiểu thêm về [đối tượng Deployment](63-deployment-vi.md).
- Tìm hiểu cách [cập nhật một Deployment mà không gây gián đoạn](348-update-deployment-rolling-vi.md).
