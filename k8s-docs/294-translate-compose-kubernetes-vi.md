# Chuyển đổi file Docker Compose thành tài nguyên Kubernetes (Translate a Docker Compose File to Kubernetes Resources)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/translate-compose-kubernetes/>

Kompose là gì? Đó là một công cụ chuyển đổi mọi thứ liên quan đến compose (cụ thể là Docker
Compose) sang các trình điều phối container (Kubernetes hoặc OpenShift).

Bạn có thể tìm thêm thông tin trên website của Kompose tại
[https://kompose.io/](https://kompose.io/).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Để kiểm tra phiên bản, hãy nhập `kubectl version`.

## Cài đặt Kompose (Install Kompose)

Chúng ta có nhiều cách để cài đặt Kompose. Cách được ưu tiên là tải binary từ bản phát hành
mới nhất trên GitHub.

#### Tải từ GitHub (GitHub download)

Kompose được phát hành qua GitHub theo chu kỳ ba tuần, bạn có thể xem tất cả các bản phát hành
hiện tại trên [trang phát hành GitHub](https://github.com/kubernetes/kompose/releases).

```sh
# Linux
curl -L https://github.com/kubernetes/kompose/releases/download/v1.34.0/kompose-linux-amd64 -o kompose

# macOS
curl -L https://github.com/kubernetes/kompose/releases/download/v1.34.0/kompose-darwin-amd64 -o kompose

# Windows
curl -L https://github.com/kubernetes/kompose/releases/download/v1.34.0/kompose-windows-amd64.exe -o kompose.exe

chmod +x kompose
sudo mv ./kompose /usr/local/bin/kompose
```

Ngoài ra, bạn có thể tải [tarball](https://github.com/kubernetes/kompose/releases).

#### Build từ mã nguồn (Build from source)

Cài đặt bằng `go get` sẽ kéo mã từ nhánh master với những thay đổi phát triển mới nhất.

```sh
go get -u github.com/kubernetes/kompose
```

#### Homebrew (macOS)

Trên macOS bạn có thể cài đặt bản phát hành mới nhất qua [Homebrew](https://brew.sh):

```bash
brew install kompose
```

## Sử dụng Kompose (Use Kompose)

Chỉ trong vài bước, chúng tôi sẽ đưa bạn từ Docker Compose đến Kubernetes. Tất cả những gì bạn
cần là một file `docker-compose.yml` sẵn có.

1. Đi tới thư mục chứa file `docker-compose.yml` của bạn. Nếu bạn chưa có, hãy thử nghiệm bằng
   file này.

   ```yaml

   services:

     redis-leader:
       container_name: redis-leader
       image: redis
       ports:
         - "6379"

     redis-replica:
       container_name: redis-replica
       image: redis
       ports:
         - "6379"
       command: redis-server --replicaof redis-leader 6379 --dir /tmp

     web:
       container_name: web
       image: quay.io/kompose/web
       ports:
         - "8080:8080"
       environment:
         - GET_HOSTS_FROM=dns
       labels:
         kompose.service.type: LoadBalancer
   ```

2. Để chuyển đổi file `docker-compose.yml` thành các file mà bạn có thể dùng với `kubectl`,
   hãy chạy `kompose convert` rồi chạy `kubectl apply -f <file output>`.

   ```bash
   kompose convert
   ```

   Output tương tự như sau:

   ```none
   INFO Kubernetes file "redis-leader-service.yaml" created
   INFO Kubernetes file "redis-replica-service.yaml" created
   INFO Kubernetes file "web-tcp-service.yaml" created
   INFO Kubernetes file "redis-leader-deployment.yaml" created
   INFO Kubernetes file "redis-replica-deployment.yaml" created
   INFO Kubernetes file "web-deployment.yaml" created
   ```

   ```bash
    kubectl apply -f web-tcp-service.yaml,redis-leader-service.yaml,redis-replica-service.yaml,web-deployment.yaml,redis-leader-deployment.yaml,redis-replica-deployment.yaml
   ```

   Output tương tự như sau:

   ```none
   deployment.apps/redis-leader created
   deployment.apps/redis-replica created
   deployment.apps/web created
   service/redis-leader created
   service/redis-replica created
   service/web-tcp created
   ```

    Các Deployment của bạn đang chạy trong Kubernetes.

3. Truy cập ứng dụng của bạn.

   Nếu bạn đã dùng `minikube` trong quy trình phát triển của mình:

   ```bash
   minikube service web-tcp
   ```

   Nếu không, hãy tra xem Service của bạn đang dùng IP nào!

   ```sh
   kubectl describe svc web-tcp
   ```

   ```none
    Name:                     web-tcp
    Namespace:                default
    Labels:                   io.kompose.service=web-tcp
    Annotations:              kompose.cmd: kompose convert
                              kompose.service.type: LoadBalancer
                              kompose.version: 1.33.0 (3ce457399)
    Selector:                 io.kompose.service=web
    Type:                     LoadBalancer
    IP Family Policy:         SingleStack
    IP Families:              IPv4
    IP:                       10.102.30.3
    IPs:                      10.102.30.3
    Port:                     8080  8080/TCP
    TargetPort:               8080/TCP
    NodePort:                 8080  31624/TCP
    Endpoints:                10.244.0.5:8080
    Session Affinity:         None
    External Traffic Policy:  Cluster
    Events:                   <none>
   ```

   Nếu bạn đang dùng một nhà cung cấp cloud, IP của bạn sẽ được liệt kê cạnh
   `LoadBalancer Ingress`.

   ```sh
   curl http://192.0.2.89
   ```

4. Dọn dẹp.

   Sau khi bạn thử nghiệm xong bản triển khai ứng dụng ví dụ, chỉ cần chạy lệnh sau trong
   shell của bạn để xóa các tài nguyên đã dùng.

   ```sh
   kubectl delete -f web-tcp-service.yaml,redis-leader-service.yaml,redis-replica-service.yaml,web-deployment.yaml,redis-leader-deployment.yaml,redis-replica-deployment.yaml
   ```

## Hướng dẫn sử dụng (User Guide)

- CLI
  - [`kompose convert`](#kompose-convert)
- Tài liệu
  - [Các kiểu chuyển đổi thay thế](#các-kiểu-chuyển-đổi-thay-thế-alternative-conversions)
  - [Label](#label-labels)
  - [Restart](#restart)
  - [Các phiên bản Docker Compose](#các-phiên-bản-docker-compose-docker-compose-versions)

Kompose hỗ trợ hai provider: OpenShift và Kubernetes. Bạn có thể chọn provider đích bằng tùy
chọn toàn cục `--provider`. Nếu không chỉ định provider nào, Kubernetes được đặt làm mặc định.

## `kompose convert`

Kompose hỗ trợ chuyển đổi các file Docker Compose V1, V2 và V3 thành các object của Kubernetes
và OpenShift.

### Ví dụ `kompose convert` với Kubernetes (Kubernetes `kompose convert` example)

```shell
kompose --file docker-voting.yml convert
```

```none
WARN Unsupported key networks - ignoring
WARN Unsupported key build - ignoring
INFO Kubernetes file "worker-svc.yaml" created
INFO Kubernetes file "db-svc.yaml" created
INFO Kubernetes file "redis-svc.yaml" created
INFO Kubernetes file "result-svc.yaml" created
INFO Kubernetes file "vote-svc.yaml" created
INFO Kubernetes file "redis-deployment.yaml" created
INFO Kubernetes file "result-deployment.yaml" created
INFO Kubernetes file "vote-deployment.yaml" created
INFO Kubernetes file "worker-deployment.yaml" created
INFO Kubernetes file "db-deployment.yaml" created
```

```shell
ls
```

```none
db-deployment.yaml  docker-compose.yml         docker-gitlab.yml  redis-deployment.yaml  result-deployment.yaml  vote-deployment.yaml  worker-deployment.yaml
db-svc.yaml         docker-voting.yml          redis-svc.yaml     result-svc.yaml        vote-svc.yaml           worker-svc.yaml
```

Bạn cũng có thể cung cấp nhiều file docker-compose cùng một lúc:

```shell
kompose -f docker-compose.yml -f docker-guestbook.yml convert
```

```none
INFO Kubernetes file "frontend-service.yaml" created         
INFO Kubernetes file "mlbparks-service.yaml" created         
INFO Kubernetes file "mongodb-service.yaml" created          
INFO Kubernetes file "redis-master-service.yaml" created     
INFO Kubernetes file "redis-slave-service.yaml" created      
INFO Kubernetes file "frontend-deployment.yaml" created      
INFO Kubernetes file "mlbparks-deployment.yaml" created      
INFO Kubernetes file "mongodb-deployment.yaml" created       
INFO Kubernetes file "mongodb-claim0-persistentvolumeclaim.yaml" created
INFO Kubernetes file "redis-master-deployment.yaml" created  
INFO Kubernetes file "redis-slave-deployment.yaml" created   
```

```shell
ls
```

```none
mlbparks-deployment.yaml  mongodb-service.yaml                       redis-slave-service.jsonmlbparks-service.yaml  
frontend-deployment.yaml  mongodb-claim0-persistentvolumeclaim.yaml  redis-master-service.yaml
frontend-service.yaml     mongodb-deployment.yaml                    redis-slave-deployment.yaml
redis-master-deployment.yaml
```

Khi nhiều file docker-compose được cung cấp, cấu hình sẽ được gộp (merge) lại. Bất kỳ cấu hình
nào trùng nhau sẽ bị file sau ghi đè.

### Ví dụ `kompose convert` với OpenShift (OpenShift `kompose convert` example)

```sh
kompose --provider openshift --file docker-voting.yml convert
```

```none
WARN [worker] Service cannot be created because of missing port.
INFO OpenShift file "vote-service.yaml" created             
INFO OpenShift file "db-service.yaml" created               
INFO OpenShift file "redis-service.yaml" created            
INFO OpenShift file "result-service.yaml" created           
INFO OpenShift file "vote-deploymentconfig.yaml" created    
INFO OpenShift file "vote-imagestream.yaml" created         
INFO OpenShift file "worker-deploymentconfig.yaml" created  
INFO OpenShift file "worker-imagestream.yaml" created       
INFO OpenShift file "db-deploymentconfig.yaml" created      
INFO OpenShift file "db-imagestream.yaml" created           
INFO OpenShift file "redis-deploymentconfig.yaml" created   
INFO OpenShift file "redis-imagestream.yaml" created        
INFO OpenShift file "result-deploymentconfig.yaml" created  
INFO OpenShift file "result-imagestream.yaml" created  
```

Kompose cũng hỗ trợ tạo buildconfig cho chỉ thị build trong một service. Theo mặc định, nó
dùng repo từ xa (remote repo) của nhánh git hiện tại làm repo nguồn, và nhánh hiện tại làm
nhánh nguồn cho việc build. Bạn có thể chỉ định repo nguồn và nhánh khác bằng các tùy chọn
``--build-repo`` và ``--build-branch`` tương ứng.

```sh
kompose --provider openshift --file buildconfig/docker-compose.yml convert
```

```none
WARN [foo] Service cannot be created because of missing port.
INFO OpenShift Buildconfig using git@github.com:rtnpro/kompose.git::master as source.
INFO OpenShift file "foo-deploymentconfig.yaml" created     
INFO OpenShift file "foo-imagestream.yaml" created          
INFO OpenShift file "foo-buildconfig.yaml" created
```

> **Ghi chú:** Nếu bạn đẩy các artifact OpenShift theo cách thủ công bằng ``oc create -f``,
> bạn cần đảm bảo đẩy artifact imagestream trước artifact buildconfig, để khắc phục issue này
> của OpenShift: https://github.com/openshift/origin/issues/4518 .

## Các kiểu chuyển đổi thay thế (Alternative Conversions)

Phép chuyển đổi mặc định của `kompose` sẽ sinh ra các [Deployment](./63-deployment-vi.md) và
[Service](./82-service-vi.md) của Kubernetes, ở định dạng yaml. Bạn có tùy chọn thay thế để
sinh json với `-j`. Ngoài ra, bạn cũng có thể sinh các object
[Replication Controller](./70-replicationcontroller-vi.md),
[DaemonSet](./66-daemonset-vi.md), hoặc [Helm](https://github.com/helm/helm) chart.

```sh
kompose convert -j
INFO Kubernetes file "redis-svc.json" created
INFO Kubernetes file "web-svc.json" created
INFO Kubernetes file "redis-deployment.json" created
INFO Kubernetes file "web-deployment.json" created
```

Các file `*-deployment.json` chứa các object Deployment.

```sh
kompose convert --replication-controller
INFO Kubernetes file "redis-svc.yaml" created
INFO Kubernetes file "web-svc.yaml" created
INFO Kubernetes file "redis-replicationcontroller.yaml" created
INFO Kubernetes file "web-replicationcontroller.yaml" created
```

Các file `*-replicationcontroller.yaml` chứa các object Replication Controller. Nếu bạn muốn
chỉ định số replica (mặc định là 1), hãy dùng cờ `--replicas`:
`kompose convert --replication-controller --replicas 3`.

```shell
kompose convert --daemon-set
INFO Kubernetes file "redis-svc.yaml" created
INFO Kubernetes file "web-svc.yaml" created
INFO Kubernetes file "redis-daemonset.yaml" created
INFO Kubernetes file "web-daemonset.yaml" created
```

Các file `*-daemonset.yaml` chứa các object DaemonSet.

Nếu bạn muốn sinh một Chart để dùng với [Helm](https://github.com/kubernetes/helm), hãy chạy:

```shell
kompose convert -c
```

```none
INFO Kubernetes file "web-svc.yaml" created
INFO Kubernetes file "redis-svc.yaml" created
INFO Kubernetes file "web-deployment.yaml" created
INFO Kubernetes file "redis-deployment.yaml" created
chart created in "./docker-compose/"
```

```shell
tree docker-compose/
```

```none
docker-compose
├── Chart.yaml
├── README.md
└── templates
    ├── redis-deployment.yaml
    ├── redis-svc.yaml
    ├── web-deployment.yaml
    └── web-svc.yaml
```

Cấu trúc chart này nhằm cung cấp một bộ khung (skeleton) để bạn xây dựng các Helm chart của
mình.

## Label (Labels)

`kompose` hỗ trợ các label dành riêng cho Kompose bên trong file `docker-compose.yml` để định
nghĩa tường minh hành vi của một service khi chuyển đổi.

- `kompose.service.type` định nghĩa loại Service sẽ được tạo.

  Ví dụ:

  ```yaml
  version: "2"
  services:
    nginx:
      image: nginx
      dockerfile: foobar
      build: ./foobar
      cap_add:
        - ALL
      container_name: foobar
      labels:
        kompose.service.type: nodeport
  ```

- `kompose.service.expose` định nghĩa service có cần được truy cập từ bên ngoài cluster hay
  không. Nếu giá trị được đặt là "true", provider sẽ tự động thiết lập endpoint, còn với bất
  kỳ giá trị nào khác, giá trị đó được đặt làm hostname. Nếu nhiều port được định nghĩa trong
  một service, port đầu tiên sẽ được chọn để expose.
  - Với provider Kubernetes, một tài nguyên Ingress sẽ được tạo, và giả định rằng một ingress
    controller đã được cấu hình sẵn.
  - Với provider OpenShift, một route sẽ được tạo.

  Ví dụ:

  ```yaml
  version: "2"
  services:
    web:
      image: tuna/docker-counter23
      ports:
      - "5000:5000"
      links:
      - redis
      labels:
        kompose.service.expose: "counter.example.com"
    redis:
      image: redis:3.0
      ports:
      - "6379"
  ```

Các tùy chọn hiện được hỗ trợ là:

| Key                  | Giá trị                             |
|----------------------|-------------------------------------|
| kompose.service.type | nodeport / clusterip / loadbalancer |
| kompose.service.expose| true / hostname |

> **Ghi chú:** Label `kompose.service.type` chỉ nên được định nghĩa cùng với `ports`, nếu
> không `kompose` sẽ thất bại.

## Restart

Nếu bạn muốn tạo các Pod thông thường không có controller, bạn có thể dùng cấu trúc `restart`
của docker-compose để định nghĩa điều đó. Hãy theo bảng dưới đây để xem điều gì xảy ra với
từng giá trị `restart`.

| `docker-compose` `restart` | object được tạo   | `restartPolicy` của Pod |
|----------------------------|-------------------|---------------------|
| `""`                       | object controller | `Always`            |
| `always`                   | object controller | `Always`            |
| `on-failure`               | Pod               | `OnFailure`         |
| `no`                       | Pod               | `Never`             |

> **Ghi chú:** Object controller có thể là `deployment` hoặc `replicationcontroller`.

Ví dụ, service `pival` dưới đây sẽ trở thành một Pod. Container này tính toán giá trị của
`pi`.

```yaml
version: '2'

services:
  pival:
    image: perl
    command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
    restart: "on-failure"
```

### Cảnh báo về cấu hình Deployment (Warning about Deployment Configurations)

Nếu file Docker Compose có một volume được chỉ định cho một service, chiến lược (strategy) của
Deployment (Kubernetes) hoặc DeploymentConfig (OpenShift) sẽ được đổi thành "Recreate" thay vì
"RollingUpdate" (mặc định). Điều này nhằm tránh việc nhiều instance của một service truy cập
cùng một volume tại cùng thời điểm.

Nếu file Docker Compose có tên service chứa `_` (ví dụ, `web_service`), thì ký tự đó sẽ bị
thay bằng `-` và tên service sẽ được đổi tên tương ứng (ví dụ, `web-service`). Kompose làm vậy
vì "Kubernetes" không cho phép `_` trong tên object.

Xin lưu ý rằng việc thay đổi tên service có thể làm hỏng một số file `docker-compose`.

## Các phiên bản Docker Compose (Docker Compose Versions)

Kompose hỗ trợ các phiên bản Docker Compose: 1, 2 và 3. Chúng tôi hỗ trợ hạn chế các phiên bản
2.1 và 3.2 do tính chất thử nghiệm của chúng.

Danh sách đầy đủ về tính tương thích giữa cả ba phiên bản được liệt kê trong
[tài liệu chuyển đổi](https://github.com/kubernetes/kompose/blob/master/docs/conversion.md)
của chúng tôi, bao gồm danh sách tất cả các key Docker Compose không tương thích.
