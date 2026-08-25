# Nâng cấp node Linux (Upgrading Linux nodes)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/upgrading-linux-nodes/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Checkpoint tiếp nối — nhánh `/docs/tasks/`](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
→ [Giai đoạn 17 — Nâng cấp cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster), bài 2/5 · Kiểm chứng trên
cluster lab: nâng cấp `k8s-worker1` rồi `k8s-worker2` sau khi đã nâng `k8s-master`.

Bài rất ngắn: nó chỉ là **phần thực thi trên node worker Linux** của quy trình đã học ở bài
[221](221-kubeadm-upgrade-vi.md). Đọc bài 221 trước, nếu không các lệnh ở đây sẽ trông rời rạc.

Con số `1.36.x` trong bài là ví dụ của trang gốc. Phiên bản của cluster lab nằm ở [bảng A1.3 của Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa).

**Phải hiểu ở lần đọc này:**

- **Thứ tự trong bài chính là thứ tự phải làm:** nâng gói `kubeadm` → `kubeadm upgrade node` →
  `kubectl drain` → nâng `kubelet` và `kubectl` → restart kubelet → `kubectl uncordon`.
- Trên worker, `kubeadm upgrade node` chỉ làm một việc: **nâng cấu hình kubelet cục bộ**. Nó
  không nâng binary nào cả.
- Vai trò của `apt-mark unhold` / `hold` (và `--disableexcludes=kubernetes` với yum): các gói
  Kubernetes bị **ghim có chủ đích** để không bị nâng ngoài ý muốn; phải mở ghim để nâng rồi ghim
  lại ngay.
- Hai lệnh chạy **trên control plane, không phải trên worker**: `kubectl drain` và
  `kubectl uncordon`. Các lệnh còn lại chạy trên chính node worker.
- `systemctl daemon-reload` trước `systemctl restart kubelet` — bỏ bước này thì unit file mới
  không được nạp.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Cú pháp cho CentOS/RHEL/Fedora (DNF và DNF5) | cluster lab dùng Ubuntu theo Lab 00 | khi vận hành cluster trên distro họ Red Hat |
| Chi tiết đổi package repository | là một bước riêng, có bài riêng | bài [217](217-change-package-repository-vi.md), bài 4/5 của giai đoạn 17 |

---

Trang này giải thích cách nâng cấp các node worker Linux được tạo bằng kubeadm.

## Trước khi bạn bắt đầu (Before you begin)

Bạn cần có quyền truy cập shell vào tất cả các node, và công cụ dòng lệnh kubectl phải được
cấu hình để giao tiếp với cluster của bạn. Khuyến nghị chạy hướng dẫn này trên một cluster có
ít nhất hai node không đóng vai trò control plane. Để kiểm tra phiên bản, nhập `kubectl version`.

* Hãy làm quen với [quy trình nâng cấp phần còn lại của cluster kubeadm](221-kubeadm-upgrade-vi.md).
  Bạn nên nâng cấp các node control plane trước khi nâng cấp các node worker Linux.

## Thay đổi kho gói (Changing the package repository)

Nếu bạn đang dùng các kho gói do cộng đồng sở hữu (`pkgs.k8s.io`), bạn cần kích hoạt kho gói
cho bản phát hành minor Kubernetes mong muốn. Điều này được giải thích trong tài liệu
[Thay đổi kho gói Kubernetes](217-change-package-repository-vi.md).

