# Quản lý certificate với kubeadm (Certificate Management with kubeadm)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/>
>
> Các client certificate do kubeadm sinh ra hết hạn sau 1 năm. Trang này giải thích cách quản lý
> việc gia hạn certificate với kubeadm, cùng các tác vụ khác liên quan đến quản lý certificate
> của kubeadm.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Checkpoint tiếp nối — giai đoạn 18 Vòng đời chứng chỉ](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ),
bài 1/3 · thực hành trực tiếp trên cluster VM của [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md).

Bài này dài nhưng thực chất là **hai bài trong một**: nửa đầu là quy trình vận hành hằng ngày
(kiểm tra hạn, gia hạn) mà mọi admin cluster kubeadm phải thuộc; nửa sau là các quy trình cho
cluster dùng CA bên ngoài (external CA) — chỉ cần khi tổ chức của bạn có hạ tầng PKI riêng.
Cluster lab của bạn dùng CA do kubeadm tự sinh, nên nửa sau chỉ cần đọc lướt để biết nó tồn tại.

**Phải hiểu ở lần đọc này:**

- Vòng đời mặc định: client certificate 1 năm, CA 10 năm; hai field `certificateValidityPeriod`
  và `caCertificateValidityPeriod` đổi mặc định này, và `kubeadm upgrade` **tự gia hạn toàn bộ
  certificate** khi nâng cấp control plane (tắt bằng `--certificate-renewal=false`).
- Đọc được output `kubeadm certs check-expiration`: vì sao `kubelet.conf` không nằm trong danh
  sách (kubelet được cấu hình tự xoay certificate trong `/var/lib/kubelet/pki`), và cột
  EXTERNALLY MANAGED nghĩa là kubeadm không quản lý certificate đó.
- Gia hạn thủ công bằng `kubeadm certs renew all` — phải chạy trên **mọi** node control plane
  nếu control plane có nhiều bản sao; vì sao sau đó phải tự khởi động lại các static Pod và
  cách làm (di chuyển tạm file manifest ra khỏi `/etc/kubernetes/manifests`); cập nhật lại
  `$HOME/.kube/config` sau khi `admin.conf` được gia hạn.
- kubeadm **không ghi đè** certificate và key đã tồn tại trong `/etc/kubernetes/pki` trước
  `kubeadm init` (cách dùng CA riêng), và chế độ External CA (chỉ có `ca.crt`, không có
  `ca.key`) khác chế độ thường ở chỗ nào.
- Phân biệt `super-admin.conf` (nhóm `system:masters` — vượt qua tầng phân quyền) với
  `admin.conf` (nhóm `kubeadm:cluster-admins` — bind vào ClusterRole `cluster-admin`), và lệnh
  `kubeadm kubeconfig user` để cấp kubeconfig có thời hạn cho người dùng mới thay vì chia sẻ
  file admin.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Gia hạn certificate với Kubernetes certificates API* (signer tích hợp, cert-manager) | chủ đề nâng cao cho tổ chức cần tích hợp hạ tầng certificate riêng vào cluster | giai đoạn 18 bài 3 — *Manage TLS Certificates in a Cluster* trình bày CSR API đầy đủ |
| Toàn bộ quy trình external CA: `kubeadm certs generate-csr`, ký bằng script `openssl`, nhúng vào kubeconfig | chỉ áp dụng khi cluster dùng CA bên ngoài — cluster lab của bạn không dùng | đọc lại cùng bài [191](191-certificates-manual-vi.md) khi phải vận hành external CA thật |
| *Bật kubelet serving certificate có chữ ký* (`serverTLSBootstrap`) | chỉ cần khi dịch vụ ngoài như metrics-server phải xác thực TLS tới kubelet | giai đoạn 11 — khi cài metrics-server ở Lab 11a |
| *Xoay certificate authority (CA)* | kubeadm không hỗ trợ sẵn; trang chỉ trỏ sang tài liệu khác | bài *Manual Rotation of CA Certificates* trên kubernetes.io, khi CA sắp hết hạn thật |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.15 [stable]`

Các client certificate do [kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/)
sinh ra hết hạn sau 1 năm. Trang này giải thích cách quản lý việc gia hạn certificate với
kubeadm. Nó cũng đề cập đến các tác vụ khác liên quan đến quản lý certificate của kubeadm.

Dự án Kubernetes khuyến nghị nâng cấp kịp thời lên các bản phát hành vá lỗi (patch release)
mới nhất, đồng thời đảm bảo rằng bạn đang chạy một bản phát hành minor còn được hỗ trợ của
Kubernetes. Tuân theo khuyến nghị này giúp bạn duy trì sự an toàn.

## Trước khi bạn bắt đầu (Before you begin)

Bạn nên nắm được [các certificate PKI và yêu cầu trong Kubernetes](https://kubernetes.io/docs/setup/best-practices/certificates/).

Bạn nên biết cách truyền một file [cấu hình](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)
cho các lệnh kubeadm.

Hướng dẫn này có dùng lệnh `openssl` (cho việc ký certificate thủ công, nếu bạn chọn cách đó),
nhưng bạn có thể dùng công cụ mà bạn ưa thích.

Một số bước ở đây dùng `sudo` để có quyền quản trị. Bạn có thể dùng bất kỳ công cụ tương đương nào.

## Sử dụng certificate tùy chỉnh (Using custom certificates) {#custom-certificates}

Theo mặc định, kubeadm sinh ra tất cả các certificate cần thiết để một cluster hoạt động.
Bạn có thể ghi đè hành vi này bằng cách cung cấp certificate của riêng bạn.

Để làm vậy, bạn phải đặt chúng vào thư mục được chỉ định bởi flag `--cert-dir` hoặc field
`certificatesDir` trong `ClusterConfiguration` của kubeadm. Mặc định đây là `/etc/kubernetes/pki`.

Nếu một cặp certificate và private key đã tồn tại trước khi chạy `kubeadm init`, kubeadm sẽ
không ghi đè chúng. Điều này có nghĩa là, ví dụ, bạn có thể sao chép một CA sẵn có vào
`/etc/kubernetes/pki/ca.crt` và `/etc/kubernetes/pki/ca.key`, và kubeadm sẽ dùng CA này để ký
các certificate còn lại.

## Chọn thuật toán mã hóa (Choosing an encryption algorithm) {#choosing-encryption-algorithm}

kubeadm cho phép bạn chọn thuật toán mã hóa dùng để tạo các cặp khóa công khai và khóa riêng
(public/private key). Việc đó được thực hiện qua field `encryptionAlgorithm` của cấu hình
kubeadm:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
encryptionAlgorithm: <ALGORITHM>
```

