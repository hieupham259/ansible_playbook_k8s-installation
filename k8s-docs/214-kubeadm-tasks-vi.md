# Quản trị với kubeadm (Administration with kubeadm)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/>

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:** [Giai đoạn 8 — Dựng cluster bằng kubeadm](00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm)
→ dòng **Thực hành**, bài 1/1 · Kiểm chứng ở
[Lab 8a — Dựng cluster bằng kubeadm](labs/LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md) phần **B10** (định vị
các việc quản trị còn lại). Phần HA của giai đoạn thuộc
[Lab 8b](labs/LAB-8B-HA-VOI-STACKED-ETCD.md) và [Lab 8c](labs/LAB-8C-HA-VOI-EXTERNAL-ETCD.md), không
dùng trang này.

Đây là **trang mục lục**, không phải bài học: ngoài đoạn dẫn nhập, toàn bộ nội dung là danh sách
chín trang con. Đọc nó để biết có những việc gì và tra ở đâu, đừng mở lần lượt từng trang con —
lộ trình đã chia chúng vào các giai đoạn sau.

**Phải hiểu ở lần đọc này:**

- Câu phân định ở đầu trang: các tác vụ trong mục này dành cho người **đang quản trị một cluster
  có sẵn**; ai chưa có cluster thì trang chỉ ngược về nhánh *dựng cluster với `kubeadm`*. Đó đúng
  là ranh giới giữa giai đoạn 8 (tạo cluster) và [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)
  (sống cùng cluster).
- Mục *Danh sách các trang trong mục này* liệt kê chín việc quản trị: thêm node Linux và Windows,
  nâng cấp cluster và nâng cấp node, cgroup driver, certificate, cấu hình lại cluster, đổi package
  repository. Nhớ **có việc gì** là đủ; cách làm để dành cho đúng giai đoạn.
