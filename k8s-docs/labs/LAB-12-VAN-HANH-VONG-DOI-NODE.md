# Lab 12 — Vận hành vòng đời node

> **Điểm bắt đầu:** snapshot `04-metrics-ready` — mốc do [Lab 11a](LAB-11A-OBSERVABILITY.md) tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot). Repo `~/bookstore` ở tag `lab-11b` (hoặc bundle
> `~/bookstore-lab-11b.bundle` nếu bạn vừa restore snapshot).
> **Điểm kết thúc:** teardown trả cluster về đúng `04-metrics-ready`, **không tạo snapshot mới**. Repo
> `~/bookstore` có tag `lab-12` và một bản bundle nằm ngoài VM. Lab tắt rồi bật lại hai worker nhưng
> không cài thêm gì và không sửa cấu hình node hay control plane nào.
> **Sprint dự án:** Sprint 19 — Bảo trì node có kế hoạch của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 11b — HPA và VPA](LAB-11B-HPA-VA-VPA.md).
> **Lab sau:** [Lab 13 — DRA](LAB-13-DRA.md) (tùy chọn) hoặc [Lab 14 — CRD và Operator](LAB-14-CRD-VA-OPERATOR.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng mục
[Giai đoạn 12 — Quản trị cluster nâng cao](../00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao):
tám bài khái niệm [155](../155-cluster-administration-vi.md), [169](../169-node-shutdown-vi.md),
[170](../170-swap-memory-management-vi.md), [171](../171-node-autoscaling-vi.md),
[165](../165-addons-vi.md), [168](../168-compatibility-version-vi.md),
[167](../167-coordinated-leader-election-vi.md), [156](../156-certificates-vi.md) và hai bài thực hành
[189](../189-administer-cluster-vi.md), [198](../198-controller-manager-leader-migration-vi.md). Mọi
ticket dựa vào [hợp đồng kỹ thuật](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật) của dự án; kế hoạch gốc
của sprint nằm ở [mục 7.2](DU-AN-BOOKSTORE.md#sprint-19--lab-12-bảo-trì-node-có-kế-hoạch).

Sprint 18 để lại một hệ thống đầy đủ: `web` 2–5 bản sao do HPA điều khiển, `api` hai bản sao, PDB
cho cả hai, `db-0` trên `lab-k8s-worker2` với PV local `Retain`, Ingress HTTPS qua Traefik,
`node-agent` ở `bookstore-system`. Vấn đề `P1` của hệ thống cũ — "bảo trì OS là downtime" — đã được
thiết kế để giải ([mục 3.7](DU-AN-BOOKSTORE.md#37-chịu-lỗi-và-vận-hành)), nhưng **chưa ai đo**.
Sprint này đo: bảo trì thật hai worker, một node không giữ dữ liệu và một node giữ DB, đọc cấu hình
node để biết tắt máy có an toàn không, và để lại cho đội một runbook bảo trì đã diễn tập.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); image và add-on nằm ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Lab không chép lại con số
phiên bản nào trong phần văn bản.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi bạn đã bảo trì xong hai worker, có runbook bảo trì trong repo **và** chứng minh,
giải thích được:

- Mỗi node đang gánh những gì trước khi bạn chạm vào nó: Pod của Bookstore, add-on nền nằm trên
  đường request, dữ liệu gắn với node — và add-on nào một bản sao, không PDB, nên chắc chắn gây
  gián đoạn khi node của nó được bảo trì.
- `cordon` chỉ đánh dấu, `drain` đánh dấu **và** trục xuất qua Eviction API nên tôn trọng PDB; vì sao
  Pod DaemonSet phải được bỏ qua tường minh; node sau drain là "Node rỗng" theo bài 171.
- Bảo trì `lab-k8s-worker1` không làm gián đoạn `web` và `api` — hoặc đo đúng số request lỗi, chỉ ra
  tầng gây lỗi và cách sửa.
- Vì sao `db-0` **không thể** rời `lab-k8s-worker2`, nên bảo trì node đó là một cửa sổ bảo trì có
  downtime đo được; PDB `maxUnavailable: 0` cho một DB một instance làm gì với `drain`, và đánh đổi
  khi đưa nó vào repo.
- Graceful node shutdown **không** hoạt động trên cluster này dù feature gate bật — chứng minh bằng
  cấu hình hiệu lực của kubelet và bằng inhibitor lock của systemd; vì vậy drain trước khi tắt máy
  là bắt buộc.
- Khi một node biến mất: condition `Ready` thành `Unknown`, taint `unreachable`, Node Lease đứng
  yên, Pod trên node mang trạng thái cũ không ai cập nhật được — đó chính là phase `Unknown` mà bài 47
  nói tới (trả nợ lab của Lab 3a).
- Reboot không gỡ cordon; uncordon không tự đưa Pod về; cân bằng lại bằng cách tôn trọng PDB.
- Swap trên ba node, vì sao kubelet từ chối swap, và nếu bật `LimitedSwap` thì Pod nào của Bookstore
  được swap.
- Control plane trước mùa nâng cấp: không thành phần nào giả lập phiên bản cũ, ai giữ Lease leader,
  bầu chọn leader có phối hợp đang tắt, Leader Migration không áp dụng — bằng dữ kiện đọc được.
- Cluster ba VM cố định không có autoscaling Node, và vì sao đó là giới hạn bản chất.
- Chủ đề certificate của giai đoạn 12 **chưa** được học — nợ lab #7 vẫn mở.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 12 | Ticket kiểm chứng |
| --- | --- |
| [155 — Quản trị cluster](../155-cluster-administration-vi.md) | B1 — mỗi nhóm việc một hiện vật trên cluster: lập kế hoạch (`providerID` rỗng), quản lý (`bookstore-quota`), bảo mật cluster và kubelet (RBAC, kubelet Webhook, certificate từ TLS bootstrapping), dịch vụ tùy chọn (DNS, nguồn metric) |
| [165 — Cài đặt các Add-on](../165-addons-vi.md) | B1 — kiểm kê add-on theo năm nhóm, ba add-on ngoài danh mục, add-on một bản sao nằm trên đường request của Bookstore |
| [189 — Quản trị một Cluster](../189-administer-cluster-vi.md) | B1 — dùng trang mục lục chọn ba tác vụ cho runbook, mỗi tác vụ một hiện vật: `/api` (190), `pods/eviction` (255), StorageClass mặc định (192) |
| [171 — Tự động mở rộng Node](../171-node-autoscaling-vi.md) | B1 — không autoscaler, không cloud provider; B5, B7 — Node rỗng sau drain; B7 — câu hỏi PDB `db` khi có autoscaler hợp nhất node |
| [168 — Phiên bản tương thích](../168-compatibility-version-vi.md) | B2 — không thành phần nào đặt `--emulated-version`; binary version từ hai nguồn, đối chiếu A1.3 |
| [167 — Bầu chọn leader có phối hợp](../167-coordinated-leader-election-vi.md) | B2 — các trường Lease control plane, `renewTime` tiến lên, tính năng đang tắt; B7 — Node Lease đứng yên khi node tắt |
| [198 — Di trú sang Cloud Controller Manager](../198-controller-manager-leader-migration-vi.md) | B2 — bốn dữ kiện chứng minh Leader Migration không áp dụng, cộng nội dung role `system::leader-locking-kube-controller-manager` |
| [170 — Quản lý bộ nhớ swap](../170-swap-memory-management-vi.md) | B3 — swap trên ba node, `/etc/fstab` trước khi reboot, Node không báo swap, `failSwapOn`; ADR swap cho node chứa `db` |
| [169 — Tắt node](../169-node-shutdown-vi.md) | B3 — `shutdownGracePeriod` bằng 0, không có inhibitor lock, `db-0` không phải pod quan trọng; B5, B7 — tắt máy sau drain; B7 — vì sao `db-0` không kẹt `Terminating`, vì sao taint `out-of-service` không cứu được PV local |
| [156 — Chứng chỉ](../156-certificates-vi.md) | **Không thực hành.** Trang trỏ hướng sáu dòng, không có thao tác. Quản lý vòng đời certificate là **[nợ lab #7](README.md#5-sổ-nợ-lab)**, chưa trả ở lab này; B8 ghi nó vào runbook dưới mục "ngoài phạm vi" |

Lab còn dùng các bài đã học trước: [47](../47-pod-lifecycle-vi.md) (phase, kubelet khởi động lại),
[53](../53-disruptions-vi.md) và [339](../339-configure-pdb-vi.md) (PDB),
[143](../143-api-eviction-vi.md) (Eviction API), [23](../23-nodes-vi.md) và
[35](../35-leases-vi.md) (heartbeat, node controller), [139](../139-taint-and-toleration-vi.md)
(taint theo điều kiện), [210](../210-guaranteed-scheduling-critical-addon-pods-vi.md) (pod quan
trọng), [42](../42-container-lifecycle-hooks-vi.md) (handler `sleep`).

Nợ lab trả trong sprint này — **đọc lại bài gốc trước khi làm ticket trả nợ**:

| Nợ | Phát sinh ở | Trả ở |
| --- | --- | --- |
| Phase `Unknown` của Pod (bài [47](../47-pod-lifecycle-vi.md#pod-phase)) | [Lab 3a](LAB-3A-POD-VA-VONG-DOI.md) | B7 — quan sát Pod trên node đã mất liên lạc |
| Mục *Hành vi của Pod khi kubelet khởi động lại* (bài [47](../47-pod-lifecycle-vi.md#kubelet-restarts)), hoãn sang giai đoạn 12 | khối *Đọc bài này thế nào* của bài 47 | B3 ⭐ |

Phần không thực hành, ghi rõ lý do:

| Phần | Vì sao không làm ở đây |
| --- | --- |
| Bài 169 — **bật** graceful node shutdown, `shutdownGracePeriodByPodPriority` | Phải sửa `KubeletConfiguration` trên node đang chạy — [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). B3 đọc phần đọc được |
| Bài 169 — tắt node **không** nhẹ nhàng với `db-0` trên node, taint `out-of-service` | Cố ý dựng là cố ý tắt ngang PostgreSQL đang ghi. B7 làm đúng quy trình và chứng minh hệ quả ngược lại; B7 ⭐ dựng kịch bản đó trên một Pod thử nghiệm không có dữ liệu |
| Bài 170 — bật swap, `LimitedSwap` thật | Đổi prereq OS của [A4.1](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites) và cấu hình kubelet |
| Bài 171 — Cluster Autoscaler hay Karpenter thật | Cần API của cloud provider; ba VM tự dựng không có. Giới hạn bản chất |
| Bài 167, 168 — bật tính năng, đặt `--emulated-version` | Sửa manifest control plane (giai đoạn 20); giá trị thật chỉ hiện ra khi nâng cấp nhiều control plane (giai đoạn 17, Lab 8b/8c) |
| Bài 198 — chạy Leader Migration | Cần cloud provider in-tree và control plane nhân bản; không dữ kiện nào tồn tại ở đây |
| Bài 189 — các trang con còn lại | Mục lục của Phần II; mỗi trang có chỗ đứng ở giai đoạn 16–27. Quy trình drain chi tiết (bài [255](../255-safely-drain-node-vi.md)) học sâu ở [giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node) |

Nợ kỹ thuật của hệ thống sau sprint: **phát sinh hai nợ mới** — đường vào từ ngoài chưa có HA, và
DB một instance trên PV local — ghi ở B8. Nợ lab #1 (phần VPA), #5, #6, #7, #8 vẫn mở.

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B9) khoảng 4 giờ, trong đó hai vòng tắt–bật máy chiếm phần lớn thời gian
chờ. Các mục ⭐ là tùy chọn; ⭐ của B7 làm cửa sổ bảo trì worker2 dài thêm vài phút.
Thời gian chờ phụ thuộc cấu hình cluster và phần cứng host, nên mọi bước chờ trong lab là vòng lặp
có điều kiện dừng hoặc `kubectl wait --timeout`. Có thể chia hai phiên: phiên 1 làm B0–B5, phiên 2
làm B6–B9; đầu phiên 2 chạy lại gate ngắn A5.5 và `./tests/smoke.sh`.

---

## 2. Quy ước và an toàn

> **Cảnh báo — lab này TẮT và BẬT LẠI hai máy ảo, mỗi lần một máy.** Không bao giờ tắt
> `lab-k8s-master`: mất control plane là mất đường quan sát. Không bao giờ tắt hai worker cùng
> lúc: đó là mất toàn bộ dịch vụ, không phải bảo trì.

Trước B0, xác nhận trên **máy host** rằng cả ba VM còn mốc `04-metrics-ready` — không có mốc này
thì không có đường lui. Chạy trên máy host, PowerShell:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '04-metrics-ready') { "PASS: $f" } else { "FAIL: $f" }
}
```

**PASS:** đúng ba dòng `PASS:`. Đường dẫn `.vmx` theo máy host của bạn; sửa nếu VM nằm chỗ khác. Giữ
cửa sổ PowerShell này mở — B5 và B7 dùng lại `$vmrun` và `$vmx`.

Các quy ước còn lại:

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác. Lệnh trên worker chạy qua
  `ssh <node> 'sudo ...'` từ master. Khối PowerShell chạy trên **máy host** và luôn được ghi rõ như
  vậy.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/12/`. Bằng chứng: `~/lab-evidence/12/`.
- **Bảo trì worker1 là vận hành có kế hoạch, không phải fault injection.** Nó đi qua drain và tắt máy
  êm — đúng việc [mục 3.7](DU-AN-BOOKSTORE.md#37-chịu-lỗi-và-vận-hành) hứa là không rơi request ở
  tầng ứng dụng trên đường đo của [mục 3.1](DU-AN-BOOKSTORE.md#31-sơ-đồ-tổng-thể). Mọi
  **diễn tập sự cố** của lab (Pod thử nghiệm mất liên lạc, khởi động lại kubelet, PDB thử chặn drain)
  chỉ chạy trên `lab-k8s-worker2`.
- **Không dùng `--force`, không dùng `--disable-eviction` với `kubectl drain`.** Cái đầu xóa vĩnh viễn
  Pod không có controller; cái sau bỏ qua PDB — đúng cơ chế mà sprint này muốn đo. Drain đòi một
  trong hai cờ đó thì dừng lại và đọc [mục 4](#4-troubleshooting-của-lab-này).
- `--delete-emptydir-data` xóa dữ liệu `emptyDir` của Pod bị trục xuất. Với Bookstore, `emptyDir`
  chỉ chứa trang HTML do init container sinh lại và file tạm — xóa được. Dữ liệu thật của `db` và
  bản backup nằm trên PVC, drain không đụng tới. Đừng mang cờ này sang cluster khác mà chưa kiểm.
- **Lab chỉ đọc cấu hình node và control plane.** Không sửa `/var/lib/kubelet/config.yaml`, không sửa
  `/etc/kubernetes/manifests/*`, không đổi feature gate, không đổi số bản sao của add-on nào. B0 ghi
  checksum sáu file cấu hình; gate cuối đối chiếu lại.
- Không gỡ taint bằng tay. Taint do cordon và do node controller gắn thì để cơ chế tương ứng gỡ.
- Object thử nghiệm (`PodDisruptionBudget` `db-thu`, Pod `thu-mat-lien-lac`) nằm trong namespace
  `bookstore`, **không** mang label `app.kubernetes.io/part-of`, bị xóa khi xong ticket.
- `kubectl rollout restart`, Eviction API gọi tay chỉ dùng để **cân bằng lại** sau bảo trì — thao tác
  vận hành, không đổi cấu hình. Mọi thay đổi cấu hình của hệ thống đi qua repo.
- Chạy mỗi ticket bảo trì (B5, B7) trong **một phiên shell liền mạch** trên master: probe chạy nền và
  hàm `moc` sống trong phiên đó. Phiên này nằm trên master nên sống sót qua lúc worker tắt.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 19: Bảo trì node có kế hoạch

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `04-metrics-ready`, repo ở đúng tag `lab-11b`, hệ thống của
sprint 18 dựng lại được **chỉ từ repo**, và ghi lại cấu hình node trước khi đụng vào máy nào.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Ba điều kiện riêng của mốc `04-metrics-ready` và của lab này:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
kubectl top node
test "$(sl kubectl top node --no-headers)" -eq 3 \
  && echo 'PASS: dung moc 04-metrics-ready — top node du ba node'
test "$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.unschedulable}{"\n"}{end}' | grep -c true)" -eq 0 \
  && echo 'PASS: chua node nao bi cordon'
kubectl get pdb -A 2>&1 | grep -q 'No resources found' && echo 'PASS: cluster chua co PDB nao'
```

Repo và thư mục làm việc. Nếu vừa restore snapshot, khôi phục repo từ bundle theo
[mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
mkdir -p ~/lab-work/12 ~/lab-evidence/12
test -d ~/bookstore/.git || git clone ~/bookstore-lab-11b.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-11b' && echo 'PASS: HEAD dung tag lab-11b'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | grep -q . \
  && echo 'FAIL: con namespace du an sot tu lan truoc' || echo 'PASS: cluster chua co du an'
```

Dựng lại hệ thống của sprint 18 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/12/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/12/b0-smoke-lab-11b.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/12/b0-smoke-lab-11b.txt
```

Kéo sẵn image lên **cả hai** worker: lúc bảo trì, mọi Pod sẽ dồn sang worker còn lại, và thời gian
kéo image không được lẫn vào số đo gián đoạn. Tag lấy đúng từ
[A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và
[A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00):

```bash
for N in lab-k8s-worker1 lab-k8s-worker2; do
  for I in docker.io/library/busybox:1.37 \
           docker.io/library/postgres:17.11 \
           docker.io/postgrest/postgrest:v14.18 \
           docker.io/nginxinc/nginx-unprivileged:1.30.5-alpine; do
    ssh "$N" "sudo crictl pull $I" >/dev/null && echo "PASS: $N co $I" || echo "FAIL: $N khong keo duoc $I"
  done
done
```

Checksum của cấu hình ba kubelet và ba manifest control plane — gate cuối đối chiếu lại để chứng
minh lab không sửa gì, kể cả sau hai lần tắt–bật máy:

```bash
{
  echo "lab-k8s-master kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "lab-k8s-master $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} > ~/lab-evidence/12/b0-config-sha.txt
cat ~/lab-evidence/12/b0-config-sha.txt
test "$(awk '{print $3}' ~/lab-evidence/12/b0-config-sha.txt | grep -c '^[0-9a-f]\{64\}$')" -eq 6 \
  && echo 'PASS: ghi duoc checksum cua sau file cau hinh'
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; ba dòng `PASS:` của mốc `04-metrics-ready`; ba dòng
`PASS:` về repo và cluster; `deploy exit=0`, `smoke exit=0` và file smoke không có dòng `FAIL:`; đủ
tám dòng `PASS: ... co ...`; dòng `PASS:` về checksum.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 18 **chưa đủ** — sửa repo ở tag `lab-11b`
trước, không sửa tay trên cluster ([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)). Nếu
`kubectl top node` báo lỗi, cluster chưa ở mốc đầu vào: restore cả ba VM về `04-metrics-ready`.

## B1. [BS-12-01] Kiểm kê nền tảng và rủi ro trước bảo trì

**Loại:** Điều tra.

**Bối cảnh:** trước khi tắt một node, đội phải biết node đó đang gánh gì. Request của khách không chỉ
đi qua `web`, `api`, `db` mà còn qua ingress controller, DNS của cluster, và HPA của `web` cần nguồn
metric. Những thứ đó là **add-on** — dự án không viết chúng nhưng phụ thuộc vào chúng. Bài 155 chia
việc quản trị thành nhóm, bài 165 là danh mục add-on, bài 189 là mục lục tác vụ: ba trang mục lục
này, đọc đúng cách, cho bạn bản đồ để lập bảng rủi ro.

**Yêu cầu:**

1. Với mỗi nhóm việc của bài 155 — lập kế hoạch, quản lý cluster, bảo mật cluster (kể cả nhánh bảo
   mật kubelet), dịch vụ tùy chọn — chỉ ra **một hiện vật có thật** trên cluster của bạn.
2. Kiểm kê add-on đang chạy, xếp vào năm nhóm chức năng của bài 165, chỉ ra những add-on đang chạy
   mà danh mục **không** liệt kê, và những nhóm đang trống.
3. Chứng minh cluster không có autoscaling Node: không autoscaler nào chạy, không cloud provider nào
   đứng sau Node.
4. Dùng trang 189 đúng công dụng mục lục: chọn ba tác vụ liên quan tới runbook bảo trì — truy cập
   API, drain node, StorageClass mặc định — và chỉ ra cho mỗi tác vụ một hiện vật trên cluster.
5. Lập **bảng rủi ro** ở mục 1 của `docs/runbook-bao-tri-node.md`: với mỗi workload của Bookstore và
   mỗi add-on nằm trên đường request hoặc trên đường của HPA — số bản sao, có PDB không, đang ở node
   nào, dữ liệu gắn với node nào; rồi **dự đoán** điều gì xảy ra khi drain `lab-k8s-worker1` và khi
   drain `lab-k8s-worker2`. B5 và B7 sẽ đối chiếu dự đoán với số đo.

**Ràng buộc:** chỉ đọc. Không đổi số bản sao của add-on nào, không thêm PDB cho add-on nào — đó là đổi
hạ tầng của mốc `04-metrics-ready`.

**Đọc lại:**

- Bài [155](../155-cluster-administration-vi.md):
  [*Lập kế hoạch cho cluster*](../155-cluster-administration-vi.md#lập-kế-hoạch-cho-cluster-planning-a-cluster),
  [*Quản lý cluster*](../155-cluster-administration-vi.md#quản-lý-cluster-managing-a-cluster),
  [*Bảo mật kubelet*](../155-cluster-administration-vi.md#bảo-mật-kubelet-securing-the-kubelet),
  [*Các dịch vụ cluster tùy chọn*](../155-cluster-administration-vi.md#các-dịch-vụ-cluster-tùy-chọn-optional-cluster-services).
- Bài [165](../165-addons-vi.md) — câu mở đầu về độ đầy đủ của danh sách;
  [*Mạng và Network Policy*](../165-addons-vi.md#networking-and-network-policy),
  [*Hạ tầng*](../165-addons-vi.md#hạ-tầng-infrastructure),
  [*Đo lường*](../165-addons-vi.md#đo-lường-instrumentation).
- Bài [171](../171-node-autoscaling-vi.md): [*Cấp phát Node*](../171-node-autoscaling-vi.md#provisioning),
  [*Các autoscaler*](../171-node-autoscaling-vi.md#autoscalers).
- Bài [189](../189-administer-cluster-vi.md#danh-sách-các-trang-trong-mục-này-pages-in-this-section)
  — chỉ đọc như mục lục.
- Bài [53](../53-disruptions-vi.md#pod-disruption-budgets) — ai nên tôn trọng PDB, và PDB chỉ chặn được
  gián đoạn đi qua Eviction API.

**Bàn giao:** `docs/runbook-bao-tri-node.md` với mục `## 1. Hiện trạng và rủi ro`;
`~/lab-evidence/12/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` không node nào có `providerID`, `kube-controller-manager` không đặt cloud provider, không
  autoscaler hay descheduler nào chạy.
- `PASS:` ResourceQuota `bookstore-quota` là hiện vật của nhóm *Quản lý cluster*.
- `PASS:` API server phân quyền bằng RBAC; cả ba kubelet ủy quyền quyết định cho API server; kubelet
  của worker2 dùng client certificate do TLS bootstrapping cấp.
- `PASS:` DNS và nguồn metric đang chạy; ba nhóm *Đo lường*, *Hạ tầng*, *Trực quan hóa* đang trống;
  có IngressClass và đúng một StorageClass mặc định, không phải `bookstore-retain`.
- `PASS:` mọi PDB trên cluster đều thuộc dự án — add-on nền không có PDB nào.
- `PASS:` dữ liệu của `db-0` gắn với `lab-k8s-worker2`.
- `PASS:` `/api` và subresource `pods/eviction` tồn tại trong API.
- `PASS:` runbook có mục 1 nhắc tên mọi add-on một bản sao và nơi đặt `data-db-0`, `backup-store`.

**Câu hỏi thiết kế** — trả lời ngay trong mục 1 của runbook, phần *Nhận định*:

1. Add-on nào nằm trên đường request của khách (trình duyệt → … → `db`)? Cái nào một bản sao và
   không có PDB? Dự đoán chuyện gì xảy ra với request khi bạn drain node đang chạy nó.
2. Ba add-on đang chạy không có tên trong danh mục 165. Kết luận đúng khi tra danh mục mà không thấy
   công cụ mình cần là gì?
3. "Bảo trì worker1 không gián đoạn" và "bảo trì worker2 không gián đoạn" — cái nào khả thi với thiết
   kế hiện tại? Lý do nằm ở bảng *dữ liệu gắn node* của bạn.
4. Nếu cluster chạy trên cloud có autoscaler hợp nhất node, ai sẽ gây gián đoạn tự nguyện cho
   Bookstore mà không cần con người, và cơ chế nào của bạn còn đứng chắn?

<details>
<summary>Gợi ý</summary>

- Một Pod "nằm ở node nào": cột `NODE` của `kubectl get pods -A -o wide`.
- Số bản sao của mọi Deployment trên cluster:
  `kubectl get deployment -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,REPL:.spec.replicas`.
- PV của local-path mang node affinity tới đúng một node: `kubectl get pv -o yaml`, tìm
  `nodeAffinity`.
- Cấu hình **hiệu lực** của kubelet, kể cả giá trị mặc định không viết ra file:
  `kubectl get --raw /api/v1/nodes/<node>/proxy/configz`. Node lab không có `jq`; dùng `grep -o`.
- Danh mục API của nhóm core: `kubectl get --raw /api/v1`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
EV=~/lab-evidence/12; RB=~/bookstore/docs/runbook-bao-tri-node.md; W2=lab-k8s-worker2
NODES='lab-k8s-master lab-k8s-worker1 lab-k8s-worker2'

# 155 — lập kế hoạch; 171 — không autoscaling Node
PID_N="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.providerID}{"\n"}{end}' | grep -c .)"
CP_F="$(sudo grep -c -e '--cloud-provider' /etc/kubernetes/manifests/kube-controller-manager.yaml)"
AS_N="$(kubectl get deployment,statefulset,daemonset -A --no-headers 2>/dev/null \
  | grep -ciE 'cluster-autoscaler|karpenter|descheduler')"
echo "providerID=$PID_N cloud-provider=$CP_F autoscaler=$AS_N" | tee "$EV/b1-ke-hoach.txt"
[ "$PID_N" -eq 0 ] && [ "$CP_F" -eq 0 ] && [ "$AS_N" -eq 0 ] \
  && echo 'PASS: tu host, khong cloud provider, khong autoscaler Node'

# 155 — quản lý cluster
kubectl -n bookstore get resourcequota bookstore-quota >/dev/null 2>&1 \
  && echo 'PASS: nhom Quan ly cluster co hien vat — ResourceQuota bookstore-quota'

# 155 — bảo mật cluster và bảo mật kubelet
AUTHZ="$(sudo grep -oE -e '--authorization-mode=[^ ]+' /etc/kubernetes/manifests/kube-apiserver.yaml \
  | head -1 | cut -d= -f2)"
WH=0
for n in $NODES; do
  kubectl get --raw "/api/v1/nodes/$n/proxy/configz" > "$EV/b1-configz-$n.json" 2>/dev/null
  grep -q '"mode":"Webhook"' "$EV/b1-configz-$n.json" && WH=$((WH + 1))
done
echo "authorization-mode=$AUTHZ kubelet-webhook=$WH/3"
case "$AUTHZ" in *RBAC*) [ "$WH" -eq 3 ] && echo 'PASS: RBAC tren API server, Webhook tren ca ba kubelet' ;; esac
ssh "$W2" 'sudo test -e /var/lib/kubelet/pki/kubelet-client-current.pem' \
  && echo 'PASS: kubelet worker2 dung client certificate do TLS bootstrapping cap'

# 155 — dịch vụ tùy chọn; 165 — năm nhóm và add-on ngoài danh mục
DNS_R="$(kubectl -n kube-system get deployment coredns -o jsonpath='{.status.readyReplicas}')"
MS="$(kubectl get apiservice v1beta1.metrics.k8s.io -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
KSM="$(kubectl get deployment,statefulset -A --no-headers 2>/dev/null | grep -ci 'kube-state-metrics')"
NPD="$(kubectl get daemonset -A --no-headers 2>/dev/null | grep -ci 'node-problem-detector')"
UI="$(kubectl get deployment -A --no-headers 2>/dev/null | grep -ciE 'dashboard|headlamp')"
IC="$(kubectl get ingressclass --no-headers 2>/dev/null | wc -l)"
DEF="$(kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.name}={.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' | grep '=true$')"
echo "coredns=$DNS_R metrics=$MS ksm=$KSM npd=$NPD ui=$UI ingressclass=$IC mac_dinh=$DEF" | tee "$EV/b1-addon.txt"
{ [ "${DNS_R:-0}" -ge 1 ] && [ "$MS" = 'True' ] && [ "$KSM" -eq 0 ] && [ "$NPD" -eq 0 ] && [ "$UI" -eq 0 ] \
  && [ "$IC" -ge 1 ] && [ "$(echo "$DEF" | grep -c .)" -eq 1 ] && ! echo "$DEF" | grep -q '^bookstore-retain='; } \
  && echo 'PASS: DNS, metric, ingress co mat; ba nhom trong; StorageClass mac dinh khong phai bookstore-retain'

# Rủi ro: add-on một bản sao, PDB toàn cluster, dữ liệu gắn node
kubectl get deployment -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}={.spec.replicas}{"\n"}{end}' \
  | grep -v '^bookstore' | grep '=1$' | cut -d= -f1 > "$EV/b1-mot-ban-sao.txt"
echo '--- add-on mot ban sao:'; cat "$EV/b1-mot-ban-sao.txt"
[ "$(kubectl get pdb -A --no-headers 2>/dev/null | awk '$1 !~ /^bookstore/' | wc -l)" -eq 0 ] \
  && echo 'PASS: moi PDB tren cluster deu thuoc du an'
kubectl get pv -o jsonpath='{range .items[*]}{.spec.claimRef.namespace}/{.spec.claimRef.name}={.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}{"\n"}{end}' \
  | tee "$EV/b1-pv-node.txt"
grep -qx 'bookstore/data-db-0=lab-k8s-worker2' "$EV/b1-pv-node.txt" \
  && echo 'PASS: du lieu cua db-0 gan voi lab-k8s-worker2'

# 189 — ba hiện vật cho ba mục (StorageClass mặc định đã kiểm ở trên)
kubectl get --raw /api | grep -q '"versions"' \
  && kubectl get --raw /api/v1 | grep -q '"pods/eviction"' \
  && echo 'PASS: /api va subresource pods/eviction ton tai'

# Runbook
BAD=0
grep -q '^## 1\.' "$RB" 2>/dev/null || { echo 'FAIL: runbook chua co muc 1'; BAD=1; }
for X in $(cut -d/ -f2 "$EV/b1-mot-ban-sao.txt") data-db-0 backup-store; do
  grep -q "$X" "$RB" 2>/dev/null || { echo "FAIL: runbook chua nhac $X"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: runbook muc 1 nhac moi add-on mot ban sao va noi dat du lieu'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Lệnh thu thập, ngoài những lệnh đã có trong script nghiệm thu:

```bash
kubectl get pods -A -o wide --field-selector=status.phase=Running \
  | awk 'NR==1 || $1 !~ /^bookstore/ || $1 == "bookstore"' > ~/lab-evidence/12/b1-pods.txt
kubectl get daemonset,deployment,statefulset -A -o wide > ~/lab-evidence/12/b1-workload.txt
kubectl -n bookstore get pdb,hpa,pvc
```

Mục 1 của runbook — ví dụ, cột node điền theo cluster của bạn:

```markdown
## 1. Hiện trạng và rủi ro (đọc ngày 2026-10-13)

### 1.1. Bốn nhóm việc của bài 155 trên cluster này
- Lập kế hoạch: tự host, on-premises, VM, không HA; `providerID` rỗng ở cả ba node.
- Quản lý cluster: ba Node; ResourceQuota `bookstore-quota` của dự án.
- Bảo mật: `--authorization-mode` có RBAC; kubelet `authorization.mode: Webhook`;
  `kubelet-client-current.pem` do TLS bootstrapping cấp.
- Dịch vụ tùy chọn: CoreDNS; metrics-server; log node do `node-agent` đọc (Lab 11a).

### 1.2. Đường request và đường dữ liệu
| Thành phần | Nhóm (bài 165) | Bản sao | PDB | Node | Drain worker1 | Drain worker2 |
| --- | --- | --- | --- | --- | --- | --- |
| traefik | ngoài danh mục | 1 | không | worker1 | rơi request vài giây | không ảnh hưởng |
| coredns | Khám phá dịch vụ | 2 | không | master, worker2 | không ảnh hưởng | còn bản kia |
| metrics-server | ngoài danh mục | 1 | không | worker2 | không | HPA mất metric vài phút |
| local-path-provisioner | ngoài danh mục | 1 | không | worker1 | chỉ chặn PVC mới | không |
| calico-kube-controllers | Mạng | 1 | không | worker1 | không trên đường request | không |
| web (Bookstore) | — | 2–5 (HPA) | có | worker1, worker2 | không gián đoạn | không gián đoạn |
| api (Bookstore) | — | 2 | có | worker1, worker2 | không gián đoạn | không gián đoạn khi db còn |
| db-0 (Bookstore) | — | 1 | không | worker2, PV `data-db-0` trên worker2 | không | **ngừng** tới khi worker2 về |
| backup-store | — | PVC | — | PV trên worker1 | backup trễ tới khi worker1 về | không |

### 1.3. Nhận định
...
```

Ý chính cho phần *Nhận định*:

1. Đường request: Traefik → `web` → `api` → `db`, cộng CoreDNS khi một tầng phân giải tên lúc khởi
   động hay lúc kết nối lại. Traefik theo chart mặc định chạy **một** bản sao, không PDB: drain node
   của nó là trục xuất bản duy nhất; trong lúc Pod mới chưa Ready, NodePort không có đích — request
   rơi. PDB của Bookstore không bảo vệ được tầng này vì nó không thuộc Bookstore.
2. Ingress controller (5b), provisioner (6a), metrics-server (11a) đều là add-on theo đúng định nghĩa
   của bài — mở rộng Kubernetes, cài thêm, bên thứ ba — nhưng không có tên trong danh mục. Bài nói
   thẳng danh sách *không nhằm liệt kê đầy đủ*: tra không thấy nghĩa là danh mục chưa liệt kê.
3. Worker1 khả thi: mọi thứ của Bookstore trên đó đều có bản sao ở worker2. Worker2 không: `db-0`
   một bản sao, PV local của nó chỉ tồn tại trên đĩa worker2, và node affinity của `db` (Lab 7a) chỉ
   chấp nhận worker2. Bảo trì worker2 là một **cửa sổ bảo trì** có downtime DB.
4. Autoscaler hợp nhất node bằng cách drain node (sơ đồ ở mục *Các autoscaler* của bài 171) — tức đi
   qua Eviction API. PDB `web`, `api` vẫn đứng chắn; `db` không có PDB nên không có gì đứng chắn.
   Câu này quay lại ở ADR của B7.

</details>

## B2. [BS-12-02] Control plane trước mùa nâng cấp

**Loại:** Điều tra.

**Bối cảnh:** sau bảo trì node, việc lớn tiếp theo của đội vận hành là nâng cấp cluster (giai đoạn 17)
để Bookstore được dùng phiên bản mới. Runbook nâng cấp sẽ cần vài dữ kiện về control plane mà ngày
thường không ai nhìn: thành phần nào đang giả lập phiên bản cũ, ai giữ quyền leader và giữ bằng
cách nào, có cần quy trình Leader Migration không. Sprint này thu thập chúng, không thay đổi gì.

**Yêu cầu:**

1. Chứng minh không thành phần control plane nào giả lập phiên bản cũ; đọc binary version từ hai
   nguồn độc lập, chúng khớp nhau và khớp baseline ở A1.3.
2. Với kube-scheduler và kube-controller-manager: ai đang giữ quyền leader, Lease hiệu lực bao lâu, đã
   đổi chủ bao nhiêu lần; chứng minh leader còn sống.
3. Chứng minh kubelet của hai worker đang gửi nhịp tim qua Lease — mốc so sánh cho B7, khi một
   worker tắt.
4. Chứng minh bầu chọn leader có phối hợp đang tắt.
5. Bốn dữ kiện đọc được chứng minh Leader Migration không áp dụng cho cluster này, cộng quyền mà role
   leader-locking của controller manager đang cho phép.
6. ADR `docs/adr/NNNN-control-plane-truoc-nang-cap.md` (`NNNN` là số tiếp theo trong `docs/adr/`).

**Ràng buộc:** chỉ đọc. Không sửa manifest control plane, không patch role nào.

**Đọc lại:**

- Bài [168](../168-compatibility-version-vi.md#phiên-bản-giả-lập-emulated-version) — hai quy tắc khả
  dụng và ràng buộc `≤ binaryVersion`.
- Bài [167](../167-coordinated-leader-election-vi.md):
  [*Bật bầu chọn leader có phối hợp*](../167-coordinated-leader-election-vi.md#bật-bầu-chọn-leader-có-phối-hợp-enabling-coordinated-leader-election),
  [*Lựa chọn leader cho các thành phần Kubernetes*](../167-coordinated-leader-election-vi.md#lựa-chọn-leader-cho-các-thành-phần-kubernetes-leader-selection-for-kubernetes-components).
- Bài [198](../198-controller-manager-leader-migration-vi.md):
  [*Bối cảnh*](../198-controller-manager-leader-migration-vi.md#bối-cảnh-background),
  [*Cấp quyền truy cập Lease di trú*](../198-controller-manager-leader-migration-vi.md#cấp-quyền-truy-cập-lease-di-trú-grant-access-to-migration-lease).
- Bài [35](../35-leases-vi.md#node-heart-beats) và
  [Lab 1c phần B3](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md#b3-lease-heartbeat-leader-election-và-api-server-identity).

**Bàn giao:** `docs/adr/NNNN-control-plane-truoc-nang-cap.md`; `~/lab-evidence/12/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` không manifest control plane nào có `--emulated-version`; `gitVersion` của API server khớp
  tag image trong manifest và khớp baseline A1.3.
- `PASS:` leader của kube-scheduler nằm trên `lab-k8s-master`; `renewTime` tiến lên theo thời gian.
- `PASS:` Node Lease của từng worker được gia hạn.
- `PASS:` `LeaseCandidate` không có trong API, `coordination.k8s.io/v1beta1` không được phục vụ, không
  feature gate nào bật tính năng.
- `PASS:` bốn dữ kiện loại trừ Leader Migration; role leader-locking chưa từng được vá cho Lease di
  trú, và Lease đó không tồn tại.
- `PASS:` có ADR.

**Câu hỏi thiết kế** — trả lời trong ADR:

1. Đội nâng binary control plane lên minor kế tiếp nhưng giữ `--emulated-version` ở phiên bản hiện
   tại. Một trường API chỉ có ở phiên bản mới — manifest Bookstore dùng được chưa? Một API đã bị gỡ ở
   bản mới — còn dùng được không? Tùy chọn này tách hai việc gì?
2. Cluster chỉ có một kube-scheduler. Lease của nó bảo đảm điều gì? Nếu static Pod kube-scheduler
   khởi động lại giữa giờ cao điểm, Pod `web` đang chạy có bị ảnh hưởng không, còn Pod mới HPA vừa
   yêu cầu thì sao?
3. Bật `CoordinatedLeaderElection` có thay thế cơ chế Lease không? Nó đổi **điều gì** trong cách chọn
   leader?
4. Bốn dữ kiện nào loại trừ Leader Migration? Nếu một ngày cluster chạy trên cloud với ba control
   plane, bước đầu tiên của quy trình là vá role — vì sao?

<details>
<summary>Gợi ý</summary>

- Ba manifest control plane nằm ở `/etc/kubernetes/manifests/` trên master; đọc bằng `sudo grep`.
- `kubectl get --raw /version` trả JSON có `gitVersion`.
- Lease control plane ở namespace `kube-system`, Lease của node ở `kube-node-lease`, tên trùng tên node.
- API group đang được phục vụ: `kubectl get --raw /apis/coordination.k8s.io`.
- Role của controller manager:
  `kubectl -n kube-system get role 'system::leader-locking-kube-controller-manager' -o yaml`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Trước khi chạy, mở [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), đọc dòng
*Kubernetes control plane* rồi đặt biến — lab không chép con số vào đây:

```bash
export K8S_BASELINE='<phien ban Kubernetes control plane trong bang A1.3, dang vX.Y.Z>'
```

```bash
EV=~/lab-evidence/12; W1=lab-k8s-worker1; W2=lab-k8s-worker2
M=/etc/kubernetes/manifests

# 168
EMU=0
for f in kube-apiserver kube-controller-manager kube-scheduler; do
  EMU=$((EMU + $(sudo grep -c 'emulated-version' "$M/$f.yaml")))
done
SRV="$(kubectl get --raw /version | grep -o '"gitVersion":"[^"]*"' | cut -d'"' -f4)"
IMG="$(sudo grep -oE 'image: registry\.k8s\.io/kube-apiserver:[^[:space:]]+' "$M/kube-apiserver.yaml" | head -1 | cut -d: -f3)"
echo "emulated=$EMU gitVersion=$SRV tag_image=$IMG baseline=$K8S_BASELINE" | tee "$EV/b2-phien-ban.txt"
{ [ "$EMU" -eq 0 ] && [ -n "$SRV" ] && [ "$SRV" = "$IMG" ] && [ "$SRV" = "$K8S_BASELINE" ]; } \
  && echo 'PASS: khong gia lap; binary version khop tag image va baseline A1.3'

# 167 — Lease control plane
kubectl -n kube-system get lease kube-controller-manager kube-scheduler \
  -o custom-columns='NAME:.metadata.name,HOLDER:.spec.holderIdentity,ACQUIRE:.spec.acquireTime,RENEW:.spec.renewTime,DUR:.spec.leaseDurationSeconds,TRANS:.spec.leaseTransitions' \
  | tee "$EV/b2-lease.txt"
H="$(kubectl -n kube-system get lease kube-scheduler -o jsonpath='{.spec.holderIdentity}')"
R1="$(kubectl -n kube-system get lease kube-scheduler -o jsonpath='{.spec.renewTime}')"
sleep 20
R2="$(kubectl -n kube-system get lease kube-scheduler -o jsonpath='{.spec.renewTime}')"
case "$H" in *lab-k8s-master*) [ -n "$R1" ] && [ "$R1" != "$R2" ] \
  && echo "PASS: leader kube-scheduler tren lab-k8s-master, renewTime $R1 -> $R2" ;; esac

# Node Lease — mốc cho B7
for N in "$W1" "$W2"; do
  A="$(kubectl -n kube-node-lease get lease "$N" -o jsonpath='{.spec.renewTime}')"; sleep 15
  B="$(kubectl -n kube-node-lease get lease "$N" -o jsonpath='{.spec.renewTime}')"
  echo "$N $A -> $B" | tee -a "$EV/b2-node-lease.txt"
  [ -n "$A" ] && [ "$A" != "$B" ] && echo "PASS: kubelet $N dang gia han Node Lease"
done

# 167 — bầu chọn có phối hợp
LC="$(kubectl api-resources --api-group=coordination.k8s.io --no-headers 2>/dev/null | grep -ci leasecandidate)"
V1B="$(kubectl get --raw /apis/coordination.k8s.io | grep -c 'v1beta1')"
FG="$(sudo grep -c 'CoordinatedLeaderElection' "$M/kube-apiserver.yaml")"
echo "leasecandidate=$LC v1beta1=$V1B feature-gate=$FG" | tee "$EV/b2-cle.txt"
[ "$LC" -eq 0 ] && [ "$V1B" -eq 0 ] && [ "$FG" -eq 0 ] && echo 'PASS: bau chon leader co phoi hop dang tat'

# 198 — bốn dữ kiện và role
CPF="$(sudo grep -c -e '--cloud-provider' "$M/kube-controller-manager.yaml")"
MIG="$(sudo grep -c 'enable-leader-migration' "$M/kube-controller-manager.yaml")"
CCM="$(kubectl get deployment,daemonset,pod -A --no-headers 2>/dev/null | grep -ci 'cloud-controller-manager')"
PIDN="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.providerID}{"\n"}{end}' | grep -c .)"
CPN="$(kubectl get nodes -l node-role.kubernetes.io/control-plane --no-headers | wc -l)"
echo "cloud-provider=$CPF leader-migration=$MIG ccm=$CCM providerID=$PIDN control-plane=$CPN" | tee "$EV/b2-migration.txt"
{ [ "$CPF" -eq 0 ] && [ "$CCM" -eq 0 ] && [ "$PIDN" -eq 0 ] && [ "$CPN" -eq 1 ] && [ "$MIG" -eq 0 ]; } \
  && echo 'PASS: bon du kien loai tru Leader Migration'
kubectl -n kube-system get role 'system::leader-locking-kube-controller-manager' \
  -o jsonpath='{range .rules[*]}{.resources}{" "}{.resourceNames}{" "}{.verbs}{"\n"}{end}' | tee "$EV/b2-role-kcm.txt"
{ ! grep -q 'cloud-provider-extraction-migration' "$EV/b2-role-kcm.txt" \
  && ! kubectl -n kube-system get lease cloud-provider-extraction-migration >/dev/null 2>&1; } \
  && echo 'PASS: role chua duoc va cho Lease di tru, Lease di tru khong ton tai'

ls ~/bookstore/docs/adr/*-control-plane-truoc-nang-cap.md >/dev/null 2>&1 && echo 'PASS: co ADR'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính cho ADR</summary>

1. Không dùng được trường mới: khả năng **giới thiệu sau** emulation version thì không khả dụng. API
   bị gỡ ở bản mới **vẫn dùng được**: khả năng **bị loại bỏ sau** emulation version vẫn khả dụng.
   Emulation cắt cả hai chiều. Nó tách **thay binary** khỏi **đổi hành vi**: bước một thay binary mà
   hành vi y cũ, xác nhận Bookstore chạy ổn, bước hai mới nâng emulation version — lùi lại chỉ là đổi
   một flag.
2. Lease bảo đảm chỉ một instance chủ động điều khiển tại một thời điểm; với một instance thì không
   có gì để tranh, và khi nó khởi động lại thì không ai lập lịch thay. Pod đang chạy không bị ảnh
   hưởng — scheduler chỉ quyết định chỗ cho Pod **mới**. Pod mới do HPA yêu cầu nằm `Pending` cho tới
   khi scheduler giành lại Lease. `leaseTransitions` chỉ tăng khi thành phần khởi động lại.
3. Không thay thế: các ứng viên vẫn phối hợp qua **một Lease dùng chung**; thứ thêm vào là
   `LeaseCandidate` mang binary và emulation version. Khác biệt nằm ở cách chọn ai thắng: từ "ai cập
   nhật được Lease trước" (điều khiển đồng thời lạc quan qua `resourceVersion`) sang chọn **xác định**
   theo `OldestEmulationVersion`.
4. Không `--cloud-provider`, không `cloud-controller-manager`, `providerID` rỗng, control plane một
   node — bài 198 nói thẳng control plane một node không cần quy trình này. Bước đầu là vá role vì
   quyền mặc định của controller manager chỉ cho truy cập **Lease chính của nó**; Lease di trú dùng
   chung giữa hai manager là một Lease khác, đúng như `b2-role-kcm.txt` cho thấy.

</details>

## B3. [BS-12-03] Node có sẵn sàng để tắt máy không

**Loại:** Điều tra.

**Bối cảnh:** B5 và B7 sẽ tắt hẳn hai worker. Hai câu hỏi phải trả lời **trước**: máy có quay lại
đúng như cũ không — swap là nguyên nhân kinh điển khiến kubelet không lên sau reboot; và nếu ai đó
tắt máy mà quên drain thì kubelet có tự chấm dứt Pod êm không. Câu thứ hai quyết định drain là
"nên" hay "bắt buộc" trong runbook.

**Yêu cầu:**

1. Trên cả ba node: có thiết bị swap nào đang hoạt động không, tổng dung lượng swap, và có cấu hình
   nào sẽ bật lại swap sau reboot không.
2. Chứng minh Kubernetes nhìn thấy cùng sự thật: Node không báo dung lượng swap, và cả ba kubelet được
   cấu hình **từ chối** chạy khi có swap (giá trị hiệu lực, không phải giá trị trong file).
3. Graceful node shutdown trên hai worker: đọc hai thời lượng quyết định từ cấu hình hiệu lực, rồi
   chứng minh **ở tầng hệ điều hành** rằng kubelet không giữ khóa trì hoãn tắt máy nào.
4. Chứng minh `db-0` **không** phải "pod quan trọng" theo nghĩa mà graceful node shutdown dùng, dù nó
   mang PriorityClass `bookstore-critical`.
5. ADR `docs/adr/NNNN-swap-va-tat-node.md`; ghi kết luận "drain trước khi tắt máy: bắt buộc hay
   không" vào runbook.
6. ⭐ (trả phần hoãn của bài 47 — đọc lại mục
   [*Hành vi của Pod khi kubelet khởi động lại*](../47-pod-lifecycle-vi.md#kubelet-restarts) trước khi
   làm) Khởi động lại kubelet trên `lab-k8s-worker2`; chứng minh `db-0` không bị thay và container
   PostgreSQL không bị khởi động lại.

**Ràng buộc:** không bật swap, không sửa cấu hình kubelet, không đổi feature gate. ⭐ chỉ chạy trên
`lab-k8s-worker2` và chỉ khởi động lại dịch vụ, không đổi file nào.

**Đọc lại:**

- Bài [170](../170-swap-memory-management-vi.md):
  [*Hỗ trợ theo hệ điều hành*](../170-swap-memory-management-vi.md#hỗ-trợ-theo-hệ-điều-hành-operating-system-support),
  [*Các hành vi swap*](../170-swap-memory-management-vi.md#các-hành-vi-swap-swap-behaviors),
  [*Node báo cáo dung lượng swap*](../170-swap-memory-management-vi.md#node-báo-cáo-dung-lượng-swap-trong-trạng-thái-node-nodes-to-report-swap-capacity-as-part-of-node-status),
  [*Giới hạn swap với LimitedSwap*](../170-swap-memory-management-vi.md#how-is-the-swap-limit-being-determined-with-limitedswap),
  [*Lập lịch có nhận biết swap*](../170-swap-memory-management-vi.md#lập-lịch-có-nhận-biết-swap-swap-aware-scheduling).
- Bài [169](../169-node-shutdown-vi.md): [*Tắt node nhẹ nhàng*](../169-node-shutdown-vi.md#graceful-node-shutdown)
  — ghi chú về systemd inhibitor lock;
  [*Cấu hình tắt node nhẹ nhàng*](../169-node-shutdown-vi.md#cấu-hình-tắt-node-nhẹ-nhàng-configuring-graceful-node-shutdown);
  [*Xử lý tắt node không nhẹ nhàng*](../169-node-shutdown-vi.md#non-graceful-node-shutdown).
- Bài [210](../210-guaranteed-scheduling-critical-addon-pods-vi.md#marking-pod-as-critical) — pod quan
  trọng là pod nào.
- [A4.1 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites) —
  bạn đã tắt swap ở đâu, bằng cách nào.

**Bàn giao:** `docs/adr/NNNN-swap-va-tat-node.md`; dòng kết luận trong runbook;
`~/lab-evidence/12/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` cả ba node: không thiết bị swap, tổng swap 0, `/etc/fstab` không có dòng swap còn hiệu lực.
- `PASS:` không Node nào báo dung lượng swap; `failSwapOn` hiệu lực là `true` trên cả ba kubelet.
- `PASS:` hai thời lượng tắt nhẹ nhàng đều `0s` trên hai worker **và** không có inhibitor lock nào của
  kubelet — tính năng không hoạt động.
- `PASS:` `systemd-logind` đang chạy trên worker2.
- `PASS:` `value` của `bookstore-critical` thấp hơn ngưỡng của `system-cluster-critical`.
- `PASS:` (⭐) sau khi kubelet worker2 khởi động lại: UID của `db-0`, `startedAt` và `restartCount` của
  container `postgres` không đổi; node Ready trở lại.
- `PASS:` có ADR; runbook có dòng kết luận về drain.

**Câu hỏi thiết kế** — trả lời trong ADR:

1. Đề xuất "bật `LimitedSwap` trên worker2 cho DB đỡ OOM". Với QoS của từng Pod Bookstore trên node đó,
   Pod nào được swap, Pod nào không? `db-0` có được lợi gì không? Scheduler có tính swap khi đặt Pod
   không, và bài đề xuất cách phòng nào?
2. Ai đó `sudo shutdown -h now` trên worker2 khi `db-0` đang chạy, không drain. Trên cluster **này**,
   kubelet có chạy luồng kết thúc Pod của bài 47 (`preStop`, ân hạn 60 giây của `db`) không? Bạn dựa
   vào hai dữ kiện nào?
3. Giả sử giai đoạn 20 bật `shutdownGracePeriod=30s`, `shutdownGracePeriodCriticalPods=10s`. `db-0`
   rơi vào giai đoạn nào, có bao nhiêu giây, có đủ ân hạn 60 giây mà Lab 3a chọn cho PostgreSQL không?
   Drain còn cần không?
4. Vì sao dòng swap trong `/etc/fstab` quan trọng với **chính lab này** hơn mọi lab trước?

<details>
<summary>Gợi ý</summary>

- Ba câu hỏi về swap trên node: `swapon --show`, `free -m`, `/etc/fstab`.
- Trường swap của Node: lệnh go-template ở mục *Node báo cáo dung lượng swap* của bài 170.
- Cấu hình hiệu lực của kubelet: endpoint `configz` qua API server, như B1.
- Khóa trì hoãn tắt máy của systemd: `systemd-inhibit --list` trên chính node đó.
- Giá trị ưu tiên của hai PriorityClass hệ thống: `kubectl get priorityclass`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
EV=~/lab-evidence/12; WK=~/lab-work/12; W2=lab-k8s-worker2
NODES='lab-k8s-master lab-k8s-worker1 lab-k8s-worker2'
cfgz() { grep -o "\"$1\":[^,}]*" "$2" | head -1 | cut -d: -f2- | tr -d '"'; }

# Swap: ba câu hỏi khác nhau trên ba node
cat > "$WK/doc-swap.sh" <<'EOS'
swapon --show | wc -l
free -m | sed -n 's/^Swap:[[:space:]]*\([0-9][0-9]*\).*/\1/p'
grep -cE '^[^#].*[[:space:]]swap[[:space:]]' /etc/fstab
EOS
for n in $NODES; do
  if [ "$n" = lab-k8s-master ]; then OUT="$(sh "$WK/doc-swap.sh")"; else OUT="$(ssh "$n" 'sh -s' < "$WK/doc-swap.sh")"; fi
  echo "$n: swapon=$(echo "$OUT" | sed -n 1p) swaptotal=$(echo "$OUT" | sed -n 2p) fstab=$(echo "$OUT" | sed -n 3p)"
done > "$EV/b3-swap-node.txt"
cat "$EV/b3-swap-node.txt"
[ "$(grep -cF 'swapon=0 swaptotal=0 fstab=0' "$EV/b3-swap-node.txt")" -eq 3 ] \
  && echo 'PASS: ca ba node tat swap va fstab khong bat lai sau reboot'

# Kubernetes nhìn swap
kubectl get nodes -o json > "$EV/b3-nodes.json"
for n in $NODES; do
  kubectl get --raw "/api/v1/nodes/$n/proxy/configz" > "$EV/b3-configz-$n.json" 2>/dev/null
  echo "$n failSwapOn=$(cfgz failSwapOn "$EV/b3-configz-$n.json") memorySwap=$(grep -o '"memorySwap":{[^}]*}' "$EV/b3-configz-$n.json")"
done | tee "$EV/b3-kubelet-swap.txt"
[ "$(grep -c '"swap"' "$EV/b3-nodes.json")" -eq 0 ] && [ "$(grep -c 'failSwapOn=true' "$EV/b3-kubelet-swap.txt")" -eq 3 ] \
  && echo 'PASS: Node khong bao swap; ca ba kubelet tu choi swap'

# Graceful node shutdown: cấu hình hiệu lực và inhibitor lock
for n in lab-k8s-worker1 lab-k8s-worker2; do
  SGP="$(cfgz shutdownGracePeriod "$EV/b3-configz-$n.json")"
  SGC="$(cfgz shutdownGracePeriodCriticalPods "$EV/b3-configz-$n.json")"
  # Đọc lỗi thì LOCK=loi, không phải 0 — dòng tóm tắt khi đó không khớp gate.
  if I="$(ssh -o BatchMode=yes "$n" 'systemd-inhibit --list --no-pager')"; then
    LOCK="$(printf '%s\n' "$I" | grep -ci kubelet || true)"
  else LOCK=loi; fi
  echo "$n shutdownGracePeriod=${SGP:-<vang>} shutdownGracePeriodCriticalPods=${SGC:-<vang>} khoa_kubelet=$LOCK"
done | tee "$EV/b3-shutdown.txt"
[ "$(grep -c 'shutdownGracePeriod=0s shutdownGracePeriodCriticalPods=0s khoa_kubelet=0' "$EV/b3-shutdown.txt")" -eq 2 ] \
  && echo 'PASS: graceful node shutdown KHONG hoat dong tren hai worker — drain truoc khi tat may la bat buoc'
[ "$(ssh "$W2" 'systemctl is-active systemd-logind')" = 'active' ] && echo 'PASS: systemd-logind dang chay tren worker2'

# Pod quan trọng
V_BS="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}')"
V_SC="$(kubectl get priorityclass system-cluster-critical -o jsonpath='{.value}')"
echo "bookstore-critical=$V_BS system-cluster-critical=$V_SC" | tee "$EV/b3-priority.txt"
[ "$V_BS" -lt "$V_SC" ] && echo 'PASS: db-0 khong phai pod quan trong theo nghia bai 210'

ls ~/bookstore/docs/adr/*-swap-va-tat-node.md >/dev/null 2>&1 && grep -qi 'drain' ~/bookstore/docs/runbook-bao-tri-node.md \
  && echo 'PASS: co ADR va dong ket luan ve drain trong runbook'
```

Phần ⭐ — khởi động lại kubelet trên worker2:

```bash
NS=bookstore; EV=~/lab-evidence/12; W2=lab-k8s-worker2
jp() { kubectl -n "$NS" get pod db-0 -o jsonpath="$1"; }
C='{.status.containerStatuses[?(@.name=="postgres")]'
U0="$(jp '{.metadata.uid}')"; S0="$(jp "$C.state.running.startedAt}")"; R0="$(jp "$C.restartCount}")"
K0="$(ssh "$W2" 'systemctl show kubelet -p ActiveEnterTimestamp')"
ssh "$W2" 'sudo systemctl restart kubelet'
sleep 10
kubectl wait --for=condition=Ready "node/$W2" --timeout=300s
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s
K1="$(ssh "$W2" 'systemctl show kubelet -p ActiveEnterTimestamp')"
U1="$(jp '{.metadata.uid}')"; S1="$(jp "$C.state.running.startedAt}")"; R1="$(jp "$C.restartCount}")"
printf 'kubelet: %s -> %s\nuid: %s -> %s\nstartedAt: %s -> %s\nrestartCount: %s -> %s\n' \
  "$K0" "$K1" "$U0" "$U1" "$S0" "$S1" "$R0" "$R1" | tee "$EV/b3-kubelet-restart.txt"
{ [ "$K0" != "$K1" ] && [ "$U0" = "$U1" ] && [ "$S0" = "$S1" ] && [ "$R0" = "$R1" ]; } \
  && echo 'PASS: kubelet da khoi dong lai, db-0 va PostgreSQL chay tiep nhu cu'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính cho ADR</summary>

1. Với `LimitedSwap` chỉ Pod QoS **Burstable** được swap: `web`, `api`, `node-agent` (theo ngân sách ở
   [mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch)). `db-0` là **Guaranteed** nên
   bị cấm — đề xuất không mang lại gì cho DB, chỉ làm `web`, `api` khó dự đoán hơn. Scheduler không
   xét swap vì Pod chỉ request `memory`; bài đề xuất gắn taint cho node có swap. Quyết định: không bật.
2. Không. Hai thời lượng bằng `0s` và kubelet không giữ inhibitor lock nào, nên systemd tắt máy mà
   không chờ kubelet: không `preStop`, không ân hạn — đúng nhánh *tắt node không nhẹ nhàng* của bài
   169. API server vẫn tin `db-0` đang chạy cho tới khi node controller phát hiện node mất liên lạc.
   Vì vậy trên cluster này **drain trước khi tắt máy là bắt buộc**.
3. Pod quan trọng là Pod mang `system-cluster-critical` hoặc `system-node-critical` (bài 210);
   `bookstore-critical` thấp hơn nhiều. `db-0` rơi vào nhóm **thông thường**: 20 giây đầu (30 − 10),
   ít hơn 60 giây ân hạn của `db`. Graceful node shutdown với cấu hình đó là lưới an toàn cuối, không
   thay drain: drain tôn trọng `terminationGracePeriodSeconds` của từng Pod.
4. Vì B5 và B7 reboot hai worker. Dòng swap chưa comment trong `fstab` làm swap bật lại khi boot, và
   kubelet mặc định không khởi động trên node có swap: node quay lại `NotReady`.

Phần ⭐: bài 47 nói khởi động lại kubelet **không** dừng các Pod cục bộ, và từ phiên bản baseline,
mặc định kubelet không đổi trạng thái container sau khi khởi động lại. Pod đi qua lần khởi động lại
đó với cùng UID, cùng container. Muốn dừng Pod trên node thì dùng `kubectl drain` — đúng câu cuối
của mục bài 47.

</details>

## B4. [BS-12-04] Công cụ đo sẵn sàng qua Ingress

**Loại:** Tính năng.

**Bối cảnh:** "không gián đoạn" là một khẳng định phải đo được. Đo từ góc người dùng: HTTPS qua
Traefik tới trang chủ và tới `/api/books`, liên tục, ghi mã trả về và thời gian của từng request.
Sprint này thêm vào repo hai công cụ: một probe chạy nền, một bộ tóm tắt. Chúng là công cụ — chép
nguyên văn; phần học là đặt chúng đúng chỗ và đọc đúng con số.

**Yêu cầu:**

1. `tests/availability-probe.sh` và `tests/availability-report.sh` — chép nguyên văn từ khối dưới.
2. Xác định điểm vào mà probe dùng: IP nào, cổng nào, và vì sao điểm vào đó còn sống khi **một worker
   bất kỳ** tắt.
3. Bộ tóm tắt đếm đúng trên một file mẫu có lỗi cố ý.
4. Probe chạy nền ít nhất hai phút khi hệ thống yên tĩnh cho **0** request lỗi trên cả hai đường dẫn —
   đây là đường nền để so sánh ở B5, B7.
5. Trả lời câu hỏi thiết kế trong `docs/runbook-bao-tri-node.md`, mục *Cách đo*.

**Ràng buộc:** probe chạy trên `lab-k8s-master`, đi qua Traefik bằng HTTPS; không gọi thẳng IP Pod hay
ClusterIP — đó không phải đường của người dùng.

**Đọc lại:** [mục 3.1](DU-AN-BOOKSTORE.md#31-sơ-đồ-tổng-thể) và
[mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe) của file dự án;
[Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) — Traefik được đưa ra ngoài bằng NodePort.

**Bàn giao:** hai script trong `tests/`; mục *Cách đo* trong runbook; `~/lab-evidence/12/b4-*`.

**Nghiệm thu:**

- `PASS:` hai công cụ có trong repo và chạy được.
- `PASS:` bộ tóm tắt đếm đúng trên file mẫu.
- `PASS:` probe nền có ít nhất 200 mẫu mỗi đường dẫn, 0 lỗi.
- `PASS:` runbook có mục *Cách đo*.

Hai công cụ, chép nguyên văn:

```bash
cat > ~/bookstore/tests/availability-probe.sh <<'EOF'
#!/usr/bin/env bash
# Đo sẵn sàng của Bookstore từ góc người dùng: HTTPS qua Traefik tới / và /api/books — Lab 12.
# Dùng:  tests/availability-probe.sh <file.csv> [giay-giua-hai-luot]
# Dừng:  tạo file <file.csv>.stop, hoặc gửi TERM.
# ENTRY_IP (tùy chọn): IP node làm điểm vào; mặc định là IP của lab-k8s-master.
set -u
OUT="${1:?dung: availability-probe.sh <file.csv> [giay]}"
STEP="${2:-1}"
HOST=bookstore.lab.local
NP="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}' 2>/dev/null)"
[ -n "$NP" ] || NP="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.port==443)].nodePort}' 2>/dev/null)"
IP="${ENTRY_IP:-$(kubectl get node lab-k8s-master -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')}"
if [ -z "$NP" ] || [ -z "$IP" ]; then echo "probe: khong xac dinh duoc diem vao (np=$NP ip=$IP)" >&2; exit 1; fi
echo "probe: https://$HOST:$NP qua $IP, moi ${STEP}s, ghi $OUT" >&2
trap 'exit 0' TERM INT
rm -f "$OUT.stop"
echo 'epoch_ms,duong_dan,ma,curl_exit,thoi_gian_s' > "$OUT"
while [ ! -e "$OUT.stop" ]; do
  for P in / /api/books; do
    R="$(curl -sk -o /dev/null --max-time 2 --resolve "$HOST:$NP:$IP" \
           -w '%{http_code},%{time_total}' "https://$HOST:$NP$P")"
    E=$?
    echo "$(date +%s%3N),$P,${R%%,*},$E,${R##*,}" >> "$OUT"
  done
  sleep "$STEP" & wait $!
done
echo "probe: dung theo $OUT.stop" >&2
EOF

cat > ~/bookstore/tests/availability-report.sh <<'EOF'
#!/usr/bin/env bash
# Tóm tắt file CSV của tests/availability-probe.sh — Lab 12.
# Dùng: tests/availability-report.sh <file.csv> [tu_epoch_ms] [den_epoch_ms]
# Mỗi đường dẫn: số mẫu, số lỗi (mã khác 200), số đợt lỗi liên tiếp, đợt dài nhất, mã lỗi/exit curl.
set -eu
F="${1:?dung: availability-report.sh <file.csv> [tu_ms] [den_ms]}"
TU="${2:-0}"; DEN="${3:-99999999999999}"
awk -F, -v tu="$TU" -v den="$DEN" '
function dot_ra(p, d) { dl[++nd] = sprintf("dot_loi duong_dan=%s bat_dau_ms=%.0f dai_s=%.1f", p, bd[p], d / 1000) }
BEGIN { np = split("/ /api/books", P, " ") }
NR == 1 || NF < 5 { next }
{
  t = $1 + 0; p = $2
  if (t < tu + 0 || t > den + 0) next
  if (t0 == "") t0 = t
  t1 = t; n[p]++
  if ($3 == "200") {
    if (dang[p]) { d = t - bd[p]; if (d > mx[p]) mx[p] = d; dot_ra(p, d); dang[p] = 0 }
  } else {
    loi[p]++; k = $3 "/" $4
    if (!((p, k) in dem)) ds[p] = ds[p] (ds[p] == "" ? "" : ",") k
    dem[p, k]++
    if (!dang[p]) { dang[p] = 1; bd[p] = t; sd[p]++ }
    cuoi[p] = t
  }
}
END {
  for (i = 1; i <= np; i++) {
    p = P[i]
    if (dang[p]) { d = cuoi[p] - bd[p]; if (d > mx[p]) mx[p] = d; dot_ra(p, d) }
    m = "-"
    if (ds[p] != "") {
      m = ""; c = split(ds[p], K, ",")
      for (j = 1; j <= c; j++) m = m (j > 1 ? "," : "") K[j] ":" dem[p, K[j]]
    }
    printf "duong_dan=%s mau=%d loi=%d so_dot_loi=%d dot_dai_nhat_s=%.1f ma_loi=%s\n", \
      p, n[p], loi[p], sd[p], mx[p] / 1000, m
  }
  printf "khoang_do_s=%.0f\n", (t1 - t0) / 1000
  for (i = 1; i <= nd; i++) print dl[i]
}' "$F" | while IFS= read -r L; do
  case "$L" in
    dot_loi*) MS="${L#*bat_dau_ms=}"; MS="${MS%% *}"; echo "$L gio=$(date -d "@$((MS / 1000))" +%T)" ;;
    *) echo "$L" ;;
  esac
done
EOF
chmod +x ~/bookstore/tests/availability-*.sh
```

Cách dùng trong các ticket sau, trong cùng một phiên shell:

```bash
CSV=~/lab-evidence/12/<ten>.csv
nohup ~/bookstore/tests/availability-probe.sh "$CSV" 0.5 > "$CSV.log" 2>&1 &
PROBE=$!
# ... làm việc ...
touch "$CSV.stop"; wait "$PROBE"
~/bookstore/tests/availability-report.sh "$CSV"
```

Một dòng `ma_loi=000/7:3` nghĩa là ba request không nhận được phản hồi HTTP nào, curl thoát mã 7.
`502/0:1` nghĩa là một request nhận phản hồi HTTP 502.

**Câu hỏi thiết kế** — trả lời trong mục *Cách đo* của runbook:

1. File hosts trên máy host Windows trỏ `bookstore.lab.local` tới IP **một** worker. Vì sao probe vào
   bằng IP của master thay vì IP đó? Đo như vậy thì bỏ sót trải nghiệm gì của người dùng thật trong lúc
   bảo trì đúng worker đó?
2. `000` kèm exit curl khác 0, `502`, `503` — mỗi loại do tầng nào sinh ra trong đường
   Traefik → `web` → `api` → `db`?
3. Probe 0,5 giây một lượt bỏ sót được gì? Kể ít nhất hai thứ của Bookstore mà probe này không thấy.

<details>
<summary>Gợi ý</summary>

- NodePort mở trên **mọi** node, kể cả control plane; kube-proxy trên node đó chuyển tiếp tới Pod
  Traefik ở bất kỳ đâu — trừ khi Service của Traefik khai `externalTrafficPolicy: Local`. Kiểm:
  `kubectl -n traefik get svc traefik -o jsonpath='{.spec.externalTrafficPolicy}'`.
- File mẫu cho yêu cầu 3 nằm trong script nghiệm thu; tự đoán trước kết quả rồi mới chạy.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
EV=~/lab-evidence/12; WK=~/lab-work/12; T=~/bookstore/tests
test -x "$T/availability-probe.sh" && test -x "$T/availability-report.sh" && echo 'PASS: hai cong cu co trong repo, chay duoc'
kubectl -n traefik get svc traefik -o jsonpath='type={.spec.type} etp={.spec.externalTrafficPolicy}{"\n"}' | tee "$EV/b4-traefik-svc.txt"

cat > "$WK/mau.csv" <<'EOM'
epoch_ms,duong_dan,ma,curl_exit,thoi_gian_s
1000,/,200,0,0.010
1000,/api/books,200,0,0.020
2000,/,000,7,0.000
2000,/api/books,502,0,0.010
3000,/,000,7,0.000
3000,/api/books,200,0,0.020
4000,/,200,0,0.010
4000,/api/books,200,0,0.020
EOM
"$T/availability-report.sh" "$WK/mau.csv" | tee "$EV/b4-mau.txt"
{ grep -qx 'duong_dan=/ mau=4 loi=2 so_dot_loi=1 dot_dai_nhat_s=2.0 ma_loi=000/7:2' "$EV/b4-mau.txt" \
  && grep -qx 'duong_dan=/api/books mau=4 loi=1 so_dot_loi=1 dot_dai_nhat_s=1.0 ma_loi=502/0:1' "$EV/b4-mau.txt"; } \
  && echo 'PASS: bo tom tat dem dung tren file mau'

CSV="$EV/b4-nen.csv"
nohup "$T/availability-probe.sh" "$CSV" 0.5 > "$CSV.log" 2>&1 &
PROBE=$!
sleep 130
touch "$CSV.stop"; wait "$PROBE"
cat "$CSV.log"
"$T/availability-report.sh" "$CSV" | tee "$EV/b4-nen.txt"
BAD=0
for P in / /api/books; do
  L="$(grep "^duong_dan=$P " "$EV/b4-nen.txt")"
  N="$(echo "$L" | sed -n 's/.* mau=\([0-9]*\) .*/\1/p')"; E="$(echo "$L" | sed -n 's/.* loi=\([0-9]*\) .*/\1/p')"
  { [ "${N:-0}" -ge 200 ] && [ "${E:-1}" -eq 0 ]; } || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: probe nen du mau, 0 loi tren ca hai duong dan'
grep -qi 'cách đo\|cach do' ~/bookstore/docs/runbook-bao-tri-node.md && echo 'PASS: runbook co muc Cach do'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính cho mục <em>Cách đo</em></summary>

1. Probe đo **hệ thống**, nên điểm vào phải là thứ không bị bảo trì: master không bao giờ tắt trong
   lab, và NodePort trên master vẫn chuyển tới Traefik dù Traefik chạy ở worker nào (`etp=Cluster`).
   Cái giá: người dùng thật đi qua IP một worker trong file hosts — bảo trì đúng worker đó là họ mất
   truy cập hoàn toàn, dù hệ thống vẫn khỏe. Đường vào chưa có HA; B8 ghi thành nợ kỹ thuật.
2. `000` + exit khác 0: không có phản hồi HTTP — không ai nhận kết nối ở NodePort hoặc ở Pod
   Traefik (exit 7 từ chối kết nối, 28 hết giờ, 35/56 TLS hoặc kết nối bị cắt). `502`, `503` từ
   Traefik: không có Pod `web` Ready để chuyển tới, hoặc Pod `web` đóng kết nối. `502` từ nginx:
   Service `api` không có đích. `503` từ PostgREST: `api` sống nhưng mất DB.
3. Bỏ sót đợt lỗi ngắn hơn khoảng lấy mẫu, độ trễ tăng mà chưa lỗi; không thấy ghi dữ liệu (POST),
   không thấy Job `db-migrate`, CronJob `db-backup`, không thấy HPA mất metric. Probe là một chỉ số,
   không phải toàn bộ sức khỏe.

</details>

## B5. [BS-12-05] Bảo trì lab-k8s-worker1: cordon → drain → tắt → bật → uncordon

**Loại:** Vận hành.

**Bối cảnh:** kernel của `lab-k8s-worker1` cần vá và reboot. Đây đúng là dòng "Bảo trì một worker"
trong [mục 3.7](DU-AN-BOOKSTORE.md#37-chịu-lỗi-và-vận-hành): `web`, `api` phải không gián đoạn nhờ PDB
và bản sao trên node kia. Worker1 không giữ dữ liệu của `db`, nên đây là ca dễ — và chính vì dễ, mọi
request lỗi đo được ở đây đều là thiếu sót thiết kế cần giải thích.

**Yêu cầu:**

1. Kiểm **tiền điều kiện** và ghi thành checklist ở mục 2 của runbook: mọi node Ready; `web`, `api`
   đủ bản sao Ready và có Pod trên cả hai worker; PDB của cả hai đang cho phép ít nhất một gián đoạn;
   `db-0` không nằm trên worker1.
2. Probe chạy nền suốt từ trước khi đánh dấu node tới sau khi cân bằng lại.
3. Đưa worker1 ra khỏi lập lịch, rồi trục xuất mọi Pod trục xuất được bằng cơ chế **tôn trọng PDB**;
   node chỉ còn Pod DaemonSet.
4. Tắt máy worker1 thật sự (VM *Powered off*), giữ tắt ít nhất hai phút. Trong lúc đó `web`, `api` phục
   vụ đầy đủ từ worker2.
5. Bật lại, chờ node Ready, chứng minh node **vẫn chưa** nhận Pod mới; rồi đưa nó trở lại lập lịch.
6. Cân bằng lại: mỗi Deployment `web`, `api` lại có Pod trên cả hai worker, bằng cách không vi phạm
   PDB.
7. Ghi mốc thời gian bằng hàm `moc` bên dưới, đủ tám mốc theo đúng thứ tự.
8. Postmortem `docs/postmortem/12-bao-tri-worker1.md`: số đo của probe, phân loại **mọi** request lỗi
   (thời điểm, mã, tầng gây lỗi, vì sao, sửa ở đâu), đối chiếu với dự đoán ở B1.
9. Mục 2 của runbook: quy trình bảo trì một worker không giữ dữ liệu.

**Ràng buộc:** không `--force`, không `--disable-eviction`; không gỡ taint bằng tay; không đổi
`replicas`, HPA hay PDB trong repo để "làm cho qua"; một node một lúc; không tắt master. HPA `web` vẫn
chạy trong suốt cuộc bảo trì: số bản sao `web` có thể đổi giữa chừng, nên mọi kiểm tra so `ready` với
số bản sao **hiện tại** chứ không với một con số cố định. Ngay sau `deploy.sh`, `web` bắt đầu ở một
bản sao cho tới khi HPA nâng lên `minReplicas` (Lab 11b) — đừng bắt đầu bảo trì trong khoảng đó.

**Đọc lại:**

- Bài [23](../23-nodes-vi.md#manual-node-administration) — đánh dấu unschedulable, và ghi chú về Pod
  DaemonSet trên node unschedulable.
- Bài [53](../53-disruptions-vi.md): [*Ngân sách gián đoạn Pod*](../53-disruptions-vi.md#pod-disruption-budgets),
  [*Ví dụ về PodDisruptionBudget*](../53-disruptions-vi.md#pdb-example).
- Bài [143](../143-api-eviction-vi.md#cách-hoạt-động-của-eviction-khởi-phát-qua-api-how-api-initiated-eviction-works)
  — trình tự xóa Pod sau khi eviction được chấp nhận.
- Bài [339](../339-configure-pdb-vi.md#kiểm-tra-trạng-thái-của-pdb-check-the-status-of-the-pdb) —
  đọc `disruptionsAllowed`, `currentHealthy`, `desiredHealthy`.
- Bài [171](../171-node-autoscaling-vi.md#consolidation) — định nghĩa Node rỗng.
- Bài [139](../139-taint-and-toleration-vi.md#taint-nodes-by-condition) — toleration mà DaemonSet
  controller tự thêm.

**Bàn giao:** postmortem; mục 2 của runbook; trong `~/lab-evidence/12/`: `b5-truoc.txt`,
`b5-su-kien.txt`, `b5-probe.csv`, `b5-drain.txt` (toàn bộ output của lệnh drain), `b5-tom-tat.txt`.

Hai công cụ ghi bằng chứng, chép nguyên văn — chạy **trước** khi đánh dấu node:

```bash
EV=~/lab-evidence/12; W=lab-k8s-worker1
MOC="$EV/b5-su-kien.txt"; : > "$MOC"
moc() { echo "$(date +%s%3N) $(date -Is) $1" | tee -a "$MOC"; }
{ echo "boot_id=$(kubectl get node "$W" -o jsonpath='{.status.nodeInfo.bootID}')"
  echo "agent_uid=$(kubectl -n bookstore-system get pod -l app.kubernetes.io/component=agent \
       --field-selector "spec.nodeName=$W" -o jsonpath='{.items[0].metadata.uid}')"
} > "$EV/b5-truoc.txt"; cat "$EV/b5-truoc.txt"
```

Tám mốc, đúng tên và đúng thứ tự: `bat-dau`, `cordon`, `drain-xong`, `tat-may`, `node-ready`,
`uncordon`, `can-bang-xong`, `ket-thuc`. Mốc `bat-dau` ghi **sau** khi probe đã chạy; mốc `ket-thuc`
ghi **trước** khi dừng probe.

**Nghiệm thu:**

- `PASS:` drain dùng Eviction API cho Pod của dự án, không Pod nào bị xóa thẳng, drain báo hoàn tất.
- `PASS:` worker1 chỉ còn Pod DaemonSet — Node rỗng; PDB `web`, `api` vẫn giữ `currentHealthy` ≥
  `desiredHealthy`; `web`, `api` đủ bản sao Ready.
- `PASS:` khi máy tắt: `Ready` của worker1 là `Unknown`, không Pod nào của dự án nằm trên worker1,
  `db-0` không bị ảnh hưởng.
- `PASS:` máy đã thật sự khởi động lại (`bootID` đổi); node Ready và nhận lập lịch.
- `PASS:` Pod `node-agent` trên worker1 vẫn là Pod cũ (cùng UID).
- `PASS:` `web`, `api` lại có Pod trên cả hai worker.
- `PASS:` đủ tám mốc đúng thứ tự; probe phủ trọn cửa sổ.
- `PASS:` 0 request lỗi — **hoặc** postmortem chép đúng hai dòng số đo của bộ tóm tắt và phân loại
  từng đợt lỗi.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Ba cơ chế nào cùng làm `web`, `api` không gián đoạn khi worker1 bị drain? Cơ chế nào là của
   Kubernetes, cơ chế nào là của chính Bookstore (Lab 3a, 3c, 7a)?
2. Nếu có request lỗi: tầng nào, vì sao PDB của Bookstore không ngăn được, và sửa ở đâu — trong repo
   Bookstore hay ở hạ tầng nền?
3. Vì sao drain từ chối chạy nếu bạn không bỏ qua Pod DaemonSet một cách tường minh? Pod `node-agent`
   trên worker1 đã đi đâu trong lúc máy tắt, và vì sao sau reboot nó vẫn là Pod cũ?
4. Node `Ready` mà vẫn không nhận Pod mới: trạng thái đó nằm ở đâu, trên đĩa worker1 hay trong API
   server? Vì sao uncordon không đưa Pod về, và vì sao bạn chọn cách cân bằng lại như đã làm?
5. PDB `web` khai `maxUnavailable: 1`. Nếu bạn drain ngay sau `deploy.sh`, lúc HPA chưa nâng `web` khỏi
   một bản sao, PDB có chặn việc trục xuất Pod `web` duy nhất không? Tiền điều kiện nào của bạn ngăn
   tình huống đó?

<details>
<summary>Gợi ý</summary>

1. Đọc `kubectl drain --help`: cờ bỏ qua Pod DaemonSet, cờ chấp nhận xóa dữ liệu `emptyDir`, cờ
   `--timeout`. Chạy drain trần một lần để chính lệnh nói nó thiếu gì.
2. Lưu output của drain: `kubectl drain ... > ~/lab-evidence/12/b5-drain.txt 2>&1`.
3. Tắt máy từ master: `ssh lab-k8s-worker1 'sudo shutdown -h now'`. Xác nhận *Powered off* và bật lại
   bằng khối PowerShell trong lời giải, hoặc bằng giao diện VMware.
4. Cân bằng lại mà vẫn tôn trọng PDB: gọi Eviction API cho **một** Pod trên node đang đông, như
   [Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md) đã làm; ReplicaSet tạo Pod thay thế và scheduler đặt nó theo
   ràng buộc phân bố. `kubectl rollout restart` cũng được nhưng thay **mọi** Pod.

</details>

<details>
<summary>Script nghiệm thu — chạy theo ba thời điểm</summary>

Phần A — chạy ngay sau khi drain xong, **trước** khi tắt máy:

```bash
NS=bookstore; EV=~/lab-evidence/12; W1=lab-k8s-worker1
grep -q 'evicting pod bookstore/' "$EV/b5-drain.txt" && echo 'PASS: drain dung Eviction API voi Pod cua du an'
grep -q '^deleting pod' "$EV/b5-drain.txt" && echo 'FAIL: co Pod bi xoa thang' || echo 'PASS: khong Pod nao bi xoa thang'
grep -q "node/$W1 drained" "$EV/b5-drain.txt" && echo 'PASS: drain bao hoan tat'
KHAC="$(kubectl get pods -A --field-selector "spec.nodeName=$W1" \
  -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{"\n"}{end}' | grep -vc '^DaemonSet$')"
[ "$(kubectl get node "$W1" -o jsonpath='{.spec.unschedulable}')" = 'true' ] && [ "$KHAC" -eq 0 ] \
  && echo 'PASS: worker1 unschedulable, chi con Pod DaemonSet — Node rong theo bai 171'
for P in web api; do
  set -- $(kubectl -n "$NS" get pdb "$P" -o jsonpath='{.status.currentHealthy} {.status.desiredHealthy}')
  [ "${1:-0}" -ge "${2:-1}" ] && echo "PASS: PDB $P giu currentHealthy=$1 >= desiredHealthy=$2"
  W="$(kubectl -n "$NS" get deployment "$P" -o jsonpath='{.spec.replicas}')"
  R="$(kubectl -n "$NS" get deployment "$P" -o jsonpath='{.status.readyReplicas}')"
  [ "${R:-0}" = "$W" ] && echo "PASS: $P du $R/$W ban sao Ready"
done
```

Phần B — chạy khi worker1 đã *Powered off* (chờ tới khi dòng đầu PASS):

```bash
NS=bookstore; EV=~/lab-evidence/12; W1=lab-k8s-worker1
for i in $(seq 1 120); do
  ST="$(kubectl get node "$W1" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
  [ "$ST" = 'Unknown' ] && break; sleep 5
done
[ "$ST" = 'Unknown' ] && echo 'PASS: worker1 Ready=Unknown — control plane mat nhip tim'
kubectl -n "$NS" get pods -o wide > "$EV/b5-khi-tat.txt"
TREN_W1="$(kubectl -n "$NS" get pod -l app.kubernetes.io/part-of=bookstore --field-selector=status.phase=Running \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | grep -c "^$W1$")"
[ "$TREN_W1" -eq 0 ] && echo 'PASS: khong Pod nao cua bookstore nam tren worker1'
[ "$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = 'True' ] \
  && echo 'PASS: db-0 khong bi anh huong'
[ "$(tail -n 20 "$EV/b5-probe.csv" | awk -F, '$3 != "200"' | wc -l)" -eq 0 ] \
  && echo 'PASS: 20 mau gan nhat cua probe deu 200'
```

Phần C — chạy sau khi đã dừng probe và viết xong postmortem:

```bash
NS=bookstore; EV=~/lab-evidence/12; W1=lab-k8s-worker1; MOC="$EV/b5-su-kien.txt"
PM=~/bookstore/docs/postmortem/12-bao-tri-worker1.md
tm() { awk -v m="$1" '$3 == m {print $1; exit}' "$MOC"; }
T=0; BAD=0
for M in bat-dau cordon drain-xong tat-may node-ready uncordon can-bang-xong ket-thuc; do
  X="$(tm "$M")"
  if [ -z "$X" ] || [ "$X" -lt "$T" ]; then echo "FAIL: moc $M thieu hoac sai thu tu"; BAD=1; else T="$X"; fi
done
[ "$BAD" -eq 0 ] && echo 'PASS: du tam moc theo dung thu tu'

B0="$(sed -n 's/^boot_id=//p' "$EV/b5-truoc.txt")"; B1="$(kubectl get node "$W1" -o jsonpath='{.status.nodeInfo.bootID}')"
[ -n "$B0" ] && [ "$B0" != "$B1" ] && echo 'PASS: worker1 da khoi dong lai that (bootID doi)'
{ [ "$(kubectl get node "$W1" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = 'True' ] \
  && [ -z "$(kubectl get node "$W1" -o jsonpath='{.spec.unschedulable}')" ]; } && echo 'PASS: worker1 Ready va nhan lap lich'
A0="$(sed -n 's/^agent_uid=//p' "$EV/b5-truoc.txt")"
A1="$(kubectl -n bookstore-system get pod -l app.kubernetes.io/component=agent --field-selector "spec.nodeName=$W1" -o jsonpath='{.items[0].metadata.uid}')"
[ -n "$A0" ] && [ "$A0" = "$A1" ] && echo 'PASS: node-agent tren worker1 van la Pod cu'
for D in web api; do
  NN="$(kubectl -n "$NS" get pod -l "app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=$D" \
    --field-selector=status.phase=Running -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | grep -c .)"
  [ "$NN" -ge 2 ] && echo "PASS: $D lai co Pod tren $NN node"
done

F1="$(awk -F, 'NR == 2 {print $1}' "$EV/b5-probe.csv")"; FL="$(tail -n 1 "$EV/b5-probe.csv" | cut -d, -f1)"
[ "$F1" -le "$(tm cordon)" ] && [ "$FL" -ge "$(tm can-bang-xong)" ] && echo 'PASS: probe phu tron cua so bao tri'
~/bookstore/tests/availability-report.sh "$EV/b5-probe.csv" > "$EV/b5-tom-tat.txt"; cat "$EV/b5-tom-tat.txt"
L1="$(grep '^duong_dan=/ ' "$EV/b5-tom-tat.txt")"; L2="$(grep '^duong_dan=/api/books ' "$EV/b5-tom-tat.txt")"
E1="$(echo "$L1" | sed -n 's/.* loi=\([0-9]*\) .*/\1/p')"; E2="$(echo "$L2" | sed -n 's/.* loi=\([0-9]*\) .*/\1/p')"
if [ "$((E1 + E2))" -eq 0 ]; then
  echo 'PASS: khong gian doan — 0 request loi tren ca hai duong dan'
else
  echo "DO DUOC: $E1 loi tren /, $E2 loi tren /api/books"
  grep -qF "$L1" "$PM" && grep -qF "$L2" "$PM" && echo 'PASS: postmortem chep dung so do va phai phan loai tung dot loi'
fi
test -s "$PM" && grep -q '^## 2\.' ~/bookstore/docs/runbook-bao-tri-node.md && echo 'PASS: co postmortem va muc 2 cua runbook'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Toàn bộ trong một phiên shell trên master, sau khi đã chạy khối công cụ ở *Bàn giao*:

```bash
NS=bookstore; W1=lab-k8s-worker1; CSV="$EV/b5-probe.csv"

# 1. Tiền điều kiện
kubectl get nodes
kubectl -n "$NS" get pdb
kubectl -n "$NS" get pods -o wide -l app.kubernetes.io/part-of=bookstore

# 2. Probe nền
nohup ~/bookstore/tests/availability-probe.sh "$CSV" 0.5 > "$CSV.log" 2>&1 &
PROBE=$!; sleep 15; moc bat-dau

# 3. Đánh dấu, rồi trục xuất
kubectl cordon "$W1"; moc cordon
kubectl drain "$W1" --ignore-daemonsets --delete-emptydir-data --timeout=600s > "$EV/b5-drain.txt" 2>&1
echo "drain exit=$?"; cat "$EV/b5-drain.txt"; moc drain-xong
# -> chạy Phần A của script nghiệm thu

# 4. Tắt máy
ssh "$W1" 'sudo shutdown -h now' || true; moc tat-may
```

Chạy trên **máy host**, PowerShell — xác nhận đã tắt, chờ, chạy Phần B trên master, rồi bật lại:

```powershell
$w1 = $vmx | Where-Object { $_ -like '*worker1*' }
if ((& $vmrun -T ws list) -notcontains $w1) { "PASS: worker1 da Powered off" } else { "CHUA TAT: doi roi chay lai" }
# ... chờ ít nhất hai phút, chạy Phần B trên master ...
& $vmrun -T ws start $w1
```

Quay lại master:

```bash
kubectl wait --for=condition=Ready "node/$W1" --timeout=600s; moc node-ready
kubectl get node "$W1"                 # vẫn SchedulingDisabled
kubectl uncordon "$W1"; moc uncordon

# Cân bằng lại: Deployment nào còn dồn một node thì evict một Pod của nó qua Eviction API
can_bang() {
  for D in web api; do
    L="app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=$D"
    N="$(kubectl -n "$NS" get pod -l "$L" --field-selector=status.phase=Running \
         -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | grep -c .)"
    [ "$N" -ge 2 ] && continue
    P="$(kubectl -n "$NS" get pod -l "$L" -o jsonpath='{.items[0].metadata.name}')"
    kubectl create --raw "/api/v1/namespaces/$NS/pods/$P/eviction" -f - <<EOF
{"apiVersion":"policy/v1","kind":"Eviction","metadata":{"name":"$P","namespace":"$NS"}}
EOF
    kubectl -n "$NS" rollout status "deployment/$D" --timeout=300s
  done
  kubectl -n "$NS" get pods -o wide -l app.kubernetes.io/part-of=bookstore
}
can_bang
moc can-bang-xong

sleep 15; moc ket-thuc
touch "$CSV.stop"; wait "$PROBE"
~/bookstore/tests/availability-report.sh "$CSV" | tee "$EV/b5-tom-tat.txt"
```

Ý chính cho postmortem:

1. Bản sao trên node kia (Deployment, ràng buộc phân bố của `web` và anti-affinity của `api` từ Lab 7a);
   drain đi qua Eviction API nên PDB `maxUnavailable: 1` (Lab 3c, 4a) chỉ cho trục xuất **từng Pod
   một** — Pod thứ hai của cùng Deployment phải chờ bản thay thế của Pod thứ nhất Ready; tắt êm — Pod bị
   evict vẫn đi luồng kết thúc của bài 47, `preStop` của nginx từ Lab 3a giữ nginx phục vụ trong lúc
   control plane gỡ Pod khỏi EndpointSlice (bài 143, bước 3).
2. Hai nguồn lỗi hay gặp. (a) `000` trên **cả hai** đường dẫn cùng lúc, khoảng vài giây, ngay sau
   `cordon`: Pod Traefik duy nhất nằm trên worker1 bị evict — đúng dự đoán ở B1. PDB của Bookstore
   không với tới tầng này; sửa ở hạ tầng nền (hai bản sao Traefik và PDB) — ngoài phạm vi snapshot,
   ghi nợ ở B8. (b) Một hai `502` chỉ trên `/api/books` đúng lúc Pod `api` bị evict: PostgREST dừng
   ngay khi nhận tín hiệu trong khi kube-proxy trên node của nginx chưa kịp gỡ đích — sửa trong repo
   Bookstore ở B6.
3. Pod DaemonSet không đi đâu được: DaemonSet controller đặt lại ngay trên chính node đó và nó dung thứ
   taint `unschedulable` (bài 139), nên drain không bao giờ xong nếu cố trục xuất chúng. Trong lúc
   máy tắt, Pod `node-agent` vẫn nằm trong API gắn với worker1 — DaemonSet dung thứ `unreachable` vô
   thời hạn. Sau reboot, kubelet thấy Pod vẫn được gán cho node, dựng lại sandbox và chạy lại
   container: cùng Pod, cùng UID, `restartCount` tăng.
4. `.spec.unschedulable` nằm trong Node object ở API server, không ở đĩa worker1 — reboot không đụng
   tới nó. Pod đã được lập lịch thì ở yên; chỉ Pod **mới** mới được đặt lên worker1. Evict một Pod trên
   node đông là cách tạo Pod mới mà vẫn đi qua PDB; `rollout restart` thay mọi Pod và tốn thời gian hơn.
5. Không chặn. Với một bản sao, `maxUnavailable: 1` cho `disruptionsAllowed` bằng 1 (bài 339, mục
   *Kiểm tra trạng thái của PDB*): Pod `web` duy nhất bị trục xuất và trang chủ mất cho tới khi Pod mới
   Ready. PDB theo `maxUnavailable` bảo vệ **tỉ lệ** bản sao, không bảo vệ bản sao cuối cùng. Tiền điều
   kiện "`web` đủ bản sao Ready và có Pod trên cả hai worker" chặn đúng tình huống này — đó là lý do nó
   đứng đầu runbook.

</details>

## B6. [BS-12-06] api tắt êm không làm rơi request

**Loại:** Tính năng. Ticket này cũng là phép đo `R4` cho `api` mà Lab 4a hoãn lại: thay toàn bộ Pod
`api` dưới probe. Nếu B5 đo được lỗi trên `/api/books` trùng lúc Pod `api` bị evict, đây là nơi sửa.

**Bối cảnh:** Lab 3a cho `nginx` một `preStop` giữ nó phục vụ thêm vài giây, vì việc gỡ Pod khỏi danh
sách đích của Service diễn ra **song song** với việc tắt Pod. `api` thì chưa có: image PostgREST không
có shell, nên không chạy được lệnh `sleep` trong container. Kubelet có một loại handler tự chờ mà
không cần gì trong image.

**Yêu cầu:**

1. Khi Pod `api` bị đánh dấu kết thúc, tiến trình PostgREST tiếp tục phục vụ thêm vài giây rồi mới nhận
   tín hiệu dừng — không dùng shell, không đổi image.
2. Ân hạn tắt của `api` vẫn đủ lớn để chứa khoảng chờ đó.
3. Thay đổi nằm trong repo; triển khai bằng `./scripts/deploy.sh`.
4. Thay toàn bộ Pod `api` trong lúc probe chạy: `/api/books` có 0 request lỗi.

**Ràng buộc:** không đổi image, không thêm container, không sửa cấu hình nginx. Manifest vẫn đạt Pod
Security `restricted`.

**Đọc lại:**

- Bài [42](../42-container-lifecycle-hooks-vi.md#container-hooks) — ba loại handler, đặc biệt đoạn
  *ai gọi hook và ai chạy handler* về `sleep`.
- Bài [47](../47-pod-lifecycle-vi.md#pod-termination-flow) — đồng hồ ân hạn và hook `preStop`.
- Bài [143](../143-api-eviction-vi.md#cách-hoạt-động-của-eviction-khởi-phát-qua-api-how-api-initiated-eviction-works)
  — bước 2 và bước 3 diễn ra cùng lúc.

**Bàn giao:** manifest `api` trong `k8s/app/api/`; `~/lab-evidence/12/b6-*`; trong postmortem của B5,
mục *Đã sửa*: lỗi loại (b) đã sửa ở đâu, và dòng tóm tắt `/api/books` của lần thay Pod `api` dưới
probe — bằng chứng `R4` cho `api` mà Lab 14 sẽ dẫn.

**Nghiệm thu:**

- `PASS:` container `postgrest` có handler chờ ở `preStop`; ân hạn lớn hơn thời gian chờ.
- `PASS:` thay đổi nằm trong repo.
- `PASS:` việc thay Pod **đã thật sự xảy ra**: lệnh thành công, không còn Pod `api` nào của trước khi
  thay, revision tăng.
- `PASS:` dưới probe: ít nhất 20 mẫu, 0 lỗi trên `/api/books`.

**Câu hỏi thiết kế** — trả lời trong postmortem của B5, mục *Đã sửa*:

1. Vì sao cách Lab 3a làm cho nginx (chạy lệnh trong container) không dùng được cho `api`, còn cách
   này thì được? Ai thực thi handler?
2. Thời gian chờ nên dài bao nhiêu, bị chặn trên bởi trường nào? Chờ lâu thì việc drain một node chậm
   đi ở đâu?

<details>
<summary>Gợi ý</summary>

- Trường cần tìm: `kubectl explain pod.spec.containers.lifecycle.preStop`.
- Thay toàn bộ Pod của một Deployment: `kubectl rollout restart`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã triển khai bằng deploy.sh</summary>

```bash
NS=bookstore; EV=~/lab-evidence/12
C='{.spec.template.spec.containers[?(@.name=="postgrest")]'
S="$(kubectl -n "$NS" get deployment api -o jsonpath="$C.lifecycle.preStop.sleep.seconds}")"
G="$(kubectl -n "$NS" get deployment api -o jsonpath='{.spec.template.spec.terminationGracePeriodSeconds}')"
[ "${S:-0}" -ge 1 ] && [ "${G:-30}" -gt "$S" ] && echo "PASS: preStop cho $S giay, an han $G giay"
grep -rq 'preStop' ~/bookstore/k8s/app/api/ && echo 'PASS: thay doi nam trong repo'

CSV="$EV/b6-probe.csv"
# Số request lỗi chỉ có nghĩa khi việc thay Pod đã thật sự xảy ra: lệnh thành công, mọi Pod api của
# trước khi thay đều đã đi, revision tăng. Lệnh hỏng mà hệ thống cũ vẫn trả 200 là FAIL.
uid_api() { kubectl -n "$NS" get pods -l app.kubernetes.io/component=api \
              -o jsonpath='{range .items[*]}{.metadata.uid}{"\n"}{end}' | sort; }
rev_api() { kubectl -n "$NS" get deployment api -o jsonpath='{.metadata.annotations.deployment\.kubernetes\.io/revision}'; }
U0="$(uid_api)"; R0="$(rev_api)"; R1=0; CHUNG=-1; OK=1
{ [ -n "$U0" ] && [ "${R0:-0}" -gt 0 ]; } || { echo 'STOP: khong doc duoc Pod hoac revision cua api'; OK=0; }
nohup ~/bookstore/tests/availability-probe.sh "$CSV" 0.5 > "$CSV.log" 2>&1 &
PROBE=$!; sleep 10
if [ "$OK" -eq 1 ] && kubectl -n "$NS" rollout restart deployment/api > "$EV/b6-restart.txt" 2>&1 \
   && kubectl -n "$NS" rollout status deployment/api --timeout=300s >> "$EV/b6-restart.txt" 2>&1; then
  # Pod cũ còn chạy preStop và ân hạn sau khi rollout status báo xong — chờ chúng đi hết.
  for _ in $(seq 1 60); do
    U1="$(uid_api)"
    [ -n "$U1" ] && CHUNG="$(comm -12 <(printf '%s\n' "$U0") <(printf '%s\n' "$U1") | grep -c . || true)"
    [ "$CHUNG" = 0 ] && break; sleep 2
  done
  R1="$(rev_api)"
else OK=0; echo "FAIL: rollout restart hoac rollout status cua api loi — xem $EV/b6-restart.txt"; fi
sleep 15; touch "$CSV.stop"; wait "$PROBE"   # probe luôn được dừng, kể cả khi thao tác hỏng
{ [ "$OK" -eq 1 ] && [ "$CHUNG" = 0 ] && [ "${R1:-0}" -gt "$R0" ]; } \
  && echo "PASS: moi Pod api da duoc thay — khong UID cu nao con lai, revision $R0 -> $R1" \
  || echo 'FAIL: chua chung minh duoc toan bo Pod api da duoc thay'
~/bookstore/tests/availability-report.sh "$CSV" | tee "$EV/b6-tom-tat.txt"
TL="$(grep '^duong_dan=/api/books ' "$EV/b6-tom-tat.txt")"
MAU="$(printf '%s\n' "$TL" | sed -n 's/.* mau=\([0-9]*\) .*/\1/p')"
LOI="$(printf '%s\n' "$TL" | sed -n 's/.* loi=\([0-9]*\) .*/\1/p')"
{ [ "$OK" -eq 1 ] && [ "$CHUNG" = 0 ] && [ "${MAU:-0}" -ge 20 ] && [ "$LOI" = 0 ]; } \
  && echo "PASS: thay toan bo Pod api duoi probe: $MAU mau, 0 loi tren /api/books"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Thêm vào container `postgrest` trong `k8s/app/api/deployment.yaml`:

```yaml
        lifecycle:
          preStop:
            sleep:
              seconds: 5
```

`terminationGracePeriodSeconds: 30` của `api` (Lab 3a) đã lớn hơn 5 — giữ nguyên. Handler `sleep` do
**kubelet** thực thi, không cần gì trong image; trong 5 giây đó PostgREST vẫn phục vụ request đang tới,
còn control plane gỡ Pod khỏi EndpointSlice và kube-proxy trên mọi node cập nhật quy tắc. Hết 5 giây,
runtime mới gửi tín hiệu dừng. Đây cùng một lý do với `preStop` của nginx ở Lab 3a.

```bash
cd ~/bookstore && ./scripts/deploy.sh
```

Ý chính cho câu hỏi thiết kế:

1. Handler `exec` chạy **trong** container, nên cần shell và lệnh `sleep` trong image; image
   PostgREST chỉ có một binary. Handler `sleep` do kubelet thực thi (bài 42), image không cần gì.
2. Đủ để control plane gỡ Pod khỏi EndpointSlice và kube-proxy trên mọi node cập nhật — phụ thuộc cấu
   hình và tải của cluster, thường vài giây. Chặn trên là `terminationGracePeriodSeconds`, vì đồng hồ
   ân hạn chạy **trước** khi hook được gọi (bài 47). Mỗi Pod `api` bị evict mất thêm đúng khoảng chờ
   đó, nên drain node chứa nó chậm thêm chừng ấy — rẻ so với request rơi.

</details>

## B7. [BS-12-07] Bảo trì lab-k8s-worker2: cửa sổ bảo trì cho db

**Loại:** Vận hành, kèm diễn tập — trên `lab-k8s-worker2`.

**Bối cảnh:** worker2 cũng cần vá kernel. Khác worker1, nó giữ `db-0` và PV local của DB. B1 đã dự
đoán: `db` không thể sang worker1, nên đây là một **cửa sổ bảo trì** có downtime DB. Việc của đội là
làm cửa sổ đó **ngắn nhất và có kiểm soát**: có bản backup mới trước khi bắt đầu, dời mọi thứ không
phải `db` ra khỏi node khi DB còn chạy, chỉ mở cửa sổ khi đã sẵn sàng, đo đúng downtime, và chứng
minh dữ liệu còn nguyên. Trong lúc máy tắt, đội quan sát control plane nhìn một node biến mất ra sao.

> **Trả nợ lab của Lab 3a — phase `Unknown`.** Đọc lại bài
> [47, mục *Phase của Pod*](../47-pod-lifecycle-vi.md#pod-phase), đặc biệt dòng `Unknown` và đoạn cuối
> của mục, **trước** khi làm ticket này.

**Yêu cầu:**

1. Trước cửa sổ: một bản backup thành công của `db` ngay trước khi bắt đầu; ghi trạng thái "trước"
   bằng công cụ bên dưới (gồm một đơn hàng đánh dấu ghi vào DB).
2. **Pha 1 — DB vẫn chạy:** đưa worker2 ra khỏi lập lịch và trục xuất mọi thứ **trừ** `db-0`, bằng
   cách để chính Kubernetes giữ `db-0` lại: một PDB thử nghiệm tên `db-thu` không cho phép gián đoạn
   tự nguyện nào với `db`. Chứng minh: drain bị chặn đúng ở `db-0` và nói rõ lý do; mọi Pod khác của
   node đã đi; `db-0` vẫn là Pod cũ, vẫn Ready.
3. **Pha 2 — mở cửa sổ:** gỡ PDB thử nghiệm, drain lại. Chứng minh: `db-0` bị thay bằng Pod mới nằm
   `Pending` vì không node nào nhận được; `api` chuyển NotReady nhưng **không** bị khởi động lại
   ([mục 3.7](DU-AN-BOOKSTORE.md#37-chịu-lỗi-và-vận-hành), dòng đầu); worker2 là Node rỗng.
4. Tắt máy worker2. Trong lúc máy tắt, chứng minh ba tín hiệu của control plane — condition, taint,
   Lease — và trạng thái của Pod `node-agent` còn gắn với worker2: phase, condition `Ready`, trạng thái
   container mà API đang báo.
5. Bật lại, chờ node Ready, đưa node trở lại lập lịch; `db-0` lên lại **trên worker2, cùng PV**, dữ liệu
   còn nguyên (số sách và đơn hàng đánh dấu); `api` Ready lại; cân bằng `web`, `api` như B5.
6. Ghi mốc bằng `moc`, đủ mười bốn mốc đúng thứ tự (danh sách bên dưới).
7. Postmortem `docs/postmortem/12-bao-tri-worker2.md`: downtime DB, đợt lỗi dài nhất của `/api/books`
   trong cửa sổ, số lỗi của trang chủ, các đoạn thời gian cấu thành downtime; trả lời câu hỏi phase
   `Unknown`.
8. ADR `docs/adr/NNNN-pdb-cho-db.md`; mục 3 của runbook: quy trình cửa sổ bảo trì node giữ dữ liệu.
9. ⭐ Diễn tập node mất liên lạc với một Pod **không** phải DaemonSet: sau pha 2, tạo Pod thử nghiệm
   `thu-mat-lien-lac` gắn thẳng vào worker2 (không qua scheduler, không dữ liệu); giữ máy tắt tới khi
   control plane đánh dấu xóa nó; chứng minh nó kẹt ở trạng thái đang kết thúc cho tới khi node quay
   lại.

**Ràng buộc:** không `--force`, không `--disable-eviction`; không xóa cưỡng bức `db-0`; không gắn taint
`out-of-service`; không sửa node affinity hay PV của `db` để "dời DB sang worker1"; PDB `db-thu` và Pod
⭐ là object thử nghiệm ở `~/lab-work/12/`, xóa khi xong.

**Đọc lại:**

- Bài [339](../339-configure-pdb-vi.md#suy-nghĩ-về-cách-ứng-dụng-của-bạn-phản-ứng-với-gián-đoạn-think-about-how-your-application-reacts-to-disruptions)
  — dòng *Ứng dụng Stateful một instance*, hai giải pháp khả dĩ.
- Bài [53](../53-disruptions-vi.md#how-to-perform-disruptive-actions-on-your-cluster) — ba lựa chọn khi
  phải làm hành động gây gián đoạn.
- Bài [23](../23-nodes-vi.md): [*Nhịp tim của Node*](../23-nodes-vi.md#node-heartbeats),
  [*Node controller*](../23-nodes-vi.md#node-controller).
- Bài [139](../139-taint-and-toleration-vi.md#eviction-dựa-trên-taint-taint-based-evictions) — taint
  `unreachable`, toleration tự thêm với `tolerationSeconds`, toleration của Pod DaemonSet.
- Bài [169](../169-node-shutdown-vi.md#non-graceful-node-shutdown) — Pod StatefulSet kẹt `Terminating`,
  taint `out-of-service` và hai điều kiện của nó.
- Bài [47](../47-pod-lifecycle-vi.md): [*Phase của Pod*](../47-pod-lifecycle-vi.md#pod-phase),
  [*Thu gom rác cho Pod*](../47-pod-lifecycle-vi.md#pod-garbage-collection).
- Bài [138](../138-assign-pod-node-vi.md), mục *nodeName* — cho ⭐.

**Bàn giao:** postmortem; ADR; mục 3 của runbook; trong `~/lab-evidence/12/`: `b7-truoc.txt`,
`b7-su-kien.txt`, `b7-probe.csv`, `b7-backup.txt`, `b7-drain-pha1.txt`, `b7-drain-pha2.txt`.

Hai công cụ ghi bằng chứng, chép nguyên văn — chạy **trước** khi bắt đầu, khi `db-0` đang Ready:

```bash
EV=~/lab-evidence/12; W=lab-k8s-worker2; NS=bookstore
MOC="$EV/b7-su-kien.txt"; : > "$MOC"
moc() { echo "$(date +%s%3N) $(date -Is) $1" | tee -a "$MOC"; }
SQL() { kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -qtAc "$1"; }
{ echo "boot_id=$(kubectl get node "$W" -o jsonpath='{.status.nodeInfo.bootID}')"
  echo "agent_uid=$(kubectl -n bookstore-system get pod -l app.kubernetes.io/component=agent \
       --field-selector "spec.nodeName=$W" -o jsonpath='{.items[0].metadata.uid}')"
  echo "db_uid=$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.metadata.uid}')"
  echo "pv=$(kubectl -n "$NS" get pvc data-db-0 -o jsonpath='{.spec.volumeName}')"
  echo "so_sach=$(SQL 'SELECT count(*) FROM api.books')"
  echo "don_danh_dau=$(SQL 'INSERT INTO api.orders (book_id, quantity) SELECT min(id), 1 FROM api.books RETURNING id' | head -n 1)"
} > "$EV/b7-truoc.txt"; cat "$EV/b7-truoc.txt"
```

Mười bốn mốc, đúng tên và đúng thứ tự: `bat-dau`, `backup-xong`, `cordon`, `drain-pha1-xong`,
`xoa-pdb`, `drain-pha2-xong`, `tat-may`, `bat-may`, `node-ready`, `uncordon`, `db-ready`, `api-ready`,
`can-bang-xong`, `ket-thuc`. Cửa sổ bảo trì tính từ `xoa-pdb`.

**Nghiệm thu:**

- `PASS:` backup trước cửa sổ in dòng `db-backup: OK`; file "trước" không có giá trị rỗng.
- `PASS:` pha 1 — drain bị PDB chặn ở `db-0`; trên worker2 chỉ còn Pod DaemonSet và `db-0`;
  `db-0` cùng UID, Ready; `web`, `api` đủ bản sao.
- `PASS:` pha 2 — `db-0` mới khác UID, `Pending` với `PodScheduled=False`/`Unschedulable`;
  `api` NotReady, phase `Running`, không khởi động lại sau ít nhất 60 giây; worker2 là Node rỗng.
- `PASS:` máy tắt — `Ready=Unknown`; taint `unreachable` (`NoExecute`) cùng taint `unschedulable`; Node
  Lease đứng yên; Pod `node-agent` trên worker2 còn trong API, phase chưa kết thúc, `Ready=False`, API
  vẫn báo container đang chạy.
- `PASS:` sau bảo trì — `bootID` đổi; `db-0` trên worker2, cùng PV, cùng số sách, đơn hàng đánh dấu còn;
  `node-agent` cùng UID; `web`, `api` có Pod trên cả hai worker.
- `PASS:` đủ mười bốn mốc đúng thứ tự; probe phủ trọn; postmortem chép đúng dòng `db_ngung_s=` mà
  script tính.
- `PASS:` có ADR và mục 3 của runbook.
- `PASS:` (⭐) Pod thử nghiệm chạy trên worker2 dù node đã cordon; mang toleration `unreachable` tự thêm;
  khi máy tắt đủ lâu nó có `deletionTimestamp` nhưng vẫn còn trong API; sau khi node quay lại thì biến
  mất.

**Câu hỏi thiết kế:**

Trong ADR `docs/adr/NNNN-pdb-cho-db.md`:

1. Có nên đưa PDB `maxUnavailable: 0` cho `db` vào repo không? Dùng hai giải pháp của bài 339 cho ứng
   dụng stateful một instance, câu hỏi "trên cluster này ai gây gián đoạn tự nguyện" của bài 53, và
   câu 4 của B1 về autoscaler. Nêu rõ điều kiện khiến bạn đổi quyết định.
2. Pha 1 dùng PDB để giữ `db-0` lại. Còn cách nào khác để drain "mọi thứ trừ `db`" mà không cần
   object thử nghiệm? Đánh đổi?
3. Hai lý do độc lập khiến `db-0` không thể chạy trên worker1. Nếu worker2 hỏng hẳn, không quay lại,
   khôi phục DB bằng con đường nào đã diễn tập trong dự án?
4. Nếu ai đó tắt thẳng worker2 không drain: `db-0` sẽ ra sao? Gắn taint `out-of-service` có cứu được
   `db` không — vì sao với PV local câu trả lời khác với ví dụ volume của bài 169?

Trong postmortem `docs/postmortem/12-bao-tri-worker2.md`:

5. Downtime DB đo được gồm những đoạn nào (drain, tắt, boot, node Ready, uncordon, lập lịch, khởi động
   PostgreSQL, `api` kết nối lại)? Đoạn nào đội rút ngắn được, đoạn nào không?
6. **(Trả nợ 3a.)** Trong lúc worker2 tắt: Node condition, phase của Pod `node-agent`, condition `Ready`
   của Pod, trạng thái container mà API báo — cái nào phản ánh sự thật, cái nào là dữ liệu cũ? "Không
   thể lấy được trạng thái của Pod" của dòng `Unknown` trong bài 47 hiện ra ở đâu trên cluster của bạn?

<details>
<summary>Gợi ý</summary>

1. PDB chỉ chọn Pod bằng label; dùng đúng cặp selector của hợp đồng cho `db`.
2. Drain có cờ `--timeout`: pha 1 cần drain **bỏ cuộc** sau một khoảng ngắn thay vì chờ mãi.
3. Một bản backup ngay lúc này: `kubectl -n bookstore create job <ten> --from=cronjob/db-backup`.
4. Đọc taint của một node:
   `kubectl get node <n> -o jsonpath='{range .spec.taints[*]}{.key}={.effect}{"\n"}{end}'`.
5. Pod `api` không có shell; trạng thái NotReady đọc từ condition, số lần khởi động lại từ
   `containerStatuses`.
6. ⭐ Pod gắn thẳng vào node bằng `spec.nodeName`; namespace `bookstore` là `restricted` nên Pod thử
   nghiệm phải đủ `securityContext` và có `resources` hợp LimitRange.

</details>

<details>
<summary>Script nghiệm thu — chạy theo bốn thời điểm</summary>

Phần A — sau pha 1 (drain đã bỏ cuộc), trước khi gỡ PDB:

```bash
NS=bookstore; EV=~/lab-evidence/12; W2=lab-k8s-worker2
grep -q 'db-backup: OK' "$EV/b7-backup.txt" && [ "$(grep -c '=$' "$EV/b7-truoc.txt")" -eq 0 ] \
  && echo 'PASS: co backup moi va du trang thai truoc cua so'
grep -qi "disruption budget" "$EV/b7-drain-pha1.txt" && grep -q 'db-0' "$EV/b7-drain-pha1.txt" \
  && echo 'PASS: drain pha 1 bi PDB chan o db-0'
[ "$(kubectl -n "$NS" get pdb db-thu -o jsonpath='{.status.disruptionsAllowed}')" = '0' ] && echo 'PASS: db-thu khong cho gian doan nao'
CON="$(kubectl get pods -A --field-selector "spec.nodeName=$W2" \
  -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}={.metadata.ownerReferences[0].kind}{"\n"}{end}' \
  | grep -v '=DaemonSet$')"
[ "$CON" = 'bookstore/db-0=StatefulSet' ] && echo 'PASS: tren worker2 chi con Pod DaemonSet va db-0'
U0="$(sed -n 's/^db_uid=//p' "$EV/b7-truoc.txt")"
{ [ "$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.metadata.uid}')" = "$U0" ] \
  && [ "$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = 'True' ]; } \
  && echo 'PASS: db-0 van la Pod cu va Ready — DB chua gian doan'
for D in web api; do
  [ "$(kubectl -n "$NS" get deployment "$D" -o jsonpath='{.status.readyReplicas}')" = \
    "$(kubectl -n "$NS" get deployment "$D" -o jsonpath='{.spec.replicas}')" ] && echo "PASS: $D du ban sao Ready"
done
kubectl -n "$NS" get pod -l app.kubernetes.io/component=api \
  -o jsonpath='{range .items[*]}{.metadata.name}={.status.containerStatuses[0].restartCount}{"\n"}{end}' \
  > "$EV/b7-api-sau-pha1.txt"
```

Phần B — sau pha 2, máy còn bật:

```bash
NS=bookstore; EV=~/lab-evidence/12; W2=lab-k8s-worker2; SEL='app.kubernetes.io/part-of=bookstore'
! kubectl -n "$NS" get pdb db-thu >/dev/null 2>&1 && grep -q "node/$W2 drained" "$EV/b7-drain-pha2.txt" \
  && echo 'PASS: da go PDB thu nghiem, drain pha 2 hoan tat'
U0="$(sed -n 's/^db_uid=//p' "$EV/b7-truoc.txt")"; U1="$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.metadata.uid}')"
{ [ -n "$U1" ] && [ "$U1" != "$U0" ] && [ "$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.phase}')" = 'Pending' ] \
  && [ "$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ]; } \
  && echo 'PASS: db-0 moi Pending, Unschedulable'
kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].message}{"\n"}' | tee "$EV/b7-db-pending.txt"
kubectl -n "$NS" wait --for=condition=Ready=false pod -l "$SEL,app.kubernetes.io/component=api" --timeout=180s \
  && echo 'PASS: api NotReady khi mat DB'
sleep 60
kubectl -n "$NS" get pod -l app.kubernetes.io/component=api \
  -o jsonpath='{range .items[*]}{.metadata.name}={.status.containerStatuses[0].restartCount}{"\n"}{end}' > "$EV/b7-api-pha2.txt"
diff -q "$EV/b7-api-sau-pha1.txt" "$EV/b7-api-pha2.txt" >/dev/null \
  && [ "$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=api -o jsonpath='{.items[*].status.phase}' | tr ' ' '\n' | sort -u)" = 'Running' ] \
  && echo 'PASS: api van Running, khong bi khoi dong lai sau 60 giay'
KHAC="$(kubectl get pods -A --field-selector "spec.nodeName=$W2" \
  -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].kind}{"\n"}{end}' | grep -vc '^DaemonSet$')"
[ "$KHAC" -eq 0 ] && echo 'PASS: worker2 la Node rong'
```

Nếu làm ⭐, sau Phần B tạo Pod thử nghiệm rồi chạy:

```bash
NS=bookstore; W2=lab-k8s-worker2
kubectl -n "$NS" wait --for=condition=Ready pod/thu-mat-lien-lac --timeout=120s
[ "$(kubectl -n "$NS" get pod thu-mat-lien-lac -o jsonpath='{.spec.nodeName}')" = "$W2" ] \
  && echo 'PASS: Pod gan thang vao worker2 van chay du node da cordon'
kubectl -n "$NS" get pod thu-mat-lien-lac \
  -o jsonpath='{.spec.tolerations[?(@.key=="node.kubernetes.io/unreachable")].tolerationSeconds}{"\n"}' | tee ~/lab-evidence/12/b7-sao-toleration.txt
grep -q '^[0-9]' ~/lab-evidence/12/b7-sao-toleration.txt && echo 'PASS: toleration unreachable co tolerationSeconds do he thong tu them'
```

Phần C — khi worker2 đã *Powered off*:

```bash
NS=bookstore; EV=~/lab-evidence/12; W2=lab-k8s-worker2
for i in $(seq 1 120); do
  ST="$(kubectl get node "$W2" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
  [ "$ST" = 'Unknown' ] && break; sleep 5
done
[ "$ST" = 'Unknown' ] && echo 'PASS: worker2 Ready=Unknown'
sleep 20
kubectl get node "$W2" -o jsonpath='{range .spec.taints[*]}{.key}={.effect}{"\n"}{end}' | tee "$EV/b7-taint-khi-tat.txt"
grep -qx 'node.kubernetes.io/unreachable=NoExecute' "$EV/b7-taint-khi-tat.txt" \
  && grep -q '^node.kubernetes.io/unschedulable=' "$EV/b7-taint-khi-tat.txt" \
  && echo 'PASS: taint unreachable NoExecute, taint cordon van con'
L1="$(kubectl -n kube-node-lease get lease "$W2" -o jsonpath='{.spec.renewTime}')"; sleep 30
L2="$(kubectl -n kube-node-lease get lease "$W2" -o jsonpath='{.spec.renewTime}')"
[ -n "$L1" ] && [ "$L1" = "$L2" ] && echo "PASS: Node Lease dung yen tai $L1"

AG="$(kubectl -n bookstore-system get pod -l app.kubernetes.io/component=agent --field-selector "spec.nodeName=$W2" -o jsonpath='{.items[0].metadata.name}')"
for i in $(seq 1 24); do
  PR="$(kubectl -n bookstore-system get pod "$AG" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
  [ "$PR" = 'False' ] && break; sleep 5
done
CS='{.status.containerStatuses[?(@.name=="agent")]'
kubectl -n bookstore-system get pod "$AG" -o jsonpath="pod={.metadata.name} phase={.status.phase} pod_ready={.status.conditions[?(@.type==\"Ready\")].status} container_ready=$CS.ready} container_running_tu=$CS.state.running.startedAt}{\"\n\"}" \
  | tee "$EV/b7-agent-khi-tat.txt"
kubectl -n bookstore-system get pod "$AG" -o wide | tee -a "$EV/b7-agent-khi-tat.txt"
grep -qE 'phase=(Running|Unknown) ' "$EV/b7-agent-khi-tat.txt" && echo 'PASS: Pod node-agent con trong API, phase chua ket thuc'
grep -q 'pod_ready=False' "$EV/b7-agent-khi-tat.txt" && echo 'PASS: condition Ready cua Pod da thanh False'
grep -q 'container_running_tu=20' "$EV/b7-agent-khi-tat.txt" && echo 'PASS: API van bao container dang chay — du lieu cu, khong ai cap nhat duoc'
[ "$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.phase}')" = 'Pending' ] && echo 'PASS: db-0 van Pending'
[ "$(tail -n 40 "$EV/b7-probe.csv" | awk -F, '$2 == "/" && $3 != "200"' | wc -l)" -eq 0 ] \
  && echo 'PASS: trang chu van phuc vu tu worker1'
```

Nếu làm ⭐, giữ máy tắt và chạy tới khi PASS (thời điểm phụ thuộc `tolerationSeconds` vừa đọc):

```bash
NS=bookstore
for i in $(seq 1 120); do
  DT="$(kubectl -n "$NS" get pod thu-mat-lien-lac -o jsonpath='{.metadata.deletionTimestamp}' 2>/dev/null)"
  [ -n "$DT" ] && break; sleep 10
done
kubectl -n "$NS" get pod thu-mat-lien-lac -o wide | tee ~/lab-evidence/12/b7-sao-ket.txt
[ -n "$DT" ] && kubectl -n "$NS" get pod thu-mat-lien-lac >/dev/null 2>&1 \
  && echo "PASS: Pod da bi danh dau xoa luc $DT nhung van ket trong API vi kubelet vang mat"
```

Phần D — sau mốc `ket-thuc`, đã dừng probe và viết xong postmortem:

```bash
NS=bookstore; EV=~/lab-evidence/12; W2=lab-k8s-worker2; MOC="$EV/b7-su-kien.txt"
PM=~/bookstore/docs/postmortem/12-bao-tri-worker2.md; R=~/bookstore/tests/availability-report.sh
tm() { awk -v m="$1" '$3 == m {print $1; exit}' "$MOC"; }
SQL() { kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -qtAc "$1"; }
v() { sed -n "s/^$1=//p" "$EV/b7-truoc.txt"; }

T=0; BAD=0
for M in bat-dau backup-xong cordon drain-pha1-xong xoa-pdb drain-pha2-xong tat-may bat-may \
         node-ready uncordon db-ready api-ready can-bang-xong ket-thuc; do
  X="$(tm "$M")"
  if [ -z "$X" ] || [ "$X" -lt "$T" ]; then echo "FAIL: moc $M thieu hoac sai thu tu"; BAD=1; else T="$X"; fi
done
[ "$BAD" -eq 0 ] && echo 'PASS: du muoi bon moc theo dung thu tu'

[ "$(kubectl get node "$W2" -o jsonpath='{.status.nodeInfo.bootID}')" != "$(v boot_id)" ] && echo 'PASS: worker2 da khoi dong lai that'
{ [ "$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.spec.nodeName}')" = "$W2" ] \
  && [ "$(kubectl -n "$NS" get pvc data-db-0 -o jsonpath='{.spec.volumeName}')" = "$(v pv)" ]; } \
  && echo 'PASS: db-0 tro lai worker2, cung PV'
{ [ "$(SQL 'SELECT count(*) FROM api.books')" = "$(v so_sach)" ] \
  && [ "$(SQL "SELECT count(*) FROM api.orders WHERE id = $(v don_danh_dau)")" = '1' ]; } \
  && echo 'PASS: du lieu con nguyen — so sach va don hang danh dau'
[ "$(kubectl -n bookstore-system get pod -l app.kubernetes.io/component=agent --field-selector "spec.nodeName=$W2" -o jsonpath='{.items[0].metadata.uid}')" = "$(v agent_uid)" ] \
  && echo 'PASS: node-agent tren worker2 van la Pod cu'
for D in web api; do
  NN="$(kubectl -n "$NS" get pod -l "app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=$D" \
    --field-selector=status.phase=Running -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | grep -c .)"
  [ "$NN" -ge 2 ] && echo "PASS: $D lai co Pod tren $NN node"
done
! kubectl -n "$NS" get pod thu-mat-lien-lac >/dev/null 2>&1 && echo 'PASS: khong con Pod thu nghiem'

F1="$(awk -F, 'NR == 2 {print $1}' "$EV/b7-probe.csv")"; FL="$(tail -n 1 "$EV/b7-probe.csv" | cut -d, -f1)"
[ "$F1" -le "$(tm cordon)" ] && [ "$FL" -ge "$(tm can-bang-xong)" ] && echo 'PASS: probe phu tron'
"$R" "$EV/b7-probe.csv" > "$EV/b7-tom-tat.txt"
"$R" "$EV/b7-probe.csv" "$(tm cordon)" "$(tm xoa-pdb)" > "$EV/b7-tom-tat-pha1.txt"
"$R" "$EV/b7-probe.csv" "$(tm xoa-pdb)" "$(tm ket-thuc)" > "$EV/b7-tom-tat-cua-so.txt"
{ echo "db_ngung_s=$(( ($(tm db-ready) - $(tm xoa-pdb)) / 1000 ))"
  echo "cua_so_s=$(( ($(tm ket-thuc) - $(tm xoa-pdb)) / 1000 ))"
  grep '^duong_dan=/api/books ' "$EV/b7-tom-tat-cua-so.txt" | sed -n 's/.*dot_dai_nhat_s=\([0-9.]*\).*/api_dot_loi_dai_nhat_s=\1/p'
  grep '^duong_dan=/ ' "$EV/b7-tom-tat.txt" | sed -n 's/.* loi=\([0-9]*\) .*/trang_chu_loi=\1/p'
} | tee "$EV/b7-cua-so.txt"
cat "$EV/b7-tom-tat-pha1.txt"
grep -qF "$(grep '^db_ngung_s=' "$EV/b7-cua-so.txt")" "$PM" && echo 'PASS: postmortem chep dung downtime DB'
ls ~/bookstore/docs/adr/*-pdb-cho-db.md >/dev/null 2>&1 && grep -q '^## 3\.' ~/bookstore/docs/runbook-bao-tri-node.md \
  && echo 'PASS: co ADR va muc 3 cua runbook'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Hai object thử nghiệm, mỗi cái một file:

```yaml
# ~/lab-work/12/pdb-db-thu.yaml — giữ db-0 lại trong pha 1
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: db-thu
  namespace: bookstore
spec:
  maxUnavailable: 0
  selector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: db
```

```yaml
# ~/lab-work/12/thu-mat-lien-lac.yaml — ⭐, không dữ liệu, gắn thẳng vào worker2
apiVersion: v1
kind: Pod
metadata:
  name: thu-mat-lien-lac
  namespace: bookstore
spec:
  nodeName: lab-k8s-worker2
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: ngu
    image: busybox:1.37
    command: ["sh", "-c", "trap 'exit 0' TERM; while true; do sleep 5 & wait $!; done"]
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 50m, memory: 32Mi}
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

Quy trình, trong một phiên shell sau khối công cụ ở *Bàn giao*; hàm `can_bang` là hàm đã định nghĩa
trong lời giải của B5:

```bash
NS=bookstore; W2=lab-k8s-worker2; CSV="$EV/b7-probe.csv"
nohup ~/bookstore/tests/availability-probe.sh "$CSV" 0.5 > "$CSV.log" 2>&1 &
PROBE=$!; sleep 15; moc bat-dau

kubectl -n "$NS" create job bt12-truoc-bao-tri --from=cronjob/db-backup
kubectl -n "$NS" wait --for=condition=Complete job/bt12-truoc-bao-tri --timeout=300s
kubectl -n "$NS" logs job/bt12-truoc-bao-tri > "$EV/b7-backup.txt"; cat "$EV/b7-backup.txt"
moc backup-xong

# Pha 1 — DB còn chạy
kubectl apply -f ~/lab-work/12/pdb-db-thu.yaml
kubectl cordon "$W2"; moc cordon
kubectl drain "$W2" --ignore-daemonsets --delete-emptydir-data --timeout=90s > "$EV/b7-drain-pha1.txt" 2>&1
echo "drain pha 1 exit=$? (mong doi khac 0)"; moc drain-pha1-xong
# -> Phần A của script nghiệm thu; thông báo mở cửa sổ bảo trì

# Pha 2 — mở cửa sổ
kubectl -n "$NS" delete pdb db-thu; moc xoa-pdb
kubectl drain "$W2" --ignore-daemonsets --delete-emptydir-data --timeout=300s > "$EV/b7-drain-pha2.txt" 2>&1
echo "drain pha 2 exit=$?"; moc drain-pha2-xong
# -> Phần B. Chỉ khi làm ⭐: tạo Pod thử nghiệm rồi chạy khối ⭐ ngay sau Phần B
kubectl apply -f ~/lab-work/12/thu-mat-lien-lac.yaml

ssh "$W2" 'sudo shutdown -h now' || true; moc tat-may
```

Chạy trên **máy host**, PowerShell:

```powershell
$w2 = $vmx | Where-Object { $_ -like '*worker2*' }
if ((& $vmrun -T ws list) -notcontains $w2) { "PASS: worker2 da Powered off" } else { "CHUA TAT: doi roi chay lai" }
# ... chạy Phần C trên master (và khối ⭐ nếu làm), rồi:
& $vmrun -T ws start $w2
```

Quay lại master:

```bash
moc bat-may
kubectl wait --for=condition=Ready "node/$W2" --timeout=600s; moc node-ready
kubectl uncordon "$W2"; moc uncordon
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=600s; moc db-ready
kubectl -n "$NS" wait --for=condition=Ready pod -l app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=api \
  --timeout=300s; moc api-ready
kubectl -n "$NS" delete pod thu-mat-lien-lac --ignore-not-found --wait=true
can_bang
moc can-bang-xong
sleep 15; moc ket-thuc
touch "$CSV.stop"; wait "$PROBE"
```

Ý chính cho ADR:

1. Bài 339 cho hai giải pháp: không PDB và chấp nhận downtime thỉnh thoảng; hoặc PDB `maxUnavailable: 0`
   kèm thỏa thuận "hỏi chủ ứng dụng trước", xóa PDB khi sẵn sàng, tạo lại sau. Bài 53: trên một
   cluster cơ bản không có gián đoạn tự nguyện tự động — ở đây người duy nhất drain node là đội vận
   hành, theo runbook. Quyết định tham khảo: **chưa** đưa PDB `db` vào repo; runbook mục 3 là "thỏa
   thuận ngoài Kubernetes". Đổi quyết định ngay khi cluster có tác nhân drain tự động — autoscaler hợp
   nhất node (bài 171), nâng cấp node tự động: khi đó PDB là thứ duy nhất buộc tác nhân đó dừng lại
   trước `db-0`. Cái giá khi có PDB: drain mặc định chờ **vô hạn**, và mỗi lần bảo trì worker2 phải
   nhớ xóa rồi tạo lại PDB.
2. Drain có cờ lọc Pod theo label: `--pod-selector='app.kubernetes.io/component!=db'` trục xuất mọi
   thứ trừ `db-0` mà không cần object nào. Đánh đổi: không có gì **chặn** người quên cờ đó; PDB thì
   chặn cả người quên lẫn tác nhân tự động.
3. PV local của `data-db-0` mang node affinity tới worker2 — dữ liệu nằm trên đĩa worker2; và node
   affinity của chính `db` (Lab 7a) chỉ chấp nhận node mang label dữ liệu của dự án. Worker2 chết hẳn:
   dựng DB mới từ bản dump gần nhất, đúng quy trình đã diễn tập ở Lab 6b — vì vậy bước backup đứng đầu
   cửa sổ.
4. Không drain thì kubelet không chấm dứt gì (B3); `db-0` vẫn "chạy" trong API tới khi node controller
   gắn taint `unreachable`, hết `tolerationSeconds` thì `db-0` bị đánh dấu xóa và kẹt `Terminating` —
   StatefulSet không tạo được `db-0` mới vì tên còn bị giữ (bài 169). Taint `out-of-service` xóa cưỡng
   bức được Pod, nhưng `db-0` mới vẫn chỉ chạy được trên worker2 vì dữ liệu ở đó; ví dụ của bài 169
   nói về volume **gắn sang node khác được**, PV local thì không. Với `db`, taint đó không cứu được gì
   mà còn mở đường cho hai PostgreSQL cùng một thư mục dữ liệu nếu node chỉ đang khởi động lại.

Ý chính cho postmortem:

5. Downtime DB = drain pha 2 (PostgreSQL tắt nhanh theo `STOPSIGNAL`) + thời gian máy tắt + boot +
   node Ready + uncordon + lập lịch + PostgreSQL khởi động. `api` còn chậm thêm một đoạn vì PostgREST
   thử kết nối lại với backoff (theo tài liệu PostgREST, tối đa 32 giây). Rút được: thời gian làm việc
   trên máy, thao tác uncordon ngay khi node Ready. Không rút được: boot VM, khởi động PostgreSQL. Pha 1
   không tính vào downtime — đó là lợi ích của việc giữ `db-0` lại.
6. Node condition `Ready=Unknown` là thứ duy nhất phản ánh sự thật: node controller **không biết**.
   Phase của Pod `node-agent` vẫn là giá trị kubelet báo lần cuối, và API vẫn báo container đang chạy
   dù VM đã tắt — dữ liệu cũ, vì chỉ kubelet trên node mới cập nhật được chúng. Node controller chỉ hạ
   condition `Ready` của Pod. Câu "không thể lấy được trạng thái của Pod … do lỗi giao tiếp với node"
   của bài 47 hiện ra đúng như vậy: trạng thái không lấy được, nên API giữ giá trị cũ và đánh dấu node
   `Unknown`. Bài 47 cũng nói nếu node mất hẳn thì Pod trên đó bị đặt phase `Failed` — việc của PodGC
   khi Node object không còn, không xảy ra ở đây vì worker2 quay lại.

⭐: `nodeName` bỏ qua scheduler, nên cordon — vốn chỉ là tín hiệu cho scheduler — không chặn được Pod
đó. Pod không có toleration `unreachable` riêng nên mang toleration tự thêm với `tolerationSeconds`
mặc định (bài 139); hết khoảng đó nó bị đánh dấu xóa, nhưng không ai trên node xác nhận việc dừng nên
nó kẹt trong API — phiên bản không dữ liệu của chính kịch bản bài 169 mô tả cho StatefulSet.

</details>

## B8. [BS-12-08] Sprint review: runbook bảo trì và sổ nợ

**Loại:** Điều tra — sản phẩm là tài liệu.

**Bối cảnh:** hai lần bảo trì đã cho đội số đo thật. Trước khi đóng sprint, runbook phải đủ để một
người khác trong đội bảo trì bất kỳ worker nào mà không cần đọc lab này, và sổ nợ phải ghi đúng những
gì yêu cầu `R17` **chưa** đạt.

**Yêu cầu:**

1. `docs/runbook-bao-tri-node.md` hoàn chỉnh với các mục: `## 1.` hiện trạng và rủi ro (B1),
   `## 2.` bảo trì worker không giữ dữ liệu (B5), `## 3.` cửa sổ bảo trì node giữ dữ liệu (B7),
   `## 4.` node tắt ngoài kế hoạch — làm gì và **không** làm gì, `## 5.` ngoài phạm vi. Mỗi bước có
   điều kiện để đi tiếp, như một dòng `PASS:`.
2. Mục 5 ghi rõ những việc vận hành node mà sprint này **không** làm: kiểm và gia hạn certificate
   (nợ lab #7, bài 156 chỉ là trang trỏ hướng), nâng cấp node, rút node khỏi cluster.
3. `docs/tech-debt.md`: hai nợ mới — đường vào chưa HA (một IP node trong file hosts, ingress controller
   một bản sao, không PDB); DB một instance trên PV local, bảo trì node của nó là downtime. Mỗi nợ có
   bằng chứng trong `~/lab-evidence/12/`.
4. Một đoạn đánh giá `R17` trong runbook, theo đúng hai vế của yêu cầu ở
   [mục 2.2 của file dự án](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới): worker1 — số lỗi trên đường
   đo; worker2 — downtime DB đo được và dữ liệu giữ nguyên. Ghi riêng những gì nằm **ngoài** `R17`
   (đường trình duyệt qua file hosts, bảo trì worker2 không downtime) và nợ tương ứng.

**Ràng buộc:** hai nợ mới mang mã `TD-18` và `TD-19`, cố định theo [mục 8 của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án). Không sửa hạ tầng nền để "trả" nợ trong sprint này.

**Đọc lại:** bài [156](../156-certificates-vi.md); [sổ nợ lab](README.md#5-sổ-nợ-lab), dòng #7;
[mục 8 của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án); bài
[169](../169-node-shutdown-vi.md#non-graceful-node-shutdown) cho mục 4 của runbook.

**Bàn giao:** runbook; `docs/tech-debt.md`.

**Nghiệm thu:**

- `PASS:` runbook có đủ năm mục; mục 4 nhắc taint `out-of-service`; mục 5 nhắc certificate và nợ #7.
- `PASS:` `docs/tech-debt.md` có nợ về đường vào và nợ về DB một instance.
- `PASS:` runbook có đoạn đánh giá `R17`.

**Câu hỏi thiết kế** — trả lời trong đoạn đánh giá `R17` của runbook:

1. Bài 53 nêu ba lựa chọn khi phải làm hành động gây gián đoạn trên cluster. Dự án đang dùng lựa chọn
   nào cho `web`, `api`, và lựa chọn nào cho `db`?
2. Nếu chỉ được đầu tư một việc để `R17` đạt cho cả `db`, đó là việc gì, và vì sao nó nằm ngoài những
   gì lộ trình đã dạy tới giai đoạn 12?

<details>
<summary>Gợi ý</summary>

- Đọc lại mục *Cách thực hiện các hành động gây gián đoạn trên Cluster* của bài
  [53](../53-disruptions-vi.md#how-to-perform-disruptive-actions-on-your-cluster).
- Mục 4 của runbook là chỗ trả lời câu "node tắt mà chưa drain thì làm gì" — đọc lại hai điều kiện
  của taint `out-of-service` ở bài 169.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
RB=~/bookstore/docs/runbook-bao-tri-node.md; TD=~/bookstore/docs/tech-debt.md
BAD=0
for H in 1 2 3 4 5; do grep -q "^## $H\." "$RB" || { echo "FAIL: runbook thieu muc $H"; BAD=1; }; done
[ "$BAD" -eq 0 ] && echo 'PASS: runbook du nam muc'
sed -n '/^## 4\./,/^## 5\./p' "$RB" | grep -q 'out-of-service' && echo 'PASS: muc 4 nhac taint out-of-service'
sed -n '/^## 5\./,$p' "$RB" | grep -qi 'certificate\|chứng chỉ' && sed -n '/^## 5\./,$p' "$RB" | grep -q '#7' \
  && echo 'PASS: muc 5 ghi certificate la no lab #7'
grep -qi 'traefik\|ingress' "$TD" && grep -qi 'hosts' "$TD" && echo 'PASS: tech-debt co no duong vao'
grep -qi 'db-0\|PV local\|một instance' "$TD" && echo 'PASS: tech-debt co no DB mot instance'
grep -q 'R17' "$RB" && echo 'PASS: runbook co danh gia R17'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Khung mục 3 và mục 4 của runbook:

```markdown
## 3. Cửa sổ bảo trì cho lab-k8s-worker2 (giữ db-0 và PV data-db-0)

Thông báo trước: downtime đo được ở lần diễn tập 2026-10-13 là <db_ngung_s> giây cho DB,
<api_dot_loi_dai_nhat_s> giây cho /api/books; trang chủ không gián đoạn.

1. Tiền điều kiện như mục 2. PASS: mọi node Ready, PDB web/api cho phép >= 1.
2. Backup: Job từ CronJob db-backup. PASS: log có `db-backup: OK`. Chép bản dump ra ngoài (TD-10).
3. Pha 1: tạo PDB giữ db (`maxUnavailable: 0`), cordon, drain có `--timeout`.
   PASS: drain dừng ở db-0 vì disruption budget; node chỉ còn DaemonSet và db-0; probe 0 lỗi.
4. Mở cửa sổ: xóa PDB giữ db, drain lại. PASS: db-0 mới Pending; api NotReady, không restart.
5. Tắt máy, làm việc, bật máy. PASS: node Ready (vẫn SchedulingDisabled).
6. Uncordon. PASS: db-0 Ready trên worker2, cùng PV; api Ready; số sách khớp.
7. Cân bằng web/api bằng Eviction API. PASS: mỗi Deployment có Pod trên hai worker.
8. Đóng cửa sổ, ghi postmortem.

## 4. Node tắt ngoài kế hoạch
- Không gắn `out-of-service` khi chưa xác minh máy đã tắt hẳn (bài 169).
- web/api: Pod trên node mất bị đánh dấu xóa sau tolerationSeconds; ReplicaSet tạo bản mới ở node còn lại.
- db-0 trên worker2: KHÔNG gắn `out-of-service` — dữ liệu chỉ có trên đĩa worker2, Pod mới không chạy
  được ở đâu khác. Chờ worker2 hoặc khôi phục từ dump (Lab 6b).
```

Hai dòng nợ:

```markdown
| TD-18 | Đường vào chưa HA: hosts trỏ một IP node, Traefik một bản sao, không PDB | 12 | chưa có lab trả — sửa ở hạ tầng nền khi dựng lại cluster | mở | ~/lab-evidence/12/b1-mot-ban-sao.txt, b5-tom-tat.txt |
| TD-19 | DB một instance trên PV local của worker2: bảo trì worker2 là downtime DB | 12 | chấp nhận có điều kiện: cửa sổ bảo trì theo runbook mục 3 | mở | ~/lab-evidence/12/b7-cua-so.txt |
```

Đánh giá `R17` theo hai vế của yêu cầu: **đạt** cho worker1 — trên đường đo (NodePort Traefik qua IP
master) không request lỗi nào do `web`, `api` sau B6; đợt `000` nếu có đều rơi vào lúc Pod Traefik bị
evict, đã phân loại trong postmortem B5; **đạt** cho worker2 — DB ngừng `<db_ngung_s>` giây trong cửa
sổ, số sách và đơn hàng đánh dấu còn nguyên (B7). **Ngoài** `R17`: đường vào chưa HA — Traefik một
bản sao, file hosts trỏ một worker — `TD-18`; bảo trì worker2 không downtime — quyết định kiến trúc
(một instance, PV local), `TD-19`.

Ý chính cho câu hỏi thiết kế:

1. `web`, `api`: lựa chọn thứ ba của bài 53 — ứng dụng chịu được gián đoạn cộng PDB, không downtime.
   `db`: lựa chọn thứ nhất — chấp nhận downtime, nhưng có kiểm soát bằng cửa sổ bảo trì. Lựa chọn thứ
   hai (chuyển sang một cluster bản sao hoàn chỉnh) dự án không có; bản dump chép ra ngoài (`TD-10`)
   chỉ là đường khôi phục, không phải failover.
2. DB nhân bản — nhiều instance đồng bộ dữ liệu, mỗi instance trên một node, có cơ chế bầu primary —
   hoặc storage không gắn chặt vào một node. Cả hai đòi kiến thức về vận hành cơ sở dữ liệu nhân bản
   và hạ tầng lưu trữ mà lộ trình chưa dạy; bài 339 xếp loại ứng dụng đó vào nhóm "stateful nhiều
   instance" với quorum, khác hẳn trường hợp một instance của Bookstore.

</details>

## B9. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 19, rồi trả cluster về đúng
`04-metrics-ready` — kể cả cấu hình node không đổi sau hai lần tắt–bật.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 12 — kiểm thử khói **tích lũy**, chép nguyên văn. Nó
   giữ nguyên mọi kiểm tra của bản Lab 11b (kể cả vòng chờ có giới hạn tới khi HPA đưa `web` lên
   `minReplicas`) và thêm các hợp đồng vận hành của sprint này:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 12: hệ thống sprint 1–18 cộng hợp đồng vận hành của sprint 19.
# Kế thừa nguyên bản Lab 11b, thêm: mọi node Ready và nhận lập lịch, không còn taint bảo trì; api trải
# trên hai node (chỉ INFO — anti-affinity mềm); PDB api còn chỗ cho một gián đoạn; HTTPS qua NodePort
# sống trên mọi node.
# Tên ConfigMap đọc từ spec của workload (configMapGenerator gắn hậu tố hash từ 5a). Số bản sao web do
# HPA quyết định. Không in mật khẩu.
set -u
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd "$(dirname "$0")/.."
NS=bookstore
SYS=bookstore-system
TNS=bookstore-staging
SEL='app.kubernetes.io/part-of=bookstore'
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
lbl() { kubectl get "$1" -o jsonpath="{.metadata.labels.$2}" 2>/dev/null; }
can() { kubectl auth can-i "$@" -n "$NS" 2>/dev/null; }
psa() { kubectl get namespace "$1" -o jsonpath="{.metadata.labels.pod-security\.kubernetes\.io/$2}" 2>/dev/null; }
pst() { kubectl -n "$1" exec "$2" -c "$3" -- sh -c "awk '/^$4:/{print \$2}' /proc/1/status" 2>/dev/null; }
dry_priv() {
  printf '%s\n' 'apiVersion: v1' 'kind: Pod' 'metadata: {name: smoke-dac-quyen}' 'spec:' \
    '  containers: [{name: c, image: busybox:1.37, securityContext: {privileged: true}}]' \
    | kubectl -n "$1" create --dry-run=server -f - 2>&1
}
V="$(kubectl get --raw /version 2>/dev/null)"
PSA_VER="v$(echo "$V" | sed -n 's/.*"major": *"\([0-9]*\).*/\1/p').$(echo "$V" | sed -n 's/.*"minor": *"\([0-9]*\).*/\1/p')"
cond() { jp hpa/web "{.status.conditions[?(@.type==\"$1\")].status}"; }

# 0. HPA cần một chu kỳ để đưa web lên minReplicas và một cửa sổ đo để có số liệu — chờ có giới hạn.
MIN="$(jp hpa/web '{.spec.minReplicas}')"; MAX="$(jp hpa/web '{.spec.maxReplicas}')"
for i in $(seq 1 48); do
  R="$(jp deployment/web '{.spec.replicas}')"; RR="$(jp deployment/web '{.status.readyReplicas}')"
  U="$(jp hpa/web '{.status.currentMetrics[0].resource.current.averageUtilization}')"
  [ -n "$MIN" ] && [ "${R:-0}" -ge "$MIN" ] && [ "${RR:-0}" = "${R:-x}" ] && [ -n "$U" ] && break
  sleep 5
done

# 0b. Node sẵn sàng nhận Pod, không còn dấu bảo trì (12)
NODES="$(kubectl get nodes --no-headers | wc -l)"
RDY="$(kubectl get nodes -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}' | grep -c '^True$')"
[ "$NODES" -ge 3 ] && [ "$RDY" -eq "$NODES" ] && pass "ca $NODES node Ready" || fail "$RDY/$NODES node Ready"
UNS="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.unschedulable}{"\n"}{end}' | grep -c '^true$')"
[ "$UNS" -eq 0 ] && pass 'khong node nao bi cordon' || fail "$UNS node con bi cordon"
TNT="$(kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' \
  | grep -cE '^node\.kubernetes\.io/(unschedulable|unreachable|not-ready|out-of-service)$')"
[ "$TNT" -eq 0 ] && pass 'khong con taint bao tri hay mat lien lac' || fail "con $TNT taint bao tri"

# 1. Namespace, label dự án, Pod Security (9b, 11a)
for X in "$NS:restricted" "$TNS:restricted" "$SYS:privileged"; do
  N="${X%%:*}"; L="${X##*:}"
  { [ "$(lbl "namespace/$N" 'app\.kubernetes\.io/part-of')" = 'bookstore' ] && \
    [ "$(lbl "namespace/$N" 'pod-security\.kubernetes\.io/enforce')" = "$L" ]; } \
    && pass "namespace $N, PSS enforce=$L" || fail "namespace $N thieu, thieu label du an hoac PSS khac $L"
done
for N in "$NS" "$TNS"; do
  OK=1
  for M in enforce warn audit; do
    { [ "$(psa "$N" "$M")" = restricted ] && [ "$(psa "$N" "$M-version")" = "$PSA_VER" ]; } || OK=0
  done
  [ "$OK" -eq 1 ] && pass "$N restricted o ca ba che do, version $PSA_VER" || fail "$N sai nhan Pod Security"
  dry_priv "$N" | grep -q "violates PodSecurity \"restricted:$PSA_VER\"" \
    && pass "$N tu choi Pod dac quyen" || fail "$N khong tu choi Pod dac quyen"
done
# Lệnh dry-run phải thành công thì "không có cảnh báo" mới có nghĩa: Forbidden, timeout hay lỗi
# chứng chỉ là FAIL, không phải "sạch".
if PSW="$(kubectl label --dry-run=server --overwrite ns "$NS" pod-security.kubernetes.io/enforce-version=latest 2>&1)"; then
  echo "$PSW" | grep -q 'violate' && fail 'co Pod dang chay vi pham restricted' || pass 'moi Pod dang chay dat restricted'
else
  fail "khong hoi duoc Pod Security bang dry-run: $PSW"
fi

# 2. Object cấp cluster, label node (3a, 6a, 7a, 9a)
PRIO="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
for PC in bookstore-critical bookstore-batch; do
  [ "$(lbl "priorityclass/$PC" 'app\.kubernetes\.io/part-of')" = bookstore ] \
    && pass "co PriorityClass $PC" || fail "thieu PriorityClass $PC"
done
[ "$(kubectl get storageclass bookstore-retain -o jsonpath='{.reclaimPolicy}' 2>/dev/null)" = 'Retain' ] \
  && pass 'StorageClass bookstore-retain, Retain' || fail 'StorageClass bookstore-retain thieu hoac khong Retain'
[ "$(kubectl get nodes -l bookstore.example.com/data=true -o jsonpath='{.items[*].metadata.name}')" = lab-k8s-worker2 ] \
  && pass 'chi lab-k8s-worker2 mang label du lieu' || fail 'label node bookstore.example.com/data sai'
N_CL="$(kubectl get clusterrole,clusterrolebinding -l "$SEL" --no-headers 2>/dev/null | wc -l)"
[ "$N_CL" -eq 0 ] && pass 'du an khong co ClusterRole, ClusterRoleBinding' || fail "co $N_CL object RBAC cap cluster"

# 2b. HPA web (11b)
[ "$(jp hpa/web '{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ] && pass 'HPA web ton tai, mang label du an' || fail 'thieu HPA web'
[ "$(jp hpa/web '{.spec.scaleTargetRef.kind}/{.spec.scaleTargetRef.name} {.spec.minReplicas}-{.spec.maxReplicas} {.spec.metrics[0].resource.name}/{.spec.metrics[0].resource.target.type}')" = 'Deployment/web 2-5 cpu/Utilization' ] \
  && pass 'HPA web: Deployment/web, 2-5 ban sao, CPU Utilization' || fail 'HPA web sai dich, khoang hoac metric'
{ [ "$(cond AbleToScale)" = 'True' ] && [ "$(cond ScalingActive)" = 'True' ]; } \
  && pass 'HPA AbleToScale=True, ScalingActive=True' || fail "HPA AbleToScale=$(cond AbleToScale) ScalingActive=$(cond ScalingActive)"
[ -n "$(jp hpa/web '{.status.currentMetrics[0].resource.current.averageUtilization}')" ] \
  && pass 'HPA co so do CPU hien tai' || fail 'HPA chua co so do — kiem metrics-server va requests cua web'
LA="$(jp deployment/web '{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}')"
{ [ -n "$LA" ] && ! echo "$LA" | grep -q '"replicas"'; } \
  && pass 'manifest web khong khai replicas — HPA so huu truong nay' || fail 'manifest web con replicas'
[ "$(sl k get job,networkpolicy,pod -l app.kubernetes.io/component=loadtest --no-headers)" -eq 0 ] \
  && pass 'khong con object kiem thu tai' || fail 'con object loadtest tren cluster'

# 3. Cấu hình, bí mật, PDB, hạn mức, trạng thái backup (3b, 3c, 5b, 7b, 9a)
for P in web-config api-config db-init; do
  k get configmap -o name 2>/dev/null | grep -q "^configmap/$P" && pass "co ConfigMap $P" || fail "thieu ConfigMap $P"
done
WL="$(k get deployment,statefulset -o json 2>/dev/null)"
for B in web-config api-config db-init; do
  CM="$(echo "$WL" | grep -o "\"$B\(-[a-z0-9]*\)\?\"" | head -1 | tr -d '"')"
  [ -n "$CM" ] && k get configmap "$CM" >/dev/null 2>&1 && pass "ConfigMap $CM ton tai, duoc workload tham chieu" || fail "thieu ConfigMap $B duoc workload tham chieu"
done
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials du hai key' || fail 'Secret db-credentials thieu key'
[ "$(jp secret/bookstore-tls '{.type}')" = 'kubernetes.io/tls' ] && pass 'Secret bookstore-tls kieu TLS' || fail 'thieu Secret TLS bookstore-tls'
for R in pdb/web pdb/api resourcequota/bookstore-quota limitrange/bookstore-limits configmap/backup-status; do
  k get "$R" >/dev/null 2>&1 && pass "co $R" || fail "thieu $R"
done

# 4. Controller, vị trí, bảo mật (4a, 4b, 7a, 9b)
[ "$(jp statefulset/db '{.status.readyReplicas}')" = '1' ] && pass 'StatefulSet db 1/1 Ready' || fail 'StatefulSet db chua Ready'
{ [ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && [ "$(jp pod/db-0 '{.status.qosClass}')" = 'Guaranteed' ] && \
  [ -n "$PRIO" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$PRIO" ]; } \
  && pass 'db-0 tren lab-k8s-worker2, Guaranteed, bookstore-critical' || fail 'db-0 sai node, QoS hoac priority'
[ "$(jp pod/db-0 '{.metadata.ownerReferences[0].kind}')" = 'StatefulSet' ] && pass 'db-0 thuoc StatefulSet' || fail 'db-0 khong thuoc StatefulSet'
W="$(jp deployment/api '{.spec.replicas}')"; R="$(jp deployment/api '{.status.readyReplicas}')"
{ [ -n "$W" ] && [ "$W" -ge 2 ] && [ "${R:-0}" = "$W" ]; } && pass "Deployment api $R/$W Ready" || fail "Deployment api ${R:-0}/${W:-?} Ready"
# web: số bản sao do HPA quyết định (11b) — kiểm theo khoảng của HPA.
R="$(jp deployment/web '{.spec.replicas}')"; RR="$(jp deployment/web '{.status.readyReplicas}')"
{ [ -n "$MIN" ] && [ "${R:-0}" -ge "$MIN" ] && [ "${R:-0}" -le "$MAX" ] && [ "${RR:-0}" = "$R" ]; } \
  && pass "Deployment web $RR/$R Ready trong khoang HPA $MIN-$MAX" || fail "Deployment web ${RR:-0}/${R:-0} ngoai khoang HPA hoac chua Ready"
NW="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' 2>/dev/null | sort -u | wc -l)"
[ "$NW" -ge 2 ] && pass 'Pod web trai tren hai worker' || fail 'Pod web chi nam tren mot node'
[ "$(jp pdb/web '{.status.disruptionsAllowed}')" -ge 1 ] 2>/dev/null \
  && pass 'PDB web cho phep it nhat mot gian doan o so ban sao hien tai' || fail 'PDB web khong cho gian doan nao — drain se ket'
NA="$(k get pod -l "$SEL,app.kubernetes.io/component=api" --field-selector=status.phase=Running \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' 2>/dev/null | sort -u | wc -l)"
# api dùng anti-affinity mềm theo hợp đồng (7a): trải hai node không bảo đảm — chỉ báo INFO.
[ "$NA" -ge 2 ] && echo 'INFO: Pod api trai tren hai worker' \
  || echo 'INFO: Pod api don vao mot node — bao tri node do la mat api; can bang lai truoc khi drain'
[ "$(jp pdb/api '{.status.disruptionsAllowed}')" -ge 1 ] 2>/dev/null \
  && pass 'PDB api cho phep it nhat mot gian doan' || fail 'PDB api khong cho gian doan nao — drain se ket'
ORPHAN="$(kubectl get pods -A -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"
MISS="$(kubectl get pods -A -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container co requests va limits' || fail "container thieu tai nguyen: $(echo $MISS)"

# 5. Hardening (9b): db non-root, không capability, rootfs chỉ đọc
[ "$(pst "$NS" db-0 postgres Uid)" = 999 ] && pass 'db chay uid 999' || fail 'db khong chay uid 999'
C="$(pst "$NS" db-0 postgres CapEff)"
[ "$((0x${C:-1}))" -eq 0 ] && pass 'db khong giu capability nao' || fail 'db con capability'
NODROP="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.securityContext.capabilities.drop}{"\n"}{end}{end}' 2>/dev/null \
          | grep -v 'ALL' | sort -u)"
[ -z "$NODROP" ] && pass 'moi container cua du an drop ALL capability' || fail "container chua drop ALL: $(echo $NODROP)"
RO() { jp "deployment/$1" "{.spec.template.spec.containers[?(@.name==\"$2\")].securityContext.readOnlyRootFilesystem}"; }
{ [ "$(RO web nginx)" = true ] && [ "$(RO api postgrest)" = true ]; } \
  && pass 'nginx, postgrest khai rootfs chi doc' || fail 'nginx hoac postgrest chua rootfs chi doc'
SC='{.spec.jobTemplate.spec.template.spec.securityContext'
{ [ "$(jp cronjob/db-backup "$SC.runAsUser}")" = 999 ] && [ "$(jp cronjob/db-backup "$SC.runAsNonRoot}")" = true ]; } \
  && pass 'db-backup chay uid 999, kubelet kiem non-root' || fail 'db-backup chua non-root'
jp deployment/api '{range .spec.template.spec.securityContext.sysctls[*]}{.name}={.value}{" "}{end}' \
  | grep -q 'net.ipv4.tcp_keepalive_time=60' && pass 'api dat sysctl keepalive' || fail 'api thieu sysctl keepalive'

# 6. Dữ liệu bền (4b, 6a)
# backup-store (WaitForFirstConsumer) chỉ Bound sau lần chạy thử db-backup bên dưới (6a).
[ "$(jp pvc/data-db-0 '{.status.phase}')" = 'Bound' ] && pass 'PVC data-db-0 Bound' || fail 'PVC data-db-0 chua Bound'
[ "$(jp pvc/data-db-0 '{.spec.storageClassName}')" = 'bookstore-retain' ] && pass 'data-db-0 dung bookstore-retain' || fail 'data-db-0 sai StorageClass'
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach, can >= 8"
if k get job db-migrate >/dev/null 2>&1; then
  [ "$(jp job/db-migrate '{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ] \
    && pass 'Job db-migrate Complete' || fail 'Job db-migrate chua Complete'
else
  pass 'Job db-migrate da duoc TTL don, ket qua nam trong db'
fi

# 7. Service, DNS, đường đi của request (5a, 9b)
[ "$(jp service/db '{.spec.clusterIP}')" = 'None' ] && pass 'Service db headless' || fail 'Service db khong headless'
[ "$(jp service/web '{.spec.type}/{.spec.ports[0].port}')" = 'ClusterIP/80' ] && pass 'Service web ClusterIP:80' || fail 'Service web sai'
[ "$(jp service/api '{.spec.ports[0].port}')" = '3000' ] && pass 'Service api cong 3000' || fail 'Service api sai cong'
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
WNODE="$(jp "pod/$WEB" '{.spec.nodeName}')"
get() { k exec "$WEB" -c status-writer -- timeout 10 wget -q -T 5 -O- "$1" 2>/dev/null; }
T="$(get http://api:3000/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "web goi api bang ten: $T cuon sach" || fail "web goi api bang ten: $T cuon sach"
get http://127.0.0.1:8080/healthz >/dev/null && pass 'web /healthz' || fail 'web /healthz loi'
get http://127.0.0.1:8080/ | grep -q 'Bookstore' && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
get http://127.0.0.1:8080/status.json | grep -Eq "\"pod\" *: *\"$WEB\"" && pass 'status.json dung ten Pod' || fail 'status.json sai hoac thieu'
T="$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "web /api/books tra $T cuon sach" || fail "web /api/books tra $T cuon sach"
DB_IP="$(jp pod/db-0 '{.status.podIP}')"
k exec "$WEB" -c status-writer -- nslookup "db-0.db.$NS.svc.cluster.local" 2>/dev/null | grep -qwF "$DB_IP" \
  && pass 'ten db-0.db phan giai ve IP cua db-0' || fail 'ten db-0.db khong phan giai dung'
k exec "$WEB" -c nginx -- sh -c 'touch /thu-ghi' >/dev/null 2>&1 \
  && fail 'nginx ghi duoc root filesystem' || pass 'root filesystem cua nginx chi doc'

# 8. Ingress HTTPS qua Traefik (5b)
{ [ "$(jp ingress/bookstore '{.spec.rules[0].host}')" = 'bookstore.lab.local' ] && \
  [ "$(jp ingress/bookstore '{.spec.tls[0].secretName}')" = 'bookstore-tls' ]; } \
  && pass 'Ingress bookstore dung host va TLS' || fail 'Ingress bookstore sai host hoac TLS'
NP="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}' 2>/dev/null)"
IP="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
if [ -n "$NP" ] && curl -ks --max-time 5 --resolve "bookstore.lab.local:$NP:$IP" "https://bookstore.lab.local:$NP/" | grep -q 'Bookstore'; then
  pass "HTTPS qua Traefik (NodePort $NP) tra trang chu"
else
  fail 'khong vao duoc trang chu qua Ingress HTTPS'
fi
HP="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}' 2>/dev/null)"
[ -n "$HP" ] || HP="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.port==443)].nodePort}' 2>/dev/null)"
for NN in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  IPN="$(kubectl get node "$NN" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
  curl -sk --max-time 10 --resolve "bookstore.lab.local:${HP:-0}:$IPN" "https://bookstore.lab.local:${HP:-0}/" | grep -q 'Bookstore' \
    && pass "HTTPS qua NodePort $HP tren $NN tra trang chu" || fail "HTTPS qua NodePort tren $NN loi"
done
T="$(curl -sk --max-time 10 --resolve "bookstore.lab.local:${HP:-0}:$IPN" "https://bookstore.lab.local:${HP:-0}/api/books" | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "/api/books qua Ingress HTTPS tra $T cuon sach" || fail "/api/books qua Ingress HTTPS tra $T cuon sach"

# 9. Chính sách mạng (5b, 9a, 11a)
for N in "$NS" "$SYS"; do
  P="$(kubectl -n "$N" get networkpolicy -o jsonpath='{range .items[*]}{.spec.podSelector}|{.spec.policyTypes}{"\n"}{end}' 2>/dev/null | grep '^{}|')"
  { echo "$P" | grep -q 'Ingress' && echo "$P" | grep -q 'Egress'; } \
    && pass "moi Pod trong $N bi co lap hai chieu" || fail "namespace $N thieu policy co lap hai chieu"
done
# web không tới được API server. Đối chứng dương cùng đường đi: Pod busybox ở namespace thử
# bookstore-thu (không NetworkPolicy; chỉ xóa nếu smoke tự tạo) gọi được đúng URL đó bằng đúng lệnh đó.
# Phủ định chỉ tính khi wget hết giờ hoặc bị từ chối kết nối; lỗi khác là "không kết luận được".
WGET_API='timeout 10 wget -q -T 5 -O /dev/null --no-check-certificate https://kubernetes.default.svc/version'
PNS=bookstore-thu; MADE=0
if ! kubectl get namespace "$PNS" >/dev/null 2>&1; then
  kubectl create namespace "$PNS" >/dev/null 2>&1 \
    && kubectl label namespace "$PNS" app.kubernetes.io/part-of=bookstore >/dev/null 2>&1 && MADE=1
fi
CTRL="$(kubectl -n "$PNS" run "smoke-api-$(date +%s)" --rm -i --restart=Never --image=busybox:1.37 --command -- \
        sh -c "$WGET_API && echo TOI-DUOC || echo KHONG-TOI" 2>/dev/null)"
[ "$MADE" -eq 1 ] && kubectl delete namespace "$PNS" --ignore-not-found --wait=true --timeout=120s >/dev/null 2>&1
OUT="$(k exec "$WEB" -c status-writer -- sh -c "$WGET_API" 2>&1)"; RC=$?
if ! echo "$CTRL" | grep -q 'TOI-DUOC'; then fail 'doi chung: Pod ngoai NetworkPolicy khong goi duoc API server — khong ket luan duoc'
elif [ "$RC" -eq 0 ]; then fail 'web toi duoc API server'
elif echo "$OUT" | grep -Eq 'timed out|Connection refused|exit code 143'; then pass 'web bi NetworkPolicy chan toi API server'
else fail "web toi API server loi khac chan mang — khong ket luan duoc: $(echo $OUT)"; fi

# 10. CronJob db-backup và backup-status (4b, 6a, 7a, 9a)
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = Forbid ] && pass 'db-backup Forbid' || fail 'db-backup khong Forbid'
SDS="$(jp cronjob/db-backup '{.spec.startingDeadlineSeconds}')"
[ "${SDS:-0}" -ge 10 ] && pass "db-backup startingDeadlineSeconds=$SDS" || fail 'db-backup thieu startingDeadlineSeconds'
[ "$(jp cronjob/db-backup '{.spec.suspend}')" != true ] && pass 'db-backup khong bi tam ngung' || fail 'db-backup dang suspend'
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.priorityClassName}')" = bookstore-batch ] \
  && pass 'db-backup dung bookstore-batch' || fail 'db-backup khong dung bookstore-batch'
J="smoke-backup-$(date +%s)"
if k create job "$J" --from=cronjob/db-backup >/dev/null 2>&1 \
   && k wait --for=condition=Complete "job/$J" --timeout=300s >/dev/null 2>&1; then
  LINE="$(k logs "job/$J" 2>/dev/null | grep '^db-backup: OK' | tail -1)"
  F="$(echo "$LINE" | awk '{print $3}')"; B="$(echo "$LINE" | awk '{print $4}')"
  [ -n "$LINE" ] && pass "chay thu db-backup: $F ($B byte)" || fail 'chay thu db-backup khong in db-backup: OK'
  { [ -n "$F" ] && [ "$(jp configmap/backup-status '{.data.file}')" = "$F" ] && \
    [ "$(jp configmap/backup-status '{.data.bytes}')" = "$B" ]; } \
    && pass 'backup-status ghi dung lan chay vua roi' || fail 'backup-status khong khop lan chay'
else
  fail 'chay thu db-backup that bai'
fi
k delete job "$J" --ignore-not-found --wait=true >/dev/null 2>&1
# Hợp đồng ghi file backup (6a): đọc thẳng PVC, không tin log — tối đa 10 bản mỗi loại,
# không còn file tạm cũ hơn một giờ.
RD="smoke-doc-backup-$(date +%s)"
k apply -f - >/dev/null 2>&1 <<POD
apiVersion: v1
kind: Pod
metadata:
  name: $RD
spec:
  restartPolicy: Never
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    seccompProfile: {type: RuntimeDefault}
  containers:
  - name: doc
    image: busybox:1.37
    command: ["sh", "-c", "ls -1 /backup; echo ---; find /backup -maxdepth 1 -name '*.tmp' -mmin +60"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities: {drop: [ALL]}
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 50m, memory: 32Mi}
    volumeMounts:
    - {name: b, mountPath: /backup, readOnly: true}
  volumes:
  - name: b
    persistentVolumeClaim: {claimName: backup-store, readOnly: true}
POD
if k wait --for=jsonpath='{.status.phase}'=Succeeded "pod/$RD" --timeout=120s >/dev/null 2>&1; then
  OUT="$(k logs "$RD" 2>/dev/null)"
  NB="$(echo "$OUT" | sed '/^---$/,$d' | grep -c '^bookstore-.*\.sql$')"
  NO="$(echo "$OUT" | sed '/^---$/,$d' | grep -Ec '^dbbackup-.*\.(sql|dump)$')"
  NT="$(echo "$OUT" | sed '1,/^---$/d' | grep -c .)"
  { [ "$NB" -le 10 ] && [ "$NO" -le 10 ] && [ "$NT" -eq 0 ]; } \
    && pass "backup-store: $NB ban db-backup, $NO ban operator, khong file tam cu" \
    || fail "backup-store sai hop dong 6a ($NB ban db-backup, $NO ban operator, $NT file tam cu)"
else
  fail 'khong doc duoc noi dung backup-store'
fi
k delete pod "$RD" --ignore-not-found --wait=false >/dev/null 2>&1
[ "$(jp pvc/backup-store '{.status.phase}')" = 'Bound' ] && pass 'PVC backup-store Bound (sau lan chay thu)' || fail 'PVC backup-store chua Bound'

# 11. Danh tính và quyền (9a, 9b); ServiceAccount node-agent chuyển sang bookstore-system (11a)
BAD=0
for X in deployment/web:web deployment/api:api statefulset/db:db; do
  [ "$(jp "${X%%:*}" '{.spec.template.spec.serviceAccountName}')" = "${X##*:}" ] || { echo "FAIL: ${X%%:*} sai ServiceAccount"; BAD=1; }
done
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.serviceAccountName}')" = db-backup ] \
  || { echo 'FAIL: db-backup sai ServiceAccount'; BAD=1; }
for SA in default web api db db-migrate db-backup ci-deployer; do
  [ "$(jp "serviceaccount/$SA" '{.automountServiceAccountToken}')" = false ] \
    || { echo "FAIL: ServiceAccount $SA thieu hoac tu mount token"; BAD=1; }
done
{ [ "$(kubectl -n "$SYS" get daemonset node-agent -o jsonpath='{.spec.template.spec.serviceAccountName}' 2>/dev/null)" = node-agent ] && \
  [ "$(kubectl -n "$SYS" get serviceaccount node-agent -o jsonpath='{.automountServiceAccountToken}' 2>/dev/null)" = false ]; } \
  || { echo 'FAIL: node-agent khong dung ServiceAccount node-agent khong token o bookstore-system'; BAD=1; }
[ "$BAD" -eq 0 ] && pass 'moi thanh phan mot ServiceAccount, khong ai tu mount token' || FAIL=1
TOKP="$(kubectl get pods -A -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.volumes[*]}{.name}{"\n"}{end}{end}' 2>/dev/null | grep -c '^kube-api-access-')"
[ "$TOKP" -eq 0 ] && pass 'khong Pod nao cua du an mang token tu dong' || fail "$TOKP volume token tu dong"
B=system:serviceaccount:bookstore:db-backup
{ [ "$(can patch configmaps/backup-status --as="$B")" = yes ] && [ "$(can list configmaps --as="$B")" = no ] && \
  [ "$(can get secrets --as="$B")" = no ]; } && pass 'db-backup chi sua duoc backup-status' || fail 'quyen db-backup sai'
CI=system:serviceaccount:bookstore:ci-deployer
{ [ "$(can patch deployments --as="$CI")" = yes ] && [ "$(can get secrets --as="$CI")" = no ] && \
  [ "$(can create pods --subresource=exec --as="$CI")" = no ] && [ "$(can patch namespaces/bookstore --as="$CI")" = no ]; } \
  && pass 'ci-deployer sua Deployment, khong Secret, exec hay Namespace' || fail 'quyen ci-deployer sai'
{ [ "$(can list pods --as=dev --as-group=bookstore-devs)" = yes ] && [ "$(can get secrets --as=dev --as-group=bookstore-devs)" = no ] && \
  [ "$(can patch pods --subresource=ephemeralcontainers --as=dev --as-group=bookstore-devs)" = no ]; } \
  && pass 'bookstore-devs chi doc' || fail 'quyen bookstore-devs sai'
{ [ "$(can patch pods --subresource=ephemeralcontainers --as=truc --as-group=bookstore-oncall)" = yes ] && \
  [ "$(can get secrets --as=truc --as-group=bookstore-oncall)" = no ]; } \
  && pass 'bookstore-oncall go loi duoc, khong doc Secret' || fail 'quyen bookstore-oncall sai'

# 12. Tenant bookstore-staging (9b)
{ kubectl -n "$TNS" get resourcequota staging-quota >/dev/null 2>&1 && \
  kubectl -n "$TNS" get limitrange staging-limits >/dev/null 2>&1; } \
  && pass 'staging co quota va LimitRange' || fail 'staging thieu quota hoac LimitRange'
kubectl -n "$TNS" get networkpolicy -o jsonpath='{range .items[*]}{.spec.podSelector}|{.spec.policyTypes}{"\n"}{end}' 2>/dev/null \
  | grep -q '^{}|.*Ingress.*Egress' && pass 'staging co default deny hai chieu' || fail 'staging thieu default deny'
SW="$(kubectl -n "$TNS" get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running \
        -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
kubectl -n "$TNS" exec "$SW" -- wget -qO- -T 5 http://web/ 2>/dev/null | grep -q 'Bookstore staging' \
  && pass 'staging phuc vu trang qua Service cua chinh no' || fail 'staging khong phuc vu duoc trang'
kubectl -n "$TNS" exec "$SW" -- wget -qO- -T 3 http://api.bookstore.svc.cluster.local:3000/books >/dev/null 2>&1 \
  && fail 'staging goi duoc api cua bookstore' || pass 'staging bi chan khi goi bookstore'
DEV='--as=dev --as-group=bookstore-devs'
{ [ "$(kubectl auth can-i create deployments -n "$TNS" $DEV 2>/dev/null)" = yes ] && \
  [ "$(kubectl auth can-i get secrets -n "$TNS" $DEV 2>/dev/null)" = no ] && \
  [ "$(kubectl auth can-i patch "namespace/$TNS" $DEV 2>/dev/null)" = no ] && \
  [ "$(can create deployments $DEV)" = no ]; } \
  && pass 'bookstore-devs trien khai duoc o staging; khong Secret, Namespace, bookstore' || fail 'quyen staging sai'

# 13. node-agent ở bookstore-system (4b, 5a, 11a)
NODES="$(kubectl get nodes --no-headers | wc -l)"
R="$(kubectl -n "$SYS" get daemonset node-agent -o jsonpath='{.status.numberReady}' 2>/dev/null)"
[ "${R:-0}" -eq "$NODES" ] && pass "node-agent Ready tren $R/$NODES node" || fail "node-agent ${R:-0}/$NODES node"
[ "$(sl k get daemonset,service,serviceaccount --field-selector metadata.name=node-agent --no-headers)" -eq 0 ] \
  && pass 'khong con node-agent trong bookstore, ke ca ServiceAccount' || fail 'con node-agent trong bookstore'
[ "$(kubectl -n "$SYS" get svc node-agent -o jsonpath='{.spec.internalTrafficPolicy}/{.spec.ports[0].port}' 2>/dev/null)" = 'Local/9100' ] \
  && pass 'Service node-agent Local, cong 9100' || fail 'Service node-agent sai'
get "http://node-agent.$SYS.svc.cluster.local:9100/node.json" | grep -Eq "\"node\" *: *\"$WNODE\"" \
  && pass "web doc /node.json cua agent cung node ($WNODE)" || fail 'web khong doc duoc agent cung node'
AS="$(kubectl get service -A -l "$SEL,app.kubernetes.io/component=agent" -o jsonpath='{.items[0].metadata.namespace}/{.items[0].metadata.name}' 2>/dev/null)"
{ [ "$(kubectl -n "${AS%/*}" get service "${AS#*/}" -o jsonpath='{.spec.internalTrafficPolicy}' 2>/dev/null)" = 'Local' ] && \
  get "http://${AS#*/}.${AS%/*}.svc.cluster.local:9100/node.json" | grep -Eq "\"node\" *: *\"$WNODE\""; } \
  && pass 'node-agent cung node tra /node.json qua Service Local tim theo label' || fail 'node-agent qua Service Local loi'
BAD=0
for L in $(kubectl -n "$SYS" get pod -l "$SEL,app.kubernetes.io/component=agent" -o jsonpath='{range .items[*]}{.spec.nodeName}={.status.podIP}{"\n"}{end}' 2>/dev/null); do
  get "http://${L##*=}:9100/node.json" | grep -Eq "\"node\" *: *\"${L%%=*}\"" || { echo "FAIL: node-agent tren ${L%%=*} sai /node.json"; BAD=1; }
done
[ "$BAD" -eq 0 ] && pass 'moi node-agent tra /node.json dung ten node' || FAIL=1
LOGS=''
for i in $(seq 1 12); do
  LOGS="$(get "http://node-agent.$SYS.svc.cluster.local:9100/logs.txt")"
  echo "$LOGS" | grep -q "^last /var/log/pods/${NS}_${WEB}_" && break
  sleep 5
done
{ echo "$LOGS" | tail -n 1 | grep -q "^node=$WNODE files=[1-9]" && \
  echo "$LOGS" | grep -q "^file [0-9]* /var/log/pods/${NS}_${WEB}_" && \
  echo "$LOGS" | grep -q "^last /var/log/pods/${NS}_${WEB}_"; } \
  && pass "/logs.txt cua agent tren $WNODE liet ke log cua $WEB" || fail '/logs.txt thieu hoac sai'

# 14. Metric (11a)
[ "$(kubectl get apiservice v1beta1.metrics.k8s.io -o jsonpath='{.status.conditions[?(@.type=="Available")].status}' 2>/dev/null)" = 'True' ] \
  && pass 'APIService v1beta1.metrics.k8s.io Available' || fail 'Metrics API chua Available'
[ "$(sl kubectl top node --no-headers)" -eq "$NODES" ] && pass 'kubectl top node du moi node' || fail 'kubectl top node thieu node'
TOP=''
for i in $(seq 1 18); do
  TOP="$(kubectl top pod -n "$NS" --no-headers 2>/dev/null)"
  echo "$TOP" | grep -q '^db-0 ' && break
  sleep 5
done
echo "$TOP" | grep -q '^db-0 ' && pass 'kubectl top pod co db-0' || fail 'kubectl top pod chua co db-0'
DBN="$(jp pod/db-0 '{.spec.nodeName}')"
kubectl get --raw "/api/v1/nodes/$DBN/proxy/metrics/resource" 2>/dev/null | grep '^container_memory_working_set_bytes{' \
  | grep 'container="postgres"' | grep -q 'pod="db-0"' \
  && pass 'kubelet /metrics/resource co bo nho cua postgres' || fail 'kubelet /metrics/resource thieu db-0'
API="$(k get pod -l "$SEL,app.kubernetes.io/component=api" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
kubectl -n "$NS" port-forward "pod/$API" 13001:3001 >/dev/null 2>&1 & PF=$!
M=''
for i in $(seq 1 20); do M="$(curl -fsS http://127.0.0.1:13001/metrics 2>/dev/null)" && [ -n "$M" ] && break; sleep 1; done
kill "$PF" 2>/dev/null; wait "$PF" 2>/dev/null
echo "$M" | grep -q '^pgrst_db_pool_max' && pass 'PostgREST /metrics doc duoc qua port-forward' || fail 'khong doc duoc /metrics cua PostgREST'

# 15. Repo và bí mật (3b, 9a)
git check-ignore -q secrets/ci-deployer.kubeconfig \
  && pass 'kubeconfig CI nam ngoai git' || fail 'secrets/ khong bi .gitignore chan'
PGPW="$(k get secret db-credentials -o go-template='{{index .data "postgres-password"}}' 2>/dev/null | base64 -d 2>/dev/null)"
if [ -n "$PGPW" ]; then
  git grep --untracked -qF -e "$PGPW" && fail 'mat khau nam trong file se vao git' || pass 'repo khong chua mat khau'
  kubectl get pods -A -l "$SEL" -o yaml 2>/dev/null | grep -qF -e "$PGPW" \
    && fail 'mat khau xuat hien trong spec Pod' || pass 'spec Pod khong chua mat khau'
else
  fail 'khong doc duoc Secret db-credentials'
fi

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

2. Cập nhật `docs/tech-debt.md` theo B8; kiểm `docs/` có đủ ADR, postmortem và runbook của sprint.

3. Dọn object thử nghiệm, rồi dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
kubectl -n bookstore delete pdb db-thu --ignore-not-found
kubectl -n bookstore delete pod thu-mat-lien-lac --ignore-not-found --wait=true
kubectl -n bookstore delete job bt12-truoc-bao-tri --ignore-not-found --wait=true
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/12/b9-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/12/b9-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/12/b9-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-12: sprint 19 — bao tri node, cong cu do san sang, runbook'
git tag lab-12
git bundle create ~/bookstore-lab-12.bundle --all && git bundle verify ~/bookstore-lab-12.bundle
```

Chép `~/bookstore-lab-12.bundle` về máy host bằng `scp` trước khi đi tiếp.

5. Dọn và trả cluster:

```bash
rm -rf ~/lab-work/12
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd ~/bookstore; EV=~/lab-evidence/12; SEL='app.kubernetes.io/part-of=bookstore'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-12' && echo 'PASS: HEAD mang tag lab-12'
test -s ~/bookstore-lab-12.bundle && echo 'PASS: co bundle'
{ test -x tests/availability-probe.sh && test -x tests/availability-report.sh && test -s docs/runbook-bao-tri-node.md \
  && ls docs/postmortem/12-bao-tri-worker1.md docs/postmortem/12-bao-tri-worker2.md >/dev/null 2>&1; } \
  && echo 'PASS: cong cu, runbook va postmortem cua sprint 19 nam trong repo'
[ "$(sl kubectl get namespace,priorityclass,storageclass,pv,clusterrole,clusterrolebinding -l "$SEL" --no-headers)" -eq 0 ] \
  && [ "$(sl kubectl get all -A -l "$SEL" --no-headers)" -eq 0 ] \
  && echo 'PASS: khong con object nao cua du an'
[ "$(sl kubectl get nodes -l bookstore.example.com/data --no-headers)" -eq 0 ] && echo 'PASS: da go label node cua du an'
test ! -e ~/lab-work/12 && echo 'PASS: da don lab-work'

# Node trở về nguyên trạng sau hai lần tắt–bật
[ "$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.unschedulable}{"\n"}{end}' | grep -c true)" -eq 0 ] \
  && [ "$(kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' \
     | grep -cE 'node.kubernetes.io/(unschedulable|unreachable|not-ready)')" -eq 0 ] \
  && echo 'PASS: khong node nao con cordon hay taint bao tri'
{
  echo "lab-k8s-master kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "lab-k8s-master $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} > "$EV/b9-config-sha.txt"
diff -u "$EV/b0-config-sha.txt" "$EV/b9-config-sha.txt" && echo 'PASS: sau file cau hinh khong doi'

# Hạ tầng của mốc 04-metrics-ready còn nguyên
[ "$(sl kubectl top node --no-headers)" -eq 3 ] && echo 'PASS: top node du ba node'
[ "$(kubectl get storageclass -o jsonpath='{range .items[*]}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' | grep -c '^true$')" -eq 1 ] \
  && [ "$(kubectl get ingressclass --no-headers 2>/dev/null | wc -l)" -ge 1 ] \
  && [ "$(kubectl get apiservice v1beta1.metrics.k8s.io -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')" = 'True' ] \
  && echo 'PASS: StorageClass mac dinh, ingress controller, metrics-server con nguyen'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; mười một dòng `PASS:` của gate cuối
trước khối A5.5; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector trả `No resources
found`; CoreDNS đủ replica; namespace `default` không có Pod. Pod hạ tầng (Traefik, metrics-server,
CoreDNS…) có thể đã đổi node sau hai lần drain — đó là khác biệt hợp lệ. Thư mục
`~/lab-evidence/12/` **giữ lại**. Cluster trở về `04-metrics-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 12

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] `cordon` và `drain` khác nhau đúng hai việc gì? `drain` gọi cơ chế nào ở bên dưới, và vì sao điều
      đó làm PDB của Bookstore có hiệu lực còn `kubectl delete pod` thì không?
- [ ] Drain trần bị từ chối nhắc tới cờ nào? Vì sao gần như mọi cluster đều cần nó, và Pod `node-agent`
      trên worker1 trải qua những gì từ lúc drain tới sau reboot?
- [ ] Ở B5, ba cơ chế nào giữ `web`, `api` không gián đoạn? Nếu probe đo được lỗi, lỗi đó ở tầng nào,
      vì sao PDB của Bookstore không chặn được, và sửa ở đâu?
- [ ] Vì sao `db-0` không thể chạy trên worker1 — hai lý do độc lập? Downtime DB của bạn là bao nhiêu
      giây và gồm những đoạn nào?
- [ ] PDB `db-thu` làm gì với drain ở pha 1? Bài 339 đưa hai giải pháp nào cho ứng dụng stateful một
      instance, và ADR của bạn chọn gì, đổi quyết định khi nào?
- [ ] Graceful node shutdown bật mặc định mà vẫn không hoạt động trên cluster này: hai giá trị nào, dấu
      hiệu nào ở tầng hệ điều hành chứng minh? Nếu bật 30s/10s thì `db-0` có bao nhiêu giây, vì sao?
- [ ] Worker2 tắt: ba tín hiệu của control plane là gì, thành phần nào tạo ra từng cái? Vì sao không
      Pod nào kẹt `Terminating` trong cửa sổ bảo trì của bạn, và kịch bản nào làm `db-0` kẹt?
- [ ] **(Nợ 3a.)** Phase của Pod `node-agent` trên worker2 lúc máy tắt là gì, condition `Ready` của Pod
      là gì, API báo container ra sao? Dòng `Unknown` của bài 47 hiện ra ở đâu?
- [ ] Node `Ready` mà không nhận Pod mới: trạng thái nằm ở đâu, sửa bằng gì? Uncordon xong vì sao Pod
      không tự về, và bạn cân bằng lại thế nào mà vẫn tôn trọng PDB?
- [ ] Swap: ba câu hỏi mà `swapon --show`, `free -m`, `/etc/fstab` trả lời; vì sao câu thứ ba quan
      trọng với chính lab này; với `LimitedSwap`, Pod nào của Bookstore được swap?
- [ ] Cluster có autoscaling Node không, vì sao không? Worker sau drain là Node gì theo bài 171? Có
      autoscaler hợp nhất node thì PDB của `db` đổi vai trò ra sao?
- [ ] `--emulated-version` phải thỏa ràng buộc nào, tách hai việc gì khi nâng cấp? Bầu chọn leader có
      phối hợp thêm gì lên cơ chế Lease? Bốn dữ kiện nào loại trừ Leader Migration?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Nợ lab **#7 — quản lý vòng đời certificate** phát sinh ở chính nhóm bài này (bài
  [156](../156-certificates-vi.md) chỉ là trang trỏ hướng), **chưa trả**; trả ở
  [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) với quy trình `kubeadm
  certs` — trước khi làm giai đoạn đó, đọc lại bài 156 và bài [191](../191-certificates-manual-vi.md).
- Nợ lab #1 (phần VPA), #5 (snapshot cần CSI), #6 (mã hóa Secret at rest), #8 (backup etcd) vẫn mở,
  xem [sổ nợ lab](README.md#5-sổ-nợ-lab). Nợ phase `Unknown` của Lab 3a **đã trả** ở B7.
- Nợ kỹ thuật: hai nợ mới của B8 — đường vào chưa HA, DB một instance trên PV local; `TD-10` (backup
  cùng cluster, chấp nhận có điều kiện) cùng mọi nợ còn mở khác trong `docs/tech-debt.md`.

### Bài giải thích cuối cùng

Trong vài phút, kể lại cửa sổ bảo trì `lab-k8s-worker2` của bạn như kể cho người trực ca sau: vì sao
phải có cửa sổ trong khi worker1 thì không; bản backup và trạng thái "trước" để làm gì; pha 1 — ai
giữ `db-0` lại, Pod nào đi đâu, ai quyết định chỗ mới; pha 2 — chuyện gì xảy ra với `db-0`, với `api`
(vì sao NotReady mà không bị khởi động lại), với trang chủ; lúc máy tắt — control plane thấy gì, Pod
trên node mang trạng thái gì và vì sao trạng thái đó là cũ; lúc máy lên — hai việc bạn phải làm và
điều gì xảy ra nếu quên việc thứ hai; cuối cùng là con số downtime và bằng chứng dữ liệu còn nguyên.
Kết thúc bằng một câu: nếu cluster này có autoscaler hợp nhất node, câu chuyện đổi ở điểm nào.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `kubectl top node` lỗi | Cluster chưa ở `04-metrics-ready` | Restore cả ba VM về `04-metrics-ready` |
| B1, B3: file `configz` rỗng | Đường control plane → kubelet hỏng | Chạy lại [tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet); đọc tạm `/var/lib/kubelet/config.yaml` qua `ssh` và ghi rõ trường vắng là đang dùng mặc định |
| B2: `gitVersion` khác `K8S_BASELINE` | Điền nhầm biến, hoặc cluster bị nâng cấp ngoài lộ trình | Kiểm biến trước; lệch thật thì dừng và đối chiếu A1.3 theo quy trình Lab 00 |
| B3: hai thời lượng khác `0s`, hoặc thấy khóa của kubelet | Node đã bật graceful node shutdown từ trước — lệch baseline | Không sửa file; ghi giá trị thật vào ADR, hiểu rằng kubelet sẽ chấm dứt Pod theo hai giai đoạn khi tắt máy |
| B3: `fstab=1` trên một node | Dòng swap chưa comment | Xử lý theo [A4.1](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites) **trước** B5; không tắt máy khi chưa sửa |
| B4: mọi mẫu `000/7` | Tên cổng của Service Traefik khác, hoặc `externalTrafficPolicy: Local` nên NodePort trên master không chuyển tiếp | Xem `kubectl -n traefik get svc traefik -o yaml`; với `Local`, đặt `ENTRY_IP` là IP node đang chạy Traefik và ghi rõ hạn chế vào mục *Cách đo* |
| B4: mọi mẫu `404` | Host trong Ingress khác `bookstore.lab.local` | Đối chiếu Ingress với [mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label); không sửa probe |
| B5, B7: drain đòi `--force` | Còn Pod trần trên node: Pod thử nghiệm quên xóa, hoặc Pod ⭐ tạo trước pha 2 | Xóa Pod thử nghiệm rồi drain lại; không dùng `--force` |
| B5: drain treo, lặp `Cannot evict pod ... disruption budget` với `web` hoặc `api` | Bản sao trên node kia chưa Ready hoặc không có; PDB đang `disruptionsAllowed: 0` | `kubectl -n bookstore get pdb,pods -o wide`; tiền điều kiện của B5 chưa đạt. Uncordon, chờ đủ bản sao trải hai node, làm lại |
| B5: Pod thay thế của `web` nằm `Pending`, lý do nhắc topology spread | Ràng buộc phân bố `DoNotSchedule` tính cả node đang cordon vào độ lệch | Đọc bài [140](../140-topology-spread-constraints-vi.md), mục *Định nghĩa ràng buộc phân bố* — trường `nodeTaintsPolicy`; sửa trong repo, ghi ADR, triển khai lại rồi làm lại B5 |
| B5: trang chủ mất ngay khi drain, `web` chỉ có một Pod | Bảo trì bắt đầu khi HPA chưa nâng `web` khỏi một bản sao; PDB `maxUnavailable: 1` cho phép trục xuất Pod duy nhất | Uncordon, chờ `web` đạt `minReplicas` và trải hai worker, làm lại từ tiền điều kiện; ghi vào postmortem |
| B5, B7: số bản sao `web` đổi giữa chừng, hoặc HPA báo không lấy được metric | HPA vẫn chạy: Pod mới khởi động làm CPU tăng tạm thời; metrics-server bị evict khỏi node đang drain nên HPA tạm thiếu số đo và giữ nguyên số bản sao | Không can thiệp; kiểm tra so với số bản sao hiện tại. Ghi khoảng HPA mất metric vào postmortem — nó là một gián đoạn của add-on, không phải của `web` |
| B5, B7: Pod thay thế bị từ chối, event `exceeded quota` | Pod đang kết thúc vẫn tính vào ResourceQuota trong lúc ân hạn | Thường tự hết khi Pod cũ biến mất; lặp lại mãi thì quota của Lab 7b không chừa chỗ cho bảo trì — ghi vào postmortem |
| B5, B7: VM không chuyển sang *Powered off* | Hệ điều hành đang tắt dịch vụ | Chờ rồi chạy lại khối PowerShell; chỉ dùng `vmrun -T ws stop <vmx> hard` khi chờ mãi không xong, và ghi vào postmortem rằng lần tắt đó là cưỡng bức |
| B5, B7: `Ready` không thành `Unknown` sau nhiều vòng | Khoảng chờ của node controller phụ thuộc cấu hình | Tăng số vòng chờ; kiểm `kube-controller-manager` còn `Running` |
| B5, B7: node không trở lại `Ready` sau khi bật | VM không boot, IP tĩnh không lên, hoặc kubelet không chạy (swap bật lại) | Đi từ dưới lên: console VM → `ssh` → `systemctl is-active kubelet containerd` → `journalctl -u kubelet -n 80`. Vài phút không sửa được thì tắt và restore **cả ba** VM về `04-metrics-ready` |
| B5: sau uncordon và evict, Pod mới vẫn lên node đông | Anti-affinity của `api` là mềm; điểm số khác lấn át | Evict thêm một lần; vẫn dồn thì ghi vào postmortem — smoke ở B9 sẽ bắt nếu sau triển khai sạch vẫn dồn |
| B5, B7: `node-agent` đổi UID sau reboot | Pod DaemonSet bị xóa và tạo lại thay vì chạy lại container | Không phải lỗi thao tác; ghi lại và giải thích trong postmortem |
| B7: drain pha 1 không bị chặn, `db-0` bị evict luôn | PDB tạo sau khi drain đã chạy, hoặc selector không khớp label của `db-0` | Chờ `db-0` lên lại (uncordon nếu cần), kiểm `kubectl -n bookstore get pdb db-thu` có `ALLOWED DISRUPTIONS 0`, làm lại từ pha 1 |
| B7: `db-0` vẫn `Pending` sau uncordon | Thiếu label dữ liệu trên worker2, hoặc PV không còn | `kubectl -n bookstore describe pod db-0`; label do `deploy.sh` gắn từ Lab 7a — chạy lại `./scripts/deploy.sh` |
| B7: `api` không Ready lâu sau khi `db-0` Ready | PostgREST đang chờ backoff trước lần kết nối lại | Chờ; nếu `CrashLoopBackOff` với lỗi xác thực mật khẩu thì `secrets/db.env` đã đổi trong khi dữ liệu giữ mật khẩu cũ — xem [mục 5.4](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) |
| B7 ⭐: Pod thử nghiệm bị từ chối | Thiếu trường `securityContext` theo `restricted`, hoặc `resources` ngoài LimitRange | Đọc thông báo từ chối; sửa manifest theo lời giải |
| B7 ⭐: Pod thử nghiệm mãi không có `deletionTimestamp` | Chưa hết `tolerationSeconds` tính từ lúc gắn taint `unreachable` | Chờ thêm; khoảng thời gian phụ thuộc cấu hình và giá trị đọc được ở khối ⭐ đầu |
| B9: `diff` checksum báo khác | Cấu hình đã bị sửa hoặc node boot với file khác | Cluster lệch baseline: restore cả ba VM về `04-metrics-ready`, không sang lab sau |
| B9: `kubectl top node` chỉ hai dòng | Worker vừa boot chưa báo metric, hoặc metrics-server vừa bị dời | Chờ rồi chạy lại; APIService không `Available` thì xem [mục 4 của Lab 11a](LAB-11A-OBSERVABILITY.md#4-troubleshooting-của-lab-này) |
| B9: xóa namespace treo `Terminating` | `db-0` đang trong 60 giây ân hạn | Chờ; `teardown.sh` tự chờ |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Cluster Administration](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/)
- [Kubernetes v1.35 — Node Shutdowns](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/node-shutdown/)
- [Kubernetes v1.35 — Swap memory management](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/)
- [Kubernetes v1.35 — Node Autoscaling](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/node-autoscaling/)
- [Kubernetes v1.35 — Installing Addons](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/addons/)
- [Kubernetes v1.35 — Compatibility Version For Kubernetes Control Plane Components](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/compatibility-version/)
- [Kubernetes v1.35 — Coordinated Leader Election](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/coordinated-leader-election/)
- [Kubernetes v1.35 — Certificates](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/certificates/)
- [Kubernetes v1.35 — Administer a Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/)
- [Kubernetes v1.35 — Migrate Replicated Control Plane To Use Cloud Controller Manager](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/controller-manager-leader-migration/)
- [Kubernetes v1.35 — Safely Drain a Node](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)
- [Kubernetes v1.35 — Disruptions](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Kubernetes v1.35 — Specifying a Disruption Budget for your Application](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Kubernetes v1.35 — API-initiated Eviction](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)
- [Kubernetes v1.35 — Nodes](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/nodes/)
- [Kubernetes v1.35 — Leases](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/leases/)
- [Kubernetes v1.35 — Taints and Tolerations](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes v1.35 — Pod Lifecycle](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes v1.35 — Container Lifecycle Hooks](https://v1-35.docs.kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
- [Kubernetes v1.35 — Guaranteed Scheduling For Critical Add-On Pods](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)
- [Kubernetes v1.35 — Feature Gates](https://v1-35.docs.kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)
- [Kubernetes v1.35 — `kubectl drain`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_drain/)
- [Kubernetes v1.35 — Kubelet Configuration (v1beta1)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [PostgREST v14 — Admin Server](https://docs.postgrest.org/en/v14/references/admin_server.html)
- [PostgREST v14 — Connection Pool, mục Automatic Recovery](https://docs.postgrest.org/en/v14/references/connection_pool.html)
- [systemd — inhibitor locks](https://www.freedesktop.org/wiki/Software/systemd/inhibit/)
- [systemd — `systemd-inhibit`](https://www.freedesktop.org/software/systemd/man/latest/systemd-inhibit.html)
