# Lộ trình học Kubernetes Administrator

Giáo trình đọc **398 bài dịch** trong thư mục này theo thứ tự dành cho người muốn trở thành Kubernetes administrator. Chúng chia làm hai phần:

- **186 bài khái niệm** (số `00`–`185`, nhánh `/docs/concepts/` và `/docs/setup/`) — mạch chính, giai đoạn 1 đến 15 dưới đây.
- **212 bài thực hành** (số `186` trở lên, nhánh `/docs/tasks/`) — đã dịch xong và **nằm ngay trong mạch chính**: mỗi giai đoạn có khối **Thực hành** đặt sau phần lý thuyết và trước lab. Riêng nhóm vận hành cluster nằm ở [Checkpoint tiếp nối](#phần-ii--vận-hành-cluster) cuối file. Danh mục theo chủ đề ở [Phần 15–23 của README](README.md).

> **Số trong tên file KHÔNG phải thứ tự đọc.** Số chỉ là mã định danh bám theo cấu trúc mục của kubernetes.io (để dễ đối chiếu khi trang gốc cập nhật). Thứ tự đọc là thứ tự các bài xuất hiện trong file này. Xem [README.md](README.md) nếu muốn tra cứu theo chủ đề thay vì theo lộ trình.

Mỗi giai đoạn có **Mục tiêu** (học xong phải trả lời được gì) và **Checkpoint** (phải làm được gì trước khi sang giai đoạn sau). Đọc mà không làm checkpoint thì kiến thức không trụ được.

**Theo dõi tiến độ:** giữ `[ ]` khi chưa đọc và đổi thành `[x]` sau khi đã đọc xong
bài tương ứng. Các trang `/docs/tasks/` ở phần cuối cũng có checkbox riêng; chỉ đánh dấu
hoàn tất sau khi đã thực hành trên cluster lab, không chỉ sau khi đọc trang hướng dẫn.

**Mục có dấu 🧪 là bài lab**, đặt ở cuối nhóm bài mà nó kiểm chứng — làm lab sau khi đã đọc
hết nhóm bài đứng trên nó. Bản đồ lab, chuỗi snapshot và sổ nợ lab nằm ở
[labs/README.md](labs/README.md).

Phần thực hành vận hành thực tế (upgrade, backup etcd, drain node, xử lý sự cố…) nằm ở [Checkpoint tiếp nối](#phần-ii--vận-hành-cluster) ở cuối file.

---

## Sổ nợ lộ trình

Tám phần dưới đây **cố ý chưa thực hành ngay tại chỗ đọc**, vì thứ cần cho chúng thuộc giai
đoạn sau. Bảng này để bạn không bỏ sót: đọc xong một bài có nợ thì biết nợ đó nằm ở đâu, và
khi tới chỗ trả thì biết phải quay lại đọc gì.

Trong toàn file, hai dấu sau đánh ngay tại chỗ:

- **⏳ Nợ #N** — đặt ở nơi nợ phát sinh, nói rõ phần nào bị hoãn và trả ở đâu.
- **✅ Trả nợ #N** — đặt ở nơi nợ được trả, kèm yêu cầu đọc lại bài gốc trước khi làm.

Grep `⏳ Nợ` để thấy toàn bộ chỗ phát sinh, `✅ Trả nợ` để thấy toàn bộ chỗ trả.

| # | Món nợ | Phát sinh ở | Bị chặn bởi | Trả ở |
| --- | --- | --- | --- | --- |
| 1 | Thực hành HPA và VPA | [Giai đoạn 4](#giai-đoạn-4--workload-controller), bài [72](72-horizontal-pod-autoscale-vi.md), [73](73-vertical-pod-autoscale-vi.md) | metrics-server (giai đoạn 11) | [Lab 11b](#giai-đoạn-11--observability) |
| 2 | `volumeClaimTemplates` của StatefulSet | [Giai đoạn 4](#giai-đoạn-4--workload-controller), bài [65](65-statefulset-vi.md) | StorageClass + provisioner (giai đoạn 6) | [Lab 6a](#giai-đoạn-6--lưu-trữ) |
| 3 | Service headless quản trị cho StatefulSet | [Giai đoạn 4](#giai-đoạn-4--workload-controller), bài [65](65-statefulset-vi.md) | Service headless (giai đoạn 5) | [Lab 5a](#giai-đoạn-5--mạng-nền-tảng) |
| 4 | NetworkPolicy được thực thi thật | [Giai đoạn 5](#giai-đoạn-5--mạng-nền-tảng), bài [84](84-network-policies-vi.md) | CNI hỗ trợ policy thay Flannel | [Lab 5b](#giai-đoạn-5--mạng-nền-tảng) |
| 5 | Ảnh chụp nhanh và nhân bản volume | [Giai đoạn 6](#giai-đoạn-6--lưu-trữ), bài [99](99-volume-snapshots-vi.md)–[101](101-volume-pvc-datasource-vi.md) | **CSI driver hỗ trợ snapshot** — `local-path-provisioner` của baseline dùng `rancher.io/local-path`, không phải CSI, nên **chưa đủ điều kiện** | Lab 6b, nhánh A — chạy lại sau khi cluster có CSI driver hỗ trợ snapshot |
| 6 | Mã hóa Secret at rest | [Giai đoạn 3b](#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod), bài [109](109-secret-vi.md) | sửa cấu hình apiserver | [giai đoạn 22](#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) |
| 7 | Quản lý vòng đời certificate | [Giai đoạn 12](#giai-đoạn-12--quản-trị-cluster-nâng-cao), bài [156](156-certificates-vi.md) | quy trình `kubeadm certs` | [giai đoạn 18](#giai-đoạn-18--vòng-đời-chứng-chỉ) |
| 8 | Backup và restore etcd | [Giai đoạn 8](#giai-đoạn-8--dựng-cluster-bằng-kubeadm) | `etcdctl` và quy trình khôi phục | [giai đoạn 19](#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa) |
| 9 | Hai khối *Đọc bài này thế nào* và *Tự kiểm tra* cho 164 bài nhánh `/docs/tasks/` | mọi dòng có dấu ⏳ — danh sách nguồn dưới mỗi 🧪 lab, và [giai đoạn 16–27](#phần-ii--vận-hành-cluster) | công sức viết, không phải kiến thức — bài đọc được ngay | **trả tại chỗ**, ngay trước khi đọc bài mang dấu ⏳ |

**Quy tắc:** không đánh dấu một giai đoạn là xong khi nợ của nó chưa trả. Nợ #1–#5 trả trong
phần lab (giai đoạn 5, 6, 11); nợ #6–#8 trả ở Checkpoint tiếp nối cuối file; nợ #9 trả rải rác,
ngay tại chỗ. Bảng tương ứng phía lab nằm ở [sổ nợ lab](labs/README.md#5-sổ-nợ-lab) — hai bảng
phải khớp nhau.

### Nợ #9 — Hai khối hướng dẫn đọc cho nhánh `/docs/tasks/`

186 bài khái niệm (số `00`–`185`) đều đã có hai khối **Đọc bài này thế nào** và
**Tự kiểm tra**. Nhánh thực hành `/docs/tasks/` thì chưa: **164/212 bài còn thiếu**.

**Dấu hiệu nhận biết:** dòng bài kết thúc bằng **⏳**. Mỗi mục có bài thiếu đều mở đầu bằng
một dòng đếm `⏳ Nợ #9 — N/M bài…`, nên bạn biết trước khi vào mục đó còn bao nhiêu bài chưa
có phần định hướng.

**Nợ này khác bảy nợ trên.** Nợ #1–#8 là *chưa thực hành được* vì thiếu hạ tầng hoặc thiếu
kiến thức của giai đoạn sau. Nợ #9 chỉ là *chưa viết xong phần phụ trợ*: bản dịch đã đầy đủ và
đọc được ngay, chỉ thiếu phần nói cho bạn biết **cần hiểu sâu tới đâu ở lần đọc này** và bộ câu
tự kiểm tra. Vì vậy nó **không chặn** việc học — không được lấy nó làm cớ bỏ qua bài.

**Cách trả:** khi lộ trình dẫn bạn tới một bài mang dấu ⏳, viết hai khối cho đúng bài đó rồi
mới đọc. Viết ngay lúc sắp học là đúng thời điểm nhất — bạn đang ở đúng ngữ cảnh mà khối đó
phục vụ. Quy tắc về khuôn, vị trí đặt khối và yêu cầu với câu hỏi nằm ở mục
*Khối "Đọc bài này thế nào" và "Tự kiểm tra"* trong [CLAUDE.md](../CLAUDE.md). Hai bài dùng làm
mẫu: [197](197-configure-upgrade-etcd-vi.md) và [221](221-kubeadm-upgrade-vi.md).

**Bắt buộc:** đọc trọn bài trước khi viết khối cho nó. Không suy nội dung từ tên file hay từ
kiến thức chung về Kubernetes — mọi mục được nhắc tên trong khối phải có thật trong bài.

Kiểm tra còn bao nhiêu bài chưa trả:

```bash
cd k8s-docs && for f in *-vi.md; do n=${f%%-*}; [ "$n" -gt 185 ] 2>/dev/null || continue; \
  grep -q "^## Đọc bài này thế nào" "$f" || echo "$f"; done | wc -l
```


## Môi trường lab

Lộ trình này yêu cầu một cluster thật để thực hiện checkpoint ngay từ Giai đoạn 1.
Không sử dụng minikube. Chọn một trong hai phương án sau:

> **Runbook lab:** thư mục [labs/](labs/README.md) chứa bài thực hành chạy được cho từng nhóm
> bài, kèm bản đồ lab, chuỗi snapshot và sổ nợ lab. Phần dựng môi trường nằm một chỗ duy nhất
> ở [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md); mọi lab khác bắt đầu từ một snapshot có tên.

### Phương án 1 — Dùng cluster đã dựng

Có thể dùng cluster Kubernetes đã dựng trước đó nếu bạn có toàn quyền quản trị và
cluster không phục vụ production hoặc workload quan trọng. Cluster cần đáp ứng tối thiểu:

- Có ít nhất một control plane node và một worker node.
- Được dựng bằng kubeadm hoặc có kiến trúc đủ tương đồng để quan sát control plane,
  kubelet, container runtime, CNI, CoreDNS và kube-proxy.
- CNI đang hoạt động và có hỗ trợ NetworkPolicy để làm checkpoint Giai đoạn 5.
- Có StorageClass và dynamic provisioner hoạt động để làm checkpoint Giai đoạn 6.
- Có quyền SSH vào node, quyền `sudo` và kubeconfig có quyền quản trị cluster.
- Có thể chủ động cordon, drain, restart component, thay đổi cấu hình và khôi phục cluster.

Không thực hiện các bài phá lỗi, nâng cấp, restore etcd, xoay certificate hoặc thay đổi
control plane trên cluster đang phục vụ người dùng thật.

### Phương án 2 — Dựng một cluster lab khác tương tự

Nếu cluster hiện tại đang được sử dụng hoặc không được phép thử nghiệm phá lỗi, hãy dựng
một cluster riêng có kiến trúc tương tự. Bản dựng từng bước có sẵn ở
[Lab 00 — Môi trường](labs/LAB-00-MOI-TRUONG-1.35.7.md). Nên dùng các máy ảo để có thể snapshot và
khôi phục:

- Tối thiểu cho phần lớn bài học: 1 control plane + 2 worker.
- Cho bài HA: 3 control plane + 2 worker và một load balancer phía trước API server.
- Nếu học external etcd: bổ sung 3 node etcd riêng hoặc một nhóm VM tách biệt tương đương.
- Dùng cùng hệ điều hành, container runtime, CNI và dải mạng gần giống môi trường cần quản trị.
- Tạo snapshot VM trước các bài upgrade, restore etcd, certificate và troubleshooting phá lỗi.

Có thể tạo lại cluster nhiều lần trong quá trình học. Việc dựng, phá và khôi phục cluster
là một phần của bài thực hành, không phải thao tác chuẩn bị chỉ làm một lần.

### Quy ước an toàn

- Dùng namespace riêng cho bài tập workload thông thường.
- Ghi lại trạng thái ban đầu và sao lưu manifest/cấu hình trước khi chỉnh sửa.
- Chỉ làm bài gây gián đoạn trên cluster lab hoặc cluster có thể bỏ và dựng lại.
- Không đưa credential, private key, Secret hoặc snapshot etcd thật vào repository.
- Với bài làm đầy disk, chỉ dùng filesystem/volume dành riêng cho lab và đặt giới hạn rõ ràng;
  không làm đầy filesystem của máy host.

- [X] 🧪 [Lab 00 — Dựng môi trường lab dùng chung](labs/LAB-00-MOI-TRUONG-1.35.7.md) — làm trước giai đoạn 1; kết thúc bằng snapshot `01-cluster-ready`.

**Checkpoint môi trường:** chạy được `kubectl get nodes`, tất cả node ở trạng thái `Ready`;
SSH được vào từng node; xác định được container runtime, CNI, CoreDNS, kube-proxy và
StorageClass đang sử dụng; đồng thời có phương án snapshot hoặc dựng lại cluster khi bị hỏng.

StorageClass chỉ xuất hiện từ Lab 6a; trước đó phần đó của checkpoint chưa áp dụng.

---

## Phần I — Nền tảng Kubernetes

Mười sáu giai đoạn xây nền: mô hình Kubernetes, workload, mạng, lưu trữ, lập lịch, bảo mật, và dựng cluster bằng kubeadm. Mỗi giai đoạn có **Mục tiêu**, danh sách bài đọc, bài thực hành và **Checkpoint** vấn đáp.

### Giai đoạn 0 — Kiến thức nền ngoài thư mục này

Không có tài liệu trong thư mục. Thiếu phần này thì mọi giai đoạn sau đều học vẹt.

- **Linux**: systemd (unit, `systemctl`, `journalctl`), process, filesystem, permission, user/group.
- **Mạng**: TCP/IP, subnet, routing, DNS, NAT, firewall (iptables/nftables), load balancer L4 vs L7.
- **YAML và JSON**: cấu trúc lồng nhau, list vs map, multi-document (`---`).
- **TLS**: certificate, CA, chain of trust, SAN, thời hạn và gia hạn.
- **Công cụ**: `bash`, `curl`, `openssl`, `ss`, `ip`, `journalctl`, `systemctl`, `dig`.

**Checkpoint:** tự dựng được 2 máy Linux nối mạng với nhau, tự ký một certificate bằng `openssl` và giải thích được vì sao trình duyệt tin/không tin nó.

---

### Giai đoạn 1 — Mô hình Kubernetes

**Mục tiêu:** hiểu control plane gồm gì, API server đóng vai trò gì, object và desired state là gì, kubectl nói chuyện với cluster ra sao.

#### 1a. Kiến trúc và mô hình điều khiển

- [X] [Tổng quan](14-overview-vi.md) — Kubernetes giải quyết bài toán gì.
- [X] [Các thành phần của Kubernetes](15-components-vi.md) — trọng tâm: phân biệt thành phần control plane và thành phần chạy trên mọi node.
- [X] [Kiến trúc cluster](22-architecture-vi.md) — ghép các thành phần ở bài 2 thành bức tranh tổng thể.
- [X] [Các đối tượng trong Kubernetes](16-working-with-objects-vi.md) — trọng tâm: `spec` (mong muốn) vs `status` (thực tế).
- [X] [Kubernetes API](21-kubernetes-api-vi.md) — trọng tâm: API group, versioning (alpha/beta/stable); phần API aggregation đọc lướt, sẽ quay lại ở giai đoạn 14.
- [X] [Các Node](23-nodes-vi.md) — trọng tâm: kubelet đăng ký node, node condition, heartbeat.
- [X] [Giao tiếp giữa Node và Control Plane](24-control-plane-node-communication-vi.md) — chiều giao tiếp nào đi qua API server, chiều nào không.
- [X] [Các Controller](25-controllers-vi.md) — vòng lặp điều khiển; đây là ý tưởng cốt lõi của toàn bộ Kubernetes.
- [X] 🧪 [Lab 1a — Kiến trúc và mô hình điều khiển](labs/LAB-1A-KIEN-TRUC-VA-MO-HINH-DIEU-KHIEN.md) — quan sát component/API/Node và thực hành reconciliation. Cần [Lab 00](labs/LAB-00-MOI-TRUONG-1.35.7.md) xong trước.

#### 1b. Làm việc với object và kubectl

- [X] [Tên và ID của đối tượng](17-names-vi.md) — quy tắc đặt tên DNS subdomain/label, UID.
- [X] [Label và Selector](18-labels-vi.md) — bài quan trọng nhất nhóm này; selector là cơ chế mọi controller và Service dùng để tìm Pod.
- [X] [Annotations](20-annotations-vi.md) — phân biệt rõ với label: annotation không dùng để chọn object.
- [X] [Namespaces](19-namespaces-vi.md) — trọng tâm: tài nguyên nào có namespace, tài nguyên nào cấp cluster.
- [X] [Các label được khuyến nghị](31-common-labels-vi.md) — quy ước `app.kubernetes.io/*`.
- [X] [Công cụ dòng lệnh kubectl](26-kubectl-vi.md) — cú pháp, các động từ chính.
- [ ] [Tổ chức quyền truy cập cluster bằng file kubeconfig](111-kubeconfig-vi.md) — context, cluster, user; cần cho mọi thao tác về sau.
- [ ] [Quản lý object trong Kubernetes](27-object-management-vi.md) — trọng tâm: khác biệt giữa imperative, declarative (`apply`) và khi nào dùng cái nào.
- [ ] [Field selector](28-field-selectors-vi.md) — bổ sung cho label selector khi lọc theo trường.

**Thực hành:** [Cài đặt và thiết lập kubectl trên Linux](186-install-kubectl-linux-vi.md) · [Cài đặt và thiết lập kubectl trên macOS](187-install-kubectl-macos-vi.md) · [Cài đặt và thiết lập kubectl trên Windows](188-install-kubectl-windows-vi.md) · [Quản lý các đối tượng Kubernetes](318-manage-kubernetes-objects-vi.md) · [Quản lý object Kubernetes theo kiểu imperative bằng file cấu hình](321-imperative-config-vi.md) · [Cấu hình truy cập nhiều cluster](361-configure-access-multiple-clusters-vi.md) · [Liệt kê tất cả Container image đang chạy trong Cluster](365-list-running-container-images-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 [Lab 1b — Object, label, kubectl và kubeconfig](labs/LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md) — thực hành name/UID, namespace, label/annotation, ba kỹ thuật quản lý object, kubeconfig và field selector. Lab này đóng phần `kubectl apply -f pod.yaml` và label selector trong checkpoint giai đoạn 1.

#### 1c. Vòng đời và cơ chế nền của object

- [ ] [Finalizers](29-finalizers-vi.md) — vì sao một object xóa mãi không đi.
- [ ] [Đối tượng sở hữu và đối tượng phụ thuộc](30-owners-dependents-vi.md) — owner reference.
- [ ] [Thu gom rác](36-garbage-collection-vi.md) — xóa cascade foreground/background; dựa trên hai bài trên.
- [ ] [Các Lease](35-leases-vi.md) — heartbeat của node và bầu leader của control plane.
- [ ] [Các phiên bản lưu trữ](32-storage-version-vi.md) — object được lưu trong etcd ở version nào.
- [ ] [Proxy phiên bản hỗn hợp](37-mixed-version-proxy-vi.md) — đọc lướt, chỉ cần biết tồn tại khi cluster có nhiều version apiserver.
- [ ] [Cloud Controller Manager](34-cloud-controller-vi.md) — nếu chạy on-premise thì đọc để biết phần nào **không** có.

**Thực hành:** [Phát triển Cloud Controller Manager](203-developing-cloud-controller-manager-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 [Lab 1c — Vòng đời và cơ chế nền của object](labs/LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md) — thực hành finalizer, owner/dependent, garbage collection và Lease; quan sát đúng giới hạn của storage version, Mixed Version Proxy và cloud controller trên cluster self-managed.

**Checkpoint:** giải thích được đường đi của `kubectl apply -f pod.yaml` từ lúc gõ lệnh đến khi container chạy, kể tên từng thành phần tham gia. Dùng `kubectl explain`, `kubectl get -o yaml`, label selector và `-n` thành thạo trên cluster lab đã chuẩn bị ở đầu lộ trình.

---

### Giai đoạn 2 — Container và runtime

**Mục tiêu:** hiểu tầng dưới Pod: image, runtime, CRI, cgroup — trước khi cấu hình runtime thật.

- [ ] [Các Container](39-containers-vi.md)
- [ ] [Các Image](40-images-vi.md) — trọng tâm: tag vs digest, `imagePullPolicy`, `imagePullSecrets`; đây là nguồn lỗi vận hành rất phổ biến.
- [ ] [Môi trường Container](41-container-environment-vi.md)
- [ ] [Các hook vòng đời của Container](42-container-lifecycle-hooks-vi.md) — `postStart`, `preStop`; `preStop` liên quan trực tiếp đến shutdown êm ở giai đoạn 3.
- [ ] [Container Runtime Interface (CRI)](44-cri-vi.md) — hợp đồng giữa kubelet và runtime.
- [ ] [Giới thiệu về cgroup v2](33-cgroups-vi.md) — nền tảng của mọi giới hạn tài nguyên học ở giai đoạn 3.
- [ ] [Runtime Class](43-runtime-class-vi.md) — chọn runtime khác nhau cho từng workload.
- [ ] [Các container runtime](00-container-runtimes-vi.md) — **đọc lý thuyết ở đây** (đặc biệt mục cgroup driver: kubelet và runtime phải khớp nhau). Phần cài đặt thực tế để dành làm cùng giai đoạn 8.

**Thực hành:** [Cấu hình một kubelet image credential provider](225-kubelet-credential-provider-vi.md) · [Chuyển từ polling sang cập nhật trạng thái container dựa trên sự kiện CRI](257-switch-to-evented-pleg-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 [Lab 2 — Container, image, CRI và cgroup](labs/LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md) — xác định runtime và CRI trên node, đối chiếu cgroup driver, quy tắc mặc định của `imagePullPolicy`, `ImagePullBackOff`, hook `PostStart`/`PreStop` và giới hạn của RuntimeClass.

**Checkpoint:** trên một máy Linux, giải thích được `containerd` và `runc` khác nhau chỗ nào, kiểm tra được cgroup version của máy, và nói được hậu quả khi kubelet dùng `systemd` còn runtime dùng `cgroupfs`.

---

### Giai đoạn 3 — Pod và cấu hình

**Mục tiêu:** Pod là đơn vị nhỏ nhất — phải nắm vòng đời, probe, và cách cấp phát tài nguyên trước khi đụng tới controller.

#### 3a. Pod và vòng đời

- [ ] [Workload](45-workloads-vi.md)
- [ ] [Pod](46-pods-vi.md)
- [ ] [Vòng đời của Pod](47-pod-lifecycle-vi.md) — bài xương sống: phase, trạng thái container, `restartPolicy`, chấm dứt êm và `terminationGracePeriodSeconds`.
- [ ] [Các Condition của Pod](48-pod-condition-vi.md)
- [ ] [Các probe Liveness, Readiness và Startup](49-probes-vi.md) — trọng tâm: phân biệt ba loại; cấu hình sai liveness là nguyên nhân kinh điển của restart loop.
- [ ] [Container khởi tạo](50-init-containers-vi.md)
- [ ] [Các container sidecar](51-sidecar-containers-vi.md) — sidecar hiện là init container có `restartPolicy: Always`.
- [ ] [Các container tạm thời](52-ephemeral-containers-vi.md) — công cụ debug, dùng nhiều ở giai đoạn xử lý sự cố.
- [ ] [Không gian tên người dùng](55-user-namespaces-vi.md)
- [ ] [Downward API](56-downward-api-vi.md)
- [ ] [Cấu hình Pod nâng cao](60-advanced-pod-config-vi.md)

**Thực hành:** [Chạy các thành phần Node của Kubernetes dưới người dùng không phải root](226-kubelet-in-userns-vi.md) · [Cấu hình Pod và Container](262-configure-pod-container-vi.md) · [Gắn handler vào các sự kiện vòng đời của Container](272-attach-handler-lifecycle-event-vi.md) · [Cấu hình các probe Liveness, Readiness và Startup](274-configure-probes-vi.md) · [Cấu hình khởi tạo Pod](276-configure-pod-initialization-vi.md) · [Sử dụng Image Volume với một Pod](285-image-volumes-vi.md) · [Chia sẻ Process Namespace giữa các Container trong một Pod](292-share-process-namespace-vi.md) · [Sử dụng user namespace với Pod](295-user-namespaces-tasks-vi.md) · [Expose thông tin Pod cho container thông qua file](335-downward-api-volume-vi.md) · [Expose thông tin Pod cho container thông qua biến môi trường](336-env-variable-expose-pod-info-vi.md) · [Giao tiếp giữa các Container trong cùng Pod bằng Volume dùng chung](360-containers-shared-volume-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 [Lab 3a — Pod và vòng đời](labs/LAB-3A-POD-VA-VONG-DOI.md) — vòng đời Pod: phase và condition, `restartPolicy`, ba loại probe, init container, sidecar, kết thúc êm, ephemeral container và Downward API.

#### 3b. Cấu hình ứng dụng: ConfigMap, Secret và dữ liệu cho Pod

- [ ] [Cấu hình](107-configuration-vi.md)
- [ ] [ConfigMap](108-configmap-vi.md)
- [ ] [Secret](109-secret-vi.md) — trọng tâm: Secret **chỉ mã hóa base64**, không phải mã hóa thật. ⏳ **Nợ #6** — encryption at rest chưa làm được ở đây vì phải sửa cấu hình apiserver; trả ở [giai đoạn 22](#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu).

**Thực hành:** [Cấu hình một Pod để sử dụng ConfigMap](275-configure-pod-configmap-vi.md) · [Pull image từ một private registry](287-pull-image-private-registry-vi.md) · [Quản lý Secret](325-configmap-secret-vi.md) · [Quản lý Secret bằng file cấu hình](326-secret-config-file-vi.md) · [Quản lý Secret bằng kubectl](327-secret-kubectl-vi.md) · [Quản lý Secret bằng Kustomize](328-secret-kustomize-vi.md) · [Đưa dữ liệu vào ứng dụng](329-inject-data-application-vi.md) · [Định nghĩa command và argument cho container](330-define-command-argument-vi.md) · [Định nghĩa biến môi trường cho một Container](331-define-environment-variable-vi.md) · [Định nghĩa giá trị biến môi trường bằng một Init Container](332-define-env-via-file-vi.md) · [Định nghĩa các biến môi trường phụ thuộc](333-interdependent-env-variables-vi.md) · [Phân phối thông tin xác thực một cách an toàn bằng Secret](334-distribute-credentials-secure-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 [Lab 3b — Cấu hình ứng dụng](labs/LAB-3B-CAU-HINH-UNG-DUNG.md) — thực hành ConfigMap, Secret, biến môi trường và Downward API trên Pod trần; chứng minh Secret chỉ là base64. Phát sinh ⏳ **nợ #6** — mã hóa Secret at rest, trả ở [giai đoạn 22](#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu).

#### 3c. Tài nguyên, QoS và gián đoạn

- [ ] [Quản lý tài nguyên cho Pod và Container](110-manage-resources-containers-vi.md) — **bài bắt buộc phải chắc**: `requests` quyết định lập lịch, `limits` quyết định giới hạn thực thi. Toàn bộ QoS, eviction và scheduling phía sau đều dựa vào bài này.
- [ ] [Các lớp chất lượng dịch vụ của Pod](54-pod-qos-vi.md) — Guaranteed/Burstable/BestEffort suy ra trực tiếp từ requests và limits.
- [ ] [Sự gián đoạn](53-disruptions-vi.md) — gián đoạn tự nguyện vs không tự nguyện, PodDisruptionBudget.
- [ ] [Pod tĩnh](58-static-pods-vi.md) — kubelet tự quản; chính là cách control plane của kubeadm chạy, cần cho giai đoạn 8.

**Thực hành:** [Gán tài nguyên CPU cho Container và Pod](263-assign-cpu-resource-vi.md) · [Gán tài nguyên memory cho Container và Pod](264-assign-memory-resource-vi.md) · [Gán tài nguyên CPU và memory ở cấp Pod](265-assign-pod-level-resources-vi.md) · [Gán Extended Resource cho một Container](284-extended-resource-vi.md) · [Cấu hình Quality of Service cho Pod](288-quality-service-pod-vi.md) · [Thay đổi kích thước tài nguyên CPU và Memory được gán cho Container](289-resize-container-resources-vi.md) · [Thay đổi kích thước tài nguyên CPU và Memory được gán cho Pod](290-resize-pod-resources-vi.md) · [Chỉ định Disruption Budget cho ứng dụng của bạn](339-configure-pdb-vi.md) · [Tạo static Pod](293-static-pod-tasks-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 [Lab 3c — Tài nguyên, QoS và gián đoạn](labs/LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) — requests quyết định lập lịch còn limits quyết định thực thi — đọc thẳng `cpu.max`/`memory.max` trong cgroup v2; suy QoS từ manifest; Pod `Pending` vì vượt `Allocatable` thật; PodDisruptionBudget chặn Eviction API; static Pod và mirror Pod.

**Checkpoint:** viết tay một Pod manifest có init container, sidecar, readiness + liveness probe, requests/limits, mount ConfigMap và Secret. Cố ý đặt request vượt sức node để thấy Pod `Pending`, rồi đọc `kubectl describe` tìm lý do. Xác định QoS class của 3 Pod khác nhau chỉ bằng cách nhìn manifest.

---

### Giai đoạn 4 — Workload controller

**Mục tiêu:** hiểu các controller vận hành Pod thay bạn, và cơ chế rollout/rollback.

#### 4a. ReplicaSet, Deployment và rollout

- [ ] [Quản lý Workload — trang mục các controller](62-controllers-index-vi.md)
- [ ] [ReplicaSet](64-replicaset-vi.md) — **đọc trước Deployment**, vì Deployment vận hành thông qua ReplicaSet.
- [ ] [Deployment](63-deployment-vi.md) — bài dài nhất bộ tài liệu. Trọng tâm: rollout, rollback, chiến lược RollingUpdate/Recreate, `maxSurge`/`maxUnavailable`, revision history.
- [ ] [Quản lý Workload — vận hành bằng kubectl](61-management-vi.md) — tổ chức manifest, `kubectl apply` theo nhóm, canary thủ công.
- [ ] **Đọc như tài liệu lịch sử:** [ReplicationController](70-replicationcontroller-vi.md) — tiền thân của ReplicaSet, không dùng cho hệ thống mới. Chỉ cần biết nó tồn tại khi gặp cluster cũ.

**Thực hành:** [Sử dụng xóa theo tầng trong Cluster](260-use-cascading-deletion-vi.md) · [Quản lý object Kubernetes theo kiểu khai báo bằng file cấu hình](319-declarative-config-vi.md) · [Cập nhật đối tượng API tại chỗ bằng kubectl patch](324-kubectl-patch-vi.md) · [Chạy ứng dụng](337-run-application-vi.md) · [Chạy một ứng dụng Stateless bằng Deployment](345-run-stateless-application-vi.md) · [Scale thủ công theo chiều ngang cho một Deployment](346-scale-deployment-vi.md) · [Cập nhật một Deployment mà không gây gián đoạn](348-update-deployment-rolling-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 [Lab 4a — ReplicaSet, Deployment và rollout](labs/LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md) — chuỗi Pod → ReplicaSet → Deployment qua `ownerReferences`, rollout/rollback, `maxSurge`/`maxUnavailable`, Recreate so với RollingUpdate, và rollout thất bại.

#### 4b. StatefulSet, DaemonSet, Job và autoscaling

- [ ] [StatefulSets](65-statefulset-vi.md) — trọng tâm: định danh ổn định, thứ tự khởi tạo, `volumeClaimTemplates`. ⏳ **Nợ #2 và #3** — `volumeClaimTemplates` cần StorageClass (giai đoạn 6) nên trả ở [Lab 6a](#giai-đoạn-6--lưu-trữ); Service headless quản trị cần bài Service (giai đoạn 5) nên trả ở [Lab 5a](#giai-đoạn-5--mạng-nền-tảng). Ở đây chỉ đọc.
- [ ] [DaemonSet](66-daemonset-vi.md) — mô hình mọi node một Pod; CNI và log agent đều chạy kiểu này.
- [ ] [Jobs](67-job-vi.md) — trọng tâm: `completions`, `parallelism`, `backoffLimit`; các mục Pod failure policy và Elastic Indexed Job đọc lướt lần đầu.
- [ ] [Tự động dọn dẹp các Job đã hoàn thành](68-ttlafterfinished-vi.md)
- [ ] [CronJob](69-cron-jobs-vi.md) — trọng tâm: `concurrencyPolicy`, `startingDeadlineSeconds`.
- [ ] [Tự động co giãn Workload](71-autoscaling-vi.md)
- [ ] [Tự động co giãn Pod theo chiều ngang](72-horizontal-pod-autoscale-vi.md) — **chỉ đọc lý thuyết ở đây.** ⏳ **Nợ #1** — HPA cần metrics-server thuộc giai đoạn 11; thực hành ở [Lab 11b](#giai-đoạn-11--observability). Không cài metrics-server sớm để "chạy thử".
- [ ] [Tự động co giãn Pod theo chiều dọc](73-vertical-pod-autoscale-vi.md) — như trên. ⏳ **Nợ #1**, thực hành ở [Lab 11b](#giai-đoạn-11--observability).
- [ ] [Khả năng tự phục hồi của Kubernetes](38-self-healing-vi.md) — đọc ở đây (không phải giai đoạn 1) vì nội dung dựa trên Deployment, ReplicaSet, StatefulSet vừa học.

**Thực hành:** [Xóa cưỡng bức Pod của StatefulSet](341-force-delete-stateful-set-pod-vi.md) · [Scale một StatefulSet](347-scale-stateful-set-vi.md) · [Chạy Job](349-job-tasks-vi.md) · [Chạy các tác vụ tự động với CronJob](350-automated-tasks-cron-jobs-vi.md) · [Xử lý song song thô sử dụng hàng đợi công việc](351-coarse-parallel-work-queue-vi.md) · [Indexed Job để xử lý song song với phân công việc tĩnh](353-indexed-parallel-processing-vi.md) · [Xử lý song song bằng cách khai triển template](355-parallel-processing-expansion-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 [Lab 4b — StatefulSet, DaemonSet và Job](labs/LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) — định danh ổn định và thứ tự khởi tạo của StatefulSet, DaemonSet với taint/toleration, Job và `backoffLimit`, TTL, CronJob. ⏳ Còn treo **nợ #1, #2, #3** — xem [Sổ nợ lộ trình](#sổ-nợ-lộ-trình).

**Checkpoint:** tạo Deployment 3 replica, thực hiện rolling update, theo dõi `kubectl rollout status`, rồi rollback về revision trước. Xóa thủ công một Pod và quan sát ReplicaSet tạo lại. Giải thích được vì sao StatefulSet không thể thay bằng Deployment cho database.

---

### Giai đoạn 5 — Mạng nền tảng

**Mục tiêu:** hiểu Pod nói chuyện với nhau và với bên ngoài thế nào. Service học trước DNS và Ingress.

- [ ] [Service, cân bằng tải và mạng](81-services-networking-vi.md) — trọng tâm: mô hình mạng Kubernetes (mọi Pod thấy nhau không qua NAT).
- [ ] [Service](82-service-vi.md) — bài quan trọng nhất giai đoạn. Trọng tâm: ClusterIP, NodePort, LoadBalancer, ExternalName, headless Service, và cơ chế virtual IP.
- [ ] [EndpointSlices](83-endpoint-slices-vi.md) — Service tìm Pod qua đây.
- [ ] [DNS cho Service và Pod](10-dns-pod-service-vi.md) — trọng tâm: dạng FQDN `svc.namespace.svc.cluster.local`, `search` domain và `ndots:5` trong `/etc/resolv.conf`.
- [ ] [Hostname của Pod](57-pod-hostname-vi.md) — `hostname`/`subdomain`, dùng với headless Service.
- [ ] [Chính sách mạng](84-network-policies-vi.md) — trọng tâm: mặc định là cho phép tất cả; policy chỉ có tác dụng khi CNI hỗ trợ. ⏳ **Nợ #4** — Flannel của baseline **không** thực thi NetworkPolicy, nên policy viết ở đây chưa chặn được gì thật; trả ở [Lab 5b](#giai-đoạn-5--mạng-nền-tảng).
- [ ] [Định tuyến nhận biết topology](86-topology-aware-routing-vi.md)
- [ ] [Chính sách lưu lượng nội bộ của Service](87-service-traffic-policy-vi.md)
- [ ] [Cấp phát ClusterIP cho Service](88-cluster-ip-allocation-vi.md)

**Thực hành:** [Chuyển đổi file Docker Compose thành tài nguyên Kubernetes](294-translate-compose-kubernetes-vi.md) · [Quản lý object Kubernetes bằng lệnh imperative](320-imperative-command-vi.md) · [Quản lý object Kubernetes theo kiểu khai báo bằng Kustomize](322-kustomization-vi.md) · [Xóa một StatefulSet](340-delete-stateful-set-vi.md) · [Xử lý song song mịn sử dụng hàng đợi công việc](352-fine-parallel-work-queue-vi.md) · [Job với giao tiếp Pod-đến-Pod](354-job-pod-to-pod-communication-vi.md) · [Cấu hình DNS cho một cluster](362-configure-dns-cluster-vi.md) · [Kết nối Frontend với Backend bằng Service](363-connecting-frontend-backend-vi.md) · [Tạo bộ cân bằng tải bên ngoài](364-create-external-load-balancer-vi.md) · [Sử dụng Port Forwarding để truy cập ứng dụng trong Cluster](366-port-forward-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 [Lab 5a — Service, EndpointSlice và DNS](labs/LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) — ClusterIP/NodePort/ExternalName, EndpointSlice theo selector, DNS `ndots:5` và FQDN, `internalTrafficPolicy` theo node. ✅ **Trả nợ #3** — headless Service cho StatefulSet.
- [ ] [Ingress](11-ingress-vi.md) — trọng tâm: rule, path type, IngressClass, TLS.
- [ ] [Ingress Controllers](12-ingress-controllers-vi.md) — không có controller thì Ingress vô nghĩa.
- [ ] [Gateway API](13-gateway-vi.md) — hướng thay thế Ingress; đọc để biết định hướng tương lai.
- [ ] [Dual-stack IPv4/IPv6](85-dual-stack-vi.md)

### Tầng hạ tầng mạng của cluster

- [ ] [Mạng trong cluster](157-networking-vi.md) — mô hình mạng ở góc nhìn quản trị và các cách hiện thực.
- [ ] [Network Plugin](183-network-plugins-vi.md) — CNI; cần trước khi cài CNI thật ở giai đoạn 8.
- [ ] [Các loại proxy trong Kubernetes](164-proxies-vi.md) — kube-proxy và các loại proxy khác.
- [ ] 🧪 [Lab 5b — NetworkPolicy, Ingress và CNI](labs/LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) — chứng minh Flannel **không** thực thi NetworkPolicy, đổi CNI sang Calico, chứng minh policy chặn thật, rồi cài ingress controller. ✅ **Trả nợ #4**. Tạo snapshot `02-net-ready`.

**Checkpoint:** expose một Deployment bằng ClusterIP rồi NodePort; từ trong một Pod dùng `nslookup`/`curl` gọi Service bằng tên ngắn và FQDN. Viết một NetworkPolicy chặn toàn bộ ingress vào một namespace rồi mở đúng một cổng. Giải thích được khác biệt giữa Service, Ingress và load balancer bên ngoài.

---

### Giai đoạn 6 — Lưu trữ

**Mục tiêu:** cấp phát và quản lý dữ liệu bền vững. ConfigMap/Secret đã học ở giai đoạn 3 nên phần projected volume không còn phụ thuộc ngược.

- [ ] [Lưu trữ](90-storage-vi.md)
- [ ] [Các Volume](91-volumes-vi.md) — trọng tâm: `emptyDir`, `hostPath` (và rủi ro bảo mật của nó), `configMap`, `secret`, `persistentVolumeClaim`. Các driver in-tree đã bị loại bỏ chỉ cần đọc lướt.
- [ ] [Volume bền vững](92-persistent-volumes-vi.md) — bài xương sống: vòng đời PV/PVC, binding, access mode, reclaim policy, mở rộng dung lượng.
- [ ] [Lớp lưu trữ](96-storage-classes-vi.md) — trọng tâm: `provisioner`, `reclaimPolicy`, `volumeBindingMode`, default StorageClass.
- [ ] [Cấp phát Volume động](98-dynamic-provisioning-vi.md)
- [ ] [Volume dạng projected](93-projected-volumes-vi.md) — gộp ConfigMap, Secret, downwardAPI, service account token vào một mount.
- [ ] [Volume tạm thời](94-ephemeral-volumes-vi.md)
- [ ] [Lưu trữ tạm thời cục bộ](95-ephemeral-storage-vi.md) — liên quan trực tiếp tới eviction ở giai đoạn 7.

**Thực hành:** [Cấu hình Pod sử dụng projected Volume cho lưu trữ](277-configure-projected-volume-vi.md) · [Cấu hình Pod sử dụng Volume để lưu trữ](280-configure-volume-storage-vi.md) · [Chạy ứng dụng có trạng thái được nhân bản](343-run-replicated-stateful-application-vi.md) · [Chạy ứng dụng có trạng thái đơn thực thể](344-run-single-instance-stateful-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 [Lab 6a — PV, PVC và StorageClass](labs/LAB-6A-PV-PVC-VA-STORAGECLASS.md) — cài local-path-provisioner, vòng đời PV/PVC, `reclaimPolicy` Delete so với Retain, `volumeBindingMode`, và dữ liệu bền qua vòng đời Pod. ✅ **Trả nợ #2** — `volumeClaimTemplates`. Tạo snapshot `03-storage-ready`.
- [ ] [Lớp thuộc tính Volume](97-volume-attributes-classes-vi.md)
- [ ] [Ảnh chụp nhanh Volume](99-volume-snapshots-vi.md) — ⏳ **Nợ #5** bắt đầu từ đây và kéo tới bài [101](101-volume-pvc-datasource-vi.md): chỉ thực hành được nếu CSI driver đang dùng hỗ trợ snapshot; trả ở [Lab 6b](#giai-đoạn-6--lưu-trữ).
- [ ] [Các lớp Volume Snapshot](100-volume-snapshot-classes-vi.md)
- [ ] [Nhân bản CSI Volume](101-volume-pvc-datasource-vi.md)
- [ ] [Volume Populator và Nguồn dữ liệu](102-volume-populators-vi.md)
- [ ] [Dung lượng lưu trữ](103-storage-capacity-vi.md)
- [ ] [Giới hạn volume theo từng Node](104-storage-limits-vi.md)
- [ ] [Giám sát tình trạng volume](105-volume-health-monitoring-vi.md)
- [ ] 🧪 [Lab 6b — Snapshot và volume nâng cao](labs/LAB-6B-SNAPSHOT-VA-VOLUME-NANG-CAO.md) — kiểm tra năng lực CSI driver rồi rẽ nhánh: chụp/khôi phục/nhân bản snapshot nếu driver hỗ trợ, hoặc ghi hồ sơ nợ nếu không. ⏳ **Nợ #5 chưa trả** với provisioner của baseline.

**Checkpoint:** tạo StorageClass, xin một PVC và mount vào Pod; xóa Pod rồi tạo lại, chứng minh dữ liệu còn nguyên. Thử `reclaimPolicy: Retain` và `Delete` để thấy khác biệt khi xóa PVC. Chạy một StatefulSet có `volumeClaimTemplates` và quan sát PVC sinh ra theo từng replica.

---

### Giai đoạn 7 — Lập lịch và chính sách tài nguyên

**Mục tiêu:** điều khiển Pod chạy ở đâu, và bảo vệ cluster khỏi workload ngốn tài nguyên.

#### 7a. Scheduling và eviction

- [ ] [Lập lịch, Preemption và Eviction](136-scheduling-eviction-vi.md)
- [ ] [Bộ lập lịch của Kubernetes](137-kube-scheduler-vi.md) — chu trình filter rồi score.
- [ ] [Gán Pod cho Node](138-assign-pod-node-vi.md) — bài dài, trọng tâm: `nodeSelector`, node affinity (required vs preferred), inter-pod affinity/anti-affinity.
- [ ] [Taint và Toleration](139-taint-and-toleration-vi.md) — mặt đối ngẫu của affinity; control plane node bị taint chính là cơ chế này.
- [ ] [Ràng buộc phân bố Pod theo topology](140-topology-spread-constraints-vi.md) — trải Pod đều theo zone/node.
- [ ] [Độ ưu tiên và Preemption của Pod](141-pod-priority-preemption-vi.md) — PriorityClass.
- [ ] [Eviction do áp lực node](142-node-pressure-eviction-vi.md) — trọng tâm: ngưỡng eviction, thứ tự trục xuất theo QoS.
- [ ] [Eviction khởi phát qua API](143-api-eviction-vi.md) — cơ chế đứng sau `kubectl drain`.
- [ ] [Pod Overhead](144-pod-overhead-vi.md)
- [ ] [Mức sẵn sàng lập lịch của Pod](145-pod-scheduling-readiness-vi.md)
- [ ] [Tinh chỉnh hiệu năng bộ lập lịch](146-scheduler-perf-tuning-vi.md) — đọc lướt, chỉ cần khi cluster rất lớn.
- [ ] [Scheduling Framework](147-scheduling-framework-vi.md) — các điểm mở rộng của scheduler.
- [ ] [Đóng gói tài nguyên](148-resource-bin-packing-vi.md)

**Thực hành:** [Bảo đảm lập lịch cho các Pod add-on quan trọng](210-guaranteed-scheduling-critical-addon-pods-vi.md) · [Gán Pod vào Node bằng Node Affinity](266-assign-pods-nodes-node-affinity-vi.md) · [Gán Pod vào Node](267-assign-pods-nodes-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 [Lab 7a — Lập lịch và eviction](labs/LAB-7A-LAP-LICH-VA-EVICTION.md) — chu trình filter/score của scheduler, `nodeSelector` và node affinity, taint/toleration, topology spread, PriorityClass và preemption, `schedulingGates`, và đọc ngưỡng eviction thật của kubelet.

#### 7b. Chính sách giới hạn tài nguyên

- [ ] [Chính sách](132-policies-vi.md)
- [ ] [Khoảng giới hạn tài nguyên](133-limit-range-vi.md) — đặt mặc định và trần cho từng Pod/container trong namespace.
- [ ] [Hạn ngạch tài nguyên](134-resource-quotas-vi.md) — trần tổng cho cả namespace; công cụ chính khi chia cluster cho nhiều nhóm.
- [ ] [Giới hạn và dự trữ Process ID](135-pid-limiting-vi.md)
- [ ] [Các trình quản lý tài nguyên](74-resource-managers-vi.md) — CPU manager, memory manager, topology manager của kubelet.
- [ ] [Tính năng do Node khai báo](154-node-declared-features-vi.md) — đọc lướt.

- [ ] 🧪 [Lab 7b — Quota và giới hạn tài nguyên](labs/LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) — LimitRange đặt mặc định và trần cho từng Pod, ResourceQuota đặt trần cho cả namespace, và mối liên hệ bắt buộc giữa hai thứ; đọc `podPidsLimit` và policy của resource manager từ cấu hình kubelet thật.

**Checkpoint:** taint một node và chứng minh Pod thường không lên đó, rồi thêm toleration để lên được. Đặt ResourceQuota + LimitRange cho một namespace, thử tạo Pod vượt quota và đọc thông báo từ chối. Tạo tình huống node hết đĩa để quan sát eviction và thứ tự Pod bị trục xuất.

---

### Giai đoạn 8 — Dựng cluster bằng kubeadm

**Mục tiêu:** tự tay dựng cluster. Đến đây bạn đã hiểu component, Pod, Service, CNI và storage nên mỗi bước cài đặt đều có nghĩa, không phải gõ theo hướng dẫn.

- [ ] [Cài đặt kubeadm](01-install-kubeadm-vi.md) — kèm phần thực hành cài container runtime ở [bài 00](00-container-runtimes-vi.md) đã đọc lý thuyết tại giai đoạn 2. Trọng tâm: port cần mở, swap, cgroup driver.
- [ ] [Tạo một cluster với kubeadm](02-create-cluster-kubeadm-vi.md) — `kubeadm init`, cài CNI, join worker, taint control plane.
- [ ] [Tùy chỉnh các thành phần với kubeadm API](03-control-plane-flags-vi.md) — `ClusterConfiguration` và patch.
- [ ] [Cấu hình từng kubelet trong cluster bằng kubeadm](04-kubelet-integration-vi.md)
- [ ] [Các lựa chọn topology cho tính sẵn sàng cao](06-ha-topology-vi.md) — stacked etcd vs external etcd; **quyết định trước khi dựng HA**.
- [ ] [Thiết lập cluster etcd có tính sẵn sàng cao với kubeadm](07-setup-ha-etcd-with-kubeadm-vi.md) — chỉ cần nếu chọn external etcd; phải dựng xong trước bước sau.
- [ ] [Tạo cluster có tính sẵn sàng cao với kubeadm](08-high-availability-vi.md) — cần load balancer đứng trước các API server.
- [ ] [Hỗ trợ dual-stack với kubeadm](05-dual-stack-support-vi.md)
- [ ] [Xử lý sự cố kubeadm](09-troubleshooting-kubeadm-vi.md) — **tài liệu tra cứu, không đọc tuần tự**. Đọc lướt mục lục một lần để biết có gì, rồi quay lại khi gặp lỗi.

**Thực hành:** [Quản trị với kubeadm](214-kubeadm-tasks-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 **Lab 8a — Dựng cluster bằng kubeadm** — chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab). Phá và dựng lại chính ba VM của chuỗi snapshot, kết thúc bằng restore về `03-storage-ready`. ⏳ **Nợ #8** — dựng được cluster nhưng chưa có quy trình backup/restore etcd bằng `etcdctl`; trả ở [giai đoạn 19](#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa). Cho tới lúc đó, việc khôi phục chỉ dựa vào snapshot VM.
- [ ] 🧪 **Lab 8b — HA với stacked etcd** — chưa viết. Cần **bộ VM riêng** (3 control plane + 2 worker + 1 load balancer), snapshot tiền tố `8x-`.
- [ ] 🧪 **Lab 8c — HA với external etcd** — chưa viết. Dựng trên bộ VM của lab 8b, bổ sung nhóm node etcd tách biệt.

**Checkpoint — bắt buộc làm thật, không xem video:**

- Dựng cluster 1 control plane + 2 worker, cài CNI, chạy được một Deployment có Service.
- Dựng lại thành HA 3 control plane với load balancer phía trước.
- Dựng một lần với stacked etcd, một lần với external etcd.
- Join thêm node, remove node đúng quy trình (drain trước).
- **Phá rồi khôi phục**: tắt một control plane node và chứng minh cluster vẫn phục vụ; xóa `/etc/kubernetes/manifests/kube-apiserver.yaml` rồi khôi phục.
- Đọc được `crictl ps`, `journalctl -u kubelet` khi node không lên `Ready`.

---

### Giai đoạn 9 — Bảo mật và multi-tenancy

**Mục tiêu:** kiểm soát ai làm được gì, và cô lập workload.

- [ ] [Bảo mật](113-security-vi.md)
- [ ] [Bảo mật Cloud Native và Kubernetes](114-cloud-native-security-vi.md) — bốn giai đoạn vòng đời theo sách trắng CNCF: phát triển, phân phối, triển khai, runtime. Bản upstream cũ trình bày cùng ý dưới dạng "mô hình 4C".
- [ ] [Tài khoản dịch vụ](118-service-accounts-vi.md) — danh tính của Pod khi gọi API.
- [ ] [Kiểm soát truy cập vào Kubernetes API](119-controlling-access-vi.md) — **bài xương sống**: authentication → authorization (RBAC) → admission control, đúng thứ tự ba chặng.
- [ ] [Các thực hành tốt về kiểm soát truy cập dựa trên vai trò](120-rbac-good-practices-vi.md) — Role/ClusterRole, binding, nguyên tắc quyền tối thiểu.

**Thực hành:** [Quản trị Cloud Controller Manager](254-running-cloud-controller-vi.md) · [Sử dụng sysctl trong một cluster Kubernetes](258-sysctl-cluster-vi.md) · [Cấu hình Service Account cho Pod](279-configure-service-account-vi.md) · [Thực thi Pod Security Standards bằng cách cấu hình Admission Controller tích hợp sẵn](282-enforce-standards-admission-controller-vi.md) · [Thực thi Pod Security Standards bằng nhãn Namespace](283-enforce-standards-namespace-labels-vi.md) · [Di chuyển từ PodSecurityPolicy sang PodSecurity Admission Controller tích hợp sẵn](286-migrate-from-psp-vi.md) · [Cấu hình Security Context cho Pod hoặc Container](291-security-context-vi.md) · [Truy cập Kubernetes API từ một Pod](338-access-api-from-pod-vi.md) · [Truy cập cluster](359-access-cluster-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 **Lab 9a — ServiceAccount, authn/authz và RBAC** — chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab).
- [ ] [Chuẩn bảo mật Pod](115-pod-security-standards-vi.md) — ba profile Privileged/Baseline/Restricted.
- [ ] [Cơ chế admission bảo mật Pod](116-pod-security-admission-vi.md) — áp ba profile trên vào namespace bằng label, chế độ enforce/audit/warn.
- [ ] [Các thực hành tốt cho Kubernetes Secrets](121-secrets-good-practices-vi.md)
- [ ] [Đa người thuê](122-multi-tenancy-vi.md) — ghép namespace + RBAC + quota + NetworkPolicy thành mô hình cô lập.
- [ ] [Hướng dẫn tăng cường bảo mật — Các cơ chế xác thực](123-hardening-authentication-vi.md)
- [ ] [Bảo mật cho node Linux](126-linux-security-vi.md)
- [ ] [Các ràng buộc bảo mật của Linux kernel cho Pod và container](127-linux-kernel-security-vi.md) — capabilities, seccomp, AppArmor, SELinux.
- [ ] [Rủi ro vượt qua Kubernetes API Server](128-api-server-bypass-risks-vi.md) — vì sao static pod và quyền truy cập kubelet nguy hiểm.
- [ ] [Danh sách kiểm tra bảo mật](129-security-checklist-vi.md) — dùng như checklist đối chiếu cluster của bạn, không phải bài đọc.
- [ ] [Danh sách kiểm tra bảo mật ứng dụng](130-application-security-checklist-vi.md)
- [ ] [Ưu tiên và Công bằng cho API](166-flow-control-vi.md) — bảo vệ API server khỏi quá tải; FlowSchema và PriorityLevelConfiguration.
- [ ] [Thực hành tốt cho Admission Webhook](173-admission-webhooks-vi.md) — webhook hỏng có thể làm chết cả cluster; đọc kỹ phần failure policy.
- [ ] 🧪 **Lab 9b — Pod Security và hardening** — chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab).
- [ ] **Đọc như tài liệu lịch sử:** [Chính sách bảo mật Pod](117-pod-security-policy-vi.md) — PodSecurityPolicy đã bị gỡ khỏi Kubernetes, thay bằng bài 6–7 ở trên. Chỉ đọc khi tiếp quản cluster rất cũ.

**Để sau:** [Hướng dẫn tăng cường bảo mật — Cấu hình Scheduler](124-hardening-scheduler-vi.md) và [— Cấp phát tài nguyên động](125-hardening-dra-vi.md) nằm ở giai đoạn 13, sau khi đã học DRA và scheduler nâng cao.

**Checkpoint:** tạo một ServiceAccount + Role + RoleBinding cho phép một ứng dụng chỉ đọc được ConfigMap trong đúng một namespace, rồi tự kiểm chứng bằng `kubectl auth can-i --as=...`. Bật Pod Security Admission mức `restricted` cho một namespace và xem Pod đặc quyền bị từ chối. Rà cluster của mình theo checklist bài 14.

---

### Giai đoạn 10 — Vận hành day-2

Đây là phần **không có tài liệu trong thư mục này** vì toàn bộ nằm ở nhánh `/docs/tasks/` của kubernetes.io. Theo thiết kế lộ trình, bạn học hết lý thuyết (giai đoạn 11–15) rồi chuyển sang phần thực hành ở [Checkpoint tiếp nối](#phần-ii--vận-hành-cluster) cuối file.

Nếu đang cần vận hành gấp một cluster production, có thể nhảy tới phần đó ngay sau giai đoạn 9 — đặc biệt là ba nhóm: nâng cấp cluster, vòng đời chứng chỉ, và backup/restore etcd.

---

### Giai đoạn 11 — Observability

**Mục tiêu:** biết cluster đang khỏe hay ốm, và tìm được nguyên nhân.

- [ ] [Khả năng quan sát](162-observability-vi.md) — đọc trước để có bức tranh chung ba trụ cột.
- [ ] [Metric cho các thành phần hệ thống Kubernetes](160-system-metrics-vi.md) — endpoint `/metrics`, định dạng Prometheus.
- [ ] [Metrics cho trạng thái đối tượng Kubernetes](163-kube-state-metrics-vi.md) — khác biệt với metric tài nguyên.
- [ ] [Kiến trúc ghi log](158-logging-vi.md) — log ở mức container, node, cluster; mô hình sidecar và node agent.
- [ ] [Log hệ thống](159-system-logs-vi.md) — log của kubelet và các thành phần control plane, mức verbosity.
- [ ] [Trace cho các thành phần hệ thống Kubernetes](161-system-traces-vi.md)

**Thực hành:** [Giám sát, ghi log và gỡ lỗi](296-debug-vi.md) · [Xử lý sự cố ứng dụng](297-debug-application-vi.md) · [Truy cập shell của một container đang chạy](304-get-shell-running-container-vi.md) · [Phát triển và debug service cục bộ bằng telepresence](309-local-debugging-vi.md) · [Ghi log trong Kubernetes](316-debug-logging-vi.md) · [Giám sát trong Kubernetes](317-debug-monitoring-vi.md) · [Hướng dẫn từng bước về HorizontalPodAutoscaler](342-hpa-walkthrough-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 **Lab 11a — Observability** — chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab). Cài metrics-server và stack giám sát, tạo snapshot `04-metrics-ready`.
- [ ] 🧪 **Lab 11b — HPA và VPA** — chưa viết. ✅ **Trả nợ #1 — thực hành HPA và VPA**, phát sinh ở [giai đoạn 4](#giai-đoạn-4--workload-controller), bài [72](72-horizontal-pod-autoscale-vi.md) và [73](73-vertical-pod-autoscale-vi.md): đã đọc lý thuyết từ giai đoạn 4 nhưng chỉ thực hành được sau khi có metrics-server ở Lab 11a. **Đọc lại hai bài đó trước khi làm.**

**Checkpoint:** triển khai metrics-server và chạy được `kubectl top node`/`kubectl top pod`. Triển khai một stack Prometheus + Grafana, thu metric từ kubelet và kube-state-metrics, dựng một dashboard và một alert (ví dụ node NotReady hoặc Pod CrashLoopBackOff). Gom log tập trung bằng một agent chạy dạng DaemonSet. Sau đó cho một Deployment tự co giãn bằng HPA dưới tải và quan sát số replica thay đổi — phần này đóng nốt checkpoint autoscaling của giai đoạn 4.

---

### Giai đoạn 12 — Quản trị cluster nâng cao

**Mục tiêu:** các chủ đề vận hành ở tầng cluster.

- [ ] [Quản trị cluster](155-cluster-administration-vi.md)
- [ ] [Tắt node](169-node-shutdown-vi.md) — graceful node shutdown; quan trọng khi bảo trì phần cứng.
- [ ] [Quản lý bộ nhớ swap](170-swap-memory-management-vi.md) — swap từng bị cấm hoàn toàn, nay có hỗ trợ; đọc kỹ nếu node của bạn bật swap.
- [ ] [Tự động mở rộng Node](171-node-autoscaling-vi.md)
- [ ] [Cài đặt các Add-on](165-addons-vi.md) — danh mục add-on theo nhóm chức năng.
- [ ] [Phiên bản tương thích cho các thành phần Control Plane](168-compatibility-version-vi.md) — `--emulated-version`, hữu ích khi nâng cấp thận trọng.
- [ ] [Bầu chọn leader có phối hợp](167-coordinated-leader-election-vi.md)
- [ ] **Trang trỏ hướng:** [Chứng chỉ](156-certificates-vi.md) trong thư mục chỉ có 6 dòng, không thay thế được module quản lý certificate. ⏳ **Nợ #7** — kiểm tra hạn, gia hạn và xoay CA cần quy trình `kubeadm certs`; trả ở [giai đoạn 18](#giai-đoạn-18--vòng-đời-chứng-chỉ). Đọc xong bài này đừng gạch chủ đề certificate ra khỏi danh sách.

**Thực hành:** [Quản trị một Cluster](189-administer-cluster-vi.md) · [Di chuyển control plane được nhân bản sang dùng Cloud Controller Manager](198-controller-manager-leader-migration-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 **Lab 12 — Vận hành vòng đời node** — chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab).

**Checkpoint:** mô phỏng bảo trì một node: cordon → drain → tắt máy → bật lại → uncordon, và quan sát workload dịch chuyển. Kiểm tra graceful node shutdown có được kích hoạt không.

---

### Giai đoạn 13 — Lập lịch và workload nâng cao

**Không bắt buộc với admin mới.** Phần lớn là tính năng alpha/beta hoặc dành cho nền tảng chuyên biệt (AI/HPC, GPU). Đọc khi đã vững giai đoạn 1–12 hoặc khi công việc thực sự cần.

- [ ] [Cấp phát tài nguyên động](149-dynamic-resource-allocation-vi.md) — DRA; cách cấp phát GPU và thiết bị chuyên dụng thế hệ mới.
- [ ] [Các thực hành tốt về DRA dành cho quản trị viên cluster](172-cluster-admin-dra-vi.md)
- [ ] [Hướng dẫn tăng cường bảo mật — Cấp phát tài nguyên động](125-hardening-dra-vi.md) — phần hoãn lại từ giai đoạn 9.
- [ ] [Nhóm lập lịch](59-scheduling-group-vi.md)
- [ ] [PodGroup API](75-podgroup-api-vi.md)
- [ ] [Vòng đời của PodGroup](76-podgroup-lifecycle-vi.md)
- [ ] [Workload API](77-workload-api-vi.md)
- [ ] [Gián đoạn và độ ưu tiên của Pod Group](78-workload-disruption-priority-vi.md)
- [ ] [Các chính sách lập lịch PodGroup](79-workload-policies-vi.md)
- [ ] [Lập lịch workload nhận biết topology (Workload API)](80-workload-topology-scheduling-vi.md)
- [ ] [Lập lịch theo nhóm](150-gang-scheduling-vi.md) — hoặc chạy hết cả nhóm, hoặc không chạy gì; thiết yếu cho huấn luyện phân tán.
- [ ] [Lập lịch PodGroup](151-podgroup-scheduling-vi.md)
- [ ] [Preemption nhận biết workload](152-workload-aware-preemption-vi.md)
- [ ] [Lập lịch workload nhận biết topology (scheduling)](153-topology-aware-scheduling-vi.md)
- [ ] [Hướng dẫn tăng cường bảo mật — Cấu hình Scheduler](124-hardening-scheduler-vi.md) — phần hoãn lại từ giai đoạn 9.

**Thực hành:** [Tăng cường bảo mật cho Cấp phát tài nguyên động trong cluster của bạn](211-hardening-dra-tasks-vi.md) · [Gán thiết bị cho Pod và Container](268-assign-resources-vi.md) · [Truy cập metadata thiết bị DRA](269-access-dra-device-metadata-vi.md) · [Cấp phát thiết bị cho workload bằng DRA](270-allocate-devices-dra-vi.md) · [Thiết lập DRA trong một cluster](271-set-up-dra-cluster-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 **Lab 13 — DRA** (tùy chọn) — chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab). Chỉ làm được nếu lab có GPU hoặc thiết bị chuyên dụng.

**Checkpoint:** nếu cluster có GPU, cấp phát một GPU cho Pod bằng DRA. Nếu không, chỉ cần giải thích được DRA khác device plugin truyền thống ở điểm nào.

---

### Giai đoạn 14 — Khả năng mở rộng

**Dành cho platform administrator / người phát triển operator.**

- [ ] [Mở rộng Kubernetes](177-extend-kubernetes-vi.md) — bản đồ tất cả các điểm mở rộng.
- [ ] [Mở rộng Kubernetes API](178-api-extension-vi.md)
- [ ] [Tài nguyên tùy chỉnh](179-custom-resources-vi.md) — CRD; trọng tâm: bảng so sánh CRD với aggregated API để biết khi nào dùng cái nào.
- [ ] [Tầng tổng hợp API của Kubernetes](180-apiserver-aggregation-vi.md)
- [ ] [Mẫu Operator](181-operator-vi.md) — CRD + controller; mô hình vận hành ứng dụng stateful phức tạp.
- [ ] [Các phần mở rộng về Tính toán, Lưu trữ và Mạng](182-compute-storage-net-vi.md)
- [ ] [Device Plugin](184-device-plugins-vi.md) — cách cũ để expose GPU/thiết bị, so sánh với DRA ở giai đoạn 13.

**Thực hành:** [Khắc phục sự cố Topology Management](313-debug-topology-vi.md) · [Di trú object Kubernetes bằng Storage Version Migration](323-storage-version-migration-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 **Lab 14 — CRD và Operator** — chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab).

**Đã đọc ở giai đoạn 5:** [Network Plugin](183-network-plugins-vi.md) — nếu cần xem lại trong ngữ cảnh mở rộng thì quay lại bài đó.

**Checkpoint:** tạo một CRD đơn giản, `kubectl apply` một custom resource và đọc lại bằng `kubectl get`. Giải thích được vì sao CRD không có controller thì chỉ là kho lưu dữ liệu.

---

### Giai đoạn 15 — Windows, nếu môi trường có node Windows

Bỏ qua hoàn toàn nếu cluster chỉ có Linux.

- [ ] [Windows trong Kubernetes](174-windows-vi.md)
- [ ] [Windows containers trong Kubernetes](175-windows-intro-vi.md) — trọng tâm: những gì **không** tương đương Linux.
- [ ] [Hướng dẫn chạy Windows container trong Kubernetes](176-windows-user-guide-vi.md)
- [ ] [Mạng trên Windows](89-windows-networking-vi.md)
- [ ] [Lưu trữ trên Windows](106-windows-storage-vi.md)
- [ ] [Quản lý tài nguyên cho các node Windows](112-windows-resource-management-vi.md)
- [ ] [Bảo mật cho các node Windows](131-windows-security-vi.md)

**Thực hành:** [Cấu hình GMSA cho Pod và container Windows](273-configure-gmsa-vi.md) · [Cấu hình RunAsUserName cho Pod và container Windows](278-configure-runasusername-vi.md) · [Tạo một Windows HostProcess Pod](281-create-hostprocess-pod-vi.md) · [Mẹo debug Windows](315-debug-windows-vi.md) — làm sau khi đọc hết nhóm bài trên, trước khi mở lab.

- [ ] 🧪 **Lab 15 — Node Windows** (tùy chọn) — chưa viết, xem [bản đồ lab](labs/README.md#4-bản-đồ-lab). Cần thêm một VM Windows Server.

**Checkpoint:** join một node Windows vào cluster và chạy được một workload Windows có Service.

---

## Phần II — Vận hành cluster

Học hết 15 giai đoạn trên là bạn có **nền lý thuyết**. Phần dưới là **thực hành vận hành** — kỹ năng thực sự phân biệt người biết Kubernetes với người vận hành được Kubernetes, và cũng là phần chiếm tỷ trọng lớn nhất trong kỳ thi CKA.

Trang mục gốc của cả nhánh là [Tác vụ](367-tasks-index-vi.md). Các trang này **đã có bản dịch** trong thư mục — file mang số từ `186` trở lên, thuộc nhánh `/docs/tasks/` của kubernetes.io. Mỗi mục dưới đây trỏ thẳng vào bản dịch; hai trang chưa dịch được đánh dấu rõ. Danh mục đầy đủ nhóm này nằm ở [Phần 15–18 của README](README.md). Làm theo thứ tự checkpoint dưới đây, mỗi checkpoint làm trên cluster thật rồi mới sang checkpoint kế.

**Ba món nợ lab được trả ở phần này**, xem [Sổ nợ lộ trình](#sổ-nợ-lộ-trình) ở đầu file: nợ **#7** (vòng đời certificate) trả ở giai đoạn 18, nợ **#8** (backup/restore etcd) trả ở giai đoạn 19, nợ **#6** (mã hóa Secret at rest) trả ở giai đoạn 22.

### Giai đoạn 16 — Vòng đời node

**Mục tiêu:** giữ node khỏe qua vòng đời của nó — đưa vào, rút ra, bảo trì mà không làm gián đoạn workload.

- [ ] [Drain một node an toàn](255-safely-drain-node-vi.md) — cordon, drain, uncordon; liên hệ bài [53](53-disruptions-vi.md) và [143](143-api-eviction-vi.md).
- [ ] [Thêm node worker Linux](215-adding-linux-nodes-vi.md)
- [ ] [Thêm node worker Windows](216-adding-windows-nodes-vi.md)
- [ ] [Cấp phát dư dung lượng Node cho Cluster](250-node-overprovisioning-vi.md)

**Checkpoint:** drain `k8s-worker2` với `--ignore-daemonsets`, tắt máy, bật lại rồi `uncordon`; chứng minh Pod đã chuyển sang worker còn lại và quay về sau khi uncordon. Rời `k8s-worker2` khỏi cluster rồi join lại bằng token tự tạo bằng `kubeadm token create --print-join-command`.

### Giai đoạn 17 — Nâng cấp cluster

**Mục tiêu:** nâng cluster lên minor version kế tiếp mà không mất dịch vụ, đúng thứ tự và đúng lệnh.

- [ ] [Nâng cấp cluster kubeadm](221-kubeadm-upgrade-vi.md) — quy trình chuẩn, liên hệ bảng version skew ở bài [02](02-create-cluster-kubeadm-vi.md).
- [ ] [Nâng cấp node Linux](222-upgrading-linux-nodes-vi.md)
- [ ] [Nâng cấp các node Windows](223-upgrading-windows-nodes-vi.md)
- [ ] [Thay đổi package repository của Kubernetes](217-change-package-repository-vi.md)
- [ ] [Nâng cấp một Cluster](195-cluster-upgrade-vi.md) — góc nhìn tổng quát, không riêng kubeadm.

**Checkpoint:** nâng cluster lab lên một minor version: control plane trước (`kubeadm upgrade apply`), rồi từng worker (`kubeadm upgrade node`), mỗi node theo vòng drain → nâng gói → restart kubelet → uncordon. `kubectl get nodes` phải hiện phiên bản mới ở **cả ba** node. Giải thích được vì sao `kubeadm upgrade apply` xong mà cột VERSION chưa đổi.

### Giai đoạn 18 — Vòng đời chứng chỉ

**Mục tiêu:** kiểm soát vòng đời certificate: biết cái nào sắp hết hạn, gia hạn được, và cấp được certificate mới.

> ⏳ **Nợ #9 — 6/7 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

> ✅ **Trả nợ #7 — Quản lý vòng đời certificate.** Nợ phát sinh ở [giai đoạn 12](#giai-đoạn-12--quản-trị-cluster-nâng-cao), bài [156](156-certificates-vi.md) — bài đó chỉ là trang trỏ hướng sáu dòng, không dạy thao tác nào. **Đọc lại bài [156](156-certificates-vi.md) trước khi làm giai đoạn 18.**

- [ ] [TLS](396-tls-index-vi.md) — trang mục của nhóm, xem trước để biết nhóm gồm những gì. ⏳
- [ ] [Quản lý certificate với kubeadm](219-kubeadm-certs-vi.md) — kiểm tra hạn, gia hạn, xoay CA. Bài xương sống của giai đoạn 18.
- [ ] [Cấu hình xoay vòng certificate cho kubelet](398-certificate-rotation-vi.md) — kubelet tự gia hạn client certificate của chính nó. ⏳
- [ ] [Tạo certificate thủ công](191-certificates-manual-vi.md) — chính là trang mà bài [156](156-certificates-vi.md) trỏ tới. ⏳
- [ ] [Quản lý TLS certificate trong cluster](399-managing-tls-in-a-cluster-vi.md) — dùng CertificateSigningRequest API và signer của cluster. ⏳
- [ ] [Cấp certificate cho một API client bằng CertificateSigningRequest](397-certificate-issue-client-csr-vi.md) — tạo danh tính người dùng mới, nối tiếp bài [120](120-rbac-good-practices-vi.md). ⏳
- [ ] [Xoay vòng CA certificate thủ công](400-manual-rotation-of-ca-certificates-vi.md) — **thao tác nguy hiểm nhất nhóm**: 12 bước trên cluster đang chạy, làm sai là mất quyền truy cập cluster. Chỉ làm trên cluster lab. ⏳
- [ ] [PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/) — thuộc nhánh `/docs/setup/`, **chưa có bản dịch trong thư mục**; đọc bản gốc.

**Checkpoint:** chạy `kubeadm certs check-expiration` và đọc được từng dòng. Cấp một danh tính `myuser` trọn vẹn bằng CertificateSigningRequest, xác nhận bằng `kubectl --context myuser auth whoami`, rồi chứng minh nó **chưa làm được gì** cho tới khi có RoleBinding. Giải thích được vì sao certificate cấp qua `certificates.k8s.io` không nên giả định xác thực được với CA gốc của cluster.

### Giai đoạn 19 — etcd, backup và khôi phục thảm họa

**Mục tiêu:** bảo vệ và khôi phục được trạng thái cluster — kỹ năng phân định người vận hành được production.

> ✅ **Trả nợ #8 — Backup và restore etcd.** Nợ phát sinh ở [giai đoạn 8](#giai-đoạn-8--dựng-cluster-bằng-kubeadm): dựng được cluster nhưng chưa có quy trình `etcdctl` và khôi phục.

- [ ] [Vận hành cluster etcd cho Kubernetes](197-configure-upgrade-etcd-vi.md) — `etcdctl snapshot save`, khôi phục, nâng cấp etcd, thay thế member lỗi, chống phân mảnh.

**Checkpoint:** **bắt buộc làm thật:** `etcdctl snapshot save` → xóa vài Deployment → khôi phục từ snapshot → chứng minh cluster trở về trạng thái cũ. Giải thích được vì sao phải dừng toàn bộ API server trước khi restore, và phân biệt được `etcdctl` với `etcdutl`.

### Giai đoạn 20 — Cấu hình lại cluster đang chạy

**Mục tiêu:** đổi cấu hình một cluster đang chạy mà không phải dựng lại.

> ⏳ **Nợ #9 — 3/6 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

- [ ] [Cấu hình lại một cluster kubeadm](220-kubeadm-reconfigure-vi.md)
- [ ] [Cấu hình cgroup driver](218-configure-cgroup-driver-vi.md) — nối tiếp bài [00](00-container-runtimes-vi.md). ⏳
- [ ] [Thiết lập tham số kubelet qua file cấu hình](224-kubelet-config-file-vi.md) — nối tiếp bài [04](04-kubelet-integration-vi.md). ⏳
- [ ] [Bật hoặc tắt Feature Gate](196-configure-feature-gates-vi.md)
- [ ] [Bật hoặc tắt một Kubernetes API](207-enable-disable-api-vi.md)
- [ ] [Dành riêng tài nguyên tính toán cho các System Daemon](253-reserve-compute-resources-vi.md) — nối tiếp bài [110](110-manage-resources-containers-vi.md) và [142](142-node-pressure-eviction-vi.md). ⏳

**Checkpoint:** đổi một tham số kubelet qua file cấu hình rồi chứng minh nó có hiệu lực; bật một feature gate và kiểm chứng; đặt `--system-reserved` rồi đọc lại `Allocatable` của node thấy giảm đúng lượng đã dành. Giải thích được vì sao `kubeadm upgrade` không dùng để cấu hình lại cluster.

### Giai đoạn 21 — DNS, CNI và kube-proxy

**Mục tiêu:** vận hành tầng mạng: DNS, NetworkPolicy và dải IP của Service.

> ⏳ **Nợ #9 — 14/19 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

- [ ] [Tùy chỉnh DNS Service](204-dns-custom-nameservers-vi.md) — cấu hình CoreDNS, nối tiếp bài [10](10-dns-pod-service-vi.md).
- [ ] [Sử dụng CoreDNS cho Service Discovery](199-coredns-vi.md) — nâng cấp và chuyển đổi sang CoreDNS.
- [ ] [Gỡ lỗi phân giải DNS](205-dns-debugging-resolution-vi.md)
- [ ] [Sử dụng NodeLocal DNSCache](251-nodelocaldns-vi.md) ⏳
- [ ] [Tự động co giãn DNS Service](206-dns-horizontal-autoscaling-vi.md)
- [ ] [Khai báo Network Policy](201-declare-network-policy-vi.md) — nối tiếp bài [84](84-network-policies-vi.md).
- [ ] [Cài đặt một Network Policy Provider](243-network-policy-provider-vi.md) — trang mục, dẫn sang [Antrea](244-antrea-network-policy-vi.md), [Calico](245-calico-network-policy-vi.md), [Cilium](246-cilium-network-policy-vi.md), [kube-router](247-kube-router-network-policy-vi.md), [Weave](249-weave-network-policy-vi.md). ⏳
- [ ] **Đọc như tài liệu lịch sử:** [Romana cho NetworkPolicy](248-romana-network-policy-vi.md) — Romana đã ngừng phát triển; chỉ đọc khi tiếp quản cluster cũ còn dùng nó. ⏳
- [ ] [Hướng dẫn sử dụng IP Masquerade Agent](212-ip-masq-agent-vi.md) ⏳
- [ ] [Mạng](391-network-index-vi.md) — trang mục nhóm `tasks/network/`. ⏳
- [ ] [Thêm entry vào file /etc/hosts của Pod bằng HostAliases](392-customize-hosts-file-for-pods-vi.md) — `hostAliases`, nối tiếp bài [10](10-dns-pod-service-vi.md) và [57](57-pod-hostname-vi.md). ⏳
- [ ] [Mở rộng dải IP của Service](393-extend-service-ip-ranges-vi.md) — ServiceCIDR và IPAddress API, nối tiếp bài [88](88-cluster-ip-allocation-vi.md). ⏳
- [ ] [Cấu hình lại ServiceCIDR mặc định của Kubernetes](394-reconfigure-default-service-ip-ranges-vi.md) — **đổi dải Service IP mặc định của cluster đang chạy**; làm sau bài trên. ⏳
- [ ] [Kiểm chứng dual-stack IPv4/IPv6](395-validate-dual-stack-vi.md) — kiểm chứng dual-stack, nối tiếp bài [85](85-dual-stack-vi.md). ⏳

**Checkpoint:** sửa Corefile của CoreDNS thêm một domain chuyển tiếp rồi kiểm chứng bằng `nslookup` từ trong Pod. Dùng quy trình gỡ lỗi DNS của bài để tìm nguyên nhân một Pod không phân giải được tên. Viết một NetworkPolicy chặn toàn bộ ingress rồi mở đúng một cổng, chứng minh bằng `curl` từ Pod khác.

### Giai đoạn 22 — Audit và mã hóa dữ liệu

**Mục tiêu:** biết ai đã làm gì trên cluster, và bảo vệ dữ liệu nhạy cảm khi nằm trên đĩa.

> ⏳ **Nợ #9 — 2/6 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

> ✅ **Trả nợ #6 — Mã hóa Secret at rest.** Nợ phát sinh ở [giai đoạn 3b](#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod), bài [109](109-secret-vi.md) — bài đó nói rõ Secret **chỉ mã hóa base64** và hoãn phần encryption at rest sang đây. **Đọc lại bài [109](109-secret-vi.md) trước khi làm giai đoạn 22.**

- [ ] [Kiểm toán (Auditing)](306-audit-vi.md) — audit policy và backend.
- [ ] [Mã hóa dữ liệu bí mật khi lưu trữ](208-encrypt-data-vi.md) — phần còn thiếu của bài [109](109-secret-vi.md).
- [ ] [Giải mã dữ liệu bí mật đã được mã hóa khi lưu trữ](202-decrypt-data-vi.md)
- [ ] [Dùng một KMS provider để mã hóa dữ liệu](213-kms-provider-vi.md)
- [ ] [Bảo mật một Cluster](256-securing-a-cluster-vi.md) — nối tiếp bài [129](129-security-checklist-vi.md). ⏳
- [ ] [Xác minh các artifact Kubernetes đã ký](261-verify-signed-artifacts-vi.md) ⏳

**Checkpoint:** bật audit log với một policy tối thiểu rồi tìm lại được chính request của mình trong log. Bật mã hóa Secret at rest, tạo Secret mới, rồi chứng minh bằng `etcdctl get` rằng giá trị trong etcd không còn đọc được dưới dạng thường. Đây là chỗ **trả nợ #6**.

### Giai đoạn 23 — Giám sát và cảnh báo

**Mục tiêu:** biết cluster đang khỏe hay ốm qua số liệu, không qua cảm giác.

> ⏳ **Nợ #9 — 3/3 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

- [ ] [Pipeline metrics tài nguyên](311-resource-metrics-pipeline-vi.md) — metrics-server, điều kiện cho HPA ở bài [72](72-horizontal-pod-autoscale-vi.md). ⏳
- [ ] [Các công cụ giám sát tài nguyên](312-resource-usage-monitoring-vi.md) ⏳
- [ ] [Giám sát sức khỏe của Node](310-monitor-node-health-vi.md) — node-problem-detector. ⏳

**Checkpoint:** triển khai metrics-server và chạy được `kubectl top node` / `kubectl top pod`. Giải thích được metrics-server khác Prometheus ở điểm nào và vì sao HPA cần nó. Cài node-problem-detector và tạo ra một điều kiện node bất thường để thấy nó được báo cáo.

### Giai đoạn 24 — Xử lý sự cố

**Mục tiêu:** lần ra nguyên nhân sự cố theo quy trình, không đoán mò.

> ⏳ **Nợ #9 — 5/10 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

- [ ] [Khắc phục sự cố cluster](305-debug-cluster-vi.md) — nối tiếp bài [09](09-troubleshooting-kubeadm-vi.md). ⏳
- [ ] [Debug node Kubernetes bằng crictl](307-crictl-vi.md) — công cụ thay `docker` khi debug node.
- [ ] [Debug các node Kubernetes bằng kubectl](308-kubectl-node-debug-vi.md)
- [ ] [Khắc phục sự cố kubectl](314-troubleshoot-kubectl-vi.md)
- [ ] [Gỡ lỗi Pod](299-debug-pods-vi.md) ⏳
- [ ] [Gỡ lỗi Service](301-debug-service-vi.md) — quy trình lần từ Service về Pod.
- [ ] [Gỡ lỗi Pod đang chạy](300-debug-running-pod-vi.md) — dùng ephemeral container ở bài [52](52-ephemeral-containers-vi.md). ⏳
- [ ] [Xác định nguyên nhân Pod bị lỗi](303-determine-reason-pod-failure-vi.md) ⏳
- [ ] [Gỡ lỗi Init Container](298-debug-init-containers-vi.md) ⏳
- [ ] [Gỡ lỗi một StatefulSet](302-debug-statefulset-vi.md)

**Checkpoint:** tự tạo bốn sự cố trên `k8s-worker2` rồi chẩn đoán bằng đúng công cụ của bài: Pod `Pending`, Pod `CrashLoopBackOff`, Service không có endpoint, node `NotReady`. Với mỗi ca, nói được **bước nào trong quy trình** đã chỉ ra nguyên nhân. Dùng được `crictl ps` khi API server không trả lời.

### Giai đoạn 25 — Quản trị tài nguyên theo namespace

**Mục tiêu:** chia cluster cho nhiều nhóm mà không để nhóm nào ăn hết tài nguyên.

> ⏳ **Nợ #9 — 11/13 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

- [ ] [Chia sẻ một Cluster bằng Namespace](242-namespaces-tasks-vi.md) ⏳
- [ ] [Quản lý tài nguyên Memory, CPU và API](228-manage-resources-tasks-vi.md) — trang mục của loạt bài thực hành, nối tiếp bài [133](133-limit-range-vi.md) và [134](134-resource-quotas-vi.md). Sáu bài con: [ràng buộc CPU](229-cpu-constraint-namespace-vi.md), [CPU mặc định](230-cpu-default-namespace-vi.md), [ràng buộc memory](231-memory-constraint-namespace-vi.md), [memory mặc định](232-memory-default-namespace-vi.md), [quota memory/CPU](233-quota-memory-cpu-namespace-vi.md), [quota số Pod](234-quota-pod-namespace-vi.md). ⏳
- [ ] [Cấu hình Quota cho các đối tượng API](252-quota-api-object-vi.md) ⏳
- [ ] [Kiểm soát các chính sách quản lý CPU trên Node](200-cpu-management-policies-vi.md) — nối tiếp bài [74](74-resource-managers-vi.md).
- [ ] [Kiểm soát các chính sách quản lý bộ nhớ trên một Node](235-memory-manager-vi.md) — memory manager nhận biết NUMA. ⏳
- [ ] [Kiểm soát các chính sách quản lý topology trên một node](259-topology-manager-vi.md) ⏳
- [ ] [Quảng bá Extended Resource cho một Node](209-extended-resource-node-vi.md)

**Checkpoint:** đặt ResourceQuota và LimitRange cho một namespace, tạo Pod vượt quota và đọc đúng thông báo từ chối. Đặt quota theo số lượng object và chứng minh nó chặn. Giải thích được khác biệt giữa quota (trần của cả namespace) và LimitRange (mặc định và trần cho từng Pod).

### Giai đoạn 26 — Vận hành lưu trữ

**Mục tiêu:** vận hành lưu trữ sau khi đã cấp phát: đổi mặc định, đổi chính sách, giới hạn tiêu thụ.

> ⏳ **Nợ #9 — 1/4 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

- [ ] [Thay đổi StorageClass mặc định](192-change-default-storage-class-vi.md)
- [ ] [Thay đổi Reclaim Policy của một PersistentVolume](194-change-pv-reclaim-policy-vi.md) — nối tiếp bài [92](92-persistent-volumes-vi.md).
- [ ] [Thay đổi access mode của một PersistentVolume](193-change-pv-access-mode-vi.md)
- [ ] [Giới hạn mức tiêu thụ lưu trữ](227-limit-storage-consumption-vi.md) ⏳

**Checkpoint:** đổi StorageClass mặc định và chứng minh PVC mới dùng class mới. Đổi `reclaimPolicy` của một PV đang tồn tại từ `Delete` sang `Retain`, xóa PVC, rồi chứng minh dữ liệu còn nguyên.

### Giai đoạn 28 — Mở rộng Kubernetes

**Mục tiêu:** mở rộng chính Kubernetes: thêm kiểu tài nguyên mới và thêm API server của riêng bạn.

> ⏳ **Nợ #9 — 12/12 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

Nhóm thực hành của [giai đoạn 14](#giai-đoạn-14--khả-năng-mở-rộng). Thứ tự dưới đây theo đúng
thứ tự kubernetes.io hiển thị, không theo số file.

- [ ] [Mở rộng Kubernetes](373-extend-kubernetes-index-vi.md) — trang mục nhóm `tasks/extend-kubernetes/`. ⏳
- [ ] [Cấu hình tầng tổng hợp](374-configure-aggregation-layer-vi.md) — nối tiếp bài [180](180-apiserver-aggregation-vi.md); mọi cờ `--requestheader-*` phải chính xác. ⏳
- [ ] [Sử dụng Custom Resource](376-custom-resources-index-vi.md) — trang mục con về custom resource. ⏳
- [ ] [Mở rộng Kubernetes API bằng CustomResourceDefinition](378-custom-resource-definitions-vi.md) — **bài xương sống của nhóm**, nối tiếp bài [179](179-custom-resources-vi.md). ⏳
- [ ] [Các phiên bản trong CustomResourceDefinition](377-custom-resource-definition-versioning-vi.md) — nhiều version của một CRD, conversion webhook; nối tiếp bài [32](32-storage-version-vi.md). ⏳
- [ ] [Thiết lập một Extension API Server](380-setup-extension-api-server-vi.md) — dựng extension API server, làm sau bài [374](374-configure-aggregation-layer-vi.md). ⏳
- [ ] [Cấu hình nhiều Scheduler](375-configure-multiple-schedulers-vi.md) — chạy scheduler thứ hai, nối tiếp bài [137](137-kube-scheduler-vi.md). ⏳
- [ ] [Dùng HTTP Proxy để truy cập Kubernetes API](379-http-proxy-access-api-vi.md) — `kubectl proxy`, nối tiếp bài [164](164-proxies-vi.md). ⏳
- [ ] [Dùng SOCKS5 Proxy để truy cập Kubernetes API](382-socks5-proxy-access-api-vi.md) — SSH SOCKS5 tunnel và `proxy-url` trong kubeconfig. ⏳
- [ ] [Thiết lập dịch vụ Konnectivity](381-setup-konnectivity-vi.md) — đường control plane → node, nối tiếp bài [24](24-control-plane-node-communication-vi.md). ⏳
- [ ] [Mở rộng kubectl bằng plugin](372-kubectl-plugins-vi.md) — mở rộng kubectl bằng plugin; thuộc nhóm `tasks/extend-kubectl/`. ⏳

**Checkpoint:** tạo một CRD, apply một custom resource rồi đọc lại bằng `kubectl get`. Thêm validation vào CRD và chứng minh object sai bị từ chối. Giải thích được vì sao CRD không có controller thì chỉ là kho lưu dữ liệu, và khi nào phải dùng aggregated API thay vì CRD.

### Giai đoạn 29 — DaemonSet, Job nâng cao và thiết bị chuyên dụng

**Mục tiêu:** vận hành workload chạy trên mọi node và workload chạy một lần rồi thôi.

> ⏳ **Nợ #9 — 8/8 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

Nhóm thực hành của [giai đoạn 4](#giai-đoạn-4--workload-controller) (DaemonSet, Job) cùng hai
bài thiết bị chuyên dụng của [giai đoạn 7b](#7b-chính-sách-giới-hạn-tài-nguyên) và
[giai đoạn 13](#giai-đoạn-13--lập-lịch-và-workload-nâng-cao).

- [ ] [Quản lý các daemon của cluster](384-manage-daemon-index-vi.md) — trang mục nhóm `tasks/manage-daemon/`. ⏳
- [ ] [Xây dựng một DaemonSet cơ bản](385-create-daemon-set-vi.md) — nối tiếp bài [66](66-daemonset-vi.md). ⏳
- [ ] [Thực hiện rolling update trên một DaemonSet](388-update-daemon-set-vi.md) — rolling update cho DaemonSet; đọc trước bài rollback ngay dưới. ⏳
- [ ] [Thực hiện rollback trên một DaemonSet](387-rollback-daemon-set-vi.md) — rollback DaemonSet, dựa trên bài trên. ⏳
- [ ] [Chỉ chạy Pod trên một số Node nhất định](386-pods-some-nodes-vi.md) — chạy Pod trên một số node, nối tiếp bài [138](138-assign-pod-node-vi.md). ⏳
- [ ] [Xử lý các lần Pod thất bại có thể thử lại và không thể thử lại bằng Pod failure policy](383-pod-failure-policy-vi.md) — `podFailurePolicy`, nối tiếp bài [67](67-job-vi.md). ⏳
- [ ] [Quản lý HugePages](390-scheduling-hugepages-vi.md) — hugepages, nối tiếp bài [110](110-manage-resources-containers-vi.md); thuộc giai đoạn 7b. ⏳
- [ ] [Lập lịch GPU](389-scheduling-gpus-vi.md) — GPU qua device plugin, nối tiếp bài [184](184-device-plugins-vi.md); thuộc giai đoạn 13. ⏳

**Checkpoint:** triển khai một DaemonSet, thực hiện rolling update rồi rollback về revision trước, quan sát `ControllerRevision`. Chạy một Job có `podFailurePolicy` và chứng minh Job dừng đúng theo exit code đã khai báo thay vì thử lại tới `backoffLimit`.

### Giai đoạn 30 — Truy cập ứng dụng trong cluster

**Mục tiêu:** đưa được ứng dụng trong cluster ra cho người dùng bên ngoài chạm tới.

> ⏳ **Nợ #9 — 5/5 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

Nhóm thực hành đi kèm [giai đoạn 5](#giai-đoạn-5--mạng-nền-tảng).

- [ ] [Truy cập ứng dụng trong một cluster](368-access-application-cluster-index-vi.md) — trang mục nhóm `tasks/access-application-cluster/`. ⏳
- [ ] [Triển khai và Truy cập Kubernetes Dashboard](371-web-ui-dashboard-vi.md) — **Dashboard đã deprecated** ở upstream, khuyến nghị chuyển sang Headlamp; đọc để biết, không cài như add-on chuẩn. ⏳
- [ ] [Dùng Service để truy cập một ứng dụng trong cluster](370-service-access-application-cluster-vi.md) — expose ứng dụng bằng Service, nối tiếp bài [82](82-service-vi.md). ⏳
- [ ] [Truy cập các Service đang chạy trên cluster](369-access-cluster-services-vi.md) — apiserver proxy URL, nối tiếp bài [190](190-access-cluster-api-vi.md). ⏳

**Checkpoint:** expose một Deployment bằng NodePort rồi truy cập từ máy host. Dùng `kubectl port-forward` để gọi một Pod không có Service. Giải thích được ba đường vào cluster — port-forward, apiserver proxy, Service — khác nhau ở đâu và mỗi đường hợp với việc gì.

### Giai đoạn 27 — Di chuyển khỏi dockershim (cluster cũ)

**Mục tiêu:** tiếp quản được cluster đời cũ còn dùng Docker Engine.

> ⏳ **Nợ #9 — 6/6 bài trong mục này chưa có hai khối *Đọc bài này thế nào* và *Tự kiểm tra*** (đánh dấu ⏳ ở cuối dòng). Đọc được ngay, nhưng chưa có phần định hướng độ sâu. Xem [cách trả nợ #9](#nợ-9--hai-khối-hướng-dẫn-đọc-cho-nhánh-docstasks).

Chỉ cần khi tiếp quản cluster đời cũ còn dùng Docker Engine:

- [ ] [Chuyển đổi từ dockershim](236-migrating-from-dockershim-vi.md) ⏳
- [ ] [Tìm hiểu Container Runtime nào đang được dùng trên Node](239-find-out-runtime-vi.md) ⏳
- [ ] [Kiểm tra xem việc gỡ bỏ dockershim có ảnh hưởng không](238-check-dockershim-removal-vi.md) ⏳
- [ ] [Thay đổi Container Runtime sang containerd](237-change-runtime-containerd-vi.md) ⏳
- [ ] [Di chuyển các telemetry agent khỏi dockershim](240-migrating-telemetry-agents-vi.md) ⏳
- [ ] [Khắc phục sự cố lỗi liên quan tới CNI plugin](241-troubleshooting-cni-errors-vi.md) ⏳

---

**Checkpoint:** trên một node, xác định được container runtime đang dùng bằng lệnh của bài. Giải thích được dockershim là gì, vì sao bị gỡ, và những gì phải kiểm trước khi chuyển sang containerd. **Bỏ qua toàn bộ giai đoạn này nếu cluster của bạn đã dùng containerd** — cluster lab thuộc nhóm đó.

## Điều chỉnh so với bản phác thảo ban đầu

Năm thay đổi có chủ đích so với thứ tự phác thảo, để không còn bài nào phải tham chiếu kiến thức của giai đoạn sau:

1. **Bài [38](38-self-healing-vi.md) (Tự phục hồi) chuyển từ giai đoạn 1 xuống giai đoạn 4.** Nội dung bài này nói về cách Deployment, ReplicaSet và StatefulSet phục hồi workload — đọc ở giai đoạn 1 sẽ gặp toàn khái niệm chưa học.
2. **Bài [172](172-cluster-admin-dra-vi.md) (DRA cho quản trị viên) chỉ đặt ở giai đoạn 13**, bỏ khỏi giai đoạn 12. Nội dung phụ thuộc hoàn toàn vào DRA ở bài 149.
3. **Bài [183](183-network-plugins-vi.md) (Network Plugin) đặt chính ở giai đoạn 5**, giai đoạn 14 chỉ tham chiếu lại. Cần hiểu CNI trước khi cài CNI ở giai đoạn 8.
4. **Bài [00](00-container-runtimes-vi.md) tách làm hai lần dùng**: đọc lý thuyết ở giai đoạn 2 (cùng CRI và cgroup), thực hành cài đặt ở giai đoạn 8 (cùng kubeadm) — tránh việc cài runtime xong bỏ máy không dùng suốt sáu giai đoạn.
5. **Bài [72](72-horizontal-pod-autoscale-vi.md) và [73](73-vertical-pod-autoscale-vi.md) tách đọc khỏi thực hành**: đọc lý thuyết tại giai đoạn 4 cùng các workload controller, thực hành ở Lab 11b sau khi giai đoạn 11 dựng metrics-server. Phương án còn lại — cài metrics-server sớm ở giai đoạn 4 — bị loại vì nó buộc người học vận hành một add-on mà giai đoạn 11 mới giải thích, và làm hỏng nguyên tắc không nhảy cóc của cả lộ trình.

Ngoài ra bài [70](70-replicationcontroller-vi.md) và [117](117-pod-security-policy-vi.md) được đánh dấu là tài liệu lịch sử, không nằm trong mạch bắt buộc.

Những phần cố ý hoãn lại kiểu này được ghi tập trung ở [sổ nợ lab](labs/README.md#5-sổ-nợ-lab).
