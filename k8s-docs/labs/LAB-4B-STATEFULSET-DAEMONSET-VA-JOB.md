# Lab 4b — StatefulSet, DaemonSet và Job

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> Repo `~/bookstore` ở tag `lab-4a` (hoặc bundle `~/bookstore-lab-4a.bundle` nếu bạn vừa restore
> snapshot).
> **Điểm kết thúc:** teardown trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới. Repo
> `~/bookstore` có tag `lab-4b` và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 5 — Trạng thái và tác vụ nền của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 4a — ReplicaSet, Deployment và rollout](LAB-4A-REPLICASET-DEPLOYMENT-VA-ROLLOUT.md).
> **Lab sau:** [Lab 5a — Service, EndpointSlice và DNS](LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng mục
[4b. StatefulSet, DaemonSet, Job và autoscaling](../00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling)
của [Giai đoạn 4](../00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller). Mọi ticket dựa vào
hợp đồng kỹ thuật ở [mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật) và bố cục repo
ở [mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Kế hoạch gốc của sprint nằm ở
[mục 7.2](DU-AN-BOOKSTORE.md#sprint-5--lab-4b-trạng-thái-và-tác-vụ-nền).

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Image của dự án nằm
ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00); lab này không chép
lại con số phiên bản nào trong phần văn bản và **không cài thêm gì** vào cluster.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi hệ thống Bookstore đạt trạng thái sau sprint 5 — **mọi thành phần đều có
controller đứng sau** — **và** bạn chứng minh, giải thích được:

- `db` chạy dưới StatefulSet: Pod luôn tên `db-0` và nhiều nhất một bản chạy cùng lúc. Định danh
  đó gồm những phần nào chạy được ngay, phần nào (tên DNS) phải chờ Service headless của Lab 5a.
- Vì sao StatefulSet một bản sao vẫn hợp với DB hơn Deployment, và vì sao **không** được scale `db`
  lên hai bản sao.
- `OrderedReady`: tạo tuần tự, kết thúc ngược, thu nhỏ bị chặn khi có Pod không khỏe; rollout của
  StatefulSet đi từ ordinal lớn nhất, và vì sao hoàn nguyên template chưa đủ để gỡ rollout kẹt.
- Xóa êm và xóa cưỡng bức khác nhau ở đâu, vì sao xóa cưỡng bức phá ngữ nghĩa *nhiều nhất một*.
- Migration chạy bằng Job phải bất biến kết quả; `backoffLimit`, `activeDeadlineSeconds`,
  `restartPolicy` và TTL mỗi thứ chặn ở đâu; Job đã thất bại không tự chạy lại.
- CronJob chỉ tạo Job; `concurrencyPolicy: Forbid` và `startingDeadlineSeconds` đổi hành vi thế
  nào; vì sao bản dump hiện tại chưa phải bản sao lưu thật (`TD-08`).
- DaemonSet `node-agent` có đúng một Pod trên mỗi node kể cả control plane; ai bind Pod vào node;
  toleration nào do controller tự thêm.
- Ba tầng tự phục hồi trên hệ thống thật — kubelet, StatefulSet, DaemonSet — và hai thứ chúng
  **không** cứu được: IP của Pod và dữ liệu trên `emptyDir`.
- Chọn được mẫu xử lý song song (Indexed Job, khai triển template, hàng đợi) cho một bài toán cụ
  thể của dự án.
- Thiết kế HPA cho `web` đúng công thức của bài 72, và vì sao nó chưa chạy được trên cluster này.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 4b | Ticket kiểm chứng |
| --- | --- |
| [65 — StatefulSets](../65-statefulset-vi.md) | B1 — `db` thành StatefulSet, `serviceName` khai trước, không `volumeClaimTemplates`, mục *Hạn chế* trong `teardown.sh`; B2 — định danh, `OrderedReady`, kết thúc ngược, `apply` ghi đè scale tay; B3 — RollingUpdate từ ordinal lớn nhất, *Rollback bắt buộc* |
| [347 — Scale một StatefulSet](../347-scale-stateful-set-vi.md) | B2 — scale lên, scale xuống, scale xuống bị chặn khi có Pod không khỏe; B1 — câu hỏi vì sao không scale `db` |
| [341 — Xóa cưỡng bức Pod của StatefulSet](../341-force-delete-stateful-set-pod-vi.md) | B3 — đo xóa êm so với `--grace-period=0 --force` trên StatefulSet thử nghiệm; B1 — ngữ nghĩa *nhiều nhất một* là lý do `db` lên StatefulSet |
| [67 — Jobs](../67-job-vi.md) | B4 — `db-migrate`: `backoffLimit`, `activeDeadlineSeconds`, chạy lại an toàn; B5 — `Never` so với `OnFailure`, `BackoffLimitExceeded`, `DeadlineExceeded`, Job `Failed` không tự chạy lại; B9 — `completions`, `parallelism`, chế độ `Indexed`, ⭐ hình dạng Job hàng đợi |
| [68 — Tự động dọn dẹp Job đã hoàn thành](../68-ttlafterfinished-vi.md) | B4 — `ttlSecondsAfterFinished`, đặt TTL cho Job đã xong, xóa theo tầng; B5 — đồng hồ TTL đếm từ lúc Job kết thúc |
| [69 — CronJob](../69-cron-jobs-vi.md) | B6 — `db-backup`: `concurrencyPolicy: Forbid`, `startingDeadlineSeconds`, giới hạn lịch sử; CronJob thử nghiệm cho `Forbid` và `suspend`; câu hỏi về sửa CronJob khi IP `db-0` đổi |
| [350 — Chạy tác vụ tự động với CronJob](../350-automated-tasks-cron-jobs-vi.md) | B6 — chuỗi CronJob → Job → Pod, đọc log qua selector tên Job, xóa CronJob kéo theo Job và Pod |
| [349 — Chạy Job](../349-job-tasks-vi.md) | B9 — dùng trang làm bảng chọn mẫu: chạy Indexed Job (353), ⭐ khai triển template (355); hai mẫu hàng đợi (351, 352) không chạy được trên cluster này |
| [353 — Indexed Job](../353-indexed-parallel-processing-vi.md) | B9 — báo cáo tồn kho theo phân vùng, `JOB_COMPLETION_INDEX`, `completedIndexes` |
| [355 — Khai triển template](../355-parallel-processing-expansion-vi.md) | B1, B4 — `deploy.sh` render file `.tpl` bằng `sed` chính là kỹ thuật của bài; B9 ⭐ — một Job cho mỗi phân vùng, thao tác cả nhóm bằng một label |
| [351 — Hàng đợi công việc thô](../351-coarse-parallel-work-queue-vi.md) | **Không thực hành.** Bài dựng Service RabbitMQ (giai đoạn 5) và bắt tự build, push image worker — lab không build image. B9 — câu hỏi thiết kế đối chiếu mẫu hàng đợi với Indexed Job cho cùng bài toán |
| [66 — DaemonSet](../66-daemonset-vi.md) | B7 — `node-agent` trên cả ba node, toleration control plane, `nodeAffinity` do controller ghi, toleration tự thêm; B8 — Pod thay thế lên đúng node cũ; B10 — DaemonSet không có subresource `scale` |
| [38 — Khả năng tự phục hồi](../38-self-healing-vi.md) | B8 — ba tầng: kubelet khởi động lại container, StatefulSet thay `db-0` cùng tên, DaemonSet thay Pod cùng node; giới hạn: IP và dữ liệu `emptyDir` (`TD-03`, `TD-04`) |
| [71 — Tự động co giãn Workload](../71-autoscaling-vi.md) | B2 — co giãn ngang **thủ công** bằng `kubectl scale`; B10 — hai trục co giãn, chọn trục cho `web` |
| [72 — HPA](../72-horizontal-pod-autoscale-vi.md) | B10 — thiết kế HPA `web` thành tài liệu, kiểm bằng server dry-run; chưa chạy được vì thiếu Metrics API — **nợ lab #1**, thực hành ở Lab 11b |
| [73 — VPA](../73-vertical-pod-autoscale-vi.md) | B10 — chứng minh VPA không nằm trong API lõi; câu hỏi vì sao không dùng VPA chế độ `Recreate` cho `db` — **nợ lab #1** (phần VPA) |

Ba nợ lab của nhóm này vẫn **mở** sau sprint — nói được vì sao ở checkpoint:

| # | Phần hoãn | Vì cần | Trả ở |
| --- | --- | --- | --- |
| 1 | Thực hành HPA, VPA (bài 72, 73) | metrics-server của giai đoạn 11; VPA còn cần add-on ngoài A1.4 | HPA ở Lab 11b; VPA chưa có lab trả |
| 2 | `volumeClaimTemplates` cho `db` (bài 65) | StorageClass và provisioner của giai đoạn 6 | Lab 6a |
| 3 | Service headless quản trị `db` (bài 65) | Service của giai đoạn 5 | Lab 5a |

Nợ kỹ thuật của hệ thống sau sprint: **trả xong `TD-01`**; **phát sinh `TD-08`** (bản dump chưa có
nơi bền); `TD-03`, `TD-04`, `TD-06`, `TD-07` vẫn mở — xem
[sổ nợ của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án).

Hai bài thực hành liên quan **không** thuộc nhóm này:
[340 — Xóa một StatefulSet](../340-delete-stateful-set-vi.md) và
[352 — Hàng đợi công việc mịn](../352-fine-parallel-work-queue-vi.md) nằm ở khối thực hành của
giai đoạn 5. B3 chỉ dùng xóa `--cascade=orphan` đã học ở bài
[260](../260-use-cascading-deletion-vi.md) của nhóm 4a.

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B8, B10, B11 và phần chính của B9) khoảng 4 giờ; các mục ⭐ ở B7 và B9
là tùy chọn. B2, B5, B6 có các đoạn chờ đồng hồ của controller vài phút — chạy song song hai ticket
nếu muốn. Có thể chia hai phiên: phiên 1 làm B0–B5, phiên 2 làm B6–B11. Giữa hai phiên cứ để nguyên
cluster; đầu phiên 2 chạy lại gate ngắn A5.5 và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác.
- **Fault injection chỉ trên `lab-k8s-worker2`.** `db-0` được ghim ở đó theo
  [thiết kế dự án](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch). StatefulSet thử nghiệm ở
  B2, B3 cũng ghim `lab-k8s-worker2`; B8 chỉ phá Pod nằm trên node này.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/4b/`. Bằng chứng: `~/lab-evidence/4b/`.
- Lab chỉ dùng kiến thức nhóm 4b và các nhóm trước: Pod (3a), ConfigMap, Secret, Kustomize (3b),
  `requests`/`limits`, PDB (3c), Deployment, rollout, xóa theo tầng (4a). **Không** tạo Service
  (nợ #3), PVC hay StorageClass (nợ #2), HorizontalPodAutoscaler thật, metrics-server hay VPA
  (nợ #1), NetworkPolicy. Không tự viết affinity — B7 chỉ **đọc** `nodeAffinity` do controller ghi.
- Object thử nghiệm (StatefulSet `sts-thu`, các Job và CronJob ở `~/lab-work/4b/`) nằm trong
  namespace `bookstore` nhưng **không** mang label `app.kubernetes.io/part-of`; xóa khi xong ticket.
- `kubectl scale`, `kubectl set image`, `kubectl patch` chỉ dùng trên object thử nghiệm hoặc trong
  diễn tập. Repo là nguồn sự thật duy nhất của hệ thống.
- **Không** đổi số bản sao của `db`. **Không** xóa cưỡng bức `db-0`. Xóa cưỡng bức chỉ chạy trên
  `sts-thu` ở B3.
- Mỗi lần template của `db` đổi, StatefulSet thay `db-0` và dữ liệu trên `emptyDir` mất theo — đó là
  `TD-04`, không phải lỗi của bạn. `scripts/deploy.sh` của lab này chạy lại migration ở mỗi lần triển
  khai để bù phần dữ liệu đó.
- Lab vẫn chỉ có **một** object cấp cluster: PriorityClass `bookstore-critical`. DaemonSet
  `node-agent` là workload duy nhất của dự án được phép chạy trên control plane; không gỡ taint của
  `lab-k8s-master`.
- Chạy mỗi script nghiệm thu trong một shell liền mạch. B2 ghi thời gian đo ra file để B3 dùng lại.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 5: Trạng thái và tác vụ nền

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `01-cluster-ready`, repo ở đúng tag `lab-4a`, và hệ thống
của sprint 4 dựng lại được **chỉ từ repo**.

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
mkdir -p ~/lab-work/4b ~/lab-evidence/4b
test -d ~/bookstore/.git || git clone ~/bookstore-lab-4a.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-4a' && echo 'PASS: HEAD dung tag lab-4a'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
```

Dựng lại hệ thống của sprint 4 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/4b/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/4b/b0-smoke-lab-4a.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/4b/b0-smoke-lab-4a.txt
```

Kéo sẵn hai image mà sprint này dùng nhiều nhất lên **cả ba** node — `node-agent` sẽ chạy cả trên
control plane, còn Job có thể rơi xuống worker nào cũng được. Tag lấy đúng từ
[A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và
[A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00):

```bash
for I in docker.io/library/busybox:1.37 docker.io/library/postgres:17.11; do
  sudo crictl pull "$I" >/dev/null && echo "PASS: lab-k8s-master co $I" || echo "FAIL: lab-k8s-master khong keo duoc $I"
  for N in lab-k8s-worker1 lab-k8s-worker2; do
    ssh "$N" "sudo crictl pull $I" >/dev/null && echo "PASS: $N co $I" || echo "FAIL: $N khong keo duoc $I"
  done
done
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; ba dòng `PASS:` về repo và cluster; `deploy exit=0`,
`smoke exit=0` và file smoke không có dòng `FAIL:`; đủ sáu dòng `PASS: ... co ...`.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 4 **chưa đủ** — có thứ chỉ tồn tại trên
cluster cũ. Sửa repo ở tag `lab-4a` trước, không sửa tay trên cluster
([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-4B-01] `db` lên StatefulSet

**Loại:** Tính năng.

**Bối cảnh:** sprint 4 đã đưa `web` và `api` lên Deployment, nhưng `db` vẫn là Pod trần — mất Pod
là mất dịch vụ cho tới khi có người chạy lại `deploy.sh` (`TD-01`). DB cần một controller giữ
**danh tính** chứ không chỉ giữ **số lượng**: các sprint sau sẽ gọi nó bằng tên `db-0.db` (5a) và
gắn volume bền theo đúng ordinal (6a).

**Yêu cầu:**

1. `db` chạy dưới một StatefulSet tên `db`, **một** bản sao, Pod luôn mang tên `db-0`. Manifest ở
   `k8s/app/db/statefulset.yaml`, thay hẳn `k8s/app/db/pod.yaml`; `k8s/app/kustomization.yaml` trỏ
   tới file mới.
2. Selector của controller theo hợp đồng ([mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label)):
   `app.kubernetes.io/part-of=bookstore` + `app.kubernetes.io/component=db`.
3. Pod `db-0` giữ nguyên mọi hành vi đã có của Pod `db` ở tag `lab-4a`: schema và dữ liệu khởi tạo,
   mật khẩu đọc từ Secret, readiness qua TCP, ân hạn tắt ít nhất 60 giây, PriorityClass
   `bookstore-critical`, `requests`/`limits` cho QoS Guaranteed, ghim `lab-k8s-worker2` bằng
   `nodeSelector`.
4. StatefulSet khai trước tên Service quản trị là `db`. Service đó **chưa** được tạo — nợ lab #3.
5. Dữ liệu vẫn nằm trên `emptyDir` — nợ lab #2, `TD-04`.
6. `scripts/deploy.sh` và `scripts/teardown.sh` thay bằng phiên bản của Lab 4b — chép nguyên văn
   từ khối bên dưới. `deploy.sh` mới render **mọi** file `k8s/app/**/*.tpl` với IP của `db-0`.
7. Không còn Pod trần `db` nào trên cluster; `api` trỏ tới IP hiện tại của `db-0` và `/books` trả
   dữ liệu.

**Ràng buộc:** không tạo Service, không tạo PVC. Không scale `db` quá một bản sao. Không đổi tên
container `postgres` hay cổng `postgres`.

**Đọc lại:**

- Bài [65](../65-statefulset-vi.md): [*Sử dụng StatefulSet*](../65-statefulset-vi.md#using-statefulsets),
  [*Hạn chế*](../65-statefulset-vi.md#limitations), [*Pod Selector*](../65-statefulset-vi.md#pod-selector),
  [*Định danh của Pod*](../65-statefulset-vi.md#pod-identity),
  [*Định danh mạng ổn định*](../65-statefulset-vi.md#stable-network-id),
  [*Nhãn tên Pod*](../65-statefulset-vi.md#pod-name-label).
- Bài [341](../341-force-delete-stateful-set-pod-vi.md), mục
  [*Những điều cần cân nhắc về StatefulSet*](../341-force-delete-stateful-set-pod-vi.md#những-điều-cần-cân-nhắc-về-statefulset-statefulset-considerations)
  — ngữ nghĩa *nhiều nhất một*.
- Bài [347](../347-scale-stateful-set-vi.md), mục
  [*Trước khi bạn bắt đầu*](../347-scale-stateful-set-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin).
- Bài [355](../355-parallel-processing-expansion-vi.md), mục
  [*Tạo các manifest từ template*](../355-parallel-processing-expansion-vi.md#tạo-các-manifest-từ-template-create-manifests-from-the-template)
  — đúng kỹ thuật `deploy.sh` dùng cho file `.tpl`.

**Bàn giao:** `k8s/app/db/statefulset.yaml`; `k8s/app/kustomization.yaml`; hai script mới;
`docs/adr/NNNN-db-statefulset.md` (`NNNN` là số tiếp theo trong `docs/adr/`);
`~/lab-evidence/4b/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` `teardown.sh` rồi `deploy.sh` hai lần liên tiếp đều thành công.
- `PASS:` StatefulSet `db` 1/1 Ready; `db-0` thuộc StatefulSet `db`; không còn Pod trần `db`.
- `PASS:` `db-0` chạy trên `lab-k8s-worker2` do `nodeSelector`; QoS Guaranteed; `spec.priority` bằng
  `value` của `bookstore-critical`; ân hạn ≥ 60 giây.
- `PASS:` định danh: hostname `db-0`, label `statefulset.kubernetes.io/pod-name=db-0`, subdomain `db`.
- `PASS:` `serviceName` là `db` nhưng Service `db` chưa tồn tại và tên `db-0.db...` chưa phân giải.
- `PASS:` dữ liệu trên `emptyDir`, không có PVC nào.
- `PASS:` Deployment `api` mang IP hiện tại của `db-0`; `/books` trả ít nhất 5 cuốn sách.

Hai script, chép nguyên văn:

```bash
cat > ~/bookstore/scripts/deploy.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai dự án Bookstore từ repo — phiên bản Lab 4b.
# db là StatefulSet (Pod db-0). Chưa có Service: mọi file k8s/app/**/*.tpl được thay __DB_HOST__
# bằng IP của Pod db-0 lúc triển khai (TD-03, trả ở Lab 5a).
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore

# 1. Bí mật sinh tại chỗ khi thiếu; secrets/ không vào git (từ Lab 3b).
mkdir -p secrets
if [ ! -s secrets/db.env ]; then
  ( umask 077
    printf 'postgres-password=%s\nauthenticator-password=%s\n' \
      "$(openssl rand -hex 24)" "$(openssl rand -hex 24)" > secrets/db.env )
  echo 'deploy: da sinh secrets/db.env moi'
fi

# 2. Object cấp cluster, rồi toàn bộ namespace bookstore bằng Kustomize. secrets/db.env nằm ngoài
#    thư mục gốc k8s/app nên phải render với --load-restrictor (Lab 3b, ADR 0007).
if compgen -G 'k8s/cluster/*.yaml' >/dev/null; then kubectl apply -f k8s/cluster/; fi
kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | kubectl apply -f -
# Pod trần db của các sprint trước (nếu còn) nhường chỗ cho StatefulSet.
kubectl -n "$NS" delete pod db --ignore-not-found --wait=true

# 3. Chờ db-0 rồi lấy IP để render mọi file .tpl.
kubectl -n "$NS" rollout status statefulset/db --timeout=300s
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s
DB_IP="$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.podIP}')"
test -n "$DB_IP"
render() { sed "s/__DB_HOST__/${DB_IP}/g" "$1"; }

wait_job() {   # thành công khi Job Complete, thất bại ngay khi Job Failed
  local c=''
  for _ in $(seq 1 100); do
    c="$(kubectl -n "$NS" get job "$1" \
          -o jsonpath='{.status.conditions[?(@.status=="True")].type}' 2>/dev/null || true)"
    case " $c " in
      *' Complete '*) return 0 ;;
      *' Failed '*) kubectl -n "$NS" logs "job/$1" --tail=20 || true; return 1 ;;
    esac
    sleep 3
  done
  return 1
}

# 4. Migration chạy ở MỖI lần triển khai nên phải bất biến kết quả. Pod template của Job không
#    sửa được sau khi tạo, nên xóa Job cũ trước rồi tạo lại.
if [ -f k8s/app/jobs/db-migrate.yaml.tpl ]; then
  kubectl -n "$NS" delete job db-migrate --ignore-not-found --wait=true
  render k8s/app/jobs/db-migrate.yaml.tpl | kubectl apply -f -
  wait_job db-migrate || { echo 'deploy: db-migrate that bai'; exit 1; }
fi

# 5. Mọi file .tpl còn lại: Deployment api, CronJob db-backup, ...
find k8s/app -name '*.tpl' ! -name 'db-migrate.yaml.tpl' | sort | while read -r F; do
  render "$F" | kubectl apply -f -
done

# 6. Chờ các controller còn lại.
for D in web api; do
  if kubectl -n "$NS" get deployment "$D" >/dev/null 2>&1; then
    kubectl -n "$NS" rollout status "deployment/$D" --timeout=300s
  fi
done
if kubectl -n "$NS" get daemonset node-agent >/dev/null 2>&1; then
  kubectl -n "$NS" rollout status daemonset/node-agent --timeout=300s
fi
kubectl -n "$NS" get statefulset,deployment,daemonset,cronjob,job -o wide
EOF

cat > ~/bookstore/scripts/teardown.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ toàn bộ dự án Bookstore khỏi cluster — phiên bản Lab 4b.
set -euo pipefail
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
# Kind namespaced được dọn theo label ở mọi namespace (3b, 4a).
KINDS='deployments,replicasets,pods,poddisruptionbudgets,configmaps,secrets'

# Xóa StatefulSet không bảo đảm Pod tắt có thứ tự (bài 65, mục Hạn chế):
# thu db về 0 bản sao trước để db-0 tắt êm, rồi mới xóa namespace.
if kubectl -n "$NS" get statefulset db >/dev/null 2>&1; then
  kubectl -n "$NS" scale statefulset db --replicas=0
  kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s || true
fi

kubectl delete namespace -l "$SEL" --ignore-not-found --wait=true --timeout=300s
kubectl delete "$KINDS" --all-namespaces -l "$SEL" --ignore-not-found --wait=true --timeout=120s
kubectl delete priorityclass -l "$SEL" --ignore-not-found

LEFT="$(kubectl get namespace,priorityclass -l "$SEL" --no-headers 2>/dev/null | wc -l)"
LEFT_NS="$( { kubectl get all -A -l "$SEL" --no-headers
              kubectl get "$KINDS" -A -l "$SEL" --no-headers; } 2>/dev/null | wc -l)"
if [ "$LEFT" -eq 0 ] && [ "$LEFT_NS" -eq 0 ]; then
  echo 'teardown: sach'
else
  echo "teardown: con $LEFT object cap cluster va $LEFT_NS object co label du an"; exit 1
fi
EOF

chmod +x ~/bookstore/scripts/*.sh
```

`deploy.sh` giả định `secrets/db.env` theo đúng dạng Lab 3b sinh ra — mỗi dòng `key=value` với hai
key của Secret `db-credentials`. Nếu `deploy.sh` của tag `lab-4a` sinh file bí mật khác đi, giữ
nguyên khối số 1 của bản cũ và chỉ thay phần còn lại.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-db-statefulset.md`:

1. `db` chỉ có một bản sao. Vậy StatefulSet cho nó thứ gì mà Deployment không cho? Nói bằng định
   danh, ngữ nghĩa *nhiều nhất một* và những gì Lab 5a, 6a sẽ dựa vào.
2. Vì sao **không** được scale `db` lên `replicas: 2` dù StatefulSet làm được việc đó? `db-1` sẽ là
   gì, và bài 347 nói gì về ứng dụng stateful nào scale được?
3. `teardown.sh` thu `db` về 0 bản sao rồi mới xóa namespace. Dẫn đúng câu trong mục *Hạn chế* của
   bài 65.
4. `serviceName: db` trỏ tới một Service chưa tồn tại. Phần nào của định danh vẫn chạy, phần nào
   chưa? Nợ nào, trả ở lab nào, và lúc đó `TD-03` được trả ra sao?

<details>
<summary>Gợi ý</summary>

1. Khung StatefulSet gồm `serviceName`, `replicas`, `selector`, `template`. Phần `template.spec`
   chính là spec Pod `db` hiện có — chép sang, không viết lại.
2. `selector` phải khớp label của `template` và **không sửa được** sau khi tạo. Sai thì xóa
   StatefulSet rồi tạo lại.
3. `kubectl explain statefulset.spec.serviceName` và `kubectl explain statefulset.spec.template`.
4. Trình tự an toàn: sửa `kustomization.yaml`, `git rm k8s/app/db/pod.yaml`, chép hai script, rồi
   `./scripts/teardown.sh && ./scripts/deploy.sh`.
5. Kiểm một tên DNS bằng Pod tạm:
   `kubectl -n bookstore run dns-thu --rm -i --restart=Never --image=busybox:1.37 -- nslookup <tên>`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4b
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

./scripts/teardown.sh > "$EV/b1-teardown.txt" 2>&1
grep -q 'teardown: sach' "$EV/b1-teardown.txt" && echo 'PASS: teardown.sh go sach du an'
./scripts/deploy.sh > "$EV/b1-deploy-1.txt" 2>&1 && echo 'PASS: deploy.sh lan 1 thanh cong'
./scripts/deploy.sh > "$EV/b1-deploy-2.txt" 2>&1 && echo 'PASS: deploy.sh lan 2 thanh cong'

[ "$(jp statefulset/db '{.spec.replicas}/{.status.readyReplicas}')" = '1/1' ] && echo 'PASS: StatefulSet db 1/1 Ready'
[ "$(jp pod/db-0 '{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}')" = 'StatefulSet/db' ] \
  && echo 'PASS: db-0 thuoc StatefulSet db'
kubectl -n "$NS" get pod db >/dev/null 2>&1 && echo 'FAIL: con Pod tran db' || echo 'PASS: khong con Pod tran db'
[ "$(jp statefulset/db '{.spec.selector.matchLabels.app\.kubernetes\.io/component}')" = 'db' ] \
  && echo 'PASS: selector theo hop dong component=db'

{ [ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && \
  [ "$(jp pod/db-0 '{.spec.nodeSelector.kubernetes\.io/hostname}')" = 'lab-k8s-worker2' ]; } \
  && echo 'PASS: db-0 ghim lab-k8s-worker2 bang nodeSelector'
[ "$(jp pod/db-0 '{.status.qosClass}')" = 'Guaranteed' ] && echo 'PASS: db-0 QoS Guaranteed'
V="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}')"
[ "$(jp pod/db-0 '{.spec.priority}')" = "$V" ] && echo "PASS: db-0 co priority $V"
G="$(jp pod/db-0 '{.spec.terminationGracePeriodSeconds}')"
[ "${G:-0}" -ge 60 ] && echo "PASS: db-0 an han $G giay"

HN="$(kubectl -n "$NS" exec db-0 -c postgres -- cat /etc/hostname)"
{ [ "$HN" = 'db-0' ] && [ "$(jp pod/db-0 '{.metadata.labels.statefulset\.kubernetes\.io/pod-name}')" = 'db-0' ] \
  && [ "$(jp pod/db-0 '{.spec.subdomain}')" = 'db' ]; } \
  && echo 'PASS: dinh danh db-0: hostname, label pod-name, subdomain db'

{ [ "$(jp statefulset/db '{.spec.serviceName}')" = 'db' ] && ! kubectl -n "$NS" get service db >/dev/null 2>&1; } \
  && echo 'PASS: serviceName db da khai, Service db chua ton tai (no lab #3)'
kubectl -n "$NS" run dns-thu --rm -i --restart=Never --image=busybox:1.37 \
  -- nslookup db-0.db.bookstore.svc.cluster.local > "$EV/b1-dns.txt" 2>&1
grep -Eq "can't find|NXDOMAIN" "$EV/b1-dns.txt" && echo 'PASS: ten DNS db-0.db chua phan giai duoc'

{ [ -z "$(jp statefulset/db '{.spec.volumeClaimTemplates}')" ] && \
  [ "$(kubectl -n "$NS" get pvc --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  jp statefulset/db '{.spec.template.spec.volumes[*].emptyDir}' | grep -q '{}'; } \
  && echo 'PASS: du lieu tren emptyDir, khong co PVC (no lab #2)'

DB_IP="$(jp pod/db-0 '{.status.podIP}')"
kubectl -n "$NS" get deployment api -o yaml | grep -qF "$DB_IP" && echo "PASS: api mang IP cua db-0 ($DB_IP)"
WEB="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=web -o jsonpath='{.items[0].metadata.name}')"
API_IP="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=api -o jsonpath='{.items[0].status.podIP}')"
N="$(kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- "http://${API_IP}:3000/books" | grep -o '"title"' | wc -l)"
[ "$N" -ge 5 ] && echo "PASS: /books tra $N cuon sach"

kubectl -n "$NS" get statefulset,pod -o wide > "$EV/b1-db.txt"
{ test ! -e k8s/app/db/pod.yaml && test -f k8s/app/db/statefulset.yaml; } && echo 'PASS: statefulset.yaml thay pod.yaml'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Khung StatefulSet là phần mới. Khối `template.spec` dưới đây là spec Pod `db` theo lời giải tham
khảo của các sprint trước; nếu Pod `db` ở tag `lab-4a` của bạn khác (cách nạp mật khẩu, tên volume
ConfigMap, con số tài nguyên đã chốt ở 3c), **giữ phần của bạn** — chỉ khung bên ngoài là thay đổi.

```yaml
# k8s/app/db/statefulset.yaml — thay k8s/app/db/pod.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: db
    app.kubernetes.io/name: postgresql
spec:
  serviceName: db              # nợ lab #3: Service headless "db" được tạo ở Lab 5a
  replicas: 1                  # PostgreSQL ở đây không phải cluster nhân bản — không scale
  selector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: db
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: db
        app.kubernetes.io/name: postgresql
    spec:
      # --- từ đây là spec Pod db ở tag lab-4a, giữ nguyên ---
      nodeSelector:
        kubernetes.io/hostname: lab-k8s-worker2
      priorityClassName: bookstore-critical
      terminationGracePeriodSeconds: 60
      volumes:
      - name: initdb
        configMap:
          name: db-init
      - name: credentials
        secret:
          secretName: db-credentials
          defaultMode: 0400
          items:
          - key: postgres-password
            path: postgres-password
      - name: data
        emptyDir: {}           # nợ lab #2 / TD-04: volumeClaimTemplates ở Lab 6a
      containers:
      - name: postgres
        image: postgres:17.11
        args: ["postgres", "-c", "log_connections=on", "-c", "cluster_name=$(POD_NAME)"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef: {fieldPath: metadata.name}
        - {name: POSTGRES_DB, value: bookstore}
        - {name: POSTGRES_PASSWORD_FILE, value: /etc/db-credentials/postgres-password}
        - name: AUTHENTICATOR_PASSWORD
          valueFrom:
            secretKeyRef: {name: db-credentials, key: authenticator-password}
        ports:
        - {name: postgres, containerPort: 5432}
        readinessProbe:
          exec:
            command: ["pg_isready", "-h", "127.0.0.1", "-p", "5432", "-U", "postgres", "-d", "bookstore"]
          periodSeconds: 5
          timeoutSeconds: 3
        resources:
          requests: {cpu: 500m, memory: 512Mi}
          limits: {cpu: 500m, memory: 512Mi}
        volumeMounts:
        - name: initdb
          mountPath: /docker-entrypoint-initdb.d
          readOnly: true
        - name: credentials
          mountPath: /etc/db-credentials
          readOnly: true
        - name: data
          mountPath: /var/lib/postgresql/data
```

Trong `k8s/app/kustomization.yaml`, mục `resources`: đổi dòng `db/pod.yaml` thành
`db/statefulset.yaml`. Rồi:

```bash
cd ~/bookstore
git rm -q k8s/app/db/pod.yaml
./scripts/teardown.sh && ./scripts/deploy.sh
```

Ý chính cho ADR:

1. Deployment chỉ giữ **số lượng**: tên Pod ngẫu nhiên, và RollingUpdate mặc định có thể chạy Pod
   mới trước khi Pod cũ tắt. StatefulSet giữ **danh tính**: tên `db-0` cố định, hostname cố định, và
   ngữ nghĩa nhiều nhất một Pod mang định danh đó (bài 341). Lab 5a gọi DB bằng `db-0.db`; Lab 6a gắn
   PVC `data-db-0` theo ordinal — cả hai chỉ có nghĩa khi định danh ổn định.
2. StatefulSet cấp **định danh**, không cấp **nhân bản dữ liệu**. `db-1` sẽ là một PostgreSQL độc
   lập, chạy script khởi tạo của riêng nó trên `emptyDir` riêng — hai DB khác nhau chứ không phải một
   DB hai bản. Bài 347: không phải ứng dụng stateful nào cũng scale tốt; phải suy xét ở cấp ứng dụng.
3. Bài 65, *Hạn chế*: StatefulSet không bảo đảm gì về việc kết thúc Pod khi bị xóa; muốn kết thúc có
   thứ tự và êm thì thu nhỏ về 0 trước khi xóa.
4. Chạy: ordinal, hostname `db-0`, label `statefulset.kubernetes.io/pod-name`, `subdomain: db`.
   Chưa chạy: miền DNS `db-0.db.bookstore.svc.cluster.local`, vì Service quản trị là thứ bạn phải
   tự tạo (nợ lab #3, Lab 5a). Khi Service có mặt, `api` và các Job trỏ tới `db-0.db` thay cho IP —
   `deploy.sh` thôi render `.tpl`, trả `TD-03`.

</details>

## B2. [BS-4B-02] Điều tra: thứ tự và định danh của StatefulSet

**Loại:** Điều tra.

**Bối cảnh:** `db` chỉ có một bản sao nên không cho thấy các bảo đảm về thứ tự. Trước khi tin
StatefulSet với dữ liệu thật, đội cần thấy tận mắt các bảo đảm đó trên một StatefulSet **thử
nghiệm** — không bao giờ thử bằng cách scale `db`.

**Yêu cầu:** StatefulSet `sts-thu` ở `~/lab-work/4b/sts-thu.yaml`, ba bản sao `busybox`, ghim
`lab-k8s-worker2`, khai `serviceName: sts-thu` (Service không tồn tại). Mỗi Pod chỉ Ready sau khoảng
10 giây và cần khoảng 10 giây để tắt — đủ rộng để đo thứ tự. Chứng minh:

1. Pod sau chỉ được tạo khi Pod trước đã Running **và** Ready; thứ tự tạo là `0 → N-1`.
2. Mỗi Pod có hostname bằng tên Pod, label tên Pod, `subdomain` bằng `serviceName`; tên DNS của Pod
   chưa phân giải.
3. Scale tay lên 5: vẫn tuần tự. Apply lại manifest (3 bản sao): ghi đè việc scale tay, và Pod bị
   kết thúc theo thứ tự **ngược**, từ ordinal lớn nhất.
4. Làm `sts-thu-1` mất Ready rồi thu nhỏ xuống 2: `sts-thu-2` **không** bị đụng tới cho tới khi
   `sts-thu-1` Ready lại.
5. Xóa êm `sts-thu-1`: Pod thay thế mang đúng tên cũ nhưng là object khác. Ghi thời gian từ lúc xóa
   tới lúc có Pod thay thế vào `~/lab-evidence/4b/b2-t-grace.txt` — B3 dùng lại.

Giữ `sts-thu` lại cho B3.

**Ràng buộc:** không đụng `db`. Không đặt `terminationGracePeriodSeconds: 0`. Không tạo Service.

**Đọc lại:**

- Bài [65](../65-statefulset-vi.md):
  [*Các đảm bảo về triển khai và mở rộng*](../65-statefulset-vi.md#deployment-and-scaling-guarantees),
  [*Quản lý Pod kiểu OrderedReady*](../65-statefulset-vi.md#orderedready-pod-management),
  [*Chỉ số thứ tự*](../65-statefulset-vi.md#ordinal-index),
  [*Replicas*](../65-statefulset-vi.md#replicas).
- Bài [347](../347-scale-stateful-set-vi.md):
  [*Dùng kubectl để scale StatefulSet*](../347-scale-stateful-set-vi.md#dùng-kubectl-để-scale-statefulset-use-kubectl-to-scale-statefulsets),
  [*Scale down không hoạt động đúng*](../347-scale-stateful-set-vi.md#scale-down-không-hoạt-động-đúng-scaling-down-does-not-work-right).
- Bài [71](../71-autoscaling-vi.md), mục
  [*Co giãn workload thủ công*](../71-autoscaling-vi.md#co-giãn-workload-thủ-công-scaling-workloads-manually).

**Bàn giao:** `~/lab-work/4b/sts-thu.yaml`; mục *Điều tra* trong `docs/adr/NNNN-db-statefulset.md`;
`~/lab-evidence/4b/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` `sts-thu-0` chưa Ready thì `sts-thu-1` chưa tồn tại; thứ tự tạo là
  `sts-thu-0 sts-thu-1 sts-thu-2`.
- `PASS:` hostname, label `statefulset.kubernetes.io/pod-name` khớp tên Pod; `subdomain` là `sts-thu`.
- `PASS:` tên `sts-thu-0.sts-thu.bookstore.svc.cluster.local` chưa phân giải.
- `PASS:` scale lên 5 vẫn tuần tự; apply lại manifest đưa về 3, `sts-thu-4` kết thúc trước khi
  `sts-thu-3` bị đụng tới.
- `PASS:` `sts-thu-1` không Ready thì `sts-thu-2` không bị kết thúc; `sts-thu-1` Ready lại thì
  `sts-thu-2` mới đi.
- `PASS:` Pod thay thế của `sts-thu-1` cùng tên, khác UID; file `b2-t-grace.txt` có số giây dương.

**Câu hỏi thiết kế** — ghi vào mục *Điều tra* của ADR `db-statefulset`:

1. Với `OrderedReady`, `sts-thu-0` crash sau khi `sts-thu-1` Ready nhưng trước khi `sts-thu-2` được
   tạo. `sts-thu-2` lên khi nào? Áp vào `db`: nếu một ngày `db` có nhiều bản sao, một bản hỏng làm gì
   với việc mở rộng?
2. Vì sao Kubernetes không tự thu nhỏ khi một Pod stateful đang hỏng? Bài 347 phân biệt hai loại lỗi
   nào?
3. Bạn vừa `kubectl scale` rồi `kubectl apply`. Thao tác nào thắng, và điều đó nói gì về việc ai
   được phép giữ trường `replicas` khi sau này có HPA (B10)?

<details>
<summary>Gợi ý</summary>

- Readiness kiểu `exec` kiểm một file mà container tự tạo sau `sleep 10`; xóa file đó bằng
  `kubectl exec` là cách làm Pod mất Ready mà không giết container.
- Để Pod tắt mất khoảng 10 giây: `preStop` ngủ 8 giây và `terminationGracePeriodSeconds: 10`. Tiến
  trình `sleep` chạy ở PID 1 bỏ qua `SIGTERM`, nên Pod luôn dùng hết ân hạn.
- Label riêng cho object thử nghiệm, ví dụ `lab: sts-thu`, để lọc và xóa dễ.
- Đọc một label có dấu chấm trong jsonpath:
  `'{.metadata.labels.statefulset\.kubernetes\.io/pod-name}'`.

</details>

<details>
<summary>Script nghiệm thu — script này chính là cuộc điều tra; chạy liền một mạch</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4b; W=~/lab-work/4b
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2" 2>/dev/null; }
has() { kubectl -n "$NS" get pod "$1" >/dev/null 2>&1; }
ready() { jp "$1" '{.status.conditions[?(@.type=="Ready")].status}'; }

# 1. OrderedReady khi tạo
kubectl apply -f "$W/sts-thu.yaml"
for i in $(seq 1 120); do has sts-thu-0 && break; sleep 1; done
{ [ "$(ready sts-thu-0)" != 'True' ] && ! has sts-thu-1; } \
  && echo 'PASS: sts-thu-0 chua Ready thi sts-thu-1 chua duoc tao'
kubectl -n "$NS" rollout status statefulset/sts-thu --timeout=300s
ORDER="$(kubectl -n "$NS" get pods -l lab=sts-thu --sort-by=.metadata.creationTimestamp -o name | sed 's|pod/||' | paste -sd' ' -)"
echo "thu tu tao: $ORDER" | tee "$EV/b2-thu-tu.txt"
[ "$ORDER" = 'sts-thu-0 sts-thu-1 sts-thu-2' ] && echo 'PASS: tao tuan tu 0 -> N-1'

# 2. Định danh
BAD=0
for i in 0 1 2; do
  P="sts-thu-$i"
  HN="$(kubectl -n "$NS" exec "$P" -- hostname)"
  LB="$(jp "$P" '{.metadata.labels.statefulset\.kubernetes\.io/pod-name}')"
  SD="$(jp "$P" '{.spec.subdomain}')"
  echo "$P hostname=$HN pod-name=$LB subdomain=$SD node=$(jp "$P" '{.spec.nodeName}')" | tee -a "$EV/b2-dinh-danh.txt"
  { [ "$HN" = "$P" ] && [ "$LB" = "$P" ] && [ "$SD" = 'sts-thu' ]; } || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: hostname, label pod-name, subdomain khop dinh danh'
kubectl -n "$NS" run dns-thu --rm -i --restart=Never --image=busybox:1.37 \
  -- nslookup sts-thu-0.sts-thu.bookstore.svc.cluster.local > "$EV/b2-dns.txt" 2>&1
grep -Eq "can't find|NXDOMAIN" "$EV/b2-dns.txt" && echo 'PASS: ten DNS cua Pod chua phan giai (chua co Service quan tri)'

# 3. Scale tay lên 5, rồi apply lại manifest 3 bản sao
kubectl -n "$NS" scale statefulset sts-thu --replicas=5
for i in $(seq 1 120); do has sts-thu-3 && break; sleep 1; done
{ [ "$(ready sts-thu-3)" != 'True' ] && ! has sts-thu-4; } && echo 'PASS: mo rong van tuan tu (sts-thu-4 cho sts-thu-3)'
kubectl -n "$NS" rollout status statefulset/sts-thu --timeout=300s
kubectl apply -f "$W/sts-thu.yaml"
D4=''; for i in $(seq 1 60); do D4="$(jp sts-thu-4 '{.metadata.deletionTimestamp}')"; [ -n "$D4" ] && break; sleep 1; done
{ [ -n "$D4" ] && has sts-thu-3 && [ -z "$(jp sts-thu-3 '{.metadata.deletionTimestamp}')" ]; } \
  && echo 'PASS: sts-thu-4 ket thuc truoc, sts-thu-3 chua bi dung toi'
for i in $(seq 1 90); do [ "$(kubectl -n "$NS" get pods -l lab=sts-thu --no-headers | wc -l)" -eq 3 ] && break; sleep 2; done
[ "$(kubectl -n "$NS" get statefulset sts-thu -o jsonpath='{.spec.replicas}')" = '3' ] \
  && echo 'PASS: apply manifest ghi de viec scale tay, con 3 ban sao'

# 4. Thu nhỏ bị chặn khi có Pod không khỏe
kubectl -n "$NS" exec sts-thu-1 -- rm -f /tmp/ready
kubectl -n "$NS" wait --for=condition=Ready=false pod/sts-thu-1 --timeout=60s
kubectl -n "$NS" scale statefulset sts-thu --replicas=2
sleep 20
{ has sts-thu-2 && [ -z "$(jp sts-thu-2 '{.metadata.deletionTimestamp}')" ]; } \
  && echo 'PASS: sts-thu-1 khong khoe thi khong thu nho duoc'
kubectl -n "$NS" exec sts-thu-1 -- touch /tmp/ready
kubectl -n "$NS" wait --for=delete pod/sts-thu-2 --timeout=120s && echo 'PASS: sts-thu-1 Ready lai thi sts-thu-2 moi bi ket thuc'

# 5. Xóa êm: giữ tên, đổi object; đo thời gian cho B3
U0="$(jp sts-thu-1 '{.metadata.uid}')"; T0=$(date +%s)
kubectl -n "$NS" delete pod sts-thu-1 --wait=false
for i in $(seq 1 120); do U="$(jp sts-thu-1 '{.metadata.uid}')"; [ -n "$U" ] && [ "$U" != "$U0" ] && break; sleep 1; done
T_GRACE=$(( $(date +%s) - T0 ))
echo "T_GRACE=$T_GRACE" > "$EV/b2-t-grace.txt"
{ [ -n "$U" ] && [ "$U" != "$U0" ] && [ "$T_GRACE" -gt 0 ]; } \
  && echo "PASS: sts-thu-1 thay the cung ten, khac UID, sau ${T_GRACE}s"
kubectl -n "$NS" rollout status statefulset/sts-thu --timeout=120s
```

Nếu gate đầu fail vì `sts-thu-0` đã Ready khi đo: xóa `sts-thu` rồi chạy lại từ đầu, liền mạch.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/4b/sts-thu.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sts-thu
  namespace: bookstore
spec:
  serviceName: sts-thu          # Service này không tồn tại — cùng tình thế với db (nợ lab #3)
  replicas: 3
  selector:
    matchLabels:
      lab: sts-thu
  template:
    metadata:
      labels:
        lab: sts-thu
    spec:
      nodeSelector:
        kubernetes.io/hostname: lab-k8s-worker2
      terminationGracePeriodSeconds: 10
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh", "-c", "sleep 10; touch /tmp/ready; exec sleep 3600"]
        readinessProbe:
          exec: {command: ["test", "-f", "/tmp/ready"]}
          periodSeconds: 2
        lifecycle:
          preStop:
            exec: {command: ["sleep", "8"]}
        resources:
          requests: {cpu: 10m, memory: 8Mi}
          limits: {cpu: 50m, memory: 16Mi}
```

Ý chính:

1. `sts-thu-2` chỉ được tạo sau khi `sts-thu-0` khởi động lại thành công và Running, Ready — bài 65
   yêu cầu **mọi** Pod đứng trước phải Running và Ready. Với một DB nhiều bản sao, một bản hỏng chặn
   mọi thao tác mở rộng phía sau nó.
2. Bài 347: với `replicas > 1`, Kubernetes không phân biệt được lỗi vĩnh viễn với lỗi tạm thời. Lỗi
   vĩnh viễn mà cứ thu nhỏ có thể kéo số thành viên xuống dưới ngưỡng tối thiểu; lỗi tạm thời thì
   cản chính thao tác scale. Vì vậy controller chờ Pod khỏe lại.
3. `kubectl apply` thắng: bài 65, mục *Replicas*, nói apply manifest ghi đè việc scale thủ công trước
   đó. Khi HPA quản lý số bản sao, manifest không được giữ `replicas` (bài 72) — việc của Lab 11b.

</details>

## B3. [BS-4B-03] Diễn tập: rollout kẹt và xóa cưỡng bức

**Loại:** Diễn tập sự cố — trên `lab-k8s-worker2`, chỉ với `sts-thu`.

**Bối cảnh:** hai sự cố mà người trực DB sợ nhất với StatefulSet: một phát hành hỏng làm rollout kẹt
không tự gỡ, và một đồng nghiệp xóa cưỡng bức Pod "cho nhanh". Diễn tập cả hai trên `sts-thu` để
đội có quy trình trước khi `db` có dữ liệu bền ở Lab 6a.

**Yêu cầu:**

1. Phát hành hỏng: đổi image của `sts-thu` sang một tag không tồn tại. Chứng minh rollout bắt đầu từ
   ordinal **lớn nhất** và dừng ở đó; các Pod nhỏ hơn vẫn chạy bản cũ.
2. Hoàn nguyên template về bản tốt. Chứng minh hoàn nguyên **chưa đủ**: Pod lỗi vẫn chạy cấu hình lỗi.
   Gỡ kẹt đúng cách bài 65 chỉ, rollout hoàn tất với image tốt.
3. So sánh xóa êm (thời gian đo ở B2) với `--grace-period=0 --force` trên `sts-thu-1`: tên được giải
   phóng nhanh hơn hẳn.
4. Dọn: xóa StatefulSet theo kiểu bỏ lại Pod, chứng minh Pod còn lại không có chủ, rồi dọn Pod bằng
   label.
5. Viết postmortem ngắn.

**Ràng buộc:** không làm gì trong ticket này với `db-0`. Không gỡ finalizer.

**Đọc lại:**

- Bài [65](../65-statefulset-vi.md): [*Cập nhật cuốn chiếu*](../65-statefulset-vi.md#rolling-updates),
  [*Rollback bắt buộc*](../65-statefulset-vi.md#forced-rollback),
  [*Thực hiện rollback*](../65-statefulset-vi.md#performing-rollbacks).
- Bài [341](../341-force-delete-stateful-set-pod-vi.md):
  [*Xóa Pod*](../341-force-delete-stateful-set-pod-vi.md#xóa-pod-delete-pods),
  [*Xóa cưỡng bức*](../341-force-delete-stateful-set-pod-vi.md#xóa-cưỡng-bức-force-deletion).
- Bài [47](../47-pod-lifecycle-vi.md#pod-termination-forced) — *Buộc kết thúc Pod*.
- Bài [260](../260-use-cascading-deletion-vi.md#set-orphan-deletion-policy) — bỏ lại đối tượng phụ thuộc.

**Bàn giao:** `docs/postmortem/4b-statefulset-rollout-ket.md`; `~/lab-evidence/4b/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` sau khi đổi image, `sts-thu-2` mang image lỗi và không Ready; `sts-thu-0`, `sts-thu-1` giữ
  image cũ.
- `PASS:` sau khi hoàn nguyên template, `sts-thu-2` **vẫn** mang image lỗi.
- `PASS:` xóa tay `sts-thu-2` thì rollout hoàn tất, cả ba Pod chạy image tốt.
- `PASS:` thời gian giải phóng tên khi xóa cưỡng bức nhỏ hơn khi xóa êm.
- `PASS:` StatefulSet xóa kiểu orphan: Pod còn lại, không còn `ownerReferences`; sau đó không còn Pod
  `lab=sts-thu`.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Vì sao StatefulSet cập nhật Pod ordinal lớn nhất trước? Liên hệ với thứ tự kết thúc ở B2.
2. Vì sao hoàn nguyên template không cứu được rollout kẹt? Viết quy trình ba bước cho người trực.
3. Xóa cưỡng bức giải phóng tên nhanh hơn — vậy nó **khẳng định** điều gì về Pod cũ? Nếu đó là `db-0`
   khi dữ liệu đã nằm trên volume bền (6a), hậu quả là gì?
4. Một node mất mạng, Pod StatefulSet trên đó kẹt `Terminating`. Bài 341 khuyến nghị đường nào trước
   khi nghĩ tới xóa cưỡng bức?

<details>
<summary>Gợi ý</summary>

- Đổi image tại chỗ: `kubectl set image statefulset/sts-thu app=<image>`. Hoàn nguyên:
  `kubectl rollout undo statefulset/sts-thu`.
- Đọc image đang chạy của từng Pod: `-o jsonpath='{.spec.containers[0].image}'` trên từng Pod, không
  phải trên StatefulSet.
- Xóa StatefulSet mà để lại Pod: `--cascade=orphan`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau B2; script này chính là cuộc diễn tập</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4b
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2" 2>/dev/null; }
img() { jp "$1" '{.spec.containers[0].image}'; }
BADIMG='busybox:khong-ton-tai'

kubectl -n "$NS" scale statefulset sts-thu --replicas=3
kubectl -n "$NS" rollout status statefulset/sts-thu --timeout=300s
IMG0="$(kubectl -n "$NS" get statefulset sts-thu -o jsonpath='{.spec.template.spec.containers[0].image}')"

# 1. Phát hành hỏng
kubectl -n "$NS" set image statefulset/sts-thu app="$BADIMG"
sleep 45
for i in 0 1 2; do echo "sts-thu-$i image=$(img sts-thu-$i) ready=$(jp sts-thu-$i '{.status.conditions[?(@.type=="Ready")].status}')"; done \
  | tee "$EV/b3-rollout-hong.txt"
{ [ "$(img sts-thu-2)" = "$BADIMG" ] && [ "$(img sts-thu-1)" = "$IMG0" ] && [ "$(img sts-thu-0)" = "$IMG0" ] && \
  [ "$(jp sts-thu-2 '{.status.conditions[?(@.type=="Ready")].status}')" != 'True' ]; } \
  && echo 'PASS: rollout bat dau tu ordinal lon nhat va dung o do'

# 2. Hoàn nguyên template — chưa đủ
kubectl -n "$NS" rollout undo statefulset/sts-thu
sleep 45
if [ "$(img sts-thu-2)" = "$BADIMG" ]; then
  echo 'PASS: hoan nguyen template chua du, sts-thu-2 van chay cau hinh loi'
else
  echo 'KHAC TAI LIEU: sts-thu-2 tu ve template tot — ghi lai vao postmortem'
fi
kubectl -n "$NS" delete pod sts-thu-2 --wait=true
kubectl -n "$NS" rollout status statefulset/sts-thu --timeout=180s
{ [ "$(img sts-thu-0)" = "$IMG0" ] && [ "$(img sts-thu-1)" = "$IMG0" ] && [ "$(img sts-thu-2)" = "$IMG0" ]; } \
  && echo 'PASS: xoa tay Pod loi thi rollout hoan tat voi image tot'

# 3. Xóa cưỡng bức so với xóa êm
T_GRACE="$(sed -n 's/^T_GRACE=//p' "$EV/b2-t-grace.txt")"
U0="$(jp sts-thu-1 '{.metadata.uid}')"; T0=$(date +%s)
kubectl -n "$NS" delete pod sts-thu-1 --grace-period=0 --force
for i in $(seq 1 120); do U="$(jp sts-thu-1 '{.metadata.uid}')"; [ -n "$U" ] && [ "$U" != "$U0" ] && break; sleep 1; done
T_FORCE=$(( $(date +%s) - T0 ))
ssh lab-k8s-worker2 'sudo crictl ps -a --label io.kubernetes.pod.name=sts-thu-1' > "$EV/b3-crictl-sau-force.txt" 2>&1
echo "T_GRACE=${T_GRACE} T_FORCE=${T_FORCE}" | tee "$EV/b3-thoi-gian-xoa.txt"
[ "$T_FORCE" -lt "${T_GRACE:-0}" ] && echo 'PASS: xoa cuong buc giai phong ten nhanh hon xoa em'
kubectl -n "$NS" rollout status statefulset/sts-thu --timeout=120s

# 4. Dọn: orphan rồi xóa theo label
kubectl -n "$NS" delete statefulset sts-thu --cascade=orphan
OWN="$(kubectl -n "$NS" get pods -l lab=sts-thu -o jsonpath='{.items[*].metadata.ownerReferences}')"
{ ! kubectl -n "$NS" get statefulset sts-thu >/dev/null 2>&1 && \
  [ "$(kubectl -n "$NS" get pods -l lab=sts-thu --no-headers | wc -l)" -eq 3 ] && [ -z "$OWN" ]; } \
  && echo 'PASS: StatefulSet bien mat, ba Pod o lai khong con chu'
kubectl -n "$NS" delete pods -l lab=sts-thu --wait=true
[ "$(kubectl -n "$NS" get pods -l lab=sts-thu --no-headers 2>/dev/null | wc -l)" -eq 0 ] && echo 'PASS: da don Pod mo coi bang label'
```

File `b3-crictl-sau-force.txt` chỉ để đọc, không phải gate: tùy thời điểm, bạn có thể thấy container
của Pod cũ vẫn còn trên node trong khi Pod mới cùng tên đã được tạo.

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. RollingUpdate đi theo đúng thứ tự kết thúc: từ ordinal lớn nhất xuống, mỗi lần một Pod, chờ Pod
   vừa cập nhật Running và Ready mới sang Pod đứng trước (bài 65). Pod `0` — thường là thành viên quan
   trọng nhất — được cập nhật sau cùng.
2. Bài 65, *Rollback bắt buộc*: với `OrderedReady`, StatefulSet tiếp tục chờ Pod hỏng trở thành Ready
   — điều không bao giờ xảy ra — trước khi hoàn nguyên nó. Quy trình: (a) hoàn nguyên template;
   (b) xóa tay mọi Pod đã chạy với cấu hình lỗi; (c) theo dõi `rollout status` tới khi xong.
3. Bạn khẳng định Pod cũ sẽ không bao giờ liên lạc lại với ai (bài 341). Xóa cưỡng bức không chờ
   kubelet xác nhận; controller tạo ngay `db-0` mới trong khi tiến trình cũ có thể vẫn chạy. Hai
   PostgreSQL cùng ghi một thư mục dữ liệu trên volume bền là hỏng dữ liệu.
4. Xóa object Node nếu node chắc chắn đã chết, hoặc chờ phân mảnh mạng tự hồi phục để kubelet hoàn
   tất việc xóa. Xóa cưỡng bức là đường cuối.

</details>

## B4. [BS-4B-04] Job `db-migrate` bất biến kết quả

**Loại:** Tính năng.

**Bối cảnh:** hiệu sách nhập thêm sách. Thay đổi dữ liệu phải đi qua **migration** có kiểm soát,
không `psql` tay vào `db-0`. Migration chạy như một workload riêng — một Job — và vì `deploy.sh` chạy
nó ở **mỗi** lần triển khai, chạy lại bao nhiêu lần cũng phải cho cùng kết quả.

**Yêu cầu:**

1. Job `db-migrate` ở `k8s/app/jobs/db-migrate.yaml.tpl`, label theo hợp đồng
   (`component: migrate`, `name: postgresql`). Địa chỉ DB là chuỗi giữ chỗ `__DB_HOST__` — `deploy.sh`
   thay bằng IP của `db-0`.
2. Dùng image `postgres` đúng tag A1.4 và công cụ `psql` của nó; đăng nhập bằng superuser với mật khẩu
   đọc từ Secret `db-credentials` — mật khẩu **không** nằm trong repo.
3. Thêm ít nhất 3 cuốn sách mới vào `api.books`. Chạy lại Job lần thứ hai, thứ ba không thêm dòng nào.
4. Một lỗi SQL làm Job **thất bại** chứ không âm thầm thành công.
5. Thử lại tối đa 3 lần; cả Job không chạy quá 5 phút dù đang thử lại.
6. Job đã xong tự được dọn sau một khoảng đủ để đọc log — ít nhất 10 phút.
7. Container có `requests` và `limits`.

**Ràng buộc:** không sửa schema bằng tay trong `db-0`. Không đặt mật khẩu dạng chữ trong manifest.
Không apply file `.tpl` bằng tay — đi qua `deploy.sh`.

**Đọc lại:**

- Bài [67](../67-job-vi.md): [*Viết một Job spec*](../67-job-vi.md#writing-a-job-spec),
  [*Xử lý lỗi Pod và container*](../67-job-vi.md#xử-lý-lỗi-pod-và-container-handling-pod-and-container-failures)
  — đoạn về chương trình bị khởi động lại trong Pod mới,
  [*Chính sách backoff khi Pod lỗi*](../67-job-vi.md#pod-backoff-failure-policy),
  [*Kết thúc và dọn dẹp Job*](../67-job-vi.md#job-termination-and-cleanup).
- Bài [68](../68-ttlafterfinished-vi.md):
  [*Dọn dẹp các Job đã hoàn thành*](../68-ttlafterfinished-vi.md#dọn-dẹp-các-job-đã-hoàn-thành-cleanup-for-finished-jobs).
- Bài [69](../69-cron-jobs-vi.md), mục [*Việc tạo Job*](../69-cron-jobs-vi.md#việc-tạo-job-job-creation)
  — vì sao Job phải idempotent.
- Bài [355](../355-parallel-processing-expansion-vi.md#create-jobs-based-on-a-template) — template
  với chuỗi giữ chỗ không phải manifest hợp lệ cho tới khi được khai triển.

**Bàn giao:** `k8s/app/jobs/db-migrate.yaml.tpl`; `docs/adr/NNNN-job-migrate.md`;
`~/lab-evidence/4b/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` `deploy.sh` chạy migration thành công; Job `db-migrate` Complete, mang label
  `component=migrate`.
- `PASS:` `api.books` có ít nhất 8 dòng.
- `PASS:` triển khai lại (migration chạy lần hai) không đổi số dòng.
- `PASS:` `backoffLimit` ≤ 3; có `activeDeadlineSeconds` ≤ 300; `ttlSecondsAfterFinished` ≥ 600.
- `PASS:` Job đọc mật khẩu từ Secret `db-credentials`; mật khẩu thật không nằm trong file `.tpl`.
- `PASS:` đặt TTL của Job đã xong về 0 thì Job **và** Pod của nó biến mất.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-job-migrate.md`:

1. Kể ba tình huống khiến cùng một migration chạy nhiều hơn một lần trên hệ thống của bạn. Cơ chế
   nào trong SQL của bạn làm lần chạy thứ hai vô hại?
2. Bạn chọn `restartPolicy` nào cho Job, vì sao — liên hệ lời khuyên của bài 67 khi debug.
3. `backoffLimit: 3` và `activeDeadlineSeconds: 300` cùng có mặt. DB chưa lên, mỗi lần thử mất vài
   giây. Trường nào chặn trước, và Job kết thúc với `reason` gì trong từng trường hợp?
4. Vì sao TTL phải đủ dài? `deploy.sh` gặp gì nếu TTL bằng 0?

<details>
<summary>Gợi ý</summary>

1. `psql` đọc biến môi trường `PGHOST`, `PGPORT`, `PGUSER`, `PGDATABASE`, `PGPASSWORD`. Biến lấy từ
   Secret qua `valueFrom.secretKeyRef`.
2. `psql -v ON_ERROR_STOP=1` làm lệnh SQL lỗi trả mã thoát khác 0. Bọc thay đổi trong
   `BEGIN; ... COMMIT;`.
3. Bất biến kết quả: một unique index trên `(title, author)` tạo bằng `CREATE UNIQUE INDEX IF NOT
   EXISTS`, rồi `INSERT ... ON CONFLICT (title, author) DO NOTHING`.
4. Script nhiều dòng trong `command: ["sh", "-c", "..."]` với heredoc, như init container của Lab 3a.
5. `ttlSecondsAfterFinished` sửa được trên Job đã xong:
   `kubectl patch job db-migrate --type=merge -p ...`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4b
cd ~/bookstore
jp() { kubectl -n "$NS" get job db-migrate -o jsonpath="$1" 2>/dev/null; }
SQL() { kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc "$1"; }

./scripts/deploy.sh > "$EV/b4-deploy-1.txt" 2>&1 && echo 'PASS: deploy.sh chay migration thanh cong'
[ "$(jp '{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ] && echo 'PASS: Job db-migrate Complete'
[ "$(jp '{.metadata.labels.app\.kubernetes\.io/component}')" = 'migrate' ] && echo 'PASS: label component=migrate'
kubectl -n "$NS" logs job/db-migrate > "$EV/b4-log-1.txt" 2>&1
N1="$(SQL 'SELECT count(*) FROM api.books')"
[ "${N1:-0}" -ge 8 ] && echo "PASS: api.books co $N1 dong"

U1="$(jp '{.metadata.uid}')"
./scripts/deploy.sh > "$EV/b4-deploy-2.txt" 2>&1
N2="$(SQL 'SELECT count(*) FROM api.books')"
{ [ "$(jp '{.metadata.uid}')" != "$U1" ] && [ "$N2" = "$N1" ]; } \
  && echo "PASS: migration chay lan hai, so dong giu nguyen $N2"

B="$(jp '{.spec.backoffLimit}')"; A="$(jp '{.spec.activeDeadlineSeconds}')"; T="$(jp '{.spec.ttlSecondsAfterFinished}')"
echo "backoffLimit=$B activeDeadlineSeconds=$A ttlSecondsAfterFinished=$T" | tee "$EV/b4-gioi-han.txt"
{ [ "${B:-9}" -le 3 ] && [ -n "$A" ] && [ "$A" -le 300 ] && [ "${T:-0}" -ge 600 ]; } && echo 'PASS: gioi han thu lai, thoi han va TTL dung yeu cau'

kubectl -n "$NS" get job db-migrate -o yaml | grep -q 'name: db-credentials' && echo 'PASS: Job doc mat khau tu Secret db-credentials'
PW="$(sed -n 's/^postgres-password=//p' secrets/db.env)"
if [ -n "$PW" ]; then
  grep -qF "$PW" k8s/app/jobs/db-migrate.yaml.tpl && echo 'FAIL: mat khau nam trong repo' || echo 'PASS: mat khau khong nam trong file .tpl'
fi

PODS="$(kubectl -n "$NS" get pods -l batch.kubernetes.io/job-name=db-migrate -o name)"
kubectl -n "$NS" patch job db-migrate --type=merge -p '{"spec":{"ttlSecondsAfterFinished":0}}'
for i in $(seq 1 60); do kubectl -n "$NS" get job db-migrate >/dev/null 2>&1 || break; sleep 2; done
LEFT=0; for P in $PODS; do kubectl -n "$NS" get "$P" >/dev/null 2>&1 && LEFT=1; done
{ ! kubectl -n "$NS" get job db-migrate >/dev/null 2>&1 && [ "$LEFT" -eq 0 ]; } \
  && echo 'PASS: TTL controller xoa Job theo tang, Pod di theo'
./scripts/deploy.sh > "$EV/b4-deploy-3.txt" 2>&1 && echo 'PASS: deploy.sh tao lai Job, du lieu khong doi'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/jobs/db-migrate.yaml.tpl — __DB_HOST__ do scripts/deploy.sh thay bằng IP Pod db-0 (TD-03)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: migrate
    app.kubernetes.io/name: postgresql
spec:
  backoffLimit: 3
  activeDeadlineSeconds: 300
  ttlSecondsAfterFinished: 3600
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: migrate
        app.kubernetes.io/name: postgresql
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: postgres:17.11
        env:
        - {name: PGHOST, value: "__DB_HOST__"}
        - {name: PGPORT, value: "5432"}
        - {name: PGUSER, value: postgres}
        - {name: PGDATABASE, value: bookstore}
        - {name: PGCONNECT_TIMEOUT, value: "5"}
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef: {name: db-credentials, key: postgres-password}
        command:
        - sh
        - -c
        - |
          set -e
          psql -v ON_ERROR_STOP=1 <<'SQL'
          BEGIN;
          CREATE UNIQUE INDEX IF NOT EXISTS books_title_author_uq ON api.books (title, author);
          INSERT INTO api.books (title, author, price_vnd, stock) VALUES
            ('Chí Phèo', 'Nam Cao', 45000, 10),
            ('Vang bóng một thời', 'Nguyễn Tuân', 82000, 7),
            ('Những ngôi sao xa xôi', 'Lê Minh Khuê', 52000, 9)
          ON CONFLICT (title, author) DO NOTHING;
          COMMIT;
          SQL
          echo "db-migrate: xong, api.books co $(psql -tAc 'SELECT count(*) FROM api.books') dong"
        resources:
          requests: {cpu: 50m, memory: 64Mi}
          limits: {cpu: 200m, memory: 128Mi}
```

```bash
mkdir -p ~/bookstore/k8s/app/jobs
cd ~/bookstore && ./scripts/deploy.sh && kubectl -n bookstore logs job/db-migrate
```

Ý chính cho ADR:

1. `deploy.sh` chạy migration ở mọi lần triển khai; Job tạo Pod mới khi Pod trước thất bại, và bài 67
   nói ngay cả `completions: 1` cũng có lúc chương trình chạy hai lần; CronJob chỉ tạo Job *xấp xỉ*
   một lần mỗi lịch (bài 69). Unique index cộng `ON CONFLICT DO NOTHING` làm lần sau không thêm gì;
   `BEGIN`/`COMMIT` làm lần chạy hỏng giữa chừng không để lại nửa thay đổi.
2. `Never`: mỗi lần thử là một Pod mới, log lần hỏng còn nguyên để đọc. Với `OnFailure`, Pod bị chấm
   dứt đúng lúc chạm giới hạn backoff nên dễ mất log — bài 67 khuyên `Never` khi debug.
3. Mỗi lần thử ngắn và backoff 10s, 20s, 40s… nên 4 lần thất bại hết trước 5 phút: `backoffLimit`
   chặn trước, `reason: BackoffLimitExceeded`. Nếu mỗi lần thử treo lâu (DB không trả lời),
   `activeDeadlineSeconds` chặn trước với `reason: DeadlineExceeded` — nó được ưu tiên hơn
   `backoffLimit`.
4. TTL 0 cho phép controller xóa Job ngay khi xong: `wait_job` của `deploy.sh` có thể không còn
   Job để đọc trạng thái, và bạn mất log migration. TTL dài đủ cho người vận hành đọc kết quả.

</details>

## B5. [BS-4B-05] Điều tra: Job thất bại và dọn dẹp

**Loại:** Điều tra.

**Bối cảnh:** trước khi giao `db-migrate` cho CI ở các sprint sau, đội phải biết chắc một migration
**hỏng** cư xử thế nào: thử lại bao nhiêu lần, để lại gì, có tự chạy lại không, và ai dọn.

**Yêu cầu:** bốn Job thử nghiệm ở `~/lab-work/4b/`, đều là bản sao rút gọn của `db-migrate`:

| Job | Thay đổi | Phải chứng minh |
| --- | --- | --- |
| `mig-hong-never` | mật khẩu sai (chuỗi chữ, chỉ trong file thử nghiệm); `restartPolicy: Never`; thử lại 2 lần | Job `Failed` với `BackoffLimitExceeded`; mỗi lần thử là một Pod mới; sau đó Job đứng yên, không tự chạy lại |
| `mig-hong-onfailure` | như trên nhưng `restartPolicy: OnFailure` | chỉ một Pod; container của nó bị khởi động lại tại chỗ |
| `mig-cham` | `busybox` ngủ 120 giây; thời hạn 20 giây; thử lại 6 lần | Job `Failed` với `DeadlineExceeded` dù chưa dùng hết lượt thử |
| `ttl-thu` | `busybox` ngủ 30 giây; TTL 15 giây | Job **còn** sau 20 giây kể từ lúc tạo; biến mất không sớm hơn 15 giây sau khi xong, Pod đi theo |

Ghi kết luận vào mục *Điều tra* của `docs/adr/NNNN-job-migrate.md`.

**Ràng buộc:** mật khẩu sai chỉ nằm trong file thử nghiệm, không vào repo. Xóa cả bốn Job khi xong.

**Đọc lại:**

- Bài [67](../67-job-vi.md): [*Chính sách backoff khi Pod lỗi*](../67-job-vi.md#pod-backoff-failure-policy),
  [*Kết thúc và dọn dẹp Job*](../67-job-vi.md#job-termination-and-cleanup),
  [*Các condition kết thúc của Job*](../67-job-vi.md#terminal-job-conditions).
- Bài [68](../68-ttlafterfinished-vi.md):
  [*Dọn dẹp các Job đã hoàn thành*](../68-ttlafterfinished-vi.md#dọn-dẹp-các-job-đã-hoàn-thành-cleanup-for-finished-jobs),
  [*Lệch thời gian*](../68-ttlafterfinished-vi.md#lệch-thời-gian-time-skew).

**Bàn giao:** bốn manifest ở `~/lab-work/4b/`; mục *Điều tra* trong ADR `job-migrate`;
`~/lab-evidence/4b/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` `mig-hong-never` có condition `Failed`, reason `BackoffLimitExceeded`; số Pod bằng
  `status.failed`.
- `PASS:` 60 giây sau khi `Failed`, `mig-hong-never` không có Pod mới, `status.active` bằng 0.
- `PASS:` `mig-hong-onfailure` có đúng một Pod và `restartCount` ≥ 1.
- `PASS:` `mig-cham` có condition `Failed`, reason `DeadlineExceeded`.
- `PASS:` `ttl-thu` còn tồn tại sau 20 giây; biến mất ít nhất 15 giây sau `completionTime`; không còn
  Pod của nó.

**Câu hỏi thiết kế:**

1. Hai cách Job đếm lần thử lại là gì? Ứng với từng `restartPolicy` của bạn.
2. Job `mig-hong-never` đã `Failed`. Muốn migration chạy lại thì ai phải làm gì? `deploy.sh` của bạn
   đã làm việc đó ra sao?
3. TTL dựa trên timestamp trong object Job. Ba VM của bạn lệch giờ thì chuyện gì xảy ra, và Lab 00 đã
   phòng điều đó ở đâu?

<details>
<summary>Gợi ý</summary>

- Bản sao nhanh: chép `db-migrate.yaml.tpl` sang `~/lab-work/4b/`, đổi tên Job, bỏ label của dự án,
  thay khối `PGPASSWORD` bằng `value: sai-mat-khau`, thay lệnh bằng
  `psql -v ON_ERROR_STOP=1 -c 'SELECT 1'`.
- Script nghiệm thu tự render chuỗi giữ chỗ của hai file `.tpl` bằng IP của `db-0`, giống `deploy.sh`.
- `sleep` ở PID 1 bỏ qua `SIGTERM`; đặt `terminationGracePeriodSeconds` nhỏ cho `mig-cham` để Pod tắt
  nhanh khi hết hạn — bài 67, mục *Các condition kết thúc của Job*: condition `Failed` chỉ được
  thêm sau khi mọi Pod đã kết thúc.

</details>

<details>
<summary>Script nghiệm thu — tự viết bốn file thử nghiệm trước, rồi chạy</summary>

Tên file mà script đọc: `mig-hong-never.yaml.tpl`, `mig-hong-onfailure.yaml.tpl`, `mig-cham.yaml`,
`ttl-thu.yaml`, đều ở `~/lab-work/4b/`.

```bash
NS=bookstore; EV=~/lab-evidence/4b; W=~/lab-work/4b
jp() { kubectl -n "$NS" get job "$1" -o jsonpath="$2" 2>/dev/null; }
cond() { jp "$1" "{.status.conditions[?(@.type==\"$2\")].$3}"; }
DB_IP="$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.podIP}')"

for F in mig-hong-never mig-hong-onfailure; do sed "s/__DB_HOST__/${DB_IP}/g" "$W/$F.yaml.tpl" | kubectl apply -f -; done
kubectl apply -f "$W/mig-cham.yaml"
kubectl apply -f "$W/ttl-thu.yaml"; T_CREATE=$(date +%s)

sleep 20
kubectl -n "$NS" get job ttl-thu >/dev/null 2>&1 && echo 'PASS: ttl-thu con sau 20 giay — TTL chua dem khi Job chua xong'
C=''; for i in $(seq 1 60); do C="$(jp ttl-thu '{.status.completionTime}')"; [ -n "$C" ] && break; sleep 2; done
C_EPOCH="$(date -d "$C" +%s)"
for i in $(seq 1 90); do kubectl -n "$NS" get job ttl-thu >/dev/null 2>&1 || break; sleep 2; done
GONE=$(date +%s)
echo "tao=$T_CREATE xong=$C_EPOCH mat=$GONE" | tee "$EV/b5-ttl.txt"
{ ! kubectl -n "$NS" get job ttl-thu >/dev/null 2>&1 && [ $((GONE - C_EPOCH)) -ge 15 ] && \
  [ "$(kubectl -n "$NS" get pods -l batch.kubernetes.io/job-name=ttl-thu --no-headers 2>/dev/null | wc -l)" -eq 0 ]; } \
  && echo 'PASS: TTL dem tu luc Job xong, Pod bi xoa theo tang'

for i in $(seq 1 90); do
  P="$(kubectl -n "$NS" get pods -l batch.kubernetes.io/job-name=mig-hong-onfailure -o jsonpath='{.items[0].status.containerStatuses[0].restartCount}' 2>/dev/null)"
  [ "${P:-0}" -ge 1 ] && break; sleep 2
done
{ [ "$(kubectl -n "$NS" get pods -l batch.kubernetes.io/job-name=mig-hong-onfailure --no-headers | wc -l)" -eq 1 ] && [ "${P:-0}" -ge 1 ]; } \
  && echo "PASS: OnFailure giu mot Pod, container khoi dong lai $P lan"
kubectl -n "$NS" delete job mig-hong-onfailure --wait=true

for i in $(seq 1 120); do [ "$(cond mig-hong-never Failed status)" = 'True' ] && break; sleep 2; done
kubectl -n "$NS" describe job mig-hong-never > "$EV/b5-never.txt"
NF="$(jp mig-hong-never '{.status.failed}')"; NP="$(kubectl -n "$NS" get pods -l batch.kubernetes.io/job-name=mig-hong-never --no-headers | wc -l)"
{ [ "$(cond mig-hong-never Failed reason)" = 'BackoffLimitExceeded' ] && [ "$NP" -eq "${NF:-0}" ]; } \
  && echo "PASS: mig-hong-never Failed BackoffLimitExceeded, $NP Pod = status.failed"
sleep 60
{ [ "$(kubectl -n "$NS" get pods -l batch.kubernetes.io/job-name=mig-hong-never --no-headers | wc -l)" -eq "$NP" ] && \
  [ "$(jp mig-hong-never '{.status.active}')" = '' ]; } && echo 'PASS: Job da Failed thi dung yen, khong tu chay lai'

for i in $(seq 1 60); do [ "$(cond mig-cham Failed status)" = 'True' ] && break; sleep 2; done
[ "$(cond mig-cham Failed reason)" = 'DeadlineExceeded' ] && echo 'PASS: mig-cham Failed DeadlineExceeded truoc khi het luot thu'

kubectl -n "$NS" delete job mig-hong-never mig-cham --ignore-not-found --wait=true
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/4b/mig-hong-never.yaml.tpl — bản -onfailure chỉ đổi tên và restartPolicy: OnFailure
apiVersion: batch/v1
kind: Job
metadata:
  name: mig-hong-never
  namespace: bookstore
spec:
  backoffLimit: 2
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: postgres:17.11
        env:
        - {name: PGHOST, value: "__DB_HOST__"}
        - {name: PGUSER, value: postgres}
        - {name: PGDATABASE, value: bookstore}
        - {name: PGCONNECT_TIMEOUT, value: "5"}
        - {name: PGPASSWORD, value: sai-mat-khau}   # chỉ ở file thử nghiệm
        command: ["psql", "-v", "ON_ERROR_STOP=1", "-c", "SELECT 1"]
        resources:
          requests: {cpu: 20m, memory: 32Mi}
          limits: {cpu: 100m, memory: 64Mi}
---
# ~/lab-work/4b/mig-cham.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mig-cham
  namespace: bookstore
spec:
  backoffLimit: 6
  activeDeadlineSeconds: 20
  template:
    spec:
      restartPolicy: Never
      terminationGracePeriodSeconds: 2
      containers:
      - name: cham
        image: busybox:1.37
        command: ["sleep", "120"]
        resources:
          requests: {cpu: 10m, memory: 8Mi}
          limits: {cpu: 50m, memory: 16Mi}
---
# ~/lab-work/4b/ttl-thu.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ttl-thu
  namespace: bookstore
spec:
  ttlSecondsAfterFinished: 15
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: ngu
        image: busybox:1.37
        command: ["sleep", "30"]
        resources:
          requests: {cpu: 10m, memory: 8Mi}
          limits: {cpu: 50m, memory: 16Mi}
```

Mỗi khối là một file riêng; `mig-hong-never` và `mig-hong-onfailure` giữ đuôi `.tpl` vì còn chuỗi giữ
chỗ.

Ý chính:

1. Số Pod có phase `Failed`; và với `OnFailure`, số lần khởi động lại của container trong các Pod đang
   `Pending` hoặc `Running`. Một trong hai đạt `backoffLimit` là Job thất bại.
2. `restartPolicy` áp dụng cho Pod, không cho Job: Job `Failed` không bao giờ tự chạy lại. Phải xóa
   Job rồi tạo lại — đúng việc `deploy.sh` làm với `db-migrate` trước mỗi lần triển khai.
3. Controller TTL so timestamp lưu trong Job với đồng hồ của control plane; lệch giờ làm Job bị dọn
   sớm hoặc muộn. Lab 00 bật đồng bộ thời gian cho cả ba VM ở bước chuẩn bị OS.

</details>

## B6. [BS-4B-06] CronJob `db-backup`

**Loại:** Tính năng, kèm điều tra.

**Bối cảnh:** hệ thống cũ backup bằng vòng lặp shell cùng máy với DB (`P6`). Sprint này thay vòng
lặp đó bằng CronJob chạy `pg_dump` như một workload riêng. Bản dump **chưa** có nơi lưu bền — volume
bền thuộc giai đoạn 6 — nên đây mới là nửa đầu của lời giải, ghi thành `TD-08`.

**Yêu cầu:**

1. CronJob `db-backup` ở `k8s/app/jobs/db-backup.yaml.tpl`, label theo hợp đồng
   (`component: backup`, `name: postgresql`); địa chỉ DB là `__DB_HOST__`.
2. Chạy mỗi 30 phút. Hai lần backup **không bao giờ** chạy chồng nhau. Một lần chạy bị trễ quá 5 phút
   thì bỏ, chờ lịch kế tiếp.
3. Giữ tối đa 3 lần chạy thành công và **3** lần chạy thất bại gần nhất để chẩn đoán.
4. Mỗi lần chạy: `pg_dump` database `bookstore` ra một file trong một volume tạm của Pod, kiểm file có
   bảng `api.books`, rồi in một dòng log bắt đầu bằng `db-backup: OK`. Thử lại tối đa 2 lần, không
   chạy quá 10 phút. Mật khẩu từ Secret.
5. Điều tra trên một CronJob thử nghiệm `backup-cham` (mỗi phút, mỗi lần chạy 150 giây, không chạy
   chồng): chứng minh lần chạy mới bị bỏ qua khi lần trước chưa xong; tạm ngưng CronJob **không** dừng
   Job đang chạy; xóa CronJob kéo theo Job và Pod của nó.
6. Ghi `TD-08` vào `docs/tech-debt.md`.

**Ràng buộc:** không dùng `hostPath` hay PVC. Chưa có PriorityClass `bookstore-batch` (7a) — không
đặt độ ưu tiên cho Job.

**Đọc lại:**

- Bài [69](../69-cron-jobs-vi.md):
  [*Thời hạn cho Job khởi động trễ*](../69-cron-jobs-vi.md#starting-deadline),
  [*Chính sách chạy đồng thời*](../69-cron-jobs-vi.md#chính-sách-chạy-đồng-thời-concurrency-policy),
  [*Tạm ngưng lập lịch*](../69-cron-jobs-vi.md#tạm-ngưng-lập-lịch-schedule-suspension),
  [*Giới hạn lịch sử Job*](../69-cron-jobs-vi.md#giới-hạn-lịch-sử-job-jobs-history-limits),
  [*Các hạn chế của CronJob*](../69-cron-jobs-vi.md#cron-job-limitations).
- Bài [350](../350-automated-tasks-cron-jobs-vi.md):
  [*Tạo một CronJob*](../350-automated-tasks-cron-jobs-vi.md#creating-a-cron-job),
  [*Xóa một CronJob*](../350-automated-tasks-cron-jobs-vi.md#deleting-a-cron-job).

**Bàn giao:** `k8s/app/jobs/db-backup.yaml.tpl`; `~/lab-work/4b/backup-cham.yaml`;
`docs/adr/NNNN-db-backup.md`; `docs/tech-debt.md`; `~/lab-evidence/4b/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` CronJob `db-backup` mang label `component=backup`; `concurrencyPolicy` là `Forbid`;
  `startingDeadlineSeconds` trong khoảng 10–300; giữ 3 Job thất bại, tối đa 3 Job thành công.
- `PASS:` một Job tạo tay từ CronJob chạy xong, log có `db-backup: OK`; tên Pod khác tên Job.
- `PASS:` bản dump nằm trên `emptyDir` (`TD-08`).
- `PASS:` `backup-cham` chỉ có một Job dù đã qua ít nhất một mốc lịch nữa.
- `PASS:` tạm ngưng `backup-cham` thì Job đang chạy vẫn chạy tiếp.
- `PASS:` xóa `backup-cham` thì không còn Job, Pod nào của nó.
- `PASS:` `docs/tech-debt.md` có `TD-08`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-db-backup.md`:

1. Vì sao `Forbid` mà không `Allow` hay `Replace` cho backup? `Forbid` có ngăn `db-backup` chạy cùng
   lúc với một CronJob khác của dự án không?
2. `startingDeadlineSeconds: 300` đổi **hai** hành vi nào? Nếu control plane chết hai tiếng rồi sống
   lại, `db-backup` của bạn có chạy bù không?
3. `db-0` bị thay và đổi IP; `deploy.sh` render lại CronJob. Job backup **đang chạy** lúc đó dùng IP
   nào? Bài 69 nói gì về sửa CronJob?
4. Vì sao bản dump hiện tại chưa phải backup? Nó mất khi nào? Nợ nào, trả ở lab nào?

<details>
<summary>Gợi ý</summary>

- `pg_dump` cũng đọc `PGHOST`, `PGUSER`, `PGPASSWORD`, `PGDATABASE`. Ghi ra file bằng `--file=`.
- Chạy thử ngay, không chờ lịch: `kubectl -n bookstore create job <tên> --from=cronjob/db-backup`.
- Lấy Pod của một Job theo selector tên Job như bài 350:
  `kubectl get pods --selector=batch.kubernetes.io/job-name=<tên job>`.
- Đọc Job đang hoạt động của một CronJob: `-o jsonpath='{.status.active[*].name}'`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã `deploy.sh` và tự viết `backup-cham.yaml`</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4b; W=~/lab-work/4b
cd ~/bookstore
jp() { kubectl -n "$NS" get cronjob db-backup -o jsonpath="$1" 2>/dev/null; }

[ "$(jp '{.metadata.labels.app\.kubernetes\.io/component}')" = 'backup' ] && echo 'PASS: label component=backup'
[ "$(jp '{.spec.concurrencyPolicy}')" = 'Forbid' ] && echo 'PASS: concurrencyPolicy Forbid'
SDS="$(jp '{.spec.startingDeadlineSeconds}')"
{ [ "${SDS:-0}" -ge 10 ] && [ "$SDS" -le 300 ]; } && echo "PASS: startingDeadlineSeconds=$SDS"
{ [ "$(jp '{.spec.failedJobsHistoryLimit}')" = '3' ] && [ "$(jp '{.spec.successfulJobsHistoryLimit}')" -le 3 ]; } \
  && echo 'PASS: gioi han lich su Job'
jp '{.spec.jobTemplate.spec.template.spec.volumes[*].emptyDir}' | grep -q '{}' && echo 'PASS: dump nam tren emptyDir (TD-08)'

kubectl -n "$NS" delete job db-backup-thu --ignore-not-found --wait=true
kubectl -n "$NS" create job db-backup-thu --from=cronjob/db-backup
kubectl -n "$NS" wait --for=condition=Complete job/db-backup-thu --timeout=300s
POD="$(kubectl -n "$NS" get pods --selector=batch.kubernetes.io/job-name=db-backup-thu -o jsonpath='{.items[0].metadata.name}')"
kubectl -n "$NS" logs "$POD" > "$EV/b6-backup-log.txt" 2>&1
grep -q '^db-backup: OK' "$EV/b6-backup-log.txt" && echo 'PASS: chay thu db-backup thanh cong'
[ "$POD" != 'db-backup-thu' ] && echo "PASS: ten Pod ($POD) khac ten Job"
kubectl -n "$NS" delete job db-backup-thu --wait=true

grep -q 'TD-08' docs/tech-debt.md && echo 'PASS: tech-debt.md co TD-08'

# Điều tra trên backup-cham
kubectl apply -f "$W/backup-cham.yaml"
for i in $(seq 1 45); do [ -n "$(kubectl -n "$NS" get cronjob backup-cham -o jsonpath='{.status.active[*].name}')" ] && break; sleep 2; done
echo 'da co Job dang chay, cho qua them mot moc lich'; sleep 70
J="$(kubectl -n "$NS" get jobs -l lab=backup-cham -o name)"
kubectl -n "$NS" get jobs -l lab=backup-cham > "$EV/b6-forbid.txt"
[ "$(echo "$J" | grep -c .)" -eq 1 ] && echo 'PASS: Forbid bo qua lan chay moi khi lan truoc chua xong'
kubectl -n "$NS" patch cronjob backup-cham --type=merge -p '{"spec":{"suspend":true}}'
sleep 5
[ "$(kubectl -n "$NS" get "$J" -o jsonpath='{.status.active}')" = '1' ] && echo 'PASS: suspend khong dung Job dang chay'
kubectl -n "$NS" delete cronjob backup-cham --wait=true
for i in $(seq 1 60); do
  [ "$(kubectl -n "$NS" get jobs,pods -l lab=backup-cham --no-headers 2>/dev/null | wc -l)" -eq 0 ] && break; sleep 2
done
[ "$(kubectl -n "$NS" get jobs,pods -l lab=backup-cham --no-headers 2>/dev/null | wc -l)" -eq 0 ] \
  && echo 'PASS: xoa CronJob keo theo Job va Pod'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/jobs/db-backup.yaml.tpl — __DB_HOST__ do scripts/deploy.sh thay bằng IP Pod db-0 (TD-03)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: backup
    app.kubernetes.io/name: postgresql
spec:
  schedule: "*/30 * * * *"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: backup
        app.kubernetes.io/name: postgresql
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 600
      template:
        metadata:
          labels:
            app.kubernetes.io/part-of: bookstore
            app.kubernetes.io/component: backup
            app.kubernetes.io/name: postgresql
        spec:
          restartPolicy: Never
          volumes:
          - name: backup
            emptyDir: {}            # TD-08: bản dump mất cùng Pod; PVC backup-store ở Lab 6a
          containers:
          - name: backup
            image: postgres:17.11
            env:
            - {name: PGHOST, value: "__DB_HOST__"}
            - {name: PGUSER, value: postgres}
            - {name: PGDATABASE, value: bookstore}
            - {name: PGCONNECT_TIMEOUT, value: "5"}
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef: {name: db-credentials, key: postgres-password}
            command:
            - sh
            - -c
            - |
              set -e
              F="/backup/bookstore-$(date +%Y%m%dT%H%M%S).sql"
              pg_dump --no-owner --file="$F"
              grep -q 'CREATE TABLE api.books' "$F"
              echo "db-backup: OK $F $(wc -c < "$F") byte"
            volumeMounts:
            - {name: backup, mountPath: /backup}
            resources:
              requests: {cpu: 50m, memory: 64Mi}
              limits: {cpu: 200m, memory: 128Mi}
```

```yaml
# ~/lab-work/4b/backup-cham.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cham
  namespace: bookstore
spec:
  schedule: "* * * * *"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 30
  jobTemplate:
    metadata:
      labels:
        lab: backup-cham
    spec:
      template:
        metadata:
          labels:
            lab: backup-cham
        spec:
          restartPolicy: Never
          terminationGracePeriodSeconds: 2
          containers:
          - name: cham
            image: busybox:1.37
            command: ["sh", "-c", "echo bat-dau-backup-gia; sleep 150"]
            resources:
              requests: {cpu: 10m, memory: 8Mi}
              limits: {cpu: 50m, memory: 16Mi}
```

Dòng thêm vào `docs/tech-debt.md`:

```markdown
| TD-08 | Bản dump của db-backup nằm trên emptyDir, mất cùng Pod | 4b | 6a | mở | ~/lab-evidence/4b/b6-backup-log.txt |
```

Ý chính cho ADR:

1. Hai `pg_dump` song song làm nặng DB mà không thêm giá trị; `Replace` giết bản đang chạy dở. `Forbid`
   chỉ áp dụng trong phạm vi **một** CronJob — Job của CronJob khác vẫn chạy song song được.
2. (a) Trễ quá 300 giây thì bỏ lần đó, tính là Job thất bại. (b) Controller chỉ đếm lịch bị lỡ trong
   300 giây gần nhất thay vì từ lần lập lịch cuối tới hiện tại. Sau hai tiếng chết, các mốc cũ hơn
   300 giây không được chạy bù; lịch kế tiếp vẫn chạy bình thường. Với lịch 30 phút, hai tiếng chỉ lỡ
   4 mốc — xa ngưỡng 100 của bài 69; ngưỡng đó mới là mối lo với CronJob chạy mỗi phút.
3. IP cũ. Sửa CronJob chỉ áp dụng cho Job **mới**; Job đã khởi động chạy tiếp với template cũ — và
   thất bại vì `db-0` cũ không còn. Đây vẫn là `TD-03`.
4. File dump nằm trong `emptyDir` của Pod Job: Pod bị xóa (theo giới hạn lịch sử, hay khi bạn xóa Job)
   là bản dump mất. Ngoài ra nó cùng cluster, cùng node lưu với DB. `TD-08`, trả ở Lab 6a bằng PVC
   `backup-store`.

</details>

## B7. [BS-4B-07] DaemonSet `node-agent` trên cả ba node

**Loại:** Tính năng.

**Bối cảnh:** người vận hành cần biết request vừa rồi do node nào phục vụ và node đó đang tải ra sao.
Một agent nhỏ chạy **trên mỗi node** công bố tên node và tải hiện tại. Lab 5a sẽ cho sidecar
`status-writer` của `web` đọc agent **cùng node**; Lab 11a chuyển agent sang namespace riêng để đọc
log node.

**Yêu cầu:**

1. DaemonSet `node-agent` ở `k8s/app/agent/daemonset.yaml`, label theo hợp đồng
   (`component: agent`, `name: busybox`), thêm vào `k8s/app/kustomization.yaml`.
2. Container tên `agent`, image `busybox` đúng tag A1.3, cổng `9100` tên `http`
   ([mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe)).
3. `GET /node.json` trả JSON có ba trường: `node` (tên node mà Pod đang chạy), `load` (tải trung bình
   một phút của node), `updated` (thời điểm cập nhật). Nội dung làm mới vài giây một lần; phục vụ bằng
   `httpd` của busybox.
4. Đúng **một** Pod trên **mỗi** node, kể cả `lab-k8s-master`.
5. Agent chạy non-root, không leo thang đặc quyền; container có `requests`/`limits` theo
   [ngân sách mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch); chỉ Ready khi
   `/node.json` phục vụ được.
6. Nếu tiến trình `httpd` chết, container phải thoát để kubelet khởi động lại nó; nhận `SIGTERM` thì
   thoát gọn.
7. Đọc trên một Pod của agent: ai đặt `.spec.nodeName`, controller ghi gì vào `affinity`, và những
   toleration nào bạn không viết mà vẫn có.

**Ràng buộc:** không gỡ taint control plane. Không dùng `hostNetwork`, `hostPort`, `hostPath`. Không
tạo Service — gọi agent bằng IP Pod.

**Đọc lại:**

- Bài [66](../66-daemonset-vi.md):
  [*Viết một Spec cho DaemonSet*](../66-daemonset-vi.md#viết-một-spec-cho-daemonset-writing-a-daemonset-spec),
  [*Bộ chọn Pod*](../66-daemonset-vi.md#pod-selector),
  [*Chạy Pod trên một số Node được chọn*](../66-daemonset-vi.md#chạy-pod-trên-một-số-node-được-chọn-running-pods-on-select-nodes),
  [*Cách các Daemon Pod được lập lịch*](../66-daemonset-vi.md#how-daemon-pods-are-scheduled),
  [*Taint và toleration*](../66-daemonset-vi.md#taint-và-toleration-taints-and-tolerations),
  [*Deployment*](../66-daemonset-vi.md#deployment-deployments) trong mục các lựa chọn thay thế.
- Bài [60](../60-advanced-pod-config-vi.md#toleration-tolerations) — toleration (Lab 3a);
  [*Security context*](../60-advanced-pod-config-vi.md#security-context).
- Bài [336](../336-env-variable-expose-pod-info-vi.md) — tên node qua biến môi trường.

**Bàn giao:** `k8s/app/agent/daemonset.yaml`; `k8s/app/kustomization.yaml`;
`docs/adr/NNNN-node-agent.md`; `~/lab-evidence/4b/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` `desiredNumberScheduled` và `numberReady` bằng số node (3); mỗi node có đúng một Pod; một
  Pod nằm trên `lab-k8s-master`.
- `PASS:` container `agent` có cổng `http` 9100, `requests`/`limits`, chạy với uid khác 0.
- `PASS:` `/node.json` của từng Pod trả đúng tên node của nó, có đủ `load` và `updated`.
- `PASS:` `nodeAffinity` của Pod trỏ `metadata.name` đúng node; `schedulerName` là `default-scheduler`.
- `PASS:` Pod có sáu toleration do controller tự thêm; **không** có `network-unavailable`.
- `PASS:` (⭐) Pod Flannel (`hostNetwork: true`) có toleration `network-unavailable`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-node-agent.md`:

1. Vì sao không có toleration thì chỉ có hai Pod? Toleration của bạn mở khóa đúng taint nào, và vì
   sao không tolerate mọi taint?
2. DaemonSet controller có tự đặt `.spec.nodeName` không? Hệ quả khi node đầy tài nguyên là gì?
3. Bài 66 so DaemonSet với Deployment theo tiêu chí nào? Áp vào `node-agent`.
4. Vì sao Pod Flannel có toleration `network-unavailable` còn Pod `node-agent` thì không? Nếu Pod
   mạng không có các toleration tự thêm đó thì cluster bế tắc ra sao?

<details>
<summary>Gợi ý</summary>

1. Toleration cho taint control plane: key `node-role.kubernetes.io/control-plane`, `operator:
   Exists`, effect `NoSchedule` — đã dùng ở Lab 3a.
2. Script trong `command: ["sh", "-c", "..."]`: ghi `/www/node.json` vào một `emptyDir`, chạy
   `httpd -f -p 9100 -h /www &`, lặp ghi lại file mỗi 5 giây khi `kill -0` còn thấy `httpd`.
3. Tải một phút là trường đầu của `/proc/loadavg`. Ghi ra file tạm rồi `mv` để `httpd` không phục vụ
   file ghi dở.
4. `trap ... TERM` kết hợp `sleep 5 & wait $!` như sidecar của Lab 3a.
5. Gọi agent từ một container có `wget`: `kubectl -n bookstore exec <Pod web> -c status-writer --
   wget -qO- http://<IP agent>:9100/node.json`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau `./scripts/deploy.sh`</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4b
SEL='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=agent'
ds() { kubectl -n "$NS" get daemonset node-agent -o jsonpath="$1" 2>/dev/null; }
NODES="$(kubectl get nodes --no-headers | wc -l)"

kubectl -n "$NS" get pods -l "$SEL" -o wide | tee "$EV/b7-pods.txt"
{ [ "$(ds '{.status.desiredNumberScheduled}')" -eq "$NODES" ] && [ "$(ds '{.status.numberReady}')" -eq "$NODES" ]; } \
  && echo "PASS: node-agent Ready tren $NODES/$NODES node"
NN="$(kubectl -n "$NS" get pods -l "$SEL" -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}')"
{ [ "$(echo "$NN" | sort -u | wc -l)" -eq "$NODES" ] && [ "$(echo "$NN" | wc -l)" -eq "$NODES" ] && \
  echo "$NN" | grep -qx 'lab-k8s-master'; } && echo 'PASS: moi node dung mot Pod, ke ca lab-k8s-master'

C='{.spec.template.spec.containers[?(@.name=="agent")]'
{ [ "$(ds "$C.ports[?(@.name==\"http\")].containerPort}")" = '9100' ] && \
  [ -n "$(ds "$C.resources.requests.cpu}")" ] && [ -n "$(ds "$C.resources.limits.memory}")" ]; } \
  && echo 'PASS: container agent co cong http 9100 va requests/limits'
AP="$(kubectl -n "$NS" get pods -l "$SEL" -o jsonpath='{.items[0].metadata.name}')"
[ "$(kubectl -n "$NS" exec "$AP" -c agent -- id -u)" != '0' ] && echo 'PASS: agent chay non-root'

WEB="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=web -o jsonpath='{.items[0].metadata.name}')"
BAD=0
for L in $(kubectl -n "$NS" get pods -l "$SEL" -o jsonpath='{range .items[*]}{.spec.nodeName}={.status.podIP}{"\n"}{end}'); do
  N="${L%%=*}"; IP="${L##*=}"
  J="$(kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- "http://${IP}:9100/node.json")"
  echo "$N -> $J" | tee -a "$EV/b7-node-json.txt"
  { echo "$J" | grep -Eq "\"node\" *: *\"$N\"" && echo "$J" | grep -q '"load"' && echo "$J" | grep -q '"updated"'; } || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: moi agent tra /node.json dung ten node, du load va updated'

W2="$(kubectl -n "$NS" get pods -l "$SEL" --field-selector spec.nodeName=lab-k8s-worker2 -o jsonpath='{.items[0].metadata.name}')"
pj() { kubectl -n "$NS" get pod "$W2" -o jsonpath="$1"; }
NA='{.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[0].matchFields[0]'
{ [ "$(pj "$NA.key}")" = 'metadata.name' ] && [ "$(pj "$NA.values[0]}")" = "$(pj '{.spec.nodeName}')" ] && \
  [ "$(pj '{.spec.schedulerName}')" = 'default-scheduler' ]; } \
  && echo 'PASS: controller ghi nodeAffinity khop host, scheduler mac dinh dat nodeName'
KEYS="$(pj '{range .spec.tolerations[*]}{.key}{"\n"}{end}')"
echo "$KEYS" > "$EV/b7-tolerations.txt"
BAD=0
for K in node.kubernetes.io/not-ready node.kubernetes.io/unreachable node.kubernetes.io/disk-pressure \
         node.kubernetes.io/memory-pressure node.kubernetes.io/pid-pressure node.kubernetes.io/unschedulable; do
  echo "$KEYS" | grep -qx "$K" || { echo "FAIL: thieu toleration $K"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du sau toleration controller tu them'
echo "$KEYS" | grep -qx 'node.kubernetes.io/network-unavailable' \
  && echo 'FAIL: co network-unavailable du khong dung hostNetwork' || echo 'PASS: khong co network-unavailable'

# ⭐ Đối chiếu với Flannel — chỉ đọc
FL="$(kubectl -n kube-flannel get pods -l app=flannel -o jsonpath='{.items[0].metadata.name}')"
{ [ "$(kubectl -n kube-flannel get pod "$FL" -o jsonpath='{.spec.hostNetwork}')" = 'true' ] && \
  kubectl -n kube-flannel get pod "$FL" -o jsonpath='{range .spec.tolerations[*]}{.key}{"\n"}{end}' \
  | grep -qx 'node.kubernetes.io/network-unavailable'; } && echo 'PASS: Pod hostNetwork cua Flannel moi co network-unavailable'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/agent/daemonset.yaml — Lab 11a chuyển sang namespace bookstore-system
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: agent
    app.kubernetes.io/name: busybox
spec:
  selector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: agent
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: agent
        app.kubernetes.io/name: busybox
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      terminationGracePeriodSeconds: 10
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        runAsGroup: 65534
      volumes:
      - name: www
        emptyDir: {}
      containers:
      - name: agent
        image: busybox:1.37
        env:
        - name: NODE_NAME
          valueFrom: {fieldRef: {fieldPath: spec.nodeName}}
        command:
        - sh
        - -c
        - |
          trap 'echo "node-agent: nhan TERM"; kill "$HTTPD" 2>/dev/null; exit 0' TERM
          write() {
            printf '{"node":"%s","load":"%s","updated":"%s"}\n' \
              "$NODE_NAME" "$(cut -d' ' -f1 /proc/loadavg)" "$(date -Iseconds)" > /www/node.json.tmp
            mv /www/node.json.tmp /www/node.json
          }
          write
          httpd -f -p 9100 -h /www &
          HTTPD=$!
          while kill -0 "$HTTPD" 2>/dev/null; do
            write
            sleep 5 & wait $!
          done
          echo "node-agent: httpd da dung"
          exit 1
        ports:
        - {name: http, containerPort: 9100}
        readinessProbe:
          httpGet: {path: /node.json, port: http}
          periodSeconds: 10
        securityContext:
          allowPrivilegeEscalation: false
        resources:
          requests: {cpu: 10m, memory: 16Mi}
          limits: {cpu: 50m, memory: 32Mi}
        volumeMounts:
        - {name: www, mountPath: /www}
```

Thêm `agent/daemonset.yaml` vào mục `resources` của `k8s/app/kustomization.yaml`, rồi
`./scripts/deploy.sh`.

Ý chính cho ADR:

1. `lab-k8s-master` mang taint `node-role.kubernetes.io/control-plane:NoSchedule` do kubeadm đặt. Chỉ
   tolerate đúng taint đó: agent được lên control plane, nhưng một taint mới (ví dụ khi bảo trì ở
   7a, 12) vẫn có hiệu lực với nó.
2. Không. Controller tạo Pod kèm `nodeAffinity` với `matchFields` trỏ `metadata.name` của node đích;
   scheduler mặc định mới bind. Node đầy thì scheduler có thể phải chiếm chỗ Pod khác theo độ ưu tiên
   — cơ chế học ở 7a.
3. Bài 66: Deployment khi số lượng và việc phát hành quan trọng hơn vị trí; DaemonSet khi **mỗi**
   node phải có một bản vì nó cung cấp chức năng cấp node. `node-agent` báo tải của **node nó đang
   chạy** — chạy hai bản trên một node hay thiếu ở node nào đều sai nghĩa.
4. Controller tự thêm `not-ready:NoExecute` cho mọi Pod DaemonSet; riêng `network-unavailable` chỉ
   thêm cho Pod `hostNetwork: true`. Flannel cần cả hai để lên node trước khi node Ready; thiếu thì
   node không Ready vì chưa có mạng, mạng không chạy vì node chưa Ready.

</details>

## B8. [BS-4B-08] Diễn tập: tự phục hồi ba tầng

**Loại:** Diễn tập sự cố — chỉ trên `lab-k8s-worker2`.

**Bối cảnh:** ở Lab 3a, xóa Pod `db` là không ai tạo lại (`TD-01`). Sprint này tuyên bố đã trả nợ đó.
Diễn tập chứng minh điều ấy, và vạch rõ hai thứ mà controller **không** cứu được.

**Yêu cầu:**

1. Tầng kubelet: làm container `agent` trên `lab-k8s-worker2` thoát, và làm PostgreSQL trong `db-0`
   tắt nhanh (`SIGINT` vào tiến trình 1). Chứng minh: container được khởi động lại, object Pod giữ
   nguyên UID; `db-0` giữ IP và **giữ dữ liệu**; `api` tự Ready lại.
2. Tầng DaemonSet: xóa Pod `node-agent` trên `lab-k8s-worker2`. Chứng minh Pod thay thế tên khác, UID
   khác, nhưng nằm **đúng** `lab-k8s-worker2`.
3. Tầng StatefulSet: xóa Pod `db-0`. Chứng minh Pod thay thế **cùng tên** `db-0`, UID khác, cùng node.
   Ghi lại: IP có đổi không, `api.books` còn bao nhiêu dòng, `api` có Ready không.
4. Khôi phục bằng đúng một lệnh của repo, không sửa manifest. Chứng minh dữ liệu migration và `/books`
   trở lại.
5. Postmortem.

**Ràng buộc:** không xóa cưỡng bức. Không sửa repo trong lúc diễn tập.

**Đọc lại:**

- Bài [38](../38-self-healing-vi.md): [*Các khả năng tự phục hồi*](../38-self-healing-vi.md#self-healing-capabilities),
  [*Những điểm cần cân nhắc*](../38-self-healing-vi.md#considerations).
- Bài [47](../47-pod-lifecycle-vi.md): [*Khởi động lại container*](../47-pod-lifecycle-vi.md#restart-policy),
  [*Pod và việc khôi phục sau lỗi*](../47-pod-lifecycle-vi.md#pod-fault-recovery).
- Bài [65](../65-statefulset-vi.md#pod-identity) — định danh đi theo Pod.

**Bàn giao:** `docs/postmortem/4b-tu-phuc-hoi.md`; `~/lab-evidence/4b/b8-*.txt`.

**Nghiệm thu:**

- `PASS:` container `agent` trên worker2: `restartCount` tăng, UID giữ nguyên.
- `PASS:` `db-0` sau `SIGINT`: `restartCount` tăng, UID và IP giữ nguyên, số sách không đổi, `api`
  Ready.
- `PASS:` Pod `node-agent` thay thế khác tên, khác UID, vẫn trên `lab-k8s-worker2`.
- `PASS:` `db-0` thay thế cùng tên, khác UID, trên `lab-k8s-worker2`; `api.books` về đúng 5 dòng.
- `PASS:` nếu IP của `db-0` đổi: `api` NotReady cho tới khi chạy `deploy.sh`.
- `PASS:` sau `./scripts/deploy.sh`: `api` Ready, `api.books` ≥ 8, `/books` trả dữ liệu.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Ba tầng vừa diễn tập: ai xử lý, theo trường nào, và mỗi tầng giữ **cái gì** cho Pod hay container
   được thay — tên, UID, IP, node, dữ liệu?
2. So với kịch bản B của Lab 3a B4: phần nào giờ tự lành, phần nào vẫn không? Gọi đúng mã nợ của từng
   phần còn lại và lab trả nó.
3. Vì sao khôi phục bằng `deploy.sh` là an toàn dù migration chạy thêm một lần nữa?

<details>
<summary>Gợi ý</summary>

- Tiến trình 1 của container `agent` là shell có `trap TERM`: `kubectl exec <Pod> -c agent -- kill 1`.
- Tiến trình 1 của container `postgres` là PostgreSQL: `kubectl exec db-0 -c postgres -- kill -INT 1`.
- Chờ mọi Pod của `api` mất Ready: `kubectl wait --for=condition=Ready=false pod -l <selector>`.

</details>

<details>
<summary>Script nghiệm thu — script này chính là cuộc diễn tập</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4b
cd ~/bookstore
AG='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=agent'
API='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=api'
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2" 2>/dev/null; }
rc() { jp "$1" "{.status.containerStatuses[?(@.name==\"$2\")].restartCount}"; }
SQL() { kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc "$1" 2>/dev/null; }
agent_w2() { kubectl -n "$NS" get pods -l "$AG" --field-selector spec.nodeName=lab-k8s-worker2 -o jsonpath='{.items[0].metadata.name}'; }

# 1a. kubelet: container agent
A="$(agent_w2)"; AU="$(jp "$A" '{.metadata.uid}')"; AR="$(rc "$A" agent)"
kubectl -n "$NS" exec "$A" -c agent -- kill 1
for i in $(seq 1 60); do [ "$(rc "$A" agent)" -gt "$AR" ] && break; sleep 2; done
{ [ "$(rc "$A" agent)" -gt "$AR" ] && [ "$(jp "$A" '{.metadata.uid}')" = "$AU" ]; } \
  && echo 'PASS: kubelet khoi dong lai container agent, Pod giu nguyen UID'

# 1b. kubelet: PostgreSQL trong db-0
N0="$(SQL 'SELECT count(*) FROM api.books')"; DU="$(jp db-0 '{.metadata.uid}')"; DIP="$(jp db-0 '{.status.podIP}')"; DR="$(rc db-0 postgres)"
kubectl -n "$NS" exec db-0 -c postgres -- kill -INT 1
for i in $(seq 1 60); do [ "$(rc db-0 postgres)" -gt "$DR" ] && break; sleep 2; done
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=180s
kubectl -n "$NS" wait --for=condition=Ready pod -l "$API" --timeout=180s
N1="$(SQL 'SELECT count(*) FROM api.books')"
echo "truoc: uid=$DU ip=$DIP sach=$N0 ; sau: uid=$(jp db-0 '{.metadata.uid}') ip=$(jp db-0 '{.status.podIP}') sach=$N1" | tee "$EV/b8-kubelet-db.txt"
{ [ "$(jp db-0 '{.metadata.uid}')" = "$DU" ] && [ "$(jp db-0 '{.status.podIP}')" = "$DIP" ] && [ "$N1" = "$N0" ]; } \
  && echo 'PASS: container postgres khoi dong lai: giu UID, IP, du lieu; api Ready lai'

# 2. DaemonSet thay Pod đúng node
A="$(agent_w2)"; AU="$(jp "$A" '{.metadata.uid}')"
kubectl -n "$NS" delete pod "$A" --wait=true
for i in $(seq 1 60); do NA="$(agent_w2)"; [ -n "$NA" ] && [ "$NA" != "$A" ] && break; sleep 2; done
{ [ "$NA" != "$A" ] && [ "$(jp "$NA" '{.metadata.uid}')" != "$AU" ] && [ "$(jp "$NA" '{.spec.nodeName}')" = 'lab-k8s-worker2' ]; } \
  && echo "PASS: DaemonSet tao $NA thay $A, dung lab-k8s-worker2"

# 3. StatefulSet thay db-0
DU="$(jp db-0 '{.metadata.uid}')"; DIP="$(jp db-0 '{.status.podIP}')"
kubectl -n "$NS" delete pod db-0 --wait=true
for i in $(seq 1 120); do U="$(jp db-0 '{.metadata.uid}')"; [ -n "$U" ] && [ "$U" != "$DU" ] && break; sleep 2; done
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s
NIP="$(jp db-0 '{.status.podIP}')"; N2="$(SQL 'SELECT count(*) FROM api.books')"
echo "db-0 cu: uid=$DU ip=$DIP ; moi: uid=$(jp db-0 '{.metadata.uid}') ip=$NIP sach=$N2" | tee "$EV/b8-statefulset.txt"
{ [ "$(jp db-0 '{.metadata.uid}')" != "$DU" ] && [ "$(jp db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ]; } \
  && echo 'PASS: StatefulSet tao lai db-0 cung ten, khac UID, cung node (TD-01 da tra)'
[ "$N2" = '5' ] && echo 'PASS: du lieu migration mat cung emptyDir — con 5 cuon (TD-04)'
if [ "$NIP" != "$DIP" ]; then
  sleep 30
  kubectl -n "$NS" get pods -l "$API" -o wide | tee "$EV/b8-api.txt"
  [ "$(kubectl -n "$NS" get pods -l "$API" -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}' | grep -c True)" -eq 0 ] \
    && echo 'PASS: IP db-0 doi, api NotReady vi con tro IP cu (TD-03)'
else
  echo 'IP db-0 trung IP cu (hiem gap): api se tu Ready lai — ghi vao postmortem'
fi

# 4. Khôi phục bằng repo
./scripts/deploy.sh > "$EV/b8-deploy.txt" 2>&1
kubectl -n "$NS" rollout status deployment/api --timeout=300s
N3="$(SQL 'SELECT count(*) FROM api.books')"
WEB="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=web -o jsonpath='{.items[0].metadata.name}')"
API_IP="$(kubectl -n "$NS" get pod -l "$API" --field-selector=status.phase=Running -o jsonpath='{.items[0].status.podIP}')"
T="$(kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- "http://${API_IP}:3000/books" | grep -o '"title"' | wc -l)"
{ [ "${N3:-0}" -ge 8 ] && [ "$T" -ge 8 ]; } && echo "PASS: deploy.sh khoi phuc: $N3 cuon trong db, /books tra $T"
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. | Tầng | Ai xử lý | Theo trường | Giữ | Đổi |
   | --- | --- | --- | --- | --- |
   | Container | kubelet | `restartPolicy: Always` | Pod, UID, IP, `emptyDir` | tiến trình, `restartCount` |
   | Pod StatefulSet | StatefulSet controller | `replicas` + định danh | tên `db-0`, node (do `nodeSelector`) | UID, IP, `emptyDir` |
   | Pod DaemonSet | DaemonSet controller | tập node đủ điều kiện | node | tên, UID, IP |

2. Giờ tự lành: Pod `db` được tạo lại ngay, đúng tên — `TD-01` đã trả. Vẫn không tự lành: `api` trỏ
   IP cũ (`TD-03`, Lab 5a — Service headless và tên `db-0.db`); dữ liệu sau khởi tạo mất theo
   `emptyDir` (`TD-04`, nợ lab #2, Lab 6a — `volumeClaimTemplates`). Bài 38: Kubernetes khởi động lại
   được container, còn vấn đề của chính ứng dụng — trỏ sai địa chỉ, mất dữ liệu — phải xử lý riêng.
3. `deploy.sh` render lại `api` với IP mới (Deployment tự rollout vì template đổi) và chạy lại
   migration. Migration bất biến kết quả (B4) nên trên DB mới nó thêm đúng phần thiếu, trên DB cũ nó
   không thêm gì.

</details>

## B9. [BS-4B-09] Điều tra: báo cáo tồn kho song song

**Loại:** Điều tra — phần ⭐ là tùy chọn.

**Bối cảnh:** chủ hiệu sách muốn một báo cáo tồn kho chạy theo yêu cầu. Danh mục sẽ lớn dần, nên đội
muốn chia việc ra nhiều Pod chạy song song. Có ba mẫu trong bảng chọn của bài 349; chọn mẫu bằng thí
nghiệm, không bằng cảm giác.

**Yêu cầu:**

1. Indexed Job `bao-cao-ton-kho` ở `~/lab-work/4b/bao-cao.yaml.tpl` (chuỗi giữ chỗ `__DB_HOST__`,
   render giống `deploy.sh`). Chia `api.books` thành **3 phân vùng** theo `id`; mỗi Pod xử lý đúng một
   phân vùng, tối đa **2** Pod chạy cùng lúc.
2. Mỗi Pod in đúng một dòng `phan-vung=<i> so-sach=<n> ton-kho=<s>`.
3. Chứng minh: đủ 3 index hoàn thành; label chỉ số của mỗi Pod khớp phân vùng nó báo; tổng số sách và
   tổng tồn kho của ba phân vùng bằng số liệu truy vấn thẳng trên `db-0`; không lúc nào quá 2 Pod chạy.
4. ⭐ Cùng báo cáo bằng **khai triển template**: một template, ba Job `bao-cao-pv-0..2`, một label
   nhóm chung; xem log và xóa cả nhóm bằng một selector.
5. ⭐ Dựng **hình dạng** Job hàng đợi công việc của bài 67: một Job chỉ đặt mức song song 3, bỏ trống
   số lần hoàn thành; ghi lại API server điền gì cho trường bỏ trống và Job hoàn tất lúc nào.
6. Ghi ADR chọn mẫu cho báo cáo tồn kho. Xóa mọi Job thử nghiệm khi xong.

**Ràng buộc:** không thêm báo cáo vào repo `k8s/` ở sprint này — nó là thí nghiệm. Không tạo Service,
không dựng hàng đợi.

**Đọc lại:**

- Bài [349](../349-job-tasks-vi.md) — bảng chọn mẫu.
- Bài [353](../353-indexed-parallel-processing-vi.md):
  [*Chọn cách tiếp cận*](../353-indexed-parallel-processing-vi.md#chọn-cách-tiếp-cận-choose-an-approach),
  [*Định nghĩa một Indexed Job*](../353-indexed-parallel-processing-vi.md#định-nghĩa-một-indexed-job-define-an-indexed-job),
  [*Chạy Job*](../353-indexed-parallel-processing-vi.md#chạy-job-running-the-job).
- Bài [67](../67-job-vi.md): [*Thực thi song song cho Job*](../67-job-vi.md#parallel-jobs),
  [*Kiểm soát mức song song*](../67-job-vi.md#kiểm-soát-mức-song-song-controlling-parallelism),
  [*Chế độ hoàn thành*](../67-job-vi.md#completion-mode), [*Các mẫu sử dụng Job*](../67-job-vi.md#job-patterns).
- Bài [355](../355-parallel-processing-expansion-vi.md):
  [*Label trên Job và Pod*](../355-parallel-processing-expansion-vi.md#label-trên-job-và-pod-labels-on-jobs-and-pods),
  [*Các lựa chọn thay thế*](../355-parallel-processing-expansion-vi.md#các-lựa-chọn-thay-thế-alternatives).
- Bài [351](../351-coarse-parallel-work-queue-vi.md):
  [*Các cách thay thế*](../351-coarse-parallel-work-queue-vi.md#các-cách-thay-thế-alternatives),
  [*Lưu ý*](../351-coarse-parallel-work-queue-vi.md#lưu-ý-caveats).

**Bàn giao:** manifest ở `~/lab-work/4b/`; `docs/adr/NNNN-bao-cao-song-song.md`;
`~/lab-evidence/4b/b9-*.txt`.

**Nghiệm thu:**

- `PASS:` `completedIndexes` là `0-2`; mức song song quan sát được không vượt 2.
- `PASS:` label `batch.kubernetes.io/job-completion-index` của từng Pod khớp `phan-vung` trong log.
- `PASS:` tổng `so-sach` và tổng `ton-kho` bằng kết quả truy vấn thẳng.
- `PASS:` (⭐) ba Job `jobgroup=bao-cao-pv` đều thành công, tổng khớp; xóa bằng một lệnh theo label.
- `PASS:` (⭐) Job hàng đợi: `spec.completions` vẫn trống; Job `Complete`.
- `PASS:` không còn Job thử nghiệm nào.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-bao-cao-song-song.md`:

1. `completions: 3`, `parallelism: 2`: Job tạo tổng cộng bao nhiêu Pod, vì sao không tạo cả ba ngay?
2. So ba mẫu cho báo cáo tồn kho: số object Job, cần dịch vụ hàng đợi hay không, sửa chương trình
   hay không. Mẫu của bài 351 cần hai thứ gì mà cluster và lộ trình của bạn chưa có?
3. Mẫu hàng đợi của bài 351 đặt số lần hoàn thành bằng số phần tử. Đặt thừa hay thiếu thì sao?
4. Kỹ thuật nào của bài 355 bạn **đã** dùng hằng ngày trong dự án mà không gọi tên?

<details>
<summary>Gợi ý</summary>

- Biến `JOB_COMPLETION_INDEX` có sẵn trong mọi container của Indexed Job. Phân vùng:
  `WHERE id % 3 = ${JOB_COMPLETION_INDEX}`.
- `psql -tA -F ' ' -c "SELECT count(*), coalesce(sum(stock), 0) FROM ..."` in hai số cách nhau một
  dấu cách; `set -- $R` tách chúng thành `$1`, `$2`.
- Theo dõi mức song song: đọc `.status.active` của Job mỗi giây cho tới khi Job `Complete`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự viết `bao-cao.yaml.tpl`</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4b; W=~/lab-work/4b; J=bao-cao-ton-kho
jp() { kubectl -n "$NS" get job "$1" -o jsonpath="$2" 2>/dev/null; }
DB_IP="$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.podIP}')"
sum_log() { awk '{for(i=1;i<=NF;i++){split($i,a,"="); if(a[1]=="so-sach") n+=a[2]; if(a[1]=="ton-kho") s+=a[2]}} END{print n+0, s+0}' "$1"; }
TRUTH="$(kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -tA -F' ' -c 'SELECT count(*), coalesce(sum(stock),0) FROM api.books')"

sed "s/__DB_HOST__/${DB_IP}/g" "$W/bao-cao.yaml.tpl" | kubectl apply -f -
MAXA=0
for i in $(seq 1 180); do
  A="$(jp "$J" '{.status.active}')"; [ "${A:-0}" -gt "$MAXA" ] && MAXA="${A:-0}"
  [ "$(jp "$J" '{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ] && break
  sleep 1
done
echo "completedIndexes=$(jp "$J" '{.status.completedIndexes}') maxActive=$MAXA" | tee "$EV/b9-indexed.txt"
[ "$(jp "$J" '{.status.completedIndexes}')" = '0-2' ] && echo 'PASS: du 3 index 0-2'
{ [ "$MAXA" -ge 1 ] && [ "$MAXA" -le 2 ]; } && echo "PASS: muc song song cao nhat $MAXA <= 2"

BAD=0; : > "$EV/b9-bao-cao.txt"
for P in $(kubectl -n "$NS" get pods -l batch.kubernetes.io/job-name="$J" -o jsonpath='{.items[*].metadata.name}'); do
  IDX="$(kubectl -n "$NS" get pod "$P" -o jsonpath='{.metadata.labels.batch\.kubernetes\.io/job-completion-index}')"
  LOG="$(kubectl -n "$NS" logs "$P")"; echo "$LOG" >> "$EV/b9-bao-cao.txt"
  echo "$LOG" | grep -q "^phan-vung=$IDX " || BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: label chi so cua moi Pod khop phan vung no bao'
echo "tong tu bao cao: $(sum_log "$EV/b9-bao-cao.txt") ; truy van thang: $TRUTH"
[ "$(sum_log "$EV/b9-bao-cao.txt")" = "$TRUTH" ] && echo 'PASS: tong ba phan vung bang so lieu truy van thang'
kubectl -n "$NS" delete job "$J" --wait=true

# ⭐ Khai triển template: tự viết bao-cao-pv.yaml.tpl với $PV và __DB_HOST__
if [ -f "$W/bao-cao-pv.yaml.tpl" ]; then
  mkdir -p "$W/jobs"
  for PV in 0 1 2; do sed -e "s/\$PV/$PV/g" -e "s/__DB_HOST__/${DB_IP}/g" "$W/bao-cao-pv.yaml.tpl" > "$W/jobs/bao-cao-pv-$PV.yaml"; done
  kubectl create -f "$W/jobs"
  for PV in 0 1 2; do kubectl -n "$NS" wait --for=condition=Complete "job/bao-cao-pv-$PV" --timeout=180s; done
  kubectl -n "$NS" logs -l jobgroup=bao-cao-pv > "$EV/b9-template.txt"
  { [ "$(kubectl -n "$NS" get jobs -l jobgroup=bao-cao-pv --no-headers | wc -l)" -eq 3 ] && \
    [ "$(sum_log "$EV/b9-template.txt")" = "$TRUTH" ]; } && echo 'PASS: ba Job tu mot template, tong khop'
  kubectl -n "$NS" delete job -l jobgroup=bao-cao-pv --wait=true && echo 'PASS: mot lenh theo label xoa ca nhom'
  rm -rf "$W/jobs"
fi

# ⭐ Hình dạng Job hàng đợi: tự viết hang-doi-thu.yaml
if [ -f "$W/hang-doi-thu.yaml" ]; then
  kubectl apply -f "$W/hang-doi-thu.yaml"
  kubectl -n "$NS" wait --for=condition=Complete job/hang-doi-thu --timeout=180s
  { [ -z "$(jp hang-doi-thu '{.spec.completions}')" ] && [ "$(jp hang-doi-thu '{.spec.parallelism}')" = '3' ]; } \
    && echo 'PASS: bo trong completions thi API server khong dien mac dinh; Job Complete'
  kubectl -n "$NS" delete job hang-doi-thu --wait=true
fi

[ "$(kubectl -n "$NS" get jobs -l '!app.kubernetes.io/part-of' --no-headers 2>/dev/null | wc -l)" -eq 0 ] && echo 'PASS: khong con Job thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/4b/bao-cao.yaml.tpl
apiVersion: batch/v1
kind: Job
metadata:
  name: bao-cao-ton-kho
  namespace: bookstore
spec:
  completionMode: Indexed
  completions: 3
  parallelism: 2
  backoffLimit: 2
  ttlSecondsAfterFinished: 900
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: bao-cao
        image: postgres:17.11
        env:
        - {name: PGHOST, value: "__DB_HOST__"}
        - {name: PGUSER, value: postgres}
        - {name: PGDATABASE, value: bookstore}
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef: {name: db-credentials, key: postgres-password}
        command:
        - sh
        - -c
        - |
          set -e
          R="$(psql -tA -F ' ' -c "SELECT count(*), coalesce(sum(stock), 0) FROM api.books WHERE id % 3 = ${JOB_COMPLETION_INDEX}")"
          set -- $R
          echo "phan-vung=${JOB_COMPLETION_INDEX} so-sach=$1 ton-kho=$2"
        resources:
          requests: {cpu: 20m, memory: 32Mi}
          limits: {cpu: 100m, memory: 64Mi}
```

⭐ `bao-cao-pv.yaml.tpl`: cùng container, bỏ `completionMode`/`completions`/`parallelism`, đặt
`name: bao-cao-pv-$PV`, label `jobgroup: bao-cao-pv` ở **cả** `metadata` của Job lẫn
`template.metadata`, và trong SQL thay `${JOB_COMPLETION_INDEX}` bằng `$PV`.

⭐ `hang-doi-thu.yaml`: Job `busybox` chỉ có `parallelism: 3`, lệnh
`echo "worker $(hostname) het viec"; sleep 5`.

Ý chính cho ADR:

1. Tổng 3 Pod — `completions` quyết định tổng số, `parallelism` quyết định số chạy cùng lúc. Vì
   `parallelism` nhỏ hơn, control plane chờ một Pod xong rồi mới chạy Pod thứ ba — không phải vì
   cluster thiếu chỗ.
2. Indexed: một Job, không hàng đợi, không sửa chương trình — chỉ cần chỉ số. Khai triển template: ba
   object Job phải quản lý, tạo cả lô lớn gây tải control plane (bài 355). Hàng đợi (bài 351): một
   Job nhưng cần **dịch vụ hàng đợi chạy trong cluster** (Service, giai đoạn 5) và **image worker tự
   build** đẩy lên registry. Với 3 phân vùng tĩnh, Indexed là đủ và rẻ nhất.
3. Thiếu: còn phần tử không ai xử lý. Thừa: Job không bao giờ có vẻ hoàn tất, các Pod thừa bị chặn
   chờ thông điệp không bao giờ tới.
4. `deploy.sh` khai triển `__DB_HOST__` bằng `sed` trên mọi file `.tpl` — file `.tpl` không phải
   manifest hợp lệ cho tới khi được khai triển, đúng như `job-tmpl.yaml` của bài 355.

</details>

## B10. [BS-4B-10] Thiết kế HPA cho `web`

**Loại:** Điều tra — sản phẩm là tài liệu.

**Bối cảnh:** yêu cầu `R16` của dự án là tầng web tự co giãn theo tải. Cơ chế đã đọc ở bài 71–73,
nhưng cluster chưa có Metrics API — metrics-server thuộc giai đoạn 11. Sprint này **thiết kế** HPA và
chứng minh vì sao chưa chạy được; Lab 11b áp nó. Đây là nợ lab #1.

**Yêu cầu:**

1. File thiết kế `docs/hpa-web.draft.yaml`: một HorizontalPodAutoscaler cho Deployment `web`, 2–5 bản
   sao theo [mục 3.2 của file dự án](DU-AN-BOOKSTORE.md#32-thành-phần-ứng-dụng), co giãn theo mức dùng
   CPU tính trên `requests`, thu nhỏ chậm hơn mở rộng. **Không** apply.
2. API server chấp nhận thiết kế đó (kiểm bằng server dry-run) — HPA là API lõi.
3. Chứng minh ba giới hạn trên cluster hiện tại: không có Metrics API; VPA không có trong API;
   DaemonSet `node-agent` không có subresource `scale` trong khi Deployment `web` có.
4. Chứng minh mọi container của Pod `web`, kể cả sidecar, có `requests.cpu` — điều kiện để
   `Utilization` tính được.
5. ADR `docs/adr/NNNN-hpa-web.md`.

**Ràng buộc:** không cài metrics-server, không cài VPA, không tạo HPA thật. Không bỏ `replicas` khỏi
Deployment `web` ở sprint này.

**Đọc lại:**

- Bài [71](../71-autoscaling-vi.md):
  [*Co giãn workload theo chiều ngang*](../71-autoscaling-vi.md#scaling-workloads-horizontally),
  [*Co giãn workload theo chiều dọc*](../71-autoscaling-vi.md#scaling-workloads-vertically).
- Bài [72](../72-horizontal-pod-autoscale-vi.md):
  [*HorizontalPodAutoscaler hoạt động như thế nào?*](../72-horizontal-pod-autoscale-vi.md#horizontalpodautoscaler-hoạt-động-như-thế-nào-how-does-a-horizontalpodautoscaler-work),
  [*Chi tiết thuật toán*](../72-horizontal-pod-autoscale-vi.md#algorithm-details),
  [*Hỗ trợ metric tài nguyên*](../72-horizontal-pod-autoscale-vi.md#hỗ-trợ-metric-tài-nguyên-support-for-resource-metrics),
  [*Hỗ trợ các API metric*](../72-horizontal-pod-autoscale-vi.md#support-for-metrics-apis).
- Bài [73](../73-vertical-pod-autoscale-vi.md):
  [*Đối tượng API*](../73-vertical-pod-autoscale-vi.md#đối-tượng-api-api-object),
  [*Recreate*](../73-vertical-pod-autoscale-vi.md#updateMode-Recreate).

**Bàn giao:** `docs/hpa-web.draft.yaml`; `docs/adr/NNNN-hpa-web.md`; `~/lab-evidence/4b/b10-*.txt`.

**Nghiệm thu:**

- `PASS:` server dry-run chấp nhận `docs/hpa-web.draft.yaml`; đích là `Deployment/web`, min ≥ 2,
  max ≤ 5; metric CPU kiểu `Utilization`.
- `PASS:` namespace `bookstore` không có HPA nào.
- `PASS:` `nginx` và `status-writer` của `web` đều có `requests.cpu`.
- `PASS:` Metrics API không tồn tại; VPA không có trong API; HPA có.
- `PASS:` Deployment `web` có subresource `scale`, DaemonSet `node-agent` thì không.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-hpa-web.md`:

1. Mục tiêu của bạn là X% CPU. Ba bản sao đang dùng trung bình gấp đôi X. HPA đặt bao nhiêu bản sao,
   và giới hạn nào của bạn chặn lại? Nếu mức đo chỉ vượt mục tiêu 5% thì sao?
2. Vì sao `web` phải bỏ `replicas` khỏi manifest khi HPA vào ở Lab 11b, và vì sao chưa bỏ bây giờ?
3. Vì sao không giao `db` cho VPA chế độ `Recreate`? Nói bằng số bản sao của `db` và `TD-04`.
4. Một thành viên đề xuất HPA cho `node-agent` "khi node bận". Trả lời bằng kết quả bước 3.

<details>
<summary>Gợi ý</summary>

- API version ổn định của HPA là `autoscaling/v2`; khối `metrics` kiểu `Resource`.
- `kubectl create --dry-run=server -f <file>` gửi object tới API server mà không lưu.
- Liệt kê API theo nhóm: `kubectl api-resources --api-group=<nhóm> -o name`.
- Subresource `scale`: `kubectl get --raw /apis/apps/v1/namespaces/bookstore/deployments/web/scale`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/4b
cd ~/bookstore
F=docs/hpa-web.draft.yaml
OUT="$(kubectl create --dry-run=server -f "$F" -o jsonpath='{.spec.scaleTargetRef.kind}/{.spec.scaleTargetRef.name} {.spec.minReplicas} {.spec.maxReplicas} {.spec.metrics[0].resource.name}/{.spec.metrics[0].resource.target.type}' 2>"$EV/b10-dry-run-err.txt")"
echo "$OUT" | tee "$EV/b10-dry-run.txt"
set -- $OUT
{ [ "$1" = 'Deployment/web' ] && [ "${2:-0}" -ge 2 ] && [ "${3:-9}" -le 5 ] && [ "$4" = 'cpu/Utilization' ]; } \
  && echo 'PASS: API server chap nhan thiet ke HPA web, 2-5 ban sao, CPU Utilization'
[ "$(kubectl -n "$NS" get hpa --no-headers 2>/dev/null | wc -l)" -eq 0 ] && echo 'PASS: chua tao HPA that'
ls docs/adr/*-hpa-web.md >/dev/null 2>&1 && echo 'PASS: co ADR hpa-web'

{ [ -n "$(kubectl -n "$NS" get deployment web -o jsonpath='{.spec.template.spec.containers[?(@.name=="nginx")].resources.requests.cpu}')" ] && \
  [ -n "$(kubectl -n "$NS" get deployment web -o jsonpath='{.spec.template.spec.initContainers[?(@.name=="status-writer")].resources.requests.cpu}')" ]; } \
  && echo 'PASS: nginx va sidecar status-writer deu co requests.cpu'

kubectl get --raw /apis/metrics.k8s.io/v1beta1 >/dev/null 2>&1 \
  && echo 'FAIL: da co Metrics API — cluster lech baseline' || echo 'PASS: chua co Metrics API (no lab #1)'
[ -z "$(kubectl api-resources --api-group=autoscaling.k8s.io -o name 2>/dev/null)" ] && echo 'PASS: VPA khong nam trong API'
kubectl api-resources --api-group=autoscaling -o name | grep -qx 'horizontalpodautoscalers.autoscaling' && echo 'PASS: HPA la API loi'

kubectl get --raw "/apis/apps/v1/namespaces/$NS/deployments/web/scale" > "$EV/b10-scale-web.json" \
  && ! kubectl get --raw "/apis/apps/v1/namespaces/$NS/daemonsets/node-agent/scale" >/dev/null 2>&1 \
  && echo 'PASS: Deployment co subresource scale, DaemonSet thi khong'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# docs/hpa-web.draft.yaml — BẢN THIẾT KẾ, KHÔNG apply. Áp ở Lab 11b (nợ lab #1).
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
```

Ý chính cho ADR:

1. `ceil(3 × 2X / X) = 6`, bị `maxReplicas: 5` chặn còn 5. Vượt 5% thì tỷ lệ 1.05 nằm trong dung sai
   mặc định 0.1 — không làm gì. `Utilization` tính trên `requests`, nên con số `requests` đã chốt ở 3c
   quyết định lúc nào HPA thấy "bận".
2. Bài 72: còn `replicas` trong manifest thì mỗi lần `kubectl apply` kéo số Pod về giá trị đó, đánh
   nhau với HPA. Bỏ bây giờ thì Deployment về mặc định 1 bản sao mà chưa có HPA nào nâng lên — mất
   tính sẵn sàng và PDB `web` mất ý nghĩa.
3. `db` có đúng một bản sao. `Recreate` evict Pod để áp `requests` mới (bài 73): mỗi lần là một lần
   `db-0` bị thay — mất dịch vụ, và với `emptyDir` là mất dữ liệu sau khởi tạo (`TD-04`). Thêm vào đó
   VPA là add-on không có trong API (bước 3).
4. HPA ghi số bản sao qua subresource `scale`; DaemonSet không có subresource đó — số Pod của nó do
   tập node quyết định (bài 72: HPA không áp dụng cho đối tượng không co giãn được).

</details>

## B11. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 5, rồi trả cluster về đúng
`01-cluster-ready`.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 4b — kiểm thử khói **tích lũy**, chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 4b: StatefulSet db, Deployment web/api, Job db-migrate,
# CronJob db-backup, DaemonSet node-agent. Kiểm mọi hợp đồng còn hiệu lực tới sprint 5.
set -u
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

# 1. Namespace và object cấp cluster
[ "$(kubectl get namespace "$NS" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = 'bookstore' ] \
  && pass 'namespace bookstore mang label du an' || fail 'namespace bookstore thieu hoac thieu label'
PRIO="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
[ -n "$PRIO" ] && pass "co PriorityClass bookstore-critical ($PRIO)" || fail 'thieu PriorityClass bookstore-critical'

# 2. Cấu hình, bí mật (3b) và PDB (3c)
for CM in web-config api-config db-init; do
  k get configmap "$CM" >/dev/null 2>&1 && pass "co ConfigMap $CM" || fail "thieu ConfigMap $CM"
done
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials du hai key' || fail 'Secret db-credentials thieu key'
for P in web api; do
  k get pdb "$P" >/dev/null 2>&1 && pass "co PodDisruptionBudget $P" || fail "thieu PodDisruptionBudget $P"
done

# 3. Controller cho mọi thành phần (TD-01 đã trả)
[ "$(jp statefulset/db '{.status.readyReplicas}')" = '1' ] && pass 'StatefulSet db 1/1 Ready' || fail 'StatefulSet db chua Ready'
[ "$(jp statefulset/db '{.spec.serviceName}')" = 'db' ] && pass 'StatefulSet db khai serviceName db' || fail 'serviceName khong phai db'
[ "$(jp pod/db-0 '{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}')" = 'StatefulSet/db' ] \
  && pass 'db-0 thuoc StatefulSet db' || fail 'db-0 khong thuoc StatefulSet db'
[ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && pass 'db-0 tren lab-k8s-worker2' || fail 'db-0 khong o lab-k8s-worker2'
[ "$(jp pod/db-0 '{.status.qosClass}')" = 'Guaranteed' ] && pass 'db-0 QoS Guaranteed' || fail 'db-0 khong Guaranteed'
[ -n "$PRIO" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$PRIO" ] && pass 'db-0 dung bookstore-critical' || fail 'db-0 sai priority'
for X in web:3 api:2; do
  N="${X%%:*}"; WANT="${X##*:}"
  [ "$(jp "deployment/$N" '{.status.readyReplicas}')" = "$WANT" ] && pass "Deployment $N $WANT/$WANT Ready" \
    || fail "Deployment $N chua du $WANT ban sao Ready"
done
ORPHAN="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"

# 4. Mọi container chính có requests và limits (3c)
MISS="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container co requests va limits' || fail "container thieu tai nguyen: $(echo $MISS)"

# 5. Dữ liệu và migration
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach (khoi tao + migration)" || fail "db co ${N:-0} cuon sach, can >= 8"
if k get job db-migrate >/dev/null 2>&1; then
  [ "$(jp job/db-migrate '{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ] \
    && pass 'Job db-migrate Complete' || fail 'Job db-migrate chua Complete'
else
  pass 'Job db-migrate da duoc TTL don, ket qua nam trong db'
fi

# 6. web và api
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
API_IP="$(k get pod -l "$SEL,app.kubernetes.io/component=api" --field-selector=status.phase=Running -o jsonpath='{.items[0].status.podIP}' 2>/dev/null)"
get() { k exec "$WEB" -c status-writer -- wget -qO- "$1" 2>/dev/null; }
T="$(get "http://${API_IP}:3000/books" | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "api /books tra $T cuon sach" || fail "api /books tra $T cuon sach"
get http://127.0.0.1:8080/ | grep -q 'Bookstore' && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
get http://127.0.0.1:8080/status.json | grep -Eq "\"pod\" *: *\"$WEB\"" \
  && pass 'web phuc vu status.json dung ten Pod' || fail 'status.json sai hoac thieu'

# 7. CronJob db-backup: chính sách và một lần chạy thử
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = 'Forbid' ] && pass 'db-backup Forbid' || fail 'db-backup khong Forbid'
SDS="$(jp cronjob/db-backup '{.spec.startingDeadlineSeconds}')"
[ "${SDS:-0}" -ge 10 ] && pass "db-backup startingDeadlineSeconds=$SDS" || fail 'db-backup thieu startingDeadlineSeconds >= 10'
[ "$(jp cronjob/db-backup '{.spec.suspend}')" != 'true' ] && pass 'db-backup khong bi tam ngung' || fail 'db-backup dang suspend'
J="smoke-backup-$(date +%s)"
if k create job "$J" --from=cronjob/db-backup >/dev/null 2>&1 \
   && k wait --for=condition=Complete "job/$J" --timeout=180s >/dev/null 2>&1 \
   && k logs "job/$J" 2>/dev/null | grep -q '^db-backup: OK'; then
  pass 'chay thu db-backup thanh cong'
else
  fail 'chay thu db-backup that bai'
fi
k delete job "$J" --ignore-not-found --wait=true >/dev/null 2>&1

# 8. DaemonSet node-agent: mọi node, cổng http 9100, /node.json
NODES="$(kubectl get nodes --no-headers | wc -l)"
R="$(jp daemonset/node-agent '{.status.numberReady}')"
[ "${R:-0}" -eq "$NODES" ] && pass "node-agent Ready tren $R/$NODES node" || fail "node-agent ${R:-0}/$NODES node"
[ "$(jp daemonset/node-agent '{.spec.template.spec.containers[?(@.name=="agent")].ports[?(@.name=="http")].containerPort}')" = '9100' ] \
  && pass 'node-agent cong http 9100' || fail 'node-agent sai cong'
BAD=0
for L in $(k get pod -l "$SEL,app.kubernetes.io/component=agent" -o jsonpath='{range .items[*]}{.spec.nodeName}={.status.podIP}{"\n"}{end}' 2>/dev/null); do
  get "http://${L##*=}:9100/node.json" | grep -Eq "\"node\" *: *\"${L%%=*}\"" || { echo "FAIL: node-agent tren ${L%%=*} sai /node.json"; BAD=1; }
done
[ "$BAD" -eq 0 ] && pass 'moi node-agent tra /node.json dung ten node' || FAIL=1

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

2. Cập nhật `docs/tech-debt.md`: `TD-01` đổi trạng thái thành `đã trả` (4b, bằng chứng
   `~/lab-evidence/4b/b8-statefulset.txt`); `TD-08` đã thêm ở B6. Ghi chú ở `TD-03`: từ sprint này
   `deploy.sh` render cả Job và CronJob với IP `db-0`.

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/4b/b11-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/4b/b11-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/4b/b11-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-4b: sprint 5 — StatefulSet db, Job, CronJob, DaemonSet'
git tag lab-4b
git bundle create ~/bookstore-lab-4b.bundle --all && git bundle verify ~/bookstore-lab-4b.bundle
```

Chép `~/bookstore-lab-4b.bundle` về máy host bằng `scp` trước khi đi tiếp.

5. Dọn object thử nghiệm và trả cluster:

```bash
kubectl -n bookstore delete statefulset,cronjob,job,pod -l '!app.kubernetes.io/part-of' --ignore-not-found --wait=true
rm -rf ~/lab-work/4b
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối:

```bash
cd ~/bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-4b' && echo 'PASS: HEAD mang tag lab-4b'
test -s ~/bookstore-lab-4b.bundle && echo 'PASS: co bundle'
{ test ! -e k8s/app/db/pod.yaml && test -f k8s/app/db/statefulset.yaml && test -f k8s/app/jobs/db-migrate.yaml.tpl \
  && test -f k8s/app/jobs/db-backup.yaml.tpl && test -f k8s/app/agent/daemonset.yaml; } && echo 'PASS: bo cuc repo dung muc 5.5'
grep -E 'TD-01.*đã trả' docs/tech-debt.md >/dev/null && grep -q 'TD-08' docs/tech-debt.md && echo 'PASS: tech-debt.md cap nhat TD-01, TD-08'
BAD=0
for F in 'adr/*-db-statefulset.md' 'adr/*-job-migrate.md' 'adr/*-db-backup.md' 'adr/*-node-agent.md' \
         'adr/*-bao-cao-song-song.md' 'adr/*-hpa-web.md' 'postmortem/4b-statefulset-rollout-ket.md' 'postmortem/4b-tu-phuc-hoi.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR va postmortem cua sprint 5'
{ [ "$(kubectl get namespace,priorityclass -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  [ "$(kubectl get all -A -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an'
test ! -e ~/lab-work/4b && echo 'PASS: da don lab-work'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; tám dòng `PASS:` đầu của gate cuối
xuất hiện; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica; namespace `default` không có Pod. Thư mục `~/lab-evidence/4b/` **giữ lại**.
Cluster trở về `01-cluster-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 4b

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] `db` chỉ một bản sao. StatefulSet cho nó thứ gì mà Deployment không cho? Định danh của `db-0`
      gồm những phần nào, phần nào chưa hoạt động trên cluster của bạn và vì sao?
- [ ] Vì sao không scale `db` lên hai bản sao? `db-1` sẽ là gì?
- [ ] Ở B8 bạn xóa `db-0`. Ai tạo lại, tên, UID, node, IP, dữ liệu — cái nào giữ, cái nào mất? Nợ
      nào giải thích phần mất, trả ở lab nào?
- [ ] `OrderedReady` trên `sts-thu`: thứ tự tạo, thứ tự kết thúc, và khi nào thu nhỏ bị chặn?
      `kubectl apply` và `kubectl scale` — cái nào thắng?
- [ ] Rollout `sts-thu` kẹt vì image lỗi: Pod nào bị cập nhật trước, vì sao hoàn nguyên template chưa
      đủ, quy trình gỡ kẹt ba bước là gì?
- [ ] Xóa êm và `--grace-period=0 --force` khác nhau ở đâu? Vì sao không bao giờ làm điều đó với `db-0`
      khi dữ liệu đã bền?
- [ ] `db-migrate` phải bất biến kết quả vì những tình huống nào? `backoffLimit` và
      `activeDeadlineSeconds` — cái nào chặn trước trong trường hợp nào, `reason` là gì?
- [ ] `restartPolicy: Never` và `OnFailure` trong Job khác nhau thế nào khi container thoát mã khác 0?
      Job `Failed` có tự chạy lại không? `deploy.sh` xử lý điều đó ra sao?
- [ ] TTL của Job đếm từ lúc nào, xóa những gì? Vì sao TTL của `db-migrate` phải đủ dài?
- [ ] `db-backup`: `Forbid` áp dụng trong phạm vi nào? `startingDeadlineSeconds` đổi hai hành vi nào?
      Vì sao bản dump hiện tại chưa phải backup (`TD-08`)?
- [ ] `node-agent`: vì sao ba Pod, ai bind Pod vào node, toleration nào controller tự thêm và cái nào
      chỉ dành cho Pod `hostNetwork`? Pod thay thế phải lên node nào?
- [ ] Báo cáo tồn kho: vì sao Indexed Job thay vì hàng đợi hay khai triển template? HPA cho `web`: công
      thức, vì sao `Utilization` cần `requests`, và vì sao HPA chưa chạy được trên cluster này?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Nợ lab #1 (HPA, VPA — Lab 11b cho HPA; VPA chưa có lab), #2 (`volumeClaimTemplates` — Lab 6a),
  #3 (Service headless `db` — Lab 5a).
- Nợ kỹ thuật: `TD-03` (IP Pod chèn lúc triển khai, giờ áp cho cả Job và CronJob — Lab 5a), `TD-04`
  (dữ liệu trên `emptyDir` — Lab 6a), `TD-06` (`db` khởi động bằng root — Lab 9b), `TD-07` (quyền
  debug — Lab 9a), `TD-08` mới (bản dump chưa có nơi bền — Lab 6a). `TD-01` **đã trả**.

### Bài giải thích cuối cùng

Trong vài phút, kể lại cuộc đời của `db-0` và các tác vụ quanh nó: từ lúc `scripts/deploy.sh` apply
StatefulSet — ai tạo Pod, định danh nó nhận những gì; `deploy.sh` chờ gì rồi lấy IP để render `api`,
`db-migrate`, `db-backup`; migration chạy, thử lại và được dọn thế nào; một lần backup đi từ CronJob
tới Pod ra sao. Rồi kể kịch bản B8: `db-0` bị xóa — controller nào phản ứng, cái gì quay lại đúng, cái
gì hỏng và vì sao `deploy.sh` cứu được. Kết thúc bằng việc chỉ ra Lab 5a và Lab 6a sẽ thay đổi câu
chuyện này ở điểm nào.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `git describe` không ra `lab-4a` | Repo đang ở commit khác, hoặc clone từ bundle cũ | `git checkout lab-4a` hoặc clone lại từ `~/bookstore-lab-4a.bundle` mới nhất |
| B1: `deploy.sh` treo ở `rollout status statefulset/db` | `db-0` `Pending` vì `nodeSelector` sai, hoặc `CreateContainerConfigError` vì tên ConfigMap, Secret sai khi chép spec | `kubectl -n bookstore describe pod db-0`; so khối `template.spec` với `pod.yaml` ở tag `lab-4a` |
| B1: `deploy.sh` báo `spec: Forbidden: updates to statefulset spec for fields other than ...` | Sửa `selector` hoặc `serviceName` của StatefulSet đã tạo | Hai trường đó bất biến: `./scripts/teardown.sh && ./scripts/deploy.sh` |
| B1: Pod trần `db` vẫn chạy song song `db-0` | `kubectl apply` không gỡ object đã bỏ khỏi repo | `deploy.sh` của lab xóa Pod `db`; hoặc teardown rồi deploy |
| B1, B8: `api` `CrashLoopBackOff`, log báo lỗi xác thực mật khẩu | `secrets/db.env` bị sinh lại trong khi `db-0` vẫn giữ thư mục dữ liệu cũ với mật khẩu cũ | Xóa `db-0` để dữ liệu khởi tạo lại với Secret mới, rồi `./scripts/deploy.sh` |
| B2: gate đầu fail vì `sts-thu-0` đã Ready | Chạy chậm giữa `apply` và lần đo | `kubectl -n bookstore delete statefulset sts-thu --wait=true` rồi chạy lại script liền mạch |
| B2: thu nhỏ không bị chặn | `sts-thu-1` chưa kịp mất Ready trước lệnh scale | Giữ dòng `kubectl wait --for=condition=Ready=false`; kiểm probe đọc đúng file `/tmp/ready` |
| B3: sau `rollout undo`, `sts-thu-2` tự về image tốt | Hành vi khác mô tả của bài 65 ở phiên bản đang chạy | Không phải lỗi thao tác — ghi lại vào postmortem kèm `kubectl -n bookstore get controllerrevision` |
| B4: `deploy.sh` dừng ở `db-migrate that bai`, log `password authentication failed` | Sai tên key của Secret, hoặc đọc nhầm key `authenticator-password` | `secretKeyRef.key` phải là `postgres-password` |
| B4: số sách tăng sau mỗi lần deploy | Migration không bất biến kết quả | Unique index trên `(title, author)` cộng `ON CONFLICT DO NOTHING` |
| B4: `apply` file `.tpl` báo `field is immutable` | Apply đè lên Job cũ với template khác | Không apply `.tpl` bằng tay; `deploy.sh` xóa Job cũ trước |
| B4: `deploy.sh` báo không đọc được trạng thái `db-migrate` | TTL quá ngắn, Job bị dọn trước khi `wait_job` kịp đọc | Đặt TTL ≥ 600 giây như yêu cầu |
| B5: `mig-cham` mãi chưa có condition `Failed` | Condition cuối của Job chỉ được thêm khi mọi Pod đã kết thúc (bài 67); `sleep` ở PID 1 bỏ qua `SIGTERM` | Đặt `terminationGracePeriodSeconds` nhỏ cho Pod của `mig-cham` |
| B6: Job backup `Failed`, log `pg_dump: error: connection ... timeout` | `db-0` đã đổi IP sau lần render CronJob gần nhất (`TD-03`) | `./scripts/deploy.sh` để render lại; Job đang chạy dở vẫn giữ IP cũ |
| B6: thấy hai Job của `backup-cham` | Đo sau khi Job đầu đã xong và `startingDeadlineSeconds` cho chạy tiếp | Xóa CronJob, tạo lại, đo ngay sau khi Job đầu `active` |
| B7: chỉ hai Pod `node-agent` | Thiếu toleration cho taint control plane | Thêm toleration `node-role.kubernetes.io/control-plane`, effect `NoSchedule` |
| B7: `node-agent` `CrashLoopBackOff`, log `can't create /www/...: Permission denied` | Ghi vào filesystem image khi chạy uid 65534 | Mount `emptyDir` tại `/www` |
| B7: `wget` tới agent trên node khác bị timeout | Mạng Pod xuyên node hỏng | Sự cố môi trường — chạy lại tầng 3 của gate A5.4 ở Lab 00 |
| B8: `api` vẫn Ready sau khi xóa `db-0` | `db-0` mới tình cờ nhận lại IP cũ | Xóa `db-0` lần nữa và chạy lại phần 3 của script |
| B9: `completedIndexes` thiếu một index | Một Pod `psql` lỗi (biến môi trường, mật khẩu) | `kubectl -n bookstore logs` Pod thất bại; sửa template rồi tạo lại Job |
| B10: dry-run HPA báo `no matches for kind "HorizontalPodAutoscaler"` | `apiVersion` cũ đã bị gỡ | Dùng `autoscaling/v2` |
| B11: `teardown.sh` chờ lâu | `db-0` dùng hết 60 giây ân hạn khi thu về 0 | Chờ; script chờ tối đa 180 giây cho `db-0` và 300 giây cho namespace |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — StatefulSets](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes v1.35 — DaemonSet](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Kubernetes v1.35 — Jobs](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes v1.35 — Automatic Cleanup for Finished Jobs](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
- [Kubernetes v1.35 — CronJob](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
- [Kubernetes v1.35 — Autoscaling Workloads](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/autoscaling/)
- [Kubernetes v1.35 — Horizontal Pod Autoscaling](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/autoscaling/horizontal-pod-autoscale/)
- [Kubernetes v1.35 — Vertical Pod Autoscaling](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/autoscaling/vertical-pod-autoscale/)
- [Kubernetes v1.35 — Kubernetes Self-Healing](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/self-healing/)
- [Kubernetes v1.35 — Scale a StatefulSet](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/scale-stateful-set/)
- [Kubernetes v1.35 — Force Delete StatefulSet Pods](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/force-delete-stateful-set-pod/)
- [Kubernetes v1.35 — Run Jobs](https://v1-35.docs.kubernetes.io/docs/tasks/job/)
- [Kubernetes v1.35 — Running Automated Tasks with a CronJob](https://v1-35.docs.kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/)
- [Kubernetes v1.35 — Coarse Parallel Processing Using a Work Queue](https://v1-35.docs.kubernetes.io/docs/tasks/job/coarse-parallel-processing-work-queue/)
- [Kubernetes v1.35 — Indexed Job for Parallel Processing with Static Work Assignment](https://v1-35.docs.kubernetes.io/docs/tasks/job/indexed-parallel-processing-static/)
- [Kubernetes v1.35 — Parallel Processing using Expansions](https://v1-35.docs.kubernetes.io/docs/tasks/job/parallel-processing-expansion/)
- [Kubernetes v1.35 — Use Cascading Deletion in a Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/use-cascading-deletion/)
- [Kubernetes — StatefulSet API reference](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/stateful-set-v1/)
- [Kubernetes — `kubectl create job`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_job/)
- [PostgreSQL 17 — psql](https://www.postgresql.org/docs/17/app-psql.html)
- [PostgreSQL 17 — pg_dump](https://www.postgresql.org/docs/17/app-pgdump.html)
- [PostgreSQL 17 — INSERT, mục ON CONFLICT](https://www.postgresql.org/docs/17/sql-insert.html)
- [PostgreSQL 17 — Biến môi trường của libpq](https://www.postgresql.org/docs/17/libpq-envars.html)