`<ALGORITHM>` có thể là một trong `RSA-2048` (mặc định), `RSA-3072`, `RSA-4096` hoặc `ECDSA-P256`.

## Chọn thời hạn hiệu lực của certificate (Choosing certificate validity period) {#choosing-cert-validity-period}

kubeadm cho phép bạn chọn thời hạn hiệu lực của CA certificate và các leaf certificate.
Việc đó được thực hiện qua hai field `certificateValidityPeriod` và `caCertificateValidityPeriod`
của cấu hình kubeadm:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
certificateValidityPeriod: 8760h # Mặc định: 365 ngày × 24 giờ = 1 năm
caCertificateValidityPeriod: 87600h # Mặc định: 365 ngày × 24 giờ × 10 = 10 năm
```

Giá trị của các field này tuân theo định dạng được chấp nhận cho
[giá trị `time.Duration` của Go](https://pkg.go.dev/time#ParseDuration), với đơn vị lớn nhất
được hỗ trợ là `h` (giờ).

## Chế độ CA bên ngoài (External CA mode) {#external-ca-mode}

Cũng có thể chỉ cung cấp file `ca.crt` mà không cung cấp file `ca.key` (điều này chỉ áp dụng
cho file CA gốc, không áp dụng cho các cặp certificate khác). Nếu tất cả certificate và
kubeconfig khác đã ở đúng chỗ, kubeadm nhận ra điều kiện này và kích hoạt chế độ "External CA".
kubeadm sẽ tiếp tục mà không có CA key trên đĩa.

Thay vào đó, hãy chạy controller-manager độc lập với `--controllers=csrsigner` và trỏ tới
certificate và key của CA.

Có nhiều cách khác nhau để chuẩn bị credential cho các thành phần khi dùng chế độ external CA.

### Chuẩn bị thủ công credential cho các thành phần (Manual preparation of component credentials)

[Các certificate PKI và yêu cầu](https://kubernetes.io/docs/setup/best-practices/certificates/)
có thông tin về cách chuẩn bị thủ công tất cả credential của các thành phần mà kubeadm yêu cầu.

Hướng dẫn này có dùng lệnh `openssl` (cho việc ký certificate thủ công, nếu bạn chọn cách đó),
nhưng bạn có thể dùng công cụ mà bạn ưa thích.

### Chuẩn bị credential bằng cách ký các CSR do kubeadm sinh ra (Preparation of credentials by signing CSRs generated by kubeadm)

kubeadm có thể [sinh ra các file CSR](#signing-csr) mà bạn ký thủ công bằng các công cụ như
`openssl` cùng external CA của bạn. Các file CSR này sẽ bao gồm toàn bộ đặc tả cho credential
mà các thành phần do kubeadm triển khai cần đến.

### Tự động chuẩn bị credential cho các thành phần bằng kubeadm phase (Automated preparation of component credentials by using kubeadm phases)

Ngoài ra, có thể dùng các lệnh kubeadm phase để tự động hóa quá trình này.

- Đi tới host mà bạn muốn chuẩn bị làm node control plane kubeadm với external CA.
- Sao chép các file external CA `ca.crt` và `ca.key` mà bạn có vào `/etc/kubernetes/pki`
  trên node.
- Chuẩn bị một [file cấu hình kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file)
  tạm thời tên là `config.yaml`, có thể dùng với `kubeadm init`. Hãy đảm bảo file này chứa mọi
  thông tin liên quan ở phạm vi cluster hoặc riêng của host có thể xuất hiện trong certificate,
  chẳng hạn `ClusterConfiguration.controlPlaneEndpoint`, `ClusterConfiguration.certSANs` và
  `InitConfiguration.APIEndpoint`.
- Trên cùng host đó, thực thi các lệnh `kubeadm init phase kubeconfig all --config config.yaml`
  và `kubeadm init phase certs all --config config.yaml`. Việc này sẽ sinh ra tất cả các file
  kubeconfig và certificate cần thiết trong `/etc/kubernetes/` và thư mục con `pki` của nó.
- Kiểm tra các file được sinh ra. Xóa `/etc/kubernetes/pki/ca.key`, xóa hoặc chuyển tới nơi an
  toàn file `/etc/kubernetes/super-admin.conf`.
- Trên các node sẽ gọi `kubeadm join`, cũng xóa `/etc/kubernetes/kubelet.conf`. File này chỉ
  cần trên node đầu tiên nơi `kubeadm init` sẽ được gọi.
- Lưu ý rằng một số file như `pki/sa.*`, `pki/front-proxy-ca.*` và `pki/etc/ca.*` được chia sẻ
  giữa các node control plane. Bạn có thể sinh chúng một lần và
  [phân phối thủ công](08-high-availability-vi.md#manual-certs) tới các node sẽ gọi
  `kubeadm join`, hoặc dùng tính năng
  [`--upload-certs`](08-high-availability-vi.md) của `kubeadm init` và `--certificate-key` của
  `kubeadm join` để tự động hóa việc phân phối này.

Khi credential đã được chuẩn bị trên tất cả các node, hãy gọi `kubeadm init` và `kubeadm join`
để các node này gia nhập cluster. kubeadm sẽ dùng các file kubeconfig và certificate sẵn có
trong `/etc/kubernetes/` và thư mục con `pki` của nó.

## Hạn hiệu lực và quản lý certificate (Certificate expiry and management) {#check-certificate-expiration}

> **Ghi chú:** `kubeadm` không thể quản lý các certificate do một CA bên ngoài ký.

Bạn có thể dùng lệnh con `check-expiration` để kiểm tra khi nào các certificate hết hạn:

```shell
kubeadm certs check-expiration
```

Output tương tự như sau:

```console
CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Dec 30, 2020 23:36 UTC   364d                                    no
apiserver                  Dec 30, 2020 23:36 UTC   364d            ca                      no
apiserver-etcd-client      Dec 30, 2020 23:36 UTC   364d            etcd-ca                 no
apiserver-kubelet-client   Dec 30, 2020 23:36 UTC   364d            ca                      no
controller-manager.conf    Dec 30, 2020 23:36 UTC   364d                                    no
etcd-healthcheck-client    Dec 30, 2020 23:36 UTC   364d            etcd-ca                 no
etcd-peer                  Dec 30, 2020 23:36 UTC   364d            etcd-ca                 no
etcd-server                Dec 30, 2020 23:36 UTC   364d            etcd-ca                 no
front-proxy-client         Dec 30, 2020 23:36 UTC   364d            front-proxy-ca          no
scheduler.conf             Dec 30, 2020 23:36 UTC   364d                                    no

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Dec 28, 2029 23:36 UTC   9y              no
etcd-ca                 Dec 28, 2029 23:36 UTC   9y              no
front-proxy-ca          Dec 28, 2029 23:36 UTC   9y              no
```

Lệnh này hiển thị thời điểm hết hạn/thời gian còn lại cho các client certificate trong thư mục
`/etc/kubernetes/pki` và cho client certificate được nhúng trong các file kubeconfig mà kubeadm
sử dụng (`admin.conf`, `controller-manager.conf` và `scheduler.conf`).

Ngoài ra, kubeadm thông báo cho người dùng nếu certificate được quản lý bên ngoài (externally
managed); trong trường hợp đó, người dùng phải tự lo việc gia hạn certificate một cách thủ
công/bằng công cụ khác.

File cấu hình `kubelet.conf` không có trong danh sách trên vì kubeadm cấu hình kubelet
[tự động gia hạn certificate](398-certificate-rotation-vi.md)
với các certificate xoay được (rotatable) trong `/var/lib/kubelet/pki`.
Để sửa một kubelet client certificate đã hết hạn, xem
[Xoay vòng client certificate của kubelet thất bại](09-troubleshooting-kubeadm-vi.md#kubelet-client-cert).

> **Ghi chú:** Trên các node được tạo bằng `kubeadm init` từ các phiên bản kubeadm trước 1.17,
> có một [lỗi](https://github.com/kubernetes/kubeadm/issues/1753) khiến bạn phải sửa thủ công
> nội dung của `kubelet.conf`. Sau khi `kubeadm init` hoàn tất, bạn nên cập nhật `kubelet.conf`
> để trỏ tới các kubelet client certificate được xoay vòng, bằng cách thay
> `client-certificate-data` và `client-key-data` bằng:
>
> ```yaml
> client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
> client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
> ```

## Gia hạn certificate tự động (Automatic certificate renewal)

kubeadm gia hạn tất cả các certificate trong quá trình
[nâng cấp](221-kubeadm-upgrade-vi.md)
control plane.

Tính năng này được thiết kế cho những trường hợp sử dụng đơn giản nhất; nếu bạn không có yêu
cầu đặc biệt về việc gia hạn certificate và thực hiện nâng cấp phiên bản Kubernetes đều đặn
(cách nhau chưa tới 1 năm giữa mỗi lần nâng cấp), kubeadm sẽ lo việc giữ cho cluster của bạn
luôn cập nhật và an toàn ở mức hợp lý.

Nếu bạn có các yêu cầu phức tạp hơn về gia hạn certificate, bạn có thể từ chối hành vi mặc
định bằng cách truyền `--certificate-renewal=false` cho `kubeadm upgrade apply` hoặc
`kubeadm upgrade node`.

## Gia hạn certificate thủ công (Manual certificate renewal) {#manual-certificate-renewal}

Bạn có thể gia hạn certificate thủ công bất cứ lúc nào với lệnh `kubeadm certs renew`, kèm các
tùy chọn dòng lệnh phù hợp. Nếu bạn đang chạy cluster với control plane có nhiều bản sao
(replicated), lệnh này cần được thực thi trên tất cả các node control plane.

Lệnh này thực hiện việc gia hạn bằng certificate và key của CA (hoặc front-proxy-CA) được lưu
trong `/etc/kubernetes/pki`.

`kubeadm certs renew` dùng các certificate hiện có làm nguồn có thẩm quyền cho các thuộc tính
(Common Name, Organization, subject alternative name) và không dựa vào ConfigMap
`kubeadm-config`. Dù vậy, dự án Kubernetes khuyến nghị giữ cho certificate đang phục vụ và các
giá trị tương ứng trong ConfigMap đó đồng bộ với nhau, để tránh mọi nguy cơ nhầm lẫn.

Sau khi chạy lệnh, bạn nên khởi động lại các Pod của control plane. Điều này là bắt buộc vì
hiện tại việc nạp lại certificate động (dynamic certificate reload) chưa được hỗ trợ cho mọi
thành phần và certificate.
[Static Pod](293-static-pod-tasks-vi.md) được quản
lý bởi kubelet cục bộ chứ không phải bởi API Server, do đó không thể dùng kubectl để xóa và
khởi động lại chúng.
Để khởi động lại một static Pod, bạn có thể tạm thời di chuyển file manifest của nó ra khỏi
`/etc/kubernetes/manifests/` và chờ 20 giây (xem giá trị `fileCheckFrequency` trong
[struct KubeletConfiguration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)).
kubelet sẽ chấm dứt Pod nếu nó không còn trong thư mục manifest.
Sau đó bạn có thể chuyển file trở lại, và sau một chu kỳ `fileCheckFrequency` nữa, kubelet sẽ
tạo lại Pod và việc gia hạn certificate cho thành phần đó có thể hoàn tất.

`kubeadm certs renew` có thể gia hạn bất kỳ certificate cụ thể nào, hoặc với lệnh con `all`,
nó gia hạn tất cả:

```shell
# Nếu bạn đang chạy cluster với control plane có nhiều bản sao, lệnh này
# cần được thực thi trên tất cả các node control plane.
kubeadm certs renew all
```

### Sao chép certificate của quản trị viên (tùy chọn) (Copying the administrator certificate (optional)) {#admin-certificate-copy}

Các cluster dựng bằng kubeadm thường sao chép certificate trong `admin.conf` vào
`$HOME/.kube/config`, như hướng dẫn trong
[Tạo cluster với kubeadm](02-create-cluster-kubeadm-vi.md).
Trên một hệ thống như vậy, để cập nhật nội dung của `$HOME/.kube/config` sau khi gia hạn
`admin.conf`, bạn có thể chạy các lệnh sau:

```shell
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Gia hạn certificate với Kubernetes certificates API (Renew certificates with the Kubernetes certificates API)

