# Quản lý TLS Certificate trong một Cluster (Manage TLS Certificates in a Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Checkpoint tiếp nối — nhánh `/docs/tasks/`](00-ALO-TRINH-ADMIN.md#checkpoint-tiếp-nối--nhánh-docstasks)
→ [CP3 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#cp3--vòng-đời-chứng-chỉ), bài 5/7 · Kiểm chứng
trên cluster lab: đóng cả ba vai — người xin, người phê duyệt, người ký — cho một CSR, rồi lấy
certificate về.

Bài này bổ ngang bài [397](397-certificate-issue-client-csr-vi.md): 397 cấp certificate cho
**client** gọi API, còn bài này cấp **serving certificate** cho workload, và giải thích trọn bộ
máy `certificates.k8s.io`.

**Lưu ý về công cụ:** bài dùng `cfssl` và `jq`, cả hai **không nằm trong baseline** của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md). Cài thêm là thay đổi môi trường lab — quyết định
trước khi làm, đừng cài giữa chừng.

**Phải hiểu ở lần đọc này:**

- **Ba vai tách bạch** trong `certificates.k8s.io`: *người xin* tạo CSR; *người phê duyệt* quyết
  định có cho phép không; *người ký* (signer) theo dõi các CSR mang đúng `signerName` của mình,
  ký rồi ghi certificate vào `.status`. **Phê duyệt không phải là ký** — sau khi approve, CSR vẫn
  nằm chờ signer.
- **Cảnh báo quan trọng nhất của bài:** certificate cấp qua API này được ký bởi một **CA chuyên
  dụng**, **không phải** CA gốc của cluster. Đừng bao giờ giả định chúng xác thực được với CA gốc
  của cluster, kể cả khi bạn có thể cấu hình để dùng CA gốc.
- Hệ quả cho workload: **không dùng `kube-root-ca.crt`** làm gốc tin cậy cho ứng dụng của bạn —
  CA đó chỉ để xác thực endpoint nội bộ của Kubernetes. Muốn có trust riêng thì tạo CA riêng và
  phân phối qua [ConfigMap](275-configure-pod-configmap-vi.md).
- `signerName` là **bắt buộc** và phải cụ thể (ví dụ trong bài là `example.com/serving`); nó
  chính là thứ để signer biết CSR nào thuộc về mình.
- **Người phê duyệt phải xác minh đúng hai điều**, và chỉ approve khi **cả hai** thỏa: (1) subject
  thực sự kiểm soát private key đã ký CSR — chống mạo danh; (2) subject được phép hoạt động trong
  ngữ cảnh nó xin — chống kẻ lạ chen vào cluster. Bài nhấn: quyền `approve` quyết định *ai tin
  ai*, không được cấp rộng rãi.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách `signerName` được hỗ trợ sẵn | thuộc nhánh `/docs/reference/`, chưa có bản dịch | khi cần chọn signer thật thay cho signer ví dụ |
