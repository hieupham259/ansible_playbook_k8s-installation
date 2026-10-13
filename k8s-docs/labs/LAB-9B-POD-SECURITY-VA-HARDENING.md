# Lab 9b — Pod Security và hardening

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md)
> tạo, xem [chuỗi snapshot](README.md#3-chuỗi-snapshot). Repo `~/bookstore` ở tag `lab-9a` (hoặc
> bundle `~/bookstore-lab-9a.bundle` nếu bạn vừa restore snapshot).
> **Điểm kết thúc:** teardown trả cluster về đúng `03-storage-ready`, **không tạo snapshot mới**.
> Repo `~/bookstore` có tag `lab-9b` và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 16 — Hardening của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 9a — ServiceAccount, authn/authz và RBAC](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md).
> **Lab sau:** [Lab 11a — Observability](LAB-11A-OBSERVABILITY.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng **phần policy và hardening** của
[Giai đoạn 9 — Bảo mật và multi-tenancy](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy):
các bài [115](../115-pod-security-standards-vi.md), [116](../116-pod-security-admission-vi.md),
[121](../121-secrets-good-practices-vi.md), [122](../122-multi-tenancy-vi.md),
[123](../123-hardening-authentication-vi.md), [126](../126-linux-security-vi.md),
[127](../127-linux-kernel-security-vi.md), [128](../128-api-server-bypass-risks-vi.md),
[129](../129-security-checklist-vi.md), [130](../130-application-security-checklist-vi.md),
[166](../166-flow-control-vi.md), [173](../173-admission-webhooks-vi.md) và bài lịch sử
[117](../117-pod-security-policy-vi.md), cộng sáu bài thực hành của giai đoạn 9 mà lab này kiểm
chứng ([254](../254-running-cloud-controller-vi.md), [258](../258-sysctl-cluster-vi.md),
[282](../282-enforce-standards-admission-controller-vi.md),
[283](../283-enforce-standards-namespace-labels-vi.md), [286](../286-migrate-from-psp-vi.md),
[291](../291-security-context-vi.md)). Mọi ticket dựa vào
[hợp đồng kỹ thuật](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật), [thiết kế bảo mật ở mục
3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật) và hàng "9b" của
[bố cục repo theo sprint](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Kế hoạch gốc nằm ở
[mục 7.2](DU-AN-BOOKSTORE.md#sprint-16--lab-9b-hardening).

Sprint 15 cho mỗi thành phần một danh tính và quyền tối thiểu. Nhưng `db` vẫn khởi động bằng root
(`TD-06`), namespace `bookstore` vẫn ở mặc định `privileged` của Pod Security Admission, và chưa
container nào drop capability hay khóa root filesystem. Sprint này đóng các lỗ đó, dựng tenant thứ
hai, rồi rà toàn bộ hệ thống theo hai checklist bảo mật.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); image của dự án nằm
ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Lab không chép lại con
số phiên bản nào trong phần văn bản, **không cài thêm gì**, **không cài admission webhook** và
**không sửa cấu hình `kube-apiserver` hay kubelet**. Chỗ duy nhất cần số minor của Kubernetes là nhãn
`pod-security.kubernetes.io/<MODE>-version`: nó nằm trong manifest như một tag image, và mọi script
nghiệm thu đối chiếu nó với `/version` của chính API server.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi namespace `bookstore` chạy **toàn bộ** hệ thống dưới `enforce: restricted`,
`bookstore-staging` là một tenant cô lập, `docs/security-review.md` ghi kết quả rà soát **và** bạn
chứng minh, giải thích được:

- Ranh giới của ba profile bằng thực nghiệm: thứ gì làm một Pod trượt Baseline, Restricted **thêm**
  bốn ràng buộc nào, và vì sao một Pod không khai gì vẫn trượt Restricted.
- Ba chế độ `enforce`, `warn`, `audit` khác nhau ở đâu — `violates` so với `would violate`, vì sao
  `enforce` không áp cho Deployment còn `warn` thì có, và phần nào của `audit` cluster này không đọc
  được.
- Lộ trình siết một namespace đang chạy: `warn`/`audit` trước, kiểm kê, sửa, dry-run, rồi mới
  `enforce` — và chỗ dry-run bỏ sót (CronJob chưa tới giờ).
- `db` chạy uid 999 từ đầu tới cuối: vì sao `PGDATA` phải là thư mục con của volume, vì sao file mật
  khẩu cần `fsGroup`, và bằng chứng nào cho thấy entrypoint không cần `chown` khi chạy non-root.
- Đọc được ràng buộc kernel **thật** của từng container (`CapEff`, `NoNewPrivs`, `Seccomp`) thay vì
  tin manifest; vì sao `readOnlyRootFilesystem` không nằm trong Restricted mà nằm trong checklist.
- Gỡ lỗi `api` không có shell **dưới** `restricted`: vì sao ephemeral container mặc định bị từ chối.
- Mô hình bốn lớp của bài 122 trên `bookstore-staging`: mỗi lớp chặn gì, và tenant **không** được
  bảo vệ khỏi điều gì.
- Ai đọc được Secret `db-credentials` — trực tiếp qua RBAC và gián tiếp qua quyền tạo workload — và
  vì sao nợ lab #6 vẫn mở.
- Tầng API server: request của admin, CI và đội phát triển rơi vào FlowSchema nào; `failurePolicy`
  mặc định của webhook; PodSecurityPolicy đã biến mất khỏi API.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài | Ticket kiểm chứng |
| --- | --- |
| [115 — Chuẩn bảo mật Pod](../115-pod-security-standards-vi.md) | B1 — ranh giới Baseline bằng năm lần từ chối, Restricted thêm bốn ràng buộc; B3–B5 — sửa từng vi phạm của Bookstore theo bảng Restricted |
| [116 — Pod Security Admission](../116-pod-security-admission-vi.md) | B1 — ba chế độ, `enforce` không áp cho workload resource, metric `pod_security_evaluations_total`; B2 — `warn`/`audit` trước; B6 — `enforce` và nhãn version |
| [117 — Chính sách bảo mật Pod](../117-pod-security-policy-vi.md) *(lịch sử)* | B10 — chứng minh API đã bị gỡ; không dựng lại PSP |
| [121 — Thực hành tốt cho Secret](../121-secrets-good-practices-vi.md) | B9 — ai đọc được `db-credentials`, `list` là đọc, người tạo được workload đọc gián tiếp, Secret chỉ tới đúng container cần; nợ lab #6 |
| [122 — Đa người thuê](../122-multi-tenancy-vi.md) | B8 — tenant `bookstore-staging` với đủ bốn lớp, kiểm từng lớp; lớp còn thiếu ghi vào ADR |
| [123 — Hardening xác thực](../123-hardening-authentication-vi.md) | B11 — cơ chế xác thực nào đang bật; nhóm trong trường `O` của certificate admin |
| [126 — Bảo mật cho node Linux](../126-linux-security-vi.md) | B9 — volume Secret là `tmpfs`, ba node không có swap |
| [127 — Ràng buộc kernel](../127-linux-kernel-security-vi.md) | B3, B4 — đọc `CapEff`, `NoNewPrivs`, `Seccomp` của container thật; B6 — đọc từ ephemeral container; B7 ⭐ — container đặc quyền ghi đè cả ba |
| [128 — Rủi ro vượt qua API server](../128-api-server-bypass-risks-vi.md) | B9 — không danh tính nào của dự án có `nodes/proxy`; B11 — bốn đường vòng, kiểm phần đọc được trên node |
| [129 — Checklist bảo mật](../129-security-checklist-vi.md) | B11 — tám nhóm ô, mỗi ô một câu trả lời lấy từ cluster, ghi vào `docs/security-review.md` |
| [130 — Checklist bảo mật ứng dụng](../130-application-security-checklist-vi.md) | B11 — soi từng workload của Bookstore theo checklist |
| [166 — API Priority and Fairness](../166-flow-control-vi.md) | B10 — đối tượng bắt buộc và được đề xuất, FlowSchema mà request của admin, CI, đội phát triển rơi vào |
| [173 — Admission webhook](../173-admission-webhooks-vi.md) | B10 — tồn kho webhook, `failurePolicy` mặc định đọc từ server dry-run; **không cài webhook** |
| [254 — Cloud Controller Manager](../254-running-cloud-controller-vi.md) | B11 — cluster không có cloud provider, nên hai ô của checklist 129 không áp dụng |
| [258 — sysctl](../258-sysctl-cluster-vi.md) | B4 — ba sysctl keepalive của `api` (Lab 9a, phía kubelet) vẫn hợp lệ dưới Restricted; B7 ⭐ — sysctl không an toàn chết ở hai mặt phẳng khác nhau |
| [282 — Cấu hình admission controller](../282-enforce-standards-admission-controller-vi.md) | B1 — namespace không nhãn mặc định `privileged`; B10 — apiserver không có `--admission-control-config-file` |
| [283 — Nhãn namespace](../283-enforce-standards-namespace-labels-vi.md) | B1 — khuôn nhãn; B2 — `enforce` mức chắc chắn, `warn`/`audit` mức mong muốn; B6 — dry-run trước khi siết |
| [286 — Di chuyển từ PSP](../286-migrate-from-psp-vi.md) | B2 — bước 3.a, 3.b và chỗ dry-run bỏ sót; B6 — bước 3.c; B9 — bước 1 rà quyền namespace; B10 — bước 2, 3.d, 5 không còn làm được |
| [291 — Security Context](../291-security-context-vi.md) | B3 — `runAsUser`, `runAsGroup`, `fsGroup` trên volume thật (uid cấp Pod và `allowPrivilegeEscalation` đã có ở Lab 9a); B4 — cấp Pod và cấp container, capability, `readOnlyRootFilesystem`; B7 ⭐ — `/proc` bị che |

Bài [120](../120-rbac-good-practices-vi.md) thuộc nhóm của Lab 9a; B9 dùng lại các mục *Tạo
workload*, *Liệt kê secret* và *Sửa đổi Namespace* của nó.

Các phần **không thực hành** trong lab này, kèm lý do:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài 116, 282: file `AdmissionConfiguration`, miễn trừ theo username, RuntimeClass, namespace | Phải truyền cho `kube-apiserver` qua cờ `--admission-control-config-file`. Lab cấm sửa cấu hình control plane — B0 ghi checksum, gate cuối đối chiếu. Thao tác thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) |
| Bài 116, 286: **nội dung** audit annotation của chế độ `audit` | Cần audit backend, tức sửa cờ apiserver. B1 kiểm phần quan sát được — không chặn, không cảnh báo, bộ đếm tăng. Phần đọc log nằm ở [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) |
| Bài 115: trường Windows, sandbox, policy engine bên thứ ba | Cluster chỉ có node Linux; sandbox cần RuntimeClass và runtime thay thế; policy engine là phần mềm ngoài lộ trình |
| Bài 291, 127: seccomp hoặc AppArmor profile `Localhost`, SELinux, `procMount: Unmasked`, `supplementalGroupsPolicy` | Profile tùy chỉnh phải nạp lên từng node; node Ubuntu chạy AppArmor nên không có SELinux để quan sát; `Unmasked` đòi user namespace; `supplementalGroupsPolicy` cần feature gate |
| Bài 258: bật sysctl không an toàn bằng cờ kubelet | Là sửa cấu hình kubelet. B7 ⭐ chứng minh mặt còn lại của cùng câu đó |
| Bài 173: dựng máy chủ webhook, `matchConditions`, kiểm tính lũy đẳng | Webhook sai phạm vi làm chết cả cluster. B10 chỉ đọc tồn kho và gửi một cấu hình webhook bằng **server dry-run** — không object nào được lưu |
| Bài 166: tạo hoặc sửa FlowSchema, PriorityLevelConfiguration; metric APF | Object phạm vi cluster chi phối mọi request; metric cần Prometheus ([giai đoạn 11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)) |
| Bài 122: sandbox, control plane ảo, service mesh, chặn tra cứu DNS liên namespace | Runtime thay thế, dự án bên thứ ba, hoặc sửa CoreDNS ([giai đoạn 21](../00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy)). Cách ly node đã làm ở [Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md) |
| Bài 121, 129: mã hóa Secret at rest — **[nợ lab #6](README.md#5-sổ-nợ-lab)** | Sửa cờ apiserver. B9 và B11 ghi nhận ô này **không đạt**; nợ trả ở giai đoạn 22 |
| Bài 123: OIDC, webhook xác thực, proxy xác thực | Cần hệ thống danh tính ngoài cluster và sửa cờ apiserver. B11 kiểm cơ chế nào đang bật |
| Bài 128: dựng static Pod ẩn, gọi API kubelet bằng credential, đọc etcd bằng client certificate | Là mô phỏng tấn công trên cluster dùng cho mọi lab sau. B11 kiểm **biện pháp giảm thiểu** — đúng phần quản trị viên phải rà |
| Bài 129, 130: quét image, ký image, digest `sha256`, thời hạn certificate | Thuộc pipeline và quy trình ngoài cluster; vòng đời certificate là [nợ lab #7](README.md#5-sổ-nợ-lab). B11 ghi các ô này là chưa đạt kèm lý do |
| Bài 254: chạy `cloud-controller-manager` | Không có cloud provider. B11 kiểm chứng sự vắng mặt và hệ quả |
| Bài 286: chuẩn hóa PSP, bỏ qua PSP, tắt admission PSP | API PodSecurityPolicy không còn tồn tại — B10 chứng minh |

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B6, B8–B12) khoảng 4 giờ 30 nếu không bí lâu ở ticket nào; B7 và các mục
đánh dấu ⭐ là tùy chọn. Có thể chia hai phiên: phiên 1 làm B0–B6 (Bookstore lên `restricted`), phiên 2
làm B7–B12 (tenant, rà soát, đóng sprint). Giữa hai phiên cứ để nguyên cluster; đầu phiên 2 chạy lại
gate ngắn A5.5, `. ~/lab-work/9b/ham.sh` và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác. Lệnh `sudo` đọc file của
  control plane chạy trên master; lệnh trên worker đi qua `ssh <node> 'sudo ...'` và **chỉ đọc**.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/9b/`. Bằng chứng: `~/lab-evidence/9b/`.
  Mọi script nghiệm thu bắt đầu bằng `. ~/lab-work/9b/ham.sh` (B0 tạo file này).
- **Namespace thử nghiệm** đặt tên `bookstore-thu*`, mang label dự án cộng
  `bookstore.example.com/thu: "true"` để `teardown.sh` dọn được nếu bạn quên. Object bên trong chúng
  **không** mang label `app.kubernetes.io/part-of`. `teardown.sh` xóa luôn chúng — ticket nào cần thì
  `kubectl apply -f ~/lab-work/9b/b1-namespaces.yaml` lại. Đóng sprint xóa hẳn.
- **Pod đặc quyền chỉ xuất hiện ở B7 ⭐**, trong `bookstore-thu`, ghim `lab-k8s-worker2` bằng
  `nodeSelector`, và bị xóa ngay trong bước tạo ra nó. Mọi phép thử khác với Pod vi phạm chính sách
  dùng **server dry-run** — request đi qua admission nhưng không object nào được lưu.
- **Không** sửa `/etc/kubernetes/manifests/*` hay `/var/lib/kubelet/config.yaml`: không bật audit
  log, không thêm `--admission-control-config-file`, không thêm `--allowed-unsafe-sysctls`. B0 ghi
  checksum, gate cuối đối chiếu.
- **Không cài admission webhook.** Mọi `ValidatingWebhookConfiguration` trong lab đi kèm
  `--dry-run=server`; gate cuối so số webhook với con số ở B0.
- **Không** đổi nhãn Pod Security của namespace nào ngoài `bookstore`, `bookstore-staging` và bốn
  namespace thử nghiệm. Lệnh `kubectl label` trên namespace khác luôn kèm `--dry-run=server`.
- **Vệ sinh bí mật** như Lab 3b: script không in mật khẩu, không chạy `set -x`, không ghi mật khẩu
  vào `~/lab-evidence/`.
- Lab chỉ dùng kiến thức tới giai đoạn 9. **Không** dùng metrics-server, HPA (giai đoạn 11), CRD,
  `ValidatingAdmissionPolicy`, webhook (giai đoạn 14).
- Đổi spec workload thì chạy `./scripts/deploy.sh`; dữ liệu `db` nằm trên PVC nên sống qua việc thay
  `db-0`. `teardown.sh` xóa luôn dữ liệu — đó là chủ đích, `deploy.sh` khởi tạo lại.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 16: Hardening

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `03-storage-ready`, repo ở tag `lab-9a`, chụp trạng thái
"trước" của những thứ lab **không được** đụng, rồi dựng lại hệ thống của sprint 15 **chỉ từ repo**.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Ba điểm riêng của mốc `03-storage-ready`:

```bash
kubectl get storageclass | grep -q '(default)' && echo 'PASS: co StorageClass mac dinh'
kubectl get pods -A -o name | grep -q 'calico-node' && echo 'PASS: Calico dang chay — NetworkPolicy co hieu luc'
kubectl -n traefik get deployment traefik >/dev/null 2>&1 && echo 'PASS: Traefik da cai'
```

Repo và thư mục làm việc. Nếu vừa restore snapshot, repo đã mất — khôi phục từ bundle theo
[mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) trước:

```bash
mkdir -p ~/lab-work/9b ~/lab-evidence/9b
test -d ~/bookstore/.git || git clone ~/bookstore-lab-9a.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-9a' && echo 'PASS: HEAD dung tag lab-9a'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
ls docs/adr/ | tail -3      # số ADR tiếp theo của bạn
```

File hàm dùng chung cho mọi script nghiệm thu của lab — chép nguyên văn. Mở shell mới thì
`. ~/lab-work/9b/ham.sh` lại:

```bash
cat > ~/lab-work/9b/ham.sh <<'EOF'
# ~/lab-work/9b/ham.sh — biến và hàm dùng chung cho script nghiệm thu Lab 9b.
NS=bookstore; TNS=bookstore-staging; THU=bookstore-thu
EV=~/lab-evidence/9b; WK=~/lab-work/9b; W2=lab-k8s-worker2
V="$(kubectl get --raw /version 2>/dev/null)"
PSA_VER="v$(echo "$V" | sed -n 's/.*"major": *"\([0-9]*\).*/\1/p').$(echo "$V" | sed -n 's/.*"minor": *"\([0-9]*\).*/\1/p')"
CI="system:serviceaccount:$NS:ci-deployer"
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
pod_of() {   # tên một Pod Running của thành phần $1 trong bookstore
  kubectl -n "$NS" get pod -l "app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=$1" \
    --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null; }
proc_val() { # proc_val <ns> <pod> <container> <Truong> — doc mot dong cua /proc/1/status
  kubectl -n "$1" exec "$2" -c "$3" -- sh -c "awk '/^$4:/{print \$2}' /proc/1/status" 2>/dev/null; }
EOF
. ~/lab-work/9b/ham.sh
echo "$PSA_VER" | grep -qE '^v[0-9]+\.[0-9]+$' && echo "PASS: minor cua server = $PSA_VER"
```

Chụp trạng thái "trước". Gate cuối so lại đúng các con số này:

```bash
. ~/lab-work/9b/ham.sh
{
  echo "apiserver $(sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml | awk '{print $1}')"
  echo "kubelet-lab-k8s-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for N in lab-k8s-worker1 lab-k8s-worker2; do
    echo "kubelet-$N $(ssh "$N" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
} > "$EV/b0-cauhinh-sha.txt"
kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.securityContext.privileged}{"\n"}{end}{end}' \
  | grep -c '^true$' > "$EV/b0-priv.txt" || true
kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations --no-headers 2>/dev/null \
  | wc -l > "$EV/b0-webhook.txt"
test "$(grep -c '^[a-z0-9-]* [0-9a-f]\{64\}$' "$EV/b0-cauhinh-sha.txt")" -eq 4 \
  && echo 'PASS: da ghi checksum cau hinh apiserver va kubelet ba node'
echo "container privileged truoc lab = $(cat "$EV/b0-priv.txt"); webhook = $(cat "$EV/b0-webhook.txt")"
```

Kéo sẵn image. `busybox` lên cả ba node vì `node-agent` chạy cả trên control plane; tag lấy đúng từ
[A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và
[A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00):

```bash
sudo crictl pull docker.io/library/busybox:1.37 >/dev/null && echo 'PASS: lab-k8s-master co busybox'
for N in lab-k8s-worker1 lab-k8s-worker2; do
  for I in docker.io/library/busybox:1.37 docker.io/library/postgres:17.11 \
           docker.io/postgrest/postgrest:v14.18 docker.io/nginxinc/nginx-unprivileged:1.30.5-alpine; do
    ssh "$N" "sudo crictl pull $I" >/dev/null && echo "PASS: $N co $I" || echo "FAIL: $N khong keo duoc $I"
  done
done
```

Dựng lại sprint 15 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/9b/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/9b/b0-smoke-lab-9a.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/9b/b0-smoke-lab-9a.txt
kubectl get namespace bookstore --show-labels
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; ba dòng `PASS:` của mốc `03-storage-ready`; ba dòng
`PASS:` về repo và cluster; `PASS: minor cua server`; `PASS:` của checksum; đủ chín dòng
`PASS: ... co ...`; `deploy exit=0`, `smoke exit=0` và file smoke không có dòng `FAIL:`. Nhãn của
namespace `bookstore` chưa có khóa `pod-security.kubernetes.io/*`.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 15 **chưa đủ**. Sửa repo ở tag `lab-9a`
trước, không sửa tay trên cluster ([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-9B-01] Điều tra: ba profile và ba chế độ trên namespace thử nghiệm

**Loại:** Điều tra.

**Bối cảnh:** trước khi dán nhãn lên namespace đang chạy hệ thống thật, đội cần biết chính xác mỗi
mức chặn cái gì và mỗi chế độ phản ứng ra sao — đặc biệt chỗ dễ nhầm nhất: một Deployment được API
server nhận "thành công" mà không đẻ ra nổi Pod nào. Thí nghiệm làm trên bốn namespace thử nghiệm,
không đụng `bookstore`.

**Yêu cầu:**

1. Bốn namespace thử nghiệm, tạo bằng **một** manifest `~/lab-work/9b/b1-namespaces.yaml`, mang label
   dự án và `bookstore.example.com/thu: "true"`:

   | Namespace | Cấu hình Pod Security |
   | --- | --- |
   | `bookstore-thu` | không nhãn nào |
   | `bookstore-thu-baseline` | chỉ `enforce` mức `baseline` |
   | `bookstore-thu-restricted` | chỉ `enforce` mức `restricted` — **không** `warn` |
   | `bookstore-thu-mix` | `enforce` mức `privileged`, `warn` mức `baseline`, `audit` mức `restricted` |

   Mọi chế độ được đặt đều ghim **phiên bản chính sách** bằng minor của server, không để `latest`.
2. Chứng minh bằng **server dry-run** — không tạo Pod vi phạm thật nào:
   - namespace không nhãn nhận một Pod vi phạm cả Baseline lẫn Restricted mà không một cảnh báo;
   - năm Pod, mỗi Pod vi phạm **đúng một** kiểm soát của Baseline, bị từ chối với năm lý do khác nhau;
   - một Pod không khai `securityContext` qua Baseline nhưng trượt Restricted vì bốn lý do;
   - một Pod viết đủ bốn ràng buộc được Restricted nhận.
3. Chứng minh `enforce` **không** áp cho workload resource: một Deployment không khai
   `securityContext` được tạo **thật** trong `bookstore-thu-restricted`, apply báo thành công, không
   cảnh báo, và không có Pod nào. Tìm ra chỗ Kubernetes ghi lý do.
4. Chứng minh `warn` **có** áp cho workload resource: dry-run một Deployment đặc quyền vào
   `bookstore-thu-mix` nhận cảnh báo ở mức `baseline` nhưng vẫn được nhận.
5. Chứng minh `audit` không chặn, không cảnh báo, nhưng **có** đánh giá: một Pod thường (không đặc
   quyền) tạo thật trong `bookstore-thu-mix` trên `lab-k8s-worker2` chạy được, và bộ đếm đánh giá của
   chế độ `audit` trên API server tăng.
6. Thử trước khi siết: hỏi API server điều gì xảy ra nếu `bookstore-thu-mix` lên `enforce: restricted`
   — nhận được tên Pod sẽ vi phạm — mà nhãn của namespace không đổi.

**Ràng buộc:** mọi Pod có `privileged`, `hostNetwork`, `hostPath` hay capability thêm vào chỉ đi qua
`--dry-run=server`. Pod thật duy nhất của ticket là Pod thường ở yêu cầu 5, ghim `lab-k8s-worker2`.
Không sửa nhãn namespace nào ngoài bốn namespace thử nghiệm.

**Đọc lại:**

- Bài [115](../115-pod-security-standards-vi.md):
  [*Chi tiết các profile*](../115-pod-security-standards-vi.md#chi-tiết-các-profile-profile-details) —
  ghi chú wildcard đứng trước hai bảng, bảng [Baseline](../115-pod-security-standards-vi.md#baseline) và
  [Restricted](../115-pod-security-standards-vi.md#restricted) dùng để **tra** câu từ chối; mục FAQ về
  security profile và security context.
- Bài [116](../116-pod-security-admission-vi.md): bảng chế độ ở mục
  [*Các label Pod Security Admission cho namespace*](../116-pod-security-admission-vi.md#pod-security-admission-labels-for-namespaces),
  [*Tài nguyên workload và Pod template*](../116-pod-security-admission-vi.md#workload-resources-and-pod-templates),
  [*Số liệu đo*](../116-pod-security-admission-vi.md#số-liệu-đo-metrics).
- Bài [282](../282-enforce-standards-admission-controller-vi.md#configure-the-admission-controller) —
  khối `defaults` cho namespace không nhãn.
- Bài [283](../283-enforce-standards-namespace-labels-vi.md): khuôn manifest
  `my-baseline-namespace`, mục
  [*Thêm nhãn vào các namespace hiện có*](../283-enforce-standards-namespace-labels-vi.md#thêm-nhãn-vào-các-namespace-hiện-có-bằng-kubectl-label-add-labels-to-existing-namespaces-with-kubectl-label)
  và selector `!pod-security.kubernetes.io/enforce`.

**Bàn giao:** `~/lab-work/9b/b1-*.yaml`; `docs/adr/NNNN-pod-security-standard.md`;
`~/lab-evidence/9b/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` bốn namespace đúng cấu hình, mang label dự án, version ghim bằng minor của server.
- `PASS:` namespace không nhãn nhận Pod vi phạm mà không cảnh báo, và nằm trong danh sách "chưa đánh
  giá tường minh".
- `PASS:` năm câu từ chối Baseline với năm lý do khác nhau, ghi đúng `baseline:<minor>`.
- `PASS:` cùng một Pod: Baseline nhận, Restricted từ chối với bốn lý do; Pod viết đủ ràng buộc được
  Restricted nhận.
- `PASS:` Deployment trong `bookstore-thu-restricted` được nhận không cảnh báo, có 0 Pod, ReplicaSet
  ghi lý do `violates PodSecurity`.
- `PASS:` dry-run Deployment đặc quyền vào `bookstore-thu-mix` nhận cảnh báo `would violate` mức
  `baseline` và vẫn được nhận.
- `PASS:` Pod thường trong `bookstore-thu-mix` chạy, không cảnh báo, bộ đếm `audit` tăng.
- `PASS:` dry-run nhãn `enforce=restricted` gọi tên Pod sẽ vi phạm; nhãn thật không đổi.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-pod-security-standard.md`:

1. Cuối sprint, `bookstore` chạy ở mức nào cho cả ba chế độ? Vì sao không dừng ở Baseline, dù
   Baseline nhận được hệ thống hiện tại mà không sửa gì? Còn `bookstore-system` của Lab 11a thì sao
   ([mục 3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật))?
2. Ở yêu cầu 3, CI sẽ báo xanh trong khi không có gì chạy. Vì sao khuôn manifest của bài 283 luôn đặt
   `warn` kèm `enforce`? Phân biệt `violates` với `would violate` khi đọc log CI.
3. Vì sao ghim nhãn version bằng minor của cluster thay cho `latest`? Điều gì xảy ra với lần deploy
   kế tiếp sau một lần nâng cấp cluster nếu để `latest`?
4. Pod Security Admission **không** sửa Pod (bài 286, mục 0). Vậy ai phải viết `securityContext` cho
   từng workload của Bookstore, và FAQ của bài 115 phân biệt security profile với security context
   thế nào?

<details>
<summary>Gợi ý</summary>

1. Ba nhãn mức và ba nhãn version dùng khóa `pod-security.kubernetes.io/<MODE>` và
   `pod-security.kubernetes.io/<MODE>-version`. Minor của server: `kubectl get --raw /version`, hoặc
   biến `$PSA_VER` trong `ham.sh`. Heredoc **không** nháy (`<<EOF`) để `$PSA_VER` được thay.
2. `kubectl apply --dry-run=server -f <file>`: request đi qua toàn bộ chuỗi admission, kết quả in ra
   như thật, nhưng không lưu object. Một file nhiều Pod cho một câu trả lời mỗi Pod.
3. Năm kiểm soát Baseline dễ dựng nhất: `hostNetwork`, `privileged`, volume `hostPath`, thêm
   capability `SYS_ADMIN`, `seccompProfile.type: Unconfined`.
4. Deployment không có Pod: nhìn **ReplicaSet** của nó, condition `ReplicaFailure`.
5. Bộ đếm của Pod Security: `kubectl get --raw /metrics | grep '^pod_security_evaluations_total'`.
6. Hỏi trước khi siết: `kubectl label --dry-run=server --overwrite ns <ns> pod-security.kubernetes.io/enforce=restricted`.
7. ⭐ Cùng lệnh đó với `ns --all` và mức `baseline` cho biết namespace nào của **cả cluster** sẽ có
   Pod gãy — vẫn là dry-run, không namespace nào bị đổi.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã viết đủ các file b1-*.yaml</summary>

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
. ~/lab-work/9b/ham.sh; cd "$WK"
lbl() { kubectl get ns "$1" -o jsonpath="{.metadata.labels.$2}" 2>/dev/null; }
PSL='pod-security\.kubernetes\.io'

kubectl apply -f b1-namespaces.yaml > "$EV/b1-namespaces.txt" 2>&1
BAD=0
for X in bookstore-thu: bookstore-thu-baseline:baseline bookstore-thu-restricted:restricted \
         bookstore-thu-mix:privileged; do
  N="${X%%:*}"; L="${X##*:}"
  [ "$(lbl "$N" "$PSL/enforce")" = "$L" ] || { echo "FAIL: $N enforce khong phai '$L'"; BAD=1; }
  [ "$(lbl "$N" 'app\.kubernetes\.io/part-of')" = 'bookstore' ] || { echo "FAIL: $N thieu label du an"; BAD=1; }
done
{ [ "$(lbl bookstore-thu-restricted "$PSL/enforce-version")" = "$PSA_VER" ] && \
  [ -z "$(lbl bookstore-thu-restricted "$PSL/warn")" ] && \
  [ "$(lbl bookstore-thu-mix "$PSL/warn")" = 'baseline' ] && \
  [ "$(lbl bookstore-thu-mix "$PSL/audit")" = 'restricted' ] && \
  [ "$(lbl bookstore-thu-mix "$PSL/audit-version")" = "$PSA_VER" ]; } || BAD=1
[ "$BAD" -eq 0 ] && echo "PASS: bon namespace dung cau hinh, version ghim $PSA_VER"

# 1. Namespace khong nhan: mac dinh privileged (bai 282)
kubectl -n bookstore-thu apply --dry-run=server -f b1-mac-dinh.yaml > "$EV/b1-mac-dinh.txt" 2>&1
{ grep -q 'created (server dry run)' "$EV/b1-mac-dinh.txt" && ! grep -qi 'violate' "$EV/b1-mac-dinh.txt"; } \
  && echo 'PASS: namespace khong nhan nhan Pod vi pham ma khong mot canh bao'
kubectl get ns --selector='!pod-security.kubernetes.io/enforce' -o name | grep -qx 'namespace/bookstore-thu' \
  && echo 'PASS: bookstore-thu nam trong nhom chua danh gia tuong minh'

# 2. Nam ranh gioi cua Baseline
kubectl -n bookstore-thu-baseline apply --dry-run=server -f b1-vi-pham-baseline.yaml \
  > "$EV/b1-vi-pham-baseline.txt" 2>&1
N=0
for K in 'host namespaces' 'securityContext.privileged' 'hostPath volumes' 'non-default capabilities' 'Unconfined'; do
  grep -q "$K" "$EV/b1-vi-pham-baseline.txt" && N=$((N + 1)) || echo "THIEU ly do: $K"
done
{ [ "$N" -eq 5 ] && \
  [ "$(grep -c "violates PodSecurity \"baseline:$PSA_VER\"" "$EV/b1-vi-pham-baseline.txt")" -eq 5 ]; } \
  && echo 'PASS: nam Pod, nam ly do Baseline khac nhau, dung muc va version da ghim'

# 3. Restricted them gi tren Baseline
kubectl -n bookstore-thu-baseline apply --dry-run=server -f b1-tran.yaml > "$EV/b1-tran-baseline.txt" 2>&1
kubectl -n bookstore-thu-restricted apply --dry-run=server -f b1-tran.yaml > "$EV/b1-tran-restricted.txt" 2>&1
grep -q 'created (server dry run)' "$EV/b1-tran-baseline.txt" && echo 'PASS: Pod khong khai gi dat Baseline'
N=0
for K in 'allowPrivilegeEscalation != false' 'unrestricted capabilities' 'runAsNonRoot != true' 'seccompProfile'; do
  grep -q "$K" "$EV/b1-tran-restricted.txt" && N=$((N + 1)) || echo "THIEU: $K"
done
[ "$N" -eq 4 ] && echo 'PASS: cung Pod do, Restricted tu choi vi bon ly do'
kubectl -n bookstore-thu-restricted apply --dry-run=server -f b1-dat-restricted.yaml \
  > "$EV/b1-dat-restricted.txt" 2>&1
grep -q 'created (server dry run)' "$EV/b1-dat-restricted.txt" && echo 'PASS: Pod viet du rang buoc duoc Restricted nhan'

# 4. enforce khong ap cho workload resource
kubectl -n bookstore-thu-restricted apply -f b1-deploy-tran.yaml > "$EV/b1-deploy-tran.txt" 2>&1
{ grep -Eq 'created|configured|unchanged' "$EV/b1-deploy-tran.txt" && ! grep -qi 'violate' "$EV/b1-deploy-tran.txt"; } \
  && echo 'PASS: Deployment duoc nhan, khong mot canh bao'
MSG=''
for i in $(seq 1 60); do
  MSG="$(kubectl -n bookstore-thu-restricted get rs -o jsonpath='{.items[*].status.conditions[?(@.type=="ReplicaFailure")].message}' 2>/dev/null)"
  [ -n "$MSG" ] && break; sleep 2
done
echo "$MSG" > "$EV/b1-replicafailure.txt"
{ echo "$MSG" | grep -q 'violates PodSecurity' && \
  [ "$(sl kubectl -n bookstore-thu-restricted get pods --no-headers)" -eq 0 ]; } \
  && echo 'PASS: 0 Pod; ly do nam o condition ReplicaFailure cua ReplicaSet'

# 5. warn co ap cho workload resource
kubectl -n bookstore-thu-mix apply --dry-run=server -f b1-deploy-dac-quyen.yaml > "$EV/b1-warn.txt" 2>&1
{ grep -q "would violate PodSecurity \"baseline:$PSA_VER\"" "$EV/b1-warn.txt" && \
  grep -q 'created (server dry run)' "$EV/b1-warn.txt"; } \
  && echo 'PASS: warn canh bao ngay luc apply mot Deployment, va van nhan'

# 6. audit: khong chan, khong canh bao, nhung co danh gia
aud() { kubectl get --raw /metrics \
  | awk '/^pod_security_evaluations_total\{/ && /mode="audit"/ && /decision="deny"/ {s+=$2} END {print s+0}'; }
kubectl -n bookstore-thu-mix delete pod thu-tran --ignore-not-found --wait=true >/dev/null
A0="$(aud)"
kubectl -n bookstore-thu-mix apply -f b1-tran.yaml > "$EV/b1-audit.txt" 2>&1
kubectl -n bookstore-thu-mix wait --for=condition=Ready pod/thu-tran --timeout=180s >/dev/null
A1="$(aud)"
{ grep -Eq 'created|configured|unchanged' "$EV/b1-audit.txt" && ! grep -qi 'violate' "$EV/b1-audit.txt"; } \
  && echo 'PASS: apply thanh cong, audit khong canh bao'
{ [ "$(kubectl -n bookstore-thu-mix get pod thu-tran -o jsonpath='{.status.phase}')" = 'Running' ] && \
  [ "$(kubectl -n bookstore-thu-mix get pod thu-tran -o jsonpath='{.spec.nodeName}')" = "$W2" ]; } \
  && echo 'PASS: audit khong chan — Pod chay tren lab-k8s-worker2'
[ "$A1" -gt "$A0" ] && echo "PASS: bo dem audit/deny tang $A0 -> $A1"

# 7. Hoi truoc khi siet
kubectl label --dry-run=server --overwrite ns bookstore-thu-mix pod-security.kubernetes.io/enforce=restricted \
  > "$EV/b1-label-dry-run.txt" 2>&1
{ grep -q 'violate the new PodSecurity enforce level' "$EV/b1-label-dry-run.txt" && \
  grep -q 'thu-tran' "$EV/b1-label-dry-run.txt" && \
  [ "$(lbl bookstore-thu-mix "$PSL/enforce")" = 'privileged' ]; } \
  && echo 'PASS: dry-run goi ten Pod se vi pham; nhan that khong doi'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
. ~/lab-work/9b/ham.sh
NSL='app.kubernetes.io/part-of: bookstore, bookstore.example.com/thu: "true"'
PS='pod-security.kubernetes.io'
cat > ~/lab-work/9b/b1-namespaces.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata: {name: bookstore-thu, labels: {$NSL}}
---
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore-thu-baseline
  labels: {$NSL, $PS/enforce: baseline, $PS/enforce-version: $PSA_VER}
---
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore-thu-restricted
  labels: {$NSL, $PS/enforce: restricted, $PS/enforce-version: $PSA_VER}
---
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore-thu-mix
  labels: {$NSL, $PS/enforce: privileged, $PS/enforce-version: $PSA_VER,
           $PS/warn: baseline, $PS/warn-version: $PSA_VER,
           $PS/audit: restricted, $PS/audit-version: $PSA_VER}
EOF
cat ~/lab-work/9b/b1-namespaces.yaml      # kiểm $PSA_VER đã được thay bằng minor thật
```

Các Pod và Deployment thử nghiệm, mọi file ghim `lab-k8s-worker2`:

```yaml
# ~/lab-work/9b/b1-mac-dinh.yaml — vi phạm cả Baseline lẫn Restricted; chỉ dùng với --dry-run=server
apiVersion: v1
kind: Pod
metadata: {name: thu-mac-dinh}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  containers:
  - name: c
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      seccompProfile: {type: Unconfined}
      capabilities: {add: ["SYS_ADMIN"]}
```

```yaml
# ~/lab-work/9b/b1-vi-pham-baseline.yaml — năm Pod, mỗi Pod đúng một vi phạm; chỉ dùng với --dry-run=server
apiVersion: v1
kind: Pod
metadata: {name: vp-hostnet}
spec:
  hostNetwork: true
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  containers: [{name: c, image: busybox:1.37, command: ["sleep", "3600"]}]
---
apiVersion: v1
kind: Pod
metadata: {name: vp-privileged}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  containers:
  - {name: c, image: busybox:1.37, command: ["sleep", "3600"], securityContext: {privileged: true}}
---
apiVersion: v1
kind: Pod
metadata: {name: vp-hostpath}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  volumes: [{name: tmp-cua-node, hostPath: {path: /tmp, type: Directory}}]
  containers:
  - name: c
    image: busybox:1.37
    command: ["sleep", "3600"]
    volumeMounts: [{name: tmp-cua-node, mountPath: /host-tmp, readOnly: true}]
---
apiVersion: v1
kind: Pod
metadata: {name: vp-cap}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  containers:
  - {name: c, image: busybox:1.37, command: ["sleep", "3600"], securityContext: {capabilities: {add: ["SYS_ADMIN"]}}}
---
apiVersion: v1
kind: Pod
metadata: {name: vp-seccomp}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  containers:
  - {name: c, image: busybox:1.37, command: ["sleep", "3600"], securityContext: {seccompProfile: {type: Unconfined}}}
```

```yaml
# ~/lab-work/9b/b1-tran.yaml — không khai securityContext
apiVersion: v1
kind: Pod
metadata: {name: thu-tran}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  containers: [{name: c, image: busybox:1.37, command: ["sh", "-c", "sleep 3600"]}]
```

```yaml
# ~/lab-work/9b/b1-dat-restricted.yaml — đủ bốn ràng buộc Restricted thêm vào
apiVersion: v1
kind: Pod
metadata: {name: thu-dat-restricted}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    runAsGroup: 65534
    seccompProfile: {type: RuntimeDefault}
  containers:
  - name: c
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities: {drop: ["ALL"]}
```

```yaml
# ~/lab-work/9b/b1-deploy-tran.yaml — b1-deploy-dac-quyen.yaml giống hệt, đổi tên thành
# thu-dac-quyen (cả label app) và thêm securityContext: {privileged: true} cho container
apiVersion: apps/v1
kind: Deployment
metadata: {name: thu-tran}
spec:
  replicas: 2
  selector: {matchLabels: {app: thu-tran}}
  template:
    metadata: {labels: {app: thu-tran}}
    spec:
      nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
      containers: [{name: c, image: busybox:1.37, command: ["sh", "-c", "sleep 3600"]}]
```

Ý chính cho ADR:

1. `enforce`, `warn`, `audit` đều `restricted`, ghim version. Baseline chỉ chặn các cách leo thang
   **đã biết**; container mặc định vẫn chạy root, giữ một tập capability và cho phép leo thang —
   đúng vấn đề `P7` của hệ thống cũ. `bookstore-system` là `privileged` vì `node-agent` cần
   `hostPath`; namespace riêng để mức thấp không lan sang ứng dụng.
2. `enforce` chỉ áp cho Pod object; `audit` và `warn` áp cả cho workload resource. Không có `warn`
   thì lỗi chỉ hiện ở condition `ReplicaFailure` của ReplicaSet — không ai đọc trong CI. `violates`
   là **đã bị chặn**; `would violate` là **sẽ bị chặn nếu siết**.
3. Nội dung một mức có thể chặt hơn ở minor mới. Để `latest` thì sau nâng cấp, lần deploy kế tiếp bị
   từ chối dù không ai đổi manifest. Ghim version biến việc nâng mức thành một thay đổi có chủ đích.
4. Người viết manifest. PSA chỉ validate; security context là tham số của Pod truyền cho runtime,
   security profile là cơ chế control plane kiểm các tham số đó (FAQ bài 115).

</details>

## B2. [BS-9B-02] Bật `warn` và `audit` cho `bookstore`, kiểm kê vi phạm

**Loại:** Tính năng.

**Bối cảnh:** siết thẳng `enforce: restricted` lên một namespace đang chạy là đánh cược. Bài 283 và
bước 3 của bài 286 dạy cách an toàn: `enforce` ở mức **chắc chắn đạt hôm nay**, `warn` và `audit` ở
mức **mong muốn**, kiểm kê mọi vi phạm rồi mới sửa. Kiểm kê phải đủ — kể cả workload không có Pod
nào đang chạy lúc bạn nhìn.

**Yêu cầu:**

1. Namespace `bookstore` khai trong `k8s/app/00-namespace.yaml` mang thêm sáu nhãn Pod Security:
   `enforce` mức `baseline`, `warn` và `audit` mức `restricted`, cả ba ghim version bằng minor của
   server. Trước khi đặt `enforce: baseline`, có bằng chứng API server **không** cảnh báo Pod nào của
   hệ thống hiện tại ở mức đó.
2. Triển khai lại từ đầu (`teardown` rồi `deploy`) và giữ lại **toàn bộ** cảnh báo mà API server trả
   về trong lúc tạo object.
3. `docs/pss-kiem-ke.md` — bảng kiểm kê: mỗi workload, mỗi container, vi phạm Restricted nào, cách sửa,
   ticket sửa (B3, B4 hoặc B5). Bảng phải có cả `db-backup`.
4. Hệ thống vẫn chạy đủ như sprint 15 — `tests/smoke.sh` của tag `lab-9a` vẫn PASS.

**Ràng buộc:** nhãn nằm trong manifest của repo, không đặt bằng `kubectl label` trên namespace thật.
Chưa sửa workload nào ở ticket này.

**Đọc lại:**

- Bài [283](../283-enforce-standards-namespace-labels-vi.md):
  [*Yêu cầu chuẩn `baseline` bằng nhãn namespace*](../283-enforce-standards-namespace-labels-vi.md#yêu-cầu-chuẩn-pod-security-standard-baseline-bằng-nhãn-namespace-requiring-the-baseline-pod-security-standard-with-namespace-labels)
  — chú thích "mức `enforce` mà mình *mong muốn*".
- Bài [286](../286-migrate-from-psp-vi.md):
  [*3.a Xác định mức phù hợp*](../286-migrate-from-psp-vi.md#identify-appropriate-level) — đoạn
  **Thận trọng** về workload hiện không chạy;
  [*3.b Kiểm chứng mức*](../286-migrate-from-psp-vi.md#verify-pss-level).
- Bài [116](../116-pod-security-admission-vi.md#workload-resources-and-pod-templates) — vì sao `warn`
  thấy được CronJob.

**Bàn giao:** `k8s/app/00-namespace.yaml`; `docs/pss-kiem-ke.md`; `docs/adr/NNNN-pss-lo-trinh-siet.md`;
`~/lab-evidence/9b/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` `bookstore` mang đúng sáu nhãn; ba nhãn version bằng minor của server.
- `PASS:` `enforce: baseline` đang có hiệu lực: Pod đặc quyền bị từ chối ở mức `baseline`.
- `PASS:` lần triển khai mới sinh ít nhất năm cảnh báo `would violate` mức `restricted`, nhắc tới các container `postgres`, `postgrest`, `nginx`, `status-writer`.
- `PASS:` dry-run `enforce=restricted` trên `bookstore` gọi tên Pod đang chạy sẽ vi phạm.
- `PASS:` `docs/pss-kiem-ke.md` nhắc đủ `db`, `api`, `web`, `node-agent`, `db-migrate`, `db-backup`.
- `PASS:` `tests/smoke.sh` hiện có vẫn PASS.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-pss-lo-trinh-siet.md`:

1. Dry-run nhãn trên namespace nhìn thấy những Pod nào, và **không** nhìn thấy gì của Bookstore? Bạn
   phát hiện vi phạm của `db-backup` bằng đường nào?
2. Vì sao `enforce` ở bước này là `baseline` mà không bỏ trống? Nó bảo vệ được gì ngay hôm nay?
3. Theo bài 286, `audit` có ưu điểm gì so với dry-run mà `warn` không có? Cluster này đọc được phần
   nào của `audit`?

<details>
<summary>Gợi ý</summary>

1. Hỏi trước: `kubectl label --dry-run=server --overwrite ns bookstore pod-security.kubernetes.io/enforce=baseline`.
   Không có dòng `violate` nào là an toàn để đặt.
2. `kubectl apply` gặp object không đổi thì không gửi request — không có cảnh báo. Tạo mới từ đầu thì
   mọi object đều đi qua admission: `./scripts/teardown.sh && ./scripts/deploy.sh > file 2>&1`.
3. Cảnh báo in ra **trước** dòng `... created` của đúng object gây ra nó, và nêu tên container.
4. CronJob không có Pod nào cho tới lần chạy kế tiếp.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
. ~/lab-work/9b/ham.sh; cd ~/bookstore
lbl() { kubectl get ns "$NS" -o jsonpath="{.metadata.labels.pod-security\.kubernetes\.io/$1}" 2>/dev/null; }

./scripts/teardown.sh > "$EV/b2-teardown.txt" 2>&1
./scripts/deploy.sh > "$EV/b2-deploy.txt" 2>&1; echo "deploy exit=$?"
{ [ "$(lbl enforce)" = 'baseline' ] && [ "$(lbl warn)" = 'restricted' ] && [ "$(lbl audit)" = 'restricted' ] && \
  [ "$(lbl enforce-version)" = "$PSA_VER" ] && [ "$(lbl warn-version)" = "$PSA_VER" ] && \
  [ "$(lbl audit-version)" = "$PSA_VER" ]; } && echo "PASS: bookstore mang du sau nhan, version $PSA_VER"

printf '%s\n' 'apiVersion: v1' 'kind: Pod' 'metadata: {name: thu-dac-quyen}' 'spec:' \
  '  containers: [{name: c, image: busybox:1.37, securityContext: {privileged: true}}]' \
  | kubectl -n "$NS" create --dry-run=server -f - > "$EV/b2-enforce-baseline.txt" 2>&1
grep -q "violates PodSecurity \"baseline:$PSA_VER\"" "$EV/b2-enforce-baseline.txt" \
  && echo 'PASS: enforce baseline dang co hieu luc'

W="$(grep -c "would violate PodSecurity \"restricted:$PSA_VER\"" "$EV/b2-deploy.txt")"
BAD=0
for C in postgres postgrest nginx status-writer; do grep -q "\"$C\"" "$EV/b2-deploy.txt" || BAD=1; done
{ [ "$W" -ge 5 ] && [ "$BAD" -eq 0 ]; } && echo "PASS: $W canh bao restricted luc tao workload"

kubectl label --dry-run=server --overwrite ns "$NS" pod-security.kubernetes.io/enforce=restricted \
  > "$EV/b2-label-dry-run.txt" 2>&1
{ grep -q 'violate the new PodSecurity enforce level' "$EV/b2-label-dry-run.txt" && \
  grep -q 'db-0' "$EV/b2-label-dry-run.txt"; } && echo 'PASS: dry-run goi ten Pod dang chay se vi pham'

BAD=0
for X in db api web node-agent db-migrate db-backup; do grep -q "$X" docs/pss-kiem-ke.md || { echo "FAIL: kiem ke thieu $X"; BAD=1; }; done
[ "$BAD" -eq 0 ] && echo 'PASS: bang kiem ke du sau workload'
./tests/smoke.sh > "$EV/b2-smoke.txt" 2>&1 && echo 'PASS: smoke cua lab-9a van PASS'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/00-namespace.yaml — sau B2
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    # Pod Security (9b): enforce mức đạt được hôm nay, warn/audit mức mong muốn (bài 283).
    # v1.35 là minor của cluster — đối chiếu bảng A1.3; script nghiệm thu so với /version.
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: v1.35
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.35
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.35
```

```bash
cd ~/bookstore
kubectl label --dry-run=server --overwrite ns bookstore pod-security.kubernetes.io/enforce=baseline \
  | tee ~/lab-evidence/9b/b2-dry-run-baseline.txt
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/9b/b2-deploy.txt 2>&1
grep -A1 'would violate' ~/lab-evidence/9b/b2-deploy.txt
```

Khung `docs/pss-kiem-ke.md` — điền từ cảnh báo thật của bạn:

```markdown
# Kiểm kê vi phạm Pod Security "restricted" — Lab 9b, BS-9B-02

| Workload | Container | Vi phạm | Cách sửa | Ticket |
| --- | --- | --- | --- | --- |
| StatefulSet db | postgres | runAsNonRoot, allowPrivilegeEscalation, capabilities, seccomp | uid 999, PGDATA thư mục con, fsGroup | B3 |
| Deployment api | postgrest (và init chờ DB nếu còn) | capabilities, seccomp; init chạy image `postgres` thêm runAsNonRoot, allowPrivilegeEscalation | securityContext cấp Pod và container | B4 |
| Deployment web | render-html, status-writer, nginx | allowPrivilegeEscalation, capabilities, seccomp | như trên, thêm /tmp cho nginx | B4 |
| DaemonSet node-agent | agent | capabilities, seccomp | như trên | B5 |
| Job db-migrate | migrate | capabilities, seccomp — uid 999 và allowPrivilegeEscalation đã có từ Lab 9a | drop ALL, seccomp | B5 |
| CronJob db-backup | pg-dump (init), report-status | như db-migrate — **không** hiện trong dry-run nhãn | drop ALL, seccomp | B5 |
```

Ý chính cho ADR:

1. Dry-run nhãn chỉ đánh giá Pod **đang tồn tại**: `db-0`, Pod của `api`, `web`, `node-agent`. CronJob
   `db-backup` không có Pod nào giữa hai lần chạy — đúng chỗ bài 286 cảnh báo. `warn` bắt được nó vì
   áp cho cả workload resource, lúc CronJob được tạo.
2. Không có nhãn `enforce` thì namespace ở mặc định `privileged` (bài 282). `baseline` đã chặn ngay
   Pod đặc quyền, `hostPath`, namespace của host — mà hệ thống hiện tại không gãy gì.
3. `audit` ghi vi phạm vào audit log trong cả một khoảng theo dõi, nên bắt được workload chỉ chạy
   thỉnh thoảng; `warn` chỉ báo cho người đang gõ lệnh. Cluster này không bật audit backend, nên chỉ
   đọc được bộ đếm `pod_security_evaluations_total`, không đọc được annotation — giai đoạn 22.

</details>

## B3. [BS-9B-03] `db` chạy non-root — trả `TD-06`

**Loại:** Tính năng.

**Bối cảnh:** từ Lab 3a, `db` là thành phần duy nhất khởi động bằng root: entrypoint của image
`postgres` chạy root để chỉnh quyền thư mục dữ liệu rồi mới hạ xuống uid 999
([mục 4.4](DU-AN-BOOKSTORE.md#44-image)). Lab 3a ghi điều đó thành `TD-06` và hẹn trả ở sprint này.
Từ Lab 6a dữ liệu nằm trên PVC, nên lời giải phải đúng với **cả hai** trường hợp: volume rỗng lần
đầu, và volume đã có dữ liệu của sprint trước.

**Yêu cầu:**

1. Mọi tiến trình trong Pod `db-0` chạy uid 999, gid 999 ngay từ đầu; kubelet phải **kiểm** điều đó
   chứ không chỉ tin image.
2. Thư mục dữ liệu PostgreSQL là **thư mục con** của volume `data`, thuộc 999 và có quyền `0700`.
   `deploy.sh` của bạn đặt `PGDATA` thư mục con từ Lab 6a thì giữ nguyên.
3. Mật khẩu superuser vẫn đi qua `POSTGRES_PASSWORD_FILE` từ volume Secret với `defaultMode: 0400`.
   Tiến trình uid 999 đọc được file đó; "others" vẫn không có quyền gì.
4. Container `postgres` không leo thang đặc quyền, không giữ capability nào, chạy seccomp profile
   mặc định của runtime.
5. Khởi tạo từ volume rỗng thành công — schema, dữ liệu, role `authenticator`, `api` Ready — và log
   của lần khởi tạo cho thấy entrypoint **không** phải đổi quyền sở hữu file nào.
6. Chuyển đổi trên dữ liệu **đang có** cũng thành công: áp manifest mới bằng `deploy.sh` (không
   teardown), `db-0` được thay, dữ liệu giữ nguyên.
7. `docs/tech-debt.md`: `TD-06` đổi thành `đã trả (9b)`, trỏ tới bằng chứng.

**Ràng buộc:** không `privileged`, không thêm capability nào — kể cả `CHOWN`, `FOWNER`,
`DAC_OVERRIDE`. Không init container chạy root để `chown`. Không đổi image, tên container, cổng.
Không dùng `hostUsers: false` cho `db`.

**Đọc lại:**

- Bài [291](../291-security-context-vi.md#set-the-security-context-for-a-pod) — đoạn giải thích
  `runAsUser`, `runAsGroup` (bỏ trống thì gid là 0), `fsGroup` sở hữu volume và file trong volume;
  mục *Cấu hình chính sách thay đổi quyền và quyền sở hữu volume cho Pod* — câu "chỉ áp dụng cho các
  loại volume hỗ trợ kiểm soát quyền sở hữu".
- Bài [127](../127-linux-kernel-security-vi.md#run-without-root) — đoạn **Thận trọng** về user không
  đủ quyền với file.
- Bài [115](../115-pod-security-standards-vi.md#restricted) — *Running as Non-root*, *Running as
  Non-root user*.
- [Mục 4.4 của file dự án](DU-AN-BOOKSTORE.md#44-image); tài liệu image `postgres` trên Docker Hub,
  mục *Arbitrary `--user` Notes*.

**Bàn giao:** `k8s/app/db/statefulset.yaml`; `docs/adr/NNNN-db-non-root.md`; `docs/tech-debt.md`;
`~/lab-evidence/9b/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` PVC `data-db-0` cũ hơn Pod `db-0` hiện tại — Pod đã được thay trên dữ liệu đang có.
- `PASS:` `runAsNonRoot: true`, `runAsUser: 999` có hiệu lực cho Pod `db-0`.
- `PASS:` tiến trình 1 của `postgres`: uid 999, gid 999, `CapEff` bằng 0, `NoNewPrivs` bằng 1,
  `Seccomp` bằng 2.
- `PASS:` thư mục dữ liệu là thư mục con của mount `data`, thuộc 999, quyền `700`.
- `PASS:` uid 999 đọc được file mật khẩu; "others" không có quyền.
- `PASS:` sau teardown và deploy, khởi tạo từ volume rỗng xong, log không có lỗi quyền hay `chown`.
- `PASS:` `api.books` ít nhất 8 dòng; `tests/smoke.sh` hiện có vẫn PASS.
- `PASS:` dry-run `enforce=restricted` không còn gọi tên `db-0`.
- `PASS:` `docs/tech-debt.md` ghi `TD-06` đã trả.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-db-non-root.md`:

1. Vì sao `PGDATA` phải là thư mục **con** của volume? Thư mục gốc của volume do ai tạo, thuộc ai, quyền
   gì? Điều gì xảy ra với `initdb` nếu `PGDATA` trỏ thẳng vào gốc volume?
2. Thiếu `fsGroup` thì container `postgres` chết ở đâu, vì sao? `fsGroup` đổi gì trên volume Secret,
   và vì sao nó **không** đụng tới thư mục dữ liệu trên PV `local-path`?
3. Bằng chứng nào cho thấy entrypoint không cần `chown` khi chạy non-root? Đoạn nào trong
   `docker-entrypoint.sh` quyết định điều đó?
4. Yêu cầu 6 chạy được vì dữ liệu cũ thuộc ai? Nếu một ngày bạn nạp lại một thư mục dữ liệu do root
   tạo, quy trình đúng là gì?

<details>
<summary>Gợi ý</summary>

1. Đọc trạng thái hiện tại trước khi sửa:
   `kubectl -n bookstore exec db-0 -c postgres -- sh -c 'id; stat -c "%U %a %n" /var/lib/postgresql/data /var/lib/postgresql/data/*'`.
2. `runAsNonRoot: true` với image không khai `USER` mà thiếu `runAsUser` thì kubelet từ chối khởi
   động container (Lab 3a, B9).
3. Thử bỏ `fsGroup` rồi đọc `kubectl logs db-0 -c postgres --previous`: entrypoint đọc
   `POSTGRES_PASSWORD_FILE` bằng chính uid của tiến trình.
4. `kubectl exec db-0 -c postgres -- psql -U postgres -tAc 'SHOW data_directory'` cho đường dẫn thật.
5. Đọc kernel: `proc_val bookstore db-0 postgres CapEff` (hàm trong `ham.sh`).

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã áp manifest mới bằng deploy.sh; script teardown rồi deploy lại ở cuối</summary>

```bash
. ~/lab-work/9b/ham.sh; cd ~/bookstore
ts() { date -d "$1" +%s; }

# Chuyển đổi trên dữ liệu đang có
{ [ "$(ts "$(jp pvc/data-db-0 '{.metadata.creationTimestamp}')")" -lt \
    "$(ts "$(jp pod/db-0 '{.metadata.creationTimestamp}')")" ] && \
  [ "$(jp pod/db-0 '{.status.conditions[?(@.type=="Ready")].status}')" = 'True' ]; } \
  && echo 'PASS: db-0 duoc thay tren PVC cu va Ready'

check_db() {   # $1 = nhan cua lan kiem
  { jp pod/db-0 '{.spec.securityContext.runAsNonRoot}{.spec.containers[?(@.name=="postgres")].securityContext.runAsNonRoot}' \
      | grep -q true && \
    [ "$(jp pod/db-0 '{.spec.securityContext.runAsUser}{.spec.containers[?(@.name=="postgres")].securityContext.runAsUser}' | cut -c1-3)" = '999' ]; } \
    && echo "PASS: [$1] runAsNonRoot va runAsUser 999"
  U="$(proc_val "$NS" db-0 postgres Uid)"; G="$(proc_val "$NS" db-0 postgres Gid)"
  C="$(proc_val "$NS" db-0 postgres CapEff)"; P="$(proc_val "$NS" db-0 postgres NoNewPrivs)"
  S="$(proc_val "$NS" db-0 postgres Seccomp)"
  echo "[$1] uid=$U gid=$G CapEff=$C NoNewPrivs=$P Seccomp=$S" | tee -a "$EV/b3-kernel.txt"
  { [ "$U" = 999 ] && [ "$G" = 999 ] && [ "$((0x${C:-1}))" -eq 0 ] && [ "$P" = 1 ] && [ "$S" = 2 ]; } \
    && echo "PASS: [$1] tien trinh 1 uid/gid 999, khong capability, no_new_privs, seccomp filter"
  MP="$(jp pod/db-0 '{.spec.containers[?(@.name=="postgres")].volumeMounts[?(@.name=="data")].mountPath}')"
  DD="$(kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -tAc 'SHOW data_directory' 2>/dev/null)"
  ST="$(kubectl -n "$NS" exec db-0 -c postgres -- stat -c '%u %a' "$DD" 2>/dev/null)"
  RT="$(kubectl -n "$NS" exec db-0 -c postgres -- stat -c '%u %a' "$MP" 2>/dev/null)"
  echo "[$1] mount=$MP ($RT) data_directory=$DD ($ST)" | tee -a "$EV/b3-pgdata.txt"
  { case "$DD" in "$MP"/?*) true ;; *) false ;; esac && [ "$ST" = '999 700' ]; } \
    && echo "PASS: [$1] PGDATA la thu muc con cua volume, thuoc 999, quyen 700"
  F="$(jp pod/db-0 '{.spec.containers[?(@.name=="postgres")].env[?(@.name=="POSTGRES_PASSWORD_FILE")].value}')"
  M="$(kubectl -n "$NS" exec db-0 -c postgres -- stat -L -c '%a' "$F" 2>/dev/null)"
  { kubectl -n "$NS" exec db-0 -c postgres -- test -r "$F" && [ "${M#??}" = '0' ]; } \
    && echo "PASS: [$1] uid 999 doc duoc file mat khau (quyen $M), others khong co quyen"
}
check_db 'du lieu cu'

# Khoi tao tu volume rong
./scripts/teardown.sh > "$EV/b3-teardown.txt" 2>&1
./scripts/deploy.sh > "$EV/b3-deploy.txt" 2>&1; echo "deploy exit=$?"
kubectl -n "$NS" logs db-0 -c postgres > "$EV/b3-init.log" 2>&1
{ grep -q 'PostgreSQL init process complete' "$EV/b3-init.log" && \
  ! grep -Eqi 'chown|operation not permitted|permission denied' "$EV/b3-init.log"; } \
  && echo 'PASS: khoi tao tu volume rong thanh cong, khong loi quyen, khong chown'
check_db 'volume moi'
N="$(kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && echo "PASS: api.books co $N dong"
./tests/smoke.sh > "$EV/b3-smoke.txt" 2>&1 && echo 'PASS: smoke hien co van PASS'
# "Không gọi tên db-0" chỉ có nghĩa khi lệnh dry-run thành công.
if kubectl label --dry-run=server --overwrite ns "$NS" pod-security.kubernetes.io/enforce=restricted \
     > "$EV/b3-label-dry-run.txt" 2>&1; then
  ! grep -q 'db-0' "$EV/b3-label-dry-run.txt" && echo 'PASS: dry-run restricted khong con goi ten db-0'
else
  echo "FAIL: lenh dry-run loi, chua ket luan duoc — xem $EV/b3-label-dry-run.txt"
fi
grep -E 'TD-06.*đã trả' docs/tech-debt.md >/dev/null && echo 'PASS: tech-debt.md ghi TD-06 da tra'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Phần thay đổi của StatefulSet, đặt trong `spec.template.spec`. Dòng không có chú thích `# 9b` là bối
cảnh từ các sprint trước — **giữ đúng bản trong repo của bạn** (đường dẫn `PGDATA`, tên volume có thể
khác).

```yaml
# k8s/app/db/statefulset.yaml — spec.template.spec, sau Lab 9b
      securityContext:                         # 9b
        runAsNonRoot: true                     # 9b: kubelet kiểm
        runAsUser: 999                         # 9b: user postgres của image
        runAsGroup: 999                        # 9b: bỏ trống thì gid là 0
        fsGroup: 999                           # 9b: file của volume Secret, ConfigMap thuộc group 999
        seccompProfile: {type: RuntimeDefault} # 9b
      volumes:
      - name: credentials                      # 3b, giữ nguyên
        secret:
          secretName: db-credentials
          defaultMode: 0400
          items: [{key: postgres-password, path: postgres-password}]
      containers:
      - name: postgres
        env:
        - {name: POSTGRES_PASSWORD_FILE, value: /etc/db-credentials/postgres-password}   # 3b
        - {name: PGDATA, value: /var/lib/postgresql/data/pgdata}   # 6a: thư mục con của volume data
        # ... các biến khác, args, probe, resources giữ nguyên
        securityContext:                       # 9b
          allowPrivilegeEscalation: false      # 9b
          capabilities: {drop: ["ALL"]}        # 9b
        volumeMounts:
        - {name: credentials, mountPath: /etc/db-credentials, readOnly: true}
        - {name: data, mountPath: /var/lib/postgresql/data}      # PVC từ volumeClaimTemplates (6a)
        # ... initdb giữ nguyên
```

```bash
cd ~/bookstore
./scripts/deploy.sh                       # yêu cầu 6: thay db-0 trên dữ liệu đang có
kubectl -n bookstore exec db-0 -c postgres -- id
sed -i 's/^\(| *TD-06 .*| *\)mở\( *|.*\)$/\1đã trả (9b)\2/' docs/tech-debt.md   # hoặc sửa tay
```

Ý chính cho ADR:

1. local-path-provisioner tạo thư mục gốc của volume bằng `mkdir -m 0777`, thuộc root. `initdb` đòi
   thư mục dữ liệu `0700` thuộc chính nó; uid 999 không `chmod` được thư mục của root nên `initdb`
   gãy nếu `PGDATA` là gốc volume. Thư mục con thì do chính uid 999 tạo — đúng chủ, đúng quyền.
2. File trong volume Secret thuộc root; `0400` chỉ cho root đọc, nên entrypoint chết ngay ở bước đọc
   `POSTGRES_PASSWORD_FILE` với `Permission denied`. `fsGroup` làm kubelet gán group 999 và thêm quyền
   đọc cho group. PV `local-path` mặc định là loại `hostPath`, không thuộc nhóm volume mà kubelet quản
   quyền sở hữu (bài 291), nên `PGDATA` giữ nguyên `0700` — PostgreSQL từ chối thư mục dữ liệu có
   quyền ghi cho group.
3. Entrypoint chỉ chạy nhánh `find ... -exec chown` khi `id -u` bằng 0. Bằng chứng: `CapEff` bằng 0 —
   tiến trình **không thể** `chown` — mà khởi tạo vẫn xong và log không có lỗi quyền.
4. `initdb` của các sprint trước đã chạy dưới `gosu postgres`, nên file dữ liệu vốn thuộc 999. Dữ
   liệu do root tạo thì phải `chown -R 999:999` một lần ngoài luồng (Pod thử nghiệm có quyền, hoặc
   trên node) trước khi chuyển — ghi vào runbook, không đưa capability vào `db`.

</details>

## B4. [BS-9B-04] `web` và `api` đạt `restricted`, root filesystem chỉ đọc

**Loại:** Tính năng.

**Bối cảnh:** `web` và `api` đã chạy non-root từ Lab 3a, nhưng vẫn giữ capability mặc định của
runtime, không khai seccomp, và init container, sidecar chưa khai gì. Checklist 130 còn đòi thêm một
thứ Restricted **không** đòi: root filesystem chỉ đọc — kẻ chiếm được tiến trình không cài thêm được
gì vào image.

**Yêu cầu:**

1. Mọi container của Pod `web` (`render-html`, `status-writer`, `nginx`) và Pod `api` (`postgrest`, và
   init container chờ DB nếu bạn còn giữ) đạt Restricted.
2. `nginx` và `postgrest` chạy với root filesystem **chỉ đọc**. Mọi chỗ `nginx` cần ghi nằm trên một
   volume tạm riêng.
3. Một container gỡ lỗi gắn vào Pod `api` về sau (B6) phải tự động chạy cùng uid không phải root với
   `postgrest` mà không cần khai thêm.
4. Hợp đồng giữ nguyên ([mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe)):
   `/`, `/healthz`, `/status.json`, `/api/books` của `web`; `/books`, `/live`, `/ready` của `api`.

**Ràng buộc:** không đổi image, cổng, tên container. Không tắt `readOnlyRootFilesystem` để "chữa"
lỗi của `nginx`. Không thêm capability nào — `nginx` nghe 8080 nên không cần `NET_BIND_SERVICE`.

**Đọc lại:**

- Bài [291](../291-security-context-vi.md): danh sách đầu bài (`readOnlyRootFilesystem`,
  `allowPrivilegeEscalation`),
  [*Đặt security context cho một Container*](../291-security-context-vi.md#đặt-security-context-cho-một-container-set-the-security-context-for-a-container),
  [*Đặt capabilities*](../291-security-context-vi.md#set-capabilities-for-a-container),
  [*Đặt Seccomp Profile*](../291-security-context-vi.md#set-the-seccomp-profile-for-a-container),
  mục *Thảo luận*.
- Bài [130](../130-application-security-checklist-vi.md#security-context-container) — khuyến nghị cấp
  container.
- Bài [127](../127-linux-kernel-security-vi.md#seccomp-considerations) — vì sao dùng profile mặc định
  của runtime.

**Bàn giao:** `k8s/app/web/deployment.yaml`, `k8s/app/api/deployment.yaml`;
`docs/adr/NNNN-rootfs-chi-doc.md`; `~/lab-evidence/9b/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` hai Deployment rollout xong; dry-run `enforce=restricted` không gọi tên Pod `web-*`, `api-*`.
- `PASS:` `nginx` và `postgrest` có root filesystem chỉ đọc; ghi vào `/` của `nginx` thất bại, ghi vào
  `/tmp` thành công.
- `PASS:` `nginx`: `CapEff` bằng 0, `NoNewPrivs` bằng 1, `Seccomp` bằng 2.
- `PASS:` Pod `api` đặt uid không phải root ở **cấp Pod**.
- `PASS:` `/`, `/healthz`, `/status.json`, `/api/books` của `web` và `/books` của `api` trả đúng.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-rootfs-chi-doc.md`:

1. Vì sao bảng Restricted của bài 115 không có `readOnlyRootFilesystem` trong khi checklist 130 có?
   Dự án vừa đặt cho mình chuẩn gì vượt Restricted?
2. `nginx` ghi những gì, ở đâu? Bạn tìm ra bằng cách nào? Vì sao chọn volume tạm chứ không bỏ cờ?
3. Trường nào chỉ đặt được ở cấp container, trường nào ở cả hai cấp? Vì sao bài 291 nói thiết lập cấp
   container không ảnh hưởng tới volume?
4. Init container chờ DB (nếu còn) dùng image `postgres`. Nó chạy uid nào, và vì sao uid đó phải có
   trong `/etc/passwd` của image thì `pg_isready` mới làm việc?

<details>
<summary>Gợi ý</summary>

1. `allowPrivilegeEscalation`, `capabilities`, `readOnlyRootFilesystem` chỉ có ở `securityContext` của
   container; `runAsNonRoot`, `runAsUser`, `seccompProfile` đặt ở cấp Pod là phủ được mọi container,
   kể cả ephemeral container gắn sau này.
2. Bật cờ chỉ đọc trước rồi đọc `kubectl logs` của `nginx`: dòng `open() "/tmp/..." failed (30:
   Read-only file system)` chỉ ra đúng thư mục cần volume.
3. `postgrest` không có shell — chỉ kiểm được spec và hành vi `/ready`; B6 đọc kernel của nó bằng
   ephemeral container.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-work/9b/ham.sh; cd ~/bookstore
for D in web api; do kubectl -n "$NS" rollout status "deployment/$D" --timeout=300s >/dev/null; done
# "Không gọi tên web, api" chỉ có nghĩa khi lệnh dry-run thành công.
if kubectl label --dry-run=server --overwrite ns "$NS" pod-security.kubernetes.io/enforce=restricted \
     > "$EV/b4-label-dry-run.txt" 2>&1; then
  ! grep -Eq '(^|[^-a-z])(web|api)-[a-z0-9]' "$EV/b4-label-dry-run.txt" \
    && echo 'PASS: dry-run restricted khong con goi ten Pod web, api'
else
  echo "FAIL: lenh dry-run loi, chua ket luan duoc — xem $EV/b4-label-dry-run.txt"
fi

RO() { jp "deployment/$1" "{.spec.template.spec.containers[?(@.name==\"$2\")].securityContext.readOnlyRootFilesystem}"; }
{ [ "$(RO web nginx)" = 'true' ] && [ "$(RO api postgrest)" = 'true' ]; } && echo 'PASS: nginx va postgrest rootfs chi doc'
WEB="$(pod_of web)"
kubectl -n "$NS" exec "$WEB" -c nginx -- sh -c 'touch /thu-ghi 2>&1; echo "rc_root=$?"; touch /tmp/thu-ghi && echo rc_tmp=0' \
  > "$EV/b4-ghi.txt" 2>&1
{ grep -q 'Read-only file system' "$EV/b4-ghi.txt" && grep -q 'rc_tmp=0' "$EV/b4-ghi.txt"; } \
  && echo 'PASS: ghi vao / that bai, ghi vao /tmp thanh cong'
C="$(proc_val "$NS" "$WEB" nginx CapEff)"; P="$(proc_val "$NS" "$WEB" nginx NoNewPrivs)"; S="$(proc_val "$NS" "$WEB" nginx Seccomp)"
echo "nginx CapEff=$C NoNewPrivs=$P Seccomp=$S" | tee "$EV/b4-kernel.txt"
{ [ "$((0x${C:-1}))" -eq 0 ] && [ "$P" = 1 ] && [ "$S" = 2 ]; } && echo 'PASS: nginx khong capability, no_new_privs, seccomp filter'
U="$(jp deployment/api '{.spec.template.spec.securityContext.runAsUser}')"
{ [ -n "$U" ] && [ "$U" -ne 0 ]; } && echo "PASS: Pod api dat uid $U o cap Pod"

get() { kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- "$1" 2>/dev/null; }
{ get http://127.0.0.1:8080/ | grep -q 'Bookstore' && get http://127.0.0.1:8080/healthz >/dev/null && \
  get http://127.0.0.1:8080/status.json | grep -q '"pod"' && \
  [ "$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)" -ge 8 ] && \
  [ "$(get http://api:3000/books | grep -o '"title"' | wc -l)" -ge 8 ]; } \
  && echo 'PASS: hop dong cua web va api giu nguyen'
```

</details>

<details>
<summary>Lời giải tham khảo — phần thêm vào hai Deployment</summary>

Chỉ phần thay đổi; giữ nguyên mọi thứ khác của bạn (topology spread, ServiceAccount, tài nguyên,
probe, ConfigMap sinh bằng Kustomize).

```yaml
# k8s/app/web/deployment.yaml — thêm vào spec.template.spec
      hostUsers: false                         # 3a, giữ nguyên
      securityContext:
        runAsNonRoot: true                     # 3a
        runAsUser: 101                         # 3a
        runAsGroup: 101                        # 3a
        seccompProfile: {type: RuntimeDefault} # 9b
      volumes:
      - name: tmp                              # 9b: pid và file tạm của nginx-unprivileged nằm ở /tmp
        emptyDir: {}
      # ... các volume html, podinfo, nginx-conf, page-template giữ nguyên
      initContainers:
      - name: render-html
        # ... giữ nguyên
        securityContext:                       # 9b
          allowPrivilegeEscalation: false
          capabilities: {drop: ["ALL"]}
      - name: status-writer
        # ... giữ nguyên, restartPolicy: Always
        securityContext:                       # 9b
          allowPrivilegeEscalation: false
          capabilities: {drop: ["ALL"]}
      containers:
      - name: nginx
        # ... giữ nguyên
        securityContext:
          allowPrivilegeEscalation: false      # 3a
          capabilities: {drop: ["ALL"]}        # 9b
          readOnlyRootFilesystem: true         # 9b
        volumeMounts:
        - {name: tmp, mountPath: /tmp}         # 9b
        # ... html, nginx-conf giữ nguyên
```

```yaml
# k8s/app/api/deployment.yaml — spec.template.spec
      securityContext:                         # 9b: cấp Pod, phủ cả ephemeral container ở B6
        runAsNonRoot: true                     # 9b
        runAsUser: 1000                        # 9b: USER của image postgrest
        runAsGroup: 1000                       # 9b
        seccompProfile: {type: RuntimeDefault} # 9b
        sysctls:                               # 9a: giữ nguyên — sysctl an toàn, Restricted cho phép
        - {name: net.ipv4.tcp_keepalive_time, value: "60"}
        - {name: net.ipv4.tcp_keepalive_intvl, value: "10"}
        - {name: net.ipv4.tcp_keepalive_probes, value: "6"}
      initContainers:                          # chỉ khi bạn còn giữ init chờ DB
      - name: wait-for-db
        # ... giữ nguyên
        securityContext:
          runAsUser: 999                       # image postgres: libpq cần uid có trong /etc/passwd
          runAsGroup: 999
          allowPrivilegeEscalation: false
          capabilities: {drop: ["ALL"]}
      containers:
      - name: postgrest
        # ... giữ nguyên
        securityContext:
          allowPrivilegeEscalation: false      # 3a
          capabilities: {drop: ["ALL"]}        # 9b
          readOnlyRootFilesystem: true         # 9b: binary tĩnh, không ghi file
```

Ý chính cho ADR:

1. Restricted là chuẩn **dùng chung**, đánh đổi tương thích có giới hạn; rootfs chỉ đọc phá quá nhiều
   ứng dụng ghi file tạm nên không nằm trong đó. Dự án đặt chuẩn riêng: mọi container dịch vụ dài
   hạn có rootfs chỉ đọc, chỗ ghi khai tường minh bằng `emptyDir`.
2. `nginx-unprivileged` đặt pid và năm thư mục tạm (`client_temp`, `proxy_temp`, …) dưới `/tmp`; log
   lỗi khi thiếu volume chỉ đúng đường dẫn. Bỏ cờ là bỏ luôn lớp bảo vệ cho cả image; `emptyDir` chỉ
   mở đúng một thư mục, và mất khi Pod mất.
3. Chỉ cấp container: `allowPrivilegeEscalation`, `capabilities`, `privileged`,
   `readOnlyRootFilesystem`. Cả hai cấp: `runAsUser`, `runAsGroup`, `runAsNonRoot`, `seccompProfile`.
   Chỉ cấp Pod: `fsGroup`, `supplementalGroups` — vì chúng tác động lên volume, thứ thuộc cả Pod.
4. Uid 999 (`postgres`). Không truyền `-U` thì libpq tra tên người dùng mặc định từ `/etc/passwd`
   theo uid; uid 1000 không có trong image `postgres`, `pg_isready` không thử kết nối và vòng chờ
   chạy mãi.

</details>

## B5. [BS-9B-05] Job, CronJob và `node-agent` đạt `restricted`

**Loại:** Tính năng.

**Bối cảnh:** ba workload còn lại trong bảng kiểm kê. Lab 9a đã cho `db-migrate` và `db-backup` chạy
uid 999 (container `report-status` uid 101) và tắt leo thang đặc quyền; còn thiếu capability và
seccomp. Hai tác vụ này phần lớn thời gian **không có Pod nào chạy**, nên sai sót của chúng chỉ lộ
ra vào lần chạy kế tiếp: lúc deploy, hoặc lúc nửa đêm. Ticket kết thúc khi một lần triển khai từ đầu
không còn một cảnh báo Pod Security nào.

**Yêu cầu:**

1. Mọi container trong template của Job `db-migrate`, CronJob `db-backup` (init `pg-dump` và container
   `report-status` của Lab 9a) và DaemonSet `node-agent` đạt Restricted.
2. `db-migrate` vẫn bất biến kết quả và `deploy.sh` vẫn chờ nó Complete; `db-backup` vẫn ghi bản dump
   vào PVC `backup-store`, vẫn in `db-backup: OK`, vẫn cập nhật ConfigMap `backup-status`.
3. `node-agent` vẫn Ready trên cả ba node, cổng `http` 9100, `/node.json` đúng tên node.
4. Một lần triển khai từ đầu **không** sinh cảnh báo Pod Security nào; dry-run `enforce=restricted`
   trên `bookstore` không gọi tên Pod nào.
5. ⭐ Root filesystem của `node-agent` chỉ đọc.

**Ràng buộc:** không đổi lịch, `concurrencyPolicy`, `startingDeadlineSeconds` của `db-backup`; không đổi
cổng của `node-agent`.

**Đọc lại:**

- Bài [286](../286-migrate-from-psp-vi.md#identify-appropriate-level) — đoạn **Thận trọng**.
- Bài [116](../116-pod-security-admission-vi.md#workload-resources-and-pod-templates).
- Bài [115](../115-pod-security-standards-vi.md#restricted) — ghi chú wildcard: một container không đạt
  là cả Pod trượt.

**Bàn giao:** `k8s/app/jobs/*`, `k8s/app/agent/daemonset.yaml`; `docs/adr/NNNN-tac-vu-nen-restricted.md`;
`~/lab-evidence/9b/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` teardown rồi deploy: không một dòng `would violate PodSecurity` nào.
- `PASS:` dry-run `enforce=restricted` trên `bookstore` không có vi phạm nào.
- `PASS:` `db-migrate` Complete (hoặc đã được TTL dọn) và `api.books` ít nhất 8 dòng.
- `PASS:` một Job tạo từ CronJob `db-backup` Complete, log có `db-backup: OK`, Pod của nó khai
  `runAsNonRoot: true`; ConfigMap `backup-status` được cập nhật.
- `PASS:` `node-agent` Ready trên mọi node; `/node.json` đúng tên node; `CapEff` của `agent` bằng 0.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-tac-vu-nen-restricted.md`:

1. Ở B2, dry-run nhãn không thấy `db-backup`. Nếu bạn bỏ sót nó rồi bật `enforce: restricted` (B6),
   chuyện gì xảy ra vào lần chạy kế tiếp của CronJob, và bạn nhìn ở **object nào** để thấy?
2. Lab 11a chuyển `node-agent` sang `bookstore-system` mức `privileged` vì cần `hostPath`. Vậy làm nó đạt
   Restricted bây giờ có thừa không?
3. `report-status` chạy uid 101 trong khi cấp Pod đặt 999. Restricted có quan tâm uid cụ thể không?
   Ràng buộc nào phải khai ở **từng** container (kể cả init), ràng buộc nào đặt một lần ở cấp Pod là đủ?

<details>
<summary>Gợi ý</summary>

1. Uid đã đúng từ Lab 9a; việc còn lại là hai trường mà mọi container — kể cả init `pg-dump` — phải tự
   khai, và một trường đặt một lần ở cấp Pod.
2. Cảnh báo của lần tạo mới là thước đo đúng nhất: `./scripts/teardown.sh && ./scripts/deploy.sh 2>&1 | grep -c 'would violate'`.
3. Pod Job đã xong không còn để `exec`; đọc spec của nó, hoặc log.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
. ~/lab-work/9b/ham.sh; cd ~/bookstore
./scripts/teardown.sh > "$EV/b5-teardown.txt" 2>&1
./scripts/deploy.sh > "$EV/b5-deploy.txt" 2>&1; DE=$?; echo "deploy exit=$DE"
# "Không có cảnh báo" chỉ có nghĩa khi lệnh đã thực sự chạy tới API server và thành công.
{ [ "$DE" -eq 0 ] && [ "$(grep -c 'would violate PodSecurity' "$EV/b5-deploy.txt")" -eq 0 ]; } \
  && echo 'PASS: trien khai tu dau khong con canh bao Pod Security nao'
if kubectl label --dry-run=server --overwrite ns "$NS" pod-security.kubernetes.io/enforce=restricted \
     > "$EV/b5-label-dry-run.txt" 2>&1; then
  ! grep -q 'violate' "$EV/b5-label-dry-run.txt" && echo 'PASS: dry-run restricted khong co vi pham nao'
else
  echo "FAIL: lenh dry-run loi, chua ket luan duoc — xem $EV/b5-label-dry-run.txt"
fi

if kubectl -n "$NS" get job db-migrate >/dev/null 2>&1; then
  [ "$(jp job/db-migrate '{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ] && echo 'PASS: db-migrate Complete'
else
  echo 'PASS: db-migrate da duoc TTL don'
fi
N="$(kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && echo "PASS: api.books co $N dong"

J="b5-backup-$(date +%s)"
kubectl -n "$NS" create job "$J" --from=cronjob/db-backup > "$EV/b5-backup-create.txt" 2>&1
{ kubectl -n "$NS" wait --for=condition=Complete "job/$J" --timeout=180s >/dev/null && \
  kubectl -n "$NS" logs "job/$J" --all-containers 2>/dev/null | grep -q '^db-backup: OK' && \
  kubectl -n "$NS" get pod -l "batch.kubernetes.io/job-name=$J" \
    -o jsonpath='{.items[0].spec.securityContext.runAsNonRoot}{.items[0].spec.containers[*].securityContext.runAsNonRoot}' \
    | grep -q true; } \
  && echo 'PASS: Job tu CronJob db-backup Complete, in db-backup: OK, chay non-root'
kubectl -n "$NS" delete job "$J" --wait=true >/dev/null
[ -n "$(jp configmap/backup-status '{.data.last-run}')" ] && echo 'PASS: backup-status duoc cap nhat'

NODES="$(kubectl get nodes --no-headers | wc -l)"
[ "$(jp daemonset/node-agent '{.status.numberReady}')" = "$NODES" ] && echo "PASS: node-agent Ready tren $NODES node"
AG="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=agent -o jsonpath='{.items[0].metadata.name}')"
AN="$(kubectl -n "$NS" get pod "$AG" -o jsonpath='{.spec.nodeName}')"
AIP="$(kubectl -n "$NS" get pod "$AG" -o jsonpath='{.status.podIP}')"
CN="$(jp daemonset/node-agent '{.spec.template.spec.containers[0].name}')"
C="$(proc_val "$NS" "$AG" "$CN" CapEff)"
{ kubectl -n "$NS" exec "$(pod_of web)" -c status-writer -- wget -qO- "http://$AIP:9100/node.json" 2>/dev/null \
    | grep -Eq "\"node\" *: *\"$AN\"" && [ "$((0x${C:-1}))" -eq 0 ]; } \
  && echo 'PASS: node-agent tra /node.json dung ten node, khong capability'
```

</details>

<details>
<summary>Lời giải tham khảo — phần thêm vào ba workload</summary>

```yaml
# Job db-migrate: spec.template.spec
      securityContext:
        runAsNonRoot: true                     # 9a
        runAsUser: 999                         # 9a
        runAsGroup: 999                        # 9a
        seccompProfile: {type: RuntimeDefault} # 9b
      containers:
      - name: migrate
        # ... giữ nguyên
        securityContext:
          allowPrivilegeEscalation: false      # 9a
          capabilities: {drop: ["ALL"]}        # 9b
```

```yaml
# CronJob db-backup: spec.jobTemplate.spec.template.spec
          securityContext:
            runAsNonRoot: true                 # 9a
            runAsUser: 999                     # 9a
            runAsGroup: 999                    # 9a
            seccompProfile: {type: RuntimeDefault}   # 9b
          initContainers:
          - name: pg-dump
            # ... giữ nguyên
            securityContext:
              allowPrivilegeEscalation: false  # 9a
              capabilities: {drop: ["ALL"]}    # 9b
          containers:
          - name: report-status
            # ... giữ nguyên
            securityContext:
              runAsUser: 101                   # 9a
              allowPrivilegeEscalation: false  # 9a
              capabilities: {drop: ["ALL"]}    # 9b
```

```yaml
# k8s/app/agent/daemonset.yaml — spec.template.spec
      securityContext:
        runAsNonRoot: true                     # 4b
        runAsUser: 65534                       # 4b
        runAsGroup: 65534                      # 4b
        seccompProfile: {type: RuntimeDefault} # 9b
      containers:
      - name: agent
        # ... giữ nguyên; /www là emptyDir
        securityContext:
          allowPrivilegeEscalation: false      # 4b
          capabilities: {drop: ["ALL"]}        # 9b
          readOnlyRootFilesystem: true         # 9b ⭐
```

Ý chính cho ADR:

1. CronJob vẫn tạo Job đúng giờ; Job controller tạo Pod và bị `enforce` từ chối, nên Job không bao giờ
   có Pod và không bao giờ Complete. Dấu vết nằm ở **sự kiện của Job** (`FailedCreate` kèm
   `violates PodSecurity`), không nằm ở `kubectl get pods`. Bản backup đêm đó không tồn tại mà không
   ai được báo — đúng loại workload bài 286 nói dry-run bỏ sót.
2. Không thừa. Namespace `privileged` chỉ **cho phép**, không **bắt buộc**: agent vẫn nên drop mọi
   capability và mount `hostPath` chỉ đọc. Mức namespace là trần, `securityContext` là thứ thực sự
   giới hạn tiến trình.
3. Không quan tâm uid cụ thể: Restricted chỉ đòi `runAsNonRoot: true` và `runAsUser` khác 0 — uid
   101 hợp lệ. `allowPrivilegeEscalation: false` và drop `ALL` là trường **chỉ có** ở cấp container,
   nên phải khai ở từng container kể cả init; `runAsNonRoot` và `seccompProfile` đặt ở cấp Pod là phủ
   mọi container (cột chính sách của hai dòng tương ứng trong bảng Restricted, bài 115).

</details>

## B6. [BS-9B-06] Bật `enforce: restricted`

**Loại:** Tính năng.

**Bối cảnh:** bảng kiểm kê đã trống. Bước 3.c của bài 286 là thực thi — nhưng chỉ sau khi chứng minh
được là an toàn. Siết xong, một quy trình vận hành cũ sẽ gãy: runbook gỡ lỗi `api` của Lab 3a gắn
ephemeral container không khai `securityContext`. Ticket này siết và sửa luôn chỗ gãy đó.

**Yêu cầu:**

1. `bookstore` lên `enforce: restricted`, ghim version; `warn` và `audit` giữ `restricted`. Trước khi
   đổi nhãn, có bằng chứng API server không gọi tên Pod nào sẽ vi phạm.
2. Triển khai lại từ đầu thành công dưới `enforce: restricted`.
3. Pod đặc quyền bị từ chối ở mức `restricted` (chứng minh bằng dry-run).
4. Quy trình gỡ lỗi `api` chạy được dưới `restricted`: cách cũ của Lab 3a bị từ chối — ghi lại câu từ
   chối; cách mới gắn được container gỡ lỗi nhắm vào `postgrest`, và từ đó đọc được uid, `CapEff`,
   `NoNewPrivs`, `Seccomp` của tiến trình `postgrest`. Cập nhật `docs/runbook-debug-api.md`.
5. Ghi nhận: `tests/smoke.sh` của tag `lab-9a` có bước nào tạo Pod tạm không khai `securityContext`
   thì bước đó giờ fail — đúng hành vi; smoke mới ở mục đóng sprint thay nó.

**Ràng buộc:** nhãn nằm trong manifest. Không miễn trừ gì — miễn trừ chỉ khai được trong cấu hình
apiserver (bài 282), lab cấm sửa.

**Đọc lại:**

- Bài [286](../286-migrate-from-psp-vi.md#enforce-pod-security-level) — bước 3.c; bước
  [3.b](../286-migrate-from-psp-vi.md#verify-pss-level).
- Bài [283](../283-enforce-standards-namespace-labels-vi.md) — ghi chú đầu mục *Thêm nhãn vào các
  namespace hiện có*: thêm hay đổi nhãn `enforce` **hoặc nhãn version** làm plugin soi lại Pod đang
  chạy.
- Bài [116](../116-pod-security-admission-vi.md#miễn-trừ-exemptions) — vì sao không miễn trừ.
- Bài [52](../52-ephemeral-containers-vi.md) (Lab 3a, B8) và
  [Debug Running Pods](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/),
  mục *Debugging a Pod or Node while applying a profile*.

**Bàn giao:** `k8s/app/00-namespace.yaml`; `docs/runbook-debug-api.md`;
`docs/adr/NNNN-enforce-restricted.md`; `~/lab-evidence/9b/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` `bookstore` có `enforce`, `warn`, `audit` mức `restricted`, version bằng minor của server.
- `PASS:` dry-run đổi nhãn version không gọi tên Pod nào — mọi Pod đang chạy đạt mức đang thực thi.
- `PASS:` Pod đặc quyền bị từ chối `violates PodSecurity "restricted:<minor>"`.
- `PASS:` teardown rồi deploy thành công dưới `enforce`.
- `PASS:` gắn container gỡ lỗi theo cách cũ bị từ chối; cách mới chạy được.
- `PASS:` từ container gỡ lỗi: `postgrest` chạy uid không phải root, `CapEff` 0, `NoNewPrivs` 1,
  `Seccomp` 2.
- `PASS:` `docs/runbook-debug-api.md` ghi cách gỡ lỗi mới.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-enforce-restricted.md`:

1. Đổi nhãn `enforce` lên `restricted` có làm chết Pod đang vi phạm không? Vậy vì sao vẫn phải dry-run
   trước khi đổi, và dry-run sau khi đã đổi thì dùng nhãn nào?
2. Ai trong cluster hạ được mức này về `privileged`? Bước 1 của bài 286 và mục *Sửa đổi Namespace* của
   bài 120 nói gì? (B9 kiểm bằng `can-i`.)
3. Container gỡ lỗi mặc định bị từ chối vì những ràng buộc nào? Profile mới đặt gì cho nó, và vì sao
   uid đặt ở **cấp Pod** của `api` (B4) khiến nó khởi động được?
4. Lab 11a cần `hostPath` cho `node-agent`. Vì sao dự án tạo namespace `bookstore-system` mức
   `privileged` thay vì hạ `bookstore` hay miễn trừ?

<details>
<summary>Gợi ý</summary>

1. Dry-run trước khi đổi: `kubectl label --dry-run=server --overwrite ns bookstore pod-security.kubernetes.io/enforce=restricted`.
   Sau khi đã đổi, cùng lệnh với cùng giá trị không gửi gì (nhãn không đổi); đổi **nhãn version**
   sang `latest` bằng dry-run thì plugin soi lại.
2. `kubectl debug` có cờ `--profile`; giá trị `restricted` đặt các trường mà Restricted đòi cho
   ephemeral container. Profile không đặt uid — uid đến từ `securityContext` cấp Pod.
3. Đọc kernel của `postgrest` từ container gỡ lỗi: hai container chung process namespace nhờ
   `--target`, nên `/proc/1` trong container gỡ lỗi là `postgrest`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
. ~/lab-work/9b/ham.sh; cd ~/bookstore
lbl() { kubectl get ns "$NS" -o jsonpath="{.metadata.labels.pod-security\.kubernetes\.io/$1}" 2>/dev/null; }
BAD=0
for M in enforce warn audit; do
  { [ "$(lbl "$M")" = 'restricted' ] && [ "$(lbl "$M-version")" = "$PSA_VER" ]; } || BAD=1
done
[ "$BAD" -eq 0 ] && echo "PASS: bookstore restricted o ca ba che do, version $PSA_VER"

./scripts/teardown.sh > "$EV/b6-teardown.txt" 2>&1
./scripts/deploy.sh > "$EV/b6-deploy.txt" 2>&1 && echo 'PASS: trien khai tu dau thanh cong duoi enforce restricted'
if kubectl label --dry-run=server --overwrite ns "$NS" pod-security.kubernetes.io/enforce-version=latest \
     > "$EV/b6-label-dry-run.txt" 2>&1; then
  ! grep -q 'violate' "$EV/b6-label-dry-run.txt" && echo 'PASS: dry-run doi version khong goi ten Pod nao'
else
  echo "FAIL: lenh dry-run loi, chua ket luan duoc — xem $EV/b6-label-dry-run.txt"
fi
printf '%s\n' 'apiVersion: v1' 'kind: Pod' 'metadata: {name: thu-dac-quyen}' 'spec:' \
  '  containers: [{name: c, image: busybox:1.37, securityContext: {privileged: true}}]' \
  | kubectl -n "$NS" create --dry-run=server -f - > "$EV/b6-dac-quyen.txt" 2>&1
grep -q "violates PodSecurity \"restricted:$PSA_VER\"" "$EV/b6-dac-quyen.txt" && echo 'PASS: Pod dac quyen bi tu choi'

API="$(pod_of api)"; T="$(date +%s)"
kubectl -n "$NS" debug "pod/$API" --image=busybox:1.37 --target=postgrest -c "cu-$T" -- sleep 600 \
  > "$EV/b6-debug-cu.txt" 2>&1
grep -q 'violates PodSecurity' "$EV/b6-debug-cu.txt" && echo 'PASS: cach go loi cu bi restricted tu choi'
DBG="dbg-$T"
kubectl -n "$NS" debug "pod/$API" --image=busybox:1.37 --target=postgrest -c "$DBG" --profile=restricted \
  -- sleep 600 > "$EV/b6-debug-moi.txt" 2>&1
S=''
for i in $(seq 1 60); do
  S="$(kubectl -n "$NS" get pod "$API" -o jsonpath="{.status.ephemeralContainerStatuses[?(@.name==\"$DBG\")].state.running.startedAt}")"
  [ -n "$S" ] && break; sleep 2
done
[ -n "$S" ] && echo 'PASS: container go loi chay duoi restricted'
rd() { kubectl -n "$NS" exec "$API" -c "$DBG" -- awk "/^$1:/{print \$2}" /proc/1/status 2>/dev/null; }
U="$(rd Uid)"; C="$(rd CapEff)"; P="$(rd NoNewPrivs)"; SC="$(rd Seccomp)"
kubectl -n "$NS" exec "$API" -c "$DBG" -- ps > "$EV/b6-ps.txt" 2>&1
echo "postgrest uid=$U CapEff=$C NoNewPrivs=$P Seccomp=$SC" | tee "$EV/b6-postgrest-kernel.txt"
{ grep -q 'postgrest' "$EV/b6-ps.txt" && [ -n "$U" ] && [ "$U" -ne 0 ] && [ "$((0x${C:-1}))" -eq 0 ] && \
  [ "$P" = 1 ] && [ "$SC" = 2 ]; } && echo 'PASS: postgrest non-root, khong capability, no_new_privs, seccomp filter'
grep -q -- '--profile=restricted' docs/runbook-debug-api.md && echo 'PASS: runbook ghi cach go loi moi'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/00-namespace.yaml — sau B6
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    # Pod Security (9b): cả ba chế độ restricted. v1.35 là minor của cluster — đối chiếu bảng A1.3.
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.35
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.35
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.35
```

```bash
cd ~/bookstore
kubectl label --dry-run=server --overwrite ns bookstore pod-security.kubernetes.io/enforce=restricted \
  | tee ~/lab-evidence/9b/b6-truoc-khi-siet.txt     # không có dòng "violate" mới đổi manifest
./scripts/deploy.sh
```

Đoạn thay thế trong `docs/runbook-debug-api.md`:

```markdown
## Gắn container gỡ lỗi vào api (từ Lab 9b, namespace enforce restricted)

kubectl -n bookstore debug pod/<pod-api> --image=busybox:1.37 --target=postgrest \
  -c debugger --profile=restricted -- sleep 3600
kubectl -n bookstore exec <pod-api> -c debugger -- ps
kubectl -n bookstore exec <pod-api> -c debugger -- wget -qO- http://127.0.0.1:3001/ready

- Không có --profile=restricted: API server từ chối (allowPrivilegeEscalation, capabilities).
- Container gỡ lỗi chạy uid 1000 lấy từ securityContext cấp Pod của api; không có capability nào.
- Đã gắn thì không gỡ được; Pod được thay ở lần phát hành sau.
```

Ý chính cho ADR:

1. Không. Bài 283: đổi nhãn chỉ làm plugin soi lại Pod đang chạy và trả **cảnh báo**; Pod vi phạm vẫn
   chạy cho tới lần bị thay — rồi mới không tạo lại được. Dry-run trước để biết điều đó không xảy ra.
   Sau khi đã đổi, dry-run đổi nhãn version (sang `latest`) để buộc plugin soi lại.
2. Ai `update`, `patch` hoặc `create` được Namespace. Đó là đường vòng qua toàn bộ chính sách, nên rà
   quyền namespace là bước 1 của bài 286 — trước mọi việc khác.
3. Profile mặc định để trống `allowPrivilegeEscalation` và giữ capability mặc định — hai ràng buộc
   Restricted đòi ở **mọi** container, kể cả ephemeral. Profile `restricted` đặt `runAsNonRoot`,
   `allowPrivilegeEscalation: false`, drop `ALL`, seccomp `RuntimeDefault`; uid 1000 thì đến từ cấp
   Pod — thiếu nó, `busybox` chạy root và kubelet từ chối vì `runAsNonRoot`.
4. Mức Pod Security là **của namespace**: hạ `bookstore` là mở cửa cho mọi workload ứng dụng. Miễn trừ
   cần sửa cấu hình apiserver. Tách workload hạ tầng sang namespace riêng giữ mức cao cho phần còn
   lại — đúng tinh thần "mỗi workload một namespace" của bài 122.

</details>

## B7. [BS-9B-07] ⭐ Điều tra: kernel thật sự áp gì, và sysctl

**Loại:** Điều tra — tùy chọn.

**Bối cảnh:** mọi dòng `securityContext` của B3–B6 chỉ có giá trị nếu kernel thật sự áp chúng. Ticket
này đặt container Bookstore cạnh một container không khai gì và một container đặc quyền, đọc trạng
thái từ `/proc`, rồi dựng lại hai cách một sysctl không an toàn bị chặn.

**Yêu cầu:** trong `bookstore-thu`, Pod ghim `lab-k8s-worker2`:

1. So ba container — `busybox` không khai gì (`thu-thuong`), `nginx` của `web`, và `busybox`
   `privileged: true` **kèm** `seccompProfile: RuntimeDefault` (`thu-dac-quyen`) — theo bốn giá trị:
   `CapPrm`, `NoNewPrivs`, `Seccomp`, profile AppArmor. Xóa `thu-dac-quyen` ngay khi đo xong.
2. Từ `thu-thuong`: `/proc/sys` đọc được nhưng không ghi được; `/proc/kcore` bị che.
3. Sysctl an toàn `net.ipv4.ip_unprivileged_port_start=80` đặt cho Pod: đổi bên trong Pod, không đổi
   trên node.
4. Sysctl không an toàn `net.core.somaxconn`: trong `bookstore` bị từ chối ngay ở admission (dry-run);
   trong `bookstore-thu` được lập lịch rồi kubelet từ chối khởi chạy.

**Ràng buộc:** Pod đặc quyền chỉ trong `bookstore-thu`, chỉ trên `lab-k8s-worker2`, bị xóa trong cùng
script. Không `hostPath`. Không đổi cấu hình kubelet.

**Đọc lại:**

- Bài [127](../127-linux-kernel-security-vi.md#kernel-security-features-privileged-containers) và
  [*Container đặc quyền*](../127-linux-kernel-security-vi.md#privileged-containers).
- Bài [291](../291-security-context-vi.md#set-capabilities-for-a-container) — so bitmap `CapPrm`;
  [*Quản lý truy cập vào filesystem `/proc`*](../291-security-context-vi.md#proc-access).
- Bài [258](../258-sysctl-cluster-vi.md):
  [*Sysctl an toàn và không an toàn*](../258-sysctl-cluster-vi.md#sysctl-an-toàn-và-không-an-toàn-safe-and-unsafe-sysctls),
  [*Thiết lập sysctl cho một Pod*](../258-sysctl-cluster-vi.md#thiết-lập-sysctl-cho-một-pod-setting-sysctls-for-a-pod).

**Bàn giao:** `~/lab-work/9b/b7-*.yaml`; `docs/adr/NNNN-rang-buoc-kernel.md`; `~/lab-evidence/9b/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` `nginx` của `web` giữ ít capability hơn container không khai gì; `NoNewPrivs` 1 so với 0.
- `PASS:` container đặc quyền: nhiều capability hơn container thường, `Seccomp` 0 dù khai
  `RuntimeDefault`, AppArmor `unconfined`; đã bị xóa.
- `PASS:` `/proc/sys` không ghi được; `/proc/kcore` kích thước 0.
- `PASS:` sysctl an toàn có giá trị 80 trong Pod, khác giá trị trên node.
- `PASS:` sysctl không an toàn: `bookstore` từ chối với `forbidden sysctls`; `bookstore-thu` cho Pod
  lập lịch rồi kết thúc `Failed`, lý do `SysctlForbidden`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-rang-buoc-kernel.md`:

1. Một manifest khai `RuntimeDefault` và `privileged: true`: thứ nào thắng, và bạn chứng minh bằng ba
   giá trị nào? Cách đúng khi ứng dụng cần một quyền quản trị hệ điều hành là gì?
2. Cùng một Pod dùng sysctl không an toàn, hai namespace, hai triệu chứng. Bạn nhìn ở đâu trong mỗi
   trường hợp? Cùng câu hỏi cho `runAsNonRoot` với image chạy root.
3. Vì sao `nginx` của dự án không cần `NET_BIND_SERVICE`, capability duy nhất Restricted cho thêm lại?

<details>
<summary>Script nghiệm thu và lời giải tham khảo</summary>

```yaml
# ~/lab-work/9b/b7-pods.yaml — áp vào bookstore-thu
apiVersion: v1
kind: Pod
metadata: {name: thu-thuong}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  containers: [{name: c, image: busybox:1.37, command: ["sh", "-c", "sleep 3600"]}]
---
apiVersion: v1
kind: Pod
metadata: {name: thu-dac-quyen}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  securityContext: {seccompProfile: {type: RuntimeDefault}}
  containers:
  - {name: c, image: busybox:1.37, command: ["sh", "-c", "sleep 3600"], securityContext: {privileged: true}}
---
apiVersion: v1
kind: Pod
metadata: {name: thu-sysctl-an-toan}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  securityContext:
    sysctls: [{name: net.ipv4.ip_unprivileged_port_start, value: "80"}]
  containers: [{name: c, image: busybox:1.37, command: ["sh", "-c", "sleep 3600"]}]
```

```yaml
# ~/lab-work/9b/b7-sysctl-khong-an-toan.yaml — không khai namespace; script áp vào hai namespace
apiVersion: v1
kind: Pod
metadata: {name: thu-sysctl-khong-an-toan}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    seccompProfile: {type: RuntimeDefault}
    sysctls: [{name: net.core.somaxconn, value: "1024"}]
  containers:
  - name: c
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    securityContext: {allowPrivilegeEscalation: false, capabilities: {drop: ["ALL"]}}
```

```bash
. ~/lab-work/9b/ham.sh
kubectl get ns "$THU" >/dev/null 2>&1 || kubectl apply -f "$WK/b1-namespaces.yaml"
kubectl -n "$THU" apply -f "$WK/b7-pods.yaml"
kubectl -n "$THU" wait --for=condition=Ready pod/thu-thuong pod/thu-dac-quyen pod/thu-sysctl-an-toan --timeout=180s
aa() { kubectl -n "$1" exec "$2" -c "$3" -- cat /proc/self/attr/current 2>/dev/null; }
WEB="$(pod_of web)"
CT="$(proc_val "$THU" thu-thuong c CapPrm)";   NT="$(proc_val "$THU" thu-thuong c NoNewPrivs)"
CW="$(proc_val "$NS" "$WEB" nginx CapPrm)";     NW="$(proc_val "$NS" "$WEB" nginx NoNewPrivs)"
CP="$(proc_val "$THU" thu-dac-quyen c CapPrm)"; SP="$(proc_val "$THU" thu-dac-quyen c Seccomp)"
AT="$(aa "$THU" thu-thuong c)"; AP="$(aa "$THU" thu-dac-quyen c)"
{
  echo "thuong     CapPrm=$CT NoNewPrivs=$NT Seccomp=$(proc_val "$THU" thu-thuong c Seccomp) AppArmor=$AT"
  echo "web/nginx  CapPrm=$CW NoNewPrivs=$NW Seccomp=$(proc_val "$NS" "$WEB" nginx Seccomp)"
  echo "dac-quyen  CapPrm=$CP Seccomp=$SP AppArmor=$AP node=$(kubectl -n "$THU" get pod thu-dac-quyen -o jsonpath='{.spec.nodeName}')"
} | tee "$EV/b7-kernel.txt"
{ [ "$((0x$CW))" -lt "$((0x$CT))" ] && [ "$NW" = 1 ] && [ "$NT" = 0 ]; } \
  && echo 'PASS: nginx cua web it capability hon container thuong, no_new_privs bat'
{ [ "$((0x$CP))" -gt "$((0x$CT))" ] && [ "$SP" = 0 ] && echo "$AP" | grep -q unconfined; } \
  && echo 'PASS: privileged ghi de RuntimeDefault va AppArmor, them capability'
kubectl -n "$THU" delete pod thu-dac-quyen --wait=true >/dev/null && echo 'PASS: da xoa Pod dac quyen'

kubectl -n "$THU" exec thu-thuong -- sh -c \
  'echo 100 > /proc/sys/net/ipv4/ip_unprivileged_port_start 2>/dev/null && echo ghi=ok || echo ghi=loi; stat -c "kcore=%s" /proc/kcore' \
  | tee "$EV/b7-proc.txt"
{ grep -q 'ghi=loi' "$EV/b7-proc.txt" && grep -q 'kcore=0' "$EV/b7-proc.txt"; } \
  && echo 'PASS: /proc/sys chi doc, /proc/kcore bi che'

IN="$(kubectl -n "$THU" exec thu-sysctl-an-toan -- cat /proc/sys/net/ipv4/ip_unprivileged_port_start)"
ON="$(ssh "$W2" 'sysctl -n net.ipv4.ip_unprivileged_port_start')"
echo "trong Pod=$IN tren node=$ON" | tee "$EV/b7-sysctl-an-toan.txt"
{ [ "$IN" = 80 ] && [ "$ON" != "$IN" ]; } && echo 'PASS: sysctl an toan doi trong Pod, khong doi tren node'

kubectl -n "$NS" create --dry-run=server -f "$WK/b7-sysctl-khong-an-toan.yaml" > "$EV/b7-sysctl-admission.txt" 2>&1
grep -q 'forbidden sysctls' "$EV/b7-sysctl-admission.txt" && echo 'PASS: bookstore tu choi o admission'
kubectl -n "$THU" apply -f "$WK/b7-sysctl-khong-an-toan.yaml" >/dev/null
R=''; for i in $(seq 1 60); do
  R="$(kubectl -n "$THU" get pod thu-sysctl-khong-an-toan -o jsonpath='{.status.reason}')"; [ -n "$R" ] && break; sleep 2
done
P="$(kubectl -n "$THU" get pod thu-sysctl-khong-an-toan -o jsonpath='{.status.phase}/{.spec.nodeName}')"
echo "reason=$R phase/node=$P" | tee "$EV/b7-sysctl-kubelet.txt"
{ [ "$R" = 'SysctlForbidden' ] && [ "$P" = "Failed/$W2" ]; } && echo 'PASS: bookstore-thu: lap lich roi kubelet tu choi'
kubectl -n "$THU" delete pod thu-thuong thu-sysctl-an-toan thu-sysctl-khong-an-toan --wait=true >/dev/null
```

Ý chính cho ADR:

1. `privileged` thắng: seccomp về `Unconfined` (giá trị 0), AppArmor `unconfined`, được thêm toàn bộ
   capability. Bài 127: cấp đúng capability cần qua `capabilities`, chỉ dùng chế độ đặc quyền khi cần
   thứ không cấp được qua `securityContext` — và khi đó workload thuộc namespace `privileged` riêng.
2. Namespace có `enforce`: chặn ở **admission** — chỉ có câu lỗi ở client, không object nào. Namespace
   không nhãn: Pod tồn tại, được lập lịch, kubelet từ chối — đọc `status.reason`, `describe`, sự kiện.
   `runAsNonRoot` với image root luôn chặn ở **kubelet** (`CreateContainerConfigError`), kể cả trong
   namespace `restricted`.
3. Image `nginx-unprivileged` nghe 8080, trên ngưỡng cổng đặc quyền; Service `web` ánh xạ 80 sang
   cổng `http` (Lab 5a), nên không tiến trình nào cần bind cổng dưới 1024.

</details>

## B8. [BS-9B-08] Tenant thứ hai `bookstore-staging`

**Loại:** Tính năng.

**Bối cảnh:** đội phát triển cần một chỗ thử bản phát hành mà không chạm vào `bookstore`
([mục 3.1](DU-AN-BOOKSTORE.md#31-sơ-đồ-tổng-thể)). Bài 122 không đưa cơ chế mới nào — nó ghép
namespace, RBAC, quota và NetworkPolicy thành một mô hình cô lập. Ngân sách là ràng buộc thật: hai
worker, mỗi máy 2 vCPU ([mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch)). Một bản sao
đầy đủ thứ hai cần thêm một `db` Guaranteed 500m CPU, một PVC và một bộ bí mật riêng — đọc
`kubectl describe node lab-k8s-worker2` mục *Allocated resources* để thấy nó không còn chỗ. Vì vậy
tenant chạy **tối thiểu**: một `web` tĩnh, đủ để kiểm từng lớp cô lập. Dựng cả stack staging là việc
của một overlay Kustomize khi có thêm node, không thuộc sprint này.

**Yêu cầu:**

1. Thư mục `k8s/tenants/staging/` là một kustomization **tự đứng** — không đọc file ngoài thư mục —
   khai mọi object của tenant. Mọi object mang label dự án và `app.kubernetes.io/instance: staging`.
2. **Lớp 1 — namespace và Pod Security:** namespace `bookstore-staging`, `enforce`, `warn`, `audit` mức
   `restricted`, ghim version.
3. **Lớp 2 — RBAC:** nhóm `bookstore-devs` (Lab 9a) được triển khai trong staging — Deployment,
   ConfigMap, Service; đọc Pod và log — nhưng **không** đọc Secret, **không** sửa quota, NetworkPolicy,
   RBAC, và **không** sửa Namespace. Ở `bookstore` nhóm này vẫn chỉ đọc như Lab 9a. Không tạo object
   RBAC phạm vi cluster nào.
4. **Lớp 3 — quota và giới hạn:** ResourceQuota `staging-quota` giới hạn số Pod, tổng
   `requests`/`limits`, **cấm** PVC, Service `NodePort` và `LoadBalancer`; LimitRange `staging-limits`
   đặt mặc định và trần cho từng container.
5. **Lớp 4 — mạng:** mặc định chặn cả hai chiều; trong namespace nói chuyện được với nhau; mọi Pod
   tra được DNS. Không selector namespace rỗng.
6. **Workload:** Deployment `web` một bản sao, `nginx-unprivileged` đúng tag A1.4, phục vụ trang chứa
   chữ `Bookstore staging` trên cổng `http` 8080, rootfs chỉ đọc, ServiceAccount riêng không mount
   token; Service `web` cổng 80.
7. `scripts/deploy.sh` và `scripts/teardown.sh` thay bằng phiên bản Lab 9b — chép nguyên văn khối bên
   dưới. `deploy.sh` áp mọi `k8s/tenants/*` **sau** `bookstore`; `teardown.sh` gỡ cả tenant.

**Ràng buộc:** không tạo ClusterRole, ClusterRoleBinding. Tenant không dùng PVC, không dùng
StorageClass `bookstore-retain`. Không Ingress cho staging ở sprint này.

**Đọc lại:**

- Bài [122](../122-multi-tenancy-vi.md):
  [*Namespace*](../122-multi-tenancy-vi.md#namespace-namespaces),
  [*Kiểm soát truy cập*](../122-multi-tenancy-vi.md#kiểm-soát-truy-cập-access-controls),
  [*Hạn ngạch*](../122-multi-tenancy-vi.md#hạn-ngạch-quotas),
  [*Cách ly mạng*](../122-multi-tenancy-vi.md#cách-ly-mạng-network-isolation),
  [*Cách ly lưu trữ*](../122-multi-tenancy-vi.md#cách-ly-lưu-trữ-storage-isolation),
  [*Namespace cho mỗi tenant*](../122-multi-tenancy-vi.md#namespace-cho-mỗi-tenant-namespace-per-tenant).
- Bài [120](../120-rbac-good-practices-vi.md): [*Tạo workload*](../120-rbac-good-practices-vi.md#workload-creation),
  [*Sửa đổi Namespace*](../120-rbac-good-practices-vi.md#namespace-modification),
  [*Từ chối dịch vụ qua việc tạo đối tượng*](../120-rbac-good-practices-vi.md#object-creation-dos).
- [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) (quota, LimitRange) và
  [Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) (NetworkPolicy) — cách bạn đã làm cho `bookstore`.

**Bàn giao:** `k8s/tenants/staging/*`; hai script mới; `docs/adr/NNNN-tenant-staging.md`;
`~/lab-evidence/9b/b8-*.txt`.

**Nghiệm thu:**

- `PASS:` `teardown.sh` sạch; `deploy.sh` chạy hai lần liên tiếp đều thành công.
- `PASS:` `bookstore-staging` mang label dự án, ba chế độ `restricted`, version bằng minor của server.
- `PASS:` lớp 1: Pod đặc quyền bị từ chối trong staging.
- `PASS:` lớp 2: `bookstore-devs` tạo được Deployment trong staging; không đọc được Secret, không sửa
  được quota, NetworkPolicy, Namespace; không tạo được Deployment trong `bookstore`.
- `PASS:` lớp 3: PVC và Service `NodePort` bị quota từ chối; số Pod đang dùng không vượt trần.
- `PASS:` lớp 4: trong staging gọi được Service `web`; ra `api` của `bookstore` thì không, dù DNS vẫn
  phân giải; Pod ở namespace khác không vào được `web` của staging.
- `PASS:` `web` của staging trả trang `Bookstore staging`, rootfs chỉ đọc, không mount token.

Hai script, chép nguyên văn:

```bash
cat > ~/bookstore/scripts/deploy.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai dự án Bookstore từ repo — phiên bản Lab 9b.
# Mới ở Lab 9b:
#  - Mỗi tenant k8s/tenants/<tên>/ (một kustomization tự đứng, namespace bookstore-<tên>) được render,
#    apply và chờ Available, SAU namespace bookstore.
#  - Cảnh báo "would violate PodSecurity" lúc apply là lỗi triển khai: namespace enforce restricted sẽ
#    không tạo được Pod của workload đó, nên dừng ngay thay vì chờ rollout hết hạn.
# Các bước còn lại giữ cơ chế từ Lab 3b tới 9a (DU-AN-BOOKSTORE.md, mục 5.5). Kubeconfig của CI
# vẫn do scripts/ci-kubeconfig.sh (Lab 9a) phát hành, không do deploy.sh.
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore
HOST=bookstore.lab.local
DATA_NODE=lab-k8s-worker2
DATA_LABEL=bookstore.example.com/data
ERR="$(mktemp)"
APP="$(mktemp)"
trap 'rm -f "$ERR" "$APP"' EXIT

render() { kubectl kustomize --load-restrictor=LoadRestrictionsNone "$1"; }
apply_dir() {   # $1 = thư mục kustomization; tham số còn lại (ví dụ -l <selector>) chuyển cho kubectl apply
  local d="$1"; shift
  if ! render "$d" | kubectl apply -f - "$@" 2>"$ERR"; then cat "$ERR" >&2; return 1; fi
  cat "$ERR" >&2
  if grep -q 'would violate PodSecurity' "$ERR"; then
    echo "deploy: $d co workload vi pham Pod Security — xem canh bao o tren" >&2
    return 1
  fi
}
wait_job() {   # thành công khi Job Complete, thất bại ngay khi Job Failed (4b)
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

# 0. Hạ tầng nền thuộc cluster, không thuộc repo: thiếu thì dừng, không tự cài.
#    Calico, IngressClass traefik, Service traefik (02-net-ready, 5b); provisioner
#    rancher.io/local-path đang chạy (03-storage-ready, 6a).
for X in 'daemonset/calico-node -n kube-system' 'ingressclass/traefik' 'service/traefik -n traefik'; do
  kubectl get $X >/dev/null 2>&1 || { echo "deploy: thieu $X - cluster chua o moc 02-net-ready"; exit 1; }
done
kubectl get storageclass -o jsonpath='{range .items[*]}{.provisioner}{"\n"}{end}' \
  | grep -qx 'rancher.io/local-path' \
  || { echo 'deploy: cluster chua co provisioner rancher.io/local-path (moc 03-storage-ready)'; exit 1; }
kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=120s

# 1. Bí mật sinh tại chỗ khi thiếu (3b, 5b); secrets/ không vào git. Dữ liệu DB cũ còn trên cluster
#    thì không sinh mới: mật khẩu thật nằm trong chính dữ liệu đó (6a).
mkdir -p secrets
if [ ! -s secrets/db.env ]; then
  CU="$( { kubectl -n "$NS" get pvc data-db-0 -o name 2>/dev/null || true
           kubectl get pv -o jsonpath='{range .items[*]}{.spec.claimRef.namespace}/{.spec.claimRef.name}{"\n"}{end}' \
             | grep -x "$NS/data-db-0" || true; } | wc -l)"
  if [ "$CU" -gt 0 ]; then
    echo 'deploy: thieu secrets/db.env trong khi du lieu db cu con tren cluster.'
    echo 'deploy: chep lai secrets/db.env cu, hoac chay scripts/teardown.sh de xoa du lieu cu.'
    exit 1
  fi
  ( umask 077
    printf 'postgres-password=%s\nauthenticator-password=%s\n' \
      "$(openssl rand -hex 24)" "$(openssl rand -hex 24)" > secrets/db.env )
  echo 'deploy: da sinh secrets/db.env moi'
fi
if [ ! -s secrets/tls.crt ] || [ ! -s secrets/tls.key ]; then
  ( umask 077
    openssl req -x509 -newkey rsa:2048 -nodes -days 365 -subj '/CN=bookstore.lab.local' \
      -addext 'subjectAltName=DNS:bookstore.lab.local' \
      -keyout secrets/tls.key -out secrets/tls.crt 2>/dev/null )
  echo 'deploy: da sinh certificate tu ky moi'
fi

# 2. Node dữ liệu (7a): đúng một node mang label của dự án; teardown.sh gỡ label này.
kubectl label node "$DATA_NODE" "${DATA_LABEL}=true" --overwrite >/dev/null
for N in $(kubectl get nodes -l "$DATA_LABEL" -o jsonpath='{.items[*].metadata.name}'); do
  [ "$N" = "$DATA_NODE" ] || kubectl label node "$N" "${DATA_LABEL}-"
done

# 3. Object cấp cluster: PriorityClass, StorageClass (3a, 6a, 7a).
if compgen -G 'k8s/cluster/*.yaml' >/dev/null; then kubectl apply -f k8s/cluster/; fi

# 4. Namespace bookstore và mọi object trong đó, TRỪ migration (6a, 7a). Template của Job không sửa
#    được sau khi tạo, nên Job migration cũ bị xóa trước (4b).
kubectl -n "$NS" delete job -l app.kubernetes.io/component=migrate --ignore-not-found --wait=true
apply_dir k8s/app -l 'app.kubernetes.io/component!=migrate'

# 5. Chờ db-0, rồi mới tạo Job migration (6a, 7a) — cũng qua kiểm Pod Security — và chờ Complete.
#    Job migration nằm ngoài Kustomize thì tạo từ file của nó.
kubectl -n "$NS" rollout status statefulset/db --timeout=300s
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s
render k8s/app > "$APP"
if grep -q 'app.kubernetes.io/component: migrate' "$APP"; then
  apply_dir k8s/app -l 'app.kubernetes.io/component=migrate'
elif [ -f k8s/app/jobs/db-migrate.yaml ]; then
  kubectl apply -f k8s/app/jobs/db-migrate.yaml
fi
wait_job db-migrate || { echo 'deploy: db-migrate that bai' >&2; exit 1; }

# 6. Chờ các controller còn lại của bookstore.
for D in web api; do kubectl -n "$NS" rollout status "deployment/$D" --timeout=300s; done
kubectl -n "$NS" rollout status daemonset/node-agent --timeout=300s

# 7. Tenant (9b): mỗi thư mục k8s/tenants/<tên>/ có ít nhất một Deployment.
for T in k8s/tenants/*/; do
  [ -f "${T}kustomization.yaml" ] || continue
  TNS="bookstore-$(basename "$T")"
  apply_dir "$T"
  kubectl -n "$TNS" wait --for=condition=Available deployment --all --timeout=180s
done

# 8. Đường vào (5b): Traefik phải phục vụ HTTPS bằng đúng certificate của dự án.
NP="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}')"
IP="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
OK=0
for _ in $(seq 1 30); do
  if curl -fsS --max-time 5 --cacert secrets/tls.crt --resolve "${HOST}:${NP}:${IP}" \
       "https://${HOST}:${NP}/healthz" >/dev/null 2>&1; then OK=1; break; fi
  sleep 2
done
[ "$OK" -eq 1 ] || { echo "deploy: https://${HOST}:${NP}/ chua phuc vu qua Traefik"; exit 1; }

kubectl get namespace -l app.kubernetes.io/part-of=bookstore -L pod-security.kubernetes.io/enforce
kubectl -n "$NS" get statefulset,deployment,daemonset,cronjob,job -o wide
EOF

cat > ~/bookstore/scripts/teardown.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ toàn bộ dự án Bookstore khỏi cluster — phiên bản Lab 9b.
# Mới ở Lab 9b: namespace tenant (bookstore-staging) và namespace thử nghiệm (bookstore-thu*) mang
# label dự án nên bị xóa cùng lượt; phần tự kiểm đếm thêm Role, RoleBinding, NetworkPolicy,
# ResourceQuota, LimitRange ở mọi namespace. Giữ nguyên secrets/ trên máy.
set -euo pipefail
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
DATA_LABEL=bookstore.example.com/data
WORKERS='lab-k8s-worker1 lab-k8s-worker2'
# Kind namespaced được dọn theo label ở mọi namespace (3b, 4a, 5a, 5b, 6a); 9b thêm Role,
# RoleBinding, ResourceQuota, LimitRange.
KINDS='statefulsets,deployments,daemonsets,cronjobs,jobs,replicasets,pods,poddisruptionbudgets,services,networkpolicies,ingresses,configmaps,secrets,persistentvolumeclaims,roles,rolebindings,resourcequotas,limitranges'
# Thư mục volume của local-path-provisioner tên <PV>_<namespace>_<PVC>: khớp namespace bookstore và
# bookstore-* (6a).
LP_FIND="sudo find /opt/local-path-provisioner -mindepth 1 -maxdepth 1 \( -name '*_bookstore_*' -o -name '*_bookstore-*_*' \)"
# Lệnh chạy trên worker. Thư mục gốc chưa từng có nghĩa là không có gì để dọn; lỗi ssh, sudo hay find
# thì trả mã khác 0 — "không kiểm được" không bao giờ được hiểu thành "không còn gì".
LP_ON_NODE="sudo -n test ! -e /opt/local-path-provisioner || $LP_FIND"
lp_left() { ssh -o BatchMode=yes "$1" "$LP_ON_NODE"; }

# 1. db tắt êm trước khi xóa namespace (4b; bài 65, mục Hạn chế).
if kubectl -n "$NS" get statefulset db >/dev/null 2>&1; then
  kubectl -n "$NS" scale statefulset db --replicas=0
  kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s || true
fi

# 2. Ghi lại PV của dự án trước khi PVC biến mất: PV cấp phát động không mang label của PVC, PV tạo
#    tay thì mang label dự án (6a).
#    Hai truy vấn chạy riêng: một cái lỗi thì dừng, không đi tiếp với danh sách thiếu.
PVA="$(kubectl get pv --no-headers \
        -o custom-columns='N:.metadata.name,NS:.spec.claimRef.namespace,SC:.spec.storageClassName' 2>/dev/null)" \
  || { echo 'teardown: STOP - khong liet ke duoc PV'; exit 1; }
PVB="$(kubectl get pv -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}')" \
  || { echo 'teardown: STOP - khong liet ke duoc PV'; exit 1; }
PVS="$( { printf '%s\n' "$PVA" | awk '$2 ~ /^bookstore/ || $3 == "bookstore-retain" {print $1}'
          printf '%s\n' "$PVB"; } | awk 'NF' | sort -u)"

