# Quản trị một Cluster (Administer a Cluster)

> Bản dịch tiếng Việt của trang: <https://kubernetes.io/docs/tasks/administer-cluster/>
>
> Tìm hiểu các tác vụ thường gặp để quản trị một cluster.

---

## Đọc bài này thế nào

> Phần này không có trong trang gốc. Nó cho biết ở lần đọc này bạn cần hiểu sâu tới đâu và
> phần nào để dành cho giai đoạn sau. Xem [lộ trình](00-ALO-TRINH-ADMIN.md).

**Vị trí:**
[Giai đoạn 12 — Quản trị cluster nâng cao](00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao)
→ dòng **Thực hành**, bài 1/2 · Kiểm chứng ở
[Lab 12 — Vận hành vòng đời node](labs/LAB-12-VAN-HANH-VONG-DOI-NODE.md) phần B9.1, nơi lab dùng
trang này đúng công dụng của một mục lục: chọn ba mục rồi tìm trong cluster một hiện vật cho
từng mục.

Đây là **trang mục lục**, không phải bài học — và là trang mục lục lớn nhất bạn gặp: gần 50 mục.
Đừng đọc tuần tự từ trên xuống, cũng đừng mở các link ra đọc luôn. Đọc nó như đọc bản đồ.

**Phải hiểu ở lần đọc này:**

- Trang chỉ gồm đoạn mô tả cộng **danh sách các trang con**, xếp theo **thứ tự hiển thị trên
  trang web** — không phải thứ tự học, không phải thứ tự ưu tiên. Bản dịch nói rõ điều đó ngay
  dưới tiêu đề.
- Đây là bản đồ của **phần lớn [Phần II — Vận hành cluster](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)**:
  nâng cấp cluster ([195](195-cluster-upgrade-vi.md)), drain node ([255](255-safely-drain-node-vi.md)),
  vận hành etcd ([197](197-configure-upgrade-etcd-vi.md)), mã hóa dữ liệu khi lưu trữ
  ([208](208-encrypt-data-vi.md)), chia cluster bằng namespace ([242](242-namespaces-tasks-vi.md))
  đều nằm trong danh sách này. Công dụng của trang là **định vị** một tác vụ, rồi đọc bài đó ở
  đúng giai đoạn của nó.
- Ba mục dùng được ngay ở giai đoạn 12 — đúng ba mục Lab 12 chọn ở B9.1:
  [190 — Truy cập cluster bằng Kubernetes API](190-access-cluster-api-vi.md),
  [255 — Drain một node an toàn](255-safely-drain-node-vi.md) (cơ chế nằm dưới `kubectl drain`
  của vòng bảo trì node), và [192 — Đổi StorageClass mặc định](192-change-default-storage-class-vi.md)
  (cluster lab đã có StorageClass từ giai đoạn 6).
