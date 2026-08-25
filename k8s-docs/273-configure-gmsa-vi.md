# Cấu hình GMSA cho Pod và container Windows (Configure GMSA for Windows Pods and containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 15 — Windows, nếu môi trường có node Windows](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows)
→ dòng **Thực hành**, bài 1/4 · Kiểm chứng ở [Lab 15 — Node Windows](labs/LAB-15-NODE-WINDOWS.md)
phần **B1.4** — và **chỉ** ở đó.

Nói thẳng ngay từ đầu: cluster lab là ba VM Ubuntu `lab-k8s-master`, `lab-k8s-worker1`,
`lab-k8s-worker2` — **không có node Windows**, nên Lab 15 chạy **nhánh B** (đọc-hiểu, có kiểm chứng
ranh giới trên chính cluster Linux). Riêng bài này còn vượt phạm vi **cả nhánh A**: mục *Trước khi
bạn bắt đầu* đòi worker node Windows, và mục *Cấu hình GMSA và node Windows trong Active Directory*
đòi các worker node đó phải được cấu hình **trong một domain Active Directory** để có quyền đọc
mật khẩu của GMSA. Nhánh A chỉ thêm một VM Windows Server làm worker, không dựng domain
controller. Vì vậy Lab 15 chỉ đo **bề mặt API** của GMSA ở B1.4 — CRD và hai webhook có tồn tại
trên cluster hay không — và ghi rõ lý do cho phần còn lại ở
[bảng 1.1](labs/LAB-15-NODE-WINDOWS.md#11-ánh-xạ-tài-liệu-sang-bài-thực-hành). Đọc bài này để nắm
**cơ chế và chuỗi phụ thuộc**, không phải để làm theo.

**Phải hiểu ở lần đọc này:**

- GMSA giải quyết việc gì: đó là một loại tài khoản Active Directory **quản lý mật khẩu tự động**,
  và trong Kubernetes nó cho container Windows nhận **danh tính domain** để làm các chức năng dựa
  trên domain — ví dụ xác thực Kerberos — khi gọi sang dịch vụ Windows khác.
- GMSA **không đi kèm Kubernetes**. Mục *Trước khi bạn bắt đầu* liệt kê hai bước khởi tạo một lần
  cho mỗi cluster: cài CRD `GMSACredentialSpec`, và cài **hai** webhook — một **mutating** mở rộng
  tham chiếu theo tên thành credspec JSON đầy đủ trong Pod spec, một **validating** kiểm service
  account của Pod có được phép dùng GMSA đó không.
- Credspec **không chứa dữ liệu bí mật hay nhạy cảm** (mục *Tạo các resource đặc tả thông tin xác
  thực GMSA*). Nó chỉ mô tả cho Windows biết GMSA nào — tên tài khoản, domain, GUID, SID. Mật khẩu
  thật do node Windows lấy từ Active Directory, nên node phải được cấp quyền trong AD; đó là lý do
  mục *Cấu hình GMSA và node Windows trong Active Directory* tồn tại như một bước riêng.
- Đường phân quyền là RBAC bạn đã học ở [giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy):
  một ClusterRole cấp verb **`use`** trên resource `gmsacredentialspecs` của nhóm API
  `windows.k8s.io`, giới hạn bằng `resourceNames` xuống đúng một credspec; rồi một RoleBinding gắn
  ClusterRole đó cho ServiceAccount mà Pod sẽ chạy dưới danh nghĩa.
- Trường tham chiếu là `securityContext.windowsOptions.gmsaCredentialSpecName`, đặt được ở **hai
  cấp**: ở cấp Pod thì **mọi** container trong Pod dùng GMSA đó; ở cấp container thì chỉ container
  đó. Khi Pod được apply, chuỗi ba sự kiện chạy theo thứ tự: mutating webhook mở rộng tham chiếu →
  validating webhook kiểm quyền `use` → container runtime cấu hình từng container Windows.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Các bước cài thật của mục *Cài đặt CRD GMSACredentialSpec* và *Cài đặt các webhook* — cặp khóa certificate, Secret chứa certificate, Deployment webhook, script `deploy-gmsa-webhook.sh` | không có domain thì cài xong chỉ được một CRD rỗng và một Pod webhook không ai gọi tới; đó là diễn chứ không phải kiểm chứng | không có trong lộ trình. [Lab 15](labs/LAB-15-NODE-WINDOWS.md) phần **B1.4** đo đúng phần đo được: ba thứ đó **có tồn tại** trên cluster hay không |
| Mục *Cấu hình GMSA và node Windows trong Active Directory*, và các bước sinh credspec ở mục *Tạo các resource đặc tả thông tin xác thực GMSA* (`ipmo CredentialSpec.psm1`, `New-CredentialSpec`, `Get-CredentialSpec`, đổi JSON sang YAML) | cần một domain controller thật cộng host Windows đã join domain — ngoài phạm vi **cả hai nhánh** của Lab 15 | không có trong lộ trình; [bảng 1.1 của Lab 15](labs/LAB-15-NODE-WINDOWS.md#11-ánh-xạ-tài-liệu-sang-bài-thực-hành) ghi đúng lý do này |
| Mục *Xác thực tới các chia sẻ mạng bằng hostname hoặc FQDN* — registry key `EnableCompartmentNamespace` | registry và HNS là cơ chế của Windows; ba VM Linux không có thứ tương đương để quan sát | ranh giới HNS đã đọc ở bài [89 — Mạng trên Windows](89-windows-networking-vi.md) của chính giai đoạn 15 |
| Toàn bộ mục *Xử lý sự cố* — `nltest.exe /parentdomain`, `nltest.exe /query`, `nltest /sc_reset`, lifecycle hook `postStart` khởi động lại `netlogon` | mọi lệnh chạy **bên trong một container Windows đã join domain** | không có trong lộ trình; quy trình debug Windows nói chung đọc ở bài [315](315-debug-windows-vi.md), đối chiếu với quy trình Linux ở [Lab 15](labs/LAB-15-NODE-WINDOWS.md) phần **B8** |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [stable]`

Trang này hướng dẫn cách cấu hình
[Group Managed Service Accounts](https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview) (GMSA)
cho các Pod và container sẽ chạy trên node Windows. Group Managed Service Account là một loại
tài khoản Active Directory đặc biệt, cung cấp khả năng quản lý mật khẩu tự động, đơn giản hóa
việc quản lý service principal name (SPN), và cho phép ủy quyền việc quản lý cho các quản trị
viên khác trên nhiều máy chủ.

Trong Kubernetes, các đặc tả thông tin xác thực GMSA (GMSA credential spec) được cấu hình ở
phạm vi toàn cluster dưới dạng Custom Resource. Các Pod Windows, cũng như từng container riêng
lẻ bên trong một Pod, có thể được cấu hình để dùng một GMSA cho các chức năng dựa trên domain
(ví dụ: xác thực Kerberos) khi tương tác với các dịch vụ Windows khác.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh `kubectl` phải được cấu hình để giao
tiếp với cluster của bạn. Cluster này cần có các worker node chạy Windows. Mục này trình bày
một loạt bước khởi tạo chỉ cần thực hiện một lần cho mỗi cluster:

### Cài đặt CRD GMSACredentialSpec (Install the GMSACredentialSpec CRD)

Một [CustomResourceDefinition](378-custom-resource-definitions-vi.md) (CRD)
cho các resource đặc tả thông tin xác thực GMSA cần được cấu hình trên cluster để định nghĩa
kiểu custom resource `GMSACredentialSpec`. Tải file
[YAML](https://github.com/kubernetes-sigs/windows-gmsa/blob/master/admission-webhook/deploy/gmsa-crd.yml)
của GMSA CRD và lưu thành gmsa-crd.yaml. Tiếp theo, cài đặt CRD bằng lệnh
`kubectl apply -f gmsa-crd.yaml`

### Cài đặt các webhook để xác thực người dùng GMSA (Install webhooks to validate GMSA users)

Hai webhook cần được cấu hình trên cluster Kubernetes để điền và kiểm tra hợp lệ các tham chiếu
tới đặc tả thông tin xác thực GMSA ở cấp Pod hoặc container:

1. Một mutating webhook mở rộng các tham chiếu tới GMSA (theo tên trong đặc tả Pod) thành
   toàn bộ đặc tả thông tin xác thực ở dạng JSON bên trong Pod spec.

1. Một validating webhook đảm bảo mọi tham chiếu tới GMSA đều được phép sử dụng bởi service
   account của Pod.

Việc cài đặt các webhook trên và các đối tượng liên quan yêu cầu các bước sau:

1. Tạo một cặp khóa certificate (sẽ được dùng để cho phép container webhook giao tiếp với
   cluster)

1. Cài đặt một secret chứa certificate ở trên.

1. Tạo một deployment cho phần logic chính của webhook.

1. Tạo các cấu hình validating webhook và mutating webhook tham chiếu tới deployment đó.

Bạn có thể dùng một
[script](https://github.com/kubernetes-sigs/windows-gmsa/blob/master/admission-webhook/deploy/deploy-gmsa-webhook.sh)
để triển khai và cấu hình các webhook GMSA cùng những đối tượng liên quan đã nêu ở trên.
Script này có thể chạy với tùy chọn `--dry-run=server` để bạn xem trước những thay đổi sẽ
được áp dụng lên cluster của mình.

[Mẫu YAML](https://github.com/kubernetes-sigs/windows-gmsa/blob/master/admission-webhook/deploy/gmsa-webhook.yml.tpl)
mà script sử dụng cũng có thể được dùng để triển khai thủ công các webhook và đối tượng liên
quan (với các thay thế tham số phù hợp)

## Cấu hình GMSA và node Windows trong Active Directory (Configure GMSAs and Windows nodes in Active Directory)

Trước khi các Pod trong Kubernetes có thể được cấu hình để dùng GMSA, các GMSA mong muốn cần
được cấp phát (provision) trong Active Directory như mô tả trong
[tài liệu Windows GMSA](https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/getting-started-with-group-managed-service-accounts#BKMK_Step1).
Các worker node Windows (thuộc cluster Kubernetes) cần được cấu hình trong Active Directory
để có quyền truy cập thông tin xác thực bí mật gắn với GMSA mong muốn, như mô tả trong
[tài liệu Windows GMSA](https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/getting-started-with-group-managed-service-accounts#to-add-member-hosts-using-the-set-adserviceaccount-cmdlet).

## Tạo các resource đặc tả thông tin xác thực GMSA (Create GMSA credential spec resources)

Sau khi CRD GMSACredentialSpec đã được cài đặt (như mô tả ở trên), bạn có thể cấu hình các
custom resource chứa đặc tả thông tin xác thực GMSA. Đặc tả thông tin xác thực GMSA không chứa
dữ liệu bí mật hay nhạy cảm. Đó là thông tin mà container runtime có thể dùng để mô tả cho
Windows biết GMSA mong muốn của một container. Các đặc tả thông tin xác thực GMSA có thể được
sinh ra ở định dạng YAML bằng một
[script PowerShell](https://github.com/kubernetes-sigs/windows-gmsa/tree/master/scripts/GenerateCredentialSpecResource.ps1)
tiện ích.

Sau đây là các bước để tạo thủ công một đặc tả thông tin xác thực GMSA ở định dạng JSON rồi
chuyển đổi nó sang YAML:

1. Import [module](https://github.com/MicrosoftDocs/Virtualization-Documentation/blob/live/windows-server-container-tools/ServiceAccounts/CredentialSpec.psm1)
   CredentialSpec: `ipmo CredentialSpec.psm1`

1. Tạo một đặc tả thông tin xác thực ở định dạng JSON bằng `New-CredentialSpec`.
   Để tạo một đặc tả thông tin xác thực GMSA tên là WebApp1, hãy chạy
   `New-CredentialSpec -Name WebApp1 -AccountName WebApp1 -Domain $(Get-ADDomain -Current LocalComputer)`

1. Dùng `Get-CredentialSpec` để hiển thị đường dẫn của file JSON.

1. Chuyển đổi file credspec từ định dạng JSON sang YAML và thêm các trường phần đầu cần thiết
   `apiVersion`, `kind`, `metadata` và `credspec` để biến nó thành một custom resource
   GMSACredentialSpec có thể cấu hình trong Kubernetes.

Cấu hình YAML sau mô tả một đặc tả thông tin xác thực GMSA có tên `gmsa-WebApp1`:

```yaml
apiVersion: windows.k8s.io/v1
kind: GMSACredentialSpec
metadata:
  name: gmsa-WebApp1  # Đây là tên tùy ý nhưng sẽ được dùng làm tham chiếu
credspec:
  ActiveDirectoryConfig:
    GroupManagedServiceAccounts:
    - Name: WebApp1   # Username của tài khoản GMSA
      Scope: CONTOSO  # Tên domain NETBIOS
    - Name: WebApp1   # Username của tài khoản GMSA
      Scope: contoso.com # Tên domain DNS
  CmsPlugins:
  - ActiveDirectory
  DomainJoinConfig:
    DnsName: contoso.com  # Tên domain DNS
    DnsTreeName: contoso.com # Gốc của tên domain DNS
    Guid: 244818ae-87ac-4fcd-92ec-e79e5252348a  # GUID của Domain
    MachineAccountName: WebApp1 # Username của tài khoản GMSA
    NetBiosName: CONTOSO  # Tên domain NETBIOS
    Sid: S-1-5-21-2126449477-2524075714-3094792973 # SID của Domain
```

Resource đặc tả thông tin xác thực ở trên có thể được lưu thành `gmsa-Webapp1-credspec.yaml`
và áp dụng vào cluster bằng lệnh: `kubectl apply -f gmsa-Webapp1-credspec.yml`

## Cấu hình cluster role để bật RBAC trên các đặc tả thông tin xác thực GMSA cụ thể (Configure cluster role to enable RBAC on specific GMSA credential specs)

Cần định nghĩa một cluster role cho mỗi resource đặc tả thông tin xác thực GMSA. Cluster role
này cấp quyền dùng verb `use` trên một resource GMSA cụ thể cho một chủ thể (subject), thường
là một service account. Ví dụ sau cho thấy một cluster role cấp quyền sử dụng đặc tả thông tin
xác thực `gmsa-WebApp1` ở trên. Lưu file thành gmsa-webapp1-role.yaml và áp dụng bằng
`kubectl apply -f gmsa-webapp1-role.yaml`

```yaml
# Tạo Role để đọc credspec
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: webapp1-role
rules:
- apiGroups: ["windows.k8s.io"]
  resources: ["gmsacredentialspecs"]
  verbs: ["use"]
  resourceNames: ["gmsa-WebApp1"]
```

## Gán role cho các service account để dùng các credspec GMSA cụ thể (Assign role to service accounts to use specific GMSA credspecs)

Một service account (mà các Pod sẽ được cấu hình cùng) cần được gắn (bind) với cluster role đã
tạo ở trên. Việc này cho phép service account đó sử dụng resource đặc tả thông tin xác thực
GMSA mong muốn. Đoạn sau cho thấy service account default được gắn với cluster role
`webapp1-role` để dùng resource đặc tả thông tin xác thực `gmsa-WebApp1` đã tạo ở trên.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: allow-default-svc-account-read-on-gmsa-WebApp1
  namespace: default
subjects:
- kind: ServiceAccount
  name: default
  namespace: default
roleRef:
  kind: ClusterRole
  name: webapp1-role
  apiGroup: rbac.authorization.k8s.io
```

## Cấu hình tham chiếu đặc tả thông tin xác thực GMSA trong Pod spec (Configure GMSA credential spec reference in Pod spec)

Trường `securityContext.windowsOptions.gmsaCredentialSpecName` trong Pod spec được dùng để chỉ
định tham chiếu tới custom resource đặc tả thông tin xác thực GMSA mong muốn trong các Pod
spec. Cấu hình này khiến tất cả các container trong Pod spec sử dụng GMSA được chỉ định. Một
Pod spec mẫu với trường được điền để tham chiếu tới `gmsa-WebApp1`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: with-creds
  name: with-creds
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: with-creds
  template:
    metadata:
      labels:
        run: with-creds
    spec:
      securityContext:
        windowsOptions:
          gmsaCredentialSpecName: gmsa-webapp1
      containers:
      - image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2019
        imagePullPolicy: Always
        name: iis
      nodeSelector:
        kubernetes.io/os: windows
```

Từng container riêng lẻ trong một Pod spec cũng có thể chỉ định credspec GMSA mong muốn bằng
trường `securityContext.windowsOptions.gmsaCredentialSpecName` ở cấp container. Ví dụ:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: with-creds
  name: with-creds
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: with-creds
  template:
    metadata:
      labels:
        run: with-creds
    spec:
      containers:
      - image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2019
        imagePullPolicy: Always
        name: iis
        securityContext:
          windowsOptions:
            gmsaCredentialSpecName: gmsa-Webapp1
      nodeSelector:
        kubernetes.io/os: windows
```

Khi các Pod spec có các trường GMSA được điền (như mô tả ở trên) được áp dụng vào cluster,
chuỗi sự kiện sau sẽ diễn ra:

1. Mutating webhook phân giải và mở rộng tất cả các tham chiếu tới resource đặc tả thông tin
   xác thực GMSA thành nội dung đầy đủ của đặc tả thông tin xác thực GMSA.

1. Validating webhook đảm bảo service account gắn với Pod được phép dùng verb `use` trên đặc
   tả thông tin xác thực GMSA được chỉ định.

1. Container runtime cấu hình mỗi container Windows với đặc tả thông tin xác thực GMSA được
   chỉ định, để container có thể nhận danh tính (identity) của GMSA trong Active Directory
   và truy cập các dịch vụ trong domain bằng danh tính đó.

## Xác thực tới các chia sẻ mạng bằng hostname hoặc FQDN (Authenticating to network shares using hostname or FQDN)

Nếu bạn gặp sự cố khi kết nối tới các chia sẻ SMB (SMB share) từ Pod bằng hostname hoặc FQDN,
nhưng lại truy cập được các chia sẻ đó qua địa chỉ IPv4, hãy chắc chắn rằng registry key sau
được thiết lập trên các node Windows.

```cmd
reg add "HKLM\SYSTEM\CurrentControlSet\Services\hns\State" /v EnableCompartmentNamespace /t REG_DWORD /d 1
```

Sau đó, các Pod đang chạy cần được tạo lại để nhận các thay đổi hành vi. Thông tin thêm về
cách registry key này được sử dụng có thể xem
[tại đây](https://github.com/microsoft/hcsshim/blob/885f896c5a8548ca36c88c4b87fd2208c8d16543/internal/uvm/create.go#L74-L83)

## Xử lý sự cố (Troubleshooting)

Nếu bạn gặp khó khăn khi đưa GMSA vào hoạt động trong môi trường của mình, có một số bước xử
lý sự cố bạn có thể thực hiện.

Trước tiên, hãy chắc chắn rằng credspec đã được truyền vào Pod. Để làm điều này, bạn cần
`exec` vào một trong các Pod của mình và kiểm tra kết quả của lệnh `nltest.exe /parentdomain`.

Trong ví dụ dưới đây, Pod đã không nhận được credspec đúng cách:

```PowerShell
kubectl exec -it iis-auth-7776966999-n5nzr powershell.exe
```

`nltest.exe /parentdomain` trả về lỗi sau:

```output
Getting parent domain failed: Status = 1722 0x6ba RPC_S_SERVER_UNAVAILABLE
```

Nếu Pod của bạn đã nhận được credspec đúng cách, bước tiếp theo là kiểm tra giao tiếp với
domain. Trước tiên, từ bên trong Pod của bạn, hãy nhanh chóng chạy một lệnh nslookup để tìm
gốc (root) của domain.

Việc này cho chúng ta biết 3 điều:

1. Pod có thể kết nối tới DC
1. DC có thể kết nối tới Pod
1. DNS đang hoạt động đúng.

Nếu kiểm tra DNS và giao tiếp thành công, tiếp theo bạn cần kiểm tra xem Pod đã thiết lập được
kênh giao tiếp bảo mật (secure channel) với domain hay chưa. Để làm điều này, một lần nữa,
hãy `exec` vào Pod của bạn và chạy lệnh `nltest.exe /query`.

```PowerShell
nltest.exe /query
```

Kết quả trả về output sau:

```output
I_NetLogonControl failed: Status = 1722 0x6ba RPC_S_SERVER_UNAVAILABLE
```

Điều này cho chúng ta biết rằng, vì lý do nào đó, Pod đã không thể đăng nhập vào domain bằng
tài khoản được chỉ định trong credspec. Bạn có thể thử sửa chữa kênh bảo mật bằng cách chạy
lệnh sau:

```PowerShell
nltest /sc_reset:domain.example
```

Nếu lệnh thành công, bạn sẽ thấy output tương tự như sau:

```output
Flags: 30 HAS_IP  HAS_TIMESERV
Trusted DC Name \\dc10.domain.example
Trusted DC Connection Status Status = 0 0x0 NERR_Success
The command completed successfully
```

Nếu cách trên khắc phục được lỗi, bạn có thể tự động hóa bước này bằng cách thêm lifecycle
hook sau vào Pod spec của mình. Nếu nó không khắc phục được lỗi, bạn sẽ cần xem xét lại
credspec của mình và xác nhận rằng nó chính xác và đầy đủ.

```yaml
        image: registry.domain.example/iis-auth:1809v1
        lifecycle:
          postStart:
            exec:
              command: ["powershell.exe","-command","do { Restart-Service -Name netlogon } while ( $($Result = (nltest.exe /query); if ($Result -like '*0x0 NERR_Success*') {return $true} else {return $false}) -eq $false)"]
        imagePullPolicy: IfNotPresent
```

Nếu bạn thêm mục `lifecycle` như trên vào Pod spec, Pod sẽ thực thi các lệnh được liệt kê để
khởi động lại dịch vụ `netlogon` cho tới khi lệnh `nltest.exe /query` kết thúc mà không có
lỗi.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 15 — kể cả khi bạn
chạy nhánh B của Lab 15 trên ba VM Linux:

1. Một GMSA credential spec chứa `Name`, `Scope`, `DnsName`, `Guid`, `Sid` của domain. Nó có chứa
   **mật khẩu** của tài khoản GMSA không? Nếu không thì thành phần nào lấy được mật khẩu, và bài
   đặt điều kiện gì lên các worker node Windows để việc đó xảy ra được?
2. **Câu bẫy.** Bạn apply CRD `GMSACredentialSpec` cùng hai webhook lên cluster ba VM Ubuntu
   `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`, và `kubectl get crd | grep -i gmsa` cho
   ra kết quả. Cluster của bạn đã **dùng được** GMSA chưa? Còn thiếu những gì, theo đúng các điều
   kiện bài liệt kê?
3. Hai webhook làm hai việc khác nhau. Việc nào của cái nào, và nếu thiếu **validating** webhook
   thì cluster mất bảo đảm gì?
4. `securityContext.windowsOptions.gmsaCredentialSpecName` đặt ở cấp Pod khác gì đặt ở cấp
   container? Và để một Pod dùng được credspec `gmsa-WebApp1`, chuỗi RBAC phải gồm những object
   nào, cấp verb gì, trên resource nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Bài nói thẳng ở mục *Tạo các resource đặc tả thông tin xác thực GMSA*: đặc tả thông
   tin xác thực GMSA **không chứa dữ liệu bí mật hay nhạy cảm** — nó chỉ là thông tin để container
   runtime **mô tả cho Windows biết GMSA mong muốn** của một container. Mật khẩu do
   **node Windows** lấy trực tiếp từ Active Directory. Điều kiện bài đặt ra nằm ở mục *Cấu hình
   GMSA và node Windows trong Active Directory*: các worker node Windows **cần được cấu hình trong
   Active Directory để có quyền truy cập thông tin xác thực bí mật** gắn với GMSA đó. Không có bước
   đó thì credspec đúng đến mấy cũng vô dụng.
2. **Chưa.** Có CRD và webhook mới chỉ là **bề mặt API**. Bài liệt kê tiếp hai điều kiện nữa mà
   cluster này không thỏa: mục *Trước khi bạn bắt đầu* nói cluster **cần có các worker node chạy
   Windows** — ba VM của bạn đều là Ubuntu; và mục *Cấu hình GMSA và node Windows trong Active
   Directory* nói các worker node Windows đó phải **được cấu hình trong Active Directory**. Chỗ dễ
   sai là tưởng `kubectl apply` thành công nghĩa là tính năng chạy được: API server chấp nhận CRD
   trên **mọi** cluster, vì nó chỉ đăng ký một kiểu object mới, không kiểm tra hệ điều hành của
   node nào cả. Muốn xác nhận, hỏi đủ ba tầng: bề mặt API, node Windows, node đã join domain.
3. **Mutating** webhook **mở rộng** các tham chiếu tới GMSA — vốn chỉ là **một cái tên** trong Pod
   spec — thành **toàn bộ nội dung credspec ở dạng JSON** bên trong Pod spec. **Validating** webhook
   **đảm bảo mọi tham chiếu tới GMSA đều được phép sử dụng bởi service account của Pod**. Thiếu
   validating webhook thì mất đúng vế phân quyền: **bất kỳ ai tạo được Pod cũng gọi tên được bất kỳ
   credspec nào**, tức mượn được danh tính domain mà service account của họ không được cấp — trong
   khi ClusterRole với verb `use` vẫn nằm đó nhưng không còn ai thực thi.
4. Đặt ở **cấp Pod** thì **tất cả các container trong Pod spec** dùng GMSA được chỉ định; đặt ở
   **cấp container** thì chỉ container đó dùng. Chuỗi RBAC gồm **hai object**: một **ClusterRole**
   cấp verb **`use`** trên resource **`gmsacredentialspecs`** thuộc nhóm API **`windows.k8s.io`**,
   thu hẹp bằng `resourceNames: ["gmsa-WebApp1"]`; và một **RoleBinding** gắn ClusterRole đó cho
   **ServiceAccount** mà Pod chạy dưới danh nghĩa (ví dụ của bài dùng service account `default`
   trong namespace `default`). Cần cả hai vì ClusterRole chỉ **định nghĩa** quyền, RoleBinding mới
   **trao** quyền cho một chủ thể cụ thể.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
