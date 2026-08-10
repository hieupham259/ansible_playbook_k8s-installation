# Kết nối Frontend với Backend bằng Service (Connect a Frontend to a Backend Using Services)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/access-application-cluster/connecting-frontend-backend/

Tác vụ này hướng dẫn cách tạo một microservice _frontend_ và một microservice _backend_.
Microservice backend là một dịch vụ chào hỏi (hello greeter). Frontend expose backend
bằng nginx và một đối tượng Service của Kubernetes.

## Mục tiêu (Objectives)

* Tạo và chạy một microservice backend `hello` mẫu bằng đối tượng Deployment.
* Dùng đối tượng Service để gửi lưu lượng tới nhiều bản sao (replica) của microservice backend.
* Tạo và chạy một microservice frontend `nginx`, cũng bằng đối tượng Deployment.
* Cấu hình microservice frontend để gửi lưu lượng tới microservice backend.
* Dùng đối tượng Service có `type=LoadBalancer` để expose microservice frontend
  ra bên ngoài cluster.

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

Tác vụ này sử dụng
[Service với bộ cân bằng tải bên ngoài (external load balancer)](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/),
loại Service này yêu cầu một môi trường được hỗ trợ. Nếu môi trường của bạn không hỗ trợ,
bạn có thể dùng Service kiểu
[NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) thay thế.

## Tạo backend bằng Deployment (Creating the backend using a Deployment)

Backend là một microservice chào hỏi đơn giản. Đây là file cấu hình
cho Deployment của backend:

**`service/access/backend-deployment.yaml`**

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  selector:
    matchLabels:
      app: hello
      tier: backend
      track: stable
  replicas: 3
  template:
    metadata:
      labels:
        app: hello
        tier: backend
        track: stable
    spec:
      containers:
        - name: hello
          image: "gcr.io/google-samples/hello-go-gke:1.0"
          ports:
            - name: http
              containerPort: 80
...
```

Tạo Deployment cho backend:

```shell
kubectl apply -f https://k8s.io/examples/service/access/backend-deployment.yaml
```

Xem thông tin về Deployment của backend:

```shell
kubectl describe deployment backend
```

Kết quả đầu ra (output) tương tự như sau:

```
Name:                           backend
Namespace:                      default
CreationTimestamp:              Mon, 24 Oct 2016 14:21:02 -0700
Labels:                         app=hello
                                tier=backend
                                track=stable
Annotations:                    deployment.kubernetes.io/revision=1
Selector:                       app=hello,tier=backend,track=stable
Replicas:                       3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:                   RollingUpdate
MinReadySeconds:                0
RollingUpdateStrategy:          1 max unavailable, 1 max surge
Pod Template:
  Labels:       app=hello
                tier=backend
                track=stable
  Containers:
   hello:
    Image:              "gcr.io/google-samples/hello-go-gke:1.0"
    Port:               80/TCP
    Environment:        <none>
    Mounts:             <none>
  Volumes:              <none>
Conditions:
  Type          Status  Reason
  ----          ------  ------
  Available     True    MinimumReplicasAvailable
  Progressing   True    NewReplicaSetAvailable
OldReplicaSets:                 <none>
NewReplicaSet:                  hello-3621623197 (3/3 replicas created)
Events:
...
```

## Tạo đối tượng Service `hello` (Creating the `hello` Service object)

Chìa khóa để gửi request từ frontend tới backend chính là Service của backend.
Một Service tạo ra một địa chỉ IP bền vững và một bản ghi tên DNS, nhờ đó
microservice backend luôn có thể được truy cập tới. Service dùng
các selector để tìm các Pod mà nó định tuyến lưu lượng đến.

Trước tiên, hãy xem file cấu hình của Service:

**`service/access/backend-service.yaml`**

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
    tier: backend
  ports:
  - protocol: TCP
    port: 80
    targetPort: http
...
```

Trong file cấu hình, bạn có thể thấy Service có tên `hello` định tuyến
lưu lượng tới các Pod mang label `app: hello` và `tier: backend`.

Tạo Service cho backend:

```shell
kubectl apply -f https://k8s.io/examples/service/access/backend-service.yaml
```

Đến thời điểm này, bạn đã có một Deployment `backend` chạy ba bản sao của ứng dụng
`hello`, và bạn có một Service có thể định tuyến lưu lượng tới chúng. Tuy nhiên,
service này chưa thể truy cập được cũng như chưa phân giải được (resolvable) từ bên ngoài cluster.

## Tạo frontend (Creating the frontend)

Bây giờ backend của bạn đã chạy, bạn có thể tạo một frontend có thể truy cập được
từ bên ngoài cluster và kết nối tới backend bằng cách proxy các request tới nó.

