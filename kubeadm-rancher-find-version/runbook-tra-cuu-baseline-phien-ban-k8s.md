# Runbook nhanh: chọn và xác minh baseline Kubernetes/Rancher

> **Mục tiêu:** tìm một bộ phiên bản dùng được cho homelab kubeadm trong khoảng 15–30 phút.
>
> **Nguyên tắc:** ưu tiên bộ đã chạy thực tế; chỉ tra lại thành phần chưa kiểm hoặc sắp thay đổi. Không lập danh sách mọi candidate, không dùng state machine, không tạo fingerprint và không chạy semantic gate dài.
>
> File `runbook-tra-cuu-baseline-phien-ban-k8s-candidate-first.md` là tài liệu audit chuyên sâu, không phải quy trình mặc định.

---

## 1. Khi nào baseline được chấp nhận?

Mỗi thành phần chỉ dùng một trong bốn trạng thái:

| Trạng thái | Ý nghĩa |
| --- | --- |
| `RUNNING` | Đã cài và smoke test thành công trên đúng cluster đích |
| `RENDERED` | Chưa cài, nhưng version/range hợp lệ và manifest hoặc Helm chart render thành công |
| `PENDING` | Chưa đủ bằng chứng; tiếp tục kiểm đúng thành phần này |
| `FAIL` | Có xung đột version hoặc phép thử thất bại |

Quy tắc kết luận:

- Bộ đang chạy: chấp nhận khi mọi thành phần bắt buộc là `RUNNING`.
- Bộ chuẩn bị cài: chấp nhận có điều kiện khi mọi thành phần là `RUNNING` hoặc `RENDERED`; sau khi cài phải đổi thành `RUNNING`.
- Một thành phần `PENDING` không làm các thành phần đã chạy trở thành không tương thích.
- Một thành phần `FAIL` mới buộc đổi version hoặc cấu hình liên quan.

`latest` không tự động là lỗi tương thích. Nếu workload dùng `latest` đã chạy, ghi trạng thái `RUNNING`, đồng thời lưu image digest để tái lập lần sau. Với bản cài mới, ưu tiên pin tag cố định hoặc digest.

---

## 2. Lối nhanh cho cluster đã chạy

Không chọn lại toàn bộ stack. Chụp trạng thái thật, điền bảng ở mục 4, rồi chỉ xử lý dòng `PENDING`.

### 2.1. Ghi version thật đang chạy

Chạy các lệnh read-only sau trên control plane:

```bash
lsb_release -ds
dpkg --print-architecture
containerd --version
kubeadm version -o short
kubelet --version
kubectl version
helm version --short

kubectl get nodes -o wide
helm list -A
kubectl get deployment,daemonset -A
kubectl get pods -A -o wide
```

Lấy image và digest thực tế của `cloudflared`:

```bash
kubectl -n cloudflare get pods \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .status.containerStatuses[*]}  image={.image}{"\n"}  imageID={.imageID}{"\n"}{end}{end}'
```

Nếu namespace khác `cloudflare`, thay bằng namespace thực tế. Giữ cả hai giá trị:

- `image`: cấu hình yêu cầu, ví dụ `cloudflare/cloudflared:latest`;
- `imageID`: digest image thật đã chạy, dùng để tái lập.

### 2.2. Smoke test phần đã cài

Chỉ cần các phép thử phản ánh luồng sử dụng thật:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get storageclass
kubectl get ingressclass
kubectl get ingress -A
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

PASS khi:

- tất cả node `Ready`;
- các Deployment/DaemonSet bắt buộc đã `Available`/`Ready`;
- Pod test đi qua Flannel và Traefik thành công;
- hostname public đi qua cloudflared tới đúng ứng dụng;
- PVC test được bind nếu dùng local-path-provisioner;
- không có lỗi lặp lại liên quan tới image pull, webhook hoặc API bị loại.

Kết quả runtime mạnh hơn việc một dự án không công bố compatibility matrix. Ghi `RUNNING`; không hạ thành `PENDING` chỉ vì thiếu matrix. Runtime test không phủ định một incompatibility hoặc cảnh báo bảo mật được upstream công bố rõ ràng.

---

## 3. Lối chọn mới từ đầu

Chỉ xét **một candidate tại một thời điểm**. Candidate đầu tiên qua năm gate ở mục 5 là baseline; không cần xếp hạng mọi tổ hợp có thể có.

