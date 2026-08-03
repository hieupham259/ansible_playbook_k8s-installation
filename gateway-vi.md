# Gateway API

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/gateway/>
>
> Gateway API là một họ các loại API (API kinds) cung cấp khả năng cấp phát hạ tầng động (dynamic infrastructure provisioning)
> và định tuyến lưu lượng nâng cao.

Làm cho các dịch vụ mạng khả dụng bằng một cơ chế cấu hình có khả năng mở rộng, hướng theo vai trò (role-oriented) và hiểu giao thức (protocol-aware). [Gateway API](https://gateway-api.sigs.k8s.io/) là một [add-on](https://kubernetes.io/docs/concepts/cluster-administration/addons/)
chứa các [loại](https://gateway-api.sigs.k8s.io/references/spec/) API cung cấp khả năng cấp phát hạ tầng
động và định tuyến lưu lượng nâng cao.

## Các nguyên tắc thiết kế (Design principles)

Các nguyên tắc sau đây đã định hình thiết kế và kiến trúc của Gateway API:

* __Hướng theo vai trò (Role-oriented):__ Các loại API của Gateway API được mô hình hóa theo các vai trò trong tổ chức
  chịu trách nhiệm quản lý mạng dịch vụ (service networking) của Kubernetes:
  * __Nhà cung cấp hạ tầng (Infrastructure Provider):__ Quản lý hạ tầng cho phép nhiều cluster tách biệt
    phục vụ nhiều bên thuê (tenant), ví dụ: một nhà cung cấp cloud.
  * __Người vận hành cluster (Cluster Operator):__ Quản lý các cluster và thường quan tâm đến các chính sách, quyền truy cập
    mạng, quyền hạn của ứng dụng, v.v.
  * __Nhà phát triển ứng dụng (Application Developer):__ Quản lý một ứng dụng chạy trong cluster và thường
    quan tâm đến cấu hình ở cấp ứng dụng và việc tổ hợp các [Service](https://kubernetes.io/docs/concepts/services-networking/service/).
* __Khả chuyển (Portable):__ Các đặc tả của Gateway API được định nghĩa dưới dạng [custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources)
  và được hỗ trợ bởi nhiều [bản hiện thực](https://gateway-api.sigs.k8s.io/implementations/) (implementation).
* __Giàu khả năng biểu đạt (Expressive):__ Các loại API của Gateway API hỗ trợ chức năng cho các trường hợp định tuyến lưu lượng phổ biến
  như so khớp dựa trên header, phân bổ lưu lượng theo trọng số (traffic weighting), và những tính năng khác mà trước đây chỉ có thể làm được trong
  [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) bằng cách dùng các annotation tùy chỉnh.
* __Khả năng mở rộng (Extensible):__ Gateway cho phép liên kết các custom resource ở nhiều tầng khác nhau của API.
  Điều này giúp việc tùy chỉnh chi tiết trở nên khả thi ở những vị trí phù hợp trong cấu trúc API.

## Mô hình tài nguyên (Resource model)

Gateway API có bốn loại API ổn định (stable):

* __GatewayClass:__ Định nghĩa một tập các gateway có cấu hình chung và được quản lý bởi một controller
  hiện thực class đó.

* __Gateway:__ Định nghĩa một thực thể (instance) của hạ tầng xử lý lưu lượng, chẳng hạn một cloud load balancer.

* __HTTPRoute:__ Định nghĩa các quy tắc dành riêng cho HTTP để ánh xạ lưu lượng từ một listener của Gateway đến
  một đại diện của các endpoint mạng backend. Các endpoint này thường được biểu diễn dưới dạng một
  [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

* __GRPCRoute:__ Định nghĩa các quy tắc dành riêng cho gRPC để ánh xạ lưu lượng từ một listener của Gateway đến
  một đại diện của các endpoint mạng backend. Các endpoint này thường được biểu diễn dưới dạng một
  [Service](https://kubernetes.io/docs/concepts/services-networking/service/).

Gateway API được tổ chức thành các loại API khác nhau có mối quan hệ phụ thuộc lẫn nhau nhằm hỗ trợ
bản chất hướng theo vai trò của các tổ chức. Một đối tượng Gateway được liên kết với đúng một GatewayClass;
GatewayClass mô tả gateway controller chịu trách nhiệm quản lý các Gateway thuộc class đó.
Sau đó, một hoặc nhiều loại route, chẳng hạn HTTPRoute, được liên kết với các Gateway. Một Gateway có thể lọc các route
được phép gắn vào các `listeners` của nó, hình thành một mô hình tin cậy hai chiều (bidirectional trust model) với các route.

Hình sau minh họa mối quan hệ giữa ba loại API ổn định của Gateway API:

![Hình minh họa mối quan hệ giữa ba loại API ổn định của Gateway API](https://kubernetes.io/docs/images/gateway-kind-relationships.svg)

### GatewayClass {#api-kind-gateway-class}

Các Gateway có thể được hiện thực bởi những controller khác nhau, thường với cấu hình khác nhau. Một Gateway
phải tham chiếu đến một GatewayClass chứa tên của controller hiện thực
class đó.

Một ví dụ GatewayClass tối giản:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: example-class
spec:
  controllerName: example.com/gateway-controller
```

Trong ví dụ này, một controller đã hiện thực Gateway API được cấu hình để quản lý các GatewayClass
có tên controller là `example.com/gateway-controller`. Các Gateway thuộc class này sẽ được quản lý bởi
controller của bản hiện thực đó.

Xem tài liệu tham chiếu [GatewayClass](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1.GatewayClass)
để có định nghĩa đầy đủ về loại API này.

### Gateway {#api-kind-gateway}

Một Gateway mô tả một thực thể của hạ tầng xử lý lưu lượng. Nó định nghĩa một endpoint mạng
có thể được dùng để xử lý lưu lượng, tức là lọc (filtering), cân bằng (balancing), phân chia (splitting), v.v. cho các backend
chẳng hạn như một Service. Ví dụ, một Gateway có thể đại diện cho một cloud load balancer hoặc một proxy
server bên trong cluster (in-cluster) được cấu hình để chấp nhận lưu lượng HTTP.

Một ví dụ tài nguyên Gateway điển hình:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: example-gateway
  namespace: example-namespace
spec:
  gatewayClassName: example-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "www.example.com"
    allowedRoutes:
      namespaces:
        from: Same
```

Trong ví dụ này, một thực thể của hạ tầng xử lý lưu lượng được lập trình để lắng nghe lưu lượng HTTP
trên port 80. Vì trường `addresses` không được chỉ định, một địa chỉ hoặc hostname sẽ được
controller của bản hiện thực gán cho Gateway. Địa chỉ này được dùng làm endpoint mạng để
xử lý lưu lượng của các endpoint mạng backend được định nghĩa trong các route.

Xem tài liệu tham chiếu [Gateway](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1.Gateway)
để có định nghĩa đầy đủ về loại API này.

> **Ghi chú:**
>
> Theo mặc định, một Gateway chỉ chấp nhận các Route từ cùng namespace. Các Route liên namespace (cross-namespace) yêu cầu cấu hình `allowedRoutes`.

### HTTPRoute {#api-kind-httproute}

Loại HTTPRoute chỉ định hành vi định tuyến của các request HTTP từ một listener của Gateway đến các endpoint
mạng backend. Đối với backend là Service, một bản hiện thực có thể biểu diễn endpoint mạng backend dưới dạng IP
của Service hoặc các EndpointSlice đứng sau Service đó. Một HTTPRoute đại diện cho phần cấu hình được áp dụng lên
bản hiện thực Gateway bên dưới. Ví dụ, việc định nghĩa một HTTPRoute mới có thể dẫn đến việc cấu hình thêm
các tuyến lưu lượng trong một cloud load balancer hoặc proxy server bên trong cluster.

Một ví dụ HTTPRoute điển hình:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-httproute
spec:
  parentRefs:
  - name: example-gateway
  hostnames:
  - "www.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /login
    backendRefs:
    - name: example-svc
      port: 8080
```

Trong ví dụ này, lưu lượng HTTP từ Gateway `example-gateway` với header Host: được đặt là `www.example.com`
và path của request là `/login` sẽ được định tuyến đến Service `example-svc` trên port `8080`.

Xem tài liệu tham chiếu [HTTPRoute](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1.HTTPRoute)
để có định nghĩa đầy đủ về loại API này.

### GRPCRoute {#api-kind-grpcroute}

Loại GRPCRoute chỉ định hành vi định tuyến của các request gRPC từ một listener của Gateway đến các endpoint
mạng backend. Đối với backend là Service, một bản hiện thực có thể biểu diễn endpoint mạng backend dưới dạng IP
của Service hoặc các EndpointSlice đứng sau Service đó. Một GRPCRoute đại diện cho phần cấu hình được áp dụng lên
bản hiện thực Gateway bên dưới. Ví dụ, việc định nghĩa một GRPCRoute mới có thể dẫn đến việc cấu hình thêm
các tuyến lưu lượng trong một cloud load balancer hoặc proxy server bên trong cluster.

Các Gateway hỗ trợ GRPCRoute bắt buộc phải hỗ trợ HTTP/2 mà không cần bước nâng cấp (upgrade) ban đầu từ HTTP/1,
để đảm bảo lưu lượng gRPC luôn được truyền đúng cách.

Một ví dụ GRPCRoute điển hình:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: example-grpcroute
spec:
  parentRefs:
  - name: example-gateway
  hostnames:
  - "svc.example.com"
  rules:
  - backendRefs:
    - name: example-svc
      port: 50051
```

Trong ví dụ này, lưu lượng gRPC từ Gateway `example-gateway` với host được đặt là `svc.example.com`
sẽ được chuyển hướng đến service `example-svc` trên port `50051` trong cùng namespace.

GRPCRoute cho phép so khớp các dịch vụ gRPC cụ thể, theo ví dụ sau:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: example-grpcroute
spec:
  parentRefs:
  - name: example-gateway
  hostnames:
  - "svc.example.com"
  rules:
  - matches:
    - method:
        service: com.example
        method: Login
    backendRefs:
    - name: foo-svc
      port: 50051
```

Trong trường hợp này, GRPCRoute sẽ khớp mọi lưu lượng đến svc.example.com và áp dụng các quy tắc định tuyến của nó
để chuyển tiếp lưu lượng đến backend đúng. Vì chỉ có một điều kiện khớp (match) được chỉ định, chỉ những request
gọi đến method com.example.User.Login của svc.example.com mới được chuyển tiếp.
RPC của bất kỳ method nào khác sẽ không được Route này khớp.

Xem tài liệu tham chiếu [GRPCRoute](https://gateway-api.sigs.k8s.io/references/spec/#grpcroute)
để có định nghĩa đầy đủ về loại API này.

## Luồng request (Request flow)

Dưới đây là một ví dụ đơn giản về việc lưu lượng HTTP được định tuyến đến một Service bằng cách dùng một Gateway và một HTTPRoute:

![Sơ đồ minh họa ví dụ lưu lượng HTTP được định tuyến đến một Service bằng một Gateway và một HTTPRoute](https://kubernetes.io/docs/images/gateway-request-flow.svg)

Trong ví dụ này, luồng request đối với một Gateway được hiện thực dưới dạng reverse proxy như sau:

1. Client bắt đầu chuẩn bị một request HTTP cho URL `http://www.example.com`
2. Trình phân giải DNS (DNS resolver) của client truy vấn tên đích và biết được ánh xạ đến
   một hoặc nhiều địa chỉ IP gắn với Gateway.
3. Client gửi request đến địa chỉ IP của Gateway; reverse proxy nhận request HTTP
   và dùng header Host: để khớp với một cấu hình được suy ra từ Gateway
   và HTTPRoute gắn kèm.
4. (Tùy chọn) Reverse proxy có thể thực hiện so khớp header và/hoặc path của request dựa
   trên các quy tắc match của HTTPRoute.
5. (Tùy chọn) Reverse proxy có thể chỉnh sửa request; ví dụ, thêm hoặc bớt header,
   dựa trên các quy tắc filter của HTTPRoute.
6. Cuối cùng, reverse proxy chuyển tiếp request đến một hoặc nhiều backend.

## Tính tuân thủ (Conformance)

Gateway API bao phủ một tập hợp tính năng rộng và được hiện thực rộng rãi. Sự kết hợp này đòi hỏi
các định nghĩa và bài kiểm tra tuân thủ (conformance) rõ ràng để đảm bảo API mang lại trải nghiệm nhất quán
ở bất cứ nơi nào nó được sử dụng.

Xem tài liệu về [conformance](https://gateway-api.sigs.k8s.io/concepts/conformance/) để
hiểu các chi tiết như kênh phát hành (release channel), mức độ hỗ trợ (support level), và cách chạy các bài kiểm tra tuân thủ.

## Di chuyển từ Ingress (Migrating from Ingress)

Gateway API là hậu duệ (successor) của [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) API.
Tuy nhiên, nó không bao gồm loại Ingress. Do đó, cần một lần chuyển đổi duy nhất từ các tài nguyên
Ingress hiện có của bạn sang các tài nguyên Gateway API.

Tham khảo hướng dẫn [di chuyển từ ingress](https://gateway-api.sigs.k8s.io/guides/getting-started/migrating-from-ingress)
để biết chi tiết về việc di chuyển các tài nguyên Ingress sang tài nguyên Gateway API.

## Tiếp theo (What's next)

Thay vì được Kubernetes hiện thực một cách nguyên bản (natively), các đặc tả của tài nguyên Gateway API
được định nghĩa dưới dạng [Custom Resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
và được hỗ trợ bởi rất nhiều [bản hiện thực](https://gateway-api.sigs.k8s.io/implementations/).
[Cài đặt](https://gateway-api.sigs.k8s.io/guides/#installing-gateway-api) các CRD của Gateway API hoặc
làm theo hướng dẫn cài đặt của bản hiện thực mà bạn chọn. Sau khi cài đặt một
bản hiện thực, hãy dùng hướng dẫn [Getting Started](https://gateway-api.sigs.k8s.io/guides/) để
nhanh chóng bắt đầu làm việc với Gateway API.

> **Ghi chú:**
>
> Hãy chắc chắn xem kỹ tài liệu của bản hiện thực mà bạn chọn để hiểu mọi lưu ý (caveat) của nó.

Tham khảo [đặc tả API](https://gateway-api.sigs.k8s.io/reference/api-spec/main/spec/) để biết thêm
chi tiết về tất cả các loại API của Gateway API.
