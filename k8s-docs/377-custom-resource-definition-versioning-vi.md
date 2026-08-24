# Các phiên bản trong CustomResourceDefinition (Versions in CustomResourceDefinitions)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/>
>
> Trang này giải thích cách thêm thông tin phiên bản vào
> [CustomResourceDefinition](https://kubernetes.io/docs/reference/kubernetes-api/extend-resources/custom-resource-definition-v1/),
> để chỉ ra mức độ ổn định của CustomResourceDefinition của bạn hoặc để đưa API của bạn lên một
> phiên bản mới kèm việc chuyển đổi giữa các cách biểu diễn API. Trang này cũng mô tả cách nâng
> cấp một object từ phiên bản này sang phiên bản khác.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn cần có hiểu biết ban đầu về [custom resource](179-custom-resources-vi.md).

Kubernetes server của bạn phải ở phiên bản v1.16 hoặc mới hơn. Để kiểm tra phiên bản, hãy
chạy `kubectl version`.

## Tổng quan (Overview)

CustomResourceDefinition API cung cấp một quy trình để giới thiệu và nâng cấp lên các phiên
bản mới của một CustomResourceDefinition.

Khi một CustomResourceDefinition được tạo, phiên bản đầu tiên được đặt trong danh sách
`spec.versions` của CustomResourceDefinition với một mức độ ổn định và một số phiên bản phù
hợp. Ví dụ, `v1beta1` cho biết phiên bản đầu tiên chưa ổn định. Ban đầu, mọi custom resource
object đều sẽ được lưu trữ ở phiên bản này.

Sau khi CustomResourceDefinition được tạo, các client có thể bắt đầu dùng API `v1beta1`.

Về sau, có thể sẽ cần thêm một phiên bản mới, chẳng hạn `v1`.

Thêm một phiên bản mới:

1. Chọn một chiến lược chuyển đổi (conversion strategy). Vì các custom resource object cần có
   khả năng được phục vụ ở cả hai phiên bản, nghĩa là đôi khi chúng sẽ được phục vụ ở một
   phiên bản khác với phiên bản đang được lưu trữ. Để làm được điều đó, đôi khi các custom
   resource object phải được chuyển đổi giữa phiên bản chúng được lưu trữ và phiên bản chúng
   được phục vụ. Nếu việc chuyển đổi có kèm thay đổi schema và cần logic tùy chỉnh, bạn nên
   dùng conversion webhook. Nếu không có thay đổi schema nào, có thể dùng chiến lược chuyển
   đổi mặc định `None`, và khi phục vụ các phiên bản khác nhau thì chỉ trường `apiVersion` bị
   thay đổi.
2. Nếu dùng conversion webhook, hãy tạo và triển khai conversion webhook đó. Xem mục
   [Chuyển đổi bằng webhook](#webhook-conversion) để biết thêm chi tiết.
3. Cập nhật CustomResourceDefinition để đưa phiên bản mới vào danh sách `spec.versions` với
   `served:true`. Đồng thời, đặt trường `spec.conversion` theo chiến lược chuyển đổi đã chọn.
   Nếu dùng conversion webhook, hãy cấu hình trường `spec.conversion.webhookClientConfig` để
   gọi tới webhook.

Sau khi phiên bản mới được thêm vào, các client có thể di trú dần dần sang phiên bản mới.
Việc một số client dùng phiên bản cũ trong khi những client khác dùng phiên bản mới là hoàn
toàn an toàn.

Di trú các object đã lưu trữ sang phiên bản mới:

1. Xem mục [nâng cấp các object hiện có lên storage version mới](#upgrade-existing-objects-to-a-new-stored-version).

Việc các client dùng cả phiên bản cũ lẫn phiên bản mới là an toàn ở cả trước, trong và sau
khi nâng cấp các object lên storage version mới.

Gỡ bỏ một phiên bản cũ:

1. Bảo đảm mọi client đã được di trú hoàn toàn sang phiên bản mới. Bạn có thể xem lại log của
   kube-apiserver để giúp xác định những client nào vẫn còn truy cập qua phiên bản cũ.
2. Đặt `served` thành `false` cho phiên bản cũ trong danh sách `spec.versions`. Nếu vẫn còn
   client nào ngoài dự kiến đang dùng phiên bản cũ, chúng có thể bắt đầu báo lỗi khi cố truy
   cập các custom resource object ở phiên bản cũ. Nếu điều này xảy ra, hãy chuyển phiên bản
   cũ về lại `served:true`, di trú nốt các client còn lại sang phiên bản mới rồi lặp lại bước
   này.
3. Bảo đảm bước [nâng cấp các object hiện có lên storage version mới](#upgrade-existing-objects-to-a-new-stored-version)
   đã hoàn tất.
   1. Kiểm tra rằng `storage` được đặt thành `true` cho phiên bản mới trong danh sách
      `spec.versions` của CustomResourceDefinition.
   2. Kiểm tra rằng phiên bản cũ không còn được liệt kê trong `status.storedVersions` của
      CustomResourceDefinition.
4. Gỡ phiên bản cũ khỏi danh sách `spec.versions` của CustomResourceDefinition.
5. Bỏ phần hỗ trợ chuyển đổi cho phiên bản cũ trong các conversion webhook.

## Chỉ định nhiều phiên bản (Specify multiple versions) {#specify-multiple-versions}

Trường `versions` của CustomResourceDefinition API có thể dùng để hỗ trợ nhiều phiên bản của
các custom resource mà bạn đã phát triển. Các phiên bản có thể có schema khác nhau, và
conversion webhook có thể chuyển đổi custom resource giữa các phiên bản. Việc chuyển đổi bằng
webhook nên tuân theo [các quy ước API của Kubernetes](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-architecture/api-conventions.md)
ở mọi chỗ áp dụng được. Cụ thể, hãy xem
[tài liệu về thay đổi API](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-architecture/api_changes.md)
để biết một tập hợp các cạm bẫy và gợi ý hữu ích.

> **Ghi chú:** Trong `apiextensions.k8s.io/v1beta1`, có một trường `version` thay vì
> `versions`. Trường `version` đã bị deprecated và là tùy chọn, nhưng nếu nó không rỗng thì nó
> phải khớp với phần tử đầu tiên trong trường `versions`.

Ví dụ sau cho thấy một CustomResourceDefinition với hai phiên bản. Với ví dụ đầu tiên, giả
định là mọi phiên bản dùng chung một schema và không có chuyển đổi nào giữa chúng. Các comment
trong YAML cung cấp thêm ngữ cảnh.

#### apiextensions.k8s.io/v1

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name phải khớp với các trường spec bên dưới, và có dạng: <plural>.<group>
  name: crontabs.example.com
spec:
  # tên group dùng cho REST API: /apis/<group>/<version>
  group: example.com
  # danh sách các phiên bản được CustomResourceDefinition này hỗ trợ
  versions:
  - name: v1beta1
    # Mỗi phiên bản có thể được bật/tắt bằng cờ Served.
    served: true
    # Một và chỉ một phiên bản được đánh dấu là storage version.
    storage: true
    # Bắt buộc phải có schema
    schema:
      openAPIV3Schema:
        type: object
        properties:
          host:
            type: string
          port:
            type: string
  - name: v1
    served: true
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          host:
            type: string
          port:
            type: string
  # Mục conversion được giới thiệu từ Kubernetes 1.13+ với giá trị mặc định là
  # chuyển đổi None (trường con strategy đặt bằng None).
  conversion:
    # Chuyển đổi None giả định mọi phiên bản dùng chung một schema và chỉ đặt trường apiVersion
    # của custom resource về giá trị đúng
    strategy: None
  # hoặc Namespaced hoặc Cluster
  scope: Namespaced
  names:
    # tên số nhiều dùng trong URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # tên số ít dùng làm bí danh trên CLI và để hiển thị
    singular: crontab
    # kind thường là kiểu số ít viết theo CamelCase. Manifest resource của bạn dùng giá trị này.
    kind: CronTab
    # shortNames cho phép dùng chuỗi ngắn hơn để khớp resource của bạn trên CLI
    shortNames:
    - ct
```

#### apiextensions.k8s.io/v1beta1

```yaml
# Đã bị deprecated ở v1.16, thay bằng apiextensions.k8s.io/v1
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name phải khớp với các trường spec bên dưới, và có dạng: <plural>.<group>
  name: crontabs.example.com
spec:
  # tên group dùng cho REST API: /apis/<group>/<version>
  group: example.com
  # danh sách các phiên bản được CustomResourceDefinition này hỗ trợ
  versions:
  - name: v1beta1
    # Mỗi phiên bản có thể được bật/tắt bằng cờ Served.
    served: true
    # Một và chỉ một phiên bản được đánh dấu là storage version.
    storage: true
  - name: v1
    served: true
    storage: false
  validation:
    openAPIV3Schema:
      type: object
      properties:
        host:
          type: string
        port:
          type: string
  # Mục conversion được giới thiệu từ Kubernetes 1.13+ với giá trị mặc định là
  # chuyển đổi None (trường con strategy đặt bằng None).
  conversion:
    # Chuyển đổi None giả định mọi phiên bản dùng chung một schema và chỉ đặt trường apiVersion
    # của custom resource về giá trị đúng
    strategy: None
  # hoặc Namespaced hoặc Cluster
  scope: Namespaced
  names:
    # tên số nhiều dùng trong URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # tên số ít dùng làm bí danh trên CLI và để hiển thị
    singular: crontab
    # kind thường là kiểu số ít viết theo PascalCase. Manifest resource của bạn dùng giá trị này.
    kind: CronTab
    # shortNames cho phép dùng chuỗi ngắn hơn để khớp resource của bạn trên CLI
    shortNames:
    - ct
```

Bạn có thể lưu CustomResourceDefinition vào một file YAML, rồi dùng `kubectl apply` để tạo nó.

```shell
kubectl apply -f my-versioned-crontab.yaml
```

Sau khi tạo, API server bắt đầu phục vụ từng phiên bản đã được bật tại một endpoint HTTP REST.
Trong ví dụ trên, các phiên bản API sẵn có tại `/apis/example.com/v1beta1` và
`/apis/example.com/v1`.

### Thứ tự ưu tiên của phiên bản (Version priority)

Bất kể thứ tự các phiên bản được định nghĩa trong một CustomResourceDefinition như thế nào,
phiên bản có mức ưu tiên cao nhất sẽ được kubectl dùng làm phiên bản mặc định để truy cập
object. Mức ưu tiên được xác định bằng cách phân tích trường _name_ để lấy ra số phiên bản,
mức độ ổn định (GA, Beta hoặc Alpha) và thứ tự trong mức độ ổn định đó.

Thuật toán dùng để sắp xếp các phiên bản được thiết kế để sắp xếp phiên bản theo cùng cách mà
dự án Kubernetes sắp xếp các phiên bản Kubernetes. Phiên bản bắt đầu bằng `v`, theo sau là một
số, một chỉ định tùy chọn `beta` hoặc `alpha`, và thông tin đánh số phiên bản bổ sung tùy
chọn. Nói chung, một chuỗi phiên bản có thể trông như `v2` hoặc `v2beta1`. Các phiên bản được
sắp xếp theo thuật toán sau:

- Các mục tuân theo mẫu phiên bản của Kubernetes được xếp trước những mục không tuân theo.
- Với các mục tuân theo mẫu phiên bản của Kubernetes, phần số của chuỗi phiên bản được sắp xếp
  từ lớn tới nhỏ.
- Nếu chuỗi `beta` hoặc `alpha` đứng sau phần số đầu tiên, chúng được sắp xếp theo đúng thứ tự
  đó, sau chuỗi tương đương không có hậu tố `beta` hay `alpha` (chuỗi này được coi là phiên
  bản GA).
- Nếu có thêm một số nữa đứng sau `beta` hoặc `alpha`, các số đó cũng được sắp xếp từ lớn tới
  nhỏ.
- Các chuỗi không khớp định dạng trên được sắp xếp theo thứ tự chữ cái và phần số không được
  xử lý đặc biệt. Chú ý rằng trong ví dụ dưới đây, `foo1` được xếp trên `foo10`. Điều này khác
  với cách sắp xếp phần số của những mục có tuân theo mẫu phiên bản của Kubernetes.

Danh sách phiên bản đã sắp xếp sau đây có thể giúp bạn hình dung rõ hơn:

```none
- v10
- v2
- v1
- v11beta2
- v10beta3
- v3beta1
- v12alpha1
- v11alpha2
- foo1
- foo10
```

Với ví dụ ở mục [Chỉ định nhiều phiên bản](#specify-multiple-versions), thứ tự sắp xếp phiên
bản là `v1`, rồi tới `v1beta1`. Điều này khiến lệnh kubectl dùng `v1` làm phiên bản mặc định
trừ khi object được cung cấp có chỉ định phiên bản.

### Deprecate một phiên bản (Version deprecation)

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.19 [stable]`

Bắt đầu từ v1.19, một CustomResourceDefinition có thể chỉ ra rằng một phiên bản cụ thể của
resource mà nó định nghĩa đã bị deprecated. Khi có request API tới phiên bản đã deprecated của
resource đó, một thông điệp cảnh báo được trả về trong header của phản hồi API. Thông điệp
cảnh báo cho từng phiên bản đã deprecated của resource có thể được tùy chỉnh nếu bạn muốn.

Một thông điệp cảnh báo tùy chỉnh nên chỉ ra API group, phiên bản và kind đã bị deprecated, và
nên chỉ ra API group, phiên bản và kind nào nên dùng thay thế, nếu có.

#### apiextensions.k8s.io/v1

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.example.com
spec:
  group: example.com
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
  scope: Namespaced
  versions:
  - name: v1alpha1
    served: true
    storage: false
    # Điều này cho biết phiên bản v1alpha1 của custom resource đã bị deprecated.
    # Request API tới phiên bản này nhận được một header cảnh báo trong phản hồi của server.
    deprecated: true
    # Dòng này ghi đè cảnh báo mặc định trả về cho các API client gửi request API v1alpha1.
    deprecationWarning: "example.com/v1alpha1 CronTab is deprecated; see http://example.com/v1alpha1-v1 for instructions to migrate to example.com/v1 CronTab"

    schema: ...
  - name: v1beta1
    served: true
    # Điều này cho biết phiên bản v1beta1 của custom resource đã bị deprecated.
    # Request API tới phiên bản này nhận được một header cảnh báo trong phản hồi của server.
    # Một thông điệp cảnh báo mặc định được trả về cho phiên bản này.
    deprecated: true
    schema: ...
  - name: v1
    served: true
    storage: true
    schema: ...
```

#### apiextensions.k8s.io/v1beta1

```yaml
# Đã bị deprecated ở v1.16, thay bằng apiextensions.k8s.io/v1
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.example.com
spec:
  group: example.com
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
  scope: Namespaced
  validation: ...
  versions:
  - name: v1alpha1
    served: true
    storage: false
    # Điều này cho biết phiên bản v1alpha1 của custom resource đã bị deprecated.
    # Request API tới phiên bản này nhận được một header cảnh báo trong phản hồi của server.
    deprecated: true
    # Dòng này ghi đè cảnh báo mặc định trả về cho các API client gửi request API v1alpha1.
    deprecationWarning: "example.com/v1alpha1 CronTab is deprecated; see http://example.com/v1alpha1-v1 for instructions to migrate to example.com/v1 CronTab"
  - name: v1beta1
    served: true
    # Điều này cho biết phiên bản v1beta1 của custom resource đã bị deprecated.
    # Request API tới phiên bản này nhận được một header cảnh báo trong phản hồi của server.
    # Một thông điệp cảnh báo mặc định được trả về cho phiên bản này.
    deprecated: true
  - name: v1
    served: true
    storage: true
```

### Gỡ bỏ một phiên bản (Version removal)

Không thể bỏ một phiên bản API cũ khỏi manifest của CustomResourceDefinition cho tới khi dữ
liệu đã lưu trữ hiện có được di trú sang phiên bản API mới hơn trên mọi cluster từng phục vụ
phiên bản cũ của custom resource, và phiên bản cũ đã được gỡ khỏi `status.storedVersions` của
CustomResourceDefinition.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.example.com
spec:
  group: example.com
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
  scope: Namespaced
  versions:
  - name: v1beta1
    # Điều này cho biết phiên bản v1beta1 của custom resource không còn được phục vụ nữa.
    # Request API tới phiên bản này nhận được lỗi not found trong phản hồi của server.
    served: false
    schema: ...
  - name: v1
    served: true
    # Phiên bản mới được phục vụ nên được đặt làm storage version
    storage: true
    schema: ...
```

## Chuyển đổi bằng webhook (Webhook conversion) {#webhook-conversion}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.16 [stable]`

> **Ghi chú:** Chuyển đổi bằng webhook đã ở mức beta từ 1.15, và ở mức alpha từ Kubernetes
> 1.13. Feature gate `CustomResourceWebhookConversion` phải được bật, điều này tự động đúng
> với nhiều cluster đối với các tính năng beta. Vui lòng tham khảo tài liệu về
> [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
> để biết thêm thông tin.

Ví dụ ở trên dùng chuyển đổi None giữa các phiên bản, tức là khi chuyển đổi chỉ đặt lại trường
`apiVersion` và không thay đổi phần còn lại của object. API server cũng hỗ trợ chuyển đổi bằng
webhook, tức gọi tới một dịch vụ bên ngoài trong trường hợp cần chuyển đổi. Ví dụ khi:

* custom resource được yêu cầu ở một phiên bản khác với storage version.
* Watch được tạo ở một phiên bản nhưng object bị thay đổi lại được lưu ở một phiên bản khác.
* request PUT của custom resource ở một phiên bản khác với storage version.

Để bao phủ tất cả các trường hợp này và để tối ưu việc chuyển đổi ở phía API server, một
request chuyển đổi có thể chứa nhiều object nhằm giảm thiểu số lần gọi ra bên ngoài. Webhook
nên thực hiện các chuyển đổi này một cách độc lập với nhau.

### Viết một conversion webhook server (Write a conversion webhook server)

Vui lòng tham khảo phần hiện thực của
[custom resource conversion webhook server](https://github.com/kubernetes/kubernetes/tree/v1.25.3/test/images/agnhost/crd-conversion-webhook/main.go)
đã được kiểm chứng trong một bài test e2e của Kubernetes. Webhook này xử lý các request
`ConversionReview` do các API server gửi tới, và gửi lại kết quả chuyển đổi được đóng gói
trong `ConversionResponse`. Lưu ý rằng request chứa một danh sách các custom resource cần được
chuyển đổi một cách độc lập mà không được thay đổi thứ tự các object.
Server ví dụ này được tổ chức theo cách có thể tái sử dụng cho các chuyển đổi khác. Phần lớn
code dùng chung nằm trong
[file framework](https://github.com/kubernetes/kubernetes/tree/v1.25.3/test/images/agnhost/crd-conversion-webhook/converter/framework.go),
chỉ để lại
[một hàm](https://github.com/kubernetes/kubernetes/tree/v1.25.3/test/images/agnhost/crd-conversion-webhook/converter/example_converter.go#L29-L80)
cần được hiện thực cho từng loại chuyển đổi khác nhau.

> **Ghi chú:** Conversion webhook server ví dụ để trống trường `ClientAuth`
> ([xem tại đây](https://github.com/kubernetes/kubernetes/tree/v1.25.3/test/images/agnhost/crd-conversion-webhook/config.go#L47-L48)),
> nên giá trị mặc định là `NoClientCert`. Điều này nghĩa là webhook server không xác thực danh
> tính của các client, vốn được cho là các API server. Nếu bạn cần mutual TLS hoặc các cách
> khác để xác thực client, hãy xem cách
> [xác thực các API server](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#authenticate-apiservers).

#### Những thay đổi được phép (Permissible mutations)

Một conversion webhook không được phép thay đổi bất cứ thứ gì bên trong `metadata` của object
đã chuyển đổi, ngoại trừ `labels` và `annotations`. Các nỗ lực thay đổi `name`, `UID` và
`namespace` sẽ bị từ chối và làm thất bại request đã gây ra việc chuyển đổi. Mọi thay đổi khác
đều bị bỏ qua.

### Triển khai conversion webhook service (Deploy the conversion webhook service)

Tài liệu về việc triển khai conversion webhook giống với tài liệu về
[dịch vụ admission webhook ví dụ](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#deploy-the-admission-webhook-service).
Các mục tiếp theo giả định rằng conversion webhook server được triển khai thành một service
tên là `example-conversion-webhook-server` trong namespace `default` và phục vụ lưu lượng trên
đường dẫn `/crdconvert`.

> **Ghi chú:** Khi webhook server được triển khai vào trong cluster Kubernetes dưới dạng một
> service, nó phải được expose qua một service trên port 443 (bản thân server có thể dùng port
> bất kỳ, nhưng object Service phải ánh xạ nó về port 443). Giao tiếp giữa API server và
> webhook service có thể thất bại nếu dùng port khác cho service.

### Cấu hình CustomResourceDefinition để dùng conversion webhook (Configure CustomResourceDefinition to use conversion webhooks)

Ví dụ chuyển đổi `None` có thể được mở rộng để dùng conversion webhook bằng cách sửa mục
`conversion` của `spec`:

#### apiextensions.k8s.io/v1

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name phải khớp với các trường spec bên dưới, và có dạng: <plural>.<group>
  name: crontabs.example.com
spec:
  # tên group dùng cho REST API: /apis/<group>/<version>
  group: example.com
  # danh sách các phiên bản được CustomResourceDefinition này hỗ trợ
  versions:
  - name: v1beta1
    # Mỗi phiên bản có thể được bật/tắt bằng cờ Served.
    served: true
    # Một và chỉ một phiên bản được đánh dấu là storage version.
    storage: true
    # Mỗi phiên bản có thể định nghĩa schema riêng khi không có schema
    # nào được định nghĩa ở mức trên cùng.
    schema:
      openAPIV3Schema:
        type: object
        properties:
          hostPort:
            type: string
  - name: v1
    served: true
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          host:
            type: string
          port:
            type: string
  conversion:
    # chiến lược Webhook chỉ thị API server gọi một webhook bên ngoài cho mọi chuyển đổi giữa các custom resource.
    strategy: Webhook
    # webhook là bắt buộc khi strategy là `Webhook` và nó cấu hình endpoint webhook mà API server sẽ gọi.
    webhook:
      # conversionReviewVersions cho biết webhook hiểu/ưu tiên những phiên bản ConversionReview nào.
      # Phiên bản đầu tiên trong danh sách mà API server hiểu được sẽ được gửi tới webhook.
      # Webhook phải phản hồi bằng một object ConversionReview ở đúng phiên bản mà nó nhận được.
      conversionReviewVersions: ["v1","v1beta1"]
      clientConfig:
        service:
          namespace: default
          name: example-conversion-webhook-server
          path: /crdconvert
        caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle>...tLS0K"
  # hoặc Namespaced hoặc Cluster
  scope: Namespaced
  names:
    # tên số nhiều dùng trong URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # tên số ít dùng làm bí danh trên CLI và để hiển thị
    singular: crontab
    # kind thường là kiểu số ít viết theo CamelCase. Manifest resource của bạn dùng giá trị này.
    kind: CronTab
    # shortNames cho phép dùng chuỗi ngắn hơn để khớp resource của bạn trên CLI
    shortNames:
    - ct
```

#### apiextensions.k8s.io/v1beta1

```yaml
# Đã bị deprecated ở v1.16, thay bằng apiextensions.k8s.io/v1
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name phải khớp với các trường spec bên dưới, và có dạng: <plural>.<group>
  name: crontabs.example.com
spec:
  # tên group dùng cho REST API: /apis/<group>/<version>
  group: example.com
  # cắt bỏ (prune) các trường của object không được khai báo trong các schema OpenAPI bên dưới.
  preserveUnknownFields: false
  # danh sách các phiên bản được CustomResourceDefinition này hỗ trợ
  versions:
  - name: v1beta1
    # Mỗi phiên bản có thể được bật/tắt bằng cờ Served.
    served: true
    # Một và chỉ một phiên bản được đánh dấu là storage version.
    storage: true
    # Mỗi phiên bản có thể định nghĩa schema riêng khi không có schema
    # nào được định nghĩa ở mức trên cùng.
    schema:
      openAPIV3Schema:
        type: object
        properties:
          hostPort:
            type: string
  - name: v1
    served: true
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          host:
            type: string
          port:
            type: string
  conversion:
    # chiến lược Webhook chỉ thị API server gọi một webhook bên ngoài cho mọi chuyển đổi giữa các custom resource.
    strategy: Webhook
    # webhookClientConfig là bắt buộc khi strategy là `Webhook` và nó cấu hình endpoint webhook mà API server sẽ gọi.
    webhookClientConfig:
      service:
        namespace: default
        name: example-conversion-webhook-server
        path: /crdconvert
      caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle>...tLS0K"
  # hoặc Namespaced hoặc Cluster
  scope: Namespaced
  names:
    # tên số nhiều dùng trong URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # tên số ít dùng làm bí danh trên CLI và để hiển thị
    singular: crontab
    # kind thường là kiểu số ít viết theo CamelCase. Manifest resource của bạn dùng giá trị này.
    kind: CronTab
    # shortNames cho phép dùng chuỗi ngắn hơn để khớp resource của bạn trên CLI
    shortNames:
    - ct
```

Bạn có thể lưu CustomResourceDefinition vào một file YAML, rồi dùng `kubectl apply` để áp dụng
nó.

```shell
kubectl apply -f my-versioned-crontab-with-conversion.yaml
```

Hãy chắc chắn rằng dịch vụ chuyển đổi đã sẵn sàng và đang chạy trước khi áp dụng các thay đổi
mới.

### Liên hệ với webhook (Contacting the webhook)

Khi API server đã xác định rằng một request cần được gửi tới conversion webhook, nó cần biết
cách liên hệ với webhook đó. Điều này được chỉ định trong mục `webhookClientConfig` của phần
cấu hình webhook.

Conversion webhook có thể được gọi qua một URL hoặc qua một tham chiếu service, và có thể tùy
chọn kèm theo một CA bundle tùy chỉnh để dùng khi xác minh kết nối TLS.

### URL

`url` cho biết vị trí của webhook, ở dạng URL chuẩn (`scheme://host:port/path`).

Phần `host` không nên trỏ tới một service đang chạy trong cluster; thay vào đó hãy dùng tham
chiếu service bằng cách chỉ định trường `service`. Ở một số apiserver, host có thể được phân
giải qua DNS bên ngoài (nghĩa là `kube-apiserver` không thể phân giải DNS trong cluster vì như
vậy là vi phạm phân tầng). `host` cũng có thể là một địa chỉ IP.

Xin lưu ý rằng việc dùng `localhost` hoặc `127.0.0.1` làm `host` là rủi ro, trừ khi bạn hết
sức cẩn thận để chạy webhook này trên mọi host có chạy apiserver có thể cần gọi tới webhook.
Những cách cài đặt như vậy nhiều khả năng sẽ không có tính di động hoặc không dễ chạy trên một
cluster mới.

Scheme phải là "https"; URL phải bắt đầu bằng "https://".

Không được phép dùng user hoặc basic auth (ví dụ "user:password@"). Fragment ("#...") và query
parameter ("?...") cũng không được phép.

Dưới đây là một ví dụ về conversion webhook được cấu hình để gọi tới một URL (và mong đợi
chứng chỉ TLS được xác minh bằng system trust root, nên không chỉ định caBundle):

#### apiextensions.k8s.io/v1

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
...
spec:
  ...
  conversion:
    strategy: Webhook
    webhook:
      clientConfig:
        url: "https://my-webhook.example.com:9443/my-webhook-path"
...
```

#### apiextensions.k8s.io/v1beta1

```yaml
# Đã bị deprecated ở v1.16, thay bằng apiextensions.k8s.io/v1
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
...
spec:
  ...
  conversion:
    strategy: Webhook
    webhookClientConfig:
      url: "https://my-webhook.example.com:9443/my-webhook-path"
...
```

### Tham chiếu Service (Service Reference)

Mục `service` bên trong `webhookClientConfig` là một tham chiếu tới service của conversion
webhook. Nếu webhook chạy bên trong cluster, bạn nên dùng `service` thay vì `url`. Namespace
và tên của service là bắt buộc. Port là tùy chọn và mặc định là 443. Path là tùy chọn và mặc
định là "/".

Dưới đây là ví dụ về một webhook được cấu hình để gọi tới một service trên port "1234" tại
đường dẫn con "/my-path", và để xác minh kết nối TLS đối chiếu với ServerName
`my-service-name.my-service-namespace.svc` bằng một CA bundle tùy chỉnh.

#### apiextensions.k8s.io/v1

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
...
spec:
  ...
  conversion:
    strategy: Webhook
    webhook:
      clientConfig:
        service:
          namespace: my-service-namespace
          name: my-service-name
          path: /my-path
          port: 1234
        caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle>...tLS0K"
...
```

#### apiextensions.k8s.io/v1beta1

```yaml
# Đã bị deprecated ở v1.16, thay bằng apiextensions.k8s.io/v1
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
...
spec:
  ...
  conversion:
    strategy: Webhook
    webhookClientConfig:
      service:
        namespace: my-service-namespace
        name: my-service-name
        path: /my-path
        port: 1234
      caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle>...tLS0K"
...
```

## Request và response của webhook (Webhook request and response)

### Request

Webhook nhận được một request POST, với `Content-Type: application/json`, và thân request là
một object `ConversionReview` thuộc API group `apiextensions.k8s.io` được serialize sang JSON.

Webhook có thể chỉ định chúng chấp nhận những phiên bản `ConversionReview` nào bằng trường
`conversionReviewVersions` trong CustomResourceDefinition của chúng:

#### apiextensions.k8s.io/v1

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
...
spec:
  ...
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1", "v1beta1"]
      ...
```

`conversionReviewVersions` là trường bắt buộc khi tạo custom resource definition
`apiextensions.k8s.io/v1`. Webhook bắt buộc phải hỗ trợ ít nhất một phiên bản
`ConversionReview` mà API server hiện tại và API server trước đó đều hiểu được.

#### apiextensions.k8s.io/v1beta1

```yaml
# Đã bị deprecated ở v1.16, thay bằng apiextensions.k8s.io/v1
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
...
spec:
  ...
  conversion:
    strategy: Webhook
    conversionReviewVersions: ["v1", "v1beta1"]
    ...
```

Nếu không chỉ định `conversionReviewVersions` nào, giá trị mặc định khi tạo custom resource
definition `apiextensions.k8s.io/v1beta1` là `v1beta1`.

API server gửi phiên bản `ConversionReview` đầu tiên trong danh sách `conversionReviewVersions`
mà nó hỗ trợ. Nếu API server không hỗ trợ phiên bản nào trong danh sách, custom resource
definition sẽ không được phép tạo. Nếu một API server gặp cấu hình conversion webhook đã được
tạo từ trước và cấu hình đó không hỗ trợ bất kỳ phiên bản `ConversionReview` nào mà API server
biết cách gửi, thì các lần gọi tới webhook sẽ thất bại.

Ví dụ sau cho thấy dữ liệu chứa trong một object `ConversionReview` cho một request chuyển đổi
các object `CronTab` sang `example.com/v1`:

#### apiextensions.k8s.io/v1

```yaml
{
  "apiVersion": "apiextensions.k8s.io/v1",
  "kind": "ConversionReview",
  "request": {
    # uid ngẫu nhiên, định danh duy nhất cho lần gọi chuyển đổi này
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",

    # API group và phiên bản mà các object cần được chuyển đổi sang
    "desiredAPIVersion": "example.com/v1",

    # Danh sách các object cần chuyển đổi.
    # Có thể chứa một hoặc nhiều object, ở một hoặc nhiều phiên bản.
    "objects": [
      {
        "kind": "CronTab",
        "apiVersion": "example.com/v1beta1",
        "metadata": {
          "creationTimestamp": "2019-09-04T14:03:02Z",
          "name": "local-crontab",
          "namespace": "default",
          "resourceVersion": "143",
          "uid": "3415a7fc-162b-4300-b5da-fd6083580d66"
        },
        "hostPort": "localhost:1234"
      },
      {
        "kind": "CronTab",
        "apiVersion": "example.com/v1beta1",
        "metadata": {
          "creationTimestamp": "2019-09-03T13:02:01Z",
          "name": "remote-crontab",
          "resourceVersion": "12893",
          "uid": "359a83ec-b575-460d-b553-d859cedde8a0"
        },
        "hostPort": "example.com:2345"
      }
    ]
  }
}
```

#### apiextensions.k8s.io/v1beta1

```yaml
{
  # Đã bị deprecated ở v1.16, thay bằng apiextensions.k8s.io/v1
  "apiVersion": "apiextensions.k8s.io/v1beta1",
  "kind": "ConversionReview",
  "request": {
    # uid ngẫu nhiên, định danh duy nhất cho lần gọi chuyển đổi này
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",

    # API group và phiên bản mà các object cần được chuyển đổi sang
    "desiredAPIVersion": "example.com/v1",

    # Danh sách các object cần chuyển đổi.
    # Có thể chứa một hoặc nhiều object, ở một hoặc nhiều phiên bản.
    "objects": [
      {
        "kind": "CronTab",
        "apiVersion": "example.com/v1beta1",
        "metadata": {
          "creationTimestamp": "2019-09-04T14:03:02Z",
          "name": "local-crontab",
          "namespace": "default",
          "resourceVersion": "143",
          "uid": "3415a7fc-162b-4300-b5da-fd6083580d66"
        },
        "hostPort": "localhost:1234"
      },
      {
        "kind": "CronTab",
        "apiVersion": "example.com/v1beta1",
        "metadata": {
          "creationTimestamp": "2019-09-03T13:02:01Z",
          "name": "remote-crontab",
          "resourceVersion": "12893",
          "uid": "359a83ec-b575-460d-b553-d859cedde8a0"
        },
        "hostPort": "example.com:2345"
      }
    ]
  }
}
```

### Response

Webhook phản hồi bằng mã trạng thái HTTP 200, `Content-Type: application/json`, và thân phản
hồi chứa một object `ConversionReview` (ở cùng phiên bản mà chúng đã nhận được), với mục
`response` đã được điền, được serialize sang JSON.

Nếu việc chuyển đổi thành công, webhook nên trả về một mục `response` chứa các trường sau:

* `uid`, sao chép từ `request.uid` đã gửi tới webhook
* `result`, đặt bằng `{"status":"Success"}`
* `convertedObjects`, chứa toàn bộ các object từ `request.objects`, đã được chuyển đổi sang
  `request.desiredAPIVersion`

Ví dụ về một phản hồi thành công tối giản từ webhook:

#### apiextensions.k8s.io/v1

```yaml
{
  "apiVersion": "apiextensions.k8s.io/v1",
  "kind": "ConversionReview",
  "response": {
    # phải khớp với <request.uid>
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "result": {
      "status": "Success"
    },
    # Các object phải khớp thứ tự của request.objects, và có apiVersion đặt bằng <request.desiredAPIVersion>.
    # Webhook không được thay đổi các trường kind, metadata.uid, metadata.name và metadata.namespace.
    # Webhook có thể thay đổi các trường metadata.labels và metadata.annotations.
    # Mọi thay đổi khác của webhook đối với các trường metadata đều bị bỏ qua.
    "convertedObjects": [
      {
        "kind": "CronTab",
        "apiVersion": "example.com/v1",
        "metadata": {
          "creationTimestamp": "2019-09-04T14:03:02Z",
          "name": "local-crontab",
          "namespace": "default",
          "resourceVersion": "143",
          "uid": "3415a7fc-162b-4300-b5da-fd6083580d66"
        },
        "host": "localhost",
        "port": "1234"
      },
      {
        "kind": "CronTab",
        "apiVersion": "example.com/v1",
        "metadata": {
          "creationTimestamp": "2019-09-03T13:02:01Z",
          "name": "remote-crontab",
          "resourceVersion": "12893",
          "uid": "359a83ec-b575-460d-b553-d859cedde8a0"
        },
        "host": "example.com",
        "port": "2345"
      }
    ]
  }
}
```

#### apiextensions.k8s.io/v1beta1

```yaml
{
  # Đã bị deprecated ở v1.16, thay bằng apiextensions.k8s.io/v1
  "apiVersion": "apiextensions.k8s.io/v1beta1",
  "kind": "ConversionReview",
  "response": {
    # phải khớp với <request.uid>
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "result": {
      "status": "Failed"
    },
    # Các object phải khớp thứ tự của request.objects, và có apiVersion đặt bằng <request.desiredAPIVersion>.
    # Webhook không được thay đổi các trường kind, metadata.uid, metadata.name và metadata.namespace.
    # Webhook có thể thay đổi các trường metadata.labels và metadata.annotations.
    # Mọi thay đổi khác của webhook đối với các trường metadata đều bị bỏ qua.
    "convertedObjects": [
      {
        "kind": "CronTab",
        "apiVersion": "example.com/v1",
        "metadata": {
          "creationTimestamp": "2019-09-04T14:03:02Z",
          "name": "local-crontab",
          "namespace": "default",
          "resourceVersion": "143",
          "uid": "3415a7fc-162b-4300-b5da-fd6083580d66"
        },
        "host": "localhost",
        "port": "1234"
      },
      {
        "kind": "CronTab",
        "apiVersion": "example.com/v1",
        "metadata": {
          "creationTimestamp": "2019-09-03T13:02:01Z",
          "name": "remote-crontab",
          "resourceVersion": "12893",
          "uid": "359a83ec-b575-460d-b553-d859cedde8a0"
        },
        "host": "example.com",
        "port": "2345"
      }
    ]
  }
}
```

Nếu việc chuyển đổi thất bại, webhook nên trả về một mục `response` chứa các trường sau:

* `uid`, sao chép từ `request.uid` đã gửi tới webhook
* `result`, đặt bằng `{"status":"Failed"}`

> **Cảnh báo:** Việc chuyển đổi thất bại có thể làm gián đoạn quyền đọc và ghi đối với các
> custom resource, bao gồm cả khả năng cập nhật hoặc xóa resource. Nên tránh để việc chuyển
> đổi thất bại bất cứ khi nào có thể, và không nên dùng nó để áp đặt các ràng buộc validation
> (hãy dùng validation schema hoặc webhook admission thay thế).

Ví dụ về một phản hồi từ webhook cho biết request chuyển đổi đã thất bại, kèm một thông điệp
tùy chọn:

#### apiextensions.k8s.io/v1

```yaml
{
  "apiVersion": "apiextensions.k8s.io/v1",
  "kind": "ConversionReview",
  "response": {
    "uid": "<value from request.uid>",
    "result": {
      "status": "Failed",
      "message": "hostPort could not be parsed into a separate host and port"
    }
  }
}
```

#### apiextensions.k8s.io/v1beta1

```yaml
{
  # Đã bị deprecated ở v1.16, thay bằng apiextensions.k8s.io/v1
  "apiVersion": "apiextensions.k8s.io/v1beta1",
  "kind": "ConversionReview",
  "response": {
    "uid": "<value from request.uid>",
    "result": {
      "status": "Failed",
      "message": "hostPort could not be parsed into a separate host and port"
    }
  }
}
```

## Ghi, đọc và cập nhật các object CustomResourceDefinition có nhiều phiên bản (Writing, reading, and updating versioned CustomResourceDefinition objects)

Khi một object được ghi, nó được lưu ở phiên bản đang được chỉ định là storage version tại
thời điểm ghi. Nếu storage version thay đổi, các object hiện có không bao giờ được chuyển đổi
một cách tự động. Tuy nhiên, các object mới tạo hoặc mới được cập nhật thì sẽ được ghi ở
storage version mới. Có khả năng một object đã được ghi ở một phiên bản mà nay không còn được
phục vụ nữa.

Khi bạn đọc một object, bạn chỉ định phiên bản như một phần của đường dẫn. Bạn có thể yêu cầu
một object ở bất kỳ phiên bản nào hiện đang được phục vụ. Nếu bạn chỉ định một phiên bản khác
với phiên bản đã lưu của object, Kubernetes vẫn trả về object cho bạn ở phiên bản bạn yêu cầu,
nhưng object đã lưu trên đĩa thì không bị thay đổi.

Điều gì xảy ra với object được trả về trong lúc phục vụ request đọc phụ thuộc vào những gì
được chỉ định trong `spec.conversion` của CRD:

- nếu giá trị `strategy` mặc định `None` được chỉ định, thay đổi duy nhất đối với object là
  đổi chuỗi `apiVersion` và có thể là [cắt bỏ các trường không xác định
  (pruning unknown fields)](378-custom-resource-definitions-vi.md#field-pruning)
  (tùy theo cấu hình). Lưu ý rằng cách này khó có thể cho kết quả tốt nếu schema khác nhau
  giữa phiên bản lưu trữ và phiên bản được yêu cầu. Cụ thể, bạn không nên dùng chiến lược này
  nếu cùng một dữ liệu được biểu diễn ở các trường khác nhau giữa các phiên bản.
- nếu [chuyển đổi bằng webhook](#webhook-conversion) được chỉ định, thì cơ chế này sẽ điều
  khiển việc chuyển đổi.

Nếu bạn cập nhật một object hiện có, nó sẽ được ghi lại ở phiên bản đang là storage version.
Đây là cách duy nhất để các object chuyển từ phiên bản này sang phiên bản khác.

Để minh họa điều này, hãy xét chuỗi sự kiện giả định sau:

1. Storage version là `v1beta1`. Bạn tạo một object. Nó được lưu ở phiên bản `v1beta1`.
2. Bạn thêm phiên bản `v1` vào CustomResourceDefinition của mình và chỉ định nó làm storage
   version. Ở đây schema của `v1` và `v1beta1` là giống hệt nhau, đây là trường hợp thường gặp
   khi đưa một API lên mức ổn định trong hệ sinh thái Kubernetes.
3. Bạn đọc object của mình ở phiên bản `v1beta1`, rồi đọc lại object đó ở phiên bản `v1`. Cả
   hai object trả về đều giống hệt nhau, ngoại trừ trường apiVersion.
4. Bạn tạo một object mới. Nó được lưu ở phiên bản `v1`. Bây giờ bạn có hai object, một cái ở
   `v1beta1` và cái còn lại ở `v1`.
5. Bạn cập nhật object đầu tiên. Nay nó được lưu ở phiên bản `v1` vì đó là storage version
   hiện tại.

### Các storage version trước đây (Previous storage versions)

API server ghi lại mọi phiên bản từng được đánh dấu là storage version vào trường status
`storedVersions`. Các object có thể đã được lưu ở bất kỳ phiên bản nào từng được chỉ định làm
storage version. Không thể có object nào tồn tại trong bộ lưu trữ ở một phiên bản chưa từng là
storage version.

## Nâng cấp các object hiện có lên storage version mới (Upgrade existing objects to a new stored version) {#upgrade-existing-objects-to-a-new-stored-version}

Khi deprecate các phiên bản và bỏ hỗ trợ chúng, hãy chọn một quy trình nâng cấp bộ lưu trữ.

*Phương án 1:* Dùng Storage Version Migrator

1. Chạy [storage version migrator](https://github.com/kubernetes-sigs/kube-storage-version-migrator)
2. Gỡ phiên bản cũ khỏi trường `status.storedVersions` của CustomResourceDefinition.

*Phương án 2:* Nâng cấp thủ công các object hiện có lên storage version mới

Sau đây là một quy trình ví dụ để nâng cấp từ `v1beta1` lên `v1`.

1. Đặt `v1` làm storage trong file CustomResourceDefinition và áp dụng nó bằng kubectl. Lúc
   này `storedVersions` là `v1beta1, v1`.
2. Viết một quy trình nâng cấp để liệt kê toàn bộ các object hiện có và ghi lại chúng với đúng
   nội dung cũ. Việc này buộc backend ghi các object ở storage version hiện tại, tức là `v1`.
3. Gỡ `v1beta1` khỏi trường `status.storedVersions` của CustomResourceDefinition.

> **Ghi chú:** Dưới đây là ví dụ về cách vá subresource `status` của một object CRD bằng
> `kubectl`:
>
> ```bash
> kubectl patch customresourcedefinitions <CRD_Name> --subresource='status' --type='merge' -p '{"status":{"storedVersions":["v1"]}}'
> ```
