# Lab 8a — Dựng cluster bằng kubeadm

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do Lab 6a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot). Repo `~/bookstore` ở tag `lab-7b` (hoặc bundle
> `~/bookstore-lab-7b.bundle` nếu bạn vừa restore snapshot).
> **Điểm kết thúc:** lab **phá cluster, dựng lại cluster và Bookstore chỉ từ repo cộng một bản dump,
> rồi restore `03-storage-ready`**. Không tạo snapshot mới. Repo có tag `lab-8a` và bundle nằm ngoài
> VM **trước** khi restore — restore xóa đĩa của cả ba VM, kể cả `~/bookstore`.
> **Sprint dự án:** Sprint 12 — Diễn tập dựng lại cluster của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 7b — Quota và giới hạn tài nguyên](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md).
> **Lab sau:** [Lab 8b — HA với stacked etcd](LAB-8B-HA-VOI-STACKED-ETCD.md), trên bộ VM riêng, bắt
> đầu từ tag `lab-8a`.
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng
[Giai đoạn 8 — Dựng cluster bằng kubeadm](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm)
— sáu bài [01](../01-install-kubeadm-vi.md), [02](../02-create-cluster-kubeadm-vi.md),
[03](../03-control-plane-flags-vi.md), [04](../04-kubelet-integration-vi.md),
[05](../05-dual-stack-support-vi.md), [09](../09-troubleshooting-kubeadm-vi.md), cộng bài thực hành
[214](../214-kubeadm-tasks-vi.md). Ba bài HA — [06](../06-ha-topology-vi.md),
[07](../07-setup-ha-etcd-with-kubeadm-vi.md), [08](../08-high-availability-vi.md) — **không** thuộc lab
này; chúng nằm ở Lab 8b, Lab 8c trên bộ VM riêng.

Trước khi mở B0, đọc lại [mục 4 — hợp đồng kỹ thuật](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật),
[mục 3.3 — hạ tầng nền](DU-AN-BOOKSTORE.md#33-hạ-tầng-nền-của-cluster),
[mục 5.4 — sao lưu repo](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm), hàng "8a" của
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint) và
[kế hoạch sprint 12](DU-AN-BOOKSTORE.md#sprint-12--lab-8a-diễn-tập-dựng-lại-cluster).

Bạn đã chạy kubeadm một lần ở [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md) theo kiểu chép lệnh. Sprint này
là nơi học thật phần đó, trên đúng cluster đang chạy Bookstore. Câu hỏi của ban quản lý rất ngắn:
**nếu mất cả cluster thì bao lâu Bookstore bán hàng lại được, và mất bao nhiêu đơn hàng?** Câu trả lời
phải là hai con số đo được — RTO (thời gian khôi phục) và RPO (lượng dữ liệu mất) — từ một cuộc diễn
tập có kế hoạch: đọc cluster đang chạy, ghi cách dựng nó vào repo, đưa bản sao lưu ra ngoài, phá, rồi
dựng lại cả cluster lẫn Bookstore **chỉ** từ repo và bản dump.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); ba thành phần nền
(Calico, Traefik qua Helm, local-path-provisioner) và image của dự án nằm ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Lab **không chép lại con
số phiên bản nào** trong phần văn bản: mọi giá trị của cluster đều đọc từ cluster thật, rồi được ghi
vào repo của bạn.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi cluster đã bị phá và dựng lại, Bookstore chạy lại trên cluster mới với dữ liệu của
bản dump, cluster đã về đúng `03-storage-ready`, **và** bạn chứng minh, giải thích được:

- Mỗi tiền điều kiện của bài 01 tồn tại để chặn lỗi gì — định danh duy nhất, cổng, swap tạm và swap
  bền, cgroup driver hai phía — bằng **giá trị thật** của ba node, không phải bằng cách cài lại.
- Ba gói `kubeadm`, `kubelet`, `kubectl` có ba vai trò tách biệt, và quy tắc lệch phiên bản **bất đối
  xứng**: kubelet không bao giờ được cao hơn API server.
- `kubeadm init` để lại hiện vật gì ở từng phase — `pki/`, năm kubeconfig, bốn manifest static Pod,
  ConfigMap `kubeadm-config` và `kubelet-config`, bootstrap token, CoreDNS, kube-proxy — và đọc được
  output của nó như một danh sách phase.
- Vì sao control plane chạy dạng static Pod; ai sở hữu mirror Pod; khi `kube-apiserver` hoặc cả máy
  control plane vắng mặt thì Bookstore **vẫn bán được hàng** và chính xác mất khả năng gì.
- Ba thành phần của lệnh `kubeadm join`, cái nào là bí mật, và tự tính lại hash CA bằng `openssl`.
- Ba `kind` trong file cấu hình kubeadm — `InitConfiguration`, `ClusterConfiguration`,
  `KubeletConfiguration` — mỗi cái có phạm vi gì; vì sao patch là lối thoát cho cấu hình khác nhau
  theo node; và vì sao cấu hình cluster phải nằm trong repo chứ không chỉ trong ConfigMap của chính
  cluster sắp mất.
- kubeadm ghi gì cho kubelet, `init` và `join` đi ngược chiều dữ liệu ra sao, và ai thắng khi các
  nguồn cấu hình kubelet mâu thuẫn.
- Cluster của bạn là single-stack, đọc từ ba nguồn độc lập, và quyết định đó chốt một lần lúc `init`.
- Gỡ cluster đúng thứ tự `drain` → `kubeadm reset` → `kubectl delete node`; vì sao `drain` của dự án
  bị PodDisruptionBudget chặn; hai thứ `kubeadm reset` không đặt lại.
- Thứ tự dựng nền: control plane → CNI → join → ingress controller và provisioner → ứng dụng → dữ liệu.
- RPO và RTO của Bookstore bằng con số đo được; vì sao bản dump logic khôi phục được dữ liệu mà
  **không** kéo theo mật khẩu role cũ.
- Đưa cluster về **đúng** `03-storage-ready` và chứng minh đó là cluster cũ chứ không phải cluster vừa
  dựng.
