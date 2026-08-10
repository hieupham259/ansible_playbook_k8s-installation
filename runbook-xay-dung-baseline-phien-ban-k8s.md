# Runbook xây dựng baseline phiên bản Kubernetes/Rancher

> **Mục tiêu duy nhất:** nghiên cứu, kiểm chứng và phát hành một bảng phiên bản tương tự mục **§2.1. Phiên bản** của runbook triển khai. Runbook này **không cài cluster** và không thay thế runbook cài đặt.
>
> **Nguyên tắc:** bảng phiên bản là kết quả của một lần nghiên cứu có ngày hết hạn, không phải danh sách version được chép lại vĩnh viễn.

---

## 0. Khởi tạo phiên nghiên cứu

### 0.1. Chạy ở đâu và không được làm gì

Chạy runbook trên một **research host Ubuntu cùng major release và architecture với node mục tiêu**. Host này có thể là VM dùng riêng; không chạy các lệnh `helm install`, `kubectl apply` hoặc thay đổi cluster production. Các lệnh trong tài liệu chỉ đọc nguồn official, đọc package index, tải metadata/chart/manifest vào thư mục evidence và render cục bộ.

Mọi block có `exit 1` đều phải nằm trong subshell `( ... )`. Khi gate fail, **STOP tại mục đó**; không tự sửa bảng thành `PASS`, không tiếp tục dựa trên phỏng đoán.

### 0.2. Gate công cụ và kết nối nguồn official

Các công cụ sau phải có sẵn trên research host. Runbook này không cài công cụ; nếu thiếu, chuẩn bị host theo quy trình quản trị phần mềm của tổ chức rồi chạy lại gate.

```bash
set -o pipefail
PRECHECK_LOG="$(mktemp)"
export PRECHECK_LOG

(
  for cmd in bash curl jq grep sed awk tar sha256sum python3 rg helm kubectl nano; do
    command -v "${cmd}" >/dev/null 2>&1 || {
      echo "FAIL: thiếu công cụ ${cmd}"; exit 1;
    }
  done

  bash --version | head -n1
  curl --version | head -n1
  jq --version
  python3 --version
  rg --version | head -n1
  helm version --short
  kubectl version --client

  for url in \
    'https://kubernetes.io/releases/' \
    'https://releases.rancher.com/server-charts/stable/index.yaml' \
    'https://cert-manager.io/docs/releases/' \
    'https://github.com/traefik/traefik-helm-chart/releases'; do
    curl -fsSL --connect-timeout 10 --max-time 30 \
      --range 0-0 -o /dev/null "${url}" || {
      echo "FAIL: không đọc được nguồn official ${url}"; exit 1;
    }
  done

  echo 'PASS: đủ công cụ nghiên cứu và đọc được các nguồn official bắt buộc'
) 2>&1 | tee "${PRECHECK_LOG}"
PRECHECK_RC=${PIPESTATUS[0]}
echo "precheck rc=${PRECHECK_RC}"
```

**PASS:** có dòng `PASS` và `precheck rc=0`.

**FAIL/STOP:** thiếu công cụ hoặc không đọc được một nguồn. Không chuyển sang §0.3 cho tới khi chạy lại đạt.

> `kubeadm` được kiểm riêng ở §4.5 vì chỉ cần sau khi đã chốt Kubernetes candidate. Công cụ đọc image digest được kiểm ở §7.5.

### 0.3. Tạo workspace và state file

Mỗi lần nghiên cứu dùng một thư mục mới. Không tái sử dụng evidence của baseline cũ.

```bash
export RESEARCH_DATE="$(date +%F)"
export BASELINE_ROOT="$(mktemp -d "${PWD}/version-baseline-${RESEARCH_DATE}-XXXXXX")"
export RESEARCH_RUN_ID="$(basename "${BASELINE_ROOT}")"

mkdir -p \
  "${BASELINE_ROOT}/evidence/00-preflight" \
  "${BASELINE_ROOT}/evidence/04-kubernetes" \
  "${BASELINE_ROOT}/evidence/05-runtime" \
  "${BASELINE_ROOT}/evidence/06-cni" \
  "${BASELINE_ROOT}/evidence/07-charts" \
  "${BASELINE_ROOT}/evidence/09-render" \
  "${BASELINE_ROOT}/evidence/12-final"

mv "${PRECHECK_LOG}" \
  "${BASELINE_ROOT}/evidence/00-preflight/gate-02-tools.txt"

touch \
  "${BASELINE_ROOT}/baseline-versions.md" \
  "${BASELINE_ROOT}/compatibility-matrix.md" \
  "${BASELINE_ROOT}/sources.md" \
  "${BASELINE_ROOT}/decision-log.md"

printf 'export RESEARCH_DATE=%q\nexport BASELINE_ROOT=%q\nexport K8S_EVIDENCE=%q\nexport RUNTIME_EVIDENCE=%q\nexport CNI_EVIDENCE=%q\nexport CHART_EVIDENCE=%q\nexport RENDER_EVIDENCE=%q\nexport FINAL_EVIDENCE=%q\n' \
  "${RESEARCH_DATE}" "${BASELINE_ROOT}" \
  "${BASELINE_ROOT}/evidence/04-kubernetes" \
  "${BASELINE_ROOT}/evidence/05-runtime" \
  "${BASELINE_ROOT}/evidence/06-cni" \
  "${BASELINE_ROOT}/evidence/07-charts" \
  "${BASELINE_ROOT}/evidence/09-render" \
  "${BASELINE_ROOT}/evidence/12-final" \
  > "${BASELINE_ROOT}/research-session.env"

cd "${BASELINE_ROOT}"
```

Gate workspace:

```bash
set -o pipefail
(
  test -d "${BASELINE_ROOT}/evidence/09-render" || {
    echo 'FAIL: thiếu cây evidence'; exit 1;
  }

  for file in baseline-versions.md compatibility-matrix.md sources.md decision-log.md; do
    test -f "${BASELINE_ROOT}/${file}" || {
      echo "FAIL: thiếu ${file}"; exit 1;
    }
  done

  test "${PWD}" = "${BASELINE_ROOT}" || {
    echo "FAIL: shell chưa đứng trong ${BASELINE_ROOT}"; exit 1;
  }

  echo "PASS: workspace mới sẵn sàng tại ${BASELINE_ROOT}"
) 2>&1 | tee "${BASELINE_ROOT}/evidence/00-preflight/gate-03-workspace.txt"
WORKSPACE_RC=${PIPESTATUS[0]}
echo "workspace gate rc=${WORKSPACE_RC}"
```

Khi mở phiên SSH mới, khôi phục state bằng:

```bash
source '/đường/dẫn/tới/version-baseline-YYYY-MM-DD-XXXXXX/research-session.env'
cd "${BASELINE_ROOT}"
```

**PASS:** `workspace gate rc=0`. **FAIL/STOP:** thiếu bất kỳ file/thư mục nào hoặc đang đứng sai thư mục.

### 0.4. Đầu ra bắt buộc

Mỗi lần chạy runbook này phải tạo đủ các đầu ra:

1. `baseline-versions.md`: bảng phiên bản cuối cùng để đưa vào runbook triển khai.
2. `baseline-detailed.md` và `baseline-data.tsv`: source of truth chi tiết và dữ liệu máy kiểm được.
3. `compatibility-matrix.md`/`.tsv`: giao của các dải tương thích và lý do chọn/loại từng candidate.
4. `sources.md`/`.tsv`: URL official, ngày truy cập, claim và evidence dùng để ra quyết định.
5. `decision-log.md`, `config-contract.tsv`, `resource-ownership.tsv` và `baseline-metadata.env`.
6. `evidence/`: metadata chart, values, manifest đã render, lab report, checksum và kết quả mọi gate.

Baseline chỉ được gắn trạng thái `APPROVED` khi tất cả gate trong tài liệu này đạt. Nếu chưa đủ bằng chứng, dùng `DRAFT`, `BLOCKED` hoặc `LAB-ONLY`; không dùng từ “supported” theo suy đoán. Tất cả dòng component khởi tạo ở trạng thái `NOT-TESTED`, không điền sẵn `PASS`.

---

## 1. Các khái niệm không được đánh đồng

### 1.1. “Mới”, “stable”, “supported” và “work được”

| Khái niệm | Nghĩa dùng trong runbook |
| --- | --- |
| Latest release | Release mới nhất upstream đã phát hành; có thể là RC, channel thử nghiệm hoặc chưa được thành phần khác hỗ trợ. |
| Stable channel | Channel được vendor chỉ định cho production-like usage, ví dụ `rancher-stable`; vẫn phải kiểm tra support matrix và release notes. |
| Maintained | Nhánh còn nhận bản vá lỗi/bảo mật theo lifecycle upstream. |
| Supported | Vendor tuyên bố hỗ trợ đúng tổ hợp sản phẩm, phiên bản, distro và vai trò đang xét. |
| Tested | Upstream chạy test định kỳ trên version đó. `tested` có thể hẹp hơn hoặc khác `supported`. |
| Metadata-compatible | Chart/package cho phép version đó, ví dụ `Chart.yaml.kubeVersion`; đây chưa phải chứng nhận của vendor. |
| Render-compatible | Template render đúng với cấu hình dự kiến; chưa chứng minh controller chạy đúng lúc runtime. |
| Lab-validated | Tổ hợp đã qua test trên môi trường thử tương đương; vẫn không đồng nghĩa vendor-certified. |

Một baseline “stable” phải đồng thời:

- không dùng alpha, beta, RC hoặc prerelease;
- nằm trên nhánh còn được duy trì;
- nằm trong giao support của các thành phần bắt buộc;
- dùng patch release được upstream còn hỗ trợ;
- không có cảnh báo breaking change/CVE/regression chưa được xử lý trong release notes;
- pin được artifact cụ thể, không để `latest`, `stable`, `master` hoặc version range trong bảng cuối;
- render đúng và vượt qua các gate tĩnh;
- ghi rõ cấp cam kết: `VENDOR-SUPPORTED`, `TECHNICALLY-COMPATIBLE` hay `LAB-ONLY`.

### 1.2. Thứ tự ưu tiên bằng chứng

Khi nguồn mâu thuẫn, dùng thứ tự sau và ghi mâu thuẫn vào decision log:

1. Support/certification matrix đúng version của vendor.
2. Lifecycle, release policy, security advisory và release notes chính thức.
3. Metadata của đúng artifact: `Chart.yaml`, package index, image manifest.
4. `values.yaml`, schema, template và CRD của đúng chart version.
5. Kết quả render/dry-run.
6. Kết quả test trên lab dùng topology tương đương.

Blog, diễn đàn, issue và câu trả lời cộng đồng chỉ dùng để tìm hướng điều tra. Chúng không thay thế nguồn official ở các bước 1–4.

> **Quy tắc quan trọng:** trang “Helm chart options” có thể không liệt kê mọi key mới. `values.yaml` và template của **đúng chart version** mới là hợp đồng thực thi cần audit.

### 1.3. Phân biệt các loại version

Không gộp các trường sau thành một cột mơ hồ:

| Loại | Ví dụ | Cách pin |
| --- | --- | --- |
| Product/app version | Rancher `2.x.y`, Traefik Proxy `3.x.y` | Exact SemVer |
| Helm chart version | Traefik chart `4x.y.z` | Exact chart version |
| Debian package version | kubeadm `1.xx.y-1.1` | Exact package string |
| Container image | `repo/name:vX.Y.Z` | Exact tag; production nên lưu thêm digest |
| Kubernetes minor | `1.xx` | Dùng để chọn repo và xét support window |
| Kubernetes patch | `1.xx.y` | Version thực tế của control plane/node tools |
| Git source | tag/release/commit | Tag immutable hoặc commit SHA, không dùng branch động |

---

## 2. Khai báo và kiểm chứng yêu cầu

### 2.1. Tạo input contract

Tạo `baseline.env`; file này không chứa mật khẩu/token:

```bash
tee "${BASELINE_ROOT}/baseline.env" >/dev/null <<'EOF'
TARGET_INSTALL_DATE=YYYY-MM-DD
ENVIRONMENT=homelab
SUPPORT_GOAL=technically-compatible
ARCHITECTURE=amd64
OS_ID=ubuntu
OS_VERSION=24.04
CLUSTER_BOOTSTRAP=kubeadm
RANCHER_REQUIRED=true
RANCHER_ROLE=manager-host
POD_CIDR=10.244.0.0/16
SERVICE_CIDR=10.96.0.0/12
LAN_CIDR=192.168.100.0/24
CNI=flannel
INGRESS_CONTROLLER=traefik
RANCHER_EXPOSURE=ingress
GATEWAY_API_INSTALLED=false
TLS_CONTROLLER=cert-manager
OPTIONAL_ADDONS=local-path-provisioner,metallb,cloudflared
EOF

nano "${BASELINE_ROOT}/baseline.env"
```

Thay `TARGET_INSTALL_DATE` và mọi giá trị khác theo hệ thống thực. Nếu có nhiều dải LAN/VPN, bổ sung chúng vào kiểm tra overlap hoặc ghi trong `decision-log.md`.

### 2.2. Gate cú pháp, enum và CIDR

```bash
set -o pipefail
set -a
source "${BASELINE_ROOT}/baseline.env"
set +a

(
  required_vars=(
    TARGET_INSTALL_DATE ENVIRONMENT SUPPORT_GOAL ARCHITECTURE OS_ID OS_VERSION
    CLUSTER_BOOTSTRAP RANCHER_REQUIRED RANCHER_ROLE POD_CIDR SERVICE_CIDR
    LAN_CIDR CNI INGRESS_CONTROLLER RANCHER_EXPOSURE GATEWAY_API_INSTALLED
    TLS_CONTROLLER OPTIONAL_ADDONS
  )

  for name in "${required_vars[@]}"; do
    test -n "${!name:-}" || {
      echo "FAIL: biến ${name} đang rỗng"; exit 1;
    }
    [[ "${!name}" != *'YYYY'* ]] || {
      echo "FAIL: biến ${name} còn placeholder ${!name}"; exit 1;
    }
  done

  date -d "${TARGET_INSTALL_DATE}" '+%F' >/dev/null 2>&1 || {
    echo 'FAIL: TARGET_INSTALL_DATE không đúng YYYY-MM-DD'; exit 1;
  }

  [[ "${ENVIRONMENT}" =~ ^(homelab|staging|production)$ ]] || {
    echo 'FAIL: ENVIRONMENT không hợp lệ'; exit 1;
  }
  [[ "${SUPPORT_GOAL}" =~ ^(vendor-supported|technically-compatible|lab-only)$ ]] || {
    echo 'FAIL: SUPPORT_GOAL không hợp lệ'; exit 1;
  }
  [[ "${ARCHITECTURE}" =~ ^(amd64|arm64)$ ]] || {
    echo 'FAIL: ARCHITECTURE không hợp lệ'; exit 1;
  }
  [[ "${CLUSTER_BOOTSTRAP}" =~ ^(kubeadm|rke2|k3s|managed-kubernetes)$ ]] || {
    echo 'FAIL: CLUSTER_BOOTSTRAP không hợp lệ'; exit 1;
  }
  [[ "${RANCHER_REQUIRED}" =~ ^(true|false)$ ]] || {
    echo 'FAIL: RANCHER_REQUIRED phải là true/false'; exit 1;
  }
  [[ "${RANCHER_ROLE}" =~ ^(manager-host|downstream-imported|both|none)$ ]] || {
    echo 'FAIL: RANCHER_ROLE không hợp lệ'; exit 1;
  }
  [[ "${RANCHER_EXPOSURE}" =~ ^(ingress|gateway|none)$ ]] || {
    echo 'FAIL: RANCHER_EXPOSURE không hợp lệ'; exit 1;
  }

  if [[ "${RANCHER_REQUIRED}" == true && "${RANCHER_ROLE}" == none ]]; then
    echo 'FAIL: Rancher required nhưng role=none'; exit 1
  fi
  if [[ "${RANCHER_EXPOSURE}" == gateway && "${GATEWAY_API_INSTALLED}" != true ]]; then
    echo 'FAIL: chọn Gateway nhưng chưa khai Gateway API'; exit 1
  fi
  if [[ "${CLUSTER_BOOTSTRAP}" != kubeadm || "${RANCHER_REQUIRED}" != true || \
        "${CNI}" != flannel || "${INGRESS_CONTROLLER}" != traefik || \
        "${RANCHER_EXPOSURE}" != ingress || "${GATEWAY_API_INSTALLED}" != false || \
        "${TLS_CONTROLLER}" != cert-manager ]]; then
    echo 'FAIL: runbook này chỉ bao phủ baseline kubeadm + Flannel + Traefik Ingress + cert-manager + Rancher'; exit 1
  fi

  python3 - <<'PY'
import ipaddress
import os
import sys

names = ('POD_CIDR', 'SERVICE_CIDR', 'LAN_CIDR')
nets = {}
for name in names:
    try:
        nets[name] = ipaddress.ip_network(os.environ[name], strict=False)
    except ValueError as exc:
        print(f'FAIL: {name} không hợp lệ: {exc}')
        sys.exit(1)

for index, left in enumerate(names):
    for right in names[index + 1:]:
        if nets[left].overlaps(nets[right]):
            print(f'FAIL: {left}={nets[left]} overlap {right}={nets[right]}')
            sys.exit(1)

print('PASS: ba CIDR hợp lệ và không overlap')
PY

  echo 'PASS: input contract đầy đủ, enum hợp lệ và không mâu thuẫn'
) 2>&1 | tee "${BASELINE_ROOT}/evidence/00-preflight/gate-02-input.txt"
INPUT_RC=${PIPESTATUS[0]}
echo "input gate rc=${INPUT_RC}"
```

**PASS:** hai dòng `PASS` và `input gate rc=0`.

**FAIL/STOP:** sửa `baseline.env`, source lại file và chạy lại toàn bộ §2.2. Chỉ sang §3 khi gate này đạt.

---

## 3. Xây dependency graph và chọn “anchor”

### 3.1. Thứ tự nghiên cứu chuẩn

```text
Topology + support goal + OS/architecture
                  |
                  v
Rancher stable candidate (nếu Rancher là bắt buộc)
                  |
                  v
Giao: Rancher support matrix ∩ Kubernetes còn maintained
                  |
                  v
Kubernetes exact patch + kubeadm/kubelet/kubectl package
                  |
                  v
Container runtime + CNI
                  |
                  v
Helm + cert-manager + ingress controller
                  |
                  v
Storage / LoadBalancer / tunnel / addon khác
                  |
                  v
Audit chart + render gate + bảng baseline
```

