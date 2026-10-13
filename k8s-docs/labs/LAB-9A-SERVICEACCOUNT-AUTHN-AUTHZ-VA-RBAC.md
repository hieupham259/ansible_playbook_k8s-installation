# Lab 9a — ServiceAccount, authn/authz và RBAC

> **Điểm bắt đầu:** snapshot `03-storage-ready` trên **bộ VM chính** (`lab-k8s-master`,
> `lab-k8s-worker1`, `lab-k8s-worker2`) — xem [chuỗi snapshot](README.md#3-chuỗi-snapshot). Repo
> `~/bookstore` ở tag `lab-8c`, mang về từ bundle `bookstore-lab-8c.bundle` của bộ VM HA — B0 làm
> việc này, vì sau khi Lab 8a restore snapshot thì repo trên master vẫn đứng ở `lab-8a`.
> **Điểm kết thúc:** teardown trả cluster về đúng `03-storage-ready`, **không tạo snapshot mới**.
> Repo `~/bookstore` có tag `lab-9a` và một bản bundle nằm ngoài VM. Lab không tạo object cấp
> cluster nào và không sửa cấu hình control plane hay kubelet.
> **Sprint dự án:** Sprint 15 — Danh tính và quyền của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 8c — HA với external etcd](LAB-8C-HA-VOI-EXTERNAL-ETCD.md) (bộ VM riêng).
> **Lab sau:** [Lab 9b — Pod Security và hardening](LAB-9B-POD-SECURITY-VA-HARDENING.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng **phần truy cập** của
[Giai đoạn 9 — Bảo mật và multi-tenancy](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy):
năm bài khái niệm 113, 114, 118, 119, 120 và khối **Thực hành** đứng ngay trước 🧪 Lab 9a trong lộ
trình. Mọi ticket dựa vào hợp đồng kỹ thuật ở
[mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật), thiết kế danh tính ở
[mục 3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật) và bố cục repo ở
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Kế hoạch gốc của sprint nằm ở
[mục 7.2](DU-AN-BOOKSTORE.md#sprint-15--lab-9a-danh-tính-và-quyền).

**Phần policy của giai đoạn 9 không thuộc lab này.** Pod Security Standards, Pod Security
Admission, seccomp/AppArmor, admission webhook và checklist hardening đọc **sau** Lab 9a trong lộ
trình (bài 115, 116 trở đi) và thực hành ở Lab 9b. Vì vậy sprint này **không** gắn nhãn Pod
Security nào lên namespace, kể cả ở chế độ `warn` hay `audit`.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Ba image ứng dụng
nằm ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00); lab này không chép
lại con số phiên bản nào trong phần văn bản và **không thêm image nào** ngoài hai bảng đó.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi hệ thống Bookstore đạt trạng thái sau sprint 15 — **mỗi chủ thể có đúng danh
tính và đúng quyền nó cần, không hơn** — **và** bạn chứng minh, giải thích được:

- Một request tới API server đi qua TLS, rồi **xác thực → phân quyền → kiểm soát tiếp nhận → validate
  rồi ghi**. Với mỗi chặng, chỉ ra một request **của chính hệ thống Bookstore** chết ở đó và dấu hiệu
  nhận ra từ phía client: `401`, `403` kèm tên danh tính, và một câu từ chối nói về **nội dung
  object** trong khi `kubectl auth can-i` vẫn trả `yes`.
- Trước sprint, mọi Pod chạy dưới ServiceAccount `default` và mang token API dù không cần; vì sao
  NetworkPolicy chặn đường tới API server **chưa đủ** để coi token đó vô hại.
- Mỗi thành phần một ServiceAccount; tắt tự động mount ở cấp nào, cấp nào thắng khi hai cấp cùng
  khai; vì sao không đổi được ServiceAccount của Pod đang chạy.
- `db-backup` ghi trạng thái vào ConfigMap `backup-status` qua Kubernetes API với quyền tối thiểu đo
  được: đúng hai verb, đúng một object; token chỉ có trong đúng một container, sống ngắn, và client
  xác minh server bằng CA của cluster. Vì sao công cụ gọi API phải chọn giữa các image đã khóa, và
  NetworkPolicy tới API server trên CNI của cluster phải viết theo đích nào.
- Danh tính CI `ci-deployer`: token có thời hạn, kubeconfig riêng ngoài git, thu hồi ngay bằng cách
  xóa ServiceAccount — và vì sao tạo lại ServiceAccount **cùng tên** là trao lại nguyên quyền cũ.
- Vì sao quyền sửa Deployment gần như bằng quyền đọc Secret và mượn ServiceAccount khác (`TD-15`).
- Kubernetes không có object `User`; quyền của nhóm `bookstore-devs` chỉ dựa vào **tên nhóm** mà chặng
  xác thực trả ra. `kubectl auth can-i --as --as-group` chính là một SubjectAccessReview.
- Quyền gắn debug container là quyền riêng trên subresource — cơ chế giới hạn `TD-07`.
- `securityContext` cấp Pod và cấp container: ai chạy bằng uid nào, cờ `no_new_privs`, và
  `runAsNonRoot` được kiểm ở **kubelet** chứ không ở API server.
- Sysctl an toàn đặt cho một Pod mà không chạm node; sysctl không an toàn được lập lịch nhưng không
  chạy, và dấu vết nằm ở đâu.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong phần truy cập của giai đoạn 9 | Ticket kiểm chứng |
| --- | --- |
| [113 — Bảo mật](../113-security-vi.md) | B1 — đối chiếu năm nhóm cơ chế với cluster: kiểm soát truy cập API (cả sprint), TLS khi truyền, Secret là "bảo vệ cơ bản", admission, audit chưa bật; B6 — admission chặn theo nội dung object |
| [114 — Bảo mật Cloud Native](../114-cloud-native-security-vi.md) | B1 — lĩnh vực *truy cập* của giai đoạn Runtime: mọi Pod thường đều mang một danh tính, kể cả thành phần của cluster; B2 — mỗi workload một danh tính riêng |
| [118 — Tài khoản dịch vụ](../118-service-accounts-vi.md) | B1 — ServiceAccount `default` và trần quyền discovery; B2 — ba bước dùng ServiceAccount, tắt tự động tiêm; B3 — cấp quyền tối thiểu, token qua projected volume; B4 — token có hạn qua TokenRequest, bound token chết theo object; B6 — năm bước kiểm bearer token, sai audience là `401` |
| [119 — Kiểm soát truy cập vào Kubernetes API](../119-controlling-access-vi.md) | B1 — tầng TLS, chặng 1 rút username và group từ client certificate; B5 — chặng 2 và SubjectAccessReview; B6 — dựng lại request chết ở từng chặng và ở bước 4 |
| [120 — Thực hành tốt về RBAC](../120-rbac-good-practices-vi.md) | B3 — đặc quyền tối thiểu cho `db-backup`; B4 — *Tạo workload* (`TD-15`), *Sửa đổi Namespace*, *Yêu cầu token*; B5 — `list` trên Secret, `nodes/proxy`, `escalate`/`bind`/`impersonate`, rà soát định kỳ |
| [254 — Quản trị Cloud Controller Manager](../254-running-cloud-controller-vi.md) | B1 — cluster **không có** cloud-controller-manager: không cờ `--cloud-provider=external`, không taint `uninitialized`; mục *Yêu cầu* của bài thành một dòng trong ma trận danh tính (CCM nếu có cũng là một chủ thể cần RBAC). Không thực hành gì thêm — lab không có nhà cung cấp cloud |
| [258 — Sử dụng sysctl](../258-sysctl-cluster-vi.md) | B8 — sysctl an toàn cho `api` qua `securityContext` cấp Pod, giá trị trong Pod khác trên node; diễn tập sysctl không an toàn bị kubelet từ chối |
| [279 — Cấu hình Service Account cho Pod](../279-configure-service-account-vi.md) | B1 — plugin ServiceAccount chèn ServiceAccount, volume token và `imagePullSecrets`; B2 — từ chối tự động mount ở hai cấp, `serviceAccountName` bất biến; B3 — chiếu token vào volume; B4 — `kubectl create token --duration`, ⭐ issuer discovery |
| [282 — Thực thi PSS bằng cấu hình admission controller](../282-enforce-standards-admission-controller-vi.md) | **Không thực hành ở lab này.** Cần khái niệm của bài 115, 116 — lộ trình xếp **sau** Lab 9a — và phần áp dụng là sửa cờ `kube-apiserver`. Thực hành ở Lab 9b |
| [283 — Thực thi PSS bằng nhãn namespace](../283-enforce-standards-namespace-labels-vi.md) | **Không thực hành ở lab này**, cùng lý do với bài 282: nhãn Pod Security (kể cả `warn`/`audit`) là việc của Lab 9b |
| [286 — Di chuyển từ PodSecurityPolicy](../286-migrate-from-psp-vi.md) | B4 — chỉ mục 1 *Rà soát quyền trên namespace*: chứng minh CI không sửa được Namespace `bookstore`. Phần còn lại cần PSS — Lab 9b |
| [291 — Cấu hình Security Context](../291-security-context-vi.md) | B7 — `runAsUser`/`runAsGroup` cấp Pod, ghi đè cấp container, `allowPrivilegeEscalation` và `NoNewPrivs`, group ngầm định từ `/etc/group`; B8 — `securityContext.sysctls` cấp Pod. Phần `readOnlyRootFilesystem`, capability, seccomp thuộc Lab 9b |
| [338 — Truy cập Kubernetes API từ một Pod](../338-access-api-from-pod-vi.md) | B1 — Pod gọi discovery bằng token tự mount; B3 — container báo trạng thái gọi REST trực tiếp, không proxy: ba file, CA, `kubernetes.default.svc` |
| [359 — Truy cập cluster](../359-access-cluster-vi.md) | B1 — `kubectl config view`, client certificate; B4 — kubeconfig cho ServiceAccount, `kubectl proxy` mang đúng quyền của kubeconfig, ⭐ các loại proxy có thật trên cluster |
| [190 — Truy cập cluster bằng Kubernetes API](../190-access-cluster-api-vi.md) | B4 — gọi REST thẳng bằng token kèm CA thay cho `--insecure`; `kubectl proxy` bằng kubeconfig của CI |

Các phần dưới đây **không kiểm chứng được** trong lab này:

| Phần | Vì sao |
| --- | --- |
| Bài 113, 119: mã hóa khi lưu trữ, audit log, ABAC, webhook phân quyền, bật/tắt admission plugin | Đều là sửa cờ `kube-apiserver`; lab cấm sửa cấu hình control plane và biến điều đó thành gate. Mã hóa Secret là **nợ lab #6**, trả ở [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) |
| Bài 118: TokenReview, SPIFFE, OIDC, giới hạn audience trên node | Dành cho người viết dịch vụ xác minh token, hoặc là hardening danh tính kubelet — bài [123](../123-hardening-authentication-vi.md), [128](../128-api-server-bypass-risks-vi.md), Lab 9b |
| Bài 279: kéo image thật từ registry riêng | Cần registry và tài khoản thật; B1 chỉ kiểm phần thuộc chặng tiếp nhận |
| Bài 338: thư viện client Go/Python, `kubectl proxy` dạng sidecar | Cần cài SDK hoặc image có `kubectl` — ngoài A1.3/A1.4 |
| Bài 359: proxy hoặc load balancer trước API server, cloud load balancer | Cluster chính có một control plane, không có cloud. Load balancer trước API server là nội dung Lab 8b, 8c |

Lab **không phát sinh nợ lab mới** và **không trả nợ lab nào**. Nợ kỹ thuật: **giới hạn `TD-07`**
(quyền gỡ lỗi thành quyền riêng của nhóm trực), **phát sinh `TD-15`** (quyền sửa Deployment của CI
gián tiếp đọc được Secret); `TD-06` vẫn mở, trả ở 9b.

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B9, trừ các mục ⭐) khoảng 4 giờ. Ticket dài nhất là B3 (khoảng 50 phút)
vì có một điều tra NetworkPolicy. Có thể chia hai phiên: phiên 1 làm B0–B4, phiên 2 làm B5–B9.
Giữa hai phiên cứ để nguyên cluster; đầu phiên 2 chạy lại gate ngắn A5.5 và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` bằng kubeconfig quản trị, trừ khi ghi rõ node khác.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Mọi Pod thử nghiệm và diễn tập ghim node này.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/9a/` (quyền `700`, vì có lúc chứa
  certificate và token). Bằng chứng: `~/lab-evidence/9a/`.
- Lab chỉ dùng kiến thức tới phần truy cập của giai đoạn 9 và các giai đoạn trước. **Không** gắn
  nhãn Pod Security, không dùng ValidatingAdmissionPolicy hay admission webhook, không
  `readOnlyRootFilesystem`, capability, seccomp — các thứ đó thuộc Lab 9b.
- **An toàn bí mật:** không bao giờ ghi token thô vào `~/lab-evidence/9a/`. Bằng chứng chỉ chứa mã
  HTTP, câu thông báo của API server, kết quả `kubectl auth whoami`/`can-i` và **claim đã giải mã**.
  Token chỉ sống trong biến shell, trong `secrets/` của repo (quyền `600`, bị `.gitignore` chặn) hoặc
  trong `~/lab-work/9a/`. Gate cuối quét thư mục bằng chứng.
- **An toàn quyền:** không cấp `cluster-admin`, không thêm ai vào `system:masters`, không tạo
  ClusterRole hay ClusterRoleBinding — mọi quyền của dự án nằm trong namespace `bookstore`. Cờ `--as`
  và `--as-group` chỉ dùng để **kiểm chứng**; bạn có quyền mạo danh vì kubeconfig quản trị có nó.
- **Không sửa** `/etc/kubernetes/manifests/*` hay cấu hình kubelet. B0 ghi checksum, B9 đối chiếu.
- Object thử nghiệm không mang label `app.kubernetes.io/part-of`, nằm trong `bookstore` hoặc trong
  namespace thử nghiệm `lab-9a-thu` (chỉ sống trong B1), và bị xóa khi xong ticket. Pod thử nghiệm
  trong `bookstore` khai `requests`/`limits` nhỏ vì namespace có hạn mức từ Lab 7b.
- `kubectl patch`, `kubectl debug`, `kubectl delete serviceaccount` chỉ dùng trên object thử nghiệm
  hoặc trong diễn tập. Repo là nguồn sự thật duy nhất của hệ thống.
- Chạy mỗi script nghiệm thu trong một shell liền mạch. Dòng `PASS:` là gate; dòng `FAIL:` nghĩa là
  ticket chưa xong.

---

# Phần B — Sprint 15: Danh tính và quyền

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận bộ VM chính ở đúng `03-storage-ready`, đưa repo ở tag `lab-8c` từ bộ VM HA về
`lab-k8s-master`, và chứng minh hệ thống của sprint 11 dựng lại được **chỉ từ repo**.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Đúng bộ VM, đúng snapshot, cluster sạch:

```bash
mkdir -p ~/lab-work/9a ~/lab-evidence/9a && chmod 700 ~/lab-work/9a
{ test "$(kubectl get nodes --no-headers | wc -l)" -eq 3 && \
  kubectl get node lab-k8s-master lab-k8s-worker1 lab-k8s-worker2 >/dev/null; } \
  && echo 'PASS: dang o bo VM chinh, ba node lab-k8s-*'
kubectl get storageclass | grep -q '(default)' && echo 'PASS: co StorageClass mac dinh — dung 03-storage-ready'
kubectl get pods -A --no-headers | grep -q 'calico-node-' && echo 'PASS: Calico dang chay'
kubectl -n traefik get deployment traefik >/dev/null 2>&1 && echo 'PASS: Traefik dang chay'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot' || echo 'PASS: cluster chua co du an'
test -z "$(kubectl get nodes -l bookstore.example.com/data -o name)" \
  && echo 'PASS: chua node nao mang label du lieu cua du an'
```

Mang repo về. Bundle `bookstore-lab-8c.bundle` được chép ra máy host ở cuối Lab 8c. Trên máy host
Windows, PowerShell, sửa user cho đúng máy của bạn:

```powershell
scp "$HOME\lab-backup\bookstore-lab-8c.bundle" <user>@192.168.100.221:~/
```

Trên `lab-k8s-master`:

```bash
git bundle verify ~/bookstore-lab-8c.bundle >/dev/null && echo 'PASS: bundle lab-8c hop le'
git bundle list-heads ~/bookstore-lab-8c.bundle | grep -q 'refs/tags/lab-8c$' && echo 'PASS: bundle co tag lab-8c'
if [ -d ~/bookstore/.git ] && ! git -C ~/bookstore describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-8c'; then
  mv ~/bookstore ~/bookstore-truoc-9a && echo 'repo cu da doi ten thanh ~/bookstore-truoc-9a'
fi
test -d ~/bookstore/.git || git clone -q ~/bookstore-lab-8c.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-8c' && echo 'PASS: HEAD dung tag lab-8c'
test "$(git rev-parse --abbrev-ref HEAD)" != 'HEAD' && echo 'PASS: HEAD dang o mot nhanh, commit duoc'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
git diff --quiet lab-7b lab-8c -- k8s/app && echo 'PASS: k8s/app o lab-8c trung lab-7b — he thong cua sprint 11'
```

Repo mới clone không có `secrets/` vì thư mục đó không vào git. Không cần chép từ repo cũ:
`deploy.sh` tự sinh bí mật mới và certificate tự ký, DB được khởi tạo lại trên PVC mới
([mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm)).

Dựng lại hệ thống của sprint 11 bằng script của chính tag đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/9a/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/9a/b0-smoke-lab-8c.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/9a/b0-smoke-lab-8c.txt
```

Image cho sprint này: container gọi API ở B3 chạy trên một worker bất kỳ. Tag lấy từ
[A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và
[A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00):

```bash
for N in lab-k8s-worker1 lab-k8s-worker2; do
  for I in docker.io/library/busybox:1.37 docker.io/library/postgres:17.11 \
           docker.io/nginxinc/nginx-unprivileged:1.30.5-alpine; do
    ssh "$N" "sudo crictl pull $I" >/dev/null && echo "PASS: $N co $I" || echo "FAIL: $N khong keo duoc $I"
  done
done
```

Ảnh chụp "trước" cho gate cuối — cấu hình control plane và tồn kho RBAC cấp cluster:

```bash
EV=~/lab-evidence/9a
{ echo "apiserver-manifest $(sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml | awk '{print $1}')"
  echo "kubelet-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "kubelet-$n $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done; } > "$EV/b0-cauhinh-sha.txt"
{ echo "clusterrole=$(kubectl get clusterrole --no-headers | wc -l)"
  echo "clusterrolebinding=$(kubectl get clusterrolebinding --no-headers | wc -l)"; } > "$EV/b0-rbac-cluster.txt"
