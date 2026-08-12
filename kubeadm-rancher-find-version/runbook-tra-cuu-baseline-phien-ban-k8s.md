# Runbook nhanh: chọn và xác minh baseline Kubernetes/Rancher

> **Mục tiêu:** tìm một bộ phiên bản dùng được cho homelab kubeadm bằng số gate tối thiểu nhưng đủ bắt các xung đột version quan trọng.
>
> **Nguyên tắc:** ưu tiên bộ đã chạy thực tế; chỉ tra lại thành phần chưa kiểm hoặc sắp thay đổi. Không lập danh sách mọi candidate, không dùng state machine, không tạo fingerprint và không chạy semantic gate dài.
>
> File `runbook-tra-cuu-baseline-phien-ban-k8s-candidate-first.md` là tài liệu audit chuyên sâu, không phải quy trình mặc định.
>
> Lối kiểm cluster đã chạy ở §2 thường có thể hoàn thành trong 15–30 phút khi cluster, mạng và công cụ đã sẵn sàng. Lối chọn mới ở §3 không cam kết thời gian.

---

## 1. Khi nào baseline được chấp nhận?

Mỗi thành phần chỉ dùng một trong bốn trạng thái:

| Trạng thái | Ý nghĩa |
| --- | --- |
| `RUNNING` | Đã cài và smoke test thành công trên đúng cluster đích |
| `VERIFIED` | Chưa cài, nhưng version/range hợp lệ và exact artifact đã được xác minh đúng cách mà Gate B quy định cho **loại artifact đó** |
| `PENDING` | Chưa đủ bằng chứng; tiếp tục kiểm đúng thành phần này |
| `FAIL` | Có xung đột version hoặc phép thử thất bại |

`VERIFIED` áp dụng cho mọi loại artifact, không riêng chart: chart render exit `0`, manifest đọc được tại exact tag, Debian package có trong APT `Packages` của đúng repo/suite/architecture, image resolve được digest. Một thành phần đã xác minh đủ nhưng chưa cài là `VERIFIED`, **không** phải `PENDING` — `PENDING` chỉ dành cho trường hợp còn thiếu bằng chứng. Trạng thái này trước đây tên là `RENDERED`; phiếu cũ ghi `RENDERED` đọc là `VERIFIED`.

Quy tắc kết luận:

- Bộ đang chạy: chấp nhận khi mọi thành phần bắt buộc là `RUNNING`.
- Bộ chuẩn bị cài: chấp nhận có điều kiện khi mọi thành phần là `RUNNING` hoặc `VERIFIED`; sau khi cài phải đổi thành `RUNNING`.
- Một thành phần `PENDING` không làm các thành phần đã chạy trở thành không tương thích.
- Một thành phần `FAIL` mới buộc đổi version hoặc cấu hình liên quan.

`latest`, `stable`, `2.x` hoặc “gói từ Ubuntu” là **selector động**, không tự động là lỗi. Nếu tài liệu official của nhà phát hành dùng hoặc cho phép selector đó, ghi lại URL và ngày kiểm. Sau đó resolve selector thành artifact thực tế (exact package, release tag hoặc image digest). Chưa resolve được thì `PENDING`, không phải `FAIL`.

Chỉ ghi `FAIL` khi có ít nhất một bằng chứng rõ ràng:

- tài liệu official nói candidate không được hỗ trợ;
- metadata của exact artifact loại Kubernetes/architecture đích;
- artifact không tồn tại trong nguồn official đã khai;
- render hoặc runtime test thất bại vì incompatibility đã xác định.

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
3. Helm: dùng Helm 3 đồng thời thỏa yêu cầu của Rancher và dải skew Helm → Kubernetes; ghi exact patch thật sẽ chạy.
4. cert-manager: chọn patch cuối của một minor còn supported và có dải Kubernetes chứa candidate.
5. Traefik: chọn final chart, đọc `kubeVersion`, ghi cả chart version và `appVersion`.
6. Rancher: ghi cả chart version và `appVersion`; kiểm support matrix, `kubeVersion` và render.
7. Addon: kiểm độc lập từng addon; không gom local-path, cloudflared và MetalLB thành một ô duy nhất.

