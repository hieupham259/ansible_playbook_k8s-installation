# Sử dụng sysctl trong một cluster Kubernetes (Using sysctls in a Kubernetes Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần I — Nền tảng Kubernetes](00-ALO-TRINH-ADMIN.md#phần-i--nền-tảng-kubernetes)
→ [Giai đoạn 9 — Bảo mật và multi-tenancy](00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy),
dòng **Thực hành**, bài 2/10 · Kiểm chứng ở
[Lab 9b — Pod Security và hardening](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md), phần B5 (B5.1 vì sao
không ghi thẳng vào `/proc/sys`, B5.2 sysctl an toàn, B5.3 sysctl không an toàn).

Bài mở đầu bằng phần *Trước khi bạn bắt đầu* gợi ý minikube và các playground. Bỏ qua đoạn đó:
mọi lab của lộ trình chạy trên ba VM `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`, và
điều kiện "ít nhất hai node không phải control plane" thì cluster của bạn đã thỏa.

**Phải hiểu ở lần đọc này:**

- Ba điều kiện để một sysctl được xếp vào nhóm **an toàn**: đặt nó cho một Pod không được ảnh
  hưởng Pod khác trên cùng node, không được hại sức khỏe node, không được giành thêm CPU hay bộ
  nhớ vượt ngoài resource limits của Pod. Bài nói rõ **phần lớn sysctl được namespace hóa vẫn
  chưa được coi là an toàn** (mục *Sysctl an toàn và không an toàn*).
- Mặc định: sysctl an toàn **bật sẵn**, sysctl không an toàn **tắt** và chỉ được bật thủ công
  trên **từng node** bằng cờ kubelet `--allowed-unsafe-sysctls`. Hệ quả quan trọng nhất của bài:
  Pod dùng sysctl không an toàn đang bị tắt **vẫn được lập lịch, nhưng không khởi chạy được**
  (mục *Bật các sysctl không an toàn*).
- Chỉ sysctl **được namespace hóa** mới cấu hình được qua `securityContext` của Pod, và
  `securityContext` cấp Pod áp cho **tất cả** container trong Pod đó. Sysctl không có namespace
  gọi là sysctl *cấp node*: muốn đổi thì phải cấu hình trên hệ điều hành của từng node, hoặc dùng
  DaemonSet với container đặc quyền (mục *Thiết lập sysctl cho một Pod*).
- Trong `spec` **không có** chỗ nào phân biệt an toàn với không an toàn — cả ba giá trị của ví dụ
  `sysctl-example` nằm chung một danh sách `spec.securityContext.sysctls`. Sự phân biệt nằm ở
  phía kubelet, không ở phía manifest (cùng mục).
- Hai ngoại lệ của nhóm an toàn: các sysctl `net.*` **không được phép** khi host networking được
  bật; và `net.ipv4.tcp_syncookies` không được namespace hóa trên Linux kernel 4.5 trở xuống
  (ghi chú trong mục *Sysctl an toàn và không an toàn*).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Danh sách đầy đủ tên sysctl an toàn kèm phiên bản Kubernetes và kernel tối thiểu của từng cái | là bảng tra cứu, không phải thứ học thuộc; ở đây chỉ cần biết danh sách này **ngắn và đóng** | bài [115](115-pod-security-standards-vi.md) đọc ngay sau Lab 9a — chính danh sách này là một ô trong bảng đặc tả Baseline; [Lab 9b](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md) B5.2 dùng đúng một mục trong đó |
