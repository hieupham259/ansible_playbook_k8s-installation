# Lab 3a — Pod và vòng đời

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> Repo dự án chưa tồn tại; lab này khởi tạo nó.
> **Điểm kết thúc:** teardown trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới. Repo
> `~/bookstore` có tag `lab-3a` và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 1 — Pod hóa ba thành phần của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 2 — Container, image, CRI và cgroup](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng mục [3a. Pod và vòng đời](../00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời) của
[Giai đoạn 3](../00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình). Đây là **lab đầu tiên dạng
dự án**: đọc [file dự án tổng](DU-AN-BOOKSTORE.md) trước, ít nhất mục 1, 3, 4, 5 và 6. Mọi ticket
dưới đây dựa vào hợp đồng kỹ thuật ở [mục 4 của file đó](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật).

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Ba image ứng dụng
của dự án nằm ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00); lab
này không chép lại con số phiên bản nào trong phần văn bản.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi bạn đã dựng được ba Pod của dự án **và** chứng minh, giải thích được:

- Vì sao init container của `db` phải có tính **bất biến kết quả**, và vì sao script khởi tạo của
  PostgreSQL chỉ chạy một lần với một thư mục dữ liệu.
- Vì sao `api` dùng **ba** probe khác nhau, và vì sao liveness **không được** phụ thuộc vào DB còn
  readiness thì **phải** phụ thuộc vào DB.
- Container khởi động lại thì Pod giữ nguyên IP và volume; Pod bị thay thì mọi thứ đó đổi — và
  điều này phá `api` của bạn như thế nào.
- Sidecar khởi động **trước** container chính, ở lại suốt vòng đời Pod, và bị tắt **sau** container
  chính; downward API qua volume nhận được thay đổi label còn qua biến môi trường thì không.
- Trình tự tắt êm của từng thành phần: `preStop`, tín hiệu dừng lấy từ `STOPSIGNAL` của image, thời
  gian ân hạn, và hậu quả khi hết ân hạn.
- Đọc `phase`, trạng thái container và condition để chẩn đoán một Pod không lên: `CrashLoopBackOff`
  không phải phase; `Pending` có lý do nằm trong condition.
- Gỡ lỗi một container **không có shell** bằng ephemeral container, và ba giới hạn của nó.
- `securityContext` hai cấp, user namespace, PriorityClass — mỗi thứ đổi điều gì trên node thật.
- Pod trần không có ai đứng sau, gần như bất biến — và đó là danh sách nợ kỹ thuật mà các sprint sau
  phải trả.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 3a | Ticket kiểm chứng |
| --- | --- |
| [45 — Workload](../45-workloads-vi.md) | B11 — Pod trần không có `ownerReferences`, không ai tạo lại |
| [46 — Pod](../46-pods-vi.md) | B2, B3, B5 — Pod nhiều container dùng chung volume và mạng; B4 — thay Pod là có Pod mới; B11 — Pod gần như bất biến |
| [47 — Vòng đời của Pod](../47-pod-lifecycle-vi.md) | B4 — khởi động lại container so với thay Pod; B6 — luồng kết thúc, `STOPSIGNAL`, buộc xóa; B7 — phase, `restartPolicy`, `CrashLoopBackOff` |
| [48 — Các Condition của Pod](../48-pod-condition-vi.md) | B7 — `PodScheduled` với lý do `Unschedulable`, `Initialized` khi init hỏng, `readinessGates` |
| [49 — Các probe Liveness, Readiness và Startup](../49-probes-vi.md) | B3 — chọn probe cho `api`; B4 — readiness thất bại không giết container |
| [274 — Cấu hình các probe](../274-configure-probes-vi.md) | B2 — readiness `exec`; B3 — `httpGet` qua cổng có tên, startup probe |
| [50 — Container khởi tạo](../50-init-containers-vi.md) | B2 — init chuẩn bị file; B3 — init chờ phụ thuộc; B7 — init thất bại với `restartPolicy: Never` |
| [276 — Cấu hình khởi tạo Pod](../276-configure-pod-initialization-vi.md) | B2 — init container ghi vào volume dùng chung cho container chính |
| [51 — Các container sidecar](../51-sidecar-containers-vi.md) | B5 — sidecar `status-writer`; B6 — thứ tự tắt sidecar |
| [360 — Giao tiếp giữa các Container bằng Volume dùng chung](../360-containers-shared-volume-vi.md) | B5 — init, sidecar và nginx dùng chung một `emptyDir` |
| [272 — Gắn handler vào sự kiện vòng đời](../272-attach-handler-lifecycle-event-vi.md) | B6 — `preStop` của nginx; `preStop` không chạy khi container tự hoàn thành |
| [52 — Các container tạm thời](../52-ephemeral-containers-vi.md) | B8 — gỡ lỗi `api` không có shell |
| [292 — Chia sẻ Process Namespace](../292-share-process-namespace-vi.md) | B8 — `--target` nhìn tiến trình của container đích; ADR có bật `shareProcessNamespace` cho `web` không |
| [55 — Không gian tên người dùng](../55-user-namespaces-vi.md) | B9 — điều kiện tiên quyết và `hostUsers: false` cho `web` |
| [295 — Sử dụng user namespace với Pod](../295-user-namespaces-tasks-vi.md) | B9 — đọc `uid_map` |
| [56 — Downward API](../56-downward-api-vi.md) | B5 — chọn đúng cơ chế cho từng trường |
| [336 — Expose thông tin Pod qua biến môi trường](../336-env-variable-expose-pod-info-vi.md) | B5 — tên Pod, tên node vào trang chủ |
| [335 — Expose thông tin Pod qua file](../335-downward-api-volume-vi.md) | B5 — label của Pod qua volume, cập nhật khi đổi label |
| [60 — Cấu hình Pod nâng cao](../60-advanced-pod-config-vi.md) | B2 — `nodeSelector`; B9 — `securityContext`, PriorityClass, taint và toleration |
| [285 — Sử dụng Image Volume](../285-image-volumes-vi.md) | B10 ⭐ — đánh giá image volume (tùy chọn) |
| [226 — Chạy thành phần Node dưới người dùng không phải root](../226-kubelet-in-userns-vi.md) | **Không thực hành.** Rootless mode cần dựng lại node bằng kind/minikube/K3s hoặc feature gate `KubeletInUserNamespace` — lệch baseline Lab 00. B9 chỉ đối chiếu: user namespace **cho Pod** khác với kubelet chạy trong user namespace |
| [262 — Cấu hình Pod và Container](../262-configure-pod-container-vi.md) | **Không thực hành riêng.** Đây là trang mục lục của nhánh tác vụ; mọi bài nó dẫn tới đã nằm trong bảng trên |

Bốn phần bị hoãn có chủ đích, là **nợ lab** — nói được vì sao ở checkpoint:

| Phần hoãn | Vì cần | Trả ở |
| --- | --- | --- |
| Thực hành static Pod (bài [293](../293-static-pod-tasks-vi.md)) | khái niệm ở bài [58](../58-static-pods-vi.md), nhóm 3c | Lab 3c |
| Request/limit hiệu dụng của init container và sidecar (bài 50, 51) | `requests`/`limits` của bài [110](../110-manage-resources-containers-vi.md) | Lab 3c |
| Phase `Unknown` (bài 47) | cắt liên lạc với node, giai đoạn 12 | Lab 12 |
| Readiness thất bại loại Pod khỏi tập backend sẵn sàng nhận lưu lượng — endpoint `ready=false` (bài 49) | Service, giai đoạn 5 | Lab 5a |

### 1.2. Thời lượng

4–6 giờ. Phần cốt lõi (B0–B9, B11, B12) khoảng 4 giờ nếu không bí lâu ở ticket nào; B10 là tùy
chọn. Có thể chia hai phiên: phiên 1 làm B0–B5, phiên 2 làm B6–B12. Giữa hai phiên cứ để nguyên
cluster; đầu phiên 2 chạy lại gate ngắn A5.5 và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác.
- **Fault injection chỉ trên `lab-k8s-worker2`.** `db` được ghim vào node này theo
  [thiết kế dự án](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch); mọi Pod diễn tập cũng
  ghim vào đây.
- Repo dự án: `~/bookstore`. Manifest của Pod thử nghiệm: `~/lab-work/3a/`. Bằng chứng:
  `~/lab-evidence/3a/`.
- Lab chỉ dùng kiến thức nhóm 3a và các nhóm trước. **Không** dùng ConfigMap, Secret (nhóm 3b),
  `requests`/`limits` (3c), Deployment (4a), Service (5a), PVC (6a). Volume chỉ dùng `emptyDir`,
  `downwardAPI` và — ở B10 — `image`, đúng như các bài thực hành của nhóm này dùng.
- Pod trần gần như bất biến. Muốn đổi spec của `db`, `api` hoặc `web` thì xóa Pod rồi tạo lại; cách
  nhanh nhất cho cả hệ thống là `./scripts/teardown.sh && ./scripts/deploy.sh` (B1 tạo hai script
  này). Dữ liệu trong `emptyDir` mất theo — đó là nợ `TD-04`, không phải lỗi của bạn.
- Lab tạo đúng **một** object cấp cluster: PriorityClass `bookstore-critical` (B9). Teardown xóa
  nó.
- Không gỡ taint của `lab-k8s-master`. Pod thử nghiệm ở B9 chỉ **dung thứ** taint đó vài phút rồi
  bị xóa.
- Không sửa cấu hình node, không cài gói. Lệnh qua `ssh` tới worker chỉ để đọc, trừ `crictl pull`
  ở B0.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 1: Pod hóa ba thành phần

## B0. Khởi động dự án

**Mục đích:** xác nhận cluster ở đúng `01-cluster-ready`, máy có đủ công cụ cho repo, và các node
đã có sẵn image để thời gian kéo image không lẫn vào thời gian phản ứng của kubelet ở các bước sau.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Công cụ, thư mục, và namespace dự án chưa tồn tại:

```bash
git --version && echo 'PASS: co git'
openssl version && echo 'PASS: co openssl'
test ! -e ~/bookstore && echo 'PASS: repo chua ton tai'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore da ton tai' || echo 'PASS: chua co namespace bookstore'
mkdir -p ~/lab-work/3a ~/lab-evidence/3a
```

Kéo sẵn image lên hai worker. Tag lấy đúng từ [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa)
và [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00):

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

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; có `git` và `openssl`; repo và namespace chưa tồn tại;
đủ tám dòng `PASS: ... co ...`, không có dòng `FAIL:`.

Không có `git` thì **dừng** — cài gói là quyết định của bạn, lab không tự cài. Kéo image fail thì
xem mục 4.

## B1. [BS-3A-01] Khởi tạo repo dự án

**Loại:** Tính năng.

**Bối cảnh:** dự án sống trong repo, không sống trong cluster. Mọi sprint sau đều bắt đầu bằng việc
dựng lại hệ thống chỉ từ repo, nên khung repo và ba script dùng chung phải có ngay từ đầu.

**Yêu cầu:**

