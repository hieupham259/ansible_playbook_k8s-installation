# Lab 8a — Dựng cluster bằng kubeadm

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do Lab 6a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** lab **phá cluster rồi restore `03-storage-ready`**. Lab **không tạo mốc mới**.
> Cluster sau lab phải giống hệt lúc vào: CNI thực thi NetworkPolicy, ingress controller và
> StorageClass mặc định do Lab 5b và Lab 6a cài. **Dựng lại bằng tay không ra được trạng thái
> đó**, nên đường về duy nhất là snapshot — restore là một bước bắt buộc của lab, không phải
> phương án dự phòng.
> **Lab trước:** [Lab 7b — Quota và giới hạn tài nguyên](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md),
> cũng trả cluster về `03-storage-ready`.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[Giai đoạn 8 — Dựng cluster bằng kubeadm](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm)
— sáu bài [01](../01-install-kubeadm-vi.md), [02](../02-create-cluster-kubeadm-vi.md),
[03](../03-control-plane-flags-vi.md), [04](../04-kubelet-integration-vi.md),
[05](../05-dual-stack-support-vi.md), [09](../09-troubleshooting-kubeadm-vi.md), cộng bài thực
hành [214](../214-kubeadm-tasks-vi.md). Ba bài HA — [06](../06-ha-topology-vi.md),
[07](../07-setup-ha-etcd-with-kubeadm-vi.md), [08](../08-high-availability-vi.md) — **không**
thuộc lab này; chúng cần bộ VM riêng và nằm ở Lab 8b, Lab 8c.

**Đây là lab đóng lại ngoại lệ của Lab 00.** [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md) đã chạy đúng
những lệnh của bài 01 và bài 02 ở chế độ copy-paste, và tự khai báo trong
[mục "Lab này là ngoại lệ có chủ đích"](LAB-00-MOI-TRUONG-1.35.7.md#lab-này-là-ngoại-lệ-có-chủ-đích)
rằng "Lab 8a phá chính ba VM này và dựng lại có hiểu". Lab 8a giữ đúng phân vai đó: **Lab 00 chỉ
dẫn làm thế nào, lab này giải thích vì sao**. Không mục nào dưới đây chép lại hướng dẫn cài OS,
containerd, kubeadm hay CNI; chỗ nào cần lệnh dựng môi trường thì link về Lab 00.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không
chép lại con số phiên bản nào**. Hai giá trị mà lab cần đến — phiên bản Kubernetes và Pod CIDR —
được điền vào biến ở B0.2, có gate chặn placeholder và gate đối chiếu với chính cluster đang chạy.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm ba lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Ba lệnh riêng của lab 8a: xác nhận đúng mốc 03-storage-ready, chưa node nào bị cordon.
kubectl get storageclass
kubectl get ingressclass
kubectl get nodes -o custom-columns='NODE:.metadata.name,UNSCHEDULABLE:.spec.unschedulable'
```

**PASS:** ba node `Ready`; dòng `PASS: readyz ok` xuất hiện; lệnh field selector trả
`No resources found`; CoreDNS đủ replica `READY`; namespace `default` không có Pod; có **đúng một**
StorageClass mang hậu tố `(default)`; có **đúng một** IngressClass; cột `UNSCHEDULABLE` của cả ba
node là `<none>`.

Nếu `kubectl get storageclass` không có class mặc định hoặc `kubectl get ingressclass` trả
`No resources found`, cluster của bạn **không** ở mốc `03-storage-ready` — restore cả ba VM về mốc
đó trước khi tiếp tục. Lab này kết thúc bằng việc restore về đúng mốc ấy, nên vào lab bằng một
trạng thái khác là tự tạo ra một cluster không ai biết nó phải trông thế nào.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Vì sao mỗi tiền điều kiện của bài [01](../01-install-kubeadm-vi.md) tồn tại — hostname/MAC/
  product_uuid duy nhất, port đang mở, swap tắt, cgroup driver khớp — và chỉ ra được **giá trị
  thật** của từng thứ đó trên ba node của mình, không phải cài lại chúng.
- Ba gói `kubeadm`, `kubelet`, `kubectl` có ba vai trò tách biệt; kubeadm **không** quản lý hai gói
  kia; và quy tắc lệch phiên bản bất đối xứng: kubelet không bao giờ được cao hơn API server.
- Đọc được **từng dòng output** của `kubeadm init` và chỉ ra hiện vật mà mỗi phase để lại trên đĩa
  hoặc trong cluster: `pki/`, `manifests/`, năm file kubeconfig, ConfigMap `kubeadm-config`,
  ConfigMap `kubelet-config`, bootstrap token, CoreDNS và kube-proxy.
- Vì sao control plane chạy dạng **static Pod**: ai tạo chúng, ai sở hữu mirror Pod trong API, và
  điều gì xảy ra khi manifest biến mất.
- Ba thành phần của lệnh `kubeadm join` — endpoint, token, `--discovery-token-ca-cert-hash` — và
  tự tính lại được hash đó từ `pki/ca.crt` bằng `openssl`.
- Taint mặc định của control plane node, và vì sao kubelet **không** tự gán được label
  `node-role.kubernetes.io/*` cho chính nó.
- `ClusterConfiguration` thật của cluster mình đang chạy nằm ở đâu, có **phạm vi toàn cục** nghĩa
  là gì, và vì sao patch là lối thoát duy nhất cho cấu hình khác nhau theo node — chứng minh bằng
  một patch tự viết và đọc lại kết quả trên manifest.
- kubeadm ghi những file nào cho kubelet, file nào được đưa lên cluster để node join sau lấy về,
  và thứ tự ưu tiên của các biến trong `ExecStart` của file drop-in systemd.
- Cluster của mình là **single-stack**, đọc từ ba nguồn độc lập, và chỉ ra đúng chỗ mà quyết định
  họ địa chỉ được chốt một lần lúc `kubeadm init`.
- Dùng bài [09](../09-troubleshooting-kubeadm-vi.md) **đúng công dụng của nó**: gặp triệu chứng
  thật rồi tra ngược về mục tương ứng, không học thuộc danh sách.
- Phá rồi khôi phục ở hai mức: xóa manifest `kube-apiserver` rồi đưa lại, và tắt hẳn control plane
  node rồi chứng minh **data plane vẫn phục vụ** trong lúc API server vắng mặt.
- Gỡ cluster đúng thứ tự `drain` → `kubeadm reset` → `kubectl delete node`, và chỉ ra hai thứ mà
  `kubeadm reset` **không** dọn.
- Dựng lại trọn vẹn một cluster một control plane hai worker bằng `kubeadm init --config`, cài CNI,
  join hai worker, và chạy được một Deployment có Service trên đó.
- Đưa cluster về **đúng** `03-storage-ready` bằng restore, và chứng minh bằng bằng chứng đọc được
  rằng đây là cluster cũ chứ không phải cluster mình vừa dựng.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm giai đoạn 8 | Phần lab kiểm chứng |
| --- | --- |
| [01 — Cài đặt kubeadm](../01-install-kubeadm-vi.md) | B1 — identity ba node, port đang LISTEN và đường tới API, swap tạm và swap vĩnh viễn, cgroup driver hai phía, ba gói và quy tắc lệch phiên bản; B6.4 chứng minh prereq bền vững qua reboot; B7.2 chạy thật phase `preflight` |
| [02 — Tạo một cluster với kubeadm](../02-create-cluster-kubeadm-vi.md) | B2 — giải phẫu cluster kubeadm đã dựng: cây `/etc/kubernetes`, `pki/`, năm kubeconfig, static Pod và mirror Pod, taint control plane, ba thành phần của `kubeadm join`, bảng version skew; B6 — mục *Dọn dẹp* và hai thứ `kubeadm reset` không đụng tới; B7–B8 — dựng lại theo đúng ba mục quyết định của bài: khởi tạo control plane, cài Pod network add-on, join worker |
| [03 — Tùy chỉnh các thành phần với kubeadm API](../03-control-plane-flags-vi.md) | B3 — đọc `ClusterConfiguration` thật trong ConfigMap `kubeadm-config`, so với `kubeadm config print init-defaults`, đối chiếu `extraArgs` với args thật của static Pod, và soạn thư mục patch theo quy ước `target[suffix][+patchtype].extension`; B7.5 — chứng minh patch đã được áp lên manifest sinh ra |
| [04 — Cấu hình từng kubelet bằng kubeadm](../04-kubelet-integration-vi.md) | B4 — năm hiện vật kubelet trên node, so trường của ConfigMap `kubelet-config` với file trên cả ba node, đọc `kubeadm-flags.env`, đọc thứ tự biến trong `ExecStart` của `10-kubeadm.conf`, và chứng minh `bootstrap-kubelet.conf` đã bị xóa sau TLS bootstrap; B8.3 — lặp lại phép đối chiếu đó trên hai worker vừa join |
| [09 — Xử lý sự cố kubeadm](../09-troubleshooting-kubeadm-vi.md) | B5 — hai phép phá của checkpoint giai đoạn 8, mỗi phép kèm bước tra ngược về đúng mục của bài 09; B7.6 — CoreDNS `Pending` trước khi có CNI, đúng mục mà bài 09 nói là *bình thường chứ không phải lỗi*; [mục 4](#4-troubleshooting-của-lab-này) trỏ mọi sự cố kubeadm về bài này thay vì chép lại nội dung |
| [05 — Hỗ trợ dual-stack với kubeadm](../05-dual-stack-support-vi.md) | B9 — đọc họ địa chỉ thật từ ba nguồn độc lập trên cluster mình vừa dựng: flag của apiserver và controller-manager, `.spec.podCIDRs` của Node, `ipFamilies`/`ipFamilyPolicy` của Service; đọc `net.ipv6.conf.all.forwarding`; và chỉ ra đúng chỗ trong file cấu hình B7.1 nơi quyết định single-stack được chốt |
| [214 — Quản trị với kubeadm](../214-kubeadm-tasks-vi.md) | B10 — đây là **trang mục lục**; lab dùng nó đúng công dụng: đối chiếu chín trang con với những gì đã làm ở lab này (`kubeadm token`, `kubeadm certs`, thêm worker node Linux, cgroup driver) và định vị phần còn lại vào đúng giai đoạn sẽ học |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [02](../02-create-cluster-kubeadm-vi.md) — *Thêm control plane node*, và checkpoint "tắt một control plane node mà **API** vẫn phục vụ" | Cần từ ba control plane node cộng load balancer. Topology của chuỗi snapshot chính là **một** control plane, nên B5.2 chỉ chứng minh được vế đúng với topology này: **data plane vẫn phục vụ** trong lúc control plane vắng mặt. Vế API vẫn phục vụ thuộc Lab 8b và Lab 8c với bài [06](../06-ha-topology-vi.md), [07](../07-setup-ha-etcd-with-kubeadm-vi.md), [08](../08-high-availability-vi.md), chạy trên **bộ VM riêng** với mốc tiền tố `8x-` |
| Bài [02](../02-create-cluster-kubeadm-vi.md) — *Chính sách chênh lệch phiên bản* ở dạng thử thật | Muốn thấy skew phải cài đồng thời hai minor version của `kubeadm`/`kubelet` lên node lab, tức phá bộ package đã `apt-mark hold` ở [A4.3](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl). B2.6 kiểm chứng **phần đọc được**: cluster đang thỏa cả ba bảng skew, và mỗi con số đọc từ cluster thật. Việc nâng minor có chỗ đứng riêng ở [giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster) |
| Bài [02](../02-create-cluster-kubeadm-vi.md) — *Chuẩn bị các container image cần thiết*, *Proxy API Server về localhost*, `kubeadm kubeconfig user` | Cái đầu chỉ cần khi node không có Internet egress — ba VM lab có. Cái thứ hai là tiện ích, không đổi cách cluster chạy. Cái thứ ba sinh danh tính mới rồi phải cấp quyền bằng RoleBinding, tức RBAC của [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) và đã có [Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) |
| Bài [03](../03-control-plane-flags-vi.md) — `UpgradeConfiguration` với `apply:` / `node:` | Chỉ có nghĩa khi chạy `kubeadm upgrade`. B3.3 nêu đúng hệ quả mà bài cảnh báo — patch phải được cung cấp lại lúc nâng cấp, nếu không tùy chỉnh mất — nhưng phép thử thật thuộc [giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster) |
| Bài [03](../03-control-plane-flags-vi.md) — `dns.disabled: true` và tùy chỉnh CoreDNS bằng `corednsdeployment` | Thay hoặc tắt CoreDNS là việc vận hành DNS, thuộc [giai đoạn 21](../00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy). B3.3 chỉ dùng target `kube-scheduler` cho patch mẫu vì nó vô hại và kiểm chứng được ngay trên đĩa |
| Bài [03](../03-control-plane-flags-vi.md) — nội dung các flag ví dụ: `enable-admission-plugins`, `audit-log-path`, `election-timeout` | Chúng là cấu hình bảo mật, audit và tinh chỉnh etcd. Lab kiểm chứng **cơ chế** `extraArgs`/`extraVolumes`/patch chứ không kiểm chứng giá trị của từng flag; audit thuộc [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu), vận hành etcd thuộc [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa) |
| Bài [04](../04-kubelet-integration-vi.md) — cơ chế TLS Bootstrap sinh ra "thông tin xác thực duy nhất" | Cần hiểu cách cấp và xoay certificate, tức [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ). B4.5 và B8.3 kiểm chứng **kết quả** của TLS Bootstrap: `bootstrap-kubelet.conf` biến mất và `kubelet.conf` xuất hiện |
| Bài [05](../05-dual-stack-support-vi.md) — dựng cluster dual-stack thật | Mạng lab của [A1.2](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) chỉ có IPv4, và bài nói rõ `kubeadm upgrade` **không hỗ trợ đổi** cluster CIDR hay Service CIDR về sau — nên đây là quyết định một chiều không thử lại được trên cùng bộ VM. B9 kiểm chứng phần đọc được và phần **single-stack có chủ đích** mà chính bài mô tả ở mục *Tạo một cluster single-stack* |
| Bài [09](../09-troubleshooting-kubeadm-vi.md) — phần lớn các mục | Chúng là **di sản lịch sử**: RBAC v1.18/v1.17, Docker 1.13.1.84 trên CentOS 7, `/usr` read-only của Flatcar, NIC mặc định trong Vagrant, cloud-controller-manager, lỗi nâng cấp v1.28.0–v1.28.2. Cluster lab không thể tái tạo chúng, và cố dựng lại là học sai cách dùng một tài liệu tra cứu |
| Bài [214](../214-kubeadm-tasks-vi.md) — chín trang con | Mỗi trang có chỗ đứng riêng: [215](../215-adding-linux-nodes-vi.md) làm ở B8 và ở [giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node); [216](../216-adding-windows-nodes-vi.md) cần VM Windows của Lab 15; [217](../217-change-package-repository-vi.md), [221](../221-kubeadm-upgrade-vi.md), [222](../222-upgrading-linux-nodes-vi.md), [223](../223-upgrading-windows-nodes-vi.md) thuộc [giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster); [219](../219-kubeadm-certs-vi.md) thuộc [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ); [218](../218-configure-cgroup-driver-vi.md) và [220](../220-kubeadm-reconfigure-vi.md) thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |

**[Nợ #8](README.md#5-sổ-nợ-lab) phát sinh ở lab này và không được trả ở đây.** Lab dựng được
cluster nhưng **không** có quy trình backup và restore etcd bằng `etcdctl`; mọi lần khôi phục
trong lab đều dựa vào snapshot VM. Nợ được trả ở
[giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa) với bài
[197](../197-configure-upgrade-etcd-vi.md). Lab cũng **không trả** [nợ #7](README.md#5-sổ-nợ-lab)
— quản lý vòng đời certificate — dù B2.2 và B10 có đọc `kubeadm certs check-expiration`; đọc hạn
không phải là quản lý vòng đời.

### 1.2. Thời lượng

4 giờ, tính từ lúc gate mở đầu đã PASS. Phân bổ tham khảo: khoảng một giờ cho B1–B4 (đọc cluster
đang chạy), nửa giờ cho B5, hai giờ cho B6–B8 (phá và dựng lại), nửa giờ cho B9–B11.

Ba chỗ phải chờ mà **không có con số cam kết**: `kubeadm init` kéo image và chờ control plane lên
(phụ thuộc tốc độ mạng và ổ đĩa), node chuyển sang `Ready` sau khi cài CNI, và ba VM boot lại sau
restore. Mọi bước chờ trong lab viết dưới dạng `kubectl wait --timeout` hoặc vòng lặp có điều kiện
dừng, không phải một khoảng thời gian cố định.

Lab này **không chia nhỏ được**. Từ B6 trở đi cluster không còn phục vụ được gì cho tới khi B8
kết thúc, và trạng thái giữa chừng không có gate nào để quay lại ngoài snapshot. Đừng mở B6 nếu
không còn đủ thời gian chạy hết tới B11.

---

## 2. Quy ước và an toàn

> **CẢNH BÁO — LAB NÀY PHÁ CLUSTER CÓ CHỦ ĐÍCH.**
> Từ [B5](#b5-phá-có-kiểm-soát-trên-control-plane-và-khôi-phục) trở đi, lab xóa manifest của
> `kube-apiserver`, tắt control plane node, rồi ở [B6](#b6-gỡ-cluster-bằng-kubeadm-reset-đúng-thứ-tự)
> chạy `kubeadm reset` trên **cả ba node**. Sau B6 **không còn cluster nào** cho tới khi B8 dựng
> xong cái mới, và cái mới đó **không** phải mốc `03-storage-ready`: nó thiếu CNI thực thi
> NetworkPolicy, thiếu ingress controller, thiếu StorageClass mặc định.
> **Đường về duy nhất là restore snapshot ở [B11](#b11-xuất-bằng-chứng-restore-03-storage-ready-và-gate-cuối).**
> Không có snapshot thì không có lab.

**Gate bắt buộc trước khi chạy bất kỳ lệnh nào của phần B.** Chạy trên **máy host**, PowerShell.
Không có đủ ba dòng `PASS:` thì **dừng tại đây** — không mở B0:

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

**PASS:** đúng **ba** dòng `PASS:`, không dòng `FAIL:` nào. `-ccontains` so sánh phân biệt hoa
thường nên gate này bắt được cả lỗi gõ sai tên snapshot. Đường dẫn `.vmx` ở trên theo máy host
đang dùng; sửa lại nếu VM nằm chỗ khác.

**Giữ nguyên cửa sổ PowerShell này tới hết lab.** B5.2, B7 và B11 dùng lại hai biến `$vmrun` và
`$vmx`; mở cửa sổ mới giữa chừng là phải đặt lại cả hai.

Nếu một VM thiếu mốc `03-storage-ready`, **không** chụp bù bằng trạng thái hiện tại: một mốc chụp
sai thời điểm sẽ đưa sai lệch vào mọi lab sau. Dựng lại mốc bằng cách chạy lại
[Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md) từ `02-net-ready`.

Các quy ước còn lại:

- **Ngoại lệ về fault injection.** [Mục 6 của README](README.md#6-quy-ước-chung-trong-mọi-lab) quy
  định fault injection chỉ chạy trên `lab-k8s-worker2`. Lab 8a là lab duy nhất phá **cả ba node**,
  vì nội dung bài học chính là dựng lại cluster. Điều kiện đánh đổi đã nằm ở gate trên: mốc
  `03-storage-ready` tồn tại trên cả ba VM và lab kết thúc bằng restore. Đừng cho lab khác viện
  dẫn ngoại lệ này.
- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi rõ
  node khác**. Lệnh cần đọc file trên worker chạy qua `ssh` từ master, đúng cách
  [Lab 12](LAB-12-VAN-HANH-VONG-DOI-NODE.md) đã dùng.
- Các khối PowerShell chạy trên **máy host Windows**, không phải trên VM; chúng luôn được mở đầu
  bằng dòng "Chạy trên máy host".
- **Mở đủ ba terminal SSH** — master, worker 1, worker 2 — như
  [mục 3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#3-quy-ước-và-an-toàn) yêu cầu. B6 và B8 chạy lệnh
  trên chính worker, không qua master.
- **Phiên shell trên master sẽ chết ít nhất hai lần** trong lab này: khi B5.2 tắt master và khi
  B6.4 reboot ba node. Vì vậy B0.2 ghi mọi biến vào `~/lab-evidence/8a/vars.sh`, và mỗi lần kết
  nối lại phải chạy `. ~/lab-evidence/8a/vars.sh` trước khi đi tiếp. Chỗ nào cần, lab nhắc lại
  lệnh này.
- **B1 tới B4 chỉ ĐỌC.** Không sửa file nào dưới `/etc/kubernetes`, `/var/lib/kubelet`,
  `/etc/containerd`. Việc phá bắt đầu ở B5 và luôn được nói rõ bằng một dòng in đậm.
- Manifest tạm ghi vào `~/lab-work/8a/`; bằng chứng ghi vào `~/lab-evidence/8a/`. **Cả hai thư mục
  nằm trên đĩa của VM master và sẽ biến mất khi restore ở B11**, nên B11.2 chép bằng chứng ra máy
  host trước. Đừng bỏ bước đó.
- **Không ghi token, private key hay nội dung kubeconfig vào evidence.** Lab đọc `Subject` của
  certificate và hash công khai của CA — không đọc và không lưu phần bí mật. Bài
  [02](../02-create-cluster-kubeadm-vi.md) nói rõ: `admin.conf`, `super-admin.conf` và token trong
  lệnh join đều là bí mật.
- **Không cài thêm phần mềm nào lên node.** Mọi lệnh trong lab dùng công cụ đã có từ
  [A4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a4-chuẩn-bị-os-và-container-runtime): `kubeadm`,
  `kubectl`, `crictl`, `openssl`, `ss`, `iptables`, và `/dev/tcp` của bash. Chỗ nào bài gốc gợi ý
  một công cụ chưa có — ví dụ `nc` hoặc `ipvsadm` — lab nói rõ dùng gì thay thế và vì sao.
- Image dùng cho workload test là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), đúng image mà
  [A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) dùng.
- **Mọi con số của cluster đều đọc từ cluster thật** — IP node, endpoint control plane, CRI socket,
  Pod CIDR, phiên bản. Không con số nào của cluster được viết cứng vào gate.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức giai đoạn 8

## B0. Chuẩn bị workspace, biến khóa và hồ sơ của `03-storage-ready`

**Mục đích:** dựng chỗ làm việc, đọc từ cluster đang chạy mọi giá trị mà B7 sẽ cần để dựng lại, và
chụp một hồ sơ đầy đủ của mốc `03-storage-ready` để B11 chứng minh cluster đã về đúng chỗ cũ.

### B0.1. Workspace và tên node

```bash
mkdir -p ~/lab-work/8a/patches ~/lab-evidence/8a
WK=~/lab-work/8a
EV=~/lab-evidence/8a

kubectl config current-context

MASTER='lab-k8s-master'
W1='lab-k8s-worker1'
W2='lab-k8s-worker2'
NODES="$MASTER $W1 $W2"

CP_N="$(kubectl get nodes -l 'node-role.kubernetes.io/control-plane' --no-headers | wc -l)"
WK_N="$(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' --no-headers | wc -l)"
echo "control plane=$CP_N | worker=$WK_N"

test "$CP_N" -eq 1 && test "$WK_N" -eq 2 \
  && echo 'PASS: dung topology mot control plane hai worker'
```

**Ý nghĩa:** con số `1` và `2` này là dữ kiện của B5.2 (mất control plane nghĩa là mất gì) và của
B8 (join lại bao nhiêu node). Topology khác thì gần như mọi gate sau đều sai.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B0.2. Bảy giá trị mà B7 sẽ cần, đọc từ chính cluster đang chạy

Không giá trị nào dưới đây được gõ tay. Tất cả đọc từ cluster mà bạn sắp phá — đó là cách duy nhất
để cluster dựng lại có cùng danh tính mạng với cluster cũ:

```bash
MASTER_IP="$(kubectl get node "$MASTER" \
  -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
W1_IP="$(kubectl get node "$W1" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
W2_IP="$(kubectl get node "$W2" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"

SERVER="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
CP_ENDPOINT="${SERVER#https://}"
API_PORT="${CP_ENDPOINT##*:}"

CRI_SOCKET="$(kubectl get node "$MASTER" \
  -o jsonpath='{.metadata.annotations.kubeadm\.alpha\.kubernetes\.io/cri-socket}')"

NODE_CT="$(kubectl get node "$MASTER" -o jsonpath='{.metadata.creationTimestamp}')"

printf 'MASTER_IP=%s\nW1_IP=%s\nW2_IP=%s\nCP_ENDPOINT=%s\nAPI_PORT=%s\nCRI_SOCKET=%s\nNODE_CT=%s\n' \
  "$MASTER_IP" "$W1_IP" "$W2_IP" "$CP_ENDPOINT" "$API_PORT" "$CRI_SOCKET" "$NODE_CT"

test -n "$MASTER_IP" && test -n "$W1_IP" && test -n "$W2_IP" \
  && test -n "$CP_ENDPOINT" && test -n "$API_PORT" && test -n "$CRI_SOCKET" \
  && test -n "$NODE_CT" \
  && echo 'PASS: doc du bay gia tri tu cluster dang chay'
```

**Ý nghĩa của từng giá trị:**

- `MASTER_IP` sẽ thành `localAPIEndpoint.advertiseAddress` ở B7.1 — tương đương cờ
  `--apiserver-advertise-address` mà [A5.1](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane)
  truyền. Bài [02](../02-create-cluster-kubeadm-vi.md) nhấn: cờ này chỉ đặt địa chỉ quảng bá **của
  riêng node này**.
- `CP_ENDPOINT` sẽ thành `controlPlaneEndpoint` — endpoint **dùng chung cho mọi control plane
  node**. Đây là quyết định một chiều: bài nói thẳng rằng chuyển một cluster tạo **không có**
  `--control-plane-endpoint` thành HA **không được kubeadm hỗ trợ**. Lab 00 đã đặt sẵn nó dưới
  dạng tên DNS chứ không phải IP, nên cluster này còn đường lên HA.
- `CRI_SOCKET` là thứ kubeadm **tự phát hiện** bằng cách quét danh sách endpoint đã biết của bài
  [01](../01-install-kubeadm-vi.md). Node lab chỉ có một runtime nên kubeadm không hỏi; ta vẫn khai
  tường minh ở B7.1 để phép dựng lại không phụ thuộc vào việc dò tìm.
- `NODE_CT` là thời điểm Node object của master được tạo lần đầu. B11.4 dùng nó làm bằng chứng
  rằng cluster sau restore là **cluster cũ**, không phải cluster bạn vừa dựng ở B7.

**PASS:** dòng `PASS:` của bước này xuất hiện và bảy giá trị in ra đều khác rỗng.

Hai giá trị còn lại **không đọc được từ cluster ở dạng đủ tin cậy**, nên bạn điền tay từ bảng
baseline. Mở [bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), đọc dòng
*Kubernetes control plane* và dòng *Pod CIDR*, rồi điền vào hai biến dưới đây. **Không chép con số
vào file lab**:

```bash
export K8S_BASELINE='<phien ban Kubernetes control plane trong bang A1.3, dang vX.Y.Z>'
export POD_CIDR='<Pod CIDR trong bang A1.3, dang a.b.c.d/nn>'

case "$K8S_BASELINE" in
  ''|*'<'*) echo 'FAIL: chua dien gia tri that vao K8S_BASELINE — quay lai bang A1.3' ;;
  v*)       echo "OK: K8S_BASELINE = $K8S_BASELINE" ;;
  *)        echo 'FAIL: gia tri phai bat dau bang chu v' ;;
esac

case "$POD_CIDR" in
  ''|*'<'*) echo 'FAIL: chua dien gia tri that vao POD_CIDR — quay lai bang A1.3' ;;
  */*)      echo "OK: POD_CIDR = $POD_CIDR" ;;
  *)        echo 'FAIL: gia tri phai co dang CIDR' ;;
