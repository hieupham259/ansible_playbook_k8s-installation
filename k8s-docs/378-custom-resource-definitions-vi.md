# Mở rộng Kubernetes API bằng CustomResourceDefinition (Extend the Kubernetes API with CustomResourceDefinitions)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/>
>
> Trang này trình bày cách cài đặt một custom resource vào Kubernetes API bằng cách tạo một CustomResourceDefinition.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 28 — Mở rộng Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes), bài 4/11 ·
Giai đoạn 28 **không có lab riêng**: kiểm chứng bằng *Checkpoint của giai đoạn 28* — tạo một CRD,
apply một custom resource rồi đọc lại bằng `kubectl get`, thêm validation và chứng minh object sai
bị từ chối — chạy trên cluster ba VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md). Phần lớn nội
dung bài này bạn **đã làm bằng tay rồi** ở [Lab 14 — CRD và Operator](labs/LAB-14-CRD-VA-OPERATOR.md):
B3 tạo CRD và đọc lại custom resource, B4 siết schema và xem trường lạ bị cắt tỉa, B5
`shortNames`/`categories`/`additionalPrinterColumns`, B6 `scope` cùng RBAC cho kind mới, B6.5 bẫy
"xóa namespace không dọn object phạm vi cluster", B7 subresource `status`, B8 vòng lặp điều khiển
thủ công, B9 storage version. Lần đọc này là lần bạn đối chiếu những gì đã làm với tài liệu gốc.

**Đây là tài liệu tham chiếu đầy đủ về CRD, không phải bài đọc một lượt.** Bài dài hơn 2000 dòng và
quá nửa độ dài nằm ở một mục duy nhất — *Quy tắc kiểm tra hợp lệ* — vốn là một ngôn ngữ biểu thức
riêng (CEL) với tám mục con. Mạch chính phải nắm ở lần đọc này gồm bốn mục đầu của phần thực hành:
*Tạo một CustomResourceDefinition*, *Tạo các đối tượng tùy chỉnh*, *Xóa một CustomResourceDefinition*,
*Khai báo structural schema* (kèm *Cắt tỉa trường*); cộng bốn mục nằm rải trong *Các chủ đề nâng cao*:
*Kiểm tra hợp lệ*, *Cột hiển thị bổ sung*, *Subresource status* và *Category*. Mọi mục còn lại là
phần tra khi cần — bảng bên dưới nói rõ tra ở đâu và khi nào.

**Phải hiểu ở lần đọc này:**

- Một CRD đăng ký một `kind` mới và làm API server sinh **một đường dẫn REST mới cho mỗi version**
  khai trong `spec.versions`, dạng `/apis/<group>/<version>/<plural>`; `name` của object CRD bắt
  buộc là `<plural>.<group>`; `spec.scope` quyết định custom object là `Namespaced` hay `Cluster`.
  Bản thân CRD **không thuộc namespace nào**, còn xóa một namespace thì xóa hết custom object
  namespaced trong đó (mục *Tạo một CustomResourceDefinition*). Endpoint mất vài giây mới sẵn sàng
  — theo dõi condition `Established` hoặc thông tin discovery của API server. Chiều ngược lại: xóa
  CRD **gỡ luôn endpoint REST và xóa toàn bộ custom object đang lưu trong đó**, tạo lại đúng CRD ấy
  thì nó bắt đầu ở trạng thái rỗng (mục *Xóa một CustomResourceDefinition*).
- Với `apiextensions.k8s.io/v1`, structural schema là **bắt buộc**: bốn quy tắc ở mục *Khai báo
  structural schema* — có `type` khác rỗng ở node gốc và ở mọi trường/phần tử; trường nào khai
  trong `allOf`/`anyOf`/`oneOf`/`not` phải khai cả ở ngoài; không đặt `description`, `type`,
  `default`, `additionalProperties`, `nullable` bên trong các toán tử đó; `metadata` chỉ được ràng
  buộc `name` và `generateName`. Vi phạm được báo ở condition `NonStructural` của CRD.
- Hai cách API server xử lý dữ liệu sai là **khác nhau**: mục *Cắt tỉa trường* — trường không khai
  báo bị **loại bỏ trước khi lưu**, request vẫn thành công; mục *Kiểm tra hợp lệ* — giá trị vi phạm
  `pattern`, `minimum`, `maximum` làm **cả request bị từ chối**. Vì schema còn được công bố ra
  OpenAPI (mục *Công bố schema kiểm tra hợp lệ trong OpenAPI*), `kubectl` chặn ngay ở phía client —
  đó là lý do ví dụ cắt tỉa trong bài phải thêm `--validate=false`.
- Subresource `status` (mục *Subresource status*) tách hai đường ghi: `PUT` tới `/status` bỏ qua mọi
  thay đổi ngoài `status`, còn `PUT`/`POST`/`PATCH` lên chính resource bỏ qua thay đổi lên `status`;
  `.metadata.generation` tăng theo mọi thay đổi **trừ** thay đổi lên `.metadata` hoặc `.status`.
- `additionalPrinterColumns` (mục *Cột hiển thị bổ sung*), cùng `shortNames` và `categories` (mục
  *Category*) đổi **bề mặt CLI** của kind mới, và chúng là thuộc tính của **server**: kubectl dựa
  vào việc định dạng đầu ra ở phía server, chính API server quyết định `kubectl get` in ra cột nào.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Trước khi bạn bắt đầu* — minikube, các playground online, yêu cầu server ≥ 1.16 | cluster `lab-k8s-master` + `lab-k8s-worker1` + `lab-k8s-worker2` đã vượt yêu cầu này từ lâu và đúng dạng "ít nhất hai node không phải control plane host" mà bài đòi | không cần: dùng cluster của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md), phiên bản khóa ở bảng A1.3 của lab đó |