1. `~/bookstore` là git repo có đúng cấu trúc ở
   [mục 5.1 của file dự án](DU-AN-BOOKSTORE.md#51-vị-trí-và-cấu-trúc). Thư mục rỗng giữ bằng file
   `.gitkeep`.
2. `k8s/app/00-namespace.yaml` khai Namespace `bookstore` **theo hợp đồng**
   ([mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label)) — tự viết.
3. `scripts/deploy.sh`, `scripts/teardown.sh`, `tests/smoke.sh` — chép nguyên văn từ khối bên dưới.
   Đây là công cụ, không phải nội dung học.
4. `docs/adr/0000-mau.md` là mẫu ADR; `docs/tech-debt.md` là bảng nợ, ban đầu rỗng.
5. Commit đầu tiên.

**Ràng buộc:** Namespace được tạo bằng **manifest**, không bằng `kubectl create namespace` — lệnh
imperative không gắn label của dự án, và teardown tìm namespace theo label.

**Đọc lại:** [Lab 1b](LAB-1B-OBJECT-LABEL-KUBECTL-VA-KUBECONFIG.md) phần label và namespace; bài
[45](../45-workloads-vi.md) để hiểu vì sao sprint này cố ý dùng Pod trần.

**Bàn giao:** repo có một commit; Namespace `bookstore` tồn tại trên cluster.

**Nghiệm thu:**

- `PASS:` `~/bookstore` là git repo, có commit.
- `PASS:` có đủ các thư mục `docs/adr`, `docs/postmortem`, `k8s/cluster`, `k8s/app`, `platform`,
  `scripts`, `tests`.
- `PASS:` Namespace `bookstore` tồn tại và mang label `app.kubernetes.io/part-of=bookstore`.
- `PASS:` `.gitignore` loại trừ `secrets/*.env`.

Ba script, chép nguyên văn:

```bash
mkdir -p ~/bookstore/{docs/adr,docs/postmortem,k8s/cluster,k8s/app,platform,secrets,scripts,tests}
cd ~/bookstore

cat > scripts/deploy.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai dự án Bookstore từ repo — phiên bản Lab 3a.
# Lab 3a chưa có Service: api tìm db bằng IP của Pod db, chèn vào manifest lúc triển khai (TD-03).
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore

has_yaml() { compgen -G "$1/*.yaml" >/dev/null; }

if has_yaml k8s/cluster; then kubectl apply -f k8s/cluster/; fi
kubectl apply -f k8s/app/00-namespace.yaml

if has_yaml k8s/app/db; then
  kubectl apply -f k8s/app/db/
  kubectl -n "$NS" wait --for=condition=Ready pod/db --timeout=300s
fi

if [ -f k8s/app/api/pod.yaml.tpl ]; then
  DB_IP="$(kubectl -n "$NS" get pod db -o jsonpath='{.status.podIP}')"
  sed "s/__DB_HOST__/${DB_IP}/g" k8s/app/api/pod.yaml.tpl | kubectl apply -f -
fi

if has_yaml k8s/app/web; then kubectl apply -f k8s/app/web/; fi

for P in db api web; do
  if kubectl -n "$NS" get pod "$P" >/dev/null 2>&1; then
    kubectl -n "$NS" wait --for=condition=Ready "pod/$P" --timeout=300s
  fi
done
kubectl -n "$NS" get pods -o wide
EOF

cat > scripts/teardown.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ toàn bộ dự án Bookstore khỏi cluster — phiên bản Lab 3a.
set -euo pipefail
SEL='app.kubernetes.io/part-of=bookstore'
kubectl delete namespace -l "$SEL" --ignore-not-found --wait=true --timeout=300s
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
if [ "$LEFT" -eq 0 ]; then echo 'teardown: sach'; else echo "teardown: con $LEFT object"; exit 1; fi
EOF

cat > tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — phiên bản tạm của B1. B12 thay bằng phiên bản đầy đủ của Lab 3a.
set -u
kubectl get namespace bookstore >/dev/null 2>&1 \
  && { echo 'PASS: namespace bookstore ton tai'; exit 0; } \
  || { echo 'FAIL: chua co namespace bookstore'; exit 1; }
EOF

chmod +x scripts/*.sh tests/*.sh
printf 'secrets/*.env\n' > .gitignore
touch docs/postmortem/.gitkeep k8s/cluster/.gitkeep platform/.gitkeep secrets/.gitkeep
```

Mẫu ADR, chép nguyên văn vào `docs/adr/0000-mau.md`:

```markdown
# NNNN — Tiêu đề quyết định

- Trạng thái: đề xuất | chấp nhận | bị thay bởi NNNN
- Sprint: Lab <mã>, ticket BS-<mã>-<nn>

## Bối cảnh

Vấn đề gì, ràng buộc nào, bằng chứng nào (đường dẫn file trong ~/lab-evidence).

## Quyết định

Chọn gì. Một hai câu.

## Lý do

Vì sao, dẫn đúng mục tài liệu đã đọc.

## Hệ quả

Được gì, mất gì, nợ nào phát sinh.
```

<details>
<summary>Gợi ý</summary>

- Namespace là object cấp cluster nhưng vẫn có `metadata.labels` như mọi object.
- `docs/tech-debt.md` chỉ cần một bảng Markdown với các cột `Mã | Nợ | Phát sinh | Trả ở | Trạng
  thái`. Bảng đầy đủ được điền ở B11.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
cd ~/bookstore
git rev-parse --is-inside-work-tree >/dev/null 2>&1 && git log --oneline -1 >/dev/null 2>&1 \
  && echo 'PASS: la git repo, co commit' || echo 'FAIL: chua co repo hoac chua commit'

BAD=0
for D in docs/adr docs/postmortem k8s/cluster k8s/app platform scripts tests; do
  test -d "$D" || { echo "FAIL: thieu $D"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: du cau truc thu muc'

kubectl apply -f k8s/app/00-namespace.yaml
L="$(kubectl get namespace bookstore -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}')"
test "$L" = 'bookstore' && echo 'PASS: namespace mang label cua du an'

grep -qx 'secrets/\*.env' .gitignore && echo 'PASS: .gitignore loai tru secrets/*.env'
./tests/smoke.sh
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
```

```bash
cd ~/bookstore
cat > README.md <<'EOF'
# Bookstore

Ứng dụng ba tầng (nginx → PostgREST → PostgreSQL) chạy trên Kubernetes.
Thiết kế: k8s-docs/labs/DU-AN-BOOKSTORE.md trong repo khóa học.

- Triển khai: ./scripts/deploy.sh
- Kiểm thử:   ./tests/smoke.sh
- Gỡ bỏ:      ./scripts/teardown.sh
EOF
printf '| Mã | Nợ | Phát sinh | Trả ở | Trạng thái |\n| --- | --- | --- | --- | --- |\n' > docs/tech-debt.md
git init -q && git add -A && git commit -qm 'lab-3a: khoi tao repo du an' && git log --oneline
```

</details>

## B2. [BS-3A-02] Pod `db` tự khởi tạo schema

**Loại:** Tính năng.

**Bối cảnh:** hệ thống cũ nạp schema bằng thư mục `./sql` mount vào
`/docker-entrypoint-initdb.d` ([hiện trạng](DU-AN-BOOKSTORE.md#21-hiện-trạng)). Chưa có ConfigMap,
nên file SQL phải được **sinh ra bên trong Pod** trước khi PostgreSQL khởi động.

**Yêu cầu:**

1. Pod trần `db` trong namespace `bookstore`, label theo hợp đồng, container tên `postgres`, image
   `postgres` đúng tag của A1.4, cổng `5432` tên `postgres`.
2. Một **init container** (`busybox`) ghi file SQL khởi tạo vào một `emptyDir`; container
   `postgres` mount cùng volume đó tại `/docker-entrypoint-initdb.d`.
3. Script SQL tạo đúng dữ liệu ở [mục 4.3 của file dự án](DU-AN-BOOKSTORE.md#43-dữ-liệu): schema
   `api`, hai bảng, 5 cuốn sách, role `web_anon`, role `authenticator` với mật khẩu
   `lab-3a-authenticator`.
4. Database tên `bookstore`; mật khẩu superuser đặt thẳng trong manifest.
5. Thư mục dữ liệu PostgreSQL nằm trên một `emptyDir` riêng.
6. Pod chạy trên `lab-k8s-worker2`, chọn bằng **`nodeSelector`** chứ không bằng `nodeName`.
7. Readiness probe chỉ thành công khi PostgreSQL **nhận kết nối TCP** trên cổng 5432.
8. Manifest ở `k8s/app/db/pod.yaml`.

**Ràng buộc:** chưa có ConfigMap và Secret — mật khẩu nằm trong manifest là nợ `TD-02`, ghi chú bằng
comment ngay cạnh giá trị. Không đặt liveness probe trong ticket này; quyết định có cần hay không là
câu hỏi thiết kế.

**Đọc lại:**

- Bài [50](../50-init-containers-vi.md), mục *Hiểu về init container* và *Hành vi chi tiết* — chú ý
  câu về **tính bất biến kết quả** khi ghi vào `emptyDir`.
- Bài [276](../276-configure-pod-initialization-vi.md#create-a-pod-that-has-an-init-container) —
  mẫu init container ghi file cho container chính.
- Bài [274](../274-configure-probes-vi.md#define-readiness-probes) — readiness probe; bài
  [49](../49-probes-vi.md#check-mechanisms) — bốn cơ chế kiểm tra.
- Bài [60](../60-advanced-pod-config-vi.md#scheduling) — `nodeSelector`.
- Tài liệu image `postgres` trên Docker Hub, mục *Initialization scripts* — script chỉ chạy khi
  thư mục dữ liệu rỗng.

**Bàn giao:** `k8s/app/db/pod.yaml`; `docs/adr/0001-db-probe.md` trả lời câu hỏi thiết kế;
`~/lab-evidence/3a/b2-db.txt`.

**Nghiệm thu:**

- `PASS:` Pod `db` Ready, chạy trên `lab-k8s-worker2`, và spec chọn node bằng `nodeSelector`.
- `PASS:` init container thoát mã 0; container `postgres` mount `/docker-entrypoint-initdb.d`.
- `PASS:` log của `postgres` cho thấy entrypoint đã chạy script trong `/docker-entrypoint-initdb.d`.
- `PASS:` `api.books` có ít nhất 5 dòng; có đủ role `authenticator` và `web_anon`.
- `PASS:` container `postgres` có readiness probe, không có liveness probe.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0001-db-probe.md`:

1. Readiness kiểm qua TCP `127.0.0.1` hay qua Unix socket thì khác gì, trong lúc entrypoint đang
   chạy script khởi tạo? Vì sao khác biệt đó quan trọng với `api`?
2. `db` có cần liveness probe không? Dẫn đúng câu của bài 49 về khi nào **không cần** liveness.
3. Init container của bạn có bất biến kết quả không, nếu kubelet chạy lại nó lần thứ hai trên cùng
   `emptyDir`? Khi nào Pod chạy lại init container?
4. Yêu cầu bắt buộc `nodeSelector` thay vì đặt thẳng `nodeName`. Với mỗi cách, ai quyết định Pod
   chạy ở node nào, và trường nào của Pod ghi lại quyết định đó?

<details>
<summary>Gợi ý</summary>

1. Ghi file trong init container bằng heredoc của shell bên trong `command: ["sh", "-c", "..."]`.
   Dùng khối YAML `|` để giữ xuống dòng; dòng kết thúc heredoc sẽ ở cột 0 sau khi YAML bỏ thụt lề.
2. Image `postgres` có sẵn `pg_isready`. Đọc `pg_isready --help` bằng
   `kubectl exec ... -- pg_isready --help` khi Pod đã chạy, tìm cờ chọn host.
3. Label của node dùng cho `nodeSelector`: `kubectl get node lab-k8s-worker2 --show-labels`.
4. Biến môi trường của image: `POSTGRES_DB`, `POSTGRES_PASSWORD`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3a
jp() { kubectl -n "$NS" get pod db -o jsonpath="$1"; }

kubectl -n "$NS" wait --for=condition=Ready pod/db --timeout=300s && echo 'PASS: db Ready'
kubectl -n "$NS" get pod db -o wide | tee "$EV/b2-db.txt"

test "$(jp '{.spec.nodeName}')" = 'lab-k8s-worker2' && echo 'PASS: db chay tren lab-k8s-worker2'
test "$(jp '{.spec.nodeSelector.kubernetes\.io/hostname}')" = 'lab-k8s-worker2' \
  && echo 'PASS: chon node bang nodeSelector'

N_INIT="$(jp '{.spec.initContainers[*].name}' | wc -w)"
EC="$(jp '{.status.initContainerStatuses[0].state.terminated.exitCode}')"
{ test "$N_INIT" -ge 1 && test "$EC" = '0'; } && echo 'PASS: init container thoat ma 0'
jp '{.spec.containers[?(@.name=="postgres")].volumeMounts[*].mountPath}' \
  | grep -qw '/docker-entrypoint-initdb.d' && echo 'PASS: postgres mount /docker-entrypoint-initdb.d'

kubectl -n "$NS" logs db -c postgres | tee -a "$EV/b2-db.txt" \
  | grep -q 'running /docker-entrypoint-initdb.d/' && echo 'PASS: entrypoint da chay script khoi tao'

SQL() { kubectl -n "$NS" exec db -c postgres -- psql -U postgres -d bookstore -tAc "$1"; }
N="$(SQL 'SELECT count(*) FROM api.books')"
test "${N:-0}" -ge 5 && echo "PASS: api.books co $N dong"
R="$(SQL "SELECT string_agg(rolname, ',' ORDER BY rolname) FROM pg_roles WHERE rolname IN ('authenticator','web_anon')")"
test "$R" = 'authenticator,web_anon' && echo 'PASS: du role authenticator va web_anon'

test -n "$(jp '{.spec.containers[?(@.name=="postgres")].readinessProbe}')" && echo 'PASS: co readinessProbe'
test -z "$(jp '{.spec.containers[?(@.name=="postgres")].livenessProbe}')" && echo 'PASS: chua co livenessProbe'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/db/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: db
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: db
    app.kubernetes.io/name: postgresql
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  volumes:
  - name: initdb
    emptyDir: {}
  - name: data
    emptyDir: {}
  initContainers:
  - name: render-schema
    image: busybox:1.37
    command:
    - sh
    - -c
    - |
      set -e
      cat > /initdb/01-schema.sql <<'SQL'
      CREATE SCHEMA api;
      CREATE TABLE api.books (
        id        serial  PRIMARY KEY,
        title     text    NOT NULL,
        author    text    NOT NULL,
        price_vnd integer NOT NULL CHECK (price_vnd >= 0),
        stock     integer NOT NULL DEFAULT 0
      );
      CREATE TABLE api.orders (
        id         serial      PRIMARY KEY,
        book_id    integer     NOT NULL REFERENCES api.books (id),
        quantity   integer     NOT NULL CHECK (quantity > 0),
        created_at timestamptz NOT NULL DEFAULT now()
      );
      INSERT INTO api.books (title, author, price_vnd, stock) VALUES
        ('Dế Mèn phiêu lưu ký', 'Tô Hoài', 68000, 12),
        ('Số đỏ', 'Vũ Trọng Phụng', 75000, 8),
        ('Tắt đèn', 'Ngô Tất Tố', 59000, 5),
        ('Truyện Kiều', 'Nguyễn Du', 95000, 20),
        ('Nỗi buồn chiến tranh', 'Bảo Ninh', 110000, 6);
      CREATE ROLE web_anon NOLOGIN;
      GRANT USAGE ON SCHEMA api TO web_anon;
      GRANT SELECT ON api.books TO web_anon;
      GRANT SELECT, INSERT ON api.orders TO web_anon;
      GRANT USAGE ON SEQUENCE api.orders_id_seq TO web_anon;
      CREATE ROLE authenticator NOINHERIT LOGIN PASSWORD 'lab-3a-authenticator';
      GRANT web_anon TO authenticator;
      SQL
      echo "render-schema: da ghi /initdb/01-schema.sql"
    volumeMounts:
    - name: initdb
      mountPath: /initdb
  containers:
  - name: postgres
    image: postgres:17.11
    env:
    - name: POSTGRES_DB
      value: bookstore
    - name: POSTGRES_PASSWORD
      value: lab-3a-mat-khau-tam        # TD-02: mật khẩu nằm trong manifest, trả ở Lab 3b
    ports:
    - name: postgres
      containerPort: 5432
    readinessProbe:
      exec:
        command: ["pg_isready", "-h", "127.0.0.1", "-p", "5432", "-U", "postgres", "-d", "bookstore"]
      periodSeconds: 5
      timeoutSeconds: 3
    volumeMounts:
    - name: initdb
      mountPath: /docker-entrypoint-initdb.d
      readOnly: true
    - name: data
      mountPath: /var/lib/postgresql/data
```

```bash
kubectl apply -f ~/bookstore/k8s/app/db/pod.yaml
kubectl -n bookstore get pod db -w     # Ctrl+C khi READY 1/1
```

Ý chính cho ADR 0001:

1. Trong lúc chạy script khởi tạo, entrypoint dựng một server **tạm** chỉ nghe Unix socket. Kiểm
   bằng socket sẽ báo sẵn sàng quá sớm; kiểm bằng TCP thì chỉ thành công khi server thật đã mở cổng —
   đúng con đường mà `api` sẽ dùng.
2. Không bắt buộc. Bài 49 nói nếu tiến trình tự crash khi gặp sự cố thì kubelet đã xử lý theo
   `restartPolicy`. Một liveness đặt sai cho DB còn gây hại: giết DB đang bận đúng lúc tải cao.
3. Có — `cat >` ghi đè cả file nên chạy lại cho cùng kết quả. Init container chạy lại khi Pod
   khởi động lại hoặc sandbox bị dựng lại (bài 50, *Các lý do khởi động lại Pod*).
4. `nodeSelector` là ràng buộc gửi cho **scheduler** (bài 60, *Node selector*): scheduler chọn trong
   các node mang đúng label rồi ghi kết quả vào `spec.nodeName` qua subresource `binding` (bài 46).
   Đặt thẳng `nodeName` là tự làm thay việc đó — scheduler bị bỏ qua hoàn toàn. Dự án cần đi qua
   scheduler vì các sprint sau đặt thêm điều kiện lên chính quyết định này (tài nguyên ở 3c, taint ở
   B9); nghiệm thu đọc `spec.nodeName` để biết kết quả, đọc `nodeSelector` để biết cơ chế.

</details>

## B3. [BS-3A-03] Pod `api` với ba probe đúng vai trò

**Loại:** Tính năng.

**Bối cảnh:** hệ thống cũ không có health check (`P4`). PostgREST có sẵn admin server với hai
endpoint `/live` và `/ready` ([mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe)).
Việc của ticket là gắn đúng endpoint vào đúng loại probe.

**Yêu cầu:**

1. Pod trần `api`, label theo hợp đồng, container tên `postgrest`, image `postgrest` đúng tag A1.4.
2. Cấu hình PostgREST bằng biến môi trường: URI kết nối tới `db` với role `authenticator`; schema
   `api`; role ẩn danh `web_anon`; cổng chính `3000`; admin server cổng `3001`.
3. Hai cổng khai báo có tên: `http` (3000) và `admin` (3001). Probe tham chiếu **tên cổng**.
4. Có **startup**, **readiness** và **liveness** probe; mỗi probe trỏ vào đúng endpoint theo vai
   trò của nó.
5. Một init container chờ tới khi DB **nhận kết nối** rồi mới cho `postgrest` khởi động.
6. Địa chỉ DB là IP của Pod `db`. Manifest lưu dạng mẫu `k8s/app/api/pod.yaml.tpl` với chuỗi giữ
   chỗ `__DB_HOST__`; `scripts/deploy.sh` thay chuỗi này lúc triển khai.
7. `GET /books` trên cổng 3000 trả về danh sách sách.

**Ràng buộc:** chưa có Service — gọi `api` bằng IP Pod. Image `postgrest` không có shell, nên không
dùng được probe `exec` cho nó.

**Đọc lại:**

- Bài [49](../49-probes-vi.md): [*Các loại probe*](../49-probes-vi.md#types-of-probe), toàn bộ mục
  [*Khi nào dùng từng loại probe*](../49-probes-vi.md#when-to-use-each-probe) — đặc biệt đoạn về
  ứng dụng phụ thuộc chặt vào backend, và [*Kết quả của probe*](../49-probes-vi.md#probe-results).
- Bài [274](../274-configure-probes-vi.md): liveness HTTP, *Sử dụng port có tên*,
  [startup probe](../274-configure-probes-vi.md#define-startup-probes).
- Bài [50](../50-init-containers-vi.md), ví dụ *Init container trong thực tế*.
- PostgREST v14: [Admin Server](https://docs.postgrest.org/en/v14/references/admin_server.html),
  [Configuration](https://docs.postgrest.org/en/v14/references/configuration.html) — mục *Environment
  Variables* cho biết tên biến tương ứng với từng tham số.

**Bàn giao:** `k8s/app/api/pod.yaml.tpl`; `docs/adr/0002-api-probes.md`;
`~/lab-evidence/3a/b3-books.json`.

**Nghiệm thu:**

- `PASS:` Pod `api` Ready; init container thoát mã 0.
- `PASS:` đủ ba probe trên container `postgrest`; readiness trỏ `/ready`, liveness trỏ `/live`, cả
  hai qua cổng tên `admin`.
- `PASS:` URI kết nối trong spec trỏ đúng IP hiện tại của Pod `db`.
- `PASS:` `GET http://<IP api>:3000/books` từ một Pod khác trả về ít nhất 5 cuốn sách.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0002-api-probes.md`:

1. Vì sao readiness trỏ `/ready` mà liveness trỏ `/live`? Nếu đổi ngược, điều gì xảy ra khi DB chết
   trong 5 phút?
2. Startup probe ở đây bảo vệ khỏi điều gì? Tính thời gian tối đa nó cho `postgrest` khởi động từ
   cấu hình của bạn.
3. PostgREST tự thử kết nối lại DB khi mất kết nối. Vậy init container chờ DB còn đáng giữ không?
   Đánh đổi là gì?

<details>
<summary>Gợi ý</summary>

1. Tên biến môi trường: tiền tố `PGRST_`, viết hoa, gạch dưới thay gạch ngang — `db-uri` thành
   `PGRST_DB_URI`. Cần năm biến.
2. Init container chờ DB dùng image `postgres` vì nó có `pg_isready`; vòng `until ...; do sleep 2;
   done` là đủ.
3. Gọi API từ một Pod tạm:
   `kubectl -n bookstore run curl-tam --rm -i --restart=Never --image=busybox:1.37 -- wget -qO- http://<IP>:3000/books`.
4. Lấy IP Pod `db`: `kubectl -n bookstore get pod db -o jsonpath='{.status.podIP}'`. Chạy
   `./scripts/deploy.sh` để script tự chèn IP.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3a
jp() { kubectl -n "$NS" get pod api -o jsonpath="$1"; }
C='{.spec.containers[?(@.name=="postgrest")]'

kubectl -n "$NS" wait --for=condition=Ready pod/api --timeout=300s && echo 'PASS: api Ready'
test "$(jp '{.status.initContainerStatuses[0].state.terminated.exitCode}')" = '0' \
  && echo 'PASS: init container thoat ma 0'

BAD=0
for P in startupProbe readinessProbe livenessProbe; do
  test -n "$(jp "$C.$P}")" || { echo "FAIL: thieu $P"; BAD=1; }
done
test "$BAD" -eq 0 && echo 'PASS: du ba probe'
{ test "$(jp "$C.readinessProbe.httpGet.path}")" = '/ready' && \
  test "$(jp "$C.readinessProbe.httpGet.port}")" = 'admin' && \
  test "$(jp "$C.livenessProbe.httpGet.path}")"  = '/live'  && \
  test "$(jp "$C.livenessProbe.httpGet.port}")"  = 'admin'; } \
  && echo 'PASS: readiness -> /ready, liveness -> /live, qua cong admin'

