# Danh sách kiểm tra bảo mật (Security Checklist)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/security-checklist/>
>
> Danh sách kiểm tra cơ bản (baseline checklist) để đảm bảo bảo mật trong các cluster Kubernetes.

Danh sách kiểm tra (checklist) này nhằm cung cấp một danh sách hướng dẫn cơ bản kèm các liên kết
đến tài liệu đầy đủ hơn cho từng chủ đề. Nó không tự nhận là đầy đủ
và sẽ tiếp tục được phát triển theo thời gian.

Về cách đọc và sử dụng tài liệu này:

- Thứ tự các chủ đề không phản ánh thứ tự ưu tiên.
- Một số mục trong danh sách kiểm tra được giải thích chi tiết trong đoạn văn bên dưới danh sách của mỗi phần.

> **Thận trọng:**
> Danh sách kiểm tra tự nó **không** đủ để đạt được một thế trận bảo mật (security posture) tốt.
> Một thế trận bảo mật tốt đòi hỏi sự chú ý và cải thiện liên tục, nhưng
> danh sách kiểm tra có thể là bước đầu tiên trên hành trình không có điểm dừng hướng tới
> sự sẵn sàng về bảo mật. Một số khuyến nghị trong danh sách này có thể quá
> chặt chẽ hoặc quá lỏng lẻo so với nhu cầu bảo mật cụ thể của bạn. Vì bảo mật Kubernetes
> không phải là "một khuôn mẫu chung cho tất cả" (one size fits all), mỗi nhóm mục trong danh sách kiểm tra
> nên được đánh giá dựa trên giá trị riêng của nó.

## Xác thực & Phân quyền (Authentication & Authorization)

- [ ] Nhóm `system:masters` không được sử dụng để xác thực người dùng hoặc thành phần sau khi bootstrap.
- [ ] kube-controller-manager đang chạy với `--use-service-account-credentials`
  được bật.
- [ ] Chứng chỉ gốc (root certificate) được bảo vệ (hoặc là một CA offline, hoặc một CA online
  được quản lý với các biện pháp kiểm soát truy cập hiệu quả).
- [ ] Các certificate trung gian (intermediate) và certificate lá (leaf) có ngày hết hạn không quá 3
  năm trong tương lai.
- [ ] Tồn tại một quy trình rà soát quyền truy cập định kỳ, và các lần rà soát cách nhau không
  quá 24 tháng.
