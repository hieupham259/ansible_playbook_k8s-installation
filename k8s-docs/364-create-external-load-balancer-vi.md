# Tạo bộ cân bằng tải bên ngoài (Create an External Load Balancer)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/

Trang này hướng dẫn cách tạo một bộ cân bằng tải bên ngoài (external load balancer).

Khi tạo một Service, bạn có tùy chọn tự động tạo một bộ cân bằng tải trên cloud.
Điều này cung cấp một địa chỉ IP truy cập được từ bên ngoài, gửi lưu lượng tới
đúng port trên các node của cluster,
_với điều kiện cluster của bạn chạy trong môi trường được hỗ trợ và được cấu hình với
đúng gói nhà cung cấp bộ cân bằng tải cloud (cloud load balancer provider package)_.

Bạn cũng có thể dùng Ingress thay cho Service.
Để biết thêm thông tin, hãy xem tài liệu
[Ingress](11-ingress-vi.md).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Cluster của bạn phải chạy trên cloud hoặc một môi trường khác đã hỗ trợ sẵn
việc cấu hình bộ cân bằng tải bên ngoài.

## Tạo một Service (Create a Service)

### Tạo Service từ manifest (Create a Service from a manifest)

Để tạo một bộ cân bằng tải bên ngoài, hãy thêm dòng sau vào
manifest của Service:

```yaml
    type: LoadBalancer
```

Khi đó manifest của bạn có thể trông như sau:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
spec:
  selector:
    app: example
  ports:
    - port: 8765
      targetPort: 9376
  type: LoadBalancer
```

### Tạo Service bằng kubectl (Create a Service using kubectl)

Bạn cũng có thể tạo service bằng lệnh `kubectl expose` với
flag `--type=LoadBalancer`:

```bash
kubectl expose deployment example --port=8765 --target-port=9376 \
        --name=example-service --type=LoadBalancer
```

Lệnh này tạo một Service mới dùng cùng các selector với tài nguyên được tham chiếu
(trong ví dụ trên là một Deployment có tên `example`).

Để biết thêm thông tin, bao gồm các flag tùy chọn, hãy xem
[tài liệu tham khảo `kubectl expose`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#expose).

## Tìm địa chỉ IP của bạn (Finding your IP address)

Bạn có thể tìm địa chỉ IP đã được tạo cho service của bạn bằng cách lấy thông tin
service thông qua `kubectl`:

```bash
kubectl describe services example-service
```

lệnh này sẽ cho ra kết quả tương tự như sau:

```
Name:                     example-service
Namespace:                default
Labels:                   app=example
Annotations:              <none>
Selector:                 app=example
Type:                     LoadBalancer
IP Families:              <none>
IP:                       10.3.22.96
IPs:                      10.3.22.96
LoadBalancer Ingress:     192.0.2.89
Port:                     <unset>  8765/TCP
TargetPort:               9376/TCP
NodePort:                 <unset>  30593/TCP
Endpoints:                172.17.0.3:9376
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Địa chỉ IP của bộ cân bằng tải được liệt kê bên cạnh mục `LoadBalancer Ingress`.

> **Ghi chú:**
> Nếu bạn chạy service của mình trên Minikube, bạn có thể tìm địa chỉ IP và port được gán bằng lệnh:
>
> ```bash
> minikube service example-service --url
> ```

## Giữ nguyên địa chỉ IP nguồn của client (Preserving the client source IP) {#preserving-the-client-source-ip}

Theo mặc định, địa chỉ IP nguồn mà container đích nhìn thấy *không phải là IP nguồn
gốc* của client. Để bật tính năng giữ nguyên IP của client, bạn có thể cấu hình
các trường sau trong `.spec` của Service:

* `.spec.externalTrafficPolicy` - cho biết Service này muốn định tuyến lưu lượng
  từ bên ngoài tới các endpoint cục bộ trên node (node-local) hay trên toàn cluster
  (cluster-wide). Có hai tùy chọn: `Cluster` (mặc định) và `Local`. `Cluster` che mất
  IP nguồn của client và có thể gây ra bước nhảy (hop) thứ hai sang node khác, nhưng
  nhìn chung phân tải tốt. `Local` giữ nguyên IP nguồn của client và tránh bước nhảy
  thứ hai đối với Service kiểu LoadBalancer và NodePort, nhưng có nguy cơ phân bổ
  lưu lượng không đồng đều.
