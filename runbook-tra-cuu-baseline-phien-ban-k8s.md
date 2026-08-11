# Runbook tra cứu tài liệu và chốt baseline phiên bản Kubernetes/Rancher

> **Mục tiêu:** nghiên cứu và phát hành một bảng phiên bản cho cụm Kubernetes dựng bằng kubeadm, CNI Flannel, ingress Traefik, TLS cert-manager, quản lý bởi Rancher — bằng cách **đọc tài liệu official, tra cứu và ghi chép**, rồi dùng **script ngắn** để kiểm chứng tính phù hợp, tương thích và khả thi.
>
> **Phạm vi:** runbook này độc lập và tự đủ. Phương pháp là hướng-tài-liệu: người làm tự tay tra nguồn official, hiểu **vì sao** chọn từng phiên bản và ghi chép vào phiếu; không yêu cầu evidence pipeline, checksum hay gate tự động nào — công cụ duy nhất là các script kiểm nhanh có sẵn trong từng bước.
>
> **Nguyên tắc:** tài liệu official là nguồn quyết định; script chỉ xác nhận lại điều bạn đã đọc, không thay bạn đọc. Bảng phiên bản là kết quả của một lần nghiên cứu **có ngày hết hạn**, không phải danh sách chép lại vĩnh viễn.

---

## Cách dùng runbook này

- Làm **tuần tự từ Bước 0 đến Bước 12**. Mỗi bước có bốn phần lặp lại:
  - **Đọc** — mở nguồn official nào, nhìn vào bảng/mục nào, đọc cột nào.
  - **Ghi** — chép giá trị nào vào phiếu ghi (tạo ở Bước 0).
  - **Kiểm nhanh** — một script ngắn xác nhận giá trị vừa ghi là có thật và đúng dạng.
  - **Đi tiếp khi** — điều kiện tối thiểu để sang bước sau.
- Hai hướng cam kết chạy song song trong cùng tài liệu: **VENDOR-SUPPORTED** và **TECHNICALLY-COMPATIBLE**. Bạn chọn một hướng ở Bước 1; các đoạn đánh dấu **[Theo hướng]** chỉ ra chỗ hai hướng đọc nguồn khác nhau. Các bước không có đánh dấu thì hai hướng làm giống hệt nhau.
- Script chạy trên máy Linux/WSL bất kỳ có internet và các công cụ ở Bước 0. Riêng Bước 4 và Bước 5 (tra gói `.deb`) cần chạy trên **Ubuntu cùng bản và cùng kiến trúc với node mục tiêu**.
- Khi một bước kết luận "tắc" (ví dụ hướng VENDOR-SUPPORTED không có tổ hợp hợp lệ), **dừng và ghi lý do vào phiếu** thay vì đổi số liệu cho qua.

---

## Bước 0 — Chuẩn bị

### 0.1. Công cụ

Cần: một **browser** (nhiều trang tra cứu render bằng JavaScript, `curl` không đọc được), và một shell có `curl`, `jq`, `helm`, `python3`. Kiểm:

```bash
for c in curl jq helm python3; do
  command -v "$c" >/dev/null || echo "THIẾU: $c"
done
echo "Kiểm công cụ xong — không có dòng THIẾU nào là đạt."
```

**Đi tiếp khi:** không in ra dòng `THIẾU` nào.

### 0.2. Tạo phiếu ghi

Tạo một file mới `phieu-ghi-baseline-<YYYY-MM-DD>.md` (ngày là ngày bắt đầu tra cứu) và dán nguyên template sau. Mọi ô `___` sẽ được điền dần khi đi qua các bước; ô nào không áp dụng thì ghi `N/A`, **không để trống**:

```markdown
# Phiếu ghi baseline — ngày tra cứu: ___

## A. Đầu vào
- Ngày cài đặt dự kiến: ___
- Hướng cam kết (VENDOR-SUPPORTED / TECHNICALLY-COMPATIBLE): ___
- OS node: Ubuntu ___ | Kiến trúc (amd64/arm64): ___
- Topology: kubeadm + Flannel + Traefik (Kubernetes Ingress, không Gateway API) + cert-manager + Rancher (cài lên chính cluster này)
- Pod CIDR: ___ | Service CIDR: ___ | LAN CIDR: ___
- Addon tuỳ chọn (local-path-provisioner / MetalLB / cloudflared): ___

## B. Kết quả tra cứu từng thành phần
| # | Thành phần | Loại version | Giá trị chốt | Dải K8s liên quan | EOL / hết support | Nguồn (URL) | Ngày tra |
| - | --- | --- | --- | --- | --- | --- | --- |
| 1 | Rancher (chart + app) | chart = app | ___ | ___ | ___ | ___ | ___ |
| 2 | Kubernetes | exact patch | ___ | N/A | ___ | ___ | ___ |
| 3 | kubeadm/kubelet/kubectl | Debian package | ___ | = K8s patch | theo K8s | ___ | ___ |
| 4 | Ubuntu | OS release | ___ | N/A | ___ | ___ | ___ |
| 5 | containerd | Debian package + upstream | ___ | ___ | ___ | ___ | ___ |
| 6 | Flannel | release tag | ___ | không công bố | ___ | ___ | ___ |
| 7 | Helm | release tag | ___ | ___ | N/A | ___ | ___ |
| 8 | cert-manager | release tag | ___ | ___ | ___ | ___ | ___ |
| 9 | Traefik | chart version / appVersion | ___ | ___ | ___ | ___ | ___ |
| 10 | local-path-provisioner | release tag | ___ | không công bố | N/A | ___ | ___ |
| 11 | MetalLB | release tag | ___ | ___ | N/A | ___ | ___ |
| 12 | cloudflared | release tag | ___ | N/A | N/A | ___ | ___ |

## C. Giao tương thích Kubernetes (điền ở Bước 3)
| K8s minor | EOL upstream | Rancher (theo hướng đã chọn) | cert-manager | Helm | containerd | Kết luận (CHỌN/LOẠI + lý do) |
| --- | --- | --- | --- | --- | --- | --- |
| ___ | ___ | ___ | ___ | ___ | ___ | ___ |
| ___ | ___ | ___ | ___ | ___ | ___ | ___ |
| ___ | ___ | ___ | ___ | ___ | ___ | ___ |

## D. Kết quả kiểm nhanh
| Bước | Nội dung kiểm | Kết quả (OK/FAIL) | Ghi chú |
| --- | --- | --- | --- |

## E. Bảng phiên bản phát hành (điền ở Bước 12)
| Thành phần | Phiên bản | Nhãn cam kết | Ghi chú |
| --- | --- | --- | --- |

- Ngày phải review lại baseline: ___
- Quyết định/tắc nghẽn đáng nhớ trong lần tra này: ___
```

