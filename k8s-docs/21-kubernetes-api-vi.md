# Kubernetes API (The Kubernetes API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/overview/kubernetes-api/>
>
> Kubernetes API cho phép bạn truy vấn và thao tác trạng thái của các object trong Kubernetes.
> Cốt lõi của control plane Kubernetes là API server và HTTP API mà nó cung cấp. Người dùng, các thành phần khác nhau trong cluster của bạn và các thành phần bên ngoài đều giao tiếp với nhau thông qua API server.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 1 → nhóm [1a](LO-TRINH-ADMIN.md#1a-kiến-trúc-và-mô-hình-điều-khiển),
bài 5/8 · Kiểm chứng ở [Lab 1a](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) phần B5.

Bài này có nhiều chi tiết giao thức (header, hash, protobuf) dành cho người viết client. Ở
giai đoạn 1 bạn là người **dùng** API, không phải người viết client — bỏ qua những chỗ đó.

**Phải hiểu ở lần đọc này:**

- API server là cổng vào duy nhất; mọi thành phần giao tiếp qua nó.
- Cấu trúc đường dẫn: `/api/v1` là **core group**, `/apis/<group>/<version>` là **named API
  group**. Biết đọc một chuỗi dạng `group/version/resource`.
- Versioning nằm ở **cấp API**, không phải cấp field hay cấp cluster.
- `v1alphaN`, `v1betaN`, `v1` là ba mức trưởng thành, với cam kết tương thích khác nhau.
- Object được lưu bền vững trong etcd — mục *Lưu trữ bền vững*, chỉ hai dòng nhưng quan trọng.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Aggregated vs unaggregated discovery, header `Accept`, ETag | chi tiết giao thức cho client | khi viết client |
| Toàn bộ mục *Định nghĩa giao diện OpenAPI* (v2, v3, bảng header, hash URL) | tài liệu tra cứu | khi viết client |
| Tuần tự hóa Protobuf | giao tiếp nội bộ cluster | không cần |
| *Mở rộng API* — custom resource và aggregation layer | là chủ đề riêng của platform admin | giai đoạn 14 |

Lộ trình đã ghi rõ phần API aggregation chỉ đọc lướt ở đây và quay lại ở giai đoạn 14.

---

Cốt lõi của control plane Kubernetes
là API server. API server
cung cấp một HTTP API cho phép người dùng cuối, các thành phần khác nhau trong cluster của bạn và
các thành phần bên ngoài giao tiếp với nhau.

Kubernetes API cho phép bạn truy vấn và thao tác trạng thái của các API object trong Kubernetes
(ví dụ: Pod, Namespace, ConfigMap và Event).

Hầu hết các thao tác có thể được thực hiện thông qua giao diện dòng lệnh
[kubectl](https://kubernetes.io/docs/reference/kubectl/) hoặc các công cụ dòng lệnh khác, chẳng hạn
[kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/), mà bản thân chúng cũng sử dụng API.
Tuy nhiên, bạn cũng có thể truy cập API trực tiếp bằng các lời gọi REST. Kubernetes
cung cấp một bộ [thư viện client](https://kubernetes.io/docs/reference/using-api/client-libraries/)
cho những ai muốn viết ứng dụng sử dụng Kubernetes API.

Mỗi cluster Kubernetes đều công bố đặc tả (specification) của các API mà cluster đó phục vụ.
Kubernetes dùng hai cơ chế để công bố các đặc tả API này; cả hai đều hữu ích
cho khả năng tương tác tự động (automatic interoperability). Ví dụ, công cụ `kubectl` tải về và cache đặc tả
API để hỗ trợ tính năng tự hoàn thành dòng lệnh (command-line completion) cùng các tính năng khác.
Hai cơ chế được hỗ trợ như sau:

- [Discovery API](#discovery-api) cung cấp thông tin về các API của Kubernetes:
  tên API, resource, phiên bản và các thao tác được hỗ trợ. Đây là một thuật ngữ
  riêng của Kubernetes vì nó là một API tách biệt với Kubernetes OpenAPI.
  Nó được thiết kế như một bản tóm tắt ngắn gọn về các resource khả dụng và không
  mô tả chi tiết schema cụ thể của các resource. Để tham khảo về schema của resource,
  hãy xem tài liệu OpenAPI.

- [Tài liệu Kubernetes OpenAPI](#openapi-interface-definition) cung cấp (đầy đủ)
  [schema OpenAPI v2.0 và 3.0](https://www.openapis.org/) cho tất cả các endpoint của Kubernetes API.
  OpenAPI v3 là phương thức được ưu tiên để truy cập OpenAPI vì nó cung cấp
  góc nhìn toàn diện và chính xác hơn về API. Nó bao gồm tất cả các đường dẫn
  API khả dụng, cũng như mọi resource được tiêu thụ (consume) và tạo ra (produce) cho mọi thao tác
  trên mọi endpoint. Nó cũng bao gồm mọi thành phần mở rộng mà cluster hỗ trợ.
  Dữ liệu này là một đặc tả hoàn chỉnh và lớn hơn đáng kể so với dữ liệu từ
  Discovery API.

## Discovery API {#discovery-api}

Kubernetes công bố danh sách tất cả các group version và resource được hỗ trợ thông qua
Discovery API. Với mỗi resource, danh sách này bao gồm:

- Tên
- Phạm vi thuộc cluster hay thuộc namespace (cluster hoặc namespaced scope)
- URL endpoint và các verb được hỗ trợ
- Các tên thay thế
- Group, version, kind

API này khả dụng ở cả dạng tổng hợp (aggregated) lẫn không tổng hợp (unaggregated). Discovery
dạng tổng hợp phục vụ qua hai endpoint, trong khi discovery không tổng hợp phục vụ
một endpoint riêng cho mỗi group version.

### Discovery tổng hợp (Aggregated discovery)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.30 [stable]`

Kubernetes cung cấp hỗ trợ ổn định cho _discovery tổng hợp_ (aggregated discovery), công bố
tất cả resource được cluster hỗ trợ thông qua hai endpoint (`/api` và
`/apis`). Việc truy vấn
endpoint này giúp giảm mạnh số lượng request phải gửi để lấy dữ liệu
discovery từ cluster. Bạn có thể truy cập dữ liệu bằng cách
gửi request tới các endpoint tương ứng kèm header `Accept` chỉ ra
resource discovery tổng hợp:
`Accept: application/json;v=v2;g=apidiscovery.k8s.io;as=APIGroupDiscoveryList`.

Nếu không chỉ ra loại resource bằng header `Accept`, phản hồi mặc định
của endpoint `/api` và `/apis` là một tài liệu discovery
không tổng hợp.

[Tài liệu discovery](https://github.com/kubernetes/kubernetes/blob/release-1.36/api/discovery/aggregated_v2.json)
cho các resource tích hợp sẵn (built-in) có thể được tìm thấy trong kho GitHub của Kubernetes.
Tài liệu GitHub này có thể được dùng làm tài liệu tham khảo về tập hợp cơ sở các resource khả dụng
khi bạn không có sẵn một cluster Kubernetes để truy vấn.

Endpoint này cũng hỗ trợ ETag và mã hóa protobuf.

### Discovery không tổng hợp (Unaggregated discovery)

Khi không có tổng hợp discovery, discovery được công bố theo từng cấp (level), trong đó các
endpoint gốc công bố thông tin discovery cho các tài liệu ở cấp dưới.

Danh sách tất cả group version được cluster hỗ trợ được công bố tại
endpoint `/api` và `/apis`. Ví dụ:

```
{
  "kind": "APIGroupList",
  "apiVersion": "v1",
  "groups": [
    {
      "name": "apiregistration.k8s.io",
      "versions": [
        {
          "groupVersion": "apiregistration.k8s.io/v1",
          "version": "v1"
        }
      ],
      "preferredVersion": {
        "groupVersion": "apiregistration.k8s.io/v1",
        "version": "v1"
      }
    },
    {
      "name": "apps",
      "versions": [
        {
          "groupVersion": "apps/v1",
          "version": "v1"
        }
      ],
      "preferredVersion": {
        "groupVersion": "apps/v1",
        "version": "v1"
      }
    },
    ...
}
```

Cần thêm các request khác để lấy tài liệu discovery cho từng group version tại
`/apis/<group>/<version>` (ví dụ:
`/apis/rbac.authorization.k8s.io/v1alpha1`), endpoint này liệt kê danh sách các
resource được phục vụ dưới một group version cụ thể. Các endpoint này được
kubectl dùng để lấy danh sách resource mà một cluster hỗ trợ.

<a id="#api-specification" />

## Định nghĩa giao diện OpenAPI (OpenAPI interface definition) {#openapi-interface-definition}

Để biết chi tiết về các đặc tả OpenAPI, xem [tài liệu OpenAPI](https://www.openapis.org/).

Kubernetes phục vụ cả OpenAPI v2.0 lẫn OpenAPI v3.0. OpenAPI v3 là
phương thức được ưu tiên để truy cập OpenAPI vì nó cung cấp cách biểu diễn toàn diện hơn
(không mất mát thông tin — lossless) về các resource của Kubernetes. Do những hạn chế của OpenAPI
phiên bản 2, một số field bị loại bỏ khỏi bản OpenAPI được công bố, bao gồm nhưng không
giới hạn ở `default`, `nullable`, `oneOf`.

### OpenAPI V2

API server của Kubernetes phục vụ một đặc tả OpenAPI v2 tổng hợp qua
endpoint `/openapi/v2`. Bạn có thể yêu cầu định dạng phản hồi bằng
các request header như sau:

| Header | Các giá trị có thể dùng | Ghi chú |
|---|---|---|
| `Accept-Encoding` | `gzip` | *có thể không cung cấp header này* |
| `Accept` | `application/com.github.proto-openapi.spec.v2@v1.0+protobuf` | *chủ yếu dùng cho giao tiếp nội bộ cluster* |
| `Accept` | `application/json` | *mặc định* |
| `Accept` | `*` | *phục vụ* `application/json` |

> **Cảnh báo:** Các quy tắc kiểm tra hợp lệ (validation) được công bố trong schema OpenAPI có thể không đầy đủ, và thường là không đầy đủ.
> Quá trình kiểm tra bổ sung diễn ra bên trong API server. Nếu bạn muốn việc kiểm tra chính xác và đầy đủ,
> lệnh `kubectl apply --dry-run=server` sẽ chạy tất cả các bước kiểm tra hợp lệ áp dụng được (và cũng kích hoạt
> các kiểm tra ở giai đoạn admission).

### OpenAPI V3

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.27 [stable]`

Kubernetes hỗ trợ công bố mô tả các API của nó dưới dạng OpenAPI v3.

Một endpoint discovery `/openapi/v3` được cung cấp để xem danh sách tất cả các
group/version khả dụng. Endpoint này chỉ trả về JSON. Các
group/version này được cung cấp theo định dạng sau:

```yaml
{
    "paths": {
        ...,
        "api/v1": {
            "serverRelativeURL": "/openapi/v3/api/v1?hash=CC0E9BFD992D8C59AEC98A1E2336F899E8318D3CF4C68944C3DEC640AF5AB52D864AC50DAA8D145B3494F75FA3CFF939FCBDDA431DAD3CA79738B297795818CF"
        },
        "apis/admissionregistration.k8s.io/v1": {
            "serverRelativeURL": "/openapi/v3/apis/admissionregistration.k8s.io/v1?hash=E19CC93A116982CE5422FC42B590A8AFAD92CDE9AE4D59B5CAAD568F083AD07946E6CB5817531680BCE6E215C16973CD39003B0425F3477CFD854E89A9DB6597"
        },
        ....
    }
}
```

Các URL tương đối này trỏ tới các mô tả OpenAPI bất biến (immutable),
nhằm cải thiện việc cache phía client. API server cũng đặt các HTTP caching header
phù hợp cho mục đích đó (`Expires` là 1 năm trong
tương lai, và `Cache-Control` là `immutable`). Khi một URL lỗi thời được
sử dụng, API server trả về một redirect tới URL mới nhất.

API server của Kubernetes công bố một đặc tả OpenAPI v3 cho mỗi group version
của Kubernetes tại endpoint `/openapi/v3/apis/<group>/<version>?hash=<hash>`.

Tham khảo bảng dưới đây về các request header được chấp nhận.

| Header | Các giá trị có thể dùng | Ghi chú |
|---|---|---|
| `Accept-Encoding` | `gzip` | *có thể không cung cấp header này* |
| `Accept` | `application/com.github.proto-openapi.spec.v3@v1.0+protobuf` | *chủ yếu dùng cho giao tiếp nội bộ cluster* |
| `Accept` | `application/json` | *mặc định* |
| `Accept` | `*` | *phục vụ* `application/json` |

Một hiện thực (implementation) bằng Golang để tải OpenAPI V3 được cung cấp trong package
[`k8s.io/client-go/openapi3`](https://pkg.go.dev/k8s.io/client-go/openapi3).

Kubernetes 1.36 công bố
OpenAPI v2.0 và v3.0; chưa có kế hoạch hỗ trợ 3.1 trong tương lai gần.

### Tuần tự hóa Protobuf (Protobuf serialization)

Kubernetes hiện thực một định dạng tuần tự hóa (serialization) thay thế dựa trên Protobuf,
chủ yếu dành cho giao tiếp nội bộ cluster. Để biết thêm thông tin
về định dạng này, xem đề xuất thiết kế [tuần tự hóa Protobuf trong Kubernetes](https://git.k8s.io/design-proposals-archive/api-machinery/protobuf.md)
và các file Ngôn ngữ Định nghĩa Giao diện (Interface Definition Language — IDL) của từng schema, nằm trong các Go
package định nghĩa các API object.

## Lưu trữ bền vững (Persistence)

Kubernetes lưu trạng thái đã tuần tự hóa của các object bằng cách ghi chúng vào
etcd.

## Các nhóm API và quản lý phiên bản (API groups and versioning)

Để việc loại bỏ field hoặc tái cấu trúc cách biểu diễn resource được dễ dàng hơn,
Kubernetes hỗ trợ nhiều phiên bản API, mỗi phiên bản ở một đường dẫn API khác nhau, chẳng hạn
`/api/v1` hoặc `/apis/rbac.authorization.k8s.io/v1alpha1`.

Việc quản lý phiên bản được thực hiện ở cấp API thay vì ở cấp resource hay field
nhằm bảo đảm API thể hiện một góc nhìn rõ ràng, nhất quán về các resource
và hành vi của hệ thống, đồng thời cho phép kiểm soát quyền truy cập tới các API
đã hết vòng đời (end-of-life) và/hoặc các API thử nghiệm.

Để việc phát triển và mở rộng API được dễ dàng hơn, Kubernetes hiện thực
[các nhóm API (API group)](https://kubernetes.io/docs/reference/using-api/#api-groups) có thể được
[bật hoặc tắt](https://kubernetes.io/docs/reference/using-api/#enabling-or-disabling).

Các API resource được phân biệt bởi API group, loại resource, namespace
(đối với resource thuộc namespace) và tên. API server xử lý việc chuyển đổi giữa
các phiên bản API một cách trong suốt: tất cả các phiên bản khác nhau thực chất đều là các cách biểu diễn
của cùng một dữ liệu được lưu trữ bền vững. API server có thể phục vụ cùng một dữ liệu nền
thông qua nhiều phiên bản API.

Ví dụ, giả sử có hai phiên bản API, `v1` và `v1beta1`, cho cùng một
resource. Nếu ban đầu bạn tạo một object bằng phiên bản `v1beta1` của
API đó, sau này bạn có thể đọc, cập nhật hoặc xóa object đó bằng phiên bản API `v1beta1`
hoặc `v1`, cho đến khi phiên bản `v1beta1` bị ngưng hỗ trợ (deprecated) và gỡ bỏ.
Từ thời điểm đó, bạn vẫn có thể tiếp tục truy cập và chỉnh sửa object bằng API `v1`.

### Các thay đổi của API (API changes)

Bất kỳ hệ thống thành công nào cũng cần phát triển và thay đổi khi các trường hợp sử dụng mới xuất hiện hoặc các trường hợp hiện có thay đổi.
Vì vậy, Kubernetes đã thiết kế Kubernetes API để có thể liên tục thay đổi và phát triển.
Dự án Kubernetes đặt mục tiêu _không_ phá vỡ tính tương thích với các client hiện có, và duy trì
tính tương thích đó trong một khoảng thời gian đủ dài để các dự án khác có cơ hội thích nghi.

Nhìn chung, các API resource mới và các field mới của resource có thể được thêm vào thường xuyên.
Việc loại bỏ resource hoặc field phải tuân theo
[chính sách ngưng hỗ trợ API (API deprecation policy)](https://kubernetes.io/docs/reference/using-api/deprecation-policy/).

Kubernetes cam kết mạnh mẽ trong việc duy trì tính tương thích cho các API chính thức của Kubernetes
một khi chúng đạt mức phổ biến rộng rãi (general availability — GA), thường là ở phiên bản API `v1`. Ngoài ra,
Kubernetes duy trì tính tương thích với dữ liệu đã được lưu trữ qua các phiên bản API _beta_ của các API chính thức của Kubernetes,
và bảo đảm dữ liệu đó có thể được chuyển đổi và truy cập qua các phiên bản API GA khi tính năng trở nên ổn định.

Nếu bạn sử dụng một phiên bản API beta, bạn sẽ cần chuyển sang một phiên bản API beta kế tiếp hoặc phiên bản ổn định
một khi API đó được nâng hạng (graduate). Thời điểm tốt nhất để làm việc này là khi API beta
đang trong giai đoạn ngưng hỗ trợ, vì khi đó các object có thể được truy cập đồng thời
qua cả hai phiên bản API. Khi API beta kết thúc giai đoạn ngưng hỗ trợ
và không còn được phục vụ, bạn bắt buộc phải dùng phiên bản API thay thế.

> **Ghi chú:** Mặc dù Kubernetes cũng đặt mục tiêu duy trì tính tương thích cho các phiên bản API _alpha_, trong một số
> trường hợp điều này là không thể. Nếu bạn dùng bất kỳ phiên bản API alpha nào, hãy kiểm tra release notes
> của Kubernetes khi nâng cấp cluster, phòng trường hợp API đã thay đổi theo cách không tương thích
> và đòi hỏi phải xóa tất cả các object alpha hiện có trước khi nâng cấp.

Tham khảo [tài liệu tham chiếu về phiên bản API (API versions reference)](https://kubernetes.io/docs/reference/using-api/#api-versioning)
để biết thêm chi tiết về định nghĩa các mức phiên bản API.

## Mở rộng API (API Extension)

Kubernetes API có thể được mở rộng theo một trong hai cách:

1. [Custom resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
   cho phép bạn định nghĩa một cách khai báo cách API server cung cấp API resource mà bạn chọn.
1. Bạn cũng có thể mở rộng Kubernetes API bằng cách hiện thực một
   [tầng tổng hợp (aggregation layer)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/).

## Tiếp theo (What's next)

- Tìm hiểu cách mở rộng Kubernetes API bằng cách thêm
  [CustomResourceDefinition](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) của riêng bạn.
- [Kiểm soát truy cập vào Kubernetes API (Controlling Access To The Kubernetes API)](https://kubernetes.io/docs/concepts/security/controlling-access/) mô tả
  cách cluster quản lý xác thực (authentication) và phân quyền (authorization) cho việc truy cập API.
- Tìm hiểu về các API endpoint, các loại resource và ví dụ mẫu bằng cách đọc
  [Tài liệu tham chiếu API (API Reference)](https://kubernetes.io/docs/reference/kubernetes-api/).
- Tìm hiểu điều gì tạo nên một thay đổi tương thích, và cách thay đổi API, trong
  [API changes](https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#readme).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 1:

1. `/api/v1` và `/apis/apps/v1` khác nhau ở chỗ nào?
2. Một resource được nâng từ `v1beta1` lên `v1`. Object bạn đã tạo bằng `v1beta1` có mất
   không? Bạn còn đọc và sửa nó được bằng cách nào?
3. Cluster có một resource ở `v1alpha1`. Điều đó có nghĩa cả cluster đang ở mức alpha không?
4. Object bạn tạo qua API server cuối cùng được lưu ở đâu?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. `/api/v1` là **core group** — group rỗng, nên đường dẫn không mang tên group.
   `/apis/apps/v1` là một **named API group** tên `apps`, version `v1`. Mọi group ngoài core
   đều nằm dưới dạng `/apis/<group>/<version>`.
2. **Không mất.** Các phiên bản API chỉ là những cách biểu diễn khác nhau của **cùng một dữ
   liệu lưu trữ**, và API server chuyển đổi giữa chúng một cách trong suốt. Bạn đọc, sửa, xóa
   được bằng cả `v1beta1` lẫn `v1` cho tới khi `v1beta1` bị ngưng hỗ trợ và gỡ bỏ; sau đó dùng
   `v1`.
3. **Không.** Versioning được thực hiện ở **cấp API group**, không phải cấp cluster. Một
   cluster hoàn toàn có thể vừa phục vụ `v1` cho group này vừa phục vụ `v1alpha1` cho group
   khác. Mức trưởng thành nói về **API đó**, không nói về độ ổn định của cluster.
4. Trong **etcd**. API server ghi trạng thái đã tuần tự hóa của object vào đó — mục *Lưu trữ
   bền vững*.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