* `.spec.healthCheckNodePort` - chỉ định node port dùng cho kiểm tra sức khỏe
  (health check) của service (là một số port). Nếu bạn không chỉ định
  `healthCheckNodePort`, service controller sẽ cấp phát một port từ dải NodePort
  của cluster.
  Bạn có thể cấu hình dải này bằng tùy chọn dòng lệnh của API server,
  `--service-node-port-range`. Service sẽ dùng giá trị `healthCheckNodePort`
  do người dùng chỉ định nếu bạn khai báo nó, với điều kiện `type` của Service
  được đặt là LoadBalancer và `externalTrafficPolicy` được đặt là `Local`.

Đặt `externalTrafficPolicy` thành Local trong manifest của Service
sẽ kích hoạt tính năng này. Ví dụ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-service
spec:
  selector:
    app: example
  ports:
    - port: 8765
      targetPort: 9376
  externalTrafficPolicy: Local
  type: LoadBalancer
```

### Lưu ý và hạn chế khi giữ nguyên IP nguồn (Caveats and limitations when preserving source IPs)

Dịch vụ cân bằng tải của một số nhà cung cấp cloud không cho phép bạn cấu hình trọng số (weight) khác nhau cho từng đích (target).

Khi mọi đích có trọng số bằng nhau trong việc gửi lưu lượng tới các Node, lưu lượng
từ bên ngoài sẽ không được cân bằng đều giữa các Pod khác nhau. Bộ cân bằng tải bên ngoài
không biết số lượng Pod trên mỗi node được dùng làm đích.

Khi `NumServicePods <<  NumNodes` hoặc `NumServicePods >> NumNodes`, sự phân bổ
sẽ gần như đồng đều, ngay cả khi không có trọng số.

Lưu lượng nội bộ giữa các Pod nên hoạt động tương tự Service kiểu ClusterIP, với xác suất bằng nhau trên tất cả các Pod.

## Thu dọn (garbage collect) các bộ cân bằng tải (Garbage collecting load balancers)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.17 [stable]`

Trong trường hợp thông thường, các tài nguyên bộ cân bằng tải tương ứng ở nhà cung cấp
cloud sẽ được dọn dẹp ngay sau khi Service kiểu LoadBalancer bị xóa. Nhưng đã có nhiều
trường hợp ngoại lệ (corner case) được ghi nhận, trong đó tài nguyên cloud bị bỏ rơi
(orphaned) sau khi Service liên quan đã bị xóa. Cơ chế Finalizer Protection cho
Service LoadBalancer được đưa vào để ngăn điều này xảy ra. Bằng cách dùng finalizer,
một tài nguyên Service sẽ không bao giờ bị xóa cho tới khi các tài nguyên bộ cân bằng
tải tương ứng cũng bị xóa.

Cụ thể, nếu một Service có `type` là LoadBalancer, service controller sẽ gắn
một finalizer tên là `service.kubernetes.io/load-balancer-cleanup`.
Finalizer này chỉ được gỡ bỏ sau khi tài nguyên bộ cân bằng tải đã được dọn dẹp.
Điều này ngăn tình trạng tài nguyên bộ cân bằng tải bị treo lơ lửng (dangling)
ngay cả trong các trường hợp ngoại lệ như service controller bị crash.

## Các nhà cung cấp bộ cân bằng tải bên ngoài (External load balancer providers)

Điều quan trọng cần lưu ý là đường đi dữ liệu (datapath) của chức năng này được cung cấp bởi một bộ cân bằng tải nằm bên ngoài cluster Kubernetes.

Khi `type` của Service được đặt là LoadBalancer, Kubernetes cung cấp chức năng tương đương
với `type` bằng ClusterIP cho các pod bên trong cluster, và mở rộng nó bằng cách lập trình
bộ cân bằng tải (nằm bên ngoài Kubernetes) với các mục (entry) trỏ tới các node đang chạy
các pod Kubernetes liên quan. Control plane của Kubernetes tự động hóa việc tạo bộ cân bằng
tải bên ngoài, các kiểm tra sức khỏe (nếu cần) và các quy tắc lọc gói tin (nếu cần). Sau khi
nhà cung cấp cloud cấp phát một địa chỉ IP cho bộ cân bằng tải, control plane sẽ tra cứu
địa chỉ IP bên ngoài đó và điền nó vào đối tượng Service.

## Tiếp theo (What's next)

* Làm theo hướng dẫn [Kết nối ứng dụng bằng Service](https://kubernetes.io/docs/tutorials/services/connect-applications-service/)
* Đọc về [Service](82-service-vi.md)
* Đọc về [Ingress](11-ingress-vi.md)
