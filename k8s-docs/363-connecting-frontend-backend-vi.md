# Kết nối Frontend với Backend bằng Service (Connect a Frontend to a Backend Using Services)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/access-application-cluster/connecting-frontend-backend/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 5 — Mạng nền tảng](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), dòng
**Thực hành**, bài 8/10 · Kiểm chứng ở
[Lab 5a — Service, EndpointSlice và DNS](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B12.

Ba bài cuối của dòng Thực hành là ba **đường vào ứng dụng khác nhau** và rất dễ nhầm. Bài này nối
hai tầng **bên trong** cluster bằng Service ClusterIP và tên DNS. Bài
[364](364-create-external-load-balancer-vi.md) là đường từ **bên ngoài** vào qua load balancer của
nhà cung cấp cloud. Bài [366](366-port-forward-vi.md) là đường **gỡ lỗi** từ máy trạm và không
dùng cho production. Đọc bài này để nắm đường thứ nhất.

**Phải hiểu ở lần đọc này:**

- Mục *Tạo đối tượng Service `hello`*: Service của backend mới là chìa khóa — nó tạo một địa chỉ
  IP bền vững và một bản ghi tên DNS, rồi dùng **selector** để tìm các Pod nhận lưu lượng.
- Mục *Tạo frontend*: frontend **không** biết IP nào của backend. File `frontend-nginx.conf` chỉ
  ghi `server hello;`, và `hello` chính là giá trị trường `name` của Service backend.
- Selector của Service `hello` gồm **hai** label `app: hello` và `tier: backend`. Cả hai
  Deployment đều mang `app: hello`, nên `tier` mới là thứ tách frontend khỏi backend.
- Ranh giới của Service backend: bài nói thẳng rằng sau khi tạo xong, service này "chưa thể truy
  cập được cũng như chưa phân giải được từ bên ngoài cluster". Muốn ra ngoài phải có một Service
  khác — ở đây là Service `frontend` với `type: LoadBalancer`.
- Mục *Trước khi bạn bắt đầu* cho phép thay Service có load balancer bên ngoài bằng Service kiểu
  [NodePort](82-service-vi.md#type-nodeport) khi môi trường không hỗ trợ. Cluster lab đúng vào
  trường hợp đó, nên đây là câu cần nhớ nhất của bài.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Tương tác với Service frontend* và *Gửi lưu lượng qua frontend* — chờ `EXTERNAL-IP` đổi từ `<pending>` sang IP thật rồi `curl` vào đó | cluster lab là kubeadm trên VM, không có cloud provider nào điền IP đó, nên hai bước này không chạy được nguyên xi | bài [364](364-create-external-load-balancer-vi.md) ngay sau đây giải thích ai điền IP; [Lab 5a](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) phần B6.3 chứng minh nó treo `<pending>`, còn B12 dựng lại bài này bằng NodePort |
| Ghi chú "nên dùng ConfigMap thay vì đóng gói cấu hình nginx vào image" | ở đây chỉ là lời khuyên thực hành tốt, bài không có bước làm nào cho nó | bài [275](275-configure-pod-configmap-vi.md), đã đọc ở giai đoạn 3 |
| Trường `lifecycle.preStop` trong `frontend-deployment.yaml` | không liên quan tới việc nối frontend với backend, bài không giải thích gì thêm | bài [42](42-container-lifecycle-hooks-vi.md), đã đọc ở giai đoạn 2 |

---

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
[Service với bộ cân bằng tải bên ngoài (external load balancer)](364-create-external-load-balancer-vi.md),
loại Service này yêu cầu một môi trường được hỗ trợ. Nếu môi trường của bạn không hỗ trợ,
bạn có thể dùng Service kiểu
[NodePort](82-service-vi.md#type-nodeport) thay thế.

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
> dùng một [ConfigMap](275-configure-pod-configmap-vi.md),
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

* Tìm hiểu thêm về [Service](82-service-vi.md)
* Tìm hiểu thêm về [ConfigMap](275-configure-pod-configmap-vi.md)
* Tìm hiểu thêm về [DNS cho Service và Pod](10-dns-pod-service-vi.md)

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5.

1. Cấu hình nginx của frontend chỉ có đúng một dòng `server hello;`. Chuỗi `hello` lấy từ đâu
   trong manifest, và thứ gì trong cluster biến chuỗi đó thành một đích gọi được?
2. Cả Deployment `backend` lẫn Deployment `frontend` đều mang label `app: hello`. Vì sao Service
   `hello` vẫn chỉ gửi lưu lượng tới Pod của backend?
3. **Câu bẫy.** Bạn đã tạo xong Deployment `backend` và Service `hello`, và `kubectl get svc` cho
   thấy Service đã có IP. Từ máy trạm của mình, bạn `curl` thẳng vào IP đó được không? Bài nói gì
   về điều này?
4. Cluster lab của bạn là kubeadm trên ba VM `lab-k8s-master`, `lab-k8s-worker1` và
   `lab-k8s-worker2`, không có cloud provider. Bạn apply nguyên xi `frontend-service.yaml`. Cột
   `EXTERNAL-IP` sẽ ra gì, và bài đã chuẩn bị sẵn cách thay thế nào?
5. Bạn scale Deployment `backend` từ 3 lên 5 replica. Có phải sửa `frontend-nginx.conf` và build
   lại image frontend không? Vì sao?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **`hello` là giá trị trường `name` của Service backend** trong
   `examples/service/access/backend-service.yaml` — bài nói thẳng rằng tên DNS cấp cho Service
   backend chính là trường `name` đó. Thứ biến nó thành đích gọi được là **bản thân đối tượng
   Service**: nó tạo một địa chỉ IP bền vững và một bản ghi tên DNS, nên frontend chỉ cần biết
   cái tên.
2. Vì **selector của Service `hello` có hai label chứ không phải một**: `app: hello` *và*
   `tier: backend`. Pod của frontend mang `tier: frontend` nên không khớp. Đây chính là lý do bài
   đặt thêm label `tier` vào cả hai Deployment.
3. **Không.** Bài viết thẳng: đến thời điểm đó "service này chưa thể truy cập được cũng như chưa
   phân giải được (resolvable) từ bên ngoài cluster". Chỗ dễ nhầm là thấy Service **đã có IP** thì
   tưởng IP ấy dùng được từ mọi nơi — IP và tên DNS đó chỉ có nghĩa **bên trong** cluster. Muốn
   gọi từ ngoài phải qua Service `frontend`, thứ duy nhất trong bài khai báo
   `type: LoadBalancer`.
4. `EXTERNAL-IP` sẽ nằm ở **`<pending>` và không bao giờ đổi**, vì bài nói rõ bộ cân bằng tải là
   thứ **do nhà cung cấp cloud cấp phát (provision)** — cluster lab không có ai làm việc đó. Cách
   thay thế nằm ngay ở mục *Trước khi bạn bắt đầu*: dùng Service kiểu
   [NodePort](82-service-vi.md#type-nodeport) thay cho Service có load balancer bên ngoài.
5. **Không.** Frontend chỉ tham chiếu **tên** của Service backend, còn việc tìm ra tập Pod nào
   đang đứng sau tên đó là việc của Service — nó dùng selector để tìm Pod. Đúng như mục tiêu bài
   đặt ra ở đầu: dùng đối tượng Service để gửi lưu lượng tới **nhiều bản sao** của microservice
   backend.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
