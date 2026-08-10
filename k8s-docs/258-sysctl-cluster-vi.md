# Sử dụng sysctl trong một cluster Kubernetes (Using sysctls in a Kubernetes Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/>

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.21 [stable]`

Tài liệu này mô tả cách cấu hình và sử dụng các tham số kernel (kernel parameters) trong một
cluster Kubernetes thông qua giao diện sysctl.

> **Ghi chú:**
>
> Kể từ Kubernetes phiên bản 1.23, kubelet hỗ trợ dùng cả `/` lẫn `.`
> làm ký tự phân tách trong tên sysctl.
> Kể từ Kubernetes phiên bản 1.25, việc thiết lập sysctl cho một Pod hỗ trợ đặt tên sysctl có
> dấu gạch chéo (slash).
> Ví dụ, bạn có thể biểu diễn cùng một tên sysctl là `kernel.shm_rmid_forced` khi dùng dấu chấm
> làm ký tự phân tách, hoặc là `kernel/shm_rmid_forced` khi dùng dấu gạch chéo làm ký tự phân tách.
> Để biết thêm chi tiết về phương pháp chuyển đổi tham số sysctl, hãy tham khảo
> trang [sysctl.d(5)](https://man7.org/linux/man-pages/man5/sysctl.d.5.html) của
> dự án Linux man-pages.

## Trước khi bạn bắt đầu (Before you begin)

> **Ghi chú:**
>
> `sysctl` là một công cụ dòng lệnh dành riêng cho Linux, được dùng để cấu hình nhiều tham số
> kernel khác nhau, và nó không có sẵn trên các hệ điều hành không phải Linux.

Bạn cần có một cluster Kubernetes, và công cụ dòng lệnh kubectl phải được cấu hình để giao tiếp
với cluster của bạn. Bạn nên chạy hướng dẫn này trên một cluster có ít nhất hai node không đóng
vai trò control plane host. Nếu bạn chưa có cluster, bạn có thể tạo một cluster bằng
[minikube](https://minikube.sigs.k8s.io/docs/tutorials/multi_node/) hoặc dùng một trong các sân
chơi (playground) Kubernetes sau:

* [iximiuz Labs](https://labs.iximiuz.com/playgrounds?category=kubernetes&filter=all)
* [Killercoda](https://killercoda.com/playgrounds/scenario/kubernetes)
* [KodeKloud](https://kodekloud.com/public-playgrounds)

Với một số bước, bạn cũng cần có khả năng cấu hình lại các tùy chọn dòng lệnh
cho các kubelet đang chạy trên cluster của mình.

## Liệt kê tất cả các tham số sysctl (Listing all Sysctl Parameters)

Trong Linux, giao diện sysctl cho phép quản trị viên sửa đổi các tham số kernel
lúc chạy (runtime). Các tham số này có sẵn qua hệ thống file tiến trình ảo `/proc/sys/`.
Các tham số bao phủ nhiều hệ thống con (subsystem) khác nhau, chẳng hạn như:

- kernel (tiền tố phổ biến: `kernel.`)
- mạng (tiền tố phổ biến: `net.`)
- bộ nhớ ảo (tiền tố phổ biến: `vm.`)
- MDADM (tiền tố phổ biến: `dev.`)
- Nhiều hệ thống con khác được mô tả trong [tài liệu Kernel](https://www.kernel.org/doc/Documentation/sysctl/README).

Để lấy danh sách tất cả các tham số, bạn có thể chạy

```shell
sudo sysctl -a
```

## Sysctl an toàn và không an toàn (Safe and Unsafe Sysctls)

Kubernetes phân loại các sysctl thành _an toàn_ (safe) hoặc _không an toàn_ (unsafe). Ngoài việc
được namespace hóa đúng cách, một sysctl _an toàn_ còn phải được _cách ly_ (isolated) đúng mức
giữa các Pod trên cùng một node. Điều này có nghĩa là việc thiết lập một sysctl _an toàn_ cho
một Pod

- không được ảnh hưởng đến bất kỳ Pod nào khác trên node
- không được phép gây hại đến sức khỏe của node
- không được phép giành thêm tài nguyên CPU hay bộ nhớ vượt ra ngoài giới hạn tài nguyên
  (resource limits) của Pod.

Cho đến nay, phần lớn các sysctl _được namespace hóa_ (namespaced) chưa chắc đã được coi là
_an toàn_. Các sysctl sau đây được hỗ trợ trong nhóm _an toàn_:

- `kernel.shm_rmid_forced`;
- `net.ipv4.ip_local_port_range`;
- `net.ipv4.tcp_syncookies`;
- `net.ipv4.ping_group_range` (từ Kubernetes 1.18);
- `net.ipv4.ip_unprivileged_port_start` (từ Kubernetes 1.22);
- `net.ipv4.ip_local_reserved_ports` (từ Kubernetes 1.27, cần kernel 3.16+);
- `net.ipv4.tcp_keepalive_time` (từ Kubernetes 1.29, cần kernel 4.5+);
- `net.ipv4.tcp_fin_timeout` (từ Kubernetes 1.29, cần kernel 4.6+);
- `net.ipv4.tcp_keepalive_intvl` (từ Kubernetes 1.29, cần kernel 4.5+);
- `net.ipv4.tcp_keepalive_probes` (từ Kubernetes 1.29, cần kernel 4.5+).
- `net.ipv4.tcp_rmem` (từ Kubernetes 1.32, cần kernel 4.15+).
- `net.ipv4.tcp_wmem` (từ Kubernetes 1.32, cần kernel 4.15+).

> **Ghi chú:**
>
> Có một số ngoại lệ đối với nhóm sysctl an toàn:
>
> - Các sysctl `net.*` không được phép dùng khi host networking được bật.
> - Sysctl `net.ipv4.tcp_syncookies` không được namespace hóa trên Linux kernel phiên bản 4.5
>   trở xuống.

Danh sách này sẽ được mở rộng trong các phiên bản Kubernetes tương lai, khi kubelet
hỗ trợ các cơ chế cách ly tốt hơn.

### Bật các sysctl không an toàn (Enabling Unsafe Sysctls)

Tất cả các sysctl _an toàn_ đều được bật theo mặc định.

Tất cả các sysctl _không an toàn_ đều bị tắt theo mặc định và phải được quản trị viên cluster
cho phép thủ công trên từng node. Các Pod dùng sysctl không an toàn đang bị tắt vẫn sẽ được
lập lịch (schedule), nhưng sẽ không khởi chạy được.

Với cảnh báo nêu trên trong tâm trí, quản trị viên cluster có thể cho phép một số sysctl
_không an toàn_ nhất định trong các tình huống rất đặc biệt, chẳng hạn như tinh chỉnh ứng dụng
hiệu năng cao hoặc thời gian thực (real-time). Các sysctl _không an toàn_ được bật trên từng
node bằng một flag của kubelet; ví dụ:

```shell
kubelet --allowed-unsafe-sysctls \
  'kernel.msg*,net.core.somaxconn' ...
