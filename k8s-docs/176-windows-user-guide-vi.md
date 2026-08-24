# Hướng dẫn chạy Windows container trong Kubernetes (Guide for Running Windows Containers in Kubernetes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/windows/user-guide/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 15](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows),
bài 3/7 · Kiểm chứng ở Lab 15 (tùy chọn, chưa viết, xem
[bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Lộ trình ghi rõ: bỏ qua hoàn toàn giai đoạn 15 nếu cluster của bạn chỉ có Linux.** Bài này còn
ghi ngay ở mục *Trước khi bạn bắt đầu* rằng bạn cần một cluster **có worker node chạy Windows
Server** — cluster lab ba VM Ubuntu của bạn không có, nên phần thực hành không chạy được.

Bài vốn là **hướng dẫn từng bước**, nhưng phần đáng đọc với admin không phải các lệnh mà là
**quy tắc lập lịch**: làm sao workload Windows và Linux không lạc sang nhầm node. Bỏ qua manifest
và lệnh PowerShell, đọc kỹ ba mục cuối.

**Phải hiểu ở lần đọc này:**

- **`.spec.os.name` không được scheduler dùng để chọn node.** Bài nói thẳng: bộ lập lịch không
  dùng giá trị đó khi gán Pod vào node, và "giá trị `.spec.os.name` không có tác dụng đối với
  việc lập lịch các pod Windows". Nó chỉ khai báo Pod được thiết kế cho hệ điều hành nào.
- Cơ chế thật là **`nodeSelector` `kubernetes.io/os: windows`** — thực hành tốt nhất mà bài
  khuyến nghị. Mọi node đều có sẵn label `kubernetes.io/os` và `kubernetes.io/arch`.
- Vì hệ sinh thái manifest Linux có sẵn (Helm chart, Pod sinh bằng operator) không thể sửa hết,
  giải pháp thay thế là **taint node Windows ngay khi đăng ký**:
  `--register-with-taints='os=windows:NoSchedule'`. Khi đó Pod Windows cần **cả `nodeSelector`
  lẫn toleration khớp**.
- Cùng cluster có nhiều phiên bản Windows Server thì `kubernetes.io/os` là chưa đủ: phiên bản của
  Pod phải khớp phiên bản node, và Kubernetes tự thêm label
  **`node.kubernetes.io/windows-build`** (Server 2022 = `10.0.20348`, Server 2025 = `10.0.26100`).
  **RuntimeClass** đóng gói sẵn `nodeSelector` + toleration để Pod chỉ cần khai
  `runtimeClassName`.
- Khả năng quan sát khác Linux: workload Windows thường ghi log vào **ETW hoặc event log của ứng
  dụng**, không ra STDOUT — nên `kubectl logs <pod>` không thấy gì. Cách được khuyến nghị là dùng
  **LogMonitor** chuyển tiếp các nguồn log đó ra STDOUT.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Manifest `win-webserver.yaml` và lệnh PowerShell dài trong `command` | là ví dụ để chạy trên cluster có node Windows | khi môi trường thực sự có node Windows |
| Danh sách bảy phép kiểm chứng kết nối (pod↔pod, service, NodePort, DNS…) | là quy trình nghiệm thu trên cluster thật | khi môi trường thực sự có node Windows |
| Ghi chú "host container Windows không truy cập được IP service lập lịch trên chính nó" | là hạn chế của ngăn xếp mạng Windows | bài [89](89-windows-networking-vi.md) |
| *Sử dụng username có thể cấu hình* và *GMSA* | là chủ đề danh tính và bảo mật Windows | bài [131](131-windows-security-vi.md) |
| Feature gate `IdentifyPodOS` cho bản Kubernetes cũ hơn 1.24 | cluster hiện đại không cần | không cần |

---

Trang này cung cấp hướng dẫn từng bước để bạn chạy Windows container bằng Kubernetes.
Trang này cũng làm nổi bật một số chức năng dành riêng cho Windows trong Kubernetes.

Điều quan trọng cần lưu ý là việc tạo và triển khai các Service và workload trên Kubernetes
hoạt động gần như giống nhau đối với container Linux và Windows.
Các [lệnh kubectl](https://kubernetes.io/docs/reference/kubectl/) để tương tác với cluster là giống hệt nhau.
Các ví dụ trong trang này được cung cấp nhằm giúp bạn nhanh chóng làm quen với Windows container.

## Mục tiêu (Objectives)

Cấu hình một deployment ví dụ để chạy Windows container trên một node Windows.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có sẵn quyền truy cập vào một cluster Kubernetes có chứa một
worker node chạy Windows Server.

## Bắt đầu: Triển khai một workload Windows (Getting Started: Deploying a Windows workload)

File YAML ví dụ dưới đây triển khai một ứng dụng webserver đơn giản chạy bên trong một Windows container.

Tạo một manifest tên là `win-webserver.yaml` với nội dung dưới đây:

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: win-webserver
  labels:
    app: win-webserver
spec:
  ports:
    # port mà service này sẽ phục vụ
    - port: 80
      targetPort: 80
  selector:
    app: win-webserver
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: win-webserver
  name: win-webserver
spec:
  replicas: 2
  selector:
    matchLabels:
      app: win-webserver
  template:
    metadata:
      labels:
        app: win-webserver
      name: win-webserver
    spec:
     containers:
      - name: windowswebserver
        image: mcr.microsoft.com/windows/servercore:ltsc2019
        command:
        - powershell.exe
        - -command
        - "<#code used from https://gist.github.com/19WAS85/5424431#> ; $$listener = New-Object System.Net.HttpListener ; $$listener.Prefixes.Add('http://*:80/') ; $$listener.Start() ; $$callerCounts = @{} ; Write-Host('Listening at http://*:80/') ; while ($$listener.IsListening) { ;$$context = $$listener.GetContext() ;$$requestUrl = $$context.Request.Url ;$$clientIP = $$context.Request.RemoteEndPoint.Address ;$$response = $$context.Response ;Write-Host '' ;Write-Host('> {0}' -f $$requestUrl) ;  ;$$count = 1 ;$$k=$$callerCounts.Get_Item($$clientIP) ;if ($$k -ne $$null) { $$count += $$k } ;$$callerCounts.Set_Item($$clientIP, $$count) ;$$ip=(Get-NetAdapter | Get-NetIpAddress); $$header='<html><body><H1>Windows Container Web Server</H1>' ;$$callerCountsString='' ;$$callerCounts.Keys | % { $$callerCountsString+='<p>IP {0} callerCount {1} ' -f $$ip[1].IPAddress,$$callerCounts.Item($$_) } ;$$footer='</body></html>' ;$$content='{0}{1}{2}' -f $$header,$$callerCountsString,$$footer ;Write-Output $$content ;$$buffer = [System.Text.Encoding]::UTF8.GetBytes($$content) ;$$response.ContentLength64 = $$buffer.Length ;$$response.OutputStream.Write($$buffer, 0, $$buffer.Length) ;$$response.Close() ;$$responseStatus = $$response.StatusCode ;Write-Host('< {0}' -f $$responseStatus)  } ; "
     nodeSelector:
      kubernetes.io/os: windows
```

> **Ghi chú:**
> Ánh xạ port (port mapping) cũng được hỗ trợ, nhưng để đơn giản, ví dụ này
> expose port 80 của container trực tiếp cho Service.

1. Kiểm tra rằng tất cả các node đều ở trạng thái khỏe mạnh (healthy):

    ```bash
    kubectl get nodes
    ```

1. Triển khai service và theo dõi các cập nhật của pod:

    ```bash
    kubectl apply -f win-webserver.yaml
    kubectl get pods -o wide -w
    ```

    Khi service được triển khai đúng, cả hai Pod đều được đánh dấu là Ready. Để thoát khỏi lệnh watch, nhấn Ctrl+C.

1. Kiểm tra rằng deployment đã thành công. Để xác minh:

    * Liệt kê được nhiều pod từ node control plane Linux, dùng `kubectl get pods`
    * Giao tiếp node-tới-pod qua mạng: `curl` vào port 80 của các IP pod từ node control plane Linux
      để kiểm tra phản hồi của web server
    * Giao tiếp pod-tới-pod: ping giữa các pod (và giữa các host với nhau, nếu bạn có nhiều hơn một node Windows)
      bằng `kubectl exec`
    * Giao tiếp service-tới-pod: `curl` vào IP ảo của service (xem được bằng `kubectl get services`)
      từ node control plane Linux và từ từng pod
    * Khám phá service (service discovery): `curl` tên service kèm [hậu tố DNS mặc định](10-dns-pod-service-vi.md#services) của Kubernetes
    * Kết nối chiều vào (inbound connectivity): `curl` NodePort từ node control plane Linux hoặc từ các máy bên ngoài cluster
    * Kết nối chiều ra (outbound connectivity): `curl` các IP bên ngoài từ bên trong pod bằng `kubectl exec`

> **Ghi chú:**
> Do các hạn chế hiện tại của nền tảng ngăn xếp mạng (networking stack) Windows, các host container Windows
> không thể truy cập IP của các service được lập lịch trên chính chúng.
> Chỉ các pod Windows mới có thể truy cập IP của service.

## Khả năng quan sát (Observability)

### Thu thập log từ các workload (Capturing logs from workloads)

Log là một thành phần quan trọng của khả năng quan sát (observability); chúng giúp người dùng
hiểu rõ khía cạnh vận hành của các workload và là yếu tố then chốt để khắc phục sự cố.
Vì Windows container và các workload bên trong Windows container hoạt động khác với container Linux,
người dùng từng gặp khó khăn trong việc thu thập log, làm hạn chế khả năng quan sát khi vận hành.
Ví dụ, các workload Windows thường được cấu hình để ghi log vào ETW (Event Tracing for Windows)
hoặc đẩy các mục log vào event log của ứng dụng.
[LogMonitor](https://github.com/microsoft/windows-container-tools/tree/master/LogMonitor), một công cụ mã nguồn mở của Microsoft,
là cách được khuyến nghị để giám sát các nguồn log đã cấu hình bên trong Windows container.
LogMonitor hỗ trợ giám sát event log, các ETW provider, và log ứng dụng tùy chỉnh,
chuyển tiếp chúng tới STDOUT để `kubectl logs <pod>` có thể đọc được.

Làm theo hướng dẫn trên trang GitHub của LogMonitor để sao chép các file nhị phân (binary) và file cấu hình của nó
vào tất cả các container của bạn, và thêm các entrypoint cần thiết để LogMonitor đẩy log của bạn ra STDOUT.

## Cấu hình user cho container (Configuring container user)

### Sử dụng username có thể cấu hình cho Container (Using configurable Container usernames)

Windows container có thể được cấu hình để chạy entrypoint và các tiến trình (process)
với username khác với mặc định của image.
Tìm hiểu thêm [tại đây](278-configure-runasusername-vi.md).

### Quản lý danh tính workload với Group Managed Service Accounts (Managing Workload Identity with Group Managed Service Accounts) {#managing-workload-identity-with-group-managed-service-accounts}

Các workload Windows container có thể được cấu hình để sử dụng Group Managed Service Accounts (GMSA).
Group Managed Service Accounts là một loại tài khoản Active Directory đặc biệt cung cấp khả năng quản lý mật khẩu tự động,
đơn giản hóa việc quản lý service principal name (SPN), và khả năng ủy quyền việc quản lý cho các quản trị viên khác trên nhiều server.
Các container được cấu hình với GMSA có thể truy cập các tài nguyên miền (domain) Active Directory bên ngoài
trong khi mang danh tính đã được cấu hình cùng GMSA.
Tìm hiểu thêm về cách cấu hình và sử dụng GMSA cho Windows container [tại đây](273-configure-gmsa-vi.md).

## Taint và toleration (Taints and tolerations)

Người dùng cần sử dụng kết hợp taint và node selector để lập lịch
các workload Linux và Windows vào đúng các node dành riêng cho hệ điều hành tương ứng.
Cách tiếp cận được khuyến nghị được trình bày dưới đây,
với một trong những mục tiêu chính là cách tiếp cận này không được phá vỡ tính tương thích của các workload Linux hiện có.

Bạn có thể (và nên) đặt `.spec.os.name` cho mỗi Pod để chỉ ra hệ điều hành
mà các container trong Pod đó được thiết kế để chạy trên đó. Với các Pod chạy container Linux, đặt
`.spec.os.name` là `linux`. Với các Pod chạy container Windows, đặt `.spec.os.name`
là `windows`.

> **Ghi chú:**
> Nếu bạn đang chạy phiên bản Kubernetes cũ hơn 1.24, bạn có thể cần bật
> [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) `IdentifyPodOS`
> để có thể đặt giá trị cho `.spec.pod.os`.

Bộ lập lịch (scheduler) không sử dụng giá trị của `.spec.os.name` khi gán Pod vào node. Bạn nên
sử dụng các cơ chế thông thường của Kubernetes để
[gán pod vào node](138-assign-pod-node-vi.md)
nhằm đảm bảo control plane của cluster đặt các pod lên những node đang chạy
hệ điều hành phù hợp.

Giá trị `.spec.os.name` không có tác dụng đối với việc lập lịch các pod Windows,
vì vậy taint và toleration (hoặc node selector) vẫn cần thiết
để đảm bảo các pod Windows được đặt lên các node Windows phù hợp.

### Đảm bảo workload đặc thù hệ điều hành được đặt lên đúng host container (Ensuring OS-specific workloads land on the appropriate container host) {#ensuring-os-specific-workloads-land-on-the-appropriate-container-host}

Người dùng có thể đảm bảo Windows container được lập lịch trên đúng host bằng cách sử dụng taint và toleration.
Tất cả các node chạy Kubernetes v1.36 đều có sẵn các label mặc định sau:

* kubernetes.io/os = [windows|linux]
* kubernetes.io/arch = [amd64|arm64|...]

Nếu đặc tả (specification) của Pod không chỉ định `nodeSelector` chẳng hạn như `"kubernetes.io/os": windows`,
Pod đó có thể bị lập lịch lên bất kỳ host nào, Windows hoặc Linux.
Điều này có thể gây ra vấn đề vì Windows container chỉ chạy được trên Windows và Linux container chỉ chạy được trên Linux.
Thực hành tốt nhất cho Kubernetes v1.36 là sử dụng `nodeSelector`.

Tuy nhiên, trong nhiều trường hợp, người dùng đã có sẵn một số lượng lớn deployment cho container Linux,
cũng như một hệ sinh thái các cấu hình dựng sẵn, chẳng hạn như các Helm chart của cộng đồng, và các trường hợp sinh Pod bằng chương trình, chẳng hạn với các operator.
Trong những tình huống đó, bạn có thể ngần ngại thay đổi cấu hình để thêm trường `nodeSelector` vào tất cả các Pod và Pod template.
Giải pháp thay thế là sử dụng taint. Vì kubelet có thể đặt taint trong quá trình đăng ký,
nó có thể dễ dàng được điều chỉnh để tự động thêm một taint khi chỉ chạy trên Windows.

Ví dụ: `--register-with-taints='os=windows:NoSchedule'`

Bằng cách thêm taint vào tất cả các node Windows, sẽ không có gì được lập lịch lên chúng (bao gồm cả các Pod Linux hiện có).
Để một Pod Windows được lập lịch lên một node Windows,
Pod đó cần cả `nodeSelector` lẫn toleration khớp tương ứng để chọn Windows.

```yaml
nodeSelector:
    kubernetes.io/os: windows
    node.kubernetes.io/windows-build: '10.0.20348'
tolerations:
    - key: "os"
      operator: "Equal"
      value: "windows"
      effect: "NoSchedule"
```

### Xử lý nhiều phiên bản Windows trong cùng một cluster (Handling multiple Windows versions in the same cluster)

Phiên bản Windows Server mà mỗi pod sử dụng phải khớp với phiên bản của node. Nếu bạn muốn sử dụng nhiều phiên bản Windows
Server trong cùng một cluster, bạn nên đặt thêm các node label và trường `nodeSelector` bổ sung.

Kubernetes tự động thêm một label,
[`node.kubernetes.io/windows-build`](https://kubernetes.io/docs/reference/labels-annotations-taints/#nodekubernetesiowindows-build),
để đơn giản hóa việc này.

Label này phản ánh số phiên bản chính (major), phụ (minor) và số build của Windows cần khớp để đảm bảo tương thích.
Dưới đây là các giá trị được dùng cho từng phiên bản Windows Server:

| Tên sản phẩm                         | Phiên bản              |
|--------------------------------------|------------------------|
| Windows Server 2022                  | 10.0.20348             |
| Windows Server 2025                  | 10.0.26100             |

### Đơn giản hóa với RuntimeClass (Simplifying with RuntimeClass)

[RuntimeClass] có thể được dùng để đơn giản hóa quá trình sử dụng taint và toleration.
Quản trị viên cluster có thể tạo một object `RuntimeClass` dùng để đóng gói các taint và toleration này.

1. Lưu file này vào `runtimeClasses.yml`. File này bao gồm `nodeSelector` phù hợp
   cho hệ điều hành, kiến trúc và phiên bản Windows.

   ```yaml
   ---
   apiVersion: node.k8s.io/v1
   kind: RuntimeClass
   metadata:
     name: windows-2019
   handler: example-container-runtime-handler
   scheduling:
     nodeSelector:
       kubernetes.io/os: 'windows'
       kubernetes.io/arch: 'amd64'
       node.kubernetes.io/windows-build: '10.0.20348'
     tolerations:
     - effect: NoSchedule
       key: os
       operator: Equal
       value: "windows"
   ```

1. Chạy `kubectl create -f runtimeClasses.yml` với vai trò quản trị viên cluster
1. Thêm `runtimeClassName: windows-2019` vào các spec của Pod khi phù hợp

   Ví dụ:

   ```yaml
   ---
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: iis-2019
     labels:
       app: iis-2019
   spec:
     replicas: 1
     template:
       metadata:
         name: iis-2019
         labels:
           app: iis-2019
       spec:
         runtimeClassName: windows-2019
         containers:
         - name: iis
           image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2019
           resources:
             limits:
               cpu: 1
               memory: 800Mi
             requests:
               cpu: .1
               memory: 300Mi
           ports:
             - containerPort: 80
    selector:
       matchLabels:
         app: iis-2019
   ---
   apiVersion: v1
   kind: Service
   metadata:
     name: iis
   spec:
     type: LoadBalancer
     ports:
     - protocol: TCP
       port: 80
     selector:
       app: iis-2019
   ```

[RuntimeClass]: https://kubernetes.io/docs/concepts/containers/runtime-class/

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 15:

1. Câu bẫy: bạn đặt `.spec.os.name: windows` cho một Pod và không đặt gì thêm. Scheduler có nhờ
   đó mà đưa Pod lên node Windows không? Vậy trường đó dùng để làm gì?
2. Trên cluster lab hiện tại, mọi Deployment Linux của bạn đều không có `nodeSelector`. Nếu thêm
   một node Windows vào cluster, chuyện gì có thể xảy ra, và bài đề xuất cách nào để không phải
   sửa toàn bộ manifest cũ?
3. Cluster có cả node Windows Server 2022 và 2025. Vì sao `nodeSelector` với
   `kubernetes.io/os: windows` là chưa đủ, và Kubernetes cung cấp label nào để giải quyết?
4. Ứng dụng .NET của bạn ghi log vào event log của Windows. `kubectl logs` thấy gì, và bài khuyến
   nghị cách nào?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Không.** Bài viết: "Bộ lập lịch (scheduler) **không sử dụng giá trị của `.spec.os.name`** khi
   gán Pod vào node", và nhắc lại "giá trị `.spec.os.name` không có tác dụng đối với việc lập
   lịch các pod Windows, vì vậy taint và toleration (hoặc node selector) vẫn cần thiết". Trường
   này chỉ **khai báo** hệ điều hành mà các container trong Pod được thiết kế để chạy trên đó —
   và như bài [175](175-windows-intro-vi.md) cho thấy, khai báo đó khóa một loạt trường `spec`
   khác. Đây là bẫy kinh điển: cái tên `os` khiến người ta tưởng nó là điều kiện lập lịch.
2. Pod Linux **có thể bị lập lịch lên node Windows** (và ngược lại), vì "nếu đặc tả của Pod không
   chỉ định `nodeSelector` chẳng hạn như `"kubernetes.io/os": windows`, Pod đó có thể bị lập lịch
   lên **bất kỳ host nào**" — mà Linux container chỉ chạy được trên Linux. Cách bài đề xuất khi
   không muốn sửa hàng loạt manifest và Helm chart có sẵn: **dùng taint**. Kubelet có thể đặt
   taint ngay khi đăng ký node, ví dụ `--register-with-taints='os=windows:NoSchedule'`. Sau đó
   **không gì được lập lịch lên node Windows** trừ Pod có **cả `nodeSelector` lẫn toleration
   khớp**.
3. Vì "phiên bản Windows Server mà mỗi pod sử dụng **phải khớp với phiên bản của node**" — chọn
   đúng "là Windows" chưa đủ, còn phải đúng bản build. Kubernetes tự động thêm label
   **`node.kubernetes.io/windows-build`**, phản ánh số major, minor và build: Windows Server 2022
   là `10.0.20348`, Windows Server 2025 là `10.0.26100`. Bạn đưa label này vào `nodeSelector`,
   hoặc gói cả `nodeSelector` lẫn toleration vào một **RuntimeClass** và Pod chỉ cần khai
   `runtimeClassName`.
4. **Không thấy gì**, vì log không đi qua STDOUT. Bài nêu rõ workload Windows "thường được cấu
   hình để ghi log vào **ETW (Event Tracing for Windows)** hoặc đẩy các mục log vào **event log
   của ứng dụng**", và đó là lý do người dùng từng gặp khó khi thu thập log. Cách được khuyến
   nghị là **LogMonitor** — công cụ mã nguồn mở của Microsoft giám sát event log, ETW provider và
   log ứng dụng tùy chỉnh rồi **chuyển tiếp chúng tới STDOUT để `kubectl logs <pod>` đọc được**.
   Bạn phải chép binary và cấu hình của nó vào container rồi sửa entrypoint.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
