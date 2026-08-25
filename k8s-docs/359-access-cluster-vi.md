# Truy cập cluster (Accessing Clusters)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/>
>
> Chủ đề này thảo luận nhiều cách khác nhau để tương tác với cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9 — Bảo mật và multi-tenancy](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy),
dòng **Thực hành**, bài 9/10 — bài cuối cùng trước khi mở lab · Kiểm chứng ở
[Lab 9a](labs/LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md): phần B2.1 (`kubectl config view` và
client certificate), phần B6.1 (dựng kubeconfig cho một ServiceAccount), phần B6.2 và B6.3
(`kubectl proxy` và quyền mà nó mang theo), phần B6.4 (bốn loại proxy, cái nào thật sự có trên
cluster này), phần B6.5 (nhánh *Không dùng kubectl proxy*).

Bài này là mặt kia của bài [338](338-access-api-from-pod-vi.md) vừa đọc. Bài 338 đứng ở góc nhìn
**tiến trình bên trong cluster**: không có kubeconfig, danh tính là ServiceAccount, mọi thứ nằm sẵn
trong `/var/run/secrets/kubernetes.io/serviceaccount/`. Bài này đứng ở góc nhìn **người dùng bên
ngoài**: có kubeconfig trong `~/.kube`, danh tính là client certificate, và phải tự lo phần định vị
apiserver cùng xác minh TLS. Đọc xong hai bài phải phân biệt được ngay một đoạn `curl` đang đứng ở
phía nào.

**Phải hiểu ở lần đọc này:**

- Truy cập cluster luôn cần đúng hai thứ: **vị trí** của cluster và **thông tin xác thực**. `kubectl`
  làm hộ cả hai, và `kubectl config view` là chỗ đọc ra chúng (mục *Truy cập lần đầu với kubectl*).
- Hai đường gọi thẳng REST API và cái giá của từng đường (mục *Truy cập trực tiếp REST API*): chạy
  `kubectl proxy` là cách **được khuyến nghị** — nó dùng vị trí apiserver đã lưu, xác minh danh tính
  apiserver bằng certificate tự ký nên không bị tấn công xen giữa, và tự xác thực hộ; còn tự cấp vị
  trí và credential cho http client là **cách thay thế**, và khi đó bạn phải import root certificate
  vào client thì mới phòng được MITM.
- `kubectl proxy --port=8080` biến kubectl thành một reverse proxy: đoạn **client tới proxy đi bằng
  HTTP** trên localhost, còn đoạn **proxy tới apiserver đi bằng HTTPS** và được kubectl gắn thêm
  header xác thực. Vì vậy `curl http://localhost:8080/api/` chạy được mà không cần token nào.
- Nhánh *Không dùng kubectl proxy*: lấy địa chỉ server từ kubeconfig bằng `kubectl config view
  --minify`, lấy token của ServiceAccount từ một Secret kiểu
  `kubernetes.io/service-account-token`, rồi tự gắn header `Authorization: Bearer`. Chính bài
  cảnh báo: ví dụ đó dùng `--insecure` nên hở cho MITM — kubectl thật thì dùng root certificate
  và client certificate lưu trong `~/.kube`.
- Bảng *Rất nhiều loại proxy* là bản đồ để về sau không gọi nhầm tên: `kubectl proxy` (chạy trên máy
  người dùng hoặc trong pod, thêm header xác thực), apiserver proxy (bastion nằm **trong** tiến
  trình apiserver, tới được Node/Pod/Service), kube proxy (chạy trên **mỗi node**, proxy TCP/UDP,
  **không hiểu HTTP**, chỉ để tới service), proxy/load-balancer đặt trước apiserver, và cloud load
  balancer. Bài chốt: người dùng thường chỉ cần hai loại đầu.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Truy cập API bằng lập trình* — Go client, Python client, các ngôn ngữ khác | dành cho người **viết ứng dụng**; `go get` và `pip install` nằm ngoài bộ công cụ được khóa của lab | vai trò người viết controller xuất hiện ở [giai đoạn 14](00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), bài [181](181-operator-vi.md) |