# 3. Mọi namespace mang label dự án, rồi object còn sót ở namespace khác.
kubectl delete namespace -l "$SEL" --ignore-not-found --wait=true --timeout=300s
kubectl delete "$KINDS" --all-namespaces -l "$SEL" --ignore-not-found --wait=true --timeout=120s

# 4. PV Retain và thư mục dữ liệu trên node: đọc đường dẫn và node từ chính PV trước khi xóa (6a).
for PV in $PVS; do
  P="$(kubectl get pv "$PV" -o jsonpath='{.spec.hostPath.path}{.spec.local.path}' 2>/dev/null || true)"
  N="$(kubectl get pv "$PV" \
        -o jsonpath='{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}' 2>/dev/null || true)"
  kubectl delete pv "$PV" --ignore-not-found --wait=true --timeout=120s
  if [ -n "$N" ] && [ -n "$P" ]; then
    case "$P" in /opt/local-path-provisioner/?*) ssh "$N" "sudo rm -rf '$P'" ;; esac
  fi
done
#    Thư mục còn sót trên mọi worker, kể cả của PV đã mất từ trước (6a).
for W in $WORKERS; do
  ssh -o BatchMode=yes "$W" "$LP_ON_NODE -exec rm -rf {} +" \
    || { echo "teardown: STOP - khong xoa duoc thu muc du lieu tren $W (ssh, sudo hoac find loi)"; exit 1; }