Mục này cung cấp thêm chi tiết về cách thực hiện gia hạn certificate thủ công bằng Kubernetes
certificates API.

> **Thận trọng:** Đây là các chủ đề nâng cao dành cho người dùng cần tích hợp hạ tầng
> certificate của tổ chức mình vào một cluster dựng bằng kubeadm. Nếu cấu hình kubeadm mặc
> định đã đáp ứng nhu cầu của bạn, bạn nên để kubeadm quản lý certificate.

### Thiết lập một signer (Set up a signer) {#set-up-a-signer}

Kubernetes Certificate Authority không hoạt động ngay khi cài đặt (out of the box).
Bạn có thể cấu hình một signer bên ngoài như [cert-manager](https://cert-manager.io/docs/configuration/ca/),
hoặc dùng signer tích hợp sẵn.

Signer tích hợp sẵn là một phần của
[`kube-controller-manager`](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/).

Để kích hoạt signer tích hợp sẵn, bạn phải truyền hai flag `--cluster-signing-cert-file` và
`--cluster-signing-key-file`.

Nếu bạn đang tạo một cluster mới, bạn có thể dùng một
[file cấu hình](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/) kubeadm:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
controllerManager:
  extraArgs:
  - name: "cluster-signing-cert-file"
    value: "/etc/kubernetes/pki/ca.crt"
  - name: "cluster-signing-key-file"
    value: "/etc/kubernetes/pki/ca.key"
```

### Tạo certificate signing request (CSR) (Create certificate signing requests (CSR))

Xem [Tạo CertificateSigningRequest](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#create-certificatessigningrequest)
để tạo CSR với Kubernetes API.

## Gia hạn certificate với CA bên ngoài (Renew certificates with external CA)

Mục này cung cấp thêm chi tiết về cách thực hiện gia hạn certificate thủ công bằng một CA
bên ngoài.

Để tích hợp tốt hơn với các CA bên ngoài, kubeadm cũng có thể tạo ra các certificate signing
request (CSR). Một CSR đại diện cho một yêu cầu gửi tới CA để xin một certificate được ký cho
một client. Theo cách gọi của kubeadm, bất kỳ certificate nào thông thường được ký bởi một CA
trên đĩa đều có thể được tạo ra dưới dạng CSR thay thế. Tuy nhiên, một CA thì không thể được
tạo dưới dạng CSR.

### Gia hạn bằng certificate signing request (CSR) (Renewal by using certificate signing requests (CSR))

Việc gia hạn certificate có thể thực hiện bằng cách sinh các CSR mới và ký chúng với CA bên
ngoài. Để biết thêm chi tiết về cách làm việc với các CSR do kubeadm sinh ra, xem mục
[Ký các certificate signing request (CSR) do kubeadm sinh ra](#signing-csr).

## Xoay certificate authority (CA) (Certificate authority (CA) rotation) {#certificate-authority-rotation}

kubeadm không hỗ trợ sẵn việc xoay vòng (rotation) hay thay thế các CA certificate.

Để biết thêm về việc xoay hoặc thay thế CA thủ công, xem
[xoay thủ công các CA certificate](400-manual-rotation-of-ca-certificates-vi.md).

## Bật kubelet serving certificate có chữ ký (Enabling signed kubelet serving certificates) {#kubelet-serving-certs}

Theo mặc định, kubelet serving certificate do kubeadm triển khai là tự ký (self-signed).
Điều này có nghĩa là kết nối từ các dịch vụ bên ngoài như
[metrics-server](https://github.com/kubernetes-sigs/metrics-server) tới một kubelet không thể
được bảo vệ bằng TLS.

Để cấu hình các kubelet trong một cluster kubeadm mới nhận serving certificate được ký đúng
cách, bạn phải truyền cấu hình tối thiểu sau cho `kubeadm init`:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
serverTLSBootstrap: true
```

Nếu bạn đã tạo cluster từ trước, bạn phải điều chỉnh nó bằng cách làm như sau:

- Tìm và sửa ConfigMap `kubelet-config` trong namespace `kube-system`.
  Trong ConfigMap đó, key `kubelet` có giá trị là một document
  [KubeletConfiguration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).
  Sửa document KubeletConfiguration để đặt `serverTLSBootstrap: true`.
- Trên mỗi node, thêm field `serverTLSBootstrap: true` vào `/var/lib/kubelet/config.yaml`
  và khởi động lại kubelet với `systemctl restart kubelet`

Field `serverTLSBootstrap: true` sẽ bật việc bootstrap các kubelet serving certificate bằng
cách yêu cầu chúng từ API `certificates.k8s.io`. Một hạn chế đã biết là các CSR (Certificate
Signing Request) cho những certificate này không thể được tự động phê duyệt (approve) bởi
signer mặc định trong kube-controller-manager —
[`kubernetes.io/kubelet-serving`](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#kubernetes-signers).
Việc này đòi hỏi hành động từ người dùng hoặc một controller của bên thứ ba.

Các CSR này có thể được xem bằng:

```shell
kubectl get csr
```
```console
NAME        AGE     SIGNERNAME                        REQUESTOR                      CONDITION
csr-9wvgt   112s    kubernetes.io/kubelet-serving     system:node:worker-1           Pending
csr-lz97v   1m58s   kubernetes.io/kubelet-serving     system:node:control-plane-1    Pending
```

Để phê duyệt chúng, bạn có thể làm như sau:

```shell
kubectl certificate approve <CSR-name>
```

Theo mặc định, các serving certificate này sẽ hết hạn sau một năm. kubeadm đặt field
`rotateCertificates` của `KubeletConfiguration` là `true`, nghĩa là gần thời điểm hết hạn, một
bộ CSR mới cho các serving certificate sẽ được tạo ra và phải được phê duyệt để hoàn tất việc
xoay vòng. Để hiểu thêm, xem
[Xoay vòng certificate](https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/#certificate-rotation).

Nếu bạn đang tìm một giải pháp tự động phê duyệt các CSR này, khuyến nghị là bạn liên hệ nhà
cung cấp cloud của mình và hỏi xem họ có một CSR signer nào xác minh danh tính node bằng một
cơ chế ngoài băng (out of band) hay không.

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần.
> Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án đó. Xem
> [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content)
> để biết thêm chi tiết.

Có thể dùng các controller tùy chỉnh của bên thứ ba:

- [kubelet-csr-approver](https://github.com/postfinance/kubelet-csr-approver)

Một controller như vậy không phải là một cơ chế an toàn trừ khi nó không chỉ xác minh
CommonName trong CSR mà còn xác minh cả các IP và tên miền được yêu cầu. Điều này sẽ ngăn một
tác nhân độc hại có quyền truy cập vào một kubelet client certificate tạo ra các CSR yêu cầu
serving certificate cho bất kỳ IP hay tên miền nào.

## Sinh file kubeconfig cho người dùng bổ sung (Generating kubeconfig files for additional users) {#kubeconfig-additional-users}

Trong quá trình tạo cluster, `kubeadm init` ký certificate trong `super-admin.conf` với
`Subject: O = system:masters, CN = kubernetes-super-admin`.
[`system:masters`](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles)
là một nhóm siêu người dùng dạng khẩn cấp (break-glass), vượt qua tầng phân quyền
(authorization layer, ví dụ [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)).
File `admin.conf` cũng được kubeadm tạo trên các node control plane và chứa một certificate với
`Subject: O = kubeadm:cluster-admins, CN = kubernetes-admin`. `kubeadm:cluster-admins` là một
nhóm thuộc về kubeadm về mặt logic. Nếu cluster của bạn dùng RBAC (mặc định của kubeadm), nhóm
`kubeadm:cluster-admins` được bind vào ClusterRole
[`cluster-admin`](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles).

> **Cảnh báo:** Tránh chia sẻ các file `super-admin.conf` hoặc `admin.conf`. Thay vào đó, hãy
> tạo quyền truy cập với đặc quyền tối thiểu (least privileged) ngay cả cho những người làm
> công việc quản trị viên, và dùng phương án đặc quyền tối thiểu đó cho mọi việc, ngoại trừ
> truy cập khẩn cấp (break-glass).

Bạn có thể dùng lệnh
[`kubeadm kubeconfig user`](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-kubeconfig)
để sinh các file kubeconfig cho người dùng bổ sung.
Lệnh này chấp nhận kết hợp các flag dòng lệnh và các tùy chọn
[cấu hình kubeadm](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/).
Kubeconfig được sinh ra sẽ được ghi ra stdout và có thể được chuyển vào một file bằng
`kubeadm kubeconfig user ... > somefile.conf`.

Ví dụ file cấu hình có thể dùng với `--config`:

```yaml
# example.yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
# Sẽ được dùng làm "cluster" đích trong kubeconfig
clusterName: "kubernetes"
# Sẽ được dùng làm "server" (IP hoặc tên DNS) của cluster này trong kubeconfig
controlPlaneEndpoint: "some-dns-address:6443"
# Key và certificate của CA cluster sẽ được nạp từ thư mục cục bộ này
certificatesDir: "/etc/kubernetes/pki"
```

Hãy đảm bảo rằng các thiết lập này khớp với thiết lập của cluster đích mong muốn.
Để xem thiết lập của một cluster hiện có, dùng:

```shell
kubectl get cm kubeadm-config -n kube-system -o=jsonpath="{.data.ClusterConfiguration}"
```

Ví dụ sau sẽ sinh một file kubeconfig với credential có hiệu lực 24 giờ cho một người dùng mới
`johndoe` thuộc nhóm `appdevs`:

```shell
kubeadm kubeconfig user --config example.yaml --org appdevs --client-name johndoe --validity-period 24h
```

Ví dụ sau sẽ sinh một file kubeconfig với credential quản trị viên có hiệu lực 1 tuần:

```shell
kubeadm kubeconfig user --config example.yaml --client-name admin --validity-period 168h
```

## Ký các certificate signing request (CSR) do kubeadm sinh ra (Signing certificate signing requests (CSR) generated by kubeadm) {#signing-csr}

Bạn có thể tạo các certificate signing request với `kubeadm certs generate-csr`.
Gọi lệnh này sẽ sinh ra các cặp file `.csr` / `.key` cho các certificate thông thường. Với các
certificate được nhúng trong file kubeconfig, lệnh sẽ sinh ra một cặp `.csr` / `.conf`, trong
đó key đã được nhúng sẵn trong file `.conf`.

Một file CSR chứa mọi thông tin cần thiết để một CA ký một certificate.
kubeadm dùng một
[đặc tả được định nghĩa rõ ràng](https://kubernetes.io/docs/setup/best-practices/certificates/#all-certificates)
cho tất cả certificate và CSR của nó.

Thư mục certificate mặc định là `/etc/kubernetes/pki`, còn thư mục mặc định cho các file
kubeconfig là `/etc/kubernetes`. Các giá trị mặc định này có thể được ghi đè lần lượt bằng các
flag `--cert-dir` và `--kubeconfig-dir`.

Để truyền các tùy chọn tùy chỉnh cho `kubeadm certs generate-csr`, dùng flag `--config`, flag
này chấp nhận một file [cấu hình kubeadm](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/),
tương tự các lệnh như `kubeadm init`. Mọi đặc tả như SAN bổ sung và địa chỉ IP tùy chỉnh phải
được lưu trong cùng một file cấu hình và dùng cho tất cả các lệnh kubeadm liên quan bằng cách
truyền nó qua `--config`.

> **Ghi chú:** Hướng dẫn này dùng thư mục Kubernetes mặc định `/etc/kubernetes`, vốn yêu cầu
> quyền siêu người dùng (super user). Nếu bạn làm theo hướng dẫn này và dùng các thư mục mà bạn
> có quyền ghi (thường có nghĩa là chạy `kubeadm` với `--cert-dir` và `--kubeconfig-dir`) thì
> bạn có thể bỏ lệnh `sudo`.
>
> Sau đó bạn phải sao chép các file đã tạo vào trong thư mục `/etc/kubernetes` để
> `kubeadm init` hoặc `kubeadm join` tìm thấy chúng.

### Chuẩn bị các file CA và service account (Preparing CA and service account files)

Trên node control plane chính (primary), nơi `kubeadm init` sẽ được thực thi, gọi các lệnh sau:

```shell
sudo kubeadm init phase certs ca
sudo kubeadm init phase certs etcd-ca
sudo kubeadm init phase certs front-proxy-ca
sudo kubeadm init phase certs sa
```

Việc này sẽ điền vào hai thư mục `/etc/kubernetes/pki` và `/etc/kubernetes/pki/etcd` tất cả
các file CA tự ký (certificate và key) và service account (khóa công khai và khóa riêng) mà
kubeadm cần cho một node control plane.

> **Ghi chú:** Nếu bạn dùng một external CA, bạn phải sinh chính các file này bằng phương thức
> bên ngoài (out of band) và sao chép thủ công chúng vào node control plane chính tại
> `/etc/kubernetes`.
>
> Khi tất cả CSR đã được ký, bạn có thể xóa key của CA gốc (`ca.key`) như đã lưu ý trong mục
> [Chế độ External CA](#external-ca-mode).

Với các node control plane phụ (secondary, `kubeadm join --control-plane`), không cần gọi các
lệnh trên. Tùy vào cách bạn thiết lập cluster
[tính sẵn sàng cao (High Availability)](08-high-availability-vi.md), bạn hoặc phải sao chép
thủ công chính các file đó từ node control plane chính, hoặc dùng tính năng tự động
`--upload-certs` của `kubeadm init`.

### Sinh các CSR (Generate CSRs)

Lệnh `kubeadm certs generate-csr` sinh CSR cho tất cả các certificate đã biết mà kubeadm quản
lý. Khi lệnh chạy xong, bạn phải xóa thủ công các file `.csr`, `.conf` hoặc `.key` mà bạn
không cần.

#### Lưu ý đối với kubelet.conf (Considerations for kubelet.conf) {#considerations-kubelet-conf}

Mục này áp dụng cho cả node control plane lẫn worker node.

Nếu bạn đã xóa file `ca.key` khỏi các node control plane
([chế độ External CA](#external-ca-mode)), kube-controller-manager đang hoạt động trong
cluster này sẽ không thể ký các kubelet client certificate. Nếu trong thiết lập của bạn không
tồn tại phương thức bên ngoài nào để ký các certificate này (chẳng hạn một
[signer bên ngoài](#set-up-a-signer)), bạn có thể ký thủ công `kubelet.conf.csr` như được
giải thích trong hướng dẫn này.

Lưu ý rằng điều này cũng có nghĩa là việc
[tự động xoay vòng kubelet client certificate](398-certificate-rotation-vi.md#enabling-client-certificate-rotation)
sẽ bị vô hiệu hóa. Trong trường hợp đó, gần thời điểm certificate hết hạn, bạn phải sinh một
`kubelet.conf.csr` mới, ký certificate, nhúng nó vào `kubelet.conf` và khởi động lại kubelet.

Nếu điều này không áp dụng với thiết lập của bạn, bạn có thể bỏ qua việc xử lý
`kubelet.conf.csr` trên các node control plane phụ và trên các worker node (tất cả các node
gọi `kubeadm join ...`). Lý do là kube-controller-manager đang hoạt động sẽ chịu trách nhiệm
ký các kubelet client certificate mới.

> **Ghi chú:** Bạn phải xử lý file `kubelet.conf.csr` trên node control plane chính (host nơi
> bạn chạy `kubeadm init` ban đầu). Đó là vì `kubeadm` coi node này là node bootstrap cluster,
> và cần một `kubelet.conf` được điền sẵn.

#### Các node control plane (Control plane nodes)

Thực thi lệnh sau trên các node control plane chính (`kubeadm init`) và phụ
(`kubeadm join --control-plane`) để sinh tất cả các file CSR:

```shell
sudo kubeadm certs generate-csr
```

Nếu dùng etcd bên ngoài (external etcd), hãy làm theo hướng dẫn
[External etcd với kubeadm](08-high-availability-vi.md) để hiểu những file CSR nào cần trên
các node kubeadm và node etcd. Các file `.csr` và `.key` khác trong
`/etc/kubernetes/pki/etcd` có thể được xóa.

Dựa trên phần giải thích trong
[Lưu ý đối với kubelet.conf](#considerations-kubelet-conf), giữ hoặc xóa hai file
`kubelet.conf` và `kubelet.conf.csr`.

#### Các worker node (Worker nodes)

Dựa trên phần giải thích trong
[Lưu ý đối với kubelet.conf](#considerations-kubelet-conf), tùy chọn gọi:

```shell
sudo kubeadm certs generate-csr
```

và chỉ giữ lại hai file `kubelet.conf` và `kubelet.conf.csr`. Hoặc bỏ qua hoàn toàn các bước
cho worker node.

### Ký CSR cho tất cả certificate (Signing CSRs for all certificates)

> **Ghi chú:** Nếu bạn dùng external CA và đã có các file số serial của CA (`.srl`) cho
> `openssl`, bạn có thể sao chép các file đó tới node kubeadm nơi các CSR sẽ được xử lý.
> Các file `.srl` cần sao chép là `/etc/kubernetes/pki/ca.srl`,
> `/etc/kubernetes/pki/front-proxy-ca.srl` và `/etc/kubernetes/pki/etcd/ca.srl`.
> Sau đó các file này có thể được chuyển sang node mới, nơi các file CSR sẽ được xử lý.
>
> Nếu một file `.srl` bị thiếu cho một CA trên node, script bên dưới sẽ sinh một file SRL mới
> với số serial khởi đầu ngẫu nhiên.
>
> Để đọc thêm về các file `.srl`, xem tài liệu
> [`openssl`](https://www.openssl.org/docs/man3.0/man1/openssl-x509.html) cho flag `--CAserial`.

Lặp lại bước này trên tất cả các node có file CSR.

Ghi script sau vào thư mục `/etc/kubernetes`, di chuyển vào thư mục đó và thực thi script.
Script sẽ sinh certificate cho tất cả các file CSR hiện diện trong cây thư mục
`/etc/kubernetes`.

```bash
#!/bin/bash

# Đặt thời hạn hết hạn của certificate theo số ngày
DAYS=365

# Xử lý tất cả các file CSR, trừ các file cho front-proxy và etcd
find ./ -name "*.csr" | grep -v "pki/etcd" | grep -v "front-proxy" | while read -r FILE;
do
    echo "* Processing ${FILE} ..."
    FILE=${FILE%.*} # Cắt bỏ phần mở rộng
    if [ -f "./pki/ca.srl" ]; then
        SERIAL_FLAG="-CAserial ./pki/ca.srl"
    else
        SERIAL_FLAG="-CAcreateserial"
    fi
    openssl x509 -req -days "${DAYS}" -CA ./pki/ca.crt -CAkey ./pki/ca.key ${SERIAL_FLAG} \
        -in "${FILE}.csr" -out "${FILE}.crt"
    sleep 2
done

# Xử lý tất cả các CSR của etcd
find ./pki/etcd -name "*.csr" | while read -r FILE;
do
    echo "* Processing ${FILE} ..."
    FILE=${FILE%.*} # Cắt bỏ phần mở rộng
    if [ -f "./pki/etcd/ca.srl" ]; then
        SERIAL_FLAG=-CAserial ./pki/etcd/ca.srl
    else
        SERIAL_FLAG=-CAcreateserial
    fi
    openssl x509 -req -days "${DAYS}" -CA ./pki/etcd/ca.crt -CAkey ./pki/etcd/ca.key ${SERIAL_FLAG} \
        -in "${FILE}.csr" -out "${FILE}.crt"
done

# Xử lý các CSR của front-proxy
echo "* Processing ./pki/front-proxy-client.csr ..."
openssl x509 -req -days "${DAYS}" -CA ./pki/front-proxy-ca.crt -CAkey ./pki/front-proxy-ca.key -CAcreateserial \
    -in ./pki/front-proxy-client.csr -out ./pki/front-proxy-client.crt
```

### Nhúng certificate vào các file kubeconfig (Embedding certificates in kubeconfig files)

Lặp lại bước này trên tất cả các node có file CSR.

Ghi script sau vào thư mục `/etc/kubernetes`, di chuyển vào thư mục đó và thực thi script.
Script sẽ lấy các file `.crt` đã được ký cho các file kubeconfig từ các CSR ở bước trước và
nhúng chúng vào các file kubeconfig.

```bash
#!/bin/bash

CLUSTER=kubernetes
find ./ -name "*.conf" | while read -r FILE;
do
    echo "* Processing ${FILE} ..."
    KUBECONFIG="${FILE}" kubectl config set-cluster "${CLUSTER}" --certificate-authority ./pki/ca.crt --embed-certs
    USER=$(KUBECONFIG="${FILE}" kubectl config view -o jsonpath='{.users[0].name}')
    KUBECONFIG="${FILE}" kubectl config set-credentials "${USER}" --client-certificate "${FILE}.crt" --embed-certs
done
```

### Dọn dẹp (Performing cleanup) {#post-csr-cleanup}

Thực hiện bước này trên tất cả các node có file CSR.

Ghi script sau vào thư mục `/etc/kubernetes`, di chuyển vào thư mục đó và thực thi script.

```bash
#!/bin/bash

# Dọn dẹp các file CSR
rm -f ./*.csr ./pki/*.csr ./pki/etcd/*.csr # Xóa tất cả các file CSR

# Dọn dẹp các file CRT đã được nhúng vào các file kubeconfig
rm -f ./*.crt
```

Tùy chọn, chuyển các file `.srl` sang node kế tiếp cần xử lý.

Tùy chọn, nếu dùng external CA, xóa file `/etc/kubernetes/pki/ca.key`, như đã giải thích
trong mục [Chế độ External CA](#external-ca-mode).

### Khởi tạo node bằng kubeadm (kubeadm node initialization)

Khi các file CSR đã được ký và các certificate cần thiết đã ở đúng chỗ trên các host mà bạn
muốn dùng làm node, bạn có thể dùng các lệnh `kubeadm init` và `kubeadm join` để tạo một
cluster Kubernetes từ các node này. Trong quá trình `init` và `join`, kubeadm dùng các
certificate, khóa mã hóa và file kubeconfig sẵn có mà nó tìm thấy trong cây thư mục
`/etc/kubernetes` trên hệ thống file cục bộ của host.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu dưới đây mà không nhìn lại bài là đủ cho lần đọc ở checkpoint này.

1. Cluster lab của bạn được dựng bằng kubeadm cách đây 11 tháng và chưa từng nâng cấp. Chuyện
   gì sắp xảy ra với các client certificate, và bài nêu những cách xử lý nào? Nếu control
   plane có nhiều node thì lệnh gia hạn thủ công phải chạy ở đâu?
2. Sau khi chạy `kubeadm certs renew all` trên `k8s-master`, bạn chạy
   `kubectl delete pod kube-apiserver-k8s-master -n kube-system` để khởi động lại apiserver.
   Cách này có tác dụng không? Nếu không thì phải làm thế nào?
3. Output của `kubeadm certs check-expiration` không liệt kê `kubelet.conf` — vì sao, và các
   client certificate của kubelet thực tế nằm ở đâu?
4. Bạn muốn kubeadm dùng CA của công ty thay vì tự sinh: phải làm gì **trước** khi chạy
   `kubeadm init`? Và điều gì thay đổi nếu bộ phận bảo mật chỉ giao cho bạn `ca.crt` mà giữ
   lại `ca.key`?
5. Một đồng nghiệp xin bạn file `admin.conf` để làm việc hằng ngày trên cluster. Bài khuyến
   nghị gì trong tình huống này, và lệnh nào của kubeadm là phương án thay thế?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Các client certificate do kubeadm sinh ra **hết hạn sau 1 năm**, nên chúng sắp hết hạn. Hai
   cách xử lý trong bài: (a) **nâng cấp control plane** — kubeadm tự gia hạn tất cả
   certificate trong `kubeadm upgrade`; (b) **gia hạn thủ công** bằng `kubeadm certs renew all`.
   Với control plane có nhiều bản sao, lệnh này phải được thực thi **trên tất cả các node
   control plane**, vì nó thao tác trên các file trong `/etc/kubernetes/pki` của từng node.
2. **Không có tác dụng.** kube-apiserver là **static Pod**, được quản lý bởi kubelet cục bộ
   chứ không phải API Server, nên kubectl không thể dùng để xóa và khởi động lại nó. Cách đúng
   theo bài: tạm **di chuyển file manifest ra khỏi `/etc/kubernetes/manifests/`**, chờ khoảng
   20 giây (chu kỳ `fileCheckFrequency`) để kubelet chấm dứt Pod, rồi chuyển file trở lại để
   kubelet tạo lại Pod với certificate mới.
3. Vì kubeadm cấu hình kubelet **tự động gia hạn certificate** với các certificate xoay được;
   chúng nằm trong **`/var/lib/kubelet/pki`** (không phải `/etc/kubernetes/pki`), nên
   `check-expiration` không quản và không liệt kê `kubelet.conf`.
4. Sao chép CA của công ty vào **`/etc/kubernetes/pki/ca.crt` và `/etc/kubernetes/pki/ca.key`
   trước khi chạy `kubeadm init`** — kubeadm không ghi đè cặp certificate/key đã tồn tại và sẽ
   dùng CA đó để ký các certificate còn lại. Nếu chỉ có `ca.crt` mà không có `ca.key`, kubeadm
   kích hoạt **chế độ External CA**: nó tiếp tục mà không có CA key trên đĩa, và mọi việc ký
   certificate phải do bên ngoài đảm nhiệm (ví dụ chạy controller-manager độc lập với
   `--controllers=csrsigner` trỏ tới certificate và key của CA, hoặc ký các CSR do kubeadm
   sinh ra).
5. Bài **cảnh báo rõ: tránh chia sẻ `admin.conf` hoặc `super-admin.conf`** — kể cả với người
   làm công việc quản trị, hãy tạo quyền truy cập đặc quyền tối thiểu và chỉ dành các file kia
   cho truy cập khẩn cấp (break-glass). Phương án thay thế là
   **`kubeadm kubeconfig user`** với `--client-name`, `--org` và `--validity-period` để cấp
   một kubeconfig riêng, có thời hạn, cho người đó. Trực giác "admin.conf chỉ là quyền admin
   bình thường" sai ở chỗ: nhóm trong certificate của nó được bind thẳng vào ClusterRole
   `cluster-admin`, và certificate đã phát hành thì không thu hồi dễ dàng.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