Không nâng component đã `RUNNING` chỉ vì có bản mới hơn, trừ khi bản hiện tại hết support, có lỗi bảo mật áp dụng hoặc chặn component bắt buộc khác.

---

## 4. Phiếu baseline duy nhất

Không tạo candidate sheet riêng. Tách selector động khỏi artifact đã resolve và tham chiếu nguồn bằng `Source ID` ở §6:

| Thành phần | Selector/chính sách | Artifact đã resolve | Trạng thái | Source ID + bằng chứng runtime |
| --- | --- | --- | --- | --- |
| Ubuntu + architecture | `24.04.x LTS`, `amd64` | point release đang cài |  | `S-UBUNTU-POINT`, `S-UBUNTU-LIFECYCLE` + `lsb_release` |
| Kubernetes | minor maintained | exact patch |  | `S-K8S-RELEASES` + `kubectl version` |
| kubeadm/kubelet/kubectl | repo theo minor | exact Debian package |  | `S-K8S-PACKAGES` + APT metadata |
| containerd | Ubuntu archive, major `2.x` | exact Debian/upstream version |  | `S-UBUNTU-CONTAINERD`, `S-K8S-RUNTIME` + runtime |
| Flannel | final release | exact tag và image |  | `S-FLANNEL-RELEASES` + manifest/network test |
| Helm | Helm 3 phù hợp Rancher/Kubernetes | exact patch |  | `S-HELM3-SKEW`, `S-RANCHER-HELM` + `helm version` |
| cert-manager | supported minor | exact patch/chart |  | `S-CERT-RELEASES`, `S-RANCHER-INSTALL` + render/runtime |
| Traefik | final chart | chart / `appVersion` |  | `S-TRAEFIK-RELEASES` + metadata/render/runtime |
| cloudflared | vendor `latest` hoặc final tag | digest + runtime version |  | `S-CLOUDFLARED-K8S`, `S-K8S-IMAGES` + Pod/HTTP test |
| local-path-provisioner | final release | exact tag/image |  | `S-LOCALPATH-RELEASES` + PVC test hoặc `N/A` |
| MetalLB | final release | exact tag/images |  | `S-METALLB-RELEASES` + LB test hoặc `N/A` |
| Rancher | `rancher-stable` | chart / `appVersion` |  | `S-RANCHER-MATRIX`, `S-RANCHER-CHANNEL` + metadata/render/runtime |

Mỗi claim dùng để chọn hoặc loại version phải có ít nhất một nguồn official. Một dòng có thể cần nhiều nguồn; không ép quan hệ một dòng–một URL. Mỗi cạnh Gate A phải tham chiếu nguồn riêng.

Kết luận cuối bảng dùng một trong ba câu:

```text
READY-RUNNING: mọi component bắt buộc đã chạy và smoke test PASS.
READY-CONDITIONAL: mọi component đã RUNNING/VERIFIED; còn phải install test các dòng VERIFIED.
PARTIAL: các dòng RUNNING giữ nguyên; chỉ các dòng PENDING/FAIL cần xử lý.
```

Review lại trước mỗi lần upgrade và tối thiểu mỗi 90 ngày. Không cần tính một ngày review bằng script từ lifecycle của mọi addon.

---

## 5. Năm gate bắt buộc

### Gate A — Dải version có giao nhau

Kiểm mọi cạnh áp dụng trong bảng sau:

| Cạnh | Khi áp dụng | Nguồn tối thiểu |
| --- | --- | --- |
| Rancher → Kubernetes | Có Rancher | `S-RANCHER-MATRIX` của đúng Rancher version |
| Rancher chart → Kubernetes | Có Rancher | `kubeVersion` từ exact chart |
| Rancher → Helm | Có Rancher | `S-RANCHER-HELM` và release notes exact Rancher |
| Helm → Kubernetes | Luôn có Helm | `S-HELM3-SKEW` hoặc `S-HELM4-SKEW` đúng major đang dùng |
| cert-manager → Kubernetes | Có cert-manager | `S-CERT-RELEASES` |
| Rancher chart → cert-manager | `ingress.tls.source=rancher` hoặc `letsEncrypt` | `S-RANCHER-INSTALL`, exact chart helper/metadata và render |
| Traefik chart → Kubernetes | Có Traefik | `kubeVersion` từ exact chart |
| Kubernetes → containerd | Luôn có containerd | `S-K8S-RUNTIME` và metadata/runtime containerd |
| Flannel → Kubernetes | Có Flannel | `S-FLANNEL-RELEASES` và mọi `apiVersion` trong manifest tại exact tag |
| local-path-provisioner → Kubernetes | Có local-path | `S-LOCALPATH-RELEASES` và mọi `apiVersion` trong manifest tại exact tag |
| MetalLB → Kubernetes | Có MetalLB | `S-METALLB-RELEASES` và mọi `apiVersion` trong manifest tại exact tag |

Với mỗi cạnh, ghi bốn trường:

```text
Raw constraint: <trích nguyên văn dòng/range quyết định>
Lower bound:    <giá trị | NOT-PUBLISHED | N/A-CAPABILITY>
Upper bound:    <giá trị | NOT-PUBLISHED | N/A-CAPABILITY>
Decision:       PASS | PENDING | FAIL
```

Ví dụ:

```text
Edge: Helm 3.21.x → Kubernetes
Raw constraint: 3.21.x | 1.36.x - 1.33.x
Lower bound: 1.33.x
Upper bound: 1.36.x
Decision: PASS với Kubernetes 1.35.x

Edge: Traefik chart → Kubernetes
Raw constraint: kubeVersion: ">=1.25.0-0"
Lower bound: 1.25.0-0
Upper bound: NOT-PUBLISHED
Decision: PASS với Kubernetes 1.35.6
```

Không ghi chung chung “compatible” hoặc chỉ `PASS`. Nếu upstream chỉ công bố một biên, biên còn lại ghi `NOT-PUBLISHED`; không tự suy diễn. Với ràng buộc capability như CRI v1, hai biên ghi `N/A-CAPABILITY`.

Ba cạnh cuối là ca **upstream không công bố compatibility matrix**. Không tự động FAIL và cũng không bỏ trống cạnh: ghi `NOT-PUBLISHED` cho cả hai biên, rồi lấy bằng chứng thay thế bằng cách đọc mọi `apiVersion` trong manifest tại exact tag và đối chiếu với các API đã bị loại ở Kubernetes minor đang chọn. `Decision: PASS` khi không `apiVersion` nào bị loại; có một cái bị loại thì `FAIL`. Runtime test tương ứng ở Gate E mới chốt cạnh này thành `RUNNING`.

### Gate B — Exact artifact tồn tại

Selector động được chấp nhận khi **official docs của đúng vendor** công bố selector/chính sách đó. Gate B có hai phần:

1. `VERIFY POLICY`: lưu URL official, raw rule liên quan và ngày kiểm;
2. `RESOLVE ARTIFACT`: ghi exact artifact mà máy sẽ cài hoặc đang chạy.

Trang danh sách release/tag chỉ dùng để tìm candidate. Bằng chứng cuối phải trỏ tới exact artifact hoặc metadata exact của nó:

| Loại artifact | Cách xác minh bắt buộc |
| --- | --- |
| Debian package | APT `Packages` metadata của đúng repo/minor hoặc suite/component/architecture; ghi nguyên version từ `apt-cache policy`/`madison` |
| GitHub/GitLab release | exact `/releases/tag/<tag>`; không dùng trang danh sách releases làm bằng chứng cuối |
| Dự án chỉ phát hành tag | exact tag/commit URL; trang `/tags` chung chỉ dùng để discovery |
| Helm chart HTTP repo | official repo URL + exact entry trong `index.yaml` hoặc output `helm show chart --version` |
| Helm chart OCI | output `helm show chart <oci-url> --version <exact>` |
| Manifest | raw URL tại exact tag; đọc `apiVersion` và mọi image tag/digest bên trong |
| Rolling image tag | official policy cho selector + digest đã pull + version do binary tự báo |

