# Bảo mật một Cluster (Securing a Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 22 — Audit và mã hóa dữ liệu](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu),
bài 5/6 · Phần II không có lab: thực hành thẳng trên cluster VM của
[Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) và tự chấm bằng **Checkpoint** ở cuối
[mục giai đoạn 22](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu).

Đây là bài **tổng hợp**, không phải bài dạy cơ chế mới: gần như mọi mục đều dẫn về một bài bạn đã
đọc. Đọc nó như một bảng rà soát, và ở mỗi mục hãy tự hỏi "cluster lab của mình đang ở đâu trên
thang này". Nó nối tiếp bài [129](129-security-checklist-vi.md) đã đọc ở mạch chính.

> ✅ **Trả nợ #6 — Mã hóa Secret at rest.** Mục *Mã hóa secret khi lưu trữ* của bài này là chỗ nợ
> phát sinh từ [giai đoạn 3b](00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod)
> được trả. **Đọc lại bài [109](109-secret-vi.md) — phần nói Secret chỉ mã hóa base64 — trước khi
> làm Checkpoint của giai đoạn 22**, rồi mới sang bài [208](208-encrypt-data-vi.md).

**Phải hiểu ở lần đọc này:**

- Trục của mục *Kiểm soát truy cập vào Kubernetes API*, đúng ba chặng đã học: **TLS cho toàn bộ
  lưu lượng API** → **xác thực** (mọi client đều phải xác thực, kể cả node, proxy, scheduler và
  volume plugin) → **phân quyền**, và bài khuyên dùng **kết hợp authorizer `Node` và `RBAC` cùng
  admission plugin `NodeRestriction`**.
- Bẫy leo thang đặc quyền gián tiếp, cũng ở mục đó: không được tạo Pod trực tiếp nhưng được tạo
  Deployment thì **vẫn tạo được Pod**; xóa một node khỏi API thì Pod trên node đó **bị kết thúc và
  tạo lại nơi khác**. Role hẹp phải rà kỹ vì lý do này.
- Mục *Kiểm soát truy cập vào Kubelet*: endpoint HTTPS của kubelet trao **quyền kiểm soát rất
  mạnh** với node và container, và **mặc định cho phép truy cập không cần xác thực** — cluster
  production **phải** bật xác thực và phân quyền cho kubelet.
- Bốn nhóm kiểm soát lúc chạy ở mục *Kiểm soát năng lực của workload hoặc người dùng lúc chạy*:
  [resource quota](134-resource-quotas-vi.md) và limit range; [security
  context](291-security-context-vi.md) cùng [Pod Security
  admission](116-pod-security-admission-vi.md) ở mức **Baseline** hoặc **Restricted**; chặn nạp
  kernel module không mong muốn (blacklist trong `/etc/modprobe.d/`, hoặc dùng LSM từ chối
  `module_request`); và [network policy](201-declare-network-policy-vi.md) để hạn chế lưu lượng.
- Năm việc ở mục *Bảo vệ các thành phần cluster khỏi bị xâm phạm*: **quyền ghi etcd tương đương
  root toàn cluster** nên phải dùng mutual TLS và cô lập sau tường lửa; **bật audit log** và lưu
  file audit ở máy an toàn; **xoay vòng credential**, thu hồi bootstrap token sau khi bootstrap
  xong; **rà quyền của tích hợp bên thứ ba** trước khi bật; và **mã hóa Secret at rest**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Mục *Trước khi bạn bắt đầu* — minikube và các playground | lộ trình không dùng minikube hay cluster dùng chung | cluster VM ba node của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) là môi trường thực hành duy nhất |
| Mục *Hạn chế truy cập API metadata của cloud* | cluster lab chạy VM cục bộ theo [A1.2 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm), không có dịch vụ metadata của cloud | ngoài phạm vi lộ trình; công cụ dùng để chặn là NetworkPolicy, đã thực hành ở [Lab 5b](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) |
| Câu nói về mã hóa **custom resource** khi lưu trữ | chưa có CRD nào trên cluster lab để mã hóa | [Giai đoạn 28 — Mở rộng Kubernetes](00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes) |
| Đoạn nhắc tới [PodSecurityPolicy](117-pod-security-policy-vi.md) | PSP đã bị gỡ khỏi Kubernetes; lộ trình xếp bài đó là tài liệu lịch sử | bài [117](117-pod-security-policy-vi.md) ở [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), đã đọc |
| Admission plugin beta `PodNodeSelector` | là plugin beta, không có trong cấu hình cluster lab | cơ chế bố trí Pod đã học ở bài [138](138-assign-pod-node-vi.md) và [139](139-taint-and-toleration-vi.md) |

