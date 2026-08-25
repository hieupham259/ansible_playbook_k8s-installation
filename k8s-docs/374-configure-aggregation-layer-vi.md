# Cấu hình tầng tổng hợp (Configure the Aggregation Layer)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/>
>
> Việc cấu hình tầng tổng hợp (aggregation layer) cho phép mở rộng Kubernetes apiserver bằng các
> API bổ sung không thuộc nhóm API lõi của Kubernetes.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 28 — Mở rộng Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes),
bài 2/11 · Phần II không có lab riêng: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ghi ở cuối mục giai đoạn
28. Phần kiểm chứng được ngay: **đọc lại cấu hình tầng tổng hợp mà cluster lab đã có sẵn** — cụm cờ
`--requestheader-*` và `--proxy-client-*` của kube-apiserver trên `lab-k8s-master`, ConfigMap
`extension-apiserver-authentication` trong `kube-system`, và object `APIService` của metrics-server.

**Đây đúng là phần [Lab 14](labs/LAB-14-CRD-VA-OPERATOR.md) đã ghi nợ.** Bảng "không kiểm chứng được
trong lab này" ở mục 1.1 của Lab 14 nói rõ lý do: dựng một extension API server của riêng bạn thì
**phải viết và build binary cùng image, rồi cấu hình cụm cờ `--requestheader-*` cho
kube-apiserver** — nên lab đẩy sang đúng bài này và bài
[380](380-setup-extension-api-server-vi.md). Ở Lab 14 bạn mới làm phần **đọc được**: metrics-server
là APIService duy nhất trên cluster lab có `.spec.service`, tức API group được proxy sang một server
khác. Bài này giải thích **vì sao đường proxy đó chạy được** và cái giá cấu hình của nó. Đọc kỹ
luồng xác thực; phần đăng ký `APIService` để dành cho bài 380.

**Phải hiểu ở lần đọc này:**

- Khác biệt gốc so với CRD, bài nêu ngay đầu mục *Luồng xác thực*: aggregated API có **một server
  khác** tham gia bên cạnh kube-apiserver, và **hai chiều đều phải xác thực được nhau**. Mọi rắc rối
  cấu hình của bài đều sinh ra từ chỗ này; CRD không có vấn đề đó vì không có server thứ hai.
- Năm bước của luồng: kube-apiserver **xác thực và phân quyền người dùng** → **proxy** yêu cầu sang
  extension API server → extension API server **xác thực rằng yêu cầu đến từ kube-apiserver** →
  **phân quyền cho người dùng ban đầu** → **thực thi**. Chú ý bước 4: bên kia phân quyền cho *người
  dùng gốc*, không phải cho kube-apiserver.
- Hai câu hỏi mà kube-apiserver phải trả lời khi proxy, và hai cụm cờ tương ứng: **tự xác thực bằng
  client certificate** (`--proxy-client-cert-file`, `--proxy-client-key-file`, kèm
  `--requestheader-client-ca-file` và `--requestheader-allowed-names` để bên kia kiểm), và **báo
  danh tính người dùng gốc qua http header** (`--requestheader-username-headers`,
  `--requestheader-group-headers`, `--requestheader-extra-headers-prefix`).
- ConfigMap `extension-apiserver-authentication` trong namespace `kube-system` là **kênh truyền cấu
  hình giữa hai server**: kube-apiserver ghi vào đó CA certificate, danh sách CN được phép và tên
  các header; extension API server đọc lại để kiểm yêu cầu. Muốn đọc, nó cần role
  `extension-apiserver-authentication-reader`; muốn gửi `SubjectAccessReview` để phân quyền, service
  account của nó cần ClusterRole `system:auth-delegator`.
