# Service

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/service/>
>
> Expose một ứng dụng đang chạy trong cluster của bạn ra phía sau một endpoint
> hướng ra ngoài duy nhất, ngay cả khi workload được chia thành nhiều backend.

Trong Kubernetes, Service là một phương thức để expose một ứng dụng mạng đang chạy
dưới dạng một hoặc nhiều Pod trong cluster của bạn.

Một mục tiêu then chốt của Service trong Kubernetes là bạn không cần phải sửa đổi
ứng dụng hiện có của mình để dùng một cơ chế khám phá service (service discovery) xa lạ.
Bạn có thể chạy code trong các Pod, dù đó là code được thiết kế cho thế giới cloud-native,
hay một ứng dụng cũ mà bạn đã đóng gói vào container. Bạn dùng một Service để làm cho
tập hợp Pod đó khả dụng trên mạng, nhờ vậy các client có thể tương tác với nó.

Nếu bạn dùng một [Deployment](./63-deployment-vi.md) để chạy ứng dụng,
Deployment đó có thể tạo và hủy các Pod một cách linh hoạt. Từ thời điểm này sang
thời điểm khác, bạn không biết có bao nhiêu Pod đang hoạt động và khỏe mạnh; thậm chí
bạn có thể không biết những Pod khỏe mạnh đó tên là gì.
Các Pod của Kubernetes được tạo ra và hủy đi để khớp với trạng thái mong muốn
của cluster. Pod là tài nguyên phù du (bạn không nên kỳ vọng rằng một Pod
riêng lẻ là đáng tin cậy và bền vững).

Mỗi Pod nhận địa chỉ IP riêng của nó (Kubernetes kỳ vọng các network plugin đảm bảo điều này).
Với một Deployment cho trước trong cluster của bạn, tập hợp Pod đang chạy tại một thời điểm
có thể khác với tập hợp Pod đang chạy ứng dụng đó ở thời điểm ngay sau.

Điều này dẫn đến một vấn đề: nếu một tập hợp Pod nào đó (gọi là "backend") cung cấp
chức năng cho các Pod khác (gọi là "frontend") bên trong cluster của bạn,
làm sao các frontend tìm ra và theo dõi được địa chỉ IP nào cần kết nối tới,
để frontend có thể sử dụng phần backend của workload?

Đó là lúc _Service_ xuất hiện.

## Service trong Kubernetes (Services in Kubernetes)

Service API, một phần của Kubernetes, là một lớp trừu tượng giúp bạn expose các nhóm
Pod qua mạng. Mỗi đối tượng Service định nghĩa một tập hợp logic các endpoint (thường
các endpoint này là Pod) cùng với một chính sách về cách làm cho các Pod đó có thể truy cập được.

Ví dụ, hãy xét một backend xử lý ảnh không trạng thái (stateless) đang chạy với
3 bản sao (replica). Các bản sao này có thể hoán đổi cho nhau — các frontend không quan tâm
chúng dùng backend nào. Mặc dù các Pod thực tế tạo nên tập backend có thể thay đổi,
các client frontend không cần phải biết điều đó, cũng không cần tự theo dõi
tập hợp các backend.

Lớp trừu tượng Service cho phép sự tách rời (decoupling) này.