cat "$EV/b0-cauhinh-sha.txt" "$EV/b0-rbac-cluster.txt"
test "$(grep -c '[0-9a-f]\{64\}' "$EV/b0-cauhinh-sha.txt")" -eq 4 && echo 'PASS: ghi duoc bon checksum cau hinh'
```

**PASS:** gate A5.5 đạt; sáu dòng `PASS:` về bộ VM và cluster; sáu dòng `PASS:` về bundle và repo;
`deploy exit=0`, `smoke exit=0` và file smoke không có dòng `FAIL:`; đủ sáu dòng `PASS: ... co ...`;
dòng `PASS: ghi duoc bon checksum cau hinh`.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint trước **chưa đủ**. Sửa repo ở tag `lab-8c`
trước, không sửa tay trên cluster ([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-9A-01] Điều tra: hệ thống đang mang những danh tính nào

**Loại:** Điều tra.

**Bối cảnh:** sau sprint 11, hệ thống chạy đủ, mạng đã zero-trust, nhưng chưa ai hỏi câu "**ai**
được gọi Kubernetes API, và gọi được gì". Trước khi cấp hay thu quyền, đội cần bản đồ: request đi
qua những chặng nào, mỗi Pod đang mang danh tính gì, danh tính đó làm được gì. Kết quả là ma trận
danh tính đích — bảng ở [mục 3.5 của file dự án](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật) áp lên hệ
thống thật — làm đầu vào cho B2–B5.

**Yêu cầu:**

1. Chỉ ra danh tính mà kubeconfig quản trị trình cho API server — username, các group — và chúng
   nằm ở trường nào của client certificate.
2. ⭐ Chứng minh tầng TLS đứng trước chặng 1: cổng API chỉ nói HTTPS; client không có CA của cluster
   thì không tin server, có CA thì bắt tay xong.
3. Lập bảng mọi Pod của dự án: chạy dưới ServiceAccount nào, có token API trong filesystem không.
   Đối chiếu toàn cluster: Pod thường nào cũng có danh tính; ngoại lệ là gì.
4. Chứng minh ServiceAccount `default` của `bookstore` chỉ có quyền discovery — bằng câu hỏi gửi chặng
   phân quyền, không bằng phỏng đoán.
5. Trong namespace thử nghiệm `lab-9a-thu` (không có NetworkPolicy), một Pod `busybox` dùng token tự
   mount gọi API: discovery thành công, đọc ConfigMap của namespace bị từ chối. Chỉ ra **ba** thứ
   chặng tiếp nhận đã thêm vào Pod mà manifest của bạn không khai — trong đó có `imagePullSecrets`
   khi ServiceAccount của namespace có khai.
6. Trong `bookstore`: Pod `web` đang mang token nhưng không tới được API server. Chứng minh token đó,
   một khi bị đọc ra, **vẫn** dùng được từ máy khác.
7. Đối chiếu với cluster: audit log có bật không; admission plugin nào được khai tường minh; có
   cloud-controller-manager không; kube-controller-manager gọi API dưới danh tính nào.
8. Ghi ADR: ma trận danh tính đích — mỗi thành phần, CI, con người: có cần gọi API không, verb gì
   trên object gì, phạm vi nào.

**Ràng buộc:** không sửa cấu hình control plane. Namespace `lab-9a-thu` không mang label dự án và bị
xóa cuối ticket. Pod thử nghiệm ghim `lab-k8s-worker2`. Không ghi token vào bằng chứng.

**Đọc lại:**

- Bài [119](../119-controlling-access-vi.md):
  [*Bảo mật tầng truyền tải*](../119-controlling-access-vi.md#bảo-mật-tầng-truyền-tải-transport-security),
  [*Xác thực*](../119-controlling-access-vi.md#authentication),
  [*Phân quyền*](../119-controlling-access-vi.md#authorization),
  [*Kiểm soát tiếp nhận*](../119-controlling-access-vi.md#kiểm-soát-tiếp-nhận-admission-control).
- Bài [118](../118-service-accounts-vi.md):
  [*Service account mặc định*](../118-service-accounts-vi.md#default-service-accounts),
  [*Gán ServiceAccount cho một Pod*](../118-service-accounts-vi.md#assign-to-pod).
- Bài [338](../338-access-api-from-pod-vi.md):
  [*Truy cập trực tiếp REST API*](../338-access-api-from-pod-vi.md#truy-cập-trực-tiếp-rest-api-directly-accessing-the-rest-api),
  [*Không sử dụng proxy*](../338-access-api-from-pod-vi.md#không-sử-dụng-proxy-without-using-a-proxy).
- Bài [359](../359-access-cluster-vi.md#truy-cập-lần-đầu-với-kubectl-accessing-for-the-first-time-with-kubectl)
  — vị trí và thông tin xác thực trong kubeconfig.
- Bài [279](../279-configure-service-account-vi.md#add-imagepullsecrets-to-a-service-account) —
  `imagePullSecrets` gắn ở ServiceAccount.
- Bài [113](../113-security-vi.md#security-mechanisms); bài
  [114](../114-cloud-native-security-vi.md#protection-runtime-access); bài
  [254](../254-running-cloud-controller-vi.md#chạy-cloud-controller-manager-running-cloud-controller-manager).

**Bàn giao:** `docs/adr/NNNN-ban-do-danh-tinh.md` (`NNNN` là số tiếp theo trong `docs/adr/`);
`~/lab-work/9a/goi-api.yaml`; `~/lab-evidence/9a/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` username của chặng 1 trùng CN của client certificate; group lấy từ trường O.
- `PASS:` (⭐) HTTP thuần tới cổng API bị từ chối; không CA thì `curl` thất bại, có CA thì thành công.
- `PASS:` trước sprint, mọi Pod dự án chạy dưới `default` và mang volume token; mọi Pod thường của
  cluster có ServiceAccount, Pod không có đều là mirror Pod.
- `PASS:` `default` của `bookstore` được `get /api`, không được `list` ConfigMap hay Secret.
- `PASS:` Pod thử nghiệm gọi được discovery, bị `403` khi đọc ConfigMap; object có ServiceAccount,
  volume token và `imagePullSecrets` dù manifest không khai.
- `PASS:` Pod `web` có file token nhưng không tới được API server; token đó dùng từ master vẫn `200`.
- `PASS:` audit chưa bật; không có cloud-controller-manager; kube-controller-manager dùng
  ServiceAccount riêng cho từng controller.
- `PASS:` namespace `lab-9a-thu` đã bị xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-ban-do-danh-tinh.md`:

1. NetworkPolicy đã chặn `web` gọi API server. Vậy vì sao vẫn phải gỡ token khỏi `web`? Hai lớp
   phòng thủ này bảo vệ khỏi hai rủi ro khác nhau nào?
2. Ba thứ chặng tiếp nhận thêm vào Pod thử nghiệm do ai thêm, và vì sao việc đó nằm ở chặng 3 chứ
   không ở chặng 2?
3. Bài 254 nói cloud-controller-manager cần xác thực và phân quyền với Kubernetes. Nếu cluster này
   chạy trên cloud, ma trận danh tính của bạn thêm dòng nào — và nó giống `ci-deployer` ở điểm nào?
4. Ma trận đích: thành phần nào cần gọi API? Viết bảng `chủ thể | loại | verb | resource | phạm vi`.

<details>
<summary>Gợi ý</summary>

- `kubectl auth whoami` cho kết quả của chặng 1. Client certificate nằm trong kubeconfig dạng base64
  ở `users[0].user.client-certificate-data`; đọc subject bằng
  `openssl x509 -noout -subject -nameopt multiline`.
- Danh sách volume của Pod: `-o jsonpath='{.spec.volumes[*].name}'`. Volume token tự động có tên
  bắt đầu bằng `kube-api-access-`.
- Hỏi chặng 2 thay cho một chủ thể: `kubectl auth can-i list configmaps -n bookstore
  --as=system:serviceaccount:bookstore:default`; xem toàn bộ bằng `--list`.
- Pod thử nghiệm chỉ cần `image`, `command: ["sh", "-c", "sleep 3600"]` và `nodeSelector`. Gọi API từ
  trong Pod đúng như đoạn shell của bài 338, thay `curl` bằng `wget --header ... --no-check-certificate`
  của busybox.
- `kubectl exec ... -- timeout 10 wget ...`: busybox có `timeout`; không có nó, một kết nối bị
  NetworkPolicy chặn có thể treo rất lâu.
- Cờ của kube-controller-manager nằm ở `/etc/kubernetes/manifests/kube-controller-manager.yaml`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã viết <code>~/lab-work/9a/goi-api.yaml</code></summary>

```bash
NS=bookstore; EV=~/lab-evidence/9a; W=~/lab-work/9a
SRV="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  | base64 -d > "$W/ca.crt"

# 1. Danh tính của kubeconfig quản trị
kubectl auth whoami > "$EV/b1-whoami-admin.txt"
U="$(kubectl auth whoami -o jsonpath='{.status.userInfo.username}')"
G="$(kubectl auth whoami -o jsonpath='{.status.userInfo.groups}')"
kubectl config view --raw --minify -o jsonpath='{.users[0].user.client-certificate-data}' \
  | base64 -d > "$W/admin-client.crt"
SUBJ="$(openssl x509 -noout -subject -nameopt multiline -in "$W/admin-client.crt")"
CN="$(echo "$SUBJ" | sed -n 's/^ *commonName *= *//p')"
O="$(echo "$SUBJ" | sed -n 's/^ *organizationName *= *//p')"
echo "username=$U | CN=$CN | groups=$G | O=$O" | tee -a "$EV/b1-whoami-admin.txt"
[ -n "$CN" ] && [ "$CN" = "$U" ] && echo 'PASS: username o chang 1 lay tu CN cua client certificate'
[ -n "$O" ] && echo "$G" | grep -qF "$O" && echo 'PASS: group lay tu truong O cua certificate'

# 2. ⭐ Tầng TLS
curl -s -m 5 "http://${SRV#https://}/version" > "$EV/b1-http-tho.txt"
grep -qi 'HTTP request to an HTTPS server' "$EV/b1-http-tho.txt" && echo 'PASS: cong API chi noi HTTPS'
curl -s -m 5 -o /dev/null "$SRV/version"; RC0=$?
curl -s -m 5 -o /dev/null --cacert "$W/ca.crt" "$SRV/version"; RC1=$?
echo "khong CA: exit=$RC0 | co CA: exit=$RC1" | tee "$EV/b1-tls.txt"
[ "$RC0" -ne 0 ] && [ "$RC1" -eq 0 ] && echo 'PASS: khong co CA thi khong tin server, co CA thi bat tay xong'

# 3. Danh tính của từng Pod
kubectl -n "$NS" get pods -l app.kubernetes.io/part-of=bookstore \
  -o jsonpath='{range .items[*]}{.metadata.name}{"|"}{.spec.serviceAccountName}{"|"}{range .spec.volumes[*]}{.name}{","}{end}{"\n"}{end}' \
  > "$EV/b1-pod-sa-truoc.txt"
TONG="$(grep -c . "$EV/b1-pod-sa-truoc.txt")"
DEF="$(awk -F'|' '$2=="default"' "$EV/b1-pod-sa-truoc.txt" | wc -l)"
TOK="$(grep -c 'kube-api-access-' "$EV/b1-pod-sa-truoc.txt")"
echo "pod du an=$TONG | duoi default=$DEF | co volume token=$TOK"
[ "$TONG" -gt 0 ] && [ "$DEF" -eq "$TONG" ] && [ "$TOK" -eq "$TONG" ] \
  && echo 'PASS: truoc sprint, moi Pod du an chay duoi default va mang token API'
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"|"}{.metadata.name}{"|"}{.spec.serviceAccountName}{"|"}{.metadata.annotations.kubernetes\.io/config\.mirror}{"\n"}{end}' \
  > "$EV/b1-pod-sa-cluster.txt"
KHONG="$(awk -F'|' '$3==""' "$EV/b1-pod-sa-cluster.txt" | wc -l)"
MIRROR="$(awk -F'|' '$3=="" && $4!=""' "$EV/b1-pod-sa-cluster.txt" | wc -l)"
[ "$KHONG" -eq "$MIRROR" ] && echo "PASS: moi Pod thuong co ServiceAccount; $MIRROR Pod khong co deu la mirror Pod"

# 4. Trần quyền của default
D='system:serviceaccount:bookstore:default'
kubectl auth can-i --list -n "$NS" --as="$D" > "$EV/b1-quyen-default.txt"
{ [ "$(kubectl auth can-i get /api --as="$D")" = yes ] && \
  [ "$(kubectl auth can-i list configmaps -n "$NS" --as="$D")" = no ] && \
  [ "$(kubectl auth can-i list secrets -n "$NS" --as="$D")" = no ]; } \
  && echo 'PASS: default cua bookstore chi co quyen discovery'

# 5. Pod gọi API trong namespace không có NetworkPolicy
kubectl create namespace lab-9a-thu
until kubectl -n lab-9a-thu get serviceaccount default >/dev/null 2>&1; do sleep 1; done
kubectl -n lab-9a-thu create secret docker-registry registry-gia --docker-server=registry.vi-du.local \
  --docker-username=DUMMY_USERNAME --docker-password=DUMMY_DOCKER_PASSWORD
kubectl -n lab-9a-thu patch serviceaccount default -p '{"imagePullSecrets":[{"name":"registry-gia"}]}'
kubectl apply -f "$W/goi-api.yaml"
kubectl -n lab-9a-thu wait --for=condition=Ready pod/goi-api --timeout=180s
BAD=0
for F in serviceAccountName volumes imagePullSecrets; do grep -q "$F" "$W/goi-api.yaml" && BAD=1; done
P_SA="$(kubectl -n lab-9a-thu get pod goi-api -o jsonpath='{.spec.serviceAccountName}')"
P_VOL="$(kubectl -n lab-9a-thu get pod goi-api -o jsonpath='{.spec.volumes[*].name}')"
P_IPS="$(kubectl -n lab-9a-thu get pod goi-api -o jsonpath='{.spec.imagePullSecrets[*].name}')"
echo "sa=$P_SA | volumes=$P_VOL | imagePullSecrets=$P_IPS" | tee "$EV/b1-admission-them.txt"
{ [ "$BAD" -eq 0 ] && [ "$P_SA" = default ] && echo "$P_VOL" | grep -q 'kube-api-access-' && \
  [ "$P_IPS" = registry-gia ]; } \
  && echo 'PASS: chang tiep nhan da them ServiceAccount, volume token, imagePullSecrets ma manifest khong khai'
kubectl -n lab-9a-thu exec goi-api -- sh -c '
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  wget -O- --no-check-certificate --header "Authorization: Bearer $(cat $SA/token)" \
    https://kubernetes.default.svc/api 2>&1' > "$EV/b1-pod-discovery.txt"
kubectl -n lab-9a-thu exec goi-api -- sh -c '
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  wget -O- --no-check-certificate --header "Authorization: Bearer $(cat $SA/token)" \
    "https://kubernetes.default.svc/api/v1/namespaces/$(cat $SA/namespace)/configmaps" 2>&1' \
  > "$EV/b1-pod-configmaps.txt"
grep -q 'APIVersions' "$EV/b1-pod-discovery.txt" && echo 'PASS: Pod goi duoc discovery bang token cua chinh no'
grep -q '403' "$EV/b1-pod-configmaps.txt" && echo 'PASS: cung token do bi 403 khi doc ConfigMap'

# 6. web: có token, không có đường — nhưng token rời Pod vẫn dùng được
kubectl -n "$NS" exec deploy/web -c status-writer -- test -s /var/run/secrets/kubernetes.io/serviceaccount/token \
  && echo 'PASS: Pod web dang mang token API'
kubectl -n "$NS" exec deploy/web -c status-writer -- timeout 10 wget -qO- --no-check-certificate \
  https://kubernetes.default.svc/version > "$EV/b1-web-goi-api.txt" 2>&1 \
  && echo 'FAIL: web goi duoc API server' || echo 'PASS: NetworkPolicy chan web toi API server'
TOK_WEB="$(kubectl -n "$NS" exec deploy/web -c status-writer -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
C="$(curl -s -o /dev/null -w '%{http_code}' --cacert "$W/ca.crt" -H "Authorization: Bearer $TOK_WEB" "$SRV/api")"
unset TOK_WEB
echo "token cua web dung tu lab-k8s-master: GET /api = $C" | tee -a "$EV/b1-web-goi-api.txt"
[ "$C" = 200 ] && echo 'PASS: token lay ra khoi Pod van xac thuc duoc — NetworkPolicy khong bao ve token'

# 7. Bài 113, 254 và danh tính của control plane
AUDIT="$(sudo grep -cE 'audit-policy-file|audit-log-path' /etc/kubernetes/manifests/kube-apiserver.yaml || true)"
sudo grep -E 'enable-admission-plugins' /etc/kubernetes/manifests/kube-apiserver.yaml > "$EV/b1-admission-plugins.txt"
[ "$AUDIT" -eq 0 ] && echo 'PASS: audit log chua bat tren cluster nay'
CCM="$(kubectl get pods -A --no-headers | grep -c 'cloud-controller-manager' || true)"
TAINT="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.taints[*].key}{"\n"}{end}' | grep -c 'node.cloudprovider.kubernetes.io/uninitialized' || true)"
EXT="$(sudo grep -c 'cloud-provider=external' /etc/kubernetes/manifests/kube-controller-manager.yaml || true)"
echo "pod ccm=$CCM | taint uninitialized=$TAINT | cloud-provider=external=$EXT" | tee "$EV/b1-ccm.txt"
[ "$CCM" -eq 0 ] && [ "$TAINT" -eq 0 ] && [ "$EXT" -eq 0 ] && echo 'PASS: cluster khong co cloud-controller-manager'
sudo grep -q -- '--use-service-account-credentials=true' /etc/kubernetes/manifests/kube-controller-manager.yaml \
  && echo 'PASS: kube-controller-manager goi API bang ServiceAccount rieng cho tung controller'

# 8. Dọn
kubectl delete namespace lab-9a-thu --wait=true --timeout=180s
kubectl get namespace lab-9a-thu >/dev/null 2>&1 || echo 'PASS: da xoa namespace lab-9a-thu'
rm -f "$W/admin-client.crt"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/9a/goi-api.yaml — cố ý không khai serviceAccountName, volumes, imagePullSecrets
apiVersion: v1
kind: Pod
metadata:
  name: goi-api
  namespace: lab-9a-thu
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
```

Ma trận đích cho ADR — khớp [mục 3.5 của file dự án](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật), thêm
nhóm trực sự cố cho `TD-07`:

| Chủ thể | Loại | Verb | Resource | Phạm vi |
| --- | --- | --- | --- | --- |
| `web`, `api`, `db`, `db-migrate`, `node-agent` | ServiceAccount, không token | — | — | — |
| `db-backup` | ServiceAccount, token chiếu tay | `get`, `patch` | ConfigMap `backup-status` | `bookstore` |
| `ci-deployer` | ServiceAccount, token có hạn | đọc, `patch`, `update` / `create` | Deployment, ReplicaSet (đọc) / ConfigMap | `bookstore` |
| nhóm `bookstore-devs` | Group | `get`, `list`, `watch` (ClusterRole `view`) | hầu hết object, **không** Secret | `bookstore` |
| nhóm `bookstore-oncall` | Group | `patch` / `get`, `create` | `pods/ephemeralcontainers` / `pods/exec`, `pods/attach` | `bookstore` |

Ý chính cho ADR:

1. NetworkPolicy chặn **đường mạng** từ Pod; token là **credential**. Ai đọc được filesystem của Pod
   (`exec`, debug container, lỗ hổng trong ứng dụng) mang token ra chỗ khác và dùng được — dòng `200`
   từ master là bằng chứng. Gỡ token là xóa luôn credential; hai lớp bổ sung nhau.
2. Plugin admission `ServiceAccount`: điền `default`, thêm projected volume `kube-api-access-*` và mount
   của nó, chép `imagePullSecrets` từ ServiceAccount. Chặng 2 chỉ thấy username, verb, resource;
   chỉ chặng 3 thấy và **sửa được** nội dung object (bài 119).
3. CCM là một tiến trình gọi API như mọi chủ thể khác: bài 254 mục *Yêu cầu* nói nó có thể cần quy
   tắc RBAC. Giống `ci-deployer`: một danh tính phi con người, cần quyền tối thiểu và quy trình thu hồi.
   Trên cluster này, chính các controller trong kube-controller-manager cũng là những
   ServiceAccount riêng trong `kube-system` — B6 sẽ dùng một cái.
4. Bảng ở trên.

</details>

## B2. [BS-9A-02] Mỗi thành phần một ServiceAccount

**Loại:** Tính năng.

**Bối cảnh:** B1 cho thấy mọi Pod của dự án chạy dưới `default` và mang token mà không thành phần nào
dùng. Sprint này đặt danh tính riêng cho từng thành phần — để sau này cấp quyền cho một thành phần
không vô tình cấp cho cả namespace — và thu hồi token ở mọi nơi không cần.

**Yêu cầu:**

1. Sáu ServiceAccount `web`, `api`, `db`, `db-migrate`, `db-backup`, `node-agent` trong `bookstore`, label
   theo hợp đồng: `part-of` và `component` của thành phần tương ứng (`db-migrate` là `migrate`,
   `db-backup` là `backup`, `node-agent` là `agent`). Manifest ở `k8s/app/rbac/serviceaccounts.yaml`.
2. Mỗi workload chạy dưới ServiceAccount của nó: Deployment `web`, `api`; StatefulSet `db`; Job
   `db-migrate`; CronJob `db-backup`; DaemonSet `node-agent`.
3. Không Pod nào của dự án có token API trong filesystem. Thiết lập đặt ở **ServiceAccount**, kể cả
   ServiceAccount `default` của `bookstore`.
4. Hệ thống vẫn đạt toàn bộ smoke của tag `lab-8c`.
5. Chứng minh hai điều của bài 279: đổi ServiceAccount của một Pod đang chạy bị API server từ chối;
   Pod tự khai tự mount ở cấp Pod **thắng** thiết lập của ServiceAccount.

**Ràng buộc:** chưa cấp Role nào trong ticket này. Pod thử nghiệm ở `~/lab-work/9a/`, không mang label
dự án, xóa khi xong. Không sửa ServiceAccount bằng `kubectl edit` — mọi thay đổi đi qua repo.

**Đọc lại:**