DB_IP="$(kubectl -n "$NS" get pod db -o jsonpath='{.status.podIP}')"
jp "$C.env[?(@.name==\"PGRST_DB_URI\")].value}" | grep -q "@${DB_IP}:5432/" \
  && echo "PASS: URI tro dung IP hien tai cua db ($DB_IP)"

API_IP="$(jp '{.status.podIP}')"
kubectl -n "$NS" run curl-tam --rm -i --restart=Never --image=busybox:1.37 \
  -- wget -qO- "http://${API_IP}:3000/books" > "$EV/b3-books.json" 2>/dev/null
N="$(grep -o '"title"' "$EV/b3-books.json" | wc -l)"
test "$N" -ge 5 && echo "PASS: /books tra $N cuon sach"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/api/pod.yaml.tpl — __DB_HOST__ được scripts/deploy.sh thay bằng IP Pod db (TD-03)
apiVersion: v1
kind: Pod
metadata:
  name: api
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: api
    app.kubernetes.io/name: postgrest
spec:
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
    - name: DB_HOST
      value: "__DB_HOST__"
  containers:
  - name: postgrest
    image: postgrest/postgrest:v14.18
    env:
    - name: PGRST_DB_URI
      value: "postgres://authenticator:lab-3a-authenticator@__DB_HOST__:5432/bookstore"   # TD-02
    - name: PGRST_DB_SCHEMAS
      value: "api"
    - name: PGRST_DB_ANON_ROLE
      value: "web_anon"
    - name: PGRST_SERVER_PORT
      value: "3000"
    - name: PGRST_ADMIN_SERVER_PORT
      value: "3001"
    ports:
    - name: http
      containerPort: 3000
    - name: admin
      containerPort: 3001
    startupProbe:
      httpGet:
        path: /live
        port: admin
      periodSeconds: 2
      failureThreshold: 30
    readinessProbe:
      httpGet:
        path: /ready
        port: admin
      periodSeconds: 5
      failureThreshold: 2
    livenessProbe:
      httpGet:
        path: /live
        port: admin
      periodSeconds: 10
      failureThreshold: 3
```

```bash
cd ~/bookstore && ./scripts/deploy.sh
```

Ý chính cho ADR 0002:

1. Bài 49: liveness đạt khi **bản thân ứng dụng** khỏe; readiness kiểm thêm các backend cần thiết.
   Đổi ngược thì DB chết 5 phút là kubelet giết `postgrest` liên tục — khởi động lại không chữa được
   DB, chỉ thêm `CrashLoopBackOff` và mất log.
2. Startup probe chặn liveness và readiness tới khi nó thành công, nên một lần khởi động chậm không
   bị liveness giết. Thời gian tối đa ≈ `failureThreshold × periodSeconds` = 60 giây với cấu hình
   trên.
3. Giữ được, nhưng không bắt buộc: PostgREST tự thử lại và `/ready` đã phản ánh trạng thái DB. Init
   container làm Pod nằm ở `Init:0/1` — dễ nhìn hơn khi DB chưa lên, nhưng `postgrest` không chạy
   nên không có log hay metric của nó trong lúc chờ.

</details>

## B4. [BS-3A-04] Diễn tập: DB biến mất

**Loại:** Diễn tập sự cố — trên `lab-k8s-worker2`, nơi `db` chạy.

**Bối cảnh:** trước khi tin các probe vừa viết, phải thấy chúng phản ứng đúng khi DB mất. Diễn tập
hai kịch bản có hậu quả rất khác nhau: **container** DB chết, và **Pod** DB bị thay.

**Yêu cầu:**

1. Kịch bản A — gửi `SIGINT` tới tiến trình 1 của container `postgres` (PostgreSQL hiểu tín hiệu
   này là *fast shutdown*). Chứng minh: container `postgres` được khởi động lại, Pod `db` giữ nguyên
   IP, `api` tự Ready lại mà **không** bị khởi động lại.
2. Kịch bản B — xóa Pod `db`. Chứng minh: `api` chuyển NotReady nhưng vẫn `Running` và **không** bị
   liveness giết; tạo lại `db` thì `api` **vẫn** NotReady.
3. Khôi phục `api` cho đúng. Thử trước bằng `kubectl apply` bản manifest có IP mới, ghi lại API
   server trả lời gì, rồi khôi phục bằng cách đúng.
4. Viết postmortem ngắn.

**Ràng buộc:** không sửa manifest trong repo để "chữa" sự cố trong lúc diễn tập. Kịch bản B làm mất
dữ liệu trong `emptyDir` — chấp nhận được, script khởi tạo sẽ dựng lại.

**Đọc lại:**

- Bài [47](../47-pod-lifecycle-vi.md): [*Thời gian sống của Pod*](../47-pod-lifecycle-vi.md#pod-lifetime)
  — Pod thay thế là Pod khác dù cùng tên;
  [*Khởi động lại container*](../47-pod-lifecycle-vi.md#restart-policy).
- Bài [49](../49-probes-vi.md#probe-results) — hậu quả của `Failure` theo từng loại probe.
- Bài [46](../46-pods-vi.md), mục *Cập nhật và thay thế Pod* — trường nào sửa được trên Pod đang
  chạy.

**Bàn giao:** `docs/postmortem/3a-db-bien-mat.md`; `~/lab-evidence/3a/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` kịch bản A — `restartCount` của `postgres` tăng 1, IP Pod `db` không đổi, log lần chạy
  trước có dòng *fast shutdown*, `restartCount` của `postgrest` không đổi, `api` Ready lại.
- `PASS:` kịch bản B — sau khi xóa `db`, `api` có `Ready=False`, phase `Running`, sự kiện
  `Readiness probe failed`, và `restartCount` của `postgrest` không đổi sau ít nhất 60 giây.
- `PASS:` `db` mới có IP khác IP cũ và `api` vẫn NotReady.
- `PASS:` `kubectl apply` manifest `api` với IP mới bị API server từ chối.
- `PASS:` sau khi khôi phục, `api` Ready và `/books` trả dữ liệu.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Vì sao kịch bản A tự lành còn kịch bản B thì không? Nói bằng khái niệm container restart và Pod
   replacement của bài 47.
2. Nếu liveness của `api` trỏ `/ready`, kịch bản B diễn ra thế nào? Việc khởi động lại có chữa được
   gì không?
3. Thứ gì trong thiết kế hiện tại khiến kịch bản B không tự lành? Nợ kỹ thuật nào, trả ở lab nào?

<details>
<summary>Gợi ý</summary>

- Tiến trình 1 của container `postgres` chính là PostgreSQL. `kubectl exec db -c postgres -- kill -INT 1`.
- Log của lần chạy trước: `kubectl logs --previous`.
- Chờ một condition cụ thể: `kubectl wait --for=condition=Ready=false pod/api`.
- Muốn đổi biến môi trường của Pod trần thì phải xóa Pod. `./scripts/deploy.sh` chèn IP mới khi tạo
  lại.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script này chính là cuộc diễn tập</summary>

Kịch bản A:

```bash
NS=bookstore; EV=~/lab-evidence/3a
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2"; }
rc() { jp "$1" "{.status.containerStatuses[?(@.name==\"$2\")].restartCount}"; }

IP0="$(jp db '{.status.podIP}')"; DR0="$(rc db postgres)"; AR0="$(rc api postgrest)"
echo "truoc: db ip=$IP0 restarts=$DR0 ; api restarts=$AR0" | tee "$EV/b4-a.txt"

