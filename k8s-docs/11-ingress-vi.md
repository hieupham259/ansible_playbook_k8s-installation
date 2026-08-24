# Ingress

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/services-networking/ingress/>
>
> Làm cho dịch vụ mạng HTTP (hoặc HTTPS) của bạn khả dụng bằng một cơ chế cấu hình hiểu giao thức (protocol-aware), nắm được các khái niệm web như URI, hostname, path, v.v. Khái niệm Ingress cho phép bạn ánh xạ lưu lượng (traffic) đến các backend khác nhau dựa trên các quy tắc bạn định nghĩa thông qua Kubernetes API.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 5](00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), bài 10/16 · Kiểm chứng
ở Lab 5b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Hai điều cần biết trước khi đọc. Thứ nhất, **Ingress API đã bị đóng băng** và dự án Kubernetes
khuyến nghị dùng Gateway thay thế — nhưng bạn vẫn phải học nó, vì phần lớn cluster đang chạy
đều dùng Ingress. Thứ hai, **cluster lab chưa có ingress controller**, nên mọi Ingress bạn tạo
lúc này sẽ nằm im. Controller được cài ở Lab 5b, cùng lúc với việc đổi CNI.

**Phải hiểu ở lần đọc này:**

- Ingress mở **route HTTP và HTTPS** từ ngoài vào Service trong cluster; nó **không** mở port
  hay giao thức tùy ý — thứ đó thuộc về `type: NodePort` và `type: LoadBalancer`. Và **phải có
  ingress controller đang chạy**: chỉ tạo tài nguyên Ingress thì không có tác dụng gì.
- Cấu trúc một quy tắc: `host` (tùy chọn) + danh sách `path`, mỗi path có backend là
  `service.name` cộng `service.port`. **Cả host lẫn path đều phải khớp** thì request mới được
  chuyển đi; request không khớp quy tắc nào rơi về `defaultBackend`.
- Ba `pathType`: `Exact`, `Prefix` và `ImplementationSpecific`. `Prefix` so khớp **theo từng
  phần tử path tách bởi `/`**, không phải theo tiền tố chuỗi — `/foo/bar` khớp `/foo/bar/baz`
  nhưng không khớp `/foo/barbaz`. Khai thiếu `pathType` thì manifest không qua được validation.
- Khi nhiều path cùng khớp: **path khớp dài nhất thắng**; hòa thì `Exact` thắng `Prefix`.
- IngressClass và `ingressClassName`: mỗi Ingress nên chỉ định class. Đánh dấu class mặc định
  bằng annotation `ingressclass.kubernetes.io/is-default-class: "true"`, và **nếu có nhiều hơn
  một class mặc định thì admission controller chặn việc tạo Ingress không ghi `ingressClassName`**.
- TLS: dùng Secret kiểu `kubernetes.io/tls` với hai khóa `tls.crt` và `tls.key`; chỉ hỗ trợ một
  port TLS là 443; **TLS kết thúc tại điểm ingress**, đoạn tới Service và Pod là plaintext; và
  `hosts` trong phần `tls` phải khớp tường minh `host` trong `rules`.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Resource backend* | trỏ tới một tài nguyên tùy chỉnh qua CRD | giai đoạn 14 |