done

# 5. Object cấp cluster (3a, 6a, 7a, 9a), rồi label và taint mà dự án đặt lên node (7a).
kubectl delete priorityclass,storageclass,clusterrole,clusterrolebinding -l "$SEL" --ignore-not-found
kubectl delete storageclass bookstore-retain --ignore-not-found
for N in $(kubectl get nodes -l "$DATA_LABEL" -o jsonpath='{.items[*].metadata.name}'); do
  kubectl label node "$N" "${DATA_LABEL}-" >/dev/null
done
for N in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  for K in $(kubectl get node "$N" -o jsonpath='{range .spec.taints[*]}{.key}{"\n"}{end}' \
             | grep '^bookstore\.example\.com/' | sort -u || true); do
    kubectl taint node "$N" "${K}-"
  done
done

# 6. Tự kiểm: không còn gì của dự án, cả trên cluster lẫn PV, label, taint node và thư mục dữ liệu.
# Mỗi truy vấn kiểm kê chạy riêng và phải thành công: API không trả lời hay Forbidden thì dừng —
# "không hỏi được" không bao giờ được đếm thành "không còn gì". dem <biến> <lệnh...> cộng số dòng
# output của lệnh vào biến; hàm chạy trong shell chính nên exit dừng cả script.
dem() {
  local v="$1" out; shift
  out="$("$@" 2>/dev/null)" || { echo "teardown: STOP - truy van loi, chua ket luan sach: $*"; exit 1; }
  printf -v "$v" '%d' "$(( ${!v:-0} + $(printf '%s\n' "$out" | awk 'NF' | wc -l) ))"
}
taints() {    # key của taint mà dự án đặt lên node (7a); lấy dữ liệu trước, lọc sau
  local t; t="$(kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}')" || return 1
  printf '%s\n' "$t" | awk '/^bookstore\.example\.com\//'
}
pv_claim() {  # PV từng gắn với PVC của dự án hoặc thuộc class của dự án; lấy trước, lọc sau
  local p; p="$(kubectl get pv --no-headers -o custom-columns='NS:.spec.claimRef.namespace,SC:.spec.storageClassName')" || return 1
  printf '%s\n' "$p" | awk '$1 ~ /^bookstore/ || $2 == "bookstore-retain"'
}
LEFT=0
dem LEFT taints
dem LEFT kubectl get namespace,priorityclass,storageclass,clusterrole,clusterrolebinding -l "$SEL" --no-headers
dem LEFT kubectl get storageclass bookstore-retain --no-headers --ignore-not-found
dem LEFT kubectl get "all,$KINDS,endpointslices" --all-namespaces -l "$SEL" --no-headers
dem LEFT pv_claim
dem LEFT kubectl get pv -l "$SEL" --no-headers
dem LEFT kubectl get nodes -l "$DATA_LABEL" --no-headers
#    Thư mục dữ liệu: mỗi worker cho một trong ba kết quả — còn thư mục, đã xác nhận không còn, hoặc
#    không kiểm được. Kết quả thứ ba dừng script: chưa kết luận sạch thì không được chụp snapshot.
LEFT_DIR=0
for W in $WORKERS; do
  L="$(lp_left "$W")" \
    || { echo "teardown: STOP - khong kiem duoc thu muc du lieu tren $W; chua ket luan sach, KHONG chup snapshot"; exit 1; }
  LEFT_DIR=$(( LEFT_DIR + $(printf '%s\n' "$L" | grep -c . || true) ))
