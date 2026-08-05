# Không gian tên người dùng (User Namespaces)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.36 [stable]`

Trang này giải thích cách user namespace (không gian tên người dùng) được sử dụng trong
các Pod của Kubernetes. User namespace cô lập người dùng đang chạy bên trong container
khỏi người dùng trên máy chủ (host).

Một tiến trình chạy với quyền root trong container có thể chạy dưới một người dùng khác
(không phải root) trên host; nói cách khác, tiến trình đó có đầy đủ đặc quyền đối với các
thao tác bên trong user namespace, nhưng không có đặc quyền đối với các thao tác bên
ngoài namespace đó.

Bạn có thể dùng tính năng này để giảm thiểu thiệt hại mà một container bị xâm nhập có thể
gây ra cho host hoặc cho các Pod khác trên cùng node. Có [một số lỗ hổng bảo
mật][KEP-vulns] được xếp hạng **HIGH** (cao) hoặc **CRITICAL** (nghiêm trọng) không thể
bị khai thác khi user namespace được kích hoạt. Dự kiến user namespace cũng sẽ giảm nhẹ
một số lỗ hổng trong tương lai.

[KEP-vulns]: https://github.com/kubernetes/enhancements/tree/217d790720c5aef09b8bd4d6ca96284a0affe6c2/keps/sig-node/127-user-namespaces#motivation

## Trước khi bạn bắt đầu (Before you begin)

