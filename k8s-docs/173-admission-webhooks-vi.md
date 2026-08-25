# Thực hành tốt cho Admission Webhook (Admission Webhook Good Practices)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/>
>
> Các khuyến nghị khi thiết kế và triển khai admission webhook trong Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 17/18 · Kiểm chứng ở [Lab 9b](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md).

Bài dài và viết cho người **thiết kế** webhook. Với vai quản trị viên, giá trị lớn nhất nằm ở
những chỗ một webhook có thể **làm chết cả cluster**: failure policy, tự biến đổi chính mình,
vòng lặp phụ thuộc, và những object tuyệt đối không được đụng. Đọc kỹ ba mục
*"Fail open" và kiểm tra hợp lệ trạng thái cuối cùng*, *Tránh tự biến đổi chính mình* và
*Tránh vòng lặp phụ thuộc*; phần còn lại đọc lấy nguyên tắc. Đây là bài cuối trước Lab 9b.

**Phải hiểu ở lần đọc này:**

- Vị trí trong luồng admission: **mutating webhook được gọi tuần tự**, **validating webhook được
  gọi song song**, và **toàn bộ mutating chạy xong trước khi bất kỳ validating nào chạy**. Mỗi
  lần gọi mutating webhook đều cộng thêm độ trễ cho quá trình kết nạp.
- `failurePolicy`: **mặc định là `Fail`** — webhook timeout hoặc lỗi logic thì API server **từ
  chối request**, nên một máy chủ webhook chết có thể chặn cả những request hợp lệ. Khuyến nghị
  của bài: để mutating webhook **"fail open"** bằng `Ignore`, rồi dùng một validating controller
  kiểm tra **trạng thái cuối cùng** để việc thực thi chính sách vẫn diễn ra.
- Ba kiểu tự làm chết mình: **tự biến đổi chính mình** — webhook chặn đúng tài nguyên cần để Pod
  của chính nó khởi động, nên khi node hỏng thì không lập lịch lại được; **vòng lặp phụ thuộc**
  — hai webhook kiểm tra Pod của nhau, hoặc webhook chặn add-on mà chính nó phụ thuộc; và **vòng
  lặp do controller cạnh tranh** — webhook thêm một label mà controller khác xóa đi, khiến
  webhook bị gọi lại mãi. Cách tránh chung: loại trừ bằng `namespaceSelector` và `objectSelector`.
- Những thứ webhook **không được đụng**: object trong namespace `kube-system`; **node lease**
  dạng Lease trong `kube-node-lease` — biến đổi chúng **có thể làm nâng cấp node thất bại**;
  **TokenReview và SubjectAccessReview** vì đây luôn là request chỉ đọc và sửa chúng **có thể
  làm hỏng cluster**; và các object bất biến như **mirror Pod của static Pod** (nhận ra qua
  annotation `kubernetes.io/config.mirror`).
- **Tính lũy đẳng**: mỗi mutating webhook phải chạy lại được trên chính object nó đã sửa mà
  không tạo thêm thay đổi — và **cả tập hợp webhook trong cluster cũng phải lũy đẳng**, không
  chỉ từng cái. Đi kèm: **đừng dựa vào thứ tự gọi**, vì mutating webhook không chạy theo thứ tự
  nhất quán.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Bảng bốn cơ chế mutating/validating, phần so sánh webhook với admission policy dùng CEL | cần biết CEL và các điểm mở rộng API | giai đoạn 14 |
| *Dùng cơ chế kiểm tra hợp lệ và đặt giá trị mặc định có sẵn cho CustomResourceDefinition* | chưa học CRD | giai đoạn 14 |
| Chi tiết `matchConditions`, `matchPolicy`, chính sách gọi lại (reinvocation policy) | tra cứu khi cấu hình webhook thật | giai đoạn 9, khi làm Lab 9b |
| Chính sách audit `RequestResponse` để bắt vòng lặp | cần audit backend | giai đoạn 22 audit/encryption |
| Danh sách ví dụ biến đổi lũy đẳng và không lũy đẳng | nắm nguyên tắc là đủ; ví dụ đọc khi tự viết webhook | giai đoạn 14 |
| *Ví dụ về các bản hiện thực tốt* — cert-manager, Gatekeeper | dự án bên thứ ba | không cần |

---

Trang này cung cấp các thực hành tốt và những điểm cần cân nhắc khi thiết kế
_admission webhook_ trong Kubernetes. Thông tin này dành cho
những người vận hành cluster đang chạy máy chủ admission webhook hoặc các ứng dụng bên thứ ba
có sửa đổi hoặc kiểm tra hợp lệ (validate) các request API của bạn.

Trước khi đọc trang này, hãy đảm bảo bạn đã quen thuộc với các khái niệm sau:

