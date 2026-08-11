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
- Cam kết đi hết Bước 0–12 áp dụng cho hướng **TECHNICALLY-COMPATIBLE**. Hướng **VENDOR-SUPPORTED** có thể kết thúc hợp lệ ở Bước 2.4 nếu topology kubeadm không nằm trong support matrix; đây là kết quả nghiên cứu, không phải lỗi thao tác.
- Mỗi lần mở một phiên làm việc mới, phải tạo lại biến ngày ở Bước 0.1 từ đồng hồ của máy đang chạy. Không chép ngày từ ví dụ, phiên trước hay nội dung cũ trong phiếu.

---

## Bước 0 — Chuẩn bị

### 0.1. Neo thời gian của phiên làm việc

Chạy đoạn này **ở đầu mỗi phiên làm việc mới** và chạy lại nếu phiên kéo dài qua nửa đêm:

```bash
unset BASELINE_TODAY BASELINE_SESSION_STARTED_AT BASELINE_TIMEZONE
export BASELINE_TODAY="$(date +%F)"
export BASELINE_SESSION_STARTED_AT="$(date --iso-8601=seconds)"
export BASELINE_TIMEZONE="$(date +%Z%z)"
printf 'Ngày tra hiện tại: %s\nBắt đầu phiên: %s\nMúi giờ: %s\n' \
  "$BASELINE_TODAY" "$BASELINE_SESSION_STARTED_AT" "$BASELINE_TIMEZONE"
```

`BASELINE_TODAY` là nguồn duy nhất cho khái niệm “hôm nay” trong runbook. Ngày cài dự kiến là đầu vào riêng của người làm, không được gán bằng ngày hiện tại.

Nếu tiếp tục một phiếu từ phiên trước:

- chạy lại đoạn trên;
- ghi phiên mới vào mục “Lịch sử phiên tra” của phiếu;
- mỗi dòng nguồn ghi ngày thực sự mở/kiểm nguồn trong phiên hiện tại;
- nếu quyết định version thay đổi, cập nhật dữ liệu và lý do, không chỉ sửa cột ngày.

**Đi tiếp khi:** giá trị in ra khớp ngày, giờ và múi giờ của máy đang dùng.

### 0.2. Công cụ

Cần: một **browser** (nhiều trang tra cứu render bằng JavaScript, `curl` không đọc được), và một shell có `curl`, `jq`, `helm`, `python3`. Kiểm:

```bash
for c in curl jq helm python3; do
  command -v "$c" >/dev/null || echo "THIẾU: $c"
done
echo "Kiểm công cụ xong — không có dòng THIẾU nào là đạt."
```

**Đi tiếp khi:** không in ra dòng `THIẾU` nào.

### 0.3. Tạo phiếu ghi

Tên phiếu của phiên đầu tiên được tạo từ ngày hiện tại, không gõ tay:

```bash
test -n "${BASELINE_TODAY:-}" || { echo 'FAIL: chưa chạy Bước 0.1'; exit 1; }
export BASELINE_SHEET="phieu-ghi-baseline-${BASELINE_TODAY}.md"
printf '%s\n' "$BASELINE_SHEET"
```

Tạo file có tên vừa in và dán template sau. Thay `<BASELINE_TODAY>` và `<BASELINE_SESSION_STARTED_AT>` bằng đúng giá trị Bước 0.1. Mọi ô `___` sẽ được điền dần; ô không áp dụng ghi `N/A`, **không để trống**:

```markdown
# Phiếu ghi baseline — ngày bắt đầu: <BASELINE_TODAY>

## Lịch sử phiên tra
| Bắt đầu phiên | Ngày hiện tại | Múi giờ | Phạm vi tra/cập nhật |
| --- | --- | --- | --- |
| <BASELINE_SESSION_STARTED_AT> | <BASELINE_TODAY> | ___ | Khởi tạo phiếu |

## A. Đầu vào
- Ngày cài đặt dự kiến: ___
- Hướng cam kết (VENDOR-SUPPORTED / TECHNICALLY-COMPATIBLE): ___
- OS node: Ubuntu ___ | Kiến trúc (amd64/arm64): ___
- Topology: kubeadm + Flannel + Traefik (Kubernetes Ingress, không Gateway API) + cert-manager + Rancher (cài lên chính cluster này)
- Pod CIDR: ___ | Service CIDR: ___ | LAN CIDR: ___
- Addon tuỳ chọn (local-path-provisioner / MetalLB / cloudflared): ___

## B. Kết quả tra cứu từng thành phần
| # | Thành phần | Loại version | Giá trị chốt | Dải K8s liên quan | Lifecycle (DATE/EVENT + giá trị) | Nguồn (URL) | Ngày tra |
| - | --- | --- | --- | --- | --- | --- | --- |
| 1 | Rancher (chart + app) | chart = app | ___ | ___ | ___ | ___ | ___ |
| 2 | Kubernetes | exact patch | ___ | N/A | ___ | ___ | ___ |
| 3 | kubeadm/kubelet/kubectl | Debian package | ___ | = K8s patch | theo K8s | ___ | ___ |
| 4 | Ubuntu | OS release | ___ | N/A | ___ | ___ | ___ |
| 5 | containerd | Debian package + upstream | ___ | ___ | package: ___; upstream: ___ | ___ | ___ |
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
- Các lifecycle EVENT và review proxy nội bộ: ___
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
helm search repo rancher-stable/rancher --versions --output json \
  | jq '.[0] | {name, version, app_version, description}'
```

Object đầu tiên trong mảng JSON là candidate. `helm search repo` mặc định không lấy pre-release; không thêm `--devel`. Dùng JSON để không nhầm dòng header của output dạng bảng là dữ liệu.

**Ghi:** dòng 1 phần B — chart version và app version chính xác, nguồn, `BASELINE_TODAY`. Với chart Rancher hai giá trị thường cùng số nhưng vẫn phải ghi đúng output, không tự suy ra.

### 2.3. Đọc — support matrix của đúng phiên bản đó

Mở [SUSE Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/) **bằng browser** (trang render bằng JavaScript — `curl` chỉ lấy được vỏ HTML rỗng, đừng mất công). Chọn đúng phiên bản Rancher vừa chốt, rồi đọc **riêng từng bảng** — chúng trả lời ba câu hỏi khác nhau:

1. **Supported Kubernetes Platforms for Rancher Manager** — được cài Rancher Server lên nền tảng nào. Đây là bảng quyết định cho topology của runbook này (Rancher cài lên chính cluster).
2. **Downstream Cluster Support** — Rancher tạo/quản lý được cluster nào.
3. **All Other Distros / Imported** — cluster có sẵn nào import vào được. Dòng "Any" ở bảng này **không** biến kubeadm thành nền tảng được chứng nhận làm Manager host — đừng dùng nhầm bảng.

Cũng trong matrix, đọc bảng **OS được chứng nhận** và ghi lại Ubuntu bản nào được liệt kê.