| Ba ví dụ *không phải structural schema* số 1–3 và cách sửa từng ví dụ | chúng minh họa quy tắc 2 và 3 cho schema có `allOf`/`anyOf`; CRD bạn viết ở [Lab 14](labs/LAB-14-CRD-VA-OPERATOR.md) phần B3–B4 không dùng toán tử logic nào | tra lại chính mục [*Khai báo structural schema*](#specifying-a-structural-schema) của bài này khi CRD báo condition `NonStructural` |
| *Kiểm soát việc cắt tỉa* (`x-kubernetes-preserve-unknown-fields`), *IntOrString*, *RawExtension* | ba lối thoát khỏi cắt tỉa mặc định; chỉ cần khi CRD phải chứa JSON tự do hoặc nhúng nguyên một object Kubernetes | không có bài nào khác trong lộ trình dạy ba phần này; nhánh cắt tỉa mặc định ở [Lab 14](labs/LAB-14-CRD-VA-OPERATOR.md) phần B4 là đủ cho giai đoạn 28, tra lại tại chỗ khi tự viết CRD như vậy |
| *Phục vụ nhiều phiên bản của một CRD* | mục này chỉ có một câu và một link, không trình bày gì | bài [377 — Các phiên bản trong CustomResourceDefinition](377-custom-resource-definition-versioning-vi.md), **bài 5/11 ngay sau bài này** của giai đoạn 28; nền khái niệm ở bài [32 — Các phiên bản lưu trữ](32-storage-version-vi.md) đã đọc ở giai đoạn 1c |
| *Finalizer* | pre-delete hook không phải thứ riêng của CRD — custom object dùng finalizer y hệt object dựng sẵn | bài [29 — Finalizers](29-finalizers-vi.md) đã đọc ở giai đoạn 1c và thực hành ở [Lab 1c](labs/LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md) phần B1 |
| *Validation ratcheting* và danh sách những kiểm tra hợp lệ **không** được ratcheting | cơ chế dành cho tác giả CRD muốn siết schema mà không phá dữ liệu cũ; bạn chưa có CRD nào đang chạy phải nâng cấp | đối chiếu với con đường còn lại — thêm hẳn một version mới — ở bài [377](377-custom-resource-definition-versioning-vi.md) ngay sau bài này |
| *Quy tắc kiểm tra hợp lệ* (CEL) cùng tám mục con `messageExpression`, `message`, `reason`, `fieldPath`, `optionalOldSelf`, *Các hàm kiểm tra hợp lệ*, *Transition rule*, *Tài nguyên tiêu tốn bởi các hàm kiểm tra hợp lệ* | một ngôn ngữ biểu thức riêng, chiếm quá nửa độ dài bài; Checkpoint giai đoạn 28 chỉ đòi "object sai bị từ chối", việc mà `pattern`, `minimum`, `maximum` của mục *Kiểm tra hợp lệ* đã làm được | không có bài nào khác trong lộ trình dạy CEL; quay lại nhóm mục này khi cần ràng buộc liên trường (`self.minReplicas <= self.replicas`) hoặc ràng buộc bất biến giữa hai lần ghi |
| *Giá trị mặc định* và *Defaulting và Nullable* | `default` chỉ có nghĩa khi đã làm chủ structural schema; ba thời điểm áp giá trị mặc định dính tới storage version | phần "khi đọc từ etcd, dùng giá trị mặc định của storage version" thuộc bài [32](32-storage-version-vi.md) đã đọc ở giai đoạn 1c; phần còn lại tra lại khi thêm trường mới vào một CRD đang chạy |
| *Tương thích với OpenAPI V2* | phép chuyển đổi có mất mát, tồn tại để không phá `kubectl` từ 1.13 trở về trước | không có bài nào khác trong lộ trình dùng tới; `kubectl` của cluster lab tiêu thụ tài liệu OpenAPI v3 mà chính mục *Công bố schema kiểm tra hợp lệ trong OpenAPI* khuyến nghị |
| *Priority*, *Type*, *Format* của cột hiển thị bổ sung | chi tiết trình bày: cột nào hiện ở `-o wide`, kiểu và định dạng lúc in | quay lại ba mục con này khi cần cột `-o wide`; [Lab 14](labs/LAB-14-CRD-VA-OPERATOR.md) phần B5 chỉ tạo cột với priority mặc định `0` |
| *Field selector* và *Các trường có thể chọn cho custom resource* (`selectableFields`) | khái niệm field selector không mới; đây chỉ là cách mở thêm trường của kind bạn cho selector | bài [28 — Field selector](28-field-selectors-vi.md) đã đọc ở giai đoạn 1b; `selectableFields` tra lại khi CRD của bạn cần lọc theo trường nằm trong `spec` |
| *Subresource scale* (`specReplicasPath`, `statusReplicasPath`, `labelSelectorPath`) và `kubectl scale` trên custom resource | subresource này chỉ có ích khi đã có thứ đứng sau nó để co giãn hoặc bảo vệ | HPA thực hành ở giai đoạn 11 với bài [342 — Hướng dẫn từng bước về HorizontalPodAutoscaler](342-hpa-walkthrough-vi.md); PodDisruptionBudget ở bài [339](339-configure-pdb-vi.md) |

---

Trang này trình bày cách cài đặt một
[custom resource](179-custom-resources-vi.md)
vào Kubernetes API bằng cách tạo một
[CustomResourceDefinition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#customresourcedefinition-v1-apiextensions-k8s-io).

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Khuyến nghị chạy hướng dẫn này trên một cluster có ít nhất hai node không
đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Phiên bản Kubernetes server của bạn phải bằng hoặc mới hơn 1.16. Để kiểm tra phiên bản, nhập
`kubectl version`.
Nếu bạn đang dùng một phiên bản Kubernetes cũ hơn nhưng vẫn còn được hỗ trợ, hãy chuyển sang
tài liệu của phiên bản đó để xem hướng dẫn phù hợp với cluster của bạn.

## Tạo một CustomResourceDefinition (Create a CustomResourceDefinition)

Khi bạn tạo một CustomResourceDefinition (CRD) mới, Kubernetes API Server sẽ tạo một đường dẫn
tài nguyên RESTful mới cho mỗi phiên bản (version) mà bạn khai báo. Custom resource được tạo ra
từ một đối tượng CRD có thể ở phạm vi namespace (namespaced) hoặc phạm vi cluster
(cluster-scoped), tùy theo khai báo trong trường `spec.scope` của CRD. Cũng giống như các đối
tượng dựng sẵn (built-in) hiện có, việc xóa một namespace sẽ xóa toàn bộ custom object trong
namespace đó. Bản thân các CustomResourceDefinition thì không thuộc namespace nào và có sẵn cho
mọi namespace.

Ví dụ, nếu bạn lưu CustomResourceDefinition sau vào file `resourcedefinition.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name phải khớp với các trường spec bên dưới, và theo dạng: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # tên group dùng cho REST API: /apis/<group>/<version>
  group: stable.example.com
  # danh sách các version mà CustomResourceDefinition này hỗ trợ
  versions:
    - name: v1
      # Mỗi version có thể được bật/tắt bằng cờ Served.
      served: true
      # Một và chỉ một version được đánh dấu là version lưu trữ (storage version).
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # hoặc Namespaced hoặc Cluster
  scope: Namespaced
  names:
    # tên số nhiều dùng trong URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # tên số ít dùng như một bí danh trên CLI và để hiển thị
    singular: crontab
    # kind thường là dạng CamelCase của tên số ít. Các manifest tài nguyên của bạn dùng giá trị này.
    kind: CronTab
    # shortNames cho phép dùng chuỗi ngắn hơn để khớp tài nguyên của bạn trên CLI
    shortNames:
    - ct
```

và tạo nó:

```shell
kubectl apply -f resourcedefinition.yaml
```

Khi đó một endpoint API RESTful mới ở phạm vi namespace sẽ được tạo tại:

```
/apis/stable.example.com/v1/namespaces/*/crontabs/...
```

URL endpoint này sau đó có thể được dùng để tạo và quản lý các custom object.
`kind` của các đối tượng này sẽ là `CronTab`, lấy từ phần spec của đối tượng
CustomResourceDefinition mà bạn vừa tạo ở trên.

Có thể mất vài giây để endpoint được tạo xong.
Bạn có thể theo dõi điều kiện (condition) `Established` của CustomResourceDefinition chuyển sang
true, hoặc theo dõi thông tin discovery của API server cho tới khi tài nguyên của bạn xuất hiện.

## Tạo các đối tượng tùy chỉnh (Create custom objects)

Sau khi đối tượng CustomResourceDefinition đã được tạo, bạn có thể tạo các custom object. Custom
object có thể chứa các trường tùy chỉnh. Các trường này có thể chứa JSON bất kỳ.
Trong ví dụ dưới đây, các trường tùy chỉnh `cronSpec` và `image` được thiết lập trong một custom
object thuộc kind `CronTab`. Kind `CronTab` đến từ phần spec của đối tượng
CustomResourceDefinition mà bạn đã tạo ở trên.

Nếu bạn lưu YAML sau vào `my-crontab.yaml`:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

và tạo nó:

```shell
kubectl apply -f my-crontab.yaml
```

Sau đó bạn có thể quản lý các đối tượng CronTab của mình bằng kubectl. Ví dụ:

```shell
kubectl get crontab
```

Lệnh này sẽ in ra danh sách như sau:

```none
NAME                 AGE
my-new-cron-object   6s
```

Tên tài nguyên không phân biệt hoa thường khi dùng kubectl, và bạn có thể dùng dạng số ít hoặc
số nhiều được định nghĩa trong CRD, cũng như bất kỳ short name nào.

Bạn cũng có thể xem dữ liệu YAML thô:

```shell
kubectl get ct -o yaml
```

Bạn sẽ thấy nó chứa các trường tùy chỉnh `cronSpec` và `image` từ YAML mà bạn đã dùng để tạo:

```yaml
apiVersion: v1
items:
- apiVersion: stable.example.com/v1
  kind: CronTab
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"stable.example.com/v1","kind":"CronTab","metadata":{"annotations":{},"name":"my-new-cron-object","namespace":"default"},"spec":{"cronSpec":"* * * * */5","image":"my-awesome-cron-image"}}
    creationTimestamp: "2021-06-20T07:35:27Z"
    generation: 1
    name: my-new-cron-object
    namespace: default
    resourceVersion: "1326"
    uid: 9aab1d66-628e-41bb-a422-57b8b3b1f5a9
  spec:
    cronSpec: '* * * * */5'
    image: my-awesome-cron-image
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

## Xóa một CustomResourceDefinition (Delete a CustomResourceDefinition)

Khi bạn xóa một CustomResourceDefinition, server sẽ gỡ cài đặt endpoint API RESTful và xóa toàn
bộ custom object đang được lưu trong đó.

```shell
kubectl delete -f resourcedefinition.yaml
kubectl get crontabs
```

```none
Error from server (NotFound): Unable to list {"stable.example.com" "v1" "crontabs"}: the server could not
find the requested resource (get crontabs.stable.example.com)
```

Nếu sau này bạn tạo lại đúng CustomResourceDefinition đó, nó sẽ bắt đầu ở trạng thái rỗng.

## Khai báo structural schema (Specifying a structural schema) {#specifying-a-structural-schema}

CustomResource lưu dữ liệu có cấu trúc trong các trường tùy chỉnh (bên cạnh các trường dựng sẵn
`apiVersion`, `kind` và `metadata`, vốn được API server kiểm tra hợp lệ một cách ngầm định). Với
[kiểm tra hợp lệ theo OpenAPI v3.0](#validation), bạn có thể khai báo một schema; schema này được
kiểm tra khi tạo và khi cập nhật, xem bên dưới để biết chi tiết và các giới hạn của loại schema
này.

Với `apiextensions.k8s.io/v1`, việc định nghĩa một structural schema là **bắt buộc** đối với các
CustomResourceDefinition. Ở phiên bản beta của CustomResourceDefinition, structural schema là tùy
chọn.

Một structural schema là một [schema kiểm tra hợp lệ OpenAPI v3.0](#validation) thỏa mãn:

1. khai báo một `type` khác rỗng (qua `type` trong OpenAPI) cho node gốc, cho mỗi trường được
   khai báo của một node object (qua `properties` hoặc `additionalProperties` trong OpenAPI) và
   cho mỗi phần tử trong một node mảng (qua `items` trong OpenAPI), ngoại trừ:
   * một node có `x-kubernetes-int-or-string: true`
   * một node có `x-kubernetes-preserve-unknown-fields: true`
2. với mỗi trường trong một object và mỗi phần tử trong một mảng được khai báo bên trong bất kỳ
   `allOf`, `anyOf`, `oneOf` hoặc `not` nào, schema cũng phải khai báo trường/phần tử đó ở bên
   ngoài các toán tử logic nói trên (so sánh ví dụ 1 và 2).
3. không đặt `description`, `type`, `default`, `additionalProperties`, `nullable` bên trong
   `allOf`, `anyOf`, `oneOf` hoặc `not`, ngoại trừ hai khuôn mẫu dành cho
   `x-kubernetes-int-or-string: true` (xem bên dưới).
4. nếu `metadata` được khai báo, thì chỉ được phép đặt ràng buộc lên `metadata.name` và
   `metadata.generateName`.

Ví dụ không phải structural schema, số 1:

```yaml
allOf:
- properties:
    foo:
      # ...
```

xung đột với quy tắc 2. Cách viết sau mới đúng:

```yaml
properties:
  foo:
    # ...
allOf:
- properties:
    foo:
      # ...
```

Ví dụ không phải structural schema, số 2:

```yaml
allOf:
- items:
    properties:
      foo:
        # ...
```

xung đột với quy tắc 2. Cách viết sau mới đúng:

```yaml
items:
  properties:
    foo:
      # ...
allOf:
- items:
    properties:
      foo:
        # ...
```

Ví dụ không phải structural schema, số 3:

```yaml
properties:
  foo:
    pattern: "abc"
  metadata:
    type: object
    properties:
      name:
        type: string
        pattern: "^a"
      finalizers:
        type: array
        items:
          type: string
          pattern: "my-finalizer"
anyOf:
- properties:
    bar:
      type: integer
      minimum: 42
  required: ["bar"]
  description: "foo bar object"
```

không phải là structural schema vì các vi phạm sau:

* thiếu `type` ở node gốc (quy tắc 1).
* thiếu `type` của `foo` (quy tắc 1).
* `bar` nằm trong `anyOf` nhưng không được khai báo ở bên ngoài (quy tắc 2).
* `type` của `bar` nằm bên trong `anyOf` (quy tắc 3).
* `description` được đặt bên trong `anyOf` (quy tắc 3).
* `metadata.finalizers` không được phép bị ràng buộc (quy tắc 4).

Ngược lại, schema tương ứng sau đây là structural:

```yaml
type: object
description: "foo bar object"
properties:
  foo:
    type: string
    pattern: "abc"
  bar:
    type: integer
  metadata:
    type: object
    properties:
      name:
        type: string
        pattern: "^a"
anyOf:
- properties:
    bar:
      minimum: 42
  required: ["bar"]
```

Các vi phạm quy tắc structural schema được báo cáo trong condition `NonStructural` của
CustomResourceDefinition.

### Cắt tỉa trường (Field pruning) {#field-pruning}

CustomResourceDefinition lưu dữ liệu tài nguyên đã được kiểm tra hợp lệ vào kho lưu trữ bền vững
của cluster, tức etcd.
Cũng như với các tài nguyên gốc của Kubernetes chẳng hạn ConfigMap, nếu bạn khai báo một trường
mà API server không nhận ra, trường không xác định đó sẽ bị *cắt tỉa* (pruned — loại bỏ) trước
khi được lưu xuống.

Các CRD được chuyển đổi từ `apiextensions.k8s.io/v1beta1` sang `apiextensions.k8s.io/v1` có thể
thiếu structural schema, và `spec.preserveUnknownFields` có thể đang là `true`.

Với các đối tượng CustomResourceDefinition kiểu cũ được tạo dưới dạng
`apiextensions.k8s.io/v1beta1` với `spec.preserveUnknownFields` đặt thành `true`, các điều sau
cũng đúng:

* Việc cắt tỉa không được bật.
* Bạn có thể lưu dữ liệu tùy ý.

Để tương thích với `apiextensions.k8s.io/v1`, hãy cập nhật các custom resource definition của bạn
để:

1. Dùng một OpenAPI schema dạng structural.
2. Đặt `spec.preserveUnknownFields` thành `false`.

Nếu bạn lưu YAML sau vào `my-crontab.yaml`:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
  someRandomField: 42
```

và tạo nó:

```shell
kubectl create --validate=false -f my-crontab.yaml -o yaml
```

Kết quả của bạn sẽ tương tự như sau:

```yaml
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  creationTimestamp: 2017-05-31T12:56:35Z
  generation: 1
  name: my-new-cron-object
  namespace: default
  resourceVersion: "285"
  uid: 9423255b-4600-11e7-af6a-28d2447dc82b
spec:
  cronSpec: '* * * * */5'
  image: my-awesome-cron-image
```

Hãy để ý rằng trường `someRandomField` đã bị cắt tỉa.

Ví dụ này đã tắt việc kiểm tra hợp lệ phía client (bằng cách thêm tùy chọn dòng lệnh
`--validate=false`) nhằm minh họa hành vi của API server.
Bởi vì [các schema kiểm tra hợp lệ OpenAPI cũng được công bố](#publish-validation-schema-in-openapi)
tới client, `kubectl` cũng kiểm tra các trường không xác định và từ chối những đối tượng đó từ
trước khi chúng kịp được gửi tới API server.

#### Kiểm soát việc cắt tỉa (Controlling pruning)

Mặc định, mọi trường không được khai báo của một custom resource, trên tất cả các version, đều bị
cắt tỉa. Tuy nhiên bạn có thể chọn không áp dụng điều đó cho một số nhánh con (sub-tree) trường cụ
thể bằng cách thêm `x-kubernetes-preserve-unknown-fields: true` vào
[structural OpenAPI v3 validation schema](#specifying-a-structural-schema).

Ví dụ:

```yaml
type: object
properties:
  json:
    x-kubernetes-preserve-unknown-fields: true
```

Trường `json` có thể lưu bất kỳ giá trị JSON nào mà không có gì bị cắt tỉa.

Bạn cũng có thể khai báo một phần phần JSON được cho phép; ví dụ:

```yaml
type: object
properties:
  json:
    x-kubernetes-preserve-unknown-fields: true
    type: object
    description: this is arbitrary JSON
```

Với cách này, chỉ các giá trị kiểu `object` mới được cho phép.

Việc cắt tỉa được bật trở lại cho mỗi property được khai báo (hoặc `additionalProperties`):

```yaml
type: object
properties:
  json:
    x-kubernetes-preserve-unknown-fields: true
    type: object
    properties:
      spec:
        type: object
        properties:
          foo:
            type: string
          bar:
            type: string
```

Với cách này, giá trị:

```yaml
json:
  spec:
    foo: abc
    bar: def
    something: x
  status:
    something: x
```

bị cắt tỉa thành:

```yaml
json:
  spec:
    foo: abc
    bar: def
  status:
    something: x
```

Nghĩa là trường `something` bên trong object `spec` đã được khai báo thì bị cắt tỉa, còn mọi thứ
nằm ngoài phạm vi khai báo thì không.

### IntOrString

Các node trong một schema có `x-kubernetes-int-or-string: true` được loại trừ khỏi quy tắc 1, do
đó schema sau là structural:

```yaml
type: object
properties:
  foo:
    x-kubernetes-int-or-string: true
```

Những node này cũng được loại trừ một phần khỏi quy tắc 3, theo nghĩa là hai khuôn mẫu sau được
cho phép (đúng chính xác như vậy, không được đổi thứ tự hay thêm trường):

```yaml
x-kubernetes-int-or-string: true
anyOf:
  - type: integer
  - type: string
...
```

và

```yaml
x-kubernetes-int-or-string: true
allOf:
  - anyOf:
      - type: integer
      - type: string
  - # ... không có hoặc có nhiều mục
...
```

Với một trong hai cách khai báo đó, cả số nguyên lẫn chuỗi đều hợp lệ.

Trong [Công bố schema kiểm tra hợp lệ](#publish-validation-schema-in-openapi),
`x-kubernetes-int-or-string: true` được khai triển thành một trong hai khuôn mẫu nêu trên.

### RawExtension

RawExtension (như [`runtime.RawExtension`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/controller-revision-v1#RawExtension))
chứa các đối tượng Kubernetes hoàn chỉnh, tức là có cả trường `apiVersion` và `kind`.

Bạn có thể khai báo các đối tượng nhúng đó (hoàn toàn không ràng buộc, hoặc khai báo một phần)
bằng cách đặt `x-kubernetes-embedded-resource: true`. Ví dụ:

```yaml
type: object
properties:
  foo:
    x-kubernetes-embedded-resource: true
    x-kubernetes-preserve-unknown-fields: true
```

Ở đây, trường `foo` chứa một đối tượng hoàn chỉnh, ví dụ:

```yaml
foo:
  apiVersion: v1
  kind: Pod
  spec:
    # ...
```

Vì `x-kubernetes-preserve-unknown-fields: true` được khai báo kèm theo nên không có gì bị cắt tỉa.
Tuy vậy, việc dùng `x-kubernetes-preserve-unknown-fields: true` là tùy chọn.

Với `x-kubernetes-embedded-resource: true`, các trường `apiVersion`, `kind` và `metadata` được
khai báo và kiểm tra hợp lệ một cách ngầm định.

## Phục vụ nhiều phiên bản của một CRD (Serving multiple versions of a CRD)

Xem [Đánh phiên bản custom resource definition](377-custom-resource-definition-versioning-vi.md)
để biết thêm thông tin về việc phục vụ nhiều phiên bản của CustomResourceDefinition và di trú
(migrate) các đối tượng của bạn từ phiên bản này sang phiên bản khác.

## Các chủ đề nâng cao (Advanced topics)

### Finalizer

*Finalizer* cho phép controller hiện thực các hook bất đồng bộ chạy trước khi xóa (pre-delete
hook). Custom object hỗ trợ finalizer tương tự như các đối tượng dựng sẵn.

Bạn có thể thêm một finalizer vào custom object như sau:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  finalizers:
  - stable.example.com/finalizer
```

Định danh của finalizer tùy chỉnh gồm một tên miền, một dấu gạch chéo và tên của finalizer. Bất
kỳ controller nào cũng có thể thêm một finalizer vào danh sách finalizer của bất kỳ đối tượng nào.

Yêu cầu xóa đầu tiên trên một đối tượng có finalizer sẽ đặt giá trị cho trường
`metadata.deletionTimestamp` nhưng không xóa đối tượng. Một khi giá trị này đã được đặt, các mục
trong danh sách `finalizers` chỉ có thể bị gỡ bỏ. Chừng nào còn finalizer thì cũng không thể ép
xóa (force delete) đối tượng.

Khi trường `metadata.deletionTimestamp` được đặt, các controller đang theo dõi đối tượng sẽ thực
thi những finalizer mà chúng phụ trách rồi gỡ finalizer đó khỏi danh sách sau khi hoàn tất. Mỗi
controller có trách nhiệm tự gỡ finalizer của mình khỏi danh sách.

Giá trị của `metadata.deletionGracePeriodSeconds` kiểm soát khoảng thời gian giữa các lần cập
nhật polling.

Khi danh sách finalizer trống, nghĩa là mọi finalizer đã được thực thi xong, tài nguyên sẽ được
Kubernetes xóa.

### Kiểm tra hợp lệ (Validation) {#validation}

Custom resource được kiểm tra hợp lệ thông qua
[schema OpenAPI v3.0](https://github.com/OAI/OpenAPI-Specification/blob/3.0.0/versions/3.0.0.md#schema-object),
thông qua x-kubernetes-validations khi [tính năng Validation Rules](#validation-rules) được bật,
và bạn có thể bổ sung thêm kiểm tra hợp lệ bằng
[admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook).

Ngoài ra, các hạn chế sau được áp dụng lên schema:

- Không được đặt các trường sau:

  - `definitions`,
  - `dependencies`,
  - `deprecated`,
  - `discriminator`,
  - `id`,
  - `patternProperties`,
  - `readOnly`,
  - `writeOnly`,
  - `xml`,
  - `$ref`.

- Trường `uniqueItems` không được đặt thành `true`.
- Trường `additionalProperties` không được đặt thành `false`.
- Trường `additionalProperties` loại trừ lẫn nhau với `properties`.

Phần mở rộng `x-kubernetes-validations` có thể được dùng để kiểm tra hợp lệ custom resource bằng
các biểu thức [Common Expression Language (CEL)](https://github.com/google/cel-spec) khi tính năng
[Validation rules](#validation-rules) được bật và schema của CustomResourceDefinition là một
[structural schema](#specifying-a-structural-schema).

Tham khảo mục [structural schema](#specifying-a-structural-schema) để biết các hạn chế và tính
năng khác của CustomResourceDefinition.

Schema được định nghĩa bên trong CustomResourceDefinition. Trong ví dụ dưới đây,
CustomResourceDefinition áp dụng các kiểm tra hợp lệ sau lên custom object:

- `spec.cronSpec` phải là một chuỗi và phải có dạng được mô tả bởi biểu thức chính quy.
- `spec.replicas` phải là số nguyên, có giá trị nhỏ nhất là 1 và lớn nhất là 10.

Lưu CustomResourceDefinition vào `resourcedefinition.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        # openAPIV3Schema là schema dùng để kiểm tra hợp lệ các custom object.
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                  pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

và tạo nó:

```shell
kubectl apply -f resourcedefinition.yaml
```

Một yêu cầu tạo custom object thuộc kind CronTab sẽ bị từ chối nếu các trường của nó chứa giá trị
không hợp lệ. Trong ví dụ dưới đây, custom object chứa các trường có giá trị không hợp lệ:

- `spec.cronSpec` không khớp biểu thức chính quy.
- `spec.replicas` lớn hơn 10.

Nếu bạn lưu YAML sau vào `my-crontab.yaml`:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * *"
  image: my-awesome-cron-image
  replicas: 15
```

và thử tạo nó:

```shell
kubectl apply -f my-crontab.yaml
```

thì bạn sẽ nhận được lỗi:

```console
The CronTab "my-new-cron-object" is invalid: []: Invalid value: map[string]interface {}{"apiVersion":"stable.example.com/v1", "kind":"CronTab", "metadata":map[string]interface {}{"name":"my-new-cron-object", "namespace":"default", "deletionTimestamp":interface {}(nil), "deletionGracePeriodSeconds":(*int64)(nil), "creationTimestamp":"2017-09-05T05:20:07Z", "uid":"e14d79e7-91f9-11e7-a598-f0761cb232d1", "clusterName":""}, "spec":map[string]interface {}{"cronSpec":"* * * *", "image":"my-awesome-cron-image", "replicas":15}}:
validation failure list:
spec.cronSpec in body should match '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
spec.replicas in body should be less than or equal to 10
```

Nếu các trường chứa giá trị hợp lệ thì yêu cầu tạo đối tượng sẽ được chấp nhận.

Lưu YAML sau vào `my-crontab.yaml`:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
  replicas: 5
```

Và tạo nó:

```shell
kubectl apply -f my-crontab.yaml
crontab "my-new-cron-object" created
```

### Validation ratcheting {#validation-ratcheting}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]`

Nếu bạn đang dùng phiên bản Kubernetes cũ hơn v1.30, bạn cần bật tường minh
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`CRDValidationRatcheting` để dùng hành vi này; khi bật, nó áp dụng cho toàn bộ
CustomResourceDefinition trong cluster của bạn.

Với điều kiện bạn đã bật feature gate, Kubernetes hiện thực cơ chế *validation ratcheting* cho
CustomResourceDefinition. API server sẵn sàng chấp nhận các bản cập nhật lên tài nguyên mà sau khi
cập nhật vẫn không hợp lệ, miễn là mọi phần của tài nguyên không vượt qua kiểm tra hợp lệ đều
không bị thay đổi bởi thao tác cập nhật đó. Nói cách khác, bất kỳ phần không hợp lệ nào còn lại
của tài nguyên thì trước đó nó đã sai sẵn rồi.
Bạn không thể dùng cơ chế này để cập nhật một tài nguyên đang hợp lệ thành không hợp lệ.

Tính năng này cho phép tác giả của CRD tự tin thêm các kiểm tra hợp lệ mới vào schema OpenAPIV3
trong một số điều kiện nhất định. Người dùng có thể cập nhật lên schema mới một cách an toàn mà
không cần nâng version của đối tượng hay phá vỡ quy trình làm việc.

Mặc dù hầu hết các kiểm tra hợp lệ đặt trong schema OpenAPIV3 của một CRD đều hỗ trợ ratcheting,
vẫn có vài ngoại lệ. Các kiểm tra hợp lệ OpenAPIV3 sau đây **không** được ratcheting hỗ trợ trong
phần hiện thực ở Kubernetes v1.36, và nếu bị vi phạm thì vẫn tiếp tục báo lỗi như bình thường:

- Các toán tử lượng từ (quantor)
  - `allOf`
  - `oneOf`
  - `anyOf`
  - `not`
  - bất kỳ kiểm tra hợp lệ nào nằm trong nhánh con của một trong các trường trên
- `x-kubernetes-validations`
  Với Kubernetes 1.28, các [validation rule](#validation-rules) của CRD bị ratcheting bỏ qua. Kể
  từ Alpha 2 trong Kubernetes 1.29, `x-kubernetes-validations` chỉ được ratcheting nếu chúng không
  tham chiếu tới `oldSelf`.

  Transition rule thì không bao giờ được ratcheting: chỉ những lỗi phát sinh từ các rule không
  dùng `oldSelf` mới được tự động ratcheting nếu giá trị của chúng không thay đổi.

  Để viết logic ratcheting tùy chỉnh cho biểu thức CEL, xem [optionalOldSelf](#field-optional-oldself).
- `x-kubernetes-list-type`
  Lỗi phát sinh do thay đổi list type của một schema con sẽ không được ratcheting. Ví dụ, thêm
  `set` lên một list đang có phần tử trùng lặp thì luôn luôn dẫn tới lỗi.
- `x-kubernetes-list-map-keys`
  Lỗi phát sinh do thay đổi các khóa map của một list schema sẽ không được ratcheting.
- `required`
  Lỗi phát sinh do thay đổi danh sách các trường bắt buộc sẽ không được ratcheting.
- `properties`
  Việc thêm/xóa/sửa tên của các property không được ratcheting, nhưng những thay đổi về kiểm tra
  hợp lệ bên trong schema và schema con của từng property thì có thể được ratcheting nếu tên
  property giữ nguyên.
- `additionalProperties`
  Việc gỡ bỏ một kiểm tra hợp lệ `additionalProperties` đã khai báo trước đó sẽ không được
  ratcheting.
- `metadata`
  Các lỗi đến từ kiểm tra hợp lệ dựng sẵn của Kubernetes đối với `metadata` của một đối tượng thì
  không được ratcheting (chẳng hạn tên đối tượng, hay các ký tự trong giá trị label). Nếu bạn khai
  báo thêm quy tắc riêng cho metadata của một custom resource thì phần kiểm tra hợp lệ bổ sung đó
  sẽ được ratcheting.

### Quy tắc kiểm tra hợp lệ (Validation rules) {#validation-rules}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.29 [stable]`

Validation rule dùng [Common Expression Language (CEL)](https://github.com/google/cel-spec) để
kiểm tra hợp lệ giá trị của custom resource. Validation rule được đưa vào schema của
CustomResourceDefinition thông qua phần mở rộng `x-kubernetes-validations`.

Rule có phạm vi (scope) là vị trí đặt phần mở rộng `x-kubernetes-validations` trong schema.
Và biến `self` trong biểu thức CEL được gán vào giá trị tại phạm vi đó.

Mọi validation rule đều có phạm vi trong đối tượng hiện tại: không hỗ trợ các rule kiểm tra chéo
đối tượng (cross-object) hay có trạng thái (stateful).

Ví dụ:

```yaml
  # ...
  openAPIV3Schema:
    type: object
    properties:
      spec:
        type: object
        x-kubernetes-validations:
          - rule: "self.minReplicas <= self.replicas"
            message: "replicas should be greater than or equal to minReplicas."
          - rule: "self.replicas <= self.maxReplicas"
            message: "replicas should be smaller than or equal to maxReplicas."
        properties:
          # ...
          minReplicas:
            type: integer
          replicas:
            type: integer
          maxReplicas:
            type: integer
        required:
          - minReplicas
          - replicas
          - maxReplicas
```

sẽ từ chối yêu cầu tạo custom resource này:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  minReplicas: 0
  replicas: 20
  maxReplicas: 10
```

với phản hồi:

```
The CronTab "my-new-cron-object" is invalid:
* spec: Invalid value: map[string]interface {}{"maxReplicas":10, "minReplicas":0, "replicas":20}: replicas should be smaller than or equal to maxReplicas.
```

`x-kubernetes-validations` có thể chứa nhiều rule.
Trường `rule` bên dưới `x-kubernetes-validations` biểu diễn biểu thức sẽ được CEL đánh giá.
Trường `message` biểu diễn thông điệp hiển thị khi kiểm tra hợp lệ thất bại. Nếu không đặt
`message`, phản hồi ở trên sẽ là:

```
The CronTab "my-new-cron-object" is invalid:
* spec: Invalid value: map[string]interface {}{"maxReplicas":10, "minReplicas":0, "replicas":20}: failed rule: self.replicas <= self.maxReplicas
```

> **Ghi chú:** Bạn có thể thử nhanh các biểu thức CEL trong [CEL Playground](https://playcel.undistro.io).

Validation rule được biên dịch khi CRD được tạo/cập nhật.
Yêu cầu tạo/cập nhật CRD sẽ thất bại nếu việc biên dịch validation rule thất bại.
Quá trình biên dịch bao gồm cả kiểm tra kiểu (type checking).

Các lỗi biên dịch:

- `no_matching_overload`: hàm này không có overload nào cho các kiểu của đối số.

  Ví dụ, một rule như `self == true` áp lên một trường kiểu integer sẽ nhận lỗi:

  ```none
  Invalid value: apiextensions.ValidationRule{Rule:"self == true", Message:""}: compilation failed: ERROR: \<input>:1:6: found no matching overload for '_==_' applied to '(int, bool)'
  ```

- `no_such_field`: không chứa trường mong muốn.

  Ví dụ, một rule như `self.nonExistingField > 0` áp lên một trường không tồn tại sẽ trả về lỗi
  sau:

  ```none
  Invalid value: apiextensions.ValidationRule{Rule:"self.nonExistingField > 0", Message:""}: compilation failed: ERROR: \<input>:1:5: undefined field 'nonExistingField'
  ```

- `invalid argument`: đối số không hợp lệ cho macro.

  Ví dụ, một rule như `has(self)` sẽ trả về lỗi:

  ```none
  Invalid value: apiextensions.ValidationRule{Rule:"has(self)", Message:""}: compilation failed: ERROR: <input>:1:4: invalid argument to has() macro
  ```

Ví dụ về Validation Rule:

| Rule                                                                             | Mục đích                                                                           |
| ----------------                                                                 | ------------                                                                      |
| `self.minReplicas <= self.replicas && self.replicas <= self.maxReplicas`         | Kiểm tra ba trường định nghĩa số replica được sắp thứ tự đúng cách        |
| `'Available' in self.stateCounts`                                                | Kiểm tra rằng một mục có khóa 'Available' tồn tại trong map                   |
| `(size(self.list1) == 0) != (size(self.list2) == 0)`                             | Kiểm tra rằng một trong hai list khác rỗng, nhưng không phải cả hai                         |
| <code>!('MY_KEY' in self.map1) &#124;&#124; self['MY_KEY'].matches('^[a-zA-Z]*$')</code>               | Kiểm tra giá trị của map ứng với một khóa cụ thể, nếu khóa đó có trong map               |
| `self.envars.filter(e, e.name == 'MY_ENV').all(e, e.value.matches('^[a-zA-Z]*$')` | Kiểm tra trường 'value' của một mục listMap có trường khóa 'name' bằng 'MY_ENV'  |
| `has(self.expired) && self.created + self.ttl < self.expired`                    | Kiểm tra rằng ngày 'expired' nằm sau ngày 'create' cộng với khoảng thời gian 'ttl'       |
| `self.health.startsWith('ok')`                                                   | Kiểm tra trường chuỗi 'health' có tiền tố là 'ok'                              |
| `self.widgets.exists(w, w.key == 'x' && w.foo < 10)`                             | Kiểm tra property 'foo' của một mục listMap có khóa 'x' nhỏ hơn 10 |
| `type(self) == string ? self == '100%' : self == 1000`                           | Kiểm tra một trường int-or-string cho cả trường hợp int lẫn string             |
| `self.metadata.name.startsWith(self.prefix)`                                     | Kiểm tra tên của đối tượng có tiền tố là giá trị của một trường khác              |
| `self.set1.all(e, !(e in self.set2))`                                            | Kiểm tra hai listSet không giao nhau                                           |
| `size(self.names) == size(self.details) && self.names.all(n, n in self.details)` | Kiểm tra map 'details' được đánh khóa theo các phần tử trong listSet 'names'           |
| `size(self.clusters.filter(c, c.name == self.primary)) == 1`                     | Kiểm tra rằng property 'primary' xuất hiện đúng một lần trong listMap 'clusters'           |

Tham chiếu chéo: [Các phép đánh giá được hỗ trợ trên CEL](https://github.com/google/cel-spec/blob/v0.6.0/doc/langdef.md#evaluation)

- Nếu Rule có phạm vi ở gốc của tài nguyên, nó có thể chọn (select) bất kỳ trường nào được khai
  báo trong schema OpenAPIv3 của CRD, cũng như `apiVersion`, `kind`, `metadata.name` và
  `metadata.generateName`. Điều này bao gồm cả việc chọn các trường trong `spec` lẫn `status`
  trong cùng một biểu thức:

  ```yaml
    # ...
    openAPIV3Schema:
      type: object
      x-kubernetes-validations:
        - rule: "self.status.availableReplicas >= self.spec.minReplicas"
      properties:
          spec:
            type: object
            properties:
              minReplicas:
                type: integer
              # ...
          status:
            type: object
            properties:
              availableReplicas:
                type: integer
  ```

- Nếu Rule có phạm vi là một object có các property, thì các property truy cập được của object đó
  có thể được chọn qua `self.field` và sự hiện diện của trường có thể được kiểm tra qua
  `has(self.field)`. Các trường có giá trị null được coi như trường vắng mặt trong biểu thức CEL.

  ```yaml
    # ...
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          x-kubernetes-validations:
            - rule: "has(self.foo)"
          properties:
            # ...
            foo:
              type: integer
  ```

- Nếu Rule có phạm vi là một object có additionalProperties (tức là một map) thì các giá trị của
  map truy cập được qua `self[mapKey]`, việc kiểm tra map có chứa khóa nào đó thực hiện qua
  `mapKey in self`, và toàn bộ các mục của map truy cập được qua các macro và hàm CEL chẳng hạn
  `self.all(...)`.

  ```yaml
    # ...
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          x-kubernetes-validations:
            - rule: "self['xyz'].foo > 0"
          additionalProperties:
            # ...
            type: object
            properties:
              foo:
                type: integer
  ```

- Nếu Rule có phạm vi là một mảng, các phần tử của mảng truy cập được qua `self[i]` và cũng qua
  các macro và hàm.

  ```yaml
    # ...
    openAPIV3Schema:
      type: object
      properties:
        # ...
        foo:
          type: array
          x-kubernetes-validations:
            - rule: "size(self) == 1"
          items:
            type: string
  ```

- Nếu Rule có phạm vi là một giá trị vô hướng (scalar), `self` được gán vào chính giá trị vô hướng
  đó.

  ```yaml
    # ...
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          properties:
            # ...
            foo:
              type: integer
              x-kubernetes-validations:
              - rule: "self > 0"
  ```

Ví dụ:

|kiểu của trường mà rule có phạm vi tại đó    | Ví dụ rule             |
| -----------------------| -----------------------|
| object gốc            | `self.status.actual <= self.spec.maxDesired` |
| map của các object         | `self.components['Widget'].priority < 10` |
| list các số nguyên       | `self.values.all(value, value >= 0 && value < 100)` |
| chuỗi                 | `self.startsWith('kube')` |

Các trường `apiVersion`, `kind`, `metadata.name` và `metadata.generateName` luôn truy cập được từ
gốc của đối tượng và từ bất kỳ object nào được đánh dấu `x-kubernetes-embedded-resource`. Không có
property metadata nào khác truy cập được.

Dữ liệu không xác định được giữ lại trong custom resource nhờ
`x-kubernetes-preserve-unknown-fields` thì không truy cập được trong biểu thức CEL. Điều này bao
gồm:

- Giá trị của các trường không xác định được giữ lại bởi các object schema có
  `x-kubernetes-preserve-unknown-fields`.
- Các property của object mà schema của property đó thuộc "kiểu không xác định" (unknown type).
  "Kiểu không xác định" được định nghĩa đệ quy như sau:

  - Một schema không có `type` và có `x-kubernetes-preserve-unknown-fields` đặt thành true
  - Một mảng mà schema của items thuộc "kiểu không xác định"
  - Một object mà schema của additionalProperties thuộc "kiểu không xác định"

Chỉ những tên property có dạng `[a-zA-Z_.-/][a-zA-Z0-9_.-/]*` mới truy cập được.
Các tên property truy cập được sẽ được escape theo các quy tắc sau khi truy cập trong biểu thức:

| chuỗi escape         | tên property tương đương  |
| ----------------------- | -----------------------|
| `__underscores__`       | `__`                  |
| `__dot__`               | `.`                   |
|`__dash__`               | `-`                   |
| `__slash__`             | `/`                   |
| `__{keyword}__`         | [từ khóa RESERVED của CEL](https://github.com/google/cel-spec/blob/v0.6.0/doc/langdef.md#syntax)       |

Lưu ý: từ khóa RESERVED của CEL phải khớp chính xác với tên property thì mới được escape (ví dụ
`int` trong từ `sprint` sẽ không được escape).

Ví dụ về escape:

|tên property    | rule với tên property đã escape     |
| ----------------| -----------------------             |
| namespace       | `self.__namespace__ > 0`            |
| x-prop          | `self.x__dash__prop > 0`            |
| redact__d       | `self.redact__underscores__d > 0`   |
| string          | `self.startsWith('kube')`           |

Phép so sánh bằng trên các mảng có `x-kubernetes-list-type` là `set` hoặc `map` bỏ qua thứ tự phần
tử, tức là `[1, 2] == [2, 1]`. Phép nối (concatenation) trên các mảng có x-kubernetes-list-type
tuân theo ngữ nghĩa của list type:

- `set`: `X + Y` thực hiện phép hợp, trong đó vị trí trong mảng của mọi phần tử thuộc `X` được giữ
  nguyên và các phần tử không giao nhau của `Y` được nối vào cuối, giữ nguyên thứ tự bộ phận của
  chúng.

- `map`: `X + Y` thực hiện phép trộn (merge), trong đó vị trí trong mảng của mọi khóa thuộc `X`
  được giữ nguyên nhưng giá trị bị ghi đè bởi giá trị trong `Y` khi tập khóa của `X` và `Y` giao
  nhau. Các phần tử trong `Y` có khóa không giao nhau được nối vào cuối, giữ nguyên thứ tự bộ phận
  của chúng.

Dưới đây là ánh xạ kiểu khai báo giữa OpenAPIv3 và kiểu CEL:

| Kiểu OpenAPIv3                                     | Kiểu CEL                                                                                                                     |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 'object' có Properties                           | object / "message type"                                                                                                      |
| 'object' có AdditionalProperties                 | map                                                                                                                          |
| 'object' có x-kubernetes-embedded-type           | object / "message type", các trường 'apiVersion', 'kind', 'metadata.name' và 'metadata.generateName' được đưa vào schema một cách ngầm định |
| 'object' có x-kubernetes-preserve-unknown-fields | object / "message type", các trường không xác định KHÔNG truy cập được trong biểu thức CEL                                 |
| x-kubernetes-int-or-string                         | object động, hoặc là int hoặc là string, có thể dùng `type(value)` để kiểm tra kiểu                |
| 'array                                             | list                                                                                                                         |
| 'array' có x-kubernetes-list-type=map            | list với phép so sánh bằng dựa trên map và đảm bảo khóa duy nhất                                                                         |
| 'array' có x-kubernetes-list-type=set            | list với phép so sánh bằng dựa trên set và đảm bảo phần tử duy nhất                                                                         |
| 'boolean'                                          | boolean                                                                                                                      |
| 'number' (mọi format)                             | double                                                                                                                       |
| 'integer' (mọi format)                            | int (64)                                                                                                                     |
| 'null'                                             | null_type                                                                                                                    |
| 'string'                                           | string                                                                                                                       |
| 'string' với format=byte (mã hóa base64)         | bytes                                                                                                                        |
| 'string' với format=date                          | timestamp (google.protobuf.Timestamp)                                                                                        |
| 'string' với format=datetime                      | timestamp (google.protobuf.Timestamp)                                                                                        |
| 'string' với format=duration                      | duration (google.protobuf.Duration)                                                                                          |

Tham chiếu chéo: [Các kiểu của CEL](https://github.com/google/cel-spec/blob/v0.6.0/doc/langdef.md#values),
[Các kiểu OpenAPI](https://swagger.io/specification/#data-types),
[Structural Schema của Kubernetes](#specifying-a-structural-schema).

#### Trường messageExpression (The messageExpression field)

Tương tự trường `message` — trường định nghĩa chuỗi được báo cáo khi một validation rule thất bại
— `messageExpression` cho phép bạn dùng một biểu thức CEL để tạo ra chuỗi thông điệp.
Nhờ đó bạn có thể đưa thêm thông tin mô tả chi tiết vào thông điệp báo lỗi kiểm tra hợp lệ.
`messageExpression` phải đánh giá ra một chuỗi và có thể dùng cùng các biến khả dụng cho trường
`rule`. Ví dụ:

```yaml
x-kubernetes-validations:
- rule: "self.x <= self.maxLimit"
  messageExpression: '"x exceeded max limit of " + string(self.maxLimit)'
```

Hãy nhớ rằng phép nối chuỗi trong CEL (toán tử `+`) không tự động ép kiểu về chuỗi. Nếu bạn có một
giá trị vô hướng không phải chuỗi, hãy dùng hàm `string(<value>)` để ép giá trị đó về chuỗi như
trong ví dụ trên.

`messageExpression` phải đánh giá ra một chuỗi, và điều này được kiểm tra ngay khi CRD được ghi.
Lưu ý rằng bạn có thể đặt cả `message` lẫn `messageExpression` trên cùng một rule; nếu cả hai cùng
có mặt thì `messageExpression` sẽ được dùng. Tuy nhiên, nếu `messageExpression` đánh giá ra lỗi
thì chuỗi định nghĩa trong `message` sẽ được dùng thay thế, và lỗi của `messageExpression` sẽ được
ghi log. Cơ chế dự phòng này cũng xảy ra nếu biểu thức CEL định nghĩa trong `messageExpression`
sinh ra một chuỗi rỗng, hoặc một chuỗi có chứa ký tự xuống dòng.

Nếu một trong các điều kiện trên xảy ra mà `message` lại chưa được đặt, thì thông điệp lỗi kiểm
tra hợp lệ mặc định sẽ được dùng thay thế.

`messageExpression` là một biểu thức CEL, nên các hạn chế được liệt kê trong
[Tài nguyên tiêu tốn bởi các hàm kiểm tra hợp lệ](#resource-use-by-validation-functions) cũng được
áp dụng. Nếu quá trình đánh giá bị dừng do ràng buộc tài nguyên trong khi thực thi
`messageExpression`, thì sẽ không có validation rule nào được thực thi tiếp.

Việc đặt `messageExpression` là tùy chọn.

#### Trường `message` (The `message` field) {#field-message}

Nếu bạn muốn đặt một thông điệp tĩnh, bạn có thể cung cấp `message` thay vì `messageExpression`.
Giá trị của `message` được dùng làm chuỗi lỗi mờ (opaque) khi kiểm tra hợp lệ thất bại.

Việc đặt `message` là tùy chọn.

#### Trường `reason` (The `reason` field) {#field-reason}

Bạn có thể thêm một lý do thất bại kiểm tra hợp lệ ở dạng máy đọc được vào trong một `validation`,
lý do này sẽ được trả về mỗi khi một yêu cầu không vượt qua validation rule đó.

Ví dụ:

```yaml
x-kubernetes-validations:
- rule: "self.x <= self.maxLimit"
  reason: "FieldValueInvalid"
```

Mã trạng thái HTTP trả về cho bên gọi sẽ khớp với `reason` của validation rule thất bại đầu tiên.
Các lý do hiện được hỗ trợ là: "FieldValueInvalid", "FieldValueForbidden", "FieldValueRequired",
"FieldValueDuplicate". Nếu không đặt hoặc đặt lý do không xác định thì mặc định dùng
"FieldValueInvalid".

Việc đặt `reason` là tùy chọn.

#### Trường `fieldPath` (The `fieldPath` field) {#field-field-path}

Bạn có thể chỉ định đường dẫn trường (field path) được trả về khi kiểm tra hợp lệ thất bại.

Ví dụ:

```yaml
x-kubernetes-validations:
- rule: "self.foo.test.x <= self.maxLimit"
  fieldPath: ".foo.test.x"
```

Trong ví dụ trên, phần kiểm tra hợp lệ kiểm tra rằng giá trị của trường `x` phải nhỏ hơn giá trị
của `maxLimit`. Nếu không chỉ định `fieldPath`, khi kiểm tra hợp lệ thất bại thì fieldPath mặc
định là vị trí mà `self` đang có phạm vi. Khi có chỉ định `fieldPath`, lỗi trả về sẽ có
`fieldPath` trỏ đúng tới vị trí của trường `x`.

Giá trị `fieldPath` phải là một đường dẫn JSON tương đối, lấy phạm vi là vị trí của phần mở rộng
x-kubernetes-validations này trong schema.
Ngoài ra, nó phải trỏ tới một trường đang tồn tại trong schema.
Ví dụ, khi phần kiểm tra hợp lệ kiểm tra một thuộc tính cụ thể `foo` nằm dưới một map `testMap`,
bạn có thể đặt `fieldPath` thành `".testMap.foo"` hoặc `.testMap['foo']'`.
Nếu phần kiểm tra hợp lệ cần kiểm tra thuộc tính duy nhất trong hai list, thì fieldPath có thể đặt
thành một trong hai list đó. Ví dụ, có thể đặt thành `.testList1` hoặc `.testList2`.
Hiện tại nó hỗ trợ phép truy cập con (child operation) để trỏ tới một trường đang tồn tại.
Xem [Hỗ trợ JSONPath trong Kubernetes](https://kubernetes.io/docs/reference/kubectl/jsonpath/) để
biết thêm thông tin.
Trường `fieldPath` không hỗ trợ đánh chỉ số mảng bằng số.

Việc đặt `fieldPath` là tùy chọn.

#### Trường `optionalOldSelf` (The `optionalOldSelf` field) {#field-optional-oldself}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [stable]`

Nếu cluster của bạn không bật [CRD validation ratcheting](#validation-ratcheting), thì API
CustomResourceDefinition sẽ không có trường này, và việc cố đặt nó có thể dẫn tới lỗi.

Trường `optionalOldSelf` là một trường kiểu boolean làm thay đổi hành vi của
[Transition Rule](#transition-rules) được mô tả bên dưới. Thông thường, một transition rule sẽ
không được đánh giá nếu không xác định được `oldSelf`: khi đang tạo đối tượng hoặc khi một giá trị
mới được đưa vào trong một lần cập nhật.

Nếu `optionalOldSelf` được đặt thành true, thì transition rule sẽ luôn được đánh giá và kiểu của
`oldSelf` được đổi thành kiểu [`Optional`](https://pkg.go.dev/github.com/google/cel-go/cel#OptionalTypes)
của CEL.

`optionalOldSelf` hữu ích trong các trường hợp tác giả schema muốn có một công cụ kiểm soát tinh
tế hơn [so với hành vi mặc định dựa trên phép so sánh bằng](#validation-ratcheting), nhằm đưa vào
các ràng buộc mới, thường là chặt hơn, cho các giá trị mới, trong khi vẫn cho phép các giá trị cũ
được "miễn trừ" (grandfathered) hoặc được ratcheting theo phần kiểm tra hợp lệ cũ.

Ví dụ sử dụng:

| CEL                                     | Mô tả |
|-----------------------------------------|-------------|
| <code>self.foo == "foo" &#124;&#124; (oldSelf.hasValue() && oldSelf.value().foo != "foo")</code> | Rule có ratcheting. Một khi giá trị đã được đặt thành "foo" thì nó phải giữ nguyên là foo. Nhưng nếu nó đã tồn tại từ trước khi ràng buộc "foo" được đưa vào thì nó có thể mang giá trị bất kỳ |
| <code>[oldSelf.orValue(""), self].all(x, ["OldCase1", "OldCase2"].exists(case, x == case)) &#124;&#124; ["NewCase1", "NewCase2"].exists(case, self == case) &#124;&#124; ["NewCase"].has(self)</code> | "Kiểm tra hợp lệ có ratcheting cho các giá trị enum đã bị gỡ bỏ, nếu oldSelf từng dùng chúng" |
| <code>oldSelf.optMap(o, o.size()).orValue(0) < 4 &#124;&#124; self.size() >= 4</code> | Kiểm tra hợp lệ có ratcheting cho kích thước tối thiểu của map hoặc list vừa được tăng lên |

#### Các hàm kiểm tra hợp lệ (Validation functions) {#available-validation-functions}

Các hàm khả dụng bao gồm:

- Các hàm chuẩn của CEL, được định nghĩa trong [danh sách định nghĩa chuẩn](https://github.com/google/cel-spec/blob/v0.7.0/doc/langdef.md#list-of-standard-definitions)
- Các [macro](https://github.com/google/cel-spec/blob/v0.7.0/doc/langdef.md#macros) chuẩn của CEL
- [Thư viện hàm chuỗi mở rộng](https://pkg.go.dev/github.com/google/cel-go@v0.11.2/ext#Strings) của CEL
- [Thư viện mở rộng CEL](https://pkg.go.dev/k8s.io/apiextensions-apiserver@v0.24.0/pkg/apiserver/schema/cel/library#pkg-functions) của Kubernetes

#### Transition rule (Transition rules) {#transition-rules}

Một rule chứa biểu thức tham chiếu tới định danh `oldSelf` thì được ngầm định coi là một
*transition rule*. Transition rule cho phép tác giả schema ngăn một số phép chuyển đổi nhất định
giữa hai trạng thái mà bản thân chúng đều hợp lệ. Ví dụ:

```yaml
type: string
enum: ["low", "medium", "high"]
x-kubernetes-validations:
- rule: "!(self == 'high' && oldSelf == 'low') && !(self == 'low' && oldSelf == 'high')"
  message: cannot transition directly between 'low' and 'high'
```

Khác với các rule khác, transition rule chỉ áp dụng cho những thao tác thỏa mãn các tiêu chí sau:

- Thao tác đó cập nhật một đối tượng đã tồn tại. Transition rule không bao giờ áp dụng cho thao
  tác tạo mới.

- Cả giá trị cũ lẫn giá trị mới đều tồn tại. Bạn vẫn có thể kiểm tra xem một giá trị đã được thêm
  vào hay bị gỡ bỏ hay chưa bằng cách đặt transition rule lên node cha. Transition rule không bao
  giờ được áp dụng khi tạo custom resource. Khi được đặt lên một trường tùy chọn, transition rule
  sẽ không áp dụng cho các thao tác cập nhật đặt giá trị hoặc bỏ giá trị của trường đó.

- Đường dẫn tới node schema đang được transition rule kiểm tra phải phân giải tới một node có thể
  so sánh tương ứng (correlate) được giữa đối tượng cũ và đối tượng mới. Ví dụ, các phần tử list
  và các node con của chúng (`spec.foo[10].bar`) không nhất thiết tương ứng được giữa một đối
  tượng đang tồn tại và một lần cập nhật sau đó lên chính đối tượng ấy.

Lỗi sẽ được sinh ra khi ghi CRD nếu một node schema chứa transition rule không bao giờ có thể được
áp dụng, ví dụ "oldSelf cannot be used on the uncorrelatable portion of the schema within *path*".

Transition rule chỉ được phép đặt trên các *phần tương ứng được* (correlatable portion) của
schema. Một phần của schema là tương ứng được nếu mọi schema cha kiểu `array` đều có
`x-kubernetes-list-type=map`; bất kỳ schema mảng cha nào kiểu `set` hoặc `atomic` đều khiến việc
tương ứng `self` với `oldSelf` trở nên không rõ ràng.

Dưới đây là một số ví dụ về transition rule:

*Bảng: Ví dụ về transition rule*

| Trường hợp sử dụng                                                 | Rule
| --------                                                          | --------
| Bất biến (immutability)                                           | `self.foo == oldSelf.foo`
| Ngăn sửa đổi/gỡ bỏ sau khi đã gán                                 | `oldSelf != 'bar' \|\| self == 'bar'` hoặc `!has(oldSelf.field) \|\| has(self.field)`
| Tập hợp chỉ thêm (append-only set)                                | `self.all(element, element in oldSelf)`
| Nếu giá trị trước là X thì giá trị mới chỉ có thể là A hoặc B, không phải Y hay Z | `oldSelf != 'X' \|\| self in ['A', 'B']`
| Bộ đếm đơn điệu (không giảm)                                      | `self >= oldSelf`

#### Tài nguyên tiêu tốn bởi các hàm kiểm tra hợp lệ (Resource use by validation functions) {#resource-use-by-validation-functions}

Khi bạn tạo hoặc cập nhật một CustomResourceDefinition có dùng validation rule, API server sẽ kiểm
tra tác động dự kiến của việc chạy các validation rule đó. Nếu một rule được ước lượng là quá tốn
kém để thực thi, API server sẽ từ chối thao tác tạo hoặc cập nhật và trả về một thông điệp lỗi.
Một cơ chế tương tự cũng được dùng tại thời điểm chạy để quan sát các hành động mà trình thông
dịch thực hiện. Nếu trình thông dịch thực thi quá nhiều chỉ thị, việc thực thi rule sẽ bị dừng và
dẫn tới lỗi.
Mỗi CustomResourceDefinition cũng chỉ được cấp một lượng tài nguyên nhất định để chạy xong toàn bộ
validation rule của nó. Nếu tổng chi phí ước lượng của tất cả các rule tại thời điểm tạo vượt quá
giới hạn đó thì cũng sẽ xảy ra lỗi kiểm tra hợp lệ.

Bạn khó gặp vấn đề với ngân sách tài nguyên dành cho kiểm tra hợp lệ nếu bạn chỉ khai báo những
rule luôn tốn cùng một lượng thời gian bất kể đầu vào lớn tới đâu.
Ví dụ, một rule khẳng định rằng `self.foo == 1` thì bản thân nó không có rủi ro bị từ chối vì lý
do ngân sách tài nguyên kiểm tra hợp lệ.
Nhưng nếu `foo` là một chuỗi và bạn định nghĩa validation rule `self.foo.contains("someString")`,
thì rule đó mất nhiều thời gian thực thi hơn tùy theo độ dài của `foo`.
Một ví dụ khác là nếu `foo` là một mảng và bạn khai báo validation rule `self.foo.all(x, x > 5)`.
Hệ thống tính chi phí luôn giả định trường hợp xấu nhất nếu không có giới hạn về độ dài của `foo`,
và điều này xảy ra với bất cứ thứ gì có thể duyệt qua (list, map, v.v.).

Vì vậy, thực hành tốt nhất là đặt giới hạn qua `maxItems`, `maxProperties` và `maxLength` cho bất
cứ thứ gì sẽ được xử lý trong một validation rule, để tránh lỗi kiểm tra hợp lệ trong quá trình
ước lượng chi phí. Ví dụ, cho schema sau với một rule:

```yaml
openAPIV3Schema:
  type: object
  properties:
    foo:
      type: array
      items:
        type: string
      x-kubernetes-validations:
        - rule: "self.all(x, x.contains('a string'))"
```

thì API server sẽ từ chối rule này vì lý do ngân sách kiểm tra hợp lệ, với lỗi:

```
spec.validation.openAPIV3Schema.properties[spec].properties[foo].x-kubernetes-validations[0].rule: Forbidden:
CEL rule exceeded budget by more than 100x (try simplifying the rule, or adding maxItems, maxProperties, and
maxLength where arrays, maps, and strings are used)
```

Việc từ chối xảy ra vì `self.all` kéo theo việc gọi `contains()` trên mọi chuỗi trong `foo`, và
mỗi lần gọi lại kiểm tra chuỗi đó xem có chứa `'a string'` hay không. Nếu không có giới hạn thì
đây là một rule rất tốn kém.

Nếu bạn không khai báo bất kỳ giới hạn kiểm tra hợp lệ nào, chi phí ước lượng của rule này sẽ vượt
quá giới hạn chi phí cho mỗi rule. Nhưng nếu bạn thêm giới hạn vào đúng chỗ thì rule sẽ được chấp
nhận:

```yaml
openAPIV3Schema:
  type: object
  properties:
    foo:
      type: array
      maxItems: 25
      items:
        type: string
        maxLength: 10
      x-kubernetes-validations:
        - rule: "self.all(x, x.contains('a string'))"
```

Hệ thống ước lượng chi phí có tính đến số lần rule sẽ được thực thi, bên cạnh chi phí ước lượng
của bản thân rule đó. Chẳng hạn, rule sau đây sẽ có chi phí ước lượng bằng với ví dụ trước (mặc dù
rule giờ được định nghĩa trên từng phần tử của mảng):

```yaml
openAPIV3Schema:
  type: object
  properties:
    foo:
      type: array
      maxItems: 25
      items:
        type: string
        x-kubernetes-validations:
          - rule: "self.contains('a string'))"
        maxLength: 10
```

Nếu một list nằm bên trong một list có validation rule dùng `self.all`, thì chi phí sẽ cao hơn
đáng kể so với một list không lồng nhau với cùng rule đó. Một rule vốn được chấp nhận trên list
không lồng nhau có thể cần đặt giới hạn thấp hơn trên cả hai list lồng nhau thì mới được chấp
nhận. Ví dụ, ngay cả khi không đặt giới hạn nào, rule sau vẫn được chấp nhận:

```yaml
openAPIV3Schema:
  type: object
  properties:
    foo:
      type: array
      items:
        type: integer
    x-kubernetes-validations:
      - rule: "self.all(x, x == 5)"
```

Nhưng cũng rule đó trên schema sau (có thêm một mảng lồng nhau) lại sinh ra lỗi kiểm tra hợp lệ:

```yaml
openAPIV3Schema:
  type: object
  properties:
    foo:
      type: array
      items:
        type: array
        items:
          type: integer
        x-kubernetes-validations:
          - rule: "self.all(x, x == 5)"
```

Lý do là vì mỗi phần tử của `foo` bản thân nó là một mảng, và mỗi mảng con lại gọi `self.all`.
Hãy tránh dùng list và map lồng nhau ở những nơi có validation rule, nếu có thể.

### Giá trị mặc định (Defaulting)

> **Ghi chú:** Để dùng cơ chế đặt giá trị mặc định, CustomResourceDefinition của bạn phải dùng
> phiên bản API `apiextensions.k8s.io/v1`.

Cơ chế đặt giá trị mặc định cho phép khai báo giá trị mặc định trong
[schema kiểm tra hợp lệ OpenAPI v3](#validation):

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        # openAPIV3Schema là schema dùng để kiểm tra hợp lệ các custom object.
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                  pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
                  default: "5 0 * * *"
                image:
                  type: string
                replicas:
                  type: integer
                  minimum: 1
                  maximum: 10
                  default: 1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

Với cấu hình này, cả `cronSpec` lẫn `replicas` đều được gán giá trị mặc định:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  image: my-awesome-cron-image
```

dẫn tới

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "5 0 * * *"
  image: my-awesome-cron-image
  replicas: 1
```

Việc gán giá trị mặc định diễn ra trên đối tượng:

* trong yêu cầu gửi tới API server, dùng giá trị mặc định của version trong yêu cầu,
* khi đọc từ etcd, dùng giá trị mặc định của storage version,
* sau các plugin mutating admission có patch khác rỗng, dùng giá trị mặc định của version đối
  tượng trong admission webhook.

Các giá trị mặc định được áp dụng khi đọc dữ liệu từ etcd sẽ không tự động được ghi ngược trở lại
etcd. Cần một yêu cầu cập nhật qua API để lưu bền các giá trị mặc định đó vào etcd.

Giá trị mặc định cho các trường không phải node lá phải được cắt tỉa (ngoại trừ giá trị mặc định
cho các trường `metadata`) và phải hợp lệ theo schema đã cung cấp. Ví dụ, trong ví dụ trên, một
giá trị mặc định `{"replicas": "foo", "badger": 1}` cho trường `spec` sẽ không hợp lệ, vì `badger`
là một trường không xác định và `replicas` không phải là chuỗi.

Giá trị mặc định cho các trường `metadata` của những node có `x-kubernetes-embedded-resources: true`
(hoặc các phần của một giá trị mặc định bao trùm `metadata`) thì không bị cắt tỉa trong lúc tạo
CustomResourceDefinition, mà bị cắt tỉa ở bước pruning trong quá trình xử lý yêu cầu.

#### Defaulting và Nullable (Defaulting and Nullable)

Giá trị null của những trường không khai báo cờ nullable, hoặc khai báo cờ đó với giá trị `false`,
sẽ bị cắt tỉa trước khi việc gán giá trị mặc định diễn ra. Nếu có giá trị mặc định thì nó sẽ được
áp dụng. Khi nullable là `true`, giá trị null sẽ được giữ lại và không được gán giá trị mặc định.

Ví dụ, cho schema OpenAPI dưới đây:

```yaml
type: object
properties:
  spec:
    type: object
    properties:
      foo:
        type: string
        nullable: false
        default: "default"
      bar:
        type: string
        nullable: true
      baz:
        type: string
```

việc tạo một đối tượng với giá trị null cho `foo`, `bar` và `baz`

```yaml
spec:
  foo: null
  bar: null
  baz: null
```

dẫn tới

```yaml
spec:
  foo: "default"
  bar: null
```

trong đó `foo` bị cắt tỉa rồi được gán giá trị mặc định vì trường này không nullable, `bar` giữ
nguyên giá trị null nhờ `nullable: true`, còn `baz` bị cắt tỉa vì trường này không nullable và
không có giá trị mặc định.

### Công bố schema kiểm tra hợp lệ trong OpenAPI (Publish Validation Schema in OpenAPI) {#publish-validation-schema-in-openapi}

Các [schema kiểm tra hợp lệ OpenAPI v3](#validation) của CustomResourceDefinition mà thỏa mãn tính
[structural](#specifying-a-structural-schema) và [bật cắt tỉa](#field-pruning) sẽ được Kubernetes
API server công bố dưới dạng
[OpenAPI v3](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#openapi-and-swagger-definitions)
và OpenAPI v2. Khuyến nghị dùng tài liệu OpenAPI v3 vì nó là biểu diễn không mất mát của schema
kiểm tra hợp lệ OpenAPI v3 trong CustomResourceDefinition, trong khi OpenAPI v2 là một phép chuyển
đổi có mất mát.

Công cụ dòng lệnh [kubectl](https://kubernetes.io/docs/reference/kubectl/) tiêu thụ schema đã công
bố để thực hiện kiểm tra hợp lệ phía client (`kubectl create` và `kubectl apply`), giải thích
schema (`kubectl explain`) trên các custom resource. Schema đã công bố cũng có thể được dùng cho
các mục đích khác, chẳng hạn sinh client hoặc sinh tài liệu.

#### Tương thích với OpenAPI V2 (Compatibility with OpenAPI V2)

Để tương thích với OpenAPI V2, schema kiểm tra hợp lệ OpenAPI v3 được chuyển đổi có mất mát sang
schema OpenAPI v2. Schema này xuất hiện trong các trường `definitions` và `paths` của
[đặc tả OpenAPI v2](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#openapi-and-swagger-definitions).

Các sửa đổi sau được áp dụng trong quá trình chuyển đổi nhằm giữ tương thích ngược với kubectl ở
phiên bản 1.13 trước đây. Những sửa đổi này ngăn kubectl trở nên quá nghiêm ngặt và từ chối các
schema OpenAPI hợp lệ mà nó không hiểu. Việc chuyển đổi không làm thay đổi schema kiểm tra hợp lệ
được định nghĩa trong CRD, và do đó không ảnh hưởng tới việc [kiểm tra hợp lệ](#validation) ở API
server.

1. Các trường sau bị loại bỏ vì OpenAPI v2 không hỗ trợ chúng.

   - Các trường `allOf`, `anyOf`, `oneOf` và `not` bị loại bỏ

2. Nếu `nullable: true` được đặt, chúng ta bỏ đi `type`, `nullable`, `items` và `properties` vì
   OpenAPI v2 không diễn đạt được tính nullable. Điều này là cần thiết để kubectl không từ chối
   các đối tượng vốn hợp lệ.

### Cột hiển thị bổ sung (Additional printer columns)

Công cụ kubectl dựa vào việc định dạng đầu ra ở phía server. API server của cluster quyết định
những cột nào được hiển thị bởi lệnh `kubectl get`. Bạn có thể tùy biến các cột này cho một
CustomResourceDefinition. Ví dụ dưới đây thêm các cột `Spec`, `Replicas` và `Age`.

Lưu CustomResourceDefinition vào `resourcedefinition.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              cronSpec:
                type: string
              image:
                type: string
              replicas:
                type: integer
    additionalPrinterColumns:
    - name: Spec
      type: string
      description: The cron spec defining the interval a CronJob is run
      jsonPath: .spec.cronSpec
    - name: Replicas
      type: integer
      description: The number of jobs launched by the CronJob
      jsonPath: .spec.replicas
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

Tạo CustomResourceDefinition:

```shell
kubectl apply -f resourcedefinition.yaml
```

Tạo một thực thể (instance) bằng file `my-crontab.yaml` ở mục trước.

Kích hoạt việc in ấn phía server:

```shell
kubectl get crontab my-new-cron-object
```

Hãy để ý các cột `NAME`, `SPEC`, `REPLICAS` và `AGE` trong kết quả:

```
NAME                 SPEC        REPLICAS   AGE
my-new-cron-object   * * * * *   1          7s
```

> **Ghi chú:** Cột `NAME` là ngầm định và không cần được định nghĩa trong
> CustomResourceDefinition.

#### Priority

Mỗi cột đều có một trường `priority`. Hiện tại, priority dùng để phân biệt giữa các cột hiển thị ở
chế độ xem tiêu chuẩn và chế độ xem rộng (dùng cờ `-o wide`).

- Các cột có priority bằng `0` được hiển thị ở chế độ xem tiêu chuẩn.
- Các cột có priority lớn hơn `0` chỉ được hiển thị ở chế độ xem rộng.

#### Type

Trường `type` của một cột có thể nhận một trong các giá trị sau (đối chiếu
[kiểu dữ liệu OpenAPI v3](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#dataTypes)):

- `integer` – số không dấu phẩy động
- `number` – số dấu phẩy động
- `string` – chuỗi
- `boolean` – `true` hoặc `false`
- `date` – được hiển thị dưới dạng khoảng thời gian đã trôi qua kể từ mốc thời gian này.

Nếu giá trị bên trong một CustomResource không khớp với kiểu đã khai báo cho cột thì giá trị đó bị
bỏ qua. Hãy dùng cơ chế kiểm tra hợp lệ của CustomResource để đảm bảo kiểu của giá trị là đúng.

#### Format

Trường `format` của một cột có thể nhận một trong các giá trị sau:

- `int32`
- `int64`
- `float`
- `double`
- `byte`
- `date`
- `date-time`
- `password`

Trường `format` của cột kiểm soát kiểu trình bày được `kubectl` dùng khi in giá trị.

### Field selector (Field selectors)

[Field selector](28-field-selectors-vi.md) cho phép client chọn các custom resource dựa trên giá
trị của một hoặc nhiều trường tài nguyên.

Mọi custom resource đều hỗ trợ field selector `metadata.name` và `metadata.namespace`.

Các trường được khai báo trong một CustomResourceDefinition cũng có thể được dùng với field
selector khi chúng được đưa vào trường `spec.versions[*].selectableFields` của
CustomResourceDefinition.

#### Các trường có thể chọn cho custom resource (Selectable fields for custom resources) {#crd-selectable-fields}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [stable]`

Trường `spec.versions[*].selectableFields` của một CustomResourceDefinition có thể được dùng để
khai báo những trường nào khác trong một custom resource có thể được dùng trong field selector,
với tính năng
[feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
`CustomResourceFieldSelectors` (feature gate này được bật mặc định kể từ Kubernetes v1.31).
Ví dụ dưới đây thêm các trường `.spec.color` và `.spec.size` làm trường có thể chọn.

Lưu CustomResourceDefinition vào `shirt-resource-definition.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: shirts.stable.example.com
spec:
  group: stable.example.com
  scope: Namespaced
  names:
    plural: shirts
    singular: shirt
    kind: Shirt
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              color:
                type: string
              size:
                type: string
    selectableFields:
    - jsonPath: .spec.color
    - jsonPath: .spec.size
    additionalPrinterColumns:
    - jsonPath: .spec.color
      name: Color
      type: string
    - jsonPath: .spec.size
      name: Size
      type: string
```

Tạo CustomResourceDefinition:

```shell
kubectl apply -f https://k8s.io/examples/customresourcedefinition/shirt-resource-definition.yaml
```

Định nghĩa một vài Shirt bằng cách soạn file `shirt-resources.yaml`; ví dụ:

```yaml
---
apiVersion: stable.example.com/v1
kind: Shirt
metadata:
  name: example1
spec:
  color: blue
  size: S
---
apiVersion: stable.example.com/v1
kind: Shirt
metadata:
  name: example2
spec:
  color: blue
  size: M
---
apiVersion: stable.example.com/v1
kind: Shirt
metadata:
  name: example3
spec:
  color: green
  size: M
```

Tạo các custom resource:

```shell
kubectl apply -f https://k8s.io/examples/customresourcedefinition/shirt-resources.yaml
```

Lấy toàn bộ tài nguyên:

```shell
kubectl get shirts.stable.example.com
```

Kết quả là:

```
NAME       COLOR  SIZE
example1   blue   S
example2   blue   M
example3   green  M
```

Lấy các áo màu xanh dương (truy xuất các Shirt có `color` là `blue`):

```shell
kubectl get shirts.stable.example.com --field-selector spec.color=blue
```

Kết quả sẽ là:

```
NAME       COLOR  SIZE
example1   blue   S
example2   blue   M
```

Chỉ lấy các tài nguyên có `color` là `green` và `size` là `M`:

```shell
kubectl get shirts.stable.example.com --field-selector spec.color=green,spec.size=M
```

Kết quả sẽ là:

```
NAME       COLOR   SIZE
example3   green   M
```

### Subresource (Subresources)

Custom resource hỗ trợ các subresource `/status` và `/scale`.

Các subresource status và scale có thể được bật tùy chọn bằng cách định nghĩa chúng trong
CustomResourceDefinition.

#### Subresource status (Status subresource)

Khi subresource status được bật, subresource `/status` của custom resource sẽ được phơi bày.

- Phần status và phần spec được biểu diễn tương ứng bởi các JSONPath `.status` và `.spec` bên
  trong một custom resource.
- Yêu cầu `PUT` tới subresource `/status` nhận một đối tượng custom resource và bỏ qua mọi thay
  đổi ngoại trừ phần status.
- Yêu cầu `PUT` tới subresource `/status` chỉ kiểm tra hợp lệ phần status của custom resource.
- Yêu cầu `PUT`/`POST`/`PATCH` tới custom resource sẽ bỏ qua các thay đổi lên phần status.
- Giá trị `.metadata.generation` được tăng lên với mọi thay đổi, ngoại trừ thay đổi lên
  `.metadata` hoặc `.status`.
- Chỉ những cấu trúc sau được phép đặt ở gốc của schema kiểm tra hợp lệ OpenAPI của CRD:

  - `description`
  - `example`
  - `exclusiveMaximum`
  - `exclusiveMinimum`
  - `externalDocs`
  - `format`
  - `items`
  - `maximum`
  - `maxItems`
  - `maxLength`
  - `minimum`
  - `minItems`
  - `minLength`
  - `multipleOf`
  - `pattern`
  - `properties`
  - `required`
  - `title`
  - `type`
  - `uniqueItems`

#### Subresource scale (Scale subresource)

Khi subresource scale được bật, subresource `/scale` của custom resource sẽ được phơi bày.
Đối tượng `autoscaling/v1.Scale` được gửi làm payload cho `/scale`.

Để bật subresource scale, các trường sau được định nghĩa trong CustomResourceDefinition.

- `specReplicasPath` định nghĩa JSONPath bên trong custom resource tương ứng với
  `scale.spec.replicas`.

  - Đây là giá trị bắt buộc.
  - Chỉ cho phép các JSONPath nằm dưới `.spec` và dùng ký pháp dấu chấm.
  - Nếu không có giá trị nào tại `specReplicasPath` trong custom resource thì subresource `/scale`
    sẽ trả về lỗi khi GET.

- `statusReplicasPath` định nghĩa JSONPath bên trong custom resource tương ứng với
  `scale.status.replicas`.

  - Đây là giá trị bắt buộc.
  - Chỉ cho phép các JSONPath nằm dưới `.status` và dùng ký pháp dấu chấm.
  - Nếu không có giá trị nào tại `statusReplicasPath` trong custom resource thì giá trị status
    replica trong subresource `/scale` sẽ mặc định là 0.

- `labelSelectorPath` định nghĩa JSONPath bên trong custom resource tương ứng với
  `Scale.Status.Selector`.

  - Đây là giá trị tùy chọn.
  - Nó phải được đặt thì mới hoạt động được với HPA và VPA.
  - Chỉ cho phép các JSONPath nằm dưới `.status` hoặc `.spec` và dùng ký pháp dấu chấm.
  - Nếu không có giá trị nào tại `labelSelectorPath` trong custom resource thì giá trị status
    selector trong subresource `/scale` sẽ mặc định là chuỗi rỗng.
  - Trường mà JSON path này trỏ tới phải là một trường kiểu chuỗi (không phải một struct selector
    phức tạp) chứa một label selector đã được tuần tự hóa ở dạng chuỗi.

Trong ví dụ dưới đây, cả subresource status lẫn scale đều được bật.

Lưu CustomResourceDefinition vào `resourcedefinition.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
            status:
              type: object
              properties:
                replicas:
                  type: integer
                labelSelector:
                  type: string
      # subresources mô tả các subresource cho custom resource.
      subresources:
        # status bật subresource status.
        status: {}
        # scale bật subresource scale.
        scale:
          # specReplicasPath định nghĩa JSONPath bên trong custom resource tương ứng với Scale.Spec.Replicas.
          specReplicasPath: .spec.replicas
          # statusReplicasPath định nghĩa JSONPath bên trong custom resource tương ứng với Scale.Status.Replicas.
          statusReplicasPath: .status.replicas
          # labelSelectorPath định nghĩa JSONPath bên trong custom resource tương ứng với Scale.Status.Selector.
          labelSelectorPath: .status.labelSelector
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
```

Và tạo nó:

```shell
kubectl apply -f resourcedefinition.yaml
```

Sau khi đối tượng CustomResourceDefinition đã được tạo, bạn có thể tạo các custom object.

Nếu bạn lưu YAML sau vào `my-crontab.yaml`:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
  replicas: 3
```

và tạo nó:

```shell
kubectl apply -f my-crontab.yaml
```

Khi đó các endpoint API RESTful mới ở phạm vi namespace được tạo tại:

```none
/apis/stable.example.com/v1/namespaces/*/crontabs/status
```

và

```none
/apis/stable.example.com/v1/namespaces/*/crontabs/scale
```

Một custom resource có thể được scale bằng lệnh `kubectl scale`.
Ví dụ, lệnh sau đặt `.spec.replicas` của custom resource vừa tạo ở trên thành 5:

```shell
kubectl scale --replicas=5 crontabs/my-new-cron-object
crontabs "my-new-cron-object" scaled

kubectl get crontabs my-new-cron-object -o jsonpath='{.spec.replicas}'
5
```

Bạn có thể dùng [PodDisruptionBudget](339-configure-pdb-vi.md) để bảo vệ các custom resource đã
bật subresource scale.

### Category (Categories)

Category là danh sách các nhóm tài nguyên mà custom resource thuộc về (ví dụ `all`).
Bạn có thể dùng `kubectl get <category-name>` để liệt kê các tài nguyên thuộc category đó.

Ví dụ dưới đây thêm `all` vào danh sách category trong CustomResourceDefinition và minh họa cách
xuất custom resource bằng `kubectl get all`.

Lưu CustomResourceDefinition sau vào `resourcedefinition.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
    # categories là danh sách các nhóm tài nguyên mà custom resource thuộc về.
    categories:
    - all
```

và tạo nó:

```shell
kubectl apply -f resourcedefinition.yaml
```

Sau khi đối tượng CustomResourceDefinition đã được tạo, bạn có thể tạo các custom object.

Lưu YAML sau vào `my-crontab.yaml`:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

và tạo nó:

```shell
kubectl apply -f my-crontab.yaml
```

Bạn có thể chỉ định category khi dùng `kubectl get`:

```shell
kubectl get all
```

và kết quả sẽ bao gồm cả các custom resource thuộc kind `CronTab`:

```none
NAME                          AGE
crontabs/my-new-cron-object   3s
```

## Tiếp theo (What's next)

* Đọc về [custom resource](179-custom-resources-vi.md).

* Xem [CustomResourceDefinition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#customresourcedefinition-v1-apiextensions-k8s-io).

* Phục vụ [nhiều phiên bản](377-custom-resource-definition-versioning-vi.md)
  của một CustomResourceDefinition.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 28.

1. Khi bạn `kubectl apply` một object CustomResourceDefinition, API server tạo ra chính xác cái gì,
   và vì sao `metadata.name` của CRD bắt buộc phải viết theo dạng `<plural>.<group>`? Ngay sau khi
   lệnh apply trả về, bạn nhìn vào đâu để biết endpoint đã sẵn sàng nhận custom object?
2. Trên `lab-k8s-master`, bạn tạo namespace `thu-nghiem`, apply vào đó một `CronTab`, rồi chạy
   `kubectl delete namespace thu-nghiem`. Sau lệnh đó object `CronTab` còn không? Còn
   CustomResourceDefinition `crontabs.stable.example.com` thì sao? Muốn dọn sạch cả kind lẫn dữ
   liệu thì phải xóa gì, và việc đó kéo theo hậu quả gì với mọi custom object cùng kind trên toàn
   cluster?
3. **Câu bẫy.** Schema của bạn khai `spec.replicas` kiểu `integer` với `minimum: 1`, `maximum: 10`,
   và **không** khai trường `spec.owner`. Bạn gửi thẳng lên API server một object có
   `replicas: 15` và `owner: "team-a"`. API server xử lý hai trường sai này giống nhau hay khác
   nhau? Nói rõ điều gì xảy ra với từng trường, và vì sao muốn thấy hành vi đó thì bài phải chạy
   `kubectl create` kèm `--validate=false`.
4. CRD của bạn đã bật `subresources: {status: {}}`. Một người dùng có quyền `patch` trên kind đó
   chạy `kubectl patch <tên> --type=merge -p '{"status":{"phase":"Ready"}}'` lên chính resource.
   Giá trị `.status.phase` có đổi không? Và khi controller ghi `status` qua đường `/status` thì
   `.metadata.generation` có tăng không?
5. Vì sao `kubectl get crontab` in ra đúng các cột `SPEC`, `REPLICAS`, `AGE` mà bạn không cấu hình
   gì trên máy chạy `kubectl`, và vì sao `kubectl get all` lại liệt kê được một kind do bạn định
   nghĩa?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. API server **tạo một đường dẫn tài nguyên RESTful mới cho mỗi version** khai trong
   `spec.versions` — với ví dụ trong bài là `/apis/stable.example.com/v1/namespaces/*/crontabs/...`
   vì `scope: Namespaced` — và đăng ký `kind` lấy từ `spec.names.kind`. `name` phải là
   `<plural>.<group>` vì **nó phải khớp với các trường `spec` bên dưới**: `plural` là đoạn cuối của
   URL và `group` là đoạn `/apis/<group>`; đặt tên khác đi thì CRD không hợp lệ. Về phần chờ: bài
   ghi "có thể mất vài giây để endpoint được tạo xong" — **theo dõi condition `Established` của
   CRD chuyển sang true**, hoặc theo dõi thông tin discovery của API server cho tới khi resource
   xuất hiện.
2. Object `CronTab` **mất theo namespace**: "việc xóa một namespace sẽ xóa toàn bộ custom object
   trong namespace đó", y hệt object dựng sẵn. CustomResourceDefinition thì **vẫn còn nguyên**:
   "bản thân các CustomResourceDefinition thì không thuộc namespace nào và có sẵn cho mọi
   namespace" — nó là object phạm vi cluster nên `kubectl delete namespace` không chạm tới. Muốn
   dọn cả kind thì phải **xóa chính CRD**, và hậu quả được bài nêu rõ: server **gỡ endpoint API
   RESTful và xóa toàn bộ custom object đang được lưu trong đó** — tức mọi `CronTab` ở mọi
   namespace, không riêng namespace bạn đang làm; tạo lại đúng CRD ấy sau này thì nó **bắt đầu ở
   trạng thái rỗng**. Đây chính là thí nghiệm bạn đã chạy ở [Lab 14](labs/LAB-14-CRD-VA-OPERATOR.md)
   phần B6.5.
3. **Khác nhau hoàn toàn** — đây là chỗ trực giác "sai thì bị từ chối hết" đánh lừa. `replicas: 15`
   vi phạm `maximum: 10`, thuộc mục *Kiểm tra hợp lệ*: **cả yêu cầu tạo bị từ chối**, kèm dòng
   `spec.replicas in body should be less than or equal to 10`, và **không có gì được lưu**.
   `owner` là trường **không được khai báo trong schema**, thuộc mục *Cắt tỉa trường*: nó **bị cắt
   bỏ trước khi lưu xuống**, còn yêu cầu vẫn thành công — object lưu trong etcd đơn giản là không
   có trường đó. Vì sao phải `--validate=false`: schema đã được **công bố ra OpenAPI tới client**,
   nên `kubectl` tự kiểm tra trường không xác định và **từ chối object ngay trước khi gửi đi**; tắt
   kiểm tra phía client mới quan sát được hành vi thật của API server.
4. **Không đổi.** Khi subresource `status` đã bật, "yêu cầu `PUT`/`POST`/`PATCH` tới custom resource
   sẽ bỏ qua các thay đổi lên phần status" — muốn ghi `status` phải đi qua đường `/status`, và yêu
   cầu tới `/status` thì ngược lại "bỏ qua mọi thay đổi ngoại trừ phần status". Về `generation`:
   **không tăng** — `.metadata.generation` tăng với mọi thay đổi **ngoại trừ** thay đổi lên
   `.metadata` hoặc `.status`. Nhờ đó `generation` chỉ phản ánh ý muốn người dùng đặt trong `spec`,
   còn controller ghi `status` bao nhiêu lần cũng không đụng tới nó.
5. Vì **kubectl dựa vào việc định dạng đầu ra ở phía server**: "API server của cluster quyết định
   những cột nào được hiển thị bởi lệnh `kubectl get`", và các cột đó đến từ
   `additionalPrinterColumns` khai trong CRD (cột `NAME` là ngầm định, không cần khai). `kubectl get all`
   liệt kê được kind mới vì `all` là một **category**: `spec.names.categories` khai kind đó thuộc
   nhóm `all`, và `kubectl get <category-name>` liệt kê mọi tài nguyên thuộc nhóm đó. Cùng lý do
   ấy, `shortNames` như `ct` cũng là khai báo trên CRD chứ không phải bí danh cấu hình ở máy bạn.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