---

Tài liệu này đề cập đến các chủ đề liên quan tới việc bảo vệ một cluster khỏi truy cập vô tình
hoặc có chủ đích xấu, và đưa ra các khuyến nghị về bảo mật tổng thể.

## Trước khi bạn bắt đầu (Before you begin)

* Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao
  tiếp với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node
  không đóng vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster
  bằng [minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong
  các sân chơi (playground) Kubernetes sau:

  * [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
  * [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
  * [KodeKloud](https://kodekloud.com/public-playgrounds)

  Để kiểm tra phiên bản, nhập `kubectl version`.

## Kiểm soát truy cập vào Kubernetes API (Controlling access to the Kubernetes API)

Vì Kubernetes hoàn toàn được điều khiển qua API, việc kiểm soát và giới hạn ai có thể truy cập
cluster và họ được phép thực hiện những hành động nào chính là tuyến phòng thủ đầu tiên.

### Dùng Transport Layer Security (TLS) cho toàn bộ lưu lượng API (Use Transport Layer Security (TLS) for all API traffic)

Kubernetes kỳ vọng rằng mọi giao tiếp API trong cluster đều được mã hóa mặc định bằng TLS, và
phần lớn các phương thức cài đặt sẽ cho phép tạo và phân phối các certificate cần thiết tới các
thành phần của cluster. Lưu ý rằng một số thành phần và phương thức cài đặt có thể bật các port
cục bộ qua HTTP, và quản trị viên nên tìm hiểu kỹ cấu hình của từng thành phần để nhận diện
những luồng lưu lượng có khả năng không được bảo mật.

### Xác thực API (API Authentication)

Khi cài đặt một cluster, hãy chọn cơ chế xác thực (authentication) cho các API server sao cho
phù hợp với các mẫu hình truy cập phổ biến của bạn. Ví dụ, các cluster nhỏ dùng bởi một người
có thể muốn dùng cách tiếp cận đơn giản với certificate hoặc static Bearer token. Các cluster
lớn hơn có thể muốn tích hợp với một máy chủ OIDC hoặc LDAP sẵn có, cho phép chia người dùng
thành các nhóm (group).

Mọi API client đều phải được xác thực, kể cả những client thuộc về hạ tầng như các node, các
proxy, scheduler và các volume plugin. Những client này thường là các
[service account](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
hoặc dùng x509 client certificate, và chúng được tạo tự động khi cluster khởi động hoặc được
thiết lập như một phần của quá trình cài đặt cluster.

Tham khảo [tài liệu tham chiếu về xác thực](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
để biết thêm thông tin.

### Phân quyền API (API Authorization)

Sau khi được xác thực, mỗi lời gọi API còn được kỳ vọng phải vượt qua một bước kiểm tra phân
quyền (authorization). Kubernetes tích hợp sẵn thành phần
[Role-Based Access Control (RBAC)](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
để đối chiếu người dùng hoặc nhóm gửi yêu cầu với một tập các quyền được gói trong các role.
Các quyền này kết hợp các động từ (get, create, delete) với các tài nguyên (pods, services,
nodes) và có thể có phạm vi namespace hoặc phạm vi cluster. Một bộ role có sẵn (out-of-the-box)
được cung cấp, mang lại sự phân tách trách nhiệm mặc định hợp lý tùy theo những hành động mà
một client có thể muốn thực hiện. Bạn nên dùng kết hợp cả hai authorizer
[Node](https://kubernetes.io/docs/reference/access-authn-authz/node/) và
[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/), cùng với admission
plugin [NodeRestriction](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction).

Cũng như với xác thực, các role đơn giản và rộng có thể phù hợp với những cluster nhỏ, nhưng
khi có nhiều người dùng tương tác với cluster hơn, có thể sẽ cần tách các nhóm (team) vào các
namespace riêng biệt với các role bị giới hạn chặt hơn.

Với phân quyền, điều quan trọng là hiểu được việc cập nhật trên một object có thể gây ra hành
động ở những chỗ khác như thế nào. Ví dụ, một người dùng có thể không được tạo Pod trực tiếp,
nhưng nếu cho phép họ tạo một Deployment — thứ sẽ tạo Pod thay mặt họ — thì họ vẫn tạo được
các Pod đó một cách gián tiếp. Tương tự, việc xóa một node khỏi API sẽ khiến các Pod đã được
lập lịch lên node đó bị kết thúc và được tạo lại trên các node khác. Các role có sẵn thể hiện
sự cân bằng giữa tính linh hoạt và các trường hợp sử dụng phổ biến, nhưng các role bị giới hạn
chặt hơn cần được rà soát cẩn thận để tránh leo thang đặc quyền (escalation) ngoài ý muốn. Bạn
có thể tự tạo các role riêng cho trường hợp sử dụng của mình nếu các role có sẵn không đáp ứng
được nhu cầu.

Tham khảo [phần tham chiếu về phân quyền](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
để biết thêm thông tin.

Nếu cluster của bạn dùng
[Dynamic Resource Allocation (DRA)](149-dynamic-resource-allocation-vi.md), hãy rà soát cơ chế
phân quyền các synthetic subresource của DRA (`resourceclaims/binding` và
`resourceclaims/driver`) và chỉ cấp tập động từ (verb) tối thiểu cần thiết cho từng thành
phần. Để biết chi tiết, xem
[Hướng dẫn tăng cường bảo mật - Dynamic Resource Allocation](125-hardening-dra-vi.md)
và
[Tăng cường bảo mật Dynamic Resource Allocation trong Cluster của bạn](211-hardening-dra-tasks-vi.md).

## Kiểm soát truy cập vào Kubelet (Controlling access to the Kubelet)

Các kubelet công khai (expose) các endpoint HTTPS trao quyền kiểm soát rất mạnh đối với node
và các container. Theo mặc định, kubelet cho phép truy cập không cần xác thực vào API này.

Các cluster production nên bật xác thực và phân quyền cho kubelet.

Tham khảo [tài liệu tham chiếu về xác thực/phân quyền của Kubelet](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-authn-authz/)
để biết thêm thông tin.

## Kiểm soát năng lực của workload hoặc người dùng lúc chạy (Controlling the capabilities of a workload or user at runtime)

Phân quyền trong Kubernetes được thiết kế có chủ đích ở mức cao, tập trung vào các hành động
thô trên các tài nguyên. Các cơ chế kiểm soát mạnh hơn tồn tại dưới dạng các **chính sách
(policy)** nhằm giới hạn — theo từng trường hợp sử dụng — cách các object đó tác động lên
cluster, lên chính chúng và lên các tài nguyên khác.

### Giới hạn sử dụng tài nguyên trên một cluster (Limiting resource usage on a cluster)

[Resource quota](134-resource-quotas-vi.md) giới hạn số lượng hoặc dung lượng tài nguyên được
cấp cho một namespace. Cơ chế này thường được dùng nhất để giới hạn lượng CPU, bộ nhớ (memory)
hoặc đĩa lưu trữ bền vững (persistent disk) mà một namespace có thể cấp phát, nhưng cũng có
thể kiểm soát số lượng Pod, Service hoặc volume tồn tại trong mỗi namespace.

[Limit range](232-memory-default-namespace-vi.md) giới hạn kích thước lớn nhất hoặc nhỏ nhất
của một số tài nguyên nêu trên, nhằm ngăn người dùng yêu cầu các giá trị cao hoặc thấp một
cách bất hợp lý đối với những tài nguyên thường được dự trữ như bộ nhớ, hoặc để cung cấp giới
hạn mặc định khi không có giới hạn nào được chỉ định.

### Kiểm soát các đặc quyền mà container chạy với (Controlling what privileges containers run with)

Định nghĩa của một Pod chứa một
[security context](291-security-context-vi.md)
cho phép nó yêu cầu quyền chạy dưới một người dùng Linux cụ thể trên node (như root), quyền
chạy ở chế độ privileged hoặc truy cập mạng của host, và các quyền kiểm soát khác mà nếu không
có sẽ cho phép nó chạy không bị ràng buộc trên node đang chứa nó.

Bạn có thể cấu hình [Pod security admission](116-pod-security-admission-vi.md) để cưỡng chế
(enforce) việc dùng một [Pod Security Standard](115-pod-security-standards-vi.md) cụ thể trong
một namespace, hoặc để phát hiện các vi phạm.

Nói chung, hầu hết các workload ứng dụng chỉ cần quyền truy cập hạn chế tới tài nguyên của
host để có thể chạy thành công như một tiến trình root (uid 0) mà không cần truy cập thông tin
của host. Tuy nhiên, xét đến các đặc quyền gắn liền với người dùng root, bạn nên viết các
container ứng dụng sao cho chúng chạy dưới một người dùng không phải root. Tương tự, quản trị
viên muốn ngăn các ứng dụng phía client thoát khỏi (escape) container của chúng nên áp dụng
Pod Security Standard mức **Baseline** hoặc **Restricted**.

### Ngăn container nạp các kernel module không mong muốn (Preventing containers from loading unwanted kernel modules)

Kernel Linux tự động nạp các kernel module từ đĩa khi cần trong một số tình huống nhất định,
chẳng hạn khi một thiết bị phần cứng được gắn vào hoặc một filesystem được mount. Điều đặc
biệt đáng lưu ý với Kubernetes là ngay cả các tiến trình không có đặc quyền (unprivileged)
cũng có thể khiến một số kernel module liên quan đến giao thức mạng được nạp, chỉ bằng cách
tạo một socket thuộc kiểu tương ứng. Điều này có thể cho phép kẻ tấn công khai thác một lỗ
hổng bảo mật trong một kernel module mà quản trị viên vẫn tưởng là không được sử dụng.

Để ngăn các module cụ thể bị nạp tự động, bạn có thể gỡ chúng khỏi node, hoặc thêm các quy tắc
để chặn chúng. Trên hầu hết các bản phân phối Linux, bạn có thể làm việc đó bằng cách tạo một
file như `/etc/modprobe.d/kubernetes-blacklist.conf` với nội dung dạng:

```
# DCCP nhiều khả năng không cần đến, đã từng có nhiều lỗ hổng
# nghiêm trọng, và không được bảo trì tốt.
blacklist dccp

# SCTP không được dùng trong hầu hết các cluster Kubernetes, và
# trước đây cũng từng có lỗ hổng.
blacklist sctp
```

Để chặn việc nạp module một cách tổng quát hơn, bạn có thể dùng một Linux Security Module
(chẳng hạn SELinux) để từ chối hoàn toàn quyền `module_request` đối với các container, ngăn
kernel nạp module cho container trong mọi tình huống. (Các Pod vẫn có thể dùng những module đã
được nạp thủ công, hoặc những module được kernel nạp thay mặt cho một tiến trình có đặc quyền
cao hơn.)

### Hạn chế truy cập mạng (Restricting network access)

[Network policy](201-declare-network-policy-vi.md) của một namespace cho phép tác giả ứng dụng
hạn chế những Pod nào ở namespace khác được truy cập các Pod và port bên trong namespace của
họ. Nhiều [nhà cung cấp giải pháp mạng cho Kubernetes](157-networking-vi.md) hiện đã tôn trọng
network policy.

Quota và limit range cũng có thể được dùng để kiểm soát việc người dùng có được yêu cầu node
port hoặc Service kiểu cân bằng tải (load-balanced) hay không — điều mà trên nhiều cluster có
thể quyết định việc ứng dụng của những người dùng đó có nhìn thấy được từ bên ngoài cluster
hay không.

Có thể có thêm các lớp bảo vệ bổ sung để kiểm soát quy tắc mạng theo từng plugin hoặc từng môi
trường, chẳng hạn tường lửa (firewall) trên từng node, tách biệt vật lý các node của cluster
để tránh nhiễu chéo, hoặc các chính sách mạng nâng cao.

### Hạn chế truy cập API metadata của cloud (Restricting cloud metadata API access)

Các nền tảng cloud (AWS, Azure, GCE, v.v.) thường công khai các dịch vụ metadata cục bộ cho
các instance. Theo mặc định, các API này có thể được truy cập bởi những Pod đang chạy trên một
instance, và có thể chứa thông tin xác thực (credential) cloud của node đó, hoặc dữ liệu cấp
phát (provisioning data) như credential của kubelet. Các credential này có thể bị dùng để leo
thang đặc quyền bên trong cluster hoặc sang các dịch vụ cloud khác thuộc cùng tài khoản.

Khi chạy Kubernetes trên một nền tảng cloud, hãy giới hạn các quyền được cấp cho credential
của instance, dùng [network policy](201-declare-network-policy-vi.md) để hạn chế Pod truy cập
API metadata, và tránh dùng dữ liệu cấp phát để truyền secret.

### Kiểm soát những node mà Pod có thể truy cập (Controlling which nodes pods may access)

Theo mặc định, không có bất kỳ hạn chế nào về việc node nào được chạy một Pod. Kubernetes cung
cấp cho người dùng cuối một
[tập hợp phong phú các chính sách kiểm soát việc bố trí Pod lên node](138-assign-pod-node-vi.md)
và cơ chế [bố trí và evict Pod dựa trên taint](139-taint-and-toleration-vi.md). Với nhiều
cluster, việc dùng các chính sách này để tách biệt các workload có thể là một quy ước mà các
tác giả tự áp dụng hoặc cưỡng chế thông qua công cụ.

Với tư cách quản trị viên, admission plugin ở trạng thái beta `PodNodeSelector` có thể được
dùng để buộc các Pod trong một namespace phải mang một node selector mặc định hoặc bắt buộc,
và nếu người dùng cuối không thể sửa đổi namespace, cơ chế này có thể giới hạn chặt việc bố
trí của toàn bộ các Pod trong một workload cụ thể.

## Bảo vệ các thành phần cluster khỏi bị xâm phạm (Protecting cluster components from compromise)

Phần này mô tả một số mẫu hình phổ biến để bảo vệ cluster khỏi bị xâm phạm.

### Hạn chế truy cập etcd (Restrict access to etcd)

Có quyền ghi vào backend etcd của API tương đương với việc chiếm được quyền root trên toàn bộ
cluster, và quyền đọc cũng có thể bị lợi dụng để leo thang đặc quyền khá nhanh chóng. Quản trị
viên nên luôn dùng credential mạnh cho kết nối từ các API server tới máy chủ etcd, chẳng hạn
xác thực lẫn nhau (mutual auth) qua TLS client certificate, và thường được khuyến nghị là cô
lập các máy chủ etcd sau một tường lửa mà chỉ các API server mới truy cập được.

> **Thận trọng:** Cho phép các thành phần khác trong cluster truy cập instance etcd chính
> (master) với quyền đọc hoặc ghi trên toàn bộ keyspace là tương đương với việc cấp quyền
> cluster-admin. Rất nên dùng các instance etcd riêng cho những thành phần không thuộc master,
> hoặc dùng etcd ACL để giới hạn quyền đọc và ghi vào một phần của keyspace.

### Bật ghi log audit (Enable audit logging)

[Audit logger](306-audit-vi.md) là một tính năng
beta ghi lại các hành động do API thực hiện, phục vụ việc phân tích về sau nếu xảy ra sự cố
xâm phạm. Bạn nên bật ghi log audit và lưu trữ (archive) file audit trên một máy chủ an toàn.

### Hạn chế truy cập các tính năng alpha hoặc beta (Restrict access to alpha or beta features)

Các tính năng alpha và beta của Kubernetes đang trong quá trình phát triển tích cực và có thể
có những hạn chế hoặc lỗi dẫn tới lỗ hổng bảo mật. Hãy luôn cân nhắc giá trị mà một tính năng
alpha hoặc beta mang lại so với rủi ro có thể có đối với thế trận bảo mật (security posture)
của bạn. Khi còn nghi ngờ, hãy tắt những tính năng bạn không dùng.

### Xoay vòng credential hạ tầng thường xuyên (Rotate infrastructure credentials frequently)

Vòng đời của một secret hoặc credential càng ngắn thì kẻ tấn công càng khó lợi dụng được
credential đó. Hãy đặt thời hạn ngắn cho các certificate và tự động hóa việc xoay vòng
(rotation) chúng. Dùng một nhà cung cấp xác thực có khả năng kiểm soát thời gian hiệu lực của
các token được phát hành, và dùng thời hạn ngắn ở bất cứ đâu có thể. Nếu bạn dùng token
service account trong các tích hợp bên ngoài, hãy lên kế hoạch xoay vòng các token đó thường
xuyên. Ví dụ, khi giai đoạn bootstrap đã hoàn tất, bootstrap token dùng để thiết lập các node
nên bị thu hồi hoặc bị gỡ bỏ quyền.

### Rà soát các tích hợp bên thứ ba trước khi bật chúng (Review third party integrations before enabling them)

Nhiều tích hợp bên thứ ba với Kubernetes có thể làm thay đổi hồ sơ bảo mật của cluster của
bạn. Khi bật một tích hợp, hãy luôn rà soát các quyền mà phần mở rộng đó yêu cầu trước khi cấp
quyền truy cập cho nó. Ví dụ, nhiều tích hợp bảo mật có thể yêu cầu quyền xem toàn bộ secret
trên cluster của bạn — điều này thực chất biến thành phần đó thành một cluster admin. Khi còn
nghi ngờ, hãy giới hạn tích hợp đó chỉ hoạt động trong một namespace duy nhất nếu có thể.

Các thành phần có khả năng tạo Pod cũng có thể trở nên mạnh một cách bất ngờ nếu chúng được
phép làm vậy bên trong những namespace như `kube-system`, bởi vì các Pod đó có thể chiếm được
quyền truy cập vào secret của các service account, hoặc chạy với quyền được nâng cao nếu các
service account đó được cấp quyền theo các
[PodSecurityPolicy](117-pod-security-policy-vi.md) dễ dãi.

Nếu bạn dùng [Pod Security admission](116-pod-security-admission-vi.md) và cho phép bất kỳ
thành phần nào tạo Pod trong một namespace chấp nhận Pod privileged, các Pod đó có thể thoát
khỏi container của chúng và lợi dụng quyền truy cập được mở rộng này để leo thang đặc quyền.

Bạn không nên cho phép các thành phần không đáng tin cậy tạo Pod trong bất kỳ namespace hệ
thống nào (những namespace có tên bắt đầu bằng `kube-`), cũng như trong bất kỳ namespace nào
mà việc cấp quyền đó mở ra khả năng leo thang đặc quyền.

### Mã hóa secret khi lưu trữ (Encrypt secrets at rest)

Nói chung, cơ sở dữ liệu etcd sẽ chứa mọi thông tin có thể truy cập được qua Kubernetes API,
và có thể cho kẻ tấn công cái nhìn rất sâu vào trạng thái cluster của bạn. Hãy luôn mã hóa các
bản backup bằng một giải pháp backup và mã hóa đã được thẩm định kỹ, và cân nhắc dùng mã hóa
toàn bộ đĩa (full disk encryption) ở những nơi có thể.

Kubernetes hỗ trợ tùy chọn [mã hóa khi lưu trữ (encryption at rest)](208-encrypt-data-vi.md)
cho thông tin trong Kubernetes API. Điều này giúp bạn đảm bảo rằng khi Kubernetes lưu dữ liệu
của các object (ví dụ các object `Secret` hoặc `ConfigMap`), API server ghi xuống một biểu
diễn đã được mã hóa của object đó. Việc mã hóa này có nghĩa là ngay cả người có quyền truy cập
dữ liệu backup của etcd cũng không thể xem được nội dung của các object đó. Trong Kubernetes
v1.36, bạn cũng có thể mã hóa các custom resource; khả năng mã hóa khi lưu trữ cho các API mở
rộng được định nghĩa bằng CustomResourceDefinition đã được thêm vào Kubernetes từ bản phát
hành v1.26.

### Nhận cảnh báo về cập nhật bảo mật và báo cáo lỗ hổng (Receiving alerts for security updates and reporting vulnerabilities)

Hãy tham gia nhóm
[kubernetes-announce](https://groups.google.com/forum/#!forum/kubernetes-announce)
để nhận email về các thông báo bảo mật. Xem trang
[báo cáo bảo mật](https://kubernetes.io/docs/reference/issues-security/security/)
để biết thêm về cách báo cáo lỗ hổng.

## Tiếp theo (What's next)

- [Danh sách kiểm tra bảo mật (Security Checklist)](129-security-checklist-vi.md) để biết thêm
  thông tin về các hướng dẫn bảo mật của Kubernetes.
- [Tài liệu tham chiếu Seccomp cho Node](https://kubernetes.io/docs/reference/node/seccomp/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 22:

1. Bài xếp việc kiểm soát truy cập API thành mấy tuyến, theo thứ tự nào? Riêng ở chặng phân quyền,
   bài khuyên dùng kết hợp những thành phần nào?
2. **Câu bẫy.** Bạn viết một Role không cấp động từ nào trên `pods`, chỉ cấp `create` trên
   `deployments`. Người dùng gắn Role đó có tạo được Pod trên cluster không? Bài còn nêu ví dụ leo
   thang gián tiếp nào nữa?
3. Vì sao bài nói quyền **ghi** vào etcd tương đương chiếm quyền root trên toàn bộ cluster? Hai
   biện pháp bảo vệ etcd mà bài khuyến nghị là gì?
4. Bạn đã join `lab-k8s-worker1` và `lab-k8s-worker2` vào cluster bằng bootstrap token. Theo mục
   *Xoay vòng credential hạ tầng thường xuyên*, token đó phải được xử lý thế nào sau khi giai đoạn
   bootstrap kết thúc, và nguyên tắc chung phía sau lời khuyên đó là gì?
5. Bài nói gì về hành vi **mặc định** của API HTTPS trên kubelet, và cluster production phải làm gì
   với nó?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Ba tuyến, theo thứ tự: TLS cho toàn bộ lưu lượng API → xác thực → phân quyền.** Bài nhấn mạnh
   **mọi** API client đều phải được xác thực, kể cả client hạ tầng như node, proxy, scheduler và
   volume plugin. Ở chặng phân quyền, bài khuyên dùng **kết hợp authorizer `Node` và `RBAC`, cộng
   với admission plugin `NodeRestriction`**.
2. **Có — tạo được, một cách gián tiếp.** Deployment **tạo Pod thay mặt người dùng**, nên quyền tạo
   Deployment kéo theo khả năng tạo Pod dù Role không hề nhắc tới `pods`. Ví dụ thứ hai của bài:
   **xóa một node khỏi API** khiến các Pod đã lập lịch trên node đó **bị kết thúc và được tạo lại
   trên node khác** — một hành động trên object này gây hậu quả ở chỗ khác. Đó là lý do bài bảo
   phải rà soát cẩn thận các role hẹp để tránh leo thang đặc quyền ngoài ý muốn.
3. Vì **etcd là backend chứa mọi thứ truy cập được qua Kubernetes API**: ghi được vào đó là sửa
   được trạng thái cluster mà không đi qua bất kỳ chặng authn/authz nào; ngay cả **quyền đọc cũng
   đủ để leo thang khá nhanh**. Hai biện pháp: dùng **credential mạnh cho kết nối apiserver ↔ etcd,
   chẳng hạn xác thực lẫn nhau qua TLS client certificate**, và **cô lập các máy chủ etcd sau một
   tường lửa mà chỉ API server truy cập được**.
4. **Bị thu hồi hoặc bị gỡ bỏ quyền** ngay khi giai đoạn bootstrap hoàn tất. Nguyên tắc chung:
   **vòng đời credential càng ngắn thì kẻ tấn công càng khó lợi dụng** — đặt thời hạn ngắn cho
   certificate, tự động hóa việc xoay vòng, và dùng token có thời gian hiệu lực ngắn ở bất cứ đâu
   có thể.
5. Bài nói kubelet phơi các endpoint HTTPS **trao quyền kiểm soát rất mạnh đối với node và các
   container**, và **theo mặc định cho phép truy cập không cần xác thực** vào API đó. Vì vậy
   **cluster production phải bật xác thực và phân quyền cho kubelet** — đây là một trong những mục
   đáng rà lại đầu tiên khi tiếp quản một cluster lạ.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