| *Truy cập API từ một Pod* | chỉ là một con trỏ, không có nội dung mới | bài [338](338-access-api-from-pod-vi.md) vừa đọc ngay trước bài này |
| *Truy cập các service đang chạy trên cluster*, và chi tiết URL của **apiserver proxy** | ở đây chỉ cần biết apiserver proxy tồn tại và nó tới được Node/Pod/Service; cú pháp URL là chuyện của nhóm bài truy cập ứng dụng | bài [369](369-access-cluster-services-vi.md), [giai đoạn 30](00-ALO-TRINH-ADMIN.md#giai-đoạn-30--truy-cập-ứng-dụng-trong-cluster) |
| Loại proxy thứ 4 và thứ 5 — *Proxy/Load-balancer đặt trước apiserver* và *Cloud Load Balancer* | cluster lab có **một** control plane và không có nhà cung cấp cloud, nên hai loại này không tồn tại để quan sát | load balancer đặt trước apiserver thuộc [giai đoạn 8](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) phần HA; Lab 9a phần B6.4 chỉ ghi nhận loại nào có thật trên cluster này |
| *Yêu cầu chuyển hướng* | khả năng redirect đã bị gỡ khỏi Kubernetes, không còn gì để dùng | thứ thay thế nó chính là mục *Truy cập trực tiếp REST API* ngay trong bài này |

---

## Truy cập lần đầu với kubectl (Accessing for the first time with kubectl)

Khi truy cập Kubernetes API lần đầu tiên, chúng tôi khuyên bạn dùng công cụ dòng lệnh
của Kubernetes là `kubectl`.

Để truy cập một cluster, bạn cần biết vị trí (location) của cluster và có thông tin
xác thực (credentials) để truy cập nó. Thông thường, những thứ này được thiết lập tự
động khi bạn làm theo một [Hướng dẫn bắt đầu](https://kubernetes.io/docs/setup/),
hoặc do một người khác đã dựng cluster và cung cấp cho bạn thông tin xác thực cùng vị trí.

Kiểm tra vị trí và thông tin xác thực mà kubectl đang biết bằng lệnh sau:

```shell
kubectl config view
```

Nhiều [ví dụ](https://kubernetes.io/docs/reference/kubectl/quick-reference/) cung cấp
phần giới thiệu về cách dùng `kubectl`, và tài liệu đầy đủ nằm trong
[tài liệu tham khảo kubectl](https://kubernetes.io/docs/reference/kubectl/).

## Truy cập trực tiếp REST API (Directly accessing the REST API) {#directly-accessing-the-rest-api}

Kubectl đảm nhận việc định vị (locate) và xác thực (authenticate) với apiserver.
Nếu bạn muốn truy cập trực tiếp REST API bằng một http client như
curl hay wget, hoặc bằng trình duyệt, có một số cách để định vị và xác thực:

- Chạy kubectl ở chế độ proxy.
  - Cách được khuyến nghị.
  - Dùng vị trí apiserver đã được lưu sẵn.
  - Xác minh danh tính của apiserver bằng certificate tự ký (self-signed cert). Không thể xảy ra tấn công xen giữa (MITM).
  - Tự xác thực với apiserver.
  - Trong tương lai, có thể thực hiện cân bằng tải (load balancing) và chuyển đổi dự phòng (failover) thông minh phía client.
- Cung cấp trực tiếp vị trí và thông tin xác thực cho http client.
  - Cách thay thế.
  - Phù hợp với một số loại mã client bị nhầm lẫn khi dùng proxy.
  - Cần import một root certificate vào trình duyệt của bạn để phòng chống tấn công MITM.

### Dùng kubectl proxy (Using kubectl proxy)

Lệnh sau chạy kubectl ở chế độ hoạt động như một reverse proxy. Nó đảm nhận
việc định vị apiserver và xác thực.
Chạy lệnh như sau:

```shell
kubectl proxy --port=8080
```

Xem [kubectl proxy](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#proxy) để biết thêm chi tiết.

Sau đó bạn có thể khám phá API bằng curl, wget, hoặc trình duyệt; với IPv6 hãy thay
localhost bằng [::1], như sau:

```shell
curl http://localhost:8080/api/
```

Kết quả tương tự như sau:

```json
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

### Không dùng kubectl proxy (Without kubectl proxy)

Dùng `kubectl apply` và `kubectl describe secret...` để tạo một token cho service account mặc định, với grep/cut:

Trước tiên, tạo Secret, yêu cầu một token cho ServiceAccount `default`:

```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: default-token
  annotations:
    kubernetes.io/service-account.name: default
type: kubernetes.io/service-account-token
EOF
```

Tiếp theo, chờ token controller điền token vào Secret:

```shell
while ! kubectl describe secret default-token | grep -E '^token' >/dev/null; do
  echo "waiting for token..." >&2
  sleep 1
done
```

Lấy và sử dụng token đã được sinh ra:

```shell
APISERVER=$(kubectl config view --minify | grep server | cut -f 2- -d ":" | tr -d " ")
TOKEN=$(kubectl describe secret default-token | grep -E '^token' | cut -f2 -d':' | tr -d " ")

curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
```

Kết quả tương tự như sau:

```json
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

Dùng `jsonpath`:

```shell
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
TOKEN=$(kubectl get secret default-token -o jsonpath='{.data.token}' | base64 --decode)

curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
```

Kết quả tương tự như sau:

```json
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}
```

Các ví dụ trên dùng cờ `--insecure`. Điều này khiến kết nối dễ bị tấn công MITM.
Khi kubectl truy cập cluster, nó dùng root certificate đã lưu cùng các client
certificate để truy cập server. (Chúng được cài trong thư mục `~/.kube`).
Vì certificate của cluster thường là tự ký, bạn có thể cần cấu hình đặc biệt
để http client của bạn dùng được root certificate.

Trên một số cluster, apiserver không yêu cầu xác thực; nó có thể phục vụ trên
localhost, hoặc được bảo vệ bởi tường lửa (firewall). Không có chuẩn chung nào
cho việc này. [Kiểm soát truy cập tới API](119-controlling-access-vi.md)
mô tả cách quản trị viên cluster có thể cấu hình điều này.

## Truy cập API bằng lập trình (Programmatic access to the API)

Kubernetes chính thức hỗ trợ thư viện client cho [Go](#go-client) và [Python](#python-client).

### Go client

* Để lấy thư viện, chạy lệnh sau: `go get k8s.io/client-go@kubernetes-<kubernetes-version-number>`,
  xem [INSTALL.md](https://github.com/kubernetes/client-go/blob/master/INSTALL.md#for-the-casual-user)
  để có hướng dẫn cài đặt chi tiết. Xem
  [https://github.com/kubernetes/client-go](https://github.com/kubernetes/client-go#compatibility-matrix)
  để biết các phiên bản nào được hỗ trợ.
* Viết ứng dụng dựa trên các client của client-go. Lưu ý rằng client-go định nghĩa các đối tượng API riêng của nó,
  vì vậy nếu cần, hãy import các định nghĩa API từ client-go thay vì từ repository chính,
  ví dụ `import "k8s.io/client-go/kubernetes"` là cách đúng.

Go client có thể dùng cùng [file kubeconfig](111-kubeconfig-vi.md)
như kubectl CLI để định vị và xác thực với apiserver. Xem
[ví dụ](https://git.k8s.io/client-go/examples/out-of-cluster-client-configuration/main.go) này.

Nếu ứng dụng được triển khai dưới dạng một Pod trong cluster, hãy tham khảo [mục tiếp theo](#accessing-the-api-from-a-pod).

### Python client

Để dùng [Python client](https://github.com/kubernetes-client/python), chạy lệnh sau:
`pip install kubernetes`. Xem [trang Python Client Library](https://github.com/kubernetes-client/python)
để biết thêm các tùy chọn cài đặt khác.

Python client có thể dùng cùng [file kubeconfig](111-kubeconfig-vi.md)
như kubectl CLI để định vị và xác thực với apiserver. Xem
[ví dụ](https://github.com/kubernetes-client/python/tree/master/examples) này.

### Các ngôn ngữ khác (Other languages)

Có các [thư viện client](https://kubernetes.io/docs/reference/using-api/client-libraries/) để truy cập API từ các ngôn ngữ khác.
Xem tài liệu của từng thư viện để biết cách chúng xác thực.

## Truy cập API từ một Pod (Accessing the API from a Pod) {#accessing-the-api-from-a-pod}

Khi truy cập API từ một pod, việc định vị và xác thực với API server
có đôi chút khác biệt.

Vui lòng xem [Truy cập API từ bên trong một Pod](338-access-api-from-pod-vi.md)
để biết thêm chi tiết.

## Truy cập các service đang chạy trên cluster (Accessing services running on the cluster)

Mục trước mô tả cách kết nối tới Kubernetes API server.
Để biết thông tin về việc kết nối tới các service khác đang chạy trên một cluster Kubernetes, xem
[Truy cập các Service của cluster](369-access-cluster-services-vi.md).

## Yêu cầu chuyển hướng (Requesting redirects)

Khả năng chuyển hướng (redirect) đã bị loại bỏ sau khi ngưng hỗ trợ (deprecated). Vui lòng dùng proxy (xem bên dưới) thay thế.

## Rất nhiều loại proxy (So many proxies)

Có một số loại proxy khác nhau mà bạn có thể gặp khi dùng Kubernetes:

1. [kubectl proxy](#directly-accessing-the-rest-api):

   - chạy trên máy tính của người dùng hoặc trong một pod
   - proxy từ một địa chỉ localhost tới Kubernetes apiserver
   - client tới proxy dùng HTTP
   - proxy tới apiserver dùng HTTPS
   - định vị apiserver
   - thêm các header xác thực

1. [apiserver proxy](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster-services/#discovering-builtin-services):

   - là một bastion được tích hợp sẵn trong apiserver
   - kết nối người dùng ở bên ngoài cluster tới các cluster IP mà nếu không có nó thì có thể không truy cập được
   - chạy trong các tiến trình của apiserver
   - client tới proxy dùng HTTPS (hoặc http nếu apiserver được cấu hình như vậy)
   - proxy tới đích có thể dùng HTTP hoặc HTTPS tùy proxy lựa chọn dựa trên thông tin sẵn có
   - có thể được dùng để truy cập một Node, Pod, hoặc Service
   - thực hiện cân bằng tải khi được dùng để truy cập một Service

1. [kube proxy](https://kubernetes.io/docs/concepts/services-networking/service#ips-and-vips):

   - chạy trên mỗi node
   - proxy các giao thức UDP và TCP
   - không hiểu HTTP
   - cung cấp cân bằng tải
   - chỉ được dùng để truy cập các service

1. Một Proxy/Load-balancer đặt trước (các) apiserver:

   - sự tồn tại và cách triển khai thay đổi tùy từng cluster (ví dụ nginx)
   - nằm giữa tất cả các client và một hoặc nhiều apiserver
   - đóng vai trò bộ cân bằng tải (load balancer) nếu có nhiều apiserver.

1. Cloud Load Balancer trên các service bên ngoài:

   - được cung cấp bởi một số nhà cung cấp cloud (ví dụ AWS ELB, Google Cloud Load Balancer)
   - được tạo tự động khi Kubernetes service có type `LoadBalancer`
   - chỉ dùng UDP/TCP
   - cách triển khai thay đổi tùy nhà cung cấp cloud.

Người dùng Kubernetes thường sẽ không cần bận tâm tới bất kỳ loại nào ngoài hai loại đầu tiên.
Quản trị viên cluster thường sẽ đảm bảo các loại còn lại được thiết lập đúng.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Theo bài, muốn truy cập một cluster bạn phải có đúng hai thứ. Hai thứ đó là gì, trên
   `lab-k8s-master` bạn đọc chúng ra bằng lệnh nào, và bài nói chúng được cài ở thư mục nào?
2. Bạn chạy `kubectl proxy --port=8080` trên `lab-k8s-master`, rồi `curl http://localhost:8080/api/`
   **không kèm** token hay certificate nào mà vẫn nhận được JSON. Ai đã xác thực với apiserver, và
   hai chặng `client → proxy` với `proxy → apiserver` chạy trên giao thức nào?
3. **Câu bẫy.** `kubectl proxy` và `kube proxy` — hai cái tên chỉ khác một chữ. Chúng chạy ở đâu,
   xử lý cái gì, và cái nào giúp bạn gọi được REST API của apiserver?
4. Cả hai ví dụ ở mục *Không dùng kubectl proxy* đều kết thúc bằng `curl ... --insecure`. Cờ đó đánh
   đổi điều gì, và bài chỉ ra cách làm đúng là gì?
5. Cùng là một dòng `curl` gọi `/api`, đoạn shell của bài [338](338-access-api-from-pod-vi.md) và
   đoạn shell của bài này lấy **địa chỉ apiserver** và **token** từ hai nguồn hoàn toàn khác nhau.
   Nêu hai nguồn đó, và giải thích vì sao một Pod thường không dùng được cách của bài này.

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Vị trí (location) của cluster và thông tin xác thực (credentials) để truy cập nó.** Đọc bằng
   `kubectl config view` — lệnh này in ra chính vị trí và credential mà kubectl đang biết. Bài nói
   root certificate và client certificate được cài trong thư mục **`~/.kube`**; trên cluster lab,
   đó là kubeconfig mà kubeadm đã đặt cho bạn.
2. **Chính `kubectl proxy` xác thực hộ bạn**: nó chạy ở chế độ reverse proxy, tự định vị apiserver,
   tự xác thực và **thêm các header xác thực** vào request. Hai chặng chạy trên hai giao thức khác
   nhau: **client tới proxy dùng HTTP** (nên `curl http://localhost:8080` mới hợp lệ), còn **proxy
   tới apiserver dùng HTTPS**, có xác minh danh tính apiserver bằng certificate tự ký nên không xảy
   ra tấn công xen giữa. Hệ quả cần nhớ: cổng đó mang **danh tính của chính bạn**, ai chạm được vào
   nó thì thao tác được với quyền của bạn.
3. **Hai thứ khác hẳn nhau, không thay thế nhau.** `kubectl proxy` chạy trên máy tính của người dùng
   hoặc trong một pod, proxy từ một địa chỉ localhost tới apiserver, định vị apiserver và thêm header
   xác thực — client vào bằng HTTP, ra bằng HTTPS. `kube proxy` chạy **trên mỗi node**, proxy các
   giao thức **UDP và TCP**, **không hiểu HTTP**, cung cấp cân bằng tải và **chỉ được dùng để truy
   cập các service**. Muốn gọi REST API của apiserver thì chỉ `kubectl proxy` giúp được bạn.
4. `--insecure` **bỏ qua việc xác minh certificate của apiserver**, và bài nói thẳng: điều này khiến
   kết nối **dễ bị tấn công xen giữa (MITM)**. Cách làm đúng theo bài: dùng đúng thứ kubectl vẫn
   dùng — **root certificate và client certificate lưu trong `~/.kube`** — và vì certificate của
   cluster thường là tự ký nên có thể phải cấu hình đặc biệt cho http client nhận root certificate
   đó. Còn nếu không muốn tự lo phần này thì quay lại **cách được khuyến nghị**: `kubectl proxy`.
5. Bài 338 đứng **trong** cluster: địa chỉ lấy từ biến môi trường `KUBERNETES_SERVICE_HOST` hoặc tên
   DNS `kubernetes.default.svc`, còn token và CA lấy từ ba file trong
   `/var/run/secrets/kubernetes.io/serviceaccount/` mà Kubernetes tự đặt vào container. Bài này đứng
   **ngoài** cluster: địa chỉ lấy từ **kubeconfig** của bạn (`kubectl config view --minify`), còn
   token lấy bằng **chính `kubectl`** từ một Secret kiểu `kubernetes.io/service-account-token`. Pod
   thường **không có kubeconfig và không có `kubectl`** trong image, nên nó không thể chạy hai lệnh
   đầu của bài này — đó đúng là lý do bài 338 tồn tại như một bài riêng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng, rồi đọc bài kế của dòng Thực hành —
[190 — Truy cập cluster bằng Kubernetes API](190-access-cluster-api-vi.md), bài cuối của dòng.
Xong bài đó thì mở
[Lab 9a — ServiceAccount, authn/authz và RBAC](labs/LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md),
rồi đọc tiếp nhóm bài policy của giai đoạn 9 trước khi vào
[Lab 9b — Pod Security và hardening](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md).
