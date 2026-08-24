# Gateway API

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/gateway/>
>
> Gateway API là một họ các loại API (API kinds) cung cấp khả năng cấp phát hạ tầng động (dynamic infrastructure provisioning)
> và định tuyến lưu lượng nâng cao.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), bài 12/16 · Kiểm chứng
ở Lab 5b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này đọc để **biết định hướng**, không phải để triển khai ngay. Điểm dễ hiểu nhầm nhất:
Gateway API **không có sẵn** trong Kubernetes như Ingress — nó là add-on gồm các
CustomResourceDefinition, phải cài CRD cộng với một bản hiện thực thì các loại object trong bài
mới tồn tại. Lab 5b cài ingress controller chứ không cài Gateway, nên phần thực hành để dành.

**Phải hiểu ở lần đọc này:**

- Gateway API là **add-on**: các đặc tả của nó được định nghĩa dưới dạng **Custom Resource**,
  không được Kubernetes hiện thực nguyên bản. Không cài CRD và một bản hiện thực thì không có
  gì chạy.
- Bốn loại API ổn định và chuỗi liên kết giữa chúng: **GatewayClass** (khai controller nào quản
  class đó) ← **Gateway** (`gatewayClassName`, một thực thể hạ tầng xử lý lưu lượng với các
  `listeners`) ← **HTTPRoute** / **GRPCRoute** (`parentRefs` trỏ lên Gateway, `backendRefs` trỏ
  xuống Service).
- **Mô hình tin cậy hai chiều**: Route khai Gateway cha, còn Gateway lọc những Route được phép
  gắn vào listener của nó qua `allowedRoutes`. Mặc định một Gateway **chỉ chấp nhận Route cùng
  namespace**.
- Vì sao API bị tách ra nhiều loại: nó **hướng theo vai trò** — infrastructure provider, cluster
  operator và application developer nắm những phần khác nhau của cấu hình mạng.
- Quan hệ với Ingress: Gateway API là **hậu duệ** của Ingress nhưng **không bao gồm loại
  Ingress**, nên chuyển đổi là một lần dứt điểm chứ không phải chạy song song trên cùng object.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Tính tuân thủ* — kênh phát hành, mức độ hỗ trợ, cách chạy bài kiểm tra | chỉ cần khi thẩm định một bản hiện thực cụ thể | Lab 5b |
| Ví dụ *GRPCRoute* so khớp `method` cụ thể | chỉ dùng khi chạy dịch vụ gRPC | không cần |
| Ba nguyên tắc thiết kế còn lại (khả chuyển, giàu khả năng biểu đạt, khả năng mở rộng) | là tuyên ngôn thiết kế, không đổi thao tác của bạn | không cần |

---

