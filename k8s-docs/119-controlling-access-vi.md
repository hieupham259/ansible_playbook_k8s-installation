# Kiểm soát truy cập vào Kubernetes API (Controlling Access to the Kubernetes API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/controlling-access/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](LO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](LO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 4/18 · Kiểm chứng ở Lab 9a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Đây là **bài xương sống của giai đoạn 9**. Mọi bài còn lại đều gắn vào một chặng nào đó của
luồng mô tả ở đây: bài [123](123-hardening-authentication-vi.md) mở rộng chặng xác thực, bài
[120](120-rbac-good-practices-vi.md) mở rộng chặng phân quyền, các bài
[116](116-pod-security-admission-vi.md) và [173](173-admission-webhooks-vi.md) mở rộng chặng
admission. Bài chỉ dài hơn 170 dòng — đọc kỹ, và nhớ **đúng thứ tự**.

**Phải hiểu ở lần đọc này:**

- Luồng đầy đủ của một request: **TLS** (bảo mật tầng truyền tải) → **1. Xác thực** →
  **2. Phân quyền** → **3. Kiểm soát tiếp nhận** → **4. validate rồi ghi vào object store**.
  Hai chặng đầu từ chối bằng hai mã khác nhau: **401** ở xác thực, **403** ở phân quyền.
- Chặng xác thực: các module được thử **lần lượt cho đến khi một module thành công**. Kết quả
  là một `username` (một số module cho thêm thông tin nhóm). Kubernetes **không có object
  `User`** và không lưu username trong API — nó chỉ dùng username để ra quyết định và ghi log.
- Chặng phân quyền: request phải mang username + hành động + đối tượng chịu tác động. Khi có
  nhiều module, **chỉ cần một module cho phép** là request đi tiếp; **tất cả cùng từ chối** thì
  mới 403. RBAC chỉ là một trong các module, cạnh ABAC và Webhook.
- Chặng kiểm soát tiếp nhận **ngược lại**: chỉ cần **một** module từ chối là request bị từ chối
  ngay. Đây cũng là chặng duy nhất truy cập được **nội dung object** đang được tạo hoặc sửa,
  nên nó vừa từ chối được, vừa **đặt giá trị mặc định phức tạp cho các trường**.
- Ranh giới của chặng ba: admission controller tác động lên request **tạo, sửa đổi, xóa hoặc
  proxy** một đối tượng, và **không tác động lên request chỉ đọc**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Ví dụ policy ABAC và SubjectAccessReview dạng JSON của Bob | chỉ là minh họa khái niệm "policy"; cluster kubeadm dùng RBAC | bài [120](120-rbac-good-practices-vi.md) |
| `--secure-port`, `--bind-address`, `--tls-cert-file`, `--tls-private-key-file` | là thao tác cấu hình API server | giai đoạn 8, bài [03](03-control-plane-flags-vi.md) |
| Danh sách các module xác thực cụ thể và ưu nhược điểm | được so sánh chi tiết ở bài riêng | bài [123](123-hardening-authentication-vi.md) |
| Danh sách các module Admission Control khả dụng | là bảng tra cứu | bài [129](129-security-checklist-vi.md) |
| Mục *Kiểm toán* | audit policy và backend là thao tác cấu hình | CP7 audit/encryption |

---

Trang này cung cấp cái nhìn tổng quan về việc kiểm soát truy cập vào Kubernetes API.

Người dùng truy cập [Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) bằng `kubectl`,
các thư viện client, hoặc bằng cách gửi các request REST. Cả người dùng là con người lẫn
[service account của Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) đều có thể được
cấp quyền truy cập API.
Khi một request đến API, nó đi qua nhiều giai đoạn, được minh họa trong sơ đồ sau:

![Sơ đồ các bước xử lý một request gửi tới Kubernetes API](https://kubernetes.io/images/docs/admin/access-control-overview.svg)

## Bảo mật tầng truyền tải (Transport security)

Theo mặc định, Kubernetes API server lắng nghe trên port 6443 trên network interface đầu tiên
không phải localhost, được bảo vệ bằng TLS. Trong một cluster Kubernetes production điển hình,
API phục vụ trên port 443. Port này có thể được thay đổi bằng cờ `--secure-port`, và địa chỉ IP
lắng nghe bằng cờ `--bind-address`.

API server xuất trình một certificate. Certificate này có thể được ký bởi một tổ chức phát hành
chứng chỉ (certificate authority - CA) riêng, hoặc dựa trên một hạ tầng khóa công khai (public key
infrastructure) liên kết với một CA được công nhận rộng rãi. Certificate và khóa riêng (private key)
tương ứng có thể được thiết lập bằng các cờ `--tls-cert-file` và `--tls-private-key-file`.

Nếu cluster của bạn dùng một CA riêng, bạn cần một bản sao của certificate CA đó được cấu hình
trong `~/.kube/config` phía client, để bạn có thể tin tưởng kết nối và chắc chắn rằng nó
không bị chặn bắt giữa đường (intercepted).

Client của bạn có thể xuất trình TLS client certificate ở giai đoạn này.

## Xác thực (Authentication)

Sau khi TLS được thiết lập, request HTTP chuyển sang bước Xác thực (Authentication).
Đây là bước **1** trong sơ đồ.
Script tạo cluster hoặc quản trị viên cluster cấu hình API server để chạy
một hoặc nhiều module Xác thực (Authenticator).
Các Authenticator được mô tả chi tiết hơn tại
[Xác thực (Authentication)](https://kubernetes.io/docs/reference/access-authn-authz/authentication/).

Đầu vào của bước xác thực là toàn bộ request HTTP; tuy nhiên, bước này thường
chỉ xem xét các header và/hoặc client certificate.

Các module xác thực bao gồm client certificate, mật khẩu và token thuần (plain token),
bootstrap token, và JSON Web Token (dùng cho service account).

Có thể chỉ định nhiều module xác thực cùng lúc, khi đó từng module được thử lần lượt
cho đến khi một trong số chúng thành công.

Nếu request không thể được xác thực, nó bị từ chối với mã trạng thái HTTP 401.
Ngược lại, người dùng được xác thực dưới một `username` cụ thể, và tên người dùng này
sẵn sàng cho các bước tiếp theo sử dụng trong các quyết định của chúng. Một số authenticator
còn cung cấp thông tin thành viên nhóm (group membership) của người dùng, trong khi các
authenticator khác thì không.

Mặc dù Kubernetes sử dụng username cho các quyết định kiểm soát truy cập và trong việc ghi log request,
nó không có đối tượng `User` và cũng không lưu trữ username hay các thông tin khác về
người dùng trong API của mình.

## Phân quyền (Authorization)

Sau khi request được xác thực là đến từ một người dùng cụ thể, request đó phải được
phân quyền (authorize). Đây là bước **2** trong sơ đồ.

Một request phải bao gồm username của người gửi request, hành động được yêu cầu, và
đối tượng chịu tác động của hành động đó. Request được cho phép nếu tồn tại một chính sách (policy)
tuyên bố rằng người dùng có quyền hoàn tất hành động được yêu cầu.

Ví dụ, nếu Bob có chính sách bên dưới, thì anh ấy chỉ có thể đọc các pod trong namespace `projectCaribou`:

```json
{
    "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
    "kind": "Policy",
    "spec": {
        "user": "bob",
        "namespace": "projectCaribou",
        "resource": "pods",
        "readonly": true
    }
}
```

Nếu Bob gửi request sau, request được cho phép vì anh ấy được phép đọc
các đối tượng trong namespace `projectCaribou`:

```json
{
  "apiVersion": "authorization.k8s.io/v1beta1",
  "kind": "SubjectAccessReview",
  "spec": {
    "resourceAttributes": {
      "namespace": "projectCaribou",
      "verb": "get",
      "group": "unicorn.example.org",
      "resource": "pods"
    }
  }
}
```

Nếu Bob gửi request ghi (`create` hoặc `update`) vào các đối tượng trong namespace
`projectCaribou`, request của anh ấy bị từ chối phân quyền. Nếu Bob gửi request
đọc (`get`) các đối tượng trong một namespace khác như `projectFish`, request cũng bị từ chối phân quyền.

Phân quyền trong Kubernetes yêu cầu bạn sử dụng các thuộc tính REST chung để tương tác
với các hệ thống kiểm soát truy cập sẵn có ở quy mô toàn tổ chức hoặc toàn nhà cung cấp cloud.
Việc dùng định dạng REST là quan trọng vì các hệ thống kiểm soát này có thể
tương tác với các API khác ngoài Kubernetes API.

Kubernetes hỗ trợ nhiều module phân quyền, chẳng hạn chế độ ABAC, chế độ RBAC,
và chế độ Webhook. Khi tạo một cluster, quản trị viên cấu hình các module
phân quyền sẽ được sử dụng trong API server. Nếu có nhiều hơn một module
phân quyền được cấu hình, Kubernetes kiểm tra từng module, và nếu
bất kỳ module nào cho phép request, thì request được tiếp tục xử lý. Nếu tất cả
các module đều từ chối request, thì request bị từ chối (mã trạng thái HTTP 403).

Để tìm hiểu thêm về phân quyền trong Kubernetes, bao gồm chi tiết về việc tạo
chính sách bằng các module phân quyền được hỗ trợ, xem [Phân quyền (Authorization)](https://kubernetes.io/docs/reference/access-authn-authz/authorization/).

## Kiểm soát tiếp nhận (Admission control)

Các module Kiểm soát tiếp nhận (Admission Control) là các module phần mềm có thể sửa đổi hoặc từ chối request.
Ngoài các thuộc tính mà module Phân quyền có thể truy cập, các module Admission
Control còn có thể truy cập nội dung của đối tượng đang được tạo hoặc sửa đổi.

Các admission controller tác động lên các request tạo, sửa đổi, xóa, hoặc kết nối tới (proxy) một đối tượng.
Admission controller không tác động lên các request chỉ đọc đối tượng.
Khi nhiều admission controller được cấu hình, chúng được gọi theo thứ tự.

Đây là bước **3** trong sơ đồ.

Khác với các module Xác thực và Phân quyền, chỉ cần bất kỳ module admission controller nào
từ chối, request lập tức bị từ chối.

Ngoài việc từ chối đối tượng, các admission controller còn có thể thiết lập các giá trị
mặc định phức tạp cho các trường.

Các module Admission Control khả dụng được mô tả tại [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/).

Sau khi một request vượt qua tất cả admission controller, nó được kiểm tra tính hợp lệ (validate)
bằng các thủ tục kiểm tra dành cho đối tượng API tương ứng, rồi được ghi vào
kho lưu trữ đối tượng (object store) (bước **4** trong sơ đồ).

## Kiểm toán (Auditing)

Kiểm toán (auditing) trong Kubernetes cung cấp một tập bản ghi theo trình tự thời gian, liên quan đến bảo mật, ghi lại chuỗi các hành động trong một cluster.
Cluster kiểm toán các hoạt động được tạo ra bởi người dùng, bởi các ứng dụng sử dụng Kubernetes API, và bởi chính control plane.

Để biết thêm thông tin, xem [Kiểm toán (Auditing)](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/).

## Tiếp theo (What's next)

Đọc thêm tài liệu về xác thực, phân quyền và kiểm soát truy cập API:

- [Xác thực (Authenticating)](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
   - [Xác thực bằng Bootstrap Token (Authenticating with Bootstrap Tokens)](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/)
- [Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
   - [Kiểm soát tiếp nhận động (Dynamic Admission Control)](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
- [Phân quyền (Authorization)](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
   - [Kiểm soát truy cập dựa trên vai trò (Role Based Access Control)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
   - [Kiểm soát truy cập dựa trên thuộc tính (Attribute Based Access Control)](https://kubernetes.io/docs/reference/access-authn-authz/abac/)
   - [Phân quyền Node (Node Authorization)](https://kubernetes.io/docs/reference/access-authn-authz/node/)
   - [Phân quyền Webhook (Webhook Authorization)](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)
- [Yêu cầu ký certificate (Certificate Signing Requests)](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)
   - bao gồm [phê duyệt CSR (CSR approval)](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#approval-rejection)
     và [ký certificate (certificate signing)](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#signing)
- Service account
  - [Hướng dẫn cho nhà phát triển (Developer guide)](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
  - [Quản trị (Administration)](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)

Bạn có thể tìm hiểu về:
- cách các Pod có thể sử dụng
  [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#service-accounts-automatically-create-and-attach-secrets-with-api-credentials)
  để lấy thông tin xác thực (credential) truy cập API.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Sau khi TLS đã được thiết lập, một request đi qua **ba chặng** nào, theo đúng thứ tự? Với
   mỗi chặng, nói rõ nó từ chối request **vì lý do gì** và trả về **mã HTTP nào** nếu bài có
   nêu.
2. Cả chặng phân quyền lẫn chặng kiểm soát tiếp nhận đều cho phép cấu hình nhiều module. Quy
   tắc kết hợp của hai chặng này **ngược nhau** như thế nào?
3. `kubectl get pods` có đi qua các admission controller không? Vì sao?
4. Trên cluster lab, bạn dùng kubeconfig copy từ `/etc/kubernetes/admin.conf`. Theo bài, client
   certificate trong file đó được xuất trình ở bước nào, và thứ rút ra được từ nó dùng để làm
   gì ở chặng kế tiếp?
5. Admission controller nhìn thấy thứ gì mà module phân quyền không nhìn thấy, và điều đó cho
   nó làm thêm việc gì ngoài việc từ chối?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Xác thực → Phân quyền → Kiểm soát tiếp nhận** (các bước 1, 2, 3 trong sơ đồ), rồi mới tới
   bước 4 là validate và ghi vào object store.
   - **Xác thực** từ chối vì **không xác định được request đến từ ai** — không module nào nhận
     ra credential. Mã **HTTP 401**.
   - **Phân quyền** từ chối vì **người dùng đã biết đó không được phép làm hành động đó trên
     đối tượng đó** — không có policy nào cho phép. Mã **HTTP 403**.
   - **Kiểm soát tiếp nhận** từ chối vì **nội dung của chính object không hợp lệ theo chính
     sách**, chứ không phải vì người gửi là ai.
2. **Ngược nhau hoàn toàn.** Ở **phân quyền**, Kubernetes kiểm tra từng module và **chỉ cần một
   module cho phép** là request được tiếp tục; phải **tất cả** cùng từ chối thì mới 403. Ở
   **kiểm soát tiếp nhận**, bài nói rõ: khác với Xác thực và Phân quyền, **chỉ cần bất kỳ module
   nào từ chối là request lập tức bị từ chối**. Một bên là "hoặc", một bên là "và".
3. **Không.** Bài nói admission controller tác động lên các request **tạo, sửa đổi, xóa hoặc
   kết nối tới (proxy)** một đối tượng, và **không tác động lên các request chỉ đọc đối tượng**.
   `kubectl get pods` là request chỉ đọc nên dừng lại sau chặng phân quyền.
4. Client certificate được xuất trình **ở giai đoạn bảo mật tầng truyền tải**, ngay khi thiết
   lập TLS, rồi được **module xác thực bằng client certificate** dùng ở chặng 1. Kết quả của
   chặng 1 là một **`username`** (và có thể là thông tin nhóm), và chính username đó là thứ
   chặng **phân quyền** dùng để quyết định request có được phép hay không.
5. Admission controller truy cập được **nội dung của đối tượng đang được tạo hoặc sửa đổi** —
   thứ mà module phân quyền không có, vì phân quyền chỉ làm việc với username, hành động và đối
   tượng chịu tác động. Nhờ đó, ngoài việc từ chối, admission controller còn **thiết lập được
   các giá trị mặc định phức tạp cho các trường** của đối tượng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
