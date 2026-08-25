# Lab 5b — NetworkPolicy, Ingress và CNI

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem
> [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> **Điểm kết thúc:** **tạo snapshot mới `02-net-ready`**. Lab này thay CNI và cài ingress
> controller, tức là **đổi hạ tầng vĩnh viễn** trên chuỗi snapshot chính; mọi lab từ giai đoạn 6
> trở đi bắt đầu từ mốc mới này chứ không quay lại `01-cluster-ready`.
> **Lab trước:** [Lab 5a — Service, EndpointSlice và DNS](LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md)
> đã cleanup và trả cluster về `01-cluster-ready`. Nếu bạn tới thẳng từ
> [Lab 4b](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) thì cluster cũng vẫn đang ở đúng
> `01-cluster-ready` và lab này chạy được ngay.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng phần policy/ingress/CNI của mục
[Giai đoạn 5 — Mạng nền tảng](../00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng), gồm nhóm bài
đứng **sau** Lab 5a trong giai đoạn: [11](../11-ingress-vi.md), [12](../12-ingress-controllers-vi.md),
[13](../13-gateway-vi.md), [85](../85-dual-stack-vi.md) và nhánh
[Tầng hạ tầng mạng của cluster](../00-ALO-TRINH-ADMIN.md#tầng-hạ-tầng-mạng-của-cluster)
([157](../157-networking-vi.md), [183](../183-network-plugins-vi.md), [164](../164-proxies-vi.md)).

Đây cũng là lab **trả [nợ #4](README.md#5-sổ-nợ-lab) — NetworkPolicy được thực thi thật**, phát
sinh ở bài [84 — Chính sách mạng](../84-network-policies-vi.md). **Đọc lại bài 84 trước khi bắt
đầu**; toàn bộ B3 và B5 bám đúng những gì bài đó trình bày, và lab sẽ chứng minh bằng lưu lượng
thật cả hai nửa của nó: policy nằm im khi CNI không hiện thực nó, và policy chặn thật khi CNI có
hiện thực.

Ba con số phiên bản của cluster vẫn là baseline ở
[bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không chép
lại** con số nào của bảng đó. Phần lab **thêm vào** chuỗi snapshot nằm ở [mục 2.3](#23-phiên-bản-lab-5b-đưa-vào-chuỗi-snapshot).

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Chỉ ra trên cluster của mình **ba dải IP không chồng lấn** — Pod, Service, Node — và nói đúng
  thành phần nào cấp phát từng dải; đọc được chúng từ tiến trình đang chạy chứ không từ trí nhớ.
- Nói được vì sao `ip addr` trên node và `node.status.addresses` là hai thứ khác nhau, và
  Kubernetes chỉ nhìn cái nào.
- Chỉ ra **ai nạp CNI plugin** trên node, hai đường dẫn phải thuộc lòng khi gỡ lỗi, và đọc được
  file cấu hình CNI thật trước lẫn sau khi đổi plugin.
- Chứng minh bằng lưu lượng rằng một NetworkPolicy hợp lệ **không chặn được gì** khi CNI không
  hiện thực nó — và phân biệt được "object được API server chấp nhận" với "policy có hiệu lực".
- Thực hiện được quy trình **đổi CNI** trên cluster đang chạy theo đúng thứ tự an toàn: gỡ plugin
  cũ, dọn interface và file cấu hình còn sót, cài plugin mới, đưa mọi Pod và DNS trở lại.
- Chứng minh **cùng một NetworkPolicy** giờ chặn thật; mở đúng một cổng cho đúng một nhóm Pod, và
  giải thích vì sao số cổng trong policy là cổng của **Pod đích** chứ không phải cổng của Service.
- Phân biệt phép **giao** và phép **hợp** trong `from` bằng một phép đo, chứ không bằng cách nhớ
  quy tắc.
- Chứng minh rằng cô lập chiều đi chặn luôn **DNS**, và mở lại đúng đường tối thiểu để traffic
  chạy được.
- Đọc được mode kube-proxy đang chạy và tìm thấy ClusterIP của Service mình tạo trong rule mà nó
  sinh ra trên node; nói được vì sao kube-proxy không thay được Ingress.
- Chứng minh **Ingress không có controller thì vô nghĩa**, rồi cài một ingress controller và làm
  cho cùng tài nguyên đó phục vụ thật từ ngoài cluster.
- Viết Ingress có `host`, nhiều `path` và `pathType`; đo được quy tắc "khớp theo từng phần tử
  path" và "path khớp dài nhất thắng"; giải thích `ingressClassName` và IngressClass mặc định.
- Kết thúc TLS tại điểm ingress bằng một Secret `kubernetes.io/tls` và nói được đoạn nào sau đó là
  plaintext.
- Chứng minh cluster này là **single-stack IPv4** bằng chính phản ứng của API server với
  `RequireDualStack` và `PreferDualStack`.
- Đưa cluster qua **trọn bảy tầng gate A5.4** với CNI mới rồi chụp snapshot `02-net-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

Thứ tự bảng dưới đây chính là thứ tự các mục `B`.

| Bài trong nhóm 5b | Phần lab kiểm chứng |
| --- | --- |
| [157 — Mạng trong cluster](../157-networking-vi.md) | B1 — đọc Pod CIDR từ kube-controller-manager, Service CIDR từ kube-apiserver, IP node từ `node.status.addresses`; kiểm ba dải không dùng chung tiền tố; đối chiếu số địa chỉ thật trên interface của `lab-k8s-worker2` với số địa chỉ Kubernetes ghi nhận |
| [183 — Network Plugin](../183-network-plugins-vi.md) | B2 — `/etc/cni/net.d` và `/opt/cni/bin` trên cả ba node, khai báo `portMappings`, plugin `loopback`, và bằng chứng kubelet **không** còn cờ quản lý CNI; B4 — thay chính plugin đó và đọc lại toàn bộ hai đường dẫn |
| [84 — Chính sách mạng](../84-network-policies-vi.md) | B3 — deny-all ingress trên CNI cũ mà lưu lượng vẫn qua (**bằng chứng của nợ #4**); B5 — cùng file policy đó chặn thật, mở một cổng, giao/hợp trong `from`, `endPort`, bẫy DNS của cô lập chiều đi (**trả nợ #4**) |
| [164 — Các loại proxy trong Kubernetes](../164-proxies-vi.md) | B6 — mode kube-proxy, ClusterIP trong bảng rule của node, `kubectl proxy` và apiserver proxy tới Service; loại 4 và loại 5 chỉ đọc vì cluster một control plane và không có cloud provider |
| [12 — Ingress Controllers](../12-ingress-controllers-vi.md) | B7 — cluster chưa có IngressClass nào và Ingress nằm im; B8 — cài một controller, đọc `.metadata.name` của IngressClass nó tạo và annotation class mặc định |
| [11 — Ingress](../11-ingress-vi.md) | B7 — `.status` rỗng khi chưa có controller; B9 — `host`, `pathType: Prefix` khớp theo phần tử path, path khớp dài nhất thắng, `pathType: Exact` không khớp dấu gạch chéo cuối, `ingressClassName` và class mặc định, TLS kết thúc tại ingress |
| [85 — Dual-stack IPv4/IPv6](../85-dual-stack-vi.md) | B10 — `.spec.ipFamilyPolicy`, `.spec.ipFamilies`, `.spec.clusterIPs` của Service thật; `RequireDualStack` bị API server từ chối và `PreferDualStack` tự quay về single-stack |

Ba trang lab **mượn quy trình cài đặt**, không thuộc nhóm bài giai đoạn 5 — lộ trình xếp chúng ở
[giai đoạn 21](../00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy):

| Trang | Lab dùng phần nào |
| --- | --- |
| [243 — Cài đặt một Network Policy Provider](../243-network-policy-provider-vi.md) | Trang mục; B4 dùng nó để chọn provider trong danh sách chính thức thay vì chọn theo cảm tính |
| [245 — Sử dụng Calico cho NetworkPolicy](../245-calico-network-policy-vi.md) | B4 — nhánh *cluster cục bộ với kubeadm*; phần GKE không áp dụng |
| [201 — Khai báo Network Policy](../201-declare-network-policy-vi.md) | B3 và B5 — trình tự kiểm chứng: dựng đích → đo đường cơ sở → áp policy → thử cả Pod có label lẫn Pod không có label |

Một bài của nhóm **không kiểm chứng được trên cluster lab**, đọc để biết:

| Bài | Vì sao không có bước thực hành |
| --- | --- |
| [13 — Gateway API](../13-gateway-vi.md) | Gateway API **không có sẵn** trong Kubernetes: nó là add-on gồm một bộ CustomResourceDefinition cộng với một bản hiện thực riêng. Cài nó là thêm một hạ tầng thứ hai vào cùng lab đang đổi CNI, và kiến thức CRD thuộc [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) với phần thực hành ở [giai đoạn 28](../00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes). B7 chỉ chạy **một lệnh đối chiếu** để thấy API group `gateway.networking.k8s.io` chưa tồn tại — đó là kiểm chứng cho *tính chất add-on* của Gateway API, không phải cho nội dung định tuyến của bài |

Ba điểm của nhóm bài **cố ý dừng ở phần đo được**, không phát sinh nợ mới:

- **Thứ tự ưu tiên khi hai path khớp bằng nhau** (`Exact` thắng `Prefix`) không được đưa vào gate.
  Cả bài [11](../11-ingress-vi.md) lẫn bài [12](../12-ingress-controllers-vi.md) đều cảnh báo các
  controller hoạt động hơi khác nhau, và cách xếp ưu tiên khi hòa là phần controller tự quyết. B9
  chỉ đo hai quy tắc không phụ thuộc controller: khớp **theo từng phần tử path**, và **path khớp
  dài nhất thắng**.
- **Wildcard hostname** (`*.foo.com`) cần một tên miền thật trỏ vào cluster. B9 dùng header `Host`
  nên chỉ đo được so khớp chính xác.
- **`hostPort` và điều tiết lưu lượng** của bài [183](../183-network-plugins-vi.md): B2 và B4 chỉ
  đọc khai báo `portMappings` trong file conflist chứ không dựng Pod `hostPort`; plugin
  `bandwidth` vẫn là tính năng thử nghiệm nên bài xếp là không cần.

### 1.2. Thời lượng

3–4 giờ, chia không đều:

- B0–B3 khoảng 45 phút: đọc cấu hình và dựng bằng chứng cho nợ #4.
- **B4 khoảng 60–90 phút** và là phần rủi ro nhất; đừng bắt đầu B4 khi không còn đủ thời gian chạy
  hết B4.
- B5–B6 khoảng 45 phút.
- B7–B9 khoảng 60 phút, phụ thuộc tốc độ tải chart và image.
- B10–B11 khoảng 30 phút, chưa kể thời gian tắt máy và chụp snapshot ba VM.

---

## 2. Quy ước và an toàn

- Mọi lệnh chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi mục ghi rõ node
  khác**. B2, B4.3 và một phần B6 chạy trên từng node.
- Object của lab nằm trong hai namespace `lab-5b` và `lab-5b-ext`, luôn gọi kèm `-n`. Ingress
  controller nằm ở namespace riêng của nó và **được giữ lại** sau lab.
- Manifest tạm ghi vào `~/lab-work/5b/`, bằng chứng ghi vào `~/lab-evidence/5b/`. Không lưu token,
  key hay certificate của cluster; certificate tự ký sinh ra ở B9 là của lab, không phải của
  control plane.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.
- Phần B dùng biến shell nối tiếp giữa các bước, nên chạy **toàn bộ phần B trong cùng một phiên
  shell** trên `lab-k8s-master`. Các giá trị quan trọng đều được ghi ra file trong
  `~/lab-evidence/5b/` để đọc lại được khi phải mở phiên mới.
- Lab **không** dùng PersistentVolumeClaim, StorageClass hay HorizontalPodAutoscaler — lưu trữ là
  [giai đoạn 6](../00-ALO-TRINH-ADMIN.md#giai-đoạn-6--lưu-trữ), autoscaling cần metrics-server của
  [giai đoạn 11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability). Cũng **không** đụng tới
  RBAC ([giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy)): mọi
  quyền của chart và của CNI do chính manifest của chúng khai báo, lab không tự viết Role.
- Lab này **không có bước fault injection**. Nếu bạn muốn tự thử phá thêm, chỉ được phá
  `lab-k8s-worker2`.
- **Ngoại lệ về phạm vi tác động:** B4 buộc phải chạm **cả ba node**, vì đổi CNI là thao tác trên
  từng node chứ không phải trên API. Đây là lý do lab này tạo snapshot mới thay vì trả cluster về
  mốc cũ.

### 2.1. Cảnh báo: đổi CNI là thao tác phá hoại

Giữa B4.2 và B4.4, cluster **mất tầng mạng Pod**: node chuyển `NotReady`, CoreDNS không chạy được,
Pod mới không có IP. Đó là trạng thái **đúng** của giai đoạn giữa, không phải sự cố. Nhưng nếu
dừng lại ở đó, hoặc dọn sót interface và file cấu hình cũ, cluster có thể không hồi phục bằng cách
sửa vặt.

Bắt buộc trước khi vào B4:

1. **Điểm khôi phục `01-cluster-ready` phải còn nguyên trên cả ba VM.** B0 có gate kiểm việc này
   từ máy host. Thiếu dù chỉ một VM thì **dừng**: tắt ba VM và chụp lại mốc theo đúng quy trình ở
   [A5.4.8 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot)
   trước khi tiếp tục.
2. Chạy trọn B4 trong một lần. Không tắt máy giữa chừng.

Khi B4 hỏng và không sửa được trong vài phút bằng [mục 4](#4-troubleshooting-của-lab-này): tắt và
**restore cả ba VM về `01-cluster-ready`**, chạy lại
[bảy tầng gate A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready), rồi làm lại lab từ B0.
Không restore riêng một VM; với cluster một control plane, restore là thao tác trên cả ba VM cùng
một mốc.

### 2.2. Vì sao phải đổi CNI

Bảng A1.3 của Lab 00 chọn Flannel có chủ đích và ghi rõ Flannel **không thực thi NetworkPolicy**.
Bài [183](../183-network-plugins-vi.md) giải thích vì sao điều đó vẫn hợp lệ: hợp đồng duy nhất mà
Kubernetes đặt ra cho một CNI plugin là **hiện thực mô hình mạng Kubernetes**, cộng hai yêu cầu bổ
sung là interface `lo` và hỗ trợ `hostPort`. NetworkPolicy **không** nằm trong hợp đồng đó.

Vì vậy chuỗi lab không "sửa lỗi" Flannel — nó dùng Flannel để dạy đúng một điều mà không cách nào
dạy bằng lý thuyết: một object hợp lệ, được API server nhận và lưu, vẫn có thể **không có hiệu lực
nào**. B3 lấy bằng chứng đó. B4 đổi plugin. B5 lấy lại bằng chứng ngược lại trên **cùng một file
policy**.

### 2.3. Phiên bản lab 5b đưa vào chuỗi snapshot

Snapshot `02-net-ready` chứa hai thành phần mà `01-cluster-ready` không có. Bảng dưới **không ghi con số nào** — cả ba đều đã được chốt ở
[bảng A1.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Bảng này chỉ nói lab dùng gì và vì sao, bổ sung cho
[bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) chứ không thay thế, và
mọi lab sau cần nói tới hai thành phần này thì link về đây.

| Thành phần | Phiên bản | Nguồn của con số |
| --- | --- | --- |
| CNI thay Flannel — Calico, cài bằng manifest | xem dòng *Calico (CNI thay Flannel)* trong [bảng A1.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) | Calico 3.32 ghi nhận dải Kubernetes đã kiểm thử bao gồm minor của baseline |
| Helm | xem dòng *Helm* trong [bảng A1.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) | Đã chốt sẵn ở A1.4 với ghi chú "lab nào cần chart đầu tiên" — chính là lab này |
| Ingress controller — Traefik, cài bằng chart | xem dòng *Traefik (Ingress)* trong [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) (chart **và** `appVersion`) | Đã chốt sẵn ở A1.4 với ghi chú "Lab 5b (ingress controller)" |

**Vì sao Calico chứ không phải plugin khác:** trang [243](../243-network-policy-provider-vi.md) liệt
kê sáu provider. Trong đó [Romana](../248-romana-network-policy-vi.md) đã ngừng phát triển;
[Cilium](../246-cilium-network-policy-vi.md) được trang chính thức hướng dẫn qua minikube và một
CLI riêng phải tải thêm; còn [Calico](../245-calico-network-policy-vi.md) có sẵn nhánh *cluster cục
bộ với kubeadm* và cài được bằng **một manifest duy nhất**, đúng hình dạng thao tác mà bạn đã làm
với Flannel ở [A5.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a52-cài-flannel-v0289). Ít bước lạ hơn
thì phần học nằm ở NetworkPolicy chứ không nằm ở việc vật lộn với trình cài đặt.

Chạy **một lần** ở đầu phiên shell phần B, trên `lab-k8s-master`:

```bash
export CALICO_VER='<Calico trong bảng A1.4 của Lab 00, dạng vX.Y.Z>'
export HELM_VER='<Helm trong bảng A1.4 của Lab 00, dạng vX.Y.Z>'
export TRAEFIK_CHART='<chart Traefik trong bảng A1.4 của Lab 00, dạng X.Y.Z>'

test -n "$CALICO_VER" && test -n "$HELM_VER" && test -n "$TRAEFIK_CHART" \
  && echo 'PASS: ba bien phien ban da duoc dat'
case "$CALICO_VER$HELM_VER$TRAEFIK_CHART" in
  *'<'*) echo 'FAIL: con placeholder chua thay bang so that trong bang A1.4';;
  *) echo 'PASS: khong con placeholder';;
esac
```

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`. Hai biến `HELM_VER` và
`TRAEFIK_CHART` phải được điền bằng đúng con số đọc từ bảng A1.4 — lab cố ý **không** chép chúng
vào đây, để bảng A1.4 vẫn là nơi duy nhất giữ số của hai thành phần đó.

---

# Phần B — Thực hành kiến thức 5b

## B0. Chuẩn bị workspace, điểm khôi phục và namespace

**Mục đích:** dựng chỗ làm việc, và chứng minh **trước khi đụng vào mạng** rằng cluster có đường
lùi.

Trên `lab-k8s-master`:

```bash
mkdir -p ~/lab-work/5b ~/lab-evidence/5b
kubectl config current-context
kubectl get nodes -o wide | tee ~/lab-evidence/5b/b0-nodes-truoc.txt

kubectl create namespace lab-5b
kubectl create namespace lab-5b-ext

test "$(kubectl get ns lab-5b -o jsonpath='{.status.phase}')" = 'Active' \
  && echo 'PASS: namespace lab-5b Active'
test "$(kubectl get ns lab-5b-ext -o jsonpath='{.status.phase}')" = 'Active' \
  && echo 'PASS: namespace lab-5b-ext Active'
```

Namespace mang sẵn label tự động `kubernetes.io/metadata.name`; B5 sẽ dùng chính label đó làm
`namespaceSelector`, nên xác nhận nó có thật:

```bash
kubectl get ns lab-5b-ext -o jsonpath='{.metadata.labels}'; echo
kubectl get ns lab-5b-ext \
  -o jsonpath='{.metadata.labels.kubernetes\.io/metadata\.name}' | grep -qx 'lab-5b-ext' \
  && echo 'PASS: namespace mang label kubernetes.io/metadata.name dung ten cua no'
```

**PASS:** ba dòng `PASS:` xuất hiện.

Gate điểm khôi phục — chạy trên **PowerShell của máy host Windows**, không phải trên VM. Đường dẫn
`.vmx` theo máy host đang dùng, sửa lại nếu VM nằm chỗ khác:

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

**PASS:** đúng ba dòng `PASS:`, không có dòng `FAIL:`. Thiếu bất kỳ VM nào thì **dừng lab tại
đây**: tắt ba VM và chụp mốc theo
[A5.4.8 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot),
rồi mới quay lại. Đây là điều kiện của [mục 2.1](#21-cảnh-báo-đổi-cni-là-thao-tác-phá-hoại).

---

## B1. Ba dải IP và mô hình mạng thật của cluster

**Mục đích:** bài [157](../157-networking-vi.md) nói cluster phải cấp ba dải IP không chồng lấn và
mỗi dải do một thành phần khác nhau cấp. Bước này đọc cả ba từ **tiến trình đang chạy**, không từ
tài liệu.

### B1.1. Đọc ba dải từ đúng nơi cấp phát chúng

```bash
MASTER=lab-k8s-master

SVC_CIDR="$(kubectl -n kube-system get pod "kube-apiserver-$MASTER" \
  -o jsonpath='{range .spec.containers[0].command[*]}{@}{"\n"}{end}' \
  | sed -n 's/^--service-cluster-ip-range=//p')"

POD_CIDR="$(kubectl -n kube-system get pod "kube-controller-manager-$MASTER" \
  -o jsonpath='{range .spec.containers[0].command[*]}{@}{"\n"}{end}' \
  | sed -n 's/^--cluster-cidr=//p')"

MASTER_IP="$(kubectl get node "$MASTER" \
  -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"

printf '%s\n' "$POD_CIDR" | tee ~/lab-evidence/5b/b1-pod-cidr.txt
printf '%s\n' "$SVC_CIDR" | tee ~/lab-evidence/5b/b1-svc-cidr.txt
printf '%s\n' "$MASTER_IP" | tee ~/lab-evidence/5b/b1-master-ip.txt
```

Hai file `b1-pod-cidr.txt` và `b1-svc-cidr.txt` được các mục sau đọc lại, nên đừng xóa chúng giữa
chừng.

Đối chiếu với cấu hình mà kubeadm đã ghi lại:

```bash
kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' \
  | grep -E 'podSubnet|serviceSubnet' | tee ~/lab-evidence/5b/b1-kubeadm-config.txt

grep -q "$POD_CIDR" ~/lab-evidence/5b/b1-kubeadm-config.txt \
  && echo 'PASS: podSubnet trong kubeadm-config trung --cluster-cidr dang chay'
grep -q "$SVC_CIDR" ~/lab-evidence/5b/b1-kubeadm-config.txt \
  && echo 'PASS: serviceSubnet trong kubeadm-config trung --service-cluster-ip-range dang chay'
```

**PASS:** hai dòng `PASS:` xuất hiện. Lệch nhau nghĩa là ai đó đã sửa manifest control plane sau
khi init — dừng lại và tìm nguyên nhân trước khi đổi CNI.

Kiểm nhanh ba dải không dùng chung tiền tố:

```bash
POD_PFX="$(echo "$POD_CIDR" | cut -d. -f1,2)"
SVC_PFX="$(echo "$SVC_CIDR" | cut -d. -f1,2)"
NODE_PFX="$(echo "$MASTER_IP" | cut -d. -f1,2)"
echo "pod=$POD_PFX svc=$SVC_PFX node=$NODE_PFX"

if [ "$POD_PFX" != "$SVC_PFX" ] && [ "$POD_PFX" != "$NODE_PFX" ] && [ "$SVC_PFX" != "$NODE_PFX" ]; then
  echo 'PASS: ba dai IP khong dung chung tien to hai octet dau'
else
  echo 'FAIL: hai trong ba dai dung chung tien to - kiem tra lai truoc khi doi CNI'
fi
```

**PASS:** dòng `PASS:` xuất hiện. Đây là **điều kiện cần chứ không phải điều kiện đủ** — nó bắt
được trường hợp chồng lấn thô, không chứng minh được hai dải rời nhau hoàn toàn.

**Ý nghĩa:** ba dải, ba nơi cấp. Network plugin cấp IP cho **Pod** — đó là lý do đổi plugin ở B4
làm Pod IP đổi. kube-apiserver cấp IP cho **Service** — đó là lý do ClusterIP **không** đổi khi bạn
thay CNI. kubelet cấp IP cho **Node** — dải LAN của ba VM.

### B1.2. `ip addr` không phải thứ Kubernetes nhìn

Chạy trên **`lab-k8s-worker2`**:

```bash
ip -br a | tee ~/lab-evidence/5b/b1-worker2-ip.txt
hostname -I
test "$(hostname -I | wc -w)" -gt 1 \
  && echo 'PASS: node co nhieu hon mot dia chi IPv4 tren cac interface'
```

Chạy trên `lab-k8s-master`:

```bash
kubectl get node lab-k8s-worker2 \
  -o jsonpath='{range .status.addresses[*]}{.type}{"\t"}{.address}{"\n"}{end}' \
  | tee ~/lab-evidence/5b/b1-worker2-addresses.txt

test "$(kubectl get node lab-k8s-worker2 \
  -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}' | wc -w)" = '1' \
  && echo 'PASS: Kubernetes chi ghi nhan dung mot InternalIP cho node do'
```

**PASS:** hai dòng `PASS:` xuất hiện.

**Ý nghĩa:** đúng khẳng định của bài 157 — một node có thể mang nhiều IP trên nhiều interface
(`lo`, `cni0`, `flannel.1`, card LAN), nhưng Kubernetes **chỉ xét** những địa chỉ nằm trong
`node.status.addresses` và `pod.status.ips` khi hiện thực hóa mô hình mạng. Nhìn `ip addr` để gỡ
lỗi thì được; suy ra "Kubernetes đang dùng địa chỉ này" thì sai.

### B1.3. PodCIDR của mỗi node

```bash
kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,INTERNAL-IP:.status.addresses[?(@.type=="InternalIP")].address,PODCIDR:.spec.podCIDR' \
  | tee ~/lab-evidence/5b/b1-podcidr.txt

test "$(awk 'NR>1 {print $3}' ~/lab-evidence/5b/b1-podcidr.txt | sort -u | wc -l)" = '3' \
  && echo 'PASS: ba node co ba PodCIDR khac nhau'
```

**PASS:** dòng `PASS:` xuất hiện.

**Ý nghĩa:** `--pod-network-cidr` lúc `kubeadm init` làm kube-controller-manager cắt dải lớn thành
ba khối `/24` và ghi vào `.spec.podCIDR` của từng Node. Flannel dùng đúng con số đó để dựng route.
Giữ file này lại: B4.5 sẽ cho thấy plugin mới **không** bắt buộc phải dùng nó, và đó là một khác
biệt có thật chứ không phải lỗi.

---

## B2. CNI đang chạy: đọc cấu hình thật trên node

**Mục đích:** bài [183](../183-network-plugins-vi.md) yêu cầu thuộc hai đường dẫn và biết **ai**
nạp plugin. Bước này đọc chúng trên node trước khi thay plugin, để B4.5 có cái đối chiếu.

Chạy trên **cả ba node**, bắt đầu từ `lab-k8s-worker1`:

```bash
ls -l /etc/cni/net.d/
ls /opt/cni/bin/ | tr '\n' ' '; echo

test -f /etc/cni/net.d/10-flannel.conflist \
  && echo 'PASS: co file cau hinh CNI cua Flannel trong /etc/cni/net.d'
test -x /opt/cni/bin/loopback \
  && echo 'PASS: co binary loopback trong /opt/cni/bin'
test -x /opt/cni/bin/portmap \
  && echo 'PASS: co binary portmap trong /opt/cni/bin'
```

**PASS:** ba dòng `PASS:` trên **từng node**.

Đọc chính file cấu hình:

```bash
sudo cat /etc/cni/net.d/10-flannel.conflist

sudo grep -q '"type": "flannel"' /etc/cni/net.d/10-flannel.conflist \
  && echo 'PASS: plugin chinh trong conflist la flannel'
sudo grep -q '"portMappings": true' /etc/cni/net.d/10-flannel.conflist \
  && echo 'PASS: conflist co khai bao portMappings capability'
sudo grep -q 'cniVersion' /etc/cni/net.d/10-flannel.conflist \
  && echo 'PASS: conflist co khai bao cniVersion'
```

**PASS:** ba dòng `PASS:`.

**Ý nghĩa:** đây đúng là hình dạng hai file JSON mà bài 183 lấy làm ví dụ, chỉ khác là ví dụ của
bài dùng Calico còn bản trên máy bạn đang dùng Flannel. Khối `portmap` với
`"capabilities": {"portMappings": true}` chính là thứ bài nói phải khai trong `cni-conf-dir` nếu
muốn `hostPort` hoạt động — nó có sẵn ở đây, không phải bạn thêm.

### B2.1. Ai nạp plugin: containerd, không phải kubelet

Vẫn trên node:

```bash
sudo containerd config dump | grep -A6 -iE '\.cni\]' | tee /tmp/b2-cni-conf.txt
grep -q '/opt/cni/bin' /tmp/b2-cni-conf.txt \
  && echo 'PASS: containerd tro toi /opt/cni/bin'
grep -q '/etc/cni/net.d' /tmp/b2-cni-conf.txt \
  && echo 'PASS: containerd tro toi /etc/cni/net.d'

if kubelet --help 2>&1 | grep -q -- '--cni-bin-dir'; then
  echo 'FAIL: kubelet van con co --cni-bin-dir'
else
  echo 'PASS: kubelet khong con co --cni-bin-dir, quan ly CNI khong thuoc kubelet'
fi
if kubelet --help 2>&1 | grep -q -- '--network-plugin'; then
  echo 'FAIL: kubelet van con co --network-plugin'
else
  echo 'PASS: kubelet khong con co --network-plugin'
fi
```

**PASS:** bốn dòng `PASS:`, không có dòng `FAIL:`.

**Ý nghĩa:** hai cờ `cni-bin-dir` và `network-plugin` đã bị gỡ khỏi kubelet, đúng như bài 183 nói:
việc quản lý CNI **không còn nằm trong phạm vi trách nhiệm của kubelet**. Thứ thật sự nạp plugin là
container runtime cung cấp dịch vụ CRI — ở đây là containerd — và hai đường dẫn nó trỏ tới chính là
hai đường dẫn bạn vừa `ls`. Ghi nhớ điều này: ở B4.3, dọn xong file cấu hình thì phải restart
**containerd** trước, rồi mới tới kubelet.

Lưu một bản conflist cũ làm bằng chứng, chạy trên `lab-k8s-master`:

```bash
{
  date -Is
  echo '--- /etc/cni/net.d cua lab-k8s-master'
  ls -l /etc/cni/net.d/
  echo '--- noi dung conflist'
  sudo cat /etc/cni/net.d/10-flannel.conflist
} | tee ~/lab-evidence/5b/b2-cni-truoc.txt

test -s ~/lab-evidence/5b/b2-cni-truoc.txt && echo 'PASS: da luu conflist cu lam bang chung'
```

**PASS:** dòng `PASS:` xuất hiện.

---

## B3. NetworkPolicy trên CNI hiện tại: hợp lệ nhưng không chặn

**Mục đích:** lấy **bằng chứng cốt lõi** của [nợ #4](README.md#5-sổ-nợ-lab). Đọc lại bài
[84](../84-network-policies-vi.md) trước khi làm mục này. Trình tự bốn bước dưới đây lấy từ bài
[201](../201-declare-network-policy-vi.md): dựng đích → đo đường cơ sở → áp policy → thử lại.

### B3.1. Dựng đích và client

Tạo `~/lab-work/5b/app.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-w1
  namespace: lab-5b
  labels:
    app: web
spec:
  nodeName: lab-k8s-worker1
  containers:
    - name: httpd
      image: busybox:1.37
      command:
        - sh
        - -c
        - 'mkdir -p /www && echo web-ok > /www/index.html && httpd -f -p 8080 -h /www'
      ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Pod
metadata:
  name: web-w2
  namespace: lab-5b
  labels:
    app: web
spec:
  nodeName: lab-k8s-worker2
  containers:
    - name: httpd
      image: busybox:1.37
      command:
        - sh
        - -c
        - 'mkdir -p /www && echo web-ok > /www/index.html && httpd -f -p 8080 -h /www'
      ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: lab-5b
spec:
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: 8080
---
apiVersion: v1
kind: Pod
metadata:
  name: client
  namespace: lab-5b
spec:
  nodeName: lab-k8s-worker1
  containers:
    - name: shell
      image: busybox:1.37
      command: ['sleep', '36000']
---
apiVersion: v1
kind: Pod
metadata:
  name: client-ok
  namespace: lab-5b
  labels:
    access: "true"
spec:
  nodeName: lab-k8s-worker1
  containers:
    - name: shell
      image: busybox:1.37
      command: ['sleep', '36000']
---
apiVersion: v1
kind: Pod
metadata:
  name: client-ext
  namespace: lab-5b-ext
spec:
  nodeName: lab-k8s-worker2
  containers:
    - name: shell
      image: busybox:1.37
      command: ['sleep', '36000']
```

Hai Pod `web-*` dùng `nodeName` để ghim mỗi bản lên một worker: để scheduler tự chọn thì cả hai có
thể rơi vào cùng một node và phép đo xuyên node mất ý nghĩa. `client-ext` nằm ở namespace khác và
trên worker khác — B5.5 cần đúng hai khác biệt đó.

```bash
kubectl apply -f ~/lab-work/5b/app.yaml
kubectl -n lab-5b wait --for=condition=Ready pod/web-w1 pod/web-w2 pod/client pod/client-ok --timeout=180s
kubectl -n lab-5b-ext wait --for=condition=Ready pod/client-ext --timeout=180s
kubectl -n lab-5b get pods -o wide
kubectl -n lab-5b get endpointslice -l kubernetes.io/service-name=web

test "$(kubectl -n lab-5b get pods -l app=web \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)" = '2' \
  && echo 'PASS: hai Pod web nam tren hai node khac nhau'
test "$(kubectl -n lab-5b get endpointslice -l kubernetes.io/service-name=web \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\n"}{end}' | wc -l)" = '2' \
  && echo 'PASS: EndpointSlice cua Service web co du hai dia chi'
```

**PASS:** năm Pod `Running`, hai dòng `PASS:` xuất hiện.

### B3.2. Đo đường cơ sở

```bash
kubectl -n lab-5b exec client -- wget -q -T 3 -O- http://web \
  | tee ~/lab-evidence/5b/b3-duong-co-so.txt

grep -qx 'web-ok' ~/lab-evidence/5b/b3-duong-co-so.txt \
  && echo 'PASS: duong co so thong - client goi duoc Service web'

kubectl -n lab-5b-ext exec client-ext -- wget -q -T 3 -O- http://web.lab-5b.svc.cluster.local \
  | grep -qx 'web-ok' && echo 'PASS: duong co so xuyen namespace cung thong'
```

**PASS:** hai dòng `PASS:`. Không có đường cơ sở thì mọi timeout ở bước sau đều mơ hồ: bạn không
phân biệt được "policy đang chặn" với "Service, DNS hoặc CNI vốn đã hỏng".

### B3.3. Áp một policy chặn sạch chiều đến

Tạo `~/lab-work/5b/np-01-deny-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: lab-5b
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

```bash
kubectl apply -f ~/lab-work/5b/np-01-deny-ingress.yaml
kubectl -n lab-5b get networkpolicy
kubectl -n lab-5b describe networkpolicy default-deny-ingress \
  | tee ~/lab-evidence/5b/b3-policy-da-luu.txt

test -n "$(kubectl -n lab-5b get networkpolicy default-deny-ingress -o jsonpath='{.metadata.uid}')" \
  && echo 'PASS: API server da nhan va luu object NetworkPolicy'
```

**PASS:** dòng `PASS:` xuất hiện. `podSelector: {}` chọn **tất cả** Pod trong namespace, và
`policyTypes: [Ingress]` không kèm danh sách `ingress` nào — theo bài 84, mọi Pod trong `lab-5b`
bây giờ **bị cô lập chiều đến** và không kết nối vào nào được phép.

### B3.4. Đo lại: không có gì bị chặn

```bash
if kubectl -n lab-5b exec client -- wget -q -T 3 -O- http://web >/dev/null 2>&1; then
  echo 'PASS: luu luong VAN QUA - CNI hien tai khong thuc thi NetworkPolicy'
else
  echo 'FAIL: luu luong bi chan - cluster nay khong con dung baseline cua Lab 00'
fi

kubectl -n lab-5b exec client -- wget -q -T 3 -O- http://web \
  | tee ~/lab-evidence/5b/b3-sau-policy.txt
diff -q ~/lab-evidence/5b/b3-duong-co-so.txt ~/lab-evidence/5b/b3-sau-policy.txt \
  && echo 'PASS: ket qua truoc va sau khi ap policy giong het nhau'
```

**PASS:** hai dòng `PASS:`, không có dòng `FAIL:`. Nếu ra `FAIL` thì cluster của bạn **đã** chạy một
CNI có thực thi policy — dừng lại và đối chiếu với
[bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), vì lab này giả định điểm bắt đầu
là `01-cluster-ready`.

Lưu bằng chứng của nợ #4:

```bash
{
  date -Is
  echo '--- CNI dang chay'
  kubectl get pods -A -o wide | grep -iE 'flannel|calico|cilium' || echo '(khong tim thay)'
  echo '--- NetworkPolicy dang ton tai'
  kubectl -n lab-5b get networkpolicy
  echo '--- Ket qua goi Service khi dang co deny-all ingress'
  cat ~/lab-evidence/5b/b3-sau-policy.txt
} | tee ~/lab-evidence/5b/b3-bang-chung-no-4.txt

test -s ~/lab-evidence/5b/b3-bang-chung-no-4.txt \
  && echo 'PASS: da luu bang chung cho no #4'
```

**PASS:** dòng `PASS:` xuất hiện.

Gỡ policy trước khi đổi CNI. File YAML vẫn giữ nguyên — B5 sẽ apply lại **đúng file này**:

```bash
kubectl -n lab-5b delete networkpolicy --all
test "$(kubectl -n lab-5b get networkpolicy --no-headers 2>/dev/null | wc -l)" = '0' \
  && echo 'PASS: da go het policy truoc khi doi CNI'
```

**PASS:** dòng `PASS:` xuất hiện.

**Ý nghĩa:** đây là nửa đầu của bài 84 được chứng minh bằng lưu lượng thật. `kubectl apply` thành
công chỉ nói API server chấp nhận object; nó **không** nói gì về việc có ai đó đang hiện thực object
đó. Bài 84 viết thẳng: *tạo một tài nguyên NetworkPolicy mà không có controller hiện thực nó sẽ
không có tác dụng gì*. Bạn vừa nhìn thấy điều đó xảy ra.

---

## B4. Đổi CNI sang plugin có thực thi NetworkPolicy

**Mục đích:** thay network plugin trên một cluster đang chạy, theo đúng thứ tự an toàn. Đây là mục
rủi ro nhất của lab. Đọc lại [mục 2.1](#21-cảnh-báo-đổi-cni-là-thao-tác-phá-hoại) trước khi gõ lệnh
đầu tiên, và chỉ bắt đầu khi còn đủ thời gian chạy hết B4.

Thứ tự bốn bước **không được đảo**:

1. Gỡ plugin cũ ở tầng API (B4.2) — nếu không, DaemonSet cũ sẽ ghi lại file cấu hình bạn vừa xóa.
2. Dọn file cấu hình và interface trên từng node (B4.3) — nếu không, hai plugin cùng để lại
   conflist và bạn không biết cái nào đang được nạp.
3. Cài plugin mới (B4.4).
4. Tái tạo Pod mang IP cũ (B4.6) — Pod đã có IP của plugin cũ **không** tự đổi sang dải mới.

### B4.1. Tải manifest và đọc trước khi apply

```bash
echo "$CALICO_VER"
curl -fsSL "https://raw.githubusercontent.com/projectcalico/calico/$CALICO_VER/manifests/calico.yaml" \
  -o ~/lab-work/5b/calico.yaml

test -s ~/lab-work/5b/calico.yaml && echo 'PASS: da tai duoc manifest CNI moi'

grep -c 'kind: CustomResourceDefinition' ~/lab-work/5b/calico.yaml
grep -nE '^\s+image: ' ~/lab-work/5b/calico.yaml | tee ~/lab-evidence/5b/b4-image-trong-manifest.txt
grep -n 'CALICO_IPV4POOL_CIDR' ~/lab-work/5b/calico.yaml
```

```bash
grep -q "image: .*calico/node:$CALICO_VER" ~/lab-work/5b/calico.yaml \
  && echo 'PASS: manifest mang dung tag da chot o muc 2.3'
if grep -E '^\s*#.*CALICO_IPV4POOL_CIDR' ~/lab-work/5b/calico.yaml >/dev/null; then
  echo 'PASS: CALICO_IPV4POOL_CIDR dang bi comment - plugin se tu do Pod CIDR tu cau hinh kubeadm'
else
  echo 'FAIL: CALICO_IPV4POOL_CIDR khong o dang comment - doc lai manifest truoc khi apply'
fi
```

**PASS:** ba dòng `PASS:`, không có dòng `FAIL:`.

**Ý nghĩa:** trang [245](../245-calico-network-policy-vi.md) dẫn sang nhánh *cluster cục bộ với
kubeadm*, và tài liệu chính thức của plugin nói: với kubeadm, khi Pod CIDR **khác** giá trị mặc
định của manifest thì không cần sửa gì — plugin tự dò từ cấu hình đang chạy. Lab không tin lời đó:
B4.5 có gate đối chiếu dải thật mà plugin cấp phát với `POD_CIDR` bạn đọc ở B1.

### B4.2. Gỡ plugin cũ ở tầng API

```bash
kubectl delete namespace kube-flannel --wait=true --timeout=300s
kubectl delete clusterrole flannel --ignore-not-found
kubectl delete clusterrolebinding flannel --ignore-not-found

if kubectl get ns kube-flannel >/dev/null 2>&1; then
  echo 'FAIL: namespace kube-flannel van con'
else
  echo 'PASS: namespace kube-flannel da bien mat'
fi
if kubectl get clusterrole flannel >/dev/null 2>&1; then
  echo 'FAIL: ClusterRole flannel van con'
else
  echo 'PASS: khong con ClusterRole flannel'
fi

kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,REASON:.status.conditions[?(@.type=="Ready")].reason'
```

**PASS:** hai dòng `PASS:`, không có dòng `FAIL:`.

Cột `READY` chuyển `False` với `REASON` nói về network plugin chưa sẵn sàng là **trạng thái đúng**
ở giữa bước đổi CNI. Node còn `Ready` ở thời điểm này cũng không sao: file cấu hình CNI cũ vẫn nằm
trên đĩa và sẽ bị dọn ngay ở B4.3. Thời điểm chuyển trạng thái phụ thuộc chu kỳ báo cáo của kubelet,
nên **đừng** dùng nó làm gate.

Lệnh xóa dùng tên object thay vì `kubectl delete -f <url>`: cách này không cần nhắc lại số phiên bản
của manifest cũ, mà số đó chỉ tồn tại ở
[bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa).

### B4.3. Dọn file cấu hình và interface trên từng node

Chạy trên **cả ba node**, thứ tự `lab-k8s-worker2` → `lab-k8s-worker1` → `lab-k8s-master`:

```bash
sudo rm -f /etc/cni/net.d/10-flannel.conflist
sudo rm -rf /var/lib/cni/flannel /var/lib/cni/networks /run/flannel

for i in flannel.1 cni0; do
  if ip link show "$i" >/dev/null 2>&1; then
    echo "xoa interface $i"
    sudo ip link delete "$i"
  fi
done

sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Gate trên **từng node**:

```bash
test ! -e /etc/cni/net.d/10-flannel.conflist \
  && echo 'PASS: da xoa file cau hinh CNI cu'
test "$(ls -A /etc/cni/net.d 2>/dev/null | wc -l)" = '0' \
  && echo 'PASS: /etc/cni/net.d rong - chua co plugin nao'

if ip link show flannel.1 >/dev/null 2>&1; then
  echo 'FAIL: interface flannel.1 van con'
else
  echo 'PASS: khong con interface flannel.1'
fi
if ip link show cni0 >/dev/null 2>&1; then
  echo 'FAIL: interface cni0 van con'
else
  echo 'PASS: khong con interface cni0'
fi

systemctl is-active containerd kubelet
```

**PASS:** bốn dòng `PASS:` trên **mỗi node**, không có dòng `FAIL:`; `containerd` và `kubelet` đều
`active`.

**Ý nghĩa:** hai thứ sót lại hay giết bước đổi CNI. Thứ nhất là **file conflist cũ**: containerd
chọn file cấu hình theo thứ tự tên, nên để lại hai file là để lại một cái bẫy im lặng. Thứ hai là
**interface cũ** — `cni0` giữ nguyên địa chỉ gateway của dải cũ, và plugin mới dựng đường đi riêng
trên interface riêng; hai bộ route chồng nhau thì Pod lên được nhưng gói tin đi lạc. Restart
containerd trước kubelet là vì containerd mới là thứ nạp plugin, đúng như B2.1 đã chứng minh.

### B4.4. Cài plugin mới

Trên `lab-k8s-master`:

```bash
kubectl apply -f ~/lab-work/5b/calico.yaml

kubectl -n kube-system rollout status daemonset/calico-node --timeout=600s
kubectl -n kube-system rollout status deployment/calico-kube-controllers --timeout=600s
kubectl -n kube-system get pods -l k8s-app=calico-node -o wide
```

Thời gian rollout phụ thuộc tốc độ kéo image và cấu hình máy; đừng coi con số `--timeout` ở trên là
cam kết, nó chỉ là trần chờ.

```bash
test "$(kubectl -n kube-system get pods -l k8s-app=calico-node \
  --field-selector=status.phase=Running --no-headers | wc -l)" = '3' \
  && echo 'PASS: dung ba Pod calico-node Running, moi node mot Pod'

kubectl -n kube-system get ds calico-node \
  -o jsonpath='{range .spec.template.spec.initContainers[*]}{.image}{"\n"}{end}{range .spec.template.spec.containers[*]}{.image}{"\n"}{end}' \
  | tee ~/lab-evidence/5b/b4-calico-images.txt
grep -q ":$CALICO_VER" ~/lab-evidence/5b/b4-calico-images.txt \
  && echo 'PASS: image dang chay dung tag da chot'

kubectl wait --for=condition=Ready node --all --timeout=300s \
  && echo 'PASS: ba node tro lai Ready sau khi cai plugin moi'
```

**PASS:** ba dòng `PASS:` xuất hiện.

### B4.5. Đọc lại CNI trên node và kiểm dải IP plugin cấp

Chạy trên **cả ba node**:

```bash
ls -l /etc/cni/net.d/
sudo cat /etc/cni/net.d/10-calico.conflist

test -f /etc/cni/net.d/10-calico.conflist \
  && echo 'PASS: co file cau hinh CNI cua plugin moi'
test ! -e /etc/cni/net.d/10-flannel.conflist \
  && echo 'PASS: khong con file cau hinh CNI cua plugin cu'
sudo grep -q '"type": "calico"' /etc/cni/net.d/10-calico.conflist \
  && echo 'PASS: plugin chinh trong conflist la calico'
sudo grep -q 'calico-ipam' /etc/cni/net.d/10-calico.conflist \
  && echo 'PASS: IPAM da chuyen sang calico-ipam'
sudo grep -q '"portMappings": true' /etc/cni/net.d/10-calico.conflist \
  && echo 'PASS: conflist moi van khai portMappings capability'
```

**PASS:** năm dòng `PASS:` trên **mỗi node**.

Trên `lab-k8s-master`, đối chiếu dải IP mà plugin mới tự cấu hình:

```bash
POD_CIDR="$(cat ~/lab-evidence/5b/b1-pod-cidr.txt)"

kubectl get ippools.crd.projectcalico.org -o wide
IPPOOL="$(kubectl get ippools.crd.projectcalico.org default-ipv4-ippool \
  -o jsonpath='{.spec.cidr}')"
echo "ippool=$IPPOOL pod_cidr=$POD_CIDR"

test "$IPPOOL" = "$POD_CIDR" \
  && echo 'PASS: dai IP plugin moi cap trung Pod CIDR cua cluster'
```

**PASS:** dòng `PASS:` xuất hiện. Nếu hai giá trị khác nhau, **dừng** và xử lý theo dòng tương ứng
ở [mục 4](#4-troubleshooting-của-lab-này) — chạy tiếp với dải sai sẽ làm mọi phép đo sau vô nghĩa.

Kiểm plugin mới bám đúng interface mang IP node:

```bash
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  A="$(kubectl get node "$n" -o jsonpath='{.metadata.annotations.projectcalico\.org/IPv4Address}')"
  I="$(kubectl get node "$n" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
  case "$A" in
    "$I"/*) echo "PASS: $n - plugin chon dung InternalIP ($A)";;
    *)      echo "FAIL: $n - plugin chon $A trong khi InternalIP la $I";;
  esac
done
```

**PASS:** ba dòng `PASS:`, không có dòng `FAIL:`. Đây chính là lỗi kinh điển khi VM có nhiều card
mạng: plugin tự dò và bắt nhầm interface, cluster vẫn `Ready` nhưng Pod hai node không thấy nhau.

IPAM đổi chủ — đối chiếu với `.spec.podCIDR` bạn ghi ở B1.3:

```bash
kubectl get ipamblocks.crd.projectcalico.org \
  -o custom-columns='BLOCK:.spec.cidr,AFFINITY:.spec.affinity' \
  | tee ~/lab-evidence/5b/b4-ipamblock.txt

test "$(awk 'NR>1' ~/lab-evidence/5b/b4-ipamblock.txt | wc -l)" -ge 2 \
  && echo 'PASS: plugin moi tu chia khoi dia chi va gan cho tung node'

cat ~/lab-evidence/5b/b1-podcidr.txt
```

**PASS:** dòng `PASS:` xuất hiện.

**Ý nghĩa — đọc kỹ chỗ này:** `.spec.podCIDR` của Node **vẫn nguyên** vì nó do kube-controller-manager
cấp, không phải do plugin. Nhưng IPAM giờ là của plugin mới: nó tự cắt dải lớn thành các khối riêng
và gán cho từng node, nên **Pod IP không nhất thiết còn nằm trong `/24` mà kubeadm gán cho node
đó**. Cả hai đều đúng và không mâu thuẫn — bài [157](../157-networking-vi.md) chỉ ràng buộc rằng
network plugin là thành phần cấp IP cho Pod, chứ không ràng buộc plugin phải dùng `.spec.podCIDR`.
Đây là khác biệt thật giữa hai plugin, không phải lỗi cấu hình.

### B4.6. Tái tạo Pod mang IP cũ và kiểm mạng đã liền

Pod tạo trước khi đổi plugin vẫn giữ IP của dải cũ cho tới khi bị tạo lại. CoreDNS là quan trọng
nhất trong số đó:

```bash
kubectl -n kube-system delete pod -l k8s-app=kube-dns
kubectl -n kube-system rollout status deployment/coredns --timeout=300s

kubectl delete -f ~/lab-work/5b/app.yaml --ignore-not-found
kubectl apply -f ~/lab-work/5b/app.yaml
kubectl -n lab-5b wait --for=condition=Ready pod/web-w1 pod/web-w2 pod/client pod/client-ok --timeout=300s
kubectl -n lab-5b-ext wait --for=condition=Ready pod/client-ext --timeout=300s
```

```bash
POD_PFX="$(cut -d. -f1,2 ~/lab-evidence/5b/b1-pod-cidr.txt)"
kubectl -n lab-5b get pods -o wide | tee ~/lab-evidence/5b/b4-pod-ip-moi.txt

BAD="$(kubectl -n lab-5b get pods -o jsonpath='{range .items[*]}{.status.podIP}{"\n"}{end}' \
  | grep -vc "^$POD_PFX\." || true)"
test "$BAD" = '0' \
  && echo 'PASS: moi Pod cua lab deu lay IP trong dai Pod CIDR'

kubectl -n lab-5b exec client -- wget -q -T 3 -O- http://web \
  | grep -qx 'web-ok' && echo 'PASS: Service va DNS trong cluster hoat dong lai'

BIP="$(kubectl -n lab-5b get pod web-w2 -o jsonpath='{.status.podIP}')"
kubectl -n lab-5b exec client -- wget -q -T 3 -O- "http://$BIP:8080" \
  | grep -qx 'web-ok' && echo 'PASS: TCP xuyen node bang Pod IP hoat dong'

kubectl -n lab-5b exec client -- nslookup kubernetes.default.svc.cluster.local >/dev/null \
  && echo 'PASS: DNS trong cluster tra loi'
kubectl -n lab-5b exec client -- nslookup registry.k8s.io >/dev/null \
  && echo 'PASS: DNS ra Internet tra loi'

kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
```

**PASS:** năm dòng `PASS:` xuất hiện và lệnh cuối trả `No resources found`. Đây là gate quyết định
của B4: qua được nghĩa là cluster đã sống lại đầy đủ trên plugin mới.

Ghi bằng chứng bước đổi CNI:

```bash
{
  date -Is
  echo '--- Pod cua plugin mang'
  kubectl -n kube-system get pods -l k8s-app=calico-node -o wide
  echo '--- IPPool'
  kubectl get ippools.crd.projectcalico.org -o wide
  echo '--- Pod IP sau khi doi'
  cat ~/lab-evidence/5b/b4-pod-ip-moi.txt
} | tee ~/lab-evidence/5b/b4-sau-doi-cni.txt
test -s ~/lab-evidence/5b/b4-sau-doi-cni.txt && echo 'PASS: da luu bang chung buoc doi CNI'
```

**PASS:** dòng `PASS:` xuất hiện.

---

## B5. Cùng một NetworkPolicy, lần này chặn thật

**Mục đích:** trả [nợ #4](README.md#5-sổ-nợ-lab). Mục này apply lại **đúng những file YAML** đã dùng
ở B3, không sửa một ký tự nào — thứ duy nhất đổi giữa hai lần đo là network plugin.

### B5.1. Cùng file `np-01`, lần này lưu lượng dừng

```bash
kubectl apply -f ~/lab-work/5b/np-01-deny-ingress.yaml
kubectl -n lab-5b get networkpolicy

if kubectl -n lab-5b exec client -- wget -q -T 3 -O- http://web >/dev/null 2>&1; then
  echo 'FAIL: policy van khong chan - plugin moi chua thuc thi NetworkPolicy'
else
  echo 'PASS: deny-all ingress da chan that'
fi

if kubectl -n lab-5b-ext exec client-ext -- \
     wget -q -T 3 -O- http://web.lab-5b.svc.cluster.local >/dev/null 2>&1; then
  echo 'FAIL: Pod o namespace khac van vao duoc'
else
  echo 'PASS: Pod o namespace khac cung bi chan'
fi
```

**PASS:** hai dòng `PASS:`, không có dòng `FAIL:`.

Nhìn xem policy được hiện thực ở đâu — chạy trên **`lab-k8s-worker1`**:

```bash
if sudo iptables-save 2>/dev/null | grep -q 'cali-'; then
  echo 'PASS: tim thay chain cali- o dataplane iptables cua node'
elif sudo nft list ruleset 2>/dev/null | grep -q 'cali'; then
  echo 'PASS: tim thay rule cali o dataplane nftables cua node'
else
  echo 'FAIL: khong tim thay rule nao cua plugin tren node nay'
fi
```

**PASS:** một dòng `PASS:`, không có dòng `FAIL:`.

**Ý nghĩa:** cùng một object, cùng một API, hai kết quả trái ngược. Sự khác nhau nằm ở chỗ **có
thành phần trên node dịch object đó thành rule hay không**. Đây là toàn bộ nội dung của nợ #4, và
tới đây nó đã được trả.

### B5.2. Mở đúng một cổng — và cái bẫy số cổng

Tạo `~/lab-work/5b/np-02-access-web-sai-port.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-web
  namespace: lab-5b
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              access: "true"
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl apply -f ~/lab-work/5b/np-02-access-web-sai-port.yaml

if kubectl -n lab-5b exec client-ok -- wget -q -T 3 -O- http://web >/dev/null 2>&1; then
  echo 'FAIL: khong chan - doc lai truong port cua policy'
else
  echo 'PASS: van bi chan du Pod da mang dung label access=true'
fi
```

**PASS:** dòng `PASS:` xuất hiện.

Sửa đúng: cùng file, đổi `port: 80` thành `port: 8080` và lưu thành
`~/lab-work/5b/np-03-access-web.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-web
  namespace: lab-5b
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              access: "true"
      ports:
        - protocol: TCP
          port: 8080
```

```bash
kubectl apply -f ~/lab-work/5b/np-03-access-web.yaml

kubectl -n lab-5b exec client-ok -- wget -q -T 3 -O- http://web \
  | grep -qx 'web-ok' && echo 'PASS: Pod mang label access=true di qua duoc'

if kubectl -n lab-5b exec client -- wget -q -T 3 -O- http://web >/dev/null 2>&1; then
  echo 'FAIL: Pod khong co label van vao duoc'
else
  echo 'PASS: Pod khong co label van bi chan'
fi

kubectl -n lab-5b get svc web -o jsonpath='{.spec.ports[0].port}{"\t"}{.spec.ports[0].targetPort}{"\n"}'
```

**PASS:** hai dòng `PASS:`, không có dòng `FAIL:`.

**Ý nghĩa:** Service nghe cổng `80` và chuyển tiếp tới cổng `8080` của Pod. NetworkPolicy làm việc ở
tầng Pod, nên `ports` trong policy là **cổng của Pod đích** — kết nối tới ClusterIP đã được
kube-proxy dịch địa chỉ đích trước khi tới Pod. Điền số cổng của Service vào policy là lỗi im lặng
kinh điển: object hợp lệ, gate không báo gì, traffic vẫn chết.

### B5.3. `endPort`: dải cổng có được thực thi không

Tạo `~/lab-work/5b/np-04-endport-ngoai-dai.yaml` — vẫn tên `access-web` nên nó thay thế policy
trước:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-web
  namespace: lab-5b
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              access: "true"
      ports:
        - protocol: TCP
          port: 8081
          endPort: 8090
```

```bash
kubectl apply -f ~/lab-work/5b/np-04-endport-ngoai-dai.yaml

if kubectl -n lab-5b exec client-ok -- wget -q -T 3 -O- http://web >/dev/null 2>&1; then
  echo 'FAIL: cong 8080 nam ngoai dai 8081-8090 ma van qua duoc'
else
  echo 'PASS: cong ngoai dai bi chan - endPort duoc thuc thi'
fi
```

**PASS:** dòng `PASS:` xuất hiện.

Đưa cổng đích vào trong dải — `~/lab-work/5b/np-05-endport-trong-dai.yaml` giống hệt file trên, chỉ
đổi `port: 8081` thành `port: 8080`:

```bash
sed 's/port: 8081/port: 8080/' ~/lab-work/5b/np-04-endport-ngoai-dai.yaml \
  > ~/lab-work/5b/np-05-endport-trong-dai.yaml
grep -q 'port: 8080' ~/lab-work/5b/np-05-endport-trong-dai.yaml \
  && grep -q 'endPort: 8090' ~/lab-work/5b/np-05-endport-trong-dai.yaml \
  && echo 'PASS: file moi co dai 8080-8090'

kubectl apply -f ~/lab-work/5b/np-05-endport-trong-dai.yaml
kubectl -n lab-5b exec client-ok -- wget -q -T 3 -O- http://web \
  | grep -qx 'web-ok' && echo 'PASS: cong trong dai di qua duoc'
```

**PASS:** hai dòng `PASS:` xuất hiện.

**Ý nghĩa:** bài 84 xếp `endPort` vào phần đọc lướt với lý do "cần CNI hỗ trợ đúng trường này mới có
tác dụng". Hai phép đo trên là cách duy nhất trả lời câu hỏi đó cho một cluster cụ thể: không đọc
tài liệu của plugin, mà đo trực tiếp hai cổng nằm trong và ngoài dải.

Đưa policy về bản đơn giản trước khi sang B5.4:

```bash
kubectl apply -f ~/lab-work/5b/np-03-access-web.yaml
kubectl -n lab-5b exec client-ok -- wget -q -T 3 -O- http://web \
  | grep -qx 'web-ok' && echo 'PASS: quay lai policy mot cong 8080'
```

**PASS:** dòng `PASS:` xuất hiện.

### B5.4. Giao và hợp trong `from` — một dấu gạch đầu dòng đổi cả chính sách

`client-ext` nằm ở namespace `lab-5b-ext` và **không** mang label `access=true`. Tạo
`~/lab-work/5b/np-06-tu-ns-khac-giao.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-web-tu-ns-khac
  namespace: lab-5b
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: lab-5b-ext
          podSelector:
            matchLabels:
              access: "true"
      ports:
        - protocol: TCP
          port: 8080
```

```bash
kubectl apply -f ~/lab-work/5b/np-06-tu-ns-khac-giao.yaml

if kubectl -n lab-5b-ext exec client-ext -- \
     wget -q -T 3 -O- http://web.lab-5b.svc.cluster.local >/dev/null 2>&1; then
  echo 'FAIL: phan tu giao ma Pod thieu label van qua duoc'
else
  echo 'PASS: mot phan tu chua ca hai selector la phep GIAO - thieu label thi bi chan'
fi
```

**PASS:** dòng `PASS:` xuất hiện.

Bây giờ tách đúng **một dấu gạch đầu dòng** — `~/lab-work/5b/np-07-tu-ns-khac-hop.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-web-tu-ns-khac
  namespace: lab-5b
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: lab-5b-ext
        - podSelector:
            matchLabels:
              access: "true"
      ports:
        - protocol: TCP
          port: 8080
```

```bash
kubectl apply -f ~/lab-work/5b/np-07-tu-ns-khac-hop.yaml

kubectl -n lab-5b-ext exec client-ext -- wget -q -T 3 -O- http://web.lab-5b.svc.cluster.local \
  | grep -qx 'web-ok' \
  && echo 'PASS: hai phan tu tach roi la phep HOP - moi Pod trong namespace do di qua duoc'

diff ~/lab-work/5b/np-06-tu-ns-khac-giao.yaml ~/lab-work/5b/np-07-tu-ns-khac-hop.yaml \
  | tee ~/lab-evidence/5b/b5-giao-va-hop.diff
test "$(grep -c '^[<>]' ~/lab-evidence/5b/b5-giao-va-hop.diff)" -le 4 \
  && echo 'PASS: hai chinh sach khac nhau rat it dong nhung nguoc nghia'
```

**PASS:** hai dòng `PASS:` xuất hiện.

**Ý nghĩa:** bài 84 nói một phần tử `from` chứa **cả** `namespaceSelector` lẫn `podSelector` là phép
**giao**, còn hai phần tử tách rời bằng dấu `-` là phép **hợp**. Bạn vừa đo cả hai trên cùng một
cặp Pod. Trong review manifest thật, đây là chỗ phải soi kỹ nhất: nhìn thoáng qua hai file gần như
giống nhau.

### B5.5. Cô lập chiều đi chặn luôn DNS

Tạo `~/lab-work/5b/np-08-deny-egress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-client-ok
  namespace: lab-5b
spec:
  podSelector:
    matchLabels:
      access: "true"
  policyTypes:
    - Egress
```

```bash
kubectl apply -f ~/lab-work/5b/np-08-deny-egress.yaml

if kubectl -n lab-5b exec client-ok -- nslookup web >/dev/null 2>&1; then
  echo 'FAIL: DNS van hoat dong du da co lap chieu di'
else
  echo 'PASS: co lap chieu di chan luon DNS - nslookup that bai'
fi
```

**PASS:** dòng `PASS:` xuất hiện.

Mở **đúng** đường DNS — `~/lab-work/5b/np-09-egress-dns.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-client-ok
  namespace: lab-5b
spec:
  podSelector:
    matchLabels:
      access: "true"
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
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

```bash
kubectl apply -f ~/lab-work/5b/np-09-egress-dns.yaml

kubectl -n lab-5b exec client-ok -- nslookup web >/dev/null \
  && echo 'PASS: DNS hoat dong lai sau khi mo dung duong toi Service DNS'

if kubectl -n lab-5b exec client-ok -- wget -q -T 3 -O- http://web >/dev/null 2>&1; then
  echo 'FAIL: da goi duoc web trong khi chi moi mo DNS'
else
  echo 'PASS: phan giai duoc ten nhung ket noi toi web van bi chan'
fi
```

**PASS:** hai dòng `PASS:`, không có dòng `FAIL:`.

Mở nốt đường tới Pod đích — `~/lab-work/5b/np-10-egress-dns-va-web.yaml` là file trên cộng thêm một
phần tử `to`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-client-ok
  namespace: lab-5b
spec:
  podSelector:
    matchLabels:
      access: "true"
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
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    - to:
        - podSelector:
            matchLabels:
              app: web
      ports:
        - protocol: TCP
          port: 8080
```

```bash
kubectl apply -f ~/lab-work/5b/np-10-egress-dns-va-web.yaml

kubectl -n lab-5b exec client-ok -- wget -q -T 3 -O- http://web \
  | grep -qx 'web-ok' && echo 'PASS: mo du hai duong thi ket noi chay lai'
```

**PASS:** dòng `PASS:` xuất hiện.

**Ý nghĩa:** hai điều của bài 84 vừa được đo cùng lúc. Thứ nhất, **deny egress chặn luôn traffic
DNS** — bài cảnh báo riêng chỗ này vì triệu chứng hiện ra là "không phân giải được tên", rất dễ bị
quy oan cho CoreDNS. Thứ hai, một kết nối chỉ đi được khi **cả** chính sách egress ở Pod nguồn
**lẫn** chính sách ingress ở Pod đích cùng cho phép: `client-ok` phải được `np-10` cho ra, đồng thời
`web` phải được `np-03` cho vào.

### B5.6. Xóa policy thì hết cô lập

```bash
kubectl -n lab-5b get networkpolicy | tee ~/lab-evidence/5b/b5-policy-truoc-khi-xoa.txt
kubectl -n lab-5b delete networkpolicy --all

test "$(kubectl -n lab-5b get networkpolicy --no-headers 2>/dev/null | wc -l)" = '0' \
  && echo 'PASS: khong con NetworkPolicy nao trong lab-5b'

kubectl -n lab-5b exec client -- wget -q -T 3 -O- http://web \
  | grep -qx 'web-ok' && echo 'PASS: Pod khong co label da vao lai duoc'
kubectl -n lab-5b-ext exec client-ext -- wget -q -T 3 -O- http://web.lab-5b.svc.cluster.local \
  | grep -qx 'web-ok' && echo 'PASS: Pod o namespace khac cung vao lai duoc'
```

**PASS:** ba dòng `PASS:` xuất hiện.

**Ý nghĩa:** mặc định là **cho phép tất cả**. Một Pod chỉ bị cô lập theo một hướng khi **tồn tại**
một NetworkPolicy vừa chọn nó vừa có hướng đó trong `policyTypes`. Xóa policy đi là quay về mặc
định ngay, không cần khởi động lại gì.

Chốt bằng chứng trả nợ:

```bash
{
  date -Is
  echo '=== NO #4 DA TRA TAI LAB 5B ==='
  echo '--- Truoc khi doi CNI (B3): deny-all ingress khong chan duoc'
  cat ~/lab-evidence/5b/b3-bang-chung-no-4.txt
  echo '--- Sau khi doi CNI (B5): cung file policy do chan that'
  cat ~/lab-evidence/5b/b5-policy-truoc-khi-xoa.txt
} | tee ~/lab-evidence/5b/b5-tra-no-4.txt
test -s ~/lab-evidence/5b/b5-tra-no-4.txt && echo 'PASS: da luu ho so tra no #4'
```

**PASS:** dòng `PASS:` xuất hiện.

---

## B6. kube-proxy và bốn thứ khác cũng tên là "proxy"

**Mục đích:** bài [164](../164-proxies-vi.md) xếp lại năm thứ đều được gọi là proxy. Mục này chạm
tay vào ba loại đầu; loại 4 và loại 5 chỉ đọc, vì cluster lab có một control plane và không có cloud
provider.

### B6.1. kube-proxy vẫn nguyên sau khi đổi CNI

```bash
kubectl -n kube-system get daemonset kube-proxy
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide

test "$(kubectl -n kube-system get pods -l k8s-app=kube-proxy \
  --field-selector=status.phase=Running --no-headers | wc -l)" = '3' \
  && echo 'PASS: kube-proxy van du ba Pod sau khi doi CNI'
```

**PASS:** dòng `PASS:` xuất hiện.

**Ý nghĩa:** đổi network plugin **không** đụng tới kube-proxy. Hai thứ giải hai bài toán khác nhau:
plugin lo Pod-với-Pod, kube-proxy lo Pod-với-Service. Đây cũng là lý do ClusterIP của Service `web`
không đổi khi bạn thay CNI ở B4.

### B6.2. Mode đang chạy

```bash
kubectl -n kube-system get cm kube-proxy \
  -o jsonpath='{.data.config\.conf}' | grep -E '^mode:' \
  | tee ~/lab-evidence/5b/b6-kube-proxy-mode-cm.txt

kubectl -n kube-system logs -l k8s-app=kube-proxy --tail=200 2>/dev/null \
  | grep -iE 'proxier|proxy mode|using .*proxy' | head -5 \
  | tee ~/lab-evidence/5b/b6-kube-proxy-mode-log.txt

grep -qiE 'iptables|nftables|ipvs' ~/lab-evidence/5b/b6-kube-proxy-mode-log.txt \
  && echo 'PASS: log kube-proxy noi ro dataplane dang dung'
```

**PASS:** dòng `PASS:` xuất hiện. `mode:` rỗng trong ConfigMap nghĩa là kube-proxy dùng mode mặc
định của phiên bản đang chạy — đó là lý do phải đọc log chứ không suy từ file cấu hình.

### B6.3. Tìm ClusterIP của Service mình tạo trong rule của node

Trên `lab-k8s-master`:

```bash
CIP="$(kubectl -n lab-5b get svc web -o jsonpath='{.spec.clusterIP}')"
echo "$CIP" | tee ~/lab-evidence/5b/b6-clusterip.txt
```

Chạy trên **`lab-k8s-worker1`**, thay `<CIP>` bằng giá trị vừa in:

```bash
CIP='<CIP>'
if sudo iptables-save -t nat 2>/dev/null | grep -q "$CIP"; then
  echo 'PASS: ClusterIP xuat hien trong bang nat cua node (dataplane iptables)'
elif sudo nft list ruleset 2>/dev/null | grep -q "$CIP"; then
  echo 'PASS: ClusterIP xuat hien trong ruleset nftables cua node'
else
  echo 'FAIL: khong tim thay ClusterIP trong rule cua node'
fi
```

**PASS:** một dòng `PASS:`, không có dòng `FAIL:`.

**Ý nghĩa:** ClusterIP là một địa chỉ **ảo** — không interface nào mang nó. Thứ làm nó "tồn tại" là
tập rule mà kube-proxy sinh trên **mỗi node**. Đó cũng là lý do bài 164 xếp kube-proxy vào nhóm quản
trị viên phải lo, không phải nhóm người dùng.

### B6.4. `kubectl proxy` và apiserver proxy

Trên `lab-k8s-master`:

```bash
kubectl proxy --port=18001 >/tmp/lab5b-proxy.log 2>&1 &
PF_PID=$!
for i in $(seq 1 20); do
  curl -fsS http://127.0.0.1:18001/version >/dev/null 2>&1 && break
  sleep 1
done

curl -s http://127.0.0.1:18001/version | tee ~/lab-evidence/5b/b6-kubectl-proxy.txt
grep -q 'gitVersion' ~/lab-evidence/5b/b6-kubectl-proxy.txt \
  && echo 'PASS: goi API qua kubectl proxy bang HTTP thuan, khong gui token nao'

curl -s "http://127.0.0.1:18001/api/v1/namespaces/lab-5b/services/web:http/proxy/" \
  | tee ~/lab-evidence/5b/b6-apiserver-proxy.txt
grep -qx 'web-ok' ~/lab-evidence/5b/b6-apiserver-proxy.txt \
  && echo 'PASS: apiserver proxy cham toi Service ben trong cluster'

kill "$PF_PID"
```

**PASS:** hai dòng `PASS:` xuất hiện.

**Ý nghĩa:** ba loại proxy vừa lộ ra cùng lúc. `kubectl proxy` nhận HTTP từ bạn, nói HTTPS với
apiserver và **tự thêm header xác thực** — đó là lý do `curl` không cần token. Chặng tiếp theo là
**apiserver proxy**, cái bastion tích hợp trong apiserver, đưa bạn tới một ClusterIP mà máy ngoài
vốn không chạm được. Và chặng cuối tới Pod đi qua **kube-proxy**. Ba thứ, ba vị trí, cùng một
request.

Hệ quả bảo mật đáng nhớ: bất kỳ ai chạm được cổng mà `kubectl proxy` đang lắng nghe đều phát request
đi với danh nghĩa của bạn. Đừng bind nó ra ngoài localhost.

### B6.5. kube-proxy không hiểu HTTP

```bash
curl -s -o /dev/null -w '%{http_code}\n' "http://$CIP/"
curl -s -o /dev/null -w '%{http_code}\n' "http://$CIP/duong-dan-khong-ton-tai"
```

Hai lệnh cho kết quả như nhau, và đó chính là điểm cần thấy: kube-proxy chuyển tiếp **TCP**, nó
không đọc đường dẫn, không đọc header `Host`, không định tuyến theo path. Muốn `/shop` đi một nơi và
`/blog` đi nơi khác thì phải có một thứ đọc được HTTP đứng trước — đó là Ingress cộng ingress
controller, nội dung của B7 tới B9.

Không đặt gate `PASS:` cho bước này: hai mã trả về phụ thuộc ứng dụng phía sau chứ không phụ thuộc
kube-proxy, nên nó là bước quan sát để dẫn sang B7, không phải bước kiểm chứng.

---

## B7. Ingress khi chưa có controller

**Mục đích:** bài [12](../12-ingress-controllers-vi.md) mở đầu bằng đúng một câu: để Ingress hoạt
động thì **phải có một ingress controller đang chạy**. Mục này biến câu đó thành phép đo, trước khi
cài controller ở B8.

### B7.1. Cluster chưa có IngressClass nào

```bash
kubectl get ingressclass
test "$(kubectl get ingressclass --no-headers 2>/dev/null | wc -l)" = '0' \
  && echo 'PASS: cluster chua co IngressClass nao'

kubectl api-resources --api-group=gateway.networking.k8s.io 2>/dev/null \
  | tee ~/lab-evidence/5b/b7-gateway-api.txt
if grep -qi 'gateway' ~/lab-evidence/5b/b7-gateway-api.txt; then
  echo 'FAIL: cluster da co Gateway API - lab nay khong cai no'
else
  echo 'PASS: API group gateway.networking.k8s.io chua ton tai'
fi
```

**PASS:** hai dòng `PASS:`, không có dòng `FAIL:`.

**Ý nghĩa của lệnh thứ hai:** `Ingress` là kiểu có sẵn trong Kubernetes — bạn tạo được nó ngay bây
giờ. `Gateway` thì không: bài [13](../13-gateway-vi.md) nói Gateway API là **add-on** gồm các
CustomResourceDefinition, không cài CRD và một bản hiện thực thì không loại object nào trong bài đó
tồn tại. Kết quả rỗng ở trên là bằng chứng cho tính chất add-on đó. Lab dừng lại ở đây và **không**
cài Gateway API; xem dòng tương ứng ở [mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

### B7.2. Tạo Ingress khi chưa có ai hiện thực nó

Tạo `~/lab-work/5b/ingress-som.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-som
  namespace: lab-5b
spec:
  rules:
    - host: som.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

```bash
kubectl apply -f ~/lab-work/5b/ingress-som.yaml
kubectl -n lab-5b get ingress
kubectl -n lab-5b describe ingress web-som | tee ~/lab-evidence/5b/b7-ingress-som.txt

test -n "$(kubectl -n lab-5b get ingress web-som -o jsonpath='{.metadata.uid}')" \
  && echo 'PASS: API server chap nhan va luu object Ingress'
test -z "$(kubectl -n lab-5b get ingress web-som -o jsonpath='{.status.loadBalancer.ingress}')" \
  && echo 'PASS: .status cua Ingress rong - khong controller nao nhan viec'
test -z "$(kubectl -n lab-5b get ingress web-som -o jsonpath='{.spec.ingressClassName}')" \
  && echo 'PASS: ingressClassName van rong vi cluster chua co class mac dinh'
```

**PASS:** ba dòng `PASS:` xuất hiện.

### B7.3. Không có gì phục vụ trên cổng HTTP của node

```bash
W1IP="$(kubectl get node lab-k8s-worker1 \
  -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
echo "$W1IP" | tee ~/lab-evidence/5b/b7-worker1-ip.txt

if curl -s --max-time 3 -H 'Host: som.lab.local' -o /dev/null "http://$W1IP/"; then
  echo 'FAIL: co thu gi do dang phuc vu cong 80 tren node'
else
  echo 'PASS: khong co ingress controller nen cong 80 tren node khong tra loi'
fi
```

**PASS:** dòng `PASS:` xuất hiện.

**Ý nghĩa:** đây là toàn bộ khác biệt giữa **tài nguyên** và **bản hiện thực**. Object Ingress hợp
lệ, được lưu trong etcd, `describe` ra đầy đủ rule — và không chuyển một byte nào. Cùng một hình
dạng vấn đề bạn đã gặp ở B3 với NetworkPolicy, chỉ khác tên API.

Giữ nguyên Ingress `web-som`; B8 sẽ cho thấy nó tự sống lại khi controller xuất hiện.

---

## B8. Cài ingress controller

**Mục đích:** đưa vào cluster thành phần thứ hai của snapshot `02-net-ready`. Controller và phiên
bản đã chốt ở [mục 2.3](#23-phiên-bản-lab-5b-đưa-vào-chuỗi-snapshot).

### B8.1. Cài Helm trên `lab-k8s-master`

Đây là chart đầu tiên của chuỗi lab, nên cũng là chỗ Helm được đưa vào — đúng ghi chú ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00).

```bash
cd ~/lab-work/5b
curl -fsSLO "https://get.helm.sh/helm-${HELM_VER}-linux-amd64.tar.gz"
curl -fsSLO "https://get.helm.sh/helm-${HELM_VER}-linux-amd64.tar.gz.sha256sum"

sha256sum -c "helm-${HELM_VER}-linux-amd64.tar.gz.sha256sum" \
  && echo 'PASS: checksum ban tai ve khop gia tri nha phat hanh cong bo'

tar -xzf "helm-${HELM_VER}-linux-amd64.tar.gz"
sudo install -o root -g root -m 0755 linux-amd64/helm /usr/local/bin/helm
rm -rf linux-amd64 "helm-${HELM_VER}-linux-amd64.tar.gz" "helm-${HELM_VER}-linux-amd64.tar.gz.sha256sum"

helm version --short
helm version --short | grep -q "$HELM_VER" \
  && echo 'PASS: helm tren PATH dung version da chot o bang A1.4'
cd ~
```

**PASS:** hai dòng `PASS:` xuất hiện. Checksum fail thì **dừng**: không cài binary mà mình không xác
minh được nguồn.

### B8.2. Cài chart

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update

helm install traefik traefik/traefik \
  --version "$TRAEFIK_CHART" \
  --namespace traefik --create-namespace

kubectl -n traefik rollout status deployment/traefik --timeout=600s
kubectl -n traefik get pods -o wide
```

```bash
helm list -n traefik | tee ~/lab-evidence/5b/b8-helm-list.txt
grep -q "$TRAEFIK_CHART" ~/lab-evidence/5b/b8-helm-list.txt \
  && echo 'PASS: chart dang chay dung version da chot'

kubectl -n traefik get deployment traefik \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.image}{"\n"}{end}' \
  | tee ~/lab-evidence/5b/b8-traefik-image.txt
```

**PASS:** rollout thành công; dòng `PASS:` xuất hiện. Đối chiếu tag trong
`b8-traefik-image.txt` với cột `appVersion` của dòng *Traefik* trong
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) — hai giá trị phải khớp.
Không khớp nghĩa là bảng A1.4 và chart đã lệch nhau: dừng và cập nhật bảng đó, đừng chạy tiếp với
version lạ.

### B8.3. IngressClass mà controller tạo ra

```bash
kubectl get ingressclass -o wide | tee ~/lab-evidence/5b/b8-ingressclass.txt

IC="$(kubectl get ingressclass -o jsonpath='{.items[0].metadata.name}')"
echo "$IC" | tee ~/lab-evidence/5b/b8-ingressclass-name.txt
kubectl get ingressclass "$IC" -o jsonpath='{.spec.controller}{"\n"}'

test "$(kubectl get ingressclass --no-headers | wc -l)" = '1' \
  && echo 'PASS: cluster co dung mot IngressClass'
```

**PASS:** dòng `PASS:` xuất hiện.

Đánh dấu class này là mặc định của cluster, nếu chart chưa làm:

```bash
DEF="$(kubectl get ingressclass "$IC" \
  -o jsonpath='{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}')"
echo "is-default-class=$DEF"

if [ "$DEF" != 'true' ]; then
  kubectl annotate ingressclass "$IC" \
    ingressclass.kubernetes.io/is-default-class='true' --overwrite
fi

test "$(kubectl get ingressclass "$IC" \
  -o jsonpath='{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}')" = 'true' \
  && echo 'PASS: IngressClass da duoc danh dau la mac dinh cua cluster'

test "$(kubectl get ingressclass \
  -o jsonpath='{range .items[*]}{.metadata.annotations.ingressclass\.kubernetes\.io/is-default-class}{"\n"}{end}' \
  | grep -c '^true$')" = '1' \
  && echo 'PASS: dung mot class mac dinh - khong roi vao truong hop admission controller chan'
```

**PASS:** hai dòng `PASS:` xuất hiện.

**Ý nghĩa:** bài [11](../11-ingress-vi.md) cảnh báo có **nhiều hơn một** IngressClass mặc định thì
admission controller **chặn** việc tạo Ingress không ghi `ingressClassName`. Gate thứ hai đếm đúng
số class mang annotation `true` để bảo đảm cluster không rơi vào tình trạng đó.

### B8.4. Đưa controller ra ngoài cluster bằng NodePort

```bash
kubectl -n traefik get svc traefik -o wide
kubectl -n traefik get svc traefik -o jsonpath='{.spec.type}{"\t"}{.status.loadBalancer}{"\n"}'
```

Service của chart mặc định là `type: LoadBalancer`, và trên cluster bare-metal không có cloud
controller thì `EXTERNAL-IP` treo ở `<pending>` — không ai cấp địa chỉ cho nó. Đây đúng là loại
proxy thứ năm trong bài [164](../164-proxies-vi.md): cloud load balancer do nhà cung cấp đám mây
tạo, thứ mà cluster lab không có. MetalLB nằm ngoài baseline
([bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00)), nên lab dùng NodePort:

```bash
kubectl -n traefik patch svc traefik -p '{"spec":{"type":"NodePort"}}'

test "$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.type}')" = 'NodePort' \
  && echo 'PASS: Service cua controller da chuyen sang NodePort'

HTTP_NP="$(kubectl -n traefik get svc traefik \
  -o jsonpath='{.spec.ports[?(@.name=="web")].nodePort}')"
HTTPS_NP="$(kubectl -n traefik get svc traefik \
  -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}')"
printf '%s\n' "$HTTP_NP" | tee ~/lab-evidence/5b/b8-http-nodeport.txt
printf '%s\n' "$HTTPS_NP" | tee ~/lab-evidence/5b/b8-https-nodeport.txt

test -n "$HTTP_NP" && test -n "$HTTPS_NP" \
  && echo 'PASS: doc duoc ca hai nodePort http va https'
```

**PASS:** hai dòng `PASS:` xuất hiện. Nếu tên port trong Service không phải `web` và `websecure`,
đọc `kubectl -n traefik get svc traefik -o yaml` rồi thay tên trong hai lệnh `jsonpath` — con số
nodePort phải **đọc từ cluster**, không tự đặt.

### B8.5. Ingress tạo ở B7 tự sống lại

```bash
W1IP="$(cat ~/lab-evidence/5b/b7-worker1-ip.txt)"
HTTP_NP="$(cat ~/lab-evidence/5b/b8-http-nodeport.txt)"
IC="$(cat ~/lab-evidence/5b/b8-ingressclass-name.txt)"

kubectl -n lab-5b get ingress web-som -o wide

test -z "$(kubectl -n lab-5b get ingress web-som -o jsonpath='{.spec.ingressClassName}')" \
  && echo 'PASS: Ingress tao truoc khi co class van de trong ingressClassName'
```

**PASS:** dòng `PASS:` xuất hiện.

**Ý nghĩa:** class mặc định được áp **lúc tạo object**, không hồi tố. Ingress `web-som` ra đời khi
cluster chưa có IngressClass nào, nên nó ở lại với trường rỗng dù bây giờ đã có class mặc định. Đây
là cái bẫy vận hành thật: cài controller xong mà Ingress cũ vẫn nằm im, và người ta đi sửa
controller trong khi thứ phải sửa là chính object đó.

Tạo lại chính file YAML đó — không sửa một ký tự nào trong file:

```bash
kubectl delete -f ~/lab-work/5b/ingress-som.yaml
kubectl apply -f ~/lab-work/5b/ingress-som.yaml

test "$(kubectl -n lab-5b get ingress web-som -o jsonpath='{.spec.ingressClassName}')" = "$IC" \
  && echo 'PASS: lan tao lai da duoc gan class mac dinh'

curl -s --max-time 5 -H 'Host: som.lab.local' "http://$W1IP:$HTTP_NP/" \
  | tee ~/lab-evidence/5b/b8-ingress-som-sau.txt
grep -qx 'web-ok' ~/lab-evidence/5b/b8-ingress-som-sau.txt \
  && echo 'PASS: chinh Ingress tao o B7 gio da phuc vu that'
```

**PASS:** hai dòng `PASS:` xuất hiện.

**Ý nghĩa:** không sửa một dòng nào trong `ingress-som.yaml`, chỉ thêm một controller vào cluster và
tạo lại object, và tài nguyên nằm im từ B7 bắt đầu chuyển lưu lượng. Trường `spec.ingressClassName`
được điền do **admission controller** áp class mặc định — đúng cơ chế bài 11 và bài 12 mô tả, không
phải do bạn hay do chart ghi vào. Cách khác cho Ingress cũ là ghi thẳng `ingressClassName` vào
manifest rồi `kubectl apply`; lab chọn cách tạo lại để nhìn thấy cơ chế mặc định hoạt động.

Cột `ADDRESS` của `kubectl get ingress` có thể vẫn trống với Service `NodePort`, vì không có địa chỉ
load balancer nào để controller ghi vào `.status`. Bài 11 nói thẳng phần địa chỉ này phụ thuộc bản
hiện thực, nên **đừng lấy `ADDRESS` làm gate** — gate là phản hồi HTTP thật ở trên.

---

## B9. Ingress thật: host, path, pathType và TLS

**Mục đích:** đo những quy tắc so khớp mà bài [11](../11-ingress-vi.md) đặt ra, trên lưu lượng
thật.

### B9.1. Dựng ba backend phân biệt được

Tạo `~/lab-work/5b/backends.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shop
  namespace: lab-5b
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shop
  template:
    metadata:
      labels:
        app: shop
    spec:
      containers:
        - name: httpd
          image: busybox:1.37
          command:
            - sh
            - -c
            - 'mkdir -p /www && echo shop-ok > /www/index.html && httpd -f -p 8080 -h /www'
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: shop
  namespace: lab-5b
spec:
  selector:
    app: shop
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blog
  namespace: lab-5b
spec:
  replicas: 1
  selector:
    matchLabels:
      app: blog
  template:
    metadata:
      labels:
        app: blog
    spec:
      containers:
        - name: httpd
          image: busybox:1.37
          command:
            - sh
            - -c
            - 'mkdir -p /www && echo blog-ok > /www/index.html && httpd -f -p 8080 -h /www'
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: blog
  namespace: lab-5b
spec:
  selector:
    app: blog
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chinhxac
  namespace: lab-5b
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chinhxac
  template:
    metadata:
      labels:
        app: chinhxac
    spec:
      containers:
        - name: httpd
          image: busybox:1.37
          command:
            - sh
            - -c
            - 'mkdir -p /www && echo chinhxac-ok > /www/index.html && httpd -f -p 8080 -h /www'
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: chinhxac
  namespace: lab-5b
spec:
  selector:
    app: chinhxac
  ports:
    - port: 80
      targetPort: 8080
```

`httpd` của busybox trả cùng `index.html` cho mọi đường dẫn thư mục, nên nội dung trả về cho biết
**Service nào** đã nhận request — đúng thứ cần để đọc kết quả so khớp path.

```bash
kubectl apply -f ~/lab-work/5b/backends.yaml
kubectl -n lab-5b rollout status deploy/shop --timeout=180s
kubectl -n lab-5b rollout status deploy/blog --timeout=180s
kubectl -n lab-5b rollout status deploy/chinhxac --timeout=180s
kubectl -n lab-5b get svc

test "$(kubectl -n lab-5b get endpointslice \
  -l 'kubernetes.io/service-name in (shop,blog,chinhxac)' --no-headers | wc -l)" = '3' \
  && echo 'PASS: ba Service deu co EndpointSlice'
```

**PASS:** dòng `PASS:` xuất hiện.

### B9.2. Một Ingress, một host, ba path

Tạo `~/lab-work/5b/ingress-chinh.yaml` — nhớ thay `<IC>` bằng giá trị trong
`~/lab-evidence/5b/b8-ingressclass-name.txt`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lab-ingress
  namespace: lab-5b
spec:
  ingressClassName: <IC>
  rules:
    - host: shop.lab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: shop
                port:
                  number: 80
          - path: /blog
            pathType: Prefix
            backend:
              service:
                name: blog
                port:
                  number: 80
          - path: /chinhxac
            pathType: Exact
            backend:
              service:
                name: chinhxac
                port:
                  number: 80
```

```bash
IC="$(cat ~/lab-evidence/5b/b8-ingressclass-name.txt)"
sed -i "s|<IC>|$IC|" ~/lab-work/5b/ingress-chinh.yaml
grep -q "ingressClassName: $IC" ~/lab-work/5b/ingress-chinh.yaml \
  && echo 'PASS: da dien dung ten IngressClass vao manifest'

kubectl apply -f ~/lab-work/5b/ingress-chinh.yaml
kubectl -n lab-5b describe ingress lab-ingress | tee ~/lab-evidence/5b/b9-ingress-describe.txt
```

**PASS:** dòng `PASS:` xuất hiện.

Thử bỏ `pathType` để thấy nó là trường bắt buộc:

```bash
sed '/pathType: Exact/d' ~/lab-work/5b/ingress-chinh.yaml \
  > ~/lab-work/5b/ingress-thieu-pathtype.yaml
if kubectl apply --dry-run=server -f ~/lab-work/5b/ingress-thieu-pathtype.yaml >/dev/null 2>&1; then
  echo 'FAIL: manifest thieu pathType van qua duoc validation'
else
  echo 'PASS: path khong khai pathType bi tu choi tu phia server'
fi
```

**PASS:** dòng `PASS:` xuất hiện. `--dry-run=server` gửi object lên API server để validate mà
**không** ghi gì — không có object rác nào sinh ra từ bước này.

### B9.3. Đo bốn quy tắc so khớp

```bash
W1IP="$(cat ~/lab-evidence/5b/b7-worker1-ip.txt)"
HTTP_NP="$(cat ~/lab-evidence/5b/b8-http-nodeport.txt)"

goi() {
  curl -s --max-time 5 -H "Host: $1" "http://$W1IP:$HTTP_NP$2"
}

{
  echo "/            -> $(goi shop.lab.local /)"
  echo "/blog        -> $(goi shop.lab.local /blog)"
  echo "/blog/bai-1  -> $(goi shop.lab.local /blog/bai-1)"
  echo "/blogxyz     -> $(goi shop.lab.local /blogxyz)"
  echo "/chinhxac    -> $(goi shop.lab.local /chinhxac)"
} | tee ~/lab-evidence/5b/b9-so-khop.txt
```

```bash
goi shop.lab.local / | grep -qx 'shop-ok' \
  && echo 'PASS: path / khop tien to ngan nhat va di toi shop'
goi shop.lab.local /blog/bai-1 | grep -qx 'blog-ok' \
  && echo 'PASS: Prefix /blog khop ca path con /blog/bai-1'
goi shop.lab.local /blogxyz | grep -qx 'shop-ok' \
  && echo 'PASS: /blogxyz KHONG khop /blog - so khop theo tung phan tu path, khong theo chuoi'
goi shop.lab.local /blog | grep -qx 'blog-ok' \
  && echo 'PASS: path khop dai nhat thang - /blog di toi blog chu khong ve /'
```

**PASS:** bốn dòng `PASS:` xuất hiện.

**Ý nghĩa:** hai quy tắc của bài 11 vừa được đo. `Prefix` so khớp **theo từng phần tử path tách bởi
`/`**, nên `/blog` khớp `/blog/bai-1` nhưng **không** khớp `/blogxyz` — đây là chỗ trực giác "tiền
tố chuỗi" dẫn sai. Và khi nhiều path cùng khớp thì **path khớp dài nhất thắng**, nên `/blog` không
rơi về backend của `/`.

`Exact` không bỏ qua dấu gạch chéo cuối:

```bash
CX="$(goi shop.lab.local /chinhxac)"
CXS="$(goi shop.lab.local /chinhxac/)"
echo "chinhxac=$CX chinhxac-slash=$CXS"

echo "$CX" | grep -qx 'chinhxac-ok' \
  && echo 'PASS: Exact /chinhxac khop dung duong dan do'
if echo "$CXS" | grep -qx 'chinhxac-ok'; then
  echo 'FAIL: /chinhxac/ van khop Exact /chinhxac'
else
  echo 'PASS: /chinhxac/ KHONG khop Exact /chinhxac'
fi
```

**PASS:** hai dòng `PASS:`, không có dòng `FAIL:`.

Host phải khớp:

```bash
if goi khong-ton-tai.lab.local / | grep -qx 'shop-ok'; then
  echo 'FAIL: host khac van roi vao rule cua shop.lab.local'
else
  echo 'PASS: host khong khop thi khong dung rule cua host do'
fi
```

**PASS:** dòng `PASS:` xuất hiện.

**Ý nghĩa:** bài 11 nói **cả host lẫn path đều phải khớp** thì request mới được chuyển đi. Request
không khớp rule nào rơi về `defaultBackend`, mà `defaultBackend` theo quy ước là tùy chọn cấu hình
của controller chứ không khai trong tài nguyên Ingress — nên phản hồi ở trường hợp này do controller
quyết định, và lab chỉ gate ở mức "không phải `shop-ok`".

### B9.4. Từ ngoài cluster

Chạy trên **PowerShell của máy host Windows**. Thay hai giá trị bằng nội dung của
`~/lab-evidence/5b/b7-worker1-ip.txt` và `~/lab-evidence/5b/b8-http-nodeport.txt`:

```powershell
$w1 = '<IP cua lab-k8s-worker1>'
$np = '<nodePort http>'
$out = & curl.exe -s --max-time 5 -H 'Host: shop.lab.local' "http://${w1}:${np}/"
if ($out -eq 'shop-ok') { 'PASS: goi duoc tu may host qua Ingress' }
else { "FAIL: nhan duoc '$out'" }
```

**PASS:** dòng `PASS: goi duoc tu may host qua Ingress` xuất hiện. Đây là chứng minh "truy cập được
từ ngoài": request xuất phát từ một máy **không thuộc cluster**, đi vào NodePort của controller, rồi
được định tuyến theo `Host` và path tới đúng Service.

### B9.5. TLS kết thúc tại điểm ingress

Sinh certificate tự ký cho `shop.lab.local` trên `lab-k8s-master`:

```bash
openssl req -x509 -nodes -newkey rsa:2048 -days 30 \
  -keyout ~/lab-work/5b/tls.key -out ~/lab-work/5b/tls.crt \
  -subj '/CN=shop.lab.local' -addext 'subjectAltName=DNS:shop.lab.local'

kubectl -n lab-5b create secret tls shop-tls \
  --cert=~/lab-work/5b/tls.crt --key=~/lab-work/5b/tls.key

kubectl -n lab-5b get secret shop-tls -o jsonpath='{.type}{"\n"}'
test "$(kubectl -n lab-5b get secret shop-tls -o jsonpath='{.type}')" = 'kubernetes.io/tls' \
  && echo 'PASS: Secret dung kieu kubernetes.io/tls'
test -n "$(kubectl -n lab-5b get secret shop-tls -o jsonpath='{.data.tls\.crt}')" \
  && test -n "$(kubectl -n lab-5b get secret shop-tls -o jsonpath='{.data.tls\.key}')" \
  && echo 'PASS: Secret co du hai khoa tls.crt va tls.key'
```

**PASS:** hai dòng `PASS:` xuất hiện. Nếu `create secret tls` báo không tìm thấy file, thay `~` bằng
đường dẫn tuyệt đối `$HOME` — `kubectl` không tự khai triển dấu ngã trong giá trị cờ.

Thêm khối `tls` vào Ingress. Tạo `~/lab-work/5b/ingress-tls.yaml` bằng cách chép
`ingress-chinh.yaml` và chèn ngay dưới dòng `spec:`:

```yaml
  tls:
    - hosts:
        - shop.lab.local
      secretName: shop-tls
```

```bash
kubectl apply -f ~/lab-work/5b/ingress-tls.yaml
kubectl -n lab-5b get ingress lab-ingress \
  -o jsonpath='{range .spec.tls[*]}{.secretName}{"\t"}{.hosts[0]}{"\n"}{end}'

W1IP="$(cat ~/lab-evidence/5b/b7-worker1-ip.txt)"
HTTPS_NP="$(cat ~/lab-evidence/5b/b8-https-nodeport.txt)"

curl -k -s --max-time 5 \
  --resolve "shop.lab.local:$HTTPS_NP:$W1IP" \
  "https://shop.lab.local:$HTTPS_NP/" | tee ~/lab-evidence/5b/b9-tls.txt
grep -qx 'shop-ok' ~/lab-evidence/5b/b9-tls.txt \
  && echo 'PASS: HTTPS di qua ingress toi dung backend'

curl -k -s -o /dev/null -w '%{ssl_verify_result}\n' \
  --resolve "shop.lab.local:$HTTPS_NP:$W1IP" \
  "https://shop.lab.local:$HTTPS_NP/"
```

**PASS:** dòng `PASS:` xuất hiện. Giá trị `ssl_verify_result` khác `0` là **đúng** — certificate tự
ký không có chuỗi tin cậy, nên `-k` là bắt buộc; đó cũng là lý do cột `hosts` trong khối `tls` phải
khớp tường minh `host` trong `rules`, như bài 11 nhắc.

**Ý nghĩa:** TLS **kết thúc tại điểm ingress**. Đoạn từ máy bạn tới controller được mã hóa; đoạn từ
controller tới Service và Pod là **plaintext** — chính đoạn `http://` mà backend `shop` đang phục vụ
trên cổng 8080 và bạn đã gọi trực tiếp ở B9.3. Bài 11 cũng nói Ingress chỉ hỗ trợ **một** port TLS
là 443; đây là lý do nodePort `websecure` là cửa duy nhất cho HTTPS.

---

## B10. Cluster này là single-stack IPv4

**Mục đích:** bài [85](../85-dual-stack-vi.md) không dựng được trên cluster lab, nhưng ba khẳng định
của nó **kiểm chứng được bằng phản ứng của API server**.

### B10.1. Đọc họ IP của một Service thật

```bash
kubectl -n lab-5b get svc web -o jsonpath='{.spec.ipFamilyPolicy}{"\n"}'
kubectl -n lab-5b get svc web -o jsonpath='{range .spec.ipFamilies[*]}{@}{"\n"}{end}'
kubectl -n lab-5b get svc web -o jsonpath='{range .spec.clusterIPs[*]}{@}{"\n"}{end}'
kubectl -n lab-5b get svc web -o jsonpath='{.spec.clusterIP}{"\n"}'

test "$(kubectl -n lab-5b get svc web -o jsonpath='{.spec.ipFamilyPolicy}')" = 'SingleStack' \
  && echo 'PASS: ipFamilyPolicy mac dinh la SingleStack'
test "$(kubectl -n lab-5b get svc web -o jsonpath='{.spec.ipFamilies[0]}')" = 'IPv4' \
  && echo 'PASS: ho IP chinh la IPv4'
test "$(kubectl -n lab-5b get svc web \
  -o jsonpath='{range .spec.clusterIPs[*]}{@}{"\n"}{end}' | wc -l)" = '1' \
  && echo 'PASS: clusterIPs chi co dung mot phan tu'
test "$(kubectl -n lab-5b get svc web -o jsonpath='{.spec.clusterIP}')" \
   = "$(kubectl -n lab-5b get svc web -o jsonpath='{.spec.clusterIPs[0]}')" \
  && echo 'PASS: clusterIP bang phan tu dau cua clusterIPs'
```

**PASS:** bốn dòng `PASS:` xuất hiện.

**Ý nghĩa:** `clusterIPs` là trường **chính** dạng mảng; `clusterIP` là trường **thứ cấp** tính từ
mảng đó theo phần tử đầu của `ipFamilies`. Bạn đã nhìn thấy `clusterIPs` số nhiều trong mọi
`kubectl get svc -o yaml` từ giai đoạn trước mà chưa biết vì sao — lý do là dual-stack.

### B10.2. `RequireDualStack` bị từ chối

Tạo `~/lab-work/5b/svc-require-dualstack.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-require
  namespace: lab-5b
spec:
  ipFamilyPolicy: RequireDualStack
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

```bash
if kubectl apply -f ~/lab-work/5b/svc-require-dualstack.yaml \
     2>~/lab-evidence/5b/b10-loi-require.txt; then
  echo 'FAIL: cluster single-stack ma van tao duoc Service RequireDualStack'
else
  echo 'PASS: RequireDualStack bi tu choi tren cluster single-stack'
fi
cat ~/lab-evidence/5b/b10-loi-require.txt
```

**PASS:** dòng `PASS:` xuất hiện, và file lỗi ghi rõ lý do.

### B10.3. `PreferDualStack` tự quay về single-stack

Tạo `~/lab-work/5b/svc-prefer-dualstack.yaml` giống file trên, đổi tên thành `web-prefer` và
`ipFamilyPolicy` thành `PreferDualStack`:

```bash
sed -e 's/web-require/web-prefer/' -e 's/RequireDualStack/PreferDualStack/' \
  ~/lab-work/5b/svc-require-dualstack.yaml > ~/lab-work/5b/svc-prefer-dualstack.yaml

kubectl apply -f ~/lab-work/5b/svc-prefer-dualstack.yaml \
  && echo 'PASS: PreferDualStack duoc chap nhan'

test "$(kubectl -n lab-5b get svc web-prefer \
  -o jsonpath='{range .spec.ipFamilies[*]}{@}{"\n"}{end}' | wc -l)" = '1' \
  && echo 'PASS: Service chi duoc cap mot ho IP - da quay ve single-stack'
test "$(kubectl -n lab-5b get svc web-prefer -o jsonpath='{.spec.ipFamilies[0]}')" = 'IPv4' \
  && echo 'PASS: ho IP duoc cap la IPv4'

kubectl -n lab-5b delete svc web-prefer
```

**PASS:** ba dòng `PASS:` xuất hiện.

**Ý nghĩa:** hai giá trị nghe gần giống nhau nhưng hành xử ngược nhau khi cluster không bật
dual-stack: `PreferDualStack` **quay về** single-stack, còn `RequireDualStack` làm **việc tạo Service
qua API thất bại**. Manifest dùng `RequireDualStack` copy từ tài liệu về cluster như thế này sẽ chết
ngay ở bước apply, và thông báo lỗi bạn vừa lưu là thứ giúp nhận ra nguyên nhân trong ba giây.

---

## B11. Cleanup, bảy tầng gate và snapshot `02-net-ready`

**Mục đích:** xóa mọi object của bài học, **giữ lại** hạ tầng mới, và chứng minh cluster đủ khỏe để
đóng mốc.

### B11.1. Xóa object của lab, giữ hạ tầng

```bash
kubectl delete namespace lab-5b --wait=true --timeout=300s
kubectl delete namespace lab-5b-ext --wait=true --timeout=300s

for ns in lab-5b lab-5b-ext; do
  if kubectl get namespace "$ns" >/dev/null 2>&1; then
    echo "FAIL: namespace $ns van ton tai"
  else
    echo "PASS: namespace $ns da bien mat"
  fi
done

kubectl get ingress -A
kubectl get networkpolicy -A
test "$(kubectl get networkpolicy -A --no-headers 2>/dev/null | wc -l)" = '0' \
  && echo 'PASS: khong con NetworkPolicy nao tren cluster'
```

**PASS:** ba dòng `PASS:`, không có dòng `FAIL:`.

**Giữ lại — đây là nội dung của mốc mới:**

```bash
kubectl -n kube-system get daemonset calico-node
kubectl -n traefik get deployment traefik
kubectl get ingressclass
helm list -n traefik

test "$(kubectl -n kube-system get ds calico-node \
  -o jsonpath='{.status.numberReady}')" = '3' \
  && echo 'PASS: CNI moi van chay du ba node'
test "$(kubectl -n traefik get deployment traefik \
  -o jsonpath='{.status.readyReplicas}')" -ge 1 \
  && echo 'PASS: ingress controller van chay'
test "$(kubectl get ingressclass --no-headers | wc -l)" = '1' \
  && echo 'PASS: IngressClass mac dinh van con'
```

**PASS:** ba dòng `PASS:` xuất hiện. Xóa nhầm hai thứ này là làm hỏng chính mốc sắp chụp.

Dọn file tạm, giữ nguyên thư mục bằng chứng:

```bash
rm -f ~/lab-work/5b/app.yaml ~/lab-work/5b/backends.yaml \
      ~/lab-work/5b/np-0*.yaml ~/lab-work/5b/np-10-egress-dns-va-web.yaml \
      ~/lab-work/5b/ingress-som.yaml ~/lab-work/5b/ingress-chinh.yaml \
      ~/lab-work/5b/ingress-thieu-pathtype.yaml ~/lab-work/5b/ingress-tls.yaml \
      ~/lab-work/5b/svc-require-dualstack.yaml ~/lab-work/5b/svc-prefer-dualstack.yaml \
      ~/lab-work/5b/tls.crt ~/lab-work/5b/tls.key ~/lab-work/5b/calico.yaml
rmdir ~/lab-work/5b
test ! -e ~/lab-work/5b && echo 'PASS: manifest tam da duoc xoa het'
```

**PASS:** dòng `PASS:` xuất hiện. `rmdir` cố ý không dùng `-rf`: nó fail nếu còn file lạ, và
`test ! -e` biến điều đó thành gate thay vì im lặng bỏ qua. Certificate và key tự ký bị xóa ở đây —
không đưa chúng vào snapshot.

### B11.2. Chạy trọn bảy tầng gate A5.4

Chạy **đầy đủ bảy tầng** ở
[A5.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready), từ
[tầng 0](LAB-00-MOI-TRUONG-1.35.7.md#a541-tầng-0--prereq-os-còn-đúng-sau-khi-cluster-chạy) tới
[tầng 6](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet), rồi dọn resource
test theo [A5.4.8](LAB-00-MOI-TRUONG-1.35.7.md#a548-dọn-resource-test-ghi-evidence-và-chụp-snapshot).
Lab này thuộc đúng trường hợp A5.5 nói phải chạy cả bảy tầng: **lab vừa đụng vào mạng và CNI**.

Hai tầng phải **đọc lại theo CNI mới** — đây là khác biệt duy nhất so với văn bản của Lab 00:

| Tầng | Câu trong Lab 00 | Đọc lại thế nào sau lab 5b |
| --- | --- | --- |
| [A5.4.3](LAB-00-MOI-TRUONG-1.35.7.md#a543-tầng-2--node-condition-taint-và-podcidr) | "`kube-proxy` và `kube-flannel-ds` mỗi cái có đúng một Pod `Running` trên mỗi node" | Không còn DaemonSet `kube-flannel-ds` và không còn namespace `kube-flannel`. Thay bằng: `kube-proxy` (namespace `kube-system`) và `calico-node` (namespace `kube-system`) mỗi cái đúng một Pod `Running` trên mỗi node, cộng Deployment `calico-kube-controllers` `Running`. Ba node vẫn phải có ba `.spec.podCIDR` khác nhau, nhưng **Pod IP không nhất thiết nằm trong `/24` của node** — xem [B4.5](#b45-đọc-lại-cni-trên-node-và-kiểm-dải-ip-plugin-cấp) |
| [A5.4.4](LAB-00-MOI-TRUONG-1.35.7.md#a544-tầng-3--pod-networking-xuyên-node) | Ba nguyên nhân fail: UDP `8472` bị chặn, Flannel bind nhầm interface, `--pod-network-cidr` sai | Đường hầm giữa các node giờ do CNI mới dựng, không còn là VXLAN cổng `8472` của Flannel. Ba nguyên nhân đọc lại thành: lưu lượng đóng gói giữa các node bị chặn, plugin bind nhầm interface (gate ở [B4.5](#b45-đọc-lại-cni-trên-node-và-kiểm-dải-ip-plugin-cấp) bắt được lỗi này), hoặc IPPool lệch Pod CIDR |

Gate bổ sung riêng cho mốc mới, chạy sau khi bảy tầng đã PASS:

```bash
kubectl get pods -A -o wide | tee ~/lab-evidence/5b/b11-pods-toan-cluster.txt
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

grep -qi 'flannel' ~/lab-evidence/5b/b11-pods-toan-cluster.txt \
  && echo 'FAIL: van con Pod cua CNI cu' \
  || echo 'PASS: khong con dau vet cua CNI cu tren cluster'

for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  echo "== $n : kiem tren chinh node do"
done
```

Trên **từng node**:

```bash
test ! -e /etc/cni/net.d/10-flannel.conflist \
  && test -f /etc/cni/net.d/10-calico.conflist \
  && echo 'PASS: /etc/cni/net.d chi con cau hinh cua CNI moi'
if ip link show flannel.1 >/dev/null 2>&1; then
  echo 'FAIL: interface cu van con'
else
  echo 'PASS: khong con interface cua CNI cu'
fi
```

**PASS:** trên master, lệnh field selector trả `No resources found` và dòng
`PASS: khong con dau vet cua CNI cu` xuất hiện; trên **mỗi node**, hai dòng `PASS:` và không có dòng
`FAIL:`.

Ghi hồ sơ của mốc mới:

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
} | tee ~/lab-evidence/5b/b11-ho-so-02-net-ready.txt

test -s ~/lab-evidence/5b/b11-ho-so-02-net-ready.txt \
  && echo 'PASS: da ghi ho so cua moc 02-net-ready'
```

**PASS:** dòng `PASS:` xuất hiện.

### B11.3. Chụp snapshot `02-net-ready`

Tắt ba VM sạch sẽ rồi mới chụp. Chạy trên **từng node** theo thứ tự worker 2 → worker 1 → master:

```bash
sudo shutdown -h now
```

Chờ VMware Workstation hiển thị cả ba VM ở trạng thái *Powered off*. Chụp khi VM đã tắt để snapshot
không kèm trạng thái RAM.

Chụp trên **cả ba VM**: chuột phải VM → **Snapshot → Take Snapshot** → ô *Name* điền đúng nguyên
văn:

```text
02-net-ready
```

Ô *Description* ghi lab đã dựng và ngày chụp, ví dụ
`dung bang LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md, chup <ngay>`.

Quy tắc tên giống hệt mốc trước: đúng nguyên văn `02-net-ready` trên cả ba VM, không hậu tố theo VM,
không thêm ngày, không đổi hoa thường, không thừa khoảng trắng. **Giữ nguyên** snapshot
`01-cluster-ready`: nó vẫn là điểm quay lui khi cần dựng lại chuỗi từ đầu.

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
  if ($ok) { "PASS: $f" }
  else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** đúng ba dòng `PASS:`, không có dòng `FAIL:`. `-ccontains` so sánh phân biệt hoa thường nên
gate này bắt được cả lỗi gõ sai tên. Lab 5b kết thúc ở đây; để nguyên ba VM ở trạng thái tắt.

Từ lab sau, **điểm bắt đầu là `02-net-ready`**, không phải `01-cluster-ready` — xem
[chuỗi snapshot](README.md#3-chuỗi-snapshot).

---

## 3. Checkpoint 5b

Tự trả lời không nhìn tài liệu:

- [ ] Kể tên ba dải IP của cluster và thành phần cấp phát từng dải; giải thích vì sao đổi CNI làm
      Pod IP đổi nhưng ClusterIP thì không, và vì sao `ip addr` trên node cho ra nhiều địa chỉ hơn
      `node.status.addresses`.
- [ ] Nói ai nạp CNI plugin trên node, hai đường dẫn phải nhìn khi gỡ lỗi, và vì sao ở B4.3 phải
      restart containerd trước kubelet.
- [ ] **Xác nhận nợ #4 đã được trả ở lab này:** kể lại hai phép đo trên **cùng một file
      NetworkPolicy** trước và sau khi đổi CNI, nói kết quả mỗi lần và nêu chính xác thứ đã thay
      đổi giữa hai lần đo.
- [ ] Nêu bốn bước của quy trình đổi CNI theo đúng thứ tự, và hỏng gì xảy ra nếu đảo bước 1 với
      bước 2, hoặc bỏ hẳn bước 4.
- [ ] Giải thích vì sao `ports` trong NetworkPolicy phải là cổng của Pod đích chứ không phải cổng
      của Service, và triệu chứng khi điền nhầm.
- [ ] Vẽ lại hai chính sách của B5.4 và nói dấu gạch đầu dòng nào biến phép giao thành phép hợp.
- [ ] Giải thích vì sao một policy cô lập chiều đi làm hỏng phân giải tên, và đường tối thiểu phải
      mở lại gồm những gì; kèm quy tắc hai đầu (egress ở nguồn và ingress ở đích).
- [ ] Kể năm loại proxy của bài 164, nói mỗi loại chạy ở đâu, và chỉ ra ba loại bạn đã chạm tay
      trong B6.
- [ ] Giải thích vì sao Ingress tạo ở B7 không chuyển được gói tin nào, rồi tự sống lại ở B8 mà
      không sửa manifest; nói cơ chế đã điền `spec.ingressClassName` giúp bạn.
- [ ] Với `path: /blog`, `pathType: Prefix`, nói `/blog`, `/blog/bai-1` và `/blogxyz` cái nào khớp
      và vì sao; thêm quy tắc áp dụng khi nhiều path cùng khớp.
- [ ] Nói TLS của Ingress kết thúc ở đâu, đoạn nào sau đó là plaintext, Secret phải mang kiểu và
      hai khóa nào; và vì sao `hosts` trong `tls` phải khớp tường minh `host` trong `rules`.
- [ ] Phân biệt `SingleStack`, `PreferDualStack` và `RequireDualStack` bằng đúng phản ứng của API
      server trên cluster của bạn.

### Bài giải thích cuối cùng

Trong tối đa 10 phút, kể lại luồng sau mà không mở tài liệu:

1. Cluster có ba dải IP không chồng lấn. kube-apiserver cấp ClusterIP, kubelet cấp IP node, còn
   **network plugin** cấp IP Pod — nên plugin là thứ duy nhất trong ba cái đó bạn thay được mà
   không dựng lại cluster.
2. Bạn viết một NetworkPolicy chặn sạch chiều đến. API server nhận và lưu nó. Lưu lượng **vẫn
   qua**, vì plugin đang chạy không hiện thực NetworkPolicy — hợp đồng của CNI không bắt buộc điều
   đó.
3. Bạn gỡ plugin cũ ở tầng API, dọn conflist và interface trên từng node, restart containerd rồi
   kubelet, cài plugin mới bằng một manifest. Node rời `Ready` rồi quay lại.
4. Pod cũ vẫn mang IP của dải cũ cho tới khi bị tạo lại, nên bạn xóa Pod CoreDNS và apply lại
   manifest của lab. DNS và Service sống lại.
5. Bạn apply lại **đúng file policy cũ**. Lần này lưu lượng dừng, và trên node xuất hiện các chain
   do thành phần của plugin mới sinh ra. Nợ #4 được trả ở chính chỗ này.
6. Bạn mở đúng một cổng cho đúng một nhóm Pod, và học rằng cổng trong policy là cổng **Pod**. Bạn
   tách một dấu gạch đầu dòng và biến phép giao thành phép hợp. Bạn chặn chiều đi và mất luôn DNS.
7. Bạn tạo một Ingress khi chưa có controller: object hợp lệ, không byte nào chạy — cùng hình dạng
   vấn đề với bước 2.
8. Bạn cài ingress controller. Nó tạo IngressClass, và admission controller gán class mặc định vào
   Ingress cũ. Cùng manifest đó bắt đầu định tuyến theo `Host` và path — thứ kube-proxy không làm
   được vì nó không đọc HTTP.
9. Bạn kết thúc TLS tại ingress và nhớ rằng đoạn phía sau là plaintext.
10. Bạn dọn object của bài học, **giữ** plugin mới và controller, chạy trọn bảy tầng gate với hai
    dòng đọc lại cho CNI mới, rồi chụp `02-net-ready`.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm "object được chấp nhận" với "object có hiệu
lực", không nhầm cổng Service với cổng Pod, không nhầm giao với hợp, thì Lab 5b hoàn tất. Giai đoạn
6 — lưu trữ — là bước tiếp theo, và nó bắt đầu từ mốc bạn vừa chụp.

---

## 4. Troubleshooting của lab này

Sự cố khi dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ liệt kê
sự cố phát sinh từ nội dung bài học 5b.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| B3.4 ra `FAIL` vì policy chặn được ngay | `kubectl get pods -A -o wide \| grep -iE 'calico\|cilium'` | Cluster không ở `01-cluster-ready`; restore cả ba VM về mốc đó rồi làm lại từ B0 |
| B4.2: namespace `kube-flannel` kẹt `Terminating` | `kubectl get ns kube-flannel -o yaml`; `kubectl get all -n kube-flannel` | Chờ Pod hết grace period; đừng ép xóa finalizer khi chưa biết nguyên nhân |
| **B4.4: Pod của CNI mới `CrashLoopBackOff`** | `kubectl -n kube-system logs -l k8s-app=calico-node --tail=50` | Thường do B4.3 làm sót: còn conflist cũ hoặc còn interface cũ. Chạy lại B4.3 trên **cả ba node** rồi `kubectl -n kube-system rollout restart ds/calico-node` |
| **B4.5: IPPool khác Pod CIDR** | `kubectl get ippools.crd.projectcalico.org -o wide` và `cat ~/lab-evidence/5b/b1-pod-cidr.txt` | Gỡ CNI mới (`kubectl delete -f ~/lab-work/5b/calico.yaml`), bỏ comment `CALICO_IPV4POOL_CIDR` trong `~/lab-work/5b/calico.yaml`, đặt đúng giá trị `POD_CIDR`, chạy lại B4.3 và B4.4. Không sửa IPPool bằng tay khi Pod đã lấy IP |
| **B4.5: annotation IP của node khác InternalIP** | Cột `FAIL` của vòng lặp trong B4.5 | Plugin dò nhầm interface. Gỡ CNI mới, khai phương thức tự dò theo đúng interface giữ default route (`ip route \| awk '/default/ {print $5; exit}'`) trong manifest, rồi cài lại |
| **B4.6: node `Ready` nhưng Pod hai worker không thấy nhau** | `kubectl -n lab-5b exec client -- wget -q -T 3 -O- "http://<PodIP web-w2>:8080"` | Đúng tầng 3 của A5.4. Kiểm annotation IP của node trước, rồi tới lưu lượng đóng gói giữa hai node; UFW phải `inactive` như A4.4 |
| B4.6: CoreDNS `ContainerCreating` mãi | `kubectl -n kube-system describe pod -l k8s-app=kube-dns` | CNI chưa sẵn sàng trên node đang giữ Pod đó; chờ `calico-node` của node đó `Running` rồi xóa lại Pod CoreDNS |
| Không sửa được B4 trong vài phút | — | Tắt và **restore cả ba VM về `01-cluster-ready`**, chạy lại bảy tầng A5.4, rồi làm lại lab từ B0. Không debug một cluster đã lệch state |
| B5.2: Pod có label vẫn bị chặn | `kubectl -n lab-5b get svc web -o jsonpath='{.spec.ports[0].targetPort}'` | `ports` trong policy phải là cổng Pod (`targetPort`), không phải cổng Service |
| B5.4: cả hai biến thể đều cho qua | `kubectl get ns lab-5b-ext --show-labels` | Thiếu label `kubernetes.io/metadata.name`; tạo lại namespace bằng `kubectl create namespace` thay vì manifest tự viết |
| B5.5: `nslookup` vẫn chạy sau khi deny egress | `kubectl -n lab-5b get netpol deny-egress-client-ok -o yaml` | `podSelector` không chọn đúng Pod: `client-ok` phải mang label `access: "true"` |
| B7.3: cổng 80 của node trả lời dù chưa cài controller | `sudo ss -lntp \| grep ':80'` trên node | Có dịch vụ khác đang nghe cổng 80; dừng nó hoặc đổi phép đo sang một node sạch — đừng bỏ qua gate |
| B8.2: `helm install` báo không tương thích `kubeVersion` | `helm show chart traefik/traefik --version "$TRAEFIK_CHART"` | Chart và baseline lệch nhau; dừng và cập nhật [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) sau khi đối chiếu, không tự chọn chart khác |
| B8.4: `EXTERNAL-IP` của Service treo `<pending>` | `kubectl -n traefik get svc traefik` | Đúng hành vi trên cluster không có cloud provider; lab dùng NodePort. Không cài MetalLB — nó nằm ngoài baseline |
| B8.4: `jsonpath` nodePort trả rỗng | `kubectl -n traefik get svc traefik -o yaml` | Tên port khác `web`/`websecure`; đọc tên thật rồi thay vào lệnh |
| B8.5: Ingress `web-som` vẫn không phục vụ | `kubectl -n lab-5b get ingress web-som -o yaml` | `spec.ingressClassName` còn rỗng vì chưa có class mặc định; làm lại phần đánh dấu class ở B8.3 |
| B9.3: mọi path đều trả cùng một backend | `kubectl -n lab-5b describe ingress lab-ingress` | Ingress chưa được controller nhận (sai `ingressClassName`), hoặc `curl` thiếu header `Host` |
| B9.5: `create secret tls` báo không thấy file | Đường dẫn trong lệnh | `kubectl` không khai triển dấu ngã trong giá trị cờ; dùng `$HOME/lab-work/5b/tls.crt` |
| B11.1: namespace `lab-5b` kẹt `Terminating` | `kubectl get all -n lab-5b` | Chờ Pod hết grace period; nếu state vẫn lệch thì restore cả ba VM về `01-cluster-ready` và làm lại — **chưa chụp** `02-net-ready` |

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
- [Kubernetes v1.35 — Declare Network Policy](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)
- [Kubernetes v1.35 — Install a Network Policy Provider](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/)
- [Kubernetes v1.35 — Use Calico for NetworkPolicy](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/network-policy-provider/calico-network-policy/)
- [Calico — System requirements](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements) (nguồn của dải minor Kubernetes đã kiểm thử, dùng ở [mục 2.3](#23-phiên-bản-lab-5b-đưa-vào-chuỗi-snapshot))
- [Calico — Install on on-premises deployments](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises) (nguồn của quy trình cài bằng manifest ở B4)
- [Traefik Helm chart](https://github.com/traefik/traefik-helm-chart) (nguồn của repo chart ở B8)
- [Helm — Installing Helm](https://helm.sh/docs/intro/install/)
