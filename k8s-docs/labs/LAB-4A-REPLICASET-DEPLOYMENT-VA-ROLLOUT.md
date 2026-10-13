# Lab 4a — ReplicaSet, Deployment và rollout

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> Repo `~/bookstore` ở tag `lab-3c` (hoặc bundle `~/bookstore-lab-3c.bundle` nếu bạn vừa restore
> snapshot).
> **Điểm kết thúc:** teardown trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới. Repo
> `~/bookstore` có tag `lab-4a` và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 4 — Lên Deployment của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 3c — Tài nguyên, QoS và gián đoạn](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md).
> **Lab sau:** [Lab 4b — StatefulSet, DaemonSet và Job](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng mục
[4a. ReplicaSet, Deployment và rollout](../00-ALO-TRINH-ADMIN.md#4a-replicaset-deployment-và-rollout)
của [Giai đoạn 4](../00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller). Đây là sprint thứ tư
của [dự án Bookstore](DU-AN-BOOKSTORE.md). Trước khi mở B0, đọc lại
[mục 4 — hợp đồng kỹ thuật](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật), hàng "4a" của
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint) và
[kế hoạch sprint 4](DU-AN-BOOKSTORE.md#sprint-4--lab-4a-lên-deployment).

Ba sprint đầu để lại ba Pod trần chạy đúng vòng đời, cấu hình nằm ngoài code, tài nguyên đã định
cỡ. Nhưng không ai đứng sau chúng: Pod `web` mất là trang web mất cho tới khi có người chạy lại
`deploy.sh` (`TD-01`), và mỗi lần đổi spec là một lần xóa rồi tạo lại — đúng vấn đề `P5` của hệ
thống cũ: phát hành bằng tay, dừng cả hệ thống. Sprint này đưa `web` và `api` lên Deployment, dựng
đường phát hành không gián đoạn, có lý do ghi trong lịch sử, có đường lùi, và diễn tập một bản phát
hành hỏng. `db` vẫn là Pod trần: nó cần một controller giữ **danh tính** chứ không chỉ giữ số lượng —
việc của Lab 4b.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); image của dự án nằm
ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Lab không chép lại
con số phiên bản nào trong phần văn bản và **không cài thêm gì**.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi `web` (3 bản sao) và `api` (2 bản sao) của Bookstore chạy dưới Deployment,
phát hành được mà không gián đoạn, **và** bạn chứng minh, giải thích được:

- ReplicaSet gồm đúng ba thứ — selector, số bản sao, template — và nó đếm **Pod khớp selector**
  chứ không đếm "Pod tôi tạo": nó thu nhận cả Pod trần, nhận nuôi Pod cũ mà **không** cập nhật
  chúng theo template mới. Chính chỗ dừng đó là lý do có Deployment.
- Chuỗi Pod → ReplicaSet → Deployment của `web` bằng UID thật; `pod-template-hash` nằm ở đâu và
  vì sao nhờ nó mà Deployment `web` không nuốt một Pod trần mang đúng label của `web`.
- Rollout — và revision — **chỉ** sinh ra khi Pod template đổi. Vì thế sửa trang mẫu trong
  `web-config` không phát hành gì cả, ba bản sao có thể phục vụ hai nội dung khác nhau mà
  Deployment vẫn báo mọi thứ ổn; và `rollout undo` không lấy lại nội dung cũ.
- `maxSurge`, `maxUnavailable`, `minReadySeconds` và readiness quy định khoảng dao động của một
  lần phát hành — đo được trên `web` rằng số bản sao sẵn sàng không bao giờ dưới 3.
- `kubernetes.io/change-cause` vào lịch sử ra sao; lịch sử nằm trong các ReplicaSet cũ và
  `revisionHistoryLimit` quyết định bạn còn lùi được tới đâu.
- Một bản phát hành hỏng: bản sao mới kẹt, bản cũ vẫn phục vụ, `Progressing=False` với
  `ProgressDeadlineExceeded`, `deploy.sh` thoát khác 0 — và Kubernetes **không** tự rollback. Sau
  `rollout undo`, cluster lệch khỏi repo cho tới khi repo được sửa.
- `Recreate` so với `RollingUpdate` bằng số đo, và vì sao bảo đảm của `Recreate` không áp dụng khi
  xóa tay một Pod.
- Ranh giới giữa `kubectl apply`, `kubectl scale` và `kubectl patch`: thay đổi nào lần triển khai
  sau ghi đè, thay đổi nào **trôi** lại trên cluster mà `kubectl diff` không thấy.
- Xóa Deployment kiểu `orphan` rồi tạo lại: Deployment mới nhận lại ReplicaSet cũ, không Pod nào bị
  thay. Ba kiểu xóa theo tầng khác nhau ở đâu.
- PDB của `web`, `api` giờ tính theo số bản sao của Deployment với `maxUnavailable: 1` — và vì sao
  `minAvailable` bằng số bản sao sẽ chặn drain node; Eviction API chặn gián đoạn thứ hai.
- Canary thủ công là hai Deployment khác nhau ở **một** label, tỷ lệ do số bản sao quyết định — và
  vì sao tới khi có Service nó vẫn chưa phải là chia lưu lượng.
- Nhận ra `ReplicationController` trên một cluster kế thừa và biết nó khác ReplicaSet ở đúng chỗ
  nào.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 4a | Ticket kiểm chứng |
| --- | --- |
| [62 — Quản lý Workload](../62-controllers-index-vi.md) | B1, B2 — khai báo một object cao hơn Pod, control plane tạo và xóa Pod thay bạn; Deployment cho thành phần stateless `web`, `api`, còn `db` cần định danh nên chờ Lab 4b |
| [64 — ReplicaSet](../64-replicaset-vi.md) | B1 — selector, số bản sao, template; `ownerReferences`; thu nhận Pod trần; hai ràng buộc API; xóa `orphan` rồi nhận nuôi mà không cập nhật; B2 — ReplicaSet do Deployment tạo không thu nhận Pod trần cùng label |
| [63 — Deployment](../63-deployment-vi.md) | B2 — chuỗi sở hữu, `pod-template-hash`, tự phục hồi; B4 — `maxSurge`/`maxUnavailable`, `minReadySeconds`, `progressDeadlineSeconds`, `revisionHistoryLimit`, `CHANGE-CAUSE`; B5 — rollout chỉ khi `.spec.template` đổi, rollback chỉ hoàn nguyên template; B6 — Deployment thất bại, không tự rollback, `rollout undo`; B7 — `Recreate`, ⭐ pause/resume, ⭐ dọn lịch sử; B8 — `replicas` trong manifest ghi đè scale tay; B9 — Deployment mới tiếp nhận ReplicaSet cũ |
| [61 — Quản lý Workload bằng kubectl](../61-management-vi.md) | B4, B6 — `rollout status --timeout`, `--watch=false`; B8 — `apply`, `patch`, ⭐ `replace --force`; B10 — canary bằng label `track`, `-R`, thao tác hàng loạt bằng `-l` |
| [70 — ReplicationController](../70-replicationcontroller-vi.md) | B1 — **chỉ nhận diện** trên API: tên viết tắt `rc`, selector phẳng; lab không tạo `rc` nào vì đây là tài liệu lịch sử |
| [260 — Sử dụng xóa theo tầng trong Cluster](../260-use-cascading-deletion-vi.md) | B9 — `ownerReferences` trên Pod của `web`; `orphan` trên Deployment `web` thật rồi nhận lại; foreground và background trên Deployment thử nghiệm |
| [319 — Quản lý object khai báo bằng file cấu hình](../319-declarative-config-vi.md) | B4 — chiến lược phát hành viết tường minh; B7 — vì sao đổi sang `Recreate` qua repo không vấp lỗi của `patch`; B8 — `last-applied-configuration`, ba nguồn của phép merge, `kubectl diff`, trôi cấu hình và cách gỡ |
| [324 — Cập nhật tại chỗ bằng kubectl patch](../324-kubectl-patch-vi.md) | B7 — `$retainKeys` khi đổi sang `Recreate`; B8 — strategic merge patch so với JSON merge patch trên list `containers`, `--subresource=scale`, JSON patch có phép `test` |
| [337 — Chạy ứng dụng](../337-run-application-vi.md) | **Không thực hành riêng.** Trang mục lục của nhánh `run-application`, toàn bộ nội dung là danh sách link. Ba trang con thuộc nhóm (345, 346, 348) nằm ở các dòng dưới; các trang con còn lại thuộc nhóm 4b và giai đoạn sau |
| [345 — Chạy ứng dụng stateless bằng Deployment](../345-run-stateless-application-vi.md) | B2 — tạo Deployment từ manifest, `describe`, liệt kê Pod theo label; B5 — cập nhật bằng cách apply lại cùng file: Pod mới tên mới, Pod cũ bị xóa |
| [346 — Scale thủ công một Deployment](../346-scale-deployment-vi.md) | B3 — scale `web` tạm thời để thấy ngân sách PDB đi theo số bản sao; B8 — `kubectl scale`, scale bằng `patch`, JSON patch có `test`, scale về 0 giữ Deployment và ReplicaSet |
| [348 — Cập nhật Deployment không gián đoạn](../348-update-deployment-rolling-vi.md) | B4 — rolling update bằng `kubectl apply`, theo dõi bằng `rollout status`, tham số chiến lược; B6 — phát hiện rollout đình trệ, `rollout history`, `undo`; B7 ⭐ — pause và resume |

Các phần dưới đây bị hoãn có chủ đích — nói được vì sao ở checkpoint:

| Phần hoãn | Vì cần | Trả ở |
| --- | --- | --- |
| Canary chia **lưu lượng thật**: Service chọn tập con label chung của hai bản phát hành (bài 61, mục *Triển khai canary*; bài 63, *Deployment kiểu canary*) | Service, giai đoạn 5 | Lab 5a — B10 chỉ dựng cấu trúc label và tỷ lệ bản sao |
| Đo gián đoạn ở mức **request** trong lúc rollout (client gọi một địa chỉ ổn định) | Service và EndpointSlice | Lab 5a B3 (rollback và phát hành lại `web`); thay toàn bộ Pod `api` đo ở Lab 12 B6. B4 ở đây đo ở mức controller (`availableReplicas`) |
| `kubectl autoscale`, HPA làm chủ `replicas` (bài 61, 63, 64, 346) | metrics-server, giai đoạn 11 | **[Nợ lab #1](README.md#5-sổ-nợ-lab)**, Lab 11b |
| *Scale theo tỷ lệ* khi autoscaler scale giữa chừng một rollout (bài 63) | HPA | đọc ở bài 72, Lab 11b |
| `.status.terminatingReplicas` (bài 63, 64) | feature gate `DeploymentReplicaSetTerminatingReplicas`, không bật ở baseline | không thực hành |
| `kubectl apply --prune` (bài 319) | alpha; dự án triển khai qua `deploy.sh` | không thực hành |

Hai mục của bài [339](../339-configure-pdb-vi.md) mà Lab 3c để lại cho giai đoạn 4 — *Xác định ứng
dụng cần bảo vệ* và cách ngân sách tính trên số bản sao mong muốn của controller — được trả ở B3.
Nợ lab của các sprint trước (phase `Unknown` — Lab 12; readiness loại Pod khỏi backend nhận lưu lượng — Lab 5a;
nợ lab #6 — giai đoạn 22) không đổi.

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B10 trừ các mục ⭐, cộng đóng sprint) khoảng 4 giờ nếu không bí lâu ở
ticket nào. B4, B6, B7 có đoạn chờ rollout và chờ deadline của controller — thời gian phụ thuộc cấu
hình bạn đặt. Có thể chia hai phiên: phiên 1 làm B0–B5 (hệ thống mới), phiên 2 làm B6–B10 và đóng
sprint. Giữa hai phiên cứ để nguyên cluster; đầu phiên 2 chạy lại gate ngắn A5.5 và
`./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Mọi Deployment và Pod thử nghiệm, nhất là bản phát
  hành hỏng ở B6, ghim vào node này bằng `nodeSelector`.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/4a/`. Bằng chứng: `~/lab-evidence/4a/`.
- Object thử nghiệm đặt tên `thu-*` hoặc `drill-*`, nằm trong namespace `bookstore` và **không** mang
  label `app.kubernetes.io/part-of` — để không chồng selector với `web`, `api`, PDB của dự án, và để
  teardown của dự án không phải là nơi dọn chúng. Ngoại lệ duy nhất là Pod `thu-web-tran` ở B2, tồn
  tại vài giây rồi bị xóa. Xóa mọi object thử nghiệm khi xong ticket.
- Lab chỉ dùng kiến thức nhóm 4a và các nhóm trước: Pod (3a), ConfigMap, Secret, Kustomize cho
  Secret (3b), `requests`/`limits`, QoS, PDB, Eviction API (3c). **Không** dùng StatefulSet, Job,
  DaemonSet (4b), Service (5a), HPA (11b), PVC (6a).
- **Repo là nguồn sự thật duy nhất** ([mục 5.3](DU-AN-BOOKSTORE.md#53-quy-tắc-làm-việc-với-repo)).
  `kubectl scale`, `patch`, `set`, `rollout undo` trên `web` và `api` chỉ xuất hiện trong ticket
  *Điều tra*, *Diễn tập* và trong script nghiệm thu, và luôn được trả về repo trước khi ticket đóng.
- `db` vẫn là Pod trần trên `emptyDir`: đừng xóa nó ngoài `teardown.sh` — `api` trỏ tới IP của nó
  (`TD-03`) và dữ liệu mất theo (`TD-04`).
- Lab vẫn chỉ có **một** object cấp cluster: PriorityClass `bookstore-critical`.
- Không sửa cấu hình node, không cài gói. Lệnh qua `ssh` tới worker chỉ để đọc, trừ `crictl pull` ở
  B0.
- Chạy mỗi script nghiệm thu trong một shell liền mạch: nhiều script đo trạng thái giữa hai lệnh.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 4: Lên Deployment

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `01-cluster-ready`, repo ở đúng tag `lab-3c`, và hệ thống của
sprint 3 dựng lại được **chỉ từ repo**.

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
mkdir -p ~/lab-work/4a ~/lab-evidence/4a
test -d ~/bookstore/.git || git clone ~/bookstore-lab-3c.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-3c' && echo 'PASS: HEAD dung tag lab-3c'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
{ test -f k8s/app/web/pod.yaml && test -f k8s/app/api/pod.yaml.tpl && test -f k8s/app/db/pod.yaml; } \
  && echo 'PASS: bo cuc repo dung hang 3c cua muc 5.5'
```

Kéo sẵn image lên hai worker — sprint này tạo nhiều Pod mới liên tục, và thời gian kéo image không
được lẫn vào đồng hồ deadline của controller. Tag lấy đúng từ
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

Dựng lại hệ thống của sprint 3 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/4a/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/4a/b0-smoke-lab-3c.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/4a/b0-smoke-lab-3c.txt
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; bốn dòng `PASS:` về repo và cluster; đủ tám dòng
`PASS: ... co ...`; `deploy exit=0`, `smoke exit=0` và file smoke không có dòng `FAIL:`.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 3 **chưa đủ** — có thứ chỉ tồn tại trên
cluster cũ. Sửa repo ở tag `lab-3c` trước, không sửa tay trên cluster
([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-4A-01] Điều tra: ReplicaSet trần và chỗ nó dừng lại

**Loại:** Điều tra.

**Bối cảnh:** Deployment vận hành **thông qua** ReplicaSet — mọi việc nó làm khi phát hành đều quy về
scale các ReplicaSet. Trước khi giao `web` cho một Deployment, đội phải thấy tận mắt ReplicaSet đếm
Pod thế nào, nó nhận vơ những Pod nào, và nó **không** làm được gì. Thí nghiệm trên object riêng,
không đụng hệ thống.

**Yêu cầu:**

1. Hai Pod trần `thu-tran-1`, `thu-tran-2` (`busybox`, chạy lâu, thoát ngay khi nhận `TERM`) mang
   label `thu: rs`, ghim `lab-k8s-worker2`, tạo **trước**. Sau đó ReplicaSet `thu-rs`, 3 bản sao,
   chọn đúng label đó, template cùng kiểu container. Chứng minh: tổng 3 Pod, ReplicaSet chỉ tạo thêm
   1; hai Pod trần giờ trỏ về `thu-rs` bằng UID.
2. Xóa tay một Pod do `thu-rs` tạo: một Pod **khác** thay vào. Tạo thêm Pod trần `thu-tran-3` cùng
   label: nó bị xóa ngay. Chỉ ra hai loại sự kiện của controller tương ứng.
3. Bằng server dry-run, chứng minh API từ chối hai ReplicaSet sai: template không khớp selector;
   `restartPolicy` của template khác `Always`.
4. Xóa `thu-rs` mà **giữ** Pod của nó. Tạo `thu-rs-v2` cùng selector, template khác ở đúng một chỗ:
   thêm biến môi trường `PHIEN_BAN=2`. Chứng minh `thu-rs-v2` nhận nuôi ba Pod cũ, không tạo Pod mới,
   và không Pod nào mang biến mới.
5. Nhận diện ReplicationController trên API: tên viết tắt, và selector của nó khác ReplicaSet ở
   đâu. Không tạo `rc` nào.
6. Xóa mọi object thử nghiệm.

**Ràng buộc:** object thử nghiệm không mang label `app.kubernetes.io/part-of`. Chỉ dùng image
`busybox` đúng tag A1.3 — đổi template bằng biến môi trường chứ không bằng image khác, để không phải
kéo image mới.

**Đọc lại:**

- Bài [62](../62-controllers-index-vi.md) — cả bài: vì sao có tầng workload object.
- Bài [64](../64-replicaset-vi.md): [*Cách một ReplicaSet hoạt động*](../64-replicaset-vi.md#how-a-replicaset-works),
  [*Thu nhận các Pod không theo template*](../64-replicaset-vi.md#thu-nhận-các-pod-không-theo-template-non-template-pod-acquisitions),
  [*Pod Template*](../64-replicaset-vi.md#pod-template),
  [*Bộ chọn Pod*](../64-replicaset-vi.md#bộ-chọn-pod-pod-selector),
  [*Chỉ xóa ReplicaSet*](../64-replicaset-vi.md#chỉ-xóa-replicaset-deleting-just-a-replicaset),
  [*Thay đổi quy mô một ReplicaSet*](../64-replicaset-vi.md#thay-đổi-quy-mô-một-replicaset-scaling-a-replicaset).
- Bài [70](../70-replicationcontroller-vi.md): [*Pod Selector*](../70-replicationcontroller-vi.md#pod-selector),
  [*Rolling update*](../70-replicationcontroller-vi.md#rolling-updates),
  [*Các lựa chọn thay thế*](../70-replicationcontroller-vi.md#các-lựa-chọn-thay-thế-cho-replicationcontroller-alternatives-to-replicationcontroller).

**Bàn giao:** `docs/adr/NNNN-replicaset.md` (`NNNN` là số tiếp theo trong `docs/adr/`);
`~/lab-evidence/4a/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` ReplicaSet thu nhận hai Pod trần và chỉ tạo thêm một; `ownerReferences` của Pod trần
  trỏ `thu-rs` đúng UID, `controller: true`.
- `PASS:` Pod bị xóa tay được thay bằng Pod khác tên; Pod trần thứ ba bị xóa; có sự kiện
  `SuccessfulCreate` và `SuccessfulDelete`.
- `PASS:` hai bản dry-run sai đều bị API server từ chối, lỗi nhắc đúng `selector` và `restartPolicy`.
- `PASS:` sau khi xóa `orphan`, ba Pod còn nguyên và không còn owner; `thu-rs-v2` nhận nuôi cả ba,
  không tạo Pod mới, không Pod nào có biến `PHIEN_BAN`.
- `PASS:` API có `replicationcontrollers` (viết tắt `rc`), selector của nó không có
  `matchExpressions`; cluster không có `rc` nào.
- `PASS:` không còn object thử nghiệm nào mang label `thu=rs`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-replicaset.md`:

1. Pod trần `web` của sprint 3 mang `app.kubernetes.io/part-of=bookstore` và
   `app.kubernetes.io/component=web`. Một đồng đội viết tay một ReplicaSet chọn đúng hai label đó,
   3 bản sao. Pod trần `web` ra sao, ReplicaSet tạo thêm mấy Pod, và Pod mới chạy template nào?
2. Selector và `ownerReferences` trả lời hai câu hỏi khác nhau nào? Vì sao `ownerReferences` phải mang
   UID chứ không chỉ tên?
3. Bước 4 cho thấy ReplicaSet **không** làm được việc gì mà mỗi lần phát hành `web` đều cần? Thứ gì
   trong Kubernetes làm việc đó, và làm bằng cách nào?
4. Khi `web` thu nhỏ từ 3 xuống 2, Pod nào bị xóa trước? Kể đủ thứ tự tiêu chí của bài 64.
5. Bạn tiếp quản một cluster cũ còn `rc` chạy giao diện web. Khác biệt kỹ thuật duy nhất với
   ReplicaSet là gì, và hướng xử lý của bạn?

<details>
<summary>Gợi ý</summary>

1. Container thoát ngay khi nhận `TERM`:
   `command: ["sh", "-c", "trap 'exit 0' TERM; sleep 3600 & wait"]`. Không có `trap`, mỗi lần xóa Pod
   phải chờ hết thời gian ân hạn.
2. Khung ReplicaSet: `apiVersion: apps/v1`, `spec.replicas`, `spec.selector.matchLabels`,
   `spec.template` (metadata với label, spec với container). So với ví dụ `frontend.yaml` của bài 64.
3. Xóa giữ Pod: `kubectl delete rs <tên> --cascade=orphan`. Garbage collector gỡ `ownerReferences` của
   Pod ở chế độ nền — chờ vài giây rồi mới kiểm.
4. Dry-run phía server không lưu gì: `kubectl apply --dry-run=server -f <file>`.
5. `kubectl api-resources` và `kubectl explain replicationcontroller.spec.selector`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã tự viết các manifest ở ~/lab-work/4a/; script này chính là cuộc điều tra</summary>

Cần sáu file: `thu-tran.yaml` (hai Pod trần), `thu-tran-3.yaml`, `thu-rs.yaml`, `thu-rs-v2.yaml`,
`thu-rs-sai-selector.yaml`, `thu-rs-sai-restart.yaml`.

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
NS=bookstore; EV=~/lab-evidence/4a; W=~/lab-work/4a
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
dem() { kubectl -n "$NS" get pods -l thu=rs --no-headers 2>/dev/null | grep -vc Terminating; }

# 1. Pod trần có trước, ReplicaSet tới sau
kubectl apply -f "$W/thu-tran.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/thu-tran-1 pod/thu-tran-2 --timeout=120s
[ -z "$(jp pod/thu-tran-1 '{.metadata.ownerReferences}')$(jp pod/thu-tran-2 '{.metadata.ownerReferences}')" ] \
  && echo 'PASS: hai Pod tran chua co owner'
kubectl apply -f "$W/thu-rs.yaml"
for i in $(seq 1 60); do [ "$(jp rs/thu-rs '{.status.readyReplicas}')" = '3' ] && break; sleep 2; done
kubectl -n "$NS" get pods -l thu=rs -o wide > "$EV/b1-thu-nhan.txt"
TAO="$(kubectl -n "$NS" get pods -l thu=rs -o name | grep -c 'pod/thu-rs-')"
{ [ "$(dem)" -eq 3 ] && [ "$TAO" -eq 1 ]; } && echo 'PASS: ReplicaSet thu nhan 2 Pod tran, chi tao them 1'
RS_UID="$(jp rs/thu-rs '{.metadata.uid}')"
[ "$(jp pod/thu-tran-1 '{.metadata.ownerReferences[0].uid}/{.metadata.ownerReferences[0].controller}')" = "$RS_UID/true" ] \
  && echo 'PASS: thu-tran-1 tro ve thu-rs bang UID, controller=true'

# 2. Thiếu thì tạo, thừa thì xóa
V="$(kubectl -n "$NS" get pods -l thu=rs -o name | grep 'pod/thu-rs-' | head -1)"
kubectl -n "$NS" delete "$V" --wait=true
for i in $(seq 1 60); do [ "$(dem)" -eq 3 ] && [ "$(jp rs/thu-rs '{.status.readyReplicas}')" = '3' ] && break; sleep 2; done
{ [ "$(dem)" -eq 3 ] && ! kubectl -n "$NS" get "$V" >/dev/null 2>&1; } \
  && echo 'PASS: Pod bi xoa duoc thay bang Pod khac ten'
kubectl apply -f "$W/thu-tran-3.yaml"
for i in $(seq 1 30); do kubectl -n "$NS" get pod thu-tran-3 >/dev/null 2>&1 || break; sleep 2; done
kubectl -n "$NS" describe rs thu-rs > "$EV/b1-describe-rs.txt"
{ ! kubectl -n "$NS" get pod thu-tran-3 >/dev/null 2>&1 && [ "$(dem)" -eq 3 ] \
  && grep -q SuccessfulCreate "$EV/b1-describe-rs.txt" && grep -q SuccessfulDelete "$EV/b1-describe-rs.txt"; } \
  && echo 'PASS: Pod tran thu ba bi xoa; co su kien SuccessfulCreate va SuccessfulDelete'

# 3. Hai ràng buộc API ép
kubectl apply --dry-run=server -f "$W/thu-rs-sai-selector.yaml" > "$EV/b1-sai-selector.txt" 2>&1 \
  && echo 'FAIL: API chap nhan template khong khop selector' \
  || { grep -qi 'selector' "$EV/b1-sai-selector.txt" && echo 'PASS: API tu choi template khong khop selector'; }
kubectl apply --dry-run=server -f "$W/thu-rs-sai-restart.yaml" > "$EV/b1-sai-restart.txt" 2>&1 \
  && echo 'FAIL: API chap nhan restartPolicy khac Always' \
  || { grep -qi 'restartPolicy' "$EV/b1-sai-restart.txt" && echo 'PASS: API tu choi restartPolicy khac Always'; }

# 4. Nhận nuôi nhưng không cập nhật
kubectl -n "$NS" delete rs thu-rs --cascade=orphan --wait=true
for i in $(seq 1 30); do
  REFS="$(kubectl -n "$NS" get pods -l thu=rs -o jsonpath='{range .items[*]}{.metadata.ownerReferences}{end}')"
  [ -z "$REFS" ] && break; sleep 2
done
{ [ "$(dem)" -eq 3 ] && [ -z "$REFS" ]; } && echo 'PASS: xoa orphan: con 3 Pod, khong Pod nao con owner'
kubectl apply -f "$W/thu-rs-v2.yaml"
for i in $(seq 1 30); do [ "$(jp rs/thu-rs-v2 '{.status.readyReplicas}')" = '3' ] && break; sleep 2; done
V2_UID="$(jp rs/thu-rs-v2 '{.metadata.uid}')"
OWN="$(kubectl -n "$NS" get pods -l thu=rs -o jsonpath='{range .items[*]}{.metadata.ownerReferences[0].uid}{"\n"}{end}' | sort -u)"
ENVS="$(kubectl -n "$NS" get pods -l thu=rs -o jsonpath='{range .items[*]}{.spec.containers[0].env}{end}')"
kubectl -n "$NS" get pods -l thu=rs -o wide > "$EV/b1-nhan-nuoi.txt"
{ [ "$(dem)" -eq 3 ] && [ "$OWN" = "$V2_UID" ] && [ -z "$ENVS" ] \
  && [ "$(kubectl -n "$NS" get pods -l thu=rs -o name | grep -c 'pod/thu-rs-v2-')" -eq 0 ] \
  && jp rs/thu-rs-v2 '{.spec.template.spec.containers[0].env}' | grep -q PHIEN_BAN; } \
  && echo 'PASS: thu-rs-v2 nhan nuoi 3 Pod cu, khong tao Pod moi, khong Pod nao co PHIEN_BAN'

# 5. ReplicationController: chỉ nhận diện
kubectl api-resources | grep -E '^(replicationcontrollers|replicasets) ' > "$EV/b1-api-resources.txt"
kubectl explain replicationcontroller.spec.selector > "$EV/b1-rc-selector.txt"
kubectl explain replicaset.spec.selector > "$EV/b1-rs-selector.txt"
grep -Eq '^replicationcontrollers +rc ' "$EV/b1-api-resources.txt" && echo 'PASS: API con ReplicationController, ten tat rc'
{ grep -q 'matchExpressions' "$EV/b1-rs-selector.txt" && ! grep -q 'matchExpressions' "$EV/b1-rc-selector.txt"; } \
  && echo 'PASS: selector cua rc la map phang, khong co matchExpressions'
[ "$(sl kubectl get rc -A --no-headers)" -eq 0 ] && echo 'PASS: cluster khong co rc nao'

# 6. Dọn
kubectl -n "$NS" delete rs thu-rs-v2 --ignore-not-found --wait=true
kubectl -n "$NS" delete pod -l thu=rs --ignore-not-found --wait=true
[ "$(sl kubectl -n "$NS" get rs,pods -l thu=rs --no-headers)" -eq 0 ] && echo 'PASS: da don object thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/4a/thu-tran.yaml — thêm "---" và một Pod y hệt tên thu-tran-2;
# thu-tran-3.yaml chỉ gồm một Pod như thế, tên thu-tran-3
apiVersion: v1
kind: Pod
metadata:
  name: thu-tran-1
  namespace: bookstore
  labels:
    thu: rs
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "trap 'exit 0' TERM; sleep 3600 & wait"]
```

```yaml
# ~/lab-work/4a/thu-rs.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: thu-rs
  namespace: bookstore
spec:
  replicas: 3
  selector:
    matchLabels:
      thu: rs
  template:
    metadata:
      labels:
        thu: rs
    spec:
      nodeSelector:
        kubernetes.io/hostname: lab-k8s-worker2
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh", "-c", "trap 'exit 0' TERM; sleep 3600 & wait"]
```

Ba biến thể, đều là bản sao của `thu-rs.yaml`:

- `thu-rs-v2.yaml`: tên `thu-rs-v2`; container thêm `env: [{name: PHIEN_BAN, value: "2"}]`.
- `thu-rs-sai-selector.yaml`: tên `thu-rs-sai-selector`; label của template đổi thành `thu: khac`.
- `thu-rs-sai-restart.yaml`: tên `thu-rs-sai-restart`; template thêm `restartPolicy: OnFailure`.

Ý chính cho ADR:

1. Pod trần `web` không có owner và khớp selector nên **bị thu nhận ngay**; ReplicaSet chỉ tạo thêm
   2 Pod, chạy template của **nó** — tức một tập Pod không đồng nhất (bài 64, *Thu nhận các Pod không
   theo template*). Lời khuyên của bài: đừng để Pod trần trùng label với selector của controller nào.
2. Selector trả lời "Pod nào tôi **được** thu nhận"; `ownerReferences` trả lời "Pod này **thuộc**
   về ai" — ReplicaSet theo dõi Pod qua liên kết đó. Tên có thể được dùng lại sau khi object bị xóa và
   tạo lại; UID thì không, nên Pod không bao giờ bị gán nhầm cho một ReplicaSet mới trùng tên.
3. ReplicaSet chỉ dùng template **khi tạo Pod mới**; nó không đưa Pod đang chạy sang template mới.
   Phát hành `web` cần đúng việc đó: thay dần Pod cũ bằng Pod theo template mới. Deployment làm việc
   đó bằng cách tạo một ReplicaSet mới cho template mới, scale nó lên và scale ReplicaSet cũ xuống
   (bài 64, mục *Chỉ xóa ReplicaSet*: "ReplicaSet không trực tiếp hỗ trợ rolling update").
4. Pod Pending trước, rồi `controller.kubernetes.io/pod-deletion-cost` thấp hơn, rồi Pod trên node
   có nhiều bản sao hơn, rồi Pod tạo gần đây hơn; ngang nhau hết thì ngẫu nhiên.
5. `rc` không hỗ trợ selector dựa trên tập hợp: selector là map phẳng, không có `matchExpressions`.
   Phần còn lại hành xử như ReplicaSet. Rolling update của nó là quy trình tay hai `rc` +1/−1. Hướng
   xử lý: di trú sang Deployment, không học quy trình cũ.

</details>

## B2. [BS-4A-02] `web` lên Deployment

**Loại:** Tính năng.

**Bối cảnh:** `web` là thành phần stateless: mọi bản sao hoán đổi được cho nhau (bài 62). Đây là
thành phần đầu tiên được giao cho controller. Từ ticket này tên Pod `web` không còn cố định — mọi
script và lệnh vận hành giả định tên `web` phải đổi.

**Yêu cầu:**

1. `web` chạy dưới Deployment `web` trong namespace `bookstore`, **3** bản sao, label theo hợp đồng
   ([mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label)). Selector của controller đúng cặp
   `app.kubernetes.io/part-of=bookstore` + `app.kubernetes.io/component=web`, không thêm gì.
2. Mỗi bản sao giữ nguyên mọi hành vi của Pod `web` ở tag `lab-3c`: trang có tên Pod và tên node của
   **chính nó**, sidecar `status-writer`, readiness `/healthz`, tắt êm, `securityContext`, user
   namespace, tài nguyên đã chốt ở 3c. Label `app.kubernetes.io/version` vẫn là `0.1.0`.
3. Manifest ở `k8s/app/web/deployment.yaml`, thay hẳn `k8s/app/web/pod.yaml`;
   `k8s/app/kustomization.yaml` trỏ tới file mới.
4. `scripts/deploy.sh` và `scripts/teardown.sh` thay bằng phiên bản của Lab 4a — chép nguyên văn từ
   khối bên dưới. `deploy.sh` mới chờ Deployment bằng `kubectl rollout status` và gỡ Pod trần cùng
   tên còn sót của sprint trước.
5. Mất một bản sao thì có bản sao khác thay vào mà không ai phải chạy lệnh gì; Deployment không vì
   thế mà sinh revision mới.
6. Một Pod trần mang đúng label của `web` **không** bị Deployment thu nhận.

**Ràng buộc:** không đổi tên container `nginx`, `status-writer`, không đổi cổng `http`. Không đặt
`nodeName`. `api` chưa đổi trong ticket này.

**Đọc lại:**

- Bài [63](../63-deployment-vi.md): [*Tạo một Deployment*](../63-deployment-vi.md#creating-a-deployment),
  [*Nhãn pod-template-hash*](../63-deployment-vi.md#pod-template-hash-label),
  [*Pod Template*](../63-deployment-vi.md#pod-template), [*Selector*](../63-deployment-vi.md#selector),
  [*Replicas*](../63-deployment-vi.md#replicas).
- Bài [345](../345-run-stateless-application-vi.md#creating-and-exploring-an-nginx-deployment) —
  tạo và khám phá một Deployment: `describe`, liệt kê Pod theo label.
- Bài [319](../319-declarative-config-vi.md), mục
  [*Cách xóa object*](../319-declarative-config-vi.md#cách-xóa-object-how-to-delete-objects) — vì sao
  `kubectl apply` không gỡ Pod trần `web` khi bạn bỏ `pod.yaml` khỏi repo; mục
  [*Định nghĩa selector của controller và label của PodTemplate*](../319-declarative-config-vi.md#định-nghĩa-selector-của-controller-và-label-của-podtemplate-defining-controller-selectors-and-podtemplate-labels).
- [Mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label) và [4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe)
  của file dự án.

**Bàn giao:** `k8s/app/web/deployment.yaml`; `k8s/app/kustomization.yaml`; hai script mới;
`docs/adr/NNNN-web-deployment.md`; `~/lab-evidence/4a/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` `teardown.sh` mới gỡ sạch; `deploy.sh` mới chạy hai lần liên tiếp đều thành công.
- `PASS:` `deployment.yaml` thay `pod.yaml` trong kustomization; không còn Pod trần `web`.
- `PASS:` Deployment `web` 3/3 Ready; selector đúng hai label của hợp đồng.
- `PASS:` mỗi Pod `web` → ReplicaSet (`controller: true`) → Deployment `web`, khớp UID từng bậc.
- `PASS:` tên ReplicaSet là `web-<hash>`; cùng hash nằm ở label của nó, selector của nó và label Pod.
- `PASS:` mỗi bản sao trả `/status.json` có đúng tên Pod và tên node của chính nó; trang chủ có tên
  Pod đó.
- `PASS:` xóa một Pod: Pod khác thay vào, vẫn ReplicaSet cũ, revision không đổi.
- `PASS:` Pod trần `thu-web-tran` mang đủ label của `web` không bị thu nhận; Deployment vẫn đúng 3
  Pod có `pod-template-hash`.

Hai script, chép nguyên văn:

```bash
cat > ~/bookstore/scripts/deploy.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai dự án Bookstore từ repo — phiên bản Lab 4a.
# - web và api chạy dưới Deployment; db vẫn là Pod trần (TD-01 còn phần db, trả ở Lab 4b).
# - Kustomize gom namespace, ConfigMap, Secret, PDB, Pod db, Deployment web (cơ chế từ Lab 3b).
# - api nằm ngoài Kustomize: file .tpl được thay __DB_HOST__ bằng IP Pod db (TD-03, trả ở Lab 5a).
# - Chờ Deployment bằng "kubectl rollout status": lệnh thoát khác 0 khi rollout vượt
#   progressDeadlineSeconds, nên một bản phát hành hỏng làm deploy.sh thất bại.
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore

# 1. Bí mật sinh tại chỗ khi thiếu; secrets/ không vào git (Lab 3b).
mkdir -p secrets
if [ ! -s secrets/db.env ]; then
  ( umask 077
    printf 'postgres-password=%s\nauthenticator-password=%s\n' \
      "$(openssl rand -hex 24)" "$(openssl rand -hex 24)" > secrets/db.env )
  echo 'deploy: da sinh secrets/db.env moi'
fi

# 2. Object cấp cluster, rồi phần Kustomize của namespace bookstore. secrets/db.env nằm ngoài thư
#    mục gốc k8s/app nên phải render với --load-restrictor (Lab 3b).
if compgen -G 'k8s/cluster/*.yaml' >/dev/null; then kubectl apply -f k8s/cluster/; fi
kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | kubectl apply -f -

# 3. Chờ db rồi render api với IP hiện tại của Pod db. Trong lúc chuyển đổi (B2 của Lab 4a), api
#    có thể còn là pod.yaml.tpl; từ B3 chỉ còn deployment.yaml.tpl.
kubectl -n "$NS" wait --for=condition=Ready pod/db --timeout=300s
DB_IP="$(kubectl -n "$NS" get pod db -o jsonpath='{.status.podIP}')"
test -n "$DB_IP"
API_TPL=k8s/app/api/deployment.yaml.tpl
if [ ! -f "$API_TPL" ]; then API_TPL=k8s/app/api/pod.yaml.tpl; fi
sed "s/__DB_HOST__/${DB_IP}/g" "$API_TPL" | kubectl apply -f -

# 4. kubectl apply không xóa object đã bỏ khỏi repo: Pod trần cùng tên của sprint trước nhường chỗ
#    cho Deployment. Rồi chờ — Deployment bằng rollout status, Pod trần còn lại bằng Ready.
for C in web api; do
  if kubectl -n "$NS" get deployment "$C" >/dev/null 2>&1; then
    kubectl -n "$NS" delete pod "$C" --ignore-not-found --wait=true
    kubectl -n "$NS" rollout status "deployment/$C" --timeout=300s
  else
    kubectl -n "$NS" wait --for=condition=Ready "pod/$C" --timeout=300s
  fi
done
kubectl -n "$NS" get deployment,replicaset,pod -o wide
EOF

cat > ~/bookstore/scripts/teardown.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ toàn bộ dự án Bookstore khỏi cluster — phiên bản Lab 4a.
# Giữ nguyên secrets/db.env trên máy: lần deploy sau dùng lại đúng bí mật cũ.
set -euo pipefail
SEL='app.kubernetes.io/part-of=bookstore'
KINDS='deployments,replicasets,pods,poddisruptionbudgets,configmaps,secrets'

# Xóa namespace kéo theo mọi object trong đó (xóa theo tầng background). Dòng thứ hai dọn object
# mang label dự án lỡ nằm ở namespace khác.
kubectl delete namespace -l "$SEL" --ignore-not-found --wait=true --timeout=300s
kubectl delete "$KINDS" --all-namespaces -l "$SEL" --ignore-not-found --wait=true --timeout=120s
kubectl delete priorityclass -l "$SEL" --ignore-not-found

# Mỗi truy vấn kiểm kê chạy riêng và phải thành công: API không trả lời hay Forbidden thì dừng —
# "không hỏi được" không bao giờ được đếm thành "không còn gì". dem <biến> <lệnh...> cộng số dòng
# output của lệnh vào biến; hàm chạy trong shell chính nên exit dừng cả script.
dem() {
  local v="$1" out; shift
  out="$("$@" 2>/dev/null)" || { echo "teardown: STOP - truy van loi, chua ket luan sach: $*"; exit 1; }
  printf -v "$v" '%d' "$(( ${!v:-0} + $(printf '%s\n' "$out" | awk 'NF' | wc -l) ))"
}
LEFT=0
dem LEFT kubectl get namespace,priorityclass -l "$SEL" --no-headers
dem LEFT kubectl get "$KINDS" --all-namespaces -l "$SEL" --no-headers
if [ "$LEFT" -eq 0 ]; then echo 'teardown: sach'; else echo "teardown: con $LEFT object"; exit 1; fi
EOF

chmod +x ~/bookstore/scripts/*.sh
```

`tests/smoke.sh` của Lab 3c gọi Pod theo tên `web` nên sẽ FAIL ở các dòng đó sau ticket này — đúng
dự kiến. Phiên bản mới nằm ở mục đóng sprint; trong lúc làm các ticket, dùng script nghiệm thu của
từng ticket.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-web-deployment.md`:

1. `thu-web-tran` khớp selector của Deployment `web`, vậy mà không bị thu nhận — trong khi ở B1,
   `thu-rs` thu nhận Pod trần ngay. Cơ chế nào tạo ra khác biệt đó?
2. Vì sao selector của `web` chỉ gồm `part-of` + `component`, không có `version` hay `name`? Điều gì
   xảy ra nếu sau này bạn muốn đổi selector?
3. So với kịch bản B4 của Lab 3a (Pod bị thay là mất): phần nào của `TD-01` đã trả, phần nào chưa?
4. `deploy.sh` mới tự xóa Pod trần `web` cũ. Vì sao `kubectl apply` không tự làm việc đó?
5. Liệt kê những chỗ trong repo và trong thói quen của bạn từng giả định tên Pod `web`. Thay bằng gì?

<details>
<summary>Gợi ý</summary>

1. Khung Deployment: `apiVersion: apps/v1`, `spec.replicas`, `spec.selector.matchLabels`,
   `spec.template`. Phần `template.spec` chính là spec Pod `web` hiện có — chép sang, không viết lại.
2. Template không có `metadata.name`: tên Pod do controller sinh.
3. Xem Kustomize sẽ gửi gì lên:
   `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | less`.
4. Trình tự an toàn: sửa `kustomization.yaml`, `git rm k8s/app/web/pod.yaml`, chép hai script, rồi
   `./scripts/teardown.sh && ./scripts/deploy.sh`.
5. Gọi một bản sao bất kỳ: `kubectl -n bookstore exec deploy/web -c status-writer -- wget -qO- http://127.0.0.1:8080/`.
   Gọi đúng một Pod: lấy tên theo label rồi `kubectl exec <tên> ...`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4a
WEB='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web'
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
web_pods() { kubectl -n "$NS" get pods -l "$WEB" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'; }

./scripts/teardown.sh > "$EV/b2-teardown.txt" 2>&1; grep -q 'teardown: sach' "$EV/b2-teardown.txt" \
  && echo 'PASS: teardown.sh go sach du an'
./scripts/deploy.sh > "$EV/b2-deploy-1.txt" 2>&1 && echo 'PASS: deploy.sh lan 1 thanh cong'
./scripts/deploy.sh > "$EV/b2-deploy-2.txt" 2>&1 && echo 'PASS: deploy.sh lan 2 thanh cong'

{ test ! -e k8s/app/web/pod.yaml && test -f k8s/app/web/deployment.yaml \
  && grep -q 'web/deployment.yaml' k8s/app/kustomization.yaml; } \
  && echo 'PASS: deployment.yaml thay pod.yaml trong kustomization'
kubectl -n "$NS" get pod web >/dev/null 2>&1 && echo 'FAIL: con Pod tran web' || echo 'PASS: khong con Pod tran web'

[ "$(jp deployment/web '{.spec.replicas}/{.status.readyReplicas}/{.status.updatedReplicas}')" = '3/3/3' ] \
  && echo 'PASS: Deployment web 3/3 Ready'
[ "$(kubectl -n "$NS" get deployment web -o go-template='{{range $k, $v := .spec.selector.matchLabels}}{{$k}}={{$v}} {{end}}')" \
  = 'app.kubernetes.io/component=web app.kubernetes.io/part-of=bookstore ' ] \
  && [ -z "$(jp deployment/web '{.spec.selector.matchExpressions}')" ] && echo 'PASS: selector dung hai label hop dong'

# Chuỗi sở hữu, đối chiếu bằng UID
DEP_UID="$(jp deployment/web '{.metadata.uid}')"; BAD=0; N=0; : > "$EV/b2-chuoi-so-huu.txt"
for P in $(web_pods); do
  N=$((N + 1)); RS="$(jp "pod/$P" '{.metadata.ownerReferences[0].name}')"
  [ "$(jp "pod/$P" '{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].controller}')" = 'ReplicaSet/true' ] || BAD=1
  [ "$(jp "pod/$P" '{.metadata.ownerReferences[0].uid}')" = "$(jp "replicaset/$RS" '{.metadata.uid}')" ] || BAD=1
  [ "$(jp "replicaset/$RS" '{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].uid}')" = "Deployment/$DEP_UID" ] || BAD=1
  echo "$P -> ReplicaSet/$RS -> Deployment/web" >> "$EV/b2-chuoi-so-huu.txt"
done
{ [ "$N" -eq 3 ] && [ "$BAD" -eq 0 ]; } && echo 'PASS: Pod -> ReplicaSet -> Deployment web, khop UID tung bac'

RS="$(kubectl -n "$NS" get rs -l "$WEB" -o jsonpath='{.items[0].metadata.name}')"
H="$(jp "replicaset/$RS" "{.metadata.labels['pod-template-hash']}")"
{ [ "$RS" = "web-$H" ] && [ "$(jp "replicaset/$RS" "{.spec.selector.matchLabels['pod-template-hash']}")" = "$H" ] \
  && [ "$(kubectl -n "$NS" get pods -l "$WEB,pod-template-hash=$H" --no-headers | wc -l)" -eq 3 ]; } \
  && echo "PASS: ReplicaSet web-$H: hash o ten, selector va label Pod"

# Mỗi bản sao biết mình là ai
BAD=0; : > "$EV/b2-status.txt"
for P in $(web_pods); do
  NODE="$(jp "pod/$P" '{.spec.nodeName}')"
  S="$(kubectl -n "$NS" exec "$P" -c status-writer -- wget -qO- http://127.0.0.1:8080/status.json)"
  echo "$P $S" >> "$EV/b2-status.txt"
  { echo "$S" | grep -Eq "\"pod\" *: *\"$P\"" && echo "$S" | grep -Eq "\"node\" *: *\"$NODE\"" \
    && kubectl -n "$NS" exec "$P" -c status-writer -- wget -qO- http://127.0.0.1:8080/ | grep -qF "$P"; } || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: moi ban sao bao dung ten Pod va node cua chinh no'

# Tự phục hồi
REV="$(jp deployment/web '{.metadata.annotations.deployment\.kubernetes\.io/revision}')"
V="$(kubectl -n "$NS" get pods -l "$WEB" --field-selector spec.nodeName=lab-k8s-worker2 -o jsonpath='{.items[0].metadata.name}')"
[ -n "$V" ] || V="$(web_pods | head -1)"
kubectl -n "$NS" delete pod "$V" --wait=true
for i in $(seq 1 60); do [ "$(kubectl -n "$NS" get pods -l "$WEB" --no-headers | wc -l)" -eq 3 ] && break; sleep 2; done
kubectl -n "$NS" wait --for=condition=Ready pod -l "$WEB" --timeout=180s >/dev/null
{ ! kubectl -n "$NS" get pod "$V" >/dev/null 2>&1 && [ "$(kubectl -n "$NS" get rs -l "$WEB" --no-headers | wc -l)" -eq 1 ] \
  && [ "$(jp deployment/web '{.metadata.annotations.deployment\.kubernetes\.io/revision}')" = "$REV" ]; } \
  && echo "PASS: $V bi xoa thi Pod khac thay vao, van ReplicaSet cu, revision $REV khong doi"

# Pod trần mang đúng label của web
VER="$(jp deployment/web '{.spec.template.metadata.labels.app\.kubernetes\.io/version}')"
kubectl -n "$NS" run thu-web-tran --image=busybox:1.37 --restart=Never \
  --labels="app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web,app.kubernetes.io/name=nginx,app.kubernetes.io/version=$VER" \
  --overrides='{"apiVersion":"v1","spec":{"nodeSelector":{"kubernetes.io/hostname":"lab-k8s-worker2"}}}' \
  --command -- sh -c "trap 'exit 0' TERM; sleep 3600 & wait"
sleep 15
{ [ -z "$(jp pod/thu-web-tran '{.metadata.ownerReferences}')" ] \
  && [ "$(kubectl -n "$NS" get pods -l "$WEB,pod-template-hash" --no-headers | wc -l)" -eq 3 ]; } \
  && echo 'PASS: Pod tran mang label cua web khong bi thu nhan'
kubectl -n "$NS" delete pod thu-web-tran --wait=true
kubectl -n "$NS" describe deployment web > "$EV/b2-describe.txt"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Khung Deployment là phần mới. Khối `template.spec` dưới đây là spec Pod `web` theo lời giải tham
khảo của các sprint trước, với tài nguyên lấy từ [mục 3.6 của file dự án](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch);
nếu Pod `web` ở tag `lab-3c` của bạn khác (con số tài nguyên đã chốt, tên volume, nội dung script),
**giữ phần của bạn** — chỉ khung bên ngoài là thay đổi.

```yaml
# k8s/app/web/deployment.yaml — thay k8s/app/web/pod.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
    app.kubernetes.io/name: nginx
spec:
  replicas: 3
  selector:                       # bất biến sau khi tạo — chỉ hai label của hợp đồng
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: web
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: web
        app.kubernetes.io/name: nginx
        app.kubernetes.io/version: "0.1.0"
    spec:
      # --- từ đây là spec Pod web ở tag lab-3c, giữ nguyên ---
      hostUsers: false
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
      volumes:
      - {name: html, emptyDir: {}}
      - name: podinfo
        downwardAPI: {items: [{path: labels, fieldRef: {fieldPath: metadata.labels}}]}
      - name: nginx-conf
        configMap: {name: web-config, items: [{key: default.conf, path: default.conf}]}
      - name: page-template
        configMap: {name: web-config, items: [{key: index.html.tpl, path: index.html.tpl}]}
      initContainers:
      - name: render-html
        image: busybox:1.37
        env:
        - {name: POD_NAME, valueFrom: {fieldRef: {fieldPath: metadata.name}}}
        - {name: NODE_NAME, valueFrom: {fieldRef: {fieldPath: spec.nodeName}}}
        command:
        - sh
        - -c
        - |
          set -e
          sed -e "s/__POD_NAME__/${POD_NAME}/g" -e "s/__NODE_NAME__/${NODE_NAME}/g" \
            /template/index.html.tpl > /html/index.html
          echo "render-html: da sinh /html/index.html tu trang mau"
        volumeMounts:
        - {name: page-template, mountPath: /template, readOnly: true}
        - {name: html, mountPath: /html}
      - name: status-writer
        image: busybox:1.37
        restartPolicy: Always
        env:
        - {name: POD_NAME, valueFrom: {fieldRef: {fieldPath: metadata.name}}}
        - {name: NODE_NAME, valueFrom: {fieldRef: {fieldPath: spec.nodeName}}}
        command:
        - sh
        - -c
        - |
          trap 'echo "status-writer: nhan TERM"; exit 0' TERM
          while true; do
            V="$(sed -n 's/^app\.kubernetes\.io\/version="\(.*\)"$/\1/p' /podinfo/labels)"
            printf '{"pod":"%s","node":"%s","version":"%s","updated":"%s"}\n' \
              "$POD_NAME" "$NODE_NAME" "$V" "$(date -Iseconds)" > /html/status.json.tmp
            mv /html/status.json.tmp /html/status.json
            sleep 5 & wait $!
          done
        resources:
          requests: {cpu: 10m, memory: 8Mi}
          limits: {cpu: 50m, memory: 16Mi}
        volumeMounts:
        - {name: html, mountPath: /html}
        - {name: podinfo, mountPath: /podinfo, readOnly: true}
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged:1.30.5-alpine
        ports:
        - {name: http, containerPort: 8080}
        readinessProbe:
          httpGet: {path: /healthz, port: http}
          periodSeconds: 5
        lifecycle:
          preStop:
            exec: {command: ["sh", "-c", "echo PRESTOP > /proc/1/fd/1; sleep 5"]}
        resources:
          requests: {cpu: 50m, memory: 32Mi}
          limits: {cpu: 200m, memory: 64Mi}
        securityContext: {allowPrivilegeEscalation: false}
        volumeMounts:
        - {name: html, mountPath: /usr/share/nginx/html}
        - {name: nginx-conf, mountPath: /etc/nginx/conf.d, readOnly: true}
```

Trong `k8s/app/kustomization.yaml`, mục `resources`: đổi dòng `web/pod.yaml` thành
`web/deployment.yaml`. Rồi:

```bash
cd ~/bookstore
git rm -q k8s/app/web/pod.yaml
./scripts/teardown.sh && ./scripts/deploy.sh
kubectl -n bookstore describe deployment web
kubectl -n bookstore get deployment,replicaset,pod -l app.kubernetes.io/component=web --show-labels
```

Ý chính cho ADR:

1. Deployment controller gắn `pod-template-hash` vào selector của ReplicaSet nó tạo (bài 63, *Nhãn
   pod-template-hash*). ReplicaSet chỉ thu nhận Pod khớp **selector của chính nó** — Pod trần thiếu
   label hash nên không khớp. `thu-rs` ở B1 có selector chỉ gồm label do bạn đặt, nên nuốt mọi Pod
   trần trùng label.
2. Selector của Deployment bất biến sau khi tạo (bài 63, *Cập nhật label selector*): đổi là phải xóa
   rồi tạo lại Deployment. `version` đổi ở mỗi lần phát hành nên không được nằm trong selector; bài
   319 khuyên dùng một nhãn riêng, ổn định, chỉ để chọn Pod — trong dự án là cặp `part-of` +
   `component`.
3. Đã trả phần `web`: Pod mất là ReplicaSet tạo Pod khác. Chưa trả: `api` (B3 của sprint này) và `db`
   (Lab 4b). Pod thay thế là Pod **mới** — tên, UID, IP mới, `emptyDir` mới; với `web` điều đó vô hại
   vì `web` không giữ trạng thái.
4. `kubectl apply` chỉ tạo và cập nhật object có trong file; xóa object là thao tác tường minh
   (bài 319, *Cách xóa object*). Bỏ `pod.yaml` khỏi repo không làm Pod trần biến mất trên cluster.
5. `tests/smoke.sh`, lệnh `kubectl exec web ...` trong ADR và runbook của 3a, 3b, mọi lệnh
   `kubectl logs web`. Thay bằng `deploy/web` khi cần một bản sao bất kỳ, hoặc lấy tên theo label
   `app.kubernetes.io/component=web` khi cần đi qua từng bản sao.

</details>

## B3. [BS-4A-03] `api` lên Deployment, PDB theo controller

**Loại:** Tính năng.

**Bối cảnh:** `api` cũng stateless — PostgREST giữ trạng thái trong DB, không trong Pod. Hai bản sao
cho phép mất một Pod `api` mà `/books` vẫn trả lời. Cùng lúc đó, PDB mà Lab 3c viết cho Pod trần chỉ
dùng được `minAvailable` số nguyên; giờ Pod đã có controller, ngân sách gián đoạn phải đi theo số bản
sao mong muốn của controller chứ không theo số Pod đang thấy.

**Yêu cầu:**

1. `api` chạy dưới Deployment `api`, **2** bản sao, label và selector theo hợp đồng. Manifest ở
   `k8s/app/api/deployment.yaml.tpl`, thay hẳn `pod.yaml.tpl`, vẫn **ngoài** kustomization;
   `deploy.sh` vẫn thay `__DB_HOST__` bằng IP Pod `db` lúc triển khai.
2. Mỗi bản sao giữ mọi hành vi của Pod `api` ở tag `lab-3c`: init chờ DB, ba probe, cấu hình từ
   `api-config`, mật khẩu từ Secret, PriorityClass, tài nguyên, `securityContext`.
3. Cả hai bản sao trả `/books`. Mất một bản sao thì bản còn lại vẫn trả `/books` trong lúc Deployment
   thay bản bị mất.
4. PDB `web` và `api` giữ tên, selector theo hợp đồng, và mỗi PDB cho phép **đúng một** gián đoạn tự
   nguyện tại một thời điểm — con số đó phải **tự đúng** khi số bản sao đổi.
5. Eviction API: evict một Pod `web` được chấp nhận; evict Pod `web` thứ hai ngay sau đó bị từ chối.

**Ràng buộc:** không đổi tên PDB. **Hợp đồng đã chốt của dự án:** PDB `web` và `api` đều khai
`maxUnavailable: 1` — không dùng `minAvailable`, nhất là `minAvailable` bằng số bản sao. Đây là quyết
định có chủ đích: Lab 11b đặt HPA `web` tối thiểu 2 bản sao, và Lab 12 phải drain được một worker khi
`web`, `api` đang chạy 2 bản sao (câu hỏi thiết kế 1). Không tạo Service — gọi `api` bằng IP Pod như
các sprint trước. `kubectl scale` chỉ dùng tạm trong script nghiệm thu; `deploy.sh` đưa mọi thứ về
repo.

**Đọc lại:**

- Bài [63](../63-deployment-vi.md): [*Selector*](../63-deployment-vi.md#selector),
  [*Replicas*](../63-deployment-vi.md#replicas).
- Bài [339](../339-configure-pdb-vi.md): [*Xác định ứng dụng cần bảo vệ*](../339-configure-pdb-vi.md#xác-định-ứng-dụng-cần-bảo-vệ-identify-an-application-to-protect),
  [*Chỉ định một PodDisruptionBudget*](../339-configure-pdb-vi.md#chỉ-định-một-poddisruptionbudget-specifying-a-poddisruptionbudget)
  — đoạn "số replica mong muốn là giá trị `scale` của controller",
  [*Workload tùy ý và selector tùy ý*](../339-configure-pdb-vi.md#arbitrary-controllers-and-selectors).
- Bài [346](../346-scale-deployment-vi.md), mục
  [*Scale down một Deployment*](../346-scale-deployment-vi.md#scale-down-một-deployment-scaling-down-a-deployment).
- [Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) — cách gọi Eviction API bằng `kubectl create --raw`.

**Bàn giao:** `k8s/app/api/deployment.yaml.tpl`; hai file PDB đã cập nhật;
`docs/adr/NNNN-api-deployment.md`; `~/lab-evidence/4a/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` hệ thống dựng lại từ repo; `deployment.yaml.tpl` thay `pod.yaml.tpl`, vẫn ngoài
  kustomization; không còn Pod trần `api`.
- `PASS:` Deployment `api` 2/2 Ready, selector theo hợp đồng, mang IP hiện tại của Pod `db`.
- `PASS:` mỗi Pod `api` → ReplicaSet → Deployment `api`; cả hai bản sao trả `/books` ≥ 5 cuốn.
- `PASS:` xóa một Pod `api`: bản sao còn lại vẫn trả `/books`; Deployment bù lại 2/2.
- `PASS:` hai PDB khai `maxUnavailable` bằng 1, không khai `minAvailable`.
- `PASS:` PDB `web` đòi 2 Pod khỏe trên 3, PDB `api` đòi 1 trên 2, mỗi PDB cho phép 1 gián đoạn.
- `PASS:` scale `web` lên 4 thì PDB `web` đòi 3 khỏe; `deploy.sh` đưa `web` về 3.
- `PASS:` eviction thứ nhất được chấp nhận, eviction thứ hai bị PDB từ chối; Pod thứ hai vẫn chạy;
  ngân sách hồi lại 1 khi Pod thay thế Ready.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-api-deployment.md`:

1. Hợp đồng chốt `maxUnavailable: 1` cho cả hai PDB. Vì sao `minAvailable: <số bản sao>` — 3 cho
   `web`, 2 cho `api` — chặn drain? Dẫn đúng câu của bài 339, rồi kể chuyện gì xảy ra ở Lab 12 khi
   drain một worker đang chạy một bản sao `web` và một bản sao `api`.
2. Ở 3c, Pod trần buộc bạn dùng `minAvailable` số nguyên. Vì sao giờ `maxUnavailable` tốt hơn một
   `minAvailable` cố định, khi HPA ở Lab 11b cho `web` chạy từ 2 tới 5 bản sao?
3. Mỗi lần `teardown` + `deploy`, `db` đổi IP. Điều đó làm gì với Deployment `api` — với revision, với
   lịch sử phát hành? Nợ nào gỡ chuyện này, ở lab nào?
4. Hai bản sao `api` có giúp gì khi `db` chết không? Trả lời bằng probe readiness của `api`.

<details>
<summary>Gợi ý</summary>

1. Khung Deployment giống `web`; phần `template.spec` là spec Pod `api` hiện có. Hai chỗ
   `__DB_HOST__` (init `wait-for-db` và biến `DB_HOST`) giữ nguyên.
2. Trình tự: `git rm k8s/app/api/pod.yaml.tpl`, viết `deployment.yaml.tpl`, sửa hai PDB, rồi
   `./scripts/teardown.sh && ./scripts/deploy.sh`.
3. Bài 339: `maxUnavailable` chỉ dùng được khi mọi Pod được chọn có **cùng một** controller, và nó tự
   phản ứng khi số bản sao của controller đổi. Chỉ khai một trong hai trường — bỏ hẳn `minAvailable`
   của Lab 3c khỏi file.
4. Trạng thái PDB: `kubectl -n bookstore get pdb web -o jsonpath='{.status}'` — các trường
   `expectedPods`, `desiredHealthy`, `disruptionsAllowed`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
NS=bookstore; EV=~/lab-evidence/4a; W=~/lab-work/4a
WEB='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web'
API='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=api'
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
books() { kubectl -n "$NS" exec deploy/web -c status-writer -- wget -qO- "http://$1:3000/books" 2>/dev/null | grep -o '"title"' | wc -l; }
cho_dem() { for i in $(seq 1 90); do [ "$(sl kubectl -n "$NS" get pods -l "$1" --no-headers)" -eq "$2" ] && return 0; sleep 2; done; return 1; }
pdb_cho() {   # chờ PDB $1 tính xong với expectedPods = currentHealthy = $2
  for i in $(seq 1 45); do
    [ "$(jp "pdb/$1" '{.status.observedGeneration}')" = "$(jp "pdb/$1" '{.metadata.generation}')" ] \
      && [ "$(jp "pdb/$1" '{.status.expectedPods}/{.status.currentHealthy}')" = "$2/$2" ] && return 0
    sleep 2
  done; return 1
}

./scripts/teardown.sh > "$EV/b3-teardown.txt" 2>&1 && ./scripts/deploy.sh > "$EV/b3-deploy.txt" 2>&1 \
  && echo 'PASS: he thong dung lai tu repo'
{ test ! -e k8s/app/api/pod.yaml.tpl && grep -q '__DB_HOST__' k8s/app/api/deployment.yaml.tpl \
  && ! grep -q 'deployment.yaml.tpl' k8s/app/kustomization.yaml; } \
  && echo 'PASS: deployment.yaml.tpl thay pod.yaml.tpl, van ngoai kustomization'
kubectl -n "$NS" get pod api >/dev/null 2>&1 && echo 'FAIL: con Pod tran api' || echo 'PASS: khong con Pod tran api'

[ "$(jp deployment/api '{.spec.replicas}/{.status.readyReplicas}')" = '2/2' ] && echo 'PASS: Deployment api 2/2 Ready'
[ "$(kubectl -n "$NS" get deployment api -o go-template='{{range $k, $v := .spec.selector.matchLabels}}{{$k}}={{$v}} {{end}}')" \
  = 'app.kubernetes.io/component=api app.kubernetes.io/part-of=bookstore ' ] && echo 'PASS: selector api theo hop dong'
DB_IP="$(jp pod/db '{.status.podIP}')"
kubectl -n "$NS" get deployment api -o yaml | grep -qF "$DB_IP" && echo "PASS: Deployment api mang IP hien tai cua db ($DB_IP)"

DEP_UID="$(jp deployment/api '{.metadata.uid}')"; BAD=0
for P in $(kubectl -n "$NS" get pods -l "$API" -o jsonpath='{.items[*].metadata.name}'); do
  RS="$(jp "pod/$P" '{.metadata.ownerReferences[0].name}')"
  [ "$(jp "replicaset/$RS" '{.metadata.ownerReferences[0].uid}')" = "$DEP_UID" ] || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: Pod api -> ReplicaSet -> Deployment api'
BAD=0; N=0
for IP in $(kubectl -n "$NS" get pods -l "$API" -o jsonpath='{.items[*].status.podIP}'); do
  N=$((N + 1)); T="$(books "$IP")"; echo "$IP $T" >> "$EV/b3-books.txt"; [ "$T" -ge 5 ] || BAD=1
done
{ [ "$N" -eq 2 ] && [ "$BAD" -eq 0 ]; } && echo 'PASS: ca hai ban sao api tra /books'

# Mất một bản sao api
V="$(kubectl -n "$NS" get pods -l "$API" --field-selector spec.nodeName=lab-k8s-worker2 -o jsonpath='{.items[0].metadata.name}')"
[ -n "$V" ] || V="$(kubectl -n "$NS" get pods -l "$API" -o jsonpath='{.items[0].metadata.name}')"
CON="$(kubectl -n "$NS" get pods -l "$API" -o jsonpath='{range .items[*]}{.metadata.name} {.status.podIP}{"\n"}{end}' \
  | awk -v v="$V" '$1 != v {print $2; exit}')"
kubectl -n "$NS" delete pod "$V" --wait=false
[ "$(books "$CON")" -ge 5 ] && echo 'PASS: trong luc thay Pod, ban sao con lai van tra /books'
for i in $(seq 1 60); do ! kubectl -n "$NS" get pod "$V" >/dev/null 2>&1 && break; sleep 2; done
cho_dem "$API" 2 && kubectl -n "$NS" wait --for=condition=Ready pod -l "$API" --timeout=180s >/dev/null \
  && echo 'PASS: Deployment api bu lai 2/2'

# PDB theo controller — hợp đồng: maxUnavailable 1, không minAvailable
{ [ "$(jp pdb/web '{.spec.maxUnavailable}/{.spec.minAvailable}')" = '1/' ] \
  && [ "$(jp pdb/api '{.spec.maxUnavailable}/{.spec.minAvailable}')" = '1/' ]; } \
  && echo 'PASS: PDB web va api khai maxUnavailable 1, khong khai minAvailable'
pdb_cho web 3; pdb_cho api 2
kubectl -n "$NS" get pdb -o wide > "$EV/b3-pdb.txt"
{ [ "$(jp pdb/web '{.status.desiredHealthy}/{.status.disruptionsAllowed}')" = '2/1' ] \
  && [ "$(jp pdb/api '{.status.desiredHealthy}/{.status.disruptionsAllowed}')" = '1/1' ]; } \
  && echo 'PASS: web doi 2 tren 3, api doi 1 tren 2; moi PDB cho phep 1 gian doan'
kubectl -n "$NS" scale deployment/web --replicas=4
kubectl -n "$NS" rollout status deployment/web --timeout=180s >/dev/null
pdb_cho web 4 && [ "$(jp pdb/web '{.status.desiredHealthy}')" = '3' ] \
  && echo 'PASS: web 4 ban sao thi PDB doi 3 kho — ngan sach theo so ban sao mong muon'
./scripts/deploy.sh > "$EV/b3-deploy-2.txt" 2>&1
cho_dem "$WEB" 3 && pdb_cho web 3 && [ "$(jp deployment/web '{.spec.replicas}')" = '3' ] \
  && echo 'PASS: deploy.sh dua web ve 3 ban sao theo repo'

# Eviction API: lần một được, lần hai bị chặn
P1="$(kubectl -n "$NS" get pods -l "$WEB" --field-selector spec.nodeName=lab-k8s-worker2 -o jsonpath='{.items[0].metadata.name}')"
[ -n "$P1" ] || P1="$(kubectl -n "$NS" get pods -l "$WEB" -o jsonpath='{.items[0].metadata.name}')"
P2="$(kubectl -n "$NS" get pods -l "$WEB" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep -vx "$P1" | head -1)"
for P in "$P1" "$P2"; do
  printf '{"apiVersion":"policy/v1","kind":"Eviction","metadata":{"name":"%s","namespace":"bookstore"}}\n' "$P" > "$W/evict-$P.json"
done
kubectl create --raw "/api/v1/namespaces/$NS/pods/$P1/eviction" -f "$W/evict-$P1.json" > "$EV/b3-evict-1.txt" 2>&1 \
  && echo 'PASS: eviction thu nhat duoc chap nhan'
kubectl create --raw "/api/v1/namespaces/$NS/pods/$P2/eviction" -f "$W/evict-$P2.json" > "$EV/b3-evict-2.txt" 2>&1 \
  && echo 'FAIL: eviction thu hai khong bi chan' \
  || { grep -qi 'disruption budget' "$EV/b3-evict-2.txt" && echo 'PASS: eviction thu hai bi PDB tu choi'; }
[ -z "$(jp "pod/$P2" '{.metadata.deletionTimestamp}')" ] && kubectl -n "$NS" get pod "$P2" >/dev/null 2>&1 \
  && echo "PASS: $P2 van chay"
for i in $(seq 1 60); do ! kubectl -n "$NS" get pod "$P1" >/dev/null 2>&1 && break; sleep 2; done
cho_dem "$WEB" 3 && kubectl -n "$NS" wait --for=condition=Ready pod -l "$WEB" --timeout=180s >/dev/null
pdb_cho web 3 && [ "$(jp pdb/web '{.status.disruptionsAllowed}')" = '1' ] && echo 'PASS: Pod thay the Ready, ngan sach hoi lai 1'
rm -f "$W"/evict-*.json
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/api/deployment.yaml.tpl — thay k8s/app/api/pod.yaml.tpl. Nằm NGOÀI kustomization:
# scripts/deploy.sh thay __DB_HOST__ bằng IP Pod db rồi apply (TD-03, trả ở Lab 5a).
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: api
    app.kubernetes.io/name: postgrest
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: api
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: api
        app.kubernetes.io/name: postgrest
    spec:
      # --- từ đây là spec Pod api ở tag lab-3c, giữ nguyên ---
      priorityClassName: bookstore-critical
      terminationGracePeriodSeconds: 30
      initContainers:
      - name: wait-for-db
        image: postgres:17.11
        command:
        - sh
        - -c
        - |
          until pg_isready -h "$DB_HOST" -p 5432 -t 2; do
            echo "wait-for-db: $DB_HOST:5432 chua nhan ket noi"
            sleep 2
          done
        env:
        - {name: DB_HOST, value: "__DB_HOST__"}
      containers:
      - name: postgrest
        image: postgrest/postgrest:v14.18
        envFrom:
        - configMapRef:
            name: api-config
        env:
        - name: DB_HOST
          value: "__DB_HOST__"             # deploy.sh thay bằng IP Pod db (TD-03)
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: authenticator-password
        - name: PGRST_DB_URI               # đứng SAU hai biến nó tham chiếu
          value: "postgres://authenticator:$(DB_PASSWORD)@$(DB_HOST):5432/bookstore"
        ports:
        - {name: http, containerPort: 3000}
        - {name: admin, containerPort: 3001}
        startupProbe:
          httpGet: {path: /live, port: admin}
          periodSeconds: 2
          failureThreshold: 30
        readinessProbe:
          httpGet: {path: /ready, port: admin}
          periodSeconds: 5
          failureThreshold: 2
        livenessProbe:
          httpGet: {path: /live, port: admin}
          periodSeconds: 10
          failureThreshold: 3
        resources:
          requests: {cpu: 50m, memory: 64Mi}
          limits: {cpu: 500m, memory: 256Mi}
        securityContext: {runAsNonRoot: true, allowPrivilegeEscalation: false}
```

Hai PDB — đường dẫn file theo Lab 3c của bạn (ở đây giả định `k8s/app/web/pdb.yaml` và
`k8s/app/api/pdb.yaml`). Chỉ `spec` đổi: bỏ `minAvailable`, dùng `maxUnavailable: 1` theo hợp đồng đã
chốt.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web                        # PDB api giống hệt, thay web bằng api
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: web
```

```bash
cd ~/bookstore
git rm -q k8s/app/api/pod.yaml.tpl
./scripts/teardown.sh && ./scripts/deploy.sh
kubectl -n bookstore get pdb
```

`kubectl apply` gỡ `minAvailable` khỏi PDB đang chạy vì trường đó có trong
`last-applied-configuration` mà không còn trong file — cơ chế học kỹ ở B8.

Ý chính cho ADR:

1. Bài 339, mục *Chỉ định một PodDisruptionBudget*: đặt `minAvailable` bằng số bản sao (hay
   `maxUnavailable` bằng 0) là không cho phép **bất kỳ** eviction tự nguyện nào, và khi đó không
   drain thành công được node đang chạy một trong các Pod đó — thao tác drain không bao giờ hoàn
   thành. Ở Lab 12, drain một worker đang chạy một bản sao `web` và một bản sao `api` sẽ treo mãi ở
   hai Pod đó. Với `maxUnavailable: 1`, drain evict được từng Pod một; Deployment tạo Pod thay thế ở
   worker kia, ngân sách hồi lại khi Pod mới Ready.
2. Bài 339: với Pod có controller, ngân sách tính trên giá trị `scale` của controller, và
   `maxUnavailable` tự phản ứng khi số bản sao đổi — đúng một gián đoạn dù HPA để `web` ở 2 hay 5 bản
   sao. Một `minAvailable` cố định thì đổi nghĩa theo số bản sao: `minAvailable: 2` chặn mọi eviction
   khi HPA hạ `web` xuống 2, còn `minAvailable: 1` cho phép mất 4 trên 5 cùng lúc.
3. IP nằm trong Pod template, nên template đổi, Deployment `api` sinh một revision mới dù không ai
   phát hành gì — lịch sử `api` lẫn những revision "chỉ đổi IP". `TD-03`, trả ở Lab 5a khi `api` gọi
   `db` bằng tên DNS.
4. Không. Readiness của `api` trỏ `/ready`, phụ thuộc DB (Lab 3a): `db` chết thì **cả hai** bản sao
   cùng NotReady. Bản sao chỉ che lỗi của chính Pod `api`, không che lỗi của tầng dưới.

</details>

## B4. [BS-4A-04] Phát hành `web` không gián đoạn và có lý do

**Loại:** Tính năng.

**Bối cảnh:** yêu cầu `R4` của dự án: phát hành không gián đoạn, rollback được trong vài phút
([mục 2.2](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới)). Từ B2, `web` và `api` chạy với chiến
lược phát hành **mặc định** — không ai trong đội nói được nó sẽ làm gì khi số bản sao đổi, khi nào
một lần phát hành hỏng bị phát hiện, và mỗi lần phát hành là vì sao. Ticket này biến chiến lược thành
một quyết định có chủ đích trong repo, rồi phát hành thử bản `0.1.1` để đo.

**Yêu cầu:**

1. Trong suốt một lần phát hành `web`, số bản sao sẵn sàng không bao giờ dưới 3 và tổng số bản sao
   mong muốn không vượt 4. Pod mới chỉ được tính là sẵn sàng sau khi Ready liên tục ít nhất 5 giây.
2. `api` cũng không giảm số bản sao sẵn sàng khi phát hành.
3. Chiến lược phát hành của `web` và `api` nằm **tường minh** trong repo, không dựa vào giá trị mặc
   định của API server.
4. Một lần phát hành không tiến triển phải bị báo trong vòng 3 phút, và khi đó `deploy.sh` thoát
   khác 0.
5. Lịch sử đủ để lùi ít nhất 3 bản phát hành.
6. Mỗi bản phát hành mang một dòng lý do, đọc được trong lịch sử rollout. Bản đầu tiên là `web`
   `0.1.1` — chỉ đổi phiên bản, để kiểm chứng đường phát hành; sau đó `/status.json` của mọi bản sao
   báo `0.1.1`.
7. Phát hành đi qua repo: commit trước, `./scripts/deploy.sh` sau.

**Ràng buộc:** không dùng `kubectl set image`, `kubectl annotate`, `kubectl edit` trên `web` và
`api`; không dùng cờ `--record`. Chưa có Service, nên "không gián đoạn" đo ở mức controller
(`availableReplicas`) — đo ở mức request là phần hoãn sang Lab 5a B3.

**Đọc lại:**

- Bài [63](../63-deployment-vi.md): [*Strategy*](../63-deployment-vi.md#strategy),
  [*Rolling Update*](../63-deployment-vi.md#rolling-update-deployment),
  [*Max Unavailable*](../63-deployment-vi.md#max-unavailable), [*Max Surge*](../63-deployment-vi.md#max-surge),
  [*Min Ready Seconds*](../63-deployment-vi.md#min-ready-seconds),
  [*Progress Deadline Seconds*](../63-deployment-vi.md#progress-deadline-seconds),
  [*Giới hạn lịch sử revision*](../63-deployment-vi.md#revision-history-limit),
  [*Kiểm tra lịch sử rollout*](../63-deployment-vi.md#checking-rollout-history-of-a-deployment), và
  ghi chú cuối mục [*Cập nhật một Deployment*](../63-deployment-vi.md#updating-a-deployment) về Pod
  đang kết thúc.
- Bài [348](../348-update-deployment-rolling-vi.md):
  [*Thực hiện một rolling update*](../348-update-deployment-rolling-vi.md#thực-hiện-một-rolling-update-performing-a-rolling-update),
  [*Theo dõi tiến trình rollout*](../348-update-deployment-rolling-vi.md#theo-dõi-tiến-trình-rollout-monitoring-rollout-progress),
  [*Cấu hình chiến lược rolling update*](../348-update-deployment-rolling-vi.md#cấu-hình-chiến-lược-rolling-update-configuring-rolling-update-strategy).
- Bài [61](../61-management-vi.md#quản-lý-rollout-managing-rollouts) — `rollout status` với
  `--timeout` và `--watch=false`.
- Bài [319](../319-declarative-config-vi.md#giá-trị-mặc-định-của-field-default-field-values) — vì sao
  nên viết tường minh chiến lược rollout.

**Bàn giao:** `k8s/app/web/deployment.yaml`, `k8s/app/api/deployment.yaml.tpl` đã cập nhật;
`docs/adr/NNNN-chien-luoc-phat-hanh.md`; `~/lab-evidence/4a/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` thay đổi đã commit trước khi phát hành; `deploy.sh` phát hành thành công.
- `PASS:` lấy đủ mẫu trong lúc rollout; `availableReplicas` thấp nhất ≥ 3; tổng `.spec.replicas` các
  ReplicaSet của `web` cao nhất ≤ 4.
- `PASS:` revision của `web` tăng đúng 1; ReplicaSet cũ ở lại với 0 bản sao.
- `PASS:` dòng cuối lịch sử rollout của `web` mang lý do của `0.1.1`; mọi bản sao báo `version` là
  `0.1.1`.
- `PASS:` chiến lược của `web` nằm trong cấu hình đã apply; `minReadySeconds` ≥ 5;
  `progressDeadlineSeconds` ≤ 180 và lớn hơn `minReadySeconds`; `revisionHistoryLimit` ≥ 3.
- `PASS:` `api` có chiến lược tường minh với `maxUnavailable` bằng 0, có lý do phát hành và
  `progressDeadlineSeconds` ≤ 180.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-chien-luoc-phat-hanh.md`:

1. Với 3 bản sao, giá trị mặc định 25%/25% cho đúng `maxUnavailable` 0 và `maxSurge` 1. Vì sao vẫn
   viết tường minh? Tính lại giá trị mặc định khi `web` có 2, 4 và 5 bản sao — khoảng mà HPA sẽ dùng ở
   Lab 11b.
2. `maxUnavailable: 0` dựa vào điều gì để biết Pod mới "đã sẵn sàng"? `minReadySeconds` thêm được
   gì so với readiness probe?
3. Cột số Pod thô trong file mẫu có lúc vượt 4. Có phải chiến lược bị vi phạm không?
4. `CHANGE-CAUSE` lấy từ đâu? Vì sao dự án ghi nó trong manifest thay vì dùng `kubectl annotate` như
   ví dụ của bài 63?
5. `progressDeadlineSeconds` của `web` và `--timeout` của `rollout status` trong `deploy.sh`: cái nào
   phải nhỏ hơn, vì sao?

<details>
<summary>Gợi ý</summary>

1. Bốn trường cấp `spec` của Deployment — `strategy` (kèm `rollingUpdate`), `minReadySeconds`,
   `progressDeadlineSeconds`, `revisionHistoryLimit` — và một annotation ở `metadata.annotations`.
2. Ngưỡng của một lần phát hành: `replicas − maxUnavailable` và `replicas + maxSurge`.
3. Muốn có rollout thì Pod template phải đổi: tăng label `app.kubernetes.io/version` trong template.
   Đổi `strategy` hay `replicas` một mình không sinh rollout.
4. Xem trước khi triển khai:
   `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | kubectl diff -f -`.
5. Theo dõi: `kubectl -n bookstore rollout status deployment/web`; chỉ xem trạng thái, không chờ:
   thêm `--watch=false`. Lịch sử: `kubectl -n bookstore rollout history deployment/web`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi sửa và commit repo, TRƯỚC khi triển khai; script này chính là lần phát hành</summary>

Khối đầu tạo công cụ lấy mẫu `~/lab-work/4a/lay-mau.sh`, B7 dùng lại. Hàm chỉ bắt đầu ghi khi
controller đã nhận thay đổi (`generation` tăng), và chỉ dừng khi `observedGeneration` đã bắt kịp —
nếu không, nó sẽ dừng ngay ở mẫu đầu với trạng thái của lần phát hành **trước**.

```bash
cat > ~/lab-work/4a/lay-mau.sh <<'EOF'
# lay_mau <deployment> <selector> <file> <so-ban-sao> <generation-truoc>
# Ghi mỗi vòng một dòng "availableReplicas tong-.spec.replicas-cac-ReplicaSet so-Pod-tho"
# từ lúc controller nhận thay đổi tới khi rollout xong; in ra số mẫu.
lay_mau() {
  local d="$1" sel="$2" out="$3" want="$4" gen0="$5" GEN OBS UPD AV SUM PODS i
  : > "$out"
  for i in $(seq 1 120); do
    GEN="$(kubectl -n bookstore get deploy "$d" -o jsonpath='{.metadata.generation}')"
    [ "${GEN:-0}" -gt "$gen0" ] && break
    sleep 1
  done
  for i in $(seq 1 300); do
    IFS=/ read -r GEN OBS UPD AV <<< "$(kubectl -n bookstore get deploy "$d" \
      -o jsonpath='{.metadata.generation}/{.status.observedGeneration}/{.status.updatedReplicas}/{.status.availableReplicas}')"
    SUM="$(kubectl -n bookstore get rs -l "$sel" -o jsonpath='{range .items[*]}{.spec.replicas}{"\n"}{end}' \
      | awk '{s += $1} END {print s + 0}')"
    PODS="$(kubectl -n bookstore get pods -l "$sel" --no-headers 2>/dev/null | wc -l)"
    echo "${AV:-0} ${SUM:-0} ${PODS:-0}" >> "$out"
    if [ "$i" -ge 3 ] && [ "$OBS" = "$GEN" ] && [ "${UPD:-0}" = "$want" ] \
       && [ "${AV:-0}" = "$want" ] && [ "$SUM" = "$want" ]; then
      break
    fi
    sleep 1
  done
  wc -l < "$out"
}
EOF
```

```bash
NS=bookstore; EV=~/lab-evidence/4a
WEB='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web'
cd ~/bookstore
. ~/lab-work/4a/lay-mau.sh
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

test -z "$(git status --porcelain)" && echo 'PASS: thay doi da commit truoc khi phat hanh'
REV0="$(jp deployment/web '{.metadata.annotations.deployment\.kubernetes\.io/revision}')"
GEN0="$(jp deployment/web '{.metadata.generation}')"
lay_mau web "$WEB" "$EV/b4-mau.txt" 3 "$GEN0" > "$EV/b4-so-mau.txt" & P=$!
./scripts/deploy.sh > "$EV/b4-deploy.txt" 2>&1 && echo 'PASS: deploy.sh phat hanh thanh cong'
wait "$P"

N="$(cat "$EV/b4-so-mau.txt")"
MIN_AV="$(awk '{print $1}' "$EV/b4-mau.txt" | sort -n | head -1)"
MAX_SUM="$(awk '{print $2}' "$EV/b4-mau.txt" | sort -n | tail -1)"
MAX_PODS="$(awk '{print $3}' "$EV/b4-mau.txt" | sort -n | tail -1)"
echo "so mau=$N min available=$MIN_AV max tong replica RS=$MAX_SUM max Pod tho=$MAX_PODS" | tee "$EV/b4-ket-luan.txt"
[ "$N" -ge 5 ] && echo 'PASS: lay du mau trong luc rollout'
[ "$MIN_AV" -ge 3 ] && echo 'PASS: khong luc nao duoi 3 ban sao san sang'
[ "$MAX_SUM" -le 4 ] && echo 'PASS: tong ban sao mong muon khong vuot 4'

[ "$(jp deployment/web '{.metadata.annotations.deployment\.kubernetes\.io/revision}')" = "$((REV0 + 1))" ] \
  && echo 'PASS: revision tang dung 1'
[ "$(kubectl -n "$NS" get rs -l "$WEB" -o jsonpath='{range .items[*]}{.spec.replicas}{"\n"}{end}' | grep -c '^0$')" -ge 1 ] \
  && echo 'PASS: ReplicaSet cu o lai voi 0 ban sao'
kubectl -n "$NS" rollout history deployment/web > "$EV/b4-history.txt"
awk 'NF' "$EV/b4-history.txt" | tail -1 | grep -q '0\.1\.1' && echo 'PASS: dong cuoi lich su mang ly do cua 0.1.1'
for i in $(seq 1 60); do [ "$(kubectl -n "$NS" get pods -l "$WEB" --no-headers | wc -l)" -eq 3 ] && break; sleep 2; done
BAD=0
for P in $(kubectl -n "$NS" get pods -l "$WEB" -o jsonpath='{.items[*].metadata.name}'); do
  kubectl -n "$NS" exec "$P" -c status-writer -- wget -qO- http://127.0.0.1:8080/status.json \
    | grep -Eq '"version" *: *"0\.1\.1"' || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: moi ban sao bao version 0.1.1'

jp deployment/web '{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' | grep -q '"strategy"' \
  && echo 'PASS: chien luoc cua web nam trong cau hinh da apply'
MRS="$(jp deployment/web '{.spec.minReadySeconds}')"; PDS="$(jp deployment/web '{.spec.progressDeadlineSeconds}')"
RHL="$(jp deployment/web '{.spec.revisionHistoryLimit}')"
{ [ "${MRS:-0}" -ge 5 ] && [ "$PDS" -le 180 ] && [ "$PDS" -gt "$MRS" ] && [ "$RHL" -ge 3 ]; } \
  && echo "PASS: web minReadySeconds=$MRS progressDeadlineSeconds=$PDS revisionHistoryLimit=$RHL"
{ jp deployment/api '{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}' | grep -q '"strategy"' \
  && echo " $(jp deployment/api '{.spec.strategy.rollingUpdate.maxUnavailable}') " | grep -Eq ' 0%? ' \
  && [ -n "$(jp deployment/api '{.metadata.annotations.kubernetes\.io/change-cause}')" ] \
  && [ "$(jp deployment/api '{.spec.progressDeadlineSeconds}')" -le 180 ]; } \
  && echo 'PASS: api co chien luoc tuong minh, maxUnavailable 0, co ly do phat hanh'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Trong `k8s/app/web/deployment.yaml` — thêm annotation, bốn trường cấp `spec`, và tăng version trong
template:

```yaml
metadata:
  name: web
  # namespace, labels: giữ nguyên
  annotations:
    kubernetes.io/change-cause: "0.1.1: khai bao chien luoc phat hanh tuong minh"
spec:
  replicas: 3
  revisionHistoryLimit: 5         # đủ lùi 5 bản; mỗi ReplicaSet cũ là một object trong etcd
  minReadySeconds: 5              # Ready liên tục 5 giây mới tính là sẵn sàng
  progressDeadlineSeconds: 180    # nhỏ hơn --timeout=300s của rollout status trong deploy.sh
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  # selector: giữ nguyên
  template:
    metadata:
      labels:
        # part-of, component, name: giữ nguyên
        app.kubernetes.io/version: "0.1.1"
```

Trong `k8s/app/api/deployment.yaml.tpl` — cùng khối, không có label version:

```yaml
metadata:
  name: api
  annotations:
    kubernetes.io/change-cause: "api: khai bao chien luoc phat hanh tuong minh"
spec:
  replicas: 2
  revisionHistoryLimit: 5
  minReadySeconds: 5
  progressDeadlineSeconds: 180
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

```bash
cd ~/bookstore
git add -A && git commit -qm 'web 0.1.1: chien luoc phat hanh tuong minh'
# rồi chạy script nghiệm thu: nó gọi ./scripts/deploy.sh và lấy mẫu song song
```

Ý chính cho ADR:

1. `maxUnavailable` làm tròn xuống, `maxSurge` làm tròn lên (bài 63, 348). 2 bản sao: 0 và 1. 4 bản
   sao: 1 và 1 — `web` tụt còn 3 sẵn sàng giữa lúc phát hành. 5 bản sao: 1 và 2. Mặc định đổi nghĩa
   theo số bản sao; số tuyệt đối thì không. Bài 319 thêm một lý do: trường do API server đặt mặc định
   không được tính lại trong patch, nên chiến lược viết sẵn trong file là cách để thay đổi về sau rõ
   ràng.
2. Pod chỉ được tính sẵn sàng khi Ready — với `web` là readiness `GET /healthz`. `minReadySeconds`
   bắt Pod phải giữ Ready liên tục một khoảng, chặn trường hợp Pod Ready rồi crash ngay trong khi
   controller đã kịp hạ Pod cũ.
3. Không. Bài 63: Pod đang kết thúc không được tính vào `availableReplicas`; Pod cũ của `web` chạy
   `preStop` 5 giây và có ân hạn 30 giây, nên còn hiện trong danh sách Pod sau khi ReplicaSet cũ đã
   hạ. Ràng buộc của chiến lược áp lên `.spec.replicas` của các ReplicaSet và lên `availableReplicas`.
4. Bài 63, *Kiểm tra lịch sử rollout*: `CHANGE-CAUSE` được sao từ annotation
   `kubernetes.io/change-cause` của Deployment sang revision khi revision được tạo. Ghi trong manifest
   thì lý do nằm trong cùng commit với thay đổi; `kubectl annotate` là một lần ghi tay ngoài repo —
   đúng kiểu trôi cấu hình mà B8 điều tra.
5. `progressDeadlineSeconds` phải nhỏ hơn. Khi đó `rollout status` thoát với
   `exceeded its progress deadline` — một lỗi có nghĩa, kèm condition để đọc — thay vì hết giờ chờ
   chung chung mà không biết rollout còn đang tiến hay đã kẹt.

</details>

## B5. [BS-4A-05] Phát hành nội dung mới của trang

**Loại:** Tính năng, kèm điều tra.

**Bối cảnh:** chủ hiệu sách muốn trang chủ có dòng `Miễn phí giao hàng nội thành`. Trang mẫu nằm trong
ConfigMap `web-config` từ Lab 3b; init container `render-html` sinh trang từ mẫu **lúc Pod khởi
động**. Một thành viên mới làm như mọi khi: sửa ConfigMap trong repo, chạy `deploy.sh`. Ticket này tái
hiện chuyện xảy ra sau đó, rồi phát hành bản `0.2.0` đúng cách.

**Yêu cầu:**

1. **Phần A.** Thêm đúng dòng `Miễn phí giao hàng nội thành` vào trang mẫu trong `web-config` (không
   đổi gì khác), commit, chạy `./scripts/deploy.sh`. Ghi nhận: ConfigMap trong cluster đã đổi chưa,
   Deployment có phát hành gì không, ba bản sao phục vụ gì. Rồi xóa **một** Pod `web`: ba bản sao phục
   vụ gì?
2. **Phần B.** Phát hành `web` `0.2.0` đúng cách. Cơ chế do bạn chọn trong phạm vi nhóm 4a, nhưng
   phải: đi qua repo, sinh **đúng một** revision mới, mang lý do; sau đó mọi bản sao phục vụ dòng mới
   và báo `version` `0.2.0`.
3. **Phần C.** Lùi `web` về bản trước bằng `rollout undo`, ghi nhận phiên bản và nội dung trang, rồi
   đưa `web` về đúng repo.
4. Viết quy tắc phát hành nội dung cho người trực, có hiệu lực tới khi Lab 5a tự động hóa việc này.

**Ràng buộc:** không teardown, không xóa hàng loạt Pod để "làm mới" trang; không sửa file trong
container. Không đổi selector của `web`.

**Đọc lại:**

- Bài [63](../63-deployment-vi.md): ghi chú đầu mục [*Cập nhật một Deployment*](../63-deployment-vi.md#updating-a-deployment)
  — rollout chỉ khi `.spec.template` đổi; ghi chú đầu mục
  [*Rollback một Deployment*](../63-deployment-vi.md#rolling-back-a-deployment) — rollback chỉ hoàn
  nguyên Pod template.
- Bài [345](../345-run-stateless-application-vi.md#cập-nhật-deployment-updating-the-deployment) —
  cập nhật bằng cách apply lại file: Pod mới tên mới, Pod cũ bị xóa.
- Bài [61](../61-management-vi.md#cập-nhật-annotation-updating-annotations) — annotation; bài
  [20](../20-annotations-vi.md) đã đọc ở giai đoạn 1b.
- [Lab 3b](LAB-3B-CAU-HINH-UNG-DUNG.md) ticket B4 (trang sinh từ mẫu) và B7 (cấu hình đổi thì Pod thấy
  gì).

**Bàn giao:** `k8s/app/web/configmap.yaml`, `k8s/app/web/deployment.yaml` đã cập nhật;
`docs/adr/NNNN-phat-hanh-noi-dung.md`; `~/lab-evidence/4a/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` Phần A — ConfigMap trong cluster có dòng mới; Deployment vẫn báo 3 bản sao up-to-date mà
  không bản sao nào phục vụ dòng mới.
- `PASS:` Phần A — sau khi xóa một Pod, đúng một bản sao phục vụ dòng mới; không có revision mới.
- `PASS:` Phần B — revision tăng đúng 1 so với cuối Phần A; cả ba bản sao phục vụ dòng mới và báo
  `0.2.0`; dòng cuối lịch sử mang lý do của `0.2.0`; chỉ còn một ReplicaSet có bản sao; repo sạch.
- `PASS:` Phần C — sau `rollout undo`, mọi bản sao báo `0.1.1` nhưng **vẫn** phục vụ dòng mới; sau
  `deploy.sh`, mọi bản sao về `0.2.0`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-phat-hanh-noi-dung.md`:

1. Vì sao Phần A không có rollout nào? Vì sao bản sao vừa thay lại có dòng mới? Cột `UP-TO-DATE` của
   `kubectl get deployment` nói gì và **không** nói gì?
2. So sánh ba cách: đổi label `version` trong Pod template; thêm annotation trong Pod template; thêm
   annotation ở `metadata` của Deployment. Cách nào gây rollout, vì sao? Bạn chọn cách nào?
3. Phần C cho thấy `rollout undo` không lấy lại trang cũ. Vì sao? Vậy muốn lùi một thay đổi nội dung
   thì phải làm gì?
4. Lab 5a sinh ConfigMap bằng `configMapGenerator` có hậu tố hash. Câu chuyện của Phần A sẽ khác ở
   đâu?

<details>
<summary>Gợi ý</summary>

1. Dòng mới đặt ngay dưới `<h1>Bookstore</h1>` trong mục trang mẫu của `web-config`. Giữ nguyên các
   chỗ giữ tên Pod, tên node.
2. Xem một bản sao cụ thể phục vụ gì:
   `kubectl -n bookstore exec <Pod> -c status-writer -- wget -qO- http://127.0.0.1:8080/`.
3. Bài 63 nêu ví dụ thứ kích hoạt rollout: đổi **label** hoặc image của template. Label `version` của
   `web` đã có sẵn trong template, và `status-writer` đã đọc nó cho `/status.json` từ Lab 3a.
4. Phần C: `kubectl -n bookstore rollout undo deployment/web`, rồi `./scripts/deploy.sh` để về repo.

</details>

<details>
<summary>Script nghiệm thu — ba phần, chạy đúng thời điểm ghi ở đầu mỗi khối</summary>

Phần A — chạy sau khi đã thêm dòng vào trang mẫu, commit và `./scripts/deploy.sh`, **chưa** đổi gì
khác:

```bash
NS=bookstore; EV=~/lab-evidence/4a
WEB='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web'
DONG='Miễn phí giao hàng nội thành'
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
web_pods() { kubectl -n "$NS" get pods -l "$WEB" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'; }
dem_dong() { local n=0 P; for P in $(web_pods); do
  kubectl -n "$NS" exec "$P" -c status-writer -- wget -qO- http://127.0.0.1:8080/ 2>/dev/null | grep -qF "$DONG" && n=$((n + 1))
  done; echo "$n"; }
rev() { jp deployment/web '{.metadata.annotations.deployment\.kubernetes\.io/revision}'; }
cho3() { for i in $(seq 1 60); do [ "$(kubectl -n "$NS" get pods -l "$WEB" --no-headers | wc -l)" -eq 3 ] && break; sleep 2; done; }
ver_all() { local P; for P in $(web_pods); do   # 0 khi mọi bản sao báo version $1
  kubectl -n "$NS" exec "$P" -c status-writer -- wget -qO- http://127.0.0.1:8080/status.json     | grep -Eq "\"version\" *: *\"$1\"" || return 1; done; }

jp configmap/web-config '{.data.index\.html\.tpl}' | grep -qF "$DONG" && echo 'PASS: ConfigMap trong cluster da co dong moi'
rev > "$EV/b5-rev-phan-a.txt"
{ [ "$(jp deployment/web '{.status.updatedReplicas}/{.status.availableReplicas}')" = '3/3' ] && [ "$(dem_dong)" -eq 0 ]; } \
  && echo 'PASS: Deployment bao 3/3 up-to-date nhung khong ban sao nao phuc vu dong moi'
V="$(kubectl -n "$NS" get pods -l "$WEB" --field-selector spec.nodeName=lab-k8s-worker2 -o jsonpath='{.items[0].metadata.name}')"
[ -n "$V" ] || V="$(web_pods | head -1)"
kubectl -n "$NS" delete pod "$V" --wait=true
cho3; kubectl -n "$NS" wait --for=condition=Ready pod -l "$WEB" --timeout=180s >/dev/null
kubectl -n "$NS" get pods -l "$WEB" -o wide > "$EV/b5-pods-phan-a.txt"
[ "$(dem_dong)" -eq 1 ] && echo 'PASS: Pod thay the co dong moi, hai Pod cu khong — ba ban sao, hai noi dung'
[ "$(rev)" = "$(cat "$EV/b5-rev-phan-a.txt")" ] && echo 'PASS: khong co revision moi nao'
```

Phần B — chạy sau khi phát hành `0.2.0` bằng repo và `./scripts/deploy.sh`, trong cùng shell:

```bash
cho3
[ "$(rev)" = "$(( $(cat "$EV/b5-rev-phan-a.txt") + 1 ))" ] && echo 'PASS: dung mot revision moi'
[ "$(dem_dong)" -eq 3 ] && echo 'PASS: ca ba ban sao phuc vu dong moi'
ver_all '0\.2\.0' && echo 'PASS: moi ban sao bao version 0.2.0'
kubectl -n "$NS" rollout history deployment/web | awk 'NF' | tail -1 | grep -q '0\.2\.0' \
  && echo 'PASS: dong cuoi lich su mang ly do cua 0.2.0'
[ "$(kubectl -n "$NS" get rs -l "$WEB" -o jsonpath='{range .items[*]}{.spec.replicas}{"\n"}{end}' | awk '$1 > 0' | wc -l)" -eq 1 ] \
  && echo 'PASS: chi con mot ReplicaSet co ban sao'
test -z "$(git -C ~/bookstore status --porcelain)" && echo 'PASS: ban phat hanh nam trong repo'
```

Phần C — ngay sau Phần B:

```bash
kubectl -n "$NS" rollout undo deployment/web
kubectl -n "$NS" rollout status deployment/web --timeout=180s >/dev/null; cho3
{ ver_all '0\.1\.1' && [ "$(dem_dong)" -eq 3 ]; } \
  && echo 'PASS: undo dua version ve 0.1.1 nhung trang van co dong moi — ConfigMap khong nam trong revision'
~/bookstore/scripts/deploy.sh > "$EV/b5-deploy-tra-ve.txt" 2>&1; cho3
ver_all '0\.2\.0' && echo 'PASS: deploy.sh dua web ve dung repo (0.2.0)'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Phần A — trong `k8s/app/web/configmap.yaml`, mục `index.html.tpl`, thêm một dòng:

```yaml
    <h1>Bookstore</h1>
    <p class="banner">Miễn phí giao hàng nội thành</p>
```

Phần B — trong `k8s/app/web/deployment.yaml`, đổi hai chỗ trong cùng một commit:

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "0.2.0: them dong mien phi giao hang tren trang chu"
spec:
  template:
    metadata:
      labels:
        app.kubernetes.io/version: "0.2.0"
```

```bash
cd ~/bookstore
git add -A && git commit -qm 'web 0.2.0: trang chu co dong mien phi giao hang'
./scripts/deploy.sh
```

Ý chính cho ADR:

1. Rollout chỉ kích hoạt khi `.spec.template` đổi (bài 63). Sửa ConfigMap không đụng template, nên
   không có ReplicaSet mới, không Pod nào bị thay; trang đã được `render-html` sinh vào `emptyDir` từ
   lúc Pod khởi động. Pod thay thế chạy lại init container, đọc mẫu **mới**. `UP-TO-DATE` đếm Pod
   mang Pod template mới nhất — nó không biết gì về nội dung ConfigMap mà template tham chiếu tới.
2. Hai cách đầu đổi `.spec.template` nên gây rollout (bài 63 nêu đúng ví dụ đổi label của template).
   Annotation ở `metadata` của Deployment nằm ngoài template: không rollout. Lời giải chọn label
   `version`: nó đã là một phần của hợp đồng (`/status.json` báo `version`), nên phát hành nội dung
   cũng là phát hành một phiên bản, và người trực đọc được bản nào đang chạy trên từng bản sao.
   Annotation trong template (ví dụ số hiệu bản sửa trang mẫu) cũng đúng.
3. Bài 63: rollback chỉ hoàn nguyên Pod template. Trang mẫu nằm trong ConfigMap, không thuộc
   revision nào; Pod mới của revision cũ vẫn đọc ConfigMap hiện tại. Muốn lùi nội dung thì lùi trong
   repo (`git revert` cả ConfigMap lẫn version) rồi phát hành như một bản mới.
4. Với hậu tố hash, sửa nội dung là đổi **tên** ConfigMap; tên đó nằm trong Pod template, nên
   template đổi và rollout tự xảy ra — không cần tăng version bằng tay. ConfigMap cũ còn đó với tên
   cũ, nên rollback cũng lấy lại được nội dung cũ.

</details>

## B6. [BS-4A-06] Diễn tập: bản phát hành hỏng

**Loại:** Diễn tập sự cố — mọi Pod diễn tập ghim `lab-k8s-worker2`.

**Bối cảnh:** `P5` của hệ thống cũ: mỗi lần phát hành là một lần đặt cược. `R4` đòi rollback được
trong vài phút. Trước khi tin chiến lược ở B4, đội phải thấy nó chịu một bản phát hành hỏng ra sao.
Diễn tập trên `drill-web` — bản sao rút gọn của `web` với **cùng chiến lược** — không đụng `web` thật.

**Yêu cầu:**

1. Deployment `drill-web` ở `~/lab-work/4a/drill-web.yaml`: 3 bản sao `nginx-unprivileged` dùng cấu
   hình site từ `web-config` (readiness `GET /healthz` như `web`), ghim `lab-k8s-worker2`, chiến lược
   phát hành giống `web` nhưng `progressDeadlineSeconds` rút xuống 60 để diễn tập nhanh. Mỗi bản phát
   hành mang lý do.
2. Bản phát hành hỏng ở `drill-web-hong.yaml`: chỉ khác bản ổn định ở **tag image gõ nhầm** và lý do
   phát hành. Apply nó như một commit lỗi vừa được triển khai.
3. Chứng minh: lệnh chờ rollout thoát khác 0 vì vượt deadline; condition `Progressing` là `False` với
   `ProgressDeadlineExceeded`; ba bản cũ vẫn phục vụ; ReplicaSet cũ giữ 3, ReplicaSet mới dừng ở 1;
   Pod mới kẹt ở lỗi kéo image; spec vẫn giữ image hỏng — không có rollback tự động; lịch sử cho thấy
   lý do và template của revision hỏng.
4. Lùi bằng `rollout undo`, đo thời gian từ lúc ra lệnh tới lúc rollout xong. Chứng minh revision tăng
   tiếp chứ không lùi số.
5. Chứng minh sau khi lùi, cluster đã khác file của bản hỏng — apply lại file đó là phát hành hỏng lần
   nữa.
6. ⭐ Kiểu hỏng thứ hai: bản phát hành chạy được nhưng không bao giờ Ready (readiness trỏ đường dẫn
   không tồn tại). Lùi bằng `--to-revision`.
7. Postmortem và một runbook phát hành cho người trực. Xóa `drill-web`.

**Ràng buộc:** không đụng `web`, `api` thật. Không dùng `nodeName`. Không hạ
`progressDeadlineSeconds` giữa chừng một lần chờ.

**Đọc lại:**

- Bài [63](../63-deployment-vi.md): [*Rollback một Deployment*](../63-deployment-vi.md#rolling-back-a-deployment)
  (ví dụ `nginx:1.161`), [*Deployment thất bại*](../63-deployment-vi.md#failed-deployment),
  [*Thao tác trên một deployment thất bại*](../63-deployment-vi.md#operating-on-a-failed-deployment),
  [*Rollback về một revision trước đó*](../63-deployment-vi.md#rolling-back-to-a-previous-revision).
- Bài [348](../348-update-deployment-rolling-vi.md):
  [*Phát hiện một rollout bị đình trệ*](../348-update-deployment-rolling-vi.md#phát-hiện-một-rollout-bị-đình-trệ-detecting-a-stalled-rollout),
  [*Rollback về một revision trước đó*](../348-update-deployment-rolling-vi.md#rollback).
- Bài [61](../61-management-vi.md#quản-lý-rollout-managing-rollouts) — `rollout status --timeout`.

**Bàn giao:** `docs/postmortem/4a-phat-hanh-hong.md`; `docs/runbook-phat-hanh.md`;
`~/lab-evidence/4a/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` bản ổn định 3/3, mọi Pod trên `lab-k8s-worker2`.
- `PASS:` `rollout status` thoát khác 0 với `exceeded its progress deadline`.
- `PASS:` `Progressing=False`, `ProgressDeadlineExceeded`; `availableReplicas` vẫn 3; ReplicaSet cũ
  3, mới 1; Pod mới ở `ErrImagePull`/`ImagePullBackOff`; spec vẫn giữ image hỏng.
- `PASS:` `rollout history` có lý do của bản hỏng; `--revision` in đúng template hỏng.
- `PASS:` `rollout undo` đưa về bản ổn định trong vài phút; revision tăng tiếp.
- `PASS:` `kubectl diff` với file bản hỏng thoát 1 và chỉ ra tag gõ nhầm.
- `PASS:` (⭐) Pod mới `Running` nhưng không Ready, bản cũ vẫn 3 sẵn sàng; `undo --to-revision` về
  bản ổn định.
- `PASS:` không còn object nào của `drill-web`.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Trường nào giữ cho ba bản cũ không bị đụng tới? Nếu `drill-web` có 4 bản sao và để
   `maxUnavailable` mặc định thì sao?
2. Kubernetes làm gì sau khi vượt deadline? Ai phải quyết định rollback?
3. Với `web` thật, từ lúc `deploy.sh` áp một bản hỏng tới lúc nó báo lỗi mất bao lâu? Con số đó do đâu
   quyết định?
4. Sau `rollout undo`, cluster và repo lệch nhau. Quy trình đúng để đưa hai bên về khớp là gì, và vì
   sao không được dừng ở `undo`?
5. Revision bị lùi về đi đâu trong lịch sử? Vì sao số revision không lùi?

<details>
<summary>Gợi ý</summary>

1. `drill-web` dùng label riêng `app.kubernetes.io/name: drill-web`, **không** dùng `part-of` hay
   `component` của dự án — nếu không, selector của nó chồng lên `web` và PDB `web`.
2. Chỉ mount mục `default.conf` của `web-config` vào `/etc/nginx/conf.d`: đủ cho `/healthz`, không cần
   init container hay sidecar.
3. Tag gõ nhầm kiểu bài 63 (`1.161` thay vì `1.16.1`): `1.305-alpine` thay vì `1.30.5-alpine`.
4. `kubectl rollout status ... --timeout=300s` thoát khác 0 ngay khi controller ghi
   `ProgressDeadlineExceeded`, không chờ hết 300 giây.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã tự viết drill-web.yaml và drill-web-hong.yaml; script này chính là cuộc diễn tập</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4a; W=~/lab-work/4a
DR='app.kubernetes.io/name=drill-web'
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
img() { jp deployment/drill-web '{.spec.template.spec.containers[0].image}'; }
rev() { jp deployment/drill-web '{.metadata.annotations.deployment\.kubernetes\.io/revision}'; }

kubectl apply -f "$W/drill-web.yaml"
kubectl -n "$NS" rollout status deployment/drill-web --timeout=180s >/dev/null && echo 'PASS: ban on dinh 3/3'
[ "$(kubectl -n "$NS" get pods -l "$DR" -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u)" = 'lab-k8s-worker2' ] \
  && echo 'PASS: moi Pod dien tap tren lab-k8s-worker2'
GOOD="$(img)"

# 1. Phát hành hỏng
T0="$(date +%s)"
kubectl apply -f "$W/drill-web-hong.yaml"
if kubectl -n "$NS" rollout status deployment/drill-web --timeout=300s > "$EV/b6-rollout-status.txt" 2>&1; then
  echo 'FAIL: rollout status bao thanh cong voi ban hong'
else
  grep -q 'exceeded its progress deadline' "$EV/b6-rollout-status.txt" \
    && echo "PASS: rollout status thoat khac 0 sau $(( $(date +%s) - T0 )) giay: vuot progress deadline"
fi

# 2. Đọc trạng thái kẹt
kubectl -n "$NS" describe deployment drill-web > "$EV/b6-describe.txt"
COLS='NAME:.metadata.name,DESIRED:.spec.replicas,READY:.status.readyReplicas,IMAGE:.spec.template.spec.containers[0].image'
kubectl -n "$NS" get rs -l "$DR" -o "custom-columns=$COLS" > "$EV/b6-rs.txt"; cat "$EV/b6-rs.txt"
[ "$(jp deployment/drill-web '{.status.conditions[?(@.type=="Progressing")].status}/{.status.conditions[?(@.type=="Progressing")].reason}')" \
  = 'False/ProgressDeadlineExceeded' ] && echo 'PASS: Progressing=False, ProgressDeadlineExceeded'
[ "$(jp deployment/drill-web '{.status.availableReplicas}')" = '3' ] && echo 'PASS: ba ban cu van phuc vu'
{ [ "$(awk -v g="$GOOD" 'NR > 1 && $4 == g {print $2}' "$EV/b6-rs.txt")" = '3' ] \
  && [ "$(awk -v g="$GOOD" 'NR > 1 && $4 != g {print $2}' "$EV/b6-rs.txt")" = '1' ]; } \
  && echo 'PASS: ReplicaSet cu giu 3, ReplicaSet moi dung o 1'
kubectl -n "$NS" get pods -l "$DR" -o jsonpath='{range .items[*]}{.status.containerStatuses[0].state.waiting.reason}{"\n"}{end}' \
  | grep -Eq 'ErrImagePull|ImagePullBackOff' && echo 'PASS: Pod moi ket o loi keo image'
[ "$(img)" != "$GOOD" ] && echo 'PASS: Kubernetes khong tu rollback, spec van giu image hong'
R2="$(rev)"
kubectl -n "$NS" rollout history deployment/drill-web > "$EV/b6-history.txt"
kubectl -n "$NS" rollout history deployment/drill-web --revision="$R2" > "$EV/b6-history-r$R2.txt"
{ grep -q 'drill 2' "$EV/b6-history.txt" && grep -q '1.305-alpine' "$EV/b6-history-r$R2.txt"; } \
  && echo "PASS: lich su co ly do va template cua revision hong $R2"

# 3. Lùi và đo thời gian
T2="$(date +%s)"
kubectl -n "$NS" rollout undo deployment/drill-web
kubectl -n "$NS" rollout status deployment/drill-web --timeout=180s >/dev/null
T3="$(date +%s)"; echo "khoi phuc trong $((T3 - T2)) giay" | tee "$EV/b6-khoi-phuc.txt"
for i in $(seq 1 60); do [ "$(kubectl -n "$NS" get pods -l "$DR" --no-headers | wc -l)" -eq 3 ] && break; sleep 2; done
{ [ "$(img)" = "$GOOD" ] && [ "$(jp deployment/drill-web '{.status.availableReplicas}')" = '3' ] \
  && [ "$((T3 - T2))" -le 180 ]; } && echo 'PASS: undo dua drill-web ve ban on dinh trong vai phut'
[ "$(rev)" = "$((R2 + 1))" ] && echo 'PASS: undo la mot rollout moi, revision tang tiep'

# 4. Cluster đã lùi, file thì chưa
kubectl diff -f "$W/drill-web-hong.yaml" > "$EV/b6-diff.txt" 2>&1; RC=$?
{ [ "$RC" -eq 1 ] && grep -q '1.305-alpine' "$EV/b6-diff.txt"; } \
  && echo 'PASS: file van mang ban hong — apply lai la phat hanh hong lan nua'
```

⭐ Kiểu hỏng thứ hai — cần `drill-web-ready-hong.yaml`; chạy tiếp trong cùng shell:

```bash
RG="$(rev)"
kubectl apply -f "$W/drill-web-ready-hong.yaml"
kubectl -n "$NS" rollout status deployment/drill-web --timeout=300s > "$EV/b6-ready-status.txt" 2>&1
grep -q 'exceeded its progress deadline' "$EV/b6-ready-status.txt" && echo 'PASS: ban khong Ready cung vuot deadline'
NR="$(kubectl -n "$NS" get pods -l "$DR" \
  -o jsonpath='{range .items[*]}{.status.phase}/{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}' | grep -c 'Running/False')"
{ [ "$NR" -ge 1 ] && [ "$(jp deployment/drill-web '{.status.availableReplicas}')" = '3' ]; } \
  && echo 'PASS: Pod moi Running nhung khong Ready; ban cu van 3 san sang'
kubectl -n "$NS" rollout undo deployment/drill-web --to-revision="$RG"
kubectl -n "$NS" rollout status deployment/drill-web --timeout=180s >/dev/null && echo "PASS: undo --to-revision=$RG ve ban on dinh"
```

Dọn:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
kubectl -n "$NS" delete deployment drill-web --wait=true
for i in $(seq 1 60); do [ "$(sl kubectl -n "$NS" get rs,pods -l "$DR" --no-headers)" -eq 0 ] && break; sleep 2; done
[ "$(sl kubectl -n "$NS" get deploy,rs,pods -l "$DR" --no-headers)" -eq 0 ] && echo 'PASS: da don drill-web'
```

</details>

<details>
<summary>Lời giải tham khảo — manifest, ý chính của postmortem, runbook</summary>

```yaml
# ~/lab-work/4a/drill-web.yaml — bản sao rút gọn của web để diễn tập, ghim lab-k8s-worker2
apiVersion: apps/v1
kind: Deployment
metadata:
  name: drill-web
  namespace: bookstore
  labels:
    app.kubernetes.io/name: drill-web
  annotations:
    kubernetes.io/change-cause: "drill 1: ban on dinh"
spec:
  replicas: 3
  revisionHistoryLimit: 5
  minReadySeconds: 5
  progressDeadlineSeconds: 60       # rút ngắn để diễn tập; web dùng giá trị của B4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app.kubernetes.io/name: drill-web
  template:
    metadata:
      labels:
        app.kubernetes.io/name: drill-web
    spec:
      nodeSelector:
        kubernetes.io/hostname: lab-k8s-worker2
      volumes:
      - name: nginx-conf
        configMap: {name: web-config, items: [{key: default.conf, path: default.conf}]}
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged:1.30.5-alpine
        ports:
        - {name: http, containerPort: 8080}
        readinessProbe:
          httpGet: {path: /healthz, port: http}
          periodSeconds: 5
        resources:
          requests: {cpu: 20m, memory: 32Mi}
          limits: {cpu: 100m, memory: 64Mi}
        volumeMounts:
        - {name: nginx-conf, mountPath: /etc/nginx/conf.d, readOnly: true}
```

- `drill-web-hong.yaml`: bản sao của file trên, lý do `"drill 2: go nham tag image"`, image
  `nginxinc/nginx-unprivileged:1.305-alpine`.
- ⭐ `drill-web-ready-hong.yaml`: bản sao của bản ổn định, lý do
  `"drill 3: readiness tro sai duong dan"`, readiness `path: /khong-co`.

Ý chính cho postmortem:

1. `maxUnavailable: 0`: controller không được hạ Pod cũ nào khi chưa có Pod mới sẵn sàng, và
   `maxSurge: 1` chỉ cho tạo đúng một Pod mới (bài 63, ghi chú trong *Rollback một Deployment*). Với 4
   bản sao và mặc định 25%, `maxUnavailable` là 1: một Pod cũ bị hạ ngay, dịch vụ còn 3/4 trong suốt
   thời gian kẹt.
2. Chỉ đặt condition `Progressing=False`, `reason: ProgressDeadlineExceeded` rồi tiếp tục thử lại —
   không rollback (bài 63, *Deployment thất bại*). Quyết định là của người trực hoặc của một bộ điều
   phối cấp cao hơn.
3. Khoảng `progressDeadlineSeconds` của `web` (B4) — phụ thuộc cấu hình, không phải hằng số của
   Kubernetes. `deploy.sh` thất bại theo vì nó chờ bằng `rollout status`.
4. `undo` là phanh khẩn cấp, không phải sửa lỗi: repo vẫn chứa bản hỏng, lần `deploy.sh` kế tiếp sẽ
   phát hành hỏng lại. Quy trình: `undo` → `git revert` commit hỏng → `deploy.sh` → `kubectl diff`
   không còn khác biệt.
5. Lịch sử nằm trong ReplicaSet. `undo` dùng lại ReplicaSet cũ và gán cho nó số revision kế tiếp, nên
   số cũ biến mất khỏi danh sách và số mới tăng tiếp.

```markdown
# Runbook phát hành web / api

1. Sửa repo. Nội dung, cấu hình hay image của web đổi thì tăng `app.kubernetes.io/version` trong Pod
   template; luôn ghi `kubernetes.io/change-cause`. Commit.
2. `./scripts/deploy.sh`. Thoát 0 là xong. Xem nhanh:
   `kubectl -n bookstore rollout status deployment/web --watch=false`.
3. `deploy.sh` thoát khác 0 với `exceeded its progress deadline`: bản cũ vẫn phục vụ. Đọc
   `kubectl -n bookstore describe deployment web` (Conditions, Events) và Pod mới.
4. Lùi: `kubectl -n bookstore rollout undo deployment/web` (hoặc `--to-revision=N` theo
   `rollout history`), rồi `rollout status`.
5. Đưa repo về khớp: `git revert <commit>`, `./scripts/deploy.sh`, rồi
   `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | kubectl diff -f -` không còn
   khác biệt.
6. Lùi **nội dung** trang thì không dùng `undo` (ConfigMap không thuộc revision): revert trong repo và
   phát hành bản mới.
7. Ghi postmortem.
```

</details>

## B7. [BS-4A-07] Điều tra: `Recreate` hay `RollingUpdate`

**Loại:** Điều tra.

**Bối cảnh:** một thành viên đề xuất phát hành `web` bằng `Recreate` "cho gọn, hai phiên bản không bao
giờ chạy cùng lúc". Đội cần số đo trên chính cluster này trước khi trả lời — và cần biết lời hứa
"không bao giờ cùng lúc" của `Recreate` đúng tới đâu.

**Yêu cầu:**

1. Hai Deployment thử nghiệm ghim `lab-k8s-worker2`, mỗi cái 3 bản sao nginx giống `drill-web`:
   `thu-rolling` dùng chiến lược của `web` (B4); `thu-recreate` dùng `Recreate`, có `preStop` giữ Pod
   cũ thêm 15 giây để khoảng trống quan sát được.
2. Phát hành một lần trên mỗi cái và lấy mẫu bằng công cụ của B4. Chứng minh: `RollingUpdate` luôn
   giữ ≥ 3 bản sao sẵn sàng, tổng ≤ 4; `Recreate` có lúc **không** bản sao nào sẵn sàng, tổng không
   vượt 3.
3. Xóa tay một Pod của `thu-recreate`: chứng minh có lúc 4 Pod cùng tồn tại.
4. Đổi `thu-rolling` sang `Recreate` bằng strategic merge patch thường: chứng minh bị từ chối và vì
   sao; rồi làm cho nó đi qua.
5. ⭐ Tạm dừng rollout của `thu-rolling`, sửa template hai lần: không có ReplicaSet hay revision mới,
   không rollback được khi đang dừng; tiếp tục thì hai thay đổi thành **một** revision.
6. ⭐ `revisionHistoryLimit: 1` rồi phát hành ba lần: chỉ còn một ReplicaSet cũ.
7. Xóa hai Deployment thử nghiệm.

**Ràng buộc:** không đổi chiến lược của `web`, `api` thật. Phát hành trên Deployment thử nghiệm bằng
`kubectl patch` là được phép — chúng không thuộc repo.

**Đọc lại:**

- Bài [63](../63-deployment-vi.md): [*Deployment kiểu Recreate*](../63-deployment-vi.md#recreate-deployment)
  — cả ghi chú; [*Tạm dừng và tiếp tục rollout*](../63-deployment-vi.md#pausing-and-resuming-a-deployment);
  [*Chính sách dọn dẹp*](../63-deployment-vi.md#clean-up-policy).
- Bài [324](../324-kubectl-patch-vi.md#sử-dụng-strategic-merge-patch-để-cập-nhật-một-deployment-với-chiến-lược-retainkeys-use-strategic-merge-patch-to-update-a-deployment-using-the-retainkeys-strategy)
  — `retainKeys`.
- Bài [319](../319-declarative-config-vi.md#giá-trị-mặc-định-của-field-default-field-values) — ví dụ
  đổi sang `Recreate` bằng `apply` thất bại.
- Bài [348](../348-update-deployment-rolling-vi.md#tạm-dừng-và-tiếp-tục-một-rollout-pausing-and-resuming-a-rollout)
  — pause và resume.

**Bàn giao:** `docs/adr/NNNN-recreate-hay-rollingupdate.md`; `~/lab-evidence/4a/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` hai Deployment thử nghiệm chạy trên `lab-k8s-worker2`.
- `PASS:` `RollingUpdate`: ≥ 3 sẵn sàng, tổng ≤ 4 trong mọi mẫu.
- `PASS:` `Recreate`: có mẫu 0 sẵn sàng, tổng ≤ 3.
- `PASS:` xóa tay một Pod của `thu-recreate` thì có lúc 4 Pod.
- `PASS:` patch đổi sang `Recreate` bị từ chối vì `rollingUpdate`; với `$retainKeys` thì qua.
- `PASS:` (⭐) đang pause: không ReplicaSet, không revision mới, không rollback được; resume ra đúng
  một revision.
- `PASS:` (⭐) `revisionHistoryLimit: 1` chỉ giữ một ReplicaSet cũ.
- `PASS:` hai Deployment thử nghiệm đã xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-recreate-hay-rollingupdate.md`:

1. Bằng số đo của bạn: `web` có nên dùng `Recreate` không? Khi nào một thành phần **phải** dùng
   `Recreate`?
2. Bước 3 nói gì về lời hứa "không bao giờ hai phiên bản cùng lúc"? Vì sao vì thế `db` không lên
   Deployment `Recreate` mà chờ StatefulSet ở Lab 4b?
3. Vì sao patch thường bị từ chối, trong khi nếu `web` đổi sang `Recreate` qua repo thì `apply` làm
   được — với điều kiện gì về file của bạn?
4. ⭐ Khi nào người trực dùng `rollout pause`? Vì sao trong lúc pause không vượt deadline được?

<details>
<summary>Gợi ý</summary>

1. `thu-rolling.yaml` là `drill-web.yaml` đổi tên và label thành `thu-rolling`,
   `progressDeadlineSeconds` về 180. `thu-recreate.yaml` đổi thêm `strategy` thành
   `type: Recreate` (không có `rollingUpdate`), thêm `terminationGracePeriodSeconds: 40` và `preStop`
   `sh -c "sleep 15"`.
2. Một lần phát hành trên object thử nghiệm: patch label version trong template.
3. `$retainKeys` liệt kê khóa cần giữ trong `strategy`; khóa vắng mặt bị xóa.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã tự viết thu-rolling.yaml và thu-recreate.yaml; script này chính là cuộc điều tra</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4a; W=~/lab-work/4a
. "$W/lay-mau.sh"
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
phat_hanh() { kubectl -n "$NS" patch deployment "$1" \
  -p "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"app.kubernetes.io/version\":\"$2\"}}}}}" >/dev/null; }
gen() { jp "deployment/$1" '{.metadata.generation}'; }
rev() { jp "deployment/$1" '{.metadata.annotations.deployment\.kubernetes\.io/revision}'; }
nrs() { kubectl -n "$NS" get rs -l "app.kubernetes.io/name=$1" --no-headers | wc -l; }
min_av() { awk '{print $1}' "$1" | sort -n | head -1; }
max_sum() { awk '{print $2}' "$1" | sort -n | tail -1; }
THU='app.kubernetes.io/name in (thu-rolling,thu-recreate)'

kubectl apply -f "$W/thu-rolling.yaml" -f "$W/thu-recreate.yaml"
kubectl -n "$NS" rollout status deployment/thu-rolling --timeout=180s >/dev/null
kubectl -n "$NS" rollout status deployment/thu-recreate --timeout=180s >/dev/null
[ "$(kubectl -n "$NS" get pods -l "$THU" -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u)" = 'lab-k8s-worker2' ] \
  && echo 'PASS: hai Deployment thu nghiem chay tren lab-k8s-worker2'

# 1. RollingUpdate
G="$(gen thu-rolling)"; phat_hanh thu-rolling 2
N="$(lay_mau thu-rolling app.kubernetes.io/name=thu-rolling "$EV/b7-mau-rolling.txt" 3 "$G")"
echo "rolling: $N mau, min av $(min_av "$EV/b7-mau-rolling.txt"), max tong $(max_sum "$EV/b7-mau-rolling.txt")" | tee "$EV/b7-ket-luan.txt"
{ [ "$N" -ge 5 ] && [ "$(min_av "$EV/b7-mau-rolling.txt")" -ge 3 ] && [ "$(max_sum "$EV/b7-mau-rolling.txt")" -le 4 ]; } \
  && echo 'PASS: RollingUpdate luon >= 3 san sang, tong <= 4'

# 2. Recreate
G="$(gen thu-recreate)"; phat_hanh thu-recreate 2
N="$(lay_mau thu-recreate app.kubernetes.io/name=thu-recreate "$EV/b7-mau-recreate.txt" 3 "$G")"
echo "recreate: $N mau, min av $(min_av "$EV/b7-mau-recreate.txt"), max tong $(max_sum "$EV/b7-mau-recreate.txt")" | tee -a "$EV/b7-ket-luan.txt"
{ [ "$N" -ge 5 ] && [ "$(min_av "$EV/b7-mau-recreate.txt")" -eq 0 ] && [ "$(max_sum "$EV/b7-mau-recreate.txt")" -le 3 ]; } \
  && echo 'PASS: Recreate co luc 0 ban sao san sang, tong khong vuot 3'

# 3. Recreate không bảo đảm gì khi xóa tay
V="$(kubectl -n "$NS" get pods -l app.kubernetes.io/name=thu-recreate -o jsonpath='{.items[0].metadata.name}')"
kubectl -n "$NS" delete pod "$V" --wait=false
for i in $(seq 1 30); do
  P="$(kubectl -n "$NS" get pods -l app.kubernetes.io/name=thu-recreate --no-headers | wc -l)"; [ "$P" -ge 4 ] && break; sleep 1
done
kubectl -n "$NS" get pods -l app.kubernetes.io/name=thu-recreate > "$EV/b7-xoa-tay.txt"
[ "$P" -ge 4 ] && echo 'PASS: xoa tay: Pod thay the co ngay khi Pod cu con Terminating — 4 Pod'

# 4. Đổi chiến lược bằng patch
kubectl -n "$NS" patch deployment thu-rolling --type strategic -p '{"spec":{"strategy":{"type":"Recreate"}}}' \
  > "$EV/b7-patch-recreate.txt" 2>&1 && echo 'FAIL: patch Recreate qua duoc khi con rollingUpdate' \
  || { grep -q 'may not be specified when strategy' "$EV/b7-patch-recreate.txt" && echo 'PASS: patch thuong bi tu choi vi con rollingUpdate'; }
kubectl -n "$NS" patch deployment thu-rolling --type strategic \
  -p '{"spec":{"strategy":{"$retainKeys":["type"],"type":"Recreate"}}}' >/dev/null
{ [ "$(jp deployment/thu-rolling '{.spec.strategy.type}')" = 'Recreate' ] \
  && [ -z "$(jp deployment/thu-rolling '{.spec.strategy.rollingUpdate}')" ]; } \
  && echo 'PASS: $retainKeys go rollingUpdate, doi duoc sang Recreate'
```

⭐ Pause và dọn lịch sử — chạy tiếp trong cùng shell:

```bash
R0="$(rev thu-rolling)"; RS0="$(nrs thu-rolling)"
kubectl -n "$NS" rollout pause deployment/thu-rolling
phat_hanh thu-rolling 3
kubectl -n "$NS" set env deployment/thu-rolling GHI_CHU=gom-hai-thay-doi >/dev/null
sleep 10
{ [ "$(nrs thu-rolling)" = "$RS0" ] && [ "$(rev thu-rolling)" = "$R0" ]; } \
  && echo 'PASS: dang pause: sua template khong tao ReplicaSet, khong tao revision'
kubectl -n "$NS" rollout undo deployment/thu-rolling > "$EV/b7-undo-pause.txt" 2>&1 \
  && echo 'FAIL: rollback duoc khi dang pause' || echo 'PASS: khong rollback duoc Deployment dang pause'
kubectl -n "$NS" rollout resume deployment/thu-rolling
kubectl -n "$NS" rollout status deployment/thu-rolling --timeout=180s >/dev/null
[ "$(rev thu-rolling)" = "$((R0 + 1))" ] && echo 'PASS: resume: hai thay doi thanh dung mot revision'

kubectl -n "$NS" patch deployment thu-rolling -p '{"spec":{"revisionHistoryLimit":1}}' >/dev/null
for v in 4 5 6; do phat_hanh thu-rolling "$v"; kubectl -n "$NS" rollout status deployment/thu-rolling --timeout=180s >/dev/null; done
for i in $(seq 1 30); do [ "$(nrs thu-rolling)" -le 2 ] && break; sleep 2; done
kubectl -n "$NS" rollout history deployment/thu-rolling > "$EV/b7-history-gc.txt"
[ "$(nrs thu-rolling)" -le 2 ] && echo 'PASS: revisionHistoryLimit=1 chi giu mot ReplicaSet cu'
```

Dọn:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
kubectl -n "$NS" delete deployment thu-rolling thu-recreate --ignore-not-found --wait=true
for i in $(seq 1 60); do [ "$(sl kubectl -n "$NS" get rs,pods -l "$THU" --no-headers)" -eq 0 ] && break; sleep 2; done
[ "$(sl kubectl -n "$NS" get deploy,rs,pods -l "$THU" --no-headers)" -eq 0 ] && echo 'PASS: da xoa hai Deployment thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

`thu-rolling.yaml` là `drill-web.yaml` của B6 với `name` và label `app.kubernetes.io/name` đổi thành
`thu-rolling`, lý do `"thu 1"`, `progressDeadlineSeconds: 180`. `thu-recreate.yaml` khác ở phần dưới:

```yaml
# ~/lab-work/4a/thu-recreate.yaml — phần khác thu-rolling.yaml
metadata:
  name: thu-recreate
  labels:
    app.kubernetes.io/name: thu-recreate
spec:
  strategy:
    type: Recreate                  # không có khối rollingUpdate
  selector:
    matchLabels:
      app.kubernetes.io/name: thu-recreate
  template:
    metadata:
      labels:
        app.kubernetes.io/name: thu-recreate
    spec:
      terminationGracePeriodSeconds: 40
      containers:
      - name: nginx
        # image, cổng, readiness, tài nguyên, mount: như thu-rolling
        lifecycle:
          preStop:
            exec:
              command: ["sh", "-c", "sleep 15"]
```

Ý chính cho ADR:

1. Không: `Recreate` hạ `web` về 0 bản sao sẵn sàng ở mỗi lần phát hành — đúng thứ `R4` cấm; bài
   348 nói thẳng `Recreate` gây gián đoạn. Dùng khi hai phiên bản **không được** chạy cùng lúc, ví dụ
   bản mới đổi định dạng dữ liệu mà bản cũ không đọc được, và chấp nhận downtime.
2. Bảo đảm của `Recreate` chỉ áp cho **lần nâng cấp**; Pod bị xóa tay do ReplicaSet lo và được thay
   ngay, kể cả khi Pod cũ còn `Terminating` (bài 63). Với DB, hai tiến trình cùng ghi một thư mục dữ
   liệu là hỏng dữ liệu; bài 63 chỉ sang StatefulSet cho bảo đảm "nhiều nhất" — Lab 4b.
3. Strategic merge patch chỉ đặt `type`; `rollingUpdate` vẫn còn, mà hai thứ không được cùng tồn
   tại — `$retainKeys` xóa khóa không liệt kê (bài 324). Với `apply`, nếu file đã khai
   `rollingUpdate` từ đầu (như `web` từ B4) thì bỏ nó khỏi file là `apply` xóa nó, vì nó có trong
   `last-applied-configuration`; nếu file chưa từng khai, `apply` vấp đúng lỗi của bài 319.
4. ⭐ Khi cần gom nhiều thay đổi vào một lần phát hành, hoặc giữ một rollout lưng chừng để kiểm tra.
   Bài 63: khi pause, Kubernetes không kiểm tiến trình so với deadline.

</details>

## B8. [BS-4A-08] Điều tra: `apply`, `scale`, `patch` — trôi cấu hình

**Loại:** Điều tra — phần A, B trên `web` thật rồi trả về repo; phần C trên Deployment thử nghiệm.

**Bối cảnh:** quy tắc 1 của [mục 5.3](DU-AN-BOOKSTORE.md#53-quy-tắc-làm-việc-với-repo): repo là nguồn
sự thật duy nhất. Nhưng lúc sự cố, người trực sẽ gõ `kubectl scale` hay `kubectl patch`. Đội cần biết
chính xác: thay đổi tay nào lần `deploy.sh` kế tiếp ghi đè, thay đổi nào **ở lại** trên cluster mà
không ai thấy — và từng loại patch làm gì với một list như `containers`.

**Yêu cầu:**

1. **Phần A — scale.** Scale `web` lên 5 bằng lệnh. Chứng minh: `last-applied-configuration` vẫn ghi
   3; `kubectl diff` giữa repo và cluster thấy khác biệt ở `replicas`; `deploy.sh` kéo `web` về 3.
2. **Phần B — patch ngoài repo.** Thêm vào Pod template của `web`, bằng `kubectl patch`, annotation
   `bookstore.example.com/hotfix: thu`. Chứng minh: nó sinh một rollout; `deploy.sh` **không** gỡ nó;
   `kubectl diff` giữa repo và cluster **không** thấy gì; dấu vết chỉ nằm ở chỗ cấu hình thực tế có mà
   `last-applied-configuration` không có. Gỡ nó, chứng minh Deployment quay về đúng ReplicaSet trước
   hotfix và cluster khớp repo trở lại.
3. **Phần C — các kiểu patch** trên Deployment thử nghiệm `thu-patch` (`busybox`, 2 bản sao, ghim
   `lab-k8s-worker2`): cùng một file patch thêm container `phu`, strategic merge patch **trộn** vào
   list còn JSON merge patch **thay** cả list; patch qua subresource `scale`; JSON patch có phép
   `test` chạy khi tiền kiểm đúng và bị chặn khi sai; scale về 0 giữ Deployment và ReplicaSet.
4. ⭐ **Phần D.** `kubectl replace --force` trên `thu-patch`: chứng minh object bị xóa rồi tạo lại (UID
   đổi), mất `last-applied-configuration`, và lần `apply` sau cảnh báo về điều đó.
5. Xóa `thu-patch`; `web` khớp repo.

**Ràng buộc:** phần A, B chỉ dùng `kubectl scale` và `kubectl patch` đúng như yêu cầu, rồi trả `web`
về repo trong chính ticket này. Không `kubectl edit` trên `web`. Không dùng `--type merge` trên `web`.

**Đọc lại:**

- Bài [319](../319-declarative-config-vi.md): [*Cách cập nhật object*](../319-declarative-config-vi.md#cách-cập-nhật-object-how-to-update-objects),
  [*Tính toán merge patch*](../319-declarative-config-vi.md#tính-toán-merge-patch-merge-patch-calculation),
  [*Merge từng phần tử của một list gồm các phần tử phức hợp*](../319-declarative-config-vi.md#merge-từng-phần-tử-của-một-list-gồm-các-phần-tử-phức-hợp-merge-individual-elements-of-a-list-of-complex-elements),
  [*Cách xóa các field do server đặt mặc định hoặc do người ghi khác đặt*](../319-declarative-config-vi.md#cách-xóa-các-field-do-server-đặt-mặc-định-hoặc-do-người-ghi-khác-đặt-how-to-clear-server-defaulted-fields-or-fields-set-by-other-writers).
- Bài [63](../63-deployment-vi.md#replicas) — `kubectl apply` ghi đè thao tác scale thủ công.
- Bài [324](../324-kubectl-patch-vi.md): [*strategic merge patch*](../324-kubectl-patch-vi.md#sử-dụng-strategic-merge-patch-để-cập-nhật-một-deployment-use-a-strategic-merge-patch-to-update-a-deployment),
  [*JSON merge patch*](../324-kubectl-patch-vi.md#sử-dụng-json-merge-patch-để-cập-nhật-một-deployment-use-a-json-merge-patch-to-update-a-deployment),
  [*`--subresource`*](../324-kubectl-patch-vi.md#scale-kubectl-patch).
- Bài [346](../346-scale-deployment-vi.md): [*Scale bằng `kubectl patch`*](../346-scale-deployment-vi.md#scale-bằng-kubectl-patch-scale-using-kubectl-patch),
  [*Scale về không*](../346-scale-deployment-vi.md#scale-về-không-scaling-to-zero).
- Bài [61](../61-management-vi.md#cập-nhật-gây-gián-đoạn-disruptive-updates) — `replace --force`.

**Bàn giao:** `docs/adr/NNNN-troi-cau-hinh.md`; `~/lab-evidence/4a/b8-*.txt`.

**Nghiệm thu:**

- `PASS:` trước thí nghiệm, `kubectl diff` giữa repo và cluster không có khác biệt.
- `PASS:` (A) sau `scale`, cluster có 5 nhưng `last-applied-configuration` ghi 3; `diff` thoát 1 và
  chỉ ra `replicas`; `deploy.sh` đưa về 3.
- `PASS:` (B) patch sinh đúng một revision; sau `deploy.sh` annotation vẫn còn; `diff` thoát 0;
  annotation có trong cấu hình thực tế, không có trong `last-applied-configuration`.
- `PASS:` (B) gỡ hotfix thì Deployment dùng lại đúng ReplicaSet trước hotfix; `diff` thoát 0.
- `PASS:` (C) strategic merge patch cho 2 container; JSON merge patch cùng file chỉ còn `phu`.
- `PASS:` (C) patch subresource in `scale.autoscaling/thu-patch patched`; JSON patch `test` đúng thì
  chạy, sai thì cả patch bị từ chối; scale về 0 không còn Pod nhưng còn Deployment và ReplicaSet.
- `PASS:` (⭐ D) `replace --force` in `deleted` rồi `replaced`, UID đổi, mất annotation; `apply` sau đó
  cảnh báo thiếu `last-applied-configuration`.
- `PASS:` `thu-patch` đã xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-troi-cau-hinh.md`:

1. 9 giờ, người trực `kubectl scale deployment/web --replicas=5` vì tải tăng. 10 giờ, đồng đội chạy
   `deploy.sh` cho một bản vá. `web` còn mấy bản sao, vì sao? Điều này liên quan gì tới việc Lab 11b
   phải bỏ `replicas` khỏi manifest khi HPA vào?
2. Giải thích phần B bằng ba nguồn của phép merge (bài 319). Vì sao `kubectl diff` — vốn chạy đúng
   phép tính của `apply` — không thấy trôi? Dự án có sẵn cơ chế nào bắt được loại trôi này?
3. Cùng một file patch, strategic merge và JSON merge cho kết quả ngược nhau trên `containers`. Vì
   sao? Chuyện gì xảy ra với `web` nếu ai đó dùng `--type merge` để "thêm một container"?
4. Viết quy tắc cho người trực: lệnh tay nào được dùng trong sự cố trên `web`, `api`, và sau đó phải
   làm gì; lệnh nào cấm.

<details>
<summary>Gợi ý</summary>

1. So repo với cluster cho phần Kustomize:
   `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | kubectl diff -f -`.
   `diff` thoát 0 khi không khác, 1 khi có khác biệt.
2. Đọc annotation bằng `kubectl -n bookstore get deploy web -o jsonpath=...` với đường dẫn
   `{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}`.
3. Strategic merge patch dùng `null` để xóa một khóa của map.
4. File patch của bài 324: một `spec.template.spec.containers` gồm đúng một phần tử mới.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã tự viết thu-patch.yaml và patch-them-container.yaml; script này chính là cuộc điều tra</summary>

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
NS=bookstore; EV=~/lab-evidence/4a; W=~/lab-work/4a
WEB='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web'
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
la() { jp "deployment/$1" '{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}'; }
rev() { jp deployment/web '{.metadata.annotations.deployment\.kubernetes\.io/revision}'; }
rs_chay() { kubectl -n "$NS" get rs -l "$WEB" -o jsonpath='{range .items[*]}{.metadata.name} {.spec.replicas}{"\n"}{end}' | awk '$2 > 0 {print $1}'; }
so_repo() { kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | kubectl diff -f - > "$1" 2>&1; echo $?; }
HOTFIX='{.spec.template.metadata.annotations.bookstore\.example\.com/hotfix}'

./scripts/deploy.sh > "$EV/b8-deploy-0.txt" 2>&1
[ "$(so_repo "$EV/b8-diff-0.txt")" = '0' ] && echo 'PASS: truoc thi nghiem, cluster khop repo'

# A. scale
kubectl -n "$NS" scale deployment/web --replicas=5
kubectl -n "$NS" rollout status deployment/web --timeout=180s >/dev/null
{ [ "$(jp deployment/web '{.spec.replicas}')" = '5' ] && la web | grep -q '"replicas":3'; } \
  && echo 'PASS: cluster co 5, last-applied van ghi 3'
{ [ "$(so_repo "$EV/b8-diff-scale.txt")" = '1' ] && grep -Eq '^[-+] +replicas: [35]$' "$EV/b8-diff-scale.txt"; } \
  && echo 'PASS: kubectl diff thay troi o replicas'
./scripts/deploy.sh > "$EV/b8-deploy-a.txt" 2>&1
[ "$(jp deployment/web '{.spec.replicas}')" = '3' ] && echo 'PASS: deploy.sh keo web ve 3 ban sao cua repo'

# B. patch ngoài repo
RSA="$(rs_chay)"; REVA="$(rev)"
kubectl -n "$NS" patch deployment web -p '{"spec":{"template":{"metadata":{"annotations":{"bookstore.example.com/hotfix":"thu"}}}}}'
kubectl -n "$NS" rollout status deployment/web --timeout=180s >/dev/null
[ "$(rev)" = "$((REVA + 1))" ] && echo 'PASS: patch vao template sinh dung mot rollout'
./scripts/deploy.sh > "$EV/b8-deploy-b.txt" 2>&1
[ "$(jp deployment/web "$HOTFIX")" = 'thu' ] && echo 'PASS: deploy.sh khong go hotfix'
[ "$(so_repo "$EV/b8-diff-hotfix.txt")" = '0' ] && echo 'PASS: kubectl diff khong thay troi nay'
! la web | grep -q hotfix && echo 'PASS: dau vet: co trong cau hinh thuc te, khong co trong last-applied'
kubectl -n "$NS" patch deployment web -p '{"spec":{"template":{"metadata":{"annotations":{"bookstore.example.com/hotfix":null}}}}}'
kubectl -n "$NS" rollout status deployment/web --timeout=180s >/dev/null
{ [ -z "$(jp deployment/web "$HOTFIX")" ] && [ "$(rs_chay)" = "$RSA" ]; } \
  && echo "PASS: go hotfix thi Deployment dung lai dung ReplicaSet $RSA"
[ "$(so_repo "$EV/b8-diff-sau.txt")" = '0' ] && echo 'PASS: cluster khop repo tro lai'

# C. các kiểu patch trên thu-patch
kubectl apply -f "$W/thu-patch.yaml"
kubectl -n "$NS" rollout status deployment/thu-patch --timeout=180s >/dev/null
kubectl -n "$NS" patch deployment thu-patch --patch-file "$W/patch-them-container.yaml"
kubectl -n "$NS" rollout status deployment/thu-patch --timeout=180s >/dev/null
[ "$(jp deployment/thu-patch '{.spec.template.spec.containers[*].name}' | wc -w)" -eq 2 ] \
  && echo 'PASS: strategic merge patch tron container phu vao list'
kubectl -n "$NS" patch deployment thu-patch --type merge --patch-file "$W/patch-them-container.yaml"
kubectl -n "$NS" rollout status deployment/thu-patch --timeout=180s >/dev/null
[ "$(jp deployment/thu-patch '{.spec.template.spec.containers[*].name}')" = 'phu' ] \
  && echo 'PASS: JSON merge patch cung file thay ca list, chi con phu'
kubectl -n "$NS" patch deployment thu-patch --subresource=scale --type=merge -p '{"spec":{"replicas":3}}' > "$EV/b8-subresource.txt"
{ grep -q '^scale.autoscaling/thu-patch patched' "$EV/b8-subresource.txt" && [ "$(jp deployment/thu-patch '{.spec.replicas}')" = '3' ]; } \
  && echo 'PASS: patch qua subresource scale'
kubectl -n "$NS" patch deployment thu-patch --type=json \
  -p='[{"op":"test","path":"/spec/replicas","value":3},{"op":"replace","path":"/spec/replicas","value":2}]' >/dev/null \
  && echo 'PASS: test dung thi patch chay'
kubectl -n "$NS" patch deployment thu-patch --type=json \
  -p='[{"op":"test","path":"/spec/replicas","value":3},{"op":"replace","path":"/spec/replicas","value":5}]' > "$EV/b8-test-sai.txt" 2>&1 \
  && echo 'FAIL: test sai ma patch van chay' || echo 'PASS: test sai thi ca patch bi tu choi'
[ "$(jp deployment/thu-patch '{.spec.replicas}')" = '2' ] && echo 'PASS: replicas van la 2'
NRS="$(kubectl -n "$NS" get rs -l app.kubernetes.io/name=thu-patch --no-headers | wc -l)"
kubectl -n "$NS" scale deployment/thu-patch --replicas=0
for i in $(seq 1 60); do [ "$(sl kubectl -n "$NS" get pods -l app.kubernetes.io/name=thu-patch --no-headers)" -eq 0 ] && break; sleep 2; done
{ [ "$(sl kubectl -n "$NS" get pods -l app.kubernetes.io/name=thu-patch --no-headers)" -eq 0 ] \
  && kubectl -n "$NS" get deployment thu-patch >/dev/null 2>&1 \
  && [ "$(kubectl -n "$NS" get rs -l app.kubernetes.io/name=thu-patch --no-headers | wc -l)" = "$NRS" ]; } \
  && echo 'PASS: scale ve 0: het Pod, con Deployment va ReplicaSet'
```

⭐ Phần D và dọn — chạy tiếp trong cùng shell:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
U0="$(jp deployment/thu-patch '{.metadata.uid}')"
kubectl replace --force -f "$W/thu-patch.yaml" > "$EV/b8-replace.txt" 2>&1
{ grep -q 'deleted' "$EV/b8-replace.txt" && grep -q 'replaced' "$EV/b8-replace.txt" \
  && [ "$(jp deployment/thu-patch '{.metadata.uid}')" != "$U0" ] && [ -z "$(la thu-patch)" ]; } \
  && echo 'PASS: replace --force xoa roi tao lai: UID moi, mat last-applied-configuration'
kubectl apply -f "$W/thu-patch.yaml" > "$EV/b8-apply-sau-replace.txt" 2>&1
grep -q 'missing the kubectl.kubernetes.io/last-applied-configuration' "$EV/b8-apply-sau-replace.txt" \
  && echo 'PASS: apply canh bao thieu annotation'

kubectl -n "$NS" delete deployment thu-patch --ignore-not-found --wait=true
for i in $(seq 1 60); do [ "$(sl kubectl -n "$NS" get rs,pods -l app.kubernetes.io/name=thu-patch --no-headers)" -eq 0 ] && break; sleep 2; done
[ "$(sl kubectl -n "$NS" get deploy,rs,pods -l app.kubernetes.io/name=thu-patch --no-headers)" -eq 0 ] \
  && echo 'PASS: da xoa thu-patch'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/4a/thu-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thu-patch
  namespace: bookstore
  labels:
    app.kubernetes.io/name: thu-patch
spec:
  replicas: 2
  selector:
    matchLabels:
      app.kubernetes.io/name: thu-patch
  template:
    metadata:
      labels:
        app.kubernetes.io/name: thu-patch
    spec:
      nodeSelector:
        kubernetes.io/hostname: lab-k8s-worker2
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh", "-c", "trap 'exit 0' TERM; sleep 3600 & wait"]
```

```yaml
# ~/lab-work/4a/patch-them-container.yaml
spec:
  template:
    spec:
      containers:
      - name: phu
        image: busybox:1.37
        command: ["sh", "-c", "trap 'exit 0' TERM; sleep 3600 & wait"]
```

Ý chính cho ADR:

1. Còn 3. Manifest của `web` khai `replicas: 3`, nên `replicas` nằm trong file và trong
   `last-applied-configuration`; `apply` đặt lại field có trong file mà giá trị khác cấu hình thực tế
   (bài 319; bài 63, mục *Replicas*). Thao tác cứu tải bị xóa âm thầm bởi một lần triển khai không
   liên quan. Lab 11b: HPA ghi `replicas` liên tục; còn `replicas` trong manifest thì mỗi `deploy.sh`
   kéo về 3, đánh nhau với HPA — nên phải bỏ nó khỏi file.
2. Annotation hotfix không có trong file, không có trong `last-applied-configuration`: dòng cuối của
   bảng merge — không làm gì, giữ giá trị thực tế. `diff` chỉ hiển thị thay đổi mà `apply` **sẽ** làm,
   nên nó không thấy thứ `apply` để yên. Bắt được bằng cơ chế mà dự án đã có: mỗi sprint mở bằng
   `teardown.sh` + `deploy.sh` — dựng lại chỉ từ repo, mọi trôi tay biến mất; muốn gỡ tại chỗ thì đặt
   field đó thành `null` (bài 319).
3. Hành vi do field quyết định: `containers` có `patchStrategy: merge` với khóa `name` nên strategic
   merge patch trộn; JSON merge patch (RFC 7386) không biết khóa nào, thay cả list (bài 324). `web` sẽ
   chỉ còn đúng container trong file patch — mất `nginx`, Pod mới không phục vụ được, rollout kẹt (và
   `maxUnavailable: 0` giữ bản cũ sống).
4. Được trong sự cố: `kubectl scale` (tạm, biết rằng `deploy.sh` kế tiếp ghi đè), `rollout undo`
   (B6). Sau đó: đưa thay đổi vào repo hoặc chạy `deploy.sh` để xóa nó, ghi vào postmortem. Cấm:
   `kubectl patch`, `edit`, `set`, `annotate` lên Pod template của `web`, `api` — chúng để lại trôi mà
   `diff` không thấy; `replace --force` — xóa rồi tạo lại, gây gián đoạn và mất
   `last-applied-configuration` (bài 61, 319).

</details>

## B9. [BS-4A-09] Điều tra: xóa theo tầng và nhận lại ReplicaSet

**Loại:** Điều tra — phần A trên `web` thật, phần B trên Deployment thử nghiệm.

**Bối cảnh:** bài 63 nói selector của Deployment bất biến: muốn đổi thì xóa Deployment rồi tạo lại,
và dùng `--cascade=orphan` nếu cần Pod tiếp tục chạy. Lab 5a có thể phải làm đúng việc đó với `web`.
Trước hôm đó, đội tập trên `web` thật với chính manifest hiện tại: xóa Deployment mà không đụng Pod,
rồi để Deployment mới nhận lại mọi thứ.

**Yêu cầu:**

1. **Phần A.** Chứng minh `ownerReferences` của Pod `web` trỏ ReplicaSet, với `controller` và
   `blockOwnerDeletion` đều `true`. Xóa Deployment `web` kiểu `orphan`: Deployment mất, ReplicaSet và
   ba Pod còn nguyên, ReplicaSet không còn owner, trang vẫn được phục vụ. Trong lúc không có
   Deployment, xóa một Pod `web`: chứng minh vẫn có Pod khác thay vào.
2. Chạy `./scripts/deploy.sh`: Deployment mới (UID mới) nhận lại ReplicaSet cũ — cùng tên, không tạo
   ReplicaSet mới, không Pod nào bị thay; lịch sử phát hành còn nguyên.
3. **Phần B.** Trên Deployment thử nghiệm `thu-xoa` (`busybox`, 2 bản sao, ghim `lab-k8s-worker2`, Pod
   có `preStop` giữ 15 giây): xóa kiểu `foreground` — Deployment còn đó với finalizer
   `foregroundDeletion` và `deletionTimestamp` cho tới khi Pod đi hết. Tạo lại, xóa **không** chỉ định
   kiểu: Deployment mất ngay, Pod còn lại chờ garbage collector. Cả hai lần cuối cùng đều sạch.

**Ràng buộc:** phần A chỉ dùng `--cascade=orphan` trên Deployment `web`; không xóa ReplicaSet hay Pod
bằng tay ngoài bước xóa một Pod của yêu cầu 1. Không dùng `--grace-period=0`.

**Đọc lại:**

- Bài [260](../260-use-cascading-deletion-vi.md):
  [*Kiểm tra owner reference trên các pod của bạn*](../260-use-cascading-deletion-vi.md#kiểm-tra-owner-reference-trên-các-pod-của-bạn-check-owner-references-on-your-pods),
  [*foreground*](../260-use-cascading-deletion-vi.md#use-foreground-cascading-deletion),
  [*background*](../260-use-cascading-deletion-vi.md#use-background-cascading-deletion),
  [*orphan*](../260-use-cascading-deletion-vi.md#set-orphan-deletion-policy).
- Bài [63](../63-deployment-vi.md): đoạn mở đầu (xóa Deployment hiện có và tiếp nhận tài nguyên của nó
  bằng Deployment mới), [*Cập nhật label selector*](../63-deployment-vi.md#label-selector-updates).
- Bài [30](../30-owners-dependents-vi.md) và [36](../36-garbage-collection-vi.md) đã đọc ở giai đoạn 1c.

**Bàn giao:** `docs/adr/NNNN-xoa-theo-tang.md`; `~/lab-evidence/4a/b9-*.txt`.

**Nghiệm thu:**

- `PASS:` Pod `web` trỏ ReplicaSet, `controller` và `blockOwnerDeletion` là `true`.
- `PASS:` sau `orphan`: không còn Deployment `web`, ReplicaSet không owner, ba Pod giữ nguyên UID,
  trang vẫn trả `/healthz`.
- `PASS:` không có Deployment, Pod bị xóa vẫn được ReplicaSet thay.
- `PASS:` sau `deploy.sh`: Deployment UID mới; ReplicaSet cũ trỏ về nó; không ReplicaSet mới, không
  Pod nào bị thay; lịch sử phát hành giữ nguyên.
- `PASS:` `foreground`: thấy `foregroundDeletion` và `deletionTimestamp`; xong thì sạch.
- `PASS:` không chỉ định kiểu: Deployment mất trước, Pod còn lại một lúc; xong thì sạch.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-xoa-theo-tang.md`:

1. Trong khoảng không có Deployment `web`, ai giữ số bản sao? Phát hành được không? PDB `web` còn tính
   được ngân sách không, nhờ đâu?
2. Deployment mới không tạo Pod nào. Nhờ cơ chế nào nó nhận ra ReplicaSet cũ là "của mình" và là bản
   mới nhất? Nếu manifest trong repo khác template đang chạy thì chuyện gì xảy ra?
3. Lab 5a muốn thêm một label vào selector của `web`. Dùng bài này và bài 63 mô tả quy trình, và nói
   ReplicaSet cũ sẽ ra sao.
4. `teardown.sh` xóa namespace — các object bên trong đi theo kiểu nào? Kiểu nào hợp với một script
   cần chắc chắn mọi thứ đã biến mất trước bước kế tiếp?

<details>
<summary>Gợi ý</summary>

1. `kubectl delete deployment <tên> --cascade=orphan|foreground|background`. Với `foreground`, thêm
   `--wait=false` để kịp đọc object trong lúc nó đang bị giữ.
2. Garbage collector gỡ `ownerReferences` ở chế độ nền: chờ vài giây rồi mới kiểm.
3. `thu-xoa.yaml` là `thu-patch.yaml` của B8 đổi tên và label, thêm
   `terminationGracePeriodSeconds: 40` và `preStop` `sh -c "sleep 15"`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã tự viết thu-xoa.yaml; script này chính là cuộc điều tra</summary>

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
NS=bookstore; EV=~/lab-evidence/4a; W=~/lab-work/4a
WEB='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web'
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
rs_chay() { kubectl -n "$NS" get rs -l "$WEB" -o jsonpath='{range .items[*]}{.metadata.name} {.spec.replicas}{"\n"}{end}' | awk '$2 > 0 {print $1}'; }
uids() { kubectl -n "$NS" get pods -l "$WEB" -o jsonpath='{range .items[*]}{.metadata.uid}{"\n"}{end}' | sort; }
lich_su() { kubectl -n "$NS" rollout history deployment/web | awk 'NF' | tail -n +3; }

# A. orphan rồi nhận lại
kubectl -n "$NS" get pods -l "$WEB" -o jsonpath='{range .items[*]}{.metadata.name} -> {.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name} controller={.metadata.ownerReferences[0].controller} block={.metadata.ownerReferences[0].blockOwnerDeletion}{"\n"}{end}' \
  > "$EV/b9-ownerrefs.txt"
[ "$(grep -c 'ReplicaSet/web-.* controller=true block=true' "$EV/b9-ownerrefs.txt")" -eq 3 ] \
  && echo 'PASS: Pod web tro ReplicaSet, controller va blockOwnerDeletion la true'
D0="$(jp deployment/web '{.metadata.uid}')"; RS0="$(rs_chay)"; U0="$(uids)"
NRS0="$(kubectl -n "$NS" get rs -l "$WEB" --no-headers | wc -l)"; lich_su > "$EV/b9-history-truoc.txt"

kubectl -n "$NS" delete deployment web --cascade=orphan --wait=true
for i in $(seq 1 30); do [ -z "$(jp "replicaset/$RS0" '{.metadata.ownerReferences}')" ] && break; sleep 2; done
{ ! kubectl -n "$NS" get deployment web >/dev/null 2>&1 && [ -z "$(jp "replicaset/$RS0" '{.metadata.ownerReferences}')" ] \
  && [ "$(uids)" = "$U0" ]; } && echo "PASS: orphan: Deployment mat, $RS0 mo coi, ba Pod giu nguyen UID"
[ "$(kubectl -n "$NS" exec "$(kubectl -n "$NS" get pods -l "$WEB" -o jsonpath='{.items[0].metadata.name}')" \
  -c status-writer -- wget -qO- http://127.0.0.1:8080/healthz)" = 'ok' ] && echo 'PASS: trang van duoc phuc vu'
V="$(kubectl -n "$NS" get pods -l "$WEB" --field-selector spec.nodeName=lab-k8s-worker2 -o jsonpath='{.items[0].metadata.name}')"
[ -n "$V" ] || V="$(kubectl -n "$NS" get pods -l "$WEB" -o jsonpath='{.items[0].metadata.name}')"
kubectl -n "$NS" delete pod "$V" --wait=true
for i in $(seq 1 60); do [ "$(kubectl -n "$NS" get pods -l "$WEB" --no-headers | wc -l)" -eq 3 ] && break; sleep 2; done
kubectl -n "$NS" wait --for=condition=Ready pod -l "$WEB" --timeout=180s >/dev/null \
  && ! kubectl -n "$NS" get pod "$V" >/dev/null 2>&1 && echo 'PASS: khong co Deployment, ReplicaSet van thay Pod bi mat'
U1="$(uids)"

./scripts/deploy.sh > "$EV/b9-deploy.txt" 2>&1
D1="$(jp deployment/web '{.metadata.uid}')"
{ [ -n "$D1" ] && [ "$D1" != "$D0" ] && [ "$(jp "replicaset/$RS0" '{.metadata.ownerReferences[0].uid}')" = "$D1" ] \
  && [ "$(rs_chay)" = "$RS0" ] && [ "$(kubectl -n "$NS" get rs -l "$WEB" --no-headers | wc -l)" = "$NRS0" ] \
  && [ "$(uids)" = "$U1" ]; } && echo "PASS: Deployment moi nhan lai $RS0; khong ReplicaSet moi, khong Pod nao bi thay"
lich_su > "$EV/b9-history-sau.txt"
diff -q "$EV/b9-history-truoc.txt" "$EV/b9-history-sau.txt" >/dev/null && echo 'PASS: lich su phat hanh con nguyen'

# B. foreground và background trên thu-xoa
X='app.kubernetes.io/name=thu-xoa'
tao() { kubectl apply -f "$W/thu-xoa.yaml" >/dev/null; kubectl -n "$NS" rollout status deployment/thu-xoa --timeout=180s >/dev/null; }
sach() { for i in $(seq 1 90); do [ "$(sl kubectl -n "$NS" get deploy,rs,pods -l "$X" --no-headers)" -eq 0 ] && return 0; sleep 2; done; return 1; }
tao
kubectl -n "$NS" delete deployment thu-xoa --cascade=foreground --wait=false
for i in $(seq 1 30); do FIN="$(jp deployment/thu-xoa '{.metadata.finalizers}')"; [ -n "$FIN" ] && break; sleep 1; done
{ echo "$FIN" | grep -q foregroundDeletion && [ -n "$(jp deployment/thu-xoa '{.metadata.deletionTimestamp}')" ]; } \
  && echo 'PASS: foreground: Deployment con do finalizer foregroundDeletion, co deletionTimestamp'
sach && echo 'PASS: foreground xong: khong con Deployment, ReplicaSet, Pod'
tao
kubectl -n "$NS" delete deployment thu-xoa --wait=true
{ ! kubectl -n "$NS" get deployment thu-xoa >/dev/null 2>&1 && [ "$(kubectl -n "$NS" get pods -l "$X" --no-headers 2>/dev/null | wc -l)" -gt 0 ]; } \
  && echo 'PASS: mac dinh la background: Deployment mat truoc, Pod con lai cho garbage collector'
sach && echo 'PASS: garbage collector don sach o nen'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính</summary>

`thu-xoa.yaml`: như `thu-patch.yaml` của B8 với tên và label `thu-xoa`, thêm vào `template.spec`
`terminationGracePeriodSeconds: 40` và vào container
`lifecycle: {preStop: {exec: {command: ["sh", "-c", "sleep 15"]}}}`.

1. ReplicaSet vẫn sống và vẫn chạy vòng lặp của nó (B1), nên số bản sao được giữ. Không phát hành
   được: không còn ai tạo ReplicaSet mới cho template mới. PDB `web` vẫn tính được: bài 339 lấy số
   bản sao mong muốn từ controller của Pod — giờ là ReplicaSet, cũng có subresource `scale`.
2. Deployment mới có selector khớp ReplicaSet mồ côi nên nhận nuôi nó (bài 63, đoạn mở đầu: Deployment
   mới tiếp nhận tài nguyên của Deployment cũ); ReplicaSet nào có template giống Pod template hiện tại
   (bỏ qua `pod-template-hash`) là ReplicaSet mới nhất, nên không có gì để rollout. Lịch sử nằm trong
   các ReplicaSet đó nên đi theo. Manifest khác template đang chạy thì đó là một rollout bình thường:
   ReplicaSet mới được tạo, ReplicaSet cũ được hạ dần.
3. Bài 63, *Cập nhật label selector*: thêm label vào selector thì phải thêm vào template; xóa
   Deployment cũ bằng `--cascade=orphan`, apply Deployment mới. Selector mới hẹp hơn không khớp Pod cũ
   (thiếu label mới): ReplicaSet cũ bị mồ côi và một ReplicaSet mới được tạo — phải tự dọn ReplicaSet
   cũ sau khi Pod mới sẵn sàng.
4. Background — mặc định của bài 260. Script cần chắc mọi thứ đã biến mất thì chờ đúng thứ nó cần:
   `foreground` giữ owner tới khi dependent đi hết, nên `--wait` trên owner là đủ. `teardown.sh` thay
   vào đó tự đếm object còn sót.

</details>

## B10. [BS-4A-10] Canary thủ công cho `web` — dựng cấu trúc

**Loại:** Điều tra — trên tài nguyên thử nghiệm.

**Bối cảnh:** bản `0.3.0` của trang sẽ đổi cả bố cục. Chủ hiệu sách muốn một phần nhỏ người xem thấy
nó trước. Bài 61 mô tả canary thủ công: hai bản phát hành khác nhau ở **một** label, tỷ lệ do số bản
sao quyết định, một Service chọn phần label chung. Chưa có Service (Lab 5a), nên sprint này dựng và
kiểm đúng phần **cấu trúc**, trên tài nguyên thử nghiệm, rồi dọn sạch — canary không được nằm lại
trong repo.

**Yêu cầu:**

1. Hai Deployment thử nghiệm ghim `lab-k8s-worker2`, nginx dùng cấu hình site của `web-config`, mỗi
   cái một file ở hai thư mục con khác nhau dưới `~/lab-work/4a/canary/`: `thu-stable` 3 bản sao,
   `version` `0.2.0`; `thu-canary` 1 bản sao, `version` `0.3.0`. Chung label `thu-nghiem: canary`,
   khác nhau ở label `track` (`stable`, `canary`).
2. Chứng minh `kubectl apply` thư mục `canary/` không đi vào thư mục con nếu không yêu cầu, và đi vào
   khi yêu cầu.
3. Chứng minh: một selector chung bắt cả 4 Pod, tỷ lệ phiên bản 3:1; mỗi Deployment chỉ chọn Pod
   `track` của mình.
4. Đổi tỷ lệ thành 1:1 chỉ bằng số bản sao. Rồi "thăng hạng": bản `stable` lên `0.3.0`, gỡ `canary`
   bằng selector chứ không bằng tên; selector chung còn 3 Pod, tất cả `0.3.0`.
5. Xóa tất cả bằng một selector; repo và cluster không còn dấu vết canary.

**Ràng buộc:** không gắn label `app.kubernetes.io/part-of` hay `component` cho tài nguyên canary;
không đặt gì vào `~/bookstore/k8s/`. Không tạo Service.

**Đọc lại:**

- Bài [61](../61-management-vi.md): [*Triển khai canary*](../61-management-vi.md#canary-deployments),
  [*Thao tác hàng loạt*](../61-management-vi.md#thao-tác-hàng-loạt-trong-kubectl-bulk-operations-in-kubectl),
  [*Thao tác đệ quy trên file cục bộ*](../61-management-vi.md#thao-tác-đệ-quy-trên-file-cục-bộ-recursive-operations-on-local-files).
- Bài [63](../63-deployment-vi.md): [*Deployment kiểu canary*](../63-deployment-vi.md#canary-deployment),
  ghi chú trong [*Selector*](../63-deployment-vi.md#selector) về selector chồng lấn.
- Bài [70](../70-replicationcontroller-vi.md) — mục *Nhiều track phát hành*: cùng ý tưởng ở thế hệ cũ.

**Bàn giao:** `docs/adr/NNNN-canary.md`; `~/lab-evidence/4a/b10-*.txt`.

**Nghiệm thu:**

- `PASS:` `apply` thư mục không `-R` báo lỗi không có tài nguyên; có `-R` tạo cả hai Deployment.
- `PASS:` selector chung bắt 4 Pod: `stable` 3, `canary` 1; phiên bản `0.2.0` ×3, `0.3.0` ×1.
- `PASS:` selector của mỗi Deployment chứa `track` của chính nó.
- `PASS:` tỷ lệ 1:1 chỉ bằng scale; thăng hạng xong còn 3 Pod, tất cả `0.3.0`.
- `PASS:` một lệnh xóa theo selector dọn sạch; repo không có `track:` hay `thu-nghiem`; cluster chỉ
  còn Deployment `web`, `api` mang label dự án.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-canary.md`:

1. Tỷ lệ 3:1 ở đây là tỷ lệ của cái gì? Vì sao nó **chưa** phải 25% lưu lượng? Lab 5a cần thêm gì?
2. Vì sao mỗi Deployment phải có `track` trong selector? Nếu dựng canary thật cho `web` với label
   `part-of=bookstore` + `component=web` thì nó đụng gì với Deployment `web` và PDB `web`, khi
   selector của `web` bất biến và chỉ gồm hai label đó? Ghi lại các lựa chọn cho Lab 5a.
3. Vì sao canary không được nằm lại trong repo cuối sprint?

<details>
<summary>Gợi ý</summary>

1. Mỗi file là `drill-web.yaml` của B6 đổi tên, đổi label (bỏ `app.kubernetes.io/name`, dùng
   `thu-nghiem` và `track`, thêm `app.kubernetes.io/version` vào template), đổi số bản sao.
2. Đếm Pod theo phiên bản:
   `kubectl -n bookstore get pods -l thu-nghiem=canary -L app.kubernetes.io/version,track`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã tự viết hai file trong ~/lab-work/4a/canary/; script này chính là cuộc điều tra</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4a; W=~/lab-work/4a
C='thu-nghiem=canary'
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
dem() { kubectl -n "$NS" get pods -l "$1" --no-headers 2>/dev/null | wc -l; }
ver() { kubectl -n "$NS" get pods -l "$1" -o jsonpath='{range .items[*]}{.metadata.labels.app\.kubernetes\.io/version}{"\n"}{end}' \
  | sort | uniq -c | awk '{printf "%sx%s ", $2, $1}'; }

kubectl apply -f "$W/canary" > "$EV/b10-khong-R.txt" 2>&1 && echo 'FAIL: apply thu muc long nhau chay duoc khi khong co -R' \
  || { grep -q 'must provide one or more resources' "$EV/b10-khong-R.txt" && echo 'PASS: khong co -R thi dung o cap thu muc dau'; }
kubectl apply -R -f "$W/canary" > "$EV/b10-apply-R.txt"
[ "$(grep -c 'deployment.apps/' "$EV/b10-apply-R.txt")" -eq 2 ] && echo 'PASS: -R xu ly ca hai thu muc con'
kubectl -n "$NS" rollout status deployment/thu-stable --timeout=180s >/dev/null
kubectl -n "$NS" rollout status deployment/thu-canary --timeout=180s >/dev/null

{ [ "$(dem "$C")" -eq 4 ] && [ "$(dem "$C,track=stable")" -eq 3 ] && [ "$(dem "$C,track=canary")" -eq 1 ] \
  && [ "$(ver "$C")" = '0.2.0x3 0.3.0x1 ' ]; } && echo 'PASS: selector chung bat 4 Pod, 0.2.0 x3 va 0.3.0 x1'
kubectl -n "$NS" get pods -l "$C" -L app.kubernetes.io/version,track > "$EV/b10-pods.txt"
{ [ "$(jp deployment/thu-stable '{.spec.selector.matchLabels.track}')" = 'stable' ] \
  && [ "$(jp deployment/thu-canary '{.spec.selector.matchLabels.track}')" = 'canary' ]; } \
  && echo 'PASS: moi Deployment chi chon Pod track cua minh'

kubectl -n "$NS" scale deployment/thu-canary --replicas=3
kubectl -n "$NS" rollout status deployment/thu-canary --timeout=180s >/dev/null
[ "$(ver "$C")" = '0.2.0x3 0.3.0x3 ' ] && echo 'PASS: ty le 1:1 chi bang so ban sao'
kubectl -n "$NS" patch deployment thu-stable -p '{"spec":{"template":{"metadata":{"labels":{"app.kubernetes.io/version":"0.3.0"}}}}}' >/dev/null
kubectl -n "$NS" rollout status deployment/thu-stable --timeout=180s >/dev/null
kubectl -n "$NS" delete deployment -l "$C,track=canary" --wait=true
for i in $(seq 1 60); do [ "$(dem "$C")" -eq 3 ] && break; sleep 2; done
[ "$(ver "$C")" = '0.3.0x3 ' ] && echo 'PASS: thang hang: con 3 Pod, tat ca 0.3.0'

kubectl -n "$NS" delete deployment -l "$C" --wait=true
for i in $(seq 1 60); do [ "$(dem "$C")" -eq 0 ] && break; sleep 2; done
{ [ "$(dem "$C")" -eq 0 ] && ! grep -rqE 'track:|thu-nghiem' ~/bookstore/k8s \
  && [ "$(kubectl -n "$NS" get deployments -l app.kubernetes.io/part-of=bookstore -o jsonpath='{.items[*].metadata.name}')" = 'api web' ]; } \
  && echo 'PASS: mot selector don sach canary; repo va cluster khong con dau vet'
rm -rf "$W/canary"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/4a/canary/stable/deployment.yaml — phần khác drill-web.yaml của B6; khối
# template.spec (nodeSelector, volume, container nginx) giữ nguyên như drill-web.
# canary/canary/deployment.yaml giống hệt: tên thu-canary, track canary, replicas 1, version "0.3.0"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thu-stable
  namespace: bookstore
  labels:
    thu-nghiem: canary
    track: stable
spec:
  replicas: 3
  selector:
    matchLabels:
      thu-nghiem: canary
      track: stable
  template:
    metadata:
      labels:
        thu-nghiem: canary
        track: stable
        app.kubernetes.io/version: "0.2.0"
    # spec: như drill-web.yaml
```

Ý chính cho ADR:

1. Tỷ lệ **số Pod**. Lưu lượng chỉ chia theo tỷ lệ đó khi có một Service chọn phần label chung và
   phân phối request giữa các Pod (bài 61). Hôm nay không có gì gửi request qua selector chung — phần
   chia lưu lượng thật là phần hoãn sang Lab 5a.
2. Không có `track` trong selector thì hai Deployment chọn Pod của nhau — bài 63 cảnh báo controller
   chồng selector sẽ tranh chấp. Canary thật của `web` mang `part-of` + `component=web` (Service của
   Lab 5a chọn đúng cặp đó) sẽ khớp selector của Deployment `web`, và PDB `web` sẽ đếm cả Pod canary.
   Lựa chọn cho Lab 5a: thêm một label như `track: stable` vào selector của `web` — phải xóa
   `--cascade=orphan` rồi tạo lại (B9); hoặc chấp nhận chồng selector có kiểm soát và ghi rõ rủi ro.
   Đây là quyết định thiết kế của Lab 5a, không làm ở sprint này.
3. Repo là nguồn sự thật (mục 5.3): canary trong repo nghĩa là mọi `deploy.sh` về sau đều dựng nó, và
   kiểm thử khói của từng sprint phải biết tới nó. Canary là một trạng thái tạm thời của một lần phát
   hành, không phải thành phần của hệ thống.

</details>

## B11. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 4, rồi trả cluster về đúng
`01-cluster-ready`.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 4a — kiểm thử khói **tích lũy**, chép nguyên văn. Tên
   Pod `web`, `api` không còn cố định: script lấy Pod theo label và gọi trang qua `deploy/web`.

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 4a: Deployment web (3 bản sao), Deployment api (2 bản sao),
# Pod trần db. Kiểm mọi hợp đồng còn hiệu lực tới sprint 4: cấu hình, bí mật (3b); tài nguyên, PDB
# (3c); controller, phát hành có lý do, mọi bản sao web phục vụ đúng nội dung và phiên bản (4a).
# Tên Pod web, api không cố định: lấy theo label. Script không in mật khẩu.
set -u
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd "$(dirname "$0")/.."
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
WEB="$SEL,app.kubernetes.io/component=web"
API="$SEL,app.kubernetes.io/component=api"
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
names() { k get pods -l "$1" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' 2>/dev/null; }
in_pod() { k exec "$1" -c status-writer -- wget -qO- "$2" 2>/dev/null; }
get() { k exec deploy/web -c status-writer -- wget -qO- "$1" 2>/dev/null; }
sql() { k exec db -c postgres -- psql -U postgres -d bookstore -tAc "$1" 2>/dev/null; }
sec() { k get secret db-credentials -o go-template="{{index .data \"$1\"}}" 2>/dev/null | base64 -d 2>/dev/null; }

# 0. Chờ Pod đang kết thúc của lần phát hành trước biến mất
for X in "$WEB:3" "$API:2"; do
  for i in $(seq 1 45); do [ "$(sl k get pods -l "${X%:*}" --no-headers)" -eq "${X##*:}" ] && break; sleep 2; done
done

# 1. Namespace và object cấp cluster
[ "$(kubectl get namespace "$NS" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = 'bookstore' ] \
  && pass 'namespace bookstore mang label du an' || fail 'namespace bookstore thieu hoac thieu label'
[ -n "$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)" ] \
  && pass 'co PriorityClass bookstore-critical' || fail 'thieu PriorityClass bookstore-critical'

# 2. Controller (4a): web 3, api 2; db vẫn là Pod trần (TD-01 còn phần db)
for X in web:3 api:2; do
  C="${X%%:*}"; W="${X##*:}"
  [ "$(jp "deployment/$C" '{.status.replicas}/{.status.updatedReplicas}/{.status.availableReplicas}')" = "$W/$W/$W" ] \
    && pass "Deployment $C: $W ban sao san sang, cung template moi nhat" || fail "Deployment $C chua dat $W ban sao moi nhat san sang"
  [ "$(k get deployment "$C" -o go-template='{{range $k, $v := .spec.selector.matchLabels}}{{$k}}={{$v}} {{end}}' 2>/dev/null)" \
    = "app.kubernetes.io/component=$C app.kubernetes.io/part-of=bookstore " ] \
    && pass "selector cua $C dung hop dong" || fail "selector cua $C sai hop dong"
  BAD=0
  for P in $(names "$SEL,app.kubernetes.io/component=$C"); do
    RS="$(jp "pod/$P" '{.metadata.ownerReferences[0].name}')"
    [ "$(jp "replicaset/$RS" '{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}')" = "Deployment/$C" ] || BAD=1
  done
  [ "$BAD" -eq 0 ] && pass "moi Pod $C -> ReplicaSet -> Deployment $C" || fail "chuoi so huu cua $C sai"
  { [ -n "$(jp "deployment/$C" '{.metadata.annotations.kubernetes\.io/change-cause}')" ] \
    && ! k rollout history "deployment/$C" 2>/dev/null | awk 'NF' | tail -1 | grep -q '<none>'; } \
    && pass "ban phat hanh hien tai cua $C co ly do" || fail "ban phat hanh hien tai cua $C thieu ly do"
done
BARE="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null \
  | sed -n 's/:$//p' | tr '\n' ' ')"
[ "$BARE" = 'db ' ] && pass 'Pod tran duy nhat la db (TD-01 con phan db)' || fail "Pod tran khong dung: ${BARE:-khong co}"
[ "$(jp pod/db '{.status.conditions[?(@.type=="Ready")].status}')" = 'True' ] && pass 'Pod db Ready' || fail 'Pod db chua Ready'
DEPS="$(k get deployments -l "$SEL" -o jsonpath='{.items[*].metadata.name}' 2>/dev/null)"
{ [ "$DEPS" = 'api web' ] && [ -z "$(k get pods -l track --no-headers 2>/dev/null)" ]; } \
  && pass 'chi co Deployment web, api; khong con canary' || fail "Deployment ngoai hop dong: $DEPS"

# 3. Tài nguyên (3c) và PDB theo controller (3c, 4a)
MISS="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{range .spec.initContainers[?(@.restartPolicy=="Always")]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null \
  | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container chinh va sidecar co requests va limits' || fail "thieu tai nguyen: $(echo $MISS)"
[ "$(jp pod/db '{.status.qosClass}')" = 'Guaranteed' ] && pass 'db QoS Guaranteed' || fail 'db khong Guaranteed'
for X in web:3 api:2; do
  C="${X%%:*}"; W="${X##*:}"
  [ "$(jp "pdb/$C" '{.spec.maxUnavailable}/{.spec.minAvailable}')" = '1/' ] \
    && pass "PDB $C khai maxUnavailable 1, khong minAvailable" || fail "PDB $C khong theo hop dong maxUnavailable 1"
  for i in $(seq 1 15); do
    E="$(jp "pdb/$C" '{.status.expectedPods}')"; A="$(jp "pdb/$C" '{.status.disruptionsAllowed}')"
    [ "$E" = "$W" ] && [ "${A:-0}" -ge 1 ] && break; sleep 2
  done
  { [ "$E" = "$W" ] && [ "${A:-0}" -ge 1 ]; } && pass "PDB $C tinh tren $W ban sao, cho phep $A gian doan" \
    || fail "PDB $C: expectedPods=${E:-?} disruptionsAllowed=${A:-?}"
done

# 4. Dữ liệu và api (3a, TD-03)
N="$(sql 'SELECT count(*) FROM api.books')"
[ "${N:-0}" -ge 5 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach"
DB_IP="$(jp pod/db '{.status.podIP}')"
k get deployment api -o yaml 2>/dev/null | grep -qF "${DB_IP:-khong-co-ip}" \
  && pass 'Deployment api tro IP hien tai cua db (TD-03)' || fail 'Deployment api khong tro IP hien tai cua db'
BAD=0; NA=0
for IP in $(k get pods -l "$API" -o jsonpath='{.items[*].status.podIP}' 2>/dev/null); do
  NA=$((NA + 1)); [ "$(get "http://$IP:3000/books" | grep -o '"title"' | wc -l)" -ge 5 ] || BAD=1
done
{ [ "$NA" -eq 2 ] && [ "$BAD" -eq 0 ]; } && pass 'ca hai ban sao api tra /books' || fail 'co ban sao api khong tra /books'

# 5. web: mọi bản sao phục vụ đúng danh tính, phiên bản và nội dung hiện hành (3a, 3b, 4a)
[ "$(get http://127.0.0.1:8080/healthz)" = 'ok' ] && pass 'web /healthz tra ok' || fail 'web /healthz sai'
VER="$(jp deployment/web '{.spec.template.metadata.labels.app\.kubernetes\.io/version}')"
TPL="$(jp configmap/web-config '{.data.index\.html\.tpl}')"
BAD_ID=0; BAD_VER=0; BAD_PAGE=0; NW=0
for P in $(names "$WEB"); do
  NW=$((NW + 1)); NODE="$(jp "pod/$P" '{.spec.nodeName}')"
  S="$(in_pod "$P" http://127.0.0.1:8080/status.json)"
  { echo "$S" | grep -Eq "\"pod\" *: *\"$P\"" && echo "$S" | grep -Eq "\"node\" *: *\"$NODE\""; } || BAD_ID=1
  echo "$S" | grep -Eq "\"version\" *: *\"$VER\"" || BAD_VER=1
  PAGE="$(in_pod "$P" http://127.0.0.1:8080/)"
  { echo "$PAGE" | grep -q 'Bookstore' && echo "$PAGE" | grep -qF "$P"; } || BAD_PAGE=1
  # Mỗi dòng của trang mẫu không chứa chỗ giữ dạng __TEN__ (Lab 3b) phải có trong trang đang phục vụ.
  while IFS= read -r L; do
    case "$L" in *__*) continue ;; esac
    [ -n "${L//[[:space:]]/}" ] || continue
    printf '%s\n' "$PAGE" | grep -qF -- "$L" || { BAD_PAGE=1; break; }
  done <<< "$TPL"
done
{ [ "$NW" -eq 3 ] && [ "$BAD_ID" -eq 0 ]; } && pass 'moi ban sao web bao dung ten Pod va node cua no' || fail 'status.json sai ten Pod hoac node'
[ "$BAD_VER" -eq 0 ] && pass "moi ban sao web bao version $VER cua Pod template" || fail 'co ban sao web bao sai version'
[ "$BAD_PAGE" -eq 0 ] && pass 'moi ban sao web phuc vu dung trang mau hien hanh' \
  || fail 'co ban sao web phuc vu trang cu: cau hinh doi ma chua phat hanh?'

# 6. Cấu hình và bí mật (3b)
for O in configmap/web-config configmap/api-config configmap/db-init secret/db-credentials; do
  [ "$(jp "$O" '{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ] && pass "$O mang label du an" || fail "$O thieu hoac thieu label"
done
[ "$(jp secret/db-credentials '{.type}')" = 'Opaque' ] && pass 'db-credentials loai Opaque' || fail 'db-credentials sai loai'
PGPW="$(sec postgres-password)"; AUPW="$(sec authenticator-password)"
if [ -z "$PGPW" ] || [ -z "$AUPW" ]; then
  fail 'khong doc duoc hai key cua db-credentials'
else
  login() { k exec -i db -c postgres -- env H="$DB_IP" U="$1" \
    sh -c 'PGPASSWORD="$(cat)" psql -h "$H" -U "$U" -d bookstore -tAc "SELECT current_user"' 2>/dev/null; }
  [ "$(printf '%s' "$PGPW" | login postgres)" = 'postgres' ] \
    && pass 'superuser dang nhap qua IP Pod bang mat khau trong Secret' || fail 'superuser khong dang nhap duoc'
  [ "$(printf '%s' "$AUPW" | login authenticator)" = 'authenticator' ] \
    && pass 'authenticator dang nhap bang mat khau trong Secret' || fail 'authenticator khong dang nhap duoc'
  k get pods,deployments,replicasets -o yaml 2>/dev/null | grep -qF -e "$PGPW" -e "$AUPW" \
    && fail 'mat khau xuat hien trong spec' || pass 'spec cua Pod, Deployment, ReplicaSet khong chua mat khau'
  git grep --untracked -qF -e "$PGPW" -e "$AUPW" && fail 'mat khau nam trong file se vao git' || pass 'repo khong chua mat khau'
fi
{ [ "$(git ls-files secrets)" = 'secrets/.gitkeep' ] && git check-ignore -q secrets/db.env; } \
  && pass 'secrets/db.env bi git bo qua' || fail 'secrets/ khong duoc .gitignore bao ve'
ENVS=" $(jp pod/db '{.spec.containers[?(@.name=="postgres")].env[*].name}') "
case "$ENVS" in
  *' POSTGRES_PASSWORD '*)      fail 'db van nhan mat khau superuser qua bien POSTGRES_PASSWORD' ;;
  *' POSTGRES_PASSWORD_FILE '*) pass 'db doc mat khau superuser qua file' ;;
  *)                            fail 'db khong khai POSTGRES_PASSWORD_FILE' ;;
esac
[ "$(sql 'SHOW log_connections')" = 'on' ] && pass 'db ghi log ket noi' || fail 'db chua bat log_connections'

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

2. Cập nhật `docs/tech-debt.md`: `TD-01` ghi **trả một phần ở 4a** (`web`, `api`; bằng chứng
   `~/lab-evidence/4a/b2-chuoi-so-huu.txt`), còn `db` — trả ở 4b. Ghi chú ở `TD-03`: mỗi lần `db` đổi
   IP, Deployment `api` sinh một revision mới.

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/4a/b11-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/4a/b11-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/4a/b11-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-4a: sprint 4 — Deployment web, api; phat hanh, rollback'
git tag lab-4a
git bundle create ~/bookstore-lab-4a.bundle --all && git bundle verify ~/bookstore-lab-4a.bundle
```

Chép `~/bookstore-lab-4a.bundle` về máy host bằng `scp` trước khi đi tiếp.

5. Dọn object thử nghiệm và trả cluster:

```bash
kubectl -n bookstore delete deployment,replicaset,pod -l '!app.kubernetes.io/part-of' --ignore-not-found --wait=true
rm -rf ~/lab-work/4a
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd ~/bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-4a' && echo 'PASS: HEAD mang tag lab-4a'
git bundle verify ~/bookstore-lab-4a.bundle >/dev/null 2>&1 && echo 'PASS: bundle hop le'
{ test ! -e k8s/app/web/pod.yaml && test -f k8s/app/web/deployment.yaml && test ! -e k8s/app/api/pod.yaml.tpl \
  && test -f k8s/app/api/deployment.yaml.tpl && test -f k8s/app/db/pod.yaml; } && echo 'PASS: bo cuc repo dung hang 4a cua muc 5.5'
! grep -rqE 'track:|thu-nghiem' k8s && echo 'PASS: repo khong chua canary'
grep -E 'TD-01' docs/tech-debt.md | grep -q '4a' && echo 'PASS: tech-debt.md ghi TD-01 tra mot phan o 4a'
BAD=0
for F in 'adr/*-replicaset.md' 'adr/*-web-deployment.md' 'adr/*-api-deployment.md' 'adr/*-chien-luoc-phat-hanh.md' \
         'adr/*-phat-hanh-noi-dung.md' 'adr/*-recreate-hay-rollingupdate.md' 'adr/*-troi-cau-hinh.md' \
         'adr/*-xoa-theo-tang.md' 'adr/*-canary.md' 'postmortem/4a-phat-hanh-hong.md' 'runbook-phat-hanh.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR, postmortem va runbook cua sprint 4'
{ [ "$(sl kubectl get namespace,priorityclass -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ] \
  && [ "$(sl kubectl get all -A -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an'
test ! -e ~/lab-work/4a && echo 'PASS: da don lab-work'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; chín dòng `PASS:` đầu của gate cuối
xuất hiện; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica; namespace `default` không có Pod. Thư mục `~/lab-evidence/4a/` **giữ lại**.
Cluster trở về `01-cluster-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 4a

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Ba thứ định nghĩa một ReplicaSet và vòng lặp của nó. Vì sao `thu-rs` nuốt Pod trần, còn
      ReplicaSet của Deployment `web` không nuốt `thu-web-tran` dù cùng label? Vì sao ReplicaSet nhận
      nuôi Pod cũ mà không cập nhật chúng — và đó là lý do của cái gì?
- [ ] Vẽ chuỗi sở hữu của một Pod `web` với UID; `pod-template-hash` sinh ra từ đâu và nằm ở ba chỗ
      nào?
- [ ] Thao tác nào sinh revision, thao tác nào không? Sửa trang mẫu trong `web-config` rồi chạy
      `deploy.sh` thì sao, và xóa một Pod sau đó thì sao? `UP-TO-DATE` nói gì và không nói gì?
- [ ] Cách đúng để phát hành nội dung mới ở sprint này, và vì sao `rollout undo` không lấy lại nội
      dung cũ. Lab 5a đổi câu chuyện đó thế nào?
- [ ] Chiến lược phát hành của `web`: giá trị từng trường, ngưỡng sẵn sàng và ngưỡng tổng bản sao, số
      đo của bạn ở B4. Quy tắc làm tròn khi khai bằng phần trăm; vì sao số Pod thô có lúc vượt ngưỡng.
- [ ] Kể lại B6: trường nào giữ dịch vụ sống, condition nào đổi, `deploy.sh` làm gì, Kubernetes làm gì
      sau deadline, và quy trình đưa cluster với repo về khớp sau `undo`.
- [ ] `Recreate` so với `RollingUpdate` bằng số đo B7; vì sao lời hứa của `Recreate` không áp dụng khi
      xóa tay — và vì thế `db` chờ StatefulSet.
- [ ] PDB `web`, `api` giờ tính ngân sách thế nào, vì sao hợp đồng chọn `maxUnavailable: 1` mà không
      chọn `minAvailable` bằng số bản sao (drain ở Lab 12)? Vì sao eviction thứ hai ngay sau eviction
      thứ nhất bị từ chối, còn `kubectl delete pod` thì không?
- [ ] `kubectl scale`, `kubectl patch` vào template, `kubectl apply` — thay đổi tay nào `deploy.sh`
      ghi đè, thay đổi nào trôi lại mà `kubectl diff` không thấy? Dự án bắt loại trôi đó bằng gì?
- [ ] Xóa Deployment `web` kiểu `orphan` rồi `deploy.sh`: ai giữ Pod trong khoảng trống, vì sao không
      Pod nào bị thay, lịch sử ở đâu? Foreground khác background ở output và thời điểm owner biến mất.
- [ ] Canary thủ công: label nào chung, label nào riêng, 3:1 là tỷ lệ của cái gì, vì sao chưa phải
      chia lưu lượng, và canary thật của `web` đụng gì với selector bất biến của `web`?
- [ ] `ReplicationController` khác ReplicaSet ở đúng chỗ nào; gặp nó trên cluster kế thừa thì làm gì?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- **Phần hoãn của sprint** (mục 1.1): canary chia lưu lượng thật (Lab 5a B7) và đo gián đoạn ở mức
  request (Lab 5a B3; `api` ở Lab 12 B6) — cần Service; `kubectl autoscale` và HPA làm chủ
  `replicas` — **nợ lab #1**, Lab 11b. Nợ lab của các sprint trước không đổi: phase `Unknown`
  (Lab 12), readiness loại Pod khỏi tập backend sẵn sàng nhận lưu lượng (Lab 5a), **nợ lab #6**
  (giai đoạn 22).
- **Nợ kỹ thuật:** `TD-01` **trả một phần** — `web`, `api` đã có controller, `db` còn là Pod trần
  (Lab 4b). `TD-03` vẫn mở và giờ có thêm hệ quả: mỗi lần `db` đổi IP, `api` sinh một revision (Lab
  5a). `TD-04` (Lab 6a), `TD-06` (Lab 9b), `TD-07` (Lab 9a) vẫn mở. `TD-02`, `TD-05` đã trả ở các
  sprint trước. Quy tắc tăng `version` bằng tay khi nội dung trang đổi là biện pháp tạm, Lab 5a thay
  bằng ConfigMap có hậu tố hash.

### Bài giải thích cuối cùng

Trong vài phút, kể lại cuộc đời của một bản phát hành `web`: bạn sửa trang mẫu và tăng `version`
trong repo, commit, chạy `deploy.sh`. Kustomize đưa ConfigMap và Deployment lên; Deployment controller
thấy Pod template đổi, tạo ReplicaSet mới tên `web-<hash>`, scale nó lên một Pod; Pod đó chạy
`render-html` đọc mẫu mới, `status-writer` báo `version` mới; readiness đạt, đủ `minReadySeconds`,
controller hạ một Pod cũ — Pod cũ chạy `preStop`, nhận `SIGQUIT`, ra đi trong khi chưa lúc nào dưới 3
bản sao sẵn sàng; lặp lại tới khi ReplicaSet cũ về 0 và ở lại làm lịch sử, mang `CHANGE-CAUSE` của nó;
`rollout status` thoát 0 và `deploy.sh` xong. Rồi kể phiên bản hỏng của câu chuyện đó: chỗ nó kẹt, ai
báo, ai quyết định lùi, và vì sao repo phải được sửa sau `undo`. Kết thúc bằng việc chỉ ra Lab 4b và
Lab 5a sẽ thay đổi câu chuyện này ở đâu.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `git describe` không ra `lab-3c` | Repo ở commit khác, hoặc clone từ bundle cũ | `git checkout lab-3c` hoặc clone lại từ `~/bookstore-lab-3c.bundle` mới nhất |
| B1: `thu-rs` ra nhiều hơn 3 Pod | Pod trần thừa mang `thu: rs` | Xóa Pod thừa rồi chạy lại từ đầu script; đừng tăng `replicas` cho khớp |
| B1: bước 4 thấy Pod có biến `PHIEN_BAN` | `thu-rs` bị xóa thường chứ không `orphan`, nên Pod là Pod mới | Xóa hết Pod `thu=rs`, chạy lại script |
| B2: API từ chối Deployment: `selector does not match template labels` | Label template thiếu một trong hai label của selector | Template phải mang đủ `part-of` và `component` |
| B2: `deploy.sh` báo `field is immutable` ở `spec.selector` | Đổi selector của Deployment đã tạo | `./scripts/teardown.sh && ./scripts/deploy.sh` |
| B2: `deploy.sh` treo ở `wait pod/web` | Vẫn dùng `deploy.sh` của sprint trước | Chép `deploy.sh` mới của B2 |
| B2: lệnh cũ báo `pods "web" not found` | Script hoặc thói quen giả định tên Pod cố định | Dùng `deploy/web` hoặc lấy tên theo label |
| B3: Pod `api` `CrashLoopBackOff`, log báo lỗi xác thực | `secrets/db.env` sinh lại trong khi `db` giữ dữ liệu cũ | `./scripts/teardown.sh && ./scripts/deploy.sh` |
| B3: PDB `disruptionsAllowed` mãi 0 | Có Pod chưa Ready; PDB `maxUnavailable` chọn trúng Pod trần (như `thu-web-tran`); hoặc còn `minAvailable` bằng số bản sao | Chờ Pod Ready; xóa Pod trần mang label dự án; PDB chỉ khai `maxUnavailable: 1` |
| B3: `apply` PDB báo chỉ được khai một trong hai trường | File còn cả `minAvailable` của Lab 3c lẫn `maxUnavailable` | Xóa hẳn dòng `minAvailable` khỏi file |
| B3: eviction thứ hai không bị từ chối | Chạy hai lệnh cách nhau đủ lâu cho Pod thay thế Ready | Chạy liền mạch như script |
| B4: lấy được rất ít mẫu, revision không tăng | Quên đổi Pod template — đổi `strategy` hay `replicas` không sinh rollout | Tăng `version` trong template, commit, chạy lại script |
| B4: `deploy.sh` báo `exceeded its progress deadline` ngay sau restore snapshot | Node chưa có image, kéo lâu hơn `progressDeadlineSeconds` | Chạy khối kéo image ở B0, rồi `./scripts/deploy.sh` lại |
| B5: Phần B không sinh rollout | Annotation đặt ở `metadata` của Deployment, không ở Pod template | Đặt dưới `spec.template.metadata` |
| B5, smoke: `FAIL: co ban sao web phuc vu trang cu` | Cấu hình đổi mà chưa phát hành; hoặc chỗ giữ trong trang mẫu không có dạng `__TEN__` | Phát hành theo B5; nếu chỗ giữ của bạn khác, đổi lại theo Lab 3b |
| B6: không thấy `ProgressDeadlineExceeded` | Chưa đủ thời gian theo `progressDeadlineSeconds` đã đặt | Chờ tiếp; không hạ deadline giữa chừng |
| B6: Pod diễn tập nằm ngoài `lab-k8s-worker2` | Thiếu `nodeSelector` trong template | Xóa `drill-web`, sửa manifest, chạy lại |
| B7: `Recreate` không có mẫu 0 sẵn sàng | Thiếu `preStop`, khoảng trống quá ngắn so với nhịp lấy mẫu | Giữ `preStop` 15 giây và ân hạn 40 giây |
| B8: `diff` trước thí nghiệm đã thoát 1 | Repo có thay đổi chưa triển khai, hoặc còn trôi từ ticket trước | `./scripts/deploy.sh`; còn trôi trong template thì gỡ bằng `null` như phần B |
| B8: sau khi gỡ hotfix, Deployment tạo ReplicaSet mới thay vì dùng lại | Template của bạn còn khác bản trước hotfix (ví dụ map annotation rỗng còn sót) | So `kubectl get rs -o yaml` hai bản; cluster vẫn đúng, chỉ gate tên ReplicaSet fail |
| B9: `deploy.sh` sau `orphan` tạo ReplicaSet mới | Repo khác template đang chạy | Đó là một rollout bình thường; kiểm `git status` và lịch sử phát hành |
| B9: không bắt được `foregroundDeletion` | Pod kết thúc quá nhanh | Giữ `preStop` và ân hạn trong `thu-xoa.yaml` |
| B10: `apply -R` tạo Pod mà selector chung đếm sai | Thiếu `track` trong selector, hai Deployment chọn Pod của nhau | Selector mỗi Deployment gồm cả `thu-nghiem` và `track` |
| B11: `teardown.sh` chờ lâu | Pod `db` dùng hết 60 giây ân hạn | Chờ; script chờ tối đa 300 giây |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Workload Management](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/)
- [Kubernetes v1.35 — ReplicaSet](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/replicaset/)
- [Kubernetes v1.35 — Deployments](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Kubernetes v1.35 — Managing Workloads](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/management/)
- [Kubernetes v1.35 — ReplicationController](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)
- [Kubernetes v1.35 — Use Cascading Deletion in a Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/)
- [Kubernetes v1.35 — Declarative Management of Kubernetes Objects Using Configuration Files](https://v1-35.docs.kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
- [Kubernetes v1.35 — Update API Objects in Place Using kubectl patch](https://v1-35.docs.kubernetes.io/docs/tasks/manage-kubernetes-objects/update-api-object-kubectl-patch/)
- [Kubernetes v1.35 — Run Applications](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/)
- [Kubernetes v1.35 — Run a Stateless Application Using a Deployment](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
- [Kubernetes v1.35 — Horizontal Manual Scaling for a Deployment](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/scale-deployment/)
- [Kubernetes — Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)
  — trang này được thêm sau v1.35 nên chỉ có ở bản tài liệu hiện hành.
- [Kubernetes v1.35 — Specifying a Disruption Budget for your Application](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/configure-pdb/)
- [Kubernetes v1.35 — API-initiated Eviction](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)
- [Kubernetes — `kubectl rollout`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/)
- [Kubernetes — `kubectl diff`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_diff/)
- [Kubernetes — Deployment API reference](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/)
