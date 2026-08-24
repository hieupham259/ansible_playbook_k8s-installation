# Các thực hành tốt về kiểm soát truy cập dựa trên vai trò (Role Based Access Control Good Practices)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/rbac-good-practices/>
>
> Các nguyên tắc và thực hành thiết kế RBAC tốt dành cho người vận hành cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 5/18 · Kiểm chứng ở Lab 9a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này **không dạy cú pháp RBAC**. Nó giả định bạn đã biết Role, ClusterRole, RoleBinding và
ClusterRoleBinding là gì, rồi đi thẳng vào chỗ RBAC dễ bị dùng sai: các quyền nghe vô hại nhưng
thực chất cho phép leo thang đặc quyền. Cú pháp cụ thể nằm ở trang RBAC gốc mà bài liên kết —
bạn sẽ gõ nó ở Lab 9a. Đây là bài cuối của phần truy cập trong giai đoạn 9.

**Phải hiểu ở lần đọc này:**

- Bốn quy tắc đặc quyền tối thiểu: ưu tiên **RoleBinding phạm vi namespace** hơn
  ClusterRoleBinding; **tránh wildcard** vì Kubernetes mở rộng được nên wildcard cấp cả trên
  những loại đối tượng **sẽ được tạo trong tương lai**; hạn chế dùng `cluster-admin`; và tuyệt
  đối tránh thêm người dùng vào nhóm `system:masters`.
- Vì sao `system:masters` là trường hợp đặc biệt: thành viên nhóm này **bỏ qua mọi kiểm tra
  RBAC**, có quyền superuser không giới hạn, **không thu hồi được bằng cách xóa binding**, và
  request của họ **không bao giờ được gửi tới webhook phân quyền**.
- Quyền `list` và `watch` trên Secret **cũng làm lộ nội dung** Secret, không riêng gì `get` —
  vì phản hồi List chứa luôn nội dung của tất cả Secret.
- Quyền **tạo workload** trong một namespace ngầm cấp truy cập vào Secret, ConfigMap và
  PersistentVolume mount được trong namespace đó, **và mức truy cập API của mọi service
  account** trong namespace đó. Hệ quả: **ranh giới bên trong một namespace là yếu**; namespace
  mới là đơn vị tách các mức tin cậy khác nhau.
- Các đường leo thang đặc quyền cần thuộc tên: `nodes/proxy` (quyền **get** ở đây **không phải
  quyền chỉ đọc**, và nó **bỏ qua audit logging lẫn admission control**), verb `escalate`,
  verb `bind`, verb `impersonate`, quyền phê duyệt CSR, quyền `create` trên
  `serviceaccounts/token`, và quyền tạo PersistentVolume (kéo theo `hostPath`).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Taint/Toleration, NodeAffinity, PodAntiAffinity trong mục *Giảm thiểu việc phân phối các token đặc quyền* | đã học ở giai đoạn 7; ở đây chỉ là gợi ý bố trí Pod | giai đoạn 7 |