- [ ] Tuân theo [Các thực hành tốt về Role Based Access Control](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
  để có hướng dẫn liên quan đến xác thực và phân quyền.

Sau khi bootstrap, cả người dùng lẫn các thành phần đều không nên xác thực với
Kubernetes API dưới danh nghĩa `system:masters`. Tương tự, nên tránh chạy toàn bộ
kube-controller-manager với quyền `system:masters`. Trên thực tế,
`system:masters` chỉ nên được dùng như một cơ chế khẩn cấp (break-glass), chứ không phải
như một người dùng quản trị.

## Bảo mật mạng (Network security)

- [ ] Các CNI plugin đang sử dụng có hỗ trợ network policy.
- [ ] Network policy cho ingress và egress được áp dụng cho tất cả workload trong
  cluster.
- [ ] Network policy mặc định trong mỗi namespace — chọn tất cả các pod và từ chối
  mọi thứ — đã được thiết lập.
- [ ] Nếu phù hợp, một service mesh được dùng để mã hóa toàn bộ giao tiếp bên trong cluster.
- [ ] Kubernetes API, kubelet API và etcd không bị phơi bày công khai trên Internet.
- [ ] Truy cập từ các workload đến cloud metadata API được lọc.
- [ ] Việc sử dụng LoadBalancer và ExternalIPs bị hạn chế.

Một số [Container Network Interface (CNI) plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
cung cấp chức năng hạn chế các tài nguyên mạng mà pod có thể giao tiếp.
Điều này thường được thực hiện thông qua [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) —
một tài nguyên thuộc phạm vi namespace để định nghĩa các quy tắc. Các network policy
mặc định chặn toàn bộ egress và ingress, trong mỗi namespace, chọn tất cả các pod, có thể
hữu ích để áp dụng cách tiếp cận danh sách cho phép (allow list) nhằm đảm bảo không bỏ sót workload nào.

Không phải CNI plugin nào cũng cung cấp mã hóa khi truyền (encryption in transit). Nếu plugin được chọn
thiếu tính năng này, một giải pháp thay thế là dùng service mesh để cung cấp
chức năng đó.

Kho dữ liệu etcd của control plane cần có các biện pháp kiểm soát để giới hạn truy cập và
không được phơi bày công khai trên Internet. Hơn nữa, nên dùng mutual TLS (mTLS)
để giao tiếp an toàn với nó. Certificate authority dùng cho việc này
nên là duy nhất, dành riêng cho etcd.

Truy cập Internet từ bên ngoài đến Kubernetes API server nên được hạn chế để
không phơi bày API công khai. Hãy cẩn thận, vì nhiều bản phân phối Kubernetes được quản lý (managed)
đang phơi bày API server công khai theo mặc định. Khi đó bạn có thể dùng một máy chủ trung gian (bastion host)
để truy cập server.

Truy cập đến API của [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
nên được hạn chế và không phơi bày công khai; các thiết lập xác thực và
phân quyền mặc định, khi không có file cấu hình nào được chỉ định qua flag `--config`,
là quá dễ dãi.

Nếu dùng nhà cung cấp cloud để chạy Kubernetes, truy cập từ các pod đến cloud
metadata API `169.254.169.254` cũng nên được hạn chế hoặc chặn nếu không cần thiết,
vì nó có thể làm rò rỉ thông tin.

Về việc hạn chế sử dụng LoadBalancer và ExternalIPs, xem
[CVE-2020-8554: Man in the middle using LoadBalancer or ExternalIPs](https://github.com/kubernetes/kubernetes/issues/97076)
và [admission controller DenyServiceExternalIPs](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#denyserviceexternalips)
để biết thêm thông tin.

## Bảo mật Pod (Pod security)

- [ ] Quyền RBAC để `create`, `update`, `patch`, `delete` workload chỉ được cấp khi cần thiết.
- [ ] Chính sách Pod Security Standards phù hợp được áp dụng và thực thi cho tất cả namespace.
- [ ] Giới hạn (limit) bộ nhớ được đặt cho các workload với limit bằng hoặc thấp hơn request.
- [ ] Giới hạn CPU có thể được đặt cho các workload nhạy cảm.
- [ ] Với các node hỗ trợ, Seccomp được bật với profile syscall phù hợp
  cho các chương trình.
- [ ] Với các node hỗ trợ, AppArmor hoặc SELinux được bật với profile
  phù hợp cho các chương trình.

Phân quyền RBAC rất quan trọng nhưng
[không thể đủ chi tiết để phân quyền trên các tài nguyên của Pod](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#workload-creation)
(hay trên bất kỳ tài nguyên nào quản lý Pod). Mức chi tiết duy nhất là các động từ (verb) API
trên chính tài nguyên đó, ví dụ `create` trên Pod. Nếu không có
admission bổ sung, quyền tạo các tài nguyên này cho phép truy cập trực tiếp
không hạn chế đến các node có thể lập lịch (schedulable) của cluster.

[Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
định nghĩa ba chính sách khác nhau — privileged, baseline và restricted — giới hạn
cách các trường có thể được đặt trong `PodSpec` liên quan đến bảo mật.
Các tiêu chuẩn này có thể được thực thi ở cấp namespace với admission
[Pod Security](https://kubernetes.io/docs/concepts/security/pod-security-admission/) mới,
được bật theo mặc định, hoặc bởi admission webhook của bên thứ ba. Xin lưu ý rằng,
trái với admission PodSecurityPolicy đã bị gỡ bỏ mà nó thay thế, admission
[Pod Security](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
có thể dễ dàng kết hợp với các admission webhook và dịch vụ bên ngoài.

Chính sách `restricted` của Pod Security admission — chính sách hạn chế nhất trong bộ
[Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) —
[có thể hoạt động ở nhiều chế độ](https://kubernetes.io/docs/concepts/security/pod-security-admission/#pod-security-admission-labels-for-namespaces):
`warn`, `audit` hoặc `enforce`, để dần dần áp dụng
[security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
phù hợp nhất theo các thực hành bảo mật tốt nhất. Dù vậy,
[security context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) của pod
vẫn nên được xem xét riêng để giới hạn các đặc quyền và quyền truy cập mà pod có thể có,
vượt trên các tiêu chuẩn bảo mật định sẵn, cho các trường hợp sử dụng cụ thể.

Để có hướng dẫn thực hành về [Pod Security](https://kubernetes.io/docs/concepts/security/pod-security-admission/),
xem bài blog
[Kubernetes 1.23: Pod Security Graduates to Beta](https://kubernetes.io/blog/2021/12/09/pod-security-admission-beta/).

[Giới hạn bộ nhớ và CPU](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
nên được đặt nhằm hạn chế lượng tài nguyên bộ nhớ và CPU mà một pod có thể
tiêu thụ trên node, và nhờ đó ngăn chặn các cuộc tấn công DoS tiềm tàng từ các workload
độc hại hoặc bị xâm nhập. Chính sách như vậy có thể được thực thi bởi một admission controller.
Xin lưu ý rằng giới hạn CPU sẽ điều tiết (throttle) mức sử dụng và do đó có thể gây ra
các tác động không mong muốn đối với các tính năng tự động mở rộng (auto-scaling) hay hiệu quả vận hành,
tức là chạy tiến trình theo kiểu best effort với tài nguyên CPU sẵn có.

> **Thận trọng:**
> Giới hạn bộ nhớ cao hơn request có thể khiến toàn bộ node gặp các vấn đề OOM.

### Bật Seccomp (Enabling Seccomp)

Seccomp là viết tắt của secure computing mode (chế độ điện toán an toàn) và là một tính năng của Linux kernel kể từ phiên bản 2.6.12.
Nó có thể được dùng để sandbox các đặc quyền của một tiến trình, hạn chế các lời gọi mà tiến trình có thể thực hiện
từ userspace vào kernel. Kubernetes cho phép bạn tự động áp dụng các seccomp profile được nạp trên
node cho các Pod và container của bạn.

Seccomp có thể cải thiện bảo mật cho workload của bạn bằng cách giảm bề mặt tấn công syscall
của Linux kernel sẵn có bên trong các container. Chế độ seccomp filter tận dụng BPF để tạo
danh sách cho phép hoặc từ chối các syscall cụ thể, gọi là các profile.

Kể từ Kubernetes 1.27, bạn có thể bật `RuntimeDefault` làm seccomp profile mặc định
cho tất cả workload. Đã có sẵn một [hướng dẫn bảo mật](https://kubernetes.io/docs/tutorials/security/seccomp/) về
chủ đề này. Ngoài ra,
[Kubernetes Security Profiles Operator](https://github.com/kubernetes-sigs/security-profiles-operator)
là một dự án hỗ trợ việc quản lý và sử dụng seccomp trong các cluster.

> **Ghi chú:**
> Seccomp chỉ khả dụng trên các node Linux.

### Bật AppArmor hoặc SELinux (Enabling AppArmor or SELinux)

#### AppArmor

[AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/) là một module bảo mật của Linux kernel có thể
cung cấp một cách dễ dàng để triển khai Kiểm soát truy cập bắt buộc (Mandatory Access Control - MAC) và
kiểm toán (auditing) tốt hơn thông qua log hệ thống. Một AppArmor profile mặc định được thực thi trên các node hỗ trợ nó, hoặc có thể cấu hình một profile tùy chỉnh.
Giống như seccomp, AppArmor cũng được cấu hình
thông qua các profile, trong đó mỗi profile hoặc chạy ở chế độ enforcing —
chặn truy cập vào các tài nguyên không được phép — hoặc chế độ complain — chỉ báo cáo
các vi phạm. AppArmor profile được thực thi theo từng container, thông qua một
annotation, cho phép các tiến trình chỉ nhận được đúng những đặc quyền cần thiết.

> **Ghi chú:**
> AppArmor chỉ khả dụng trên các node Linux, và được bật trong
> [một số bản phân phối Linux](https://gitlab.com/apparmor/apparmor/-/wikis/home#distributions-and-ports).

#### SELinux

[SELinux](https://github.com/SELinuxProject/selinux-notebook/blob/main/src/selinux_overview.md) cũng là một
module bảo mật của Linux kernel có thể cung cấp cơ chế hỗ trợ các chính sách bảo mật
kiểm soát truy cập, bao gồm Kiểm soát truy cập bắt buộc (MAC). Nhãn (label) SELinux
có thể được gán cho container hoặc pod
[thông qua mục `securityContext` của chúng](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#assign-selinux-labels-to-a-container).

> **Ghi chú:**
> SELinux chỉ khả dụng trên các node Linux, và được bật trong
> [một số bản phân phối Linux](https://en.wikipedia.org/wiki/Security-Enhanced_Linux#Implementations).

## Log và kiểm toán (Logs and auditing)

- [ ] Audit log, nếu được bật, được bảo vệ khỏi truy cập thông thường.

## Sắp xếp vị trí Pod (Pod placement)

- [ ] Việc sắp xếp vị trí Pod được thực hiện phù hợp với các mức độ nhạy cảm của
  ứng dụng.
- [ ] Các ứng dụng nhạy cảm được chạy cô lập trên các node riêng hoặc với các
  runtime được sandbox chuyên biệt.

Các pod thuộc các mức độ nhạy cảm khác nhau — ví dụ, một pod ứng dụng
và Kubernetes API server — nên được triển khai lên các node tách biệt. Mục đích
của việc cô lập node là ngăn việc một container ứng dụng thoát ra (breakout) rồi
trực tiếp có quyền truy cập vào các ứng dụng có mức độ nhạy cảm cao hơn, từ đó dễ dàng
di chuyển ngang (pivot) trong cluster. Sự tách biệt này nên được thực thi để tránh việc các pod
vô tình được triển khai lên cùng một node. Điều này có thể được thực thi bằng các
tính năng sau:

[Node Selectors](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
: Các cặp key-value, nằm trong đặc tả của pod, chỉ định các node sẽ
triển khai lên. Chúng có thể được thực thi ở cấp namespace và cluster với admission controller
[PodNodeSelector](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podnodeselector).

[PodTolerationRestriction](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podtolerationrestriction)
: Một admission controller cho phép quản trị viên hạn chế các
[toleration](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) được phép trong một
namespace. Các pod trong một namespace chỉ có thể sử dụng các toleration được chỉ định trên
các khóa annotation của đối tượng namespace, thứ cung cấp một tập các toleration mặc định
và được phép.

[RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
: RuntimeClass là tính năng để chọn cấu hình container runtime.
Cấu hình container runtime được dùng để chạy các container của một Pod và có thể
cung cấp mức cô lập với host nhiều hơn hoặc ít hơn, đánh đổi bằng chi phí
hiệu năng.

## Secret (Secrets)

- [ ] ConfigMap không được dùng để lưu dữ liệu bí mật.
- [ ] Mã hóa khi lưu trữ (encryption at rest) được cấu hình cho Secret API.
- [ ] Nếu phù hợp, một cơ chế inject các secret lưu trong hệ thống lưu trữ bên thứ ba
  đã được triển khai và sẵn sàng.
- [ ] Token của service account không được mount vào các pod không cần đến chúng.
- [ ] [Bound service account token volume](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#bound-service-account-token-volume)
  đang được sử dụng thay vì các token không hết hạn.

Các secret cần cho pod nên được lưu trong Kubernetes Secrets thay vì
các lựa chọn thay thế như ConfigMap. Các tài nguyên Secret lưu trong etcd nên
được [mã hóa khi lưu trữ](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/).

Các pod cần secret nên được tự động mount các secret này thông qua volume,
tốt nhất là lưu trong bộ nhớ như với [tùy chọn `emptyDir.medium`](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir).
Có thể dùng cơ chế để inject secret từ các hệ thống lưu trữ bên thứ ba dưới dạng
volume, như [Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/).
Cách này nên được ưu tiên hơn so với việc cấp cho service account của pod
quyền RBAC truy cập secret. Cách đó sẽ cho phép thêm secret vào pod dưới dạng
biến môi trường hoặc file. Xin lưu ý rằng phương pháp biến môi trường
có thể dễ bị rò rỉ hơn do các crash dump trong log và bản chất
không bảo mật của biến môi trường trong Linux, trái với
cơ chế phân quyền trên file.

Token của service account không nên được mount vào các pod không cần đến chúng. Điều này có thể được cấu hình bằng cách đặt
[`automountServiceAccountToken`](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server)
thành `false`, hoặc trong service account để áp dụng cho toàn bộ namespace,
hoặc riêng cho một pod. Với Kubernetes v1.22 trở lên, hãy dùng
[Bound Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#bound-service-account-token-volume)
để có thông tin xác thực service account có giới hạn thời gian.

## Image (Images)

- [ ] Giảm thiểu nội dung không cần thiết trong container image.
- [ ] Container image được cấu hình để chạy với người dùng không có đặc quyền (unprivileged).
- [ ] Tham chiếu đến container image được thực hiện bằng sha256 digest (thay vì
tag), hoặc nguồn gốc (provenance) của image được xác minh bằng cách kiểm tra chữ ký số
của image lúc triển khai [thông qua admission control](https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/#verifying-image-signatures-with-admission-controller).
- [ ] Container image được quét thường xuyên trong quá trình tạo và khi triển khai, và
  các phần mềm có lỗ hổng đã biết được vá.

Container image nên chứa mức tối thiểu cần thiết để chạy chương trình mà chúng
đóng gói. Tốt nhất là chỉ gồm chương trình và các phụ thuộc (dependency) của nó, build image
từ base tối giản nhất có thể. Đặc biệt, image dùng trong production không nên
chứa shell hay các tiện ích debug, vì có thể dùng
[ephemeral debug container](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container)
để xử lý sự cố.

Hãy build image sao cho khởi động trực tiếp với người dùng không có đặc quyền bằng
[chỉ thị `USER` trong Dockerfile](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#user).
[Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-the-security-context-for-a-pod)
cho phép một container image được khởi động với người dùng và nhóm cụ thể bằng
`runAsUser` và `runAsGroup`, ngay cả khi không được chỉ định trong manifest của image.
Tuy nhiên, quyền trên file trong các lớp (layer) của image có thể khiến việc chỉ đơn giản
khởi động tiến trình với một người dùng mới không có đặc quyền là bất khả thi nếu không chỉnh sửa image.

Tránh dùng tag để tham chiếu một image, đặc biệt là tag `latest`, vì
image đứng sau một tag có thể dễ dàng bị thay đổi trong registry. Ưu tiên dùng
`sha256` digest đầy đủ — giá trị duy nhất đối với manifest của image. Chính sách này có thể được
thực thi qua một [ImagePolicyWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook).
Chữ ký image cũng có thể được tự động [xác minh bằng một admission controller](https://kubernetes.io/docs/tasks/administer-cluster/verify-signed-artifacts/#verifying-image-signatures-with-admission-controller)
lúc triển khai để kiểm chứng tính xác thực và toàn vẹn của chúng.

Việc quét container image có thể ngăn các lỗ hổng nghiêm trọng được
triển khai vào cluster cùng với container image. Việc quét image nên được
hoàn tất trước khi triển khai một container image vào cluster và thường được thực hiện
như một phần của quy trình triển khai trong pipeline CI/CD. Mục đích của việc quét
image là thu thập thông tin về các lỗ hổng có thể có và cách
phòng ngừa chúng trong container image, chẳng hạn như điểm
[Common Vulnerability Scoring System (CVSS)](https://www.first.org/cvss/).
Nếu kết quả quét image được kết hợp với các quy tắc tuân thủ của pipeline,
chỉ những container image đã được vá đúng cách mới được đưa vào
Production.

## Admission controller (Admission controllers)

- [ ] Một tập hợp admission controller phù hợp được bật.
- [ ] Chính sách bảo mật pod được thực thi bởi Pod Security Admission và/hoặc một
  webhook admission controller.
- [ ] Các plugin trong chuỗi admission và các webhook được cấu hình an toàn.

Admission controller có thể giúp cải thiện bảo mật của cluster. Tuy nhiên,
bản thân chúng cũng có thể mang lại rủi ro vì chúng mở rộng API server và
[cần được bảo vệ đúng cách](https://kubernetes.io/blog/2022/01/19/secure-your-admission-controllers-and-webhooks/).

Các danh sách sau đây trình bày một số admission controller có thể được
cân nhắc để nâng cao thế trận bảo mật cho cluster và ứng dụng của bạn. Danh sách
bao gồm các controller có thể được nhắc đến ở các phần khác của tài liệu này.

Nhóm admission controller đầu tiên gồm các plugin
[được bật theo mặc định](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#which-plugins-are-enabled-by-default),
hãy cân nhắc giữ chúng ở trạng thái bật trừ khi bạn biết rõ mình đang làm gì:

[`CertificateApproval`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#certificateapproval)
: Thực hiện các kiểm tra phân quyền bổ sung để đảm bảo người dùng phê duyệt có
quyền phê duyệt yêu cầu certificate.

[`CertificateSigning`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#certificatesigning)
: Thực hiện các kiểm tra phân quyền bổ sung để đảm bảo người dùng ký có
quyền ký các yêu cầu certificate.

[`CertificateSubjectRestriction`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#certificatesubjectrestriction)
: Từ chối mọi yêu cầu certificate chỉ định 'group' (hay 'organization
attribute') là `system:masters`.

[`LimitRanger`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#limitranger)
: Thực thi các ràng buộc của LimitRange API.

[`MutatingAdmissionWebhook`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook)
: Cho phép sử dụng các controller tùy chỉnh thông qua webhook; các controller này có thể
thay đổi (mutate) các request mà chúng xem xét.

[`PodSecurity`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#podsecurity)
: Thay thế cho Pod Security Policy, hạn chế security context của các Pod
được triển khai.

[`ResourceQuota`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#resourcequota)
: Thực thi hạn ngạch tài nguyên (resource quota) để ngăn việc sử dụng tài nguyên quá mức.

[`ValidatingAdmissionWebhook`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook)
: Cho phép sử dụng các controller tùy chỉnh thông qua webhook; các controller này không
thay đổi các request mà chúng xem xét.

Nhóm thứ hai gồm các plugin không được bật theo mặc định nhưng đã ở trạng thái
phổ biến rộng rãi (general availability) và được khuyến nghị để cải thiện thế trận bảo mật của bạn:

[`DenyServiceExternalIPs`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#denyserviceexternalips)
: Từ chối mọi trường hợp sử dụng mới trường `Service.spec.externalIPs`. Đây là biện pháp giảm thiểu cho
[CVE-2020-8554: Man in the middle using LoadBalancer or ExternalIPs](https://github.com/kubernetes/kubernetes/issues/97076).

[`NodeRestriction`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)
: Giới hạn quyền của kubelet để chỉ có thể sửa đổi các tài nguyên API pod mà chúng sở hữu
hoặc tài nguyên API node đại diện cho chính chúng. Nó cũng ngăn kubelet
sử dụng annotation `node-restriction.kubernetes.io/`, thứ mà kẻ tấn công
có quyền truy cập thông tin xác thực của kubelet có thể lợi dụng để tác động đến việc sắp xếp
pod lên node đang bị kiểm soát.

Nhóm thứ ba gồm các plugin không được bật theo mặc định nhưng có thể được
cân nhắc cho một số trường hợp sử dụng nhất định:

[`AlwaysPullImages`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#alwayspullimages)
: Bắt buộc sử dụng phiên bản mới nhất của image theo tag và đảm bảo rằng người triển khai
có quyền sử dụng image đó.

[`ImagePolicyWebhook`](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook)
: Cho phép thực thi các kiểm soát bổ sung đối với image thông qua webhook.

## Tiếp theo (What's next)

- [Leo thang đặc quyền thông qua việc tạo Pod](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#privilege-escalation-via-pod-creation)
  cảnh báo bạn về một rủi ro kiểm soát truy cập cụ thể; hãy kiểm tra cách bạn đang quản lý
  mối đe dọa đó.
  - Nếu bạn dùng Kubernetes RBAC, hãy đọc
    [RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/) để biết
    thêm thông tin về phân quyền.
- [Securing a Cluster](https://kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/) để biết
  thông tin về việc bảo vệ cluster khỏi truy cập vô tình hoặc độc hại.
- [Hướng dẫn multi-tenancy cho cluster](https://kubernetes.io/docs/concepts/security/multi-tenancy/) để có
  các khuyến nghị về tùy chọn cấu hình và thực hành tốt về multi-tenancy.
- [Bài blog "A Closer Look at NSA/CISA Kubernetes Hardening Guidance"](https://kubernetes.io/blog/2021/10/05/nsa-cisa-kubernetes-hardening-guidance/#building-secure-container-images)
  là tài nguyên bổ trợ về tăng cường bảo mật (hardening) cluster Kubernetes.
