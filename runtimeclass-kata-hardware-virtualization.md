# RuntimeClass và container runtime dùng ảo hóa phần cứng — ví dụ Kata Containers

> Tài liệu này trả lời hai câu hỏi: **hardware virtualization là gì, vì sao cần nó**, và
> **làm thế nào để một nhóm Pod đòi hỏi mức đảm bảo an toàn thông tin cao chạy trong một
> container runtime dùng ảo hóa phần cứng**, đúng như động lực nêu trong bài
> [43 — Runtime Class](k8s-docs/43-runtime-class-vi.md). Ví dụ là kịch bản kinh điển của
> cộng đồng Kubernetes: **Kata Containers** trên **KVM/QEMU**, độc lập với chuỗi lab trong
> repo. Các bài liên quan: [144 — Pod Overhead](k8s-docs/144-pod-overhead-vi.md),
> [138 — Gán Pod vào Node](k8s-docs/138-assign-pod-node-vi.md).

Mục lục:

- [Phần 1 — Hardware virtualization là gì](#phần-1--hardware-virtualization-là-gì)
- [Phần 2 — Kịch bản: payment-gateway trên Kata Containers](#phần-2--kịch-bản-payment-gateway-trên-kata-containers)
- [Phần 3 — Thiết lập và config của toàn bộ thành phần](#phần-3--thiết-lập-và-config-của-toàn-bộ-thành-phần)
- [Phần 4 — Kiểm chứng](#phần-4--kiểm-chứng)
- [Phần 5 — Giới hạn, đánh đổi, và những gì Kata không bảo vệ](#phần-5--giới-hạn-đánh-đổi-và-những-gì-kata-không-bảo-vệ)
- [Phần 6 — Biến thể của cùng mẫu thiết kế](#phần-6--biến-thể-của-cùng-mẫu-thiết-kế)
- [Phần 7 — Nguồn](#phần-7--nguồn)

---

## Phần 1 — Hardware virtualization là gì

### 1.1. Định nghĩa

**Hardware virtualization** (ảo hóa có hỗ trợ phần cứng) là tập tính năng được đưa vào CPU,
MMU và chipset để một phần mềm gọi là **hypervisor** có thể tạo ra nhiều **máy ảo** (virtual
machine, VM), mỗi máy ảo chạy **kernel riêng** của nó, mà:

1. Phần lớn lệnh của guest chạy **trực tiếp trên CPU vật lý** với tốc độ gần native; các thao
   tác nhạy cảm gây VM exit để hypervisor xử lý thay vì phải dịch toàn bộ luồng lệnh.
2. CPU và MMU thực thi ranh giới địa chỉ của guest. Nếu toàn bộ chuỗi CPU, firmware, KVM, VMM
   và thiết bị ảo hoạt động đúng, guest không thể tự tham chiếu RAM ngoài vùng được cấp. Đây là
   một lớp cô lập bổ sung, không phải lời bảo đảm rằng hypervisor hay device backend không thể
   có lỗ hổng.

Trên x86, tính năng này là **Intel VT-x** và **AMD-V**; trên ARM là **Virtualization
Extensions** (EL2). Chúng đưa vào CPU một chế độ vận hành mới:

- **VMX root mode** — nơi hypervisor chạy, có toàn quyền.
- **VMX non-root mode** — nơi guest chạy. Guest vẫn thấy mình có đủ 4 ring (kernel guest ở
  ring 0, ứng dụng ở ring 3) nhưng mọi lệnh "nhạy cảm" (đụng tới bảng trang, thanh ghi điều
  khiển, I/O port, ngắt...) sẽ gây **VM exit**: CPU tự động dừng guest, lưu toàn bộ trạng thái
  vào cấu trúc **VMCS** (Intel) / **VMCB** (AMD) và trao quyền về hypervisor. Hypervisor xử lý
  rồi **VM entry** trả guest chạy tiếp.

Vì cơ chế bắt lệnh nằm trong silicon, guest kernel **không cần sửa**. Code guest không thể tự
chuyển sang root mode bằng một lệnh thông thường; một cuộc thoát VM vẫn có thể xảy ra nếu khai
thác được lỗi trong KVM, VMM hoặc một backend thiết bị mà guest được phép giao tiếp.

### 1.2. Bốn tầng phần cứng tham gia

| Nhu cầu | Intel | AMD | ARM | Giải quyết gì |
| --- | --- | --- | --- | --- |
| CPU: guest chạy ở ring 0 "giả" | VT-x (VMX) | AMD-V (SVM) | Virtualization Extensions, EL2 | Lệnh guest chạy native; lệnh nhạy cảm gây VM exit về hypervisor |
| Bộ nhớ: dịch địa chỉ hai tầng | EPT | NPT / RVI | Stage-2 translation | Guest tự quản page table của nó; phần cứng dịch tiếp *guest-physical → host-physical*. Khi được cấu hình đúng, địa chỉ guest không ánh xạ được tới RAM ngoài vùng được cấp |
| I/O: thiết bị DMA an toàn | VT-d | AMD-Vi | SMMU | IOMMU chặn thiết bị DMA vào vùng nhớ không thuộc VM; cho phép passthrough thiết bị (GPU, NIC) mà không phá cô lập |
| Ngắt | APICv, posted interrupts | AVIC | GICv3/v4 virtualization | Giảm số VM exit khi xử lý ngắt, tăng hiệu năng |
| Mã hóa bộ nhớ VM (thế hệ mới) | TDX | SEV-SNP | CCA | Thiết kế để hypervisor không đọc được plaintext RAM của guest; vẫn phải kết hợp attestation, firmware và quy trình cấp secret đúng — nền tảng của confidential computing |

Không có EPT/NPT, hypervisor phải duy trì **shadow page table** bằng phần mềm và bắt mọi thay
đổi bảng trang của guest; đó là nguồn overhead lớn nhất của thế hệ ảo hóa đầu tiên. Không có
IOMMU, một thiết bị được passthrough có thể DMA thẳng vào bộ nhớ host — phá vỡ toàn bộ mô
hình cô lập.

### 1.3. Chuỗi phần mềm phía trên phần cứng

```text
┌──────────────────────────────────────────────────────────────┐
│ Guest kernel (Linux riêng của VM) + tiến trình bên trong VM   │  ← VMX non-root
├──────────────────────────────────────────────────────────────┤
│ VMM userspace: QEMU / Firecracker / Cloud Hypervisor          │  ← emulate thiết bị,
│   (mở /dev/kvm, cấp RAM, tạo vCPU, cung cấp virtio-net,       │     quản lý vòng đời VM
│    virtio-blk, virtio-fs, vsock cho guest)                    │
├──────────────────────────────────────────────────────────────┤
│ Hypervisor trong kernel host: KVM (kvm.ko + kvm_intel/kvm_amd) │  ← VMX root
│   (lập lịch vCPU, xử lý VM exit, EPT, ngắt)                   │
├──────────────────────────────────────────────────────────────┤
│ CPU với VT-x/AMD-V + EPT/NPT + VT-d/AMD-Vi                     │
└──────────────────────────────────────────────────────────────┘
```

- **KVM** biến chính kernel Linux của host thành hypervisor. Nó chỉ làm phần cần đặc quyền:
  vCPU, bộ nhớ, VM exit.
- **VMM** (virtual machine monitor) là tiến trình userspace, mỗi VM một tiến trình. QEMU là VMM
  đầy đủ nhất. **Firecracker** (AWS viết bằng Rust cho Lambda/Fargate) và **Cloud Hypervisor**
  là "micro-VMM": bỏ hết thiết bị legacy, chỉ giữ virtio, để bề mặt tấn công và thời gian khởi
  động nhỏ nhất.
- VMware ESXi, Hyper-V, Xen là các hypervisor khác, cùng dựa trên VT-x/AMD-V.

### 1.4. Container thường và micro-VM khác nhau ở đâu

Container theo chuẩn OCI (`runc`, `crun`) **không phải** ảo hóa. Nó là một nhóm tiến trình
trên **kernel host**, được ngăn cách bằng namespace, cgroup, capabilities, seccomp và LSM
(AppArmor/SELinux). Toàn bộ các lớp ngăn cách đó là **code của kernel host** và chạy trong
**cùng một kernel** với mọi container khác trên node.

| Tiêu chí | Pod chạy `runc` | Pod chạy trong micro-VM (Kata) |
| --- | --- | --- |
| Kernel | dùng chung kernel host với mọi Pod khác | kernel guest **riêng cho mỗi Pod** |
| Ai thực thi ranh giới | kernel host (phần mềm) | CPU + EPT (phần cứng) và hypervisor |
| Bề mặt tấn công mà workload chạm được | syscall, `/proc`, `/sys`, netlink, ioctl, eBPF, io_uring... của kernel host | kernel guest và ranh giới guest-host: thiết bị virtio, VMM/KVM, shim, agent và các backend như `virtiofsd` |
| Hậu quả khi khai thác được kernel | lỗi kernel host có thể dẫn tới chiếm node nếu thỏa điều kiện khai thác | chiếm kernel guest chưa tự động đồng nghĩa chiếm host; attacker còn phải vượt qua một thành phần của ranh giới guest-host |
| Khởi động | mili giây | ~100 ms đến vài giây, phụ thuộc VMM và cấu hình |
| Bộ nhớ nền mỗi Pod | ≈ 0 | guest kernel + agent, hàng chục đến hàng trăm MiB |
| Chia sẻ page cache, image layer | trực tiếp | qua virtio-fs / virtio-blk, chậm hơn |
| Ứng dụng phải sửa gì | không | **không** — đây là điểm mấu chốt của RuntimeClass |

### 1.5. Tại sao cần ảo hóa phần cứng

1. **Vì x86 đời cũ không “classically virtualizable” bằng trap-and-emulate đơn giản.** Theo
   tiêu chí Popek–Goldberg, để trap-and-emulate hoạt động, mọi lệnh nhạy cảm phải là lệnh đặc
   quyền. x86 có các lệnh nhạy cảm nhưng **không** đặc quyền (`SGDT`, `SIDT`, `POPF`...), nên
   hypervisor không thể chỉ chờ trap. Trước VT-x/AMD-V, VMware vẫn ảo hóa được bằng **dịch nhị
   phân** code kernel lúc chạy; Xen dùng **paravirtualization**. QEMU TCG còn có thể mô phỏng
   hoàn toàn bằng phần mềm, nhưng chậm hơn KVM. VT-x/AMD-V cho guest kernel nguyên bản chạy gần
   native và làm các thao tác đã cấu hình trở thành VM exit.
2. **Vì ranh giới cô lập mạnh nhất là ranh giới do phần cứng thực thi.** Với EPT, guest không
   có địa chỉ nào trỏ được ra ngoài vùng RAM của nó, không phải "bị cấm" mà là "không tồn
   tại". Với container, ranh giới là các câu `if` trong kernel host; một lỗi logic ở bất kỳ
   đâu trong hàng triệu dòng code đó là một đường thoát.
3. **Vì thu hẹp bề mặt kernel host mà workload truy cập trực tiếp.** Workload trong VM chủ yếu
   tương tác với kernel guest và một tập giao diện guest-host có giới hạn hơn. Chiếm kernel
   guest, tự nó, chưa cấp quyền trên node; tuy vậy các lỗi ở VMM, KVM, shim hoặc `virtiofsd`
   vẫn có thể tạo đường thoát VM và phải được vá. Các CVE kernel/runtime như Dirty COW,
   Dirty Pipe, CVE-2022-0185, runc CVE-2019-5736 và CVE-2024-21626 minh họa vì sao không nên để
   workload không tin cậy truy cập trực tiếp kernel host; tác động thực tế còn phụ thuộc cấu
   hình và điều kiện khai thác của từng CVE.
4. **Vì multi-tenancy và tuân thủ.** VM boundary có thể là một control hữu ích khi tách workload
   của nhiều team hoặc vùng dữ liệu nhạy cảm. PCI DSS không tự động yêu cầu Kata và dùng Kata
   cũng không tự tạo ra compliance: đơn vị đánh giá vẫn phải kiểm chứng hiệu lực của toàn bộ
   segmentation, network control, logging, RBAC và quy trình vận hành.
5. **Vì tăng tốc so với mô phỏng hoàn toàn bằng phần mềm.** QEMU vẫn chạy được không cần KVM
   bằng TCG, nhưng chậm hơn đáng kể; mức chậm cụ thể phụ thuộc workload và phải benchmark thay
   vì gắn một hệ số cố định.
6. **Vì nó là nền cho bước tiếp theo:** confidential computing (SEV-SNP, TDX) — VM mà ngay cả
   quản trị viên host cũng không đọc được bộ nhớ — chỉ tồn tại trên nền ảo hóa phần cứng.

### 1.6. Cái giá phải trả

- **Bộ nhớ và CPU phụ trợ** cho mỗi Pod: guest kernel, agent, tiến trình VMM. Kubernetes gọi
  đây là *Pod Overhead* và có trường riêng để khai báo (bài 144).
- **Thời gian khởi động** dài hơn; scale-out nhanh kém hơn.
- **I/O chậm hơn** vì đi qua virtio; volume dạng file (`hostPath`, `emptyDir`, image layer) đi
  qua virtio-fs.
- **Mật độ thấp hơn**: cùng một node chứa được ít Pod hơn.
- **Một số tính năng Pod không có nghĩa** trong VM: `hostNetwork`, `hostPID`, `hostIPC`, thiết
  bị host, `nsenter` từ node.
- **Node phải có phần cứng phù hợp**: xem 1.7.

### 1.7. Điều kiện bắt buộc trên node

1. CPU có VT-x/AMD-V và tính năng đó **được bật trong BIOS/UEFI**.
2. Kernel host có KVM: `/dev/kvm` tồn tại, module `kvm_intel` hoặc `kvm_amd` nạp được.
3. Nếu **node đã là một VM** (rất phổ biến: node chạy trên VMware, Proxmox, cloud), hypervisor
   bên ngoài phải bật **nested virtualization** để lộ VT-x/AMD-V vào trong VM:

   | Hypervisor ngoài | Cách bật |
   | --- | --- |
   | VMware Workstation / ESXi | VM Settings → Processors → *Virtualize Intel VT-x/EPT or AMD-V/RVI* |
   | KVM / Proxmox / libvirt | trên host: `options kvm_intel nested=1` (hoặc `kvm_amd`), CPU type `host` |
   | Hyper-V | `Set-VMProcessor -VMName <vm> -ExposeVirtualizationExtensions $true` |
   | AWS EC2 | bare metal hoặc các instance type có `SupportedFeatures` chứa `nested-virtualization` (ví dụ một số dòng M7i/M8i, C7i/C8i, R7i/R8i, I7i); phải bật `NestedVirtualization=enabled` |
   | Azure | các dòng Dv3/Ev3 trở lên hỗ trợ nested |
   | GCP | tạo instance với `--enable-nested-virtualization` |

   Nested virtualization có thêm overhead và giới hạn tùy provider. Với workload nhạy về hiệu
   năng/độ trễ, phải benchmark và cân nhắc **node pool bare-metal riêng**.

Lệnh kiểm tra trên node Linux:

```bash
grep -Eoc '(vmx|svm)' /proc/cpuinfo        # > 0: CPU lộ tính năng ảo hóa
sudo modprobe kvm_intel || sudo modprobe kvm_amd
ls -l /dev/kvm                             # phải tồn tại
lsmod | grep -E '^kvm'                     # kvm và kvm_intel/kvm_amd
```

---

## Phần 2 — Kịch bản: payment-gateway trên Kata Containers

### 2.1. Bối cảnh

Công ty fintech **AcmePay** vận hành **một** cluster Kubernetes dùng chung cho nhiều team.
Trên đó có:

- **Workload thường**: `web`, `api-gateway`, `catalog`, `reporting`, các job ETL. Nhiều
  team deploy, cập nhật hàng ngày, kéo hàng trăm thư viện bên thứ ba, có cả plugin do đối tác
  viết. Đây là nhóm có xác suất bị chiếm cao nhất.
- **Workload nhạy cảm**: dịch vụ `payment-gateway` nhận số thẻ (PAN), CVV, gọi tới ngân hàng
  thanh toán (acquirer), giữ khóa ký giao dịch. Nó nằm trong **Cardholder Data Environment**
  theo PCI DSS. Yêu cầu đảm bảo an toàn thông tin ở mức cao nhất trong công ty.

Mục tiêu:

1. `payment-gateway` **không dùng chung kernel** với bất kỳ workload nào khác.
2. Nó chỉ chạy trên **một nhóm node được kiểm soát** (tầng *secure*), và workload thường
   **không được** rơi vào nhóm node đó.
3. **Không ai** deploy được Pod vào namespace `payments` mà bỏ qua ràng buộc trên, kể cả do
   quên.
4. Tài nguyên phụ trợ của VM phải được **scheduler và quota tính đúng**.
5. **Không sửa một dòng code** của ứng dụng.

### 2.2. Mô hình đe dọa — vì sao `runc` cộng hardening chưa đủ

| Mã | Đe dọa | Hậu quả nếu chỉ dùng `runc` |
| --- | --- | --- |
| T1 | Một Pod của team khác trên **cùng node** bị chiếm (dependency độc, RCE), attacker khai thác **lỗ hổng kernel** để escape | root trên node → đọc bộ nhớ, secret, token của `payment-gateway`; lấy credential kubelet |
| T2 | Lỗ hổng ở chính container runtime (`runc` CVE-2019-5736, CVE-2024-21626) | tương tự T1, không cần lỗi kernel |
| T3 | Side channel CPU giữa các tenant (Spectre, MDS, L1TF) | rò rỉ bộ nhớ chéo tiến trình trên cùng core |
| T4 | Developer deploy nhầm, quên `runtimeClassName` | Pod nhạy cảm chạy như Pod thường |
| T5 | Ai đó có quyền xóa RuntimeClass rồi tạo lại cùng tên với handler `runc` (field `handler` là immutable nên không sửa trực tiếp được) | Pod tạo mới có thể mất cô lập mà manifest ứng dụng không đổi |

Lý do `runc` cộng hardening (non-root, drop capabilities, seccomp, AppArmor, read-only
rootfs) **vẫn cần nhưng không đủ**: mọi lớp đó do **cùng một kernel host** thực thi. Một lỗ
hổng kernel host có thể vượt qua nhiều lớp cùng lúc nếu đủ điều kiện khai thác. Ảo hóa phần
cứng đặt thêm ranh giới bên ngoài kernel guest, nên T1/T2 không còn trực tiếp đặt workload vào
kernel host; attacker vẫn có thể tìm lỗi ở VMM, KVM, shim hoặc backend thiết bị để vượt ranh giới.

### 2.3. Thiết kế

```text
                          ┌──────────────── control plane ────────────────┐
                          │ API server — admission theo hai pha            │
                          │  ├─ mutation: RuntimeClass merge selector,     │
                          │  │  toleration và điền spec.overhead           │
                          │  └─ validation: PodSecurity, VAP và phần       │
                          │     validation của RuntimeClass                │
                          │ kube-scheduler: node label + taint + request   │
                          │                 + overhead                     │
                          └────────────────────────────────────────────────┘
                                             │
        ┌──────────────── tầng thường ──────┴─────┐   ┌──────────── tầng secure ──────────────┐
        │ node web-node-1..N                       │   │ node secure-node-1..3                  │
        │ label: (không có)                        │   │ label: katacontainers.io/kata-runtime  │
        │ taint: (không có)                        │   │ taint: workload-tier=secure:NoSchedule │
        │                                          │   │                                        │
        │ kubelet ─CRI─ containerd                 │   │ kubelet ─CRI─ containerd               │
        │   └ handler runc (mặc định)              │   │   ├ handler runc (Pod hệ thống: CNI…)  │
        │      └ shim-runc-v2 → runc               │   │   └ handler kata-qemu-runtime-rs       │
        │         └ Pod web / api-gateway          │   │      └ shim-kata-v2 → QEMU/KVM          │
        │           (namespace + cgroup,           │   │         └ micro-VM                     │
        │            dùng chung kernel host)       │   │            ├ guest kernel riêng        │
        │                                          │   │            ├ kata-agent                │
        │                                          │   │            └ Pod payment-gateway       │
        └──────────────────────────────────────────┘   └────────────────────────────────────────┘
```

Các quyết định thiết kế và lý do:

| Quyết định | Lý do |
| --- | --- |
| Runtime **Kata Containers** với hypervisor **QEMU/KVM** | Kata là hiện thực OCI-tương-thích phổ biến nhất cho micro-VM, tích hợp containerd qua shim v2, không sửa ứng dụng. QEMU là backend đầy đủ nhất (virtio-fs, hotplug CPU/RAM). Firecracker nhẹ hơn nhưng cần snapshotter devmapper, xem Phần 6 |
| Handler tên `kata-qemu-runtime-rs`, RuntimeClass cùng tên | Nêu rõ backend QEMU và runtime Rust hiện hành, tránh lẫn với handler `kata-qemu` của Go runtime cũ |
| Node pool riêng, có **label** và **taint** | Label để `scheduling.nodeSelector` của RuntimeClass kéo Pod về đúng node có handler (bài 43, mục *Lập lịch*). Taint để Pod thường không rơi vào node secure; RuntimeClass mang sẵn `tolerations` nên Pod không phải tự khai |
| `overhead.podFixed` trong RuntimeClass | Scheduler, quota và cgroup tính cả bộ nhớ của VM (bài 144), tránh over-commit node secure |
| Namespace `payments` riêng, Pod Security **restricted** | Cô lập phần cứng không thay thế hardening trong container; cả hai cùng bật |
| **ValidatingAdmissionPolicy** bắt buộc `runtimeClassName: kata-qemu-runtime-rs` trong `payments` | Chặn T4 ở API server, trước khi Pod tồn tại |
| **RBAC**: chỉ nhóm `platform-admins` ghi RuntimeClass | Chặn T5, đúng khuyến nghị của bài 43 |
| **NetworkPolicy** default-deny | Ranh giới mạng đi kèm ranh giới kernel |
| `podAntiAffinity` theo hostname | Ba replica nằm trên ba node secure khác nhau |

### 2.4. Luồng xử lý một Pod từ `kubectl apply` tới khi chạy

1. Developer apply Deployment trong namespace `payments` với
   `runtimeClassName: kata-qemu-runtime-rs`.
2. ReplicaSet controller tạo Pod. API server chạy admission theo hai pha:
   - Pha **mutation**: phần mutating của RuntimeClass tra object
     `kata-qemu-runtime-rs`, merge `scheduling.nodeSelector` (phép giao) và
     `scheduling.tolerations` (phép hợp), rồi điền `spec.overhead` từ `overhead.podFixed`.
   - Pha **validation**: PodSecurity kiểm tra mức `restricted`; ValidatingAdmissionPolicy kiểm
     tra đúng `runtimeClassName`; phần validating của RuntimeClass từ chối nếu class không tồn
     tại, selector xung đột hoặc Pod tự điền `spec.overhead`.
3. **kube-scheduler** lọc node: phải có label `katacontainers.io/kata-runtime=true`, Pod phải
   tolerate taint `workload-tier=secure`, node phải còn đủ `requests + overhead`. Ba replica
   rải ba node vì anti-affinity.
4. **kubelet** trên node được chọn gọi CRI `RunPodSandbox` với
   `runtime_handler: kata-qemu-runtime-rs`.
5. **containerd** tra bảng `runtimes.kata-qemu-runtime-rs` trong `config.toml`, khởi động
   `containerd-shim-kata-v2` cho sandbox này.
6. **Shim Kata runtime-rs** đọc `configuration-qemu-runtime-rs.toml`, chạy **một tiến trình
   QEMU** mở `/dev/kvm`, boot **guest kernel** của Kata với rootfs tối thiểu chứa **kata-agent**.
7. Shim nói chuyện với kata-agent bằng **ttRPC qua vsock**; agent tạo container bên trong VM từ
   image đã được containerd chuẩn bị và chia sẻ vào VM qua **virtio-fs**. Với network model
   `tcfilter`, veth do CNI tạo trên host được nối qua TAP/virtio-net vào VM. Service và
   NetworkPolicy còn phụ thuộc CNI, datapath và phiên bản cụ thể, nên phải kiểm thử end-to-end.
8. kubelet thấy container `Running`; probe, `kubectl logs`, `kubectl exec` đều đi qua shim và
   agent như bình thường.

---

## Phần 3 — Thiết lập và config của toàn bộ thành phần

Thứ tự bắt buộc theo bài 43: **cấu hình handler trên node trước (3.1–3.4), tạo object
RuntimeClass sau (3.6)**. Object trong API không tự sinh ra cấu hình trên node.

Quy ước trong phần này:

- Node secure: `secure-node-1`, `secure-node-2`, `secure-node-3`. Ubuntu 24.04, containerd 2.x.
- Node thường: `web-node-*`.
- Lệnh có `sudo` chạy **trên node** qua SSH. Lệnh `kubectl` chạy ở máy quản trị có quyền
  cluster-admin.
- Tên miền, registry, IP dưới đây là ví dụ; thay bằng giá trị thật của bạn.

### 3.1. Bước 0 — Chuẩn bị phần cứng và KVM trên từng node secure

```bash
# Chạy trên từng secure-node
grep -Eoc '(vmx|svm)' /proc/cpuinfo
# PASS: số > 0. Nếu = 0 mà CPU có VT-x: bật trong BIOS, hoặc bật nested virtualization ở
# hypervisor bên ngoài (bảng 1.7). Không có cách nào khác; Kata sẽ không chạy.

sudo modprobe kvm_intel 2>/dev/null || sudo modprobe kvm_amd
ls -l /dev/kvm
# PASS: /dev/kvm tồn tại, thuộc group kvm

echo 'kvm_intel' | sudo tee /etc/modules-load.d/kvm.conf   # hoặc kvm_amd
```

### 3.2. Bước 1 — Cài Kata Containers lên node secure

Tài liệu này dùng **Kata runtime-rs với QEMU**, là nhánh runtime Rust được upstream khuyến nghị
từ Kata 4.0. Không trộn các đường dẫn dưới đây với Go runtime cũ (`kata-go-static`,
`/opt/kata/bin/containerd-shim-kata-v2`, `configuration-qemu.toml`).

Trong production, pin đúng phiên bản đã kiểm thử và kiểm tra release note/security advisory
trước khi nâng. Ví dụ dưới đây pin `4.1.0`; nếu tổ chức phê duyệt phiên bản khác, thay đồng nhất
biến này trên mọi secure node.

```bash
# Chạy trên từng secure-node
KATA_VERSION="4.1.0"
KATA_ARCH="amd64"       # arm64/s390x/ppc64le nếu release hỗ trợ kiến trúc đó
curl -fsSLo /tmp/kata-static.tar.zst \
  "https://github.com/kata-containers/kata-containers/releases/download/${KATA_VERSION}/kata-static-${KATA_VERSION}-${KATA_ARCH}.tar.zst"

# Trước khi giải nén: đối chiếu checksum/signature với asset công bố trong đúng release.
sudo tar -xvf /tmp/kata-static.tar.zst -C /        # tạo cây /opt/kata/...

# runtime_path ở bước 3.3 trỏ thẳng tới shim, không cần symlink dựa vào tên.
test -x /opt/kata/runtime-rs/bin/containerd-shim-kata-v2
/opt/kata/runtime-rs/bin/containerd-shim-kata-v2 --version
test -r /opt/kata/share/defaults/kata-containers/runtime-rs/configuration-qemu-runtime-rs.toml
# PASS: ba lệnh trả exit code 0 và --version in đúng phiên bản đã pin.
```

Cây thư mục quan trọng sau khi giải nén:

| Đường dẫn | Vai trò |
| --- | --- |
| `/opt/kata/runtime-rs/bin/containerd-shim-kata-v2` | runtime-rs shim v2, mỗi Pod sandbox một tiến trình, nói chuyện với containerd và agent trong VM |
| `/opt/kata/bin/qemu-system-x86_64` | VMM |
| `/opt/kata/libexec/virtiofsd` | daemon chia sẻ filesystem host → guest |
| `/opt/kata/share/kata-containers/` | guest kernel và guest image/initrd; tên asset có thể mang version, đọc từ file cấu hình thay vì hard-code |
| `/opt/kata/share/defaults/kata-containers/runtime-rs/configuration-qemu-runtime-rs.toml` | cấu hình runtime-rs mặc định cho backend QEMU |

> **Cách tự động hóa tương đương:** dự án Kata cung cấp Helm chart `kata-deploy`
> (`oci://ghcr.io/kata-containers/kata-deploy-charts/kata-deploy`) để cài artifact trên node,
> cấu hình runtime, dán label node và tạo RuntimeClass. Bản hiện hành mặc định chạy các Job ngắn
> theo từng giai đoạn; chỉ dùng DaemonSet thường trực khi đặt `deploymentMode: daemonset`.
> Tài liệu này làm thủ công để thấy rõ từng config. Upstream khuyến nghị chart này cho Kubernetes;
> nếu làm thủ công trong production, phải quản lý cùng version/config bằng Ansible và drain node
> trước khi thay runtime.

### 3.3. Bước 2 — Khai báo handler `kata-qemu-runtime-rs` trong containerd

Đây chính là bước "cấu hình phần hiện thực CRI trên node" của bài 43. File
`/etc/containerd/config.toml`, định dạng của **containerd 2.x** (`version = 3`):

```toml
version = 3

[plugins.'io.containerd.cri.v1.runtime'.containerd]
  # Pod không khai runtimeClassName vẫn dùng runc — đúng hành vi "handler mặc định" của bài 43.
  default_runtime_name = "runc"

  [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
    [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
      SystemdCgroup = true

  # ---- Handler thứ hai: Kata Containers với backend QEMU ----
  [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.kata-qemu-runtime-rs]
    runtime_type = "io.containerd.kata-qemu-runtime-rs.v2"
    # Dùng đường dẫn tuyệt đối để không phụ thuộc PATH hoặc quy tắc suy tên shim.
    runtime_path = "/opt/kata/runtime-rs/bin/containerd-shim-kata-v2"
    # Pod privileged KHÔNG được nhận thiết bị host (/dev/kvm, đĩa...) vào trong VM.
    # Bắt buộc với runtime ảo hóa; nếu không, "privileged" sẽ phá cô lập từ bên trong VM.
    privileged_without_host_devices = true
    # Workload của ví dụ không cần annotation để ghi đè hypervisor.
    pod_annotations = []
    [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.kata-qemu-runtime-rs.options]
      ConfigPath = "/etc/kata-containers/configuration-qemu-runtime-rs.toml"
```

Với **containerd 1.7** (`version = 2`), thay tiền tố plugin bằng đúng chuỗi mà bài 43 trích:
`[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata-qemu-runtime-rs]`; các key bên
trong giữ nguyên. Upstream hiện khuyến nghị containerd 2.1.x trở lên; nhánh 1.7 chỉ nên dùng
khi distribution còn hỗ trợ và đã kiểm thử đầy đủ.

```bash
# Trên node production: drain node trước khi restart containerd và uncordon sau khi smoke test.
sudo systemctl restart containerd
sudo containerd config dump | grep -A9 'runtimes.kata-qemu-runtime-rs\]'
# PASS: thấy đúng runtime_type, runtime_path và ConfigPath
sudo crictl info | jq -r '.config.containerd.runtimes | keys[]'
# PASS: in ra cả "kata-qemu-runtime-rs" và "runc"
```

### 3.4. Bước 3 — Cấu hình Kata cho tầng secure

Không sửa file mặc định trong `/opt/kata/share/defaults`; copy ra `/etc/kata-containers`
(đường dẫn đã trỏ trong `ConfigPath` ở 3.3) rồi chỉnh các key liên quan tới an toàn và
tài nguyên.

```bash
sudo mkdir -p /etc/kata-containers
sudo cp /opt/kata/share/defaults/kata-containers/runtime-rs/configuration-qemu-runtime-rs.toml \
        /etc/kata-containers/configuration-qemu-runtime-rs.toml
```

Các key cần rà trong `/etc/kata-containers/configuration-qemu-runtime-rs.toml`. Giữ nguyên
`path`, `kernel`, `image`/`initrd`, `default_vcpus`, `default_memory`, `shared_fs` và
`virtio_fs_daemon` mà release đã sinh; không thay bằng tên asset của một release khác.

```toml
[hypervisor.qemu]
# Danh sách annotation Pod được phép ghi đè cấu hình hypervisor. Với tầng secure, để RỖNG:
# không cho bất kỳ Pod nào tự đổi kernel, kernel_params, bộ nhớ hay tắt IOMMU của VM.
enable_annotations = []

[runtime]
# Đưa toàn bộ tiến trình QEMU + shim vào cgroup của Pod. Overhead vì thế bị giới hạn bởi
# chính cgroup mà kubelet tạo theo requests/limits + overhead — khớp với bài 144.
sandbox_cgroup_only = true
```

```bash
sudo grep -E '^(path|kernel|image|initrd|default_vcpus|default_memory|shared_fs|virtio_fs_daemon|enable_annotations|sandbox_cgroup_only)[[:space:]]*=' \
  /etc/kata-containers/configuration-qemu-runtime-rs.toml
# PASS: mọi path được in ra đều tồn tại; enable_annotations=[] và sandbox_cgroup_only=true.
# Smoke test thật ở Phần 4 mới là gate chứng minh cả containerd, shim, QEMU/KVM và guest boot được.
```

### 3.5. Bước 4 — Label và taint node secure

```bash
kubectl label node secure-node-1 secure-node-2 secure-node-3 \
  katacontainers.io/kata-runtime=true
kubectl taint node secure-node-1 secure-node-2 secure-node-3 \
  workload-tier=secure:NoSchedule
kubectl get nodes -L katacontainers.io/kata-runtime
```

- Label `katacontainers.io/kata-runtime=true` là quy ước của Kata; RuntimeClass ở 3.6 chọn
  node bằng đúng key này.
- Taint `NoSchedule` giữ Pod thường (không có toleration) ra khỏi tầng secure. Pod hệ thống
  chạy DaemonSet (CNI, kube-proxy) thường đã tolerate mọi taint, nên vẫn chạy trên node secure
  bằng handler `runc` — đó là cố ý: chúng là hạ tầng của node, không phải tenant.

> **Hardening thêm:** label quyết định an toàn không nên để kubelet tự gán. Admission plugin
> `NodeRestriction` cấm kubelet sửa label có tiền tố `node-restriction.kubernetes.io/`. Nếu lo
> node bị chiếm tự dán label để hút Pod nhạy cảm về, dùng thêm
> `node-restriction.kubernetes.io/tier=secure` làm key cho `nodeSelector` của RuntimeClass.

### 3.6. Bước 5 — Tạo RuntimeClass `kata-qemu-runtime-rs`

File `runtimeclass-kata-qemu-runtime-rs.yaml`:

```yaml
# RuntimeClass là tài nguyên cấp cluster (không thuộc namespace) — bài 43.
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-qemu-runtime-rs
  labels:
    isolation.acmepay.io/level: hardware-vm
# Phải khớp CHÍNH XÁC tên bảng runtimes.<handler> trong config.toml của containerd (3.3).
handler: kata-qemu-runtime-rs
# Chi phí cố định của VM — bài 144. Upstream kata-deploy hiện khai 320Mi/250m cho QEMU;
# đây là điểm khởi đầu, vẫn phải đo lại với đúng release, guest image và workload.
overhead:
  podFixed:
    memory: "320Mi"
    cpu: "250m"
# Mục "Lập lịch" của bài 43: đảm bảo Pod chỉ tới node có handler này.
scheduling:
  nodeSelector:
    katacontainers.io/kata-runtime: "true"
  tolerations:
  - key: workload-tier
    operator: Equal
    value: secure
    effect: NoSchedule
```

```bash
kubectl apply -f runtimeclass-kata-qemu-runtime-rs.yaml
kubectl get runtimeclass
# NAME                    HANDLER                 AGE
# kata-qemu-runtime-rs    kata-qemu-runtime-rs    5s
kubectl api-resources --namespaced=false | grep runtimeclass   # xác nhận cấp cluster
```

### 3.7. Bước 6 — Namespace `payments`, quota, RBAC

File `payments-namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: payments
  labels:
    compliance.acmepay.io/scope: pci-dss
    # Pod Security Admission: cấm privileged, hostPath, hostNetwork, bắt buộc non-root,
    # drop ALL capabilities, seccomp RuntimeDefault. Lớp bảo vệ bên trong VM.
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
---
# Ba namespace phụ trợ dùng trong workload đối chứng và bài kiểm tra mạng ở Phần 4.
apiVersion: v1
kind: Namespace
metadata:
  name: web
---
apiVersion: v1
kind: Namespace
metadata:
  name: api-gateway
---
apiVersion: v1
kind: Namespace
metadata:
  name: payments-db
---
apiVersion: v1
kind: Namespace
metadata:
  name: kata-smoke
---
# Quota tính cả spec.overhead của Pod (bài 144), nên con số dưới đây phải chừa chỗ cho
# overhead: 3 replica x (requests + 320Mi/250m).
apiVersion: v1
kind: ResourceQuota
metadata:
  name: payments-quota
  namespace: payments
spec:
  hard:
    pods: "12"
    requests.cpu: "6"
    requests.memory: 8Gi
    limits.cpu: "12"
    limits.memory: 16Gi
---
apiVersion: v1
kind: LimitRange
metadata:
  name: payments-defaults
  namespace: payments
spec:
  limits:
  - type: Container
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    default:
      cpu: "1"
      memory: 1Gi
```

File `runtimeclass-rbac.yaml` — thực thi khuyến nghị "chỉ quản trị viên cluster mới được ghi
RuntimeClass" của bài 43. `cluster-admin` vốn đã có quyền này; mục đích của file là **đặt tên
rõ ràng cho quyền đó** để audit và **không** cấp `cluster-admin` cho developer.

```yaml
# Ai được ghi RuntimeClass: chỉ nhóm platform-admins.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: runtimeclass-admin
rules:
- apiGroups: ["node.k8s.io"]
  resources: ["runtimeclasses"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: runtimeclass-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: runtimeclass-admin
subjects:
- kind: Group
  name: platform-admins
  apiGroup: rbac.authorization.k8s.io
---
# Developer của team payments: quyền edit trong đúng namespace, không đụng tài nguyên cấp
# cluster. ClusterRole "edit" có sẵn không chứa node.k8s.io/runtimeclasses.
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payments-developers
  namespace: payments
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit
subjects:
- kind: Group
  name: payments-dev
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f payments-namespace.yaml -f runtimeclass-rbac.yaml
kubectl auth can-i create runtimeclasses --as=dev1 --as-group=payments-dev     # no
kubectl auth can-i create runtimeclasses --as=ops1 --as-group=platform-admins  # yes
```

### 3.8. Bước 7 — Admission policy bắt buộc RuntimeClass trong `payments`

Chặn đe dọa T4. Dùng `ValidatingAdmissionPolicy` có sẵn trong API server (GA từ v1.30),
không cần cài Gatekeeper hay Kyverno. File `vap-require-kata.yaml`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-kata-runtimeclass
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["pods"]
  validations:
  - expression: >-
      has(object.spec.runtimeClassName) && object.spec.runtimeClassName == 'kata-qemu-runtime-rs'
    message: >-
      Pod trong namespace thuộc phạm vi PCI DSS phải khai báo
      spec.runtimeClassName: kata-qemu-runtime-rs (cô lập bằng ảo hóa phần cứng).
    reason: Forbidden
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-kata-runtimeclass-pci
spec:
  policyName: require-kata-runtimeclass
  validationActions: [Deny, Audit]
  matchResources:
    namespaceSelector:
      matchLabels:
        compliance.acmepay.io/scope: pci-dss
```

Binding chọn namespace theo label `compliance.acmepay.io/scope=pci-dss` thay vì tên cứng, nên
namespace nhạy cảm thêm sau này chỉ cần dán label là được bảo vệ.

```bash
kubectl apply -f vap-require-kata.yaml
kubectl get validatingadmissionpolicy require-kata-runtimeclass
```

### 3.9. Bước 8 — NetworkPolicy cho `payments`

CNI phải hỗ trợ NetworkPolicy. Với network model `tcfilter`, policy thường được áp tại datapath
phía host trước khi frame đi qua TAP/virtio-net vào VM. Tuy nhiên không được suy ra rằng mọi
phiên bản Calico, Cilium hay Antrea đều tương thích: ví dụ Cilium SocketLB cần cấu hình phù hợp
để Service traffic của Kata không bị bỏ qua. Vì vậy kiểm tra Service, ingress và egress ở Phần
4 là **gate bắt buộc** cho đúng CNI/version của cluster.

File `payments-netpol.yaml`:

```yaml
# 1. Mặc định chặn toàn bộ ingress và egress trong namespace.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# 2. Chỉ api-gateway (namespace riêng) được gọi vào cổng TLS của payment-gateway.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-api-gateway
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payment-gateway
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: api-gateway
      podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 8443
---
# 3. Egress: DNS trong cluster, database của payments, và acquirer bên ngoài qua 443.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-required
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payment-gateway
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: payments-db
      podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24        # dải IP của acquirer (ví dụ)
    ports:
    - protocol: TCP
      port: 443
```

```bash
kubectl apply -f payments-netpol.yaml
```

### 3.10. Bước 9 — Workload `payment-gateway`

File `payment-gateway.yaml`. Điểm quyết định duy nhất về runtime là **một dòng**
`runtimeClassName: kata-qemu-runtime-rs`; phần còn lại là hardening bình thường của một Pod
`restricted`.

Gate trước khi apply: thay image digest, API endpoint, API key, signing key và TLS certificate;
đảm bảo Service `postgres.payments-db.svc.cluster.local` đã tồn tại. Các chuỗi `REPLACE_ME` và
domain `.example` dưới đây cố ý là placeholder, không thể làm `rollout status` PASS.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-gateway
  namespace: payments
automountServiceAccountToken: false
---
# Trong thực tế, secret này do External Secrets / Vault Agent nạp, và etcd phải bật
# encryption at rest. Các giá trị REPLACE_ME chỉ là khung: phải thay trước khi apply.
apiVersion: v1
kind: Secret
metadata:
  name: payment-gateway-keys
  namespace: payments
type: Opaque
stringData:
  ACQUIRER_API_KEY: "REPLACE_ME"
  SIGNING_KEY_PEM: |
    -----BEGIN PRIVATE KEY-----
    REPLACE_ME
    -----END PRIVATE KEY-----
---
apiVersion: v1
kind: Secret
metadata:
  name: payment-gateway-tls
  namespace: payments
type: kubernetes.io/tls
stringData:
  tls.crt: "REPLACE_ME"
  tls.key: "REPLACE_ME"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-gateway
  namespace: payments
  labels:
    app: payment-gateway
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-gateway
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: payment-gateway
        compliance.acmepay.io/scope: pci-dss
    spec:
      # ---- Dòng quyết định: chạy Pod này trong micro-VM qua runtime-rs/QEMU ----
      runtimeClassName: kata-qemu-runtime-rs
      serviceAccountName: payment-gateway
      automountServiceAccountToken: false
      # Không cần khai nodeSelector/tolerations cho tầng secure: RuntimeClass admission
      # sẽ merge chúng từ RuntimeClass (bài 43, mục Lập lịch).
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: payment-gateway
            topologyKey: kubernetes.io/hostname
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: gateway
        image: registry.acmepay.internal/payments/gateway@sha256:REPLACE_WITH_DIGEST
        ports:
        - name: https
          containerPort: 8443
        env:
        - name: LISTEN_ADDR
          value: ":8443"
        - name: ACQUIRER_ENDPOINT
          value: "https://api.acquirer.example:443"
        - name: PGHOST
          value: "postgres.payments-db.svc.cluster.local"
        envFrom:
        - secretRef:
            name: payment-gateway-keys
        resources:
          requests:
            cpu: 500m
            memory: 512Mi
          limits:
            cpu: "1"
            memory: 1Gi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: tls
          mountPath: /etc/gateway/tls
          readOnly: true
        - name: tmp
          mountPath: /tmp
        readinessProbe:
          httpGet:
            path: /healthz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /healthz
            port: https
            scheme: HTTPS
          initialDelaySeconds: 15
          periodSeconds: 20
      volumes:
      - name: tls
        secret:
          secretName: payment-gateway-tls
      - name: tmp
        emptyDir:
          medium: Memory
          sizeLimit: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: payment-gateway
  namespace: payments
spec:
  type: ClusterIP
  selector:
    app: payment-gateway
  ports:
  - name: https
    port: 8443
    targetPort: https
```

```bash
# Chỉ chạy sau khi gate placeholder và dependency ở trên đã PASS.
kubectl apply -f payment-gateway.yaml
kubectl -n payments rollout status deployment/payment-gateway
```

Sau admission, mỗi Pod thực tế có thêm những gì so với manifest (do RuntimeClass admission
điền, không phải do bạn viết):

```yaml
spec:
  overhead:                      # từ overhead.podFixed
    cpu: 250m
    memory: 320Mi
  nodeSelector:                  # từ scheduling.nodeSelector
    katacontainers.io/kata-runtime: "true"
  tolerations:                   # từ scheduling.tolerations, cộng các toleration mặc định
  - key: workload-tier
    operator: Equal
    value: secure
    effect: NoSchedule
```

Scheduler vì thế tìm node còn trống ít nhất **750m CPU và 832Mi** cho mỗi replica.

### 3.11. Bước 10 — Workload thường làm đối chứng

Không có `runtimeClassName`, không có toleration → chạy bằng `runc` trên node thường. Đây là
"hành vi handler mặc định" của bài 43.

```yaml
# Lưu thành web.yaml; namespace web đã được tạo ở bước 3.7.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f web.yaml
```

---

## Phần 4 — Kiểm chứng

### 4.0. Smoke test runtime trước khi dùng image ứng dụng

Test này không phụ thuộc registry riêng, secret, certificate hoặc database của AcmePay, nhưng node
vẫn phải pull được image thử nghiệm từ `quay.io` (hoặc thay bằng bản mirror nội bộ đã pin digest).
Nếu test không PASS, dừng tại đây và đọc event/log containerd trước khi triển khai
`payment-gateway`.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: kata-runtime-rs-test
  namespace: kata-smoke
spec:
  runtimeClassName: kata-qemu-runtime-rs
  restartPolicy: Never
  containers:
  - name: test
    image: quay.io/libpod/ubuntu:latest
    command: ["sh", "-c", "uname -r; sleep 3600"]
EOF
kubectl -n kata-smoke wait --for=condition=Ready pod/kata-runtime-rs-test --timeout=180s
kubectl -n kata-smoke logs kata-runtime-rs-test
# PASS: Pod Ready; log in kernel guest. So sánh với uname -r trên node ở mục 4.1.
kubectl -n kata-smoke delete pod kata-runtime-rs-test
```

### 4.1. Chứng minh Pod chạy trong VM, trên đúng node, với overhead được tính

```bash
POD=$(kubectl -n payments get pod -l app=payment-gateway -o jsonpath='{.items[0].metadata.name}')
NODE=$(kubectl -n payments get pod "$POD" -o jsonpath='{.spec.nodeName}')

# (1) Đúng tầng node
kubectl -n payments get pods -o wide
# PASS: cột NODE của cả 3 replica là secure-node-1/2/3, mỗi node một replica

# (2) Admission đã merge scheduling và overhead vào Pod
kubectl -n payments get pod "$POD" -o jsonpath='{.spec.runtimeClassName}{"\n"}{.spec.overhead}{"\n"}{.spec.nodeSelector}{"\n"}'
# PASS: kata-qemu-runtime-rs / {"cpu":"250m","memory":"320Mi"} / {"katacontainers.io/kata-runtime":"true"}
kubectl -n payments get pod "$POD" -o jsonpath='{.spec.tolerations[?(@.key=="workload-tier")]}{"\n"}'
# PASS: toleration workload-tier=secure:NoSchedule có mặt dù manifest không viết

# (3) Kernel bên trong Pod KHÁC kernel của node — bằng chứng trực tiếp của micro-VM
kubectl -n payments exec "$POD" -- uname -r
ssh "$NODE" uname -r
# PASS: hai chuỗi khác nhau (guest kernel của Kata vs kernel Ubuntu của node).
# Với Pod runc ở namespace web, hai lệnh tương ứng cho cùng một chuỗi.

# (4) Quan sát số vCPU guest. Đây là dữ liệu chẩn đoán, không phải PASS gate cố định:
# runtime-rs có thể cấp/hotplug vCPU khác nhau theo release và chế độ resource management.
kubectl -n payments exec "$POD" -- nproc
ssh "$NODE" nproc
# Ghi lại cả hai giá trị. Không giả định nproc phải bằng default_vcpus + limits.cpu.

# (5) Trên node: mỗi Pod Kata là đúng một tiến trình QEMU và một shim Kata
ssh "$NODE" "pgrep -fc '[q]emu-system'; pgrep -fc '[c]ontainerd-shim-kata'"
# PASS: cả hai đều = số Pod Kata đang chạy trên node đó (ở đây là 1)
ssh "$NODE" 'sudo crictl pods --name payment-gateway'
# PASS: cột RUNTIME là kata-qemu-runtime-rs

# (6) kubelet và scheduler tính overhead vào tài nguyên đã cấp của node
kubectl describe node "$NODE" | sed -n '/Non-terminated Pods/,/Events/p'
# PASS: dòng của Pod payment-gateway ghi CPU Requests 750m, Memory Requests 832Mi
#       (500m + 250m; 512Mi + 320Mi), không phải 500m / 512Mi

# (7) Quota namespace cũng đã cộng overhead
kubectl -n payments describe resourcequota payments-quota
# PASS: requests.cpu used = 2250m, requests.memory used = 2496Mi cho 3 replica

# (8) Ứng dụng làm việc bình thường qua Service và NetworkPolicy
# Pod test mang đúng label mà ingress NetworkPolicy cho phép. Production nên mirror và pin
# image này bằng digest trong registry nội bộ.
kubectl -n api-gateway run netcheck --image=curlimages/curl:8.12.1 \
  --labels=app=api-gateway --command -- sleep 1d
kubectl -n api-gateway wait --for=condition=Ready pod/netcheck --timeout=120s
kubectl -n api-gateway exec pod/netcheck -- \
  curl -sk https://payment-gateway.payments.svc.cluster.local:8443/healthz
# PASS: HTTP 200
kubectl -n api-gateway delete pod netcheck
```

### 4.2. Kiểm chứng các trường hợp lỗi mà bài 43 mô tả

**(A) Quên `runtimeClassName` trong namespace PCI — bị chặn ở API server (T4)**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oops
  namespace: payments
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile: {type: RuntimeDefault}
  containers:
  - name: c
    image: busybox:1.37
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities: {drop: ["ALL"]}
EOF
# PASS: lỗi ngay, không có Pod nào được tạo:
#   ... ValidatingAdmissionPolicy 'require-kata-runtimeclass' with binding
#   'require-kata-runtimeclass-pci' denied request: Pod trong namespace thuộc phạm vi
#   PCI DSS phải khai báo spec.runtimeClassName: kata-qemu-runtime-rs ...
```

**(B) RuntimeClass không tồn tại trong API**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-such-class
  namespace: web
spec:
  runtimeClassName: kata-fc
  containers:
  - name: c
    image: busybox:1.37
    command: ["sleep", "3600"]
EOF
# PASS: API server từ chối ở admission plugin RuntimeClass (bật mặc định):
#   pods "no-such-class" is forbidden: pod rejected: RuntimeClass "kata-fc" not found
# Nếu Pod sinh ra từ Deployment, lỗi này xuất hiện ở event FailedCreate của ReplicaSet.
```

**(C) RuntimeClass tồn tại trong API nhưng node KHÔNG có handler tương ứng**

Đây là tình huống "làm bước 2 trước bước 1" mà bài 43 cảnh báo. Tạo một RuntimeClass trỏ tới
handler chưa từng được khai trong `config.toml` của bất kỳ node nào:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-fc
handler: kata-fc
EOF
# Tạo lại Pod ở (B): lần này qua admission, được lập lịch, nhưng kubelet không tạo được sandbox.
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-such-class
  namespace: web
spec:
  runtimeClassName: kata-fc
  containers:
  - name: c
    image: busybox:1.37
    command: ["sleep", "3600"]
EOF
kubectl -n web get pod no-such-class
kubectl -n web describe pod no-such-class | tail -n 5
# PASS: event Warning FailedCreatePodSandBox với thông báo từ containerd, dạng
#   ... failed to get sandbox runtime: no runtime for "kata-fc" is configured
# PASS của negative test là event FailedCreatePodSandBox và Pod không chạy. Tài liệu RuntimeClass
# của Kubernetes mô tả kết cục terminal là phase Failed; một số tổ hợp kubelet/containerd có thể
# tiếp tục retry và tạm hiển thị STATUS ContainerCreating với status.phase=Pending. Hai trạng thái
# không đồng nghĩa: ghi lại cả phase và event nếu hành vi thực tế khác tài liệu khái niệm.
kubectl -n web get pod no-such-class -o jsonpath='{.status.phase}{"\n"}'
kubectl delete runtimeclass kata-fc
kubectl -n web delete pod no-such-class
```

**(D) Xung đột `nodeSelector` giữa Pod và RuntimeClass — bị từ chối ở admission**

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: conflict
  namespace: payments
spec:
  runtimeClassName: kata-qemu-runtime-rs
  nodeSelector:
    katacontainers.io/kata-runtime: "false"    # cùng key, khác giá trị với RuntimeClass
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    seccompProfile: {type: RuntimeDefault}
  containers:
  - name: c
    image: busybox:1.37
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities: {drop: ["ALL"]}
EOF
# PASS: bị từ chối ngay, vì phép giao của hai nodeSelector là rỗng — đúng câu "Nếu có xung
# đột, pod sẽ bị từ chối" của bài 43.
```

**(E) Pod thường không lọt vào node secure**

```bash
kubectl -n web get pods -o wide
# PASS: không Pod nào của namespace web nằm trên secure-node-*; taint NoSchedule đang làm việc.
```

**(F) Developer không đổi được RuntimeClass (T5)**

```bash
kubectl --as=dev1 --as-group=payments-dev patch runtimeclass kata-qemu-runtime-rs \
  --type=merge -p '{"handler":"runc"}'
# PASS: Error from server (Forbidden)
kubectl auth can-i delete runtimeclasses --as=dev1 --as-group=payments-dev
# PASS: no. Field handler vốn immutable; rủi ro T5 cần quyền delete rồi create lại cùng tên,
# nên phải chặn cả create/update/patch/delete chứ không chỉ patch.
```

---

## Phần 5 — Giới hạn, đánh đổi, và những gì Kata không bảo vệ

**Kata giảm rủi ro T1/T2 bằng một ranh giới bổ sung:** workload không chạy trực tiếp trên kernel
host. Chiếm được kernel guest của Pod `payment-gateway` chưa tự động cấp quyền trên node;
attacker còn phải vượt một thành phần guest-host như VMM/KVM, shim, `virtiofsd` hoặc backend
thiết bị. Đây không phải ranh giới tuyệt đối: phải pin bản đã vá, theo dõi security advisory,
giới hạn annotation và giảm tối đa filesystem/device sharing với host.

**Kata không bảo vệ chống lại — vẫn phải có lớp khác:**

| Rủi ro | Lớp bảo vệ đúng |
| --- | --- |
| Lỗi ứng dụng (SQL injection, SSRF, deserialization) | review code, WAF, kiểm thử |
| Rò rỉ secret qua log, biến môi trường, `kubectl get secret` | RBAC chặt, secret encryption at rest, secret manager ngoài |
| Image bị đầu độc | ký image (cosign) và policy chỉ cho image có chữ ký |
| Lộ mạng: Pod khác gọi thẳng vào 8443 | NetworkPolicy (3.9), mTLS |
| Side channel CPU giữa tenant (T3) | node pool riêng như thiết kế, microcode mới nhất, mitigations kernel, không dùng SMT chéo tenant |
| Quản trị viên node đọc RAM của VM | ngoài phạm vi Kata thường; cần confidential containers (Phần 6) |

**Đánh đổi vận hành:**

- Mỗi Pod tốn thêm overhead cố định (điểm khởi đầu ở đây là 320Mi/250m) và **thời gian khởi động** dài
  hơn; HPA scale-out phản ứng chậm hơn.
- `hostNetwork` không được Kata hỗ trợ như `runc`; `hostPID`/`hostIPC` không tạo cùng ý nghĩa
  xuyên qua VM. `hostPath` có thể được chia sẻ vào guest qua virtio-fs nhưng bị policy
  `restricted` của thiết kế này cấm và làm tăng bề mặt guest-host. Device passthrough cần
  VFIO/IOMMU và đánh giá threat model riêng.
- Volume qua virtio-fs chậm hơn bind-mount; workload I/O nặng nên dùng block volume (CSI với
  `volumeMode: Block` hoặc backend virtio-blk).
- Debug từ node (`nsenter`, `crictl exec` vào namespace) không thấy tiến trình trong VM; dùng
  `kubectl exec`/`kubectl debug` qua agent.
- Node là VM lồng nhau có thêm overhead và giới hạn theo instance/provider; benchmark trước khi
  dùng production, đặc biệt với workload nhạy về độ trễ.

---

## Phần 6 — Biến thể của cùng mẫu thiết kế

Mọi biến thể dưới đây dùng đúng ba mảnh của bài 43: **handler trên node → RuntimeClass →
`runtimeClassName` trong Pod**. Tên handler là ví dụ và phải đối chiếu RuntimeClass do đúng
version `kata-deploy` tạo ra hoặc bảng runtime trong containerd của cluster.

| Biến thể | Handler | Đặc điểm | Khi nào chọn |
| --- | --- | --- | --- |
| Kata + **Firecracker** | `kata-fc` (ví dụ của bài 144) | micro-VMM rất nhỏ, boot rất nhanh; **không** có virtio-fs nên containerd phải dùng snapshotter `devmapper` | serverless, function ngắn, cần mật độ và tốc độ boot |
| Kata + **Cloud Hypervisor** | `kata-clh` | micro-VMM viết bằng Rust, có virtio-fs | thay QEMU khi muốn bề mặt VMM nhỏ hơn nhưng vẫn đủ tính năng |
| **gVisor** | `runsc` | **không** phải ảo hóa phần cứng: kernel viết lại trong userspace chặn syscall; nhẹ hơn, không cần KVM (có thể dùng KVM làm platform) | cần cô lập syscall mà node không có VT-x, hoặc chấp nhận mô hình khác |
| **Confidential Containers** | `kata-qemu-tdx`, `kata-qemu-snp` | VM có bộ nhớ mã hóa bằng TDX/SEV-SNP, attestation trước khi giao secret | phải bảo vệ dữ liệu cả trước quản trị viên hạ tầng / cloud provider |
| Dịch vụ quản lý | GKE Sandbox (gVisor), AKS Pod Sandboxing (Kata trên Hyper-V), OpenShift Sandboxed Containers (Kata) | provider lo bước 1 và 2; bạn chỉ khai `runtimeClassName` | dùng cloud, không muốn quản lý node |

---

## Phần 7 — Nguồn

Kubernetes:

- [Runtime Class](https://kubernetes.io/docs/concepts/containers/runtime-class/) — bản dịch: [43](k8s-docs/43-runtime-class-vi.md)
- [Pod Overhead](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/) — bản dịch: [144](k8s-docs/144-pod-overhead-vi.md)
- [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/) — bản dịch: [138](k8s-docs/138-assign-pod-node-vi.md)
- [Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Validating Admission Policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- [Admission Control — mutating phase trước validating phase](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [RuntimeClass API reference](https://kubernetes.io/docs/reference/kubernetes-api/node/runtime-class-v1/)
- [KEP-585 RuntimeClass](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/585-runtime-class/README.md)

Kata Containers và containerd:

- [Kata Containers — How to use Kata with containerd](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/containerd-kata.md)
- [Kata Containers — Installation, runtime-rs và release tarball](https://github.com/kata-containers/kata-containers/blob/main/docs/installation.md)
- [Kata Containers 4.1.0 release](https://github.com/kata-containers/kata-containers/releases/tag/4.1.0)
- [Kata Containers — Host cgroups](https://github.com/kata-containers/kata-containers/blob/main/docs/design/host-cgroups.md)
- [Kata Containers — virtio-fs và Pod Overhead](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-use-virtio-fs-with-kata.md)
- [Kata Containers — kata-deploy](https://github.com/kata-containers/kata-containers/tree/main/tools/packaging/kata-deploy)
- [Kata Containers — Architecture](https://github.com/kata-containers/kata-containers/blob/main/docs/design/architecture/README.md)
- [Kata Containers — Threat model](https://github.com/kata-containers/documentation/blob/master/design/threat-model/threat-model.md)
- [Kata security advisories](https://github.com/kata-containers/kata-containers/security/advisories)
- [Kata issue #11149 — giới hạn với Cilium](https://github.com/kata-containers/kata-containers/issues/11149)
- [Cilium issue #33565 — SocketLB với Kata](https://github.com/cilium/cilium/issues/33565)
- [containerd CRI plugin config](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)
- [Confidential Containers](https://confidentialcontainers.org/)

Ảo hóa phần cứng:

- Intel® 64 and IA-32 Architectures Software Developer's Manual, Volume 3C — *Virtual Machine Extensions (VMX)*
- AMD64 Architecture Programmer's Manual, Volume 2 — *Secure Virtual Machine (SVM)*
- [KVM — kernel documentation](https://docs.kernel.org/virt/kvm/index.html)
- [QEMU TCG system emulation](https://www.qemu.org/docs/master/about/emulation.html)
- [Firecracker — design](https://github.com/firecracker-microvm/firecracker/blob/main/docs/design.md)
- [AWS EC2 — nested virtualization](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/amazon-ec2-nested-virtualization.html)
- [PCI SSC — Cloud Computing Guidelines](https://listings.pcisecuritystandards.org/pdfs/PCI_SSC_Cloud_Guidelines_v3.pdf)
- [`pgrep(1)` — giới hạn tên tiến trình 15 ký tự](https://www.man7.org/linux/man-pages/man1/pgrep.1.html)
- Popek, G. J.; Goldberg, R. P. (1974). *Formal Requirements for Virtualizable Third Generation Architectures*. Communications of the ACM.
