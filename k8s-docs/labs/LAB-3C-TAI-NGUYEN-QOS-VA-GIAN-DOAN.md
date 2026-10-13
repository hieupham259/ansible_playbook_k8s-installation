# Lab 3c — Tài nguyên, QoS và gián đoạn

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> Repo `~/bookstore` ở tag `lab-3b` (hoặc bundle `~/bookstore-lab-3b.bundle` nếu bạn vừa restore
> snapshot).
> **Điểm kết thúc:** teardown trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới. Repo
> có tag `lab-3c` và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 3 — Định cỡ và chịu gián đoạn của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 3b — Cấu hình ứng dụng](LAB-3B-CAU-HINH-UNG-DUNG.md).
> **Lab sau:** [Lab 4a — ReplicaSet, Deployment và rollout](LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng mục
[3c. Tài nguyên, QoS và gián đoạn](../00-ALO-TRINH-ADMIN.md#3c-tài-nguyên-qos-và-gián-đoạn) của
[Giai đoạn 3](../00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình). Đây là sprint thứ ba của
[dự án Bookstore](DU-AN-BOOKSTORE.md). Trước khi mở B0, đọc lại
[mục 4 — hợp đồng kỹ thuật](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật),
[mục 3.6 — ngân sách tài nguyên](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch), hàng "3c" của
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint) và
[kế hoạch sprint 3](DU-AN-BOOKSTORE.md#sprint-3--lab-3c-định-cỡ-và-chịu-gián-đoạn).

Sprint 2 để lại ba Pod trần chạy đúng vòng đời, cấu hình và bí mật đã ra khỏi spec — nhưng không
container nào khai tài nguyên (`TD-05`). Scheduler đặt Pod mà không biết chúng cần bao nhiêu, kernel
không chặn container nào, và một container rò bộ nhớ có thể kéo cả node. Sprint này đo mức dùng thật,
chốt ngân sách từ số đo, chọn QoS cho từng thành phần, tập đổi tài nguyên trên Pod đang chạy, và
đặt ngân sách gián đoạn cho hai thành phần phục vụ người dùng.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); image của dự án nằm
ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Lab không chép lại
con số phiên bản nào trong phần văn bản và **không cài thêm gì**. Hai dữ kiện của baseline quyết
định phạm vi sprint: resize tài nguyên container tại chỗ đã ổn định (stable) và bật mặc định; tài
nguyên cấp Pod ở beta, bật mặc định, nhưng resize tài nguyên cấp Pod còn alpha và tắt.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi mọi container của Bookstore — kể cả init container và sidecar — có tài nguyên
khai báo từ số đo thật, `web` và `api` có ngân sách gián đoạn, **và** bạn chứng minh, giải thích
được:

- Mức dùng thật của từng container đọc ở đâu khi cluster chưa có Metrics API: `cpu.stat`,
  `memory.current` trong cgroup v2 trên node; vì sao `allocatable` nhỏ hơn `capacity` và scheduler
  dùng con số nhỏ hơn.
- `requests` nói chuyện với scheduler, `limits` nói chuyện với kernel: chỉ ra `cpu.max`,
  `memory.max` và trọng số `cpu.weight` của chính container `postgres` trên `lab-k8s-worker2`.
- Vì sao `db` là `Guaranteed` còn `web`, `api` là `Burstable`; vì sao một init container hay sidecar
  khai thiếu là đủ kéo QoS của cả Pod; vì sao chỉ khai limit vẫn có thể ra `Guaranteed`.
- Request và limit **hiệu dụng** của Pod có init container và sidecar: tính được bằng tay, thấy
  scheduler dùng nó, và thấy nó thành `memory.max`, `cpu.max` của cgroup cấp Pod.
- Vượt trần thì chuyện gì xảy ra: CPU bị điều tiết, memory bị `OOMKilled` ở mức **container**; Pod
  `Pending` vì request vượt `allocatable` hoặc vượt phần còn trống, dù node gần như rảnh.
- Đổi CPU của `api` đang chạy mà không khởi động lại: tài nguyên mong muốn khác tài nguyên thực tế,
  `Infeasible`, và những gì resize **không** làm được (đổi QoS, init container thường, gỡ limit,
  static Pod); vì sao repo vẫn phải được đồng bộ sau một thao tác tại chỗ.
- PodDisruptionBudget cho Pod trần chỉ dùng được một dạng; Eviction API bị chặn còn `kubectl delete`
  thì không; gián đoạn không tự nguyện không bị chặn nhưng vẫn bị tính.
- Static Pod do kubelet tự quản: mirror Pod nhìn thấy nhưng không điều khiển được; xóa thật bằng
  file; vì sao kubeadm dựng control plane bằng nó còn agent của dự án nên chờ DaemonSet.
- Sprint đóng lại với `TD-05` đã trả, có bằng chứng.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 3c | Ticket kiểm chứng |
| --- | --- |
| [110 — Quản lý tài nguyên cho Pod và Container](../110-manage-resources-containers-vi.md) | B1 — `capacity` khác `allocatable`, đo mức dùng khi chưa có Metrics API; B2 — `requests` cho scheduler, `limits` thành `cpu.max`/`memory.max`, request CPU thành trọng số; B4 — throttle, `OOMKilled`, `Pending` kèm `FailedScheduling`; B5 — resize tại chỗ; B9 ⭐ — tài nguyên mở rộng |
| [54 — Các lớp chất lượng dịch vụ của Pod](../54-pod-qos-vi.md) | B2 — QoS từng thành phần, ADR; B3 — suy QoS từ manifest, init container và sidecar cũng bị tính, `BestEffort`; B4 — kill vì vượt limit ở mức container; B5 — QoS bất biến khi resize; B8 ⭐ — QoS theo tài nguyên cấp Pod |
| [53 — Sự gián đoạn](../53-disruptions-vi.md) | B6 — tự nguyện và không tự nguyện, Eviction API bị PDB chặn, `kubectl delete` bỏ qua PDB |
| [58 — Pod tĩnh](../58-static-pods-vi.md) | B7 — static Pod của control plane, mirror Pod, giới hạn, so với DaemonSet |
| [263 — Gán tài nguyên CPU](../263-assign-cpu-resource-vi.md) | B2 — `limits.cpu` thành `cpu.max`; B4 — throttle, request CPU vượt sức node |
| [264 — Gán tài nguyên memory](../264-assign-memory-resource-vi.md) | B2 — `limits.memory` thành `memory.max`; B4 — `OOMKilled` mã 137, request memory vượt sức node |
| [265 — Gán tài nguyên ở cấp Pod](../265-assign-pod-level-resources-vi.md) | B8 ⭐ — `spec.resources` cấp Pod, QoS theo cấp Pod, cgroup cấp Pod |
| [284 — Gán Extended Resource](../284-extended-resource-vi.md) | B9 ⭐ — quảng bá bốn dongle trên `lab-k8s-worker2`, tiêu thụ, `Pending` khi hết, request phải bằng limit |
| [288 — Cấu hình Quality of Service cho Pod](../288-quality-service-pod-vi.md) | B3 — dự đoán rồi đối chiếu `status.qosClass` cho biến thể manifest của chính dự án |
| [289 — Resize tài nguyên của Container](../289-resize-container-resources-vi.md) | B5 — gấp đôi CPU của `api` không khởi động lại, `Infeasible`, năm ranh giới |
| [290 — Resize tài nguyên của Pod](../290-resize-pod-resources-vi.md) | **Không thực hành được.** Cần feature gate alpha `InPlacePodLevelResourcesVerticalScaling` mà baseline không bật. B8 ⭐ chỉ chứng minh API server chặn mọi resize của Pod có tài nguyên cấp Pod |
| [339 — Chỉ định Disruption Budget](../339-configure-pdb-vi.md) | B6 — PDB `web`, `api` cho Pod trần, đọc `status`; ngân sách tiêu dần trên Pod thử nghiệm; ⭐ PDB chồng lấn |
| [293 — Tạo static Pod](../293-static-pod-tasks-vi.md) | B7 — static Pod thử nghiệm trên `lab-k8s-worker2`: tạo bằng file, xóa mirror, kubelet tự khởi động lại, xóa bằng file |

Hai nợ lab của [Lab 3a](LAB-3A-POD-VA-VONG-DOI.md) được **trả** ở sprint này — ticket trả nợ nhắc
đọc lại bài gốc trước khi làm:

| Nợ của Lab 3a | Trả ở |
| --- | --- |
| Request/limit hiệu dụng của init container và sidecar (bài [50](../50-init-containers-vi.md), [51](../51-sidecar-containers-vi.md)) | B3 |
| Thực hành static Pod (bài [293](../293-static-pod-tasks-vi.md)) | B7 |

Các phần dưới đây bị hoãn có chủ đích — nói được vì sao ở checkpoint:

| Phần hoãn | Vì cần | Trả ở |
| --- | --- | --- |
| `kubectl drain`, `cordon` và PDB làm drain treo (bài 53, 339) | quy trình bảo trì node | [Lab 12](LAB-12-VAN-HANH-VONG-DOI-NODE.md) |
| `unhealthyPodEvictionPolicy` của PDB (bài 339) | chỉ có nghĩa khi đang drain node | Lab 12 |
| Trục xuất do áp lực node và thứ tự `BestEffort` → `Burstable` → `Guaranteed` (bài 54) | cơ chế eviction của kubelet, bài [142](../142-node-pressure-eviction-vi.md) | [Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md) |
| Chiếm chỗ (preemption) khi Pod ưu tiên cao bị `Pending` | bài [141](../141-pod-priority-preemption-vi.md) | Lab 7a |
| Metrics API, `kubectl top` (bài 110, 263, 264) | metrics-server | [Lab 11a](LAB-11A-OBSERVABILITY.md) |
| Quảng bá extended resource bằng device plugin; bài [209](../209-extended-resource-node-vi.md) | device plugin (giai đoạn 14); bài 209 ở giai đoạn 25 | đánh giá ở [Lab 13](LAB-13-DRA.md) |
| Resize tài nguyên cấp Pod (bài 290) | feature gate alpha, sửa cấu hình apiserver và kubelet | không kiểm chứng được trong lộ trình |

Nợ lab còn mở từ các sprint trước: phase `Unknown` (Lab 12) và readiness gỡ Pod khỏi EndpointSlice
(Lab 5a) của Lab 3a; [nợ lab #6](README.md#5-sổ-nợ-lab) của Lab 3b. Sprint này không chạm tới.

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B7, B10) khoảng 4 giờ nếu không bí lâu ở ticket nào; B8, B9 là tùy
chọn, mỗi ticket chừng 20 phút. Có thể chia hai phiên: phiên 1 làm B0–B4, phiên 2 làm B5–B10. Giữa
hai phiên cứ để nguyên cluster; đầu phiên 2 chạy lại gate ngắn A5.5 và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Mọi Pod thử nghiệm và diễn tập ghim vào node này.
  `db` cũng ở đây theo [thiết kế dự án](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch).
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/3c/`. Bằng chứng: `~/lab-evidence/3c/`.
  Object thử nghiệm đặt tên bắt đầu bằng `thu-`, nằm trong namespace `bookstore` nhưng **không** mang
  label `app.kubernetes.io/part-of`, và bị xóa khi xong ticket.
- Pod thử nghiệm **không** mang PriorityClass. Một Pod ưu tiên cao bị `Pending` có thể khiến
  scheduler đuổi Pod khác để lấy chỗ — cơ chế đó học ở Lab 7a, không được xảy ra trong sprint này.
- Lab chỉ dùng kiến thức nhóm 3c và các nhóm trước. **Không** dùng Deployment (4a), Service (5a),
  PVC (6a), LimitRange hay ResourceQuota (7b). **Không** cài metrics-server — nó thuộc Lab 11a; mọi
  số đo đọc từ API object và từ cgroup v2 trên node như
  [Lab 2 B2](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md#b2-cgroup-v2-và-cgroup-driver).
- **Không** chạy `kubectl drain`, `cordon`, `uncordon`; không gỡ taint của control plane.
- Pod trần gần như bất biến: API server từ chối đổi tài nguyên qua đường cập nhật Pod thông thường.
  Đổi tài nguyên trong repo thì `./scripts/teardown.sh && ./scripts/deploy.sh`; ngoại lệ duy nhất là
  thao tác tại chỗ có kiểm soát ở B5. Dữ liệu `emptyDir` mất theo — đó là `TD-04`.
- **Chỉ đọc** `/etc/kubernetes/manifests` trên `lab-k8s-master`. Một thao tác ghi nhầm ở đó làm chết
  control plane và phải restore snapshot. Static Pod thử nghiệm chỉ đặt trên `lab-k8s-worker2` và bị
  xóa ở B7.
- File cgroup đọc được bằng user thường, không cần `sudo`. Lệnh cần `sudo` bên trong
  `ssh <node> '...'` dùng `sudo -S` và bạn nhập mật khẩu khi được hỏi, theo
  [A2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a2-cài-ubuntu-và-cấu-hình-identity); không ghi mật khẩu
  vào lệnh hay file.
- Lab chạm vào hai thứ nằm ngoài namespace: danh sách tài nguyên của Node `lab-k8s-worker2` (B9 ⭐)
  và thư mục static Pod trên node đó (B7). Ticket nào chạm thì ticket đó dọn; gate cuối kiểm lại.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 3: Định cỡ và chịu gián đoạn

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `01-cluster-ready`, repo ở đúng tag `lab-3b`, hệ thống của
sprint 2 dựng lại được **chỉ từ repo**, và hai chỗ trên `lab-k8s-worker2` mà sprint này sẽ chạm
tới đang sạch.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Repo và thư mục làm việc. Nếu vừa restore snapshot, repo đã mất — khôi phục từ bundle theo
[mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) trước:

```bash
mkdir -p ~/lab-work/3c ~/lab-evidence/3c
test -d ~/bookstore/.git || git clone ~/bookstore-lab-3b.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-3b' && echo 'PASS: HEAD dung tag lab-3b'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
kubectl get apiservices | grep -q 'metrics.k8s.io' \
  && echo 'FAIL: cluster da co Metrics API, lech baseline' || echo 'PASS: chua co Metrics API'
kubectl get node lab-k8s-worker2 -o jsonpath='{.status.capacity}' | grep -q 'dongle' \
  && echo 'FAIL: lab-k8s-worker2 con extended resource sot lai' || echo 'PASS: lab-k8s-worker2 khong co extended resource la'
ssh lab-k8s-worker2 'test -d /etc/kubernetes/manifests && echo existed || echo created' \
  > ~/lab-evidence/3c/b0-worker2-manifests.txt
grep -Eqx 'existed|created' ~/lab-evidence/3c/b0-worker2-manifests.txt && echo 'PASS: da ghi trang thai thu muc static Pod cua worker2'
```

File `b0-worker2-manifests.txt` cho B7 biết thư mục static Pod trên worker2 có sẵn từ trước hay sẽ
do lab tạo ra, để trả nó về đúng trạng thái cũ.

Vừa restore snapshot thì chạy lại cả khối kéo image ở [B0 của Lab 3a](LAB-3A-POD-VA-VONG-DOI.md):
sprint này cần `busybox` và `nginx-unprivileged` có sẵn trên `lab-k8s-worker2` cho Pod thử nghiệm.

Dựng lại sprint 2 bằng chính script của sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/3c/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/3c/b0-smoke-lab-3b.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/3c/b0-smoke-lab-3b.txt
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; sáu dòng `PASS:` của khối repo; `deploy exit=0`,
`smoke exit=0` và mọi dòng của smoke là `PASS:`.

Smoke fail nghĩa là repo của sprint 2 **chưa đủ** — có thứ chỉ tồn tại trên cluster cũ. Sửa repo ở
tag `lab-3b` trước, không sửa tay trên cluster ([mục 6.1](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-3C-01] Đo mức dùng thật trước khi định cỡ

**Loại:** Điều tra.

**Bối cảnh:** hệ thống cũ không đo được tải ([`P8`](DU-AN-BOOKSTORE.md#21-hiện-trạng)), và bảng
[3.6 của file dự án](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch) chỉ là con số gợi ý. Chốt
`requests` và `limits` mà không đo thì hoặc giữ chỗ thừa trên hai worker nhỏ, hoặc kernel giết
container giữa giờ cao điểm. Cluster chưa có Metrics API, nên số đo phải đọc thẳng từ cgroup v2 —
đúng chỗ kernel đang đếm.

**Yêu cầu:**

1. Ghi `capacity` và `allocatable` về CPU, memory của ba node; chứng minh `allocatable` nhỏ hơn
   `capacity`.
2. Chứng minh cluster chưa có Metrics API: `kubectl top` không dùng được.
3. **Đọc tay** thư mục cgroup v2 của container `postgres` trên `lab-k8s-worker2`, ghi vào
   `~/lab-evidence/3c/b1-tay.txt` đúng bốn dòng dạng `tên: giá trị`:
   `cpu.max: ...`, `memory.max: ...`, `memory.current: ...` và `usage_usec: ...` (dòng cùng tên trong
   file `cpu.stat`). Hai dòng đầu lúc này phải cho thấy container **không có trần**.
4. Đo bốn container chạy lâu — `db/postgres`, `api/postgrest`, `web/nginx`, `web/status-writer` —
   trên đúng node đang chạy chúng: CPU trung bình khi nghỉ và khi có tải (millicore), memory khi có
   tải (MiB), đỉnh memory nếu kernel có ghi. Tải do Pod thử nghiệm `thu-tai` tạo: ghim
   `lab-k8s-worker2`, gọi `/books` của `api` và trang chủ của `web` theo nhịp đều — không dồn dập —
   trong khoảng một phút rồi tự kết thúc. Kết quả vào `~/lab-evidence/3c/b1-do-dac.txt`, mỗi
   container một dòng: `<pod>/<container> <cpu_nghi_m> <cpu_tai_m> <mem_tai_mib> <mem_peak_mib|NA>`,
   dòng đầu là chú thích bắt đầu bằng `#`. B2 đọc file này.
5. Mở `docs/adr/0018-ngan-sach-tai-nguyen.md` từ mẫu ADR: mục *Bối cảnh* là bảng số đo và đường dẫn
   bằng chứng. Mục *Quyết định* viết ở B2.

**Ràng buộc:** ticket này chưa khai báo tài nguyên nào. Không cài metrics-server. Không ghi gì dưới
`/sys/fs/cgroup`. Pod `thu-tai` không mang label dự án và bị xóa khi đo xong.

**Đọc lại:**

- Bài [110](../110-manage-resources-containers-vi.md):
  [*Cách Kubernetes áp dụng request và limit tài nguyên*](../110-manage-resources-containers-vi.md#how-pods-with-resource-limits-are-run)
  — kubelet chuyển cho runtime, runtime cấu hình cgroup;
  [*Giám sát mức sử dụng tài nguyên tính toán và bộ nhớ*](../110-manage-resources-containers-vi.md#monitoring-compute-memory-resource-usage)
  — Metrics API; đoạn cuối mục
  [*Pod của tôi ở trạng thái pending…*](../110-manage-resources-containers-vi.md#my-pods-are-pending-with-event-message-failedscheduling)
  về `.status.allocatable`.
- Bài [263](../263-assign-cpu-resource-vi.md) mục *Đơn vị CPU* và bài
  [264](../264-assign-memory-resource-vi.md) mục *Đơn vị memory*; bảng *Đọc lướt* của cả hai bài —
  vì sao lab không dùng `kubectl top`.
- [Lab 2 B2](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md#b2-cgroup-v2-và-cgroup-driver) — node dùng
  cgroup v2 và driver `systemd`.

**Bàn giao:** `docs/adr/0018-ngan-sach-tai-nguyen.md` (mục *Bối cảnh*); `~/lab-evidence/3c/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` `allocatable` memory nhỏ hơn `capacity` trên cả ba node.
- `PASS:` `kubectl top` báo không có Metrics API.
- `PASS:` `b1-tay.txt` cho thấy `cpu.max` và `memory.max` của `postgres` là `max`, kèm `usage_usec`.
- `PASS:` `b1-do-dac.txt` có đúng bốn dòng số; Pod tải đã chạy đủ vòng và đã bị xóa.

**Câu hỏi thiết kế** — trả lời trong mục *Bối cảnh* của `docs/adr/0018-ngan-sach-tai-nguyen.md`:

1. Scheduler so request với `capacity` hay `allocatable` của node? Phần chênh dành cho ai? Với số của
   `lab-k8s-worker2`, tổng request CPU tối đa của mọi Pod trên đó là bao nhiêu?
2. Lúc đo, bốn container chưa có trần. Theo bài 110 và bài 264, một container không có limit làm
   được gì với node, và bị đối xử ra sao khi node hết memory?
3. Vì sao đo dưới tải chứ không chỉ lúc nghỉ? Số đo một lần, với tải giả, có rủi ro gì — và lab nào
   sẽ cho bạn cách đo tốt hơn?
4. Bạn sẽ viết limit memory là `128Mi` hay `128M`? Hai cách khác nhau bao nhiêu byte, và cách nào
   khớp với giá trị kernel ghi trong `memory.max`?

<details>
<summary>Gợi ý</summary>

1. Container nằm trong thư mục `cri-containerd-<id>.scope` dưới `/sys/fs/cgroup/kubepods.slice/`,
   sâu hai hoặc ba tầng tùy QoS của Pod. `<id>` là phần sau `containerd://` của trường `containerID`
   trong status của Pod:
   `kubectl -n bookstore get pod db -o jsonpath='{.status.containerStatuses[0].containerID}'`.
2. Tìm thư mục trên node: `ssh lab-k8s-worker2 "find /sys/fs/cgroup/kubepods.slice -name 'cri-containerd-<id>.scope'"`.
   File cgroup đọc được không cần `sudo`.
3. `cpu.stat` có dòng `usage_usec` — tổng micro-giây CPU từ khi container chạy. Đọc hai lần cách nhau
   N giây, lấy hiệu chia cho `N × 1000` là millicore trung bình.
4. `memory.current` là byte đang dùng; `memory.peak` (kernel mới có) là đỉnh từ khi tạo cgroup.
5. Script nghiệm thu của ticket này **chính là phép đo chuẩn** của dự án — chạy sau khi bạn đã tự đọc
   được `b1-tay.txt`.

</details>

<details>
<summary>Script nghiệm thu — script này chính là phép đo; chạy sau khi đã đọc tay</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3c
cg() {   # cg <pod> <container> <file>: đọc một file cgroup v2 của container, trên đúng node chạy nó
  local n id
  n="$(kubectl -n "$NS" get pod "$1" -o jsonpath='{.spec.nodeName}')"
  id="$(kubectl -n "$NS" get pod "$1" -o jsonpath="{.status.containerStatuses[?(@.name==\"$2\")].containerID}{.status.initContainerStatuses[?(@.name==\"$2\")].containerID}")"
  ssh -n "$n" "find /sys/fs/cgroup/kubepods.slice -maxdepth 3 -type d -name 'cri-containerd-${id#containerd://}.scope' -exec cat {}/$3 \;"
}
CS='db/postgres api/postgrest web/nginx web/status-writer'
usage() { for X in $CS; do echo "$X $(cg "${X%/*}" "${X#*/}" cpu.stat | awk '$1=="usage_usec"{print $2}')"; done; }
val() { echo "$1" | awk -v k="$2" '$1==k{print $2}'; }

# 1. capacity và allocatable
kubectl get nodes -o custom-columns='NODE:.metadata.name,CPU_CAP:.status.capacity.cpu,CPU_ALLOC:.status.allocatable.cpu,MEM_CAP:.status.capacity.memory,MEM_ALLOC:.status.allocatable.memory' \
  | tee "$EV/b1-allocatable.txt"
BAD=0
for N in $(kubectl get nodes -o name); do
  C="$(kubectl get "$N" -o jsonpath='{.status.capacity.memory}')"
  A="$(kubectl get "$N" -o jsonpath='{.status.allocatable.memory}')"
  case "$C$A" in *Ki*Ki) [ "${C%Ki}" -gt "${A%Ki}" ] || BAD=1 ;; *) BAD=1 ;; esac
done
[ "$BAD" -eq 0 ] && echo 'PASS: allocatable memory nho hon capacity tren ca ba node'

# 2. Chưa có Metrics API
kubectl -n "$NS" top pod > "$EV/b1-top.txt" 2>&1 \
  && echo 'FAIL: kubectl top chay duoc — cluster da co Metrics API' \
  || echo 'PASS: kubectl top khong co Metrics API, phai doc cgroup'

# 3. Kết quả đọc tay
{ grep -q '^cpu.max: max' "$EV/b1-tay.txt" && grep -qx 'memory.max: max' "$EV/b1-tay.txt" && \
  grep -Eq '^memory.current: [0-9]+$' "$EV/b1-tay.txt" && grep -Eq '^usage_usec: [0-9]+$' "$EV/b1-tay.txt"; } \
  && echo 'PASS: da doc tay cgroup cua postgres — chua co tran CPU va memory'

# 4. Lúc nghỉ: 30 giây
I0="$(usage)"; sleep 30; I1="$(usage)"

# 5. Lúc có tải: Pod thu-tai trên worker2, khoảng 4 vòng mỗi giây trong 75 giây
API_IP="$(kubectl -n "$NS" get pod api -o jsonpath='{.status.podIP}')"
WEB_IP="$(kubectl -n "$NS" get pod web -o jsonpath='{.status.podIP}')"
kubectl -n "$NS" run thu-tai --restart=Never --image=busybox:1.37 \
  --overrides='{"apiVersion":"v1","spec":{"nodeSelector":{"kubernetes.io/hostname":"lab-k8s-worker2"}}}' \
  --env="API=$API_IP" --env="WEB=$WEB_IP" -- sh -c '
    end=$(( $(date +%s) + 75 )); n=0
    while [ "$(date +%s)" -lt "$end" ]; do
      wget -qO- "http://$API:3000/books" >/dev/null && wget -qO- "http://$WEB:8080/" >/dev/null && n=$((n + 1))
      sleep 0.2
    done
    echo "thu-tai: $n vong"'
kubectl -n "$NS" wait --for=condition=Ready pod/thu-tai --timeout=120s
sleep 10; L0="$(usage)"; sleep 30; L1="$(usage)"
MEM="$(for X in $CS; do
  echo "$X $(cg "${X%/*}" "${X#*/}" memory.current) $(cg "${X%/*}" "${X#*/}" memory.peak 2>/dev/null)"
done)"
kubectl -n "$NS" wait --for=jsonpath='{.status.phase}'=Succeeded pod/thu-tai --timeout=120s
kubectl -n "$NS" logs thu-tai > "$EV/b1-tai.txt"; cat "$EV/b1-tai.txt"
kubectl -n "$NS" delete pod thu-tai --wait=true

# 6. Bảng số đo
{ echo '# thanh_phan cpu_nghi_m cpu_tai_m mem_tai_mib mem_peak_mib'
  for X in $CS; do
    a0="$(val "$I0" "$X")"; a1="$(val "$I1" "$X")"; b0="$(val "$L0" "$X")"; b1="$(val "$L1" "$X")"
    cur="$(echo "$MEM" | awk -v k="$X" '$1==k{print $2}')"; pk="$(echo "$MEM" | awk -v k="$X" '$1==k{print $3}')"
    if [ -n "$pk" ]; then pk="$(( (pk + 1048575) / 1048576 ))"; else pk=NA; fi
    echo "$X $(( (${a1:-0} - ${a0:-0}) / 30000 )) $(( (${b1:-0} - ${b0:-0}) / 30000 )) $(( (${cur:-0} + 1048575) / 1048576 )) $pk"
  done; } > "$EV/b1-do-dac.txt"
cat "$EV/b1-do-dac.txt"
[ "$(grep -cE '^[a-z]+/[a-z-]+ [0-9]+ [0-9]+ [1-9][0-9]* ([0-9]+|NA)$' "$EV/b1-do-dac.txt")" -eq 4 ] \
  && echo 'PASS: b1-do-dac.txt du bon container, moi dong du so'
V="$(sed -n 's/^thu-tai: \([0-9]*\) vong$/\1/p' "$EV/b1-tai.txt")"
{ [ "${V:-0}" -ge 100 ] && ! kubectl -n "$NS" get pod thu-tai >/dev/null 2>&1; } \
  && echo "PASS: Pod tai chay $V vong va da bi xoa"
```

Số đo CPU là trung bình trên 30 giây; memory làm tròn **lên** MiB. Kết quả khác nhau giữa các lần chạy
và giữa các máy — đó là lý do ADR phải ghi lại chính số đo của bạn chứ không chép số mẫu.

</details>

<details>
<summary>Lời giải tham khảo</summary>

Đọc tay, đúng định dạng của yêu cầu 3:

```bash
ID="$(kubectl -n bookstore get pod db -o jsonpath='{.status.containerStatuses[?(@.name=="postgres")].containerID}')"
ssh lab-k8s-worker2 "D=\$(find /sys/fs/cgroup/kubepods.slice -maxdepth 3 -type d -name 'cri-containerd-${ID#containerd://}.scope'); \
  for f in cpu.max memory.max memory.current; do echo \"\$f: \$(cat \$D/\$f)\"; done; \
  awk '\$1==\"usage_usec\"{print \"usage_usec: \" \$2}' \$D/cpu.stat" | tee ~/lab-evidence/3c/b1-tay.txt
```

Kết quả mẫu: `cpu.max: max 100000`, `memory.max: max`, `memory.current: 41893888`,
`usage_usec: 2318841`. Pod chưa khai tài nguyên nào nên thuộc QoS `BestEffort`, thư mục nằm dưới
`kubepods-besteffort.slice`.

Ý chính cho mục *Bối cảnh* của ADR 0018:

1. Scheduler so với **`allocatable`** — bài 110: lượng khả dụng cho Pod nhỏ hơn dung lượng node vì
   daemon hệ thống dùng một phần. Tổng request tối đa trên worker2 là đúng cột `CPU_ALLOC` trong
   `b1-allocatable.txt`; đó cũng là trần của ngân sách cả dự án trên một node.
2. Không limit thì container dùng được toàn bộ CPU và memory còn trống của node (bài 110, 263, 264);
   bài 264 thêm: khi OOM xảy ra, container không có giới hạn tài nguyên có khả năng bị kill cao hơn.
3. Request nên phản ánh mức dùng **bình thường**; lúc nghỉ gần như bằng 0 nên không nói lên gì. Một
   lần đo với tải giả chỉ là một mẫu — Lab 11a cài metrics-server để theo dõi liên tục, Lab 11b dùng
   chính `requests` làm mẫu số cho HPA.
4. `128Mi` = 134217728 byte, `128M` = 128000000 byte (bài 264, mục *Đơn vị memory*). `memory.max`
   ghi đúng số byte của limit, nên dùng hậu tố nhị phân thì so với cgroup dễ hơn.

</details>

## B2. [BS-3C-02] Khai báo requests và limits cho mọi container

**Loại:** Tính năng — trả `TD-05`.

**Bối cảnh:** `TD-05` mở từ sprint 1. Yêu cầu [`R3`](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới)
của hệ thống mới đòi mỗi thành phần có tài nguyên khai báo. B1 đã cho số đo; ticket này biến số đo
thành ngân sách, chọn QoS cho từng thành phần, và chứng minh từng con số đã thành ràng buộc của
kernel trên node.

**Yêu cầu:**

1. Mọi container của ba Pod — `postgres`; `wait-for-db`, `postgrest`; `render-html`,
   `status-writer`, `nginx` — có cả request lẫn limit cho cả CPU lẫn memory.
2. `db` thuộc QoS `Guaranteed`; `api` và `web` thuộc `Burstable`.
3. Con số chốt từ `b1-do-dac.txt` theo ba luật, ghi trong ADR 0018: (a) request memory **không thấp
   hơn** memory đo dưới tải; (b) limit memory **ít nhất gấp đôi** memory đo dưới tải; (c) request CPU
   **không thấp hơn** CPU đo dưới tải. Init container không đo được — chạy xong là cgroup của nó biến
   mất — nên chọn bằng lập luận, và phải hoàn thành mà không bị kill.
4. Bảng [3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch) là điểm xuất phát; được lệch
   khỏi nó nếu số đo đòi hỏi, và ghi lý do.
5. Chứng minh trên node, với bốn container chạy lâu: limit CPU thành trần thời gian CPU, limit memory
   thành trần bộ nhớ **đúng từng byte**; container có request CPU lớn hơn được trọng số CPU lớn hơn.
6. `docs/tech-debt.md`: `TD-05` chuyển sang đã trả, trỏ tới bằng chứng.
7. Hệ thống dựng lại từ repo; `tests/smoke.sh` của Lab 3b vẫn PASS.

**Ràng buộc:** chỉ thêm khối tài nguyên, không đổi gì khác trong spec. Pod trần không đổi tài nguyên
qua đường cập nhật thường — `teardown` rồi `deploy`. Không dùng LimitRange để điền mặc định (Lab 7b).

**Đọc lại:**

- Bài [110](../110-manage-resources-containers-vi.md):
  [*Yêu cầu và giới hạn*](../110-manage-resources-containers-vi.md#requests-and-limits) — kể cả ghi
  chú sao chép limit thành request;
  [*Yêu cầu và giới hạn tài nguyên của Pod và container*](../110-manage-resources-containers-vi.md#resource-requests-and-limits-of-pod-and-container);
  [*Đơn vị tài nguyên CPU*](../110-manage-resources-containers-vi.md#meaning-of-cpu),
  [*bộ nhớ*](../110-manage-resources-containers-vi.md#meaning-of-memory);
  [*Cách Kubernetes áp dụng request và limit*](../110-manage-resources-containers-vi.md#how-pods-with-resource-limits-are-run)
  — request CPU là trọng số.
- Bài [54](../54-pod-qos-vi.md): [*Các lớp chất lượng dịch vụ*](../54-pod-qos-vi.md#quality-of-service-classes)
  và ba mục *Tiêu chí*.
- Bài [288](../288-quality-service-pod-vi.md):
  [*Tạo một Pod được gán QoS class Guaranteed*](../288-quality-service-pod-vi.md#create-a-pod-that-gets-assigned-a-qos-class-of-guaranteed)
  — câu về init container.
- Bài [263](../263-assign-cpu-resource-vi.md) và [264](../264-assign-memory-resource-vi.md), mục
  *Động lực cho … request và limit*.

**Bàn giao:** `k8s/app/db/pod.yaml`, `k8s/app/api/pod.yaml.tpl`, `k8s/app/web/pod.yaml` đã cập nhật;
`docs/adr/0018-ngan-sach-tai-nguyen.md` (đủ các mục); `docs/adr/0019-qos-tung-thanh-phan.md`;
`docs/tech-debt.md`; `~/lab-evidence/3c/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` hệ thống dựng lại từ repo; hai init container hoàn thành, không bị kill.
- `PASS:` sáu container đều có đủ request và limit cho CPU lẫn memory.
- `PASS:` `db` là `Guaranteed`; `api`, `web` là `Burstable`.
- `PASS:` `cpu.max` và `memory.max` trên node khớp limit của bốn container chạy lâu.
- `PASS:` container có request CPU lớn hơn có `cpu.weight` lớn hơn.
- `PASS:` con số chốt thỏa ba luật so với `b1-do-dac.txt`.
- `PASS:` có ADR 0018, ADR 0019; `docs/tech-debt.md` ghi `TD-05` đã trả; smoke của Lab 3b PASS.

**Câu hỏi thiết kế:**

Trong `docs/adr/0018-ngan-sach-tai-nguyen.md`:

1. Với mỗi container: con số nào lấy từ số đo, con số nào từ bảng 3.6, con số nào bạn nâng lên — vì
   sao?
2. Vì sao luật (b) cho memory gấp đôi, trong khi CPU không có luật tương tự cho limit? Gợi ý: limit
   memory được thực thi theo kiểu nào, limit CPU theo kiểu nào?
3. Cộng request của cả dự án. So với `allocatable` của **một** worker, nếu mất một worker thì phần còn
   lại có xếp đủ không — và câu hỏi đó trả lời bằng request hay bằng mức dùng thật?

Trong `docs/adr/0019-qos-tung-thanh-phan.md`:

1. Vì sao `db` phải là `Guaranteed`? Nối với thứ tự trục xuất của bài 54 và với `TD-04`. `Guaranteed`
   có nghĩa là không bao giờ bị kill không?
2. Vì sao `web`, `api` chọn `Burstable` chứ không `Guaranteed`? Được gì, mất gì — và `TD-01` làm cái
   giá đó nặng hơn ra sao?
3. Init container `wait-for-db` chỉ chạy vài giây. Vì sao vẫn phải khai tài nguyên cho nó?

<details>
<summary>Gợi ý</summary>

1. Khối `resources` có hai map `requests` và `limits`, mỗi map có `cpu` và `memory`. Sidecar
   `status-writer` khai trong `initContainers`, nên khối tài nguyên của nó cũng nằm ở đó.
2. `Guaranteed` đòi mọi container — kể cả init — có request bằng limit ở cả CPU lẫn memory. `db` chỉ có
   một container nên dễ đạt; `api` và `web` chỉ cần một container có request khác limit là đủ
   `Burstable`.
3. Đọc trước một giá trị trong b1: `awk '$1=="db/postgres"' ~/lab-evidence/3c/b1-do-dac.txt`.
4. `deploy.sh` báo `pod updates may not change fields other than ...` nghĩa là Pod cũ còn chạy với tài
   nguyên cũ: `./scripts/teardown.sh && ./scripts/deploy.sh`.
5. `500m` thành `cpu.max` là `50000 100000` — quota trên chu kỳ 100 ms.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3c
cd ~/bookstore
./scripts/teardown.sh > "$EV/b2-teardown.txt" 2>&1 && ./scripts/deploy.sh > "$EV/b2-deploy.txt" 2>&1 \
  && echo 'PASS: he thong dung lai tu repo'
cg() {   # cg <pod> <container> <file>
  local n id
  n="$(kubectl -n "$NS" get pod "$1" -o jsonpath='{.spec.nodeName}')"
  id="$(kubectl -n "$NS" get pod "$1" -o jsonpath="{.status.containerStatuses[?(@.name==\"$2\")].containerID}{.status.initContainerStatuses[?(@.name==\"$2\")].containerID}")"
  ssh -n "$n" "find /sys/fs/cgroup/kubepods.slice -maxdepth 3 -type d -name 'cri-containerd-${id#containerd://}.scope' -exec cat {}/$3 \;"
}
mc() { case "$1" in *m) echo "${1%m}" ;; *) awk -v v="$1" 'BEGIN{printf "%d", v*1000}' ;; esac; }
by() { case "$1" in
  *Ki) echo $(( ${1%Ki} * 1024 )) ;; *Mi) echo $(( ${1%Mi} * 1048576 )) ;; *Gi) echo $(( ${1%Gi} * 1073741824 )) ;;
  *k) echo $(( ${1%k} * 1000 )) ;; *M) echo $(( ${1%M} * 1000000 )) ;; *G) echo $(( ${1%G} * 1000000000 )) ;;
  *) echo "${1:-0}" ;; esac; }
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2"; }
res() { jp "$1" "{.spec.containers[?(@.name==\"$2\")].resources.$3}{.spec.initContainers[?(@.name==\"$2\")].resources.$3}"; }

{ [ "$(jp api '{.status.initContainerStatuses[?(@.name=="wait-for-db")].state.terminated.reason}')" = 'Completed' ] && \
  [ "$(jp web '{.status.initContainerStatuses[?(@.name=="render-html")].state.terminated.reason}')" = 'Completed' ]; } \
  && echo 'PASS: hai init container hoan thanh, khong bi kill'

kubectl -n "$NS" get pod db api web -o jsonpath='{range .items[*]}{range .spec.initContainers[*]}{.name}={.resources.requests.cpu},{.resources.requests.memory},{.resources.limits.cpu},{.resources.limits.memory}{"\n"}{end}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.requests.memory},{.resources.limits.cpu},{.resources.limits.memory}{"\n"}{end}{end}' \
  > "$EV/b2-tai-nguyen.txt"
cat "$EV/b2-tai-nguyen.txt"
{ [ "$(grep -c '=' "$EV/b2-tai-nguyen.txt")" -eq 6 ] && ! grep -Eq '=,|,,|,$' "$EV/b2-tai-nguyen.txt"; } \
  && echo 'PASS: sau container deu co request va limit cho CPU lan memory'

[ "$(jp db '{.status.qosClass}')/$(jp api '{.status.qosClass}')/$(jp web '{.status.qosClass}')" \
  = 'Guaranteed/Burstable/Burstable' ] && echo 'PASS: db Guaranteed; api, web Burstable'

: > "$EV/b2-cgroup.txt"; BAD=0
for X in db/postgres api/postgrest web/nginx web/status-writer; do
  P="${X%/*}"; C="${X#*/}"
  LC="$(mc "$(res "$P" "$C" limits.cpu)")"; LM="$(by "$(res "$P" "$C" limits.memory)")"
  CM="$(cg "$P" "$C" cpu.max)"; MM="$(cg "$P" "$C" memory.max)"
  echo "$X limits=${LC}m/${LM}B cpu.max='$CM' memory.max=$MM" | tee -a "$EV/b2-cgroup.txt"
  { [ "$(echo "$CM" | awk '{printf "%d", $1*1000/$2}')" = "$LC" ] && [ "$MM" = "$LM" ]; } || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: limits.cpu thanh cpu.max, limits.memory thanh memory.max tren node'

RP="$(mc "$(res db postgres requests.cpu)")"; RN="$(mc "$(res web nginx requests.cpu)")"
WP="$(cg db postgres cpu.weight)"; WN="$(cg web nginx cpu.weight)"
echo "request CPU: postgres=${RP}m nginx=${RN}m ; cpu.weight: postgres=$WP nginx=$WN" | tee -a "$EV/b2-cgroup.txt"
{ { [ "$RP" -gt "$RN" ] && [ "${WP:-0}" -gt "${WN:-0}" ]; } || { [ "$RP" -lt "$RN" ] && [ "${WP:-0}" -lt "${WN:-0}" ]; }; } \
  && echo 'PASS: request CPU lon hon thi cpu.weight lon hon'

BAD=0
while read -r X CN CT MT PK; do
  case "$X" in '#'*|'') continue ;; esac
  P="${X%/*}"; C="${X#*/}"
  RC="$(mc "$(res "$P" "$C" requests.cpu)")"
  RM=$(( $(by "$(res "$P" "$C" requests.memory)") / 1048576 )); LM=$(( $(by "$(res "$P" "$C" limits.memory)") / 1048576 ))
  if [ "$RM" -ge "$MT" ] && [ "$LM" -ge $(( 2 * MT )) ] && [ "$RC" -ge "$CT" ]; then
    echo "$X: request ${RC}m/${RM}Mi, limit memory ${LM}Mi — dat ba luat (do duoc ${CT}m, ${MT}Mi)"
  else
    echo "FAIL: $X vi pham luat (do duoc ${CT}m, ${MT}Mi; request ${RC}m/${RM}Mi; limit memory ${LM}Mi)"; BAD=1
  fi
done < "$EV/b1-do-dac.txt"
[ "$BAD" -eq 0 ] && echo 'PASS: con so chot thoa ba luat so voi so do B1'

{ ls docs/adr/*-ngan-sach-tai-nguyen.md docs/adr/*-qos-tung-thanh-phan.md >/dev/null 2>&1 && \
  grep -E 'TD-05.*đã trả' docs/tech-debt.md >/dev/null; } && echo 'PASS: co ADR 0018, 0019; TD-05 da tra'
./tests/smoke.sh > "$EV/b2-smoke.txt" 2>&1 && echo 'PASS: smoke cua Lab 3b van PASS'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Con số dưới đây là bảng 3.6 cộng hai init container; số đo của bạn có thể đòi khác — giữ phần của
bạn nếu nó thỏa ba luật. Mọi khối khác của manifest giữ như cuối Lab 3b.

Trong `k8s/app/db/pod.yaml`, container `postgres`:

```yaml
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

Trong `k8s/app/api/pod.yaml.tpl`:

```yaml
  initContainers:
  - name: wait-for-db
    # ... image, command, env giữ nguyên
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 100m, memory: 32Mi}
  containers:
  - name: postgrest
    # ... envFrom, env, ports, ba probe, securityContext giữ nguyên
    resources:
      requests: {cpu: 50m, memory: 64Mi}
      limits: {cpu: 500m, memory: 256Mi}
```

Trong `k8s/app/web/pod.yaml`:

```yaml
  initContainers:
  - name: render-html
    # ...
    resources:
      requests: {cpu: 10m, memory: 8Mi}
      limits: {cpu: 50m, memory: 16Mi}
  - name: status-writer
    restartPolicy: Always
    # ...
    resources:
      requests: {cpu: 10m, memory: 8Mi}
      limits: {cpu: 50m, memory: 16Mi}
  containers:
  - name: nginx
    # ...
    resources:
      requests: {cpu: 50m, memory: 32Mi}
      limits: {cpu: 200m, memory: 64Mi}
```

```bash
cd ~/bookstore && ./scripts/teardown.sh && ./scripts/deploy.sh
```

Dòng `TD-05` trong `docs/tech-debt.md`:

```markdown
| TD-05 | Chưa có requests/limits | 3a | 3c | đã trả (3c) | ~/lab-evidence/3c/b2-tai-nguyen.txt, b2-cgroup.txt |
```

Ý chính cho ADR 0018: (1) bảng ba cột *đo được — gợi ý 3.6 — chốt*; (2) limit memory thực thi **bị
động** bằng OOM kill nên cần khoảng đệm cho đỉnh ngắn hạn, còn limit CPU chỉ điều tiết, vượt thì chậm
chứ không chết — khoảng đệm CPU là chuyện hiệu năng, không phải chuyện sống còn; (3) tổng request của
cả dự án theo bảng trên khoảng 630m CPU, 640Mi memory — vừa một worker, nên mất một worker vẫn xếp
được nếu Pod được tạo lại (việc đó chưa ai làm: `TD-01`). Câu hỏi này trả lời bằng **request**, vì
scheduler chỉ nhìn request.

Ý chính cho ADR 0019:

1. Bài 54: khi node cạn tài nguyên, `BestEffort` bị trục xuất trước, rồi `Burstable`, `Guaranteed`
   sau cùng. `db` giữ dữ liệu trên `emptyDir` (`TD-04`): bị trục xuất là mất dữ liệu. `Guaranteed`
   không phải miễn nhiễm — vượt limit của chính nó vẫn bị kill, và vẫn có thể bị chiếm chỗ.
2. `Burstable` cho `web`, `api` giữ chỗ nhỏ mà vẫn được dùng thêm khi node rảnh. Cái giá: bị trục xuất
   trước `db`; và vì là Pod trần, Pod bị trục xuất không ai tạo lại cho tới lần `deploy.sh` sau.
3. Init container cũng bị tính vào QoS (bài 288) và vào request hiệu dụng (bài 50); thiếu khai báo thì
   nó chạy không trần trong lúc khởi động, và đủ làm một Pod mong muốn `Guaranteed` rớt hạng — B3 kiểm
   chứng.

</details>

## B3. [BS-3C-03] Điều tra: QoS và tài nguyên hiệu dụng khi có init container và sidecar

**Loại:** Điều tra — **trả nợ lab của Lab 3a** (request/limit hiệu dụng của init container và
sidecar). Đọc lại mục *Chia sẻ tài nguyên giữa các container* của bài
[50](../50-init-containers-vi.md#resource-sharing-within-containers) và bài
[51](../51-sidecar-containers-vi.md) **trước** khi làm.

**Bối cảnh:** hai trong ba Pod của dự án có container phụ: `api` có init container, `web` có cả init
container lẫn sidecar. Trước khi ai đó thêm một container phụ nữa, đội cần trả lời chắc hai câu: một
container phụ khai thiếu có làm `db` rớt khỏi `Guaranteed` không, và Pod `web` thực sự giữ bao nhiêu
chỗ trên node.

**Yêu cầu:**

1. **Dự đoán QoS.** Từ manifest thật trong repo, tạo sáu biến thể ở `~/lab-work/3c/qos/qos-<n>.yaml`,
   mỗi biến thể là một Pod tên `qos-<n>`. Ghi dự đoán vào `~/lab-work/3c/qos/du-doan.txt` — mỗi dòng
   `qos-<n>=<QoS>` — **trước** khi hỏi API server; rồi đối chiếu bằng API server mà không tạo Pod nào.

   | Pod | Bắt đầu từ | Thay đổi |
   | --- | --- | --- |
   | `qos-1` | `db` | thêm một init container `busybox` không khai tài nguyên |
   | `qos-2` | `db` | giữ limit, xóa hết request |
   | `qos-3` | `db` | xóa riêng limit memory |
   | `qos-4` | `web` | `render-html` và `nginx` request bằng limit ở cả CPU lẫn memory; `status-writer` không khai tài nguyên |
   | `qos-5` | `web` | như `qos-4`, nhưng `status-writer` cũng request bằng limit |
   | `qos-6` | `web` | xóa mọi khối tài nguyên |

2. **Limit hiệu dụng trên node.** Tính bằng tay limit CPU và memory hiệu dụng của Pod `web` và `api`
   từ manifest. Chứng minh cgroup **cấp Pod** của hai Pod đó trên node mang đúng hai con số ấy.
3. **Request hiệu dụng trong lập lịch.** Gọi F là CPU còn trống trên `lab-k8s-worker2` (allocatable
   trừ phần đã cam kết). Ba Pod thử nghiệm ghim worker2, mỗi Pod một container phụ và một container
   chính, tạo lần lượt — Pod trước bị xóa rồi mới tạo Pod sau:
   - `thu-init-lon`: container phụ là init container thường, xin 120% F; container chính xin 10m.
     Phải `Pending` vì thiếu CPU.
   - `thu-init`: container phụ là init container thường; cả hai container xin 60% F. Phải chạy được,
     và phần đã cam kết trên node tăng **đúng 60% F**, không phải 120%.
   - `thu-sidecar`: giống hệt `thu-init`, **chỉ khác** container phụ là sidecar. Phải `Pending` vì
     thiếu CPU.

   Viết hai file mẫu ở `~/lab-work/3c/`: `thu-init.yaml.tpl` (dùng cho cả `thu-init-lon` lẫn
   `thu-init`) và `thu-sidecar.yaml.tpl`, với ba chỗ giữ `__TEN__`, `__CPU_PHU__`, `__CPU_CHINH__`
   để script nghiệm thu điền số tính từ node thật.

**Ràng buộc:** biến thể QoS chỉ dùng để hỏi API server, không tạo ra Pod. Pod thử nghiệm của yêu cầu 3
không mang PriorityClass, không mang label dự án, và bị xóa khi xong.

**Đọc lại:**

- Bài [50](../50-init-containers-vi.md): [*Chia sẻ tài nguyên giữa các container*](../50-init-containers-vi.md#resource-sharing-within-containers),
  [*Init container và cgroup trên Linux*](../50-init-containers-vi.md#cgroups).
- Bài [51](../51-sidecar-containers-vi.md): mục *Chia sẻ tài nguyên giữa các container* — sidecar
  cộng vào tổng của container chính; [*Sidecar container và cgroup trên Linux*](../51-sidecar-containers-vi.md#cgroups).
- Bài [54](../54-pod-qos-vi.md): ba mục *Tiêu chí*.
- Bài [288](../288-quality-service-pod-vi.md): câu về init container ở mục Guaranteed; ghi chú tự gán
  request bằng limit; mục *Truy xuất QoS class của một Pod*.
- Bài [110](../110-manage-resources-containers-vi.md#how-pods-with-resource-requests-are-scheduled)
  — scheduler so tổng request với sức chứa node.

**Bàn giao:** `docs/adr/0020-tai-nguyen-hieu-dung.md`; `~/lab-evidence/3c/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` dự đoán QoS khớp `status.qosClass` mà API server trả về cho cả sáu biến thể.
- `PASS:` sáu biến thể dựng đúng yêu cầu: `qos-1` `Burstable`, `qos-2` `Guaranteed`, `qos-3`
  `Burstable`, `qos-4` `Burstable`, `qos-5` `Guaranteed`, `qos-6` `BestEffort`.
- `PASS:` `memory.max` và `cpu.max` cấp Pod của `web` và `api` bằng limit hiệu dụng tính tay.
- `PASS:` `thu-init-lon` `Pending` với `Insufficient cpu`.
- `PASS:` `thu-init` chạy; phần cam kết trên node tăng đúng 60% F.
- `PASS:` `thu-sidecar` `Pending` với `Insufficient cpu`; không còn Pod `thu-*`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0020-tai-nguyen-hieu-dung.md`:

1. `qos-1`: một init container chạy vài giây và không khai tài nguyên làm gì với `db`? Viết luật cho
   mọi container tương lai của `db`.
2. `qos-2` ra `Guaranteed` dù không khai request, còn `qos-3` ra `Burstable` dù CPU vẫn chặt — mỗi kết
   quả do quy tắc nào của bài 110, 288?
3. Request và limit hiệu dụng của `web` và `api` với số của bạn là bao nhiêu? Vì sao `render-html`
   không cộng vào còn `status-writer` thì cộng? Dẫn đúng câu của bài 50 và 51.
4. Tổng request **hiệu dụng** của ba Pod là bao nhiêu? Đó là con số Lab 7b dùng để đặt
   ResourceQuota — vì sao không phải tổng request của các container chính?

<details>
<summary>Gợi ý</summary>

1. Sao chép rồi đổi tên Pod: `sed 's/^  name: db$/  name: qos-1/' k8s/app/db/pod.yaml > ~/lab-work/3c/qos/qos-1.yaml`,
   rồi sửa tay phần còn lại.
2. API server tính QoS ngay lúc tạo Pod và ghi vào `status`; một lần tạo thử phía server trả về object
   đó mà không lưu: `kubectl create --dry-run=server -f <file> -o jsonpath='{.status.qosClass}'`.
3. Limit hiệu dụng = giá trị lớn hơn giữa *tổng các container chính và sidecar* và *giá trị lớn nhất
   của các init container thường*. Cgroup cấp Pod nằm ở thư mục `kubepods*-pod<uid>.slice`, trong đó
   `<uid>` của Pod viết dấu `_` thay cho `-`.
4. Phần đã cam kết trên node: dòng `cpu` dưới mục `Allocated resources` của
   `kubectl describe node lab-k8s-worker2`.
5. Sidecar là init container có `restartPolicy: Always` ở cấp container.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã viết sáu biến thể và hai file mẫu</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3c; Q=~/lab-work/3c/qos
# 1. QoS theo API server, không tạo Pod
: > "$EV/b3-qos.txt"
for F in "$Q"/qos-*.yaml; do
  echo "$(kubectl create --dry-run=server -f "$F" -o jsonpath='{.metadata.name}={.status.qosClass}' 2>&1)" >> "$EV/b3-qos.txt"
done
cat "$EV/b3-qos.txt"
diff <(grep -v '^#' "$Q/du-doan.txt" | sort) <(sort "$EV/b3-qos.txt") \
  && echo 'PASS: du doan QoS khop status.qosClass' || echo 'FAIL: co du doan sai — doc lai Tieu chi cua bai 54'
BAD=0
for E in qos-1=Burstable qos-2=Guaranteed qos-3=Burstable qos-4=Burstable qos-5=Guaranteed qos-6=BestEffort; do
  grep -qx "$E" "$EV/b3-qos.txt" || { echo "FAIL: bien the ${E%%=*} khong cho ${E##*=} — xem lai cach dung"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: sau bien the dung yeu cau'

# 2. Limit hiệu dụng = cgroup cấp Pod
cgp() {   # cgp <pod> <file>: đọc một file cgroup v2 cấp Pod, trên đúng node chạy Pod
  local n u
  n="$(kubectl -n "$NS" get pod "$1" -o jsonpath='{.spec.nodeName}')"
  u="$(kubectl -n "$NS" get pod "$1" -o jsonpath='{.metadata.uid}' | tr - _)"
  ssh -n "$n" "find /sys/fs/cgroup/kubepods.slice -maxdepth 2 -type d -name 'kubepods*-pod$u.slice' -exec cat {}/$2 \;"
}
mc() { case "$1" in *m) echo "${1%m}" ;; *) awk -v v="$1" 'BEGIN{printf "%d", v*1000}' ;; esac; }
by() { case "$1" in
  *Ki) echo $(( ${1%Ki} * 1024 )) ;; *Mi) echo $(( ${1%Mi} * 1048576 )) ;; *Gi) echo $(( ${1%Gi} * 1073741824 )) ;;
  *k) echo $(( ${1%k} * 1000 )) ;; *M) echo $(( ${1%M} * 1000000 )) ;; *G) echo $(( ${1%G} * 1000000000 )) ;;
  *) echo "${1:-0}" ;; esac; }
lim() { kubectl -n "$NS" get pod "$1" -o jsonpath="{.spec.containers[?(@.name==\"$2\")].resources.limits.$3}{.spec.initContainers[?(@.name==\"$2\")].resources.limits.$3}"; }
max() { [ "$1" -gt "$2" ] && echo "$1" || echo "$2"; }
WM="$(max $(( $(by "$(lim web nginx memory)") + $(by "$(lim web status-writer memory)") )) "$(by "$(lim web render-html memory)")")"
WC="$(max $(( $(mc "$(lim web nginx cpu)") + $(mc "$(lim web status-writer cpu)") )) "$(mc "$(lim web render-html cpu)")")"
AM="$(max "$(by "$(lim api postgrest memory)")" "$(by "$(lim api wait-for-db memory)")")"
AC="$(max "$(mc "$(lim api postgrest cpu)")" "$(mc "$(lim api wait-for-db cpu)")")"
: > "$EV/b3-cgroup-pod.txt"
for X in "web $WM $WC" "api $AM $AC"; do
  set -- $X
  M="$(cgp "$1" memory.max)"; C="$(cgp "$1" cpu.max | awk '{printf "%d", $1*1000/$2}')"
  echo "$1: limit hieu dung ${3}m/${2}B ; cgroup cap Pod ${C}m/${M}B" | tee -a "$EV/b3-cgroup-pod.txt"
  { [ "$M" = "$2" ] && [ "$C" = "$3" ]; } && echo "PASS: cgroup cap Pod cua $1 mang dung limit hieu dung"
done

# 3. Request hiệu dụng trong lập lịch, trên worker2
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2" 2>/dev/null; }
used() { mc "$(kubectl describe node lab-k8s-worker2 | awk '/^Allocated resources:/{f=1} f && $1=="cpu"{print $2; exit}')"; }
r() { sed -e "s/__TEN__/$2/" -e "s/__CPU_PHU__/$3/" -e "s/__CPU_CHINH__/$4/" ~/lab-work/3c/"$1".yaml.tpl; }
pending() {
  for i in $(seq 1 24); do [ "$(jp "$1" '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ] && break; sleep 5; done
  kubectl -n "$NS" describe pod "$1" > "$EV/b3-$1.txt"
  [ "$(jp "$1" '{.status.phase}')" = 'Pending' ] && grep -q 'Insufficient cpu' "$EV/b3-$1.txt"
}
ALLOC="$(mc "$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.allocatable.cpu}')")"
U0="$(used)"; F=$(( ALLOC - U0 )); X=$(( F * 60 / 100 )); Y=$(( F * 120 / 100 ))
echo "worker2: allocatable=${ALLOC}m da cam ket=${U0}m F=${F}m X=${X}m Y=${Y}m" | tee "$EV/b3-x.txt"

r thu-init thu-init-lon "${Y}m" 10m | kubectl apply -f -
pending thu-init-lon && echo 'PASS: thu-init-lon Pending — request cua init container cung phai co cho'
kubectl -n "$NS" delete pod thu-init-lon --wait=true

r thu-init thu-init "${X}m" "${X}m" | kubectl apply -f -
kubectl -n "$NS" wait --for=condition=Ready pod/thu-init --timeout=180s && U1="$(used)"
echo "sau thu-init: da cam ket=${U1}m (tang $(( U1 - U0 ))m)" | tee -a "$EV/b3-x.txt"
[ $(( U1 - U0 )) -eq "$X" ] && echo 'PASS: thu-init chay va chiem dung X — init thuong khong cong vao container chinh'
kubectl -n "$NS" delete pod thu-init --wait=true

r thu-sidecar thu-sidecar "${X}m" "${X}m" | kubectl apply -f -
pending thu-sidecar && echo 'PASS: thu-sidecar Pending — sidecar cong vao container chinh, 2X vuot F'
kubectl -n "$NS" delete pod thu-sidecar --wait=true
kubectl -n "$NS" get pod -o name | grep -q '/thu-' && echo 'FAIL: con Pod thu-' || echo 'PASS: da xoa Pod thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Biến thể: `qos-1` thêm vào `db` khối

```yaml
  initContainers:
  - name: in-ra
    image: busybox:1.37
    command: ["sh", "-c", "echo khoi tao"]
```

`qos-2` xóa hai dòng `requests` (và hai giá trị bên dưới); `qos-3` xóa dòng `memory` trong `limits`;
`qos-4`, `qos-5` cho `render-html`, `nginx` (và ở `qos-5` cả `status-writer`) khối
`requests` giống hệt `limits`; `qos-6` xóa mọi khối `resources` của `web`. Dự đoán đúng:

```text
qos-1=Burstable
qos-2=Guaranteed
qos-3=Burstable
qos-4=Burstable
qos-5=Guaranteed
qos-6=BestEffort
```

```yaml
# ~/lab-work/3c/thu-init.yaml.tpl — thu-sidecar.yaml.tpl giống hệt, chỉ khác container "phu"
# có thêm "restartPolicy: Always" và chạy "sleep 3600" thay cho lệnh in rồi thoát
apiVersion: v1
kind: Pod
metadata:
  name: __TEN__
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  terminationGracePeriodSeconds: 5
  initContainers:
  - name: phu
    image: busybox:1.37
    command: ["sh", "-c", "echo phu: xong"]
    resources:
      requests: {cpu: __CPU_PHU__, memory: 16Mi}
      limits: {cpu: __CPU_PHU__, memory: 16Mi}
  containers:
  - name: chinh
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: __CPU_CHINH__, memory: 16Mi}
      limits: {cpu: __CPU_CHINH__, memory: 16Mi}
```

Ý chính cho ADR 0020:

1. Bài 288: các ràng buộc của `Guaranteed` áp dụng như nhau cho init container. Một init container
   không khai gì làm cả `db` rớt xuống `Burstable` — luật: mọi container của `db`, kể cả init và sidecar
   tương lai, phải request bằng limit ở cả CPU lẫn memory.
2. `qos-2`: chỉ có limit thì Kubernetes sao chép limit thành request (ghi chú bài 110, 288) — đủ bốn giá
   trị, bằng nhau. `qos-3`: chiều ngược lại không có; thiếu limit memory là trượt `Guaranteed`.
3. Với số của lời giải B2: `web` request hiệu dụng là max(50m + 10m, 10m) = 60m CPU, max(32Mi + 8Mi,
   8Mi) = 40Mi; limit hiệu dụng 250m, 80Mi. `api`: request 50m, 64Mi; limit 500m, 256Mi. Bài 50: giá trị
   cao nhất của các init container là *request khởi tạo hiệu dụng*, và request của Pod là giá trị lớn
   hơn giữa nó và tổng container chính — `render-html` chạy xong trước khi `nginx` chạy nên chỉ cần
   chỗ tạm thời. Bài 51: sidecar nằm trong tổng của các container **không phải init**, vì nó chạy song
   song với container chính suốt đời Pod. `thu-init` và `thu-sidecar` khác đúng một trường mà một Pod
   chạy, một Pod `Pending`.
4. Bài 50: quota và limit áp dụng dựa trên request và limit **hiệu dụng** của Pod. Tổng theo lời giải:
   500m + 50m + 60m = 610m CPU, 512Mi + 64Mi + 40Mi = 616Mi memory request.

</details>

## B4. [BS-3C-04] Diễn tập: vượt trần

**Loại:** Diễn tập sự cố — mọi Pod diễn tập ghim `lab-k8s-worker2`.

**Bối cảnh:** sau B2, mỗi thành phần có trần. Trước khi tin vào trần đó, người trực phải nhận ra ba
kiểu hỏng khác hẳn nhau khi một thành phần chạm nó: chậm mà không chết, chết một container mà Pod
vẫn còn, và không bao giờ được chạy. Ticket dựng lại cả ba bằng Pod thử nghiệm, không đụng vào hệ
thống thật.

**Yêu cầu:** viết bốn Pod diễn tập ở `~/lab-work/3c/` — `thu-throttle.yaml`, `thu-oom.yaml` và hai
file mẫu ghi trong bảng — rồi chạy script nghiệm thu: nó tạo Pod, ghi nhận trạng thái và xóa.

| Pod | Mô tả | Phải quan sát được |
| --- | --- | --- |
| `thu-throttle` | container `burn` (`busybox`) chạy vòng lặp bận; request CPU 50m, limit 100m | `Running`, không khởi động lại lần nào; kernel ghi nhận bị điều tiết; CPU dùng không vượt limit quá 20% |
| `thu-oom` | mô phỏng Pod `web`: container chính `nginx` (image `nginx-unprivileged` của dự án) và sidecar `status-writer` (`busybox`, limit memory 16Mi) — sidecar chờ vài giây rồi cấp phát 128MiB | sidecar `OOMKilled`, mã 137; `nginx` cùng Pod vẫn chạy, không khởi động lại |
| `thu-qua-suc` | file mẫu `thu-qua-suc.yaml.tpl`, `busybox` xin memory bằng `allocatable` của worker2 cộng 1Gi (chỗ giữ `__MEM__`) | `Pending`, `PodScheduled=False` lý do `Unschedulable`, `Insufficient memory` |
| `thu-het-cho` | file mẫu `thu-het-cho.yaml.tpl`, `busybox` xin CPU **đúng bằng** `allocatable` của worker2 (chỗ giữ `__CPU__`) | `Pending`, `Insufficient cpu` — dù request vừa khít một node trống và node gần như rảnh |

Ghi postmortem `docs/postmortem/3c-vuot-tran.md`: với mỗi kịch bản, triệu chứng người dùng thấy, dấu
vết người trực đọc được ở đâu, và nó thuộc vế `requests` hay vế `limits`.

**Ràng buộc:** không sửa manifest của hệ thống trong repo. Pod diễn tập không PriorityClass, không label
dự án, xóa hết khi xong.

**Đọc lại:**

- Bài [110](../110-manage-resources-containers-vi.md):
  [*Yêu cầu và giới hạn*](../110-manage-resources-containers-vi.md#requests-and-limits) — limit CPU là
  giới hạn cứng, limit memory thực thi bị động;
  [*Khắc phục sự cố*](../110-manage-resources-containers-vi.md#troubleshooting) — cả hai mục con.
- Bài [263](../263-assign-cpu-resource-vi.md): mục *Chỉ định CPU request và CPU limit*, mục *Chỉ định
  một CPU request quá lớn so với các Node của bạn*.
- Bài [264](../264-assign-memory-resource-vi.md): mục *Vượt quá memory limit của Container*, mục *Chỉ
  định một memory request quá lớn so với các Node của bạn*.
- Bài [54](../54-pod-qos-vi.md#class-independent-behavior) — kill vì vượt limit không ảnh hưởng
  container khác; trục xuất thì cả Pod.

**Bàn giao:** `docs/postmortem/3c-vuot-tran.md`; `~/lab-evidence/3c/b4-*`.

**Nghiệm thu:**

- `PASS:` `thu-throttle` bị điều tiết (`nr_throttled` tăng), CPU trung bình không vượt 120% limit.
- `PASS:` `thu-throttle` vẫn `Running`, `restartCount` bằng 0.
- `PASS:` sidecar của `thu-oom` bị `OOMKilled`, mã thoát 137.
- `PASS:` `nginx` của `thu-oom` vẫn chạy, `restartCount` bằng 0.
- `PASS:` `thu-qua-suc` `Pending`, `FailedScheduling`, `Insufficient memory`.
- `PASS:` `thu-het-cho` `Pending`, `FailedScheduling`, `Insufficient cpu`.
- `PASS:` không còn Pod diễn tập.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. `thu-throttle` không chết. Nếu `postgrest` thật bị điều tiết nặng vì limit CPU quá thấp, probe nào
   của nó (Lab 3a B3) có thể thất bại trước, và hậu quả là gì? Vậy "limit CPU không bao giờ giết
   container" có đúng với cả hệ thống không?
2. `thu-oom`: nếu đây là `web` thật, người dùng thấy gì ở trang chủ và ở `/status.json`? Khác gì với
   trường hợp cả Pod bị trục xuất (bài 54)?
3. Một Pod hiện `OOMKilled`. Có phải node đã hết RAM? Bằng chứng nào phân biệt (bài 264)?
4. `thu-qua-suc` và `thu-het-cho`: thêm một worker giống hệt hai worker hiện có thì cứu được Pod nào?
   Scheduler đã so request với con số nào?

<details>
<summary>Gợi ý</summary>

1. Vòng lặp bận: `sh -c 'while :; do :; done'`.
2. Cấp phát bộ nhớ bằng `busybox`: `dd if=/dev/zero of=/dev/null bs=128M count=1` — `dd` giữ một bộ đệm
   đúng bằng `bs`. Cho sidecar `sleep` vài giây trước đó để kubelet kịp khởi động `nginx`.
3. Số đo điều tiết: dòng `nr_throttled` và `usage_usec` trong `cpu.stat` của container trên node.
4. Trạng thái của sidecar nằm trong `status.initContainerStatuses`, không phải `containerStatuses`.
5. File mẫu có chỗ giữ trong dấu nháy, ví dụ `memory: "__MEM__"`, để giá trị thay vào luôn là chuỗi.

</details>

<details>
<summary>Script nghiệm thu — script này chính là cuộc diễn tập; chạy sau khi đã viết bốn file</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3c; W=~/lab-work/3c
cg() {   # cg <pod> <container> <file>
  local n id
  n="$(kubectl -n "$NS" get pod "$1" -o jsonpath='{.spec.nodeName}')"
  id="$(kubectl -n "$NS" get pod "$1" -o jsonpath="{.status.containerStatuses[?(@.name==\"$2\")].containerID}{.status.initContainerStatuses[?(@.name==\"$2\")].containerID}")"
  ssh -n "$n" "find /sys/fs/cgroup/kubepods.slice -maxdepth 3 -type d -name 'cri-containerd-${id#containerd://}.scope' -exec cat {}/$3 \;"
}
mc() { case "$1" in *m) echo "${1%m}" ;; *) awk -v v="$1" 'BEGIN{printf "%d", v*1000}' ;; esac; }
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2" 2>/dev/null; }
st() { echo "$1" | awk -v k="$2" '$1==k{print $2}'; }

kubectl apply -f "$W/thu-throttle.yaml" -f "$W/thu-oom.yaml"
MA="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.allocatable.memory}')"
CA="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.allocatable.cpu}')"
sed "s/__MEM__/$(( ${MA%Ki} / 1024 + 1024 ))Mi/" "$W/thu-qua-suc.yaml.tpl" | kubectl apply -f -
sed "s/__CPU__/$CA/" "$W/thu-het-cho.yaml.tpl" | kubectl apply -f -

# 1. Throttle
kubectl -n "$NS" wait --for=condition=Ready pod/thu-throttle --timeout=180s
T0="$(cg thu-throttle burn cpu.stat)"; sleep 20; T1="$(cg thu-throttle burn cpu.stat)"
NT=$(( $(st "$T1" nr_throttled) - $(st "$T0" nr_throttled) ))
MS=$(( ( $(st "$T1" usage_usec) - $(st "$T0" usage_usec) ) / 20000 ))
LIM="$(mc "$(jp thu-throttle '{.spec.containers[0].resources.limits.cpu}')")"
echo "nr_throttled +$NT trong 20s; CPU trung binh ${MS}m; limit ${LIM}m" | tee "$EV/b4-throttle.txt"
{ [ "$NT" -gt 0 ] && [ "$MS" -le $(( LIM * 120 / 100 )) ]; } && echo 'PASS: thu-throttle bi dieu tiet, khong vuot limit'
{ [ "$(jp thu-throttle '{.status.phase}')" = 'Running' ] && [ "$(jp thu-throttle '{.status.containerStatuses[0].restartCount}')" = '0' ]; } \
  && echo 'PASS: thu-throttle van Running, khong khoi dong lai'

# 2. OOM của sidecar
SW='{.status.initContainerStatuses[?(@.name=="status-writer")]'
R=''
for i in $(seq 1 36); do
  R="$(jp thu-oom "$SW.lastState.terminated.reason}$SW.state.terminated.reason}")"
  case "$R" in *OOMKilled*) break ;; esac; sleep 5
done
kubectl -n "$NS" describe pod thu-oom > "$EV/b4-oom-describe.txt"
EC="$(jp thu-oom "$SW.lastState.terminated.exitCode}$SW.state.terminated.exitCode}")"
{ case "$R" in *OOMKilled*) true ;; *) false ;; esac && case "$EC" in *137*) true ;; *) false ;; esac; } \
  && echo 'PASS: sidecar vuot limit memory bi OOMKilled, ma 137'
{ [ "$(jp thu-oom '{.status.containerStatuses[?(@.name=="nginx")].restartCount}')" = '0' ] && \
  [ -n "$(jp thu-oom '{.status.containerStatuses[?(@.name=="nginx")].state.running.startedAt}')" ]; } \
  && echo 'PASS: nginx cung Pod van chay, khong khoi dong lai — kill o muc container'

# 3. Hai Pod không bao giờ được chạy
for X in thu-qua-suc:memory thu-het-cho:cpu; do
  N="${X%%:*}"; K="${X##*:}"
  for i in $(seq 1 24); do [ "$(jp "$N" '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ] && break; sleep 5; done
  kubectl -n "$NS" describe pod "$N" > "$EV/b4-$N.txt"
  { [ "$(jp "$N" '{.status.phase}')" = 'Pending' ] && grep -q 'FailedScheduling' "$EV/b4-$N.txt" && \
    grep -q "Insufficient $K" "$EV/b4-$N.txt"; } && echo "PASS: $N Pending, FailedScheduling, Insufficient $K"
done
ssh -n lab-k8s-worker2 cat /proc/loadavg | tee "$EV/b4-loadavg.txt"
kubectl describe node lab-k8s-worker2 | awk '/^Allocated resources:/,/^Events:/' | tee "$EV/b4-allocated.txt"

kubectl -n "$NS" delete pod thu-throttle thu-oom thu-qua-suc thu-het-cho --wait=true
kubectl -n "$NS" get pod -o name | grep -q '/thu-' && echo 'FAIL: con Pod dien tap' || echo 'PASS: da xoa Pod dien tap'
```

`loadavg` thấp cùng lúc với `CPU Requests` gần 100% trong `b4-allocated.txt` là bức tranh của câu
"mặc dù mức sử dụng thực tế rất thấp, scheduler vẫn từ chối" ở bài 110.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/3c/thu-throttle.yaml
apiVersion: v1
kind: Pod
metadata:
  name: thu-throttle
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  terminationGracePeriodSeconds: 1
  containers:
  - name: burn
    image: busybox:1.37
    command: ["sh", "-c", "while :; do :; done"]
    resources:
      requests: {cpu: 50m, memory: 16Mi}
      limits: {cpu: 100m, memory: 32Mi}
```

```yaml
# ~/lab-work/3c/thu-oom.yaml
apiVersion: v1
kind: Pod
metadata:
  name: thu-oom
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  terminationGracePeriodSeconds: 5
  initContainers:
  - name: status-writer
    image: busybox:1.37
    restartPolicy: Always
    command: ["sh", "-c", "sleep 15; dd if=/dev/zero of=/dev/null bs=128M count=1; sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: 8Mi}
      limits: {cpu: 50m, memory: 16Mi}
  containers:
  - name: nginx
    image: nginxinc/nginx-unprivileged:1.30.5-alpine
    ports:
    - containerPort: 8080
    resources:
      requests: {cpu: 50m, memory: 32Mi}
      limits: {cpu: 200m, memory: 64Mi}
```

```yaml
# ~/lab-work/3c/thu-qua-suc.yaml.tpl — thu-het-cho.yaml.tpl giống hệt, đổi tên, và xin
# cpu: "__CPU__" (cả request lẫn limit) cùng memory 16Mi
apiVersion: v1
kind: Pod
metadata:
  name: thu-qua-suc
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: "__MEM__"}
      limits: {cpu: 10m, memory: "__MEM__"}
```

Ý chính cho postmortem:

1. Bài 110: limit CPU là giới hạn cứng thực thi bằng điều tiết, không chấm dứt container. Nhưng một
   `postgrest` bị điều tiết nặng trả lời chậm: readiness `/ready` quá thời gian chờ thì Pod rời trạng
   thái Ready, liveness `/live` thất bại đủ số lần thì kubelet khởi động lại container. Limit CPU không
   giết trực tiếp, nhưng có thể giết **gián tiếp** qua probe — nên limit CPU phải có khoảng đệm.
2. Bài 54: container vượt limit bị kill và khởi động lại mà không ảnh hưởng container khác. Trang chủ
   vẫn phục vụ; `/status.json` đứng ở lần ghi cuối cho tới khi sidecar chạy lại. Trục xuất thì kết thúc
   mọi container của Pod — với Pod trần, không ai tạo lại.
3. Không nhất thiết: `OOMKilled` là cgroup **của container** chạm trần. Bài 264 chỉ dấu hiệu
   `Memory cgroup out of memory` trong sự kiện của node; `b4-loadavg.txt` và `allocatable` cho thấy
   node còn trống.
4. `thu-het-cho` — vì nó vừa một node trống, và node mới chưa có Pod nào cam kết. `thu-qua-suc` lớn
   hơn mọi node, thêm node giống hệt không cứu được (bài 110, mục *Khắc phục sự cố*). Cả hai lần
   scheduler đều so **request** với **allocatable trừ phần đã cam kết**, không nhìn mức dùng thật.

</details>

## B5. [BS-3C-05] Tăng CPU cho `api` tại chỗ

**Loại:** Tính năng — thao tác trên hệ thống đang chạy, kèm đồng bộ repo.

**Bối cảnh:** hiệu sách chạy khuyến mãi cuối tuần, lượng gọi `/books` tăng. `api` là Pod trần duy
nhất của thành phần (`TD-01`): tạo lại Pod là `api` ngừng phục vụ tới khi Pod mới Ready. Đội cần nâng
CPU cho `api` **mà không khởi động lại gì**, rồi đưa quyết định đó về repo để lần triển khai sau không
làm mất nó.

**Yêu cầu:**

1. Container `postgrest` khai tường minh cách xử lý khi đổi tài nguyên: đổi CPU **không** khởi động
   lại; đổi memory **thì** khởi động lại container. Triển khai lại để Pod mang khai báo đó.
2. Chạy phần 1 của script nghiệm thu để **ghi mốc** trước khi đổi.
3. Gấp đôi cả request lẫn limit CPU của `postgrest` trên Pod đang chạy: không tạo lại Pod, không khởi
   động lại container, không đổi memory. `api` vẫn Ready và `/books` vẫn trả dữ liệu.
4. Chạy phần 2 của script nghiệm thu: nó kiểm kết quả, **diễn tập một lần đổi bất khả thi** trên `api`
   rồi trả về, và thử năm ranh giới bằng dry-run phía server.
5. Đồng bộ repo về đúng con số mới; `./scripts/deploy.sh` chạy lại mà **không** tạo lại `api`. Chạy
   phần 3 của script.
6. Ghi `docs/adr/0021-resize-tai-cho.md`; cập nhật bảng *Quyết định* của ADR 0018 với con số mới.

**Ràng buộc:** sau khi ghi mốc, không teardown, không xóa `api`. Không đổi memory. Không dùng
`kubectl edit` hay `kubectl apply` thông thường lên Pod đang chạy để đổi tài nguyên.

**Đọc lại:**

- Bài [289](../289-resize-container-resources-vi.md): mục *Các khái niệm chính* — tài nguyên mong
  muốn, thực tế, đã cấp phát; [*Trạng thái resize của Pod*](../289-resize-container-resources-vi.md#pod-resize-status);
  [*Chính sách resize của container*](../289-resize-container-resources-vi.md#container-resize-policies);
  [*Các hạn chế*](../289-resize-container-resources-vi.md#limitations); *Ví dụ 1* và mục *Xử lý sự
  cố: Yêu cầu resize bất khả thi*.
- Bài [110](../110-manage-resources-containers-vi.md#pod-resize-inplace) — thay đổi kích thước tại
  chỗ; bài [54](../54-pod-qos-vi.md#class-independent-behavior) — QoS xác định lúc tạo và không đổi.
- Bài [288](../288-quality-service-pod-vi.md), ghi chú đầu bài — control plane từ chối resize làm đổi
  QoS.

**Bàn giao:** `k8s/app/api/pod.yaml.tpl` đã cập nhật; `docs/adr/0021-resize-tai-cho.md`;
`docs/adr/0018-ngan-sach-tai-nguyen.md`; `~/lab-evidence/3c/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` `postgrest` khai: CPU không khởi động lại, memory khởi động lại; đã ghi mốc.
- `PASS:` request và limit CPU trong `spec` gấp đôi mốc; `status.containerStatuses[].resources` bằng
  `spec`; `cpu.max` trên node đổi theo.
- `PASS:` cùng UID Pod, cùng `containerID`, `restartCount` không đổi; `api` vẫn `Burstable`; `/books`
  trả dữ liệu.
- `PASS:` đổi bất khả thi cho condition `PodResizePending` lý do `Infeasible`, `status` giữ giá trị cũ,
  không khởi động lại; trả về giá trị khả thi thì condition biến mất.
- `PASS:` năm ranh giới: đổi QoS của `db` bị từ chối; đổi init container thường bị từ chối; gỡ limit
  bị từ chối; đổi sidecar được chấp nhận; đổi tài nguyên qua đường cập nhật Pod thường bị từ chối.
- `PASS:` repo mang con số mới; `deploy.sh` chạy lại, `api` giữ nguyên UID.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0021-resize-tai-cho.md`:

1. Vì sao CPU không cần khởi động lại còn memory thì bạn chọn khởi động lại cho `postgrest`? Đổi cả hai
   trong một lệnh thì sao?
2. Sau lệnh đổi, nhìn `spec` thôi đã đủ kết luận chưa? Trường nào, file nào trên node mới là bằng
   chứng?
3. `Infeasible` khác `Deferred` thế nào, cái nào tự khỏi? Trong lúc một lần đổi còn treo, scheduler
   tính phần cam kết của node bằng con số nào — và vì sao điều đó làm cuộc diễn tập phải ngắn?
4. Có cách nào biến `api` thành `Guaranteed` bằng resize không? Nêu **hai** lý do.
5. Sau khi đổi tại chỗ, vì sao repo bắt buộc phải mang con số mới? Nếu quên, chuyện gì xảy ra ở lần
   `deploy.sh` kế tiếp, và ở lần `teardown`/`deploy` kế tiếp?

<details>
<summary>Gợi ý</summary>

1. Khai báo cách xử lý theo từng loại tài nguyên nằm trong container, một danh sách các cặp tên tài
   nguyên và chính sách khởi động lại. Pod trần không nhận khai báo mới qua `deploy.sh`:
   `./scripts/teardown.sh && ./scripts/deploy.sh`.
2. Đổi tài nguyên của Pod đang chạy đi qua subresource `resize`: `kubectl patch pod ... --subresource resize -p '<patch>'`.
   Patch chiến lược hợp nhất theo tên container:
   `{"spec":{"containers":[{"name":"postgrest","resources":{"requests":{"cpu":"..."},"limits":{"cpu":"..."}}}]}}`.
3. Client `kubectl` báo `invalid subresource` nghĩa là bạn đang dùng client cũ, không phải client của
   baseline trên master.
4. Đồng bộ repo xong, kiểm repo đọc ra giá trị gì mà không cần server:
   `sed 's/__DB_HOST__/x/' k8s/app/api/pod.yaml.tpl | kubectl create --dry-run=client -f - -o jsonpath='{.spec.containers[?(@.name=="postgrest")].resources}'`.

</details>

<details>
<summary>Script nghiệm thu — ba phần, chạy đúng thứ tự trong cùng một shell</summary>

Hàm dùng chung cho cả ba phần:

```bash
NS=bookstore; EV=~/lab-evidence/3c
cg() {   # cg <pod> <container> <file>
  local n id
  n="$(kubectl -n "$NS" get pod "$1" -o jsonpath='{.spec.nodeName}')"
  id="$(kubectl -n "$NS" get pod "$1" -o jsonpath="{.status.containerStatuses[?(@.name==\"$2\")].containerID}{.status.initContainerStatuses[?(@.name==\"$2\")].containerID}")"
  ssh -n "$n" "find /sys/fs/cgroup/kubepods.slice -maxdepth 3 -type d -name 'cri-containerd-${id#containerd://}.scope' -exec cat {}/$3 \;"
}
mc() { case "$1" in *m) echo "${1%m}" ;; *) awk -v v="$1" 'BEGIN{printf "%d", v*1000}' ;; esac; }
jp() { kubectl -n "$NS" get pod api -o jsonpath="$1"; }
v() { awk -v k="$1" '$1==k{print $2}' "$EV/b5-moc.txt"; }
C='{.spec.containers[?(@.name=="postgrest")]'
S='{.status.containerStatuses[?(@.name=="postgrest")]'
```

Phần 1 — ghi mốc, chạy **trước** khi đổi:

```bash
{ echo "uid $(jp '{.metadata.uid}')"
  echo "cid $(jp "$S.containerID}")"
  echo "rc $(jp "$S.restartCount}")"
  echo "req $(jp "$C.resources.requests.cpu}")"
  echo "lim $(jp "$C.resources.limits.cpu}")"
  echo "cpumax $(cg api postgrest cpu.max)"; } | tee "$EV/b5-moc.txt"
{ [ "$(jp "$C.resizePolicy[?(@.resourceName==\"cpu\")].restartPolicy}")" = 'NotRequired' ] && \
  [ "$(jp "$C.resizePolicy[?(@.resourceName==\"memory\")].restartPolicy}")" = 'RestartContainer' ]; } \
  && echo 'PASS: postgrest khai: doi CPU khong khoi dong lai, doi memory thi khoi dong lai'
grep -q '^cid containerd://' "$EV/b5-moc.txt" && echo 'PASS: da ghi moc truoc khi doi'
```

Phần 2 — chạy **sau** khi bạn đã gấp đôi CPU:

```bash
REQ0="$(mc "$(v req)")"; LIM0="$(mc "$(v lim)")"
REQ1="$(mc "$(jp "$C.resources.requests.cpu}")")"; LIM1="$(mc "$(jp "$C.resources.limits.cpu}")")"
AREQ="$(mc "$(jp "$S.resources.requests.cpu}")")"; ALIM="$(mc "$(jp "$S.resources.limits.cpu}")")"
QUOTA="$(cg api postgrest cpu.max | awk '{print $1}')"
echo "moc ${REQ0}m/${LIM0}m ; spec ${REQ1}m/${LIM1}m ; status ${AREQ}m/${ALIM}m ; cpu.max quota $QUOTA" | tee "$EV/b5-sau.txt"
{ [ "$REQ1" -eq $(( REQ0 * 2 )) ] && [ "$LIM1" -eq $(( LIM0 * 2 )) ] && [ "$AREQ" -eq "$REQ1" ] && \
  [ "$ALIM" -eq "$LIM1" ] && [ "$QUOTA" = "$(( LIM1 * 100 ))" ]; } \
  && echo 'PASS: CPU gap doi o spec, o status va o cpu.max tren node'
{ [ "$(jp '{.metadata.uid}')" = "$(v uid)" ] && [ "$(jp "$S.containerID}")" = "$(v cid)" ] && \
  [ "$(jp "$S.restartCount}")" = "$(v rc)" ] && [ "$(jp '{.status.qosClass}')" = 'Burstable' ]; } \
  && echo 'PASS: cung Pod, cung container, khong khoi dong lai, van Burstable'
N="$(kubectl -n "$NS" exec web -c status-writer -- wget -qO- "http://$(jp '{.status.podIP}'):3000/books" | grep -o '"title"' | wc -l)"
[ "$N" -ge 5 ] && echo "PASS: /books van tra $N cuon sach"

# Diễn tập: đổi bất khả thi rồi trả về
kubectl -n "$NS" patch pod api --subresource resize \
  -p '{"spec":{"containers":[{"name":"postgrest","resources":{"requests":{"cpu":"1000"},"limits":{"cpu":"1000"}}}]}}'
R=''; for i in $(seq 1 24); do R="$(jp '{.status.conditions[?(@.type=="PodResizePending")].reason}')"; [ -n "$R" ] && break; sleep 5; done
jp '{.status.conditions[?(@.type=="PodResizePending")].message}' > "$EV/b5-infeasible.txt"; cat "$EV/b5-infeasible.txt"; echo
{ [ "$R" = 'Infeasible' ] && [ "$(mc "$(jp "$S.resources.limits.cpu}")")" -eq "$LIM1" ] && \
  [ "$(jp "$S.restartCount}")" = "$(v rc)" ]; } && echo 'PASS: Infeasible — spec doi, status giu gia tri cu, khong khoi dong lai'
kubectl -n "$NS" patch pod api --subresource resize \
  -p "{\"spec\":{\"containers\":[{\"name\":\"postgrest\",\"resources\":{\"requests\":{\"cpu\":\"${REQ1}m\"},\"limits\":{\"cpu\":\"${LIM1}m\"}}}]}}"
for i in $(seq 1 24); do [ -z "$(jp '{.status.conditions[?(@.type=="PodResizePending")].type}')" ] && break; sleep 5; done
[ -z "$(jp '{.status.conditions[?(@.type=="PodResizePending")].type}')" ] && echo 'PASS: tra ve gia tri kha thi, het PodResizePending'

# Năm ranh giới, dry-run phía server: không thay đổi gì trên cluster
: > "$EV/b5-ranh-gioi.txt"
thu() { local o; o="$(kubectl -n "$NS" patch pod "$1" $2 --dry-run=server -p "$3" 2>&1)"; printf '== %s\n%s\n' "$1" "$o" >> "$EV/b5-ranh-gioi.txt"; echo "$o"; }
thu db '--subresource=resize' '{"spec":{"containers":[{"name":"postgres","resources":{"requests":{"cpu":"10m"}}}]}}' \
  | grep -q 'QOS Class may not change' && echo 'PASS: resize khong duoc doi QoS cua db'
thu api '--subresource=resize' '{"spec":{"initContainers":[{"name":"wait-for-db","resources":{"limits":{"cpu":"200m"}}}]}}' \
  | grep -q 'non-sidecar init containers are immutable' && echo 'PASS: init container thuong khong doi duoc tai nguyen'
thu api '--subresource=resize' '{"spec":{"containers":[{"name":"postgrest","resources":{"limits":{"cpu":null}}}]}}' \
  | grep -q 'cannot be removed' && echo 'PASS: limit da dat thi khong go duoc'
thu web '--subresource=resize' '{"spec":{"initContainers":[{"name":"status-writer","resources":{"limits":{"cpu":"60m"}}}]}}' \
  | grep -q 'patched' && echo 'PASS: sidecar doi duoc tai nguyen tai cho'
thu api '' '{"spec":{"containers":[{"name":"postgrest","resources":{"limits":{"cpu":"3"}}}]}}' \
  | grep -q 'pod updates may not change fields' && echo 'PASS: duong cap nhat Pod thuong khong doi duoc tai nguyen'
```

Phần 3 — chạy **sau** khi đã đồng bộ repo:

```bash
cd ~/bookstore
RR="$(sed 's/__DB_HOST__/10.0.0.1/' k8s/app/api/pod.yaml.tpl \
      | kubectl create --dry-run=client -f - -o jsonpath="$C.resources.requests.cpu} $C.resources.limits.cpu}")"
set -- $RR
{ [ "$(mc "$1")" -eq "$(mc "$(jp "$C.resources.requests.cpu}")")" ] && \
  [ "$(mc "$2")" -eq "$(mc "$(jp "$C.resources.limits.cpu}")")" ]; } && echo 'PASS: repo mang dung con so CPU dang chay'
./scripts/deploy.sh > "$EV/b5-deploy.txt" 2>&1 && echo 'PASS: deploy.sh chay lai khong loi'
{ [ "$(jp '{.metadata.uid}')" = "$(v uid)" ] && [ "$(jp "$S.restartCount}")" = "$(v rc)" ]; } \
  && echo 'PASS: deploy.sh khong tao lai api, khong khoi dong lai postgrest'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Thêm vào container `postgrest` trong `k8s/app/api/pod.yaml.tpl`, rồi
`./scripts/teardown.sh && ./scripts/deploy.sh` và chạy phần 1:

```yaml
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired
    - resourceName: memory
      restartPolicy: RestartContainer
```

Gấp đôi CPU tại chỗ (với số của lời giải B2):

```bash
kubectl -n bookstore patch pod api --subresource resize \
  -p '{"spec":{"containers":[{"name":"postgrest","resources":{"requests":{"cpu":"100m"},"limits":{"cpu":"1"}}}]}}'
kubectl -n bookstore get pod api -o jsonpath='{.status.containerStatuses[?(@.name=="postgrest")].resources}{"\n"}'
```

Chạy phần 2. Rồi đổi khối tài nguyên của `postgrest` trong repo thành `requests: {cpu: 100m, memory: 64Mi}`,
`limits: {cpu: "1", memory: 256Mi}` và chạy phần 3. `deploy.sh` thành công vì giá trị trong patch mà
`kubectl apply` gửi đi **trùng** giá trị đang chạy — API server chỉ từ chối khi tài nguyên thật sự đổi.

Ý chính cho ADR 0021:

1. Kernel đổi `cpu.max` của cgroup đang chạy được ngay, tiến trình không cần biết. Bài 289: nhiều
   runtime không điều chỉnh được lượng memory đã cấp phát một cách động, và giảm memory với
   `NotRequired` chỉ là best-effort — chọn `RestartContainer` cho memory để hành vi đoán trước được.
   Đổi cả hai trong một lệnh thì container khởi động lại — chính sách của memory thắng.
2. Chưa. `spec` là thứ bạn **muốn**; `status.containerStatuses[].resources` là thứ kubelet **đã cấu
   hình**; `cpu.max` trên node là thứ kernel **đang áp**. Ba chỗ phải trùng nhau.
3. `Infeasible` là bất khả thi trên node này — không tự khỏi, phải đổi lại. `Deferred` là chưa được
   lúc này — kubelet thử lại. Bài 289: khi có resize đang chờ, scheduler dùng giá trị **lớn nhất** giữa
   request mong muốn, đã cấp phát và thực tế — một yêu cầu 1000 CPU đang treo làm node trông như đầy.
4. Không. QoS cố định lúc tạo và resize không được đổi QoS; và `wait-for-db` là init container thường,
   tài nguyên của nó không đổi được tại chỗ, nên Pod `api` không bao giờ đạt điều kiện request bằng
   limit ở **mọi** container.
5. Repo là nguồn sự thật duy nhất ([mục 5.3](DU-AN-BOOKSTORE.md#53-quy-tắc-làm-việc-với-repo)). Quên
   đồng bộ thì `deploy.sh` kế tiếp **vẫn chạy** — `kubectl apply` không thấy thay đổi nào giữa bản đã
   apply và repo nên không gửi tài nguyên — và lặng lẽ để lệch; lần `teardown`/`deploy` sau Pod mới quay
   về con số cũ, mất quyết định của đợt khuyến mãi.

</details>

## B6. [BS-3C-06] PodDisruptionBudget cho `web` và `api`

**Loại:** Tính năng.

**Bối cảnh:** yêu cầu [`R3`](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới) đòi ứng dụng chịu được
gián đoạn **tự nguyện**, và [mục 3.7](DU-AN-BOOKSTORE.md#37-chịu-lỗi-và-vận-hành) hẹn Lab 12 bảo trì
worker mà `web`, `api` không gián đoạn. Với Pod trần, một Pod bị đuổi khỏi node là mất hẳn — không
controller nào tạo lại. Hợp đồng ([mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label)) đặt hai
PodDisruptionBudget `web` và `api` từ sprint này.

**Yêu cầu:**

1. PodDisruptionBudget `web` và `api` trong namespace `bookstore`: label theo hợp đồng của thành phần
   tương ứng; selector là cặp `app.kubernetes.io/part-of=bookstore` +
   `app.kubernetes.io/component=<thành phần>`; dùng **dạng ngân sách duy nhất hợp lệ cho Pod trần**,
   giữ ít nhất một Pod khả dụng.
2. Manifest ở `k8s/app/web/pdb.yaml`, `k8s/app/api/pdb.yaml`, nằm trong `k8s/app/kustomization.yaml`.
   `scripts/deploy.sh` không đổi.
3. `status` của mỗi PDB cho thấy: một Pod dự kiến, một Pod khỏe, cần một, ngân sách gián đoạn bằng 0.
4. Một yêu cầu trục xuất `web` qua Eviction API bị từ chối; `web` nguyên vẹn.
5. `kubectl delete pod web` thành công dù có PDB; không ai tạo lại `web`; `deploy.sh` dựng lại.
6. Trên nhóm thử nghiệm ở `~/lab-work/3c/thu-pdb.yaml` — ba Pod `thu-pdb-1`, `thu-pdb-2`,
   `thu-pdb-3` ghim worker2 và PDB `thu-pdb` giữ ít nhất hai Pod, tất cả mang label
   `bookstore.example.com/thu: pdb` — yêu cầu trục xuất thứ nhất được phép và Pod kết thúc êm; yêu cầu
   thứ hai bị từ chối. ⭐ Thêm PDB thứ hai `thu-pdb-chong` (file `thu-pdb-chong.yaml`) cùng phủ ba
   Pod: yêu cầu trục xuất Pod còn lại bị từ chối **vì một lý do khác**.
7. `docs/adr/0022-pdb.md`.

**Ràng buộc:** không đặt PDB cho `db` — hợp đồng không có. Không `drain`, `cordon` (Lab 12). Object
thử nghiệm không mang label dự án, selector không chồng lên `web`, `api`; xóa khi xong.

**Đọc lại:**

- Bài [53](../53-disruptions-vi.md):
  [*Gián đoạn tự nguyện và không tự nguyện*](../53-disruptions-vi.md#voluntary-and-involuntary-disruptions)
  — kể cả khối *Thận trọng*; [*Ngân sách gián đoạn Pod*](../53-disruptions-vi.md#pod-disruption-budgets).
- Bài [339](../339-configure-pdb-vi.md): mục *Suy nghĩ về cách ứng dụng của bạn phản ứng với gián đoạn*
  — ứng dụng stateful một instance; mục *Chỉ định một PodDisruptionBudget*; mục *Kiểm tra trạng thái
  của PDB*; [*Tính khỏe mạnh của một Pod*](../339-configure-pdb-vi.md#healthiness-of-a-pod);
  [*Workload tùy ý và selector tùy ý*](../339-configure-pdb-vi.md#arbitrary-controllers-and-selectors).

**Bàn giao:** `k8s/app/web/pdb.yaml`, `k8s/app/api/pdb.yaml`, `k8s/app/kustomization.yaml`;
`docs/adr/0022-pdb.md`; `~/lab-evidence/3c/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` hai PDB trong repo và kustomization; `deploy.sh` không đổi vẫn áp được chúng.
- `PASS:` mỗi PDB mang label và selector theo hợp đồng, dùng `minAvailable` số nguyên 1.
- `PASS:` `status` mỗi PDB: dự kiến 1, khỏe 1, cần 1, cho phép 0.
- `PASS:` Eviction API từ chối trục xuất `web`; `web` giữ nguyên UID.
- `PASS:` `kubectl delete` xóa được `web`; không ai tạo lại; PDB ghi nhận dưới mức tối thiểu;
  `deploy.sh` dựng lại.
- `PASS:` nhóm thử nghiệm: ngân sách 1; trục xuất thứ nhất được phép, thứ hai bị từ chối.
- `PASS:` (⭐) Pod bị hai PDB phủ thì Eviction API từ chối với lý do nhiều PDB.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0022-pdb.md`:

1. Vì sao với Pod trần chỉ dùng được `minAvailable` dạng số nguyên? Viết `maxUnavailable: 0` hay
   `minAvailable: "100%"` thì vướng gì?
2. Ngân sách của `web` bằng 0. Điều đó nghĩa là gì cho buổi bảo trì `lab-k8s-worker1` ở Lab 12? Lab 4a
   có ba bản sao `web` — PDB nên đổi thế nào?
3. Vì sao hợp đồng **không** đặt PDB cho `db`? Đối chiếu hai giải pháp cho ứng dụng stateful một
   instance của bài 339, và `TD-04`.
4. PDB không chặn được những gián đoạn nào? Mất điện `lab-k8s-worker2` có bị tính vào ngân sách không,
   và hệ quả cho một lần trục xuất tự nguyện ngay sau đó?
5. (⭐) Vì sao Eviction API không chấp nhận Pod bị hai PDB cùng phủ, và lúc nào chồng lấn là hợp lý?

<details>
<summary>Gợi ý</summary>

1. PDB thuộc API `policy/v1`, ba trường: `selector`, và đúng một trong `minAvailable`, `maxUnavailable`.
2. Gọi Eviction API bằng `kubectl` không cần công cụ gì thêm: ghi một object `Eviction` (`policy/v1`,
   `metadata.name` là tên Pod) ra file JSON, rồi
   `kubectl create --raw /api/v1/namespaces/bookstore/pods/<tên>/eviction -f <file>`.
3. Đọc `status` của PDB: `kubectl -n bookstore get pdb web -o yaml`; cột `ALLOWED DISRUPTIONS` trong
   `kubectl get pdb`.
4. Disruption controller cần vài giây để đếm Pod — chờ ngân sách khác 0 rồi mới thử trục xuất trên
   nhóm thử nghiệm.

</details>

<details>
<summary>Script nghiệm thu — script này làm luôn các yêu cầu 4–6; chạy sau khi đã viết hai PDB và `thu-pdb.yaml`</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3c; W=~/lab-work/3c
cd ~/bookstore
{ test -f k8s/app/web/pdb.yaml && test -f k8s/app/api/pdb.yaml && grep -q 'web/pdb.yaml' k8s/app/kustomization.yaml && \
  grep -q 'api/pdb.yaml' k8s/app/kustomization.yaml; } && echo 'PASS: hai PDB nam trong repo va kustomization'
./scripts/deploy.sh > "$EV/b6-deploy.txt" 2>&1 && echo 'PASS: deploy.sh khong doi van ap duoc PDB'
pdb() { kubectl -n "$NS" get pdb "$1" -o jsonpath="$2" 2>/dev/null; }
ev() {   # ev <pod>: gửi một yêu cầu trục xuất qua Eviction API, in câu trả lời
  printf '{"apiVersion":"policy/v1","kind":"Eviction","metadata":{"name":"%s","namespace":"%s"}}\n' "$1" "$NS" > "$W/evict.json"
  kubectl create --raw "/api/v1/namespaces/$NS/pods/$1/eviction" -f "$W/evict.json" 2>&1
}

for P in web api; do
  for i in $(seq 1 24); do [ "$(pdb "$P" '{.status.expectedPods}/{.status.currentHealthy}')" = '1/1' ] && break; sleep 5; done
  [ "$(pdb "$P" '{.metadata.labels.app\.kubernetes\.io/part-of}/{.metadata.labels.app\.kubernetes\.io/component}/{.spec.selector.matchLabels.app\.kubernetes\.io/part-of}/{.spec.selector.matchLabels.app\.kubernetes\.io/component}')" \
    = "bookstore/$P/bookstore/$P" ] && echo "PASS: PDB $P mang label va selector theo hop dong"
  [ "$(pdb "$P" '{.spec.minAvailable}|{.spec.maxUnavailable}')" = '1|' ] && echo "PASS: PDB $P dung minAvailable so nguyen 1"
  [ "$(pdb "$P" '{.status.expectedPods} {.status.currentHealthy} {.status.desiredHealthy} {.status.disruptionsAllowed}')" = '1 1 1 0' ] \
    && echo "PASS: PDB $P: du kien 1, khoe 1, can 1, cho phep 0"
done
kubectl -n "$NS" get pdb | tee "$EV/b6-pdb.txt"

# Eviction API trên web thật
U0="$(kubectl -n "$NS" get pod web -o jsonpath='{.metadata.uid}')"
ev web > "$EV/b6-evict-web.txt"; cat "$EV/b6-evict-web.txt"
grep -q 'disruption budget' "$EV/b6-evict-web.txt" && echo 'PASS: Eviction API bi PDB web tu choi'
sleep 5
[ "$(kubectl -n "$NS" get pod web -o jsonpath='{.metadata.uid}' 2>/dev/null)" = "$U0" ] && echo 'PASS: web nguyen ven, cung UID'

# kubectl delete bỏ qua PDB
kubectl -n "$NS" delete pod web --wait=true --timeout=120s >/dev/null && echo 'PASS: kubectl delete xoa duoc web du co PDB'
sleep 20
kubectl -n "$NS" get pod web >/dev/null 2>&1 && echo 'FAIL: co thu gi tao lai web' || echo 'PASS: khong ai tao lai web — Pod tran'
for i in $(seq 1 12); do [ "$(pdb web '{.status.currentHealthy}')" = '0' ] && break; sleep 5; done
[ "$(pdb web '{.status.currentHealthy} {.status.desiredHealthy}')" = '0 1' ] && echo 'PASS: PDB ghi nhan web duoi muc toi thieu'
./scripts/deploy.sh >> "$EV/b6-deploy.txt" 2>&1 && echo 'PASS: deploy.sh dung lai web'

# Ngân sách tiêu dần trên nhóm thử nghiệm
kubectl apply -f "$W/thu-pdb.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/thu-pdb-1 pod/thu-pdb-2 pod/thu-pdb-3 --timeout=180s
for i in $(seq 1 24); do [ "$(pdb thu-pdb '{.status.disruptionsAllowed}')" = '1' ] && break; sleep 5; done
[ "$(pdb thu-pdb '{.status.expectedPods} {.status.currentHealthy} {.status.desiredHealthy} {.status.disruptionsAllowed}')" = '3 3 2 1' ] \
  && echo 'PASS: thu-pdb: 3 Pod, can 2, ngan sach 1'
ev thu-pdb-1 > "$EV/b6-thu-evict-1.txt"
kubectl -n "$NS" wait --for=delete pod/thu-pdb-1 --timeout=120s >/dev/null && echo 'PASS: truc xuat thu nhat duoc phep, thu-pdb-1 ket thuc'
for i in $(seq 1 24); do [ "$(pdb thu-pdb '{.status.disruptionsAllowed}')" = '0' ] && break; sleep 5; done
ev thu-pdb-2 > "$EV/b6-thu-evict-2.txt"
{ grep -q 'disruption budget' "$EV/b6-thu-evict-2.txt" && kubectl -n "$NS" get pod thu-pdb-2 >/dev/null 2>&1; } \
  && echo 'PASS: truc xuat thu hai bi tu choi, thu-pdb-2 con nguyen'

# ⭐ PDB chồng lấn — bỏ qua nếu không có file
if [ -f "$W/thu-pdb-chong.yaml" ]; then
  kubectl apply -f "$W/thu-pdb-chong.yaml"; sleep 10
  ev thu-pdb-3 > "$EV/b6-thu-evict-chong.txt"; cat "$EV/b6-thu-evict-chong.txt"
  grep -q 'more than one PodDisruptionBudget' "$EV/b6-thu-evict-chong.txt" && echo 'PASS: Pod bi hai PDB phu thi Eviction API tu choi'
fi
kubectl -n "$NS" delete pod,pdb -l 'bookstore.example.com/thu=pdb' --ignore-not-found --wait=true
[ "$(kubectl -n "$NS" get pdb -o name | sort | tr '\n' ' ')" = 'poddisruptionbudget.policy/api poddisruptionbudget.policy/web ' ] \
  && echo 'PASS: chi con hai PDB cua du an'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/web/pdb.yaml — api/pdb.yaml giống hệt, đổi web thành api và nginx thành postgrest
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
    app.kubernetes.io/name: nginx
spec:
  minAvailable: 1        # Pod trần: chỉ minAvailable số nguyên (bài 339). Lab 4a xem lại khi có bản sao.
  selector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: web
```

Thêm hai dòng `- web/pdb.yaml` và `- api/pdb.yaml` vào `resources` của `k8s/app/kustomization.yaml`.

```yaml
# ~/lab-work/3c/thu-pdb.yaml — thu-pdb-2, thu-pdb-3 chép y hệt thu-pdb-1, chỉ đổi tên
apiVersion: v1
kind: Pod
metadata:
  name: thu-pdb-1
  namespace: bookstore
  labels:
    bookstore.example.com/thu: pdb
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  terminationGracePeriodSeconds: 5
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 10m, memory: 16Mi}
---
# ... thu-pdb-2, thu-pdb-3
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: thu-pdb
  namespace: bookstore
  labels:
    bookstore.example.com/thu: pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      bookstore.example.com/thu: pdb
```

`thu-pdb-chong.yaml` là PDB thứ hai, tên `thu-pdb-chong`, cùng label và selector, `minAvailable: 1`.

Ý chính cho ADR 0022:

1. Bài 339: với Pod trần, Kubernetes không suy ra được tổng số Pod vì không có resource sở hữu nào —
   `maxUnavailable` và mọi giá trị phần trăm đều cần "số replica mong muốn". Còn lại duy nhất
   `minAvailable` số nguyên.
2. Bài 339: `minAvailable` bằng số replica nghĩa là không cho phép eviction tự nguyện nào — drain
   worker đang chạy `web` sẽ treo. Với Pod trần, đó lại là điều đúng: trục xuất là mất hẳn. Ở Lab 4a,
   ba bản sao `web` cho phép `minAvailable: 2` hoặc `maxUnavailable: 1` — ghi thành việc cần làm của
   sprint 4.
3. Bài 339 cho ứng dụng stateful một instance hai lựa chọn: không PDB và chấp nhận downtime, hoặc PDB
   cấm mọi gián đoạn cộng thỏa thuận ngoài Kubernetes. Dự án chọn cách một và đưa việc bảo trì
   `lab-k8s-worker2` thành cửa sổ bảo trì có kế hoạch ở Lab 12 — vì dữ liệu trên `emptyDir` (`TD-04`)
   mất theo Pod dù ai trục xuất, PDB không cứu được dữ liệu.
4. Bài 53: PDB chỉ ràng buộc gián đoạn tự nguyện **đi qua Eviction API**; `kubectl delete` và xóa
   workload bỏ qua nó; gián đoạn không tự nguyện không bị chặn nhưng **vẫn bị tính** vào ngân sách —
   sau khi mất `lab-k8s-worker2`, mọi yêu cầu trục xuất tự nguyện sẽ bị từ chối cho tới khi đủ lại Pod
   khỏe.
5. Bài 339: Eviction API không cho phép trục xuất Pod bị nhiều PDB phủ; chồng lấn chỉ hợp lý trong lúc
   chuyển Pod từ PDB này sang PDB khác.

</details>

## B7. [BS-3C-07] Điều tra: static Pod và mirror Pod

**Loại:** Điều tra — **trả nợ lab của Lab 3a** (thực hành static Pod, bài 293). Đọc lại bài
[58](../58-static-pods-vi.md) và bài [293](../293-static-pod-tasks-vi.md) **trước** khi làm.

**Bối cảnh:** đội vận hành đề xuất chạy một agent ghi heartbeat của node bằng static Pod trên
`lab-k8s-worker2`, "vì nó sống được cả khi API server chết", thay vì chờ DaemonSet `node-agent` ở
sprint 5 (Lab 4b). Trước khi quyết, đội muốn thấy tận mắt static Pod hoạt động thế nào trên chính
cluster này — bắt đầu từ bốn static Pod đang chạy control plane.

**Yêu cầu:**

1. Kiểm kê **chỉ đọc** trên `lab-k8s-master`: số file manifest static Pod bằng số mirror Pod của
   control plane; tên mirror Pod bằng tên trong manifest cộng hostname của node.
2. Đọc thư mục static Pod mà kubelet của `lab-k8s-worker2` đang quét.
3. Viết `~/lab-work/3c/lab3c-static.yaml`: Pod `lab3c-static` trong namespace `default`, label
   `app: lab3c-static`, một container `agent` (`busybox`) ghi heartbeat định kỳ, thoát gọn khi nhận
   `TERM`, có khai tài nguyên. Đặt file đó vào thư mục static Pod của worker2 — **không** `kubectl apply`.
4. Chạy script nghiệm thu ngay khi Pod lên. Script chứng minh mirror Pod xuất hiện đúng tên, đúng node,
   mang annotation mirror, label được lan truyền, và không đổi được tài nguyên qua API; rồi làm luôn ba
   việc: xóa mirror Pod bằng `kubectl` (mirror quay lại, container thật không đổi); làm container thoát
   từ bên trong (kubelet tự khởi động lại); xóa file trên node (Pod biến mất, thư mục trả về trạng thái
   ghi ở B0).
5. `docs/adr/0023-static-pod.md`: static Pod hay DaemonSet cho agent của dự án.

**Ràng buộc:** không ghi gì vào `/etc/kubernetes/manifests` của master. Không dùng `crictl` — bài 293
để phần đó cho giai đoạn 24. Lệnh `sudo` qua `ssh` dùng `sudo -S`. Không để file nào khác ngoài
`lab3c-static.yaml` trong thư mục static Pod của worker2, kể cả bản sao lưu.

**Đọc lại:**

- Bài [58](../58-static-pods-vi.md): [*Mirror Pod*](../58-static-pods-vi.md#mirror-pods),
  [*Giới hạn*](../58-static-pods-vi.md#limitations),
  [*Static Pod so với DaemonSet*](../58-static-pods-vi.md#static-pods-vs-daemonsets).
- Bài [293](../293-static-pod-tasks-vi.md):
  [*Manifest static Pod đặt trên hệ thống file*](../293-static-pod-tasks-vi.md#configuration-files) —
  khối *Thận trọng*; [*Quan sát hành vi của static Pod*](../293-static-pod-tasks-vi.md#behavior-of-static-pods);
  mục *Thêm và xóa static Pod một cách động*.

**Bàn giao:** `docs/adr/0023-static-pod.md`; `~/lab-evidence/3c/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` bốn file manifest ứng với bốn mirror Pod trên master; tên mirror đúng quy tắc.
- `PASS:` kubelet của worker2 quét `/etc/kubernetes/manifests`.
- `PASS:` mirror Pod `lab3c-static-lab-k8s-worker2` đúng node, có annotation mirror, label lan truyền.
- `PASS:` API server từ chối đổi tài nguyên của static Pod.
- `PASS:` xóa mirror Pod bằng `kubectl` thì mirror quay lại, `containerID` và `restartCount` giữ nguyên.
- `PASS:` container thoát thì kubelet tự khởi động lại, không cần API server ra lệnh.
- `PASS:` xóa file trên node thì Pod biến mất; worker2 không còn file của lab.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0023-static-pod.md`:

1. Xóa mirror Pod bằng `kubectl` thì thứ gì bị xóa, thứ gì không? Vì sao annotation mirror và hậu tố
   hostname là dấu hiệu nhận biết quan trọng khi trực sự cố?
2. Vì sao kubeadm chạy `kube-apiserver` bằng static Pod chứ không phải DaemonSet?
3. Agent của dự án cần đọc cấu hình từ ConfigMap và cần được phát hành phiên bản mới trên cả hai
   worker. Static Pod đáp ứng được không? Kết luận: dự án chọn gì, và vì sao điều đó chờ được tới Lab 4b?
4. Một đồng nghiệp sao lưu manifest ngay trong thư mục static Pod của master bằng `cp ... .backup`.
   Chuyện gì có thể xảy ra?

<details>
<summary>Gợi ý</summary>

1. Thư mục static Pod là trường `staticPodPath` trong `/var/lib/kubelet/config.yaml` của node.
2. Đưa file lên worker2 mà không cần `sudo` hai lần qua stdin: `scp` file vào `/tmp` của worker2, rồi
   một lệnh `ssh lab-k8s-worker2 'sudo -S sh -c "mkdir -p ... && install -m 0644 ... && rm -f ..."'`.
3. Container thoát gọn khi nhận `TERM`: `trap 'exit 0' TERM` cộng `sleep 5 & wait $!` — shell bỏ qua
   `TERM` khi `sleep` chạy foreground.
4. Mirror Pod nằm ở namespace ghi trong manifest — `default` ở đây — không phải `kube-system`.

</details>

<details>
<summary>Script nghiệm thu — chạy khi static Pod của bạn vừa lên; script làm luôn yêu cầu 4</summary>

```bash
EV=~/lab-evidence/3c; M=lab3c-static-lab-k8s-worker2
mp() { kubectl -n default get pod "$M" -o jsonpath="$1" 2>/dev/null; }

# 1. Control plane trên master — chỉ đọc
sudo ls -1 /etc/kubernetes/manifests/ > "$EV/b7-cp-manifests.txt"; cat "$EV/b7-cp-manifests.txt"
kubectl -n kube-system get pods -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,MIRROR:.metadata.annotations.kubernetes\.io/config\.mirror' \
  > "$EV/b7-mirror-cp.txt"
FILES="$(grep -c '\.yaml$' "$EV/b7-cp-manifests.txt")"
MIRRORS="$(kubectl -n kube-system get pods -o jsonpath='{range .items[*]}{.metadata.annotations.kubernetes\.io/config\.mirror}{"\n"}{end}' | grep -c .)"
{ [ "$FILES" -eq 4 ] && [ "$MIRRORS" -eq 4 ]; } && echo 'PASS: bon file manifest ung voi bon mirror Pod tren master'
BAD=0
for C in etcd kube-apiserver kube-controller-manager kube-scheduler; do
  kubectl -n kube-system get pod "$C-lab-k8s-master" >/dev/null 2>&1 || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: ten mirror Pod = ten trong manifest + hostname node'

# 2. Thư mục static Pod của worker2
SPP="$(ssh lab-k8s-worker2 "sudo -S awk '/^staticPodPath:/{print \$2}' /var/lib/kubelet/config.yaml")"
echo "staticPodPath tren lab-k8s-worker2 = $SPP" | tee "$EV/b7-staticpodpath.txt"
[ "$SPP" = '/etc/kubernetes/manifests' ] && echo 'PASS: kubelet worker2 quet /etc/kubernetes/manifests'

# 3. Mirror Pod
for i in $(seq 1 36); do kubectl -n default get pod "$M" >/dev/null 2>&1 && break; sleep 5; done
kubectl -n default wait --for=condition=Ready "pod/$M" --timeout=120s
{ [ -n "$(mp '{.metadata.annotations.kubernetes\.io/config\.mirror}')" ] && [ "$(mp '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && \
  [ "$(kubectl get pods -A -l app=lab3c-static -o name | wc -l)" -eq 1 ]; } \
  && echo 'PASS: mirror Pod dung ten, dung node, co annotation mirror, label lan truyen'
kubectl -n default patch pod "$M" --subresource resize --dry-run=server \
  -p '{"spec":{"containers":[{"name":"agent","resources":{"limits":{"cpu":"60m"}}}]}}' > "$EV/b7-resize.txt" 2>&1
grep -q 'static pods cannot be resized' "$EV/b7-resize.txt" && echo 'PASS: API khong doi duoc tai nguyen cua static Pod'

# 4. Xóa mirror Pod
CID0="$(mp '{.status.containerStatuses[0].containerID}')"; RC0="$(mp '{.status.containerStatuses[0].restartCount}')"
kubectl -n default delete pod "$M" --wait=false
sleep 5; CID1=''
for i in $(seq 1 36); do
  if kubectl -n default get pod "$M" >/dev/null 2>&1 && [ -z "$(mp '{.metadata.deletionTimestamp}')" ]; then
    CID1="$(mp '{.status.containerStatuses[0].containerID}')"; [ -n "$CID1" ] && break
  fi
  sleep 5
done
RC1="$(mp '{.status.containerStatuses[0].restartCount}')"
echo "truoc: $CID0 rc=$RC0 ; sau: $CID1 rc=$RC1" | tee "$EV/b7-mirror-delete.txt"
{ [ -n "$CID1" ] && [ "$CID1" = "$CID0" ] && [ "$RC1" = "$RC0" ]; } \
  && echo 'PASS: mirror Pod quay lai, container that khong he dung'

# 5. Container thoát — kubelet tự khởi động lại
kubectl -n default exec "$M" -- kill 1
for i in $(seq 1 24); do [ "$(mp '{.status.containerStatuses[0].restartCount}')" -gt "$RC1" ] 2>/dev/null && break; sleep 5; done
[ "$(mp '{.status.containerStatuses[0].restartCount}')" -gt "$RC1" ] 2>/dev/null \
  && echo 'PASS: container thoat, kubelet tu khoi dong lai static Pod'

# 6. Xóa file trên node — nhập mật khẩu sudo khi được hỏi
ssh lab-k8s-worker2 'sudo -S rm -f /etc/kubernetes/manifests/lab3c-static.yaml'
grep -qx created "$EV/b0-worker2-manifests.txt" && ssh lab-k8s-worker2 'sudo -S rmdir /etc/kubernetes/manifests'
for i in $(seq 1 36); do kubectl -n default get pod "$M" >/dev/null 2>&1 || break; sleep 5; done
kubectl -n default get pod "$M" >/dev/null 2>&1 && echo 'FAIL: mirror Pod van con' || echo 'PASS: xoa file tren node moi xoa that static Pod'
ssh lab-k8s-worker2 'sudo -S ls -A /etc/kubernetes/manifests 2>/dev/null' | grep -q 'lab3c' \
  && echo 'FAIL: worker2 con file static Pod cua lab' || echo 'PASS: worker2 khong con file static Pod cua lab'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/3c/lab3c-static.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lab3c-static
  namespace: default
  labels:
    app: lab3c-static
spec:
  containers:
  - name: agent
    image: busybox:1.37
    command: ["sh", "-c", "trap 'exit 0' TERM; while :; do date > /tmp/heartbeat; sleep 5 & wait $!; done"]
    resources:
      requests: {cpu: 10m, memory: 8Mi}
      limits: {cpu: 50m, memory: 16Mi}
```

```bash
scp -q ~/lab-work/3c/lab3c-static.yaml lab-k8s-worker2:/tmp/lab3c-static.yaml
ssh lab-k8s-worker2 'sudo -S sh -c "mkdir -p /etc/kubernetes/manifests && install -m 0644 -o root -g root /tmp/lab3c-static.yaml /etc/kubernetes/manifests/lab3c-static.yaml && rm -f /tmp/lab3c-static.yaml"'
kubectl -n default get pod -w     # Ctrl+C khi thấy lab3c-static-lab-k8s-worker2 Running
```

Ý chính cho ADR 0023:

1. Bài 58: xóa bằng `kubectl` chỉ xóa **bản phản chiếu** trên API server; kubelet không xóa static Pod
   và tạo lại mirror. Tên có hậu tố hostname và annotation `kubernetes.io/config.mirror` báo cho người
   trực biết: sửa Pod này phải sửa file trên node, không phải qua API.
2. DaemonSet cần một control plane đang chạy; static Pod được kubelet khởi động trước khi API server
   sẵn sàng — thứ duy nhất bootstrap được chính API server.
3. Không: spec static Pod không tham chiếu được ConfigMap, Secret, ServiceAccount; không rollout,
   rollback, co giãn bằng cơ chế chuẩn — phải sửa file trên từng node; và như B7 vừa thấy, không đổi
   được tài nguyên qua API. Dự án chọn DaemonSet `node-agent` ở Lab 4b; heartbeat của node chưa phải
   yêu cầu của sprint này nên chờ được.
4. Bài 293, khối *Thận trọng*: kubelet đọc mọi file không bắt đầu bằng dấu chấm, cả hai file định nghĩa
   cùng một Pod — hành vi không xác định, spec cũ trong bản backup có thể âm thầm có hiệu lực. Sao lưu
   ra ngoài thư mục, ví dụ `/etc/kubernetes/backup/`.

</details>

## B8. [BS-3C-08] ⭐ Điều tra: tài nguyên cấp Pod

**Loại:** Điều tra — tùy chọn.

**Bối cảnh:** Pod `web` có ba container mà đội phải định cỡ từng cái. Một ý tưởng: khai **một** ngân
sách cho cả Pod và để các container chia nhau. Baseline của cluster đã bật tài nguyên cấp Pod (beta);
trước khi cân nhắc cho `web`, kiểm nó hoạt động thế nào và thiếu gì.

**Yêu cầu:** Pod thử nghiệm `thu-cap-pod` ở `~/lab-work/3c/thu-cap-pod.yaml`, ghim worker2: ngân sách
cấp Pod 200m CPU và 64Mi memory, request bằng limit; container `a` (`busybox`) tự khai CPU request 50m,
limit 100m; container `b` (`busybox`) không khai gì. Chứng minh: API server giữ ngân sách cấp Pod và
xếp Pod vào `Guaranteed` dù container `a` có request khác limit; cgroup cấp Pod trên node mang đúng
ngân sách đó; mọi lần đổi tài nguyên tại chỗ của Pod này — cấp Pod lẫn cấp container — bị từ chối trên
baseline. Ghi `docs/adr/0024-tai-nguyen-cap-pod.md`: có dùng cho `web` không. Xóa Pod khi xong.

**Đọc lại:** bài [265](../265-assign-pod-level-resources-vi.md) — mục mở đầu (ưu tiên cấp Pod, QoS),
mục *Giới hạn*, mục *Tạo một Pod với request và limit ở cả cấp Pod lẫn cấp container*; bài
[54](../54-pod-qos-vi.md), đoạn tiêu chí `Guaranteed` khi dùng tài nguyên cấp Pod; bài
[290](../290-resize-pod-resources-vi.md), mục *Trước khi bạn bắt đầu*.

**Nghiệm thu:**

- `PASS:` dry-run phía server giữ `spec.resources` và cho QoS `Guaranteed`.
- `PASS:` cgroup cấp Pod trên node: `memory.max` là 67108864, quota trong `cpu.max` là 20000.
- `PASS:` resize cấp Pod bị từ chối vì gate `InPlacePodLevelResourcesVerticalScaling` chưa bật.
- `PASS:` resize container `a` của chính Pod này cũng bị từ chối với cùng lý do.

<details>
<summary>Script nghiệm thu và lời giải tham khảo</summary>

```yaml
# ~/lab-work/3c/thu-cap-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: thu-cap-pod
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  terminationGracePeriodSeconds: 1
  resources:                      # cấp Pod: ngang hàng với containers
    requests: {cpu: 200m, memory: 64Mi}
    limits: {cpu: 200m, memory: 64Mi}
  containers:
  - name: a
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 50m}
      limits: {cpu: 100m}
  - name: b
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
```

```bash
NS=bookstore; EV=~/lab-evidence/3c
cgp() {
  local n u
  n="$(kubectl -n "$NS" get pod "$1" -o jsonpath='{.spec.nodeName}')"
  u="$(kubectl -n "$NS" get pod "$1" -o jsonpath='{.metadata.uid}' | tr - _)"
  ssh -n "$n" "find /sys/fs/cgroup/kubepods.slice -maxdepth 2 -type d -name 'kubepods*-pod$u.slice' -exec cat {}/$2 \;"
}
R="$(kubectl create --dry-run=server -f ~/lab-work/3c/thu-cap-pod.yaml -o jsonpath='{.spec.resources.limits.cpu}/{.spec.resources.limits.memory}/{.status.qosClass}')"
echo "$R" | tee "$EV/b8-cap-pod.txt"
[ "$R" = '200m/64Mi/Guaranteed' ] && echo 'PASS: API server giu spec.resources, QoS tinh theo cap Pod'
kubectl apply -f ~/lab-work/3c/thu-cap-pod.yaml
kubectl -n "$NS" wait --for=condition=Ready pod/thu-cap-pod --timeout=180s
M="$(cgp thu-cap-pod memory.max)"; C="$(cgp thu-cap-pod cpu.max)"
echo "cgroup cap Pod: memory.max=$M cpu.max=$C" | tee -a "$EV/b8-cap-pod.txt"
{ [ "$M" = '67108864' ] && [ "$(echo "$C" | awk '{print $1}')" = '20000' ]; } && echo 'PASS: cgroup cap Pod mang dung ngan sach cap Pod'
O1="$(kubectl -n "$NS" patch pod thu-cap-pod --subresource resize --dry-run=server \
      -p '{"spec":{"resources":{"requests":{"cpu":"300m"},"limits":{"cpu":"300m"}}}}' 2>&1)"
O2="$(kubectl -n "$NS" patch pod thu-cap-pod --subresource resize --dry-run=server \
      -p '{"spec":{"containers":[{"name":"a","resources":{"requests":{"cpu":"60m"},"limits":{"cpu":"120m"}}}]}}' 2>&1)"
printf '%s\n%s\n' "$O1" "$O2" >> "$EV/b8-cap-pod.txt"
echo "$O1" | grep -q 'pod-level resources is forbidden' && echo 'PASS: resize cap Pod bi tu choi tren baseline'
echo "$O2" | grep -q 'pod-level resources is forbidden' && echo 'PASS: resize ca container cua Pod co tai nguyen cap Pod cung bi tu choi'
kubectl -n "$NS" delete pod thu-cap-pod --wait=true
```

Ý chính cho ADR 0024: bài 265 — cấp Pod được ưu tiên cho request, limit và cả QoS; hai container
cộng lại không vượt ngân sách cấp Pod. Cái giá trên baseline: Pod có tài nguyên cấp Pod **không resize
tại chỗ được** — kể cả container của nó — vì gate của bài 290 còn alpha. `web` vừa cần định cỡ từng
container (sidecar `status-writer` có trần riêng để một lỗi của nó không ăn bộ nhớ của `nginx` — B4
`thu-oom`) nên dự án **chưa** dùng; xem lại khi gate resize cấp Pod lên beta.

</details>

## B9. [BS-3C-09] ⭐ Điều tra: extended resource

**Loại:** Điều tra — tùy chọn.

**Bối cảnh:** yêu cầu [`R18`](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới) hẹn một tính năng gợi
ý sách cần thiết bị chuyên dụng (Lab 13). Trước khi nói tới thiết bị thật, đội muốn thấy cluster đếm
và cấp phát một tài nguyên **do mình đặt tên** ra sao.

**Yêu cầu:** quảng bá 4 `example.com/dongle` trên `lab-k8s-worker2` bằng cách vá dung lượng của Node
(lệnh `PATCH` ở bài 110). Ba Pod thử nghiệm, mỗi Pod một file cùng tên ở `~/lab-work/3c/`, **không**
ghim node: `thu-dongle-a` xin 3 dongle; `thu-dongle-b` xin 2 dongle; `thu-dongle-sai` xin request 1,
limit 2. Chứng minh: chỉ
worker2 quảng bá; `thu-dongle-a` rơi đúng vào worker2 và vẫn là `BestEffort`; `thu-dongle-b` `Pending`
vì hết dongle; `thu-dongle-sai` bị API server từ chối. Xóa Pod và **gỡ** dongle khỏi worker2 khi xong.
Ghi `docs/adr/0025-extended-resource.md`.

**Đọc lại:** bài [110](../110-manage-resources-containers-vi.md):
[*Các tài nguyên khác*](../110-manage-resources-containers-vi.md#other-resources) — quảng bá bằng
`PATCH`, `~1` trong đường dẫn, `allocatable` cập nhật bất đồng bộ;
[*Tiêu thụ tài nguyên mở rộng*](../110-manage-resources-containers-vi.md#consuming-extended-resources);
bài [284](../284-extended-resource-vi.md) — cả bài; bài [54](../54-pod-qos-vi.md), câu cuối mục
`BestEffort`.

**Nghiệm thu:**

- `PASS:` worker2 có `capacity` và `allocatable` 4 dongle; worker1 không có.
- `PASS:` `thu-dongle-a` chạy trên worker2, QoS `BestEffort`.
- `PASS:` `thu-dongle-b` `Pending` với `Insufficient example.com/dongle`.
- `PASS:` `thu-dongle-sai` bị từ chối — request phải bằng limit.
- `PASS:` đã xóa Pod; worker2 không còn quảng bá dongle.

<details>
<summary>Script nghiệm thu và lời giải tham khảo</summary>

```bash
# Quảng bá — đây là phần bạn tự làm trước khi chạy script
kubectl patch node lab-k8s-worker2 --subresource=status --type=json \
  -p='[{"op": "add", "path": "/status/capacity/example.com~1dongle", "value": "4"}]'
```

```yaml
# ~/lab-work/3c/thu-dongle-a.yaml — thu-dongle-b xin "2"; thu-dongle-sai có requests "1", limits "2"
apiVersion: v1
kind: Pod
metadata:
  name: thu-dongle-a
  namespace: bookstore
spec:
  terminationGracePeriodSeconds: 1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        example.com/dongle: "3"
      limits:
        example.com/dongle: "3"
```

```bash
NS=bookstore; EV=~/lab-evidence/3c; W=~/lab-work/3c
dg() { kubectl get node "$1" -o jsonpath="{.status.$2.example\.com/dongle}" 2>/dev/null; }
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2" 2>/dev/null; }
for i in $(seq 1 24); do [ "$(dg lab-k8s-worker2 allocatable)" = '4' ] && break; sleep 5; done
{ [ "$(dg lab-k8s-worker2 capacity)" = '4' ] && [ "$(dg lab-k8s-worker2 allocatable)" = '4' ] && \
  [ -z "$(dg lab-k8s-worker1 capacity)" ]; } && echo 'PASS: chi worker2 quang ba 4 dongle'

kubectl apply -f "$W/thu-dongle-a.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/thu-dongle-a --timeout=180s
{ [ "$(jp thu-dongle-a '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && [ "$(jp thu-dongle-a '{.status.qosClass}')" = 'BestEffort' ]; } \
  && echo 'PASS: thu-dongle-a len worker2, van BestEffort'
kubectl apply -f "$W/thu-dongle-b.yaml"
for i in $(seq 1 24); do [ "$(jp thu-dongle-b '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ] && break; sleep 5; done
kubectl -n "$NS" describe pod thu-dongle-b > "$EV/b9-dongle-b.txt"
{ [ "$(jp thu-dongle-b '{.status.phase}')" = 'Pending' ] && grep -q 'Insufficient example.com/dongle' "$EV/b9-dongle-b.txt"; } \
  && echo 'PASS: het dongle thi Pod tiep theo Pending'
kubectl apply --dry-run=server -f "$W/thu-dongle-sai.yaml" > "$EV/b9-dongle-sai.txt" 2>&1 \
  && echo 'FAIL: API server chap nhan request khac limit' || echo 'PASS: extended resource khong overcommit duoc'

kubectl -n "$NS" delete pod thu-dongle-a thu-dongle-b --wait=true
kubectl patch node lab-k8s-worker2 --subresource=status --type=json \
  -p='[{"op": "remove", "path": "/status/capacity/example.com~1dongle"}]'
for i in $(seq 1 24); do [ -z "$(dg lab-k8s-worker2 allocatable)" ] && break; sleep 5; done
{ [ -z "$(dg lab-k8s-worker2 capacity)" ] && [ -z "$(dg lab-k8s-worker2 allocatable)" ]; } \
  && echo 'PASS: worker2 khong con quang ba dongle'
```

Ý chính cho ADR 0025: Kubernetes không kiểm node có dongle thật hay không, chỉ ghi nhận con số người
vận hành vá vào và trừ dần khi cấp phát; tài nguyên mở rộng là số nguyên, không overcommit, và không
làm Pod thoát `BestEffort`. Người quảng bá trong thực tế là device plugin (giai đoạn 14); cách cấp
phát thiết bị linh hoạt hơn là DRA — Lab 13 đánh giá cho `R18`.

</details>

## B10. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 3 chỉ từ repo và một file
bí mật sinh tại chỗ, rồi trả cluster về đúng `01-cluster-ready`.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 3c — chép nguyên văn. Đây là kiểm thử khói **tích
   lũy**: mọi hợp đồng của sprint 1, 2 cộng hợp đồng mới của sprint 3.

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 3c: ba Pod trần db, api, web; cấu hình từ ConfigMap; mật khẩu từ
# Secret db-credentials; mọi container có requests/limits khớp cgroup trên node; QoS theo thiết kế;
# PodDisruptionBudget web, api chặn Eviction API. Script không in mật khẩu: mọi so sánh nằm trong biến.
set -u
cd "$(dirname "$0")/.."
NS=bookstore
FAIL=0
TMP="$(mktemp)"; trap 'rm -f "$TMP"' EXIT
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
cond() { kubectl -n "$NS" get pod "$1" -o jsonpath="{.status.conditions[?(@.type==\"$2\")].status}" 2>/dev/null; }
label() { kubectl -n "$NS" get "$1" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null; }
web() { kubectl -n "$NS" exec web -c status-writer -- wget -qO- "$1" 2>/dev/null; }
sec() { kubectl -n "$NS" get secret db-credentials -o go-template="{{index .data \"$1\"}}" 2>/dev/null | base64 -d 2>/dev/null; }
sql() { kubectl -n "$NS" exec db -c postgres -- psql -U postgres -d bookstore -tAc "$1" 2>/dev/null; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
by() { case "$1" in
  *Ki) echo $(( ${1%Ki} * 1024 )) ;; *Mi) echo $(( ${1%Mi} * 1048576 )) ;; *Gi) echo $(( ${1%Gi} * 1073741824 )) ;;
  *k) echo $(( ${1%k} * 1000 )) ;; *M) echo $(( ${1%M} * 1000000 )) ;; *G) echo $(( ${1%G} * 1000000000 )) ;;
  *) echo "${1:-0}" ;; esac; }
cg() {   # cg <pod> <container> <file>: đọc một file cgroup v2 của container trên node đang chạy nó
  local n id
  n="$(jp "pod/$1" '{.spec.nodeName}')"
  id="$(jp "pod/$1" "{.status.containerStatuses[?(@.name==\"$2\")].containerID}{.status.initContainerStatuses[?(@.name==\"$2\")].containerID}")"
  [ -n "$n" ] && [ -n "$id" ] && ssh -n "$n" "find /sys/fs/cgroup/kubepods.slice -maxdepth 3 -type d -name 'cri-containerd-${id#containerd://}.scope' -exec cat {}/$3 \;" 2>/dev/null
}

# Sprint 1: ba Pod, dữ liệu, API, trang web
for P in db api web; do
  [ "$(cond "$P" Ready)" = 'True' ] && pass "Pod $P Ready" || fail "Pod $P chua Ready"
done
N="$(sql 'SELECT count(*) FROM api.books')"
[ "${N:-0}" -ge 5 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach"
API_IP="$(jp pod/api '{.status.podIP}')"
T="$(web "http://${API_IP}:3000/books" | grep -o '"title"' | wc -l)"
[ "$T" -ge 5 ] && pass "api /books tra $T cuon sach" || fail "api /books tra $T cuon sach"
web http://127.0.0.1:8080/ | grep -q 'Bookstore' && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
web http://127.0.0.1:8080/healthz >/dev/null && pass 'web /healthz tra 200' || fail 'web /healthz khong tra 200'
web http://127.0.0.1:8080/status.json | grep -q '"pod":"web"' \
  && pass 'web phuc vu status.json' || fail 'web khong phuc vu status.json'

# Sprint 2: object cấu hình theo hợp đồng
for O in configmap/web-config configmap/api-config configmap/db-init secret/db-credentials \
         priorityclass/bookstore-critical; do
  [ "$(label "$O")" = 'bookstore' ] && pass "$O ton tai, mang label du an" || fail "$O thieu hoac thieu label"
done
[ "$(jp secret/db-credentials '{.type}')" = 'Opaque' ] && pass 'db-credentials loai Opaque' || fail 'db-credentials sai loai'

# Sprint 2: mật khẩu chỉ nằm trong Secret, và dùng được
PGPW="$(sec postgres-password)"; AUPW="$(sec authenticator-password)"
if [ -z "$PGPW" ] || [ -z "$AUPW" ]; then
  fail 'khong doc duoc hai key cua db-credentials'
else
  DB_IP="$(jp pod/db '{.status.podIP}')"
  login() { kubectl -n "$NS" exec -i db -c postgres -- env H="$DB_IP" U="$1" \
    sh -c 'PGPASSWORD="$(cat)" psql -h "$H" -U "$U" -d bookstore -tAc "SELECT current_user"' 2>/dev/null; }
  [ "$(printf '%s' "$PGPW" | login postgres)" = 'postgres' ] \
    && pass 'superuser dang nhap qua IP Pod bang mat khau trong Secret' || fail 'superuser khong dang nhap duoc'
  [ "$(printf '%s' "$AUPW" | login authenticator)" = 'authenticator' ] \
    && pass 'authenticator dang nhap bang mat khau trong Secret' || fail 'authenticator khong dang nhap duoc'
  kubectl -n "$NS" get pods -o yaml 2>/dev/null | grep -qF -e "$PGPW" -e "$AUPW" \
    && fail 'mat khau xuat hien trong spec cua Pod' || pass 'spec cua Pod khong chua mat khau'
  git grep --untracked -qF -e "$PGPW" -e "$AUPW" \
    && fail 'mat khau nam trong file se vao git' || pass 'repo khong chua mat khau'
fi
git grep --untracked -qE 'lab-3a-mat-khau-tam|lab-3a-authenticator' -- k8s scripts \
  && fail 'con mat khau cu cua Lab 3a trong k8s/ hoac scripts/' || pass 'khong con mat khau cu trong manifest'
{ [ "$(git ls-files secrets)" = 'secrets/.gitkeep' ] && git check-ignore -q secrets/db.env; } \
  && pass 'secrets/db.env bi git bo qua' || fail 'secrets/ khong duoc .gitignore bao ve'
ENVS=" $(jp pod/db '{.spec.containers[?(@.name=="postgres")].env[*].name}') "
case "$ENVS" in
  *' POSTGRES_PASSWORD '*)      fail 'db van nhan mat khau superuser qua bien POSTGRES_PASSWORD' ;;
  *' POSTGRES_PASSWORD_FILE '*) pass 'db doc mat khau superuser qua file' ;;
  *)                            fail 'db khong khai POSTGRES_PASSWORD_FILE' ;;
esac
[ "$(sql 'SHOW log_connections')" = 'on' ] && pass 'db ghi log ket noi' || fail 'db chua bat log_connections'

# Sprint 3: tài nguyên cho mọi container, kể cả init và sidecar (TD-05)
kubectl -n "$NS" get pod db api web -o jsonpath='{range .items[*]}{range .spec.initContainers[*]}{.name}={.resources.requests.cpu},{.resources.requests.memory},{.resources.limits.cpu},{.resources.limits.memory}{"\n"}{end}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.requests.memory},{.resources.limits.cpu},{.resources.limits.memory}{"\n"}{end}{end}' > "$TMP" 2>/dev/null
N="$(grep -c '=' "$TMP")"
{ [ "$N" -ge 6 ] && ! grep -Eq '=,|,,|,$' "$TMP"; } && pass "ca $N container co du requests va limits" \
  || fail "container thieu requests/limits: $(grep -E '=,|,,|,$' "$TMP" | cut -d= -f1 | tr '\n' ' ')"
for X in db:Guaranteed api:Burstable web:Burstable; do
  Q="$(jp "pod/${X%%:*}" '{.status.qosClass}')"
  [ "$Q" = "${X##*:}" ] && pass "${X%%:*} QoS $Q" || fail "${X%%:*} QoS ${Q:-?}, thiet ke la ${X##*:}"
done
for X in db/postgres api/postgrest web/nginx web/status-writer; do
  P="${X%/*}"; C="${X#*/}"
  L="$(jp "pod/$P" "{.spec.containers[?(@.name==\"$C\")].resources.limits.memory}{.spec.initContainers[?(@.name==\"$C\")].resources.limits.memory}")"
  M="$(cg "$P" "$C" memory.max)"
  { [ -n "$L" ] && [ "$M" = "$(by "$L")" ]; } && pass "$X memory.max tren node khop limits ($L)" \
    || fail "$X memory.max=${M:-?} khong khop limits ${L:-?}"
done
[ "$(jp pod/api '{.spec.containers[?(@.name=="postgrest")].resizePolicy[?(@.resourceName=="cpu")].restartPolicy}')" = 'NotRequired' ] \
  && pass 'postgrest doi CPU tai cho khong can khoi dong lai' || fail 'postgrest thieu khai bao resize CPU NotRequired'

# Sprint 3: PodDisruptionBudget web, api
for P in web api; do
  S=''
  for i in $(seq 1 12); do
    S="$(jp "pdb/$P" '{.metadata.labels.app\.kubernetes\.io/part-of} {.spec.minAvailable} {.status.expectedPods} {.status.currentHealthy} {.status.disruptionsAllowed}')"
    [ "$S" = 'bookstore 1 1 1 0' ] && break; sleep 5
  done
  [ "$S" = 'bookstore 1 1 1 0' ] && pass "PDB $P: minAvailable 1, dung 1 Pod khoe, ngan sach 0" \
    || fail "PDB $P sai hoac chua dong bo: '$S'"
  printf '{"apiVersion":"policy/v1","kind":"Eviction","metadata":{"name":"%s","namespace":"%s"},"deleteOptions":{"dryRun":["All"]}}\n' "$P" "$NS" > "$TMP"
  kubectl create --raw "/api/v1/namespaces/$NS/pods/$P/eviction" -f "$TMP" 2>&1 | grep -q 'disruption budget' \
    && pass "Eviction API (dry-run) voi $P bi PDB chan" || fail "Eviction API voi $P khong bi PDB chan"
done

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

Yêu cầu trục xuất trong smoke mang `dryRun: ["All"]`: API server kiểm PDB như thật nhưng không
xóa gì — smoke không bao giờ làm hỏng hệ thống nó đang kiểm.

2. Kiểm `docs/tech-debt.md`: `TD-05` đã trả (B2). Thêm một dòng ghi chú ở `TD-01`: PDB `minAvailable: 1`
   trên Pod trần chặn mọi trục xuất tự nguyện; Lab 4a xem lại PDB khi có bản sao.

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/3c/b10-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/3c/b10-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/3c/b10-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-3c: sprint 3 — dinh co tai nguyen, QoS, resize tai cho, PDB'
git tag lab-3c
git bundle create ~/bookstore-lab-3c.bundle --all && git bundle verify ~/bookstore-lab-3c.bundle
```

Chép `~/bookstore-lab-3c.bundle` về máy host bằng `scp` trước khi đi tiếp. Bundle **không** chứa
`secrets/db.env` — đúng chủ đích ([mục 5.4](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm)).

5. Dọn object thử nghiệm, phần chạm ra ngoài namespace, rồi trả cluster:

```bash
kubectl -n bookstore delete pod,pdb -l '!app.kubernetes.io/part-of' --ignore-not-found --wait=true
kubectl get node lab-k8s-worker2 -o jsonpath='{.status.capacity}' | grep -q 'dongle' \
  && kubectl patch node lab-k8s-worker2 --subresource=status --type=json \
       -p='[{"op": "remove", "path": "/status/capacity/example.com~1dongle"}]'
rm -rf ~/lab-work/3c
cd ~/bookstore && ./scripts/teardown.sh
```

`teardown.sh` của Lab 3b không đổi: PodDisruptionBudget nằm trong namespace `bookstore` nên bị xóa
cùng namespace.

6. Gate cuối — nhập mật khẩu `sudo` khi được hỏi:

```bash
SEL='app.kubernetes.io/part-of=bookstore'
cd ~/bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-3c' && echo 'PASS: HEAD mang tag lab-3c'
git bundle verify ~/bookstore-lab-3c.bundle >/dev/null 2>&1 && echo 'PASS: bundle hop le'
{ test -f k8s/app/web/pdb.yaml && test -f k8s/app/api/pdb.yaml; } && echo 'PASS: bo cuc repo dung hang 3c cua muc 5.5'
grep -E 'TD-05.*đã trả' docs/tech-debt.md >/dev/null && echo 'PASS: tech-debt.md ghi TD-05 da tra'
BAD=0
for F in 'adr/*-ngan-sach-tai-nguyen.md' 'adr/*-qos-tung-thanh-phan.md' 'adr/*-tai-nguyen-hieu-dung.md' \
         'adr/*-resize-tai-cho.md' 'adr/*-pdb.md' 'adr/*-static-pod.md' 'postmortem/3c-vuot-tran.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR va postmortem cua sprint 3'
test "$( { kubectl get namespace,priorityclass -l "$SEL" --no-headers
           kubectl get pods,configmaps,secrets -A -l "$SEL" --no-headers; } 2>/dev/null | wc -l)" -eq 0 \
  && echo 'PASS: khong con object nao cua du an'
test ! -e ~/lab-work/3c && echo 'PASS: da don lab-work'
kubectl get node lab-k8s-worker2 -o jsonpath='{.status.capacity}' | grep -q 'dongle' \
  && echo 'FAIL: worker2 con extended resource' || echo 'PASS: worker2 khong con extended resource'
ssh lab-k8s-worker2 'sudo -S ls -A /etc/kubernetes/manifests 2>/dev/null' | grep -q 'lab3c' \
  && echo 'FAIL: worker2 con static Pod cua lab' || echo 'PASS: worker2 khong con static Pod cua lab'
test "$(sudo ls -1 /etc/kubernetes/manifests | grep -c '\.yaml$')" -eq 4 && echo 'PASS: bon manifest control plane nguyen ven'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** `smoke exit=0` và mọi dòng của smoke ở bước 3 là `PASS:`; mười một dòng `PASS:` đầu của gate
cuối xuất hiện, không dòng `FAIL:`; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector trả
`No resources found`; CoreDNS đủ replica; namespace `default` không có Pod. Thư mục
`~/lab-evidence/3c/` **giữ lại** — nó không chứa mật khẩu thật. Cluster trở về `01-cluster-ready`;
**không tạo snapshot mới**.

---

## 3. Checkpoint 3c

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Kể hành trình con số CPU của `db`: từ manifest, qua scheduler — so với con số nào của node —,
      qua kubelet, tới hai file cgroup nào trên `lab-k8s-worker2`. Con số memory tới file nào, và
      kernel áp nó theo kiểu gì?
- [ ] Ở B1, bạn đo mức dùng thế nào khi chưa có `kubectl top`? Ba luật ngân sách của dự án là gì, vì
      sao memory có luật "gấp đôi" mà CPU thì không?
- [ ] Vì sao `db` là `Guaranteed` còn `web`, `api` là `Burstable`? Thêm vào `db` một init container
      không khai tài nguyên thì sao? Chỉ khai limit mà không khai request thì sao?
- [ ] Request hiệu dụng của Pod `web` của bạn bằng bao nhiêu? Vì sao `render-html` không cộng vào còn
      `status-writer` thì cộng? `thu-init` và `thu-sidecar` khác đúng một trường — kết quả khác thế nào?
- [ ] `thu-throttle` và `thu-oom`: cái gì chết, cái gì không? Khi `status-writer` bị `OOMKilled`,
      `nginx` ra sao — và khác gì khi cả Pod bị trục xuất? Limit CPU có giết container gián tiếp được
      không?
- [ ] `thu-het-cho` xin đúng bằng `allocatable` mà vẫn `Pending` dù node gần như rảnh — vì sao? Thêm
      một worker giống hệt cứu được `thu-qua-suc` không?
- [ ] Sau khi gấp đôi CPU của `api`: trường nào là mong muốn, trường nào là thực tế, bằng chứng nào cho
      thấy container không khởi động lại? `Infeasible` khác `Deferred` thế nào?
- [ ] Kể năm thứ resize không làm được trên hệ thống của bạn. Vì sao `deploy.sh` vẫn chạy được nếu quên
      đồng bộ repo, và vì sao đó lại là điều nguy hiểm?
- [ ] PDB `web`: vì sao chỉ được `minAvailable` số nguyên, ngân sách bằng mấy, và điều đó nghĩa là gì
      cho buổi bảo trì ở Lab 12? Eviction API và `kubectl delete` khác nhau thế nào trước PDB?
- [ ] Mất điện `lab-k8s-worker2` có bị PDB chặn không, có bị tính vào ngân sách không? Vì sao hợp đồng
      không đặt PDB cho `db`?
- [ ] Xóa mirror Pod bằng `kubectl` thì sao, xóa file trên node thì sao? Vì sao kubeadm chạy
      `kube-apiserver` bằng static Pod, còn agent của dự án nên chờ DaemonSet ở Lab 4b?
- [ ] (⭐) Tài nguyên cấp Pod đổi cách tính QoS thế nào, và vì sao Pod dùng nó không resize được trên
      baseline? Extended resource có làm Pod thoát `BestEffort` không?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- **Nợ lab đã trả ở sprint này:** request/limit hiệu dụng của init container và sidecar (B3), thực
  hành static Pod (B7) — hai nợ của Lab 3a.
- **Nợ lab còn mở:** phase `Unknown` (Lab 12) và readiness gỡ Pod khỏi EndpointSlice (Lab 5a) của Lab
  3a; [nợ lab #6](README.md#5-sổ-nợ-lab) — mã hóa Secret at rest — của Lab 3b.
- **Phần hoãn của lab này** (mục 1.1): `drain` và PDB (Lab 12); trục xuất do áp lực node, chiếm chỗ
  (Lab 7a); Metrics API (Lab 11a); quảng bá bằng device plugin (giai đoạn 14, đánh giá ở Lab 13);
  resize tài nguyên cấp Pod — không kiểm chứng được trong lộ trình.
- **Nợ kỹ thuật:** `TD-05` **đã trả**. `TD-01` (Pod trần — Lab 4a, 4b; ghi chú: PDB `minAvailable: 1`
  chặn mọi trục xuất tự nguyện cho tới khi có bản sao), `TD-03` (Lab 5a), `TD-04` (Lab 6a), `TD-06`
  (Lab 9b), `TD-07` (Lab 9a) vẫn mở.

### Bài giải thích cuối cùng

Trong vài phút, kể lại một cuối tuần khuyến mãi của Bookstore trên hệ thống bạn vừa dựng:

1. **Trước khuyến mãi.** Ngân sách đến từ đâu: số đo nào, luật nào, ADR nào. Pod `web` xin bao nhiêu
   chỗ trên node — tính cả init container và sidecar — và kernel áp những trần nào lên từng container,
   lên cả Pod.
2. **Khuyến mãi bắt đầu.** `api` cần thêm CPU: bạn đổi tại chỗ, kiểm ba nơi để biết nó đã có hiệu lực,
   và đưa quyết định về repo. Một con số bất khả thi thì hệ thống báo thế nào.
3. **Sự cố giữa chừng.** `status-writer` rò bộ nhớ: ai giết nó, `nginx` có sao không, người dùng thấy
   gì. Một Pod mới không lên được vì thiếu chỗ: đọc lý do ở đâu, và vì sao node "rảnh" vẫn từ chối.
4. **Sáng thứ Hai.** Đội hạ tầng muốn bảo trì `lab-k8s-worker1`: yêu cầu trục xuất đi tới đâu thì dừng,
   vì sao, và điều gì chưa được bảo vệ (`kubectl delete`, mất node, `db`).

Kết thúc bằng việc chỉ ra Lab 4a sẽ thay đổi câu chuyện ở đâu (bản sao, PDB mới, Pod được tạo lại) và
Lab 7a, 7b, 11a sẽ dùng tiếp những con số bạn vừa chốt thế nào.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B1: hàm `cg` không in gì | Đọc sai node, hoặc container chưa chạy nên `containerID` rỗng | `kubectl -n bookstore get pod -o wide`; chờ Pod Ready rồi chạy lại. Node phải là cgroup v2 với driver `systemd` — [Lab 2 B2](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md#b2-cgroup-v2-và-cgroup-driver) |
| B1: `thu-tai` chạy rất ít vòng hoặc log trống | Pod trên worker2 không gọi được IP Pod ở worker1 | Sự cố mạng Pod xuyên node — chạy lại tầng 3 của gate A5.4 ở Lab 00 |
| B1: `kubectl run` báo lỗi `--overrides` | Gõ sai JSON, thiếu dấu nháy đơn bao ngoài | Chép nguyên khối của script nghiệm thu |
| B2: `deploy.sh` báo `pod updates may not change fields other than ...` | Pod trần cũ còn chạy với tài nguyên cũ | `./scripts/teardown.sh && ./scripts/deploy.sh` |
| B2: `db` không Ready, `postgres` bị `OOMKilled` lúc khởi tạo | Limit memory quá thấp cho bước khởi tạo dữ liệu | Nâng limit (vẫn giữ request bằng limit), ghi lý do vào ADR 0018 |
| B2: `api` treo `Init:0/1` rất lâu | Limit CPU của `wait-for-db` quá nhỏ hoặc `db` chưa Ready | `kubectl logs api -c wait-for-db`; nâng limit init, chờ `db` |
| B2: `cpu.max` khớp nhưng `memory.max` lệch | Viết `64M` thay vì `64Mi` | `M` là thập phân, `Mi` là nhị phân (bài 264) |
| B2: quota trong `cpu.max` lớn hơn limit | Limit CPU dưới `10m` bị làm tròn lên mức quota tối thiểu 1 ms | Không đặt limit CPU dưới `10m` |
| B2: luật ngân sách FAIL ở một container | Con số chốt thấp hơn số đo | Nâng con số đó; nếu số đo bất thường, chạy lại phép đo B1 và ghi cả hai lần vào ADR |
| B3: dry-run báo `AlreadyExists` | Biến thể quên đổi tên Pod | Đổi `metadata.name` thành `qos-<n>` |
| B3: `thu-init` không chạy được | Còn Pod thử nghiệm cũ trên worker2 chiếm chỗ, F đo ra đã lỗi thời | Xóa Pod `thu-*` còn sót rồi chạy lại phần 3 của script |
| B3: `thu-sidecar` lại chạy được | Container phụ thiếu `restartPolicy: Always` — nó là init thường | Thêm trường đó ở cấp container phụ |
| B3: `memory.max` cấp Pod là `max` | Có container — kể cả init — thiếu limit memory | Kiểm `b2-tai-nguyen.txt`; khai đủ rồi teardown, deploy |
| B4: `nr_throttled` không tăng | Thiếu limit CPU, hoặc Pod chưa chạy | `cpu.max` phải khác `max`; chờ Pod `Running` |
| B4: sidecar của `thu-oom` không bao giờ `OOMKilled` | Swap đang bật trên worker2, hoặc `bs` không lớn hơn limit | `swapon --show` phải rỗng ([gate A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready)); tăng `bs` |
| B4: `nginx` của `thu-oom` không chạy | Sidecar chết trước khi kubelet kịp khởi động container chính | Giữ `sleep 15` trước lệnh `dd` |
| B5: `error: invalid subresource` | Client `kubectl` cũ hơn mốc mà bài 289 nêu | Dùng `kubectl` của baseline trên `lab-k8s-master` |
| B5: lần đổi CPU nằm ở `Deferred` | Request mới lớn hơn phần còn trống của node | Giảm con số, hoặc xóa Pod thử nghiệm còn sót trên node đó |
| B5: `restartCount` của `postgrest` tăng | Patch có đổi cả memory; chính sách memory là khởi động lại | Chỉ patch CPU; ghi hiện tượng vào ADR 0021 |
| B5: phần 3 báo repo lệch giá trị đang chạy | Quên đồng bộ repo, hoặc viết khác đơn vị | So bằng lệnh dry-run phía client ở gợi ý 4 |
| B6: `ALLOWED DISRUPTIONS` của `web` trống hoặc `expectedPods` bằng 0 | Selector của PDB không khớp label của Pod | So `spec.selector` với `kubectl get pod web --show-labels` |
| B6: trục xuất thứ nhất của nhóm thử nghiệm bị từ chối | PDB chưa được disruption controller xử lý xong, hoặc một Pod chưa Ready | Chờ ngân sách bằng 1 rồi gửi lại |
| B7: mirror Pod không xuất hiện | YAML sai, file đặt sai thư mục | `ssh lab-k8s-worker2 'sudo -S journalctl -u kubelet -n 50'`; đặt file đúng `staticPodPath` |
| B7: `sudo: a terminal is required to read the password` | Gọi `sudo` qua `ssh` mà thiếu `-S` | Dùng `sudo -S` ([A2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a2-cài-ubuntu-và-cấu-hình-identity)) |
| B7: `containerID` đổi sau khi xóa mirror Pod | Kubelet khởi động lại container vì lý do khác | Kết luận của ticket không đổi; ghi hiện tượng vào ADR 0023, không đụng manifest của control plane |
| B8 ⭐: dry-run không giữ `spec.resources` | Cluster của bạn tắt `PodLevelResources`, lệch baseline | Ghi vào ADR 0024; không bật gate trong lab này |
| B9 ⭐: `allocatable` không hiện dongle | Kubelet cập nhật `allocatable` bất đồng bộ; hoặc đường dẫn JSON-Patch sai chỗ `~1` | Chờ hết vòng lặp; kiểm `status.capacity` trước |
| B10: namespace kẹt `Terminating` | `db` đang trong 60 giây ân hạn | Chờ; `teardown.sh` chờ tối đa 300 giây |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Resource Management for Pods and Containers](https://v1-35.docs.kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Kubernetes v1.35 — Pod Quality of Service Classes](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/pod-qos/)
- [Kubernetes v1.35 — Disruptions](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/disruptions/)
- [Kubernetes v1.35 — Static Pods](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/static-pods/)
- [Kubernetes v1.35 — Init Containers, Resource sharing within containers](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/init-containers/#resource-sharing-within-containers)
- [Kubernetes v1.35 — Sidecar Containers](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
- [Kubernetes v1.35 — Assign CPU Resources to Containers and Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)
- [Kubernetes v1.35 — Assign Memory Resources to Containers and Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/assign-memory-resource/)
- [Kubernetes v1.35 — Assign Pod-level CPU and memory resources](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/assign-pod-level-resources/)
- [Kubernetes v1.35 — Assign Extended Resources to a Container](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/extended-resource/)
- [Kubernetes v1.35 — Configure Quality of Service for Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Kubernetes v1.35 — Resize CPU and Memory Resources assigned to Containers](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)
- [Kubernetes v1.35 — Resize CPU and Memory Resources assigned to Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/resize-pod-resources/)
- [Kubernetes v1.35 — Specifying a Disruption Budget for your Application](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Kubernetes v1.35 — API-initiated Eviction](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)
- [Kubernetes v1.35 — Create static Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
- [Kubernetes v1.35 — Feature Gates](https://v1-35.docs.kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) (`PodLevelResources`, `InPlacePodVerticalScaling`, `InPlacePodLevelResourcesVerticalScaling`)
- [Linux kernel — Control Group v2](https://docs.kernel.org/admin-guide/cgroup-v2.html) (`cpu.max`, `cpu.stat`, `cpu.weight`, `memory.max`, `memory.current`, `memory.peak`)