Mở thêm [SUSE Product Support Lifecycle — Rancher](https://www.suse.com/lifecycle/#rancher), tìm đúng minor Rancher vừa chốt và đọc riêng **EOM** và **EOL**. EOM là mốc giảm mức code maintenance; EOL là mốc hết support. Không suy EOL từ ngày phát hành hoặc từ support matrix.

**Ghi:** vào cột "Dải K8s liên quan" của dòng 1 phần B — các phiên bản/nền tảng K8s mà bảng (1) liệt kê; kèm ghi chú OS được chứng nhận. Cột lifecycle ghi `DATE: EOM=...; EOL=...`. Nên chụp màn hình matrix và lifecycle rồi lưu cạnh phiếu vì hai trang có thể đổi theo thời gian.

### 2.4. Điểm rẽ theo hướng

**[Theo hướng VENDOR-SUPPORTED]** — nếu bảng (1) không liệt kê kubeadm (thực tế thường vậy): topology hiện tại **không đạt** hướng này. Dừng tại đây và chọn một trong hai: (a) đổi topology sang nền tảng có trong bảng (RKE2/K3s) — khi đó dải K8s chính là các phiên bản RKE2/K3s được liệt kê; hoặc (b) chuyển hướng sang TECHNICALLY-COMPATIBLE và ghi quyết định này vào cuối phiếu. Không được lấy dòng "Any" của bảng Imported làm bằng chứng đi tiếp.

**[Theo hướng TECHNICALLY-COMPATIBLE]** — hợp đồng kỹ thuật là `kubeVersion` trong `Chart.yaml` của **đúng chart version**. Tra bằng:

```bash
RANCHER_VERSION='<chart-version-chốt-ở-bước-2.2>'
test "$RANCHER_VERSION" != '<chart-version-chốt-ở-bước-2.2>' \
  || { echo 'FAIL: chưa điền RANCHER_VERSION'; exit 1; }
export RANCHER_VERSION
helm show chart rancher-stable/rancher --version "$RANCHER_VERSION" \
  | grep -E '^(version|appVersion|kubeVersion):'
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
K8S_VERSION='<exact-patch-không-có-tiền-tố-v>'
test "$K8S_VERSION" != '<exact-patch-không-có-tiền-tố-v>' \
  || { echo 'FAIL: chưa điền K8S_VERSION'; exit 1; }
export K8S_VERSION
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

Mở [Ubuntu release cycle](https://ubuntu.com/about/release-cycle). Tìm đúng bản LTS đã ghi trong phần A, đọc ngày kết thúc **standard support**. Ngày này phải **sau** ngày cài dự kiến.

**[Theo hướng VENDOR-SUPPORTED]** — đối chiếu thêm: bản Ubuntu này có trong bảng OS của support matrix (đã đọc ở Bước 2.3) không. Không có → đổi OS hoặc ghi nhận không đạt hướng.

**Ghi:** dòng 4 phần B.

### 4.2. Đọc — repo gói theo minor

Mở [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), mục dành cho Debian-based. Điểm cần hiểu: `pkgs.k8s.io` tách **repo riêng cho từng minor** (`.../core:/stable:/v<K8S_MINOR>/deb/`) — đổi minor là phải đổi dòng repo, và repo minor này không chứa gói minor khác. Làm theo đúng trang đó để khai repo của minor đã chọn trên máy Ubuntu cùng bản với node.

### 4.3. Kiểm nhanh — repo có đúng gói của patch đã chọn

Chạy trên máy Ubuntu vừa khai repo:

```bash
sudo apt-get update
for p in kubeadm kubelet kubectl; do
  apt-cache madison "$p" | head -n 3
done
```

Cách đọc: cột thứ hai là Debian package version, dạng `<K8S_PATCH>-<DEBIAN_REVISION>`. Đạt khi **cả ba gói** đều có dòng bắt đầu bằng đúng patch đã chốt, và bạn ghi lại **nguyên chuỗi** đó — đây là giá trị pin khi cài, không chỉ là upstream patch.

**Ghi:** dòng 3 phần B (chuỗi package đầy đủ); kết quả vào phần D.

**Đi tiếp khi:** ba gói cùng version trong repo đúng minor, và Ubuntu còn standard support qua ngày cài.

---

## Bước 5 — containerd

### 5.1. Đọc — yêu cầu từ phía Kubernetes

Mở [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/). Ba điều cần rút ra: runtime phải nói chuyện qua **CRI**; cgroup driver nên là **systemd** (và phải khớp giữa kubelet với runtime); trang có mục riêng cho containerd với các bước cấu hình chuẩn.

### 5.2. Đọc — tương thích và lifecycle upstream

Mở [containerd RELEASES.md](https://github.com/containerd/containerd/blob/main/RELEASES.md). Đọc hai bảng riêng biệt:

- **Current State of containerd Releases**: status và upstream EOL của từng nhánh;
- **Kubernetes Support**: version containerd được khuyến nghị/kiểm thử kỹ với từng minor Kubernetes.

Không gộp hai bảng thành một khái niệm. Bảng Kubernetes Support quyết định giao tương thích; upstream EOL mô tả lifecycle của nhánh upstream và còn phải được đặt cạnh lifecycle của nguồn package ở Bước 5.4.

### 5.3. Tra + kiểm nhanh — Ubuntu repo có gì

Chạy trên máy Ubuntu cùng bản với node:

```bash
apt-cache policy containerd
```

Cách đọc: dòng `Candidate:` là Debian package version sẽ được cài. Phần đầu chuỗi cho biết upstream version; đối chiếu version đó với hàng Kubernetes Support cho minor đã chọn. Lưu ý containerd 1.x và 2.x có thể dùng khác **thế hệ file config** (`version = 2` / `version = 3`) — phải đọc tài liệu của đúng candidate và ghi lại thế hệ config, không suy từ ví dụ cũ.

### 5.4. Chọn nguồn lifecycle có thẩm quyền

Lifecycle chặn phải đi theo **artifact thực sự sẽ cài**:

- nếu cài binary/package từ upstream containerd: dùng upstream EOL trong `RELEASES.md`;
- nếu cài gói `.deb` từ Ubuntu archive: dùng coverage của Ubuntu/Canonical cho đúng release, repository component và subscription; vẫn ghi upstream EOL như rủi ro và mốc cần tái kiểm, nhưng không tự động thay thế lifecycle của package downstream;
- nếu đổi nguồn package, phải tra lại toàn bộ version, compatibility và lifecycle của nguồn mới.

Trên node mục tiêu, kiểm coverage thực tế:

```bash
pro security-status
```

Đối chiếu thêm [Ubuntu release cycle](https://ubuntu.com/about/release-cycle) và [Ubuntu security maintenance](https://ubuntu.com/security/esm). Không mặc định “Ubuntu LTS” đồng nghĩa mọi package đều có cùng coverage: Main và Universe, hệ thống có hoặc không có Ubuntu Pro, có thể khác nhau.

**Ghi:** dòng 5 phần B phải có cả chuỗi Debian package, upstream version/branch/status/EOL, nguồn package, coverage downstream và ngày tra. Cột `containerd` bảng C ghi PASS/FAIL theo bảng Kubernetes Support; kết quả `apt-cache policy` và coverage ghi vào phần D.

**Đi tiếp khi:** candidate chính xác đã được xác định trên đúng OS/architecture; version upstream của candidate nằm trong hàng Kubernetes Support cho minor đã chọn; lifecycle của đúng nguồn package còn hiệu lực qua ngày cài dự kiến. Nếu review bị stale, phải xử lý đúng thành phần gây stale — không mặc định đổi Kubernetes minor sẽ làm candidate containerd thay đổi.

---

## Bước 6 — CNI Flannel

### 6.1. Đọc — release mới nhất và những gì upstream (không) hứa

Mở [Flannel releases](https://github.com/flannel-io/flannel/releases). Lấy release **final** mới nhất (bỏ pre-release). Đọc release notes của bản đó; lưu ý trung thực: Flannel **không công bố** ma trận tương thích Kubernetes chính thức — vì vậy trong phiếu, dải K8s của Flannel ghi `không công bố`, và độ tin cậy dựa vào render/dựng thử (Bước 11) chứ không dựa vào tuyên bố vendor.

Tham chiếu nền: [Kubernetes Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) — cluster cần **plugin binary** tương thích spec từ `v0.4.0` trở lên. Không được diễn giải yêu cầu này thành “trường `cniVersion` trong conflist phải ≥0.4”. Theo [CNI specification](https://www.cni.dev/docs/spec/#version-considerations), runtime, plugin và network configuration có thể hỗ trợ các phiên bản độc lập; plugin công bố khả năng qua thao tác `VERSION`.

### 6.2. Kiểm nhanh — manifest của đúng tag

```bash
FLANNEL_VERSION='<release-tag-final-vừa-tra>'
test "$FLANNEL_VERSION" != '<release-tag-final-vừa-tra>' \
  || { echo 'FAIL: chưa điền FLANNEL_VERSION'; exit 1; }
export FLANNEL_VERSION
curl -fsSL -o kube-flannel.yml \
  "https://github.com/flannel-io/flannel/releases/download/${FLANNEL_VERSION}/kube-flannel.yml"
grep -E '^[[:space:]]+image:|"Network"|"cniVersion"' kube-flannel.yml
```

Cách đọc output:

- Các dòng `image:` phải pin tag cụ thể, **không** có `latest`/`master`; ghi riêng tag của `flannel` và `flannel-cni-plugin`.
- Dòng `"Network"` là Pod CIDR mặc định của manifest (thường `10.244.0.0/16`). Nếu khác Pod CIDR ở phần A của phiếu, ghi chú rõ: *khi cài phải sửa trường Network* — việc sửa thuộc runbook cài đặt, không sửa ở đây.
- Dòng `"cniVersion"` chỉ được **ghi nhận** là version của network configuration; giá trị `0.3.1` vẫn có thể hợp lệ khi plugin binary hỗ trợ các spec mới hơn. Không áp threshold vào trường này.

Sau khi binary đã có trên node, phép kiểm đúng theo CNI protocol là:

```bash
printf '{"cniVersion":"1.0.0"}' \
  | CNI_COMMAND=VERSION /opt/cni/bin/flannel | jq
printf '{"cniVersion":"1.0.0"}' \
  | CNI_COMMAND=VERSION /opt/cni/bin/portmap | jq
```

`supportedVersions` phải chứa `0.4.0` hoặc `1.x`. Nếu Bước 6 chỉ là tra cứu và node chưa có binary, ghi kiểm này là `DEFERRED-TO-INSTALL-TEST`; không được đổi thành FAIL chỉ vì conflist khai `0.3.1`.

**Ghi:** dòng 6 phần B; kết quả ba ý trên vào phần D.

**Đi tiếp khi:** tag final tải được manifest; mọi image đều pin tag cụ thể; Pod CIDR đã được đối chiếu; `cniVersion` đã được ghi nhận nhưng không bị áp threshold sai. Kiểm `VERSION` của binary phải đạt trong runbook dựng thử trước khi phát hành cluster.

---

## Bước 7 — Helm

### 7.1. Đọc

Hai nguồn: [Helm Version Support Policy](https://helm.sh/docs/v3/topics/version_skew/) — bảng Helm minor ↔ dải Kubernetes được hỗ trợ; và [Rancher Helm Version Requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements) — phiên bản Helm tối thiểu mà Rancher yêu cầu.

Cách chọn: lấy Helm minor mới nhất mà bảng skew ghi hỗ trợ minor K8s đã chọn, và không thấp hơn yêu cầu của Rancher. Điền cột `Helm` của bảng C.

### 7.2. Kiểm nhanh — Helm đang cài đúng bản định dùng

```bash
helm version --short
```

Đạt khi version in ra **đúng exact version đã chốt**. Nếu chưa đúng, cài lại ngay tại đây theo trang official; không đi tiếp sang Bước 8. OCI được bật mặc định từ Helm 3.8.0, trong khi Bước 8 dùng OCI chart. Mọi lệnh `helm show` và render từ đây trở đi phải chạy bằng đúng Helm của baseline.

**Ghi:** dòng 7 phần B; cột `Helm` bảng C; kết quả vào phần D.

**Đi tiếp khi:** `helm version --short` khớp exact version đã chốt và version đó đạt cả Helm↔Kubernetes skew lẫn yêu cầu Rancher.

---

## Bước 8 — cert-manager

### 8.1. Đọc

Mở [cert-manager Supported Releases](https://cert-manager.io/docs/releases/). Trang có bảng: minor nào đang supported, lifecycle và **hai dải Kubernetes riêng biệt** — *Supported* và *Tested*. Chọn minor đang supported có dải Supported chứa minor K8s đã chọn; trong minor đó lấy **patch mới nhất**. Nếu K8s nằm trong Supported nhưng ngoài Tested, vẫn có thể dùng nhưng phải ghi rõ, không tự ghi “tested”.

Lifecycle cert-manager có thể là ngày (`DATE`) hoặc sự kiện (`EVENT`, ví dụ “Release of X.Y”). Ghi nguyên kiểu và giá trị official; không tự đổi một sự kiện chưa có lịch chắc chắn thành ngày EOL giả. Cách tạo review proxy cho EVENT nằm ở Bước 12.2.

Đừng lấy số version từ command ví dụ trong [trang cài đặt Helm](https://cert-manager.io/docs/installation/helm/) làm "mới nhất" — trang ví dụ có thể cũ hơn trang releases.

**Ghi:** dòng 8 phần B (version, dải Supported/Tested, EOL); cột `cert-manager` bảng C.

### 8.2. Kiểm nhanh — release và chart có thật

```bash
CERT_MANAGER_VERSION='<release-tag-final-vừa-tra>'
test "$CERT_MANAGER_VERSION" != '<release-tag-final-vừa-tra>' \
  || { echo 'FAIL: chưa điền CERT_MANAGER_VERSION'; exit 1; }
export CERT_MANAGER_VERSION
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
TRAEFIK_CHART_VERSION='<chart-version-final-vừa-tra>'
test "$TRAEFIK_CHART_VERSION" != '<chart-version-final-vừa-tra>' \
  || { echo 'FAIL: chưa điền TRAEFIK_CHART_VERSION'; exit 1; }
export TRAEFIK_CHART_VERSION
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

Kiểm nhanh (chạy cho từng addon đã chọn bằng repo và tag vừa tra, không dùng version mẫu):

```bash
ADDON_REPO='<owner/repository>'
ADDON_TAG='<release-tag-final-vừa-tra>'
test "$ADDON_REPO" != '<owner/repository>' \
  && test "$ADDON_TAG" != '<release-tag-final-vừa-tra>' \
  || { echo 'FAIL: chưa điền ADDON_REPO/ADDON_TAG'; exit 1; }
curl -fsSL "https://api.github.com/repos/${ADDON_REPO}/releases/tags/${ADDON_TAG}" \
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

Các biến phải đến từ kết quả đã chốt trong chính phiên nghiên cứu hiện tại. Không gán version minh họa. Nếu mở shell mới, export lại từ phiếu rồi chạy gate sau:

```bash
: "${K8S_VERSION:?export K8S_VERSION từ Bước 3}"
: "${CERT_MANAGER_VERSION:?export CERT_MANAGER_VERSION từ Bước 8}"
: "${TRAEFIK_CHART_VERSION:?export TRAEFIK_CHART_VERSION từ Bước 9}"
: "${RANCHER_VERSION:?export RANCHER_VERSION từ Bước 2}"
: "${BASELINE_TODAY:?chạy lại Bước 0.1 trong phiên hiện tại}"
test "$BASELINE_TODAY" = "$(date +%F)" \
  || { echo 'FAIL: ngày phiên đã đổi; chạy lại Bước 0.1 và cập nhật lịch sử phiên'; exit 1; }

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

Ngày review trả lời câu hỏi: *"đến ngày nào thì phiếu này không còn tin được nữa và phải tra lại?"*. Không được ép mọi lifecycle thành một ngày giả. Trước tiên phân loại từng giá trị trong phần B:

- `DATE`: nguồn official công bố ngày hết support/EOL cụ thể.
- `EVENT`: nguồn official chỉ công bố một sự kiện, ví dụ `Release of 1.x`, chưa có ngày chắc chắn.

Chọn đúng **cơ quan chịu trách nhiệm cho artifact được cài**:

- Gói `deb` do Ubuntu cung cấp: lifecycle bắt buộc để ra quyết định là thời hạn security maintenance của đúng package/pocket theo Canonical. Ghi EOL upstream ở một trường riêng để theo dõi, nhưng không thay thế lifecycle downstream bằng EOL upstream.
- Binary/chart/manifest lấy trực tiếp từ upstream: dùng lifecycle upstream.
- Nếu chưa chứng minh được package thuộc pocket nào hoặc có được security coverage hay không, ghi `CHƯA XÁC MINH` và chưa phát hành baseline.

Tính các mốc như sau:

1. **Tiên quyết với `DATE`:** mọi ngày EOL bắt buộc phải **sau ngày cài dự kiến**. Thành phần hết hạn vào hoặc trước ngày cài phải bị loại hoặc đổi nguồn cung cấp.
2. Với mỗi `DATE`, tạo mốc review `EOL − 90 ngày`.
3. Với mỗi `EVENT`, tạo **mốc rà soát nội bộ**, không gọi là EOL:
   - nếu upstream công bố lịch tháng/quý cho release kích hoạt sự kiện, lấy ngày đầu của tháng/quý đó làm proxy bảo thủ và ghi rõ `PROXY`;
   - nếu chưa có lịch chính thức, lấy `BASELINE_TODAY + 90 ngày` làm mốc kiểm lại tạm thời;
   - khi sự kiện có ngày chính thức, đổi hàng đó sang `DATE` và tính lại.
4. **Mốc cài đặt** = `ngày cài dự kiến − 30 ngày`. Nếu mốc này đã qua, dùng chính `ngày cài dự kiến`; lần tra hiện tại đã nằm trong cửa sổ 30 ngày trước khi cài.
5. `ngày review = min(tất cả mốc DATE, tất cả proxy EVENT, mốc cài đặt)`.

Dùng đoạn dưới để tính theo ngày của **phiên hiện tại** và đồng thời chỉ ra thành phần tạo mốc sớm nhất. Không điền ngày mẫu vào runbook:

```bash
: "${BASELINE_TODAY:?chạy Bước 0.1 trong phiên hiện tại}"
test "$BASELINE_TODAY" = "$(date +%F)" \
  || { echo 'FAIL: BASELINE_TODAY không còn là ngày hiện tại'; exit 1; }

export PLANNED_INSTALL_DATE='<YYYY-MM-DD>'
export DATE_EOL_VALUES='<component=YYYY-MM-DD,component=YYYY-MM-DD>'
# Để chuỗi rỗng nếu không có EVENT. Giá trị ở đây là proxy review, không phải EOL.
export EVENT_REVIEW_PROXIES='<component=YYYY-MM-DD>'

python3 - <<'PY'
import os
from datetime import date, timedelta

today = date.fromisoformat(os.environ["BASELINE_TODAY"])
if today != date.today():
    raise SystemExit("FAIL: ngày phiên đã đổi; chạy lại Bước 0.1")

install_raw = os.environ["PLANNED_INSTALL_DATE"]
if install_raw.startswith("<"):
    raise SystemExit("FAIL: chưa điền PLANNED_INSTALL_DATE")
install = date.fromisoformat(install_raw)

def parse_pairs(name, required=False):
    raw = os.environ.get(name, "").strip()
    if not raw or raw.startswith("<"):
        if required:
            raise SystemExit(f"FAIL: chưa điền {name}")
        return {}
    result = {}
    for item in raw.split(","):
        component, value = item.strip().split("=", 1)
        result[component.strip()] = date.fromisoformat(value.strip())
    return result

hard_eols = parse_pairs("DATE_EOL_VALUES", required=True)
event_proxies = parse_pairs("EVENT_REVIEW_PROXIES")

expired = {name: value for name, value in hard_eols.items() if value <= install}
if expired:
    detail = ", ".join(f"{name}={value}" for name, value in expired.items())
    raise SystemExit(f"FAIL: EOL không sau ngày cài: {detail}")

candidates = {
    f"{name} (EOL-90)": value - timedelta(days=90)
    for name, value in hard_eols.items()
}
candidates.update({f"{name} (EVENT proxy)": value for name, value in event_proxies.items()})

install_minus_30 = install - timedelta(days=30)
candidates["installation"] = install if install_minus_30 < today else install_minus_30

cause, review = min(candidates.items(), key=lambda item: item[1])
print(f"BASELINE_TODAY={today}")
print(f"REVIEW_DATE={review}")
print(f"LIMITING_CAUSE={cause}")
if review < today:
    raise SystemExit("FAIL: baseline stale ngay khi lập")
print("OK: baseline chưa stale")
PY
```

Nếu baseline stale, sửa **đúng nguyên nhân trong `LIMITING_CAUSE`**: đổi phiên bản của thành phần đó, đổi kênh/nguồn cung cấp có lifecycle phù hợp, hoặc đổi ngày cài. Chỉ đổi minor Kubernetes khi chính Kubernetes hoặc giao của các dải tương thích là nguyên nhân. Nếu repository chỉ có một candidate và không có nguồn được hỗ trợ nào khác, ghi `BLOCKED` và dừng; không được tiếp tục thay version không liên quan cho tới khi phép tính tình cờ đạt.

Sau ngày review, bảng phải được tra cứu lại toàn bộ (ít nhất từ Bước 2) — không chỉ sửa một ô patch.

### 12.3. Kiểm nhanh — phiếu không còn ô bỏ trống

```bash
: "${BASELINE_SHEET:?chạy Bước 0.3 trong phiên hiện tại}"
grep -n '___' "$BASELINE_SHEET" && echo "FAIL: còn ô chưa điền" || echo "OK: phiếu đã điền đủ"
```

### 12.4. Phát hành

Bảng E + ngày review chính là nội dung đưa vào mục "Phiên bản" của runbook cài đặt. Kèm theo phiếu ghi đầy đủ (phần A–D) để người sau tái dựng được quyết định — một danh sách số phiên bản không kèm nguồn và ngày tra là thứ sẽ bị chép lại vĩnh viễn, đúng điều nguyên tắc đầu tài liệu cấm.

---

## Phụ lục A — Danh mục nguồn official theo thành phần

| Thành phần | Nguồn bắt buộc | Dùng ở bước |
| --- | --- | --- |
| Rancher — channel | [Choosing a Rancher Version](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/choose-a-rancher-version) | 2.1 |
| Rancher — compatibility | [SUSE Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/) (mở bằng browser) | 2.3 |
| Rancher — lifecycle | [SUSE Product Support Lifecycle](https://www.suse.com/lifecycle/) | 2.3, 12.2 |
| Rancher — yêu cầu cài | [Installation Requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/installation-requirements) | 2, 4 |
| Kubernetes — lifecycle | [Releases](https://kubernetes.io/releases/) | 3.1 |
| Kubernetes — skew | [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) | 3.4 |
| kubeadm — gói | [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) | 4.2 |
| Runtime — yêu cầu | [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) | 5.1 |
| containerd — lifecycle | [RELEASES.md](https://github.com/containerd/containerd/blob/main/RELEASES.md) | 5.2 |
| Ubuntu — package security/lifecycle | [`pro security-status`](https://documentation.ubuntu.com/pro-client/en/latest/explanations/how_to_interpret_the_security_status_command/) và [Ubuntu release cycle](https://ubuntu.com/about/release-cycle) | 5.4, 12.2 |
| CNI — yêu cầu | [Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) | 6.1 |
| CNI — lệnh `VERSION` | [CNI Specification](https://www.cni.dev/docs/spec/) | 6.2 |
| Flannel | [Releases](https://github.com/flannel-io/flannel/releases) | 6 |
| Helm — skew | [Version Support Policy](https://helm.sh/docs/v3/topics/version_skew/) | 7.1 |
| Helm — theo Rancher | [Helm Version Requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements) | 7.1 |
| Helm — OCI | [Use OCI-based registries](https://helm.sh/docs/topics/registries/) | 7.2, 8.2 |
| cert-manager | [Supported Releases](https://cert-manager.io/docs/releases/), [Helm install](https://cert-manager.io/docs/installation/helm/) | 8 |
| Traefik | [Chart releases](https://github.com/traefik/traefik-helm-chart/releases), [Kubernetes docs](https://doc.traefik.io/traefik/setup/kubernetes/) | 9 |
| local-path-provisioner | [Releases](https://github.com/rancher/local-path-provisioner/releases) | 10 |
| MetalLB | [Releases](https://github.com/metallb/metallb/releases), [metallb.io](https://metallb.io/) | 10 |
| cloudflared | [Downloads](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/), [Releases](https://github.com/cloudflare/cloudflared/releases) | 10 |

Trang "current" đổi nội dung theo thời gian — vì vậy phiếu luôn ghi **ngày tra** cạnh mỗi URL, và với trang render bằng JavaScript (SUSE matrix) nên lưu thêm screenshot.

---

## Phụ lục B — Mẫu mức chi tiết, không chứa snapshot

Phụ lục này chỉ mô tả **cách ghi**. Mọi giá trị phải được tra lại trong phiên hiện tại; không chọn version từ phụ lục.

**Phần B (trích):**

| Thành phần | Giá trị phải ghi |
| --- | --- |
| Rancher | exact chart/app version; dải `kubeVersion`; EOM và EOL loại `DATE`; URL SUSE; `$BASELINE_TODAY` |
| Kubernetes | exact patch; ngày EOL minor loại `DATE`; URL releases; `$BASELINE_TODAY` |
| kubeadm/kubelet/kubectl | exact Debian version; candidate từ repo minor đã chọn; `$BASELINE_TODAY` |
| containerd từ Ubuntu | exact Debian version + upstream version; pocket; Canonical security coverage loại `DATE`; upstream EOL ở trường riêng; `$BASELINE_TODAY` |
| cert-manager | exact release/chart version; dải Supported/Tested; lifecycle loại `DATE` hoặc `EVENT`; URL official; `$BASELINE_TODAY` |

**Phần E (trích):**

| Thành phần | Phiên bản | Nhãn cam kết | Ghi chú |
| --- | --- | --- | --- |
| Kubernetes | `<exact patch vừa tra>` | `TECHNICALLY-COMPATIBLE` hoặc `VENDOR-SUPPORTED` | package pin cùng patch |
| Rancher | `<exact chart/app vừa tra>` | theo Bước 12.1 | ghi topology và kết quả render |
| Flannel | `<exact tag vừa tra>` | theo Bước 12.1 | upstream không công bố dải K8s; ghi kết quả CNI `VERSION` |
| cert-manager | `<exact version vừa tra>` | theo Bước 12.1 | ghi dải Supported/Tested và lifecycle `DATE`/`EVENT` |

Cuối phiếu ghi `BASELINE_TODAY`, ngày cài dự kiến, `REVIEW_DATE`, `LIMITING_CAUSE` và quyết định loại/giữ. Những giá trị này phải do Bước 0.1 và 12.2 tạo trong phiên đang làm việc.
