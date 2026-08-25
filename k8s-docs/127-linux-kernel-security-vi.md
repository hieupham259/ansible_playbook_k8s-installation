# Các ràng buộc bảo mật của Linux kernel cho Pod và container (Linux kernel security constraints for Pods and containers)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/>
>
> Tổng quan về các module bảo mật và các ràng buộc của Linux kernel mà bạn có thể dùng để tăng cường bảo mật (harden) cho Pod và container của mình.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 9](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài 12/18 · Kiểm chứng ở [Lab 9b](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md).

Đây là bài giải thích **ý nghĩa** của những cái tên đã xuất hiện trong bảng Baseline và
Restricted ở bài [115](115-pod-security-standards-vi.md): seccomp, AppArmor, SELinux,
capability, `allowPrivilegeEscalation`, `privileged`. Đọc để hiểu mỗi cơ chế hạn chế **cái gì**
và ai cưỡng chế nó. Cách viết profile cụ thể là tutorial riêng, không cần ở lần đọc này.

**Phải hiểu ở lần đọc này:**

- Biện pháp đầu tiên và rẻ nhất là **chạy workload không cần đặc quyền root**: đặt user và group
  Linux trong `securityContext` của Pod. Giá trị trong **manifest được ưu tiên hơn** giá trị
  trong container image — hữu ích khi bạn chạy image không do mình sở hữu. Chạy non-root làm
  **giảm khả năng bạn phải cưỡng chế các cơ chế bảo mật kernel** đã cấu hình.
- Ba cơ chế hạn chế ba thứ khác nhau: **seccomp** lọc **từng lời gọi hệ thống**; **AppArmor**
  hạn chế **tài nguyên mà một chương trình được truy cập**, định nghĩa tài nguyên bằng **đường
  dẫn file**; **SELinux** hạn chế truy cập bằng **nhãn và chính sách**, định danh tài nguyên
  bằng **inode**. Node Linux thường chỉ có **một trong hai** AppArmor hoặc SELinux, và tính năng
  phải được bật sẵn trong kernel của hệ điều hành bạn chọn.
- Khuyến nghị của bài về seccomp: **dùng seccomp profile mặc định đi kèm container runtime**.
  Tự viết profile chỉ khi thật sự cần kiểm soát chi tiết syscall, vì ba rủi ro: cấu hình có thể
  hỏng khi ứng dụng được cập nhật, kẻ tấn công vẫn lợi dụng được các syscall **được phép**, và
  quản lý profile cho từng ứng dụng rất khó ở quy mô lớn.
- Linux dùng **capability** để chia nhỏ đặc quyền root thành từng nhóm; nên cấp đúng capability
  cần qua trường `capabilities` trong `securityContext` thay vì bật chế độ đặc quyền.
- **Container đặc quyền ghi đè cả ba cơ chế cùng lúc**: seccomp chạy với profile `Unconfined`,
  AppArmor profile bị bỏ qua, SELinux chạy trong domain `unconfined_t`; và container đó **được
  cấp toàn bộ Linux capability**. Ngược lại, `allowPrivilegeEscalation: false` ngăn tiến trình
  **giành thêm capability mới** và ngăn người dùng không đặc quyền **đổi sang seccomp profile dễ
  dãi hơn**.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cách viết seccomp profile, AppArmor profile hay chính sách SELinux cụ thể | là tutorial riêng của kubernetes.io | giai đoạn 9, khi làm Lab 9b |
| Kubernetes Security Profiles Operator | là mẫu operator để quản lý ở quy mô lớn | giai đoạn 14 |
| *User namespace* — `hostUsers: false` | tính năng còn ở giai đoạn phát triển ban đầu | bài [55](55-user-namespaces-vi.md) |
| `windowsOptions.hostProcess` và Pod HostProcess trên Windows | cluster lab chỉ có node Linux | giai đoạn 15 |
| Khuyến nghị làm cô lập ở tầng mạng trước | NetworkPolicy đã học rồi | bài [84](84-network-policies-vi.md) |

---

Trang này mô tả một số tính năng bảo mật được tích hợp sẵn trong Linux kernel mà bạn có thể
sử dụng cho các workload Kubernetes của mình. Để tìm hiểu cách áp dụng các tính năng này
cho Pod và container, tham khảo
[Cấu hình SecurityContext cho Pod hoặc Container](291-security-context-vi.md).
Bạn nên đã quen thuộc với Linux và những kiến thức cơ bản về workload trong Kubernetes.

## Chạy workload không cần đặc quyền root (Run workloads without root privileges) {#run-without-root}

Khi bạn triển khai một workload trong Kubernetes, hãy dùng đặc tả Pod (Pod specification) để
hạn chế workload đó chạy dưới người dùng root trên node. Bạn có thể dùng trường
`securityContext` của Pod để định nghĩa cụ thể người dùng (user) và nhóm (group) Linux cho các
tiến trình trong Pod, và hạn chế một cách tường minh việc các container chạy dưới người dùng
root. Việc đặt các giá trị này trong manifest của Pod sẽ được ưu tiên hơn các giá trị tương tự
trong container image, điều này đặc biệt hữu ích nếu bạn đang chạy những image không phải
do bạn sở hữu.

> **Thận trọng:**
> Hãy đảm bảo rằng người dùng hoặc nhóm mà bạn gán cho workload có đủ quyền cần thiết
> để ứng dụng hoạt động đúng. Việc đổi sang một người dùng hoặc nhóm không có quyền
> phù hợp có thể dẫn tới các vấn đề truy cập file hoặc các thao tác thất bại.

Việc cấu hình các tính năng bảo mật của kernel trong trang này mang lại khả năng kiểm soát
chi tiết (fine-grained) đối với những hành động mà các tiến trình trong cluster của bạn có thể
thực hiện, nhưng quản lý các cấu hình này ở quy mô lớn có thể là một thách thức. Chạy các
container dưới người dùng không phải root (non-root), hoặc trong user namespace nếu bạn
cần đặc quyền root, sẽ giúp giảm khả năng bạn phải cưỡng chế các khả năng bảo mật kernel
mà bạn đã cấu hình.

## Các tính năng bảo mật trong Linux kernel (Security features in the Linux kernel) {#linux-security-features}

Kubernetes cho phép bạn cấu hình và sử dụng các tính năng của Linux kernel để cải thiện
sự cô lập (isolation) và tăng cường bảo mật cho các workload chạy trong container của bạn.
Các tính năng phổ biến bao gồm:

* **Chế độ điện toán an toàn (Secure computing mode — seccomp)**: Lọc những lời gọi hệ thống
  (system call) mà một tiến trình được phép thực hiện
* **AppArmor**: Hạn chế quyền truy cập của từng chương trình riêng lẻ
* **Security Enhanced Linux (SELinux)**: Gán nhãn bảo mật (security label) cho các đối tượng
  để việc cưỡng chế chính sách bảo mật dễ quản lý hơn

Để cấu hình thiết lập cho một trong các tính năng này, hệ điều hành mà bạn chọn cho các node
phải bật tính năng đó trong kernel. Ví dụ, Ubuntu 7.10 trở lên bật AppArmor theo mặc định.
Để biết hệ điều hành của bạn có bật một tính năng cụ thể hay không, hãy tham khảo tài liệu
của hệ điều hành đó.

Bạn dùng trường `securityContext` trong đặc tả Pod để định nghĩa các ràng buộc áp dụng cho
những tiến trình đó. Trường `securityContext` cũng hỗ trợ các thiết lập bảo mật khác, chẳng
hạn như các Linux capability cụ thể hoặc quyền truy cập file bằng UID và GID. Để tìm hiểu
thêm, tham khảo
[Cấu hình SecurityContext cho Pod hoặc Container](291-security-context-vi.md).

### seccomp

Một số workload của bạn có thể cần đặc quyền để thực hiện những hành động cụ thể với tư cách
người dùng root trên máy chủ (host) của node. Linux dùng *capability* để chia các đặc quyền
sẵn có thành các nhóm, nhờ đó tiến trình có thể nhận được đúng những đặc quyền cần thiết để
thực hiện các hành động cụ thể mà không cần được cấp toàn bộ đặc quyền. Mỗi capability có
một tập các lời gọi hệ thống (syscall) mà tiến trình có thể thực hiện. seccomp cho phép bạn
hạn chế từng syscall riêng lẻ này. Nó có thể được dùng để "đóng hộp cát" (sandbox) đặc quyền
của một tiến trình, hạn chế những lời gọi mà tiến trình đó có thể thực hiện từ userspace
vào kernel.

Trong Kubernetes, bạn dùng một *container runtime* trên mỗi node để chạy các container.
Một số runtime ví dụ gồm CRI-O, Docker, hoặc containerd. Mỗi runtime theo mặc định chỉ cho
phép một tập con các Linux capability. Bạn có thể tiếp tục giới hạn từng syscall được phép
bằng cách dùng một seccomp profile. Các container runtime thường đi kèm một seccomp
profile mặc định. Kubernetes cho phép bạn tự động áp dụng các seccomp profile đã nạp trên
node cho các Pod và container của bạn.

> **Ghi chú:**
> Kubernetes cũng có thiết lập `allowPrivilegeEscalation` cho Pod và container. Khi được đặt
> là `false`, thiết lập này ngăn các tiến trình giành thêm capability mới và hạn chế người
> dùng không có đặc quyền thay đổi seccomp profile đang được áp dụng sang một profile
> dễ dãi (permissive) hơn.

Để tìm hiểu cách triển khai seccomp trong Kubernetes, tham khảo
[Hạn chế syscall của Container bằng seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/)
hoặc [Tài liệu tham khảo seccomp cho node](https://kubernetes.io/docs/reference/node/seccomp/)

Để tìm hiểu thêm về seccomp, xem
[Seccomp BPF](https://www.kernel.org/doc/html/latest/userspace-api/seccomp_filter.html)
trong tài liệu Linux kernel.

#### Những cân nhắc với seccomp (Considerations for seccomp) {#seccomp-considerations}

seccomp là một cấu hình bảo mật ở tầng thấp mà bạn chỉ nên tự cấu hình nếu bạn cần kiểm
soát chi tiết các syscall của Linux. Việc dùng seccomp, đặc biệt ở quy mô lớn, có các rủi ro
sau:

* Cấu hình có thể bị hỏng khi ứng dụng được cập nhật
* Kẻ tấn công vẫn có thể lợi dụng các syscall được phép để khai thác lỗ hổng
* Việc quản lý profile cho từng ứng dụng riêng lẻ trở nên khó khăn ở quy mô lớn

**Khuyến nghị**: Hãy dùng seccomp profile mặc định đi kèm với container runtime của bạn.
Nếu bạn cần một môi trường cô lập hơn, hãy cân nhắc dùng một sandbox, chẳng hạn như gVisor.
Sandbox giải quyết các rủi ro nêu trên bằng các seccomp profile tùy chỉnh, nhưng đòi hỏi
nhiều tài nguyên tính toán hơn trên các node và có thể gặp vấn đề tương thích với GPU cũng
như các phần cứng chuyên dụng khác.

### AppArmor và SELinux: kiểm soát truy cập bắt buộc dựa trên chính sách (AppArmor and SELinux: policy-based mandatory access control) {#policy-based-mac}

Bạn có thể dùng các cơ chế kiểm soát truy cập bắt buộc (mandatory access control — MAC)
dựa trên chính sách của Linux, chẳng hạn như AppArmor và SELinux, để tăng cường bảo mật
cho các workload Kubernetes của mình.

#### AppArmor

[AppArmor](https://apparmor.net/) là một module bảo mật của Linux kernel, bổ sung cho cơ chế
phân quyền chuẩn dựa trên người dùng và nhóm của Linux nhằm giới hạn các chương trình trong
một tập tài nguyên hạn chế. AppArmor có thể được cấu hình cho bất kỳ ứng dụng nào để giảm
bề mặt tấn công (attack surface) tiềm tàng của nó và cung cấp khả năng phòng thủ theo chiều
sâu tốt hơn. AppArmor được cấu hình thông qua các profile được tinh chỉnh để cho phép đúng
những truy cập mà một chương trình hoặc container cụ thể cần, chẳng hạn như các Linux
capability, truy cập mạng, và quyền trên file. Mỗi profile có thể chạy ở chế độ *enforcing*
(cưỡng chế) — chặn truy cập tới các tài nguyên không được phép, hoặc chế độ *complain*
(than phiền) — chỉ báo cáo các vi phạm.

AppArmor có thể giúp bạn vận hành một hệ thống triển khai an toàn hơn bằng cách hạn chế
những gì container được phép làm, và/hoặc cung cấp khả năng kiểm toán (audit) tốt hơn thông
qua log hệ thống. Container runtime mà bạn dùng có thể đi kèm một AppArmor profile mặc định,
hoặc bạn có thể dùng một profile tùy chỉnh.

Để tìm hiểu cách dùng AppArmor trong Kubernetes, tham khảo
[Hạn chế quyền truy cập tài nguyên của Container bằng AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/).

#### SELinux

SELinux là một module bảo mật của Linux kernel cho phép bạn hạn chế quyền truy cập mà một
*chủ thể* (subject) cụ thể, chẳng hạn như một tiến trình, có được đối với các file trên hệ
thống của bạn. Bạn định nghĩa các chính sách bảo mật áp dụng cho những chủ thể mang các
nhãn SELinux cụ thể. Khi một tiến trình mang nhãn SELinux cố truy cập một file, SELinux
server sẽ kiểm tra xem chính sách bảo mật của tiến trình đó có cho phép truy cập hay không
và đưa ra quyết định cấp quyền.

Trong Kubernetes, bạn có thể đặt nhãn SELinux trong trường `securityContext` của manifest.
Các nhãn được chỉ định sẽ được gán cho những tiến trình đó. Nếu bạn đã cấu hình các chính
sách bảo mật tác động tới những nhãn này, kernel của hệ điều hành máy chủ sẽ cưỡng chế
các chính sách đó.

Để tìm hiểu cách dùng SELinux trong Kubernetes, tham khảo
[Gán nhãn SELinux cho container](291-security-context-vi.md#assign-selinux-labels-to-a-container).

#### Khác biệt giữa AppArmor và SELinux (Differences between AppArmor and SELinux) {#apparmor-selinux-diff}

Hệ điều hành trên các node Linux của bạn thường bao gồm một trong hai cơ chế AppArmor hoặc
SELinux. Cả hai cơ chế đều cung cấp các kiểu bảo vệ tương tự nhau, nhưng có những khác biệt
như sau:

* **Cấu hình**: AppArmor dùng các profile để định nghĩa quyền truy cập tài nguyên.
  SELinux dùng các chính sách áp dụng cho những nhãn cụ thể.
* **Cách áp dụng chính sách**: Trong AppArmor, bạn định nghĩa tài nguyên bằng đường dẫn file.
  SELinux dùng chỉ mục nút (index node — inode) của tài nguyên để định danh tài nguyên đó.

### Tóm tắt các tính năng (Summary of features) {#summary}

Bảng sau mô tả các trường hợp sử dụng và phạm vi của từng cơ chế kiểm soát bảo mật.
Bạn có thể dùng tất cả các cơ chế này cùng nhau để xây dựng một hệ thống được tăng cường
bảo mật hơn.

*Tóm tắt các tính năng bảo mật của Linux kernel*

| Tính năng bảo mật | Mô tả | Cách sử dụng | Ví dụ |
|---|---|---|---|
| seccomp | Hạn chế từng lời gọi kernel riêng lẻ từ userspace. Giảm khả năng một lỗ hổng lợi dụng syscall bị hạn chế có thể xâm phạm hệ thống. | Chỉ định một seccomp profile đã nạp trong đặc tả Pod hoặc container để áp dụng các ràng buộc của profile đó cho các tiến trình trong Pod. | Từ chối syscall `unshare`, syscall đã bị lợi dụng trong [CVE-2022-0185](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-0185). |
| AppArmor | Hạn chế chương trình truy cập những tài nguyên cụ thể. Giảm bề mặt tấn công của chương trình. Cải thiện việc ghi log kiểm toán. | Chỉ định một AppArmor profile đã nạp trong đặc tả container. | Hạn chế một chương trình chỉ-đọc không được ghi vào bất kỳ đường dẫn file nào trong hệ thống. |
| SELinux | Hạn chế truy cập tới các tài nguyên như file, ứng dụng, port và tiến trình bằng nhãn và chính sách bảo mật. | Chỉ định các hạn chế truy cập cho những nhãn cụ thể. Gắn nhãn cho các tiến trình bằng những nhãn đó để cưỡng chế các hạn chế truy cập liên quan đến nhãn. | Hạn chế một container không được truy cập file bên ngoài filesystem của chính nó. |

> **Ghi chú:**
> Các cơ chế như AppArmor và SELinux có thể cung cấp sự bảo vệ vượt ra ngoài phạm vi
> container. Ví dụ, bạn có thể dùng SELinux để giúp giảm thiểu
> [CVE-2019-5736](https://access.redhat.com/security/cve/cve-2019-5736).

### Những cân nhắc khi quản lý cấu hình tùy chỉnh (Considerations for managing custom configurations) {#considerations-custom-configurations}

seccomp, AppArmor và SELinux thường có một cấu hình mặc định cung cấp các mức bảo vệ cơ bản.
Bạn cũng có thể tạo các profile và chính sách tùy chỉnh đáp ứng yêu cầu của workload của
mình. Việc quản lý và phân phối các cấu hình tùy chỉnh này ở quy mô lớn có thể là thách
thức, đặc biệt nếu bạn dùng cả ba tính năng cùng lúc. Để giúp quản lý các cấu hình này ở
quy mô lớn, hãy dùng một công cụ như
[Kubernetes Security Profiles Operator](https://github.com/kubernetes-sigs/security-profiles-operator).

## Tính năng bảo mật ở tầng kernel và container đặc quyền (Kernel-level security features and privileged containers) {#kernel-security-features-privileged-containers}

Kubernetes cho phép bạn chỉ định rằng một số container đáng tin cậy có thể chạy ở chế độ
*đặc quyền* (privileged). Bất kỳ container nào trong một Pod đều có thể chạy ở chế độ đặc
quyền để sử dụng các khả năng quản trị của hệ điều hành mà bình thường không thể truy cập
được. Tính năng này khả dụng cho cả Windows lẫn Linux.

Container đặc quyền ghi đè một cách tường minh một số ràng buộc của Linux kernel mà bạn có
thể đang dùng trong các workload của mình, cụ thể như sau:

* **seccomp**: Container đặc quyền chạy với seccomp profile `Unconfined`, ghi đè bất kỳ
  seccomp profile nào mà bạn đã chỉ định trong manifest.
* **AppArmor**: Container đặc quyền bỏ qua mọi AppArmor profile đang được áp dụng.
* **SELinux**: Container đặc quyền chạy trong domain `unconfined_t`.

### Container đặc quyền (Privileged containers) {#privileged-containers}

Bất kỳ container nào trong một Pod cũng có thể bật *chế độ đặc quyền* (Privileged mode) nếu
bạn đặt trường `privileged: true` trong trường
[`securityContext`](291-security-context-vi.md)
của container đó. Container đặc quyền ghi đè hoặc vô hiệu hóa nhiều thiết lập tăng cường
bảo mật khác như seccomp profile, AppArmor profile, hoặc các ràng buộc SELinux đang được
áp dụng. Container đặc quyền được cấp toàn bộ các Linux capability, bao gồm cả những
capability mà chúng không cần. Ví dụ, một người dùng root trong container đặc quyền có thể
sử dụng các capability `CAP_SYS_ADMIN` và `CAP_NET_ADMIN` trên node, vượt qua cấu hình
seccomp của runtime và các hạn chế khác.

Trong hầu hết các trường hợp, bạn nên tránh dùng container đặc quyền, và thay vào đó hãy cấp
đúng những capability mà container của bạn cần bằng trường `capabilities` trong trường
`securityContext`. Chỉ dùng chế độ đặc quyền khi bạn cần một capability mà bạn không thể cấp
được thông qua securityContext. Điều này hữu ích cho những container muốn sử dụng các khả
năng quản trị của hệ điều hành, chẳng hạn như thao tác với network stack hoặc truy cập
thiết bị phần cứng.

Từ phiên bản Kubernetes 1.26 trở đi, bạn cũng có thể chạy các container Windows ở chế độ
đặc quyền tương tự bằng cách đặt cờ `windowsOptions.hostProcess` trong security context của
đặc tả Pod. Để biết chi tiết và hướng dẫn, xem
[Tạo một Pod HostProcess trên Windows](281-create-hostprocess-pod-vi.md).

## Khuyến nghị và các thực hành tốt (Recommendations and best practices) {#recommendations-best-practices}

* Trước khi cấu hình các khả năng bảo mật ở tầng kernel, bạn nên cân nhắc triển khai cô lập
  ở tầng mạng (network-level isolation). Để biết thêm thông tin, đọc
  [Danh sách kiểm tra bảo mật (Security Checklist)](129-security-checklist-vi.md#network-security).
* Trừ khi thật sự cần thiết, hãy chạy các workload Linux dưới người dùng không phải root
  bằng cách đặt user ID và group ID cụ thể trong manifest của Pod và chỉ định
  `runAsNonRoot: true`.

Ngoài ra, bạn có thể chạy workload trong user namespace bằng cách đặt `hostUsers: false`
trong manifest của Pod. Điều này cho phép bạn chạy các container với tư cách người dùng root
bên trong user namespace, nhưng lại là người dùng không phải root trong namespace của máy
chủ trên node. Tính năng này vẫn đang ở giai đoạn phát triển ban đầu và có thể chưa đạt
mức hỗ trợ mà bạn cần. Để biết hướng dẫn, tham khảo
[Dùng User Namespace với một Pod](295-user-namespaces-tasks-vi.md).

## Tiếp theo (What's next)

* [Tìm hiểu cách dùng AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/)
* [Tìm hiểu cách dùng seccomp](https://kubernetes.io/docs/tutorials/security/seccomp/)
* [Tìm hiểu cách dùng SELinux](291-security-context-vi.md#assign-selinux-labels-to-a-container)
* [Tài liệu tham khảo seccomp cho node](https://kubernetes.io/docs/reference/node/seccomp/)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. seccomp, AppArmor và SELinux hạn chế **cái gì** — mỗi cơ chế nhắm vào một đối tượng khác
   nhau. Nói luôn AppArmor và SELinux định danh tài nguyên bằng gì.
2. Bạn đã gán một seccomp profile chặt và một AppArmor profile riêng cho container, rồi thêm
   `privileged: true` vì ứng dụng cần thao tác với network stack. Hai cấu hình bảo mật kia còn
   tác dụng không?
3. Ba VM của cluster lab chạy Ubuntu Server. Theo bài, node của bạn nhiều khả năng đã bật sẵn
   cơ chế kiểm soát truy cập bắt buộc nào, và điều đó ảnh hưởng gì tới lựa chọn giữa AppArmor
   và SELinux?
4. Vì sao bài khuyên dùng seccomp profile mặc định của container runtime thay vì tự viết? Nêu
   ba rủi ro mà bài liệt kê.
5. `allowPrivilegeEscalation: false` chặn đúng hai việc gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **seccomp** hạn chế **các lời gọi hệ thống (syscall)** mà một tiến trình được phép thực hiện
   từ userspace vào kernel. **AppArmor** hạn chế **quyền truy cập của từng chương trình** tới
   một tập tài nguyên — Linux capability, truy cập mạng, quyền trên file — và **định nghĩa tài
   nguyên bằng đường dẫn file**. **SELinux** hạn chế quyền truy cập mà một **chủ thể** (ví dụ
   một tiến trình) có được đối với tài nguyên, dựa trên **nhãn bảo mật và chính sách**, và
   **định danh tài nguyên bằng inode**.
2. **Không.** Đây là bẫy đáng nhớ nhất của bài: **container đặc quyền ghi đè một cách tường minh
   cả ba ràng buộc** — seccomp chạy với profile **`Unconfined`**, ghi đè profile bạn chỉ định
   trong manifest; **mọi AppArmor profile đang áp dụng đều bị bỏ qua**; SELinux chạy trong domain
   **`unconfined_t`**. Container đó còn được cấp **toàn bộ Linux capability**, kể cả những cái
   nó không cần. Đúng cách là **cấp đúng capability cần qua trường `capabilities`**, và chỉ dùng
   chế độ đặc quyền khi cần một capability không cấp được qua `securityContext`.
3. **AppArmor**, vì bài nói **Ubuntu 7.10 trở lên bật AppArmor theo mặc định**. Điều đó quyết
   định luôn lựa chọn: bài nói hệ điều hành trên node Linux **thường chỉ bao gồm một trong hai**
   AppArmor hoặc SELinux, và muốn cấu hình một tính năng thì **kernel của hệ điều hành đó phải
   bật tính năng ấy**. Nên trên cluster lab, bạn viết AppArmor profile chứ không viết chính sách
   SELinux.
4. Vì seccomp là **cấu hình bảo mật ở tầng thấp**, chỉ nên tự cấu hình khi thật sự cần kiểm soát
   chi tiết syscall. Ba rủi ro: **cấu hình có thể bị hỏng khi ứng dụng được cập nhật**; **kẻ tấn
   công vẫn có thể lợi dụng các syscall được phép để khai thác lỗ hổng**; và **việc quản lý
   profile cho từng ứng dụng riêng lẻ trở nên khó khăn ở quy mô lớn**.
5. Nó **ngăn các tiến trình giành thêm capability mới**, và **hạn chế người dùng không có đặc
   quyền thay đổi seccomp profile đang được áp dụng sang một profile dễ dãi hơn**.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
