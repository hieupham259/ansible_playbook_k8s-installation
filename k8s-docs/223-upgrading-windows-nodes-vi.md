# Nâng cấp các node Windows (Upgrading Windows nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-windows-nodes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 17 — Nâng cấp cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster), bài 3/5 · Chỉ kiểm chứng
được nếu cluster có node Windows; xem **Lab 15 — Node Windows** (tùy chọn) trong
[bản đồ lab](labs/README.md#4-bản-đồ-lab).

**Bỏ qua nếu cluster của bạn chỉ có Linux.** Đọc lướt để thấy quy trình *giống* node Linux ở
đâu và *khác* ở đâu; phần khung khái niệm đã nằm ở bài [221](221-kubeadm-upgrade-vi.md).

**Phải hiểu ở lần đọc này:**

- Khung quy trình **giống hệt** node Linux: nâng kubeadm → `kubeadm upgrade node` → drain →
  nâng kubelet → uncordon. Nguyên tắc "control plane trước, worker sau" cũng giữ nguyên.
- Khác biệt thực thi: không có package manager — binary được **tải trực tiếp** bằng `curl.exe`
  từ `dl.k8s.io`, và các thành phần chạy như **Windows Service** nên phải `stop-service` /
  `restart-service`.
- Trên Windows phải nâng thêm **`kube-proxy`** một cách tường minh; trên node Linux kube-proxy
  thường chạy bằng DaemonSet nên được nâng theo cluster.
- Ngoại lệ: nếu kube-proxy chạy trong HostProcess container bên trong Pod thì nâng bằng cách
  apply manifest mới, không dùng `stop-service`.
- `kubectl drain` và `kubectl uncordon` vẫn chạy **từ máy có quyền truy cập API**, không phải
  trên node Windows.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Đường dẫn cụ thể tới `kubeadm.exe`, `kubelet.exe`, `kube-proxy.exe` | chỉ có nghĩa khi ngồi trước máy Windows thật | Lab 15 (tùy chọn) |
| HostProcess container cho kube-proxy | là một khái niệm riêng của Windows | bài [281](281-create-hostprocess-pod-vi.md) ở giai đoạn 15 |

---

**TRẠNG THÁI TÍNH NĂNG:** `Kubernetes v1.18 [beta]`

Trang này giải thích cách nâng cấp một node Windows được tạo bằng kubeadm.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có quyền truy cập shell vào tất cả các node, và công cụ dòng lệnh kubectl phải được
cấu hình để giao tiếp với cluster của bạn. Khuyến nghị chạy hướng dẫn này trên một cluster có
ít nhất hai node không đóng vai trò máy chủ control plane.

Máy chủ Kubernetes của bạn phải ở phiên bản bằng hoặc mới hơn 1.17. Để kiểm tra phiên bản,
hãy chạy `kubectl version`.

* Hãy làm quen với
  [quy trình nâng cấp phần còn lại của cluster kubeadm](221-kubeadm-upgrade-vi.md).
  Bạn nên nâng cấp các node control plane trước khi nâng cấp các node Windows.

## Nâng cấp các node worker (Upgrading worker nodes)

### Nâng cấp kubeadm (Upgrade kubeadm)

1. Từ node Windows, nâng cấp kubeadm:

   ```powershell
   # thay 1.36.0 bằng phiên bản mà bạn mong muốn
   curl.exe -Lo <path-to-kubeadm.exe>  "https://dl.k8s.io/v1.36.0/bin/windows/amd64/kubeadm.exe"
   ```

### Drain node (Drain the node)

1. Từ một máy có quyền truy cập tới API Kubernetes, chuẩn bị node cho việc bảo trì bằng cách
   đánh dấu node là không thể lập lịch (unschedulable) và trục xuất (evict) các workload:

   ```shell
   # thay <node-to-drain> bằng tên node mà bạn đang drain
   kubectl drain <node-to-drain> --ignore-daemonsets
   ```

   Bạn sẽ thấy đầu ra tương tự như sau:

   ```
   node/ip-172-31-85-18 cordoned
   node/ip-172-31-85-18 drained
   ```

### Nâng cấp cấu hình kubelet (Upgrade the kubelet configuration)

1. Từ node Windows, gọi lệnh sau để đồng bộ cấu hình kubelet mới:

   ```powershell
   kubeadm upgrade node
   ```

### Nâng cấp kubelet và kube-proxy (Upgrade kubelet and kube-proxy)

1. Từ node Windows, nâng cấp và khởi động lại kubelet:

   ```powershell
   stop-service kubelet
   curl.exe -Lo <path-to-kubelet.exe> "https://dl.k8s.io/v1.36.0/bin/windows/amd64/kubelet.exe"
   restart-service kubelet
   ```

2. Từ node Windows, nâng cấp và khởi động lại kube-proxy.

   ```powershell
   stop-service kube-proxy
   curl.exe -Lo <path-to-kube-proxy.exe> "https://dl.k8s.io/v1.36.0/bin/windows/amd64/kube-proxy.exe"
   restart-service kube-proxy
   ```

> **Ghi chú:**
>
> Nếu bạn đang chạy kube-proxy trong một container HostProcess bên trong một Pod, thay vì chạy
> như một Windows Service, bạn có thể nâng cấp kube-proxy bằng cách apply phiên bản mới hơn của
> các manifest kube-proxy.

### Uncordon node (Uncordon the node)

1. Từ một máy có quyền truy cập tới API Kubernetes, đưa node hoạt động trở lại bằng cách đánh
   dấu node là có thể lập lịch (schedulable):

   ```shell
   # thay <node-to-drain> bằng tên node của bạn
   kubectl uncordon <node-to-drain>
   ```

## Tiếp theo (What's next)

* Xem cách [Nâng cấp các node Linux](222-upgrading-linux-nodes-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 17:

1. So với nâng cấp một worker Linux, quy trình trên node Windows **giữ nguyên** những bước nào
   và **thay** những bước nào?
2. **Câu bẫy.** Trên node Windows bạn phải nâng `kube-proxy` bằng một bước riêng. Trên cluster
   lab thuần Linux của bạn, vì sao không có bước tương ứng?
3. Lệnh `drain` và `uncordon` trong bài chạy ở đâu, và vì sao không chạy trên chính node Windows?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. **Giữ nguyên** toàn bộ khung: nâng kubeadm → `kubeadm upgrade node` → drain node → nâng
   kubelet → uncordon, và nguyên tắc nâng control plane trước worker. **Thay đổi** ở cách thực
   thi: không dùng package manager mà **tải binary bằng `curl.exe`** từ `dl.k8s.io`; các thành
   phần là **Windows Service** nên dùng `stop-service`/`restart-service` thay cho `systemctl`;
   và có thêm bước nâng `kube-proxy`.
2. Vì trên cluster Linux kubeadm, **kube-proxy chạy dưới dạng DaemonSet** do control plane quản
   lý — nâng cấp control plane là nó được nâng theo, bạn không đụng tay vào từng node. Trên
   Windows, kube-proxy chạy như một **Windows Service cài trực tiếp trên máy**, nên không có ai
   nâng hộ. Chỗ dễ nhầm là tưởng kube-proxy "vốn dĩ" là thành phần cài trên máy như kubelet;
   thực ra hình thức triển khai của nó khác nhau theo nền tảng.
3. Chúng chạy **từ một máy có quyền truy cập API Kubernetes** — thường là node control plane
   Linux. `drain` và `uncordon` là thao tác **trên object Node qua API server**, không phải thao
   tác trên hệ điều hành của node, nên nơi gõ lệnh không liên quan tới việc node đó chạy Windows
   hay Linux.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng. Nếu cluster của bạn chỉ có Linux,
đánh dấu đã đọc và sang bài [217](217-change-package-repository-vi.md).