- Bài [118](../118-service-accounts-vi.md): [*Cách sử dụng service account*](../118-service-accounts-vi.md#how-to-use),
  [*Gán ServiceAccount cho một Pod*](../118-service-accounts-vi.md#assign-to-pod),
  [*Các trường hợp sử dụng*](../118-service-accounts-vi.md#use-cases).
- Bài [279](../279-configure-service-account-vi.md):
  [*Từ chối tự động mount thông tin xác thực API*](../279-configure-service-account-vi.md#từ-chối-tự-động-mount-thông-tin-xác-thực-api-opt-out-of-api-credential-automounting),
  [*Sử dụng nhiều hơn một ServiceAccount*](../279-configure-service-account-vi.md#use-multiple-service-accounts).
- Bài [120](../120-rbac-good-practices-vi.md): [*Gia cố*](../120-rbac-good-practices-vi.md#gia-cố-hardening),
  [*Giảm thiểu việc phân phối các token đặc quyền*](../120-rbac-good-practices-vi.md#giảm-thiểu-việc-phân-phối-các-token-đặc-quyền-minimize-distribution-of-privileged-tokens).

**Bàn giao:** `k8s/app/rbac/serviceaccounts.yaml`; `k8s/app/kustomization.yaml`; sáu manifest workload;
`~/lab-work/9a/sa-ep-token.yaml`; `docs/adr/NNNN-serviceaccount-tung-thanh-phan.md`;
`~/lab-evidence/9a/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` `teardown.sh` rồi `deploy.sh` từ repo thành công.
- `PASS:` sáu ServiceAccount đúng tên, đúng label, tắt tự mount; `default` của `bookstore` cũng tắt.
- `PASS:` sáu workload chạy dưới đúng ServiceAccount của mình.
- `PASS:` không Pod nào của dự án có volume token tự động; trong Pod `web` không còn file token.
- `PASS:` smoke của tag `lab-8c` vẫn PASS.
- `PASS:` đổi ServiceAccount của Pod `api` đang chạy bị từ chối.
- `PASS:` Pod thử nghiệm dùng ServiceAccount `web` nhưng khai tự mount ở cấp Pod thì có token.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-serviceaccount-tung-thanh-phan.md`:

1. Tắt tự mount ở ServiceAccount, nhưng yêu cầu 5 cho thấy người viết Pod ghi đè được. Vậy thứ gì
   thật sự ngăn một người đưa token trở lại Pod? (Đọc mục *Tạo workload* của bài 120.)
2. Năm trong sáu thành phần không gọi API. Vì sao vẫn tách ServiceAccount riêng thay vì để chung
   `default` đã tắt token?
3. Đổi `serviceAccountName` làm mọi Pod của dự án bị thay. Vì sao — nói bằng tính bất biến của field
   này trên Pod và bằng template của controller.
4. `node-agent` chạy trên cả ba node, kể cả control plane. Nếu ServiceAccount của nó có quyền mạnh,
   bài 120 cảnh báo gì về DaemonSet?

<details>
<summary>Gợi ý</summary>

- Field trên ServiceAccount: `automountServiceAccountToken` (cấp object, không nằm trong `spec`).
- Field trên Pod template: `serviceAccountName`. CronJob đặt nó ở
  `spec.jobTemplate.spec.template.spec`.
- ServiceAccount `default` đã tồn tại sẵn; `kubectl apply` một manifest cùng tên chỉ sửa nó.
- Thử đổi field bất biến mà không làm hỏng gì: `kubectl patch ... --dry-run=server`.
- Pod thử nghiệm cho yêu cầu 5: `serviceAccountName: web`, `automountServiceAccountToken: true` ở
  `spec`, ghim worker2, có `requests`/`limits`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/9a; W=~/lab-work/9a
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

./scripts/teardown.sh > "$EV/b2-teardown.txt" 2>&1 && ./scripts/deploy.sh > "$EV/b2-deploy.txt" 2>&1 \
  && echo 'PASS: teardown + deploy tu repo thanh cong'

BAD=0
for X in web:web api:api db:db db-migrate:migrate db-backup:backup node-agent:agent; do
  N="${X%%:*}"; C="${X##*:}"
  { [ "$(jp "serviceaccount/$N" '{.metadata.labels.app\.kubernetes\.io/part-of}')" = bookstore ] && \
    [ "$(jp "serviceaccount/$N" '{.metadata.labels.app\.kubernetes\.io/component}')" = "$C" ] && \
    [ "$(jp "serviceaccount/$N" '{.automountServiceAccountToken}')" = false ]; } \
    || { echo "FAIL: ServiceAccount $N"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: sau ServiceAccount dung ten, label, tat tu mount'
[ "$(jp serviceaccount/default '{.automountServiceAccountToken}')" = false ] \
  && echo 'PASS: default cua bookstore khong tu mount token'

T='{.spec.template.spec.serviceAccountName}'
{ [ "$(jp deployment/web "$T")" = web ] && [ "$(jp deployment/api "$T")" = api ] && \
  [ "$(jp statefulset/db "$T")" = db ] && [ "$(jp daemonset/node-agent "$T")" = node-agent ] && \
  [ "$(jp job/db-migrate "$T")" = db-migrate ] && \
  [ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.serviceAccountName}')" = db-backup ]; } \
  && echo 'PASS: moi workload chay duoi ServiceAccount cua no'

kubectl -n "$NS" get pods -l app.kubernetes.io/part-of=bookstore \
  -o jsonpath='{range .items[*]}{.metadata.name}{"|"}{.spec.serviceAccountName}{"|"}{range .spec.volumes[*]}{.name}{","}{end}{"|"}{range .spec.containers[*]}{range .volumeMounts[*]}{.mountPath}{","}{end}{end}{range .spec.initContainers[*]}{range .volumeMounts[*]}{.mountPath}{","}{end}{end}{"\n"}{end}' \
  > "$EV/b2-pod-sa-sau.txt"
grep -qE 'kube-api-access-|/var/run/secrets/kubernetes.io/serviceaccount' "$EV/b2-pod-sa-sau.txt" \
  && echo 'FAIL: con Pod mang token tu dong' || echo 'PASS: khong Pod nao cua du an mang token tu dong'
kubectl -n "$NS" exec deploy/web -c status-writer -- test -e /var/run/secrets/kubernetes.io/serviceaccount/token \
  && echo 'FAIL: web van co file token' || echo 'PASS: trong Pod web khong con file token'
./tests/smoke.sh > "$EV/b2-smoke.txt" 2>&1 && echo 'PASS: smoke cua lab-8c van PASS'

P="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=api -o jsonpath='{.items[0].metadata.name}')"
kubectl -n "$NS" patch pod "$P" --dry-run=server -p '{"spec":{"serviceAccountName":"db-backup"}}' \
  > "$EV/b2-doi-sa.txt" 2>&1 && echo 'FAIL: doi duoc ServiceAccount cua Pod dang chay' \
  || echo 'PASS: API server tu choi doi ServiceAccount cua Pod dang chay'

kubectl apply -f "$W/sa-ep-token.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/sa-ep-token --timeout=120s
kubectl -n "$NS" exec sa-ep-token -- test -s /var/run/secrets/kubernetes.io/serviceaccount/token \
  && echo 'PASS: tu mount o cap Pod thang thiet lap cua ServiceAccount'
kubectl -n "$NS" delete pod sa-ep-token --wait=true
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/rbac/serviceaccounts.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default                      # có sẵn trong mọi namespace; apply chỉ sửa nó
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
automountServiceAccountToken: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
    app.kubernetes.io/name: nginx
automountServiceAccountToken: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: api
    app.kubernetes.io/name: postgrest
automountServiceAccountToken: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: db
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: db
    app.kubernetes.io/name: postgresql
automountServiceAccountToken: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: db-migrate
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: migrate
    app.kubernetes.io/name: postgresql
automountServiceAccountToken: false
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: db-backup
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: backup
    app.kubernetes.io/name: postgresql
automountServiceAccountToken: false     # token chỉ chiếu tay vào đúng một container ở B3
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: node-agent
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: agent
    app.kubernetes.io/name: busybox
automountServiceAccountToken: false
```

Thêm `rbac/serviceaccounts.yaml` vào mục `resources` của `k8s/app/kustomization.yaml`. Trong spec Pod
của từng workload, cùng cấp với `containers`, thêm đúng một dòng:

```yaml
      serviceAccountName: web          # api, db, db-migrate, db-backup, node-agent tương ứng
```

```yaml
# ~/lab-work/9a/sa-ep-token.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sa-ep-token
  namespace: bookstore
spec:
  serviceAccountName: web
  automountServiceAccountToken: true    # cấp Pod thắng cấp ServiceAccount
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 50m, memory: 32Mi}
```

```bash
cd ~/bookstore && ./scripts/deploy.sh
```

Ý chính cho ADR:

1. Bài 279: spec của Pod thắng. Thứ ngăn thật sự là **RBAC trên quyền tạo workload**: ai tạo hay sửa
   được workload trong `bookstore` thì chọn được ServiceAccount và cách mount — bài 120, *Tạo
   workload*. B4 đo đúng rủi ro này với `ci-deployer`.
2. Danh tính riêng là chỗ để cấp quyền **sau này** mà không lan sang thành phần khác (B3 chỉ chạm
   `db-backup`); request và log của API server gọi đúng tên thành phần; bài 118 liệt kê việc nhóm Pod
   theo danh tính cho phần mềm bảo mật. Xóa hay đổi một ServiceAccount không ảnh hưởng thành phần khác.
3. Bài 279: `serviceAccountName` chỉ đặt được lúc tạo Pod hoặc trong template. Đổi template là
   revision mới, controller thay từng Pod — không có cách "cập nhật tại chỗ".
4. Bài 120, *Giảm thiểu việc phân phối các token đặc quyền*: DaemonSet có mặt ở **mọi** node, nên một
   container escape ở bất kỳ node nào cũng chạm tới token đó. `node-agent` không gọi API — không có
   token là lời giải gọn nhất.

</details>

## B3. [BS-9A-03] `db-backup` ghi `backup-status` qua Kubernetes API

**Loại:** Tính năng, kèm điều tra.

**Bối cảnh:** người vận hành cần biết lần backup thành công gần nhất mà không phải lục log Job — Job
bị dọn theo giới hạn lịch sử. Dự án lưu kết quả vào ConfigMap `backup-status`
([mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label)). `db-backup` thành thành phần **duy nhất** gọi API,
và là chỗ đầu tiên đặt nguyên tắc quyền tối thiểu lên một workload thật.

**Yêu cầu:**

1. ConfigMap `backup-status` nằm trong repo, label `part-of` và `component: backup`. Repo **không** khai
   giá trị trạng thái; triển khai lại không xóa trạng thái mà Job đã ghi.
2. Mỗi lần backup thành công, ConfigMap có bốn key: `last-run` (thời điểm UTC dạng
   `2026-10-13T08:00:00Z`), `file` (đường dẫn bản dump), `bytes` (kích thước), `pod` (tên Pod đã chạy).
   Dòng log `db-backup: OK <file> <bytes> byte` chỉ in khi **cả** dump lẫn cập nhật trạng thái thành
   công. Dump hỏng thì trạng thái giữ nguyên. Hợp đồng ghi file của Lab 6a giữ nguyên dù tách
   container: ghi tạm → kiểm → đổi tên, giữ tối đa 10 bản `bookstore-*.sql`, dọn file tạm bỏ dở.
3. ServiceAccount `db-backup` chỉ được `get` và `patch` đúng ConfigMap tên `backup-status`: không sửa
   ConfigMap khác, không `list`, không `update`, không Secret.
4. Token API chỉ có mặt trong **đúng một** container — container gọi API — sống tối đa 10 phút, và
   container đó xác minh server bằng CA của cluster (không tắt kiểm certificate).
5. Công cụ gọi API lấy từ một image đã khóa ở A1.3/A1.4. Ghi vào ADR vì sao chọn image đó.
6. Điều tra trước khi viết NetworkPolicy: trên CNI của cluster, rule egress theo ClusterIP của Service
   `kubernetes` (cổng 443) so với rule theo endpoint thật của API server — cái nào mở được đường. Rồi
   mở egress cho Pod backup tới API server, hẹp nhất có thể.
7. Các Pod khác của dự án vẫn không tới được API server.

**Ràng buộc:** không ClusterRole, ClusterRoleBinding; không `hostNetwork`; không bật tự mount; không
thêm image ngoài A1.3/A1.4; không token dài hạn dạng Secret. Pod và NetworkPolicy thử nghiệm ở
`~/lab-work/9a/`, không mang label dự án, xóa khi xong. Không ghi token vào bằng chứng.

**Đọc lại:**

- Bài [338](../338-access-api-from-pod-vi.md):
  [*Truy cập trực tiếp REST API*](../338-access-api-from-pod-vi.md#truy-cập-trực-tiếp-rest-api-directly-accessing-the-rest-api),
  [*Không sử dụng proxy*](../338-access-api-from-pod-vi.md#không-sử-dụng-proxy-without-using-a-proxy).
- Bài [279](../279-configure-service-account-vi.md):
  [*Chiếu token của ServiceAccount vào volume*](../279-configure-service-account-vi.md#serviceaccount-token-volume-projection),
  [*Khởi chạy Pod dùng chiếu service account token*](../279-configure-service-account-vi.md#launch-a-pod-using-service-account-token-projection).
- Bài [118](../118-service-accounts-vi.md): [*Cấp quyền cho một ServiceAccount*](../118-service-accounts-vi.md#grant-permissions),
  [*Xác thực credentials của service account*](../118-service-accounts-vi.md#authenticating-credentials).
- Bài [120](../120-rbac-good-practices-vi.md#least-privilege) — đặc quyền tối thiểu; tài liệu RBAC
  chính thức mà bài 120 bảo đọc kèm, mục
  [Referring to resources](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-resources)
  — `resourceNames` và giới hạn của nó.
- Bài [84](../84-network-policies-vi.md#behavior-of-ipblock-selectors) — `ipBlock` và việc ghi đè IP
  đích; [*NetworkPolicy và các pod `hostNetwork`*](../84-network-policies-vi.md#networkpolicy-và-các-pod-hostnetwork-networkpolicy-and-hostnetwork-pods).
- Bài [50](../50-init-containers-vi.md) — init container chạy xong mới tới container chính (Lab 3a).

**Bàn giao:** `k8s/app/backup/status-configmap.yaml`; `k8s/app/rbac/backup-status-writer.yaml`; manifest
CronJob `db-backup`; `k8s/app/network/backup-to-apiserver.yaml`; `k8s/app/kustomization.yaml`;
`~/lab-work/9a/np-thu.yaml`; `docs/adr/NNNN-backup-status-qua-api.md`; `~/lab-evidence/9a/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` rule theo endpoint thật của API server mở được đường cho Pod thử nghiệm; kết quả của rule
  theo ClusterIP được ghi lại.
- `PASS:` `backup-status` mang label dự án; `deploy.sh` lần hai không xóa trạng thái đã ghi.
- `PASS:` `db-backup` `get`/`patch` được `backup-status`; không làm được tám việc khác trong danh sách.
- `PASS:` Job chạy tay từ CronJob `Complete`; `file`, `bytes`, `pod` trong ConfigMap khớp đúng lần chạy;
  `last-run` mới hơn thời điểm bắt đầu.
- `PASS:` trong Pod backup, đúng một container mount token; token xin thời hạn 600 giây; nguồn có CA
  của cluster; CronJob không chứa cờ tắt kiểm certificate.
- `PASS:` Pod `web` vẫn không tới được API server; dự án không có object RBAC cấp cluster.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-backup-status-qua-api.md`:

1. Vì sao `resourceNames` dùng được với `get`/`patch` nhưng không giới hạn được `create`? Hệ quả: ai
   phải tạo `backup-status`, và vì sao nó **không** được đi qua `configMapGenerator`?
2. Vì sao token nằm trong một volume chiếu tay, mount vào đúng một container, thay vì bật tự mount cho
   cả Pod? Container nào trong Pod của bạn không cần token, và nó đang cầm credential gì khác?
3. Ba image đã khóa đều có shell trừ `postgrest`. Vì sao `busybox` và `postgres` không làm được việc
   gọi API này, và cái giá của việc dùng image `web` làm công cụ là gì?
4. Rule theo ClusterIP có hiệu lực trên cluster của bạn không? Dẫn câu của bài 84 về `ipBlock`. Rule
   bạn chọn gãy khi nào (đổi IP control plane, chuyển sang topology HA của Lab 8b)?
5. Token của Pod backup bị lộ: kẻ cầm nó làm được gì, trong bao lâu? So với token tự mount của mọi Pod
   trước sprint.

<details>
<summary>Gợi ý</summary>

1. Kiểm công cụ có trong image bằng Pod tạm, ví dụ:

   ```bash
   kubectl -n default run kiem-anh --rm -i --restart=Never --image=<image> \
     --command -- sh -c 'command -v curl wget'
   ```

   Với `busybox`, đọc `wget --help`: chỉ có `GET` và `POST` (`--post-data`); không có `PATCH`.
2. Gọi API từ trong Pod cần ba thứ của bài 338: địa chỉ (`kubernetes.default.svc`), CA, token. Một
   projected volume ghép được cả ba: `serviceAccountToken` (`expirationSeconds`), `configMap`
   `kube-root-ca.crt` (key `ca.crt`) và `downwardAPI` (`metadata.namespace`). Xem mẫu thật trên một
   Pod có tự mount:
   `kubectl -n kube-system get pod -l k8s-app=kube-dns -o yaml | grep -A25 'kube-api-access'`.
3. "Dump xong mới báo": dump trong init container, báo trạng thái trong container chính; hai container
   trao kết quả qua một `emptyDir`.
4. Sửa ConfigMap bằng HTTP `PATCH` với header `Content-Type: application/merge-patch+json`, thân
   `{"data":{"key":"value"}}`. Thử mà không đổi gì: thêm `?dryRun=All` vào URL.
5. Role: `rules[].resourceNames: ["backup-status"]`. ConfigMap không khai `data` thì `kubectl apply` lần
   sau không đụng vào các key do Job ghi.
6. Endpoint thật của API server:
   `kubectl -n default get endpointslice -l kubernetes.io/service-name=kubernetes -o wide`.
7. `kubectl logs job/<tên>` đọc container chính đầu tiên; log init container cần `-c <tên>`.

</details>

<details>
<summary>Script nghiệm thu — phần điều tra NetworkPolicy, chạy trước khi viết policy thật</summary>

Tự viết `~/lab-work/9a/np-thu.yaml`: Pod `busybox` tên `np-thu` trong `bookstore`, label
`lab-9a: np-thu` (không mang label dự án), `sleep 3600`, ghim worker2, có `requests`/`limits`. Script
tự sinh hai NetworkPolicy thử nghiệm chỉ chọn Pod đó.

```bash
NS=bookstore; EV=~/lab-evidence/9a; W=~/lab-work/9a
SVC_IP="$(kubectl -n default get service kubernetes -o jsonpath='{.spec.clusterIP}')"
SVC_PORT="$(kubectl -n default get service kubernetes -o jsonpath='{.spec.ports[?(@.name=="https")].port}')"
EP_IP="$(kubectl -n default get endpointslice -l kubernetes.io/service-name=kubernetes -o jsonpath='{.items[0].endpoints[0].addresses[0]}')"
EP_PORT="$(kubectl -n default get endpointslice -l kubernetes.io/service-name=kubernetes -o jsonpath='{.items[0].ports[0].port}')"
echo "Service kubernetes $SVC_IP:$SVC_PORT -> endpoint $EP_IP:$EP_PORT" | tee "$EV/b3-dieu-tra-np.txt"

np() {   # $1 = tên, $2 = CIDR, $3 = cổng
cat > "$W/$1.yaml" <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: $1
  namespace: bookstore
spec:
  podSelector:
    matchLabels:
      lab-9a: np-thu
  policyTypes: ["Egress"]
  egress:
  - to:
    - ipBlock:
        cidr: $2
    ports:
    - protocol: TCP
      port: $3
EOF
kubectl apply -f "$W/$1.yaml"
}
thu() { kubectl -n "$NS" exec np-thu -- timeout 8 wget -qO- --no-check-certificate \
          "https://${SVC_IP}:${SVC_PORT}/version" >/dev/null 2>&1; }

kubectl apply -f "$W/np-thu.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/np-thu --timeout=120s
thu && echo 'FAIL: chua co rule nao ma da toi duoc API server' || echo 'PASS: default deny chan Pod thu nghiem'

np thu-clusterip "${SVC_IP}/32" "$SVC_PORT"
R1=khong; for i in 1 2 3; do sleep 3; thu && { R1=co; break; }; done
echo "rule ClusterIP ${SVC_IP}:${SVC_PORT} -> toi duoc API server: $R1" | tee -a "$EV/b3-dieu-tra-np.txt"
kubectl -n "$NS" delete networkpolicy thu-clusterip

np thu-endpoint "${EP_IP}/32" "$EP_PORT"
R2=khong; for i in $(seq 1 10); do sleep 3; thu && { R2=co; break; }; done
echo "rule endpoint ${EP_IP}:${EP_PORT} -> toi duoc API server: $R2" | tee -a "$EV/b3-dieu-tra-np.txt"
[ "$R2" = co ] && echo 'PASS: rule theo endpoint that cua API server mo duoc duong'
echo "KET QUA can giai thich trong ADR: rule ClusterIP = $R1, rule endpoint = $R2"

kubectl -n "$NS" delete networkpolicy thu-endpoint
kubectl -n "$NS" delete pod np-thu --wait=true
```

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
NS=bookstore; EV=~/lab-evidence/9a
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
./scripts/deploy.sh > "$EV/b3-deploy.txt" 2>&1 && echo 'PASS: deploy.sh thanh cong'

L='{.metadata.labels.app\.kubernetes\.io/part-of}/{.metadata.labels.app\.kubernetes\.io/component}'
[ "$(jp configmap/backup-status "$L")" = bookstore/backup ] && echo 'PASS: backup-status mang label du an'

B=system:serviceaccount:bookstore:db-backup
can() { kubectl auth can-i "$@" -n "$NS" --as="$B"; }
{ [ "$(can get configmaps/backup-status)" = yes ] && [ "$(can patch configmaps/backup-status)" = yes ]; } \
  && echo 'PASS: db-backup get va patch duoc backup-status'
BAD=0
for Q in "update configmaps/backup-status" "get configmaps/khac" "patch configmaps/khac" "list configmaps" \
         "create configmaps" "get secrets" "list secrets" "create pods"; do
  [ "$(can $Q)" = no ] || { echo "FAIL: db-backup lai duoc phep: $Q"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: db-backup khong lam duoc tam viec khac'

T0="$(date +%s)"; J="b3-backup-$T0"
kubectl -n "$NS" create job "$J" --from=cronjob/db-backup
kubectl -n "$NS" wait --for=condition=Complete "job/$J" --timeout=300s && echo 'PASS: Job backup Complete'
kubectl -n "$NS" logs "job/$J" > "$EV/b3-backup-log.txt" 2>&1
LINE="$(grep '^db-backup: OK' "$EV/b3-backup-log.txt" | tail -1)"
F="$(echo "$LINE" | awk '{print $3}')"; BY="$(echo "$LINE" | awk '{print $4}')"
POD="$(kubectl -n "$NS" get pods -l batch.kubernetes.io/job-name="$J" -o jsonpath='{.items[0].metadata.name}')"
kubectl -n "$NS" get configmap backup-status -o jsonpath='{.data}' > "$EV/b3-backup-status.json"; cat "$EV/b3-backup-status.json"; echo
{ [ -n "$F" ] && [ "$(jp configmap/backup-status '{.data.file}')" = "$F" ] && \
  [ "$(jp configmap/backup-status '{.data.bytes}')" = "$BY" ] && \
  [ "$(jp configmap/backup-status '{.data.pod}')" = "$POD" ]; } && echo 'PASS: backup-status khop dung lan chay vua roi'
LR="$(jp configmap/backup-status '{.data.last-run}')"
[ "$(date -d "$LR" +%s 2>/dev/null || echo 0)" -ge "$((T0 - 5))" ] && echo "PASS: last-run=$LR moi hon thoi diem bat dau"

VT="$(kubectl -n "$NS" get pod "$POD" -o jsonpath='{range .spec.volumes[*]}{.name}{"="}{.projected.sources[*].serviceAccountToken.expirationSeconds}{"\n"}{end}' | grep -v '=$')"
VOLT="${VT%%=*}"; EXP="${VT##*=}"
N_MOUNT="$( { kubectl -n "$NS" get pod "$POD" -o jsonpath='{range .spec.initContainers[*]}{range .volumeMounts[*]}{.name}{"\n"}{end}{end}'
             kubectl -n "$NS" get pod "$POD" -o jsonpath='{range .spec.containers[*]}{range .volumeMounts[*]}{.name}{"\n"}{end}{end}'; } \
           | grep -cx "$VOLT")"
echo "volume token=$VOLT expirationSeconds=$EXP so container mount=$N_MOUNT" | tee "$EV/b3-token-volume.txt"
{ [ -n "$VOLT" ] && [ "$N_MOUNT" -eq 1 ] && [ "${EXP:-99999}" -le 600 ]; } \
  && echo 'PASS: dung mot container mount token, token xin toi da 600 giay'
kubectl -n "$NS" get pod "$POD" -o jsonpath='{.spec.volumes[*].projected.sources[*].configMap.name}' \
  | grep -qw kube-root-ca.crt && echo 'PASS: container goi API co CA cua cluster'
kubectl -n "$NS" get cronjob db-backup -o yaml | grep -Eq -- '--insecure|--no-check-certificate|curl -k|curl -sk' \
  && echo 'FAIL: CronJob tat kiem certificate' || echo 'PASS: khong co co tat kiem certificate'
kubectl -n "$NS" delete job "$J" --wait=true

./scripts/deploy.sh > "$EV/b3-deploy-2.txt" 2>&1
[ "$(jp configmap/backup-status '{.data.file}')" = "$F" ] && echo 'PASS: trien khai lai khong xoa trang thai da ghi'

kubectl -n "$NS" exec deploy/web -c status-writer -- timeout 10 wget -qO- --no-check-certificate \
  https://kubernetes.default.svc/version >/dev/null 2>&1 \
  && echo 'FAIL: web toi duoc API server' || echo 'PASS: web van khong toi duoc API server'
[ "$(sl kubectl get clusterrole,clusterrolebinding -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ] \
  && echo 'PASS: du an khong co object RBAC cap cluster'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/backup/status-configmap.yaml — cố ý không có "data": Job db-backup ghi, repo không ghi
apiVersion: v1
kind: ConfigMap
metadata:
  name: backup-status
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: backup
    app.kubernetes.io/name: postgresql
```

```yaml
# k8s/app/rbac/backup-status-writer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backup-status-writer
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: backup
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["backup-status"]   # không giới hạn được create — nên ConfigMap phải có sẵn
  verbs: ["get", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backup-status-writer
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: backup
subjects:
- kind: ServiceAccount
  name: db-backup
  namespace: bookstore
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: backup-status-writer
```

```yaml
# k8s/app/network/backup-to-apiserver.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backup-to-apiserver
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: backup
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: backup
  policyTypes: ["Egress"]
  egress:
  - to:
    - ipBlock:
        cidr: 192.168.100.221/32     # endpoint của Service kubernetes — ĐỌC từ EndpointSlice, đừng chép số này
    ports:
    - protocol: TCP
      port: 6443
```

Spec Pod trong `jobTemplate` của CronJob `db-backup`. Phần `env` của `pg-dump`, `volumes` PVC
`backup-store` và `priorityClassName` là của bạn từ các sprint trước — giữ nguyên; phần mang chú thích
`9a` là mới. Khối `securityContext` thuộc B7, để sẵn ở đây cho trọn manifest:

```yaml
        spec:
          serviceAccountName: db-backup                 # 9a — B2
          priorityClassName: bookstore-batch            # 7a, giữ nguyên
          restartPolicy: Never
          securityContext:                              # 9a — B7
            runAsNonRoot: true
            runAsUser: 999
            runAsGroup: 999
          volumes:
          - name: backup
            persistentVolumeClaim:
              claimName: backup-store                   # 6a, giữ nguyên
          - name: run                                   # 9a — kết quả dump chuyển cho container báo
            emptyDir: {}
          - name: api-credentials                       # 9a — token chỉ cho container gọi API
            projected:
              sources:
              - serviceAccountToken:
                  path: token
                  expirationSeconds: 600
              - configMap:
                  name: kube-root-ca.crt
                  items:
                  - {key: ca.crt, path: ca.crt}
              - downwardAPI:
                  items:
                  - path: namespace
                    fieldRef: {fieldPath: metadata.namespace}
          initContainers:
          - name: pg-dump                               # 9a — container backup cũ chuyển thành init
            image: postgres:17.11
            env:
            # --- PGHOST, PGUSER, PGDATABASE, PGCONNECT_TIMEOUT, PGPASSWORD: giữ nguyên của bạn ---
            command:
            - sh
            - -c
            - |
              set -e
              echo "pg-dump: $(id)"
              grep NoNewPrivs /proc/self/status | sed 's/^/pg-dump: /'
              find /backup -maxdepth 1 -name 'bookstore-*.sql.tmp' -mmin +60 -delete   # 6a
              F="/backup/bookstore-$(date +%Y%m%dT%H%M%S).sql"
              pg_dump --no-owner --file="$F.tmp"                                   # 6a: ghi tạm
              grep -q 'CREATE TABLE api.books' "$F.tmp"
              mv "$F.tmp" "$F"                                       # 6a: chỉ đổi tên sau khi kiểm
              ls -1t /backup/bookstore-*.sql | tail -n +11 | xargs -r rm -f        # 6a: giữ 10 bản
              B="$(wc -c < "$F")"
              printf 'FILE=%s\nBYTES=%s\n' "$F" "$B" > /run/backup/result
              echo "pg-dump: xong $F $B byte, chu so huu $(stat -c '%u:%g' "$F")"
            securityContext:
              allowPrivilegeEscalation: false           # 9a — B7
            resources:
              requests: {cpu: 50m, memory: 64Mi}
              limits: {cpu: 200m, memory: 128Mi}
            volumeMounts:
            - {name: backup, mountPath: /backup}
            - {name: run, mountPath: /run/backup}
          containers:
          - name: report-status                         # 9a — container duy nhất cầm token
            image: nginxinc/nginx-unprivileged:1.30.5-alpine   # chỉ để dùng curl có sẵn trong image
            env:
            - name: POD_NAME
              valueFrom: {fieldRef: {fieldPath: metadata.name}}
            command:
            - sh
            - -c
            - |
              set -eu
              echo "report-status: $(id)"
              grep NoNewPrivs /proc/self/status | sed 's/^/report-status: /'
              SA=/var/run/secrets/bookstore-api
              F="$(sed -n 's/^FILE=//p' /run/backup/result)"
              B="$(sed -n 's/^BYTES=//p' /run/backup/result)"
              NOW="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
              BODY="$(printf '{"data":{"last-run":"%s","file":"%s","bytes":"%s","pod":"%s"}}' \
                "$NOW" "$F" "$B" "$POD_NAME")"
              CODE="$(curl -sS --max-time 20 -o /run/backup/api-response.json -w '%{http_code}' \
                --cacert "$SA/ca.crt" \
                -H "Authorization: Bearer $(cat "$SA/token")" \
                -H 'Content-Type: application/merge-patch+json' \
                -X PATCH --data-binary "$BODY" \
                "https://kubernetes.default.svc/api/v1/namespaces/$(cat "$SA/namespace")/configmaps/backup-status")" \
                || CODE=000
              if [ "$CODE" != 200 ]; then
                echo "db-backup: LOI cap nhat backup-status, HTTP $CODE"
                cat /run/backup/api-response.json 2>/dev/null; echo
                exit 1
              fi
              echo "db-backup: OK $F $B byte"
            securityContext:
              runAsUser: 101                            # 9a — B7: ghi đè cấp Pod, user của image này
              allowPrivilegeEscalation: false
            resources:
              requests: {cpu: 10m, memory: 16Mi}
              limits: {cpu: 100m, memory: 32Mi}
            volumeMounts:
            - {name: run, mountPath: /run/backup}
            - {name: api-credentials, mountPath: /var/run/secrets/bookstore-api, readOnly: true}
```

```yaml
# ~/lab-work/9a/np-thu.yaml
apiVersion: v1
kind: Pod
metadata:
  name: np-thu
  namespace: bookstore
  labels:
    lab-9a: np-thu
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 50m, memory: 32Mi}
```

Thêm vào `resources` của `k8s/app/kustomization.yaml`: `backup/status-configmap.yaml`,
`rbac/backup-status-writer.yaml`, `network/backup-to-apiserver.yaml`. Rồi `./scripts/deploy.sh`.

Ý chính cho ADR:

1. Tài liệu RBAC: không giới hạn được `create` (và `deletecollection`) theo tên, vì lúc phân quyền
   tên object mới có thể chưa biết. Vậy repo phải tạo `backup-status` trước; Job chỉ sửa. Đi qua
   `configMapGenerator` thì tên mang hậu tố hash và đổi theo nội dung — `resourceNames` không còn khớp.
2. `pg-dump` chỉ cần mật khẩu DB (đã có qua Secret), không cần API. Tự mount gắn token vào **mọi**
   container; chiếu tay chỉ gắn vào `report-status`. Container nào bị chiếm cũng chỉ mất đúng
   credential nó vốn cần.
3. `wget` của busybox chỉ có `GET`/`POST`, và đường TLS dựng sẵn của nó không xác minh certificate
   (nó tự in cảnh báo này khi gặp `https`). Image `postgres` không có HTTP client. Image `web` có
   `curl` và CA — dùng được, nhưng bạn đang phụ thuộc vào một công cụ phụ của image ứng dụng: đổi tag
   `web` là phải kiểm lại `curl` của Job backup.
4. Trên cluster lab, kết quả thường là: rule ClusterIP **không** mở được đường, rule endpoint thì mở.
   Bài 84: việc ghi đè IP đích xảy ra trước hay sau khi xử lý NetworkPolicy là **không được định
   nghĩa**, tùy CNI và cách hiện thực Service — vì vậy phải đo, và ghi đúng số đo của bạn. Rule theo
   IP control plane gãy khi IP đổi hoặc khi API server đứng sau load balancer (Lab 8b): khi đó endpoint
   là nhiều IP, policy phải liệt kê đủ.
5. Kẻ cầm token: `get`/`patch` đúng một ConfigMap, tối đa vài phút (TTL xin 600 giây, và token chết
   khi Pod bị xóa — bài 118). Trước sprint: token của `default` tồn tại ở mọi Pod, sống theo Pod, quyền
   discovery — ít quyền nhưng **có mặt khắp nơi**.

</details>

## B4. [BS-9A-04] Danh tính cho CI: `ci-deployer`

**Loại:** Tính năng.

**Bối cảnh:** đội sắp có pipeline CI tự cập nhật image của `web`, `api` và ConfigMap cấu hình. CI chạy
**ngoài** cluster, nên cần một danh tính phi con người, một kubeconfig riêng, token có hạn, và một quy
trình thu hồi khi token lộ. Không ai được copy kubeconfig quản trị cho CI.

**Yêu cầu:**

1. ServiceAccount `ci-deployer` trong `bookstore`, không tự mount. Quyền: cập nhật Deployment và theo
   dõi rollout; đọc, tạo, sửa ConfigMap — trong `bookstore`. **Không**: đọc Secret, `exec`, gắn debug
   container, sửa Namespace `bookstore`, tạo Pod, xóa Deployment, đụng object RBAC, xin token cho
   ServiceAccount khác, đọc Node. Manifest ở `k8s/app/rbac/ci-deployer.yaml`.
2. `scripts/ci-kubeconfig.sh` — chép nguyên văn từ khối bên dưới — sinh
   `secrets/ci-deployer.kubeconfig`: token có thời hạn, CA của cluster, namespace mặc định `bookstore`;
   file quyền `600`, không vào git.
3. Chỉ dùng kubeconfig đó: chứng minh nó là `ci-deployer`, cập nhật được Deployment (server dry-run),
   theo dõi được rollout, bị `Forbidden` khi đọc Secret.
4. Chứng minh `kubectl proxy` chạy bằng kubeconfig CI mang **đúng** quyền CI; và gọi REST trực tiếp
   bằng token cần CA của cluster — không dùng `--insecure`.
5. Diễn tập thu hồi: token CI coi như đã lộ — vô hiệu nó **ngay**, không chờ hết hạn; phát hành lại.
   Ghi runbook.
6. Điều tra bằng server dry-run: quyền sửa Deployment có đủ để làm một Pod đọc Secret, hoặc chạy dưới
   ServiceAccount khác không? Ghi thành nợ `TD-15` nếu có.
7. ⭐ Một hệ thống ngoài muốn tự xác minh token CI: chỉ ra issuer cluster công bố và đối chiếu với
   claim `iss` của token; liệt kê các loại proxy của bài 359 có thật trên cluster.

**Ràng buộc:** không token dài hạn dạng Secret; không ClusterRole; kubeconfig chỉ nằm trong
`secrets/`. Mọi thử "sửa" dùng `--dry-run=server` trừ phần thu hồi. Không in token ra màn hình hay
bằng chứng.

**Đọc lại:**

- Bài [279](../279-configure-service-account-vi.md#manually-create-an-api-token-for-a-serviceaccount) —
  `kubectl create token`, `--duration`; mục
  [*Tạo thủ công một API token dài hạn*](../279-configure-service-account-vi.md#tạo-thủ-công-một-api-token-dài-hạn-cho-serviceaccount-manually-create-a-long-lived-api-token-for-a-serviceaccount)
  — lý do **không** dùng; ⭐ [*Khám phá issuer*](../279-configure-service-account-vi.md#khám-phá-issuer-của-service-account-service-account-issuer-discovery).
- Bài [118](../118-service-accounts-vi.md#get-a-token) — ghi chú về token dài hạn cho ứng dụng ngoài
  cluster; [*Xác thực credentials*](../118-service-accounts-vi.md#authenticating-credentials).
- Bài [359](../359-access-cluster-vi.md#directly-accessing-the-rest-api);
  [*Rất nhiều loại proxy*](../359-access-cluster-vi.md#rất-nhiều-loại-proxy-so-many-proxies).
- Bài [190](../190-access-cluster-api-vi.md):
  [*Sử dụng kubectl proxy*](../190-access-cluster-api-vi.md#sử-dụng-kubectl-proxy-using-kubectl-proxy),
  [*Không dùng kubectl proxy*](../190-access-cluster-api-vi.md#không-dùng-kubectl-proxy-without-kubectl-proxy).
- Bài [120](../120-rbac-good-practices-vi.md): [*Tạo workload*](../120-rbac-good-practices-vi.md#workload-creation),
  [*Sửa đổi Namespace*](../120-rbac-good-practices-vi.md#namespace-modification),
  [*Yêu cầu token*](../120-rbac-good-practices-vi.md#yêu-cầu-token-token-request),
  [*Rà soát định kỳ*](../120-rbac-good-practices-vi.md#rà-soát-định-kỳ-periodic-review).
- Bài [286](../286-migrate-from-psp-vi.md#review-namespace-permissions) — mục 1, ai sửa được namespace.
- Bài [111](../111-kubeconfig-vi.md#context) — cluster, user, context trong kubeconfig (Lab 1b).

**Bàn giao:** `k8s/app/rbac/ci-deployer.yaml`; `scripts/ci-kubeconfig.sh`; `docs/runbook-ci-deployer.md`;
`docs/adr/NNNN-danh-tinh-ci.md`; dòng `TD-15` trong `docs/tech-debt.md`; `~/lab-evidence/9a/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` `ci-deployer` tồn tại, không tự mount.
- `PASS:` kubeconfig CI quyền `600`, nằm ngoài git; xác thực đúng `ci-deployer`; token có `exp` chưa tới.
- `PASS:` CI cập nhật được Deployment (dry-run), theo dõi được rollout, đọc được ConfigMap; đọc Secret
  bị `Forbidden`.
- `PASS:` CI không có bất kỳ quyền nào trong danh sách cấm.
- `PASS:` qua `kubectl proxy` của CI: Deployment `200`, Secret `403`.
- `PASS:` REST trực tiếp: có CA thì `200`, không CA thì `curl` từ chối.
- `PASS:` xóa ServiceAccount là token cũ `401`; tạo lại cùng tên không làm token cũ sống lại; kubeconfig
  phát hành lại dùng được ngay.
- `PASS:` (`TD-15`) dry-run đổi ServiceAccount của `web` và gắn Secret vào Pod `web` đều được chấp nhận;
  cluster không đổi; `docs/tech-debt.md` có `TD-15`.
- `PASS:` (⭐) `iss` của token khớp issuer công bố; apiserver proxy tới kubelet có thật.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-danh-tinh-ci.md`:

1. Vì sao token qua TokenRequest (`kubectl create token`) thay vì Secret
   `kubernetes.io/service-account-token`? So về claim, vòng đời, cách thu hồi.
2. Sau khi bạn xóa rồi tạo lại `ci-deployer`, kubeconfig mới dùng được **ngay** với đúng quyền cũ.
   Vì sao? Mục *Rà soát định kỳ* của bài 120 cảnh báo gì về chính tính chất này?
3. `kubectl proxy` xác thực hộ hay phân quyền hộ? Một dashboard dùng chung chạy `kubectl proxy` bằng
   kubeconfig quản trị thì ai có quyền gì?
4. `TD-15`: vì sao quyền sửa Deployment gần như bằng quyền đọc Secret và mượn ServiceAccount? Trong
   phạm vi kiến thức hiện tại, bạn giảm rủi ro bằng gì, và cái gì phải chờ công cụ ở giai đoạn sau?
5. Vì sao CI tuyệt đối không được `patch` Namespace `bookstore`? Hai thứ gì đọc label namespace trên
   cluster này hoặc sắp đọc (bài 120 *Sửa đổi Namespace*, bài 286 mục 1)?

`scripts/ci-kubeconfig.sh`, chép nguyên văn:

```bash
cat > ~/bookstore/scripts/ci-kubeconfig.sh <<'EOF'
#!/usr/bin/env bash
# Phát hành kubeconfig cho danh tính CI ci-deployer — Lab 9a.
# Token có thời hạn (TokenRequest qua kubectl create token); file nằm trong secrets/, không vào git.
# Dùng: ./scripts/ci-kubeconfig.sh [thời hạn, mặc định 1h]
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore
SA=ci-deployer
DURATION="${1:-1h}"
OUT=secrets/ci-deployer.kubeconfig

kubectl -n "$NS" get serviceaccount "$SA" >/dev/null
SERVER="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
TMP="$(mktemp -d)"; trap 'rm -rf "$TMP"' EXIT
kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  | base64 -d > "$TMP/ca.crt"
TOKEN="$(kubectl -n "$NS" create token "$SA" --duration="$DURATION")"

mkdir -p secrets
rm -f "$OUT"; ( umask 077; touch "$OUT" )
kubectl --kubeconfig="$OUT" config set-cluster bookstore-lab --server="$SERVER" \
  --certificate-authority="$TMP/ca.crt" --embed-certs=true >/dev/null
kubectl --kubeconfig="$OUT" config set-credentials "$SA" --token="$TOKEN" >/dev/null
kubectl --kubeconfig="$OUT" config set-context ci --cluster=bookstore-lab --user="$SA" --namespace="$NS" >/dev/null
kubectl --kubeconfig="$OUT" config use-context ci >/dev/null
unset TOKEN
echo "ci-kubeconfig: da ghi $OUT (thoi han yeu cau $DURATION)"
kubectl --kubeconfig="$OUT" auth whoami -o jsonpath='{.status.userInfo.username}{"\n"}'
EOF
chmod +x ~/bookstore/scripts/ci-kubeconfig.sh
```

Lệnh `config set-credentials --token` đặt token vào tham số dòng lệnh trong chốc lát; chấp nhận được
trên master một người dùng — trên máy nhiều người dùng thì phải ghi file theo cách khác.

<details>
<summary>Gợi ý</summary>

- Role cần hai nhóm API: `apps` (deployments, replicasets) và nhóm lõi `""` (configmaps).
  `kubectl set image` đọc rồi `patch`; `kubectl rollout status` đọc và `watch`; `rollout undo` cần đọc
  ReplicaSet.
- Kiểm quyền thay CI mà không cần token: `kubectl auth can-i <verb> <resource> -n bookstore
  --as=system:serviceaccount:bookstore:ci-deployer`. Subresource dùng `--subresource=exec`,
  `--subresource=ephemeralcontainers`, `--subresource=token`.
- Đọc token từ kubeconfig vào biến để giải mã claim — `kubectl config view --raw` với
  `-o jsonpath='{.users[0].user.token}'` và `--kubeconfig` trỏ tới file của CI. Phần giữa hai dấu
  chấm là payload base64url của JWT.
- Thu hồi: token do TokenRequest phát hành gắn với **UID** của ServiceAccount (bài 118, bước kiểm
  thứ ba). Repo vẫn khai ServiceAccount, nên `./scripts/deploy.sh` tạo lại nó với UID mới.
- Dry-run đổi ServiceAccount của `web`: `kubectl patch deployment web --dry-run=server --type=json -p
  '[{"op":"replace","path":"/spec/template/spec/serviceAccountName","value":"db-backup"}]'`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script xóa và tạo lại ServiceAccount CI</summary>

```bash
NS=bookstore; EV=~/lab-evidence/9a; W=~/lab-work/9a
cd ~/bookstore
K=secrets/ci-deployer.kubeconfig
CI=system:serviceaccount:bookstore:ci-deployer
jwt_pay() {
  P="$(printf '%s' "$1" | cut -d. -f2)"
  case $(( ${#P} % 4 )) in 2) P="${P}==" ;; 3) P="${P}=" ;; esac
  printf '%s' "$P" | tr -- '-_' '+/' | base64 -d 2>/dev/null
}
jwt_str() { jwt_pay "$1" | sed -n "s/.*\"$2\":\"\([^\"]*\)\".*/\1/p"; }
jwt_num() { jwt_pay "$1" | sed -n "s/.*\"$2\":\([0-9]*\).*/\1/p"; }

./scripts/deploy.sh > "$EV/b4-deploy.txt" 2>&1 && echo 'PASS: deploy.sh thanh cong'
[ "$(kubectl -n "$NS" get serviceaccount ci-deployer -o jsonpath='{.automountServiceAccountToken}')" = false ] \
  && echo 'PASS: ci-deployer ton tai, khong tu mount token'

./scripts/ci-kubeconfig.sh 30m > "$EV/b4-ci-kubeconfig.txt"
{ [ "$(stat -c '%a' "$K")" = 600 ] && git check-ignore -q "$K" && \
  [ -z "$(git ls-files secrets/ | grep -v '\.gitkeep$')" ]; } && echo 'PASS: kubeconfig CI quyen 600, nam ngoai git'
[ "$(kubectl --kubeconfig="$K" auth whoami -o jsonpath='{.status.userInfo.username}')" = "$CI" ] \
  && echo 'PASS: kubeconfig nay xac thuc dung ci-deployer'
TOK="$(kubectl --kubeconfig="$K" config view --raw -o jsonpath='{.users[0].user.token}')"
EXP="$(jwt_num "$TOK" exp)"; IAT="$(jwt_num "$TOK" iat)"
echo "sub=$(jwt_str "$TOK" sub) | iss=$(jwt_str "$TOK" iss) | exp-iat=$(( ${EXP:-0} - ${IAT:-0} )) giay" \
  | tee "$EV/b4-token-claim.txt"
{ [ -n "$EXP" ] && [ "$EXP" -gt "$(date +%s)" ]; } && echo 'PASS: token co claim exp va chua het han'

kubectl --kubeconfig="$K" -n "$NS" set image deployment/web nginx=nginxinc/nginx-unprivileged:1.30.5-alpine \
  --dry-run=server > "$EV/b4-ci-set-image.txt" 2>&1 && echo 'PASS: CI cap nhat duoc Deployment (server dry-run)'
kubectl --kubeconfig="$K" -n "$NS" rollout status deployment/web --timeout=60s >/dev/null \
  && echo 'PASS: CI theo doi duoc rollout'
kubectl --kubeconfig="$K" -n "$NS" get configmaps >/dev/null && echo 'PASS: CI doc duoc ConfigMap'
kubectl --kubeconfig="$K" -n "$NS" get secret db-credentials > "$EV/b4-ci-secret.txt" 2>&1
grep -qi 'forbidden' "$EV/b4-ci-secret.txt" && echo 'PASS: CI bi Forbidden khi doc Secret'

BAD=0
for Q in "get secrets" "list secrets" "create pods --subresource=exec" \
         "patch pods --subresource=ephemeralcontainers" "patch namespaces/bookstore" "create pods" \
         "delete deployments" "create rolebindings" "create serviceaccounts --subresource=token" \
         "list nodes" "get nodes --subresource=proxy"; do
  [ "$(kubectl auth can-i $Q -n "$NS" --as="$CI")" = no ] || { echo "FAIL: ci-deployer lai duoc: $Q"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: ci-deployer khong co quyen nao trong danh sach cam'

ss -ltn | grep -q ':8001 ' && echo 'FAIL: cong 8001 dang ban — dong tien trinh cu truoc'
kubectl --kubeconfig="$K" proxy --port=8001 > "$EV/b4-proxy.log" 2>&1 & PX=$!
for i in $(seq 1 30); do curl -s -o /dev/null http://127.0.0.1:8001/api/ && break; sleep 1; done
C1="$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8001/apis/apps/v1/namespaces/bookstore/deployments)"
C2="$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8001/api/v1/namespaces/bookstore/secrets)"
kill "$PX"; wait "$PX" 2>/dev/null
echo "qua kubectl proxy cua CI: deployments=$C1 secrets=$C2" | tee "$EV/b4-proxy.txt"
[ "$C1" = 200 ] && [ "$C2" = 403 ] && echo 'PASS: kubectl proxy mang dung quyen cua kubeconfig no dung'

SRV="$(kubectl --kubeconfig="$K" config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
kubectl --kubeconfig="$K" config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  | base64 -d > "$W/ca.crt"
code() { curl -s -o /dev/null -w '%{http_code}' --cacert "$W/ca.crt" -H "Authorization: Bearer $1" "$SRV$2"; }
C3="$(code "$TOK" /apis/apps/v1/namespaces/bookstore/deployments/web)"
curl -s -o /dev/null -H "Authorization: Bearer $TOK" "$SRV/api"; RC=$?
echo "REST co CA: $C3 | khong CA: curl exit=$RC" | tee "$EV/b4-rest.txt"
[ "$C3" = 200 ] && [ "$RC" -ne 0 ] && echo 'PASS: goi REST truc tiep can CA cua cluster de tin server'

# Thu hồi khẩn cấp
OLD="$TOK"
kubectl -n "$NS" delete serviceaccount ci-deployer
C4=''; for i in $(seq 1 60); do C4="$(code "$OLD" /api)"; [ "$C4" = 401 ] && break; sleep 1; done
echo "token cu sau khi xoa ServiceAccount: $C4" | tee "$EV/b4-thu-hoi.txt"
[ "$C4" = 401 ] && echo 'PASS: xoa ServiceAccount la token CI het hieu luc ngay'
./scripts/deploy.sh > "$EV/b4-deploy-2.txt" 2>&1
C5="$(code "$OLD" /api)"; unset OLD
echo "token cu sau khi tao lai ServiceAccount cung ten: $C5" | tee -a "$EV/b4-thu-hoi.txt"
[ "$C5" = 401 ] && echo 'PASS: tao lai ServiceAccount cung ten khong lam token cu song lai'
./scripts/ci-kubeconfig.sh 30m >/dev/null
TOK="$(kubectl --kubeconfig="$K" config view --raw -o jsonpath='{.users[0].user.token}')"
kubectl --kubeconfig="$K" -n "$NS" get deployments >/dev/null \
  && echo 'PASS: kubeconfig phat hanh lai dung duoc ngay, voi dung quyen cu'

# TD-15
kubectl --kubeconfig="$K" -n "$NS" patch deployment web --dry-run=server --type=json \
  -p '[{"op":"replace","path":"/spec/template/spec/serviceAccountName","value":"db-backup"}]' \
  > "$EV/b4-td11-muon-sa.txt" 2>&1 && echo 'PASS (TD-15): CI dat duoc web chay duoi ServiceAccount db-backup'
kubectl --kubeconfig="$K" -n "$NS" patch deployment web --dry-run=server -p \
  '{"spec":{"template":{"spec":{"initContainers":[{"name":"status-writer","env":[{"name":"LO_MAT_KHAU","valueFrom":{"secretKeyRef":{"name":"db-credentials","key":"postgres-password"}}}]}]}}}}' \
  > "$EV/b4-td11-doc-secret.txt" 2>&1 && echo 'PASS (TD-15): CI gan duoc Secret vao Pod du khong doc duoc Secret qua API'
[ "$(kubectl -n "$NS" get deployment web -o jsonpath='{.spec.template.spec.serviceAccountName}')" = web ] \
  && echo 'PASS: dry-run khong doi gi tren cluster'
grep -q 'TD-15' docs/tech-debt.md && echo 'PASS: tech-debt.md co TD-15'

# ⭐ Issuer và các loại proxy
ISS="$(kubectl get --raw /.well-known/openid-configuration | sed -n 's/.*"issuer":"\([^"]*\)".*/\1/p')"
[ -n "$ISS" ] && [ "$ISS" = "$(jwt_str "$TOK" iss)" ] && echo "PASS: iss cua token CI khop issuer cong bo ($ISS)"
[ "$(kubectl get --raw /api/v1/nodes/lab-k8s-worker2/proxy/healthz)" = ok ] \
  && echo 'PASS: apiserver proxy toi kubelet co that — va CI khong co nodes/proxy'
unset TOK
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/rbac/ci-deployer.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
automountServiceAccountToken: false     # không Pod nào chạy dưới danh tính này
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "patch", "update"]
- apiGroups: ["apps"]
  resources: ["replicasets"]
  verbs: ["get", "list", "watch"]          # rollout history/undo
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "patch", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
subjects:
- kind: ServiceAccount
  name: ci-deployer
  namespace: bookstore
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ci-deployer
```

Thêm `rbac/ci-deployer.yaml` vào `resources` của `k8s/app/kustomization.yaml`.

```markdown
<!-- docs/runbook-ci-deployer.md -->
# Runbook — credential của CI (ci-deployer)

- Phát hành: `./scripts/ci-kubeconfig.sh 1h` → `secrets/ci-deployer.kubeconfig` (600, ngoài git).
  Chuyển file cho CI qua kênh bí mật của CI; không commit, không gửi qua chat.
- Thời hạn: xin đúng bằng thời gian một lần chạy pipeline; hết hạn thì phát hành lại.
- Nghi lộ token: `kubectl -n bookstore delete serviceaccount ci-deployer` (token cũ 401 ngay),
  `./scripts/deploy.sh` (tạo lại từ repo), `./scripts/ci-kubeconfig.sh` (phát hành mới).
- Dấu hiệu sai: `401` = token hết hạn hoặc đã thu hồi; `403 ... ci-deployer cannot ...` = pipeline
  làm việc ngoài phạm vi Role — sửa pipeline, không nới Role khi chưa có ADR.
```

Dòng thêm vào `docs/tech-debt.md`:

```markdown
| TD-15 | ci-deployer sửa được pod template của Deployment nên gián tiếp đọc được Secret và mượn được mọi ServiceAccount trong bookstore | 9a | chấp nhận có điều kiện: token CI ngắn hạn, thu hồi bằng xóa ServiceAccount; xem lại khi học admission policy (giai đoạn 14) | mở | ~/lab-evidence/9a/b4-td11-*.txt |
```

Ý chính cho ADR:

1. Token TokenRequest có `exp`, gắn với UID của ServiceAccount, chết ngay khi ServiceAccount bị xóa.
   Token dạng Secret **không có** `exp`, lộ là dùng được tới khi có người nhớ xóa; bài 118 và 279
   khuyên tránh, đặc biệt cho ứng dụng ngoài cluster.
2. RoleBinding lưu **tên** chủ thể. ServiceAccount mới cùng tên có ngay quyền cũ — tiện cho thu hồi,
   nhưng bài 120 cảnh báo: ai tạo lại được một danh tính trùng tên với danh tính đã xóa thì thừa hưởng
   quyền của nó. Rà binding mồ côi định kỳ.
3. Proxy **xác thực hộ** (gắn credential của kubeconfig nó cầm), không phân quyền hộ — ba chặng vẫn chạy
   ở API server. Dashboard dùng chung với kubeconfig quản trị = trao quyền quản trị cho mọi ai chạm tới
   cổng proxy.
4. Bài 120, *Tạo workload*: ai sửa được pod template thì gắn được Secret vào Pod, chạy Pod dưới bất kỳ
   ServiceAccount nào trong namespace. Hiện giảm bằng: token ngắn, thu hồi nhanh, kubeconfig không ra
   khỏi `secrets/`, CI chỉ ở một namespace. Chặn hẳn cần chính sách kiểm nội dung object ở chặng
   tiếp nhận (ValidatingAdmissionPolicy, webhook — giai đoạn 14, bài 173) — ghi `TD-15`.
5. Label namespace là đầu vào của chính sách: NetworkPolicy của sprint 7 chọn namespace `traefik`
   theo label, và Pod Security ở Lab 9b cũng đọc label namespace. Ai `patch` được Namespace là nới
   được chính sách (bài 120, bài 286 mục 1).

</details>

## B5. [BS-9A-05] Đội phát triển chỉ đọc, người trực được gỡ lỗi

**Loại:** Tính năng.

**Bối cảnh:** đội phát triển cần xem trạng thái hệ thống mà không đụng được gì. Người trực sự cố cần
gắn debug container vào `api` — image không có shell (Lab 3a B8) — nhưng Lab 3a đã ghi `TD-07`: ai gắn
được debug container là đọc được biến môi trường của `api`, kể cả mật khẩu DB. Sprint này biến quyền
gỡ lỗi thành quyền **riêng của một nhóm** thay vì đi kèm quyền đọc.

**Yêu cầu:**

1. Nhóm `bookstore-devs`: đọc Pod, log, Deployment, Service, ConfigMap, sự kiện trong `bookstore` — dùng
   ClusterRole có sẵn của cluster, không viết lại. Không đọc Secret, không `exec`, không gắn debug
   container, không ghi gì, không thấy namespace khác hay Node.
2. Nhóm `bookstore-oncall`: gắn được ephemeral container và `exec` vào Pod của `bookstore`, đọc Pod và
   log. Không đọc Secret qua API, không sửa workload, không tạo hay xóa Pod.
3. Chứng minh bằng mạo danh và một SubjectAccessReview; chứng minh quyết định dựa vào **group** chứ
   không vào tên user; cluster không có object `User` nào.
4. Thử thật: thành viên `bookstore-oncall` gắn debugger vào một Pod `api` và thấy tiến trình
   `postgrest`; thành viên `bookstore-devs` bị từ chối. Thay Pod đã gắn debugger bằng Pod mới.
5. Rà soát: không binding nào của dự án trỏ tới `cluster-admin`, `admin`, `edit`; không Role nào có
   wildcard, Secret, `escalate`, `bind`, `impersonate`, `nodes/proxy`, `serviceaccounts/token`; không
   ClusterRoleBinding nào nhắc tới chủ thể của dự án.
6. Cập nhật `TD-07` trong `docs/tech-debt.md`.

**Ràng buộc:** không tạo user, không sửa certificate; nhóm chỉ là tên trong binding. Không
ClusterRoleBinding. Debugger chỉ gắn vào Pod `api`, và Pod đó bị thay khi xong.

**Đọc lại:**

- Bài [120](../120-rbac-good-practices-vi.md): [*Đặc quyền tối thiểu*](../120-rbac-good-practices-vi.md#least-privilege),
  [*Liệt kê secret*](../120-rbac-good-practices-vi.md#liệt-kê-secret-listing-secrets),
  [*Truy cập subresource `proxy` của Node*](../120-rbac-good-practices-vi.md#truy-cập-subresource-proxy-của-node-access-to-proxy-subresource-of-nodes),
  [*Verb escalate*](../120-rbac-good-practices-vi.md#verb-escalate-escalate-verb),
  [*Verb bind*](../120-rbac-good-practices-vi.md#verb-bind-bind-verb),
  [*Verb impersonate*](../120-rbac-good-practices-vi.md#verb-impersonate-impersonate-verb),
  [*Rà soát định kỳ*](../120-rbac-good-practices-vi.md#rà-soát-định-kỳ-periodic-review).
- Bài [119](../119-controlling-access-vi.md#authorization) — SubjectAccessReview; bài
  [118](../118-service-accounts-vi.md#what-are-service-accounts) — bảng ServiceAccount so với user.
- Bài [52](../52-ephemeral-containers-vi.md#ephemeral-container-là-gì-what-is-an-ephemeral-container) —
  handler `ephemeralcontainers`; [Lab 3a](LAB-3A-POD-VA-VONG-DOI.md) ticket B8.
- Tài liệu RBAC chính thức, bảng
  [User-facing roles](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/#user-facing-roles)
  — `view` không gồm Secret và vì sao.

**Bàn giao:** `k8s/app/rbac/nguoi-dung.yaml`; `k8s/app/kustomization.yaml`;
`docs/adr/NNNN-quyen-nguoi-dung.md`; `docs/tech-debt.md`; `~/lab-evidence/9a/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` `bookstore-devs` đọc được sáu loại object, không làm được bảy việc cấm, không thấy
  `kube-system` hay Node.
- `PASS:` `bookstore-oncall` gắn debug container, `exec`, đọc Pod và log được; không đọc Secret, không sửa
  workload.
- `PASS:` cùng tên user thiếu group thì không được; user lạ có group thì được; không có resource `users`.
- `PASS:` SubjectAccessReview và `can-i` cho cùng câu trả lời.
- `PASS:` devs gắn debugger bị `Forbidden`; oncall gắn được và thấy `postgrest`; Pod đó đã được thay.
- `PASS:` ba kiểm tra rà soát; `TD-07` ghi giới hạn ở 9a.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-quyen-nguoi-dung.md`:

1. Vì sao dùng ClusterRole `view` qua RoleBinding thay vì viết Role riêng? Phạm vi hiệu lực của nó
   do object nào quyết định?
2. Vì sao `view` không có Secret, còn đội phát triển lại thường xin `list secrets` "cho tiện"?
3. `TD-07` được **giới hạn** chứ chưa **đóng**: người trực vẫn đọc được gì của `api`? Muốn đóng hẳn
   cần đổi gì ở cách `api` nhận mật khẩu?
4. Kubernetes không có object `User`. Tên `bookstore-devs` đến từ đâu khi một người thật đăng nhập
   bằng client certificate (B1)? Một tên nhóm gõ sai trong binding gây ra chuyện gì — lỗi, hay im lặng?

<details>
<summary>Gợi ý</summary>

- Chủ thể nhóm trong binding: `kind: Group`, `apiGroup: rbac.authorization.k8s.io`.
- Ephemeral container là subresource `pods/ephemeralcontainers`; `kubectl debug` gửi `patch` lên đó.
  `exec` và `attach` là subresource `pods/exec`, `pods/attach`. Phiên bản baseline kiểm thêm quyền
  `create` cho kết nối websocket của chúng — cấp cả `get` lẫn `create`.
- `--as-group` bắt buộc đi cùng `--as`. `kubectl auth can-i --list -n bookstore --as=x --as-group=y`.
- SubjectAccessReview là object chỉ tạo, không lưu: `kubectl create -o jsonpath='{.status.allowed}' -f -`
  với `spec.user`, `spec.groups`, `spec.resourceAttributes`.
- Gắn debugger mạo danh người trực: `kubectl debug ... --as=<user> --as-group=bookstore-oncall`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/9a
cd ~/bookstore && ./scripts/deploy.sh > "$EV/b5-deploy.txt" 2>&1
DEV='--as=dev-an --as-group=bookstore-devs'
ONC='--as=truc-binh --as-group=bookstore-oncall'
ci() { kubectl auth can-i "$@" -n "$NS"; }
{ echo "== dev-an (bookstore-devs)"; kubectl auth can-i --list -n "$NS" $DEV
  echo "== truc-binh (bookstore-oncall)"; kubectl auth can-i --list -n "$NS" $ONC; } > "$EV/b5-can-i-list.txt"

BAD=0
for Q in "list pods" "get pods --subresource=log" "list deployments" "list services" "list configmaps" "list events"; do
  [ "$(ci $Q $DEV)" = yes ] || { echo "FAIL: devs khong duoc: $Q"; BAD=1; }
done
for Q in "get secrets" "list secrets" "create pods --subresource=exec" "patch pods --subresource=ephemeralcontainers" \
         "delete pods" "patch deployments" "list roles"; do
  [ "$(ci $Q $DEV)" = no ] || { echo "FAIL: devs lai duoc: $Q"; BAD=1; }
done
[ "$(kubectl auth can-i list pods -n kube-system $DEV)" = no ] || { echo 'FAIL: devs thay kube-system'; BAD=1; }
[ "$(kubectl auth can-i list nodes $DEV)" = no ] || { echo 'FAIL: devs thay Node'; BAD=1; }
[ "$BAD" -eq 0 ] && echo 'PASS: bookstore-devs chi doc trong bookstore, khong Secret, exec hay debug'

BAD=0
for Q in "patch pods --subresource=ephemeralcontainers" "create pods --subresource=exec" "get pods" \
         "get pods --subresource=log"; do
  [ "$(ci $Q $ONC)" = yes ] || { echo "FAIL: oncall khong duoc: $Q"; BAD=1; }
done
for Q in "get secrets" "list secrets" "patch deployments" "delete pods" "create pods"; do
  [ "$(ci $Q $ONC)" = no ] || { echo "FAIL: oncall lai duoc: $Q"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: bookstore-oncall chi them quyen go loi'

{ [ "$(ci list pods --as=dev-an)" = no ] && [ "$(ci list pods --as=nguoi-la --as-group=bookstore-devs)" = yes ]; } \
  && echo 'PASS: quyet dinh dua vao group, khong dua vao ten nguoi dung'
kubectl get users >/dev/null 2>&1 && echo 'FAIL: co resource users' || echo 'PASS: cluster khong co object User'

SAR="$(kubectl create -o jsonpath='{.status.allowed}' -f - <<'YAML'
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  user: dev-an
  groups: ["bookstore-devs"]
  resourceAttributes:
    namespace: bookstore
    verb: get
    resource: secrets
YAML
)"
[ "$SAR" = false ] && [ "$(ci get secrets $DEV)" = no ] && echo 'PASS: SubjectAccessReview va can-i cung tra loi khong'

API="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=api -o jsonpath='{.items[0].metadata.name}')"
kubectl -n "$NS" debug "pod/$API" --image=busybox:1.37 --target=postgrest -c dev-dbg $DEV -- sleep 300 \
  > "$EV/b5-debug-dev.txt" 2>&1 && echo 'FAIL: devs gan duoc debugger' || echo 'PASS: devs bi tu choi gan debugger'
grep -qi 'forbidden' "$EV/b5-debug-dev.txt" && echo 'PASS: devs chet o chang phan quyen (Forbidden)'
kubectl -n "$NS" debug "pod/$API" --image=busybox:1.37 --target=postgrest -c truc-dbg $ONC -- sleep 300 \
  > "$EV/b5-debug-oncall.txt" 2>&1 && echo 'PASS: oncall gan duoc debugger vao Pod api'
for i in $(seq 1 30); do
  [ -n "$(kubectl -n "$NS" get pod "$API" -o jsonpath='{.status.ephemeralContainerStatuses[?(@.name=="truc-dbg")].state.running.startedAt}')" ] && break
  sleep 2
done
kubectl -n "$NS" exec "$API" -c truc-dbg $ONC -- ps > "$EV/b5-debug-ps.txt" 2>&1
grep -q postgrest "$EV/b5-debug-ps.txt" && echo 'PASS: oncall thay tien trinh postgrest qua debugger'
kubectl -n "$NS" delete pod "$API" --wait=true && kubectl -n "$NS" rollout status deployment/api --timeout=180s >/dev/null \
  && echo 'PASS: Pod api mang debugger da duoc thay bang Pod moi'

CRB="$(kubectl get clusterrolebinding -o jsonpath='{range .items[*]}{.metadata.name}{"|"}{range .subjects[*]}{.kind}{":"}{.namespace}{":"}{.name}{" "}{end}{"\n"}{end}' \
  | grep -E 'ServiceAccount:bookstore:|Group::bookstore-' || true)"
[ -z "$CRB" ] && echo 'PASS: khong ClusterRoleBinding nao nhac toi chu the cua du an'
kubectl -n "$NS" get rolebinding -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.roleRef.kind}/{.roleRef.name}{"\n"}{end}' \
  > "$EV/b5-rolebinding.txt"
grep -Eq 'ClusterRole/(cluster-admin|admin|edit)$' "$EV/b5-rolebinding.txt" \
  && echo 'FAIL: co binding tro toi role qua rong' || echo 'PASS: khong binding nao tro toi cluster-admin, admin, edit'
kubectl -n "$NS" get role -o jsonpath='{range .items[*]}{.metadata.name}{": "}{range .rules[*]}{.resources}{" "}{.verbs}{" "}{end}{"\n"}{end}' \
  > "$EV/b5-role-rules.txt"
grep -Eq '"\*"|secrets|escalate|"bind"|impersonate|nodes/proxy|serviceaccounts/token' "$EV/b5-role-rules.txt" \
  && echo 'FAIL: co Role chua quyen nguy hiem' || echo 'PASS: khong Role nao co wildcard, Secret hay verb leo thang'
grep 'TD-07' docs/tech-debt.md | grep -q '9a' && echo 'PASS: TD-07 ghi gioi han o 9a'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/rbac/nguoi-dung.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bookstore-devs-view
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
subjects:
- kind: Group
  name: bookstore-devs
  apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view                     # có sẵn; RoleBinding thu phạm vi về đúng bookstore
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: debug-pods
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods/ephemeralcontainers"]
  verbs: ["patch"]
- apiGroups: [""]
  resources: ["pods/exec", "pods/attach"]
  verbs: ["get", "create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bookstore-oncall-debug
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
subjects:
- kind: Group
  name: bookstore-oncall
  apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: debug-pods
```

Thêm `rbac/nguoi-dung.yaml` vào `resources` của `k8s/app/kustomization.yaml`. Dòng `TD-07` trong
`docs/tech-debt.md`:

```markdown
| TD-07 | Ai exec hay gắn debug container được vào Pod api là đọc được mật khẩu trong biến môi trường | 3a | 3b (giảm), 9a (giới hạn) | đã giới hạn ở 9a: chỉ nhóm bookstore-oncall có quyền exec và ephemeral container; người trực vẫn đọc được biến môi trường của api | docs/adr/NNNN-quyen-nguoi-dung.md |
```

Ý chính cho ADR:

1. `view` được dự án Kubernetes duy trì, tự theo các loại object mới; viết lại là tự gánh việc đó.
   ClusterRole chỉ nói **quyền gì**; **RoleBinding** trong `bookstore` quyết định **ở đâu** — nên
   devs không thấy `kube-system` hay Node.
2. Bài 120: `list`/`watch` trả về cả nội dung Secret, không chỉ tên. Tài liệu RBAC nói rõ `view`
   không có Secret vì đọc Secret là đọc được credential của ServiceAccount — một dạng leo thang.
3. Người trực vẫn: thấy tiến trình, đọc `/proc/<pid>/environ` qua debugger nhắm vào `postgrest` (tùy
   profile của `kubectl debug`), gọi endpoint nội bộ. Đóng hẳn khi mật khẩu không còn nằm trong biến
   môi trường của `api` — khi đó debug không còn đồng nghĩa với đọc bí mật.
4. Tên nhóm đến từ chặng 1: với client certificate là trường O (B1). Binding chỉ lưu tên, nên gõ sai
   tên nhóm **không báo lỗi** — binding tồn tại hợp lệ và không ai được quyền. Chỉ `can-i` và rà soát
   mới phát hiện.

</details>

## B6. [BS-9A-06] Diễn tập: request chết ở chặng nào

**Loại:** Diễn tập sự cố — mọi object diễn tập không mang label dự án và bị xóa khi xong.

**Bối cảnh:** khi một thành phần báo "không gọi được API", người trực phải biết request chết ở
**chặng nào** trước khi sửa — sửa RBAC cho một lỗi `401` là sửa sai chỗ. Ticket này dựng lại từng kiểu
chết bằng chính các danh tính vừa tạo.

**Yêu cầu:** với mỗi chặng, dựng lại ít nhất một request chết ở đó và ghi dấu hiệu nhận biết từ phía
client vào postmortem:

| Chặng | Request phải dựng lại |
| --- | --- |
| 1 — Xác thực | token rác; token của `db-backup` đúng chữ ký nhưng xin cho audience khác; ⭐ không gửi credential tới `/api` và `/version` |
| 2 — Phân quyền | `ci-deployer` đọc Secret `db-credentials`; token `db-backup` sửa một ConfigMap **khác** (dry-run) và `list` ConfigMap |
| 3 — Tiếp nhận | một Job gõ sai tên ServiceAccount: Job có, Pod không; chứng minh chặng 2 cho phép chủ thể tạo Pod; đọc và ghi vào một namespace không tồn tại |
| 4 — Validate | ⭐ đổi `roleRef` của RoleBinding cấp quyền cho `db-backup` |
| Sau API server | tham chiếu B7 (`runAsNonRoot`) và B8 (sysctl): object được ghi, kubelet từ chối |

Postmortem `docs/postmortem/9a-request-bi-chan.md`: một bảng bốn cột — triệu chứng, chặng, xác
nhận bằng gì, sửa ở đâu.

**Ràng buộc:** không sửa cấu hình API server dù kết quả khác kỳ vọng — ghi lại. Mọi thao tác ghi lên
object thật dùng dry-run. Không ghi token vào bằng chứng; phản hồi JSON của API server thì được.

**Đọc lại:**

- Bài [119](../119-controlling-access-vi.md) — cả bài, đúng thứ tự bốn bước.
- Bài [118](../118-service-accounts-vi.md#authenticating-credentials) — năm việc API server kiểm trên
  bearer token.
- Bài [120](../120-rbac-good-practices-vi.md#gia-cố-hardening) — binding của `system:unauthenticated`.
- Bài [279](../279-configure-service-account-vi.md#use-multiple-service-accounts) — ServiceAccount phải
  tồn tại khi Pod được tạo.
- Tài liệu RBAC chính thức, mục
  [RoleBinding and ClusterRoleBinding](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding)
  — sau khi tạo binding thì không đổi được `roleRef`.

**Bàn giao:** `~/lab-work/9a/drill-sa-sai.yaml`; `docs/postmortem/9a-request-bi-chan.md`;
`~/lab-evidence/9a/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` token rác `401`; token sai audience `401` trong khi token đúng audience của cùng
  ServiceAccount `200`.
- `PASS:` `ci-deployer` đọc Secret: `Forbidden` kèm tên danh tính; token `db-backup`: sửa ConfigMap
  khác `403`, `list` `403`, `get` `backup-status` `200`.
- `PASS:` Job gõ sai ServiceAccount có sự kiện `FailedCreate` nói về ServiceAccount, không có Pod nào;
  `job-controller` được phép tạo Pod ở chặng 2.
- `PASS:` đọc namespace không tồn tại thành công, ghi thì bị từ chối dù `can-i` trả `yes`.
- `PASS:` (⭐) không gửi credential: `/api` và `/version` cho kết quả đúng với cấu hình `anonymous-auth`;
  đổi `roleRef` bị từ chối.
- `PASS:` object diễn tập đã xóa; postmortem tồn tại.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Ứng dụng báo `401` dù token vừa cấp và RoleBinding đầy đủ. Kể ba nguyên nhân bạn đã dựng lại được
   trong sprint này (B4, B6) và cách phân biệt.
2. `403` không đồng nghĩa "thiếu RBAC". Khi nào `403` đến từ chặng 3? Lệnh nào phân biệt hai trường hợp?
3. Với Job gõ sai ServiceAccount, `kubectl apply` thành công — vì sao? Ai nhận lỗi, và người trực thấy
   nó ở đâu?
4. ⭐ Request không credential nhận `403` chứ không `401` — điều đó nói gì về cấu hình cluster, và bài
   120 khuyên rà gì?

<details>
<summary>Gợi ý</summary>

- Token xin cho audience khác:
  `kubectl -n bookstore create token db-backup --audience=<tên> --duration=10m`.
- Sửa thử mà không đổi gì qua REST: thêm `?dryRun=All` vào URL `PATCH`.
- Job diễn tập: `busybox`, `command: ["true"]`, `restartPolicy: Never`, `backoffLimit: 0`, có
  `requests`/`limits`, `serviceAccountName` gõ sai một chữ. Sự kiện của Job:
  `kubectl get events --field-selector involvedObject.name=<tên Job>`.
- Controller tạo Pod của Job dưới danh tính `system:serviceaccount:kube-system:job-controller` (B1 đã
  thấy kube-controller-manager dùng ServiceAccount riêng cho từng controller).
- Cấu hình `anonymous-auth`: `sudo grep anonymous-auth /etc/kubernetes/manifests/kube-apiserver.yaml` —
  chỉ đọc.

</details>

<details>
<summary>Script nghiệm thu — script này chính là cuộc diễn tập; viết Job diễn tập trước</summary>

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
NS=bookstore; EV=~/lab-evidence/9a; W=~/lab-work/9a
cd ~/bookstore
SRV="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
  | base64 -d > "$W/ca.crt"
code() { curl -s -o "$W/resp.json" -w '%{http_code}' --cacert "$W/ca.crt" "$@"; }

# Chặng 1
C_RAC="$(code -H 'Authorization: Bearer day-khong-phai-token' "$SRV/api")"
T="$(kubectl -n "$NS" create token db-backup --audience=bookstore-khong-phai-api --duration=10m)"
C_AUD="$(code -H "Authorization: Bearer $T" "$SRV/api")"
TB="$(kubectl -n "$NS" create token db-backup --duration=10m)"
C_DUNG="$(code -H "Authorization: Bearer $TB" "$SRV/api")"
unset T
echo "token rac=$C_RAC | sai audience=$C_AUD | dung audience=$C_DUNG" | tee "$EV/b6-chang1.txt"
[ "$C_RAC" = 401 ] && echo 'PASS: chang 1 — token rac bi 401'
[ "$C_AUD" = 401 ] && [ "$C_DUNG" = 200 ] && echo 'PASS: chang 1 — cung ServiceAccount, sai audience la 401'

# ⭐ Không gửi credential
if sudo grep -q 'anonymous-auth=false' /etc/kubernetes/manifests/kube-apiserver.yaml; then EA=401; EV2=401; else EA=403; EV2=200; fi
C_AN_API="$(code "$SRV/api")"; C_AN_VER="$(code "$SRV/version")"
echo "khong credential: /api=$C_AN_API (ky vong $EA) | /version=$C_AN_VER (ky vong $EV2)" | tee -a "$EV/b6-chang1.txt"
[ "$C_AN_API" = "$EA" ] && [ "$C_AN_VER" = "$EV2" ] && echo 'PASS: (⭐) request khong credential dung voi cau hinh anonymous-auth'

# Chặng 2
K=secrets/ci-deployer.kubeconfig; [ -s "$K" ] || ./scripts/ci-kubeconfig.sh 30m >/dev/null
kubectl --kubeconfig="$K" -n "$NS" get secret db-credentials > "$EV/b6-chang2-ci.txt" 2>&1
grep -qi 'forbidden' "$EV/b6-chang2-ci.txt" && grep -q 'system:serviceaccount:bookstore:ci-deployer' "$EV/b6-chang2-ci.txt" \
  && echo 'PASS: chang 2 — ci-deployer doc Secret bi Forbidden, thong bao goi dung ten'
CM_KHAC="$(kubectl -n "$NS" get configmap -o name | grep -v -e backup-status -e kube-root-ca.crt | head -1 | cut -d/ -f2)"
C_P="$(code -X PATCH -H "Authorization: Bearer $TB" -H 'Content-Type: application/merge-patch+json' \
  --data-binary '{"data":{"thu":"1"}}' "$SRV/api/v1/namespaces/bookstore/configmaps/$CM_KHAC?dryRun=All")"
cp "$W/resp.json" "$EV/b6-chang2-backup-patch.json"
C_L="$(code -H "Authorization: Bearer $TB" "$SRV/api/v1/namespaces/bookstore/configmaps")"
C_G="$(code -H "Authorization: Bearer $TB" "$SRV/api/v1/namespaces/bookstore/configmaps/backup-status")"
unset TB
echo "token db-backup: patch $CM_KHAC=$C_P | list=$C_L | get backup-status=$C_G" | tee "$EV/b6-chang2-backup.txt"
[ "$C_P" = 403 ] && [ "$C_L" = 403 ] && [ "$C_G" = 200 ] \
  && echo 'PASS: chang 2 — dung ten thi 200, ten khac hoac list thi 403'

# Chặng 3
kubectl apply -f "$W/drill-sa-sai.yaml"
JN="$(kubectl create --dry-run=client -o jsonpath='{.metadata.name}' -f "$W/drill-sa-sai.yaml")"
for i in $(seq 1 30); do
  kubectl -n "$NS" get events --field-selector involvedObject.name="$JN" -o jsonpath='{.items[*].message}' \
    | grep -q 'error looking up service account' && break
  sleep 2
done
kubectl -n "$NS" get events --field-selector involvedObject.name="$JN" \
  -o jsonpath='{range .items[*]}{.reason}{": "}{.message}{"\n"}{end}' > "$EV/b6-chang3-events.txt"
cat "$EV/b6-chang3-events.txt"
{ grep -q 'FailedCreate' "$EV/b6-chang3-events.txt" && grep -q 'error looking up service account' "$EV/b6-chang3-events.txt"; } \
  && echo 'PASS: chang 3 — Pod bi tu choi vi ServiceAccount khong ton tai'
[ "$(sl kubectl -n "$NS" get pods -l batch.kubernetes.io/job-name="$JN" --no-headers)" -eq 0 ] \
  && echo 'PASS: Job ton tai nhung khong co Pod nao'
[ "$(kubectl auth can-i create pods -n "$NS" --as=system:serviceaccount:kube-system:job-controller)" = yes ] \
  && echo 'PASS: chang 2 cho phep job-controller tao Pod — thu giet request nam o chang 3'
kubectl -n "$NS" delete job "$JN" --wait=true

NS_MA=bookstore-khong-ton-tai
kubectl get configmaps -n "$NS_MA" > "$EV/b6-doc-ns-ma.txt" 2>&1; RC_DOC=$?
kubectl create configmap thu -n "$NS_MA" --from-literal=a=b > "$EV/b6-ghi-ns-ma.txt" 2>&1; RC_GHI=$?
{ [ "$RC_DOC" -eq 0 ] && [ "$RC_GHI" -ne 0 ] && grep -qi 'not found' "$EV/b6-ghi-ns-ma.txt" && \
  [ "$(kubectl auth can-i create configmaps -n "$NS_MA")" = yes ]; } \
  && echo 'PASS: doc lot qua, ghi bi chan o chang 3 du chang 2 tra loi yes'

# ⭐ Bước 4
RB="$(kubectl -n "$NS" get rolebinding -o jsonpath='{range .items[*]}{.metadata.name}{" "}{range .subjects[*]}{.name}{" "}{end}{"\n"}{end}' \
  | awk '/ db-backup /{print $1; exit}')"
kubectl -n "$NS" patch rolebinding "$RB" --dry-run=server --type=merge \
  -p '{"roleRef":{"apiGroup":"rbac.authorization.k8s.io","kind":"ClusterRole","name":"view"}}' > "$EV/b6-buoc4.txt" 2>&1 \
  && echo 'FAIL: doi duoc roleRef' || { grep -qi 'roleRef' "$EV/b6-buoc4.txt" && echo "PASS: (⭐) buoc 4 — roleRef cua $RB khong doi duoc"; }

test -f docs/postmortem/9a-request-bi-chan.md && echo 'PASS: co postmortem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/9a/drill-sa-sai.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: drill-sa-sai
  namespace: bookstore
spec:
  backoffLimit: 0
  template:
    spec:
      serviceAccountName: db-bakup          # gõ sai: thiếu chữ c
      restartPolicy: Never
      containers:
      - name: c
        image: busybox:1.37
        command: ["true"]
        resources:
          requests: {cpu: 10m, memory: 16Mi}
          limits: {cpu: 50m, memory: 32Mi}
```

Bảng cho postmortem:

| Triệu chứng | Chặng | Xác nhận bằng | Sửa ở đâu |
| --- | --- | --- | --- |
| `401 Unauthorized`, không nhắc tên ai | 1 | Giải mã `aud`, `exp` của token; ServiceAccount còn tồn tại với UID cũ không | Phát hành token mới, đúng audience |
| `403`, thông báo `User "..." cannot <verb> resource ...` | 2 | `kubectl auth can-i ... --as=<danh tính>` trả `no` | Role/RoleBinding trong repo |
| `Forbidden`/`not found` nói về **nội dung** object (ServiceAccount, namespace, hạn mức), `can-i` trả `yes` | 3 | `kubectl describe` object, sự kiện `FailedCreate` của controller | Manifest của object |
| `Invalid` / `cannot change ...` khi sửa object đã có | 4 | Đọc field bị nêu tên | Xóa rồi tạo lại object, hoặc giữ nguyên |
| Object được tạo, Pod `CreateContainerConfigError` hoặc `SysctlForbidden` | sau API server, ở kubelet | `kubectl get pod -o wide`, `describe` | `securityContext` của Pod (B7, B8) |

Ý chính:

1. (a) Token xin cho audience khác — `401` dù chữ ký đúng; (b) ServiceAccount đã bị xóa hoặc tạo lại
   (B4) — token gắn UID cũ; (c) token hết hạn hoặc không phải token. Phân biệt bằng claim `aud`, `exp`
   và UID của ServiceAccount hiện tại.
2. Chặng 3 trả `Forbidden` khi chính sách tiếp nhận từ chối — ví dụ ServiceAccount không tồn tại.
   `kubectl auth can-i` hỏi đúng chặng 2: `yes` mà request vẫn chết thì thủ phạm ở chặng 3.
3. Bạn **tạo Job** — hợp lệ ở cả ba chặng. Pod do `job-controller` tạo sau đó mới bị chặng 3 từ chối;
   lỗi đi vào sự kiện `FailedCreate` của Job, không về terminal của bạn.
4. `anonymous-auth` đang bật: request trắng qua chặng 1 dưới danh tính `system:anonymous` rồi chết ở
   chặng 2. Rà binding của nhóm `system:unauthenticated` (bài 120, *Gia cố*) — `/version` trả `200` vì
   có binding cho thông tin công khai.

</details>

## B7. [BS-9A-07] Danh tính tiến trình của tác vụ nền

**Loại:** Tính năng, kèm diễn tập trên `lab-k8s-worker2`.

**Bối cảnh:** ServiceAccount là danh tính **với API server**; trên node, mỗi tiến trình còn một danh
tính khác — uid và gid. `db-migrate` và `db-backup` chạy image `postgres`, image này không khai `USER`
nên tiến trình chạy bằng root dù chỉ là client `psql`/`pg_dump`. Container `report-status` của B3 cầm
token API. Cả hai không cần root.

**Yêu cầu:**

1. Mọi tiến trình trong Pod của `db-migrate` và `db-backup` chạy uid `999`, gid `999` — user `postgres`
   của image — và kubelet phải **kiểm** chúng không phải root. Riêng container gọi API của `db-backup`
   chạy bằng user mà image của nó khai (`101`), gid vẫn `999`: một thiết lập cấp Pod, một ghi đè cấp
   container.
2. Không tiến trình nào trong hai Pod đó leo thang đặc quyền được — chứng minh bằng cờ của kernel.
3. File dump mới trên PVC thuộc uid `999`.
4. Log của **mỗi** container in kết quả `id` và dòng `NoNewPrivs` của chính nó.
5. Diễn tập: Pod thử nghiệm dùng image `busybox` chỉ khai "phải non-root" mà không khai uid — chứng
   minh nó được lập lịch nhưng không khởi động, đọc lý do.
6. `db` **không** đổi ở sprint này (`TD-06`, Lab 9b).

**Ràng buộc:** không capability, seccomp, `readOnlyRootFilesystem` (Lab 9b). Pod diễn tập ghim
worker2, không label dự án, xóa khi xong.

**Đọc lại:**

- Bài [291](../291-security-context-vi.md):
  [*Đặt security context cho một Pod*](../291-security-context-vi.md#set-the-security-context-for-a-pod),
  [*Đặt security context cho một Container*](../291-security-context-vi.md#đặt-security-context-cho-một-container-set-the-security-context-for-a-container),
  [*Tư cách thành viên group ngầm định*](../291-security-context-vi.md#tư-cách-thành-viên-group-ngầm-định-được-định-nghĩa-trong-etcgroup-của-container-image-implicit-group-memberships-defined-in-etcgroup-in-the-container-image),
  [*Thảo luận*](../291-security-context-vi.md#thảo-luận-discussion).
- Bài [60](../60-advanced-pod-config-vi.md#security-context) — security context (Lab 3a B9).
- [Mục 4.4 của file dự án](DU-AN-BOOKSTORE.md#44-image) — user của từng image.

**Bàn giao:** manifest Job `db-migrate` và CronJob `db-backup`; `~/lab-work/9a/nonroot-thieu-uid.yaml`;
`docs/adr/NNNN-danh-tinh-tien-trinh.md`; `~/lab-evidence/9a/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` CronJob và Job khai non-root ở cấp Pod.
- `PASS:` một lần backup: có container chạy uid `999`, có container chạy uid `101`, không container nào
  uid `0`; mọi container gid `999`; mọi container `NoNewPrivs` bằng `1`.
- `PASS:` file dump mới thuộc `999:999`.
- `PASS:` log `db-migrate` cho thấy uid `999`.
- `PASS:` Pod diễn tập được lập lịch, kẹt `CreateContainerConfigError` với lý do nhắc `runAsNonRoot`;
  đã xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-danh-tinh-tien-trinh.md`:

1. Vì sao không cần `fsGroup` cho hai Pod này? Kiểm quyền của thư mục `emptyDir` và thư mục PVC bằng
   `ls -ld`. Khi nào thì cần, và bài 291 nói `fsGroup` tác động lên loại volume nào?
2. `id` trong `pg-dump` có group nào bạn không khai ở đâu cả không? Nó đến từ đâu, và vì sao bài 291
   coi đó là rủi ro khi truy cập volume?
3. Vì sao `db` chưa chạy non-root được bằng đúng cách này? (Entrypoint của image — mục 4.4.)
4. `runAsNonRoot` bị kiểm ở đâu? So với cách chặng tiếp nhận từ chối Job gõ sai ServiceAccount ở B6.

<details>
<summary>Gợi ý</summary>

- `securityContext` cấp Pod: `runAsNonRoot`, `runAsUser`, `runAsGroup`. Cấp container ghi đè khi trùng
  field; `allowPrivilegeEscalation` chỉ có ở cấp container.
- Dòng `NoNewPrivs` trong `/proc/self/status` là cờ kernel mà `allowPrivilegeEscalation: false` bật.
- In chủ sở hữu file: `stat -c '%u:%g' <file>` (image `postgres` có `stat` của coreutils).
- Đọc log init container: `kubectl logs <pod> -c <tên init>`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/9a; W=~/lab-work/9a
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
./scripts/deploy.sh > "$EV/b7-deploy.txt" 2>&1 && echo 'PASS: deploy.sh thanh cong'

S='{.spec.jobTemplate.spec.template.spec.securityContext.runAsNonRoot}'
{ [ "$(jp cronjob/db-backup "$S")" = true ] && \
  [ "$(jp job/db-migrate '{.spec.template.spec.securityContext.runAsNonRoot}')" = true ]; } \
  && echo 'PASS: CronJob va Job khai non-root o cap Pod'

J="b7-backup-$(date +%s)"
kubectl -n "$NS" create job "$J" --from=cronjob/db-backup
kubectl -n "$NS" wait --for=condition=Complete "job/$J" --timeout=300s >/dev/null && echo 'PASS: Job backup Complete'
POD="$(kubectl -n "$NS" get pods -l batch.kubernetes.io/job-name="$J" -o jsonpath='{.items[0].metadata.name}')"
CS="$(kubectl -n "$NS" get pod "$POD" -o jsonpath='{.spec.initContainers[*].name} {.spec.containers[*].name}')"
: > "$EV/b7-backup-log.txt"
for C in $CS; do kubectl -n "$NS" logs "$POD" -c "$C" | sed "s/^/[$C] /" >> "$EV/b7-backup-log.txt"; done
cat "$EV/b7-backup-log.txt"
NC="$(echo $CS | wc -w)"
{ grep -q 'uid=999' "$EV/b7-backup-log.txt" && grep -q 'uid=101' "$EV/b7-backup-log.txt" && \
  ! grep -q 'uid=0(' "$EV/b7-backup-log.txt"; } && echo 'PASS: co container uid 999, co container uid 101, khong ai la root'
[ "$(grep -c 'gid=999' "$EV/b7-backup-log.txt")" -eq "$NC" ] && echo "PASS: ca $NC container gid 999"
[ "$(grep -cE 'NoNewPrivs:[[:space:]]*1' "$EV/b7-backup-log.txt")" -eq "$NC" ] \
  && echo "PASS: ca $NC container NoNewPrivs = 1"
grep -q 'chu so huu 999:999' "$EV/b7-backup-log.txt" && echo 'PASS: file dump moi thuoc 999:999'
kubectl -n "$NS" delete job "$J" --wait=true

kubectl -n "$NS" logs job/db-migrate > "$EV/b7-migrate-log.txt" 2>&1
grep -q 'uid=999' "$EV/b7-migrate-log.txt" && echo 'PASS: db-migrate chay uid 999'

# Diễn tập trên worker2
kubectl apply -f "$W/nonroot-thieu-uid.yaml"
for i in $(seq 1 30); do
  [ "$(jp pod/nonroot-thieu-uid '{.status.containerStatuses[0].state.waiting.reason}')" = CreateContainerConfigError ] && break
  sleep 2
done
R="$(jp pod/nonroot-thieu-uid '{.status.containerStatuses[0].state.waiting.reason}')"
M="$(jp pod/nonroot-thieu-uid '{.status.containerStatuses[0].state.waiting.message}')"
echo "node=$(jp pod/nonroot-thieu-uid '{.spec.nodeName}') reason=$R message=$M" | tee "$EV/b7-nonroot.txt"
{ [ "$(jp pod/nonroot-thieu-uid '{.spec.nodeName}')" = lab-k8s-worker2 ] && [ "$R" = CreateContainerConfigError ] && \
  echo "$M" | grep -q runAsNonRoot; } && echo 'PASS: duoc lap lich nhung kubelet tu choi — runAsNonRoot kiem o node'
kubectl -n "$NS" delete pod nonroot-thieu-uid --wait=true && echo 'PASS: da xoa Pod dien tap'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

CronJob `db-backup`: khối `securityContext` cấp Pod và hai khối cấp container đã có trong lời giải
B3. Job `db-migrate` — thêm vào spec Pod và container `migrate`; phần SQL giữ nguyên:

```yaml
    spec:
      serviceAccountName: db-migrate
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
      containers:
      - name: migrate
        # image, env, resources: giữ nguyên
        securityContext:
          allowPrivilegeEscalation: false
        command:
        - sh
        - -c
        - |
          set -e
          echo "db-migrate: $(id)"
          grep NoNewPrivs /proc/self/status | sed 's/^/db-migrate: /'
          # --- phần migration của bạn giữ nguyên từ đây ---
```

```yaml
# ~/lab-work/9a/nonroot-thieu-uid.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nonroot-thieu-uid
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  securityContext:
    runAsNonRoot: true               # không khai runAsUser; image busybox chạy root
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 50m, memory: 32Mi}
```

Ý chính cho ADR:

1. `emptyDir` và thư mục do local-path-provisioner tạo cho phép mọi user ghi (đọc `ls -ld` của bạn),
   nên uid `999` ghi được mà không cần đổi group sở hữu. `fsGroup` cần khi volume chỉ cho một group
   ghi; bài 291 *Thảo luận*: nó chỉ tác động lên volume **hỗ trợ quản lý quyền sở hữu**.
2. Thường có thêm một gid (nhóm mà gói PostgreSQL thêm user `postgres` vào trong `/etc/group` của
   image). Kubernetes gộp group của image với group khai trong Pod — tập group thật rộng hơn manifest,
   và group quyết định quyền đọc ghi file trên volume.
3. Entrypoint của `postgres` khởi động bằng root để chỉnh quyền thư mục dữ liệu rồi mới hạ xuống
   `999`. Ép uid `999` cho `db` cần chuẩn bị `PGDATA` khác đi — việc của Lab 9b (`TD-06`). Job chỉ chạy
   client, không qua entrypoint đó.
4. Ở **kubelet**, sau khi Pod đã được ghi và lập lịch — Pod tồn tại, có node, kẹt
   `CreateContainerConfigError`. B6: chặng tiếp nhận từ chối **trước** khi Pod tồn tại, lỗi nằm ở sự
   kiện của controller.

</details>

## B8. [BS-9A-08] Tham số kernel an toàn cho `api`

**Loại:** Tính năng, kèm diễn tập trên `lab-k8s-worker2`.

**Bối cảnh:** `api` giữ một pool kết nối TCP tới `db-0`. Khi node của DB mất mạng đột ngột, kết nối
nhàn rỗi có thể treo rất lâu trước khi bị coi là chết — mặc định của kernel tính bằng giờ. Thư viện
client PostgreSQL bật TCP keepalive nhưng dùng thông số của kernel. Ta chỉnh thông số đó **cho riêng
Pod `api`**, không chạm node.

**Yêu cầu:**

1. Trong Pod `api`: TCP keepalive thăm dò sau 60 giây nhàn rỗi, mỗi 10 giây một lần, bỏ sau 6 lần.
   Chỉ áp cho Pod `api`; giá trị trên node không đổi.
2. Chỉ dùng sysctl trong nhóm an toàn; không đổi cấu hình kubelet.
3. Chứng minh giá trị có hiệu lực **bên trong** Pod `api` — image không có shell, dùng cách gỡ lỗi đã
   học — và giá trị trên node vẫn là mặc định. Thay Pod đã gắn debugger bằng Pod mới khi xong.
4. Diễn tập: Pod thử nghiệm dùng một sysctl **không** an toàn, ghim worker2. Chứng minh nó được lập
   lịch nhưng không chạy; đọc lý do; xóa.

**Ràng buộc:** không `--allowed-unsafe-sysctls`, không `hostNetwork`, không ghi vào `/proc/sys`.

**Đọc lại:**

- Bài [258](../258-sysctl-cluster-vi.md):
  [*Sysctl an toàn và không an toàn*](../258-sysctl-cluster-vi.md#sysctl-an-toàn-và-không-an-toàn-safe-and-unsafe-sysctls),
  [*Bật các sysctl không an toàn*](../258-sysctl-cluster-vi.md#bật-các-sysctl-không-an-toàn-enabling-unsafe-sysctls),
  [*Thiết lập sysctl cho một Pod*](../258-sysctl-cluster-vi.md#thiết-lập-sysctl-cho-một-pod-setting-sysctls-for-a-pod).
- Bài [291](../291-security-context-vi.md#set-the-security-context-for-a-pod) — `securityContext` cấp Pod
  áp cho mọi container.
- Bài [52](../52-ephemeral-containers-vi.md) và [Lab 3a](LAB-3A-POD-VA-VONG-DOI.md) ticket B8 — debug
  container dùng chung network namespace của Pod.
- PostgreSQL 17, tham số
  [`keepalives`](https://www.postgresql.org/docs/17/libpq-connect.html#LIBPQ-KEEPALIVES) của libpq.

**Bàn giao:** manifest Deployment `api`; `~/lab-work/9a/sysctl-khong-an-toan.yaml`;
`docs/adr/NNNN-sysctl-api.md`; `~/lab-evidence/9a/b8-*.txt`.

**Nghiệm thu:**

- `PASS:` template của `api` khai ba sysctl đúng giá trị; `api` 2/2 Ready.
- `PASS:` trong Pod `api` đọc được đúng ba giá trị; trên node chạy Pod đó, `tcp_keepalive_time` khác 60.
- `PASS:` Pod `api` đã gắn debugger được thay; `api` Ready lại.
- `PASS:` Pod diễn tập có node, `reason` là `SysctlForbidden`, phase `Failed`, nằm trên worker2; đã xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-sysctl-api.md`:

1. Ba điều kiện của bài 258 để một sysctl là "an toàn". Vì sao đặt keepalive cho Pod `api` không ảnh
   hưởng Pod khác trên cùng node — giá trị bạn đo được ở bước 3 chứng minh điều gì?
2. Vì sao sysctl khai ở cấp Pod chứ không ở từng container?
3. Nếu `node-agent` chuyển sang `hostNetwork`, nó còn đặt được sysctl `net.*` không?
4. Pod dùng sysctl không an toàn: được lập lịch không, chạy không, bạn tìm dấu vết ở đâu? So với hai
   kiểu chết ở B6 và B7.

<details>
<summary>Gợi ý</summary>

- Field: `spec.securityContext.sysctls` — danh sách `name`/`value`, `value` là chuỗi.
- Đọc giá trị trong network namespace của Pod: gắn debug container `busybox` (không cần `--target`), rồi
  `kubectl exec <pod> -c <debugger> -- cat /proc/sys/net/ipv4/tcp_keepalive_time`.
- Trên node: `ssh <node> 'sysctl -n net.ipv4.tcp_keepalive_time'`.
- Sysctl không an toàn mà bài 258 dùng làm ví dụ: `net.core.somaxconn`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/9a; W=~/lab-work/9a
cd ~/bookstore && ./scripts/deploy.sh > "$EV/b8-deploy.txt" 2>&1
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

jp deployment/api '{range .spec.template.spec.securityContext.sysctls[*]}{.name}={.value}{"\n"}{end}' > "$EV/b8-sysctl-spec.txt"
BAD=0
for X in net.ipv4.tcp_keepalive_time=60 net.ipv4.tcp_keepalive_intvl=10 net.ipv4.tcp_keepalive_probes=6; do
  grep -qx "$X" "$EV/b8-sysctl-spec.txt" || BAD=1
done
{ [ "$BAD" -eq 0 ] && [ "$(jp deployment/api '{.status.readyReplicas}')" = 2 ]; } \
  && echo 'PASS: api khai ba sysctl keepalive va 2/2 Ready'

API="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=api -o jsonpath='{.items[0].metadata.name}')"
NODE="$(jp "pod/$API" '{.spec.nodeName}')"
kubectl -n "$NS" debug "pod/$API" --image=busybox:1.37 -c sysctl-dbg -- sleep 300 >/dev/null
for i in $(seq 1 30); do
  [ -n "$(jp "pod/$API" '{.status.ephemeralContainerStatuses[?(@.name=="sysctl-dbg")].state.running.startedAt}')" ] && break
  sleep 2
done
kubectl -n "$NS" exec "$API" -c sysctl-dbg -- sh -c \
  'for f in tcp_keepalive_time tcp_keepalive_intvl tcp_keepalive_probes; do echo "$f=$(cat /proc/sys/net/ipv4/$f)"; done' \
  > "$EV/b8-trong-pod.txt"
TREN_NODE="$(ssh "$NODE" 'sysctl -n net.ipv4.tcp_keepalive_time')"
cat "$EV/b8-trong-pod.txt"; echo "tren node $NODE: tcp_keepalive_time=$TREN_NODE" | tee "$EV/b8-tren-node.txt"
{ grep -qx 'tcp_keepalive_time=60' "$EV/b8-trong-pod.txt" && grep -qx 'tcp_keepalive_intvl=10' "$EV/b8-trong-pod.txt" && \
  grep -qx 'tcp_keepalive_probes=6' "$EV/b8-trong-pod.txt"; } && echo 'PASS: ba gia tri co hieu luc ben trong Pod api'
[ -n "$TREN_NODE" ] && [ "$TREN_NODE" != 60 ] && echo 'PASS: gia tri tren node khong doi'
kubectl -n "$NS" delete pod "$API" --wait=true && kubectl -n "$NS" rollout status deployment/api --timeout=180s >/dev/null \
  && echo 'PASS: Pod api mang debugger da duoc thay, api Ready lai'

kubectl apply -f "$W/sysctl-khong-an-toan.yaml"
for i in $(seq 1 30); do [ "$(jp pod/sysctl-khong-an-toan '{.status.reason}')" = SysctlForbidden ] && break; sleep 2; done
echo "node=$(jp pod/sysctl-khong-an-toan '{.spec.nodeName}') reason=$(jp pod/sysctl-khong-an-toan '{.status.reason}') phase=$(jp pod/sysctl-khong-an-toan '{.status.phase}')" \
  | tee "$EV/b8-khong-an-toan.txt"
{ [ "$(jp pod/sysctl-khong-an-toan '{.spec.nodeName}')" = lab-k8s-worker2 ] && \
  [ "$(jp pod/sysctl-khong-an-toan '{.status.reason}')" = SysctlForbidden ] && \
  [ "$(jp pod/sysctl-khong-an-toan '{.status.phase}')" = Failed ]; } \
  && echo 'PASS: sysctl khong an toan — duoc lap lich, kubelet tu choi, Pod Failed'
kubectl -n "$NS" delete pod sysctl-khong-an-toan --wait=true && echo 'PASS: da xoa Pod dien tap'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Thêm vào spec Pod của Deployment `api` (cấp Pod, cùng cấp với `serviceAccountName`):

```yaml
      securityContext:
        sysctls:
        - name: net.ipv4.tcp_keepalive_time
          value: "60"
        - name: net.ipv4.tcp_keepalive_intvl
          value: "10"
        - name: net.ipv4.tcp_keepalive_probes
          value: "6"
```

```yaml
# ~/lab-work/9a/sysctl-khong-an-toan.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-khong-an-toan
  namespace: bookstore
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  securityContext:
    sysctls:
    - name: net.core.somaxconn
      value: "1024"
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 50m, memory: 32Mi}
```

Ý chính cho ADR:

1. Không ảnh hưởng Pod khác trên node, không hại sức khỏe node, không giành thêm CPU/bộ nhớ ngoài
   limits. Ba sysctl keepalive nằm trong danh sách an toàn và được namespace hóa theo network namespace:
   trong Pod là `60`, trên node vẫn là mặc định — đó là bằng chứng cách ly.
2. Sysctl gắn với namespace của kernel, mà các container của Pod dùng chung network namespace — không có
   khái niệm "sysctl mạng của riêng một container". Bài 258: `securityContext` cấp Pod áp cho mọi
   container.
3. Không. Ghi chú của bài 258: sysctl `net.*` không được phép khi bật host networking — Pod dùng network
   namespace của node, đặt sysctl là đổi node.
4. Được lập lịch, **không** chạy: kubelet trên worker2 từ chối, Pod `Failed` với `SysctlForbidden`.
   Dấu vết ở trạng thái Pod và sự kiện của kubelet — giống B7 (đã qua API server, chết ở node), khác B6
   (chết trong API server, chỉ để lại sự kiện của controller hoặc câu lỗi ở client).

</details>

## B9. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 15, rồi trả cluster về đúng
`03-storage-ready` — không object RBAC nào sót, cấu hình control plane không đổi, không token nào
trong bằng chứng.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 9a — kiểm thử khói **tích lũy**, chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 9a: hệ thống của sprint 11 (dữ liệu bền, mạng zero-trust, HTTPS,
# lập lịch, hạn mức) cộng danh tính và quyền của sprint 15. Chạy trên bộ VM chính, không cho overlay ha-lab.
set -u
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
can() { kubectl auth can-i "$@" -n "$NS" 2>/dev/null; }

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

# 10. Repo
git -C "$(dirname "$0")/.." check-ignore -q secrets/ci-deployer.kubeconfig \
  && pass 'kubeconfig CI nam ngoai git' || fail 'secrets/ khong bi .gitignore chan'

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

2. Cập nhật `docs/tech-debt.md`: `TD-07` ghi giới hạn ở 9a (B5), `TD-15` mới (B4). `TD-06` vẫn mở.

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/9a/b9-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/9a/b9-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/9a/b9-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
rm -f secrets/ci-deployer.kubeconfig
git status --short
git add -A && git commit -m 'lab-9a: sprint 15 — danh tinh va quyen'
git tag lab-9a
git bundle create ~/bookstore-lab-9a.bundle --all && git bundle verify ~/bookstore-lab-9a.bundle
```

Chép `~/bookstore-lab-9a.bundle` về máy host bằng `scp` trước khi đi tiếp.

5. Dọn object thử nghiệm và trả cluster:

```bash
kubectl -n bookstore delete pod,job,networkpolicy -l '!app.kubernetes.io/part-of' --ignore-not-found --wait=true
kubectl delete namespace lab-9a-thu --ignore-not-found --wait=true
rm -rf ~/lab-work/9a ~/bookstore-truoc-9a
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd ~/bookstore; EV=~/lab-evidence/9a; SEL=app.kubernetes.io/part-of=bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-9a' && echo 'PASS: HEAD mang tag lab-9a'
test -s ~/bookstore-lab-9a.bundle && echo 'PASS: co bundle'
{ test -f k8s/app/rbac/serviceaccounts.yaml && test -f k8s/app/rbac/backup-status-writer.yaml && \
  test -f k8s/app/rbac/ci-deployer.yaml && test -f k8s/app/rbac/nguoi-dung.yaml && \
  test -f k8s/app/backup/status-configmap.yaml && test -f k8s/app/network/backup-to-apiserver.yaml && \
  test -x scripts/ci-kubeconfig.sh; } && echo 'PASS: bo cuc repo dung muc 5.5'
BAD=0
for F in 'adr/*-ban-do-danh-tinh.md' 'adr/*-serviceaccount-tung-thanh-phan.md' 'adr/*-backup-status-qua-api.md' \
         'adr/*-danh-tinh-ci.md' 'adr/*-quyen-nguoi-dung.md' 'adr/*-danh-tinh-tien-trinh.md' 'adr/*-sysctl-api.md' \
         'postmortem/9a-request-bi-chan.md' 'runbook-ci-deployer.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR, postmortem, runbook cua sprint 15'
grep -q 'TD-15' docs/tech-debt.md && grep 'TD-07' docs/tech-debt.md | grep -q '9a' && echo 'PASS: tech-debt.md cap nhat TD-07, TD-15'

{ [ "$(sl kubectl get namespace,priorityclass,storageclass,clusterrole,clusterrolebinding -l "$SEL" --no-headers)" -eq 0 ] && \
  [ "$(sl kubectl get all,configmaps,secrets,serviceaccounts,roles,rolebindings,networkpolicies -A -l "$SEL" --no-headers)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an, ke ca RBAC'
test -z "$(kubectl get nodes -l bookstore.example.com/data -o name)" && echo 'PASS: label node cua du an da go'
{ echo "clusterrole=$(kubectl get clusterrole --no-headers | wc -l)"
  echo "clusterrolebinding=$(kubectl get clusterrolebinding --no-headers | wc -l)"; } > "$EV/b9-rbac-cluster.txt"
diff -q "$EV/b0-rbac-cluster.txt" "$EV/b9-rbac-cluster.txt" >/dev/null && echo 'PASS: so ClusterRole, ClusterRoleBinding tro ve dung B0'
{ echo "apiserver-manifest $(sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml | awk '{print $1}')"
  echo "kubelet-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "kubelet-$n $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done; } > "$EV/b9-cauhinh-sha.txt"
diff -q "$EV/b0-cauhinh-sha.txt" "$EV/b9-cauhinh-sha.txt" >/dev/null && echo 'PASS: cau hinh control plane va kubelet khong doi'
[ "$(grep -rl 'eyJhbGciOi' "$EV" 2>/dev/null | wc -l)" -eq 0 ] && echo 'PASS: khong file bang chung nao chua token'
test ! -e ~/lab-work/9a && echo 'PASS: da don lab-work'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
kubectl get storageclass | grep -q '(default)' && echo 'PASS: van con StorageClass mac dinh cua 03-storage-ready'
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; mười hai dòng `PASS:` đầu của gate
cuối xuất hiện; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica; namespace `default` không có Pod; dòng `PASS: van con StorageClass mac dinh ...`.
Thư mục `~/lab-evidence/9a/` **giữ lại**. Cluster trở về `03-storage-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 9a

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Kể bốn bước của một request sau TLS. Với mỗi bước, một request **của Bookstore** chết ở đó trong
      sprint này, và dấu hiệu nhận ra từ phía client. Một request chết **sau** API server trông thế nào?
- [ ] Ứng dụng báo `401` dù token vừa cấp và RoleBinding đủ. Ba nguyên nhân đã dựng lại được, và cách
      phân biệt bằng claim của token.
- [ ] Trước sprint, Pod `web` mang token nhưng NetworkPolicy chặn đường tới API server. Vì sao vẫn gỡ
      token? Tắt tự mount ở ServiceAccount và ở Pod — cấp nào thắng, và điều đó có phải hàng rào không?
- [ ] Role của `db-backup` chỉ có `get`, `patch` trên đúng `backup-status`. Vì sao không thêm được
      `create` theo tên, vì sao ConfigMap phải có sẵn từ repo, và vì sao nó không đi qua generator?
- [ ] Token trong Pod backup: ở đâu, trong container nào, sống bao lâu, container xác minh server bằng
      gì? Vì sao không dùng `wget` của busybox hay image `postgres` để gọi API?
- [ ] NetworkPolicy cho Pod backup tới API server: rule theo đích nào có hiệu lực trên cluster của bạn,
      bài 84 nói gì về việc đó, và rule gãy khi nào?
- [ ] Danh tính CI: phát hành, thời hạn, thu hồi ngay bằng cách nào. Tạo lại ServiceAccount cùng tên
      thì kubeconfig mới có quyền gì — vì sao, và rủi ro rà soát nào đi kèm?
- [ ] `TD-15`: vì sao quyền sửa Deployment gần như bằng quyền đọc Secret và mượn ServiceAccount? Bạn đã
      chứng minh bằng gì mà không làm lộ gì?
- [ ] `bookstore-devs` và `bookstore-oncall`: mỗi nhóm làm được gì, không làm được gì. Vì sao `view`
      không có Secret? `TD-07` còn mở phần nào?
- [ ] Kubernetes không có object `User`. Vậy RoleBinding cho `bookstore-devs` có tác dụng nhờ đâu?
      `kubectl auth can-i --as --as-group` liên quan gì tới SubjectAccessReview?
- [ ] Trong Pod backup: ai chạy uid nào, gid nào, nhờ thiết lập ở cấp nào; cờ `NoNewPrivs` đến từ field
      nào; `runAsNonRoot` bị kiểm ở đâu?
- [ ] Sysctl keepalive của `api`: vì sao là "an toàn", đặt ở cấp nào, giá trị trên node có đổi không.
      Pod dùng sysctl không an toàn: lập lịch? chạy? dấu vết ở đâu?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Nợ lab #6 (mã hóa Secret at rest — giai đoạn 22). Lab không phát sinh nợ lab mới.
- Nợ kỹ thuật: `TD-06` (`db` khởi động bằng root — Lab 9b); `TD-07` **đã giới hạn** nhưng người trực
  vẫn đọc được biến môi trường của `api`; `TD-15` mới (quyền sửa Deployment của CI — chấp nhận có điều
  kiện, xem lại khi học admission policy ở giai đoạn 14); `TD-10` giữ trạng thái từ Lab 6b.

### Bài giải thích cuối cùng

Trong vài phút, kể lại một lần chạy của `db-backup` từ góc nhìn danh tính: CronJob tạo Job;
`job-controller` — một ServiceAccount của `kube-system` — tạo Pod; chặng tiếp nhận kiểm ServiceAccount
`db-backup` có tồn tại; kubelet chiếu token 10 phút, CA và namespace vào **đúng** container
`report-status`, chạy hai container bằng uid nào, gid nào; `pg-dump` xong mới tới `report-status`;
request `PATCH` đi qua NetworkPolicy nào, tới API server, qua chặng 1 bằng claim nào, qua chặng 2 nhờ
Role nào, rồi ghi `backup-status`. Rồi kể ngược: nếu tên ServiceAccount gõ sai, nếu token xin sai
audience, nếu Role thiếu `patch`, nếu NetworkPolicy theo ClusterIP — mỗi trường hợp chết ở đâu và bạn
thấy gì. Kết thúc bằng việc chỉ ra Lab 9b sẽ siết thêm điều gì lên chính Pod này.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `git describe` không ra `lab-8c`, hoặc HEAD ở trạng thái detached | Clone từ bundle cũ, hoặc bundle tạo khi HEAD không ở nhánh | Clone lại từ `bookstore-lab-8c.bundle` mới nhất; `git checkout <nhánh chính>` rồi kiểm `HEAD` trùng commit của tag |
| B0: smoke của `lab-8c` fail ở phần Ingress hoặc NetworkPolicy | Bản smoke của tag đó viết cho overlay `ha-lab` trên bộ VM HA | Chạy bản tích lũy cuối cùng của chuỗi chính: `git show lab-7b:tests/smoke.sh \| bash`; ghi lại vào ADR của B1 |
| B0: `deploy.sh` treo, PVC `Pending` | Không ở `03-storage-ready`, thiếu StorageClass mặc định | Restore cả ba VM về `03-storage-ready` rồi làm lại B0 |
| B1: `CN` hoặc `O` rỗng | Kubeconfig trỏ file certificate thay vì nhúng dữ liệu | Đọc trường `client-certificate` rồi `openssl x509 -in <đường dẫn>`; không sửa kubeconfig |
| B1: `patch serviceaccount default` báo `not found` | Controller chưa kịp tạo `default` trong namespace mới | Vòng `until` trong script chờ đúng việc đó; chạy lại khối |
| B1: `wget` trong Pod thử nghiệm lỗi bắt tay TLS | `busybox` không hoàn tất TLS với API server | Lấy token ra biến trên master bằng `kubectl exec ... cat`, gọi bằng `curl --cacert`, rồi `unset`; không thêm image |
| B2: rollout kẹt, sự kiện `FailedCreate ... error looking up service account` | File ServiceAccount chưa nằm trong `resources` của kustomization | Thêm `rbac/serviceaccounts.yaml`, `./scripts/deploy.sh` |
| B2: `deploy.sh` báo `field is immutable` cho Job `db-migrate` | Template Job đổi mà Job cũ còn (TTL chưa dọn) | `kubectl -n bookstore delete job db-migrate` rồi deploy lại; nếu `deploy.sh` của bạn không tự xóa Job trước khi apply, sửa nó và ghi vào ADR |
| B2: Pod vẫn có `kube-api-access-*` | Pod tạo trước khi ServiceAccount đổi, hoặc template khai `automountServiceAccountToken: true` | Quyết định này xảy ra lúc tạo Pod: xóa Pod hoặc rollout lại; bỏ dòng `true` khỏi template |
| B3: Job backup `Init` xong nhưng `report-status` báo `HTTP 000` | NetworkPolicy chưa mở, hoặc mở theo ClusterIP | Rule theo endpoint thật của Service `kubernetes`, cổng của endpoint; xem lại điều tra NetworkPolicy |
| B3: `HTTP 403` từ `PATCH` | Sai `resourceNames`, subject của RoleBinding sai namespace, hoặc ConfigMap mang hậu tố hash | `kubectl auth can-i patch configmaps/backup-status --as=system:serviceaccount:bookstore:db-backup -n bookstore`; `backup-status` phải là resource thường, không qua generator |
| B3: `HTTP 404` từ `PATCH` | `backup-status` chưa tồn tại — `patch` không tạo object | Kiểm `backup/status-configmap.yaml` có trong kustomization |
| B3: `HTTP 401` | Token khai `audience` khác API server, hoặc đọc nhầm file | Bỏ `audience` khỏi nguồn `serviceAccountToken`; kiểm đường mount |
| B3: `curl: (60) SSL certificate problem` | Trỏ `--cacert` sai file | Dùng `ca.crt` chiếu từ ConfigMap `kube-root-ca.crt` |
| B3: `curl: not found` | Dùng biến thể image không có `curl` (`-alpine-slim`) | Dùng đúng tag ở A1.4; kiểm bằng `command -v curl` trong Pod tạm |
| B3: trạng thái mất sau mỗi `deploy.sh` | Manifest `backup-status` khai `data` | Bỏ hẳn `data` khỏi manifest trong repo |
| B3: tạo Job backup báo `exceeded quota` | Container mới làm tổng `requests` của Pod backup vượt hạn mức Lab 7b | Giảm `requests` của container mới, hoặc nâng quota trong repo kèm ADR |
| B4: `kubeconfig` quyền khác `600` hoặc `git status` thấy file | Tạo file ngoài script; `.gitignore` không phải `secrets/*` | Chạy lại `ci-kubeconfig.sh`; `.gitignore` theo Lab 3b |
| B4: `set image --dry-run=server` bị `Forbidden` | Role thiếu `patch` trên `deployments` | `kubectl set image` đọc rồi `patch`; thêm verb vào Role |
| B4: vòng lặp thu hồi chạy hết mà vẫn `200` | ServiceAccount chưa thật sự bị xóa | `kubectl -n bookstore get sa ci-deployer`; thời gian lan truyền **phụ thuộc cấu hình**, tăng số vòng lặp |
| B4: `kubectl proxy` không lên | Cổng `8001` bận | `ss -ltnp \| grep 8001`, dừng tiến trình cũ |
| B5: `kubectl debug` của oncall bị `Forbidden` | Thiếu `get pods`, hoặc verb trên `pods/ephemeralcontainers` không phải `patch` | Đọc thông báo — nó nêu đúng verb và resource bị từ chối |
| B5: `kubectl exec` của oncall vào debugger bị `Forbidden` | Chỉ cấp `create` hoặc chỉ `get` trên `pods/exec` | Cấp cả hai |
| B5: `--as-group` báo lỗi | Thiếu `--as` | Luôn đi cùng `--as=<tên bất kỳ>` |
| B6: không có sự kiện `FailedCreate` | ServiceAccount gõ sai tình cờ tồn tại, hoặc lọc sự kiện sai tên | Đổi tên sai khác đi; lọc theo `involvedObject.name` đúng tên Job |
| B6: request không credential ra mã khác kỳ vọng | Cấu hình `anonymous-auth` của cluster khác baseline | Ghi giá trị thật vào postmortem; **không** sửa manifest API server |
| B7: `pg-dump` báo `Permission denied` khi ghi dump | Thư mục PVC không cho uid `999` ghi | `ls -ld` thư mục của PV trên node; ghi vào ADR; `fsGroup` chỉ giúp với volume hỗ trợ quản lý quyền sở hữu |
| B7: `CreateContainerConfigError` ở Job backup | Khai `runAsNonRoot` mà thiếu `runAsUser` ở cấp áp cho container đó | Đặt `runAsUser` cấp Pod, ghi đè cấp container nếu cần |
| B8: Pod `api` không lên sau khi thêm sysctl | Tên sysctl gõ sai, hoặc dùng sysctl không nằm trong danh sách an toàn | `kubectl describe pod`; đối chiếu danh sách ở bài 258 |
| B9: namespace kẹt `Terminating` | `db-0` đang trong thời gian ân hạn | Chờ; `teardown.sh` chờ có giới hạn |
| B9: gate token trong bằng chứng fail | Một lệnh đã ghi cả token vào file | Xóa file đó, chạy lại đúng bước sinh ra nó theo bản trong lab |
| B9: số ClusterRole/ClusterRoleBinding khác B0 | Có object cấp cluster tạo ngoài kịch bản | `kubectl get clusterrole,clusterrolebinding --sort-by=.metadata.creationTimestamp \| tail`; xóa đúng cái đó, **không** đụng object `system:` |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Security](https://v1-35.docs.kubernetes.io/docs/concepts/security/)
- [Kubernetes v1.35 — Cloud Native Security and Kubernetes](https://v1-35.docs.kubernetes.io/docs/concepts/security/cloud-native-security/)
- [Kubernetes v1.35 — Service Accounts](https://v1-35.docs.kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes v1.35 — Controlling Access to the Kubernetes API](https://v1-35.docs.kubernetes.io/docs/concepts/security/controlling-access/)
- [Kubernetes v1.35 — Role Based Access Control Good Practices](https://v1-35.docs.kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Kubernetes v1.35 — Cloud Controller Manager Administration](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/)
- [Kubernetes v1.35 — Using sysctls in a Kubernetes Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/)
- [Kubernetes v1.35 — Configure Service Accounts for Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [Kubernetes v1.35 — Enforce Pod Security Standards by Configuring the Built-in Admission Controller](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/)
- [Kubernetes v1.35 — Enforce Pod Security Standards with Namespace Labels](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)
- [Kubernetes v1.35 — Migrate from PodSecurityPolicy to the Built-In PodSecurity Admission Controller](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/migrate-from-psp/)
- [Kubernetes v1.35 — Configure a Security Context for a Pod or Container](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Kubernetes v1.35 — Accessing the Kubernetes API from a Pod](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/access-api-from-pod/)
- [Kubernetes v1.35 — Accessing Clusters](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/access-cluster/)
- [Kubernetes v1.35 — Access Clusters Using the Kubernetes API](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/)
- [Kubernetes v1.35 — Authenticating](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes v1.35 — Authorization](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Kubernetes v1.35 — Using RBAC Authorization](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes v1.35 — Admission Controllers Reference](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Kubernetes v1.35 — Managing Service Accounts](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
- [Kubernetes v1.35 — TokenRequest](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)
- [Kubernetes v1.35 — SubjectAccessReview](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/authorization-resources/subject-access-review-v1/)
- [Kubernetes v1.35 — Network Policies](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes v1.35 — Debug Running Pods](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- [PostgreSQL 17 — libpq, tham số `keepalives`](https://www.postgresql.org/docs/17/libpq-connect.html#LIBPQ-KEEPALIVES)
- [BusyBox — `networking/wget.c`](https://github.com/mirror/busybox/blob/master/networking/wget.c) — chỉ `GET`/`POST`, ghi chú về kiểm certificate
- [GitHub — nginxinc/docker-nginx-unprivileged, `stable/alpine/Dockerfile`](https://github.com/nginxinc/docker-nginx-unprivileged/blob/main/stable/alpine/Dockerfile) — `curl` và `ca-certificates` có trong biến thể `-alpine`
- [GitHub — docker-library/postgres, `17/bookworm/Dockerfile`](https://github.com/docker-library/postgres/blob/master/17/bookworm/Dockerfile) — không có HTTP client trong image