done
if [ "$LEFT" -eq 0 ] && [ "$LEFT_DIR" -eq 0 ]; then
  echo 'teardown: sach'
else
  echo "teardown: con $LEFT object cua du an va $LEFT_DIR thu muc du lieu tren worker"; exit 1
fi
# Hạ tầng nền không mang label dự án và phải còn nguyên (5b).
{ kubectl get ingressclass traefik >/dev/null 2>&1 && kubectl -n kube-system get daemonset calico-node >/dev/null 2>&1; } \
  || echo 'teardown: CANH BAO - ha tang 02-net-ready khong con nguyen (khong do script nay go)'
EOF

chmod +x ~/bookstore/scripts/*.sh
```

Hai script viết lại **toàn bộ** cơ chế đã có từ Lab 3b tới 9a theo
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Nếu bản ở tag `lab-9a` của bạn làm khác ở
một khối (tên file Job migration, đường dẫn dữ liệu trên node), giữ khối đó của bản cũ. Phần bắt
buộc của Lab 9b là hàm `apply_dir` và khối 7 của `deploy.sh`; `teardown.sh` chỉ
cần xóa namespace theo label và đếm thêm năm kind namespaced trong phần tự kiểm.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-tenant-staging.md`:

1. Vì sao staging chỉ chạy `web`? Muốn chạy đủ ba tầng thì cần thêm gì (tài nguyên, bí mật, lưu trữ,
   cách đóng gói manifest), và quota hiện tại chặn chỗ nào?
2. Mỗi object trong `k8s/tenants/staging/` thuộc lớp nào của bài 122? Bài gọi lớp nào là quan trọng
   nhất, và vì sao quyền `patch` Namespace sẽ phá cả bốn lớp?
3. Quota cấm PVC. Liên hệ với mục *Cách ly lưu trữ*: PersistentVolume phạm vi gì, và
   `bookstore-retain` có `reclaimPolicy` gì?
4. Mô hình này **không** bảo vệ được điều gì? Bạn đang ở đâu trên phổ "mềm — cứng", và tenant chạy mã
   không đáng tin cậy thì phải thêm lớp nào?

<details>
<summary>Gợi ý</summary>

1. `kubectl kustomize k8s/tenants/staging` (không cần cờ nới lỏng) phải render được — đó là định
   nghĩa của "tự đứng".
2. Quyền cho nhóm: Role và RoleBinding trong `bookstore-staging`, `subjects` kind `Group`. Kiểm bằng
   `kubectl auth can-i ... --as=dev-an --as-group=bookstore-devs`.
3. Quota đếm object: `persistentvolumeclaims`, `services.nodeports`, `services.loadbalancers`. Có quota
   CPU/bộ nhớ thì mọi container phải khai `requests`/`limits` — LimitRange điền hộ.
4. DNS: cho phép egress tới Pod `k8s-app: kube-dns` trong namespace có nhãn
   `kubernetes.io/metadata.name: kube-system`, cổng 53 UDP và TCP.
5. Kiểm mạng bằng hai Pod tạm: một trong staging (phải đạt `restricted`), một trong `bookstore-thu`.
   `wget -T 3` để không chờ lâu.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```yaml
# ~/lab-work/9b/b8-thu-client.yaml — Pod tạm trong staging, đạt restricted
apiVersion: v1
kind: Pod
metadata: {name: thu-client, namespace: bookstore-staging}
spec:
  automountServiceAccountToken: false
  securityContext: {runAsNonRoot: true, runAsUser: 65534, runAsGroup: 65534, seccompProfile: {type: RuntimeDefault}}
  containers:
  - name: c
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    securityContext: {allowPrivilegeEscalation: false, capabilities: {drop: ["ALL"]}}
---
apiVersion: v1
kind: Pod
metadata: {name: thu-ngoai, namespace: bookstore-thu}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  containers: [{name: c, image: busybox:1.37, command: ["sh", "-c", "sleep 3600"]}]
```

```bash
. ~/lab-work/9b/ham.sh; cd ~/bookstore
./scripts/teardown.sh > "$EV/b8-teardown.txt" 2>&1 && grep -q 'teardown: sach' "$EV/b8-teardown.txt" \
  && echo 'PASS: teardown.sh go sach'
./scripts/deploy.sh > "$EV/b8-deploy-1.txt" 2>&1 && ./scripts/deploy.sh > "$EV/b8-deploy-2.txt" 2>&1 \
  && echo 'PASS: deploy.sh chay hai lan lien tiep thanh cong'
kubectl get ns "$THU" >/dev/null 2>&1 || kubectl apply -f "$WK/b1-namespaces.yaml" >/dev/null

tl() { kubectl get ns "$TNS" -o jsonpath="{.metadata.labels.$1}" 2>/dev/null; }
BAD=0
for M in enforce warn audit; do
  { [ "$(tl "pod-security\.kubernetes\.io/$M")" = 'restricted' ] && \
    [ "$(tl "pod-security\.kubernetes\.io/$M-version")" = "$PSA_VER" ]; } || BAD=1
done
{ [ "$BAD" -eq 0 ] && [ "$(tl 'app\.kubernetes\.io/part-of')" = 'bookstore' ]; } \
  && echo "PASS: $TNS mang label du an, restricted o ca ba che do"

# Lop 1
printf '%s\n' 'apiVersion: v1' 'kind: Pod' 'metadata: {name: thu-dac-quyen}' 'spec:' \
  '  containers: [{name: c, image: busybox:1.37, securityContext: {privileged: true}}]' \
  | kubectl -n "$TNS" create --dry-run=server -f - > "$EV/b8-lop1.txt" 2>&1
grep -q "violates PodSecurity \"restricted:$PSA_VER\"" "$EV/b8-lop1.txt" && echo 'PASS: lop 1 chan Pod dac quyen'

# Lop 2
DEV='--as=dev-an --as-group=bookstore-devs'
ci() { kubectl auth can-i $DEV "$@" 2>/dev/null; }
{
  for Q in "create deployments -n $TNS" "get secrets -n $TNS" "delete resourcequotas -n $TNS" \
           "create networkpolicies -n $TNS" "patch namespace/$TNS" "create deployments -n $NS" "list pods -n $NS"; do
    printf '%-44s %s\n' "$Q" "$(ci $Q)"
  done
} | tee "$EV/b8-lop2.txt"
{ [ "$(ci create deployments -n "$TNS")" = yes ] && [ "$(ci get secrets -n "$TNS")" = no ] && \
  [ "$(ci delete resourcequotas -n "$TNS")" = no ] && [ "$(ci create networkpolicies -n "$TNS")" = no ] && \
  [ "$(ci patch "namespace/$TNS")" = no ] && [ "$(ci create deployments -n "$NS")" = no ]; } \
  && echo 'PASS: lop 2 — devs trien khai duoc trong staging, khong cham Secret, quota, policy, Namespace, bookstore'
[ -z "$(kubectl get clusterrole,clusterrolebinding -l 'app.kubernetes.io/instance=staging' -o name 2>/dev/null)" ] \
  && echo 'PASS: tenant khong tao object RBAC pham vi cluster'

# Lop 3
printf '%s\n' 'apiVersion: v1' 'kind: PersistentVolumeClaim' 'metadata: {name: thu-pvc}' \
  'spec: {accessModes: [ReadWriteOnce], resources: {requests: {storage: 1Gi}}}' \
  | kubectl -n "$TNS" create --dry-run=server -f - > "$EV/b8-lop3-pvc.txt" 2>&1
printf '%s\n' 'apiVersion: v1' 'kind: Service' 'metadata: {name: thu-np}' \
  'spec: {type: NodePort, selector: {app: x}, ports: [{port: 80}]}' \
  | kubectl -n "$TNS" create --dry-run=server -f - > "$EV/b8-lop3-np.txt" 2>&1
{ grep -q 'exceeded quota' "$EV/b8-lop3-pvc.txt" && grep -q 'exceeded quota' "$EV/b8-lop3-np.txt"; } \
  && echo 'PASS: lop 3 — quota tu choi PVC va NodePort'
U="$(kubectl -n "$TNS" get resourcequota staging-quota -o jsonpath='{.status.used.pods}')"
H="$(kubectl -n "$TNS" get resourcequota staging-quota -o jsonpath='{.status.hard.pods}')"
{ [ -n "$H" ] && [ "${U:-0}" -le "$H" ] && kubectl -n "$TNS" get limitrange staging-limits >/dev/null 2>&1; } \
  && echo "PASS: quota pods $U/$H, co LimitRange staging-limits"

# Lop 4
kubectl apply -f "$WK/b8-thu-client.yaml" > "$EV/b8-thu-client.txt" 2>&1
kubectl -n "$TNS" wait --for=condition=Ready pod/thu-client --timeout=120s >/dev/null
kubectl -n "$THU" wait --for=condition=Ready pod/thu-ngoai --timeout=120s >/dev/null
SIP="$(kubectl -n "$TNS" get pod -l app.kubernetes.io/component=web -o jsonpath='{.items[0].status.podIP}')"
ex() { kubectl -n "$1" exec "$2" -- sh -c "$3" >/dev/null 2>&1 && echo thong || echo chan; }
TRONG="$(ex "$TNS" thu-client 'wget -q -T 3 -O- http://web/ | grep -q "Bookstore staging"')"
DNS="$(ex "$TNS" thu-client 'nslookup api.bookstore.svc.cluster.local')"
RA="$(ex "$TNS" thu-client 'wget -q -T 3 -O- http://api.bookstore.svc.cluster.local:3000/books')"
VAO="$(ex "$THU" thu-ngoai "wget -q -T 3 -O- http://$SIP:8080/")"
NGOAI_DNS="$(ex "$THU" thu-ngoai 'nslookup kubernetes.default.svc.cluster.local')"
echo "trong=$TRONG dns=$DNS ra-bookstore=$RA ngoai-vao=$VAO ngoai-dns=$NGOAI_DNS" | tee "$EV/b8-lop4.txt"
{ [ "$TRONG" = thong ] && [ "$DNS" = thong ] && [ "$RA" = chan ]; } \
  && echo 'PASS: lop 4 — trong staging thong, DNS thong, ra bookstore bi chan'
{ [ "$VAO" = chan ] && [ "$NGOAI_DNS" = thong ]; } \
  && echo 'PASS: lop 4 — Pod namespace khac khong vao duoc staging, du mang cua no van chay'
kubectl delete -f "$WK/b8-thu-client.yaml" --wait=true >/dev/null

WP="$(kubectl -n "$TNS" get pod -l app.kubernetes.io/component=web -o jsonpath='{.items[0].metadata.name}')"
{ [ "$(kubectl -n "$TNS" get pod "$WP" -o jsonpath='{.spec.containers[0].securityContext.readOnlyRootFilesystem}')" = 'true' ] && \
  ! kubectl -n "$TNS" get pod "$WP" -o jsonpath='{.spec.containers[0].volumeMounts[*].mountPath}' | grep -q serviceaccount; } \
  && echo 'PASS: web cua staging rootfs chi doc, khong mount token'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/tenants/staging/kustomization.yaml — tenant thứ hai (9b), tự đứng
resources:
- namespace.yaml
- rbac.yaml
- policy.yaml
- network.yaml
- web.yaml
```

```yaml
# k8s/tenants/staging/namespace.yaml — lớp 1
apiVersion: v1
kind: Namespace
metadata:
  name: bookstore-staging
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/instance: staging
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.35      # minor của cluster — đối chiếu A1.3
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: v1.35
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: v1.35
```

```yaml
# k8s/tenants/staging/rbac.yaml — lớp 2; chỉ object thuộc namespace
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web
  namespace: bookstore-staging
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/instance: staging}
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: staging-developer
  namespace: bookstore-staging
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/instance: staging}
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["replicasets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps", "services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods", "pods/log", "events"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: staging-developer
  namespace: bookstore-staging
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/instance: staging}
subjects:
- {kind: Group, name: bookstore-devs, apiGroup: rbac.authorization.k8s.io}
roleRef: {kind: Role, name: staging-developer, apiGroup: rbac.authorization.k8s.io}
```

```yaml
# k8s/tenants/staging/policy.yaml — lớp 3
apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: bookstore-staging
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/instance: staging}
spec:
  hard:
    pods: "4"
    requests.cpu: 200m
    requests.memory: 256Mi
    limits.cpu: 500m
    limits.memory: 512Mi
    persistentvolumeclaims: "0"
    services.nodeports: "0"
    services.loadbalancers: "0"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: staging-limits
  namespace: bookstore-staging
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/instance: staging}
spec:
  limits:
  - type: Container
    default: {cpu: 100m, memory: 64Mi}
    defaultRequest: {cpu: 25m, memory: 32Mi}
    max: {cpu: 200m, memory: 128Mi}
```

```yaml
# k8s/tenants/staging/network.yaml — lớp 4
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: bookstore-staging
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/instance: staging}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: bookstore-staging
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/instance: staging}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress: [{from: [{podSelector: {}}]}]
  egress: [{to: [{podSelector: {}}]}]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: bookstore-staging
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/instance: staging}
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: kube-system}}
      podSelector: {matchLabels: {k8s-app: kube-dns}}
    ports: [{protocol: UDP, port: 53}, {protocol: TCP, port: 53}]
```

```yaml
# k8s/tenants/staging/web.yaml — workload tối thiểu của tenant
apiVersion: v1
kind: ConfigMap
metadata:
  name: staging-page
  namespace: bookstore-staging
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: web, app.kubernetes.io/instance: staging}
data:
  index.html: |
    <!doctype html>
    <html lang="vi"><head><meta charset="utf-8"><title>Bookstore staging</title></head>
    <body><h1>Bookstore staging</h1><p>Tenant thử nghiệm, không có dữ liệu thật.</p></body></html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: bookstore-staging
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
    app.kubernetes.io/name: nginx
    app.kubernetes.io/instance: staging
spec:
  replicas: 1
  selector:
    matchLabels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: web}
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: web
        app.kubernetes.io/name: nginx
        app.kubernetes.io/instance: staging
    spec:
      serviceAccountName: web
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
        seccompProfile: {type: RuntimeDefault}
      volumes:
      - {name: tmp, emptyDir: {}}
      - {name: page, configMap: {name: staging-page}}
      containers:
      - name: nginx
        image: nginxinc/nginx-unprivileged:1.30.5-alpine
        ports: [{name: http, containerPort: 8080}]
        readinessProbe:
          httpGet: {path: /, port: http}
          periodSeconds: 5
        resources:
          requests: {cpu: 25m, memory: 32Mi}
          limits: {cpu: 100m, memory: 64Mi}
        securityContext:
          allowPrivilegeEscalation: false
          capabilities: {drop: ["ALL"]}
          readOnlyRootFilesystem: true
        volumeMounts:
        - {name: tmp, mountPath: /tmp}
        - {name: page, mountPath: /usr/share/nginx/html, readOnly: true}
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: bookstore-staging
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: web, app.kubernetes.io/instance: staging}
spec:
  selector: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: web}
  ports: [{name: http, port: 80, targetPort: http}]
```

Ý chính cho ADR:

1. Ngân sách: `db` của `bookstore` đã giữ 500m CPU Guaranteed trên `lab-k8s-worker2`; một `db` thứ
   hai, PVC thứ hai và bộ bí mật thứ hai không vừa, và quota staging cố ý cấm PVC. Đủ ba tầng cần một
   overlay Kustomize của `k8s/app` (namespace, tên ConfigMap, Secret riêng, `replicas: 1`, nới quota),
   cộng thêm node. Sprint này chỉ cần một workload để kiểm bốn lớp.
2. Namespace + nhãn Pod Security: lớp 1. ServiceAccount, Role, RoleBinding: lớp 2 — bài 122 gọi phân
   quyền là cách ly **quan trọng nhất** của control plane. ResourceQuota, LimitRange: lớp 3.
   NetworkPolicy: lớp 4. Ai `patch` được Namespace thì hạ được mức Pod Security, đổi được nhãn mà
   NetworkPolicy dựa vào — ba lớp còn lại không cứu được.
3. PersistentVolume phạm vi cluster, vòng đời độc lập; `bookstore-retain` là `Retain`, nên PV cũ của
   tenant này có thể nằm lại và bị gắn nhầm. Bài 122 khuyên `Delete` khi dùng chung StorageClass;
   staging chọn cách chặt hơn: không PVC nào.
4. Data plane vẫn chung: chung kernel, chung kubelet, chung node; DNS cho tra cứu liên namespace (DNS
   vẫn phân giải `api.bookstore`); APF chung API server. Đây là mềm — hợp với hai môi trường của cùng
   một đội. Mã không đáng tin cần sandbox (RuntimeClass) hoặc cluster riêng.

</details>

## B9. [BS-9B-09] Điều tra: ai chạm được Secret của Bookstore

**Loại:** Điều tra.

**Bối cảnh:** `db-credentials` là bí mật đắt nhất của hệ thống. Lab 9a cấp quyền tối thiểu cho từng
danh tính; bài 121 và 120 nhắc rằng RBAC chỉ trả lời **một nửa** câu hỏi "ai đọc được Secret": nửa
còn lại là ai **tạo được workload** dùng Secret đó, Secret nằm ở đâu trên node, và nó được lưu thế nào
trong etcd. Kết quả ghi vào phần đầu của `docs/security-review.md`.

**Yêu cầu:**

1. Với **mọi** danh tính của dự án — ServiceAccount của từng workload, `ci-deployer`, nhóm
   `bookstore-devs` và `bookstore-oncall` — trả lời bằng chính API server: đọc được `db-credentials` không, `list` Secret
   trong `bookstore` được không, sửa Namespace `bookstore` được không, có quyền `get` trên `nodes/proxy`
   không.
2. Chỉ ra danh tính nào **gián tiếp** chạm được Secret — qua quyền sửa workload (`TD-15` của Lab 9a)
   hoặc quyền `exec` (`TD-07`) — và ghi trạng thái **sau sprint này** của hai nợ đó vào
   `docs/tech-debt.md`: `restricted` đổi được gì, không đổi được gì.
3. Liệt kê container nào nhận `db-credentials` — qua volume hay biến môi trường — và chứng minh các
   container của `web` và `node-agent` không nhận.
4. Chứng minh Secret mount vào `db-0` nằm trên `tmpfs` và ba node không có swap.
5. Chứng minh `web`, `api`, `db` không mount token ServiceAccount.
6. Ghi nhận trạng thái mã hóa Secret at rest; nợ lab #6 **vẫn mở**.
7. `docs/security-review.md` có phần *Secret và RBAC* với kết luận cho từng điểm trên.

**Ràng buộc:** không đọc giá trị Secret ra màn hình hay vào bằng chứng. Mọi phép thử quyền dùng
`kubectl auth can-i` hoặc server dry-run. Không sửa RBAC của Lab 9a ở ticket này.

**Đọc lại:**

- Bài [121](../121-secrets-good-practices-vi.md):
  [*Cấu hình truy cập đặc quyền tối thiểu cho Secret*](../121-secrets-good-practices-vi.md#least-privilege-secrets),
  [*Giới hạn quyền truy cập Secret cho các container cụ thể*](../121-secrets-good-practices-vi.md#giới-hạn-quyền-truy-cập-secret-cho-các-container-cụ-thể-restrict-secret-access-to-specific-containers).
- Bài [120](../120-rbac-good-practices-vi.md):
  [*Liệt kê secret*](../120-rbac-good-practices-vi.md#liệt-kê-secret-listing-secrets),
  [*Tạo workload*](../120-rbac-good-practices-vi.md#workload-creation),
  [*Truy cập subresource `proxy` của Node*](../120-rbac-good-practices-vi.md#truy-cập-subresource-proxy-của-node-access-to-proxy-subresource-of-nodes),
  [*Sửa đổi Namespace*](../120-rbac-good-practices-vi.md#namespace-modification).
- Bài [126](../126-linux-security-vi.md#bảo-vệ-dữ-liệu-secret-trên-node-protection-for-secret-data-on-nodes).
- Bài [286](../286-migrate-from-psp-vi.md#review-namespace-permissions) — bước 1.
- Bài [130](../130-application-security-checklist-vi.md#service-account) — `automountServiceAccountToken`.

**Bàn giao:** `docs/security-review.md` (phần *Secret và RBAC*); `docs/tech-debt.md` (`TD-07`, `TD-15`);
`~/lab-evidence/9b/b9-*.txt`.

**Nghiệm thu:**

- `PASS:` không danh tính nào của dự án `get` hay `list` được Secret trong `bookstore`.
- `PASS:` không danh tính nào của dự án `patch` được Namespace `bookstore` hay `get` được `nodes/proxy`.
- `PASS:` `ci-deployer` sửa được Deployment, `bookstore-oncall` `exec` được — hai đường đọc gián tiếp —
  và `docs/tech-debt.md` ghi trạng thái của `TD-07`, `TD-15` sau Lab 9b.
- `PASS:` chỉ container `postgres`, `postgrest` (và container của Job) nhận `db-credentials`; container
  của `web` và `node-agent` không nhận.
- `PASS:` Secret trong `db-0` là `tmpfs`; ba node không có swap.
- `PASS:` `web`, `api`, `db-0` không mount token ServiceAccount.
- `PASS:` apiserver không có `--encryption-provider-config` — ghi nhận nợ lab #6.
- `PASS:` `docs/security-review.md` có phần *Secret và RBAC*, nhắc `TD-15` và nợ lab #6.

**Câu hỏi thiết kế** — trả lời trong phần *Secret và RBAC* của `docs/security-review.md`:

1. Vì sao "không có `get` trên Secret" chưa đủ để nói một danh tính không đọc được mật khẩu DB? Kể hai
   đường còn lại mà bài 120 và 121 chỉ ra.
2. `restricted` có làm gì được với đường đọc gián tiếp của `ci-deployer` không? Nó chặn được loại tấn
   công nào mà trước sprint này `ci-deployer` làm được?
3. Bài 121 đề xuất giảm thiểu bằng gì? Cái nào dự án đã có, cái nào thuộc giai đoạn sau?
4. "Secret nằm trên `tmpfs` nên không bao giờ chạm đĩa" — đúng trong điều kiện nào trên cluster này?

<details>
<summary>Gợi ý</summary>

1. ServiceAccount của mỗi Pod: `.spec.serviceAccountName`; của CronJob: trong `jobTemplate`. Hỏi API
   server: `kubectl auth can-i get secret/db-credentials -n bookstore --as=system:serviceaccount:bookstore:<sa>`.
2. Subresource: `kubectl auth can-i get nodes --subresource=proxy --as=...`. Một Namespace cụ thể:
   `kubectl auth can-i patch namespace/bookstore --as=...`.
3. ⭐ Chứng minh đường gián tiếp mà không đụng gì: `kubectl -n bookstore patch deployment web
   --dry-run=server --as=system:serviceaccount:bookstore:ci-deployer -p '<thêm volume secret>'` được
   chấp nhận nghĩa là CI gắn được Secret vào Pod của nó.
4. Container nào nhận Secret: đọc `volumes[].secret`, `volumeMounts`, `env[].valueFrom.secretKeyRef`,
   `envFrom[].secretRef` của từng container — node có `python3` để đọc JSON nếu bạn không muốn viết
   jsonpath dài.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-work/9b/ham.sh; cd ~/bookstore
DEV='--as=dev-an --as-group=bookstore-devs'; ONC='--as=truc --as-group=bookstore-oncall'
SAS="$( { kubectl -n "$NS" get pods -l app.kubernetes.io/part-of=bookstore \
            -o jsonpath='{range .items[*]}{.spec.serviceAccountName}{"\n"}{end}'
          jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.serviceAccountName}'; echo
          echo ci-deployer; } | grep -v '^$' | sort -u)"
ci() { kubectl auth can-i "$@" 2>/dev/null; }
BAD=0
{
  printf '%-52s %-7s %-7s %-7s %-7s %-7s\n' 'DANH TINH' 'get' 'list' 'patchNS' 'proxy' 'updDep'
  for A in $(for S in $SAS; do echo "system:serviceaccount:$NS:$S"; done) DEV ONC; do
    case "$A" in DEV) AS="$DEV" ;; ONC) AS="$ONC" ;; *) AS="--as=$A" ;; esac
    G="$(ci get secret/db-credentials -n "$NS" $AS)"; L="$(ci list secrets -n "$NS" $AS)"
    P="$(ci patch namespace/"$NS" $AS)"; X="$(ci get nodes --subresource=proxy $AS)"
    D="$(ci update deployments -n "$NS" $AS)"
    printf '%-52s %-7s %-7s %-7s %-7s %-7s\n' "$A" "$G" "$L" "$P" "$X" "$D"
    [ "$G$L$P$X" = 'nononono' ] || BAD=1
  done
} | tee "$EV/b9-quyen.txt"
grep -v '^DANH' "$EV/b9-quyen.txt" | awk '$2!="no"||$3!="no"{f=1} END{exit f}' \
  && echo 'PASS: khong danh tinh nao cua du an get hay list duoc Secret'
grep -v '^DANH' "$EV/b9-quyen.txt" | awk '$4!="no"||$5!="no"{f=1} END{exit f}' \
  && echo 'PASS: khong danh tinh nao patch duoc Namespace bookstore hay get duoc nodes/proxy'
{ { [ "$(ci update deployments -n "$NS" --as="$CI")" = yes ] || [ "$(ci patch deployments -n "$NS" --as="$CI")" = yes ]; } && \
  [ "$(ci create pods --subresource=exec -n "$NS" $ONC)" = yes ] && \
  grep 'TD-15' docs/tech-debt.md | grep -q '9b' && grep 'TD-07' docs/tech-debt.md | grep -q '9b'; } \
  && echo 'PASS: hai duong doc gian tiep (TD-15, TD-07) da ghi trang thai sau 9b'

kubectl -n "$NS" get pods -o json | python3 -c '
import json, sys
for p in json.load(sys.stdin)["items"]:
    s = p["spec"]
    vols = {v["name"] for v in s.get("volumes", [])
            if v.get("secret", {}).get("secretName") == "db-credentials"
            or any(x.get("secret", {}).get("name") == "db-credentials" for x in v.get("projected", {}).get("sources", []))}
    for c in s.get("initContainers", []) + s["containers"]:
        e = any(x.get("valueFrom", {}).get("secretKeyRef", {}).get("name") == "db-credentials" for x in c.get("env", []))
        f = any(x.get("secretRef", {}).get("name") == "db-credentials" for x in c.get("envFrom", []))
        m = any(x["name"] in vols for x in c.get("volumeMounts", []))
        print("%-32s %-16s %s" % (p["metadata"]["name"], c["name"], "NHAN" if (e or f or m) else "-"))
' | tee "$EV/b9-container-secret.txt"
{ awk '$3=="NHAN"{print $2}' "$EV/b9-container-secret.txt" | grep -qx postgres && \
  awk '$3=="NHAN"{print $2}' "$EV/b9-container-secret.txt" | grep -qx postgrest && \
  ! awk '$3=="NHAN"{print $2}' "$EV/b9-container-secret.txt" | grep -Eqx 'nginx|status-writer|render-html|agent'; } \
  && echo 'PASS: Secret chi toi postgres, postgrest; web va node-agent khong nhan'

MNT="$(jp pod/db-0 '{.spec.containers[?(@.name=="postgres")].volumeMounts[?(@.name=="credentials")].mountPath}')"
kubectl -n "$NS" exec db-0 -c postgres -- grep " ${MNT:-/etc/db-credentials} " /proc/mounts > "$EV/b9-tmpfs.txt" 2>&1
# Swap trên ba node: mỗi lần đọc phải thành công, đọc lỗi không được đếm thành "không có swap".
SW=0; SWOK=1
O="$(swapon --show --noheadings)" || SWOK=0; SW=$(( SW + $(printf '%s\n' "$O" | grep -c . || true) ))
for N in lab-k8s-worker1 lab-k8s-worker2; do
  O="$(ssh -o BatchMode=yes "$N" 'swapon --show --noheadings')" || SWOK=0
  SW=$(( SW + $(printf '%s\n' "$O" | grep -c . || true) ))
done
{ grep -q ' tmpfs ' "$EV/b9-tmpfs.txt" && [ "$SWOK" -eq 1 ] && [ "$SW" -eq 0 ]; } && echo 'PASS: Secret tren tmpfs, ba node khong co swap'

BAD=0
for P in "$(pod_of web)" "$(pod_of api)" db-0; do
  kubectl -n "$NS" get pod "$P" -o jsonpath='{.spec.containers[*].volumeMounts[*].mountPath}' \
    | grep -q 'kubernetes.io/serviceaccount' && { echo "FAIL: $P mount token"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: web, api, db-0 khong mount token ServiceAccount'
[ "$(sudo grep -c -- '--encryption-provider-config' /etc/kubernetes/manifests/kube-apiserver.yaml)" = '0' ] \
  && echo 'PASS: chua ma hoa Secret at rest — no lab #6 van mo, da ghi nhan'
{ grep -q '^## .*Secret và RBAC' docs/security-review.md && grep -q 'TD-15' docs/security-review.md && \
  grep -q '#6' docs/security-review.md; } && echo 'PASS: security-review.md co phan Secret va RBAC'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Cập nhật cột trạng thái của hai dòng có sẵn từ Lab 9a trong `docs/tech-debt.md`:

```markdown
| TD-07 | ... | 3a | 3b, 9a | đã giới hạn ở 9a; 9b: container gỡ lỗi chạy non-root, không capability — người trực vẫn đọc được biến môi trường và file Secret của Pod mình exec | ~/lab-evidence/9b/b9-quyen.txt |
| TD-15 | ... | 9a | ... | mở — 9b: restricted chặn CI chạy Pod đặc quyền, hostPath, namespace của host; phần gắn Secret và mượn ServiceAccount giữ nguyên, xem lại ở giai đoạn 14 | ~/lab-evidence/9b/b9-quyen.txt |
```

Phần đầu của `docs/security-review.md`:

```markdown
# Rà soát bảo mật Bookstore — Lab 9b

Ngày: <ngày>. Cluster: snapshot 03-storage-ready, repo tag lab-9b. Bằng chứng: ~/lab-evidence/9b/.

## 1. Secret và RBAC

| Điểm | Kết quả | Bằng chứng | Quyết định |
| --- | --- | --- | --- |
| get/list Secret trong bookstore | không danh tính nào của dự án | b9-quyen.txt | giữ |
| Đọc gián tiếp qua workload | ci-deployer sửa được Deployment | b9-quyen.txt | TD-15 (9a), chấp nhận có điều kiện |
| Đọc gián tiếp qua exec | bookstore-oncall exec được vào Pod | b9-quyen.txt | TD-07, đã giới hạn ở 9a |
| patch Namespace bookstore | không ai ngoài admin | b9-quyen.txt | giữ — đây là bước 1 của bài 286 |
| nodes/proxy | không ai | b9-quyen.txt | giữ |
| Container nhận db-credentials | postgres, postgrest, Job migrate/backup | b9-container-secret.txt | giữ |
| Secret trên node | tmpfs; swap tắt trên 3 node | b9-tmpfs.txt | giữ; bật swap thì cần kernel có noswap |
| Token ServiceAccount | web, api, db không mount | b9-* | giữ |
| Mã hóa at rest | chưa bật | grep manifest apiserver | nợ lab #6, giai đoạn 22 |
```

Ý chính cho câu hỏi:

1. `list` (và `watch`) trả về **nội dung** mọi Secret; và ai tạo hoặc sửa được workload trong
   namespace thì gắn được mọi Secret của namespace vào Pod của mình (bài 120, *Tạo workload*; bài 121).
2. Không chặn được đường đó — Pod gắn Secret vẫn đạt `restricted`. Nhưng nó chặn `ci-deployer` chạy Pod
   đặc quyền, `hostPath`, namespace của host, tức là chặn đường chiếm **node** từ quyền tạo workload.
3. Secret thời gian sống ngắn và audit rule cảnh báo đọc Secret bất thường; namespace riêng để cách ly
   Secret được mount. Dự án có: namespace riêng cho staging, token CI ngắn hạn (9a). Audit rule cần
   audit backend — giai đoạn 22; chặn hẳn thay đổi pod template của CI cần admission policy — giai
   đoạn 14, đúng chỗ `TD-15` hẹn xem lại.
4. Chỉ khi node không có swap (bài 126). Ba node của lab tắt swap theo yêu cầu kubeadm; bật swap thì
   phải có kernel hỗ trợ `noswap`.

</details>

## B10. [BS-9B-10] Điều tra: tầng API server — admission, PSP, webhook, APF

**Loại:** Điều tra.

**Bối cảnh:** mọi chính sách của sprint này chạy **bên trong** API server. Người vận hành phải đọc được
ba thứ ở tầng đó mà không cài gì: chuỗi admission thật đang chạy, tồn kho admission webhook cùng
`failurePolicy` của chúng, và cách API Priority and Fairness phân loại request của chính dự án — một CI
lỗi bắn dồn request thì có làm đói kubelet không.

**Yêu cầu:**

1. Liệt kê admission plugin đang hoạt động từ số liệu của chính API server: `PodSecurity` có mặt,
   `PodSecurityPolicy` không. Chứng minh apiserver không có file cấu hình admission riêng — bộ mặc định
   của bài 282 đang có hiệu lực.
2. Chứng minh PodSecurityPolicy đã bị gỡ khỏi API: nhóm `policy` không còn kind đó, phiên bản
   `policy/v1beta1` không tồn tại, manifest `privileged-psp.yaml` của bài 286 không apply được (dry-run),
   trong khi `policy/v1` vẫn còn và phục vụ gì.
3. Tồn kho mutating/validating webhook của cluster cùng `failurePolicy`, `timeoutSeconds` của từng cái.
   Gửi một `ValidatingWebhookConfiguration` không khai `failurePolicy` bằng **server dry-run** và đọc
   giá trị mặc định server điền vào. Số webhook trên cluster không đổi.
4. Đọc FlowSchema và PriorityLevelConfiguration thật: đủ bốn đối tượng bắt buộc, sáu mức được đề xuất.
   Chỉ ra request của **admin** (kubeconfig của bạn), của **CI** (`ci-deployer`) và của **đội phát
   triển** (`bookstore-devs`) rơi vào FlowSchema và mức ưu tiên nào, đọc từ header phản hồi của API
   server; giải thích bằng nhóm trong certificate admin.
5. ADR: dự án có nên viết mutating webhook để tự điền `securityContext` cho mọi Pod không.

**Ràng buộc:** không tạo, sửa FlowSchema hay PriorityLevelConfiguration. Mọi cấu hình webhook chỉ đi
qua `--dry-run=server`. Không sửa manifest apiserver.

**Đọc lại:**

- Bài [282](../282-enforce-standards-admission-controller-vi.md#configure-the-admission-controller);
  bài [286](../286-migrate-from-psp-vi.md#disable-psp) — bước 5 và danh sách plugin; bài
  [117](../117-pod-security-policy-vi.md).
- Bài [173](../173-admission-webhooks-vi.md):
  [*Xác định xem bạn có dùng admission webhook hay không*](../173-admission-webhooks-vi.md#identify-admission-webhooks),
  [*"Fail open" và kiểm tra hợp lệ trạng thái cuối cùng*](../173-admission-webhooks-vi.md#fail-open-validate-final-state),
  [*Giới hạn phạm vi*](../173-admission-webhooks-vi.md#webhook-limit-scope),
  [*Tránh tự biến đổi chính mình*](../173-admission-webhooks-vi.md#avoid-self-mutation).
- Bài [166](../166-flow-control-vi.md):
  [*FlowSchema*](../166-flow-control-vi.md#flowschema),
  [*Các đối tượng cấu hình bắt buộc*](../166-flow-control-vi.md#các-đối-tượng-cấu-hình-bắt-buộc-mandatory-configuration-objects),
  [*Các đối tượng cấu hình được đề xuất*](../166-flow-control-vi.md#các-đối-tượng-cấu-hình-được-đề-xuất-suggested-configuration-objects).
- Bài [123](../123-hardening-authentication-vi.md#x509-client-certificate-authentication) — nhóm nằm
  trong trường `O`; bài [129](../129-security-checklist-vi.md) — ô đầu tiên về `system:masters`.

**Bàn giao:** `docs/adr/NNNN-tang-api-server.md`; `~/lab-evidence/9b/b10-*.txt`.

**Nghiệm thu:**

- `PASS:` danh sách plugin có `PodSecurity`, `ResourceQuota`, `LimitRanger`, không có
  `PodSecurityPolicy`; apiserver không có `--admission-control-config-file`.
- `PASS:` nhóm `policy` không còn `podsecuritypolicies`, `/apis/policy/v1beta1` không tồn tại,
  manifest PSP bị từ chối `no matches for kind`, `policy/v1` vẫn phục vụ `poddisruptionbudgets`.
- `PASS:` ghi được tồn kho webhook; số webhook bằng con số ở B0.
- `PASS:` server dry-run điền `failurePolicy: Fail` và `timeoutSeconds: 10` khi không khai.
- `PASS:` đủ bốn đối tượng APF bắt buộc và sáu mức được đề xuất.
- `PASS:` request của CI rơi vào `service-accounts` / `workload-low`; của đội phát triển vào
  `global-default`; của admin khớp với nhóm trong certificate (`exempt` chỉ khi nhóm là
  `system:masters`).
- `PASS:` ADR tồn tại.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-tang-api-server.md`:

1. Pipeline CI lỗi bắn hàng nghìn request mỗi giây vào API server. Theo cấu hình được đề xuất, kubelet
   của hai worker và việc bầu leader có bị ảnh hưởng không? Cơ chế nào giữ chúng?
2. Kubeconfig admin của bạn rơi vào FlowSchema nào, vì sao? Liên hệ ô đầu tiên của checklist 129 về
   `system:masters`.
3. PSA không mutating, nên mọi manifest phải tự viết `securityContext`. Vì sao dự án **không** viết
   mutating webhook để điền hộ? Nếu buộc phải viết, bài 173 dặn đặt `failurePolicy`, `timeoutSeconds`,
   `namespaceSelector` thế nào?
4. Trong năm bước di trú của bài 286, bước nào còn giá trị với cluster chưa từng có PSP, và Lab 9b đã
   làm chúng ở ticket nào?

<details>
<summary>Gợi ý</summary>

1. Mỗi plugin từng chạy có một dòng metric
   `apiserver_admission_controller_admission_duration_seconds_count{name="..."}` trong
   `kubectl get --raw /metrics`.
2. API server trả hai header `X-Kubernetes-Pf-Flowschema-Uid` và `X-Kubernetes-Pf-Prioritylevel-Uid`;
   `kubectl ... -v=8` in header phản hồi. Ghép uid với `kubectl get flowschema -o
   jsonpath='{range .items[*]}{.metadata.uid} {.metadata.name}{"\n"}{end}'`.
3. Đóng vai CI hay đội phát triển: `--as=system:serviceaccount:bookstore:ci-deployer`,
   `--as=dev-an --as-group=bookstore-devs`. Request nhẹ và ai cũng được phép:
   `kubectl auth can-i list pods -n bookstore`.
4. Nhóm của admin: `kubectl config view --raw --minify -o jsonpath='{.users[0].user.client-certificate-data}' | base64 -d | openssl x509 -noout -subject`.
5. Webhook dry-run: một `ValidatingWebhookConfiguration` tối thiểu cần `name`, `clientConfig`,
   `rules`, `sideEffects`, `admissionReviewVersions`. `kubectl create --dry-run=server -o yaml` in
   object sau khi server điền mặc định.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
. ~/lab-work/9b/ham.sh; cd ~/bookstore
AC=/etc/kubernetes/manifests/kube-apiserver.yaml
kubectl get --raw /metrics \
  | grep -o 'apiserver_admission_controller_admission_duration_seconds_count{name="[^"]*"' \
  | sed 's/.*name="//; s/"$//' | sort -u > "$EV/b10-admission-plugin.txt"
{ grep -qx 'PodSecurity' "$EV/b10-admission-plugin.txt" && grep -qx 'ResourceQuota' "$EV/b10-admission-plugin.txt" && \
  grep -qx 'LimitRanger' "$EV/b10-admission-plugin.txt" && ! grep -qx 'PodSecurityPolicy' "$EV/b10-admission-plugin.txt" && \
  [ "$(sudo grep -c -- '--admission-control-config-file' "$AC")" = '0' ]; } \
  && echo 'PASS: PodSecurity co mat, PodSecurityPolicy khong; bo mac dinh cua bai 282 dang hieu luc'

kubectl api-resources --api-group=policy -o name > "$EV/b10-policy.txt" 2>&1
{ ! grep -q 'podsecuritypolic' "$EV/b10-policy.txt" && grep -q '^poddisruptionbudgets' "$EV/b10-policy.txt" && \
  ! kubectl get --raw /apis/policy/v1beta1 >/dev/null 2>&1 && kubectl api-versions | grep -qx 'policy/v1'; } \
  && echo 'PASS: PSP da bi go khoi API; policy/v1 van phuc vu PodDisruptionBudget'
printf '%s\n' 'apiVersion: policy/v1beta1' 'kind: PodSecurityPolicy' 'metadata: {name: privileged}' \
  'spec: {privileged: true, seLinux: {rule: RunAsAny}, runAsUser: {rule: RunAsAny}, supplementalGroups: {rule: RunAsAny}, fsGroup: {rule: RunAsAny}}' \
  | kubectl apply --dry-run=server -f - > "$EV/b10-psp-apply.txt" 2>&1
grep -q 'no matches for kind' "$EV/b10-psp-apply.txt" && echo 'PASS: manifest PSP cua bai 286 khong apply duoc'

kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations \
  -o jsonpath='{range .items[*]}{.kind} {.metadata.name}{"\n"}{range .webhooks[*]}{"  "}{.name} failurePolicy={.failurePolicy} timeoutSeconds={.timeoutSeconds}{"\n"}{end}{end}' \
  > "$EV/b10-webhook.txt" 2>&1
WH="$(kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations --no-headers 2>/dev/null | wc -l)"
[ "$WH" -eq "$(cat "$EV/b0-webhook.txt")" ] && echo "PASS: ton kho webhook ghi xong, so webhook = $WH nhu B0"
printf '%s\n' 'apiVersion: admissionregistration.k8s.io/v1' 'kind: ValidatingWebhookConfiguration' \
  'metadata: {name: bookstore-thu-webhook}' 'webhooks:' '- name: thu.bookstore.example.com' \
  '  clientConfig: {service: {namespace: bookstore-thu, name: khong-ton-tai, port: 443}}' \
  '  rules: [{apiGroups: [""], apiVersions: [v1], operations: [CREATE], resources: [configmaps]}]' \
  '  namespaceSelector: {matchLabels: {kubernetes.io/metadata.name: bookstore-thu}}' \
  '  sideEffects: None' '  admissionReviewVersions: [v1]' \
  | kubectl create --dry-run=server -o jsonpath='{.webhooks[0].failurePolicy} {.webhooks[0].timeoutSeconds}' -f - \
  > "$EV/b10-webhook-mac-dinh.txt" 2>&1
{ [ "$(cat "$EV/b10-webhook-mac-dinh.txt")" = 'Fail 10' ] && \
  [ "$(kubectl get validatingwebhookconfigurations --no-headers 2>/dev/null | grep -c bookstore-thu-webhook)" -eq 0 ]; } \
  && echo 'PASS: mac dinh failurePolicy Fail, timeoutSeconds 10 — dry-run khong de lai webhook nao'

BB=0
for X in 'flowschema exempt' 'flowschema catch-all' 'prioritylevelconfiguration exempt' \
         'prioritylevelconfiguration catch-all'; do kubectl get $X >/dev/null 2>&1 && BB=$((BB + 1)); done
DX=0
for P in node-high system leader-election workload-high workload-low global-default; do
  kubectl get prioritylevelconfiguration "$P" >/dev/null 2>&1 && DX=$((DX + 1))
done
kubectl get flowschema --sort-by=.spec.matchingPrecedence \
  -o custom-columns='FS:.metadata.name,PRECEDENCE:.spec.matchingPrecedence,PLC:.spec.priorityLevelConfiguration.name,DIST:.spec.distinguisherMethod.type' \
  > "$EV/b10-flowschema.txt"
{ [ "$BB" -eq 4 ] && [ "$DX" -eq 6 ]; } && echo 'PASS: du bon doi tuong APF bat buoc va sau muc duoc de xuat'

pf()  { kubectl "$@" -v=8 2>&1 | grep -io 'x-kubernetes-pf-flowschema-uid: *[0-9a-f-]*' | head -1 | sed 's/.*: *//'; }
fsn() { kubectl get flowschema -o jsonpath='{range .items[*]}{.metadata.uid} {.metadata.name}{"\n"}{end}' \
          | awk -v u="$1" '$1==u {print $2}'; }
