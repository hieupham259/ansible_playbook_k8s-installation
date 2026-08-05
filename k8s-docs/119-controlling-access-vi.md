# Kiểm soát truy cập vào Kubernetes API (Controlling Access to the Kubernetes API)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/controlling-access/>

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