| *Phạm vi của IngressClass* (`parameters.scope`, Cluster và Namespaced) | là cách ủy quyền cấu hình giữa các đội, cần RBAC | giai đoạn 9 |
| *Annotation đã bị loại bỏ* `kubernetes.io/ingress.class` | chỉ cần khi gặp manifest cũ | không cần |
| *Wildcard cho hostname* | chỉ có ý nghĩa khi đã có tên miền thật trỏ vào cluster | Lab 5b |
| *Cân bằng tải* và *Chịu lỗi giữa các availability zone* | phụ thuộc hoàn toàn vào từng controller và nhà cung cấp cloud | không cần |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.19 [stable]`

Ingress là một đối tượng API quản lý truy cập từ bên ngoài vào các service trong một cluster, thường là HTTP. Ingress có thể cung cấp cân bằng tải (load balancing), kết thúc phiên SSL (SSL termination) và virtual hosting dựa trên tên (name-based virtual hosting).

> **Ghi chú:**
>
> Dự án Kubernetes khuyến nghị sử dụng [Gateway](https://gateway-api.sigs.k8s.io/) thay cho
> [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).
> Ingress API đã bị đóng băng (frozen).
>
> Điều này có nghĩa là:
> * Ingress API đã ở trạng thái generally available (GA), và tuân theo các [đảm bảo về tính ổn định](https://kubernetes.io/docs/reference/using-api/deprecation-policy/#deprecating-parts-of-the-api) dành cho các API đã GA.
>   Dự án Kubernetes không có kế hoạch loại bỏ Ingress khỏi Kubernetes.
> * Ingress API sẽ không được phát triển thêm nữa, và sẽ không có bất kỳ thay đổi
>   hay cập nhật nào tiếp theo.

## Thuật ngữ (Terminology)

Để rõ ràng, tài liệu này định nghĩa các thuật ngữ sau:

* Node: Một máy worker trong Kubernetes, là một phần của cluster.
* Cluster: Một tập hợp các Node chạy các ứng dụng được đóng gói trong container, do Kubernetes quản lý.
  Trong ví dụ này, cũng như trong hầu hết các triển khai Kubernetes phổ biến, các node trong cluster
  không thuộc mạng internet công cộng.
* Edge router (router biên): Một router thực thi chính sách tường lửa cho cluster của bạn. Nó
  có thể là một gateway do nhà cung cấp cloud quản lý hoặc là một thiết bị phần cứng vật lý.
* Cluster network (mạng cluster): Một tập hợp các liên kết, logic hoặc vật lý, hỗ trợ việc giao tiếp
  bên trong một cluster theo [mô hình mạng](157-networking-vi.md) của Kubernetes.
* Service: Một [Service](82-service-vi.md) của Kubernetes định danh
  một tập hợp các Pod bằng các bộ chọn [label](18-labels-vi.md) (label selector).
  Trừ khi được nói rõ, các Service được giả định là chỉ có các IP ảo (virtual IP) chỉ định tuyến được bên trong mạng cluster.

## Ingress là gì?

[Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.33/#ingress-v1-networking-k8s-io)
mở các route HTTP và HTTPS từ bên ngoài cluster đến các
[service](82-service-vi.md) bên trong cluster.
Việc định tuyến lưu lượng được điều khiển bởi các quy tắc (rule) được định nghĩa trên tài nguyên Ingress.

Dưới đây là một ví dụ đơn giản trong đó một Ingress gửi toàn bộ lưu lượng của nó đến một Service duy nhất:

![ingress-diagram](https://kubernetes.io/docs/images/ingress.svg)

*Hình. Ingress* — [xem sơ đồ trên mermaid.live](https://mermaid.live/edit#pako:eNqNkstuwyAQRX8F4U0r2VHqPlSRKqt0UamLqlnaWWAYJygYLB59KMm_Fxcix-qmGwbuXA7DwAEzzQETXKutof0Ovb4vaoUQkwKUu6pi3FwXM_QSHGBt0VFFt8DRU2OWSGrKUUMlVQwMmhVLEV1Vcm9-aUksiuXRaO_CEhkv4WjBfAgG1TrGaLa-iaUw6a0DcwGI-WgOsF7zm-pN881fvRx1UDzeiFq7ghb1kgqFWiElyTjnuXVG74FkbdumefEpuNuRu_4rZ1pqQ7L5fL6YQPaPNiFuywcG9_-ihNyUkm6YSONWkjVNM8WUIyaeOJLO3clTB_KhL8NQDmVe-OJjxgZM5FhFiiFTK5zjDkxHBQ9_4zB4a-x20EGNSZhyaKmXrg7f5hSsvufUwTMXThtMWiot5Jh6p9ffimHijIezaSVoeN0uiqcfMJvf7w)

Một Ingress có thể được cấu hình để cấp cho các Service những URL truy cập được từ bên ngoài,
cân bằng tải lưu lượng, kết thúc phiên SSL / TLS, và cung cấp virtual hosting dựa trên tên.
Một [Ingress controller](12-ingress-controllers-vi.md)
chịu trách nhiệm hiện thực hóa Ingress, thường là bằng một bộ cân bằng tải (load balancer), tuy nhiên
nó cũng có thể cấu hình router biên của bạn hoặc các frontend bổ sung để giúp xử lý lưu lượng.

Một Ingress không mở các port hay giao thức tùy ý. Việc đưa các dịch vụ không phải HTTP và HTTPS ra internet thường
sử dụng service loại [Service.Type=NodePort](82-service-vi.md#type-nodeport) hoặc
[Service.Type=LoadBalancer](82-service-vi.md#loadbalancer).

## Điều kiện tiên quyết (Prerequisites)

Bạn phải có một [Ingress controller](12-ingress-controllers-vi.md)
để Ingress hoạt động. Việc chỉ tạo tài nguyên Ingress mà thôi sẽ không có tác dụng gì.

Bạn có thể chọn từ nhiều [Ingress controller](12-ingress-controllers-vi.md) khác nhau.

Lý tưởng nhất, mọi Ingress controller đều nên tuân theo đặc tả tham chiếu (reference specification). Trên thực tế, các Ingress
controller khác nhau hoạt động hơi khác nhau một chút.

> **Ghi chú:**
>
> Hãy chắc chắn rằng bạn đã đọc kỹ tài liệu của Ingress controller mà bạn chọn để hiểu những lưu ý (caveat) khi sử dụng nó.

## Tài nguyên Ingress (The Ingress resource)

Một ví dụ tài nguyên Ingress tối giản:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
spec:
  ingressClassName: nginx-example
  rules:
  - http:
      paths:
      - path: /testpath
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

Một Ingress cần các trường `apiVersion`, `kind`, `metadata` và `spec`.
Tên của một đối tượng Ingress phải là một
[tên miền con DNS hợp lệ](17-names-vi.md#dns-subdomain-names) (DNS subdomain name).
Để biết thông tin chung về cách làm việc với các file cấu hình, xem
[triển khai ứng dụng](345-run-stateless-application-vi.md),
[cấu hình container](275-configure-pod-configmap-vi.md),
[quản lý tài nguyên](61-management-vi.md).
Các Ingress controller thường sử dụng [annotation](20-annotations-vi.md) để cấu hình hành vi.
Hãy xem tài liệu của Ingress controller mà bạn chọn để biết những annotation nào được kỳ vọng và / hoặc được hỗ trợ.

[Ingress spec](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/ingress-v1/#IngressSpec)
chứa toàn bộ thông tin cần thiết để cấu hình một bộ cân bằng tải hoặc một proxy server. Quan trọng nhất, nó
chứa một danh sách các quy tắc được so khớp với mọi request đến. Tài nguyên Ingress chỉ hỗ trợ các quy tắc
để định hướng lưu lượng HTTP(S).

Nếu `ingressClassName` bị bỏ trống, một [Ingress class mặc định](#default-ingress-class)
cần được định nghĩa.

Một số Ingress controller vẫn hoạt động ngay cả khi không định nghĩa
IngressClass mặc định. Ngay cả khi bạn dùng một Ingress controller có thể
hoạt động mà không cần bất kỳ IngressClass nào, dự án Kubernetes vẫn khuyến nghị
bạn định nghĩa một IngressClass mặc định.

### Quy tắc Ingress (Ingress rules)

Mỗi quy tắc HTTP chứa các thông tin sau:

* Một host (tùy chọn). Trong ví dụ này, không có host nào được chỉ định, do đó quy tắc áp dụng cho mọi lưu lượng
  HTTP đi vào qua địa chỉ IP được chỉ định. Nếu một host được cung cấp (ví dụ,
  foo.bar.com), các quy tắc sẽ áp dụng cho host đó.
* Một danh sách các path (ví dụ, `/testpath`), mỗi path có một
  backend tương ứng được định nghĩa bằng `service.name` và `service.port.name` hoặc
  `service.port.number`. Cả host lẫn path đều phải khớp với nội dung của
  request đến trước khi bộ cân bằng tải chuyển hướng lưu lượng đến
  Service được tham chiếu.
* Một backend là sự kết hợp của tên Service và port như được mô tả trong
  [tài liệu về Service](82-service-vi.md) hoặc là một [custom resource backend](#resource-backend)
  thông qua một [CRD](179-custom-resources-vi.md) (CustomResourceDefinition). Các request HTTP (và HTTPS) đến
  Ingress mà khớp với host và path của quy tắc sẽ được gửi tới backend được liệt kê.

Một `defaultBackend` thường được cấu hình trong Ingress controller để phục vụ mọi request không
khớp với bất kỳ path nào trong spec.

### DefaultBackend {#default-backend}

Một Ingress không có quy tắc nào sẽ gửi toàn bộ lưu lượng đến một backend mặc định duy nhất và `.spec.defaultBackend`
chính là backend xử lý các request trong trường hợp đó.
`defaultBackend` theo quy ước là một tùy chọn cấu hình của
[Ingress controller](12-ingress-controllers-vi.md) và
không được khai báo trong các tài nguyên Ingress của bạn.
Nếu không có `.spec.rules` nào được chỉ định, thì `.spec.defaultBackend` bắt buộc phải được chỉ định.
Nếu `defaultBackend` không được thiết lập, việc xử lý các request không khớp với bất kỳ quy tắc nào sẽ do
ingress controller quyết định (hãy tham khảo tài liệu của ingress controller của bạn để biết nó xử lý trường hợp này như thế nào).

Nếu không có host hay path nào khớp với request HTTP trong các đối tượng Ingress, lưu lượng sẽ được
định tuyến đến backend mặc định của bạn.

### Resource backend {#resource-backend}

Một backend kiểu `Resource` là một ObjectRef trỏ đến một tài nguyên Kubernetes khác nằm trong
cùng namespace với đối tượng Ingress. `Resource` là thiết lập loại trừ lẫn nhau
(mutually exclusive) với Service, và sẽ không qua được bước kiểm tra hợp lệ (validation) nếu cả hai cùng được chỉ định. Một cách dùng
phổ biến của backend kiểu `Resource` là dẫn dữ liệu vào một backend lưu trữ đối tượng (object storage)
chứa các tài nguyên tĩnh (static assets).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-resource-backend
spec:
  defaultBackend:
    resource:
      apiGroup: k8s.example.com
      kind: StorageBucket
      name: static-assets
  rules:
    - http:
        paths:
          - path: /icons
            pathType: ImplementationSpecific
            backend:
              resource:
                apiGroup: k8s.example.com
                kind: StorageBucket
                name: icon-assets
```