pl()  { kubectl get flowschema "$1" -o jsonpath='{.spec.priorityLevelConfiguration.name}' 2>/dev/null; }
FA="$(fsn "$(pf get ns default)")"
FC="$(fsn "$(pf auth can-i list pods -n "$NS" --as="$CI")")"
FD="$(fsn "$(pf auth can-i list pods -n "$NS" --as=dev-an --as-group=bookstore-devs)")"
O="$(kubectl config view --raw --minify -o jsonpath='{.users[0].user.client-certificate-data}' | base64 -d \
     | openssl x509 -noout -subject 2>/dev/null)"
echo "admin: $O -> $FA/$(pl "$FA") | ci: $FC/$(pl "$FC") | dev: $FD/$(pl "$FD")" | tee "$EV/b10-apf.txt"
{ [ "$FC" = 'service-accounts' ] && [ "$(pl "$FC")" = 'workload-low' ] && [ "$FD" = 'global-default' ]; } \
  && echo 'PASS: CI vao service-accounts/workload-low, doi phat trien vao global-default'
if echo "$O" | grep -q 'system:masters'; then
  [ "$FA" = 'exempt' ] && echo 'PASS: admin thuoc system:masters nen vao exempt'
else
  { [ -n "$FA" ] && [ "$FA" != 'exempt' ]; } && echo "PASS: admin khong thuoc system:masters nen vao $FA, khong phai exempt"
