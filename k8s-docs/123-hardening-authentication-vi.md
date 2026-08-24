# Hướng dẫn tăng cường bảo mật - Các cơ chế xác thực (Hardening Guide - Authentication Mechanisms)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/hardening-guide/authentication-mechanisms/>
>
> Thông tin về các tùy chọn xác thực trong Kubernetes và các đặc tính bảo mật của chúng.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 10/18 · Kiểm chứng ở Lab 9b (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài [119](119-controlling-access-vi.md) chỉ nói "có nhiều module xác thực, thử lần lượt". Bài
này mở đúng chỗ đó ra và **so sánh từng cơ chế theo nhược điểm**, không phải theo cách cấu hình.
Đọc như một bảng đối chiếu: mỗi mục trả lời câu "vì sao cơ chế này không nên dùng để xác thực
người dùng". Nó cũng giải thích trực tiếp vì sao kubeconfig admin của cluster lab là một
credential nguy hiểm.

**Phải hiểu ở lần đọc này:**

- Nguyên tắc nền: **bật càng ít cơ chế xác thực càng tốt**. Kubernetes **không có cơ sở dữ liệu
  người dùng bên trong cluster**; nó lấy thông tin người dùng từ hệ thống xác thực đã cấu hình,
  nên muốn kiểm toán quyền truy cập thì phải rà **mọi nguồn xác thực đã bật**.
- Với cluster production có nhiều người dùng truy cập trực tiếp API: khuyến nghị dùng **nguồn
  bên ngoài như OIDC**; client certificate và token của service account **không phù hợp** cho
  trường hợp này.
- Vì sao client certificate X.509 không hợp để xác thực người dùng: **không thu hồi được từng
  cái riêng lẻ** — muốn vô hiệu thì phải **re-key cả certificate authority**, kéo theo rủi ro về
  tính sẵn sàng; **không có bản ghi lưu vĩnh viễn** các certificate đã cấp; private key **không
  đặt được mật khẩu**; phải kết nối thẳng tới API server, không được có TLS termination trung
  gian; và **dữ liệu nhóm nằm trong giá trị `O` của certificate**, nên tư cách thành viên nhóm
  không đổi được trong suốt vòng đời certificate.
- Vì sao file token tĩnh và bootstrap token không dùng cho người dùng: token nằm **dạng văn bản
  thuần trên đĩa node control plane**; đổi token phải **khởi động lại API server**; người dùng
  **không tự xoay vòng được**; và **không có cơ chế khóa tài khoản chống brute-force**.
  Bootstrap token còn có nhóm gán cứng, và vốn chỉ để join node.
- Với các cơ chế bên ngoài: phần mềm hỗ trợ OIDC hoặc webhook chạy với **đặc quyền cao**, phải
  cách ly khỏi workload thường; token nên **ngắn hạn**. Riêng **proxy xác thực** có hai điều
  kiện bắt buộc — TLS cấu hình an toàn giữa proxy và API server, và bảo vệ header, vì **ai sửa
  được header là giành được quyền truy cập trái phép**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Token dạng Secret của ServiceAccount* và *Token của TokenRequest API* | đã học ở bài service account | bài [118](118-service-accounts-vi.md) |
| Cách tích hợp một identity provider OIDC cụ thể | cần dựng identity provider ngoài cluster | không cần |
| Cấu hình xác thực webhook trên hệ thống file của control plane | là thao tác sửa cấu hình API server | giai đoạn 8, bài [03](03-control-plane-flags-vi.md) |

---

Việc lựa chọn (các) cơ chế xác thực phù hợp là một khía cạnh then chốt trong việc bảo mật cluster của bạn.
Kubernetes cung cấp sẵn nhiều cơ chế, mỗi cơ chế có những điểm mạnh và điểm yếu riêng
cần được cân nhắc kỹ lưỡng khi chọn cơ chế xác thực tốt nhất cho cluster của bạn.

Nhìn chung, bạn nên bật càng ít cơ chế xác thực càng tốt để đơn giản hóa
việc quản lý người dùng và ngăn ngừa trường hợp người dùng vẫn giữ được quyền truy cập vào cluster dù không còn cần nữa.

Điều quan trọng cần lưu ý là Kubernetes không có cơ sở dữ liệu người dùng tích hợp sẵn bên trong cluster.
Thay vào đó, Kubernetes lấy thông tin người dùng từ hệ thống xác thực đã được cấu hình và dùng thông tin đó để đưa ra
các quyết định phân quyền (authorization). Do đó, để kiểm toán (audit) quyền truy cập của người dùng, bạn cần rà soát
thông tin xác thực (credential) từ mọi nguồn xác thực đã được cấu hình.

Đối với các cluster production có nhiều người dùng truy cập trực tiếp vào Kubernetes API, khuyến nghị là
sử dụng các nguồn xác thực bên ngoài như OIDC. Các cơ chế xác thực nội bộ,
chẳng hạn như client certificate và token của service account, được mô tả bên dưới, không
phù hợp cho trường hợp sử dụng này.

## Xác thực bằng client certificate X.509 (X.509 client certificate authentication) {#x509-client-certificate-authentication}

Kubernetes tận dụng cơ chế xác thực bằng [client certificate X.509](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#x509-client-certificates)
cho các thành phần hệ thống, chẳng hạn như khi kubelet xác thực với API Server.
Mặc dù cơ chế này cũng có thể được dùng để xác thực người dùng, nó có thể không phù hợp cho
môi trường production do một số hạn chế sau:

- Client certificate không thể bị thu hồi (revoke) từng cái riêng lẻ. Một khi bị lộ, certificate có thể bị
  kẻ tấn công sử dụng cho đến khi hết hạn. Để giảm thiểu rủi ro này, khuyến nghị cấu hình thời hạn
  sống ngắn cho các thông tin xác thực người dùng được tạo bằng client certificate.
- Nếu cần vô hiệu hóa một certificate, tổ chức phát hành chứng chỉ (certificate authority) phải được thay khóa (re-key), điều này
  có thể gây ra rủi ro về tính sẵn sàng (availability) cho cluster.
- Không có bản ghi lưu vĩnh viễn nào về các client certificate đã được tạo trong cluster. Do đó, tất cả
  certificate đã cấp phát phải được ghi lại nếu bạn cần theo dõi chúng.
- Khóa riêng (private key) dùng cho xác thực bằng client certificate không thể được bảo vệ bằng mật khẩu. Bất kỳ ai
  đọc được file chứa khóa đều có thể sử dụng nó.
- Việc sử dụng xác thực bằng client certificate yêu cầu kết nối trực tiếp từ client đến
  API server mà không có bất kỳ điểm kết thúc TLS (TLS termination) trung gian nào, điều này có thể làm phức tạp kiến trúc mạng.
- Dữ liệu nhóm (group) được nhúng trong giá trị `O` của client certificate, nghĩa là tư cách thành viên nhóm
  của người dùng không thể thay đổi trong suốt vòng đời của certificate.

## File token tĩnh (Static token file) {#static-token-file}

Mặc dù Kubernetes cho phép bạn nạp thông tin xác thực từ một
[file token tĩnh](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#static-token-file) nằm
trên ổ đĩa của các node control plane, cách tiếp cận này không được khuyến nghị cho các máy chủ production vì
một số lý do sau:

- Thông tin xác thực được lưu dưới dạng văn bản thuần (clear text) trên ổ đĩa của node control plane, điều này có thể là một rủi ro bảo mật.
- Việc thay đổi bất kỳ thông tin xác thực nào đều yêu cầu khởi động lại tiến trình API server để có hiệu lực, điều này có thể
  ảnh hưởng đến tính sẵn sàng.
- Không có cơ chế nào cho phép người dùng tự xoay vòng (rotate) thông tin xác thực của họ. Để xoay vòng một
  thông tin xác thực, quản trị viên cluster phải sửa token trên đĩa và phân phối nó cho người dùng.
- Không có cơ chế khóa tài khoản (lockout) nào để ngăn chặn các cuộc tấn công dò mật khẩu vét cạn (brute-force).

## Bootstrap token (Bootstrap tokens) {#bootstrap-tokens}

[Bootstrap token](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/) được dùng để join
node vào cluster và không được khuyến nghị cho việc xác thực người dùng vì một số lý do sau:

- Chúng có tư cách thành viên nhóm được gán cứng (hard-coded) không phù hợp cho mục đích sử dụng chung, khiến chúng
  không thích hợp cho mục đích xác thực.
- Việc tạo bootstrap token thủ công có thể dẫn đến các token yếu mà kẻ tấn công có thể đoán được,
  điều này có thể là một rủi ro bảo mật.
- Không có cơ chế khóa tài khoản nào để ngăn chặn tấn công brute-force, khiến kẻ tấn công
  dễ dàng đoán hoặc bẻ khóa token hơn.

## Token dạng Secret của ServiceAccount (ServiceAccount secret tokens) {#serviceaccount-secret-tokens}

[Secret của service account](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#manual-secret-management-for-serviceaccounts)
là một tùy chọn cho phép các workload chạy trong cluster xác thực với
API server. Trong Kubernetes < 1.23, đây là tùy chọn mặc định, tuy nhiên chúng đang được thay thế
bằng token của TokenRequest API. Mặc dù các secret này có thể được dùng để xác thực người dùng, chúng
nhìn chung không phù hợp vì một số lý do:

- Chúng không thể được đặt thời hạn hết hạn và sẽ vẫn còn hiệu lực cho đến khi service account liên quan bị xóa.
- Các token xác thực này hiển thị với bất kỳ người dùng cluster nào có quyền đọc secret trong namespace
  mà chúng được định nghĩa.
- Service account không thể được thêm vào các nhóm tùy ý, làm phức tạp việc quản lý RBAC ở những nơi chúng được sử dụng.

## Token của TokenRequest API (TokenRequest API tokens) {#tokenrequest-api-tokens}

TokenRequest API là một công cụ hữu ích để tạo các thông tin xác thực có thời hạn ngắn cho việc xác thực
dịch vụ với API server hoặc các hệ thống bên thứ ba. Tuy nhiên, nó nhìn chung không được khuyến nghị
cho việc xác thực người dùng vì không có phương thức thu hồi nào, và việc phân phối thông tin xác thực
cho người dùng một cách an toàn có thể là một thách thức.

Khi sử dụng token TokenRequest cho việc xác thực dịch vụ, khuyến nghị đặt thời hạn
sống ngắn để giảm tác động khi token bị lộ.

## Xác thực bằng token OpenID Connect (OpenID Connect token authentication) {#openid-connect-token-authentication}

Kubernetes hỗ trợ tích hợp các dịch vụ xác thực bên ngoài với Kubernetes API bằng
[OpenID Connect (OIDC)](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens).
Có rất nhiều phần mềm có thể được sử dụng để tích hợp Kubernetes với một nhà cung cấp danh tính
(identity provider). Tuy nhiên, khi sử dụng xác thực OIDC trong Kubernetes, điều quan trọng là cần cân nhắc
các biện pháp tăng cường bảo mật sau:

- Phần mềm được cài đặt trong cluster để hỗ trợ xác thực OIDC nên được cách ly khỏi
  các workload thông thường vì nó sẽ chạy với đặc quyền cao.
- Một số dịch vụ Kubernetes được quản lý (managed service) bị giới hạn về các nhà cung cấp OIDC có thể sử dụng.
- Giống như token TokenRequest, token OIDC nên có thời hạn sống ngắn để giảm tác động khi
  token bị lộ.

## Xác thực token bằng webhook (Webhook token authentication) {#webhook-token-authentication}

[Xác thực token bằng webhook](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)
là một tùy chọn khác để tích hợp các nhà cung cấp xác thực bên ngoài vào Kubernetes. Cơ chế này
cho phép một dịch vụ xác thực, chạy bên trong cluster hoặc bên ngoài, được
liên hệ để đưa ra quyết định xác thực thông qua một webhook. Điều quan trọng cần lưu ý là mức độ phù hợp
của cơ chế này nhiều khả năng sẽ phụ thuộc vào phần mềm được dùng cho dịch vụ xác thực, và có
một số điểm cần cân nhắc riêng đối với Kubernetes.

Để cấu hình xác thực Webhook, cần có quyền truy cập vào hệ thống file của máy chủ control plane. Điều này
có nghĩa là sẽ không thể thực hiện được với Managed Kubernetes trừ khi nhà cung cấp chủ động hỗ trợ
tính năng này. Ngoài ra, bất kỳ phần mềm nào được cài đặt trong cluster để hỗ trợ quyền truy cập này nên được
cách ly khỏi các workload thông thường, vì nó sẽ chạy với đặc quyền cao.

## Proxy xác thực (Authenticating proxy) {#authenticating-proxy}

Một tùy chọn khác để tích hợp các hệ thống xác thực bên ngoài vào Kubernetes là sử dụng
[proxy xác thực (authenticating proxy)](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#authenticating-proxy).
Với cơ chế này, Kubernetes kỳ vọng nhận các yêu cầu từ proxy kèm theo các giá trị header
cụ thể, cho biết tên người dùng và tư cách thành viên nhóm để gán cho mục đích phân quyền.
Điều quan trọng cần lưu ý là có những điểm cần cân nhắc riêng khi sử dụng
cơ chế này.

Thứ nhất, phải sử dụng TLS được cấu hình an toàn giữa proxy và Kubernetes API server để
giảm thiểu rủi ro bị chặn bắt lưu lượng hoặc các cuộc tấn công nghe lén (sniffing). Điều này đảm bảo rằng giao tiếp
giữa proxy và Kubernetes API server được an toàn.

Thứ hai, cần lưu ý rằng kẻ tấn công có khả năng sửa đổi các header của
yêu cầu có thể giành được quyền truy cập trái phép vào các tài nguyên Kubernetes. Do đó, điều quan trọng là
phải đảm bảo các header được bảo vệ đúng cách và không thể bị can thiệp.

## Tiếp theo (What's next)

- [Xác thực người dùng (User Authentication)](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Xác thực với Bootstrap Token (Authenticating with Bootstrap Tokens)](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/)
- [Xác thực kubelet (kubelet Authentication)](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/#kubelet-authentication)
- [Xác thực với token của Service Account (Authenticating with Service Account Tokens)](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#bound-service-account-tokens)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Trên cluster lab bạn đăng nhập bằng kubeconfig copy từ `/etc/kubernetes/admin.conf`. Đó là
   cơ chế xác thực nào? Nếu file đó bị lộ, theo bài bạn thu hồi bằng cách nào, và vì sao đó là
   chuyện phiền phức?
2. Bạn muốn chuyển một người dùng sang nhóm khác để đổi quyền của họ. Với client certificate,
   phải làm gì? Vì sao không sửa nhanh được như sửa một RoleBinding?
3. Vì sao bài khuyên bật càng ít cơ chế xác thực càng tốt? Nêu hệ quả cụ thể khi cluster có
   nhiều nguồn xác thực cùng bật.
4. Hai cơ chế nào bài nói **không có cơ chế khóa tài khoản** chống tấn công brute-force, và với
   loại nào thì điều đó đặc biệt đáng lo?
5. Dùng proxy xác thực thì hai điều kiện bắt buộc là gì, và nếu thiếu điều kiện thứ hai thì kẻ
   tấn công làm được gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Đó là **xác thực bằng client certificate X.509**. Bài nói **client certificate không thể bị
   thu hồi từng cái riêng lẻ**: một khi bị lộ, certificate dùng được **cho tới khi hết hạn**.
   Muốn vô hiệu hóa nó thì **certificate authority phải được thay khóa (re-key)** — việc này
   **có thể gây rủi ro về tính sẵn sàng cho cluster**, vì mọi thứ đang tin CA đó đều bị ảnh
   hưởng. Vì vậy khuyến nghị của bài là để thời hạn sống ngắn cho credential dạng này.
2. Phải **cấp một certificate mới**. **Dữ liệu nhóm được nhúng trong giá trị `O` của client
   certificate**, nghĩa là **tư cách thành viên nhóm không thể thay đổi trong suốt vòng đời của
   certificate**. Trực giác "quyền nằm ở RBAC nên sửa binding là xong" sai ở đây, vì thứ quyết
   định người dùng thuộc nhóm nào không nằm trong cluster mà nằm **bên trong certificate**.
3. Để **đơn giản hóa việc quản lý người dùng và ngăn trường hợp người dùng vẫn giữ được quyền
   truy cập cluster dù không còn cần nữa**. Hệ quả cụ thể khi có nhiều nguồn: Kubernetes **không
   có cơ sở dữ liệu người dùng bên trong cluster**, nên để kiểm toán ai đang truy cập được, bạn
   phải **rà soát thông tin xác thực từ mọi nguồn xác thực đã cấu hình** — bỏ sót một nguồn là
   bỏ sót cả một tập người dùng.
4. **File token tĩnh** và **bootstrap token**. Đáng lo nhất với **bootstrap token tạo thủ công**,
   vì bài nói việc tạo thủ công có thể sinh ra **token yếu mà kẻ tấn công đoán được** — không có
   khóa tài khoản thì kẻ tấn công cứ thử đến khi trúng.
5. Thứ nhất, **phải dùng TLS được cấu hình an toàn giữa proxy và Kubernetes API server**, để
   giảm rủi ro bị chặn bắt hoặc nghe lén lưu lượng. Thứ hai, **các header phải được bảo vệ và
   không thể bị can thiệp**. Thiếu điều kiện thứ hai, kẻ tấn công **sửa được header của yêu cầu
   sẽ giành được quyền truy cập trái phép vào tài nguyên Kubernetes** — vì Kubernetes tin thẳng
   tên người dùng và nhóm mà header khai báo.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
