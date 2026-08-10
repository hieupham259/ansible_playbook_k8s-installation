# Cấu hình RunAsUserName cho Pod và container Windows (Configure RunAsUserName for Windows pods and containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-runasusername/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [stable]`

Trang này hướng dẫn cách sử dụng thiết lập `runAsUserName` cho các Pod và container sẽ chạy trên
node Windows. Thiết lập này gần tương đương với thiết lập `runAsUser` dành riêng cho Linux, cho
phép bạn chạy ứng dụng trong container dưới một tên người dùng (username) khác với mặc định.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có một cluster Kubernetes và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Cluster cần có các worker node Windows, nơi các Pod với container chạy
workload Windows sẽ được lập lịch (schedule).

## Đặt username cho Pod (Set the Username for a Pod)

Để chỉ định username dùng để thực thi các tiến trình trong container của Pod, hãy thêm field
`securityContext`
([PodSecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#podsecuritycontext-v1-core))
vào đặc tả (specification) của Pod, và bên trong nó là field `windowsOptions`
([WindowsSecurityContextOptions](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#windowssecuritycontextoptions-v1-core))
chứa field `runAsUserName`.

Các tùy chọn security context Windows mà bạn chỉ định cho Pod sẽ áp dụng cho tất cả các
container và init container trong Pod.

Đây là file cấu hình cho một Pod Windows có field `runAsUserName` được thiết lập:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: run-as-username-pod-demo
spec:
  securityContext:
    windowsOptions:
      runAsUserName: "ContainerUser"
  containers:
  - name: run-as-username-demo
    image: mcr.microsoft.com/windows/servercore:ltsc2019
    command: ["ping", "-t", "localhost"]
  nodeSelector:
    kubernetes.io/os: windows
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/windows/run-as-username-pod.yaml
```

Xác nhận rằng container của Pod đang chạy:

```shell
kubectl get pod run-as-username-pod-demo
```

Mở một shell tới container đang chạy:

```shell
kubectl exec -it run-as-username-pod-demo -- powershell
```

Kiểm tra rằng shell đang chạy dưới đúng username:

```powershell
echo $env:USERNAME
```

Kết quả sẽ là:

```
ContainerUser
```

## Đặt username cho Container (Set the Username for a Container)

Để chỉ định username dùng để thực thi các tiến trình của một container, hãy thêm field
`securityContext`
([SecurityContext](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#securitycontext-v1-core))
vào manifest của container, và bên trong nó là field `windowsOptions`
([WindowsSecurityContextOptions](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.36/#windowssecuritycontextoptions-v1-core))
chứa field `runAsUserName`.

Các tùy chọn security context Windows mà bạn chỉ định cho một container chỉ áp dụng cho riêng
container đó, và chúng ghi đè (override) các thiết lập đã đặt ở cấp Pod.

Đây là file cấu hình cho một Pod có một container, với field `runAsUserName` được thiết lập ở cả
cấp Pod và cấp container:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: run-as-username-container-demo
spec:
  securityContext:
    windowsOptions:
      runAsUserName: "ContainerUser"
  containers:
  - name: run-as-username-demo
    image: mcr.microsoft.com/windows/servercore:ltsc2019
    command: ["ping", "-t", "localhost"]
    securityContext:
        windowsOptions:
            runAsUserName: "ContainerAdministrator"
  nodeSelector:
    kubernetes.io/os: windows
```

Tạo Pod:

```shell
kubectl apply -f https://k8s.io/examples/windows/run-as-username-container.yaml
```

Xác nhận rằng container của Pod đang chạy:

```shell
kubectl get pod run-as-username-container-demo
```

Mở một shell tới container đang chạy:

```shell
kubectl exec -it run-as-username-container-demo -- powershell
```

Kiểm tra rằng shell đang chạy dưới đúng username (username được thiết lập ở cấp container):

```powershell
echo $env:USERNAME
```

Kết quả sẽ là:

```
ContainerAdministrator
```

## Các giới hạn về username trên Windows (Windows Username limitations)

Để sử dụng tính năng này, giá trị đặt trong field `runAsUserName` phải là một username hợp lệ.
Nó phải có định dạng: `DOMAIN\USER`, trong đó phần `DOMAIN\` là tùy chọn. Username trên Windows
không phân biệt chữ hoa chữ thường. Ngoài ra, có một số ràng buộc đối với `DOMAIN` và `USER`:

- Field `runAsUserName` không được rỗng, và không được chứa các ký tự điều khiển (giá trị ASCII:
  `0x00-0x1F`, `0x7F`)
- `DOMAIN` phải là một tên NetBios hoặc một tên DNS, mỗi loại có ràng buộc riêng:
  - Tên NetBios: tối đa 15 ký tự, không được bắt đầu bằng `.` (dấu chấm), và không được chứa
    các ký tự sau: `\ / : * ? " < > |`
  - Tên DNS: tối đa 255 ký tự, chỉ chứa các ký tự chữ và số, dấu chấm và dấu gạch ngang, và
    không được bắt đầu hoặc kết thúc bằng `.` (dấu chấm) hoặc `-` (dấu gạch ngang).
- `USER` có tối đa 20 ký tự, không được chứa *chỉ toàn* dấu chấm hoặc dấu cách, và không được
  chứa các ký tự sau: `" / \ [ ] : ; | = , + * ? < > @`.

Ví dụ về các giá trị chấp nhận được cho field `runAsUserName`: `ContainerAdministrator`,
`ContainerUser`, `NT AUTHORITY\NETWORK SERVICE`, `NT AUTHORITY\LOCAL SERVICE`.

Để biết thêm thông tin về các giới hạn này, hãy xem
[tại đây](https://support.microsoft.com/en-us/help/909264/naming-conventions-in-active-directory-for-computers-domains-sites-and)
và
[tại đây](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/new-localuser?view=powershell-5.1).

## Tiếp theo (What's next)

* [Hướng dẫn lập lịch container Windows trong Kubernetes](https://kubernetes.io/docs/concepts/windows/user-guide/)
* [Quản lý danh tính workload với Group Managed Service Accounts (GMSA)](https://kubernetes.io/docs/concepts/windows/user-guide/#managing-workload-identity-with-group-managed-service-accounts)
* [Cấu hình GMSA cho Pod và container Windows](https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa/)
