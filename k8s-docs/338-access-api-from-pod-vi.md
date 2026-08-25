# Truy cập Kubernetes API từ một Pod (Accessing the Kubernetes API from a Pod)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/run-application/access-api-from-pod/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9 — Bảo mật và multi-tenancy](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy),
dòng **Thực hành**, bài 8/10 · Kiểm chứng ở [Lab 9a](labs/LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md)
phần B5: B5.1 Pod gọi API bằng token của chính nó, B5.2 đối chiếu ba file và hai biến môi trường
mà bài liệt kê với giá trị thật trên cluster, B5.3 cùng một Pod nhưng hai namespace cho hai kết quả.

Bài này và bài [359](359-access-cluster-vi.md) ngay sau nó rất dễ nhầm vào nhau. Bài này trả lời
câu hỏi của **tiến trình chạy bên trong cluster**: một container không có kubeconfig thì tìm API
server ở đâu và lấy danh tính từ đâu. Bài 359 trả lời câu hỏi của **người dùng ngồi ngoài
cluster**: kubeconfig, `kubectl proxy`, và các cách gọi thẳng REST API từ máy trạm. Đọc bài này
với đúng góc nhìn "tôi là Pod", đừng lẫn sang góc nhìn "tôi là admin".

**Phải hiểu ở lần đọc này:**

- Từ trong Pod, việc **định vị** API server khác hẳn client bên ngoài: lấy từ hai biến môi trường
  `KUBERNETES_SERVICE_HOST` và `KUBERNETES_SERVICE_PORT_HTTPS`, hoặc dùng tên DNS
  `kubernetes.default.svc` — địa chỉ in-cluster của API server được công bố qua Service tên
  `kubernetes` trong namespace `default` (mục *Truy cập trực tiếp REST API*).
- Ba file nằm sẵn trong cây filesystem của **mọi container** thuộc Pod, tại
  `/var/run/secrets/kubernetes.io/serviceaccount/`: `token` là bearer token của service account gắn
  với Pod, `ca.crt` là bundle certificate dùng để **xác minh serving certificate** của API server,
  `namespace` là namespace mặc định cho các thao tác API có phạm vi namespace.
- Cảnh báo về certificate ngay trong bài: Kubernetes **không đảm bảo** API server có certificate
  hợp lệ cho hostname `kubernetes.default.svc`; điều được kỳ vọng là control plane xuất trình
  certificate hợp lệ cho hostname hoặc địa chỉ IP mà `$KUBERNETES_SERVICE_HOST` đại diện. Chọn
  endpoint nào là một quyết định có hệ quả lên việc xác minh TLS.
- Ba cách kết nối mà bài đưa ra và ranh giới giữa chúng: thư viện client chính thức
  (`rest.InClusterConfig()`, `config.load_incluster_config()`) tự khám phá host và tự xác thực;
  `kubectl proxy` chạy làm container sidecar, xác thực hộ rồi mở API trên interface `localhost`
  của Pod; hoặc tự gọi REST. Cả ba đều dùng **cùng một credential**: thông tin xác thực service
  account của Pod.
- Đoạn shell ở mục *Không sử dụng proxy* là bản rút gọn của mọi thứ trên: đọc `namespace`, đọc
  `token`, trỏ `--cacert` vào `ca.crt`, đặt header `Authorization: Bearer ${TOKEN}` rồi `curl` tới
  `${APISERVER}/api`. Đó chính là phần mà thư viện client làm giúp bạn.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Sử dụng các thư viện client chính thức* — ví dụ Go và Python | dành cho người **viết ứng dụng** gọi API, không phải cho quản trị viên; cài SDK cũng nằm ngoài bộ image được khóa của lab | vai trò người viết controller xuất hiện ở [giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), bài [181](181-operator-vi.md) |
| *Sử dụng kubectl proxy* chạy dạng sidecar trong Pod | image của Pod phải có sẵn `kubectl`, mà bộ image được khóa của lab không có | bài [359](359-access-cluster-vi.md) ngay sau bài này mô tả đầy đủ `kubectl proxy`; Lab 9a kiểm chứng nhánh proxy ở phần B6.2, chạy trên `lab-k8s-master` |
| Cơ chế **tiêm** ba file đó vào Pod: projected volume, token ngắn hạn tự xoay vòng, audience | bài này chỉ nói các file "được đặt vào", không nói ai đặt và token sống bao lâu | bài [118](118-service-accounts-vi.md) và [279](279-configure-service-account-vi.md) của cùng giai đoạn 9 |

---