esac
```

Bây giờ đối chiếu hai giá trị vừa điền với chính cluster đang chạy. Điền sai thì cluster dựng lại ở
B7 sẽ lệch baseline mà không ai phát hiện:

```bash
SRV_VER="$(kubectl get --raw /version | grep -o '"gitVersion":"[^"]*"' | cut -d'"' -f4)"
KUBEADM_VER="$(kubeadm version -o short)"
CLUSTER_CIDR="$(sudo grep -oE '\-\-cluster-cidr=[^[:space:]]+' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml | cut -d= -f2)"
echo "gitVersion=$SRV_VER | kubeadm=$KUBEADM_VER | cluster-cidr=$CLUSTER_CIDR"

test "$SRV_VER" = "$K8S_BASELINE" && test "$KUBEADM_VER" = "$K8S_BASELINE" \
  && echo 'PASS: K8S_BASELINE khop ca API server lan binary kubeadm'
test "$CLUSTER_CIDR" = "$POD_CIDR" \
  && echo 'PASS: POD_CIDR khop --cluster-cidr cua kube-controller-manager dang chay'
```

**PASS:** hai dòng `OK:` chứa con số thật, và hai dòng `PASS:` của bước này xuất hiện. Nếu
`CLUSTER_CIDR` khác `POD_CIDR`, **dừng lại**: hoặc bạn đọc nhầm dòng trong bảng A1.3, hoặc cluster
đã lệch baseline — dựng lại ở B7 bằng giá trị sai sẽ tạo ra một cluster mà Pod không định tuyến được.

### B0.3. Ghi mọi biến ra file để sống sót qua hai lần mất phiên shell

```bash
cat > "$EV/vars.sh" <<EOF
MASTER='$MASTER'
W1='$W1'
W2='$W2'
NODES='$NODES'
MASTER_IP='$MASTER_IP'
W1_IP='$W1_IP'
W2_IP='$W2_IP'
CP_ENDPOINT='$CP_ENDPOINT'
API_PORT='$API_PORT'
CRI_SOCKET='$CRI_SOCKET'
NODE_CT='$NODE_CT'
K8S_BASELINE='$K8S_BASELINE'
POD_CIDR='$POD_CIDR'
WK=$HOME/lab-work/8a
EV=$HOME/lab-evidence/8a
EOF

chmod 600 "$EV/vars.sh"
test "$(grep -c '=' "$EV/vars.sh")" -eq 15 \
  && echo 'PASS: ghi du 15 bien vao vars.sh'
```

**Ý nghĩa:** heredoc ở đây **không** đặt `EOF` trong nháy đơn, nên Bash thay biến bằng giá trị
trước khi ghi — file nhận được là hằng số, không phải tham chiếu. Mỗi lần SSH lại vào master trong
lab này, chạy:

```bash
. ~/lab-evidence/8a/vars.sh
```

**PASS:** dòng `PASS: ghi du 15 bien vao vars.sh` xuất hiện.

### B0.4. Hồ sơ đầy đủ của mốc `03-storage-ready`

```bash
{
  echo "=== $(date -Is) — truoc Lab 8a, moc 03-storage-ready ==="
  echo '--- nodes'
  kubectl get nodes -o wide
  echo '--- node creationTimestamp'
  kubectl get nodes \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}'
  echo '--- pods'
  kubectl get pods -A -o wide
  echo '--- storageclass'
  kubectl get storageclass
  echo '--- ingressclass'
  kubectl get ingressclass
  echo '--- namespaces'
  kubectl get namespaces
  echo '--- kube-proxy mode'
  kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' \
    | grep -E '^[[:space:]]*mode:'
} | tee "$EV/b0-truoc.txt"

test -s "$EV/b0-truoc.txt" \
  && grep -q "$W2" "$EV/b0-truoc.txt" \
  && echo 'PASS: da ghi ho so cua moc 03-storage-ready'
```

**Ý nghĩa:** dòng `mode:` của ConfigMap `kube-proxy` là dữ kiện của B6.2. Bài
[02](../02-create-cluster-kubeadm-vi.md) nói `kubeadm reset` **không** đặt lại quy tắc iptables và
**không** đặt lại bảng IPVS, rồi đưa hai lệnh dọn khác nhau cho hai chế độ. Đọc chế độ thật ở đây
để B6.2 biết phải dùng lệnh nào — và biết `ipvsadm` có cần thiết không, vì node lab không cài nó và
lab **không** được cài thêm phần mềm.

**PASS:** dòng `PASS: da ghi ho so cua moc 03-storage-ready` xuất hiện.

---

## B1. Bài 01 — điều kiện để một máy trở thành node kubeadm

**Mục đích:** bài [01](../01-install-kubeadm-vi.md) là danh sách tiền điều kiện, và
[A4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a4-chuẩn-bị-os-và-container-runtime) đã thỏa từng cái
một cách copy-paste. Ở đây bạn **không cài lại gì**: bạn đọc trạng thái thật và trả lời câu hỏi
"nếu thiếu cái này thì hỏng ở đâu".

### B1.1. Ba định danh phải khác nhau giữa các node

Bài liệt kê ba thứ phải là duy nhất cho mỗi node: hostname, địa chỉ MAC và `product_uuid`.

```bash
. ~/lab-evidence/8a/vars.sh

for n in $NODES; do
  H="$(ssh "$n" 'hostnamectl --static')"
  U="$(ssh "$n" 'sudo cat /sys/class/dmi/id/product_uuid')"
  IFACE="$(ssh "$n" "ip route | awk '/default/ {print \$5; exit}'")"
  M="$(ssh "$n" "ip -br link show $IFACE | awk '{print \$3}'")"
  echo "$n hostname=$H uuid=$U iface=$IFACE mac=$M"
done | tee "$EV/b1-identity.txt"

# Cot 2 = hostname, cot 3 = product_uuid, cot 5 = MAC.
for f in 2 3 5; do
  N_UNIQ="$(awk -v f="$f" '{print $f}' "$EV/b1-identity.txt" | sort -u | wc -l)"
  echo "cot $f co $N_UNIQ gia tri khac nhau"
  test "$N_UNIQ" -eq 3 || echo "FAIL: cot $f bi trung giua cac node"
done

test "$(awk '{print $2}' "$EV/b1-identity.txt" | sort -u | wc -l)" -eq 3 \
  && test "$(awk '{print $3}' "$EV/b1-identity.txt" | sort -u | wc -l)" -eq 3 \
  && test "$(awk '{print $5}' "$EV/b1-identity.txt" | sort -u | wc -l)" -eq 3 \
  && echo 'PASS: hostname, product_uuid va MAC deu duy nhat tren ba node'
```

**Ý nghĩa:** bài nói Kubernetes dùng những giá trị này để **định danh duy nhất node trong
cluster**, và nếu chúng trùng thì quá trình cài đặt **có thể thất bại**. Đây chính là lý do
[A2.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a22-vm-được-tạo-bằng-full-clone) bắt chuẩn hóa
identity trên VM clone **trước** khi đặt hostname, và lý do
[A2.3](LAB-00-MOI-TRUONG-1.35.7.md#a23-đặt-hostname-etchosts-và-kiểm-tra-identity) dừng lại nếu
`product_uuid` hoặc MAC trùng: hai giá trị đó do hypervisor sinh, không sửa được từ bên trong
Ubuntu. Ở B8 bạn sẽ join lại hai worker vào một cluster mới, và `product_uuid` trùng nhau sẽ hiện
ra đúng lúc đó.

**PASS:** dòng `PASS: hostname, product_uuid va MAC deu duy nhat tren ba node` xuất hiện, không có
dòng `FAIL:`.

### B1.2. Port cần mở — đọc ai đang lắng nghe, và thử đường thật tới API

Bài yêu cầu "một số port nhất định phải được mở" và gợi ý dùng `netcat`. Node lab không cài
`netcat`, và lab **không cài thêm phần mềm**; dùng `/dev/tcp` của bash, đúng cách
[A5.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a53-join-hai-worker) đã dùng.

Trước hết, đọc **ai** đang lắng nghe trên control plane node:

```bash
sudo ss -lntp | tee "$EV/b1-ports-master.txt"

for p in kube-apiserver etcd kubelet kube-controller kube-scheduler; do
  grep -q "$p" "$EV/b1-ports-master.txt" \
    && echo "PASS: $p dang LISTEN tren $MASTER" \
    || echo "FAIL: khong thay $p LISTEN tren $MASTER"
done
```

Rồi trên **worker**, kiểm hai tiến trình phải có mặt ở mọi node:

```bash
for n in "$W1" "$W2"; do
  ssh "$n" 'sudo ss -lntp' > "$EV/b1-ports-$n.txt"
  grep -q 'kubelet'    "$EV/b1-ports-$n.txt" && echo "PASS: kubelet LISTEN tren $n"
  grep -q 'kube-proxy' "$EV/b1-ports-$n.txt" && echo "PASS: kube-proxy LISTEN tren $n"
done
```

Cuối cùng, thử **đường thật** từ worker tới API server, đúng phép thử mà bài mô tả:

```bash
for n in "$W1" "$W2"; do
  ssh "$n" "timeout 3 bash -c 'echo > /dev/tcp/${CP_ENDPOINT%%:*}/$API_PORT'" \
    && echo "PASS: $n toi duoc API $CP_ENDPOINT" \
    || echo "FAIL: $n khong toi duoc API $CP_ENDPOINT"
done
```

**Ý nghĩa:** bài không bảo bạn mở port bằng tay — nó bảo bạn **kiểm tra** port có mở không. Trên
mạng lab, [A4.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập) đã tắt
UFW nên mọi port đều thông; danh sách tiến trình ở trên cho thấy **thực tế cần những port nào**.
Đọc bảng port chính thức mà bài trỏ tới rồi đối chiếu với output `ss` của bạn: mỗi dòng LISTEN ứng
với một mục trong bảng. Trên cluster production, thiếu một trong số đó thì lỗi không hiện ra ở
`kubeadm init` mà hiện ra muộn hơn — lúc worker join, lúc `kubectl logs`, hoặc lúc etcd bầu leader.

**PASS:** năm dòng `PASS:` ở khối đầu, bốn dòng `PASS:` ở khối worker, và hai dòng
`PASS: ... toi duoc API ...`; không có dòng `FAIL:` nào.

### B1.3. Swap — phân biệt tắt tạm với tắt vĩnh viễn

```bash
for n in $NODES; do
  echo "== $n"
  ssh "$n" 'swapon --show; echo "swapon-rc=$?"'
  ssh "$n" 'grep -nE "^[^#].*[[:space:]]swap[[:space:]]" /etc/fstab; echo "fstab-rc=$?"'
done | tee "$EV/b1-swap.txt"