**Đi tiếp khi:** phiếu đã tạo, phần A điền xong các giá trị đã biết (ngày cài, OS, CIDR, addon); hướng cam kết điền sau Bước 1.

---

## Bước 1 — Khái niệm và chọn hướng cam kết

### 1.1. Các khái niệm không được đánh đồng

Khi đọc tài liệu ở các bước sau, các từ dưới đây có nghĩa khác nhau — nhầm một từ là chọn sai phiên bản:

| Khái niệm | Nghĩa dùng trong runbook |
| --- | --- |
| Latest release | Bản mới nhất upstream đã phát hành; có thể là RC/beta hoặc chưa được thành phần khác hỗ trợ. |
| Stable channel | Channel vendor chỉ định cho production (ví dụ `rancher-stable`); vẫn phải đối chiếu support matrix. |
| Maintained | Nhánh còn nhận bản vá theo lifecycle upstream. |
| Supported | Vendor tuyên bố hỗ trợ **đúng tổ hợp** sản phẩm + phiên bản + nền tảng + vai trò. |
| Tested | Upstream có chạy test trên version đó; có thể hẹp hơn Supported. |
| Metadata-compatible | Chart/package **cho phép** version đó (ví dụ `kubeVersion` trong `Chart.yaml`); chưa phải chứng nhận vendor. |

Và các **loại version** không được gộp vào một cột:

| Loại | Ví dụ | Cách ghi vào phiếu |
| --- | --- | --- |
| Product/app version | Rancher `2.x.y`, Traefik Proxy `3.x.y` | SemVer chính xác |
| Helm chart version | Traefik chart `4x.y.z` | chart version chính xác (khác app version) |
| Debian package version | kubeadm `1.xx.y-1.1` | nguyên chuỗi package |
| Kubernetes minor / patch | `1.xx` / `1.xx.y` | minor để tra support window, patch để cài |
| Image tag | `repo/name:vX.Y.Z` | tag chính xác, không `latest`/`stable` |

**Thứ tự tin cậy khi nguồn mâu thuẫn:** (1) support matrix của vendor đúng version → (2) lifecycle/release notes official → (3) metadata của đúng artifact (`Chart.yaml`, package index) → (4) `values.yaml`/template của đúng chart version → (5) kết quả render thử. Blog, diễn đàn, câu trả lời cộng đồng chỉ dùng để tìm hướng, không dùng làm căn cứ chốt.

### 1.2. Hai hướng cam kết

Hình dung bằng chuyện độ xe: dùng phụ tùng chính hãng đúng chỉ định thì **còn bảo hành** (hỏng, hãng chịu); lắp đồ ngoài đúng chuẩn kỹ thuật thì **xe vẫn chạy nhưng tự chịu trách nhiệm**. Hai hướng của runbook đúng như vậy:

- **VENDOR-SUPPORTED** — chỉ chấp nhận tổ hợp mà SUSE **ghi rõ trong support matrix** là được chứng nhận. Được quyền mở ticket khi sự cố. Giá phải trả: tập lựa chọn hẹp. **Nói trước:** SUSE chỉ chứng nhận Rancher Manager chạy trên RKE2/K3s/managed Kubernetes — topology "Rancher cài lên cluster kubeadm" của runbook này **gần như chắc chắn tắc** ở Bước 2.4; khi đó phải đổi topology (ví dụ sang RKE2) hoặc chuyển hướng.
- **TECHNICALLY-COMPATIBLE** — chấp nhận tổ hợp vendor không chứng nhận, miễn tự chứng minh được về kỹ thuật: metadata cho phép, render thử thành công. Phù hợp homelab/staging tự vận hành. **Luật cứng:** mọi verify đều đạt thì nhãn cuối vẫn là `TECHNICALLY-COMPATIBLE` — test của bạn không thay được tuyên bố của hãng, cấm tự nâng nhãn.

Chọn hướng theo câu hỏi: *"khi hệ thống hỏng, bạn định dựa vào ai?"* — có hợp đồng support với SUSE → VENDOR-SUPPORTED; tự vận hành → TECHNICALLY-COMPATIBLE.

**Ghi:** điền hướng đã chọn vào phần A của phiếu.

### 1.3. Kiểm nhanh — ba CIDR không chồng lấn

Thay ba giá trị bằng CIDR thật ở phần A rồi chạy:

```bash
python3 - <<'PY'
import ipaddress
nets = {
    'POD':     '10.244.0.0/16',
    'SERVICE': '10.96.0.0/12',
    'LAN':     '192.168.100.0/24',
}
ns = {k: ipaddress.ip_network(v, strict=False) for k, v in nets.items()}
for a, b in [('POD', 'SERVICE'), ('POD', 'LAN'), ('SERVICE', 'LAN')]:
    print(a, 'vs', b, '->', 'OVERLAP' if ns[a].overlaps(ns[b]) else 'OK')
PY
```

**Đi tiếp khi:** ba dòng đều `OK` (ghi kết quả vào phần D). Nếu có `OVERLAP`, sửa quy hoạch mạng trước — không có phiên bản nào cứu được CIDR chồng lấn.

---

## Bước 2 — Rancher (điểm neo của mọi phiên bản khác)

Rancher đi trước vì cửa sổ Kubernetes mà Rancher hỗ trợ **hẹp hơn** cửa sổ upstream — chọn K8s trước rồi mới soi Rancher là dễ phải làm lại.

### 2.1. Đọc — channel nào để lấy version

Mở [Choosing a Rancher Version](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/choose-a-rancher-version). Đọc mục về Helm chart repositories: `rancher-latest` dành cho thử nghiệm, `rancher-stable` dành cho production. Runbook này dùng **`rancher-stable`**.

### 2.2. Tra — phiên bản stable mới nhất

