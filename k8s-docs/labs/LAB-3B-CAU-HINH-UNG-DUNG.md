# Lab 3b — Cấu hình ứng dụng: ConfigMap, Secret và dữ liệu cho Pod

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> Repo `~/bookstore` ở tag `lab-3a`.
> **Điểm kết thúc:** teardown trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới. Repo
> có tag `lab-3b` và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 2 — Cấu hình ngoài code của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 3a — Pod và vòng đời](LAB-3A-POD-VA-VONG-DOI.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng mục
[3b. Cấu hình ứng dụng: ConfigMap, Secret và dữ liệu cho Pod](../00-ALO-TRINH-ADMIN.md#3b-cấu-hình-ứng-dụng-configmap-secret-và-dữ-liệu-cho-pod)
của [Giai đoạn 3](../00-ALO-TRINH-ADMIN.md#giai-đoạn-3--pod-và-cấu-hình). Đây là sprint thứ hai
của [dự án Bookstore](DU-AN-BOOKSTORE.md). Trước khi mở B0, đọc lại
[mục 4 — hợp đồng kỹ thuật](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật), hàng "3b" của
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint) và
[kế hoạch sprint 2](DU-AN-BOOKSTORE.md#sprint-2--lab-3b-cấu-hình-ngoài-code).

Sprint 1 để lại ba Pod trần chạy đúng vòng đời, nhưng cấu hình và mật khẩu nằm thẳng trong spec
Pod: schema SQL trong `command` của init container, biến `PGRST_*` trong `env`, mật khẩu superuser
và mật khẩu `authenticator` viết thẳng trong manifest (`TD-02`). Sprint này đưa toàn bộ những thứ
đó ra khỏi spec: cấu hình vào ConfigMap, bí mật vào Secret sinh tại chỗ, không bao giờ vào git.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); image của dự án nằm
ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Lab không chép lại
con số phiên bản nào trong phần văn bản và **không cài thêm gì**.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi hệ thống Bookstore của bạn không còn mật khẩu hay cấu hình nào nằm trong spec
Pod **và** bạn chứng minh, giải thích được:

- Mỗi mẩu dữ liệu của dự án đi đường nào: cấu hình không bí mật vào ConfigMap `web-config`,
  `api-config`, `db-init`; mật khẩu vào Secret `db-credentials` sinh từ `secrets/db.env` — và
  chứng minh được git không chứa mật khẩu nào.
- Vì sao Secret của dự án sinh bằng Kustomize nhưng giữ **tên cố định**, và vì sao
  `scripts/deploy.sh` không gọi thẳng `kubectl apply -k`.
- Ba cách Pod nhận dữ liệu mà dự án dùng — từng key thành biến, cả ConfigMap thành biến, mỗi key
  thành một file — và chọn cách nào cho dữ liệu nào, kể cả `items` và quyền file.
- Ranh giới cập nhật: file trong volume được cập nhật nhưng ứng dụng chưa chắc đọc lại; biến môi
  trường không bao giờ đổi trong đời container; script khởi tạo DB chỉ chạy khi dữ liệu rỗng;
  ConfigMap bất biến; ranh giới namespace; tham chiếu tùy chọn.
- Biến môi trường phụ thuộc: thứ tự trong danh sách quyết định, tham chiếu hỏng không ngăn container
  chạy — và PostgREST hỏng ra sao khi URI ghép sai.
- `command` khác `args` thế nào với entrypoint của image `postgres`; biến khai trong Pod ghi đè biến
  của image.
- Secret chỉ là base64: tự giải ngược mật khẩu của chính dự án, tìm bản sao `tmpfs` trên node, kể
  được ai trong cluster đọc được mật khẩu DB và vì sao nợ lab #6 vẫn mở.
- Nguồn đóng gói nào của `kubectl create configmap` cho ra key nào, và cái bẫy dấu nháy của env-file.
- Secret registry gồm hai lớp base64; `imagePullSecrets` khai ở đâu; event
  `FailedToRetrieveImagePullSecret` nói lên điều gì.
- Sprint đóng lại với `TD-02` đã trả và `TD-07` đã giảm, có bằng chứng.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 3b | Ticket kiểm chứng |
| --- | --- |
| [107 — Cấu hình](../107-configuration-vi.md) | **Không thực hành riêng.** Trang mục lục: một câu giới thiệu và năm trang con. Phạm vi nó khai báo — tài nguyên để cấu hình Pod — được phủ bởi B1–B5 |
| [108 — ConfigMap](../108-configmap-vi.md) | B2 — ConfigMap thành file trong volume; B4 — `items` chọn key thành file; B6 — `data` và `binaryData`, trần 1 MiB ⭐; B7 — ranh giới cập nhật, `immutable`, cùng namespace |
| [109 — Secret](../109-secret-vi.md) | B1 — Secret loại `Opaque`, ràng buộc key; B5 — Secret thành file và thành biến; B8 — base64 không phải mã hóa, `tmpfs`, ai đọc được, nợ #6; B10 — loại `kubernetes.io/dockerconfigjson` |
| [275 — Cấu hình một Pod để sử dụng ConfigMap](../275-configure-pod-configmap-vi.md) | B2 — volume từ ConfigMap; B3 — nạp cả ConfigMap thành biến, bẫy dấu nháy của env-file; B4 — key vào path cụ thể, mount che file của image; B6 — bốn nguồn dữ liệu; B7 — cập nhật tự động, tham chiếu `optional`, các hạn chế |
| [287 — Pull image từ một private registry](../287-pull-image-private-registry-vi.md) | B10 — tạo và mổ xẻ Secret registry, `imagePullSecrets` ở cấp Pod, event khi thiếu Secret. Phần pull thật xem bảng dưới |
| [325 — Quản lý Secret](../325-configmap-secret-vi.md) | **Không thực hành riêng.** Trang mục lục của 326, 327, 328; cả ba được chạy liền nhau ở phần A của B8 |
| [326 — Quản lý Secret bằng file cấu hình](../326-secret-config-file-vi.md) | B8 — `data` base64, `stringData`, ai thắng khi trùng key |
| [327 — Quản lý Secret bằng kubectl](../327-secret-kubectl-vi.md) | B8 — `--from-literal`, `--from-file`, bẫy `echo` thiếu `-n`; `describe` giấu giá trị, `jsonpath` thì không |
| [328 — Quản lý Secret bằng Kustomize](../328-secret-kustomize-vi.md) | B1 — `secretGenerator` sinh Secret của dự án từ file `.env`; B8 — hậu tố hash, sửa dữ liệu sinh object mới |
| [329 — Đưa dữ liệu vào ứng dụng](../329-inject-data-application-vi.md) | **Không thực hành riêng.** Trang mục lục; các trang con thuộc 3b (330–334) ở B3, B5, B9, B11. Hai trang Downward API thuộc [nhóm 3a](../00-ALO-TRINH-ADMIN.md#3a-pod-và-vòng-đời) |
| [330 — Định nghĩa command và argument](../330-define-command-argument-vi.md) | B9 — `args` giữ entrypoint, `command` bỏ qua entrypoint, `$(VAR)` trong `args` |
| [331 — Định nghĩa biến môi trường](../331-define-environment-variable-vi.md) | B3 — `env` khác `envFrom`, key thành tên biến, `$(VAR)` trong cấu hình; B8 — `envFrom` với `prefix`; B9 — biến trong Pod ghi đè biến của image |
| [332 — Biến môi trường bằng init container](../332-define-env-via-file-vi.md) | B11 ⭐ — `fileKeyRef`, cú pháp file env, vì sao không dùng cho mật khẩu |
| [333 — Biến môi trường phụ thuộc](../333-interdependent-env-variables-vi.md) | B3 — thứ tự trong `env`, tham chiếu hỏng, `$$(VAR)`; URI kết nối của `api` |
| [334 — Phân phối thông tin xác thực bằng Secret](../334-distribute-credentials-secure-vi.md) | B5 — volume, `items`, `defaultMode`, `secretKeyRef`; B7 — biến lấy từ Secret không cập nhật |

Các phần dưới đây bị hoãn có chủ đích — nói được vì sao ở checkpoint:

| Phần hoãn | Vì cần | Trả ở |
| --- | --- | --- |
| Mã hóa Secret at rest (bài 109) — **[nợ lab #6](README.md#5-sổ-nợ-lab)** | sửa cấu hình kube-apiserver | [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) |
| Pull thật một image riêng tư (bài 287, mục *Tạo một Pod sử dụng Secret của bạn*) | một private registry và tài khoản thật; dựng registry là hạ tầng ngoài baseline Lab 00 | không kiểm chứng được trong lộ trình; B10 làm phần còn lại: nội dung Secret registry và hành vi của kubelet với `imagePullSecrets` |
| Gắn `imagePullSecrets` vào ServiceAccount (bài 109, 287) | ServiceAccount, giai đoạn 9 | Lab 9a |
| Cách thứ tư dùng ConfigMap — Pod tự gọi Kubernetes API (bài 108) | ServiceAccount và RBAC, giai đoạn 9 | Lab 9a |
| Kustomize đầy đủ: `resources`, `generatorOptions`, `configMapGenerator` (bài 275, 328) | bài [322](../322-kustomization-vi.md), giai đoạn 5 | Lab 5a — sprint này chỉ học `secretGenerator`; phần còn lại của `kustomization.yaml` được đưa sẵn như công cụ |
| `subPath` như một cơ chế volume (bài 108 chỉ nêu nó không nhận cập nhật) | bài [91](../91-volumes-vi.md), giai đoạn 6 | Lab 6a. Ở sprint này, ticket B7 ⭐ của chính Lab 3b chỉ dùng đúng một trường `subPath` để kiểm câu của bài 108 |

Nợ lab của Lab 3a (static Pod, request hiệu dụng của init container và sidecar, phase `Unknown`,
readiness gỡ Pod khỏi EndpointSlice) vẫn mở; sprint này không chạm tới.

### 1.2. Thời lượng

4–6 giờ. Phần cốt lõi (B0–B10, B12) khoảng 4 giờ 30 nếu không bí lâu ở ticket nào; B11 và các mục
đánh dấu ⭐ trong B6, B7 là tùy chọn. Có thể chia hai phiên: phiên 1 làm B0–B5 (hệ thống mới),
phiên 2 làm B6–B12 (điều tra và đóng sprint). Giữa hai phiên cứ để nguyên cluster; đầu phiên 2
chạy lại gate ngắn A5.5 và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Mọi Pod thử nghiệm cố ý hỏng ghim vào node này.
- Repo dự án: `~/bookstore`. Manifest của Pod và object thử nghiệm: `~/lab-work/3b/`. Bằng chứng:
  `~/lab-evidence/3b/`. Object thử nghiệm đặt tên bắt đầu bằng `thu-` và bị xóa khi xong ticket.
- **Vệ sinh bí mật.** Mật khẩu thật của dự án chỉ nằm ở `secrets/db.env` trên master và trong
  Secret `db-credentials`. Script nghiệm thu so sánh mật khẩu **trong biến**, không in ra màn hình,
  không ghi vào `~/lab-evidence/`. Không chạy `set -x` khi làm với Secret. Mọi Secret thử nghiệm
  dùng giá trị giả: `demo-user`, `demo-password`, `registry.bookstore.invalid`.
- Lab chỉ dùng kiến thức nhóm 3b và các nhóm trước. **Không** dùng `requests`/`limits` (3c),
  Deployment (4a), Service (5a), PVC (6a), ServiceAccount và RBAC (9a). Kustomize chỉ dùng phần
  `secretGenerator` của bài 328; các trường khác trong `kustomization.yaml` được đưa sẵn nguyên văn.
- Pod trần gần như bất biến. Đổi spec của `db`, `api` hoặc `web` thì chạy
  `./scripts/teardown.sh && ./scripts/deploy.sh`. Dữ liệu `emptyDir` mất theo — đó là `TD-04`.
- Namespace thử nghiệm duy nhất là `bookstore-thu` (B7), mang label của dự án để teardown dọn
  được nếu bạn quên.
- Lab **chỉ đọc** `/etc/kubernetes/manifests/kube-apiserver.yaml` ở B8. Không sửa file đó, không
  truy cập etcd trực tiếp.
- Không sửa cấu hình node, không cài gói. Lệnh qua `ssh` tới worker chỉ để đọc.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 2: Cấu hình ngoài code

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `01-cluster-ready`, và hệ thống của sprint 1 dựng lại được
**chỉ từ repo** ở tag `lab-3a`.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Repo và công cụ:

```bash
mkdir -p ~/lab-work/3b ~/lab-evidence/3b
test -d ~/bookstore/.git && echo 'PASS: co repo ~/bookstore' \
  || echo 'FAIL: chua co repo — khoi phuc tu bundle (xem ben duoi)'
git -C ~/bookstore describe --tags --exact-match HEAD | grep -qx 'lab-3a' && echo 'PASS: HEAD dung tag lab-3a'
test -z "$(git -C ~/bookstore status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lab truoc' || echo 'PASS: chua co namespace bookstore'
openssl version >/dev/null && echo 'PASS: co openssl'
```

Repo mất vì bạn đã restore snapshot: chép `bookstore-lab-3a.bundle` từ máy host lên master theo
[mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm), rồi
`git clone ~/bookstore-lab-3a.bundle ~/bookstore` và chạy lại khối trên. Vừa restore snapshot thì
chạy lại cả khối kéo image ở [B0 của Lab 3a](LAB-3A-POD-VA-VONG-DOI.md).

Dựng lại sprint 1 từ repo:

```bash
cd ~/bookstore
./scripts/deploy.sh
./tests/smoke.sh > ~/lab-evidence/3b/b0-smoke.txt; echo "smoke exit=$?"
cat ~/lab-evidence/3b/b0-smoke.txt
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; năm dòng `PASS:` của khối repo và công cụ; `deploy.sh`
thoát `0`; `smoke exit=0` và mọi dòng của smoke là `PASS:`.

Smoke fail nghĩa là repo của sprint 1 **chưa đủ** — có thứ chỉ tồn tại trên cluster cũ. Sửa repo,
commit, gắn lại tag `lab-3a` trước khi làm tiếp; không sửa tay trên cluster.

## B1. [BS-3B-01] Bí mật sinh tại chỗ, triển khai bằng Kustomize

**Loại:** Tính năng.

**Bối cảnh:** hệ thống cũ để mật khẩu thẳng trong file compose
([`P3`](DU-AN-BOOKSTORE.md#21-hiện-trạng)); sprint 1 lặp lại đúng lỗi đó (`TD-02`). Bước đầu tiên
là có **chỗ chứa** bí mật nằm ngoài git và một cơ chế đưa nó vào cluster thành Secret. Từ ticket
này, `scripts/deploy.sh` render bằng Kustomize thay cho `kubectl apply -f` từng thư mục.

**Yêu cầu:**

1. Hai mật khẩu — superuser `postgres` và role `authenticator` — được sinh ngẫu nhiên **trên
   master**, lưu ở `secrets/db.env`, chỉ chủ sở hữu đọc ghi được. Sinh **một lần**; các lần triển
   khai sau dùng lại. `scripts/deploy.sh` mới làm việc này — chép nguyên văn ở khối dưới.
2. Git bỏ qua mọi thứ trong `secrets/` trừ `secrets/.gitkeep`.
3. Secret `db-credentials` trong namespace `bookstore`: loại mặc định, đúng hai key
   `postgres-password` và `authenticator-password`
   ([mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label)), label của dự án, **tên cố định** không hậu tố.
   Secret được Kustomize sinh từ `secrets/db.env` — không tạo bằng lệnh gõ tay.
4. `k8s/app/kustomization.yaml` gom namespace và hai Pod `db`, `web`. Phần gom tài nguyên và khối
   tùy chọn của bộ sinh chép nguyên văn; tên và nguồn dữ liệu của bộ sinh tự viết.
5. `scripts/teardown.sh` mới — chép nguyên văn.
6. Ticket này **chưa** nối Secret vào Pod nào. Hệ thống vẫn chạy như sprint 1 và `tests/smoke.sh`
   của Lab 3a vẫn PASS.

**Ràng buộc:** nguồn bí mật duy nhất là `secrets/db.env` (hàng "3b" của
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint)); không chép file đó sang chỗ khác trong
repo. Pod `api` vẫn là file `.tpl` nằm **ngoài** Kustomize (`TD-03`).

**Đọc lại:**

- Bài [328](../328-secret-kustomize-vi.md): mục *Tạo file kustomization* — ba dạng nguồn, đặc biệt
  *File .env*; mục *Apply file kustomization* — tên mang hậu tố hash.
- Bài [109](../109-secret-vi.md): [*Các loại Secret*](../109-secret-vi.md#secret-types) — `Opaque`;
  [*Ràng buộc về tên và dữ liệu của Secret*](../109-secret-vi.md#restriction-names-data) — ký tự
  hợp lệ của key.
- Bài [325](../325-configmap-secret-vi.md) — đặt đường Kustomize cạnh hai đường kia trước khi chọn.
- [Mục 5.3 và 5.4 của file dự án](DU-AN-BOOKSTORE.md#53-quy-tắc-làm-việc-với-repo) — quy tắc 3, và
  vì sao bundle không chứa `secrets/`.

**Bàn giao:** `.gitignore`, `k8s/app/kustomization.yaml`, `scripts/deploy.sh`,
`scripts/teardown.sh`; dòng mô tả `secrets/` trong `README.md`;
`docs/adr/0007-bi-mat-sinh-tai-cho.md`; `~/lab-evidence/3b/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` git bỏ qua `secrets/db.env`; trong `secrets/` git chỉ theo dõi `secrets/.gitkeep`.
- `PASS:` `secrets/db.env` có quyền `600`, đúng hai dòng `postgres-password=` và
  `authenticator-password=`, mỗi giá trị 48 ký tự hex.
- `PASS:` render mặc định của Kustomize với `k8s/app` **thất bại** — lý do `deploy.sh` render khác.
- `PASS:` Secret `db-credentials` loại `Opaque`, mang label dự án, tên không hậu tố, đúng hai key.
- `PASS:` giá trị giải mã từ Secret khớp `secrets/db.env`.
- `PASS:` chạy `deploy.sh` lần thứ hai không lỗi, không sinh bí mật mới, Secret giữ nguyên
  `resourceVersion`.
- `PASS:` `tests/smoke.sh` của Lab 3a vẫn PASS.

Hai script, chép nguyên văn:

```bash
cd ~/bookstore

cat > scripts/deploy.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai dự án Bookstore từ repo — phiên bản Lab 3b.
# - Bí mật sinh tại chỗ vào secrets/db.env nếu chưa có; file này không bao giờ vào git.
# - Kustomize gom namespace, ConfigMap, Pod db, web và sinh Secret db-credentials (k8s/app).
#   Secret sinh từ ../../secrets/db.env, nằm ngoài thư mục gốc của kustomization, nên phải render
#   bằng "kubectl kustomize --load-restrictor=LoadRestrictionsNone" rồi mới apply.
# - api vẫn là file .tpl, chèn IP của Pod db lúc triển khai (TD-03).
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore

mkdir -p secrets
if [ ! -s secrets/db.env ]; then
  PG_PW="$(openssl rand -hex 24)"
  AU_PW="$(openssl rand -hex 24)"
  ( umask 077
    printf 'postgres-password=%s\nauthenticator-password=%s\n' "$PG_PW" "$AU_PW" > secrets/db.env )
  unset PG_PW AU_PW
  echo 'deploy: da sinh secrets/db.env moi'
fi

if compgen -G 'k8s/cluster/*.yaml' >/dev/null; then kubectl apply -f k8s/cluster/; fi

kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | kubectl apply -f -
kubectl -n "$NS" wait --for=condition=Ready pod/db --timeout=300s

DB_IP="$(kubectl -n "$NS" get pod db -o jsonpath='{.status.podIP}')"
sed "s/__DB_HOST__/${DB_IP}/g" k8s/app/api/pod.yaml.tpl | kubectl apply -f -

for P in db api web; do
  kubectl -n "$NS" wait --for=condition=Ready "pod/$P" --timeout=300s
done
kubectl -n "$NS" get pods -o wide
EOF

cat > scripts/teardown.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ toàn bộ dự án Bookstore khỏi cluster — phiên bản Lab 3b.
# Giữ nguyên secrets/db.env trên máy: lần deploy sau dùng lại đúng bí mật cũ.
set -euo pipefail
SEL='app.kubernetes.io/part-of=bookstore'
kubectl delete namespace -l "$SEL" --ignore-not-found --wait=true --timeout=300s
kubectl delete pods,configmaps,secrets --all-namespaces -l "$SEL" --ignore-not-found --wait=true --timeout=120s
kubectl delete priorityclass -l "$SEL" --ignore-not-found
LEFT="$( { kubectl get namespace,priorityclass -l "$SEL" --no-headers
           kubectl get pods,configmaps,secrets --all-namespaces -l "$SEL" --no-headers; } 2>/dev/null | wc -l)"
if [ "$LEFT" -eq 0 ]; then echo 'teardown: sach'; else echo "teardown: con $LEFT object"; exit 1; fi
EOF

chmod +x scripts/*.sh
```

Khung `k8s/app/kustomization.yaml` — chép nguyên văn rồi điền hai chỗ đánh dấu `<tự viết>`:

```yaml
# k8s/app/kustomization.yaml — Lab 3b.
# "resources" và khối "options" là công cụ, chép nguyên văn; Kustomize đầy đủ học ở Lab 5a
# (bài 322). Tên và nguồn dữ liệu của secretGenerator là nội dung ticket BS-3B-01 (bài 328).
# Pod api KHÔNG nằm trong danh sách: nó vẫn là file .tpl do scripts/deploy.sh render (TD-03).
# Mỗi ticket sau thêm ConfigMap của mình vào "resources".
resources:
- 00-namespace.yaml
- db/pod.yaml
- web/pod.yaml

secretGenerator:
- name: <tự viết>           # tên theo hợp đồng
  namespace: bookstore
  # <tự viết>: nguồn dữ liệu là file secrets/db.env — chọn đúng dạng nguồn trong bài 328
  options:
    disableNameSuffixHash: true   # giữ tên cố định — vì sao: câu hỏi thiết kế 1
    labels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: db
```

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0007-bi-mat-sinh-tai-cho.md`:

1. Bài 328 coi hậu tố hash là ưu điểm: dữ liệu đổi là có Secret mới. Vì sao Secret của dự án
   **phải** tắt hậu tố ở sprint này? Mất gì? (Gợi ý: ai tham chiếu Secret theo tên mà Kustomize
   không nhìn thấy?)
2. Vì sao mật khẩu sinh bằng hex chứ không bằng base64? Nghĩ tới chỗ mật khẩu `authenticator` sẽ
   nằm ở B3–B5.
3. Restore snapshot rồi clone lại từ bundle: `secrets/db.env` không còn. Hệ thống có dựng lại được
   không, vì sao? Điều gì sẽ khác khi dữ liệu DB bắt đầu bền ở Lab 6a?
4. Vì sao `deploy.sh` không gọi thẳng `kubectl apply -k k8s/app` như bài 328? Đánh đổi của cách
   đang dùng là gì?

<details>
<summary>Gợi ý</summary>

1. Bài 328 có ba dạng nguồn: `literals`, `files`, `envs`. File `secrets/db.env` gồm các dòng
   `key=value` — chọn dạng nguồn để **mỗi dòng thành một key**.
2. Đường dẫn trong `kustomization.yaml` tính từ thư mục chứa file đó: từ `k8s/app` tới
   `secrets/db.env` là `../../secrets/db.env`.
3. Quy tắc `.gitignore` có thể phủ định bằng `!`: một dòng chặn cả thư mục, một dòng giữ lại
   `.gitkeep`. Kiểm bằng `git check-ignore -v secrets/db.env`.
4. Xem Kustomize sẽ gửi gì lên cluster mà không gửi:
   `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | less`.
5. Đổi cơ chế triển khai giữa chừng: `./scripts/teardown.sh && ./scripts/deploy.sh`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3b
cd ~/bookstore
git check-ignore -q secrets/db.env && test "$(git ls-files secrets)" = 'secrets/.gitkeep' \
  && echo 'PASS: git bo qua secrets/db.env, chi theo doi secrets/.gitkeep'
{ test "$(stat -c %a secrets/db.env)" = '600' && \
  test "$(grep -c '' secrets/db.env)" -eq 2 && \
  grep -Eq '^postgres-password=[0-9a-f]{48}$' secrets/db.env && \
  grep -Eq '^authenticator-password=[0-9a-f]{48}$' secrets/db.env; } \
  && echo 'PASS: secrets/db.env quyen 600, hai khoa, gia tri hex 48 ky tu'

kubectl kustomize k8s/app > "$EV/b1-kustomize-mac-dinh.txt" 2>&1 \
  && echo 'FAIL: render mac dinh doc duoc file ngoai k8s/app' \
  || echo 'PASS: mac dinh Kustomize khong doc file ngoai thu muc goc'

jp() { kubectl -n "$NS" get secret db-credentials -o jsonpath="$1"; }
{ test "$(jp '{.type}')" = 'Opaque' && \
  test "$(jp '{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' && \
  test "$(kubectl -n "$NS" get secret -o name | grep -c '^secret/db-credentials')" -eq 1 && \
  test "$(kubectl -n "$NS" get secret db-credentials -o go-template='{{range $k, $v := .data}}{{$k}} {{end}}')" \
       = 'authenticator-password postgres-password '; } \
  && echo 'PASS: db-credentials Opaque, mang label, ten co dinh, dung hai key'

OK=1
for K in postgres-password authenticator-password; do
  F="$(sed -n "s/^${K}=//p" secrets/db.env)"
  S="$(kubectl -n "$NS" get secret db-credentials -o go-template="{{index .data \"$K\"}}" | base64 -d)"
  { test -n "$F" && test "$F" = "$S"; } || OK=0
done
test "$OK" -eq 1 && echo 'PASS: gia tri trong Secret khop secrets/db.env'

H0="$(sha256sum secrets/db.env)"; RV0="$(jp '{.metadata.resourceVersion}')"
./scripts/deploy.sh > "$EV/b1-deploy-lan-2.txt" 2>&1 && echo 'PASS: deploy.sh chay lan hai khong loi'
{ test "$(sha256sum secrets/db.env)" = "$H0" && test "$(jp '{.metadata.resourceVersion}')" = "$RV0"; } \
  && echo 'PASS: chay lai khong sinh bi mat moi, Secret khong doi'
./tests/smoke.sh
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
cd ~/bookstore
printf 'secrets/*\n!secrets/.gitkeep\n' > .gitignore
cat >> README.md <<'EOF'
- Bí mật:     secrets/db.env do scripts/deploy.sh sinh tại chỗ; KHÔNG commit, không có trong bundle.
EOF
```

Phần tự viết của `k8s/app/kustomization.yaml`:

```yaml
secretGenerator:
- name: db-credentials
  namespace: bookstore
  envs:
  - ../../secrets/db.env
  options:
    disableNameSuffixHash: true
    labels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: db
```

```bash
./scripts/teardown.sh && ./scripts/deploy.sh
kubectl -n bookstore get secret db-credentials
```

Ý chính cho ADR 0007:

1. `api` là file `.tpl` nằm ngoài kustomization và tham chiếu Secret theo tên. Kustomize chỉ tự
   thay tên đã sinh trong những resource nó quản lý, nên với hậu tố hash `api` sẽ trỏ vào một tên
   không tồn tại. Mất: đổi bí mật không còn sinh object mới, tức không có tín hiệu ở cấp tên — nhưng
   với Pod trần thì tín hiệu đó cũng chưa có ai dùng. Lab 5a, khi mọi thứ vào kustomization, đổi lại.
2. Mật khẩu `authenticator` sẽ nằm trong URI `postgres://user:pass@host:5432/db`. Base64 chứa `/`,
   `+`, `=` — các ký tự có nghĩa trong URI. Hex chỉ có `0-9a-f`.
3. Dựng lại được: `deploy.sh` sinh bí mật mới; DB nằm trên `emptyDir` nên khởi tạo lại từ đầu với
   mật khẩu mới. Từ Lab 6a dữ liệu sống trên volume bền: script khởi tạo không chạy lại, DB giữ mật
   khẩu cũ, `api` sẽ không đăng nhập được với bí mật mới. Lúc đó cần sao lưu `secrets/` ra ngoài
   bằng một kênh an toàn riêng, hoặc có quy trình đổi mật khẩu trong DB.
4. Mặc định Kustomize chỉ cho kustomization đọc file nằm trong hoặc dưới thư mục gốc của nó;
   `kubectl apply -k` không có cờ nới lỏng. Đánh đổi: kustomization không còn tự đứng được — gõ tay
   `kubectl apply -k k8s/app` sẽ lỗi, mọi triển khai phải đi qua `deploy.sh`.

</details>

## B2. [BS-3B-02] Schema của `db` từ ConfigMap `db-init`

**Loại:** Tính năng.

**Bối cảnh:** schema SQL đang nằm trong `command` của init container `render-schema` — sửa một
dòng SQL là sửa spec Pod. Hệ thống cũ mount thư mục `./sql` vào `/docker-entrypoint-initdb.d`
([hiện trạng](DU-AN-BOOKSTORE.md#21-hiện-trạng)). ConfigMap là chỗ đúng cho dữ liệu đó.

**Yêu cầu:**

1. ConfigMap `db-init` trong `bookstore`, label theo hợp đồng (thành phần `db`), chứa script khởi
   tạo của sprint 1 dưới một key có đuôi `.sql`.
2. Container `postgres` thấy script đó trong `/docker-entrypoint-initdb.d`, thư mục chỉ đọc. Pod
   `db` không còn init container sinh schema, không còn volume `emptyDir` cho script.
3. Sau khởi tạo, dữ liệu đúng [mục 4.3](DU-AN-BOOKSTORE.md#43-dữ-liệu): 5 cuốn sách, hai role.
4. Manifest ở `k8s/app/db/configmap.yaml` và được Kustomize gom.
5. Mật khẩu của role `authenticator` **tạm thời** vẫn nằm trong SQL, ghi comment `TD-02` ngay cạnh;
   B5 trả.

**Ràng buộc:** không dùng `subPath`, không đổi image. Đổi spec Pod trần thì teardown rồi deploy.

**Đọc lại:**

- Bài [108](../108-configmap-vi.md): [*Đối tượng ConfigMap*](../108-configmap-vi.md#configmap-object);
  mục *ConfigMap và Pod* — Pod và ConfigMap phải cùng namespace; mục *Dùng ConfigMap làm file
  trong một Pod* — bốn bước.
- Bài [275](../275-configure-pod-configmap-vi.md):
  [*Tạo ConfigMap từ một thư mục*](../275-configure-pod-configmap-vi.md#create-configmaps-from-directories);
  mục *Nạp dữ liệu lưu trong ConfigMap vào một Volume* — kể cả ghi chú về file có sẵn của image;
  mục *Các hạn chế*.
- [Mục 4.3 của file dự án](DU-AN-BOOKSTORE.md#43-dữ-liệu) — script khởi tạo chỉ chạy khi thư mục
  dữ liệu rỗng.

**Bàn giao:** `k8s/app/db/configmap.yaml`; `k8s/app/db/pod.yaml` đã cập nhật; một dòng mới trong
`resources` của kustomization; `docs/adr/0008-db-init-tu-configmap.md`; `~/lab-evidence/3b/b2-*`.

**Nghiệm thu:**

- `PASS:` Pod `db` Ready và không còn init container.
- `PASS:` `/docker-entrypoint-initdb.d` của container `postgres` là volume lấy từ ConfigMap
  `db-init`, và không ghi được.
- `PASS:` log của `postgres` cho thấy entrypoint đã chạy script `.sql` trong thư mục đó.
- `PASS:` `api.books` có ít nhất 5 dòng; đủ hai role `authenticator`, `web_anon`.
- `PASS:` `db-init` mang label của dự án; `tests/smoke.sh` của Lab 3a vẫn PASS.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0008-db-init-tu-configmap.md`:

1. Sửa `01-schema.sql` trong ConfigMap khi `db` đang chạy: file trong container có đổi không? Dữ
   liệu trong DB có đổi không? Vì sao hai câu trả lời khác nhau?
2. Quên thêm ConfigMap vào kustomization: Pod `db` hỏng thế nào, đọc lý do ở đâu? So với init
   container của sprint 1, ConfigMap đem lại phụ thuộc mới nào?
3. Bài 108 nói ConfigMap không giữ bí mật. Script SQL lúc này có vi phạm điều đó không?

<details>
<summary>Gợi ý</summary>

1. Hai cách viết ConfigMap: viết tay YAML với khối `|` như ví dụ `game-demo` của bài 108; hoặc giữ
   SQL thành file trong một thư mục ở `~/lab-work/3b/`, rồi
   `kubectl create configmap db-init --from-file=<thư mục> --dry-run=client -o yaml` và tự thêm
   `namespace`, `labels`.
2. Volume kiểu `configMap` khai ở `spec.volumes`, mount ở `volumeMounts` của container.
3. Entrypoint của image in `running /docker-entrypoint-initdb.d/<file>` cho từng script nó chạy.
4. Kiểm thư mục chỉ đọc: `kubectl -n bookstore exec db -c postgres -- touch /docker-entrypoint-initdb.d/x`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3b
cd ~/bookstore && ./scripts/teardown.sh && ./scripts/deploy.sh > "$EV/b2-deploy.txt" 2>&1
jp() { kubectl -n "$NS" get pod db -o jsonpath="$1"; }

kubectl -n "$NS" wait --for=condition=Ready pod/db --timeout=300s \
  && test -z "$(jp '{.spec.initContainers}')" && echo 'PASS: db Ready, khong con init container'
V="$(jp '{.spec.volumes[?(@.configMap.name=="db-init")].name}')"
{ test -n "$V" && \
  test "$(jp "{.spec.containers[?(@.name==\"postgres\")].volumeMounts[?(@.name==\"$V\")].mountPath}")" \
       = '/docker-entrypoint-initdb.d'; } && echo 'PASS: /docker-entrypoint-initdb.d lay tu ConfigMap db-init'
kubectl -n "$NS" exec db -c postgres -- touch /docker-entrypoint-initdb.d/thu-ghi 2>/dev/null \
  && echo 'FAIL: ghi duoc vao thu muc khoi tao' || echo 'PASS: thu muc khoi tao chi doc'

kubectl -n "$NS" logs db -c postgres > "$EV/b2-db.log"
grep -q 'running /docker-entrypoint-initdb.d/.*\.sql' "$EV/b2-db.log" \
  && echo 'PASS: entrypoint chay script SQL tu ConfigMap'

SQL() { kubectl -n "$NS" exec db -c postgres -- psql -U postgres -d bookstore -tAc "$1"; }
N="$(SQL 'SELECT count(*) FROM api.books')"
R="$(SQL "SELECT string_agg(rolname, ',' ORDER BY rolname) FROM pg_roles WHERE rolname IN ('authenticator','web_anon')")"
{ test "${N:-0}" -ge 5 && test "$R" = 'authenticator,web_anon'; } && echo "PASS: $N cuon sach, du hai role"

test "$(kubectl -n "$NS" get configmap db-init -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}')" \
  = 'bookstore' && echo 'PASS: db-init mang label du an'
./tests/smoke.sh
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/db/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-init
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: db
    app.kubernetes.io/name: postgresql
data:
  01-schema.sql: |
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
    -- TD-02: mật khẩu tạm trong ConfigMap, B5 chuyển sang Secret
    CREATE ROLE authenticator NOINHERIT LOGIN PASSWORD 'lab-3a-authenticator';
    GRANT web_anon TO authenticator;
```

Trong `k8s/app/db/pod.yaml`: xóa hẳn khối `initContainers` (`render-schema`); volume `initdb`
đổi từ `emptyDir` sang ConfigMap. Mọi thứ khác giữ như cuối Lab 3a; manifest đầy đủ của `db` nằm ở
lời giải B5.

```yaml
  volumes:
  - name: initdb
    configMap:
      name: db-init
  - name: data
    emptyDir: {}
  # ... container postgres giữ nguyên, kể cả mount:
    volumeMounts:
    - name: initdb
      mountPath: /docker-entrypoint-initdb.d
      readOnly: true
    - name: data
      mountPath: /var/lib/postgresql/data
```

Thêm `- db/configmap.yaml` vào `resources` của `k8s/app/kustomization.yaml`, rồi
`./scripts/teardown.sh && ./scripts/deploy.sh`.

Ý chính cho ADR 0008:

1. File trong container **có** đổi: kubelet chiếu lại ConfigMap đã mount theo chu kỳ đồng bộ (độ
   trễ phụ thuộc cấu hình, bài 108). DB **không** đổi: entrypoint chỉ chạy script khi thư mục dữ
   liệu rỗng ([mục 4.3](DU-AN-BOOKSTORE.md#43-dữ-liệu)). Muốn áp schema mới lúc này phải tạo lại Pod
   (`emptyDir` mới); khi dữ liệu bền, đó là việc của migration (Lab 4b).
2. Pod được tạo nhưng nằm `ContainerCreating`: kubelet không mount được volume, event `FailedMount`
   báo `configmap "db-init" not found` — đúng mục *Các hạn chế* của bài 275. Phụ thuộc mới: một
   object khác phải tồn tại **trước** Pod, trong cùng namespace.
3. Có — một mật khẩu đang nằm trong object không được thiết kế để giữ bí mật, và trong git. `TD-02`
   vẫn mở; B5 trả.

</details>

## B3. [BS-3B-03] `api`: cấu hình từ ConfigMap, URI ghép từ biến

**Loại:** Tính năng.

**Bối cảnh:** bốn biến `PGRST_*` không bí mật đang nằm thẳng trong spec, còn URI kết nối gộp cả
địa chỉ DB lẫn mật khẩu thành một chuỗi viết tay. Tách phần không bí mật ra ConfigMap, và ghép URI
từ các mảnh riêng — để B5 chỉ cần đổi **nguồn** của mảnh mật khẩu, không đụng tới phần còn lại.

**Yêu cầu:**

1. Thí nghiệm trước khi sửa `api`: Pod `thu-bien-phu-thuoc` ở `~/lab-work/3b/`, ghim
   `lab-k8s-worker2`, `busybox`, chạy một lần rồi kết thúc, in ra ba biến. Danh sách biến theo
   **đúng thứ tự** sau:

   | Thứ tự | Biến | Giá trị khai báo |
   | --- | --- | --- |
   | 1 | `DB_HOST` | `10.0.0.1` |
   | 2 | `URI_SAI_THU_TU` | `postgres://authenticator:$(DB_PASSWORD)@$(DB_HOST):5432/bookstore` |
   | 3 | `DB_PASSWORD` | `demo-password` |
   | 4 | `URI_DUNG` | `postgres://authenticator:$(DB_PASSWORD)@$(DB_HOST):5432/bookstore` |
   | 5 | `URI_THOAT` | `postgres://authenticator:$$(DB_PASSWORD)@$(DB_HOST):5432/bookstore` |

   Ghi lại ba giá trị thật mà container thấy, rồi xóa Pod.
2. ConfigMap `api-config` (thành phần `api`) chứa đúng bốn biến không bí mật của PostgREST: schema
   công khai, role ẩn danh, cổng chính `3000`, cổng admin `3001`. Không chứa địa chỉ DB, không chứa
   mật khẩu. Giá trị không kèm dấu nháy.
3. Container `postgrest` nạp **toàn bộ** `api-config` thành biến môi trường mà không liệt kê từng
   key.
4. URI kết nối được ghép từ biến riêng cho địa chỉ DB (vẫn chèn lúc triển khai) và biến riêng cho
   mật khẩu. Mật khẩu **tạm thời** vẫn viết thẳng, ghi comment `TD-02`; B5 trả. Chuỗi URI khai báo
   trong manifest không chứa IP hay mật khẩu.
5. `api` Ready, `GET /books` trả ít nhất 5 cuốn sách.
6. ConfigMap ở `k8s/app/api/configmap.yaml`, nằm trong kustomization. Pod `api` vẫn là
   `k8s/app/api/pod.yaml.tpl` ngoài kustomization.

**Ràng buộc:** image `postgrest` không có shell — URI phải được ghép **trước** khi tiến trình
chạy, không phải bằng shell trong container.

**Đọc lại:**

- Bài [331](../331-define-environment-variable-vi.md): mục *Định nghĩa một biến môi trường cho một
  container* — `env` khác `envFrom`, và hai ghi chú cuối mục;
  [*Sử dụng biến môi trường bên trong cấu hình*](../331-define-environment-variable-vi.md#using-environment-variables-inside-of-your-config).
- Bài [333](../333-interdependent-env-variables-vi.md) — cả bài: thứ tự, tham chiếu không phân giải
  được, `$$(VAR)`.
- Bài [275](../275-configure-pod-configmap-vi.md):
  [*Cấu hình tất cả các cặp key-value trong ConfigMap thành biến môi trường*](../275-configure-pod-configmap-vi.md#configure-all-key-value-pairs-in-a-configmap-as-container-environment-variables);
  [*Tạo ConfigMap từ file*](../275-configure-pod-configmap-vi.md#create-configmaps-from-files) —
  đoạn `--from-env-file` và dòng về dấu nháy.
- PostgREST v14 [Configuration](https://docs.postgrest.org/en/v14/references/configuration.html),
  mục *Environment Variables*.

**Bàn giao:** `k8s/app/api/configmap.yaml`; `k8s/app/api/pod.yaml.tpl` đã cập nhật;
`docs/adr/0009-cau-hinh-api.md`; `~/lab-evidence/3b/b3-*`.

**Nghiệm thu:**

- `PASS:` Pod thí nghiệm `Succeeded`: `URI_DUNG` phân giải đủ; `URI_SAI_THU_TU` và `URI_THOAT` giữ
  nguyên chuỗi `$(DB_PASSWORD)`; Pod đã bị xóa.
- `PASS:` `api` Ready; `/books` trả ít nhất 5 cuốn sách.
- `PASS:` `api-config` có đúng bốn key `PGRST_*`; schema là `api`, không dấu nháy.
- `PASS:` container `postgrest` nạp `api-config` bằng cách nạp cả ConfigMap.
- `PASS:` chuỗi URI khai báo tham chiếu biến, không chứa IP hiện tại của `db`.
- `PASS:` `tests/smoke.sh` của Lab 3a vẫn PASS.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0009-cau-hinh-api.md`:

1. Đưa URI lên **trên** biến mật khẩu trong danh sách: Pod có được tạo không, container có chạy
   không, PostgREST làm gì, bạn thấy dấu vết ở đâu? Dùng kết quả thí nghiệm để trả lời.
2. Ở hai cách nạp — từng key và cả ConfigMap — tên biến môi trường do ai quyết định? Vì sao
   `api-config` hợp với cách nạp cả map, còn mật khẩu ở B5 lấy từng key?
3. Tạo `api-config` bằng file `.env` của hệ thống cũ có dòng `PGRST_DB_SCHEMAS="api"` thì `api`
   hỏng ra sao? (B6 kiểm chứng.)
4. Sửa `api-config` khi `api` đang chạy thì PostgREST có thấy giá trị mới không? (B7 kiểm chứng.)

<details>
<summary>Gợi ý</summary>

1. Pod thí nghiệm: `command: ["sh", "-c"]`, `args` là một lệnh `printf` in ba dòng `TEN=giá-trị`
   bằng biến shell `$TEN`. Kubernetes chỉ đụng tới dạng `$(TEN)`, không đụng `$TEN`. Viết file bằng
   heredoc có nháy (`<<'EOF'`) để shell trên master không ăn mất `$`.
2. Trong YAML, `$$(DB_PASSWORD)` viết nguyên như vậy.
3. Tạo ConfigMap từ file kiểu `.env`: `kubectl create configmap api-config --from-env-file=<file>
   --dry-run=client -o yaml`, rồi thêm `namespace` và `labels`. Viết tay YAML cũng được.
4. Biến khai bằng `env` có thể tham chiếu biến đứng **trước** nó bằng `$(TEN)`.
5. Script `deploy.sh` thay mọi chuỗi `__DB_HOST__` trong file `.tpl`, kể cả trong `value` của một
   biến riêng.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Thí nghiệm, sau khi đã tự viết `~/lab-work/3b/thu-bien-phu-thuoc.yaml`:

```bash
NS=bookstore; EV=~/lab-evidence/3b
kubectl apply -f ~/lab-work/3b/thu-bien-phu-thuoc.yaml
kubectl -n "$NS" wait --for=jsonpath='{.status.phase}'=Succeeded pod/thu-bien-phu-thuoc --timeout=180s \
  && echo 'PASS: tham chieu hong khong ngan container chay'
kubectl -n "$NS" logs thu-bien-phu-thuoc > "$EV/b3-bien-phu-thuoc.txt"
cat "$EV/b3-bien-phu-thuoc.txt"
grep -qxF 'URI_DUNG=postgres://authenticator:demo-password@10.0.0.1:5432/bookstore' "$EV/b3-bien-phu-thuoc.txt" \
  && echo 'PASS: bien dung sau cac bien no tham chieu thi duoc phan giai'
grep -qxF 'URI_SAI_THU_TU=postgres://authenticator:$(DB_PASSWORD)@10.0.0.1:5432/bookstore' "$EV/b3-bien-phu-thuoc.txt" \
  && echo 'PASS: tham chieu toi bien khai sau giu nguyen chuoi'
grep -qxF 'URI_THOAT=postgres://authenticator:$(DB_PASSWORD)@10.0.0.1:5432/bookstore' "$EV/b3-bien-phu-thuoc.txt" \
  && echo 'PASS: $$(VAR) khong bao gio duoc khai trien'
kubectl -n "$NS" delete pod thu-bien-phu-thuoc --wait=true && echo 'PASS: da xoa Pod thi nghiem'
```

Hệ thống:

```bash
NS=bookstore; EV=~/lab-evidence/3b
cd ~/bookstore && ./scripts/teardown.sh && ./scripts/deploy.sh > "$EV/b3-deploy.txt" 2>&1
C='{.spec.containers[?(@.name=="postgrest")]'
jp() { kubectl -n "$NS" get pod api -o jsonpath="$1"; }

kubectl -n "$NS" wait --for=condition=Ready pod/api --timeout=300s && echo 'PASS: api Ready'
API_IP="$(jp '{.status.podIP}')"
kubectl -n "$NS" run curl-tam --rm -i --restart=Never --image=busybox:1.37 \
  -- wget -qO- "http://${API_IP}:3000/books" > "$EV/b3-books.json" 2>/dev/null
N="$(grep -o '"title"' "$EV/b3-books.json" | wc -l)"
test "$N" -ge 5 && echo "PASS: /books tra $N cuon sach"

{ test "$(kubectl -n "$NS" get configmap api-config -o go-template='{{range $k, $v := .data}}{{$k}} {{end}}')" \
       = 'PGRST_ADMIN_SERVER_PORT PGRST_DB_ANON_ROLE PGRST_DB_SCHEMAS PGRST_SERVER_PORT ' && \
  test "$(kubectl -n "$NS" get configmap api-config -o jsonpath='{.data.PGRST_DB_SCHEMAS}')" = 'api'; } \
  && echo 'PASS: api-config dung bon key PGRST_*, schema la api khong dau nhay'
jp "$C.envFrom[*].configMapRef.name}" | grep -qw 'api-config' && echo 'PASS: postgrest nap ca api-config'

URI="$(jp "$C.env[?(@.name==\"PGRST_DB_URI\")].value}")"
DB_IP="$(kubectl -n "$NS" get pod db -o jsonpath='{.status.podIP}')"
{ echo "$URI" | grep -q '\$(' && ! echo "$URI" | grep -qF "$DB_IP"; } \
  && echo 'PASS: URI ghep tu bien, khong viet thang dia chi DB'
./tests/smoke.sh
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/3b/thu-bien-phu-thuoc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: thu-bien-phu-thuoc
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: in-bien
    image: busybox:1.37
    command: ["sh", "-c"]
    args:
    - printf 'URI_SAI_THU_TU=%s\nURI_DUNG=%s\nURI_THOAT=%s\n' "$URI_SAI_THU_TU" "$URI_DUNG" "$URI_THOAT"
    env:
    - name: DB_HOST
      value: "10.0.0.1"
    - name: URI_SAI_THU_TU
      value: "postgres://authenticator:$(DB_PASSWORD)@$(DB_HOST):5432/bookstore"
    - name: DB_PASSWORD
      value: "demo-password"
    - name: URI_DUNG
      value: "postgres://authenticator:$(DB_PASSWORD)@$(DB_HOST):5432/bookstore"
    - name: URI_THOAT
      value: "postgres://authenticator:$$(DB_PASSWORD)@$(DB_HOST):5432/bookstore"
```

```yaml
# k8s/app/api/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: api
    app.kubernetes.io/name: postgrest
data:
  PGRST_DB_SCHEMAS: "api"
  PGRST_DB_ANON_ROLE: "web_anon"
  PGRST_SERVER_PORT: "3000"
  PGRST_ADMIN_SERVER_PORT: "3001"
```

Trong `k8s/app/api/pod.yaml.tpl`, khối `env` cũ của container `postgrest` (năm biến `PGRST_*`)
thay bằng khối dưới. Metadata, init container `wait-for-db`, cổng, ba probe, `securityContext` giữ
nguyên như cuối Lab 3a.

```yaml
  containers:
  - name: postgrest
    image: postgrest/postgrest:v14.18
    envFrom:
    - configMapRef:
        name: api-config
    env:
    - name: DB_HOST
      value: "__DB_HOST__"              # deploy.sh thay bằng IP Pod db (TD-03)
    - name: DB_PASSWORD
      value: "lab-3a-authenticator"     # TD-02: B5 chuyển sang Secret
    - name: PGRST_DB_URI                # phải đứng SAU hai biến nó tham chiếu
      value: "postgres://authenticator:$(DB_PASSWORD)@$(DB_HOST):5432/bookstore"
    # ports, startupProbe, readinessProbe, livenessProbe, securityContext: giữ nguyên
```

Thêm `- api/configmap.yaml` vào `resources`, rồi `./scripts/teardown.sh && ./scripts/deploy.sh`.

Ý chính cho ADR 0009:

1. Pod vẫn được tạo và container vẫn chạy — bài 333: biến phân tích sai không ngăn container khởi
   động, tham chiếu tới biến khai sau được giữ nguyên như chuỗi thường. PostgREST nhận mật khẩu là
   chuỗi `$(DB_PASSWORD)`, bị PostgreSQL từ chối; lỗi xác thực là lỗi PostgREST **không thử lại**,
   tiến trình thoát, Pod vào `CrashLoopBackOff`. Dấu vết: `kubectl logs api -c postgrest --previous`
   và chuỗi `$(` còn nguyên trong `kubectl get pod api -o yaml`.
2. Nạp từng key: **bạn** đặt tên biến. Nạp cả ConfigMap: **key** là tên biến (bài 331). Bốn key
   của `api-config` đã mang đúng tên PostgREST cần, nên nạp cả map. Mật khẩu chỉ là một mảnh của
   URI, cần một tên biến do bạn đặt để ghép, và không nên kéo theo key nào khác của Secret.
3. Bài 275: env-file không xử lý dấu nháy, giá trị thành `"api"` kèm nháy. PostgREST tìm schema
   tên `"api"` (có nháy), không có, nên không phục vụ được `/books`.
4. Không — biến môi trường được đặt một lần khi container khởi động (bài 108, mục *ConfigMap đã
   mount được cập nhật tự động*, câu cuối). Với Pod trần phải tạo lại Pod.

</details>

## B4. [BS-3B-04] `web`: cấu hình nginx và trang mẫu từ ConfigMap

**Loại:** Tính năng.

**Bối cảnh:** trang chủ của `web` đang là một heredoc HTML trong `command` của init container, và
nginx chạy với cấu hình site mặc định của image. Lab 5a sẽ biến nginx thành reverse proxy cho
`/api/`, tức phải sửa cấu hình site. Ticket này đưa cả cấu hình site lẫn trang mẫu vào ConfigMap,
thiết kế sao cho Lab 5a **chỉ việc thêm** một khối `location`.

**Yêu cầu:**

1. ConfigMap `web-config` (thành phần `web`) có hai mục: cấu hình site của nginx và trang chủ mẫu.
2. Cấu hình site:
   - nghe `8080`, phục vụ tệp tĩnh từ `/usr/share/nginx/html`;
   - `GET /healthz` trả `200` với nội dung đúng `ok`, do **chính nginx** sinh — không cần file nào
     trong thư mục web;
   - `GET /status.json` trả kèm header `Cache-Control: no-store`;
   - có một dòng chú thích đánh dấu chỗ Lab 5a thêm `location /api/`.
3. nginx đọc cấu hình site từ ConfigMap. Thư mục cấu hình site trong container chỉ chứa đúng file
   cấu hình đó, không chứa trang mẫu. Sửa ConfigMap thì file trong container phải được cập nhật
   theo (B7 kiểm).
4. Init container `render-html` sinh `index.html` từ trang mẫu trong ConfigMap, thay chỗ giữ tên
   Pod và tên node bằng giá trị thật (downward API qua biến môi trường như sprint 1). Nó không còn
   ghi file `healthz`.
5. Sidecar `status-writer`, `preStop`, `securityContext`, user namespace giữ nguyên như sprint 1.
6. ConfigMap ở `k8s/app/web/configmap.yaml`, nằm trong kustomization.

**Ràng buộc:** không thay `/etc/nginx/nginx.conf` — dòng log `SIGQUIT` của Lab 3a B6 phụ thuộc vào
nó. Không dùng `subPath`.

**Đọc lại:**

- Bài [108](../108-configmap-vi.md), mục *ConfigMap và Pod* — ví dụ `game-demo`: bốn key nhưng
  container chỉ thấy hai file.
- Bài [275](../275-configure-pod-configmap-vi.md): mục *Thêm dữ liệu ConfigMap vào một path cụ thể
  trong Volume*; ghi chú ở mục *Nạp dữ liệu lưu trong ConfigMap vào một Volume* — mount che mất
  file có sẵn của image; mục *Định nghĩa key sử dụng khi tạo ConfigMap từ file*.
- [nginx-unprivileged](https://github.com/nginxinc/docker-nginx-unprivileged) — vị trí cấu hình
  site `/etc/nginx/conf.d/default.conf`.
- [Lab 3a](LAB-3A-POD-VA-VONG-DOI.md) ticket B5 — downward API cho tên Pod và tên node.

**Bàn giao:** `k8s/app/web/configmap.yaml`; `k8s/app/web/pod.yaml` đã cập nhật;
`docs/adr/0010-cau-hinh-web.md`; `~/lab-evidence/3b/b4-*`.

**Nghiệm thu:**

- `PASS:` Pod `web` Ready; `web-config` có đúng hai key.
- `PASS:` thư mục cấu hình site trong container `nginx` chỉ có một file, trùng nội dung mục cấu hình
  site trong ConfigMap; Pod không mount bằng `subPath`.
- `PASS:` `/healthz` trả `ok`; thư mục web không còn file `healthz`.
- `PASS:` `/status.json` mang header `Cache-Control: no-store` và trường `pod` là `web`.
- `PASS:` trang chủ có chữ `Bookstore` và tên node đang chạy Pod; trang mẫu trong ConfigMap không
  chứa tên node đó.
- `PASS:` `tests/smoke.sh` của Lab 3a vẫn PASS.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0010-cau-hinh-web.md`:

1. Mount ConfigMap vào cả thư mục `/etc/nginx/conf.d`: file `default.conf` gốc của image đi đâu?
   Nếu mount một file bằng `subPath` để giữ phần còn lại của thư mục thì mất gì?
2. Vì sao không mount cả `web-config` vào `/etc/nginx/conf.d` cho gọn?
3. Lab 5a thêm `location /api/`. Sửa ở đâu, và với Pod trần hiện tại phải làm gì để nginx dùng cấu
   hình mới? (B7 chứng minh.)
4. Vì sao trang mẫu đi qua ConfigMap mà tên Pod, tên node vẫn đi qua downward API?

<details>
<summary>Gợi ý</summary>

1. Một ConfigMap có thể sinh **hai** volume trong cùng Pod, mỗi volume một `items` khác nhau.
2. nginx: `return 200 "ok";` trong một `location = /healthz`; `add_header ... always;` trong
   `location = /status.json`.
3. Trang mẫu: dùng chỗ giữ như `__POD_NAME__`, `__NODE_NAME__`; init container thay bằng `sed`.
   `busybox` có `sed`.
4. Xem header: `kubectl -n bookstore exec web -c status-writer -- wget -S -O /dev/null http://127.0.0.1:8080/status.json`.
5. File cấu hình nginx lỗi cú pháp thì container `nginx` thoát ngay; đọc `kubectl logs web -c nginx`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3b
cd ~/bookstore && ./scripts/teardown.sh && ./scripts/deploy.sh > "$EV/b4-deploy.txt" 2>&1
get() { kubectl -n "$NS" exec web -c status-writer -- wget -qO- "http://127.0.0.1:8080$1"; }
cm() { kubectl -n "$NS" get configmap web-config -o jsonpath="$1"; }

kubectl -n "$NS" wait --for=condition=Ready pod/web --timeout=180s \
  && test "$(kubectl -n "$NS" get configmap web-config -o go-template='{{len .data}}')" -eq 2 \
  && echo 'PASS: web Ready, web-config co hai key'

F="$(kubectl -n "$NS" exec web -c nginx -- ls /etc/nginx/conf.d)"
cm '{.data.default\.conf}' > "$EV/b4-default.conf.cm"
kubectl -n "$NS" exec web -c nginx -- cat "/etc/nginx/conf.d/$F" > "$EV/b4-default.conf.pod"
{ test "$(echo "$F" | wc -l)" -eq 1 && diff -B -q "$EV/b4-default.conf.cm" "$EV/b4-default.conf.pod" >/dev/null && \
  test -z "$(kubectl -n "$NS" get pod web -o jsonpath='{.spec.containers[*].volumeMounts[*].subPath}')"; } \
  && echo 'PASS: conf.d chi co mot file, trung ConfigMap, khong dung subPath'

{ test "$(get /healthz)" = 'ok' && \
  kubectl -n "$NS" exec web -c nginx -- test ! -e /usr/share/nginx/html/healthz; } \
  && echo 'PASS: /healthz do nginx tra ok, khong con file healthz'
kubectl -n "$NS" exec web -c status-writer -- wget -S -O /dev/null http://127.0.0.1:8080/status.json \
  > "$EV/b4-status-headers.txt" 2>&1
{ grep -qi 'cache-control: no-store' "$EV/b4-status-headers.txt" && get /status.json | grep -q '"pod":"web"'; } \
  && echo 'PASS: status.json khong cho cache, truong pod la web'

NODE="$(kubectl -n "$NS" get pod web -o jsonpath='{.spec.nodeName}')"
get / > "$EV/b4-index.html"
{ grep -q 'Bookstore' "$EV/b4-index.html" && grep -q "$NODE" "$EV/b4-index.html" && \
  ! kubectl -n "$NS" get configmap web-config -o yaml | grep -q "$NODE"; } \
  && echo "PASS: trang chu sinh tu trang mau, co ten node $NODE"
./tests/smoke.sh
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/web/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
    app.kubernetes.io/name: nginx
data:
  default.conf: |
    # Cấu hình site của web (Bookstore). nginx chỉ đọc lại file này khi reload hoặc khởi động.
    server {
        listen      8080;
        server_name _;
        root        /usr/share/nginx/html;

        location = /healthz {
            access_log   off;
            default_type text/plain;
            return 200 "ok";
        }

        location = /status.json {
            add_header Cache-Control "no-store" always;
        }

        location / {
            index index.html;
        }

        # Lab 5a: thêm "location /api/" (reverse proxy sang api) ngay dưới dòng này.
    }
  index.html.tpl: |
    <!doctype html>
    <html lang="vi"><head><meta charset="utf-8"><title>Bookstore</title></head>
    <body>
    <h1>Bookstore</h1>
    <ul id="books"><li>Đang tải danh sách sách...</li></ul>
    <footer>Phục vụ bởi Pod __POD_NAME__ trên node __NODE_NAME__</footer>
    <script>
    fetch('/api/books').then(r => r.json()).then(bs => {
      document.getElementById('books').innerHTML =
        bs.map(b => '<li>' + b.title + ' — ' + b.author + '</li>').join('');
    }).catch(() => {
      document.getElementById('books').innerHTML = '<li>Chưa kết nối được API</li>';
    });
    </script>
    </body></html>
```

Trong `k8s/app/web/pod.yaml`: thêm hai volume từ `web-config`, viết lại `render-html`, thêm một
mount cho `nginx`. Metadata, `hostUsers`, `securityContext` cấp Pod, ân hạn, sidecar
`status-writer` giữ nguyên như cuối Lab 3a.

```yaml
  volumes:
  - name: html
    emptyDir: {}
  - name: podinfo
    downwardAPI:
      items:
      - path: labels
        fieldRef:
          fieldPath: metadata.labels
  - name: nginx-conf               # chỉ cấu hình site
    configMap:
      name: web-config
      items:
      - key: default.conf
        path: default.conf
  - name: page-template            # chỉ trang mẫu
    configMap:
      name: web-config
      items:
      - key: index.html.tpl
        path: index.html.tpl
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
      sed -e "s/__POD_NAME__/${POD_NAME}/g" -e "s/__NODE_NAME__/${NODE_NAME}/g" \
        /template/index.html.tpl > /html/index.html
      echo "render-html: da sinh /html/index.html tu trang mau"
    volumeMounts:
    - name: page-template
      mountPath: /template
      readOnly: true
    - name: html
      mountPath: /html
  # - name: status-writer ... giữ nguyên như Lab 3a
  containers:
  - name: nginx
    # image, cổng http, readinessProbe GET /healthz, preStop, securityContext: giữ nguyên
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
    - name: nginx-conf
      mountPath: /etc/nginx/conf.d
      readOnly: true
```

Thêm `- web/configmap.yaml` vào `resources`, rồi `./scripts/teardown.sh && ./scripts/deploy.sh`.

Ý chính cho ADR 0010:

1. Mount che mất **mọi** file sẵn có của image trong `/etc/nginx/conf.d` (ghi chú bài 275) — đó
   chính là điều ta muốn: chỉ còn cấu hình của dự án. `subPath` giữ được các file khác, nhưng bài
   108 nói container mount kiểu `subPath` **không nhận cập nhật** của ConfigMap.
2. nginx nạp mọi file `*.conf` trong thư mục đó; trang mẫu nằm lẫn vào là thứ không thuộc về cấu
   hình. `items` của bài 108 cho mỗi container thấy đúng phần nó cần.
3. Sửa mục `default.conf` của `web-config` trong repo. File trong container được cập nhật, nhưng
   nginx chỉ đọc cấu hình khi khởi động hoặc reload — với Pod trần, tạo lại Pod hoặc reload nginx.
4. ConfigMap là dữ liệu **chung** cho mọi bản sao `web` (Lab 4a có ba bản sao); tên Pod và tên node
   là dữ liệu **riêng** của từng Pod, chỉ downward API cho được.

</details>

## B5. [BS-3B-05] Mật khẩu ra khỏi manifest

**Loại:** Tính năng — trả `TD-02`.

**Bối cảnh:** Secret `db-credentials` có từ B1 nhưng chưa ai dùng; ba chỗ vẫn chứa mật khẩu viết
tay: biến superuser của `db`, dòng `CREATE ROLE authenticator` trong `db-init`, biến mật khẩu của
`api`. Ba chỗ phải đổi **cùng lúc**: đổi riêng `db` thì `api` còn dùng mật khẩu cũ, mà PostgREST
coi lỗi xác thực là lỗi không thử lại — `api` sẽ chết và `deploy.sh` treo tới hết thời gian chờ.

**Yêu cầu:**

1. Không file nào git theo dõi chứa mật khẩu: không có giá trị nào trong `secrets/db.env`, và
   `k8s/`, `scripts/` không còn mật khẩu cũ của sprint 1.
2. `db` nhận mật khẩu superuser dưới dạng **file**, qua biến môi trường kiểu `*_FILE` mà image
   `postgres` hỗ trợ; không còn biến chứa giá trị mật khẩu superuser. Container `postgres` chỉ thấy
   đúng **một** file mật khẩu là file nó cần, và file đó chỉ chủ sở hữu đọc được (quyền `400`).
3. Mật khẩu role `authenticator` lấy từ Secret và được đặt cho role **trong lúc khởi tạo DB**, bằng
   một script đặt cạnh `01-schema.sql` trong `db-init`. SQL không còn mật khẩu nào.
4. `api` lấy mảnh mật khẩu trong URI từ Secret; mọi thứ khác của B3 giữ nguyên.
5. Spec của Pod `db` và `api` không chứa giá trị mật khẩu nào.
6. Đăng nhập qua TCP bằng mật khẩu trong Secret thành công cho cả superuser và `authenticator`; mật
   khẩu cũ của sprint 1 và mật khẩu sai bị từ chối.

**Ràng buộc:** Secret `db-credentials` là nguồn duy nhất, không tạo Secret mới. Không dùng
`subPath`. Kết nối từ `127.0.0.1` bên trong container `postgres` **không chứng minh gì**: image cấu
hình `trust` cho localhost — kiểm bằng IP của Pod.

**Đọc lại:**

- Bài [109](../109-secret-vi.md): khối *Thận trọng* đầu bài, bước 3;
  [*Sử dụng Secret dưới dạng file trong Pod*](../109-secret-vi.md#using-secrets-as-files-from-a-pod);
  [*Sử dụng Secret làm biến môi trường*](../109-secret-vi.md#using-secrets-as-environment-variables);
  [*Secret tùy chọn*](../109-secret-vi.md#restriction-secret-must-exist).
- Bài [334](../334-distribute-credentials-secure-vi.md):
  [volume](../334-distribute-credentials-secure-vi.md#create-a-pod-that-has-access-to-the-secret-data-through-a-volume),
  [`items`](../334-distribute-credentials-secure-vi.md#project-secret-keys-to-specific-file-paths),
  [`defaultMode`](../334-distribute-credentials-secure-vi.md#set-posix-permissions-for-secret-keys),
  [biến môi trường từ Secret](../334-distribute-credentials-secure-vi.md#define-container-environment-variables-using-secret-data).
- [Docker Hub — postgres](https://hub.docker.com/_/postgres): mục *Environment Variables* (đoạn về
  `trust` cho localhost dưới `POSTGRES_PASSWORD`), mục *Docker Secrets* (biến `_FILE`), mục
  *Initialization scripts* (file `.sh` được chạy hay được *source*).

**Bàn giao:** `k8s/app/db/configmap.yaml`, `k8s/app/db/pod.yaml`, `k8s/app/api/pod.yaml.tpl` đã
cập nhật; `docs/adr/0011-mat-khau-tu-secret.md`; `~/lab-evidence/3b/b5-*`.

**Nghiệm thu:**

- `PASS:` hệ thống dựng lại từ repo; ba Pod Ready.
- `PASS:` `k8s/`, `scripts/` không còn mật khẩu cũ; không file nào sẽ vào git chứa mật khẩu mới.
- `PASS:` spec của `db` và `api` không chứa mật khẩu.
- `PASS:` `db` khai biến `POSTGRES_PASSWORD_FILE`, không khai `POSTGRES_PASSWORD`; file mật khẩu có
  quyền `400` và là file duy nhất trong thư mục của nó.
- `PASS:` superuser và `authenticator` đăng nhập qua IP Pod bằng mật khẩu trong Secret; mật khẩu cũ
  và mật khẩu sai bị từ chối.
- `PASS:` log `db` cho thấy entrypoint đã chạy (hoặc source) script `.sh` trong
  `/docker-entrypoint-initdb.d`.
- `PASS:` `tests/smoke.sh` của Lab 3a vẫn PASS.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0011-mat-khau-tu-secret.md`:

1. Superuser qua file, `authenticator` qua biến môi trường — trong cùng một container. Vì sao không
   dùng file cho cả hai? (Script khởi tạo chạy dưới user nào? File quyền `400` thuộc về ai?)
2. Mật khẩu cũ `lab-3a-*` vẫn nằm trong lịch sử git và trong bundle `lab-3a`. Có cần viết lại lịch
   sử không? Vì sao?
3. Sửa `secrets/db.env` khi hệ thống đang chạy rồi chạy `deploy.sh`: Secret đổi; `db` và `api` có
   đổi theo không? Phải làm gì để hệ thống dùng mật khẩu mới?
4. Đánh dấu tham chiếu Secret trong `api` là tùy chọn thì sao khi Secret chưa tồn tại? Có nên làm
   vậy không?

<details>
<summary>Gợi ý</summary>

1. Volume kiểu `secret` với `items` chỉ một key, `defaultMode: 0400`. Biến
   `POSTGRES_PASSWORD_FILE` trỏ tới đường dẫn file đó. Không khai đồng thời `POSTGRES_PASSWORD` —
   entrypoint báo lỗi hai biến loại trừ nhau.
2. Biến lấy từng key của Secret: `valueFrom.secretKeyRef`.
3. Script `02-authenticator.sh` trong `db-init`: file từ ConfigMap mặc định không có quyền thực thi,
   nên entrypoint *source* nó trong chính shell của entrypoint, sau khi đã chạy `01-schema.sql`, với
   user `postgres`. Dùng `psql -v ON_ERROR_STOP=1 --set=pw="$BIEN"` và viết `:'pw'` trong SQL để
   psql tự đặt nháy, khỏi tự ghép chuỗi.
4. Kiểm đăng nhập mà không để mật khẩu lọt vào dòng lệnh — đẩy mật khẩu qua stdin, nối tới IP Pod:

   ```bash
   printf '%s' "$PW" | kubectl -n bookstore exec -i db -c postgres -- env H="$DB_IP" \
     sh -c 'PGPASSWORD="$(cat)" psql -h "$H" -U authenticator -d bookstore -tAc "SELECT current_user"'
   ```

5. Ba file đổi cùng lúc, rồi `./scripts/teardown.sh && ./scripts/deploy.sh`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3b
cd ~/bookstore && ./scripts/teardown.sh && ./scripts/deploy.sh > "$EV/b5-deploy.txt" 2>&1 \
  && echo 'PASS: he thong dung lai tu repo, ba Pod Ready'
sec() { kubectl -n "$NS" get secret db-credentials -o go-template="{{index .data \"$1\"}}" | base64 -d; }
PGPW="$(sec postgres-password)"; AUPW="$(sec authenticator-password)"
DB_IP="$(kubectl -n "$NS" get pod db -o jsonpath='{.status.podIP}')"
login() { kubectl -n "$NS" exec -i db -c postgres -- env H="$DB_IP" U="$1" \
  sh -c 'PGPASSWORD="$(cat)" psql -h "$H" -U "$U" -d bookstore -tAc "SELECT current_user"' 2>/dev/null; }

{ ! git grep --untracked -qE 'lab-3a-mat-khau-tam|lab-3a-authenticator' -- k8s scripts && \
  ! git grep --untracked -qF -e "$PGPW" -e "$AUPW"; } \
  && echo 'PASS: repo khong con mat khau cu, khong chua mat khau moi'
kubectl -n "$NS" get pod db api -o yaml > "$EV/b5-spec.yaml"
grep -qF -e "$PGPW" -e "$AUPW" "$EV/b5-spec.yaml" \
  && echo 'FAIL: spec chua mat khau' || echo 'PASS: spec cua db va api khong chua mat khau'

C='{.spec.containers[?(@.name=="postgres")]'
ENVS=" $(kubectl -n "$NS" get pod db -o jsonpath="$C.env[*].name}") "
F="$(kubectl -n "$NS" get pod db -o jsonpath="$C.env[?(@.name==\"POSTGRES_PASSWORD_FILE\")].value}")"
{ case "$ENVS" in *' POSTGRES_PASSWORD '*) false ;; *' POSTGRES_PASSWORD_FILE '*) true ;; *) false ;; esac && \
  test "$(kubectl -n "$NS" exec db -c postgres -- stat -L -c %a "$F")" = '400' && \
  test "$(kubectl -n "$NS" exec db -c postgres -- ls "$(dirname "$F")")" = "$(basename "$F")"; } \
  && echo 'PASS: superuser doc qua file 400, file duy nhat trong thu muc'

{ test "$(printf '%s' "$PGPW" | login postgres)" = 'postgres' && \
  test "$(printf '%s' "$AUPW" | login authenticator)" = 'authenticator'; } \
  && echo 'PASS: hai user dang nhap qua IP Pod bang mat khau trong Secret'
{ test -z "$(printf '%s' 'lab-3a-authenticator' | login authenticator)" && \
  test -z "$(printf '%s' 'sai-mat-khau' | login postgres)"; } \
  && echo 'PASS: mat khau cu va mat khau sai bi tu choi'

kubectl -n "$NS" logs db -c postgres > "$EV/b5-db.log"
grep -Eq '(running|sourcing) /docker-entrypoint-initdb.d/[^ ]+\.sh' "$EV/b5-db.log" \
  && echo 'PASS: script dat mat khau authenticator da chay'
./tests/smoke.sh
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Trong `k8s/app/db/configmap.yaml`, dòng tạo role đổi thành không mật khẩu, và thêm một key:

```yaml
    CREATE ROLE authenticator NOINHERIT LOGIN;
    GRANT web_anon TO authenticator;
  02-authenticator.sh: |
    # Entrypoint của image postgres "source" file này (không có quyền thực thi) ngay sau
    # 01-schema.sql, dưới user postgres, và chỉ khi thư mục dữ liệu còn rỗng.
    if [ -z "${AUTHENTICATOR_PASSWORD:-}" ]; then
      echo '02-authenticator.sh: thieu AUTHENTICATOR_PASSWORD' >&2
      exit 1
    fi
    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" \
      --set=pw="$AUTHENTICATOR_PASSWORD" <<'EOSQL'
    ALTER ROLE authenticator WITH PASSWORD :'pw';
    EOSQL
    echo '02-authenticator.sh: da dat mat khau cho authenticator'
```

```yaml
# k8s/app/db/pod.yaml — sau B5
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
  priorityClassName: bookstore-critical
  terminationGracePeriodSeconds: 60
  volumes:
  - name: initdb
    configMap:
      name: db-init
  - name: credentials
    secret:
      secretName: db-credentials
      defaultMode: 0400               # bát phân, tức 256 thập phân
      items:
      - key: postgres-password        # chỉ key container này cần
        path: postgres-password
  - name: data
    emptyDir: {}
  containers:
  - name: postgres
    image: postgres:17.11
    env:
    - name: POSTGRES_DB
      value: bookstore
    - name: POSTGRES_PASSWORD_FILE
      value: /etc/db-credentials/postgres-password
    - name: AUTHENTICATOR_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: authenticator-password
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
    - name: credentials
      mountPath: /etc/db-credentials
      readOnly: true
    - name: data
      mountPath: /var/lib/postgresql/data
```

Trong `k8s/app/api/pod.yaml.tpl`, chỉ biến mật khẩu đổi nguồn:

```yaml
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: authenticator-password
```

```bash
cd ~/bookstore && ./scripts/teardown.sh && ./scripts/deploy.sh
```

Ý chính cho ADR 0011:

1. Entrypoint đọc `POSTGRES_PASSWORD_FILE` khi **còn là root**, trước khi hạ xuống uid 999; file
   quyền `400` thuộc root nên đọc được. Script khởi tạo chạy **sau** khi hạ quyền, dưới user
   `postgres`, nên không đọc được file `400` của root. Biến môi trường chấp nhận được ở đây vì ai
   `exec` được vào `db` thì đã có superuser qua Unix socket `trust` rồi.
2. Không cần. Mật khẩu cũ đã **vô hiệu**: DB mới được khởi tạo với mật khẩu sinh ngẫu nhiên, gate
   chứng minh mật khẩu cũ bị từ chối. Đổi bí mật đã lộ mới là biện pháp; viết lại lịch sử không thu
   hồi được các bản clone, bundle đã phát tán.
3. Không. Biến môi trường chỉ đặt lúc container khởi động (bài 334), file mật khẩu superuser chỉ
   được đọc lúc khởi tạo DB, và script khởi tạo chỉ chạy khi thư mục dữ liệu rỗng. Với Pod trần và
   `emptyDir`: `./scripts/teardown.sh && ./scripts/deploy.sh`. Tên Secret cố định không giúp gì vì
   không có controller nào theo dõi Secret để tạo lại Pod.
4. Bài 109: Secret tùy chọn không tồn tại thì Kubernetes bỏ qua — `api` khởi động với mật khẩu
   rỗng, PostgreSQL từ chối, PostgREST thoát. Một lỗi cấu hình rõ ràng (`CreateContainerConfigError`)
   biến thành một vòng `CrashLoopBackOff` khó đọc hơn. Không nên.

</details>

## B6. [BS-3B-06] Điều tra: nguồn đóng gói nào cho ra key nào

**Loại:** Điều tra.

**Bối cảnh:** đội vận hành cũ có sẵn một thư mục cấu hình nginx và một file `.env` cho PostgREST
từ hệ thống compose. Trước khi ai đó "đóng gói nhanh" chúng bằng `kubectl create configmap`, đội
cần biết chắc mỗi kiểu nguồn cho ra key nào — và chỗ nào âm thầm sai.

**Yêu cầu:** dựng nguồn trong `~/lab-work/3b/nguon/` **đúng như khối dưới**, rồi chứng minh bằng
`kubectl create configmap` ở chế độ chỉ in, không tạo object (trừ mục ⭐ 5):

1. Tạo từ **thư mục** `web/`: chỉ file thông thường thành key; thư mục con và symlink bị bỏ qua.
2. Tạo từ **env-file** `api.env`: mỗi dòng một key; dòng chú thích và dòng trống bị bỏ; dấu nháy
   nằm lại trong giá trị.
3. Tạo từ **file** `api.env`: cả file thành một key mang tên file; đổi được tên key.
4. ⭐ Một file nhị phân rơi vào trường nào của ConfigMap.
5. ⭐ ConfigMap vượt 1 MiB: API server có nhận không.

```bash
W=~/lab-work/3b/nguon
mkdir -p "$W/web/cu"
kubectl -n bookstore get configmap web-config -o jsonpath='{.data.default\.conf}'   > "$W/web/default.conf"
kubectl -n bookstore get configmap web-config -o jsonpath='{.data.index\.html\.tpl}' > "$W/web/index.html.tpl"
echo 'cau hinh cu' > "$W/web/cu/default.conf.bak"
ln -sf default.conf "$W/web/lien-ket.conf"
cat > "$W/api.env" <<'EOF'
# Cau hinh PostgREST — chep tu file .env cua he thong compose cu
PGRST_DB_SCHEMAS="api"
PGRST_DB_ANON_ROLE=web_anon

PGRST_SERVER_PORT=3000
EOF
printf '\x89PNG\r\n\x1a\n\xff\xfe' > "$W/logo.bin"
head -c 1500000 /dev/zero | tr '\0' 'a' > "$W/qua-lon.txt"
```

**Ràng buộc:** không sửa `web-config` hay `api-config` của dự án. Object duy nhất được phép tạo là
ConfigMap vượt trần ở mục 5 — và nó phải **không** được tạo.

**Đọc lại:**

- Bài [275](../275-configure-pod-configmap-vi.md):
  [*Tạo ConfigMap từ một thư mục*](../275-configure-pod-configmap-vi.md#create-configmaps-from-directories),
  [*Tạo ConfigMap từ file*](../275-configure-pod-configmap-vi.md#create-configmaps-from-files) — kể
  cả quy tắc của `--from-env-file`, mục *Định nghĩa key sử dụng khi tạo ConfigMap từ file*, mục
  *Hiểu về ConfigMap và Pod* — đoạn về `binaryData`.
- Bài [108](../108-configmap-vi.md): mục *Động lực* — trần 1 MiB;
  [*Đối tượng ConfigMap*](../108-configmap-vi.md#configmap-object).

**Bàn giao:** `docs/adr/0012-dong-goi-cau-hinh.md`; `~/lab-evidence/3b/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` từ thư mục: đúng hai key `default.conf`, `index.html.tpl`.
- `PASS:` từ env-file: đúng ba key; giá trị schema là `"api"` kèm dấu nháy.
- `PASS:` từ file: đúng một key `api.env`; đổi được tên key.
- `PASS:` (⭐) file nhị phân nằm ở `binaryData`, `data` rỗng.
- `PASS:` (⭐) ConfigMap vượt 1 MiB bị từ chối và không tồn tại.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0012-dong-goi-cau-hinh.md`:

1. Với `web-config`, `api-config`, `db-init`, kiểu nguồn nào hợp với từng cái? Vì sao repo vẫn lưu
   **YAML của ConfigMap** chứ không lưu file nguồn rồi sinh lúc triển khai? (Lab 5a sẽ đổi điều
   này.)
2. Ai đó bê nguyên `api.env` vào `api-config` bằng `--from-env-file`. Lệnh có báo lỗi không, và
   `api` hỏng ở đâu?
3. Kể hai thứ trong tương lai của dự án **không** được nhét vào ConfigMap, và đưa chúng đi đâu.

<details>
<summary>Gợi ý</summary>

- Chỉ in, không tạo object: thêm `--dry-run=client -o yaml` vào `kubectl create configmap`.
- Liệt kê key của kết quả: `-o go-template='{{range $k, $v := .data}}{{$k}} {{end}}'`.
- Đổi tên key: `--from-file=<key>=<đường dẫn>`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3b; W=~/lab-work/3b/nguon
keys() { kubectl -n "$NS" create configmap thu-nguon "$@" --dry-run=client \
           -o go-template='{{range $k, $v := .data}}{{$k}} {{end}}'; }
one() { kubectl -n "$NS" create configmap thu-nguon "$@" --dry-run=client -o jsonpath="$J"; }

keys --from-file="$W/web/" | tee "$EV/b6-thu-muc.txt"; echo
test "$(keys --from-file="$W/web/")" = 'default.conf index.html.tpl ' \
  && echo 'PASS: tu thu muc chi file thuong thanh key; thu muc con va symlink bi bo'

kubectl -n "$NS" create configmap thu-nguon --from-env-file="$W/api.env" --dry-run=client -o yaml \
  > "$EV/b6-env-file.yaml"
J='{.data.PGRST_DB_SCHEMAS}'
{ test "$(keys --from-env-file="$W/api.env")" = 'PGRST_DB_ANON_ROLE PGRST_DB_SCHEMAS PGRST_SERVER_PORT ' && \
  test "$(one --from-env-file="$W/api.env")" = '"api"'; } \
  && echo 'PASS: env-file: moi dong mot key, bo chu thich va dong trong, dau nhay nam trong gia tri'

{ test "$(keys --from-file="$W/api.env")" = 'api.env ' && \
  test "$(keys --from-file=cau-hinh-api="$W/api.env")" = 'cau-hinh-api '; } \
  && echo 'PASS: from-file: ca file thanh mot key mang ten file; doi duoc ten key'

# ⭐ 4
J='{.binaryData.logo}'; BIN="$(one --from-file=logo="$W/logo.bin")"
J='{.data}';            DAT="$(one --from-file=logo="$W/logo.bin")"
{ test -n "$BIN" && test -z "$DAT"; } && echo 'PASS: file nhi phan nam o binaryData, data rong'

# ⭐ 5
kubectl -n "$NS" create configmap thu-qua-lon --from-file=lon="$W/qua-lon.txt" > "$EV/b6-qua-lon.txt" 2>&1 \
  && echo 'FAIL: API server nhan ConfigMap vuot 1 MiB' || echo 'PASS: API server tu choi ConfigMap vuot 1 MiB'
kubectl -n "$NS" get configmap thu-qua-lon >/dev/null 2>&1 \
  && echo 'FAIL: thu-qua-lon ton tai' || echo 'PASS: thu-qua-lon khong duoc tao'
cat "$EV/b6-qua-lon.txt"
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của ADR 0012</summary>

1. `web-config`: tạo từ **thư mục** nếu giữ `default.conf` và trang mẫu thành file thật để sửa bằng
   trình soạn thảo có tô màu cú pháp. `api-config`: **env-file** là tự nhiên nhất — nếu file sạch
   dấu nháy. `db-init`: **file** với đúng tên `01-schema.sql`, vì entrypoint dựa vào đuôi `.sql`.
   Repo lưu YAML vì lúc này chưa có bước sinh nào nằm trong quy trình triển khai: YAML là thứ được
   apply, nên YAML phải là nguồn sự thật. Lab 5a đưa `configMapGenerator` vào và đổi điều đó.
2. Không báo lỗi gì. Bài 275: env-file **không xử lý đặc biệt dấu nháy** — giá trị thành `"api"`.
   `api` vẫn khởi động, nhưng PostgREST đi tìm một schema tên có nháy, không có; `/books` hỏng. Lỗi
   lộ ra ở tầng ứng dụng, không ở tầng Kubernetes.
3. Ví dụ: ảnh bìa sách và bản dump DB (Lab 4b, 6a) — dữ liệu nhị phân, lớn, thay đổi theo dữ liệu
   chứ không theo cấu hình. Bài 108, mục *Động lực*: vượt 1 MiB thì cân nhắc volume, cơ sở dữ liệu
   hoặc dịch vụ file riêng. Dự án sẽ đưa bản dump vào PVC ở Lab 6a.

Thư mục con `cu/` và symlink `lien-ket.conf` không thành key vì bài 275 nói rõ: tạo từ thư mục chỉ
đóng gói **file thông thường**; thư mục con, symlink, device, pipe bị bỏ qua. Byte `0xff` không hợp
lệ trong UTF-8 nên `kubectl` đẩy file sang `binaryData`.

</details>

## B7. [BS-3B-07] Điều tra: cấu hình đổi thì Pod thấy gì

**Loại:** Điều tra — phần A, C, D trên Pod thử nghiệm ghim `lab-k8s-worker2`; phần B trên `web`
thật.

**Bối cảnh:** sprint này đặt cấu hình ra ngoài spec để **đổi được mà không sửa manifest Pod**. Trước
khi viết quy trình đổi cấu hình cho người trực, đội phải biết chính xác: đổi ConfigMap thì cái gì
đổi theo, cái gì không, và ứng dụng có tự đọc lại không.

**Yêu cầu:**

1. **Phần A — volume và biến môi trường.** ConfigMap `thu-cap-nhat` có key `mode` giá trị `v1`. Pod
   `thu-cap-nhat` (`busybox`, chạy lâu) nhận cùng key đó theo hai đường: biến `APP_MODE` và file
   `/config/mode`. Sửa ConfigMap thành `v2`, chứng minh: file đổi, biến không đổi, Pod vẫn là Pod
   cũ. Tạo lại Pod thì biến mới đổi. ⭐ Thêm Pod `thu-subpath` mount **riêng** key `mode` bằng
   `subPath` tại `/config/mode`; chứng minh file của nó **không** đổi.
2. **Phần B — ứng dụng có đọc lại không.** Áp một bản `web-config` trong đó `/healthz` trả `ok-v2`.
   Chứng minh: file cấu hình trong container `nginx` đổi; nginx **vẫn** trả `ok`; reload nginx thì
   trả `ok-v2`; container không khởi động lại. Trả `web-config` về đúng bản trong repo.
3. **Phần C — bất biến.** ConfigMap `thu-bat-bien` có `immutable: true`: sửa dữ liệu bị từ chối, gỡ
   cờ bất biến cũng bị từ chối.
4. **Phần D — ranh giới namespace và tham chiếu tùy chọn.** Trong namespace `bookstore-thu` (mang
   label dự án), Pod `thu-khac-ns` lấy một biến từ key `default.conf` của `web-config` → container
   không được tạo. Pod `thu-tuy-chon` tham chiếu y hệt nhưng đánh dấu tùy chọn → chạy xong, biến
   không có giá trị. Xóa namespace.
5. Xóa mọi object thử nghiệm của phần A, C.

**Ràng buộc:** sửa ConfigMap của dự án chỉ trong phần B, và phải trả về bản trong repo trước khi
đóng ticket. `subPath` chỉ dùng trong Pod ⭐ của phần A, đúng một trường, để kiểm câu của bài 108 —
cơ chế `subPath` đầy đủ học ở giai đoạn 6.

**Đọc lại:**

- Bài [108](../108-configmap-vi.md): mục *ConfigMap đã mount được cập nhật tự động* — cả ghi chú
  `subPath`; [*ConfigMap bất biến*](../108-configmap-vi.md#configmap-immutable); mục *ConfigMap và
  Pod* — cùng namespace.
- Bài [275](../275-configure-pod-configmap-vi.md): mục *ConfigMap được mount sẽ tự động được cập
  nhật*; [*ConfigMap optional*](../275-configure-pod-configmap-vi.md#optional-configmaps); mục *Các
  hạn chế*.
- Bài [334](../334-distribute-credentials-secure-vi.md#define-container-environment-variables-using-secret-data)
  — đoạn đầu: biến lấy từ Secret cũng không thấy bản cập nhật.
- Bài [109](../109-secret-vi.md#secret-immutable) — Secret bất biến, cùng quy tắc.

**Bàn giao:** `docs/adr/0013-doi-cau-hinh.md`; `~/lab-evidence/3b/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` phần A — file trong volume thành `v2`, biến vẫn `v1`, cùng UID Pod; Pod tạo lại thấy biến
  `v2`. (⭐) File mount bằng `subPath` vẫn `v1`.
- `PASS:` phần B — file trong `nginx` đổi; `/healthz` vẫn `ok` trước reload, thành `ok-v2` sau reload;
  `restartCount` của `nginx` không đổi; `web-config` đã về bản trong repo và `/healthz` lại trả `ok`.
- `PASS:` phần C — sửa dữ liệu và gỡ cờ bất biến đều bị từ chối.
- `PASS:` phần D — `thu-khac-ns` kẹt `CreateContainerConfigError` với sự kiện nhắc `web-config`;
  `thu-tuy-chon` `Succeeded` và in `KHONG`; namespace `bookstore-thu` đã xóa.
- `PASS:` không còn object nào tên `thu-*` trong `bookstore`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0013-doi-cau-hinh.md`, dạng bảng cho người trực:

1. Với từng mục cấu hình của dự án — `db-init`, `api-config`, mục cấu hình site và mục trang mẫu
   của `web-config`, Secret `db-credentials` — đổi xong thì có hiệu lực khi nào, và người trực phải
   làm gì?
2. Có nên đánh dấu `web-config` bất biến ngay bây giờ không? Khi nào thì hợp lý? (Gợi ý: Lab 5a sinh
   ConfigMap có hậu tố hash.)
3. Muốn nginx tự reload khi file đổi, sidecar `status-writer` phải gửi tín hiệu cho nginx. Điều đó
   cần gì, và vì sao ADR 0005 của Lab 3a đã từ chối nó?

<details>
<summary>Gợi ý</summary>

- Sửa một key: `kubectl -n bookstore patch configmap thu-cap-nhat --type merge -p '{"data":{"mode":"v2"}}'`.
- Thời gian để file trong volume đổi phụ thuộc chu kỳ đồng bộ của kubelet và loại cache — phụ thuộc
  cấu hình; vòng chờ trong script cho tối đa ba phút. Bài 275 nêu cách ép làm mới sớm: sửa một
  annotation của Pod.
- Container `sleep` không phản ứng với `TERM` nên xóa Pod chờ hết ân hạn. Dùng
  `trap 'exit 0' TERM; sleep 3600 & wait`.
- Bản `web-config` thử nghiệm: `sed 's/return 200 "ok";/return 200 "ok-v2";/' k8s/app/web/configmap.yaml`.
- Reload nginx: `kubectl -n bookstore exec web -c nginx -- nginx -s reload`.
- Tham chiếu tùy chọn: thêm `optional: true` vào `configMapKeyRef`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã tự viết các manifest ở ~/lab-work/3b/</summary>

Phần A — cần `thu-cap-nhat-cm.yaml` (chỉ ConfigMap), `thu-cap-nhat-pod.yaml` (chỉ Pod), và ⭐
`thu-subpath.yaml`:

```bash
NS=bookstore; EV=~/lab-evidence/3b
kubectl apply -f ~/lab-work/3b/thu-cap-nhat-cm.yaml -f ~/lab-work/3b/thu-cap-nhat-pod.yaml
test -f ~/lab-work/3b/thu-subpath.yaml && kubectl apply -f ~/lab-work/3b/thu-subpath.yaml
kubectl -n "$NS" wait --for=condition=Ready pod/thu-cap-nhat --timeout=120s
f() { kubectl -n "$NS" exec thu-cap-nhat -- cat /config/mode; }
e() { kubectl -n "$NS" exec thu-cap-nhat -- printenv APP_MODE; }
U0="$(kubectl -n "$NS" get pod thu-cap-nhat -o jsonpath='{.metadata.uid}')"
{ test "$(f)" = 'v1' && test "$(e)" = 'v1'; } && echo 'PASS: truoc khi sua, file va bien deu v1'

kubectl -n "$NS" patch configmap thu-cap-nhat --type merge -p '{"data":{"mode":"v2"}}'
for i in $(seq 1 36); do test "$(f)" = 'v2' && break; sleep 5; done
{ echo "file=$(f)"; echo "env=$(e)"; } > "$EV/b7-a.txt"; cat "$EV/b7-a.txt"
{ test "$(f)" = 'v2' && test "$(e)" = 'v1' && \
  test "$(kubectl -n "$NS" get pod thu-cap-nhat -o jsonpath='{.metadata.uid}')" = "$U0"; } \
  && echo 'PASS: cung mot Pod: file trong volume doi, bien moi truong khong doi'
if kubectl -n "$NS" get pod thu-subpath >/dev/null 2>&1; then
  test "$(kubectl -n "$NS" exec thu-subpath -- cat /config/mode)" = 'v1' \
    && echo 'PASS: (sao) file mount bang subPath khong nhan cap nhat'
fi

kubectl -n "$NS" delete pod thu-cap-nhat --wait=true
kubectl apply -f ~/lab-work/3b/thu-cap-nhat-pod.yaml
kubectl -n "$NS" wait --for=condition=Ready pod/thu-cap-nhat --timeout=120s
test "$(e)" = 'v2' && echo 'PASS: Pod tao lai moi thay bien v2'
```

Phần B — trên `web` thật:

```bash
NS=bookstore; EV=~/lab-evidence/3b
cd ~/bookstore
get()  { kubectl -n "$NS" exec web -c status-writer -- wget -qO- "http://127.0.0.1:8080$1"; }
conf() { kubectl -n "$NS" exec web -c nginx -- cat /etc/nginx/conf.d/default.conf; }
rc()   { kubectl -n "$NS" get pod web -o jsonpath='{.status.containerStatuses[?(@.name=="nginx")].restartCount}'; }
R0="$(rc)"
sed 's/return 200 "ok";/return 200 "ok-v2";/' k8s/app/web/configmap.yaml > ~/lab-work/3b/web-config-v2.yaml
grep -q 'ok-v2' ~/lab-work/3b/web-config-v2.yaml \
  || echo 'DUNG: khong thay dong return 200 "ok"; sua lenh sed cho khop cau hinh cua ban'
kubectl apply -f ~/lab-work/3b/web-config-v2.yaml
for i in $(seq 1 36); do conf | grep -q 'ok-v2' && break; sleep 5; done
conf | grep -q 'ok-v2' && echo 'PASS: file cau hinh trong container nginx da doi'
test "$(get /healthz)" = 'ok' && echo 'PASS: nginx van phuc vu cau hinh cu'
kubectl -n "$NS" exec web -c nginx -- nginx -s reload; sleep 2
test "$(get /healthz)" = 'ok-v2' && echo 'PASS: reload xong nginx moi dung cau hinh moi'
test "$(rc)" = "$R0" && echo 'PASS: nginx khong khoi dong lai'

kubectl apply -f k8s/app/web/configmap.yaml
for i in $(seq 1 36); do conf | grep -q 'ok-v2' || break; sleep 5; done
kubectl -n "$NS" exec web -c nginx -- nginx -s reload; sleep 2
test "$(get /healthz)" = 'ok' && echo 'PASS: web-config da ve ban trong repo'
```

Phần C và D — cần `thu-bat-bien.yaml`, `ns-thu.yaml`, `thu-khac-ns.yaml`, `thu-tuy-chon.yaml`:

```bash
NS=bookstore; EV=~/lab-evidence/3b
kubectl apply -f ~/lab-work/3b/thu-bat-bien.yaml
kubectl -n "$NS" patch configmap thu-bat-bien --type merge -p '{"data":{"mode":"v2"}}' > "$EV/b7-c.txt" 2>&1 \
  && echo 'FAIL: sua duoc ConfigMap bat bien' || echo 'PASS: khong sua duoc du lieu khi immutable'
kubectl -n "$NS" patch configmap thu-bat-bien --type merge -p '{"immutable":false}' >> "$EV/b7-c.txt" 2>&1 \
  && echo 'FAIL: go duoc co immutable' || echo 'PASS: khong go duoc co immutable'

kubectl apply -f ~/lab-work/3b/ns-thu.yaml
kubectl apply -f ~/lab-work/3b/thu-khac-ns.yaml -f ~/lab-work/3b/thu-tuy-chon.yaml
R=''
for i in $(seq 1 24); do
  R="$(kubectl -n bookstore-thu get pod thu-khac-ns -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}')"
  test "$R" = 'CreateContainerConfigError' && break; sleep 5
done
kubectl -n bookstore-thu describe pod thu-khac-ns > "$EV/b7-d.txt"
{ test "$R" = 'CreateContainerConfigError' && grep -q 'web-config' "$EV/b7-d.txt"; } \
  && echo 'PASS: Pod o namespace khac khong tham chieu duoc web-config'
kubectl -n bookstore-thu wait --for=jsonpath='{.status.phase}'=Succeeded pod/thu-tuy-chon --timeout=120s \
  && test "$(kubectl -n bookstore-thu logs thu-tuy-chon)" = 'KHONG' \
  && echo 'PASS: tham chieu tuy chon thi Pod van chay, bien khong co gia tri'
kubectl delete namespace bookstore-thu --wait=true && echo 'PASS: da xoa namespace bookstore-thu'

kubectl -n "$NS" delete pod thu-cap-nhat thu-subpath --ignore-not-found --wait=true
kubectl -n "$NS" delete configmap thu-cap-nhat thu-bat-bien --ignore-not-found
test -z "$(kubectl -n "$NS" get pods,configmaps -o name | grep '/thu-')" && echo 'PASS: da don object thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/3b/thu-cap-nhat-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: thu-cap-nhat
  namespace: bookstore
data:
  mode: v1
```

```yaml
# ~/lab-work/3b/thu-cap-nhat-pod.yaml — thu-subpath.yaml giống hệt, đổi tên Pod, bỏ khối env,
# và mount bằng: mountPath: /config/mode, subPath: mode
apiVersion: v1
kind: Pod
metadata:
  name: thu-cap-nhat
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  volumes:
  - name: cau-hinh
    configMap:
      name: thu-cap-nhat
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "trap 'exit 0' TERM; sleep 3600 & wait"]
    env:
    - name: APP_MODE
      valueFrom:
        configMapKeyRef:
          name: thu-cap-nhat
          key: mode
    volumeMounts:
    - name: cau-hinh
      mountPath: /config
      readOnly: true
```

```yaml
# ~/lab-work/3b/thu-bat-bien.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: thu-bat-bien
  namespace: bookstore
data:
  mode: v1
immutable: true
```

```yaml
# ~/lab-work/3b/ns-thu.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore-thu
  labels:
    app.kubernetes.io/part-of: bookstore
```

```yaml
# ~/lab-work/3b/thu-khac-ns.yaml — thu-tuy-chon.yaml giống hệt, đổi tên Pod và thêm optional: true
apiVersion: v1
kind: Pod
metadata:
  name: thu-khac-ns
  namespace: bookstore-thu
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "if [ -n \"$WEB_CONF\" ]; then echo CO; else echo KHONG; fi"]
    env:
    - name: WEB_CONF
      valueFrom:
        configMapKeyRef:
          name: web-config
          key: default.conf
          # optional: true          # chỉ có ở thu-tuy-chon.yaml
```

Ý chính cho ADR 0013:

| Mục cấu hình | Đổi xong có hiệu lực khi nào | Người trực làm gì (Pod trần) |
| --- | --- | --- |
| `db-init` | Chỉ khi thư mục dữ liệu rỗng | Tạo lại `db` — mất dữ liệu `emptyDir` (`TD-04`); khi dữ liệu bền, dùng migration |
| `api-config` | Lúc container `postgrest` khởi động | Tạo lại `api` |
| `web-config` — cấu hình site | File đổi sau chu kỳ đồng bộ; nginx dùng khi reload | `nginx -s reload`, hoặc tạo lại `web` |
| `web-config` — trang mẫu | Lúc init container chạy | Tạo lại `web` |
| `db-credentials` | Biến: lúc container khởi động; mật khẩu trong DB: lúc khởi tạo | `teardown.sh && deploy.sh` (xem ADR 0011) |

1. Bảng trên. Điểm mấu chốt của phần B: **file đổi không có nghĩa ứng dụng đổi** — nginx chỉ đọc
   cấu hình lúc khởi động hoặc reload.
2. Chưa: `immutable` là một chiều (bài 108), mỗi lần sửa phải xóa, tạo lại ConfigMap rồi tạo lại Pod.
   Hợp lý khi mỗi phiên bản cấu hình mang **tên mới** — đúng điều `configMapGenerator` có hậu tố
   hash làm ở Lab 5a; khi đó không ai sửa một ConfigMap cũ nữa.
3. Cần `shareProcessNamespace: true` để sidecar thấy và gửi tín hiệu cho tiến trình nginx. Bài 292
   (ADR 0005): mọi container thấy `/proc` của nhau, kể cả biến môi trường — và từ sprint này biến
   môi trường của `api`, `db` chứa mật khẩu. Lý do từ chối còn mạnh hơn.

Phần D: bài 108 và 275 — Pod chỉ tham chiếu được ConfigMap trong **cùng namespace**; kubelet không
tìm thấy `web-config` trong `bookstore-thu` nên không tạo được container. Với `optional: true`,
phần cấu hình đó chỉ đơn giản trống.

</details>

## B8. [BS-3B-08] Điều tra: Secret — tạo bằng đường nào, bảo vệ tới đâu

**Loại:** Điều tra — phần A trên Secret thử nghiệm giá trị giả; phần B trên Secret thật của dự án,
không in giá trị nào ra.

**Bối cảnh:** dự án đã chọn đường Kustomize cho Secret mà chưa so với hai đường còn lại. Và câu hỏi
mà người duyệt bảo mật sẽ hỏi đầu tiên: "giờ mật khẩu DB nằm trong Secret rồi, vậy ai đọc được nó?"
Trả lời bằng bằng chứng, không bằng cảm giác.

**Yêu cầu:**

1. **Phần A — ba đường tạo.** Trong `bookstore`, giá trị giả:
   - `thu-literal`: tạo bằng dòng lệnh với mật khẩu `S!B\*d$zDsb=`; giải mã ra đúng chuỗi đó.
   - `thu-file-dung` và `thu-file-sai`: tạo từ file chứa `demo-password`, một file ghi bằng `echo -n`,
     một file ghi bằng `echo`; chứng minh độ dài giá trị là 13 và 14 byte.
   - `thu-manifest`: manifest có cả `data` (`username` = base64 của `demo-user`) và `stringData`
     (`username: demo-admin`); chứng minh `stringData` thắng và đọc lại không còn `stringData`.
   - Thư mục `~/lab-work/3b/kust`: bộ sinh Secret `thu-kust` từ hai giá trị trực tiếp; apply, đổi
     một giá trị, apply lại; chứng minh tên mang hậu tố, hai lần cho hai object, object cũ còn đó.
   - Pod `thu-envfrom` nạp cả Secret `thu-file-dung` thành biến với tiền tố `THU_`; chứng minh biến
     `THU_password` có mặt.
2. **Phần B — bảo vệ tới đâu,** trên `db-credentials`:
   - `describe` không in giá trị; một lệnh `jsonpath` cộng `base64 -d` ra đúng mật khẩu trong
     `secrets/db.env`.
   - Trên `lab-k8s-worker2`, volume Secret của Pod `db` nằm trên `tmpfs`.
   - Manifest kube-apiserver **chưa** có cờ cấu hình mã hóa at rest — nợ lab #6 vẫn mở.
   - Ai `exec` được vào `db` thì đọc được mật khẩu `authenticator` trong biến môi trường — `TD-07`
     mới giảm, chưa hết.
3. Cập nhật `docs/tech-debt.md`: `TD-02` trạng thái `đã trả`, `TD-07` trạng thái `giảm`, mỗi dòng
   trỏ tới bằng chứng.
4. Xóa mọi Secret và Pod `thu-*`, thư mục `~/lab-work/3b/kust`.

**Ràng buộc:** không in mật khẩu thật ra màn hình hay file. Chỉ **đọc** manifest kube-apiserver.
Không dùng `etcdctl`.

**Đọc lại:**

- Bài [325](../325-configmap-secret-vi.md) — ba trang con là ba đường.
- Bài [327](../327-secret-kubectl-vi.md): mục *Dùng dữ liệu thô*, *Dùng file nguồn* — cờ `-n`;
  [*Xác minh Secret*](../327-secret-kubectl-vi.md#verify-the-secret);
  [*Giải mã Secret*](../327-secret-kubectl-vi.md#decoding-secret) — khối *Thận trọng*.
- Bài [326](../326-secret-config-file-vi.md): [*Tạo Secret*](../326-secret-config-file-vi.md#create-the-config-file);
  mục *Chỉ định cả `data` lẫn `stringData`*.
- Bài [328](../328-secret-kustomize-vi.md): mục *Apply file kustomization*;
  [*Sửa một Secret*](../328-secret-kustomize-vi.md#edit-secret).
- Bài [109](../109-secret-vi.md): khối *Thận trọng* đầu bài — bốn bước tối thiểu;
  [*An toàn thông tin cho Secret*](../109-secret-vi.md#information-security-for-secrets).
- Bài [331](../331-define-environment-variable-vi.md) — `envFrom` và tiền tố chung;
  bài [334](../334-distribute-credentials-secure-vi.md#configure-all-key-value-pairs-in-a-secret-as-container-environment-variables).

**Bàn giao:** `docs/adr/0014-duong-tao-va-bao-ve-secret.md`; `docs/tech-debt.md` đã cập nhật;
`~/lab-evidence/3b/b8-*.txt` (không chứa mật khẩu thật).

**Nghiệm thu:**

- `PASS:` `thu-literal` giải mã ra đúng chuỗi có ký tự đặc biệt.
- `PASS:` giá trị từ file `echo -n` dài 13 byte, từ file `echo` dài 14 byte.
- `PASS:` `thu-manifest` mang `demo-admin`; đọc lại không có `stringData`.
- `PASS:` hai lần apply Kustomize cho hai Secret `thu-kust-<hash>` khác tên, cùng tồn tại.
- `PASS:` `thu-envfrom` có biến `THU_password`.
- `PASS:` `describe db-credentials` chỉ in số byte; `jsonpath` + `base64 -d` khớp `secrets/db.env`.
- `PASS:` volume Secret của `db` trên `lab-k8s-worker2` là `tmpfs`.
- `PASS:` kube-apiserver chưa bật mã hóa at rest — nợ lab #6 vẫn mở.
- `PASS:` `exec` vào `db` thấy mật khẩu `authenticator` trong môi trường.
- `PASS:` `docs/tech-debt.md` ghi `TD-02` đã trả, `TD-07` giảm; object thử nghiệm đã dọn.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0014-duong-tao-va-bao-ve-secret.md`:

1. So ba đường cho Secret của dự án. Vì sao **không** commit một manifest Secret với `data` base64
   vào repo? Đường dòng lệnh hợp lúc nào?
2. Liệt kê mọi đường mà một người trong cluster lab đọc được mật khẩu DB hôm nay. Đường nào dự án đã
   đóng, đường nào là nợ, trả ở lab nào?
3. Bốn bước tối thiểu trong khối *Thận trọng* của bài 109: dự án đã làm bước nào, bước nào còn nợ?

<details>
<summary>Gợi ý</summary>

- Ký tự đặc biệt trên dòng lệnh: bọc giá trị bằng nháy đơn.
- Độ dài giá trị đã giải mã: `kubectl get secret X -o jsonpath='{.data.password}' | base64 -d | wc -c`.
- Đường dẫn volume Secret trên node:
  `/var/lib/kubelet/pods/<UID của Pod>/volumes/kubernetes.io~secret/<tên volume>`. Hỏi loại
  filesystem: `findmnt -n -o FSTYPE --mountpoint <đường dẫn>`.
- Tên volume Secret của Pod `db`: `kubectl -n bookstore get pod db -o jsonpath='{.spec.volumes[?(@.secret)].name}'`.
- So sánh mà không in: đưa giá trị vào biến rồi `grep -qF "$BIEN"`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

Phần A — các Secret thử nghiệm đã tạo, `~/lab-work/3b/kust` đã apply hai lần, `thu-envfrom` đã
chạy:

```bash
NS=bookstore; EV=~/lab-evidence/3b
d() { kubectl -n "$NS" get secret "$1" -o jsonpath="{.data.$2}" | base64 -d; }
test "$(d thu-literal password)" = 'S!B\*d$zDsb=' && echo 'PASS: nhay don giu nguyen ky tu dac biet'
{ test "$(d thu-file-dung password | wc -c)" -eq 13 && test "$(d thu-file-sai password | wc -c)" -eq 14; } \
  && echo 'PASS: thieu -n thi ky tu xuong dong bi ma hoa vao gia tri'
{ test "$(d thu-manifest username)" = 'demo-admin' && \
  test -z "$(kubectl -n "$NS" get secret thu-manifest -o jsonpath='{.stringData}')"; } \
  && echo 'PASS: stringData thang data; doc lai khong con stringData'
kubectl -n "$NS" get secret -o name | grep 'secret/thu-kust-' | tee "$EV/b8-kust.txt"
test "$(grep -c 'secret/thu-kust-.' "$EV/b8-kust.txt")" -eq 2 \
  && echo 'PASS: Kustomize sinh object moi mang hau to hash, object cu con nguyen'
kubectl -n "$NS" wait --for=jsonpath='{.status.phase}'=Succeeded pod/thu-envfrom --timeout=120s
kubectl -n "$NS" logs thu-envfrom | grep -q '^THU_password=' && echo 'PASS: envFrom secretRef co tien to'
```

Phần B — Secret thật, không in giá trị:

```bash
NS=bookstore; EV=~/lab-evidence/3b
cd ~/bookstore
kubectl -n "$NS" describe secret db-credentials > "$EV/b8-describe.txt"
AU_F="$(sed -n 's/^authenticator-password=//p' secrets/db.env)"
AU_S="$(kubectl -n "$NS" get secret db-credentials -o go-template='{{index .data "authenticator-password"}}' | base64 -d)"
{ grep -q 'bytes' "$EV/b8-describe.txt" && ! grep -qF "$AU_F" "$EV/b8-describe.txt" && test "$AU_S" = "$AU_F"; } \
  && echo 'PASS: describe chi in so byte; jsonpath + base64 -d ra dung mat khau'

UID_DB="$(kubectl -n "$NS" get pod db -o jsonpath='{.metadata.uid}')"
VOL="$(kubectl -n "$NS" get pod db -o jsonpath='{.spec.volumes[?(@.secret)].name}')"
FS="$(ssh lab-k8s-worker2 "sudo findmnt -n -o FSTYPE --mountpoint /var/lib/kubelet/pods/$UID_DB/volumes/kubernetes.io~secret/$VOL")"
echo "fstype=$FS" | tee "$EV/b8-tmpfs.txt"
test "$FS" = 'tmpfs' && echo 'PASS: ban sao Secret tren node nam tren tmpfs'

ENC="$(sudo grep -c -- '--encryption-provider-config' /etc/kubernetes/manifests/kube-apiserver.yaml || true)"
test "$ENC" = '0' && echo 'PASS: chua bat ma hoa at rest — no lab #6 van mo'

kubectl -n "$NS" exec db -c postgres -- env | grep -qF "$AU_F" \
  && echo 'PASS: exec vao db doc duoc mat khau authenticator — TD-07 moi giam'

{ grep 'TD-02' docs/tech-debt.md | grep -q 'đã trả' && grep 'TD-07' docs/tech-debt.md | grep -q 'giảm'; } \
  && echo 'PASS: tech-debt.md ghi TD-02 da tra, TD-07 giam'
```

Dọn:

```bash
NS=bookstore
kubectl -n "$NS" delete pod thu-envfrom --ignore-not-found --wait=true
kubectl -n "$NS" get secret -o name | grep '^secret/thu-' | xargs -r kubectl -n "$NS" delete
rm -rf ~/lab-work/3b/kust
test -z "$(kubectl -n "$NS" get pods,secrets -o name | grep '/thu-')" && echo 'PASS: da don Secret va Pod thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
NS=bookstore; W=~/lab-work/3b
kubectl -n "$NS" create secret generic thu-literal --from-literal=username=demo-user \
  --from-literal=password='S!B\*d$zDsb='
echo -n 'demo-password' > "$W/pw-dung.txt"
echo 'demo-password'    > "$W/pw-sai.txt"
kubectl -n "$NS" create secret generic thu-file-dung --from-literal=username=demo-user \
  --from-file=password="$W/pw-dung.txt"
kubectl -n "$NS" create secret generic thu-file-sai --from-file=password="$W/pw-sai.txt"
rm -f "$W/pw-dung.txt" "$W/pw-sai.txt"

cat > "$W/thu-manifest.yaml" <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: thu-manifest
  namespace: bookstore
type: Opaque
data:
  username: ZGVtby11c2Vy        # base64 của demo-user
stringData:
  username: demo-admin
EOF
kubectl apply -f "$W/thu-manifest.yaml"

mkdir -p "$W/kust"
cat > "$W/kust/kustomization.yaml" <<'EOF'
secretGenerator:
- name: thu-kust
  namespace: bookstore
  literals:
  - username=demo-admin
  - password=demo-password
EOF
kubectl apply -k "$W/kust"
sed -i 's/password=demo-password/password=demo-password-2/' "$W/kust/kustomization.yaml"
kubectl apply -k "$W/kust"
```

```yaml
# ~/lab-work/3b/thu-envfrom.yaml
apiVersion: v1
kind: Pod
metadata:
  name: thu-envfrom
  namespace: bookstore
spec:
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.37
    command: ["env"]
    envFrom:
    - prefix: THU_
      secretRef:
        name: thu-file-dung
```

Hai dòng mới trong `docs/tech-debt.md` (giữ nguyên các dòng khác):

```markdown
| TD-02 | Mật khẩu trong manifest | 3a | 3b | đã trả | ~/lab-evidence/3b/b5-*; tests/smoke.sh của 3b |
| TD-07 | Debug/exec vào api, db đọc được mật khẩu | 3a | 3b (giảm), 9a | giảm | spec sạch (B5); exec vẫn đọc được (B8) |
```

Ý chính cho ADR 0014:

1. Manifest Secret với `data` base64 trong git là **mật khẩu trong git** — base64 không che được gì
   (phần B). Kustomize sinh Secret từ file nằm ngoài git; manifest thì không. Đường dòng lệnh hợp
   cho Secret dùng một lần lúc xử lý sự cố; bài 327 cảnh báo giá trị nằm lại trong lịch sử shell.
2. Các đường đọc được mật khẩu DB hôm nay:

   | Đường | Trạng thái | Trả ở |
   | --- | --- | --- |
   | Đọc repo, đọc bundle | đã đóng — `secrets/` không vào git | — |
   | `get` Secret qua API | mở — base64 | Lab 9a (RBAC tối thiểu) |
   | Tạo Pod trong `bookstore` rồi mount Secret (bài 109, khối *Thận trọng*) | mở | Lab 9a, 9b |
   | Đọc etcd | mở — nợ lab #6 | giai đoạn 22 |
   | `exec` vào `db` hoặc gắn debug container vào `api` | giảm — spec không còn mật khẩu (`TD-07`) | Lab 9a |
   | Root trên `lab-k8s-worker2` hoặc container `privileged` trên node đó (bài 109) | mở | Lab 9b (PSS) |
   | Đọc `secrets/db.env` trên master | quyền `600` | — |

3. Đã làm bước 3 — mỗi container chỉ thấy key nó cần (`items` ở B5). Bước 1 là nợ lab #6; bước 2
   (RBAC tối thiểu) ở Lab 9a; bước 4 (kho Secret ngoài) nằm ngoài phạm vi lộ trình.

</details>

## B9. [BS-3B-09] Tham số khởi động của `db` bằng `args`

**Loại:** Tính năng, kèm điều tra.

**Bối cảnh:** khi `api` không đăng nhập được, log của `db` hiện chỉ báo lỗi; nó không cho biết ai đã
kết nối thành công, và các tiến trình PostgreSQL trên node không mang tên Pod nào. Đội muốn bật hai
tham số khởi động của PostgreSQL mà **không** build image mới và **không** làm mất những gì
entrypoint của image đang làm cho mình (đọc `*_FILE`, chạy script khởi tạo, hạ quyền).

**Yêu cầu:**

1. PostgreSQL trong `db` khởi động với `log_connections=on` và `cluster_name` bằng **tên Pod**; tên
   Pod lấy bằng downward API rồi đưa vào tham số khởi động.
2. Entrypoint của image vẫn chạy: script khởi tạo vẫn được thực thi, mật khẩu vẫn đọc từ file.
3. Log `db` ghi lại kết nối thành công của `authenticator` từ `api`.
4. Điều tra ở `~/lab-work/3b/`, ghim `lab-k8s-worker2`, chạy một lần:
   - Pod `thu-command`: image `postgres`, **ghi đè lệnh chính** thành `postgres` — chứng minh
     container thất bại và đọc được vì sao.
   - Hai Pod `thu-env-goc` và `thu-env-de`: image `postgres`, cùng in giá trị `PGDATA`; Pod thứ hai
     khai `PGDATA=/tmp/pgdata-khac` trong manifest. Chứng minh biến trong Pod thắng biến của image.
   - Xóa ba Pod khi xong.

**Ràng buộc:** không đổi image, không thêm init container, không ghi đè lệnh chính của `db` thật.

**Đọc lại:**

- Bài [330](../330-define-command-argument-vi.md) — cả bài: `command` tương ứng gì, `args` tương ứng
  gì, chỉ khai `args` thì sao, `$(VAR)` trong `args`.
- Bài [331](../331-define-environment-variable-vi.md) — ghi chú thứ nhất: biến khai trong Pod ghi đè
  biến của image.
- [Lab 3a](LAB-3A-POD-VA-VONG-DOI.md) ticket B5 — tên Pod qua downward API thành biến môi trường.
- [Docker Hub — postgres](https://hub.docker.com/_/postgres) — image chạy `docker-entrypoint.sh`
  với lệnh mặc định `postgres`; PostgreSQL 17:
  [Error Reporting and Logging](https://www.postgresql.org/docs/17/runtime-config-logging.html)
  (`log_connections`, `cluster_name`).

**Bàn giao:** `k8s/app/db/pod.yaml` đã cập nhật; `docs/adr/0015-tham-so-khoi-dong-db.md`;
`~/lab-evidence/3b/b9-*`.

**Nghiệm thu:**

- `PASS:` `SHOW log_connections` trả `on`; `SHOW cluster_name` trả `db`.
- `PASS:` container `postgres` của `db` không ghi đè lệnh chính; log vẫn có dòng entrypoint chạy
  script khởi tạo.
- `PASS:` log `db` có `connection authorized: user=authenticator`.
- `PASS:` `thu-command` ở phase `Failed`, log báo PostgreSQL không được chạy bằng root.
- `PASS:` `thu-env-goc` in `/var/lib/postgresql/data`; `thu-env-de` in `/tmp/pgdata-khac`.
- `PASS:` ba Pod thử nghiệm đã xóa; `tests/smoke.sh` của Lab 3a vẫn PASS.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0015-tham-so-khoi-dong-db.md`:

1. Khai `args` thay vì `command`: entrypoint còn làm cho bạn những việc gì? Ghi đè `command` thì mất
   những việc đó — `thu-command` chứng minh việc nào?
2. `args` thay **toàn bộ** lệnh mặc định của image. Vì sao danh sách `args` của bạn phải bắt đầu
   bằng `postgres`?
3. Ai khai triển `$(POD_NAME)` trong `args`, và vào lúc nào? Viết `$POD_NAME` thì kết quả ra sao?

<details>
<summary>Gợi ý</summary>

1. PostgreSQL nhận tham số dạng `-c ten=gia_tri`.
2. Biến `POD_NAME` lấy từ `metadata.name` như `render-html` của `web`, rồi viết `$(POD_NAME)` trong
   `args`.
3. Đọc tham số đang chạy: `kubectl -n bookstore exec db -c postgres -- psql -U postgres -tAc 'SHOW cluster_name'`.
4. Pod thử nghiệm in một biến: `command: ["printenv", "PGDATA"]`, `restartPolicy: Never`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3b
cd ~/bookstore && ./scripts/teardown.sh && ./scripts/deploy.sh > "$EV/b9-deploy.txt" 2>&1
SQL() { kubectl -n "$NS" exec db -c postgres -- psql -U postgres -d bookstore -tAc "$1"; }
{ test "$(SQL 'SHOW log_connections')" = 'on' && test "$(SQL 'SHOW cluster_name')" = 'db'; } \
  && echo 'PASS: log_connections bat, cluster_name lay tu ten Pod'
kubectl -n "$NS" logs db -c postgres > "$EV/b9-db.log"
{ test -z "$(kubectl -n "$NS" get pod db -o jsonpath='{.spec.containers[?(@.name=="postgres")].command}')" && \
  grep -q 'running /docker-entrypoint-initdb.d/' "$EV/b9-db.log"; } \
  && echo 'PASS: khong ghi de lenh chinh, entrypoint van chay script khoi tao'
grep -q 'connection authorized: user=authenticator' "$EV/b9-db.log" && echo 'PASS: log ghi ket noi cua api'

kubectl apply -f ~/lab-work/3b/thu-command.yaml -f ~/lab-work/3b/thu-env.yaml
kubectl -n "$NS" wait --for=jsonpath='{.status.phase}'=Failed pod/thu-command --timeout=180s
kubectl -n "$NS" logs thu-command > "$EV/b9-thu-command.log" 2>&1
grep -q '"root" execution of the PostgreSQL server is not permitted' "$EV/b9-thu-command.log" \
  && echo 'PASS: ghi de command thi entrypoint bi bo qua, postgres chay bang root va tu choi'
kubectl -n "$NS" wait --for=jsonpath='{.status.phase}'=Succeeded pod/thu-env-goc pod/thu-env-de --timeout=180s
{ test "$(kubectl -n "$NS" logs thu-env-goc)" = '/var/lib/postgresql/data' && \
  test "$(kubectl -n "$NS" logs thu-env-de)" = '/tmp/pgdata-khac'; } \
  && echo 'PASS: bien khai trong Pod ghi de bien cua image'
kubectl -n "$NS" delete pod thu-command thu-env-goc thu-env-de --wait=true && echo 'PASS: da xoa Pod thu nghiem'
./tests/smoke.sh
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Container `postgres` trong `k8s/app/db/pod.yaml` — thêm `args` và biến `POD_NAME`, phần còn lại giữ
như B5:

```yaml
  containers:
  - name: postgres
    image: postgres:17.11
    args: ["postgres", "-c", "log_connections=on", "-c", "cluster_name=$(POD_NAME)"]
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POSTGRES_DB
      value: bookstore
    - name: POSTGRES_PASSWORD_FILE
      value: /etc/db-credentials/postgres-password
    - name: AUTHENTICATOR_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: authenticator-password
```

```yaml
# ~/lab-work/3b/thu-command.yaml
apiVersion: v1
kind: Pod
metadata:
  name: thu-command
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: postgres
    image: postgres:17.11
    command: ["postgres"]
```

```yaml
# ~/lab-work/3b/thu-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: thu-env-goc
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: app
    image: postgres:17.11
    command: ["printenv", "PGDATA"]
---
apiVersion: v1
kind: Pod
metadata:
  name: thu-env-de
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  restartPolicy: Never
  containers:
  - name: app
    image: postgres:17.11
    command: ["printenv", "PGDATA"]
    env:
    - name: PGDATA
      value: /tmp/pgdata-khac
```

Ý chính cho ADR 0015:

1. Bài 330: `command` tương ứng `ENTRYPOINT`, `args` tương ứng `CMD`. Chỉ khai `args` thì
   `docker-entrypoint.sh` vẫn chạy: đọc `POSTGRES_PASSWORD_FILE`, khởi tạo thư mục dữ liệu, chạy
   `/docker-entrypoint-initdb.d`, rồi hạ quyền xuống user `postgres`. Ghi đè `command` thì mất hết —
   `thu-command` chứng minh việc **hạ quyền**: binary `postgres` chạy thẳng bằng root và tự từ chối.
2. Lệnh mặc định của image là `postgres`. `args` thay toàn bộ phần đó, nên phải tự viết lại
   `postgres` rồi mới tới các cặp `-c`.
3. Kubernetes khai triển `$(POD_NAME)` khi tạo container, trước khi tiến trình chạy — không có
   shell nào ở đây. Bài 330: cặp ngoặc `$(VAR)` là bắt buộc; `$POD_NAME` được truyền nguyên văn,
   `cluster_name` sẽ thành chuỗi `$POD_NAME`.

</details>

## B10. [BS-3B-10] Chuẩn bị kéo image từ registry riêng

**Loại:** Điều tra.

**Bối cảnh:** khi dự án bắt đầu tự build image (trang tĩnh của `web`, operator ở Lab 14), image sẽ
nằm trong registry riêng. Cluster lab không có registry riêng nào; ticket này chuẩn bị mọi thứ còn
lại: hiểu Secret registry chứa gì, khai nó ở đâu, và kubelet phản ứng thế nào khi Secret sai hoặc
thiếu.

**Yêu cầu:**

1. Secret `thu-regcred` cho registry `registry.bookstore.invalid:5000`, user `demo-user`, mật khẩu
   `demo-password`, email `demo@bookstore.invalid`, tạo bằng dòng lệnh. Chứng minh: loại đúng của
   Secret registry; giải hai lớp base64 thì ra `demo-user:demo-password`.
2. Pod `thu-pull-co-secret` (`busybox`, ghim `lab-k8s-worker2`) khai `thu-regcred` làm Secret kéo
   image. Chứng minh Pod vẫn chạy, và giải thích vì sao.
3. Pod `thu-pull-thieu-secret` giống hệt nhưng khai Secret kéo image tên `khong-ton-tai`. Chứng
   minh Pod vẫn chạy **và** có event `FailedToRetrieveImagePullSecret`.
4. Xóa hai Pod và Secret.

**Ràng buộc:** giá trị giả, tên miền `.invalid`. Không dựng registry, không sửa cấu hình
containerd.

**Đọc lại:**

- Bài [287](../287-pull-image-private-registry-vi.md):
  [*Tạo Secret bằng cách cung cấp thông tin xác thực trên dòng lệnh*](../287-pull-image-private-registry-vi.md#create-a-secret-by-providing-credentials-on-the-command-line);
  mục *Kiểm tra Secret `regcred`*; mục *Tạo một Pod sử dụng Secret của bạn* — ghi chú namespace và
  event; mục *Sử dụng image từ nhiều registry*.
- Bài [109](../109-secret-vi.md): mục *Secret cấu hình Docker* — khối *Thận trọng*;
  [*Secret để kéo container image*](../109-secret-vi.md#using-imagepullsecrets).

**Bàn giao:** `docs/adr/0016-registry-rieng.md`; `~/lab-evidence/3b/b10-*.txt`.

**Nghiệm thu:**

- `PASS:` `thu-regcred` loại `kubernetes.io/dockerconfigjson`; trường `auth` giải ra
  `demo-user:demo-password`.
- `PASS:` `thu-pull-co-secret` Ready.
- `PASS:` `thu-pull-thieu-secret` Ready và có event `FailedToRetrieveImagePullSecret`.
- `PASS:` hai Pod và Secret đã xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0016-registry-rieng.md`:

1. Vì sao `thu-pull-co-secret` chạy được dù Secret của nó trỏ tới một registry không tồn tại? Vì
   sao `thu-pull-thieu-secret` cũng chạy được, và event của nó **nói gì, không nói gì**?
2. Khi dự án có registry riêng: Secret registry nằm ở namespace nào, khai ở đâu trong spec, sinh
   từ đâu để không vào git? Lab nào sẽ cho cách khai gọn hơn từng Pod?
3. Ai đọc được Secret registry thì có được gì? So với `db-credentials`.

<details>
<summary>Gợi ý</summary>

- `kubectl create secret docker-registry` có các cờ `--docker-server`, `--docker-username`,
  `--docker-password`, `--docker-email`.
- Lớp 1: `-o jsonpath='{.data.\.dockerconfigjson}' | base64 -d`. Lớp 2: lấy chuỗi `auth` trong JSON
  — không có `jq` trên node: `tr ',' '\n' | grep '"auth":' | cut -d'"' -f4` — rồi `base64 -d`.
- Secret kéo image khai ở **cấp Pod**, không trong `containers`.
- Event của một Pod: `kubectl get events --field-selector involvedObject.name=<Pod>,reason=<lý do>`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/3b
test "$(kubectl -n "$NS" get secret thu-regcred -o jsonpath='{.type}')" = 'kubernetes.io/dockerconfigjson' \
  && echo 'PASS: Secret registry dung loai'
kubectl -n "$NS" get secret thu-regcred -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d > "$EV/b10-config.json"
AUTH="$(tr ',' '\n' < "$EV/b10-config.json" | grep '"auth":' | cut -d'"' -f4)"
test "$(echo "$AUTH" | base64 -d)" = 'demo-user:demo-password' \
  && echo 'PASS: auth chi la base64 cua user:password'

kubectl -n "$NS" wait --for=condition=Ready pod/thu-pull-co-secret pod/thu-pull-thieu-secret --timeout=180s \
  && echo 'PASS: ca hai Pod deu chay'
EVT=''
for i in $(seq 1 24); do
  EVT="$(kubectl -n "$NS" get events -o name \
    --field-selector involvedObject.name=thu-pull-thieu-secret,reason=FailedToRetrieveImagePullSecret)"
  test -n "$EVT" && break; sleep 5
done
kubectl -n "$NS" describe pod thu-pull-thieu-secret > "$EV/b10-thieu-secret.txt"
test -n "$EVT" && echo 'PASS: event FailedToRetrieveImagePullSecret xuat hien'

kubectl -n "$NS" delete pod thu-pull-co-secret thu-pull-thieu-secret --wait=true
kubectl -n "$NS" delete secret thu-regcred && echo 'PASS: da don Pod va Secret thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
kubectl -n bookstore create secret docker-registry thu-regcred \
  --docker-server=registry.bookstore.invalid:5000 --docker-username=demo-user \
  --docker-password='demo-password' --docker-email=demo@bookstore.invalid
```

```yaml
# ~/lab-work/3b/thu-pull.yaml
apiVersion: v1
kind: Pod
metadata:
  name: thu-pull-co-secret
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  imagePullSecrets:
  - name: thu-regcred
  containers:
  - name: app
    image: busybox:1.37
    imagePullPolicy: IfNotPresent
    command: ["sh", "-c", "trap 'exit 0' TERM; sleep 3600 & wait"]
---
apiVersion: v1
kind: Pod
metadata:
  name: thu-pull-thieu-secret
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  imagePullSecrets:
  - name: khong-ton-tai
  containers:
  - name: app
    image: busybox:1.37
    imagePullPolicy: IfNotPresent
    command: ["sh", "-c", "trap 'exit 0' TERM; sleep 3600 & wait"]
```

Ý chính cho ADR 0016:

1. Bài 287, mục *Sử dụng image từ nhiều registry*: việc kéo image được thử bằng thông tin xác thực
   **khớp với registry của image**; không có cái nào khớp thì kéo không ủy quyền. `busybox` là image
   công khai trên registry khác `registry.bookstore.invalid`, nên Secret không được dùng tới. Với
   `khong-ton-tai`, kubelet ghi event `FailedToRetrieveImagePullSecret` — "không tìm thấy Secret mang
   tên đó" — rồi vẫn tiếp tục; image công khai nên Pod chạy. Event **không** nói registry từ chối gì
   cả. Với một image riêng tư, cùng event đó sẽ đi kèm `ImagePullBackOff`.
2. Secret phải nằm **cùng namespace** với Pod (ghi chú bài 287). Khai ở `spec.imagePullSecrets` của
   Pod. Sinh tại chỗ giống `db-credentials` — từ một file trong `secrets/`, qua Kustomize. Lab 9a gắn
   nó vào ServiceAccount để mọi Pod của thành phần tự nhận.
3. Bài 109 và 287: `auth` chỉ là base64 của `user:password` — ai đọc được Secret là có quyền đăng
   nhập registry, kéo và có thể đẩy image. Bị đẩy image độc vào registry còn nguy hiểm hơn lộ mật
   khẩu DB: nó chạy trong mọi Pod dùng image đó.

</details>

## B11. [BS-3B-11] ⭐ Đánh giá biến môi trường lấy từ file

**Loại:** Điều tra — tùy chọn.

**Bối cảnh:** một đề xuất cho sprint sau: init container của `api` đo thông tin lúc khởi động (ví
dụ địa chỉ DB đã kiểm được) rồi truyền cho container chính dưới dạng biến môi trường, qua một file
trong `emptyDir`. Trước khi quyết, kiểm cluster có hỗ trợ không, cú pháp file ra sao, và có dùng
được cho mật khẩu không.

**Yêu cầu:** ConfigMap `thu-envfile-nguon` chứa nội dung file env dưới đây; Pod `thu-envfile` ở
`~/lab-work/3b/` có init container chép file đó vào một `emptyDir`; container chính **không mount**
`emptyDir` mà nhận `DB_ENDPOINT` và `GHI_CHU` từ file. Chứng minh: hai biến đúng giá trị, `#` trong
nháy đơn không phải chú thích, key không được khai thì không thành biến. Ghi ADR. Xóa object khi
xong.

```text
# dong nay la chu thich
DB_ENDPOINT='10.0.0.1:5432'
GHI_CHU='a#b'
KHONG_DUNG='x'
```

**Đọc lại:** bài [332](../332-define-env-via-file-vi.md) — mục *Cách thiết kế này hoạt động*,
[*Cú pháp file env*](../332-define-env-via-file-vi.md#env-file-syntax), ghi chú về `emptyDir` và
thông tin nhạy cảm. Trạng thái tính năng ở baseline: đối chiếu
[A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) với dòng *Trạng thái tính năng* của bài.

**Nghiệm thu:**

- `PASS:` `kubectl explain` thấy trường của tính năng.
- `PASS:` `DB_ENDPOINT=10.0.0.1:5432` và `GHI_CHU=a#b` có trong môi trường; `KHONG_DUNG` thì không.
- `PASS:` object thử nghiệm đã xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/0017-bien-tu-file.md`: dùng cách này cho đề xuất
trên không? Có đưa mật khẩu `authenticator` đi đường này không?

<details>
<summary>Script nghiệm thu và lời giải tham khảo</summary>

```bash
kubectl explain pods.spec.containers.env.valueFrom.fileKeyRef > ~/lab-evidence/3b/b11-explain.txt \
  && echo 'PASS: baseline co truong fileKeyRef'
```

Lệnh trên báo không tìm thấy trường thì **dừng** B11 và ghi kết quả vào ADR. Không bật feature gate,
không sửa cấu hình kubelet hay apiserver — đó là lệch baseline của Lab 00.

```yaml
# ~/lab-work/3b/thu-envfile.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: thu-envfile-nguon
  namespace: bookstore
data:
  config.env: |
    # dong nay la chu thich
    DB_ENDPOINT='10.0.0.1:5432'
    GHI_CHU='a#b'
    KHONG_DUNG='x'
---
apiVersion: v1
kind: Pod
metadata:
  name: thu-envfile
  namespace: bookstore
spec:
  restartPolicy: Never
  initContainers:
  - name: setup-envfile
    image: busybox:1.37
    command: ["sh", "-c", "cp /source/config.env /data/config.env"]
    volumeMounts:
    - name: source
      mountPath: /source
      readOnly: true
    - name: config
      mountPath: /data
  containers:
  - name: use-envfile
    image: busybox:1.37
    command: ["env"]
    env:
    - name: DB_ENDPOINT
      valueFrom:
        fileKeyRef:
          path: config.env
          volumeName: config
          key: DB_ENDPOINT
          optional: false
    - name: GHI_CHU
      valueFrom:
        fileKeyRef:
          path: config.env
          volumeName: config
          key: GHI_CHU
          optional: false
  volumes:
  - name: source
    configMap:
      name: thu-envfile-nguon
  - name: config
    emptyDir: {}
```

```bash
NS=bookstore; EV=~/lab-evidence/3b
kubectl apply --dry-run=server -f ~/lab-work/3b/thu-envfile.yaml >/dev/null && kubectl apply -f ~/lab-work/3b/thu-envfile.yaml
kubectl -n "$NS" wait --for=jsonpath='{.status.phase}'=Succeeded pod/thu-envfile --timeout=180s
kubectl -n "$NS" logs thu-envfile -c use-envfile | sort > "$EV/b11-env.txt"
{ grep -qx 'DB_ENDPOINT=10.0.0.1:5432' "$EV/b11-env.txt" && grep -qx 'GHI_CHU=a#b' "$EV/b11-env.txt" && \
  ! grep -q '^KHONG_DUNG=' "$EV/b11-env.txt"; } \
  && echo 'PASS: chi key duoc khai thanh bien; # trong nhay don la ky tu thuong'
kubectl delete -f ~/lab-work/3b/thu-envfile.yaml --wait=true && echo 'PASS: da xoa object thu nghiem'
```

Ý chính cho ADR 0017: hợp với giá trị **sinh lúc khởi động** mà container chính không có shell để
tự đọc — đúng trường hợp `postgrest`. Không dùng cho mật khẩu: bài 332 nói thẳng `emptyDir` không có
các cơ chế bảo vệ như đối tượng Secret chuyên dụng — những cơ chế bạn đã thấy ở B5 và B8: chỉ gửi
tới node có Pod cần, bản sao trên `tmpfs`, `items` chọn đúng key cho từng container.

</details>

## B12. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 2 chỉ từ repo và một file
bí mật sinh tại chỗ, rồi trả cluster về đúng `01-cluster-ready`.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 3b — chép nguyên văn. Đây là kiểm thử khói **tích
   lũy**: mọi hợp đồng của sprint 1 cộng hợp đồng mới của sprint 2.

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 3b: ba Pod trần db, api, web; cấu hình từ ConfigMap;
# mật khẩu từ Secret db-credentials sinh tại chỗ. Script không in mật khẩu: mọi so sánh nằm trong biến.
set -u
cd "$(dirname "$0")/.."
NS=bookstore
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
cond() { kubectl -n "$NS" get pod "$1" -o jsonpath="{.status.conditions[?(@.type==\"$2\")].status}" 2>/dev/null; }
label() { kubectl -n "$NS" get "$1" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null; }
web() { kubectl -n "$NS" exec web -c status-writer -- wget -qO- "$1" 2>/dev/null; }
sec() { kubectl -n "$NS" get secret db-credentials -o go-template="{{index .data \"$1\"}}" 2>/dev/null | base64 -d 2>/dev/null; }
sql() { kubectl -n "$NS" exec db -c postgres -- psql -U postgres -d bookstore -tAc "$1" 2>/dev/null; }

# Sprint 1: ba Pod, dữ liệu, API, trang web
for P in db api web; do
  [ "$(cond "$P" Ready)" = 'True' ] && pass "Pod $P Ready" || fail "Pod $P chua Ready"
done
N="$(sql 'SELECT count(*) FROM api.books')"
[ "${N:-0}" -ge 5 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach"
API_IP="$(kubectl -n "$NS" get pod api -o jsonpath='{.status.podIP}' 2>/dev/null)"
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
[ "$(kubectl -n "$NS" get secret db-credentials -o jsonpath='{.type}' 2>/dev/null)" = 'Opaque' ] \
  && pass 'db-credentials loai Opaque' || fail 'db-credentials sai loai'

# Sprint 2: mật khẩu chỉ nằm trong Secret, và dùng được
PGPW="$(sec postgres-password)"; AUPW="$(sec authenticator-password)"
if [ -z "$PGPW" ] || [ -z "$AUPW" ]; then
  fail 'khong doc duoc hai key cua db-credentials'
else
  DB_IP="$(kubectl -n "$NS" get pod db -o jsonpath='{.status.podIP}' 2>/dev/null)"
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
ENVS=" $(kubectl -n "$NS" get pod db -o jsonpath='{.spec.containers[?(@.name=="postgres")].env[*].name}' 2>/dev/null) "
case "$ENVS" in
  *' POSTGRES_PASSWORD '*)      fail 'db van nhan mat khau superuser qua bien POSTGRES_PASSWORD' ;;
  *' POSTGRES_PASSWORD_FILE '*) pass 'db doc mat khau superuser qua file' ;;
  *)                            fail 'db khong khai POSTGRES_PASSWORD_FILE' ;;
esac

# Sprint 2: tham số khởi động của db
[ "$(sql 'SHOW log_connections')" = 'on' ] && pass 'db ghi log ket noi' || fail 'db chua bat log_connections'

exit "$FAIL"
EOF
```

2. Cập nhật `docs/tech-debt.md` nếu B8 chưa làm: `TD-02` đã trả, `TD-07` giảm.

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh
./tests/smoke.sh > ~/lab-evidence/3b/b12-smoke.txt; echo "smoke exit=$?"
cat ~/lab-evidence/3b/b12-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-3b: sprint 2 — cau hinh ngoai code, mat khau tu Secret'
git tag lab-3b
git bundle create ~/bookstore-lab-3b.bundle --all && git bundle verify ~/bookstore-lab-3b.bundle
```

Chép `~/bookstore-lab-3b.bundle` về máy host bằng `scp` trước khi đi tiếp. Bundle **không** chứa
`secrets/db.env` — đúng chủ đích ([mục 5.4](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm)).

5. Dọn và trả cluster:

```bash
kubectl delete namespace bookstore-thu --ignore-not-found --wait=true
rm -rf ~/lab-work/3b
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối:

```bash
SEL='app.kubernetes.io/part-of=bookstore'
test -z "$(git -C ~/bookstore status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git -C ~/bookstore describe --tags --exact-match HEAD | grep -qx 'lab-3b' && echo 'PASS: HEAD mang tag lab-3b'
git -C ~/bookstore bundle verify ~/bookstore-lab-3b.bundle >/dev/null 2>&1 && echo 'PASS: bundle hop le'
test "$(git -C ~/bookstore ls-files secrets)" = 'secrets/.gitkeep' && test -s ~/bookstore/secrets/db.env \
  && echo 'PASS: secrets/db.env con tren master, git khong theo doi'
test "$( { kubectl get namespace,priorityclass -l "$SEL" --no-headers
           kubectl get pods,configmaps,secrets -A -l "$SEL" --no-headers; } 2>/dev/null | wc -l)" -eq 0 \
  && echo 'PASS: khong con object nao cua du an'
test ! -e ~/lab-work/3b && echo 'PASS: da don lab-work'
test "$(sudo grep -c -- '--encryption-provider-config' /etc/kubernetes/manifests/kube-apiserver.yaml || true)" = '0' \
  && echo 'PASS: manifest kube-apiserver nguyen ven'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** `smoke exit=0` và mọi dòng của smoke ở bước 3 là `PASS:`; tám dòng `PASS:` của gate cuối
xuất hiện; ba node `Ready`; lệnh field selector trả `No resources found`; CoreDNS đủ replica;
namespace `default` không có Pod. Thư mục `~/lab-evidence/3b/` **giữ lại** — nó không chứa mật khẩu
thật. Cluster trở về `01-cluster-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 3b

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Kể hành trình của mật khẩu `authenticator`: từ `openssl` trên master tới lúc PostgREST đăng
      nhập được. Ở chặng nào nó là base64, chặng nào là chữ thường, ai khai triển nó vào URI?
- [ ] Vì sao Secret của dự án sinh bằng Kustomize nhưng **tắt** hậu tố hash, trong khi bài 328 coi
      hậu tố là ưu điểm? Lab nào sẽ đổi điều đó, và vì sao `deploy.sh` không gọi `kubectl apply -k`?
- [ ] Sửa `api-config`, mục cấu hình site của `web-config`, `db-init` khi hệ thống đang chạy: mỗi
      cái có hiệu lực khi nào, người trực phải làm gì?
- [ ] B7 phần B: file cấu hình trong container `nginx` đã đổi mà `/healthz` vẫn trả `ok`. Vì sao?
      Hai cách để nginx dùng cấu hình mới với Pod trần?
- [ ] URI của `api` đặt **trước** biến mật khẩu: Pod có được tạo không, container có chạy không,
      PostgREST làm gì, bạn tìm dấu vết ở đâu? `$$(VAR)` khác tham chiếu tới biến khai sau thế nào?
- [ ] Mount `conf.d` cả thư mục từ ConfigMap khác mount một file bằng `subPath` ở hai điểm nào? Vì
      sao `web-config` sinh hai volume với `items` khác nhau?
- [ ] Superuser đọc mật khẩu qua file quyền `400`, `authenticator` qua biến môi trường — vì sao? Vì
      sao kết nối từ `127.0.0.1` trong container `db` không chứng minh mật khẩu đúng?
- [ ] `immutable: true` đã đặt rồi, giờ cần đổi một giá trị. Làm thế nào, Pod đang mount cần gì?
      Pod ở namespace khác tham chiếu `web-config` thì hỏng thế nào; `optional: true` đổi gì, và vì
      sao không đặt nó cho Secret của `api`?
- [ ] `kubectl describe secret db-credentials` chỉ in số byte. Vậy mật khẩu đã được bảo vệ chưa? Kể
      các đường đọc được mật khẩu DB trong cluster lab hôm nay, đường nào đã đóng, đường nào là nợ.
- [ ] Image `postgres`: khai `command: ["postgres"]` thì sao, khai `args` thì entrypoint còn làm gì
      cho bạn? Biến `PGDATA` khai trong Pod và biến của image, cái nào thắng?
- [ ] `--from-env-file` với dòng `PGRST_DB_SCHEMAS="api"` cho ra giá trị gì? Tạo ConfigMap từ thư
      mục thì thư mục con và symlink đi đâu?
- [ ] Secret registry có mấy lớp base64, `imagePullSecrets` khai ở đâu? Event
      `FailedToRetrieveImagePullSecret` nói gì, và **không** nói gì?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- **Nợ lab #6** — mã hóa Secret at rest: phải sửa cấu hình kube-apiserver, trả ở
  [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu). Tới lúc đó, mọi
  kết luận "Secret an toàn" chỉ đúng ở mức quyền truy cập và phạm vi node, không ở mức lưu trữ.
- Phần hoãn ở mục 1.1: pull thật image riêng tư (không kiểm chứng được trong lộ trình),
  `imagePullSecrets` qua ServiceAccount và cách thứ tư dùng ConfigMap (Lab 9a), Kustomize đầy đủ
  (Lab 5a), `subPath` như cơ chế volume (Lab 6a). Nợ lab của Lab 3a vẫn mở như cũ.
- Nợ kỹ thuật: `TD-02` **đã trả**. `TD-07` **giảm** — spec không còn mật khẩu, nhưng ai `exec` được
  vào `db` hay gắn debug container vào `api` vẫn đọc được; giới hạn quyền ở Lab 9a. `TD-01`, `TD-03`,
  `TD-04`, `TD-05`, `TD-06` vẫn mở như sprint 1.

### Bài giải thích cuối cùng

Trong vài phút, kể lại hai câu chuyện của hệ thống bạn vừa dựng:

1. **Một mật khẩu.** `deploy.sh` thấy thiếu `secrets/db.env` và sinh nó; Kustomize biến file thành
   Secret `db-credentials`; kubelet trên `lab-k8s-worker2` chép một key vào `tmpfs` làm file cho
   entrypoint của `postgres` đọc khi còn là root, và đưa key kia vào biến môi trường cho script khởi
   tạo; kubelet trên node của `api` khai triển `$(DB_PASSWORD)` vào URI trước khi PostgREST chạy.
   Ở mỗi chặng nói rõ: dữ liệu nằm ở đâu, dạng gì, ai đọc được.
2. **Một thay đổi cấu hình.** Bạn sửa cấu hình site trong `web-config` để Lab 5a thêm `/api/`: thứ gì
   đổi ngay, thứ gì đổi sau chu kỳ đồng bộ của kubelet, thứ gì không bao giờ đổi cho tới khi bạn
   reload hoặc tạo lại Pod.

Kết thúc bằng việc chỉ ra Lab 5a sẽ thay đổi câu chuyện thứ hai ở đâu (ConfigMap mang hậu tố hash,
Pod được tạo lại khi cấu hình đổi) và vì sao nợ lab #6 làm câu chuyện thứ nhất chưa trọn.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B1: `kubectl apply -k k8s/app` báo `security; file ... is not in or below ...` | Kustomize mặc định không đọc file ngoài thư mục gốc; `secrets/db.env` nằm ngoài `k8s/app` | Triển khai bằng `./scripts/deploy.sh`; xem Kustomize render gì bằng `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app` |
| B1: Secret tên `db-credentials-<chuỗi>` hoặc ở namespace `default` | Thiếu khối `options` hoặc dòng `namespace` của bộ sinh | Chép lại khung kustomization nguyên văn; xóa Secret thừa bằng tay |
| B1: Secret có một key duy nhất tên `db.env` | Dùng dạng nguồn `files` thay vì `envs` | Bài 328: `files` lấy cả file làm một key; `envs` lấy mỗi dòng làm một key |
| B2: Pod `db` kẹt `ContainerCreating`, event `FailedMount ... configmap "db-init" not found` | ConfigMap chưa vào `resources` của kustomization, hoặc khác namespace | Thêm vào `resources`; `kubectl kustomize ... \| grep -A3 'kind: ConfigMap'` để kiểm |
| B2: `db` Ready nhưng không có schema; log có `ignoring /docker-entrypoint-initdb.d/...` | Key của ConfigMap không có đuôi `.sql` hay `.sh` | Đặt lại tên key; xóa Pod để có `emptyDir` mới |
| B3: `api` `CrashLoopBackOff`, log lần trước báo xác thực thất bại; spec còn chuỗi `$(DB_PASSWORD)` | URI đứng trước biến mật khẩu trong danh sách `env` | Đưa URI xuống sau mọi biến nó tham chiếu (bài 333) |
| B3: `api` Ready nhưng `/books` lỗi, schema không tìm thấy | `api-config` tạo từ env-file có dấu nháy, giá trị thành `"api"` | Sửa giá trị trong `k8s/app/api/configmap.yaml`, tạo lại `api` |
| B4: container `nginx` thoát ngay, log có `[emerg]` | Cú pháp cấu hình site sai | `kubectl logs web -c nginx`; sửa `default.conf` trong ConfigMap, tạo lại Pod |
| B4: trang chủ còn `__POD_NAME__` | `sed` trong `render-html` không khớp chỗ giữ, hoặc init mount sai volume | `kubectl logs web -c render-html`; kiểm tên chỗ giữ trong trang mẫu |
| B5: `db` `CrashLoopBackOff`, log `Database is uninitialized and superuser password is not specified` | `POSTGRES_PASSWORD_FILE` trỏ sai đường dẫn, hoặc `items.path` khác tên file | `kubectl exec db -c postgres -- ls -la <thư mục>`; sửa cho khớp |
| B5: log `both POSTGRES_PASSWORD and POSTGRES_PASSWORD_FILE are set` | Còn sót biến cũ | Xóa `POSTGRES_PASSWORD` khỏi manifest |
| B5: `api` không đăng nhập được dù Secret đúng | Script đặt mật khẩu lỗi ở lần khởi tạo đầu; lần khởi động lại thấy thư mục dữ liệu không rỗng nên bỏ qua script | Đọc log `db` tìm dòng của script; sửa rồi **xóa Pod** `db` để có `emptyDir` mới |
| B5: quyền file mật khẩu là `620` thay vì `400` | `defaultMode` bị hiểu là số thập phân 400 | Viết `0400` trong YAML, hoặc `256` |
| B5: đổi `secrets/db.env` khi hệ thống đang chạy, `api` hỏng xác thực | DB giữ mật khẩu cũ; Pod mới nhận mật khẩu mới | `./scripts/teardown.sh && ./scripts/deploy.sh` (ADR 0011) |
| B7: file trong volume chưa đổi sau 3 phút | Patch vào sai namespace, hoặc đang xem Pod mount bằng `subPath` | `kubectl -n bookstore get cm thu-cap-nhat -o yaml`; đối chiếu đúng Pod `thu-cap-nhat` |
| B7: `nginx -s reload` báo không mở được file pid | `exec` vào sai container | Thêm `-c nginx` |
| B8: `findmnt` không in gì | Hỏi sai node hoặc sai tên volume | `kubectl get pod db -o wide`; lấy tên volume từ spec như gợi ý |
| B9: `SHOW cluster_name` trả `$(POD_NAME)` | Biến `POD_NAME` chưa khai trong container `postgres` | Thêm biến downward API vào chính container đó |
| B9: `db` không khởi động, log có `"root" execution ... not permitted` | Khai `command` thay vì `args` cho `db` thật | Đổi lại thành `args` |
| B10: Pod `thu-pull-*` ở `ErrImagePull` | Node chưa có `busybox` và không kéo được từ Docker Hub | Chạy lại khối kéo image ở B0 của Lab 3a |
| B12: smoke `FAIL: mat khau xuat hien trong spec cua Pod` | Còn một biến mật khẩu viết thẳng trong manifest | `kubectl get pods -o yaml \| grep -n PASSWORD`; sửa repo, teardown rồi deploy |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Configuration](https://v1-35.docs.kubernetes.io/docs/concepts/configuration/)
- [Kubernetes v1.35 — ConfigMaps](https://v1-35.docs.kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes v1.35 — Secrets](https://v1-35.docs.kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes v1.35 — Configure a Pod to Use a ConfigMap](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Kubernetes v1.35 — Pull an Image from a Private Registry](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)
- [Kubernetes v1.35 — Managing Secrets](https://v1-35.docs.kubernetes.io/docs/tasks/configmap-secret/)
- [Kubernetes v1.35 — Managing Secrets using kubectl](https://v1-35.docs.kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/)
- [Kubernetes v1.35 — Managing Secrets using Configuration File](https://v1-35.docs.kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-config-file/)
- [Kubernetes v1.35 — Managing Secrets using Kustomize](https://v1-35.docs.kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kustomize/)
- [Kubernetes v1.35 — Inject Data Into Applications](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/)
- [Kubernetes v1.35 — Define a Command and Arguments for a Container](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/)
- [Kubernetes v1.35 — Define Environment Variables for a Container](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
- [Kubernetes v1.35 — Define Environment Variable Values Using An Init Container](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-via-file/)
- [Kubernetes v1.35 — Define Dependent Environment Variables](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/define-interdependent-environment-variables/)
- [Kubernetes v1.35 — Distribute Credentials Securely Using Secrets](https://v1-35.docs.kubernetes.io/docs/tasks/inject-data-application/distribute-credentials-secure/)
- [Kubernetes — kubectl kustomize (cờ `--load-restrictor`)](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_kustomize/)
- [Docker Hub — postgres (Environment Variables, Docker Secrets, Initialization scripts)](https://hub.docker.com/_/postgres)
- [PostgreSQL 17 — Error Reporting and Logging](https://www.postgresql.org/docs/17/runtime-config-logging.html)
- [PostgREST v14 — Configuration](https://docs.postgrest.org/en/v14/references/configuration.html)
- [GitHub — nginxinc/docker-nginx-unprivileged](https://github.com/nginxinc/docker-nginx-unprivileged)