BAD=0
for n in $NODES; do
  test -z "$(ssh "$n" 'swapon --show')" || { echo "FAIL: $n dang bat swap"; BAD=1; }
  ssh "$n" 'grep -qE "^[^#].*[[:space:]]swap[[:space:]]" /etc/fstab' \
    && { echo "FAIL: $n con dong swap chua comment trong /etc/fstab"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: swap tat ca o hien tai lan sau reboot tren ba node'
```

Đọc thêm giá trị **hiệu lực** mà kubelet đang dùng:

```bash
for n in $NODES; do
  echo -n "$n failSwapOn="
  ssh "$n" 'sudo grep -i "^failSwapOn" /var/lib/kubelet/config.yaml || echo "(vang mat — dung mac dinh)"'
done | tee "$EV/b1-failswapon.txt"

test -s "$EV/b1-failswapon.txt" && echo 'PASS: da ghi trang thai failSwapOn cua ba kubelet'
```

**Ý nghĩa:** bài nêu **hai** lối thoát cho swap, không phải một: tắt swap, hoặc để kubelet chấp
nhận nó bằng `failSwapOn: false`. Và bài cảnh báo rằng ngay cả khi đã tolerate, **workload vẫn
không dùng được swap** trừ khi đổi `swapBehavior` khác mặc định `NoSwap`. Baseline chọn vế thứ
nhất. Điểm dễ sai là ranh giới giữa hai lệnh: `swapoff -a` chỉ tắt **tạm thời**, còn dòng swap
trong `/etc/fstab` mới quyết định trạng thái **sau reboot** — và hành vi mặc định của kubelet là
**không khởi động** nếu phát hiện swap. Bạn sẽ thấy đúng cặp này ở B6.4, khi ba node reboot sau
`kubeadm reset`: kubelet phải lên lại được, và nó chỉ lên được vì `/etc/fstab` đã sạch.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không có dòng `FAIL:`.

### B1.4. cgroup driver — hai phía phải khớp

[Lab 2 B2](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md#b2-cgroup-v2-và-cgroup-driver) đã đọc kỹ hai
phía và giải thích cơ chế. Ở đây chỉ cần một phép kiểm nhanh, vì B7.1 sắp **khai tường minh** giá
trị này trong file cấu hình kubeadm:

```bash
for n in $NODES; do
  KD="$(ssh "$n" "sudo awk -F': *' '/^cgroupDriver/{print \$2}' /var/lib/kubelet/config.yaml")"
  CD="$(ssh "$n" "sudo awk -F'= *' '/SystemdCgroup/{gsub(/ /,\"\",\$2); print \$2}' /etc/containerd/config.toml")"
  echo "$n kubelet=$KD containerd_SystemdCgroup=$CD"
done | tee "$EV/b1-cgroupdriver.txt"

test "$(awk '{print $2}' "$EV/b1-cgroupdriver.txt" | sort -u)" = 'kubelet=systemd' \
  && test "$(awk '{print $3}' "$EV/b1-cgroupdriver.txt" | sort -u)" = 'containerd_SystemdCgroup=true' \
  && echo 'PASS: ca ba node co kubelet va containerd cung dung cgroup driver systemd'
```

**Ý nghĩa:** bài 01 đặt riêng một cảnh báo cho việc này — cgroup driver của container runtime và
của kubelet **bắt buộc phải khớp**, nếu không **tiến trình kubelet sẽ thất bại**. Đây là thuộc tính
quản lý cgroup, không phải chuyện thiếu tài nguyên, nên node còn dư RAM và CPU cũng không cứu được.
Bài [09](../09-troubleshooting-kubeadm-vi.md) liệt kê cgroup driver lệch là một trong ba nguyên
nhân của mục *kubeadm bị treo khi chờ control plane* — triệu chứng bạn sẽ tra tới nếu B7.2 treo.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B1.5. Ba gói, ba vai trò, và quy tắc lệch phiên bản

```bash
for n in $NODES; do
  echo "== $n"
  ssh "$n" "dpkg-query -W -f='\${Package} \${Version}\n' kubeadm kubelet kubectl cri-tools kubernetes-cni"
  ssh "$n" "apt-mark showhold | tr '\n' ' '; echo"
done | tee "$EV/b1-packages.txt"

SRV_VER="$(kubectl get --raw /version | grep -o '"gitVersion":"[^"]*"' | cut -d'"' -f4)"
KUBELET_V="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.status.nodeInfo.kubeletVersion}{"\n"}{end}' | sort -u)"
echo "kubelet tren cac node: $KUBELET_V"
echo "API server: $SRV_VER"

test "$(echo "$KUBELET_V" | wc -l)" -eq 1 && test "$KUBELET_V" = "$SRV_VER" \
  && echo 'PASS: ba kubelet cung mot phien ban va khong cao hon API server'

for n in $NODES; do
  ssh "$n" 'apt-mark showhold' | grep -q '^kubelet$' \
    && ssh "$n" 'apt-mark showhold' | grep -q '^kubeadm$' \
    && ssh "$n" 'apt-mark showhold' | grep -q '^kubectl$' \
    && echo "PASS: $n da hold ca ba goi Kubernetes" \
    || echo "FAIL: $n chua hold du ba goi"
done
```

**Ý nghĩa:** bài nói thẳng rằng kubeadm **sẽ không** cài và **không** quản lý `kubelet` hay
`kubectl` cho bạn — việc khớp phiên bản là trách nhiệm của người vận hành. Quy tắc bài đưa ra là
**bất đối xứng**: lệch một minor giữa kubelet và control plane thì được, nhưng **kubelet không bao
giờ được vượt quá phiên bản của API server**. Trực giác "cứ mới hơn là an toàn" sai đúng ở chỗ này.
`apt-mark hold` của [A4.3](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl)
tồn tại vì lý do đó, và bài cảnh báo trong khung riêng rằng các gói Kubernetes phải bị loại khỏi
mọi đợt nâng cấp hệ thống — nâng cấp là quy trình riêng của
[giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster).

**PASS:** dòng `PASS: ba kubelet cung mot phien ban ...` và ba dòng
`PASS: ... da hold ca ba goi Kubernetes` xuất hiện; không có dòng `FAIL:`.

---

## B2. Bài 02 — giải phẫu cluster mà kubeadm đã dựng

**Mục đích:** trước khi phá, đọc cho hết những gì `kubeadm init` và `kubeadm join` đã để lại trên
`03-storage-ready`. Mỗi hiện vật ở đây sẽ xuất hiện lại ở B7 dưới dạng một dòng output của
`kubeadm init` — lúc đó bạn phải nhận ra nó, không phải gặp lần đầu.

Toàn bộ B2 **chỉ đọc**, trừ B2.5 sinh một bootstrap token mới.

### B2.1. Cây `/etc/kubernetes` và năm file kubeconfig

```bash
. ~/lab-evidence/8a/vars.sh

sudo ls -la /etc/kubernetes/ | tee "$EV/b2-etc-kubernetes.txt"
sudo ls -la /etc/kubernetes/manifests/
sudo ls /etc/kubernetes/pki/ /etc/kubernetes/pki/etcd/

for f in admin.conf super-admin.conf controller-manager.conf scheduler.conf kubelet.conf; do
  sudo test -f "/etc/kubernetes/$f" \
    && echo "PASS: co /etc/kubernetes/$f" \
    || echo "FAIL: thieu /etc/kubernetes/$f"
done

for f in etcd kube-apiserver kube-controller-manager kube-scheduler; do
  sudo test -f "/etc/kubernetes/manifests/$f.yaml" \
    && echo "PASS: co manifest static Pod $f.yaml" \
    || echo "FAIL: thieu manifest $f.yaml"
done
```

**Ý nghĩa:** năm kubeconfig là năm danh tính khác nhau, không phải năm bản sao. `admin.conf` là
danh tính người vận hành; `super-admin.conf` là danh tính break-glass; ba file còn lại là danh
tính của ba tiến trình. Bốn manifest trong `manifests/` là toàn bộ control plane — kể cả etcd, cơ
sở dữ liệu của cluster, cũng chạy ở dạng static Pod trên chính node này. Đó là lý do bài
[02](../02-create-cluster-kubeadm-vi.md) gọi cluster một control plane là có
**một cơ sở dữ liệu etcd duy nhất** và cảnh báo mất node là có thể mất dữ liệu.

**PASS:** năm dòng `PASS: co /etc/kubernetes/...` và bốn dòng `PASS: co manifest static Pod ...`;
không có dòng `FAIL:`.

### B2.2. `pki/` — ba CA và hai danh tính quản trị khác nhau

```bash
for c in pki/ca.crt pki/front-proxy-ca.crt pki/etcd/ca.crt; do
  sudo test -f "/etc/kubernetes/$c" \
    && echo "PASS: co CA $c" \
    || echo "FAIL: thieu CA $c"
done

sudo kubeadm certs check-expiration | tee "$EV/b2-certs.txt"
grep -qi 'invalid\|expired' "$EV/b2-certs.txt" \
  && echo 'FAIL: co certificate da het han' \
  || echo 'PASS: khong certificate nao da het han'
```

Đọc `Subject` của hai kubeconfig quản trị — đây là điểm mà bài 02 dành hẳn một khung cảnh báo:

```bash
for f in admin super-admin; do
  echo -n "$f: "
  sudo grep 'client-certificate-data' "/etc/kubernetes/$f.conf" \
    | awk '{print $2}' | base64 -d | openssl x509 -noout -subject
done | tee "$EV/b2-admin-subject.txt"

grep -q 'kubeadm:cluster-admins' "$EV/b2-admin-subject.txt" \
  && echo 'PASS: admin.conf mang nhom kubeadm:cluster-admins'
grep -q 'system:masters' "$EV/b2-admin-subject.txt" \
  && echo 'PASS: super-admin.conf mang nhom system:masters'
```

**Ý nghĩa:** `admin.conf` mang `O = kubeadm:cluster-admins` — một nhóm được **bind với ClusterRole
`cluster-admin`**, tức quyền của nó đi qua RBAC. `super-admin.conf` mang `O = system:masters` —
nhóm siêu người dùng **bỏ qua lớp phân quyền**, kể cả RBAC. Khác biệt đó có nghĩa thật: gỡ
ClusterRoleBinding của `kubeadm:cluster-admins` thì `admin.conf` mất quyền, còn `super-admin.conf`
thì không. Bài dặn không chia sẻ cả hai và khuyến nghị đưa `super-admin.conf` vào nơi lưu trữ an
toàn. Lab chỉ đọc trường `Subject` — **không** đọc và **không** lưu phần private key.

**PASS:** ba dòng `PASS: co CA ...`, dòng `PASS: khong certificate nao da het han`, và hai dòng
`PASS: ... mang nhom ...` xuất hiện; không có dòng `FAIL:`.

### B2.3. Static Pod: ai tạo, ai sở hữu, ai dựng lại

```bash
kubectl -n kube-system get pods -o wide \
  --field-selector "spec.nodeName=$MASTER" | tee "$EV/b2-static-pods.txt"

APISRV_POD="kube-apiserver-$MASTER"

kubectl -n kube-system get pod "$APISRV_POD" \
  -o jsonpath='{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}{"\n"}'
kubectl -n kube-system get pod "$APISRV_POD" \
  -o jsonpath='{.metadata.annotations.kubernetes\.io/config\.source}{"\n"}'

test "$(kubectl -n kube-system get pod "$APISRV_POD" \
  -o jsonpath='{.metadata.ownerReferences[0].kind}')" = 'Node' \
  && echo 'PASS: mirror Pod cua apiserver thuoc so huu cua Node, khong phai controller'
test "$(kubectl -n kube-system get pod "$APISRV_POD" \
  -o jsonpath='{.metadata.annotations.kubernetes\.io/config\.source}')" = 'file' \
  && echo 'PASS: nguon cua Pod nay la file tren dia, khong phai API server'
```

**Ý nghĩa:** đây là chỗ mà mô hình điều khiển của Kubernetes bị **đảo chiều**. Với Pod thường,
API server giữ desired state và kubelet đi theo. Với static Pod, **file trên đĩa** giữ desired
state; kubelet đọc thư mục `/etc/kubernetes/manifests` và tự chạy container, rồi tạo một *mirror
Pod* trong API chỉ để bạn nhìn thấy. Mirror Pod không có Deployment hay DaemonSet nào đứng sau —
`ownerReferences` trỏ về chính **Node**. Hệ quả trực tiếp, và bạn sẽ thử nó ở B5.1: xóa mirror Pod
bằng `kubectl delete` thì kubelet dựng lại ngay vì file vẫn còn; xóa **file** thì container biến
mất thật. Đó cũng là lý do control plane tự dựng được: kubelet khởi động apiserver mà **không cần**
apiserver.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B2.4. Taint của control plane và giới hạn của `NodeRestriction`

```bash
kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,ROLE:.metadata.labels.node-role\.kubernetes\.io/control-plane,TAINTS:.spec.taints[*].key'

TAINT="$(kubectl get node "$MASTER" -o jsonpath='{.spec.taints[*].key}')"
echo "taint tren $MASTER: $TAINT"
case "$TAINT" in
  *node-role.kubernetes.io/control-plane*) echo 'PASS: control plane con giu taint mac dinh' ;;
  *) echo 'FAIL: taint control plane da bi go — moi lab sau se lap lich khac di' ;;
esac

test -z "$(kubectl get node "$W1" -o jsonpath='{.spec.taints[*].key}')" \
  && test -z "$(kubectl get node "$W2" -o jsonpath='{.spec.taints[*].key}')" \
  && echo 'PASS: hai worker khong mang taint nao'

for n in "$W1" "$W2"; do
  ssh "$n" 'sudo grep -c "node-labels" /var/lib/kubelet/kubeadm-flags.env' \
    | grep -q '^0$' \
    && echo "PASS: $n khong tu gan label qua --node-labels" \
    || echo "FAIL: $n co --node-labels trong kubeadm-flags.env"