### Bước 1 — Chốt ràng buộc cố định

Ghi đúng một dòng:

```text
Ubuntu=<release>  arch=<amd64|arm64>  topology=kubeadm
Rancher=<có|không>  Ingress=<tên>  CNI=<tên>  Addon=<danh sách hoặc N/A>
```

Không đưa MetalLB vào baseline nếu kiến trúc chỉ dùng Cloudflare Tunnel → Traefik `ClusterIP`.

### Bước 2 — Chọn Kubernetes minor

Nếu cần Rancher:

1. Chọn một Rancher bản final từ `rancher-stable`.
2. Mở support matrix của **đúng Rancher version**.
3. Lấy giao giữa dải Kubernetes của Rancher và các minor Kubernetes đang maintained.
4. Chọn minor cao nhất trong giao đó, miễn còn đủ thời gian sử dụng theo kế hoạch của lab.

Nếu không cần Rancher, chọn minor Kubernetes maintained phù hợp với addon bắt buộc.

Không cần lập bảng cho mọi minor. Nếu candidate thất bại một gate, lùi đúng một minor hoặc một version của component gây lỗi rồi thử lại.

Nguồn:

- Kubernetes maintained releases: <https://kubernetes.io/releases/>
- Rancher support matrix: <https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/>

### Bước 3 — Chốt exact Kubernetes patch và package

Chọn patch final mới nhất của minor đã chọn, rồi xác nhận ba package tồn tại trong repo `pkgs.k8s.io` của minor đó:

```bash
apt-cache madison kubeadm
apt-cache madison kubelet
apt-cache madison kubectl
```

Ghi nguyên Debian package version. Với cluster mới, dùng cùng patch cho `kubeadm`, `kubelet`, `kubectl` và control plane.

### Bước 4 — Chốt các component còn lại

Đi theo thứ tự sau:

1. containerd từ Ubuntu: ghi exact version trả về bởi `apt-cache policy containerd`; kiểm CRI v1 và `SystemdCgroup = true`.
2. Flannel: chọn final tag, dùng manifest của đúng tag và đúng Pod CIDR.
3. Helm: dùng Helm 3 đáp ứng yêu cầu tối thiểu của Rancher; ghi exact patch thật sẽ chạy.
4. cert-manager: chọn patch cuối của một minor còn supported và có dải Kubernetes chứa candidate.
5. Traefik: chọn final chart, đọc `kubeVersion`, ghi cả chart version và `appVersion`.
6. Rancher: ghi cả chart version và `appVersion`; kiểm support matrix, `kubeVersion` và render.
7. Addon: kiểm độc lập từng addon; không gom local-path, cloudflared và MetalLB thành một ô duy nhất.

Không nâng component đã `RUNNING` chỉ vì có bản mới hơn, trừ khi bản hiện tại hết support, có lỗi bảo mật áp dụng hoặc chặn component bắt buộc khác.

---

## 4. Phiếu baseline duy nhất

Không tạo candidate sheet riêng. Sao chép bảng này, điền exact version và một bằng chứng ngắn cho mỗi dòng:

| Thành phần | Exact version đang/chọn dùng | Trạng thái | Bằng chứng ngắn |
| --- | --- | --- | --- |
| Ubuntu + architecture |  |  | `lsb_release`, `dpkg` |
| Kubernetes |  |  | release maintained hoặc `kubectl version` |
| kubeadm/kubelet/kubectl |  |  | `apt-cache madison` hoặc version đã cài |
| containerd |  |  | version, CRI v1, systemd cgroup |
| Flannel |  |  | manifest pin + pod network test |
| Helm |  |  | `helm version --short` |
| cert-manager |  |  | supported range + render/runtime |
| Traefik chart / app |  |  | `kubeVersion` + render/runtime |
| cloudflared tag / digest |  |  | Pod Ready + HTTP end-to-end |
| local-path-provisioner |  |  | PVC bind test hoặc `N/A` |
| MetalLB |  |  | LoadBalancer IP test hoặc `N/A` |
| Rancher chart / app |  |  | matrix + metadata + render/runtime |

Kết luận cuối bảng dùng một trong ba câu:

```text
READY-RUNNING: mọi component bắt buộc đã chạy và smoke test PASS.
READY-CONDITIONAL: mọi component đã RUNNING/RENDERED; còn phải install test các dòng RENDERED.
PARTIAL: các dòng RUNNING giữ nguyên; chỉ các dòng PENDING/FAIL cần xử lý.
```

Review lại trước mỗi lần upgrade và tối thiểu mỗi 90 ngày. Không cần tính một ngày review bằng script từ lifecycle của mọi addon.

---

## 5. Năm gate bắt buộc

### Gate A — Dải version có giao nhau

Chỉ kiểm các cạnh có ràng buộc công bố:

- Rancher → Kubernetes: support matrix của đúng Rancher version;
- Rancher chart → Kubernetes: trường `kubeVersion`;
- cert-manager → Kubernetes: bảng Supported Releases;
- Traefik chart → Kubernetes: trường `kubeVersion`;
- Kubernetes → containerd: CRI v1 và yêu cầu runtime của Kubernetes.

Flannel không có Kubernetes matrix thì không tự động FAIL; kiểm manifest API và runtime test.

### Gate B — Exact artifact tồn tại

Không dùng version range trong phiếu cuối. Xác nhận đúng package, release tag, chart và manifest sẽ cài.

Ngoại lệ vận hành: workload hiện hữu dùng `latest` được ghi `RUNNING` nếu smoke test đạt, nhưng phải lưu digest. Không dùng một digest chưa kiểm để thay image đang chạy.

### Gate C — Render ba chart

Chỉ render cert-manager, Traefik và Rancher. Không cần pull chart ra thư mục, audit template internals hoặc tạo hash.

Trong các lệnh dưới, `<KUBERNETES_VERSION>` dùng dạng `1.x.y` không có tiền tố `v`.

```bash
helm template cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --version <CERT_MANAGER_VERSION> \
  --kube-version <KUBERNETES_VERSION> \
  --set crds.enabled=true >/dev/null

helm template traefik traefik/traefik \
  --namespace traefik \
  --version <TRAEFIK_CHART_VERSION> \
  --kube-version <KUBERNETES_VERSION> \
  --set providers.kubernetesIngress.enabled=true \
  --set providers.kubernetesGateway.enabled=false >/dev/null

helm template rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version <RANCHER_VERSION> \
  --kube-version <KUBERNETES_VERSION> \
  --api-versions cert-manager.io/v1 \
  --set hostname=rancher.example.com \
  --set ingress.ingressClassName=traefik \
  --set ingress.tls.source=rancher \
  --set certmanager.version=<CERT_MANAGER_VERSION> >/dev/null
```

Exit code `0` là `RENDERED`. Render không thay thế install test.

### Gate D — Kiểm metadata bằng mắt

```bash
helm show chart traefik/traefik --version <TRAEFIK_CHART_VERSION>
helm show chart rancher-stable/rancher --version <RANCHER_VERSION>
helm show chart oci://quay.io/jetstack/charts/cert-manager \
  --version <CERT_MANAGER_VERSION>
```

Đối chiếu `version`, `appVersion` và `kubeVersion` với phiếu. Đây là kiểm tra ngắn, không cần parser riêng.

### Gate E — Smoke test sau cài

Sau mỗi component:

```bash
helm list -A
kubectl get pods -A
kubectl get deployment,daemonset -A
kubectl get events -A --sort-by=.lastTimestamp | tail -30
```

Thêm đúng một test theo chức năng:

- Flannel: Pod ở hai node liên lạc được;
- local-path: PVC chuyển `Bound` và Pod đọc/ghi được;
- Traefik: HTTP qua Ingress trả đúng ứng dụng;
- cloudflared: hostname public trả đúng ứng dụng;
- cert-manager: ba Deployment Ready và webhook hoạt động;
- Rancher: Deployment Ready, HTTPS nội bộ qua Traefik thành công và UI mở được.

Khi test đạt, đổi `RENDERED` thành `RUNNING`.

---

## 6. Fast path cho baseline hiện tại của lab

Dựa trên kết quả cài đặt đã có, không chọn lại Ubuntu, Kubernetes, containerd, Flannel, Traefik, cloudflared, cert-manager hay storage. Chỉ chụp exact version/digest thật bằng mục 2 và giữ chúng là `RUNNING`.

Phiếu nên có dạng:

| Thành phần | Candidate | Trạng thái cần ghi |
| --- | --- | --- |
| Ubuntu | exact point release, ví dụ `24.04.4`, và `amd64` từ máy thật | `RUNNING` |
| Kubernetes | `v1.35.6`; package `1.35.6-1.1` | `RUNNING` |
| containerd | exact output trên node, không chỉ ghi `2.x` | `RUNNING` |
| Flannel | `v0.28.7` | `RUNNING` |
| Traefik | chart `41.0.2` / app `v3.7.6` | `RUNNING` |
| cloudflared | `latest` + digest đang chạy | `RUNNING`, kèm cảnh báo tái lập |
| cert-manager | `v1.21.1` | `RUNNING` nếu ba Deployment/webhook đã PASS |
| local-path-provisioner | `v0.0.36` | `RUNNING` nếu PVC test đã PASS |
| MetalLB | `v0.16.1` hoặc `N/A` | chỉ `RUNNING` nếu thực sự cài và test |
| Rancher | chart `2.14.3` / app `v2.14.3` | `PENDING` cho tới khi Gate A–E PASS |

Đối với Rancher 2.14.3, chỉ làm bốn việc:

1. Xác nhận support matrix Rancher 2.14.3 ghi dải `1.33–1.35` cho “All Other Distros / Any” imported cluster; đồng thời ghi rõ kubeadm host không phải host platform được chứng nhận.
2. `helm show chart` phải trả chart `2.14.3`, app `v2.14.3` và `kubeVersion` chứa `1.35.6`.
3. Render theo Gate C với cert-manager `v1.21.1` và IngressClass `traefik`.
4. Cài theo runbook VMware, chờ Deployment Ready và test HTTPS/UI.

Nếu bước 1–3 PASS nhưng chưa cài, kết luận:

```text
READY-CONDITIONAL: stack hiện tại RUNNING; Rancher 2.14.3 RENDERED, chờ install test.
```

Nếu bước 4 PASS:

```text
READY-RUNNING: toàn bộ baseline, gồm Rancher 2.14.3, đã được kiểm trên cluster đích.
```

Lưu ý phạm vi: Rancher chạy trên chính cluster kubeadm có thể tương thích kỹ thuật và hoạt động trong homelab, nhưng không được gọi là topology Rancher Manager host được SUSE chứng nhận end-to-end nếu support matrix không liệt kê kubeadm ở bảng host platform.

---

## 7. Nguồn official tối thiểu

Chỉ mở nguồn liên quan tới candidate đang xét:

| Thành phần | Nguồn |
| --- | --- |
| Kubernetes releases | <https://kubernetes.io/releases/> |
| Kubernetes packages | <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/> |
| Container runtime | <https://kubernetes.io/docs/setup/production-environment/container-runtimes/> |
| Rancher support matrix | <https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/> |
| Rancher stable channel | <https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/choose-a-rancher-version> |
| cert-manager releases | <https://cert-manager.io/docs/releases/> |
| Traefik chart releases | <https://github.com/traefik/traefik-helm-chart/releases> |
| Flannel releases | <https://github.com/flannel-io/flannel/releases> |
| local-path-provisioner | <https://github.com/rancher/local-path-provisioner/releases> |
| MetalLB releases | <https://github.com/metallb/metallb/releases> |
| cloudflared releases | <https://github.com/cloudflare/cloudflared/releases> |

Quy tắc bằng chứng: một URL official và một dòng ghi kết quả là đủ. Chỉ lưu screenshot/HTML khi trang support matrix khó truy cập hoặc nội dung thay đổi động.

---

## 8. Những gì cố ý bỏ khỏi quy trình cũ

- Không duyệt và xếp hạng mọi Kubernetes minor maintained.
- Không tạo `SURVIVE`, `RESERVE`, `ELIMINATED` hoặc semantic state machine.
- Không bắt mọi addon dùng chung một cột.
- Không tạo file evidence cho mọi lệnh đã chạy.
- Không phân tích toàn bộ Helm template, `lookup`, capability site hoặc tạo SHA256 fingerprint.
- Không tính lifecycle limit của mọi artifact bằng script.
- Không biến thiếu compatibility matrix thành lỗi.
- Không phủ nhận bằng chứng runtime của một stack đang hoạt động.

Các kiểm tra sâu chỉ dùng khi render/install thực sự lỗi, khi chuẩn bị production có SLA, hoặc khi audit bảo mật yêu cầu khả năng tái lập tuyệt đối.