Làm cho các dịch vụ mạng khả dụng bằng một cơ chế cấu hình có khả năng mở rộng, hướng theo vai trò (role-oriented) và hiểu giao thức (protocol-aware). [Gateway API](https://gateway-api.sigs.k8s.io/) là một [add-on](165-addons-vi.md)
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
    quan tâm đến cấu hình ở cấp ứng dụng và việc tổ hợp các [Service](82-service-vi.md).
* __Khả chuyển (Portable):__ Các đặc tả của Gateway API được định nghĩa dưới dạng [custom resource](179-custom-resources-vi.md)
  và được hỗ trợ bởi nhiều [bản hiện thực](https://gateway-api.sigs.k8s.io/implementations/) (implementation).
* __Giàu khả năng biểu đạt (Expressive):__ Các loại API của Gateway API hỗ trợ chức năng cho các trường hợp định tuyến lưu lượng phổ biến
  như so khớp dựa trên header, phân bổ lưu lượng theo trọng số (traffic weighting), và những tính năng khác mà trước đây chỉ có thể làm được trong
  [Ingress](11-ingress-vi.md) bằng cách dùng các annotation tùy chỉnh.
* __Khả năng mở rộng (Extensible):__ Gateway cho phép liên kết các custom resource ở nhiều tầng khác nhau của API.
  Điều này giúp việc tùy chỉnh chi tiết trở nên khả thi ở những vị trí phù hợp trong cấu trúc API.

## Mô hình tài nguyên (Resource model)

Gateway API có bốn loại API ổn định (stable):

* __GatewayClass:__ Định nghĩa một tập các gateway có cấu hình chung và được quản lý bởi một controller
  hiện thực class đó.

* __Gateway:__ Định nghĩa một thực thể (instance) của hạ tầng xử lý lưu lượng, chẳng hạn một cloud load balancer.

* __HTTPRoute:__ Định nghĩa các quy tắc dành riêng cho HTTP để ánh xạ lưu lượng từ một listener của Gateway đến
  một đại diện của các endpoint mạng backend. Các endpoint này thường được biểu diễn dưới dạng một
  [Service](82-service-vi.md).

* __GRPCRoute:__ Định nghĩa các quy tắc dành riêng cho gRPC để ánh xạ lưu lượng từ một listener của Gateway đến
  một đại diện của các endpoint mạng backend. Các endpoint này thường được biểu diễn dưới dạng một
  [Service](82-service-vi.md).

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

Gateway API là hậu duệ (successor) của [Ingress](11-ingress-vi.md) API.
Tuy nhiên, nó không bao gồm loại Ingress. Do đó, cần một lần chuyển đổi duy nhất từ các tài nguyên
Ingress hiện có của bạn sang các tài nguyên Gateway API.

Tham khảo hướng dẫn [di chuyển từ ingress](https://gateway-api.sigs.k8s.io/guides/getting-started/migrating-from-ingress)
để biết chi tiết về việc di chuyển các tài nguyên Ingress sang tài nguyên Gateway API.

## Tiếp theo (What's next)

Thay vì được Kubernetes hiện thực một cách nguyên bản (natively), các đặc tả của tài nguyên Gateway API
được định nghĩa dưới dạng [Custom Resource](179-custom-resources-vi.md)
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

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Bạn `kubectl apply` một manifest HTTPRoute lên cluster lab đúng như nó đang có. Chuyện gì
   xảy ra, và khác gì với việc apply một Ingress lên cùng cluster đó?
2. Cần tối thiểu những loại object nào để một request HTTP từ ngoài tới được một Service, và
   trường nào nối chúng lại với nhau?
3. Một HTTPRoute nằm ở namespace `app`, còn Gateway nằm ở namespace `infra`. Route có gắn được
   vào Gateway không?
4. Vì sao Gateway API tách thành nhiều loại object thay vì gói tất cả vào một object như
   Ingress?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Lệnh thất bại vì cluster không biết kind đó.** Gateway API không được Kubernetes hiện thực
   nguyên bản: các đặc tả tài nguyên của nó **được định nghĩa dưới dạng Custom Resource**, nên
   phải cài CRD của Gateway API cùng một bản hiện thực trước đã. Đây đúng là chỗ khác biệt với
   Ingress: `kind: Ingress` **có sẵn** trong Kubernetes nên object được tạo bình thường — nó chỉ
   nằm im nếu thiếu controller. Một bên thiếu *kiểu dữ liệu*, một bên thiếu *người thực thi*.
2. **GatewayClass → Gateway → HTTPRoute → Service.** Gateway tham chiếu GatewayClass qua
   `gatewayClassName`; GatewayClass khai `controllerName` của bản hiện thực sẽ quản lý các
   Gateway thuộc class đó. HTTPRoute gắn lên Gateway qua **`parentRefs`** và trỏ tới backend qua
   **`backendRefs`** (tên Service + port).
3. **Không, theo mặc định.** Bài ghi rõ: theo mặc định một Gateway **chỉ chấp nhận các Route từ
   cùng namespace**; Route liên namespace yêu cầu cấu hình **`allowedRoutes`**. Đó là vế thứ hai
   của mô hình tin cậy hai chiều — Route chọn Gateway cha, nhưng Gateway vẫn có quyền lọc Route
   nào được gắn vào `listeners` của nó.
4. Vì API được thiết kế **hướng theo vai trò**: các loại API được mô hình hóa theo những vai trò
   trong tổ chức chịu trách nhiệm quản lý mạng dịch vụ — **infrastructure provider** (hạ tầng đa
   tenant), **cluster operator** (chính sách, quyền truy cập mạng) và **application developer**
   (cấu hình ở cấp ứng dụng, tổ hợp Service). Tách object cho phép mỗi vai trò sở hữu đúng phần
   cấu hình của mình thay vì cùng ghi vào một object.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