Tập hợp Pod mà một Service nhắm tới thường được xác định
bởi một selector mà bạn định nghĩa.
Để tìm hiểu các cách khác để định nghĩa endpoint cho Service,
xem [Service _không có_ selector](#services-without-selectors).

Nếu workload của bạn nói HTTP, bạn có thể chọn dùng một
[Ingress](./11-ingress-vi.md) để điều khiển cách web traffic
đến được workload đó.
Ingress không phải là một loại Service, nhưng nó đóng vai trò điểm vào (entry point)
cho cluster của bạn. Ingress cho phép bạn hợp nhất các quy tắc định tuyến vào một tài nguyên
duy nhất, nhờ đó bạn có thể expose nhiều thành phần của workload, chạy tách biệt
trong cluster, phía sau một listener duy nhất.

[Gateway](https://gateway-api.sigs.k8s.io/#what-is-the-gateway-api) API cho Kubernetes
cung cấp các khả năng bổ sung ngoài Ingress và Service. Bạn có thể thêm Gateway vào cluster —
đây là một họ các API mở rộng, được triển khai bằng
CustomResourceDefinition —
rồi dùng chúng để cấu hình việc truy cập các dịch vụ mạng đang chạy trong cluster của bạn.

### Khám phá service kiểu cloud-native (Cloud-native service discovery)

Nếu bạn có thể dùng các API của Kubernetes cho việc khám phá service trong ứng dụng của mình,
bạn có thể truy vấn API server
để tìm các EndpointSlice khớp. Kubernetes cập nhật các EndpointSlice của một Service
mỗi khi tập hợp Pod trong Service đó thay đổi.

Với các ứng dụng không phải cloud-native, Kubernetes cung cấp các cách để đặt một network port
hoặc bộ cân bằng tải (load balancer) vào giữa ứng dụng của bạn và các Pod backend.

Dù theo cách nào, workload của bạn cũng có thể dùng các cơ chế
[khám phá service](#discovering-services) này để tìm mục tiêu mà nó muốn kết nối tới.

## Định nghĩa một Service (Defining a Service)

Service là một đối tượng (object)
(giống như Pod hay ConfigMap là các đối tượng). Bạn có thể tạo,
xem hoặc sửa các định nghĩa Service bằng Kubernetes API. Thông thường
bạn dùng một công cụ như `kubectl` để thực hiện các lời gọi API đó thay cho bạn.

Ví dụ, giả sử bạn có một tập hợp Pod, mỗi Pod lắng nghe trên TCP port 9376
và được gắn nhãn (label) `app.kubernetes.io/name=MyApp`. Bạn có thể định nghĩa một Service để
công bố TCP listener đó:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

Áp dụng manifest này sẽ tạo một Service mới tên "my-service" với
[loại service](#publishing-services-service-types) mặc định là ClusterIP. Service này
nhắm tới TCP port 9376 trên bất kỳ Pod nào có nhãn `app.kubernetes.io/name: MyApp`.

Kubernetes gán cho Service này một địa chỉ IP (gọi là _cluster IP_),
địa chỉ này được cơ chế địa chỉ IP ảo (virtual IP) sử dụng. Để biết thêm chi tiết về cơ chế đó,
đọc [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/).

Controller của Service đó liên tục quét tìm các Pod khớp với
selector của nó, rồi thực hiện mọi cập nhật cần thiết cho tập hợp
EndpointSlice của Service.

Tên của một đối tượng Service phải là một
[tên nhãn RFC 1123](./17-names-vi.md#rfc-1123-label-names) hợp lệ.

> **Ghi chú:**
> Một Service có thể ánh xạ _bất kỳ_ `port` đến nào sang một `targetPort`. Theo mặc định và
> để thuận tiện, `targetPort` được đặt cùng giá trị với trường `port`.

### Định nghĩa port (Port definitions) {#field-spec-ports}

Các định nghĩa port trong Pod có tên, và bạn có thể tham chiếu các tên này trong
thuộc tính `targetPort` của một Service. Ví dụ, chúng ta có thể gắn `targetPort`
của Service với port của Pod theo cách sau:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app.kubernetes.io/name: proxy
  ports:
  - name: name-of-service-port
    protocol: TCP
    port: 80
    targetPort: http-web-svc

---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: proxy
spec:
  containers:
  - name: nginx
    image: nginx:stable
    ports:
      - containerPort: 80
        name: http-web-svc
```

Cách này hoạt động ngay cả khi trong Service có hỗn hợp nhiều Pod cùng dùng một
tên đã cấu hình, với cùng một giao thức mạng khả dụng qua các số port
khác nhau. Điều này mang lại rất nhiều linh hoạt cho việc triển khai và phát triển
các Service của bạn. Ví dụ, bạn có thể thay đổi số port mà các Pod expose
trong phiên bản kế tiếp của phần mềm backend, mà không làm hỏng các client.

Giao thức mặc định cho Service là
[TCP](https://kubernetes.io/docs/reference/networking/service-protocols/#protocol-tcp); bạn cũng có thể
dùng bất kỳ [giao thức được hỗ trợ](https://kubernetes.io/docs/reference/networking/service-protocols/) nào khác.

Vì nhiều Service cần expose nhiều hơn một port, Kubernetes hỗ trợ
[nhiều định nghĩa port](#multi-port-services) cho một Service duy nhất.
Mỗi định nghĩa port có thể có cùng `protocol`, hoặc khác nhau.

### Service không có selector (Services without selectors)

Service thường trừu tượng hóa việc truy cập các Pod của Kubernetes nhờ selector,
nhưng khi được dùng cùng một tập hợp đối tượng
EndpointSlice
tương ứng và không có selector, Service có thể trừu tượng hóa các loại backend khác,
bao gồm cả những backend chạy bên ngoài cluster.

Ví dụ:

* Bạn muốn dùng một cluster cơ sở dữ liệu bên ngoài ở môi trường production, nhưng trong
  môi trường test bạn dùng cơ sở dữ liệu của riêng mình.
* Bạn muốn trỏ Service của mình tới một Service trong một
  namespace khác hoặc trên một cluster khác.
* Bạn đang di chuyển (migrate) một workload sang Kubernetes. Trong lúc đánh giá cách tiếp cận,
  bạn chỉ chạy một phần các backend trong Kubernetes.

Trong bất kỳ kịch bản nào kể trên, bạn có thể định nghĩa một Service _không_ chỉ định
selector để khớp Pod. Ví dụ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
```

Vì Service này không có selector, các đối tượng EndpointSlice tương ứng
không được tạo tự động. Bạn có thể ánh xạ Service
tới địa chỉ mạng và port nơi nó đang chạy, bằng cách thêm thủ công một đối tượng
EndpointSlice. Ví dụ:

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-1 # theo quy ước, dùng tên của Service
                     # làm tiền tố cho tên của EndpointSlice
  labels:
    # Bạn nên đặt nhãn "kubernetes.io/service-name".
    # Đặt giá trị của nó khớp với tên của Service
    kubernetes.io/service-name: my-service
addressType: IPv4
ports:
  - name: http # nên khớp với tên của port trong Service đã định nghĩa ở trên
    appProtocol: http
    protocol: TCP
    port: 9376
endpoints:
  - addresses:
      - "10.4.5.6"
  - addresses:
      - "10.1.2.3"
```

#### EndpointSlice tùy chỉnh (Custom EndpointSlices)

Khi bạn tạo một đối tượng [EndpointSlice](#endpointslices) cho một Service, bạn có thể
dùng bất kỳ tên nào cho EndpointSlice. Mỗi EndpointSlice trong một namespace phải có
tên duy nhất. Bạn liên kết một EndpointSlice với một Service bằng cách đặt nhãn
`kubernetes.io/service-name`
trên EndpointSlice đó.

> **Ghi chú:**
> Các IP của endpoint _không được_ là: loopback (127.0.0.0/8 cho IPv4, ::1/128 cho IPv6), hoặc
> link-local (169.254.0.0/16 và 224.0.0.0/24 cho IPv4, fe80::/64 cho IPv6).
>
> Địa chỉ IP của endpoint không thể là cluster IP của các Service Kubernetes khác,
> vì kube-proxy không hỗ trợ IP ảo
> làm đích đến.

Với một EndpointSlice mà bạn tự tạo, hoặc tạo trong code của riêng bạn,
bạn cũng nên chọn một giá trị để dùng cho nhãn
[`endpointslice.kubernetes.io/managed-by`](https://kubernetes.io/docs/reference/labels-annotations-taints/#endpointslicekubernetesiomanaged-by).
Nếu bạn tự viết code controller để quản lý các EndpointSlice, hãy cân nhắc dùng một
giá trị tương tự `"my-domain.example/name-of-controller"`. Nếu bạn dùng một công cụ
bên thứ ba, hãy dùng tên công cụ viết thường toàn bộ và đổi khoảng trắng cùng các
dấu câu khác thành dấu gạch ngang (`-`).
Nếu mọi người trực tiếp dùng một công cụ như `kubectl` để quản lý các EndpointSlice,
hãy dùng một tên mô tả việc quản lý thủ công này, chẳng hạn `"staff"` hoặc
`"cluster-admins"`. Bạn nên
tránh dùng giá trị dành riêng `"controller"`, giá trị này định danh các EndpointSlice
được quản lý bởi chính control plane của Kubernetes.

#### Truy cập một Service không có selector (Accessing a Service without a selector) {#service-no-selector-access}

Việc truy cập một Service không có selector hoạt động y hệt như khi nó có selector.
Trong [ví dụ](#services-without-selectors) về Service không có selector,
traffic được định tuyến tới một trong hai endpoint được định nghĩa trong
manifest EndpointSlice: một kết nối TCP tới 10.1.2.3 hoặc 10.4.5.6, trên port 9376.

> **Ghi chú:**
> Kubernetes API server không cho phép proxy tới các endpoint không được ánh xạ tới
> Pod. Các hành động như `kubectl port-forward service/<service-name> forwardedPort:servicePort` khi service không có
> selector sẽ thất bại do ràng buộc này. Điều này ngăn Kubernetes API server
> bị lợi dụng làm proxy tới các endpoint mà bên gọi có thể không được phép truy cập.

Service kiểu `ExternalName` là một trường hợp đặc biệt của Service không có
selector và dùng tên DNS thay thế. Để biết thêm thông tin, xem
mục [ExternalName](#externalname).

### EndpointSlices

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [stable]`

[EndpointSlice](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) là các đối tượng
đại diện cho một tập con (một _lát cắt_ — slice) của các network endpoint phía sau một Service.

Cluster Kubernetes của bạn theo dõi mỗi EndpointSlice đại diện cho bao nhiêu endpoint.
Nếu một Service có quá nhiều endpoint đến mức chạm ngưỡng, thì
Kubernetes thêm một EndpointSlice rỗng khác và lưu thông tin endpoint mới
ở đó.
Theo mặc định, Kubernetes tạo một EndpointSlice mới khi tất cả các EndpointSlice hiện có
đều đã chứa ít nhất 100 endpoint. Kubernetes không tạo EndpointSlice mới
cho tới khi cần thêm một endpoint nữa.

Xem [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) để biết thêm
thông tin về API này.

### Endpoints (deprecated) {#endpoints}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [deprecated]`

EndpointSlice API là bước tiến hóa của API
[Endpoints](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/endpoints-v1/)
cũ hơn. API Endpoints đã lỗi thời (deprecated) có một số vấn đề so với
EndpointSlice:

  - Nó không hỗ trợ cluster dual-stack.
  - Nó không chứa thông tin cần thiết để hỗ trợ các tính năng mới hơn, chẳng hạn
    [trafficDistribution](https://kubernetes.io/docs/concepts/services-networking/service/#traffic-distribution).
  - Nó sẽ cắt bớt danh sách endpoint nếu danh sách quá dài, không vừa trong một đối tượng duy nhất.

Vì vậy, khuyến nghị là mọi client nên dùng
EndpointSlice API thay vì Endpoints.

#### Endpoint vượt dung lượng (Over-capacity endpoints)

Kubernetes giới hạn số endpoint có thể chứa trong một đối tượng Endpoints
duy nhất. Khi một Service có hơn 1000 endpoint phía sau, Kubernetes
cắt bớt dữ liệu trong đối tượng Endpoints. Vì một Service có thể được liên kết
với nhiều EndpointSlice, giới hạn 1000 endpoint phía sau chỉ
ảnh hưởng tới API Endpoints cũ.

Trong trường hợp đó, Kubernetes chọn tối đa 1000 endpoint backend khả dĩ để lưu
vào đối tượng Endpoints, và đặt một annotation trên Endpoints:
[`endpoints.kubernetes.io/over-capacity: truncated`](https://kubernetes.io/docs/reference/labels-annotations-taints/#endpoints-kubernetes-io-over-capacity).
Control plane cũng gỡ annotation đó nếu số Pod backend giảm xuống dưới 1000.

Traffic vẫn được gửi tới các backend, nhưng bất kỳ cơ chế cân bằng tải nào dựa trên
API Endpoints cũ chỉ gửi traffic tới tối đa 1000 endpoint backend khả dụng.

Cùng giới hạn API đó nghĩa là bạn không thể cập nhật thủ công một Endpoints để có hơn 1000 endpoint.

### Giao thức ứng dụng (Application protocol)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.20 [stable]`

Trường `appProtocol` cung cấp một cách để chỉ định giao thức ứng dụng cho
mỗi port của Service. Trường này được dùng như một gợi ý để các triển khai (implementation)
cung cấp hành vi phong phú hơn cho những giao thức mà chúng hiểu.
Giá trị của trường này được sao chép sang các đối tượng
Endpoints và EndpointSlice tương ứng.

Trường này tuân theo cú pháp nhãn tiêu chuẩn của Kubernetes. Các giá trị hợp lệ là một trong:

* [Tên service chuẩn IANA](https://www.iana.org/assignments/service-names).

* Tên có tiền tố do triển khai tự định nghĩa, chẳng hạn `mycompany.com/my-custom-protocol`.

* Tên có tiền tố do Kubernetes định nghĩa:

| Giao thức | Mô tả |
|----------|-------------|
| `kubernetes.io/h2c` | HTTP/2 qua kênh không mã hóa (cleartext) như mô tả trong [RFC 9113](https://www.rfc-editor.org/rfc/rfc9113) |
| `kubernetes.io/ws`  | WebSocket qua kênh không mã hóa như mô tả trong [RFC 6455](https://www.rfc-editor.org/rfc/rfc6455) |
| `kubernetes.io/wss` | WebSocket qua TLS như mô tả trong [RFC 6455](https://www.rfc-editor.org/rfc/rfc6455) |

### Service nhiều port (Multi-port Services)

Với một số Service, bạn cần expose nhiều hơn một port.
Kubernetes cho phép bạn cấu hình nhiều định nghĩa port trên một đối tượng Service.
Khi dùng nhiều port cho một Service, bạn phải đặt tên cho tất cả các port
để chúng không mơ hồ.
Ví dụ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 9376
    - name: https
      protocol: TCP
      port: 443
      targetPort: 9377
```

> **Ghi chú:**
> Giống như tên (name) trong Kubernetes nói chung, tên port
> chỉ được chứa các ký tự chữ và số viết thường cùng dấu `-`. Tên port cũng phải
> bắt đầu và kết thúc bằng một ký tự chữ hoặc số.
>
> Ví dụ, các tên `123-abc` và `web` là hợp lệ, nhưng `123_abc` và `-web` thì không.

## Loại Service (Service type) {#publishing-services-service-types}

Với một số phần của ứng dụng (ví dụ, các frontend), bạn có thể muốn expose một
Service ra một địa chỉ IP bên ngoài, tức là địa chỉ có thể truy cập từ bên ngoài
cluster của bạn.

Các loại Service của Kubernetes cho phép bạn chỉ định loại Service bạn muốn.

Các giá trị `type` khả dụng và hành vi của chúng là:

[`ClusterIP`](#type-clusterip)
: Expose Service trên một IP nội bộ của cluster. Chọn giá trị này
  khiến Service chỉ có thể truy cập được từ bên trong cluster. Đây là
  giá trị mặc định được dùng nếu bạn không chỉ định `type` cho một Service một cách tường minh.
  Bạn có thể expose Service ra internet công cộng bằng một
  [Ingress](./11-ingress-vi.md) hoặc một
  [Gateway](https://gateway-api.sigs.k8s.io/).

[`NodePort`](#type-nodeport)
: Expose Service trên IP của mỗi Node tại một port tĩnh (gọi là `NodePort`).
  Để node port khả dụng, Kubernetes thiết lập một địa chỉ cluster IP,
  giống như khi bạn yêu cầu một Service `type: ClusterIP`.

[`LoadBalancer`](#loadbalancer)
: Expose Service ra bên ngoài bằng một bộ cân bằng tải (load balancer) bên ngoài. Kubernetes
  không trực tiếp cung cấp thành phần cân bằng tải; bạn phải tự cung cấp, hoặc
  bạn có thể tích hợp cluster Kubernetes của mình với một nhà cung cấp cloud.

[`ExternalName`](#externalname)
: Ánh xạ Service tới nội dung của trường `externalName` (ví dụ,
  tới hostname `api.foo.bar.example`). Việc ánh xạ này cấu hình DNS server
  của cluster trả về một bản ghi `CNAME` với giá trị hostname bên ngoài đó.
  Không có bất kỳ hình thức proxy nào được thiết lập.

Trường `type` trong Service API được thiết kế như chức năng lồng nhau — mỗi cấp
bổ sung thêm cho cấp trước. Tuy nhiên có một ngoại lệ cho thiết kế lồng nhau này. Bạn có thể
định nghĩa một Service `LoadBalancer` bằng cách
[tắt việc cấp phát `NodePort` cho load balancer](https://kubernetes.io/docs/concepts/services-networking/service/#load-balancer-nodeport-allocation).

### `type: ClusterIP` {#type-clusterip}

Loại Service mặc định này gán một địa chỉ IP từ một vùng (pool) địa chỉ IP mà
cluster của bạn đã dành riêng cho mục đích đó.

Một số loại Service khác xây dựng trên nền tảng của loại `ClusterIP`.

Nếu bạn định nghĩa một Service với `.spec.clusterIP` được đặt là `"None"` thì
Kubernetes không gán địa chỉ IP. Xem [headless Service](#headless-services)
để biết thêm thông tin.

#### Tự chọn địa chỉ IP (Choosing your own IP address)

Bạn có thể chỉ định địa chỉ cluster IP của riêng mình như một phần của yêu cầu tạo
`Service`. Để làm điều đó, đặt trường `.spec.clusterIP`. Ví dụ, khi bạn
đã có sẵn một bản ghi DNS muốn tái sử dụng, hoặc các hệ thống cũ (legacy)
đã được cấu hình cho một địa chỉ IP cụ thể và khó cấu hình lại.

Địa chỉ IP mà bạn chọn phải là một địa chỉ IPv4 hoặc IPv6 hợp lệ nằm trong
dải CIDR `service-cluster-ip-range` đã được cấu hình cho API server.
Nếu bạn thử tạo một Service với giá trị `clusterIP` không hợp lệ, API
server sẽ trả về mã trạng thái HTTP 422 để báo hiệu có vấn đề.

Đọc [tránh xung đột](https://kubernetes.io/docs/reference/networking/virtual-ips/#avoiding-collisions)
để tìm hiểu cách Kubernetes giúp giảm rủi ro và tác động của việc hai Service khác nhau
cùng cố dùng một địa chỉ IP.

### `type: NodePort` {#type-nodeport}

Nếu bạn đặt trường `type` là `NodePort`, control plane của Kubernetes
cấp phát một port từ dải được chỉ định bởi cờ `--service-node-port-range` (mặc định: 30000-32767).
Mỗi node proxy port đó (cùng một số port trên mọi Node) vào Service của bạn.
Service của bạn báo cáo port đã được cấp phát trong trường `.spec.ports[*].nodePort`.

Dùng NodePort cho bạn quyền tự do thiết lập giải pháp cân bằng tải của riêng mình,
cấu hình các môi trường chưa được Kubernetes hỗ trợ đầy đủ, hoặc thậm chí
expose trực tiếp địa chỉ IP của một hoặc nhiều node.

Với một node port Service, Kubernetes cấp phát thêm một port (TCP, UDP hoặc
SCTP tùy theo giao thức của Service). Mọi node trong cluster tự cấu hình
để lắng nghe trên port đã được gán đó và chuyển tiếp traffic tới một trong các endpoint
sẵn sàng (ready) gắn với Service đó. Bạn sẽ có thể liên lạc với Service `type: NodePort`
từ bên ngoài cluster, bằng cách kết nối tới bất kỳ node nào dùng giao thức phù hợp
(ví dụ: TCP), và port phù hợp (đã được gán cho Service đó).

#### Tự chọn port (Choosing your own port) {#nodeport-custom-port}

Nếu bạn muốn một số port cụ thể, bạn có thể chỉ định giá trị trong trường
`nodePort`. Control plane hoặc sẽ cấp phát cho bạn port đó, hoặc báo rằng
giao dịch API đã thất bại.
Điều này có nghĩa là bạn cần tự lo liệu các xung đột port có thể xảy ra.
Bạn cũng phải dùng một số port hợp lệ, nằm trong dải đã được cấu hình
cho việc dùng NodePort.

Đây là một manifest ví dụ cho Service `type: NodePort` chỉ định
một giá trị NodePort (30007, trong ví dụ này):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - port: 80
      # Theo mặc định và để thuận tiện, `targetPort` được đặt
      # cùng giá trị với trường `port`.
      targetPort: 80
      # Trường tùy chọn
      # Theo mặc định và để thuận tiện, control plane của Kubernetes
      # sẽ cấp phát một port từ một dải (mặc định: 30000-32767)
      nodePort: 30007
```

#### Dành riêng dải NodePort để tránh xung đột (Reserve NodePort ranges to avoid collisions) {#avoid-nodeport-collisions}

Chính sách gán port cho các NodePort service áp dụng cho cả kịch bản gán tự động lẫn
kịch bản gán thủ công. Khi một người dùng muốn tạo một NodePort service
dùng một port cụ thể, port mục tiêu có thể xung đột với một port khác đã được gán trước đó.

Để tránh vấn đề này, dải port cho NodePort service được chia thành hai băng (band).
Việc gán port động dùng băng trên theo mặc định, và có thể dùng băng dưới khi
băng trên đã cạn. Nhờ đó người dùng có thể cấp phát từ băng dưới với rủi ro xung đột port thấp hơn.

Khi dùng dải NodePort mặc định 30000-32767, các băng được phân chia như sau:

- Băng tĩnh: 30000-30085
- Băng động: 30086-32767

Xem [Avoid Collisions Assigning Ports to NodePort Services](https://kubernetes.io/blog/2023/05/11/nodeport-dynamic-and-static-allocation/)
để biết thêm chi tiết về cách tính băng tĩnh và băng động.

#### Cấu hình địa chỉ IP tùy chỉnh cho Service `type: NodePort` (Custom IP address configuration for `type: NodePort` Services) {#service-nodeport-custom-listen-address}

Bạn có thể thiết lập các node trong cluster dùng một địa chỉ IP cụ thể để phục vụ các node port
service. Bạn có thể muốn làm vậy nếu mỗi node được kết nối tới nhiều mạng (ví dụ:
một mạng cho traffic ứng dụng, và một mạng khác cho traffic giữa các node và
control plane).

Nếu bạn muốn chỉ định (các) địa chỉ IP cụ thể để proxy port, bạn có thể đặt
cờ `--nodeport-addresses` cho kube-proxy hoặc trường tương đương `nodePortAddresses`
trong [file cấu hình kube-proxy](https://kubernetes.io/docs/reference/config-api/kube-proxy-config.v1alpha1/)
thành (các) khối IP cụ thể.

Cờ này nhận một danh sách các khối IP phân tách bằng dấu phẩy (ví dụ `10.0.0.0/8`, `192.0.2.0/25`)
để chỉ định các dải địa chỉ IP mà kube-proxy nên coi là cục bộ (local) đối với node này.

Ví dụ, nếu bạn khởi động kube-proxy với cờ `--nodeport-addresses=127.0.0.0/8`,
kube-proxy chỉ chọn giao diện loopback cho các NodePort Service.
Giá trị mặc định của `--nodeport-addresses` là danh sách rỗng.
Điều đó có nghĩa kube-proxy nên xét mọi giao diện mạng khả dụng cho NodePort.
(Cách này cũng tương thích với các bản phát hành Kubernetes trước đây.)

> **Ghi chú:**
> Service này có thể thấy được qua `<NodeIP>:spec.ports[*].nodePort` và `.spec.clusterIP:spec.ports[*].port`.
> Nếu cờ `--nodeport-addresses` của kube-proxy hoặc trường tương đương
> trong file cấu hình kube-proxy được đặt, `<NodeIP>` sẽ là (các) địa chỉ IP node
> đã được lọc.

### `type: LoadBalancer` {#loadbalancer}

Trên các nhà cung cấp cloud hỗ trợ bộ cân bằng tải bên ngoài, việc đặt trường `type`
là `LoadBalancer` sẽ cấp phát (provision) một load balancer cho Service của bạn.
Việc tạo load balancer thực tế diễn ra bất đồng bộ, và
thông tin về load balancer đã cấp phát được công bố trong trường
`.status.loadBalancer` của Service.
Ví dụ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  clusterIP: 10.0.171.239
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 192.0.2.127
```

Traffic từ load balancer bên ngoài được chuyển hướng tới các Pod backend. Nhà cung cấp
cloud quyết định cách cân bằng tải.

Để triển khai một Service `type: LoadBalancer`, Kubernetes thường bắt đầu
bằng việc thực hiện các thay đổi tương đương với khi bạn yêu cầu một Service
`type: NodePort`. Thành phần cloud-controller-manager sau đó cấu hình load balancer
bên ngoài để chuyển tiếp traffic tới node port đã được gán đó.

Bạn có thể cấu hình một Service cân bằng tải
[bỏ qua](#load-balancer-nodeport-allocation) việc gán node port, miễn là
triển khai của nhà cung cấp cloud hỗ trợ điều này.

Một số nhà cung cấp cloud cho phép bạn chỉ định `loadBalancerIP`. Trong các trường hợp đó, load balancer được tạo
với `loadBalancerIP` do người dùng chỉ định. Nếu trường `loadBalancerIP` không được chỉ định,
load balancer được thiết lập với một địa chỉ IP tạm thời. Nếu bạn chỉ định `loadBalancerIP`
nhưng nhà cung cấp cloud của bạn không hỗ trợ tính năng này, trường `loadbalancerIP` mà bạn
đặt sẽ bị bỏ qua.

> **Ghi chú:**
> Trường `.spec.loadBalancerIP` của Service đã bị đánh dấu lỗi thời (deprecated) từ Kubernetes v1.24.
>
> Trường này được đặc tả không đầy đủ và ý nghĩa của nó khác nhau giữa các triển khai.
> Nó cũng không thể hỗ trợ mạng dual-stack. Trường này có thể bị gỡ bỏ trong một phiên bản API tương lai.
>
> Nếu bạn đang tích hợp với một nhà cung cấp hỗ trợ chỉ định (các) địa chỉ IP của load balancer
> cho một Service qua một annotation (riêng của nhà cung cấp), bạn nên chuyển sang cách đó.
>
> Nếu bạn đang viết code tích hợp load balancer với Kubernetes, tránh dùng trường này.
> Bạn có thể tích hợp với [Gateway](https://gateway-api.sigs.k8s.io/) thay vì Service, hoặc bạn
> có thể định nghĩa các annotation (riêng của nhà cung cấp) của riêng mình trên Service để chỉ định chi tiết tương đương.

#### Ảnh hưởng của tình trạng sống của node đến traffic của load balancer (Node liveness impact on load balancer traffic)

Kiểm tra sức khỏe (health check) của load balancer là yếu tố then chốt với các ứng dụng hiện đại. Chúng được dùng để
xác định server nào (máy ảo, hay địa chỉ IP) mà load balancer nên
gửi traffic tới. Các API của Kubernetes không định nghĩa cách health check phải được
triển khai cho các load balancer do Kubernetes quản lý; thay vào đó, chính các nhà cung cấp cloud
(và những người viết code tích hợp) quyết định hành vi này. Health check của load
balancer được sử dụng rộng rãi trong ngữ cảnh hỗ trợ trường
`externalTrafficPolicy` của Service.

#### Load balancer với nhiều loại giao thức (Load balancers with mixed protocol types)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Theo mặc định, với các Service loại LoadBalancer, khi có nhiều hơn một port được định nghĩa, tất cả
các port phải có cùng giao thức, và giao thức đó phải được nhà cung cấp cloud
hỗ trợ.

Feature gate `MixedProtocolLBService` (được bật mặc định cho kube-apiserver từ v1.24) cho phép dùng
các giao thức khác nhau cho Service loại LoadBalancer, khi có nhiều hơn một port được định nghĩa.

> **Ghi chú:**
> Tập hợp các giao thức có thể dùng cho các Service cân bằng tải do nhà cung cấp
> cloud của bạn định nghĩa; họ có thể áp đặt các hạn chế ngoài những gì Kubernetes API bắt buộc.

#### Tắt cấp phát NodePort cho load balancer (Disabling load balancer NodePort allocation) {#load-balancer-nodeport-allocation}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Bạn có thể tùy chọn tắt việc cấp phát node port cho một Service `type: LoadBalancer`, bằng cách đặt
trường `spec.allocateLoadBalancerNodePorts` là `false`. Điều này chỉ nên dùng cho các triển khai load balancer
định tuyến traffic trực tiếp tới Pod thay vì dùng node port. Theo mặc định, `spec.allocateLoadBalancerNodePorts`
là `true` và các Service loại LoadBalancer sẽ tiếp tục cấp phát node port. Nếu `spec.allocateLoadBalancerNodePorts`
được đặt là `false` trên một Service hiện có đã được cấp phát node port, những node port đó sẽ **không** được thu hồi tự động.
Bạn phải gỡ bỏ tường minh mục `nodePorts` trong mọi port của Service để thu hồi các node port đó.

#### Chỉ định class cho triển khai load balancer (Specifying class of load balancer implementation) {#load-balancer-class}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.24 [stable]`

Với một Service có `type` là `LoadBalancer`, trường `.spec.loadBalancerClass`
cho phép bạn dùng một triển khai load balancer khác với mặc định của nhà cung cấp cloud.

Theo mặc định, `.spec.loadBalancerClass` không được đặt và một Service loại
`LoadBalancer` dùng triển khai load balancer mặc định của nhà cung cấp cloud nếu
cluster được cấu hình với một nhà cung cấp cloud qua cờ thành phần
`--cloud-provider`.

Nếu bạn chỉ định `.spec.loadBalancerClass`, hệ thống giả định rằng một triển khai
load balancer khớp với class được chỉ định đang theo dõi (watch) các Service.
Mọi triển khai load balancer mặc định (ví dụ, triển khai do
nhà cung cấp cloud cung cấp) sẽ bỏ qua các Service có đặt trường này.
`spec.loadBalancerClass` chỉ có thể được đặt trên Service loại `LoadBalancer`.
Một khi đã đặt, nó không thể thay đổi.
Giá trị của `spec.loadBalancerClass` phải là một định danh kiểu nhãn (label-style),
với tiền tố tùy chọn, chẳng hạn "`internal-vip`" hoặc "`example.com/internal-vip`".
Các tên không có tiền tố được dành riêng cho người dùng cuối.

#### Chế độ địa chỉ IP của load balancer (Load balancer IP address mode) {#load-balancer-ip-mode}

Với một Service `type: LoadBalancer`, một controller có thể đặt `.status.loadBalancer.ingress.ipMode`.
Trường `.status.loadBalancer.ingress.ipMode` chỉ định cách IP của load balancer hành xử.
Nó chỉ có thể được chỉ định khi trường `.status.loadBalancer.ingress.ip` cũng được chỉ định.

Có hai giá trị khả dĩ cho `.status.loadBalancer.ingress.ipMode`: "VIP" và "Proxy".
Giá trị mặc định là "VIP", nghĩa là traffic được chuyển tới node
với đích đến (destination) được đặt là IP và port của load balancer.
Có hai trường hợp khi đặt giá trị này là "Proxy", tùy vào cách load balancer
của nhà cung cấp cloud chuyển traffic:

- Nếu traffic được chuyển tới node rồi được DNAT tới Pod, đích đến sẽ được đặt là IP của node và node port;
- Nếu traffic được chuyển trực tiếp tới Pod, đích đến sẽ được đặt là IP và port của Pod.

Các triển khai Service có thể dùng thông tin này để điều chỉnh việc định tuyến traffic.

#### Load balancer nội bộ (Internal load balancer)

Trong một môi trường hỗn hợp, đôi khi cần định tuyến traffic từ các Service bên trong cùng
một khối địa chỉ mạng (ảo).

Trong môi trường DNS split-horizon, bạn sẽ cần hai Service để có thể định tuyến cả traffic
bên ngoài lẫn bên trong tới các endpoint của bạn.

Để thiết lập một load balancer nội bộ, thêm một trong các annotation sau vào Service
tùy theo nhà cung cấp dịch vụ cloud mà bạn đang dùng:

##### Default

Chọn một trong các tab.

##### GCP

```yaml
metadata:
  name: my-service
  annotations:
    networking.gke.io/load-balancer-type: "Internal"
```

##### AWS

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internal"
```

##### Azure

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

##### IBM Cloud

```yaml
metadata:
  name: my-service
  annotations:
    service.kubernetes.io/ibm-load-balancer-cloud-provider-ip-type: "private"
```

##### OpenStack

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/openstack-internal-load-balancer: "true"
```

##### Baidu Cloud

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/cce-load-balancer-internal-vpc: "true"
```

##### Tencent Cloud

```yaml
metadata:
  annotations:
    service.kubernetes.io/qcloud-loadbalancer-internal-subnetid: subnet-xxxxx
```

##### Alibaba Cloud

```yaml
metadata:
  annotations:
    service.beta.kubernetes.io/alibaba-cloud-loadbalancer-address-type: "intranet"
```

##### OCI

```yaml
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/oci-load-balancer-internal: true
```

### `type: ExternalName` {#externalname}

Service loại ExternalName ánh xạ một Service tới một tên DNS, chứ không phải tới một selector thông thường như
`my-service` hay `cassandra`. Bạn chỉ định các Service này bằng tham số `spec.externalName`.

Ví dụ, định nghĩa Service này ánh xạ
Service `my-service` trong namespace `prod` tới `my.database.example.com`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

> **Ghi chú:**
> Một Service `type: ExternalName` chấp nhận một chuỗi địa chỉ IPv4,
> nhưng coi chuỗi đó như một tên DNS gồm các chữ số,
> chứ không phải một địa chỉ IP (tuy nhiên internet không cho phép những tên như vậy trong DNS).
> Các Service có external name giống địa chỉ IPv4
> sẽ không được các DNS server phân giải.
>
> Nếu bạn muốn ánh xạ một Service trực tiếp tới một địa chỉ IP cụ thể, hãy cân nhắc dùng
> [headless Service](#headless-services).

Khi tra cứu host `my-service.prod.svc.cluster.local`, DNS Service của cluster
trả về một bản ghi `CNAME` với giá trị `my.database.example.com`. Việc truy cập
`my-service` hoạt động giống các Service khác nhưng với khác biệt then chốt
là việc chuyển hướng diễn ra ở tầng DNS thay vì qua proxy hay
chuyển tiếp (forwarding). Nếu sau này bạn quyết định chuyển cơ sở dữ liệu vào trong cluster, bạn
có thể khởi động các Pod của nó, thêm các selector hoặc endpoint phù hợp, và đổi
`type` của Service.

> **Thận trọng:**
> Bạn có thể gặp rắc rối khi dùng ExternalName với một số giao thức phổ biến, bao gồm HTTP và HTTPS.
> Nếu bạn dùng ExternalName thì hostname mà các client bên trong cluster sử dụng khác với
> tên mà ExternalName tham chiếu tới.
>
> Với các giao thức dùng hostname, sự khác biệt này có thể dẫn đến lỗi hoặc phản hồi không mong đợi.
> Các HTTP request sẽ có header `Host:` mà server gốc không nhận ra;
> các TLS server sẽ không thể cung cấp certificate khớp với hostname mà client đã kết nối tới.

## Headless Service (Headless Services)

Đôi khi bạn không cần cân bằng tải và một Service IP duy nhất. Trong
trường hợp này, bạn có thể tạo cái gọi là _headless Service_, bằng cách chỉ định
tường minh `"None"` cho địa chỉ cluster IP (`.spec.clusterIP`).

Bạn có thể dùng headless Service để giao tiếp với các cơ chế khám phá service khác,
mà không bị ràng buộc vào cách triển khai của Kubernetes.

Với headless Service, cluster IP không được cấp phát, kube-proxy không xử lý
các Service này, và nền tảng không thực hiện cân bằng tải hay proxy cho chúng.

Headless Service cho phép client kết nối trực tiếp tới bất kỳ Pod nào nó muốn. Các Service headless không
cấu hình định tuyến và chuyển tiếp gói tin bằng
[địa chỉ IP ảo và proxy](https://kubernetes.io/docs/reference/networking/virtual-ips/); thay vào đó, headless Service báo cáo
địa chỉ IP endpoint của từng Pod qua các bản ghi DNS nội bộ, được phục vụ thông qua
[DNS service](./10-dns-pod-service-vi.md) của cluster.
Để định nghĩa một headless Service, bạn tạo một Service với `.spec.type` là ClusterIP (cũng là giá trị mặc định của `type`),
và đồng thời đặt `.spec.clusterIP` là None.

Giá trị chuỗi None là một trường hợp đặc biệt và không giống với việc bỏ trống trường `.spec.clusterIP`.

Cách DNS được cấu hình tự động phụ thuộc vào việc Service có định nghĩa selector hay không:

### Có selector (With selectors)

Với các headless Service có định nghĩa selector, endpoints controller tạo
các EndpointSlice trong Kubernetes API, và sửa đổi cấu hình DNS để trả về
các bản ghi A hoặc AAAA (địa chỉ IPv4 hoặc IPv6) trỏ trực tiếp tới các Pod phía sau Service.

### Không có selector (Without selectors)

Với các headless Service không định nghĩa selector, control plane không
tạo các đối tượng EndpointSlice. Tuy nhiên, hệ thống DNS sẽ tìm và cấu hình
một trong hai:

* Bản ghi DNS CNAME cho Service [`type: ExternalName`](#externalname).
* Bản ghi DNS A / AAAA cho tất cả địa chỉ IP của các endpoint sẵn sàng của Service,
  với mọi loại Service khác `ExternalName`.
  * Với endpoint IPv4, hệ thống DNS tạo bản ghi A.
  * Với endpoint IPv6, hệ thống DNS tạo bản ghi AAAA.

Khi bạn định nghĩa một headless Service không có selector, `port` phải
khớp với `targetPort`.

## Khám phá service (Discovering services)

Với các client chạy bên trong cluster, Kubernetes hỗ trợ hai phương thức chính để
tìm một Service: biến môi trường và DNS.

### Biến môi trường (Environment variables)

Khi một Pod chạy trên một Node, kubelet thêm một tập hợp biến môi trường
cho mỗi Service đang hoạt động. Nó thêm các biến `{SVCNAME}_SERVICE_HOST` và `{SVCNAME}_SERVICE_PORT`,
trong đó tên Service được viết hoa và dấu gạch ngang được chuyển thành gạch dưới.

Ví dụ, Service `redis-primary` expose TCP port 6379 và đã được
cấp phát địa chỉ cluster IP 10.0.0.11, sinh ra các biến môi trường
sau:

```shell
REDIS_PRIMARY_SERVICE_HOST=10.0.0.11
REDIS_PRIMARY_SERVICE_PORT=6379
REDIS_PRIMARY_PORT=tcp://10.0.0.11:6379
REDIS_PRIMARY_PORT_6379_TCP=tcp://10.0.0.11:6379
REDIS_PRIMARY_PORT_6379_TCP_PROTO=tcp
REDIS_PRIMARY_PORT_6379_TCP_PORT=6379
REDIS_PRIMARY_PORT_6379_TCP_ADDR=10.0.0.11
```

> **Ghi chú:**
> Khi bạn có một Pod cần truy cập một Service, và bạn dùng
> phương thức biến môi trường để công bố port và cluster IP cho các Pod
> client, bạn phải tạo Service *trước khi* các Pod client ra đời.
> Nếu không, các Pod client đó sẽ không được nạp các biến môi trường.
>
> Nếu bạn chỉ dùng DNS để khám phá cluster IP của một Service, bạn không cần
> lo lắng về vấn đề thứ tự này.

Kubernetes cũng hỗ trợ và cung cấp các biến tương thích với tính năng
"_[legacy container links](https://docs.docker.com/network/links/)_" của Docker
Engine. Bạn có thể đọc [`makeLinkVariables`](https://github.com/kubernetes/kubernetes/blob/dd2d12f6dc0e654c15d5db57a5f9f6ba61192726/pkg/kubelet/envvars/envvars.go#L72)
để xem cách điều này được triển khai trong Kubernetes.

### DNS

Bạn có thể (và gần như luôn luôn nên) thiết lập một dịch vụ DNS cho cluster
Kubernetes của mình bằng một [add-on](https://kubernetes.io/docs/concepts/cluster-administration/addons/).

Một DNS server hiểu cluster (cluster-aware), chẳng hạn CoreDNS, theo dõi Kubernetes API để phát hiện các
Service mới và tạo một tập hợp bản ghi DNS cho mỗi Service. Nếu DNS đã được bật
trong toàn cluster thì mọi Pod sẽ có thể tự động phân giải
các Service theo tên DNS của chúng.

Ví dụ, nếu bạn có một Service tên `my-service` trong namespace
`my-ns` của Kubernetes, control plane và DNS Service phối hợp cùng nhau
tạo một bản ghi DNS cho `my-service.my-ns`. Các Pod trong namespace `my-ns`
sẽ có thể tìm thấy service bằng cách tra cứu tên `my-service`
(`my-service.my-ns` cũng hoạt động).

Các Pod trong những namespace khác phải dùng tên đầy đủ `my-service.my-ns`. Những tên này
sẽ phân giải thành cluster IP đã được gán cho Service.

Kubernetes cũng hỗ trợ bản ghi DNS SRV (Service) cho các port có tên. Nếu
Service `my-service.my-ns` có một port tên `http` với giao thức là
`TCP`, bạn có thể thực hiện truy vấn DNS SRV cho `_http._tcp.my-service.my-ns` để tìm ra
số port của `http`, cũng như địa chỉ IP.

DNS server của Kubernetes là cách duy nhất để truy cập các Service `ExternalName`.
Bạn có thể tìm thêm thông tin về việc phân giải `ExternalName` tại
[DNS cho Service và Pod](./10-dns-pod-service-vi.md).

<!-- giữ nguyên các hyperlink hiện có -->
<a id="shortcomings" />
<a id="the-gory-details-of-virtual-ips" />
<a id="proxy-modes" />
<a id="proxy-mode-userspace" />
<a id="proxy-mode-iptables" />
<a id="proxy-mode-ipvs" />
<a id="ips-and-vips" />

## Cơ chế đánh địa chỉ IP ảo (Virtual IP addressing mechanism)

Đọc [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/) — trang này giải thích
cơ chế mà Kubernetes cung cấp để expose một Service với một địa chỉ IP ảo.

### Chính sách traffic (Traffic policies)

Bạn có thể đặt các trường `.spec.internalTrafficPolicy` và `.spec.externalTrafficPolicy`
để điều khiển cách Kubernetes định tuyến traffic tới các backend khỏe mạnh ("ready").

Xem [Traffic Policies](https://kubernetes.io/docs/reference/networking/virtual-ips/#traffic-policies) để biết thêm chi tiết.

### Điều khiển phân phối traffic (Traffic distribution control) {#traffic-distribution}

Trường `.spec.trafficDistribution` cung cấp một cách khác để tác động đến việc định tuyến
traffic bên trong một Service của Kubernetes. Trong khi các chính sách traffic tập trung vào
các đảm bảo ngữ nghĩa chặt chẽ, phân phối traffic cho phép bạn diễn đạt các _ưu tiên_
(chẳng hạn định tuyến tới các endpoint gần hơn về mặt topology). Điều này có thể giúp tối ưu
hiệu năng, chi phí, hoặc độ tin cậy. Trong Kubernetes v1.36, các
giá trị sau được hỗ trợ:

`PreferSameZone`
: Biểu thị ưu tiên định tuyến traffic tới các endpoint nằm cùng
  zone với client.

`PreferSameNode`
: Biểu thị ưu tiên định tuyến traffic tới các endpoint nằm trên cùng
  node với client.

`PreferClose` (deprecated)
: Đây là bí danh cũ hơn của `PreferSameZone`, kém rõ ràng hơn về
  mặt ngữ nghĩa.

Nếu trường này không được đặt, triển khai sẽ áp dụng chiến lược định tuyến mặc định của nó.

Xem [Traffic
Distribution](https://kubernetes.io/docs/reference/networking/virtual-ips/#traffic-distribution) để biết
thêm chi tiết.

### Độ bám phiên (Session stickiness)

Nếu bạn muốn đảm bảo các kết nối từ một client cụ thể luôn được chuyển tới
cùng một Pod mỗi lần, bạn có thể cấu hình session affinity dựa trên
địa chỉ IP của client. Đọc [session affinity](https://kubernetes.io/docs/reference/networking/virtual-ips/#session-affinity)
để tìm hiểu thêm.

## IP bên ngoài (External IPs)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [deprecated]`

Mọi người dùng nên bắt đầu chuyển đổi khỏi `externalIPs`.
Hãy cân nhắc dùng một controller load balancer bên ngoài hoặc một triển khai
Gateway API thay thế.

Nếu có các IP bên ngoài định tuyến tới một hoặc nhiều node của cluster, các Service Kubernetes
có thể được expose trên các `externalIPs` đó. Khi traffic mạng đi vào cluster, với
IP bên ngoài (là IP đích) và port khớp với Service đó, các quy tắc và tuyến (route)
mà Kubernetes đã cấu hình sẽ đảm bảo traffic được định tuyến tới một trong các endpoint
của Service đó.

Khi bạn định nghĩa một Service, bạn có thể chỉ định `externalIPs` cho bất kỳ
[loại service](#publishing-services-service-types) nào.
Trong ví dụ dưới đây, Service tên `"my-service"` có thể được các client truy cập bằng TCP,
trên `"198.51.100.32:80"` (tính từ `.spec.externalIPs[]` và `.spec.ports[].port`).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 49152
  externalIPs:
    - 198.51.100.32
```

> **Ghi chú:**
> Kubernetes không quản lý việc cấp phát `externalIPs`; đây là trách nhiệm
> của quản trị viên cluster.

## Đối tượng API (API Object)

Service là một tài nguyên cấp cao nhất (top-level) trong Kubernetes REST API. Bạn có thể tìm thêm chi tiết
về [đối tượng Service API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#service-v1-core).

## Tiếp theo (What's next)

Tìm hiểu thêm về Service và cách chúng khớp vào Kubernetes:

* Làm theo hướng dẫn [Connecting Applications with Services](https://kubernetes.io/docs/tutorials/services/connect-applications-service/).
* Đọc về [Ingress](./11-ingress-vi.md), thứ
  expose các tuyến HTTP và HTTPS từ bên ngoài cluster tới các Service bên trong
  cluster của bạn.
* Đọc về [Gateway](./13-gateway-vi.md), một phần mở rộng cho
  Kubernetes mang lại nhiều linh hoạt hơn Ingress.

Để có thêm bối cảnh, đọc các tài liệu sau:

* [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
* [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
* [Service API reference](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/)
* [EndpointSlice API reference](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/endpoint-slice-v1/)
* [Endpoint API reference (legacy)](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/endpoints-v1/)