Nếu Rancher là thành phần bắt buộc, bắt đầu từ Rancher stable vì cửa sổ Kubernetes do Rancher hỗ trợ thường hẹp hơn cửa sổ upstream Kubernetes. Nếu không cần Rancher, Kubernetes maintained minor có thể là anchor.

### 3.2. Rancher Manager host khác downstream/imported

Trong [SUSE Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/), luôn đọc riêng:

- **Supported Kubernetes Platforms for Rancher Manager**: cluster thực sự chạy Rancher Server.
- **Downstream Cluster Support**: cluster Rancher tạo hoặc quản lý.
- **All Other Distros / Imported**: cluster có sẵn được import vào Rancher.

Không dùng dòng “Any” của imported cluster để tuyên bố kubeadm là một distro được chứng nhận làm Rancher Manager host. Nếu cài Rancher trực tiếp lên kubeadm cho homelab, có thể chốt `TECHNICALLY-COMPATIBLE` sau render/lab gate, nhưng không được ghi `VENDOR-SUPPORTED` nếu support matrix không chứng nhận topology đó.

Rancher khuyến nghị `rancher-latest` cho thử nghiệm và `rancher-stable` cho production; xem [Choosing a Rancher Version](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/choose-a-rancher-version). Stable channel chỉ tạo candidate, không miễn bước kiểm tra support matrix và release notes.

### 3.3. Gate chốt anchor và cấp cam kết dự kiến

```bash
set -a
source "${BASELINE_ROOT}/baseline.env"
set +a

set -o pipefail
(
  if [[ "${RANCHER_REQUIRED}" == true ]]; then
    ANCHOR_COMPONENT=rancher
  else
    ANCHOR_COMPONENT=kubernetes
  fi

  case "${SUPPORT_GOAL}:${RANCHER_ROLE}:${CLUSTER_BOOTSTRAP}" in
    vendor-supported:manager-host:kubeadm|vendor-supported:both:kubeadm)
      echo 'NOTICE: kubeadm Manager host chỉ được giữ nếu support matrix exact Rancher version chứng nhận topology này.'
      echo 'NOTICE: dòng All Other Distros/Imported không đủ làm bằng chứng Manager host.'
      ;;
  esac

  printf 'ANCHOR_COMPONENT=%s\n' "${ANCHOR_COMPONENT}" \
    > "${BASELINE_ROOT}/research-state.env"

  printf '| %s | %s | %s | %s | %s | %s |\n' \
    "${RESEARCH_DATE}" 'Anchor' "${ANCHOR_COMPONENT}" 'KEEP' \
    "Rancher required=${RANCHER_REQUIRED}; support goal=${SUPPORT_GOAL}" 'khi topology đổi' \
    >> "${BASELINE_ROOT}/decision-log.md"

  grep -Eq '^ANCHOR_COMPONENT=(rancher|kubernetes)$' \
    "${BASELINE_ROOT}/research-state.env" || {
    echo 'FAIL: không ghi được anchor'; exit 1;
  }

  echo "PASS: anchor=${ANCHOR_COMPONENT}; chuyển sang lập giao version"
) 2>&1 | tee "${BASELINE_ROOT}/evidence/00-preflight/gate-33-anchor.txt"
ANCHOR_RC=${PIPESTATUS[0]}
echo "anchor gate rc=${ANCHOR_RC}"
```

**PASS:** `anchor gate rc=0`. **FAIL/STOP:** chưa có `research-state.env` hợp lệ.

---

## 4. Chọn Kubernetes minor và patch

### 4.1. Thu snapshot release và lập danh sách minor maintained

