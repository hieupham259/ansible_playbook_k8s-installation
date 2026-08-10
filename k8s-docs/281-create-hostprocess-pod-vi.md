# Tạo một Windows HostProcess Pod (Create a Windows HostProcess Pod)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/configure-pod-container/create-hostprocess-pod/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.26 [stable]`

Windows HostProcess container cho phép bạn chạy các workload đã được đóng gói container trên một
host Windows. Các container này hoạt động như những tiến trình (process) bình thường nhưng có
quyền truy cập vào network namespace, kho lưu trữ và các thiết bị của host khi được cấp đặc
quyền user phù hợp. HostProcess container có thể được dùng để triển khai các network plugin,
cấu hình lưu trữ, device plugin, kube-proxy và các thành phần khác lên các node Windows mà không
cần đến các proxy chuyên dụng hay việc cài đặt trực tiếp các host service.

Các tác vụ quản trị như cài đặt bản vá bảo mật, thu thập event log, và nhiều việc khác có thể
được thực hiện mà không yêu cầu người vận hành cluster phải đăng nhập vào từng node Windows.
HostProcess container có thể chạy dưới bất kỳ user nào tồn tại trên host hoặc thuộc domain của
máy host, cho phép quản trị viên hạn chế quyền truy cập tài nguyên thông qua quyền của user.
Mặc dù cả cách ly filesystem lẫn cách ly tiến trình đều không được hỗ trợ, một volume mới sẽ
được tạo trên host khi container khởi động để cung cấp cho nó một không gian làm việc sạch và
hợp nhất. HostProcess container cũng có thể được build dựa trên các Windows base image sẵn có
và không thừa kế các
[yêu cầu tương thích](https://docs.microsoft.com/virtualization/windowscontainers/deploy-containers/version-compatibility)
giống như Windows server container, nghĩa là phiên bản của base image không cần khớp với phiên
bản của host. Tuy nhiên, bạn nên dùng cùng phiên bản base image với các workload Windows Server
container của mình để đảm bảo không có image không dùng đến chiếm dung lượng trên node.
HostProcess container cũng hỗ trợ [mount volume](#volume-mounts) bên trong volume của container.

### Khi nào tôi nên dùng Windows HostProcess container? (When should I use a Windows HostProcess container?)

- Khi bạn cần thực hiện các tác vụ yêu cầu network namespace của host. HostProcess container có
  quyền truy cập vào các network interface và địa chỉ IP của host.
- Bạn cần truy cập các tài nguyên trên host như filesystem, event log, v.v.
- Cài đặt các device driver hoặc Windows service cụ thể.
- Hợp nhất các tác vụ quản trị và chính sách bảo mật. Điều này làm giảm mức đặc quyền mà các
  node Windows cần có.

## Trước khi bạn bắt đầu (Before you begin)

Hướng dẫn này dành riêng cho Kubernetes v1.36. Nếu bạn không chạy Kubernetes v1.36, hãy xem tài
liệu của đúng phiên bản Kubernetes đó.

Trong Kubernetes 1.36, tính năng HostProcess container được bật theo mặc định. kubelet sẽ giao
tiếp trực tiếp với containerd bằng cách truyền flag hostprocess qua CRI. Bạn có thể dùng phiên
bản containerd mới nhất (v1.6+) để chạy HostProcess container.
[Cách cài đặt containerd.](./00-container-runtimes-vi.md#containerd)

## Các hạn chế (Limitations)

Các hạn chế sau áp dụng cho Kubernetes v1.36:

- HostProcess container yêu cầu container runtime containerd 1.6 trở lên, và containerd 1.7
  được khuyến nghị.
- HostProcess pod chỉ có thể chứa các HostProcess container. Đây là hạn chế hiện tại của hệ điều
  hành Windows; các Windows container không có đặc quyền không thể chia sẻ vNIC với IP namespace
  của host.
- HostProcess container chạy như một tiến trình trên host và không có bất kỳ mức độ cách ly nào
  ngoài các ràng buộc tài nguyên áp lên tài khoản user HostProcess. Cả cách ly filesystem lẫn
  cách ly Hyper-V đều không được hỗ trợ cho HostProcess container.
- Mount volume được hỗ trợ và được mount bên trong volume của container. Xem
  [Mount volume](#volume-mounts)
- Theo mặc định, chỉ một tập hạn chế các tài khoản user của host khả dụng cho HostProcess
  container. Xem [Chọn tài khoản user](#choosing-a-user-account).
- Giới hạn tài nguyên (disk, memory, số lượng CPU) được hỗ trợ theo cùng cách như các tiến trình
  trên host.
- Cả mount Named pipe lẫn Unix domain socket đều **không** được hỗ trợ; thay vào đó, chúng nên
  được truy cập thông qua đường dẫn của chúng trên host (ví dụ \\\\.\\pipe\\\*)

## Yêu cầu cấu hình cho HostProcess Pod (HostProcess Pod configuration requirements)

Để kích hoạt một Windows HostProcess pod, bạn cần đặt đúng các cấu hình trong cấu hình pod
security. Trong số các chính sách được định nghĩa ở
[Pod Security Standards](./115-pod-security-standards-vi.md), HostProcess pod bị cấm bởi các
chính sách baseline và restricted. Vì vậy, khuyến nghị là HostProcess pod nên chạy theo đúng
profile privileged.

Khi chạy dưới chính sách privileged, đây là các cấu hình cần được đặt để cho phép tạo một
HostProcess pod:

<table>
  <caption style="display: none">Đặc tả chính sách privileged</caption>
  <thead>
    <tr>
      <th>Điều khiển (Control)</th>
      <th>Chính sách (Policy)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td style="white-space: nowrap"><a href="./115-pod-security-standards-vi.md"><tt>securityContext.windowsOptions.hostProcess</tt></a></td>
      <td>
        <p>Pod Windows cung cấp khả năng chạy <a href="./281-create-hostprocess-pod-vi.md">
        HostProcess container</a>, cho phép truy cập đặc quyền vào node Windows. </p>
        <p><strong>Giá trị được phép</strong></p>
        <ul>
          <li><code>true</code></li>
        </ul>
      </td>
    </tr>
    <tr>
      <td style="white-space: nowrap"><a href="./115-pod-security-standards-vi.md"><tt>hostNetwork</tt></a></td>
      <td>
        <p>Pod chứa HostProcess container phải sử dụng network namespace của host.</p>
        <p><strong>Giá trị được phép</strong></p>
        <ul>
          <li><code>true</code></li>
        </ul>
      </td>
    </tr>
    <tr>
      <td style="white-space: nowrap"><a href="https://kubernetes.io/docs/tasks/configure-pod-container/configure-runasusername/"><tt>securityContext.windowsOptions.runAsUserName</tt></a></td>
      <td>
        <p>Pod spec bắt buộc phải chỉ định user mà HostProcess container sẽ chạy với tư cách đó.</p>
        <p><strong>Giá trị được phép</strong></p>
        <ul>
          <li><code>NT AUTHORITY\SYSTEM</code></li>
          <li><code>NT AUTHORITY\Local service</code></li>
          <li><code>NT AUTHORITY\NetworkService</code></li>
          <li>Tên các local usergroup (xem bên dưới)</li>
        </ul>
      </td>
    </tr>
    <tr>
      <td style="white-space: nowrap"><a href="./115-pod-security-standards-vi.md"><tt>runAsNonRoot</tt></a></td>
      <td>
        <p>Vì HostProcess container có quyền truy cập đặc quyền vào host, field <tt>runAsNonRoot</tt> không thể được đặt là true.</p>
        <p><strong>Giá trị được phép</strong></p>
        <ul>
          <li>Không định nghĩa/Nil</li>
          <li><code>false</code></li>
        </ul>
      </td>
    </tr>
  </tbody>
</table>

### Ví dụ manifest — trích đoạn (Example manifest - excerpt) {#manifest-example}

```yaml
spec:
  securityContext:
    windowsOptions:
      hostProcess: true
      runAsUserName: "NT AUTHORITY\\Local service"
  hostNetwork: true
  containers:
  - name: test
    image: image1:latest
    command:
      - ping
      - -t
      - 127.0.0.1
  nodeSelector:
    "kubernetes.io/os": windows
```

## Mount volume (Volume mounts) {#volume-mounts}

HostProcess container hỗ trợ khả năng mount volume bên trong không gian volume của container.
Hành vi mount volume khác nhau tùy theo phiên bản containerd runtime mà node đang sử dụng.

### Containerd v1.6

Các ứng dụng chạy bên trong container có thể truy cập trực tiếp các mount volume qua đường dẫn
tương đối hoặc tuyệt đối. Một biến môi trường `$CONTAINER_SANDBOX_MOUNT_POINT` được đặt khi
container được tạo, cung cấp đường dẫn tuyệt đối trên host tới volume của container. Đường dẫn
tương đối được tính dựa trên cấu hình `.spec.containers.volumeMounts.mountPath`.

Ví dụ, để truy cập token của service account, các cấu trúc đường dẫn sau được hỗ trợ bên trong
container:

- `.\var\run\secrets\kubernetes.io\serviceaccount\`
- `$CONTAINER_SANDBOX_MOUNT_POINT\var\run\secrets\kubernetes.io\serviceaccount\`

### Containerd v1.7 trở lên (Containerd v1.7 and greater)

Các ứng dụng chạy bên trong container có thể truy cập trực tiếp các mount volume qua `mountPath`
được chỉ định trong volumeMount (giống như container Linux và Windows container không phải
HostProcess).

Để tương thích ngược, volume cũng có thể được truy cập thông qua các đường dẫn tương đối giống
cách containerd v1.6 cấu hình.

Ví dụ, để truy cập token của service account bên trong container, bạn sẽ dùng một trong các
đường dẫn sau:

- `c:\var\run\secrets\kubernetes.io\serviceaccount`
- `/var/run/secrets/kubernetes.io/serviceaccount/`
- `$CONTAINER_SANDBOX_MOUNT_POINT\var\run\secrets\kubernetes.io\serviceaccount\`

## Giới hạn tài nguyên (Resource limits)

Giới hạn tài nguyên (disk, memory, số lượng CPU) được áp lên job và có hiệu lực trên toàn job.
Ví dụ, với limit 10MB được đặt, lượng memory cấp phát cho bất kỳ HostProcess job object nào sẽ
bị giới hạn ở mức 10MB. Đây là hành vi giống với các loại Windows container khác. Các limit này
được chỉ định theo đúng cách hiện tại của bất kỳ orchestrator hay runtime nào đang được dùng.
Điểm khác biệt duy nhất nằm ở cách tính mức sử dụng tài nguyên disk phục vụ việc theo dõi tài
nguyên, do sự khác biệt trong cách HostProcess container được khởi tạo (bootstrap).

## Chọn tài khoản user (Choosing a user account) {#choosing-a-user-account}

### Tài khoản hệ thống (System accounts)

Theo mặc định, HostProcess container hỗ trợ khả năng chạy dưới một trong ba tài khoản Windows
service được hỗ trợ:

- **[LocalSystem](https://docs.microsoft.com/windows/win32/services/localsystem-account)**
- **[LocalService](https://docs.microsoft.com/windows/win32/services/localservice-account)**
- **[NetworkService](https://docs.microsoft.com/windows/win32/services/networkservice-account)**

Bạn nên chọn một tài khoản Windows service phù hợp cho từng HostProcess container, với mục tiêu
giới hạn mức đặc quyền để tránh gây hư hại vô tình (hoặc thậm chí có chủ đích) cho host. Tài
khoản service LocalSystem có mức đặc quyền cao nhất trong ba tài khoản và chỉ nên được dùng khi
thực sự cần thiết. Khi có thể, hãy dùng tài khoản service LocalService vì nó là tài khoản ít
đặc quyền nhất trong ba lựa chọn.

### Tài khoản cục bộ (Local accounts) {#local-accounts}

Nếu được cấu hình, HostProcess container cũng có thể chạy dưới các tài khoản user cục bộ (local
user account), cho phép người vận hành node cấp quyền truy cập chi tiết (fine-grained) cho các
workload.

Để chạy HostProcess container dưới một user cục bộ: trước tiên một local usergroup phải được
tạo trên node và tên của local usergroup đó phải được chỉ định trong field `runAsUserName` của
deployment. Trước khi khởi tạo HostProcess container, một tài khoản user cục bộ **tạm thời
(ephemeral)** mới sẽ được tạo và tham gia vào usergroup đã chỉ định, và container sẽ chạy từ tài
khoản này. Cách làm này mang lại nhiều lợi ích, bao gồm việc loại bỏ nhu cầu quản lý mật khẩu
cho các tài khoản user cục bộ. Một HostProcess container ban đầu chạy dưới một service account
có thể được dùng để chuẩn bị các user group cho các HostProcess container về sau.

> **Ghi chú:** Chạy HostProcess container dưới tài khoản user cục bộ yêu cầu containerd v1.7+

Ví dụ:

1. Tạo một local user group trên node (việc này có thể được thực hiện trong một HostProcess
   container khác).

    ```cmd
    net localgroup hpc-localgroup /add
    ```

1. Cấp quyền truy cập các tài nguyên mong muốn trên node cho local usergroup đó. Việc này có thể
   được thực hiện bằng các công cụ như
   [icacls](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/icacls).

1. Đặt `runAsUserName` bằng tên của local usergroup cho pod hoặc cho từng container riêng lẻ.

    ```yaml
    securityContext:
      windowsOptions:
        hostProcess: true
        runAsUserName: hpc-localgroup
    ```

1. Lập lịch (schedule) cho pod!

## Base Image cho HostProcess Container (Base Image for HostProcess Containers)

HostProcess container có thể được build từ bất kỳ
[Windows Container base image](https://learn.microsoft.com/virtualization/windowscontainers/manage-containers/container-base-images)
sẵn có nào.

Ngoài ra, một base image mới đã được tạo riêng cho HostProcess container! Để biết thêm thông
tin, hãy xem
[dự án github windows-host-process-containers-base-image](https://github.com/microsoft/windows-host-process-containers-base-image#overview).

## Khắc phục sự cố HostProcess container (Troubleshooting HostProcess containers)

- HostProcess container không khởi động được với lỗi
  `failed to create user process token: failed to logon user: Access is denied.: unknown`

  Hãy đảm bảo containerd đang chạy dưới tài khoản service `LocalSystem` hoặc `LocalService`.
  Các tài khoản user (kể cả tài khoản Administrator) không có quyền tạo logon token cho bất kỳ
  [tài khoản user](#choosing-a-user-account) được hỗ trợ nào.