- Trong chín trang đó chỉ hai trang nằm trong tầm giai đoạn 8: [215](215-adding-linux-nodes-vi.md)
  (thêm worker node Linux — chính là `kubeadm join` bạn vừa làm) và
  [218](218-configure-cgroup-driver-vi.md) (cgroup driver — thứ bạn đã khai lúc cài và lúc `init`).
  Bảy trang còn lại chưa có chỗ đứng ở giai đoạn này.

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| [216 — Thêm worker node Windows](216-adding-windows-nodes-vi.md) | cần một VM Windows Server, cluster lab chỉ có ba VM Linux | [giai đoạn 15](00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows), [Lab 15](labs/LAB-15-NODE-WINDOWS.md) |
| [221 — Nâng cấp cluster kubeadm](221-kubeadm-upgrade-vi.md), [222 — Nâng cấp node Linux](222-upgrading-linux-nodes-vi.md), [223 — Nâng cấp node Windows](223-upgrading-windows-nodes-vi.md), [217 — Đổi package repository](217-change-package-repository-vi.md) | chạy `kubeadm upgrade` bây giờ sẽ phá bộ phiên bản đã khóa của cluster lab | [giai đoạn 17](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster) |
| [219 — Quản lý certificate với kubeadm](219-kubeadm-certs-vi.md) | Lab 8a mới **đọc hạn** ở B2.2 và B10; gia hạn và xoay CA là việc khác hẳn | [giai đoạn 18](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) |
| [220 — Cấu hình lại một cluster kubeadm](220-kubeadm-reconfigure-vi.md), và phần *đổi cgroup driver trên cluster đang chạy* của [218](218-configure-cgroup-driver-vi.md) | giai đoạn 8 đổi cấu hình bằng cách **dựng lại**; sửa cấu hình của cluster đang chạy là quy trình khác | [giai đoạn 20](00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| [215 — Thêm worker node Linux](215-adding-linux-nodes-vi.md) ở góc **vòng đời node** | ở đây bạn mới join node vào một cluster vừa dựng, chưa đặt nó cạnh cordon/drain/remove | [giai đoạn 16](00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node) |

---

Nếu bạn chưa có cluster, hãy xem
[dựng cluster với `kubeadm`](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/).

Các tác vụ trong mục này dành cho những người đang quản trị một cluster có sẵn:

Trang gốc là trang mục lục của phần *Tasks → Administer a Cluster → Administration with
kubeadm*: ngoài phần dẫn nhập ở trên, nội dung trang là danh sách các trang con, được liệt kê
dưới đây theo đúng thứ tự hiển thị trên trang web.

## Danh sách các trang trong mục này (Pages in this section)

- [Thêm worker node Linux (Adding Linux worker nodes)](215-adding-linux-nodes-vi.md)
- [Thêm worker node Windows (Adding Windows worker nodes)](216-adding-windows-nodes-vi.md)
- [Nâng cấp cluster kubeadm (Upgrading kubeadm clusters)](221-kubeadm-upgrade-vi.md)
- [Nâng cấp các node Linux (Upgrading Linux nodes)](222-upgrading-linux-nodes-vi.md)
- [Nâng cấp các node Windows (Upgrading Windows nodes)](223-upgrading-windows-nodes-vi.md)
- [Cấu hình cgroup driver (Configuring a cgroup driver)](218-configure-cgroup-driver-vi.md)
- [Quản lý certificate với kubeadm (Certificate Management with kubeadm)](219-kubeadm-certs-vi.md)
- [Cấu hình lại một cluster kubeadm (Reconfiguring a kubeadm cluster)](220-kubeadm-reconfigure-vi.md)
- [Thay đổi package repository của Kubernetes (Changing The Kubernetes Package Repository)](217-change-package-repository-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 8:

1. Đoạn dẫn nhập của trang phân định đối tượng đọc bằng đúng một câu. Câu đó nói gì, và vì sao nó
   giải thích được chỗ đứng của trang này ở **cuối** giai đoạn 8 chứ không phải ở đầu?
2. **Câu bẫy.** Trang tên là *Quản trị với kubeadm* và nằm trong nhánh Tasks của kubernetes.io.
   Nếu bạn chưa có cluster nào, đọc hết trang này có dựng được cluster không — và trang bảo bạn
   đi đâu?
3. Cluster lab của bạn là ba VM Linux `lab-k8s-master`, `lab-k8s-worker1`, `lab-k8s-worker2`.
   Trong chín trang con, trang nào **không** áp dụng được cho cluster này và vì sao; ngược lại,
   trang nào bạn đã đi qua trên thực tế khi join hai worker?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Câu đó là: **các tác vụ trong mục này dành cho những người đang quản trị một cluster có sẵn**.
   Nó tách hai loại việc: *tạo* cluster và *sống cùng* cluster. Giai đoạn 8 làm việc thứ nhất, nên
   trang này chỉ có nghĩa **sau khi** cluster đã chạy — nó là bản đồ những việc sẽ tới, không phải
   hướng dẫn để bắt đầu.
2. **Không.** Trang mở đầu bằng đúng lời dặn ngược lại: *nếu bạn chưa có cluster, hãy xem dựng
   cluster với `kubeadm`* — tức nhánh *setup*, không phải nhánh *tasks*. Chỗ dễ nhầm là chữ
   "kubeadm" trong tiêu đề khiến người đọc tưởng đây là trang cài đặt; thực ra ngoài đoạn dẫn nhập
   thì **toàn bộ nội dung trang là một danh sách liên kết**, không có một lệnh nào.
3. **Hai trang Windows không áp dụng được**: [216 — Thêm worker node Windows](216-adding-windows-nodes-vi.md)
   và [223 — Nâng cấp các node Windows](223-upgrading-windows-nodes-vi.md), vì cả ba VM đều là
   Linux nên không có node Windows nào để thêm hay để nâng cấp. Ngược lại,
   **[215 — Thêm worker node Linux](215-adding-linux-nodes-vi.md) là trang bạn đã đi qua**: đó
   chính là thao tác join `lab-k8s-worker1` và `lab-k8s-worker2` vào control plane.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng, rồi mở
[Lab 8a — Dựng cluster bằng kubeadm](labs/LAB-8A-DUNG-CLUSTER-BANG-KUBEADM.md) — trang này được
dùng lại ở phần B10 của lab.