Với `latest`, không tự coi final release gần nhất là artifact đang chạy. Chỉ ánh xạ digest/runtime version sang final tag khi vendor metadata hoặc binary version chứng minh chúng là cùng release.

Lệnh snapshot cho package động:

```bash
apt-cache policy containerd
dpkg-query -W -f='${Package} ${Version} ${Architecture}\n' containerd
containerd --version
```

Lệnh snapshot image Kubernetes đang chạy:

```bash
kubectl -n <namespace> get pods \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{range .status.containerStatuses[*]}  image={.image}{"\n"}  imageID={.imageID}{"\n"}{end}{end}'

kubectl -n <namespace> exec deployment/<deployment> -- <binary> --version
```

Nếu chưa cài và máy có sẵn công cụ inspect registry (`docker buildx imagetools`, `skopeo` hoặc công cụ tương đương), dùng nó để lấy digest. Không cài thêm công cụ chỉ cho gate này; chưa resolve được digest thì giữ `PENDING` và resolve ngay sau lần pull/install đầu tiên.

Workload hiện hữu dùng `latest` được ghi `RUNNING` nếu official docs xác nhận selector, Pod Ready, smoke test đạt và đã lưu digest. Không dùng một digest chưa kiểm để thay image đang chạy.

Nếu vendor không công bố version cụ thể nhưng công bố lifecycle/update policy, policy đó là bằng chứng hợp lệ. Ghi `PENDING` cho tới khi có artifact/runtime snapshot; không suy diễn `FAIL` chỉ từ việc thiếu số version.

### Gate C — Render ba chart

Chỉ render cert-manager, Traefik và Rancher. Không cần pull chart ra thư mục, audit template internals hoặc tạo hash.

Trong các lệnh dưới, `<KUBERNETES_VERSION>` dùng dạng `1.x.y` không có tiền tố `v`.

Khai repo trước, vì Gate C và Gate D đều gọi chart qua alias `traefik/` và `rancher-stable/`. Bỏ bước này thì hai gate báo `repo traefik not found` chứ không phải lỗi version:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

`rancher-stable` phải trỏ đúng channel đã chốt ở phiếu. Dùng channel khác thì đổi URL theo `S-RANCHER-CHANNEL`, đừng giữ nguyên alias. cert-manager gọi thẳng bằng OCI URL nên không cần repo.

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

Exit code `0` là bằng chứng `VERIFIED` cho ba dòng chart. Render không thay thế install test.

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

Khi test đạt, đổi `VERIFIED` thành `RUNNING`.

---

## 6. Nguồn official tối thiểu

Chỉ mở nguồn liên quan tới candidate đang xét:

| Source ID | Claim | Nguồn |
| --- | --- | --- |
| `S-K8S-RELEASES` | Maintained minor, exact patch, lifecycle | <https://kubernetes.io/releases/> |
| `S-K8S-PACKAGES` | Repo/package kubeadm, kubelet, kubectl | <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/> |
| `S-K8S-RUNTIME` | CRI/runtime requirements | <https://kubernetes.io/docs/setup/production-environment/container-runtimes/> |
| `S-K8S-IMAGES` | Tag và digest image | <https://kubernetes.io/docs/concepts/containers/images/> |
| `S-HELM3-SKEW` | Helm 3 → Kubernetes | <https://helm.sh/docs/v3/topics/version_skew/> |
| `S-HELM4-SKEW` | Helm 4 → Kubernetes | <https://helm.sh/docs/topics/version_skew/> |
| `S-UBUNTU-POINT` | Noble point release/ISO/architecture | <https://releases.ubuntu.com/noble/> |
| `S-UBUNTU-LIFECYCLE` | Ubuntu lifecycle | <https://ubuntu.com/about/release-cycle> |
| `S-UBUNTU-APT` | Cách APT resolve `Installed`/`Candidate` | <https://documentation.ubuntu.com/server/tutorial/managing-software/> |
| `S-UBUNTU-CONTAINERD` | containerd package Noble | <https://packages.ubuntu.com/noble-updates/containerd> |
| `S-RANCHER-MATRIX` | Rancher → Kubernetes/platform | <https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/> |
| `S-RANCHER-CHANNEL` | Stable/latest/alpha policy | <https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/choose-a-rancher-version> |
| `S-RANCHER-HELM` | Rancher → Helm | <https://ranchermanager.docs.rancher.com/v2.14/getting-started/installation-and-upgrade/resources/helm-version-requirements> |
| `S-RANCHER-INSTALL` | TLS source và cert-manager | <https://ranchermanager.docs.rancher.com/v2.14/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster/> |
| `S-CERT-RELEASES` | cert-manager → Kubernetes/lifecycle | <https://cert-manager.io/docs/releases/> |
| `S-TRAEFIK-RELEASES` | Traefik exact chart release | <https://github.com/traefik/traefik-helm-chart/releases> |
| `S-FLANNEL-RELEASES` | Flannel exact release | <https://github.com/flannel-io/flannel/releases> |
| `S-LOCALPATH-RELEASES` | local-path exact release | <https://github.com/rancher/local-path-provisioner/releases> |
| `S-METALLB-RELEASES` | MetalLB exact release | <https://github.com/metallb/metallb/releases> |
| `S-CLOUDFLARED-K8S` | Cloudflare Kubernetes selector `latest` | <https://developers.cloudflare.com/tunnel/deployment-guides/kubernetes/> |
| `S-CLOUDFLARED-POLICY` | cloudflared support/update policy | <https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/> |

Quy tắc pin URL:

> Với claim phụ thuộc version, dùng URL đã pin tới phạm vi hẹp nhất mà vendor cung cấp: major, minor hoặc exact artifact. URL rolling chỉ dùng để chứng minh current policy và phải kèm ngày kiểm cùng raw constraint. Exact artifact phải dùng exact release/tag/chart/package evidence; trang danh sách chỉ dùng để discovery.

Mỗi dòng §4 phải tham chiếu ít nhất một `Source ID` cho từng claim dùng để quyết định. Một dòng có thể có nhiều nguồn. Mỗi cạnh Gate A phải lưu raw constraint ngắn, lower/upper bound và URL official tương ứng.

Runbook không lưu kèm snapshot baseline của một lab cụ thể. Kết quả mỗi lần tra cứu phải được ghi vào tài liệu cài đặt hoặc phiếu review đang được người dùng chủ động quản lý, để dữ liệu cũ không bị hiểu nhầm là khuyến nghị hiện hành.

Chỉ lưu screenshot/HTML khi support matrix khó truy cập hoặc nội dung thay đổi động.

---

## 7. Những gì cố ý bỏ khỏi quy trình cũ

- Không duyệt và xếp hạng mọi Kubernetes minor maintained.
- Không tạo `SURVIVE`, `RESERVE`, `ELIMINATED` hoặc semantic state machine.
- Không bắt mọi addon dùng chung một cột.
- Không tạo file evidence cho mọi lệnh đã chạy.
- Không phân tích toàn bộ Helm template, `lookup`, capability site hoặc tạo SHA256 fingerprint.
- Không tính lifecycle limit của mọi artifact bằng script.
- Không biến thiếu compatibility matrix thành lỗi.
- Không phủ nhận bằng chứng runtime của một stack đang hoạt động.

Các kiểm tra sâu chỉ dùng khi render/install thực sự lỗi, khi chuẩn bị production có SLA, hoặc khi audit bảo mật yêu cầu khả năng tái lập tuyệt đối.