Mở [Rancher releases trên GitHub](https://github.com/rancher/rancher/releases). Cách đọc: các release có hậu tố `-rc`, `-alpha`, `-beta` hoặc gắn nhãn *Pre-release* đều bỏ qua; tìm bản **final mới nhất có mặt trong stable channel**. Vì trang GitHub không nói bản nào đã vào stable channel, xác nhận bằng chính index của channel:

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable --force-update
helm search repo rancher-stable/rancher --versions | head -n 5
```

Dòng đầu tiên (không có hậu tố `-rc`...) là candidate. Ví dụ output (số minh họa):

```text
NAME                    CHART VERSION  APP VERSION  DESCRIPTION
rancher-stable/rancher  2.14.3         v2.14.3      Install Rancher Server...
```

**Ghi:** dòng 1 phần B — giá trị chốt (ví dụ `2.14.3`; với chart Rancher, chart version = app version), nguồn, ngày tra.

### 2.3. Đọc — support matrix của đúng phiên bản đó

Mở [SUSE Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/) **bằng browser** (trang render bằng JavaScript — `curl` chỉ lấy được vỏ HTML rỗng, đừng mất công). Chọn đúng phiên bản Rancher vừa chốt, rồi đọc **riêng từng bảng** — chúng trả lời ba câu hỏi khác nhau:

1. **Supported Kubernetes Platforms for Rancher Manager** — được cài Rancher Server lên nền tảng nào. Đây là bảng quyết định cho topology của runbook này (Rancher cài lên chính cluster).
2. **Downstream Cluster Support** — Rancher tạo/quản lý được cluster nào.
3. **All Other Distros / Imported** — cluster có sẵn nào import vào được. Dòng "Any" ở bảng này **không** biến kubeadm thành nền tảng được chứng nhận làm Manager host — đừng dùng nhầm bảng.

Cũng trong matrix, đọc bảng **OS được chứng nhận** và ghi lại Ubuntu bản nào được liệt kê.

**Ghi:** vào cột "Dải K8s liên quan" của dòng 1 phần B — các phiên bản/nền tảng K8s mà bảng (1) liệt kê; kèm ghi chú OS được chứng nhận. Nên chụp màn hình bảng và lưu cạnh phiếu (trang này đổi nội dung theo thời gian).

### 2.4. Điểm rẽ theo hướng

**[Theo hướng VENDOR-SUPPORTED]** — nếu bảng (1) không liệt kê kubeadm (thực tế thường vậy): topology hiện tại **không đạt** hướng này. Dừng tại đây và chọn một trong hai: (a) đổi topology sang nền tảng có trong bảng (RKE2/K3s) — khi đó dải K8s chính là các phiên bản RKE2/K3s được liệt kê; hoặc (b) chuyển hướng sang TECHNICALLY-COMPATIBLE và ghi quyết định này vào cuối phiếu. Không được lấy dòng "Any" của bảng Imported làm bằng chứng đi tiếp.

**[Theo hướng TECHNICALLY-COMPATIBLE]** — hợp đồng kỹ thuật là `kubeVersion` trong `Chart.yaml` của **đúng chart version**. Tra bằng:

```bash
helm show chart rancher-stable/rancher --version 2.14.3 | grep -E '^(version|appVersion|kubeVersion):'
```

(thay `2.14.3` bằng giá trị đã chốt). Ví dụ output (số minh họa):

```text
appVersion: v2.14.3
kubeVersion: '>=1.32.0-0 < 1.36.0-0'
version: 2.14.3
```

**Ghi:** dải `kubeVersion` vào cột "Dải K8s liên quan" dòng 1 phần B, kết quả lệnh vào phần D. Support matrix vẫn đáng đọc để biết SUSE test Rancher này với K8s nào — nhưng ở hướng này nó là tham khảo, không phải điều kiện chặn.

**Đi tiếp khi:** dòng 1 phần B đã đủ giá trị + dải K8s + nguồn + ngày tra.

---

## Bước 3 — Kubernetes: chọn minor và exact patch

### 3.1. Đọc — minor nào còn maintained, EOL khi nào

Mở [Kubernetes Releases](https://kubernetes.io/releases/). Trang liệt kê các minor đang được duy trì kèm patch mới nhất và ngày **End of Life**. Lưu ý: bình thường có 3 minor, nhưng trong giai đoạn chuyển tiếp (minor mới vừa ra, minor cũ chưa EOL) có thể có 4 — **ghi đúng những gì trang hiển thị**, đừng ép về 3.

**Ghi:** mỗi minor một dòng vào bảng C của phiếu — cột "K8s minor" và "EOL upstream". Patch mới nhất của từng minor ghi tạm vào cột Kết luận hoặc ghi chú.

### 3.2. Lập giao và chọn

Điền tiếp các cột còn lại của bảng C cho từng minor:

- **Rancher:** minor có nằm trong dải đã ghi ở Bước 2.4 không? (`PASS` / `FAIL`).
- **cert-manager, Helm, containerd:** tạm để trống — sẽ điền khi đọc tài liệu ở Bước 5, 7, 8; quay lại bảng C sau mỗi bước đó. (Nếu muốn điền một lượt ngay bây giờ, đọc trước ba nguồn ở các bước ấy rồi quay lại.)

Quy tắc kết luận từng dòng: `CHỌN` chỉ khi mọi cột đều đạt; nhiều dòng cùng đạt thì ưu tiên **EOL xa nhất trong số các minor đã được Rancher hỗ trợ** (minor mới nhất chưa chắc được Rancher hỗ trợ). Ghi lý do LOẠI cho từng dòng bị loại — người đọc sau cần biết vì sao, không chỉ biết kết quả.

Chốt: minor được chọn + **patch mới nhất của minor đó** (đọc từ trang Releases ở 3.1).

### 3.3. Kiểm nhanh — patch là release final có thật

```bash
K8S_VERSION=1.35.7   # thay bằng patch đã chốt, không có tiền tố v
curl -fsSL "https://api.github.com/repos/kubernetes/kubernetes/releases/tags/v${K8S_VERSION}" \
  | jq '{tag: .tag_name, prerelease, draft}'
```

Đạt khi in đúng tag với `"prerelease": false` và `"draft": false`; lỗi 404 nghĩa là gõ sai patch.

### 3.4. Đọc — version skew

Mở [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) để hiểu ràng buộc giữa kubelet/kube-apiserver/kubectl. Với cluster **dựng mới**, quy ước của baseline này đơn giản hơn policy: **kubeadm, kubelet, kubectl cùng một exact version** — không chủ động dùng skew.

**Ghi:** dòng 2 phần B (exact patch, EOL của minor, nguồn, ngày tra); kết quả 3.3 vào phần D.

**Đi tiếp khi:** bảng C có đúng một dòng `CHỌN` (ít nhất đã đạt cột Rancher; các cột còn lại xác nhận dần ở Bước 5–8) và dòng 2 phần B đã điền.

---

## Bước 4 — Gói kubeadm/kubelet/kubectl và OS Ubuntu

### 4.1. Đọc — Ubuntu còn được bảo trì đến khi nào

Mở [Ubuntu release cycle](https://ubuntu.com/about/release-cycle). Tìm bản LTS đang dùng (ví dụ 24.04), đọc ngày kết thúc **standard support**. Ngày này phải **sau** ngày cài dự kiến.

**[Theo hướng VENDOR-SUPPORTED]** — đối chiếu thêm: bản Ubuntu này có trong bảng OS của support matrix (đã đọc ở Bước 2.3) không. Không có → đổi OS hoặc ghi nhận không đạt hướng.

**Ghi:** dòng 4 phần B.

### 4.2. Đọc — repo gói theo minor

Mở [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), mục dành cho Debian-based. Điểm cần hiểu: `pkgs.k8s.io` tách **repo riêng cho từng minor** (`.../core:/stable:/v1.35/deb/`) — đổi minor là phải đổi dòng repo, và repo minor này không chứa gói minor khác. Làm theo đúng trang đó để khai repo của minor đã chọn trên máy Ubuntu cùng bản với node.

### 4.3. Kiểm nhanh — repo có đúng gói của patch đã chọn

Chạy trên máy Ubuntu vừa khai repo:

```bash
sudo apt-get update
for p in kubeadm kubelet kubectl; do
  apt-cache madison "$p" | head -n 3
done
```

Cách đọc: cột thứ hai là Debian package version, dạng `1.35.7-1.1`. Đạt khi **cả ba gói** đều có dòng bắt đầu bằng đúng patch đã chốt, và bạn ghi lại **nguyên chuỗi** đó (ví dụ `1.35.7-1.1`) — đây là giá trị pin khi cài, không phải `1.35.7`.

**Ghi:** dòng 3 phần B (chuỗi package đầy đủ); kết quả vào phần D.

**Đi tiếp khi:** ba gói cùng version trong repo đúng minor, và Ubuntu còn standard support qua ngày cài.

---

## Bước 5 — containerd

### 5.1. Đọc — yêu cầu từ phía Kubernetes

Mở [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/). Ba điều cần rút ra: runtime phải nói chuyện qua **CRI**; cgroup driver nên là **systemd** (và phải khớp giữa kubelet với runtime); trang có mục riêng cho containerd với các bước cấu hình chuẩn.

### 5.2. Đọc — nhánh containerd nào còn được hỗ trợ

Mở [containerd RELEASES.md](https://github.com/containerd/containerd/blob/main/RELEASES.md). Đọc hai bảng: bảng **branch status** (nhánh nào Active/LTS, end-of-support khi nào) và bảng **Kubernetes support** (nhánh containerd nào được khuyến nghị với K8s minor nào). Minor K8s đã chọn phải nằm trong hàng của nhánh còn được hỗ trợ — điền kết quả này vào cột `containerd` của bảng C.

### 5.3. Tra + kiểm nhanh — Ubuntu repo có gì

Chạy trên máy Ubuntu cùng bản với node:

```bash
apt-cache policy containerd
```

Cách đọc: dòng `Candidate:` là Debian package version sẽ được cài (ví dụ `2.0.5-1ubuntu1` — số minh họa). Phần đầu chuỗi cho biết upstream version → đối chiếu ngược lại RELEASES.md xem nhánh đó (ví dụ 2.0) còn được hỗ trợ và có trong bảng Kubernetes support không. Lưu ý cấu hình mặc định: containerd 1.x và 2.x khác **thế hệ file config** (`version = 2` / `version = 3`) — ghi chú lại thế hệ config để runbook cài đặt dùng đúng mẫu.

**Ghi:** dòng 5 phần B (cả chuỗi Debian package lẫn upstream version, nhánh, ngày end-of-support); cột `containerd` bảng C; kết quả lệnh vào phần D.

**Đi tiếp khi:** nhánh containerd còn hỗ trợ, có trong bảng Kubernetes support cho minor đã chọn.

---

## Bước 6 — CNI Flannel

### 6.1. Đọc — release mới nhất và những gì upstream (không) hứa

Mở [Flannel releases](https://github.com/flannel-io/flannel/releases). Lấy release **final** mới nhất (bỏ pre-release). Đọc release notes của bản đó; lưu ý trung thực: Flannel **không công bố** ma trận tương thích Kubernetes chính thức — vì vậy trong phiếu, dải K8s của Flannel ghi `không công bố`, và độ tin cậy dựa vào render/dựng thử (Bước 11) chứ không dựa vào tuyên bố vendor.

Tham chiếu nền: [Kubernetes Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) — cluster cần CNI plugin tương thích spec từ `v0.4.0` trở lên.

### 6.2. Kiểm nhanh — manifest của đúng tag

```bash
FLANNEL_VERSION=v0.27.4   # thay bằng tag đã chốt
curl -fsSL -o kube-flannel.yml \
  "https://github.com/flannel-io/flannel/releases/download/${FLANNEL_VERSION}/kube-flannel.yml"
grep -E '^[[:space:]]+image:|"Network"|"cniVersion"' kube-flannel.yml
```

Cách đọc output:

- Các dòng `image:` phải pin tag cụ thể (ví dụ `flannel/flannel:v0.27.4`), **không** có `latest`/`master`.
- Dòng `"Network"` là Pod CIDR mặc định của manifest (thường `10.244.0.0/16`). Nếu khác Pod CIDR ở phần A của phiếu, ghi chú rõ: *khi cài phải sửa trường Network* — việc sửa thuộc runbook cài đặt, không sửa ở đây.
- Dòng `"cniVersion"` phải từ `0.4` trở lên hoặc `1.x`.

**Ghi:** dòng 6 phần B; kết quả ba ý trên vào phần D.

**Đi tiếp khi:** tag final tải được manifest, image pin cụ thể, cniVersion đạt.

---

## Bước 7 — Helm

### 7.1. Đọc

Hai nguồn: [Helm Version Support Policy](https://helm.sh/docs/v3/topics/version_skew/) — bảng Helm minor ↔ dải Kubernetes được hỗ trợ; và [Rancher Helm Version Requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements) — phiên bản Helm tối thiểu mà Rancher yêu cầu.

Cách chọn: lấy Helm minor mới nhất mà bảng skew ghi hỗ trợ minor K8s đã chọn, và không thấp hơn yêu cầu của Rancher. Điền cột `Helm` của bảng C.

### 7.2. Kiểm nhanh — Helm đang cài đúng bản định dùng

```bash
helm version --short
```

Đạt khi version in ra đúng bản đã chốt (nếu chưa cài đúng bản, cài lại theo trang official trước khi làm Bước 11 — mọi lần render phải chạy bằng đúng Helm của baseline).

**Ghi:** dòng 7 phần B; cột `Helm` bảng C; kết quả vào phần D.

---

## Bước 8 — cert-manager

### 8.1. Đọc

Mở [cert-manager Supported Releases](https://cert-manager.io/docs/releases/). Trang có bảng: minor nào đang supported, EOL khi nào, và **hai dải Kubernetes riêng biệt** — *Supported* và *Tested*. Chọn minor đang supported có dải Supported chứa minor K8s đã chọn; trong minor đó lấy **patch mới nhất**. Nếu K8s của bạn nằm trong Supported nhưng ngoài Tested, vẫn dùng được nhưng ghi chú rõ điều đó vào phiếu — đừng tự ghi "tested".

Đừng lấy số version từ command ví dụ trong [trang cài đặt Helm](https://cert-manager.io/docs/installation/helm/) làm "mới nhất" — trang ví dụ có thể cũ hơn trang releases.

**Ghi:** dòng 8 phần B (version, dải Supported/Tested, EOL); cột `cert-manager` bảng C.

### 8.2. Kiểm nhanh — release và chart có thật

```bash
CERT_MANAGER_VERSION=v1.19.1   # thay bằng version đã chốt
curl -fsSL "https://api.github.com/repos/cert-manager/cert-manager/releases/tags/${CERT_MANAGER_VERSION}" \
  | jq '{tag: .tag_name, prerelease, draft}'
helm show chart "oci://quay.io/jetstack/charts/cert-manager" \
  --version "${CERT_MANAGER_VERSION}" | grep -E '^(version|appVersion):'
```

Đạt khi tag là final (`prerelease/draft: false`) và chart OCI đọc được metadata đúng version.

**Ghi:** kết quả vào phần D.

---

## Bước 9 — Traefik

### 9.1. Đọc

Ba nguồn:

- [Traefik Helm chart releases](https://github.com/traefik/traefik-helm-chart/releases) — chọn **chart version** final mới nhất; đọc release notes bản đó (chart version dạng `4x.y.z` **khác** app version `3.x.y` của Traefik Proxy — phiếu ghi cả hai).
- [Traefik Kubernetes docs](https://doc.traefik.io/traefik/setup/kubernetes/) — xác nhận cách cài bằng Helm chart và các provider (runbook này dùng provider **Kubernetes Ingress**, không bật Gateway API).
- Metadata của đúng chart version (lệnh dưới) — `kubeVersion` là dải K8s mà chart cho phép.

### 9.2. Kiểm nhanh — metadata của đúng chart version

```bash
TRAEFIK_CHART_VERSION=37.1.2   # thay bằng chart version đã chốt
helm repo add traefik https://traefik.github.io/charts --force-update
helm show chart traefik/traefik --version "${TRAEFIK_CHART_VERSION}" \
  | grep -E '^(version|appVersion|kubeVersion):'
```

Đối chiếu bằng mắt: `kubeVersion` phải chứa minor K8s đã chọn (Bước 11 sẽ để Helm tự kiểm điều này một lần nữa khi render).

**Ghi:** dòng 9 phần B (chart version, appVersion, kubeVersion); kết quả vào phần D.

---

## Bước 10 — Addon tuỳ chọn

Chỉ làm với addon đã khai ở phần A; addon không dùng thì ghi `N/A` vào dòng tương ứng phần B.

| Addon | Đọc ở đâu | Ghi gì |
| --- | --- | --- |
| local-path-provisioner | [Releases](https://github.com/rancher/local-path-provisioner/releases) | tag final mới nhất; upstream không công bố dải K8s |
| MetalLB | [Releases](https://github.com/metallb/metallb/releases) và [metallb.io](https://metallb.io/) (mục requirements — có ghi yêu cầu phiên bản Kubernetes) | tag final + yêu cầu K8s nếu có |
| cloudflared | [Cloudflare Tunnel downloads](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/) và [Releases](https://github.com/cloudflare/cloudflared/releases) | tag final mới nhất |

Kiểm nhanh (chạy cho từng addon đã chọn, thay repo/tag tương ứng):

```bash
curl -fsSL "https://api.github.com/repos/rancher/local-path-provisioner/releases/tags/v0.0.34" \
  | jq '{tag: .tag_name, prerelease, draft}'
```

**Ghi:** dòng 10–12 phần B; kết quả vào phần D.

---

## Bước 11 — Kiểm khả thi: render thử ba chart bằng đúng phiên bản đã chốt

Đây là bước "khả thi để dựng được": render ba chart bằng **đúng Helm, đúng chart version, đúng `--kube-version`** đã chốt. Hai điều được kiểm cùng lúc:

1. `helm template --kube-version` **tự đối chiếu** với `kubeVersion` trong `Chart.yaml` — nếu K8s đã chọn nằm ngoài dải chart cho phép, lệnh fail ngay. Đây là verify tự động cho các dải đã đọc ở Bước 2.4 và 9.2.
2. Manifest render ra đúng kiến trúc đã khai (Ingress class `traefik`, không lạc sang nhánh Gateway API, không image tag động).

Render thành công chỉ chứng minh **metadata + template hợp lệ** — chưa chứng minh controller chạy đúng lúc runtime; điều đó thuộc bước dựng thử của runbook cài đặt.

### 11.1. Tạo ba file values tối thiểu

```bash
mkdir -p render-check && cd render-check

cat > cert-manager-values.yaml <<'EOF'
crds:
  enabled: true
EOF

cat > traefik-values.yaml <<'EOF'
providers:
  kubernetesIngress:
    enabled: true
  kubernetesGateway:
    enabled: false
ingressClass:
  enabled: true
EOF

cat > rancher-values.yaml <<'EOF'
hostname: rancher.example.com
networkExposure:
  type: ingress
ingress:
  enabled: true
  ingressClassName: traefik
  tls:
    source: rancher
    secretName: tls-rancher-ingress
EOF
```

Vì sao phải pin mấy key này dù default hiện tại có thể đã đúng: chart Rancher có nhánh Gateway API song song nhánh Ingress, và nhánh Ingress chỉ bật khi `networkExposure.type=ingress` **và** `ingress.enabled=true` cùng lúc. Default có thể đổi giữa các bản chart — pin rõ để render hôm nay và render sau nâng cấp cho cùng kết quả.

### 11.2. Render và kiểm

Thay bốn biến bằng giá trị đã chốt trong phiếu:

```bash
K8S_VERSION=1.35.7
CERT_MANAGER_VERSION=v1.19.1
TRAEFIK_CHART_VERSION=37.1.2
RANCHER_VERSION=2.14.3

helm template cert-manager "oci://quay.io/jetstack/charts/cert-manager" \
  --namespace cert-manager --version "${CERT_MANAGER_VERSION}" \
  --kube-version "${K8S_VERSION}" -f cert-manager-values.yaml \
  > cert-manager-rendered.yaml && echo "OK: cert-manager render"

helm template traefik traefik/traefik \
  --namespace traefik --version "${TRAEFIK_CHART_VERSION}" \
  --kube-version "${K8S_VERSION}" -f traefik-values.yaml \
  > traefik-rendered.yaml && echo "OK: traefik render"

helm template rancher rancher-stable/rancher \
  --namespace cattle-system --version "${RANCHER_VERSION}" \
  --kube-version "${K8S_VERSION}" -f rancher-values.yaml \
  > rancher-rendered.yaml && echo "OK: rancher render"
```

Nếu một lệnh fail với thông báo dạng `chart requires kubeVersion ... which is incompatible with Kubernetes v...` — đó chính là mâu thuẫn giữa K8s đã chọn và dải chart cho phép: quay lại bảng C, xem lại bước tương ứng, **không** ép qua bằng cách bỏ `--kube-version`.

Kiểm nội dung render:

```bash
grep -c '^kind: Ingress$' rancher-rendered.yaml
grep 'ingressClassName' rancher-rendered.yaml
grep -E '^kind: (Gateway|GatewayClass|HTTPRoute)$' *-rendered.yaml || echo "OK: không có Gateway API"
grep -E 'image:.*:(latest|stable)["'\'']?$' *-rendered.yaml || echo "OK: không có image tag động"
grep -c '^kind: CustomResourceDefinition$' cert-manager-rendered.yaml
```

Cách đọc từng dòng:

| Lệnh | Đạt khi |
| --- | --- |
| đếm `kind: Ingress` trong rancher | ≥ 1 — chart đi đúng nhánh Ingress |
| `ingressClassName` | in ra `ingressClassName: traefik` |
| grep Gateway/HTTPRoute | in `OK: không có Gateway API` |
| grep tag động | in `OK: không có image tag động` |
| đếm CRD của cert-manager | ≥ 6 — CRDs được render kèm |

**Ghi:** cả năm kết quả vào phần D.

**Đi tiếp khi:** ba lệnh render đều `OK` và năm kiểm nội dung đều đạt.

---

## Bước 12 — Hoàn thiện phiếu, dán nhãn và tính ngày review

### 12.1. Dán nhãn cam kết cho từng thành phần

Điền bảng E từ bảng B, với quy tắc nhãn:

- `VENDOR-SUPPORTED` — **chỉ khi** đi hướng VENDOR-SUPPORTED và support matrix/tài liệu vendor chứng nhận đúng tổ hợp + vai trò đang dùng.
- `TECHNICALLY-COMPATIBLE` — metadata cho phép + render thử đạt, nhưng vendor không chứng nhận topology. Toàn bộ baseline "Rancher trên kubeadm" nhận nhãn này là **trần** — không thành phần nào được ghi cao hơn.
- Thành phần upstream không công bố dải tương thích (Flannel, local-path-provisioner, cloudflared): ghi chú thêm `upstream không công bố dải K8s` trong cột Ghi chú.

Không dùng `latest`, `stable`, `master`, version range hay ô trống trong bảng E — mọi ô là giá trị chính xác hoặc `N/A`.

### 12.2. Tính ngày review lại

Ngày review trả lời câu hỏi: *"đến ngày nào thì phiếu này không còn tin được nữa và phải tra lại?"*. Tính bằng ba bước:

1. **Tiên quyết:** EOL gần nhất (trong các thành phần có lifecycle) phải **sau ngày cài dự kiến**. Thành phần hết hạn trước khi cài thì quay lại bảng C loại nó ngay — không cần tính tiếp.
2. **Mốc lifecycle** = `EOL gần nhất − 90 ngày`.
3. **Mốc cài đặt** = `ngày cài dự kiến − 30 ngày`. Nếu mốc này **đã qua** (tức bạn đang tra cứu trong vòng 30 ngày trước ngày cài), lấy chính **ngày cài dự kiến** làm mốc. Lý do: buffer 30 ngày tồn tại để ép tra lại khi phiếu được lập quá sớm trước ngày cài; còn khi đang tra ngay trong cửa sổ 30 ngày đó thì chính lần tra này đã là lần tra "sát ngày cài", phiếu hợp lệ đến ngày cài.

`ngày review = min( mốc lifecycle , mốc cài đặt )`. Nếu kết quả **sớm hơn ngày tra cứu hôm nay** → baseline stale ngay khi lập → quay lại bảng C chọn minor có EOL xa hơn. Với cách tính ở bước 3, tình huống stale chỉ có thể do vế lifecycle gây ra — nên lời khuyên "đổi minor" luôn thực hiện được (vế cài đặt không phụ thuộc minor, đổi minor không cứu được nó).

Ví dụ tính tay (số minh họa), ngày cài dự kiến 2026-09-15:

- Tra cứu 2026-08-11, chọn K8s 1.34 (EOL 2026-10-28): tiên quyết đạt (EOL sau ngày cài). Mốc lifecycle = 2026-07-30; mốc cài đặt = 2026-08-16 (chưa qua, giữ nguyên); min = **2026-07-30** — sớm hơn hôm nay → 1.34 bị loại vì lifecycle, dù mọi cột tương thích đều đạt.
- Tra cứu 2026-08-11, chọn K8s 1.35 (EOL 2027-02-28): mốc lifecycle = 2026-11-30; mốc cài đặt = 2026-08-16; min = **2026-08-16** — hợp lệ, ghi vào phiếu.
- Tra cứu 2026-09-05 (sát ngày cài), chọn K8s 1.35 (EOL 2027-02-28): mốc lifecycle = 2026-11-30; `cài − 30 = 2026-08-16` **đã qua** → mốc cài đặt = ngày cài = 2026-09-15; min = **2026-09-15** — hợp lệ. Nếu cứ giữ 2026-08-16 làm mốc, phiếu bị coi là stale ngay khi lập mà không minor nào cứu được — đó chính là trường hợp bước 3 tồn tại để xử lý.

Sau ngày review, bảng phải được tra cứu lại toàn bộ (ít nhất từ Bước 2) — không chỉ sửa một ô patch.

### 12.3. Kiểm nhanh — phiếu không còn ô bỏ trống

```bash
grep -n '___' phieu-ghi-baseline-*.md && echo "FAIL: còn ô chưa điền" || echo "OK: phiếu đã điền đủ"
```

### 12.4. Phát hành

Bảng E + ngày review chính là nội dung đưa vào mục "Phiên bản" của runbook cài đặt. Kèm theo phiếu ghi đầy đủ (phần A–D) để người sau tái dựng được quyết định — một danh sách số phiên bản không kèm nguồn và ngày tra là thứ sẽ bị chép lại vĩnh viễn, đúng điều nguyên tắc đầu tài liệu cấm.

---

## Phụ lục A — Danh mục nguồn official theo thành phần

| Thành phần | Nguồn bắt buộc | Dùng ở bước |
| --- | --- | --- |
| Rancher — channel | [Choosing a Rancher Version](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/choose-a-rancher-version) | 2.1 |
| Rancher — compatibility | [SUSE Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/) (mở bằng browser) | 2.3 |
| Rancher — yêu cầu cài | [Installation Requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/installation-requirements) | 2, 4 |
| Kubernetes — lifecycle | [Releases](https://kubernetes.io/releases/) | 3.1 |
| Kubernetes — skew | [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) | 3.4 |
| kubeadm — gói | [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) | 4.2 |
| Runtime — yêu cầu | [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) | 5.1 |
| containerd — lifecycle | [RELEASES.md](https://github.com/containerd/containerd/blob/main/RELEASES.md) | 5.2 |
| CNI — yêu cầu | [Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) | 6.1 |
| Flannel | [Releases](https://github.com/flannel-io/flannel/releases) | 6 |
| Helm — skew | [Version Support Policy](https://helm.sh/docs/v3/topics/version_skew/) | 7.1 |
| Helm — theo Rancher | [Helm Version Requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements) | 7.1 |
| cert-manager | [Supported Releases](https://cert-manager.io/docs/releases/), [Helm install](https://cert-manager.io/docs/installation/helm/) | 8 |
| Traefik | [Chart releases](https://github.com/traefik/traefik-helm-chart/releases), [Kubernetes docs](https://doc.traefik.io/traefik/setup/kubernetes/) | 9 |
| local-path-provisioner | [Releases](https://github.com/rancher/local-path-provisioner/releases) | 10 |
| MetalLB | [Releases](https://github.com/metallb/metallb/releases), [metallb.io](https://metallb.io/) | 10 |
| cloudflared | [Downloads](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/), [Releases](https://github.com/cloudflare/cloudflared/releases) | 10 |

Trang "current" đổi nội dung theo thời gian — vì vậy phiếu luôn ghi **ngày tra** cạnh mỗi URL, và với trang render bằng JavaScript (SUSE matrix) nên lưu thêm screenshot.

---

## Phụ lục B — Ví dụ phiếu đã điền (toàn bộ số là minh họa, không chép lại)

Trích phần B và E của một phiếu giả định tra ngày 2026-08-11, để hình dung mức chi tiết cần đạt:

**Phần B (trích):**

| # | Thành phần | Loại version | Giá trị chốt | Dải K8s liên quan | EOL / hết support | Nguồn | Ngày tra |
| - | --- | --- | --- | --- | --- | --- | --- |
| 1 | Rancher | chart = app | 2.14.3 | kubeVersion `>=1.32 <1.36` (chart) | theo minor 2.14 | releases.rancher.com + Chart.yaml | 2026-08-11 |
| 2 | Kubernetes | exact patch | 1.35.7 | N/A | minor 1.35 EOL 2027-02-28 | kubernetes.io/releases | 2026-08-11 |
| 3 | kubeadm/kubelet/kubectl | Debian package | 1.35.7-1.1 | = K8s patch | theo K8s | pkgs.k8s.io (madison) | 2026-08-11 |
| 5 | containerd | deb + upstream | 2.0.5-1ubuntu1 (2.0.5, nhánh 2.0 Active) | có trong bảng K8s support | theo RELEASES.md | Ubuntu repo + RELEASES.md | 2026-08-11 |
| 8 | cert-manager | release tag | v1.19.1 | Supported 1.32–1.35; Tested 1.33–1.35 | minor 1.19 theo trang releases | cert-manager.io/docs/releases | 2026-08-11 |

**Phần E (trích):**

| Thành phần | Phiên bản | Nhãn cam kết | Ghi chú |
| --- | --- | --- | --- |
| Kubernetes | 1.35.7 | TECHNICALLY-COMPATIBLE | kubeadm/kubelet/kubectl pin `1.35.7-1.1`, cùng version |
| Rancher | 2.14.3 | TECHNICALLY-COMPATIBLE | manager trên kubeadm — matrix không chứng nhận; render đạt |
| Flannel | v0.27.4 | TECHNICALLY-COMPATIBLE | upstream không công bố dải K8s; Network sửa theo Pod CIDR khi cài |
| cert-manager | v1.19.1 | TECHNICALLY-COMPATIBLE | K8s 1.35 nằm trong cả Supported và Tested |

- Ngày phải review lại baseline: 2026-08-16 (= ngày cài 2026-09-15 − 30, sớm hơn EOL 1.35 − 90 = 2026-11-30)
- Quyết định đáng nhớ: loại K8s 1.34 vì ngày review tính ra 2026-07-30 — sớm hơn ngày tra cứu.
