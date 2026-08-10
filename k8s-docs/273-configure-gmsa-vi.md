# Cấu hình GMSA cho Pod và container Windows (Configure GMSA for Windows Pods and containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa/>

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

Một [CustomResourceDefinition](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/) (CRD)
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