kubectl -n "$NS" exec db -c postgres -- kill -INT 1
for i in $(seq 1 36); do test "$(rc db postgres)" -gt "$DR0" && break; sleep 5; done
kubectl -n "$NS" wait --for=condition=Ready pod/db  --timeout=180s
kubectl -n "$NS" wait --for=condition=Ready pod/api --timeout=180s

IP1="$(jp db '{.status.podIP}')"; DR1="$(rc db postgres)"; AR1="$(rc api postgrest)"
echo "sau: db ip=$IP1 restarts=$DR1 ; api restarts=$AR1" | tee -a "$EV/b4-a.txt"
test "$DR1" -eq $((DR0 + 1)) && echo 'PASS: container postgres duoc khoi dong lai'
test "$IP1" = "$IP0"          && echo 'PASS: Pod db giu nguyen IP'
kubectl -n "$NS" logs db -c postgres --previous | tee -a "$EV/b4-a.txt" \
  | grep -q 'fast shutdown' && echo 'PASS: PostgreSQL nhan SIGINT -> fast shutdown'
test "$AR1" -eq "$AR0"        && echo 'PASS: api khong bi khoi dong lai va da Ready lai'
```

Kịch bản B:

```bash
NS=bookstore; EV=~/lab-evidence/3a
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2"; }
rc() { jp "$1" "{.status.containerStatuses[?(@.name==\"$2\")].restartCount}"; }

IP0="$(jp db '{.status.podIP}')"; AR0="$(rc api postgrest)"
kubectl -n "$NS" delete pod db --wait=true

kubectl -n "$NS" wait --for=condition=Ready=false pod/api --timeout=120s
test "$(jp api '{.status.phase}')" = 'Running' && echo 'PASS: api NotReady nhung van Running'
kubectl -n "$NS" describe pod api | tee "$EV/b4-b-describe.txt" \
  | grep -q 'Readiness probe failed' && echo 'PASS: su kien Readiness probe failed'
sleep 60
test "$(rc api postgrest)" -eq "$AR0" && echo 'PASS: sau 60 giay liveness van khong khoi dong lai api'

kubectl apply -f ~/bookstore/k8s/app/db/
kubectl -n "$NS" wait --for=condition=Ready pod/db --timeout=300s
IP2="$(jp db '{.status.podIP}')"
echo "db cu=$IP0 db moi=$IP2" | tee "$EV/b4-b.txt"
sleep 45
if [ "$IP2" != "$IP0" ]; then
  echo 'PASS: db moi co IP khac'
  test "$(jp api '{.status.conditions[?(@.type=="Ready")].status}')" = 'False' \
    && echo 'PASS: api van NotReady vi con tro toi IP cu'
else
  echo 'IP trung voi IP cu (hiem gap): api se tu Ready lai — xoa db va chay lai kich ban B'
fi

sed "s/__DB_HOST__/${IP2}/g" ~/bookstore/k8s/app/api/pod.yaml.tpl \
  | kubectl apply -f - 2>&1 | tee "$EV/b4-b-apply.txt" | grep -qi 'forbidden' \
  && echo 'PASS: API server tu choi doi env cua Pod dang chay'

kubectl -n "$NS" delete pod api --wait=true
~/bookstore/scripts/deploy.sh
API_IP="$(jp api '{.status.podIP}')"
kubectl -n "$NS" run curl-tam --rm -i --restart=Never --image=busybox:1.37 \
  -- wget -qO- "http://${API_IP}:3000/books" 2>/dev/null | grep -q '"title"' \
  && echo 'PASS: api da khoi phuc, /books tra du lieu'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. Kịch bản A chỉ khởi động lại **container**: Pod giữ nguyên UID, IP, `emptyDir`; PostgREST tự
   kết nối lại cùng địa chỉ. Kịch bản B **thay Pod**: bài 47 nói Pod thay thế là một Pod mới với
   UID mới và — trên cluster này — IP mới. `api` vẫn trỏ IP cũ.
2. Liveness trỏ `/ready` thì kubelet giết `postgrest` sau vài chu kỳ, rồi `CrashLoopBackOff`. Khởi
   động lại không đổi biến môi trường, nên không chữa được gì; chỉ thêm nhiễu.
3. `api` tìm `db` bằng IP Pod (`TD-03`, trả ở Lab 5a bằng Service và DNS). Thêm vào đó Pod trần không
   tự được thay (`TD-01`) và biến môi trường của Pod đang chạy không sửa được (bài 46).

API server từ chối bằng thông báo dạng `Pod "api" is invalid: spec: Forbidden: pod updates may not
change fields other than ...` — đúng danh sách trường sửa được của bài 46.

</details>

## B5. [BS-3A-05] Pod `web`: trang có danh tính, sidecar trạng thái

**Loại:** Tính năng.

**Bối cảnh:** khi có nhiều bản sao `web` (Lab 4a), người vận hành cần biết request vừa rồi do Pod
nào, trên node nào phục vụ, và đang chạy phiên bản nào. Thông tin đó lấy từ chính Kubernetes, không
cấu hình tay.

**Yêu cầu:**

1. Pod trần `web`, label theo hợp đồng, thêm label `app.kubernetes.io/version: "0.1.0"`; container
   chính tên `nginx`, image `nginx-unprivileged` đúng tag A1.4, cổng `8080` tên `http`.
2. Một init container sinh `index.html` có chữ `Bookstore`, **tên Pod** và **tên node**, cùng file
   `healthz`, vào một `emptyDir` mà nginx phục vụ ở `/usr/share/nginx/html`.
3. Một **sidecar** tên `status-writer` (`busybox`) cứ vài giây ghi `/status.json` vào cùng thư mục,
   gồm bốn trường `pod`, `node`, `version`, `updated`. Trường `version` lấy từ label
   `app.kubernetes.io/version` của Pod **tại thời điểm ghi**.
4. Readiness của `nginx` là `GET /healthz` qua cổng tên `http`.
5. Đổi label `app.kubernetes.io/version` trên Pod đang chạy thì `/status.json` phản ánh giá trị mới
   **mà không container nào khởi động lại**.
6. Manifest ở `k8s/app/web/pod.yaml`. Trang có thể chứa đoạn script gọi `/api/books`; chưa nối được
   là bình thường — việc nối thuộc Lab 5a.

**Ràng buộc:** chọn đúng cơ chế downward API cho từng trường — có trường chỉ lấy được qua biến môi
trường, có trường chỉ lấy được qua volume. Sidecar phải thoát gọn khi nhận `SIGTERM`.

**Đọc lại:**