- Bẫy lớn nhất của bài, mục [*Tái sử dụng CA và xung đột*](#ca-reusage-and-conflicts):
  `--client-ca-file` và `--requestheader-client-ca-file` hoạt động **độc lập**, và yêu cầu được kiểm
  với `--requestheader-client-ca-file` **trước**. Dùng **chung một CA** cho cả hai thì client thường
  lẽ ra hợp lệ sẽ **bị từ chối**, vì CN của họ không nằm trong `--requestheader-allowed-names` —
  kubelet, các thành phần control plane và cả người dùng cuối đều có thể mất đường xác thực.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Đăng ký các đối tượng APIService* và *Liên hệ với extension API server* — ví dụ YAML với `groupPriorityMinimum`, `versionPriority`, `caBundle`, khối `service` và `port` mặc định 443 | là **bước cuối**, chỉ có nghĩa khi đã có một server thứ hai đang chạy để trỏ tới | bài [380](380-setup-extension-api-server-vi.md) của chính giai đoạn 28 |
| Cờ `--enable-aggregator-routing=true` | chỉ cần cho topology **không** chạy kube-proxy trên host đang chạy API server | khi dựng extension API server thật ở bài [380](380-setup-extension-api-server-vi.md) |
| Mục *Trước khi bạn bắt đầu* — minikube và danh sách sân chơi Kubernetes | bạn đã có cluster ba VM từ Lab 00; lộ trình không dùng minikube hay cluster dùng chung | [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) |
| Sơ đồ swimlane `aggregation-api-auth-flow` và câu về mã nguồn sơ đồ | là hình minh họa lại đúng năm bước đã đọc bằng chữ, không thêm thông tin | không cần |

---

Việc cấu hình [tầng tổng hợp (aggregation layer)](180-apiserver-aggregation-vi.md) cho phép
Kubernetes apiserver được mở rộng bằng các API bổ sung, vốn không phải là một phần của các API lõi
của Kubernetes.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp với
cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng vai trò
control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân chơi
(playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

> **Ghi chú:**
>
> Có một vài yêu cầu thiết lập để tầng tổng hợp hoạt động được trong môi trường của bạn, nhằm hỗ trợ
> xác thực TLS hai chiều (mutual TLS auth) giữa proxy và các extension API server. Kubernetes và
> kube-apiserver có nhiều CA khác nhau, vì vậy hãy đảm bảo rằng proxy được ký bởi CA của tầng tổng
> hợp chứ không phải bởi một CA nào khác, chẳng hạn CA chung của Kubernetes.

> **Thận trọng:**
>
> Việc dùng lại cùng một CA cho nhiều loại client khác nhau có thể ảnh hưởng tiêu cực tới khả năng
> hoạt động của cluster. Để biết thêm chi tiết, xem
> [Tái sử dụng CA và xung đột](#ca-reusage-and-conflicts).

## Luồng xác thực (Authentication Flow)

Không giống Custom Resource Definitions (CRD), API tổng hợp (Aggregation API) có sự tham gia của một
server khác — extension API server của bạn — bên cạnh Kubernetes apiserver tiêu chuẩn. Kubernetes
apiserver sẽ cần giao tiếp với extension API server của bạn, và extension API server của bạn cũng sẽ
cần giao tiếp với Kubernetes apiserver. Để việc giao tiếp này được bảo mật, Kubernetes apiserver sử
dụng certificate x509 để tự xác thực với extension API server.

Mục này mô tả cách các luồng xác thực (authentication) và phân quyền (authorization) hoạt động, cùng
cách cấu hình chúng.

Luồng ở mức tổng quan như sau:

1. Kubernetes apiserver: xác thực người dùng gửi yêu cầu và phân quyền cho quyền truy cập của họ tới
   đường dẫn API được yêu cầu.
2. Kubernetes apiserver: chuyển tiếp (proxy) yêu cầu tới extension API server.
3. Extension API server: xác thực yêu cầu đến từ Kubernetes apiserver.
4. Extension API server: phân quyền cho yêu cầu của người dùng ban đầu.
5. Extension API server: thực thi.

Phần còn lại của mục này mô tả chi tiết các bước trên.

Luồng này được minh họa trong sơ đồ dưới đây.

![aggregation auth flows](https://kubernetes.io/images/docs/aggregation-api-auth-flow.png)

Mã nguồn của sơ đồ swimlane ở trên có thể tìm thấy trong mã nguồn của tài liệu này.

### Xác thực và phân quyền tại Kubernetes apiserver (Kubernetes Apiserver Authentication and Authorization)

Một yêu cầu tới đường dẫn API do extension API server phục vụ bắt đầu giống hệt mọi yêu cầu API
khác: giao tiếp với Kubernetes apiserver. Đường dẫn này đã được extension API server đăng ký trước
đó với Kubernetes apiserver.

Người dùng giao tiếp với Kubernetes apiserver, yêu cầu truy cập vào đường dẫn đó. Kubernetes
apiserver sử dụng cơ chế xác thực và phân quyền tiêu chuẩn đã được cấu hình trên Kubernetes apiserver
để xác thực người dùng và phân quyền truy cập vào đường dẫn cụ thể đó.

Để có cái nhìn tổng quan về việc xác thực vào một cluster Kubernetes, xem
["Xác thực vào cluster"](https://kubernetes.io/docs/reference/access-authn-authz/authentication/).
Để có cái nhìn tổng quan về việc phân quyền truy cập tài nguyên của cluster Kubernetes, xem
["Tổng quan về phân quyền"](https://kubernetes.io/docs/reference/access-authn-authz/authorization/).

Mọi thứ cho tới thời điểm này đều là các yêu cầu API, xác thực và phân quyền tiêu chuẩn của
Kubernetes.

Kubernetes apiserver bây giờ đã sẵn sàng gửi yêu cầu tới extension API server.

### Kubernetes apiserver chuyển tiếp yêu cầu (Kubernetes Apiserver Proxies the Request)

Kubernetes apiserver bây giờ sẽ gửi, hay chuyển tiếp (proxy), yêu cầu tới extension API server đã
đăng ký xử lý yêu cầu đó. Để làm được điều này, nó cần biết một số thứ:

1. Kubernetes apiserver phải xác thực với extension API server như thế nào, để cho extension API
   server biết rằng yêu cầu đến qua mạng thực sự xuất phát từ một Kubernetes apiserver hợp lệ?
2. Kubernetes apiserver phải thông báo cho extension API server về username và group mà yêu cầu ban
   đầu đã được xác thực như thế nào?

Để đáp ứng hai điều này, bạn phải cấu hình Kubernetes apiserver bằng một số cờ (flag).

#### Xác thực client của Kubernetes apiserver (Kubernetes Apiserver Client Authentication)

Kubernetes apiserver kết nối tới extension API server qua TLS, tự xác thực bằng một client
certificate. Bạn phải cung cấp những thứ sau cho Kubernetes apiserver khi khởi động, thông qua các
cờ tương ứng:

* file private key qua `--proxy-client-key-file`
* file client certificate đã được ký qua `--proxy-client-cert-file`
* certificate của CA đã ký file client certificate đó qua `--requestheader-client-ca-file`
* các giá trị Common Name (CN) hợp lệ trong client certificate đã ký qua
  `--requestheader-allowed-names`

Kubernetes apiserver sẽ dùng các file được chỉ định bởi `--proxy-client-*-file` để xác thực với
extension API server. Để yêu cầu được một extension API server tuân thủ chuẩn coi là hợp lệ, các
điều kiện sau phải được thỏa mãn:

1. Kết nối phải được thực hiện bằng một client certificate được ký bởi CA có certificate nằm trong
   `--requestheader-client-ca-file`.
2. Kết nối phải được thực hiện bằng một client certificate có CN nằm trong danh sách được liệt kê
   tại `--requestheader-allowed-names`.

> **Ghi chú:**
>
> Bạn có thể đặt tùy chọn này thành rỗng, dạng `--requestheader-allowed-names=""`. Điều này báo cho
> extension API server biết rằng _bất kỳ_ CN nào cũng được chấp nhận.

Khi được khởi động với các tùy chọn này, Kubernetes apiserver sẽ:

1. Dùng chúng để xác thực với extension API server.
2. Tạo một configmap trong namespace `kube-system` có tên `extension-apiserver-authentication`, nơi
   nó đặt certificate của CA và danh sách CN được phép. Đến lượt mình, các extension API server có
   thể đọc những thông tin này để kiểm tra tính hợp lệ của các yêu cầu.

Lưu ý rằng Kubernetes apiserver dùng cùng một client certificate để xác thực với _tất cả_ các
extension API server. Nó không tạo một client certificate riêng cho từng extension API server, mà
chỉ dùng một certificate duy nhất để xác thực với danh nghĩa Kubernetes apiserver. Chính certificate
này được tái sử dụng cho mọi yêu cầu gửi tới extension API server.

#### Username và group của yêu cầu ban đầu (Original Request Username and Group)

Khi Kubernetes apiserver chuyển tiếp yêu cầu tới extension API server, nó thông báo cho extension
API server biết username và group mà yêu cầu ban đầu đã xác thực thành công. Nó cung cấp các thông
tin này trong các http header của yêu cầu được chuyển tiếp. Bạn phải cho Kubernetes apiserver biết
tên của các header sẽ được dùng.

* header dùng để lưu username, qua `--requestheader-username-headers`
* header dùng để lưu group, qua `--requestheader-group-headers`
* tiền tố (prefix) được thêm vào tất cả các header phụ (extra), qua
  `--requestheader-extra-headers-prefix`

Tên các header này cũng được đặt trong configmap `extension-apiserver-authentication`, để các
extension API server có thể đọc và sử dụng.

### Extension API server xác thực yêu cầu (Extension Apiserver Authenticates the Request)

Extension API server, khi nhận được một yêu cầu được chuyển tiếp từ Kubernetes apiserver, phải kiểm
tra rằng yêu cầu đó thực sự đến từ một proxy xác thực (authenticating proxy) hợp lệ — vai trò mà
Kubernetes apiserver đang đảm nhiệm. Extension API server kiểm tra điều đó bằng cách:

1. Đọc những thông tin sau từ configmap trong `kube-system`, như đã mô tả ở trên:
    * Client CA certificate
    * Danh sách tên được phép (CN)
    * Tên các header chứa username, group và thông tin phụ (extra info)
2. Kiểm tra rằng kết nối TLS đã được xác thực bằng một client certificate mà:
    * Được ký bởi CA có certificate khớp với CA certificate đã đọc được.
    * Có CN nằm trong danh sách CN được phép, trừ khi danh sách đó rỗng — trong trường hợp đó mọi CN
      đều được chấp nhận.
    * Trích xuất username và group từ các header tương ứng.

Nếu các bước trên đều vượt qua, thì yêu cầu là một yêu cầu chuyển tiếp hợp lệ đến từ một proxy xác
thực chính danh, trong trường hợp này là Kubernetes apiserver.

Lưu ý rằng việc cung cấp những cơ chế trên là trách nhiệm của bản hiện thực extension API server.
Nhiều bản hiện thực làm điều đó một cách mặc định, tận dụng gói `k8s.io/apiserver/`. Một số bản khác
có thể cung cấp các tùy chọn để ghi đè hành vi này thông qua tham số dòng lệnh.

Để có quyền đọc configmap đó, extension API server cần role phù hợp. Có một role mặc định tên là
`extension-apiserver-authentication-reader` trong namespace `kube-system` có thể được gán cho nó.

### Extension API server phân quyền cho yêu cầu (Extension Apiserver Authorizes the Request)

Extension API server bây giờ có thể kiểm tra xem user/group lấy được từ các header có được phép thực
thi yêu cầu đã cho hay không. Nó làm điều đó bằng cách gửi một yêu cầu
[SubjectAccessReview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) tiêu
chuẩn tới Kubernetes apiserver.

Để bản thân extension API server được phép gửi yêu cầu `SubjectAccessReview` tới Kubernetes
apiserver, nó cần có quyền đúng. Kubernetes có sẵn một `ClusterRole` mặc định tên là
`system:auth-delegator` với các quyền phù hợp. Role này có thể được gán cho service account của
extension API server.

### Extension API server thực thi (Extension Apiserver Executes)

Nếu `SubjectAccessReview` vượt qua, extension API server sẽ thực thi yêu cầu.

## Bật các cờ của Kubernetes apiserver (Enable Kubernetes Apiserver flags)

Bật tầng tổng hợp thông qua các cờ `kube-apiserver` sau. Có thể nhà cung cấp của bạn đã xử lý sẵn
những cờ này.

```
--requestheader-client-ca-file=<path to aggregator CA cert>
--requestheader-allowed-names=front-proxy-client
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User
--proxy-client-cert-file=<path to aggregator proxy cert>
--proxy-client-key-file=<path to aggregator proxy key>
```

### Tái sử dụng CA và xung đột (CA Reusage and Conflicts) {#ca-reusage-and-conflicts}

Kubernetes apiserver có hai tùy chọn client CA:

* `--client-ca-file`
* `--requestheader-client-ca-file`

Mỗi tùy chọn hoạt động độc lập và có thể xung đột với nhau nếu không được dùng đúng cách.

* `--client-ca-file`: Khi một yêu cầu đến Kubernetes apiserver, nếu tùy chọn này được bật, Kubernetes
  apiserver sẽ kiểm tra certificate của yêu cầu. Nếu certificate đó được ký bởi một trong các CA
  certificate nằm trong file được `--client-ca-file` trỏ tới, thì yêu cầu được coi là hợp lệ, user là
  giá trị của common name `CN=`, còn group là giá trị của organization `O=`. Xem
  [tài liệu về xác thực TLS](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#x509-client-certificates).
* `--requestheader-client-ca-file`: Khi một yêu cầu đến Kubernetes apiserver, nếu tùy chọn này được
  bật, Kubernetes apiserver sẽ kiểm tra certificate của yêu cầu. Nếu certificate đó được ký bởi một
  trong các CA certificate nằm trong file được `--requestheader-client-ca-file` trỏ tới, thì yêu cầu
  được coi là có khả năng hợp lệ. Kubernetes apiserver sau đó kiểm tra xem common name `CN=` có nằm
  trong danh sách tên do `--requestheader-allowed-names` cung cấp hay không. Nếu tên đó được phép,
  yêu cầu được chấp nhận; nếu không, yêu cầu bị từ chối.

Nếu _cả hai_ `--client-ca-file` và `--requestheader-client-ca-file` cùng được cung cấp, thì yêu cầu
sẽ được kiểm tra với CA của `--requestheader-client-ca-file` trước, rồi mới tới `--client-ca-file`.
Thông thường, người ta dùng các CA khác nhau — có thể là root CA hoặc intermediate CA — cho mỗi tùy
chọn này: các yêu cầu client thông thường khớp với `--client-ca-file`, còn các yêu cầu tổng hợp
(aggregation) khớp với `--requestheader-client-ca-file`. Tuy nhiên, nếu cả hai dùng _cùng một_ CA,
thì những yêu cầu client lẽ ra phải vượt qua nhờ `--client-ca-file` sẽ thất bại, bởi vì CA đó khớp
với CA trong `--requestheader-client-ca-file`, nhưng common name `CN=` lại **không** khớp với bất kỳ
common name nào được chấp nhận trong `--requestheader-allowed-names`. Điều này có thể khiến các
kubelet và các thành phần control plane khác, cũng như người dùng cuối, không thể xác thực được với
Kubernetes apiserver.

Vì lý do đó, hãy dùng các CA cert khác nhau cho tùy chọn `--client-ca-file` — để phân quyền cho các
thành phần control plane và người dùng cuối — và cho tùy chọn `--requestheader-client-ca-file` — để
phân quyền cho các yêu cầu tới aggregation apiserver.

> **Cảnh báo:**
>
> **Không** tái sử dụng một CA đang được dùng trong một ngữ cảnh khác, trừ khi bạn hiểu rõ các rủi ro
> và các cơ chế bảo vệ việc sử dụng CA đó.

Nếu bạn không chạy kube-proxy trên host đang chạy API server, thì bạn phải đảm bảo hệ thống được bật
cờ `kube-apiserver` sau:

```
--enable-aggregator-routing=true
```

### Đăng ký các đối tượng APIService (Register APIService objects)

Bạn có thể cấu hình động những yêu cầu client nào sẽ được chuyển tiếp tới extension API server. Dưới
đây là một ví dụ đăng ký:

```yaml

apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: <name of the registration object>
spec:
  group: <API group name this extension apiserver hosts>
  version: <API version this extension apiserver hosts>
  groupPriorityMinimum: <priority this APIService for this group, see API documentation>
  versionPriority: <prioritizes ordering of this version within a group, see API documentation>
  service:
    namespace: <namespace of the extension apiserver service>
    name: <name of the extension apiserver service>
  caBundle: <pem encoded ca cert that signs the server cert used by the webhook>
```

Tên của một đối tượng APIService phải là một
[tên dùng làm phân đoạn đường dẫn](https://kubernetes.io/docs/concepts/overview/working-with-objects/names#path-segment-names)
hợp lệ.

#### Liên hệ với extension API server (Contacting the extension apiserver)

Một khi Kubernetes apiserver đã xác định rằng một yêu cầu cần được gửi tới extension API server, nó
cần biết cách liên hệ với server đó.

Khối `service` là một tham chiếu tới Service của extension API server. Namespace và name của Service
là bắt buộc. Port là tùy chọn và mặc định là 443.

Dưới đây là ví dụ về một extension API server được cấu hình để được gọi ở port "1234", và để kiểm
tra kết nối TLS dựa trên ServerName `my-service-name.my-service-namespace.svc` bằng một CA bundle
tùy chỉnh.

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
...
spec:
  ...
  service:
    namespace: my-service-namespace
    name: my-service-name
    port: 1234
  caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle>...tLS0K"
...
```

## Tiếp theo (What's next)

* [Thiết lập một extension api-server](380-setup-extension-api-server-vi.md)
  để làm việc với tầng tổng hợp.
* Để có cái nhìn tổng quan ở mức cao, xem
  [Mở rộng Kubernetes API bằng tầng tổng hợp](180-apiserver-aggregation-vi.md).
* Tìm hiểu cách
  [Mở rộng Kubernetes API bằng Custom Resource Definitions](378-custom-resource-definitions-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 28:

1. Bài mở đầu mục *Luồng xác thực* bằng một so sánh với CRD. Khác biệt gốc giữa hai cách mở rộng API
   là gì, và vì sao chính khác biệt đó sinh ra toàn bộ đống cấu hình certificate trong bài?
2. Khi kube-apiserver chuyển tiếp một yêu cầu sang extension API server, nó phải trả lời hai câu
   hỏi. Đó là hai câu nào, mỗi câu được trả lời bằng cơ chế gì, và cụm cờ nào phục vụ cho từng cơ
   chế?
3. **Câu bẫy.** Cluster của bạn đã có sẵn một CA dùng cho `--client-ca-file`. Để đỡ phải quản lý hai
   bộ certificate, bạn trỏ luôn `--requestheader-client-ca-file` vào cùng CA đó. Chuyện gì hỏng, và
   hỏng với ai?
4. Trên `lab-k8s-master`, metrics-server phục vụ `v1beta1.metrics.k8s.io` qua tầng tổng hợp — bạn đã
   đọc APIService đó ở Lab 14. Theo bài, nó phải đọc được một ConfigMap trong `kube-system` thì mới
   kiểm được yêu cầu đến từ kube-apiserver: ConfigMap đó tên gì, chứa những gì, và bản thân
   metrics-server cần hai quyền dựng sẵn nào?
5. Cluster có ba extension API server khác nhau. kube-apiserver phải chuẩn bị mấy client certificate,
   và điều đó nói lên gì về ý nghĩa của `--requestheader-allowed-names`?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **CRD không có server thứ hai, aggregated API thì có.** Với CRD, kube-apiserver tự phục vụ kiểu
   tài nguyên mới. Với aggregated API, **extension API server của bạn** tham gia bên cạnh
   kube-apiserver tiêu chuẩn, và **cả hai chiều đều phải nói chuyện được**: kube-apiserver cần gọi
   sang extension API server, còn extension API server cần gọi ngược lại kube-apiserver. Muốn đường
   đó bảo mật thì kube-apiserver phải **tự xác thực bằng certificate x509**, và từ đó mọi thứ về CA,
   CN và header trong bài mới xuất hiện.
2. Câu một: **kube-apiserver xác thực với extension API server như thế nào**, để bên kia tin yêu cầu
   thực sự đến từ một kube-apiserver hợp lệ. Trả lời bằng **client certificate qua TLS**, khai bằng
   `--proxy-client-cert-file` và `--proxy-client-key-file`; phía kiểm thì dựa vào
   `--requestheader-client-ca-file` (CA đã ký cert đó) và `--requestheader-allowed-names` (danh sách
   CN được chấp nhận). Câu hai: **báo cho extension API server biết username và group mà yêu cầu ban
   đầu đã xác thực thành công**. Trả lời bằng **http header của yêu cầu được chuyển tiếp**, tên
   header khai bằng `--requestheader-username-headers`, `--requestheader-group-headers` và
   `--requestheader-extra-headers-prefix`.
3. **Hỏng phần xác thực client thông thường, và hỏng với gần như tất cả mọi người.** Bài giải thích
   theo đúng thứ tự kiểm: yêu cầu được đối chiếu với CA của `--requestheader-client-ca-file`
   **trước**, rồi mới tới `--client-ca-file`. Khi hai cờ dùng chung một CA, certificate của client
   thường **khớp ngay ở vòng đầu**, nên kube-apiserver chuyển sang kiểm tiếp `CN=` với
   `--requestheader-allowed-names` — mà CN của họ đâu có nằm trong danh sách đó, nên **bị từ chối**.
   Nạn nhân: **kubelet, các thành phần control plane khác, và người dùng cuối**. Vì vậy bài cảnh báo
   thẳng: **không** tái sử dụng một CA đang dùng ở ngữ cảnh khác, và dùng **hai CA khác nhau** cho
   hai cờ này.
4. ConfigMap tên **`extension-apiserver-authentication`**, nằm trong namespace `kube-system`, do
   **chính kube-apiserver tạo và ghi**. Nó chứa **client CA certificate**, **danh sách CN được phép**
   và **tên các header** chứa username, group cùng thông tin phụ. Hai quyền dựng sẵn:
   **role `extension-apiserver-authentication-reader`** trong `kube-system` để đọc được ConfigMap
   đó, và **ClusterRole `system:auth-delegator`** gán cho service account của nó để được phép gửi
   `SubjectAccessReview` sang kube-apiserver khi phân quyền.
5. **Chỉ một.** Bài nói rõ kube-apiserver dùng **cùng một client certificate để xác thực với *tất
   cả* các extension API server** — nó không sinh cert riêng cho từng server, mà chỉ chứng minh "tôi
   là kube-apiserver", và cert đó được tái sử dụng cho mọi yêu cầu chuyển tiếp. Hệ quả:
   `--requestheader-allowed-names` **không phải danh sách các server được phép**, mà là danh sách
   **CN hợp lệ của bên proxy** — tức của chính kube-apiserver. Đặt nó rỗng
   (`--requestheader-allowed-names=""`) là báo cho extension API server chấp nhận **bất kỳ CN nào**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