Sau khi tạo Ingress ở trên, bạn có thể xem nó bằng lệnh sau:

```bash
kubectl describe ingress ingress-resource-backend
```

```
Name:             ingress-resource-backend
Namespace:        default
Address:
Default backend:  APIGroup: k8s.example.com, Kind: StorageBucket, Name: static-assets
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /icons   APIGroup: k8s.example.com, Kind: StorageBucket, Name: icon-assets
Annotations:  <none>
Events:       <none>
```

### Các loại path (Path types)

Mỗi path trong một Ingress bắt buộc phải có một loại path (path type) tương ứng. Những path
không khai báo `pathType` rõ ràng sẽ không qua được bước kiểm tra hợp lệ. Có ba
loại path được hỗ trợ:

* `ImplementationSpecific`: Với loại path này, việc so khớp tùy thuộc vào
  IngressClass. Các bản hiện thực (implementation) có thể coi đây là một `pathType` riêng biệt hoặc xử lý
  nó giống hệt như loại `Prefix` hoặc `Exact`.

* `Exact`: Khớp chính xác đường dẫn URL, có phân biệt hoa thường.

* `Prefix`: Khớp dựa trên tiền tố (prefix) của đường dẫn URL được tách bởi dấu `/`. Việc so khớp có phân
  biệt hoa thường và được thực hiện trên từng phần tử của path một. Một phần tử path là
  một nhãn trong danh sách các nhãn của path được tách bởi dấu phân cách `/`. Một request
  khớp với path _p_ nếu mọi _p_ là tiền tố theo-từng-phần-tử của _p_ trong
  path của request.

  > **Ghi chú:**
  >
  > Nếu phần tử cuối cùng của path là một chuỗi con (substring) của phần tử
  > cuối cùng trong path của request, thì đó không phải là một kết quả khớp (ví dụ: `/foo/bar`
  > khớp với `/foo/bar/baz`, nhưng không khớp với `/foo/barbaz`).

