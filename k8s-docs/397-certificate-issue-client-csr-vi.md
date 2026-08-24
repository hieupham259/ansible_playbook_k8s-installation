# Cấp certificate cho một client của Kubernetes API bằng CertificateSigningRequest (Issue a Certificate for a Kubernetes API Client Using a CertificateSigningRequest)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/tls/certificate-issue-client-csr/>
>
> Trang này hướng dẫn cách tạo private key, dùng CertificateSigningRequest để xin cấp một
> certificate X.509, rồi cấu hình certificate đó cho một client của Kubernetes API.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Checkpoint tiếp nối — nhánh `/docs/tasks/`](00-ALO-TRINH-ADMIN.md#checkpoint-tiếp-nối--nhánh-docstasks)
→ [CP3 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#cp3--vòng-đời-chứng-chỉ), bài 6/7 · Kiểm chứng
trên cluster lab: tạo trọn một danh tính `myuser`, xác nhận bằng `kubectl --context myuser auth
whoami`, rồi chứng minh nó **chưa làm được gì** cho tới khi có RoleBinding.

Đây là bài trả lời câu hỏi mà giai đoạn 9 để ngỏ: **Kubernetes không có object User** — vậy một
người dùng thật được tạo ra bằng cách nào. Nối tiếp bài [119](119-controlling-access-vi.md),
[120](120-rbac-good-practices-vi.md) và [111](111-kubeconfig-vi.md).

**Phải hiểu ở lần đọc này:**

- **Danh tính người dùng nằm trong chính certificate**: trường **CN là username**, trường **O là
  group**. Cluster không lưu user ở đâu cả — nó đọc hai trường này từ certificate mà client trình
  ra. Đây là ý cốt lõi của cả bài.
- Phân biệt hai thứ trùng tên: **file CSR X.509** (do `openssl` tạo) **không phải** object
  **CertificateSigningRequest** của Kubernetes. File X.509 được base64 rồi nhét vào trường
  `request` của object đó — bài có hẳn một ghi chú cảnh báo chỗ này.
- Ba trường bắt buộc của object CSR: `signerName: kubernetes.io/kube-apiserver-client`,
  `usages` **phải là** `client auth`, và `expirationSeconds` — đặt dài ngắn tùy ý nhưng
  **không được ngắn hơn 10 phút**.
- Luồng đầy đủ: `openssl genrsa` → `openssl req` tạo CSR → `base64` → tạo object
  CertificateSigningRequest → **`kubectl certificate approve`** (ở đây là phê duyệt **thủ công**,
  khác bài [398](398-certificate-rotation-vi.md) nơi controller manager tự duyệt) → lấy
  `.status.certificate` rồi `base64 -d` → `kubectl config set-credentials` + `set-context`.
- **Certificate chỉ lo xác thực, không lo phân quyền.** Sau khi `auth whoami` trả về `myuser`,
  người dùng vẫn chưa làm được gì; phải tạo Role và RoleBinding thì mới có quyền. Và **private
  key phải giữ bí mật** — ai có nó thì mạo danh được người dùng đó.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách group chuẩn của RBAC (`system:*`) | thuộc nhánh `/docs/reference/`, chưa có bản dịch | bài [120](120-rbac-good-practices-vi.md) đã đọc ở giai đoạn 9 |
| Tham chiếu API đầy đủ của CertificateSigningRequest | chưa có bản dịch; ở đây chỉ cần ba trường nêu trên | khi cần tự động hóa việc cấp certificate |
| Trường hợp cluster **không** dùng RBAC | cluster lab dùng RBAC mặc định | không áp dụng cho lộ trình này |

---

Kubernetes cho phép bạn dùng hạ tầng khóa công khai (public key infrastructure - PKI) để xác
thực với cluster của bạn với tư cách một client.

Cần vài bước để một người dùng thông thường (normal user) có thể xác thực và gọi được API.
Trước hết, người dùng này phải có một certificate [X.509](https://www.itu.int/rec/T-REC-X.509)
được cấp bởi một tổ chức phát hành (authority) mà Kubernetes cluster của bạn tin tưởng. Sau đó
client phải trình certificate đó cho Kubernetes API.

Trong quá trình này bạn sẽ dùng một
[CertificateSigningRequest](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/),
và hoặc bạn hoặc một principal nào đó phải phê duyệt (approve) yêu cầu này.

Bạn sẽ tạo một private key, sau đó xin cấp một certificate, và cuối cùng cấu hình private key
đó cho một client.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một Kubernetes cluster, và công cụ dòng lệnh kubectl phải được cấu hình để giao
  tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
  không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
  bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
  các sân chơi (playground) Kubernetes sau:

  * [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  * [KodeKloud](https://kodekloud.com/public-playgrounds)

* Bạn cần các tiện ích `kubectl`, `openssl` và `base64`.

Trang này giả định bạn đang dùng cơ chế kiểm soát truy cập dựa trên vai trò (role based access
control - RBAC) của Kubernetes. Nếu bạn có cơ chế bảo mật thay thế hoặc bổ sung khác cho việc
phân quyền (authorization), bạn cũng cần tính đến chúng.

## Tạo private key (Create private key)

Ở bước này, bạn tạo một private key. Bạn cần giữ bí mật private key này; bất kỳ ai có nó đều có
thể mạo danh (impersonate) người dùng đó.

```shell
# Tạo một private key
openssl genrsa -out myuser.key 3072
```

## Tạo một certificate signing request X.509 (Create an X.509 certificate signing request) {#create-x.509-certificatessigningrequest}

> **Ghi chú:** Đây không phải là API CertificateSigningRequest có tên gần giống; file bạn tạo ra
> ở đây sẽ được đưa vào bên trong CertificateSigningRequest.

Việc thiết lập các thuộc tính CN và O của CSR là rất quan trọng. CN là tên của người dùng, còn O
là group mà người dùng này sẽ thuộc về. Bạn có thể tham khảo
[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) để biết các group chuẩn.

```shell
# Đổi common name "myuser" thành tên người dùng thực tế mà bạn muốn dùng
openssl req -new -key myuser.key -out myuser.csr -subj "/CN=myuser"
```

## Tạo một CertificateSigningRequest của Kubernetes (Create a Kubernetes CertificateSigningRequest) {#create-k8s-certificatessigningrequest}

Mã hóa (encode) tài liệu CSR bằng lệnh sau:

```shell
cat myuser.csr | base64 | tr -d "\n"
```

Tạo một
[CertificateSigningRequest](https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/certificate-signing-request-v1/)
và gửi nó tới một Kubernetes cluster thông qua kubectl. Dưới đây là một đoạn shell mà bạn có thể
dùng để sinh ra CertificateSigningRequest.

```shell
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser # example
spec:
  # This is an encoded CSR. Change this to the base64-encoded contents of myuser.csr
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
EOF
```

Ngoài cách trên, bạn cũng có thể tạo một file manifest YAML rồi áp dụng nó bằng `kubectl`:

[`tls/myuser.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/tls/myuser.yaml)

```yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: myuser # example
spec:
  # This is an encoded CSR. Change this to the base64-encoded contents of myuser.csr
  request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZqQ0NBVDRDQVFBd0VURVBNQTBHQTFVRUF3d0dZVzVuWld4aE1JSUJJakFOQmdrcWhraUc5dzBCQVFFRgpBQU9DQVE4QU1JSUJDZ0tDQVFFQTByczhJTHRHdTYxakx2dHhWTTJSVlRWMDNHWlJTWWw0dWluVWo4RElaWjBOCnR2MUZtRVFSd3VoaUZsOFEzcWl0Qm0wMUFSMkNJVXBGd2ZzSjZ4MXF3ckJzVkhZbGlBNVhwRVpZM3ExcGswSDQKM3Z3aGJlK1o2MVNrVHF5SVBYUUwrTWM5T1Nsbm0xb0R2N0NtSkZNMUlMRVI3QTVGZnZKOEdFRjJ6dHBoaUlFMwpub1dtdHNZb3JuT2wzc2lHQ2ZGZzR4Zmd4eW8ybmlneFNVekl1bXNnVm9PM2ttT0x1RVF6cXpkakJ3TFJXbWlECklmMXBMWnoyalVnald4UkhCM1gyWnVVV1d1T09PZnpXM01LaE8ybHEvZi9DdS8wYk83c0x0MCt3U2ZMSU91TFcKcW90blZtRmxMMytqTy82WDNDKzBERHk5aUtwbXJjVDBnWGZLemE1dHJRSURBUUFCb0FBd0RRWUpLb1pJaHZjTgpBUUVMQlFBRGdnRUJBR05WdmVIOGR4ZzNvK21VeVRkbmFjVmQ1N24zSkExdnZEU1JWREkyQTZ1eXN3ZFp1L1BVCkkwZXpZWFV0RVNnSk1IRmQycVVNMjNuNVJsSXJ3R0xuUXFISUh5VStWWHhsdnZsRnpNOVpEWllSTmU3QlJvYXgKQVlEdUI5STZXT3FYbkFvczFqRmxNUG5NbFpqdU5kSGxpT1BjTU1oNndLaTZzZFhpVStHYTJ2RUVLY01jSVUyRgpvU2djUWdMYTk0aEpacGk3ZnNMdm1OQUxoT045UHdNMGM1dVJVejV4T0dGMUtCbWRSeEgvbUNOS2JKYjFRQm1HCkkwYitEUEdaTktXTU0xMzhIQXdoV0tkNjVoVHdYOWl4V3ZHMkh4TG1WQzg0L1BHT0tWQW9FNkpsYWFHdTlQVmkKdjlOSjVaZlZrcXdCd0hKbzZXdk9xVlA3SVFjZmg3d0drWm89Ci0tLS0tRU5EIENFUlRJRklDQVRFIFJFUVVFU1QtLS0tLQo=
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400 # one day
  usages:
  - client auth
```

Áp dụng manifest:

```bash
kubectl apply -f myuser.yaml --server-side
```

Một vài điểm cần lưu ý:

- `usages` bắt buộc phải là `client auth`
- `expirationSeconds` có thể đặt dài hơn (ví dụ `864000` cho mười ngày) hoặc ngắn hơn (ví dụ
  `3600` cho một giờ). Bạn không thể yêu cầu thời hạn ngắn hơn 10 phút.
- `request` là giá trị đã được mã hóa base64 của nội dung file CSR.

## Phê duyệt CertificateSigningRequest (Approve the CertificateSigningRequest) {#approve-certificate-signing-request}

Dùng kubectl để tìm CSR bạn vừa tạo, rồi phê duyệt (approve) nó một cách thủ công.

Lấy danh sách các CSR:

```shell
kubectl get csr
```

Phê duyệt CSR:

```shell
kubectl certificate approve myuser
```

## Lấy certificate (Get the certificate)

Truy xuất certificate từ CSR để kiểm tra xem nó có ổn không.

```shell
kubectl get csr/myuser -o yaml
```

Giá trị certificate nằm ở định dạng mã hóa Base64, trong trường `.status.certificate`.

Xuất certificate đã được cấp ra khỏi CertificateSigningRequest.

```shell
kubectl get csr myuser -o jsonpath='{.status.certificate}'| base64 -d > myuser.crt
```

## Cấu hình certificate vào kubeconfig (Configure the certificate into kubeconfig)

Bước tiếp theo là thêm người dùng này vào file kubeconfig.

Trước hết, bạn cần thêm thông tin xác thực (credentials) mới:

```shell
kubectl config set-credentials myuser --client-key=myuser.key --client-certificate=myuser.crt --embed-certs=true

```

Sau đó, bạn cần thêm context:

```shell
kubectl config set-context myuser --cluster=kubernetes --user=myuser
```

Để kiểm thử:

```shell
kubectl --context myuser auth whoami
```

Bạn sẽ thấy output xác nhận rằng bạn chính là "myuser".

## Tạo Role và RoleBinding (Create Role and RoleBinding)

> **Ghi chú:** Nếu bạn không dùng RBAC của Kubernetes, hãy bỏ qua bước này và thực hiện các thay
> đổi phù hợp với cơ chế phân quyền (authorization) mà cluster của bạn đang thực sự dùng.

Sau khi certificate đã được tạo, đã đến lúc định nghĩa Role và RoleBinding để người dùng này
truy cập được các tài nguyên của Kubernetes cluster.

Đây là một lệnh mẫu để tạo Role cho người dùng mới này:

```shell
kubectl create role developer --verb=create --verb=get --verb=list --verb=update --verb=delete --resource=pods
```

YAML tương đương:

[`tls/role.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/tls/role.yaml)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "get", "list", "update", "delete"]
```

Đây là một lệnh mẫu để tạo RoleBinding cho người dùng mới này:

```shell
kubectl create rolebinding developer-binding-myuser --role=developer --user=myuser
```

YAML tương đương:

[`tls/rolebinding.yaml`](https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/tls/rolebinding.yaml)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding-myuser
subjects:
- kind: User
  name: myuser
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

## Tiếp theo (What's next)

* Đọc [Quản lý TLS certificate trong một cluster](399-managing-tls-in-a-cluster-vi.md)
* Để biết chi tiết về bản thân chuẩn X.509, hãy tham khảo mục 3.1 của
  [RFC 5280](https://tools.ietf.org/html/rfc5280#section-3.1)
* Để biết thông tin về cú pháp của certificate signing request theo PKCS#10, hãy tham khảo
  [RFC 2986](https://tools.ietf.org/html/rfc2986)
* Đọc về [ClusterTrustBundles](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#cluster-trust-bundles)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở CP3:

1. Kubernetes không có object `User`. Vậy khi bạn gọi API bằng certificate, cluster lấy **tên
   người dùng** và **group** của bạn từ đâu?
2. **Câu bẫy.** Bạn chạy `openssl req` và được file `myuser.csr`. Vậy bạn đã tạo xong một
   CertificateSigningRequest của Kubernetes chưa?
3. Bạn đã `kubectl certificate approve myuser`, đã cấu hình kubeconfig, và
   `kubectl --context myuser auth whoami` trả về đúng `myuser`. Bây giờ `myuser` liệt kê được Pod
   chưa? Vì sao?
4. Ai phê duyệt CSR trong bài này, và điều đó khác gì so với CSR của kubelet ở bài
   [398](398-certificate-rotation-vi.md)?
5. Bạn muốn certificate chỉ sống 5 phút cho một phiên thao tác ngắn. Đặt `expirationSeconds: 300`
   được không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Từ **chính certificate mà client trình ra**: trường **CN (common name) là username**, trường
   **O (organization) là group**. Không có nơi nào trong cluster lưu danh sách người dùng — đó là
   lý do "tạo user" trong Kubernetes thực chất là "cấp một certificate có CN và O phù hợp".
2. **Chưa.** `myuser.csr` là **file CSR X.509**, một tài liệu mật mã học. Object
   **CertificateSigningRequest của Kubernetes** là một tài nguyên API riêng; bạn phải base64 nội
   dung file đó rồi đưa vào trường `request` của object. Bài đặt hẳn một ghi chú ở đây vì hai thứ
   tên gần giống nhau và rất dễ tưởng là một.
3. **Chưa làm được gì.** Certificate chỉ giải quyết **xác thực (authentication)** — cluster biết
   bạn là ai. Việc bạn **được phép làm gì** là **phân quyền (authorization)**, do RBAC quyết
   định. Phải tạo Role và RoleBinding gắn vào `myuser` thì mới liệt kê được Pod. `auth whoami`
   trả lời đúng tên chính là bằng chứng authn đã xong và authz thì chưa.
4. Trong bài này **bạn phê duyệt thủ công** bằng `kubectl certificate approve myuser`. Ở bài 398,
   CSR của kubelet được **kube-controller-manager tự phê duyệt** khi thỏa các tiêu chí. Cùng một
   cơ chế CSR API, khác nhau ở chỗ ai đóng vai người duyệt.
5. **Không.** Bài nói rõ **không thể yêu cầu thời hạn ngắn hơn 10 phút**; `expirationSeconds: 300`
   nằm dưới ngưỡng đó. Giá trị hợp lệ gần nhất là `600`.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi sang bài cuối của [CP3 — Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#cp3--vòng-đời-chứng-chỉ) —
[400](400-manual-rotation-of-ca-certificates-vi.md).
