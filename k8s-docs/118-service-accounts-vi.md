# Tài khoản dịch vụ (Service Accounts)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/service-accounts/>
>
> Tìm hiểu về các object ServiceAccount trong Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 3/18 · Kiểm chứng ở [Lab 9a](labs/LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md).

Đây là bài trả lời câu hỏi bạn đã gặp hai lần trước đó. Bài
[24](24-control-plane-node-communication-vi.md) nói Pod kết nối an toàn tới API server bằng
cách "tận dụng service account" mà không giải thích; còn [Lab 1a phần
B8](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) cho bạn xóa ServiceAccount `default` rồi
xem controller tạo lại nó. Bài này là chỗ hai mảnh đó khớp vào nhau.

**Phải hiểu ở lần đọc này:**

- ServiceAccount là **tài khoản phi con người**, có object thật trong API server, thuộc phạm vi
  namespace. Đối lập: tài khoản người dùng **không tồn tại** trong API server — API server coi
  danh tính người dùng là dữ liệu mờ (opaque). Bảng so sánh trong bài chốt điểm này.
- Mỗi namespace có ServiceAccount `default`; xóa đi thì **control plane thay thế nó bằng một
  object mới**. Pod không được gán ServiceAccount thủ công thì nhận `default`, và `default`
  mặc định **không có quyền hạn nào** ngoài các quyền API discovery.
- Ba bước dùng một service account: tạo object → **cấp quyền bằng cơ chế phân quyền như RBAC**
  → gán cho Pod qua `spec.serviceAccountName`.
- Từ v1.22, Kubernetes cấp cho Pod một token **ngắn hạn, tự động xoay vòng** lấy qua API
  `TokenRequest` và mount bằng projected volume — khác hẳn token tĩnh dạng Secret của các bản
  trước. Tắt việc tự động tiêm bằng `automountServiceAccountToken: false`.
- Cách API server kiểm tra bearer token: chữ ký → hết hạn → tham chiếu object trong claim còn
  hợp lệ → token còn hiệu lực → claim về audience. Token của `TokenRequest` là **bound token**,
  gắn với vòng đời của Pod đang dùng nó.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Truy cập liên namespace bằng ServiceAccount* | cần Role và RoleBinding trước | bài [120](120-rbac-good-practices-vi.md) |
| *Giới hạn audience trên node đối với service account token* | là hardening cho danh tính kubelet | bài [128](128-api-server-bypass-risks-vi.md) |
| *Hạn chế truy cập Secret (đã lỗi thời)* — `kubernetes.io/enforce-mountable-secrets` | đã deprecated từ v1.32, thay bằng namespace riêng | bài [121](121-secrets-good-practices-vi.md) |
| *Xác thực credentials trong mã của riêng bạn* — TokenReview, OIDC discovery | dành cho người viết dịch vụ, không phải quản trị viên | không cần |
| *Các phương án thay thế* — SPIFFE, service mesh, OIDC, IAM của cloud | là phần so sánh cơ chế xác thực | bài [123](123-hardening-authentication-vi.md) |

---

Trang này giới thiệu object ServiceAccount trong Kubernetes, cung cấp
thông tin về cách service account hoạt động, các trường hợp sử dụng, những hạn chế,
các phương án thay thế, cùng những liên kết đến tài nguyên hướng dẫn bổ sung.

## Service account là gì? (What are service accounts?) {#what-are-service-accounts}

Tài khoản dịch vụ (service account) là một loại tài khoản phi con người (non-human account),
trong Kubernetes, cung cấp một danh tính (identity) riêng biệt trong một cluster Kubernetes.
Các Pod ứng dụng, các thành phần hệ thống, và các thực thể bên trong lẫn bên ngoài cluster
có thể dùng thông tin xác thực (credentials) của một ServiceAccount cụ thể để định danh
là ServiceAccount đó. Danh tính này hữu ích trong nhiều tình huống, bao gồm việc xác thực
với API server hoặc triển khai các chính sách bảo mật dựa trên danh tính.