```

Với minikube, việc này có thể được thực hiện qua flag `extra-config`:

```shell
minikube start --extra-config="kubelet.allowed-unsafe-sysctls=kernel.msg*,net.core.somaxconn"...
```

Chỉ các sysctl _được namespace hóa_ mới có thể được bật theo cách này.

## Thiết lập sysctl cho một Pod (Setting Sysctls for a Pod)

Trong các Linux kernel hiện nay, một số sysctl _được namespace hóa_. Điều này có nghĩa là chúng
có thể được thiết lập độc lập cho từng Pod trên một node. Chỉ những sysctl được namespace hóa
mới có thể cấu hình được qua securityContext của Pod trong Kubernetes.

Các sysctl sau được biết là đã được namespace hóa. Danh sách này có thể thay đổi
trong các phiên bản tương lai của Linux kernel.

- `kernel.shm*`,
- `kernel.msg*`,
- `kernel.sem`,
- `fs.mqueue.*`,
- Những sysctl `net.*` có thể được thiết lập trong networking namespace của container. Tuy nhiên,
  vẫn có các ngoại lệ (ví dụ: `net.netfilter.nf_conntrack_max` và
  `net.netfilter.nf_conntrack_expect_max` có thể được thiết lập trong networking namespace
  của container nhưng không được namespace hóa trước Linux 5.12.2).

Các sysctl không có namespace được gọi là sysctl _cấp node_ (node-level). Nếu bạn cần thiết lập
chúng, bạn phải tự cấu hình chúng trên hệ điều hành của từng node, hoặc dùng một DaemonSet với
các container đặc quyền (privileged containers).

Hãy dùng securityContext của Pod để cấu hình các sysctl được namespace hóa. securityContext
áp dụng cho tất cả các container trong cùng một Pod.

Ví dụ này dùng securityContext của Pod để thiết lập một sysctl an toàn
`kernel.shm_rmid_forced` và hai sysctl không an toàn `net.core.somaxconn` và
`kernel.msgmax`. Trong đặc tả (specification) không có sự phân biệt nào giữa sysctl _an toàn_
và _không an toàn_.

> **Cảnh báo:**
>
> Chỉ sửa đổi các tham số sysctl sau khi bạn đã hiểu rõ tác động của chúng, để tránh
> làm mất ổn định hệ điều hành của bạn.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-example
spec:
  securityContext:
    sysctls:
    - name: kernel.shm_rmid_forced
      value: "0"
    - name: net.core.somaxconn
      value: "1024"
    - name: kernel.msgmax
      value: "65536"
  ...
```

> **Cảnh báo:**
>
> Do bản chất _không an toàn_ của chúng, việc sử dụng các sysctl _không an toàn_
> là rủi ro do bạn tự chịu và có thể dẫn đến các vấn đề nghiêm trọng như hành vi sai lệch của
> các container, thiếu hụt tài nguyên hoặc hỏng hoàn toàn một node.

Một thực hành tốt là coi những node có thiết lập sysctl đặc biệt như những node bị
_taint_ trong cluster, và chỉ lập lịch lên chúng những Pod nào cần các thiết lập
sysctl đó. Bạn nên dùng tính năng [_taints và toleration_
của Kubernetes](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#taint) để thực hiện điều này.

Một Pod dùng các sysctl _không an toàn_ sẽ không khởi chạy được trên bất kỳ node nào chưa
bật tường minh hai sysctl _không an toàn_ đó. Tương tự như với các sysctl _cấp node_,
bạn nên dùng
[tính năng _taints và toleration_](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#taint) hoặc
[taint trên node](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
để lập lịch các Pod đó lên đúng node.
