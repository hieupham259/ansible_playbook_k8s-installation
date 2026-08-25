# Không gian tên người dùng (User Namespaces)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** Giai đoạn 3 → nhóm [3a](00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời), bài 9/11 · Kiểm chứng
ở [Lab 3a](labs/LAB-3A-POD-VA-VONG-DOI.md).

Bài này **vốn viết cho người hardening node**, không phải bài giới thiệu Pod. Nó tham chiếu
capability của Linux, Pod Security Standards và cấu hình kubelet — toàn thứ của giai đoạn 8 và 9.
Ở lần đọc này chỉ cần hiểu tính năng làm gì, bật bằng trường nào, và nó cấm những gì; các mục
thiết lập node để **đọc lướt**.

**Phải hiểu ở lần đọc này:**

- Cách bật: đặt `pod.spec.hostUsers` thành `false`. Kubelet tự chọn dải UID/GID trên host cho Pod
  và bảo đảm **không hai Pod nào trên cùng một node dùng chung một ánh xạ**.
- Điều nó mua được: tiến trình chạy root **bên trong** container được ánh xạ sang một người dùng
  **không đặc quyền trên host**, nên khi container thoát ra host thì thiệt hại bị giới hạn.
  Capability cũng chỉ có hiệu lực trong namespace — `CAP_SYS_MODULE` mất tác dụng hoàn toàn,
  `CAP_SYS_ADMIN` bị giới hạn trong user namespace của Pod.
- Ranh giới dễ nhầm: `runAsUser`, `runAsGroup`, `fsGroup` **luôn trỏ tới người dùng bên trong
  container**. Vì vậy bật hay tắt user namespace **không đổi quyền sở hữu file trên volume**, và
  Pod vẫn chia sẻ được volume với Pod không dùng user namespace.
- Điều kiện tiên quyết là ở tầng hạ tầng chứ không phải Kubernetes: kernel Linux hỗ trợ idmap
  mount cho `/var/lib/kubelet/pods/` **và** cho mọi filesystem dùng trong volume của Pod, cộng
  với OCI runtime và CRI runtime đủ mới.
- Hạn chế cứng: đặt `hostUsers: false` thì **không được** đặt `hostNetwork: true`, `hostIPC: true`
  hay `hostPID: true`; không container nào trong `containers`, `initContainers`,
  `ephemeralContainers` được dùng `volumeDevices`; volume NFS không mount được.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| *Thiết lập node để hỗ trợ user namespace* — người dùng `kubelet`, `getsubids`, `/etc/subuid`, `/etc/subgid`, ràng buộc bội số 65536 | là thao tác cấu hình node | giai đoạn 8 — bài [04](04-kubelet-integration-vi.md) |
| *Số lượng ID cho mỗi Pod* (`idsPerPod` trong `KubeletConfiguration`) | cũng là cấu hình kubelet | giai đoạn 8 — bài [04](04-kubelet-integration-vi.md) |
| Chi tiết từng capability được nhắc tới | capability, seccomp, AppArmor học thành bài riêng | giai đoạn 9 — bài [127](127-linux-kernel-security-vi.md) |
| *Tích hợp với kiểm tra Pod security admission* và danh sách trường được nới lỏng | chưa học Pod Security Standards | giai đoạn 9 — bài [115](115-pod-security-standards-vi.md), [116](116-pod-security-admission-vi.md) |
| *Metric và khả năng quan sát* | chưa học metric của kubelet | giai đoạn 11 — bài [160](160-system-metrics-vi.md) |
| CVE và KEP được dẫn link | là bối cảnh, không phải nội dung phải nắm | không cần |

---

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
[Pod Security Standards](115-pod-security-standards-vi.md)
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

* Hãy xem [Dùng User Namespace với một Pod](295-user-namespaces-tasks-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 3:

1. Trường nào bật user namespace cho một Pod, và ai quyết định Pod được ánh xạ sang dải UID/GID
   nào trên host?
2. Bạn đặt `runAsUser: 1000` cho một Pod rồi bật user namespace cho nó. Quyền sở hữu file mà Pod
   ghi vào volume có đổi không?
3. Trước khi thử tính năng này trên `lab-k8s-worker2`, bạn phải kiểm những gì trên node? Ở đâu tra
   được phiên bản đang chạy của cluster lab?
4. Một Pod đang dùng `hostNetwork: true` để lấy IP của node. Bật thêm `hostUsers: false` cho nó
   được không?
5. Nếu kẻ tấn công thoát được khỏi container của một Pod có user namespace, vì sao hắn vẫn không
   nạp được kernel module trên node?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Trường **`pod.spec.hostUsers`**, đặt thành **`false`**. **Kubelet** là bên chọn UID/GID trên
   host mà Pod được ánh xạ tới, và nó làm việc đó theo cách bảo đảm **không có hai Pod nào trên
   cùng một node dùng chung một ánh xạ** — chính điều này giới hạn thứ một Pod có thể làm với các
   Pod khác trên cùng node.
2. **Không đổi.** `runAsUser`, `runAsGroup`, `fsGroup` **luôn tham chiếu người dùng bên trong
   container**, và chính những người dùng đó được dùng cho các volume mount, nên **UID/GID trên
   host không ảnh hưởng gì tới việc đọc/ghi trên volume**. Bài nói rõ hệ quả: các inode được
   tạo/đọc trong volume **giống hệt như khi Pod không dùng user namespace**, nhờ vậy bạn bật tắt
   tính năng này tùy ý và vẫn chia sẻ được volume với Pod không dùng user namespace.
3. Ba tầng, và **không tầng nào thuộc Kubernetes**: (a) **kernel Linux hỗ trợ idmap mount** cho
   filesystem của `/var/lib/kubelet/pods/` và cho **mọi** filesystem dùng trong volume của Pod —
   thực tế nghĩa là kernel đủ mới, vì tmpfs mới hỗ trợ từ một mốc nhất định và Kubernetes dùng
   tmpfs cho token service account lẫn Secret; (b) **OCI runtime** đủ mới — `runc` hoặc `crun`;
   (c) **CRI runtime** đủ mới — containerd hoặc CRI-O. Phiên bản thật của cluster lab tra ở
   [bảng phiên bản được khóa của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); phần
   kernel thì kiểm trực tiếp trên node bằng `uname -r`.
4. **Không.** Bài liệt kê hạn chế cứng: khi dùng user namespace cho Pod thì **không được phép dùng
   các namespace khác của host** — đặt `hostUsers: false` thì **không được** đặt
   `hostNetwork: true`, `hostIPC: true` hay `hostPID: true`. Điều đó hợp lý: mục đích của tính
   năng là cắt đường tới host, còn ba trường kia là mở đường tới host. Nếu Pod cần biết IP của
   node thì dùng downward API (bài [56](56-downward-api-vi.md)) thay vì `hostNetwork`.
5. Vì **capability được cấp cho một Pod chỉ có hiệu lực bên trong user namespace của nó**. Bài lấy
   đúng ví dụ này: **`CAP_SYS_MODULE` không có tác dụng gì** nếu được cấp cho Pod dùng user
   namespace — Pod đó không thể nạp kernel module. Tiến trình nghĩ nó là root, nhưng trên host nó
   là một người dùng không có đặc quyền, nên **những gì nó làm được với host là có giới hạn**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