| Cờ kubelet `--allowed-unsafe-sysctls` và biến thể `--extra-config` của minikube | bật nó là **sửa cấu hình kubelet trên từng node**, việc mà Lab 9b cấm | [giai đoạn 20 — Cấu hình lại cluster đang chạy](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy); [Lab 9b](labs/LAB-9B-POD-SECURITY-VA-HARDENING.md) B5.3 chứng minh mặt còn lại của cùng một câu mà không đụng vào node nào |
| Mục *Liệt kê tất cả các tham số sysctl* — `sudo sysctl -a`, `/proc/sys/`, các tiền tố `kernel.` `net.` `vm.` `dev.` | là kiến thức Linux nền, không phải cơ chế của Kubernetes | [giai đoạn 0 — Kiến thức nền ngoài thư mục này](00-ALO-TRINH-ADMIN.md#giai-đoạn-0--kiến-thức-nền-ngoài-thư-mục-này) |
| Khuyến nghị coi node có sysctl đặc biệt như node bị taint, và dùng taint/toleration để ghim Pod vào đúng node | đây là lời khuyên vận hành dựa trên cơ chế đã học rồi | bài [139](139-taint-and-toleration-vi.md) ở [giai đoạn 7](00-ALO-TRINH-ADMIN.md#giai-đoạn-7--lập-lịch-và-chính-sách-tài-nguyên), đã thực hành ở [Lab 7a](labs/LAB-7A-LAP-LICH-VA-EVICTION.md) |

---

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
[taint trên node](139-taint-and-toleration-vi.md)
để lập lịch các Pod đó lên đúng node.

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 9:

1. Bạn viết một Pod đặt `net.core.somaxconn` qua `spec.securityContext.sysctls` và ghim nó lên
   `lab-k8s-worker2`. Chưa node nào của cluster lab được bật `--allowed-unsafe-sysctls`. Pod có
   được **lập lịch** không, có **khởi chạy** được không, và bạn sẽ tìm dấu vết của việc đó ở đâu?
2. **Câu bẫy.** `kernel.shm_rmid_forced` và `kernel.msgmax` đều là sysctl **được namespace hóa**.
   Vì sao chỉ một trong hai đặt được ngay mà không cần đổi bất cứ thứ gì trên node?
3. Ba điều kiện nào khiến một sysctl được Kubernetes xếp vào nhóm *an toàn*?
4. Bạn cần đổi một sysctl **không** được namespace hóa cho một workload.
   `spec.securityContext.sysctls` có làm được không, và bài đưa ra hai đường nào thay thế?
5. Pod của bạn chạy với `hostNetwork` bật và muốn đặt `net.ipv4.tcp_syncookies` — vốn nằm trong
   danh sách an toàn. Có được không?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Được lập lịch, nhưng không khởi chạy được.** Đây là câu quan trọng nhất của bài: "Các Pod
   dùng sysctl không an toàn đang bị tắt vẫn sẽ được lập lịch, nhưng sẽ không khởi chạy được."
   Nghĩa là scheduler đã gán Pod cho `lab-k8s-worker2` — Pod **có** node — nhưng kubelet trên node
   đó từ chối khởi chạy nó. Dấu vết vì vậy nằm ở phía kubelet của node, không nằm ở phía
   scheduler: Pod kẹt lại chứ không phải `Pending` vì thiếu chỗ.
2. Vì **được namespace hóa không đồng nghĩa với an toàn** — đó chính là chỗ dễ nhầm. Bài liệt kê
   `kernel.shm*`, `kernel.msg*`, `kernel.sem`, `fs.mqueue.*` là nhóm **được namespace hóa**, và
   nói thẳng "phần lớn các sysctl được namespace hóa chưa chắc đã được coi là an toàn". Nhóm *an
   toàn* là một **danh sách riêng, ngắn hơn**, và trong danh sách đó có
   `kernel.shm_rmid_forced` nhưng **không** có `kernel.msgmax`. Namespace hóa là điều kiện **cần**
   để đặt được qua `securityContext`; an toàn là điều kiện đủ để đặt được **mà không cần bật gì
   trên node**.
3. Đặt sysctl đó cho một Pod: **không được ảnh hưởng đến bất kỳ Pod nào khác trên node**, **không
   được gây hại đến sức khỏe của node**, và **không được giành thêm tài nguyên CPU hay bộ nhớ
   vượt ra ngoài resource limits của Pod**. Ba điều kiện này nằm trên nền một yêu cầu trước đó:
   sysctl phải được namespace hóa đúng cách.
4. **Không.** Chỉ những sysctl được namespace hóa mới cấu hình được qua `securityContext` của
   Pod. Với sysctl **cấp node**, bài đưa ra hai đường: **tự cấu hình chúng trên hệ điều hành của
   từng node**, hoặc **dùng một DaemonSet với các container đặc quyền**.
5. **Không.** Đây là một trong hai ngoại lệ mà ghi chú của bài nêu ra: **các sysctl `net.*` không
   được phép dùng khi host networking được bật** — bất kể chúng có nằm trong danh sách an toàn
   hay không. Ngoại lệ còn lại: `net.ipv4.tcp_syncookies` không được namespace hóa trên Linux
   kernel 4.5 trở xuống.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
