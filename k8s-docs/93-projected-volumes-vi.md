# Volume dạng projected (Projected Volumes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/storage/projected-volumes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 6](00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), bài 6/16 · Kiểm chứng ở
Lab 6a (chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab)).

Bài này không phải về lưu trữ bền vững — nó là về cách gộp nhiều nguồn dữ liệu của Kubernetes
vào **một thư mục duy nhất** trong container. Bạn đã dùng ConfigMap và Secret ở giai đoạn 3,
nên phần cần hiểu ở đây chỉ là phần trên của bài. Ba mục cuối (`clusterTrustBundle`,
`podCertificate`, Windows) đều là tính năng beta cần feature gate, đọc để biết là đủ.

**Phải hiểu ở lần đọc này:**

- Volume `projected` ánh xạ **nhiều nguồn** vào cùng một thư mục, và **mọi nguồn phải nằm cùng
  namespace với Pod** — mục *Giới thiệu*.
- Hai khác biệt cú pháp so với volume rời: với secret, trường `secretName` đổi thành `name`;
  và `defaultMode` chỉ đặt được ở **cấp projected**, còn `mode` thì đặt được cho từng projection
  — đoạn ngay sau hai ví dụ cấu hình.
- `serviceAccountToken` tiêm token của ServiceAccount vào một đường dẫn để container xác thực
  với API server; ba trường cần nắm là `audience` (bên nhận phải tự nhận đúng định danh này,
  nếu không phải từ chối token), `expirationSeconds` (mặc định 1 giờ, **tối thiểu 600 giây**)
  và `path` (tương đối so với điểm mount) — mục *Volume projected serviceAccountToken*.
- Bẫy `subPath`: container mount một nguồn projected qua `subPath` sẽ **không nhận được cập
  nhật** cho nguồn đó — ghi chú cuối mục *Volume projected serviceAccountToken*.
- Trên Linux, khi mọi container trong Pod dùng cùng `runAsUser`, kubelet đặt nội dung volume
  `serviceAccountToken` thuộc sở hữu user đó với quyền `0600`; ephemeral container thêm sau
  **không** làm đổi quyền đã đặt và phải dùng đúng `runAsUser` đó mới đọc được token — mục
  *Tương tác với SecurityContext*, phần *Linux*.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Volume projected clusterTrustBundle* | beta, cần feature gate và `--runtime-config` riêng của kube-apiserver | giai đoạn 9 |
| *Volume projected podCertificate* | beta, cần feature gate; thuộc mảng quản lý certificate | giai đoạn 12, bài [156](156-certificates-vi.md) |
| Phần *Windows* trong *Tương tác với SecurityContext* | cluster lab không có node Windows | giai đoạn 15 |

---

Tài liệu này mô tả *volume dạng projected* (projected volume) trong Kubernetes. Bạn nên làm quen trước với [volume](91-volumes-vi.md).

## Giới thiệu (Introduction)

Một volume `projected` ánh xạ nhiều nguồn volume có sẵn vào cùng một thư mục.

Hiện tại, các loại nguồn volume sau có thể được projected:

* [`secret`](91-volumes-vi.md#secret)
* [`downwardAPI`](91-volumes-vi.md#downwardapi)
* [`configMap`](91-volumes-vi.md#configmap)
* [`serviceAccountToken`](#serviceaccounttoken)
* [`clusterTrustBundle`](#clustertrustbundle)
* [`podCertificate`](#podcertificate)

Tất cả các nguồn bắt buộc phải nằm trong cùng namespace với Pod. Để biết thêm chi tiết,
xem tài liệu thiết kế [all-in-one volume](https://git.k8s.io/design-proposals-archive/node/all-in-one-volume.md).

### Ví dụ cấu hình với một secret, một downwardAPI và một configMap (Example configuration with a secret, a downwardAPI, and a configMap) {#example-configuration-secret-downwardapi-configmap}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox:1.28
    command: ["sleep", "3600"]
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
          - key: username
            path: my-group/my-username
      - downwardAPI:
          items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "cpu_limit"
            resourceFieldRef:
              containerName: container-test
              resource: limits.cpu
      - configMap:
          name: myconfigmap
          items:
          - key: config
            path: my-group/my-config
```

### Ví dụ cấu hình: các secret với chế độ quyền không mặc định (Example configuration: secrets with a non-default permission mode set) {#example-configuration-secrets-nondefault-permission-mode}

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox:1.28
    command: ["sleep", "3600"]
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
          - key: username
            path: my-group/my-username
      - secret:
          name: mysecret2
          items:
          - key: password
            path: my-group/my-password
            mode: 0777
```

Mỗi nguồn volume projected được liệt kê trong spec dưới trường `sources`. Các
tham số gần như giống nhau, với hai ngoại lệ:

* Đối với secret, trường `secretName` đã được đổi thành `name` để nhất quán
  với cách đặt tên của ConfigMap.
* `defaultMode` chỉ có thể được chỉ định ở cấp projected chứ không thể chỉ định
  cho từng nguồn volume. Tuy nhiên, như minh họa ở trên, bạn có thể đặt tường minh
  `mode` cho từng projection riêng lẻ.

## Volume projected serviceAccountToken (serviceAccountToken projected volumes) {#serviceaccounttoken}

Bạn có thể tiêm (inject) token của [service account](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens)
hiện tại vào một Pod tại một đường dẫn được chỉ định. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-token-test
spec:
  containers:
  - name: container-test
    image: busybox:1.28
    command: ["sleep", "3600"]
    volumeMounts:
    - name: token-vol
      mountPath: "/service-account"
      readOnly: true
  serviceAccountName: default
  volumes:
  - name: token-vol
    projected:
      sources:
      - serviceAccountToken:
          audience: api
          expirationSeconds: 3600
          path: token
```

Pod ví dụ này có một volume projected chứa token của service account được tiêm vào.
Các container trong Pod này có thể dùng token đó để truy cập Kubernetes API
server, xác thực với danh tính của [ServiceAccount của Pod](279-configure-service-account-vi.md).
Trường `audience` chứa đối tượng nhận (audience) dự kiến của
token. Bên nhận token phải tự định danh bằng một định danh được chỉ định
trong audience của token, nếu không thì phải từ chối token. Trường này
là tùy chọn và mặc định là định danh của API server.

`expirationSeconds` là khoảng thời gian hiệu lực dự kiến của token service account.
Mặc định là 1 giờ và phải ít nhất 10 phút (600 giây). Quản trị viên
cũng có thể giới hạn giá trị tối đa của nó bằng cách chỉ định tùy chọn
`--service-account-max-token-expiration` cho API server. Trường `path` chỉ định
đường dẫn tương đối so với điểm mount của volume projected.

> **Ghi chú:**
> Một container dùng nguồn volume projected làm volume mount [`subPath`](91-volumes-vi.md#using-subpath)
> sẽ không nhận được các cập nhật cho những nguồn volume đó.

## Volume projected clusterTrustBundle (clusterTrustBundle projected volumes) {#clustertrustbundle}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.33 [beta]`

> **Ghi chú:**
> Để dùng tính năng này trong Kubernetes v1.36, bạn phải bật hỗ trợ cho các object ClusterTrustBundle
> bằng [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `ClusterTrustBundle` và
> flag `--runtime-config=certificates.k8s.io/v1beta1/clustertrustbundles=true` của kube-apiserver,
> sau đó bật feature gate `ClusterTrustBundleProjection`.

Nguồn volume projected `clusterTrustBundle` tiêm nội dung của một hoặc nhiều
object [ClusterTrustBundle](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests#cluster-trust-bundles)
dưới dạng một file tự động cập nhật trong hệ thống file của container.

Các ClusterTrustBundle có thể được chọn theo [tên](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests#ctb-signer-unlinked)
hoặc theo [tên signer](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests#ctb-signer-linked).

Để chọn theo tên, dùng trường `name` để chỉ định một object ClusterTrustBundle duy nhất.

Để chọn theo tên signer, dùng trường `signerName` (và tùy chọn thêm trường
`labelSelector`) để chỉ định một tập các object ClusterTrustBundle sử dụng
tên signer đó. Nếu `labelSelector` không có mặt, thì tất cả
ClusterTrustBundle của signer đó được chọn.

Kubelet loại bỏ trùng lặp các certificate trong các object ClusterTrustBundle được chọn,
chuẩn hóa các biểu diễn PEM (loại bỏ comment và header), sắp xếp lại các certificate,
và ghi chúng vào file có tên theo `path`.
Khi tập các ClusterTrustBundle được chọn hoặc nội dung của chúng thay đổi, kubelet giữ cho file luôn được cập nhật.

Mặc định, kubelet sẽ ngăn Pod khởi động nếu ClusterTrustBundle được nêu tên không được tìm thấy,
hoặc nếu `signerName` / `labelSelector` không khớp với ClusterTrustBundle nào.
Nếu hành vi này không phải là điều bạn muốn, hãy đặt trường `optional` thành `true`,
và Pod sẽ khởi động với một file rỗng tại `path`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-ctb-name-test
spec:
  containers:
  - name: container-test
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: token-vol
      mountPath: "/root-certificates"
      readOnly: true
  serviceAccountName: default
  volumes:
  - name: token-vol
    projected:
      sources:
      - clusterTrustBundle:
          name: example
          path: example-roots.pem
      - clusterTrustBundle:
          signerName: "example.com/mysigner"
          labelSelector:
            matchLabels:
              version: live
          path: mysigner-roots.pem
          optional: true
```

## Volume projected podCertificate (podCertificate projected volumes) {#podcertificate}

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.35 [beta]`

> **Ghi chú:**
> Trong Kubernetes v1.36, bạn phải bật hỗ trợ cho Pod
> Certificates bằng [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `PodCertificateRequest`
> và flag `--runtime-config=certificates.k8s.io/v1beta1/podcertificaterequests=true`
> của kube-apiserver.

Nguồn volume projected `podCertificate` cấp phát (provision) một cách an toàn một private key
và một chuỗi certificate X.509 cho Pod dùng làm thông tin xác thực (credential) phía client hoặc server.
Kubelet sau đó sẽ xử lý việc làm mới private key và chuỗi certificate khi
chúng sắp hết hạn. Ứng dụng chỉ cần đảm bảo rằng nó
nạp lại file kịp thời khi file thay đổi, bằng một cơ chế như `inotify` hoặc
polling.

Mỗi projection `podCertificate` hỗ trợ các trường cấu hình sau:

* `signerName`: [Signer](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests#signers)
  mà bạn muốn cấp certificate. Lưu ý rằng các signer có thể có các yêu cầu
  truy cập riêng, và có thể từ chối cấp certificate cho Pod của bạn.
* `keyType`: Loại private key cần được sinh ra. Các giá trị hợp lệ là
  `ED25519`, `ECDSAP256`, `ECDSAP384`, `ECDSAP521`, `RSA3072`, và `RSA4096`.
* `maxExpirationSeconds`: Thời gian sống tối đa mà bạn chấp nhận cho
  certificate được cấp cho Pod. Nếu không đặt, sẽ mặc định là `86400` (24
  giờ). Phải ít nhất là `3600` (1 giờ), và nhiều nhất là `7862400` (91 ngày).
  Các signer tích hợp sẵn của Kubernetes bị giới hạn thời gian sống tối đa là `86400` (1
  ngày). Signer được phép cấp certificate có thời gian sống ngắn hơn
  giá trị bạn đã chỉ định.
* `credentialBundlePath`: Đường dẫn tương đối bên trong projection nơi
  credential bundle sẽ được ghi. Credential bundle là một file định dạng PEM,
  trong đó block đầu tiên là block "PRIVATE KEY" chứa một
  private key được serialize theo PKCS#8, và các block còn lại là các block "CERTIFICATE"
  tạo thành chuỗi certificate (leaf certificate và các certificate
  trung gian nếu có).
* `keyPath` và `certificateChainPath`: Các đường dẫn riêng biệt nơi Kubelet sẽ
  *chỉ* ghi private key hoặc chuỗi certificate.
* `userAnnotations`: một map cho phép bạn truyền thêm thông tin tới
  hiện thực của signer. Nó được sao chép nguyên văn vào trường
  `spec.unverifiedUserAnnotations` của các object
  [PodCertificateRequest](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests#pod-certificate-requests)
  mà Kubelet tạo. Các mục (entry) chịu cùng quy tắc kiểm tra hợp lệ như annotation
  trong metadata của object, với điểm bổ sung là tất cả các key phải có tiền tố domain.
  Không có hạn chế nào được đặt lên giá trị, ngoại trừ một giới hạn kích thước tổng thể trên
  toàn bộ trường. Ngoài các kiểm tra cơ bản này, API server không
  thực hiện thêm bất kỳ kiểm tra nào khác. Các hiện thực signer nên hết sức
  cẩn trọng khi sử dụng dữ liệu này. Signer không được mặc nhiên tin tưởng dữ liệu này
  khi chưa thực hiện các bước xác minh phù hợp trước. Signer nên
  tài liệu hóa các key và giá trị mà chúng hỗ trợ. Signer nên từ chối các request
  chứa key mà chúng không nhận ra.

> **Ghi chú:**
> Hầu hết ứng dụng nên ưu tiên dùng `credentialBundlePath` trừ khi chúng cần
> key và certificate trong các file riêng biệt vì lý do tương thích. Kubelet
> dùng một chiến lược ghi nguyên tử (atomic) dựa trên symlink để đảm bảo rằng khi bạn
> mở các file mà nó project, bạn đọc được hoặc là nội dung cũ hoặc là nội dung mới.
> Tuy nhiên, nếu bạn đọc key và chuỗi certificate từ các file riêng biệt, Kubelet
> có thể xoay vòng (rotate) credential sau lần đọc thứ nhất và trước lần đọc thứ hai của bạn,
> dẫn đến việc ứng dụng của bạn nạp một cặp key và certificate không khớp nhau.

```yaml
# Ví dụ spec của Pod dùng projection podCertificate để yêu cầu một private key
# ED25519, một certificate từ signer `coolcert.example.com/foo`, và
# ghi kết quả vào `/var/run/my-x509-credentials/credentialbundle.pem`.
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: podcertificate-pod
spec:
  serviceAccountName: default
  containers:
  - image: debian
    name: main
    command: ["sleep", "infinity"]
    volumeMounts:
    - name: my-x509-credentials
      mountPath: /var/run/my-x509-credentials
  volumes:
  - name: my-x509-credentials
    projected:
      defaultMode: 0644
      sources:
      - podCertificate:
          keyType: ED25519
          signerName: coolcert.example.com/foo
          credentialBundlePath: credentialbundle.pem
          userAnnotations:
            example.com/annotation1: "value1"
            example.com/annotation2: "value2"
```

## Tương tác với SecurityContext (SecurityContext interactions)

[Đề xuất](https://git.k8s.io/enhancements/keps/sig-storage/2451-service-account-token-volumes#proposal)
về xử lý quyền file trong cải tiến volume service account projected
đã giới thiệu việc các file được project có quyền sở hữu (owner permission) chính xác.

### Linux

Trong các Pod Linux có volume projected và `RunAsUser` được đặt trong
[`SecurityContext`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#security-context) của Pod,
các file được project có quyền sở hữu được đặt chính xác, bao gồm cả quyền sở hữu
của user trong container.

Khi tất cả các container trong một Pod có cùng `runAsUser` trong
[`PodSecurityContext`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#security-context)
hoặc trong
[`SecurityContext`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#security-context-1)
của container, thì kubelet đảm bảo rằng nội dung của volume `serviceAccountToken` thuộc sở hữu của user đó,
và file token có chế độ quyền được đặt là `0600`.

> **Ghi chú:**
> Các container tạm thời (ephemeral container)
> được thêm vào một Pod sau khi Pod được tạo sẽ *không* thay đổi quyền của volume đã được
> đặt khi Pod được tạo.
>
> Nếu quyền của volume `serviceAccountToken` của một Pod được đặt là `0600` vì
> tất cả các container khác trong Pod có cùng `runAsUser`, các ephemeral
> container phải dùng cùng `runAsUser` đó để có thể đọc được token.

### Windows

Trong các Pod Windows có volume projected và `RunAsUsername` được đặt trong
`SecurityContext` của Pod, quyền sở hữu không được áp đặt do cách các tài khoản
người dùng được quản lý trong Windows. Windows lưu trữ và quản lý các tài khoản người dùng
và nhóm cục bộ trong một file cơ sở dữ liệu gọi là Security Account Manager (SAM). Mỗi
container duy trì một bản sao riêng của cơ sở dữ liệu SAM, mà host không có
khả năng nhìn thấy khi container đang chạy. Các container Windows được
thiết kế để chạy phần user mode của hệ điều hành tách biệt với host,
do đó mới có việc duy trì một cơ sở dữ liệu SAM ảo. Kết quả là, kubelet chạy
trên host không có khả năng cấu hình động quyền sở hữu file của host
cho các tài khoản container được ảo hóa. Khuyến nghị rằng nếu các file trên
máy host cần được chia sẻ với container thì chúng nên được đặt vào
volume mount riêng của chúng, bên ngoài `C:\`.

Mặc định, các file được project sẽ có quyền sở hữu như dưới đây, minh họa cho
một file volume projected ví dụ:

```powershell
PS C:\> Get-Acl C:\var\run\secrets\kubernetes.io\serviceaccount\..2021_08_31_22_22_18.318230061\ca.crt | Format-List

Path   : Microsoft.PowerShell.Core\FileSystem::C:\var\run\secrets\kubernetes.io\serviceaccount\..2021_08_31_22_22_18.318230061\ca.crt
Owner  : BUILTIN\Administrators
Group  : NT AUTHORITY\SYSTEM
Access : NT AUTHORITY\SYSTEM Allow  FullControl
         BUILTIN\Administrators Allow  FullControl
         BUILTIN\Users Allow  ReadAndExecute, Synchronize
Audit  :
Sddl   : O:BAG:SYD:AI(A;ID;FA;;;SY)(A;ID;FA;;;BA)(A;ID;0x1200a9;;;BU)
```

Điều này ngụ ý rằng tất cả các user quản trị như `ContainerAdministrator` sẽ có
quyền đọc, ghi và thực thi, trong khi các user không phải quản trị sẽ có quyền đọc và
thực thi.

> **Ghi chú:**
> Nói chung, việc cấp cho container quyền truy cập vào host là không được khuyến khích vì nó có thể
> mở đường cho các lỗ hổng bảo mật tiềm ẩn.
>
> Tạo một Pod Windows với `RunAsUser` trong `SecurityContext` của nó sẽ khiến
> Pod bị kẹt mãi ở trạng thái `ContainerCreating`. Vì vậy, khuyến cáo không dùng
> tùy chọn `RunAsUser` (vốn chỉ dành cho Linux) với các Pod Windows.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 6:

1. Bạn đang mount một ConfigMap và một Secret bằng hai volume riêng. Chuyển sang một volume
   `projected` duy nhất thì đổi được gì, và ràng buộc nào **không** đổi?
2. Khi bê nguyên khối `secret:` từ một volume `secret` sang `projected.sources`, trường nào bắt
   buộc phải đổi tên? Còn `defaultMode` bạn từng đặt cho volume secret thì giờ đặt ở đâu, và
   muốn đặt quyền riêng cho một file thì dùng gì?
3. Một Pod trên worker của bạn cần gọi API server bằng danh tính ServiceAccount của chính nó.
   Bạn khai `expirationSeconds: 300` cho gọn. Có hợp lệ không? `audience` dùng để làm gì?
4. Bạn mount nguồn projected bằng `subPath` để lấy đúng một file cho gọn thư mục. Hệ quả là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Đổi được: **cả hai nguồn xuất hiện trong cùng một thư mục** thay vì hai điểm mount riêng, và
   bạn kiểm soát được đường dẫn của từng mục qua `items[].path`. **Không đổi**: mọi nguồn vẫn
   **phải nằm cùng namespace với Pod** — projected volume không cho phép lấy ConfigMap hay
   Secret từ namespace khác.
2. Trường **`secretName` phải đổi thành `name`**, để nhất quán với cách đặt tên của ConfigMap.
   `defaultMode` **chỉ đặt được ở cấp `projected`**, không đặt được cho từng nguồn; muốn quyền
   riêng cho một file thì đặt tường minh **`mode` cho từng projection** trong `items`, đúng như
   ví dụ `mode: 0777` của bài.
3. **Không hợp lệ.** `expirationSeconds` mặc định là 1 giờ và **phải ít nhất 600 giây** (10
   phút); quản trị viên còn có thể giới hạn thêm giá trị tối đa bằng
   `--service-account-max-token-expiration` trên API server. `audience` khai **bên nhận dự kiến
   của token**: bên nhận phải tự định danh bằng một định danh nằm trong audience của token, nếu
   không thì phải từ chối token đó. Bỏ trống thì mặc định là định danh của API server.
4. **Container đó sẽ không nhận được các cập nhật cho nguồn projected.** Đây là cùng một cái
   bẫy đã gặp với `configMap` và `secret` ở bài [91](91-volumes-vi.md): `subPath` cho bạn một
   file tĩnh chứ không phải một file được kubelet giữ đồng bộ. Với token của service account
   hay với `clusterTrustBundle` — vốn được thiết kế để tự làm mới — thì đây là lỗi nghiêm
   trọng, không chỉ là bất tiện.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
