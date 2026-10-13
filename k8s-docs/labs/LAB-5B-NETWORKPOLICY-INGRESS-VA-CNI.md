# Lab 5b — NetworkPolicy, Ingress và CNI

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> Repo `~/bookstore` ở tag `lab-5a` (hoặc bundle `~/bookstore-lab-5a.bundle` nếu bạn vừa restore
> snapshot).
> **Điểm kết thúc:** **tạo snapshot mới `02-net-ready`**. Lab đổi CNI sang Calico và cài ingress
> controller Traefik, tức là **đổi hạ tầng vĩnh viễn** trên chuỗi snapshot chính. Snapshot chụp
> **sau** teardown nên chỉ chứa hạ tầng, không chứa dự án. Repo `~/bookstore` có tag `lab-5b` và một
> bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 7 — Mạng zero-trust và HTTPS của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 5a — Service, EndpointSlice và DNS](LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md).
> **Lab sau:** [Lab 6a — PV, PVC và StorageClass](LAB-6A-PV-PVC-VA-STORAGECLASS.md), bắt đầu từ
> `02-net-ready` và tag `lab-5b`.
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng phần policy/ingress/CNI của
[Giai đoạn 5 — Mạng nền tảng](../00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng): nhóm bài đứng
**sau** Lab 5a — [11](../11-ingress-vi.md), [12](../12-ingress-controllers-vi.md),
[13](../13-gateway-vi.md), [85](../85-dual-stack-vi.md) — và nhánh
[Tầng hạ tầng mạng của cluster](../00-ALO-TRINH-ADMIN.md#tầng-hạ-tầng-mạng-của-cluster)
([157](../157-networking-vi.md), [183](../183-network-plugins-vi.md), [164](../164-proxies-vi.md)).
Mọi ticket dựa vào hợp đồng ở [mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật), ma trận
NetworkPolicy ở [mục 3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật) và bố cục repo ở
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Kế hoạch gốc của sprint nằm ở
[mục 7.2](DU-AN-BOOKSTORE.md#sprint-7--lab-5b-mạng-zero-trust-và-https).

Đây cũng là lab **trả [nợ lab #4](README.md#5-sổ-nợ-lab) — NetworkPolicy được thực thi thật** — và
nợ kỹ thuật `TD-09`. **Đọc lại bài [84 — Chính sách mạng](../84-network-policies-vi.md) trước khi bắt
đầu**: sprint 6 đã viết policy theo bài đó nhưng chưa ai thấy policy chặn được gì.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Ba thành phần lab
này đưa vào chuỗi snapshot — Calico, Helm, Traefik — đã chốt version ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Lab **không chép lại**
con số nào trong văn bản: repo đọc chúng từ một file bạn tự chép từ A1.4 ở ticket B3.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi hệ thống Bookstore đạt trạng thái sau sprint 7 — **mạng giữa các tầng đóng theo
ma trận, đường vào duy nhất là HTTPS qua Ingress** — cluster đã được chụp `02-net-ready`, **và** bạn
chứng minh, giải thích được:

- Một NetworkPolicy hợp lệ, được API server lưu, **không chặn gì** khi CNI không hiện thực nó; và
  **cùng một phép đo** cho kết quả ngược lại sau khi đổi CNI. Thứ duy nhất đổi giữa hai lần đo là
  network plugin.
- Ba dải IP của cluster — Pod, Service, Node — do ba thành phần khác nhau cấp; vì sao đổi CNI làm
  IP của Pod đổi chủ còn ClusterIP thì không; vì sao `ip addr` không phải thứ Kubernetes nhìn.
- Ai nạp CNI plugin trên node, hai đường dẫn phải đọc khi gỡ lỗi, và quy trình đổi CNI an toàn: vì
  sao gỡ dự án trước, vì sao gỡ plugin cũ ở tầng API trước khi dọn node, vì sao khởi động lại
  containerd trước kubelet.
- Ma trận zero-trust của dự án: mặc định chặn **hai chiều**; mỗi luồng cần rule ở **cả hai đầu**; DNS
  phải mở riêng; cổng trong policy là cổng **Pod**; một phần tử `from` chứa hai selector là phép
  **giao**; và vì sao probe của kubelet không bị chặn.
- Không có controller thì Ingress vô nghĩa; IngressClass mặc định chỉ được áp lúc tạo object;
  controller là phần mềm bên thứ ba với hành vi riêng, có chỗ lệch khỏi đặc tả.
- TLS kết thúc tại Traefik: Secret kiểu `kubernetes.io/tls`, `hosts` khớp `host`, đoạn nào sau đó là
  plaintext, và vì sao certificate tự ký được sinh tại chỗ, không vào git.
- kube-proxy không hiểu HTTP; ClusterIP chỉ tồn tại dưới dạng rule trên node; năm thứ cùng tên
  "proxy" chạy ở năm chỗ khác nhau.
- Cluster là single-stack IPv4, chứng minh bằng phản ứng của API server; Gateway API là add-on chưa
  có trên cluster, khác Ingress ở chỗ thiếu cả **kiểu dữ liệu** chứ không chỉ thiếu người thực thi.
- Đưa cluster qua trọn bảy tầng gate A5.4 với CNI mới, và chụp `02-net-ready` khi cluster **chỉ còn
  hạ tầng**.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài | Ticket kiểm chứng |
| --- | --- |
| [84 — Chính sách mạng](../84-network-policies-vi.md) (nhóm 5a, **nợ #4**) | B1 — policy của sprint 6 được lưu mà không chặn gì trên Flannel; B5 — mặc định chặn hai chiều, rule ở hai đầu, cổng Pod, phép giao, bẫy DNS, *Vòng đời Pod*, ⭐ `endPort` (**trả nợ #4**) |
| [157 — Mạng trong cluster](../157-networking-vi.md) | B2 — ba dải IP đọc từ tiến trình đang chạy, `node.status.addresses` so với `ip addr`, PodCIDR từng node; B3 — IPAM đổi chủ trong khi `.spec.podCIDR` giữ nguyên |
| [183 — Network Plugin](../183-network-plugins-vi.md) | B2 — `/etc/cni/net.d`, `/opt/cni/bin`, `portMappings`, `loopback`, containerd nạp plugin, kubelet không còn cờ quản lý CNI; B3 — thay plugin rồi đọc lại hai đường dẫn |
| [12 — Ingress Controllers](../12-ingress-controllers-vi.md) | B4 — Ingress nằm im khi chưa có controller; cài Traefik (controller bên thứ ba); IngressClass mặc định duy nhất; B8 ⭐ — controller lệch khỏi đặc tả |
| [11 — Ingress](../11-ingress-vi.md) | B4 — `.status` rỗng, class mặc định không hồi tố; B6 — Ingress `bookstore`: `host`, `pathType: Prefix`, `ingressClassName`, TLS bằng Secret `kubernetes.io/tls`, TLS kết thúc tại ingress; B8 ⭐ — `Prefix` theo phần tử path, path khớp dài nhất thắng, `Exact`, wildcard host, `pathType` bắt buộc |
| [164 — Các loại proxy](../164-proxies-vi.md) | B4 — Service mặc định `LoadBalancer` của chart (loại 5) không có ai cấp địa chỉ; B7 — mode kube-proxy, ClusterIP trong rule của node, `kubectl proxy`, apiserver proxy. Loại 4 chỉ đọc: cần control plane HA (Lab 8b) |
| [85 — Dual-stack IPv4/IPv6](../85-dual-stack-vi.md) | B7 — `ipFamilyPolicy`, `ipFamilies`, `clusterIPs` của Service `api` và `db`; `RequireDualStack` bị từ chối, `PreferDualStack` quay về single-stack; ⭐ headless không selector |
| [13 — Gateway API](../13-gateway-vi.md) | B7 — **chỉ kiểm tính add-on**: API group không tồn tại, `HTTPRoute` bị từ chối, trong khi Traefik tự mang CRD riêng. Không cài Gateway API: đó là thêm một hạ tầng thứ hai vào lab đang đổi CNI, và CRD thuộc [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), thực hành ở [giai đoạn 28](../00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes). Quyết định định hướng ghi thành ADR ở B7 |

Các trang **mượn quy trình**, không thuộc nhóm 5b:

| Trang | Dùng ở |
| --- | --- |
| [201 — Khai báo Network Policy](../201-declare-network-policy-vi.md) | B1, B5 — trình tự đo: dựng đích → đo đường cơ sở → áp policy → thử Pod có quyền và Pod không có quyền |
| [243 — Cài đặt một Network Policy Provider](../243-network-policy-provider-vi.md), [245 — Sử dụng Calico cho NetworkPolicy](../245-calico-network-policy-vi.md) | B2 — chọn provider từ danh sách chính thức; B3 — nhánh *cluster cục bộ với kubeadm* |
| [109 — Secret](../109-secret-vi.md), [328 — Secret bằng Kustomize](../328-secret-kustomize-vi.md), [322 — Kustomize](../322-kustomization-vi.md) | B6 — Secret TLS sinh bằng `secretGenerator` từ file trong `secrets/` |

Nợ của sprint:

| Nợ | Trạng thái sau sprint 7 |
| --- | --- |
| Nợ lab #4 — NetworkPolicy được thực thi thật | **Trả** ở B5; bằng chứng hai đầu ở B1 và B5 |
| `TD-09` — Lưu lượng giữa các tầng chưa bị giới hạn | **Trả** ở B5 |
| `TD-11` (mới) — Certificate tự ký, client không tin được, gia hạn bằng tay; đoạn sau Traefik là plaintext | Phát sinh ở B6. Chưa có lab trả: công cụ cấp certificate nằm ngoài chuỗi lab ([A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00)) — chấp nhận có điều kiện |

Nợ còn mở từ trước: nợ lab #1 (HPA ở Lab 11b, VPA chưa có lab), #2 (`volumeClaimTemplates`, Lab
6a), #5 → #8; nợ kỹ thuật `TD-04`, `TD-06`, `TD-07`, `TD-08`.

Ba điểm cố ý dừng ở phần đo được, không phát sinh nợ:

- **Thứ tự khi hai path khớp ngang nhau** (`Exact` thắng `Prefix`) không vào gate: bài 11 và bài 12
  đều nói các controller hoạt động hơi khác nhau. B8 chỉ gate phần đặc tả nói rõ, và **so** chỗ
  Traefik lệch với cấu hình đang chạy của nó.
- **`hostPort` và điều tiết băng thông** của bài 183: chỉ đọc khai báo `portMappings` trong conflist;
  plugin `bandwidth` là tính năng thử nghiệm.
- **Loại proxy 4** của bài 164 cần nhiều apiserver — thực hành ở Lab 8b.

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B7, B9) khoảng 4 giờ rưỡi nếu không bí lâu ở ticket nào; B8 và các mục ⭐
là tùy chọn. **B3 dài nhất (60–90 phút) và rủi ro nhất** — đừng bắt đầu B3 khi không còn đủ thời gian
chạy hết B3 và B4. Chia hai phiên: phiên 1 làm B0–B4 (kết thúc khi cluster đã có Calico và Traefik,
dự án chưa triển khai lại); phiên 2 làm B5–B9. Giữa hai phiên có thể tắt ba VM. Đầu phiên 2 chạy lại
gate ngắn A5.5, rồi gate tiếp tục dưới đây — chỉ đọc, kiểm đúng trạng thái mà phiên 1 để lại. **Không**
chạy `deploy.sh` hay smoke: dự án chỉ được dựng lại ở B5, và smoke của tag `lab-5a` còn đòi NodePort
`30080` mà sprint này sẽ bỏ.

```bash
kubectl -n kube-system rollout status daemonset/calico-node --timeout=300s >/dev/null \
  && echo 'PASS: calico-node chay du tren moi node'
kubectl -n traefik rollout status deployment/traefik --timeout=300s >/dev/null && echo 'PASS: Traefik san sang'
kubectl get ingressclass traefik >/dev/null && echo 'PASS: co IngressClass traefik'
if R="$(kubectl get namespace kube-flannel --ignore-not-found -o name)"; then
  [ -z "$R" ] && echo 'PASS: khong con Flannel' || echo 'FAIL: namespace kube-flannel con'
else echo 'STOP: khong hoi duoc API'; fi
if R="$(kubectl get namespace -l app.kubernetes.io/part-of=bookstore -o name)"; then
  [ -z "$R" ] && echo 'PASS: Bookstore chua trien khai lai — dung diem dung cua phien 1' \
    || echo 'FAIL: da co namespace du an — phien 1 khong dung o diem dung'
else echo 'STOP: khong hoi duoc API'; fi
test -z "$(git -C ~/bookstore status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
```

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác. Lệnh trên node chạy qua
  `ssh <node> '...'` từ master như các lab trước; script dùng hàm `on` chạy tại chỗ khi node đích
  chính là master.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm và file tải về: `~/lab-work/5b/`. Bằng chứng:
  `~/lab-evidence/5b/`.
- Lab chỉ dùng kiến thức nhóm 5b và các nhóm trước. **Không** dùng PVC, StorageClass (giai đoạn 6),
  HPA, metrics-server (giai đoạn 11), RBAC tự viết (giai đoạn 9) — quyền của Calico và Traefik do
  chính manifest và chart của chúng khai báo.
- Object thử nghiệm nằm ở namespace `np-ngoai`, `ing-thu`, `net-thu`, hoặc là Pod không label trong
  `bookstore`, `traefik`. Chúng **không** mang label `app.kubernetes.io/part-of`; xóa khi xong ticket
  hoặc ở mục đóng sprint.
- **Ngoại lệ về phạm vi tác động:** B3 chạm **cả ba node**, vì đổi CNI là thao tác trên từng node chứ
  không trên API. Đó là lý do lab tạo snapshot mới. Ngoài B3, lệnh trên node chỉ để đọc. Lab không có
  ticket diễn tập sự cố; muốn tự phá thêm thì chỉ phá `lab-k8s-worker2`.
- Hạ tầng nền thuộc **cluster**, không thuộc repo dự án
  ([mục 3.3 của file dự án](DU-AN-BOOKSTORE.md#33-hạ-tầng-nền-của-cluster)). Calico và Traefik không
  mang label dự án, `teardown.sh` không gỡ chúng. Repo chỉ giữ **ghi chép** cách cài, trong
  `platform/`.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

### 2.1. Đổi CNI là thao tác phá hoại

Trong B3, giữa lúc gỡ Flannel và lúc Calico chạy xong, cluster **mất tầng mạng Pod**: node có thể
chuyển `NotReady`, CoreDNS không chạy được, Pod mới không có IP. Đó là trạng thái **đúng** của giai
đoạn giữa, không phải sự cố. Nhưng dừng lại ở đó, hoặc dọn sót interface và file cấu hình cũ, thì
cluster có thể không hồi phục bằng cách sửa vặt.

Bắt buộc trước B3:

1. **Snapshot `01-cluster-ready` còn nguyên trên cả ba VM** — B0 kiểm việc này từ máy host. Thiếu dù
   một VM thì **dừng**.
2. Dự án đã được gỡ khỏi cluster và repo đã có bundle ngoài VM của tag `lab-5a`.
3. Chạy trọn B3 trong một lần, không tắt máy giữa chừng.

Khi B3 hỏng và không sửa được trong vài phút bằng [mục 4](#4-troubleshooting-của-lab-này): tắt và
**restore cả ba VM về `01-cluster-ready`**, khôi phục repo từ bundle, chạy lại bảy tầng
[gate A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready), rồi làm lại từ B0. Không
restore riêng một VM.

**Phạm vi của B3: thí nghiệm, không phải quy trình được nhà phát hành hỗ trợ.** Trang
[System requirements của Calico](https://docs.tigera.io/calico/3.32/getting-started/kubernetes/requirements),
mục *Other network providers*, nói thẳng: cluster đã dùng CNI khác thì không migrate sang Calico được.
Kubernetes của baseline nằm trong danh sách được Calico test; vấn đề là **cách chuyển**, không phải
version. Vì vậy:

- B3 là thí nghiệm migration trên môi trường **hủy được** — ba VM có snapshot `01-cluster-ready`.
  Gate của B3 PASS một lần không biến quy trình thành bảo đảm của Calico, và cũng không phải quy
  trình để mang ra một cluster thật.
- Đường đúng với một cluster cần Calico là **dựng cluster mới với Calico từ đầu** — nhánh A của runbook,
  Lab 8a dùng đúng nhánh đó.
- Lưu bằng chứng của từng bước vào `~/lab-evidence/5b/b3-*.txt`, kể cả lần hỏng và lần restore nếu có.
  Chuỗi lab chưa được pilot trọn trên đúng baseline ([README mục 7.8](README.md#78-giới-hạn-hiện-tại)).

Lab vẫn giữ B3 vì hai lý do: lộ trình cần cluster thật từ giai đoạn 1 nhưng tới giai đoạn 5 mới học
NetworkPolicy, và tự tay đổi plugin là cách rõ nhất để thấy ai cấp IP Pod, ai nạp cấu hình CNI.

### 2.2. Phiên bản hạ tầng nằm ở đâu

Calico, Helm và chart Traefik (kèm `appVersion`) đã chốt ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). B3 cho bạn tạo
`platform/versions.env` trong repo bằng cách **chép tay** bốn giá trị từ bảng đó; mọi lệnh cài nền
trong lab và trong runbook `platform/` đọc số từ file này. A1.4 vẫn là nguồn duy nhất; lab không chép
con số nào.

---

# Phần B — Sprint 7: Mạng zero-trust và HTTPS

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `01-cluster-ready` với Flannel, repo ở đúng tag `lab-5a`, hệ
thống của sprint 6 dựng lại được **chỉ từ repo**, và có đường lùi trước khi đụng vào mạng.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Repo, thư mục làm việc và điểm xuất phát của mạng. Nếu vừa restore snapshot, repo đã mất — khôi phục
từ bundle theo [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) trước:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
mkdir -p ~/lab-work/5b ~/lab-evidence/5b
test -d ~/bookstore/.git || git clone ~/bookstore-lab-5a.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-5a' && echo 'PASS: HEAD dung tag lab-5a'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
test -s ~/bookstore-lab-5a.bundle && echo 'PASS: co bundle lab-5a de lui lai neu B3 hong'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
kubectl -n kube-flannel get daemonset kube-flannel-ds >/dev/null 2>&1 && echo 'PASS: CNI dang la Flannel'
kubectl -n kube-system get daemonset calico-node >/dev/null 2>&1 \
  && echo 'FAIL: cluster da co Calico - khong phai 01-cluster-ready' || echo 'PASS: chua co Calico'
test "$(sl kubectl get ingressclass --no-headers)" -eq 0 && echo 'PASS: chua co IngressClass nao'
```

Dựng lại hệ thống của sprint 6 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/5b/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/5b/b0-smoke-lab-5a.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/5b/b0-smoke-lab-5a.txt
```

Kéo sẵn image dùng cho Pod đo lên hai worker — Pod đo của B1 và B5 chạy image `postgres` vì nó có
`bash`, `getent` và `timeout`. Tag lấy đúng từ [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa)
và [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00):

```bash
for N in lab-k8s-worker1 lab-k8s-worker2; do
  for I in docker.io/library/busybox:1.37 docker.io/library/postgres:17.11; do
    ssh "$N" "sudo crictl pull $I" >/dev/null && echo "PASS: $N co $I" || echo "FAIL: $N khong keo duoc $I"
  done
done
```

Gate điểm khôi phục — chạy trên **PowerShell của máy host Windows**. Đường dẫn `.vmx` theo máy host
đang dùng, sửa lại nếu VM nằm chỗ khác:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '01-cluster-ready') { "PASS: $f" }
  else { "FAIL: $f -> $($names -join ', ')" }
}
```

Chép `~/bookstore-lab-5a.bundle` về máy host bằng `scp` nếu bản trên host chưa phải bản mới nhất.

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; bảy dòng `PASS:` về repo và cluster; `deploy exit=0`,
`smoke exit=0` và file smoke không có dòng `FAIL:`; đủ bốn dòng `PASS: ... co ...`; đúng ba dòng
`PASS:` trên máy host.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 6 **chưa đủ** — sửa repo ở tag `lab-5a`
trước, không sửa tay trên cluster ([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)). Thiếu
snapshot trên bất kỳ VM nào thì **dừng lab**: tắt ba VM, chụp lại mốc theo
[A5.4.8 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot).

## B1. [BS-5B-01] Điều tra: NetworkPolicy của sprint 6 trên Flannel

**Loại:** Điều tra — bằng chứng của nợ lab #4.

**Bối cảnh:** sprint 6 đã viết NetworkPolicy theo ma trận ở
[mục 3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật), và `deploy.sh` đã áp chúng. API server nhận,
`kubectl get` liệt kê đủ — nhưng chưa ai đo xem chúng có chặn gì không. Trước khi đổi CNI, đội cần
**một phép đo chuẩn**: chạy nó bây giờ trên Flannel, rồi chạy lại đúng phép đo đó ở B5 trên Calico.
Hai kết quả ngược nhau chính là nội dung của nợ #4.

**Yêu cầu:**

1. Liệt kê NetworkPolicy của dự án đang có trong `bookstore` và lưu cách Kubernetes diễn giải từng
   cái.
2. Dựng hai Pod đo theo file `~/lab-work/5b/probe-pods.yaml` — một Pod **không mang label nào** trong
   `bookstore`, một Pod ở namespace riêng `np-ngoai`. File chép nguyên văn từ khối Script nghiệm thu.
3. Đo **bảy cặp mà ma trận không cho phép** bằng `~/lab-work/5b/probe.sh` (chép nguyên văn): Pod lạ
   trong `bookstore` tới `web`, `api`, `db`; Pod ở namespace khác tới `web`, `api`, `db`; và `db-0`
   tới `web`. Chứng minh **cả bảy** đều mở được kết nối TCP.
4. Chứng minh đường vào NodePort `30080` của sprint 6 vẫn trả trang chủ — một đường vào đi vòng ngoài
   Traefik mà ma trận không có.
5. Ghi hồ sơ bằng chứng của nợ #4. Xóa Pod đo và namespace đo, **giữ** hai file trong
   `~/lab-work/5b/` cho B5.

**Ràng buộc:** không sửa, không xóa policy; không đổi gì trong repo. Pod đo không mang label
`app.kubernetes.io/part-of`.

**Đọc lại:**

- Bài [84](../84-network-policies-vi.md):
  [*Điều kiện tiên quyết*](../84-network-policies-vi.md#điều-kiện-tiên-quyết-prerequisites),
  [*Hai kiểu cô lập pod*](../84-network-policies-vi.md#hai-kiểu-cô-lập-pod-the-two-sorts-of-pod-isolation),
  [*Tài nguyên NetworkPolicy*](../84-network-policies-vi.md#networkpolicy-resource) — đoạn về
  `podSelector` rỗng và ghi chú "POST nội dung này lên API server sẽ không có tác dụng gì".
- Bài [183](../183-network-plugins-vi.md), đoạn mở đầu và
  [*Yêu cầu đối với Network Plugin*](../183-network-plugins-vi.md#yêu-cầu-đối-với-network-plugin-network-plugin-requirements)
  — hợp đồng của một CNI plugin gồm những gì.
- Bài [201](../201-declare-network-policy-vi.md) — trình tự dựng đích, đo đường cơ sở, áp policy, đo
  lại.

**Bàn giao:** `docs/adr/NNNN-networkpolicy-tren-flannel.md` (`NNNN` là số tiếp theo trong
`docs/adr/`); `~/lab-evidence/5b/b1-*.txt`; `~/lab-work/5b/probe-pods.yaml`, `~/lab-work/5b/probe.sh`.

**Nghiệm thu:**

- `PASS:` API server đang lưu ít nhất một NetworkPolicy mang label dự án trong `bookstore`.
- `PASS:` cả bảy cặp bị cấm đều `THONG` trên Flannel, không cặp nào lỗi DNS.
- `PASS:` NodePort `30080` vẫn trả trang chủ có chữ `Bookstore`.
- `PASS:` đã lưu hồ sơ bằng chứng của nợ #4.
- `PASS:` Pod đo và namespace `np-ngoai` đã bị xóa; hai file đo còn trong `~/lab-work/5b/`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-networkpolicy-tren-flannel.md`:

1. API server nhận và lưu policy, `describe` in đủ rule. Vậy thiếu thứ gì để policy có hiệu lực, và
   thành phần nào trên cluster này lẽ ra phải làm việc đó?
2. Bài 183 liệt kê những gì một CNI plugin **bắt buộc** phải làm. NetworkPolicy có trong danh sách
   không? Flannel có "lỗi" không?
3. Policy của sprint 6 chưa từng chạy thật. **Dự đoán** — ghi lại, chưa sửa: khi CNI bắt đầu thực
   thi, luồng nào **được phép** theo ma trận có thể hỏng vì policy viết thiếu? B5 đối chiếu dự đoán
   này.
4. NodePort `30080` đưa lưu lượng vào `web` không qua Traefik. Ma trận mục 3.5 có cho phép đường đó
   không? Khi policy có hiệu lực, bạn chờ đợi gì ở đường này?

<details>
<summary>Gợi ý</summary>

1. `kubectl -n bookstore describe networkpolicy` in `PodSelector: <none> (Allowing the specific
   traffic to all pods in this namespace)` cho policy có `podSelector` rỗng.
2. Không thấy NetworkPolicy nào sau `deploy.sh`? Sprint 6 có thể mới viết file trong
   `k8s/app/network/` mà chưa đưa vào `k8s/app/kustomization.yaml`. Đó là thiếu sót của tag `lab-5a`:
   sửa, commit, rồi chạy lại B0 — không `kubectl apply` tay.
3. `probe.sh` dùng `bash` với `/dev/tcp` để mở kết nối TCP thuần, nên đo được cả cổng 5432 của DB lẫn
   cổng HTTP. `getent hosts` tách lỗi DNS khỏi lỗi kết nối.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script dựng và xóa Pod đo</summary>

Hai file đo, chép nguyên văn:

```bash
cat > ~/lab-work/5b/probe-pods.yaml <<'EOF'
# Pod đo của Lab 5b — KHÔNG mang label dự án. B1 tạo rồi xóa; B5 tạo lại đúng file này.
apiVersion: v1
kind: Namespace
metadata:
  name: np-ngoai
---
apiVersion: v1
kind: Pod
metadata:
  name: np-thu
  namespace: bookstore
spec:
  terminationGracePeriodSeconds: 1
  containers:
  - name: probe
    image: postgres:17.11
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: np-ngoai
  namespace: np-ngoai
spec:
  terminationGracePeriodSeconds: 1
  containers:
  - name: probe
    image: postgres:17.11
    command: ["sleep", "3600"]
EOF

cat > ~/lab-work/5b/probe.sh <<'EOF'
#!/usr/bin/env bash
# Đo bảy cặp mà ma trận mục 3.5 của file dự án KHÔNG cho phép. Mỗi dòng một kết quả:
#   THONG  mở được kết nối TCP
#   CHAN   tên phân giải được nhưng kết nối không mở trong 5 giây
#   LOIDNS không phân giải được tên — chưa kết luận được gì về policy
set -u
F=bookstore.svc.cluster.local
tcp() {  # $1 namespace  $2 Pod  $3 container  $4 host  $5 cổng
  local X="kubectl -n $1 exec $2 -c $3 --"
  if ! $X getent hosts "$4" >/dev/null 2>&1; then echo "LOIDNS $2 -> $4:$5"; return; fi
  if $X timeout 5 bash -c "exec 3<>/dev/tcp/$4/$5" >/dev/null 2>&1; then
    echo "THONG  $2 -> $4:$5"
  else
    echo "CHAN   $2 -> $4:$5"
  fi
}
tcp bookstore np-thu   probe    "web.$F"      80
tcp bookstore np-thu   probe    "api.$F"      3000
tcp bookstore np-thu   probe    "db-0.db.$F"  5432
tcp np-ngoai  np-ngoai probe    "web.$F"      80
tcp np-ngoai  np-ngoai probe    "api.$F"      3000
tcp np-ngoai  np-ngoai probe    "db-0.db.$F"  5432
tcp bookstore db-0     postgres "web.$F"      80
EOF
chmod +x ~/lab-work/5b/probe.sh
```

Đo:

```bash
NS=bookstore; EV=~/lab-evidence/5b
SEL='app.kubernetes.io/part-of=bookstore'

kubectl -n "$NS" get networkpolicy -o wide > "$EV/b1-networkpolicy.txt"
kubectl -n "$NS" describe networkpolicy >> "$EV/b1-networkpolicy.txt"
N="$(kubectl -n "$NS" get networkpolicy -l "$SEL" --no-headers 2>/dev/null | wc -l)"
[ "$N" -ge 1 ] && echo "PASS: API server dang luu $N NetworkPolicy cua du an"

kubectl apply -f ~/lab-work/5b/probe-pods.yaml
kubectl -n "$NS" wait --for=condition=Ready pod/np-thu --timeout=180s
kubectl -n np-ngoai wait --for=condition=Ready pod/np-ngoai --timeout=180s
~/lab-work/5b/probe.sh > "$EV/b1-probe-flannel.txt"
cat "$EV/b1-probe-flannel.txt"
{ [ "$(grep -c '^THONG' "$EV/b1-probe-flannel.txt")" -eq 7 ] && ! grep -q '^LOIDNS' "$EV/b1-probe-flannel.txt"; } \
  && echo 'PASS: ca bay cap bi cam deu THONG tren Flannel'

W1="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
curl -s --max-time 5 "http://$W1:30080/" | grep -q 'Bookstore' && echo 'PASS: NodePort 30080 van tra trang chu'

{
  date -Is
  echo '--- CNI dang chay'
  kubectl get pods -A -o wide | grep -iE 'flannel|calico' || echo '(khong tim thay)'
  echo '--- NetworkPolicy dang ton tai'
  kubectl -n "$NS" get networkpolicy
  echo '--- Bay cap bi cam, do tren Flannel'
  cat "$EV/b1-probe-flannel.txt"
} > "$EV/b1-bang-chung-no-4.txt"
test -s "$EV/b1-bang-chung-no-4.txt" && echo 'PASS: da luu ho so bang chung no #4'

kubectl delete -f ~/lab-work/5b/probe-pods.yaml --wait=true
{ ! kubectl get namespace np-ngoai >/dev/null 2>&1 && ! kubectl -n "$NS" get pod np-thu >/dev/null 2>&1 \
  && test -f ~/lab-work/5b/probe-pods.yaml && test -x ~/lab-work/5b/probe.sh; } \
  && echo 'PASS: da xoa Pod do, con giu hai file do cho B5'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Ticket không thay đổi repo. Đọc thêm từng policy theo đúng thành phần mà nó chọn:

```bash
kubectl -n bookstore get networkpolicy \
  -o custom-columns='NAME:.metadata.name,SELECTOR:.spec.podSelector,TYPES:.spec.policyTypes'
```

Ý chính cho ADR:

1. Bài 84, *Điều kiện tiên quyết*: NetworkPolicy do **network plugin** hiện thực; tạo tài nguyên mà
   không có controller hiện thực nó thì không có tác dụng gì. API server chỉ kiểm cú pháp và lưu vào
   etcd. Trên cluster này, người đáng lẽ phải dịch policy thành rule trên từng node là CNI — và
   Flannel không làm việc đó.
2. Bài 183: plugin **bắt buộc** hiện thực mô hình mạng Kubernetes, cộng interface `lo` và hỗ trợ
   `hostPort`. NetworkPolicy không nằm trong hợp đồng. Flannel hợp lệ; nó chỉ không phải lựa chọn
   đúng cho yêu cầu `R7` của dự án.
3. Dự đoán điển hình: thiếu đường DNS ra CoreDNS (mọi Pod mất tên), thiếu rule ở một trong hai đầu
   (egress ở `web` hoặc ingress ở `api`), viết cổng `80` của Service thay cho `8080` của Pod, quên Job
   `db-migrate` và CronJob `db-backup` trong danh sách được vào `db`.
4. Không. Ma trận chỉ cho Traefik vào `web`. Bài 84 chỉ miễn cho kết nối từ **node đang chạy Pod**;
   lưu lượng NodePort vào qua một node khác thì không phải từ node đó. Khi policy có hiệu lực, đường
   này có thể chỉ còn chạy khi gói tin rơi đúng node đang chạy Pod `web` — một đường vào không đoán
   trước được và đi vòng qua ma trận. B6 đóng hẳn nó.

</details>

## B2. [BS-5B-02] Điều tra: mạng của cluster trước khi đổi CNI

**Loại:** Điều tra.

**Bối cảnh:** B3 sẽ thay network plugin trên cả ba node. Trước khi thay phải biết chính xác cái đang
chạy: dải nào do ai cấp, file nào cấu hình plugin, ai nạp nó. Đây vừa là bằng chứng "trước" để so với
"sau" ở B3, vừa là dữ liệu B3 và B7 cần.

**Yêu cầu:**

1. Đọc ba dải IP từ **tiến trình đang chạy**, không từ tài liệu: dải Pod, dải Service, IP của node.
   Đối chiếu với cấu hình kubeadm lưu trong cluster. Ghi dải Pod và ClusterIP của Service DNS ra file
   cho B3, B7.
2. Trên `lab-k8s-worker2`: số địa chỉ IPv4 trên các interface so với số địa chỉ Kubernetes ghi nhận
   cho node đó.
3. PodCIDR của từng node.
4. Trên **cả ba** node: thư mục cấu hình CNI, thư mục binary, file conflist của plugin đang chạy —
   plugin chính, khai báo `portMappings`, binary `loopback` và `portmap`.
5. Chứng minh **ai** nạp plugin: thành phần trỏ tới hai thư mục trên, và kubelet không còn cờ quản lý
   CNI.
6. Chọn CNI thay thế từ danh sách provider chính thức, ghi lý do vào ADR.

**Ràng buộc:** chỉ đọc; không sửa file nào trên node.

**Đọc lại:**

- Bài [157](../157-networking-vi.md):
  [*Các dải địa chỉ IP của Kubernetes*](../157-networking-vi.md#các-dải-địa-chỉ-ip-của-kubernetes-kubernetes-ip-address-ranges),
  [*Các kiểu mạng cluster*](../157-networking-vi.md#cluster-network-ipfamilies) — đoạn về
  `node.status.addresses`,
  [*Cách hiện thực mô hình mạng*](../157-networking-vi.md#how-to-implement-the-kubernetes-network-model).
- Bài [183](../183-network-plugins-vi.md):
  [*Cài đặt*](../183-network-plugins-vi.md#cài-đặt-installation) — ghi chú về Kubernetes 1.24;
  [*Loopback CNI*](../183-network-plugins-vi.md#loopback-cni);
  [*Hỗ trợ hostPort*](../183-network-plugins-vi.md#hỗ-trợ-hostport-support-hostport).
- Bài [243](../243-network-policy-provider-vi.md) — danh sách provider; bài
  [245](../245-calico-network-policy-vi.md#creating-a-local-calico-cluster-with-kubeadm) — nhánh
  kubeadm.

**Bàn giao:** `docs/adr/NNNN-mang-cluster-va-chon-cni.md`; `~/lab-evidence/5b/b2-*.txt` — trong đó
`b2-pod-cidr.txt` dùng ở B3, `b2-dns-clusterip.txt` dùng ở B7.

**Nghiệm thu:**

- `PASS:` đọc đủ ba dải từ tiến trình đang chạy; khớp cấu hình kubeadm; ba dải không chung tiền tố.
- `PASS:` `lab-k8s-worker2` có nhiều IPv4 trên interface hơn số InternalIP mà Kubernetes ghi nhận.
- `PASS:` ba node có ba PodCIDR khác nhau.
- `PASS:` trên cả ba node: conflist của Flannel, plugin chính `flannel`, có `portMappings`, có binary
  `loopback` và `portmap`.
- `PASS:` containerd trỏ tới hai thư mục CNI; kubelet không còn cờ `--cni-bin-dir`, `--network-plugin`.
- `PASS:` đã lưu ClusterIP của Service DNS và bản conflist cũ.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-mang-cluster-va-chon-cni.md`:

1. Ba dải, ba nơi cấp. Khi đổi network plugin, dải nào đổi người cấp, dải nào không? Hệ quả với Pod
   đang chạy và với ClusterIP?
2. `ip addr` trên worker2 cho nhiều địa chỉ hơn `node.status.addresses`. Kubernetes dựa vào cái nào
   khi hiện thực mô hình mạng?
3. Ai nạp CNI plugin trên node? Vậy ở B3, sau khi dọn cấu hình CNI cũ, phải khởi động lại tiến trình
   nào trước?
4. Chọn provider nào trong danh sách của bài 243, và vì sao? Nêu tiêu chí, không chỉ nêu tên.

<details>
<summary>Gợi ý</summary>

1. Static Pod `kube-apiserver-lab-k8s-master` và `kube-controller-manager-lab-k8s-master` mang cờ
   dòng lệnh trong `.spec.containers[0].command`. Tìm `--service-cluster-ip-range` và `--cluster-cidr`.
2. Cấu hình kubeadm nằm trong ConfigMap `kubeadm-config` của `kube-system`.
3. Thành phần nạp plugin in cấu hình của nó bằng `containerd config dump`.
4. Cờ đã bị gỡ thì không còn trong `kubelet --help`.

</details>

<details>
<summary>Script nghiệm thu — chỉ đọc</summary>

```bash
EV=~/lab-evidence/5b
MASTER=lab-k8s-master
on() { if [ "$1" = "$(hostname)" ]; then bash -c "$2"; else ssh "$1" "$2"; fi; }
args() { kubectl -n kube-system get pod "$1" -o jsonpath='{range .spec.containers[0].command[*]}{@}{"\n"}{end}'; }

# 1. Ba dải IP
SVC_CIDR="$(args "kube-apiserver-$MASTER" | sed -n 's/^--service-cluster-ip-range=//p')"
POD_CIDR="$(args "kube-controller-manager-$MASTER" | sed -n 's/^--cluster-cidr=//p')"
NODE_IP="$(kubectl get node "$MASTER" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
printf '%s\n' "$POD_CIDR" > "$EV/b2-pod-cidr.txt"
printf '%s\n' "$SVC_CIDR" > "$EV/b2-svc-cidr.txt"
echo "pod=$POD_CIDR svc=$SVC_CIDR node=$NODE_IP"
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' \
  | grep -E 'podSubnet|serviceSubnet' > "$EV/b2-kubeadm-config.txt"
P="$(echo "$POD_CIDR" | cut -d. -f1,2)"; S="$(echo "$SVC_CIDR" | cut -d. -f1,2)"; H="$(echo "$NODE_IP" | cut -d. -f1,2)"
{ [ -n "$POD_CIDR" ] && [ -n "$SVC_CIDR" ] && [ -n "$NODE_IP" ] \
  && grep -qF "$POD_CIDR" "$EV/b2-kubeadm-config.txt" && grep -qF "$SVC_CIDR" "$EV/b2-kubeadm-config.txt" \
  && [ "$P" != "$S" ] && [ "$P" != "$H" ] && [ "$S" != "$H" ]; } \
  && echo 'PASS: ba dai doc tu tien trinh, khop kubeadm-config, khong chung tien to'

# 2. ip addr so với node.status.addresses
on lab-k8s-worker2 'ip -br a' > "$EV/b2-worker2-ip.txt"
NIP="$(on lab-k8s-worker2 'hostname -I' | wc -w)"
NA="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}' | wc -w)"
{ [ "$NIP" -gt 1 ] && [ "$NA" -eq 1 ]; } \
  && echo "PASS: worker2 co $NIP IPv4 tren interface, Kubernetes chi ghi nhan $NA InternalIP"

# 3. PodCIDR từng node
kubectl get nodes -o custom-columns='NODE:.metadata.name,PODCIDR:.spec.podCIDR' > "$EV/b2-podcidr.txt"
[ "$(awk 'NR>1 {print $2}' "$EV/b2-podcidr.txt" | sort -u | wc -l)" -eq 3 ] && echo 'PASS: ba node co ba PodCIDR khac nhau'

# 4. CNI trên cả ba node
BAD=0
for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  on "$N" 'test -f /etc/cni/net.d/10-flannel.conflist && test -x /opt/cni/bin/loopback \
    && test -x /opt/cni/bin/portmap \
    && sudo grep -q "\"type\": \"flannel\"" /etc/cni/net.d/10-flannel.conflist \
    && sudo grep -q "\"portMappings\": true" /etc/cni/net.d/10-flannel.conflist' || { echo "FAIL: $N"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: ca ba node: conflist Flannel, plugin flannel, portMappings, loopback, portmap'

# 5. Ai nạp plugin
BAD=0
for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  on "$N" 'sudo containerd config dump | grep -A6 -iE "\.cni\]"' > "$EV/b2-containerd-cni-$N.txt"
  { grep -q '/opt/cni/bin' "$EV/b2-containerd-cni-$N.txt" && grep -q '/etc/cni/net.d' "$EV/b2-containerd-cni-$N.txt"; } || BAD=1
  on "$N" 'kubelet --help 2>&1' | grep -qE -- '--(cni-bin-dir|network-plugin)' && BAD=1
done
[ "$BAD" -eq 0 ] && echo 'PASS: containerd tro toi hai thu muc CNI; kubelet khong con co quan ly CNI'

# 6. Dữ liệu cho B3, B7
kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.clusterIP}' > "$EV/b2-dns-clusterip.txt"
{ date -Is; echo '--- /etc/cni/net.d cua lab-k8s-master'; ls -l /etc/cni/net.d/
  echo '--- conflist'; sudo cat /etc/cni/net.d/10-flannel.conflist; } > "$EV/b2-cni-truoc.txt"
{ test -s "$EV/b2-dns-clusterip.txt" && test -s "$EV/b2-cni-truoc.txt"; } \
  && echo 'PASS: da luu ClusterIP cua DNS va conflist cu'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính cho ADR</summary>

1. Network plugin cấp IP **Pod**; kube-apiserver cấp IP **Service**; kubelet cấp IP **Node** (bài
   157). Đổi plugin là đổi người cấp IP Pod: Pod đang chạy giữ IP của plugin cũ cho tới khi bị tạo
   lại. ClusterIP không đổi vì apiserver không liên quan tới CNI — B7 kiểm lại bằng ClusterIP của DNS.
2. Chỉ `node.status.addresses` (và `pod.status.ips` với Pod). Interface `cni0`, `flannel.1`, `lo`
   đều có địa chỉ nhưng không phải thứ Kubernetes xét. Nhìn `ip addr` để gỡ lỗi thì được, suy ra
   "Kubernetes đang dùng địa chỉ này" thì sai.
3. **containerd** — container runtime cung cấp CRI. Từ 1.24, `cni-bin-dir` và `network-plugin` bị gỡ
   khỏi kubelet (bài 183, *Cài đặt*). Dọn cấu hình xong phải khởi động lại containerd trước để nó nạp
   lại thư mục CNI, rồi mới tới kubelet.
4. Calico: nằm trong danh sách chính thức của bài 243; trang 245 có nhánh *cluster cục bộ với
   kubeadm*; cài bằng **một manifest** giống hệt hình dạng thao tác bạn đã làm với Flannel ở
   [A5.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a52-cài-flannel-v0289); version đã chốt ở A1.4. Ít
   bước lạ thì phần học nằm ở NetworkPolicy chứ không nằm ở trình cài đặt.

</details>

## B3. [BS-5B-03] Vận hành: gỡ dự án, đổi CNI sang Calico

**Loại:** Vận hành — chạm cả ba node. Đọc lại [mục 2.1](#21-đổi-cni-là-thao-tác-phá-hoại) trước khi gõ
lệnh đầu tiên.

**Bối cảnh:** yêu cầu `R7` của dự án — mạng zero-trust giữa các tầng — cần một CNI thực thi
NetworkPolicy. Thay plugin là việc của **hạ tầng nền**: nó làm ra mốc `02-net-ready` cho mọi lab sau,
và Lab 8a sẽ phải làm lại nó trên một cluster dựng mới. Vì vậy ngoài việc đổi plugin, đội phải để lại
một runbook trong repo.

**Yêu cầu:**

1. `platform/versions.env` chứa bốn giá trị chép đúng từ A1.4: version Calico, version Helm, version
   chart Traefik và `appVersion` của Traefik. Mẫu file ở khối Lời giải; giá trị là phần bạn tự điền.
2. Dự án được gỡ **trước** khi chạm vào mạng; cluster không còn object nào của dự án.
3. Sau ticket: không còn object, Pod, file cấu hình hay interface nào của Flannel trên cả ba node;
   mỗi node chỉ còn cấu hình CNI của Calico.
4. Calico chạy đúng một Pod trên mỗi node, đúng version trong `versions.env`; ba node `Ready`.
5. Dải IP mà Calico cấp **trùng dải Pod** đọc ở B2; Calico chọn đúng InternalIP của từng node.
6. Không còn Pod nào mang IP do plugin cũ cấp; Pod trên hai worker nói chuyện được với nhau; DNS trong
   cluster và ra Internet chạy.
7. `platform/calico/README.md` là runbook của bạn, gồm hai nhánh: cài mới trên cluster chưa có CNI
   (Lab 8a sẽ dùng) và thay Flannel trên cluster đang chạy (ticket này), kèm các gate.

**Ràng buộc:** manifest Calico tải đúng version trong `versions.env`. Không sửa IPPool bằng tay khi Pod
đã lấy IP. Không restore riêng một VM. Pod kiểm tra mạng nằm ở namespace `net-thu`, xóa khi xong.

**Đọc lại:**

- Bài [183](../183-network-plugins-vi.md#cài-đặt-installation) — ai nạp plugin; hai thư mục.
- Bài [157](../157-networking-vi.md#các-dải-địa-chỉ-ip-của-kubernetes-kubernetes-ip-address-ranges) —
  ai cấp IP Pod.
- Bài [245](../245-calico-network-policy-vi.md#creating-a-local-calico-cluster-with-kubeadm) — nhánh
  kubeadm trỏ sang tài liệu cài của Calico; cách xác minh bằng Pod tên `calico`.
- Calico: [Install on on-premises deployments](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises),
  tab *Manifest* — với kubeadm, Pod CIDR khác mặc định thì không cần sửa gì vì Calico tự dò. Lab
  không tin lời đó: có gate đối chiếu.
- Calico: [System requirements](https://docs.tigera.io/calico/3.32/getting-started/kubernetes/requirements),
  mục *Other network providers* — vì sao B3 chỉ là thí nghiệm ([mục 2.1](#21-đổi-cni-là-thao-tác-phá-hoại)).
- [A5.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a52-cài-flannel-v0289) — hình dạng thao tác cài một
  CNI bằng manifest.

**Bàn giao:** `platform/versions.env`, `platform/calico/README.md`,
`docs/adr/NNNN-quy-trinh-doi-cni.md`; `~/lab-evidence/5b/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` `platform/versions.env` có đủ bốn giá trị, đúng dạng.
- `PASS:` không còn object nào của dự án trên cluster.
- `PASS:` không còn namespace, ClusterRole, Pod nào của Flannel.
- `PASS:` trên cả ba node: không còn conflist và interface của Flannel; có conflist của Calico với
  plugin `calico`, IPAM `calico-ipam`, `portMappings`.
- `PASS:` `calico-node` đúng ba Pod `Running`, đúng version; `calico-kube-controllers` sẵn sàng; ba
  node `Ready`.
- `PASS:` IPPool trùng dải Pod; địa chỉ Calico chọn cho từng node là InternalIP của node đó; IPAM tự
  chia khối cho các node.
- `PASS:` mọi Pod dùng mạng Pod đều do Calico cấp IP.
- `PASS:` TCP xuyên node, DNS trong cluster và DNS ra Internet đều chạy.
- `PASS:` runbook `platform/calico/README.md` có mặt và đọc version từ `versions.env`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-quy-trinh-doi-cni.md`:

1. Bốn bước của quy trình đổi CNI theo đúng thứ tự. Đảo bước gỡ ở tầng API với bước dọn node thì
   hỏng gì? Bỏ bước tạo lại Pod cũ thì hỏng gì?
2. Vì sao gỡ dự án **trước**, thay vì đổi CNI ngay dưới chân hệ thống đang chạy?
3. `.spec.podCIDR` của từng node không đổi, nhưng IP Pod mới không nhất thiết nằm trong `/24` đó. Có
   mâu thuẫn với bài 157 không?
4. Đường lùi khi B3 hỏng giữa chừng là gì, và vì sao không restore riêng một VM?
5. Calico nói gì về việc chuyển từ CNI khác sang nó? Vậy gate của B3 PASS chứng minh được điều gì,
   không chứng minh được điều gì — và với một cluster thật cần Calico, bạn chọn đường nào?

<details>
<summary>Gợi ý</summary>

1. Thứ tự an toàn: (1) gỡ Flannel ở tầng API — namespace `kube-flannel`, ClusterRole và
   ClusterRoleBinding `flannel`; (2) dọn từng node — conflist, thư mục trạng thái, interface
   `flannel.1` và `cni0`, khởi động lại containerd rồi kubelet; (3) cài Calico; (4) tạo lại Pod còn
   mang IP cũ. Sau khi gỡ dự án, Pod như vậy chỉ còn CoreDNS.
2. Đọc trước khi apply: `grep` dòng image `calico/node` và biến `CALICO_IPV4POOL_CIDR` trong manifest.
3. Calico lưu cấu hình trong CRD: `ippools.crd.projectcalico.org`, `ipamblocks.crd.projectcalico.org`;
   ghi địa chỉ chọn cho node vào annotation `projectcalico.org/IPv4Address` của Node, và ghi IP đã
   cấp vào annotation `cni.projectcalico.org/podIP` của từng Pod.
4. Thời gian rollout phụ thuộc tốc độ kéo image; con số `--timeout` chỉ là trần chờ, không phải cam
   kết.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
EV=~/lab-evidence/5b
SEL='app.kubernetes.io/part-of=bookstore'
on() { if [ "$1" = "$(hostname)" ]; then bash -c "$2"; else ssh "$1" "$2"; fi; }
. ~/bookstore/platform/versions.env
POD_CIDR="$(cat "$EV/b2-pod-cidr.txt")"

# 0. Phiên bản
BAD=0
for V in "${CALICO_VER:-}" "${HELM_VER:-}" "${TRAEFIK_APP:-}"; do echo "$V" | grep -Eq '^v[0-9]+\.[0-9]+\.[0-9]+$' || BAD=1; done
echo "${TRAEFIK_CHART:-}" | grep -Eq '^[0-9]+\.[0-9]+\.[0-9]+$' || BAD=1
[ "$BAD" -eq 0 ] && echo 'PASS: platform/versions.env du bon gia tri, dung dang'

# 1. Dự án đã gỡ
{ [ "$(sl kubectl get namespace,priorityclass -l "$SEL" --no-headers)" -eq 0 ] && \
  [ "$(sl kubectl get all -A -l "$SEL" --no-headers)" -eq 0 ]; } && echo 'PASS: khong con object cua du an'

# 2. Flannel ở tầng API
{ ! kubectl get namespace kube-flannel >/dev/null 2>&1 && ! kubectl get clusterrole flannel >/dev/null 2>&1 \
  && ! kubectl get pods -A --no-headers | grep -q flannel; } && echo 'PASS: khong con namespace, ClusterRole, Pod cua Flannel'

# 3. Từng node
BAD=0
for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  on "$N" 'test ! -e /etc/cni/net.d/10-flannel.conflist && test -f /etc/cni/net.d/10-calico.conflist \
    && ! ip link show flannel.1 >/dev/null 2>&1 && ! ip link show cni0 >/dev/null 2>&1 \
    && sudo grep -q "\"type\": \"calico\"" /etc/cni/net.d/10-calico.conflist \
    && sudo grep -q "calico-ipam" /etc/cni/net.d/10-calico.conflist \
    && sudo grep -q "\"portMappings\": true" /etc/cni/net.d/10-calico.conflist' || { echo "FAIL: $N"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: ca ba node chi con cau hinh CNI cua Calico, khong con interface cu'
on lab-k8s-master 'ls -l /etc/cni/net.d/; sudo cat /etc/cni/net.d/10-calico.conflist' > "$EV/b3-cni-sau.txt"

# 4. Calico chạy đủ, đúng version
kubectl -n kube-system get ds calico-node \
  -o jsonpath='{range .spec.template.spec.initContainers[*]}{.image}{"\n"}{end}{range .spec.template.spec.containers[*]}{.image}{"\n"}{end}' \
  > "$EV/b3-calico-images.txt"
{ [ "$(kubectl -n kube-system get pods -l k8s-app=calico-node --field-selector=status.phase=Running --no-headers | wc -l)" -eq 3 ] \
  && grep -q "calico/node:$CALICO_VER" "$EV/b3-calico-images.txt" \
  && kubectl -n kube-system wait --for=condition=Available deployment/calico-kube-controllers --timeout=60s >/dev/null \
  && kubectl wait --for=condition=Ready node --all --timeout=60s >/dev/null; } \
  && echo "PASS: calico-node 3/3 Running, dung $CALICO_VER; calico-kube-controllers san sang; ba node Ready"

# 5. Dải IP và địa chỉ node
IPPOOL="$(kubectl get ippools.crd.projectcalico.org default-ipv4-ippool -o jsonpath='{.spec.cidr}')"
BAD=0
for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  A="$(kubectl get node "$N" -o jsonpath='{.metadata.annotations.projectcalico\.org/IPv4Address}')"
  I="$(kubectl get node "$N" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
  case "$A" in "$I"/*) ;; *) echo "FAIL: $N - Calico chon $A, InternalIP la $I"; BAD=1 ;; esac
done
kubectl get ipamblocks.crd.projectcalico.org -o custom-columns='BLOCK:.spec.cidr,AFFINITY:.spec.affinity' \
  > "$EV/b3-ipamblock.txt"
{ [ "$IPPOOL" = "$POD_CIDR" ] && [ "$BAD" -eq 0 ] && [ "$(awk 'NR>1' "$EV/b3-ipamblock.txt" | wc -l)" -ge 2 ]; } \
  && echo "PASS: IPPool $IPPOOL trung dai Pod; Calico chon dung InternalIP; IPAM chia khoi theo node"

# 6. Không còn Pod mang IP của plugin cũ: Pod dùng mạng Pod phải có annotation do Calico CNI ghi
OLD="$(kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name} {.status.podIP} {.status.hostIP} {.metadata.annotations.cni\.projectcalico\.org/podIP}{"\n"}{end}' \
  | awk '$2!=$3 && $4==""')"
[ -z "$OLD" ] && echo 'PASS: moi Pod dung mang Pod deu do Calico cap IP' || echo "FAIL: con Pod mang IP cu: $OLD"

# 7. Mạng liền: TCP xuyên node, DNS trong cluster và ra ngoài
cat > ~/lab-work/5b/net-thu.yaml <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: net-thu
---
apiVersion: v1
kind: Pod
metadata:
  name: net-w1
  namespace: net-thu
spec:
  nodeName: lab-k8s-worker1
  terminationGracePeriodSeconds: 1
  containers:
  - name: shell
    image: busybox:1.37
    command: ["sleep", "3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: net-w2
  namespace: net-thu
spec:
  nodeName: lab-k8s-worker2
  terminationGracePeriodSeconds: 1
  containers:
  - name: httpd
    image: busybox:1.37
    command: ["sh", "-c", "mkdir -p /www && echo net-w2-ok > /www/index.html && httpd -f -p 8080 -h /www"]
EOF
kubectl apply -f ~/lab-work/5b/net-thu.yaml
kubectl -n net-thu wait --for=condition=Ready pod/net-w1 pod/net-w2 --timeout=180s
IP2="$(kubectl -n net-thu get pod net-w2 -o jsonpath='{.status.podIP}')"
KIP="$(kubectl get svc kubernetes -o jsonpath='{.spec.clusterIP}')"
{ kubectl -n net-thu exec net-w1 -- wget -qO- -T 5 "http://$IP2:8080" | grep -qx 'net-w2-ok' \
  && kubectl -n net-thu exec net-w1 -- nslookup kubernetes.default.svc.cluster.local | grep -q "$KIP" \
  && kubectl -n net-thu exec net-w1 -- nslookup registry.k8s.io >/dev/null; } \
  && echo 'PASS: TCP xuyen node, DNS trong cluster va ra Internet deu chay'
kubectl get pods -A -o wide > "$EV/b3-pods-sau-doi-cni.txt"
kubectl delete -f ~/lab-work/5b/net-thu.yaml --wait=true

# 8. Runbook
grep -q 'CALICO_VER' ~/bookstore/platform/calico/README.md && echo 'PASS: runbook Calico co mat va doc version tu versions.env'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Mẫu `platform/versions.env` — chép vào repo, thay bốn chỗ `<...>` bằng đúng giá trị ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00):

```bash
# platform/versions.env — phiên bản hạ tầng nền của cluster lab.
# Nguồn duy nhất: bảng A1.4 của Lab 00. Sửa ở A1.4 trước, rồi mới sửa file này.
CALICO_VER='<Calico, dạng vX.Y.Z>'
HELM_VER='<Helm, dạng vX.Y.Z>'
TRAEFIK_CHART='<chart Traefik, dạng X.Y.Z>'
TRAEFIK_APP='<appVersion Traefik, dạng vX.Y.Z>'
```

Quy trình thí nghiệm, chạy trên `lab-k8s-master` trong một phiên shell. Đây **không** phải quy trình
Calico hỗ trợ ([mục 2.1](#21-đổi-cni-là-thao-tác-phá-hoại)) và chưa được pilot trọn trên đúng
baseline: mọi gate của B3 là bắt buộc, đường lùi duy nhất là restore cả ba VM về `01-cluster-ready`.

```bash
. ~/bookstore/platform/versions.env
on() { if [ "$1" = "$(hostname)" ]; then bash -c "$2"; else ssh "$1" "$2"; fi; }

# 0. Gỡ dự án trước khi chạm vào mạng
cd ~/bookstore && ./scripts/teardown.sh

# 1. Tải manifest, đọc trước khi apply
curl -fsSL "https://raw.githubusercontent.com/projectcalico/calico/$CALICO_VER/manifests/calico.yaml" \
  -o ~/lab-work/5b/calico.yaml
grep -q "image: .*calico/node:$CALICO_VER" ~/lab-work/5b/calico.yaml && echo 'PASS: manifest dung version'
grep -E '^\s*#.*CALICO_IPV4POOL_CIDR' ~/lab-work/5b/calico.yaml >/dev/null \
  && echo 'PASS: CALICO_IPV4POOL_CIDR dang bi comment - Calico tu do dai Pod cua kubeadm'

# 2. Gỡ Flannel ở tầng API — trước khi dọn node, nếu không DaemonSet cũ ghi lại conflist
kubectl delete namespace kube-flannel --wait=true --timeout=300s
kubectl delete clusterrole flannel --ignore-not-found
kubectl delete clusterrolebinding flannel --ignore-not-found

# 3. Dọn từng node: worker2 → worker1 → master. containerd nạp plugin nên khởi động lại trước kubelet
for N in lab-k8s-worker2 lab-k8s-worker1 lab-k8s-master; do
  on "$N" 'sudo rm -f /etc/cni/net.d/10-flannel.conflist
           sudo rm -rf /var/lib/cni/flannel /var/lib/cni/networks /run/flannel
           for i in flannel.1 cni0; do ip link show "$i" >/dev/null 2>&1 && sudo ip link delete "$i"; done
           sudo systemctl restart containerd
           sudo systemctl restart kubelet'
done

# 4. Cài Calico
kubectl apply -f ~/lab-work/5b/calico.yaml
kubectl -n kube-system rollout status daemonset/calico-node --timeout=600s
kubectl -n kube-system rollout status deployment/calico-kube-controllers --timeout=600s
kubectl wait --for=condition=Ready node --all --timeout=300s

# 5. Tạo lại Pod còn mang IP cũ — sau khi gỡ dự án chỉ còn CoreDNS
kubectl -n kube-system delete pod -l k8s-app=kube-dns
kubectl -n kube-system rollout status deployment/coredns --timeout=300s
```

`platform/calico/README.md` — khung tối thiểu; nội dung lệnh là khối trên, viết bằng lời của bạn:

```markdown
# Calico — CNI có thực thi NetworkPolicy

Phiên bản: biến CALICO_VER trong platform/versions.env (nguồn: bảng A1.4 của Lab 00).
Cài bằng một manifest; mọi object nằm trong kube-system. Với kubeadm, không sửa CALICO_IPV4POOL_CIDR.

## A. Cluster mới, chưa có CNI (Lab 8a)
tải manifest → đọc image và CALICO_IPV4POOL_CIDR → apply → chờ calico-node, calico-kube-controllers
→ gate C.

## B. Thay Flannel trên cluster đang chạy (Lab 5b) — chỉ là thí nghiệm
Calico không hỗ trợ migrate từ CNI khác (System requirements, mục Other network providers). Chỉ chạy
trên môi trường hủy được; cluster thật cần Calico thì dựng mới theo nhánh A.
0. teardown dự án  1. gỡ Flannel ở tầng API  2. dọn từng node, containerd trước kubelet
3. apply Calico  4. tạo lại Pod còn IP cũ (CoreDNS)  → gate C.
Đường lùi: restore cả ba VM về 01-cluster-ready.

## C. Gate
IPPool = dải Pod của kubeadm; annotation projectcalico.org/IPv4Address = InternalIP;
Pod dùng mạng Pod có annotation cni.projectcalico.org/podIP; TCP xuyên node; DNS trong và ngoài.
```

Ý chính cho ADR:

1. Gỡ API → dọn node → cài mới → tạo lại Pod cũ. Dọn node trước khi gỡ API thì Pod Flannel còn sống
   ghi lại conflist vừa xóa, node có **hai** cấu hình CNI và containerd chọn theo thứ tự tên — một cái
   bẫy im lặng. Bỏ bước cuối thì CoreDNS giữ IP của dải cũ, không ai định tuyến tới nó: DNS chết.
2. Pod đang chạy giữ IP của plugin cũ cho tới khi bị tạo lại; `db` còn dữ liệu trên `emptyDir`
   (`TD-04`) sẽ mất khi Pod bị thay. Dự án sống trong repo, dựng lại chỉ tốn một lần `deploy.sh` — rẻ
   hơn nhiều so với gỡ rối một hệ thống đang nằm giữa hai plugin.
3. Không. `.spec.podCIDR` do kube-controller-manager cấp và vẫn nguyên. Bài 157 chỉ nói **network
   plugin** cấp IP Pod; nó không bắt plugin dùng `.spec.podCIDR`. Calico có IPAM riêng, tự chia dải
   lớn thành khối nhỏ gán cho từng node.
4. Restore **cả ba VM** về `01-cluster-ready` rồi clone repo từ bundle. Với cluster một control plane,
   etcd trên master và trạng thái mạng trên worker phải cùng một mốc; restore lệch là có hai thế giới.
5. Calico nói không migrate được từ CNI khác. Gate PASS chứng minh **cluster lab này, lần này** đã về
   trạng thái mạng đúng — không chứng minh quy trình an toàn ở cluster khác hay lần chạy khác, và không
   biến nó thành quy trình nhà phát hành bảo đảm. Cluster thật cần Calico thì dựng mới với Calico từ đầu
   (nhánh A, Lab 8a) rồi chuyển workload sang; dữ liệu và manifest nằm ngoài cluster nên làm được.

</details>

## B4. [BS-5B-04] Vận hành: Helm, Traefik và IngressClass mặc định

**Loại:** Vận hành.

**Bối cảnh:** yêu cầu `R7` cũng nói truy cập từ ngoài phải qua HTTPS. Sprint 6 mở `web` bằng NodePort
— cổng TCP thô, không hiểu host hay path, không kết thúc TLS. Cluster cần một **ingress controller**.
Đây là chart đầu tiên của chuỗi lab, nên cũng là lúc Helm vào cluster, đúng ghi chú ở A1.4.

**Yêu cầu:**

1. **Trước khi cài controller**, chứng minh bằng Ingress thử nghiệm `som` ở namespace `ing-thu` (file
   chép nguyên văn ở khối Script nghiệm thu): API server nhận Ingress, nhưng cluster không có
   IngressClass nào, Ingress không được gán class, `.status` rỗng, và không có gì nghe ở cổng NodePort
   mà controller sẽ dùng.
2. Helm trên `lab-k8s-master`, đúng version trong `versions.env`, binary đã kiểm checksum do nhà phát
   hành công bố.
3. Traefik: release `traefik`, namespace `traefik`, chart đúng version, image đúng `appVersion`. Mọi
   cấu hình nằm trong `platform/traefik/values.yaml`; không sửa tay object của chart sau khi cài.
4. Service của Traefik là NodePort **cố định**: `30081` cho HTTP, `30443` cho HTTPS.
5. Cluster có đúng **một** IngressClass, tên `traefik`, được đánh dấu mặc định.
6. Ingress `som` tạo ở yêu cầu 1 vẫn không có class sau khi cài. Tạo lại đúng file đó thì được gán
   `traefik` và phục vụ được qua NodePort HTTP với đúng header `Host`.
7. `platform/traefik/README.md`: cách cài, cách kiểm, và vì sao không cài thêm load balancer.

**Ràng buộc:** không cài MetalLB (nằm ngoài baseline), không cài Gateway API. Không `kubectl patch`,
`edit` object của chart — Helm và file values là nguồn sự thật. Namespace `ing-thu` không mang label
dự án; **giữ** tới B7, B8, xóa ở mục đóng sprint.

**Đọc lại:**

- Bài [12](../12-ingress-controllers-vi.md): đoạn mở đầu;
  [*Các ingress controller của bên thứ ba*](../12-ingress-controllers-vi.md#các-ingress-controller-của-bên-thứ-ba-third-party-ingress-controllers);
  [*Sử dụng nhiều Ingress controller*](../12-ingress-controllers-vi.md#sử-dụng-nhiều-ingress-controller-using-multiple-ingress-controllers).
- Bài [11](../11-ingress-vi.md): [*Điều kiện tiên quyết*](../11-ingress-vi.md#điều-kiện-tiên-quyết-prerequisites),
  [*Ingress class*](../11-ingress-vi.md#ingress-class),
  [*IngressClass mặc định*](../11-ingress-vi.md#default-ingress-class).
- Bài [164](../164-proxies-vi.md#proxy-proxies) — loại 5, cloud load balancer.
- Bài [82](../82-service-vi.md): [`type: NodePort`](../82-service-vi.md#type-nodeport),
  [*Tự chọn port*](../82-service-vi.md#nodeport-custom-port), [`type: LoadBalancer`](../82-service-vi.md#loadbalancer).
- [Traefik Helm chart](https://github.com/traefik/traefik-helm-chart) — file `values.yaml` của đúng
  version chart; [Helm — Installing Helm](https://helm.sh/docs/intro/install/).

**Bàn giao:** `platform/traefik/values.yaml`, `platform/traefik/README.md`;
`docs/adr/NNNN-ingress-controller.md`; `~/lab-work/5b/ing-thu.yaml`; `~/lab-evidence/5b/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` trước khi cài: không có IngressClass; Ingress `som` được lưu, không class, `.status` rỗng;
  cổng `30081` không trả lời.
- `PASS:` `helm` đúng version.
- `PASS:` release `traefik` đã triển khai, đúng chart; image đúng `appVersion`; Deployment sẵn sàng.
- `PASS:` chart mặc định tạo Service kiểu `LoadBalancer`; Service đang chạy là `NodePort` `30081`,
  `30443`.
- `PASS:` đúng một IngressClass, tên `traefik`, mặc định.
- `PASS:` Ingress cũ vẫn không class; tạo lại thì được gán `traefik`; `som.lab.local` trả `som-ok`.
- `PASS:` `platform/traefik/` có đủ hai file.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-ingress-controller.md`:

1. Ingress `som` hợp lệ, được lưu, `describe` in đủ rule — vì sao trước khi cài không chuyển một byte
   nào? So với NetworkPolicy ở B1: cùng hình dạng vấn đề hay khác?
2. Cài controller xong, `som` vẫn không có class. Cơ chế nào điền class khi tạo lại? Vì sao Ingress
   của dự án ở B6 nên ghi `ingressClassName` tường minh thay vì dựa vào class mặc định?
3. Chart mặc định Service `LoadBalancer`. Trên cluster này nó sẽ ra sao, và đó là loại proxy nào của
   bài 164? Vì sao chọn NodePort cố định thay vì cài một load balancer?
4. Bài 12 nói Traefik là controller **bên thứ ba**. Điều đó đổi trách nhiệm của đội vận hành ở đâu?

<details>
<summary>Gợi ý</summary>

1. Bản phát hành Helm có file `.sha256sum` đi kèm; `sha256sum -c` kiểm nó.
2. `helm show values traefik/traefik --version "$TRAEFIK_CHART"` in toàn bộ giá trị mặc định của
   chart. Tìm khối `service`, khối `ports.web`, `ports.websecure` (có khóa `nodePort`) và khối
   `ingressClass`.
3. Class mặc định là annotation `ingressclass.kubernetes.io/is-default-class: "true"` trên
   IngressClass.
4. Tạo lại chỉ Ingress: xóa đúng object `ingress/som` rồi apply lại cả file — các object khác trong file
   không đổi.

</details>

<details>
<summary>Script nghiệm thu — phần 1 chạy TRƯỚC khi cài Traefik, phần 2 sau</summary>

File thử nghiệm, chép nguyên văn:

```bash
cat > ~/lab-work/5b/ing-thu.yaml <<'EOF'
# Đích thử nghiệm của B4, B7 — KHÔNG mang label dự án.
apiVersion: v1
kind: Namespace
metadata:
  name: ing-thu
---
apiVersion: v1
kind: Pod
metadata:
  name: som
  namespace: ing-thu
  labels:
    app: som
spec:
  terminationGracePeriodSeconds: 1
  containers:
  - name: httpd
    image: busybox:1.37
    command: ["sh", "-c", "mkdir -p /www && echo som-ok > /www/index.html && httpd -f -p 8080 -h /www"]
    ports:
    - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: som
  namespace: ing-thu
spec:
  selector:
    app: som
  ports:
  - name: http
    port: 80
    targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: som
  namespace: ing-thu
spec:
  rules:
  - host: som.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: som
            port:
              number: 80
EOF
```

Phần 1 — trước khi cài:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
NS=ing-thu; EV=~/lab-evidence/5b
W1="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
[ "$(sl kubectl get ingressclass --no-headers)" -eq 0 ] && echo 'PASS: cluster chua co IngressClass nao'
kubectl apply -f ~/lab-work/5b/ing-thu.yaml
kubectl -n "$NS" wait --for=condition=Ready pod/som --timeout=180s
kubectl -n "$NS" describe ingress som > "$EV/b4-ingress-som-truoc.txt"
{ [ -n "$(kubectl -n "$NS" get ingress som -o jsonpath='{.metadata.uid}')" ] \
  && [ -z "$(kubectl -n "$NS" get ingress som -o jsonpath='{.spec.ingressClassName}')" ] \
  && [ -z "$(kubectl -n "$NS" get ingress som -o jsonpath='{.status.loadBalancer.ingress}')" ]; } \
  && echo 'PASS: Ingress som duoc luu, khong class, .status rong'
if curl -s --max-time 3 -o /dev/null -H 'Host: som.lab.local' "http://$W1:30081/"; then
  echo 'FAIL: da co thu gi nghe cong 30081'
else
  echo 'PASS: chua co gi nghe cong 30081 tren node'
fi
```

Phần 2 — sau khi cài:

```bash
NS=ing-thu; EV=~/lab-evidence/5b
. ~/bookstore/platform/versions.env
W1="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"

helm version --short | grep -q "^${HELM_VER}" && echo "PASS: helm dung $HELM_VER"

helm list -n traefik > "$EV/b4-helm-list.txt"
IMG="$(kubectl -n traefik get deployment traefik -o jsonpath='{.spec.template.spec.containers[0].image}')"
echo "$IMG" > "$EV/b4-traefik-image.txt"
{ helm status traefik -n traefik | grep -q 'STATUS: deployed' && grep -q "traefik-$TRAEFIK_CHART" "$EV/b4-helm-list.txt" \
  && echo "$IMG" | grep -q ":${TRAEFIK_APP}$" \
  && kubectl -n traefik rollout status deployment/traefik --timeout=300s >/dev/null; } \
  && echo "PASS: release traefik da trien khai, chart $TRAEFIK_CHART, image $IMG"

helm show values traefik/traefik --version "$TRAEFIK_CHART" > "$EV/b4-values-mac-dinh.yaml"
grep -qE '^\s+type: LoadBalancer' "$EV/b4-values-mac-dinh.yaml" && echo 'PASS: chart mac dinh tao Service LoadBalancer'
SVC='{.spec.type} {.spec.ports[?(@.name=="web")].nodePort} {.spec.ports[?(@.name=="websecure")].nodePort}'
[ "$(kubectl -n traefik get svc traefik -o jsonpath="$SVC")" = 'NodePort 30081 30443' ] \
  && echo 'PASS: Service traefik la NodePort 30081 (http), 30443 (https)'

kubectl get ingressclass -o wide > "$EV/b4-ingressclass.txt"
DEF='{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}'
{ [ "$(kubectl get ingressclass --no-headers | wc -l)" -eq 1 ] \
  && [ "$(kubectl get ingressclass traefik -o jsonpath="$DEF")" = 'true' ]; } \
  && echo 'PASS: dung mot IngressClass, ten traefik, la mac dinh'

[ -z "$(kubectl -n "$NS" get ingress som -o jsonpath='{.spec.ingressClassName}')" ] \
  && echo 'PASS: Ingress tao truoc khi co class van khong class - mac dinh khong hoi to'
kubectl -n "$NS" delete ingress som --wait=true
kubectl apply -f ~/lab-work/5b/ing-thu.yaml
[ "$(kubectl -n "$NS" get ingress som -o jsonpath='{.spec.ingressClassName}')" = 'traefik' ] \
  && echo 'PASS: tao lai thi Ingress duoc gan class traefik'
for i in $(seq 1 15); do
  curl -s --max-time 3 -H 'Host: som.lab.local' "http://$W1:30081/" | grep -qx 'som-ok' && break; sleep 2
done
curl -s --max-time 3 -H 'Host: som.lab.local' "http://$W1:30081/" | grep -qx 'som-ok' \
  && echo 'PASS: som.lab.local qua NodePort 30081 tra som-ok'

{ test -s ~/bookstore/platform/traefik/values.yaml && test -s ~/bookstore/platform/traefik/README.md; } \
  && echo 'PASS: platform/traefik co du values.yaml va README.md'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# platform/traefik/values.yaml — Lab 5b. Chart và appVersion: platform/versions.env (bảng A1.4).
# Cluster lab không có cloud load balancer: Service của Traefik là NodePort với cổng cố định để file
# hosts và bookmark trên máy host không đổi giữa các lần cài.
service:
  spec:
    type: NodePort
ports:
  web:
    nodePort: 30081
  websecure:
    nodePort: 30443
ingressClass:
  enabled: true
  isDefaultClass: true
  name: traefik
```

```bash
. ~/bookstore/platform/versions.env

# Helm — kiểm checksum trước khi cài binary
cd ~/lab-work/5b
curl -fsSLO "https://get.helm.sh/helm-${HELM_VER}-linux-amd64.tar.gz"
curl -fsSLO "https://get.helm.sh/helm-${HELM_VER}-linux-amd64.tar.gz.sha256sum"
sha256sum -c "helm-${HELM_VER}-linux-amd64.tar.gz.sha256sum" && echo 'PASS: checksum khop'
tar -xzf "helm-${HELM_VER}-linux-amd64.tar.gz"
sudo install -o root -g root -m 0755 linux-amd64/helm /usr/local/bin/helm
rm -rf linux-amd64 "helm-${HELM_VER}-linux-amd64.tar.gz" "helm-${HELM_VER}-linux-amd64.tar.gz.sha256sum"
cd ~

# Traefik
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik --version "$TRAEFIK_CHART" \
  --namespace traefik --create-namespace -f ~/bookstore/platform/traefik/values.yaml
kubectl -n traefik rollout status deployment/traefik --timeout=600s
```

`platform/traefik/README.md` ghi: nguồn version (`versions.env`), lệnh cài Helm có kiểm checksum, lệnh
`helm install` với file values, cách kiểm (Service NodePort `30081`/`30443`, IngressClass `traefik`
mặc định duy nhất), cách nâng cấp (`helm upgrade ... -f values.yaml`, không sửa tay), và lý do không cài
load balancer.

Ý chính cho ADR:

1. Bài 11, *Điều kiện tiên quyết*: phải có Ingress controller; chỉ tạo tài nguyên thì không có tác
   dụng gì. Cùng hình dạng với B1 — object hợp lệ nhưng không ai hiện thực — chỉ khác API.
2. Bài 11, *IngressClass mặc định*: annotation bảo đảm Ingress **mới** không ghi class được gán class
   mặc định — admission điền lúc tạo, không sửa object cũ. Bài 11 cũng nói một số controller vẫn xử lý
   Ingress không class — hành vi riêng của từng controller; đừng dựa vào nó. Ghi class tường minh thì
   Ingress của dự án không đổi nghĩa khi cluster có thêm controller hay đổi class mặc định.
3. Không có cloud provider thì không ai cấp địa chỉ: `EXTERNAL-IP` treo `<pending>`. Đó là loại 5 của
   bài 164 — cloud load balancer do nhà cung cấp tạo khi Service kiểu `LoadBalancer`. MetalLB nằm ngoài
   baseline; NodePort cố định đủ cho cluster lab và cho file hosts.
4. Dự án Kubernetes chỉ bảo trì controller AWS và GCE; Traefik là bên thứ ba. Đội vận hành tự chịu
   trách nhiệm chọn version, đọc tài liệu và lưu ý của nó, theo dõi bản vá — và kiểm hành vi thật chứ
   không mặc định nó đúng đặc tả (B8).

</details>

## B5. [BS-5B-05] Zero-trust trong namespace `bookstore`

**Loại:** Tính năng — trả nợ lab #4 và `TD-09`.

**Bối cảnh:** hệ thống cũ chạy mọi thứ trên một mạng phẳng (`P7`): một container bị chiếm là chạm được
mọi thứ. Cluster giờ đã có CNI thực thi NetworkPolicy, nên ma trận ở
[mục 3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật) lần đầu có hiệu lực thật. Policy của sprint 6 chưa
từng được đo; ticket này dựng lại hệ thống, xem chỗ nào hỏng, rồi đưa `k8s/app/network/` về đúng ma
trận.

**Yêu cầu:**

1. Triển khai lại hệ thống bằng chính `deploy.sh` của tag `lab-5a`, **chưa sửa gì**. Ghi lại vào ADR:
   cái gì hỏng khi policy của sprint 6 lần đầu có hiệu lực, đối chiếu với dự đoán ở B1.
2. `k8s/app/network/` hiện thực đúng ma trận: **mặc định chặn cả hai chiều** với mọi Pod trong
   `bookstore`; chỉ mở đúng năm luồng — Traefik tới `web` 8080, `web` tới `api` 3000, `web` tới
   `node-agent` 9100, `api`/`db-migrate`/`db-backup` tới `db` 5432, mọi Pod tới CoreDNS 53 UDP và TCP.
   Không mở thêm gì.
3. Mọi NetworkPolicy mang label dự án và chọn Pod bằng cặp label của hợp đồng
   ([mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label)).
4. Từ trạng thái sạch, `deploy.sh` thành công: `api` Ready, migration xong, backup chạy được, `web` gọi
   được `api` và `node-agent` cùng node.
5. **Đúng phép đo B1, trên đúng hai file đo:** cả bảy cặp bị cấm giờ bị chặn, trong khi DNS vẫn phân
   giải.
6. Ba phép đo ranh giới: `web` tới `api` ở cổng admin bị chặn dù cổng 3000 thông; một Pod lạ trong
   namespace `traefik` không tới được `web`; `db` phân giải được một tên ngoài Internet nhưng không mở
   được kết nối ra ngoài.
7. Tìm thấy rule do Calico sinh cho policy trên một worker.
8. `docs/tech-debt.md`: `TD-09` đã trả, trỏ tới bằng chứng.

**Ràng buộc:** không dùng `ipBlock` để chọn Pod. Không viết policy cho namespace khác (`traefik`,
`kube-system`). Không đổi cổng, label hay tên của hợp đồng. Đường DNS chỉ mở tới CoreDNS, không mở cổng
53 tới mọi nơi.

**Đọc lại** — bài [84](../84-network-policies-vi.md), lần này đọc để viết:

- [*Hai kiểu cô lập pod*](../84-network-policies-vi.md#hai-kiểu-cô-lập-pod-the-two-sorts-of-pod-isolation)
  — kết nối từ node của Pod luôn được phép; một kết nối cần **cả** egress ở nguồn **lẫn** ingress ở
  đích.
- [*Hành vi của các bộ chọn `to` và `from`*](../84-network-policies-vi.md#hành-vi-của-các-bộ-chọn-to-và-from-behavior-of-to-and-from-selectors)
  — giao và hợp.
- [*Các chính sách mặc định*](../84-network-policies-vi.md#các-chính-sách-mặc-định-default-policies)
  — `default-deny-all`, và cảnh báo deny egress chặn luôn DNS.
- [*Nhắm đến một Namespace theo tên*](../84-network-policies-vi.md#nhắm-đến-một-namespace-theo-tên-targeting-a-namespace-by-its-name).
- [*Vòng đời Pod*](../84-network-policies-vi.md#vòng-đời-pod-pod-lifecycle) — Pod có thể khởi động
  trước khi rule cho phép được áp.
- [*Những gì bạn không thể làm với network policy*](../84-network-policies-vi.md#những-gì-bạn-không-thể-làm-với-network-policy-ít-nhất-là-chưa-what-you-cant-do-with-network-policies-at-least-not-yet)
  — loopback và lưu lượng từ host.
- Bài [201](../201-declare-network-policy-vi.md) — thử cả Pod có quyền lẫn Pod không có quyền.

**Bàn giao:** `k8s/app/network/*.yaml`; `k8s/app/kustomization.yaml`; `docs/adr/NNNN-zero-trust.md`;
`docs/tech-debt.md`; `~/lab-evidence/5b/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` `teardown.sh` rồi `deploy.sh` thành công khi ma trận có hiệu lực; mọi policy mang label dự án.
- `PASS:` `api` Ready (probe của kubelet và luồng `api` → `db` thông); `db-migrate` Complete; một lần
  chạy `db-backup` thành công.
- `PASS:` `web` gọi `api:3000/books` theo tên; `web` đọc `/node.json` của agent cùng node; `db-0` phân
  giải được tên.
- `PASS:` `probe.sh` trên đúng hai file đo của B1: cả bảy cặp `CHAN`, không cặp nào lỗi DNS.
- `PASS:` `web` → `api:3001` bị chặn; Pod lạ trong `traefik` → `web` bị chặn; `db-0` phân giải được
  tên ngoài nhưng không kết nối ra ngoài được.
- `PASS:` `lab-k8s-worker1` có chain policy của Calico.
- `PASS:` đã lưu hồ sơ trả nợ #4; `tech-debt.md` ghi `TD-09` đã trả.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-zero-trust.md`:

1. Luồng `web` → `api` cần rule ở đâu, vì sao một policy ở `api` là chưa đủ khi đã chặn hai chiều?
2. Readiness của `api` gọi `/ready` ở cổng 3001 — cổng mà không ai được mở tới. Vì sao Pod vẫn Ready?
   Readiness của `db` là `pg_isready -h 127.0.0.1` — NetworkPolicy có chặn được nó không?
3. Rule Traefik → `web`: một phần tử `from` chứa cả hai selector, hay hai phần tử tách rời? Với Pod lạ
   trong namespace `traefik`, mỗi cách viết cho kết quả gì?
4. Policy ghi cổng `8080` cho `web`, trong khi Service `web` nghe cổng `80`. Vì sao? Triệu chứng khi
   ghi nhầm là gì?
5. Bài 84 nói một Pod có thể khởi động khi quy tắc cô lập đã áp mà quy tắc cho phép chưa áp. Hệ thống
   của bạn đã có sẵn cơ chế nào chịu được điều đó, và thành phần nào chưa có?

<details>
<summary>Gợi ý</summary>

1. Bắt đầu bằng một policy chọn mọi Pod (`podSelector: {}`) với cả `Ingress` lẫn `Egress` trong
   `policyTypes`, không có rule nào. Rồi mỗi luồng thêm rule cho phép.
2. Namespace mang sẵn label `kubernetes.io/metadata.name`. CoreDNS là Pod label `k8s-app: kube-dns`
   trong `kube-system`. Pod của Traefik mang `app.kubernetes.io/name: traefik` — kiểm bằng
   `kubectl -n traefik get pods --show-labels`.
3. Nhiều thành phần cùng được vào `db`: `matchExpressions` với `operator: In` trên key
   `app.kubernetes.io/component`.
4. `kubectl apply` không xóa object đã bỏ khỏi repo. Đổi tên hay gộp policy thì
   `./scripts/teardown.sh && ./scripts/deploy.sh` để không còn policy cũ — policy cộng gộp, một policy
   sót lại có thể mở một luồng bạn tưởng đã đóng.
5. Khi không chắc Kubernetes hiểu policy ra sao: `kubectl -n bookstore describe networkpolicy <tên>`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5b
SEL='app.kubernetes.io/part-of=bookstore'
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

# 1. Dựng sạch với ma trận mới
./scripts/teardown.sh > "$EV/b5-teardown.txt" 2>&1
./scripts/deploy.sh > "$EV/b5-deploy.txt" 2>&1 && grep -q 'teardown: sach' "$EV/b5-teardown.txt" \
  && echo 'PASS: teardown sach, deploy.sh thanh cong khi ma tran co hieu luc'
ALL="$(kubectl -n "$NS" get networkpolicy --no-headers 2>/dev/null | wc -l)"
LAB="$(kubectl -n "$NS" get networkpolicy -l "$SEL" --no-headers 2>/dev/null | wc -l)"
kubectl -n "$NS" describe networkpolicy > "$EV/b5-networkpolicy.txt"
{ [ "$ALL" -ge 1 ] && [ "$ALL" -eq "$LAB" ]; } && echo "PASS: $ALL NetworkPolicy, tat ca mang label du an"

# 2. Các luồng được phép
{ [ "$(jp deployment/api '{.status.readyReplicas}')" = "$(jp deployment/api '{.spec.replicas}')" ] \
  && [ "$(jp job/db-migrate '{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ]; } \
  && echo 'PASS: api Ready (probe kubelet va api -> db thong), db-migrate -> db thong'
J="b5-backup-$(date +%s)"
kubectl -n "$NS" create job "$J" --from=cronjob/db-backup >/dev/null
kubectl -n "$NS" wait --for=condition=Complete "job/$J" --timeout=180s >/dev/null \
  && kubectl -n "$NS" logs "job/$J" | grep -q '^db-backup: OK' && echo 'PASS: db-backup -> db thong'
kubectl -n "$NS" delete job "$J" --wait=true >/dev/null
WEB="$(kubectl -n "$NS" get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"
WNODE="$(jp "pod/$WEB" '{.spec.nodeName}')"
get() { kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- -T 5 "$1" 2>/dev/null; }
{ [ "$(get http://api:3000/books | grep -o '"title"' | wc -l)" -ge 8 ] \
  && get http://node-agent:9100/node.json | grep -Eq "\"node\" *: *\"$WNODE\"" \
  && kubectl -n "$NS" exec db-0 -c postgres -- getent hosts api >/dev/null; } \
  && echo 'PASS: web -> api:3000, web -> node-agent cung node, DNS tu db-0 deu thong'

# 3. Đúng phép đo B1
kubectl apply -f ~/lab-work/5b/probe-pods.yaml
kubectl -n "$NS" wait --for=condition=Ready pod/np-thu --timeout=180s
kubectl -n np-ngoai wait --for=condition=Ready pod/np-ngoai --timeout=180s
~/lab-work/5b/probe.sh > "$EV/b5-probe-calico.txt"
paste "$EV/b1-probe-flannel.txt" "$EV/b5-probe-calico.txt" | tee "$EV/b5-truoc-sau.txt"
{ [ "$(grep -c '^CHAN' "$EV/b5-probe-calico.txt")" -eq 7 ] && ! grep -q '^LOIDNS' "$EV/b5-probe-calico.txt"; } \
  && echo 'PASS: cung phep do cua B1 - ca bay cap bi cam deu CHAN, DNS van phan giai'

# 4. Ranh giới
API_IP="$(kubectl -n "$NS" get pod -l "$SEL,app.kubernetes.io/component=api" -o jsonpath='{.items[0].status.podIP}')"
WEB_IP="$(jp "pod/$WEB" '{.status.podIP}')"
kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -q -O /dev/null -T 5 "http://$API_IP:3001/live" >/dev/null 2>&1 \
  && echo 'FAIL: web toi duoc cong admin 3001 cua api' || echo 'PASS: web -> api:3001 bi chan, chi cong 3000 duoc mo'
kubectl -n traefik run np-la --image=postgres:17.11 --restart=Never --command -- sleep 600 >/dev/null
kubectl -n traefik wait --for=condition=Ready pod/np-la --timeout=180s >/dev/null
kubectl -n traefik exec np-la -- timeout 5 bash -c "exec 3<>/dev/tcp/$WEB_IP/8080" >/dev/null 2>&1 \
  && echo 'FAIL: Pod la trong namespace traefik vao duoc web' \
  || echo 'PASS: Pod la trong traefik bi chan - rule Traefik -> web la phep giao'
kubectl -n traefik delete pod np-la --wait=true >/dev/null
if kubectl -n "$NS" exec db-0 -c postgres -- getent hosts registry.k8s.io >/dev/null 2>&1; then
  kubectl -n "$NS" exec db-0 -c postgres -- timeout 5 bash -c 'exec 3<>/dev/tcp/registry.k8s.io/443' >/dev/null 2>&1 \
    && echo 'FAIL: db-0 ket noi ra Internet duoc' \
    || echo 'PASS: db-0 phan giai duoc ten ngoai nhung khong ket noi ra ngoai duoc'
fi

# 5. Calico dịch policy thành rule trên node
ssh lab-k8s-worker1 "sudo iptables-save 2>/dev/null | grep -qE 'cali-p[io]-' || sudo nft list ruleset 2>/dev/null | grep -qE 'cali-p[io]-'" \
  && echo 'PASS: lab-k8s-worker1 co chain policy cua Calico'

# 6. Hồ sơ
{
  date -Is
  echo '=== NO #4 DA TRA TAI LAB 5B ==='
  echo '--- B1, Flannel: policy ton tai nhung khong chan'
  cat "$EV/b1-bang-chung-no-4.txt"
  echo '--- B5, Calico: cung hai file do, cung probe.sh'
  cat "$EV/b5-truoc-sau.txt"
} > "$EV/b5-tra-no-4.txt"
{ test -s "$EV/b5-tra-no-4.txt" && grep -E 'TD-09.*đã trả' docs/tech-debt.md >/dev/null; } \
  && echo 'PASS: da luu ho so tra no #4; tech-debt.md ghi TD-09 da tra'
kubectl delete -f ~/lab-work/5b/probe-pods.yaml --wait=true
```

⭐ `endPort` — đo trên đích thử nghiệm `som` của B4, không đụng vào dự án:

```bash
cat > ~/lab-work/5b/endport-thu.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: endport-thu
  namespace: ing-thu
spec:
  podSelector:
    matchLabels:
      app: som
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}
    ports:
    - protocol: TCP
      port: 8081
      endPort: 8090
EOF
SOM="$(kubectl -n ing-thu get pod som -o jsonpath='{.status.podIP}')"
kubectl -n ing-thu run ep-client --image=busybox:1.37 --restart=Never --command -- sleep 600 >/dev/null
kubectl -n ing-thu wait --for=condition=Ready pod/ep-client --timeout=120s >/dev/null
kubectl apply -f ~/lab-work/5b/endport-thu.yaml
kubectl -n ing-thu exec ep-client -- wget -qO- -T 5 "http://$SOM:8080/" >/dev/null 2>&1 \
  && echo 'FAIL: 8080 ngoai dai 8081-8090 van qua' || echo 'PASS: cong ngoai dai bi chan - endPort duoc thuc thi'
sed 's/port: 8081/port: 8080/' ~/lab-work/5b/endport-thu.yaml | kubectl apply -f -
kubectl -n ing-thu exec ep-client -- wget -qO- -T 5 "http://$SOM:8080/" | grep -qx 'som-ok' \
  && echo 'PASS: dua 8080 vao dai thi qua duoc'
kubectl -n ing-thu delete networkpolicy endport-thu; kubectl -n ing-thu delete pod ep-client --wait=true
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Bảy file trong `k8s/app/network/`. Nếu sprint 6 đặt tên khác, thay hẳn bằng bộ của bạn rồi teardown,
deploy — đừng để hai bộ cùng tồn tại.

```yaml
# k8s/app/network/00-default-deny.yaml — mọi Pod trong bookstore: chặn cả hai chiều
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# k8s/app/network/01-allow-dns.yaml — mọi Pod tới CoreDNS, đúng cổng 53
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - {protocol: UDP, port: 53}
    - {protocol: TCP, port: 53}
```

```yaml
# k8s/app/network/web.yaml — vào: chỉ Pod Traefik; ra: api 3000, node-agent 9100
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:              # MỘT phần tử, hai selector: phép giao
        matchLabels:
          kubernetes.io/metadata.name: traefik
      podSelector:
        matchLabels:
          app.kubernetes.io/name: traefik
    ports:
    - {protocol: TCP, port: 8080}     # cổng của Pod nginx, không phải cổng 80 của Service
  egress:
  - to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/part-of: bookstore
          app.kubernetes.io/component: api
    ports:
    - {protocol: TCP, port: 3000}
  - to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/part-of: bookstore
          app.kubernetes.io/component: agent
    ports:
    - {protocol: TCP, port: 9100}
---
# k8s/app/network/api.yaml — vào: chỉ web ở 3000; ra: db 5432
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: api
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/part-of: bookstore
          app.kubernetes.io/component: web
    ports:
    - {protocol: TCP, port: 3000}
  egress:
  - to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/part-of: bookstore
          app.kubernetes.io/component: db
    ports:
    - {protocol: TCP, port: 5432}
```

```yaml
# k8s/app/network/db.yaml — vào: api, db-migrate, db-backup ở 5432. Lab 14 thêm Job của operator.
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: db
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/part-of: bookstore
        matchExpressions:
        - {key: app.kubernetes.io/component, operator: In, values: [api, migrate, backup]}
    ports:
    - {protocol: TCP, port: 5432}
---
# k8s/app/network/jobs.yaml — db-migrate và db-backup ra db 5432
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: jobs
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
    matchExpressions:
    - {key: app.kubernetes.io/component, operator: In, values: [migrate, backup]}
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/part-of: bookstore
          app.kubernetes.io/component: db
    ports:
    - {protocol: TCP, port: 5432}
---
# k8s/app/network/agent.yaml — node-agent: vào chỉ từ web ở 9100
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: node-agent
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: agent
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: agent
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/part-of: bookstore
          app.kubernetes.io/component: web
    ports:
    - {protocol: TCP, port: 9100}
```

Trong `k8s/app/kustomization.yaml`, mục `resources` liệt kê đúng bảy file trên (bỏ các dòng `network/`
cũ của sprint 6 nếu tên khác). Rồi:

```bash
cd ~/bookstore && ./scripts/teardown.sh && ./scripts/deploy.sh
```

Dòng `TD-09` trong `docs/tech-debt.md`: trạng thái `đã trả`, trả ở `5b`, bằng chứng
`~/lab-evidence/5b/b5-tra-no-4.txt`.

Ý chính cho ADR:

1. Bài 84: kết nối chỉ đi được khi **cả** egress ở Pod nguồn **lẫn** ingress ở Pod đích cho phép.
   `default-deny-all` cô lập `web` chiều đi, nên chỉ mở ingress ở `api` thì gói tin không ra khỏi
   `web`.
2. Kết nối từ node của Pod luôn được phép (bài 84, *Hai kiểu cô lập pod*): probe `httpGet` đi từ
   kubelet trên chính node đó. `pg_isready -h 127.0.0.1` chạy trong Pod qua loopback — bài 84 liệt kê
   việc chặn loopback là thứ NetworkPolicy **không** làm được.
3. Một phần tử chứa cả `namespaceSelector` và `podSelector` là phép **giao**: chỉ Pod Traefik trong
   namespace `traefik`. Hai phần tử tách rời là phép **hợp**: **mọi** Pod trong `traefik`, hoặc Pod
   label `traefik` trong chính `bookstore` — Pod lạ ở `traefik` lọt vào. Script nghiệm thu đo đúng chỗ
   này.
4. NetworkPolicy làm việc ở tầng Pod; kết nối tới ClusterIP đã được đổi đích sang `PodIP:8080` trước
   khi tới Pod. Ghi `80` thì object hợp lệ, không gate nào báo lỗi, nhưng Traefik nhận timeout — lỗi im
   lặng kinh điển.
5. Init container `wait-for-db` của `api` lặp tới khi DB nhận kết nối — đúng cách bài 84 khuyên khi
   Pod phải tiếp cận đích trước khi chạy. `db-migrate` chưa có: lượt thử đầu có thể hỏng nếu rule chưa
   áp; nó dựa vào `backoffLimit` và `deploy.sh` tạo lại Job một lần.

</details>

## B6. [BS-5B-06] HTTPS qua Ingress, `web` về ClusterIP

**Loại:** Tính năng — thay `deploy.sh`, `teardown.sh`.

**Bối cảnh:** người dùng của hiệu sách phải vào bằng một tên và bằng HTTPS, không bằng `IP:30080`. Với
ma trận đã đóng, Traefik là đường vào **duy nhất** được phép tới `web`; NodePort `30080` vừa thừa vừa
đi vòng qua ma trận. Certificate là bí mật sinh tại chỗ, theo đúng cơ chế mà `secrets/db.env` đã dùng
từ Lab 3b ([mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint)).

**Yêu cầu:**

1. `scripts/deploy.sh` và `scripts/teardown.sh` thay bằng phiên bản Lab 5b — chép nguyên văn từ khối
   bên dưới. `deploy.sh` sinh `secrets/tls.crt`, `secrets/tls.key` (tự ký, tên `bookstore.lab.local`)
   khi thiếu, và chỉ thoát `0` khi Traefik đã phục vụ HTTPS bằng đúng certificate đó.
2. Secret `bookstore-tls`: kiểu TLS, label dự án, tên cố định, **Kustomize sinh** từ hai file trong
   `secrets/` — không tạo bằng lệnh gõ tay.
3. Ingress `bookstore` ở `k8s/app/ingress/ingress.yaml`: host `bookstore.lab.local`, class `traefik`
   ghi **tường minh**, TLS bằng Secret trên, mọi path về Service `web`.
4. Service `web` về `ClusterIP`; cổng `30080` không còn trả lời.
5. Từ master: HTTPS tới `bookstore.lab.local` qua cổng `30443` của **cả hai** worker trả trang chủ và
   `/api/books`; client kiểm certificate bằng chính `secrets/tls.crt`, không dùng `-k`.
6. Traefik chỉ dùng certificate của dự án cho đúng tên đó; tên khác nhận certificate khác.
7. Từ máy host Windows: file hosts trỏ `bookstore.lab.local` tới IP một worker; trình duyệt mở
   `https://bookstore.lab.local:30443/` thấy trang Bookstore có danh sách sách (cảnh báo certificate
   là bình thường — `TD-11`).
8. `README.md` của repo ghi URL truy cập và việc cần làm trên máy host.

**Ràng buộc:** không commit certificate hay key. Không `kubectl create secret tls`. Ingress không mở
đường tới `api` trực tiếp — ma trận chỉ cho Traefik vào `web`.

**Đọc lại:**

- Bài [11](../11-ingress-vi.md): [*Quy tắc Ingress*](../11-ingress-vi.md#quy-tắc-ingress-ingress-rules),
  [*Các loại path*](../11-ingress-vi.md#các-loại-path-path-types), [*TLS*](../11-ingress-vi.md#tls) —
  Secret, cổng 443 duy nhất, TLS kết thúc tại ingress, ghi chú `hosts` khớp `rules`.
- Bài [109](../109-secret-vi.md#secret-tls-tls-secrets) — Secret TLS.
- Bài [328](../328-secret-kustomize-vi.md#file-files) — nguồn `files`, tên file thành key; bài
  [322](../322-kustomization-vi.md#generatoroptions) — `generatorOptions`.
- Kustomize: [secretGenerator](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/secretgenerator/)
  — ví dụ Secret kiểu `kubernetes.io/tls`.
- Bài [82](../82-service-vi.md#type-clusterip) — `type: ClusterIP`.

**Bàn giao:** `scripts/deploy.sh`, `scripts/teardown.sh`, `k8s/app/ingress/ingress.yaml`,
`k8s/app/web/service.yaml`, `k8s/app/kustomization.yaml`, `README.md`; `docs/adr/NNNN-https-ingress.md`;
`docs/tech-debt.md` thêm `TD-11`; `~/lab-evidence/5b/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` teardown sạch mà hạ tầng còn nguyên; `deploy.sh` chạy hai lần liên tiếp đều thành công.
- `PASS:` git bỏ qua `secrets/tls.crt`, `secrets/tls.key`; key quyền `600`; certificate mang tên
  `bookstore.lab.local` và còn hạn.
- `PASS:` Secret `bookstore-tls` kiểu `kubernetes.io/tls`, label dự án, dữ liệu khớp file trong
  `secrets/`.
- `PASS:` Ingress `bookstore`: đúng host, class `traefik`, TLS khớp host, Secret `bookstore-tls`, về
  Service `web`.
- `PASS:` Service `web` là `ClusterIP`; `30080` không trả lời.
- `PASS:` HTTPS qua cả hai worker: trang chủ và `/api/books`, certificate kiểm được bằng
  `secrets/tls.crt`.
- `PASS:` tên khác nhận certificate khác certificate của dự án.
- `PASS:` máy host Windows mở được `https://bookstore.lab.local:30443/` qua file hosts.

Hai script, chép nguyên văn:

```bash
cat > ~/bookstore/scripts/deploy.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai dự án Bookstore từ repo — phiên bản Lab 5b.
# Cần cluster ở mốc 02-net-ready: Calico thực thi NetworkPolicy; Traefik và IngressClass traefik.
# - Bí mật sinh tại chỗ khi thiếu, không bao giờ vào git: secrets/db.env (từ 3b) và cặp certificate
#   tự ký secrets/tls.crt, secrets/tls.key cho bookstore.lab.local (từ 5b).
# - Mọi object của namespace bookstore nằm trong k8s/app/kustomization.yaml (từ 5a). Secret sinh từ
#   file ngoài thư mục gốc của kustomization nên phải render với --load-restrictor (Lab 3b, ADR 0007).
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore
HOST=bookstore.lab.local

# 0. Hạ tầng nền thuộc cluster, không thuộc repo: thiếu thì dừng, không tự cài.
for X in 'daemonset/calico-node -n kube-system' 'ingressclass/traefik' 'service/traefik -n traefik'; do
  kubectl get $X >/dev/null 2>&1 || { echo "deploy: thieu $X - cluster chua o moc 02-net-ready"; exit 1; }
done

# 1. Bí mật sinh tại chỗ khi thiếu.
mkdir -p secrets
if [ ! -s secrets/db.env ]; then
  ( umask 077
    printf 'postgres-password=%s\nauthenticator-password=%s\n' \
      "$(openssl rand -hex 24)" "$(openssl rand -hex 24)" > secrets/db.env )
  echo 'deploy: da sinh secrets/db.env moi'
fi
if [ ! -s secrets/tls.crt ] || [ ! -s secrets/tls.key ]; then
  ( umask 077
    openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
      -keyout secrets/tls.key -out secrets/tls.crt \
      -subj "/CN=${HOST}" -addext "subjectAltName=DNS:${HOST}" )
  echo "deploy: da sinh certificate tu ky moi cho ${HOST}"
fi

# 2. Object cấp cluster, rồi toàn bộ namespace bookstore. Pod template của Job không sửa được sau khi
#    tạo: xóa db-migrate cũ để lần apply này tạo Job mới (migration bất biến kết quả, Lab 4b).
if compgen -G 'k8s/cluster/*.yaml' >/dev/null; then kubectl apply -f k8s/cluster/; fi
if kubectl get namespace "$NS" >/dev/null 2>&1; then
  kubectl -n "$NS" delete job db-migrate --ignore-not-found --wait=true
fi
render() { kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app; }
render | kubectl apply -f -

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

# 3. Chờ theo thứ tự phụ thuộc: db → migration → api, web → agent.
kubectl -n "$NS" rollout status statefulset/db --timeout=300s
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s
if kubectl -n "$NS" get job db-migrate >/dev/null 2>&1; then
  if ! wait_job db-migrate; then
    # Job tạo cùng lúc với db-0 có thể hết lượt thử trước khi DB nhận kết nối: tạo lại một lần.
    echo 'deploy: db-migrate that bai lan dau, tao lai khi db-0 da Ready'
    kubectl -n "$NS" delete job db-migrate --wait=true
    render | kubectl apply -f -
    wait_job db-migrate || { echo 'deploy: db-migrate that bai'; exit 1; }
  fi
fi
for D in api web; do kubectl -n "$NS" rollout status "deployment/$D" --timeout=300s; done
kubectl -n "$NS" rollout status daemonset/node-agent --timeout=300s

# 4. Đường vào: Traefik phải phục vụ HTTPS bằng đúng certificate của dự án.
NP="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}')"
IP="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
OK=0
for _ in $(seq 1 30); do
  if curl -fsS --max-time 5 --cacert secrets/tls.crt --resolve "${HOST}:${NP}:${IP}" \
       "https://${HOST}:${NP}/healthz" >/dev/null 2>&1; then OK=1; break; fi
  sleep 2
done
[ "$OK" -eq 1 ] || { echo "deploy: https://${HOST}:${NP}/ chua phuc vu qua Traefik"; exit 1; }

kubectl -n "$NS" get statefulset,deployment,daemonset,cronjob,job,service,ingress,networkpolicy
echo "deploy: xong - https://${HOST}:${NP}/ (file hosts tro ${HOST} toi IP mot worker)"
EOF

cat > ~/bookstore/scripts/teardown.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ toàn bộ dự án Bookstore khỏi cluster — phiên bản Lab 5b.
# Chỉ gỡ object mang label dự án. Hạ tầng của mốc 02-net-ready (Calico, Traefik, IngressClass traefik)
# không mang label đó và phải còn nguyên. Giữ nguyên secrets/ trên máy.
set -euo pipefail
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
# Kind namespaced được dọn theo label ở mọi namespace (3b, 4a, 5a); 5b thêm ingresses.
KINDS='statefulsets,deployments,daemonsets,cronjobs,jobs,replicasets,pods,poddisruptionbudgets,services,networkpolicies,ingresses,configmaps,secrets'

# Xóa StatefulSet không bảo đảm Pod tắt có thứ tự (bài 65, mục Hạn chế): thu db về 0 trước.
if kubectl -n "$NS" get statefulset db >/dev/null 2>&1; then
  kubectl -n "$NS" scale statefulset db --replicas=0
  kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s || true
fi

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
LEFT=0; LEFT_NS=0
dem LEFT kubectl get namespace,priorityclass -l "$SEL" --no-headers
dem LEFT_NS kubectl get all -A -l "$SEL" --no-headers
dem LEFT_NS kubectl get "$KINDS,endpointslices" -A -l "$SEL" --no-headers
if [ "$LEFT" -eq 0 ] && [ "$LEFT_NS" -eq 0 ]; then
  echo 'teardown: sach'
else
  echo "teardown: con $LEFT object cap cluster va $LEFT_NS object co label du an"; exit 1
fi
{ kubectl get ingressclass traefik >/dev/null 2>&1 && kubectl -n kube-system get daemonset calico-node >/dev/null 2>&1; } \
  || echo 'teardown: CANH BAO - ha tang 02-net-ready khong con nguyen (khong do script nay go)'
EOF

chmod +x ~/bookstore/scripts/*.sh
```

`deploy.sh` giả định mọi object của `bookstore` — kể cả Job `db-migrate` và CronJob `db-backup` — nằm
trong `k8s/app/kustomization.yaml`, như cơ chế của sprint 6 ([mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint)).
Nếu tag `lab-5a` của bạn còn object nào triển khai ngoài Kustomize, đưa nó vào `resources` trước khi
dùng bản mới.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-https-ingress.md`:

1. TLS kết thúc ở đâu? Đoạn nào từ trình duyệt tới DB là plaintext, và vì sao dự án tạm chấp nhận
   (`TD-11`)?
2. Vì sao `hosts` trong phần `tls` phải khớp tường minh `host` trong `rules`? Bài 11 nói gì về TLS trên
   quy tắc mặc định?
3. Vì sao `web` về `ClusterIP` thay vì giữ thêm NodePort `30080` "cho tiện"? Trả lời bằng ma trận và
   bằng điều bạn thấy ở câu 4 của B1.
4. Certificate và key sinh tại chỗ, không vào git, không có trong bundle. Restore snapshot rồi clone lại
   từ bundle thì chuyện gì xảy ra với trình duyệt của người dùng? Vì sao vẫn không commit chúng?
5. Bài 109 nói Secret TLS chỉ là kiểu "để thuận tiện". Vì sao dự án vẫn dùng kiểu `kubernetes.io/tls`
   thay vì `Opaque`?

<details>
<summary>Gợi ý</summary>

1. `secretGenerator` có trường `type`; nguồn `files` lấy tên file làm key — hai file đã mang đúng tên
   `tls.crt` và `tls.key` mà Secret TLS bắt buộc.
2. Tên cố định: cùng khối `options` mà `db-credentials` đã dùng từ Lab 3b.
3. Backend của Ingress trỏ **Service** `web` ở cổng của Service (80), không phải cổng Pod.
4. Gọi bằng tên mà không sửa `/etc/hosts` của master:
   `curl --cacert secrets/tls.crt --resolve bookstore.lab.local:30443:<IP worker> https://bookstore.lab.local:30443/`.
5. Đổi `type` của Service đang có thì `kubectl apply` gỡ `nodePort`; nếu báo lỗi, teardown rồi deploy.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5b; HOST=bookstore.lab.local
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

./scripts/teardown.sh > "$EV/b6-teardown.txt" 2>&1
{ grep -q 'teardown: sach' "$EV/b6-teardown.txt" && kubectl get ingressclass traefik >/dev/null \
  && kubectl -n traefik rollout status deployment/traefik --timeout=60s >/dev/null; } \
  && echo 'PASS: teardown sach, ha tang con nguyen'
{ ./scripts/deploy.sh > "$EV/b6-deploy-1.txt" 2>&1 && ./scripts/deploy.sh > "$EV/b6-deploy-2.txt" 2>&1; } \
  && echo 'PASS: deploy.sh chay hai lan lien tiep deu thanh cong'

{ git check-ignore -q secrets/tls.crt && git check-ignore -q secrets/tls.key \
  && [ "$(stat -c %a secrets/tls.key)" = '600' ] \
  && openssl x509 -in secrets/tls.crt -noout -ext subjectAltName | grep -q "DNS:$HOST" \
  && openssl x509 -in secrets/tls.crt -noout -checkend 2592000 >/dev/null; } \
  && echo "PASS: cert/key ngoai git, key 600, certificate cho $HOST con han"

{ [ "$(jp secret/bookstore-tls '{.type}')" = 'kubernetes.io/tls' ] \
  && [ "$(jp secret/bookstore-tls '{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ] \
  && kubectl -n "$NS" get secret bookstore-tls -o jsonpath='{.data.tls\.crt}' | base64 -d | cmp -s - secrets/tls.crt \
  && kubectl -n "$NS" get secret bookstore-tls -o jsonpath='{.data.tls\.key}' | base64 -d | cmp -s - secrets/tls.key; } \
  && echo 'PASS: Secret bookstore-tls kieu TLS, label du an, du lieu khop secrets/'

ING='{.spec.rules[0].host}|{.spec.ingressClassName}|{.spec.tls[0].hosts[0]}|{.spec.tls[0].secretName}|{.spec.rules[0].http.paths[0].backend.service.name}'
[ "$(jp ingress/bookstore "$ING")" = "$HOST|traefik|$HOST|bookstore-tls|web" ] \
  && echo 'PASS: Ingress bookstore dung host, class traefik, TLS khop host, ve Service web'

W1="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
{ [ "$(jp service/web '{.spec.type}')" = 'ClusterIP' ] && [ -z "$(jp service/web '{.spec.ports[*].nodePort}')" ] \
  && ! curl -s --max-time 3 -o /dev/null "http://$W1:30080/"; } \
  && echo 'PASS: Service web la ClusterIP, cong 30080 khong con tra loi'

NP="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}')"
BAD=0
for N in lab-k8s-worker1 lab-k8s-worker2; do
  IP="$(kubectl get node "$N" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
  C="curl -fsS --max-time 10 --cacert secrets/tls.crt --resolve $HOST:$NP:$IP"
  $C "https://$HOST:$NP/" > "$EV/b6-https-$N.html" 2>&1 && grep -q 'Bookstore' "$EV/b6-https-$N.html" || BAD=1
  [ "$($C "https://$HOST:$NP/api/books" 2>/dev/null | grep -o '"title"' | wc -l)" -ge 8 ] || BAD=1
done
[ "$BAD" -eq 0 ] && echo "PASS: HTTPS qua ca hai worker, cong $NP: trang chu va /api/books, certificate kiem duoc"

fp() { openssl s_client -connect "$W1:$NP" -servername "$1" </dev/null 2>/dev/null | openssl x509 -noout -fingerprint -sha256; }
MINE="$(openssl x509 -in secrets/tls.crt -noout -fingerprint -sha256)"
{ [ "$(fp "$HOST")" = "$MINE" ] && [ -n "$(fp khac.lab.local)" ] && [ "$(fp khac.lab.local)" != "$MINE" ]; } \
  && echo 'PASS: Traefik chon certificate theo SNI - ten khac nhan certificate khac'
```

Trên **PowerShell chạy "Run as administrator" của máy host Windows** — thay IP bằng InternalIP của
`lab-k8s-worker1`:

```powershell
$w1 = '<IP cua lab-k8s-worker1>'
$hosts = "$env:SystemRoot\System32\drivers\etc\hosts"
if (-not (Select-String -Path $hosts -Pattern 'bookstore\.lab\.local' -Quiet)) {
  Add-Content -Path $hosts -Value "`r`n$w1 bookstore.lab.local"
}
ipconfig /flushdns | Out-Null
$ip = [System.Net.Dns]::GetHostAddresses('bookstore.lab.local') | Select-Object -First 1 -ExpandProperty IPAddressToString
if ($ip -eq $w1) { 'PASS: file hosts tro bookstore.lab.local toi worker' } else { "FAIL: phan giai ra '$ip'" }
$page = (& curl.exe -k -s --max-time 10 'https://bookstore.lab.local:30443/') -join "`n"
if ($page -match 'Bookstore') { 'PASS: may host mo duoc https://bookstore.lab.local:30443/' } else { 'FAIL: khong co trang Bookstore' }
$api = (& curl.exe -k -s --max-time 10 'https://bookstore.lab.local:30443/api/books') -join ''
if (([regex]::Matches($api, '"title"')).Count -ge 8) { 'PASS: /api/books qua HTTPS tra du sach' } else { 'FAIL: /api/books thieu sach' }
```

`-k` ở đây chỉ vì máy host chưa tin certificate tự ký (`TD-11`); phía master đã kiểm certificate thật.
Mở thêm trang đó bằng trình duyệt để thấy danh sách sách do JavaScript gọi `/api/books`.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bookstore
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
spec:
  ingressClassName: traefik        # tường minh: không phụ thuộc class mặc định của cluster
  tls:
  - hosts:
    - bookstore.lab.local          # phải khớp tường minh host trong rules
    secretName: bookstore-tls
  rules:
  - host: bookstore.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80           # cổng của Service; policy ở B5 ghi 8080 của Pod
```

`k8s/app/web/service.yaml` — chỉ đổi phần `spec`: bỏ `type: NodePort` và `nodePort: 30080`:

```yaml
spec:
  type: ClusterIP
  selector:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
  ports:
  - name: http
    port: 80
    targetPort: http
```

Thêm vào `k8s/app/kustomization.yaml`:

```yaml
resources:
# ... các dòng đã có ...
- ingress/ingress.yaml

secretGenerator:
# ... db-credentials giữ nguyên ...
- name: bookstore-tls
  namespace: bookstore
  type: kubernetes.io/tls
  files:
  - ../../secrets/tls.crt
  - ../../secrets/tls.key
  options:
    disableNameSuffixHash: true
    labels:
      app.kubernetes.io/part-of: bookstore
      app.kubernetes.io/component: web
```

```bash
mkdir -p ~/bookstore/k8s/app/ingress
cat >> ~/bookstore/README.md <<'EOF'
- Truy cập:   https://bookstore.lab.local:30443/ — máy host thêm dòng "<IP worker> bookstore.lab.local"
              vào file hosts. Certificate tự ký sinh tại chỗ ở secrets/tls.crt (TD-11).
EOF
cd ~/bookstore && ./scripts/teardown.sh && ./scripts/deploy.sh
```

Dòng mới trong `docs/tech-debt.md`:

```markdown
| TD-11 | Certificate tự ký sinh tại chỗ, client không tin; gia hạn bằng tay (365 ngày); Traefik → web là plaintext | 5b | chưa có lab trả — chấp nhận có điều kiện | mở | docs/adr/NNNN-https-ingress.md |
```

Ý chính cho ADR:

1. Bài 11, *TLS*: TLS kết thúc tại điểm ingress — trình duyệt tới Traefik được mã hóa; Traefik tới
   `web`, `web` tới `api` và `api` tới `db` là plaintext trong mạng Pod. Chấp nhận vì mọi chặng đó giờ
   bị ma trận giới hạn đúng cặp nguồn–đích; mã hóa trong cluster cần service mesh hoặc cấu hình TLS
   từng thành phần — bài 84 xếp "mọi thứ liên quan TLS" ra ngoài NetworkPolicy.
2. Certificate phải cấp cho đúng tên; TLS trên quy tắc mặc định (không host) sẽ cần certificate cho
   mọi tên con có thể có. Traefik chọn certificate theo SNI — tên khác nhận certificate khác, đúng điều
   script đo.
3. Ma trận chỉ cho Traefik vào `web`. NodePort đưa lưu lượng vào qua node; ngoại lệ "kết nối từ node
   của Pod" của bài 84 không phủ được node khác, nên với policy có hiệu lực đường này không đoán trước
   được, lại đi vòng qua ma trận. Một đường vào duy nhất thì dễ bảo vệ, dễ đo.
4. `deploy.sh` sinh certificate mới; trình duyệt thấy một certificate khác và cảnh báo lại. Không commit
   vì key riêng nằm trong git là lộ cho mọi người đọc được repo — đúng quy tắc 3 của
   [mục 5.3](DU-AN-BOOKSTORE.md#53-quy-tắc-làm-việc-với-repo). Muốn giữ certificate qua các lần dựng
   lại thì sao lưu `secrets/` bằng một kênh an toàn riêng.
5. API server kiểm Secret kiểu TLS có đủ `tls.crt` và `tls.key` (bài 109); thiếu một key là bị từ
   chối ngay lúc apply thay vì lỗi lúc Traefik nạp certificate. Kiểu chuẩn cũng là thứ controller tìm.

</details>

## B7. [BS-5B-07] Điều tra: proxy, dual-stack và Gateway API

**Loại:** Điều tra.

**Bối cảnh:** trước khi đóng sprint, đội cần trả lời ba câu hỏi mà người vận hành sẽ gặp: lưu lượng tới
ClusterIP thực sự được xử lý ở đâu; cluster có dùng được IPv6 không; và có nên chuyển Ingress sang
Gateway API ngay không. Mỗi câu trả lời phải dựa trên phản ứng thật của cluster.

**Yêu cầu:**

1. kube-proxy: số Pod không đổi qua lần đổi CNI; đọc mode đang chạy; tìm ClusterIP của Service `web`
   trong rule của một worker; ClusterIP của Service DNS **bằng** giá trị ghi ở B2.
2. `kubectl proxy` gọi được API mà `curl` không gửi token; apiserver proxy tới Service `som` ở
   `ing-thu` trả `som-ok`. ⭐ Thử cùng dạng URL với Service `web` của `bookstore`, ghi kết quả và giải
   thích bằng bài 84.
3. Dual-stack: đọc `ipFamilyPolicy`, `ipFamilies`, `clusterIPs` của Service `api` và `db`. Ở
   `ing-thu`: Service `RequireDualStack` bị từ chối, `PreferDualStack` quay về IPv4. ⭐ Headless không
   selector, không khai `ipFamilyPolicy`: đọc giá trị API server tự đặt.
4. Gateway API: API group `gateway.networking.k8s.io` không tồn tại; một `HTTPRoute` bị từ chối ngay ở
   bước ánh xạ kiểu; trong khi Traefik đã mang theo API riêng dạng CRD.

**Ràng buộc:** object thử nghiệm chỉ ở `ing-thu`, xóa khi xong. Không cài CRD của Gateway API.

**Đọc lại:**

- Bài [164](../164-proxies-vi.md#proxy-proxies) — năm loại proxy.
- Bài [85](../85-dual-stack-vi.md): [*Service*](../85-dual-stack-vi.md#service-services),
  [*Headless Service không có selector*](../85-dual-stack-vi.md#headless-service-không-có-selector-headless-services-without-selector).
- Bài [13](../13-gateway-vi.md): đoạn mở đầu (add-on),
  [*Mô hình tài nguyên*](../13-gateway-vi.md#mô-hình-tài-nguyên-resource-model),
  [*Di chuyển từ Ingress*](../13-gateway-vi.md#di-chuyển-từ-ingress-migrating-from-ingress).
- Bài [157](../157-networking-vi.md#cluster-network-ipfamilies) — các kiểu mạng cluster.

**Bàn giao:** `docs/adr/NNNN-proxy-dual-stack-gateway.md`; `~/lab-evidence/5b/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` kube-proxy vẫn đủ ba Pod; log nói rõ dataplane; ClusterIP của `web` có trong rule của
  `lab-k8s-worker1`; ClusterIP của DNS không đổi qua lần đổi CNI.
- `PASS:` `kubectl proxy` trả `/version` không cần token; apiserver proxy tới `som` trả `som-ok`.
- `PASS:` Service `api` là `SingleStack`, `IPv4`, một `clusterIPs`, `clusterIP` bằng phần tử đầu;
  Service `db` headless cũng `SingleStack`.
- `PASS:` `RequireDualStack` bị từ chối với lỗi nhắc dual-stack; `PreferDualStack` được tạo với đúng
  một họ `IPv4`.
- `PASS:` (⭐) headless không selector được API server đặt `RequireDualStack`.
- `PASS:` không có API group Gateway; `HTTPRoute` bị từ chối; có `IngressRoute` của Traefik.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-proxy-dual-stack-gateway.md`:

1. Kể năm loại proxy của bài 164, mỗi loại chạy ở đâu, và ba loại bạn đã chạm trong sprint này.
2. ClusterIP của `web` không gắn với interface nào. Nó tồn tại ở đâu, và vì sao đổi CNI không đụng tới
   kube-proxy?
3. (⭐) Apiserver proxy tới `web` của `bookstore` cho kết quả gì? Lưu lượng đó xuất phát từ đâu, và
   ngoại lệ "kết nối từ node của Pod" của bài 84 có áp cho nó không?
4. `PreferDualStack` và `RequireDualStack` khác nhau thế nào **trên cluster của bạn**? Manifest copy từ
   tài liệu của một cluster dual-stack sẽ hỏng ở bước nào?
5. Dự án có nên chuyển Ingress `bookstore` sang Gateway API ngay không? Nếu chuyển, những object nào
   thay Ingress, ai sở hữu từng object theo vai trò, và vì sao không chạy song song trên cùng object?

<details>
<summary>Gợi ý</summary>

1. Mode kube-proxy: ConfigMap `kube-proxy` trong `kube-system` (khóa `config.conf`, dòng `mode:`). Rỗng
   nghĩa là dùng mặc định của phiên bản đang chạy — nên đọc thêm log của Pod kube-proxy.
2. Rule của node: `sudo iptables-save -t nat` (hoặc `sudo nft list ruleset`) trên worker.
3. Đường dẫn apiserver proxy tới Service: `/api/v1/namespaces/<ns>/services/<tên>:<tên cổng>/proxy/`.
4. Một kind lạ bị `kubectl` từ chối trước cả khi gửi lên server, kể cả với `--dry-run=server`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau B6, khi hệ thống đang chạy</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5b
jp() { kubectl -n "$1" get "$2" -o jsonpath="$3" 2>/dev/null; }

# 1. kube-proxy
kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' | grep -E '^mode:' > "$EV/b7-mode-cm.txt"
kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=200 2>/dev/null \
  | grep -iE 'proxier|proxy mode|using .*proxy' | head -5 > "$EV/b7-mode-log.txt"
CIP="$(jp "$NS" svc/web '{.spec.clusterIP}')"
{ [ "$(kubectl -n kube-system get pods -l k8s-app=kube-proxy --field-selector=status.phase=Running --no-headers | wc -l)" -eq 3 ] \
  && grep -qiE 'iptables|nftables|ipvs' "$EV/b7-mode-log.txt" \
  && ssh lab-k8s-worker1 "sudo iptables-save -t nat 2>/dev/null | grep -qF '$CIP' || sudo nft list ruleset 2>/dev/null | grep -qF '$CIP'" \
  && [ "$(jp kube-system svc/kube-dns '{.spec.clusterIP}')" = "$(cat "$EV/b2-dns-clusterip.txt")" ]; } \
  && echo "PASS: kube-proxy 3 Pod; dataplane trong log; ClusterIP $CIP trong rule worker1; ClusterIP DNS khong doi"

# 2. kubectl proxy và apiserver proxy
kubectl proxy --port=18001 > /tmp/lab5b-proxy.log 2>&1 & PF=$!
for i in $(seq 1 20); do curl -fsS http://127.0.0.1:18001/version >/dev/null 2>&1 && break; sleep 1; done
curl -s http://127.0.0.1:18001/version > "$EV/b7-kubectl-proxy.txt"
curl -s --max-time 15 http://127.0.0.1:18001/api/v1/namespaces/ing-thu/services/som:http/proxy/ > "$EV/b7-apiserver-proxy.txt"
{ grep -q 'gitVersion' "$EV/b7-kubectl-proxy.txt" && grep -qx 'som-ok' "$EV/b7-apiserver-proxy.txt"; } \
  && echo 'PASS: kubectl proxy goi API khong token; apiserver proxy toi som tra som-ok'
# ⭐ cùng dạng URL cho web của bookstore — chỉ ghi lại, giải thích trong ADR
PN="$(jp "$NS" svc/web '{.spec.ports[0].name}')"
curl -s -o /dev/null -w '%{http_code}\n' --max-time 40 \
  "http://127.0.0.1:18001/api/v1/namespaces/$NS/services/web${PN:+:$PN}/proxy/" | tee "$EV/b7-apiserver-proxy-web.txt"
kill "$PF"

# 3. Dual-stack
F='{.spec.ipFamilyPolicy}|{.spec.ipFamilies[*]}|{.spec.clusterIPs[*]}|{.spec.clusterIP}'
A="$(jp "$NS" svc/api "$F")"; D="$(jp "$NS" svc/db "$F")"
echo "api: $A" > "$EV/b7-dual-stack.txt"; echo "db: $D" >> "$EV/b7-dual-stack.txt"
IFS='|' read -r POL FAM IPS IP1 <<< "$A"
{ [ "$POL" = 'SingleStack' ] && [ "$FAM" = 'IPv4' ] && [ "$(echo "$IPS" | wc -w)" -eq 1 ] && [ "$IPS" = "$IP1" ] \
  && echo "$D" | grep -q '^SingleStack|IPv4|None|None$'; } \
  && echo 'PASS: api SingleStack IPv4 mot clusterIPs; db headless cung SingleStack'

cat > ~/lab-work/5b/ds-thu.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: ds-require
  namespace: ing-thu
spec:
  ipFamilyPolicy: RequireDualStack
  selector:
    app: som
  ports:
  - port: 80
    targetPort: 8080
EOF
if kubectl apply -f ~/lab-work/5b/ds-thu.yaml 2> "$EV/b7-loi-require.txt"; then
  echo 'FAIL: cluster single-stack ma van tao duoc RequireDualStack'
else
  grep -qi 'dual-stack' "$EV/b7-loi-require.txt" && echo 'PASS: RequireDualStack bi tu choi, loi nhac dual-stack'
fi
sed -e 's/ds-require/ds-prefer/' -e 's/RequireDualStack/PreferDualStack/' ~/lab-work/5b/ds-thu.yaml | kubectl apply -f -
[ "$(jp ing-thu svc/ds-prefer '{.spec.ipFamilyPolicy}|{.spec.ipFamilies[*]}')" = 'PreferDualStack|IPv4' ] \
  && echo 'PASS: PreferDualStack duoc tao, quay ve dung mot ho IPv4'
# ⭐ headless không selector, không khai ipFamilyPolicy
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: ds-headless
  namespace: ing-thu
spec:
  clusterIP: None
  ports:
  - port: 80
EOF
jp ing-thu svc/ds-headless '{.spec.ipFamilyPolicy}|{.spec.ipFamilies[*]}' | tee -a "$EV/b7-dual-stack.txt"; echo
[ "$(jp ing-thu svc/ds-headless '{.spec.ipFamilyPolicy}')" = 'RequireDualStack' ] \
  && echo 'PASS: headless khong selector duoc dat RequireDualStack'
kubectl -n ing-thu delete svc ds-prefer ds-headless --ignore-not-found

# 4. Gateway API
kubectl api-resources --api-group=gateway.networking.k8s.io > "$EV/b7-gateway-api.txt" 2>&1
cat > ~/lab-work/5b/httproute-thu.yaml <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bookstore
  namespace: ing-thu
spec:
  parentRefs:
  - name: khong-ton-tai
  hostnames:
  - som.lab.local
  rules:
  - backendRefs:
    - name: som
      port: 80
EOF
kubectl apply --dry-run=server -f ~/lab-work/5b/httproute-thu.yaml > "$EV/b7-httproute.txt" 2>&1
{ ! grep -qi 'gateway' "$EV/b7-gateway-api.txt" && grep -qi 'no matches for kind' "$EV/b7-httproute.txt" \
  && kubectl api-resources --api-group=traefik.io | grep -q IngressRoute; } \
  && echo 'PASS: chua co Gateway API, HTTPRoute bi tu choi; Traefik da mang CRD IngressRoute rieng'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính cho ADR</summary>

1. `kubectl proxy` — máy người dùng hoặc trong Pod; apiserver proxy — trong tiến trình apiserver;
   kube-proxy — trên mỗi node; load balancer trước nhiều apiserver — ngoài cluster, chỉ có ở control
   plane HA; cloud load balancer — của nhà cung cấp cloud. Sprint này chạm ba loại đầu, và gặp loại 5
   ở dạng `<pending>` khi đọc giá trị mặc định của chart Traefik.
2. Trong rule NAT mà kube-proxy sinh trên **mỗi** node. kube-proxy lo Pod-với-Service; CNI lo
   Pod-với-Pod (bài 157). Vì vậy kube-proxy giữ nguyên ba Pod, và ClusterIP của DNS — do apiserver
   cấp — không đổi.
3. Thường là lỗi kết nối hoặc timeout (`503`). Apiserver chạy `hostNetwork` trên master; nó gọi thẳng
   IP của một Pod `web` trên worker, nên nguồn là master — không phải node đang chạy Pod đó — và
   không phải Pod Traefik. Ngoại lệ của bài 84 chỉ dành cho node của chính Pod. Zero-trust đóng cả
   "bastion tích hợp" của apiserver; muốn mở phải viết rule `ipBlock` cho node, một quyết định có chủ
   đích.
4. Bài 85: `PreferDualStack` quay về single-stack; `RequireDualStack` làm việc tạo Service qua API
   thất bại. Manifest copy sẽ chết ngay ở `kubectl apply`, với lỗi bạn đã lưu. Headless không selector
   là ngoại lệ: không khai gì thì mặc định là `RequireDualStack` mà vẫn tạo được, vì nó không cần cấp
   IP nào.
5. Chưa. Gateway API là add-on gồm CRD cộng một bản hiện thực; cluster chưa có cả hai — HTTPRoute bị
   từ chối vì cluster **không biết kiểu**, khác Ingress ở B4 (có kiểu, thiếu người thực thi). Nếu
   chuyển: GatewayClass (người vận hành hạ tầng) ← Gateway với listener HTTPS (người vận hành cluster)
   ← HTTPRoute `bookstore` trỏ Service `web` (đội ứng dụng). Gateway API không bao gồm kiểu Ingress,
   nên chuyển một lần dứt điểm chứ không chạy song song trên cùng object. Ghi điều kiện chuyển: có CRD
   và bản hiện thực được chốt version ở A1.4, có lab cho CRD (giai đoạn 14).

</details>

## B8. [BS-5B-08] ⭐ Điều tra: Traefik so khớp path có đúng đặc tả không

**Loại:** Điều tra — tùy chọn.

**Bối cảnh:** bài 11 và bài 12 cùng cảnh báo các controller hoạt động hơi khác nhau. Trước khi dự án
thêm path thứ hai vào Ingress, đội muốn biết Traefik — với cấu hình đang chạy — so khớp path và host
có đúng như đặc tả không, và lệch ở đâu.

**Yêu cầu:** dựng ba backend phân biệt được và một Ingress thử nghiệm `duong-dan` ở `ing-thu` (file chép
nguyên văn ở khối dưới). Đo: `Prefix` khớp path con; path khớp dài nhất thắng; `Exact` không khớp dấu
gạch chéo cuối; host phải khớp; wildcard `*.thu.lab.local` khớp một nhãn; thiếu `pathType` bị từ chối.
Đo thêm `/blogxyz` và `a.b.thu.lab.local` — hai chỗ đặc tả nói **không** khớp — rồi giải thích kết quả
bằng cấu hình đang chạy của Traefik. Xóa object thử nghiệm khi xong.

**Đọc lại:** bài [11](../11-ingress-vi.md): [*Các loại path*](../11-ingress-vi.md#các-loại-path-path-types),
[*Ví dụ*](../11-ingress-vi.md#ví-dụ-examples),
[*Khớp nhiều path*](../11-ingress-vi.md#khớp-nhiều-path-multiple-matches),
[*Wildcard cho hostname*](../11-ingress-vi.md#wildcard-cho-hostname-hostname-wildcards);
[Traefik — Kubernetes Ingress provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/),
tùy chọn `strictPrefixMatching`.

**Nghiệm thu:**

- `PASS:` `/` → backend gốc; `/blog/bai-1` → `blog`; `/blog/` → `blog` (dài nhất thắng).
- `PASS:` `/chinhxac` → `chinhxac`; `/chinhxac/` không về `chinhxac`; host lạ không về backend nào của
  rule.
- `PASS:` `abc.thu.lab.local` khớp wildcard; `thu.lab.local` không khớp.
- `PASS:` `/blogxyz` cho đúng kết quả mà cấu hình `strictPrefixMatching` đang chạy quy định.
- `PASS:` manifest thiếu `pathType` bị API server từ chối.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-so-khop-path.md`: Traefik lệch đặc tả ở đâu với
cấu hình mặc định? Dự án có nên bật chế độ khớp chặt trong `platform/traefik/values.yaml` không, và
cái giá là gì?

<details>
<summary>Script nghiệm thu và lời giải tham khảo</summary>

```bash
cat > ~/lab-work/5b/duong-dan.yaml <<'EOF'
# Ba backend busybox httpd. Mỗi backend chỉ có đúng các file mà phép đo cần, nội dung là tên của nó,
# nên chuỗi trả về cho biết backend nào đã nhận request. httpd trả index.html cho đường dẫn thư mục.
apiVersion: v1
kind: Pod
metadata: {name: dd-goc, namespace: ing-thu, labels: {app: dd-goc}}
spec:
  terminationGracePeriodSeconds: 1
  containers:
  - name: httpd
    image: busybox:1.37
    command: ["sh", "-c", "set -e; mkdir -p /www/chinhxac; for f in index.html blogxyz chinhxac/index.html; do echo goc-ok > /www/$f; done; exec httpd -f -p 8080 -h /www"]
---
apiVersion: v1
kind: Pod
metadata: {name: dd-blog, namespace: ing-thu, labels: {app: dd-blog}}
spec:
  terminationGracePeriodSeconds: 1
  containers:
  - name: httpd
    image: busybox:1.37
    command: ["sh", "-c", "set -e; mkdir -p /www/blog; for f in index.html blog/index.html blog/bai-1 blogxyz; do echo blog-ok > /www/$f; done; exec httpd -f -p 8080 -h /www"]
---
apiVersion: v1
kind: Pod
metadata: {name: dd-chinhxac, namespace: ing-thu, labels: {app: dd-chinhxac}}
spec:
  terminationGracePeriodSeconds: 1
  containers:
  - name: httpd
    image: busybox:1.37
    command: ["sh", "-c", "set -e; mkdir -p /www; echo chinhxac-ok > /www/chinhxac; exec httpd -f -p 8080 -h /www"]
---
apiVersion: v1
kind: Service
metadata: {name: dd-goc, namespace: ing-thu}
spec: {selector: {app: dd-goc}, ports: [{port: 80, targetPort: 8080}]}
---
apiVersion: v1
kind: Service
metadata: {name: dd-blog, namespace: ing-thu}
spec: {selector: {app: dd-blog}, ports: [{port: 80, targetPort: 8080}]}
---
apiVersion: v1
kind: Service
metadata: {name: dd-chinhxac, namespace: ing-thu}
spec: {selector: {app: dd-chinhxac}, ports: [{port: 80, targetPort: 8080}]}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: {name: duong-dan, namespace: ing-thu}
spec:
  ingressClassName: traefik
  rules:
  - host: duongdan.lab.local
    http:
      paths:
      - {path: /, pathType: Prefix, backend: {service: {name: dd-goc, port: {number: 80}}}}
      - {path: /blog, pathType: Prefix, backend: {service: {name: dd-blog, port: {number: 80}}}}
      - {path: /chinhxac, pathType: Exact, backend: {service: {name: dd-chinhxac, port: {number: 80}}}}
  - host: "*.thu.lab.local"
    http:
      paths:
      - {path: /, pathType: Prefix, backend: {service: {name: dd-blog, port: {number: 80}}}}
EOF
kubectl apply -f ~/lab-work/5b/duong-dan.yaml
kubectl -n ing-thu wait --for=condition=Ready pod/dd-goc pod/dd-blog pod/dd-chinhxac --timeout=180s
sleep 5

EV=~/lab-evidence/5b
W1="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
goi() { curl -s --max-time 5 -H "Host: $1" "http://$W1:30081$2"; }
for P in / /blog/bai-1 /blog/ /blogxyz /chinhxac /chinhxac/; do echo "$P -> $(goi duongdan.lab.local "$P")"; done \
  > "$EV/b8-so-khop.txt"
cat "$EV/b8-so-khop.txt"

{ [ "$(goi duongdan.lab.local /)" = 'goc-ok' ] && [ "$(goi duongdan.lab.local /blog/bai-1)" = 'blog-ok' ] \
  && [ "$(goi duongdan.lab.local /blog/)" = 'blog-ok' ]; } \
  && echo 'PASS: Prefix khop path con; path khop dai nhat thang'
{ [ "$(goi duongdan.lab.local /chinhxac)" = 'chinhxac-ok' ] && [ "$(goi duongdan.lab.local /chinhxac/)" != 'chinhxac-ok' ] \
  && ! goi khac.lab.local / | grep -qE '^(goc|blog|chinhxac)-ok$'; } \
  && echo 'PASS: Exact khong khop dau gach cheo cuoi; host la khong dung rule nao'
{ [ "$(goi abc.thu.lab.local /)" = 'blog-ok' ] && [ "$(goi thu.lab.local /)" != 'blog-ok' ]; } \
  && echo 'PASS: wildcard *.thu.lab.local khop abc, khong khop thu.lab.local'

STRICT="$(helm get values traefik -n traefik --all -o json | grep -o '"strictPrefixMatching":[a-z]*' | head -1 | cut -d: -f2)"
[ "$STRICT" = 'true' ] && WANT='goc-ok' || WANT='blog-ok'
X="$(goi duongdan.lab.local /blogxyz)"
echo "strictPrefixMatching=${STRICT:-khong khai} /blogxyz=$X a.b.thu=$(goi a.b.thu.lab.local /)" | tee -a "$EV/b8-so-khop.txt"
[ "$X" = "$WANT" ] && echo "PASS: /blogxyz -> $X, dung voi strictPrefixMatching=${STRICT:-false}"

sed '/pathType: Exact/s/, pathType: Exact//' ~/lab-work/5b/duong-dan.yaml > ~/lab-work/5b/duong-dan-thieu.yaml
kubectl apply --dry-run=server -f ~/lab-work/5b/duong-dan-thieu.yaml >/dev/null 2>&1 \
  && echo 'FAIL: thieu pathType van qua validation' || echo 'PASS: path khong khai pathType bi tu choi'
kubectl delete -f ~/lab-work/5b/duong-dan.yaml --wait=true
```

Ý chính: chart mặc định để `strictPrefixMatching` tắt — Traefik so `Prefix` theo **chuỗi**, nên
`/blogxyz` rơi vào `/blog`, trái bảng ví dụ của bài 11 (`/aaa/bbb` không khớp `/aaa/bbbxyz`). Kết quả
`a.b.thu.lab.local` cũng là hành vi riêng của controller — đối chiếu với bảng wildcard của bài 11 và
ghi lại. Bật chế độ khớp chặt thì Ingress cư xử đúng đặc tả, giá phải trả là kiểm lại mọi Ingress đang
dựa vào kiểu khớp chuỗi. Dự án hiện chỉ có path `/` nên chưa bị ảnh hưởng; ghi điều kiện xem lại khi
thêm path.

</details>

## B9. Đóng sprint: cleanup, bảy tầng gate và snapshot `02-net-ready`

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 7, rồi đưa cluster về trạng
thái **chỉ còn hạ tầng** — Calico, Traefik — chứng minh nó khỏe, và chụp `02-net-ready`.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 5b — kiểm thử khói **tích lũy**, chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 5b, cluster ở mốc 02-net-ready (Calico, Traefik). Kiểm mọi hợp
# đồng còn hiệu lực tới sprint 7: controller (4a, 4b), Service và DNS (5a), NetworkPolicy zero-trust có
# hiệu lực và đường vào duy nhất là HTTPS qua Ingress (5b). Script không in bí mật.
set -u
cd "$(dirname "$0")/.."
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
HOST=bookstore.lab.local
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
has_cm() {  # $1 = các tên ConfigMap workload tham chiếu; $2 = tên theo hợp đồng (từ 5a có thể kèm hậu tố hash)
  for N in $1; do case "$N" in "$2"|"$2"-*) k get configmap "$N" >/dev/null 2>&1 && return 0 ;; esac; done
  return 1
}

# 1. Namespace, object cấp cluster, hạ tầng của mốc 02-net-ready
[ "$(kubectl get namespace "$NS" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = 'bookstore' ] \
  && pass 'namespace bookstore mang label du an' || fail 'namespace bookstore thieu hoac thieu label'
PRIO="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
[ -n "$PRIO" ] && pass "co PriorityClass bookstore-critical ($PRIO)" || fail 'thieu PriorityClass bookstore-critical'
NODES="$(kubectl get nodes --no-headers | wc -l)"
R="$(kubectl -n kube-system get daemonset calico-node -o jsonpath='{.status.numberReady}' 2>/dev/null)"
[ "${R:-0}" -eq "$NODES" ] && pass "Calico Ready tren $R/$NODES node" || fail "Calico ${R:-0}/$NODES node"
[ "$(kubectl get ingressclass traefik -o jsonpath='{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}' 2>/dev/null)" = 'true' ] \
  && pass 'IngressClass traefik la mac dinh' || fail 'thieu IngressClass traefik mac dinh'

# 2. Cấu hình và bí mật (3b, 5b), PDB (3c)
has_cm "$(jp deployment/web '{.spec.template.spec.volumes[*].configMap.name}')" web-config \
  && pass 'web dung ConfigMap web-config' || fail 'web khong tham chieu ConfigMap web-config ton tai'
has_cm "$(jp deployment/api '{.spec.template.spec.containers[*].envFrom[*].configMapRef.name}')" api-config \
  && pass 'api dung ConfigMap api-config' || fail 'api khong tham chieu ConfigMap api-config ton tai'
has_cm "$(jp statefulset/db '{.spec.template.spec.volumes[*].configMap.name}')" db-init \
  && pass 'db dung ConfigMap db-init' || fail 'db khong tham chieu ConfigMap db-init ton tai'
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials du hai key' || fail 'Secret db-credentials thieu key'
D="$(jp secret/bookstore-tls '{.data}')"
{ [ "$(jp secret/bookstore-tls '{.type}')" = 'kubernetes.io/tls' ] && echo "$D" | grep -q '"tls.crt"' \
  && echo "$D" | grep -q '"tls.key"'; } \
  && pass 'Secret bookstore-tls kieu TLS, du tls.crt va tls.key' || fail 'Secret bookstore-tls sai kieu hoac thieu key'
for P in web api; do
  k get pdb "$P" >/dev/null 2>&1 && pass "co PodDisruptionBudget $P" || fail "thieu PodDisruptionBudget $P"
done

# 3. Controller cho mọi thành phần (4a, 4b) và tài nguyên (3c)
[ "$(jp statefulset/db '{.status.readyReplicas}')" = '1' ] && pass 'StatefulSet db 1/1 Ready' || fail 'StatefulSet db chua Ready'
[ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && pass 'db-0 tren lab-k8s-worker2' || fail 'db-0 khong o lab-k8s-worker2'
[ "$(jp pod/db-0 '{.status.qosClass}')" = 'Guaranteed' ] && pass 'db-0 QoS Guaranteed' || fail 'db-0 khong Guaranteed'
[ -n "$PRIO" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$PRIO" ] && pass 'db-0 dung bookstore-critical' || fail 'db-0 sai priority'
for X in web api; do
  W="$(jp "deployment/$X" '{.spec.replicas}')"; RR="$(jp "deployment/$X" '{.status.readyReplicas}')"
  { [ "${W:-0}" -ge 2 ] && [ "${RR:-0}" = "$W" ]; } && pass "Deployment $X $RR/$W Ready" || fail "Deployment $X ${RR:-0}/${W:-0} Ready"
done
ORPHAN="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"
MISS="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container co requests va limits' || fail "container thieu tai nguyen: $(echo $MISS)"

# 4. Dữ liệu và migration (4b)
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach (khoi tao + migration)" || fail "db co ${N:-0} cuon sach, can >= 8"
if k get job db-migrate >/dev/null 2>&1; then
  [ "$(jp job/db-migrate '{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ] \
    && pass 'Job db-migrate Complete' || fail 'Job db-migrate chua Complete'
else
  pass 'Job db-migrate da duoc TTL don, ket qua nam trong db'
fi

# 5. Service và DNS (5a); web về ClusterIP (5b)
svc() {  # $1 tên; $2 cổng Service; $3 các targetPort chấp nhận, ngăn bằng |
  local P T; P="$(jp "service/$1" '{.spec.ports[0].port}')"; T="$(jp "service/$1" '{.spec.ports[0].targetPort}')"
  { [ "$P" = "$2" ] && echo "|$3|" | grep -qF "|$T|"; } && pass "Service $1 cong $P -> $T" || fail "Service $1 sai cong ($P -> $T)"
}
svc web 80 'http|8080'; svc api 3000 'http|3000'; svc db 5432 'postgres|5432'; svc node-agent 9100 'http|9100'
{ [ "$(jp service/web '{.spec.type}')" = 'ClusterIP' ] && [ -z "$(jp service/web '{.spec.ports[*].nodePort}')" ]; } \
  && pass 'Service web la ClusterIP, khong con NodePort' || fail 'Service web van mo NodePort'
[ "$(jp service/db '{.spec.clusterIP}')" = 'None' ] && pass 'Service db headless' || fail 'Service db khong headless'
[ "$(jp service/node-agent '{.spec.internalTrafficPolicy}')" = 'Local' ] \
  && pass 'Service node-agent internalTrafficPolicy Local' || fail 'Service node-agent khong Local'
DB_IP="$(jp pod/db-0 '{.status.podIP}')"

# 6. Các luồng ma trận cho phép (5b)
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
WNODE="$(jp "pod/$WEB" '{.spec.nodeName}')"
get() { k exec "$WEB" -c status-writer -- wget -qO- -T 5 "$1" 2>/dev/null; }
# Hỏi DNS của cluster từ Pod khác: trong chính db-0, tên này nằm sẵn ở /etc/hosts do kubelet ghi (5a).
k exec "$WEB" -c status-writer -- nslookup "db-0.db.$NS.svc.cluster.local" 2>/dev/null | grep -qwF "$DB_IP" \
  && pass 'ten db-0.db phan giai ve IP cua db-0' || fail 'ten db-0.db khong phan giai dung'
T="$(get http://api:3000/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "web -> api:3000 theo ten, /books tra $T cuon" || fail "web -> api tra $T cuon"
get http://127.0.0.1:8080/ | grep -q 'Bookstore' && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
get http://127.0.0.1:8080/status.json | grep -Eq "\"pod\" *: *\"$WEB\"" \
  && pass 'web phuc vu status.json dung ten Pod' || fail 'status.json sai hoac thieu'
T="$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "nginx proxy /api/ toi api ($T cuon)" || fail "nginx /api/ tra $T cuon"
get http://node-agent:9100/node.json | grep -Eq "\"node\" *: *\"$WNODE\"" \
  && pass "web doc node-agent cung node ($WNODE)" || fail 'web khong doc duoc node-agent cung node'

# 7. CronJob db-backup (4b): chính sách và một lần chạy thử — luồng backup -> db
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = 'Forbid' ] && pass 'db-backup Forbid' || fail 'db-backup khong Forbid'
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

# 8. DaemonSet node-agent (4b)
R="$(jp daemonset/node-agent '{.status.numberReady}')"
[ "${R:-0}" -eq "$NODES" ] && pass "node-agent Ready tren $R/$NODES node" || fail "node-agent ${R:-0}/$NODES node"
BAD=0
for L in $(k get pod -l "$SEL,app.kubernetes.io/component=agent" -o jsonpath='{range .items[*]}{.spec.nodeName}={.status.podIP}{"\n"}{end}' 2>/dev/null); do
  get "http://${L##*=}:9100/node.json" | grep -Eq "\"node\" *: *\"${L%%=*}\"" || { echo "FAIL: node-agent tren ${L%%=*} sai /node.json"; BAD=1; }
done
[ "$BAD" -eq 0 ] && pass 'moi node-agent tra /node.json dung ten node' || FAIL=1

# 9. Các luồng ma trận KHÔNG cho phép phải bị chặn (5b)
API_IP="$(k get pod -l "$SEL,app.kubernetes.io/component=api" --field-selector=status.phase=Running -o jsonpath='{.items[0].status.podIP}' 2>/dev/null)"
if k exec "$WEB" -c status-writer -- wget -q -O /dev/null -T 5 "http://${API_IP}:3001/live" >/dev/null 2>&1; then
  fail 'web toi duoc cong admin 3001 cua api - ma tran rong hon muc 3.5'
else
  pass 'web -> api:3001 bi chan, chi cong 3000 duoc mo'
fi
if k exec db-0 -c postgres -- getent hosts web >/dev/null 2>&1; then
  if k exec db-0 -c postgres -- timeout 5 bash -c 'exec 3<>/dev/tcp/web/80' >/dev/null 2>&1; then
    fail 'db-0 mo duoc ket noi toi web:80 - chieu di chua bi chan mac dinh'
  else
    pass 'db-0 -> web:80 bi chan, DNS van phan giai'
  fi
else
  fail 'db-0 khong phan giai duoc ten web - DNS bi chan?'
fi

# 10. Đường vào duy nhất: HTTPS qua Traefik bằng certificate của dự án (5b)
ING='{.spec.rules[0].host}|{.spec.ingressClassName}|{.spec.tls[0].hosts[0]}|{.spec.tls[0].secretName}|{.spec.rules[0].http.paths[0].backend.service.name}'
[ "$(jp ingress/bookstore "$ING")" = "$HOST|traefik|$HOST|bookstore-tls|web" ] \
  && pass "Ingress bookstore: $HOST, class traefik, TLS bookstore-tls, ve web" || fail 'Ingress bookstore sai host, class, TLS hoac backend'
NP="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}' 2>/dev/null)"
if [ -s secrets/tls.crt ] && [ -n "$NP" ]; then
  openssl x509 -in secrets/tls.crt -noout -checkend 0 >/dev/null && pass 'certificate trong secrets/ con han' \
    || fail 'certificate trong secrets/ da het han (TD-11)'
  for N in lab-k8s-worker1 lab-k8s-worker2; do
    IP="$(kubectl get node "$N" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
    C="curl -fsS --max-time 10 --cacert secrets/tls.crt --resolve $HOST:$NP:$IP"
    $C "https://$HOST:$NP/" 2>/dev/null | grep -q 'Bookstore' \
      && pass "HTTPS $HOST qua $N:$NP tra trang chu" || fail "HTTPS qua $N khong tra trang chu"
    T="$($C "https://$HOST:$NP/api/books" 2>/dev/null | grep -o '"title"' | wc -l)"
    [ "$T" -ge 8 ] && pass "HTTPS /api/books qua $N tra $T cuon" || fail "HTTPS /api/books qua $N tra $T cuon"
  done
else
  fail 'thieu secrets/tls.crt hoac NodePort websecure cua Traefik'
fi

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

2. Cập nhật `docs/tech-debt.md`: `TD-09` đã trả (B5); thêm `TD-11` (B6). Ghi chú ở `TD-07`: sprint này
   không đổi gì. Kiểm `README.md` của repo có URL truy cập.

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/5b/b9-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/5b/b9-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/5b/b9-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-5b: sprint 7 — Calico, NetworkPolicy zero-trust, Ingress HTTPS'
git tag lab-5b
git bundle create ~/bookstore-lab-5b.bundle --all && git bundle verify ~/bookstore-lab-5b.bundle
```

Chép `~/bookstore-lab-5b.bundle` về máy host bằng `scp` **trước** khi đi tiếp — bước 8 tắt máy và
chụp snapshot.

5. Dọn object thử nghiệm, rồi gỡ dự án:

```bash
kubectl delete namespace ing-thu np-ngoai net-thu --ignore-not-found --wait=true --timeout=300s
kubectl -n bookstore delete pod -l '!app.kubernetes.io/part-of' --ignore-not-found --wait=true
kubectl -n traefik delete pod np-la --ignore-not-found --wait=true
rm -rf ~/lab-work/5b
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate: repo đủ, cluster **chỉ còn hạ tầng**:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd ~/bookstore
SEL='app.kubernetes.io/part-of=bookstore'
on() { if [ "$1" = "$(hostname)" ]; then bash -c "$2"; else ssh "$1" "$2"; fi; }
. platform/versions.env

test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-5b' && echo 'PASS: HEAD mang tag lab-5b'
test -s ~/bookstore-lab-5b.bundle && echo 'PASS: co bundle'
{ ls k8s/app/network/*.yaml >/dev/null 2>&1 && test -f k8s/app/ingress/ingress.yaml \
  && test -f platform/versions.env && test -f platform/calico/README.md \
  && test -f platform/traefik/values.yaml && test -f platform/traefik/README.md \
  && [ -z "$(find k8s -name '*.tpl')" ]; } && echo 'PASS: bo cuc repo dung muc 5.5 cho sprint 7'
{ grep -E 'TD-09.*đã trả' docs/tech-debt.md >/dev/null && grep -q 'TD-11' docs/tech-debt.md; } \
  && echo 'PASS: tech-debt.md: TD-09 da tra, co TD-11'
BAD=0
for F in networkpolicy-tren-flannel mang-cluster-va-chon-cni quy-trinh-doi-cni ingress-controller \
         zero-trust https-ingress proxy-dual-stack-gateway; do
  ls docs/adr/*-"$F".md >/dev/null 2>&1 || { echo "FAIL: thieu docs/adr/*-$F.md"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR cua sprint 7'

{ [ "$(sl kubectl get namespace,priorityclass -l "$SEL" --no-headers)" -eq 0 ] \
  && [ "$(sl kubectl get all,configmaps,secrets,networkpolicies,ingresses -A -l "$SEL" --no-headers)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an'
{ [ "$(sl kubectl get networkpolicy -A --no-headers)" -eq 0 ] \
  && [ "$(sl kubectl get ingress -A --no-headers)" -eq 0 ]; } \
  && echo 'PASS: cluster khong con NetworkPolicy hay Ingress nao'
[ "$(kubectl get ns -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | sort | tr '\n' ' ')" \
  = 'default kube-node-lease kube-public kube-system traefik ' ] && echo 'PASS: chi con namespace he thong va traefik'
{ [ "$(kubectl -n kube-system get ds calico-node -o jsonpath='{.status.numberReady}')" -eq 3 ] \
  && kubectl -n kube-system wait --for=condition=Available deployment/calico-kube-controllers --timeout=60s >/dev/null; } \
  && echo 'PASS: Calico chay du ba node'
{ kubectl -n traefik wait --for=condition=Available deployment/traefik --timeout=60s >/dev/null \
  && [ "$(kubectl get ingressclass --no-headers | wc -l)" -eq 1 ] \
  && [ "$(kubectl get ingressclass traefik -o jsonpath='{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}')" = 'true' ] \
  && [ "$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.type} {.spec.ports[?(@.name=="web")].nodePort} {.spec.ports[?(@.name=="websecure")].nodePort}')" = 'NodePort 30081 30443' ] \
  && helm list -n traefik | grep -q "traefik-$TRAEFIK_CHART"; } \
  && echo 'PASS: Traefik san sang, IngressClass traefik mac dinh duy nhat, NodePort 30081/30443'
BAD=0
for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  on "$N" 'test ! -e /etc/cni/net.d/10-flannel.conflist && test -f /etc/cni/net.d/10-calico.conflist \
    && ! ip link show flannel.1 >/dev/null 2>&1' || { echo "FAIL: $N con dau vet CNI cu"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: ca ba node chi con cau hinh CNI cua Calico'
test ! -e ~/lab-work/5b && echo 'PASS: da don lab-work'
```

<a id="b112-chạy-trọn-bảy-tầng-gate-a54"></a>

7. Chạy **đầy đủ bảy tầng** ở [A5.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready),
   từ [tầng 0](LAB-00-MOI-TRUONG-1.35.7.md#a541-tầng-0--prereq-os-còn-đúng-sau-khi-cluster-chạy) tới
   [tầng 6](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet), rồi **chỉ phần dọn
   resource test** của [A5.4.8](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot)
   — không chụp `01-cluster-ready`. Lab này thuộc đúng trường hợp A5.5 nói phải chạy cả bảy tầng: **lab
   vừa đụng vào mạng và CNI**. Hai tầng phải **đọc lại theo CNI mới**:

| Tầng | Câu trong Lab 00 | Đọc lại thế nào sau Lab 5b |
| --- | --- | --- |
| [A5.4.3](LAB-00-MOI-TRUONG-1.35.7.md#a543-tầng-2--node-condition-taint-và-podcidr) | "`kube-proxy` và `kube-flannel-ds` mỗi cái có đúng một Pod `Running` trên mỗi node" | Không còn DaemonSet `kube-flannel-ds`, không còn namespace `kube-flannel`. Thay bằng: `kube-proxy` và `calico-node` (cùng namespace `kube-system`) mỗi cái đúng một Pod `Running` trên mỗi node, cộng Deployment `calico-kube-controllers` sẵn sàng. Ba node vẫn có ba `.spec.podCIDR` khác nhau, nhưng **IP Pod không nhất thiết nằm trong `/24` của node** — xem câu 3 của B3 |
| [A5.4.4](LAB-00-MOI-TRUONG-1.35.7.md#a544-tầng-3--pod-networking-xuyên-node) | Ba nguyên nhân fail: UDP `8472` bị chặn, Flannel bind nhầm interface, `--pod-network-cidr` sai | Đường hầm giữa các node do Calico dựng, không còn là VXLAN cổng `8472`. Ba nguyên nhân đọc lại: lưu lượng đóng gói giữa các node bị chặn, Calico chọn nhầm interface (gate ở B3 bắt lỗi này), IPPool lệch dải Pod |

**PASS:** bảy tầng đều PASS với hai tầng đã đọc lại như trên.

8. Ghi hồ sơ của mốc mới, tắt máy và chụp snapshot:

```bash
{
  date -Is
  echo '=== 02-net-ready ==='
  echo '--- CNI'
  kubectl -n kube-system get ds calico-node -o wide
  kubectl get ippools.crd.projectcalico.org -o wide
  echo '--- Ingress controller'
  helm list -n traefik
  kubectl -n traefik get svc traefik -o wide
  kubectl get ingressclass -o wide
  echo '--- Helm'
  helm version --short
  echo '--- Namespace'
  kubectl get ns
} > ~/lab-evidence/5b/b9-ho-so-02-net-ready.txt
test -s ~/lab-evidence/5b/b9-ho-so-02-net-ready.txt && echo 'PASS: da ghi ho so cua moc 02-net-ready'
```

Tắt ba VM sạch sẽ rồi mới chụp. Chạy trên **từng node** theo thứ tự worker 2 → worker 1 → master:

```bash
sudo shutdown -h now
```

Chờ VMware Workstation hiển thị cả ba VM ở trạng thái *Powered off*. Chụp trên **cả ba VM**: chuột
phải VM → **Snapshot → Take Snapshot** → ô *Name* điền đúng nguyên văn:

```text
02-net-ready
```

Ô *Description* ghi lab đã dựng và ngày chụp, ví dụ
`dung bang LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md, repo lab-5b, chup <ngay>`. Quy tắc tên giống hệt
mốc trước: đúng nguyên văn trên cả ba VM, không hậu tố, không ngày, không đổi hoa thường. **Giữ
nguyên** `01-cluster-ready`.

Verify từ PowerShell trên máy host:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  $ok = ($names -ccontains '02-net-ready') -and ($names -ccontains '01-cluster-ready')
  if ($ok) { "PASS: $f" } else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; mười ba dòng `PASS:` của gate bước 6;
bảy tầng A5.4 PASS với hai dòng đọc lại; dòng `PASS:` của hồ sơ; đúng ba dòng `PASS:` trên máy host.
Thư mục `~/lab-evidence/5b/` **giữ lại**. Dòng hosts `bookstore.lab.local` trên máy host cũng giữ —
các lab sau đi vào hệ thống bằng đúng đường đó. Từ lab sau, **điểm bắt đầu là `02-net-ready`**, không
phải `01-cluster-ready` — xem [chuỗi snapshot](README.md#3-chuỗi-snapshot).

---

## 3. Checkpoint 5b

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] **Nợ #4:** kể hai lần chạy `probe.sh` trên cùng hai file đo — ở B1 và ở B5 — kết quả mỗi lần,
      và chính xác thứ gì đã đổi giữa hai lần.
- [ ] Ba dải IP của cluster bạn, ai cấp từng dải; vì sao đổi CNI làm IP Pod đổi người cấp còn
      ClusterIP của DNS giữ nguyên; vì sao `ip addr` trên worker2 không phải thứ Kubernetes nhìn.
- [ ] Ai nạp CNI plugin, hai thư mục phải đọc; bốn bước đổi CNI theo thứ tự, hậu quả khi đảo bước 1
      với 2 hoặc bỏ bước 4; vì sao gỡ dự án trước và vì sao snapshot chụp sau teardown.
- [ ] Luồng `web` → `api`: những rule nào, ở policy nào, cho chiều nào? Vì sao DNS phải mở riêng, và
      vì sao probe của kubelet không bị chặn?
- [ ] Rule Traefik → `web`: vì sao cổng `8080` chứ không `80`; phép giao khác phép hợp ở đâu; Pod lạ
      trong namespace `traefik` vào được `web` với cách viết nào?
- [ ] `db-0` phân giải được `registry.k8s.io` nhưng không kết nối được tới nó. Giải thích bằng đường
      đi của truy vấn DNS và bằng ma trận.
- [ ] Ingress `som` tạo trước khi có controller: vì sao không chuyển byte nào, vì sao sau khi cài vẫn
      không có class, cơ chế nào điền class khi tạo lại? Vì sao Ingress `bookstore` ghi class tường
      minh?
- [ ] Kể đường đi của `https://bookstore.lab.local:30443/` từ trình duyệt: file hosts, NodePort,
      Traefik, Service `web`, Pod `web`. TLS kết thúc ở đâu, Secret mang kiểu và hai key nào, chặng nào
      plaintext?
- [ ] Vì sao `web` về ClusterIP; vì sao certificate sinh tại chỗ, không vào git; điều gì xảy ra với
      trình duyệt sau khi dựng lại từ bundle (`TD-11`)?
- [ ] Năm loại proxy của bài 164, nơi mỗi loại chạy; ClusterIP của `web` tồn tại ở đâu; kube-proxy có
      định tuyến theo path được không?
- [ ] `SingleStack`, `PreferDualStack`, `RequireDualStack` — phản ứng của API server trên cluster của
      bạn; ngoại lệ của headless không selector.
- [ ] Vì sao `HTTPRoute` bị từ chối còn Ingress thì không; nếu dự án chuyển sang Gateway API, object
      nào thay Ingress `bookstore` và ai sở hữu nó?

Nợ sau lab này — nói được **vì sao**:

- **Đã trả:** nợ lab #4 (NetworkPolicy được thực thi thật) và `TD-09` (lưu lượng giữa các tầng).
- **Mới:** `TD-11` — certificate tự ký, gia hạn bằng tay, đoạn sau Traefik là plaintext; chưa có lab
  trả vì công cụ cấp certificate nằm ngoài chuỗi lab.
- **Còn mở:** nợ lab #1 (VPA chưa có lab; HPA ở Lab 11b), #2 (`volumeClaimTemplates` — Lab 6a), #5 →
  #8; nợ kỹ thuật `TD-04` (dữ liệu trên `emptyDir` — Lab 6a), `TD-06` (`db` khởi động bằng root — Lab
  9b), `TD-07` (quyền debug — Lab 9a), `TD-08` (bản dump chưa có nơi bền — Lab 6a).

### Bài giải thích cuối cùng

Trong vài phút, kể lại đường đi của một request `https://bookstore.lab.local:30443/api/books` từ trình
duyệt trên máy host tới PostgreSQL và ngược lại. Ở mỗi chặng nói rõ: thành phần nào xử lý (file hosts,
kube-proxy với NodePort, Traefik, Service, nginx, PostgREST, PostgreSQL), chặng đó mã hóa hay plaintext,
và **rule NetworkPolicy nào** cho phép nó — ở đầu nguồn và ở đầu đích. Rồi kể lại cùng request đó trên
cluster của B1 (Flannel, NodePort `30080`): chặng nào khác, và vì sao một Pod lạ trên cluster đó chạm
được tới `db`. Kết thúc bằng việc chỉ ra Lab 6a sẽ thay đổi câu chuyện này ở điểm nào.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `git describe` không ra `lab-5a` | Repo ở commit khác, hoặc clone từ bundle cũ | `git checkout lab-5a` hoặc clone lại từ `~/bookstore-lab-5a.bundle` mới nhất |
| B1: không có NetworkPolicy nào sau `deploy.sh` | Sprint 6 viết file trong `k8s/app/network/` nhưng chưa đưa vào `k8s/app/kustomization.yaml` | Sửa repo ở tag `lab-5a`, commit, chạy lại B0 — không `kubectl apply` tay |
| B1: `probe.sh` in `LOIDNS` | Pod đo chưa Ready, hoặc tên Service của sprint 6 khác hợp đồng | `kubectl -n bookstore get svc`; đối chiếu [mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe) |
| B1: có cặp `CHAN` trên Flannel | Cluster đã có CNI thực thi policy — không phải `01-cluster-ready` | `kubectl get pods -A -o wide \| grep -iE 'calico\|cilium'`; restore cả ba VM về `01-cluster-ready` |
| B3: namespace `kube-flannel` kẹt `Terminating` | Pod Flannel chưa hết ân hạn | Chờ; đừng ép xóa finalizer khi chưa biết nguyên nhân |
| **B3: Pod `calico-node` `CrashLoopBackOff`** | B3 dọn sót: còn conflist cũ hoặc interface cũ trên một node | Chạy lại bước dọn node trên **cả ba node**, rồi `kubectl -n kube-system rollout restart ds/calico-node` |
| **B3: IPPool khác dải Pod** | Calico không dò được dải của kubeadm | `kubectl delete -f ~/lab-work/5b/calico.yaml`, bỏ comment `CALICO_IPV4POOL_CIDR`, đặt đúng giá trị ở `b2-pod-cidr.txt`, làm lại bước dọn node và cài. Không sửa IPPool khi Pod đã lấy IP |
| **B3: annotation `projectcalico.org/IPv4Address` khác InternalIP** | Calico dò nhầm interface | Gỡ Calico, khai phương thức dò theo interface giữ default route (`ip route \| awk '/default/ {print $5; exit}'`) trong manifest, cài lại |
| B3: CoreDNS `ContainerCreating` mãi | CNI chưa sẵn sàng trên node giữ Pod đó | Chờ `calico-node` của node đó `Running`, xóa lại Pod CoreDNS |
| B3: node `Ready` nhưng `net-w1` không tới `net-w2` | Lưu lượng đóng gói giữa node bị chặn, hoặc Calico chọn nhầm interface | Kiểm annotation IP của node trước; UFW phải `inactive` như A4.4 |
| Không sửa được B3 trong vài phút | — | Tắt và **restore cả ba VM về `01-cluster-ready`**, clone repo từ bundle, chạy bảy tầng A5.4, làm lại từ B0 |
| B4: `sha256sum -c` fail | Tải hỏng hoặc sai file | Xóa, tải lại; không cài binary chưa kiểm được nguồn |
| B4: `helm install` báo không tương thích `kubeVersion` | Chart và baseline lệch | `helm show chart traefik/traefik --version "$TRAEFIK_CHART"`; dừng và cập nhật [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00), không tự chọn chart khác |
| B4: Service `traefik` vẫn `LoadBalancer` | Khóa trong `values.yaml` không khớp chart đang dùng | Đối chiếu `helm show values` của đúng version: kiểu Service nằm ở `service.spec.type`. Sửa file values rồi `helm upgrade traefik traefik/traefik --version "$TRAEFIK_CHART" -n traefik -f platform/traefik/values.yaml`; không `kubectl patch` |
| B4: `curl` tới `30081` trả `404` | Thiếu header `Host: som.lab.local`, hoặc Ingress `som` chưa có class | Thêm `-H 'Host: ...'`; tạo lại Ingress như yêu cầu 6 |
| B5: `deploy.sh` treo ở rollout `api` hoặc báo `db-migrate that bai` | Thiếu đường DNS, hoặc thiếu rule ở một trong hai đầu `api`/`db-migrate` → `db` | `kubectl -n bookstore describe networkpolicy`; `kubectl -n bookstore logs` init container `wait-for-db` |
| B5: `probe.sh` vẫn còn `THONG` | Policy cũ của sprint 6 còn sót và mở rộng hơn (policy cộng gộp), hoặc Calico chưa chạy | `./scripts/teardown.sh && ./scripts/deploy.sh`; kiểm `calico-node` |
| B5: Pod lạ trong `traefik` vào được `web` | Rule Traefik → `web` viết thành hai phần tử (phép hợp) | Gộp `namespaceSelector` và `podSelector` vào **một** phần tử `from` |
| B5: `web` không tới `api:3000` theo tên | Thiếu egress ở `web`, hoặc DNS bị chặn | Mỗi luồng cần egress ở nguồn **và** ingress ở đích; kiểm policy DNS chọn mọi Pod |
| B5: `web` không đọc được `node-agent` | Thiếu egress `web` → agent 9100 hoặc ingress ở agent | Bổ sung rule ở cả hai đầu |
| B6: `deploy.sh` báo `thieu ingressclass/traefik` | Cluster không ở trạng thái của mốc `02-net-ready` | Làm B4; ở lab sau, kiểm đúng snapshot đầu vào |
| B6: `curl: (60) SSL certificate problem` khi dùng `--cacert` | Traefik phục vụ certificate mặc định: Secret sai kiểu, sai tên, `tls.hosts` không khớp `host`, hoặc Ingress không được Traefik nhận | `kubectl -n bookstore describe ingress bookstore`; kiểm `type` của Secret `bookstore-tls` |
| B6: HTTPS timeout hoặc `504` | Rule Traefik → `web` ghi cổng `80`, hoặc phép hợp sai namespace | Cổng trong policy là `8080` của Pod |
| B6: `apply` Service `web` báo lỗi `nodePort` | Đổi `type` trên object đang có | `./scripts/teardown.sh && ./scripts/deploy.sh` |
| B6: máy host phân giải sai `bookstore.lab.local` | File hosts chưa lưu bằng quyền quản trị, hoặc bộ đệm DNS cũ | Mở PowerShell "Run as administrator"; `ipconfig /flushdns` |
| B7: log kube-proxy không có dòng mode | Pod đã chạy lâu, `--tail=200` không còn dòng khởi động | Đọc thêm `mode:` trong ConfigMap `kube-proxy`; tăng `--tail` |
| B7: apiserver proxy tới `som` trả lỗi | Sai tên cổng trong URL | Service `som` có cổng tên `http`: `.../services/som:http/proxy/` |
| B8: mọi path trả cùng một backend | Ingress `duong-dan` chưa được Traefik nhận, hoặc thiếu header `Host` | `kubectl -n ing-thu describe ingress duong-dan`; giữ `-H "Host: ..."` |
| B9: namespace kẹt `Terminating` | Pod chưa hết ân hạn (`db-0` 60 giây) | Chờ; nếu state vẫn lệch thì restore cả ba VM về `01-cluster-ready` và làm lại — **chưa chụp** `02-net-ready` |
| B9: gate namespace thấy thừa namespace | Object thử nghiệm chưa dọn | Xóa namespace thừa; mốc mới chỉ chứa namespace hệ thống và `traefik` |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Network Policies](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes v1.35 — Ingress](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/ingress/)
- [Kubernetes v1.35 — Ingress Controllers](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Kubernetes v1.35 — Gateway API](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/gateway/)
- [Kubernetes v1.35 — IPv4/IPv6 dual-stack](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/dual-stack/)
- [Kubernetes v1.35 — Cluster Networking](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Kubernetes v1.35 — Network Plugins](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [Kubernetes v1.35 — Proxies in Kubernetes](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/proxies/)
- [Kubernetes v1.35 — Secrets, mục TLS Secrets](https://v1-35.docs.kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes v1.35 — Declare Network Policy](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)
- [Kubernetes v1.35 — Install a Network Policy Provider](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/)
- [Kubernetes v1.35 — Use Calico for NetworkPolicy](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/calico-network-policy/)
- [Kubernetes v1.35 — Managing Secrets using Kustomize](https://v1-35.docs.kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kustomize/)
- [Kustomize — secretGenerator](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/secretgenerator/)
- [Calico — System requirements](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements)
- [Calico — Install on on-premises deployments](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises)
- [Traefik Helm chart](https://github.com/traefik/traefik-helm-chart)
- [Traefik — Kubernetes Ingress provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/)
- [Helm — Installing Helm](https://helm.sh/docs/intro/install/)
