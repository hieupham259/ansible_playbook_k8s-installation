# Báo cáo review & verify — `vmware-k8s-lab.md`

> **Đối tượng review:** `vmware-k8s-lab.md` (1557 dòng, footer ghi *"Runbook tạo ngày 2026-06-26"*)
> **Ngày review:** 2026-07-20
> **Phạm vi:** toàn bộ §1 → §17, bao gồm cả Phụ lục A/B/C
> **Kết luận ngắn:** Runbook **khả thi, không có lỗi kiến trúc**, chất lượng trên trung bình rõ rệt. Nhưng có **8 lỗi chặn** khiến người làm tuần tự sẽ khựng hoặc nhận kết quả verify sai, và vấn đề hệ thống lớn nhất là **runbook được viết dựa trên một ảnh chụp version đã trôi** (chart Traefik < v33, containerd 1.7, Rancher 2.13.4, k8s 1.34 sắp hết hạn).

---


## Mục lục

> Sắp theo **trình tự thi công** — đọc và làm từ trên xuống.

**Định hướng**

1. [Phương pháp & mức độ tin cậy](#1-phương-pháp--mức-độ-tin-cậy)
2. [Bảng tổng hợp](#2-bảng-tổng-hợp)

**Thi công — làm theo thứ tự**

3. [Bước 0 — Chọn bộ version](#3-bước-0--chọn-bộ-version)
4. [Bước 1 — Ma trận tương thích & checklist đổi version](#4-bước-1--ma-trận-tương-thích--checklist-đổi-version)
5. [Bước 2 — Vá lỗi chặn](#5-bước-2--vá-lỗi-chặn) 🔴 8 mục
6. [Bước 3 — Vá lỗi quan trọng](#6-bước-3--vá-lỗi-quan-trọng) 🟠 14 mục
7. [Bước 4 — Cải thiện](#7-bước-4--cải-thiện) 🟡 13 mục
8. [Bước 5 — Thêm chương mới & kế hoạch](#8-bước-5--thêm-chương-mới--kế-hoạch)

**Tham chiếu**

9. [Đánh giá theo chương](#9-đánh-giá-theo-chương)
10. [Những chỗ runbook làm ĐÚNG (đã verify)](#10-những-chỗ-runbook-làm-đúng-đã-verify)
11. [Verify §9.2 — chương chính xác nhất runbook](#11-verify-92--chương-chính-xác-nhất-runbook)
12. [Nguồn tham chiếu](#12-nguồn-tham-chiếu)
## 1. Phương pháp & mức độ tin cậy

Review gồm 2 lớp, **được đánh dấu tách bạch** trong từng mục:

| Ký hiệu                 | Cơ sở                                                                                                                    | Độ tin cậy                                                                                     |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| 📖**Đọc mã**     | Suy ra từ việc đọc trực tiếp lệnh/YAML trong runbook, đối chiếu ngữ nghĩa shell/kubectl/Helm                   | Cao — không phụ thuộc nguồn ngoài                                                           |
| 🌐**Verify nguồn** | Tra cứu tài liệu chính chủ (kubernetes.io, GitHub upstream, suse.com, developers.cloudflare.com, packages.ubuntu.com) | Cao — có URL + trích dẫn                                                                      |
| ⏳**Chờ**          | Chưa xác nhận được                                                                                                   | **Không dùng làm căn cứ** — *(vòng 2: không còn mục nào ở trạng thái này)* |

**Nguyên tắc áp dụng:** không claim nào được ghi "sai" nếu chỉ dựa trên trực giác. Ba chỗ mà nhận định ban đầu của người review **bị chính quá trình verify bác bỏ** đều được giữ lại và đính chính công khai ở [§10.4](#104-ba-nhận-định-ban-đầu-đã-bị-bác-bỏ) — để người đọc biết chỗ nào từng bị nghi oan.

---

## 2. Bảng tổng hợp

### 🔴 Lỗi chặn (8)

| ID       | Vị trí               | Vấn đề                                                                              | Cơ sở |
| -------- | ---------------------- | -------------------------------------------------------------------------------------- | ------- |
| [A1](#a1) | §8.7 dòng 821        | Dùng`helm` trước khi Helm được cài (§9.3 dòng 991)                          | 📖      |
| [A2](#a2) | §8.7b dòng 835–848  | Thiếu hẳn lệnh cài`local-path-provisioner`; PVC test chắc chắn `Pending`     | 📖      |
| [A3](#a3) | §8.3 dòng 756–762   | "Test cross-node"**không** đảm bảo cross-node → PASS giả                   | 📖      |
| [A4](#a4) | §8.8 dòng 850–863   | Checklist yêu cầu Traefik (§9) và cert-manager (§14.1) nhưng nằm ở §8         | 📖      |
| [C1](#c1) | §2.1 dòng 88 + §14  | Toàn bộ lý do ghim k8s 1.34 đã sai — Rancher 2.14.3 đã support 1.35            | 🌐      |
| [C2](#c2) | §5.5 dòng 429–443   | Ubuntu 24.04 ship containerd**2.2.1**, lệnh `sed` im lặng vô hiệu          | 🌐      |
| [C7](#c7) | §9.3 dòng 1025–1032 | Dashboard Traefik: sai cổng (9000→8080)**và** không được bật mặc định | 🌐      |
| [C8](#c8) | §9.4 dòng 1107       | `--set service.type=NodePort` sai key, Helm im lặng bỏ qua                         | 🌐      |

### 🟠 Sai/thiếu quan trọng (14)

| ID         | Vị trí                  | Vấn đề                                                                   | Cơ sở |
| ---------- | ------------------------- | --------------------------------------------------------------------------- | ------- |
| [C3](#c3)   | §2.1 ~dòng 96–98       | Chính sách version-skew sai (1 minor → thực tế 3 minor)                | 🌐      |
| [C4](#c4)   | §12.2 dòng 1217–1254   | Manifest`cloudflared` thiếu `--metrics` + `livenessProbe`            | 🌐      |
| [C5](#c5)   | §16 dòng 1450–1451     | 2 URL Cloudflare đã đổi đường dẫn (301)                             | 🌐      |
| [C6](#c6)   | §12.3 dòng 1265, §14.3 | Tên UI Cloudflare đã đổi: "Public Hostname" → "Published application" | 🌐      |
| [B1](#b1)   | §14.5, §15              | Thiếu hẳn quy trình`kubeadm upgrade` — trong khi 1.34 sắp EOL        | 📖      |
| [B2](#b2)   | §14.3                    | Rancher UI phơi ra Internet không có Cloudflare Access                   | 📖      |
| [B3](#b3)   | §14.2 dòng 1352         | Giải thích cơ chế TLS sai (SNI vs Host header)                          | 📖      |
| [B5](#b5)   | §2.2, §2.3              | RAM host 16 GB là kịch trần, không đủ thực tế                       | 📖      |
| [B6](#b6)   | §8.4–§8.6              | Phụ thuộc biến`$A` đặt từ §8.3                                     | 📖      |
| [B7](#b7)   | §8.5 dòng 789           | `kubectl get endpoints` — API deprecated + sai chỗ cần nhìn           | 📖🌐    |
| [B8](#b8)   | §8.6 dòng 811–813      | Race giữa`port-forward &` và `curl`; `kill %1` cần job control     | 📖      |
| [B9](#b9)   | §4.1, §4.5 dòng 227    | Mô tả clone không khớp thực tế VMware                                 | 📖      |
| [B13](#b13) | §15                      | Không có backup/etcd snapshot                                             | 📖      |
| [B14](#b14) | §4 (8 link)              | `servers.md` **không tồn tại** cạnh runbook → mọi link gãy   | 📖      |

> ℹ️ **Về khoảng trống trong dãy B:** thiếu B4, B10, B11, B12 là **có chủ đích**, không phải sót mục. Bốn ID đó được nêu ở vòng đọc đầu rồi bị hạ mức sau khi verify: **B4** (`image: cloudflared:latest`) → chuyển sang [§10.4](#104-ba-nhận-định-ban-đầu-đã-bị-bác-bỏ) vì official Cloudflare cũng dùng `:latest`; **B10** → [D8](#d8); **B11** → [D9](#d9); **B12** → [D2](#d2). Giữ nguyên số cũ thay vì đánh lại để mọi tham chiếu chéo trong file không gãy.

### 🟡 Cải thiện (13)

| ID         | Vị trí                          | Vấn đề                                                                            |
| ---------- | --------------------------------- | ------------------------------------------------------------------------------------ |
| [D1](#d1)   | §2.1 dòng 88                    | Patch pin lạc hậu:`1.34.8` → `1.34.9`                                         |
| [D2](#d2)   | §2.1                             | cert-manager "vd v1.16.x" đã cũ (hiện 1.20.x/1.21)                               |
| [D3](#d3)   | §5.4                             | Thiếu cảnh báo`br_netfilter` — kubeadm 1.30+ không còn preflight-check       |
| [D4](#d4)   | §8.7 dòng 823                   | `--set args={...}` thiếu quote                                                    |
| [D5](#d5)   | §9.1 dòng 943                   | "đọc`Endpoints`" → thực tế **EndpointSlices**                           |
| [D6](#d6)   | §9.1 dòng 947                   | Tên IngressClass bám theo tên Helm release, không hardcode                       |
| [D7](#d7)   | §9.3 dòng 1006–1007            | Nên nói rõ container nghe 8000/8443, Service publish 80/443                       |
| [D8](#d8)   | §5.3 dòng 396                   | `systemctl mask swap.target` phi chuẩn, thừa                                     |
| [D9](#d9)   | §17 Phụ lục B dòng 1512–1515 | Bước`strictARP` thừa với kube-proxy mode iptables                              |
| [D10](#d10) | §14/§15                         | Thiếu 2 giới hạn Cloudflare: upload 100 MB, WebSocket idle timeout                |
| [D11](#d11) | §12.3                            | DNS tự tạo chỉ đúng với zone Full Setup                                        |
| [D12](#d12) | §9.2 dòng 979                   | Lớp tương thích annotation mô tả quá mơ hồ, thiếu footgun                  |
| [D13](#d13) | §9.2 dòng 960                   | CVSS 9.2 trình bày như số tuyệt đối, trong khi nginx.org xếp**Medium** |

### Đối chiếu version — runbook vs thực tế

Toàn bộ số hiệu phiên bản runbook khai (chủ yếu ở §2.1 dòng 85–98), đối chiếu với thực tế ngày **2026-07-20**:

| # | Thành phần | Runbook khai | Thực tế (verify 2026-07-20) | Trạng thái |
| --- | --- | --- | --- | --- |
| 1 | **Ubuntu Server** | 24.04.x LTS (Noble) | 24.04.4 là point release hiện hành | ✅ Khớp |
| 2 | **Kubernetes** (minor) | `v1.34` | 3 nhánh còn support: **1.36 / 1.35 / 1.34**. 1.34 vào maintenance **27/08/2026**, EOL **27/10/2026** | ⚠️ Đúng nhưng sắp hết đời |
| 3 | **Kubernetes** (patch) | `1.34.8` / `1.34.8-1.1` | **`1.34.9`** / `1.34.9-1.1` (09/06/2026) | ⚠️ Chậm 1 patch → [D1](#d1) |
| 4 | **containerd** | `1.7.x` (gói Ubuntu) | **`2.2.1-0ubuntu1~24.04.3`** (amd64)<br>`1.7.12-0ubuntu4` (arm64) | ❌ **SAI** → [C2](#c2) |
| 5 | **Flannel** | không ghi version | **v0.28.7** (07/07/2026) | ✅ Không xung đột |
| 6 | **Traefik** (proxy) | `v3.x` | **v3.7.8** mới nhất; chart 41.0.2 pin **v3.7.6** | ✅ Khớp dòng |
| 7 | **Traefik** (chart) | "chart mới nhất" | **41.0.2** | ⚠️ Khai đúng, **hướng dẫn viết theo chart < v33** → [C7](#c7) |
| 8 | **cloudflared** | `:latest` | Manifest official Cloudflare **cũng dùng `:latest`** | ✅ Khớp nguồn |
| 9 | **Rancher** | `2.13.4` — *"bản mới nhất"* | **2.14.3** (29/06/2026). Nhánh 2.13 đã tới **2.13.7** | ❌ **SAI** → [C1](#c1) |
| 10 | **Rancher** → k8s support | 2.13.4 certify `1.32–1.34` | Đúng | ✅ Khớp |
| 11 | *"1.35 chưa Rancher nào support"* | — | **2.14.3 certify `1.33–1.35`** | ❌ **SAI** → [C1](#c1) |
| 12 | **cert-manager** | "mới nhất (vd `v1.16.x`)" | **1.20.x** (2 nhánh support: 1.19, 1.20); artifacthub có 1.21.0 | ⚠️ Ví dụ lạc hậu 4 minor → [D2](#d2) |
| 13 | **MetalLB** | `v0.16.1` | **v0.16.1** | ✅ Khớp |
| 14 | **metrics-server** | không ghim | **0.9.x**, support k8s 1.34+ | ✅ Phù hợp |
| 15 | **ingress-nginx** | bản cuối `v1.15.1`, retired 03/2026 | `controller-v1.15.1` **19/03/2026**; repo archived **24/03/2026** | ✅ **Chính xác** |
| 16 | **nginx** (vá CVE-2026-42945) | `1.30.1` / `1.31.0` | **1.30.1 / 1.31.0** | ✅ Khớp verbatim |

**Ngoài bảng — một spec bị khai sai:** version-skew. Runbook (§2.1, ghi chú dưới bảng) nói *kubelet thấp hơn control plane tối đa **1 minor***; chính sách hiện hành là **3 minor** → [C3](#c3).

#### Cách đọc bảng

| Nhóm | Số mục | Nhận định |
| --- | --- | --- |
| ❌ **Sai thật sự** | 2 (mục 4, 9+11) | Cả hai đều là 🔴, và **cùng một nguyên nhân**: runbook chụp ảnh hệ sinh thái ở thời điểm viết (26/06/2026) rồi upstream trôi tiếp — ở đúng chỗ runbook lại tuyên bố chắc chắn (*"bản mới nhất"*, *"chưa Rancher nào support"*) |
| ⚠️ **Lạc hậu nhẹ** | 3 (mục 3, 7, 12) | Vô hại nếu cập nhật số — **trừ mục 7**: runbook khai đúng ("chart mới nhất") nhưng phần hướng dẫn dashboard viết theo chart cũ. Loại lỗi khó thấy nhất, vì bảng version không hề sai |
| ✅ **Khớp** | 7 | Trong đó mục **15 và 16** là hai claim bị nghi ngờ nặng nhất ở vòng đọc đầu, và hoá ra runbook **chính xác tuyệt đối** (xem [§11](#11-verify-92--chương-chính-xác-nhất-runbook)) |

> 💡 Bảng này là căn cứ trực tiếp cho khuyến nghị **rebase** ở [§3](#3-bước-0--chọn-bộ-version): vấn đề không nằm ở tư duy kiến trúc mà ở **độ trôi version**.

---

## 3. Bước 0 — Chọn bộ version

**Nhận định về tính khả thi:** runbook **chạy được** sau khi vá [C2](#c2) và [C7](#c7) — **không có lỗi kiến trúc**. Mô hình cloudflared in-cluster → Traefik ClusterIP là đúng và gọn.

**Nhưng vấn đề lớn nhất không phải sai kỹ thuật rời rạc**, mà là runbook được viết dựa trên **một ảnh chụp version đã trôi**:

| Thành phần  | Runbook giả định          | Thực tế 2026-07-20                                                                     |
| ------------- | ---------------------------- | ---------------------------------------------------------------------------------------- |
| chart Traefik | < v33 (cổng dashboard 9000) | **41.0.2** (cổng 8080)                                                            |
| containerd    | 1.7.x                        | **2.2.1**                                                                          |
| Rancher       | 2.13.4 "mới nhất"          | **2.14.3**                                                                         |
| k8s           | 1.34 "trần Rancher"         | 1.34 vào maintenance mode**27/08/2026**; 1.35 đã được Rancher 2.14.3 certify |

Bốn lỗi chặn ([C1](#c1) [C2](#c2) [C7](#c7) [C8](#c8)) đều là **hệ quả trực tiếp** của độ trôi này, không phải lỗi tư duy.

### Khuyến nghị

**Rebase**, không vá điểm:

1. **k8s 1.35** (thay 1.34) — runway tới 2027-02-28 thay vì 2026-10-27. ⚠️ Đọc [§4](#4-bước-1--ma-trận-tương-thích--checklist-đổi-version) trước: bộ version này có 3 rủi ro và 2 điểm phải sửa
2. **Rancher 2.14.3** (thay 2.13.4) — certify 1.33–1.35, đúng matrix
3. **containerd 2.x** — viết lại §5.5 với khối config tường minh
4. **chart Traefik 41.x** — sửa cổng dashboard, `service.spec.type`, rà lại toàn §9
5. **Bổ sung 3 mục còn thiếu:** quy trình `kubeadm upgrade` ([B1](#b1)), backup etcd ([B13](#b13)), Cloudflare Access cho Rancher ([B2](#b2))

Nếu **buộc phải giữ 1.34 + Rancher 2.13.4** (ví dụ để khớp môi trường có sẵn), thì tối thiểu phải vá [C2](#c2) và [C7](#c7), và **sửa lại lý do** ở §2.1 dòng 88 — vì lý do hiện tại đã sai, dù kết luận (ghim 1.34) vẫn có thể chấp nhận được như một lựa chọn bảo thủ.

---

## 4. Bước 1 — Ma trận tương thích & checklist đổi version

> **Vì sao có mục này:** khuyến nghị *"rebase sang Rancher 2.14.3 + k8s 1.35"* ở [§3](#3-bước-0--chọn-bộ-version) ban đầu chỉ dựa trên **một** cạnh đã verify (Rancher → k8s, lấy từ support matrix SUSE). Tương thích trong cụm k8s là **đồ thị ~10 cạnh**, không phải chuỗi. Mục này kiểm chứng toàn bộ đồ thị — và tìm ra **2 lỗi trong chính khuyến nghị đó** cùng **1 rủi ro chưa ai nêu**.

### 4.1. Kết quả từng cạnh

Nền: **k8s v1.35 "Timbernetes"** ra 17/12/2025, patch mới nhất **v1.35.6** (09/06/2026), EOL **28/02/2027**.

| Cạnh | Verdict | Bằng chứng |
| --- | --- | --- |
| **Rancher 2.14.3 → k8s 1.35** | ✅ | Matrix: RKE2/k3s/AKS/EKS/GKE/ACK đều `v1.33–v1.35`. Chart `kubeVersion: < 1.36.0-0` |
| **containerd → k8s 1.35** | ⚠️ | Matrix containerd: 1.35 cần **`2.2.0+` / `2.1.5+` / `1.7.28+`**. Xem [4.3](#43-rủi-ro-1--containerd-221-của-ubuntu) |
| **pause image** | ✅ | kubeadm 1.35 `PauseVersion = "3.10.1"`; containerd 2.2 `DefaultSandboxImage = pause:3.10.1` — **khớp, không cần override** |
| **Traefik chart 41.0.2 → k8s 1.35** | ✅ | `kubeVersion: ">=1.25.0-0"` (chỉ có sàn, không trần) |
| **cert-manager → k8s 1.35** | ⚠️ | 1.20 support `1.32–1.35` (**1.35 ở mép trên**, EOL ~11/2026); **1.21 support `1.33–1.36`** |
| **Rancher 2.14.3 → cert-manager** | ✅ | Chart `_helpers.tpl`: `$requiredVersion := "1.15.0"` — **sàn 1.15.0, không trần** |
| **metrics-server 0.9.x → k8s 1.35** | ✅ | Matrix README: `0.9.x → 1.34+`. v0.9.0 ra **13/07/2026**, sau khi 1.35 đã phát hành |
| **Flannel v0.28.7 → k8s 1.35** | 🟡 | **Flannel không công bố matrix.** README chỉ nói *"For Kubernetes v1.17+"*. Manifest dùng toàn API GA. Suy ra từ **vắng mặt ràng buộc trần**, không phải khẳng định của dự án |
| **MetalLB v0.16.1 → k8s 1.35** | 🟡 | **MetalLB cũng không công bố matrix.** Chart `kubeVersion: ">= 1.19.0-0"`. Tương tự Flannel: suy ra, không khẳng định |
| **kubeadm 1.35 trên Ubuntu 24.04** | ✅ | Repo `pkgs.k8s.io/core:/stable:/v1.35/deb/` hợp lệ, có `amd64`. Patch mới nhất **`1.35.6-1.1`** |
| **Rancher Manager trên kubeadm** | ⚠️ | **Docs cho phép, matrix KHÔNG certify** — xem [4.5](#45-rủi-ro-3--kubeadm-không-nằm-trong-matrix-certified) |

### 4.2. Bộ version đề xuất (đã sửa 2 lỗi)

| Thành phần | Bản đề xuất | Ghi chú |
| --- | --- | --- |
| Ubuntu Server | 24.04.x LTS | — |
| kubeadm / kubelet / kubectl | **`1.35.6-1.1`** | Repo `v1.35` |
| containerd | **≥ 2.2.5** (mới nhất 2.2.6) | ⚠️ **Sửa lỗi** — xem [4.3](#43-rủi-ro-1--containerd-221-của-ubuntu) |
| pause | `registry.k8s.io/pause:3.10.1` | Mặc định của cả hai, không cần chỉnh |
| Flannel | **v0.28.7** | Dùng URL **release-pinned**, không dùng master |
| Traefik chart | **41.0.2** → Traefik **v3.7.6** | ⚠️ **Sửa lỗi** — chart 41.0.2 pin **v3.7.6**, không phải v3.7.8 |
| cert-manager | **v1.21.0** | ⚠️ **Đổi từ 1.20** — 1.20 có 1.35 ở mép trên và EOL ~11/2026 |
| Rancher | **2.14.3** | Helm **≥ 3.18** |
| metrics-server | **v0.9.0** | — |
| MetalLB | **v0.16.1** | Qua Helm chart `0.16.1` hoặc image tag |

### 4.3. Rủi ro 1 — containerd 2.2.1 của Ubuntu

containerd **2.2.1** (18/12/2025) nằm **trước** bản **2.2.5**, bản vá 5 CVE trong đó **3 Critical**: `CVE-2026-50195` (CRI image-cache poisoning → RCE xuyên pod), `CVE-2026-53488` (image-config label propagation → chạy lệnh trên host), `CVE-2026-53492` (CDI spec trust). Fix ở `2.2.5` / `2.1.9` / `2.3.2`.

> ⚠️ **CHƯA XÁC MINH — không được kết luận vội:** gói Ubuntu là **`2.2.1-0ubuntu1~24.04.3`**, nằm trong pocket **`[security]`**, hậu tố `~24.04.3` cho thấy đã qua vài vòng cập nhật. Ubuntu **backport bản vá bảo mật mà giữ nguyên số version upstream** — nên `2.2.1-0ubuntu1~24.04.3` **có thể đã chứa các fix của 2.2.5**. Báo cáo này **chưa kiểm chứng** điều đó.
>
> **Việc cần làm trước khi kết luận:** tra Ubuntu Security Notice (USN) cho 3 CVE trên, hoặc chạy `apt changelog containerd` trên node. **Không** tự ý cài containerd upstream thay gói Ubuntu chỉ dựa trên số version — làm vậy có thể còn tệ hơn.

Ràng buộc kèm theo: **k8s 1.35 là bản cuối hỗ trợ containerd dòng 1.x**. Muốn lên 1.36 sau này thì bắt buộc phải ở containerd 2.0+.

### 4.4. Rủi ro 2 — thay đổi RBAC exec/WebSocket ở k8s 1.35 ⚠️ ẢNH HƯỞNG RANCHER UI

Đây là rủi ro **chưa mục nào trong báo cáo nêu**, và nó đánh trúng đúng thứ §8.6 của runbook lo lắng.

k8s 1.35 chuyển `pods/exec`, `pods/attach`, `pods/portforward` từ SPDY sang **WebSocket**, và bật mặc định feature gate `AuthorizePodWebsocketUpgradeCreatePermission`. Hệ quả:

> Gọi exec/attach/port-forward qua WebSocket giờ cần verb **`create`**, không còn đủ với `get`.

Ảnh hưởng trực tiếp: **shell trong UI Rancher**, tab **Logs**, và mọi role read-only đang dựa vào `get` để exec. Nếu nâng lên 1.35 mà không sửa RBAC, triệu chứng sẽ là *"tab Shell của Rancher trắng"* — **đúng triệu chứng mà §8.6 của runbook mô tả**, nhưng nguyên nhân hoàn toàn khác và không có trong bảng troubleshooting.

**Xử lý:** cấp verb `create` cho `pods/exec` trong Role/ClusterRole liên quan, hoặc tạm đặt feature gate về `false`.

### 4.5. Rủi ro 3 — kubeadm không nằm trong matrix certified

Matrix Rancher 2.14.3 có **hai bảng tách biệt**, rất dễ đọc nhầm:

| Bảng | Nội dung | Có kubeadm? |
| --- | --- | --- |
| *"Supported Kubernetes Platforms for **Rancher Manager**"* | RKE2, k3s, AKS, EKS, GKE, ACK — `v1.33–v1.35` | ❌ **Không có dòng nào** |
| *"All Other Distros — **Importing** Existing Clusters"* | `Any` → `v1.33–v1.35` | ✅ nhưng chỉ để **import cụm downstream** |

Dòng `Any` **không** phải chứng nhận cho việc **host Rancher Manager**. Footnote SLA cũng tách bạch hai việc này.

→ Đọc đúng: **cài được, docs cho phép (*"any Kubernetes cluster… upstream Kubernetes"*), nhưng ngoài vùng certified/SLA.** Ràng buộc kỹ thuật cứng duy nhất docs nêu: *"the Kubernetes cluster must have the **aggregation API layer** properly configured"*.

### 4.6. Rủi ro 4 — tài nguyên dưới ngưỡng tài liệu hoá 4 lần

| | Rancher docs yêu cầu | Runbook cấp |
| --- | --- | --- |
| Cấu hình nhỏ nhất ("Small") | **4 vCPU / 16 GB RAM — mỗi node, trên 3 node** | **2 vCPU / 4 GB** |
| Replica Rancher | Chart mặc định **3**; docs: production cần **≥ 3 node** | Runbook ép `replicas=1` |
| Sizing cho single-replica | **Không tồn tại trong tài liệu** | — |

Đây là nâng cấp mức nghiêm trọng cho [B5](#b5): không phải *"chật"* mà là **¼ RAM và ½ CPU so với ngưỡng thấp nhất Rancher công bố**. Trả lời thẳng cho câu hỏi *"có stable không"*: cấu hình này **không nằm trong bất kỳ vùng nào Rancher từng test**.

### 4.7. Thay đổi khác của k8s 1.35 cần biết

| Thay đổi | Ảnh hưởng tới lab này |
| --- | --- |
| **cgroup v1 bị gỡ bỏ** — kubelet **từ chối khởi động** trên host cgroup v1 (`failCgroupV1` mặc định `true`) | ✅ **An toàn** — Ubuntu 24.04 mặc định cgroup v2. Liên quan trực tiếp tới [C2](#c2) |
| `--pod-infra-container-image` bị gỡ khỏi kubelet | ✅ Không ảnh hưởng cụm kubeadm |
| kube-proxy `ipvs` **deprecated** (cảnh báo, còn chạy) | Củng cố [D9](#d9): `strictARP` của MetalLB càng không cần |
| `KubeletEnsureSecretPulledImages` lên beta, bật mặc định | Có thể ảnh hưởng pod dùng image private đã cache |
| **Không có API GA nào bị gỡ ở 1.35** | ✅ Flannel, Traefik, cert-manager, metrics-server, Rancher đều **sạch về API** |

### 4.8. Checklist nâng runbook lên 1.35 + 2.14.3 — chỗ nào chưa khớp

**Kết luận cài đặt: không có lỗi chặn.** Đã kiểm `chart/values.yaml` của Rancher `release/v2.14`: `networkExposure.type: "ingress"` (mặc định) + `ingress.enabled: true` → Ingress vẫn được tạo, lệnh `helm install` của §14.2 chạy nguyên si, mọi cờ đang dùng đều còn hợp lệ. Phần lớn việc là **đổi chuỗi version**.

#### A. Đổi số version — thuần cơ học

| Dòng | Hiện tại | Đổi thành |
| --- | --- | --- |
| 31 (mục lục) | `Cài Rancher 2.13.4 & quản lý cụm` | `2.14.3` |
| 88 | `v1.34` — patch `1.34.8` | **`v1.35`** — patch **`1.35.6`** |
| 89 | `containerd 1.7.x` | **`containerd 2.x`** (gói Ubuntu ship 2.2.1) |
| 93 | `Rancher 2.13.4` … chứng nhận `1.32–1.34` | **`Rancher 2.14.3`** … chứng nhận **`1.33–1.35`** |
| 94 | cert-manager "mới nhất (vd `v1.16.x`)" | **`v1.21.0`** (sàn chart Rancher: `1.15.0`) |
| 448 | tiêu đề §5.6 `pin v1.34` | `pin v1.35` |
| 457, 460 | `pkgs.k8s.io/core:/stable:/**v1.34**/deb/` | **`v1.35`** — **2 chỗ**, sửa cả `Release.key` lẫn dòng `deb` |
| 465–467 | `VER='1.34.8-1.1'` | **`VER='1.35.6-1.1'`** |
| 671 | `3 node phải cùng minor v1.34` | `v1.35` |
| 854 (gate #1) | Server **≤ v1.34** | Server **≤ v1.35** |
| 1303, 1325 | tiêu đề `Rancher 2.13.4` | `2.14.3` |
| 1334 | `--version 2.13.4` | **`--version 2.14.3`** |
| 1349 | *"ghim đúng bản trong support matrix (k8s ≤ 1.34)"* | `k8s ≤ 1.35` |
| 1452 (§16) | URL support matrix `rancher-v2-13-4` | **`rancher-v2-14-3`** |
| 1558 (footer) | `Kubernetes v1.34.8`, `containerd 1.7.x`, `Rancher 2.13.4` | cập nhật cả 3 |

#### B. Đoạn phải viết lại, không chỉ sửa số

| Dòng | Vấn đề |
| --- | --- |
| **77** | *"Phiên bản k8s bị pin ở 1.34 vì Rancher 2.13.4 chỉ chứng nhận tới đó"* — lý do không còn đúng ([C1](#c1)) |
| **96** | Version-skew sai: "tối đa **1 minor**" → **3 minor** ([C3](#c3)) |
| **98** | Cả đoạn *"Vì sao pin 1.34 chứ không 1.35"* — tiền đề (*"1.35 chưa Rancher nào support"*) đã sai ([C1](#c1)). Phải viết lại thành lý do chọn 1.35 |
| **1391** | *"khi Rancher 2.14+ ra và chứng nhận 1.35/1.36"* — **2.14 đã ra rồi**. Viết lại theo mốc mới: 2.14.3 certify `1.33–1.35`, chưa bản nào certify 1.36 |
| **732** | *"Rancher 2.13.4 + cert-manager ăn thêm ~2–3 GB"* — đổi tên bản; con số giữ nguyên |

#### C. Mục MỚI cần thêm (runbook chưa có gì tương ứng)

| Nội dung | Đặt ở | Vì sao |
| --- | --- | --- |
| **Helm ≥ 3.18** là yêu cầu của Rancher 2.14 | §9.3 (chỗ cài Helm) | Runbook cài Helm bản mới nhất nên đạt, nhưng chưa nêu thành điều kiện |
| **Cảnh báo RBAC exec/WebSocket của k8s 1.35** | §8.6 + bảng lỗi §15 | Xem mục **4.4** ở trên. Triệu chứng *"tab Shell trắng"* trùng hệt §8.6 nhưng nguyên nhân khác |
| **k8s 1.32 bị gỡ ở Rancher 2.14** | §14.5 | Ai đang ở 2.13.4 + k8s 1.32 phải nâng k8s lên ≥1.33 **trước** khi nâng Rancher |
| **Tuỳ chọn Gateway API** | §14.2 (ghi chú) | 2.14 thêm `networkExposure.type: gateway` với `gatewayClass.name: "traefik"` — hướng thay thế Ingress |
| **`imagePullCredentialsVerificationPolicy`** | §15 | Gate `KubeletEnsureSecretPulledImages` lên beta ở 1.35, có thể ảnh hưởng pod dùng image private đã cache |

#### D. Không cần làm gì

- **`--set ingress.ingressClassName=traefik`** — vẫn là key hợp lệ ở chart 2.14
- **`--set ingress.tls.source=rancher`**, **`--set replicas=1`**, **`--set hostname=`**, **`--set bootstrapPassword=`** — không đổi
- **cert-manager cài không `--version`** — cho ra 1.21.0, thoả sàn `1.15.0` của chart 2.14
- **Rancher 2.14 bỏ 3 annotation `nginx.ingress.kubernetes.io/proxy-*-timeout`** — runbook dùng Traefik nên **có lợi**, không phải sửa
- **cgroup v1 bị gỡ ở 1.35** — Ubuntu 24.04 mặc định cgroup v2, không ảnh hưởng
- **pause image** — kubeadm 1.35 và containerd 2.2 đều `3.10.1`, khớp sẵn

> ⚠️ Các lỗi [C2](#c2), [C7](#c7), [C8](#c8) và nhóm [A1](#a1)–[A4](#a4) **không liên quan tới việc đổi version** — chúng là lỗi của runbook và phải vá độc lập, dù ở lại 1.34 hay lên 1.35.

### 4.9. Hai điều KHÔNG khẳng định được

Ghi lại để không lặp đúng lỗi mà báo cáo này phê phán:

1. **Flannel và MetalLB không có nguồn nào nêu đích danh k8s 1.35.** Cả hai chỉ công bố **sàn mở** (`v1.17+`, `>= 1.19.0-0`). Kết luận "tương thích" ở đây suy ra từ **vắng mặt ràng buộc trần + vắng mặt issue được báo cáo**, không phải từ khẳng định của dự án.
2. **metrics-server `1.34+` cũng là sàn mở**, không có bằng chứng conformance riêng cho 1.35.

---

## 5. Bước 2 — Vá lỗi chặn

### A1

**§8.7 dùng `helm` trước khi Helm được cài** — 📖

**Runbook viết** (dòng 821–823):

```bash
helm upgrade --install metrics-server metrics-server \
  --repo https://kubernetes-sigs.github.io/metrics-server/ \
  -n kube-system --set args={--kubelet-insecure-tls}
```

**Bằng chứng:** Lệnh cài Helm duy nhất trong runbook nằm ở **§9.3 dòng 991**:

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Tác động:** Người đọc tuần tự §8 → §8.7 gặp `helm: command not found`. Nặng hơn: **§8.8 mục 10** (dòng 863) lấy `kubectl top nodes` làm điều kiện PASS bắt buộc trước khi sang Rancher → người dùng bị chặn ở gate mà không hiểu vì sao.

**Sửa:** Chèn bước cài Helm vào đầu §8.7, hoặc dời cả §8.7 xuống sau §9.3.

---

### A2

**§8.7b `local-path-provisioner` — thiếu lệnh cài, và PVC test chắc chắn không `Bound`** — 📖

**Runbook viết** (dòng 835):

> Lab 1 master 2 worker thì `local-path-provisioner` (do Rancher làm) là lựa chọn gọn nhất. **Sau khi cài**, verify bằng một PVC thử — phải lên `Bound`

Rồi ngay sau đó (dòng 838–847) là PVC test.

**Hai lỗi chồng nhau:**

1. **Không có bước cài nào cả.** Cụm từ "Sau khi cài" trỏ tới một lệnh không tồn tại trong runbook.
2. **Kể cả khi cài đúng, PVC vẫn `Pending`.** Manifest `local-path-storage.yaml` upstream **không** gắn annotation `storageclass.kubernetes.io/is-default-class: "true"`. PVC ở dòng 838–845 **không khai `storageClassName`** → cần default StorageClass → không có → `Pending` vĩnh viễn.

**Bằng chứng nội tại:** chính runbook ở dòng 832 đã ghi `kubectl get storageclass   # cụm kubeadm thuần: RỖNG` — tức tác giả biết cụm không có SC nào, nhưng vẫn viết PVC không khai `storageClassName`.

**Sửa:**

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl -n local-path-storage rollout status deploy/local-path-provisioner
```

---

### A3

**§8.3 "test cross-node" KHÔNG đảm bảo cross-node** — 📖

Đây là lỗi nặng nhất về mặt hệ quả, vì §8.3 tự nhận (dòng 747):

> ⭐ Tầng **quan trọng nhất** và hay bị bỏ qua nhất. Cụm hỏng ở đây vẫn hiện `Ready` đủ 3 node.

**Runbook viết** (dòng 756–762):

```bash
kubectl create deploy nettest --image=nicolaka/netshoot --replicas=4 -- sleep 3600
...
A=$(kubectl get pod -l app=nettest -o jsonpath='{.items[0].metadata.name}')
BIP=$(kubectl get pod -l app=nettest -o jsonpath='{.items[3].status.podIP}')
kubectl exec $A -- ping -c3 $BIP             # PASS: 0% packet loss
```

**Bằng chứng:** `kubectl get pod` trả về danh sách sắp theo **tên**, không theo node. 4 replica rải trên 2 worker → `items[0]` và `items[3]` hoàn toàn có thể **nằm cùng một node**. Khi đó ping đi qua bridge nội bộ của node, **không** chạm tới VXLAN.

**Tác động:** VXLAN UDP 8472 giữa 2 node đang đứt mà test vẫn **PASS**. Đây đúng là kịch bản mà §8.3 (dòng 765) tuyên bố sẽ bắt được:

> ❌ **Fail ở đây** = VXLAN **UDP 8472** bị chặn giữa các node […] Đây là nguyên nhân số 1 của triệu chứng *"cụm nhìn Ready mà app không gọi được nhau"*.

Runbook có dòng 758 `kubectl get pods -l app=nettest -o wide  # xác nhận đã rải trên cả 2 worker` — nhưng đó là kiểm tra bằng mắt, không ràng buộc gì tới 2 dòng `jsonpath` phía sau.

**Sửa — ép chọn pod theo node:**

```bash
W1=$(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[0].metadata.name}')
W2=$(kubectl get nodes -l '!node-role.kubernetes.io/control-plane' -o jsonpath='{.items[1].metadata.name}')
A=$(kubectl get pod -l app=nettest --field-selector spec.nodeName=$W1 -o jsonpath='{.items[0].metadata.name}')
BIP=$(kubectl get pod -l app=nettest --field-selector spec.nodeName=$W2 -o jsonpath='{.items[0].status.podIP}')
echo "ping từ $A ($W1) → $BIP ($W2)"
kubectl exec $A -- ping -c3 $BIP
```

---

### A4

**§8.8 checklist mâu thuẫn thứ tự với chính nó** — 📖

**Runbook viết** (dòng 850): tiêu đề *"Gate cuối — checklist trước khi cài Rancher"*, đặt trong **§8**.

Nhưng bảng gate yêu cầu:

| Mục | Dòng | Yêu cầu                                                                      | Được cài ở  |
| ---- | ----- | ------------------------------------------------------------------------------ | ---------------- |
| 5    | 858   | `kubectl get pods,svc -n traefik` → pod Running + IngressClass DEFAULT=true | **§9.3**  |
| 6    | 859   | `kubectl get pods -n cert-manager` → 3 Running, ≥ 6 CRD                    | **§14.1** |

**Tác động:** Người đọc tuần tự chạy §8.8 sẽ fail 2/10 mục, cả hai đều là "không tìm thấy namespace", không có manh mối nào chỉ ra rằng đơn giản là *chưa tới lúc*.

**Sửa:** Tách bảng gate thành **§14.0 "Gate trước khi cài Rancher"**, hoặc thêm dòng cảnh báo đầu §8.8: *"Bảng này chạy SAU khi đã xong §9 và §14.1 — quay lại đây trước bước §14.2."*

---

### C1

**Toàn bộ lý do ghim k8s ở 1.34 đã sai** — 🌐

**Runbook viết** (dòng 88, §2.1):

> Pin ở **1.34** vì **Rancher 2.13.4 chỉ chứng nhận tới k8s 1.34** (1.35 chưa Rancher nào support)

Và lặp lại ở §2.1 ghi chú, §8.8 mục 1 (`Server ≤ v1.34`), §14.2 dòng 1349, §14.5 dòng 1391.

**Đối chiếu thực tế:**

| Claim                                      | Verdict            | Bằng chứng                                                                                                                                                                                                                                |
| ------------------------------------------ | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rancher 2.13.4 chứng nhận k8s 1.32–1.34 | ✅**ĐÚNG** | [Support matrix v2.13.4](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-13-4/) (last-changed 2026-07-07) liệt kê v1.32–v1.34 cho RKE2, K3s, EKS, AKS, GKE, ACK và imported cluster. 1.35 vắng mặt. |
| Rancher 2.13.4 là bản mới nhất         | ❌**SAI**    | Bản stable mới nhất là**2.14.3** (29/06/2026). Ngay trong nhánh 2.13 cũng đã có 2.13.5, 2.13.6, **2.13.7**.                                                                                                            |
| "1.35 chưa Rancher nào support"          | ❌**SAI**    | **Rancher 2.14.3 chứng nhận k8s v1.33–v1.35** — [support matrix v2.14.3](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-14-3/)                                                                 |

Nguồn bổ trợ: [github.com/rancher/rancher/releases](https://github.com/rancher/rancher/releases) · [endoflife.date/rancher](https://endoflife.date/rancher) · [SUSE support matrix index](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/)

**Vì sao đây là lỗi chặn chứ không phải chuyện cập nhật số version:** runbook bắt người đọc dựng **cụm mới toanh** trên k8s 1.34 (EOL 27/10/2026) + Rancher lạc hậu 1 minor + 3 patch, dựa trên một lý do **không còn đúng**. Dựng xong là phải nâng ngay — đúng việc mà runbook **không có hướng dẫn** (xem [B1](#b1)).

**Bẫy nâng cấp phải ghi rõ:** Rancher **2.14.3 bỏ hỗ trợ k8s 1.32** (sàn dời lên 1.33). Cửa sổ 2.13.4 (1.32–1.34) và 2.14.3 (1.33–1.35) **chỉ giao nhau ở 1.33–1.34**. Ai đã lỡ dựng theo runbook cũ phải nâng **Rancher trước**, k8s sau — đúng như §14.5 dòng 1391 đã dặn (điểm này runbook nói đúng).

**Ghi chú thêm:** Rancher **2.15 chưa phát hành** — GitHub chỉ có pre-release (v2.15.0-alpha21, 15/07/2026). Không có bản Rancher nào certify k8s 1.36.

**Sửa:** Rebase sang **Rancher 2.14.3 + k8s 1.35**.

---

### C2

**Ubuntu 24.04 giờ ship containerd 2.2.1 → lệnh `sed` ở §5.5 im lặng vô hiệu** — 🌐

#### Bối cảnh: vì sao cgroup driver quan trọng

Trên Linux, **kubelet** và **container runtime** đều thao tác với cgroup để giới hạn CPU/RAM của container. Cả hai **bắt buộc dùng cùng một cgroup driver**, chỉ có 2 lựa chọn:

| Driver       | Ai quản cgroup                                    |
| ------------ | -------------------------------------------------- |
| `cgroupfs` | Tiến trình tự ghi thẳng vào`/sys/fs/cgroup` |
| `systemd`  | Giao cho systemd quản lý                         |

Ubuntu 24.04 chạy **cgroup v2**, nơi systemd là "single writer" của cây cgroup. Nếu containerd dùng `cgroupfs` trong khi systemd cũng đang quản → hai bên tranh nhau ghi → node bất ổn, pod restart ngẫu nhiên, kubelet báo lỗi khó truy. Vì vậy `kubeadm` đặt kubelet ở `systemd`, và containerd **phải** được đặt tương ứng.

Đó là toàn bộ lý do tồn tại của §5.5.

#### Vấn đề

**Runbook viết** (§2.1 dòng ~89 và §5.5 dòng 429–443):

```bash
sudo apt-get install -y containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
...
grep SystemdCgroup /etc/containerd/config.toml       # dòng 443
```

**Đối chiếu thực tế:**

|                                | Runbook giả định                     | Thực tế trên noble-updates                                                                              |
| ------------------------------ | --------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Version                        | containerd**1.7.x**               | **2.2.1-0ubuntu1~24.04.3** (2026-06-22/25). Bước nhảy 1.7.28 → 2.2.1 landed **08/04/2026** |
| `config default` sinh        | `version = 2`                         | **`version = 3`**                                                                                  |
| Khoá CRI plugin               | `plugins."io.containerd.grpc.v1.cri"` | `plugins.'io.containerd.cri.v1.runtime'`                                                                 |
| Dòng`SystemdCgroup = false` | Có                                     | **KHÔNG CÒN**                                                                                      |

Nguồn: [packages.ubuntu.com/noble/containerd](https://packages.ubuntu.com/noble/containerd) · [launchpad noble containerd](https://launchpad.net/ubuntu/noble/+package/containerd) · [containerd 2.0 migration doc](https://github.com/containerd/containerd/blob/main/docs/containerd-2.0.md) · [containerd issue #12101](https://github.com/containerd/containerd/issues/12101)

**Tiền đề đã kiểm chứng — containerd KHÔNG cài sẵn trong Ubuntu:** manifest chính thức [`ubuntu-24.04.4-live-server-amd64.manifest`](https://releases.ubuntu.com/24.04/) (~680 gói trong live filesystem) **không chứa** `containerd`, `runc`, `docker.io`, lẫn `kubelet`/`kubeadm`/`kubectl`. → Bước `apt-get install -y containerd` ở §5.5 dòng 431 là **đúng và bắt buộc**. containerd có trong kho Ubuntu (section `admin`) nhưng không nằm trong bộ cài mặc định.

> ℹ️ Hai cách containerd "tự xuất hiện" mà người ta hay nhầm là cài sẵn: (a) cài `docker.io` từ kho Ubuntu → kéo containerd theo dạng dependency; (b) chọn snap **docker** ở màn hình installer → containerd nằm **bên trong snap**, không phải gói apt, và không dùng được cho kubeadm theo cách runbook mô tả.

**⚠️ Phạm vi áp dụng — C2 chỉ đúng trên amd64:** kho noble cấp **version khác nhau theo kiến trúc**:

| Kiến trúc               | containerd                       | `sed` ở §5.5                      |
| ------------------------- | -------------------------------- | ------------------------------------- |
| **amd64**           | **2.2.1-0ubuntu1~24.04.3** | ❌ vô hiệu →**C2 áp dụng** |
| arm64 và các arch khác | **1.7.12-0ubuntu4**        | ✅ vẫn chạy đúng                  |

Runbook chạy VMware trên host Windows → **amd64** → C2 áp dụng. Nhưng ai làm lab trên Apple Silicon (VMware Fusion/UTM, arm64) sẽ **không tái hiện được lỗi**. Hệ quả cho [§8.3](#83-phụ-lục--kiểm-chứng-c2-bằng-quan-sát-tuỳ-chọn): kịch bản kiểm chứng **phải chạy trên container amd64**, nếu không sẽ cho kết quả ngược.

**Cơ chế lỗi:** `SystemdCgroup` bị loại khỏi output của `config default` trong 2.x (regression từ commit 07c2ae1, fix merged tại PR #12244). `sed` không khớp gì → **exit code 0** → trông như thành công. Rồi `grep` ở dòng 443 **không in ra gì**, trong khi runbook chú thích `# verify SystemdCgroup = true`.

**Điểm mấu chốt:** `sed` không khớp gì **vẫn trả exit code 0**. Không có bất kỳ tín hiệu lỗi nào. Người làm thấy lệnh chạy trôi, `grep` im lặng, rồi đi tiếp.

**Ba chỗ bị ảnh hưởng cùng lúc:**

| Vị trí        | Lệnh                                                                         | Hậu quả                                                                              |
| --------------- | ----------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| §5.5 dòng 443 | `grep SystemdCgroup`                                                        | Bước verify ngay sau khi cấu hình →**luôn rỗng**                          |
| §8.0 dòng 668 | `grep SystemdCgroup`                                                        | Tầng 0 của quy trình verify 8 tầng →**luôn rỗng**                         |
| §15 dòng 1413 | *"Pod kẹt `ContainerCreating` → `grep SystemdCgroup` phải `true`"* | **Mục troubleshooting** → dẫn người đang gặp sự cố thật vào ngõ cụt |

Chỗ thứ ba tệ nhất: đúng lúc ai đó **thực sự** gặp lỗi cgroup, runbook bảo họ chạy một lệnh vĩnh viễn trả về rỗng. Họ sẽ kết luận sai và đi tìm nguyên nhân ở nơi khác.

**Sắc thái quan trọng — không thổi phồng:** containerd 2.x **tự dò systemd** khi không runtime nào khai `SystemdCgroup`, và containerd 2.0+ còn có cơ chế **bắt tay tự động** về cgroup driver với kubelet qua CRI. Nên cụm **nhiều khả năng vẫn lên bình thường**.

→ Nói gọn: **C2 không làm cụm sập, nó làm mù bộ đồ nghề chẩn đoán.**

*(Cách xác minh C2 bằng quan sát thực tế, không cần dựng VM: xem [§8.3](#83-phụ-lục--kiểm-chứng-c2-bằng-quan-sát-tuỳ-chọn).)*

#### Sửa

##### Chuẩn tham chiếu — tài liệu Kubernetes chính thức

Trang [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#containerd-systemd) quy định **đích đến**, và bản thân nó đã xác nhận việc tách đôi theo phiên bản — đúng điều [C2](#c2) phát hiện:

> **Configuring the `systemd` cgroup driver**
>
> To use the `systemd` cgroup driver in `/etc/containerd/config.toml` with `runc`, set the following config based on your Containerd version
>
> Containerd versions 1.x:
>
> ```toml
> [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
>   ...
>   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
>     SystemdCgroup = true
> ```
>
> Containerd versions 2.x:
>
> ```toml
> [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc]
>   ...
>   [plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.runc.options]
>     SystemdCgroup = true
> ```
>
> The `systemd` cgroup driver is recommended if you use **cgroup v2**.

Câu cuối xác nhận luôn phần *Bối cảnh* ở đầu C2: Ubuntu 24.04 chạy cgroup v2 → `systemd` là driver được khuyến nghị.

##### Một điều duy nhất tài liệu chuẩn KHÔNG nói

Tài liệu k8s mô tả **file phải trông như thế nào**. Điều nó không nói: với containerd 2.x, `config default` **không sinh ra** dòng `SystemdCgroup`.

Hệ quả — và đây là toàn bộ hạt nhân của C2:

> Bạn đang **chèn một dòng mới**, không phải **sửa một dòng có sẵn**.

Đó chính xác là lý do `sed 's/SystemdCgroup = false/.../'` của runbook chết: nó là lệnh **sửa**, dùng cho một dòng **không tồn tại**.

**Một chi tiết chưa xác minh, cần nói rõ:** bảng `[...runtimes.runc.options]` **có** xuất hiện trong output `config default` của 2.x hay không thì báo cáo này chưa kiểm chứng. Hai khả năng, thao tác lệch nhau chút:

| Nếu bảng**có sẵn** trong file             | Nếu bảng**không có**              |
| --------------------------------------------------- | ------------------------------------------- |
| Chèn`SystemdCgroup = true` vào trong bảng đó | Thêm cả dòng header bảng lẫn dòng key |

Bước *"mở file ra xem"* ở phần **Cách áp dụng** ngay dưới tự trả lời câu hỏi này, nên quy trình đúng trong cả hai trường hợp. [§8.3](#83-phụ-lục--kiểm-chứng-c2-bằng-quan-sát-tuỳ-chọn) mục 3 cũng cho biết ngay — đây là công dụng cụ thể của phụ lục đó.

##### Cách áp dụng

```bash
# 1) Xác định version → chọn khối config tương ứng trong tài liệu chuẩn ở trên
containerd --version

# 2) Sinh config mặc định
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

# 3) Mở file, tìm bảng runtimes.runc.options ỨNG VỚI VERSION của bạn,
#    thêm dòng  SystemdCgroup = true  vào TRONG bảng đó
sudo nano /etc/containerd/config.toml

# 4) Nạp lại
sudo systemctl restart containerd
```

Nếu muốn tự động hoá cho môi trường đã biết chắc là **containerd 2.x** (Ubuntu 24.04 amd64 — trường hợp của runbook này), nhắm đúng bảng đó:

```bash
sudo sed -i "/^\[plugins\.'io\.containerd\.cri\.v1\.runtime'\.containerd\.runtimes\.runc\.options\]/a\\    SystemdCgroup = true" /etc/containerd/config.toml
```

> ⚠️ **Không nên viết logic `if/else` dò version trong runbook.** Runbook này ghim cứng mọi thành phần khác (k8s `1.34.8`, Rancher `2.13.4`, Ubuntu 24.04 amd64), nên containerd cũng chỉ có **một** version đích. Tài liệu k8s cũng không dò gì — nó nói *"set the following config **based on your Containerd version**"*, tức người đọc nhìn version rồi chọn khối. Thêm nhánh điều kiện chỉ làm tăng bề mặt lỗi: pattern `sed` phải nới lỏng để khớp cả hai cú pháp, và pattern lỏng thì có thể chèn nhầm vào nhiều bảng runtime khác.

##### Verify — phải hỏi daemon, không đọc đĩa

Ba lệnh, ba mức tin cậy khác nhau. **Chỉ lệnh thứ ba trả lời đúng câu hỏi.**

| Lệnh | Cho biết điều gì |
| --- | --- |
| `grep SystemdCgroup /etc/containerd/config.toml` | Chỉ đúng thứ bạn vừa gõ vào file — **lệnh của runbook, mù** |
| `containerd config dump` | File + `imports` + merge defaults — **vẫn chỉ là đĩa** |
| **`crictl info`** | **Daemon đang chạy thực sự dùng driver nào** ✅ |

```bash
sudo crictl -r unix:///run/containerd/containerd.sock info | grep -i systemdcgroup
# kỳ vọng: "systemdCgroup": true
```

`crictl` có sẵn trên node theo §5.6 — gói `kubeadm` phụ thuộc `cri-tools`.

> ⚠️ **Đừng dùng `containerd config dump` cho việc này.** Nó **đọc file từ đĩa**, không hỏi daemon — [containerd issue #9417](https://github.com/containerd/containerd/issues/9417) ghi rõ: sửa file mà chưa `restart` thì `config dump` vẫn in giá trị **mới trong file** trong khi daemon đang chạy giá trị **cũ**. ([#11600](https://github.com/containerd/containerd/issues/11600) bổ sung: nó cũng không phản ánh đúng khi config được migrate từ version cũ.)
>
> Nguy hiểm hơn: nếu `SystemdCgroup` **không được khai tường minh** thì nhiều khả năng `config dump` **cũng không in ra dòng đó** — cùng lý do khiến `config default` bỏ nó ([#12101](https://github.com/containerd/containerd/issues/12101)). Tức dùng `config dump` thay `grep` là **đổi một bước verify mù lấy một bước verify có thể cũng mù**.

**Verify phía kubelet** (sau khi cụm chạy) — mới thực sự chứng minh **cả hai bên** đều ở `systemd`, đúng mục tiêu của §5.5:

```bash
kubectl get --raw "/api/v1/nodes/$(hostname)/proxy/configz" | grep -o '"cgroupDriver":"[^"]*"'
# kỳ vọng: "cgroupDriver":"systemd"
```

**Gotcha kèm theo cần ghi vào runbook:** [kubeadm issue #3146](https://github.com/kubernetes/kubeadm/issues/3146) — với containerd v2, kubeadm đọc sandbox image ra **chuỗi rỗng** và in cảnh báo `detected that the sandbox image "" of the container runtime is inconsistent with that used by kubeadm`. **Cosmetic, không fatal**, nhưng không báo trước thì người học sẽ hoảng.

---

### C7

**§9.3 Dashboard Traefik sai cả cổng lẫn cách bật** — 🌐

**Runbook viết** (dòng 1025–1032):

```bash
kubectl -n traefik port-forward deploy/traefik 9000:9000
# mở http://localhost:9000/dashboard/   ← BẮT BUỘC có dấu / ở cuối, thiếu là 404
```

**Hỏng 2 tầng:**

**Tầng 1 — sai cổng.** Entrypoint `traefik` (admin/ping) hiện là **8080**: `ports.traefik.port: 8080` (values.yaml dòng 912), `exposedPort: 8080` (dòng 930), `expose.default: false`. Cổng 9000 đúng tới chart **v32**; đổi tại **v33.0.0**. [Release notes v33.0.0](https://github.com/traefik/traefik-helm-chart/releases/tag/v33.0.0) ghi rõ:

> *"The default port of `traefik` entrypoint has changed from `9000` to `8080`."*

**Tầng 2 — dashboard mặc định KHÔNG được phục vụ.** `api.dashboard: true` **đã là mặc định** (values.yaml dòng 208) → `--set api.dashboard=true` là **no-op**, không phải cờ còn thiếu. Cờ thật sự thiếu là `ingressRoute.dashboard.enabled` (mặc định `false`, dòng 226), còn `api.insecure` cũng để trống (dòng 214). [EXAMPLES.md](https://github.com/traefik/traefik-helm-chart/blob/master/EXAMPLES.md#access-traefik-dashboard-without-exposing-it) nói thẳng:

> *"This Chart does not expose the Traefik local dashboard by default."*

→ Port-forward **đúng cổng** vẫn ra **404**.

**Tác động lan rộng:** runbook trỏ dashboard làm công cụ debug ở **3 chỗ** — §9.1 dòng 949, §10 dòng 1179, bảng lỗi dòng 1417. Cả ba đều dẫn vào ngõ cụt.

**Sửa:**

```bash
helm upgrade --install traefik traefik/traefik -n traefik --create-namespace \
  --set ingressRoute.dashboard.enabled=true

kubectl port-forward -n traefik \
  $(kubectl get pods --selector "app.kubernetes.io/name=traefik" -o name -n traefik) 8080:8080
# http://127.0.0.1:8080/dashboard/     ← dấu / cuối vẫn bắt buộc
```

**Tín hiệu hệ thống:** con số 9000 chứng tỏ runbook được viết dựa trên chart **cũ hơn v33**. Nên **rà lại toàn bộ §9** theo chart hiện hành (41.0.2 / Traefik v3.7.6), không chỉ vá đoạn dashboard.

---

### C8

**§9.4 `--set service.type=NodePort` sai key, Helm im lặng bỏ qua** — 🌐

**Runbook viết** (dòng 1107):

> 💡 Nếu thấy `<pending>` gây khó chịu […] có thể cài chart với `--set service.type=NodePort` để Service không bao giờ ở trạng thái chờ.

**Bằng chứng:** trong `values.yaml` của chart, key là **`service.spec.type`** (dòng 1135), không phải `service.type`.

**Tác động:** Helm `--set` với key không tồn tại **không báo lỗi**. Service vẫn là `LoadBalancer`, `EXTERNAL-IP` vẫn `<pending>`, người dùng tưởng đã đổi và đi tìm nguyên nhân ở chỗ khác.

**Sửa:** `--set service.spec.type=NodePort`

---

## 6. Bước 3 — Vá lỗi quan trọng

### C3

**§2.1 nói sai chính sách version-skew** — 🌐

**Runbook viết** (ghi chú ngay dưới bảng §2.1, ~dòng 96–98):

> ⚠️ **Version-skew:** `kubelet`/`kubeadm`/`kubectl` nên cùng minor (v1.34). `kubelet` được phép thấp hơn control plane **tối đa 1 minor**, không bao giờ cao hơn.

**Thực tế** — [version-skew-policy](https://kubernetes.io/docs/setup/release/version-skew-policy/):

> `kubelet` must not be newer than `kube-apiserver`. `kubelet` may be up to **three** minor versions older than `kube-apiserver` (`kubelet` < 1.25 may only be up to two minor versions older).

Nửa sau ("không bao giờ cao hơn") **đúng**. Nửa đầu sai — quy tắc 1-minor là thời **tiền-1.28**.

**Runbook KHÔNG bịa:** trang *Installing kubeadm* của kubernetes.io **đến giờ vẫn còn câu cũ sai đó** (*"One minor version skew between the kubelet and the control plane is supported…"*), mâu thuẫn với trang version-skew-policy. Đây là lỗi của tài liệu upstream, tác giả runbook chép lại trung thực.

**Sửa:** trích trang version-skew-policy, không trích install-kubeadm.

---

### C4

**§12.2 manifest `cloudflared` thiếu `--metrics` + `livenessProbe`** — 🌐

**Runbook viết** (dòng 1247):

```yaml
args: ["tunnel", "--no-autoupdate", "run"]
```

Không có probe nào trong toàn bộ manifest (dòng 1217–1254).

**Manifest official** ([Cloudflare Kubernetes deployment guide](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/deployment-guides/kubernetes/)):

```yaml
command: ["cloudflared", "tunnel", "--no-autoupdate", "--loglevel", "info",
          "--metrics", "0.0.0.0:2000", "run"]
livenessProbe:
  httpGet: { path: /ready, port: 2000 }
  failureThreshold: 1
  initialDelaySeconds: 10
  periodSeconds: 10
```

Docs: *"Cloudflared has a `/ready` endpoint which returns 200 if and only if it has an active connection."*

**Tác động:** cloudflared mất kết nối tới edge vẫn giữ process sống → pod báo `Running`, tunnel thực tế chết, k8s **không** restart. Với kiến trúc lấy tunnel làm **đường vào duy nhất**, đây là điểm mù vận hành nghiêm trọng.

**Phụ thuộc phải lưu ý:** probe chỉ hoạt động vì có `--metrics 0.0.0.0:2000`. Thêm probe mà quên `--metrics` → probe fail 100% → pod restart vô hạn.

**Đính chính:** official chỉ có **`livenessProbe`**, **không có `readinessProbe`**. Đừng thêm readiness.

**Tuỳ chọn bổ sung** (có trong official, runbook thiếu): `securityContext.sysctls` với `net.ipv4.ping_group_range: "65532 65532"` — cần nếu muốn ICMP/ping xuyên tunnel.

---

### C5

**§16 — 2 URL Cloudflare đã đổi đường dẫn** — 🌐

Cloudflare đã dời cây tài liệu từ `connections/connect-networks/` sang `networks/connectors/cloudflare-tunnel/`. Cả 3 URL đều còn resolve (301, không 404), nhưng 2 cái đã stale:

| §16 hiện tại                                                                    | Trạng thái            | Canonical mới                                                                                                                                        |
| ---------------------------------------------------------------------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/tunnel/setup/` (dòng 1449)                                                    | ✅ 200, không redirect | Giữ nguyên                                                                                                                                          |
| `.../connections/connect-networks/deployment-guides/kubernetes/` (dòng 1451)    | 301                     | `https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/deployment-guides/kubernetes/`                              |
| `.../connections/connect-networks/get-started/create-local-tunnel/` (dòng 1450) | 301 → 301 (2 hop)      | `https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/create-local-tunnel/` |

---

### C6

**§12.3 và §14.3 dùng tên UI Cloudflare đã bị đổi** — 🌐

**Runbook viết** (dòng 1265):

> Trong tunnel vừa tạo → tab **Published application routes** (hoặc **Public Hostname**) → **Add a public hostname**

**Thực tế:** "Public Hostname" là **tên cũ**, nay là **"Published application"**. Luồng hiện hành:

**Networking → Tunnels → chọn tunnel → tab `Routes` → Add route → Published application**

Runbook đoán đúng một nửa (có nhắc "Published application routes") nhưng tab giờ tên là **`Routes`**, và nav không còn đi qua Zero Trust. §12.1 dòng 1209 (*"Dashboard → Zero Trust → Networks → Tunnels"*) cũng thuộc luồng cũ.

Nguồn: [Add routes](https://developers.cloudflare.com/cloudflare-one/networks/routes/add-routes/) · [Create a remote tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/)

**Tác động:** với runbook mà giá trị nằm ở chỗ copy-paste/làm theo từng bước, tên UI sai là lỗi thật, không phải chuyện thẩm mỹ.

*(Lưu ý: docs Cloudflare đang giữa quá trình migrate, hai cách gọi vẫn cùng tồn tại ở các trang khác nhau.)*

---

### B1

**Thiếu hẳn quy trình `kubeadm upgrade` — và đây là việc gấp** — 📖

**Runbook nhắc đi nhắc lại nhưng không đưa lệnh nào:**

- §14.5 dòng 1391: *"khi **Rancher 2.14+** ra và chứng nhận k8s 1.35/1.36 → **`helm upgrade` Rancher TRƯỚC** […] **rồi** mới `kubeadm upgrade` lên minor kế tiếp (từng bậc 1.34→1.35→1.36)"*
- §15 dòng 1403: *"kubeadm **không skip được minor** (phải nhảy từng bậc) nên để càng lâu càng khó nâng"*

**Vấn đề:** không có một lệnh `kubeadm upgrade plan` / `kubeadm upgrade apply` nào trong toàn bộ 1557 dòng.

**Vì sao gấp:**

- k8s 1.34 **EOL 27/10/2026** (~3 tháng nữa) — runbook nói đúng ✅
- k8s 1.34 vào **maintenance mode 27/08/2026** (~5 tuần nữa), sau đó chỉ còn vá bảo mật — runbook **không nhắc**
- Rancher 2.14+ **đã ra rồi** (xem [C1](#c1)) — điều kiện mà §14.5 chờ đợi đã xảy ra

**Chi tiết bị bỏ quên:** §5.6 dòng 470 chạy `sudo apt-mark hold kubelet kubeadm kubectl`, nhưng **không đâu** trong runbook nhắc `apt-mark unhold` — bước bắt buộc đầu tiên của mọi lần nâng cấp.

---

### B2

**Rancher UI phơi ra Internet không có Cloudflare Access** — 📖

**Runbook** §14.3 (dòng 1358–1371) đưa `rancher.example.com` ra public qua tunnel, bảo vệ **chỉ bằng mật khẩu admin**.

**Vấn đề:** runbook dùng bộ **Zero Trust** để tạo tunnel nhưng **không hề bật Access policy** — thành phần duy nhất trong Zero Trust thực sự chặn người lạ trước khi họ chạm tới ứng dụng.

**Mâu thuẫn nội tại:** §9.2 dựng cả một chương lập luận bảo mật để loại ingress-nginx, với lý do (dòng 964):

> ingress controller nằm **ngay rìa cụm** và §12 chủ động đưa nó ra Internet. Đó chính xác là kịch bản *"unauthenticated RCE at the cluster's edge"*.

Rồi chính runbook đặt **UI quản trị toàn cụm** ra Internet trần trụi. Cũng lưu ý §9.3 dòng 1032 tự khen *"Dashboard truy cập qua `port-forward` nên **không phơi ra Internet** — đúng ý đồ bảo mật của runbook"* — tiêu chuẩn áp cho dashboard Traefik nhưng không áp cho Rancher.

**Sửa:** thêm một mục về Cloudflare Access policy (email OTP hoặc IdP) cho hostname `rancher.*`.

---

### B3

**§14.2 giải thích cơ chế TLS sai một nhịp** — 📖

**Runbook viết** (dòng 1352):

> `ingress.tls.source=rancher`: Rancher tự sinh cert self-signed qua cert-manager […] Rancher tạo sẵn một `Ingress` cho `rancher.example.com`, **Traefik terminate TLS bằng cert đó** ở cổng 443.

**Vì sao sai:** cloudflared đặt **SNI theo URL đích**, tức `traefik.traefik.svc.cluster.local` (§14.3 dòng 1367), **không** phải `rancher.example.com`. SNI này không khớp SAN của cert Rancher → Traefik trả **cert mặc định self-signed của chính nó**, rồi mới route theo Host header.

**Kết quả cuối vẫn chạy** — nhờ `No TLS Verify=ON` (dòng 1368) và `HTTP Host Header` (dòng 1369). Nhưng **lý do thì khác hẳn** với mô tả.

**Tác động:** ai debug theo giải thích này (ví dụ đi kiểm tra cert Rancher có đúng SAN không) sẽ đi lạc hoàn toàn. Đây là loại lỗi nguy hiểm hơn lỗi cú pháp: lệnh chạy đúng nhưng mô hình tư duy sai.

---

### B5

**RAM: khả thi trên giấy, rất chật trên thực tế** — 📖

**Runbook viết:** §2.3 *"RAM host nên ≥ 16 GB (3 VM × 4 GB + Windows)"*; §2.2 mỗi node 4 GB / 2 vCPU.

**Tính lại:** 3 × 4 GB = 12 GB + Windows 4–6 GB = **16–18 GB đã kịch trần**, chưa tính overhead VMware.

**Chồng thêm tải:** cụm cuối cùng phải chạy control plane + Flannel + CoreDNS + kube-proxy + **Traefik + metrics-server + cloudflared ×2 + cert-manager ×3 + Rancher**. Chính runbook ở §2.2 đã cảnh báo *"Rancher + cert-manager chiếm thêm tài nguyên"* và §8.2 dòng 732 ước tính *"~2–3 GB RAM"*.

**Rủi ro cộng dồn:** §8.2 dòng 742 lại gợi ý (tuỳ chọn) gỡ taint master:

```bash
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-
```

→ dồn workload vào master 2 vCPU đang chạy etcd + apiserver. Với cụm sẽ cài Rancher thì đây là lời khuyên **phản tác dụng**.

**Sửa:** nói thẳng **host 24 GB**, hoặc master 6 GB / worker 4 GB; và ghi rõ **không gỡ taint master** nếu định cài Rancher.

---

### B6

**§8.4–§8.6 phụ thuộc biến `$A` đặt từ §8.3** — 📖

Biến `A` được gán ở dòng 760 (§8.3), rồi dùng lại ở dòng 773, 776 (§8.4), 792 (§8.5), 810 (§8.6).

**Tác động:** SSH rớt hoặc làm ngắt quãng → `$A` rỗng → `kubectl exec  -- nslookup ...` báo lỗi khó hiểu (thiếu tên pod), dễ bị chẩn đoán nhầm thành lỗi DNS. Xác suất cao vì chính §5.2 dòng 372 cảnh báo phiên SSH **sẽ rớt** khi đổi IP.

**Sửa:** đặt lại `A=$(...)` ở đầu mỗi tiểu mục, hoặc thêm guard `[ -n "$A" ] || echo "chạy lại bước gán A ở §8.3"`.

---

### B7

**§8.5 `kubectl get endpoints` — API deprecated + sai chỗ cần nhìn** — 📖🌐

**Runbook viết** (dòng 789): `kubectl get endpoints web   # PASS: liệt kê 3 endpoint IP`

**Hai vấn đề:**

1. API `Endpoints` (core/v1) đã **deprecated**, thay bằng `EndpointSlice`.
2. **Sai chỗ cần nhìn khi debug Traefik.** RBAC của chart Traefik chỉ xin `discovery.k8s.io` → `endpointslices` (list, watch) — xem [`clusterrole.yaml`](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/templates/rbac/clusterrole.yaml). Core `endpoints` chỉ xuất hiện trong khối Traefik Hub thương mại. Tức Traefik **không đọc** cái mà runbook bảo đi xem.

**Sửa:** `kubectl get endpointslices -l kubernetes.io/service-name=web`

---

### B8

**§8.6 race giữa `port-forward &` và `curl`** — 📖

**Runbook viết** (dòng 811–813):

```bash
kubectl port-forward svc/web 18080:80 &
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:18080   # PASS: 200
kill %1
```

**Hai vấn đề:**

1. **Race:** `port-forward` chưa kịp bind cổng thì `curl` đã chạy → `000`/connection refused → **fail giả** ở chính tầng mà §8.6 nói là "Rancher UI sống chết ở đây".
2. **`kill %1` cần job control** — dán vào script không tương tác sẽ lỗi `no job control`.

**Sửa:**

```bash
kubectl port-forward svc/web 18080:80 >/dev/null 2>&1 &
PF=$!
for i in $(seq 1 10); do curl -sf -o /dev/null http://localhost:18080 && break; sleep 1; done
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:18080
kill $PF
```

---

### B9

**§4 mô tả clone không khớp thực tế VMware** — 📖

**Runbook viết** (dòng 227):

> Xong bước này bạn có 3 VM nhưng **cả 3 đang trùng hệt nhau** (cùng hostname/IP `.101`/`machine-id`/SSH key) — đúng hiện trạng bạn đang gặp.

Và dòng 275: *"Bản gốc hiện là `.101`; theo `servers.md` nó thành **`.100`**"*.

**Mâu thuẫn:** §4.1 bảo tạo VM gốc *"theo §3"*, mà **§3 dùng DHCP** khi cài (dòng ~176: *"Network: cứ để DHCP khi cài, ta sẽ đặt IP tĩnh sau"*). VMware full-clone **sinh MAC mới** cho mỗi bản clone → 3 máy DHCP sẽ ra **3 IP khác nhau**, không "trùng `.101`".

Kịch bản của runbook chỉ đúng nếu VM gốc **đã được đặt IP tĩnh `.101`** trước khi snapshot — điều không có trong §4.1.

**Tác động:** thấp về kỹ thuật (bước §4.5 vẫn đặt IP đúng), nhưng mạch kể lệch làm người đọc bối rối, đặc biệt câu *"đúng hiện trạng bạn đang gặp"* ám chỉ một tình huống cụ thể không được thiết lập.

**Sửa:** nói rõ ở §4.1 rằng VM gốc đặt IP tĩnh `.100` ngay từ đầu, và bỏ mô tả "trùng IP".

---

### B13

**§15 tự nhận là "Vận hành" nhưng không có backup** — 📖

Không có `etcdctl snapshot save`, không có Rancher Backup operator, không có hướng dẫn khôi phục. Phụ lục C (dòng 1547–1554) chỉ có `kubeadm reset` — tức **làm lại từ đầu**, không phải khôi phục.

Với cụm **phơi ra Internet qua tunnel** và có Rancher quản trị, đây là thiếu sót đáng kể hơn là "lab nên bỏ qua". Tối thiểu nên có:

```bash
kubectl -n kube-system exec etcd-k8s-master -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/lib/etcd-backup/snap-$(date +%F).db
```

---

### B14

**`servers.md` không tồn tại — mọi link ở §4 đều gãy** — 📖

**Runbook tham chiếu `servers.md` 8 lần**, bắt đầu ngay từ header đầu file (*"dựng 1 VM gốc → snapshot → full-clone ra 3 server […] theo [`servers.md`](servers.md)"*) và xuyên suốt §4 — kể cả tiêu đề chương: **"§4. Tạo và nhân bản 3 server theo servers.md"**.

**Bằng chứng:** liệt kê thư mục chứa runbook (`./`) — **không có file `servers.md`**. Toàn bộ 8 link `](servers.md)` trỏ vào hư không.

**Tác động:**

- §4 tự mô tả là *"theo `servers.md`"* nhưng người đọc **không có cách nào đối chiếu** bảng đích (dòng 200–202) với nguồn mà nó tuyên bố lấy ra.
- Bảng §4 chứa thông tin **chỉ có ý nghĩa trong ngữ cảnh `servers.md`** — đặc biệt cột Domain với `https://teleport-onpre.devopseduvn.live`, và hai hostname `load-balancer` / `teleport` **không được dùng ở bất kỳ chương nào khác** của runbook.
- Cộng với xung đột IP `.103` giữa Lab A (`teleport`) và Lab B (`k8s-worker2`) mà runbook tự cảnh báo ở đầu file, §4 hiện là **chương lạc lõng nhất**: nó phục vụ một inventory bên ngoài không kèm theo.

**Sửa — chọn 1 trong 3:**

1. **Kèm `servers.md`** vào cùng thư mục khi phát hành runbook.
2. **Nội hoá:** chuyển nội dung `servers.md` thành một bảng inline trong §4, bỏ hết link ngoài.
3. **Tách rời:** nếu Lab A không liên quan tới cụm k8s, đưa §4 sang file riêng — vừa gỡ luôn xung đột IP `.103`, vừa làm runbook chính liền mạch (§3 → §5 → §6…).

---

## 7. Bước 4 — Cải thiện

### D1

**Patch pin lạc hậu** — 🌐 — §2.1 dòng 88 + §5.6 dòng 467.

Runbook ghim `1.34.8` / `VER='1.34.8-1.1'`. Patch mới nhất của nhánh 1.34 là **1.34.9** (2026-06-09), deb string **`1.34.9-1.1`**.

`1.34.8-1.1` là version **có thật và cài được** (đã xác nhận trong repo index) — chỉ thiếu vòng vá CVE tháng 6.

Bối cảnh 3 nhánh còn support ([kubernetes.io/releases/patch-releases](https://kubernetes.io/releases/patch-releases/)):

| Minor          | Patch mới nhất | Maintenance mode     | EOL                  |
| -------------- | ---------------- | -------------------- | -------------------- |
| 1.36           | 1.36.2           | 2027-04-28           | 2027-06-28           |
| 1.35           | 1.35.6           | 2026-12-28           | 2027-02-28           |
| **1.34** | **1.34.9** | **2026-08-27** | **2026-10-27** |

---

### D2

**cert-manager "vd v1.16.x" đã cũ** — 🌐 — §2.1. Bản hiện hành là **1.20.x** (2 nhánh còn support: 1.20 và 1.19); artifacthub đã có 1.21.0. Cờ `--set crds.enabled=true` ở §14.1 dòng 1319 thì **đúng** với cert-manager ≥ 1.15.

---

### D3

**Thiếu cảnh báo `br_netfilter`** — 🌐 — §5.4.

Flannel **yêu cầu** module `br_netfilter`, và **từ kubeadm 1.30 preflight không còn kiểm tra module này**. Thiếu nó thì Flannel fail **không có tín hiệu rõ ràng** — đây là kiểu lỗi mà §8.3 định bắt. Runbook có `modprobe br_netfilter` ở §5.4 nhưng không giải thích hệ quả nếu module rớt sau reboot.

Nguồn: [flannel troubleshooting](https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md)

---

### D4

**`--set args={...}` thiếu quote** — 🌐 — §8.7 dòng 823.

Phải là `--set args="{--kubelet-insecure-tls}"`, không thì bash/PowerShell nuốt mất dấu ngoặc nhọn.

*(metrics-server bản mới nhất 0.9.x, support k8s 1.34+ — phù hợp. `--kubelet-insecure-tls` vẫn là workaround chuẩn cho kubeadm; upstream mô tả *"For testing purposes only"* → nên ghi rõ không dùng cho production.)*

---

### D5

**"đọc `Endpoints`" → thực tế EndpointSlices** — 🌐 — §9.1 dòng 943.

Kết luận "proxy thẳng tới Pod IP, không qua ClusterIP" thì **đúng** (`nativeLBByDefault: false`, values.yaml dòng 358). Chỉ cơ chế mô tả sai. Liên quan trực tiếp tới [B7](#b7).

---

### D6

**Tên IngressClass bám theo tên Helm release** — 🌐 — §9.1 dòng 947.

`ingressClass.name` mặc định là `.Values.ingressClass.name | default (include "traefik.fullname" .)` — tức **theo tên release**, không hardcode `traefik`. Chỉ ra `traefik` vì runbook cài bằng `helm ... install traefik`. Đổi tên release → `ingressClassName: traefik` ở §10 dòng 1151 gãy.

---

### D7

**Nói rõ 8000/8443 vs 80/443** — 🌐 — §9.3 bảng dòng 1006–1007.

Bảng ghi `ports.web → exposedPort 80`: **đúng**. Nhưng nên bổ sung: container lắng nghe **8000/8443** (unprivileged, không cần `NET_BIND_SERVICE`), **Service** publish **80/443** với `targetPort` 8000/8443. Đọc lướt dễ tưởng Traefik bind cổng 80 trong pod.

---

### D8

**`systemctl mask swap.target` phi chuẩn** — 📖 — §5.3 dòng 396.

Chạy được nhưng thừa: `swapoff -a` + comment `/etc/fstab` (đã có ở dòng 393–394) là đủ. Mask một target unit có thể gây tác dụng phụ với unit khác phụ thuộc nó. Cách chuẩn hơn cho `swap.img` của Ubuntu là `systemctl disable --now swap.img.swap`.

---

### D9

**Bước `strictARP` thừa** — 📖 — §17 Phụ lục B dòng 1512–1515.

MetalLB chỉ cần `strictARP: true` khi kube-proxy chạy chế độ **IPVS**. kubeadm mặc định là **iptables** → bước này không cần.

*(Version `v0.16.1` ở dòng 1518 đã verify là **có thật** ✅.)*

---

### D10

**Thiếu 2 giới hạn Cloudflare** — 🌐 — nên thêm vào §14/§15:

1. **Upload trần 100 MB trên Free plan** ([Error 413](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/): Free/Pro 100 MB, Business 200 MB, Enterprise 500+ MB). Traffic qua tunnel là **proxied** nên **có áp dụng**. Ảnh hưởng khi upload Helm chart lớn / backup qua UI Rancher.
2. **WebSocket idle timeout** — chỉnh được chỉ trên Enterprise. Rancher dùng WebSocket cho shell và log stream → **terminal để idle lâu sẽ bị rớt**, phải reconnect. Không phải lỗi cụm, nhưng không biết trước thì debug nhầm.

*(Bản thân WebSocket được hỗ trợ **đầy đủ trên mọi plan** — khẳng định ở §14.2 dòng 1354 là **đúng** ✅.)*

---

### D11

**DNS tự tạo chỉ đúng với zone Full Setup** — 🌐 — §12.3 dòng 1274.

Runbook viết: *"Cloudflare **tự tạo bản ghi DNS (CNAME)** […] Bạn **không phải** tự thêm DNS record."* — **đúng** với zone **Full Setup** (Cloudflare quản DNS, đúng luồng §11.2). Nhưng với zone **Partial (CNAME) Setup** thì phải tự tạo CNAME trỏ `<UUID>.cfargotunnel.com` tại nhà cung cấp DNS. Nên ghi rõ giả định.

---

### D12

**Lớp tương thích annotation mô tả quá mơ hồ** — 🌐 — §9.2 dòng 979.

Runbook viết: *"Có **lớp tương thích đọc được nhiều annotation của ingress-nginx**"* — **đúng** ✅, nhưng thiếu tên và thiếu cảnh báo.

Tên cụ thể: provider **`providers.kubernetesIngressNGINX`** (Traefik Proxy **v3.6.2+**), mặc định `enabled: false`, `controllerClass: k8s.io/ingress-nginx`, `ingressClass: nginx`.

**Footgun bị bỏ sót:** provider này discover Ingress **toàn cụm** theo mặc định → bật cùng lúc với `kubernetesIngress` sẽ **sinh router trùng**.

Giới hạn đã tài liệu hoá ([reference](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/ingress-nginx/#limitations)): chỉ round-robin (không EWMA/IP-hash), session affinity chỉ persistent-mode, không leader election, forward-auth khác semantics và không cache session, khác biệt về CORS/trailing-slash/retry, rate limiting là token-bucket (429) thay vì leaky-bucket (503), và **~30 annotation không hỗ trợ**.

---

### D13

**CVSS 9.2 trình bày như con số tuyệt đối** — 🌐 — §9.2 dòng 960.

Runbook ghi *"CVSS v4.0 **9.2 Critical**"* mà không dẫn nguồn. Thực tế **hai bên chấm khác nhau**:

| Nguồn                                     | Xếp hạng                          |
| ------------------------------------------ | ----------------------------------- |
| **nginx.org** (advisory chính chủ) | **Medium**                    |
| Akamai, Orca, NSFOCUS, SOC Prime, Axonius  | **CVSS v4.0 9.2 — Critical** |

Không phải runbook sai — hai bên giả định khác nhau về khả năng khai thác: nginx upstream xếp Medium vì **RCE chỉ đạt được khi ASLR bị bypass hoặc tắt** (mặc định thì hậu quả là crash worker → DoS); các hãng bảo mật chấm theo kịch bản xấu nhất.

**Sửa:** giữ 9.2 nhưng ghi rõ nguồn và nêu kèm xếp hạng Medium của nginx.org. Chi tiết đầy đủ ở [§11](#11-verify-92--chương-chính-xác-nhất-runbook).

*(Sai lệch nhỏ kèm theo: runbook ghi khai thác thực tế "từ 18/05/2026"; mốc VulnCheck ghi nhận đầu tiên là **16/05/2026** — 18/05 là ngày báo chí đăng.)*

---

## 8. Bước 5 — Thêm chương mới & kế hoạch

Sau khi đã đổi version ([§4](#4-bước-1--ma-trận-tương-thích--checklist-đổi-version)) và vá xong 3 nhóm lỗi ([§5](#5-bước-2--vá-lỗi-chặn) → [§7](#7-bước-4--cải-thiện)), còn lại là **viết mới**: những chương runbook chưa hề có.

> 💡 **Nên tạo file mới** (vd `vmware-k8s-lab-v2.md`) thay vì sửa đè lên `vmware-k8s-lab.md` — để đối chiếu được, và giữ bản gốc làm tham chiếu cho chính báo cáo này (mọi số dòng trong báo cáo trỏ tới **bản gốc**).

> **⚠️ Vì sao KHÔNG có giai đoạn "xác minh C2"**
>
> Bản trước của kế hoạch này có một giai đoạn đầu: chạy thí nghiệm chứng minh [C2](#c2) trước khi rebase. **Đã bỏ — đó là công đoạn thừa.**
>
> Lý do: **cách sửa không phụ thuộc vào việc chứng minh cơ chế lỗi.** Chỉ cần `containerd --version` để biết 1.x hay 2.x, rồi áp đúng khối config theo [tài liệu Kubernetes chính thức](#chuẩn-tham-chiếu--tài-liệu-kubernetes-chính-thức). Dù `sed` có im lặng vô hiệu hay không, hành động vẫn y hệt — một thí nghiệm mà **mọi kết quả đều dẫn tới cùng một hành động** thì không phải thí nghiệm.
>
> Thêm nữa, cả 4 mệnh đề của C2 **đều đã có nguồn sơ cấp**, không cần quan sát bổ sung:
>
> | Mệnh đề của C2                                                                    | Nguồn                                                                          |
> | ------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
> | Ubuntu 24.04 amd64 ship containerd**2.2.1**, không phải 1.7.x như §2.1 khai | [packages.ubuntu.com](https://packages.ubuntu.com/noble/containerd)              |
> | Đường dẫn plugin**đổi tên** giữa 1.x và 2.x                            | **Tài liệu k8s chính thức** — trích trong [C2](#c2)                  |
> | containerd**không cài sẵn** trong Ubuntu                                     | [Manifest live-server](https://releases.ubuntu.com/24.04/)                       |
> | `config default` của 2.x **bỏ dòng** `SystemdCgroup`                     | [containerd issue #12101](https://github.com/containerd/containerd/issues/12101) |
>
> Mệnh đề thứ 2 một mình đã đủ bác bỏ `sed`. Kịch bản kiểm chứng vẫn được giữ lại ở [§8.3](#83-phụ-lục--kiểm-chứng-c2-bằng-quan-sát-tuỳ-chọn) cho ai muốn thấy tận mắt, nhưng **không phải điều kiện tiên quyết để làm bất cứ việc gì**.

---

### 8.1. Bản đồ công việc — việc nào đụng chương nào

Bảng này là **góc nhìn theo chương của runbook**, bổ trợ cho checklist theo dòng ở [§4.8](#48-checklist-nâng-runbook-lên-135--2143--chỗ-nào-chưa-khớp). Dùng nó để ước lượng khối lượng và tránh sót chương.

| Việc                                          | Chương đụng tới            | Căn cứ                                                     |
| ---------------------------------------------- | ------------------------------- | ------------------------------------------------------------ |
| k8s**1.34 → 1.35**                      | §2.1, §2.2, §5.6, §6, §8.8 | [C1](#c1), [D1](#d1)                                           |
| Rancher**2.13.4 → 2.14.3**              | §2.1, §14.2, §14.5           | [C1](#c1)                                                     |
| containerd**1.7 → 2.x**                 | §5.5, §8.0, §15              | [C2](#c2) — *theo khối config chuẩn của tài liệu k8s* |
| chart Traefik**→ 41.x**                       | §9.1, §9.3, §9.4, §10       | [C7](#c7), [C8](#c8), [D5](#d5), [D6](#d6), [D7](#d7)             |
| Sửa 4 lỗi §8                                | §8.3, §8.7, §8.8             | [A1](#a1), [A2](#a2), [A3](#a3), [A4](#a4)                       |
| Cập nhật URL + tên UI Cloudflare            | §12.3, §14.3, §16            | [C5](#c5), [C6](#c6)                                           |
| Thêm probe cloudflared                        | §12.2                          | [C4](#c4)                                                     |
| **Thêm mục mới:** `kubeadm upgrade` | §15 (mới)                     | [B1](#b1)                                                     |
| **Thêm mục mới:** backup etcd         | §15 (mới)                     | [B13](#b13)                                                   |
| Xử lý`servers.md`                          | §4                             | [B14](#b14)                                                   |

**Lưu ý bắt buộc khi rebase version:** Rancher 2.14.3 **bỏ hỗ trợ k8s 1.32** (sàn dời lên 1.33). Nếu bản v2 nhắm 1.35 thì phải ghi rõ đường nâng cấp cho ai đang ở 2.13.4 — **nâng Rancher trước, k8s sau**, vì hai cửa sổ chỉ giao nhau ở 1.33–1.34.

---

### 8.2. Chương bảo mật cần viết mới

Gom các phát hiện bảo mật rải rác thành một chương riêng, thay vì để lẫn trong các chương kỹ thuật:

| Nội dung                                                                                                                                                                                  | Căn cứ         |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------- |
| **Cloudflare Access** cho `rancher.*` — hiện UI quản trị cụm phơi ra Internet chỉ có mật khẩu                                                                            | [B2](#b2)         |
| Soi lại**§5.8 root SSH** — Cách B (root + mật khẩu) kết hợp`ufw disable` ở §5.7 trên cụm có đường ra Internet                                                            | 📖 chưa soi kỹ |
| Nhất quán hoá lập trường bảo mật — §9.2 lập luận rất chặt để loại ingress-nginx, nhưng §14.3 lại phơi Rancher UI; §9.3 tự khen dashboard Traefik không ra Internet | [B2](#b2)         |
| Giới hạn Cloudflare cần biết trước: upload 100 MB, WebSocket idle timeout                                                                                                            | [D10](#d10)       |
| Bổ sung backup như một yêu cầu bảo mật, không phải tuỳ chọn                                                                                                                     | [B13](#b13)       |

---

### 8.3. Phụ lục — Kiểm chứng C2 bằng quan sát (TUỲ CHỌN)

> 🔸 **Không bắt buộc — không phải điều kiện tiên quyết của giai đoạn nào.** Xem hộp *"Vì sao KHÔNG có giai đoạn xác minh C2"* ở đầu §10: cách sửa không phụ thuộc kết quả của phụ lục này. Giữ lại cho ai muốn thấy tận mắt cơ chế lỗi, hoặc cần bằng chứng quan sát để thuyết phục người khác.

**Nguyên tắc:** `containerd config default` chỉ **in cấu hình mặc định ra stdout**. Nó không khởi động daemon, không đụng kernel, không cần cgroup, không cần privileged. Vì vậy **một container Ubuntu 24.04 là đủ** — không cần VMware, không cần cài OS, không cần dựng node.

> ⚠️ **Điều kiện:** Docker engine Linux phải đang chạy. Có thể chạy trên bất kỳ máy Linux/WSL/Docker Desktop nào.

#### Kịch bản

> ⚠️ **Bắt buộc `--platform linux/amd64`.** Kho noble cấp containerd **2.2.1 cho amd64** nhưng **1.7.12 cho arm64** (xem [C2](#c2)). Chạy trên máy ARM mà không ép platform sẽ nhận containerd 1.7 → kịch bản cho kết quả **ngược**, tưởng nhầm là C2 sai.

```bash
docker run --rm --platform linux/amd64 ubuntu:24.04 bash -c '
set -e
apt-get update -qq
apt-get install -y -qq containerd >/dev/null 2>&1

echo "=== 1. Version thực tế mà apt cài ==="
dpkg-query -W -f="apt package: \${Version}\n" containerd
containerd --version

echo ""
echo "=== 2. Config version (runbook giả định 2) ==="
containerd config default | grep -E "^version"

echo ""
echo "=== 3. Có dòng SystemdCgroup không? ==="
N=$(containerd config default | grep -c "SystemdCgroup" || true)
echo "số dòng chứa SystemdCgroup: $N"

echo ""
echo "=== 4. sandbox_image (đối chiếu pause của kubeadm) ==="
containerd config default | grep -i "sandbox_image" || echo "(không có)"

echo ""
echo "=== 5. MÔ PHỎNG CHÍNH XÁC LỆNH §5.5 ==="
containerd config default > /tmp/config.toml
sed -i "s/SystemdCgroup = false/SystemdCgroup = true/" /tmp/config.toml
echo "sed exit code: $?"
echo "--- kết quả grep (đúng lệnh dòng 443 của runbook) ---"
grep "SystemdCgroup" /tmp/config.toml || echo ">>> GREP RỖNG — C2 ĐƯỢC XÁC NHẬN <<<"
'
```

#### Cách đọc kết quả

| Mục                          | Nếu runbook đúng           | Nếu[C2](#c2) đúng                                       |
| ----------------------------- | ----------------------------- | --------------------------------------------------------- |
| 1. Version                    | `1.7.x`                     | **`2.2.1-0ubuntu1~24.04.x`**                      |
| 2. Config version             | `version = 2`               | **`version = 3`**                                 |
| 3. Số dòng`SystemdCgroup` | `≥ 1`                      | **`0`**                                           |
| 5. grep cuối                 | in ra`SystemdCgroup = true` | **`>>> GREP RỖNG — C2 ĐƯỢC XÁC NHẬN <<<`** |

Mục **3 và 5 là quyết định**. Mục 4 xác nhận lại bằng quan sát nhận định phụ đã nêu ở [§10.4](#104-ba-nhận-định-ban-đầu-đã-bị-bác-bỏ) (containerd 2.2 dùng `pause:3.10.1`, khớp kubeadm 1.34).

#### Ba điều kịch bản này KHÔNG chứng minh

Phải nói rõ giới hạn, nếu không lại rơi vào đúng lỗi mà báo cáo này đang phê phán:

1. **Không chứng minh cgroup driver cuối cùng trên node thật.** Container không chạy containerd daemon → không quan sát được cơ chế auto-detect. Muốn biết phải chạy `crictl info` trên node có daemon thật (xem [C2 — Verify](#c2)).
2. **Không chứng minh kubelet + containerd bắt tay thành công.** Việc đó cần cụm thật.
3. **Không phản ánh máy đã `apt upgrade` từ trước.** Image `ubuntu:24.04` là bản sạch; node đã dùng lâu có thể ở trạng thái khác.

→ Kịch bản xác minh **chính xác một điều**: lệnh `sed` ở §5.5 có vô hiệu hay không. Đó là hạt nhân của C2, và là thứ duy nhất cần chứng minh bằng quan sát.

---

## 9. Đánh giá theo chương

| Chương        | Tình trạng | Ghi chú                                                                                                                                                                                                        |
| --------------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| §1 Kiến trúc | ✅ Tốt      | Sơ đồ + lý do chọn tunnel đúng và rõ                                                                                                                                                                   |
| §2 Quy hoạch  | 🔴           | [C1](#c1) sai lý do ghim version; [C3](#c3) skew sai; [D1](#d1) [D2](#d2) [B5](#b5)                                                                                                                                 |
| §3 Tạo VM     | ✅ Ổn       | Phần Virtual Network Editor / Hyper-V rất thực dụng                                                                                                                                                         |
| §4 Clone       | 🟠           | [B14](#b14) `servers.md` không tồn tại → 8 link gãy; [B9](#b9) mạch kể lệch; trùng IP `.103` với §5 (runbook **tự cảnh báo** ở đầu file ✅)                                            |
| §5 OS chung    | 🔴           | [C2](#c2) **containerd 2.2.1 + sed vô hiệu** — nặng nhất chương này; [D3](#d3) [D8](#d8)                                                                                                             |
| §6 Init master | ✅ Sạch     | `--control-plane-endpoint`, `--apiserver-advertise-address`, `--pod-network-cidr` đều đúng                                                                                                            |
| §7 Join worker | ✅ Sạch     | Bước pre-check`/dev/tcp/k8s-master/6443` là ý hay                                                                                                                                                         |
| §8 Verify      | 🔴           | [A1](#a1) [A2](#a2) [A3](#a3) [A4](#a4) + [B6](#b6) [B7](#b7) [B8](#b8). **Ý tưởng rất tốt, thực thi có lỗ**                                                                                             |
| §9 Traefik     | 🔴           | [C7](#c7) dashboard hỏng; [C8](#c8) sai key. **Nhưng §9.1, §9.2, §9.4 xuất sắc** — riêng §9.2 đã verify 7/8 claim đúng tuyệt đối ([§11](#11-verify-92--chương-chính-xác-nhất-runbook)) |
| §10 App mẫu   | ✅ Sạch     | YAML đúng; test bằng Host header hợp lý                                                                                                                                                                    |
| §11 Domain     | ✅ Sạch     |                                                                                                                                                                                                                 |
| §12 Tunnel     | 🟠           | **Chính xác nhất runbook**; thiếu [C4](#c4), tên UI cũ [C6](#c6), [D11](#d11)                                                                                                                          |
| §13 Kiểm tra  | ✅ Sạch     |                                                                                                                                                                                                                 |
| §14 Rancher    | 🔴           | [C1](#c1) kéo theo; [B2](#b2) không có Access; [B3](#b3) giải thích TLS sai                                                                                                                                   |
| §15 Vận hành | 🟠           | [B1](#b1) thiếu upgrade; [B13](#b13) thiếu backup. Bảng lỗi thường gặp thì tốt                                                                                                                           |
| §16 Nguồn     | 🟠           | [C5](#c5) 2 URL Cloudflare stale                                                                                                                                                                                 |
| §17 Phụ lục  | ✅ Ổn       | [D9](#d9) strictARP thừa; MetalLB v0.16.1 đúng                                                                                                                                                                |

---

## 10. Những chỗ runbook làm ĐÚNG (đã verify)

### 10.1. Thiết kế & giải thích xuất sắc

| Chỗ                                                                 | Vì sao đáng giữ                                                                                                                                                                                       |
| -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **§8 chia 8 tầng verify**                                    | Nguyên tắc "tầng dưới fail thì tầng trên chắc chắn fail, đừng nhảy cóc" — đúng và**hiếm gặp** trong tài liệu K8s. Phân biệt rõ `STATUS=Ready` ≠ cụm chạy được app. |
| **§8.0 bắt `product_uuid`/`machine-id` trùng**          | Bẫy thật của full-clone: node thứ 2 join xong thì node thứ 1 biến mất. Rất ít tài liệu nhắc.                                                                                                 |
| **§9.4 giải thích `<pending>`**                           | Truy đúng cơ chế CCM, và quan hệ bao hàm`ClusterIP ⊂ NodePort ⊂ LoadBalancer` là **chính xác**. Kết luận "không phải lỗi cần sửa" đúng và đúng chỗ.                      |
| **§8.8 cảnh báo `cattle-cluster-agent`** (dòng 865–872) | Bẫy thật của kiến trúc tunnel: agent phải tự resolve hostname public từ**trong** cụm. Đa số người dựng sẽ dính.                                                                     |
| **§2.2.1 kiểm tra dải DHCP router**                         | Bước gần như mọi runbook bỏ qua, dẫn tới trùng IP và cụm chập chờn.                                                                                                                          |
| **Kiến trúc cloudflared in-cluster → Traefik ClusterIP**    | Đúng và gọn — bỏ được MetalLB/NodePort/LoadBalancer. §9.4 chứng minh tác giả**hiểu sâu**, không chép.                                                                              |

### 10.2. Claim kỹ thuật đã verify là ĐÚNG

**Kubernetes / kubeadm:**

- ✅ **§5.6 repo `pkgs.k8s.io`: đúng verbatim** — cả dòng `Release.key` lẫn dòng `deb [signed-by=...]` khớp chính xác [docs v1.34](https://v1-34.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- ✅ **k8s 1.34 EOL = 27/10/2026** — runbook nói đúng con số
- ✅ **§8.1 cert kubeadm 1 năm** — `CertificateValidityPeriod = 365 ngày` (leaf), CA 10 năm, xác nhận trong source `release-1.34`. `kubeadm certs check-expiration` là lệnh hiện hành. *(Bổ sung đáng giá: từ `v1beta4` có thể chỉnh `certificateValidityPeriod` trong `ClusterConfiguration` — hợp với cụm lab sống lâu.)*
- ✅ **§6.1 Flannel** — URL `releases/latest/download/kube-flannel.yml` còn dùng được, namespace `kube-flannel`, `net-conf.json` mặc định `10.244.0.0/16` khớp `--pod-network-cidr`. Bản mới nhất v0.28.7 (07/07/2026).

**Traefik:**

- ✅ **`ingressClass.isDefaultClass: true` LÀ mặc định** — đây là claim được soi kỹ nhất, và runbook **đúng**. values.yaml dòng 124–127, giống hệt ở tag v41.0.2; [`ingressclass.yaml`](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/templates/ingressclass.yaml) render `ingressclass.kubernetes.io/is-default-class: "true"`. `kubectl get ingressclass` thật sự hiện `DEFAULT=true` → gate §8.8 mục 5 hợp lệ.
- ✅ `providers.kubernetesIngress.enabled: true` mặc định (dòng 329)
- ✅ `service.spec.type: LoadBalancer` mặc định (dòng 1135) → §9.4 đúng
- ✅ Selector `app.kubernetes.io/name=traefik` cho `kubectl wait` (§9.3 dòng 1022)
- ✅ Repo `https://traefik.github.io/charts`
- ✅ **Traefik không tự redirect HTTP→HTTPS** (`redirections.entryPoint: {}` rỗng) → §10 dòng 1151 và §13 dòng 1299 đúng
- ✅ **Không có admission webhook** — webhook duy nhất trong chart là `hub-admission-controller.yaml`, gated sau `hub.token` (Traefik Hub thương mại)
- ✅ Traefik v3.x là dòng hiện hành (mới nhất v3.7.8; chart 41.0.2 pin appVersion v3.7.6)

**Cloudflare:**

- ✅ `TUNNEL_TOKEN` qua `secretKeyRef`, **không cần cờ `--token`**
- ✅ **`replicas: 2` + giải thích ở dòng 1237 chính xác 100%** — *"Replicas do not support traffic steering"*; request đi tới replica **gần nhất về địa lý**, fail thì chuyển. *(Bổ sung: trần **100 connection / 25 replica** mỗi tunnel, mỗi replica mở 4 connection; **đừng autoscale** vì scale-down làm đứt connection đang chạy.)*
- ✅ **"No TLS Verify" + "HTTP Host Header"** (§14.3) — tên nhãn **vẫn đúng nguyên văn**, map sang `noTLSVerify` / `httpHostHeader`
- ✅ **Catch-all `service: http_status:404`** (Phụ lục A) — **bắt buộc thật**: *"must always include a catch-all rule that concludes the file"*. Cấu trúc `tunnel:` / `credentials-file:` / `ingress:` + `originRequest.noTLSVerify` đều là key hiện hành
- ✅ **Lệnh apt cài cloudflared** (Phụ lục A dòng 1465–1470) **khớp verbatim** với [pkg.cloudflare.com](https://pkg.cloudflare.com/)
- ✅ `image: cloudflare/cloudflared:latest` — **official cũng dùng `:latest` không ghim**; không có resources requests/limits trong official → runbook **không lệch chuẩn**

**Rancher:**

- ✅ **Tên Helm flag đúng hết** — `hostname`, `bootstrapPassword`, `replicas`, `ingress.ingressClassName`, `ingress.tls.source`, `tls`. Riêng `ingress.ingressClassName` là **field thật** trong `chart/templates/ingress.yaml`, không phải annotation:
  ```
  {{- if .Values.ingress.ingressClassName }}
    ingressClassName: {{ .Values.ingress.ingressClassName }}
  {{- end }}
  ```
- ✅ **`replicas` mặc định = 3** — §14.2 dòng 1350 nói đúng
- ✅ **cert-manager vẫn bắt buộc** với `ingress.tls.source=rancher` — §14.1 đúng. Chart README: *"This step is only required to use certificates issued by Rancher's generated CA `(ingress.tls.source=rancher)`…"*
- ✅ **Lệnh lấy bootstrap password đúng** (dòng 1377–1378) — chỉ thiếu `{{ "\n" }}` cuối nên password dính liền dấu nhắc shell (vặt)
- ✅ **§16 URL guide migration Rancher có thật** — nội dung xác nhận *"upstream best-effort maintenance […] ended March 2026"* và **Rancher thực sự đẩy mạnh Traefik**: *"To support users during this transition, Rancher provides clear migration paths to Traefik."*
- ✅ **MetalLB v0.16.1** (§17 Phụ lục B dòng 1518) — có thật

### 10.3. Sắc thái cần thêm cho claim đúng

**Cài Rancher trên kubeadm** (§14): docs Rancher nói *"Rancher can be installed on any Kubernetes cluster. This cluster can use upstream Kubernetes…"* → **được tài liệu công nhận**. Nhưng chữ "kubeadm" **không xuất hiện ở đâu**, và support matrix **không certify** distro chung làm **host** cho Rancher Manager (chỉ K3s/RKE2/EKS/AKS/GKE; dòng "All Other Distros" chỉ nói về **import** cluster downstream).

→ Đọc đúng: **"supported-by-documentation nhưng not certified-by-matrix"**. Runbook nên nói rõ sắc thái này thay vì ngầm coi là setup chuẩn.

Hai yêu cầu cứng dễ bị bỏ sót trên kubeadm, cả hai runbook **đều đã đáp ứng**: (1) aggregation API layer phải cấu hình đúng; (2) *"The Kubernetes cluster that you install Rancher in must contain an Ingress controller"* — RKE2/K3s có sẵn, kubeadm **không** → §9 đã lo.

**"SUSE/Rancher chính thức chọn Traefik"** (§9.2 dòng 978): **hợp lý nhưng hơi quá lời**. Trang guide chỉ nêu tên Traefik và chỉ Traefik có migration guide, nhưng có hedge một câu: *"Users must migrate to Traefik **or another supported Ingress controller**"*. Ngoài ra tuỳ chọn **"Dual Mode"** trong guide **cần Rancher v2.14.0+** — không có trên 2.13.x mà runbook ghim.

### 10.4. Ba nhận định ban đầu đã bị bác bỏ

Ghi lại để minh bạch — đây là các điểm người review nêu ở vòng đọc đầu, sau đó **verify bác bỏ**:

| Nhận định ban đầu                                                          | Kết quả verify                                                                                                                                                                                                                                                                 |
| ------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "§5.5 thiếu xử lý`sandbox_image` (pause) lệch version"                   | ❌**Không đúng với containerd 2.2** — nó dùng `pause:3.10.1`, **khớp** `PauseVersion` của kubeadm 1.34. Mismatch chỉ tồn tại ở containerd 1.7 (`pause:3.8`). Vấn đề thật là [C2](#c2), khác hẳn.                                            |
| "`image: cloudflared:latest` là thiếu sót"                                 | ⚠️**Hạ mức** — manifest official của Cloudflare **cũng dùng `:latest`**. Không sai so với nguồn; chỉ **không nhất quán với triết lý ghim version của chính runbook** (§2.1 ghim tới tận patch). Giữ như góp ý, không phải lỗi. |
| "Manifest cloudflared thiếu`livenessProbe` **và** `readinessProbe`" | ⚠️**Đính chính** — official **chỉ có liveness**, không có readiness. Đừng thêm readiness. Thiếu sót thật vẫn còn ([C4](#c4)) nhưng phạm vi hẹp hơn.                                                                                             |

Ngoài ra, một agent verify báo rằng runbook truyền **đồng thời** `tls=external` và `ingress.tls.source=rancher` (hai cờ mâu thuẫn). **Kiểm lại dòng 1332–1356: không phải.** Runbook dùng `ingress.tls.source=rancher` trong lệnh chính, còn `tls=external` chỉ nêu ở ghi chú dòng 1356 như **phương án thay thế**. → **Runbook viết đúng**, agent đọc nhầm.

---

## 11. Verify §9.2 — chương chính xác nhất runbook

> **Cập nhật 2026-07-20 (vòng 2).** Toàn bộ 8 claim của §9.2 đã được xác minh. **Kết quả: 7/8 đúng hoàn toàn, 1 điểm có tranh chấp về mức độ nghiêm trọng.**
>
> ⚠️ **Ghi nhận sai sót của người review:** ở vòng đọc đầu, claim `CVE-2026-42945` bị đánh dấu **"nghi vấn cao — có thể là CVE bịa"**, với lý do "mức độ cụ thể bất thường là đặc trưng của nội dung sinh ra không nguồn". **Nhận định đó SAI.** CVE có thật, và chính mức độ cụ thể ấy là dấu hiệu tác giả runbook đã đọc nguồn gốc. Đây là bài học ngược: chi tiết dày đặc **không phải** chỉ dấu đáng tin cậy của nội dung bịa.

### 11.1. Kết quả từng claim

| # | Claim                                                                                                                                        | Vị trí         | Verdict                                                                             |
| - | -------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- | ----------------------------------------------------------------------------------- |
| 1 | ingress-nginx retired chính thức**03/2026**                                                                                          | dòng 959        | ✅**ĐÚNG**                                                                  |
| 2 | Bản cuối là**`v1.15.1`**                                                                                                                | dòng 959, 962   | ✅**ĐÚNG — chính xác tuyệt đối**                                      |
| 3 | `CVE-2026-42945` "NGINX Rift", heap overflow trong `ngx_http_rewrite_module`, RCE không cần xác thực, đang bị khai thác thực tế | dòng 960        | ✅**CÓ THẬT** — riêng mức CVSS có tranh chấp, xem mục 9.2 bên dưới |
| 4 | nginx upstream đã vá ở**1.30.1/1.31.0**                                                                                            | dòng 962        | ✅**ĐÚNG — khớp verbatim advisory**                                       |
| 5 | **InGate** "cũng đã bị khai tử"                                                                                                   | dòng 974        | ✅**ĐÚNG**                                                                  |
| 6 | ingress-nginx chạy ở**~50% cụm K8s**                                                                                                | dòng 953        | ✅**ĐÚNG**                                                                  |
| 7 | Tuyên bố Steering + SRC**01/2026**                                                                                                   | dòng 958        | ✅**ĐÚNG — trích dẫn khớp nguyên văn**                                |
| 8 | 2 URL blog kubernetes.io ở §16                                                                                                             | dòng 1445–1446 | ✅**Cả 2 còn sống**                                                        |

### Chi tiết bằng chứng

**Claim 1 & 2 — retirement và bản cuối.** [github.com/kubernetes/ingress-nginx/releases](https://github.com/kubernetes/ingress-nginx/releases):

- Release cuối cùng: **`controller-v1.15.1`**, phát hành **19/03/2026** (cùng ngày với `helm-chart-4.15.1`, `controller-v1.14.5`, `controller-v1.13.9`)
- Repo hiển thị: *"This repository was archived by the owner on **Mar 24, 2026**. It is now read-only."*

→ Runbook ghi *"03/2026 — Retired chính thức. Bản cuối `v1.15.1`"` — **đúng cả tháng lẫn số hiệu**.

**Claim 7 — tuyên bố 01/2026.** [Blog kubernetes.io 29/01/2026](https://kubernetes.io/blog/2026/01/29/ingress-nginx-statement/), tiêu đề *"Ingress NGINX: Statement from the Kubernetes Steering and Security Response Committees"* (tác giả Kat Cosgrove). Nguyên văn:

> *"Half of you will be affected. You have two months left to prepare."*

→ Runbook dịch: *"Một nửa trong số các bạn sẽ bị ảnh hưởng. Các bạn còn hai tháng để chuẩn bị."* — **dịch sát, không bịa**.

**Claim 6 — ~50%.** Cùng bài blog trên: *"a piece of critical infrastructure for about half of cloud native environments"*, dẫn nghiên cứu nội bộ của **Datadog**. ✅

**Claim 5 — InGate.** Xác nhận InGate không thu hút đủ đóng góp để trưởng thành và **cũng bị retire** cùng đợt 03/2026. → Bảng dòng 974 của runbook (*"Không kịp trưởng thành, cũng đã bị khai tử"*) **đúng**.

**Claim 3 & 4 — CVE.** Xác nhận qua nhiều nguồn độc lập gồm [nginx.org security advisories](https://nginx.org/en/security_advisories.html), [Akamai](https://www.akamai.com/blog/security-research/nginx-critical-heap-buffer-overflow-cve-2026-42945), [Orca Security](https://orca.security/resources/blog/nginx-rewrite-module-vulnerability-cve-2026-42945/), [Help Net Security](https://www.helpnetsecurity.com/2026/05/18/ngnix-vulnerability-exploited-cve-2026-42945/), [The Hacker News](https://thehackernews.com/2026/05/nginx-cve-2026-42945-exploited-in-wild.html), [NSFOCUS](https://nsfocusglobal.com/nginx-remote-code-execution-vulnerability-cve-2026-42945-notice/):

| Thuộc tính         | Xác nhận                                                                                                             |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Tên                 | **NGINX Rift** ✅                                                                                                |
| Loại                | Heap buffer overflow trong**`ngx_http_rewrite_module`** ✅                                                           |
| Kích hoạt          | directive`rewrite` / `if` / `set` với biểu thức không đặt tên (`$1`, `$2`) thay chuỗi sau dấu `?` |
| Tác động          | Unauthenticated RCE + DoS — 1 request HTTP thủ công là đủ ✅                                                     |
| Phạm vi             | nginx OSS**0.6.27 – 1.30.0**, NGINX Plus R32–R36, NGINX Instance Manager, F5 WAF, App Protect                  |
| Tuổi đời          | ~**18 năm** (2026 − 18 = **2008**) → khớp runbook *"tồn tại âm thầm từ 2008"* ✅                |
| Bản vá             | **1.30.1** hoặc **1.31.0** (Plus: R32 P6 / R36 P4) → **khớp verbatim** runbook dòng 962 ✅       |
| Khai thác thực tế | VulnCheck ghi nhận từ**16/05/2026**, 3 ngày sau khi PoC công khai                                            |

→ Ảnh hưởng tới ingress-nginx là **có thật, không suy diễn** — HeroDevs có bài riêng *"NGINX Rift Heap Buffer Overflow **Hits Ingress NGINX**"*.

### 11.2. Điểm duy nhất có tranh chấp: mức CVSS

**Runbook viết** (dòng 960): *"CVSS v4.0 **9.2 Critical**"*.

Đây là chỗ **duy nhất** trong §9.2 không thống nhất giữa các nguồn:

| Nguồn                                                   | Xếp hạng                          |
| -------------------------------------------------------- | ----------------------------------- |
| **nginx.org** (advisory chính chủ)               | **Medium**                    |
| Akamai, Orca, NSFOCUS, SOC Prime, Axonius, pentest-tools | **CVSS v4.0 9.2 — Critical** |

**Cách hiểu đúng:** không phải runbook sai, mà là **hai bên chấm điểm theo giả định khác nhau về khả năng khai thác**. nginx upstream xếp Medium vì RCE **chỉ đạt được khi ASLR bị bypass hoặc tắt** — mặc định thì hậu quả là crash worker (DoS). Các hãng bảo mật chấm 9.2 theo kịch bản xấu nhất.

**Khuyến nghị cho runbook:** giữ nguyên con số 9.2 nhưng **ghi rõ nguồn và nêu cả xếp hạng của nginx.org**, thay vì trình bày 9.2 như con số tuyệt đối. Đây là [D13](#d13).

**Sai lệch nhỏ thứ hai:** runbook ghi khai thác thực tế *"từ 18/05/2026"*; mốc ghi nhận đầu tiên của VulnCheck là **16/05/2026** (18/05 là ngày các bài báo đăng). Chênh 2 ngày, không ảnh hưởng lập luận.

### 11.3. Kết luận về §9.2

**§9.2 là chương được kiểm chứng kỹ nhất và chính xác nhất của toàn runbook.** Không cần viết lại gì. Lập luận kiến trúc — bỏ ingress-nginx vì nó nằm ở rìa cụm và §12 chủ động đưa nó ra Internet — **đứng vững hoàn toàn** trên nền sự kiện có thật.

Ba bảng phân loại của §9.2 (dòng 966–974) tách bạch **nginx (web server)** ✅ sống khỏe / **Ingress API** ✅ không deprecate / **`kubernetes/ingress-nginx`** ❌ retired / **`nginx/kubernetes-ingress`** (F5) ✅ dự án khác, vẫn maintain / **InGate** ❌ khai tử — đều **đúng**, và là cách trình bày rõ ràng hiếm thấy về một chủ đề rất hay bị gộp nhầm.

Duy chỉ có [D12](#d12) (mô tả lớp tương thích annotation quá mơ hồ, thiếu footgun router trùng) là đáng bổ sung.

---

## 12. Nguồn tham chiếu

**Kubernetes**

- [Patch releases &amp; EOL](https://kubernetes.io/releases/patch-releases/)
- [Version skew policy](https://kubernetes.io/docs/setup/release/version-skew-policy/)
- [Installing kubeadm (v1.34)](https://v1-34.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
- [Container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [kubeadm certs](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
- [kubeadm issue #3146 — sandbox image &#34;&#34; với containerd v2](https://github.com/kubernetes/kubeadm/issues/3146)

**containerd / Ubuntu**

- [packages.ubuntu.com/noble/containerd](https://packages.ubuntu.com/noble/containerd) — *version khác nhau theo kiến trúc: amd64 = 2.2.1, arm64 = 1.7.12*
- [Ubuntu 24.04 releases — live-server manifest](https://releases.ubuntu.com/24.04/) — *chứng minh containerd KHÔNG cài sẵn*
- [launchpad — noble containerd](https://launchpad.net/ubuntu/noble/+package/containerd)
- [containerd 2.0 migration guide](https://github.com/containerd/containerd/blob/main/docs/containerd-2.0.md)
- [containerd issue #12101 — SystemdCgroup mất khỏi config default](https://github.com/containerd/containerd/issues/12101)
- [containerd issue #9417 — `config dump` đọc file từ đĩa, không hỏi daemon](https://github.com/containerd/containerd/issues/9417)
- [containerd issue #11600 — `config dump` sai khi config migrate từ version cũ](https://github.com/containerd/containerd/issues/11600)
- [containerd discussion #5413 — cách kiểm tra containerd có dùng systemd cgroup](https://github.com/containerd/containerd/discussions/5413)

**Flannel / metrics-server / MetalLB**

- [flannel-io/flannel](https://github.com/flannel-io/flannel)
- [flannel troubleshooting](https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md)
- [metrics-server](https://github.com/kubernetes-sigs/metrics-server)
- [metallb/metallb releases](https://github.com/metallb/metallb/releases)

**Traefik**

- [traefik-helm-chart values.yaml](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/values.yaml)
- [traefik-helm-chart EXAMPLES.md](https://github.com/traefik/traefik-helm-chart/blob/master/EXAMPLES.md)
- [Release v33.0.0 — đổi cổng 9000 → 8080](https://github.com/traefik/traefik-helm-chart/releases/tag/v33.0.0)
- [templates/ingressclass.yaml](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/templates/ingressclass.yaml)
- [templates/rbac/clusterrole.yaml](https://github.com/traefik/traefik-helm-chart/blob/master/traefik/templates/rbac/clusterrole.yaml)
- [API &amp; dashboard reference](https://doc.traefik.io/traefik/reference/install-configuration/api-dashboard/)
- [kubernetesIngress provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress/)
- [ingress-nginx provider limitations](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/ingress-nginx/#limitations)

**Rancher / SUSE**

- [Support matrix v2.13.4](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-13-4/)
- [Support matrix v2.14.3](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-14-3/)
- [Support matrix index](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/)
- [rancher/rancher releases](https://github.com/rancher/rancher/releases)
- [endoflife.date/rancher](https://endoflife.date/rancher)
- [Install on a Kubernetes cluster](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/install-upgrade-on-a-kubernetes-cluster)
- [Installation requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/installation-requirements)
- [Guide to Ingress NGINX Retirement](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/kubernetes-resources-setup/load-balancer-and-ingress-controller/guide-to-ingress-nginx-retirement)
- [Bootstrap password](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/bootstrap-password)
- [chart values.yaml (release/v2.13)](https://raw.githubusercontent.com/rancher/rancher/release/v2.13/chart/values.yaml)

**Cloudflare**

- [Kubernetes deployment guide](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/deployment-guides/kubernetes/)
- [Tunnel availability &amp; failover](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-availability/)
- [Deploy cloudflared replicas](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/tunnel-availability/deploy-replicas/)
- [Origin parameters](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/origin-parameters/)
- [Configuration file](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/configuration-file/)
- [Add routes](https://developers.cloudflare.com/cloudflare-one/networks/routes/add-routes/)
- [Create a remote tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/)
- [Tunnels FAQ](https://developers.cloudflare.com/cloudflare-one/faq/cloudflare-tunnels-faq/)
- [WebSockets](https://developers.cloudflare.com/network/websockets/)
- [Error 413 — upload limits](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/4xx-client-error/error-413/)
- [Cloudflare Package Repository](https://pkg.cloudflare.com/)

**ingress-nginx / CVE-2026-42945**

- [Ingress NGINX Retirement: What You Need to Know (11/11/2025)](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Statement from Steering &amp; Security Response Committees (29/01/2026)](https://kubernetes.io/blog/2026/01/29/ingress-nginx-statement/)
- [kubernetes/ingress-nginx releases (archived 24/03/2026)](https://github.com/kubernetes/ingress-nginx/releases)
- [nginx.org security advisories](https://nginx.org/en/security_advisories.html)
- [Akamai — CVE-2026-42945 heap buffer overflow](https://www.akamai.com/blog/security-research/nginx-critical-heap-buffer-overflow-cve-2026-42945)
- [Orca Security — NGINX rewrite module flaw](https://orca.security/resources/blog/nginx-rewrite-module-vulnerability-cve-2026-42945/)
- [Help Net Security — exploited in the wild](https://www.helpnetsecurity.com/2026/05/18/ngnix-vulnerability-exploited-cve-2026-42945/)
- [The Hacker News — worker crashes, possible RCE](https://thehackernews.com/2026/05/nginx-cve-2026-42945-exploited-in-wild.html)
- [NSFOCUS — RCE notice](https://nsfocusglobal.com/nginx-remote-code-execution-vulnerability-cve-2026-42945-notice/)
- [HeroDevs — NGINX Rift hits Ingress NGINX](https://www.herodevs.com/blog-posts/cve-2026-42945-nginx-rift-heap-buffer-overflow-hits-ingress-nginx)

**cert-manager**

- [Supported releases](https://cert-manager.io/docs/releases/)

---

*Báo cáo lập 2026-07-20. **Vòng 2:** bổ sung [§11](#11-verify-92--chương-chính-xác-nhất-runbook) (verify §9.2) và [D13](#d13). **Vòng 3:** bổ sung [B14](#b14), mở rộng [C2](#c2) với bối cảnh cgroup driver, thêm [§8 Bước 5](#8-bước-5--thêm-chương-mới--kế-hoạch). **Vòng 4:** bổ sung bằng chứng containerd không cài sẵn + phạm vi amd64 vào [C2](#c2); rút §10 từ 3 giai đoạn xuống **2**, hạ bước "xác minh C2" thành [§8.3](#83-phụ-lục--kiểm-chứng-c2-bằng-quan-sát-tuỳ-chọn) tuỳ chọn. **Vòng 5:** bỏ logic `if/else` dò version trong cách áp dụng [C2](#c2), thay bằng quy trình bám sát tài liệu k8s chuẩn. **Vòng 6:** bỏ cảnh báo "duplicate table" gây nhiễu ở [C2](#c2); ghi rõ chi tiết **chưa xác minh** (bảng `runtimes.runc.options` có trong `config default` của 2.x hay không). **Vòng 7:** ⚠️ **sửa lỗi trong chính báo cáo** — `containerd config dump` **đọc file từ đĩa chứ không hỏi daemon** ([#9417](https://github.com/containerd/containerd/issues/9417)), nên không dùng để verify được; thay bằng `crictl info`. **Vòng 8:** thêm bảng **Đối chiếu version — runbook vs thực tế** (16 mục) vào §2. **Vòng 9:** thêm §11 **Ma trận tương thích** — verify toàn bộ đồ thị version cho bản rebase; phát hiện 2 lỗi trong khuyến nghị §8 (containerd, cert-manager) và 1 rủi ro chưa nêu (RBAC exec/WebSocket ở k8s 1.35). **Vòng 10:** thêm [4.8](#48-checklist-nâng-runbook-lên-135--2143--chỗ-nào-chưa-khớp) — checklist nâng runbook lên 1.35 + 2.14.3 theo từng dòng. **Vòng 11:** **sắp xếp lại toàn bộ file theo trình tự thi công** — phần đổi version từ cuối (§11) đưa lên **§4 Bước 1**; 3 nhóm lỗi thành Bước 2–4; các mục tham chiếu (đánh giá theo chương, chỗ runbook đúng, verify §9.2) dồn xuống cuối.*

***Không còn mục nào đang chờ verify.***

*Giới hạn còn lại: toàn bộ báo cáo là **đọc mã + tra tài liệu**, không có kiểm chứng thực nghiệm. Điều này **không làm giảm độ tin cậy của các phát hiện** — mọi mệnh đề chịu lực đều có nguồn sơ cấp (tài liệu chính chủ, issue upstream, manifest phát hành, kho gói). [§8.3](#83-phụ-lục--kiểm-chứng-c2-bằng-quan-sát-tuỳ-chọn) cung cấp thêm một quan sát trực tiếp cho C2 nếu cần, nhưng không phải điều kiện để áp dụng bất kỳ khuyến nghị nào.*