### Ví dụ (Examples)

| Loại   | Path                            | Path của request              | Khớp?                                     |
|--------|---------------------------------|-------------------------------|--------------------------------------------|
| Prefix | `/`                             | (mọi path)                    | Có                                         |
| Exact  | `/foo`                          | `/foo`                        | Có                                         |
| Exact  | `/foo`                          | `/bar`                        | Không                                      |
| Exact  | `/foo`                          | `/foo/`                       | Không                                      |
| Exact  | `/foo/`                         | `/foo`                        | Không                                      |
| Prefix | `/foo`                          | `/foo`, `/foo/`               | Có                                         |
| Prefix | `/foo/`                         | `/foo`, `/foo/`               | Có                                         |
| Prefix | `/aaa/bb`                       | `/aaa/bbb`                    | Không                                      |
| Prefix | `/aaa/bbb`                      | `/aaa/bbb`                    | Có                                         |
| Prefix | `/aaa/bbb/`                     | `/aaa/bbb`                    | Có, bỏ qua dấu gạch chéo cuối              |
| Prefix | `/aaa/bbb`                      | `/aaa/bbb/`                   | Có, khớp cả dấu gạch chéo cuối             |
| Prefix | `/aaa/bbb`                      | `/aaa/bbb/ccc`                | Có, khớp path con (subpath)                |
| Prefix | `/aaa/bbb`                      | `/aaa/bbbxyz`                 | Không, không khớp tiền tố chuỗi            |
| Prefix | `/`, `/aaa`                     | `/aaa/ccc`                    | Có, khớp tiền tố `/aaa`                    |
| Prefix | `/`, `/aaa`, `/aaa/bbb`         | `/aaa/bbb`                    | Có, khớp tiền tố `/aaa/bbb`                |
| Prefix | `/`, `/aaa`, `/aaa/bbb`         | `/ccc`                        | Có, khớp tiền tố `/`                       |
| Prefix | `/aaa`                          | `/ccc`                        | Không, dùng backend mặc định               |
| Hỗn hợp| `/foo` (Prefix), `/foo` (Exact) | `/foo`                        | Có, ưu tiên Exact                          |

#### Khớp nhiều path (Multiple matches)

Trong một số trường hợp, nhiều path trong một Ingress cùng khớp với một request. Trong những
trường hợp đó, ưu tiên trước hết sẽ dành cho path khớp dài nhất. Nếu hai path
vẫn khớp ngang nhau, ưu tiên sẽ dành cho path có loại exact
hơn là loại prefix.

## Wildcard cho hostname (Hostname wildcards)

Host có thể là so khớp chính xác (ví dụ "`foo.bar.com`") hoặc là một wildcard (ví
dụ "`*.foo.com`"). So khớp chính xác yêu cầu header HTTP `host`
phải khớp với trường `host`. So khớp wildcard yêu cầu header HTTP `host`
bằng với phần hậu tố (suffix) của quy tắc wildcard.

| Host        | Header Host       | Khớp?                                                        |
| ----------- |-------------------| -------------------------------------------------------------|
| `*.foo.com` | `bar.foo.com`     | Khớp dựa trên hậu tố chung                                    |
| `*.foo.com` | `baz.bar.foo.com` | Không khớp, wildcard chỉ bao phủ một nhãn DNS duy nhất        |
| `*.foo.com` | `foo.com`         | Không khớp, wildcard chỉ bao phủ một nhãn DNS duy nhất        |

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wildcard-host
spec:
  rules:
  - host: "foo.bar.com"
    http:
      paths:
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: "*.foo.com"
    http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: service2
            port:
              number: 80
```

## Ingress class

Các Ingress có thể được hiện thực bởi những controller khác nhau, thường với cấu hình
khác nhau. Mỗi Ingress nên chỉ định một class — một tham chiếu đến một
tài nguyên IngressClass chứa cấu hình bổ sung, bao gồm tên
của controller sẽ hiện thực class đó.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: external-lb
spec:
  controller: example.com/ingress-controller
  parameters:
    apiGroup: k8s.example.com
    kind: IngressParameters
    name: external-lb
```

Trường `.spec.parameters` của một IngressClass cho phép bạn tham chiếu đến một
tài nguyên khác cung cấp cấu hình liên quan đến IngressClass đó.

Kiểu tham số (parameters) cụ thể cần dùng phụ thuộc vào ingress controller
mà bạn chỉ định trong trường `.spec.controller` của IngressClass.