- Bài [51](../51-sidecar-containers-vi.md): [*Sidecar container trong Kubernetes*](../51-sidecar-containers-vi.md#pod-sidecar-containers),
  [*Sidecar container và vòng đời của Pod*](../51-sidecar-containers-vi.md#sidecar-containers-and-pod-lifecycle).
- Bài [56](../56-downward-api-vi.md#available-fields) — hai cơ chế và các trường của từng cơ chế.
- Bài [336](../336-env-variable-expose-pod-info-vi.md) và bài [335](../335-downward-api-volume-vi.md)
  — đặc biệt đoạn về symlink `..data` và cảnh báo `subPath`.
- Bài [360](../360-containers-shared-volume-vi.md) — hai container dùng chung một volume.

**Bàn giao:** `k8s/app/web/pod.yaml`; `docs/adr/0003-web-downward-api.md`;
`~/lab-evidence/3a/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` Pod `web` Ready; `status-writer` khai trong `initContainers` với `restartPolicy: Always`,
  đang chạy, và khởi động **không muộn hơn** `nginx`.
- `PASS:` trang chủ chứa `Bookstore` và đúng tên node mà Pod đang chạy.
- `PASS:` `/status.json` có đủ bốn trường, trường `pod` là `web`, và trường `updated` thay đổi theo
  thời gian.
- `PASS:` sau khi đổi label version trên Pod, `/status.json` hiện version mới; `restartCount` của cả
  hai container không đổi.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0003-web-downward-api.md`:

1. Vì sao tên node phải đi qua biến môi trường, còn label version phải đi qua volume? Nếu dùng biến
   môi trường cho version thì yêu cầu 5 có đạt không?
2. `status-writer` là sidecar chứ không phải container thường trong `containers`. Nó được lợi gì về
   thứ tự khởi động và thứ tự tắt?
3. `nginx` và `status-writer` dùng chung network namespace. Ràng buộc nào về cổng mà `status-writer`
   phải tôn trọng?

<details>
<summary>Gợi ý</summary>

1. Sidecar là một mục trong `initContainers` có `restartPolicy: Always` ở **cấp container**.
2. Volume `downwardAPI` có thể chiếu `metadata.labels` thành một file, mỗi label một dòng dạng
   `key="value"`. Dùng `sed -n` để rút giá trị.
3. Ghi `status.json` ra file tạm rồi `mv` để nginx không bao giờ đọc phải file ghi dở.
4. Shell bỏ qua `SIGTERM` trong lúc `sleep` chạy foreground. Dùng `trap ... TERM` kết hợp
   `sleep 5 & wait $!` để sidecar phản ứng ngay.
5. Gọi trang từ bên trong Pod: `kubectl exec web -c status-writer -- wget -qO- http://127.0.0.1:8080/`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3a
jp() { kubectl -n "$NS" get pod web -o jsonpath="$1"; }
get() { kubectl -n "$NS" exec web -c status-writer -- wget -qO- "http://127.0.0.1:8080$1"; }

kubectl -n "$NS" wait --for=condition=Ready pod/web --timeout=180s && echo 'PASS: web Ready'
test "$(jp '{.spec.initContainers[?(@.name=="status-writer")].restartPolicy}')" = 'Always' \
  && echo 'PASS: status-writer la sidecar'
TS="$(jp '{.status.initContainerStatuses[?(@.name=="status-writer")].state.running.startedAt}')"
TN="$(jp '{.status.containerStatuses[?(@.name=="nginx")].state.running.startedAt}')"
echo "status-writer start=$TS ; nginx start=$TN" | tee "$EV/b5-thu-tu.txt"
{ test -n "$TS" && test "$(date -d "$TS" +%s)" -le "$(date -d "$TN" +%s)"; } \
  && echo 'PASS: sidecar dang chay va khoi dong khong muon hon nginx'

NODE="$(jp '{.spec.nodeName}')"
get / > "$EV/b5-index.html"
grep -q 'Bookstore' "$EV/b5-index.html" && grep -q "$NODE" "$EV/b5-index.html" \
  && echo "PASS: trang chu co Bookstore va ten node $NODE"

S1="$(get /status.json)"; sleep 12; S2="$(get /status.json)"
echo "$S1"; echo "$S2" | tee "$EV/b5-status.txt"
BAD=0
for K in pod node version updated; do echo "$S2" | grep -q "\"$K\"" || { echo "FAIL: thieu $K"; BAD=1; }; done
{ test "$BAD" -eq 0 && echo "$S2" | grep -q '"pod":"web"'; } && echo 'PASS: status.json du bon truong'
test "$S1" != "$S2" && echo 'PASS: status.json duoc cap nhat theo thoi gian'

R0="$(jp '{.status.containerStatuses[*].restartCount} {.status.initContainerStatuses[*].restartCount}')"
kubectl -n "$NS" label pod web app.kubernetes.io/version=0.1.1 --overwrite
for i in $(seq 1 36); do get /status.json | grep -q '"version":"0.1.1"' && break; sleep 5; done
get /status.json | tee -a "$EV/b5-status.txt" | grep -q '"version":"0.1.1"' \
  && echo 'PASS: status.json nhan version moi qua volume downwardAPI'
R1="$(jp '{.status.containerStatuses[*].restartCount} {.status.initContainerStatuses[*].restartCount}')"
test "$R0" = "$R1" && echo 'PASS: khong container nao khoi dong lai'
kubectl -n "$NS" label pod web app.kubernetes.io/version=0.1.0 --overwrite
```

Thời gian để file trong volume `downwardAPI` cập nhật phụ thuộc chu kỳ đồng bộ của kubelet; vòng
chờ ở trên cho tối đa ba phút.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/web/pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
    app.kubernetes.io/name: nginx
    app.kubernetes.io/version: "0.1.0"
spec:
  volumes:
  - name: html
    emptyDir: {}
  - name: podinfo
    downwardAPI:
      items:
      - path: labels
        fieldRef:
          fieldPath: metadata.labels
  initContainers:
  - name: render-html
    image: busybox:1.37
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    command:
    - sh
    - -c
    - |
      set -e
      cat > /html/index.html <<EOF
      <!doctype html>
      <html lang="vi"><head><meta charset="utf-8"><title>Bookstore</title></head>
      <body>
      <h1>Bookstore</h1>
      <ul id="books"><li>Đang tải danh sách sách...</li></ul>
      <footer>Phục vụ bởi Pod ${POD_NAME} trên node ${NODE_NAME}</footer>
      <script>
      fetch('/api/books').then(r => r.json()).then(bs => {
        document.getElementById('books').innerHTML =
          bs.map(b => '<li>' + b.title + ' — ' + b.author + '</li>').join('');
      }).catch(() => {
        document.getElementById('books').innerHTML = '<li>Chưa kết nối được API</li>';
      });
      </script>
      </body></html>
      EOF
      echo ok > /html/healthz
    volumeMounts:
    - name: html
      mountPath: /html
  - name: status-writer
    image: busybox:1.37
    restartPolicy: Always
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
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
    volumeMounts:
    - name: html
      mountPath: /html
    - name: podinfo
      mountPath: /podinfo
      readOnly: true
  containers:
  - name: nginx
    image: nginxinc/nginx-unprivileged:1.30.5-alpine
    ports:
    - name: http
      containerPort: 8080
    readinessProbe:
      httpGet:
        path: /healthz
        port: http
      periodSeconds: 5
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
```

Ý chính cho ADR 0003:

1. Bài 56: `spec.nodeName` chỉ có qua biến môi trường; `metadata.labels` dạng đầy đủ chỉ có qua
   volume. Biến môi trường được đặt **một lần** khi container khởi động; file trong volume được
   kubelet làm mới qua symlink `..data`. Dùng env cho version thì yêu cầu 5 không đạt.
2. Kubelet chỉ khởi động container chính sau khi sidecar `started`, và chỉ gửi `TERM` cho sidecar
   sau khi container chính đã dừng hẳn. B6 đo điều thứ hai.
3. Chung network namespace thì chung không gian cổng: `status-writer` không được nghe 8080. Bài 46
   gọi đây là việc các container phải tự phối hợp.

</details>

## B6. [BS-3A-06] Tắt êm cho cả ba thành phần

**Loại:** Tính năng, kèm kiểm chứng.

**Bối cảnh:** khi Pod bị xóa — do bạn, do bảo trì node, do phát hành bản mới — request đang xử lý
không được đứt ngang, DB không được tắt kiểu mất điện. Mỗi thành phần cần một thời gian ân hạn và
một cách dừng hợp với nó.

**Yêu cầu:**

1. `db`: `terminationGracePeriodSeconds` đủ cho PostgreSQL ghi checkpoint khi tắt — ít nhất 60
   giây. Chứng minh PostgreSQL nhận đúng tín hiệu mà image khai trong `STOPSIGNAL` và tắt sạch.
2. `web`: hook `preStop` giữ nginx thêm 5 giây trước khi nó nhận tín hiệu dừng, và ghi một dòng
   đánh dấu `PRESTOP` vào log của container `nginx`. Chứng minh thứ tự: `preStop` chạy → nginx nhận
   tín hiệu dừng → **sau đó** `status-writer` mới nhận `TERM`.
3. `api`: đặt `terminationGracePeriodSeconds` tường minh và ghi lý do chọn con số đó.
4. Điều tra: `preStop` có chạy khi container **tự hoàn thành** không? Chứng minh bằng một Pod thử
   nghiệm ở `~/lab-work/3a/`.
5. Cập nhật ba manifest trong repo, triển khai lại toàn bộ.

**Ràng buộc:** không dùng `--force`/`--grace-period=0` với `db`. Pod thử nghiệm ghim
`lab-k8s-worker2` và bị xóa khi xong.

**Đọc lại:**

- Bài [47](../47-pod-lifecycle-vi.md): [*Kết thúc Pod*](../47-pod-lifecycle-vi.md#pod-termination),
  [*Tín hiệu dừng*](../47-pod-lifecycle-vi.md#pod-termination-stop-signals),
  [*Luồng kết thúc Pod*](../47-pod-lifecycle-vi.md#pod-termination-flow),
  [*Buộc kết thúc Pod*](../47-pod-lifecycle-vi.md#pod-termination-forced),
  [*Tắt Pod và sidecar container*](../47-pod-lifecycle-vi.md#termination-with-sidecars).
- Bài [272](../272-attach-handler-lifecycle-event-vi.md) — mục *Thảo luận* về khi nào `preStop`
  **không** được gọi.
- [Mục 4.4 của file dự án](DU-AN-BOOKSTORE.md#44-image) — `STOPSIGNAL` của từng image.

**Bàn giao:** ba manifest đã cập nhật; `docs/adr/0004-tat-em.md`; `~/lab-evidence/3a/b6-*.log`.

**Nghiệm thu:**

- `PASS:` `db` có `terminationGracePeriodSeconds >= 60`; log lúc xóa Pod có *fast shutdown* và
  *database system is shut down*.
- `PASS:` log của `nginx` có dòng `PRESTOP` và dòng `SIGQUIT`; `status-writer` nhận `TERM` muộn hơn
  dòng `PRESTOP` ít nhất 5 giây.
- `PASS:` `api` có `terminationGracePeriodSeconds` tường minh trong spec.
- `PASS:` Pod thử nghiệm tự hoàn thành mà `preStop` không chạy.
- `PASS:` `./scripts/deploy.sh` dựng lại đủ ba Pod Ready sau khi diễn tập.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0004-tat-em.md`:

1. Containerd gửi tín hiệu gì cho `postgres` và cho `nginx`, và vì sao không phải `SIGTERM`? Mỗi tín
   hiệu nghĩa là kiểu tắt nào với phần mềm đó?
2. `preStop` của `web` chạy 5 giây. Đồng hồ ân hạn bắt đầu lúc nào — trước hay sau hook? Nếu hook
   chạy lâu hơn ân hạn thì sao?
3. Một lý do thực tế để `web` cần `preStop` trì hoãn việc dừng nginx là gì? (Gợi ý: nghĩ tới việc
   các thành phần khác của cluster cần thời gian để **thôi gửi** request tới Pod đang tắt.)
4. Vì sao buộc xóa (`--force --grace-period=0`) một Pod DB là nguy hiểm, dù API server trả lời ngay?

<details>
<summary>Gợi ý</summary>

1. Ghi vào log của container từ bên trong hook: ghi vào `/proc/1/fd/1` — stdout của tiến trình 1.
2. Theo dõi log trong lúc xóa Pod: mở `kubectl logs -f --timestamps` ở nền **trước** khi `delete`.
3. Pod thử nghiệm cho yêu cầu 4: một container `busybox` in một dòng rồi `exit 0`, `restartPolicy:
   Never`, có `preStop` ghi một dòng khác vào `/proc/1/fd/1`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script xóa Pod rồi dựng lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3a
cd ~/bookstore && ./scripts/teardown.sh && ./scripts/deploy.sh

G="$(kubectl -n "$NS" get pod db -o jsonpath='{.spec.terminationGracePeriodSeconds}')"
test "${G:-0}" -ge 60 && echo "PASS: db co an han $G giay"
test -n "$(kubectl -n "$NS" get pod api -o jsonpath='{.spec.terminationGracePeriodSeconds}')" \
  && grep -q 'terminationGracePeriodSeconds' k8s/app/api/pod.yaml.tpl \
  && echo 'PASS: api dat an han tuong minh'

# web: preStop -> SIGQUIT -> sidecar TERM
kubectl -n "$NS" logs -f web -c nginx --timestamps > "$EV/b6-nginx.log" 2>&1 & P1=$!
kubectl -n "$NS" logs -f web -c status-writer --timestamps > "$EV/b6-sidecar.log" 2>&1 & P2=$!
sleep 3
kubectl -n "$NS" delete pod web --wait=true --timeout=120s
sleep 2; kill "$P1" "$P2" 2>/dev/null
grep -q 'SIGQUIT' "$EV/b6-nginx.log" && echo 'PASS: nginx nhan SIGQUIT theo STOPSIGNAL cua image'
TP="$(awk '/PRESTOP/{print $1; exit}' "$EV/b6-nginx.log")"
TT="$(awk '/nhan TERM/{print $1; exit}' "$EV/b6-sidecar.log")"
echo "PRESTOP=$TP ; sidecar TERM=$TT"
{ test -n "$TP" && test -n "$TT" && \
  test $(( $(date -d "$TT" +%s) - $(date -d "$TP" +%s) )) -ge 5; } \
  && echo 'PASS: sidecar nhan TERM sau khi preStop va nginx da xong'

# db: tin hieu dung va tat sach
kubectl -n "$NS" logs -f db -c postgres --timestamps > "$EV/b6-db.log" 2>&1 & P3=$!
sleep 3
kubectl -n "$NS" delete pod db --wait=true --timeout=180s
sleep 2; kill "$P3" 2>/dev/null
grep -q 'fast shutdown request' "$EV/b6-db.log" && echo 'PASS: postgres nhan SIGINT -> fast shutdown'
grep -q 'database system is shut down' "$EV/b6-db.log" && echo 'PASS: postgres tat sach'

./scripts/teardown.sh && ./scripts/deploy.sh && echo 'PASS: he thong dung lai du ba Pod'
```

Pod thử nghiệm cho yêu cầu 4 — tự viết `~/lab-work/3a/hook-complete.yaml` theo gợi ý 3, rồi:

```bash
NS=bookstore
kubectl apply -f ~/lab-work/3a/hook-complete.yaml
kubectl -n "$NS" wait --for=jsonpath='{.status.phase}'=Succeeded pod/hook-complete --timeout=180s \
  && echo 'PASS: container tu hoan thanh'
kubectl -n "$NS" logs hook-complete | tee ~/lab-evidence/3a/b6-hook-complete.log
grep -qi 'prestop' ~/lab-evidence/3a/b6-hook-complete.log \
  && echo 'FAIL: preStop da chay' || echo 'PASS: preStop khong chay khi container hoan thanh'
kubectl -n "$NS" delete pod hook-complete
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Thêm vào `k8s/app/db/pod.yaml`, cấp `spec`:

```yaml
  terminationGracePeriodSeconds: 60
```

Thêm vào `k8s/app/api/pod.yaml.tpl`, cấp `spec`:

```yaml
  terminationGracePeriodSeconds: 30
```

Thêm vào `k8s/app/web/pod.yaml` — cấp `spec` và container `nginx`:

```yaml
  terminationGracePeriodSeconds: 30
  ...
  - name: nginx
    ...
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "echo PRESTOP > /proc/1/fd/1; sleep 5"]
```

Pod thử nghiệm:

```yaml
# ~/lab-work/3a/hook-complete.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hook-complete
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "echo XONG-BINH-THUONG; exit 0"]
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "echo PRESTOP-KHONG-NEN-CHAY > /proc/1/fd/1"]
```

Ý chính cho ADR 0004:

1. Bài 47: runtime tôn trọng `STOPSIGNAL` của image. `postgres` khai `SIGINT` — *fast shutdown*:
   cắt kết nối, ghi checkpoint, tắt. `nginx-unprivileged` khai `SIGQUIT` — *graceful shutdown*: phục
   vụ nốt request đang mở rồi thoát.
2. Đồng hồ ân hạn bắt đầu **trước** khi hook được gọi. Hook chưa xong khi hết ân hạn thì kubelet chỉ
   xin thêm một khoảng nhỏ, một lần; phải nới `terminationGracePeriodSeconds` nếu hook dài.
3. Khi Pod bắt đầu tắt, việc loại nó khỏi danh sách đích của Service diễn ra **song song** chứ không
   trước. Vài giây `preStop` cho phép các nơi gửi request kịp thôi gửi. Cơ chế cụ thể học ở Lab 5a.
4. Bài 47: buộc xóa không chờ kubelet xác nhận; tiến trình có thể còn chạy trên node trong khi API
   đã cho phép tạo Pod mới cùng tên. Với DB, hai tiến trình cùng ghi một thư mục dữ liệu là hỏng dữ
   liệu.

</details>

## B7. [BS-3A-07] Diễn tập: Pod không lên — đọc phase và condition

**Loại:** Diễn tập sự cố — mọi Pod diễn tập ghim `lab-k8s-worker2` hoặc không lập lịch được.

**Bối cảnh:** khi một thay đổi làm Pod không lên, người trực phải đọc được trạng thái thay vì đoán.
Ticket này dựng lại ba lỗi điển hình bằng **bản sao hỏng** của manifest `db`, không đụng vào `db`
thật.

**Yêu cầu:** với mỗi kịch bản, tạo Pod diễn tập từ bản sao manifest `db` ở `~/lab-work/3a/`, đặt tên
riêng, ghi nhận **phase**, **trạng thái container** (hoặc init container) và **condition** liên
quan, rồi xóa Pod.

| Kịch bản | Thay đổi so với `db` | Phải quan sát được |
| --- | --- | --- |
| `drill-crash` | bỏ biến mật khẩu superuser | `STATUS` là `CrashLoopBackOff` trong khi phase là `Running`; log lần chạy trước nói rõ nguyên nhân |
| `drill-init-never` | init container ghi sai đường dẫn; `restartPolicy: Never` | phase `Failed`; container chính chưa từng chạy |
| `drill-init-always` | init container ghi sai đường dẫn; `restartPolicy` mặc định | phase `Pending`; condition `Initialized=False`; init container bị khởi động lại |
| `drill-pending` | `nodeSelector` trỏ tới `lab-k8s-worker3` | phase `Pending`; condition `PodScheduled=False` với lý do `Unschedulable` |
| `drill-gate` ⭐ | Pod `busybox` đơn giản có `readinessGates` tên `bookstore.example.com/qa-approved` | `ContainersReady=True` nhưng `Ready=False`; đặt condition thì `Ready=True` |

Ghi kết quả vào `docs/postmortem/3a-pod-khong-len.md`: với mỗi kịch bản, **bước nào** trong năm bước
điều tra `CrashLoopBackOff` của bài 47 chỉ ra nguyên nhân.

**Ràng buộc:** không sửa `k8s/app/db/pod.yaml`. Xóa mọi Pod diễn tập khi xong.

**Đọc lại:**

- Bài [47](../47-pod-lifecycle-vi.md): [*Phase của Pod*](../47-pod-lifecycle-vi.md#pod-phase),
  [*Trạng thái container*](../47-pod-lifecycle-vi.md#container-states),
  [*Cách Pod xử lý sự cố với container*](../47-pod-lifecycle-vi.md#container-restarts),
  [*Chính sách khởi động lại ở cấp Pod*](../47-pod-lifecycle-vi.md#pod-level-container-restart-policy).
- Bài [48](../48-pod-condition-vi.md): [*Các Pod condition vòng đời*](../48-pod-condition-vi.md#lifecycle-pod-conditions),
  [*Độ sẵn sàng nâng cao của Pod*](../48-pod-condition-vi.md#enhanced-pod-readiness).
- Bài [50](../50-init-containers-vi.md), mục *Hành vi chi tiết* — init container với `Never` và với
  `Always`.

**Bàn giao:** `docs/postmortem/3a-pod-khong-len.md`; `~/lab-evidence/3a/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` `drill-crash` — `waiting.reason` là `CrashLoopBackOff`, phase `Running`, log `--previous`
  có thông báo về mật khẩu superuser.
- `PASS:` `drill-init-never` — phase `Failed`, container `postgres` không có `state.running`.
- `PASS:` `drill-init-always` — phase `Pending`, `Initialized=False`, `restartCount` của init
  container `>= 1`.
- `PASS:` `drill-pending` — phase `Pending`, `PodScheduled=False`, lý do `Unschedulable`.
- `PASS:` (⭐) `drill-gate` — `ContainersReady=True`, `Ready=False` với lý do
  `ReadinessGatesNotReady`; sau khi đặt condition thì `Ready=True`.
- `PASS:` không còn Pod nào tên `drill-*`.

<details>
<summary>Gợi ý</summary>

- Bản sao manifest: `sed 's/name: db$/name: drill-crash/' k8s/app/db/pod.yaml > ~/lab-work/3a/drill-crash.yaml`
  rồi sửa tay phần còn lại. Cẩn thận: `sed` trên chỉ đổi dòng `name: db` đứng một mình.
- Sai đường dẫn kiểu gõ nhầm: ghi vào `/initdbb/...` trong khi volume mount ở `/initdb`. Init
  script phải có `set -e` thì lỗi ghi file mới thành mã thoát khác 0; thiếu nó, lệnh `echo` cuối
  cùng thoát 0 và init container báo thành công dù không ghi được gì.
- Đọc một condition: `kubectl get pod X -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}'`.
- `readinessGates` là trường của `spec`; đặt condition tùy chỉnh bằng `kubectl patch ...
  --subresource=status`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã tạo đủ Pod diễn tập</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3a
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2"; }
cond() { jp "$1" "{.status.conditions[?(@.type==\"$2\")].$3}"; }

for i in $(seq 1 36); do
  test "$(jp drill-crash '{.status.containerStatuses[0].state.waiting.reason}')" = 'CrashLoopBackOff' && break
  sleep 5
done
kubectl -n "$NS" get pods -o wide | tee "$EV/b7-pods.txt"

{ test "$(jp drill-crash '{.status.containerStatuses[0].state.waiting.reason}')" = 'CrashLoopBackOff' && \
  test "$(jp drill-crash '{.status.phase}')" = 'Running'; } \
  && echo 'PASS: drill-crash — CrashLoopBackOff nhung phase Running'
kubectl -n "$NS" logs drill-crash --previous 2>&1 | tee "$EV/b7-crash.txt" \
  | grep -qi 'superuser password' && echo 'PASS: log lan truoc noi ro nguyen nhan'

kubectl -n "$NS" wait --for=jsonpath='{.status.phase}'=Failed pod/drill-init-never --timeout=180s
{ test "$(jp drill-init-never '{.status.phase}')" = 'Failed' && \
  test -z "$(jp drill-init-never '{.status.containerStatuses[0].state.running}')"; } \
  && echo 'PASS: drill-init-never — Failed, container chinh chua tung chay'

for i in $(seq 1 24); do test "$(jp drill-init-always '{.status.initContainerStatuses[0].restartCount}')" -ge 1 && break; sleep 5; done
{ test "$(jp drill-init-always '{.status.phase}')" = 'Pending' && \
  test "$(cond drill-init-always Initialized status)" = 'False' && \
  test "$(jp drill-init-always '{.status.initContainerStatuses[0].restartCount}')" -ge 1; } \
  && echo 'PASS: drill-init-always — Pending, Initialized=False, init bi khoi dong lai'

{ test "$(jp drill-pending '{.status.phase}')" = 'Pending' && \
  test "$(cond drill-pending PodScheduled status)" = 'False' && \
  test "$(cond drill-pending PodScheduled reason)" = 'Unschedulable'; } \
  && echo 'PASS: drill-pending — PodScheduled=False, Unschedulable'

if kubectl -n "$NS" get pod drill-gate >/dev/null 2>&1; then
  { test "$(cond drill-gate ContainersReady status)" = 'True' && \
    test "$(cond drill-gate Ready status)" = 'False' && \
    test "$(cond drill-gate Ready reason)" = 'ReadinessGatesNotReady'; } \
    && echo 'PASS: drill-gate — ContainersReady=True, Ready=False'
  kubectl -n "$NS" patch pod drill-gate --subresource=status -p \
    '{"status":{"conditions":[{"type":"bookstore.example.com/qa-approved","status":"True","lastProbeTime":null,"lastTransitionTime":"2026-10-13T00:00:00Z"}]}}'
  kubectl -n "$NS" wait --for=condition=Ready pod/drill-gate --timeout=60s \
    && echo 'PASS: drill-gate — dat condition thi Pod Ready'
fi

kubectl -n "$NS" delete pod drill-crash drill-init-never drill-init-always drill-pending drill-gate \
  --ignore-not-found --wait=true
test -z "$(kubectl -n "$NS" get pods -o name | grep 'drill-')" && echo 'PASS: da xoa het Pod dien tap'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Bốn kịch bản đầu đều là bản sao của `k8s/app/db/pod.yaml`, giữ `nodeSelector` về
`lab-k8s-worker2`, chỉ khác:

- `drill-crash`: xóa khối biến `POSTGRES_PASSWORD`.
- `drill-init-never`: thêm `restartPolicy: Never` ở `spec`; trong init container đổi
  `cat > /initdb/01-schema.sql` thành `cat > /initdbb/01-schema.sql`.
- `drill-init-always`: như trên nhưng không thêm `restartPolicy`.
- `drill-pending`: đổi giá trị `nodeSelector` thành `lab-k8s-worker3`.

```yaml
# ~/lab-work/3a/drill-gate.yaml
apiVersion: v1
kind: Pod
metadata:
  name: drill-gate
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  readinessGates:
  - conditionType: "bookstore.example.com/qa-approved"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
```

Ý chính cho postmortem:

- `drill-crash`: bước 1 (log) chỉ thẳng nguyên nhân — entrypoint báo DB chưa khởi tạo và chưa có
  mật khẩu superuser. `CrashLoopBackOff` là trạng thái chờ của container, không phải phase.
- `drill-init-*`: bước 2 (sự kiện) và `kubectl logs -c <tên init>`. Với `Never`, một init hỏng làm
  cả Pod `Failed`; với `Always`, init container dùng `OnFailure` và bị khởi động lại mãi trong khi Pod
  nằm ở `Pending`.
- `drill-pending`: không có log nào vì chưa có container; nguyên nhân nằm ở condition
  `PodScheduled` và sự kiện của scheduler.
- `drill-gate`: Pod chỉ Ready khi mọi container Ready **và** mọi condition trong `readinessGates` là
  `True`; kubelet ghi lý do vào `reason`.

</details>

## B8. [BS-3A-08] Gỡ lỗi container không có shell

**Loại:** Điều tra.

**Bối cảnh:** image `postgrest` chỉ có một binary ([mục 4.4](DU-AN-BOOKSTORE.md#44-image)). Khi
`api` có vấn đề, `kubectl exec ... -- sh` không dùng được. Đội cần một quy trình gỡ lỗi chuẩn.

**Yêu cầu:**

1. Chứng minh `kubectl exec` vào `postgrest` với `sh` thất bại.
2. Gắn một ephemeral container `busybox` tên `debugger` vào Pod `api`, nhắm vào container
   `postgrest` để nhìn thấy tiến trình của nó.
3. Từ `debugger`: liệt kê tiến trình và thấy `postgrest`; gọi `/ready` của admin server qua
   `127.0.0.1`.
4. Chứng minh ba giới hạn của ephemeral container bằng `--dry-run=server`: không thêm được bằng cập
   nhật Pod thông thường; không xóa được sau khi đã thêm; không có probe.
5. Chứng minh `postgrest` không bị khởi động lại vì việc gắn debugger.
6. Viết `docs/runbook-debug-api.md` — quy trình gỡ lỗi `api` cho người trực, và một ADR ngắn: có
   bật `shareProcessNamespace` cho Pod `web` không.

**Ràng buộc:** không đổi image của `api`, không thêm shell vào Pod bằng cách sửa manifest.

**Đọc lại:**

- Bài [52](../52-ephemeral-containers-vi.md) — cả bài, đặc biệt đoạn về distroless image.
- Bài [292](../292-share-process-namespace-vi.md) — mục *Hiểu về chia sẻ process namespace*: ba hệ
  quả bảo mật.
- Bài [46](../46-pods-vi.md), mục *Subresource của Pod*.

**Bàn giao:** `docs/runbook-debug-api.md`; `docs/adr/0005-share-process-namespace.md`;
`~/lab-evidence/3a/b8-*.txt`.

**Nghiệm thu:**

- `PASS:` `exec` với `sh` vào `postgrest` thất bại.
- `PASS:` ephemeral container `debugger` đang chạy trong Pod `api`.
- `PASS:` `debugger` thấy tiến trình `postgrest` và gọi được `/ready`.
- `PASS:` ba thao tác dry-run bị từ chối đúng như bài 52 mô tả.
- `PASS:` `restartCount` của `postgrest` không đổi.

**Câu hỏi thiết kế:**

1. `debugger` thấy được tiến trình của `postgrest` nhờ cơ chế nào? Không có cơ chế đó thì nó thấy
   gì?
2. Với `shareProcessNamespace: true` cho `web`, sidecar có thể gửi tín hiệu cho nginx để nạp lại cấu
   hình. Ba hệ quả nào bài 292 cảnh báo khiến bạn chấp nhận hoặc từ chối đánh đổi này?
3. Ai được phép gắn ephemeral container vào Pod `api` thì người đó đọc được những gì của `api`? Ghi
   thành nợ `TD-07`.

<details>
<summary>Gợi ý</summary>

- `kubectl debug` có cờ chọn image, tên container và container đích. Không cần `-it`; cho
  `debugger` chạy `sleep 3600` rồi `kubectl exec -c debugger` vào nó.
- Ba lệnh dry-run: một `kubectl patch pod` thường thêm `spec.ephemeralContainers`; một `patch` với
  `--subresource=ephemeralcontainers` kiểu JSON patch `remove`; một `patch` vào cùng subresource thêm
  container có `livenessProbe`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3a
rc() { kubectl -n "$NS" get pod api -o jsonpath='{.status.containerStatuses[?(@.name=="postgrest")].restartCount}'; }
R0="$(rc)"

kubectl -n "$NS" exec api -c postgrest -- sh -c true > "$EV/b8-no-shell.txt" 2>&1 \
  && echo 'FAIL: postgrest co shell' || echo 'PASS: exec sh vao postgrest that bai'

S="$(kubectl -n "$NS" get pod api -o jsonpath='{.status.ephemeralContainerStatuses[?(@.name=="debugger")].state.running.startedAt}')"
test -n "$S" && echo 'PASS: debugger dang chay trong Pod api'

kubectl -n "$NS" exec api -c debugger -- ps | tee "$EV/b8-ps.txt" | grep -q 'postgrest' \
  && echo 'PASS: debugger thay tien trinh postgrest'
kubectl -n "$NS" exec api -c debugger -- wget -q -O /dev/null http://127.0.0.1:3001/ready \
  && echo 'PASS: debugger goi duoc /ready'

kubectl -n "$NS" patch pod api --dry-run=server \
  -p '{"spec":{"ephemeralContainers":[{"name":"them-bang-tay","image":"busybox:1.37"}]}}' >/dev/null 2>&1 \
  && echo 'FAIL: them duoc bang cap nhat Pod thuong' || echo 'PASS: cap nhat Pod thuong khong them duoc'
kubectl -n "$NS" patch pod api --subresource=ephemeralcontainers --dry-run=server --type=json \
  -p '[{"op":"remove","path":"/spec/ephemeralContainers/0"}]' >/dev/null 2>&1 \
  && echo 'FAIL: xoa duoc ephemeral container' || echo 'PASS: da them thi khong xoa duoc'
kubectl -n "$NS" patch pod api --subresource=ephemeralcontainers --dry-run=server \
  -p '{"spec":{"ephemeralContainers":[{"name":"dbg2","image":"busybox:1.37","livenessProbe":{"exec":{"command":["true"]}}}]}}' >/dev/null 2>&1 \
  && echo 'FAIL: dat duoc probe' || echo 'PASS: ephemeral container khong duoc co probe'

test "$(rc)" -eq "$R0" && echo 'PASS: postgrest khong bi khoi dong lai'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
kubectl -n bookstore debug pod/api --image=busybox:1.37 --target=postgrest -c debugger -- sleep 3600
kubectl -n bookstore exec api -c debugger -- ps
kubectl -n bookstore exec api -c debugger -- wget -S -O /dev/null http://127.0.0.1:3001/ready
```

Ý chính:

1. `--target` cho `debugger` vào process namespace của `postgrest`. Không có nó, `debugger` chỉ thấy
   tiến trình của chính mình; nó vẫn chung **network** namespace với cả Pod nên vẫn gọi được
   `127.0.0.1:3001`.
2. Bài 292: tiến trình của container không còn là PID 1; mọi container thấy `/proc` của nhau, kể cả
   biến môi trường; filesystem đọc được qua `/proc/$pid/root`. Với `web` hiện tại, sidecar không cần
   gửi tín hiệu cho nginx, nên lợi ích không bù được rủi ro — không bật.
3. Người đó nhìn thấy tiến trình, gọi được endpoint nội bộ, và — giống ai đọc được spec của Pod —
   thấy mật khẩu DB nằm trong biến môi trường. `TD-07` giảm ở Lab 3b (mật khẩu ra khỏi spec), giới
   hạn ở Lab 9a (quyền tạo ephemeral container là quyền riêng trên subresource).

</details>

## B9. [BS-3A-09] Tăng cứng ở mức Pod

**Loại:** Tính năng.

**Bối cảnh:** hệ thống cũ chạy mọi thứ bằng root (`P7`). Sprint này chưa có Pod Security Admission
(giai đoạn 9), nhưng đã có đủ công cụ ở mức Pod để giảm rủi ro cho hai thành phần không cần root, và
để đánh dấu thành phần nào quan trọng nhất khi cluster thiếu chỗ.

**Yêu cầu:**

1. `api` và container `nginx` của `web`: chạy non-root (kubelet phải **kiểm** điều này) và không
   được leo thang đặc quyền.
2. Toàn bộ Pod `web` — cả init container và sidecar — chạy cùng một user không phải root mà image
   nginx dùng.
3. Kiểm điều kiện tiên quyết của user namespace trên hai worker. Nếu đạt, `web` chạy với
   `hostUsers: false` và chứng minh root bên trong không phải root trên host.
4. PriorityClass `bookstore-critical` ở `k8s/cluster/`, label theo hợp đồng; `db` và `api` dùng nó.
5. Thử nghiệm: một Pod ở `~/lab-work/3a/` ghim `lab-k8s-master` bằng `nodeSelector` — chứng minh nó
   `Pending` vì taint; thêm toleration thì nó chạy. Xóa ngay sau khi chứng minh.
6. Ghi ADR: vì sao `db` **chưa** chạy non-root ở sprint này (nợ `TD-06`).

**Ràng buộc:** không đặt `.spec.priority` trực tiếp. Không dùng `privileged`. Không gỡ taint của
master.

**Đọc lại:**

- Bài [60](../60-advanced-pod-config-vi.md): [PriorityClasses](../60-advanced-pod-config-vi.md#priorityclasses),
  [*Security context*](../60-advanced-pod-config-vi.md#security-context), mục *Toleration*.
- Bài [55](../55-user-namespaces-vi.md): *Trước khi bạn bắt đầu*,
  [*Hiểu về user namespace cho Pod*](../55-user-namespaces-vi.md#pods-and-userns), *Các hạn chế*.
- Bài [295](../295-user-namespaces-tasks-vi.md#create-pod).
- [Mục 4.4 của file dự án](DU-AN-BOOKSTORE.md#44-image) — user của từng image.

**Bàn giao:** manifest đã cập nhật; `k8s/cluster/priorityclass-critical.yaml`;
`docs/adr/0006-tang-cung-pod.md`; `~/lab-evidence/3a/b9-*.txt`.

**Nghiệm thu:**

- `PASS:` `api` và `nginx` có `runAsNonRoot: true` và `allowPrivilegeEscalation: false` trong spec, và
  Pod Ready (kubelet đã kiểm).
- `PASS:` mọi container của `web` chạy với uid 101.
- `PASS:` (nếu tiên quyết đạt) `hostUsers` là `false`; `uid_map` trong `nginx` ánh xạ sang một UID
  khác 0 trên host, độ dài 65536.
- `PASS:` PriorityClass mang label dự án; `spec.priority` của `db` và `api` bằng `value` của nó.
- `PASS:` Pod thử nghiệm không toleration nằm `Pending` với lý do `Unschedulable` nhắc tới taint; có
  toleration thì chạy trên `lab-k8s-master`; cả hai đã bị xóa.

**Câu hỏi thiết kế:**

1. `runAsNonRoot: true` với image `postgrest` thành công, nhưng với image `postgres` ở cấu hình hiện
   tại thì sao? Vì sao? (Đọc lại ghi chú về entrypoint của `postgres` ở mục 4.4.)
2. `hostUsers: false` cắt đường nào tới host, và nó **không** đổi gì về quyền sở hữu file trên
   volume?
3. PriorityClass chỉ phát huy khi nào? Nó **không** làm gì khi cluster còn dư chỗ?

<details>
<summary>Gợi ý</summary>

- `securityContext` cấp Pod đặt mặc định cho mọi container, kể cả init container và sidecar.
- Điều kiện tiên quyết: kernel và filesystem của `/var/lib/kubelet` trên worker.
  `ssh lab-k8s-worker1 'uname -r; stat -f -c %T /var/lib/kubelet'`.
- Toleration cho taint control plane: key `node-role.kubernetes.io/control-plane`, effect
  `NoSchedule`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã triển khai lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3a
cd ~/bookstore && ./scripts/teardown.sh && ./scripts/deploy.sh
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2"; }

for X in "api postgrest" "web nginx"; do
  set -- $X
  RNR="$(jp "$1" "{.spec.securityContext.runAsNonRoot}{.spec.containers[?(@.name==\"$2\")].securityContext.runAsNonRoot}")"
  APE="$(jp "$1" "{.spec.containers[?(@.name==\"$2\")].securityContext.allowPrivilegeEscalation}")"
  { echo "$RNR" | grep -q true && test "$APE" = 'false'; } && echo "PASS: $1/$2 non-root, khong leo thang"
done

BAD=0
for C in render-html status-writer nginx; do
  if [ "$C" = render-html ]; then
    U="$(jp web '{.spec.securityContext.runAsUser}')"
  else
    U="$(kubectl -n "$NS" exec web -c "$C" -- id -u)"
  fi
  echo "$C uid=$U" | tee -a "$EV/b9-uid.txt"
  test "$U" = '101' || BAD=1
done
test "$BAD" -eq 0 && echo 'PASS: moi container cua web chay uid 101'

ssh lab-k8s-worker1 'uname -r; stat -f -c %T /var/lib/kubelet' | tee "$EV/b9-prereq.txt"
if [ "$(jp web '{.spec.hostUsers}')" = 'false' ]; then
  MAP="$(kubectl -n "$NS" exec web -c nginx -- cat /proc/self/uid_map)"
  echo "$MAP" | tee "$EV/b9-uidmap.txt"
  echo "$MAP" | awk '{ exit !($1==0 && $2!=0 && $3==65536) }' \
    && echo 'PASS: root trong web anh xa sang UID khong dac quyen tren host'
fi

V="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}')"
test "$(kubectl get priorityclass bookstore-critical -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' \
  && echo 'PASS: PriorityClass mang label du an'
{ test "$(jp db '{.spec.priority}')" = "$V" && test "$(jp api '{.spec.priority}')" = "$V"; } \
  && echo "PASS: db va api co spec.priority = $V"
```

Pod thử nghiệm taint — tự viết `~/lab-work/3a/tren-master.yaml` và `~/lab-work/3a/tren-master-tol.yaml`,
rồi:

```bash
NS=bookstore; EV=~/lab-evidence/3a
kubectl apply -f ~/lab-work/3a/tren-master.yaml -f ~/lab-work/3a/tren-master-tol.yaml
sleep 15
R="$(kubectl -n "$NS" get pod tren-master -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}')"
kubectl -n "$NS" describe pod tren-master | tee "$EV/b9-taint.txt" | grep -qi 'taint' \
  && test "$R" = 'Unschedulable' && echo 'PASS: khong toleration thi taint chan Pod'
kubectl -n "$NS" wait --for=condition=Ready pod/tren-master-tol --timeout=120s
test "$(kubectl -n "$NS" get pod tren-master-tol -o jsonpath='{.spec.nodeName}')" = 'lab-k8s-master' \
  && echo 'PASS: co toleration thi Pod len duoc master'
kubectl -n "$NS" delete pod tren-master tren-master-tol --wait=true && echo 'PASS: da xoa Pod thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/cluster/priorityclass-critical.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: bookstore-critical
  labels:
    app.kubernetes.io/part-of: bookstore
value: 100000
globalDefault: false
description: "Thanh phan cot loi cua Bookstore: db, api"
```

Thêm vào `spec` của `db` và `api`:

```yaml
  priorityClassName: bookstore-critical
```

Thêm vào container `postgrest`:

```yaml
    securityContext:
      runAsNonRoot: true
      allowPrivilegeEscalation: false
```

Thêm vào `spec` của `web`, và vào container `nginx`:

```yaml
  hostUsers: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    runAsGroup: 101
  ...
  - name: nginx
    ...
    securityContext:
      allowPrivilegeEscalation: false
```

Pod thử nghiệm:

```yaml
# ~/lab-work/3a/tren-master.yaml — bản -tol thêm khối tolerations và đổi tên thành tren-master-tol
apiVersion: v1
kind: Pod
metadata:
  name: tren-master
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-master
  # tolerations:
  # - key: node-role.kubernetes.io/control-plane
  #   operator: Exists
  #   effect: NoSchedule
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
```

Ý chính cho ADR 0006:

1. Image `postgres` không khai `USER`; entrypoint khởi động bằng root để chỉnh quyền thư mục dữ
   liệu rồi mới hạ xuống uid 999. `runAsNonRoot: true` mà không đặt `runAsUser` thì kubelet từ chối
   khởi động container. Chạy được non-root cần chuẩn bị thư mục dữ liệu khác đi — để tới Lab 9b
   (`TD-06`).
2. Bài 55: root trong container ánh xạ sang một UID không đặc quyền trên host, capability chỉ có
   hiệu lực trong user namespace của Pod. `runAsUser`, `fsGroup` vẫn trỏ tới user **bên trong**
   container, nên quyền sở hữu file trên volume không đổi.
3. Bài 60: chỉ khi Pod không lập lịch được vì thiếu tài nguyên — lúc đó scheduler có thể chiếm chỗ
   Pod ưu tiên thấp hơn. Còn dư chỗ thì độ ưu tiên không đổi gì; cơ chế chiếm chỗ học ở Lab 7a.

</details>

## B10. [BS-3A-10] ⭐ Đánh giá image volume

**Loại:** Điều tra — tùy chọn.

**Bối cảnh:** một ý tưởng cho các sprint sau: đóng gói nội dung tĩnh của `web` thành một OCI image
riêng, rồi mount vào Pod mà không cần init container sinh file. Trước khi quyết, kiểm cluster có
hỗ trợ không và hành vi mount ra sao.

**Yêu cầu:** Pod thử nghiệm `img-vol` ở `~/lab-work/3a/` mount image `busybox` đúng tag A1.3 làm
volume tại `/img`. Chứng minh: API server chấp nhận trường; nội dung image có trong `/img`; mount là
**chỉ đọc**. Ghi ADR có dùng cách này cho `web` không. Xóa Pod khi xong.

**Đọc lại:** bài [285](../285-image-volumes-vi.md) — mục *Trước khi bạn bắt đầu* và
[*Chạy một Pod sử dụng image volume*](../285-image-volumes-vi.md#create-pod).

**Nghiệm thu:**

- `PASS:` dry-run server giữ nguyên `volumes[].image.reference`.
- `PASS:` `/img/bin/busybox` tồn tại trong container.
- `PASS:` ghi vào `/img` thất bại.

<details>
<summary>Script nghiệm thu và lời giải tham khảo</summary>

```yaml
# ~/lab-work/3a/img-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: img-vol
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  volumes:
  - name: goi-image
    image:
      reference: busybox:1.37
      pullPolicy: IfNotPresent
  containers:
  - name: shell
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - name: goi-image
      mountPath: /img
```

```bash
NS=bookstore
R="$(kubectl create -f ~/lab-work/3a/img-vol.yaml --dry-run=server \
     -o jsonpath='{.spec.volumes[?(@.name=="goi-image")].image.reference}')"
test "$R" = 'busybox:1.37' && echo 'PASS: API server nhan truong volumes[].image'
kubectl apply -f ~/lab-work/3a/img-vol.yaml
kubectl -n "$NS" wait --for=condition=Ready pod/img-vol --timeout=240s
kubectl -n "$NS" exec img-vol -- test -e /img/bin/busybox && echo 'PASS: noi dung image co trong /img'
kubectl -n "$NS" exec img-vol -- sh -c 'touch /img/thu-ghi' >/dev/null 2>&1 \
  && echo 'FAIL: ghi duoc vao image volume' || echo 'PASS: image volume chi doc'
kubectl -n "$NS" delete pod img-vol --wait=true
```

Nếu Pod không khởi động được: API server chấp nhận trường nhưng container runtime chưa hỗ trợ — đối
chiếu runtime ở [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), ghi kết quả vào ADR.
Lý do để chưa dùng cho `web` dù cluster hỗ trợ: dự án chưa có quy trình build và đẩy image nội
dung; cái giá đó lớn hơn lợi ích ở quy mô hiện tại.

</details>

## B11. [BS-3A-11] Sprint review: giới hạn của Pod trần

**Loại:** Điều tra.

**Bối cảnh:** trước khi đóng sprint, đội phải liệt kê rõ hệ thống hiện tại **yếu ở đâu** và vì sao,
để sprint sau có đầu vào. Mỗi điểm yếu phải có bằng chứng, không phải cảm nhận.

**Yêu cầu:**

1. Chứng minh Pod `web` không có ai đứng sau: không có `ownerReferences`; xóa thì không có gì tạo
   lại.
2. Tạo lại `web` từ repo; chứng minh đó là Pod **khác** dù cùng tên.
3. Bằng `--dry-run=server` trên Pod `api`: đổi image được chấp nhận; đổi biến môi trường và thêm
   `nodeSelector` bị từ chối.
4. Điền `docs/tech-debt.md` với `TD-01` đến `TD-07` theo
   [sổ nợ của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án), mỗi dòng trỏ tới một bằng chứng
   trong `~/lab-evidence/3a/`.

**Đọc lại:** bài [45](../45-workloads-vi.md); bài [46](../46-pods-vi.md): mục *Pod và controller*,
*Cập nhật và thay thế Pod*, [*Generation của Pod*](../46-pods-vi.md#pod-generation); bài
[47](../47-pod-lifecycle-vi.md#pod-fault-recovery) — *Pod và việc khôi phục sau lỗi*.

**Bàn giao:** `docs/tech-debt.md`; `~/lab-evidence/3a/b11-*.txt`.

**Nghiệm thu:**

- `PASS:` `web` không có `ownerReferences`; 30 giây sau khi xóa vẫn không có Pod `web`.
- `PASS:` Pod `web` tạo lại có `metadata.uid` khác.
- `PASS:` dry-run đổi image được chấp nhận; đổi env và thêm `nodeSelector` bị từ chối.
- `PASS:` `docs/tech-debt.md` có đủ bảy mã `TD-01` → `TD-07`.

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3a
cd ~/bookstore

test -z "$(kubectl -n "$NS" get pod web -o jsonpath='{.metadata.ownerReferences}')" \
  && echo 'PASS: web khong co ownerReferences'
U0="$(kubectl -n "$NS" get pod web -o jsonpath='{.metadata.uid}')"
kubectl -n "$NS" delete pod web --wait=true
sleep 30
kubectl -n "$NS" get pod web >/dev/null 2>&1 \
  && echo 'FAIL: co thu gi do tao lai web' || echo 'PASS: khong ai tao lai web'

kubectl apply -f k8s/app/web/
kubectl -n "$NS" wait --for=condition=Ready pod/web --timeout=180s
U1="$(kubectl -n "$NS" get pod web -o jsonpath='{.metadata.uid}')"
echo "uid cu=$U0 ; uid moi=$U1" | tee "$EV/b11-uid.txt"
test "$U0" != "$U1" && echo 'PASS: Pod tao lai la Pod khac'

kubectl -n "$NS" patch pod api --dry-run=server \
  -p '{"spec":{"containers":[{"name":"postgrest","image":"postgrest/postgrest:tag-thu-doi"}]}}' >/dev/null 2>&1 \
  && echo 'PASS: doi image duoc chap nhan'
kubectl -n "$NS" patch pod api --dry-run=server \
  -p '{"spec":{"containers":[{"name":"postgrest","env":[{"name":"PGRST_DB_SCHEMAS","value":"khac"}]}]}}' \
  2>&1 | tee "$EV/b11-immutable.txt" | grep -qi 'forbidden' && echo 'PASS: doi env bi tu choi'
kubectl -n "$NS" patch pod api --dry-run=server -p '{"spec":{"nodeSelector":{"disktype":"ssd"}}}' >/dev/null 2>&1 \
  && echo 'FAIL: them nodeSelector duoc chap nhan' || echo 'PASS: them nodeSelector bi tu choi'

BAD=0
for T in TD-01 TD-02 TD-03 TD-04 TD-05 TD-06 TD-07; do grep -q "$T" docs/tech-debt.md || { echo "FAIL: thieu $T"; BAD=1; }; done
test "$BAD" -eq 0 && echo 'PASS: tech-debt.md du TD-01 -> TD-07'
```

</details>

<details>
<summary>Lời giải tham khảo — bảng nợ</summary>

```markdown
| Mã | Nợ | Phát sinh | Trả ở | Trạng thái | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| TD-01 | Pod trần, không ai tạo lại | 3a | 4a, 4b | mở | ~/lab-evidence/3a/b11-uid.txt |
| TD-02 | Mật khẩu trong manifest | 3a | 3b | mở | k8s/app/db/pod.yaml |
| TD-03 | api tìm db bằng IP Pod | 3a | 5a | mở | ~/lab-evidence/3a/b4-b.txt |
| TD-04 | Dữ liệu trên emptyDir | 3a | 6a | mở | kịch bản B của B4 |
| TD-05 | Chưa có requests/limits | 3a | 3c | mở | kubectl get pod -o yaml |
| TD-06 | db khởi động bằng root | 3a | 9b | mở | docs/adr/0006-tang-cung-pod.md |
| TD-07 | Debug/exec vào api đọc được mật khẩu | 3a | 3b, 9a | mở | docs/adr/0005-share-process-namespace.md |
```

Bài 45 khép lại lập luận: lỗi node là chung cuộc với Pod trên node đó, và Pod trần không có
controller nào thay nó. Sprint 4 (Lab 4a) bắt đầu trả `TD-01`.

</details>

## B12. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 1, rồi trả cluster về
đúng `01-cluster-ready`.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 3a — chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 3a: ba Pod trần db, api, web.
set -u
NS=bookstore
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
cond() { kubectl -n "$NS" get pod "$1" -o jsonpath="{.status.conditions[?(@.type==\"$2\")].status}" 2>/dev/null; }

for P in db api web; do
  [ "$(cond "$P" Ready)" = 'True' ] && pass "Pod $P Ready" || fail "Pod $P chua Ready"
done

N="$(kubectl -n "$NS" exec db -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 5 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach"

API_IP="$(kubectl -n "$NS" get pod api -o jsonpath='{.status.podIP}' 2>/dev/null)"
T="$(kubectl -n "$NS" exec web -c status-writer -- wget -qO- "http://${API_IP}:3000/books" 2>/dev/null | grep -o '"title"' | wc -l)"
[ "$T" -ge 5 ] && pass "api /books tra $T cuon sach" || fail "api /books tra $T cuon sach"

kubectl -n "$NS" exec web -c status-writer -- wget -qO- http://127.0.0.1:8080/ 2>/dev/null | grep -q 'Bookstore' \
  && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
kubectl -n "$NS" exec web -c status-writer -- wget -qO- http://127.0.0.1:8080/status.json 2>/dev/null | grep -q '"pod":"web"' \
  && pass 'web phuc vu status.json' || fail 'web khong phuc vu status.json'

[ "$(kubectl get priorityclass bookstore-critical -o jsonpath='{.metadata.name}' 2>/dev/null)" = 'bookstore-critical' ] \
  && pass 'co PriorityClass bookstore-critical' || fail 'thieu PriorityClass bookstore-critical'

exit "$FAIL"
EOF
```

2. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh && ./tests/smoke.sh | tee ~/lab-evidence/3a/b12-smoke.txt
```

3. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-3a: sprint 1 — Pod hoa db, api, web'
git tag lab-3a
git bundle create ~/bookstore-lab-3a.bundle --all && git bundle verify ~/bookstore-lab-3a.bundle
```

Chép `~/bookstore-lab-3a.bundle` về máy host bằng `scp` trước khi đi tiếp.

4. Dọn và trả cluster:

```bash
kubectl -n bookstore delete pod -l '!app.kubernetes.io/part-of' --ignore-not-found --wait=true
rm -rf ~/lab-work/3a
cd ~/bookstore && ./scripts/teardown.sh
```

5. Gate cuối:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
test -z "$(git -C ~/bookstore status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git -C ~/bookstore describe --tags --exact-match HEAD | grep -qx 'lab-3a' && echo 'PASS: HEAD mang tag lab-3a'
test -s ~/bookstore-lab-3a.bundle && echo 'PASS: co bundle'
test "$(sl kubectl get namespace,priorityclass -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 \
  && echo 'PASS: khong con object nao cua du an'
test ! -e ~/lab-work/3a && echo 'PASS: da don lab-work'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** `tests/smoke.sh` ở bước 2 in toàn `PASS:`; sáu dòng `PASS:` của gate cuối xuất hiện; ba
node `Ready`; lệnh field selector trả `No resources found`; CoreDNS đủ replica; namespace `default`
không có Pod. Thư mục `~/lab-evidence/3a/` **giữ lại**. Cluster trở về `01-cluster-ready`; **không
tạo snapshot mới**.

---

## 3. Checkpoint 3a

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Init container của `db` chạy lúc nào, bao nhiêu lần trong đời Pod, và vì sao nó phải bất biến
      kết quả? Script trong `/docker-entrypoint-initdb.d` chạy lại khi nào?
- [ ] Vẽ ba probe của `api`: mỗi cái gọi endpoint nào, kết quả `Failure` dẫn tới hành động gì của
      kubelet, và startup probe chặn hai cái kia tới khi nào.
- [ ] Ở B4, vì sao kịch bản A tự lành còn kịch bản B thì không? Container restart và Pod replacement
      khác nhau ở UID, IP, volume ra sao?
- [ ] `kubectl get pod` hiện `CrashLoopBackOff`. Phase là gì? Bạn chẩn đoán theo những bước nào, và
      bước nào cho `drill-crash` đáp án?
- [ ] Init container hỏng: điều gì khác nhau giữa Pod `restartPolicy: Never` và Pod mặc định? Phase,
      condition `Initialized` của mỗi trường hợp?
- [ ] Pod `Pending` vì `nodeSelector` sai: đọc lý do ở đâu? Vì sao không có log container nào để đọc?
- [ ] Sidecar khai ở đâu, khởi động và tắt theo thứ tự nào so với container chính? Vì sao mã thoát
      khác 0 của sidecar lúc Pod tắt là bình thường?
- [ ] Vì sao tên node vào trang chủ qua biến môi trường nhưng version vào `status.json` qua volume?
      Đổi label thì cái nào thấy giá trị mới?
- [ ] Kể luồng tắt êm của `web` từ lúc `kubectl delete pod web`: hook, tín hiệu, sidecar, ân hạn,
      `SIGKILL`. `nginx` và `postgres` nhận tín hiệu gì, vì sao?
- [ ] Ephemeral container: vì sao không thêm được bằng `kubectl edit`, ba nhóm trường nào bị cấm, và
      `--target` cho nó thấy gì?
- [ ] `hostUsers: false` đổi gì với tiến trình root trong container? PriorityClass `bookstore-critical`
      có tác dụng gì khi cluster còn dư chỗ?
- [ ] Kể bảy nợ kỹ thuật của sprint 1, mỗi nợ trả ở lab nào và bằng cơ chế gì.

Nợ lab chưa trả sau lab này — nói được **vì sao chưa trả**: thực hành static Pod, request/limit
hiệu dụng của init container và sidecar, phase `Unknown`, ảnh hưởng của readiness lên EndpointSlice
(mục 1.1).

### Bài giải thích cuối cùng

Trong vài phút, kể lại cuộc đời của Pod `api` của bạn: từ lúc `scripts/deploy.sh` chèn IP của `db`
vào manifest, qua init container chờ DB, startup probe, lúc Pod Ready; rồi kịch bản B4-B — DB biến
mất, readiness và liveness mỗi cái làm gì; tới lúc bạn xóa `api` và nó tắt êm. Ở mỗi mốc nói rõ:
condition nào vừa đổi, container nào đang chạy, ai quyết định Pod có nhận lưu lượng hay không. Kết
thúc bằng việc chỉ ra Lab 4a và Lab 5a sẽ thay đổi câu chuyện này ở điểm nào.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `crictl pull` báo `toomanyrequests` | Docker Hub giới hạn số lần kéo ẩn danh theo IP | Chờ rồi chạy lại; các node đã có image thì không kéo lại |
| B2: `db` chạy nhưng không có schema `api` | Script SQL lỗi ở lần chạy đầu; lần khởi động lại thấy thư mục dữ liệu không rỗng nên bỏ qua script | Đọc `kubectl logs db -c postgres` tìm dòng `ERROR`; sửa script rồi **xóa Pod** để có `emptyDir` mới |
| B2: Pod `db` treo `Init:0/1` hoặc init `Error` | Heredoc sai: dòng kết thúc không ở cột 0 sau khi YAML bỏ thụt lề | So thụt lề của dòng kết thúc heredoc với các dòng SQL; `kubectl logs db -c <tên init>` |
| B3: `api` `CrashLoopBackOff`, log báo lỗi xác thực mật khẩu | Mật khẩu trong URI khác mật khẩu của role `authenticator`; PostgREST coi lỗi xác thực là lỗi **không thử lại** | So hai chuỗi mật khẩu; `deploy.sh` không tự sửa được, phải sửa manifest rồi tạo lại Pod |
| B3: `api` treo `Init:0/1` | `__DB_HOST__` chưa được thay, hoặc `db` chưa Ready | Triển khai bằng `./scripts/deploy.sh`, không `kubectl apply` file `.tpl` |
| B4: `kill -INT 1` báo không có quyền | Đang exec vào sai container | Thêm `-c postgres` |
| B5: `status.json` không đổi version sau 3 phút | Đổi label của **manifest** chứ không phải Pod đang chạy; hoặc mount bằng `subPath` | `kubectl label pod web ... --overwrite`; không dùng `subPath` cho volume `downwardAPI` |
| B6: không có dòng `PRESTOP` | Hook không ghi vào `/proc/1/fd/1`, hoặc luồng `logs -f` mở sau khi xóa | Giữ đúng lệnh hook; mở luồng log trước `delete` và chờ vài giây |
| B6: không có dòng `SIGQUIT` | `error_log` của nginx không ở mức `notice` vì cấu hình đã bị thay | Không thay `nginx.conf` ở sprint này |
| B9: `web` kẹt `ContainerCreating`, sự kiện nhắc idmap | Kernel hoặc filesystem không hỗ trợ idmap mount | Đối chiếu output tiên quyết; bỏ `hostUsers: false`, ghi lý do vào ADR 0006 |
| B9: `CreateContainerConfigError` với `runAsNonRoot` | Áp `runAsNonRoot` cho container dùng image chạy root (init container `postgres` của `api`, hoặc `db`) | Chỉ áp cho container có image khai `USER` khác 0, như yêu cầu ghi |
| B12: xóa namespace treo `Terminating` | Pod `db` đang trong 60 giây ân hạn | Chờ hết ân hạn; `teardown.sh` chờ tối đa 300 giây |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Workloads](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/)
- [Kubernetes v1.35 — Pods](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/)
- [Kubernetes v1.35 — Pod Lifecycle](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Kubernetes v1.35 — Pod Conditions](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/pod-condition/)
- [Kubernetes v1.35 — Liveness, Readiness and Startup Probes](https://v1-35.docs.kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [Kubernetes v1.35 — Init Containers](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/init-containers/)
- [Kubernetes v1.35 — Sidecar Containers](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)
- [Kubernetes v1.35 — Ephemeral Containers](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/)
- [Kubernetes v1.35 — User Namespaces](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/user-namespaces/)
- [Kubernetes v1.35 — Downward API](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/downward-api/)
- [Kubernetes v1.35 — Advanced Pod Configuration](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/advanced-pod-config/)
- [Kubernetes v1.35 — Attach Handlers to Container Lifecycle Events](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/)
- [Kubernetes v1.35 — Configure Liveness, Readiness and Startup Probes](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Kubernetes v1.35 — Configure Pod Initialization](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-pod-initialization/)
- [Kubernetes v1.35 — Use an Image Volume With a Pod](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/image-volumes/)
- [Kubernetes v1.35 — Share Process Namespace between Containers in a Pod](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/)
- [Kubernetes v1.35 — Use a User Namespace With a Pod](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/user-namespaces/)
- [Kubernetes v1.35 — Expose Pod Information to Containers Through Files](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/)
- [Kubernetes v1.35 — Expose Pod Information to Containers Through Environment Variables](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/)
- [Kubernetes v1.35 — Communicate Between Containers in the Same Pod Using a Shared Volume](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/)
- [Kubernetes v1.35 — Debug Running Pods (kubectl debug)](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [PostgREST v14 — Configuration](https://docs.postgrest.org/en/v14/references/configuration.html)
- [PostgREST v14 — Admin Server](https://docs.postgrest.org/en/v14/references/admin_server.html)
- [PostgREST v14 — Connection Pool, mục Automatic Recovery](https://docs.postgrest.org/en/v14/references/connection_pool.html)
- [Docker Hub — postgres (Environment Variables, Initialization scripts)](https://hub.docker.com/_/postgres)
- [GitHub — nginxinc/docker-nginx-unprivileged](https://github.com/nginxinc/docker-nginx-unprivileged)