* [Admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
* [Admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#what-are-admission-webhooks)

## Tầm quan trọng của việc thiết kế webhook tốt (Importance of good webhook design) {#why-good-webhook-design-matters}

Kiểm soát kết nạp (admission control) diễn ra khi bất kỳ request tạo, cập nhật hoặc xóa nào
được gửi tới Kubernetes API. Các admission controller chặn lại những request
khớp với các tiêu chí cụ thể mà bạn định nghĩa. Các request này sau đó được gửi tới
các mutating admission webhook hoặc validating admission webhook. Những webhook này
thường được viết để đảm bảo rằng các trường cụ thể trong đặc tả (specification) của đối tượng tồn tại hoặc
có những giá trị được phép cụ thể.

Webhook là một cơ chế mạnh mẽ để mở rộng Kubernetes API. Các webhook được thiết kế tồi
thường dẫn đến gián đoạn workload, vì webhook nắm giữ rất nhiều quyền kiểm soát
đối với các đối tượng trong cluster. Giống như các cơ chế mở rộng API khác,
webhook rất khó kiểm thử ở quy mô lớn để đảm bảo tương thích với
toàn bộ workload, các webhook khác, add-on và plugin của bạn.

Ngoài ra, ở mỗi bản phát hành, Kubernetes bổ sung hoặc thay đổi API với các tính năng
mới, thăng cấp tính năng lên trạng thái beta hoặc stable, và các phần bị loại bỏ (deprecation). Ngay cả
các API Kubernetes ổn định (stable) cũng có khả năng thay đổi. Ví dụ, API `Pod` đã thay đổi
ở v1.29 để bổ sung tính năng
[Sidecar container](51-sidecar-containers-vi.md).
Dù hiếm khi một đối tượng Kubernetes rơi vào trạng thái hỏng vì một API Kubernetes mới,
những webhook từng hoạt động đúng như mong đợi với các phiên bản API trước đó
có thể không xử lý được những thay đổi mới hơn của API đó. Điều này có thể dẫn tới
hành vi ngoài dự kiến sau khi bạn nâng cấp cluster lên phiên bản mới hơn.

Trang này mô tả các kịch bản lỗi webhook thường gặp và cách tránh chúng bằng cách
thiết kế và hiện thực webhook một cách thận trọng và có cân nhắc kỹ.

## Xác định xem bạn có dùng admission webhook hay không (Identify whether you use admission webhooks) {#identify-admission-webhooks}

Ngay cả khi bạn không tự chạy admission webhook của mình, một số ứng dụng bên thứ ba
mà bạn chạy trong cluster có thể dùng mutating hoặc validating admission
webhook.

Để kiểm tra xem cluster của bạn có mutating admission webhook nào không, hãy chạy
lệnh sau:

```shell
kubectl get mutatingwebhookconfigurations
```
Kết quả liệt kê mọi mutating admission controller trong cluster.

Để kiểm tra xem cluster của bạn có validating admission webhook nào không, hãy chạy
lệnh sau:

```shell
kubectl get validatingwebhookconfigurations
```
Kết quả liệt kê mọi validating admission controller trong cluster.

## Chọn cơ chế kiểm soát kết nạp (Choose an admission control mechanism) {#choose-admission-mechanism}

Kubernetes bao gồm nhiều lựa chọn kiểm soát kết nạp và thực thi chính sách (policy).
Biết khi nào nên dùng một lựa chọn cụ thể có thể giúp bạn cải thiện độ trễ và
hiệu năng, giảm chi phí quản lý, và tránh sự cố khi nâng cấp phiên bản.
Bảng dưới đây mô tả các cơ chế cho phép bạn biến đổi (mutate) hoặc
kiểm tra hợp lệ (validate) tài nguyên trong quá trình kết nạp:

<!-- Bảng này viết bằng HTML vì nó dùng danh sách không đánh số để dễ đọc. -->
<table>
  <caption>Kiểm soát kết nạp kiểu mutating và validating trong Kubernetes</caption>
  <thead>
    <tr>
      <th>Cơ chế</th>
      <th>Mô tả</th>
      <th>Tình huống sử dụng</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><a href="https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/">Mutating admission webhook</a></td>
      <td>Chặn các request API trước khi kết nạp và sửa đổi khi cần bằng
        logic tùy chỉnh.</td>
      <td><ul>
        <li>Thực hiện những sửa đổi then chốt bắt buộc phải diễn ra trước khi tài nguyên
          được kết nạp.</li>
        <li>Thực hiện những sửa đổi phức tạp cần logic nâng cao, chẳng hạn gọi
          các API bên ngoài.</li>
      </ul></td>
    </tr>
    <tr>
      <td><a href="https://kubernetes.io/docs/reference/access-authn-authz/mutating-admission-policy/">Mutating admission policy</a></td>
      <td>Chặn các request API trước khi kết nạp và sửa đổi khi cần bằng
        các biểu thức Common Expression Language (CEL).</td>
      <td><ul>
        <li>Thực hiện những sửa đổi then chốt bắt buộc phải diễn ra trước khi tài nguyên
          được kết nạp.</li>
        <li>Thực hiện những sửa đổi đơn giản, chẳng hạn điều chỉnh label hay số lượng
        replica.</li>
      </ul></td>
    </tr>
    <tr>
      <td><a href="https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/">Validating admission webhook</a></td>
      <td>Chặn các request API trước khi kết nạp và kiểm tra hợp lệ dựa trên các khai báo
        chính sách phức tạp.</td>
      <td><ul>
        <li>Kiểm tra hợp lệ các cấu hình then chốt trước khi tài nguyên được kết nạp.</li>
        <li>Thực thi logic chính sách phức tạp trước khi kết nạp.</li>
      </ul></td>
    </tr>
    <tr>
      <td><a href="https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/">Validating admission policy</a></td>
      <td>Chặn các request API trước khi kết nạp và kiểm tra hợp lệ dựa trên các biểu thức
        CEL.</td>
      <td><ul>
        <li>Kiểm tra hợp lệ các cấu hình then chốt trước khi tài nguyên được kết nạp.</li>
        <li>Thực thi logic chính sách bằng các biểu thức CEL.</li>
      </ul></td>
    </tr>
  </tbody>
</table>

Nhìn chung, hãy dùng kiểm soát kết nạp kiểu _webhook_ khi bạn muốn một cách mở rộng để
khai báo hoặc cấu hình logic. Hãy dùng kiểm soát kết nạp dựa trên CEL có sẵn khi
bạn muốn khai báo logic đơn giản hơn mà không phải chịu chi phí vận hành một máy chủ
webhook. Dự án Kubernetes khuyến nghị bạn dùng kiểm soát kết nạp dựa trên CEL
khi có thể.

### Dùng cơ chế kiểm tra hợp lệ và đặt giá trị mặc định có sẵn cho CustomResourceDefinition (Use built-in validation and defaulting for CustomResourceDefinitions) {#no-crd-validation-defaulting}

Nếu bạn dùng CustomResourceDefinitions,
đừng dùng admission webhook để kiểm tra hợp lệ các giá trị trong đặc tả CustomResource
hoặc để đặt giá trị mặc định cho các trường. Kubernetes cho phép bạn định nghĩa các quy tắc kiểm tra hợp lệ
và giá trị mặc định của trường khi bạn tạo CustomResourceDefinition.

Để tìm hiểu thêm, xem các tài nguyên sau:

* [Quy tắc kiểm tra hợp lệ (Validation rules)](378-custom-resource-definitions-vi.md#validation-rules)
* [Đặt giá trị mặc định (Defaulting)](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#defaulting)

## Hiệu năng và độ trễ (Performance and latency) {#performance-latency}

Phần này mô tả các khuyến nghị để cải thiện hiệu năng và giảm
độ trễ. Tóm lại, chúng gồm:

* Hợp nhất các webhook và giới hạn số lần gọi API trên mỗi webhook.
* Dùng audit log để phát hiện các webhook lặp đi lặp lại cùng một hành động.
* Dùng cân bằng tải (load balancing) để đảm bảo tính sẵn sàng của webhook.
* Đặt giá trị timeout nhỏ cho mỗi webhook.
* Cân nhắc nhu cầu về tính sẵn sàng của cluster khi thiết kế webhook.

### Thiết kế admission webhook có độ trễ thấp (Design admission webhooks for low latency) {#design-admission-webhooks-low-latency}

Các mutating admission webhook được gọi tuần tự. Tùy vào cách thiết lập mutating
webhook, một số webhook có thể được gọi nhiều lần. Mỗi lần gọi mutating
webhook đều làm tăng độ trễ cho quá trình kết nạp. Điều này khác với validating
webhook, vốn được gọi song song.

Khi thiết kế các mutating webhook, hãy cân nhắc yêu cầu và ngưỡng chấp nhận về độ trễ của bạn.
Càng có nhiều mutating webhook trong cluster, khả năng độ trễ tăng lên càng lớn.

Hãy cân nhắc những điều sau để giảm độ trễ:

* Hợp nhất các webhook thực hiện thao tác biến đổi tương tự nhau trên các đối tượng khác nhau.
* Giảm số lần gọi API trong logic của máy chủ mutating webhook.
* Giới hạn các điều kiện khớp (match condition) của mỗi mutating webhook để giảm số lượng
  webhook bị kích hoạt bởi một request API cụ thể.
* Hợp nhất các webhook nhỏ vào một máy chủ và một cấu hình để dễ sắp xếp thứ tự
  và tổ chức.

### Ngăn vòng lặp gây ra bởi các controller cạnh tranh nhau (Prevent loops caused by competing controllers) {#prevent-loops-competing-controllers}

Hãy cân nhắc mọi thành phần khác đang chạy trong cluster có thể xung đột với
các biến đổi mà webhook của bạn thực hiện. Ví dụ, nếu webhook của bạn thêm một label
mà một controller khác lại xóa đi, webhook của bạn sẽ bị gọi lại. Điều này dẫn tới
một vòng lặp.

Để phát hiện các vòng lặp này, hãy thử cách sau:

1.  Cập nhật chính sách audit của cluster để ghi log các sự kiện audit. Dùng các
    tham số sau:

      * `level`: `RequestResponse`
      * `verbs`: `["patch"]`
      * `omitStages`: `RequestReceived`

    Đặt quy tắc audit để tạo sự kiện cho đúng những tài nguyên mà
    webhook của bạn biến đổi.

1.  Kiểm tra các sự kiện audit xem có webhook nào bị gọi lại nhiều lần với
    cùng một patch áp lên cùng một đối tượng hay không, hoặc có đối tượng nào bị
    cập nhật rồi hoàn tác một trường nhiều lần hay không.

### Đặt giá trị timeout nhỏ (Set a small timeout value) {#small-timeout}

Admission webhook nên xử lý xong càng nhanh càng tốt (thường tính bằng
mili giây), vì chúng làm tăng độ trễ cho request API. Hãy dùng timeout nhỏ cho
webhook.

Để biết chi tiết, xem
[Timeout](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#timeouts).

### Dùng bộ cân bằng tải để đảm bảo tính sẵn sàng của webhook (Use a load balancer to ensure webhook availability) {#load-balancer-webhook}

Admission webhook nên tận dụng một hình thức cân bằng tải nào đó để có được
tính sẵn sàng cao (high availability) và lợi ích về hiệu năng. Nếu webhook chạy bên trong
cluster, bạn có thể chạy nhiều backend webhook phía sau một Service kiểu
`ClusterIP`.

### Dùng mô hình triển khai có tính sẵn sàng cao (Use a high-availability deployment model) {#ha-deployment}

Hãy cân nhắc các yêu cầu về tính sẵn sàng của cluster khi thiết kế webhook.
Ví dụ, trong lúc node ngừng hoạt động hoặc mất cả một zone, Kubernetes đánh dấu các Pod là
`NotReady` để bộ cân bằng tải định tuyến lại lưu lượng sang các zone và node còn khả dụng.
Những cập nhật này lên Pod có thể kích hoạt các mutating webhook của bạn. Tùy vào
số lượng Pod bị ảnh hưởng, máy chủ mutating webhook có nguy cơ bị timeout
hoặc gây chậm trễ trong việc xử lý Pod. Kết quả là lưu lượng sẽ không được
định tuyến lại nhanh như bạn cần.

Hãy tính đến những tình huống như ví dụ trên khi viết webhook.
Loại trừ những thao tác phát sinh do Kubernetes phản ứng với các sự cố không thể tránh khỏi.

## Lọc request (Request filtering) {#request-filtering}

Phần này cung cấp các khuyến nghị về việc lọc xem request nào sẽ kích hoạt
webhook cụ thể nào. Tóm lại, chúng gồm:

* Giới hạn phạm vi webhook để tránh các thành phần hệ thống và các request chỉ đọc.
* Giới hạn webhook trong các namespace cụ thể.
* Dùng match condition để lọc request một cách chi tiết.
* Khớp với mọi phiên bản của một đối tượng.

### Giới hạn phạm vi của từng webhook (Limit the scope of each webhook) {#webhook-limit-scope}

Admission webhook chỉ được gọi khi một request API khớp với cấu hình webhook
tương ứng. Hãy giới hạn phạm vi của mỗi webhook để giảm những lần gọi không cần thiết
tới máy chủ webhook. Cân nhắc các giới hạn phạm vi sau:

* Tránh khớp với các đối tượng trong namespace `kube-system`. Nếu bạn chạy Pod của riêng mình
  trong namespace `kube-system`, hãy dùng
  [`objectSelector`](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#matching-requests-objectselector)
  để tránh biến đổi một workload trọng yếu.
* Đừng biến đổi các node lease, vốn tồn tại dưới dạng đối tượng Lease trong
  namespace hệ thống `kube-node-lease`. Biến đổi node lease có thể dẫn tới
  việc nâng cấp node thất bại. Chỉ áp dụng các biện pháp kiểm tra hợp lệ lên đối tượng Lease trong
  namespace này nếu bạn chắc chắn rằng chúng không đặt cluster của bạn vào rủi ro.
* Đừng biến đổi các đối tượng TokenReview hay SubjectAccessReview. Đây luôn là
  các request chỉ đọc. Sửa đổi những đối tượng này có thể làm hỏng cluster của bạn.
* Giới hạn mỗi webhook trong một namespace cụ thể bằng cách dùng
  [`namespaceSelector`](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#matching-requests-namespaceselector).

### Lọc các request cụ thể bằng match condition (Filter for specific requests by using match conditions) {#filter-match-conditions}

Admission controller hỗ trợ nhiều trường mà bạn có thể dùng để khớp các request
đáp ứng những tiêu chí cụ thể. Ví dụ, bạn có thể dùng `namespaceSelector` để
lọc các request nhắm tới một namespace cụ thể.

Để lọc request chi tiết hơn, hãy dùng trường `matchConditions` trong
cấu hình webhook của bạn. Trường này cho phép bạn viết nhiều biểu thức CEL mà
tất cả phải cho kết quả `true` thì request mới kích hoạt admission webhook của bạn. Dùng
`matchConditions` có thể giảm đáng kể số lần gọi tới máy chủ webhook
của bạn.

Để biết chi tiết, xem
[Khớp request: `matchConditions`](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#matching-requests-matchconditions).

### Khớp với mọi phiên bản của một API (Match all versions of an API) {#match-all-versions}

Theo mặc định, admission webhook chạy trên mọi phiên bản API có ảnh hưởng tới một
tài nguyên đã chỉ định. Trường `matchPolicy` trong cấu hình webhook điều khiển hành vi này.
Hãy chỉ định giá trị `Equivalent` trong trường `matchPolicy` hoặc bỏ qua
trường này để cho phép webhook chạy trên mọi phiên bản API.

Để biết chi tiết, xem
[Khớp request: `matchPolicy`](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#matching-requests-matchpolicy).

## Phạm vi biến đổi và những lưu ý về trường (Mutation scope and field considerations) {#mutation-scope-considerations}

Phần này cung cấp các khuyến nghị về phạm vi của các biến đổi và những lưu ý
đặc biệt đối với các trường của đối tượng. Tóm lại, chúng gồm:

* Chỉ patch những trường bạn cần patch.
* Đừng ghi đè giá trị mảng.
* Tránh tác dụng phụ (side effect) trong các biến đổi khi có thể.
* Tránh tự biến đổi chính mình (self-mutation).
* "Fail open" và kiểm tra hợp lệ trạng thái cuối cùng.
* Dự phòng cho việc cập nhật trường trong các phiên bản sau.
* Ngăn webhook tự kích hoạt chính nó.
* Đừng thay đổi các đối tượng bất biến (immutable).

### Chỉ patch những trường cần thiết (Patch only required fields) {#patch-required-fields}

Máy chủ admission webhook gửi phản hồi HTTP để cho biết cần làm gì với một
request API Kubernetes cụ thể. Phản hồi này là một đối tượng AdmissionReview.
Một mutating webhook có thể thêm các trường cụ thể cần biến đổi trước khi cho phép kết nạp
bằng cách dùng trường `patchType` và trường `patch` trong phản hồi. Hãy đảm bảo
rằng bạn chỉ sửa những trường cần thay đổi.

Ví dụ, hãy xét một mutating webhook được cấu hình để đảm bảo rằng các
Deployment `web-server` có ít nhất ba replica. Khi một request tạo
đối tượng Deployment khớp với cấu hình webhook của bạn, webhook
chỉ nên cập nhật giá trị trong trường `spec.replicas`.

### Đừng ghi đè giá trị mảng (Don't overwrite array values) {#dont-overwrite-arrays}

Các trường trong đặc tả đối tượng Kubernetes có thể chứa mảng. Một số mảng
chứa các cặp key:value (như trường `envVar` trong đặc tả container),
trong khi các mảng khác không có khóa (như trường `readinessGates` trong đặc tả
Pod). Thứ tự các giá trị trong một trường mảng có thể quan trọng trong một số
tình huống. Ví dụ, thứ tự các tham số trong trường `args` của một
đặc tả container có thể ảnh hưởng tới container đó.

Hãy cân nhắc những điều sau khi sửa mảng:

* Bất cứ khi nào có thể, dùng thao tác JSONPatch `add` thay vì `replace` để
  tránh vô tình thay thế một giá trị bắt buộc.
* Coi các mảng không dùng cặp key:value như là tập hợp (set).
* Đảm bảo rằng các giá trị trong trường bạn sửa không bắt buộc phải theo
  một thứ tự cụ thể.
* Đừng ghi đè các cặp key:value đã có trừ khi thực sự cần thiết.
* Hãy thận trọng khi sửa các trường label. Một sửa đổi vô ý có thể
  làm hỏng các label selector, dẫn đến hành vi ngoài ý muốn.

### Tránh tác dụng phụ (Avoid side effects) {#avoid-side-effects}

Hãy đảm bảo webhook của bạn chỉ thao tác trên nội dung của AdmissionReview
được gửi tới nó, và không thực hiện các thay đổi ngoài luồng (out-of-band). Những thay đổi
bổ sung này, gọi là _tác dụng phụ (side effect)_, có thể gây xung đột trong quá trình kết nạp nếu chúng
không được điều hòa (reconcile) đúng cách. Trường `.webhooks[].sideEffects` nên
được đặt là `None` nếu webhook không có bất kỳ tác dụng phụ nào.

Nếu tác dụng phụ là bắt buộc trong quá trình đánh giá kết nạp, chúng phải được
nén lại (suppress) khi xử lý một đối tượng AdmissionReview có `dryRun` đặt là
`true`, và trường `.webhooks[].sideEffects` nên được đặt là `NoneOnDryRun`.

Để biết chi tiết, xem
[Tác dụng phụ (Side effects)](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#side-effects).

### Tránh tự biến đổi chính mình (Avoid self-mutations) {#avoid-self-mutation}

Một webhook chạy bên trong cluster có thể gây deadlock cho chính bản triển khai
của nó nếu nó được cấu hình để chặn các tài nguyên cần thiết để khởi động chính
các Pod của nó.

Ví dụ, một mutating admission webhook được cấu hình để chỉ cho phép các request
**create** Pod nếu Pod có đặt một label nhất định (chẳng hạn `env: prod`).
Máy chủ webhook lại chạy trong một Deployment không đặt label `env`.

Khi một node đang chạy các Pod của máy chủ webhook trở nên không khỏe mạnh, Deployment
của webhook sẽ cố lập lịch lại các Pod sang node khác. Tuy nhiên, máy chủ webhook
hiện có lại từ chối các request đó vì label `env` chưa được đặt. Kết quả là
việc di chuyển không thể diễn ra.

Hãy loại trừ namespace nơi webhook của bạn đang chạy bằng
[`namespaceSelector`](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#matching-requests-namespaceselector).

### Tránh vòng lặp phụ thuộc (Avoid dependency loops) {#avoid-dependency-loops}

Vòng lặp phụ thuộc có thể xảy ra trong những kịch bản như sau:

* Hai webhook kiểm tra Pod của nhau. Nếu cả hai webhook cùng không khả dụng
  tại cùng một thời điểm, không webhook nào có thể khởi động được.
* Webhook của bạn chặn các thành phần add-on của cluster, chẳng hạn plugin mạng
  hoặc plugin lưu trữ, mà chính webhook của bạn lại phụ thuộc vào. Nếu cả webhook và
  add-on phụ thuộc đó đều không khả dụng, không thành phần nào có thể hoạt động.

Để tránh các vòng lặp phụ thuộc này, hãy thử cách sau:

* Dùng
  [ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
  để tránh tạo ra các phụ thuộc.
* Ngăn các webhook kiểm tra hợp lệ hoặc biến đổi các webhook khác. Hãy cân nhắc
  [loại trừ những namespace cụ thể](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#matching-requests-namespaceselector)
  khỏi việc kích hoạt webhook của bạn.
* Ngăn webhook của bạn tác động lên các add-on mà nó phụ thuộc bằng cách dùng
  [`objectSelector`](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#matching-requests-objectselector).

### "Fail open" và kiểm tra hợp lệ trạng thái cuối cùng (Fail open and validate the final state) {#fail-open-validate-final-state}

Mutating admission webhook hỗ trợ trường cấu hình `failurePolicy`.
Trường này cho biết API server nên kết nạp hay từ chối request
nếu webhook gặp lỗi. Lỗi webhook có thể xảy ra do timeout hoặc do lỗi
trong logic của máy chủ.

Theo mặc định, admission webhook đặt trường `failurePolicy` là Fail. API
server sẽ từ chối một request nếu webhook gặp lỗi. Tuy nhiên, việc từ chối request theo
mặc định có thể khiến các request hợp lệ bị từ chối trong thời gian webhook
ngừng hoạt động.

Hãy để các mutating webhook của bạn "fail open" bằng cách đặt trường `failurePolicy` là
Ignore. Dùng một validating controller để kiểm tra trạng thái của các request nhằm đảm bảo
chúng tuân thủ các chính sách của bạn.

Cách tiếp cận này có những lợi ích sau:

* Thời gian mutating webhook ngừng hoạt động không ảnh hưởng tới việc triển khai các tài nguyên hợp lệ.
* Việc thực thi chính sách diễn ra trong giai đoạn kiểm soát kết nạp kiểu validating.
* Mutating webhook không can nhiễu tới các controller khác trong cluster.

### Dự phòng cho các cập nhật trường trong tương lai (Plan for future updates to fields) {#plan-future-field-updates}

Nhìn chung, hãy thiết kế webhook của bạn với giả định rằng các API Kubernetes có thể
thay đổi ở phiên bản sau. Đừng viết một máy chủ mặc nhiên cho rằng một API sẽ
ổn định mãi mãi. Ví dụ, việc phát hành sidecar container trong Kubernetes
đã thêm trường `restartPolicy` vào API Pod.

### Ngăn webhook của bạn tự kích hoạt chính nó (Prevent your webhook from triggering itself) {#prevent-webhook-self-trigger}

Các mutating webhook phản hồi cho một phạm vi rộng các request API có thể
vô tình tự kích hoạt chính chúng. Ví dụ, hãy xét một webhook phản hồi
mọi request trong cluster. Nếu bạn cấu hình webhook để tạo
đối tượng Event cho mỗi lần biến đổi, nó sẽ phản hồi chính các request tạo
đối tượng Event của mình.

Để tránh điều này, hãy cân nhắc đặt một label duy nhất trong mọi tài nguyên mà
webhook của bạn tạo ra. Loại trừ label này khỏi các match condition của webhook.

### Đừng thay đổi các đối tượng bất biến (Don't change immutable objects) {#dont-change-immutable-objects}

Một số đối tượng Kubernetes trong API server không thể thay đổi. Ví dụ, khi bạn
triển khai một static Pod,
kubelet trên node sẽ tạo một
mirror Pod trong API
server để theo dõi static Pod đó. Tuy nhiên, các thay đổi lên mirror Pod không
lan truyền tới static Pod.

Đừng cố biến đổi những đối tượng này trong quá trình kết nạp. Mọi mirror Pod đều có
annotation `kubernetes.io/config.mirror`. Để loại trừ mirror Pod mà vẫn giảm
rủi ro bảo mật của việc bỏ qua một annotation, hãy chỉ cho phép static Pod chạy trong
những namespace cụ thể.

## Thứ tự và tính lũy đẳng của mutating webhook (Mutating webhook ordering and idempotence) {#ordering-idempotence}

Phần này cung cấp các khuyến nghị về thứ tự webhook và cách thiết kế webhook
lũy đẳng (idempotent). Tóm lại, chúng gồm:

* Đừng dựa vào một thứ tự thực thi cụ thể.
* Kiểm tra hợp lệ các biến đổi trước khi kết nạp.
* Kiểm tra xem các biến đổi có bị controller khác ghi đè hay không.
* Đảm bảo rằng cả tập hợp các mutating webhook là lũy đẳng, chứ không chỉ
  từng webhook riêng lẻ.

### Đừng dựa vào thứ tự gọi của mutating webhook (Don't rely on mutating webhook invocation order) {#dont-rely-webhook-order}

Các mutating admission webhook không chạy theo một thứ tự nhất quán. Nhiều yếu tố khác nhau
có thể làm thay đổi thời điểm một webhook cụ thể được gọi. Đừng dựa vào việc webhook của bạn
chạy tại một thời điểm cụ thể trong quá trình kết nạp. Các webhook khác vẫn có thể
biến đổi đối tượng đã được bạn sửa đổi.

Những khuyến nghị sau có thể giúp giảm thiểu rủi ro về các thay đổi ngoài ý muốn:

* [Kiểm tra hợp lệ các biến đổi trước khi kết nạp](#validate-mutations)
* Dùng chính sách gọi lại (reinvocation policy) để quan sát các thay đổi lên một đối tượng do các plugin khác
  thực hiện và chạy lại webhook khi cần. Để biết chi tiết, xem
  [Chính sách gọi lại (Reinvocation policy)](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#reinvocation-policy).

### Đảm bảo các mutating webhook trong cluster của bạn là lũy đẳng (Ensure that the mutating webhooks in your cluster are idempotent) {#ensure-mutating-webhook-idempotent}

Mọi mutating admission webhook đều nên _lũy đẳng (idempotent)_. Webhook phải có thể
chạy trên một đối tượng mà chính nó đã sửa đổi mà không tạo thêm
thay đổi nào ngoài thay đổi ban đầu.

Ngoài ra, toàn bộ các mutating webhook trong cluster của bạn, xét như một
tập hợp, cũng phải lũy đẳng. Sau khi giai đoạn biến đổi của kiểm soát kết nạp kết thúc,
mỗi mutating webhook riêng lẻ đều phải có thể chạy trên đối tượng đó mà không
tạo thêm thay đổi nào lên đối tượng.

Tùy vào môi trường của bạn, việc đảm bảo tính lũy đẳng ở quy mô lớn có thể là
thách thức. Những khuyến nghị sau có thể giúp ích:

* Dùng validating admission controller để xác minh trạng thái cuối cùng của
  các workload trọng yếu.
* Kiểm thử các bản triển khai của bạn trong một cluster staging để xem có đối tượng nào bị sửa đổi
  nhiều lần bởi cùng một webhook hay không.
* Đảm bảo phạm vi của mỗi mutating webhook là cụ thể và có giới hạn.

Các ví dụ sau cho thấy logic biến đổi lũy đẳng:

1. Với một request **create** Pod, đặt trường
  `.spec.securityContext.runAsNonRoot` của Pod thành true.

1. Với một request **create** Pod, nếu trường
   `.spec.containers[].resources.limits` của một container chưa được đặt, hãy đặt các
   giới hạn tài nguyên mặc định.

1. Với một request **create** Pod, chèn một sidecar container tên
   `foo-sidecar` nếu chưa tồn tại container nào tên `foo-sidecar`.

Trong những trường hợp này, webhook có thể được gọi lại một cách an toàn, hoặc kết nạp một đối tượng
vốn đã có sẵn các trường được đặt.

Các ví dụ sau cho thấy logic biến đổi không lũy đẳng:

1. Với một request **create** Pod, chèn một sidecar container tên
   `foo-sidecar` có hậu tố là dấu thời gian hiện tại (chẳng hạn
   `foo-sidecar-19700101-000000`).

   Việc gọi lại webhook có thể dẫn đến cùng một sidecar bị chèn nhiều lần
   vào một Pod, mỗi lần với một tên container khác nhau. Tương tự,
   webhook có thể chèn các container trùng lặp nếu sidecar đã tồn tại sẵn
   trong pod do người dùng cung cấp.

1. Với một request **create**/**update** Pod, từ chối nếu Pod có đặt label `env`,
   ngược lại thì thêm label `env: prod` vào Pod.

   Việc gọi lại webhook sẽ khiến webhook thất bại trên chính đầu ra của nó.

1. Với một request **create** Pod, nối thêm một sidecar container tên `foo-sidecar`
   mà không kiểm tra xem container `foo-sidecar` đã tồn tại hay chưa.

   Việc gọi lại webhook sẽ dẫn tới các container trùng lặp trong Pod, khiến
   request trở nên không hợp lệ và bị API server từ chối.

## Kiểm thử và kiểm tra hợp lệ các biến đổi (Mutation testing and validation) {#mutation-testing-validation}

Phần này cung cấp các khuyến nghị về việc kiểm thử các mutating webhook và
kiểm tra hợp lệ các đối tượng đã bị biến đổi. Tóm lại, chúng gồm:

* Kiểm thử webhook trong môi trường staging.
* Tránh các biến đổi vi phạm các quy tắc kiểm tra hợp lệ.
* Kiểm thử các lần nâng cấp phiên bản minor để phát hiện hồi quy và xung đột.
* Kiểm tra hợp lệ các đối tượng đã biến đổi trước khi kết nạp.

### Kiểm thử webhook trong môi trường staging (Test webhooks in staging environments) {#test-in-staging-environments}

Kiểm thử kỹ lưỡng nên là một phần cốt lõi trong chu trình phát hành của bạn cho các webhook mới hoặc
được cập nhật. Nếu có thể, hãy kiểm thử mọi thay đổi lên các webhook của cluster trong một môi trường
staging giống với các cluster production của bạn nhất có thể. Tối thiểu,
hãy cân nhắc dùng một công cụ như [minikube](https://minikube.sigs.k8s.io/docs/) hoặc
[kind](https://kind.sigs.k8s.io/) để tạo một cluster kiểm thử nhỏ cho các thay đổi
webhook.

### Đảm bảo các biến đổi không vi phạm các quy tắc kiểm tra hợp lệ (Ensure that mutations don't violate validations) {#ensure-mutations-dont-violate-validations}

Các mutating webhook của bạn không nên phá vỡ bất kỳ quy tắc kiểm tra hợp lệ nào áp dụng cho một
đối tượng trước khi kết nạp. Ví dụ, hãy xét một mutating webhook đặt
CPU request mặc định của một Pod thành một giá trị cụ thể. Nếu CPU limit của Pod đó
được đặt thấp hơn giá trị request đã bị biến đổi, Pod sẽ không qua được bước kết nạp.

Hãy kiểm thử mọi mutating webhook đối chiếu với các quy tắc kiểm tra hợp lệ đang chạy trong cluster của bạn.

### Kiểm thử các lần nâng cấp phiên bản minor để đảm bảo hành vi nhất quán (Test minor version upgrades to ensure consistent behavior) {#test-minor-version-upgrades}

Trước khi nâng cấp các cluster production lên một phiên bản minor mới, hãy kiểm thử các
webhook và workload của bạn trong một môi trường staging. So sánh kết quả để đảm bảo
rằng các webhook của bạn vẫn hoạt động như mong đợi sau khi nâng cấp.

Ngoài ra, hãy dùng các tài nguyên sau để cập nhật thông tin về các thay đổi API:

* [Ghi chú phát hành Kubernetes (Kubernetes release notes)](https://kubernetes.io/releases/)
* [Blog Kubernetes](https://kubernetes.io/blog/)

### Kiểm tra hợp lệ các biến đổi trước khi kết nạp (Validate mutations before admission) {#validate-mutations}

Các mutating webhook chạy xong hoàn toàn trước khi bất kỳ validating webhook nào chạy. Không có
thứ tự ổn định nào cho việc áp dụng các biến đổi lên đối tượng. Kết quả là, các
biến đổi của bạn có thể bị ghi đè bởi một mutating webhook chạy sau đó.

Hãy thêm một validating admission controller như ValidatingAdmissionWebhook hoặc
ValidatingAdmissionPolicy vào cluster của bạn để đảm bảo rằng các biến đổi của bạn
vẫn còn nguyên. Ví dụ, hãy xét một mutating webhook chèn trường
`restartPolicy: Always` vào các init container cụ thể để chúng chạy như
sidecar container. Bạn có thể chạy một validating webhook để đảm bảo rằng những
init container đó vẫn giữ cấu hình `restartPolicy: Always` sau khi mọi
biến đổi đã hoàn tất.

Để biết chi tiết, xem các tài nguyên sau:

* [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
* [ValidatingAdmissionWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook)

## Triển khai mutating webhook (Mutating webhook deployment) {#mutating-webhook-deployment}

Phần này cung cấp các khuyến nghị để triển khai các mutating admission
webhook của bạn. Tóm lại, chúng gồm:

* Triển khai cấu hình webhook dần dần và theo dõi vấn đề theo từng namespace.
* Giới hạn quyền truy cập để chỉnh sửa các tài nguyên cấu hình webhook.
* Giới hạn quyền truy cập vào namespace chạy máy chủ webhook, nếu máy chủ nằm
  trong cluster.

### Cài đặt và bật một mutating webhook (Install and enable a mutating webhook) {#install-enable-mutating-webhook}

Khi bạn đã sẵn sàng triển khai mutating webhook lên một cluster, hãy dùng
thứ tự thao tác sau:

1.  Cài đặt máy chủ webhook và khởi động nó.
1.  Đặt trường `failurePolicy` trong manifest MutatingWebhookConfiguration
    thành Ignore. Điều này giúp bạn tránh gián đoạn do các webhook bị cấu hình sai.
1.  Đặt trường `namespaceSelector` trong manifest MutatingWebhookConfiguration
    trỏ tới một namespace dùng để kiểm thử.
1.  Triển khai MutatingWebhookConfiguration lên cluster của bạn.

Hãy theo dõi webhook trong namespace kiểm thử để phát hiện mọi vấn đề, rồi mới triển khai
webhook ra các namespace khác. Nếu webhook chặn một request API mà lẽ ra
nó không nên chặn, hãy tạm dừng việc triển khai và điều chỉnh phạm vi của
cấu hình webhook.

### Giới hạn quyền chỉnh sửa mutating webhook (Limit edit access to mutating webhooks) {#limit-edit-access}

Mutating webhook là những controller Kubernetes mạnh mẽ. Hãy dùng RBAC hoặc một cơ chế
phân quyền khác để giới hạn quyền truy cập vào các cấu hình và máy chủ webhook của bạn. Với RBAC,
hãy đảm bảo rằng các quyền truy cập sau chỉ dành cho những thực thể đáng tin cậy:

* Verb: **create**, **update**, **patch**, **delete**, **deletecollection**
* API group: `admissionregistration.k8s.io/v1`
* API kind: MutatingWebhookConfigurations

Nếu máy chủ mutating webhook của bạn chạy trong cluster, hãy giới hạn quyền tạo hoặc
sửa đổi bất kỳ tài nguyên nào trong namespace đó.

## Ví dụ về các bản hiện thực tốt (Examples of good implementations) {#example-good-implementations}

> **Ghi chú:** Mục này liên kết tới các sản phẩm bên thứ ba hoặc dự án cung cấp
> chức năng cần thiết cho Kubernetes. Dự án Kubernetes không chịu trách nhiệm
> về các sản phẩm hoặc dự án bên thứ ba này.

Các dự án sau là ví dụ về những bản hiện thực máy chủ webhook tùy chỉnh "tốt".
Bạn có thể dùng chúng làm điểm khởi đầu khi thiết kế webhook của riêng mình.
Đừng dùng các ví dụ này y nguyên; hãy dùng chúng làm điểm khởi đầu và
thiết kế webhook của bạn sao cho chạy tốt trong môi trường cụ thể của bạn.

* [`cert-manager`](https://github.com/cert-manager/cert-manager/tree/master/internal/webhook)
* [Gatekeeper Open Policy Agent (OPA)](https://open-policy-agent.github.io/gatekeeper/website/docs/mutation)

## Tiếp theo (What's next)

* [Dùng webhook cho xác thực và phân quyền](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)
* [Tìm hiểu về MutatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/mutating-admission-policy/)
* [Tìm hiểu về ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. `failurePolicy` mặc định là gì? Chuyện gì xảy ra với cluster khi máy chủ webhook chết, và
   bài khuyên đặt thế nào cho mutating webhook cùng bù lại bằng gì?
2. Một webhook chạy trong cluster, cấu hình chỉ cho phép request **create** Pod khi Pod có label
   `env: prod`; nhưng Deployment của chính máy chủ webhook lại không đặt label đó. Cluster đang
   chạy bình thường. Chuyện gì xảy ra khi node đang chạy Pod webhook trở nên không khỏe mạnh?
3. Mutating webhook và validating webhook khác nhau thế nào về thứ tự gọi, và điều đó dẫn tới
   khuyến nghị nào về việc kiểm tra trạng thái cuối cùng?
4. Trên cluster lab, control plane chạy bằng static Pod nên kubelet tạo mirror Pod trong API
   server, còn `lab-k8s-worker1` và `lab-k8s-worker2` liên tục gia hạn Lease trong `kube-node-lease`.
   Nếu bạn cài một mutating webhook khớp mọi object trong cluster, hai chỗ nào bài cảnh báo
   tuyệt đối không được biến đổi, và hậu quả là gì?
5. "Lũy đẳng" nghĩa là gì với một mutating webhook, và vì sao từng webhook lũy đẳng vẫn chưa đủ?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Mặc định là **`Fail`**: API server **từ chối request nếu webhook gặp lỗi**, dù lỗi đó chỉ là
   timeout. Hệ quả là **trong thời gian webhook ngừng hoạt động, các request hợp lệ cũng bị từ
   chối** — một máy chủ webhook chết có thể chặn việc triển khai trên toàn cluster. Bài khuyên
   để **mutating webhook "fail open"** bằng cách đặt `failurePolicy: Ignore`, rồi **dùng một
   validating controller để kiểm tra trạng thái của request**, nhờ đó việc thực thi chính sách
   chuyển sang giai đoạn validating và thời gian mutating webhook chết không ảnh hưởng tới việc
   triển khai tài nguyên hợp lệ.
2. **Deadlock — Pod webhook không bao giờ được lập lịch lại.** Deployment của webhook cố lập
   lịch Pod sang node khác, nhưng **chính máy chủ webhook hiện có lại từ chối các request đó** vì
   Pod không có label `env`. Cluster chạy bình thường cho đến khi cần khởi động lại đúng thành
   phần đó — đó là lý do bài gọi đây là **tự biến đổi chính mình**. Cách tránh: **loại trừ
   namespace nơi webhook đang chạy bằng `namespaceSelector`**.
3. **Mutating webhook được gọi tuần tự**, có thể bị gọi nhiều lần, và **không có thứ tự nhất
   quán**; **validating webhook được gọi song song**, và **toàn bộ mutating chạy xong trước khi
   bất kỳ validating nào chạy**. Vì không có thứ tự ổn định, **biến đổi của bạn có thể bị một
   mutating webhook chạy sau ghi đè** — nên bài khuyên thêm một **validating admission
   controller** (ValidatingAdmissionWebhook hoặc ValidatingAdmissionPolicy) để **đảm bảo các
   biến đổi vẫn còn nguyên** sau khi mọi biến đổi đã hoàn tất.
4. **Node lease trong `kube-node-lease`** và **mirror Pod của static Pod**. Bài nói **đừng biến
   đổi node lease, vì biến đổi chúng có thể dẫn tới việc nâng cấp node thất bại** — chỉ nên áp
   biện pháp kiểm tra hợp lệ lên Lease trong namespace này khi bạn chắc chắn không đặt cluster
   vào rủi ro. Với mirror Pod, đây là **object bất biến**: thay đổi lên mirror Pod **không lan
   truyền tới static Pod**, nên biến đổi nó là vô nghĩa và gây sai lệch; nhận ra mirror Pod qua
   annotation **`kubernetes.io/config.mirror`**, và để giảm rủi ro của việc bỏ qua theo annotation
   thì **chỉ cho phép static Pod chạy trong những namespace cụ thể**.
5. Lũy đẳng nghĩa là webhook **có thể chạy trên một object mà chính nó đã sửa đổi mà không tạo
   thêm thay đổi nào ngoài thay đổi ban đầu**. Từng webhook lũy đẳng chưa đủ vì bài đòi hỏi
   **toàn bộ tập mutating webhook trong cluster, xét như một tập hợp, cũng phải lũy đẳng**: sau
   khi giai đoạn biến đổi kết thúc, **mỗi webhook riêng lẻ đều phải chạy lại được trên object đó
   mà không tạo thêm thay đổi**. Hai webhook, mỗi cái tự nó ổn định, vẫn có thể liên tục sửa
   ngược nhau.

</details>

Đây là bài cuối của **phần policy và hardening** trong giai đoạn 9. Câu nào chưa trả lời được
thì quay lại đúng mục tương ứng trước khi vào [**Lab 9b — Pod Security và hardening**](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md).