| Pod Security Standard mức **Baseline**/**Restricted** được nhắc tên | chưa học ba profile | bài [115](115-pod-security-standards-vi.md) |
| *Kiểm soát các admission webhook* | cần hiểu webhook trước | bài [173](173-admission-webhooks-vi.md) |
| *Sửa đổi Namespace* — patch label của namespace | rủi ro này chỉ có nghĩa khi đã bật Pod Security Admission | bài [116](116-pod-security-admission-vi.md) |
| *Các rủi ro từ chối dịch vụ* — quota số lượng đối tượng | ResourceQuota đã học rồi | giai đoạn 7 |

---

Kubernetes RBAC là một cơ chế kiểm soát bảo mật then chốt
để đảm bảo rằng người dùng và workload của cluster chỉ có quyền truy cập vào các tài nguyên cần thiết
để thực hiện vai trò của họ. Điều quan trọng là, khi thiết kế quyền cho người dùng
cluster, quản trị viên cluster phải hiểu rõ những khu vực có thể xảy ra leo thang đặc quyền (privilege escalation),
để giảm rủi ro việc truy cập quá mức dẫn đến các sự cố bảo mật.

Các thực hành tốt trình bày ở đây nên được đọc cùng với
[tài liệu RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-creation-or-update) tổng quát.

## Thực hành tốt tổng quát (General good practice)

### Đặc quyền tối thiểu (Least privilege) {#least-privilege}

Lý tưởng nhất, chỉ nên gán các quyền RBAC tối thiểu cho người dùng và service account. Chỉ những quyền
thực sự cần thiết cho hoạt động của họ mới nên được sử dụng. Mặc dù mỗi cluster sẽ khác nhau,
một số quy tắc chung có thể áp dụng là:

- Gán quyền ở cấp namespace nếu có thể. Sử dụng RoleBinding thay vì
  ClusterRoleBinding để cấp cho người dùng quyền chỉ trong phạm vi một namespace cụ thể.
- Tránh cấp quyền dạng wildcard khi có thể, đặc biệt là quyền trên tất cả các tài nguyên.
  Vì Kubernetes là một hệ thống có thể mở rộng (extensible), việc cấp quyền wildcard không chỉ
  cấp quyền trên tất cả các loại đối tượng hiện có trong cluster, mà còn trên tất cả các loại đối tượng
  được tạo ra trong tương lai.
- Quản trị viên không nên sử dụng tài khoản `cluster-admin` trừ khi thực sự cần thiết.
  Việc cấp cho một tài khoản đặc quyền thấp
  [quyền mạo danh (impersonation rights)](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)
  có thể tránh được việc vô tình sửa đổi các tài nguyên của cluster.
- Tránh thêm người dùng vào nhóm `system:masters`. Bất kỳ người dùng nào là thành viên của nhóm này
  đều bỏ qua mọi kiểm tra quyền RBAC và luôn có quyền truy cập siêu người dùng (superuser) không giới hạn,
  quyền này không thể bị thu hồi bằng cách xóa RoleBinding hay ClusterRoleBinding. Thêm nữa, nếu cluster
  đang dùng một webhook phân quyền, việc là thành viên của nhóm này cũng bỏ qua webhook đó (các request
  từ những người dùng là thành viên của nhóm đó không bao giờ được gửi tới webhook)

### Giảm thiểu việc phân phối các token đặc quyền (Minimize distribution of privileged tokens)

Lý tưởng nhất, không nên gán cho pod các service account đã được cấp quyền mạnh
(ví dụ, bất kỳ quyền nào được liệt kê trong phần [rủi ro leo thang đặc quyền](#privilege-escalation-risks)).
Trong trường hợp một workload cần các quyền mạnh, hãy cân nhắc các thực hành sau:

- Giới hạn số lượng node chạy các pod đặc quyền. Đảm bảo rằng mọi DaemonSet bạn chạy
  đều thực sự cần thiết và được chạy với đặc quyền tối thiểu để giới hạn phạm vi ảnh hưởng (blast radius)
  của các cuộc thoát container (container escape).
- Tránh chạy các pod đặc quyền cạnh các pod không đáng tin cậy hoặc bị công khai ra ngoài. Cân nhắc sử dụng
  [Taints và Toleration](139-taint-and-toleration-vi.md),
  [NodeAffinity](138-assign-pod-node-vi.md#node-affinity), hoặc
  [PodAntiAffinity](138-assign-pod-node-vi.md#inter-pod-affinity-and-anti-affinity)
  để đảm bảo các pod không chạy cạnh các Pod không đáng tin cậy hoặc ít tin cậy hơn. Đặc biệt chú ý đến
  các tình huống mà những Pod kém tin cậy hơn không đáp ứng Pod Security Standard mức **Restricted**.

### Gia cố (Hardening)

Kubernetes mặc định cung cấp một số quyền truy cập có thể không cần thiết trong mọi cluster. Việc rà soát
các quyền RBAC được cấp mặc định có thể mở ra cơ hội gia cố bảo mật.
Nói chung, không nên thay đổi các quyền được cấp cho các tài khoản `system:`; tuy nhiên có một số
lựa chọn để gia cố quyền của cluster:

- Rà soát các binding của nhóm `system:unauthenticated` và loại bỏ chúng nếu có thể, vì điều này cấp
  quyền truy cập cho bất kỳ ai có thể kết nối tới API server ở tầng mạng.
- Tránh việc tự động mount token service account mặc định bằng cách thiết lập
  `automountServiceAccountToken: false`. Để biết thêm chi tiết, xem
  [sử dụng token service account mặc định](279-configure-service-account-vi.md#use-the-default-service-account-to-access-the-api-server).
  Việc thiết lập giá trị này cho một Pod sẽ ghi đè thiết lập của service account; các workload
  cần token service account vẫn có thể mount chúng.

### Rà soát định kỳ (Periodic review)

Việc rà soát định kỳ các thiết lập RBAC của Kubernetes để tìm các mục dư thừa và
các khả năng leo thang đặc quyền là rất quan trọng.
Nếu kẻ tấn công có thể tạo một tài khoản người dùng trùng tên với một người dùng đã bị xóa,
chúng có thể tự động thừa hưởng toàn bộ các quyền của người dùng đã xóa đó, đặc biệt là
các quyền đã được gán cho người dùng đó.

## Kubernetes RBAC - các rủi ro leo thang đặc quyền (privilege escalation risks) {#privilege-escalation-risks}

Trong Kubernetes RBAC có một số quyền mà nếu được cấp, có thể cho phép một người dùng hoặc một service account
leo thang đặc quyền của họ trong cluster hoặc gây ảnh hưởng đến các hệ thống bên ngoài cluster.

Phần này nhằm giúp người vận hành cluster nhận diện được các khu vực
cần lưu ý, để đảm bảo họ không vô tình cho phép truy cập vào cluster nhiều hơn dự định.

### Liệt kê secret (Listing secrets)

Nhìn chung ai cũng hiểu rằng việc cho phép quyền `get` trên Secret sẽ cho phép người dùng đọc nội dung của chúng.
Điều quan trọng không kém cần lưu ý là quyền `list` và `watch` trên thực tế cũng cho phép người dùng xem được nội dung của Secret.
Ví dụ, khi một phản hồi List được trả về (chẳng hạn qua `kubectl get secrets -A -o yaml`), phản hồi đó
bao gồm nội dung của tất cả các Secret.

### Tạo workload (Workload creation) {#workload-creation}

Quyền tạo workload (Pod, hoặc
[các tài nguyên workload](./62-controllers-index-vi.md) quản lý Pod) trong một namespace
ngầm cấp quyền truy cập vào nhiều tài nguyên khác trong namespace đó, chẳng hạn như Secret, ConfigMap và
PersistentVolume có thể được mount vào Pod. Ngoài ra, vì Pod có thể chạy dưới bất kỳ
[ServiceAccount](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/) nào, việc cấp quyền
tạo workload cũng ngầm cấp các mức truy cập API của mọi service account trong
namespace đó.

Người dùng có thể chạy các Pod đặc quyền (privileged Pod) có thể lợi dụng quyền đó để chiếm quyền truy cập node và có khả năng
tiếp tục leo thang đặc quyền. Khi bạn không hoàn toàn tin tưởng một người dùng hay một chủ thể (principal) khác
về khả năng tạo các Pod đủ an toàn và được cách ly phù hợp, bạn nên thực thi
Pod Security Standard mức **Baseline** hoặc **Restricted**.
Bạn có thể sử dụng [Pod Security admission](116-pod-security-admission-vi.md)
hoặc các cơ chế (bên thứ ba) khác để thực hiện việc thực thi đó.

Vì những lý do này, các namespace nên được dùng để tách biệt các tài nguyên đòi hỏi các mức
tin cậy hoặc mức thuê (tenancy) khác nhau. Việc tuân theo nguyên tắc [đặc quyền tối thiểu](#least-privilege)
và gán tập quyền tối thiểu vẫn được xem là thực hành tốt nhất, nhưng các ranh giới bên trong một namespace
nên được xem là yếu.

### Tạo persistent volume (Persistent volume creation)

Nếu một ai đó - hoặc một ứng dụng nào đó - được phép tạo PersistentVolume tùy ý, quyền đó
bao gồm cả việc tạo các volume kiểu `hostPath`, điều này đồng nghĩa một Pod sẽ có quyền truy cập
vào hệ thống file của host bên dưới trên node liên quan. Việc cấp khả năng đó là một rủi ro bảo mật.

Có nhiều cách để một container với quyền truy cập không giới hạn vào hệ thống file của host có thể leo thang đặc quyền, bao gồm
đọc dữ liệu của các container khác, và lạm dụng thông tin xác thực (credential) của các dịch vụ hệ thống, chẳng hạn Kubelet.

Bạn chỉ nên cho phép quyền tạo các đối tượng PersistentVolume đối với:

- Người dùng (người vận hành cluster) cần quyền này cho công việc của họ, và là người bạn tin tưởng.
- Các thành phần control plane của Kubernetes, vốn tạo các PersistentVolume dựa trên các PersistentVolumeClaim
  được cấu hình để cấp phát (provision) tự động.
  Việc này thường được thiết lập bởi nhà cung cấp Kubernetes hoặc bởi người vận hành khi cài đặt một CSI driver.

Khi cần truy cập vào lưu trữ bền vững (persistent storage), các quản trị viên đáng tin cậy nên tạo
PersistentVolume, còn những người dùng bị giới hạn nên dùng PersistentVolumeClaim để truy cập kho lưu trữ đó.

### Truy cập subresource `proxy` của Node (Access to `proxy` subresource of Nodes)

Người dùng có quyền truy cập vào subresource `nodes/proxy` có quyền đối với Kubelet API,
điều này cho phép thực thi lệnh trên mọi pod thuộc (các) node mà họ có quyền.
Quyền truy cập này bỏ qua ghi log kiểm toán (audit logging) và admission control, vì vậy cần thận trọng trước khi
cấp bất kỳ quyền nào đối với tài nguyên này.
Các API này có thể được sử dụng qua các request websocket HTTP `GET`, vốn chỉ yêu cầu được phân quyền cho verb **get**.
Điều này có nghĩa là quyền **get** trên `nodes/proxy` không phải là một quyền chỉ đọc.
Ví dụ, quyền **get** trên `nodes/proxy` cấp quyền truy cập vào các API kubelet đặc quyền
có thể lấy log của container hoặc thực thi (exec) và gắn (attach) vào các tiến trình của pod,
ngay cả khi người gọi không có các quyền tương đương thông qua
Kubernetes API.

Xem [Xác thực/phân quyền Kubelet (Kubelet authentication/authorization)](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/#get-nodes-proxy-warning)
để biết thêm thông tin.

### Verb escalate (Escalate verb)

Nhìn chung, hệ thống RBAC ngăn người dùng tạo các clusterrole có nhiều quyền hơn những gì người dùng đó sở hữu.
Ngoại lệ là verb `escalate`. Như đã ghi trong [tài liệu RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#restrictions-on-role-creation-or-update),
người dùng có quyền này trên thực tế có thể leo thang đặc quyền của họ.

### Verb bind (Bind verb)

Tương tự verb `escalate`, việc cấp cho người dùng quyền này cho phép bỏ qua các cơ chế bảo vệ
tích hợp sẵn của Kubernetes chống leo thang đặc quyền, cho phép người dùng tạo các binding tới
các role có những quyền mà họ chưa sở hữu.

### Verb impersonate (Impersonate verb)

Verb này cho phép người dùng mạo danh (impersonate) và giành được các quyền của những người dùng khác trong cluster.
Cần thận trọng khi cấp quyền này, để đảm bảo không thể giành được các quyền quá mức
thông qua một trong các tài khoản bị mạo danh.

### CSR và việc cấp certificate (CSRs and certificate issuing)

CSR API cho phép người dùng có quyền `create` trên CSR và quyền `update` trên `certificatesigningrequests/approval`,
với signer là `kubernetes.io/kube-apiserver-client`, tạo các client certificate mới
cho phép người dùng xác thực với cluster. Các client certificate đó có thể mang tên tùy ý,
kể cả trùng tên với các thành phần hệ thống của Kubernetes. Điều này trên thực tế sẽ cho phép leo thang đặc quyền.

### Yêu cầu token (Token request)

Người dùng có quyền `create` trên `serviceaccounts/token` có thể tạo TokenRequest để phát hành
token cho các service account hiện có.

### Kiểm soát các admission webhook (Control admission webhooks)

Người dùng có quyền kiểm soát `validatingwebhookconfigurations` hoặc `mutatingwebhookconfigurations`
có thể kiểm soát các webhook có khả năng đọc bất kỳ đối tượng nào được tiếp nhận (admit) vào cluster, và trong trường hợp
mutating webhook, còn có thể sửa đổi (mutate) các đối tượng được tiếp nhận.

### Sửa đổi Namespace (Namespace modification) {#namespace-modification}

Người dùng có thể thực hiện thao tác **patch** trên các đối tượng Namespace (thông qua một RoleBinding trong phạm vi namespace tới một Role có quyền đó) có thể sửa đổi
các label trên namespace đó. Trong các cluster có sử dụng Pod Security Admission, điều này có thể cho phép người dùng cấu hình namespace
theo một chính sách lỏng lẻo hơn so với dự định của các quản trị viên.
Với các cluster có sử dụng NetworkPolicy, người dùng có thể thiết lập các label gián tiếp cho phép
truy cập đến các service mà quản trị viên không có ý định cho phép.

## Kubernetes RBAC - các rủi ro từ chối dịch vụ (denial of service risks) {#denial-of-service-risks}

### Từ chối dịch vụ qua việc tạo đối tượng (Object creation denial-of-service) {#object-creation-dos}

Người dùng có quyền tạo đối tượng trong một cluster có thể tạo ra các đối tượng đủ lớn
để gây ra tình trạng từ chối dịch vụ (denial of service), dựa trên kích thước hoặc số lượng đối tượng, như được thảo luận trong
[etcd used by Kubernetes is vulnerable to OOM attack](https://github.com/kubernetes/kubernetes/issues/107325). Điều này có thể
đặc biệt liên quan trong các cluster đa người thuê (multi-tenant) nếu những người dùng bán tin cậy hoặc không đáng tin cậy
được phép truy cập hạn chế vào hệ thống.

Một phương án giảm thiểu vấn đề này là sử dụng
[hạn ngạch tài nguyên (resource quota)](https://kubernetes.io/docs/concepts/policy/resource-quotas#object-count-quota)
để giới hạn số lượng đối tượng có thể được tạo.

## Tiếp theo (What's next)

* Để tìm hiểu thêm về RBAC, xem [tài liệu RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Bạn cấp cho một service account quyền `list` trên Secret nhưng cố ý **không** cấp `get`, vì
   nghĩ như vậy nó chỉ thấy được tên Secret. Sai ở đâu?
2. Vì sao bài nói các ranh giới **bên trong** một namespace nên được xem là yếu? Cụ thể, cấp
   quyền tạo Deployment trong một namespace thì ngầm cấp thêm những gì?
3. Trên cluster lab, bạn định cấp cho một tài khoản giám sát quyền **get** trên `nodes/proxy`
   để nó đọc dữ liệu từ `k8s-worker1` và `k8s-worker2`. Theo bài, bạn vừa cấp thêm khả năng gì,
   và hai cơ chế kiểm soát nào bị vượt qua?
4. Thu hồi quyền của một tài khoản đã được gán `cluster-admin` khác gì thu hồi quyền của một
   tài khoản đã được thêm vào nhóm `system:masters`?
5. Vì sao bài coi việc cấp quyền dạng wildcard trong Kubernetes là nguy hiểm hơn trong một hệ
   thống thông thường?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không có khác biệt về mức lộ dữ liệu.** Trực giác "list chỉ ra tên" là sai vì trong
   Kubernetes, một phản hồi List **bao gồm nội dung của tất cả các Secret** — ví dụ
   `kubectl get secrets -A -o yaml`. Bài nói rõ `list` và `watch` trên thực tế **cũng cho phép
   người dùng xem được nội dung Secret**, ngang với `get`.
2. Vì **quyền tạo workload ngầm cấp truy cập vào nhiều tài nguyên khác trong cùng namespace**:
   các Secret, ConfigMap và PersistentVolume có thể được mount vào Pod. Nặng hơn, vì Pod có thể
   chạy dưới **bất kỳ ServiceAccount nào** trong namespace, quyền tạo workload còn ngầm cấp
   **mức truy cập API của mọi service account trong namespace đó**. Vì vậy namespace phải được
   dùng để tách các mức tin cậy khác nhau; **đặc quyền tối thiểu vẫn cần, nhưng đừng trông cậy
   vào ranh giới bên trong một namespace**.
3. Bạn vừa cấp quyền **thực thi lệnh trong mọi Pod trên hai node đó**, cộng với việc lấy log và
   attach vào tiến trình của Pod — vì các endpoint này chạy qua websocket trên HTTP `GET`, nên
   **`get` trên `nodes/proxy` không phải là một quyền chỉ đọc**. Hai cơ chế bị vượt qua: **ghi
   log kiểm toán (audit logging)** và **admission control**. Người gọi làm được việc này **ngay
   cả khi không có quyền tương đương qua Kubernetes API**.
4. `cluster-admin` được cấp qua một ClusterRoleBinding, nên **xóa binding là thu hồi xong**.
   Còn thành viên `system:masters` **bỏ qua toàn bộ kiểm tra RBAC** và quyền superuser đó
   **không thể bị thu hồi bằng cách xóa RoleBinding hay ClusterRoleBinding**; ngoài ra request
   của họ **không bao giờ được gửi tới webhook phân quyền**, nên webhook cũng không chặn được.
5. Vì **Kubernetes là một hệ thống có thể mở rộng**. Cấp quyền wildcard không chỉ cấp trên tất
   cả loại đối tượng **hiện có**, mà còn trên **tất cả loại đối tượng sẽ được tạo ra trong
   tương lai** — nghĩa là quyền tự nới rộng theo thời gian mà bạn không hề sửa gì trong Role.

</details>

Đây là bài cuối của **phần truy cập** trong giai đoạn 9. Câu nào chưa trả lời được thì quay lại
đúng mục tương ứng trước khi vào **Lab 9a — ServiceAccount, authn/authz và RBAC** (chưa viết,
xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).