fi
ls docs/adr/*-tang-api-server.md >/dev/null 2>&1 && echo 'PASS: co ADR tang API server'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của ADR</summary>

1. Không đáng kể. Request của CI (ServiceAccount) rơi vào FlowSchema `service-accounts`, mức
   `workload-low`; kubelet nằm ở `system` và `node-high`, bầu leader ở `leader-election`. Mỗi mức có
   giới hạn concurrency riêng nên không làm đói nhau; trong `workload-low`, fair queuing tách flow theo
   người dùng nên CI còn không làm đói được controller khác cùng mức.
2. Kubeadm của baseline cấp `admin.conf` với nhóm `kubeadm:cluster-admins` — quyền quản trị qua một
   ClusterRoleBinding, **không** phải `system:masters`. Vì vậy request của bạn không đi FlowSchema
   `exempt` mà vào `global-default`. Đây chính là khuyến nghị của checklist 129: `system:masters` chỉ
   dành cho tình huống khẩn cấp (`super-admin.conf` trên master), không dùng hằng ngày. Nếu kết quả của
   bạn là `exempt`, kubeconfig đang dùng thuộc `system:masters` — ghi vào rà soát ở B11.
3. Một mutating webhook là thêm một thành phần nằm trên đường ghi của **mọi** Pod: mặc định `Fail` nên
   chết là chặn triển khai; dễ tự chặn chính Pod của nó; phải lũy đẳng; và quyền sửa cấu hình của nó
   là quyền sửa mọi object. Ba workload và manifest trong git không đáng cái giá đó — `restricted` +
   cảnh báo ở `deploy.sh` đã bắt lỗi trước khi lên cluster. Nếu buộc phải viết: `failurePolicy:
   Ignore` cho mutating kèm kiểm tra trạng thái cuối bằng validating, `timeoutSeconds` nhỏ,
   `namespaceSelector` loại `kube-system` và namespace của chính webhook, bật trên namespace thử nghiệm
   trước.
4. Bước 1 (rà quyền namespace) — B9; bước 3.a, 3.b (chọn mức, kiểm chứng bằng dry-run và
   `warn`/`audit`) — B2; bước 3.c (thực thi) — B6; bước 4 (quy trình tạo namespace mới) — namespace mới
   của dự án đi qua manifest có nhãn sẵn (B2, B8). Bước 2, 3.d, 5 không còn đối tượng — API đã bị gỡ.

</details>

## B11. [BS-9B-11] Rà soát bảo mật theo hai checklist

**Loại:** Điều tra.

**Bối cảnh:** bài 129 và 130 không phải bài đọc mà là **công cụ rà soát**. Sprint kết thúc bằng một
bản rà soát mà người tiếp quản đọc được: mỗi ô có câu trả lời lấy từ cluster, kèm quyết định — đạt,
chưa đạt và vì sao, hoặc không áp dụng và vì sao. Bản rà soát cũng ghi bốn đường vòng qua API server
của bài 128 và cơ chế xác thực đang bật của bài 123.

**Yêu cầu:**

1. Thu bằng chứng từ cluster cho **tám nhóm ô** của checklist 129 và **từng workload** của Bookstore
   theo checklist 130 (bảo mật cơ bản).
2. Bài 128: kiểm biện pháp giảm thiểu cho cả bốn đường vòng — thư mục static Pod manifest, cổng API
   kubelet, địa chỉ etcd đang nghe, socket container runtime — chỉ đọc, trên node.
3. Bài 123: cơ chế xác thực nào đang bật trên apiserver, cơ chế nào không.
4. Bài 254: cluster có cloud provider không; hệ quả cho hai ô về cloud metadata API và
   `LoadBalancer`/`externalIPs`.
5. Hoàn thành `docs/security-review.md`: phần *Secret và RBAC* (B9), phần *Checklist 129* tám nhóm,
   phần *Checklist 130* theo workload, phần *Đường vòng qua API server*. Mỗi dòng có kết quả, bằng chứng
   và quyết định; ô chưa đạt trỏ tới nợ (`TD-nn`, nợ lab `#N`) hoặc lý do chấp nhận.

**Ràng buộc:** chỉ đọc. Không mount `hostPath` để đọc file node — đọc trực tiếp bằng `sudo` trên node.
Không gọi API kubelet bằng credential.

**Đọc lại:**

- Bài [129](../129-security-checklist-vi.md) — ba điều kiện sử dụng ở đầu bài; mục
  [*Admission controller*](../129-security-checklist-vi.md#admission-controller-admission-controllers).
- Bài [130](../130-application-security-checklist-vi.md):
  [*securityContext cấp Pod*](../130-application-security-checklist-vi.md#security-context-pod),
  [*cấp container*](../130-application-security-checklist-vi.md#security-context-container),
  [*RBAC*](../130-application-security-checklist-vi.md#rbac).
- Bài [128](../128-api-server-bypass-risks-vi.md): [*Static Pod*](../128-api-server-bypass-risks-vi.md#static-pods-mitigations),
  [*API của kubelet*](../128-api-server-bypass-risks-vi.md#kubelet-api),
  [*API của etcd*](../128-api-server-bypass-risks-vi.md#etcd-api-mitigations),
  [*Socket của container runtime*](../128-api-server-bypass-risks-vi.md#runtime-socket-mitigations).
- Bài [123](../123-hardening-authentication-vi.md); bài
  [254](../254-running-cloud-controller-vi.md#vấn-đề-con-gà-và-quả-trứng-chicken-and-egg).

**Bàn giao:** `docs/security-review.md` hoàn chỉnh; `~/lab-evidence/9b/b11-*.txt`.

**Nghiệm thu:**

- `PASS:` bằng chứng thu đủ mười một mục.
- `PASS:` cổng chỉ-đọc 10255 của kubelet đóng trên ba node; cổng 10250 từ chối truy cập ẩn danh.
- `PASS:` etcd không nghe trên mọi địa chỉ; thư mục static Pod manifest và socket containerd thuộc root.
- `PASS:` không có cloud provider: không `cloud-controller-manager`, không `--cloud-provider`, không taint
  `uninitialized`.
- `PASS:` không container nào của dự án dùng tag `latest`.
- `PASS:` `docs/security-review.md` có đủ bốn phần, tám nhóm của checklist 129 và bốn đường vòng.

**Câu hỏi thiết kế** — trả lời ở cuối `docs/security-review.md`:

1. Kể ba ô **chưa đạt** quan trọng nhất và quyết định cho từng ô. Ô nào bạn chấp nhận vĩnh viễn trên
   cluster lab, vì sao?
2. Hai ô nào **không áp dụng** vì cluster không có cloud provider? Nếu dự án chuyển lên một cloud, ô
   nào phải làm trước?
3. Vì sao bốn đường vòng của bài 128 nguy hiểm hơn một request có đặc quyền cao đi qua API server? Bạn
   kiểm được biện pháp nào, không kiểm được biện pháp nào?
4. Theo bài 123, vì sao kubeconfig admin bạn đang dùng là credential rủi ro cao, và thu hồi nó tốn gì?

<details>
<summary>Gợi ý</summary>

- Bài 129 nói rõ: đánh dấu hết ô **không** có nghĩa là an toàn; ô nào quá chặt hay quá lỏng với dự án
  thì ghi quyết định, đừng bỏ trống.
- Cổng kubelet đọc bằng `curl` từ master tới IP node: `000` là không kết nối được, `401`/`403` là có
  cổng nhưng từ chối ẩn danh.
- etcd và socket: `sudo ss -ltn`, `sudo stat` trên master.
- Bảng checklist 130 theo workload nhanh nhất là đọc JSON của Deployment, StatefulSet, DaemonSet,
  CronJob bằng `python3`.

</details>

<details>
<summary>Script nghiệm thu — thu bằng chứng rồi kiểm; chạy sau khi đã viết xong security-review.md</summary>

```bash
. ~/lab-work/9b/ham.sh; cd ~/bookstore
AC=/etc/kubernetes/manifests/kube-apiserver.yaml
KCM=/etc/kubernetes/manifests/kube-controller-manager.yaml
ip_of() { kubectl get node "$1" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'; }
{
  echo '[A] Xac thuc va phan quyen'
  kubectl config view --raw --minify -o jsonpath='{.users[0].user.client-certificate-data}' | base64 -d \
    | openssl x509 -noout -subject -enddate
  kubectl get clusterrolebinding -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{range .subjects[*]}{.name}{" "}{end}{"\n"}{end}' \
    | grep -E 'system:masters|kubeadm:cluster-admins' || echo '  (khong co)'
  echo "  use-service-account-credentials: $(sudo grep -c -- '--use-service-account-credentials=true' "$KCM")"
  for F in client-ca-file token-auth-file oidc-issuer-url authentication-token-webhook-config-file \
           requestheader-client-ca-file service-account-issuer anonymous-auth; do
    echo "  --$F: $(sudo grep -c -- "--$F" "$AC")"
  done
  echo '[B] Mang'
  kubectl get pods -A -o name | grep -c calico-node | sed 's/^/  calico-node: /'
  kubectl get networkpolicy -A --no-headers | awk '{print "  "$1"/"$2}'
  echo "  Service LoadBalancer: $(kubectl get svc -A --no-headers | awk '$3=="LoadBalancer"' | wc -l)"
  echo "  Service co externalIPs: $(kubectl get svc -A -o jsonpath='{range .items[*]}{.spec.externalIPs}{"\n"}{end}' | grep -c .)"
  echo '[C] Bao mat Pod'
  kubectl get ns --selector='!pod-security.kubernetes.io/enforce' --no-headers | awk '{print "  khong enforce: "$1}'
  kubectl get ns -L pod-security.kubernetes.io/enforce --no-headers | awk '$4!="" {print "  enforce "$4": "$1}'
  for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
    echo "  apparmor $N: $(ssh "$N" 'cat /sys/module/apparmor/parameters/enabled' 2>/dev/null)"
  done
  echo '[D] Log va kiem toan'
  echo "  --audit-policy-file: $(sudo grep -c -- '--audit-policy-file' "$AC")"
  echo '[E] Sap xep vi tri Pod'
  echo "  db-0 tren: $(jp pod/db-0 '{.spec.nodeName}')  RuntimeClass: $(kubectl get runtimeclass --no-headers 2>/dev/null | wc -l)"
  echo '[F] Secret'
  echo "  --encryption-provider-config: $(sudo grep -c -- '--encryption-provider-config' "$AC")"
  echo "  secret service-account-token o bookstore: $(kubectl -n "$NS" get secret --field-selector type=kubernetes.io/service-account-token --no-headers 2>/dev/null | wc -l)"
  echo '[G] Image'
  kubectl get pods -A -l app.kubernetes.io/part-of=bookstore -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
    | sort -u | sed 's/^/  /'
  echo '[H] Admission'; sed 's/^/  /' "$EV/b10-admission-plugin.txt"
  echo '[I] Duong vong qua API server (bai 128)'
  sudo stat -c '  static manifests: %U:%G %a %n' /etc/kubernetes/manifests
  for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
    IP="$(ip_of "$N")"
    RO="$(curl -s -o /dev/null -m 3 -w '%{http_code}' "http://$IP:10255/pods" || true)"
    AU="$(curl -sk -o /dev/null -m 5 -w '%{http_code}' "https://$IP:10250/pods" || true)"
    echo "  kubelet $N: readonly_10255=${RO:-000} anonymous_10250=${AU:-000}"
  done
  sudo ss -ltn | awk '$4 ~ /:2379$/ {print "  etcd nghe: "$4}'
  sudo stat -c '  socket: %U:%G %a %n' /run/containerd/containerd.sock
  echo '[J] Cloud provider (bai 254)'
  echo "  cloud-controller-manager: $(kubectl get pods -A -o name | grep -c cloud-controller)"
  echo "  --cloud-provider: $(sudo grep -c -- '--cloud-provider' "$KCM" "$AC" | awk -F: '{s+=$2} END{print s+0}')"
  echo "  taint uninitialized: $(kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' | grep -c cloudprovider)"
  echo '[K] Checklist 130 theo workload'
  kubectl get deployments,statefulsets,daemonsets,cronjobs -A -l app.kubernetes.io/part-of=bookstore -o json | python3 -c '
import json, sys
for o in json.load(sys.stdin)["items"]:
    s = o["spec"]
    t = s["jobTemplate"]["spec"]["template"]["spec"] if o["kind"] == "CronJob" else s["template"]["spec"]
    p = t.get("securityContext", {})
    for c in t.get("initContainers", []) + t["containers"]:
        x = c.get("securityContext", {})
        print("  %-18s %-26s %-14s sa=%-12s nonroot=%-5s ape=%-5s dropALL=%-5s rofs=%-5s mem=%s" % (
            o["metadata"]["namespace"], o["kind"] + "/" + o["metadata"]["name"], c["name"],
            t.get("serviceAccountName", "default"), x.get("runAsNonRoot", p.get("runAsNonRoot")),
            x.get("allowPrivilegeEscalation"), "ALL" in x.get("capabilities", {}).get("drop", []),
            x.get("readOnlyRootFilesystem", False), c.get("resources", {}).get("limits", {}).get("memory", "-")))'
} > "$EV/b11-checklist.txt" 2>&1
cat "$EV/b11-checklist.txt"

[ "$(grep -c '^\[[A-K]\]' "$EV/b11-checklist.txt")" -eq 11 ] && echo 'PASS: bang chung du muoi mot muc'
{ [ "$(grep -c 'readonly_10255=000' "$EV/b11-checklist.txt")" -eq 3 ] && \
  [ "$(grep -cE 'anonymous_10250=(401|403)' "$EV/b11-checklist.txt")" -eq 3 ]; } \
  && echo 'PASS: kubelet: cong chi-doc dong, API tu choi truy cap an danh tren ba node'
{ ! grep -Eq 'etcd nghe: (0\.0\.0\.0|\*|\[::\]):2379' "$EV/b11-checklist.txt" && grep -q 'etcd nghe:' "$EV/b11-checklist.txt" && \
  grep -q 'static manifests: root:root 7[05][05]' "$EV/b11-checklist.txt" && \
  grep -q 'socket: root:root' "$EV/b11-checklist.txt"; } \
  && echo 'PASS: etcd khong nghe moi dia chi; static manifests va socket containerd thuoc root'
{ grep -q 'cloud-controller-manager: 0' "$EV/b11-checklist.txt" && grep -q -- '--cloud-provider: 0' "$EV/b11-checklist.txt" && \
  grep -q 'taint uninitialized: 0' "$EV/b11-checklist.txt"; } && echo 'PASS: cluster khong co cloud provider'
! sed -n '/^\[G\]/,/^\[H\]/p' "$EV/b11-checklist.txt" | grep -q ':latest$' && echo 'PASS: khong image nao cua du an dung tag latest'

R=docs/security-review.md; BAD=0
for H in 'Secret và RBAC' 'Checklist 129' 'Checklist 130' 'Đường vòng'; do grep -q "^## .*$H" "$R" || { echo "FAIL: thieu phan $H"; BAD=1; }; done
for K in 'Xác thực' 'mạng' 'Bảo mật Pod' 'Log' 'Sắp xếp' 'Secret' 'Image' 'Admission'; do grep -q "$K" "$R" || { echo "FAIL: thieu nhom $K"; BAD=1; }; done
for K in 'static Pod' 'kubelet' 'etcd' 'socket'; do grep -qi "$K" "$R" || { echo "FAIL: thieu duong vong $K"; BAD=1; }; done
[ "$BAD" -eq 0 ] && echo 'PASS: security-review.md du bon phan, tam nhom 129, bon duong vong'
```

</details>

<details>
<summary>Lời giải tham khảo — khung các phần còn lại của security-review.md</summary>

```markdown
## 2. Checklist 129 — cluster

Theo cảnh báo của bài 129: checklist không đủ cho một thế trận an toàn; mỗi nhóm đánh giá theo giá trị
riêng. Bằng chứng: ~/lab-evidence/9b/b11-checklist.txt (mục [A]–[J]).

| Nhóm | Ô | Kết quả | Quyết định |
| --- | --- | --- | --- |
| Xác thực & phân quyền | system:masters không dùng hằng ngày | admin.conf thuộc kubeadm:cluster-admins | đạt |
| | --use-service-account-credentials | bật | đạt |
| | thời hạn certificate, quy trình rà quyền định kỳ | chưa có quy trình | nợ lab #7, giai đoạn 18 |
| Bảo mật mạng | CNI thực thi NetworkPolicy | Calico | đạt |
| | default deny mọi namespace | chỉ bookstore, bookstore-staging | chấp nhận: namespace hệ thống do add-on quản |
| | cloud metadata API, LoadBalancer/externalIPs | không có cloud provider | không áp dụng (bài 254) |
| Bảo mật Pod | PSS cho mọi namespace | 2 namespace restricted; còn lại không nhãn | namespace hệ thống giữ mặc định — ghi rõ |
| | giới hạn bộ nhớ | mọi container của dự án có limits | đạt |
| | seccomp, AppArmor | RuntimeDefault cho mọi Pod dự án; AppArmor bật 3 node | đạt |
| Log và kiểm toán | audit log | chưa bật | giai đoạn 22 |
| Sắp xếp vị trí Pod | ứng dụng nhạy cảm tách node | db ghim worker2; không RuntimeClass | chấp nhận ở quy mô lab |
| Secret | mã hóa at rest | chưa bật | nợ lab #6 |
| | token không mount khi không cần | web, api, db không mount | đạt |
| Image | không tag latest; digest; quét, ký | tag cố định; chưa dùng digest, chưa quét | chưa đạt — ngoài cluster |
| Admission | PodSecurity bật; webhook an toàn | có PodSecurity; webhook: xem b10-webhook.txt | đạt |

## 3. Checklist 130 — từng workload

| Workload | SA riêng, không token | non-root | không leo thang | drop ALL | rootfs chỉ đọc | limits |
| --- | --- | --- | --- | --- | --- | --- |
| Deployment web | có | 101 | có | có | nginx: có | có |
| Deployment api | có | 1000 | có | có | postgrest: có | có |
| StatefulSet db | có | 999 | có | có | **không** — PostgreSQL ghi ngoài PGDATA; quyết định giữ | có |
| ... | | | | | | |

## 4. Đường vòng qua API server (bài 128) và xác thực (bài 123)

| Đường | Biện pháp đã kiểm | Kết quả |
| --- | --- | --- |
| Static Pod | /etc/kubernetes/manifests thuộc root, không ai khác ghi được | đạt |
| API kubelet | 10255 đóng; 10250 từ chối ẩn danh | đạt |
| API etcd | chỉ nghe 127.0.0.1 và IP node | đạt; mTLS và CA riêng: giai đoạn 19 |
| Socket runtime | thuộc root; không Pod nào mount hostPath tới nó | đạt |
| Xác thực | client certificate X.509 bật; không token file tĩnh, không OIDC | chấp nhận ở lab; production cần OIDC |
```

Ý chính cho câu hỏi: (1) ba ô chưa đạt quan trọng: mã hóa at rest (#6), audit log (giai đoạn 22),
image không quét/không digest — hai ô đầu cần sửa cờ apiserver nên chờ giai đoạn sau, ô thứ ba cần
pipeline ngoài cluster; (2) cloud metadata API và `LoadBalancer`/`externalIPs` — lên cloud thì lọc
`169.254.169.254` bằng NetworkPolicy egress trước tiên; (3) chúng không qua admission và không vào
audit log, kẻ đi đường đó không để lại dấu vết; kiểm được quyền thư mục, cổng, địa chỉ nghe; không kiểm
được kiểm soát private key của etcd và việc kiểm toán truy cập thư mục manifest; (4) certificate không
thu hồi riêng được, nhóm nằm trong `O`, không có bản ghi các certificate đã cấp — thu hồi nghĩa là thay
khóa CA, ảnh hưởng mọi thành phần.

</details>

## B12. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 16 — Bookstore dưới
`restricted`, tenant `bookstore-staging` — rồi trả cluster về đúng `03-storage-ready`, chứng minh lab
không để lại Pod đặc quyền, webhook hay thay đổi cấu hình nào.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 9b — kiểm thử khói **tích lũy**, chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 9b. Mục 1–10 giữ nguyên bản của Lab 9a (dữ liệu bền, mạng
# zero-trust, HTTPS, lập lịch, hạn mức, danh tính và quyền); mục 11–14 là sprint 16: Pod Security
# restricted, db non-root, rootfs chỉ đọc, tenant bookstore-staging. Không in mật khẩu.
set -u
cd "$(dirname "$0")/.."
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
can() { kubectl auth can-i "$@" -n "$NS" 2>/dev/null; }
TNS=bookstore-staging
psa() { kubectl get namespace "$1" -o jsonpath="{.metadata.labels.pod-security\.kubernetes\.io/$2}" 2>/dev/null; }
pst() { kubectl -n "$1" exec "$2" -c "$3" -- sh -c "awk '/^$4:/{print \$2}' /proc/1/status" 2>/dev/null; }
dry_priv() {
  printf '%s\n' 'apiVersion: v1' 'kind: Pod' 'metadata: {name: smoke-dac-quyen}' 'spec:' \
    '  containers: [{name: c, image: busybox:1.37, securityContext: {privileged: true}}]' \
    | kubectl -n "$1" create --dry-run=server -f - 2>&1
}
V="$(kubectl get --raw /version 2>/dev/null)"
PSA_VER="v$(echo "$V" | sed -n 's/.*"major": *"\([0-9]*\).*/\1/p').$(echo "$V" | sed -n 's/.*"minor": *"\([0-9]*\).*/\1/p')"