### Phạm vi của IngressClass (IngressClass scope)

Tùy thuộc vào ingress controller của bạn, bạn có thể sử dụng các tham số
được thiết lập ở phạm vi toàn cluster (cluster-wide), hoặc chỉ cho một namespace.

#### Cluster

Phạm vi mặc định cho các tham số của IngressClass là toàn cluster.

Nếu bạn thiết lập trường `.spec.parameters` mà không thiết lập
`.spec.parameters.scope`, hoặc nếu bạn thiết lập `.spec.parameters.scope` là
`Cluster`, thì IngressClass tham chiếu đến một tài nguyên có phạm vi cluster (cluster-scoped).
`kind` (kết hợp với `apiGroup`) của tham số
tham chiếu đến một API có phạm vi cluster (có thể là một custom resource), và
`name` của tham số định danh một tài nguyên cluster-scoped
cụ thể cho API đó.

Ví dụ:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: external-lb-1
spec:
  controller: example.com/ingress-controller
  parameters:
    # Tham số cho IngressClass này được chỉ định trong một
    # ClusterIngressParameter (API group k8s.example.net) có tên
    # "external-config-1". Định nghĩa này báo cho Kubernetes
    # tìm một tài nguyên tham số có phạm vi cluster.
    scope: Cluster
    apiGroup: k8s.example.net
    kind: ClusterIngressParameter
    name: external-config-1
```

#### Namespaced

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.23 [stable]`

Nếu bạn thiết lập trường `.spec.parameters` và thiết lập
`.spec.parameters.scope` là `Namespace`, thì IngressClass tham chiếu
đến một tài nguyên có phạm vi namespace (namespaced-scoped). Bạn cũng phải thiết lập trường `namespace`
bên trong `.spec.parameters` thành namespace chứa
các tham số mà bạn muốn dùng.

`kind` (kết hợp với `apiGroup`) của tham số
tham chiếu đến một API có phạm vi namespace (ví dụ: ConfigMap), và
`name` của tham số định danh một tài nguyên cụ thể
trong namespace mà bạn đã chỉ định ở `namespace`.

Các tham số phạm vi namespace giúp người vận hành cluster (cluster operator) ủy quyền việc kiểm soát
phần cấu hình (ví dụ: thiết lập bộ cân bằng tải, định nghĩa API gateway)
được sử dụng cho một workload. Nếu bạn dùng tham số phạm vi cluster thì hoặc là:

- đội vận hành cluster phải phê duyệt các thay đổi của đội khác mỗi
  khi có một thay đổi cấu hình mới được áp dụng.
- người vận hành cluster phải định nghĩa các cơ chế kiểm soát truy cập cụ thể, chẳng hạn các
  role và binding [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), cho phép
  đội ứng dụng thay đổi tài nguyên tham số phạm vi cluster.

Bản thân API IngressClass luôn có phạm vi cluster.

Đây là ví dụ về một IngressClass tham chiếu đến các tham số có phạm vi
namespace:

```yaml
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: external-lb-2
spec:
  controller: example.com/ingress-controller
  parameters:
    # Tham số cho IngressClass này được chỉ định trong một
    # IngressParameter (API group k8s.example.com) có tên "external-config",
    # nằm trong namespace "external-configuration".
    scope: Namespace
    apiGroup: k8s.example.com
    kind: IngressParameter
    namespace: external-configuration
    name: external-config
```

### Annotation đã bị loại bỏ (Deprecated annotation) {#deprecated-annotation}

Trước khi tài nguyên IngressClass và trường `ingressClassName` được thêm vào
Kubernetes 1.18, Ingress class được chỉ định bằng annotation
`kubernetes.io/ingress.class` trên Ingress. Annotation này
chưa bao giờ được định nghĩa chính thức, nhưng được các Ingress controller hỗ trợ rộng rãi.

Trường `ingressClassName` mới hơn trên Ingress là sự thay thế cho
annotation đó, nhưng không phải là bản tương đương trực tiếp. Trong khi annotation thường được
dùng để tham chiếu đến tên của Ingress controller sẽ hiện thực
Ingress, thì trường này là một tham chiếu đến tài nguyên IngressClass chứa
cấu hình Ingress bổ sung, bao gồm cả tên của Ingress controller.

### IngressClass mặc định (Default IngressClass) {#default-ingress-class}

Bạn có thể đánh dấu một IngressClass cụ thể làm mặc định cho cluster của bạn. Việc thiết lập
annotation `ingressclass.kubernetes.io/is-default-class` thành `true` trên một
tài nguyên IngressClass sẽ đảm bảo rằng các Ingress mới không chỉ định
trường `ingressClassName` sẽ được gán IngressClass mặc định này.

> **Thận trọng:**
>
> Nếu bạn có nhiều hơn một IngressClass được đánh dấu là mặc định cho cluster,
> admission controller sẽ chặn việc tạo các đối tượng Ingress mới không có
> `ingressClassName` được chỉ định. Bạn có thể giải quyết bằng cách đảm bảo tối đa chỉ có 1
> IngressClass được đánh dấu là mặc định trong cluster của bạn.