Service account tồn tại dưới dạng các object ServiceAccount trong API server. Service
account có các đặc điểm sau:

* **Thuộc phạm vi namespace (Namespaced):** Mỗi service account gắn với một
  namespace của Kubernetes. Mỗi namespace
  khi được tạo đều có một [ServiceAccount `default`](#default-service-accounts).

* **Gọn nhẹ (Lightweight):** Service account tồn tại trong cluster và được
  định nghĩa trong Kubernetes API. Bạn có thể nhanh chóng tạo các service account để
  phục vụ những tác vụ cụ thể.

* **Di động (Portable):** Một gói cấu hình cho một workload đóng gói container phức tạp
  có thể bao gồm các định nghĩa service account cho các thành phần của hệ thống. Bản chất
  gọn nhẹ của service account và danh tính theo phạm vi namespace giúp
  các cấu hình này dễ di chuyển.

Service account khác với tài khoản người dùng (user account) — vốn là những người dùng
thực đã được xác thực trong cluster. Mặc định, tài khoản người dùng không tồn tại trong
API server của Kubernetes; thay vào đó, API server coi danh tính người dùng là dữ liệu
mờ (opaque). Bạn có thể xác thực dưới danh nghĩa một tài khoản người dùng bằng nhiều
phương thức khác nhau. Một số bản phân phối Kubernetes có thể thêm các API mở rộng tùy chỉnh
để biểu diễn tài khoản người dùng trong API server.

*So sánh giữa service account và người dùng (Comparison between service accounts and users)*

| Mô tả | ServiceAccount | Người dùng hoặc nhóm (User or group) |
| --- | --- | --- |
| Vị trí | Kubernetes API (object ServiceAccount) | Bên ngoài |
| Kiểm soát truy cập | Kubernetes RBAC hoặc các [cơ chế phân quyền khác](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules) | Kubernetes RBAC hoặc các cơ chế quản lý danh tính và truy cập (IAM) khác |
| Mục đích sử dụng | Workload, tự động hóa | Con người |

### Service account mặc định (Default service accounts) {#default-service-accounts}

Khi bạn tạo một cluster, Kubernetes tự động tạo một object ServiceAccount
tên là `default` cho mỗi namespace trong cluster của bạn. Các service account `default`
trong từng namespace mặc định không có quyền hạn nào ngoài
[các quyền API discovery mặc định](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings)
mà Kubernetes cấp cho mọi principal đã xác thực nếu kiểm soát truy cập dựa trên vai trò (role-based access control — RBAC) được bật.
Nếu bạn xóa object ServiceAccount `default` trong một namespace,
control plane sẽ thay thế nó bằng một object mới.

Nếu bạn triển khai một Pod trong một namespace và bạn không
[gán ServiceAccount cho Pod một cách thủ công](#assign-to-pod), Kubernetes sẽ
gán ServiceAccount `default` của namespace đó cho Pod.

## Các trường hợp sử dụng service account của Kubernetes (Use cases for Kubernetes service accounts) {#use-cases}

Theo hướng dẫn chung, bạn có thể dùng service account để cung cấp danh tính trong
các kịch bản sau:

* Các Pod của bạn cần giao tiếp với API server của Kubernetes, ví dụ trong
  những tình huống như:
  * Cung cấp quyền truy cập chỉ đọc (read-only) đối với thông tin nhạy cảm lưu trong Secret.
  * Cấp [quyền truy cập liên namespace (cross-namespace)](#cross-namespace), chẳng hạn cho phép một
    Pod trong namespace `example` được read, list và watch các object Lease trong
    namespace `kube-node-lease`.
* Các Pod của bạn cần giao tiếp với một dịch vụ bên ngoài. Ví dụ, một
  workload Pod cần một danh tính cho một cloud API thương mại,
  và nhà cung cấp thương mại đó cho phép cấu hình một quan hệ tin cậy (trust relationship) phù hợp.
* [Xác thực với một private image registry bằng `imagePullSecret`](279-configure-service-account-vi.md#add-imagepullsecrets-to-a-service-account).
* Một dịch vụ bên ngoài cần giao tiếp với API server của Kubernetes. Ví dụ,
  xác thực với cluster như một phần của pipeline CI/CD.
* Bạn sử dụng phần mềm bảo mật của bên thứ ba trong cluster, phần mềm này dựa vào
  danh tính ServiceAccount của các Pod khác nhau để nhóm các Pod đó vào những
  ngữ cảnh (context) khác nhau.

## Cách sử dụng service account (How to use service accounts) {#how-to-use}

Để dùng một service account của Kubernetes, bạn thực hiện các bước sau:

1. Tạo một object ServiceAccount bằng một Kubernetes client
   như `kubectl` hoặc một manifest định nghĩa object đó.
1. Cấp quyền cho object ServiceAccount bằng một cơ chế phân quyền,
   chẳng hạn
   [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).
1. Gán object ServiceAccount cho Pod trong quá trình tạo Pod.

   Nếu bạn dùng danh tính này từ một dịch vụ bên ngoài,
   hãy [lấy token của ServiceAccount](#get-a-token) và dùng token đó từ dịch vụ
   bên ngoài thay thế.

Để xem hướng dẫn chi tiết, tham khảo
[Cấu hình Service Account cho Pod](279-configure-service-account-vi.md).

### Cấp quyền cho một ServiceAccount (Grant permissions to a ServiceAccount) {#grant-permissions}

Bạn có thể dùng cơ chế
[kiểm soát truy cập dựa trên vai trò (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
tích hợp sẵn của Kubernetes để cấp các quyền tối thiểu mà mỗi service account cần.
Bạn tạo một *role* (vai trò) — thứ cấp quyền truy cập — rồi *bind* (ràng buộc) role đó với
ServiceAccount của bạn. RBAC cho phép bạn định nghĩa một tập quyền tối thiểu sao cho
quyền của service account tuân theo nguyên tắc đặc quyền tối thiểu (principle of least privilege).
Các Pod dùng service account đó sẽ không nhận được nhiều quyền hơn mức cần thiết
để hoạt động đúng.

Để xem hướng dẫn chi tiết, tham khảo
[Quyền của ServiceAccount](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions).

#### Truy cập liên namespace bằng ServiceAccount (Cross-namespace access using a ServiceAccount) {#cross-namespace}

Bạn có thể dùng RBAC để cho phép các service account trong một namespace thực hiện hành động
trên các tài nguyên ở một namespace khác trong cluster. Ví dụ, xét một
kịch bản trong đó bạn có một service account và một Pod trong namespace `dev` và
bạn muốn Pod của mình thấy được các Job đang chạy trong namespace `maintenance`. Bạn có thể
tạo một object Role cấp quyền list các object Job. Sau đó,
bạn tạo một object RoleBinding trong namespace `maintenance` để ràng buộc
Role với object ServiceAccount. Lúc này, các Pod trong namespace `dev` có thể list
các object Job trong namespace `maintenance` bằng service account đó.

### Gán ServiceAccount cho một Pod (Assign a ServiceAccount to a Pod) {#assign-to-pod}

Để gán một ServiceAccount cho một Pod, bạn đặt trường `spec.serviceAccountName`
trong đặc tả (specification) của Pod. Kubernetes sau đó sẽ tự động cung cấp
credentials của ServiceAccount đó cho Pod. Từ v1.22 trở đi, Kubernetes
lấy một token ngắn hạn, **tự động xoay vòng (rotate)** bằng API `TokenRequest`
và mount token đó dưới dạng một
[projected volume](93-projected-volumes-vi.md#serviceaccounttoken).

Mặc định, Kubernetes cung cấp cho Pod
credentials của ServiceAccount đã được gán, dù đó là
ServiceAccount `default` hay một ServiceAccount tùy chỉnh mà bạn chỉ định.

Để ngăn Kubernetes tự động tiêm (inject)
credentials của một ServiceAccount được chỉ định hoặc của ServiceAccount `default`, hãy đặt
trường `automountServiceAccountToken` trong đặc tả Pod của bạn thành `false`.

Ở các phiên bản trước 1.22, Kubernetes cung cấp cho Pod một token tĩnh,
tồn tại lâu dài dưới dạng một Secret.

#### Lấy credentials của ServiceAccount một cách thủ công (Manually retrieve ServiceAccount credentials) {#get-a-token}

Nếu bạn cần credentials của một ServiceAccount để mount vào một vị trí không chuẩn,
hoặc cho một audience không phải là API server, hãy dùng một trong các
phương pháp sau:

* [TokenRequest API](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)
  (khuyến nghị): Yêu cầu một service account token ngắn hạn từ bên trong
  *mã ứng dụng* của chính bạn. Token này tự động hết hạn và có thể được xoay vòng
  khi hết hạn.
  Nếu bạn có một ứng dụng cũ (legacy) không nhận biết được Kubernetes, bạn
  có thể dùng một sidecar container trong cùng pod để lấy các token này
  và cung cấp chúng cho workload ứng dụng.
* [Token Volume Projection](279-configure-service-account-vi.md#serviceaccount-token-volume-projection)
  (cũng được khuyến nghị): Từ Kubernetes v1.20 trở đi, dùng đặc tả Pod để
  yêu cầu kubelet thêm service account token vào Pod dưới dạng một
  *projected volume*. Các projected token tự động hết hạn, và kubelet
  xoay vòng token trước khi nó hết hạn.
* [Service Account Token Secrets](279-configure-service-account-vi.md#manually-create-an-api-token-for-a-serviceaccount)
  (không khuyến nghị): Bạn có thể mount các service account token dưới dạng Kubernetes
  Secret vào Pod. Những token này không hết hạn và không được xoay vòng. Ở các phiên bản trước v1.24, một token vĩnh viễn được tự động tạo cho mỗi service account.
  Phương pháp này không còn được khuyến nghị nữa, đặc biệt ở quy mô lớn, do các rủi ro gắn liền
  với credentials tĩnh, tồn tại lâu dài. [Feature gate LegacyServiceAccountTokenNoAutoGeneration](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates-removed)
  (được bật mặc định từ Kubernetes v1.24 đến v1.26) đã ngăn Kubernetes tự động tạo các token này cho
  ServiceAccount. Feature gate này bị gỡ bỏ ở v1.27 vì đã được nâng lên trạng thái GA; bạn vẫn có thể tạo thủ công các service account token không có thời hạn, nhưng cần cân nhắc các hệ quả về bảo mật.

#### Giới hạn audience trên node đối với service account token (Node audience restriction for service account tokens) {#node-audience-restriction}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [beta]` (được bật mặc định: true)

Khi [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `ServiceAccountNodeAudienceRestriction`
được bật, admission plugin [NodeRestriction](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers#noderestriction)
sẽ giới hạn những audience mà một kubelet có thể yêu cầu khi tạo service
account token qua API `TokenRequest`. Mặc định, kubelet chỉ có thể yêu cầu
token cho các audience đã được tham chiếu bởi các pod trên node đó (thông qua projected service
account token volume hoặc các yêu cầu token của CSI driver). Quản trị viên có thể cấp cho
kubelet quyền truy cập các audience bổ sung bằng các quy tắc RBAC với
verb `request-serviceaccounts-token-audience`.

Giới hạn này chỉ áp dụng cho kubelet (danh tính node) và không ảnh hưởng đến các
bên gọi khác của API `TokenRequest`. Để biết chi tiết và các ví dụ RBAC,
xem [Giới hạn audience của service account token](https://kubernetes.io/docs/reference/access-authn-authz/node/#service-account-token-audience-restriction).

> **Ghi chú:**
>
> Với các ứng dụng chạy bên ngoài cluster Kubernetes của bạn, bạn có thể đang cân nhắc
> tạo một ServiceAccount token tồn tại lâu dài được lưu trong một Secret. Cách này cho phép xác thực, nhưng dự án Kubernetes khuyến nghị bạn tránh cách tiếp cận này.
> Bearer token tồn tại lâu dài tiềm ẩn rủi ro bảo mật vì một khi bị lộ, token
> có thể bị lạm dụng. Thay vào đó, hãy cân nhắc một phương án thay thế. Ví dụ, ứng dụng
> bên ngoài của bạn có thể xác thực bằng một private key được bảo vệ tốt `và` một certificate,
> hoặc bằng một cơ chế tùy chỉnh chẳng hạn như một [authentication webhook](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication) do chính bạn triển khai.
>
> Bạn cũng có thể dùng TokenRequest để lấy các token ngắn hạn cho ứng dụng bên ngoài của mình.

### Hạn chế truy cập Secret (đã lỗi thời) (Restricting access to Secrets (deprecated)) {#enforce-mountable-secrets}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.32 [deprecated]`

> **Ghi chú:**
>
> `kubernetes.io/enforce-mountable-secrets` đã lỗi thời (deprecated) kể từ Kubernetes v1.32. Hãy dùng các namespace riêng biệt để cô lập quyền truy cập vào các secret được mount.

Kubernetes cung cấp một annotation tên là `kubernetes.io/enforce-mountable-secrets`
mà bạn có thể thêm vào các ServiceAccount của mình. Khi annotation này được áp dụng,
các secret của ServiceAccount chỉ có thể được mount trên những loại tài nguyên được chỉ định,
giúp tăng cường thế trận bảo mật (security posture) cho cluster của bạn.

Bạn có thể thêm annotation vào một ServiceAccount bằng manifest:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    kubernetes.io/enforce-mountable-secrets: "true"
  name: my-serviceaccount
  namespace: my-namespace
```
Khi annotation này được đặt là "true", control plane của Kubernetes đảm bảo rằng
các Secret của ServiceAccount này phải tuân theo một số hạn chế mount nhất định.

1. Tên của mỗi Secret được mount dưới dạng volume trong một Pod phải xuất hiện trong trường `secrets` của
   ServiceAccount thuộc Pod đó.
1. Tên của mỗi Secret được tham chiếu qua `envFrom` trong một Pod cũng phải xuất hiện trong trường `secrets`
   của ServiceAccount thuộc Pod đó.
1. Tên của mỗi Secret được tham chiếu qua `imagePullSecrets` trong một Pod cũng phải xuất hiện trong trường `secrets`
   của ServiceAccount thuộc Pod đó.

Bằng cách hiểu và thực thi các hạn chế này, quản trị viên cluster có thể duy trì một hồ sơ bảo mật chặt chẽ hơn và đảm bảo rằng các secret chỉ được truy cập bởi những tài nguyên phù hợp.

## Xác thực credentials của service account (Authenticating service account credentials) {#authenticating-credentials}

ServiceAccount dùng JSON Web Token (JWT)
đã được ký để xác thực với API server của Kubernetes, cũng như với bất kỳ hệ thống nào khác
có tồn tại quan hệ tin cậy. Tùy vào cách token được phát hành
(hoặc có giới hạn thời gian qua `TokenRequest`, hoặc qua cơ chế cũ (legacy) với
một Secret), một ServiceAccount token cũng có thể có thời điểm hết hạn, một audience,
và một thời điểm mà từ đó token *bắt đầu* có hiệu lực. Khi một client đang
hoạt động dưới danh nghĩa một ServiceAccount cố gắng giao tiếp với API server của Kubernetes,
client đó gắn header `Authorization: Bearer <token>` vào HTTP
request. API server kiểm tra tính hợp lệ của bearer token đó như sau:

1. Kiểm tra chữ ký của token.
1. Kiểm tra token đã hết hạn hay chưa.
1. Kiểm tra các tham chiếu object trong claim của token có còn hợp lệ hay không.
1. Kiểm tra token có đang còn hiệu lực hay không.
1. Kiểm tra các claim về audience.

API TokenRequest tạo ra các _bound token_ (token bị ràng buộc) cho một ServiceAccount. Sự
ràng buộc này gắn với vòng đời của client — chẳng hạn một Pod — đang hoạt động
dưới danh nghĩa ServiceAccount đó. Xem [Token Volume Projection](279-configure-service-account-vi.md#serviceaccount-token-volume-projection)
để có ví dụ về schema và payload JWT của một bound service account token của pod.

Với các token phát hành qua API `TokenRequest`, API server còn kiểm tra rằng
tham chiếu object cụ thể đang dùng ServiceAccount vẫn tồn tại,
so khớp bằng ID duy nhất (unique ID) của
object đó. Với các token kiểu cũ (legacy) được mount dưới dạng Secret trong Pod, API server
kiểm tra token dựa trên Secret.

Để biết thêm thông tin về quy trình xác thực, tham khảo
[Xác thực (Authentication)](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens).

### Xác thực credentials của service account trong mã của riêng bạn (Authenticating service account credentials in your own code) {#authenticating-in-code}

Nếu bạn có các dịch vụ của riêng mình cần xác minh (validate) credentials của service
account Kubernetes, bạn có thể dùng các phương pháp sau:

* [TokenReview API](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-review-v1/)
  (khuyến nghị)
* OIDC discovery

Dự án Kubernetes khuyến nghị bạn dùng API TokenReview, vì
phương pháp này vô hiệu hóa những token bị ràng buộc với các object API như Secret,
ServiceAccount, Pod hoặc Node khi các object đó bị xóa. Ví dụ, nếu bạn
xóa Pod chứa một projected ServiceAccount token, cluster sẽ
vô hiệu hóa token đó ngay lập tức và một TokenReview sẽ thất bại ngay.
Ngược lại, nếu bạn dùng xác minh OIDC, các client của bạn vẫn tiếp tục coi token
là hợp lệ cho đến khi token chạm mốc thời gian hết hạn của nó.

Ứng dụng của bạn luôn luôn nên định nghĩa audience mà nó chấp nhận, và nên
kiểm tra rằng các audience của token khớp với các audience mà ứng dụng
mong đợi. Điều này giúp thu hẹp phạm vi của token để nó chỉ có thể được
dùng trong ứng dụng của bạn và không ở bất kỳ nơi nào khác.

## Các phương án thay thế (Alternatives)

* Tự phát hành token bằng một cơ chế khác, rồi dùng
  [Webhook Token Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)
  để xác minh các bearer token bằng dịch vụ xác minh của riêng bạn.
* Tự cung cấp danh tính cho các Pod.
  * [Dùng plugin SPIFFE CSI driver để cung cấp SPIFFE SVID dưới dạng cặp certificate X.509 cho Pod](https://cert-manager.io/docs/projects/csi-driver-spiffe/).
    > **Ghi chú:** Mục này liên kết tới một dự án của bên thứ ba, không phải là một phần của Kubernetes. Các tác giả của dự án Kubernetes không chịu trách nhiệm về dự án bên thứ ba đó.
  * [Dùng một service mesh như Istio để cung cấp certificate cho Pod](https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/).
* Xác thực từ bên ngoài cluster với API server mà không dùng service account token:
  * [Cấu hình API server để chấp nhận token OpenID Connect (OIDC) từ nhà cung cấp danh tính (identity provider) của bạn](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens).
  * Dùng các service account hoặc user account được tạo bởi một dịch vụ quản lý danh tính
    và truy cập (Identity and Access Management — IAM) bên ngoài, chẳng hạn từ một nhà cung cấp cloud, để
    xác thực với cluster của bạn.
  * [Dùng API CertificateSigningRequest với client certificate](399-managing-tls-in-a-cluster-vi.md).
* [Cấu hình kubelet để lấy credentials từ một image registry](225-kubelet-credential-provider-vi.md).
* Dùng một Device Plugin để truy cập một Trusted Platform Module (TPM) ảo, từ đó
  cho phép xác thực bằng một private key.

## Tiếp theo (What's next)

* Tìm hiểu cách [quản lý các ServiceAccount với vai trò quản trị viên cluster](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/).
* Tìm hiểu cách [gán một ServiceAccount cho một Pod](279-configure-service-account-vi.md).
* Đọc [tài liệu tham khảo API ServiceAccount](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/service-account-v1/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Ở [Lab 1a phần B8](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) bạn xóa ServiceAccount
   `default` trong một namespace rồi thấy nó xuất hiện lại. Bài này giải thích hành vi đó bằng
   câu nào? Và nếu bạn tạo một namespace mới trên `lab-k8s-master`, nó có sẵn ServiceAccount nào?
2. ServiceAccount và tài khoản người dùng — cái nào là object trong Kubernetes API? Bạn có thể
   liệt kê toàn bộ người dùng là con người của cluster bằng `kubectl` không, vì sao?
3. Bài [24](24-control-plane-node-communication-vi.md) nói Pod gọi API server "bằng cách tận
   dụng service account". Cụ thể Kubernetes đưa cái gì vào Pod, ở dạng nào, và cách làm từ
   v1.22 khác các bản trước ở điểm nào?
4. Bạn tạo một Pod mà không đặt `spec.serviceAccountName`. Pod đó gọi API server dưới danh tính
   nào, và danh tính đó có sẵn quyền gì? Muốn Pod hoàn toàn không nhận credential thì đặt gì?
5. Vì sao dự án Kubernetes khuyến nghị dùng TokenReview API thay vì xác minh bằng OIDC
   discovery khi dịch vụ của bạn cần kiểm tra một service account token?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Bài nói: nếu bạn xóa object ServiceAccount `default` trong một namespace, **control plane sẽ
   thay thế nó bằng một object mới**. Đó chính là vòng lặp điều hòa bạn quan sát ở B8. Và
   **mỗi namespace khi được tạo đều có một ServiceAccount `default`** — namespace mới cũng vậy,
   bạn không phải tạo gì thêm.
2. **Chỉ ServiceAccount** là object trong Kubernetes API. **Không liệt kê được người dùng bằng
   `kubectl`**: mặc định tài khoản người dùng **không tồn tại trong API server**, API server
   coi danh tính người dùng là **dữ liệu mờ (opaque)** và không lưu username hay thông tin nào
   khác về người dùng trong API của mình.
3. Kubernetes tự động tiêm **certificate gốc công khai của cluster và một bearer token hợp lệ**
   vào Pod. Từ v1.22, token đó là **token ngắn hạn, tự động xoay vòng**, lấy qua API
   `TokenRequest` và **mount dưới dạng projected volume**. Trước v1.22, Pod nhận một **token
   tĩnh, tồn tại lâu dài, lưu dưới dạng Secret** — không hết hạn, không xoay vòng.
4. Pod chạy dưới **ServiceAccount `default` của namespace đó**. Danh tính này **mặc định không
   có quyền hạn nào** ngoài các quyền API discovery mặc định mà Kubernetes cấp cho mọi
   principal đã xác thực khi RBAC được bật. Muốn không nhận credential: đặt
   **`automountServiceAccountToken: false`** trong đặc tả Pod.
5. Vì **TokenReview vô hiệu hóa ngay các token bị ràng buộc với object API đã bị xóa.** Xóa Pod
   đang giữ một projected token thì cluster vô hiệu token đó lập tức và TokenReview thất bại
   ngay. Ngược lại, với xác minh **OIDC**, client vẫn tiếp tục coi token là hợp lệ **cho đến khi
   token chạm mốc hết hạn** — tức là còn một khoảng thời gian token đã bị thu hồi vẫn dùng được.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