# 1. Namespace và object cấp cluster
[ "$(kubectl get namespace "$NS" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = bookstore ] \
  && pass 'namespace bookstore mang label du an' || fail 'namespace bookstore thieu hoac thieu label'
for PC in bookstore-critical bookstore-batch; do
  [ "$(kubectl get priorityclass "$PC" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = bookstore ] \
    && pass "co PriorityClass $PC" || fail "thieu PriorityClass $PC"
done
[ "$(kubectl get storageclass bookstore-retain -o jsonpath='{.reclaimPolicy}' 2>/dev/null)" = Retain ] \
  && pass 'StorageClass bookstore-retain Retain' || fail 'thieu StorageClass bookstore-retain Retain'
[ "$(kubectl get nodes -l bookstore.example.com/data=true -o jsonpath='{.items[*].metadata.name}')" = lab-k8s-worker2 ] \
  && pass 'chi lab-k8s-worker2 mang label du lieu' || fail 'label node bookstore.example.com/data sai'
N_CL="$(kubectl get clusterrole,clusterrolebinding -l "$SEL" --no-headers 2>/dev/null | wc -l)"
[ "$N_CL" -eq 0 ] && pass 'du an khong co ClusterRole, ClusterRoleBinding' || fail "co $N_CL object RBAC cap cluster"

# 2. Cấu hình, bí mật, PDB, hạn mức
for P in web-config api-config db-init; do
  k get configmap -o name 2>/dev/null | grep -q "^configmap/$P" && pass "co ConfigMap $P" || fail "thieu ConfigMap $P"
done
k get configmap backup-status >/dev/null 2>&1 && pass 'co ConfigMap backup-status' || fail 'thieu ConfigMap backup-status'
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials du hai key' || fail 'Secret db-credentials thieu key'
[ "$(jp secret/bookstore-tls '{.type}')" = kubernetes.io/tls ] && pass 'co Secret TLS bookstore-tls' || fail 'thieu Secret bookstore-tls'
for P in web api; do k get pdb "$P" >/dev/null 2>&1 && pass "co PDB $P" || fail "thieu PDB $P"; done
k get resourcequota bookstore-quota >/dev/null 2>&1 && pass 'co ResourceQuota bookstore-quota' || fail 'thieu bookstore-quota'
k get limitrange bookstore-limits >/dev/null 2>&1 && pass 'co LimitRange bookstore-limits' || fail 'thieu bookstore-limits'

# 3. Controller, vị trí, dữ liệu bền
[ "$(jp statefulset/db '{.status.readyReplicas}')" = 1 ] && pass 'StatefulSet db 1/1 Ready' || fail 'StatefulSet db chua Ready'
[ "$(jp pod/db-0 '{.spec.nodeName}')" = lab-k8s-worker2 ] && pass 'db-0 tren lab-k8s-worker2' || fail 'db-0 khong o lab-k8s-worker2'
[ "$(jp pod/db-0 '{.status.qosClass}')" = Guaranteed ] && pass 'db-0 QoS Guaranteed' || fail 'db-0 khong Guaranteed'
PRIO="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
[ -n "$PRIO" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$PRIO" ] && pass 'db-0 dung bookstore-critical' || fail 'db-0 sai priority'
# backup-store (WaitForFirstConsumer) chỉ Bound sau lần chạy thử db-backup bên dưới (6a).
[ "$(jp pvc/data-db-0 '{.status.phase}')" = Bound ] && pass 'PVC data-db-0 Bound' || fail 'PVC data-db-0 chua Bound'
[ "$(jp pvc/data-db-0 '{.spec.storageClassName}')" = bookstore-retain ] && pass 'data-db-0 dung bookstore-retain' || fail 'data-db-0 sai StorageClass'
R="$(jp deployment/web '{.status.readyReplicas}')"; S="$(jp deployment/web '{.spec.replicas}')"
[ "${R:-0}" -ge 2 ] && [ "$R" = "$S" ] && pass "Deployment web $R/$S Ready" || fail "Deployment web ${R:-0}/${S:-?} Ready"
[ "$(jp deployment/api '{.status.readyReplicas}')" = 2 ] && pass 'Deployment api 2/2 Ready' || fail 'Deployment api chua du 2 ban sao'
NODES="$(kubectl get nodes --no-headers | wc -l)"
[ "$(jp daemonset/node-agent '{.status.numberReady}')" = "$NODES" ] && pass "node-agent Ready tren $NODES node" || fail 'node-agent thieu node'
ORPHAN="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"
MISS="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container co requests va limits' || fail "container thieu tai nguyen: $(echo $MISS)"

# 4. Service và đường đi của request trong cluster
[ "$(jp service/web '{.spec.type}/{.spec.ports[0].port}')" = ClusterIP/80 ] && pass 'Service web ClusterIP:80' || fail 'Service web sai'
[ "$(jp service/api '{.spec.ports[0].port}')" = 3000 ] && pass 'Service api :3000' || fail 'Service api sai'
[ "$(jp service/db '{.spec.clusterIP}')" = None ] && pass 'Service db headless' || fail 'Service db khong headless'
[ "$(jp service/node-agent '{.spec.internalTrafficPolicy}')" = Local ] && pass 'Service node-agent Local' || fail 'Service node-agent sai'
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach"
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
get() { k exec "$WEB" -c status-writer -- wget -qO- -T 5 "$1" 2>/dev/null; }
T="$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "web /api/books tra $T cuon sach" || fail "web /api/books tra $T cuon sach"
get http://127.0.0.1:8080/ | grep -q Bookstore && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
get http://127.0.0.1:8080/status.json | grep -Eq "\"pod\" *: *\"$WEB\"" && pass 'status.json dung ten Pod' || fail 'status.json sai'
WN="$(jp "pod/$WEB" '{.spec.nodeName}')"
get http://node-agent:9100/node.json | grep -Eq "\"node\" *: *\"$WN\"" && pass 'node-agent cung node tra /node.json' || fail 'node-agent qua Service sai'

# 5. HTTPS từ ngoài qua Ingress
[ "$(jp ingress/bookstore '{.spec.rules[0].host}')" = bookstore.lab.local ] && pass 'Ingress bookstore dung host' || fail 'Ingress bookstore sai host'
[ "$(jp ingress/bookstore '{.spec.tls[0].secretName}')" = bookstore-tls ] && pass 'Ingress dung TLS bookstore-tls' || fail 'Ingress thieu TLS'
HP="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}' 2>/dev/null)"
IP="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
curl -sk --max-time 10 --resolve "bookstore.lab.local:${HP}:${IP}" "https://bookstore.lab.local:${HP}/" | grep -q Bookstore \
  && pass "HTTPS qua Traefik ($IP:$HP)" || fail 'HTTPS qua Traefik khong tra trang chu'

# 6. NetworkPolicy
NPS="$(k get networkpolicy -o jsonpath='{range .items[*]}{.spec.podSelector}{"|"}{.spec.policyTypes}{"\n"}{end}' 2>/dev/null)"
{ echo "$NPS" | awk -F'|' '$1=="{}" && $2 ~ /Ingress/' | grep -q . && \
  echo "$NPS" | awk -F'|' '$1=="{}" && $2 ~ /Egress/' | grep -q .; } \
  && pass 'NetworkPolicy co lap ca hai chieu cho moi Pod' || fail 'thieu NetworkPolicy mac dinh chan hai chieu'
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

# 7. CronJob db-backup và backup-status
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
[ "$(jp pvc/backup-store '{.status.phase}')" = Bound ] && pass 'PVC backup-store Bound (sau lan chay thu)' || fail 'PVC backup-store chua Bound'

# 8. Danh tính và quyền
BAD=0
for X in deployment/web:web deployment/api:api statefulset/db:db daemonset/node-agent:node-agent; do
  [ "$(jp "${X%%:*}" '{.spec.template.spec.serviceAccountName}')" = "${X##*:}" ] || { echo "FAIL: ${X%%:*} sai ServiceAccount"; BAD=1; }
done
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.serviceAccountName}')" = db-backup ] \
  || { echo 'FAIL: db-backup sai ServiceAccount'; BAD=1; }
for SA in default web api db db-migrate db-backup node-agent ci-deployer; do
  [ "$(jp "serviceaccount/$SA" '{.automountServiceAccountToken}')" = false ] \
    || { echo "FAIL: ServiceAccount $SA thieu hoac tu mount token"; BAD=1; }
done
[ "$BAD" -eq 0 ] && pass 'moi thanh phan mot ServiceAccount, khong ai tu mount token' || FAIL=1
TOKP="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.volumes[*]}{.name}{"\n"}{end}{end}' 2>/dev/null | grep -c '^kube-api-access-')"
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

# 9. Danh tính tiến trình và tham số kernel
SC='{.spec.jobTemplate.spec.template.spec.securityContext'
{ [ "$(jp cronjob/db-backup "$SC.runAsUser}")" = 999 ] && [ "$(jp cronjob/db-backup "$SC.runAsNonRoot}")" = true ]; } \
  && pass 'db-backup chay uid 999, kubelet kiem non-root' || fail 'db-backup chua non-root'