- Ranh giới của trang: nó **không chứa hướng dẫn nào**. Kể cả việc bạn đang thiếu ở giai đoạn 12
  — quản lý vòng đời certificate — trang cũng chỉ trỏ sang mục
  [214 — Quản trị với kubeadm](214-kubeadm-tasks-vi.md) và
  [191 — Tạo certificate thủ công](191-certificates-manual-vi.md). Đó là
  ⏳ [nợ #7](00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình), trả ở
  [giai đoạn 18](00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ).

**Đọc lướt, chưa cần hiểu:**

| Phần | Vì sao hoãn | Sẽ hiểu ở |
| --- | --- | --- |
| Phần lớn các mục còn lại trong danh sách | mỗi mục là một tác vụ vận hành độc lập và đã có chỗ đứng riêng trong lộ trình; đọc trước là nhảy cóc | đúng giai đoạn của từng bài — ví dụ [195](195-cluster-upgrade-vi.md) ở [giai đoạn 17](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster), [197](197-configure-upgrade-etcd-vi.md) ở [giai đoạn 19](00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa), [208](208-encrypt-data-vi.md) ở [giai đoạn 22](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu), [192](192-change-default-storage-class-vi.md) ở [giai đoạn 26](00-ALO-TRINH-ADMIN.md#giai-đoạn-26--vận-hành-lưu-trữ) |
| Mục *Di trú khỏi dockershim* — [236](236-migrating-from-dockershim-vi.md) | chỉ áp dụng cho cluster cũ còn dùng dockershim; cluster lab dùng containerd ngay từ đầu | [giai đoạn 27](00-ALO-TRINH-ADMIN.md#giai-đoạn-27--di-chuyển-khỏi-dockershim-cluster-cũ) |
| Ba mục về cloud provider — [254](254-running-cloud-controller-vi.md), [203](203-developing-cloud-controller-manager-vi.md), [198](198-controller-manager-leader-migration-vi.md) | cluster lab không đặt `--cloud-provider` nên không mục nào áp dụng được | bài [198](198-controller-manager-leader-migration-vi.md) là bài 2/2 của chính nhóm này — đọc ngay sau đây để biết vì sao; kiểm chứng ở [Lab 12](labs/LAB-12-VAN-HANH-VONG-DOI-NODE.md) phần B10 |
| Ghi chú "đã có bản dịch trong repo này" gắn ở hai mục | là chú thích của bản dịch về tình trạng kho tài liệu, không phải nội dung của trang gốc | không thuộc giai đoạn nào |

---

Trang gốc là trang mục lục của phần *Tasks → Administer a Cluster*: ngoài phần mô tả ở trên,
nội dung trang là danh sách các trang con, được liệt kê dưới đây theo đúng thứ tự hiển thị
trên trang web.

## Danh sách các trang trong mục này (Pages in this section)

- [Quản trị với kubeadm (Administration with kubeadm)](214-kubeadm-tasks-vi.md)
- [Cấp phát dư dung lượng node cho một cluster (Overprovision Node Capacity For A Cluster)](250-node-overprovisioning-vi.md)
- [Di trú khỏi dockershim (Migrating from dockershim)](236-migrating-from-dockershim-vi.md)
- [Tạo certificate thủ công (Generate Certificates Manually)](191-certificates-manual-vi.md) — đã có bản dịch trong repo này
- [Quản lý tài nguyên bộ nhớ, CPU và API (Manage Memory, CPU, and API Resources)](228-manage-resources-tasks-vi.md)
- [Cài đặt một Network Policy Provider (Install a Network Policy Provider)](243-network-policy-provider-vi.md)
- [Truy cập cluster bằng Kubernetes API (Access Clusters Using the Kubernetes API)](190-access-cluster-api-vi.md) — đã có bản dịch trong repo này
- [Bật hoặc tắt các Feature Gate (Enable Or Disable Feature Gates)](196-configure-feature-gates-vi.md)
- [Quảng bá extended resource cho một node (Advertise Extended Resources for a Node)](209-extended-resource-node-vi.md)
- [Tự động co giãn DNS Service trong một cluster (Autoscale the DNS Service in a Cluster)](206-dns-horizontal-autoscaling-vi.md)
- [Đổi access mode của một PersistentVolume sang ReadWriteOncePod (Change the Access Mode of a PersistentVolume to ReadWriteOncePod)](193-change-pv-access-mode-vi.md)
- [Đổi StorageClass mặc định (Change the default StorageClass)](192-change-default-storage-class-vi.md)
- [Chuyển từ polling sang cập nhật trạng thái container dựa trên sự kiện CRI (Switching from Polling to CRI Event-based Updates to Container Status)](257-switch-to-evented-pleg-vi.md)
- [Đổi reclaim policy của một PersistentVolume (Change the Reclaim Policy of a PersistentVolume)](194-change-pv-reclaim-policy-vi.md)
- [Quản trị Cloud Controller Manager (Cloud Controller Manager Administration)](254-running-cloud-controller-vi.md)
- [Cấu hình một kubelet image credential provider (Configure a kubelet image credential provider)](225-kubelet-credential-provider-vi.md)
- [Cấu hình quota cho các API object (Configure Quotas for API Objects)](252-quota-api-object-vi.md)
- [Kiểm soát các chính sách quản lý CPU trên node (Control CPU Management Policies on the Node)](200-cpu-management-policies-vi.md)
- [Kiểm soát các chính sách quản lý bộ nhớ trên một node (Control Memory Management Policies on a Node)](235-memory-manager-vi.md)
- [Kiểm soát các chính sách quản lý topology trên một node (Control Topology Management Policies on a node)](259-topology-manager-vi.md)
- [Tùy chỉnh DNS Service (Customizing DNS Service)](204-dns-custom-nameservers-vi.md)
- [Gỡ lỗi phân giải DNS (Debugging DNS Resolution)](205-dns-debugging-resolution-vi.md)
- [Khai báo Network Policy (Declare Network Policy)](201-declare-network-policy-vi.md)
- [Phát triển Cloud Controller Manager (Developing Cloud Controller Manager)](203-developing-cloud-controller-manager-vi.md)
- [Bật hoặc tắt một Kubernetes API (Enable Or Disable A Kubernetes API)](207-enable-disable-api-vi.md)
- [Mã hóa dữ liệu bí mật khi lưu trữ (Encrypting Confidential Data at Rest)](208-encrypt-data-vi.md)
- [Giải mã dữ liệu bí mật đã được mã hóa khi lưu trữ (Decrypt Confidential Data that is Already Encrypted at Rest)](202-decrypt-data-vi.md)
- [Bảo đảm lập lịch cho các Pod add-on quan trọng (Guaranteed Scheduling For Critical Add-On Pods)](210-guaranteed-scheduling-critical-addon-pods-vi.md)
- [Hướng dẫn sử dụng IP Masquerade Agent (IP Masquerade Agent User Guide)](212-ip-masq-agent-vi.md)
- [Giới hạn mức tiêu thụ lưu trữ (Limit Storage Consumption)](227-limit-storage-consumption-vi.md)
- [Di trú control plane dạng nhân bản sang dùng Cloud Controller Manager (Migrate Replicated Control Plane To Use Cloud Controller Manager)](198-controller-manager-leader-migration-vi.md)
- [Vận hành các cluster etcd cho Kubernetes (Operating etcd clusters for Kubernetes)](197-configure-upgrade-etcd-vi.md)
- [Dành riêng tài nguyên tính toán cho các daemon hệ thống (Reserve Compute Resources for System Daemons)](253-reserve-compute-resources-vi.md)
- [Chạy các thành phần node của Kubernetes bằng người dùng không phải root (Running Kubernetes Node Components as a Non-root User)](226-kubelet-in-userns-vi.md)
- [Drain một node an toàn (Safely Drain a Node)](255-safely-drain-node-vi.md)
- [Bảo mật một cluster (Securing a Cluster)](256-securing-a-cluster-vi.md)
- [Tăng cường bảo mật Dynamic Resource Allocation trong cluster của bạn (Harden Dynamic Resource Allocation in Your Cluster)](211-hardening-dra-tasks-vi.md)
- [Đặt tham số kubelet qua file cấu hình (Set Kubelet Parameters Via A Configuration File)](224-kubelet-config-file-vi.md)
- [Chia sẻ một cluster bằng namespace (Share a Cluster with Namespaces)](242-namespaces-tasks-vi.md)
- [Nâng cấp một cluster (Upgrade A Cluster)](195-cluster-upgrade-vi.md)
- [Dùng xóa theo tầng trong một cluster (Use Cascading Deletion in a Cluster)](260-use-cascading-deletion-vi.md)
- [Dùng một KMS provider để mã hóa dữ liệu (Using a KMS provider for data encryption)](213-kms-provider-vi.md)
- [Dùng CoreDNS cho khám phá dịch vụ (Using CoreDNS for Service Discovery)](199-coredns-vi.md)
- [Dùng NodeLocal DNSCache trong các cluster Kubernetes (Using NodeLocal DNSCache in Kubernetes Clusters)](251-nodelocaldns-vi.md)
- [Dùng sysctl trong một cluster Kubernetes (Using sysctls in a Kubernetes Cluster)](258-sysctl-cluster-vi.md)
- [Xác minh các artifact Kubernetes đã ký (Verify Signed Kubernetes Artifacts)](261-verify-signed-artifacts-vi.md)

---

## Tự kiểm tra

> Phần này không có trong trang gốc.

Trả lời được các câu sau mà không nhìn lại bài là đủ cho lần đọc ở giai đoạn 12:

1. Ngoài đoạn mô tả, trang này gồm những gì? Thứ tự các mục trong danh sách nói lên điều gì — và
   **không** nói lên điều gì?
2. **Câu bẫy.** Bạn đang ở giai đoạn 12 và thấy trong danh sách có "Nâng cấp một cluster",
   "Vận hành các cluster etcd cho Kubernetes" và "Mã hóa dữ liệu bí mật khi lưu trữ". Cả ba đều
   là việc của quản trị viên cluster. Vậy có nên đọc luôn không?
3. Ở [Lab 12](labs/LAB-12-VAN-HANH-VONG-DOI-NODE.md) bạn dùng trang này để tìm trong cluster
   `lab-k8s-master` một hiện vật cho ba mục. Ba mục nào, và vì sao đúng ba mục đó là những mục
   dùng được ngay ở giai đoạn 12?

<details>
<summary>Đáp án — chỉ mở sau khi đã tự trả lời</summary>

1. Gồm **danh sách các trang con** và không gì khác — bản dịch ghi rõ "ngoài phần mô tả ở trên,
   nội dung trang là danh sách các trang con". Thứ tự chỉ nói lên **thứ tự hiển thị trên trang
   web**. Nó **không** nói lên thứ tự học, mức độ ưu tiên, hay quan hệ phụ thuộc giữa các tác vụ.
   Thứ tự học nằm ở [lộ trình](00-ALO-TRINH-ADMIN.md), không nằm ở đây.
2. **Không.** Đây là chỗ dễ sai: trang trông như một mục lục giáo trình nên dễ tưởng là danh sách
   việc phải làm. Thật ra nó là **bản đồ của gần như toàn bộ
   [Phần II](00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster)**, mỗi mục có giai đoạn riêng:
   nâng cấp cluster ở [giai đoạn 17](00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster), etcd ở
   [giai đoạn 19](00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa), mã hóa
   at rest ở [giai đoạn 22](00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu). Đọc
   trước là nhảy cóc; việc đúng ở giai đoạn 12 là **biết chúng nằm ở đây** để sau này tìm lại.
3. Ba mục: [190 — Truy cập cluster bằng Kubernetes API](190-access-cluster-api-vi.md),
   [255 — Drain một node an toàn](255-safely-drain-node-vi.md), và
   [192 — Đổi StorageClass mặc định](192-change-default-storage-class-vi.md). Đúng ba mục đó dùng
   được ngay vì **cluster lab đã có sẵn thứ để soi**: API server vẫn đang phục vụ nên đường truy
   cập API kiểm chứng được; vòng bảo trì node của giai đoạn 12 chạy thẳng qua `kubectl drain` nên
   cơ chế của bài 255 nằm ngay dưới thao tác bạn làm; và StorageClass đã tồn tại từ giai đoạn 6
   nên khái niệm "class mặc định" có object thật để chỉ vào. Các mục còn lại thiếu điều kiện đó ở
   thời điểm này.

</details>

Câu nào chưa trả lời được thì quay lại đúng mục tương ứng trước khi đọc bài sau.