Đọc [Kubernetes Releases](https://kubernetes.io/releases/). Kubernetes duy trì ba minor gần nhất và các release từ 1.19 nhận khoảng một năm patch support. Lưu cả trang lifecycle và release metadata từ repository official:

```bash
K8S_EVIDENCE="${BASELINE_ROOT}/evidence/04-kubernetes"

curl -fsSL 'https://kubernetes.io/releases/' \
  -o "${K8S_EVIDENCE}/kubernetes-releases.html"

curl -fsSL 'https://api.github.com/repos/kubernetes/kubernetes/releases?per_page=100' \
  -o "${K8S_EVIDENCE}/kubernetes-github-releases.json"

curl -fsSL 'https://kubernetes.io/releases/version-skew-policy/' \
  -o "${K8S_EVIDENCE}/version-skew-policy.html"

jq -r '
  .[]
  | select(.draft == false and .prerelease == false)
  | .tag_name
  | select(test("^v1\\.[0-9]+\\.[0-9]+$"))
' "${K8S_EVIDENCE}/kubernetes-github-releases.json" \
  | sort -Vr \
  | awk -F. '!seen[$1 "." $2]++ {print}' \
  | head -n3 \
  > "${K8S_EVIDENCE}/maintained-latest-patches.txt"

sha256sum "${K8S_EVIDENCE}"/* \
  > "${K8S_EVIDENCE}/SHA256SUMS"

cat "${K8S_EVIDENCE}/maintained-latest-patches.txt"
```

Mở trang lifecycle đã lưu hoặc trang official trực tiếp, rồi tạo file TSV với đúng ba minor và EOL tương ứng:

```bash
{
  printf 'minor\tlatest_patch\teol\tmaintained\tsource\n'
  printf '1.__\tv1.__.__\tYYYY-MM-DD\tYES\thttps://kubernetes.io/releases/\n'
  printf '1.__\tv1.__.__\tYYYY-MM-DD\tYES\thttps://kubernetes.io/releases/\n'
  printf '1.__\tv1.__.__\tYYYY-MM-DD\tYES\thttps://kubernetes.io/releases/\n'
} > "${K8S_EVIDENCE}/kubernetes-candidates.tsv"

nano "${K8S_EVIDENCE}/kubernetes-candidates.tsv"
```

Gate candidate upstream:

```bash
set -o pipefail
(
  FILE="${K8S_EVIDENCE}/kubernetes-candidates.tsv"
  test "$(awk 'NR>1 && NF {count++} END {print count+0}' "${FILE}")" -eq 3 || {
    echo 'FAIL: bảng phải có đúng ba minor maintained'; exit 1;
  }

  while IFS=$'\t' read -r minor patch eol maintained source; do
    [[ "${minor}" =~ ^1\.[0-9]+$ ]] || {
      echo "FAIL: minor không hợp lệ ${minor}"; exit 1;
    }
    [[ "${patch}" =~ ^v1\.[0-9]+\.[0-9]+$ ]] || {
      echo "FAIL: patch không hợp lệ ${patch}"; exit 1;
    }
    [[ "${patch%.*}" == "v${minor}" ]] || {
      echo "FAIL: ${patch} không thuộc minor ${minor}"; exit 1;
    }
    grep -Fxq "${patch}" "${K8S_EVIDENCE}/maintained-latest-patches.txt" || {
      echo "FAIL: ${patch} không khớp latest patch đã thu từ official release"; exit 1;
    }
    date -d "${eol}" '+%F' >/dev/null 2>&1 || {
      echo "FAIL: EOL không hợp lệ ${eol}"; exit 1;
    }
    grep -Fq "${eol}" "${K8S_EVIDENCE}/kubernetes-releases.html" || {
      echo "FAIL: EOL ${eol} không xuất hiện trong official release snapshot"; exit 1;
    }
    [[ "${maintained}" == YES ]] || {
      echo "FAIL: minor ${minor} không maintained"; exit 1;
    }
    [[ "${source}" == 'https://kubernetes.io/releases/' ]] || {
      echo "FAIL: source lifecycle không đúng official URL"; exit 1;
    }
  done < <(tail -n +2 "${FILE}")

  echo 'PASS: có đúng ba minor maintained, latest patch và EOL hợp lệ'
) 2>&1 | tee "${K8S_EVIDENCE}/gate-41-upstream.txt"
K8S_UPSTREAM_RC=${PIPESTATUS[0]}
echo "Kubernetes upstream gate rc=${K8S_UPSTREAM_RC}"
```

**PASS:** `rc=0`. **FAIL/STOP:** không lập ma trận support cho tới khi ba dòng khớp official release page.

### 4.2. Thu Rancher/cert-manager/Helm evidence và lập giao version

Lưu các nguồn quyết định dải tương thích:

```bash
set -a
source "${BASELINE_ROOT}/baseline.env"
set +a

curl -fsSL 'https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/' \
  -o "${K8S_EVIDENCE}/rancher-support-matrix.html"

curl -fsSL 'https://cert-manager.io/docs/releases/' \
  -o "${K8S_EVIDENCE}/cert-manager-supported-releases.html"

curl -fsSL 'https://helm.sh/docs/v3/topics/version_skew/' \
  -o "${K8S_EVIDENCE}/helm-version-support.html"

helm repo add rancher-stable \
  'https://releases.rancher.com/server-charts/stable' \
  --force-update

helm search repo rancher-stable/rancher --versions \
  > "${K8S_EVIDENCE}/rancher-stable-versions.txt"

RANCHER_CANDIDATE="$(awk 'NR>1 && $2 !~ /-/ {print $2; exit}' \
  "${K8S_EVIDENCE}/rancher-stable-versions.txt")"

set -o pipefail
(
  test -n "${RANCHER_CANDIDATE}" || {
    echo 'FAIL: rancher-stable không có candidate final release'; exit 1;
  }

  helm show chart rancher-stable/rancher \
    --version "${RANCHER_CANDIDATE}" \
    > "${K8S_EVIDENCE}/rancher-${RANCHER_CANDIDATE}-Chart.yaml" || {
    echo 'FAIL: không đọc được metadata của Rancher candidate'; exit 1;
  }

  grep -E '^(version|appVersion|kubeVersion):' \
    "${K8S_EVIDENCE}/rancher-${RANCHER_CANDIDATE}-Chart.yaml" || {
    echo 'FAIL: Rancher Chart.yaml thiếu version/appVersion/kubeVersion'; exit 1;
  }

  echo "PASS: Rancher stable candidate và chart metadata hợp lệ = ${RANCHER_CANDIDATE}"
) 2>&1 | tee "${K8S_EVIDENCE}/gate-421-rancher-discovery.txt"
RANCHER_DISCOVERY_RC=${PIPESTATUS[0]}
echo "Rancher discovery gate rc=${RANCHER_DISCOVERY_RC}"

if [[ "${RANCHER_DISCOVERY_RC}" -eq 0 ]]; then
  printf 'RANCHER_CANDIDATE=%q\n' "${RANCHER_CANDIDATE}" \
    >> "${BASELINE_ROOT}/research-state.env"
else
  echo 'STOP: sửa Rancher repository/candidate rồi chạy lại mục này; không chạy §4.2 tiếp theo'
fi
```

**PASS:** `Rancher discovery gate rc=0`. **FAIL/STOP:** không thực hiện các block còn lại của §4.2; subshell chỉ đóng gate, không đóng phiên SSH.

Mở bốn evidence files, đọc đúng các bảng `Manager host`, `Downstream/Imported`, cert-manager `Supported/Tested` và Helm version support. Sau đó tạo matrix; không dùng `UNKNOWN` để lách gate:

```bash
{
  printf 'k8s_minor\tupstream\trancher_manager\trancher_imported\tcert_manager\thelm\trancher_chart\tdecision\treason\n'
  while IFS=$'\t' read -r minor _; do
    [[ "${minor}" == minor ]] && continue
    printf '%s\tPASS\tUNKNOWN\tUNKNOWN\tUNKNOWN\tUNKNOWN\tUNKNOWN\tREJECT\tchưa đánh giá\n' "${minor}"
  done < "${K8S_EVIDENCE}/kubernetes-candidates.tsv"
} > "${BASELINE_ROOT}/compatibility-matrix.tsv"

nano "${BASELINE_ROOT}/compatibility-matrix.tsv"
```

Quy ước ô: `PASS`, `FAIL`, `N/A`, `NOT-CERTIFIED`; decision chỉ là `KEEP` hoặc `REJECT`. Với support goal `vendor-supported`, `NOT-CERTIFIED` ở vai trò Rancher đang dùng không được `KEEP`. Với `technically-compatible`/`lab-only`, có thể `KEEP` nhưng support class cuối không được ghi vendor-supported.

Gate ma trận:

```bash
set -o pipefail
(
  python3 - "${BASELINE_ROOT}/compatibility-matrix.tsv" <<'PY'
import csv
import os
import sys

path = sys.argv[1]
allowed = {'PASS', 'FAIL', 'N/A', 'NOT-CERTIFIED'}
rows = list(csv.DictReader(open(path, encoding='utf-8'), delimiter='\t'))
if len(rows) != 3:
    raise SystemExit('FAIL: compatibility matrix phải có đúng ba candidate upstream')

keep = []
for row in rows:
    for key in ('upstream', 'rancher_manager', 'rancher_imported',
                'cert_manager', 'helm', 'rancher_chart'):
        if row[key] not in allowed:
            raise SystemExit(f'FAIL: {row["k8s_minor"]} cột {key}={row[key]} không hợp lệ')
    if row['decision'] not in {'KEEP', 'REJECT'}:
        raise SystemExit(f'FAIL: decision không hợp lệ cho {row["k8s_minor"]}')
    if not row['reason'].strip() or row['reason'] == 'chưa đánh giá':
        raise SystemExit(f'FAIL: thiếu reason cho {row["k8s_minor"]}')
    if row['decision'] == 'KEEP':
        if any(row[key] != 'PASS' for key in ('upstream', 'cert_manager', 'helm', 'rancher_chart')):
            raise SystemExit(f'FAIL: {row["k8s_minor"]} KEEP nhưng gate bắt buộc chưa PASS')
        goal = os.environ['SUPPORT_GOAL']
        role = os.environ['RANCHER_ROLE']
        required = os.environ['RANCHER_REQUIRED'] == 'true'
        if required and goal == 'vendor-supported':
            if role in {'manager-host', 'both'} and row['rancher_manager'] != 'PASS':
                raise SystemExit(f'FAIL: {row["k8s_minor"]} không vendor-supported cho Manager host')
            if role in {'downstream-imported', 'both'} and row['rancher_imported'] != 'PASS':
                raise SystemExit(f'FAIL: {row["k8s_minor"]} không vendor-supported cho imported role')
        keep.append(row['k8s_minor'])

if not keep:
    raise SystemExit('FAIL: giao version rỗng; không có candidate KEEP')
print('PASS: candidate KEEP = ' + ', '.join(keep))
PY
) 2>&1 | tee "${K8S_EVIDENCE}/gate-42-intersection.txt"
INTERSECTION_RC=${PIPESTATUS[0]}
echo "intersection gate rc=${INTERSECTION_RC}"
```

**PASS:** có ít nhất một candidate `KEEP` và `rc=0`. **FAIL/STOP:** không được tự chọn version ngoài giao.

### 4.3. Chọn minor, exact patch và exact Debian package

Liệt kê các dòng KEEP rồi nhập minor được chọn. Nếu có nhiều KEEP, ưu tiên thời gian tới EOL, support class và mức trưởng thành; ghi lý do vào decision log.

```bash
awk -F'\t' 'NR==1 || $8=="KEEP"' "${BASELINE_ROOT}/compatibility-matrix.tsv"
read -r -p 'Nhập K8S_MINOR từ dòng KEEP (vd 1.35): ' K8S_MINOR

while ! grep -Eq "^${K8S_MINOR}"$'\t.*\tKEEP\t' \
  "${BASELINE_ROOT}/compatibility-matrix.tsv"; do
  echo "FAIL: ${K8S_MINOR} không phải candidate KEEP"
  read -r -p 'Nhập lại K8S_MINOR từ dòng KEEP: ' K8S_MINOR
done

K8S_VERSION="$(awk -F'\t' -v minor="${K8S_MINOR}" \
  '$1==minor {sub(/^v/, "", $2); print $2}' \
  "${K8S_EVIDENCE}/kubernetes-candidates.tsv")"

printf 'K8S_MINOR=%q\nK8S_VERSION=%q\n' \
  "${K8S_MINOR}" "${K8S_VERSION}" \
  >> "${BASELINE_ROOT}/research-state.env"

printf '| %s | %s | %s | %s | %s | %s |\n' \
  "${RESEARCH_DATE}" 'Kubernetes' "${K8S_MINOR}" 'KEEP' \
  "selected from KEEP; exact patch ${K8S_VERSION}" 'khi support matrix/lifecycle đổi' \
  >> "${BASELINE_ROOT}/decision-log.md"
```

Theo [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), research host phải dùng repository `pkgs.k8s.io` riêng cho minor đã chọn. Xác nhận repo, làm mới index rồi thu package inventory; không cần cài cluster:

```bash
set -o pipefail
(
  grep -R "core:/stable:/v${K8S_MINOR}/deb" \
    /etc/apt/sources.list /etc/apt/sources.list.d 2>/dev/null || {
    echo "FAIL: chưa cấu hình repo Kubernetes v${K8S_MINOR}"; exit 1;
  }
  echo "PASS: research host dùng đúng pkgs.k8s.io repo minor v${K8S_MINOR}"
) 2>&1 | tee "${K8S_EVIDENCE}/gate-431-repository.txt"
K8S_REPO_RC=${PIPESTATUS[0]}
echo "Kubernetes repo gate rc=${K8S_REPO_RC}"
```

**FAIL/STOP:** làm đúng mục Debian-based trong official Installing kubeadm rồi chạy lại gate. Chỉ khi `rc=0` mới chạy:

```bash

sudo apt-get update

for package in kubeadm kubelet kubectl; do
  apt-cache madison "${package}" \
    > "${K8S_EVIDENCE}/${package}-madison.txt"
done
```

Gate exact package:

```bash
set -o pipefail
(
  versions=()
  for package in kubeadm kubelet kubectl; do
    version="$(awk -F'|' -v prefix="${K8S_VERSION}-" \
      '{gsub(/ /, "", $2); if (index($2, prefix)==1) {print $2; exit}}' \
      "${K8S_EVIDENCE}/${package}-madison.txt")"
    test -n "${version}" || {
      echo "FAIL: repo không có ${package} ${K8S_VERSION}-*"; exit 1;
    }
    echo "${package}=${version}"
    versions+=("${version}")
  done

  unique_count="$(printf '%s\n' "${versions[@]}" | sort -u | wc -l)"
  [[ "${unique_count}" -eq 1 ]] || {
    echo 'FAIL: kubeadm/kubelet/kubectl không có cùng exact Debian version'; exit 1;
  }

  K8S_DEB_VERSION="${versions[0]}"
  printf 'K8S_DEB_VERSION=%q\n' "${K8S_DEB_VERSION}" \
    >> "${BASELINE_ROOT}/research-state.env"
  echo "PASS: ba package cùng version ${K8S_DEB_VERSION}"
) 2>&1 | tee "${K8S_EVIDENCE}/gate-43-packages.txt"
PACKAGE_RC=${PIPESTATUS[0]}
echo "package gate rc=${PACKAGE_RC}"
```

**PASS:** ba package cùng exact version và `rc=0`. **FAIL/STOP:** không chốt Kubernetes patch.

`stable.txt` có thể nhảy sang minor khác; không dùng nó thay cho candidate đã chọn từ giao support.

### 4.4. Gate version-skew decision

Theo [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/), kubelet không được mới hơn API server, còn kubectl có dải skew riêng. Với cluster mới, baseline này yêu cầu ba package cùng exact version dù policy cho phép biên rộng hơn.

```bash
source "${BASELINE_ROOT}/research-state.env"

set -o pipefail
(
  grep -Fq "PASS: ba package cùng version ${K8S_DEB_VERSION}" \
    "${K8S_EVIDENCE}/gate-43-packages.txt" || {
    echo 'FAIL: package equality gate chưa PASS'; exit 1;
  }

  grep -Fq 'Version Skew Policy' \
    "${K8S_EVIDENCE}/version-skew-policy.html" || {
    echo 'FAIL: chưa lưu được official skew policy'; exit 1;
  }

  echo "PASS: baseline pin kubeadm/kubelet/kubectl=${K8S_DEB_VERSION}; không chủ động dùng skew"
) 2>&1 | tee "${K8S_EVIDENCE}/gate-44-version-skew.txt"
SKEW_RC=${PIPESTATUS[0]}
echo "skew gate rc=${SKEW_RC}"
```

**PASS:** `skew gate rc=0`. **FAIL/STOP:** evidence skew hoặc package equality chưa đạt.

### 4.5. Ghi inventory image do đúng kubeadm version chọn

Kubeadm chọn `kube-apiserver`, controller, scheduler, kube-proxy, CoreDNS, etcd và pause image. Không tự nâng riêng các image này.

Trên **research VM**, bảo đảm kubeadm đúng exact candidate. Nếu chưa có, có thể cài đúng package candidate trên VM nghiên cứu; thao tác này không khởi tạo cluster:

```bash
source "${BASELINE_ROOT}/research-state.env"

if ! command -v kubeadm >/dev/null 2>&1 || \
   [[ "$(kubeadm version -o short 2>/dev/null)" != "v${K8S_VERSION}" ]]; then
  echo "Research VM cần kubeadm=${K8S_DEB_VERSION} để sinh inventory chính xác."
  echo "Sau khi được phê duyệt theo chính sách phần mềm, cài đúng package rồi chạy lại §4.5:"
  echo "sudo apt-get install kubeadm='${K8S_DEB_VERSION}'"
fi
```

Chỉ chạy block tiếp theo khi `kubeadm version -o short` khớp:

```bash
set -o pipefail
(
  command -v kubeadm >/dev/null 2>&1 || {
    echo 'FAIL: chưa có kubeadm'; exit 1;
  }
  [[ "$(kubeadm version -o short)" == "v${K8S_VERSION}" ]] || {
    echo "FAIL: kubeadm hiện tại không phải v${K8S_VERSION}"; exit 1;
  }

  kubeadm config images list \
    --kubernetes-version "v${K8S_VERSION}" \
    > "${K8S_EVIDENCE}/kubeadm-images-${K8S_VERSION}.txt"

  for component in kube-apiserver kube-controller-manager kube-scheduler kube-proxy coredns etcd pause; do
    grep -qi "${component}" "${K8S_EVIDENCE}/kubeadm-images-${K8S_VERSION}.txt" || {
      echo "FAIL: inventory thiếu ${component}"; exit 1;
    }
  done

  if grep -Eq ':(latest|stable)(@|$)' \
    "${K8S_EVIDENCE}/kubeadm-images-${K8S_VERSION}.txt"; then
    echo 'FAIL: kubeadm inventory có tag động'; exit 1
  fi

  echo "PASS: kubeadm v${K8S_VERSION} sinh đủ inventory và không có tag động"
) 2>&1 | tee "${K8S_EVIDENCE}/gate-45-images.txt"
KUBEADM_IMAGES_RC=${PIPESTATUS[0]}
echo "kubeadm image gate rc=${KUBEADM_IMAGES_RC}"
```

**PASS:** `rc=0`. Digest được resolve ở §7.5 bằng cùng image-inspection tool với các addon. **FAIL/STOP:** không sang runtime/CNI.

---

## 5. Chọn OS và container runtime

### 5.1. Thu OS evidence và gate đúng research host

```bash
RUNTIME_EVIDENCE="${BASELINE_ROOT}/evidence/05-runtime"
set -a
source "${BASELINE_ROOT}/baseline.env"
set +a

curl -fsSL 'https://ubuntu.com/about/release-cycle' \
  -o "${RUNTIME_EVIDENCE}/ubuntu-release-cycle.html"

cp /etc/os-release "${RUNTIME_EVIDENCE}/research-host-os-release.txt"
uname -a > "${RUNTIME_EVIDENCE}/research-host-uname.txt"
dpkg --print-architecture \
  > "${RUNTIME_EVIDENCE}/research-host-dpkg-architecture.txt"
```

Đọc Ubuntu lifecycle và điền ngày kết thúc standard security maintenance của OS target:

```bash
{
  printf 'os_id\tos_version\tarchitecture\tstandard_support_end\trancher_role_support\n'
  printf '%s\t%s\t%s\tYYYY-MM-DD\tUNKNOWN\n' \
    "${OS_ID}" "${OS_VERSION}" "${ARCHITECTURE}"
} > "${RUNTIME_EVIDENCE}/os-candidate.tsv"

nano "${RUNTIME_EVIDENCE}/os-candidate.tsv"
```

`rancher_role_support` dùng `PASS`, `NOT-CERTIFIED` hoặc `N/A`, lấy từ đúng Rancher support matrix và đúng vai trò. Gate:

```bash
set -o pipefail
(
  source /etc/os-release
  [[ "${ID}" == "${OS_ID}" ]] || {
    echo "FAIL: research host ID=${ID}, target=${OS_ID}"; exit 1;
  }
  [[ "${VERSION_ID}" == "${OS_VERSION}" ]] || {
    echo "FAIL: research host VERSION_ID=${VERSION_ID}, target=${OS_VERSION}"; exit 1;
  }
  [[ "$(dpkg --print-architecture)" == "${ARCHITECTURE}" ]] || {
    echo "FAIL: architecture research host không khớp target"; exit 1;
  }

  IFS=$'\t' read -r _ os_version architecture support_end rancher_support \
    < <(tail -n1 "${RUNTIME_EVIDENCE}/os-candidate.tsv")
  [[ "${os_version}" == "${OS_VERSION}" && "${architecture}" == "${ARCHITECTURE}" ]] || {
    echo 'FAIL: os-candidate.tsv không khớp input contract'; exit 1;
  }
  date -d "${support_end}" '+%F' >/dev/null 2>&1 || {
    echo 'FAIL: standard_support_end chưa hợp lệ'; exit 1;
  }
  [[ "$(date -d "${support_end}" +%s)" -gt "$(date -d "${TARGET_INSTALL_DATE}" +%s)" ]] || {
    echo 'FAIL: OS hết standard support trước ngày cài dự kiến'; exit 1;
  }
  [[ "${rancher_support}" =~ ^(PASS|NOT-CERTIFIED|N/A)$ ]] || {
    echo 'FAIL: rancher_role_support chưa đánh giá'; exit 1;
  }
  if [[ "${SUPPORT_GOAL}" == vendor-supported && "${RANCHER_REQUIRED}" == true && \
        "${rancher_support}" != PASS ]]; then
    echo 'FAIL: OS/topology không đạt vendor-supported cho Rancher role'; exit 1
  fi

  echo "PASS: research host khớp ${OS_ID} ${OS_VERSION}/${ARCHITECTURE}; lifecycle và Rancher role đã đánh giá"
) 2>&1 | tee "${RUNTIME_EVIDENCE}/gate-51-os.txt"
OS_RC=${PIPESTATUS[0]}
echo "OS gate rc=${OS_RC}"
```

**PASS:** `OS gate rc=0`. **FAIL/STOP:** không dùng package inventory từ một OS/architecture khác.

### 5.2. Chọn exact containerd package và branch

Đọc [Kubernetes Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) và [containerd release lifecycle](https://github.com/containerd/containerd/blob/main/RELEASES.md), rồi lưu evidence:

```bash
curl -fsSL \
  'https://kubernetes.io/docs/setup/production-environment/container-runtimes/' \
  -o "${RUNTIME_EVIDENCE}/kubernetes-container-runtimes.html"

curl -fsSL \
  'https://raw.githubusercontent.com/containerd/containerd/main/RELEASES.md' \
  -o "${RUNTIME_EVIDENCE}/containerd-RELEASES.md"

apt-cache policy containerd \
  > "${RUNTIME_EVIDENCE}/containerd-policy.txt"
apt-cache madison containerd \
  > "${RUNTIME_EVIDENCE}/containerd-madison.txt"

CONTAINERD_DEB_VERSION="$(awk '/Candidate:/ {print $2}' \
  "${RUNTIME_EVIDENCE}/containerd-policy.txt")"

apt-cache show "containerd=${CONTAINERD_DEB_VERSION}" \
  > "${RUNTIME_EVIDENCE}/containerd-package.txt"

grep -E '^(Package|Version|Architecture|Source):' \
  "${RUNTIME_EVIDENCE}/containerd-package.txt" | head
```

Từ exact Debian version và `RELEASES.md`, điền config generation và branch status:

```bash
{
  printf 'deb_version\tupstream_version\tconfig_generation\tbranch_status\tcri_api\tcgroup_driver\n'
  printf '%s\tX.Y.Z\t1.x-or-2.x\tSUPPORTED-or-EOL\tv1\tsystemd\n' \
    "${CONTAINERD_DEB_VERSION}"
} > "${RUNTIME_EVIDENCE}/containerd-candidate.tsv"

nano "${RUNTIME_EVIDENCE}/containerd-candidate.tsv"
```

Gate metadata:

```bash
set -o pipefail
(
  test -n "${CONTAINERD_DEB_VERSION}" && \
    [[ "${CONTAINERD_DEB_VERSION}" != '(none)' ]] || {
    echo 'FAIL: Ubuntu repo không có containerd candidate'; exit 1;
  }

  IFS=$'\t' read -r deb upstream generation branch cri cgroup \
    < <(tail -n1 "${RUNTIME_EVIDENCE}/containerd-candidate.tsv")

  [[ "${deb}" == "${CONTAINERD_DEB_VERSION}" ]] || {
    echo 'FAIL: Debian version không khớp apt candidate'; exit 1;
  }
  [[ "${upstream}" =~ ^[0-9]+\.[0-9]+\.[0-9]+ ]] || {
    echo 'FAIL: upstream version còn placeholder'; exit 1;
  }
  [[ "${generation}" =~ ^(1\.x|2\.x)$ ]] || {
    echo 'FAIL: config_generation phải là 1.x hoặc 2.x'; exit 1;
  }
  [[ "${branch}" == SUPPORTED ]] || {
    echo 'FAIL: containerd branch đã EOL hoặc chưa xác nhận'; exit 1;
  }
  [[ "${cri}" == v1 && "${cgroup}" == systemd ]] || {
    echo 'FAIL: contract phải là CRI v1 + systemd'; exit 1;
  }

  printf 'CONTAINERD_DEB_VERSION=%q\nCONTAINERD_UPSTREAM_VERSION=%q\nCONTAINERD_CONFIG_GENERATION=%q\n' \
    "${deb}" "${upstream}" "${generation}" \
    >> "${BASELINE_ROOT}/research-state.env"

  echo "PASS: containerd ${deb} -> upstream ${upstream}, ${generation}, CRI v1, systemd"
) 2>&1 | tee "${RUNTIME_EVIDENCE}/gate-52-containerd-metadata.txt"
CONTAINERD_METADATA_RC=${PIPESTATUS[0]}
echo "containerd metadata gate rc=${CONTAINERD_METADATA_RC}"
```

**PASS:** `rc=0`. Không để baseline ghi chung chung “containerd 2.x”.

### 5.3. Gate runtime configuration trên disposable research VM

Gate này cần exact package candidate đang chạy trên VM nghiên cứu; không chạy trên cluster production. Nếu package chưa được phê duyệt/cài, trạng thái component chỉ là `METADATA-PASS`, baseline chưa được `APPROVED`.

```bash
source "${BASELINE_ROOT}/research-state.env"

set -o pipefail
(
  command -v containerd >/dev/null 2>&1 || {
    echo 'FAIL: containerd chưa có trên research VM'; exit 1;
  }
  command -v crictl >/dev/null 2>&1 || {
    echo 'FAIL: thiếu crictl để kiểm CRI runtime'; exit 1;
  }
  [[ "$(dpkg-query -W -f='${Version}' containerd 2>/dev/null)" == \
      "${CONTAINERD_DEB_VERSION}" ]] || {
    echo 'FAIL: containerd đang chạy không đúng exact package candidate'; exit 1;
  }
  systemctl is-active --quiet containerd || {
    echo 'FAIL: containerd không active'; exit 1;
  }
  ! grep -Eq '^\s*disabled_plugins\s*=.*"cri"' /etc/containerd/config.toml || {
    echo 'FAIL: CRI đang bị disable'; exit 1;
  }
  crictl info > "${RUNTIME_EVIDENCE}/crictl-info.json"
  jq -e '.. | objects | select(.SystemdCgroup? == true)' \
    "${RUNTIME_EVIDENCE}/crictl-info.json" >/dev/null || {
      echo 'FAIL: crictl info không chứng minh SystemdCgroup=true'; exit 1;
    }

  echo 'PASS: exact containerd package active, CRI hoạt động và systemd cgroup được xác nhận'
) 2>&1 | tee "${RUNTIME_EVIDENCE}/gate-53-runtime.txt"
RUNTIME_RC=${PIPESTATUS[0]}
echo "runtime gate rc=${RUNTIME_RC}"
```

**PASS:** `runtime gate rc=0`. **FAIL/STOP:** sửa research VM hoặc đổi candidate; không chuyển CNI khi runtime gate chưa đạt.

---

## 6. Chọn CNI

[Kubernetes Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) yêu cầu một CNI plugin, tối thiểu tương thích CNI spec `v0.4.0` và khuyến nghị tương thích `v1.0.0`. Kubernetes không vì thế chứng nhận mọi release của mọi CNI.

### 6.1. Discovery exact Flannel release

```bash
CNI_EVIDENCE="${BASELINE_ROOT}/evidence/06-cni"
set -a
source "${BASELINE_ROOT}/baseline.env"
set +a
source "${BASELINE_ROOT}/research-state.env"

curl -fsSL 'https://api.github.com/repos/flannel-io/flannel/releases?per_page=30' \
  -o "${CNI_EVIDENCE}/flannel-releases.json"

jq -r '
  .[]
  | select(.draft == false and .prerelease == false)
  | [.tag_name, .published_at, .html_url]
  | @tsv
' "${CNI_EVIDENCE}/flannel-releases.json" \
  > "${CNI_EVIDENCE}/flannel-final-releases.tsv"

head "${CNI_EVIDENCE}/flannel-final-releases.tsv"
read -r -p 'Nhập exact FLANNEL_VERSION từ final release (vd v0.xx.y): ' FLANNEL_VERSION
```

Gate release candidate trước khi tải manifest:

```bash
set -o pipefail
(
  [[ "${FLANNEL_VERSION}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]] || {
    echo 'FAIL: FLANNEL_VERSION không phải exact final SemVer'; exit 1;
  }
  awk -F'\t' -v v="${FLANNEL_VERSION}" '$1==v {found=1} END {exit !found}' \
    "${CNI_EVIDENCE}/flannel-final-releases.tsv" || {
    echo 'FAIL: version không có trong official final releases'; exit 1;
  }
  echo "PASS: Flannel candidate ${FLANNEL_VERSION} là final release"
) 2>&1 | tee "${CNI_EVIDENCE}/gate-61-flannel-discovery.txt"
FLANNEL_DISCOVERY_RC=${PIPESTATUS[0]}
echo "Flannel discovery gate rc=${FLANNEL_DISCOVERY_RC}"
```

**PASS:** `rc=0`. **FAIL/STOP:** không dùng URL `latest/download` để thay thế.

### 6.2. Thu manifest exact tag và audit tĩnh

```bash
curl -fsSL \
  "https://github.com/flannel-io/flannel/releases/download/${FLANNEL_VERSION}/kube-flannel.yml" \
  -o "${CNI_EVIDENCE}/kube-flannel-${FLANNEL_VERSION}.yml"

curl -fsSL \
  "https://raw.githubusercontent.com/flannel-io/flannel/${FLANNEL_VERSION}/Documentation/kubernetes.md" \
  -o "${CNI_EVIDENCE}/flannel-kubernetes-${FLANNEL_VERSION}.md"

sha256sum "${CNI_EVIDENCE}/kube-flannel-${FLANNEL_VERSION}.yml" \
  > "${CNI_EVIDENCE}/kube-flannel-${FLANNEL_VERSION}.sha256"

grep -E '^apiVersion:|^kind:|^[[:space:]]+image:|"cniVersion"|"Network"' \
  "${CNI_EVIDENCE}/kube-flannel-${FLANNEL_VERSION}.yml" \
  > "${CNI_EVIDENCE}/flannel-manifest-inventory.txt"
```

Nếu `POD_CIDR` khác manifest release, tạo một bản candidate riêng trong evidence, chỉ đổi trường `Network` và lưu diff. Không sửa file release gốc:

```bash
cp "${CNI_EVIDENCE}/kube-flannel-${FLANNEL_VERSION}.yml" \
  "${CNI_EVIDENCE}/kube-flannel-candidate.yml"

# Chỉ chạy sed nếu POD_CIDR không phải default đang có trong manifest.
CURRENT_FLANNEL_CIDR="$(grep -oE '"Network"[[:space:]]*:[[:space:]]*"[^"]+"' \
  "${CNI_EVIDENCE}/kube-flannel-candidate.yml" | head -n1 | cut -d'"' -f4)"

if [[ "${CURRENT_FLANNEL_CIDR}" != "${POD_CIDR}" ]]; then
  sed -i "s#\"Network\": \"${CURRENT_FLANNEL_CIDR}\"#\"Network\": \"${POD_CIDR}\"#" \
    "${CNI_EVIDENCE}/kube-flannel-candidate.yml"
fi

diff -u \
  "${CNI_EVIDENCE}/kube-flannel-${FLANNEL_VERSION}.yml" \
  "${CNI_EVIDENCE}/kube-flannel-candidate.yml" \
  > "${CNI_EVIDENCE}/flannel-cidr.patch" || true
```

Gate static:

```bash
set -o pipefail
(
  MANIFEST="${CNI_EVIDENCE}/kube-flannel-candidate.yml"

  ! grep -Eqi '(latest|/master/|:master)' "${MANIFEST}" || {
    echo 'FAIL: manifest còn tag/branch động'; exit 1;
  }
  grep -Fq "\"Network\": \"${POD_CIDR}\"" "${MANIFEST}" || {
    echo "FAIL: Flannel Network không khớp POD_CIDR=${POD_CIDR}"; exit 1;
  }
  grep -q 'kind: DaemonSet' "${MANIFEST}" || {
    echo 'FAIL: thiếu Flannel DaemonSet'; exit 1;
  }
  grep -q 'kind: ClusterRole' "${MANIFEST}" || {
    echo 'FAIL: thiếu Flannel RBAC'; exit 1;
  }
  grep -Eq '"cniVersion"[[:space:]]*:[[:space:]]*"(0\.[4-9]|1\.)' \
    "${MANIFEST}" || {
    echo 'FAIL: CNI spec thấp hơn v0.4 hoặc không đọc được'; exit 1;
  }

  kubectl apply --dry-run=client --validate=false -f "${MANIFEST}" \
    > "${CNI_EVIDENCE}/flannel-client-dry-run.txt"

  FLANNEL_CNI_SPEC="$(grep -oE '"cniVersion"[[:space:]]*:[[:space:]]*"[^"]+"' \
    "${MANIFEST}" | head -n1 | cut -d'"' -f4)"

  printf 'FLANNEL_VERSION=%q\nFLANNEL_CNI_SPEC=%q\nFLANNEL_SUPPORT_CLASS=%q\n' \
    "${FLANNEL_VERSION}" "${FLANNEL_CNI_SPEC}" 'UPSTREAM-RANGE-NOT-STATED' \
    >> "${BASELINE_ROOT}/research-state.env"

  echo "PASS: Flannel ${FLANNEL_VERSION}, CNI spec ${FLANNEL_CNI_SPEC}, podCIDR ${POD_CIDR}, client dry-run đạt"
) 2>&1 | tee "${CNI_EVIDENCE}/gate-62-flannel-static.txt"
FLANNEL_STATIC_RC=${PIPESTATUS[0]}
echo "Flannel static gate rc=${FLANNEL_STATIC_RC}"
```

**PASS:** `rc=0`, nhưng support class vẫn là `UPSTREAM-RANGE-NOT-STATED` nếu release docs không công bố exact Kubernetes matrix.

### 6.3. Server dry-run/lab gate

Chỉ chạy trên disposable cluster có đúng `K8S_VERSION`, runtime và topology target:

```bash
kubectl version \
  > "${CNI_EVIDENCE}/lab-kubectl-version.txt"

kubectl apply --dry-run=server \
  -f "${CNI_EVIDENCE}/kube-flannel-candidate.yml" \
  > "${CNI_EVIDENCE}/flannel-server-dry-run.txt"
```

Gate:

```bash
set -o pipefail
(
  grep -Fq "Server Version: v${K8S_VERSION}" \
    "${CNI_EVIDENCE}/lab-kubectl-version.txt" || {
    echo "FAIL: disposable cluster không chạy v${K8S_VERSION}"; exit 1;
  }
  test -s "${CNI_EVIDENCE}/flannel-server-dry-run.txt" || {
    echo 'FAIL: thiếu server dry-run evidence'; exit 1;
  }
  echo 'PASS: Flannel server dry-run đạt trên exact Kubernetes candidate'
) 2>&1 | tee "${CNI_EVIDENCE}/gate-63-flannel-server.txt"
FLANNEL_SERVER_RC=${PIPESTATUS[0]}
echo "Flannel server gate rc=${FLANNEL_SERVER_RC}"
```

**PASS:** `rc=0`. **FAIL/STOP:** candidate không được `LAB-PASS`. Phân biệt release Flannel, CNI spec và version CNI binaries trên node trong bảng cuối.

---

## 7. Chọn Helm và các chart

### 7.1. Helm

Đọc [Helm Version Support Policy](https://helm.sh/docs/v3/topics/version_skew/) và [Rancher Helm Version Requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements).

```bash
CHART_EVIDENCE="${BASELINE_ROOT}/evidence/07-charts"
source "${BASELINE_ROOT}/research-state.env"

curl -fsSL 'https://helm.sh/docs/v3/topics/version_skew/' \
  -o "${CHART_EVIDENCE}/helm-version-support.html"
curl -fsSL \
  'https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements' \
  -o "${CHART_EVIDENCE}/rancher-helm-requirements.html"
curl -fsSL 'https://api.github.com/repos/helm/helm/releases?per_page=30' \
  -o "${CHART_EVIDENCE}/helm-releases.json"

jq -r '.[] | select(.draft==false and .prerelease==false) | [.tag_name,.published_at,.html_url] | @tsv' \
  "${CHART_EVIDENCE}/helm-releases.json" \
  > "${CHART_EVIDENCE}/helm-final-releases.tsv"

head "${CHART_EVIDENCE}/helm-final-releases.tsv"
read -r -p 'Nhập HELM_VERSION exact final release (vd v3.xx.y): ' HELM_VERSION
```

Đọc support table, xác định dải Kubernetes của Helm minor và Rancher minimum, rồi điền:

```bash
{
  printf 'helm_version\tk8s_min\tk8s_max\trancher_requirement\n'
  printf '%s\t1.__\t1.__\tPASS-or-FAIL\n' "${HELM_VERSION}"
} > "${CHART_EVIDENCE}/helm-candidate.tsv"

nano "${CHART_EVIDENCE}/helm-candidate.tsv"
```

Gate Helm:

```bash
set -o pipefail
(
  awk -F'\t' -v v="${HELM_VERSION}" '$1==v {found=1} END {exit !found}' \
    "${CHART_EVIDENCE}/helm-final-releases.tsv" || {
    echo 'FAIL: HELM_VERSION không phải official final release'; exit 1;
  }

  IFS=$'\t' read -r version k8s_min k8s_max rancher_req \
    < <(tail -n1 "${CHART_EVIDENCE}/helm-candidate.tsv")
  [[ "${version}" == "${HELM_VERSION}" ]] || {
    echo 'FAIL: helm-candidate.tsv không khớp selection'; exit 1;
  }
  [[ "${rancher_req}" == PASS ]] || {
    echo 'FAIL: Helm không đạt Rancher minimum'; exit 1;
  }

  python3 - "${K8S_MINOR}" "${k8s_min}" "${k8s_max}" <<'PY'
import sys
def minor(value):
    major, minor_value = value.split('.')[:2]
    return int(major), int(minor_value)
target, low, high = map(minor, sys.argv[1:])
if not low <= target <= high:
    raise SystemExit(f'FAIL: Kubernetes {sys.argv[1]} ngoài dải Helm {sys.argv[2]}-{sys.argv[3]}')
print(f'PASS: Kubernetes {sys.argv[1]} nằm trong dải Helm {sys.argv[2]}-{sys.argv[3]}')
PY

  [[ "$(helm version --template '{{.Version}}')" == "${HELM_VERSION}" ]] || {
    echo "FAIL: Helm đang chạy không phải ${HELM_VERSION}"; exit 1;
  }

  helm version --template '{{.Version}} {{.GitCommit}} {{.GoVersion}}{{"\n"}}' \
    > "${CHART_EVIDENCE}/helm-build.txt"
  printf 'HELM_VERSION=%q\n' "${HELM_VERSION}" \
    >> "${BASELINE_ROOT}/research-state.env"
  echo "PASS: Helm ${HELM_VERSION} đạt Kubernetes range, Rancher minimum và đúng binary đang chạy"
) 2>&1 | tee "${CHART_EVIDENCE}/gate-71-helm.txt"
HELM_RC=${PIPESTATUS[0]}
echo "Helm gate rc=${HELM_RC}"
```

**PASS:** `Helm gate rc=0`. **FAIL/STOP:** chuẩn bị đúng Helm candidate rồi chạy lại; không render chart bằng một Helm version khác baseline.

### 7.2. cert-manager

Đọc [cert-manager Supported Releases](https://cert-manager.io/docs/releases/) và [official Helm installation](https://cert-manager.io/docs/installation/helm/).

Thu final releases và chọn một minor đang nằm trong bảng Supported Releases. Không lấy version từ command ví dụ của trang cài đặt làm “mới nhất”.

```bash
curl -fsSL 'https://cert-manager.io/docs/releases/' \
  -o "${CHART_EVIDENCE}/cert-manager-supported-releases.html"
curl -fsSL 'https://cert-manager.io/docs/installation/helm/' \
  -o "${CHART_EVIDENCE}/cert-manager-helm.html"
curl -fsSL 'https://api.github.com/repos/cert-manager/cert-manager/releases?per_page=50' \
  -o "${CHART_EVIDENCE}/cert-manager-releases.json"

jq -r '.[] | select(.draft==false and .prerelease==false) | [.tag_name,.published_at,.html_url] | @tsv' \
  "${CHART_EVIDENCE}/cert-manager-releases.json" \
  > "${CHART_EVIDENCE}/cert-manager-final-releases.tsv"

head "${CHART_EVIDENCE}/cert-manager-final-releases.tsv"
read -r -p 'Nhập CERT_MANAGER_VERSION exact patch thuộc supported minor: ' CERT_MANAGER_VERSION
```

Điền range đúng từ trang Supported Releases:

```bash
{
  printf 'version\tk8s_supported_min\tk8s_supported_max\tk8s_tested_min\tk8s_tested_max\tbranch_status\n'
  printf '%s\t1.__\t1.__\t1.__\t1.__\tSUPPORTED\n' "${CERT_MANAGER_VERSION}"
} > "${CHART_EVIDENCE}/cert-manager-candidate.tsv"

nano "${CHART_EVIDENCE}/cert-manager-candidate.tsv"
```

Gate release/range và chart metadata:

```bash
set -o pipefail
(
  [[ "${CERT_MANAGER_VERSION}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]] || {
    echo 'FAIL: cert-manager version không phải exact SemVer'; exit 1;
  }
  awk -F'\t' -v v="${CERT_MANAGER_VERSION}" '$1==v {found=1} END {exit !found}' \
    "${CHART_EVIDENCE}/cert-manager-final-releases.tsv" || {
    echo 'FAIL: cert-manager candidate không phải official final release'; exit 1;
  }

  CM_MINOR="$(sed -E 's/^v([0-9]+\.[0-9]+)\..*/v\1/' <<<"${CERT_MANAGER_VERSION}")"
  LATEST_IN_MINOR="$(awk -F'\t' -v prefix="${CM_MINOR}." \
    'index($1,prefix)==1 {print $1}' \
    "${CHART_EVIDENCE}/cert-manager-final-releases.tsv" | sort -Vr | head -n1)"
  [[ "${CERT_MANAGER_VERSION}" == "${LATEST_IN_MINOR}" ]] || {
    echo "FAIL: policy chỉ hỗ trợ patch cuối; latest của ${CM_MINOR} là ${LATEST_IN_MINOR}"; exit 1;
  }

  IFS=$'\t' read -r _ supported_min supported_max tested_min tested_max branch_status \
    < <(tail -n1 "${CHART_EVIDENCE}/cert-manager-candidate.tsv")
  [[ "${branch_status}" == SUPPORTED ]] || {
    echo 'FAIL: cert-manager branch không còn supported'; exit 1;
  }

  python3 - "${K8S_MINOR}" "${supported_min}" "${supported_max}" \
    "${tested_min}" "${tested_max}" <<'PY'
import sys
def value(text):
    a, b = text.split('.')[:2]
    return int(a), int(b)
target, supported_min, supported_max, tested_min, tested_max = map(value, sys.argv[1:])
if not supported_min <= target <= supported_max:
    raise SystemExit('FAIL: Kubernetes target ngoài cert-manager Supported range')
if not tested_min <= target <= tested_max:
    print('NOTICE: Kubernetes target supported nhưng ngoài Tested range')
else:
    print('PASS: Kubernetes target nằm trong cả Supported và Tested range')
PY

  helm show chart oci://quay.io/jetstack/charts/cert-manager \
    --version "${CERT_MANAGER_VERSION}" \
    > "${CHART_EVIDENCE}/cert-manager-${CERT_MANAGER_VERSION}-Chart.yaml"
  helm show values oci://quay.io/jetstack/charts/cert-manager \
    --version "${CERT_MANAGER_VERSION}" \
    > "${CHART_EVIDENCE}/cert-manager-${CERT_MANAGER_VERSION}-values.yaml"

  grep -Eq '^version: ' "${CHART_EVIDENCE}/cert-manager-${CERT_MANAGER_VERSION}-Chart.yaml" || {
    echo 'FAIL: cert-manager Chart metadata thiếu version'; exit 1;
  }
  printf 'CERT_MANAGER_VERSION=%q\n' "${CERT_MANAGER_VERSION}" \
    >> "${BASELINE_ROOT}/research-state.env"
  echo "PASS: cert-manager ${CERT_MANAGER_VERSION} là latest supported patch và chart metadata đọc được"
) 2>&1 | tee "${CHART_EVIDENCE}/gate-72-cert-manager.txt"
CERT_MANAGER_RC=${PIPESTATUS[0]}
echo "cert-manager gate rc=${CERT_MANAGER_RC}"
```

**PASS:** `rc=0`. Nếu chỉ Supported mà không Tested, ghi rõ trong support class và bắt buộc lab gate; không tự ghi `TESTED`.

### 7.3. Traefik

Dùng [Traefik Helm chart releases](https://github.com/traefik/traefik-helm-chart/releases) và [Traefik Kubernetes documentation](https://doc.traefik.io/traefik/setup/kubernetes/) làm điểm vào official.

```bash
helm repo add traefik 'https://traefik.github.io/charts' --force-update
helm search repo traefik/traefik --versions \
  > "${CHART_EVIDENCE}/traefik-chart-versions.txt"

curl -fsSL 'https://api.github.com/repos/traefik/traefik-helm-chart/releases?per_page=50' \
  -o "${CHART_EVIDENCE}/traefik-chart-releases.json"
curl -fsSL 'https://doc.traefik.io/traefik/setup/kubernetes/' \
  -o "${CHART_EVIDENCE}/traefik-kubernetes-docs.html"

head "${CHART_EVIDENCE}/traefik-chart-versions.txt"
read -r -p 'Nhập TRAEFIK_CHART_VERSION exact final chart: ' TRAEFIK_CHART_VERSION

helm show chart traefik/traefik \
  --version "${TRAEFIK_CHART_VERSION}" \
  > "${CHART_EVIDENCE}/traefik-${TRAEFIK_CHART_VERSION}-Chart.yaml"
helm show values traefik/traefik \
  --version "${TRAEFIK_CHART_VERSION}" \
  > "${CHART_EVIDENCE}/traefik-${TRAEFIK_CHART_VERSION}-values.yaml"
helm pull traefik/traefik \
  --version "${TRAEFIK_CHART_VERSION}" \
  --destination "${CHART_EVIDENCE}"

TRAEFIK_APP_VERSION="$(awk '/^appVersion:/ {gsub(/"/, "", $2); print $2}' \
  "${CHART_EVIDENCE}/traefik-${TRAEFIK_CHART_VERSION}-Chart.yaml")"
TRAEFIK_KUBE_RANGE="$(sed -n 's/^kubeVersion:[[:space:]]*//p' \
  "${CHART_EVIDENCE}/traefik-${TRAEFIK_CHART_VERSION}-Chart.yaml")"
```

Gate metadata/source archive:

```bash
set -o pipefail
(
  awk -v v="${TRAEFIK_CHART_VERSION}" 'NR>1 && $2==v {found=1} END {exit !found}' \
    "${CHART_EVIDENCE}/traefik-chart-versions.txt" || {
    echo 'FAIL: Traefik chart version không có trong official repo'; exit 1;
  }
  [[ "${TRAEFIK_CHART_VERSION}" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]] || {
    echo 'FAIL: Traefik chart không phải exact final SemVer'; exit 1;
  }
  [[ "${TRAEFIK_APP_VERSION}" =~ ^v?[0-9]+\.[0-9]+\.[0-9]+ ]] || {
    echo 'FAIL: không lấy được Traefik appVersion'; exit 1;
  }
  test -n "${TRAEFIK_KUBE_RANGE}" || {
    echo 'FAIL: Chart.yaml không khai kubeVersion'; exit 1;
  }
  test -s "${CHART_EVIDENCE}/traefik-${TRAEFIK_CHART_VERSION}.tgz" || {
    echo 'FAIL: thiếu exact chart archive'; exit 1;
  }

  sha256sum "${CHART_EVIDENCE}/traefik-${TRAEFIK_CHART_VERSION}.tgz" \
    > "${CHART_EVIDENCE}/traefik-${TRAEFIK_CHART_VERSION}.sha256"
  tar -xzf "${CHART_EVIDENCE}/traefik-${TRAEFIK_CHART_VERSION}.tgz" \
    -C "${CHART_EVIDENCE}"

  printf 'TRAEFIK_CHART_VERSION=%q\nTRAEFIK_APP_VERSION=%q\nTRAEFIK_KUBE_RANGE=%q\n' \
    "${TRAEFIK_CHART_VERSION}" "${TRAEFIK_APP_VERSION}" "${TRAEFIK_KUBE_RANGE}" \
    >> "${BASELINE_ROOT}/research-state.env"
  echo "PASS: Traefik chart=${TRAEFIK_CHART_VERSION}, app=${TRAEFIK_APP_VERSION}, kubeVersion=${TRAEFIK_KUBE_RANGE}"
) 2>&1 | tee "${CHART_EVIDENCE}/gate-73-traefik-metadata.txt"
TRAEFIK_METADATA_RC=${PIPESTATUS[0]}
echo "Traefik metadata gate rc=${TRAEFIK_METADATA_RC}"
```

**PASS:** `rc=0`. Provider, IngressClass, Service exposure, CRDs và Gateway API được kiểm bằng exact values/render ở §9. Không suy diễn annotation nginx có tác dụng với Traefik.

### 7.4. Rancher

§4.2 đã chọn candidate từ stable channel và dùng nó để lập giao Kubernetes. Không được đổi Rancher version sau đó mà không chạy lại §4.2.

```bash
source "${BASELINE_ROOT}/research-state.env"
RANCHER_VERSION="${RANCHER_CANDIDATE}"

curl -fsSL "https://api.github.com/repos/rancher/rancher/releases/tags/v${RANCHER_VERSION}" \
  -o "${CHART_EVIDENCE}/rancher-${RANCHER_VERSION}-release.json"

helm show chart rancher-stable/rancher \
  --version "${RANCHER_VERSION}" \
  > "${CHART_EVIDENCE}/rancher-${RANCHER_VERSION}-Chart.yaml"
helm show values rancher-stable/rancher \
  --version "${RANCHER_VERSION}" \
  > "${CHART_EVIDENCE}/rancher-${RANCHER_VERSION}-values.yaml"
helm pull rancher-stable/rancher \
  --version "${RANCHER_VERSION}" \
  --destination "${CHART_EVIDENCE}"
```

Gate exact stable chart/release:

```bash
set -o pipefail
(
  [[ "${RANCHER_VERSION}" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]] || {
    echo 'FAIL: Rancher version không phải exact final SemVer'; exit 1;
  }
  awk -v v="${RANCHER_VERSION}" 'NR>1 && $2==v {found=1} END {exit !found}' \
    "${K8S_EVIDENCE}/rancher-stable-versions.txt" || {
    echo 'FAIL: Rancher version không có trong stable repository'; exit 1;
  }
  [[ "$(jq -r '.draft' "${CHART_EVIDENCE}/rancher-${RANCHER_VERSION}-release.json")" == false ]] || {
    echo 'FAIL: Rancher release là draft'; exit 1;
  }
  [[ "$(jq -r '.prerelease' "${CHART_EVIDENCE}/rancher-${RANCHER_VERSION}-release.json")" == false ]] || {
    echo 'FAIL: Rancher release là prerelease'; exit 1;
  }

  CHART_VERSION="$(awk '/^version:/ {print $2}' \
    "${CHART_EVIDENCE}/rancher-${RANCHER_VERSION}-Chart.yaml")"
  APP_VERSION="$(awk '/^appVersion:/ {gsub(/"/, "", $2); sub(/^v/, "", $2); print $2}' \
    "${CHART_EVIDENCE}/rancher-${RANCHER_VERSION}-Chart.yaml")"
  KUBE_RANGE="$(sed -n 's/^kubeVersion:[[:space:]]*//p' \
    "${CHART_EVIDENCE}/rancher-${RANCHER_VERSION}-Chart.yaml")"

  [[ "${CHART_VERSION}" == "${RANCHER_VERSION}" && \
      "${APP_VERSION}" == "${RANCHER_VERSION}" ]] || {
    echo 'FAIL: Rancher chart version/appVersion không khớp product version'; exit 1;
  }
  test -n "${KUBE_RANGE}" || {
    echo 'FAIL: Rancher chart không khai kubeVersion'; exit 1;
  }

  sha256sum "${CHART_EVIDENCE}/rancher-${RANCHER_VERSION}.tgz" \
    > "${CHART_EVIDENCE}/rancher-${RANCHER_VERSION}.sha256"
  tar -xzf "${CHART_EVIDENCE}/rancher-${RANCHER_VERSION}.tgz" \
    -C "${CHART_EVIDENCE}"

  printf 'RANCHER_VERSION=%q\nRANCHER_KUBE_RANGE=%q\n' \
    "${RANCHER_VERSION}" "${KUBE_RANGE}" \
    >> "${BASELINE_ROOT}/research-state.env"
  echo "PASS: Rancher ${RANCHER_VERSION} final release, stable chart/app khớp, kubeVersion=${KUBE_RANGE}"
) 2>&1 | tee "${CHART_EVIDENCE}/gate-74-rancher-metadata.txt"
RANCHER_METADATA_RC=${PIPESTATUS[0]}
echo "Rancher metadata gate rc=${RANCHER_METADATA_RC}"
```

**PASS:** `rc=0`. Audit full values/templates ở §8 và render ở §9. Không dừng ở [Rancher Helm Chart Options](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/installation-references/helm-chart-options), vì trang đó có thể thiếu key mới.

### 7.5. Addon còn lại

### 7.5.1. Gate công cụ đọc image digest

Baseline `APPROVED` phải lưu digest. Dùng một công cụ đã được tổ chức phê duyệt và có sẵn; runbook không tự cài:

```bash
ADDON_EVIDENCE="${CHART_EVIDENCE}/addons"
mkdir -p "${ADDON_EVIDENCE}"

set -o pipefail
(
  if command -v crane >/dev/null 2>&1; then
    IMAGE_INSPECTOR=crane
  elif command -v skopeo >/dev/null 2>&1; then
    IMAGE_INSPECTOR=skopeo
  else
    echo 'FAIL: cần crane hoặc skopeo để resolve digest'; exit 1
  fi

  printf 'IMAGE_INSPECTOR=%q\n' "${IMAGE_INSPECTOR}" \
    >> "${BASELINE_ROOT}/research-state.env"
  echo "PASS: image inspector=${IMAGE_INSPECTOR}"
) 2>&1 | tee "${ADDON_EVIDENCE}/gate-751-image-tool.txt"
IMAGE_TOOL_RC=${PIPESTATUS[0]}
echo "image tool gate rc=${IMAGE_TOOL_RC}"
```

**FAIL/STOP:** không được để digest trống hoặc thay bằng `latest`.

Tạo helper dùng lại ở §9:

```bash
tee "${BASELINE_ROOT}/resolve-image-digest.sh" >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
image_ref="${1:?usage: resolve-image-digest.sh IMAGE}"
inspector="${IMAGE_INSPECTOR:?source research-state.env first}"

if [[ "${image_ref}" == *@sha256:* ]]; then
  printf '%s\n' "${image_ref##*@}"
elif [[ "${inspector}" == crane ]]; then
  crane digest "${image_ref}"
else
  skopeo inspect --format '{{.Digest}}' "docker://${image_ref}"
fi
EOF
chmod 0755 "${BASELINE_ROOT}/resolve-image-digest.sh"
```

### 7.5.2. Discovery exact addon releases

```bash
set -a
source "${BASELINE_ROOT}/baseline.env"
set +a

for spec in \
  'cloudflared cloudflare/cloudflared' \
  'local-path-provisioner rancher/local-path-provisioner' \
  'metallb metallb/metallb'; do
  name="${spec%% *}"
  repo="${spec#* }"
  curl -fsSL "https://api.github.com/repos/${repo}/releases?per_page=30" \
    -o "${ADDON_EVIDENCE}/${name}-releases.json"
  jq -r '.[] | select(.draft==false and .prerelease==false) | [.tag_name,.published_at,.html_url] | @tsv' \
    "${ADDON_EVIDENCE}/${name}-releases.json" \
    > "${ADDON_EVIDENCE}/${name}-final-releases.tsv"
done

echo '--- cloudflared ---'; head "${ADDON_EVIDENCE}/cloudflared-final-releases.tsv"
echo '--- local-path-provisioner ---'; head "${ADDON_EVIDENCE}/local-path-provisioner-final-releases.tsv"
echo '--- MetalLB ---'; head "${ADDON_EVIDENCE}/metallb-final-releases.tsv"

read -r -p 'CLOUDFLARED_VERSION exact final release (hoặc N/A): ' CLOUDFLARED_VERSION
read -r -p 'LOCAL_PATH_VERSION exact final release (hoặc N/A): ' LOCAL_PATH_VERSION
read -r -p 'METALLB_VERSION exact final release (hoặc N/A): ' METALLB_VERSION
```

Gate selection khớp `OPTIONAL_ADDONS`:

```bash
set -o pipefail
(
  check_addon() {
    local addon="$1" version="$2" release_file="$3"
    if [[ ",${OPTIONAL_ADDONS}," == *",${addon},"* ]]; then
      [[ "${version}" != N/A ]] || {
        echo "FAIL: ${addon} được yêu cầu nhưng version=N/A"; return 1;
      }
      awk -F'\t' -v v="${version}" '$1==v {found=1} END {exit !found}' \
        "${release_file}" || {
        echo "FAIL: ${addon} ${version} không phải official final release"; return 1;
      }
    else
      [[ "${version}" == N/A ]] || {
        echo "FAIL: ${addon} không nằm trong OPTIONAL_ADDONS nhưng đã chọn ${version}"; return 1;
      }
    fi
  }

  check_addon cloudflared "${CLOUDFLARED_VERSION}" \
    "${ADDON_EVIDENCE}/cloudflared-final-releases.tsv"
  check_addon local-path-provisioner "${LOCAL_PATH_VERSION}" \
    "${ADDON_EVIDENCE}/local-path-provisioner-final-releases.tsv"
  check_addon metallb "${METALLB_VERSION}" \
    "${ADDON_EVIDENCE}/metallb-final-releases.tsv"

  printf 'CLOUDFLARED_VERSION=%q\nLOCAL_PATH_VERSION=%q\nMETALLB_VERSION=%q\n' \
    "${CLOUDFLARED_VERSION}" "${LOCAL_PATH_VERSION}" "${METALLB_VERSION}" \
    >> "${BASELINE_ROOT}/research-state.env"
  echo 'PASS: addon selections khớp input contract và official final releases'
) 2>&1 | tee "${ADDON_EVIDENCE}/gate-752-addon-selection.txt"
ADDON_SELECTION_RC=${PIPESTATUS[0]}
echo "addon selection gate rc=${ADDON_SELECTION_RC}"
```

### 7.5.3. Manifest, static gate và digest

```bash
source "${BASELINE_ROOT}/research-state.env"

if [[ "${LOCAL_PATH_VERSION}" != N/A ]]; then
  curl -fsSL \
    "https://raw.githubusercontent.com/rancher/local-path-provisioner/${LOCAL_PATH_VERSION}/deploy/local-path-storage.yaml" \
    -o "${ADDON_EVIDENCE}/local-path-${LOCAL_PATH_VERSION}.yaml"
fi

if [[ "${METALLB_VERSION}" != N/A ]]; then
  curl -fsSL \
    "https://raw.githubusercontent.com/metallb/metallb/${METALLB_VERSION}/config/manifests/metallb-native.yaml" \
    -o "${ADDON_EVIDENCE}/metallb-${METALLB_VERSION}.yaml"
fi

: > "${ADDON_EVIDENCE}/addon-image-digests.tsv"
printf 'component\timage\tdigest\n' \
  > "${ADDON_EVIDENCE}/addon-image-digests.tsv"

if [[ "${CLOUDFLARED_VERSION}" != N/A ]]; then
  CLOUDFLARED_TAG="${CLOUDFLARED_VERSION#v}"
  image="docker.io/cloudflare/cloudflared:${CLOUDFLARED_TAG}"
  digest="$(IMAGE_INSPECTOR="${IMAGE_INSPECTOR}" \
    "${BASELINE_ROOT}/resolve-image-digest.sh" "${image}")"
  printf 'cloudflared\t%s\t%s\n' "${image}" "${digest}" \
    >> "${ADDON_EVIDENCE}/addon-image-digests.tsv"
fi

for manifest in "${ADDON_EVIDENCE}"/*.yaml; do
  [[ -e "${manifest}" ]] || continue
  sed -n -E "s/^[[:space:]]*image:[[:space:]]*['\"]?([^'\"[:space:]]+)['\"]?.*/\1/p" \
    "${manifest}" \
    | sort -u \
    | while read -r image; do
        digest="$(IMAGE_INSPECTOR="${IMAGE_INSPECTOR}" \
          "${BASELINE_ROOT}/resolve-image-digest.sh" "${image}")"
        printf '%s\t%s\t%s\n' "$(basename "${manifest}")" "${image}" "${digest}" \
          >> "${ADDON_EVIDENCE}/addon-image-digests.tsv"
      done
done
```

Gate static:

```bash
set -o pipefail
(
  for manifest in "${ADDON_EVIDENCE}"/*.yaml; do
    [[ -e "${manifest}" ]] || continue
    ! grep -Eqi '(latest|/master/|:master)' "${manifest}" || {
      echo "FAIL: ${manifest} còn tag/branch động"; exit 1;
    }
    kubectl apply --dry-run=client --validate=false -f "${manifest}" \
      > "${manifest}.client-dry-run.txt"
    sha256sum "${manifest}" > "${manifest}.sha256"
  done

  awk -F'\t' 'NR>1 && $3 !~ /^sha256:[0-9a-f]{64}$/ {bad=1} END {exit bad}' \
    "${ADDON_EVIDENCE}/addon-image-digests.tsv" || {
    echo 'FAIL: có image chưa resolve được SHA256 digest'; exit 1;
  }

  echo 'PASS: addon manifests pin exact tag, client dry-run đạt và mọi image có digest'
) 2>&1 | tee "${ADDON_EVIDENCE}/gate-753-addon-static.txt"
ADDON_STATIC_RC=${PIPESTATUS[0]}
echo "addon static gate rc=${ADDON_STATIC_RC}"
```

Nếu project không công bố Kubernetes compatibility range, ghi `UPSTREAM-RANGE-NOT-STATED` và yêu cầu server/lab gate ở §9. Không dùng manifest `latest` dù official quick-start dùng nó.

---

## 8. Audit chart để phát hiện hidden defaults và nhánh tính năng

### 8.1. Thu bằng chứng của đúng version

§7 đã lưu Rancher và Traefik archives. Thu thêm exact cert-manager OCI archive vào thư mục riêng; không đọc branch `main` để kết luận về release đã pin:

```bash
source "${BASELINE_ROOT}/research-state.env"
AUDIT_EVIDENCE="${CHART_EVIDENCE}/audit"
mkdir -p "${AUDIT_EVIDENCE}/cert-manager-bundle"

helm pull oci://quay.io/jetstack/charts/cert-manager \
  --version "${CERT_MANAGER_VERSION}" \
  --destination "${AUDIT_EVIDENCE}/cert-manager-bundle"

CM_ARCHIVE="$(find "${AUDIT_EVIDENCE}/cert-manager-bundle" \
  -maxdepth 1 -type f -name '*.tgz' -print -quit)"
test -n "${CM_ARCHIVE}" || echo 'FAIL: không tìm thấy cert-manager archive'
sha256sum "${CM_ARCHIVE}" \
  > "${AUDIT_EVIDENCE}/cert-manager-bundle/SHA256SUMS"
tar -xzf "${CM_ARCHIVE}" -C "${AUDIT_EVIDENCE}/cert-manager-bundle"

CERT_MANAGER_SOURCE="${AUDIT_EVIDENCE}/cert-manager-bundle/cert-manager"
TRAEFIK_SOURCE="${CHART_EVIDENCE}/traefik"
RANCHER_SOURCE="${CHART_EVIDENCE}/rancher"

printf 'CERT_MANAGER_SOURCE=%q\nTRAEFIK_SOURCE=%q\nRANCHER_SOURCE=%q\n' \
  "${CERT_MANAGER_SOURCE}" "${TRAEFIK_SOURCE}" "${RANCHER_SOURCE}" \
  >> "${BASELINE_ROOT}/research-state.env"
```

Gate source tree và tạo inventory:

```bash
set -o pipefail
(
  for entry in \
    "cert-manager:${CERT_MANAGER_SOURCE}" \
    "traefik:${TRAEFIK_SOURCE}" \
    "rancher:${RANCHER_SOURCE}"; do
    component="${entry%%:*}"
    directory="${entry#*:}"
    test -f "${directory}/Chart.yaml" || {
      echo "FAIL: ${component} thiếu Chart.yaml"; exit 1;
    }
    test -f "${directory}/values.yaml" || {
      echo "FAIL: ${component} thiếu values.yaml"; exit 1;
    }
    test -d "${directory}/templates" || {
      echo "FAIL: ${component} thiếu templates/"; exit 1;
    }

    rg -n '\.Values\.|Capabilities\.APIVersions|lookup|kind:|apiVersion:' \
      "${directory}" \
      > "${AUDIT_EVIDENCE}/${component}-values-capabilities-kinds.txt" || true
    rg -n '{{-?\s*(if|with|range)|include|define' \
      "${directory}/templates" \
      > "${AUDIT_EVIDENCE}/${component}-template-branches.txt" || true

    test -s "${AUDIT_EVIDENCE}/${component}-values-capabilities-kinds.txt" || {
      echo "FAIL: ${component} audit inventory rỗng"; exit 1;
    }
  done

  echo 'PASS: đã lưu exact source tree và audit inventory cho cả ba chart'
) 2>&1 | tee "${AUDIT_EVIDENCE}/gate-81-source-trees.txt"
SOURCE_TREE_RC=${PIPESTATUS[0]}
echo "source tree gate rc=${SOURCE_TREE_RC}"
```

**PASS:** `rc=0`. **FAIL/STOP:** không tạo config contract từ docs/options page thay cho source chart.

### 8.2. Tạo “config contract”

Với mọi key có khả năng đổi loại resource, controller, TLS, storage, network exposure hoặc security behavior, điền bảng:

```bash
set -a
source "${BASELINE_ROOT}/baseline.env"
set +a
```

Tạo bảng TSV. Các giá trị `UNKNOWN` phải được thay bằng kết quả đọc exact `values.yaml`/template. Cột `evidence` dùng đường dẫn tương đối dưới `evidence/07-charts`, có thể kèm `:line`:

```bash
{
  printf 'component\tkey_path\tdefault\ttemplate_gate\tpinned_value\tdependency\tfailure_mode\tevidence\n'
  printf 'cert-manager\tcrds.enabled\tUNKNOWN\tUNKNOWN\ttrue\tKubernetes CRDs\tCRDs không được render\tUNKNOWN\n'
  printf 'traefik\tproviders.kubernetesIngress.enabled\tUNKNOWN\tUNKNOWN\ttrue\tIngress API\tIngress không được reconcile\tUNKNOWN\n'
  printf 'traefik\tproviders.kubernetesGateway.enabled\tUNKNOWN\tUNKNOWN\tfalse\tGateway API CRDs\tnhầm feature branch\tUNKNOWN\n'
  printf 'traefik\tingressClass.enabled\tUNKNOWN\tUNKNOWN\ttrue\tIngressClass\tclass không tồn tại\tUNKNOWN\n'
  printf 'rancher\tnetworkExposure.type\tUNKNOWN\tUNKNOWN\t%s\tIngress hoặc Gateway API\tRancher không có route\tUNKNOWN\n' "${RANCHER_EXPOSURE}"
  printf 'rancher\tingress.enabled\tUNKNOWN\tUNKNOWN\ttrue\tnetworkExposure.type=ingress\tIngress không render\tUNKNOWN\n'
  printf 'rancher\tingress.ingressClassName\tUNKNOWN\tUNKNOWN\ttraefik\tTraefik IngressClass\tIngress bị bỏ qua\tUNKNOWN\n'
  printf 'rancher\tingress.includeDefaultExtraAnnotations\tUNKNOWN\tUNKNOWN\tfalse\tIngress controller\tannotation nginx vô tác dụng\tUNKNOWN\n'
  printf 'rancher\tingress.tls.source\tUNKNOWN\tUNKNOWN\trancher\tcert-manager\tTLS resource sai owner\tUNKNOWN\n'
  printf 'rancher\tingress.tls.secretName\tUNKNOWN\tUNKNOWN\ttls-rancher-ingress\tcert-manager ingress-shim\tgate chờ sai resource\tUNKNOWN\n'
  printf 'rancher\tagentTLSMode\tUNKNOWN\tUNKNOWN\tstrict\tCA trust\tagent TLS không đúng policy\tUNKNOWN\n'
  printf 'rancher\tgateway.gatewayClass.name\tUNKNOWN\tUNKNOWN\tN/A\tGateway API CRDs và GatewayClass\tnhầm Gateway branch\tUNKNOWN\n'
} > "${BASELINE_ROOT}/config-contract.tsv"

nano "${BASELINE_ROOT}/config-contract.tsv"
```

Phải trả lời được cho từng dòng:

- Key nào gate việc có/không render resource?
- Default mới nào đang khiến manifest “tình cờ đúng”?
- Nhánh thay thế cần CRD/controller/GatewayClass nào?
- Resource do Helm render hay controller khác tạo bất đồng bộ?
- Template có `lookup` và phụ thuộc state cluster không?
- Chart pin image hay ghép tag từ `appVersion`?
- Annotation có dành riêng cho ingress controller khác không?

Gate contract:

```bash
set -o pipefail
(
  python3 - "${BASELINE_ROOT}/config-contract.tsv" "${CHART_EVIDENCE}" <<'PY'
import csv
import os
import sys

contract_file, evidence_root = sys.argv[1:]
rows = list(csv.DictReader(open(contract_file, encoding='utf-8'), delimiter='\t'))
required = {
    ('cert-manager', 'crds.enabled'),
    ('traefik', 'providers.kubernetesIngress.enabled'),
    ('traefik', 'providers.kubernetesGateway.enabled'),
    ('traefik', 'ingressClass.enabled'),
    ('rancher', 'networkExposure.type'),
    ('rancher', 'ingress.enabled'),
    ('rancher', 'ingress.ingressClassName'),
    ('rancher', 'ingress.includeDefaultExtraAnnotations'),
    ('rancher', 'ingress.tls.source'),
    ('rancher', 'ingress.tls.secretName'),
    ('rancher', 'agentTLSMode'),
    ('rancher', 'gateway.gatewayClass.name'),
}
found = {(row['component'], row['key_path']) for row in rows}
missing = required - found
if missing:
    raise SystemExit('FAIL: config contract thiếu ' + repr(sorted(missing)))

for row in rows:
    for key, value in row.items():
        if not value.strip() or 'UNKNOWN' in value:
            raise SystemExit(f'FAIL: {row["component"]}.{row["key_path"]} cột {key} chưa audit')
    evidence_ref = row['evidence']
    path_part, separator, line_part = evidence_ref.rpartition(':')
    if not separator or not line_part.isdigit():
        path_part = evidence_ref
    evidence_path = os.path.realpath(os.path.join(evidence_root, path_part))
    root_path = os.path.realpath(evidence_root)
    if os.path.commonpath([root_path, evidence_path]) != root_path:
        raise SystemExit(f'FAIL: evidence thoát chart workspace: {evidence_ref}')
    if not os.path.isfile(evidence_path):
        raise SystemExit(f'FAIL: evidence file không tồn tại: {evidence_ref}')

lookup = {(r['component'], r['key_path']): r for r in rows}
if lookup[('cert-manager', 'crds.enabled')]['pinned_value'] != 'true':
    raise SystemExit('FAIL: baseline yêu cầu cert-manager crds.enabled=true')
if lookup[('traefik', 'providers.kubernetesIngress.enabled')]['pinned_value'] != 'true':
    raise SystemExit('FAIL: baseline yêu cầu Traefik KubernetesIngress provider')
if lookup[('traefik', 'providers.kubernetesGateway.enabled')]['pinned_value'] != 'false':
    raise SystemExit('FAIL: baseline Ingress không được bật Gateway provider ngầm')
exposure = os.environ['RANCHER_EXPOSURE']
if lookup[('rancher', 'networkExposure.type')]['pinned_value'] != exposure:
    raise SystemExit('FAIL: Rancher exposure contract không khớp input')
if exposure == 'ingress' and lookup[('rancher', 'ingress.enabled')]['pinned_value'] != 'true':
    raise SystemExit('FAIL: ingress exposure nhưng ingress.enabled không true')

print('PASS: config contract đủ key, không còn UNKNOWN và khớp input architecture')
PY
) 2>&1 | tee "${AUDIT_EVIDENCE}/gate-82-config-contract.txt"
CONTRACT_RC=${PIPESTATUS[0]}
echo "config contract gate rc=${CONTRACT_RC}"
```

**PASS:** `rc=0`. **FAIL/STOP:** chưa tạo candidate values hoặc render.

Nếu exact chart đã đổi tên/xóa một key bắt buộc, giữ gate ở trạng thái FAIL, ghi breaking change vào decision log và cập nhật cả contract, candidate values lẫn render assertion dựa trên source official mới. Không xóa assertion chỉ để chart mới đi qua.

### 8.3. Case study bắt buộc: Rancher 2.14 và `networkExposure.type`

Đây là ví dụ lịch sử để học cách audit, **không phải khuyến nghị chọn Rancher 2.14.3 trong tương lai**.

Trong [`values.yaml` của Rancher chart v2.14.3](https://github.com/rancher/rancher/blob/v2.14.3/chart/values.yaml), `networkExposure.type` mặc định là `ingress`. [Helper của đúng tag v2.14.3](https://github.com/rancher/rancher/blob/v2.14.3/chart/templates/_helpers.tpl) chỉ bật nhánh Ingress khi đồng thời:

```gotemplate
eq .Values.networkExposure.type "ingress"
.Values.ingress.enabled
```

Nếu chỉ set `ingress.enabled: true`, manifest hiện vẫn đúng nhờ default mới. Đó là dependency ngầm và phải pin:

```yaml
networkExposure:
  type: ingress

ingress:
  enabled: true
  includeDefaultExtraAnnotations: false
  ingressClassName: traefik
  servicePort: 80
  tls:
    source: rancher
    secretName: tls-rancher-ingress

agentTLSMode: strict
```

`servicePort: 80` trong ví dụ là contract cho kiến trúc Traefik chuyển tiếp HTTP tới Rancher Service. Nếu baseline bật `service.disableHTTP`, chart yêu cầu backend port `443`; phải đổi contract và render gate tương ứng, không copy nguyên ví dụ.

Cùng chart còn có nhánh Gateway API và default `gateway.gatewayClass.name: traefik`. Tên này **không tự cài** Gateway API CRDs, GatewayClass hoặc một controller đã bật provider Gateway. Nếu kiến trúc chỉ cài Traefik Kubernetes Ingress, chọn nhầm `networkExposure.type: gateway` có thể render/fail vì thiếu dependency. Vì vậy contract phải pin `ingress` và render gate phải cấm `Gateway`/`HTTPRoute` ngoài dự kiến.

Case study này rút ra quy tắc tổng quát:

> Pin mọi value quyết định **loại resource hoặc nhánh controller**, kể cả khi default hiện tại đúng. Mỗi lần nâng chart, diff `values.yaml` và template trước khi tái sử dụng file values cũ.

### 8.4. Resource render-time khác runtime-time

Audit phải ghi owner và thời điểm xuất hiện của resource:

| Loại | Ví dụ | Hệ quả với gate |
| --- | --- | --- |
| Helm-rendered | Deployment/Ingress nằm trong output `helm template` | Có thể kiểm tra tĩnh ngay. |
| Controller-generated | cert-manager ingress-shim tạo `Certificate` từ annotation trên Ingress | Không được kỳ vọng có trong chart render; cần wait theo tên resource sau khi controller chạy. |
| Application-generated | Rancher server tạo bootstrap secret khi khởi động lần đầu nếu chart không render sẵn | Lệnh đọc secret sớm có thể `NotFound`, không chứng minh install fail. |
| Runtime-managed | `cattle-cluster-agent` do Rancher tạo/quản lý theo runtime | Không được giả định luôn tồn tại trong chart; phải discovery trước khi đặt log gate. |

Phân loại này không trực tiếp chọn version, nhưng nó ngăn một candidate tốt bị loại bởi gate sai và ngăn candidate xấu vượt qua vì Helm `--wait` không theo dõi controller khác.

Tạo resource ownership contract cho exact baseline:

```bash
{
  printf 'resource\towner\texpected_phase\tverification\n'
  printf 'Rancher Deployment/Ingress\tHelm\trender-time\thelm template inventory\n'
  printf 'Certificate tls-rancher-ingress\tcert-manager ingress-shim\truntime-after-Ingress\twait for create then Ready\n'
  printf 'bootstrap-secret\tRancher application when bootstrapPassword is unset\truntime-after-Rancher-ready\tget secret only after Pod Ready\n'
  printf 'cattle-cluster-agent\tRancher runtime controller\truntime/discovery\tget deployment inventory before logs\n'
} > "${BASELINE_ROOT}/resource-ownership.tsv"
```

### 8.5. Gate hoàn thành chart audit

```bash
set -o pipefail
(
  for gate in \
    gate-81-source-trees.txt \
    gate-82-config-contract.txt; do
    grep -Fq 'PASS:' "${AUDIT_EVIDENCE}/${gate}" || {
      echo "FAIL: ${gate} chưa PASS"; exit 1;
    }
  done

  test "$(awk 'NR>1 && NF {count++} END {print count+0}' \
    "${BASELINE_ROOT}/resource-ownership.tsv")" -ge 4 || {
    echo 'FAIL: resource ownership contract thiếu dòng'; exit 1;
  }
  ! grep -Eq 'UNKNOWN|TBD|YYYY|X\.Y' \
    "${BASELINE_ROOT}/config-contract.tsv" \
    "${BASELINE_ROOT}/resource-ownership.tsv" || {
    echo 'FAIL: chart audit còn placeholder'; exit 1;
  }

  echo 'PASS: exact chart sources, config contract và resource ownership đã audit xong'
) 2>&1 | tee "${AUDIT_EVIDENCE}/gate-85-chart-audit-complete.txt"
CHART_AUDIT_RC=${PIPESTATUS[0]}
echo "chart audit gate rc=${CHART_AUDIT_RC}"
```

**PASS:** `rc=0`. **FAIL/STOP:** không sang render gate.

---

## 9. Render gate trước khi phê duyệt baseline

### 9.1. Tạo exact candidate values

Baseline tham chiếu dùng Traefik Kubernetes Ingress, không bật Gateway API. Tạo ba values files từ config contract:

```bash
RENDER_EVIDENCE="${BASELINE_ROOT}/evidence/09-render"
set -a
source "${BASELINE_ROOT}/baseline.env"
set +a
source "${BASELINE_ROOT}/research-state.env"

tee "${RENDER_EVIDENCE}/cert-manager-values.yaml" >/dev/null <<'EOF'
crds:
  enabled: true
prometheus:
  enabled: false
EOF

tee "${RENDER_EVIDENCE}/traefik-values.yaml" >/dev/null <<'EOF'
providers:
  kubernetesIngress:
    enabled: true
  kubernetesGateway:
    enabled: false
ingressClass:
  enabled: true
  isDefaultClass: false
service:
  type: ClusterIP
EOF

tee "${RENDER_EVIDENCE}/rancher-values.yaml" >/dev/null <<'EOF'
hostname: rancher.example.com
replicas: 3
networkExposure:
  type: ingress
ingress:
  enabled: true
  includeDefaultExtraAnnotations: false
  ingressClassName: traefik
  servicePort: 80
  tls:
    source: rancher
    secretName: tls-rancher-ingress
agentTLSMode: strict
EOF
```

Gate values khớp contract:

```bash
set -o pipefail
(
  grep -Fq 'enabled: true' "${RENDER_EVIDENCE}/cert-manager-values.yaml" || {
    echo 'FAIL: cert-manager CRD contract thiếu'; exit 1;
  }
  grep -A2 'kubernetesGateway:' "${RENDER_EVIDENCE}/traefik-values.yaml" \
    | grep -Fq 'enabled: false' || {
    echo 'FAIL: Traefik Gateway provider chưa tắt'; exit 1;
  }
  grep -A1 'networkExposure:' "${RENDER_EVIDENCE}/rancher-values.yaml" \
    | grep -Fq 'type: ingress' || {
    echo 'FAIL: Rancher chưa pin networkExposure.type=ingress'; exit 1;
  }
  grep -Fq 'ingressClassName: traefik' "${RENDER_EVIDENCE}/rancher-values.yaml" || {
    echo 'FAIL: Rancher chưa pin Traefik IngressClass'; exit 1;
  }
  echo 'PASS: ba candidate values files khớp kiến trúc Ingress đã audit'
) 2>&1 | tee "${RENDER_EVIDENCE}/gate-91-candidate-values.txt"
VALUES_RC=${PIPESTATUS[0]}
echo "candidate values gate rc=${VALUES_RC}"
```

### 9.2. Render cả ba chart bằng exact Kubernetes version

```bash
helm template cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --version "${CERT_MANAGER_VERSION}" \
  --kube-version "${K8S_VERSION}" \
  --values "${RENDER_EVIDENCE}/cert-manager-values.yaml" \
  > "${RENDER_EVIDENCE}/cert-manager-rendered.yaml"

helm template traefik traefik/traefik \
  --namespace traefik \
  --version "${TRAEFIK_CHART_VERSION}" \
  --kube-version "${K8S_VERSION}" \
  --values "${RENDER_EVIDENCE}/traefik-values.yaml" \
  > "${RENDER_EVIDENCE}/traefik-rendered.yaml"

helm template rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version "${RANCHER_VERSION}" \
  --kube-version "${K8S_VERSION}" \
  --values "${RENDER_EVIDENCE}/rancher-values.yaml" \
  > "${RENDER_EVIDENCE}/rancher-rendered.yaml"

for manifest in "${RENDER_EVIDENCE}"/*-rendered.yaml; do
  sha256sum "${manifest}" > "${manifest}.sha256"
  awk '/^apiVersion:|^kind:|^[[:space:]]+image:/' "${manifest}" \
    > "${manifest}.inventory.txt"
done
```

`helm template` thành công là `RENDER-PASS`, không thay support matrix hoặc lab gate.

### 9.3. Gate cert-manager render

```bash
set -o pipefail
(
  MANIFEST="${RENDER_EVIDENCE}/cert-manager-rendered.yaml"
  test -s "${MANIFEST}" || { echo 'FAIL: cert-manager render rỗng'; exit 1; }
  for expected in \
    'kind: CustomResourceDefinition' \
    'kind: Deployment' \
    'name: cert-manager-webhook' \
    'name: cert-manager-cainjector'; do
    grep -Fq "${expected}" "${MANIFEST}" || {
      echo "FAIL: cert-manager thiếu ${expected}"; exit 1;
    }
  done
  [[ "$(grep -c '^kind: CustomResourceDefinition$' "${MANIFEST}")" -ge 6 ]] || {
    echo 'FAIL: số cert-manager CRD thấp bất thường'; exit 1;
  }
  ! grep -Eqi 'image:.*:(latest|stable)(@|$)' "${MANIFEST}" || {
    echo 'FAIL: cert-manager render có tag động'; exit 1;
  }
  echo 'PASS: cert-manager render có CRDs/controllers/webhook và không có tag động'
) 2>&1 | tee "${RENDER_EVIDENCE}/gate-93-cert-manager-render.txt"
CM_RENDER_RC=${PIPESTATUS[0]}
echo "cert-manager render gate rc=${CM_RENDER_RC}"
```

### 9.4. Gate Traefik render

```bash
set -o pipefail
(
  MANIFEST="${RENDER_EVIDENCE}/traefik-rendered.yaml"
  for expected in 'kind: Deployment' 'kind: Service' 'kind: IngressClass'; do
    grep -Fq "${expected}" "${MANIFEST}" || {
      echo "FAIL: Traefik thiếu ${expected}"; exit 1;
    }
  done
  grep -Fqi 'kubernetesingress' "${MANIFEST}" || {
    echo 'FAIL: Traefik chưa bật Kubernetes Ingress provider'; exit 1;
  }
  if grep -Eqi -- '--providers\.kubernetesgateway($|=true)' "${MANIFEST}" || \
     grep -Eq '^kind: (Gateway|GatewayClass|HTTPRoute)$' "${MANIFEST}"; then
    echo 'FAIL: Traefik render có Gateway API ngoài contract'; exit 1
  fi
  ! grep -Eqi 'image:.*:(latest|stable)(@|$)' "${MANIFEST}" || {
    echo 'FAIL: Traefik render có tag động'; exit 1;
  }
  echo 'PASS: Traefik Ingress provider/class/service đúng, Gateway API tắt, image pin'
) 2>&1 | tee "${RENDER_EVIDENCE}/gate-94-traefik-render.txt"
TRAEFIK_RENDER_RC=${PIPESTATUS[0]}
echo "Traefik render gate rc=${TRAEFIK_RENDER_RC}"
```

### 9.5. Gate Rancher render an toàn cho SSH

```bash
set -o pipefail
(
  MANIFEST="${RENDER_EVIDENCE}/rancher-rendered.yaml"
  grep -q '^kind: Ingress$' "${MANIFEST}" || {
    echo 'FAIL: chart không render Ingress'; exit 1;
  }

  for expected in \
    'ingressClassName: traefik' \
    'host: rancher.example.com' \
    'secretName: tls-rancher-ingress'; do
    grep -Fq "${expected}" "${MANIFEST}" || {
      echo "FAIL: Rancher manifest thiếu ${expected}"; exit 1;
    }
  done

  if grep -Eq '^kind: (Gateway|HTTPRoute)$' "${MANIFEST}"; then
    echo 'FAIL: Rancher chart đã đi vào nhánh Gateway API'; exit 1
  fi
  ! grep -Eqi 'image:.*:(latest|stable)(@|$)' "${MANIFEST}" || {
    echo 'FAIL: Rancher render có tag động'; exit 1;
  }

  echo 'PASS: Rancher Ingress mode, class/host/TLS đúng; không có Gateway/HTTPRoute'
) 2>&1 | tee "${RENDER_EVIDENCE}/gate-95-rancher-render.txt"
RANCHER_RENDER_RC=${PIPESTATUS[0]}
echo "Rancher render gate rc=${RANCHER_RENDER_RC}"
```

`exit 1` chỉ thoát subshell; phiên SSH còn nguyên và exit code vẫn được lưu.

### 9.6. Resolve digest cho toàn bộ stack

```bash
source "${BASELINE_ROOT}/research-state.env"
ALL_DIGESTS="${RENDER_EVIDENCE}/all-image-digests.tsv"
printf 'component\timage\tdigest\n' > "${ALL_DIGESTS}"

record_image() {
  local component="$1" image="$2" digest
  digest="$(IMAGE_INSPECTOR="${IMAGE_INSPECTOR}" \
    "${BASELINE_ROOT}/resolve-image-digest.sh" "${image}")" || return 1
  printf '%s\t%s\t%s\n' "${component}" "${image}" "${digest}" \
    >> "${ALL_DIGESTS}"
}

while read -r image; do
  [[ -n "${image}" ]] && record_image kubeadm "${image}"
done < "${K8S_EVIDENCE}/kubeadm-images-${K8S_VERSION}.txt"

for spec in \
  "flannel:${CNI_EVIDENCE}/kube-flannel-candidate.yml" \
  "cert-manager:${RENDER_EVIDENCE}/cert-manager-rendered.yaml" \
  "traefik:${RENDER_EVIDENCE}/traefik-rendered.yaml" \
  "rancher:${RENDER_EVIDENCE}/rancher-rendered.yaml"; do
  component="${spec%%:*}"
  manifest="${spec#*:}"
  sed -n -E "s/^[[:space:]]*image:[[:space:]]*['\"]?([^'\"[:space:]]+)['\"]?.*/\1/p" \
    "${manifest}" | sort -u | while read -r image; do
      record_image "${component}" "${image}"
    done
done

if [[ -f "${ADDON_EVIDENCE}/addon-image-digests.tsv" ]]; then
  tail -n +2 "${ADDON_EVIDENCE}/addon-image-digests.tsv" \
    >> "${ALL_DIGESTS}"
fi
```

Gate digest:

```bash
set -o pipefail
(
  test "$(awk 'NR>1 && NF {count++} END {print count+0}' "${ALL_DIGESTS}")" -gt 0 || {
    echo 'FAIL: image inventory rỗng'; exit 1;
  }
  while IFS=$'\t' read -r component image digest; do
    [[ "${component}" == component ]] && continue
    [[ "${image}" != *':latest' && "${image}" != *':stable' ]] || {
      echo "FAIL: ${component} dùng tag động ${image}"; exit 1;
    }
    [[ "${digest}" =~ ^sha256:[0-9a-f]{64}$ ]] || {
      echo "FAIL: digest không hợp lệ ${component} ${image} ${digest}"; exit 1;
    }
  done < "${ALL_DIGESTS}"
  echo 'PASS: mọi image của kubeadm/CNI/charts/addons có exact ref và SHA256 digest'
) 2>&1 | tee "${RENDER_EVIDENCE}/gate-96-image-digests.txt"
DIGEST_RC=${PIPESTATUS[0]}
echo "image digest gate rc=${DIGEST_RC}"
```

### 9.7. Server dry-run và lab-status gate

Chạy trên disposable cluster đúng `K8S_VERSION`. Không dùng cluster production. Cluster phải có dependency CRDs/controller theo đúng thứ tự của runbook lab; server dry-run không tự persist CRDs từ document trước.

```bash
kubectl version > "${RENDER_EVIDENCE}/lab-kubectl-version.txt"

for manifest in \
  "${RENDER_EVIDENCE}/cert-manager-rendered.yaml" \
  "${RENDER_EVIDENCE}/traefik-rendered.yaml" \
  "${RENDER_EVIDENCE}/rancher-rendered.yaml"; do
  kubectl apply --dry-run=server -f "${manifest}" \
    > "${manifest}.server-dry-run.txt"
done
```

Sau khi workflow lab đã cài đúng candidates, chạy gate read-only sau. `set -e` nằm trong subshell nên lỗi không đóng SSH và không thể in `OVERALL: PASS` giả:

```bash
set -o pipefail
(
  set -euo pipefail

  echo '== Kubernetes/node versions =='
  kubectl get nodes -o wide
  BAD_KUBELETS="$(kubectl get nodes \
    -o jsonpath='{range .items[*]}{.status.nodeInfo.kubeletVersion}{"\n"}{end}' \
    | awk -v expected="v${K8S_VERSION}" '$0 != expected {bad++} END {print bad+0}')"
  test "${BAD_KUBELETS}" -eq 0
  kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{end}'

  echo '== Flannel =='
  kubectl -n kube-flannel rollout status daemonset/kube-flannel-ds --timeout=300s

  echo '== cert-manager =='
  kubectl -n cert-manager rollout status deploy/cert-manager --timeout=300s
  kubectl -n cert-manager rollout status deploy/cert-manager-webhook --timeout=300s
  kubectl -n cert-manager rollout status deploy/cert-manager-cainjector --timeout=300s

  echo '== Traefik =='
  kubectl -n traefik rollout status deploy/traefik --timeout=300s
  kubectl get ingressclass traefik

  echo '== Rancher =='
  kubectl -n cattle-system rollout status deploy/rancher --timeout=600s
  kubectl -n cattle-system get ingress rancher \
    -o custom-columns=CLASS:.spec.ingressClassName,HOST:.spec.rules[0].host,TLS:.spec.tls[0].secretName
  kubectl -n cattle-system wait --for=create \
    certificate/tls-rancher-ingress --timeout=120s
  kubectl -n cattle-system wait --for=condition=Ready \
    certificate/tls-rancher-ingress --timeout=300s

  if [[ "${LOCAL_PATH_VERSION}" != N/A ]]; then
    kubectl -n local-path-storage rollout status \
      deploy/local-path-provisioner --timeout=300s
  fi
  if [[ "${METALLB_VERSION}" != N/A ]]; then
    kubectl -n metallb-system rollout status deploy/controller --timeout=300s
    kubectl -n metallb-system rollout status daemonset/speaker --timeout=300s
  fi

  echo 'OVERALL: PASS'
) 2>&1 | tee "${RENDER_EVIDENCE}/lab-smoke-report.txt"
LAB_SMOKE_RC=${PIPESTATUS[0]}
echo "lab smoke gate rc=${LAB_SMOKE_RC}"
```

Gate:

```bash
set -o pipefail
(
  grep -Fq "Server Version: v${K8S_VERSION}" \
    "${RENDER_EVIDENCE}/lab-kubectl-version.txt" || {
    echo 'FAIL: lab Kubernetes version không khớp candidate'; exit 1;
  }
  for report in "${RENDER_EVIDENCE}"/*.server-dry-run.txt; do
    test -s "${report}" || {
      echo "FAIL: server dry-run evidence rỗng ${report}"; exit 1;
    }
  done
  test -s "${RENDER_EVIDENCE}/lab-smoke-report.txt" || {
    echo 'FAIL: thiếu lab-smoke-report.txt'; exit 1;
  }
  grep -Fq 'OVERALL: PASS' "${RENDER_EVIDENCE}/lab-smoke-report.txt" || {
    echo 'FAIL: lab smoke report chưa OVERALL: PASS'; exit 1;
  }
  echo 'PASS: exact-version server dry-run và lab smoke report đều đạt'
) 2>&1 | tee "${RENDER_EVIDENCE}/gate-97-lab.txt"
LAB_RC=${PIPESTATUS[0]}
echo "lab gate rc=${LAB_RC}"
```

Trạng thái evidence chỉ được đi theo thứ tự `NOT-TESTED` → `METADATA-PASS` → `RENDER-PASS` → `SERVER-DRY-RUN-PASS` → `LAB-PASS`. `LAB-PASS` không tự biến thành `VENDOR-SUPPORTED`.

---

## 10. Bảng quyết định và bảng baseline

### 10.1. Hoàn thiện decision/rejection log

Không xóa candidate bị loại. Bảo đảm file có header và ít nhất một decision cho mỗi component:

```bash
if ! grep -Fq '| Date | Component | Candidate | Decision | Reason | Revisit when |' \
  "${BASELINE_ROOT}/decision-log.md"; then
  tmp="${BASELINE_ROOT}/decision-log.tmp"
  {
    printf '| Date | Component | Candidate | Decision | Reason | Revisit when |\n'
    printf '| --- | --- | --- | --- | --- | --- |\n'
    cat "${BASELINE_ROOT}/decision-log.md"
  } > "${tmp}"
  mv "${tmp}" "${BASELINE_ROOT}/decision-log.md"
fi

nano "${BASELINE_ROOT}/decision-log.md"
```

Gate:

```bash
set -o pipefail
(
  for component in Kubernetes containerd Flannel Helm cert-manager Traefik Rancher; do
    grep -Fqi "| ${component} " "${BASELINE_ROOT}/decision-log.md" || {
      echo "FAIL: decision log thiếu ${component}"; exit 1;
    }
  done
  ! grep -Eq '\.\.\.|UNKNOWN|TBD|chưa đánh giá' \
    "${BASELINE_ROOT}/decision-log.md" || {
    echo 'FAIL: decision log còn placeholder'; exit 1;
  }
  echo 'PASS: decision/rejection log đủ component và không còn placeholder'
) 2>&1 | tee "${BASELINE_ROOT}/evidence/12-final/gate-101-decision-log.txt"
DECISION_LOG_RC=${PIPESTATUS[0]}
echo "decision log gate rc=${DECISION_LOG_RC}"
```

### 10.2. Tạo và kiểm chứng source registry

Tạo machine-readable source registry; một URL chỉ được tính khi có claim và evidence file:

```bash
{
  printf 'component\tevidence_type\tofficial_url\tpage_version\taccessed_at\tclaim\tevidence_file\n'
  for component in Kubernetes Rancher containerd Flannel Helm cert-manager Traefik cloudflared local-path-provisioner MetalLB; do
    printf '%s\tNOT-RECORDED\thttps://NOT-RECORDED\tNOT-RECORDED\t%s\tNOT-RECORDED\tNOT-RECORDED\n' \
      "${component}" "${RESEARCH_DATE}"
  done
} > "${BASELINE_ROOT}/sources.tsv"

nano "${BASELINE_ROOT}/sources.tsv"
```

`evidence_file` là đường dẫn tương đối dưới `${BASELINE_ROOT}`. Gate:

```bash
set -o pipefail
(
  python3 - "${BASELINE_ROOT}/sources.tsv" "${BASELINE_ROOT}" <<'PY'
import csv
import os
import sys
from urllib.parse import urlparse

source_file, root = sys.argv[1:]
rows = list(csv.DictReader(open(source_file, encoding='utf-8'), delimiter='\t'))
required = {'Kubernetes','Rancher','containerd','Flannel','Helm','cert-manager','Traefik'}
found = {row['component'] for row in rows}
if not required <= found:
    raise SystemExit('FAIL: sources.tsv thiếu component bắt buộc')
for row in rows:
    if any(not value.strip() or 'NOT-RECORDED' in value for value in row.values()):
        raise SystemExit(f'FAIL: source row {row["component"]} chưa hoàn thiện')
    parsed = urlparse(row['official_url'])
    if parsed.scheme != 'https' or not parsed.netloc:
        raise SystemExit(f'FAIL: URL không phải HTTPS {row["component"]}')
    evidence = os.path.realpath(os.path.join(root, row['evidence_file']))
    if os.path.commonpath([os.path.realpath(root), evidence]) != os.path.realpath(root):
        raise SystemExit(f'FAIL: evidence path thoát workspace {row["component"]}')
    if not os.path.isfile(evidence) or os.path.getsize(evidence) == 0:
        raise SystemExit(f'FAIL: evidence file thiếu/rỗng {row["component"]}: {evidence}')
print('PASS: source registry đủ official URL, claim và evidence file')
PY
) 2>&1 | tee "${BASELINE_ROOT}/evidence/12-final/gate-102-sources.txt"
SOURCES_RC=${PIPESTATUS[0]}
echo "sources gate rc=${SOURCES_RC}"
```

Sinh `sources.md`:

```bash
python3 - "${BASELINE_ROOT}/sources.tsv" "${BASELINE_ROOT}/sources.md" <<'PY'
import csv, sys
rows = list(csv.DictReader(open(sys.argv[1], encoding='utf-8'), delimiter='\t'))
with open(sys.argv[2], 'w', encoding='utf-8') as out:
    out.write('| Component | Evidence type | Official URL | Page/release | Accessed | Claim | Evidence |\n')
    out.write('| --- | --- | --- | --- | --- | --- | --- |\n')
    for r in rows:
        out.write(f'| {r["component"]} | {r["evidence_type"]} | {r["official_url"]} | {r["page_version"]} | {r["accessed_at"]} | {r["claim"]} | {r["evidence_file"]} |\n')
PY
```

### 10.3. Tạo source-of-truth baseline data

Khởi tạo trạng thái `NOT-TESTED`; không copy bảng có `PASS` sẵn:

```bash
{
  printf 'component\trole\texact_version\tartifact\tk8s_range\tlifecycle\tsupport_class\texplicit_contract\tevidence\tofficial_source\n'
  printf 'OS\tNode OS\tNOT-SET\tNOT-SET\tN/A\tNOT-SET\tNOT-SET\tarch/kernel/cgroup\tNOT-TESTED\tNOT-SET\n'
  printf 'Kubernetes\tControl plane/node\tNOT-SET\tNOT-SET\tN/A\tNOT-SET\tUPSTREAM-MAINTAINED\tsame exact patch\tNOT-TESTED\tNOT-SET\n'
  printf 'kubeadm-images\tControl-plane dependencies\tNOT-SET\tNOT-SET\tN/A\tKubernetes lifecycle\tKUBEADM-SELECTED\tno independent upgrade\tNOT-TESTED\tNOT-SET\n'
  printf 'containerd\tCRI runtime\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\tCRI v1/systemd\tNOT-TESTED\tNOT-SET\n'
  printf 'Flannel\tPod network\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\tUPSTREAM-RANGE-NOT-STATED\tpodCIDR/CNI spec\tNOT-TESTED\tNOT-SET\n'
  printf 'Helm\tChart client\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\tSUPPORTED\ttoolchain exact version\tNOT-TESTED\tNOT-SET\n'
  printf 'Traefik\tIngress\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\tIngress provider/class\tNOT-TESTED\tNOT-SET\n'
  printf 'cert-manager\tTLS controller\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\tCRDs/webhook\tNOT-TESTED\tNOT-SET\n'
  printf 'Rancher\tCluster manager\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\texposure/TLS/agent mode\tNOT-TESTED\tNOT-SET\n'
  printf 'cloudflared\tTunnel\tNOT-SET\tNOT-SET\tN/A\tNOT-SET\tUPSTREAM-RANGE-NOT-STATED\texact image digest\tNOT-TESTED\tNOT-SET\n'
  printf 'local-path-provisioner\tStorage\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\tUPSTREAM-RANGE-NOT-STATED\tclass/path\tNOT-TESTED\tNOT-SET\n'
  printf 'MetalLB\tLAN LoadBalancer\tNOT-SET\tNOT-SET\tNOT-SET\tNOT-SET\tUPSTREAM-RANGE-NOT-STATED\tCRDs/IP pool\tNOT-TESTED\tNOT-SET\n'
} > "${BASELINE_ROOT}/baseline-data.tsv"

nano "${BASELINE_ROOT}/baseline-data.tsv"
```

Điền từ `research-state.env`, `all-image-digests.tsv`, lifecycle evidence và các gate. Component không được chọn phải dùng `N/A` ở mọi field thích hợp và `evidence=N/A`; không xóa row.

### 10.4. Gate dữ liệu và sinh bảng §2.1

```bash
set -a
source "${BASELINE_ROOT}/baseline.env"
source "${BASELINE_ROOT}/research-state.env"
set +a

set -o pipefail
(
  python3 - "${BASELINE_ROOT}/baseline-data.tsv" <<'PY'
import csv
import os
import re
import sys

rows = list(csv.DictReader(open(sys.argv[1], encoding='utf-8'), delimiter='\t'))
expected = {'OS','Kubernetes','kubeadm-images','containerd','Flannel','Helm',
            'Traefik','cert-manager','Rancher','cloudflared',
            'local-path-provisioner','MetalLB'}
if {r['component'] for r in rows} != expected:
    raise SystemExit('FAIL: baseline-data.tsv thiếu/thừa component')

bad_markers = ('NOT-SET','NOT-TESTED','UNKNOWN','TBD','YYYY','...')
for row in rows:
    if any(marker in value for value in row.values() for marker in bad_markers):
        raise SystemExit(f'FAIL: {row["component"]} còn placeholder')
    if re.search(r'(^|[:/])(latest|stable|master)(@|$|/)', row['artifact'], re.I):
        raise SystemExit(f'FAIL: {row["component"]} còn tag/branch động')
    if row['evidence'] != 'N/A' and not row['evidence'].endswith('-PASS'):
        raise SystemExit(f'FAIL: evidence status không PASS cho {row["component"]}')
    if row['official_source'] != 'N/A' and not row['official_source'].startswith('https://'):
        raise SystemExit(f'FAIL: official source không phải HTTPS cho {row["component"]}')

lookup = {r['component']: r for r in rows}
checks = {
    'Kubernetes': 'v' + os.environ['K8S_VERSION'],
    'containerd': os.environ['CONTAINERD_UPSTREAM_VERSION'],
    'Flannel': os.environ['FLANNEL_VERSION'],
    'Helm': os.environ['HELM_VERSION'],
    'Traefik': os.environ['TRAEFIK_CHART_VERSION'],
    'cert-manager': os.environ['CERT_MANAGER_VERSION'],
    'Rancher': os.environ['RANCHER_VERSION'],
}
for component, selected in checks.items():
    if lookup[component]['exact_version'] != selected:
        raise SystemExit(f'FAIL: {component} baseline={lookup[component]["exact_version"]}, selected={selected}')

print('PASS: baseline data đủ exact versions, support/evidence/source và khớp research state')
PY
) 2>&1 | tee "${BASELINE_ROOT}/evidence/12-final/gate-104-baseline-data.txt"
BASELINE_DATA_RC=${PIPESTATUS[0]}
echo "baseline data gate rc=${BASELINE_DATA_RC}"
```

Sinh bảng Markdown chi tiết và bảng rút gọn dùng cho §2.1:

```bash
python3 - "${BASELINE_ROOT}/baseline-data.tsv" \
  "${BASELINE_ROOT}/baseline-detailed.md" \
  "${BASELINE_ROOT}/baseline-versions.md" <<'PY'
import csv, os, sys
rows = list(csv.DictReader(open(sys.argv[1], encoding='utf-8'), delimiter='\t'))

with open(sys.argv[2], 'w', encoding='utf-8') as out:
    out.write('| Component | Role | Exact version | Artifact | K8s range | Lifecycle | Support class | Contract | Evidence | Official source |\n')
    out.write('| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |\n')
    for r in rows:
        out.write('| ' + ' | '.join(r[k] for k in r) + ' |\n')

with open(sys.argv[3], 'w', encoding='utf-8') as out:
    out.write('## 2.1. Phiên bản\n\n')
    out.write('| Thành phần | Phiên bản dùng trong runbook | Ghi chú tương thích |\n')
    out.write('| --- | --- | --- |\n')
    for r in rows:
        if r['exact_version'] == 'N/A':
            continue
        note = f'{r["artifact"]}; {r["support_class"]}; {r["evidence"]}'
        out.write(f'| {r["component"]} | **{r["exact_version"]}** | {note} |\n')
    out.write('\n> Version skew: kubeadm/kubelet/kubectl được pin cùng exact Debian version.\n')
    out.write(f'> Phạm vi cam kết: {os.environ.get("SUPPORT_GOAL", "UNSET")}.\n')
PY

python3 - "${BASELINE_ROOT}/compatibility-matrix.tsv" \
  "${BASELINE_ROOT}/compatibility-matrix.md" <<'PY'
import csv, sys
rows = list(csv.DictReader(open(sys.argv[1], encoding='utf-8'), delimiter='\t'))
with open(sys.argv[2], 'w', encoding='utf-8') as out:
    out.write('| K8s minor | Upstream | Rancher Manager | Rancher imported | cert-manager | Helm | Rancher chart | Decision | Reason |\n')
    out.write('| --- | --- | --- | --- | --- | --- | --- | --- | --- |\n')
    for r in rows:
        out.write('| ' + ' | '.join(r[k] for k in r) + ' |\n')
PY
```

**PASS:** `baseline data gate rc=0`, hai bảng được sinh từ cùng TSV. Không sửa tay bảng Markdown; sửa TSV, chạy gate rồi sinh lại.

---

## 11. Freshness gate

### 11.1. Tính review date

Nhập EOL gần nhất trong các component có lifecycle. Policy mẫu yêu cầu review lại ít nhất 90 ngày trước EOL và trong vòng 30 ngày trước ngày cài nếu baseline được lập quá sớm.

```bash
read -r -p 'Nhập NEAREST_EOL (YYYY-MM-DD): ' NEAREST_EOL
EOL_REVIEW_BUFFER_DAYS=90

TARGET_MINUS_30="$(date -d "${TARGET_INSTALL_DATE} -30 days" +%F)"
if [[ "$(date -d "${TARGET_MINUS_30}" +%s)" -gt "$(date -d "${RESEARCH_DATE}" +%s)" ]]; then
  INSTALL_REVIEW_DATE="${TARGET_MINUS_30}"
else
  INSTALL_REVIEW_DATE="${TARGET_INSTALL_DATE}"
fi

EOL_REVIEW_DATE="$(date -d "${NEAREST_EOL} -${EOL_REVIEW_BUFFER_DAYS} days" +%F)"

REVIEW_BY="$(printf '%s\n%s\n' "${INSTALL_REVIEW_DATE}" "${EOL_REVIEW_DATE}" \
  | sort | head -n1)"

{
  printf 'BASELINE_STATUS=DRAFT\n'
  printf 'RESEARCHED_AT=%s\n' "${RESEARCH_DATE}"
  printf 'APPROVED_AT=NOT-APPROVED\n'
  printf 'TARGET_INSTALL_DATE=%s\n' "${TARGET_INSTALL_DATE}"
  printf 'REVIEW_BY=%s\n' "${REVIEW_BY}"
  printf 'NEAREST_EOL=%s\n' "${NEAREST_EOL}"
  printf 'SUPPORT_GOAL=%s\n' "${SUPPORT_GOAL}"
} > "${BASELINE_ROOT}/baseline-metadata.env"

cat "${BASELINE_ROOT}/baseline-metadata.env"
```

### 11.2. Gate freshness

```bash
set -o pipefail
(
  set -a
  source "${BASELINE_ROOT}/baseline-metadata.env"
  set +a

  for value in RESEARCHED_AT TARGET_INSTALL_DATE REVIEW_BY NEAREST_EOL; do
    date -d "${!value}" '+%F' >/dev/null 2>&1 || {
      echo "FAIL: ${value} không phải ngày hợp lệ"; exit 1;
    }
  done
  [[ "${BASELINE_STATUS}" == DRAFT ]] || {
    echo 'FAIL: status phải là DRAFT trước final approval'; exit 1;
  }
  [[ "$(date -d "${NEAREST_EOL}" +%s)" -gt "$(date -d "${TARGET_INSTALL_DATE}" +%s)" ]] || {
    echo 'FAIL: component hết hạn trước ngày cài dự kiến'; exit 1;
  }
  [[ "$(date -d "${REVIEW_BY}" +%s)" -ge "$(date -d "${RESEARCHED_AT}" +%s)" ]] || {
    echo 'FAIL: baseline đã stale ngay khi lập; chọn candidate có lifecycle dài hơn'; exit 1;
  }
  [[ "$(date -d "${REVIEW_BY}" +%s)" -le "$(date -d "${TARGET_INSTALL_DATE}" +%s)" ]] || {
    echo 'FAIL: review_by nằm sau ngày cài'; exit 1;
  }
  echo "PASS: baseline DRAFT hợp lệ tới ${REVIEW_BY}; nearest EOL=${NEAREST_EOL}"
) 2>&1 | tee "${BASELINE_ROOT}/evidence/12-final/gate-112-freshness.txt"
FRESHNESS_RC=${PIPESTATUS[0]}
echo "freshness gate rc=${FRESHNESS_RC}"
```

Sau khi phát hành, nếu ngày hiện tại lớn hơn `REVIEW_BY`, trạng thái vận hành phải coi là `STALE` dù file vẫn ghi `APPROVED`. Phải chạy lại toàn bộ giao version và chart audit, không chỉ sửa một ô patch.

---

## 12. Phê duyệt và phát hành

### 12.1. Checklist phê duyệt cuối

### Nguồn và lifecycle

- [ ] Mọi row có ít nhất một official source và ngày truy cập.
- [ ] Kubernetes minor còn maintained và exact patch đã kiểm release notes.
- [ ] Mọi component có lifecycle/EOL hoặc ghi rõ upstream không công bố.
- [ ] Không có prerelease, `latest`, `stable`, `master`, version range hoặc image tag rỗng trong baseline cuối.
- [ ] Chart version, appVersion, package version và image digest không bị gộp.

### Compatibility

- [ ] Rancher Manager host và downstream/imported được đánh giá riêng.
- [ ] Kubernetes target nằm trong giao của Rancher, cert-manager, Helm và chart metadata.
- [ ] kubeadm/kubelet/kubectl có cùng exact package version cho cluster mới.
- [ ] Đã lưu inventory và digest của etcd/CoreDNS/pause/control-plane images do đúng kubeadm version chọn.
- [ ] Runtime hỗ trợ CRI v1; CRI enabled; cgroup driver đồng nhất.
- [ ] CNI release, CNI spec, CNI binaries và podCIDR đã được kiểm riêng.
- [ ] Component không có official compatibility range được gắn nhãn, không được suy diễn “supported”.

### Chart/config contract

- [ ] Đã lưu `Chart.yaml`, full `values.yaml`, templates/CRDs và checksum đúng version.
- [ ] Đã audit các nhánh `if`, `Capabilities`, `lookup` và cross-controller resources.
- [ ] Mọi key quyết định exposure/controller/TLS/storage/security đều pin rõ.
- [ ] Rancher pin `networkExposure.type` và render gate cấm nhánh Gateway ngoài dự kiến.
- [ ] Không giả định annotation của nginx có tác dụng với Traefik.
- [ ] Đã phân biệt resource do Helm, controller và application tạo.

### Validation và phát hành

- [ ] Tất cả chart render bằng exact `--kube-version` và exact values.
- [ ] Gate lỗi chạy trong subshell, trả non-zero mà không đóng SSH.
- [ ] Candidate/rejection log đầy đủ.
- [ ] Lab report tồn tại cho mọi tổ hợp cần `LAB-PASS`.
- [ ] Bảng chi tiết và bảng §2.1 rút gọn khớp nhau.
- [ ] Có `researched_at`, `review_by`, `nearest_eol` và support class.

Chỉ khi toàn bộ mục bắt buộc đạt mới đổi `baseline_status` thành `APPROVED`.

Sao chép checklist trên thành `${BASELINE_ROOT}/approval-checklist.md`, đánh dấu `[x]` sau khi đối chiếu evidence. Gate human checklist:

```bash
set -o pipefail
(
  test -s "${BASELINE_ROOT}/approval-checklist.md" || {
    echo 'FAIL: thiếu approval-checklist.md'; exit 1;
  }
  ! grep -Fq -- '- [ ]' "${BASELINE_ROOT}/approval-checklist.md" || {
    echo 'FAIL: checklist còn mục chưa xác nhận'; exit 1;
  }
  echo 'PASS: human approval checklist đã hoàn tất'
) 2>&1 | tee "${BASELINE_ROOT}/evidence/12-final/gate-121-checklist.txt"
CHECKLIST_RC=${PIPESTATUS[0]}
echo "checklist gate rc=${CHECKLIST_RC}"
```

### 12.2. Gate tự động toàn bộ pipeline

```bash
source "${BASELINE_ROOT}/research-session.env"
set -a
source "${BASELINE_ROOT}/baseline.env"
source "${BASELINE_ROOT}/research-state.env"
set +a
ADDON_EVIDENCE="${CHART_EVIDENCE}/addons"
AUDIT_EVIDENCE="${CHART_EVIDENCE}/audit"

set -o pipefail
(
  required_gates=(
    "${BASELINE_ROOT}/evidence/00-preflight/gate-02-tools.txt"
    "${BASELINE_ROOT}/evidence/00-preflight/gate-03-workspace.txt"
    "${BASELINE_ROOT}/evidence/00-preflight/gate-02-input.txt"
    "${BASELINE_ROOT}/evidence/00-preflight/gate-33-anchor.txt"
    "${K8S_EVIDENCE}/gate-41-upstream.txt"
    "${K8S_EVIDENCE}/gate-42-intersection.txt"
    "${K8S_EVIDENCE}/gate-421-rancher-discovery.txt"
    "${K8S_EVIDENCE}/gate-431-repository.txt"
    "${K8S_EVIDENCE}/gate-43-packages.txt"
    "${K8S_EVIDENCE}/gate-44-version-skew.txt"
    "${K8S_EVIDENCE}/gate-45-images.txt"
    "${RUNTIME_EVIDENCE}/gate-51-os.txt"
    "${RUNTIME_EVIDENCE}/gate-52-containerd-metadata.txt"
    "${RUNTIME_EVIDENCE}/gate-53-runtime.txt"
    "${CNI_EVIDENCE}/gate-61-flannel-discovery.txt"
    "${CNI_EVIDENCE}/gate-62-flannel-static.txt"
    "${CNI_EVIDENCE}/gate-63-flannel-server.txt"
    "${CHART_EVIDENCE}/gate-71-helm.txt"
    "${CHART_EVIDENCE}/gate-72-cert-manager.txt"
    "${CHART_EVIDENCE}/gate-73-traefik-metadata.txt"
    "${CHART_EVIDENCE}/gate-74-rancher-metadata.txt"
    "${ADDON_EVIDENCE}/gate-751-image-tool.txt"
    "${ADDON_EVIDENCE}/gate-752-addon-selection.txt"
    "${ADDON_EVIDENCE}/gate-753-addon-static.txt"
    "${AUDIT_EVIDENCE}/gate-85-chart-audit-complete.txt"
    "${RENDER_EVIDENCE}/gate-91-candidate-values.txt"
    "${RENDER_EVIDENCE}/gate-93-cert-manager-render.txt"
    "${RENDER_EVIDENCE}/gate-94-traefik-render.txt"
    "${RENDER_EVIDENCE}/gate-95-rancher-render.txt"
    "${RENDER_EVIDENCE}/gate-96-image-digests.txt"
    "${RENDER_EVIDENCE}/gate-97-lab.txt"
    "${FINAL_EVIDENCE}/gate-101-decision-log.txt"
    "${FINAL_EVIDENCE}/gate-102-sources.txt"
    "${FINAL_EVIDENCE}/gate-104-baseline-data.txt"
    "${FINAL_EVIDENCE}/gate-112-freshness.txt"
    "${FINAL_EVIDENCE}/gate-121-checklist.txt"
  )

  for gate in "${required_gates[@]}"; do
    test -s "${gate}" || {
      echo "FAIL: gate file thiếu/rỗng ${gate}"; exit 1;
    }
    grep -Fq 'PASS:' "${gate}" || {
      echo "FAIL: gate chưa PASS ${gate}"; exit 1;
    }
  done

  for final_file in \
    baseline-versions.md baseline-detailed.md compatibility-matrix.tsv compatibility-matrix.md \
    sources.md sources.tsv decision-log.md config-contract.tsv \
    resource-ownership.tsv baseline-metadata.env approval-checklist.md; do
    test -s "${BASELINE_ROOT}/${final_file}" || {
      echo "FAIL: final artifact thiếu/rỗng ${final_file}"; exit 1;
    }
  done

  ! grep -Eqi 'NOT-SET|NOT-TESTED|NOT-RECORDED|UNKNOWN|TBD|YYYY|\.\.\.' \
    "${BASELINE_ROOT}/baseline-versions.md" \
    "${BASELINE_ROOT}/baseline-detailed.md" \
    "${BASELINE_ROOT}/sources.md" \
    "${BASELINE_ROOT}/decision-log.md" \
    "${BASELINE_ROOT}/config-contract.tsv" || {
    echo 'FAIL: final artifacts còn placeholder'; exit 1;
  }

  echo 'PASS: toàn bộ discovery/compatibility/runtime/CNI/chart/render/lab/freshness gates đạt'
) 2>&1 | tee "${FINAL_EVIDENCE}/gate-122-pipeline.txt"
PIPELINE_RC=${PIPESTATUS[0]}
echo "pipeline gate rc=${PIPELINE_RC}"
```

**PASS:** `pipeline gate rc=0`. Bất kỳ file thiếu hoặc không có `PASS:` đều là **STOP**.

### 12.3. Đổi trạng thái APPROVED và đóng gói

Chỉ chạy sau §12.2 PASS:

```bash
source "${BASELINE_ROOT}/research-session.env"
set -a
source "${BASELINE_ROOT}/baseline-metadata.env"
set +a

(
  grep -Fq 'PASS: toàn bộ discovery/compatibility/runtime/CNI/chart/render/lab/freshness gates đạt' \
    "${FINAL_EVIDENCE}/gate-122-pipeline.txt" || {
    echo 'FAIL: pipeline chưa PASS'; exit 1;
  }

  sed -i 's/^BASELINE_STATUS=.*/BASELINE_STATUS=APPROVED/' \
    "${BASELINE_ROOT}/baseline-metadata.env"
  sed -i "s/^APPROVED_AT=.*/APPROVED_AT=${RESEARCH_DATE}/" \
    "${BASELINE_ROOT}/baseline-metadata.env"

  find "${BASELINE_ROOT}" -type f \
    ! -path '*/evidence/12-final/FINAL-SHA256SUMS' \
    -print0 | sort -z | xargs -0 sha256sum \
    > "${FINAL_EVIDENCE}/FINAL-SHA256SUMS"

  grep -Fq 'BASELINE_STATUS=APPROVED' \
    "${BASELINE_ROOT}/baseline-metadata.env" || {
    echo 'FAIL: không đổi được APPROVED'; exit 1;
  }
  test -s "${FINAL_EVIDENCE}/FINAL-SHA256SUMS" || {
    echo 'FAIL: checksum bundle rỗng'; exit 1;
  }

  echo "PASS: baseline APPROVED; review_by=${REVIEW_BY}; checksum bundle hoàn tất"
)
APPROVAL_RC=$?
echo "approval gate rc=${APPROVAL_RC}"
```

Đầu ra dùng để cập nhật runbook triển khai là `baseline-versions.md`. Giữ nguyên toàn bộ workspace/evidence để lần sau tái dựng được quyết định.

---

## 13. Danh mục nguồn official tối thiểu

| Thành phần | Nguồn bắt buộc |
| --- | --- |
| Kubernetes lifecycle | [Releases](https://kubernetes.io/releases/) |
| Kubernetes skew | [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) |
| kubeadm packages | [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) |
| Runtime/CRI/cgroup | [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) |
| CNI requirement | [Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) |
| containerd lifecycle | [containerd RELEASES.md](https://github.com/containerd/containerd/blob/main/RELEASES.md) |
| Flannel | [Repository](https://github.com/flannel-io/flannel), [Releases](https://github.com/flannel-io/flannel/releases) |
| Helm | [Version Support Policy](https://helm.sh/docs/v3/topics/version_skew/) |
| Rancher release channel | [Choosing a Rancher Version](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/choose-a-rancher-version) |
| Rancher compatibility | [SUSE Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/) |
| Rancher requirements | [Installation Requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/installation-requirements) |
| Rancher Helm | [Helm Version Requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements) |
| Rancher chart behavior | Exact release `Chart.yaml`, `values.yaml`, schema, templates and CRDs; ví dụ [v2.14.3 values](https://github.com/rancher/rancher/blob/v2.14.3/chart/values.yaml) |
| cert-manager | [Supported Releases](https://cert-manager.io/docs/releases/), [Helm](https://cert-manager.io/docs/installation/helm/) |
| Traefik | [Helm chart releases](https://github.com/traefik/traefik-helm-chart/releases), [Kubernetes docs](https://doc.traefik.io/traefik/setup/kubernetes/) |
| cloudflared | [Official downloads/releases](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/) |
| local-path-provisioner | [Official releases](https://github.com/rancher/local-path-provisioner/releases) |
| MetalLB | [Official releases](https://github.com/metallb/metallb/releases), [documentation](https://metallb.io/) |

Nếu một URL “current” đổi nội dung theo thời gian, `sources.md` phải lưu ngày truy cập, version đang hiển thị và URL/tag immutable của artifact đã dùng. Đó là điều kiện để người sau tái dựng được quyết định thay vì chỉ thấy một danh sách số phiên bản.