jp deployment/api '{range .spec.template.spec.securityContext.sysctls[*]}{.name}={.value}{" "}{end}' \
  | grep -q 'net.ipv4.tcp_keepalive_time=60' && pass 'api dat sysctl keepalive' || fail 'api thieu sysctl keepalive'

# 10. Repo (đầu script đã cd về gốc repo)
git check-ignore -q secrets/ci-deployer.kubeconfig \
  && pass 'kubeconfig CI nam ngoai git' || fail 'secrets/ khong bi .gitignore chan'

# 11. Pod Security (9b)
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

# 12. Hardening (9b): db non-root, không capability, rootfs chỉ đọc
[ "$(pst "$NS" db-0 postgres Uid)" = 999 ] && pass 'db chay uid 999' || fail 'db khong chay uid 999'
C="$(pst "$NS" db-0 postgres CapEff)"
[ "$((0x${C:-1}))" -eq 0 ] && pass 'db khong giu capability nao' || fail 'db con capability'
NODROP="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.securityContext.capabilities.drop}{"\n"}{end}{end}' 2>/dev/null \
          | grep -v 'ALL' | sort -u)"
[ -z "$NODROP" ] && pass 'moi container cua du an drop ALL capability' || fail "container chua drop ALL: $(echo $NODROP)"
RO() { jp "deployment/$1" "{.spec.template.spec.containers[?(@.name==\"$2\")].securityContext.readOnlyRootFilesystem}"; }
{ [ "$(RO web nginx)" = true ] && [ "$(RO api postgrest)" = true ]; } \
  && pass 'nginx, postgrest khai rootfs chi doc' || fail 'nginx hoac postgrest chua rootfs chi doc'
k exec "$WEB" -c nginx -- sh -c 'touch /smoke-ghi' >/dev/null 2>&1 \
  && fail 'ghi duoc vao rootfs cua nginx' || pass 'rootfs cua nginx chi doc that'

# 13. Tenant bookstore-staging (9b)
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

# 14. Bí mật không vào git (3b)
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

2. Cập nhật `docs/tech-debt.md`: `TD-06` → `đã trả (9b)`, bằng chứng `~/lab-evidence/9b/b3-kernel.txt`;
   trạng thái sau 9b của `TD-07`, `TD-15` (B9). Ghi chú cạnh nợ lab #6: vẫn mở, xem
   `docs/security-review.md`.

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/9b/b12-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/9b/b12-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/9b/b12-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-9b: sprint 16 — Pod Security restricted, db non-root, tenant staging, ra soat bao mat'
git tag lab-9b
git bundle create ~/bookstore-lab-9b.bundle --all && git bundle verify ~/bookstore-lab-9b.bundle
```

Chép `~/bookstore-lab-9b.bundle` về máy host bằng `scp` trước khi đi tiếp. Bundle không chứa
`secrets/` — kể cả `secrets/ci-deployer.kubeconfig` — đúng chủ đích.

5. Dọn object thử nghiệm và trả cluster:

```bash
kubectl delete namespace -l bookstore.example.com/thu=true --ignore-not-found --wait=true --timeout=180s
rm -rf ~/lab-work/9b
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
EV=~/lab-evidence/9b; SEL='app.kubernetes.io/part-of=bookstore'
cd ~/bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-9b' && echo 'PASS: HEAD mang tag lab-9b'
test -s ~/bookstore-lab-9b.bundle && echo 'PASS: co bundle'
{ test -f k8s/tenants/staging/kustomization.yaml && test -f docs/security-review.md && \
  test -f docs/pss-kiem-ke.md && grep -q 'profile=restricted' docs/runbook-debug-api.md; } \
  && echo 'PASS: bo cuc repo dung hang 9b cua muc 5.5'
grep -E 'TD-06.*đã trả' docs/tech-debt.md >/dev/null && grep 'TD-15' docs/tech-debt.md | grep -q '9b' \
  && echo 'PASS: tech-debt.md ghi TD-06 da tra, TD-15 co trang thai sau 9b'
BAD=0
for F in pod-security-standard pss-lo-trinh-siet db-non-root rootfs-chi-doc tac-vu-nen-restricted \
         enforce-restricted tenant-staging tang-api-server; do
  ls docs/adr/*-"$F".md >/dev/null 2>&1 || { echo "FAIL: thieu docs/adr/*-$F.md"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR cua sprint 16'
{ [ "$(sl kubectl get namespace,priorityclass,storageclass,clusterrole,clusterrolebinding -l "$SEL" --no-headers)" -eq 0 ] && \
  [ "$(sl kubectl get all -A -l "$SEL" --no-headers)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an'
test -z "$(kubectl get namespace -o name | grep 'namespace/bookstore')" && echo 'PASS: khong con namespace bookstore*'
test ! -e ~/lab-work/9b && echo 'PASS: da don lab-work'
P9="$(kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.securityContext.privileged}{"\n"}{end}{end}' \
      | grep -c '^true$' || true)"
[ "$P9" -eq "$(cat "$EV/b0-priv.txt")" ] && echo "PASS: so container dac quyen bang luc dau ($P9)"
[ "$(kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations --no-headers 2>/dev/null | wc -l)" \
  -eq "$(cat "$EV/b0-webhook.txt")" ] && echo 'PASS: so webhook bang luc dau'
{
  echo "apiserver $(sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml | awk '{print $1}')"
  echo "kubelet-lab-k8s-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for N in lab-k8s-worker1 lab-k8s-worker2; do
    echo "kubelet-$N $(ssh "$N" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
} > "$EV/b12-cauhinh-sha.txt"
diff -q "$EV/b0-cauhinh-sha.txt" "$EV/b12-cauhinh-sha.txt" >/dev/null \
  && echo 'PASS: cau hinh apiserver va kubelet ba node khong doi trong suot lab'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
kubectl get storageclass | grep -q '(default)' && echo 'PASS: van con StorageClass mac dinh cua 03-storage-ready'
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; mười hai dòng `PASS:` đầu của gate
cuối xuất hiện; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica; namespace `default` không có Pod; dòng `PASS:` của StorageClass mặc định. Thư mục
`~/lab-evidence/9b/` **giữ lại** — nó là hồ sơ rà soát bảo mật của cluster và không chứa mật khẩu.
Cluster trở về `03-storage-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 9b

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Ba profile theo thứ tự tích lũy. Kể bốn thứ Baseline cấm và bốn thứ Restricted **thêm** vào —
      với mỗi thứ, workload nào của Bookstore từng vi phạm và bạn sửa ở ticket nào.
- [ ] Ba chế độ làm gì khi có vi phạm? Phân biệt `violates` với `would violate`. Phần nào của `audit`
      cluster này không đọc được, vì sao, và nó ở giai đoạn nào?
- [ ] Một Deployment vi phạm được apply vào namespace chỉ có `enforce`: lệnh apply báo gì, có Pod nào
      không, lý do nằm ở object nào? Vì sao `deploy.sh` của Lab 9b coi cảnh báo `would violate` là lỗi?
- [ ] Kể lộ trình siết `bookstore` từ B2 tới B6: nhãn ở từng bước, công cụ kiểm kê, và chỗ dry-run bỏ
      sót. Đổi nhãn `enforce` có làm chết Pod đang vi phạm không?
- [ ] `db` chạy uid 999: vì sao `PGDATA` phải là thư mục con, vì sao cần `fsGroup`, và bằng chứng nào
      cho thấy entrypoint không cần `chown`? Vì sao `fsGroup` không làm hỏng quyền `0700` của thư mục
      dữ liệu trên PV `local-path`?
- [ ] Trường nào chỉ có ở cấp container, trường nào ở cấp Pod? Vì sao Restricted không đòi
      `readOnlyRootFilesystem`, và `nginx` cần gì để chạy với nó?
- [ ] Bạn đọc trạng thái kernel thật của một container bằng những giá trị nào trong `/proc`? Container
      `privileged` khai `RuntimeDefault` cho ra ba giá trị gì? (⭐ B7)
- [ ] Vì sao `kubectl debug` theo cách của Lab 3a bị từ chối sau B6? Profile `restricted` đặt gì, và vì
      sao uid đặt ở cấp Pod của `api` là điều kiện để container gỡ lỗi khởi động?
- [ ] Bốn lớp của `bookstore-staging`: object nào, chặn gì, kiểm bằng phép thử nào. Ai `patch` được
      Namespace thì sụp lớp nào trước? Mô hình này **không** bảo vệ điều gì?
- [ ] Ai đọc được `db-credentials`: trực tiếp qua RBAC, gián tiếp qua workload (`TD-15`) và `exec`
      (`TD-07`), trên node (`tmpfs`, swap), trong etcd (nợ #6)?
- [ ] Request của admin, CI và đội phát triển rơi vào FlowSchema và mức ưu tiên nào? Vì sao admin của
      bạn không đi `exempt`, và vì sao điều đó đúng với khuyến nghị của checklist 129?
- [ ] `failurePolicy` mặc định của webhook là gì, đọc ra bằng cách nào mà không cài webhook? Vì sao dự
      án không viết mutating webhook để điền `securityContext`? Kể bốn đường vòng qua API server và biện
      pháp bạn đã kiểm cho từng đường.

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Nợ lab **#6** — mã hóa Secret at rest: cần sửa cấu hình `kube-apiserver`, trả ở
  [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu). B9 và B11 ghi ô này
  **không đạt**. Nợ lab #7 (vòng đời certificate) liên quan một ô của checklist 129, trả ở giai đoạn 18.
- Nợ kỹ thuật: **`TD-06` đã trả** ở sprint này. `TD-15` (Lab 9a) **vẫn mở**: `restricted` chặn CI chạy
  Pod đặc quyền, nhưng CI vẫn gắn được Secret và mượn ServiceAccount qua pod template — xem lại ở
  giai đoạn 14. `TD-07` giữ trạng thái đã giới hạn của Lab 9a. Sprint này **không phát sinh** nợ kỹ
  thuật hay nợ lab mới; các nợ khác giữ trạng thái của `docs/tech-debt.md` sau Lab 9a.

### Bài giải thích cuối cùng

Trong vài phút, kể lại đường đi của `db-0` trong sprint này. Bắt đầu từ `./scripts/deploy.sh`: Kustomize
render namespace `bookstore` với sáu nhãn Pod Security — API server tạo namespace trước. StatefulSet
được gửi lên: `warn` và `audit` đánh giá Pod template ngay lúc đó, `enforce` thì chưa; vì sao? Controller
tạo Pod `db-0`: Pod Security Admission tra nhãn nào trên namespace, áp mức nào, phiên bản nào, và cái gì
xảy ra nếu template thiếu một trường. Pod qua admission, xuống kubelet: kubelet kiểm `runAsNonRoot` với
uid nào, gán `fsGroup` cho volume nào và **không** cho volume nào; entrypoint chạy uid 999, đọc file mật
khẩu nhờ đâu, `initdb` tạo thư mục dữ liệu ở đâu và vì sao không cần `chown`. Kể tiếp hai lớp mà Pod này
không tự bảo vệ được: ai có thể đọc mật khẩu của nó mà không cần quyền `get` trên Secret, và mật khẩu đó
nằm thế nào trong etcd. Kết thúc bằng việc chỉ ra Lab 11a sẽ thêm namespace `bookstore-system` mức
`privileged` ở đâu trong câu chuyện này, và vì sao điều đó không làm yếu `bookstore`.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `git describe` không ra `lab-9a` | Repo đang ở commit khác, hoặc clone từ bundle cũ | `git checkout lab-9a` hoặc clone lại từ `~/bookstore-lab-9a.bundle` mới nhất |
| B1: nhãn `*-version` rỗng hoặc là chữ `$PSA_VER` | Heredoc bị đóng nháy (`<<'EOF'`), hoặc chưa `. ~/lab-work/9b/ham.sh` | Dùng `<<EOF` không nháy cho file namespace; source lại `ham.sh` |
| B1: số câu từ chối Baseline ít hơn 5 | Một Pod trong file vi phạm hai kiểm soát, hoặc thiếu `nodeSelector` không liên quan | Mỗi Pod đúng một vi phạm; đọc từng câu trong `b1-vi-pham-baseline.txt` |
| B1: `ReplicaFailure` mãi không xuất hiện | ReplicaSet chưa kịp thử tạo Pod | Chờ thêm; thời gian controller thử lại **phụ thuộc cấu hình** |
| B1: bộ đếm `audit` không tăng | Pod `thu-tran` đã tồn tại, `apply` báo `unchanged` nên không có request | Xóa Pod rồi chạy lại khối đó; script đã làm việc này |
| B2: deploy không in cảnh báo `would violate` nào | Apply lên object không đổi — không có request nào đi qua admission | `./scripts/teardown.sh` trước rồi mới `deploy.sh` |
| B2, B3, B5, B6: namespace `bookstore-thu*` biến mất | `teardown.sh` xóa mọi namespace mang label dự án | Ticket cần thì `kubectl apply -f ~/lab-work/9b/b1-namespaces.yaml` lại |
| B3: `db-0` `CrashLoopBackOff`, log `Permission denied` ở `/etc/db-credentials/...` | File trong volume Secret thuộc root `0400`, Pod thiếu `fsGroup` | Thêm `fsGroup: 999` ở cấp Pod |
| B3: `initdb: could not change permissions of directory` | `PGDATA` trỏ vào gốc volume do provisioner tạo, thuộc root | Đặt `PGDATA` là thư mục con của mount |
| B3: `CreateContainerConfigError`, `image will run as root` | `runAsNonRoot: true` mà thiếu `runAsUser`, image không khai `USER` | Thêm `runAsUser: 999`, `runAsGroup: 999` |
| B3: `data directory ... has invalid permissions` | PV loại `local` (không phải `hostPath`) nên kubelet áp `fsGroup` đệ quy, thêm quyền group lên `PGDATA` | `kubectl get pv -o yaml` xem loại; thêm `fsGroupChangePolicy: OnRootMismatch`, hoặc giữ StorageClass ở loại `hostPath` mặc định của local-path |
| B3: `api` báo lỗi xác thực sau teardown | `secrets/db.env` bị sinh lại khi dữ liệu cũ còn trên PV | Như Lab 6a: `teardown.sh` phải xóa cả PV và thư mục dữ liệu |
| B4: `nginx` `CrashLoopBackOff`, log `open() "/tmp/nginx.pid" failed (30: Read-only file system)` | Bật rootfs chỉ đọc mà chưa mount `emptyDir` ở `/tmp` | Thêm volume `tmp` mount vào `/tmp` của `nginx` |
| B4: Pod `api` treo `Init:0/1` mãi | Init chờ DB chạy uid 1000 kế thừa từ cấp Pod; image `postgres` không có uid đó trong `/etc/passwd` nên `pg_isready` không thử kết nối | Đặt `runAsUser: 999` ở cấp container cho init đó |
| B4: rollout `web` hoặc `api` kẹt `exceeded quota` | `maxSurge` cần chỗ vượt `bookstore-quota` (Lab 7b) | Xử lý như Lab 7b; không nới quota chỉ để qua ticket |
| B5: Job từ CronJob không bao giờ Complete, không có Pod | Còn container trong `jobTemplate` thiếu trường Restricted; sự kiện `FailedCreate` của Job ghi `violates PodSecurity` | `kubectl describe job <tên>`; sửa đúng container được nêu |
| B5: `node-agent` `CrashLoopBackOff` sau khi bật rootfs chỉ đọc (⭐) | Agent ghi file ngoài `/www` | Mọi file tạm phải nằm trong `emptyDir` `/www` |
| B6: container gỡ lỗi `CreateContainerConfigError`, `runAsNonRoot` | `api` chỉ đặt uid ở cấp container `postgrest`; `busybox` chạy root | Đặt `runAsUser` ở cấp Pod của `api` (B4) |
| B6: `smoke.sh` của tag `lab-9a` fail sau khi `enforce` | Smoke cũ tạo Pod tạm không khai `securityContext` | Đúng hành vi; smoke của Lab 9b ở B12 thay nó |
| B7: Pod `thu-sysctl-khong-an-toan` kẹt `Pending` | `lab-k8s-worker2` mang taint hoặc `NotReady` | Chạy lại gate A5.5; không đổi `nodeSelector` sang worker1 |
| B8: `deploy.sh` dừng ở `co workload vi pham Pod Security` | Workload của tenant thiếu một trường Restricted | Đọc cảnh báo ngay trên dòng báo lỗi; sửa manifest tenant |
| B8: trong staging không phân giải được tên | Policy DNS sai nhãn: `kube-system` thiếu `kubernetes.io/metadata.name`, hoặc Pod CoreDNS không mang `k8s-app: kube-dns` | `kubectl -n kube-system get pods --show-labels`; sửa selector, không bỏ policy |
| B8: Pod của staging bị từ chối `must specify limits` | LimitRange chưa có khi Pod được tạo | Apply cả kustomization một lượt; xóa Pod để tạo lại |
| B8: `teardown.sh` báo còn object, liệt kê PV | Thư mục trên node không xóa được qua `ssh` | Kiểm `ssh lab-k8s-worker2 'sudo ls /opt/local-path-provisioner'`, xóa PV trước, thư mục sau |
| B9: `can-i update deployments` của `ci-deployer` là `no` | Lab 9a cấp `patch` thay vì `update` | Script chấp nhận một trong hai; ghi đúng verb vào rà soát |
| B10: không có header `X-Kubernetes-Pf-*` | Bản `kubectl` in header ở mức log khác | Thử `-v=9`; **không** cài `kubectl` khác |
| B10: dry-run webhook báo lỗi validation | Thiếu `sideEffects` hoặc `admissionReviewVersions` | Bổ sung hai trường; luôn kèm `--dry-run=server` |
| B11: cổng 10250 trả `000` trên cả ba node | `curl` không tới được node, chưa chắc kubelet đóng cổng | Chạy từ `lab-k8s-master`; `000` khác hẳn `401` |
| B12: số container đặc quyền lớn hơn B0 | Pod `thu-dac-quyen` của B7 còn sót | `kubectl delete namespace -l bookstore.example.com/thu=true`; không xóa Pod nào trong `kube-system` |
| B12: checksum cấu hình khác B0 | Cấu hình control plane hoặc kubelet đã bị sửa trong lab | Cluster lệch baseline: restore **cả ba VM** về `03-storage-ready` trước lab sau |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Pod Security Standards](https://v1-35.docs.kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes v1.35 — Pod Security Admission](https://v1-35.docs.kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Kubernetes v1.35 — Pod Security Policies (removed feature)](https://v1-35.docs.kubernetes.io/docs/concepts/security/pod-security-policy/)
- [Kubernetes v1.35 — Enforce Pod Security Standards with Namespace Labels](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)
- [Kubernetes v1.35 — Enforce Pod Security Standards by Configuring the Built-in Admission Controller](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/)
- [Kubernetes v1.35 — Migrate from PodSecurityPolicy to the Built-In PodSecurity Admission Controller](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/migrate-from-psp/)
- [Kubernetes v1.35 — Configure a Security Context for a Pod or Container](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Kubernetes v1.35 — Linux kernel security constraints for Pods and containers](https://v1-35.docs.kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/)
- [Kubernetes v1.35 — Security For Linux Nodes](https://v1-35.docs.kubernetes.io/docs/concepts/security/linux-security/)
- [Kubernetes v1.35 — Using sysctls in a Kubernetes Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/)
- [Kubernetes v1.35 — Good practices for Kubernetes Secrets](https://v1-35.docs.kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [Kubernetes v1.35 — Role Based Access Control Good Practices](https://v1-35.docs.kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Kubernetes v1.35 — Multi-tenancy](https://v1-35.docs.kubernetes.io/docs/concepts/security/multi-tenancy/)
- [Kubernetes v1.35 — Hardening Guide: Authentication Mechanisms](https://v1-35.docs.kubernetes.io/docs/concepts/security/hardening-guide/authentication-mechanisms/)
- [Kubernetes v1.35 — Kubernetes API Server Bypass Risks](https://v1-35.docs.kubernetes.io/docs/concepts/security/api-server-bypass-risks/)
- [Kubernetes v1.35 — Security Checklist](https://v1-35.docs.kubernetes.io/docs/concepts/security/security-checklist/)
- [Kubernetes v1.35 — Application Security Checklist](https://v1-35.docs.kubernetes.io/docs/concepts/security/application-security-checklist/)
- [Kubernetes v1.35 — API Priority and Fairness](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/flow-control/)
- [Kubernetes v1.35 — Admission Webhook Good Practices](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/)
- [Kubernetes v1.35 — Cloud Controller Manager Administration](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/)
- [Kubernetes v1.35 — Debug Running Pods](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [Kubernetes — Admission Controllers Reference](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Kubernetes — Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
- [Kubernetes — Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes — Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Docker Hub — postgres (Arbitrary `--user` Notes, Initialization scripts)](https://hub.docker.com/_/postgres)
- [GitHub — nginxinc/docker-nginx-unprivileged](https://github.com/nginxinc/docker-nginx-unprivileged)
- [GitHub — rancher/local-path-provisioner (README, mục volume type và setup script)](https://github.com/rancher/local-path-provisioner)