| Cấu hình `tls.Config`/`RootCAs` phía ứng dụng Go | là việc của lập trình viên ứng dụng, không phải quản trị cluster | khi tự viết workload dùng mTLS |
| Viết certificate controller tự động | chỉ cần khi dùng API này ở quy mô lớn | [CP13](00-ALO-TRINH-ADMIN.md#cp13--mở-rộng-kubernetes), sau khi biết mẫu controller |

---

Kubernetes cung cấp một API tên là `certificates.k8s.io`, cho phép bạn cấp phát (provision)
các TLS certificate được ký bởi một Certificate Authority (CA) do chính bạn kiểm soát. Các CA
và certificate này có thể được các workload của bạn sử dụng để thiết lập quan hệ tin cậy
(trust).

API `certificates.k8s.io` dùng một giao thức tương tự với
[bản nháp ACME](https://github.com/ietf-wg-acme/acme/).

> **Ghi chú:** Các certificate được tạo bằng API `certificates.k8s.io` được ký bởi một
> [CA chuyên dụng](#configuring-your-cluster-to-provide-signing). Bạn có thể cấu hình cluster
> của mình để dùng CA gốc (root CA) của cluster cho mục đích này, nhưng bạn không bao giờ nên
> dựa vào điều đó. Đừng giả định rằng những certificate này sẽ xác thực thành công đối với CA
> gốc của cluster.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
  tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
  không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
  bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
  các sân chơi (playground) Kubernetes sau:

  * [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  * [KodeKloud](https://kodekloud.com/public-playgrounds)

Bạn cần công cụ `cfssl`. Bạn có thể tải `cfssl` từ
[https://github.com/cloudflare/cfssl/releases](https://github.com/cloudflare/cfssl/releases).

Một số bước trong trang này dùng công cụ `jq`. Nếu bạn chưa có `jq`, bạn có thể cài nó qua
kho phần mềm của hệ điều hành, hoặc lấy nó từ
[https://jqlang.github.io/jq/](https://jqlang.github.io/jq/).

## Tin cậy TLS trong một cluster (Trusting TLS in a cluster)

Việc tin cậy [CA tùy chỉnh](#configuring-your-cluster-to-provide-signing) từ một ứng dụng chạy
dưới dạng pod thường đòi hỏi thêm một số cấu hình phía ứng dụng. Bạn sẽ cần thêm bundle
certificate của CA vào danh sách các CA certificate mà TLS client hoặc server tin cậy. Ví dụ,
với một cấu hình TLS trong Golang, bạn sẽ làm điều này bằng cách phân tích (parse) chuỗi
certificate rồi thêm các certificate đã phân tích vào field `RootCAs` trong struct
[`tls.Config`](https://pkg.go.dev/crypto/tls#Config).

> **Ghi chú:** Ngay cả khi certificate của CA tùy chỉnh có thể đã nằm sẵn trong hệ thống tệp
> (trong ConfigMap `kube-root-ca.crt`), bạn không nên dùng certificate authority đó cho bất kỳ
> mục đích nào khác ngoài việc xác thực các endpoint nội bộ của Kubernetes. Một ví dụ về
> endpoint nội bộ của Kubernetes là Service có tên `kubernetes` trong namespace default.
>
> Nếu bạn muốn dùng một certificate authority tùy chỉnh cho các workload của mình, bạn nên tạo
> CA đó riêng biệt, rồi phân phối certificate của CA đó bằng một
> [ConfigMap](275-configure-pod-configmap-vi.md) mà các pod của bạn có quyền đọc.

## Yêu cầu một certificate (Requesting a certificate)

Phần sau đây minh họa cách tạo một TLS certificate cho một Kubernetes service được truy cập
thông qua DNS.

> **Ghi chú:** Hướng dẫn này dùng CFSSL: bộ công cụ PKI và TLS của Cloudflare. Xem bài viết
> [Introducing CFSSL - CloudFlare's PKI toolkit](https://blog.cloudflare.com/introducing-cfssl/)
> trên blog của Cloudflare để biết thêm thông tin.

## Tạo một certificate signing request (Create a certificate signing request)

Sinh một private key và một certificate signing request (hay CSR) bằng cách chạy lệnh sau:

```shell
cat <<EOF | cfssl genkey - | cfssljson -bare server
{
  "hosts": [
    "my-svc.my-namespace.svc.cluster.local",
    "my-pod.my-namespace.pod.cluster.local",
    "192.0.2.24",
    "10.0.34.2"
  ],
  "CN": "my-pod.my-namespace.pod.cluster.local",
  "key": {
    "algo": "ecdsa",
    "size": 256
  }
}
EOF
```

Trong đó `192.0.2.24` là cluster IP của service,
`my-svc.my-namespace.svc.cluster.local` là tên DNS của service,
`10.0.34.2` là IP của pod và `my-pod.my-namespace.pod.cluster.local`
là tên DNS của pod. Bạn sẽ thấy đầu ra tương tự như sau:

```
2022/02/01 11:45:32 [INFO] generate received request
2022/02/01 11:45:32 [INFO] received CSR
2022/02/01 11:45:32 [INFO] generating key: ecdsa-256
2022/02/01 11:45:32 [INFO] encoded CSR
```

Lệnh này sinh ra hai file: file `server.csr` chứa yêu cầu cấp certificate
[PKCS#10](https://tools.ietf.org/html/rfc2986) mã hóa dạng PEM, và file `server-key.pem` chứa
key mã hóa dạng PEM của certificate sắp được tạo.

## Tạo một đối tượng CertificateSigningRequest để gửi tới Kubernetes API (Create a CertificateSigningRequest object to send to the Kubernetes API)

Sinh một manifest CSR (dạng YAML) và gửi nó tới API server. Bạn có thể làm điều đó bằng cách
chạy lệnh sau:

```shell
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: my-svc.my-namespace
spec:
  request: $(cat server.csr | base64 | tr -d '\n')
  signerName: example.com/serving
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```

Lưu ý rằng file `server.csr` được tạo ở bước 1 đã được mã hóa base64 và đặt vào field
`.spec.request`. Bạn cũng đang yêu cầu một certificate với các key usage "digital signature",
"key encipherment" và "server auth", được ký bởi một signer ví dụ là `example.com/serving`.
Bắt buộc phải yêu cầu một `signerName` cụ thể.
Xem tài liệu về
[các tên signer được hỗ trợ](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#signers)
để biết thêm thông tin.

CSR bây giờ sẽ hiển thị trong API ở trạng thái Pending. Bạn có thể xem nó bằng cách chạy:

```shell
kubectl describe csr my-svc.my-namespace
```

```none
Name:                   my-svc.my-namespace
Labels:                 <none>
Annotations:            <none>
CreationTimestamp:      Tue, 01 Feb 2022 11:49:15 -0500
Requesting User:        yourname@example.com
Signer:                 example.com/serving
Status:                 Pending
Subject:
        Common Name:    my-pod.my-namespace.pod.cluster.local
        Serial Number:
Subject Alternative Names:
        DNS Names:      my-pod.my-namespace.pod.cluster.local
                        my-svc.my-namespace.svc.cluster.local
        IP Addresses:   192.0.2.24
                        10.0.34.2
Events: <none>
```

## Phê duyệt CertificateSigningRequest (Get the CertificateSigningRequest approved) {#get-the-certificate-signing-request-approved}

Việc phê duyệt (approve) một
[certificate signing request](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)
được thực hiện hoặc bởi một quy trình phê duyệt tự động, hoặc theo từng trường hợp riêng lẻ bởi
một quản trị viên cluster. Nếu bạn được cấp quyền phê duyệt một yêu cầu certificate, bạn có thể
làm điều đó thủ công bằng `kubectl`; ví dụ:

```shell
kubectl certificate approve my-svc.my-namespace
```

```none
certificatesigningrequest.certificates.k8s.io/my-svc.my-namespace approved
```

Bây giờ bạn sẽ thấy như sau:

```shell
kubectl get csr
```

```none
NAME                  AGE   SIGNERNAME            REQUESTOR              REQUESTEDDURATION   CONDITION
my-svc.my-namespace   10m   example.com/serving   yourname@example.com   <none>              Approved
```

Điều này có nghĩa là yêu cầu certificate đã được phê duyệt và đang chờ signer được yêu cầu ký
nó.

## Ký CertificateSigningRequest (Sign the CertificateSigningRequest) {#sign-the-certificate-signing-request}

Tiếp theo, bạn sẽ đóng vai một certificate signer, cấp phát certificate và tải nó lên API.

Thông thường một signer sẽ theo dõi (watch) API CertificateSigningRequest để tìm các đối tượng
mang `signerName` của nó, kiểm tra xem chúng đã được phê duyệt hay chưa, ký certificate cho
những yêu cầu đó, rồi cập nhật status của đối tượng API với certificate đã cấp.

### Tạo một Certificate Authority (Create a Certificate Authority)

Bạn cần một authority để đặt chữ ký số lên certificate mới.

Trước tiên, tạo một certificate dùng để ký bằng cách chạy lệnh sau:

```shell
cat <<EOF | cfssl gencert -initca - | cfssljson -bare ca
{
  "CN": "My Example Signer",
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF
```

Bạn sẽ thấy đầu ra tương tự như sau:

```none
2022/02/01 11:50:39 [INFO] generating a new CA key and certificate from CSR
2022/02/01 11:50:39 [INFO] generate received request
2022/02/01 11:50:39 [INFO] received CSR
2022/02/01 11:50:39 [INFO] generating key: rsa-2048
2022/02/01 11:50:39 [INFO] encoded CSR
2022/02/01 11:50:39 [INFO] signed certificate with serial number 263983151013686720899716354349605500797834580472
```

Lệnh này tạo ra một file key của certificate authority (`ca-key.pem`) và một certificate
(`ca.pem`).

### Cấp phát một certificate (Issue a certificate)

```json
{
    "signing": {
        "default": {
            "usages": [
                "digital signature",
                "key encipherment",
                "server auth"
            ],
            "expiry": "876000h",
            "ca_constraint": {
                "is_ca": false
            }
        }
    }
}
```

Dùng file cấu hình ký `server-signing-config.json` cùng với file key và certificate của
certificate authority để ký yêu cầu certificate:

```shell
kubectl get csr my-svc.my-namespace -o jsonpath='{.spec.request}' | \
  base64 --decode | \
  cfssl sign -ca ca.pem -ca-key ca-key.pem -config server-signing-config.json - | \
  cfssljson -bare ca-signed-server
```

Bạn sẽ thấy đầu ra tương tự như sau:

```
2022/02/01 11:52:26 [INFO] signed certificate with serial number 576048928624926584381415936700914530534472870337
```

Lệnh này tạo ra một file serving certificate đã được ký, `ca-signed-server.pem`.

### Tải certificate đã ký lên (Upload the signed certificate)

Cuối cùng, điền certificate đã ký vào status của đối tượng API:

```shell
kubectl get csr my-svc.my-namespace -o json | \
  jq '.status.certificate = "'$(base64 ca-signed-server.pem | tr -d '\n')'"' | \
  kubectl replace --raw /apis/certificates.k8s.io/v1/certificatesigningrequests/my-svc.my-namespace/status -f -
```

> **Ghi chú:** Lệnh trên dùng công cụ dòng lệnh [`jq`](https://jqlang.github.io/jq/) để điền
> nội dung đã mã hóa base64 vào field `.status.certificate`. Nếu bạn không có `jq`, bạn cũng có
> thể lưu đầu ra JSON ra một file, tự điền field này thủ công, rồi tải file kết quả lên.

Sau khi CSR đã được phê duyệt và certificate đã ký được tải lên, hãy chạy:

```shell
kubectl get csr
```

Đầu ra sẽ tương tự như sau:
```none
NAME                  AGE   SIGNERNAME            REQUESTOR              REQUESTEDDURATION   CONDITION
my-svc.my-namespace   20m   example.com/serving   yourname@example.com   <none>              Approved,Issued
```

## Tải certificate về và sử dụng nó (Download the certificate and use it)

Bây giờ, với vai trò là người dùng đã gửi yêu cầu, bạn có thể tải certificate đã được cấp về và
lưu nó vào file `server.crt` bằng cách chạy lệnh sau:

```shell
kubectl get csr my-svc.my-namespace -o jsonpath='{.status.certificate}' \
    | base64 --decode > server.crt
```

Bây giờ bạn có thể đưa `server.crt` và `server-key.pem` vào một Secret để sau này gắn (mount)
vào một Pod (ví dụ, để dùng với một webserver phục vụ HTTPS).

```shell
kubectl create secret tls server --cert server.crt --key server-key.pem
```

```none
secret/server created
```

Cuối cùng, bạn có thể lưu `ca.pem` vào một ConfigMap và dùng nó làm gốc tin cậy (trust root) để
xác thực serving certificate:

```shell
kubectl create configmap example-serving-ca --from-file ca.crt=ca.pem
```

```none
configmap/example-serving-ca created
```

## Phê duyệt các CertificateSigningRequest (Approving CertificateSigningRequests) {#approving-certificate-signing-requests}

Một quản trị viên Kubernetes (với quyền phù hợp) có thể phê duyệt (hoặc từ chối) các
CertificateSigningRequest một cách thủ công bằng các lệnh `kubectl certificate approve` và
`kubectl certificate deny`. Tuy nhiên, nếu bạn định sử dụng API này ở quy mô lớn, bạn nên cân
nhắc viết một certificate controller tự động.

> **Thận trọng:** Khả năng phê duyệt CSR quyết định ai tin cậy ai trong môi trường của bạn. Khả
> năng phê duyệt CSR không nên được cấp một cách rộng rãi hay dễ dãi.
>
> Bạn nên bảo đảm rằng mình thực sự hiểu rõ cả các yêu cầu xác minh mà người phê duyệt phải thực
> hiện **lẫn** hậu quả của việc cấp phát một certificate cụ thể, trước khi bạn trao quyền
> `approve`.

Dù người phê duyệt là một máy hay là một con người dùng kubectl như ở trên, vai trò của _người
phê duyệt_ là xác minh rằng CSR thỏa mãn hai yêu cầu:

1. Subject của CSR kiểm soát private key đã dùng để ký CSR. Điều này xử lý mối đe dọa một bên
   thứ ba giả mạo thành một subject được ủy quyền. Trong ví dụ ở trên, bước này là xác minh
   rằng pod kiểm soát private key đã dùng để sinh CSR.
2. Subject của CSR được ủy quyền hoạt động trong ngữ cảnh được yêu cầu. Điều này xử lý mối đe
   dọa một subject không mong muốn tham gia vào cluster. Trong ví dụ ở trên, bước này là xác
   minh rằng pod được phép tham gia vào service được yêu cầu.

Chỉ khi và chỉ khi cả hai yêu cầu này được thỏa mãn thì người phê duyệt mới nên phê duyệt CSR;
ngược lại thì nên từ chối CSR.

Để biết thêm thông tin về việc phê duyệt certificate và kiểm soát truy cập, hãy đọc trang tham
khảo
[Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/).

## Cấu hình cluster của bạn để cung cấp khả năng ký (Configuring your cluster to provide signing) {#configuring-your-cluster-to-provide-signing}

Trang này giả định rằng đã có một signer được thiết lập để phục vụ API certificates. Kubernetes
controller manager cung cấp một cài đặt mặc định của signer. Để bật nó, hãy truyền các tham số
`--cluster-signing-cert-file` và `--cluster-signing-key-file` cho controller manager kèm theo
đường dẫn tới cặp khóa (keypair) của Certificate Authority của bạn.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở CP3:

1. Ba vai trong quy trình `certificates.k8s.io` là gì, và mỗi vai làm gì?
2. **Câu bẫy.** Bạn `kubectl certificate approve` xong và lệnh báo thành công. Certificate đã có
   chưa? Lấy ở đâu?
3. Một ứng dụng trong cluster cần tin tưởng serving certificate mà bạn vừa cấp bằng API này. Dùng
   `kube-root-ca.crt` có được không? Vì sao?
4. Người phê duyệt phải xác minh **hai** điều gì trước khi approve, và mỗi điều chống lại mối đe
   dọa nào?
5. Cluster lab của bạn muốn tự ký được CSR bằng signer mặc định. Bật bằng cách nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Người xin (requester)** tạo private key và CSR rồi gửi object CertificateSigningRequest.
   **Người phê duyệt (approver)** — người hoặc controller — quyết định chấp nhận hay từ chối.
   **Người ký (signer)** theo dõi các CSR mang đúng `signerName` của mình, kiểm tra chúng đã được
   phê duyệt, ký certificate rồi cập nhật `.status` của object.
2. **Chưa có.** Approve chỉ chuyển CSR sang trạng thái đã được phê duyệt và **đang chờ signer
   ký**. Certificate chỉ xuất hiện sau khi signer ghi nó vào **`.status.certificate`**; bạn lấy
   ra bằng cách đọc field đó rồi giải mã base64. Đây là chỗ dễ tưởng xong việc vì lệnh approve
   trả về thành công ngay.
3. **Không nên.** Certificate cấp qua API này được ký bởi một **CA chuyên dụng, không phải CA gốc
   của cluster**, nên không có gì bảo đảm chúng xác thực được với `kube-root-ca.crt`. Ngoài ra
   bài nói rõ: CA gốc của cluster chỉ nên dùng để xác thực **endpoint nội bộ của Kubernetes**.
   Cách đúng là tạo CA riêng cho workload và phân phối certificate của CA đó qua ConfigMap.
4. (1) **Subject kiểm soát private key đã ký CSR** — chống việc một bên thứ ba giả mạo thành
   subject hợp lệ. (2) **Subject được ủy quyền hoạt động trong ngữ cảnh nó yêu cầu** — chống việc
   một subject không mong muốn chen vào cluster. Chỉ approve khi **cả hai** cùng thỏa; thiếu một
   thì từ chối.
5. Truyền **`--cluster-signing-cert-file`** và **`--cluster-signing-key-file`** cho
   kube-controller-manager, trỏ tới cặp khóa CA của bạn — controller manager có sẵn một cài đặt
   signer mặc định, chỉ cần bật lên.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi sang bài
[397](397-certificate-issue-client-csr-vi.md).