> **Ghi chú:** Các kho gói cũ (`apt.kubernetes.io` và `yum.kubernetes.io`) đã bị
> [ngưng sử dụng và đóng băng kể từ ngày 13-09-2023](https://kubernetes.io/blog/2023/08/31/legacy-package-repository-deprecation/).
> **Việc sử dụng [các kho gói mới được lưu trữ tại `pkgs.k8s.io`](https://kubernetes.io/blog/2023/08/15/pkgs-k8s-io-introduction/)
> được khuyến nghị mạnh mẽ và là bắt buộc để cài đặt các phiên bản Kubernetes phát hành sau ngày 13-09-2023.**
> Các kho cũ đã ngưng sử dụng, cùng nội dung của chúng, có thể bị xóa bất cứ lúc nào trong tương
> lai mà không cần thông báo trước. Các kho gói mới cung cấp bản tải về cho các phiên bản
> Kubernetes bắt đầu từ v1.24.0.

## Nâng cấp các node worker (Upgrading worker nodes)

### Nâng cấp kubeadm (Upgrade kubeadm)

Nâng cấp kubeadm:

#### Ubuntu, Debian hoặc HypriotOS

```shell
# thay x trong 1.36.x-* bằng phiên bản vá mới nhất
sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.36.x-*' && \
sudo apt-mark hold kubeadm
```

#### CentOS, RHEL hoặc Fedora

Với các hệ thống dùng DNF:
```shell
# thay x trong 1.36.x-* bằng phiên bản vá mới nhất
sudo yum install -y kubeadm-'1.36.x-*' --disableexcludes=kubernetes
```
Với các hệ thống dùng DNF5:
```shell
# thay x trong 1.36.x-* bằng phiên bản vá mới nhất
sudo yum install -y kubeadm-'1.36.x-*' --setopt=disable_excludes=kubernetes
```

### Gọi "kubeadm upgrade" (Call "kubeadm upgrade")

Đối với các node worker, lệnh này nâng cấp cấu hình kubelet cục bộ:

```shell
sudo kubeadm upgrade node
```

### Drain node (Drain the node)

Chuẩn bị node cho việc bảo trì bằng cách đánh dấu node là không thể lập lịch (unschedulable)
và trục xuất (evict) các workload:

```shell
# chạy lệnh này trên một node control plane
# thay <node-to-drain> bằng tên của node mà bạn đang drain
kubectl drain <node-to-drain> --ignore-daemonsets
```

### Nâng cấp kubelet và kubectl (Upgrade kubelet and kubectl)

1. Nâng cấp kubelet và kubectl:

   #### Ubuntu, Debian hoặc HypriotOS

   ```shell
   # thay x trong 1.36.x-* bằng phiên bản vá mới nhất
   sudo apt-mark unhold kubelet kubectl && \
   sudo apt-get update && sudo apt-get install -y kubelet='1.36.x-*' kubectl='1.36.x-*' && \
   sudo apt-mark hold kubelet kubectl
   ```

   #### CentOS, RHEL hoặc Fedora

   Với các hệ thống dùng DNF:
   ```shell
   # thay x trong 1.36.x-* bằng phiên bản vá mới nhất
   sudo yum install -y kubelet-'1.36.x-*' kubectl-'1.36.x-*' --disableexcludes=kubernetes
   ```
   Với các hệ thống dùng DNF5:
   ```shell
   # thay x trong 1.36.x-* bằng phiên bản vá mới nhất
   sudo yum install -y kubelet-'1.36.x-*' kubectl-'1.36.x-*' --setopt=disable_excludes=kubernetes
   ```

1. Khởi động lại kubelet:

   ```shell
   sudo systemctl daemon-reload
   sudo systemctl restart kubelet
   ```

### Uncordon node (Uncordon the node)

Đưa node trở lại hoạt động bằng cách đánh dấu node là có thể lập lịch (schedulable):

```shell
# chạy lệnh này trên một node control plane
# thay <node-to-uncordon> bằng tên node của bạn
kubectl uncordon <node-to-uncordon>
```

## Tiếp theo (What's next)

* Xem cách [Nâng cấp node Windows](223-upgrading-windows-nodes-vi.md).

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 17:

1. Trên `k8s-worker2`, bạn chạy `kubeadm upgrade node` và lệnh báo thành công. `kubectl get nodes`
   vẫn hiển thị worker đó ở phiên bản cũ. Bạn đã bỏ sót bước nào?
2. **Câu bẫy.** Trong toàn bộ quy trình nâng cấp một worker, những lệnh nào phải chạy **trên
   control plane** chứ không phải trên chính worker đó?
3. Vì sao các gói `kubeadm`, `kubelet`, `kubectl` bị `apt-mark hold`, và vì sao phải ghim lại
   ngay sau khi nâng?
4. Nếu bạn nâng `kubelet` nhưng quên `systemctl daemon-reload` trước khi restart, hậu quả là gì?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Bỏ sót bước **nâng chính gói `kubelet`** (và `kubectl`) bằng package manager, rồi
   `daemon-reload` + `restart kubelet`. Trên node worker, `kubeadm upgrade node` **chỉ nâng cấu
   hình kubelet cục bộ**, nó không thay binary. Phiên bản mà `kubectl get nodes` hiển thị là
   phiên bản kubelet đang chạy.
2. **`kubectl drain <node> --ignore-daemonsets` và `kubectl uncordon <node>`** — bài ghi rõ
   "chạy lệnh này trên một node control plane". Chúng là thao tác qua API server, không phải
   thao tác trên máy. Mọi lệnh còn lại (`apt-get install`, `kubeadm upgrade node`,
   `systemctl restart kubelet`) chạy tại chỗ trên worker. Chỗ dễ nhầm: cả quy trình đọc như một
   mạch liên tục nên dễ tưởng gõ hết ở một nơi.
3. Vì chúng bị **ghim có chủ đích** để một lệnh `apt-get upgrade` thông thường không vô tình nâng
   Kubernetes lên phiên bản khác — điều đó sẽ phá vỡ version skew và có thể làm node rời cluster.
   Phải mở ghim (`unhold`) để nâng đúng phiên bản mình chọn, rồi `hold` lại ngay để khôi phục lớp
   bảo vệ đó.
4. `systemd` vẫn dùng **unit file cũ đã nạp trong bộ nhớ**. Nếu bản nâng cấp có thay đổi unit file
   hoặc drop-in của kubelet, thay đổi đó không có hiệu lực; kubelet khởi động lại với cấu hình
   cũ và có thể lệch với cấu hình mà `kubeadm upgrade node` vừa ghi ra.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi sang bài kế của [Giai đoạn 17 — Nâng cấp cluster](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster).
