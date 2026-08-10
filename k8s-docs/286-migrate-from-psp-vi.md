# Di chuyển từ PodSecurityPolicy sang PodSecurity Admission Controller tích hợp sẵn (Migrate from PodSecurityPolicy to the Built-In PodSecurity Admission Controller)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/migrate-from-psp/>

Trang này mô tả quy trình di chuyển từ PodSecurityPolicy sang admission controller PodSecurity
tích hợp sẵn. Việc này có thể được thực hiện hiệu quả bằng cách kết hợp dry-run cùng các chế độ
`audit` và `warn`, mặc dù sẽ trở nên khó hơn nếu các PSP có tính chất mutating (tự động sửa đổi
Pod) đang được sử dụng.

## Trước khi bạn bắt đầu (Before you begin)

Kubernetes server của bạn phải ở phiên bản v1.22 trở lên. Để kiểm tra phiên bản, hãy nhập
`kubectl version`.

Nếu bạn hiện đang chạy một phiên bản Kubernetes khác 1.36, bạn có thể muốn chuyển sang xem
trang này trong tài liệu của đúng phiên bản Kubernetes mà bạn đang chạy.

Trang này giả định rằng bạn đã quen với các khái niệm cơ bản về
[Pod Security Admission](./116-pod-security-admission-vi.md).

## Cách tiếp cận tổng thể (Overall approach)

Có nhiều chiến lược bạn có thể áp dụng để di chuyển từ PodSecurityPolicy sang Pod Security
Admission. Các bước sau đây là một lộ trình di chuyển khả dĩ, với mục tiêu giảm thiểu cả rủi ro
gây gián đoạn môi trường production lẫn rủi ro tạo ra lỗ hổng bảo mật.

0. Quyết định xem Pod Security Admission có phù hợp với trường hợp sử dụng của bạn không.
1. Rà soát quyền trên namespace
2. Đơn giản hóa và chuẩn hóa các PodSecurityPolicy
3. Cập nhật các namespace
   1. Xác định mức Pod Security phù hợp
   2. Kiểm chứng mức Pod Security
   3. Thực thi mức Pod Security
   4. Bỏ qua PodSecurityPolicy
4. Rà soát các quy trình tạo namespace
5. Tắt PodSecurityPolicy

## 0. Quyết định xem Pod Security Admission có phù hợp với bạn không (Decide whether Pod Security Admission is right for you) {#is-psa-right-for-you}

Pod Security Admission được thiết kế để đáp ứng sẵn các nhu cầu bảo mật phổ biến nhất, và để
cung cấp một bộ mức bảo mật chuẩn dùng chung cho các cluster. Tuy nhiên, nó kém linh hoạt hơn
PodSecurityPolicy. Đáng chú ý, các tính năng sau được PodSecurityPolicy hỗ trợ nhưng Pod
Security Admission thì không:

- **Đặt các ràng buộc bảo mật mặc định** - Pod Security Admission là một admission controller
  không mutating (non-mutating), nghĩa là nó sẽ không sửa đổi Pod trước khi kiểm tra tính hợp lệ
  (validate) của chúng. Nếu bạn đang dựa vào khía cạnh này của PSP, bạn sẽ cần hoặc sửa đổi
  workload của mình để đáp ứng các ràng buộc Pod Security, hoặc dùng một
  [Mutating Admission Webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
  để thực hiện những thay đổi đó. Xem
  [Đơn giản hóa và chuẩn hóa các PodSecurityPolicy](#simplify-psps) bên dưới để biết thêm chi tiết.
- **Kiểm soát chi tiết đối với định nghĩa chính sách** - Pod Security Admission chỉ hỗ trợ
  [3 mức chuẩn](./115-pod-security-standards-vi.md).
  Nếu bạn cần kiểm soát nhiều hơn đối với các ràng buộc cụ thể, bạn sẽ cần dùng một
  [Validating Admission Webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
  để thực thi các chính sách đó.
- **Độ chi tiết chính sách dưới mức namespace** - PodSecurityPolicy cho phép bạn gắn các chính
  sách khác nhau cho các Service Account hoặc người dùng khác nhau, ngay cả trong cùng một
  namespace. Cách tiếp cận này có nhiều cạm bẫy và không được khuyến nghị, nhưng nếu bạn vẫn cần
  tính năng này thì bạn sẽ phải dùng một webhook của bên thứ ba thay thế. Ngoại lệ là khi bạn
  chỉ cần miễn trừ hoàn toàn cho những người dùng cụ thể hoặc các
  [RuntimeClass](./43-runtime-class-vi.md), trong trường hợp đó Pod Security Admission có cung
  cấp một số
  [cấu hình tĩnh cho việc miễn trừ](./116-pod-security-admission-vi.md#miễn-trừ-exemptions).

Ngay cả khi Pod Security Admission không đáp ứng mọi nhu cầu của bạn, nó được thiết kế để _bổ
trợ_ cho các cơ chế thực thi chính sách khác, và có thể là một lớp dự phòng hữu ích chạy song
song với các admission webhook khác.

## 1. Rà soát quyền trên namespace (Review namespace permissions) {#review-namespace-permissions}

Pod Security Admission được điều khiển bởi
[các label trên namespace](./116-pod-security-admission-vi.md#các-label-pod-security-admission-cho-namespace-pod-security-admission-labels-for-namespaces).
Điều này nghĩa là bất kỳ ai có thể cập nhật (hoặc patch hoặc tạo) một namespace cũng có thể
thay đổi mức Pod Security cho namespace đó — điều này có thể bị lợi dụng để vượt qua một chính
sách hạn chế hơn. Trước khi tiếp tục, hãy bảo đảm rằng chỉ những người dùng đặc quyền, đáng tin
cậy mới có các quyền namespace này. Không nên cấp các quyền mạnh này cho những người dùng không
đáng có quyền nâng cao, nhưng nếu bắt buộc, bạn sẽ cần dùng một
[admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
để đặt thêm hạn chế lên việc thiết lập các label Pod Security trên đối tượng Namespace.

## 2. Đơn giản hóa và chuẩn hóa các PodSecurityPolicy (Simplify & standardize PodSecurityPolicies) {#simplify-psps}

Trong mục này, bạn sẽ giảm bớt các PodSecurityPolicy mutating và loại bỏ các tùy chọn nằm ngoài
phạm vi của Pod Security Standards. Bạn nên thực hiện các thay đổi được khuyến nghị ở đây trên
một bản sao offline của PodSecurityPolicy gốc đang được chỉnh sửa. Bản PSP nhân bản nên có tên
khác, đứng trước tên gốc theo thứ tự bảng chữ cái (ví dụ, thêm số `0` vào đầu tên). Chưa vội
tạo các chính sách mới trong Kubernetes — việc đó sẽ được đề cập trong mục
[Triển khai các chính sách đã cập nhật](#psp-update-rollout) bên dưới.

### 2.a. Loại bỏ các field thuần túy mutating (Eliminate purely mutating fields) {#eliminate-mutating-fields}

Nếu một PodSecurityPolicy đang mutating (tự động sửa đổi) Pod, bạn có thể rơi vào tình huống có
những Pod không đáp ứng yêu cầu của mức Pod Security khi cuối cùng bạn tắt PodSecurityPolicy.
Để tránh điều này, bạn nên loại bỏ mọi hành vi mutation của PSP trước khi chuyển đổi. Đáng
tiếc, PSP không tách bạch rõ ràng giữa các field mutating và validating, vì vậy đây không phải
là một cuộc di chuyển đơn giản.

Bạn có thể bắt đầu bằng cách loại bỏ các field thuần túy mutating và không có ảnh hưởng gì tới
chính sách validating. Các field này (cũng được liệt kê trong tài liệu tham khảo
[Ánh xạ PodSecurityPolicy sang Pod Security Standards](https://kubernetes.io/docs/reference/access-authn-authz/psp-to-pod-security-standards/))
là:

- `.spec.defaultAllowPrivilegeEscalation`
- `.spec.runtimeClass.defaultRuntimeClassName`
- `.metadata.annotations['seccomp.security.alpha.kubernetes.io/defaultProfileName']`
- `.metadata.annotations['apparmor.security.beta.kubernetes.io/defaultProfileName']`
- `.spec.defaultAddCapabilities` - Mặc dù về mặt kỹ thuật đây là field vừa mutating vừa
  validating, các giá trị này nên được gộp vào `.spec.allowedCapabilities`, field thực hiện
  cùng việc kiểm tra tính hợp lệ mà không mutation.

> **Thận trọng:** Việc loại bỏ các field này có thể khiến workload thiếu cấu hình cần thiết và
> gây ra sự cố. Xem [Triển khai các chính sách đã cập nhật](#psp-update-rollout) bên dưới để
> biết cách triển khai các thay đổi này một cách an toàn.

### 2.b. Loại bỏ các tùy chọn không được Pod Security Standards bao phủ (Eliminate options not covered by the Pod Security Standards) {#eliminate-non-standard-options}

Có một số field trong PodSecurityPolicy không được Pod Security Standards bao phủ. Nếu bạn bắt
buộc phải thực thi các tùy chọn này, bạn sẽ cần bổ sung cho Pod Security Admission một
[admission webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/),
điều này nằm ngoài phạm vi của hướng dẫn này.

Trước tiên, bạn có thể loại bỏ các field thuần túy validating mà Pod Security Standards không
bao phủ. Các field này (cũng được liệt kê trong tài liệu tham khảo
[Ánh xạ PodSecurityPolicy sang Pod Security Standards](https://kubernetes.io/docs/reference/access-authn-authz/psp-to-pod-security-standards/)
với ghi chú "no opinion") là:

- `.spec.allowedHostPaths`
- `.spec.allowedFlexVolumes`
- `.spec.allowedCSIDrivers`
- `.spec.forbiddenSysctls`
- `.spec.runtimeClass`

Bạn cũng có thể loại bỏ các field sau, vốn liên quan đến kiểm soát group POSIX / UNIX.

> **Thận trọng:** Nếu bất kỳ field nào trong số này dùng chiến lược `MustRunAs`, chúng có thể
> là mutating! Việc loại bỏ chúng có thể khiến workload không đặt các group cần thiết và gây ra
> sự cố. Xem [Triển khai các chính sách đã cập nhật](#psp-update-rollout) bên dưới để biết cách
> triển khai các thay đổi này một cách an toàn.

- `.spec.runAsGroup`
- `.spec.supplementalGroups`
- `.spec.fsGroup`

Các field mutating còn lại là cần thiết để hỗ trợ đúng Pod Security Standards, và sẽ cần được
xử lý theo từng trường hợp cụ thể ở giai đoạn sau:

- `.spec.requiredDropCapabilities` - Cần thiết để drop `ALL` cho profile Restricted.
- `.spec.seLinux` - (Chỉ mutating với rule `MustRunAs`) cần thiết để thực thi các yêu cầu
  SELinux của các profile Baseline và Restricted.
- `.spec.runAsUser` - (Không mutating với rule `RunAsAny`) cần thiết để thực thi `RunAsNonRoot`
  cho profile Restricted.
- `.spec.allowPrivilegeEscalation` - (Chỉ mutating nếu được đặt thành `false`) cần thiết cho
  profile Restricted.

### 2.c. Triển khai các PSP đã cập nhật (Rollout the updated PSPs) {#psp-update-rollout}

Tiếp theo, bạn có thể triển khai các chính sách đã cập nhật lên cluster của mình. Bạn nên tiến
hành thận trọng, vì việc loại bỏ các tùy chọn mutating có thể khiến workload thiếu cấu hình cần
thiết.

Với mỗi PodSecurityPolicy đã cập nhật:

1. Xác định các Pod đang chạy dưới PSP gốc. Việc này có thể được thực hiện bằng annotation
   `kubernetes.io/psp`. Ví dụ, dùng kubectl:
   ```sh
   PSP_NAME="original" # Đặt tên của PSP mà bạn đang kiểm tra
   kubectl get pods --all-namespaces -o jsonpath="{range .items[?(@.metadata.annotations.kubernetes\.io\/psp=='$PSP_NAME')]}{.metadata.namespace} {.metadata.name}{'\n'}{end}"
   ```
2. So sánh các Pod đang chạy này với spec Pod ban đầu để xác định xem PodSecurityPolicy có sửa
   đổi Pod hay không. Với các Pod được tạo bởi một
   [tài nguyên workload](./62-controllers-index-vi.md), bạn có thể so sánh Pod với PodTemplate
   trong tài nguyên controller. Nếu phát hiện bất kỳ thay đổi nào, Pod gốc hoặc PodTemplate cần
   được cập nhật với cấu hình mong muốn. Các field cần rà soát là:
   - `.metadata.annotations['container.apparmor.security.beta.kubernetes.io/*']` (thay * bằng tên từng container)
   - `.spec.runtimeClassName`
   - `.spec.securityContext.fsGroup`
   - `.spec.securityContext.seccompProfile`
   - `.spec.securityContext.seLinuxOptions`
   - `.spec.securityContext.supplementalGroups`
   - Trên các container, dưới `.spec.containers[*]` và `.spec.initContainers[*]`:
       - `.securityContext.allowPrivilegeEscalation`
       - `.securityContext.capabilities.add`
       - `.securityContext.capabilities.drop`
       - `.securityContext.readOnlyRootFilesystem`
       - `.securityContext.runAsGroup`
       - `.securityContext.runAsNonRoot`
       - `.securityContext.runAsUser`
       - `.securityContext.seccompProfile`
       - `.securityContext.seLinuxOptions`
3. Tạo các PodSecurityPolicy mới. Nếu có Role hoặc ClusterRole nào đang cấp quyền `use` trên
   tất cả các PSP, điều này có thể khiến các PSP mới được sử dụng thay cho các bản mutating
   tương ứng của chúng.
4. Cập nhật cấu hình phân quyền (authorization) của bạn để cấp quyền truy cập vào các PSP mới.
   Trong RBAC, điều này nghĩa là cập nhật mọi Role hoặc ClusterRole đang cấp quyền `use` trên
   PSP gốc để cũng cấp quyền đó cho PSP đã cập nhật.
5. Kiểm chứng: sau một khoảng thời gian theo dõi (soak time), chạy lại lệnh ở bước 1 để xem còn
   Pod nào đang dùng các PSP gốc không. Lưu ý rằng các Pod cần được tạo lại sau khi các chính
   sách mới đã được triển khai thì mới có thể kiểm chứng đầy đủ.
6. (tùy chọn) Khi bạn đã kiểm chứng rằng các PSP gốc không còn được sử dụng, bạn có thể xóa
   chúng.

## 3. Cập nhật các Namespace (Update Namespaces) {#update-namespaces}

Các bước sau đây cần được thực hiện trên mọi namespace trong cluster. Các lệnh được tham chiếu
trong những bước này dùng biến `$NAMESPACE` để chỉ namespace đang được cập nhật.

### 3.a. Xác định mức Pod Security phù hợp (Identify an appropriate Pod Security level) {#identify-appropriate-level}

Hãy bắt đầu bằng việc xem lại [Pod Security Standards](./115-pod-security-standards-vi.md) và
làm quen với 3 mức khác nhau.

Có một số cách để chọn mức Pod Security cho namespace của bạn:

1. **Theo yêu cầu bảo mật của namespace** - Nếu bạn nắm rõ mức truy cập dự kiến của namespace,
   bạn có thể chọn một mức phù hợp dựa trên các yêu cầu đó, tương tự như cách người ta tiếp cận
   việc này trên một cluster mới.
2. **Theo các PodSecurityPolicy hiện có** - Bằng tài liệu tham khảo
   [Ánh xạ PodSecurityPolicy sang Pod Security Standards](https://kubernetes.io/docs/reference/access-authn-authz/psp-to-pod-security-standards/),
   bạn có thể ánh xạ mỗi PSP sang một mức Pod Security Standard. Nếu các PSP của bạn không dựa
   trên Pod Security Standards, bạn có thể phải quyết định giữa việc chọn một mức ít nhất cũng
   rộng rãi bằng PSP, và một mức ít nhất cũng hạn chế bằng. Bạn có thể xem những PSP nào đang
   được dùng cho các Pod trong một namespace nhất định bằng lệnh này:
   ```sh
   kubectl get pods -n $NAMESPACE -o jsonpath="{.items[*].metadata.annotations.kubernetes\.io\/psp}" | tr " " "\n" | sort -u
   ```
3. **Theo các Pod hiện có** - Bằng các chiến lược trong mục
   [Kiểm chứng mức Pod Security](#verify-pss-level), bạn có thể thử cả hai mức Baseline và
   Restricted để xem chúng có đủ rộng rãi cho các workload hiện có hay không, và chọn mức hợp
   lệ có đặc quyền thấp nhất.

> **Thận trọng:** Các phương án 2 và 3 ở trên dựa trên các Pod _hiện có_, và có thể bỏ sót
> những workload hiện không chạy, chẳng hạn như CronJob, các workload scale về 0, hoặc các
> workload khác chưa được triển khai.

### 3.b. Kiểm chứng mức Pod Security (Verify the Pod Security level) {#verify-pss-level}

Khi bạn đã chọn một mức Pod Security cho namespace (hoặc nếu bạn đang thử nhiều mức), nên kiểm
thử nó trước (bạn có thể bỏ qua bước này nếu dùng mức Privileged). Pod Security bao gồm một số
công cụ giúp kiểm thử và triển khai các profile một cách an toàn.

Trước tiên, bạn có thể dry-run chính sách; việc này sẽ đánh giá các Pod hiện đang chạy trong
namespace theo chính sách được áp dụng, mà không làm cho chính sách mới có hiệu lực:
```sh
# $LEVEL là mức để dry-run, "baseline" hoặc "restricted".
kubectl label --dry-run=server --overwrite ns $NAMESPACE pod-security.kubernetes.io/enforce=$LEVEL
```
Lệnh này sẽ trả về cảnh báo cho bất kỳ Pod _hiện có_ nào không hợp lệ theo mức được đề xuất.

Phương án thứ hai tốt hơn để bắt các workload hiện không chạy: chế độ audit. Khi chạy ở chế độ
audit (khác với enforcing), các Pod vi phạm mức chính sách được ghi lại trong audit log — có
thể được xem lại sau một khoảng thời gian theo dõi — nhưng không bị từ chối. Chế độ warning
hoạt động tương tự, nhưng trả cảnh báo về cho người dùng ngay lập tức. Bạn có thể đặt mức audit
trên một namespace bằng lệnh này:
```sh
kubectl label --overwrite ns $NAMESPACE pod-security.kubernetes.io/audit=$LEVEL
```

Nếu một trong hai cách tiếp cận này cho ra các vi phạm ngoài dự kiến, bạn sẽ cần hoặc cập nhật
các workload vi phạm để đáp ứng yêu cầu của chính sách, hoặc nới lỏng mức Pod Security của
namespace.

### 3.c. Thực thi mức Pod Security (Enforce the Pod Security level) {#enforce-pod-security-level}

Khi bạn hài lòng rằng mức đã chọn có thể được thực thi an toàn trên namespace, bạn có thể cập
nhật namespace để thực thi mức mong muốn:

```sh
kubectl label --overwrite ns $NAMESPACE pod-security.kubernetes.io/enforce=$LEVEL
```

### 3.d. Bỏ qua PodSecurityPolicy (Bypass PodSecurityPolicy) {#bypass-psp}

Cuối cùng, bạn có thể vô hiệu hóa PodSecurityPolicy một cách hiệu quả ở cấp namespace bằng cách
gắn (bind) [PSP privileged có đầy đủ đặc quyền](https://k8s.io/examples/policy/privileged-psp.yaml)
cho tất cả các service account trong namespace.

Nội dung file `privileged-psp.yaml`:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  privileged: true
  allowPrivilegeEscalation: true
  allowedCapabilities:
  - '*'
  volumes:
  - '*'
  hostNetwork: true
  hostPorts:
  - min: 0
    max: 65535
  hostIPC: true
  hostPID: true
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

```sh
# Các lệnh phạm vi cluster sau đây chỉ cần chạy một lần.
kubectl apply -f privileged-psp.yaml
kubectl create clusterrole privileged-psp --verb use --resource podsecuritypolicies.policy --resource-name privileged

# Vô hiệu hóa theo từng namespace
kubectl create -n $NAMESPACE rolebinding disable-psp --clusterrole privileged-psp --group system:serviceaccounts:$NAMESPACE
```

Vì PSP privileged là non-mutating, và admission controller PSP luôn ưu tiên các PSP
non-mutating, cách này bảo đảm rằng các Pod trong namespace này không còn bị PodSecurityPolicy
sửa đổi hay hạn chế nữa.

Ưu điểm của việc tắt PodSecurityPolicy theo từng namespace như thế này là nếu có sự cố phát
sinh, bạn có thể dễ dàng hoàn tác thay đổi bằng cách xóa RoleBinding. Chỉ cần bảo đảm rằng các
PodSecurityPolicy có từ trước vẫn còn nguyên!

```sh
# Hoàn tác việc vô hiệu hóa PodSecurityPolicy.
kubectl delete -n $NAMESPACE rolebinding disable-psp
```

## 4. Rà soát các quy trình tạo namespace (Review namespace creation processes) {#review-namespace-creation-process}

Giờ đây khi các namespace hiện có đã được cập nhật để thực thi Pod Security Admission, bạn nên
bảo đảm rằng các quy trình và/hoặc chính sách tạo namespace mới của bạn cũng được cập nhật, sao
cho một profile Pod Security phù hợp được áp dụng cho các namespace mới.

Bạn cũng có thể cấu hình tĩnh admission controller Pod Security để đặt mức enforce, audit
và/hoặc warn mặc định cho các namespace không có label. Xem
[Cấu hình Admission Controller](./282-enforce-standards-admission-controller-vi.md#cấu-hình-admission-controller-configure-the-admission-controller)
để biết thêm thông tin.

## 5. Tắt PodSecurityPolicy (Disable PodSecurityPolicy) {#disable-psp}

Cuối cùng, bạn đã sẵn sàng tắt PodSecurityPolicy. Để làm điều đó, bạn sẽ cần sửa đổi cấu hình
admission của API server:
[Làm thế nào để tắt một admission controller?](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#how-do-i-turn-off-an-admission-controller)

Để xác minh rằng admission controller PodSecurityPolicy không còn được bật, bạn có thể chạy thử
thủ công bằng cách mạo danh (impersonate) một người dùng không có quyền truy cập bất kỳ
PodSecurityPolicy nào (xem
[ví dụ về PodSecurityPolicy](https://kubernetes.io/docs/concepts/security/pod-security-policy/#example)),
hoặc bằng cách kiểm tra trong log của API server. Lúc khởi động, API server ghi ra các dòng log
liệt kê các plugin admission controller đã được nạp:

```
I0218 00:59:44.903329      13 plugins.go:158] Loaded 16 mutating admission controller(s) successfully in the following order: NamespaceLifecycle,LimitRanger,ServiceAccount,NodeRestriction,TaintNodesByCondition,Priority,DefaultTolerationSeconds,ExtendedResourceToleration,PersistentVolumeLabel,DefaultStorageClass,StorageObjectInUseProtection,RuntimeClass,DefaultIngressClass,MutatingAdmissionWebhook.
I0218 00:59:44.903350      13 plugins.go:161] Loaded 14 validating admission controller(s) successfully in the following order: LimitRanger,ServiceAccount,PodSecurity,Priority,PersistentVolumeClaimResize,RuntimeClass,CertificateApproval,CertificateSigning,CertificateSubjectRestriction,DenyServiceExternalIPs,ValidatingAdmissionWebhook,ResourceQuota.
```

Bạn sẽ thấy `PodSecurity` (trong danh sách các validating admission controller), và cả hai danh
sách đều không được chứa `PodSecurityPolicy`.

Khi bạn chắc chắn rằng admission controller PSP đã bị tắt (và sau một khoảng thời gian theo dõi
đủ dài để tự tin rằng bạn sẽ không cần hoàn tác), bạn có thể thoải mái xóa các
PodSecurityPolicy của mình cùng mọi Role, ClusterRole, RoleBinding và ClusterRoleBinding liên
quan (chỉ cần bảo đảm rằng chúng không cấp thêm quyền nào khác không liên quan).