Frontend gửi request tới các Pod worker của backend bằng cách dùng tên DNS
được cấp cho Service của backend. Tên DNS đó là `hello`, chính là giá trị
của trường `name` trong file cấu hình `examples/service/access/backend-service.yaml`.

Các Pod trong Deployment của frontend chạy một image nginx được cấu hình
để proxy request tới Service backend `hello`. Đây là file cấu hình nginx:

**`service/access/frontend-nginx.conf`**

```
# Định danh Backend là tên nội bộ của nginx, dùng để đặt tên cho upstream cụ thể này
upstream Backend {
    # hello là tên DNS nội bộ mà Service backend sử dụng bên trong Kubernetes
    server hello;
}

server {
    listen 80;

    location / {
        # Câu lệnh sau sẽ proxy lưu lượng tới upstream có tên Backend
        proxy_pass http://Backend;
    }
}
```

Tương tự backend, frontend cũng có một Deployment và một Service. Một điểm khác biệt
quan trọng cần chú ý giữa Service của backend và của frontend là cấu hình
của Service frontend có `type: LoadBalancer`, nghĩa là Service này dùng một
bộ cân bằng tải (load balancer) do nhà cung cấp cloud của bạn cấp phát (provision)
và sẽ truy cập được từ bên ngoài cluster.

**`service/access/frontend-service.yaml`**

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    app: hello
    tier: frontend
  ports:
  - protocol: "TCP"
    port: 80
    targetPort: 80
  type: LoadBalancer
...
```

**`service/access/frontend-deployment.yaml`**

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
    matchLabels:
      app: hello
      tier: frontend
      track: stable
  replicas: 1
  template:
    metadata:
      labels:
        app: hello
        tier: frontend
        track: stable
    spec:
      containers:
        - name: nginx
          image: "gcr.io/google-samples/hello-frontend:1.0"
          lifecycle:
            preStop:
              exec:
                command: ["/usr/sbin/nginx","-s","quit"]
...
```

Tạo Deployment và Service cho frontend:

```shell
kubectl apply -f https://k8s.io/examples/service/access/frontend-deployment.yaml
kubectl apply -f https://k8s.io/examples/service/access/frontend-service.yaml
```

Kết quả đầu ra xác nhận cả hai tài nguyên đã được tạo:

```
deployment.apps/frontend created
service/frontend created
```

> **Ghi chú:**
> Cấu hình nginx được đóng gói sẵn (baked) trong
> [container image](https://kubernetes.io/examples/service/access/Dockerfile). Cách làm tốt hơn là
> dùng một [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/),
> để bạn có thể thay đổi cấu hình dễ dàng hơn.

## Tương tác với Service frontend (Interact with the frontend Service)

Sau khi bạn đã tạo một Service kiểu LoadBalancer, bạn có thể dùng lệnh sau
để tìm địa chỉ IP bên ngoài (external IP):

```shell
kubectl get service frontend --watch
```

Lệnh này hiển thị cấu hình của Service `frontend` và theo dõi các thay đổi.
Ban đầu, external IP được liệt kê là `<pending>`:

```
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)  AGE
frontend   LoadBalancer   10.51.252.116   <pending>     80/TCP   10s
```

Tuy nhiên, ngay khi một external IP được cấp phát, cấu hình sẽ được cập nhật
để hiển thị IP mới dưới cột `EXTERNAL-IP`:

```
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP        PORT(S)  AGE
frontend   LoadBalancer   10.51.252.116   XXX.XXX.XXX.XXX    80/TCP   1m
```

Giờ đây địa chỉ IP đó có thể được dùng để tương tác với service `frontend`
từ bên ngoài cluster.

## Gửi lưu lượng qua frontend (Send traffic through the frontend)

Frontend và backend giờ đã được kết nối với nhau. Bạn có thể truy cập endpoint
bằng lệnh curl trên external IP của Service frontend.

```shell
curl http://${EXTERNAL_IP} # thay bằng EXTERNAL-IP mà bạn đã thấy ở bước trước
```

Kết quả đầu ra hiển thị thông điệp do backend sinh ra:

```json
{"message":"Hello"}
```

## Dọn dẹp (Cleaning up)

Để xóa các Service, nhập lệnh sau:

```shell
kubectl delete services frontend backend
```

Để xóa các Deployment, các ReplicaSet và các Pod đang chạy ứng dụng backend và frontend, nhập lệnh sau:

```shell
kubectl delete deployment frontend backend
```

## Tiếp theo (What's next)

* Tìm hiểu thêm về [Service](https://kubernetes.io/docs/concepts/services-networking/service/)
* Tìm hiểu thêm về [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
* Tìm hiểu thêm về [DNS cho Service và Pod](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