Hướng dẫn này trình bày cách truy cập Kubernetes API từ bên trong một pod.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Khuyến nghị chạy hướng dẫn này trên một cluster có ít nhất hai node không
đóng vai trò là máy chủ control plane. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/)
hoặc dùng một trong các sân chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

## Truy cập API từ bên trong một Pod (Accessing the API from within a Pod)

Khi truy cập API từ bên trong một Pod, việc định vị và xác thực (authenticate)
với API server hơi khác so với trường hợp client bên ngoài.

Cách dễ nhất để sử dụng Kubernetes API từ một Pod là dùng một trong các
[thư viện client](https://kubernetes.io/docs/reference/using-api/client-libraries/) chính thức. Các
thư viện này có thể tự động khám phá (discover) API server và xác thực.

### Sử dụng các thư viện client chính thức (Using Official Client Libraries)

Từ bên trong một Pod, các cách được khuyến nghị để kết nối tới Kubernetes API là:

- Với client Go, hãy dùng
  [thư viện client Go](https://github.com/kubernetes/client-go/) chính thức.
  Hàm `rest.InClusterConfig()` tự động xử lý việc khám phá host của API và xác thực.
  Xem [một ví dụ tại đây](https://git.k8s.io/client-go/examples/in-cluster-client-configuration/main.go).

- Với client Python, hãy dùng
  [thư viện client Python](https://github.com/kubernetes-client/python/) chính thức.
  Hàm `config.load_incluster_config()` tự động xử lý việc khám phá host của API và xác thực.
  Xem [một ví dụ tại đây](https://github.com/kubernetes-client/python/blob/master/examples/in_cluster_config.py).

- Còn có nhiều thư viện khác, vui lòng tham khảo trang
  [Client Libraries](https://kubernetes.io/docs/reference/using-api/client-libraries/).

Trong mỗi trường hợp, thông tin xác thực (credential) service account của Pod được dùng để
giao tiếp một cách an toàn với API server.

### Truy cập trực tiếp REST API (Directly accessing the REST API)

Khi đang chạy trong một Pod, container của bạn có thể tạo một URL HTTPS cho Kubernetes API
server bằng cách lấy các biến môi trường `KUBERNETES_SERVICE_HOST` và
`KUBERNETES_SERVICE_PORT_HTTPS`. Địa chỉ trong cluster (in-cluster) của API server cũng được
công bố qua một Service tên là `kubernetes` trong namespace `default`, nhờ đó các pod có thể
tham chiếu `kubernetes.default.svc` như một tên DNS cho API server cục bộ.

> **Ghi chú:**
> Kubernetes không đảm bảo rằng API server có certificate hợp lệ cho
> hostname `kubernetes.default.svc`;
> tuy nhiên, control plane **được** kỳ vọng sẽ xuất trình một certificate hợp lệ cho
> hostname hoặc địa chỉ IP mà `$KUBERNETES_SERVICE_HOST` đại diện.

Cách được khuyến nghị để xác thực với API server là dùng thông tin xác thực của một
[service account](279-configure-service-account-vi.md).
Theo mặc định, một Pod được gắn với một service account, và thông tin xác thực (token) của
service account đó được đặt vào cây filesystem của mỗi container trong Pod đó,
tại `/var/run/secrets/kubernetes.io/serviceaccount/token`.

Nếu có, một bundle certificate được đặt vào cây filesystem của mỗi container
tại `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`, và nên được
dùng để xác minh serving certificate của API server.

Cuối cùng, namespace mặc định dùng cho các thao tác API có phạm vi namespace được đặt trong một
file tại `/var/run/secrets/kubernetes.io/serviceaccount/namespace` trong mỗi container.

### Sử dụng kubectl proxy (Using kubectl proxy)

Nếu bạn muốn truy vấn API mà không dùng thư viện client chính thức, bạn có thể chạy `kubectl proxy`
như là [lệnh (command)](330-define-command-argument-vi.md)
của một container sidecar mới trong Pod. Bằng cách này, `kubectl proxy` sẽ xác thực
với API và mở nó ra trên interface `localhost` của Pod, nhờ đó các container khác
trong Pod có thể dùng nó trực tiếp.

### Không sử dụng proxy (Without using a proxy)

Bạn có thể tránh dùng kubectl proxy bằng cách truyền token xác thực
trực tiếp cho API server. Certificate nội bộ bảo đảm an toàn cho kết nối.

```shell
# Trỏ tới hostname nội bộ của API server
APISERVER=https://kubernetes.default.svc

# Đường dẫn tới token của ServiceAccount
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount

# Đọc namespace của Pod này
NAMESPACE=$(cat ${SERVICEACCOUNT}/namespace)

# Đọc bearer token của ServiceAccount
TOKEN=$(cat ${SERVICEACCOUNT}/token)

# Tham chiếu certificate authority (CA) nội bộ
CACERT=${SERVICEACCOUNT}/ca.crt

# Khám phá API với TOKEN
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET ${APISERVER}/api
```

Kết quả đầu ra sẽ tương tự như sau:

```json
{
  "kind": "APIVersions",
  "versions": ["v1"],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Một Pod busybox chạy trên `lab-k8s-worker2`. Trong image không có kubeconfig, không ai copy
   file gì vào, cũng không ai truyền biến môi trường nào cho nó. Nó lấy ở đâu ra ba thứ cần để
   gọi API: **địa chỉ** API server, **thứ để xác minh** phía server, và **danh tính** của chính nó?
2. **Câu bẫy.** File `/var/run/secrets/kubernetes.io/serviceaccount/namespace` có phải là thứ
   giới hạn Pod chỉ được thao tác trong namespace của nó không?
3. Bài cảnh báo gì về certificate của hostname `kubernetes.default.svc`, và cảnh báo đó ảnh hưởng
   thế nào tới lựa chọn giữa `https://kubernetes.default.svc` và `$KUBERNETES_SERVICE_HOST`?
4. Bài đưa ba cách gọi API từ trong Pod: thư viện client chính thức, `kubectl proxy` dạng sidecar,
   và gọi thẳng REST. Điểm chung về **credential** của cả ba là gì, và cách nào đòi thêm phần mềm
   bên trong Pod?
5. Trong đoạn shell của mục *Không sử dụng proxy*, vì sao phải truyền `--cacert ${CACERT}` cho
   `curl` thay vì bỏ qua bước xác minh?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Cả ba đều do Kubernetes đặt sẵn vào Pod, không cần cấu hình gì.** Địa chỉ: hai biến môi
   trường `KUBERNETES_SERVICE_HOST` và `KUBERNETES_SERVICE_PORT_HTTPS` — biến môi trường **có sẵn**
   vì Service `kubernetes` trong namespace `default` công bố địa chỉ in-cluster của API server; hoặc
   dùng chính tên DNS `kubernetes.default.svc`. Thứ để xác minh phía server: file `ca.crt`. Danh
   tính: file `token`, tức bearer token của service account gắn với Pod. Cả ba file nằm trong
   `/var/run/secrets/kubernetes.io/serviceaccount/` của **mọi** container trong Pod.
2. **Không.** Bài nói rất hẹp: đó là namespace **mặc định** dùng cho các thao tác API *có phạm vi
   namespace* — nói cách khác là một giá trị tiện dụng để bạn không phải hardcode tên namespace vào
   image. Nó **không phải ranh giới quyền**: việc Pod được đọc hay ghi gì nằm ở chặng phân quyền
   (bài [119](119-controlling-access-vi.md)), gắn với danh tính trong `token` chứ không gắn với nội
   dung file `namespace`. Sửa hay bỏ qua file đó không cho Pod thêm quyền nào.
3. Kubernetes **không đảm bảo** API server có certificate hợp lệ cho đúng hostname
   `kubernetes.default.svc`; thứ được kỳ vọng là certificate hợp lệ cho hostname hoặc địa chỉ IP mà
   `$KUBERNETES_SERVICE_HOST` đại diện. Hệ quả: nếu bạn xác minh TLS nghiêm ngặt bằng `ca.crt`,
   **`$KUBERNETES_SERVICE_HOST` là endpoint chắc chắn hơn**; dùng tên DNS đẹp mắt kia có thể vấp
   lỗi không khớp hostname trên một số cluster.
4. Điểm chung: **cả ba đều dùng thông tin xác thực service account của Pod** — chỉ khác ai đọc nó
   hộ bạn. Thư viện client tự khám phá host và tự xác thực; `kubectl proxy` xác thực hộ rồi mở API
   ra trên `localhost` của Pod cho các container khác dùng; gọi thẳng REST thì bạn tự đọc file và
   tự gắn header. Hai cách đầu **đòi thêm phần mềm trong Pod**: SDK của ngôn ngữ, hoặc một container
   sidecar có `kubectl`.
5. Vì `ca.crt` chính là **bundle certificate để xác minh serving certificate của API server**, và
   đây là bước duy nhất bảo đảm bạn đang nói chuyện với API server thật. Bỏ xác minh đồng nghĩa với
   việc **gửi bearer token của service account** — tức toàn bộ danh tính của Pod — cho bất cứ thứ gì
   trả lời ở đầu kia. Bài nói ngắn gọn: certificate nội bộ là thứ bảo đảm an toàn cho kết nối này.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