done
```

**Ý nghĩa:** hai chiều ngược nhau của cùng một cơ chế bảo vệ. Chiều thứ nhất: control plane node
mặc định mang taint `node-role.kubernetes.io/control-plane:NoSchedule`, nên Pod thường không lên
đó — bài đưa lệnh `kubectl taint nodes --all node-role.kubernetes.io/control-plane-` để gỡ khi
cluster chỉ có một máy, và
[A5.4.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a543-tầng-2--node-condition-taint-và-podcidr) dặn
**không gỡ** vì nó đổi hành vi lập lịch của mọi lab sau. Chiều thứ hai: admission controller
`NodeRestriction` — kubeadm bật mặc định — khiến kubelet **không thể** tự gán label
`node-role.kubernetes.io/*` cho chính nó qua `--node-labels`; cố làm thì node **không đăng ký
được** với API server. Muốn gán thì phải `kubectl label` sau khi node đã join, bằng một kubeconfig
có đặc quyền.

**PASS:** dòng `PASS: control plane con giu taint mac dinh`, dòng
`PASS: hai worker khong mang taint nao`, và hai dòng `PASS: ... khong tu gan label ...` xuất hiện.

### B2.5. Ba thành phần của lệnh `kubeadm join`, tự tính lại hash

**Bước này tạo một bootstrap token mới** — thay đổi duy nhất mà B2 gây ra. Token có hạn và cluster
sẽ được restore ở B11, nên nó không để lại dấu vết.

```bash
sudo kubeadm token create --print-join-command > "$WK/b2-join-command.txt"
chmod 600 "$WK/b2-join-command.txt"

JOIN_ENDPOINT="$(awk '{print $3}' "$WK/b2-join-command.txt")"
JOIN_HASH="$(tr ' ' '\n' < "$WK/b2-join-command.txt" | grep '^sha256:' | head -1)"
echo "endpoint=$JOIN_ENDPOINT"
echo "hash tu kubeadm=$JOIN_HASH"

# Chi hai gia tri KHONG bi mat moi duoc ghi vao evidence.
printf 'endpoint=%s\nca-cert-hash=%s\n' "$JOIN_ENDPOINT" "$JOIN_HASH" \
  | tee "$EV/b2-join-info.txt"
```

Bây giờ tự tính lại hash đó từ CA công khai, không dùng kubeadm:

```bash
CALC_HASH="sha256:$(sudo openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt \
  | openssl pkey -pubin -outform der 2>/dev/null \
  | openssl dgst -sha256 -hex | awk '{print $NF}')"
echo "hash tu openssl=$CALC_HASH"

test "$JOIN_HASH" = "$CALC_HASH" \
  && echo 'PASS: hash trong lenh join dung la SHA-256 cua public key trong pki/ca.crt'
test "$JOIN_ENDPOINT" = "$CP_ENDPOINT" \
  && echo 'PASS: endpoint trong lenh join dung la controlPlaneEndpoint da doc o B0.2'
```

**Ý nghĩa:** lệnh `kubeadm join` chỉ có ba phần, và mỗi phần trả lời một câu hỏi khác nhau.
**Endpoint** trả lời "nói chuyện với ai". **Token** trả lời "chứng minh mình được phép join thế
nào" — bài nói token là **bí mật**, ai có nó đều thêm được node đã xác thực vào cluster.
**`--discovery-token-ca-cert-hash`** trả lời chiều ngược lại: "làm sao node tin được cái nó vừa
kết nối tới đúng là cluster này". Đó là xác thực **lẫn nhau**, và phép tính bạn vừa làm chứng minh
hash không phải một chuỗi ma thuật do kubeadm phát ra — nó là vân tay của public key trong
`pki/ca.crt`, ai cầm file CA công khai cũng tính lại được.

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Lệnh join đầy đủ nằm ở `~/lab-work/8a/`, **không**
nằm trong evidence — chỉ endpoint và hash công khai được ghi lại. Đừng dán token vào tài liệu nào.

### B2.6. Ba bảng lệch phiên bản, đối chiếu với cluster thật

```bash
KUBEADM_VER="$(kubeadm version -o short)"
SRV_VER="$(kubectl get --raw /version | grep -o '"gitVersion":"[^"]*"' | cut -d'"' -f4)"
KUBELET_V="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.status.nodeInfo.kubeletVersion}{"\n"}{end}' | sort -u)"

{
  echo "kubeadm binary        : $KUBEADM_VER"
  echo "kubernetesVersion     : $SRV_VER"
  echo "kubelet tren cac node : $KUBELET_V"
  echo '--- kubeadm tren tung node'
  for n in $NODES; do echo -n "$n: "; ssh "$n" 'kubeadm version -o short'; done
} | tee "$EV/b2-skew.txt"

SAME="$(grep -c "$K8S_BASELINE" "$EV/b2-skew.txt")"
echo "so dong khop baseline: $SAME"
test "$SAME" -ge 6 \
  && echo 'PASS: kubeadm, control plane va kubelet tren ba node deu cung mot phien ban'
```

**Ý nghĩa:** bài đưa **ba** bảng skew riêng biệt, và cluster lab thỏa cả ba theo cách đơn giản
nhất — mọi thứ cùng một phiên bản:

| Bảng | Ràng buộc của bài | Cluster này |
| --- | --- | --- |
| kubeadm ↔ `kubernetesVersion` | dùng được với phiên bản bằng kubeadm hoặc **cũ hơn một** minor | bằng nhau |
| kubeadm ↔ kubelet | kubelet bằng kubeadm hoặc **cũ hơn tối đa ba** minor | bằng nhau |
| kubeadm ↔ kubeadm | binary chạy `kubeadm join` phải **khớp** binary đã `kubeadm init` | cùng gói trên ba node |

Bảng thứ ba là bảng bạn sắp dựa vào ở B8: worker join bằng chính binary `kubeadm` đang có trên nó,
và cluster mới do binary cùng phiên bản trên master khởi tạo. Nếu ba node lệch nhau — ví dụ vì một
node từng nâng gói ngoài quy trình — thì `kubeadm join` là chỗ lỗi hiện ra.

**PASS:** dòng `PASS: kubeadm, control plane va kubelet ...` xuất hiện.

---

## B3. Bài 03 — `ClusterConfiguration` thật và cơ chế patch

**Mục đích:** [A5.1 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) dựng cluster
bằng **cờ dòng lệnh**. Bài [03](../03-control-plane-flags-vi.md) là cách thứ hai: mô tả cluster
bằng **file cấu hình**. Ở đây bạn đọc file cấu hình mà kubeadm đã tự sinh từ những cờ đó, rồi soạn
sẵn thư mục patch cho B7 dùng.

### B3.1. `ClusterConfiguration` mà cluster đang thực sự chạy

```bash
kubectl -n kube-system get cm kubeadm-config \
  -o jsonpath='{.data.ClusterConfiguration}' | tee "$EV/b3-clusterconfig.txt"

kubeadm config print init-defaults > "$EV/b3-init-defaults.txt"
head -30 "$EV/b3-init-defaults.txt"

for k in kubernetesVersion controlPlaneEndpoint networking; do
  grep -q "^$k\|^  $k" "$EV/b3-clusterconfig.txt" \
    && echo "PASS: ClusterConfiguration co truong $k" \
    || echo "FAIL: khong thay truong $k"
done

CC_VER="$(awk -F': *' '/^kubernetesVersion/{print $2}' "$EV/b3-clusterconfig.txt")"
CC_EP="$(awk -F': *' '/^controlPlaneEndpoint/{print $2}' "$EV/b3-clusterconfig.txt")"
echo "kubernetesVersion=$CC_VER | controlPlaneEndpoint=$CC_EP"

test "$CC_VER" = "$K8S_BASELINE" \
  && echo 'PASS: kubernetesVersion trong ConfigMap khop baseline'
test "$CC_EP" = "$CP_ENDPOINT" \
  && echo 'PASS: controlPlaneEndpoint trong ConfigMap khop gia tri doc o B0.2'
```

**Ý nghĩa:** cờ dòng lệnh của `kubeadm init` **không biến mất sau khi chạy**. Phase `upload-config`
gói chúng lại thành một `ClusterConfiguration` và lưu vào ConfigMap `kubeadm-config` trong
`kube-system` — đó là "bản ghi sự thật" mà `kubeadm join`, `kubeadm upgrade` và `kubeadm certs` đọc
về sau. So file này với `kubeadm config print init-defaults` để thấy đâu là mặc định, đâu là thứ
Lab 00 đã đặt.

**PASS:** ba dòng `PASS: ClusterConfiguration co truong ...` và hai dòng `PASS: ... khop ...` xuất
hiện; không có dòng `FAIL:`.

### B3.2. Phạm vi toàn cục của `extraArgs`, đối chiếu với args thật

```bash
grep -nE 'extraArgs|extraVolumes' "$EV/b3-clusterconfig.txt" \
  || echo '(khong co extraArgs nao — cluster dung nguyen mac dinh cua kubeadm)'

sudo awk '/^    - kube-apiserver/,/^    image:/' \
  /etc/kubernetes/manifests/kube-apiserver.yaml | tee "$EV/b3-apiserver-args.txt"

N_ARGS="$(grep -c '^ *- --' "$EV/b3-apiserver-args.txt")"
echo "so flag kube-apiserver dang chay: $N_ARGS"

test "$N_ARGS" -gt 0 \
  && echo 'PASS: doc duoc danh sach flag that cua kube-apiserver'
grep -qE 'extraArgs' "$EV/b3-clusterconfig.txt" \
  && echo 'INFO: cluster co dat extraArgs — ghi lai vao evidence' \
  || echo 'PASS: cluster khong dat extraArgs nao, moi flag deu do kubeadm sinh'
```

**Ý nghĩa:** hàng chục flag bạn vừa đọc **không** đến từ `extraArgs`; chúng là những gì kubeadm tự
sinh cho một cluster mặc định. `extraArgs` chỉ là cửa để **ghi đè** chúng, và cửa đó có hai giới
hạn mà bài nêu rõ. Thứ nhất, `ClusterConfiguration` có **phạm vi toàn cục**: flag bạn thêm áp cho
**mọi instance của thành phần đó trên mọi node** — cái tên nghe như "cấu hình của cluster này"
nhưng chính vì toàn cục mà nó **không** mô tả được khác biệt giữa các node. Thứ hai, `extraArgs`
là danh sách cặp `name`/`value` và **không hỗ trợ key trùng lặp**. Cả hai giới hạn đều dẫn tới cùng
một lối thoát: patch. Thêm một chi tiết dễ quên — nếu flag trỏ tới một file trên host thì phải
mount file đó vào static Pod bằng `extraVolumes`, vì thành phần control plane chạy trong Pod chứ
không chạy trên host.

**PASS:** dòng `PASS: doc duoc danh sach flag that cua kube-apiserver` và một trong hai dòng còn
lại của khối thứ hai xuất hiện.

### B3.3. Soạn thư mục patch theo đúng quy ước tên file

Patch mẫu dưới đây thêm một annotation vào static Pod của `kube-scheduler`. Chọn target này vì nó
vô hại, không đổi hành vi thành phần nào, và kiểm chứng được ngay trên đĩa ở B7.5.

```bash
cat > "$WK/patches/kube-scheduler0+merge.yaml" <<'EOF'
metadata:
  annotations:
    lab-8a.local/patched-by: kubeadm-patch
EOF

ls -l "$WK/patches/"
cat "$WK/patches/kube-scheduler0+merge.yaml"

test -f "$WK/patches/kube-scheduler0+merge.yaml" \
  && echo 'PASS: da soan file patch dung ten quy uoc target[suffix][+patchtype].extension'
```

Kiểm lại từng thành phần của tên file so với quy ước bài đưa ra:

| Thành phần | Giá trị ở đây | Quy tắc của bài |
| --- | --- | --- |
| `target` | `kube-scheduler` | chỉ nhận `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, `etcd`, `kubeletconfiguration`, `corednsdeployment` |
| `suffix` | `0` | tùy chọn, quyết định thứ tự áp theo thứ tự chữ-số khi có nhiều patch cùng target |
| `patchtype` | `merge` | một trong `strategic`, `merge`, `json`; **mặc định là `strategic`** khi bỏ trống |
| `extension` | `yaml` | bắt buộc là `json` hoặc `yaml` |

```bash
BN="$(basename "$WK/patches/kube-scheduler0+merge.yaml")"
case "$BN" in
  kube-apiserver*|kube-controller-manager*|kube-scheduler*|etcd*|kubeletconfiguration*|corednsdeployment*)
    echo 'PASS: target hop le' ;;
  *) echo 'FAIL: target khong nam trong danh sach hop le' ;;
esac
case "$BN" in
  *.yaml|*.json) echo 'PASS: extension hop le' ;;
  *) echo 'FAIL: extension phai la json hoac yaml' ;;
esac
```

**Ý nghĩa:** patch là **bước tùy chỉnh cuối cùng trước khi cấu hình của thành phần được ghi xuống
đĩa**. Nói cách khác, patch **thắng** mọi thứ kubeadm sinh ra trước đó, kể cả `extraArgs`. Nó cũng
là cơ chế duy nhất áp **trên từng node riêng lẻ**, vì `patches.directory` nằm trong
`InitConfiguration`/`JoinConfiguration` — hai kind gắn với một node cụ thể — chứ không nằm trong
`ClusterConfiguration`. Cái giá đi kèm: khi nâng cấp, bài cảnh báo bạn **phải cung cấp lại chính
các patch đó** qua `UpgradeConfiguration`, nếu không tùy chỉnh **không được bảo toàn**. Đó là lý do
lab này chỉ dùng patch cho một annotation vô hại, và vì sao một cluster đầy patch là một cluster
khó nâng cấp.

Lưu ý ranh giới cuối: `kube-proxy` **không** nằm trong danh sách target hợp lệ. Vì kubeadm triển
khai kube-proxy dưới dạng **DaemonSet**, `KubeProxyConfiguration` luôn áp cho toàn bộ instance
trong cluster và không có cách nào cho một node chạy cấu hình kube-proxy khác.

**PASS:** ba dòng `PASS:` của bước này xuất hiện, không có dòng `FAIL:`.

---

## B4. Bài 04 — kubeadm ghi gì cho kubelet

**Mục đích:** trả lời chính xác câu hỏi của bài [04](../04-kubelet-integration-vi.md): `kubeadm
init` và `kubeadm join` ghi những file nào cho kubelet, và **ai thắng** khi chúng mâu thuẫn.

### B4.1. Năm hiện vật trên node

```bash
. ~/lab-evidence/8a/vars.sh

for n in $NODES; do
  echo "== $n"
  ssh "$n" 'sudo ls -l /var/lib/kubelet/config.yaml /var/lib/kubelet/instance-config.yaml \
    /var/lib/kubelet/kubeadm-flags.env /etc/kubernetes/kubelet.conf 2>&1'
done | tee "$EV/b4-kubelet-files.txt"

kubectl -n kube-system get cm kubelet-config \
  -o jsonpath='{.data.kubelet}' | tee "$EV/b4-kubelet-configmap.txt" | head -20

for n in $NODES; do
  ssh "$n" 'sudo test -f /var/lib/kubelet/config.yaml' \
    && ssh "$n" 'sudo test -f /var/lib/kubelet/kubeadm-flags.env' \
    && ssh "$n" 'sudo test -f /etc/kubernetes/kubelet.conf' \
    && echo "PASS: $n co du ba file kubelet bat buoc" \
    || echo "FAIL: $n thieu file kubelet"
done

test -s "$EV/b4-kubelet-configmap.txt" \
  && echo 'PASS: ConfigMap kubelet-config ton tai trong kube-system'
```

**Ý nghĩa:** năm hiện vật, hai mẫu cấu hình đối lập nhau. `/var/lib/kubelet/config.yaml` và
ConfigMap `kubelet-config` mang phần **giống nhau toàn cluster**; `instance-config.yaml` và
`kubeadm-flags.env` mang phần **riêng từng máy**; `/etc/kubernetes/kubelet.conf` là danh tính của
kubelet để nói chuyện với API server. Bài liệt kê rõ những thứ **không nhét được** vào
`KubeletConfiguration` cấp cluster: `--resolv-conf`, `--hostname-override`, `--cgroup-driver`,
`--container-runtime-endpoint` — và khuyến nghị dùng patch target `kubeletconfiguration` cho nhóm
đó. `instance-config.yaml` có thể vắng mặt tùy phiên bản; gate ở trên chỉ bắt buộc ba file còn lại.

**PASS:** ba dòng `PASS: ... co du ba file kubelet bat buoc` và dòng
`PASS: ConfigMap kubelet-config ton tai ...` xuất hiện.

### B4.2. ConfigMap và file trên node có cùng một cấu hình cơ sở không

```bash
CM_DNS="$(grep -A2 '^clusterDNS' "$EV/b4-kubelet-configmap.txt" | grep -oE '[0-9.]+' | head -1)"
CM_DOMAIN="$(awk -F': *' '/^clusterDomain/{print $2}' "$EV/b4-kubelet-configmap.txt")"
CM_DRIVER="$(awk -F': *' '/^cgroupDriver/{print $2}' "$EV/b4-kubelet-configmap.txt")"
echo "ConfigMap: clusterDNS=$CM_DNS clusterDomain=$CM_DOMAIN cgroupDriver=$CM_DRIVER"

: > "$EV/b4-kubelet-compare.txt"
BAD=0
for n in $NODES; do
  N_DNS="$(ssh "$n" "sudo grep -A2 '^clusterDNS' /var/lib/kubelet/config.yaml \
    | grep -oE '[0-9.]+' | head -1")"
  N_DOMAIN="$(ssh "$n" "sudo awk -F': *' '/^clusterDomain/{print \$2}' /var/lib/kubelet/config.yaml")"
  N_DRIVER="$(ssh "$n" "sudo awk -F': *' '/^cgroupDriver/{print \$2}' /var/lib/kubelet/config.yaml")"
  echo "$n: clusterDNS=$N_DNS clusterDomain=$N_DOMAIN cgroupDriver=$N_DRIVER" \
    | tee -a "$EV/b4-kubelet-compare.txt"
  { test "$N_DNS" = "$CM_DNS" && test "$N_DOMAIN" = "$CM_DOMAIN" \
    && test "$N_DRIVER" = "$CM_DRIVER"; } || { echo "FAIL: $n lech ConfigMap"; BAD=1; }
done

test "$BAD" -eq 0 \
  && echo 'PASS: ba node dung cung mot KubeletConfiguration co so voi ConfigMap'
```

**Ý nghĩa:** đây là "lan truyền cấu hình cấp cluster tới từng kubelet" nhìn từ bên trong. Trên
master, `kubeadm init` ghi file **rồi tải lên** ConfigMap. Trên worker, `kubeadm join` đi ngược
chiều: nó **tải xuống** ConfigMap rồi ghi ra file. Cùng một nội dung, hai chiều dữ liệu khác nhau —
và đó là lý do node join sau vẫn dùng đúng `clusterDNS` mà bạn chưa hề gõ trên máy đó.

**PASS:** dòng `PASS: ba node dung cung mot KubeletConfiguration co so voi ConfigMap` xuất hiện,
không có dòng `FAIL:`.

### B4.3. `kubeadm-flags.env` — phần riêng của từng máy

```bash
for n in $NODES; do
  echo "== $n"
  ssh "$n" 'sudo cat /var/lib/kubelet/kubeadm-flags.env'
done | tee "$EV/b4-flags-env.txt"

for n in $NODES; do
  ssh "$n" 'sudo grep -q "KUBELET_KUBEADM_ARGS=" /var/lib/kubelet/kubeadm-flags.env' \
    && echo "PASS: $n co bien KUBELET_KUBEADM_ARGS" \
    || echo "FAIL: $n thieu bien KUBELET_KUBEADM_ARGS"
done

grep -o '\-\-container-runtime-endpoint=[^" ]*' "$EV/b4-flags-env.txt" | sort -u
grep -q 'container-runtime-endpoint' "$EV/b4-flags-env.txt" \
  && echo 'PASS: CRI socket duoc truyen qua kubeadm-flags.env, dung nhom cau hinh rieng tung may'
```

**Ý nghĩa:** file này là **file môi trường động**: kubeadm phát hiện CRI socket trên node rồi ghi
nó ra đây dưới dạng cờ dòng lệnh, cùng các tham số động khác. Nó thuộc mẫu "cấu hình riêng cho từng
máy", đối lập với `config.yaml` ở B4.2. Đây cũng chính là giá trị `CRI_SOCKET` bạn đọc từ annotation
Node ở B0.2 — hai nguồn độc lập cho cùng một sự thật.

**PASS:** ba dòng `PASS: ... co bien KUBELET_KUBEADM_ARGS` và dòng
`PASS: CRI socket duoc truyen qua kubeadm-flags.env ...` xuất hiện.

### B4.4. Thứ tự trong `ExecStart` chính là thứ tự ưu tiên

```bash
systemctl cat kubelet | tee "$EV/b4-kubelet-unit.txt"

EXEC="$(grep -m1 '^ExecStart=/usr/bin/kubelet' "$EV/b4-kubelet-unit.txt")"
echo "$EXEC"

for v in KUBELET_KUBECONFIG_ARGS KUBELET_CONFIG_ARGS KUBELET_KUBEADM_ARGS KUBELET_EXTRA_ARGS; do
  case "$EXEC" in
    *"\$$v"*) echo "PASS: ExecStart co $v" ;;
    *) echo "FAIL: ExecStart thieu $v" ;;
  esac
done

LAST="$(echo "$EXEC" | awk '{print $NF}')"
test "$LAST" = '$KUBELET_EXTRA_ARGS' \
  && echo 'PASS: KUBELET_EXTRA_ARGS dung o cuoi chuoi co, tuc co do uu tien cao nhat'

ls -l /etc/systemd/system/kubelet.service.d/ 2>/dev/null \
  || echo '(khong co thu muc override cuc bo — dung nhu Lab 00 de lai)'
test -f /etc/default/kubelet \
  && echo 'INFO: co /etc/default/kubelet — doc noi dung va ghi vao evidence' \
  || echo 'PASS: khong co /etc/default/kubelet, nen KUBELET_EXTRA_ARGS rong'
```

**Ý nghĩa:** khi cùng một cờ xuất hiện ở nhiều nguồn, kubelet lấy **giá trị cuối cùng** trên dòng
lệnh. Vì `$KUBELET_EXTRA_ARGS` đứng cuối `ExecStart` và được nạp từ `/etc/default/kubelet`, nó
**thắng** mọi thứ kubeadm sinh ra — kể cả `kubeadm-flags.env`. Bài gọi đây là "phương án cuối cùng"
và khuyên dùng `nodeRegistration.kubeletExtraArgs` thay thế. Một chi tiết dễ hỏng: file drop-in
`10-kubeadm.conf` nằm dưới `/usr/lib/...` và **lệnh kubeadm CLI không bao giờ đụng vào nó**; muốn
ghi đè unit thì đặt file riêng trong `/etc/systemd/system/kubelet.service.d/`, **không** sửa thẳng
`10-kubeadm.conf` — sửa thẳng thì lần cài lại gói `kubeadm` sẽ ghi đè mất.

**PASS:** bốn dòng `PASS: ExecStart co ...`, dòng `PASS: KUBELET_EXTRA_ARGS dung o cuoi ...` xuất
hiện; không có dòng `FAIL:`.

### B4.5. Dấu vết của TLS Bootstrap trên worker

```bash
for n in "$W1" "$W2"; do
  if ssh "$n" 'sudo test -f /etc/kubernetes/bootstrap-kubelet.conf'; then
    echo "FAIL: $n con bootstrap-kubelet.conf — TLS Bootstrap chua hoan tat"
  else
    echo "PASS: $n khong con bootstrap-kubelet.conf"
  fi
  ssh "$n" 'sudo test -f /etc/kubernetes/kubelet.conf' \
    && echo "PASS: $n co kubelet.conf — danh tinh duy nhat da duoc cap"
done
```

**Ý nghĩa:** hai file này **không bao giờ cùng tồn tại lâu**. `bootstrap-kubelet.conf` chỉ chứa CA
certificate và Bootstrap Token; kubelet dùng nó để thực hiện TLS Bootstrap và nhận về một thông tin
xác thực duy nhất, lưu tại `/etc/kubernetes/kubelet.conf`. Xong việc, **kubeadm xóa**
`bootstrap-kubelet.conf`. Vì vậy thiếu file bootstrap trên một node đã join **không phải lỗi** — nó
là bằng chứng quy trình đã chạy đúng. Bạn sẽ thấy lại đúng cặp này ở B8.3 trên hai worker vừa join
vào cluster mới.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện, không có dòng `FAIL:`.

---

## B5. Phá có kiểm soát trên control plane và khôi phục

> **Từ đây lab bắt đầu phá cluster.** Hai phép thử trong mục này là yêu cầu nguyên văn của
> **checkpoint giai đoạn 8**: xóa `/etc/kubernetes/manifests/kube-apiserver.yaml` rồi khôi phục, và
> tắt một control plane node rồi chứng minh cluster vẫn phục vụ. Cả hai chạy trên
> `lab-k8s-master` — đây là ngoại lệ đã khai báo ở [mục 2](#2-quy-ước-và-an-toàn).
> Cả hai đều **khôi phục được ngay trong mục này**; việc phá không hồi lại bắt đầu ở B6.

### B5.1. Xóa manifest của `kube-apiserver` và đưa nó trở lại

Ghi lại checksum trước khi đụng vào, để chứng minh file đưa lại là **đúng file cũ**:

```bash
. ~/lab-evidence/8a/vars.sh

sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml \
  | awk '{print $1}' | tee "$EV/b5-apiserver-sha-truoc.txt"

test "$(wc -c < "$EV/b5-apiserver-sha-truoc.txt")" -eq 65 \
  && echo 'PASS: da ghi checksum cua manifest kube-apiserver'
```

Bây giờ **chuyển file ra khỏi thư mục `manifests`** — không xóa hẳn, vì đây là bài học chứ không
phải tai nạn:

```bash
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /etc/kubernetes/kube-apiserver.yaml.off

for i in $(seq 1 60); do
  sudo crictl ps --name kube-apiserver -q | grep -q . || break
  sleep 2
done

sudo crictl ps -a --name kube-apiserver | tee "$EV/b5-crictl-apiserver.txt"

test -z "$(sudo crictl ps --name kube-apiserver -q)" \
  && echo 'PASS: khong con container kube-apiserver nao dang chay'
```

`kubectl` bây giờ phải chết, và **đó là điều kiện phải đạt**:

```bash
kubectl get nodes 2>&1 | head -3
if kubectl get nodes >/dev/null 2>&1; then
  echo 'FAIL: API van tra loi — manifest chua thuc su bi go'
else
  echo 'PASS: kubectl mat duong toi API server'
fi
```

**Trong lúc `kubectl` vô dụng, hai công cụ còn lại vẫn làm việc.** Đây đúng là tình huống mà
checkpoint giai đoạn 8 và bài [09](../09-troubleshooting-kubeadm-vi.md) yêu cầu bạn xoay xở được:

```bash
sudo crictl ps | tee "$EV/b5-crictl-con-lai.txt"
sudo journalctl -u kubelet -n 40 --no-pager | tee "$EV/b5-journal-kubelet.txt"

for c in etcd kube-scheduler kube-controller-manager; do
  grep -q "$c" "$EV/b5-crictl-con-lai.txt" \
    && echo "PASS: crictl van thay container $c dang chay" \
    || echo "FAIL: khong thay $c — su co rong hon mot manifest"
done
test -s "$EV/b5-journal-kubelet.txt" \
  && echo 'PASS: doc duoc log kubelet bang journalctl khi kubectl da chet'
```

Đưa file trở lại và chờ control plane lên:

```bash
sudo mv /etc/kubernetes/kube-apiserver.yaml.off /etc/kubernetes/manifests/kube-apiserver.yaml

sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml \
  | awk '{print $1}' > "$EV/b5-apiserver-sha-sau.txt"
diff -u "$EV/b5-apiserver-sha-truoc.txt" "$EV/b5-apiserver-sha-sau.txt" \
  && echo 'PASS: manifest dua lai la dung file cu, khong sai mot byte'

for i in $(seq 1 120); do
  test "$(kubectl get --raw='/readyz' 2>/dev/null)" = 'ok' && break
  sleep 2
done

test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok tro lai'
kubectl get pods -n kube-system -o wide | grep kube-apiserver
```

**Ý nghĩa:** phép thử này chứng minh điều B2.3 mới chỉ nói bằng lời. Desired state của control
plane **không nằm trong etcd** — nó nằm trong bốn file trên đĩa của một node. Bỏ file đi thì
kubelet dừng container tương ứng và không có controller nào dựng lại, vì mirror Pod thuộc sở hữu
của Node chứ không của Deployment. Đưa file lại thì kubelet phát hiện và chạy lại. Vòng lặp chờ ở
trên có điều kiện dừng chứ không có con số cam kết: thời gian kubelet quét lại thư mục manifest và
thời gian apiserver sẵn sàng **phụ thuộc cấu hình và phần cứng**.

Đây cũng là lý do bài [128](../128-api-server-bypass-risks-vi.md) của giai đoạn 9 xếp static Pod
vào nhóm rủi ro: ai ghi được vào `/etc/kubernetes/manifests` trên một control plane node thì chạy
được bất cứ thứ gì mà không đi qua API server, không qua RBAC, không qua admission.

**PASS:** dòng `PASS: da ghi checksum ...`, `PASS: khong con container kube-apiserver ...`,
`PASS: kubectl mat duong toi API server`, ba dòng `PASS: crictl van thay ...`, dòng
`PASS: doc duoc log kubelet ...`, dòng `PASS: manifest dua lai la dung file cu ...` và dòng
`PASS: readyz ok tro lai` — theo đúng thứ tự đó.

### B5.2. Tắt hẳn control plane node và chứng minh data plane vẫn phục vụ

Dựng workload trước khi tắt máy. Không có workload thì không có gì để chứng minh:

```bash
kubectl create namespace lab-8a

kubectl -n lab-8a create deployment web --image=busybox:1.37 --replicas=2 \
  -- sh -c 'mkdir -p /www && echo lab-8a-ok > /www/index.html && httpd -f -p 8080 -h /www'
kubectl -n lab-8a rollout status deploy/web --timeout=300s

kubectl -n lab-8a expose deploy web --name=web-np --type=NodePort --port=80 --target-port=8080
NP="$(kubectl -n lab-8a get svc web-np -o jsonpath='{.spec.ports[0].nodePort}')"
echo "NodePort=$NP"
echo "NP='$NP'" >> "$EV/vars.sh"

for ip in "$W1_IP" "$W2_IP"; do
  curl -s -o /dev/null -w "$ip -> %{http_code}\n" "http://$ip:$NP"
done

test "$(curl -s "http://$W1_IP:$NP")" = 'lab-8a-ok' \
  && test "$(curl -s "http://$W2_IP:$NP")" = 'lab-8a-ok' \
  && echo 'PASS: NodePort tra dung noi dung qua ca hai worker truoc khi tat master'

echo "GHI LAI DE DUNG TREN MAY HOST: W1_IP=$W1_IP W2_IP=$W2_IP NodePort=$NP"
```

**PASS:** dòng `PASS: NodePort tra dung noi dung qua ca hai worker ...` xuất hiện. Chép ba giá trị
ở dòng cuối ra chỗ khác — bạn sắp mất phiên shell này.

Trên **terminal của `lab-k8s-worker1`**, xác nhận đường tới API còn sống trước khi tắt:

```bash
timeout 3 bash -c "echo > /dev/tcp/lab-k8s-master/6443" \
  && echo 'PASS: API con mo truoc khi tat master' \
  || echo 'FAIL: API da khong toi duoc — dung tat may, xu ly theo muc 4'
```

Thay `6443` bằng đúng `API_PORT` mà B0.2 đọc được nếu cluster của bạn dùng cổng khác.

Bây giờ tắt master, trên **terminal của `lab-k8s-master`**:

```bash
sudo shutdown -h now
```

Chạy trên **máy host**, PowerShell, dùng lại `$vmrun` và `$vmx` từ [mục 2](#2-quy-ước-và-an-toàn):

```powershell
& $vmrun -T ws list
```

**PASS:** danh sách VM đang chạy **không còn** `lab-k8s-master`, và vẫn còn hai worker.

Ba phép đo trong lúc control plane vắng mặt. Trên **terminal của `lab-k8s-worker1`**:

```bash
if timeout 3 bash -c "echo > /dev/tcp/lab-k8s-master/6443"; then
  echo 'FAIL: API van mo — master chua tat han'
else
  echo 'PASS: API server da bien mat khoi mang'
fi

sudo crictl ps | grep -c . 
systemctl is-active kubelet containerd
```

**PASS:** dòng `PASS: API server da bien mat khoi mang` xuất hiện; `crictl ps` vẫn liệt kê
container; `kubelet` và `containerd` vẫn `active` — kubelet mất API server nhưng **không** dừng
container đang chạy.

Chạy trên **máy host**, PowerShell — thay ba giá trị bằng những gì bạn đã ghi lại:

```powershell
$w1 = '<W1_IP doc o B5.2>'
$w2 = '<W2_IP doc o B5.2>'
$np = '<NodePort doc o B5.2>'

foreach ($ip in @($w1, $w2)) {
  try {
    $r = Invoke-WebRequest -UseBasicParsing -Uri "http://${ip}:$np" -TimeoutSec 5
    if ($r.StatusCode -eq 200 -and $r.Content.Trim() -eq 'lab-8a-ok') { "PASS: $ip van phuc vu" }
    else { "FAIL: $ip tra ve $($r.StatusCode)" }
  } catch { "FAIL: $ip khong tra loi — $($_.Exception.Message)" }
}
```

**PASS:** đúng **hai** dòng `PASS: ... van phuc vu`, không dòng `FAIL:` nào.

**Ý nghĩa — và ranh giới thật của phép thử này.** Kết quả trên nói: **data plane vẫn phục vụ khi
control plane biến mất**. Pod đã chạy thì kubelet trên worker giữ chúng sống; quy tắc chuyển tiếp
mà kube-proxy đã ghi vào node vẫn còn nguyên nên NodePort vẫn forward. Thứ mất đi là khả năng
**thay đổi**: không lập lịch được Pod mới, không xử lý được node hỏng, không `kubectl` được gì.
Với topology một control plane, đó là toàn bộ những gì chứng minh được — bài
[02](../02-create-cluster-kubeadm-vi.md) gọi thẳng đây là *Hạn chế* của cluster một control plane
và đưa hai giải pháp: sao lưu etcd, hoặc dùng nhiều control plane node. Cái đầu là
[nợ #8](README.md#5-sổ-nợ-lab) chưa trả; cái sau là Lab 8b và Lab 8c.

Bật master lại. Chạy trên **máy host**:

```powershell
& $vmrun -T ws start $vmx[0]
& $vmrun -T ws list
```

SSH lại vào master, nạp lại biến, rồi chờ cluster hội tụ:

```bash
. ~/lab-evidence/8a/vars.sh

for i in $(seq 1 150); do
  test "$(kubectl get --raw='/readyz' 2>/dev/null)" = 'ok' && break
  sleep 2
done
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok sau khi master boot lai'

kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl -n lab-8a rollout status deploy/web --timeout=300s
kubectl get nodes -o wide
```

**PASS:** dòng `PASS: readyz ok sau khi master boot lai` xuất hiện; ba node `Ready`;
Deployment `web` vẫn đủ replica.

---

## B6. Gỡ cluster bằng `kubeadm reset` đúng thứ tự

> **Từ mục này trở đi cluster bị phá không hồi lại được bằng lệnh.** Đường về duy nhất là restore
> `03-storage-ready` ở B11. Đọc lại cảnh báo ở [mục 2](#2-quy-ước-và-an-toàn) trước khi gõ lệnh
> đầu tiên.

Bài [02](../02-create-cluster-kubeadm-vi.md), mục *Gỡ bỏ node*, quy định đúng ba bước và đúng thứ
tự: `kubectl drain` → `kubeadm reset` **trên chính node đó** → `kubectl delete node`.

`kubectl drain` là công cụ của [giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node)
và được [Lab 12](LAB-12-VAN-HANH-VONG-DOI-NODE.md) mổ xẻ đầy đủ — quan hệ với Eviction API, với
PodDisruptionBudget, khác biệt với `cordon`. Ở đây bạn dùng nó **đúng như bài 02 mô tả**, không hơn:
một bước dọn node trước khi rút nó ra. Trên cluster sắp bị phá, bước này gần như mang tính nghi
thức — nhưng chính vì vậy nó đáng làm đúng: trên cluster thật, bỏ `drain` nghĩa là kéo phích một
node còn đang phục vụ.

### B6.1. Rút hai worker

Chạy trên `lab-k8s-master`:

```bash
. ~/lab-evidence/8a/vars.sh

kubectl drain "$W2" --delete-emptydir-data --force --ignore-daemonsets \
  2>&1 | tee "$EV/b6-drain-w2.txt"

kubectl get node "$W2" -o jsonpath='{.spec.unschedulable}{"\n"}'
kubectl get pods -A -o wide --field-selector "spec.nodeName=$W2" | tee "$EV/b6-pods-w2-sau-drain.txt"

test "$(kubectl get node "$W2" -o jsonpath='{.spec.unschedulable}')" = 'true' \
  && echo 'PASS: worker2 da bi danh dau unschedulable'

DS_P="$(kubectl get pods -A --field-selector "spec.nodeName=$W2" \
  -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{"\n"}{end}' \
  | grep -c '^DaemonSet$')"
ALL_P="$(kubectl get pods -A --no-headers --field-selector "spec.nodeName=$W2" | wc -l)"
echo "Pod con lai tren worker2=$ALL_P, trong do thuoc DaemonSet=$DS_P"
test "$ALL_P" -eq "$DS_P" \
  && echo 'PASS: worker2 chi con Pod cua DaemonSet — dung dinh nghia node da drain xong'
```

**Ý nghĩa:** hai cờ trong lệnh trên đến thẳng từ bài 02. `--ignore-daemonsets` cần vì Pod DaemonSet
không đi đâu được — controller của chúng sẽ đặt lại ngay trên chính node đó, nên `drain` từ chối
chạy nếu bạn không nói rõ là chấp nhận bỏ qua chúng. `--force` xóa cả Pod **không có controller
quản lý**, tức xóa vĩnh viễn; ở đây nó an toàn vì cluster sắp bị gỡ, nhưng
[Lab 12](LAB-12-VAN-HANH-VONG-DOI-NODE.md#2-quy-ước-và-an-toàn) cấm dùng cờ này đúng vì lý do đó.

Trên **terminal của `lab-k8s-worker2`**:

```bash
sudo kubeadm reset -f 2>&1 | tee /tmp/b6-reset-w2.txt
sudo ls /etc/kubernetes/ 2>&1
systemctl is-active kubelet
```

**PASS:** lệnh kết thúc không lỗi; `/etc/kubernetes/` không còn `kubelet.conf` và không còn thư
mục `pki`; `kubelet` không còn `active` (nó quay lại vòng lặp chờ như
[A4.3](LAB-00-MOI-TRUONG-1.35.7.md#a43-cài-kubeadm-kubelet-kubectl-và-crictl) mô tả).

Quay lại `lab-k8s-master`:

```bash
kubectl delete node "$W2"
kubectl get nodes
test "$(kubectl get nodes --no-headers | wc -l)" -eq 2 \
  && echo 'PASS: cluster con hai node'
```

Lặp lại đúng ba bước đó cho `lab-k8s-worker1`:

```bash
kubectl drain "$W1" --delete-emptydir-data --force --ignore-daemonsets \
  2>&1 | tee "$EV/b6-drain-w1.txt"
```

Trên **terminal của `lab-k8s-worker1`**: `sudo kubeadm reset -f`. Rồi trên master:

```bash
kubectl delete node "$W1"
kubectl get nodes
kubectl -n lab-8a get pods -o wide

test "$(kubectl get nodes --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: chi con control plane node trong cluster'
```

**Ý nghĩa:** Pod của `web` giờ ở `Pending` và sẽ nằm im. Không phải vì scheduler hỏng — vì node
duy nhất còn lại mang taint `node-role.kubernetes.io/control-plane:NoSchedule` mà B2.4 đã đọc, và
Pod thường không có toleration cho nó. Đây là cùng một cơ chế nhìn từ chiều ngược lại: taint bảo vệ
control plane kể cả khi cluster không còn worker nào.

**PASS:** ba dòng `PASS:` của B6.1 xuất hiện (`unschedulable`, `chi con Pod cua DaemonSet`,
`cluster con hai node`) cộng dòng `PASS: chi con control plane node trong cluster`.

### B6.2. Reset control plane, và hai thứ `kubeadm reset` không đụng tới

Trước khi reset, đếm số quy tắc mà kube-proxy và CNI đã ghi vào `iptables`, và kiểm tra policy mặc
định để chắc chắn lệnh flush ở dưới không cắt mất phiên SSH của bạn:

```bash
sudo iptables -S | head -3
sudo iptables -S | grep -q '^-P INPUT ACCEPT' \
  && echo 'PASS: policy INPUT la ACCEPT, flush khong lam mat SSH' \
  || echo 'FAIL: policy INPUT khong phai ACCEPT — dung flush, xu ly theo muc 4'

RULES_TRUOC="$(sudo iptables-save | grep -cE 'KUBE-|CALI-')"
echo "so dong iptables cua Kubernetes/CNI truoc reset: $RULES_TRUOC"
```

Reset:

```bash
sudo kubeadm reset -f 2>&1 | tee "$EV/b6-reset-master.txt"
rm -f "$HOME/.kube/config"

RULES_SAU="$(sudo iptables-save | grep -cE 'KUBE-|CALI-')"
echo "so dong iptables sau reset: $RULES_SAU"

test "$RULES_SAU" -gt 0 \
  && echo 'PASS: reset KHONG don iptables — dung nhu bai 02 canh bao'
```

**Ý nghĩa:** bài 02 nói `kubeadm reset` là dọn dẹp **best-effort** và nêu đích danh hai thứ nó
không làm: **không đặt lại quy tắc iptables** và **không đặt lại bảng IPVS**. Con số `RULES_SAU` ở
trên là bằng chứng của vế thứ nhất. Vế thứ hai không áp dụng ở đây, và bạn kiểm được điều đó từ dữ
kiện đã ghi ở B0.4: chế độ của kube-proxy trên cluster này không phải `ipvs`, nên `ipvsadm -C`
không cần chạy — đúng lúc, vì `ipvsadm` không có trên node lab và
[mục 2](#2-quy-ước-và-an-toàn) cấm cài thêm phần mềm.

```bash
grep -E '^[[:space:]]*mode:' "$EV/b0-truoc.txt"
grep -qE '^[[:space:]]*mode: *ipvs' "$EV/b0-truoc.txt" \
  && echo 'FAIL: kube-proxy chay ipvs — can quy trinh don khac, doc lai bai 02' \
  || echo 'PASS: kube-proxy khong chay ipvs, khong can ipvsadm'
```

Dọn iptables bằng đúng lệnh bài 02 đưa ra, rồi dọn nốt hai bảng còn chain rỗng:

```bash
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
sudo iptables -t nat -X 2>/dev/null; sudo iptables -t mangle -X 2>/dev/null

sudo iptables-save | grep -cE 'KUBE-|CALI-'
```

Làm y hệt trên **hai worker**, mỗi node trên terminal của chính nó:

```bash
sudo iptables -S | grep -q '^-P INPUT ACCEPT' || echo 'FAIL: dung flush tren node nay'
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
sudo iptables -t nat -X 2>/dev/null; sudo iptables -t mangle -X 2>/dev/null
```

**PASS:** dòng `PASS: policy INPUT la ACCEPT ...`, dòng `PASS: reset KHONG don iptables ...` và
dòng `PASS: kube-proxy khong chay ipvs ...` xuất hiện. Số đếm cuối cùng trên mỗi node không tăng
lên so với trước khi flush; con số bằng `0` sẽ được gate lại sau reboot ở B6.4.

### B6.3. Dọn cấu hình CNI và gate "không còn cluster nào"

`kubeadm reset` in ra lời nhắc rằng nó không dọn cấu hình CNI. Làm nốt trên **cả ba node**, mỗi
node trên terminal của chính nó:

```bash
sudo ls -la /etc/cni/net.d/ 2>&1
sudo rm -rf /etc/cni/net.d/*
sudo ls -A /etc/cni/net.d/ 2>/dev/null | wc -l
```

Gate trên **từng node**:

```bash
test -z "$(sudo ls -A /etc/cni/net.d/ 2>/dev/null)" \
  && echo 'PASS: /etc/cni/net.d da rong'
sudo test ! -d /etc/kubernetes/pki \
  && echo 'PASS: khong con thu muc /etc/kubernetes/pki'
test -z "$(sudo ls -A /etc/kubernetes/manifests/ 2>/dev/null)" \
  && echo 'PASS: /etc/kubernetes/manifests da rong'
sudo test ! -d /var/lib/etcd \
  && echo 'PASS: khong con du lieu etcd tren node nay'
```

Trên master, thư mục `/var/lib/etcd` là thứ **thật sự chứa toàn bộ trạng thái cluster**; trên
worker nó vốn không tồn tại nên gate cuối tự đạt. Trên master, gate đó biến mất đúng nghĩa đen:
mọi object bạn từng tạo từ Lab 1a tới Lab 7b vừa đi cùng nó.

```bash
sudo crictl ps -a | tee "$EV/b6-crictl-sau-reset.txt"
grep -cE 'kube-apiserver|etcd|kube-scheduler|kube-controller-manager' "$EV/b6-crictl-sau-reset.txt"
```

**PASS:** trên mỗi node có đủ bốn dòng `PASS:` của khối gate; trên master, `crictl ps -a` không còn
container control plane nào ở trạng thái `Running`.

### B6.4. Reboot ba node — và dùng nó để kiểm chứng lại bài 01

Reboot không phải để "cho chắc". Nó xóa sạch quy tắc iptables còn sót và các interface mà CNI cũ
để lại, đồng thời **kiểm chứng đúng điều bài [01](../01-install-kubeadm-vi.md) nhấn mạnh**: tiền
điều kiện phải bền vững qua lần khởi động lại, không phải chỉ đúng lúc bạn gõ lệnh.

Trên **từng node**, theo thứ tự worker 2 → worker 1 → master:

```bash
sudo reboot
```

Sau khi cả ba lên lại, SSH vào từng node và chạy **cùng một khối gate**:

```bash
swapon --show
test -z "$(swapon --show)" && echo 'PASS: swap van tat sau reboot'

lsmod | grep -cE '^(overlay|br_netfilter)'
test "$(lsmod | grep -cE '^(overlay|br_netfilter)')" -eq 2 \
  && echo 'PASS: hai module kernel duoc nap lai sau reboot'

SYSCTL_OK="$(sysctl -n net.bridge.bridge-nf-call-iptables \
  net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward | grep -c '^1$')"
test "$SYSCTL_OK" -eq 3 && echo 'PASS: ba sysctl van bang 1 sau reboot'

systemctl is-active containerd
systemctl is-enabled containerd kubelet
test "$(systemctl is-active containerd)" = 'active' \
  && test "$(systemctl is-enabled kubelet)" = 'enabled' \
  && echo 'PASS: containerd active va kubelet enabled — dung trang thai truoc kubeadm init'

test "$(sudo iptables-save | grep -cE 'KUBE-|CALI-')" -eq 0 \
  && echo 'PASS: khong con quy tac iptables nao cua cluster cu'

LEFT=0
for i in cni0 flannel.1 vxlan.calico; do
  ip link show "$i" >/dev/null 2>&1 && { echo "FAIL: con interface $i"; LEFT=1; }
done
test "$LEFT" -eq 0 && echo 'PASS: khong con interface nao cua CNI cu'
```

**Ý nghĩa:** khối gate này là bài 01 đọc ngược. Nếu `/etc/fstab` còn dòng swap chưa comment thì
`swapon --show` sẽ có output ngay tại đây, và bước `kubeadm init` ở B7 sẽ chết ở preflight —
kubelet mặc định **không khởi động** trên node đang bật swap. Nếu module hoặc sysctl không bền
vững thì Pod networking sẽ hỏng sau khi cài CNI chứ không hỏng lúc init, và bạn sẽ mất thời gian
tìm sai chỗ. `kubelet` lúc này `enabled` nhưng **không** `active` là trạng thái đúng: nó đang ở
vòng lặp chờ `kubeadm` ra lệnh, y như A4.3 đã báo trước.

**PASS:** trên **mỗi** node, đủ sáu dòng `PASS:` của khối trên và không có dòng `FAIL:`.

---

## B7. Dựng lại control plane bằng `kubeadm init --config`

**Mục đích:** [A5.1 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) dựng cluster
bằng **cờ dòng lệnh**. Lần này bạn dựng bằng **file cấu hình** — cách thứ hai mà bài
[03](../03-control-plane-flags-vi.md) mô tả — để mọi quyết định nằm ở một chỗ đọc được, và để cơ
chế patch có chỗ áp vào.

### B7.1. Soạn file cấu hình từ những giá trị đã đọc ở B0.2

```bash
. ~/lab-evidence/8a/vars.sh

cat > "$WK/kubeadm-config.yaml" <<EOF
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: $MASTER_IP
  bindPort: $API_PORT
nodeRegistration:
  name: $MASTER
  criSocket: $CRI_SOCKET
patches:
  directory: $WK/patches
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: $K8S_BASELINE
controlPlaneEndpoint: $CP_ENDPOINT
networking:
  podSubnet: $POD_CIDR
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF

cat "$WK/kubeadm-config.yaml" | tee "$EV/b7-kubeadm-config.yaml"

test "$(grep -c '^kind:' "$WK/kubeadm-config.yaml")" -eq 3 \
  && ! grep -q '<' "$WK/kubeadm-config.yaml" \
  && grep -q "podSubnet: $POD_CIDR" "$WK/kubeadm-config.yaml" \
  && echo 'PASS: file cau hinh co du ba kind, khong con placeholder, va dung Pod CIDR baseline'
```

**Ý nghĩa — ba `kind` là ba phạm vi khác nhau, đây là toàn bộ bảng phân vai của bài 03:**

| `kind` | Phạm vi | Cái gì phải nằm ở đây |
| --- | --- | --- |
| `InitConfiguration` | **node này**, lúc `kubeadm init` | `advertiseAddress`, tên node, CRI socket, `patches.directory` |
| `ClusterConfiguration` | **toàn cluster** | `kubernetesVersion`, `controlPlaneEndpoint`, dải mạng, `extraArgs` của bốn thành phần control plane |
| `KubeletConfiguration` | **mọi kubelet**, làm cấu hình cơ sở | thứ giống nhau ở mọi node, ví dụ `cgroupDriver` |

Ba chi tiết đáng nhớ trong file trên:

- `controlPlaneEndpoint` là tên DNS chứ không phải IP. Bài [02](../02-create-cluster-kubeadm-vi.md)
  nói đây là điều kiện để **sau này** nâng lên HA — chuyển một cluster tạo mà **không có** cờ này
  thành HA là việc kubeadm **không hỗ trợ**. Giữ nguyên giá trị mà B0.2 đọc được, đừng đổi thành
  IP cho tiện.
- `serviceSubnet` **không** được khai. Đó là chủ ý: để kubeadm áp mặc định của nó, rồi B9 đọc lại
  giá trị hiệu lực từ flag thật của apiserver. Bảng A1.3 ghi Service CIDR chính là mặc định đó.
- `cgroupDriver: systemd` lặp lại đúng giá trị mà B1.4 đọc được trên ba node. Nó không đổi gì —
  nhưng nó biến một mặc định ngầm thành một quyết định viết ra giấy, và đó là mục đích của việc
  dùng file cấu hình.

**PASS:** dòng `PASS: file cau hinh co du ba kind ...` xuất hiện.

### B7.2. Preflight, kéo image, rồi `kubeadm init`

```bash
sudo kubeadm config images list --config "$WK/kubeadm-config.yaml" | tee "$EV/b7-images.txt"
grep -c "$K8S_BASELINE" "$EV/b7-images.txt"
test "$(grep -c "$K8S_BASELINE" "$EV/b7-images.txt")" -ge 1 \
  && echo 'PASS: danh sach image dung phien ban baseline, khong bi do remote'

sudo kubeadm config images pull --config "$WK/kubeadm-config.yaml" 2>&1 \
  | tee "$EV/b7-images-pull.txt"

sudo kubeadm init phase preflight --config "$WK/kubeadm-config.yaml" 2>&1 \
  | tee "$EV/b7-preflight.txt"
grep -qiE '\[ERROR|error execution phase preflight' "$EV/b7-preflight.txt" \
  && echo 'FAIL: preflight bao loi — doc muc 4 truoc khi init' \
  || echo 'PASS: preflight khong bao loi, may du dieu kien chay Kubernetes'
```

**Ý nghĩa:** phase `preflight` chính là "một loạt kiểm tra sơ bộ" mà bài
[02](../02-create-cluster-kubeadm-vi.md) mô tả, và nó kiểm gần hết danh sách của bài
[01](../01-install-kubeadm-vi.md): swap, cgroup, module kernel, port đang bận, phiên bản kernel qua
`SystemVerification`, container runtime. Chạy riêng phase này trước là cách rẻ nhất để biết B1 và
B6.4 có thật sự sạch hay không — thay vì phát hiện giữa chừng `kubeadm init`. Giữ
`--config` ở cả `images pull` lẫn `init` vì cùng lý do mà A5.1 giữ `--kubernetes-version`: bỏ đi
thì kubeadm có thể dò remote và chọn một patch khác với gói đã cài.

**PASS:** dòng `PASS: danh sach image dung phien ban baseline ...` và dòng
`PASS: preflight khong bao loi ...` xuất hiện.

Bây giờ chạy init. **Không ngắt lệnh này:**

```bash
sudo kubeadm init --config "$WK/kubeadm-config.yaml" 2>&1 | tee "$EV/b7-init-output.txt"

grep -q 'Your Kubernetes control-plane has initialized successfully' "$EV/b7-init-output.txt" \
  && echo 'PASS: kubeadm init bao thanh cong'
```

**PASS:** dòng `PASS: kubeadm init bao thanh cong` xuất hiện. Nếu lệnh treo hoặc báo lỗi, **đừng
chạy lại ngay** — sang [mục 4](#4-troubleshooting-của-lab-này) và tra bài
[09](../09-troubleshooting-kubeadm-vi.md) trước; chạy `kubeadm init` lần hai trên state bẩn sẽ đẻ
ra một triệu chứng khác che mất nguyên nhân gốc.

### B7.3. Đọc từng dòng output và tìm hiện vật mà mỗi phase để lại

Trước hết, rút danh sách phase từ chính output:

```bash
grep -oE '^\[[a-z-]+\]' "$EV/b7-init-output.txt" | sort -u | tee "$EV/b7-init-phases.txt"
wc -l < "$EV/b7-init-phases.txt"
```

Đọc từ trên xuống một lượt, rồi kiểm **hiện vật** thay vì kiểm chữ. Mỗi dòng gate dưới đây ứng với
một dòng bạn vừa đọc trong output:

```bash
sudo test -f /etc/kubernetes/pki/ca.crt \
  && echo 'PASS: phase certs — CA cua cluster da duoc sinh'
sudo test -f /etc/kubernetes/admin.conf && sudo test -f /etc/kubernetes/super-admin.conf \
  && echo 'PASS: phase kubeconfig — hai kubeconfig quan tri da duoc ghi'
sudo test -f /etc/kubernetes/manifests/etcd.yaml \
  && echo 'PASS: phase etcd — manifest static Pod cua etcd da duoc ghi'
sudo test -f /etc/kubernetes/manifests/kube-apiserver.yaml \
  && sudo test -f /etc/kubernetes/manifests/kube-controller-manager.yaml \
  && sudo test -f /etc/kubernetes/manifests/kube-scheduler.yaml \
  && echo 'PASS: phase control-plane — ba manifest con lai da duoc ghi'
sudo test -f /var/lib/kubelet/config.yaml && sudo test -f /var/lib/kubelet/kubeadm-flags.env \
  && echo 'PASS: phase kubelet-start — hai file cau hinh kubelet da duoc ghi'
```

Bốn hiện vật còn lại nằm **trong cluster**, nên phải có kubeconfig trước. Chạy lại đúng khối bốn
lệnh cấu hình kubeconfig cho user thường ở
[A5.1 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) — Lab 00 giải thích từng lệnh
một, lab này không chép lại — rồi:

```bash
kubectl config current-context
test "$(kubectl config current-context)" = 'kubernetes-admin@kubernetes' \
  && echo 'PASS: kubeconfig cua user thuong da tro dung cluster moi'

kubectl -n kube-system get cm kubeadm-config >/dev/null \
  && echo 'PASS: phase upload-config — ClusterConfiguration da len cluster'
kubectl -n kube-system get cm kubelet-config >/dev/null \
  && echo 'PASS: phase kubelet — KubeletConfiguration da len cluster'
test "$(kubectl get nodes -l 'node-role.kubernetes.io/control-plane' --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: phase mark-control-plane — node da mang label control-plane'
test "$(kubectl -n kube-system get secrets \
  --field-selector type=bootstrap.kubernetes.io/token --no-headers | wc -l)" -ge 1 \
  && echo 'PASS: phase bootstrap-token — token cho kubeadm join da ton tai'
kubectl -n kube-system get deployment coredns >/dev/null \
  && kubectl -n kube-system get daemonset kube-proxy >/dev/null \
  && echo 'PASS: phase addons — CoreDNS va kube-proxy da duoc trien khai'
```

**Ý nghĩa:** output của `kubeadm init` không phải một dòng chảy log — nó là **danh sách phase**, và
mỗi phase để lại đúng một loại hiện vật. Đọc theo cặp "dòng output ↔ hiện vật" là cách duy nhất để
sau này biết phải sửa ở đâu khi một phase hỏng: hỏng ở `certs` thì sửa trong `pki/`, hỏng ở
`kubelet-start` thì đọc `journalctl -u kubelet`, hỏng ở `wait-control-plane` thì dùng `crictl` như
B5.1 đã làm. Chính vì các phase độc lập nên `kubeadm init phase <ten>` chạy lại được từng phần —
đó là thứ bạn vừa dùng ở B7.2.

**PASS:** đủ **mười** dòng `PASS:` của mục này (năm dòng hiện vật trên đĩa, dòng kubeconfig, bốn
dòng hiện vật trong cluster).

### B7.4. Trạng thái ngay sau init: một node, chưa `Ready`

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide | tee "$EV/b7-pods-sau-init.txt"

test "$(kubectl get nodes --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: cluster moi co dung mot node'

TAINT_NEW="$(kubectl get node "$MASTER" -o jsonpath='{.spec.taints[*].key}')"
echo "taint: $TAINT_NEW"
case "$TAINT_NEW" in
  *node-role.kubernetes.io/control-plane*)
    echo 'PASS: taint control plane duoc dat lai mac dinh, khong can go tay' ;;
  *) echo 'FAIL: khong thay taint control plane' ;;
esac
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Node có thể còn `NotReady` — đó là trạng thái
đúng khi chưa có CNI, và B7.6 xử lý nó.

### B7.5. Chứng minh patch đã được áp

```bash
for i in $(seq 1 60); do
  sudo test -f /etc/kubernetes/manifests/kube-scheduler.yaml && break
  sleep 2
done

sudo grep -n 'lab-8a.local/patched-by' /etc/kubernetes/manifests/kube-scheduler.yaml
sudo grep -q 'lab-8a.local/patched-by' /etc/kubernetes/manifests/kube-scheduler.yaml \
  && echo 'PASS: patch da duoc ap vao manifest tren dia'

for i in $(seq 1 90); do
  kubectl -n kube-system get pod "kube-scheduler-$MASTER" >/dev/null 2>&1 && break
  sleep 2
done

kubectl -n kube-system get pod "kube-scheduler-$MASTER" -o yaml \
  | grep -n 'lab-8a.local/patched-by'
kubectl -n kube-system get pod "kube-scheduler-$MASTER" -o yaml \
  | grep -q 'lab-8a.local/patched-by' \
  && echo 'PASS: annotation cua patch xuat hien ca tren mirror Pod trong API'

sudo grep -q 'lab-8a.local/patched-by' /etc/kubernetes/manifests/kube-apiserver.yaml \
  && echo 'FAIL: patch lan sang thanh phan khac' \
  || echo 'PASS: patch chi cham dung target kube-scheduler'
```

**Ý nghĩa:** ba dòng gate trên chứng minh ba điều khác nhau. Dòng đầu: patch được áp **trước khi
manifest được ghi xuống đĩa**, đúng như bài 03 mô tả — bạn không sửa file sau khi kubeadm tạo nó,
kubeadm tạo file đã mang sẵn nội dung của bạn. Dòng thứ hai: kubelet đọc file đó và mirror Pod thừa
hưởng annotation, tức patch tác động thật chứ không chỉ là chữ trên đĩa. Dòng thứ ba: `target`
trong tên file quyết định phạm vi — `kube-scheduler0+merge.yaml` không chạm tới `kube-apiserver`.

Nhắc lại cái giá: nếu patch này là một tùy chỉnh thật, bạn sẽ **phải cung cấp lại** nó qua
`UpgradeConfiguration` ở mỗi lần `kubeadm upgrade`, nếu không nó biến mất sau nâng cấp.

**PASS:** đủ ba dòng `PASS:` của bước này, không có dòng `FAIL:`.

### B7.6. CoreDNS `Pending`, rồi cài CNI

Đọc trạng thái CoreDNS **trước** khi cài mạng — đây là mục mà bài
[09](../09-troubleshooting-kubeadm-vi.md) nói rõ là **bình thường chứ không phải lỗi**:

```bash
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide

PENDING="$(kubectl -n kube-system get pods -l k8s-app=kube-dns \
  -o jsonpath='{range .items[*]}{.status.phase}{"\n"}{end}' | grep -c '^Pending$')"
echo "so Pod CoreDNS dang Pending: $PENDING"
test "$PENDING" -ge 1 \
  && echo 'PASS: CoreDNS dang Pending vi chua co Pod network — dung thiet ke'

kubectl -n kube-system describe pod -l k8s-app=kube-dns \
  | grep -A3 'Events:' | tee "$EV/b7-coredns-pending.txt"
```

**Ý nghĩa:** bài [02](../02-create-cluster-kubeadm-vi.md) in đậm câu này: **DNS của cluster
(CoreDNS) sẽ không khởi động trước khi một mạng được cài đặt**. Bài 09 nhắc lại từ góc người gỡ
lỗi — kubeadm không phụ thuộc nhà cung cấp mạng nào, nên nó **cố ý** để CoreDNS chờ. Nếu bạn gặp
đúng triệu chứng này trên một cluster lạ, việc cần làm là cài CNI, không phải đi sửa CoreDNS.

Cài CNI bằng đúng khối lệnh của
[A5.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a52-cài-flannel-v0289), gồm cả hai lệnh đối chiếu
image và `net-conf.json` với bảng A1.3. Lab này **không chép lại URL, tên manifest hay số phiên bản
của CNI**.

> **Vì sao lại là CNI của Lab 00 chứ không phải CNI của mốc `03-storage-ready`.** Cluster bạn đang
> dựng là **cluster dùng một lần**: nó tồn tại để chứng minh bạn dựng được, rồi bị thay bằng
> snapshot ở B11. Cài lại CNI thực thi NetworkPolicy và ingress controller ở đây là chép lại
> [Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) mà chẳng thêm kiến thức nào của giai đoạn 8, lại
> tốn thêm cả giờ. Baseline A1.3 khóa sẵn một CNI đơn giản cho đúng việc này.

Sau khi CNI lên:

```bash
kubectl -n kube-system rollout status deployment coredns --timeout=600s
kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl -n kube-system get pods -o wide

test "$(kubectl -n kube-system get pods -l k8s-app=kube-dns \
  -o jsonpath='{range .items[*]}{.status.phase}{"\n"}{end}' | grep -c '^Running$')" -ge 1 \
  && echo 'PASS: CoreDNS chuyen sang Running ngay sau khi co Pod network'
test "$(kubectl get node "$MASTER" \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = 'True' \
  && echo 'PASS: control plane node da Ready'
```

**PASS:** ba dòng `PASS:` của mục này xuất hiện (`Pending` trước, `Running` sau, node `Ready`).

---

## B8. Join hai worker và chứng minh cluster mới chạy được

### B8.1. Ba thành phần của lệnh join, lần này để dùng thật

Trên `lab-k8s-master`:

```bash
. ~/lab-evidence/8a/vars.sh

sudo kubeadm token create --print-join-command > "$WK/join-command.txt"
chmod 600 "$WK/join-command.txt"
cat "$WK/join-command.txt"

NEW_HASH="$(tr ' ' '\n' < "$WK/join-command.txt" | grep '^sha256:' | head -1)"
CALC_HASH="sha256:$(sudo openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt \
  | openssl pkey -pubin -outform der 2>/dev/null \
  | openssl dgst -sha256 -hex | awk '{print $NF}')"

test "$NEW_HASH" = "$CALC_HASH" \
  && echo 'PASS: hash join khop CA moi — day la CA vua sinh o B7, khong phai CA cu'

OLD_HASH="$(awk -F= '/^ca-cert-hash=/{print $2}' "$EV/b2-join-info.txt")"
test "$NEW_HASH" != "$OLD_HASH" \
  && echo 'PASS: hash khac han cluster cu — CA moi, danh tinh cluster moi'
```

**Ý nghĩa:** hai gate trên nói cùng một điều từ hai phía: cluster này **không phải** cluster cũ.
`kubeadm init` sinh một CA hoàn toàn mới, nên mọi certificate, mọi kubeconfig và mọi hash join của
cluster trước đó đều vô giá trị. Đó cũng là lý do bạn phải xóa `$HOME/.kube/config` ở B6.2: giữ lại
file cũ thì `kubectl` sẽ báo lỗi certificate — đúng mục *Lỗi certificate TLS* của bài
[09](../09-troubleshooting-kubeadm-vi.md).

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B8.2. Verify trước, join sau

Đúng thứ tự mà [A5.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a53-join-hai-worker) đặt ra: kiểm
đường mạng trên từng worker **trước**, join **sau**. Join khi worker chưa resolve được master sẽ để
lại state bẩn phải `kubeadm reset` mới sửa được.

Trên **terminal của từng worker**, chạy khối verify của A5.3, rồi chạy lệnh in ra ở
`$WK/join-command.txt` bằng `sudo`. Đọc lệnh đó từ màn hình master; **không** chép token vào file
nào khác.

Quay lại master:

```bash
kubectl wait --for=condition=Ready node --all --timeout=600s
kubectl get nodes -o wide | tee "$EV/b8-nodes.txt"

test "$(kubectl get nodes --no-headers | wc -l)" -eq 3 \
  && echo 'PASS: cluster moi co du ba node'
test "$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}' \
  | grep -c '^True$')" -eq 3 \
  && echo 'PASS: ca ba node Ready'
test "$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.spec.podCIDR}{"\n"}{end}' | sort -u | wc -l)" -eq 3 \
  && echo 'PASS: ba node duoc cap ba PodCIDR khac nhau'
```

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B8.3. Dấu vết của `kubeadm join` trên hai worker — đọc lại bài 04

```bash
for n in "$W1" "$W2"; do
  if ssh "$n" 'sudo test -f /etc/kubernetes/bootstrap-kubelet.conf'; then
    echo "FAIL: $n con bootstrap-kubelet.conf"
  else
    echo "PASS: $n khong con bootstrap-kubelet.conf sau TLS Bootstrap"
  fi
  ssh "$n" 'sudo test -f /etc/kubernetes/kubelet.conf' \
    && echo "PASS: $n da co danh tinh rieng trong kubelet.conf"
done

CM_DRIVER2="$(kubectl -n kube-system get cm kubelet-config -o jsonpath='{.data.kubelet}' \
  | awk -F': *' '/^cgroupDriver/{print $2}')"
echo "cgroupDriver trong ConfigMap moi: $CM_DRIVER2"

BAD=0
for n in "$W1" "$W2"; do
  N_DRIVER="$(ssh "$n" "sudo awk -F': *' '/^cgroupDriver/{print \$2}' /var/lib/kubelet/config.yaml")"
  echo "$n cgroupDriver=$N_DRIVER"
  test "$N_DRIVER" = "$CM_DRIVER2" || { echo "FAIL: $n lech ConfigMap"; BAD=1; }
done
test "$BAD" -eq 0 \
  && echo 'PASS: hai worker tai xuong dung KubeletConfiguration ma B7.1 da khai'
```

**Ý nghĩa:** giá trị `cgroupDriver` bạn viết tay vào file cấu hình ở B7.1 vừa đi trọn một vòng:
`kubeadm init` ghi nó ra `/var/lib/kubelet/config.yaml` trên master **và** tải lên ConfigMap
`kubelet-config`; `kubeadm join` trên mỗi worker **tải nó xuống** rồi ghi ra file cùng đường dẫn.
Bạn không hề gõ lệnh nào về cgroup driver trên hai worker — đó chính là "lan truyền cấu hình cấp
cluster tới từng kubelet" của bài [04](../04-kubelet-integration-vi.md), lần này nhìn thấy được từ
đầu tới cuối.

**PASS:** hai dòng `PASS: ... khong con bootstrap-kubelet.conf`, hai dòng
`PASS: ... da co danh tinh rieng ...` và dòng `PASS: hai worker tai xuong dung ...` xuất hiện;
không có dòng `FAIL:`.

### B8.4. Cluster mới có chạy được thật không

Cluster này vừa đổi CNI so với mốc đầu vào, nên **không** được dừng ở gate ngắn A5.5. Chạy **toàn
bộ bảy tầng** của [A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) theo đúng thứ tự từ
dưới lên. Cluster bạn vừa dựng có cùng CNI với Lab 00, nên **không cần đọc lại tầng nào** — dùng
nguyên văn bảy tầng đó, kể cả bước dọn resource test ở
[A5.4.8](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot).
**Bỏ phần chụp snapshot ở cuối A5.4.8** — cluster này không được chụp mốc nào.

Sau khi bảy tầng PASS, dựng lại đúng workload mà checkpoint giai đoạn 8 yêu cầu: một Deployment có
Service:

```bash
kubectl create namespace lab-8a
kubectl -n lab-8a create deployment web --image=busybox:1.37 --replicas=2 \
  -- sh -c 'mkdir -p /www && echo lab-8a-ok > /www/index.html && httpd -f -p 8080 -h /www'
kubectl -n lab-8a rollout status deploy/web --timeout=300s
kubectl -n lab-8a expose deploy web --port=80 --target-port=8080

kubectl -n lab-8a run probe --image=busybox:1.37 --restart=Never --rm -i --quiet \
  -- wget -q -O- http://web.lab-8a.svc.cluster.local | tee "$EV/b8-service-test.txt"

grep -qx 'lab-8a-ok' "$EV/b8-service-test.txt" \
  && echo 'PASS: cluster tu dung chay duoc Deployment va goi duoc Service qua ten DNS'

kubectl get pods -A -o wide | tee "$EV/b8-pods-cluster-moi.txt"
```

**Ý nghĩa:** phép thử cuối cùng đi qua bốn thứ cùng lúc — scheduler đặt được Pod, CNI cho Pod nói
chuyện với nhau, CoreDNS phân giải được tên Service, và kube-proxy chuyển tiếp được tới ClusterIP.
Bốn thứ đó là định nghĩa thực dụng của "cluster dùng được", và đó cũng là lý do
[A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) không dừng ở `STATUS = Ready`.

**PASS:** bảy tầng của A5.4 đều PASS, và dòng
`PASS: cluster tu dung chay duoc Deployment va goi duoc Service qua ten DNS` xuất hiện.

**Đến đây bạn đã hoàn thành phần chính của checkpoint giai đoạn 8**: dựng cluster một control
plane hai worker, cài CNI, chạy được một Deployment có Service — và làm được bằng file cấu hình
chứ không phải bằng cách gõ theo hướng dẫn.

---

## B9. Bài 05 — họ địa chỉ của cluster, đọc trên cluster mình vừa dựng

**Mục đích:** bài [05](../05-dual-stack-support-vi.md) không dựng được trên mạng lab, nhưng phần
quan trọng nhất của nó **đọc được**: họ địa chỉ là quyết định chốt một lần lúc `kubeadm init`, và
bạn vừa tự tay đưa ra quyết định đó ở B7.1.

### B9.1. Ba nguồn độc lập cùng nói cluster này là single-stack

```bash
. ~/lab-evidence/8a/vars.sh

SVC_RANGE="$(sudo grep -oE '\-\-service-cluster-ip-range=[^[:space:]]+' \
  /etc/kubernetes/manifests/kube-apiserver.yaml | cut -d= -f2)"
ADV_ADDR="$(sudo grep -oE '\-\-advertise-address=[^[:space:]]+' \
  /etc/kubernetes/manifests/kube-apiserver.yaml | cut -d= -f2)"
CC_CIDR="$(sudo grep -oE '\-\-cluster-cidr=[^[:space:]]+' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml | cut -d= -f2)"
sudo grep -oE '\-\-node-cidr-mask-size[^[:space:]]*' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml \
  || echo '(khong dat --node-cidr-mask-size — dung mac dinh)'

printf 'service-cluster-ip-range=%s\nadvertise-address=%s\ncluster-cidr=%s\n' \
  "$SVC_RANGE" "$ADV_ADDR" "$CC_CIDR" | tee "$EV/b9-address-family.txt"

case "$SVC_RANGE" in *,*) echo 'FAIL: Service CIDR co hai dai' ;;
  *) echo 'PASS: Service CIDR chi co mot dai — single-stack' ;; esac
case "$CC_CIDR" in *,*) echo 'FAIL: cluster CIDR co hai dai' ;;
  *) echo 'PASS: cluster CIDR chi co mot dai — single-stack' ;; esac
test "$CC_CIDR" = "$POD_CIDR" \
  && echo 'PASS: cluster CIDR dung bang podSubnet ma B7.1 khai'
test "$ADV_ADDR" = "$MASTER_IP" \
  && echo 'PASS: advertise-address dung bang advertiseAddress ma B7.1 khai'
```

Nguồn thứ hai — Node object:

```bash
for n in $NODES; do
  echo -n "$n podCIDRs: "
  kubectl get node "$n" -o jsonpath='{.spec.podCIDRs}{"\n"}'
done | tee "$EV/b9-podcidrs.txt"

BAD=0
for n in $NODES; do
  CNT="$(kubectl get node "$n" -o jsonpath='{.spec.podCIDRs[*]}' | wc -w)"
  test "$CNT" -eq 1 || { echo "FAIL: $n co $CNT dai podCIDR"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: moi node chi duoc cap dung mot dai podCIDR'
```

Nguồn thứ ba — Service:

```bash
kubectl -n kube-system get svc kube-dns \
  -o jsonpath='ipFamilies={.spec.ipFamilies} ipFamilyPolicy={.spec.ipFamilyPolicy}{"\n"}'

test "$(kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.ipFamilies[*]}' | wc -w)" -eq 1 \
  && echo 'PASS: Service chi co mot ho dia chi'
test "$(kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.ipFamilyPolicy}')" = 'SingleStack' \
  && echo 'PASS: ipFamilyPolicy la SingleStack'
```

**Ý nghĩa:** bài dạy cú pháp dual-stack bằng **danh sách hai dải cách nhau bằng dấu phẩy** ở
`--pod-network-cidr`, `--service-cidr`, và ở `node-ip` trong `nodeRegistration.kubeletExtraArgs`.
Ba phép đọc trên chứng minh cluster của bạn dùng đúng cú pháp đó với **một** phần tử — nghĩa là
single-stack **không phải** một chế độ khác, nó là dual-stack với danh sách dài một. Chính bài xác
nhận điều này ở mục *Tạo một cluster single-stack*: bạn triển khai được cluster single-stack mà
tính năng mạng dual-stack **vẫn được bật**.

Một ranh giới dễ nhầm mà bạn vừa kiểm chứng: `advertiseAddress` — tương đương cờ
`--apiserver-advertise-address` — chỉ nói API Server quảng bá nó lắng nghe ở địa chỉ nào, và cờ này
**không hỗ trợ dual-stack**. Nó không phải chỗ khai họ địa chỉ của cluster; chỗ đó là
`networking.podSubnet` và `networking.serviceSubnet`.

**PASS:** bốn dòng `PASS:` ở khối thứ nhất, dòng `PASS: moi node chi duoc cap dung mot dai
podCIDR`, và hai dòng `PASS:` ở khối Service; không có dòng `FAIL:`.

### B9.2. Điều kiện IPv6 mà cluster này chưa có

```bash
for n in $NODES; do
  echo -n "$n: "
  ssh "$n" 'sysctl -n net.ipv6.conf.all.forwarding'
done | tee "$EV/b9-ipv6-forwarding.txt"

test -s "$EV/b9-ipv6-forwarding.txt" \
  && echo 'PASS: da ghi trang thai chuyen tiep IPv6 cua ba node'
```

**Ý nghĩa:** bài đặt `net.ipv6.conf.all.forwarding = 1` làm **điều kiện tiên quyết** của mọi node
trong cluster dual-stack — bản song sinh IPv6 của `net.ipv4.ip_forward` mà
[A4.1 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites)
đã đặt. Giá trị bạn vừa đọc là trạng thái thật của node lab, chưa được baseline đặt bền vững; nếu
sau này dựng cluster dual-stack, đây là dòng đầu tiên phải sửa.

Phần còn lại của bài **không kiểm chứng được ở đây và sẽ không kiểm chứng được trên bộ VM này**:
mạng `A1.2` chỉ có IPv4, và bài nói rõ `kubeadm upgrade` **không hỗ trợ thay đổi** cluster CIDR hay
Service CIDR của cluster đã có — nên đây là quyết định một chiều giống hệt `--control-plane-endpoint`,
chỉ đảo lại được bằng cách dựng cluster mới. Lý do đầy đủ nằm ở bảng
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

**PASS:** dòng `PASS: da ghi trang thai chuyen tiep IPv6 cua ba node` xuất hiện.

---

## B10. Bài 214 — định vị các việc quản trị còn lại

**Mục đích:** [214](../214-kubeadm-tasks-vi.md) là **trang mục lục** của nhóm task
*Administration with kubeadm*. Dùng nó đúng công dụng: đánh dấu việc nào bạn vừa làm, và việc nào
có chỗ đứng ở giai đoạn sau.

```bash
sudo kubeadm token list | tee "$EV/b10-token-list.txt"
test "$(grep -vc '^TOKEN' "$EV/b10-token-list.txt")" -ge 1 \
  && echo 'PASS: con it nhat mot bootstrap token — dung thu ma B8 vua dung'

sudo kubeadm certs check-expiration | tee "$EV/b10-certs.txt"
grep -qi 'invalid\|expired' "$EV/b10-certs.txt" \
  && echo 'FAIL: cluster moi da co certificate het han — bat thuong' \
  || echo 'PASS: khong certificate nao cua cluster moi da het han'

kubeadm version -o short
kubectl get --raw /version
```

| Trang con của 214 | Trạng thái sau lab này |
| --- | --- |
| [215 — Thêm worker node Linux](../215-adding-linux-nodes-vi.md) | **đã làm** ở B8; làm lại ở [giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node) ở góc vòng đời node |
| [218 — Cấu hình cgroup driver](../218-configure-cgroup-driver-vi.md) | **đã đọc** ở B1.4 và **đã khai** ở B7.1; đổi nó trên cluster đang chạy thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| [219 — Quản lý certificate với kubeadm](../219-kubeadm-certs-vi.md) | mới **đọc hạn** ở B2.2 và ở đây; gia hạn và xoay CA là [nợ #7](README.md#5-sổ-nợ-lab), trả ở [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) |
| [220 — Cấu hình lại một cluster kubeadm](../220-kubeadm-reconfigure-vi.md) | **chưa làm**. Lab này đổi cấu hình bằng cách **dựng lại**, không phải bằng cách cấu hình lại cluster đang chạy — hai quy trình khác hẳn nhau; xem [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| [216 — Thêm worker node Windows](../216-adding-windows-nodes-vi.md) | cần một VM Windows; Lab 15 |
| [217](../217-change-package-repository-vi.md), [221](../221-kubeadm-upgrade-vi.md), [222](../222-upgrading-linux-nodes-vi.md), [223](../223-upgrading-windows-nodes-vi.md) | toàn bộ thuộc [giai đoạn 17](../00-ALO-TRINH-ADMIN.md#giai-đoạn-17--nâng-cấp-cluster). **Đừng chạy `kubeadm upgrade` trong lab này** — nó sẽ phá baseline A1.3 |

**Ý nghĩa:** trang 214 mở đầu bằng đúng một câu phân định: các tác vụ ở đây dành cho người **đang
quản trị một cluster có sẵn**. Lab 8a vừa kết thúc phần "tạo cluster"; mọi thứ trong danh sách trên
là phần "sống cùng cluster", và lộ trình đã chia chúng vào từng giai đoạn của
[Phần II](../00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster).

**PASS:** dòng `PASS: con it nhat mot bootstrap token ...` và dòng
`PASS: khong certificate nao cua cluster moi da het han` xuất hiện.

---

## B11. Xuất bằng chứng, restore `03-storage-ready` và gate cuối

> **Mục này là điểm kết thúc bắt buộc của lab.** Cluster bạn vừa dựng ở B7–B8 **không** phải mốc
> của chuỗi lab: nó thiếu CNI thực thi NetworkPolicy, thiếu ingress controller, thiếu StorageClass
> mặc định. Mọi lab sau bắt đầu từ `03-storage-ready`, nên bỏ mục này là làm hỏng đầu vào của
> [Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md),
> [Lab 9b](LAB-9B-POD-SECURITY-VA-HARDENING.md) và
> [Lab 11a](LAB-11A-OBSERVABILITY.md).

### B11.1. Hồ sơ cuối của cluster tạm

```bash
. ~/lab-evidence/8a/vars.sh

{
  echo "=== $(date -Is) — cluster tam do Lab 8a dung ==="
  echo '--- nodes'
  kubectl get nodes -o wide
  echo '--- node creationTimestamp'
  kubectl get nodes \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}'
  echo '--- pods'
  kubectl get pods -A -o wide
  echo '--- storageclass'
  kubectl get storageclass 2>&1
  echo '--- ingressclass'
  kubectl get ingressclass 2>&1
} | tee "$EV/b11-cluster-tam.txt"

NEW_CT="$(kubectl get node "$MASTER" -o jsonpath='{.metadata.creationTimestamp}')"
echo "creationTimestamp cu=$NODE_CT | moi=$NEW_CT"
test "$NEW_CT" != "$NODE_CT" \
  && echo 'PASS: Node object cua cluster tam KHAC cluster cu — dung nhu du kien'

kubectl get storageclass --no-headers 2>/dev/null | wc -l
kubectl get ingressclass --no-headers 2>/dev/null | wc -l
```

**Ý nghĩa:** hai con số cuối gần như chắc chắn là `0`. Đó chính là lý do lab kết thúc bằng restore
chứ không bằng "dựng lại cho giống": để có `03-storage-ready` bạn phải chạy lại trọn vẹn
[Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) rồi [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md) —
sáu tới tám giờ, và kết quả vẫn không byte-for-byte giống mốc cũ. `NEW_CT` khác `NODE_CT` là bằng
chứng số hóa của việc đó: đây là một cluster khác, mang danh tính khác.

**PASS:** dòng `PASS: Node object cua cluster tam KHAC cluster cu ...` xuất hiện.

### B11.2. Chép bằng chứng ra máy host trước khi restore xóa sạch

Toàn bộ `~/lab-evidence/8a/` nằm trên đĩa của VM master và **sẽ biến mất** khi snapshot được
restore. Chạy trên **máy host**, PowerShell:

```powershell
$masterUser = 'ubuntu'
$masterHost = '<IP hoac hostname cua lab-k8s-master theo bang A1.2 cua Lab 00>'
$dest = 'E:\lab-evidence\8a'

if ($masterHost -like '*<*') { throw 'FAIL: chua dien dia chi that cua master' }
New-Item -ItemType Directory -Force -Path $dest | Out-Null
scp -r "${masterUser}@${masterHost}:~/lab-evidence/8a/*" $dest

$n = (Get-ChildItem -Path $dest -File).Count
"so file da chep: $n"
if ((Test-Path "$dest\b0-truoc.txt") -and ($n -ge 15)) { 'PASS: bang chung da nam tren may host' }
else { 'FAIL: chua chep du — chua duoc restore' }
```

**PASS:** dòng `PASS: bang chung da nam tren may host` xuất hiện. **Không restore khi còn dòng
`FAIL:`** — bằng chứng mất rồi thì không dựng lại được, và `b0-truoc.txt` là thứ B11.4 dùng để
chứng minh cluster đã về đúng chỗ cũ.

Lệnh trên **không** chép `~/lab-work/8a/`, và đó là chủ ý: thư mục đó chứa
`join-command.txt` với bootstrap token thật. Token sẽ chết cùng cluster tạm ở bước sau.

### B11.3. Tắt ba VM và restore về `03-storage-ready`

Tắt sạch trước khi restore, đúng thứ tự worker 2 → worker 1 → master. Trên **từng node**:

```bash
sudo shutdown -h now
```

Chạy trên **máy host**, dùng lại `$vmrun` và `$vmx` từ [mục 2](#2-quy-ước-và-an-toàn):

```powershell
& $vmrun -T ws list
```

**PASS:** danh sách VM đang chạy **không còn** VM nào trong `$vmx`.

Kiểm lại mốc một lần nữa rồi mới revert — đây là gate cuối cùng còn cứu được bạn:

```powershell
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '03-storage-ready') { "PASS: $f co moc" }
  else { "FAIL: $f KHONG co moc — dung revert" }
}
```

**PASS:** đúng ba dòng `PASS: ... co moc`.

Revert cả ba VM, rồi bật theo thứ tự master → worker 1 → worker 2:

```powershell
foreach ($f in $vmx) { & $vmrun -T ws revertToSnapshot $f '03-storage-ready' }

& $vmrun -T ws start $vmx[0]
& $vmrun -T ws start $vmx[1]
& $vmrun -T ws start $vmx[2]
& $vmrun -T ws list
```

**PASS:** cả ba VM xuất hiện trong `vmrun list`. Không revert riêng một VM và giữ hai VM còn lại ở
state mới — với cluster một control plane, restore là thao tác trên **cả ba VM cùng một mốc**, đúng
ghi chú cuối [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).

Chép `b0-truoc.txt` ngược lại vào master để B11.4 so sánh được. Chạy trên **máy host** sau khi
master đã boot xong và `ssh` vào được:

```powershell
scp "$dest\b0-truoc.txt" "${masterUser}@${masterHost}:~/b0-truoc.txt"
```

### B11.4. Gate cuối — chứng minh cluster đã về đúng `03-storage-ready`

SSH vào `lab-k8s-master`. **Biến của lab đã biến mất cùng snapshot**, đặt lại ba tên node:

```bash
MASTER='lab-k8s-master'
W1='lab-k8s-worker1'
W2='lab-k8s-worker2'
```

**Bước 1 — gate ngắn A5.5.** Chạy khối lệnh ở
[A5.5 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab).

**PASS:** ba node `Ready`; dòng `PASS: readyz ok` xuất hiện; lệnh field selector trả
`No resources found`; CoreDNS đủ replica `READY`; `default` không có Pod.

**Bước 2 — bảy tầng A5.4.** Lab này đã đụng vào CNI và vào toàn bộ control plane, nên đây đúng
trường hợp mà A5.5 nói **phải chạy cả bảy tầng**, không được dừng ở gate ngắn. Chạy trọn bảy tầng
của [A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) từ dưới lên, kể cả bước dọn
resource test ở
[A5.4.8](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot);
**bỏ phần chụp snapshot** — lab này không tạo mốc mới.

Cluster sau restore mang CNI của Lab 5b chứ không phải của Lab 00, nên **hai tầng phải đọc lại**
theo đúng bảng ở
[mục B11.2 của Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md#b112-chạy-trọn-bảy-tầng-gate-a54):
tầng 2 (tên DaemonSet của CNI) và tầng 3 (nguyên nhân fail khi Pod không nói chuyện xuyên node).
Tập namespace hệ thống hợp lệ ở mốc này là `kube-system`, namespace của CNI và của ingress
controller do Lab 5b cài, **cộng** namespace của dynamic provisioner do Lab 6a cài.

**PASS:** bảy tầng đều PASS với hai tầng đã đọc lại như trên.

**Bước 3 — ba thành phần hạ tầng của mốc phải còn nguyên:**

```bash
kubectl get storageclass | tee ~/b11-sc.txt
kubectl get ingressclass | tee ~/b11-ic.txt

test "$(kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' \
  | grep -c '^true$')" -eq 1 \
  && echo 'PASS: co dung mot StorageClass mac dinh'
test "$(kubectl get ingressclass --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: co dung mot IngressClass'
test "$(kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded \
  --no-headers 2>/dev/null | wc -l)" -eq 0 \
  && echo 'PASS: khong Pod nao ngoai trang thai Running/Succeeded'
```

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

**Bước 4 — chứng minh đây là cluster CŨ, không phải cluster bạn vừa dựng.** Đây là gate quan trọng
nhất của mục, vì ba bước trên chỉ nói cluster *khỏe*, chưa nói cluster *đúng*:

```bash
NODE_CT_SAU="$(kubectl get node "$MASTER" -o jsonpath='{.metadata.creationTimestamp}')"
NODE_CT_TRUOC="$(grep -A4 'node creationTimestamp' ~/b0-truoc.txt | grep "$MASTER" | awk '{print $2}')"
echo "creationTimestamp truoc lab=$NODE_CT_TRUOC | sau restore=$NODE_CT_SAU"
test -n "$NODE_CT_TRUOC" && test "$NODE_CT_SAU" = "$NODE_CT_TRUOC" \
  && echo 'PASS: Node object dung la object cu — restore da tra ve dung cluster'

sudo grep -q 'lab-8a.local/patched-by' /etc/kubernetes/manifests/kube-scheduler.yaml \
  && echo 'FAIL: van con patch cua cluster tam — chua restore hoac restore sai moc' \
  || echo 'PASS: manifest kube-scheduler khong con patch cua cluster tam'

kubectl get pods -A -o wide | grep -i flannel \
  && echo 'FAIL: van con CNI cua cluster tam' \
  || echo 'PASS: khong con dau vet CNI cua cluster tam'

kubectl get namespace lab-8a >/dev/null 2>&1 \
  && echo 'FAIL: namespace lab-8a van ton tai' \
  || echo 'PASS: namespace lab-8a khong ton tai'

test ! -d ~/lab-evidence/8a \
  && echo 'PASS: thu muc bang chung tren VM da bien mat — dung nhu B11.2 canh bao'
```

**Ý nghĩa:** bốn phép kiểm này bắt bốn cách restore hỏng khác nhau. `creationTimestamp` khác nghĩa
là bạn đang nhìn một cluster khác. Annotation patch còn sót nghĩa là chưa revert. Dấu vết CNI của
Lab 00 nghĩa là revert nhầm về `01-cluster-ready`. Namespace `lab-8a` còn nghĩa là revert nhầm về
một mốc chụp giữa chừng.

**PASS:** năm dòng `PASS:` của bước này xuất hiện, không có dòng `FAIL:`.

**Bước 5 — đối chiếu hồ sơ trước và sau:**

```bash
{
  echo "=== $(date -Is) — sau Lab 8a, da restore 03-storage-ready ==="
  echo '--- nodes'
  kubectl get nodes -o wide
  echo '--- node creationTimestamp'
  kubectl get nodes \
    -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}'
  echo '--- pods'
  kubectl get pods -A -o wide
  echo '--- storageclass'
  kubectl get storageclass
  echo '--- ingressclass'
  kubectl get ingressclass
  echo '--- namespaces'
  kubectl get namespaces
  echo '--- kube-proxy mode'
  kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' \
    | grep -E '^[[:space:]]*mode:'
} | tee ~/b11-sau.txt

diff -u ~/b0-truoc.txt ~/b11-sau.txt > ~/b11-diff.txt 2>&1 || true
grep -c '^[+-]' ~/b11-diff.txt
sed -n '1,40p' ~/b11-diff.txt
```

**PASS:** `b11-diff.txt` chỉ khác ở những chỗ **được phép**: mốc thời gian ở dòng tiêu đề, cột
`AGE` của object, và `RESTARTS` của Pod hạ tầng. Khác biệt **không** hợp lệ là: thiếu một node,
thiếu StorageClass hoặc IngressClass, `creationTimestamp` của node đổi, xuất hiện namespace lạ.
Gặp bất kỳ cái nào thì xử lý theo [mục 4](#4-troubleshooting-của-lab-này) trước khi sang lab sau.

Dọn hai file tạm còn lại trên master:

```bash
rm -f ~/b0-truoc.txt ~/b11-sau.txt ~/b11-diff.txt ~/b11-sc.txt ~/b11-ic.txt
ls ~ | tee /dev/null
test ! -f ~/b11-diff.txt && echo 'PASS: khong con file tam cua lab tren home cua master'
```

**PASS:** dòng `PASS: khong con file tam cua lab tren home cua master` xuất hiện.

Cluster đã trở về `03-storage-ready`. **Lab 8a không tạo snapshot mới** — để ba VM nguyên trạng
đang chạy hoặc tắt tùy bạn; lab sau tự bật máy theo
[A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab). Bằng chứng của lab nằm trên máy
host tại thư mục bạn chọn ở B11.2, **không** nằm trên VM.

---

## 3. Checkpoint 8a

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Bạn clone một VM từ node lab rồi đổi hostname và IP, sau đó `kubeadm join`. Bài 01 nói ba thứ
      phải duy nhất cho mỗi node — kể đủ ba, và nói thứ nào **không sửa được từ bên trong Ubuntu**.
      Vì sao Kubernetes cần chúng?
- [ ] `swapoff -a` rồi `kubeadm init` thành công. Sau lần reboot đầu tiên kubelet không lên nữa.
      Chuyện gì đã xảy ra, hành vi mặc định nào của kubelet gây ra nó, và **hai** lối thoát mà bài
      01 đưa ra là gì? Nếu chọn lối thứ hai, workload có tự động dùng được swap không?
- [ ] Control plane chạy v1.35.x. Bạn được phép để kubelet trên một worker ở v1.36 không? Còn
      v1.34? Quy tắc này **bất đối xứng** ở chỗ nào, và vì sao "cứ mới hơn là an toàn" sai ở đây?
- [ ] Bạn `kubectl delete pod kube-apiserver-<master>`. Chuyện gì xảy ra? Bạn `mv` file
      `/etc/kubernetes/manifests/kube-apiserver.yaml` đi chỗ khác. Chuyện gì xảy ra? Vì sao hai câu
      trả lời khác nhau, và `ownerReferences` của mirror Pod trỏ về đâu?
- [ ] Lệnh `kubeadm join` có ba thành phần. Mỗi thành phần trả lời câu hỏi gì, cái nào là **bí
      mật**, và bạn tính lại `--discovery-token-ca-cert-hash` từ file nào bằng công cụ gì? Nếu ai
      đó có token nhưng dùng hash sai thì hỏng ở phía nào?
- [ ] Bạn muốn `kube-apiserver` trên hai control plane node ghi audit log vào hai đường dẫn khác
      nhau. Đặt hai giá trị vào `ClusterConfiguration.apiServer.extraArgs` có ra kết quả không? Vì
      sao? Cơ chế nào giải được, nó được áp vào **thời điểm nào**, và cái giá phải trả khi nâng cấp
      là gì?
- [ ] Kể tên các file mà `kubeadm init` ghi cho kubelet, và nói file nào được đưa **lên cluster**
      để node join sau lấy về. Trên một worker vừa join, `bootstrap-kubelet.conf` không tồn tại —
      lỗi hay đúng? Nếu cùng một cờ xuất hiện ở `kubeadm-flags.env` và `/etc/default/kubelet` thì
      cái nào thắng, và bạn đọc ở đâu để biết?
- [ ] `kubeadm init` xong, `kubectl get pods -A` cho thấy CoreDNS `Pending`. Đây là lỗi hay là
      thiết kế? Bài 09 xếp nó vào đâu, và việc cần làm là gì? Điều gì trong kiến trúc kubeadm buộc
      phải như vậy?
- [ ] `kubeadm reset` là dọn dẹp **best-effort**. Kể đúng hai thứ bài 02 nói nó **không** đặt lại,
      cách xử lý từng thứ, và làm sao bạn biết cluster của mình cần cách nào. Ngoài hai thứ đó, còn
      thư mục nào bạn phải tự dọn?
- [ ] Bạn tắt control plane node duy nhất. Ứng dụng đang chạy còn phục vụ không, và **vì sao**?
      Chính xác thì bạn mất khả năng gì? Muốn câu trả lời đổi thành "API vẫn phục vụ" thì cần thay
      đổi gì về topology, và bài nào của giai đoạn 8 dạy điều đó?
- [ ] `--apiserver-advertise-address` và `--control-plane-endpoint` khác nhau thế nào? Cái nào là
      **quyết định một chiều**, hệ quả của việc bỏ qua nó là gì, và vì sao Lab 00 đặt nó ở dạng tên
      DNS chứ không phải IP?
- [ ] **Nợ #8 phát sinh ở lab này và chưa được trả.** Món nợ đó là gì, vì sao dựng lại được cluster
      vẫn chưa đủ để coi là biết khôi phục, và trong lab này bạn đã khôi phục bằng cơ chế nào thay
      thế? Nó được trả ở giai đoạn nào, bằng công cụ gì?

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Một cluster ra đời như thế nào.** Bắt đầu từ ba máy Ubuntu trống. Kể đủ bốn chặng: máy phải
   thỏa những điều kiện gì trước khi kubeadm chịu chạy và **vì sao từng điều kiện tồn tại**;
   `kubeadm init` đi qua những phase nào và **mỗi phase để lại hiện vật gì** — nói được ít nhất
   năm hiện vật và nơi chúng nằm; vì sao phải cài CNI và vì sao CoreDNS chờ tới lúc đó; worker
   join bằng cách nào, hai bên xác thực lẫn nhau ra sao, và cấu hình kubelet của nó đến từ đâu
   trong khi bạn chưa gõ lệnh nào về kubelet trên máy đó. Kết bằng câu trả lời cho: nếu chỉ được
   giữ **một** quyết định trong `kubeadm init` mà không sửa được về sau, đó là quyết định nào và
   vì sao.
2. **Vì sao lab này kết thúc bằng restore chứ không bằng cluster bạn vừa dựng.** Đi qua ba ý:
   cluster tạm thiếu chính xác những gì so với `03-storage-ready` và mỗi thứ do lab nào cài; bốn
   bằng chứng đọc được ở B11.4 chứng minh cluster sau restore là **object cũ** chứ không phải
   object mới; và điều gì sẽ hỏng ở Lab 9a, 9b, 11a nếu bạn bỏ qua bước restore. Cuối cùng, nói rõ
   **một chủ đề của giai đoạn 8 mà lab này cố ý không làm**, vì sao, và nó được trả ở đâu.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm *static Pod* với *Pod thường*,
*`--apiserver-advertise-address`* với *`--control-plane-endpoint`*, *`ClusterConfiguration`* với
*`KubeletConfiguration`*, hay *`kubeadm reset`* với *khôi phục cluster* — Lab 8a và phần bài
01–05, 09 của
[giai đoạn 8](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) hoàn tất. Phần HA
của giai đoạn — bài [06](../06-ha-topology-vi.md), [07](../07-setup-ha-etcd-with-kubeadm-vi.md),
[08](../08-high-availability-vi.md) — nằm ở Lab 8b và Lab 8c, chạy trên **bộ VM riêng** và không
đụng tới chuỗi snapshot chính.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).

**Sự cố của chính kubeadm thì tra bài [09](../09-troubleshooting-kubeadm-vi.md), không tra ở đây.**
Đó là tài liệu tra cứu chính thức, mỗi mục `##` là **một triệu chứng**; cách dùng đúng là đọc tên
mục cho tới khi thấy triệu chứng khớp hình dạng với thứ bạn đang gặp. Bảng dưới chỉ liệt kê sự cố
phát sinh từ **cách lab này sắp xếp các bước**, và với mỗi dòng, cột cuối chỉ thẳng sang mục tương
ứng của bài 09 khi có.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| **B5.1: `kubectl` không trở lại sau khi `mv` file về chỗ cũ** | `sudo crictl ps -a --name kube-apiserver`; `sudo journalctl -u kubelet -n 80 --no-pager`; `ls -l /etc/kubernetes/manifests/` | Kiểm tên file và quyền: kubelet chỉ đọc file trong đúng thư mục `manifests`. Nếu container khởi động rồi chết, đọc `crictl logs <id>` — đây là mục *kubeadm bị treo khi chờ control plane* của bài 09 nhìn từ giữa chừng. **Đừng chạy `kubeadm init`** để "sửa": cluster vẫn còn etcd và pki, init lại sẽ hỏng nặng hơn |
| **B5.2: master không boot lại được** | Trên host: `vmrun -T ws list`; mở console VM trong VMware | VM không boot thì xử lý ở tầng hypervisor, đừng đụng gì trong cluster. Nếu boot mà `ssh` chết thì IP tĩnh của [A3](LAB-00-MOI-TRUONG-1.35.7.md#a3-đặt-ip-tĩnh-và-phân-giải-tên) không lên. Nếu vẫn không xong: đây là lab kết thúc bằng restore — revert cả ba VM về `03-storage-ready` và làm lại từ B0 |
| B5.2: NodePort không trả `200` từ một worker khi master đã tắt | Trên worker đó: `sudo crictl ps`; `sudo iptables-save \| grep -c KUBE-` | Pod `web` có thể nằm hết trên worker kia và quy tắc chuyển tiếp xuyên node bị chặn. Đây là tầng 3 và tầng 5 của [A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready), không phải sự cố của kubeadm. Bật master lại rồi chạy lại hai tầng đó trước khi sang B6 |
| **B6: `kubeadm reset` treo** | `sudo crictl ps -a`; `systemctl is-active kubelet containerd` | Đúng mục *kubeadm bị treo khi xóa các container được quản lý* của bài 09 — đọc mục đó. Đừng `Ctrl-C` rồi chạy lại ngay; đọc trạng thái container trước |
| B6.2: sau khi flush, `ssh` vào node rớt | — | Policy của chain `INPUT` không phải `ACCEPT` lúc flush. Vào console VM trong VMware, đặt lại `sudo iptables -P INPUT ACCEPT`, rồi kiểm lại [A4.4](LAB-00-MOI-TRUONG-1.35.7.md#a44-firewall-cho-mạng-lab-cô-lập). Gate ở đầu B6.2 tồn tại để chặn đúng tình huống này — đừng bỏ qua nó lần sau |
| **B6.4: kubelet không lên lại sau reboot** | `swapon --show`; `grep -nE "^[^#].*[[:space:]]swap[[:space:]]" /etc/fstab`; `journalctl -u kubelet -n 80 --no-pager` | Nguyên nhân phổ biến nhất là **swap bật lại sau reboot** vì `/etc/fstab` còn dòng chưa comment — đúng bài học B1.3. Sửa theo [A4.1](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites) rồi chạy lại khối gate B6.4. Lưu ý ở giai đoạn này kubelet **được phép** `inactive`; chỉ `enabled` mới là điều kiện |
| **B7.2: preflight báo `[ERROR Port-6443]` hoặc `[ERROR FileAvailable]`** | `sudo ss -lntp \| grep 6443`; `sudo ls -A /etc/kubernetes/manifests/` | `kubeadm reset` ở B6 chưa chạy hết hoặc chạy nhầm node. Chạy lại `sudo kubeadm reset -f` trên đúng node rồi làm lại B6.3 và B6.4. **Không** dùng `--ignore-preflight-errors` để đi tiếp: nó chôn nguyên nhân xuống dưới |
| **B7.2: `kubeadm init` treo ở `[wait-control-plane]`** | `sudo crictl ps -a`; `sudo crictl logs <id>`; `sudo journalctl -u kubelet -n 100 --no-pager` | Đây là mục *kubeadm bị treo khi chờ control plane* của bài 09, với **ba** nguyên nhân bài liệt kê: mạng, cgroup driver lệch, hoặc container control plane crashloop. B1.4 đã kiểm cgroup driver nên bắt đầu từ hai cái còn lại. Đừng chạy lại `init` trước khi `reset` |
| B7.2: `kubeadm init` báo không tìm thấy container runtime | `sudo crictl version`; `sudo ls -l /run/containerd/containerd.sock` | Bài 01 nêu **hai** tình huống: phát hiện nhiều runtime, hoặc không phát hiện được cái nào. File cấu hình B7.1 đã khai `criSocket` tường minh, nên nếu vẫn lỗi thì containerd chưa `active` — xem [A4.2](LAB-00-MOI-TRUONG-1.35.7.md#a42-cài-containerd-và-runc-đúng-version) |
| **B7.5: annotation của patch không xuất hiện trong manifest** | `ls -l ~/lab-work/8a/patches/`; `grep -n 'patches:' -A2 ~/lab-work/8a/kubeadm-config.yaml` | Ba nguyên nhân theo thứ tự hay gặp: `patches.directory` không phải **đường dẫn tuyệt đối**; tên file sai quy ước `target[suffix][+patchtype].extension`; hoặc `patchtype` không hợp với nội dung patch. Bỏ `+merge` để dùng mặc định `strategic` là một phép thử nhanh. Sửa rồi phải `kubeadm reset` và init lại — patch chỉ áp lúc sinh manifest |
| **B7.6: CoreDNS vẫn `Pending` sau khi cài CNI** | `kubectl -n kube-system describe pod -l k8s-app=kube-dns`; `kubectl get nodes -o wide` | Nếu node vẫn `NotReady` thì CNI chưa chạy: xem lại [A5.2](LAB-00-MOI-TRUONG-1.35.7.md#a52-cài-flannel-v0289), đặc biệt gate đối chiếu `net-conf.json` với `--pod-network-cidr`. `POD_CIDR` bạn điền ở B0.2 phải khớp dải mà manifest CNI khai — lệch là hỏng đúng ở đây |
| **B8.2: `kubeadm join` báo token không hợp lệ hoặc hash sai** | Trên master: `sudo kubeadm token list`; chạy lại `sudo kubeadm token create --print-join-command` | Token có hạn và cluster mới có CA mới. **Đừng dùng lại lệnh join của cluster cũ** — B8.1 đã chứng minh hash khác hẳn. Sinh lệnh mới rồi chạy lại |
| B8.2: worker join xong nhưng `NotReady` | `kubectl describe node <worker>`; `ssh <worker> 'sudo ls /etc/cni/net.d/'` | Thư mục CNI trên worker trống nghĩa là DaemonSet của CNI chưa chạy tới node đó; chờ `kubectl -n kube-system get pods -o wide` cho thấy Pod CNI trên worker. Nếu Pod CNI `CrashLoopBackOff`, đọc `kubectl logs` — bài 09 có mục riêng cho Pod ở `CrashLoopBackOff` |
| B8.2: worker báo lỗi state cũ khi join | `ssh <worker> 'sudo ls -A /etc/kubernetes/'` | Node còn dấu vết cluster trước. Chạy `sudo kubeadm reset -f` trên chính worker đó, dọn `/etc/cni/net.d/*`, rồi join lại |
| **`kubectl` báo lỗi certificate sau khi dựng lại** | `ls -l ~/.kube/config`; `kubectl config view --minify` | Đúng mục *Lỗi certificate TLS* của bài 09: kubeconfig đang là file của cluster cũ. B6.2 đã dặn `rm -f ~/.kube/config`; nếu quên thì xóa rồi chép lại từ `/etc/kubernetes/admin.conf` theo [A5.1](LAB-00-MOI-TRUONG-1.35.7.md#a51-init-control-plane) |
| **B11.3: `vmrun revertToSnapshot` báo lỗi** | `vmrun -T ws list`; trạng thái VM trong VMware | VM chưa tắt hẳn. Chờ VMware hiển thị *Powered off* rồi revert lại. Đừng revert khi VM đang chạy |
| **B11.4: gate cuối fail — `creationTimestamp` khác, hoặc thiếu StorageClass/IngressClass** | `kubectl get nodes -o wide`; `vmrun -T ws listSnapshots <vmx>` | Bạn revert nhầm mốc, hoặc chỉ revert một phần VM. Tắt cả ba VM và revert **cả ba** về đúng `03-storage-ready`, rồi chạy lại từ bước 1 của B11.4. **Không** sang lab sau khi gate này còn fail |
| **B11.2 bị bỏ qua và bằng chứng đã mất** | — | Không khôi phục được: `~/lab-evidence/8a/` nằm trên đĩa VM và snapshot đã ghi đè nó. Ghi nhận trong nhật ký học rằng lab 8a chạy xong nhưng không có evidence, và lần sau chép bằng chứng ra host **trước** khi revert |

---

## 5. Nguồn chính thức

Các phần giải thích trong thân bài ưu tiên snapshot tài liệu Kubernetes v1.35
(`https://v1-35.docs.kubernetes.io/`) để hành vi, flag và trường cấu hình khớp minor version của
cluster lab.

- [Kubernetes v1.35 — Installing kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Kubernetes v1.35 — Creating a cluster with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [Kubernetes v1.35 — Customizing components with the kubeadm API](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/control-plane-flags/)
- [Kubernetes v1.35 — Configuring each kubelet in your cluster using kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/kubelet-integration/)
- [Kubernetes v1.35 — Dual-stack support with kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/dual-stack-support/)
- [Kubernetes v1.35 — Troubleshooting kubeadm](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/)
- [Kubernetes v1.35 — Administration with kubeadm](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/)
- [Kubernetes v1.35 — Ports and Protocols](https://v1-35.docs.kubernetes.io/docs/reference/networking/ports-and-protocols/)
- [Kubernetes v1.35 — kubeadm init](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
- [Kubernetes v1.35 — kubeadm join](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)
- [Kubernetes v1.35 — kubeadm reset](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
- [Kubernetes v1.35 — kubeadm token](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)
- [Kubernetes v1.35 — kubeadm certs](https://v1-35.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-certs/)
- [Kubernetes v1.35 — kubeadm Configuration API (v1beta4)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)
- [Kubernetes v1.35 — Kubelet Configuration (v1beta1)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [Kubernetes v1.35 — Static Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
- [Kubernetes v1.35 — NodeRestriction admission controller](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)
- [Kubernetes v1.35 — Bootstrap tokens](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/)
- [Kubernetes — Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)