- Dùng bài 09 đúng công dụng: gặp triệu chứng thật rồi tra ngược về mục tương ứng.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong giai đoạn 8 | Ticket kiểm chứng |
| --- | --- |
| [01 — Cài đặt kubeadm](../01-install-kubeadm-vi.md) | B1 — định danh, cổng, swap, cgroup driver, ba gói, giữ phiên bản, skew kubelet; B6 — điều kiện bền qua reboot; B7 — phase `preflight` chạy thật |
| [02 — Tạo một cluster với kubeadm](../02-create-cluster-kubeadm-vi.md) | B1 — giải phẫu cluster: kubeconfig, CA, static Pod và mirror Pod, taint control plane, `NodeRestriction`, ba thành phần của lệnh join, ba bảng skew; B2 — `advertiseAddress` so với `controlPlaneEndpoint`; B4 — *Hạn chế*: một control plane, một etcd; B6 — *Dọn dẹp*, *Gỡ bỏ node*; B7 — khởi tạo control plane, cài Pod network add-on; B8 — thêm worker |
| [03 — Tùy chỉnh các thành phần với kubeadm API](../03-control-plane-flags-vi.md) | B2 — `ClusterConfiguration` thật, phạm vi toàn cục, ba `kind`, thư mục patch đúng quy ước tên; B7 — patch áp lúc sinh manifest, chỉ chạm đúng target |
| [04 — Cấu hình từng kubelet bằng kubeadm](../04-kubelet-integration-vi.md) | B2 — hai mẫu cấu hình, ConfigMap `kubelet-config` so với file trên ba node, nơi ghi CRI socket, thứ tự `ExecStart`; B8 — `join` tải ConfigMap xuống, `bootstrap-kubelet.conf` bị xóa sau TLS Bootstrap |
| [05 — Hỗ trợ dual-stack với kubeadm](../05-dual-stack-support-vi.md) | B2 — single-stack đọc từ ba nguồn, `podSubnet`/`serviceSubnet` một giá trị trong file cấu hình, `net.ipv6.conf.all.forwarding`; B8 — `podCIDRs` của cluster mới |
| [09 — Xử lý sự cố kubeadm](../09-troubleshooting-kubeadm-vi.md) | B4 — `kubectl` chết, đọc node bằng `crictl` và `journalctl`; B7 — CoreDNS `Pending` trước khi có CNI là thiết kế; [mục 4](#4-troubleshooting-của-lab-này) tra ngược mọi sự cố kubeadm về bài này |
| [214 — Quản trị với kubeadm](../214-kubeadm-tasks-vi.md) | B10 — trang mục lục: định vị chín trang con, việc nào vừa làm ([215](../215-adding-linux-nodes-vi.md) ở B8, [218](../218-configure-cgroup-driver-vi.md) ở B1, B2), việc nào thuộc giai đoạn sau |

Các phần dưới đây **không kiểm chứng được trong lab này**:

| Phần | Vì sao |
| --- | --- |
| Bài 02 — *Thêm control plane node*; "tắt một control plane mà **API** vẫn phục vụ" | Cần ba control plane và load balancer. Chuỗi chính chỉ có một control plane, nên B4 chứng minh vế đúng với topology này: **data plane** vẫn phục vụ. Vế API thuộc Lab 8b, Lab 8c với bài 06, 07, 08 |
| Bài 02 — *Chính sách chênh lệch phiên bản* ở dạng thử thật | Phải cài hai minor của `kubeadm`/`kubelet` lên node, phá bộ gói đã giữ ở [A4.3](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl). B1 kiểm phần đọc được; nâng minor thuộc [giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster) |
| Bài 02 — *Chuẩn bị các container image*, *Proxy API Server về localhost*, `kubeadm kubeconfig user` | Node lab có Internet; proxy là tiện ích; cấp quyền cho danh tính mới cần RBAC của [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) |
| Bài 03 — `UpgradeConfiguration`; `dns.disabled`, `corednsdeployment`; giá trị của các flag ví dụ | Chỉ có nghĩa khi `kubeadm upgrade` ([giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster)); thay CoreDNS thuộc [giai đoạn 21](../00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy); lab kiểm **cơ chế** `ClusterConfiguration` và patch, không kiểm từng flag |
| Bài 04 — cơ chế TLS Bootstrap cấp certificate | Cần vòng đời certificate của [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ). B8 kiểm **kết quả**: `bootstrap-kubelet.conf` biến mất, `kubelet.conf` xuất hiện |
| Bài 05 — dựng cluster dual-stack thật | Mạng [A1.2](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) chỉ có IPv4, và bài nói `kubeadm upgrade` không đổi được dải của cluster đã có. B2, B8 kiểm phần single-stack có chủ đích mà chính bài mô tả |
| Bài 09 — phần lớn các mục | Di sản lịch sử (RBAC v1.18/v1.17, Docker trên CentOS 7, Flatcar, Vagrant, cloud-controller-manager, lỗi nâng cấp v1.28) — cluster lab không tái tạo được |
| Bài 214 — bảy trang con còn lại | [216](../216-adding-windows-nodes-vi.md) cần VM Windows (Lab 15); [217](../217-change-package-repository-vi.md), [221](../221-kubeadm-upgrade-vi.md), [222](../222-upgrading-linux-nodes-vi.md), [223](../223-upgrading-windows-nodes-vi.md) thuộc giai đoạn 17; [219](../219-kubeadm-certs-vi.md) thuộc giai đoạn 18; [220](../220-kubeadm-reconfigure-vi.md) thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |

**Nợ lab:** [nợ #8](README.md#5-sổ-nợ-lab) — backup và restore etcd — phát sinh ở giai đoạn này và
**không** được trả ở đây. Lab khôi phục **dữ liệu ứng dụng** bằng bản dump logic của PostgreSQL và
khôi phục **trạng thái cluster** bằng cách dựng lại từ repo; không có lần nào đọc hay ghi etcd. Nợ
trả ở [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa). Lab
cũng **không** trả [nợ #7](README.md#5-sổ-nợ-lab): đọc hạn certificate không phải là quản lý vòng
đời của nó.

**Nợ kỹ thuật:** `TD-10` (bản backup nằm cùng cluster, chấp nhận có điều kiện ở 6b) được diễn tập
thật ở B5; **phát sinh `TD-14`** — cluster và nền dựng lại bằng tay theo `platform/README.md`, không
có backup etcd. `TD-06`, `TD-07` vẫn mở — xem
[sổ nợ của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án).

### 1.2. Thời lượng

5–6 giờ, chia hai phiên.

- **Phiên 1 — B0 tới B5, khoảng 2,5 giờ.** Cluster còn nguyên; dừng được giữa các ticket. Phiên kết
  thúc ở cuối B5, khi repo và bản dump đã nằm trên máy host.
- **Phiên 2 — B6 tới B11, khoảng 3 giờ, không dừng giữa chừng.** Từ B6 tới hết B9 cluster không phục
  vụ được gì, và trạng thái giữa chừng không có gate nào để quay lại ngoài snapshot. Đừng mở B6 nếu
  không còn đủ thời gian chạy tới hết B11.

Nghỉ giữa hai phiên lâu thì làm lại B5 trước B6: RPO tính từ lúc dump, bản dump cũ làm con số vô
nghĩa. Ba chỗ chờ **không có con số cam kết** — `kubeadm init` chờ control plane, node `Ready` sau
khi có CNI, ba VM boot lại — đều phụ thuộc mạng, ổ đĩa và cấu hình; lab viết chúng bằng
`kubectl wait --timeout` hoặc vòng lặp có điều kiện dừng.

---

## 2. Quy ước và an toàn

> **CẢNH BÁO — LAB NÀY PHÁ CLUSTER CÓ CHỦ ĐÍCH.** B4 dời manifest của `kube-apiserver` và tắt hẳn
> `lab-k8s-master`. B6 chạy `kubeadm reset` trên **cả ba node**: sau đó không còn cluster nào cho tới
> khi B7, B8 dựng cái mới, và cái mới **không** phải mốc `03-storage-ready` dù có cùng thành phần nền.
> **Đường về duy nhất là restore snapshot ở B11.** Không có snapshot thì không có lab.

**Gate bắt buộc trước B0.** Chạy trên **máy host Windows**, PowerShell. Không đủ ba dòng `PASS:` thì
dừng tại đây:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '03-storage-ready') { "PASS: $f" }
  else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** đúng ba dòng `PASS:`, không dòng `FAIL:`. `-ccontains` phân biệt hoa thường nên bắt được cả
lỗi gõ tên mốc. Sửa đường dẫn `.vmx` nếu VM nằm chỗ khác. **Giữ cửa sổ PowerShell này tới hết lab** —
B4 và B11 dùng lại `$vmrun`, `$vmx`; B5, B9, B11 dùng thêm `$masterUser`, `$masterHost`, `$dest`. Một VM thiếu mốc thì **không** chụp bù bằng trạng thái
hiện tại; dựng lại mốc bằng [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md) từ `02-net-ready`.

Các quy ước còn lại:

- **Phạm vi phá hoại.** Quy ước chung chỉ cho fault injection trên `lab-k8s-worker2`. Lab này phá cả
  control plane (B4) và cả ba node (B6), vì nội dung bài học chính là dựng lại cluster. Điều kiện
  đánh đổi nằm ở gate trên: mốc `03-storage-ready` có trên cả ba VM và lab kết thúc bằng restore.
  Không lab nào khác được viện dẫn điều này.
- Mọi lệnh `kubectl` chạy trên `lab-k8s-master`. Lệnh trên worker chạy qua `ssh <node> 'sudo ...'`
  từ master, như các lab trước. **Mở thêm một terminal SSH vào `lab-k8s-worker1`** — B4 cần nó khi
  master tắt.
- **Phiên shell trên master chết ít nhất ba lần**: B4 tắt master, B6 reboot, B11 restore. B0 ghi mọi
  biến vào `~/lab-evidence/8a/env.sh`; mỗi lần SSH lại, chạy `. ~/lab-evidence/8a/env.sh` trước.
- Repo dự án: `~/bookstore`. Thư mục làm việc: `~/lab-work/8a/`. Bằng chứng: `~/lab-evidence/8a/`.
  Cả ba nằm trên đĩa master — **sống qua `kubeadm reset` và reboot, mất khi restore**. B11 chép repo
  và bằng chứng ra máy host trước khi restore.
- **Vệ sinh bí mật.** Không ghi token, private key, nội dung kubeconfig hay mật khẩu vào bằng chứng
  hoặc repo. Lệnh `kubeadm join` chỉ nằm ở `~/lab-work/8a/` với quyền `600`; output của
  `kubeadm init` vào bằng chứng sau khi che token. `admin.conf` và `super-admin.conf` không rời master
  — bài 02 nói cả hai là bí mật.
- **B1 tới B3 chỉ đọc** trên node: không sửa file nào dưới `/etc/kubernetes`, `/var/lib/kubelet`,
  `/etc/containerd`. Thay đổi duy nhất là một bootstrap token ở B1. Việc phá bắt đầu ở B4 và luôn có
  dòng in đậm báo trước.
- **Không cài thêm phần mềm.** Lab chỉ cài lại **đúng** các thành phần đã có ở `03-storage-ready`, đúng
  phiên bản đang chạy. Helm đã nằm trên đĩa master từ Lab 5b. Công cụ dùng: `kubeadm`, `kubectl`,
  `crictl`, `helm`, `openssl`, `ss`, `iptables`, `curl`, `git`, `/dev/tcp` của bash. Không có `nc`
  và `ipvsadm` — lab nói rõ chỗ thay thế.
- Object thử nghiệm (Pod `lay-dump` ở B5) không mang label `app.kubernetes.io/part-of` và bị xóa khi
  xong. Lệnh imperative (`drain`, `scale`, `patch` Service của Traefik theo quy trình nền) chỉ dùng
  trong diễn tập hoặc đúng như `platform/README.md` ghi; ứng dụng luôn trở về trạng thái của repo bằng
  `./scripts/deploy.sh`.
- **Mọi con số của cluster đều đọc từ cluster thật** — IP node, endpoint, CRI socket, dải mạng,
  phiên bản, NodePort. Không con số nào được viết cứng vào gate.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. `FAIL:` trong script nghiệm thu
  nghĩa là ticket chưa xong. Với ticket phá hoại (B4, B6), script nghiệm thu **chính là cuộc diễn
  tập**: đọc hết, đoán trước kết quả từng khối, rồi mới chạy.

---

# Phần B — Sprint 12: Diễn tập dựng lại cluster

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `03-storage-ready`, repo ở tag `lab-7b`, hệ thống của sprint 11
dựng lại được chỉ từ repo; và ghi lại mọi giá trị của cluster mà các ticket sau cần — trước khi
cluster đó biến mất.

Gate ngắn A5.5, chạy nguyên văn, rồi ba lệnh riêng của mốc `03-storage-ready`:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

kubectl get storageclass
kubectl get ingressclass
kubectl get nodes -o custom-columns='NODE:.metadata.name,UNSCHEDULABLE:.spec.unschedulable'
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; có **đúng một** StorageClass mang hậu tố `(default)`; có
IngressClass `traefik`; cột `UNSCHEDULABLE` của cả ba node là `<none>`. Thiếu StorageClass mặc định
hoặc IngressClass nghĩa là cluster **không** ở `03-storage-ready` — restore cả ba VM về mốc đó trước.

Hồ sơ của mốc, chụp **trước** khi triển khai dự án — B11 dùng nó để chứng minh cluster sau restore là
cluster cũ:

```bash
mkdir -p ~/lab-work/8a ~/lab-evidence/8a
chmod 700 ~/lab-work/8a
{
  echo "=== $(date -Is) — truoc Lab 8a, moc 03-storage-ready ==="
  echo '--- nodes'; kubectl get nodes -o wide
  echo '--- node creationTimestamp'
  kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}'
  echo '--- storageclass'; kubectl get storageclass
  echo '--- ingressclass'; kubectl get ingressclass
  echo '--- namespaces'; kubectl get namespaces
  echo '--- kube-proxy mode'
  kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -E '^mode:'
} > ~/lab-evidence/8a/b0-truoc.txt
grep -q 'lab-k8s-worker2' ~/lab-evidence/8a/b0-truoc.txt && echo 'PASS: da ghi ho so moc 03-storage-ready'
```

Repo và công cụ:

```bash
test -d ~/bookstore/.git || git clone ~/bookstore-lab-7b.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-7b' && echo 'PASS: HEAD dung tag lab-7b'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
for T in helm curl openssl git; do command -v "$T" >/dev/null && echo "PASS: co $T"; done
```

Repo trên đĩa sau một lần restore snapshot là repo **lúc chụp mốc** (tag `lab-6a`), không phải
`lab-7b`. Gặp trường hợp đó: `mv ~/bookstore ~/bookstore-cu`, chép `bookstore-lab-7b.bundle` từ máy
host lên master theo [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm), clone
lại, chạy lại khối trên rồi xóa `~/bookstore-cu`.

Dựng lại sprint 11 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/8a/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/8a/b0-smoke-lab-7b.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/8a/b0-smoke-lab-7b.txt
```

Ghi biến dùng chung. **Không giá trị nào gõ tay** — tất cả đọc từ cluster sắp bị phá, để cluster dựng
lại có cùng danh tính mạng với nó:

```bash
EV=~/lab-evidence/8a
MASTER=lab-k8s-master; W1=lab-k8s-worker1; W2=lab-k8s-worker2
ip_of() { kubectl get node "$1" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'; }
MASTER_IP="$(ip_of "$MASTER")"; W1_IP="$(ip_of "$W1")"; W2_IP="$(ip_of "$W2")"
CC="$(kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}')"
CP_ENDPOINT="$(echo "$CC" | awk '$1=="controlPlaneEndpoint:"{print $2}')"
API_PORT="${CP_ENDPOINT##*:}"
CRI_SOCKET="$(sudo awk '$1=="containerRuntimeEndpoint:"{print $2}' /var/lib/kubelet/instance-config.yaml 2>/dev/null)"
[ -n "$CRI_SOCKET" ] || CRI_SOCKET="$(sudo grep -o -- '--container-runtime-endpoint=[^" ]*' \
  /var/lib/kubelet/kubeadm-flags.env | cut -d= -f2)"
[ -n "$CRI_SOCKET" ] || CRI_SOCKET="$(kubectl get node "$MASTER" \
  -o jsonpath='{.metadata.annotations.kubeadm\.alpha\.kubernetes\.io/cri-socket}')"
K8S_VER="$(kubectl get --raw /version | grep -o '"gitVersion":"[^"]*"' | cut -d'"' -f4)"
POD_CIDR="$(sudo grep -oE -- '--cluster-cidr=[^[:space:]]+' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml | cut -d= -f2)"
SVC_CIDR="$(sudo grep -oE -- '--service-cluster-ip-range=[^[:space:]]+' \
  /etc/kubernetes/manifests/kube-apiserver.yaml | cut -d= -f2)"
NODE_CT="$(kubectl get node "$MASTER" -o jsonpath='{.metadata.creationTimestamp}')"
HTTPS_NP="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}')"

cat > "$EV/env.sh" <<EOF
# Lab 8a — bien dung chung. Nap lai sau moi lan SSH: . ~/lab-evidence/8a/env.sh
MASTER='$MASTER'
W1='$W1'
W2='$W2'
NODES='$MASTER $W1 $W2'
MASTER_IP='$MASTER_IP'
W1_IP='$W1_IP'
W2_IP='$W2_IP'
CP_ENDPOINT='$CP_ENDPOINT'
API_PORT='$API_PORT'
CRI_SOCKET='$CRI_SOCKET'
K8S_VER='$K8S_VER'
POD_CIDR='$POD_CIDR'
SVC_CIDR='$SVC_CIDR'
NODE_CT='$NODE_CT'
HTTPS_NP='$HTTPS_NP'
NS=bookstore
REPO=\$HOME/bookstore
WK=\$HOME/lab-work/8a
EV=\$HOME/lab-evidence/8a
# Chay lenh tren mot node: master chay tai cho, worker qua ssh.
on() { local n="\$1"; shift; if [ "\$n" = "\$MASTER" ]; then bash -c "\$*"; else ssh "\$n" "\$*"; fi; }
# Ghi moc thoi gian cho RTO.
moc() { echo "\$1 \$(date +%s) \$(date -Is)" >> "\$EV/moc-thoi-gian.txt"; }
EOF
chmod 600 "$EV/env.sh"
. "$EV/env.sh"

BAD=0
for V in MASTER_IP W1_IP W2_IP CP_ENDPOINT API_PORT CRI_SOCKET K8S_VER POD_CIDR SVC_CIDR NODE_CT HTTPS_NP; do
  [ -n "${!V}" ] || { echo "FAIL: $V rong"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: doc du gia tri tu cluster dang chay'
[ "$(kubeadm version -o short)" = "$K8S_VER" ] && echo "PASS: kubeadm tren master = API server = $K8S_VER"
grep -E '^(K8S_VER|POD_CIDR|SVC_CIDR)=' "$EV/env.sh"
```

Mở [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và đối chiếu ba dòng cuối với các
dòng *Kubernetes control plane*, *Pod CIDR*, *Service CIDR*. Lệch nhau nghĩa là cluster đã rời
baseline: dừng và restore — dựng lại bằng giá trị lệch sẽ nhân sai lệch lên.

`CRI_SOCKET` có ba nguồn theo thứ tự ưu tiên. Ở bản Kubernetes của baseline, kubeadm ghi CRI socket của
từng node vào `/var/lib/kubelet/instance-config.yaml` — đúng đường dẫn bài 04 nói — thay cho annotation
trên Node object mà các bản cũ dùng. B2 hỏi lại chuyện này.

**PASS:** gate A5.5 và ba lệnh riêng của mốc đạt; dòng `PASS: da ghi ho so moc 03-storage-ready`; bốn
dòng `PASS:` về repo và cluster; đủ bốn dòng `PASS: co ...`; `deploy exit=0`, `smoke exit=0` và smoke
không có dòng `FAIL:`; hai dòng `PASS:` của khối biến; ba giá trị khớp bảng A1.3.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 11 **chưa đủ** — có thứ chỉ tồn tại trên
cluster cũ. Sửa repo ở tag `lab-7b` trước ([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)):
sprint này dựng lại cluster **từ repo**, một repo thiếu sẽ không dựng ra được hệ thống.

## B1. [BS-8A-01] Điều tra: cluster kubeadm đang chạy gồm những gì

**Loại:** Điều tra — chỉ đọc, trừ một bootstrap token.

**Bối cảnh:** cluster được dựng ở Lab 00 bằng cách chép lệnh. Ngoài các lệnh đó, không ai ghi lại ba
máy phải thỏa điều kiện gì, và kubeadm đã để lại những gì trên đĩa và trong API. Trước khi phá, đội
phải biết chính xác điều đó — cluster dựng lại phải giống cluster này ở mọi điểm quan trọng, và mục
đầu tiên của quy trình dựng lại là danh sách kiểm tra cho một máy sắp thành node.

**Yêu cầu:**

1. Chứng minh ba node thỏa điều kiện của bài 01 bằng giá trị đọc từ máy: hostname, MAC, `product_uuid`
   không trùng nhau; control plane đang lắng nghe đủ cổng của năm tiến trình, worker tới được API;
   swap tắt cả lúc này lẫn sau reboot; cgroup driver của kubelet và containerd khớp nhau; ba gói
   Kubernetes được giữ không cho nâng cấp; `kubeadm`, kubelet và API server cùng một phiên bản.
2. Giải phẫu những gì kubeadm để lại trên control plane: năm kubeconfig, bốn manifest static Pod, ba
   CA, không certificate nào hết hạn; nhóm (`O`) trong certificate của `admin.conf` và
   `super-admin.conf`; mirror Pod của `kube-apiserver` thuộc về ai và nguồn của nó là gì; taint của
   control plane, và không worker nào tự gắn label qua cờ của kubelet.
3. Tách ba thành phần của lệnh `kubeadm join` và **tự tính lại** hash CA từ `pki/ca.crt` bằng
   `openssl`; kết quả phải trùng hash kubeadm in ra.
4. Tạo `platform/README.md` với mục *1. Điều kiện của node*: bảng kiểm tra (lệnh và điều kiện PASS)
   chạy trên một máy trước khi `kubeadm init` hoặc `kubeadm join`. Mục này sẽ được dùng lại thật ở B6.

**Ràng buộc:** không sửa file nào dưới `/etc/kubernetes`, `/var/lib/kubelet`, `/etc/containerd`.
Không cài `nc` — dùng `/dev/tcp` của bash. Lệnh join đầy đủ chỉ nằm ở `~/lab-work/8a/`; bằng chứng
chỉ ghi endpoint và hash công khai. Chỉ đọc trường `Subject` của certificate, không đọc private key.

**Đọc lại:**

- Bài [01](../01-install-kubeadm-vi.md):
  [*Trước khi bạn bắt đầu*](../01-install-kubeadm-vi.md#before-you-begin),
  [*Xác minh địa chỉ MAC và product_uuid*](../01-install-kubeadm-vi.md#verify-mac-address),
  [*Kiểm tra các port cần thiết*](../01-install-kubeadm-vi.md#check-required-ports),
  [*Cấu hình swap*](../01-install-kubeadm-vi.md#swap-configuration),
  [*Cài đặt kubeadm, kubelet và kubectl*](../01-install-kubeadm-vi.md#cài-đặt-kubeadm-kubelet-và-kubectl-installing-kubeadm-kubelet-and-kubectl),
  [*Cấu hình cgroup driver*](../01-install-kubeadm-vi.md#cấu-hình-cgroup-driver-configuring-a-cgroup-driver).
- Bài [02](../02-create-cluster-kubeadm-vi.md):
  [*Khởi tạo control-plane node*](../02-create-cluster-kubeadm-vi.md#khởi-tạo-control-plane-node-initializing-your-control-plane-node)
  — cả khung cảnh báo về `admin.conf` và `super-admin.conf` ở mục *Thông tin thêm*;
  [*Các label node được quản lý*](../02-create-cluster-kubeadm-vi.md#các-label-node-được-quản-lý-managed-node-labels),
  [*Cách ly control plane node*](../02-create-cluster-kubeadm-vi.md#cách-ly-control-plane-node-control-plane-node-isolation),
  [*Chính sách chênh lệch phiên bản*](../02-create-cluster-kubeadm-vi.md#version-skew-policy).
- [Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) — phần static Pod và mirror Pod.

**Bàn giao:** `platform/README.md` (mục 1); `docs/adr/NNNN-node-va-cluster-kubeadm.md` (`NNNN` là số
tiếp theo trong `docs/adr/`); `~/lab-evidence/8a/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` hostname, `product_uuid`, MAC duy nhất trên ba node.
- `PASS:` control plane có `kube-apiserver`, `etcd`, `kubelet`, `kube-controller-manager`,
  `kube-scheduler` đang LISTEN; mỗi worker có `kubelet`, `kube-proxy` LISTEN và tới được API.
- `PASS:` swap tắt và `/etc/fstab` không còn dòng swap trên ba node; kubelet và containerd cùng dùng
  `systemd` trên ba node.
- `PASS:` ba gói được giữ trên ba node; `kubeadm`, kubelet, API server cùng một phiên bản.
- `PASS:` đủ năm kubeconfig, bốn manifest, ba CA; không certificate nào hết hạn.
- `PASS:` `admin.conf` mang `kubeadm:cluster-admins`, `super-admin.conf` mang `system:masters`.
- `PASS:` mirror Pod `kube-apiserver` thuộc về Node, nguồn là file.
- `PASS:` control plane mang taint mặc định; worker không taint, không tự gắn label.
- `PASS:` hash tự tính bằng `openssl` trùng hash trong lệnh join; endpoint là `controlPlaneEndpoint`.
- `PASS:` `platform/README.md` có mục điều kiện node; có ADR.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-node-va-cluster-kubeadm.md`:

1. Đội muốn thêm node bằng cách clone VM từ một worker rồi đổi hostname và IP. Trong ba định danh bài
   01 yêu cầu, cái nào **không** sửa được từ bên trong Ubuntu? Kubernetes cần chúng để làm gì?
2. `swapoff -a` rồi `kubeadm init` thành công; reboot xong kubelet không lên. Hành vi mặc định nào
   gây ra, và hai lối thoát của bài 01 là gì? Chọn lối thứ hai thì workload có tự dùng swap không?
3. `admin.conf` và `super-admin.conf` khác nhau ở đâu về đường đi của quyền? Vì sao repo dự án
   **không bao giờ** chứa file nào trong hai file đó, và `super-admin.conf` nên nằm ở đâu?
4. Trong lệnh join, token và hash: cái nào là bí mật, mỗi cái xác thực **chiều nào**? Ai đó có token
   nhưng dùng hash sai thì lệnh hỏng ở phía nào?
5. Control plane chạy minor N. Được để kubelet của một worker ở N+1 không? Ở N−1? Quy tắc bất đối xứng
   ở chỗ nào, và vì sao binary `kubeadm` dùng để join phải khớp binary đã `init`?

<details>
<summary>Gợi ý</summary>

- `product_uuid` ở `/sys/class/dmi/id/product_uuid`; MAC của interface mang default route ở
  `/sys/class/net/<iface>/address`. Hàm `on` trong `env.sh` chạy cùng một lệnh trên master hoặc qua
  `ssh`.
- `sudo ss -lntp` cho biết **tiến trình nào** giữ cổng nào. Tên tiến trình bị cắt ở 15 ký tự —
  `kube-controller` là `kube-controller-manager`.
- Đường thật tới API từ worker: `timeout 3 bash -c 'echo > /dev/tcp/<host>/<port>'`.
- `apt-mark showhold`; `kubeadm version -o short`; `.status.nodeInfo.kubeletVersion` của Node.
- Subject của client certificate trong kubeconfig: lấy trường `client-certificate-data`, `base64 -d`,
  rồi `openssl x509 -noout -subject`.
- Hash CA: `openssl x509 -pubkey -noout` → `openssl pkey -pubin -outform der` →
  `openssl dgst -sha256 -hex`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-evidence/8a/env.sh
cd "$REPO"

# 1. Định danh
for n in $NODES; do
  IF="$(on "$n" "ip route | awk '/^default/ {print \$5; exit}'")"
  echo "$n $(on "$n" 'hostnamectl --static') $(on "$n" 'sudo cat /sys/class/dmi/id/product_uuid') $(on "$n" "cat /sys/class/net/$IF/address")"
done > "$EV/b1-dinh-danh.txt"
cat "$EV/b1-dinh-danh.txt"
BAD=0
for f in 2 3 4; do [ "$(awk -v f="$f" '{print $f}' "$EV/b1-dinh-danh.txt" | sort -u | wc -l)" -eq 3 ] || BAD=1; done
[ "$BAD" -eq 0 ] && echo 'PASS: hostname, product_uuid, MAC duy nhat tren ba node'

# 2. Cổng và đường tới API
sudo ss -lntp > "$EV/b1-cong-master.txt"
BAD=0
for p in kube-apiserver etcd kubelet kube-controller kube-scheduler; do
  grep -q "\"$p" "$EV/b1-cong-master.txt" || { echo "FAIL: khong thay $p LISTEN"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: control plane LISTEN du nam tien trinh'
for n in "$W1" "$W2"; do
  ssh "$n" 'sudo ss -lntp' > "$EV/b1-cong-$n.txt"
  { grep -q '"kubelet' "$EV/b1-cong-$n.txt" && grep -q '"kube-proxy' "$EV/b1-cong-$n.txt" \
    && ssh "$n" "timeout 3 bash -c 'echo > /dev/tcp/${CP_ENDPOINT%%:*}/$API_PORT'"; } \
    && echo "PASS: $n co kubelet, kube-proxy LISTEN va toi duoc API $CP_ENDPOINT"
done

# 3. Swap và cgroup driver
BAD=0
for n in $NODES; do
  [ -z "$(on "$n" 'swapon --show')" ] || { echo "FAIL: $n dang bat swap"; BAD=1; }
  on "$n" 'grep -qE "^[^#].*[[:space:]]swap[[:space:]]" /etc/fstab' && { echo "FAIL: $n con dong swap trong fstab"; BAD=1; }
  KD="$(on "$n" "sudo awk '\$1==\"cgroupDriver:\"{print \$2}' /var/lib/kubelet/config.yaml")"
  on "$n" "sudo grep -Eq 'SystemdCgroup *= *true' /etc/containerd/config.toml" || { echo "FAIL: $n containerd khong SystemdCgroup"; BAD=1; }
  [ "$KD" = 'systemd' ] || { echo "FAIL: $n kubelet cgroupDriver=$KD"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: swap tat ca luc nay lan sau reboot; kubelet va containerd cung systemd'

# 4. Ba gói và phiên bản
BAD=0
for n in $NODES; do
  H="$(on "$n" 'apt-mark showhold')"
  for p in kubeadm kubelet kubectl; do echo "$H" | grep -qx "$p" || { echo "FAIL: $n chua hold $p"; BAD=1; }; done
  [ "$(on "$n" 'kubeadm version -o short')" = "$K8S_VER" ] || { echo "FAIL: kubeadm tren $n lech"; BAD=1; }
done
KV="$(kubectl get nodes -o jsonpath='{range .items[*]}{.status.nodeInfo.kubeletVersion}{"\n"}{end}' | sort -u)"
[ "$KV" = "$K8S_VER" ] || { echo "FAIL: kubelet $KV khac API server $K8S_VER"; BAD=1; }
[ "$BAD" -eq 0 ] && echo "PASS: ba goi duoc hold; kubeadm, kubelet, API server cung $K8S_VER"

# 5. Giải phẫu /etc/kubernetes
BAD=0
for f in admin.conf super-admin.conf controller-manager.conf scheduler.conf kubelet.conf \
         manifests/etcd.yaml manifests/kube-apiserver.yaml manifests/kube-controller-manager.yaml \
         manifests/kube-scheduler.yaml pki/ca.crt pki/front-proxy-ca.crt pki/etcd/ca.crt; do
  sudo test -f "/etc/kubernetes/$f" || { echo "FAIL: thieu /etc/kubernetes/$f"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du nam kubeconfig, bon manifest static Pod, ba CA'
sudo kubeadm certs check-expiration > "$EV/b1-certs.txt" 2>&1
grep -qi 'invalid\|expired' "$EV/b1-certs.txt" && echo 'FAIL: co certificate het han' \
  || echo 'PASS: khong certificate nao het han'
for f in admin super-admin; do
  printf '%s: ' "$f"
  sudo awk '/client-certificate-data/{print $2}' "/etc/kubernetes/$f.conf" | base64 -d | openssl x509 -noout -subject
done > "$EV/b1-subject.txt"
{ grep -q '^admin:.*kubeadm:cluster-admins' "$EV/b1-subject.txt" && \
  grep -q '^super-admin:.*system:masters' "$EV/b1-subject.txt"; } \
  && echo 'PASS: admin.conf mang kubeadm:cluster-admins, super-admin.conf mang system:masters'

# 6. Mirror Pod, taint, label
P="kube-apiserver-$MASTER"
{ [ "$(kubectl -n kube-system get pod "$P" -o jsonpath='{.metadata.ownerReferences[0].kind}')" = 'Node' ] && \
  [ "$(kubectl -n kube-system get pod "$P" -o jsonpath='{.metadata.annotations.kubernetes\.io/config\.source}')" = 'file' ]; } \
  && echo 'PASS: mirror Pod kube-apiserver thuoc Node, nguon la file tren dia'
case "$(kubectl get node "$MASTER" -o jsonpath='{.spec.taints[*].key}')" in
  *node-role.kubernetes.io/control-plane*) echo 'PASS: control plane mang taint mac dinh' ;;
  *) echo 'FAIL: control plane mat taint mac dinh' ;;
esac
BAD=0
for n in "$W1" "$W2"; do
  [ -z "$(kubectl get node "$n" -o jsonpath='{.spec.taints[*].key}')" ] || BAD=1
  ssh "$n" 'sudo grep -q -- "--node-labels" /var/lib/kubelet/kubeadm-flags.env' && BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: worker khong taint, khong tu gan label qua --node-labels'

# 7. Ba thành phần của lệnh join
sudo kubeadm token create --print-join-command > "$WK/b1-join.txt"; chmod 600 "$WK/b1-join.txt"
J_EP="$(awk '{print $3}' "$WK/b1-join.txt")"
J_HASH="$(tr ' ' '\n' < "$WK/b1-join.txt" | grep '^sha256:' | head -1)"
C_HASH="sha256:$(sudo openssl x509 -pubkey -noout -in /etc/kubernetes/pki/ca.crt \
  | openssl pkey -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | awk '{print $NF}')"
printf 'endpoint=%s\nca-cert-hash=%s\n' "$J_EP" "$J_HASH" > "$EV/b1-join-cong-khai.txt"
[ "$J_HASH" = "$C_HASH" ] && echo 'PASS: hash tu tinh tu pki/ca.crt trung hash trong lenh join'
[ "$J_EP" = "$CP_ENDPOINT" ] && echo 'PASS: endpoint trong lenh join la controlPlaneEndpoint'

# 8. Bàn giao
BAD=0
for K in product_uuid swapon fstab SystemdCgroup apt-mark; do
  grep -q "$K" platform/README.md 2>/dev/null || { echo "FAIL: platform/README.md chua co $K"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: platform/README.md co muc dieu kien cua node'
ls docs/adr/*-node-va-cluster-kubeadm.md >/dev/null 2>&1 && echo 'PASS: co ADR node-va-cluster-kubeadm'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Các lệnh đọc nằm trong script nghiệm thu; phần bạn phải viết là mục 1 của `platform/README.md`:

````bash
cd ~/bookstore
cat > platform/README.md <<'EOF'
# Nền cluster của Bookstore

Quy trình dựng lại cluster lab và hạ tầng nền từ một bộ ba VM đã cài gói theo Lab 00 mục A4
(Ubuntu, containerd, kubeadm, kubelet, kubectl). Phiên bản nằm ở `platform/versions.env`.
Viết và diễn tập ở Lab 8a. Mọi lệnh chạy trên `lab-k8s-master` trừ khi ghi khác.

## 1. Điều kiện của node

Chạy trên **từng** máy trước khi `kubeadm init` hoặc `kubeadm join` (bài 01). Mục này chỉ **kiểm**;
cách cài nằm ở Lab 00 mục A4.

| Kiểm | Lệnh | PASS |
| --- | --- | --- |
| Định danh duy nhất | `hostnamectl --static`; `sudo cat /sys/class/dmi/id/product_uuid`; `ip -br link` | cả ba khác mọi node khác; `product_uuid`, MAC do hypervisor sinh |
| Swap | `swapon --show`; `grep -E '^[^#].*[[:space:]]swap[[:space:]]' /etc/fstab` | cả hai không in gì |
| cgroup driver | `sudo grep SystemdCgroup /etc/containerd/config.toml` | `SystemdCgroup = true`; kubelet nhận `cgroupDriver: systemd` từ `kubeadm/kubeadm-config.yaml` |
| Gói | `apt-mark showhold`; `kubeadm version -o short` | hold `kubeadm`, `kubelet`, `kubectl`; cùng phiên bản trên mọi node, bằng `KUBERNETES_VERSION` |
| Module, sysctl | `lsmod`; `sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward` | có `overlay`, `br_netfilter`; ba giá trị bằng `1` |
| Runtime | `systemctl is-active containerd`; `sudo crictl version` | `active`; `RuntimeApiVersion: v1` |
| Đường tới API (worker) | `timeout 3 bash -c 'echo > /dev/tcp/lab-k8s-master/6443'` | thoát `0` |

Sau mỗi lần reboot chạy lại bảng này: `swapoff -a` chỉ có tác dụng tới lần khởi động kế, chỉ
`/etc/fstab` sạch mới giữ swap tắt.
EOF
````

Ý chính cho ADR:

1. `product_uuid` và MAC do hypervisor sinh — sửa ở VMware, không sửa được trong Ubuntu (Lab 00 A2.2).
   Bài 01: Kubernetes dùng chúng để định danh node; trùng thì cài đặt **có thể thất bại**.
2. Kubelet mặc định **không khởi động** khi thấy swap. Hai lối: tắt swap, hoặc `failSwapOn: false`.
   Lối thứ hai **không** cho workload dùng swap — còn phải đổi `swapBehavior` khác `NoSwap`. Baseline
   chọn lối thứ nhất, và chỉ dòng swap đã comment trong `/etc/fstab` giữ được nó qua reboot.
3. `admin.conf` mang `O = kubeadm:cluster-admins` — quyền đi qua ClusterRoleBinding tới
   `cluster-admin`, tức qua RBAC. `super-admin.conf` mang `O = system:masters` — nhóm break-glass bỏ
   qua lớp phân quyền. Cả hai là bí mật (bài 02); repo bị bundle và chép đi khắp nơi nên không bao
   giờ chứa chúng. `super-admin.conf` nên ở nơi lưu trữ an toàn ngoài máy.
4. Token là bí mật: chứng minh node mới được phép join — ai có nó đều thêm được node đã xác thực.
   Hash CA là công khai: chiều ngược lại, node mới tin rằng nó đang nói chuyện với đúng cluster. Hash
   sai thì **phía node mới** từ chối, vì CA nó nhận về không khớp vân tay.
5. N+1: không — kubelet không bao giờ được cao hơn API server. N−1: được, bài 01 cho lệch một minor.
   Bài 02 còn có bảng kubeadm ↔ kubeadm: binary chạy `join` phải khớp binary đã `init`.

</details>

## B2. [BS-8A-02] Cấu hình kubeadm vào repo

**Loại:** Vận hành.

**Bối cảnh:** Lab 00 dựng cluster bằng cờ dòng lệnh. Những cờ đó chỉ còn ở hai nơi: lịch sử shell, và
ConfigMap `kubeadm-config` — **của chính cluster sắp mất**. Muốn dựng lại giống hệt thì mọi quyết định
phải nằm trong repo, ở dạng một file `kubeadm init --config` đọc được, kiểm được, và mỗi giá trị trong
đó đọc từ cluster thật.

**Yêu cầu:**

1. `platform/kubeadm/kubeadm-config.yaml` có ba phần: phần riêng của control plane node (địa chỉ quảng
   bá, tên node, CRI socket, thư mục patch), phần chung toàn cluster (phiên bản, endpoint dùng chung,
   dải Pod và dải Service — khai **tường minh**, mỗi dải một giá trị), và cấu hình cơ sở cho mọi
   kubelet (cgroup driver). Mọi giá trị **đọc từ cluster đang chạy**, không gõ từ trí nhớ.
2. Một patch trong `platform/kubeadm/patches/` thêm annotation
   `bookstore.example.com/kubeadm-patch: platform-kubeadm` vào static Pod `kube-scheduler` — và chỉ
   vào Pod đó. Tên file đúng quy ước của bài 03.
3. File qua được `kubeadm config validate`; danh sách image mà kubeadm sẽ dùng mang đúng phiên bản
   đang chạy.
4. Đối chiếu các nguồn cấu hình đang có trên cluster: bốn giá trị toàn cluster trong file của bạn bằng
   giá trị trong ConfigMap `kubeadm-config` và bằng cờ thật của control plane; ConfigMap
   `kubelet-config` khớp `/var/lib/kubelet/config.yaml` của ba node ở `clusterDNS`, `clusterDomain`,
   `cgroupDriver`; biết CRI socket của từng node được ghi ở file nào; `ExecStart` của kubelet có đủ
   bốn biến với `KUBELET_EXTRA_ARGS` đứng cuối; không worker nào còn `bootstrap-kubelet.conf`.
5. Chứng minh cluster single-stack từ ba nguồn độc lập — cờ của apiserver và controller-manager,
   `podCIDRs` của Node, `ipFamilyPolicy` của Service — và ghi `net.ipv6.conf.all.forwarding` của ba node.
6. Mục *2. Control plane* trong `platform/README.md`: validate, kéo image, preflight, `init --config`,
   kubeconfig cho user thường (link Lab 00 A5.1).

**Ràng buộc:** không chạy `kubeadm init` hay `reset`. Không thêm `extraArgs` — cluster hiện tại không có,
thêm vào là dựng ra một cluster **khác** cluster cũ. `patches.directory` là đường dẫn tuyệt đối tới thư
mục trong repo. Không đặt token hay certificate key vào file.

**Đọc lại:**

- Bài [03](../03-control-plane-flags-vi.md):
  [*Tùy chỉnh control plane với các flag trong `ClusterConfiguration`*](../03-control-plane-flags-vi.md#tùy-chỉnh-control-plane-với-các-flag-trong-clusterconfiguration-customizing-the-control-plane-with-flags-in-clusterconfiguration)
  — hai ghi chú về phạm vi toàn cục và key trùng;
  [*Tùy chỉnh với các patch*](../03-control-plane-flags-vi.md#patches),
  [*Tùy chỉnh kubelet*](../03-control-plane-flags-vi.md#kubelet),
  [*Tùy chỉnh kube-proxy*](../03-control-plane-flags-vi.md#tùy-chỉnh-kube-proxy-customizing-kube-proxy).
- Bài [04](../04-kubelet-integration-vi.md):
  [*Lan truyền cấu hình cấp cluster*](../04-kubelet-integration-vi.md#propagating-cluster-level-configuration-to-each-kubelet),
  [*Cung cấp chi tiết cấu hình riêng cho từng máy*](../04-kubelet-integration-vi.md#providing-instance-specific-configuration-details),
  [*Quy trình khi dùng `kubeadm init`*](../04-kubelet-integration-vi.md#quy-trình-khi-dùng-kubeadm-init-workflow-when-using-kubeadm-init),
  [*File drop-in của kubelet cho systemd*](../04-kubelet-integration-vi.md#the-kubelet-drop-in-file-for-systemd).
- Bài [05](../05-dual-stack-support-vi.md):
  [*Bật chuyển tiếp gói tin IPv6*](../05-dual-stack-support-vi.md#prerequisite-ipv6-forwarding),
  [*Tạo một cluster dual-stack*](../05-dual-stack-support-vi.md#tạo-một-cluster-dual-stack-create-a-dual-stack-cluster),
  [*Tạo một cluster single-stack*](../05-dual-stack-support-vi.md#tạo-một-cluster-single-stack-create-a-single-stack-cluster).
- Bài [02](../02-create-cluster-kubeadm-vi.md),
  [*Những lưu ý về apiserver-advertise-address và ControlPlaneEndpoint*](../02-create-cluster-kubeadm-vi.md#những-lưu-ý-về-apiserver-advertise-address-và-controlplaneendpoint-considerations-about-apiserver-advertise-address-and-controlplaneendpoint).
- Tham chiếu [kubeadm Configuration (v1beta4)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)
  — tên trường của từng `kind`.

**Bàn giao:** `platform/kubeadm/kubeadm-config.yaml`; `platform/kubeadm/patches/`; `platform/README.md`
(mục 2); `docs/adr/NNNN-cau-hinh-kubeadm.md`; `~/lab-evidence/8a/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` file có đúng ba `kind` `InitConfiguration`, `ClusterConfiguration`, `KubeletConfiguration`;
  `kubeadm config validate` thoát `0`.
- `PASS:` `kubernetesVersion`, `controlPlaneEndpoint`, `podSubnet`, `serviceSubnet` trong file bằng
  ConfigMap `kubeadm-config` và bằng cờ thật của control plane.
- `PASS:` `advertiseAddress`, tên node, `criSocket` bằng giá trị đọc từ node; `cgroupDriver: systemd`.
- `PASS:` danh sách image mang đúng phiên bản đang chạy.
- `PASS:` `patches.directory` là đường dẫn tuyệt đối tới thư mục patch trong repo; mọi file trong đó
  nhắm `kube-scheduler`, đúng quy ước tên, và mang annotation yêu cầu.
- `PASS:` ConfigMap `kubelet-config` khớp file trên ba node; CRI socket của từng node tìm được trong
  `instance-config.yaml` hoặc `kubeadm-flags.env`.
- `PASS:` `ExecStart` có đủ bốn biến, `KUBELET_EXTRA_ARGS` đứng cuối; worker không còn
  `bootstrap-kubelet.conf`.
- `PASS:` single-stack ở cả ba nguồn; đã ghi `net.ipv6.conf.all.forwarding` của ba node.
- `PASS:` `platform/README.md` có mục control plane trỏ tới file cấu hình; có ADR.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-cau-hinh-kubeadm.md`:

1. Với **từng** giá trị trong file của bạn, nói vì sao nó nằm ở `kind` đó mà không thể nằm ở `kind`
   khác. Dẫn câu của bài 03 về phạm vi của `ClusterConfiguration`.
2. Giả sử sau này có control plane thứ hai cần `kube-apiserver` chạy một cờ khác với node đầu.
   `extraArgs` có làm được không? Cơ chế nào làm được, nó áp vào lúc nào, và cái giá phải trả khi
   `kubeadm upgrade` là gì?
3. `controlPlaneEndpoint` là tên DNS, `advertiseAddress` là IP. Cái nào là **quyết định một chiều**, và
   vì sao giữ đúng tên mà Lab 00 đã chọn thay vì đổi sang IP cho gọn?
4. Muốn cluster dựng lại là dual-stack thì phải đổi những trường nào trong file, thêm điều kiện gì trên
   mọi node? Vì sao không đổi được trên cluster đang chạy bằng `kubeadm upgrade`?
5. ConfigMap `kubeadm-config` đã chứa `ClusterConfiguration`. Vì sao vẫn phải chép nó vào repo?
6. Trên cluster của bạn, CRI socket của mỗi node được ghi ở file nào? Nó thuộc mẫu cấu hình nào của
   bài 04, và vì sao không đặt nó trong `KubeletConfiguration` cơ sở?

<details>
<summary>Gợi ý</summary>

- `kubeadm config print init-defaults` cho khung của ba `kind`; xóa hết những gì không cần, giữ tên
  trường.
- `ClusterConfiguration` đang chạy:
  `kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}'`. Các biến
  trong `env.sh` đã có đủ giá trị.
- Heredoc **không** đặt `EOF` trong nháy đơn thì Bash thay biến lúc ghi file — file nhận hằng số.
- Patch kiểu `merge` chỉ cần phần `metadata.annotations` của Pod.
- `kubectl -n kube-system get cm kubelet-config -o jsonpath='{.data.kubelet}'`;
  `systemctl cat kubelet`.
- Họ địa chỉ của Service: `.spec.ipFamilies`, `.spec.ipFamilyPolicy` của Service `kube-dns`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-evidence/8a/env.sh
cd "$REPO"
CFG=platform/kubeadm/kubeadm-config.yaml
f() { awk -v k="$1" '$1==k":"{print $2; exit}' "$CFG" | tr -d "\"'"; }

[ "$(awk '$1=="kind:"{print $2}' "$CFG" | sort | tr '\n' ' ')" = 'ClusterConfiguration InitConfiguration KubeletConfiguration ' ] \
  && echo 'PASS: du ba kind'
sudo kubeadm config validate --config "$CFG" > "$EV/b2-validate.txt" 2>&1 && echo 'PASS: kubeadm config validate thoat 0'

CM="$(kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}')"
cm() { echo "$CM" | awk -v k="$1" '$1==k":"{print $2; exit}'; }
BAD=0
for X in "kubernetesVersion $K8S_VER" "controlPlaneEndpoint $CP_ENDPOINT" "podSubnet $POD_CIDR" "serviceSubnet $SVC_CIDR"; do
  set -- $X
  { [ "$(f "$1")" = "$2" ] && [ "$(cm "$1")" = "$2" ]; } || { echo "FAIL: $1 file=$(f "$1") configmap=$(cm "$1") that=$2"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: bon gia tri toan cluster khop ConfigMap kubeadm-config va co that'
{ [ "$(f advertiseAddress)" = "$MASTER_IP" ] && [ "$(f name)" = "$MASTER" ] && \
  [ "$(f criSocket)" = "$CRI_SOCKET" ] && [ "$(f cgroupDriver)" = 'systemd' ]; } \
  && echo 'PASS: phan rieng cua node va cgroupDriver dung gia tri doc tu cluster'
sudo kubeadm config images list --config "$CFG" > "$EV/b2-images.txt" 2>&1
grep -q "kube-apiserver:$K8S_VER" "$EV/b2-images.txt" && echo "PASS: kubeadm se dung image $K8S_VER"

PD="$(f directory)"
N="$(ls "$PD" 2>/dev/null | wc -l)"
T="$(ls "$PD" 2>/dev/null | grep -cE '^kube-scheduler[^+.]*(\+(strategic|merge|json))?\.(yaml|json)$')"
{ [ "$PD" = "$REPO/platform/kubeadm/patches" ] && [ "$N" -ge 1 ] && [ "$T" -eq "$N" ] && \
  grep -q 'bookstore.example.com/kubeadm-patch' "$PD"/*; } \
  && echo 'PASS: patch dung quy uoc, duong dan tuyet doi, chi nham kube-scheduler'

KC="$(kubectl -n kube-system get cm kubelet-config -o jsonpath='{.data.kubelet}')"
kv() { awk -v k="$1" '$1==k":"{print $2; exit}'; }
dns() { grep -A1 '^clusterDNS:' | grep -oE '[0-9.]+' | head -1; }
CM_SIG="$(echo "$KC" | dns) $(echo "$KC" | kv clusterDomain) $(echo "$KC" | kv cgroupDriver)"
BAD=0
for n in $NODES; do
  NF="$(on "$n" 'sudo cat /var/lib/kubelet/config.yaml')"
  SIG="$(echo "$NF" | dns) $(echo "$NF" | kv clusterDomain) $(echo "$NF" | kv cgroupDriver)"
  echo "$n: $SIG" >> "$EV/b2-kubelet-config.txt"
  [ "$SIG" = "$CM_SIG" ] || { echo "FAIL: $n lech ConfigMap kubelet-config"; BAD=1; }
  W="$(on "$n" 'sudo grep -l containerd.sock /var/lib/kubelet/instance-config.yaml /var/lib/kubelet/kubeadm-flags.env 2>/dev/null' | tr '\n' ' ')"
  echo "$n: CRI socket ghi o $W" >> "$EV/b2-cri-socket.txt"
  [ -n "$W" ] || { echo "FAIL: khong tim thay CRI socket cua $n"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo "PASS: ba node khop ConfigMap kubelet-config ($CM_SIG); biet CRI socket ghi o dau"

systemctl cat kubelet > "$EV/b2-kubelet-unit.txt"
EXEC="$(grep -m1 '^ExecStart=/usr/bin/kubelet' "$EV/b2-kubelet-unit.txt")"
du_bon_bien() {
  for v in KUBELET_KUBECONFIG_ARGS KUBELET_CONFIG_ARGS KUBELET_KUBEADM_ARGS KUBELET_EXTRA_ARGS; do
    case "$EXEC" in *"\$$v"*) ;; *) return 1 ;; esac
  done
}
du_bon_bien && [ "$(echo "$EXEC" | awk '{print $NF}')" = '$KUBELET_EXTRA_ARGS' ] \
  && echo 'PASS: ExecStart du bon bien, KUBELET_EXTRA_ARGS dung cuoi'
BAD=0
for n in "$W1" "$W2"; do
  ssh "$n" 'sudo test ! -e /etc/kubernetes/bootstrap-kubelet.conf && sudo test -f /etc/kubernetes/kubelet.conf' || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: worker khong con bootstrap-kubelet.conf, co kubelet.conf'

MULTI="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.podCIDRs[*]}{"\n"}{end}' | awk 'NF!=1' | wc -l)"
POL="$(kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.ipFamilyPolicy}')"
case "$POD_CIDR$SVC_CIDR" in
  *,*) echo 'FAIL: co dai kep' ;;
  *) [ "$MULTI" -eq 0 ] && [ "$POL" = 'SingleStack' ] && echo 'PASS: single-stack o ca ba nguon' ;;
esac
for n in $NODES; do echo "$n $(on "$n" 'sysctl -n net.ipv6.conf.all.forwarding')"; done > "$EV/b2-ipv6-forwarding.txt"
[ "$(wc -l < "$EV/b2-ipv6-forwarding.txt")" -eq 3 ] && echo 'PASS: da ghi net.ipv6.conf.all.forwarding cua ba node'

grep -q 'kubeadm/kubeadm-config.yaml' platform/README.md && grep -q 'config validate' platform/README.md \
  && echo 'PASS: README co muc control plane'
ls docs/adr/*-cau-hinh-kubeadm.md >/dev/null 2>&1 && echo 'PASS: co ADR cau-hinh-kubeadm'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
. ~/lab-evidence/8a/env.sh
mkdir -p "$REPO/platform/kubeadm/patches"

cat > "$REPO/platform/kubeadm/kubeadm-config.yaml" <<EOF
# Cấu hình kubeadm của cluster lab Bookstore — mọi giá trị đọc từ cluster đang chạy ở Lab 8a.
# Dùng theo platform/README.md mục 2. Không chứa token hay certificate key.
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: $MASTER_IP
  bindPort: $API_PORT
nodeRegistration:
  name: $MASTER
  criSocket: $CRI_SOCKET
patches:
  directory: $REPO/platform/kubeadm/patches
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: $K8S_VER
controlPlaneEndpoint: $CP_ENDPOINT
networking:
  podSubnet: $POD_CIDR
  serviceSubnet: $SVC_CIDR
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF

cat > "$REPO/platform/kubeadm/patches/kube-scheduler0+merge.yaml" <<'EOF'
metadata:
  annotations:
    bookstore.example.com/kubeadm-patch: platform-kubeadm
EOF
cat "$REPO/platform/kubeadm/kubeadm-config.yaml"
```

Mục 2 của `platform/README.md`:

````bash
cat >> ~/bookstore/platform/README.md <<'EOF'

## 2. Control plane

File cấu hình: `kubeadm/kubeadm-config.yaml` (ba kind: InitConfiguration — riêng node này;
ClusterConfiguration — toàn cluster; KubeletConfiguration — cơ sở cho mọi kubelet). Patch trong
`kubeadm/patches/` được áp ngay trước khi kubeadm ghi manifest xuống đĩa; nó **không** tồn tại ở đâu
khác, nên khi `kubeadm upgrade` phải đưa lại qua UpgradeConfiguration.

```bash
CFG=~/bookstore/platform/kubeadm/kubeadm-config.yaml
sudo kubeadm config validate --config "$CFG"
sudo kubeadm config images list --config "$CFG"     # phải mang KUBERNETES_VERSION
sudo kubeadm config images pull --config "$CFG"
sudo kubeadm init phase preflight --config "$CFG"   # không được có [ERROR
sudo kubeadm init --config "$CFG"
```

Không thêm cờ nào ngoài `--config`; không dùng `--ignore-preflight-errors`. Init lỗi thì tra bài 09,
`sudo kubeadm reset -f` rồi mới chạy lại. Xong thì chép kubeconfig cho user thường theo Lab 00 A5.1
(bốn lệnh `mkdir`, `cp`, `chown`, `chmod`).

PASS: `Your Kubernetes control-plane has initialized successfully!`; node control plane mang taint
`node-role.kubernetes.io/control-plane:NoSchedule`; manifest `kube-scheduler.yaml` có annotation
`bookstore.example.com/kubeadm-patch`. CoreDNS `Pending` lúc này là **đúng thiết kế** — chưa có CNI.
EOF
````

Ý chính cho ADR:

1. `InitConfiguration` chỉ cho **node đang init**: địa chỉ quảng bá, tên, CRI socket, thư mục patch.
   `ClusterConfiguration` có **phạm vi toàn cục** — bài 03: mọi flag thêm vào áp cho mọi instance của
   thành phần trên mọi node — nên chứa phiên bản, endpoint, dải mạng. `KubeletConfiguration` là cấu
   hình **cơ sở** cho mọi kubelet; `cgroupDriver` giống nhau trên ba node nên đặt ở đây.
2. Không — `extraArgs` toàn cục và không nhận key trùng. Dùng patch target `kube-apiserver` trong
   `patches.directory` của `JoinConfiguration` của node đó. Patch áp **trước khi manifest ghi xuống
   đĩa**; khi `kubeadm upgrade` phải đưa lại đúng các patch đó qua `UpgradeConfiguration`, nếu không
   tùy chỉnh mất.
3. `controlPlaneEndpoint` — bài 02: chuyển cluster tạo không có nó thành HA không được kubeadm hỗ trợ.
   Cluster dựng lại phải giữ đúng tên để kubeconfig, certificate và lệnh join của cluster mới có cùng
   endpoint, và để Lab 8b còn đường lên HA bằng cách trỏ tên đó sang load balancer.
4. `podSubnet`, `serviceSubnet` thành danh sách hai dải cách dấu phẩy; mỗi node khai `node-ip` hai địa
   chỉ trong `kubeletExtraArgs`; mọi node đặt `net.ipv6.conf.all.forwarding = 1`. Bài 05: `kubeadm
   upgrade` không hỗ trợ đổi cluster CIDR và Service CIDR — chỉ đổi được bằng cách dựng lại.
5. ConfigMap nằm trong etcd của cluster. Mất cluster là mất ConfigMap; repo nằm ngoài cluster, có lịch
   sử, có review. Và ConfigMap không chứa phần riêng của node, cấu hình kubelet hay patch.
6. Ở baseline của lab, CRI socket nằm trong `/var/lib/kubelet/instance-config.yaml` (bài 04, quy trình
   `init` và `join`). Nó thuộc mẫu **cấu hình riêng cho từng máy**: mỗi node có runtime riêng, nên
   không thể là một giá trị chung trong `KubeletConfiguration` cơ sở.

</details>

## B3. [BS-8A-03] Quy trình dựng nền vào `platform/README.md`

**Loại:** Vận hành.

**Bối cảnh:** `03-storage-ready` có ba thành phần nền mà repo ứng dụng dựa vào
([mục 3.3 của file dự án](DU-AN-BOOKSTORE.md#33-hạ-tầng-nền-của-cluster)): Calico — không có thì mọi
NetworkPolicy của sprint 7 hợp lệ mà vô hiệu; Traefik — không có thì Ingress `bookstore` không ai phục
vụ; local-path-provisioner — không có thì PVC của `db` nằm `Pending` và `deploy.sh` treo. Hiện ba thứ
đó chỉ tồn tại trong snapshot và trong ghi chép của Lab 5b, 6a. Dựng lại cluster mà quên một bước —
hay quên một thao tác tay đã làm ngoài manifest — là Bookstore không lên.

**Yêu cầu:**

1. `platform/versions.env` — Lab 5b đã tạo file này với các giá trị của Calico, Helm, Traefik —
   **bổ sung** cho đủ phiên bản Kubernetes, Calico, Helm, chart Traefik, local-path-provisioner
   **bằng giá trị đang chạy** trên cluster, cùng sha256 của hai manifest (Calico, local-path-provisioner)
   tải từ đúng URL theo phiên bản. Giữ tên biến mà Lab 5b đã đặt. File được `source` được bằng Bash.
2. `platform/README.md` ghi tiếp quy trình theo đúng thứ tự: *3. Calico*, *4. Join worker*,
   *5. Helm và Traefik*, *6. local-path-provisioner*, *7. Gate cluster*, *8. Triển khai Bookstore*,
   *9. Nạp bản dump* (B9 hoàn thiện). Mỗi bước có lệnh và điều kiện PASS đọc được.
3. README ghi rõ cách đạt **mọi trạng thái nền** mà Lab 5b, 6a đã tạo, kể cả phần không nằm trong
   manifest gốc: Service của Traefik là NodePort cố định (Lab 5b khai trong
   `platform/traefik/values.yaml`), IngressClass và StorageClass mặc định. Thao tác nào bạn đã làm
   bằng tay thay vì qua file trong `platform/` thì ghi thành lệnh cụ thể.
4. Mọi phép **kiểm** của bước 3, 5, 6, 7 chạy PASS trên cluster hiện tại — cluster hiện tại chính là
   trạng thái đích của quy trình.

**Ràng buộc:** không cài, không gỡ gì ở ticket này. Lấy lệnh đã kiểm chứng từ
[Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) và [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md) (và ghi
chép của bạn nếu đã có trong `platform/`). Phiên bản ghi vào repo của bạn là đúng chỗ; đối chiếu với
[A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00), không đoán. StorageClass
`bookstore-retain` **không** thuộc nền — nó thuộc `k8s/cluster/` của repo ứng dụng.

**Đọc lại:**

- Bài [02](../02-create-cluster-kubeadm-vi.md),
  [*Cài đặt Pod network add-on*](../02-create-cluster-kubeadm-vi.md#pod-network) — thứ tự triển khai,
  một Pod network cho mỗi cluster, CIDR phải khớp YAML của plugin, chỉ join khi CoreDNS `Running`;
  [*Thêm worker node*](../02-create-cluster-kubeadm-vi.md#join-nodes).
- [Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) — phần đổi CNI, cài Helm, cài Traefik bằng chart.
- [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md) — phần cài provisioner và class mặc định.
- [Mục 3.3 của file dự án](DU-AN-BOOKSTORE.md#33-hạ-tầng-nền-của-cluster).

**Bàn giao:** `platform/versions.env`; `platform/README.md` (mục 3–9);
`docs/adr/NNNN-quy-trinh-nen-cluster.md`; `~/lab-evidence/8a/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` `versions.env` có đủ biến; mỗi phiên bản bằng giá trị đang chạy trên cluster.
- `PASS:` hai sha256 trong `versions.env` khớp manifest tải lại từ URL theo phiên bản.
- `PASS:` trong README, các lệnh xuất hiện theo đúng thứ tự: `kubeadm init --config` → manifest Calico →
  `kubeadm token create` → `helm ... install` → manifest local-path-provisioner → `scripts/deploy.sh`.
- `PASS:` README ghi thao tác NodePort và class mặc định.
- `PASS:` trên cluster hiện tại: `calico-node` đủ ba node, IPPool bằng `podSubnet`; IngressClass
  `traefik` là mặc định duy nhất; Service `traefik` kiểu `NodePort`; provisioner Ready; `local-path` là
  StorageClass mặc định duy nhất.
- `PASS:` có ADR.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-quy-trinh-nen-cluster.md`:

1. Vì sao thứ tự là init → CNI → join → Traefik và local-path, không cài cả ba thành phần nền ngay sau
   init? Dẫn bài 02 và taint của control plane.
2. Cài lại Traefik thì Service NodePort được cấp cổng mới. Ai bị ảnh hưởng? Làm sao giữ cổng cố định,
   và bạn có làm điều đó ở sprint này không — vì sao?
3. Tải manifest từ Internet lúc khôi phục, hay lưu sẵn trong repo? sha256 trong `versions.env` bảo vệ
   khỏi điều gì, và **không** bảo vệ khỏi điều gì?
4. Helm nằm trên đĩa master, không trong cluster và không trong repo. Mất cả máy master thì quy trình
   này hỏng ở đâu? Còn thứ gì khác của quy trình nằm ngoài cả cluster lẫn repo?
5. Kể các thao tác tay mà Lab 5b, 6a đã làm ngoài manifest. Vì sao đó là loại cấu hình dễ mất nhất khi
   dựng lại?

<details>
<summary>Gợi ý</summary>

- Phiên bản đang chạy: tag image của container `calico-node` trong DaemonSet `calico-node`
  (`kube-system`); tag image của Deployment `local-path-provisioner` (`local-path-storage`);
  `helm version --short`; cột `CHART` của `helm -n traefik list`.
- URL manifest: Calico ở `https://raw.githubusercontent.com/projectcalico/calico/<phiên bản>/manifests/calico.yaml`;
  local-path-provisioner ở
  `https://raw.githubusercontent.com/rancher/local-path-provisioner/<phiên bản>/deploy/local-path-storage.yaml`.
- `sha256sum -c -` đọc dòng `<sum>  <file>` từ stdin — đúng một lệnh để README tự kiểm manifest.
- Trong file Markdown có khối code, khi ghi bằng heredoc từ Bash: dùng `<<'EOF'` để `$` và backtick
  giữ nguyên.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-evidence/8a/env.sh
cd "$REPO"
tag() { sed 's/.*://'; }
R_CAL="$(kubectl -n kube-system get ds calico-node -o jsonpath='{.spec.template.spec.containers[?(@.name=="calico-node")].image}' | tag)"
R_LPP="$(kubectl -n local-path-storage get deploy local-path-provisioner -o jsonpath='{.spec.template.spec.containers[0].image}' | tag)"
R_HELM="$(helm version --short | sed 's/+.*//')"
R_TRF="$(helm -n traefik list --filter '^traefik$' | awk 'NR==2{print $(NF-1)}' | sed 's/^traefik-//')"
echo "dang chay: k8s=$K8S_VER calico=$R_CAL helm=$R_HELM traefik-chart=$R_TRF local-path=$R_LPP" | tee "$EV/b3-dang-chay.txt"

( . platform/versions.env
  BAD=0
  for X in "KUBERNETES_VERSION $K8S_VER" "CALICO_VER $R_CAL" "HELM_VER $R_HELM" \
           "TRAEFIK_CHART $R_TRF" "LOCAL_PATH_VERSION $R_LPP"; do
    set -- $X
    [ "${!1}" = "$2" ] || { echo "FAIL: $1=${!1} nhung cluster chay $2"; BAD=1; }
  done
  [ "$BAD" -eq 0 ] && echo 'PASS: versions.env bang phien ban dang chay'
  curl -fsSL -o "$WK/calico.yaml" "https://raw.githubusercontent.com/projectcalico/calico/$CALICO_VER/manifests/calico.yaml"
  curl -fsSL -o "$WK/local-path-storage.yaml" \
    "https://raw.githubusercontent.com/rancher/local-path-provisioner/$LOCAL_PATH_VERSION/deploy/local-path-storage.yaml"
  { echo "$CALICO_SHA256  $WK/calico.yaml" | sha256sum -c - && \
    echo "$LOCAL_PATH_SHA256  $WK/local-path-storage.yaml" | sha256sum -c -; } > "$EV/b3-sha256.txt" 2>&1 \
    && echo 'PASS: sha256 trong versions.env khop manifest tai lai' )

L=0; BAD=0
for P in 'kubeadm init .*--config' 'calico\.yaml' 'kubeadm token create' 'helm (upgrade --)?install' \
         'local-path-storage\.yaml' 'scripts/deploy\.sh'; do
  N="$(grep -nE "$P" platform/README.md | head -1 | cut -d: -f1)"
  { [ -n "$N" ] && [ "$N" -gt "$L" ]; } || { echo "FAIL: '$P' thieu hoac sai thu tu"; BAD=1; }
  L="${N:-0}"
done
[ "$BAD" -eq 0 ] && echo 'PASS: README ghi cac buoc theo dung thu tu'
grep -q 'NodePort' platform/README.md && grep -q 'is-default-class' platform/README.md \
  && echo 'PASS: README ghi thao tac NodePort va class mac dinh'

NODES_N="$(kubectl get nodes --no-headers | wc -l)"
{ [ "$(kubectl -n kube-system get ds calico-node -o jsonpath='{.status.numberReady}')" = "$NODES_N" ] && \
  [ "$(kubectl get ippools.crd.projectcalico.org default-ipv4-ippool -o jsonpath='{.spec.cidr}')" = "$POD_CIDR" ]; } \
  && echo 'PASS: calico-node du node, IPPool bang podSubnet'
DEF_IC="$(kubectl get ingressclass -o jsonpath='{range .items[*]}{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}{"\n"}{end}' | grep -c '^true$')"
{ [ "$(kubectl get ingressclass traefik -o jsonpath='{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}')" = 'true' ] && \
  [ "$DEF_IC" -eq 1 ] && [ "$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.type}')" = 'NodePort' ]; } \
  && echo 'PASS: traefik la IngressClass mac dinh duy nhat, Service NodePort'
DEF_SC="$(kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' | grep -c '^true$')"
{ [ "$(kubectl -n local-path-storage get deploy local-path-provisioner -o jsonpath='{.status.readyReplicas}')" -ge 1 ] && \
  [ "$(kubectl get storageclass local-path -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}')" = 'true' ] && \
  [ "$DEF_SC" -eq 1 ]; } && echo 'PASS: provisioner Ready, local-path la StorageClass mac dinh duy nhat'
ls docs/adr/*-quy-trinh-nen-cluster.md >/dev/null 2>&1 && echo 'PASS: co ADR quy-trinh-nen-cluster'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

`platform/versions.env`, sinh từ cluster đang chạy:

```bash
. ~/lab-evidence/8a/env.sh
cd "$REPO"
CAL="$(kubectl -n kube-system get ds calico-node -o jsonpath='{.spec.template.spec.containers[?(@.name=="calico-node")].image}' | sed 's/.*://')"
LPP="$(kubectl -n local-path-storage get deploy local-path-provisioner -o jsonpath='{.spec.template.spec.containers[0].image}' | sed 's/.*://')"
HELMV="$(helm version --short | sed 's/+.*//')"
TRF="$(helm -n traefik list --filter '^traefik$' | awk 'NR==2{print $(NF-1)}' | sed 's/^traefik-//')"
TRF_APP="$(helm -n traefik list --filter '^traefik$' | awk 'NR==2{print $NF}')"
curl -fsSL -o "$WK/calico.yaml" "https://raw.githubusercontent.com/projectcalico/calico/$CAL/manifests/calico.yaml"
curl -fsSL -o "$WK/local-path-storage.yaml" \
  "https://raw.githubusercontent.com/rancher/local-path-provisioner/$LPP/deploy/local-path-storage.yaml"
# Giữ bốn biến Lab 5b đã đặt (CALICO_VER, HELM_VER, TRAEFIK_CHART, TRAEFIK_APP), thêm phần còn thiếu.
cat > platform/versions.env <<EOF
# Phiên bản nền của cluster lab — đọc từ cluster đang chạy ở Lab 8a; nguồn: Lab 00 bảng A1.3, A1.4.
KUBERNETES_VERSION=$K8S_VER
CALICO_VER=$CAL
HELM_VER=$HELMV
TRAEFIK_CHART=$TRF
TRAEFIK_APP=$TRF_APP
LOCAL_PATH_VERSION=$LPP
CALICO_SHA256=$(sha256sum < "$WK/calico.yaml" | awk '{print $1}')
LOCAL_PATH_SHA256=$(sha256sum < "$WK/local-path-storage.yaml" | awk '{print $1}')
EOF
cat platform/versions.env
```

Mục 3–9 của `platform/README.md`:

````bash
cat >> ~/bookstore/platform/README.md <<'EOF'

## 3. Calico

Ngay sau mục 2, **trước** khi join worker. Bài 02: CoreDNS chỉ `Running` khi đã có Pod network, và
chỉ join node khi CoreDNS đã chạy. Manifest để `CALICO_IPV4POOL_CIDR` ở dạng comment, nên Calico tự
lấy dải Pod từ cấu hình kubeadm.

```bash
. ~/bookstore/platform/versions.env
curl -fsSL -o /tmp/calico.yaml \
  "https://raw.githubusercontent.com/projectcalico/calico/$CALICO_VER/manifests/calico.yaml"
echo "$CALICO_SHA256  /tmp/calico.yaml" | sha256sum -c -
kubectl apply -f /tmp/calico.yaml
kubectl -n kube-system rollout status daemonset/calico-node --timeout=600s
kubectl -n kube-system rollout status deployment/coredns --timeout=600s
kubectl get ippools.crd.projectcalico.org default-ipv4-ippool -o jsonpath='{.spec.cidr}{"\n"}'
```

PASS: `sha256sum -c` in `OK`; IPPool bằng `podSubnet` của `kubeadm/kubeadm-config.yaml`; CoreDNS
`Running`; node control plane `Ready`.

## 4. Join worker

Kiểm đường trước, join sau (Lab 00 A5.3). Lệnh join chứa token bí mật: không lưu ngoài `~/`, xóa
ngay khi xong.

```bash
sudo kubeadm token create --print-join-command > ~/join.txt; chmod 600 ~/join.txt
for N in lab-k8s-worker1 lab-k8s-worker2; do
  ssh "$N" 'timeout 3 bash -c "echo > /dev/tcp/lab-k8s-master/6443"' && ssh "$N" "sudo $(cat ~/join.txt)"
done
rm -f ~/join.txt
kubectl wait --for=condition=Ready node --all --timeout=600s
```

PASS: ba node `Ready`; mỗi node một `podCIDR` khác nhau; worker không còn
`/etc/kubernetes/bootstrap-kubelet.conf`.

## 5. Helm và Traefik

Helm nằm ở `/usr/local/bin/helm` trên master (cài ở Lab 5b, sống qua `kubeadm reset`). Mất máy master
thì cài lại theo Lab 5b — tải binary đúng `HELM_VER`, kiểm sha256 trước khi cài.

```bash
. ~/bookstore/platform/versions.env
helm version --short | grep -q "^$HELM_VER" && echo 'helm dung phien ban'
helm repo add traefik https://traefik.github.io/charts     # da co thi helm bao bo qua
helm repo update traefik
helm upgrade --install traefik traefik/traefik --version "$TRAEFIK_CHART" \
  --namespace traefik --create-namespace
kubectl -n traefik rollout status deployment/traefik --timeout=600s
```

Hai thao tác **ngoài chart** mà Lab 5b đã làm — thiếu chúng là cluster khác mốc:

```bash
kubectl -n traefik patch svc traefik -p '{"spec":{"type":"NodePort"}}'
kubectl annotate ingressclass traefik ingressclass.kubernetes.io/is-default-class=true --overwrite
kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}{"\n"}'
```

PASS: Service `traefik` kiểu `NodePort`; IngressClass `traefik` là mặc định duy nhất. NodePort HTTPS
**được cấp lại** mỗi lần cài — ghi giá trị mới và báo người dùng đổi URL trên máy host.

## 6. local-path-provisioner

```bash
. ~/bookstore/platform/versions.env
curl -fsSL -o /tmp/local-path-storage.yaml \
  "https://raw.githubusercontent.com/rancher/local-path-provisioner/$LOCAL_PATH_VERSION/deploy/local-path-storage.yaml"
echo "$LOCAL_PATH_SHA256  /tmp/local-path-storage.yaml" | sha256sum -c -
kubectl apply -f /tmp/local-path-storage.yaml
kubectl -n local-path-storage rollout status deploy/local-path-provisioner --timeout=300s
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

PASS: provisioner Ready; `kubectl get storageclass` có đúng một dòng `(default)` là `local-path`.
StorageClass `bookstore-retain` **không** cài ở đây — `scripts/deploy.sh` áp nó từ `k8s/cluster/`.

## 7. Gate cluster

Gate ngắn của Lab 00 A5.5, cộng: `kubectl get storageclass` (một class mặc định), `kubectl get
ingressclass` (có `traefik`), `kubectl -n kube-system get ds calico-node` (đủ số node).

## 8. Triển khai Bookstore

```bash
cd ~/bookstore && ./scripts/deploy.sh && ./tests/smoke.sh
```

`deploy.sh` tự sinh `secrets/` nếu thiếu. Nếu sau đó nạp lại dữ liệu cũ, đọc mục 9.

## 9. Nạp bản dump

Hoàn thiện ở Lab 8a ticket B9.
EOF
````

Ý chính cho ADR:

1. Control plane mang taint `NoSchedule`, nên Traefik và provisioner — workload thường — chỉ chạy được
   khi đã có worker. Bài 02: CNI là bắt buộc trước, CoreDNS chỉ chạy khi có mạng, và chỉ join khi
   CoreDNS `Running`. Thứ tự sai không làm lệnh lỗi ngay; nó làm Pod `Pending` không rõ lý do.
2. Mọi trình duyệt và file hosts đang dùng URL có cổng cũ. Giữ cố định bằng cách khai `nodePort` qua
   values của chart (đọc `helm show values traefik/traefik --version ...` để tìm đúng khóa) hoặc khi
   patch Service. Sprint này **không** đổi — mục tiêu là dựng lại đúng mốc `03-storage-ready`, và mốc
   đó có cổng do cluster cấp; ghi đó thành phát hiện của diễn tập.
3. Tải lúc khôi phục thì phụ thuộc GitHub đúng lúc thảm họa; lưu trong repo thì repo phình và phải tự
   cập nhật. sha256 bảo đảm file tải về đúng là file đã kiểm lúc viết quy trình — chặn nội dung bị đổi
   dưới cùng một tag; nó **không** giúp gì khi nguồn tải không còn, và không chứng minh bản đó an
   toàn.
4. Không có Helm thì không cài được Traefik — phải đi lại bước cài Helm của Lab 5b. Cũng nằm ngoài:
   ba VM đã cài gói (Lab 00 A4), file hosts trên máy host, bản dump và bundle trên máy host,
   `secrets/` trên master.
5. Đổi Service sang `NodePort`, đánh dấu IngressClass và StorageClass mặc định. Chúng là lệnh chạy một
   lần lên object do chart/manifest tạo, không nằm trong file nào; cài lại chart hay manifest là mất,
   và không ai nhớ cho tới khi Ingress hay PVC không hoạt động.

</details>

## B4. [BS-8A-04] Diễn tập: control plane vắng mặt

**Loại:** Diễn tập sự cố — trên `lab-k8s-master` (xem phạm vi phá hoại ở [mục 2](#2-quy-ước-và-an-toàn)).

> **Từ ticket này lab bắt đầu phá.** Cả hai kịch bản đều khôi phục được ngay trong ticket; việc phá
> không hồi lại bằng lệnh bắt đầu ở B6.

**Bối cảnh:** trước khi phá hẳn, đo hai sự cố nhỏ hơn mà cluster một control plane gặp thật: mất
`kube-apiserver` vì một file manifest bị dời đi, và mất cả máy control plane. Ban quản lý hỏi đúng một
câu: trong lúc đó **khách có còn xem và mua sách được không?**

**Yêu cầu:**

1. Một script kiểm data plane: từ một máy bất kỳ trong mạng lab, gọi trang chủ Bookstore qua Ingress
   HTTPS tại **cả hai** worker, ghi mã trả về của `/api/books`, và báo API server có trả lời không.
   Script sinh trên master (vì master sắp tắt), chép sang `lab-k8s-worker1`.
2. Kịch bản A — dời manifest của `kube-apiserver` ra khỏi thư mục static Pod (không xóa). Chứng minh:
   container `kube-apiserver` biến mất và `kubectl` mất đường; `crictl` và `journalctl` vẫn đọc được
   node; `etcd` vẫn chạy; trang chủ Bookstore qua Ingress vẫn trả về tại cả hai worker. Đưa file về
   đúng chỗ, giống hệt từng byte; `readyz` trở lại `ok`; `./tests/smoke.sh` PASS.
3. Kịch bản B — tắt hẳn `lab-k8s-master`. Từ `lab-k8s-worker1`: API không trả lời; trang chủ vẫn trả
   về tại cả hai worker; ghi lại `/api/books` trả gì. Bật master lại; ba node `Ready`; smoke PASS.
4. Postmortem `docs/postmortem/8a-control-plane-vang-mat.md`.

**Ràng buộc:** không `kubeadm init`, `kubeadm reset` hay `kubectl delete` để "sửa" trong ticket này. Chỉ
**dời** manifest, không xóa. Không sửa repo trong lúc diễn tập.

**Đọc lại:**

- Bài [09](../09-troubleshooting-kubeadm-vi.md):
  [*kubeadm bị treo khi chờ control plane*](../09-troubleshooting-kubeadm-vi.md#kubeadm-bị-treo-khi-chờ-control-plane-trong-quá-trình-cài-đặt-kubeadm-blocks-waiting-for-control-plane-during-installation)
  — công cụ khi `kubectl` vô dụng là `crictl`.
- Bài [02](../02-create-cluster-kubeadm-vi.md):
  [*Khả năng chống chịu của cluster*](../02-create-cluster-kubeadm-vi.md#resilience) — một control
  plane, một etcd, hai giải pháp.
- [Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) — static Pod và mirror Pod.

**Bàn giao:** `docs/postmortem/8a-control-plane-vang-mat.md`; `~/lab-work/8a/data-plane.sh`;
`~/lab-evidence/8a/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` (A) container `kube-apiserver` dừng; `kubectl` mất đường; `etcd` vẫn chạy; đọc được log
  kubelet bằng `journalctl`.
- `PASS:` (A) trang chủ qua Ingress trả về tại cả hai worker trong lúc API vắng mặt.
- `PASS:` (A) manifest đưa về trùng checksum cũ; `readyz` `ok`; smoke PASS.
- `PASS:` (B) từ worker1: API không trả lời; trang chủ trả về tại cả hai worker.
- `PASS:` (B) master bật lại, ba node `Ready`, smoke PASS.
- `PASS:` có postmortem.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. `kubectl delete pod kube-apiserver-lab-k8s-master` và `mv` file manifest đi chỗ khác — kết quả khác
   nhau thế nào? `ownerReferences` của mirror Pod trỏ về đâu, và điều đó giải thích sự khác biệt ra sao?
2. Trong kịch bản A, `kube-scheduler` và `kube-controller-manager` có còn chạy không, và làm được gì
   khi không có API server? Đọc bằng chứng từ `crictl ps -a`.
3. Kịch bản B: trang chủ vẫn phục vụ nhờ những thành phần nào trên worker? Chính xác thì bạn mất khả
   năng gì trong lúc master tắt?
4. `/api/books` trong kịch bản B trả gì trên cluster của bạn? Giải thích bằng nơi CoreDNS đang chạy và
   việc PostgREST phải phân giải `db-0.db` mỗi khi mở kết nối mới. (Không có đáp án chung — phụ thuộc
   vị trí Pod trên cluster của bạn.)
5. Bài 02 đưa hai giải pháp cho hạn chế của cluster một control plane. Cái nào là nợ lab #8, cái nào là
   Lab 8b, 8c?

<details>
<summary>Gợi ý</summary>

- `curl -sk --resolve <host>:<cổng>:<IP>` gọi đúng tên `bookstore.lab.local` (để khớp `host` của Ingress
  và TLS) tới một IP cụ thể, không cần sửa `/etc/hosts`.
- Trong heredoc không nháy, `\$H` giữ nguyên dấu `$` cho lúc script chạy, còn `$HTTPS_NP` được thay
  ngay lúc ghi — đúng thứ bạn cần khi script chạy trên máy không có `env.sh`.
- `sudo crictl ps --name <regex>`; `sudo journalctl -u kubelet -n 50 --no-pager`.
- Chờ API trở lại bằng vòng lặp trên `kubectl get --raw=/readyz`, không bằng `sleep` cố định.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đọc hết; script này chính là cuộc diễn tập</summary>

Sinh script kiểm data plane và chạy thử khi cluster còn khỏe — trên `lab-k8s-master`:

```bash
. ~/lab-evidence/8a/env.sh
cat > "$WK/data-plane.sh" <<EOF
#!/usr/bin/env bash
# Kiem data plane cua Bookstore khi control plane vang mat — sinh boi Lab 8a ticket B4.
H=bookstore.lab.local; NP=$HTTPS_NP
if timeout 3 bash -c 'echo > /dev/tcp/${CP_ENDPOINT%%:*}/$API_PORT' 2>/dev/null; then
  echo 'INFO: API server van tra loi'
else
  echo 'PASS: API server khong tra loi'
fi
for IP in $W1_IP $W2_IP; do
  if curl -sk --max-time 5 --resolve "\$H:\$NP:\$IP" "https://\$H:\$NP/" | grep -q 'Bookstore'; then
    echo "PASS: trang chu Bookstore qua Ingress tai \$IP"
  else
    echo "FAIL: trang chu khong tra ve tai \$IP"
  fi
  C="\$(curl -sk --max-time 5 -o /dev/null -w '%{http_code}' --resolve "\$H:\$NP:\$IP" "https://\$H:\$NP/api/books")"
  echo "INFO: /api/books tai \$IP -> HTTP \$C"
done
EOF
chmod +x "$WK/data-plane.sh"
bash "$WK/data-plane.sh" | tee "$EV/b4-truoc.txt"
scp -q "$WK/data-plane.sh" "$W1:/tmp/data-plane.sh" && echo 'PASS: da chep script sang worker1'
```

**PASS:** dòng `INFO: API server van tra loi`, hai dòng `PASS: trang chu ...`, `/api/books` trả `200`.

Kịch bản A — trên `lab-k8s-master`:

```bash
. ~/lab-evidence/8a/env.sh
M=/etc/kubernetes/manifests/kube-apiserver.yaml
sudo sha256sum "$M" | awk '{print $1}' > "$EV/b4-apiserver-sha.txt"
sudo crictl ps -a > "$EV/b4-a-crictl-truoc.txt"

sudo mv "$M" /etc/kubernetes/kube-apiserver.yaml.off
for i in $(seq 1 60); do [ -z "$(sudo crictl ps --name kube-apiserver -q)" ] && break; sleep 2; done
[ -z "$(sudo crictl ps --name kube-apiserver -q)" ] && echo 'PASS: container kube-apiserver da dung'
kubectl get nodes >/dev/null 2>&1 && echo 'FAIL: kubectl van chay' || echo 'PASS: kubectl mat duong toi API server'
sudo crictl ps -a > "$EV/b4-a-crictl.txt"
sudo journalctl -u kubelet -n 50 --no-pager > "$EV/b4-a-kubelet.txt"
{ [ -n "$(sudo crictl ps --name '^etcd$' -q)" ] && test -s "$EV/b4-a-kubelet.txt"; } \
  && echo 'PASS: etcd van chay; crictl va journalctl van doc duoc node'
bash "$WK/data-plane.sh" | tee "$EV/b4-a-data-plane.txt"
[ "$(grep -c '^PASS: trang chu' "$EV/b4-a-data-plane.txt")" -eq 2 ] \
  && echo 'PASS: Bookstore van phuc vu qua Ingress khi API vang mat'

sudo mv /etc/kubernetes/kube-apiserver.yaml.off "$M"
[ "$(sudo sha256sum "$M" | awk '{print $1}')" = "$(cat "$EV/b4-apiserver-sha.txt")" ] \
  && echo 'PASS: manifest dua lai dung tung byte'
for i in $(seq 1 150); do [ "$(kubectl get --raw=/readyz 2>/dev/null)" = 'ok' ] && break; sleep 2; done
[ "$(kubectl get --raw=/readyz)" = 'ok' ] && echo 'PASS: readyz ok tro lai'
kubectl wait --for=condition=Ready node --all --timeout=300s
cd "$REPO" && ./tests/smoke.sh > "$EV/b4-a-smoke.txt" 2>&1 && echo 'PASS: smoke sau kich ban A'
```

Kịch bản B — tắt master. Trên `lab-k8s-master`:

```bash
sudo shutdown -h now
```

Trên **máy host**, PowerShell của mục 2:

```powershell
& $vmrun -T ws list
```

**PASS:** danh sách VM đang chạy không còn `lab-k8s-master`, còn hai worker.

Trên **terminal của `lab-k8s-worker1`**:

```bash
bash /tmp/data-plane.sh | tee /tmp/b4-b-data-plane.txt
systemctl is-active kubelet containerd
sudo crictl ps | wc -l
```

**PASS:** dòng `PASS: API server khong tra loi`; hai dòng `PASS: trang chu ...`; `kubelet` và
`containerd` đều `active`; `crictl ps` vẫn liệt kê container. Ghi lại hai dòng `INFO: /api/books`.

Bật master lại — trên **máy host**:

```powershell
& $vmrun -T ws start $vmx[0]
```

SSH lại vào `lab-k8s-master`:

```bash
. ~/lab-evidence/8a/env.sh
for i in $(seq 1 150); do [ "$(kubectl get --raw=/readyz 2>/dev/null)" = 'ok' ] && break; sleep 2; done
[ "$(kubectl get --raw=/readyz)" = 'ok' ] && echo 'PASS: readyz ok sau khi master boot lai'
kubectl wait --for=condition=Ready node --all --timeout=600s && echo 'PASS: ba node Ready'
scp -q "$W1:/tmp/b4-b-data-plane.txt" "$EV/" && cat "$EV/b4-b-data-plane.txt"
cd "$REPO" && ./tests/smoke.sh > "$EV/b4-b-smoke.txt" 2>&1 && echo 'PASS: smoke sau kich ban B'
ls docs/postmortem/8a-control-plane-vang-mat.md >/dev/null 2>&1 && echo 'PASS: co postmortem'
```

Smoke có thể fail ngay sau khi master lên vì vài Pod còn đang hội tụ; chờ
`kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded` trả rỗng rồi chạy
lại — thời gian hội tụ phụ thuộc cấu hình, không có con số cam kết.

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

Postmortem gồm bảng diễn biến — mỗi mốc của hai kịch bản một dòng: quan sát, thời gian, file bằng chứng
`b4-*` — rồi câu trả lời. Ý chính:

1. Mirror Pod có `ownerReferences` trỏ về **Node**, nguồn `kubernetes.io/config.source: file`. Desired
   state nằm trong file trên đĩa, không trong etcd. Xóa mirror Pod thì kubelet tạo lại ngay vì file
   còn; dời file thì kubelet dừng container và không controller nào dựng lại.
2. Hai tiến trình đó vẫn được kubelet giữ chạy nhưng không làm được gì: mọi việc của chúng đi qua API
   server. Chúng có thể mất lease của leader election và tự khởi động lại — đếm `ATTEMPT` trong
   `crictl ps -a` trước và sau để thấy.
3. Kubelet trên worker giữ Pod đang chạy; quy tắc mà kube-proxy và Calico đã ghi vào node vẫn chuyển
   tiếp NodePort và lưu lượng Pod; Traefik giữ cấu hình định tuyến đã nạp. Thứ mất là mọi **thay đổi**:
   không lập lịch Pod mới, không thay Pod hỏng, không `kubectl`, không cập nhật Service hay Ingress.
4. Trang chủ là file tĩnh trong Pod `web`, không cần DNS. `/api/books` cần nginx tới `api` và PostgREST
   tới `db-0.db`; kết nối mới phải phân giải tên. Nếu CoreDNS có bản sao trên worker, nó trả lời từ dữ
   liệu đã có và `/api/books` thường vẫn `200`; nếu CoreDNS chỉ nằm trên master thì phân giải hỏng và
   request lỗi một phần hay toàn bộ. Ghi đúng điều bạn thấy và giải thích bằng vị trí Pod.
5. Sao lưu etcd định kỳ (thư mục `/var/lib/etcd`) — nợ lab #8, giai đoạn 19. Nhiều control plane —
   Lab 8b, 8c với bài 06, 07, 08.

</details>

## B5. [BS-8A-05] Sao lưu ra khỏi cluster trước khi phá

**Loại:** Vận hành.

**Bối cảnh:** sau ticket này là điểm không quay lại bằng lệnh. Mọi thứ cần để dựng lại phải **đã nằm
ngoài VM**: repo (có `platform/` của B1–B3) và một bản dump DB mới. Đây cũng là lúc đặt mốc RPO: dữ liệu
ghi **sau** bản dump sẽ mất, và diễn tập phải chứng minh đúng điều đó bằng một đơn hàng cụ thể.

**Yêu cầu:**

1. Ghi một đơn hàng "trước dump", rồi đếm số dòng `api.books` và `api.orders`.
2. Chạy **một** lần `db-backup` ngay, không chờ lịch; log có dòng `db-backup: OK`.
3. Ghi một đơn hàng "sau dump" — đơn này sẽ mất khi khôi phục.
4. Lấy đúng file dump của lần chạy đó ra khỏi PVC `backup-store` về master. sha256 trong PVC và trên
   master bằng nhau. Dump có dữ liệu của `api.books` và `api.orders`, và **không** chứa câu lệnh tạo
   hay sửa role.
5. Ghi các mốc vào `~/lab-evidence/8a/b5-moc.env`, mỗi dòng `KHOA=giatri` không dấu nháy:
   `BACKUP_JOB`, `DUMP_FILE` (tên file, không đường dẫn), `DUMP_SHA256`, `DUMP_TIME` (lúc Job hoàn
   tất), `ID_TRUOC`, `ID_SAU`, `N_BOOKS`, `N_ORDERS` — và `COMMIT` sau bước 7. B9 dựa vào file này.
6. Phần *1. Trước sự cố* của `docs/postmortem/8a-dung-lai-cluster.md`: thời điểm dump, id hai đơn, số
   dòng, sha256, và câu trả lời các câu hỏi thiết kế dưới đây.
7. Commit mọi thay đổi của B1–B5; bundle `~/bookstore-8a-truoc-pha.bundle`; chép bundle, bản dump và
   `b5-moc.env` ra máy host, sha256 của bản dump trên host bằng giá trị đã ghi.
8. Xóa bản dump trên master. B9 phải khôi phục từ bản **trên máy host**.

**Ràng buộc:** Pod dùng để lấy dump là object thử nghiệm ở `~/lab-work/8a/`, không mang label dự án,
mount PVC **chỉ đọc**, xóa khi xong. Không sửa CronJob. Không in mật khẩu. Bản dump **không** vào repo.

**Đọc lại:**

- [Mục 3.4](DU-AN-BOOKSTORE.md#34-luồng-request-và-luồng-dữ-liệu) — luồng dữ liệu bền, bản dump đi
  ra máy host bằng tay; [mục 5.4](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm);
  [mục 8](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án) — `TD-10`.
- [Lab 4b](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md), ticket B6 — tạo Job từ CronJob, đọc log theo Job.
- PostgreSQL 17: [pg_dump](https://www.postgresql.org/docs/17/app-pgdump.html) — đoạn mở đầu nói
  `pg_dump` chỉ dump **một** database; role và các đối tượng toàn cụm thuộc
  [pg_dumpall](https://www.postgresql.org/docs/17/app-pg-dumpall.html).

**Bàn giao:** `docs/postmortem/8a-dung-lai-cluster.md` (phần 1); `~/lab-work/8a/lay-dump.yaml`; trên
máy host: bundle, bản dump, `b5-moc.env`; `~/lab-evidence/8a/b5-*`.

**Nghiệm thu:**

- `PASS:` hai đơn hàng mốc tồn tại, đơn "sau dump" có `id` lớn hơn.
- `PASS:` một lần chạy `db-backup` thành công.
- `PASS:` sha256 trong PVC, trên master và trong `b5-moc.env` bằng nhau.
- `PASS:` dump có dữ liệu `api.books`, `api.orders`; không có lệnh role.
- `PASS:` Pod `lay-dump` đã xóa.
- `PASS:` repo sạch, HEAD là `COMMIT`; bundle hợp lệ và chứa commit đó; postmortem ghi sha256.
- `PASS:` (máy host) bản dump đúng sha256; bundle có trên host.
- `PASS:` bản dump trên master đã xóa.

**Câu hỏi thiết kế** — trả lời trong phần 1 của postmortem:

1. RPO của **cuộc diễn tập** là bao nhiêu — tính bằng thời gian và bằng dữ liệu? Nếu sự cố đến **không
   báo trước**, RPO thật của hệ thống là bao nhiêu với lịch `db-backup` hiện tại và cách chép tay ra
   máy host (`TD-10`)?
2. Vì sao bản dump còn nằm trong PVC `backup-store` không tính là bản sao lưu cho kịch bản mất cluster?
3. Dump không chứa role và mật khẩu. Khi khôi phục sang cluster mới với `secrets/` mới, đó là điểm yếu
   hay điểm mạnh? Đối chiếu với đoạn cuối [mục 5.4](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm).
4. Vì sao bundle phải tạo **sau** khi `platform/` đã commit? Vì sao bản dump không được vào repo?

<details>
<summary>Gợi ý</summary>

- Lấy `id` của dòng vừa chèn trong một câu lệnh: `WITH x AS (INSERT ... RETURNING id) SELECT id FROM x`
  — kết quả là một `SELECT`, nên `psql -tA` chỉ in con số.
- Một Pod `busybox` mount PVC `backup-store` với `readOnly: true`. PVC dùng volume `local-path` gắn với
  một node — scheduler tự đặt Pod lên đúng node đó.
- Namespace có ResourceQuota và LimitRange (sprint 11): khai `requests`/`limits` nhỏ cho Pod thử nghiệm.
- Log của `db-backup` có dạng `db-backup: OK <đường dẫn> <số byte> byte`; tên file nằm ở cột 3.
- `kubectl exec <pod> -- cat <file> > <file cục bộ>` chép file ra; so `sha256sum` hai đầu.
- `git bundle list-heads <bundle>` liệt kê commit mà bundle chứa.

</details>

<details>
<summary>Script nghiệm thu — ba phần, chạy đúng lúc ghi ở đầu mỗi phần</summary>

Phần 1 — trên `lab-k8s-master`, sau khi đã bundle, **trước** khi xóa bản dump:

```bash
. ~/lab-evidence/8a/env.sh; . "$EV/b5-moc.env"
SQL() { kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc "$1"; }
{ [ "$ID_SAU" -gt "$ID_TRUOC" ] && \
  [ "$(SQL "SELECT count(*) FROM api.orders WHERE id IN ($ID_TRUOC, $ID_SAU)")" = '2' ]; } \
  && echo 'PASS: hai don hang moc ton tai, don sau dump ghi sau'
{ [ "$(kubectl -n "$NS" get job "$BACKUP_JOB" -o jsonpath='{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ] && \
  kubectl -n "$NS" logs "job/$BACKUP_JOB" | grep -q '^db-backup: OK'; } \
  && echo 'PASS: mot lan chay db-backup thanh cong'
F="$WK/dump/$DUMP_FILE"
{ [ "$(sha256sum < "$F" | awk '{print $1}')" = "$DUMP_SHA256" ] && \
  [ "$(awk '{print $2}' "$EV/b5-sha.txt" | sort -u)" = "$DUMP_SHA256" ]; } \
  && echo 'PASS: sha256 trong PVC, tren master va trong b5-moc.env bang nhau'
grep -Eq '^COPY api\.books |INSERT INTO api\.books' "$F" && grep -Eq '^COPY api\.orders |INSERT INTO api\.orders' "$F" \
  && echo 'PASS: dump co du lieu api.books va api.orders'
grep -Eq '(CREATE|ALTER) ROLE' "$F" && echo 'FAIL: dump chua lenh role' || echo 'PASS: dump khong chua role hay mat khau'
kubectl -n "$NS" get pod lay-dump >/dev/null 2>&1 && echo 'FAIL: con Pod lay-dump' || echo 'PASS: da xoa Pod lay-dump'
cd "$REPO"
{ test -z "$(git status --porcelain)" && [ "$(git rev-parse HEAD)" = "$COMMIT" ]; } && echo 'PASS: repo sach, HEAD la COMMIT'
{ git bundle verify ~/bookstore-8a-truoc-pha.bundle >/dev/null 2>&1 && \
  git bundle list-heads ~/bookstore-8a-truoc-pha.bundle | grep -q "^$COMMIT"; } && echo 'PASS: bundle hop le, chua COMMIT'
grep -q "$DUMP_SHA256" docs/postmortem/8a-dung-lai-cluster.md && echo 'PASS: postmortem ghi sha256 cua dump'
```

Phần 2 — trên **máy host**, PowerShell. Sửa `$masterHost` theo
[A1.2](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) và `$masterUser` theo user quản trị của bạn:

```powershell
$masterUser = 'ubuntu'
$masterHost = '<IP cua lab-k8s-master theo bang A1.2>'
$dest = "$HOME\lab-backup\8a"
if ($masterHost -like '*<*') { throw 'FAIL: chua dien dia chi that cua master' }
New-Item -ItemType Directory -Force -Path $dest | Out-Null
scp "${masterUser}@${masterHost}:~/lab-evidence/8a/b5-moc.env" $dest
scp "${masterUser}@${masterHost}:~/bookstore-8a-truoc-pha.bundle" $dest
$file = (Select-String -Path "$dest\b5-moc.env" -Pattern '^DUMP_FILE=(.+)$').Matches[0].Groups[1].Value
$want = (Select-String -Path "$dest\b5-moc.env" -Pattern '^DUMP_SHA256=(.+)$').Matches[0].Groups[1].Value
scp "${masterUser}@${masterHost}:~/lab-work/8a/dump/$file" $dest
$got = (Get-FileHash -Algorithm SHA256 -LiteralPath "$dest\$file").Hash.ToLowerInvariant()
if ($got -eq $want) { 'PASS: ban dump tren may host dung sha256' } else { "FAIL: $got khac $want" }
if ((Get-Item "$dest\bookstore-8a-truoc-pha.bundle").Length -gt 0) { 'PASS: bundle nam tren may host' }
```

Phần 3 — trên `lab-k8s-master`, sau khi phần 2 PASS và bạn đã xóa bản dump:

```bash
test ! -e ~/lab-work/8a/dump && echo 'PASS: ban dump tren master da xoa — B9 khoi phuc tu ban tren may host'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/8a/lay-dump.yaml — Pod thử nghiệm, không mang label dự án
apiVersion: v1
kind: Pod
metadata:
  name: lay-dump
  namespace: bookstore
spec:
  restartPolicy: Never
  volumes:
  - name: backup
    persistentVolumeClaim:
      claimName: backup-store
      readOnly: true
  containers:
  - name: shell
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 50m, memory: 32Mi}
    volumeMounts:
    - name: backup
      mountPath: /backup
      readOnly: true
```

```bash
. ~/lab-evidence/8a/env.sh
SQL() { kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc "$1"; }
DON='WITH x AS (INSERT INTO api.orders (book_id, quantity) SELECT min(id), 1 FROM api.books RETURNING id) SELECT id FROM x'

# 1. Đơn "trước dump" và số dòng
ID_TRUOC="$(SQL "$DON")"
N_BOOKS="$(SQL 'SELECT count(*) FROM api.books')"
N_ORDERS="$(SQL 'SELECT count(*) FROM api.orders')"

# 2. Một lần db-backup
J="dien-tap-8a-$(date +%s)"
kubectl -n "$NS" create job "$J" --from=cronjob/db-backup
kubectl -n "$NS" wait --for=condition=Complete "job/$J" --timeout=300s
L="$(kubectl -n "$NS" logs "job/$J" | grep '^db-backup: OK' | tail -1)"; echo "$L"
DUMP_TIME="$(kubectl -n "$NS" get job "$J" -o jsonpath='{.status.completionTime}')"

# 3. Đơn "sau dump"
ID_SAU="$(SQL "$DON")"

# 4. Lấy file dump ra khỏi PVC
kubectl apply -f "$WK/lay-dump.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/lay-dump --timeout=180s
NAME="$(basename "$(echo "$L" | awk '{print $3}')")"
P="$(kubectl -n "$NS" exec lay-dump -- find /backup -name "$NAME" | head -1)"
[ -n "$P" ] || P="$(kubectl -n "$NS" exec lay-dump -- sh -c 'ls -t /backup/*.sql | head -1')"
NAME="$(basename "$P")"
mkdir -p "$WK/dump"
kubectl -n "$NS" exec lay-dump -- cat "$P" > "$WK/dump/$NAME"
{ echo "pvc $(kubectl -n "$NS" exec lay-dump -- sha256sum "$P" | awk '{print $1}')"
  echo "master $(sha256sum < "$WK/dump/$NAME" | awk '{print $1}')"; } > "$EV/b5-sha.txt"
cat "$EV/b5-sha.txt"
kubectl -n "$NS" delete pod lay-dump --wait=true

# 5. Mốc
cat > "$EV/b5-moc.env" <<EOF
BACKUP_JOB=$J
DUMP_FILE=$NAME
DUMP_SHA256=$(sha256sum < "$WK/dump/$NAME" | awk '{print $1}')
DUMP_TIME=$DUMP_TIME
ID_TRUOC=$ID_TRUOC
ID_SAU=$ID_SAU
N_BOOKS=$N_BOOKS
N_ORDERS=$N_ORDERS
EOF
cat "$EV/b5-moc.env"
```

Phần 1 của postmortem — viết bằng lời của bạn, giữ các con số:

```markdown
# Postmortem 8a — diễn tập dựng lại cluster

- Ticket: BS-8A-05 → BS-8A-10; kịch bản: mất toàn bộ cluster, khôi phục từ repo và bản dump

## 1. Trước sự cố
| Mục | Giá trị |
| --- | --- |
| Bản dump | <DUMP_FILE>, sha256 <DUMP_SHA256>, hoàn tất lúc <DUMP_TIME> |
| Dữ liệu lúc dump | <N_BOOKS> sách, <N_ORDERS> đơn; đơn mốc trước dump id <ID_TRUOC> |
| Ghi sau dump (sẽ mất) | đơn id <ID_SAU> |
| Nơi giữ bản sao | máy host: bundle repo, bản dump, b5-moc.env |
```

```bash
cd ~/bookstore
git add -A && git commit -m 'lab-8a: platform/ va ghi chep truoc dien tap dung lai cluster'
echo "COMMIT=$(git rev-parse HEAD)" >> ~/lab-evidence/8a/b5-moc.env
git bundle create ~/bookstore-8a-truoc-pha.bundle --all && git bundle verify ~/bookstore-8a-truoc-pha.bundle
```

Sau khi phần 2 của script nghiệm thu PASS trên máy host: `rm -rf ~/lab-work/8a/dump`.

Ý chính cho postmortem:

1. Thời gian: từ `DUMP_TIME` tới lúc bắt đầu phá (mốc `T0` ở B6). Dữ liệu: đúng đơn `ID_SAU`. Không báo
   trước thì RPO thật là khoảng từ lần chép ra ngoài gần nhất: tối đa một chu kỳ lịch `db-backup` **cộng**
   thời gian từ lúc dump tới lúc có người chép tay ra máy host — với `TD-10` khoảng đó có thể là hàng
   giờ hay hàng ngày.
2. PVC `backup-store` nằm trong cluster, dữ liệu của nó nằm trên đĩa một worker. Mất cluster thì mất
   object PV/PVC trỏ tới nó; mất node thì mất luôn dữ liệu. Bản sao lưu phải sống được sự cố mà nó
   phòng ngừa.
3. Điểm mạnh cho kịch bản này: role và mật khẩu do script khởi tạo của cluster **mới** tạo, từ
   `secrets/` mới; dump chỉ mang schema và dữ liệu, nên mật khẩu không lệch. Đoạn cuối mục 5.4 đúng
   khi **thư mục dữ liệu** được giữ lại (PV `Retain`), nơi role và mật khẩu nằm sẵn trong dữ liệu;
   với `pg_dump` một database thì không. Cái giá: role phải được tạo lại bởi chính hệ thống.
4. Bundle phải chứa đúng thứ dùng để dựng lại: thiếu `platform/` thì quy trình chỉ còn trong trí nhớ.
   Bản dump là dữ liệu khách hàng, không phải mã: repo bị bundle và chép đi nhiều nơi, có lịch sử
   không xóa được — dữ liệu vào đó là rò rỉ vĩnh viễn.

</details>

## B6. [BS-8A-06] Diễn tập: gỡ cluster đúng thứ tự

**Loại:** Diễn tập sự cố — cả ba node.

> **Từ ticket này cluster bị phá không hồi lại được bằng lệnh.** Đường về duy nhất là restore
> `03-storage-ready` ở B11. Kiểm lại: phần 2 của script nghiệm thu B5 đã PASS trên máy host chưa?

**Bối cảnh:** kịch bản là mất toàn bộ cluster. Diễn tập vẫn gỡ cluster **đúng quy trình** của bài 02 —
vì đó là thứ người vận hành phải làm khi rút node hay dọn cluster thật — và quan sát hệ thống Bookstore
phản ứng ra sao khi node bị rút đi.

**Yêu cầu:**

1. Ghi mốc `T0-bat-dau-pha` trước lệnh phá đầu tiên. Đồng hồ RTO chạy từ đây.
2. Rút từng worker theo đúng ba bước của bài 02, `lab-k8s-worker2` trước. Nếu `drain` bị chặn, ghi lại
   thông báo, chỉ ra object chặn, rồi chọn cách đi tiếp **có ý thức** — và ghi lý do vào postmortem.
3. Reset control plane. Chứng minh `kubeadm reset` **không** dọn quy tắc iptables; biết kube-proxy
   chạy chế độ nào để chọn đúng lệnh dọn; dọn bằng lệnh của bài 02 trên cả ba node.
4. Dọn cấu hình CNI cũ trên ba node và kubeconfig cũ của user trên master. Ghi lại dữ liệu ứng dụng còn
   sót trên worker — thư mục của local-path-provisioner — **không xóa**.
5. Reboot ba node. Chạy lại bảng *Điều kiện của node* của `platform/README.md` và chứng minh không còn
   dấu vết cluster cũ: quy tắc iptables, interface của CNI, `/etc/cni/net.d`, `pki`, `etcd`, manifest.
6. Ghi mốc `reset-xong`; phần *2. Diễn biến* của postmortem có đoạn về `drain`.

**Ràng buộc:** không `--ignore-preflight-errors`, không `kubeadm init` ở ticket này. Chỉ flush iptables
khi policy của chain `INPUT` là `ACCEPT`. Không xóa `/opt/local-path-provisioner`, `~/bookstore`,
`~/lab-*`. Không có `ipvsadm` trên node — kiểm trước xem có cần không.

**Đọc lại:**

- Bài [02](../02-create-cluster-kubeadm-vi.md): [*Dọn dẹp*](../02-create-cluster-kubeadm-vi.md#tear-down),
  [*Gỡ bỏ node*](../02-create-cluster-kubeadm-vi.md#gỡ-bỏ-node-remove-the-node),
  [*Dọn dẹp control plane*](../02-create-cluster-kubeadm-vi.md#dọn-dẹp-control-plane-clean-up-the-control-plane).
- Bài [09](../09-troubleshooting-kubeadm-vi.md),
  [*kubeadm bị treo khi xóa các container được quản lý*](../09-troubleshooting-kubeadm-vi.md#kubeadm-bị-treo-khi-xóa-các-container-được-quản-lý-kubeadm-blocks-when-removing-managed-containers).
- [Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) — PodDisruptionBudget và Eviction API.
- Tài liệu [`kubeadm reset`](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
  — thư mục CNI và `$HOME/.kube` không được dọn;
  [`kubectl drain`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_drain/) — cờ
  `--timeout`, `--disable-eviction`.

**Bàn giao:** postmortem phần 2 (đoạn `drain`); `~/lab-evidence/8a/b6-*.txt`;
`~/lab-evidence/8a/moc-thoi-gian.txt`.

**Nghiệm thu:**

- `PASS:` mỗi worker drain xong — `unschedulable`, chỉ còn Pod DaemonSet — rồi reset và bị xóa khỏi
  cluster; thông báo chặn (nếu có) nằm trong bằng chứng.
- `PASS:` kube-proxy không chạy `ipvs`; policy `INPUT` là `ACCEPT` trước khi flush.
- `PASS:` sau reset vẫn còn quy tắc `KUBE-`/`CALI-` trong iptables — reset không dọn.
- `PASS:` dữ liệu cũ của `bookstore` vẫn nằm dưới `/opt/local-path-provisioner` trên worker.
- `PASS:` sau reboot, mỗi node: swap tắt, đủ module và sysctl, `containerd` active, `kubelet` enabled,
  không còn quy tắc và interface cũ, `/etc/cni/net.d` rỗng, không còn `pki`, `etcd`, manifest.
- `PASS:` `moc-thoi-gian.txt` có `T0-bat-dau-pha` và `reset-xong`.

**Câu hỏi thiết kế** — trả lời trong postmortem phần 2:

1. `drain` của worker nào bị chặn, bởi object nào của dự án? Vì sao bị chặn là **đúng thiết kế**? Bạn
   đi tiếp bằng cách nào, vì sao chấp nhận được ở đây nhưng không chấp nhận khi bảo trì một node
   (Lab 12)?
2. Hai thứ bài 02 nói `kubeadm reset` không đặt lại, cộng hai thứ tài liệu `kubeadm reset` nhắc. Cluster
   của bạn cần lệnh dọn nào, vì sao không cần `ipvsadm`?
3. Thư mục dữ liệu PostgreSQL cũ vẫn nằm trên `lab-k8s-worker2`. Vì sao diễn tập **không** khôi phục từ
   nó? Trong tình huống nào nó lại là cứu cánh, và gắn lại nó cần làm gì (Lab 6a)?
4. Vì sao reboot ở đây là một phép kiểm bài 01 chứ không phải "cho chắc"?

<details>
<summary>Gợi ý</summary>

- `kubectl drain` thử lại mãi khi Eviction API bị từ chối; `--timeout` cho nó dừng. Cờ
  `--disable-eviction` bắt `drain` dùng xóa thường thay cho eviction — bỏ qua PodDisruptionBudget, đúng
  như `kubectl delete pod` mà Lab 3c đã so với Eviction API.
- Pod còn lại trên node sau drain: `--field-selector spec.nodeName=<node>`; kiểu chủ sở hữu ở
  `.metadata.ownerReferences[0].kind`.
- Chế độ kube-proxy nằm trong ConfigMap `kube-proxy` (`kube-system`), khóa `config.conf`, dòng `mode:`;
  bản chụp ở `b0-truoc.txt`.
- Đếm quy tắc: `sudo iptables-save | grep -cE 'KUBE-|CALI-'`.
- Hàm `on` trong `env.sh` chạy cùng một khối lệnh trên cả ba node.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đọc hết; script này chính là cuộc diễn tập</summary>

Khối 1 — rút hai worker. Trên `lab-k8s-master`:

```bash
. ~/lab-evidence/8a/env.sh
moc T0-bat-dau-pha
for N in "$W2" "$W1"; do
  if ! kubectl drain "$N" --delete-emptydir-data --force --ignore-daemonsets --timeout=180s \
       > "$EV/b6-drain-$N.txt" 2>&1; then
    echo "drain $N bi chan:"; grep -m3 'disruption budget' "$EV/b6-drain-$N.txt"
    kubectl drain "$N" --delete-emptydir-data --force --ignore-daemonsets --disable-eviction \
      --timeout=300s >> "$EV/b6-drain-$N.txt" 2>&1
  fi
  ALL="$(kubectl get pods -A --no-headers --field-selector "spec.nodeName=$N" | wc -l)"
  DS="$(kubectl get pods -A --field-selector "spec.nodeName=$N" \
        -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{"\n"}{end}' | grep -c '^DaemonSet$')"
  { [ "$(kubectl get node "$N" -o jsonpath='{.spec.unschedulable}')" = 'true' ] && [ "$ALL" -eq "$DS" ]; } \
    && echo "PASS: $N da drain xong, chi con Pod DaemonSet"
  ssh "$N" 'sudo kubeadm reset -f' > "$EV/b6-reset-$N.txt" 2>&1 && echo "PASS: kubeadm reset tren $N"
  kubectl delete node "$N"
done
grep -l 'disruption budget' "$EV"/b6-drain-*.txt && echo 'INFO: co lan drain bi PodDisruptionBudget chan — giai thich trong postmortem'
[ "$(kubectl get nodes --no-headers | wc -l)" -eq 1 ] && echo 'PASS: chi con control plane trong cluster'
kubectl -n "$NS" get pods -o wide > "$EV/b6-pod-con-lai.txt"
```

Khối 2 — reset control plane:

```bash
MODE="$(kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | awk '$1=="mode:"{print $2}')"
echo "kube-proxy mode=${MODE}" | tee "$EV/b6-kube-proxy-mode.txt"
case "$MODE" in *ipvs*) echo 'FAIL: kube-proxy chay ipvs — can quy trinh don khac, doc lai bai 02' ;;
  *) echo 'PASS: kube-proxy khong chay ipvs, khong can ipvsadm' ;; esac
sudo kubeadm reset -f > "$EV/b6-reset-master.txt" 2>&1 && echo 'PASS: kubeadm reset tren master'
rm -f "$HOME/.kube/config"
kubectl get nodes >/dev/null 2>&1 && echo 'FAIL: van con API server' || echo 'PASS: khong con cluster nao'
```

Khối 3 — iptables, CNI, dữ liệu sót, trên cả ba node:

```bash
for n in $NODES; do
  echo "$n $(on "$n" "sudo iptables-save | grep -cE 'KUBE-|CALI-'")"
done | tee "$EV/b6-iptables-sau-reset.txt"
awk '$2>0' "$EV/b6-iptables-sau-reset.txt" | grep -q . && echo 'PASS: kubeadm reset KHONG don iptables — dung bai 02'
BAD=0
for n in $NODES; do
  on "$n" 'sudo iptables -S | grep -q "^-P INPUT ACCEPT"' || { echo "FAIL: $n INPUT khong ACCEPT — dung, doc muc 4"; BAD=1; continue; }
  on "$n" 'sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X'
  on "$n" 'sudo sh -c "rm -rf /etc/cni/net.d/*"'
done
[ "$BAD" -eq 0 ] && echo 'PASS: policy INPUT ACCEPT; da flush theo bai 02 va don /etc/cni/net.d'
for n in "$W1" "$W2"; do echo "== $n"; ssh "$n" 'sudo ls -1 /opt/local-path-provisioner 2>/dev/null'; done \
  > "$EV/b6-du-lieu-sot.txt"
grep -q '_bookstore_' "$EV/b6-du-lieu-sot.txt" && echo 'PASS: du lieu cu cua bookstore van nam tren worker sau reset'
```

Khối 4 — reboot, worker trước, master sau:

```bash
ssh "$W2" 'sudo reboot'; ssh "$W1" 'sudo reboot'; sudo reboot
```

Khối 5 — SSH lại vào `lab-k8s-master` khi cả ba đã lên:

```bash
. ~/lab-evidence/8a/env.sh
for n in $NODES; do
  for i in $(seq 1 60); do on "$n" true 2>/dev/null && break; sleep 5; done
  on "$n" '
    B=0
    [ -z "$(swapon --show)" ] || { echo "FAIL: swap bat lai"; B=1; }
    [ "$(lsmod | grep -cE "^(overlay|br_netfilter) ")" -eq 2 ] || { echo "FAIL: thieu module"; B=1; }
    [ "$(sysctl -n net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward | grep -c "^1$")" -eq 3 ] \
      || { echo "FAIL: sysctl"; B=1; }
    { [ "$(systemctl is-active containerd)" = active ] && [ "$(systemctl is-enabled kubelet)" = enabled ]; } \
      || { echo "FAIL: containerd hoac kubelet"; B=1; }
    [ "$(sudo iptables-save | grep -cE "KUBE-|CALI-")" -eq 0 ] || { echo "FAIL: con quy tac iptables cu"; B=1; }
    ip -br link | grep -qE "^(cali|vxlan\.calico|flannel|cni0)" && { echo "FAIL: con interface CNI cu"; B=1; }
    [ -z "$(sudo ls -A /etc/cni/net.d 2>/dev/null)" ] || { echo "FAIL: /etc/cni/net.d chua rong"; B=1; }
    { sudo test ! -e /etc/kubernetes/pki && sudo test ! -e /var/lib/etcd; } || { echo "FAIL: con pki hoac etcd"; B=1; }
    [ -z "$(sudo ls -A /etc/kubernetes/manifests 2>/dev/null)" ] || { echo "FAIL: manifests chua rong"; B=1; }
    [ "$B" -eq 0 ] && echo "PASS: $(hostname) sach va du dieu kien sau reboot"
  ' | tee -a "$EV/b6-sau-reboot.txt"
done
moc reset-xong
cat "$EV/moc-thoi-gian.txt"
```

**PASS:** ba dòng `PASS: <node> sach va du dieu kien sau reboot`, không dòng `FAIL:`. `kubelet` lúc này
`enabled` nhưng **không** `active` — đúng: nó chờ `kubeadm` ra lệnh, như Lab 00 A4.3 đã báo trước.

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem phần 2</summary>

1. Thường là worker rút **sau cùng**. Khi worker đầu bị drain, Pod `web`, `api` dồn sang worker còn
   lại; tới worker đó, Pod thay thế không còn chỗ nào để chạy (control plane mang taint, worker kia đã
   cordon), nên PodDisruptionBudget `web`/`api` từ chối eviction để giữ số bản sao tối thiểu — đúng
   việc PDB được viết ra để làm ở Lab 3c. Ở đây cả cluster sắp bị gỡ và gián đoạn là chủ đích, nên dùng
   `--disable-eviction` (xóa thường, bỏ qua PDB) là quyết định có ý thức. Khi bảo trì một node, PDB
   chặn là tín hiệu phải dừng: thêm chỗ hoặc chờ, không phá ngân sách gián đoạn.
2. Bài 02: không đặt lại quy tắc iptables, không đặt lại bảng IPVS. Tài liệu reset thêm: không dọn
   `/etc/cni/net.d`, không xóa `$HOME/.kube/config`. kube-proxy chạy chế độ mặc định (không phải
   `ipvs`, xem `b0-truoc.txt`), nên chỉ cần lệnh `iptables -F ...` của bài 02; reboot xóa nốt phần còn
   lại vì quy tắc chỉ nằm trong bộ nhớ.
3. Kịch bản là mất cluster — trong thực tế thường kéo theo mất node hoặc đĩa; diễn tập phải chứng minh
   bản sao lưu **ngoài** cluster đủ dùng. Khi node và đĩa còn nguyên mà chỉ mất control plane, thư mục
   đó giữ dữ liệu tới tận giây cuối, tốt hơn bản dump. Gắn lại cần tạo PV tĩnh trỏ đúng thư mục và
   node, `claimRef` tới PVC `data-db-0` (Lab 6a); và dùng lại đúng `secrets/db.env` cũ, vì role và mật
   khẩu nằm trong thư mục dữ liệu đó (mục 5.4).
4. Bài 01 đòi điều kiện **bền**: swap tắt qua reboot, module và sysctl nạp lại khi khởi động. Phép kiểm
   chạy ngay sau reboot là cách duy nhất biết chúng bền thật, trước khi `kubeadm init` dựa vào chúng.

</details>

## B7. [BS-8A-07] Dựng lại control plane từ repo

**Loại:** Vận hành.

**Bối cảnh:** ba máy đã sạch. Bước đầu của quy trình là control plane — và lần này nó được dựng từ file
cấu hình đã nằm trong bundle trên máy host, không từ lịch sử shell. Nếu file đó thiếu hay sai, đây là
lúc phát hiện.

**Yêu cầu:**

1. Repo ở đúng commit đã bundle ở B5; `platform/` không có thay đổi chưa commit (ghi chép vào postmortem
   thì được — chúng được commit ở B11).
2. Control plane dựng **chỉ** từ `platform/kubeadm/kubeadm-config.yaml`: không cờ nào khác ngoài
   `--config`. Trước khi init: danh sách image đúng phiên bản; phase `preflight` không có lỗi.
3. Output của `kubeadm init` vào bằng chứng **sau khi che token**. Tìm được hiện vật của từng phase:
   `certs`, `kubeconfig`, `etcd`, `control-plane`, `kubelet-start` trên đĩa; `upload-config`,
   `mark-control-plane`, `bootstrap-token`, `addon` trong cluster.
4. Kubeconfig cho user thường theo [Lab 00 A5.1](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane).
   Node control plane có taint mặc định mà không ai gõ lệnh taint.
5. Patch của B2 có trong manifest `kube-scheduler` trên đĩa và trên mirror Pod, **không** có trong
   manifest `kube-apiserver`.
6. Trước khi có CNI, CoreDNS `Pending` — ghi lại sự kiện của nó. Cài Calico theo mục 3 của
   `platform/README.md`: CoreDNS `Running`, node `Ready`, IPPool bằng `podSubnet`.
7. CA mới: hash join của cluster mới khác hash ghi ở B1.
8. Ghi mốc `init-xong` và `cni-xong`.

**Ràng buộc:** không `--ignore-preflight-errors`. `init` lỗi thì **đừng chạy lại ngay**: tra bài 09,
`sudo kubeadm reset -f`, rồi mới chạy lại — `init` lần hai trên trạng thái bẩn đẻ ra triệu chứng khác che
nguyên nhân gốc. README thiếu hay sai bước nào thì sửa README (commit ở B11), đừng làm tay rồi quên.

**Đọc lại:**

- Bài [02](../02-create-cluster-kubeadm-vi.md):
  [*Khởi tạo control-plane node*](../02-create-cluster-kubeadm-vi.md#khởi-tạo-control-plane-node-initializing-your-control-plane-node)
  và mục *Thông tin thêm* ngay sau — preflight, muốn chạy lại `init` phải gỡ cluster trước;
  [*Cài đặt Pod network add-on*](../02-create-cluster-kubeadm-vi.md#pod-network).
- Bài [03](../03-control-plane-flags-vi.md#patches) — patch là bước tùy chỉnh cuối cùng trước khi ghi
  xuống đĩa.
- Bài [04](../04-kubelet-integration-vi.md#quy-trình-khi-dùng-kubeadm-init-workflow-when-using-kubeadm-init)
  — `init` ghi gì cho kubelet và tải gì lên cluster.
- Bài [09](../09-troubleshooting-kubeadm-vi.md):
  [*`coredns` bị kẹt ở trạng thái `Pending`*](../09-troubleshooting-kubeadm-vi.md#coredns-bị-kẹt-ở-trạng-thái-pending-coredns-is-stuck-in-the-pending-state),
  [*Lỗi certificate TLS*](../09-troubleshooting-kubeadm-vi.md#lỗi-certificate-tls-tls-certificate-errors).

**Bàn giao:** `~/lab-evidence/8a/b7-*.txt`; postmortem phần 2 (dòng thời gian, mọi chỗ README phải sửa).

**Nghiệm thu:**

- `PASS:` repo đúng commit của B5, `platform/` không đổi; image đúng phiên bản; `preflight` không lỗi.
- `PASS:` `kubeadm init` báo thành công; bằng chứng không chứa token.
- `PASS:` năm hiện vật trên đĩa và bốn hiện vật trong cluster ứng với các phase.
- `PASS:` kubeconfig của user trỏ cluster mới; một node, mang taint control plane.
- `PASS:` patch có trên manifest và mirror Pod của `kube-scheduler`, không lan sang `kube-apiserver`.
- `PASS:` CoreDNS `Pending` trước CNI; sau Calico: CoreDNS `Running`, node `Ready`, IPPool đúng.
- `PASS:` hash join khác hash của cluster cũ.
- `PASS:` mốc `init-xong`, `cni-xong` có trong `moc-thoi-gian.txt`.

**Câu hỏi thiết kế** — trả lời trong postmortem phần 2:

1. Patch được áp vào lúc nào, và ba dòng gate về patch chứng minh ba điều gì khác nhau? Điều gì xảy ra
   với nó ở lần `kubeadm upgrade` đầu tiên nếu không ai nhớ?
2. CoreDNS `Pending` ngay sau init: lỗi hay thiết kế? Vì sao kubeadm không tự cài CNI?
3. Cluster mới có CA mới. Kubeconfig cũ của user và lệnh join của cluster cũ ra sao? Mục nào của bài 09
   mô tả triệu chứng khi quên xóa kubeconfig cũ?
4. Ba thứ bạn không gõ mà cluster mới vẫn có: taint control plane, label `node-role` của control plane,
   ConfigMap `kubelet-config`. Phase nào tạo ra từng thứ?

<details>
<summary>Gợi ý</summary>

- Phase nằm trong ngoặc vuông ở đầu dòng output của `kubeadm init`:
  `grep -oE '^\[[a-z-]+\]' <output> | sort -u`.
- Che token trước khi ghi bằng chứng: `sed -E 's/(--token|--discovery-token-ca-cert-hash|--certificate-key) [^ ]+/\1 <da-che>/g'`.
- Bootstrap token là Secret kiểu `bootstrap.kubernetes.io/token` trong `kube-system`.
- Mirror Pod của scheduler tên `kube-scheduler-<tên node>`; annotation nằm ở `metadata.annotations`.

</details>

<details>
<summary>Script nghiệm thu — bốn khối, chạy đúng lúc ghi ở đầu khối</summary>

Khối 1 — trước `init`, trên `lab-k8s-master`:

```bash
. ~/lab-evidence/8a/env.sh; . "$EV/b5-moc.env"
cd "$REPO"
{ [ "$(git rev-parse HEAD)" = "$COMMIT" ] && git diff --quiet HEAD -- platform; } \
  && echo 'PASS: repo dung commit da bundle o B5, platform/ khong doi'
CFG="$REPO/platform/kubeadm/kubeadm-config.yaml"
sudo kubeadm config images list --config "$CFG" > "$EV/b7-images.txt" 2>&1
grep -q "kube-apiserver:$K8S_VER" "$EV/b7-images.txt" && echo "PASS: image dung $K8S_VER, khong do remote"
sudo kubeadm config images pull --config "$CFG" > "$EV/b7-images-pull.txt" 2>&1
sudo kubeadm init phase preflight --config "$CFG" > "$EV/b7-preflight.txt" 2>&1 \
  && ! grep -q '\[ERROR' "$EV/b7-preflight.txt" && echo 'PASS: preflight khong loi'
```

Chạy `init` theo mục 2 của `platform/README.md`, output vào thư mục làm việc rồi che token:

```bash
sudo kubeadm init --config "$CFG" > "$WK/b7-init.txt" 2>&1; echo "init exit=$?"
chmod 600 "$WK/b7-init.txt"
sed -E 's/(--token|--discovery-token-ca-cert-hash|--certificate-key) [^ ]+/\1 <da-che>/g' "$WK/b7-init.txt" > "$EV/b7-init.txt"
moc init-xong
```

Chép kubeconfig cho user thường bằng đúng bốn lệnh của
[Lab 00 A5.1](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane), rồi khối 2:

```bash
. ~/lab-evidence/8a/env.sh
grep -q 'Your Kubernetes control-plane has initialized successfully' "$EV/b7-init.txt" && echo 'PASS: kubeadm init thanh cong'
grep -qE -- '--token [a-z0-9]{6}\.' "$EV/b7-init.txt" && echo 'FAIL: bang chung chua token' || echo 'PASS: bang chung khong chua token'
grep -oE '^\[[a-z-]+\]' "$EV/b7-init.txt" | sort -u > "$EV/b7-phases.txt"; cat "$EV/b7-phases.txt"

BAD=0
sudo test -f /etc/kubernetes/pki/ca.crt || { echo 'FAIL: certs'; BAD=1; }
{ sudo test -f /etc/kubernetes/admin.conf && sudo test -f /etc/kubernetes/super-admin.conf; } || { echo 'FAIL: kubeconfig'; BAD=1; }
sudo test -f /etc/kubernetes/manifests/etcd.yaml || { echo 'FAIL: etcd'; BAD=1; }
for c in kube-apiserver kube-controller-manager kube-scheduler; do
  sudo test -f "/etc/kubernetes/manifests/$c.yaml" || { echo "FAIL: control-plane $c"; BAD=1; }
done
{ sudo test -f /var/lib/kubelet/config.yaml && \
  { sudo test -f /var/lib/kubelet/kubeadm-flags.env || sudo test -f /var/lib/kubelet/instance-config.yaml; }; } \
  || { echo 'FAIL: kubelet-start'; BAD=1; }
[ "$BAD" -eq 0 ] && echo 'PASS: hien vat tren dia cua certs, kubeconfig, etcd, control-plane, kubelet-start'

[ "$(kubectl config current-context)" = 'kubernetes-admin@kubernetes' ] && echo 'PASS: kubeconfig cua user tro cluster moi'
BAD=0
kubectl -n kube-system get cm kubeadm-config kubelet-config >/dev/null || { echo 'FAIL: upload-config'; BAD=1; }
[ "$(kubectl get nodes -l node-role.kubernetes.io/control-plane --no-headers | wc -l)" -eq 1 ] || { echo 'FAIL: mark-control-plane'; BAD=1; }
[ "$(kubectl -n kube-system get secrets --field-selector type=bootstrap.kubernetes.io/token --no-headers | wc -l)" -ge 1 ] \
  || { echo 'FAIL: bootstrap-token'; BAD=1; }
{ kubectl -n kube-system get deployment coredns >/dev/null && kubectl -n kube-system get daemonset kube-proxy >/dev/null; } \
  || { echo 'FAIL: addon'; BAD=1; }
[ "$BAD" -eq 0 ] && echo 'PASS: hien vat trong cluster cua upload-config, mark-control-plane, bootstrap-token, addon'

{ [ "$(kubectl get nodes --no-headers | wc -l)" -eq 1 ] && \
  kubectl get node "$MASTER" -o jsonpath='{.spec.taints[*].key}' | grep -q 'node-role.kubernetes.io/control-plane'; } \
  && echo 'PASS: mot node, taint control plane co san'

for i in $(seq 1 90); do kubectl -n kube-system get pod "kube-scheduler-$MASTER" >/dev/null 2>&1 && break; sleep 2; done
sudo grep -q 'bookstore.example.com/kubeadm-patch' /etc/kubernetes/manifests/kube-scheduler.yaml \
  && echo 'PASS: patch co trong manifest kube-scheduler tren dia'
kubectl -n kube-system get pod "kube-scheduler-$MASTER" -o yaml | grep -q 'bookstore.example.com/kubeadm-patch' \
  && echo 'PASS: annotation cua patch co tren mirror Pod'
sudo grep -q 'bookstore.example.com/kubeadm-patch' /etc/kubernetes/manifests/kube-apiserver.yaml \
  && echo 'FAIL: patch lan sang kube-apiserver' || echo 'PASS: patch chi cham kube-scheduler'

NEW_HASH="sha256:$(sudo openssl x509 -pubkey -noout -in /etc/kubernetes/pki/ca.crt \
  | openssl pkey -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | awk '{print $NF}')"
OLD_HASH="$(awk -F= '/^ca-cert-hash=/{print $2}' "$EV/b1-join-cong-khai.txt")"
[ "$NEW_HASH" != "$OLD_HASH" ] && echo 'PASS: CA moi — hash join khac cluster cu'
```

Khối 3 — **trước** khi cài Calico:

```bash
P="$(kubectl -n kube-system get pods -l k8s-app=kube-dns -o jsonpath='{range .items[*]}{.status.phase}{"\n"}{end}' | grep -c '^Pending$')"
[ "$P" -ge 1 ] && echo 'PASS: CoreDNS Pending vi chua co Pod network — dung thiet ke'
kubectl -n kube-system describe pods -l k8s-app=kube-dns | grep -A5 '^Events:' > "$EV/b7-coredns-pending.txt"
```

Cài Calico theo mục 3 của `platform/README.md`, rồi khối 4:

```bash
. ~/lab-evidence/8a/env.sh
kubectl -n kube-system rollout status deployment/coredns --timeout=600s
kubectl wait --for=condition=Ready node --all --timeout=600s
[ "$(kubectl -n kube-system get pods -l k8s-app=kube-dns -o jsonpath='{range .items[*]}{.status.phase}{"\n"}{end}' | grep -c '^Running$')" -ge 1 ] \
  && echo 'PASS: CoreDNS Running sau khi co Pod network'
[ "$(kubectl get node "$MASTER" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = 'True' ] \
  && echo 'PASS: control plane node Ready'
[ "$(kubectl get ippools.crd.projectcalico.org default-ipv4-ippool -o jsonpath='{.spec.cidr}')" = "$POD_CIDR" ] \
  && echo 'PASS: IPPool bang podSubnet'
moc cni-xong
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

Không có manifest mới: phần việc là chạy đúng mục 2, mục 3 của `platform/README.md` và đọc hiện vật.

1. Patch áp **trước khi manifest được ghi xuống đĩa** — kubeadm tạo file đã mang sẵn annotation. Ba gate:
   (a) manifest trên đĩa có annotation — patch được áp lúc sinh file; (b) mirror Pod có annotation —
   kubelet đọc đúng file đó, patch có tác dụng thật; (c) `kube-apiserver` không có — `target` trong tên
   file quyết định phạm vi. Ở lần `kubeadm upgrade` đầu tiên, nếu không đưa lại thư mục patch qua
   `UpgradeConfiguration`, manifest sinh lại **mất** annotation.
2. Thiết kế. Bài 02: CoreDNS không khởi động trước khi có mạng; bài 09: kubeadm không phụ thuộc nhà cung
   cấp mạng nào, người quản trị phải tự chọn và cài Pod network. Gặp triệu chứng này trên cluster lạ thì
   việc cần làm là cài CNI, không sửa CoreDNS.
3. Kubeconfig cũ mang certificate của CA cũ: `kubectl` báo `x509: certificate signed by unknown
   authority` — mục *Lỗi certificate TLS* của bài 09; đó là lý do B6 xóa `~/.kube/config`. Lệnh join cũ
   mang hash của CA cũ: node mới từ chối cluster.
4. Taint và label `node-role.kubernetes.io/control-plane`: phase `mark-control-plane`. ConfigMap
   `kubelet-config`: phase `upload-config` (bước con `kubelet`).

</details>

## B8. [BS-8A-08] Join worker và hoàn tất nền

**Loại:** Vận hành.

**Bối cảnh:** control plane đã có mạng. Bookstore cần thêm hai worker, ingress controller và
provisioner — đúng thứ tự mục 4, 5, 6 của `platform/README.md`. Đây là phần quy trình mà Lab 00, Lab
5b, Lab 6a từng làm rải rác qua ba lab; lần này nó phải chạy liền một mạch từ một tài liệu.

**Yêu cầu:**

1. Join hai worker theo mục 4: kiểm đường tới API trước, join sau, lệnh join sinh mới và xóa ngay khi
   xong. Không gõ lệnh nào về kubelet trên worker.
2. Trên mỗi worker: không còn `bootstrap-kubelet.conf`, có `kubelet.conf`; `cgroupDriver` trong
   `/var/lib/kubelet/config.yaml` bằng giá trị trong ConfigMap `kubelet-config`.
3. Ba node `Ready`, mỗi node đúng một `podCIDR`, ba dải khác nhau; `calico-node` đủ ba node.
4. Traefik theo mục 5, local-path-provisioner theo mục 6 — đúng phiên bản trong `versions.env`. Gate
   của mục 7 PASS.
5. NodePort HTTPS mới của Traefik được ghi lại và cập nhật vào `env.sh`; so với cổng cũ.
6. Ghi mốc `join-xong` và `nen-xong`.

**Ràng buộc:** chỉ cài đúng thành phần và phiên bản trong `versions.env`. Bước nào README thiếu hay sai
thì sửa README và ghi vào postmortem — đó là phát hiện giá trị nhất của một cuộc diễn tập.

**Đọc lại:**

- Bài [02](../02-create-cluster-kubeadm-vi.md#join-nodes), *Thêm worker node*; bài
  [215](../215-adding-linux-nodes-vi.md), mục *Thêm node worker Linux*.
- Bài [04](../04-kubelet-integration-vi.md#quy-trình-khi-dùng-kubeadm-join-workflow-when-using-kubeadm-join)
  — `join` tải ConfigMap xuống, TLS Bootstrap, file bootstrap bị xóa.
- [Lab 00 A5.3](LAB-00-MOI-TRUONG-1.35.7.md#a53-join-hai-worker) — verify trước, join sau.

**Bàn giao:** `platform/README.md` (nếu phải sửa); postmortem phần 2; `~/lab-evidence/8a/b8-*.txt`.

**Nghiệm thu:**

- `PASS:` ba node `Ready`, ba `podCIDR` khác nhau, mỗi node một dải; `calico-node` đủ ba node.
- `PASS:` hai worker không còn `bootstrap-kubelet.conf`, có `kubelet.conf`, `cgroupDriver` khớp
  ConfigMap.
- `PASS:` chart Traefik đúng phiên bản; Service `NodePort`; IngressClass `traefik` là mặc định duy nhất.
- `PASS:` provisioner đúng phiên bản và Ready; `local-path` là StorageClass mặc định duy nhất.
- `PASS:` gate ngắn A5.5 trên cluster mới.
- `PASS:` `env.sh` mang NodePort HTTPS mới; mốc `join-xong`, `nen-xong` có mặt.

**Câu hỏi thiết kế** — trả lời trong postmortem phần 2:

1. Bạn không gõ gì về kubelet trên worker, vậy `cgroupDriver` của nó đến từ đâu? Chiều dữ liệu của
   `join` khác `init` ở chỗ nào?
2. Worker vừa join không có `bootstrap-kubelet.conf`. Lỗi hay đúng? File nào chứng minh điều đó?
3. NodePort HTTPS đổi từ cổng cũ sang cổng mới. Ai bị ảnh hưởng, và quy trình phải ghi gì thêm?
4. README có phải sửa gì trong B7, B8 không? Mỗi chỗ sửa nói gì về khoảng cách giữa tài liệu và
   cluster thật?

<details>
<summary>Gợi ý</summary>

- Mục 4 của README chạy lệnh join qua `ssh <worker> "sudo $(cat ~/join.txt)"` — lệnh được thay ở master
  rồi mới gửi đi, không file nào trên worker chứa token.
- `podCIDRs` của từng node: `{range .items[*]}{.spec.podCIDRs[*]}{"\n"}{end}`.
- Cập nhật một dòng trong `env.sh`: `sed -i "s/^HTTPS_NP=.*/HTTPS_NP='<cổng>'/" ~/lab-evidence/8a/env.sh`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã làm xong mục 4, 5, 6 của README</summary>

```bash
. ~/lab-evidence/8a/env.sh; . "$REPO/platform/versions.env"
kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl get nodes -o wide > "$EV/b8-nodes.txt"
N="$(kubectl get nodes --no-headers | wc -l)"
CIDRS="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.podCIDRs[*]}{"\n"}{end}')"
{ [ "$N" -eq 3 ] && [ "$(echo "$CIDRS" | awk 'NF==1' | sort -u | wc -l)" -eq 3 ] && \
  [ "$(kubectl -n kube-system get ds calico-node -o jsonpath='{.status.numberReady}')" -eq 3 ]; } \
  && echo 'PASS: ba node Ready, ba podCIDR khac nhau moi node mot dai, calico-node 3/3'

CM_DRV="$(kubectl -n kube-system get cm kubelet-config -o jsonpath='{.data.kubelet}' | awk '$1=="cgroupDriver:"{print $2}')"
BAD=0
for n in "$W1" "$W2"; do
  ssh "$n" 'sudo test ! -e /etc/kubernetes/bootstrap-kubelet.conf && sudo test -f /etc/kubernetes/kubelet.conf' || BAD=1
  [ "$(ssh "$n" "sudo awk '\$1==\"cgroupDriver:\"{print \$2}' /var/lib/kubelet/config.yaml")" = "$CM_DRV" ] || BAD=1
done
[ "$BAD" -eq 0 ] && echo "PASS: worker da TLS Bootstrap xong, cgroupDriver=$CM_DRV tai tu ConfigMap"

TRF="$(helm -n traefik list --filter '^traefik$' | awk 'NR==2{print $(NF-1)}' | sed 's/^traefik-//')"
DEF_IC="$(kubectl get ingressclass -o jsonpath='{range .items[*]}{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}{"\n"}{end}' | grep -c '^true$')"
{ [ "$TRF" = "$TRAEFIK_CHART" ] && [ "$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.type}')" = 'NodePort' ] && \
  [ "$(kubectl get ingressclass traefik -o jsonpath='{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}')" = 'true' ] && \
  [ "$DEF_IC" -eq 1 ]; } && echo 'PASS: Traefik dung chart, Service NodePort, IngressClass traefik mac dinh duy nhat'
LPP="$(kubectl -n local-path-storage get deploy local-path-provisioner -o jsonpath='{.spec.template.spec.containers[0].image}' | sed 's/.*://')"
DEF_SC="$(kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' | grep -c '^true$')"
{ [ "$LPP" = "$LOCAL_PATH_VERSION" ] && \
  [ "$(kubectl -n local-path-storage get deploy local-path-provisioner -o jsonpath='{.status.readyReplicas}')" -ge 1 ] && \
  [ "$(kubectl get storageclass local-path -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}')" = 'true' ] && \
  [ "$DEF_SC" -eq 1 ]; } && echo 'PASS: provisioner dung phien ban, local-path la StorageClass mac dinh duy nhat'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

NP_MOI="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}')"
echo "NodePort HTTPS cu=$HTTPS_NP moi=$NP_MOI" | tee "$EV/b8-nodeport.txt"
sed -i "s/^HTTPS_NP=.*/HTTPS_NP='$NP_MOI'/" "$EV/env.sh"
. "$EV/env.sh"; [ "$HTTPS_NP" = "$NP_MOI" ] && echo 'PASS: env.sh mang NodePort HTTPS moi'
grep -q '^join-xong ' "$EV/moc-thoi-gian.txt" && grep -q '^nen-xong ' "$EV/moc-thoi-gian.txt" \
  && echo 'PASS: co moc join-xong va nen-xong'
```

**PASS:** sáu dòng `PASS:` của khối kiểm; ba dòng gate ngắn A5.5 như mô tả ở Lab 00 (field selector trả
`No resources found`, CoreDNS đủ replica, `default` không có Pod); hai dòng `PASS:` cuối.

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

Chạy mục 4 của README, gọi `moc join-xong`; chạy mục 5, mục 6, gọi `moc nen-xong`.

1. Từ ConfigMap `kubelet-config` mà `init` đã **tải lên** ở phase `upload-config`. `join` đi chiều ngược
   lại: dùng Bootstrap Token để TLS Bootstrap, lấy quyền **tải xuống** ConfigMap đó rồi ghi ra
   `/var/lib/kubelet/config.yaml` — giá trị `cgroupDriver` bạn viết ở B2 đi trọn một vòng mà không ai
   gõ trên worker.
2. Đúng. `bootstrap-kubelet.conf` chỉ chứa CA và Bootstrap Token; xong TLS Bootstrap kubelet có danh tính
   riêng ở `/etc/kubernetes/kubelet.conf` và kubeadm xóa file bootstrap (bài 04).
3. Mọi người dùng trên máy host gõ URL có cổng cũ. Quy trình phải ghi cổng mới vào nơi người dùng đọc
   được, hoặc ghim cổng (câu 2 của B3). Đây là một thay đổi người dùng **nhìn thấy** sau khôi phục.
4. Ghi từng chỗ sửa kèm lý do. Không có chỗ nào phải sửa cũng là một kết quả — nhưng chỉ đáng tin khi
   mọi bước đều chạy đúng như chép từ README, không bổ sung bằng trí nhớ.

</details>

## B9. [BS-8A-09] Khôi phục Bookstore từ repo và bản dump

**Loại:** Diễn tập sự cố — phần khôi phục.

**Bối cảnh:** cluster đã có đủ nền. Phần ban quản lý quan tâm bắt đầu: Bookstore bán hàng lại, với dữ
liệu của lúc dump. Để giống sự cố thật, giả định `secrets/` trên master cũng đã mất — hệ thống phải
lên được với bí mật **mới**.

**Yêu cầu:**

1. Chuyển mọi file bí mật trong `secrets/` (trừ `.gitkeep`) sang `~/lab-work/8a/secrets-cu/`. Phần triển
   khai của repo — `k8s/`, `scripts/`, `tests/` — đúng commit của B5, không thay đổi.
2. `./scripts/deploy.sh` thoát `0` và sinh bí mật mới; `./tests/smoke.sh` (bản `lab-7b`) PASS trên dữ
   liệu khởi tạo. Ghi mã thoát của hai lệnh vào `~/lab-evidence/8a/b9-ket-qua.txt`, dạng
   `deploy exit=<mã>` và `smoke-truoc-nap exit=<mã>`. Ghi mốc `deploy-xong`.
3. Lấy bản dump **từ máy host** về `~/lab-work/8a/khoi-phuc/`; sha256 bằng giá trị trong `b5-moc.env`.
4. Nạp dump vào `db` sao cho: không thành phần nào ghi vào DB trong lúc nạp; database `bookstore` sau khi
   nạp chỉ chứa dữ liệu của dump, không lẫn dữ liệu khởi tạo; nạp lỗi ở bất kỳ câu nào thì không thay
   đổi nào được giữ lại.
5. Sau khi nạp: số sách và số đơn bằng lúc dump; đơn "trước dump" có, đơn "sau dump" **không**; `api`
   lên đủ số bản sao của repo bằng mật khẩu **mới**; `/api/books` qua Ingress HTTPS trả dữ liệu.
6. `./tests/smoke.sh` PASS; ghi mốc `smoke-pass`; tính RTO (từ `T0-bat-dau-pha` tới `smoke-pass`) và
   RPO; ghi cả hai vào phần *3. Kết quả* của postmortem.
7. Mục 9 của `platform/README.md`: quy trình nạp dump đúng như vừa kiểm chứng.

**Ràng buộc:** không dùng lại `secrets/` cũ; không gắn lại thư mục dữ liệu cũ trên `lab-k8s-worker2`.
Lệnh imperative chỉ dùng trong lúc nạp; xong thì `./scripts/deploy.sh` đưa hệ thống về đúng repo.

**Đọc lại:**

- [Mục 5.4](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm), đoạn cuối — mật khẩu khi dữ liệu được giữ
  lại; [mục 4.3](DU-AN-BOOKSTORE.md#43-dữ-liệu) — role `authenticator`, `web_anon`.
- PostgreSQL 17: [psql](https://www.postgresql.org/docs/17/app-psql.html) — `-1`, `-f -`,
  `ON_ERROR_STOP`; [DROP DATABASE](https://www.postgresql.org/docs/17/sql-dropdatabase.html) —
  `WITH (FORCE)`.

**Bàn giao:** `platform/README.md` (mục 9); postmortem phần 3; `~/lab-evidence/8a/b9-*`.

**Nghiệm thu:**

- `PASS:` `secrets/` hiện tại khác bản cũ (so bằng sha256, không in nội dung); `k8s/`, `scripts/`,
  `tests/` không đổi so với commit của B5.
- `PASS:` `deploy.sh` thoát `0`; smoke `lab-7b` PASS trước khi nạp.
- `PASS:` bản dump lấy từ máy host đúng sha256.
- `PASS:` số sách, số đơn bằng lúc dump; đơn trước dump có, đơn sau dump không.
- `PASS:` `api` đủ bản sao Ready; `/api/books` qua Ingress HTTPS trả ít nhất 5 cuốn.
- `PASS:` smoke PASS sau khi nạp; `moc-thoi-gian.txt` có `deploy-xong`, `smoke-pass`.
- `PASS:` postmortem ghi RTO và RPO; README mục 9 có lệnh nạp.

**Câu hỏi thiết kế** — trả lời trong postmortem phần 3:

1. `secrets/` là mới mà `api` vẫn đăng nhập được sau khi nạp dump — vì sao? Trong trường hợp nào điều đó
   sẽ hỏng, và đoạn cuối mục 5.4 của file dự án nói đúng cho trường hợp nào?
2. Vì sao phải dừng `api` trước khi xóa database? Vì sao nạp trong **một** transaction?
3. RTO của bạn bao nhiêu phút? Giai đoạn nào chiếm nhiều nhất (đọc `moc-thoi-gian.txt`)? Một thay đổi
   nào giảm nó nhiều nhất?
4. Sau khôi phục, người dùng nhận ra được những gì khác so với trước sự cố?

<details>
<summary>Gợi ý</summary>

- `find secrets -type f ! -name .gitkeep -exec mv -t <đích> {} +` giữ `.gitkeep` mà git đang theo dõi.
- Dừng người ghi: thu `api` về 0 bản sao rồi chờ Pod biến mất. `db-backup` có thể chạy theo lịch — nó
  chỉ đọc.
- Xóa và tạo lại database phải chạy từ một database khác (`-d postgres`), từng câu riêng, **không**
  trong transaction.
- `kubectl exec -i` chuyển stdin của bạn vào container; `psql -f -` đọc lệnh từ stdin.
- Thời điểm Job hoàn tất (`DUMP_TIME`) đổi sang giây: `date -d "$DUMP_TIME" +%s`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã nạp dump và smoke PASS</summary>

```bash
. ~/lab-evidence/8a/env.sh; . "$EV/b5-moc.env"
cd "$REPO"
SQL() { kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc "$1"; }
{ test -s secrets/db.env && test -s "$WK/secrets-cu/db.env" && \
  [ "$(sha256sum < secrets/db.env)" != "$(sha256sum < "$WK/secrets-cu/db.env")" ] && \
  git diff --quiet "$COMMIT" -- k8s scripts tests; } && echo 'PASS: bi mat moi khac bi mat cu; phan trien khai khong doi'
grep -q '^deploy exit=0' "$EV/b9-ket-qua.txt" && grep -q '^smoke-truoc-nap exit=0' "$EV/b9-ket-qua.txt" \
  && echo 'PASS: deploy.sh thoat 0, smoke lab-7b PASS truoc khi nap'
[ "$(sha256sum < "$WK/khoi-phuc/$DUMP_FILE" | awk '{print $1}')" = "$DUMP_SHA256" ] \
  && echo 'PASS: ban dump lay tu may host dung sha256'
{ [ "$(SQL 'SELECT count(*) FROM api.books')" = "$N_BOOKS" ] && \
  [ "$(SQL 'SELECT count(*) FROM api.orders')" = "$N_ORDERS" ] && \
  [ "$(SQL "SELECT count(*) FROM api.orders WHERE id = $ID_TRUOC")" = '1' ] && \
  [ "$(SQL "SELECT count(*) FROM api.orders WHERE id = $ID_SAU")" = '0' ]; } \
  && echo "PASS: du lieu bang luc dump ($N_BOOKS sach, $N_ORDERS don); don $ID_SAU sau dump da mat"
W="$(kubectl -n "$NS" get deploy api -o jsonpath='{.spec.replicas}')"
R="$(kubectl -n "$NS" get deploy api -o jsonpath='{.status.readyReplicas}')"
B="$(curl -sk --max-time 5 --resolve "bookstore.lab.local:$HTTPS_NP:$W1_IP" \
     "https://bookstore.lab.local:$HTTPS_NP/api/books" | grep -o '"title"' | wc -l)"
{ [ "${R:-0}" = "$W" ] && [ "$B" -ge 5 ]; } && echo "PASS: api $R/$W Ready, /api/books qua Ingress tra $B cuon"
./tests/smoke.sh > "$EV/b9-smoke-sau-nap.txt" 2>&1 && echo 'PASS: smoke sau khi nap dump'
grep -q '^deploy-xong ' "$EV/moc-thoi-gian.txt" && grep -q '^smoke-pass ' "$EV/moc-thoi-gian.txt" \
  && echo 'PASS: co moc deploy-xong va smoke-pass'
{ grep -q 'RTO' docs/postmortem/8a-dung-lai-cluster.md && grep -q 'RPO' docs/postmortem/8a-dung-lai-cluster.md && \
  grep -q 'ON_ERROR_STOP' platform/README.md; } && echo 'PASS: postmortem ghi RTO, RPO; README co quy trinh nap dump'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Giả lập mất bí mật, triển khai từ repo:

```bash
. ~/lab-evidence/8a/env.sh; . "$EV/b5-moc.env"
cd "$REPO"
mkdir -p "$WK/secrets-cu"
find secrets -type f ! -name .gitkeep -exec mv -t "$WK/secrets-cu/" {} +
ls -A secrets
./scripts/deploy.sh > "$EV/b9-deploy.txt" 2>&1; echo "deploy exit=$?" | tee "$EV/b9-ket-qua.txt"
moc deploy-xong
./tests/smoke.sh > "$EV/b9-smoke-truoc-nap.txt" 2>&1; echo "smoke-truoc-nap exit=$?" | tee -a "$EV/b9-ket-qua.txt"
```

Lấy bản dump từ máy host — PowerShell, cùng `$masterUser`, `$masterHost`, `$dest` của B5 (mở cửa sổ mới
thì đặt lại ba biến đó như phần 2 script nghiệm thu B5):

```powershell
$file = (Select-String -Path "$dest\b5-moc.env" -Pattern '^DUMP_FILE=(.+)$').Matches[0].Groups[1].Value
ssh "${masterUser}@${masterHost}" 'mkdir -p ~/lab-work/8a/khoi-phuc'
scp "$dest\$file" "${masterUser}@${masterHost}:~/lab-work/8a/khoi-phuc/"
```

Nạp dump:

```bash
. ~/lab-evidence/8a/env.sh; . "$EV/b5-moc.env"
F="$WK/khoi-phuc/$DUMP_FILE"
[ "$(sha256sum < "$F" | awk '{print $1}')" = "$DUMP_SHA256" ] && echo 'sha256 dung'

kubectl -n "$NS" scale deployment api --replicas=0
kubectl -n "$NS" wait --for=delete pod -l app.kubernetes.io/component=api --timeout=180s || true
kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d postgres -v ON_ERROR_STOP=1 \
  -c 'DROP DATABASE bookstore WITH (FORCE)' -c 'CREATE DATABASE bookstore'
kubectl -n "$NS" exec -i db-0 -c postgres -- psql -U postgres -d bookstore -v ON_ERROR_STOP=1 -1 -q -f - \
  < "$F" > "$EV/b9-nap.txt" 2>&1; echo "nap exit=$?" | tee -a "$EV/b9-ket-qua.txt"

cd "$REPO" && ./scripts/deploy.sh > "$EV/b9-deploy-sau-nap.txt" 2>&1
./tests/smoke.sh > "$EV/b9-smoke-sau-nap.txt" 2>&1 && moc smoke-pass
```

`deploy.sh` lần hai đưa `api` về số bản sao trong repo và chạy lại `db-migrate` — migration bất biến
kết quả (Lab 4b), unique index đã nằm trong dump, nên không thêm dòng nào.

RTO và RPO:

```bash
. ~/lab-evidence/8a/env.sh; . "$EV/b5-moc.env"
T0="$(awk '$1=="T0-bat-dau-pha"{print $2}' "$EV/moc-thoi-gian.txt")"
T1="$(awk '$1=="smoke-pass"{print $2}' "$EV/moc-thoi-gian.txt" | tail -1)"
TD="$(date -d "$DUMP_TIME" +%s)"
{ echo "RTO = $(( (T1 - T0) / 60 )) phut (T0-bat-dau-pha -> smoke-pass)"
  echo "RPO = $(( (T0 - TD) / 60 )) phut (DUMP_TIME -> T0); mat don id $ID_SAU"
  awk 'NR>1{printf "  %-16s +%d phut\n", $1, ($2-p)/60} {p=$2}' "$EV/moc-thoi-gian.txt"
} | tee "$EV/b9-rto-rpo.txt"
```

Mục 9 của `platform/README.md` — thay dòng "Hoàn thiện ở Lab 8a ticket B9." bằng:

````markdown
## 9. Nạp bản dump

Bản dump là file SQL của `db-backup` (`pg_dump` một database: schema và dữ liệu, **không** có role
hay mật khẩu). Role `authenticator`, `web_anon` do script khởi tạo của DB mới tạo từ `secrets/` hiện
tại, nên nạp dump không làm lệch mật khẩu. Không áp dụng khi gắn lại thư mục dữ liệu cũ: lúc đó phải
dùng đúng `secrets/db.env` cũ.

```bash
F=<đường dẫn bản dump, chép từ máy host>        # kiểm sha256 trước
kubectl -n bookstore scale deployment api --replicas=0
kubectl -n bookstore wait --for=delete pod -l app.kubernetes.io/component=api --timeout=180s
kubectl -n bookstore exec db-0 -c postgres -- psql -U postgres -d postgres -v ON_ERROR_STOP=1 \
  -c 'DROP DATABASE bookstore WITH (FORCE)' -c 'CREATE DATABASE bookstore'
kubectl -n bookstore exec -i db-0 -c postgres -- psql -U postgres -d bookstore -v ON_ERROR_STOP=1 -1 -q -f - < "$F"
cd ~/bookstore && ./scripts/deploy.sh && ./tests/smoke.sh
```

PASS: `psql` thoát `0`; số dòng `api.books`, `api.orders` bằng lúc dump; smoke PASS.
````

Ý chính cho postmortem phần 3:

1. `pg_dump` chỉ dump một database; role là đối tượng toàn cụm, chỉ `pg_dumpall` mới mang theo. Cluster
   mới khởi tạo DB từ `secrets/` mới — script `02-authenticator.sh` đặt mật khẩu `authenticator` bằng
   giá trị mới — rồi dump chỉ thay schema và dữ liệu. Sẽ hỏng khi khôi phục bằng **thư mục dữ liệu** cũ
   (PV `Retain`, đĩa cũ) hoặc bằng `pg_dumpall`: mật khẩu cũ nằm trong dữ liệu, Secret mới không khớp,
   PostgREST coi lỗi xác thực là lỗi không thử lại. Đoạn cuối mục 5.4 đúng cho trường hợp đó, không
   đúng cho dump logic một database.
2. `DROP DATABASE` không chạy được khi còn kết nối; `WITH (FORCE)` cắt chúng, nhưng `api` sẽ nối lại và
   phục vụ dữ liệu dở dang trong lúc nạp. Một transaction: lỗi ở câu nào thì `ROLLBACK` cả khối — không
   có database nửa cũ nửa mới.
3. Đọc bảng `+n phut` trong `b9-rto-rpo.txt`. Giai đoạn chậm thường là kéo image và chờ hội tụ, hoặc thời
   gian người đọc README. Giảm: image đã có sẵn trên node, quy trình viết thành script, ghim NodePort.
4. Certificate TLS tự ký mới (trình duyệt cảnh báo lại); NodePort HTTPS có thể đổi; đơn hàng ghi sau
   lúc dump biến mất.

</details>

## B10. [BS-8A-10] Sprint review: postmortem và việc quản trị còn lại

**Loại:** Điều tra.

**Bối cảnh:** diễn tập xong, đội phải bàn giao hai thứ: một postmortem trả lời ban quản lý bằng con số,
và một ranh giới rõ ràng — quy trình trong `platform/` làm được gì, **không** làm được gì, và phần còn
thiếu nằm ở giai đoạn nào.

**Yêu cầu:**

1. Trên cluster mới: liệt kê bootstrap token; đọc hạn certificate, không cái nào hết hạn.
2. Mục *10. Ngoài phạm vi quy trình này* của `platform/README.md`: đối chiếu **chín** trang con của bài
   214 (ghi số bài) — trang nào vừa làm trong sprint, trang nào thuộc giai đoạn sau và giai đoạn nào;
   cộng backup etcd (nợ lab #8).
3. Hoàn thiện `docs/postmortem/8a-dung-lai-cluster.md`: dòng thời gian từ `moc-thoi-gian.txt`, RPO, RTO,
   mọi chỗ README phải sửa, những gì khác sau khôi phục, hành động cải tiến có người và hạn.
4. `docs/tech-debt.md`: ghi bằng chứng diễn tập cho `TD-10`; thêm `TD-14` — cluster và nền dựng lại bằng
   tay theo `platform/README.md`, không có backup etcd; trả ở giai đoạn 19 cho phần etcd, phần tự động hóa
   chấp nhận.

**Ràng buộc:** không chạy `kubeadm upgrade`, `kubeadm certs renew` — chúng thuộc giai đoạn 17, 18.

**Đọc lại:** bài [214](../214-kubeadm-tasks-vi.md), mục
[*Danh sách các trang trong mục này*](../214-kubeadm-tasks-vi.md#danh-sách-các-trang-trong-mục-này-pages-in-this-section);
bài [02](../02-create-cluster-kubeadm-vi.md#resilience), *Khả năng chống chịu của cluster*;
[sổ nợ lab](README.md#5-sổ-nợ-lab), nợ #7 và #8.

**Bàn giao:** `platform/README.md` (mục 10); postmortem đầy đủ; `docs/tech-debt.md`;
`~/lab-evidence/8a/b10-*.txt`.

**Nghiệm thu:**

- `PASS:` có ít nhất một bootstrap token; không certificate nào hết hạn.
- `PASS:` mục 10 của README nhắc đủ chín số bài 215–223 và etcd.
- `PASS:` postmortem có dòng thời gian, RTO, RPO.
- `PASS:` `docs/tech-debt.md` có `TD-10` và `TD-14`.

**Câu hỏi thiết kế** — trả lời trong postmortem phần 4:

1. Lab này dựng lại được cluster mà không cần backup etcd. Vậy nợ #8 còn cần trả không? Thứ gì sẽ mất
   nếu nó chỉ tồn tại trong cluster, không trong repo hay bản dump?
2. Cấu hình lại cluster đang chạy (bài 220) khác dựng lại ở điểm nào? Khi nào chọn cái nào?
3. Nếu chỉ được làm **một** cải tiến để giảm RTO, đó là gì, và vì sao?

<details>
<summary>Gợi ý</summary>

- `sudo kubeadm token list`; `sudo kubeadm certs check-expiration`.
- Dòng thời gian: mỗi dòng của `moc-thoi-gian.txt` là `<mốc> <epoch> <ISO>`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-evidence/8a/env.sh
cd "$REPO"
sudo kubeadm token list > "$EV/b10-token-list.txt" 2>&1
[ "$(grep -vc '^TOKEN' "$EV/b10-token-list.txt")" -ge 1 ] && echo 'PASS: co bootstrap token'
sudo kubeadm certs check-expiration > "$EV/b10-certs.txt" 2>&1
grep -qi 'invalid\|expired' "$EV/b10-certs.txt" && echo 'FAIL: co certificate het han' \
  || echo 'PASS: khong certificate nao cua cluster moi het han'
BAD=0
for B in 215 216 217 218 219 220 221 222 223 etcd; do grep -q "$B" platform/README.md || { echo "FAIL: README thieu $B"; BAD=1; }; done
[ "$BAD" -eq 0 ] && echo 'PASS: README dinh vi du chin trang con cua bai 214 va etcd'
PM=docs/postmortem/8a-dung-lai-cluster.md
{ grep -q 'T0-bat-dau-pha' "$PM" && grep -q 'smoke-pass' "$PM" && grep -q 'RTO' "$PM" && grep -q 'RPO' "$PM"; } \
  && echo 'PASS: postmortem co dong thoi gian, RTO, RPO'
grep -q 'TD-10' docs/tech-debt.md && grep -q 'TD-14' docs/tech-debt.md && echo 'PASS: tech-debt.md co TD-10, TD-14'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

````bash
cat >> ~/bookstore/platform/README.md <<'EOF'

## 10. Ngoài phạm vi quy trình này

Quy trình trên **tạo** cluster. Việc **sống cùng** cluster (bài 214) nằm ở chỗ khác:

| Trang | Trạng thái |
| --- | --- |
| 215 — Thêm worker node Linux | đã làm ở mục 4; vòng đời node ở giai đoạn 16 |
| 218 — Cấu hình cgroup driver | đã khai ở `kubeadm-config.yaml`; đổi trên cluster đang chạy ở giai đoạn 20 |
| 216 — Thêm worker node Windows; 223 — Nâng cấp node Windows | cần VM Windows (Lab 15) |
| 217 — Đổi package repository; 221 — Nâng cấp cluster; 222 — Nâng cấp node Linux | giai đoạn 17; không chạy `kubeadm upgrade` theo quy trình này |
| 219 — Quản lý certificate | giai đoạn 18 (nợ lab #7) |
| 220 — Cấu hình lại cluster đang chạy | giai đoạn 20 |
| Backup và restore etcd | giai đoạn 19 (nợ lab #8); quy trình này khôi phục bằng repo và bản dump |
EOF
````

Dòng thêm vào `docs/tech-debt.md` (giữ đúng cột của bảng hiện có):

```markdown
| TD-10 | ... | 6a | chấp nhận có điều kiện ở 6b | đã diễn tập ở 8a: dump chép tay ra máy host, khôi phục được | ~/lab-backup/8a (máy host) |
| TD-14 | Cluster và nền dựng lại bằng tay theo platform/README.md; RTO phụ thuộc người làm; không có backup etcd | 8a | giai đoạn 19 (etcd); tự động hóa: chấp nhận | mở | docs/postmortem/8a-dung-lai-cluster.md |
```

Postmortem hoàn chỉnh có bốn phần: *1. Trước sự cố* (B5); *2. Diễn biến* — bảng mỗi mốc của
`moc-thoi-gian.txt` một dòng, thời điểm, phút sau `T0`, ghi chú (PDB chặn, README phải sửa); *3. Kết quả*
— RTO, RPO, những gì khác sau khôi phục; *4. Hành động* — việc, người, hạn. Ý chính cho câu hỏi:

1. Còn cần. Repo và dump khôi phục được **ứng dụng**; etcd chứa **mọi** object của cluster — object tạo
   tay, Secret không có trong repo, trạng thái của controller, mọi namespace không thuộc dự án. Không có
   backup etcd thì những thứ đó mất hẳn, và RTO luôn gồm thời gian dựng nền.
2. Bài 220 sửa cấu hình của cluster **đang chạy**, giữ etcd, workload và danh tính; dựng lại đổi được cả
   thứ không sửa được (CIDR, endpoint) nhưng mất tất cả trạng thái. Đổi một flag thì cấu hình lại; đổi
   quyết định một chiều thì dựng lại.
3. Ví dụ: biến `platform/README.md` thành một script chạy liền mạch có gate — loại bỏ thời gian đọc và gõ,
   là phần lớn nhất của RTO ở cluster nhỏ.

</details>

## B11. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao repo có quy trình dựng lại cluster đã diễn tập, đưa nó và bằng chứng ra máy host,
trả cluster về đúng `03-storage-ready`, và chứng minh repo `lab-8a` dựng được Bookstore trên mốc đó.

> **Thứ tự ở mục này là bắt buộc.** Restore xóa đĩa của cả ba VM — repo, bằng chứng, bundle trên
> master đều mất. Commit, tag, bundle và chép ra máy host **trước** bước 5.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 8a — kiểm thử khói **tích lũy**, chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 8a: hệ thống của sprint 11 trên nền cluster dựng được từ
# platform/ (Calico, Traefik, local-path-provisioner). Kiểm mọi hợp đồng còn hiệu lực tới sprint 12.
set -u
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
HOST=bookstore.lab.local
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
lbl() { kubectl get "$1" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null; }

# 0. Nền cluster (Lab 5b, 6a; quy trình dựng lại ở platform/README.md — Lab 8a)
NODES="$(kubectl get nodes --no-headers 2>/dev/null | wc -l)"
CAL="$(kubectl get daemonset -A --field-selector metadata.name=calico-node -o jsonpath='{.items[0].status.numberReady}' 2>/dev/null)"
[ "${CAL:-0}" -eq "$NODES" ] && pass "calico-node Ready tren $CAL/$NODES node" || fail "calico-node ${CAL:-0}/$NODES node"
[ "$(kubectl get ingressclass traefik -o jsonpath='{.metadata.name}' 2>/dev/null)" = 'traefik' ] \
  && pass 'co IngressClass traefik' || fail 'thieu IngressClass traefik'
[ "$(kubectl get storageclass local-path -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}' 2>/dev/null)" = 'true' ] \
  && pass 'local-path la StorageClass mac dinh' || fail 'local-path khong phai StorageClass mac dinh'
MULTI="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.podCIDRs[*]}{"\n"}{end}' 2>/dev/null | awk 'NF!=1' | wc -l)"
[ "$MULTI" -eq 0 ] && pass 'moi node mot dai podCIDR (single-stack)' || fail "$MULTI node khong dung mot dai podCIDR"

# 1. Namespace và object cấp cluster
[ "$(lbl "namespace/$NS")" = 'bookstore' ] && pass 'namespace bookstore mang label du an' || fail 'namespace bookstore thieu hoac thieu label'
for PC in bookstore-critical bookstore-batch; do
  [ "$(lbl "priorityclass/$PC")" = 'bookstore' ] && pass "co PriorityClass $PC" || fail "thieu PriorityClass $PC"
done
{ [ "$(lbl storageclass/bookstore-retain)" = 'bookstore' ] && \
  [ "$(kubectl get storageclass bookstore-retain -o jsonpath='{.reclaimPolicy}' 2>/dev/null)" = 'Retain' ]; } \
  && pass 'StorageClass bookstore-retain, reclaimPolicy Retain' || fail 'StorageClass bookstore-retain sai hoac thieu'
[ "$(kubectl get nodes -l bookstore.example.com/data=true -o jsonpath='{.items[*].metadata.name}' 2>/dev/null)" = 'lab-k8s-worker2' ] \
  && pass 'chi lab-k8s-worker2 mang label bookstore.example.com/data=true' || fail 'label node du lieu sai'

# 2. Cấu hình, bí mật, chính sách (3b, 3c, 5b, 7b)
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials du hai key' || fail 'Secret db-credentials thieu key'
[ "$(jp secret/bookstore-tls '{.type}')" = 'kubernetes.io/tls' ] && pass 'co Secret TLS bookstore-tls' || fail 'thieu Secret TLS bookstore-tls'
for P in web api; do k get pdb "$P" >/dev/null 2>&1 && pass "co PodDisruptionBudget $P" || fail "thieu PodDisruptionBudget $P"; done
k get resourcequota bookstore-quota >/dev/null 2>&1 && pass 'co ResourceQuota bookstore-quota' || fail 'thieu ResourceQuota bookstore-quota'
k get limitrange bookstore-limits >/dev/null 2>&1 && pass 'co LimitRange bookstore-limits' || fail 'thieu LimitRange bookstore-limits'

# 3. Controller, lập lịch, dữ liệu bền (4a, 4b, 6a, 7a)
[ "$(jp statefulset/db '{.status.readyReplicas}')" = '1' ] && pass 'StatefulSet db 1/1 Ready' || fail 'StatefulSet db chua Ready'
[ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && pass 'db-0 tren lab-k8s-worker2' || fail 'db-0 khong o lab-k8s-worker2'
[ "$(jp pod/db-0 '{.status.qosClass}')" = 'Guaranteed' ] && pass 'db-0 QoS Guaranteed' || fail 'db-0 khong Guaranteed'
PRIO="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
[ -n "$PRIO" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$PRIO" ] && pass 'db-0 dung bookstore-critical' || fail 'db-0 sai priority'
{ [ "$(jp pvc/data-db-0 '{.status.phase}')" = 'Bound' ] && [ "$(jp pvc/data-db-0 '{.spec.storageClassName}')" = 'bookstore-retain' ]; } \
  && pass 'PVC data-db-0 Bound, class bookstore-retain' || fail 'PVC data-db-0 sai hoac chua Bound'
for X in web api; do
  W="$(jp "deployment/$X" '{.spec.replicas}')"; R="$(jp "deployment/$X" '{.status.readyReplicas}')"
  { [ "${W:-0}" -ge 2 ] && [ "${R:-0}" = "$W" ]; } && pass "Deployment $X $R/$W Ready" || fail "Deployment $X ${R:-0}/${W:-?} Ready"
done
WN="$(k get pod -l "$SEL,app.kubernetes.io/component=web" -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' 2>/dev/null | sort -u | wc -l)"
[ "$WN" -ge 2 ] && pass "web trai tren $WN node" || fail 'web khong trai tren hai worker'
ORPHAN="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"
MISS="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container co requests va limits' || fail "container thieu tai nguyen: $(echo $MISS)"

# 4. Service (5a)
[ "$(jp service/web '{.spec.type}/{.spec.ports[0].port}')" = 'ClusterIP/80' ] && pass 'Service web ClusterIP cong 80' || fail 'Service web sai'
[ "$(jp service/api '{.spec.ports[0].port}')" = '3000' ] && pass 'Service api cong 3000' || fail 'Service api sai'
[ "$(jp service/db '{.spec.clusterIP}/{.spec.ports[0].port}')" = 'None/5432' ] && pass 'Service db headless cong 5432' || fail 'Service db sai'
[ "$(jp service/node-agent '{.spec.internalTrafficPolicy}/{.spec.ports[0].port}')" = 'Local/9100' ] \
  && pass 'Service node-agent Local cong 9100' || fail 'Service node-agent sai'

# 5. Dữ liệu và luồng request bên trong cluster
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach, can >= 8"
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
WEB_NODE="$(jp "pod/$WEB" '{.spec.nodeName}')"
get() { k exec "$WEB" -c status-writer -- wget -q -T 5 -O- "$1" 2>/dev/null; }
T="$(get http://api:3000/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 5 ] && pass "Service api tra $T cuon sach" || fail "Service api tra $T cuon sach"
T="$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 5 ] && pass "nginx proxy /api/ tra $T cuon sach" || fail "nginx proxy /api/ tra $T cuon sach"
get http://127.0.0.1:8080/ | grep -q 'Bookstore' && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
get http://127.0.0.1:8080/status.json | grep -Eq "\"pod\" *: *\"$WEB\"" && pass 'status.json dung ten Pod' || fail 'status.json sai'
get http://node-agent:9100/node.json | grep -Eq "\"node\" *: *\"$WEB_NODE\"" \
  && pass 'node-agent qua Service tra dung node cua Pod goi' || fail 'node-agent qua Service sai node'

# 6. NetworkPolicy có hiệu lực: web chỉ được tới cổng http 3000 của api, không tới cổng admin 3001.
#    Đối chứng dương cùng đường đi: cùng Pod web tới cùng IP Pod api ở cổng 3000 được; cổng 3001 đang
#    nghe (readinessProbe của api đi qua cổng admin và Pod Ready). Phủ định chỉ tính khi wget hết giờ
#    hoặc bị từ chối kết nối; lỗi khác là "không kết luận được", không phải PASS.
API_POD="$(k get pod -l "$SEL,app.kubernetes.io/component=api" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
API_IP="$(jp "pod/$API_POD" '{.status.podIP}')"
RP="$(jp "pod/$API_POD" '{.spec.containers[?(@.name=="postgrest")].readinessProbe.httpGet.port}')"
RD="$(jp "pod/$API_POD" '{.status.conditions[?(@.type=="Ready")].status}')"
if [ -z "$API_IP" ]; then fail 'khong tim thay Pod api'
elif ! get "http://${API_IP}:3000/books" | grep -q '"title"'; then
  fail 'doi chung: web khong toi duoc api:3000 — khong ket luan duoc ve cong 3001'
elif [ "$RD" != True ] || { [ "$RP" != admin ] && [ "$RP" != 3001 ]; }; then
  fail "doi chung: chua chac cong admin 3001 dang nghe (probe=$RP, Ready=$RD)"
else
  OUT="$(k exec "$WEB" -c status-writer -- timeout 10 wget -q -T 5 -O /dev/null "http://${API_IP}:3001/live" 2>&1)"; RC=$?
  if [ "$RC" -eq 0 ]; then fail 'web goi duoc cong admin 3001 cua api — NetworkPolicy khong co hieu luc'
  elif echo "$OUT" | grep -Eq 'timed out|Connection refused|exit code 143'; then
    pass 'web bi chan toi cong admin 3001 cua api (NetworkPolicy co hieu luc)'
  else fail "web toi api:3001 loi khac chan mang — khong ket luan duoc: $(echo $OUT)"; fi
fi

# 7. Ingress HTTPS qua Traefik, từ ngoài cluster
{ [ "$(jp ingress/bookstore '{.spec.rules[0].host}')" = "$HOST" ] && [ "$(jp ingress/bookstore '{.spec.tls[0].secretName}')" = 'bookstore-tls' ]; } \
  && pass "Ingress bookstore host $HOST, TLS bookstore-tls" || fail 'Ingress bookstore sai'
NP="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}' 2>/dev/null)"
for IP in $(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' -o jsonpath='{range .items[*]}{.status.addresses[?(@.type=="InternalIP")].address}{" "}{end}'); do
  B="$(curl -sk --max-time 5 --resolve "$HOST:$NP:$IP" "https://$HOST:$NP/api/books" | grep -o '"title"' | wc -l)"
  { [ -n "$NP" ] && curl -sk --max-time 5 --resolve "$HOST:$NP:$IP" "https://$HOST:$NP/" | grep -q 'Bookstore' && [ "$B" -ge 5 ]; } \
    && pass "HTTPS qua Ingress tai $IP: trang chu va $B cuon sach" || fail "HTTPS qua Ingress tai $IP that bai"
done

# 8. CronJob db-backup: chính sách, một lần chạy thử, bản dump trên PVC backup-store
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = 'Forbid' ] && pass 'db-backup Forbid' || fail 'db-backup khong Forbid'
[ "$(jp cronjob/db-backup '{.spec.suspend}')" != 'true' ] && pass 'db-backup khong bi tam ngung' || fail 'db-backup dang suspend'
jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.volumes[*].persistentVolumeClaim.claimName}' | grep -qw backup-store \
  && pass 'db-backup ghi vao PVC backup-store' || fail 'db-backup khong dung PVC backup-store'
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.priorityClassName}')" = 'bookstore-batch' ] \
  && pass 'db-backup dung bookstore-batch' || fail 'db-backup khong dung bookstore-batch'
J="smoke-backup-$(date +%s)"
if k create job "$J" --from=cronjob/db-backup >/dev/null 2>&1 \
   && k wait --for=condition=Complete "job/$J" --timeout=300s >/dev/null 2>&1 \
   && k logs "job/$J" 2>/dev/null | grep -q '^db-backup: OK'; then
  pass 'chay thu db-backup thanh cong'
else
  fail 'chay thu db-backup that bai'
fi
k delete job "$J" --ignore-not-found --wait=true >/dev/null 2>&1
[ "$(jp pvc/backup-store '{.status.phase}')" = 'Bound' ] && pass 'PVC backup-store Bound' || fail 'PVC backup-store chua Bound'

# 9. DaemonSet node-agent
R="$(jp daemonset/node-agent '{.status.numberReady}')"
[ "${R:-0}" -eq "$NODES" ] && pass "node-agent Ready tren $R/$NODES node" || fail "node-agent ${R:-0}/$NODES node"

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

PVC `backup-store` chỉ được kiểm **sau** lần chạy thử `db-backup`: StorageClass dùng
`WaitForFirstConsumer`, nên PVC chỉ `Bound` khi đã có Pod dùng nó — trên một cluster vừa triển khai,
chưa đến lịch backup, nó `Pending` là đúng.

2. Chạy kiểm thử khói trên **cluster vừa dựng lại** — chính hệ thống đã khôi phục ở B9:

```bash
. ~/lab-evidence/8a/env.sh
cd ~/bookstore
./tests/smoke.sh > "$EV/b11-smoke-cluster-dung-lai.txt" 2>&1; echo "smoke exit=$?"
cat "$EV/b11-smoke-cluster-dung-lai.txt"
```

**PASS:** `smoke exit=0`, mọi dòng là `PASS:`. Không teardown rồi triển khai lại trên cluster này: B9 đã
là lần triển khai sạch từ repo, và teardown sẽ xóa đúng dữ liệu vừa khôi phục. Lần triển khai sạch trên
mốc chính nằm ở bước 8.

3. `docs/tech-debt.md` đã có `TD-10` (bằng chứng diễn tập) và `TD-14` từ B10. `platform/README.md` mang
   mọi chỗ sửa của B7, B8.

4. Commit, tag, đóng gói:

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-8a: sprint 12 — dien tap dung lai cluster tu platform/ va ban dump'
git tag lab-8a
git bundle create ~/bookstore-lab-8a.bundle --all && git bundle verify ~/bookstore-lab-8a.bundle
```

Chép bundle và **toàn bộ bằng chứng** ra máy host — PowerShell, cùng `$masterUser`, `$masterHost` của
B5. Không chép `~/lab-work/8a/`: nó chứa lệnh join có token và bí mật cũ của B9, chúng chết cùng cluster
ở bước sau.

```powershell
$dest = "$HOME\lab-backup\8a"
New-Item -ItemType Directory -Force -Path "$dest\evidence" | Out-Null
scp "${masterUser}@${masterHost}:~/bookstore-lab-8a.bundle" $dest
scp -r "${masterUser}@${masterHost}:~/lab-evidence/8a/*" "$dest\evidence"
$n = (Get-ChildItem -Path "$dest\evidence" -File).Count
"so file bang chung: $n"
$ok = (Test-Path "$dest\bookstore-lab-8a.bundle") -and (Test-Path "$dest\evidence\b0-truoc.txt") -and `
      (Test-Path "$dest\evidence\b9-rto-rpo.txt") -and (Test-Path "$dest\evidence\moc-thoi-gian.txt")
if ($ok) { 'PASS: bundle lab-8a va bang chung da nam tren may host' } else { 'FAIL: chua chep du — CHUA duoc restore' }
```

**PASS:** dòng `PASS: bundle lab-8a va bang chung ...`. **Không restore khi còn dòng `FAIL:`** — bằng
chứng và repo mất rồi thì không dựng lại được, và `b0-truoc.txt` là thứ bước 6 dùng để chứng minh cluster
đã về đúng chỗ cũ.

5. Tắt ba VM và restore `03-storage-ready`. Trên `lab-k8s-master`:

```bash
ssh lab-k8s-worker2 'sudo shutdown -h now'; ssh lab-k8s-worker1 'sudo shutdown -h now'; sudo shutdown -h now
```

Trên **máy host**, PowerShell của mục 2:

```powershell
& $vmrun -T ws list
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '03-storage-ready') { "PASS: $f co moc" } else { "FAIL: $f KHONG co moc — dung revert" }
}
foreach ($f in $vmx) { & $vmrun -T ws revertToSnapshot $f '03-storage-ready' }
& $vmrun -T ws start $vmx[0]
& $vmrun -T ws start $vmx[1]
& $vmrun -T ws start $vmx[2]
& $vmrun -T ws list
```

**PASS:** trước khi revert, `vmrun list` không còn VM nào trong `$vmx` và có ba dòng `PASS: ... co moc`;
sau cùng cả ba VM chạy. Không revert riêng một VM — với cluster một control plane, restore là thao tác
trên **cả ba VM cùng một mốc**
([mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường)).

Khi master đã nhận SSH, chép bundle và hồ sơ "trước" ngược lại:

```powershell
scp "$dest\bookstore-lab-8a.bundle" "${masterUser}@${masterHost}:~/"
scp "$dest\evidence\b0-truoc.txt" "${masterUser}@${masterHost}:~/"
```

6. Chứng minh cluster đã về **đúng** `03-storage-ready`. SSH vào `lab-k8s-master` — `env.sh` đã mất cùng
   snapshot. Gate ngắn A5.5, chạy nguyên văn, rồi bốn bằng chứng:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}' > ~/b11-node-ct.txt
grep -A3 'node creationTimestamp' ~/b0-truoc.txt | tail -3 | diff - ~/b11-node-ct.txt \
  && echo 'PASS: ca ba Node object la object cu — creationTimestamp trung ho so truoc lab'
sudo grep -q 'bookstore.example.com/kubeadm-patch' /etc/kubernetes/manifests/kube-scheduler.yaml \
  && echo 'FAIL: con patch cua cluster dung lai' || echo 'PASS: manifest kube-scheduler khong co patch cua cluster dung lai'
kubectl get pods -A --no-headers | grep -qi flannel && echo 'FAIL: thay Flannel — revert nham 01-cluster-ready' \
  || echo 'PASS: khong co CNI cua 01-cluster-ready'
{ kubectl -n kube-system get ds calico-node >/dev/null 2>&1 && kubectl get ingressclass traefik >/dev/null 2>&1 && \
  [ "$(kubectl get storageclass local-path -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}')" = 'true' ]; } \
  && echo 'PASS: du ba thanh phan nen cua 03-storage-ready'
[ "$(kubectl get namespace,priorityclass,storageclass,pv -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 ] \
  && echo 'PASS: moc khong chua object nao cua du an'
test ! -e ~/lab-evidence/8a && echo 'PASS: ~/lab-evidence/8a da mat cung restore — ban luu nam tren may host'
test ! -e ~/lab-work/8a && echo 'PASS: ~/lab-work/8a (lenh join, bi mat cu) da mat cung restore'
```

**PASS:** gate A5.5 đạt; bảy dòng `PASS:` của khối bằng chứng, không dòng `FAIL:`. Mỗi kiểm bắt một
kiểu restore hỏng: `creationTimestamp` khác là đang nhìn một cluster khác; annotation patch còn là chưa
revert; có Flannel là revert nhầm về `01-cluster-ready`; còn object của dự án là revert về một mốc chụp
sai lúc. Gate ngắn fail thì chạy trọn bảy tầng của
[A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) như A5.5 quy định.

7. Thay repo trên đĩa — bản từ snapshot là repo lúc chụp mốc — bằng repo `lab-8a`:

```bash
git -C ~/bookstore describe --tags 2>/dev/null | sed 's/^/repo tu snapshot dang o: /'
[ -d ~/bookstore ] && mv ~/bookstore ~/bookstore-tu-snapshot
git clone -q ~/bookstore-lab-8a.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD | grep -qx 'lab-8a' && echo 'PASS: repo clone tu bundle, HEAD mang tag lab-8a'
{ test -f platform/README.md && test -f platform/versions.env && test -f platform/kubeadm/kubeadm-config.yaml; } \
  && echo 'PASS: platform/ nam trong repo'
rm -rf ~/bookstore-tu-snapshot
mkdir -p ~/lab-evidence/8a
```

8. Dựng Bookstore `lab-8a` trên mốc chính, kiểm thử, rồi gỡ:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/8a/b11-deploy-sau-restore.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/8a/b11-smoke-sau-restore.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/8a/b11-smoke-sau-restore.txt
./scripts/teardown.sh | tee ~/lab-evidence/8a/b11-teardown.txt
```

**PASS:** `deploy exit=0`; `smoke exit=0`, mọi dòng `PASS:`; `teardown: sach`. Smoke fail ở đây nghĩa là
repo `lab-8a` chưa dựng được trên mốc chính: sửa repo, commit, `git tag -f lab-8a`, tạo lại bundle và chép
lại ra máy host — Lab 8b và Lab 9a bắt đầu từ tag này.

9. Gate cuối:

```bash
cd ~/bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-8a' && echo 'PASS: HEAD mang tag lab-8a'
test -s ~/bookstore-lab-8a.bundle && echo 'PASS: co bundle lab-8a tren master'
BAD=0
for F in 'adr/*-node-va-cluster-kubeadm.md' 'adr/*-cau-hinh-kubeadm.md' 'adr/*-quy-trinh-nen-cluster.md' \
         'postmortem/8a-control-plane-vang-mat.md' 'postmortem/8a-dung-lai-cluster.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR va postmortem cua sprint 12'
grep -q 'TD-14' docs/tech-debt.md && echo 'PASS: tech-debt.md co TD-14'
SEL=app.kubernetes.io/part-of=bookstore
{ [ "$(kubectl get namespace,priorityclass,storageclass,pv -l "$SEL" --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  [ "$(kubectl get all -A -l "$SEL" --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  [ "$(kubectl get nodes -l bookstore.example.com/data --no-headers 2>/dev/null | wc -l)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an, label node da go'
rm -f ~/b0-truoc.txt ~/b11-node-ct.txt

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** sáu dòng `PASS:` đầu của gate cuối; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector
trả `No resources found`; CoreDNS đủ replica; namespace `default` không có Pod. Cluster ở đúng
`03-storage-ready`; **không tạo snapshot mới**. Bằng chứng của lab nằm trên máy host ở
`$HOME\lab-backup\8a\`; `~/lab-evidence/8a/` trên master chỉ còn kết quả của bước 8.

---

## 3. Checkpoint 8a

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Đội muốn thêm node bằng cách clone VM. Ba định danh nào bài 01 đòi phải duy nhất, cái nào không sửa
      được từ trong Ubuntu, và Kubernetes cần chúng để làm gì?
- [ ] `swapoff -a` rồi `init` thành công, reboot xong kubelet không lên. Vì sao? Hai lối thoát của bài 01,
      và lối thứ hai có cho workload dùng swap không? B6 kiểm điều đó bằng cách nào?
- [ ] Control plane ở minor N: kubelet N+1 được không, N−1 được không? Bất đối xứng ở đâu? Vì sao binary
      `kubeadm` dùng để join phải khớp binary đã `init`?
- [ ] `kubectl delete` mirror Pod của `kube-apiserver` khác `mv` file manifest thế nào? Trong lúc API vắng
      mặt ở B4, bạn đọc node bằng gì, và Bookstore còn phục vụ được gì — vì sao `/api/books` có thể khác
      trang chủ?
- [ ] Ba thành phần của lệnh join: mỗi cái trả lời câu hỏi gì, cái nào bí mật, tính lại hash bằng gì.
      Sau B7, lệnh join và kubeconfig của cluster cũ ra sao?
- [ ] Ba `kind` trong `platform/kubeadm/kubeadm-config.yaml` của bạn chứa gì và vì sao không đổi chỗ
      được? Patch áp vào lúc nào, cái giá khi nâng cấp là gì? Vì sao phải có file này trong repo dù
      cluster đã có ConfigMap `kubeadm-config`?
- [ ] `init` ghi những file nào cho kubelet, cái gì lên cluster, `join` đi chiều nào? Worker không có
      `bootstrap-kubelet.conf` là lỗi hay đúng? CRI socket trên cluster của bạn nằm ở file nào? Ai thắng
      giữa `kubeadm-flags.env` và `/etc/default/kubelet`?
- [ ] CoreDNS `Pending` ngay sau `init` — lỗi hay thiết kế? Kể thứ tự dựng nền trong `platform/README.md`
      và lý do của từng ranh giới thứ tự. Thao tác tay nào ngoài manifest đã phải ghi lại?
- [ ] Cluster của bạn single-stack: đọc ở ba nguồn nào? Muốn dual-stack phải đổi gì, và vì sao chỉ đổi được
      bằng cách dựng lại?
- [ ] Gỡ cluster: thứ tự ba bước của bài 02; `drain` bị chặn ở đâu, bởi object nào, bạn đi tiếp thế nào và
      vì sao chấp nhận được lúc này; hai thứ `kubeadm reset` không đặt lại; dữ liệu nào vẫn sót trên node.
- [ ] RPO và RTO của bạn là bao nhiêu? Vì sao đơn "sau dump" mất? Nếu sự cố không báo trước, RPO thật là
      bao nhiêu và `TD-10` liên quan thế nào?
- [ ] `secrets/` mới mà `api` vẫn đăng nhập được sau khi nạp dump — vì sao? Khi nào sẽ hỏng? Bằng chứng
      nào ở B11 cho thấy cluster sau restore là cluster cũ chứ không phải cluster bạn vừa dựng?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Nợ lab **#8** (backup và restore etcd) — phát sinh ở giai đoạn 8, **chưa trả**: lab khôi phục dữ liệu
  ứng dụng bằng dump logic và trạng thái cluster bằng repo, không đọc hay ghi etcd; trả ở
  [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa). Nợ lab **#7**
  (vòng đời certificate) — chỉ mới đọc hạn; trả ở giai đoạn 18.
- Nợ kỹ thuật: **`TD-14` mới** (dựng lại bằng tay, không backup etcd); `TD-10` đã có bằng chứng diễn
  tập nhưng vẫn là chép tay; `TD-06` (`db` khởi động bằng root — Lab 9b); `TD-07` (quyền debug — Lab 9a).

### Bài giải thích cuối cùng

Trong vài phút, báo cáo cuộc diễn tập cho ban quản lý như một postmortem nói miệng. Đi theo dòng thời gian
trong `moc-thoi-gian.txt`: lúc nào bản dump được tạo và ra khỏi cluster; lúc nào `T0`; cluster bị gỡ theo
thứ tự nào và vì sao `drain` bị chặn; control plane mọc lại từ file nào, mỗi phase để lại gì; nền được cài
theo thứ tự nào và vì sao; Bookstore lên lại bằng bí mật mới mà vẫn đọc được dữ liệu cũ ra sao; RTO và RPO
cuối cùng. Ở mỗi chặng nói rõ thứ gì đến từ **repo**, thứ gì từ **máy host**, thứ gì từ **cluster cũ** —
và chỉ ra chỗ nào bạn phải dùng trí nhớ thay vì tài liệu, vì đó là chỗ quy trình còn hổng. Kết thúc bằng
câu trả lời: nếu ngày mai mất cluster thật, **không báo trước**, những gì trong câu chuyện này sẽ khác, và
Lab 8b, 8c thay đổi câu chuyện ở điểm nào.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
**Sự cố của chính kubeadm thì tra bài [09](../09-troubleshooting-kubeadm-vi.md)**: mỗi mục `##` của bài là
một triệu chứng; đọc tên mục tới khi thấy hình dạng khớp thứ bạn đang gặp. Bảng dưới chỉ gồm sự cố phát
sinh từ nội dung và cách sắp xếp của lab này.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: HEAD không phải `lab-7b` | Repo trên đĩa là bản lúc chụp `03-storage-ready` | `mv ~/bookstore ~/bookstore-cu`, chép bundle `lab-7b` từ máy host, clone lại |
| B0: `CRI_SOCKET rong` | Đọc nhầm nguồn | Ở baseline, CRI socket nằm trong `/var/lib/kubelet/instance-config.yaml`; khối B0 thử ba nguồn — kiểm `sudo cat` file đó |
| B1: không thấy `kube-controller-manager` trong `ss` | Tên tiến trình bị cắt ở 15 ký tự | Tìm `kube-controller` |
| B1: `ssh` tới chính master đòi mật khẩu | Gọi `ssh lab-k8s-master` thay vì hàm `on` | Dùng `on "$MASTER" '...'` — master chạy tại chỗ |
| B2: `kubeadm config validate` báo trường lạ | Sai `apiVersion`, sai thụt lề, hoặc trường đặt nhầm `kind` | Đối chiếu tham chiếu v1beta4; `kubeadm config print init-defaults` cho khung đúng |
| B3: `curl` tải manifest báo 404 | Phiên bản thiếu tiền tố `v`, hoặc đọc nhầm tag | Lấy đúng tag image đang chạy; đối chiếu bảng A1.4 |
| B4: `kubectl` không trở lại sau khi đưa file về | Sai tên hoặc sai thư mục; container khởi động rồi chết | `ls -l /etc/kubernetes/manifests/`; `sudo crictl ps -a --name kube-apiserver`; `sudo crictl logs <id>`. **Đừng** `kubeadm init` để "sửa" — etcd và pki vẫn còn |
| B4: master không boot lại | Sự cố hypervisor hoặc IP tĩnh | Xử lý ở VMware, không trong cluster. Không xong thì revert cả ba VM về `03-storage-ready` và làm lại từ B0 |
| B4: trang chủ qua Ingress không trả về ở một worker khi master tắt | Pod Traefik và `web` cùng nằm worker kia, quy tắc chuyển tiếp xuyên node hỏng | Bật master, chạy lại tầng 3 và tầng 5 của A5.4 trước khi đi tiếp |
| B5: Pod `lay-dump` bị từ chối khi tạo | ResourceQuota hoặc LimitRange của sprint 11 | `kubectl -n bookstore describe quota,limitrange`; chỉnh `requests`/`limits` của Pod thử nghiệm cho vừa |
| B5: không tìm thấy file dump trong PVC | Đường dẫn trong log khác điểm mount của Pod thử nghiệm | Tìm theo **tên file** (`find /backup -name`); vẫn không có thì lấy file `.sql` mới nhất |
| B6: `drain` treo, log lặp `Cannot evict pod as it would violate the pod's disruption budget` | PodDisruptionBudget `web`/`api` làm đúng việc | Đúng dự kiến; dùng `--timeout` rồi `--disable-eviction` như script, ghi lý do vào postmortem |
| B6: `kubeadm reset` treo | Container runtime không xóa được container | Đúng mục *kubeadm bị treo khi xóa các container được quản lý* của bài 09: khởi động lại containerd rồi chạy lại |
| B6: sau khi flush, `ssh` vào node rớt | Policy `INPUT` không phải `ACCEPT` | Vào console VM, `sudo iptables -P INPUT ACCEPT`; kiểm [A4.4](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập). Gate trong script tồn tại để chặn đúng việc này |
| B6: sau reboot `swap bat lai` hoặc kubelet không lên | `/etc/fstab` còn dòng swap | Sửa theo [A4.1](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites), chạy lại khối 5. Lúc này kubelet được phép `inactive`, chỉ cần `enabled` |
| B7: preflight `[ERROR Port-6443]` hoặc `[ERROR FileAvailable--...]` | `reset` chưa chạy hết hoặc chạy nhầm node | `sudo kubeadm reset -f` đúng node, làm lại khối 3–5 của B6. **Không** dùng `--ignore-preflight-errors` |
| B7: `init` treo ở `[wait-control-plane]` | Một trong ba nguyên nhân của bài 09: mạng, cgroup driver lệch, container control plane crashloop | `sudo crictl ps -a`; `sudo crictl logs <id>`; `journalctl -u kubelet`. B1 đã kiểm cgroup driver — bắt đầu từ hai cái còn lại; `reset` trước khi `init` lại |
| B7: annotation của patch không có trong manifest | `patches.directory` sai đường dẫn, tên file sai quy ước, hoặc `patchtype` không hợp nội dung | `ls` đúng đường dẫn trong file cấu hình; sửa, commit, rồi `reset` và `init` lại — patch chỉ áp lúc sinh manifest |
| B7: CoreDNS vẫn `Pending` sau Calico | Calico chưa chạy, hoặc dải IPPool lệch `podSubnet` | `kubectl -n kube-system get pods -o wide`; so IPPool với `podSubnet` của file cấu hình |
| B7: `kubectl` báo `x509: certificate signed by unknown authority` | Kubeconfig là của cluster cũ | Mục *Lỗi certificate TLS* của bài 09: xóa `~/.kube/config`, chép lại theo A5.1 |
| B8: `join` báo token không hợp lệ hoặc hash sai | Dùng lệnh join của cluster cũ, hoặc token hết hạn | Sinh lệnh mới bằng `kubeadm token create --print-join-command`; B7 đã chứng minh CA mới |
| B8: worker `NotReady` sau join | Pod `calico-node` chưa chạy tới node đó | `kubectl -n kube-system get pods -o wide`; Pod `CrashLoopBackOff` thì đọc `kubectl logs` — bài 09 có mục riêng |
| B8: `helm repo update` lỗi | Không ra được Internet | Sự cố môi trường — kiểm DNS và egress (tầng 4 của A5.4) |
| B8: hai IngressClass mặc định | Chart đã đánh dấu và có class cũ còn sót | Bỏ annotation khỏi class thừa; bài 11 nói nhiều class mặc định thì Ingress không khai class bị từ chối |
| B9: `deploy.sh` treo ở `db` | PVC `data-db-0` `Pending` vì provisioner chưa chạy | Hoàn tất mục 6 của README trước khi triển khai |
| B9: `api` `CrashLoopBackOff`, log lỗi xác thực | Mật khẩu trong Secret không khớp role trong DB — thường do trộn bí mật cũ và mới, hoặc gắn lại dữ liệu cũ | Không gắn thư mục dữ liệu cũ; xóa `db-0` cùng PVC của nó rồi triển khai lại với một bộ bí mật duy nhất |
| B9: nạp dump báo `already exists` | Chưa xóa và tạo lại database | Làm đúng thứ tự: dừng `api` → `DROP DATABASE ... WITH (FORCE)` → `CREATE DATABASE` → nạp |
| B9: smoke fail ở kiểm NetworkPolicy | Calico chưa thực thi trên một node | `kubectl -n kube-system get ds calico-node`; đủ node rồi chạy lại smoke |
| B11: `vmrun revertToSnapshot` báo lỗi | VM chưa tắt hẳn | Chờ *Powered off* rồi revert lại |
| B11: bước 6 fail — `creationTimestamp` khác | Revert nhầm mốc hoặc chỉ một phần VM | Tắt và revert **cả ba** về `03-storage-ready`; chạy lại bước 6 |
| B11: bỏ bước chép ra máy host và bằng chứng đã mất | — | Không khôi phục được. Ghi nhận trong nhật ký học; repo thì lấy bundle `bookstore-8a-truoc-pha` của B5 rồi làm lại phần thay đổi |

---

## 5. Nguồn chính thức

Các phần giải thích ưu tiên snapshot tài liệu Kubernetes v1.35 (`https://v1-35.docs.kubernetes.io/`) để
hành vi, cờ và trường cấu hình khớp minor của cluster lab.

- [Kubernetes v1.35 — Installing kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Kubernetes v1.35 — Creating a cluster with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Kubernetes v1.35 — Customizing components with the kubeadm API](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags/)
- [Kubernetes v1.35 — Configuring each kubelet in your cluster using kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/)
- [Kubernetes v1.35 — Dual-stack support with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/dual-stack-support/)
- [Kubernetes v1.35 — Troubleshooting kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
- [Kubernetes v1.35 — Administration with kubeadm](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/)
- [Kubernetes v1.35 — kubeadm init](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
  — phase, cờ, feature gate `NodeLocalCRISocket`
- [Kubernetes v1.35 — kubeadm config](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-config/)
  — `validate`, `images list`, `print init-defaults`
- [Kubernetes v1.35 — kubeadm join](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)
- [Kubernetes v1.35 — kubeadm reset](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
- [Kubernetes v1.35 — kubeadm token](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)
- [Kubernetes v1.35 — kubeadm certs](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-certs/)
- [Kubernetes v1.35 — kubeadm Configuration (v1beta4)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)
- [Kubernetes v1.35 — Kubelet Configuration (v1beta1)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [Kubernetes v1.35 — Ports and Protocols](https://v1-35.docs.kubernetes.io/docs/reference/networking/ports-and-protocols/)
- [Kubernetes v1.35 — Bootstrap tokens](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/)
- [Kubernetes v1.35 — NodeRestriction](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)
- [Kubernetes v1.35 — Static Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
- [Kubernetes — kubectl drain](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_drain/)
- [Kubernetes — Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [PostgreSQL 17 — pg_dump](https://www.postgresql.org/docs/17/app-pgdump.html)
- [PostgreSQL 17 — pg_dumpall](https://www.postgresql.org/docs/17/app-pg-dumpall.html)
- [PostgreSQL 17 — psql](https://www.postgresql.org/docs/17/app-psql.html)
- [PostgreSQL 17 — DROP DATABASE](https://www.postgresql.org/docs/17/sql-dropdatabase.html)
- [Calico — Install Calico networking and network policy for on-premises deployments](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)
- [Traefik — Helm chart](https://github.com/traefik/traefik-helm-chart)
- [Rancher — local-path-provisioner](https://github.com/rancher/local-path-provisioner)