Hãy bắt đầu bằng việc định nghĩa một
IngressClass mặc định. Dù vậy, vẫn khuyến nghị chỉ định rõ
IngressClass mặc định:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  labels:
    app.kubernetes.io/component: controller
  name: example-class
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/example-class
```

## Các kiểu Ingress (Types of Ingress)

### Ingress được hậu thuẫn bởi một Service duy nhất {#single-service-ingress}

Có những khái niệm sẵn có của Kubernetes cho phép bạn expose một Service duy nhất
(xem [các lựa chọn thay thế](#alternatives)). Bạn cũng có thể làm điều này với một Ingress bằng cách chỉ định một
*default backend* mà không có quy tắc nào.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  defaultBackend:
    service:
      name: test
      port:
        number: 80
```

Nếu bạn tạo nó bằng `kubectl apply -f`, bạn sẽ có thể xem trạng thái
của Ingress vừa thêm:

```bash
kubectl get ingress test-ingress
```

```
NAME           CLASS         HOSTS   ADDRESS         PORTS   AGE
test-ingress   external-lb   *       203.0.113.123   80      59s
```

Trong đó `203.0.113.123` là IP được Ingress controller cấp phát để phục vụ
Ingress này.

> **Ghi chú:**
>
> Các Ingress controller và bộ cân bằng tải có thể mất một hai phút để cấp phát địa chỉ IP.
> Trước thời điểm đó, bạn thường thấy địa chỉ hiển thị là `<pending>`.

### Fanout đơn giản (Simple fanout)

Một cấu hình fanout định tuyến lưu lượng từ một địa chỉ IP duy nhất đến nhiều hơn một Service,
dựa trên URI HTTP được yêu cầu. Một Ingress cho phép bạn giữ số lượng bộ cân bằng tải
ở mức tối thiểu. Ví dụ, một thiết lập như:

![ingress-fanout-diagram](https://kubernetes.io/docs/images/ingressFanOut.svg)

*Hình. Ingress Fan Out* — [xem sơ đồ trên mermaid.live](https://mermaid.live/edit#pako:eNqNUslOwzAQ_RXLvYCUhMQpUFzUUzkgcUBwbHpw4klr4diR7bCo8O8k2FFbFomLPZq3jP00O1xpDpjijWHtFt09zAuFUCUFKHey8vf6NE7QrdoYsDZumGIb4Oi6NAskNeOoZJKpCgxK4oXwrFVgRyi7nCVXWZKRPMlysv5yD6Q4Xryf1Vq_WzDPooJs9egLNDbolKTpT03JzKgh3zWEztJZ0Niu9L-qZGcdmAMfj4cxvWmreba613z9C0B-AMQD-V_AdA-A4j5QZu0SatRKJhSqhZR0wjmPrDP6CeikrutQxy-Cuy2dtq9RpaU2dJKm6fzI5Glmg0VOLio4_5dLjx27hFSC015KJ2VZHtuQvY2fuHcaE43G0MaCREOow_FV5cMxHZ5-oPX75UM5avuXhXuOI9yAaZjg_aLuBl6B3RYaKDDtSw4166QrcKE-emrXcubghgunDaY1kxYizDqnH99UhakzHYykpWD9hjS--fEJoIELqQ)

Sẽ cần một Ingress như sau:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-fanout-example
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 4200
      - path: /bar
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080
```

Khi bạn tạo Ingress bằng `kubectl apply -f`:

```shell
kubectl describe ingress simple-fanout-example
```

```
Name:             simple-fanout-example
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:4200 (10.8.0.90:4200)
               /bar   service2:8080 (10.8.0.91:8080)
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     22s                loadbalancer-controller  default/test
```

Ingress controller sẽ cung cấp (provision) một bộ cân bằng tải theo cách riêng của từng bản hiện thực
để phục vụ Ingress, miễn là các Service (`service1`, `service2`) tồn tại.
Khi việc đó hoàn tất, bạn có thể thấy địa chỉ của bộ cân bằng tải tại
trường Address.

> **Ghi chú:**
>
> Tùy thuộc vào [Ingress controller](12-ingress-controllers-vi.md)
> bạn đang dùng, bạn có thể cần tạo một
> [Service](82-service-vi.md) default-http-backend.

### Virtual hosting dựa trên tên (Name based virtual hosting)

Virtual host dựa trên tên hỗ trợ định tuyến lưu lượng HTTP đến nhiều hostname trên cùng một địa chỉ IP.

![ingress-namebase-diagram](https://kubernetes.io/docs/images/ingressNameBased.svg)

*Hình. Ingress Name Based Virtual hosting* — [xem sơ đồ trên mermaid.live](https://mermaid.live/edit#pako:eNqNkl9PwyAUxb8KYS-atM1Kp05m9qSJJj4Y97jugcLtRqTQAPVPdN_dVlq3qUt8gZt7zvkBN7xjbgRgiteW1Rt0_zjLNUJcSdD-ZBn21WmcoDu9tuBcXDHN1iDQVWHnSBkmUMEU0xwsSuK5DK5l745QejFNLtMkJVmSZmT1Re9NcTz_uDXOU1QakxTMJtxUHw7ss-SQLhehQEODTsdH4l20Q-zFyc84-Y67pghv5apxHuweMuj9eS2_NiJdPhix-kMgvwQShOyYMNkJoEUYM3PuGkpUKyY1KqVSdCSEiJy35gnoqCzLvo5fpPAbOqlfI26UsXQ0Ho9nB5CnqesRGTnncPYvSqsdUvqp9KRdlI6KojjEkB0mnLgjDRONhqENBYm6oXbLV5V1y6S7-l42_LowlIN2uFm_twqOcAW2YlK0H_i9c-bYb6CCHNO2FFCyRvkc53rbWptaMA83QnpjMS2ZchBh1nizeNMcU28bGEzXkrV_pArN7Sc0rBTu)

Ingress sau đây yêu cầu bộ cân bằng tải phía sau định tuyến các request dựa trên
[header Host](https://tools.ietf.org/html/rfc7230#section-5.4).

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: bar.foo.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service2
            port:
              number: 80
```

Nếu bạn tạo một tài nguyên Ingress mà không định nghĩa host nào trong các quy tắc, thì bất kỳ
lưu lượng web nào đi tới địa chỉ IP của Ingress controller của bạn đều có thể được khớp mà không cần đến
virtual host dựa trên tên.

Ví dụ, Ingress sau định tuyến lưu lượng
yêu cầu cho `first.bar.com` đến `service1`, `second.bar.com` đến `service2`,
và mọi lưu lượng có header host của request không khớp với `first.bar.com`
và `second.bar.com` sẽ đến `service3`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: name-virtual-host-ingress-no-third-host
spec:
  rules:
  - host: first.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: second.bar.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service2
            port:
              number: 80
  - http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: service3
            port:
              number: 80
```

### TLS

Bạn có thể bảo mật một Ingress bằng cách chỉ định một [Secret](109-secret-vi.md)
chứa khóa riêng (private key) và chứng chỉ (certificate) TLS. Tài nguyên Ingress chỉ
hỗ trợ một port TLS duy nhất là 443, và giả định TLS kết thúc tại điểm ingress
(lưu lượng đi tới Service và các Pod của nó ở dạng không mã hóa - plaintext).
Nếu phần cấu hình TLS trong một Ingress chỉ định các host khác nhau, chúng sẽ được
ghép kênh (multiplex) trên cùng một port dựa theo hostname được xác định thông qua
phần mở rộng SNI của TLS (với điều kiện Ingress controller hỗ trợ SNI). Secret TLS
phải chứa các khóa có tên `tls.crt` và `tls.key` chứa chứng chỉ
và khóa riêng dùng cho TLS. Ví dụ:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: testsecret-tls
  namespace: default
data:
  tls.crt: base64 encoded cert
  tls.key: base64 encoded key
type: kubernetes.io/tls
```

Việc tham chiếu Secret này trong một Ingress sẽ báo cho Ingress controller
bảo mật kênh truyền từ client đến bộ cân bằng tải bằng TLS. Bạn cần đảm bảo
rằng Secret TLS bạn tạo đến từ một chứng chỉ có chứa Common
Name (CN), còn gọi là Fully Qualified Domain Name (FQDN), cho `https-example.foo.com`.

> **Ghi chú:**
>
> Lưu ý rằng TLS sẽ không hoạt động trên quy tắc mặc định (default rule) vì
> chứng chỉ khi đó sẽ phải được cấp cho tất cả các sub-domain có thể có. Do đó,
> `hosts` trong phần `tls` cần khớp một cách tường minh với `host` trong phần
> `rules`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls:
  - hosts:
      - https-example.foo.com
    secretName: testsecret-tls
  rules:
  - host: https-example.foo.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 80
```

> **Ghi chú:**
>
> Có sự chênh lệch giữa các tính năng TLS được các ingress controller khác nhau hỗ trợ.
> Bạn nên tham khảo tài liệu của (các) ingress controller mà bạn đã chọn để
> hiểu cách TLS hoạt động trong môi trường của bạn.

### Cân bằng tải (Load balancing) {#load-balancing}

Một Ingress controller được khởi tạo (bootstrap) với một số thiết lập chính sách cân bằng tải
áp dụng cho mọi Ingress, chẳng hạn thuật toán cân bằng tải, cơ chế trọng số (weight) cho
backend, và các thiết lập khác. Các khái niệm cân bằng tải nâng cao hơn
(ví dụ: phiên bền vững - persistent sessions, trọng số động - dynamic weights) chưa được cung cấp thông qua
Ingress. Thay vào đó, bạn có thể có các tính năng này thông qua bộ cân bằng tải được dùng cho
một Service.

Cũng đáng lưu ý rằng mặc dù kiểm tra sức khỏe (health check) không được cung cấp trực tiếp
thông qua Ingress, trong Kubernetes vẫn tồn tại các khái niệm song song như
[readiness probe](274-configure-probes-vi.md)
cho phép bạn đạt được kết quả tương tự. Vui lòng xem tài liệu riêng của từng
controller để biết cách chúng xử lý health check.

## Cập nhật một Ingress (Updating an Ingress)

Để cập nhật một Ingress hiện có nhằm thêm một Host mới, bạn có thể cập nhật nó bằng cách chỉnh sửa tài nguyên:

```shell
kubectl describe ingress test
```

```
Name:             test
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:80 (10.8.0.90:80)
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     35s                loadbalancer-controller  default/test
```

```shell
kubectl edit ingress test
```

Lệnh này mở ra một trình soạn thảo với cấu hình hiện tại ở định dạng YAML.
Hãy sửa nó để thêm Host mới:

```yaml
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          service:
            name: service1
            port:
              number: 80
        path: /foo
        pathType: Prefix
  - host: bar.baz.com
    http:
      paths:
      - backend:
          service:
            name: service2
            port:
              number: 80
        path: /foo
        pathType: Prefix
..
```

Sau khi bạn lưu các thay đổi, kubectl sẽ cập nhật tài nguyên trong API server, từ đó báo cho
Ingress controller cấu hình lại bộ cân bằng tải.

Kiểm chứng điều này:

```shell
kubectl describe ingress test
```

```
Name:             test
Namespace:        default
Address:          178.91.123.132
Default backend:  default-http-backend:80 (10.8.2.3:8080)
Rules:
  Host         Path  Backends
  ----         ----  --------
  foo.bar.com
               /foo   service1:80 (10.8.0.90:80)
  bar.baz.com
               /foo   service2:80 (10.8.0.91:80)
Events:
  Type     Reason  Age                From                     Message
  ----     ------  ----               ----                     -------
  Normal   ADD     45s                loadbalancer-controller  default/test
```

Bạn có thể đạt cùng kết quả bằng cách gọi `kubectl replace -f` trên một file YAML Ingress đã chỉnh sửa.

## Chịu lỗi giữa các availability zone (Failing across availability zones)

Các kỹ thuật để phân tán lưu lượng giữa các miền lỗi (failure domain) khác nhau tùy theo nhà cung cấp cloud.
Vui lòng xem tài liệu của [Ingress controller](12-ingress-controllers-vi.md) tương ứng để biết chi tiết.

## Các lựa chọn thay thế (Alternatives) {#alternatives}

Bạn có thể expose một Service theo nhiều cách không trực tiếp liên quan đến tài nguyên Ingress:

* Dùng [Service.Type=LoadBalancer](82-service-vi.md#loadbalancer)
* Dùng [Service.Type=NodePort](82-service-vi.md#type-nodeport)

## Tiếp theo (What's next)

* Tìm hiểu về [Ingress API](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/ingress-v1/)
* Tìm hiểu về [Ingress controller](12-ingress-controllers-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 5:

1. Cluster lab chưa có ingress controller. Bạn `kubectl apply` một Ingress hợp lệ và thấy nó
   được tạo. Gọi từ ngoài vào có tới ứng dụng không? Cột `ADDRESS` của `kubectl get ingress` sẽ
   hiển thị gì?
2. Một quy tắc dùng `pathType: Prefix` với `path: /foo/bar`. Request `/foo/barbaz` có khớp
   không? Còn `/foo/bar/baz`?
3. Một Ingress khai cả `/foo` kiểu `Prefix` lẫn `/foo` kiểu `Exact`. Request `/foo` đi tới
   backend nào?
4. Bạn để trống `ingressClassName` trên một Ingress mới. Điều gì quyết định nó có được xử lý
   hay không, và trường hợp nào việc tạo Ingress bị chặn thẳng?
5. Secret dùng cho TLS của Ingress phải có những khóa nào, và sau khi qua ingress thì traffic
   tới Pod có còn được mã hóa không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không tới đâu cả.** Bài nói thẳng ở mục *Điều kiện tiên quyết*: bạn phải có một Ingress
   controller để Ingress hoạt động, và "việc chỉ tạo tài nguyên Ingress mà thôi sẽ không có tác
   dụng gì". Địa chỉ trong cột `ADDRESS` do **controller cấp phát**, nên khi chưa có controller
   nó nằm ở **`<pending>`** — trạng thái mà bài mô tả cho khoảng thời gian controller chưa cấp
   xong IP. Controller sẽ được cài ở Lab 5b.
2. `/foo/barbaz` **không khớp**; `/foo/bar/baz` **khớp**. `Prefix` khớp dựa trên tiền tố của
   đường dẫn **được tách bởi dấu `/` và so khớp trên từng phần tử một**. Bài nêu đúng cặp ví dụ
   này: nếu phần tử cuối của path chỉ là một chuỗi con của phần tử cuối trong path của request
   thì đó **không** phải kết quả khớp. Đây là chỗ trực giác "so khớp chuỗi" dẫn người ta đi sai.
3. Tới backend của **`Exact`**. Khi nhiều path cùng khớp một request, ưu tiên trước hết dành cho
   **path khớp dài nhất**; nếu vẫn khớp ngang nhau thì **loại `Exact` được ưu tiên hơn `Prefix`**.
4. Phải tồn tại một **IngressClass mặc định** — được đánh dấu bằng annotation
   `ingressclass.kubernetes.io/is-default-class: "true"` — thì Kubernetes mới gán class đó cho
   Ingress không chỉ định `ingressClassName`. Nếu cluster có **nhiều hơn một** IngressClass được
   đánh dấu mặc định, **admission controller sẽ chặn việc tạo** các Ingress không ghi
   `ingressClassName`. Chỉ có tối đa một class mặc định là cách xử lý.
5. Secret kiểu `kubernetes.io/tls`, chứa hai khóa tên **`tls.crt`** và **`tls.key`**. **Không
   còn mã hóa.** Tài nguyên Ingress hỗ trợ đúng một port TLS là 443 và **giả định TLS kết thúc
   tại điểm ingress** — lưu lượng đi tiếp tới Service và các Pod của nó ở dạng plaintext.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
