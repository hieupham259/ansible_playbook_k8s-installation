# Runbook: Khôi phục cụm kubeadm sau khi host mất điện làm hỏng etcd

> **Phụ thuộc:** cụm dựng theo [`runbook-k8s-vmware.md`](runbook-k8s-vmware.md) (Phase 1) và đang chạy [`runbook-k8s-vmware-phase2.md`](runbook-k8s-vmware-phase2.md) (Phase 2). Mọi tham số node, IP, phiên bản lấy từ Phase 1 [§2](runbook-k8s-vmware.md#2-quy-hoạch); file này không lặp lại.
>
> **Phạm vi:** một sự cố cụ thể — máy host Windows tắt đột ngột, ba VM bật lại, `kubectl` báo `dial tcp 192.168.100.111:6443: connection refused`, Pod `etcd` crashloop vì file `/var/lib/etcd/member/snap/db` hỏng. Runbook trả lời ba câu: **vì sao** lần này hỏng còn các lần sập trước thì không (§1), **restore** từ snapshot etcd tới khi Rancher và Phase 2 chạy lại (§2–§6), và **reset** nếu restore thất bại (§7).
>
> **Cách chạy bắt buộc:** đi theo đúng thứ tự. Sau mỗi khối có nhãn **DỪNG — GỬI OUTPUT**, gửi nguyên output; chỉ sang bước sau khi PASS. Không `kubeadm reset`, không xóa `/var/lib/etcd`, không đụng hai worker cho tới khi runbook nói rõ.
>
> **Môi trường:** lệnh `bash` chạy trên `k8s-master` bằng user `ubuntu`, trừ khi tiêu đề ghi rõ worker hoặc host. Lệnh `powershell` chạy trên máy host Windows.

---

## Mục lục

1. [Nguyên nhân — vì sao lần này etcd hỏng](#1-nguyên-nhân--vì-sao-lần-này-etcd-hỏng)
2. [Gate đầu vào — xác nhận đúng sự cố và có backup dùng được](#2-gate-đầu-vào--xác-nhận-đúng-sự-cố-và-có-backup-dùng-được)
3. [Restore etcd từ snapshot](#3-restore-etcd-từ-snapshot)
4. [Đưa cụm về trạng thái khỏe sau restore](#4-đưa-cụm-về-trạng-thái-khỏe-sau-restore)
5. [Cài lại cert-manager và Rancher](#5-cài-lại-cert-manager-và-rancher)
6. [Làm lại Phase 2 tới bước đang dở](#6-làm-lại-phase-2-tới-bước-đang-dở)
7. [Đường lui khi restore thất bại — reset toàn cụm](#7-đường-lui-khi-restore-thất-bại--reset-toàn-cụm)
8. [Phòng ngừa cho lần sau](#8-phòng-ngừa-cho-lần-sau)
9. [Troubleshooting của runbook này](#9-troubleshooting-của-runbook-này)
10. [Checklist hoàn tất](#10-checklist-hoàn-tất)
11. [Nguồn official](#11-nguồn-official)

---

## 1. Nguyên nhân — vì sao lần này etcd hỏng

### 1.1. Bằng chứng đã thu (30/09/2026, trên `k8s-master`)

| Bằng chứng | Giá trị | Ý nghĩa |
| --- | --- | --- |
| `kubectl get nodes` | `dial tcp 192.168.100.111:6443: connection refused` | Không có gì lắng nghe cổng 6443; không phải firewall (firewall cho `timeout`/`no route`), không phải kubeconfig |
| `systemctl is-active containerd kubelet` | cả hai `active` | Tầng OS/runtime bình thường |
| `crictl ps -a` | `etcd` Exited, ATTEMPT 91; `kube-apiserver` ATTEMPT 88–89 | etcd là gốc; apiserver chết theo vì không có etcd |
| `crictl inspect` container etcd | `exitCode 2`, `reason Error`, sống **23 ms** | etcd chết ngay lúc mở db, trước khi phục vụ gì |
| `crictl logs` etcd | `panic: freepages: failed to get all reachable pages (the first key ... on leaf page(2871) needs to be >= the key in the ancestor ...)` | Cây B+tree của bbolt không nhất quán |
| `df -h /var/lib/etcd`, `dmesg` | 29% dùng, không `I/O error`, ext4 `rw` | Không phải đĩa đầy, không phải filesystem lỗi |
| `journalctl -u kubelet \| grep probe` | rỗng | Không phải liveness probe giết etcd |
| `ls -la /var/lib/etcd/member/` | `snap/db` 34 MB sửa lần cuối `Sep 30 02:13`; WAL 375 MB | Dữ liệu nằm ở `db`; các file `.snap` là metadata raft, không chứa key-value |

### 1.2. Cơ chế hỏng — điều đã chứng minh và điều còn là giả thuyết

**Đã chứng minh** bằng bằng chứng ở §1.1:

- File `db` **không nhất quán về cấu trúc**: bbolt panic khi quét cây page, lặp lại y hệt ở mọi
  lần khởi động. Đây là hỏng dữ liệu, không phải lỗi tạm thời.
- **Không phải** đĩa đầy, I/O error, filesystem read-only, liveness probe, OOM.
- Thời điểm hỏng trùng với lần host mất điện; `db` được ghi lần cuối `Sep 30 02:13`.
- Restart không sửa được: mỗi lần kubelet khởi động lại container, etcd mở cùng file, quét cùng
  cây page, panic cùng chỗ.

**Cơ chế bbolt và vì sao panic này đáng chú ý.** etcd lưu toàn bộ key-value trong một file bbolt
(`member/snap/db`). bbolt được thiết kế để **chịu được crash giữa transaction**: nó giữ hai meta
page luân phiên; một transaction ghi các page dữ liệu mới, `fsync`, rồi mới ghi meta page trỏ tới
cây mới và `fsync` lần nữa. Khi mở file sau crash, bbolt chọn meta page hợp lệ có transaction id cao
nhất; page của transaction ghi dở không được meta page nào trỏ tới nên bị bỏ qua. Thiết kế đó đúng
với điều kiện: khi `fsync` trả về, dữ liệu đã bền vững và **đúng thứ tự**. Thông báo panic ở đây
nói rằng một meta page *hợp lệ* đang trỏ tới cây có leaf page mang key nhỏ hơn key của page cha —
tức là điều kiện đó đã bị vi phạm ở đâu đó bên dưới `fsync`, hoặc có bug ở tầng phần mềm.

**Giả thuyết hợp lý nhất, không chứng minh được sau sự cố:** cache ghi phía host. Chuỗi lưu trữ
của VM là etcd → ext4 trong guest → virtual disk → file `.vmdk` trên NTFS của host Windows → đĩa
vật lý. `fsync` của guest chỉ bảo đảm tới lớp virtual disk; host Windows và Workstation còn cache
ghi riêng. Khi **host** mất điện, phần cache đó có thể mất, và mất không theo thứ tự. Guest không
biết gì về việc này. Các nguyên nhân khác (bug ext4, bbolt, etcd, firmware đĩa) không bị loại trừ
bằng bằng chứng hiện có, nhưng không có dấu hiệu nào trỏ tới chúng và cùng bộ phiên bản đã chạy
ổn nhiều tháng.

**Vì sao các lần host sập trước không hỏng — không trả lời chắc chắn được.** Với giả thuyết trên,
`db` chỉ hỏng khi tại đúng thời điểm mất điện có **cả hai** điều: (a) còn dữ liệu guest đã `fsync`
nhưng host chưa ghi xuống đĩa vật lý; (b) etcd đang giữa một commit. Cả hai đều là chuyện thời
điểm, không kiểm tra lại được sau sự cố. Điều duy nhất nói được: xác suất (b) đã tăng mạnh sau khi
cài Rancher, vì Rancher ghi lease, status và CR liên tục, cộng với leader lease của
`kube-controller-manager`/`kube-scheduler` và `Lease` node của mỗi kubelet. Các lần trước không
"an toàn hơn"; chúng chỉ không trúng. Sự cố 15/09/2026 (login Rancher 404, xem
[§15 Phase 1](runbook-k8s-vmware.md#rancher-đăng-nhập-404-tại-v1extcattleioselfuser-sau-khi-reboot-node))
là ví dụ: cùng kịch bản reboot, etcd còn nguyên.

Hệ quả cho runbook: phần phòng ngừa ở §8 **không được dựa vào may mắn**; nó phải rút ngắn khoảng
mất dữ liệu (backup định kỳ, snapshot VM) và loại bỏ điều kiện (a) bằng cách tắt guest trước host.

### 1.3. Vì sao không cứu được file `db` hiện tại

- etcd chạy với `NoFreelistSync: true` (thấy trong log khởi động). Khi mở db, bbolt phải **quét toàn
  bộ cây page** để dựng lại freelist; chính lần quét đó phát hiện cây hỏng và panic. Đây là cơ chế
  phát hiện, không phải nguyên nhân, nhưng nó nghĩa là **mọi công cụ mở db ở chế độ ghi** (`etcd`,
  `etcdutl snapshot restore` trên chính file này) đều dừng ở cùng chỗ.
- Image `registry.k8s.io/etcd:3.6.6-0` không có công cụ sửa bbolt; đưa công cụ ngoài vào là ra khỏi
  môi trường runbook và cũng không bảo đảm dữ liệu sau khi "sửa" còn nhất quán về mặt Kubernetes.
- WAL 375 MB còn nguyên nhưng WAL chỉ replay được lên một `db` nhất quán; không dựng lại `db` từ
  WAL và `.snap` được.

**Kết luận:** đường lui duy nhất trong môi trường runbook là **restore từ snapshot etcd** đã tạo ở
[§14.0.1 Phase 1](runbook-k8s-vmware.md#1401-backup-etcd-và-cấu-hình-trước-thay-đổi-lớn). Ba VM
không có snapshot VMware nào (`vmrun listSnapshots` trả `Total snapshots: 0` cho cả ba), nên
không có đường revert VM.

### 1.4. Restore về snapshot 14/08/2026 nghĩa là gì

Snapshot `20260814-190914` được chụp ngay trước §14.1, tức là cụm đã xong §1–§13 của Phase 1.

| Còn nguyên sau restore | Mất, phải làm lại |
| --- | --- |
| OS ba node, containerd, kubelet, image đã pull | cert-manager ([§14.1](runbook-k8s-vmware.md#141-cài-cert-manager-rancher-cần-để-cấp-tls-nội-bộ)) |
| `/etc/kubernetes/pki`, kubeconfig — không đụng | Split DNS CoreDNS ([§14.2](runbook-k8s-vmware.md#142-cấu-hình-split-dns-cho-client-trong-cụm-local)) |
| Cụm kubeadm, Flannel, 2 worker đã join | Rancher ([§14.3](runbook-k8s-vmware.md#143-cài-rancher-helm-pin-2143)–[§14.7](runbook-k8s-vmware.md#147-gate-hoàn-thành)); bootstrap password mới |
| Helm, metrics-server, `local-path` ([§8.7](runbook-k8s-vmware.md#87-tầng-7--công-cụ-và-add-on-kubeadm-không-cài-sẵn)) | Mọi thứ tạo trong Rancher sau 14/08 (user, setting, project) |
| Traefik ([§9](runbook-k8s-vmware.md#9-cài-ingress-controller-traefik)), app mẫu + Ingress ([§10](runbook-k8s-vmware.md#10-deploy-app-mẫu--ingress)) | Toàn bộ Kubernetes object của Phase 2: namespace `three-tier`, Secret, MongoDB + PVC, backend, frontend |
| cloudflared + Secret token ([§12](runbook-k8s-vmware.md#12-tạo-cloudflare-tunnel-chạy-trong-cụm)) | Object PV/PVC của MongoDB. **Dữ liệu trên đĩa worker thì không mất**: `local-path` ghi vào `/opt/local-path-provisioner/` của node, restore etcd không đụng tới; xem §4.5 để quyết định giữ hay bỏ |
| Toàn bộ state phía Cloudflare: domain, tunnel, route, Access application — nằm ở Cloudflare, không ở etcd | — |
| Source và image Phase 2 — nằm ở Git và Docker Hub, không ở etcd | — |

---

## 2. Gate đầu vào — xác nhận đúng sự cố và có backup dùng được

### 2.1. Xác nhận đúng mẫu lỗi

**Mục đích:** runbook này chỉ dành cho etcd hỏng file `db`. Nếu etcd chết vì lý do khác (đĩa đầy,
probe, cert hết hạn), dừng lại và xử lý nguyên nhân đó; restore không giúp gì.

```bash
sudo systemctl is-active containerd kubelet
ls /etc/kubernetes/manifests/
sudo crictl ps -a | grep -E 'etcd|kube-apiserver'

ETCD_ID=$(sudo crictl ps -a --name '^etcd$' -q | head -1)
sudo crictl inspect "$ETCD_ID" | grep -E '"exitCode"|"reason"'
sudo crictl logs "$ETCD_ID" 2>&1 | grep -E 'panic|fatal' | head -5

df -h /var/lib/etcd
sudo dmesg | grep -iE 'I/O error|remount|read-only' | tail -5
sudo kubeadm certs check-expiration 2>/dev/null | grep -iE 'etcd|apiserver' | head -8
```

PASS khi: containerd và kubelet `active`; đủ 4 manifest; etcd `Exited` với ATTEMPT tăng; `exitCode` 2,
`reason Error`; log có dòng `panic:` liên quan bbolt/`freepages`/`page`; đĩa còn trống; `dmesg`
không có I/O error; không cert nào `EXPIRED`.

> **STOP** nếu `exitCode` là `137` (OOM), nếu `dmesg` có I/O error, hoặc nếu có cert hết hạn: đó là
> sự cố khác, xem [bảng lỗi Phase 1](runbook-k8s-vmware.md#bảng-lỗi-thường-gặp) và
> [gia hạn cert](runbook-k8s-vmware.md#quản-lý-và-gia-hạn-certificate-kubeadm).

> **DỪNG — GỬI OUTPUT CHECKPOINT 2.1.**

### 2.2. Xác nhận backup còn nguyên vẹn ở hai nơi

Backup được tạo và copy theo [§14.0.1](runbook-k8s-vmware.md#1401-backup-etcd-và-cấu-hình-trước-thay-đổi-lớn).
Đổi `STAMP` nếu bạn có bản mới hơn `20260814-190914`; luôn dùng bản **mới nhất còn hợp lệ**.

Bản được chọn được **ghi vào một file** để mọi bước sau (§2.3, §2.4, §3.3) đọc lại đúng bản đó;
không bước nào gán lại đường dẫn bằng tay, nên không thể kiểm tra một bản rồi restore bản khác.

```bash
STAMP=20260814-190914            # đổi nếu chọn bản khác; chỉ đổi ở đúng dòng này
BACKUP_DIR="$HOME/k8s-backups/$STAMP"
SNAPSHOT="$BACKUP_DIR/etcd-snapshot.db"

ls -la "$HOME/k8s-backups/"
sha256sum "$HOME/k8s-backups/$STAMP.tar.gz"
test -s "$SNAPSHOT" && echo "PASS: snapshot tồn tại, $(stat -c %s "$SNAPSHOT") byte"
sudo diff -rq /etc/kubernetes/pki "$BACKUP_DIR/etc-kubernetes/pki" && echo 'PASS: pki unchanged'

# Chốt bản đã chọn cho các bước sau
echo "$SNAPSHOT" > ~/etcd-restore-snapshot-path.txt
cat ~/etcd-restore-snapshot-path.txt
# PASS: in đúng đường dẫn của bản vừa kiểm tra
```

Trên **host Windows**:

```powershell
$Stamp = '<STAMP>'   # cùng giá trị STAMP đã đặt ở block bash phía trên
(Get-FileHash "E:\courses\Ansible\ansible_playbook_k8s-installation\k8s-backups\$Stamp.tar.gz" -Algorithm SHA256).Hash.ToLower()
```

**Điều kiện chung:** hai hash SHA-256 của archive trên `k8s-master` và host Windows trùng nhau;
`etcd-snapshot.db` tồn tại, khác rỗng; file `~/etcd-restore-snapshot-path.txt` in đúng đường dẫn
snapshot vừa kiểm tra; và **một trong hai** kết quả về PKI dưới đây đạt yêu cầu.
Nếu hai hash hiện tại khác nhau hoặc thiếu bất kỳ điều kiện chung nào, **chưa PASS**; gửi output
để đối chiếu trước khi tiếp tục, kể cả khi không còn hash ban đầu.

**Chọn nhánh theo bằng chứng checksum còn lưu:**

- **Còn hash ghi lúc tạo backup:** PASS khi đủ điều kiện chung và hai hash hiện tại cùng trùng
  hash ban đầu. Hash lệch bản ghi ban đầu → **STOP**, gửi output; không dùng nhánh thiếu hash để
  bỏ qua một khác biệt đã biết.
- **Không còn hash/output lúc tạo backup:** ghi rõ việc thiếu bằng chứng này tại checkpoint.
  Khi đủ điều kiện chung, ghi **PASS có giới hạn — được sang §2.3 để kiểm tra tiếp snapshot**.
  Hai hash hiện tại chỉ chứng minh hai archive hiện tại trùng nhau về checksum; **chưa chứng minh
  archive không thay đổi kể từ lúc tạo backup**. Không gọi hash vừa tính lại là hash ban đầu.

Nhánh thiếu hash **không bỏ qua gate nào phía sau**: §2.3 đọc metadata bằng
`etcdutl snapshot status`, §2.4 đối chiếu trạng thái trong snapshot, rồi mới xét điều kiện sang §3.
Bảng `snapshot status` đọc được **chưa phải bằng chứng đã kiểm tra hash toàn vẹn nhúng trong
snapshot**, và cột `HASH` của bảng không phải SHA-256 của archive `.tar.gz`.
Với snapshot tạo bằng `etcdctl snapshot save`, `etcdutl snapshot restore` ở §3.3 kiểm tra hash
toàn vẹn nhúng trong snapshot; giữ nguyên lệnh restore, **không thêm `--skip-hash-check`**.
Nếu restore báo lỗi, dừng theo gate §3.3 và phân loại lỗi tại §3.4; không suy ra thiếu hash ban
đầu đồng nghĩa snapshot hỏng hay cho phép reset. Cơ chế kiểm tra này áp dụng cho snapshot etcd,
không thay thế bằng chứng checksum ban đầu của toàn bộ archive chứa cả `/etc/kubernetes`.
Nguồn: [etcd v3.6 — Status of a snapshot](https://etcd.io/docs/v3.6/op-guide/recovery/#status-of-a-snapshot)
và [Integrity Checks](https://etcd.io/docs/v3.6/op-guide/recovery/#integrity-checks).

**Nếu `diff` in `PASS: pki unchanged`:** cert hiện tại vẫn là cert lúc backup, chỉ cần restore
data etcd, không đụng `/etc/kubernetes`. Sang checkpoint.

**Nếu `diff` báo khác:** `diff` chỉ chứng minh file khác, chưa nói khác ở đâu. Kubernetes phân biệt
hai việc: **renew cert thành phần** (apiserver, etcd server/peer, controller-manager…) bằng CA hiện
có — `kubeadm certs renew` làm việc này và không đổi CA — và **thay CA hoặc khóa ký service
account**. Restore etcd chỉ an toàn trong trường hợp thứ nhất: dữ liệu trong snapshot (token
service account, cert client của kubelet ghi trong Node/CSR, Secret TLS) được ký bởi CA và
`sa.key` lúc backup; nếu CA/`sa.key` đã đổi thì dữ liệu restore về không còn khớp PKI hiện tại và
cụm sẽ hỏng theo cách khó chẩn đoán. Phân biệt bằng lệnh sau:

```bash
BPKI="$BACKUP_DIR/etc-kubernetes/pki"
# 1. Các file gốc của chuỗi tin cậy PHẢI giống hệt
for f in ca.crt ca.key etcd/ca.crt etcd/ca.key front-proxy-ca.crt front-proxy-ca.key sa.key sa.pub; do
  if sudo cmp -s "/etc/kubernetes/pki/$f" "$BPKI/$f"; then echo "PASS: $f giống backup"
  else echo "STOP: $f KHÁC backup"; fi
done
# 2. Những file còn lại khác nhau là cert thành phần đã renew — liệt kê để biết
sudo diff -rq /etc/kubernetes/pki "$BPKI" | grep -vE '(^|/)(ca|front-proxy-ca)\.(crt|key)|sa\.(key|pub)' || true
# 3. Cert thành phần hiện tại phải do đúng CA hiện tại ký và chưa hết hạn
sudo openssl verify -CAfile /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/apiserver.crt
sudo openssl verify -CAfile /etc/kubernetes/pki/etcd/ca.crt /etc/kubernetes/pki/etcd/server.crt
sudo kubeadm certs check-expiration
```

- Mọi dòng ở mục 1 là `PASS`, `openssl verify` in `OK`, `check-expiration` không có `EXPIRED` →
  chỉ cert thành phần được renew bằng CA cũ. Restore etcd bình thường và **giữ PKI hiện tại**,
  không chép PKI cũ từ backup đè lên (cert cũ có thể đã hết hạn).
- Bất kỳ dòng `STOP` nào ở mục 1 (CA hoặc `sa.key` đã đổi sau backup) → **STOP toàn bộ
  runbook**. Đây là tình huống ngoài phạm vi file này: cần snapshot etcd chụp **sau** khi đổi CA,
  hoặc quy trình xoay CA có kế hoạch. Gửi output rồi mới bàn tiếp; không restore, không reset.
- `openssl verify` FAIL hoặc có cert `EXPIRED` với CA giống backup → renew cert thành phần theo
  [gia hạn cert Phase 1](runbook-k8s-vmware.md#quản-lý-và-gia-hạn-certificate-kubeadm) trước, rồi
  chạy lại mục 3; xong mới sang §2.3.

> **DỪNG — GỬI OUTPUT CHECKPOINT 2.2.**
> Gửi STAMP, hai hash hiện tại, kết quả kiểm tra snapshot/đường dẫn và PKI; kèm hash ban đầu nếu
> còn lưu, hoặc ghi rõ **không còn hash/output lúc tạo backup**. Chỉ sang §2.3 khi nhánh tương ứng
> đạt PASS hoặc PASS có giới hạn như quy định trên; các điều kiện STOP về PKI vẫn áp dụng đầy đủ.

### 2.3. Xác nhận công cụ restore và tham số etcd

etcd 3.6 đã bỏ `etcdctl snapshot restore`; chỉ còn `etcdutl`. Image etcd của kubeadm có sẵn trong
containerd và có đóng gói `etcdutl`, nên chạy nó bằng `ctr` từ chính image đó — không tải gì thêm.
Snapshot được mount **chỉ đọc**; lệnh `snapshot status` không sửa file.

```bash
(
  set -e
  sudo grep -E -- '--(name|data-dir|initial-advertise-peer-urls|initial-cluster)=' \
    /etc/kubernetes/manifests/etcd.yaml

  IMAGE='registry.k8s.io/etcd:3.6.6-0'
  SNAPSHOT=$(cat ~/etcd-restore-snapshot-path.txt)   # bản đã chốt ở §2.2
  echo "snapshot: $SNAPSHOT"

  sudo ctr -n k8s.io images ls -q | grep -Fx "$IMAGE"

  sudo ctr -n k8s.io run --rm \
    "$IMAGE" "etcdutl-version-$(date +%s)" \
    /usr/local/bin/etcdutl version

  sudo ctr -n k8s.io run --rm \
    --mount "type=bind,src=$SNAPSHOT,dst=/snapshot.db,options=bind:ro" \
    "$IMAGE" "etcdutl-status-$(date +%s)" \
    /usr/local/bin/etcdutl snapshot status /snapshot.db -w table
)
```

PASS khi output có đủ:

```text
- --data-dir=/var/lib/etcd
- --initial-advertise-peer-urls=https://192.168.100.111:2380
- --initial-cluster=k8s-master=https://192.168.100.111:2380
- --name=k8s-master
registry.k8s.io/etcd:3.6.6-0
etcdutl version: 3.6.6
API version: 3.6
```

và bảng `snapshot status` in ra HASH, REVISION, TOTAL KEYS, TOTAL SIZE (lần chạy 30/09/2026:
revision `2404659`, 385 key, 4.9 MB). **Ghi lại REVISION** — §4.2 dùng nó để chứng minh cụm
đang chạy trên dữ liệu restore.

> **DỪNG — GỬI OUTPUT CHECKPOINT 2.3.**

### 2.4. Xác nhận snapshot chứa đúng trạng thái mong đợi

**Mục đích:** bảng ở §1.4 và toàn bộ §5–§6 giả định snapshot được chụp **sau §13 và trước §14.1**.
Thứ tự runbook không phải bằng chứng; kiểm trực tiếp trên file. etcd lưu tên key Kubernetes dưới
dạng chuỗi thuần trong file bbolt, nên `grep` đọc được mà không cần restore. Lệnh chỉ đọc file.

```bash
SNAPSHOT=$(cat ~/etcd-restore-snapshot-path.txt)   # bản đã chốt ở §2.2
echo "snapshot: $SNAPSHOT"
for k in \
  /registry/namespaces/cloudflare \
  /registry/deployments/cloudflare/cloudflared \
  /registry/secrets/cloudflare/tunnel-token \
  /registry/namespaces/traefik \
  /registry/namespaces/cattle-system; do
  printf '%-48s %s\n' "$k" "$(grep -a -c -F "$k" "$SNAPSHOT")"
done
```

Đọc kết quả đúng giới hạn của nó. File bbolt của etcd giữ **lịch sử revision chưa compact và
tombstone** của key đã xóa, và etcd compact lịch sử định kỳ. Vì vậy: số đếm lớn hơn 0 chỉ chứng
minh key **từng tồn tại** tại một thời điểm nào đó còn trong cửa sổ chưa compact, không chứng minh
object còn ở revision cuối; số đếm bằng 0 chỉ chứng minh key **không có trong cửa sổ đó**, không
chứng minh nó chưa từng xuất hiện (lịch sử cũ hơn đã bị compact). Cả hai chiều đều là dấu hiệu,
không phải bằng chứng.

PASS khi bốn dòng đầu **lớn hơn 0** và dòng `cattle-system` **bằng 0** (lần chạy 01/10/2026 với
snapshot 14/08: `1 2 1 1 0`). Đây là gate **định hướng**: nó chọn nhánh cho §4–§5; kết luận cuối
về object nào thật sự có nằm ở §4.2 (`kubectl get ns`, `kubectl -n cloudflare get deploy,secret`)
sau restore, và mọi nhánh dưới đây đều phải được xác nhận lại ở đó.

Nhánh rẽ khi không PASS:

- Ba dòng `cloudflare` bằng 0 → nhiều khả năng snapshot chụp trước §12. Vẫn restore bình thường;
  §4.3 tự phát hiện Deployment `cloudflared` có hay không. Nếu không có, chạy lại
  [§12.2 Phase 1](runbook-k8s-vmware.md#122-deploy-cloudflared-vào-cụm) ngay sau checkpoint 4.3
  và **trước §4.4**, với token của tunnel `homelab-k8s` **hiện có** (lấy lại trên dashboard), không
  tạo tunnel mới; route và Access còn nguyên. Gate §4.4 giữ nguyên.
- Dòng `cattle-system` lớn hơn 0 → **nghi ngờ** snapshot chụp sau khi cài Rancher (hoặc Rancher
  từng được cài rồi gỡ). Không kết luận ở đây. Restore bình thường; ở §4.2 nếu `kubectl get ns` có
  `cattle-system` thì §5 không được `helm install` lại: kiểm tra release theo
  [§14.0](runbook-k8s-vmware.md#140-gate-trước-khi-thay-đổi-cluster) rồi repair theo
  [§15 Phase 1](runbook-k8s-vmware.md#rancher-đăng-nhập-404-tại-v1extcattleioselfuser-sau-khi-reboot-node)
  nếu login lỗi. Nếu §4.2 không có `cattle-system` thì đi tiếp §5 như bình thường.

> **DỪNG — GỬI OUTPUT CHECKPOINT 2.4.** Chỉ sang §3 khi cả 2.1–2.4 PASS.

---

## 3. Restore etcd từ snapshot

> Tài liệu Kubernetes yêu cầu: **dừng mọi API server → restore etcd → khởi động lại API server**,
> và khuyến nghị restart `kube-scheduler`, `kube-controller-manager`, `kubelet` sau restore để
> không thành phần nào giữ dữ liệu cũ. Với kubeadm, cả bốn thành phần control plane là static Pod
> do kubelet dựng từ `/etc/kubernetes/manifests/`. Cách dừng: **dời manifest ra khỏi thư mục** để
> kubelet không dựng lại Pod, **dừng kubelet**, rồi **dừng pod sandbox bằng `crictl`**. Không chờ
> kubelet tự gỡ Pod — §3.1 giải thích vì sao.

### 3.1. Đóng băng control plane

**Vì sao không chờ kubelet tự gỡ static Pod.** kubelet nhận Pod từ hai nguồn: thư mục manifest và
API server. Khi chưa thấy **mọi** nguồn gửi trạng thái đầu tiên, kubelet bỏ qua việc xóa Pod (hàm
`deletePod` trả `skipping delete because sources aren't ready yet`) và bỏ qua cả vòng dọn định kỳ
(housekeeping). Trong sự cố của runbook này, apiserver chết từ lúc node boot nên nguồn API không
bao giờ sẵn sàng. Dời manifest xong, container đang `Running` (lần chạy 01/10/2026:
`kube-controller-manager`, `kube-scheduler`) vẫn chạy mãi; chờ thêm hay restart kubelet đều không
đổi. Vì vậy §3.1 dừng kubelet trước rồi dừng sandbox bằng `crictl`. Cách này cũng đúng khi
apiserver còn sống (lặp lại §3.1 từ §3.4/§4.1). Nguồn:
[`pkg/kubelet/kubelet.go` nhánh `release-1.35`](https://github.com/kubernetes/kubernetes/blob/release-1.35/pkg/kubelet/kubelet.go)
— hàm `deletePod` và nhánh `housekeepingCh` trong `syncLoopIteration`.

**Khối 1 — dời manifest. Chỉ chạy một lần.** Chạy lại sẽ sinh `STAMP` mới, ghi đè
`~/etcd-restore-stamp.txt` và làm §3.2/§4.1 mất dấu thư mục đang giữ manifest. Nếu đã có
`/etc/kubernetes/manifests.off-*` từ lần chạy trước, bỏ khối 1, sang khối 2.

```bash
STAMP=$(date +%Y%m%d-%H%M%S)
echo "$STAMP" | tee ~/etcd-restore-stamp.txt
sudo mkdir -m 700 "/etc/kubernetes/manifests.off-$STAMP"
sudo mv /etc/kubernetes/manifests/*.yaml "/etc/kubernetes/manifests.off-$STAMP/"
sudo ls "/etc/kubernetes/manifests.off-$STAMP/"   # thư mục 700 của root, phải sudo mới đọc được
# PASS: đủ etcd.yaml, kube-apiserver.yaml, kube-controller-manager.yaml, kube-scheduler.yaml
ls /etc/kubernetes/manifests/
# PASS: không in gì — thư mục manifest đã rỗng
```

**Khối 2 — dừng kubelet rồi dừng sandbox.** Thứ tự là bắt buộc: dừng kubelet trước để sau bước
này không còn gì dựng lại hay restart container control plane.

```bash
sudo systemctl stop kubelet
sudo systemctl is-active kubelet
# PASS: inactive

# Dừng mọi pod sandbox của bốn static Pod, gồm cả sandbox NotReady của các lần crashloop trước.
# Dừng sandbox là dừng mọi container trong nó; sandbox được giữ lại, không xóa.
for p in $(sudo crictl pods -q --namespace kube-system \
    --name '^(etcd|kube-apiserver|kube-controller-manager|kube-scheduler)-k8s-master$'); do
  sudo crictl stopp "$p"
done
sudo crictl ps --name '^(etcd|kube-apiserver|kube-controller-manager|kube-scheduler)$'
# PASS: bảng chỉ có dòng tiêu đề
```

Số dòng `Stopped sandbox ...` có thể nhiều hơn 4 (lần chạy 01/10/2026: 6) vì gồm cả sandbox cũ;
dừng một sandbox đã dừng là vô hại. Tới §4.1, kubelet start lại với đủ manifest và dựng sandbox
mới cho cả bốn Pod, tức là `kube-controller-manager` và `kube-scheduler` cũng được restart — đúng
khuyến nghị ở đầu §3.

> Lưu `$STAMP`: các bước sau dùng lại để tìm đúng thư mục manifest và thư mục data cũ. Mở phiên SSH
> mới thì chạy `STAMP=$(cat ~/etcd-restore-stamp.txt)`.

> **DỪNG — GỬI OUTPUT CHECKPOINT 3.1.**

### 3.2. Giữ bằng chứng, giải phóng đường dẫn data

`etcdutl snapshot restore` tạo mới thư mục `--data-dir`; đường dẫn đó **không được tồn tại trước**.
Đổi tên thư mục hỏng thay vì xóa: giữ bằng chứng, và nếu restore thất bại thì §7 vẫn có thể cần
xem lại nó.

```bash
STAMP=$(cat ~/etcd-restore-stamp.txt)
sudo mv /var/lib/etcd "/var/lib/etcd.corrupt-$STAMP"
test ! -e /var/lib/etcd && echo 'PASS: /var/lib/etcd không còn'
sudo du -sh "/var/lib/etcd.corrupt-$STAMP"
df -h /var/lib
# PASS: thư mục corrupt cỡ ~410M; /var/lib còn trống nhiều hơn kích thước đó
```

> **DỪNG — GỬI OUTPUT CHECKPOINT 3.2.**

### 3.3. Chạy `etcdutl snapshot restore`

Tham số `--name`, `--initial-cluster`, `--initial-advertise-peer-urls` phải **trùng từng ký tự**
với manifest đã in ở §2.3, vì restore ghi membership mới vào datastore; sai một giá trị thì etcd
lên nhưng tự coi mình là member lạ. Không đặt `--initial-cluster-token`: manifest kubeadm không
đặt, nên cả hai bên dùng giá trị mặc định.

Hai flag cuối lấy từ tài liệu etcd:

- `--bump-revision 1000000000`: cộng thêm một khoảng vào revision của snapshot, mục tiêu là revision
  sau restore **lớn hơn revision cuối cùng mà client đã thấy trước sự cố**. Revision đó không đọc
  được từ `db` hỏng, nên phải ước lượng: etcd tăng revision mỗi lần ghi; tài liệu etcd lấy 10⁹ làm
  ví dụ với giả định snapshot một tuần tuổi ở 1500 ghi/giây. Snapshot 14/08 cách sự cố khoảng bảy
  tuần, nhưng cụm homelab này ghi ít hơn 1500 ghi/giây rất nhiều lần, nên 10⁹ vẫn dư. Nếu không
  ước lượng được (snapshot rất cũ hoặc cụm ghi nhiều), tăng lên `10000000000`; giá trị lớn hơn
  không gây hại trong giới hạn int64. **Không** được bỏ flag này với lý do "chắc đủ".
- `--mark-compacted`: đánh dấu mọi revision tới mốc mới là đã compact, buộc mọi watch cũ kết thúc
  và làm mất hiệu lực cache của các controller Kubernetes. Bắt buộc đi kèm khi `--bump-revision > 0`.

Mount `/var/lib` của host vào container để `--data-dir /var/lib/etcd` ghi thẳng ra host:

```bash
(
  set -e
  IMAGE='registry.k8s.io/etcd:3.6.6-0'
  SNAPSHOT=$(cat ~/etcd-restore-snapshot-path.txt)   # bản đã chốt ở §2.2, hoặc bản thứ hai do §3.4 ghi đè
  echo "restore từ: $SNAPSHOT"
  test -s "$SNAPSHOT"
  test ! -e /var/lib/etcd

  sudo ctr -n k8s.io run --rm \
    --mount "type=bind,src=$SNAPSHOT,dst=/snapshot.db,options=bind:ro" \
    --mount "type=bind,src=/var/lib,dst=/var/lib,options=rbind:rw" \
    "$IMAGE" "etcdutl-restore-$(date +%s)" \
    /usr/local/bin/etcdutl snapshot restore /snapshot.db \
      --name k8s-master \
      --initial-cluster k8s-master=https://192.168.100.111:2380 \
      --initial-advertise-peer-urls https://192.168.100.111:2380 \
      --data-dir /var/lib/etcd \
      --bump-revision 1000000000 \
      --mark-compacted

  sudo chmod 700 /var/lib/etcd
  sudo ls -la /var/lib/etcd /var/lib/etcd/member
  sudo test -s /var/lib/etcd/member/snap/db
  sudo test -d /var/lib/etcd/member/wal
)
RESTORE_RC=$?

if [ "$RESTORE_RC" -eq 0 ]; then
  echo 'PASS: data dir mới đã được tạo từ snapshot'
else
  echo 'FAIL: restore chưa đạt — xem lỗi phía trên, KHÔNG khởi động lại kubelet; sang §3.4' >&2
fi
( exit "$RESTORE_RC" )
```

PASS khi: `etcdutl` in log tới dòng hoàn tất (không `panic`, không `error`), thoát mã 0;
`/var/lib/etcd` thuộc `root:root` mode `700`, có `member/snap/db` khác rỗng và `member/wal/`.
Hash toàn vẹn nhúng trong snapshot được `etcdutl` kiểm trong lúc restore; file hỏng thì lệnh thoát
mã khác 0 và **có thể để lại data dir ghi dở** — §3.4 xử lý việc đó, không tự dọn tay.

> **STOP** nếu `RESTORE_RC` khác 0. Không sửa tay trong `/var/lib/etcd`, không chạy lại §3.2. Sang
> §3.4.

> **DỪNG — GỬI OUTPUT CHECKPOINT 3.3.** PASS thì bỏ qua §3.4, sang §4.

### 3.4. Thử lại restore khi §3.3 FAIL

Đọc lỗi trước, chọn nhánh sau. Lỗi `etcdutl` chia hai loại:

| Lỗi in ra | Loại | Việc phải làm |
| --- | --- | --- |
| `permission denied`, `no such file or directory`, lỗi `mount`/`ctr`, `data-dir ... exists`, `test` ở đầu block thoát mã 1 (thiếu file snapshot) | **Thao tác** | Sửa đường dẫn/quyền/mount rồi chạy lại. Không phải lỗi snapshot; không đổi bản snapshot, không sang §7 |
| `expected sha256 <a>, got <b>` (checksum nhúng không khớp — file bị sửa hoặc copy dở) | **Nội dung snapshot** | Thử bản thứ hai (từ host Windows) theo bước 2 dưới đây |
| `snapshot missing hash but --skip-hash-check=false` (file không phải snapshot do `etcdctl snapshot save` tạo, ví dụ copy thẳng từ `member/snap/db`) | **Nội dung snapshot** | Không thêm `--skip-hash-check`. Kiểm lại xem file chốt có trỏ đúng `etcd-snapshot.db` của backup không; nếu đúng mà vẫn báo vậy, thử bản thứ hai |
| `panic` khi đọc `/snapshot.db`, `unexpected EOF`, lỗi đọc file dở chừng | **Nội dung snapshot** | Thử bản thứ hai theo bước 2 |

Thông báo lỗi ở đây là của `etcdutl` 3.6.6; khi output thật không khớp dòng nào, gửi nguyên văn
dòng lỗi rồi mới chọn nhánh, không đoán.

**Bước 1 — dọn data dir ghi dở, giữ lại làm bằng chứng.** `etcdutl` tạo file trước khi kiểm hash,
nên lần FAIL có thể để lại `/var/lib/etcd` chưa hoàn chỉnh. Không dùng lại `mv` của §3.2: thư mục
`etcd.corrupt-$STAMP` đã tồn tại, `mv` sẽ nhét data dir dở vào **trong** nó.

```bash
FAIL_STAMP=$(date +%Y%m%d-%H%M%S)
if sudo test -e /var/lib/etcd; then
  sudo mv /var/lib/etcd "/var/lib/etcd.failed-$FAIL_STAMP"
  echo "đã dời data dir dở sang /var/lib/etcd.failed-$FAIL_STAMP"
fi
test ! -e /var/lib/etcd && echo 'PASS: đường dẫn data trống, sẵn sàng thử lại'
sudo ls -d /var/lib/etcd.* 
# PASS: thấy etcd.corrupt-<STAMP> (từ §3.2) và, nếu có, etcd.failed-<FAIL_STAMP>; không lồng nhau
```

**Bước 2 — chỉ khi lỗi thuộc loại "nội dung snapshot": lấy bản thứ hai từ host.** Trên **host
Windows**, copy archive lên master:

Thay `<STAMP>` bằng đúng STAMP đã chọn ở §2.2 (ví dụ `20260814-190914`):

```powershell
$Stamp = '<STAMP>'
scp "E:\courses\Ansible\ansible_playbook_k8s-installation\k8s-backups\$Stamp.tar.gz" "ubuntu@192.168.100.111:k8s-backups/from-host-$Stamp.tar.gz"
```

Trên **`k8s-master`**, giải nén vào thư mục riêng, so hash với giá trị ghi lúc backup, rồi **ghi
đè** file chốt đường dẫn để §3.3 dùng bản này:

```bash
STAMP=$(basename "$(dirname "$(cat ~/etcd-restore-snapshot-path.txt)")")   # STAMP của bản đã chọn ở §2.2
echo "STAMP: $STAMP"
mkdir -m 700 -p ~/k8s-backups/from-host
sha256sum ~/k8s-backups/from-host-$STAMP.tar.gz
tar -C ~/k8s-backups/from-host -xzf ~/k8s-backups/from-host-$STAMP.tar.gz
ls -la ~/k8s-backups/from-host/$STAMP/
# PASS: hash trùng giá trị ghi ở bước 1 §14.0.1; có etcd-snapshot.db khác rỗng

echo "$HOME/k8s-backups/from-host/$STAMP/etcd-snapshot.db" > ~/etcd-restore-snapshot-path.txt
cat ~/etcd-restore-snapshot-path.txt
# PASS: in đường dẫn có "from-host" — §3.3 sẽ đọc file này, không cần sửa gì trong §3.3
```

Nếu lỗi thuộc loại **thao tác**, bỏ qua bước 2; file chốt vẫn trỏ bản gốc.

**Bước 3 — chạy lại §3.3 nguyên văn.** §3.3 đọc `SNAPSHOT` từ file chốt và in ra dòng
`restore từ: ...` — đối chiếu dòng đó với bản đã định thử trước khi đọc tiếp output.

**Bước 4 — chỉ khi §3.3 PASS nhưng etcd `panic` ở §4.1: tách lỗi snapshot khỏi lỗi lưu trữ.**
Restore thành công có nghĩa bbolt đã ghi và đọc lại được file ngay lúc đó; etcd panic sau đó có thể
do snapshot, nhưng cũng có thể do mount, quyền, hoặc đĩa của `/var/lib`. Restore lại vào một thư
mục khác **trên cùng đĩa** không tách được hai khả năng này: đĩa lỗi thì cả hai lần đều hỏng dù
snapshot tốt. Vì vậy kiểm chứng trên **tmpfs** (`/dev/shm`, nằm trong RAM, không đi qua virtual
disk): restore cùng bản vào đó rồi cho chính `etcdutl` mở `db` vừa tạo.

```bash
findmnt -n -o FSTYPE -T /dev/shm; df -h /dev/shm
# PASS: tmpfs; còn trống nhiều hơn 10 lần kích thước snapshot (WAL/db restore ra chỉ vài chục MB)

(
  set -e
  IMAGE='registry.k8s.io/etcd:3.6.6-0'
  SNAPSHOT=$(cat ~/etcd-restore-snapshot-path.txt)
  VERIFY="/dev/shm/etcd-verify-$(date +%Y%m%d-%H%M%S)"
  sudo ctr -n k8s.io run --rm \
    --mount "type=bind,src=$SNAPSHOT,dst=/snapshot.db,options=bind:ro" \
    --mount "type=bind,src=/dev/shm,dst=/dev/shm,options=rbind:rw" \
    "$IMAGE" "etcdutl-verify-$(date +%s)" \
    /usr/local/bin/etcdutl snapshot restore /snapshot.db \
      --name k8s-master \
      --initial-cluster k8s-master=https://192.168.100.111:2380 \
      --initial-advertise-peer-urls https://192.168.100.111:2380 \
      --data-dir "$VERIFY"
  sudo ctr -n k8s.io run --rm \
    --mount "type=bind,src=$VERIFY/member/snap/db,dst=/verify.db,options=bind:ro" \
    "$IMAGE" "etcdutl-verify-status-$(date +%s)" \
    /usr/local/bin/etcdutl snapshot status /verify.db -w table
  echo "VERIFY_DIR=$VERIFY"
)
VERIFY_RC=$?

# Tình trạng đĩa thật của /var/lib — để đối chiếu
sudo dmesg | grep -iE 'I/O error|ext4|remount' | tail -5
findmnt -n -o SOURCE,FSTYPE,OPTIONS -T /var/lib
sudo ls -ld /var/lib/etcd /var/lib/etcd/member /var/lib/etcd/member/snap
```

Đọc kết quả:

- `VERIFY_RC = 0` và `snapshot status` in bảng trên tmpfs → snapshot **tốt**. etcd panic trên
  `/var/lib/etcd` là lỗi của **tầng lưu trữ đích** (đĩa, filesystem, mount, quyền) →
  **STOP, không reset**: reset dựng lại cụm lên chính cái đĩa đang lỗi. Sửa storage trước (`fsck`
  ở lần boot kế, đổi virtual disk, hoặc chuyển `/var/lib/etcd` sang đĩa khác và sửa `hostPath`
  trong manifest etcd), rồi lặp lại §3.1–§4.1. Dọn thư mục kiểm chứng:
  `sudo rm -rf "$VERIFY"` (thay bằng giá trị đã in; tmpfs cũng tự mất khi reboot).
- `etcdutl` panic hoặc lỗi ngay khi `snapshot status` trên `db` vừa restore **trên tmpfs** → lặp
  lại bước 4 với bản thứ hai (bước 2). Cả hai bản đều vậy trên tmpfs → snapshot không dùng được,
  đủ điều kiện 2 của §7.1.
- `VERIFY_RC ≠ 0` vì `/dev/shm` không phải tmpfs hoặc thiếu chỗ → chưa kết luận gì; dùng một
  storage độc lập khác đã xác nhận khỏe (đĩa virtual thứ hai gắn thêm, hoặc chép snapshot sang
  worker và chạy cùng block ở đó với image etcd có sẵn trên worker) rồi mới đọc kết quả.

> **DỪNG — GỬI OUTPUT CHECKPOINT 3.4** gồm: dòng lỗi nguyên văn của lần FAIL, loại lỗi đã chọn,
> output bước 1, và (nếu có) hash bước 2. Chỉ khi **cả hai bản** có hash khớp mà `etcdutl` vẫn báo
> lỗi nội dung snapshot thì mới đủ điều kiện xét §7.

---

## 4. Đưa cụm về trạng thái khỏe sau restore

### 4.1. Khởi động lại control plane

Trả manifest về chỗ cũ rồi bật kubelet. Thứ tự này bảo đảm khi kubelet lên, nó thấy đủ bốn
manifest và dựng cả bốn Pod trong một lượt.

Thư mục giữ manifest thuộc `root` mode `700`, nên wildcard phải được **root** mở rộng: nếu viết
`sudo mv .../*.yaml`, Bash của user `ubuntu` mở rộng `*` trước khi `sudo` chạy, không đọc được thư
mục và `mv` nhận nguyên chuỗi `*.yaml`. Vì vậy dùng `sudo sh -c`.

```bash
STAMP=$(cat ~/etcd-restore-stamp.txt)
sudo sh -c "mv /etc/kubernetes/manifests.off-$STAMP/*.yaml /etc/kubernetes/manifests/"
ls /etc/kubernetes/manifests/
# PASS: đủ 4 file
sudo rmdir "/etc/kubernetes/manifests.off-$STAMP"

sudo systemctl start kubelet
sudo systemctl is-active kubelet
# PASS: active

# Đợi etcd và kube-apiserver lên, tối đa 36 lần x 5 giây
for i in $(seq 1 36); do
  UP=$(sudo crictl ps -q --name '^(etcd|kube-apiserver)$' | wc -l)
  [ "$UP" -eq 2 ] && break
  echo "Đợi etcd/apiserver: $UP/2, lần $i/36"; sleep 5
done
sudo crictl ps --name '^(etcd|kube-apiserver|kube-controller-manager|kube-scheduler)$'
sleep 60
sudo crictl ps --name '^(etcd|kube-apiserver|kube-controller-manager|kube-scheduler)$'
# PASS: cả hai lần đủ 4 container Running và cột ATTEMPT của từng container KHÔNG tăng giữa
# hai lần. ATTEMPT khác 0 ở lần đầu là chấp nhận được (apiserver có thể restart một lần trong
# lúc đợi etcd); ATTEMPT tăng liên tục mới là crashloop.
```

Quyết định:

```bash
sudo crictl ps -a --name '^etcd$' | head -3
```

- etcd `Running`, ATTEMPT ổn định giữa hai lần xem → sang §4.2 (gate sức khỏe thật nằm ở đó).
- etcd lại `Exited` với ATTEMPT tăng → xem log bằng lệnh ở §2.1. Nếu log lại `panic` bbolt trên
  data dir mới → **STOP**, quay về §3.1 đóng băng lại rồi làm **§3.4 bước 4** (kiểm chứng snapshot
  trên tmpfs). Bước 4 quyết định: snapshot tốt thì lỗi là storage của `/var/lib` → sửa storage,
  không reset; snapshot mở không được trên tmpfs với **cả hai bản** mới đủ điều kiện 2 của §7.1.
  Lỗi khác `panic` (cert, flag, cổng) là lỗi cấu hình → §9.

> **DỪNG — GỬI OUTPUT CHECKPOINT 4.1.**

### 4.2. Gate control plane và chứng minh dữ liệu là dữ liệu restore

```bash
kubectl get --raw='/readyz?verbose' | tail -3
# PASS: dòng cuối "readyz check passed"

kubectl -n kube-system exec etcd-k8s-master -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key endpoint health
# PASS: "is healthy"

kubectl -n kube-system exec etcd-k8s-master -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key endpoint status -w json \
  | grep -o '"revision":[0-9]*'
# PASS: revision >= REVISION của §2.3 + 1000000000 (với snapshot 14/08: >= 1002404659)

kubectl get ns
# PASS: KHÔNG có cert-manager, cattle-system, three-tier — đúng trạng thái 14/08
kubectl -n cloudflare get deploy,secret
# PASS: có deployment.apps/cloudflared và secret/tunnel-token — khớp kết quả grep ở §2.4
kubectl get nodes -o wide
# PASS: 3 node xuất hiện; Ready có thể chưa True cho tới §4.3
```

Dòng revision là bằng chứng quyết định: giá trị lớn hơn 10⁹ chỉ có thể đến từ `--bump-revision`,
tức là apiserver đang đọc data dir vừa restore, không phải dữ liệu nào khác.

Hai lệnh `get ns` và `get deploy,secret` là nơi **kết luận thật** về nội dung snapshot; §2.4 chỉ
là dự đoán. Đọc theo nhánh đã chọn ở §2.4:

| §2.4 dự đoán | Kết quả ở đây | Kết luận và việc tiếp theo |
| --- | --- | --- |
| có cloudflared, không có Rancher | không có `cattle-system`; có `deploy/cloudflared` + `secret/tunnel-token` | Nhánh chuẩn. §5 chạy như bảng cài mới |
| thiếu cloudflared | `kubectl -n cloudflare ...` trả `No resources found` hoặc namespace không có | Chạy [§12.2 Phase 1](runbook-k8s-vmware.md#122-deploy-cloudflared-vào-cụm) với token tunnel cũ ngay sau checkpoint 4.3, trước §4.4 |
| nghi có Rancher | có `cattle-system` (và thường có `cert-manager`) | §5 đi nhánh **repair** (bảng thứ hai của §5), không `helm install` |
| bất kỳ | kết quả **khác** dự đoán | Không phải lỗi: dự đoán từ `grep` có giới hạn. Ghi nhận kết quả thật, chọn nhánh theo kết quả thật, gửi output |

PASS của §4.2 vì vậy **không** đòi "không có `cattle-system`" một cách vô điều kiện; nó đòi kết quả
đã được đọc và nhánh cho §4.3–§5 đã được chốt theo kết quả đó.

> **DỪNG — GỬI OUTPUT CHECKPOINT 4.2.**

### 4.3. Restart kubelet trên hai worker và chờ cụm hội tụ

Theo khuyến nghị của tài liệu Kubernetes, restart kubelet ở mọi node để không kubelet nào giữ
cache cũ. Kubelet worker đã mất apiserver nhiều giờ; restart cũng làm nó đăng ký lại sạch.

Trên **`k8s-worker1`**, rồi **`k8s-worker2`**:

```bash
sudo systemctl restart kubelet
sudo systemctl is-active kubelet
# PASS: active
```

Trở lại **`k8s-master`**:

```bash
kubectl wait --for=condition=Ready node --all --timeout=300s
kubectl get nodes -o wide
# PASS: 3 node Ready, VERSION giống nhau

kubectl -n kube-flannel rollout status daemonset/kube-flannel-ds --timeout=300s
kubectl -n kube-system rollout status deployment/coredns --timeout=300s
kubectl -n traefik rollout status deploy/traefik --timeout=300s
# cloudflared chỉ có nếu snapshot chụp sau §12 (nhánh §2.4); thiếu thì không phải lỗi ở đây
if kubectl -n cloudflare get deploy cloudflared >/dev/null 2>&1; then
  kubectl -n cloudflare rollout status deploy/cloudflared --timeout=300s
else
  echo 'INFO: chưa có deploy/cloudflared — nhánh "thiếu cloudflared" của §2.4: chạy §12.2 Phase 1 ngay sau checkpoint 4.3, trước §4.4'
fi
kubectl -n local-path-storage rollout status deploy/local-path-provisioner --timeout=300s
kubectl -n kube-system rollout status deploy/metrics-server --timeout=300s

kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
# PASS: No resources found
kubectl get pods -A -o wide | awk 'NR>1 && $4!="Running" && $4!="Completed"'
# PASS: không in dòng nào. Pod "Completed" (Job đã xong, phase Succeeded) là trạng thái hợp lệ,
# không phải lỗi; chỉ Pending/CrashLoopBackOff/Error/ContainerCreating kéo dài mới FAIL.
```

Trong lúc chờ, kubelet trên worker sẽ thấy các Pod từng chạy sau 14/08 (Rancher, cert-manager,
Phase 2) **không còn trong API** và tự dọn sandbox/container của chúng; không phải làm tay. Event
`FailedCreatePodSandbox ... /run/flannel/subnet.env` trong vài phút đầu là race khởi động đã
mô tả ở [§15 Phase 1](runbook-k8s-vmware.md#rancher-đăng-nhập-404-tại-v1extcattleioselfuser-sau-khi-reboot-node);
chỉ điều tra nếu Flannel không rollout xong.

> **DỪNG — GỬI OUTPUT CHECKPOINT 4.3.**

### 4.4. Gate chức năng — chạy lại các tầng của Phase 1 §8

Chạy nguyên văn các tầng sau của [§8 Phase 1](runbook-k8s-vmware.md#8-verify-cụm), gửi output
từng tầng: **8.1** (control plane), **8.2** (node, taint, podCIDR), **8.3** (Pod networking
cross-node — tầng hay bị bỏ qua nhất), **8.7** gate nhanh (Helm, `local-path`, metrics), rồi **8.9**
dọn resource test. Sau đó kiểm tra đường Internet đã có từ trước:

```bash
kubectl get ingress -A
kubectl -n cloudflare logs -l app=cloudflared --tail=20 --prefix | grep -iE 'registered|connected|error'
curl -sS -o /dev/null -w 'app=%{http_code}\n' https://app.hieupn.site/
# PASS: log cloudflared có "Registered tunnel connection", không error lặp; app trả 200
```

PASS khi mọi tầng của §8 PASS và app mẫu trả `200` qua Internet. Tunnel lên lại được vì Secret
token nằm trong snapshot 14/08 và tunnel phía Cloudflare chưa bao giờ bị xóa.

> **DỪNG — GỬI OUTPUT CHECKPOINT 4.4.** Tới đây cụm đã **khỏe ở mức 14/08**; các mục sau chỉ cài
> lại phần đã mất.

### 4.5. Kiểm kê dữ liệu MongoDB còn trên đĩa worker — tùy chọn, không phải gate

Restore etcd xóa **object** PV/PVC tạo sau snapshot, nhưng `local-path` ghi dữ liệu thẳng vào
`/opt/local-path-provisioner/` trên node và không có gì trong quy trình này đụng tới thư mục đó.
Với snapshot 14/08, dữ liệu MongoDB của Phase 2 vì vậy **vẫn còn trên đĩa worker** mà không có PV
nào tham chiếu; provisioner sẽ không bao giờ dọn nó, và PVC mới ở §6 sẽ được cấp thư mục **mới**.

**Bước 0 — bắt buộc trước khi đụng bất kỳ thư mục nào: xác định thư mục nào còn được PV tham
chiếu.** Nếu §2.2 chọn snapshot **mới hơn** đã chứa Phase 2, PV/PVC của MongoDB được restore và
đang trỏ vào chính thư mục đó; đổi tên hay xóa lúc này là phá volume đang dùng. Trên
**`k8s-master`**:

```bash
kubectl get pv -o custom-columns='PV:.metadata.name,NS:.spec.claimRef.namespace,PVC:.spec.claimRef.name,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0],HOSTPATH:.spec.hostPath.path,LOCAL:.spec.local.path'
kubectl -n three-tier get pvc 2>/dev/null || echo 'INFO: không có namespace three-tier'
```

- **Không có PV nào** với `NS=three-tier` → mọi thư mục `*_three-tier_*` trên worker đều mồ côi;
  tiếp tục bước kiểm kê bên dưới.
- **Có PV** với `NS=three-tier` → snapshot đã chứa Phase 2. Thư mục ghi ở cột `HOSTPATH`/`LOCAL`
  trên node ở cột `NODE` **đang được dùng — không đổi tên, không xóa**. Bỏ qua phần còn lại của
  §4.5; §6 đi nhánh resume. Chỉ thư mục `*_three-tier_*` **không** xuất hiện ở cột đường dẫn mới là
  mồ côi.

**Kiểm kê** trên **từng worker** (chỉ đọc), đối chiếu từng thư mục với output bước 0:

```bash
sudo find /opt/local-path-provisioner -maxdepth 1 -mindepth 1 -type d -exec du -sh {} + 2>/dev/null
# Thư mục của MongoDB Phase 2 có tên dạng pvc-<uuid>_three-tier_data-mongodb-0, chỉ nằm trên MỘT
# worker (worker nào từng chạy mongodb-0). Ghi lại tên đầy đủ và dung lượng.
```

**Quyết định** — chọn một, ghi rõ vào checkpoint:

- **Giữ** (mặc định nếu chưa chắc): đổi tên để không nhầm với PVC mới, không xóa.

  ```bash
  ORPHAN=$(sudo find /opt/local-path-provisioner -maxdepth 1 -type d -name 'pvc-*_three-tier_data-mongodb-0' | head -1)
  test -n "$ORPHAN" && sudo mv "$ORPHAN" "/opt/local-path-provisioner.orphan-$(date +%Y%m%d)-mongodb-0"
  sudo ls -d /opt/local-path-provisioner.orphan-* 2>/dev/null
  ```

  Cách lấy lại dữ liệu từ thư mục này (copy file vào thư mục PVC mới khi Pod đang dừng, hoặc chạy
  một `mongod` tạm trỏ vào nó rồi `mongodump`) nằm ngoài phạm vi runbook; chỉ làm khi thật sự
  cần dữ liệu đó và có kế hoạch riêng.

- **Bỏ**: chỉ khi xác nhận đó là dữ liệu test không cần giữ. Xóa **đúng một thư mục theo tên đã
  kiểm kê**, không dùng wildcard:

  ```bash
  sudo rm -rf '/opt/local-path-provisioner/pvc-<uuid-đã-kiểm-kê>_three-tier_data-mongodb-0'
  ```

> **DỪNG — GỬI OUTPUT CHECKPOINT 4.5** gồm kết quả kiểm kê của cả hai worker và quyết định
> giữ/bỏ. Không có điều kiện PASS/FAIL; bước này chỉ để quyết định có ý thức trước khi §6 tạo PVC
> mới.

---

## 5. Cài lại cert-manager và Rancher

Toàn bộ §14 Phase 1 được chạy lại **nguyên văn**, vì cụm đang ở đúng trạng thái mà §14 giả định
(chưa có cert-manager, chưa có Rancher). Khác biệt so với lần đầu chỉ nằm ở bảng dưới.

| Mục Phase 1 | Chạy lại? | Khác biệt |
| --- | --- | --- |
| [§14.0 gate read-only](runbook-k8s-vmware.md#140-gate-trước-khi-thay-đổi-cluster) | Có | Hai lệnh cuối phải xác nhận **không** còn release/namespace cũ. Nếu `cattle-system` hoặc `cert-manager` xuất hiện thì bảng này không áp dụng: sang **bảng repair** bên dưới (nhánh "có Rancher" đã chốt ở §4.2). |
| [§14.0.1 backup](runbook-k8s-vmware.md#1401-backup-etcd-và-cấu-hình-trước-thay-đổi-lớn) | **Có, bắt buộc** | Tạo backup mới ngay sau restore. Đây vừa là mốc lui cho §14.1 trở đi, vừa chứng minh etcd sau restore snapshot được. Copy ra host và so hash đủ hai bước. |
| [§14.1 cert-manager](runbook-k8s-vmware.md#141-cài-cert-manager-rancher-cần-để-cấp-tls-nội-bộ) | Có | Không đổi. |
| [§14.2 split DNS](runbook-k8s-vmware.md#142-cấu-hình-split-dns-cho-client-trong-cụm-local) | Có | ConfigMap CoreDNS 14/08 chưa có entry. Lấy lại `TRAEFIK_IP` bằng lệnh trong mục; Service Traefik được restore nguyên object nên ClusterIP thường không đổi, nhưng **không được giả định**. |
| [§14.3 Rancher](runbook-k8s-vmware.md#143-cài-rancher-helm-pin-2143) | Có | File `rancher-values.yaml` cũ trên master vẫn dùng được; render gate và `helm install` chạy lại nguyên văn. |
| [§14.4 gate origin](runbook-k8s-vmware.md#144-gate-origin-https-nội-bộ-trước-khi-publish) | Có | Không đổi. |
| [§14.5 Access + route](runbook-k8s-vmware.md#145-bảo-vệ-rancher-bằng-cloudflare-access-rồi-publish-qua-tunnel) | **Chỉ verify** | Access application, route `rancher.hieupn.site` và ba origin parameter ở §14.5.1 nằm ở Cloudflare, không mất. Mở lại route để xác nhận đủ `noTLSVerify: true`, `httpHostHeader`, `originServerName`; chạy `nslookup` và `curl -I` của mục. **Không tạo lại** application hay route. |
| [§14.6 đăng nhập](runbook-k8s-vmware.md#146-đăng-nhập-lần-đầu) | Có | Bootstrap password **mới**; mật khẩu admin cũ không còn. Dùng cửa sổ Incognito để không dính session cũ. Server URL vẫn `https://rancher.hieupn.site`. |
| [§14.7 gate hoàn thành](runbook-k8s-vmware.md#147-gate-hoàn-thành) | Có | Không đổi. |

**Bảng repair — chỉ dùng khi §4.2 chốt nhánh "có Rancher"** (snapshot chụp sau khi cài Rancher).
Khi đó cert-manager và Rancher đã nằm trong dữ liệu restore; cài lại là sai. Việc cần làm là xác
nhận chúng chạy lại được, rồi đi qua các gate verify:

| Mục | Làm gì |
| --- | --- |
| Kiểm release | `helm list -A` phải có `cert-manager` và `rancher` ở trạng thái `deployed`. Không có release nhưng có namespace → dừng, gửi output; không tự `helm install` đè. |
| Rollout | `kubectl -n cert-manager rollout status deploy --timeout=300s` từng Deployment; `kubectl -n cattle-system rollout status deploy/rancher --timeout=10m`. |
| [§14.0.1 backup](runbook-k8s-vmware.md#1401-backup-etcd-và-cấu-hình-trước-thay-đổi-lớn) | **Bắt buộc**, sau khi rollout PASS: backup mới sau restore. |
| [§14.2](runbook-k8s-vmware.md#142-cấu-hình-split-dns-cho-client-trong-cụm-local) | Chỉ **verify** bằng `nslookup` trong mục; entry CoreDNS đã có trong snapshot. Sửa chỉ khi ClusterIP Traefik đổi. |
| [§14.4](runbook-k8s-vmware.md#144-gate-origin-https-nội-bộ-trước-khi-publish), [§14.5](runbook-k8s-vmware.md#145-bảo-vệ-rancher-bằng-cloudflare-access-rồi-publish-qua-tunnel) | Chạy phần verify nguyên văn. |
| Đăng nhập | Dùng **mật khẩu admin đã đặt** trước thời điểm snapshot; `bootstrap-secret` có thể không còn, đó là bình thường. Login `404 selfuser` sau restore → [§15 Phase 1](runbook-k8s-vmware.md#rancher-đăng-nhập-404-tại-v1extcattleioselfuser-sau-khi-reboot-node). |
| [§14.7](runbook-k8s-vmware.md#147-gate-hoàn-thành) | Chạy nguyên văn. |

> **DỪNG — GỬI OUTPUT** của §14.0.1 (verdict backup + hash), §14.2 (nslookup trả ClusterIP Traefik),
> §14.4 (`HTTP 200`), §14.7 (`server-url`, `agent-tls-mode`, UI `local` Active). Khi §14.7 PASS,
> mục tiêu tối thiểu của runbook — **Rancher tạo lại và connect thành công** — đã đạt.

---

## 6. Làm lại Phase 2 tới bước đang dở

Trước sự cố, Phase 2 đang ở [§10 Triển khai frontend React](runbook-k8s-vmware-phase2.md#10-triển-khai-frontend-react).
Mọi thứ **ngoài etcd** vẫn còn: source trong `E:\courses\Ansible\three-tier-crud`, image đã push
với digest, manifest `k8s/*.yaml` trong repo. Với snapshot 14/08, mọi thứ **trong etcd** mất:
namespace, Secret, MongoDB + PVC, backend, frontend.

**Chọn nhánh trước theo bước 0 của §4.5:**

- **Không có** PV/namespace `three-tier` sau restore → nhánh **cài lại**, bảng dưới đây.
- **Có** namespace `three-tier` (snapshot mới hơn đã chứa Phase 2) → nhánh **resume**: preflight
  §7.1 của Phase 2 sẽ trả `STOP: namespace three-tier already exists` — đó là kết quả đúng cho
  nhánh này, không phải lỗi. Làm theo chỉ dẫn của chính mục đó: gửi danh sách resource nó in ra,
  **không** chạy lại các bước tạo namespace/Secret, **không** xóa PVC, rồi resume từ checkpoint đầu
  tiên chưa PASS (chạy lại phần verify của §8.2, §9.3, §10.2 theo thứ tự; bước nào PASS thì đi
  tiếp). Credential trong PVC là credential cũ; Secret cũ cũng được restore cùng, nên chúng khớp
  nhau. Bảng dưới đây **không** áp dụng cho nhánh resume.

| Mục Phase 2 (nhánh cài lại) | Chạy lại? | Khác biệt |
| --- | --- | --- |
| [§3 gate đầu vào](runbook-k8s-vmware-phase2.md#3-gate-đầu-vào-từ-phase-1) | Có, đủ 3.1–3.3 | Cụm vừa restore; gate này là bằng chứng độc lập rằng nó đủ tài nguyên. Image MongoDB thường còn trong containerd nên 3.3 nhanh. |
| [§4](runbook-k8s-vmware-phase2.md#4-chốt-biến-và-contract-triển-khai), [§5](runbook-k8s-vmware-phase2.md#5-gate-tạo-source-code--prompt-dùng-ở-bước-sau), [§6](runbook-k8s-vmware-phase2.md#6-build-và-push-image) | **Không** | Source đã commit, image đã push và pin digest. Chỉ chạy lại §6.4 phần *verify digest* nếu muốn chắc image còn trên Docker Hub. |
| [§7 namespace, ConfigMap, Secret](runbook-k8s-vmware-phase2.md#7-tạo-namespace-configmap-và-secret) | Có | Preflight §7.1 phải trả `PASS: fresh install`. Secret tạo **mật khẩu mới** — PVC cũ đã mất nên không có credential cũ nào để khớp. |
| [§8 MongoDB](runbook-k8s-vmware-phase2.md#8-triển-khai-mongodb) | Có | PVC mới `Bound` vào thư mục **mới**, bắt đầu rỗng. Dữ liệu cũ (nếu §4.5 chọn giữ) nằm ở thư mục đã đổi tên trên worker và không được PVC mới dùng lại; muốn đưa vào phải làm riêng, ngoài runbook. |
| [§9 backend](runbook-k8s-vmware-phase2.md#9-triển-khai-backend-fastapi) | Có | Không đổi; manifest vẫn trỏ digest đã pin. |
| [§10 frontend](runbook-k8s-vmware-phase2.md#10-triển-khai-frontend-react) | Có | Đây là bước đang dở trước sự cố. Từ đây trở đi tiếp tục Phase 2 bình thường: §11, §12, §13… |

> **DỪNG — GỬI OUTPUT** của checkpoint 7.1, 8.2, 9.3 và 10.2 theo đúng khuôn Phase 2. Khi 10.2
> PASS, cụm đã **trở lại đúng điểm trước sự cố**.

---

## 7. Đường lui khi restore thất bại — reset toàn cụm

### 7.1. Khi nào mới được vào mục này

Reset xóa CA của cụm và không lùi được. Chỉ vào mục này khi đã **chứng minh snapshot không dùng
được**, tức là một trong hai điều sau, kèm output đã gửi ở §3.4:

1. `etcdutl snapshot restore` báo lỗi thuộc loại **nội dung snapshot** (bảng §3.4) với **cả hai
   bản** — bản trên master và bản từ host — trong khi hash SHA-256 của mỗi bản đã được xác nhận
   khớp giá trị ghi lúc backup. Lỗi thao tác (quyền, mount, đường dẫn, `data-dir exists`) dù xảy
   ra với cả hai bản cũng **không** tính; sửa thao tác và chạy lại §3.4.
2. Restore thoát mã 0 nhưng etcd khởi động bằng data dir mới lại `panic` bbolt, **và** bước 4 của
   §3.4 đã chứng minh trên **storage độc lập với đĩa của `/var/lib`** (tmpfs, hoặc storage khác
   đã xác nhận khỏe) rằng chính `etcdutl` cũng không mở được `db` restore ra từ **cả hai bản**
   snapshot. Hai bản trên master và host là hai copy của **cùng một** snapshot, chỉ loại trừ lỗi
   copy; kiểm chứng trên cùng đĩa `/var/lib` không loại trừ được lỗi lưu trữ, nên không được tính.
   Nếu bước 4 PASS trên tmpfs mà etcd vẫn panic trên `/var/lib/etcd` thì lỗi là storage đích →
   **STOP theo bước 4**, sửa storage, không reset: reset dựng cụm mới lên đúng cái đĩa đang lỗi.

Các dấu hiệu sau **không** phải lý do reset; chúng là việc chẩn đoán, xem §9:

- `readyz` chưa PASS, apiserver `TLS handshake timeout`, revision nhỏ hơn 10⁹ — thường do thiếu
  flag, sai `--data-dir`, hoặc chờ chưa đủ; lặp lại §3.1–§4.1 sau khi tìm ra nguyên nhân cụ thể.
- Gate §4.3 chậm, worker `NotReady` một lúc, event `subnet.env` — race khởi động.

Trước khi chạy §7.3, gửi nguyên văn dòng lỗi của `etcdutl` hoặc `panic` của etcd cho cả hai bản
để đối chiếu lần cuối.

### 7.2. Reset xóa gì, giữ gì

`kubeadm reset` trên master xóa `/etc/kubernetes/manifests`, `/etc/kubernetes/pki` (toàn bộ CA),
các file `*.conf`, `/var/lib/kubelet`, `/var/lib/etcd`. CA mới nghĩa là cert kubelet của hai worker
không còn được tin → **hai worker cũng phải reset và join lại**. Không có cách giữ worker.

Theo tài liệu `kubeadm reset`, lệnh này **không** dọn: `/etc/cni/net.d`, rule iptables/nftables/IPVS
của kube-proxy, `$HOME/.kube`. Ngoài ra Flannel để lại interface `cni0` và `flannel.1` mang IP của
podCIDR cũ; init lại có thể cấp podCIDR khác cho node và làm §8.3 fail. Cách dọn sạch và không đoán
là **reboot node sau reset**: rule iptables và hai interface đó không persistent.

| Còn nguyên sau reset | Phải làm lại (Phase 1) |
| --- | --- |
| OS, containerd, kubelet, kubeadm, Helm binary, image trong containerd | [§6](runbook-k8s-vmware.md#6-khởi-tạo-control-plane-chỉ-master) init + Flannel, [§7](runbook-k8s-vmware.md#7-join-worker-chỉ-2-worker) join |
| Domain, tunnel, route, Access ở Cloudflare | [§8](runbook-k8s-vmware.md#8-verify-cụm) verify + [§8.7](runbook-k8s-vmware.md#87-tầng-7--công-cụ-và-add-on-kubeadm-không-cài-sẵn) add-on |
| Backup `~/k8s-backups`, source, image Phase 2 | [§9](runbook-k8s-vmware.md#9-cài-ingress-controller-traefik), [§10](runbook-k8s-vmware.md#10-deploy-app-mẫu--ingress), [§12.2](runbook-k8s-vmware.md#122-deploy-cloudflared-vào-cụm)–[§13](runbook-k8s-vmware.md#13-trỏ-domain--kiểm-tra-trên-internet), toàn bộ [§14](runbook-k8s-vmware.md#14-cài-rancher-2143--quản-lý-cụm), rồi Phase 2 từ §3 |

### 7.3. Giữ bằng chứng trước khi reset

Trên **`k8s-master`**:

```bash
STAMP=$(cat ~/etcd-restore-stamp.txt 2>/dev/null || date +%Y%m%d-%H%M%S)
mkdir -p ~/reset-evidence
sudo tar -C /etc -czf ~/reset-evidence/etc-kubernetes-before-reset-$STAMP.tar.gz kubernetes
sudo chown "$(id -u):$(id -g)" ~/reset-evidence/*.tar.gz
ls -la ~/reset-evidence/ /var/lib/etcd.corrupt-* 2>/dev/null
# PASS: có archive; thư mục etcd.corrupt-* vẫn còn (kubeadm reset không đụng tới nó)
```

`kubeadm reset` xóa `/etc/kubernetes` nhưng **không** xóa `/var/lib/etcd.corrupt-*` vì đó không
phải đường dẫn nó quản lý. Giữ lại cho tới khi cụm mới PASS §14.7, rồi xóa để lấy lại ~410 MB.

### 7.4. Reset và reboot cả ba node

Chạy trên **cả ba node**, master trước, worker sau. `-f` bỏ prompt xác nhận:

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d "$HOME/.kube"
sudo reboot
```

Sau khi cả ba node lên lại, trên **từng node**:

```bash
ip -brief link | grep -E 'cni0|flannel' || echo 'PASS: không còn cni0/flannel.1'
sudo iptables -S | grep -c KUBE- ; sudo iptables -t nat -S | grep -c KUBE-
# PASS: cả hai là 0
# kubeadm reset xóa NỘI DUNG của manifests/ và pki/ nhưng giữ lại thư mục. Gate phải phân biệt
# "đọc được và rỗng" với "không đọc được": kiểm exit code của find, không chỉ kiểm output rỗng.
CLEAN_RC=0
for d in /etc/kubernetes/manifests /etc/kubernetes/pki; do
  # Phân biệt ba trạng thái: tồn tại / không tồn tại / không kiểm tra được (sudo lỗi, I/O lỗi)
  STAT_OUT=$(sudo stat -c %n "$d" 2>&1); STAT_RC=$?
  if [ "$STAT_RC" -ne 0 ]; then
    if printf '%s' "$STAT_OUT" | grep -q 'No such file or directory'; then
      echo "PASS: $d không tồn tại"; continue
    fi
    echo "FAIL: không kiểm tra được $d: $STAT_OUT"; CLEAN_RC=1; continue
  fi
  if ! LEFT=$(sudo find "$d" -mindepth 1 2>&1); then
    echo "FAIL: không đọc được $d: $LEFT"; CLEAN_RC=1; continue
  fi
  if [ -z "$LEFT" ]; then echo "PASS: $d rỗng"; else echo "FAIL: $d còn:"; echo "$LEFT"; CLEAN_RC=1; fi
done
for f in /etc/kubernetes/admin.conf /etc/cni/net.d; do
  STAT_OUT=$(sudo stat -c %n "$f" 2>&1); STAT_RC=$?
  if [ "$STAT_RC" -eq 0 ]; then echo "FAIL: $f còn"; CLEAN_RC=1
  elif printf '%s' "$STAT_OUT" | grep -q 'No such file or directory'; then echo "PASS: $f đã xóa"
  else echo "FAIL: không kiểm tra được $f: $STAT_OUT"; CLEAN_RC=1; fi
done
[ "$CLEAN_RC" -eq 0 ] && echo 'PASS: node đã sạch' || echo 'FAIL: node chưa sạch — xem dòng FAIL ở trên'
sudo systemctl is-active containerd kubelet
# PASS: containerd active; kubelet có thể "activating" — bình thường, nó đang chờ config từ init/join
sudo crictl images | grep -cE 'registry.k8s.io|flannel|traefik|cloudflare|rancher'
# Image còn trong cache nên init/join và các bước sau không phải pull lại
```

> **DỪNG — GỬI OUTPUT CHECKPOINT 7.4 CỦA CẢ BA NODE.**

### 7.5. Dựng lại theo Phase 1 rồi Phase 2

Chạy lại **nguyên văn**, đủ gate, không bỏ mục:

1. [§6](runbook-k8s-vmware.md#6-khởi-tạo-control-plane-chỉ-master) — cùng `--kubernetes-version`,
   `--control-plane-endpoint`, `--apiserver-advertise-address`, `--pod-network-cidr` như lần đầu;
   kubeconfig cách A. Rồi [§6.1](runbook-k8s-vmware.md#61-cài-cni-flannel--chỉ-master) Flannel.
2. [§7](runbook-k8s-vmware.md#7-join-worker-chỉ-2-worker) — token join mới từ output `init`.
3. [§8](runbook-k8s-vmware.md#8-verify-cụm) đủ bảy tầng; §8.7 cài lại metrics-server và
   `local-path-provisioner` (Helm binary vẫn còn, `helm version` PASS ngay). Trước khi apply
   `local-path`, dọn `/opt/local-path-provisioner/` cũ trên hai worker như §4.5.
4. [§9](runbook-k8s-vmware.md#9-cài-ingress-controller-traefik), [§10](runbook-k8s-vmware.md#10-deploy-app-mẫu--ingress).
5. [§12.2](runbook-k8s-vmware.md#122-deploy-cloudflared-vào-cụm) — **không** tạo tunnel mới. Lấy
   lại token của tunnel `homelab-k8s` hiện có trên dashboard (Zero Trust → Networks → Connectors →
   tunnel → Install and run a connector → copy token) rồi apply `cloudflared.yaml`. Route và Access
   còn nguyên nên bỏ qua [§12.3.3](runbook-k8s-vmware.md#1233-thêm-published-application); chạy
   [§13](runbook-k8s-vmware.md#13-trỏ-domain--kiểm-tra-trên-internet) để verify.
6. Toàn bộ [§14](runbook-k8s-vmware.md#14-cài-rancher-2143--quản-lý-cụm) theo bảng ở §5 của file
   này (kể cả backup §14.0.1 mới).
7. Phase 2 từ [§3](runbook-k8s-vmware-phase2.md#3-gate-đầu-vào-từ-phase-1) theo bảng ở §6.

Ước lượng: nửa ngày trở lên nếu đi đủ gate. Đây là lý do restore (§3–§6) luôn được thử trước.

---

## 8. Phòng ngừa cho lần sau

Không có cách nào làm bbolt chịu được host mất điện với write cache của host; chỉ có thể **rút
ngắn khoảng mất dữ liệu** và **giảm xác suất**.

### 8.1. Backup etcd định kỳ, tự động

Backup 14/08 cứu được cụm nhưng mất sáu tuần thay đổi. Đặt cron chạy lại đúng bước 1 của
[§14.0.1](runbook-k8s-vmware.md#1401-backup-etcd-và-cấu-hình-trước-thay-đổi-lớn) mỗi đêm và giữ 7
bản gần nhất.

Cron **không có terminal**, nên bất kỳ lệnh `sudo` nào cần mật khẩu đều làm job chết im lặng. Trên
cụm này `sudo` của user `ubuntu` hỏi mật khẩu (đã thấy ở các bước trước), và `sudo -n true` PASS
được nhờ phiên vừa xác thực nên **không** chứng minh gì. Cách không phụ thuộc vào sudoers: đặt job
trong **crontab của root**, script không gọi `sudo`, `kubectl` dùng `admin.conf` qua `KUBECONFIG`,
và kết quả `chown` về `ubuntu` để giữ cùng thư mục `~/k8s-backups` với §14.0.1.

Trên **`k8s-master`**:

```bash
sudo tee /usr/local/sbin/etcd-backup.sh >/dev/null <<'EOS'
#!/usr/bin/env bash
# Chạy bằng root từ crontab của root. Không gọi sudo.
set -euo pipefail
umask 077
export KUBECONFIG=/etc/kubernetes/admin.conf
OWNER=ubuntu
ROOT="/home/$OWNER/k8s-backups"
STAMP=$(date +%Y%m%d-%H%M%S)
DIR="$ROOT/$STAMP"
STAGING="/var/lib/etcd/kubeadm-snapshot-$STAMP.db"

mkdir -p "$ROOT"; chmod 700 "$ROOT"; chown "$OWNER:$OWNER" "$ROOT"
mkdir -m 700 "$DIR"
kubectl -n kube-system exec etcd-k8s-master -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save "$STAGING" >/dev/null
cp "$STAGING" "$DIR/etcd-snapshot.db"
cp -a /etc/kubernetes "$DIR/etc-kubernetes"
test -s "$DIR/etcd-snapshot.db"
cmp -s "$STAGING" "$DIR/etcd-snapshot.db"
rm -f "$STAGING"
tar -C "$ROOT" -czf "$DIR.tar.gz" "$STAMP"
chmod 600 "$DIR.tar.gz"; chown "$OWNER:$OWNER" "$DIR.tar.gz"
rm -rf "$DIR"
# giữ 7 archive mới nhất
ls -1t "$ROOT"/*.tar.gz | tail -n +8 | xargs -r rm -f
echo "PASS: $STAMP $(sha256sum "$DIR.tar.gz" | cut -d' ' -f1)"
EOS
sudo chmod 700 /usr/local/sbin/etcd-backup.sh

# Gate 1 — chạy tay bằng root, đúng môi trường cron sẽ dùng (không kế thừa biến của phiên ubuntu)
sudo env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
  /usr/local/sbin/etcd-backup.sh
# PASS: in "PASS: <STAMP> <sha256>"; file ~/k8s-backups/<STAMP>.tar.gz thuộc ubuntu, mode 600
ls -la ~/k8s-backups/ | tail -3

# Gate 2 — cài vào crontab của root
( sudo crontab -l 2>/dev/null | grep -v etcd-backup.sh; \
  echo '15 2 * * * /usr/local/sbin/etcd-backup.sh >> /home/ubuntu/k8s-backups/cron.log 2>&1' ) \
  | sudo crontab -
sudo crontab -l | grep etcd-backup.sh
# PASS: đúng một dòng

# Gate 3 — chứng minh cron thật sự chạy được job: đặt tạm một lần chạy sau 2 phút, rồi gỡ
RUN_AT=$(date -d '+2 min' '+%M %H')
( sudo crontab -l; echo "$RUN_AT * * * /usr/local/sbin/etcd-backup.sh >> /home/ubuntu/k8s-backups/cron.log 2>&1 # once" ) | sudo crontab -
sleep 150
tail -2 ~/k8s-backups/cron.log
# PASS: có dòng "PASS: <STAMP> <sha256>" mới, sinh bởi cron chứ không phải bởi tay
sudo crontab -l | grep -v '# once' | sudo crontab -
sudo crontab -l
# PASS: chỉ còn dòng 02:15 hằng ngày
```

Gate 3 là bằng chứng duy nhất cho câu "backup ban đêm hoạt động": job chạy từ cron, không terminal,
không biến môi trường của phiên SSH. Sáng hôm sau kiểm `cron.log` thêm một lần nữa.

Script chỉ giữ `.tar.gz`; restore thì `tar -xzf` trước (như §3.4 bước 2). Bản copy ra host vẫn làm
tay theo bước 2 của §14.0.1 sau mỗi mốc lớn (`cron.log` cho biết STAMP và hash để điền).

### 8.2. Snapshot VMware sau mỗi mốc

Ba VM của cụm chưa từng có snapshot VMware. Sau khi §5 và §6 PASS, chụp cả ba VM **đang tắt** để
snapshot nhất quán. Trên **host Windows**, sau khi đã `sudo shutdown -h now` worker2 → worker1 →
master:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmx = @(
  'E:\Virtual Machines\k8s-master-node.vmx'
  'E:\Virtual Machines\k8s-worker1\Clone of Ubuntu 64-bit.vmx'
  'E:\Virtual Machines\k8s-worker2\k8s-worker2.vmx'
)
$name = 'rancher-restored-' + (Get-Date -Format 'yyyyMMdd')
foreach ($f in $vmx) { & $vmrun -T ws snapshot $f $name; & $vmrun -T ws listSnapshots $f }
# PASS: mỗi VM in "Total snapshots: 1" và tên vừa đặt
```

Snapshot VM lấy lại được **toàn bộ** trạng thái (Rancher, Phase 2) trong vài phút thay vì nửa ngày.
Giá phải trả là dung lượng và VM chạy chậm hơn chút khi delta lớn; xóa snapshot cũ khi đã có mốc
mới.

### 8.3. Tắt đúng thứ tự trước khi tắt host

Cả hai điều kiện ở §1.2 đều tránh được nếu guest **tắt trước host**. Trước khi shutdown/restart
host hoặc khi biết sắp mất điện: `sudo shutdown -h now` trên worker2, worker1, rồi master; chờ
Workstation báo cả ba VM đã tắt; rồi mới tắt host. Cấu hình Workstation cho VM **suspend** thay
vì power off khi host shutdown cũng đạt. Nếu host có UPS, cấu hình để nó shutdown host sạch.

### 8.4. Giảm cache ghi của host cho đĩa VM (tùy chọn)

VMware Workstation có các tham số `.vmx` bắt đầu bằng `diskLib.` để giảm cache ghi phía host
(ví dụ `diskLib.maxUnsyncedWrites = "0"` cùng nhóm `diskLib.dataCache*`). Chúng được cộng đồng
homelab dùng rộng rãi nhưng **không có trang tài liệu chính thức** để runbook này dẫn, nên chỉ ghi
nhận ở mức tùy chọn: bạn tự đọc tài liệu Workstation của đúng phiên bản đang dùng trước khi đặt,
và chấp nhận I/O chậm hơn. §8.1–§8.3 là bắt buộc, §8.4 thì không.

---

## 9. Troubleshooting của runbook này

| Triệu chứng | Nguyên nhân | Xử lý |
| --- | --- | --- |
| §2.3 `ctr run` báo `image ... not found` | Sai tên image hoặc namespace containerd | Dùng đúng `-n k8s.io`; tên image lấy từ `sudo crictl images \| grep etcd` |
| §3.1 dời manifest xong mà container control plane vẫn `Running` (thường `kube-controller-manager`, `kube-scheduler`) | apiserver chết nên kubelet chưa thấy nguồn API sẵn sàng và bỏ qua mọi lần xóa Pod; chờ thêm hay `systemctl restart kubelet` đều không đổi | Chạy khối 2 của §3.1: dừng kubelet rồi `crictl stopp`. Nếu `ls /etc/kubernetes/manifests/` còn file `.yaml`, dời nốt vào `/etc/kubernetes/manifests.off-$(cat ~/etcd-restore-stamp.txt)/`; **không** chạy lại khối 1 |
| §3.3 `data-dir ... exists` | `/var/lib/etcd` chưa được đổi tên (lần đầu) hoặc còn data dir dở của lần FAIL trước | Lần đầu: §3.2. Đã qua §3.2 rồi: §3.4 bước 1, **không** chạy lại §3.2 |
| §3.3 `expected sha256 <a>, got <b>` | Checksum nhúng của snapshot không khớp: file bị sửa hoặc copy dở | §3.4: dọn data dir dở, lấy bản từ host, so hash, chạy lại §3.3 |
| §3.3 `snapshot missing hash but --skip-hash-check=false` | File chốt không trỏ tới snapshot do `etcdctl snapshot save` tạo | Kiểm `cat ~/etcd-restore-snapshot-path.txt`; không thêm `--skip-hash-check`; xem bảng §3.4 |
| §4.1 etcd `panic` dù §3.3 PASS | Có thể là snapshot, có thể là mount/quyền/đĩa của `/var/lib` | §3.4 bước 4 tách hai trường hợp; chỉ khi `etcdutl` cũng không mở được bản restore thứ hai mới xét §7 |
| §3.3 `permission denied` / lỗi `mount` / `no such file` | Lỗi thao tác, không phải snapshot | Sửa đường dẫn/quyền; §3.4 bước 1 rồi chạy lại §3.3 với cùng bản snapshot |
| §4.1 ATTEMPT của apiserver là 1 sau khi lên | apiserver restart một lần lúc etcd chưa sẵn sàng | Bình thường nếu không tăng tiếp giữa hai lần xem cách 60 giây |
| §4.1 etcd `Running` nhưng apiserver crashloop `TLS handshake timeout` | apiserver lên trước khi etcd sẵn sàng | Race bình thường; chờ đủ vòng lặp. Chỉ điều tra khi etcd cũng restart |
| §4.2 revision nhỏ hơn 10⁹ | Thiếu `--bump-revision` hoặc apiserver đang đọc data dir khác | Kiểm `--data-dir` trong manifest; lặp lại §3.1–§3.3 với đủ flag |
| §4.3 worker `NotReady` kéo dài | kubelet worker còn cert/lease cũ, hoặc Flannel chưa lên | `sudo systemctl restart kubelet` trên worker; xem log `journalctl -u kubelet -b --no-pager \| tail -50` |
| §4.3 Pod cũ của Rancher/Phase 2 vẫn hiện trong `crictl ps` trên worker | kubelet chưa dọn | Đợi vài phút sau restart kubelet; không `crictl rm` tay |
| §4.4 app mẫu `502`/`530` qua Internet | cloudflared chưa đăng ký lại tunnel | `kubectl -n cloudflare logs -l app=cloudflared --tail=50`; Secret token phải tồn tại; tunnel trên dashboard phải Healthy |
| §5 §14.0 phát hiện namespace `cattle-system` cũ | Restore không thành công hoặc dùng snapshot sau §14 | Nếu snapshot có sẵn Rancher thì §14 đổi thành quy trình repair theo §15 Phase 1; không `helm install` đè |
| Login Rancher 404 `selfuser` sau §14.6 | Race đã biết của 2.14.x | [§15 Phase 1](runbook-k8s-vmware.md#rancher-đăng-nhập-404-tại-v1extcattleioselfuser-sau-khi-reboot-node) |
| §7.4 sau reboot vẫn còn `cni0` | Netplan/systemd-networkd quản lý interface đó (hiếm) | `sudo ip link delete cni0`; `sudo ip link delete flannel.1`; kiểm lại |

Sự cố dựng lại cụm sau §7 thuộc [bảng lỗi Phase 1](runbook-k8s-vmware.md#bảng-lỗi-thường-gặp).

---

## 10. Checklist hoàn tất

- [ ] §2.1: xác nhận etcd chết vì `panic` bbolt, không phải OOM/đĩa/cert.
- [ ] §2.2: hash backup khớp trên master và host; PKI đạt một trong hai nhánh PASS của §2.2 (không đổi, hoặc chỉ cert thành phần đã renew với CA/`sa.key` giống backup và `openssl verify` OK).
- [ ] §2.3: `etcdutl 3.6.6` chạy được từ image; tham số etcd khớp; đã ghi REVISION của snapshot.
- [ ] §2.4: đã chạy `grep`, ghi lại năm số đếm và nhánh **dự đoán** (chuẩn / thiếu cloudflared / nghi có Rancher); nhánh **thật** chốt ở §4.2 theo kết quả API sau restore.
- [ ] §3.1–§3.3: control plane dừng sạch; `/var/lib/etcd.corrupt-*` được giữ; restore thoát mã 0.
- [ ] §4.1–§4.2: 4 static Pod Running, ATTEMPT ổn định giữa hai lần xem; `readyz` PASS; revision > 10⁹; kết quả `get ns` và `get deploy,secret -n cloudflare` đã đọc và nhánh cho §4.3–§5 đã chốt theo bảng §4.2.
- [ ] §4.3–§4.4: 3 node Ready; đủ bảy tầng §8 Phase 1; app mẫu 200 qua Internet.
- [ ] §4.5: đã kiểm kê thư mục PV mồ côi của MongoDB trên hai worker và ghi rõ quyết định giữ/bỏ.
- [ ] §5: backup §14.0.1 mới đã tạo và copy ra host; §14.7 PASS — UI Rancher `local` Active qua Cloudflare Access.
- [ ] §6: Phase 2 checkpoint 7.1, 8.2, 9.3, 10.2 PASS.
- [ ] §8.1: gate 3 PASS — `cron.log` có dòng PASS do cron của root sinh ra; §8.2: ba VM có snapshot VMware; §8.3: đã nắm thứ tự tắt.
- [ ] Sau khi mọi thứ PASS: xóa `/var/lib/etcd.corrupt-*` để lấy lại đĩa.

---

## 11. Nguồn official

- Kubernetes — *Operating etcd clusters for Kubernetes* (backup, restore, dừng API server, restart component sau restore): [https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
- etcd v3.6 — *Disaster recovery* (`etcdutl snapshot restore`, integrity hash, `--bump-revision`, `--mark-compacted`): [https://etcd.io/docs/v3.6/op-guide/recovery/](https://etcd.io/docs/v3.6/op-guide/recovery/)
- etcd v3.6 — *etcdutl README* (đầy đủ option của `snapshot restore`): [https://github.com/etcd-io/etcd/blob/release-3.6/etcdutl/README.md](https://github.com/etcd-io/etcd/blob/release-3.6/etcdutl/README.md)
- Kubernetes — *kubeadm reset* (xóa gì, không xóa gì: CNI, iptables, `$HOME/.kube`): [https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
- Kubernetes — *Creating a cluster with kubeadm*: [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- Kubernetes — *Create static Pods* (kubelet quét thư mục manifest): [https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
- Kubernetes — mã nguồn kubelet nhánh `release-1.35` (`deletePod` và housekeeping bỏ qua xóa Pod khi chưa thấy đủ nguồn; vì sao §3.1 dùng `crictl stopp`): [https://github.com/kubernetes/kubernetes/blob/release-1.35/pkg/kubelet/kubelet.go](https://github.com/kubernetes/kubernetes/blob/release-1.35/pkg/kubelet/kubelet.go)
- bbolt — *README* (mô hình transaction, `NoFreelistSync`): [https://github.com/etcd-io/bbolt](https://github.com/etcd-io/bbolt)
- Flannel — *running.md* (`/run/flannel/subnet.env`, interface `flannel.1`): [https://github.com/flannel-io/flannel/blob/master/Documentation/running.md](https://github.com/flannel-io/flannel/blob/master/Documentation/running.md)
- Rancher — *local-path-provisioner* (thư mục hostPath `/opt/local-path-provisioner`): [https://github.com/rancher/local-path-provisioner](https://github.com/rancher/local-path-provisioner)
- VMware — *vmrun command reference* (`snapshot`, `listSnapshots`): [https://docs.vmware.com/en/VMware-Workstation-Pro/index.html](https://docs.vmware.com/en/VMware-Workstation-Pro/index.html)

---

*Runbook tạo ngày 2026-10-01 từ sự cố 30/09/2026. Baseline theo [bảng §2.1 Phase 1](runbook-k8s-vmware.md#21-phiên-bản-ghim-để-khỏi-lệch-version-skew); etcd `3.6.6` trong image `registry.k8s.io/etcd:3.6.6-0` của kubeadm. Đây là homelab baseline, không phải quy trình DR production.*
