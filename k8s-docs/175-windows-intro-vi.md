# Windows containers trong Kubernetes (Windows containers in Kubernetes)

> Bản dịch tiếng Việt của trang: https://kubernetes.io/docs/concepts/windows/intro/

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 15](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows),
bài 2/7 · Kiểm chứng ở Lab 15 (tùy chọn, chưa viết, xem
[bản đồ lab](labs/README.md#4-bản-đồ-lab)).

**Lộ trình ghi rõ: bỏ qua hoàn toàn giai đoạn 15 nếu cluster của bạn chỉ có Linux.** Cluster lab
ba VM Ubuntu của bạn không có node Windows, nên toàn bộ bài này là kiến thức đối chiếu, không
kiểm chứng được trên cluster hiện tại.

Đây là **bài xương sống của giai đoạn**. Lộ trình chỉ định trọng tâm rất hẹp: **những gì trên
Windows KHÔNG có tương đương Linux**. Đọc theo hướng đó — mỗi lần bài viết "không khả thi",
"chưa được triển khai", "không được hỗ trợ" là một dòng đáng nhớ; phần liệt kê tương thích thì
chỉ cần biết là "gần như giống Linux".

**Phải hiểu ở lần đọc này:**

- Ranh giới nền tảng: **control plane chỉ chạy trên Linux**, Windows chỉ làm worker; node phải
  là **Windows Server 2022 hoặc 2025**; Kubernetes **chỉ hỗ trợ cách ly tiến trình**, không hỗ
  trợ cách ly Hyper-V. Và **không trộn container Windows với container Linux trong cùng một
  Pod** — mọi container trong Pod lập lịch lên cùng một Node, mà mỗi Node là một nền tảng cụ thể.
- `.spec.os.name: windows` **khóa cứng một danh sách trường**: `hostPID`, `hostIPC`,
  `seLinuxOptions`, `seccompProfile`, `fsGroup`, `sysctls`, `shareProcessNamespace`, `runAsUser`,
  `runAsGroup`, `capabilities`, `readOnlyRootFilesystem`, `privileged`… Đặt bất kỳ trường nào
  trong số đó thì **API server không chấp nhận Pod**.
- Bốn khác biệt khái niệm ở mục *Tương thích API*: **danh tính** (Linux dùng UID/GID số nguyên,
  Windows dùng SID nhị phân trong cơ sở dữ liệu SAM, và **SAM không chia sẻ giữa host và
  container**), **quyền file** (ACL theo SID so với bitmask POSIX), **đường dẫn** (`\` thay `/`),
  **tín hiệu** (WM_CLOSE / Control Handler / Service Control Handler thay cho mô hình signal).
- Danh sách không có trên Windows: **HugePages**, **privileged container** (thay bằng HostProcess
  container), **hostNetwork**, `hostIPC`/`hostPID`, `shareProcessNamespace`, `volumeDevices`
  (thiết bị block thô), `emptyDir` với nguồn `memory`, `mountPropagation`, filesystem `/proc`,
  `readOnlyRootFilesystem`. Trong `securityContext` của Pod, **chỉ `runAsNonRoot` và
  `windowsOptions` là hoạt động**.
- Kubelet trên Windows khác Linux: **không có ràng buộc bộ nhớ hay CPU**, `--kube-reserved` và
  `--system-reserved` **chỉ trừ vào NodeAllocatable chứ không bảo đảm tài nguyên**;
  `--enforce-node-allocable` và condition `PIDPressure` **chưa được triển khai**; kubelet
  **không thực hiện eviction do OOM**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Pause container* — image `pause:3.6` và bản Microsoft ký authenticode | là chi tiết vận hành khi dựng node Windows | khi môi trường thực sự có node Windows |
| *Các container runtime* — ContainerD, Mirantis Container Runtime | chọn runtime cho node Windows | khi môi trường thực sự có node Windows |
| *Khuyến nghị và lưu ý về phần cứng*, kích thước image và ổ `C:` ảo 20GB | dùng khi mua máy và tính dung lượng | khi môi trường thực sự có node Windows |
| *Nhận trợ giúp và khắc phục sự cố*, *Windows Operational Readiness*, *Trình phát hiện sự cố node* | là quy trình nghiệm thu và báo lỗi | khi môi trường thực sự có node Windows |
| *Công cụ triển khai*, *Các kênh phân phối Windows* | thông tin tham chiếu về vòng đời sản phẩm | khi môi trường thực sự có node Windows |
| *Truy cập mạng của host* — feature gate `WindowsHostNetwork` | Kubernetes v1.36 đã **không** còn feature gate này | không cần |

---

Các ứng dụng Windows chiếm một phần lớn trong số các dịch vụ và ứng dụng đang chạy tại nhiều tổ chức. [Windows containers](https://aka.ms/windowscontainers) cung cấp một cách để đóng gói các tiến trình (process) cùng các phụ thuộc (dependency) của gói phần mềm, giúp việc áp dụng các thực hành DevOps và tuân theo các mẫu hình cloud native cho ứng dụng Windows trở nên dễ dàng hơn.

Các tổ chức đã đầu tư vào cả ứng dụng chạy trên Windows lẫn ứng dụng chạy trên Linux không cần phải tìm những bộ điều phối (orchestrator) riêng biệt để quản lý workload của mình, nhờ đó tăng hiệu quả vận hành trên toàn bộ các bản triển khai, bất kể hệ điều hành nào.

## Node Windows trong Kubernetes (Windows nodes in Kubernetes)

Để cho phép điều phối Windows container trong Kubernetes, hãy thêm node Windows vào cluster Linux hiện có của bạn. Việc lập lịch (scheduling) Windows container trong Pod trên Kubernetes tương tự như việc lập lịch các container chạy Linux.

Để chạy được Windows container, cluster Kubernetes của bạn phải bao gồm nhiều hệ điều hành. Mặc dù bạn chỉ có thể chạy control plane trên Linux, bạn có thể triển khai các worker node chạy Windows hoặc Linux.

Node Windows được [hỗ trợ](#windows-os-version-support) với điều kiện hệ điều hành là Windows Server 2022 hoặc Windows Server 2025.

Tài liệu này dùng thuật ngữ *Windows container* để chỉ các Windows container dùng cơ chế cách ly tiến trình (process isolation). Kubernetes không hỗ trợ chạy Windows container với [cách ly Hyper-V](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container).

## Tương thích và các hạn chế (Compatibility and limitations) {#limitations}

Một số tính năng của node chỉ khả dụng nếu bạn sử dụng một [container runtime](#container-runtime) cụ thể; số khác không khả dụng trên node Windows, bao gồm:

* HugePages: không được hỗ trợ cho Windows container
* Container đặc quyền (privileged container): không được hỗ trợ cho Windows container.
  [HostProcess Containers](281-create-hostprocess-pod-vi.md) cung cấp chức năng tương tự.
* TerminationGracePeriod: yêu cầu containerD

Không phải mọi tính năng của shared namespace đều được hỗ trợ. Xem [Tương thích API](#api) để biết thêm chi tiết.

Xem [Tương thích phiên bản hệ điều hành Windows](#windows-os-version-support) để biết chi tiết về các phiên bản Windows mà Kubernetes được kiểm thử.

Từ góc độ API và kubectl, Windows container hoạt động gần như giống hệt các container chạy Linux. Tuy nhiên, có một số khác biệt đáng chú ý trong các chức năng chính, được trình bày trong mục này.

### So sánh với Linux (Comparison with Linux) {#compatibility-linux-similarities}

Các thành phần chính của Kubernetes hoạt động trên Windows theo cùng cách như trên Linux. Mục này đề cập đến một số khái niệm trừu tượng (abstraction) workload quan trọng và cách chúng ánh xạ sang Windows.

* [Pod](46-pods-vi.md)

  Pod là khối dựng cơ bản của Kubernetes – đơn vị nhỏ nhất và đơn giản nhất trong mô hình đối tượng Kubernetes mà bạn tạo hoặc triển khai. Bạn không thể triển khai container Windows và container Linux trong cùng một Pod. Mọi container trong một Pod được lập lịch lên cùng một Node, trong đó mỗi Node đại diện cho một nền tảng và kiến trúc cụ thể. Các khả năng, thuộc tính và sự kiện sau của Pod được hỗ trợ với Windows container:

  * Một hoặc nhiều container trong mỗi Pod, với cách ly tiến trình và chia sẻ volume
  * Các trường `status` của Pod
  * Readiness, liveness và startup probe
  * Các hook vòng đời container postStart & preStop
  * ConfigMap, Secret: dưới dạng biến môi trường hoặc volume
  * Volume `emptyDir`
  * Mount named pipe từ host
  * Giới hạn tài nguyên (resource limits)
  * Trường OS:

    Trường `.spec.os.name` nên được đặt là `windows` để chỉ ra rằng Pod hiện tại sử dụng Windows container.

    Nếu bạn đặt trường `.spec.os.name` là `windows`, bạn không được đặt các trường sau trong `.spec` của Pod đó:

    * `spec.hostPID`
    * `spec.hostIPC`
    * `spec.securityContext.seLinuxOptions`
    * `spec.securityContext.seccompProfile`
    * `spec.securityContext.fsGroup`
    * `spec.securityContext.fsGroupChangePolicy`
    * `spec.securityContext.sysctls`
    * `spec.shareProcessNamespace`
    * `spec.securityContext.runAsUser`
    * `spec.securityContext.runAsGroup`
    * `spec.securityContext.supplementalGroups`
    * `spec.containers[*].securityContext.seLinuxOptions`
    * `spec.containers[*].securityContext.seccompProfile`
    * `spec.containers[*].securityContext.capabilities`
    * `spec.containers[*].securityContext.readOnlyRootFilesystem`
    * `spec.containers[*].securityContext.privileged`
    * `spec.containers[*].securityContext.allowPrivilegeEscalation`
    * `spec.containers[*].securityContext.procMount`
    * `spec.containers[*].securityContext.runAsUser`
    * `spec.containers[*].securityContext.runAsGroup`

    Trong danh sách trên, ký tự đại diện (`*`) biểu thị mọi phần tử trong một danh sách. Ví dụ, `spec.containers[*].securityContext` chỉ đối tượng SecurityContext của tất cả các container. Nếu bất kỳ trường nào trong số này được chỉ định, Pod sẽ không được API server chấp nhận.

* [Các workload resource](62-controllers-index-vi.md) bao gồm:
  * ReplicaSet
  * Deployment
  * StatefulSet
  * DaemonSet
  * Job
  * CronJob
  * ReplicationController
* Service
  Xem [Cân bằng tải và Service](89-windows-networking-vi.md#load-balancing-and-services) để biết thêm chi tiết (đã có bản dịch: [Mạng trên Windows](89-windows-networking-vi.md)).

Pod, các workload resource và Service là những thành phần thiết yếu để quản lý workload Windows trên Kubernetes. Tuy nhiên, chỉ riêng chúng thì chưa đủ để quản lý đúng đắn vòng đời của workload Windows trong một môi trường cloud native năng động. Các tính năng sau cũng được hỗ trợ:

* `kubectl exec`
* Số liệu (metrics) của Pod và container
* Horizontal pod autoscaling (tự động co giãn Pod theo chiều ngang)
* Resource quota (hạn ngạch tài nguyên)
* Scheduler preemption (cơ chế chiếm chỗ của scheduler)

### Các tùy chọn dòng lệnh cho kubelet (Command line options for the kubelet) {#kubelet-compatibility}

Một số tùy chọn dòng lệnh của kubelet hoạt động khác trên Windows, như mô tả dưới đây:

* Tùy chọn `--windows-priorityclass` cho phép bạn đặt mức ưu tiên lập lịch cho tiến trình kubelet
  (xem [Quản lý tài nguyên CPU](112-windows-resource-management-vi.md#resource-management-cpu))
* Các cờ `--kube-reserved`, `--system-reserved` và `--eviction-hard` cập nhật
  [NodeAllocatable](253-reserve-compute-resources-vi.md#node-allocatable)
* Việc thu hồi (eviction) bằng `--enforce-node-allocable` chưa được triển khai
* Khi chạy trên node Windows, kubelet không có các ràng buộc về bộ nhớ hay CPU.
  `--kube-reserved` và `--system-reserved` chỉ trừ vào `NodeAllocatable`
  và không bảo đảm tài nguyên cung cấp cho workload.
  Xem [Quản lý tài nguyên cho node Windows](112-windows-resource-management-vi.md)
  để biết thêm thông tin.
* Điều kiện (Condition) `PIDPressure` chưa được triển khai
* Kubelet không thực hiện các hành động thu hồi do OOM (OOM eviction)

### Tương thích API (API compatibility) {#api}

Có những khác biệt tinh tế trong cách các API của Kubernetes hoạt động đối với Windows do khác biệt về hệ điều hành và container runtime. Một số thuộc tính của workload được thiết kế cho Linux và không chạy được trên Windows.

Ở mức khái quát, các khái niệm hệ điều hành sau đây là khác nhau:

* Danh tính (Identity) - Linux sử dụng userID (UID) và groupID (GID), được biểu diễn
  dưới dạng số nguyên. Tên người dùng và tên nhóm không có tính chính danh
  (canonical) - chúng chỉ là bí danh trong `/etc/groups` hoặc `/etc/passwd`
  trỏ ngược về UID+GID. Windows sử dụng một
  [định danh bảo mật](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/security-identifiers) (SID)
  dạng nhị phân lớn hơn, được lưu trong cơ sở dữ liệu Windows Security Access Manager (SAM). Cơ sở dữ liệu này
  không được chia sẻ giữa host và các container, hay giữa các container với nhau.
* Quyền trên file (File permissions) - Windows sử dụng danh sách kiểm soát truy cập (access control list) dựa trên SID, trong khi
  các hệ thống POSIX như Linux sử dụng bitmask dựa trên quyền của đối tượng và UID+GID,
  cộng thêm danh sách kiểm soát truy cập _tùy chọn_.
* Đường dẫn file (File paths) - quy ước trên Windows là dùng `\` thay vì `/`. Các thư viện IO của Go
  thường chấp nhận cả hai và tự xử lý, nhưng khi bạn thiết lập một đường dẫn
  hoặc dòng lệnh được thông dịch bên trong container, có thể cần dùng `\`.
* Tín hiệu (Signals) - các ứng dụng tương tác trên Windows xử lý việc kết thúc theo cách khác, và có thể
  triển khai một hoặc nhiều cơ chế sau:
  * Một luồng UI xử lý các thông điệp được định nghĩa rõ, bao gồm `WM_CLOSE`.
  * Các ứng dụng console xử lý Ctrl-C hoặc Ctrl-break bằng một Control Handler.
  * Các service đăng ký một hàm Service Control Handler có thể nhận
    các mã điều khiển `SERVICE_CONTROL_STOP`.

Mã thoát (exit code) của container tuân theo cùng quy ước: 0 là thành công, khác 0 là thất bại.
Các mã lỗi cụ thể có thể khác nhau giữa Windows và Linux. Tuy nhiên, các exit code
truyền từ các thành phần Kubernetes (kubelet, kube-proxy) không thay đổi.

#### Tương thích các trường trong đặc tả container (Field compatibility for container specifications) {#compatibility-v1-pod-spec-containers}

Danh sách sau ghi lại các khác biệt trong cách đặc tả container của Pod hoạt động giữa Windows và Linux:

* Huge page chưa được triển khai trong container runtime trên Windows
  và không khả dụng. Chúng yêu cầu [khai báo một đặc quyền người dùng](https://docs.microsoft.com/en-us/windows/desktop/Memory/large-page-support)
  vốn không thể cấu hình được cho container.
* `requests.cpu` và `requests.memory` - các request được trừ vào
  lượng tài nguyên khả dụng của node, vì vậy chúng có thể dùng để tránh cấp phát quá mức (overprovision)
  cho một node. Tuy nhiên, chúng không thể dùng để bảo đảm tài nguyên trên một node đã bị cấp phát quá mức.
  Chúng nên được áp dụng cho tất cả container như một thực hành tốt nếu người vận hành
  muốn tránh hoàn toàn việc cấp phát quá mức.
* `securityContext.allowPrivilegeEscalation` -
   không khả thi trên Windows; không có capability nào được kết nối
* `securityContext.capabilities` -
   các POSIX capability chưa được triển khai trên Windows
* `securityContext.privileged` -
   Windows không hỗ trợ container đặc quyền, hãy dùng [HostProcess Containers](281-create-hostprocess-pod-vi.md) thay thế
* `securityContext.procMount` -
   Windows không có filesystem `/proc`
* `securityContext.readOnlyRootFilesystem` -
   không khả thi trên Windows; quyền ghi là bắt buộc để registry và các tiến trình
   hệ thống có thể chạy bên trong container
* `securityContext.runAsGroup` -
   không khả thi trên Windows vì không có hỗ trợ GID
* `securityContext.runAsNonRoot` -
   thiết lập này sẽ ngăn container chạy dưới danh nghĩa `ContainerAdministrator`,
   vốn là thứ gần nhất tương đương với người dùng root trên Windows.
* `securityContext.runAsUser` -
   hãy dùng [`runAsUserName`](278-configure-runasusername-vi.md)
   thay thế
* `securityContext.seLinuxOptions` -
   không khả thi trên Windows vì SELinux là đặc thù của Linux
* `terminationMessagePath` -
   có một số hạn chế do Windows không hỗ trợ ánh xạ (map) các file đơn lẻ. Giá trị
   mặc định là `/dev/termination-log`, và nó vẫn hoạt động vì đường dẫn này mặc định
   không tồn tại trên Windows.

#### Tương thích các trường trong đặc tả Pod (Field compatibility for Pod specifications) {#compatibility-v1-pod}

Danh sách sau ghi lại các khác biệt trong cách đặc tả Pod hoạt động giữa Windows và Linux:

* `hostIPC` và `hostpid` - chia sẻ namespace của host là không khả thi trên Windows
* `hostNetwork` - dùng mạng của host là không khả thi trên Windows
* `dnsPolicy` - đặt `dnsPolicy` của Pod thành `ClusterFirstWithHostNet`
   không được hỗ trợ trên Windows vì không có mạng host. Pod luôn
   chạy với mạng container.
* `podSecurityContext` [xem bên dưới](#compatibility-v1-pod-spec-containers-securitycontext)
* `shareProcessNamespace` - đây là tính năng beta và phụ thuộc vào các namespace của Linux,
  vốn chưa được triển khai trên Windows. Windows không thể chia sẻ namespace tiến trình hay
  filesystem gốc của container. Chỉ có mạng là có thể chia sẻ.
* `terminationGracePeriodSeconds` - tính năng này chưa được triển khai đầy đủ trong Docker trên Windows,
  xem [issue trên GitHub](https://github.com/moby/moby/issues/25982).
  Hành vi hiện tại là tiến trình ENTRYPOINT sẽ được gửi CTRL_SHUTDOWN_EVENT,
  sau đó Windows chờ 5 giây theo mặc định, và cuối cùng tắt
  tất cả tiến trình theo hành vi tắt máy thông thường của Windows. Giá trị mặc định
  5 giây thực chất nằm trong Windows registry
  [bên trong container](https://github.com/moby/moby/issues/25982#issuecomment-426441183),
  nên có thể ghi đè khi build container image.
* `volumeDevices` - đây là tính năng beta và chưa được triển khai trên Windows.
  Windows không thể gắn (attach) các thiết bị block thô (raw block device) vào Pod.
* `volumes`
  * Nếu bạn định nghĩa một volume `emptyDir`, bạn không thể đặt nguồn volume của nó là `memory`.
* Bạn không thể bật `mountPropagation` cho các mount của volume vì tính năng này không
  được hỗ trợ trên Windows.

#### Truy cập mạng của host (Host network access) {#compatibility-v1-pod-sec-containers-hostnetwork}

Kubernetes v1.26 đến v1.32 từng bao gồm hỗ trợ alpha cho việc chạy Pod Windows trong network namespace của host.

Kubernetes v1.36 **không** bao gồm feature gate `WindowsHostNetwork`
cũng như hỗ trợ chạy Pod Windows trong network namespace của host.

#### Tương thích các trường trong security context của Pod (Field compatibility for Pod security context) {#compatibility-v1-pod-spec-containers-securitycontext}

Chỉ có `securityContext.runAsNonRoot` và `securityContext.windowsOptions` trong các trường
[`securityContext`](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/#security-context) của Pod là hoạt động trên Windows.

## Trình phát hiện sự cố node (Node problem detector)

Trình phát hiện sự cố node (node problem detector, xem
[Giám sát sức khỏe node](310-monitor-node-health-vi.md))
có hỗ trợ sơ bộ cho Windows.
Để biết thêm thông tin, hãy truy cập [trang GitHub của dự án](https://github.com/kubernetes/node-problem-detector#windows).

## Pause container

Trong một Pod Kubernetes, một container hạ tầng hay còn gọi là "pause" container được tạo ra
trước tiên để chứa container. Trong Linux, các cgroup và namespace tạo nên một Pod
cần một tiến trình để duy trì sự tồn tại liên tục của chúng; tiến trình pause đảm nhiệm
việc này. Các container thuộc cùng một Pod, bao gồm container hạ tầng và các worker
container, chia sẻ chung một điểm cuối mạng (cùng địa chỉ IPv4 và/hoặc IPv6, cùng
không gian port mạng). Kubernetes sử dụng pause container để cho phép các worker container
gặp sự cố (crash) hoặc khởi động lại mà không mất bất kỳ cấu hình mạng nào.

Kubernetes duy trì một image đa kiến trúc (multi-architecture) có hỗ trợ Windows.
Với Kubernetes v1.36, pause image được khuyến nghị là `registry.k8s.io/pause:3.6`.
[Mã nguồn](https://github.com/kubernetes/kubernetes/tree/master/build/pause)
có sẵn trên GitHub.

Microsoft duy trì một image đa kiến trúc khác, hỗ trợ Linux và Windows
amd64, mà bạn có thể tìm thấy dưới tên `mcr.microsoft.com/oss/kubernetes/pause:3.6`.
Image này được build từ cùng mã nguồn với image do Kubernetes duy trì, nhưng
tất cả các file nhị phân Windows đều được Microsoft [ký authenticode](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/authenticode).
Dự án Kubernetes khuyến nghị sử dụng image do Microsoft duy trì nếu bạn
triển khai vào môi trường production hoặc môi trường tương tự production có yêu cầu
các file nhị phân được ký.

## Các container runtime (Container runtimes) {#container-runtime}

Bạn cần cài đặt một container runtime
vào mỗi node trong cluster để Pod có thể chạy trên đó.

Các container runtime sau hoạt động được với Windows:

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

### ContainerD

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.20 [stable]`

Bạn có thể sử dụng ContainerD 1.4.0+
làm container runtime cho các node Kubernetes chạy Windows.

Tìm hiểu cách [cài đặt ContainerD trên node Windows](00-container-runtimes-vi.md#containerd).

> **Ghi chú:** Có một [hạn chế đã biết](https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa#gmsa-limitations)
> khi dùng GMSA cùng containerd để truy cập các chia sẻ mạng (network share) của Windows, đòi hỏi
> một bản vá kernel.

### Mirantis Container Runtime {#mcr}

[Mirantis Container Runtime](https://docs.mirantis.com/mcr/25.0/overview.html) (MCR)
khả dụng dưới dạng container runtime cho tất cả các phiên bản Windows Server 2019 trở lên.

Xem [Cài đặt MCR trên Windows Server](https://docs.mirantis.com/mcr/25.0/install/mcr-windows.html) để biết thêm thông tin.

## Tương thích phiên bản hệ điều hành Windows (Windows OS version compatibility) {#windows-os-version-support}

Trên các node Windows, các quy tắc tương thích nghiêm ngặt được áp dụng: phiên bản hệ điều hành của host phải
khớp với phiên bản hệ điều hành của image cơ sở (base image) của container.

Với Kubernetes v1.36, khả năng tương thích hệ điều hành cho các node Windows (và Pod)
như sau:

Bản phát hành Windows Server LTSC
: Windows Server 2022
: Windows Server 2025

[Chính sách chênh lệch phiên bản (version-skew policy)](https://kubernetes.io/docs/setup/release/version-skew-policy/) của Kubernetes cũng được áp dụng.

## Khuyến nghị và lưu ý về phần cứng (Hardware recommendations and considerations) {#windows-hardware-recommendations}

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

> **Ghi chú:** Các thông số phần cứng nêu ở đây nên được xem là các giá trị mặc định hợp lý.
> Chúng không nhằm thể hiện yêu cầu tối thiểu hay khuyến nghị cụ thể cho môi trường production.
> Tùy vào yêu cầu của workload, các giá trị này có thể cần được điều chỉnh.

- Bộ xử lý 64-bit với 4 nhân CPU trở lên, có khả năng hỗ trợ ảo hóa
- 8GB RAM trở lên
- 50GB dung lượng đĩa trống trở lên

Tham khảo
[tài liệu Microsoft về yêu cầu phần cứng cho Windows Server](https://learn.microsoft.com/en-us/windows-server/get-started/hardware-requirements)
để có thông tin mới nhất về yêu cầu phần cứng tối thiểu. Để có hướng dẫn về việc quyết định tài nguyên cho
các worker node production, hãy tham khảo [tài liệu Kubernetes về worker node production](https://kubernetes.io/docs/setup/production-environment/#production-worker-nodes).

Để tối ưu tài nguyên hệ thống, nếu không cần giao diện đồ họa,
có thể ưu tiên dùng bản cài đặt Windows Server không bao gồm
tùy chọn cài đặt [Windows Desktop Experience](https://learn.microsoft.com/en-us/windows-server/get-started/install-options-server-core-desktop-experience),
vì cấu hình này thường giải phóng được nhiều tài nguyên
hệ thống hơn.

Khi đánh giá dung lượng đĩa cho các worker node Windows, hãy lưu ý rằng các Windows container image thường lớn hơn
các Linux container image, với kích thước image dao động
từ [300MB đến hơn 10GB](https://techcommunity.microsoft.com/t5/containers/nano-server-x-server-core-x-server-which-base-image-is-the-right/ba-p/2835785)
cho một image đơn lẻ. Ngoài ra, lưu ý rằng ổ `C:` trong Windows container mặc định thể hiện dung lượng trống ảo là
20GB; đây không phải là dung lượng thực sự được sử dụng, mà là kích thước đĩa mà một container đơn lẻ có thể tăng trưởng
để chiếm dụng khi dùng lưu trữ cục bộ (local storage) trên host.
Xem [Containers trên Windows - Tài liệu về lưu trữ container](https://learn.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/container-storage#storage-limits)
để biết thêm chi tiết.

## Nhận trợ giúp và khắc phục sự cố (Getting help and troubleshooting) {#troubleshooting}

Nguồn trợ giúp chính khi khắc phục sự cố cluster Kubernetes của bạn nên bắt đầu
từ trang [Khắc phục sự cố (Troubleshooting)](296-debug-vi.md).

Một số trợ giúp khắc phục sự cố bổ sung, dành riêng cho Windows, được trình bày
trong mục này. Log là một yếu tố quan trọng khi khắc phục
sự cố trong Kubernetes. Hãy chắc chắn đính kèm chúng mỗi khi bạn nhờ
các contributor khác hỗ trợ khắc phục sự cố. Hãy làm theo
hướng dẫn trong
[hướng dẫn đóng góp của SIG Windows về việc thu thập log](https://github.com/kubernetes/community/blob/main/sig-windows/CONTRIBUTING.md#gathering-logs).

### Báo cáo lỗi và yêu cầu tính năng (Reporting issues and feature requests)

Nếu bạn gặp thứ trông giống một lỗi (bug), hoặc bạn muốn
đưa ra yêu cầu tính năng, hãy làm theo [hướng dẫn đóng góp của SIG Windows](https://github.com/kubernetes/community/blob/main/sig-windows/CONTRIBUTING.md#reporting-issues-and-feature-requests) để tạo một issue mới.
Trước tiên bạn nên tìm kiếm trong danh sách issue phòng trường hợp vấn đề
đã được báo cáo trước đó, và bình luận với trải nghiệm của bạn trên issue đó kèm thêm
log bổ sung. Kênh SIG Windows trên Slack của Kubernetes cũng là một nơi tuyệt vời để nhận hỗ trợ ban đầu và
các ý tưởng khắc phục sự cố trước khi tạo ticket.

### Kiểm chứng khả năng vận hành của cluster Windows (Validating the Windows cluster operability)

Dự án Kubernetes cung cấp đặc tả _Windows Operational Readiness_ (mức độ sẵn sàng vận hành của Windows),
đi kèm một bộ kiểm thử có cấu trúc. Bộ kiểm thử này được chia thành hai nhóm,
cốt lõi (core) và mở rộng (extended), mỗi nhóm chứa các hạng mục nhằm kiểm thử những lĩnh vực cụ thể.
Nó có thể được dùng để kiểm chứng toàn bộ các chức năng của một hệ thống Windows hoặc hệ thống lai
(kết hợp với node Linux) với độ bao phủ đầy đủ.

Để thiết lập dự án trên một cluster mới tạo, hãy tham khảo hướng dẫn trong
[tài liệu của dự án](https://github.com/kubernetes-sigs/windows-operational-readiness/blob/main/README.md).

## Công cụ triển khai (Deployment tools)

Công cụ kubeadm giúp bạn triển khai một cluster Kubernetes, cung cấp control
plane để quản lý cluster, và các node để chạy workload của bạn.

Dự án [cluster API](https://cluster-api.sigs.k8s.io/) của Kubernetes cũng cung cấp phương tiện để tự động hóa việc triển khai các node Windows.

## Các kênh phân phối Windows (Windows distribution channels)

Để có giải thích chi tiết về các kênh phân phối Windows, hãy xem
[tài liệu của Microsoft](https://docs.microsoft.com/en-us/windows-server/get-started-19/servicing-channels-19).

Thông tin về các kênh phục vụ (servicing channel) khác nhau của Windows Server,
bao gồm cả mô hình hỗ trợ của chúng, có thể tìm thấy tại
[Các kênh phục vụ của Windows Server](https://docs.microsoft.com/en-us/windows-server/get-started/servicing-channels-comparison).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 15:

1. Câu bẫy: bạn lấy một Pod đang chạy tốt trên node Ubuntu, đổi `.spec.os.name` thành `windows`
   và giữ nguyên `spec.securityContext.runAsUser: 1000`. Chuyện gì xảy ra, và vì sao Windows
   không có tương đương cho `runAsUser`?
2. Trên node Ubuntu của cluster lab, một container vượt memory limit bị kernel OOM kill. Trên
   node Windows, kubelet xử lý tình huống tương đương thế nào?
3. Bạn cần một container chạy đặc quyền để cấu hình mạng ở mức host. Trên Linux bạn đặt
   `securityContext.privileged: true`. Trên Windows bài trả lời ra sao và đưa ra thay thế nào?
4. Bốn khác biệt khái niệm hệ điều hành bài nêu là gì? Với "danh tính", vì sao việc SAM **không
   được chia sẻ** giữa host và container lại kéo theo hệ quả ở quyền file?
5. Bạn muốn một Pod gồm một container Linux làm sidecar cho một container Windows. Bài trả lời
   thế nào, và lý do là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **API server từ chối Pod.** Bài liệt kê `spec.securityContext.runAsUser` trong danh sách các
   trường **không được đặt** khi `.spec.os.name` là `windows`, và kết luận: "Nếu bất kỳ trường
   nào trong số này được chỉ định, Pod sẽ **không được API server chấp nhận**." Nguyên nhân gốc
   nằm ở mục *Tương thích API*: Linux định danh bằng **UID/GID số nguyên**, còn Windows dùng
   **SID nhị phân lớn hơn** lưu trong cơ sở dữ liệu SAM — không có con số nào để điền vào
   `runAsUser`. Bài chỉ sang [`runAsUserName`](278-configure-runasusername-vi.md)
   làm thay thế. Đây là bẫy điển hình: `.spec.os.name` không phải một nhãn mô tả vô hại, nó
   **thay đổi tập trường hợp lệ của Pod**.
2. Bài nói thẳng: **"Kubelet không thực hiện các hành động thu hồi do OOM (OOM eviction)"** trên
   node Windows, và **eviction bằng `--enforce-node-allocable` chưa được triển khai**; kubelet
   trên Windows cũng "không có các ràng buộc về bộ nhớ hay CPU", còn `--kube-reserved` và
   `--system-reserved` **chỉ trừ vào `NodeAllocatable`** chứ không bảo đảm tài nguyên cho
   workload. Nghĩa là cơ chế bảo vệ node bằng cách giết tiến trình mà bạn quen trên Linux
   **không tồn tại**. Cơ chế thay thế của hệ điều hành được nói kỹ ở bài
   [112](112-windows-resource-management-vi.md).
3. **Windows không hỗ trợ container đặc quyền.** Bài nêu điều này hai lần: trong mục *Tương thích
   và các hạn chế* và ở dòng `securityContext.privileged`. Thay thế là **HostProcess Container**,
   thứ mà bài mô tả là "cung cấp chức năng tương tự". Lưu ý cách diễn đạt: *tương tự*, không phải
   *tương đương* — và `securityContext.privileged` vẫn nằm trong danh sách trường bị cấm khi
   `.spec.os.name` là `windows`.
4. **Danh tính, quyền trên file, đường dẫn file, tín hiệu.** Với danh tính: Windows lưu SID trong
   cơ sở dữ liệu **Security Access Manager (SAM)**, và cơ sở dữ liệu này "**không được chia sẻ
   giữa host và các container, hay giữa các container với nhau**". Quyền file trên Windows lại là
   **danh sách kiểm soát truy cập dựa trên SID** — nên nếu SID ở hai bên không cùng một không
   gian định danh thì **không có ánh xạ quyền nào giữa host và container**. Đây chính là gốc rễ
   của các hạn chế lưu trữ ở bài [106](106-windows-storage-vi.md) và của việc không có
   `fsGroup`/`runAsGroup`.
5. **Không được.** Bài viết: "Bạn **không thể** triển khai container Windows và container Linux
   trong cùng một Pod." Lý do nêu ngay sau đó: **mọi container trong một Pod được lập lịch lên
   cùng một Node**, mà mỗi Node "đại diện cho một nền tảng và kiến trúc cụ thể". Không phải hạn
   chế tạm thời của một tính năng, mà là hệ quả trực tiếp của định nghĩa Pod.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