> **Ghi chú:** Mục này liên kết đến các dự án bên thứ ba cung cấp chức năng mà Kubernetes cần. Các tác giả của dự án Kubernetes không chịu trách nhiệm về những dự án này, vốn được liệt kê theo thứ tự bảng chữ cái. Để thêm một dự án vào danh sách này, hãy đọc [hướng dẫn về nội dung](https://kubernetes.io/docs/contribute/style/content-guide/#third-party-content) trước khi gửi thay đổi.

Đây là tính năng chỉ có trên Linux và cần Linux hỗ trợ idmap mount trên các filesystem
được sử dụng. Điều này có nghĩa là:

* Trên node, filesystem mà bạn dùng cho `/var/lib/kubelet/pods/`, hoặc thư mục tùy chỉnh
  mà bạn cấu hình thay thế, cần hỗ trợ idmap mount.
* Tất cả các filesystem được dùng trong các volume của Pod phải hỗ trợ idmap mount.

Trong thực tế, điều này có nghĩa là bạn cần tối thiểu Linux 6.3, vì tmpfs bắt đầu hỗ trợ
idmap mount từ phiên bản đó. Điều này thường là bắt buộc vì nhiều tính năng của
Kubernetes sử dụng tmpfs (token của service account được mount mặc định dùng tmpfs,
Secret dùng tmpfs, v.v.)

Một số filesystem phổ biến hỗ trợ idmap mount trong Linux 6.3 là: btrfs, ext4, xfs, fat,
tmpfs, overlayfs.

Ngoài ra, container runtime và OCI runtime bên dưới của nó phải hỗ trợ user namespace.
Các OCI runtime sau đây có hỗ trợ:

* [crun](https://github.com/containers/crun) phiên bản 1.9 trở lên (khuyến nghị phiên bản 1.13+).
* [runc](https://github.com/opencontainers/runc) phiên bản 1.2 trở lên

Để dùng user namespace với Kubernetes, bạn cũng cần dùng một CRI container runtime để sử
dụng tính năng này với các Pod của Kubernetes:

* containerd: phiên bản 2.0 (trở lên) hỗ trợ user namespace cho các container.
* CRI-O: phiên bản 1.25 (trở lên) hỗ trợ user namespace cho các container.

Bạn có thể theo dõi trạng thái hỗ trợ user namespace trong cri-dockerd tại một
[issue][CRI-dockerd-issue] trên GitHub.

[CRI-dockerd-issue]: https://github.com/Mirantis/cri-dockerd/issues/74

## Giới thiệu (Introduction)

User namespace là một tính năng của Linux cho phép ánh xạ (map) người dùng trong
container sang những người dùng khác trên host. Hơn nữa, các capability được cấp cho một
Pod trong user namespace chỉ có hiệu lực bên trong namespace đó và vô hiệu bên ngoài nó.

Một Pod có thể chọn dùng user namespace bằng cách đặt trường `pod.spec.hostUsers` thành
`false`.

Kubelet sẽ chọn các UID/GID trên host mà Pod được ánh xạ tới, và thực hiện điều này theo
cách bảo đảm không có hai Pod nào trên cùng một node dùng chung một ánh xạ.

Các trường `runAsUser`, `runAsGroup`, `fsGroup`, v.v. trong `pod.spec` luôn tham chiếu
đến người dùng bên trong container. Những người dùng này sẽ được dùng cho các volume
mount (chỉ định trong `pod.spec.volumes`), do đó UID/GID trên host sẽ không ảnh hưởng gì
đến việc ghi/đọc trên các volume mà Pod có thể mount. Nói cách khác, các inode được
tạo/đọc trong các volume mà Pod mount sẽ giống hệt như khi Pod không dùng user namespace.

Nhờ vậy, một Pod có thể dễ dàng bật và tắt user namespace (mà không ảnh hưởng đến quyền
sở hữu file trên các volume của nó) và cũng có thể chia sẻ volume với các Pod không dùng
user namespace, chỉ bằng cách đặt người dùng phù hợp bên trong container (`RunAsUser`,
`RunAsGroup`, `fsGroup`, v.v.). Điều này áp dụng cho mọi volume mà Pod có thể mount, bao
gồm cả `hostPath` (nếu Pod được phép mount các volume `hostPath`).

Mặc định, khi tính năng này được bật, dải UID/GID hợp lệ là 0-65535. Điều này áp dụng cho
cả file lẫn tiến trình (`runAsUser`, `runAsGroup`, v.v.).

Các file dùng UID/GID nằm ngoài dải này sẽ được xem như thuộc về overflow ID, thường là
65534 (được cấu hình trong `/proc/sys/kernel/overflowuid` và
`/proc/sys/kernel/overflowgid`). Tuy nhiên, không thể sửa đổi những file đó, kể cả khi
chạy dưới người dùng/nhóm 65534.

Nếu dải 0-65535 được mở rộng bằng một tùy chọn cấu hình, các hạn chế nêu trên sẽ áp dụng
cho dải đã được mở rộng.

Hầu hết các ứng dụng cần chạy với quyền root nhưng không truy cập các namespace hoặc tài
nguyên khác của host sẽ tiếp tục chạy bình thường mà không cần thay đổi gì khi user
namespace được kích hoạt.

## Hiểu về user namespace cho Pod (Understanding user namespaces for pods) {#pods-and-userns}

Một số container runtime với cấu hình mặc định của chúng (như Docker Engine, containerd,
CRI-O) dùng các namespace của Linux để cô lập. Cũng tồn tại các công nghệ khác và có thể
dùng chúng với những runtime này (ví dụ: Kata Containers dùng máy ảo (VM) thay vì các
namespace của Linux). Trang này áp dụng cho các container runtime dùng namespace của
Linux để cô lập.

Khi tạo một Pod, mặc định sẽ có một số namespace mới được dùng để cô lập: một network
namespace để cô lập mạng của container, một PID namespace để cô lập cách nhìn về các tiến
trình, v.v. Nếu user namespace được sử dụng, nó sẽ cô lập người dùng trong container khỏi
người dùng trên node.

Điều này có nghĩa là các container có thể chạy với quyền root và được ánh xạ sang một
người dùng không phải root trên host. Bên trong container, tiến trình sẽ nghĩ rằng nó
đang chạy với quyền root (và do đó các công cụ như `apt`, `yum`, v.v. hoạt động bình
thường), trong khi thực tế tiến trình đó không có đặc quyền trên host. Bạn có thể kiểm
chứng điều này, ví dụ, bằng cách kiểm tra tiến trình của container đang chạy dưới người
dùng nào khi thực thi `ps aux` từ host. Người dùng mà `ps` hiển thị không giống với người
dùng bạn thấy khi thực thi lệnh `id` bên trong container.

Sự trừu tượng hóa này giới hạn những gì có thể xảy ra, ví dụ, khi container thoát được ra
host (container escape). Vì container đang chạy dưới một người dùng không có đặc quyền
trên host, những gì nó có thể làm với host là có giới hạn.

Hơn nữa, vì người dùng trên mỗi Pod sẽ được ánh xạ sang những người dùng khác nhau, không
chồng lấn trên host, những gì chúng có thể làm với các Pod khác cũng bị giới hạn.

Các capability được cấp cho một Pod cũng bị giới hạn trong user namespace của Pod và phần
lớn không có hiệu lực bên ngoài nó, một số thậm chí hoàn toàn vô hiệu. Đây là hai ví dụ:
- `CAP_SYS_MODULE` không có tác dụng gì nếu được cấp cho một Pod dùng user namespace, Pod
  đó không thể nạp các kernel module.
- `CAP_SYS_ADMIN` bị giới hạn trong user namespace của Pod và vô hiệu bên ngoài
  namespace đó.

Nếu không dùng user namespace, một container chạy với quyền root, trong trường hợp
container breakout (thoát khỏi container), sẽ có đặc quyền root trên node. Và nếu
container được cấp capability nào đó, các capability đó cũng có hiệu lực trên host. Không
điều nào trong số này còn đúng khi chúng ta dùng user namespace.

Nếu bạn muốn biết thêm chi tiết về những gì thay đổi khi user namespace được sử dụng, hãy
xem `man 7 user_namespaces`.

## Thiết lập node để hỗ trợ user namespace (Set up a node to support user namespaces)

Mặc định, kubelet gán cho các Pod các UID/GID nằm trên dải 0-65535, dựa trên giả định
rằng các file và tiến trình của host dùng UID/GID trong dải này, vốn là tiêu chuẩn của
hầu hết các bản phân phối Linux. Cách tiếp cận này ngăn mọi sự chồng lấn giữa UID/GID của
host với UID/GID của các Pod.

Việc tránh chồng lấn là quan trọng để giảm nhẹ tác động của các lỗ hổng như
[CVE-2021-25741][CVE-2021-25741], trong đó một Pod có khả năng đọc các file tùy ý trên
host. Nếu UID/GID của Pod và của host không chồng lấn, những gì Pod có thể làm sẽ bị giới
hạn: UID/GID của Pod sẽ không khớp với chủ sở hữu/nhóm sở hữu file của host.

Kubelet có thể dùng một dải tùy chỉnh cho user ID và group ID của các Pod. Để cấu hình
một dải tùy chỉnh, node cần có:

 * Một người dùng `kubelet` trong hệ thống (bạn không thể dùng bất kỳ tên người dùng nào
   khác ở đây)
 * Binary `getsubids` đã được cài đặt (một phần của [shadow-utils][shadow-utils]) và nằm
   trong `PATH` của binary kubelet.
 * Cấu hình các subordinate UID/GID (UID/GID phụ) cho người dùng `kubelet` (xem
   [`man 5 subuid`](https://man7.org/linux/man-pages/man5/subuid.5.html) và
   [`man 5 subgid`](https://man7.org/linux/man-pages/man5/subgid.5.html)).

Thiết lập này chỉ thu thập cấu hình dải UID/GID và không thay đổi người dùng đang thực
thi `kubelet`.

Bạn phải tuân theo một số ràng buộc đối với dải subordinate ID mà bạn gán cho người dùng
`kubelet`:

* Subordinate user ID, tức ID bắt đầu dải UID cho các Pod, **phải** là bội số của 65536
  và cũng phải lớn hơn hoặc bằng 65536. Nói cách khác, bạn không thể dùng bất kỳ ID nào
  trong dải 0-65535 cho các Pod; kubelet áp đặt hạn chế này để khiến việc vô tình tạo ra
  một cấu hình không an toàn trở nên khó xảy ra.

* Số lượng subordinate ID phải là bội số của 65536

* Số lượng subordinate ID phải tối thiểu là `65536 x <maxPods>`, trong đó `<maxPods>` là
  số Pod tối đa có thể chạy trên node.

* Bạn phải gán cùng một dải cho cả user ID lẫn group ID. Việc những người dùng khác có
  dải user ID không khớp với dải group ID thì không sao.

* Không dải nào được gán được phép chồng lấn với bất kỳ phần gán nào khác.

* Cấu hình subordinate phải chỉ nằm trên một dòng duy nhất. Nói cách khác, bạn không thể
  có nhiều dải.

Ví dụ, bạn có thể định nghĩa `/etc/subuid` và `/etc/subgid` để cả hai đều có các mục sau
cho người dùng `kubelet`:

```
# Định dạng là
#   name:firstID:count of IDs
# trong đó
# - firstID là 65536 (giá trị nhỏ nhất có thể)
# - count of IDs là 110 * 65536
#   (110 là giới hạn mặc định cho số lượng Pod trên node)

kubelet:65536:7208960
```

[CVE-2021-25741]: https://github.com/kubernetes/kubernetes/issues/104980
[shadow-utils]: https://github.com/shadow-maint/shadow

## Số lượng ID cho mỗi Pod (ID count for each of Pods)

Bắt đầu từ Kubernetes v1.33, số lượng ID cho mỗi Pod có thể được đặt trong
[`KubeletConfiguration`](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/).

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
userNamespaces:
  idsPerPod: 1048576
```

Giá trị của `idsPerPod` (uint32) phải là bội số của 65536.
Giá trị mặc định là 65536.
Giá trị này chỉ áp dụng cho các container được tạo sau khi kubelet được khởi động với
`KubeletConfiguration` này.
Các container đang chạy không bị ảnh hưởng bởi cấu hình này.

Trong các phiên bản Kubernetes trước v1.33, số lượng ID cho mỗi Pod được cố định cứng
(hard-coded) là 65536.

## Tích hợp với kiểm tra Pod security admission (Integration with Pod security admission checks)

Đối với các Pod Linux có bật user namespace, Kubernetes nới lỏng việc áp dụng
[Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards)
theo một cách có kiểm soát.

Nếu bạn tạo một Pod dùng user namespace, các trường sau sẽ không bị ràng buộc ngay cả
trong những ngữ cảnh áp dụng chuẩn bảo mật Pod _Baseline_ hoặc _Restricted_. Hành vi này
không gây lo ngại về bảo mật vì `root` bên trong một Pod có user namespace thực chất là
người dùng bên trong container, vốn không bao giờ được ánh xạ sang một người dùng có đặc
quyền trên host. Đây là danh sách các trường **không** bị kiểm tra đối với các Pod trong
những trường hợp đó:

- `spec.securityContext.runAsNonRoot`
- `spec.containers[*].securityContext.runAsNonRoot`
- `spec.initContainers[*].securityContext.runAsNonRoot`
- `spec.ephemeralContainers[*].securityContext.runAsNonRoot`
- `spec.securityContext.runAsUser`
- `spec.containers[*].securityContext.runAsUser`
- `spec.initContainers[*].securityContext.runAsUser`
- `spec.ephemeralContainers[*].securityContext.runAsUser`

Hơn nữa, nếu Pod nằm trong ngữ cảnh áp dụng chuẩn bảo mật Pod _Baseline_, việc kiểm tra
hợp lệ (validation) các trường sau cũng sẽ được nới lỏng tương tự:

- `spec.containers[*].securityContext.procMount`
- `spec.initContainers[*].securityContext.procMount`
- `spec.ephemeralContainers[*].securityContext.procMount`

với chuẩn bảo mật Pod _Restricted_, Pod vẫn phải chỉ dùng giá trị ProcMount mặc định hoặc
rỗng.

## Các hạn chế (Limitations)

Khi dùng user namespace cho Pod, không được phép dùng các namespace khác của host. Cụ
thể, nếu bạn đặt `hostUsers: false` thì bạn không được phép đặt bất kỳ trường nào sau
đây:

 * `hostNetwork: true`
 * `hostIPC: true`
 * `hostPID: true`

Không container nào được dùng `volumeDevices` (các raw block volume, ví dụ /dev/sda). Điều
này áp dụng cho tất cả các mảng container trong spec của Pod:
 * `containers`
 * `initContainers`
 * `ephemeralContainers`

### Hỗ trợ filesystem (Filesystem support)

Các Pod dùng user namespace yêu cầu filesystem phải hỗ trợ idmap mount. Một số filesystem
không hỗ trợ idmap mount, do đó không thể dùng với user namespace. Trong những trường hợp
như vậy, các event sau sẽ được sinh ra. Lưu ý rằng chi tiết cảnh báo phụ thuộc vào
container runtime bạn đang dùng.

```
Warning  Failed 1s kubelet Error: failed to create containerd task: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: failed to fulfil mount request: failed to set MOUNT_ATTR_IDMAP on ${your mount path} invalid argument (maybe the filesystem used doesn't support idmap mounts on this kernel?): unknown
```

Các volume NFS không thể được mount trong một Pod dùng user namespace vì Linux NFS client
chưa hỗ trợ idmap mount. Để xem danh sách các filesystem được hỗ trợ hiện tại, hãy xem
trang man [`mount_setattr(2)`](https://man7.org/linux/man-pages/man2/mount_setattr.2.html)
của Linux kernel.

## Metric và khả năng quan sát (Metrics and observability)

Kubelet xuất hai metric prometheus dành riêng cho user namespace:
 * `started_user_namespaced_pods_total`: một counter theo dõi số lượng Pod dùng user
   namespace được thử tạo.
 * `started_user_namespaced_pods_errors_total`: một counter theo dõi số lượng lỗi khi tạo
   các Pod dùng user namespace.

## Tiếp theo (What's next)

* Hãy xem [Dùng User Namespace với một Pod](https://kubernetes.io/docs/tasks/configure-pod-container/user-namespaces/)
