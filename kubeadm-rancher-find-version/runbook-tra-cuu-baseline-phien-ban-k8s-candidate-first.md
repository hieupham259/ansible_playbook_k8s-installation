# Runbook tra cứu baseline phiên bản Kubernetes/Rancher — Candidate-first

> **Mục tiêu:** tìm và phát hành **một bộ phiên bản hoàn chỉnh** cho lab Kubernetes dựng bằng kubeadm, CNI Flannel, ingress Traefik, TLS cert-manager và Rancher cài trên chính cluster. Runbook tạo danh sách Kubernetes candidate, lọc tuần tự qua mọi thành phần, rồi chọn đúng một candidate tốt nhất.
>
> **Phạm vi:** chỉ tra cứu và xác minh version/artifact. Firewall, DNS, registry, tài nguyên VM và thao tác dựng cluster thuộc runbook cài đặt.
>
> **Kết quả bắt buộc:** một trong hai trạng thái:
>
> - `SELECTED-CONDITIONAL`: còn ít nhất một candidate vượt qua toàn bộ **gate tra cứu/render**, dòng thắng mang state `SELECTED`, và điều kiện CNI `VERSION` được bàn giao bắt buộc trước khi mở cluster;
> - `BLOCKED`: đã xét hết candidate nhưng không còn tổ hợp hợp lệ, hoặc có bằng chứng bắt buộc không thể xác minh.

Không được kết luận `BLOCKED` chỉ vì candidate đầu tiên thất bại. Không được ghi `SELECTED` trước Bước 9.

---

## Cách dùng và state machine

Thực hiện tuần tự từ Bước 0 đến Bước 12. Bước 2 tạo tập candidate; Bước 3–8 chỉ lọc tập đó:

```text
K8s candidates
  → Rancher
  → containerd + gói kubeadm/kubelet/kubectl
  → Helm
  → cert-manager
  → Traefik
  → Flannel/addon
  → chọn candidate tốt nhất
  → hoàn thiện exact version và render
```

Mỗi candidate có đúng một trong các trạng thái:

| Trạng thái | Ý nghĩa | Có chạy gate tiếp theo không? |
| --- | --- | --- |
| `SURVIVE` | Chưa thất bại gate nào | Có |
| `ELIMINATED` | Đã có một ràng buộc version không đạt | Không |
| `BLOCKED-UNVERIFIED` | Không lấy/đọc được bằng chứng bắt buộc | Dừng toàn phiên, không phát hành |
| `SELECTED` | Candidate thắng sau Bước 9 | Chỉ một dòng được phép có trạng thái này |
| `RESERVE` | Candidate đã qua toàn bộ gate nhưng xếp sau `SELECTED` | Chỉ dùng để thay thế nếu final render của `SELECTED` thất bại |

Quy tắc cứng:

1. Chỉ xử lý dòng đang `SURVIVE`.
2. Khi loại, ghi gate đầu tiên gây loại và bằng chứng; không xóa dòng. Mọi ô gate phía sau còn `PENDING` phải đổi thành `SKIPPED(<gate đã loại>)`, vì candidate đó không còn được xử lý.
3. Một component không công bố compatibility matrix không đồng nghĩa `FAIL`; phải dùng hợp đồng kỹ thuật được nêu tại bước tương ứng.
4. `PENDING`, `DEFERRED`, `CHƯA XÁC MINH`, `BLOCKED-UNVERIFIED` hoặc `RESERVE` không được tồn tại khi phát hành. **Ngoại lệ duy nhất** là token `DEFERRED-TO-INSTALL-TEST` của gate CNI `VERSION` ở Bước 8.1: phép kiểm đó cần plugin binary nằm trên node, mà node chưa tồn tại ở giai đoạn tra cứu. Nó phải được bàn giao tường minh bằng dòng `HANDOFF: CNI VERSION test` trong phiếu; không gate nào khác được viện dẫn ngoại lệ này.
5. Trang official và metadata của **đúng artifact version** có quyền quyết định; blog/diễn đàn chỉ dùng để tìm hướng.
6. Mọi khối kiểm được bọc trong `( … )`. `exit` bên trong chỉ kết thúc subshell, không đóng terminal và không mất biến đã export. Khối nào cần giữ biến cho bước sau thì không bọc, và khối đó không chứa `exit`.

---

## Bước 0 — Chuẩn bị phiên và phiếu candidate

### 0.1. Neo thời gian

```bash
unset BASELINE_TODAY BASELINE_SESSION_STARTED_AT BASELINE_TIMEZONE
export BASELINE_TODAY="$(date +%F)"
export BASELINE_SESSION_STARTED_AT="$(date --iso-8601=seconds)"
export BASELINE_TIMEZONE="$(date +%Z%z)"
printf 'Ngày tra: %s\nBắt đầu: %s\nMúi giờ: %s\n' \
  "$BASELINE_TODAY" "$BASELINE_SESSION_STARTED_AT" "$BASELINE_TIMEZONE"
```

Phiên tra cứu được phép kéo dài nhiều ngày. Mỗi ngày làm việc chạy lại khối trên để `BASELINE_TODAY` luôn đúng là ngày hiện tại; dòng `- Ngày bắt đầu:` của phiếu giữ nguyên ngày mở phiếu, không sửa theo. Bảng C ghi ngày tra thật của từng dòng bằng chứng, và Bước 12.2 chỉ nhận ngày nằm trong khoảng `[Ngày bắt đầu, BASELINE_TODAY]` dài tối đa **14 ngày** — quá hạn thì phải tra lại bằng chứng cũ, không được sửa ngày cho vừa. Ngày cài dự kiến là đầu vào riêng, không tự gán bằng `BASELINE_TODAY`.

### 0.2. Công cụ

Cần browser và shell Linux/WSL có các lệnh sau:

```bash
(
  missing=0
  for c in curl jq helm python3 grep sed sort awk tr uniq date cat find mkdir rm tail wc; do
    command -v "$c" >/dev/null || { echo "THIẾU: $c"; missing=$((missing + 1)); }
  done
  test "${BASH_VERSINFO[0]:-0}" -ge 4 \
    || { echo "THIẾU: bash >= 4 (đang có ${BASH_VERSION:-?}); Bước 12 dùng mapfile"; missing=$((missing + 1)); }
  test "$missing" -eq 0 || { echo "FAIL: thiếu $missing công cụ"; exit 1; }
  echo 'PASS: đủ công cụ'
)
```

Helm phải là **3.x** và **tối thiểu 3.8.0**. Hai ràng buộc, hai lý do khác nhau — đừng gộp:

- **major = 3** là ràng buộc của **vendor**, không phải của Helm. [Rancher Helm Version Requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements) chỉ công nhận "any Helm v3 version that is officially compatible with the Kubernetes version range you are using". Helm 4 đã GA và là nhánh hiện hành của upstream, nhưng Rancher chưa công bố nó, nên topology của file này vẫn khóa ở 3.x. Khi Rancher công bố Helm 4, sửa gate này trước; không lách bằng cách bỏ qua nó.
- **>= 3.8.0** là ràng buộc kỹ thuật: từ bản đó OCI mới được bật mặc định (release notes v3.8.0 — *"OCI registry support for charts is now generally available. It has graduated out of being an experiment"*, kèm commit "Removing all the checks for oci experimental flag"), mà Bước 6 đọc chart cert-manager qua `oci://`.

Đây là gate bootstrap, khác với gate ở Bước 11 (Helm phải đúng exact version của candidate `SELECTED`).

```bash
(
  command -v helm >/dev/null || { echo 'FAIL: thiếu helm'; exit 1; }
  v="$(helm version --template '{{.Version}}')"; v="${v#v}"
  major="${v%%.*}"; rest="${v#*.}"; minor="${rest%%.*}"
  case "$major$minor" in
    *[!0-9]*) echo "FAIL: không đọc được version helm: $v"; exit 1 ;;
  esac
  test "$major" -eq 3 || { echo "FAIL: cần Helm 3.x, đang có $v"; exit 1; }
  test "$minor" -ge 8 \
    || { echo "FAIL: cần Helm >= 3.8.0 để dùng OCI ở Bước 6, đang có $v"; exit 1; }
  echo "PASS: helm $v (>= 3.8.0, OCI bật mặc định)"
)
```

Bước 4 phải chạy trên Ubuntu **cùng release và architecture với node mục tiêu**, và trên máy đó kiểm thêm:

```bash
(
  missing=0
  for c in sudo apt-get apt-cache dpkg; do
    command -v "$c" >/dev/null || { echo "THIẾU: $c"; missing=$((missing + 1)); }
  done
  test "$missing" -eq 0 || { echo "FAIL: thiếu $missing công cụ cho Bước 4"; exit 1; }
  command -v pro >/dev/null \
    || echo 'GHI CHÚ: không có pro — phải xác định coverage từ APT metadata, xem Bước 4.3'
  echo 'PASS: đủ công cụ cho Bước 4'
)
```

Runbook không tự cài công cụ còn thiếu. Ba khối trên đều thoát khác 0 khi thiếu — đừng đổi lại thành `|| echo` trần, vì khi đó `echo` trả 0 và cả vòng lặp luôn báo thành công.

### 0.3. Tạo phiếu

```bash
: "${BASELINE_TODAY:?chưa chạy Bước 0.1}"
export BASELINE_DIR="$PWD"
export BASELINE_SHEET="$BASELINE_DIR/phieu-candidate-baseline-${BASELINE_TODAY}.md"
printf '%s\n' "$BASELINE_SHEET"
```

`BASELINE_SHEET` là **đường dẫn tuyệt đối**. Các bước sau có đổi thư mục làm việc, nên tên tương đối sẽ làm mọi gate đọc phiếu trở thành pass giả.

Tạo file trên từ template sau. Ô không áp dụng ghi `N/A`; không xóa candidate bị loại.

```markdown
# Phiếu candidate baseline

## A. Đầu vào
- Ngày bắt đầu: <BASELINE_TODAY>
- Ngày cài dự kiến: ___
- Hướng: TECHNICALLY-COMPATIBLE
- OS: Ubuntu ___ | Architecture: ___
- Topology: kubeadm + Flannel + Traefik Ingress + cert-manager + Rancher trên cùng cluster
- Network/CIDR: OUT-OF-SCOPE — kiểm tra không chồng lấn trong runbook cài đặt
- Addon: ___

## B. Candidate pipeline
| ID | K8s minor | K8s patch | K8s EOL | Rancher chart/app | containerd package/upstream | kube packages | Helm | cert-manager | Traefik chart/app | Flannel/addon | Lifecycle limit | State | Gate/lý do |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| K01 | ___ | ___ | ___ | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | SURVIVE | N/A |
| K02 | ___ | ___ | ___ | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | SURVIVE | N/A |
| K03 | ___ | ___ | ___ | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | PENDING | SURVIVE | N/A |

## C. Bằng chứng
| Bước | Candidate | Kiểm tra | Kết quả | URL/artifact | Ngày tra | Ghi chú |
| --- | --- | --- | --- | --- | --- | --- |

## D. Bộ version phát hành
| Thành phần | Exact version | Lifecycle (loại + giá trị) | Nguồn/artifact | Nhãn | Ghi chú |
| --- | --- | --- | --- | --- | --- |
| Kubernetes | ___ | ___ | ___ | TECHNICALLY-COMPATIBLE | ___ |
| kubeadm/kubelet/kubectl | ___ | ___ | ___ | TECHNICALLY-COMPATIBLE | ___ |
| Ubuntu | ___ | ___ | ___ | TECHNICALLY-COMPATIBLE | ___ |
| containerd | ___ | ___ | ___ | TECHNICALLY-COMPATIBLE | ___ |
| Flannel | ___ | ___ | ___ | TECHNICALLY-COMPATIBLE | CNI VERSION: DEFERRED-TO-INSTALL-TEST |
| Helm | ___ | N/A | ___ | TECHNICALLY-COMPATIBLE | ___ |
| cert-manager | ___ | ___ | ___ | TECHNICALLY-COMPATIBLE | ___ |
| Traefik | ___ | ___ | ___ | TECHNICALLY-COMPATIBLE | ___ |
| Rancher | ___ | ___ | ___ | TECHNICALLY-COMPATIBLE | ___ |
| addon | ___ | ___ | ___ | TECHNICALLY-COMPATIBLE | ___ |

- Candidate được chọn: ___
- Ngày review: ___
- Limiting cause: ___
- Candidate bị loại: ___
- HANDOFF: CNI VERSION test — chạy `CNI_COMMAND=VERSION` trên node ở runbook cài đặt trước khi mở cluster; runbook nhận bàn giao: ___
- Kết luận phiên: ___
```

Nếu Kubernetes đang có bốn minor maintained trong giai đoạn chuyển tiếp, thêm `K04`.

Mỗi kết quả PASS/ELIMINATED phải thêm một dòng vào bảng C. Cột `Kiểm tra` bắt đầu bằng đúng một mã chuẩn sau để semantic gate có thể kiểm được bằng máy:

```text
Gate=K8S-RELEASE
Gate=RANCHER
Gate=UBUNTU
Gate=CONTAINERD
Gate=KUBE-PACKAGES
Gate=HELM
Gate=CERT-MANAGER
Gate=TRAEFIK
Gate=FLANNEL
Gate=ADDON
Gate=FINAL-RENDER
```

`Candidate` là ID `Kxx`; bằng chứng Ubuntu dùng chung có thể ghi `ALL`. Candidate bị loại phải có ít nhất dòng bằng chứng của gate đã loại nó. Candidate `SELECTED` phải có đủ toàn bộ gate chuẩn; `Gate=ADDON` vẫn bắt buộc và ghi kết quả `N/A` khi không dùng addon.

Cột `Kết quả` của bảng C chỉ nhận **đúng một** trong ba chuỗi: `PASS`, `N/A`, `ELIMINATED`. Sắc thái — `Supported-not-tested`, `NO-PUBLISHED-K8S-RANGE`, `MONTH-PROXY`, `DEFERRED-TO-INSTALL-TEST` — ghi ở cột `Ghi chú`. Ghép sắc thái vào cột `Kết quả` (ví dụ `PASS (Supported-not-tested)`) sẽ làm semantic gate ở Bước 12 báo thiếu gate, vì nó so cột này bằng chuỗi chính xác.

Ô chứa nhiều giá trị dùng đúng dấu phân tách ` / ` (khoảng trắng — gạch chéo — khoảng trắng) và đúng thứ tự nêu ở tiêu đề cột: `chart / app` cho Rancher và Traefik, `package / upstream` cho containerd, `flannel / addon` cho cột `Flannel/addon`. Cột `Flannel/addon` **luôn** có hai phần; không dùng addon thì ghi `<flannel tag> / N/A`. Bước 12 đối chiếu bảng D với dòng `SELECTED` bằng chuỗi chính xác, nên `2.14.3 / v2.14.3` và `2.14.3/v2.14.3` là hai giá trị khác nhau. Thông tin **không** nằm trong tiêu đề cột — config generation của containerd, trạng thái Tested của cert-manager — ghi ở cột `Ghi chú`, không nhét vào ô version.

Không ô nào của bảng B và bảng C được để trống, kể cả cột `Ghi chú`: gate ở Bước 9.2 và 12.2 coi ô rỗng là lỗi schema. Ô không áp dụng ghi `N/A`.

Khi một candidate bị loại, hai ô cuối của dòng nhận `State` = `ELIMINATED` và `Gate/lý do` = `Gate=<TÊN> — <lý do>`. Bảng B chỉ có hai cột cho phần này, nên **không** được gõ `ELIMINATED | Gate=<TÊN> | <lý do>` thành một chuỗi: mỗi `|` tạo thêm một ô và gate schema sẽ báo `số-cột=15`. Dùng em dash để ngăn tên gate với lý do.

**Đi tiếp khi:** phần A đầy đủ, mọi candidate row có ID duy nhất, chưa có dòng nào mang trạng thái `SELECTED`.

### 0.4. Capability theo đúng giai đoạn cài của từng chart

```bash
CERT_MANAGER_API_VERSIONS=(
  --api-versions networking.k8s.io/v1
  --api-versions apiextensions.k8s.io/v1
)
TRAEFIK_API_VERSIONS=(
  --api-versions networking.k8s.io/v1
  --api-versions policy/v1/PodDisruptionBudget
  --api-versions apiextensions.k8s.io/v1
)
RANCHER_API_VERSIONS=(
  --api-versions networking.k8s.io/v1
  --api-versions networking.k8s.io/v1/Ingress
  --api-versions cert-manager.io/v1
  --api-versions apiextensions.k8s.io/v1
)
printf 'CERT_MANAGER_API_VERSIONS: '; printf '%s ' "${CERT_MANAGER_API_VERSIONS[@]}"; echo
printf 'TRAEFIK_API_VERSIONS: '; printf '%s ' "${TRAEFIK_API_VERSIONS[@]}"; echo
printf 'RANCHER_API_VERSIONS: '; printf '%s ' "${RANCHER_API_VERSIONS[@]}"; echo
```

`helm template` có một tập API built-in do phiên bản Helm/client-go đang chạy biết, nhưng tập đó không phải kết quả discovery của target cluster và thường không chứa custom API như `cert-manager.io/v1`. Các mảng trên được Helm **cộng thêm** vào tập mặc định, không thay thế nó.

Không dùng một mảng chung cho cả ba chart: lúc render cert-manager, CRD `cert-manager.io/v1` chưa tồn tại; lúc render Rancher, kiến trúc đích yêu cầu cert-manager đã được cài. Bước 11.2 sẽ đối chiếu cả trạng thái API kỳ vọng `PRESENT` lẫn `ABSENT` của từng chart và buộc sửa các mảng nếu giả định ban đầu chưa đúng.

Đây là các mảng shell, không phải biến export: mở shell mới thì chạy lại Bước 0.1 và 0.4 trước khi chạy bất kỳ lệnh `helm` nào.

---

## Bước 1 — Chọn mức cam kết và chốt phạm vi

### 1.1. Không đánh đồng các khái niệm

| Khái niệm | Nghĩa trong runbook |
| --- | --- |
| Vendor-supported | Vendor chứng nhận đúng sản phẩm, version, platform và vai trò |
| Technically-compatible | Dải official/metadata cho phép và toàn bộ gate của runbook đạt |
| Tested | Upstream chạy test thường xuyên với version đó |
| Supported | Upstream nhận và xử lý lỗi cho version đó; có thể rộng hơn Tested |
| Metadata-compatible | Artifact cho phép version, ví dụ `kubeVersion`; chưa phải chứng nhận vendor |

Topology cố định của file này là Rancher Manager trên kubeadm. Nếu yêu cầu `VENDOR-SUPPORTED` và SUSE support matrix không liệt kê kubeadm làm Manager host, ghi `HANDOFF-TO-RKE2/K3S-RUNBOOK` và kết thúc. Không đổi topology rồi tiếp tục các bước kubeadm.

### 1.2. Không dùng ngoại cảnh để lọc version

CIDR, DNS, firewall, registry, tài nguyên VM và khả năng kết nối runtime không tham gia phép giao version của file này. Chúng được kiểm trong runbook cài đặt. Không được loại candidate hoặc kết luận `BLOCKED` ở đây vì một trong các đầu vào ngoại cảnh đó chưa sẵn sàng.

**Đi tiếp khi:** hướng đã chốt là `TECHNICALLY-COMPATIBLE`; topology vẫn là Rancher Manager trên kubeadm; không đưa điều kiện ngoài phạm vi vào bảng candidate.

---

## Bước 2 — Tạo đầy đủ danh sách Kubernetes candidate

Mở [Kubernetes Releases](https://kubernetes.io/releases/). Ghi **mọi minor đang maintained**, exact patch mới nhất và EOL vào bảng B. Không chọn một minor tại đây.

Tạo biến ánh xạ từ chính phiếu, ví dụ theo định dạng `ID=patch`:

```bash
export K8S_CANDIDATES='<K01=1.xx.y,K02=1.xx.y,K03=1.xx.y>'
```

Kiểm mọi patch là release final có thật:

```bash
(
  set -u
  case "$K8S_CANDIDATES" in
    '<'*) echo 'FAIL: chưa điền K8S_CANDIDATES'; exit 1 ;;
  esac

  IFS=',' read -ra pairs <<< "$K8S_CANDIDATES"
  failed=0
  for pair in "${pairs[@]}"; do
    id="${pair%%=*}"
    version="${pair#*=}"
    result="$(curl -fsSL "https://api.github.com/repos/kubernetes/kubernetes/releases/tags/v${version}" \
      | jq -r '[.tag_name, .prerelease, .draft] | @tsv')" || result=''
    if [[ "$result" == $'v'"${version}"$'\tfalse\tfalse' ]]; then
      printf 'PASS\t%s\t%s\n' "$id" "$version"
    else
      printf 'FAIL\t%s\t%s\n' "$id" "$version"
      failed=$((failed + 1))
    fi
  done
  test "$failed" -eq 0 || { echo "FAIL: $failed candidate chưa xác minh được release final"; exit 1; }
  echo 'PASS: mọi candidate patch là release final'
)
```

Nếu source official liệt kê patch nhưng API tạm thời không đọc được, dùng trang changelog/release official thứ hai để xác minh. Không loại candidate vì lỗi mạng; ghi `BLOCKED-UNVERIFIED` nếu không thể lấy bất kỳ bằng chứng official nào.

**Ghi:** minor/patch/EOL vào bảng B; **và** mỗi candidate một dòng bảng C với `Kiểm tra` bắt đầu bằng `Gate=K8S-RELEASE`, `Kết quả` là `PASS`, `URL/artifact` là link release official, `Ngày tra` là ngày chạy khối trên. Cột `Ghi chú` ghi minor, exact patch và ngày EOL đọc được — không được để trống, Bước 12.2 coi ghi chú rỗng là evidence không hợp lệ.

**Đi tiếp khi:** mỗi minor maintained có một ID, exact patch, EOL và bằng chứng final release; tất cả vẫn là `SURVIVE`.

---

## Bước 3 — Lọc từng K8s candidate qua Rancher

### 3.1. Tạo danh sách Rancher chart candidate

Đọc [Choosing a Rancher Version](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/choose-a-rancher-version). Dùng `rancher-stable`, bỏ alpha/beta/RC. Đọc thêm [SUSE Rancher lifecycle](https://www.suse.com/lifecycle/#rancher); chỉ đưa vào danh sách chart final còn lifecycle phù hợp với ngày cài.

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable --force-update
helm search repo rancher-stable/rancher --versions --output json \
  | jq -r '.[]
           | select(.name == "rancher-stable/rancher")
           | select(.version | test("-(rc|alpha|beta)"; "i") | not)
           | [.version, .app_version] | @tsv'
```

`helm search repo` khớp theo **chuỗi con**, nên bất kỳ chart nào khác trong repo có tên chứa `rancher` cũng lọt vào danh sách. `select(.name == ...)` giữ đúng một chart; đừng bỏ nó đi kể cả khi repo hiện chỉ có một chart.

### 3.2. Chọn Rancher mới nhất tương thích cho từng candidate

Với từng dòng `SURVIVE`, duyệt chart Rancher theo version giảm dần. Một chart đạt gate kỹ thuật khi:

1. chart final và còn lifecycle qua ngày cài;
2. `helm show chart` trả đúng chart/app version;
3. `helm template --kube-version <K8s patch>` thành công;
4. `kubeVersion` của chart chứa candidate đó;
5. K8s minor nằm trong **Rancher Manager Kubernetes range** của exact Rancher minor trên SUSE Rancher Support Matrix.

Mẫu kiểm một cặp:

```bash
export CANDIDATE_ID='<Kxx>'
export K8S_VERSION='<1.xx.y>'
export RANCHER_VERSION='<x.y.z>'

helm show chart rancher-stable/rancher --version "$RANCHER_VERSION" \
  | grep -E '^(version|appVersion|kubeVersion):'

helm template rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version "$RANCHER_VERSION" \
  --kube-version "$K8S_VERSION" \
  "${RANCHER_API_VERSIONS[@]}" \
  --set hostname=rancher.example.com \
  >/dev/null
```

Điều kiện 4 (`kubeVersion` chứa candidate) phải đọc **bằng mắt** từ output `helm show chart`, không giao khoán cho lệnh `helm template`. Ràng buộc `kubeVersion` của chart Rancher trong thực tế thường chỉ có cận trên dạng `< 1.xx.0-0`; `--kube-version` vì vậy chỉ bắt được K8s quá mới, không bắt được K8s quá cũ. Điều kiện 5 đóng nửa dải còn thiếu. Việc dùng K8s range của matrix ở đây là một ràng buộc kỹ thuật bảo thủ; nó **không** nâng topology Rancher-on-kubeadm thành vendor-supported.

Gặp chart đầu tiên đạt thì ghi chart/app vào dòng candidate và dừng duyệt Rancher cho dòng đó. Nếu không chart Rancher nào trong danh sách đạt, điền **hai ô cuối** của dòng candidate theo quy ước ở Bước 0.3:

```text
State:       ELIMINATED
Gate/lý do:  Gate=RANCHER — không có Rancher stable còn lifecycle, kubeVersion và Manager K8s range cùng chứa K8s <version>
```

Không dừng pipeline nếu vẫn còn dòng `SURVIVE`.

**Ghi:** ô Rancher của dòng candidate theo dạng `chart / app`; **và** mỗi candidate một dòng bảng C `Gate=RANCHER` — `PASS` kèm dải `kubeVersion` đọc được và K8s range của support matrix, hoặc `ELIMINATED` kèm lý do. Lưu evidence của trang matrix cạnh phiếu và ghi tên file vào cột `Ghi chú`.

#### Lưu evidence trang matrix bằng curl

Trang matrix **không** render bằng JavaScript. URL gốc `.../support-matrix/all-supported-versions/` hiện trả redirect sang một version mặc định; dùng URL đó làm evidence có thể lưu đúng HTML nhưng của **version khác candidate**. Vì vậy luôn dựng URL trang đích từ exact `RANCHER_VERSION`, rồi kiểm tiêu đề exact trong HTML.

```bash
(
  : "${BASELINE_DIR:?chạy Bước 0.3}"
  : "${BASELINE_TODAY:?chạy Bước 0.1}"
  : "${RANCHER_VERSION:?export exact Rancher chart version đang kiểm ở Bước 3.2}"
  [[ "$RANCHER_VERSION" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]] \
    || { echo "FAIL: RANCHER_VERSION không phải exact x.y.z: $RANCHER_VERSION"; exit 1; }
  RANCHER_MATRIX_SLUG="rancher-v${RANCHER_VERSION//./-}"
  out="$BASELINE_DIR/suse-matrix-${RANCHER_MATRIX_SLUG}-${BASELINE_TODAY}.html"
  curl -fsSL \
    "https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/${RANCHER_MATRIX_SLUG}/" \
    -o "$out" || { echo 'FAIL: không tải được trang matrix'; exit 1; }
  expected_heading="Rancher Manager v${RANCHER_VERSION}"
  # Chỉ chuỗi chung "Rancher Manager" là chưa đủ: một trang hợp lệ của version khác
  # vẫn có chuỗi đó và sẽ tạo PASS giả.
  if grep -qi 'http-equiv="refresh"' "$out" || ! grep -Fqi "$expected_heading" "$out"; then
    echo "FAIL: $out không chứa exact heading '$expected_heading'"
    rm -f "$out"
    exit 1
  fi
  wc -c "$out"
  echo "PASS: đã lưu evidence matrix $out"
)
```

Ghi tên file evidence này vào cột `Ghi chú` của dòng `Gate=RANCHER`. Nếu khối trên FAIL dù `RANCHER_VERSION` đúng (SUSE đổi cấu trúc URL/trang), rơi về cách thủ công: mở bằng browser, lưu screenshot cạnh phiếu và ghi tên file screenshot — đừng lưu HTML redirect hoặc matrix của version khác làm bằng chứng.

**Đi tiếp khi:** mọi dòng `SURVIVE` đã có exact Rancher chart/app; mọi dòng không tìm được Rancher đã bị `ELIMINATED` với lý do.

---

## Bước 4 — Lọc qua containerd và gói Kubernetes

Bước này chạy trên Ubuntu cùng release/architecture với node mục tiêu.

### 4.1. Ubuntu release và lifecycle

Mở [Ubuntu release cycle](https://ubuntu.com/about/release-cycle). Với đúng release đã khai ở phần A, đọc ngày kết thúc **standard support**; nếu node sẽ có Ubuntu Pro thì đọc thêm [Ubuntu security maintenance](https://ubuntu.com/security/esm). Mốc áp dụng là mốc của đúng subscription mà node **sẽ thực sự có**, không phải mốc dài nhất Canonical cung cấp cho ai đó.

Ngày này phải sau ngày cài dự kiến và là một mốc bắt buộc trong `Lifecycle limit` ở Bước 9.2.

Ubuntu không phụ thuộc K8s candidate nên chỉ tra một lần. Nhưng nếu không đạt thì **mọi** candidate đều `ELIMINATED | Gate=UBUNTU` và phiên kết luận `BLOCKED` cho tới khi đổi OS release — không được đổi component khác để bù.

**Ghi:** một dòng bảng C với `Candidate` là `ALL` và `Kiểm tra` bắt đầu bằng `Gate=UBUNTU`, `Kết quả` `PASS`, kèm exact release, architecture, loại coverage và mốc `DATE-EXACT` ở cột `Ghi chú`.

### 4.2. Lấy toàn bộ containerd package candidate từ nguồn đã cho phép

Mặc định file này dùng Ubuntu archive. Nếu muốn thêm nguồn khác, phải khai rõ nguồn và lifecycle trước khi đưa package vào tập candidate.

```bash
sudo apt-get update
apt-cache policy containerd
apt-cache madison containerd
```

Đọc [containerd RELEASES.md](https://github.com/containerd/containerd/blob/main/RELEASES.md), tách riêng:

- bảng `Current State of containerd Releases` để lấy lifecycle;
- bảng `Kubernetes Support` để lấy minimum containerd cho từng K8s minor.

Với mỗi K8s `SURVIVE`, chọn Debian package version mới nhất thỏa đồng thời:

1. upstream version nằm trong hàng `Kubernetes Support` của K8s candidate;
2. package có trên đúng OS/architecture;
3. lifecycle của artifact/package còn qua ngày cài;
4. CRI v1 và thế hệ config đã xác định.

Không có package nào đạt thì `ELIMINATED | Gate=CONTAINERD`.

**Ghi:** ô containerd của dòng candidate theo dạng `package / upstream`; config generation và pocket ghi ở bảng C, không nối vào ô version. Mỗi candidate một dòng bảng C `Gate=CONTAINERD` — `PASS` hoặc `ELIMINATED`.

### 4.3. Xác minh kubeadm/kubelet/kubectl của từng candidate

Theo [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/), khai repo `pkgs.k8s.io` riêng cho **minor đang kiểm**, rồi chạy:

```bash
for p in kubeadm kubelet kubectl; do
  apt-cache madison "$p"
done
```

Ba gói phải có version bắt đầu bằng exact K8s patch của candidate và phải ghi nguyên Debian version. Nếu một trong ba gói không tồn tại, `ELIMINATED | Gate=KUBE-PACKAGES`.

`pro security-status` chỉ mô tả package đã cài. Đối với package candidate chưa cài, phải ghi repository component từ APT metadata và áp chính sách coverage của đúng Ubuntu release/subscription. Mốc chỉ có tháng được ghi `MONTH-PROXY` bằng ngày đầu tháng, không giả vờ đó là EOL exact.

**Ghi:** ô kube packages của dòng candidate; **và** mỗi candidate một dòng bảng C `Gate=KUBE-PACKAGES` — `PASS` hoặc `ELIMINATED`, kèm nguyên chuỗi Debian version của cả ba gói ở cột `Ghi chú`.

**Đi tiếp khi:** Ubuntu release đã có mốc lifecycle `DATE-EXACT` sau ngày cài; mỗi dòng `SURVIVE` có exact containerd package/upstream và ba exact Kubernetes package; các dòng không đạt đã bị loại.

---

## Bước 5 — Lọc qua Helm

Đọc đồng thời:

- [Helm v3 Version Support Policy](https://helm.sh/docs/v3/topics/version_skew/) — **giữ nguyên đoạn `/docs/v3/` trong URL**. URL không có `v3` nay trỏ sang tài liệu Helm 4 và bảng ở đó chỉ còn các dòng `4.x`; đọc nhầm bảng đó rồi chọn Helm cho một baseline khóa ở 3.x là sai nguồn ngay từ đầu;
- [Rancher Helm Version Requirements](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/helm-version-requirements);
- release notes của exact Rancher chart/app đang gắn với candidate.

Với mỗi K8s `SURVIVE`:

1. lấy các Helm v3 minor có dải hỗ trợ chứa K8s minor;
2. loại Helm minor thấp hơn yêu cầu của exact Rancher version;
3. chọn minor mới nhất còn lại;
4. chọn patch final mới nhất của minor đó từ [Helm releases](https://github.com/helm/helm/releases).

Tra exact patch theo minor đã chọn:

```bash
export HELM_MINOR='<3.xx>'
```

```bash
(
  set -u
  case "$HELM_MINOR" in
    '<'*) echo 'FAIL: chưa điền HELM_MINOR'; exit 1 ;;
  esac
  latest=''
  # Một trang 100 release chỉ phủ khoảng hai năm gần nhất; minor cũ hơn sẽ rơi ra
  # ngoài và lệnh cũ im lặng trả chuỗi rỗng thay vì FAIL.
  for page in 1 2 3; do
    batch="$(curl -fsSL "https://api.github.com/repos/helm/helm/releases?per_page=100&page=${page}" \
      | jq -r --arg prefix "v${HELM_MINOR}." \
          '.[] | select((.prerelease or .draft) | not) | .tag_name | select(startswith($prefix))')" \
      || { echo "FAIL: không đọc được trang release $page"; exit 1; }
    latest="$(printf '%s\n%s\n' "$latest" "$batch" | grep -v '^[[:space:]]*$' | sort -V | tail -n 1)"
  done
  test -n "$latest" \
    || { echo "FAIL: không có release final nào của Helm $HELM_MINOR trong 300 release gần nhất"; exit 1; }
  echo "PASS: Helm exact = $latest"
)
```

Không có Helm exact version thỏa cả K8s và Rancher thì `ELIMINATED | Gate=HELM`.

Chỉ ghi version sẽ dùng; không yêu cầu thay Helm đang cài giữa pipeline. Trước Bước 11, shell render phải chạy đúng Helm `SELECTED`.

**Ghi:** ô Helm của dòng candidate; **và** mỗi candidate một dòng bảng C `Gate=HELM` — `PASS` hoặc `ELIMINATED`, cột `Ghi chú` nêu cả hai bằng chứng: dải skew Helm↔K8s và yêu cầu tối thiểu của exact Rancher.

**Đi tiếp khi:** mọi candidate còn sống có exact Helm version và hai bằng chứng K8s skew + Rancher requirement.

---

## Bước 6 — Lọc qua cert-manager và giao Rancher ↔ cert-manager

Đọc [cert-manager Supported Releases](https://cert-manager.io/docs/releases/). Với mỗi candidate `SURVIVE`:

1. chỉ xét cert-manager minor đang supported;
2. K8s minor phải nằm trong dải `Supported`;
3. ghi riêng `Tested` hoặc `Supported-not-tested`;
4. lấy patch final mới nhất của minor đó;
5. kiểm exact Rancher version không công bố incompatibility và cert-manager đạt minimum mà exact Rancher chart yêu cầu;
6. ghi lifecycle kiểu `DATE-EXACT`, `MONTH-PROXY` hoặc `EVENT-PROXY`.

Kiểm release và OCI chart:

```bash
export CERT_MANAGER_VERSION='<vX.Y.Z>'
curl -fsSL "https://api.github.com/repos/cert-manager/cert-manager/releases/tags/${CERT_MANAGER_VERSION}" \
  | jq '{tag: .tag_name, prerelease, draft}'

helm show chart 'oci://quay.io/jetstack/charts/cert-manager' \
  --version "$CERT_MANAGER_VERSION" \
  | grep -E '^(version|appVersion|kubeVersion):'
```

Trong hướng `TECHNICALLY-COMPATIBLE`, `Supported-not-tested` vẫn được phép sống nhưng phải ghi đúng nhãn. K8s ngoài dải Supported, chart không tồn tại, hoặc không thỏa yêu cầu Rancher thì `ELIMINATED | Gate=CERT-MANAGER`.

**Ghi:** ô cert-manager của dòng candidate; **và** mỗi candidate một dòng bảng C `Gate=CERT-MANAGER` — `Kết quả` là `PASS` hoặc `ELIMINATED`, còn `Tested` / `Supported-not-tested` và loại lifecycle ghi ở cột `Ghi chú`. Đừng nối sắc thái vào cột `Kết quả`; xem quy ước ở Bước 0.3.

**Đi tiếp khi:** mọi candidate còn sống có exact cert-manager version, trạng thái Supported/Tested, lifecycle và bằng chứng giao với Rancher.

---

## Bước 7 — Lọc qua Traefik

Thêm repo và lấy toàn bộ chart final:

```bash
helm repo add traefik https://traefik.github.io/charts --force-update
helm search repo traefik/traefik --versions --output json \
  | jq -r '.[]
           | select(.name == "traefik/traefik")
           | select(.version | test("-(rc|alpha|beta)"; "i") | not)
           | [.version, .app_version] | @tsv'
```

`select(.name == "traefik/traefik")` là bắt buộc ở đây, không phải phòng xa: repo này còn chứa `traefik/traefik-crds` và các chart khác có tên bắt đầu bằng `traefik/traefik`, và version của chúng nằm ở dải số hoàn toàn khác chart chính. Trộn vào rồi duyệt giảm dần là chọn nhầm chart.

Với từng K8s candidate `SURVIVE`, duyệt chart version giảm dần và chọn chart đầu tiên đạt:

```bash
export K8S_VERSION='<1.xx.y>'
export TRAEFIK_CHART_VERSION='<x.y.z>'

helm show chart traefik/traefik --version "$TRAEFIK_CHART_VERSION" \
  | grep -E '^(version|appVersion|kubeVersion):'

helm template traefik traefik/traefik \
  --namespace traefik \
  --version "$TRAEFIK_CHART_VERSION" \
  --kube-version "$K8S_VERSION" \
  "${TRAEFIK_API_VERSIONS[@]}" \
  --set providers.kubernetesIngress.enabled=true \
  --set providers.kubernetesGateway.enabled=false \
  >/dev/null
```

Như ở Bước 3.2, `kubeVersion` phải được đọc bằng mắt. `kubeVersion` của chart Traefik trong thực tế thường chỉ có cận dưới dạng `>=1.xx.0-0`, nên `--kube-version` chỉ bắt được K8s quá cũ, không bắt được K8s quá mới.

Không được loại toàn bộ Traefik chỉ vì chart mới nhất không đạt; phải thử chart final tiếp theo còn lifecycle/maintenance phù hợp. Không chart nào đạt mới `ELIMINATED | Gate=TRAEFIK`.

**Ghi:** ô Traefik của dòng candidate theo dạng `chart / app`; **và** mỗi candidate một dòng bảng C `Gate=TRAEFIK` — `PASS` hoặc `ELIMINATED`, kèm `kubeVersion` đọc được ở cột `Ghi chú`.

**Đi tiếp khi:** mọi candidate sống có exact chart version, appVersion và `kubeVersion`/render gate đạt.

---

## Bước 8 — Lọc qua Flannel và addon đã khai

### 8.1. Flannel

Flannel không công bố ma trận Kubernetes. Hợp đồng tra cứu của runbook vì vậy là:

1. release tag final có thật;
2. manifest của đúng tag tải được;
3. mọi image pin tag cụ thể, không dùng `latest`, `stable` hoặc `master`;
4. Pod CIDR đã đối chiếu;
5. các Kubernetes API trong manifest vẫn được phục vụ ở K8s candidate;
6. hợp đồng CNI của **plugin binary** — không kiểm được ở giai đoạn tra cứu, xem mục dưới.

```bash
export FLANNEL_VERSION='<vX.Y.Z>'
curl -fsSL -o "kube-flannel-${FLANNEL_VERSION}.yml" \
  "https://github.com/flannel-io/flannel/releases/download/${FLANNEL_VERSION}/kube-flannel.yml"
grep -E 'apiVersion:|^[[:space:]]+image:|"Network"|"cniVersion"' \
  "kube-flannel-${FLANNEL_VERSION}.yml"
```

Không có compatibility matrix được ghi là `NO-PUBLISHED-K8S-RANGE`, không phải `FAIL`. Nếu một API trong manifest đã bị loại khỏi K8s candidate, hoặc tag/manifest/image không thỏa điều kiện 1–4, candidate là `ELIMINATED | Gate=FLANNEL`.

#### Trường `"cniVersion"` không phải ngưỡng để loại candidate

Giá trị `"cniVersion"` trong `cni-conf.json` của manifest là version của **network configuration**, không phải khả năng của plugin binary. Theo [CNI specification](https://www.cni.dev/docs/spec/#version-considerations), runtime, plugin và network configuration khai báo version **độc lập với nhau**; plugin công bố khả năng của nó qua thao tác `VERSION`, không qua conflist. Một manifest khai `0.3.1` vẫn hoàn toàn hợp lệ khi binary hỗ trợ spec mới hơn.

Vì vậy: **không** áp ngưỡng `0.4.0` lên trường này, và **không** loại candidate vì nó. Chỉ ghi nhận giá trị đọc được vào bảng C.

Phép kiểm đúng theo CNI protocol cần binary đã nằm trên node, nên nó thuộc runbook cài đặt:

```bash
printf '{"cniVersion":"1.0.0"}' | CNI_COMMAND=VERSION /opt/cni/bin/flannel | jq
printf '{"cniVersion":"1.0.0"}' | CNI_COMMAND=VERSION /opt/cni/bin/portmap | jq
```

`supportedVersions` phải chứa `0.4.0` hoặc `1.x`. Ở runbook này, ô CNI VERSION của mọi candidate mang đúng token `DEFERRED-TO-INSTALL-TEST`, và phiếu phải có dòng `HANDOFF: CNI VERSION test` ở cuối phần D. Đây là ngoại lệ duy nhất của quy tắc cứng số 4.

(`/opt/cni/bin/portmap` đến từ bundle `containernetworking/plugins`, không do manifest Flannel cài. Nếu runbook cài đặt dùng portmap thì version bundle đó là một artifact phải chốt ở runbook ấy, không phải ở đây.)

Bảng B **không** có cột riêng cho CNI VERSION: ô `Flannel/addon` chỉ giữ `<flannel tag> / <addon>`. Token `DEFERRED-TO-INSTALL-TEST` nằm ở cột `Ghi chú` của dòng bảng C `Gate=FLANNEL` và ở cột `Ghi chú` dòng Flannel của bảng D.

**Ghi:** ô Flannel/addon của dòng candidate theo dạng `flannel tag / addon` — phần thứ hai là `N/A` khi phần A không khai addon nào; **và** mỗi candidate một dòng bảng C `Gate=FLANNEL` — `PASS` hoặc `ELIMINATED`, cột `Ghi chú` chứa giá trị `"cniVersion"` đọc được, `NO-PUBLISHED-K8S-RANGE` và `DEFERRED-TO-INSTALL-TEST`.

### 8.2. Addon

Chỉ xét addon đã khai trong phần A:

| Addon | Gate candidate |
| --- | --- |
| local-path-provisioner | final tag, manifest đúng tag, API còn được phục vụ, image pin cụ thể |
| MetalLB | final tag và requirement Kubernetes official chứa candidate |
| cloudflared | final tag, artifact/image có architecture mục tiêu; không có K8s API dependency thì ghi `K8S=N/A` |

Kiểm final GitHub release:

```bash
export ADDON_REPO='<owner/repository>'
export ADDON_TAG='<final-tag>'
curl -fsSL "https://api.github.com/repos/${ADDON_REPO}/releases/tags/${ADDON_TAG}" \
  | jq '{tag: .tag_name, prerelease, draft}'
```

Addon không dùng phải ghi `N/A` ở **phần thứ hai** của ô `Flannel/addon` trong bảng B và ở dòng `addon` của bảng D; không được để `PENDING`. Addon bắt buộc không tìm được version phù hợp thì loại candidate tại gate addon.

**Ghi:** mỗi candidate một dòng bảng C `Gate=ADDON`. Dòng này **bắt buộc kể cả khi không dùng addon** — khi đó `Kết quả` là `N/A` và cột `Ghi chú` nêu phần A không khai addon nào. Bước 12 coi thiếu dòng này là lỗi.

**Đi tiếp khi:** cột Flannel/addon của mọi candidate sống đã PASS theo đúng hợp đồng trên; ô CNI VERSION mang đúng token `DEFERRED-TO-INSTALL-TEST` và phiếu đã có dòng `HANDOFF: CNI VERSION test`; không còn `DEFERRED` nào khác.

---

## Bước 9 — Chọn đúng một candidate tốt nhất

### 9.1. Điều kiện tồn tại

Đếm trạng thái trong phiếu:

```bash
(
  : "${BASELINE_SHEET:?chưa chạy Bước 0.3}"
  test -f "$BASELINE_SHEET" || { echo "FAIL: không mở được phiếu $BASELINE_SHEET"; exit 1; }

  rows="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## B[.] / { inside=1; next }
    /^## C[.] / { inside=0 }
    inside && /^\|/ && trim($2) ~ /^K[0-9][0-9]$/ { count++ }
    END { print count + 0 }
  ' "$BASELINE_SHEET")"
  test "${rows:-0}" -ge 1 \
    || { echo 'FAIL: không tìm thấy dòng candidate nào trong bảng B'; exit 1; }

  # ID trùng làm hai candidate dùng chung bằng chứng và RENDER_DIR — gate sau sẽ
  # đối chiếu nhầm dòng mà vẫn PASS. Bắt ở đây, trước khi tốn công xếp hạng.
  dup_ids="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## B[.] / { inside=1; next }
    /^## C[.] / { inside=0 }
    inside && /^\|/ && trim($2) ~ /^K[0-9][0-9]$/ { print trim($2) }
  ' "$BASELINE_SHEET" | sort | uniq -d | tr '\n' ' ')"
  test -z "$dup_ids" \
    || { echo "FAIL: candidate ID trùng trong bảng B: $dup_ids"; exit 1; }

  # Bảng B có 14 cột; -F'|' đẩy chỉ số lên 1 vì dòng mở đầu bằng '|'.
  # $6..$12 = Rancher, containerd, kube packages, Helm, cert-manager, Traefik,
  # Flannel/addon — đúng tập gate của Bước 3–8, và là tập duy nhất được quét ở đây.
  # $13 = Lifecycle limit chỉ được tính ở Bước 9.2, tức là SAU gate này, nên quét nó
  # sẽ làm mọi survivor luôn còn PENDING và Bước 9.1 không bao giờ PASS được.
  # $14 = State, $15 = Gate/lý do.
  pending_gates="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## B[.] / { inside=1; next }
    /^## C[.] / { inside=0 }
    inside && /^\|/ && trim($2) ~ /^K[0-9][0-9]$/ {
      for (column = 6; column <= 12; column++) {
        if ($column ~ /PENDING/) count++
      }
    }
    END { print count + 0 }
  ' "$BASELINE_SHEET")"

  survivors="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## B[.] / { inside=1; next }
    /^## C[.] / { inside=0 }
    inside && /^\|/ && trim($14) == "SURVIVE" { count++ }
    END { print count + 0 }
  ' "$BASELINE_SHEET")"
  # Token bàn giao hợp lệ của gate CNI được gỡ trước khi đếm, xem quy tắc cứng số 4.
  blocked="$(sed 's/DEFERRED-TO-INSTALL-TEST//g' "$BASELINE_SHEET" \
    | grep -Ec 'BLOCKED-UNVERIFIED|CHƯA XÁC MINH|DEFERRED' || true)"

  printf 'CANDIDATE_ROWS=%s PENDING_GATES=%s SURVIVE=%s BLOCKED_OR_UNVERIFIED=%s\n' \
    "$rows" "$pending_gates" "${survivors:-0}" "${blocked:-0}"

  test "$pending_gates" -eq 0 \
    || { echo 'FAIL: chưa chạy hết gate cho mọi candidate'; exit 1; }
  test "${blocked:-0}" -eq 0 \
    || { echo 'BLOCKED: còn bằng chứng bắt buộc chưa xác minh'; exit 1; }
  test "${survivors:-0}" -gt 0 \
    || { echo 'BLOCKED: đã loại hết candidate'; exit 1; }
  echo 'PASS: còn candidate hợp lệ để xếp hạng'
)
```

Nếu không còn candidate, kết luận `BLOCKED` phải kèm bảng loại đầy đủ; không tự đưa minor EOL hoặc version ngoài range vào để tạo kết quả.

### 9.2. Quy tắc xếp hạng tất định

Tính `Lifecycle limit` cho từng survivor bằng mốc sớm nhất trong:

- Kubernetes EOL — key `k8s`;
- **Ubuntu standard support hoặc ESM của đúng subscription node sẽ có** (Bước 4.1) — key `ubuntu`;
- Rancher EOL — key `rancher`;
- containerd/package coverage — key `containerd`;
- cert-manager EOL hoặc event proxy — key `cert-manager`;
- coverage của ba gói `kubeadm/kubelet/kubectl` — key `kube-packages`;
- maintenance của chart Traefik — key `traefik`;
- maintenance của Flannel — key `flannel`;
- lifecycle của addon đã khai — key `addon`.

Năm key đầu là **bắt buộc**. Bốn key sau là **bắt buộc khi và chỉ khi** dòng tương ứng của bảng D có mốc lifecycle khác `N/A`; artifact không công bố mốc nào thì ghi `N/A` ở bảng D và không đưa key đó vào phép tính. Helm không có lifecycle riêng nên dòng Helm của bảng D luôn là `N/A` và không có key. Bước 12.1 chỉ nhận đúng chín key này, còn Bước 12.2 kiểm cả hai chiều: có mốc ở bảng D mà không tham gia phép tính là FAIL, và khai key cho một dòng ghi `N/A` cũng là FAIL. Không có ràng buộc hai chiều đó thì EOL của Traefik/Flannel/addon vẫn nằm trong bản phát hành nhưng không bao giờ ảnh hưởng tới `REVIEW_DATE`.

Mốc Ubuntu giống nhau ở mọi candidate nên không phá được thế hòa, nhưng bỏ nó ra khỏi phép `min` là phát hành một baseline có OS hết support trước phần còn lại.

Chuẩn hóa mốc:

- ngày chính xác: `DATE-EXACT=YYYY-MM-DD`;
- official chỉ cho tháng: `MONTH-PROXY=ngày đầu tháng`;
- lifecycle theo sự kiện: ngày official dự kiến nếu có, nếu không dùng `BASELINE_TODAY + 90 ngày` và ghi `EVENT-PROXY`.

Xếp hạng theo thứ tự:

1. `Lifecycle limit` xa nhất;
2. K8s minor cao nhất;
3. K8s patch cao nhất;
4. Rancher version cao nhất;
5. Candidate ID nhỏ nhất để phá hòa cuối.

Đổi đúng dòng thắng từ `SURVIVE` thành `SELECTED`; các survivor còn lại tạm đổi thành `RESERVE` theo đúng thứ tự xếp hạng. Chưa loại reserve cho đến khi candidate thắng qua final render. Sau đó kiểm:

```bash
(
  test -f "$BASELINE_SHEET" || { echo "FAIL: không mở được phiếu $BASELINE_SHEET"; exit 1; }
  invalid_candidates="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## B[.] / { inside=1; next }
    /^## C[.] / { inside=0 }
    inside && /^\|/ {
      id=trim($2)
      if (id == "ID" || id ~ /^---$/) next
      errors=""
      if (NF != 16) errors=errors " số-cột=" NF-2
      if (id !~ /^K[0-9][0-9]$/) errors=errors " ID-sai"
      for (column=2; column<=15; column++) if (trim($column) == "") errors=errors " cột-" column "-rỗng"
      state=trim($14)
      if (state != "SELECTED" && state != "RESERVE" && state != "ELIMINATED") errors=errors " state-sai=" state
      if (errors != "") print "line " NR ":" errors
    }
  ' "$BASELINE_SHEET")"
  if test -n "$invalid_candidates"; then
    printf '%s\n' "$invalid_candidates"
    echo 'FAIL: bảng B có candidate sai schema/trạng thái sau xếp hạng'
    exit 1
  fi

  selected="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## B[.] / { inside=1; next }
    /^## C[.] / { inside=0 }
    inside && /^\|/ && trim($14) == "SELECTED" { count++ }
    END { print count + 0 }
  ' "$BASELINE_SHEET")"
  test "${selected:-0}" -eq 1 \
    || { echo "FAIL: phải có đúng một SELECTED, hiện có ${selected:-0}"; exit 1; }
  echo 'PASS: bảng B đúng schema; đúng một SELECTED; các dòng khác RESERVE/ELIMINATED'
)
```

---

## Bước 10 — Hoàn thiện bộ exact version và lifecycle

Chép từ dòng `SELECTED` sang bảng D, không tra/chọn version mới tại bước này. Bảng bắt buộc có:

| Thành phần | Cột `Exact version` | Bắt buộc thêm ở cột `Ghi chú` |
| --- | --- | --- |
| Kubernetes | exact patch | minor tương ứng |
| kubeadm/kubelet/kubectl | nguyên Debian package version | xác nhận ba gói cùng version |
| Ubuntu | exact release + architecture | loại coverage (standard / ESM) |
| containerd | `Debian package / upstream version` | **config generation** (`version = 2` hay `3`) và pocket |
| Flannel | release tag | image tag flannel và CNI plugin; `CNI VERSION: DEFERRED-TO-INSTALL-TEST` |
| Helm | exact release tag | nhánh 3.x theo yêu cầu Rancher, và ghi rõ 3.x không còn là major hiện hành của upstream Helm |
| cert-manager | exact release/chart tag | `Tested` hay `Supported-not-tested` |
| Traefik | `chart version / appVersion` | provider đang dùng |
| Rancher | `chart version / appVersion` | topology và kết quả render |
| addon | exact version hoặc `N/A` | lý do nếu `N/A` |

Cột `Exact version` của bảng D phải **giống hệt từng ký tự** ô tương ứng ở dòng `SELECTED` của bảng B — Bước 12 so hai giá trị này bằng chuỗi chính xác. Vì vậy mọi thứ không có mặt trong ô bảng B (config generation, trạng thái Tested, pocket) phải nằm ở cột `Ghi chú`, không được nối thêm vào ô version.

Mỗi dòng phải có lifecycle, URL/artifact, ngày tra và nhãn `TECHNICALLY-COMPATIBLE`. `latest`, `stable`, version range và ô trống bị cấm trong bảng phát hành.

Export đúng version từ bảng D để dùng ở Bước 11:

```bash
export SELECTED_ID='<Kxx>'
export K8S_VERSION='<exact patch>'
export RANCHER_VERSION='<exact chart version>'
export HELM_VERSION='<vX.Y.Z — đúng dạng tag GitHub, có tiền tố v>'
export CERT_MANAGER_VERSION='<exact cert-manager tag>'
export TRAEFIK_CHART_VERSION='<exact Traefik chart version>'
export FLANNEL_VERSION='<exact Flannel tag>'
```

`SELECTED_ID` dùng để tách thư mục render theo candidate ở Bước 11, nhờ đó lần chạy lại cho `RESERVE` không đè bằng chứng của candidate trước.

**Đi tiếp khi:** mọi version ở Bước 11 đều đến từ cùng một dòng `SELECTED`, không trộn component từ candidate khác.

---

## Bước 11 — Render đồng bộ bằng candidate đã chọn

Mọi lệnh ở bước này phải chạy bằng exact Helm version trong bảng D.

```bash
(
  : "${HELM_VERSION:?export HELM_VERSION từ Bước 10}"
  actual="$(helm version --template '{{.Version}}')"
  test "$actual" = "$HELM_VERSION" \
    || { echo "FAIL: Helm đang dùng $actual, bảng D chốt $HELM_VERSION"; exit 1; }
  echo "PASS: Helm $actual"
)
```

Không khớp thì dừng và xử lý ngoài runbook này; đừng render bằng Helm khác rồi ghi kết quả cho Helm đã chốt.

### 11.1. Thư mục render và file values

```bash
: "${BASELINE_DIR:?chạy lại Bước 0.3}"
: "${SELECTED_ID:?export SELECTED_ID từ Bước 10}"
: "${CERT_MANAGER_VERSION:?export từ Bước 10}"
export RENDER_DIR="$BASELINE_DIR/render-check/$SELECTED_ID"
mkdir -p "$RENDER_DIR"
printf 'RENDER_DIR=%s\n' "$RENDER_DIR"

cat > "$RENDER_DIR/cert-manager-values.yaml" <<'EOF'
crds:
  enabled: true
EOF

cat > "$RENDER_DIR/traefik-values.yaml" <<'EOF'
providers:
  kubernetesIngress:
    enabled: true
  kubernetesGateway:
    enabled: false
ingressClass:
  enabled: true
EOF

cat > "$RENDER_DIR/rancher-values.yaml" <<EOF
hostname: rancher.example.com
networkExposure:
  type: ingress
ingress:
  enabled: true
  ingressClassName: traefik
  tls:
    source: rancher
    secretName: tls-rancher-ingress
certmanager:
  version: "${CERT_MANAGER_VERSION#v}"
EOF
```

`certmanager.version` là key thật của chart Rancher (`# Certmanager version compatibility`) và đưa exact cert-manager đã chốt vào helper của chart. Tuy nhiên helper của một số thế hệ chart chỉ phát **warning** khi version thấp hơn minimum, không hard-fail và không thay thế giao tương thích đã xác minh ở Bước 6. Bước 12 vì vậy còn đối chiếu version trong bảng D, dòng `SELECTED` và biến thực sự dùng để render.

`networkExposure` chỉ tồn tại từ thế hệ chart Rancher có nhánh Gateway API song song nhánh Ingress; chart cũ hơn không có key này và Helm sẽ nuốt nó không báo gì. Trước khi tin vào việc pin, đối chiếu với `helm show values rancher-stable/rancher --version "$RANCHER_VERSION"`. Nếu chart đã chốt không có key đó thì nhánh Ingress do `ingress.enabled` quyết định một mình — ghi rõ điều này vào bảng C thay vì để lại một dòng values không có tác dụng.

### 11.2. Audit capability và `lookup` của đúng chart version

`helm template` không nói chuyện với cluster. Hai cơ chế trong template vì vậy cho kết quả khác lúc cài thật, và cả hai đều **im lặng** — không warning, không exit khác 0:

- `.Capabilities.APIVersions.Has "<api>"` trả `false` cho mọi API ngoài tập built-in của Helm cộng với `--api-versions` bạn khai;
- `lookup` luôn trả object rỗng, nên nhánh nào phụ thuộc tài nguyên đã có trong cluster sẽ đi hướng "không tìm thấy".

Các mảng ở Bước 0.4 là **giả định về kiến trúc đích**, chưa phải bằng chứng. Bước này đọc chính template của đúng chart version, bắt người làm khai trạng thái kỳ vọng theo từng chart, rồi đối chiếu với trạng thái Helm thực sự dùng khi render.

Lấy source của ba chart đã chốt về thư mục bằng chứng. Đây là cùng artifact mà `helm template` sẽ dùng, chỉ khác là giải nén ra để đọc được:

```bash
(
  set -euo pipefail
  : "${RENDER_DIR:?chạy Bước 11.1}"
  : "${RANCHER_VERSION:?}"; : "${TRAEFIK_CHART_VERSION:?}"; : "${CERT_MANAGER_VERSION:?}"
  src="$RENDER_DIR/charts"
  test "$src" = "$RENDER_DIR/charts" || { echo 'FAIL: đường dẫn chart source ngoài dự kiến'; exit 1; }
  rm -f "$RENDER_DIR/capability-audit.ok"
  rm -rf -- "$src"
  mkdir -p "$src"

  helm pull rancher-stable/rancher --version "$RANCHER_VERSION" --untar --untardir "$src"
  helm pull traefik/traefik --version "$TRAEFIK_CHART_VERSION" --untar --untardir "$src"
  helm pull 'oci://quay.io/jetstack/charts/cert-manager' \
    --version "$CERT_MANAGER_VERSION" --untar --untardir "$src"

  for chart in rancher traefik cert-manager; do
    test -s "$src/$chart/Chart.yaml" \
      || { echo "FAIL: thiếu source/Chart.yaml của $chart"; exit 1; }
  done
  printf 'RANCHER_VERSION=%s\nTRAEFIK_CHART_VERSION=%s\nCERT_MANAGER_VERSION=%s\n' \
    "$RANCHER_VERSION" "$TRAEFIK_CHART_VERSION" "$CERT_MANAGER_VERSION" \
    > "$RENDER_DIR/capability-versions.snapshot"
  find "$src" -maxdepth 1 -mindepth 1 -type d | sort
  echo 'PASS: đã lấy đủ source của ba exact chart'
)
```

Rút ra mọi API mà template thật sự hỏi và mọi chỗ dùng `lookup`. Parser chấp nhận literal dạng `.Has "api"` hoặc `.Has ("api")`; gặp API truyền qua biến, nối chuỗi hoặc literal kiểu khác thì fail-closed thay vì âm thầm bỏ sót:

```bash
(
  set -euo pipefail
  : "${RENDER_DIR:?chạy Bước 11.1}"
  src="$RENDER_DIR/charts"
  for chart in rancher traefik cert-manager; do
    test -s "$src/$chart/Chart.yaml" || { echo "FAIL: thiếu source $chart"; exit 1; }
  done

  python3 - "$src" "$RENDER_DIR" <<'PY'
from pathlib import Path
import re
import sys

src = Path(sys.argv[1]).resolve()
out = Path(sys.argv[2]).resolve()
api_versions_ref = re.compile(r"Capabilities\.APIVersions\b")
has_token = re.compile(r"\bAPIVersions\.Has\b")
has_literal = re.compile(r'APIVersions\.Has\s*(?:\(\s*)?"([^"]+)"', re.DOTALL)
lookup_token = re.compile(r"(?<![A-Za-z0-9_.])lookup\b")

def template_actions(text, rel):
    """Yield (start, action), ignoring }} inside Go-template strings/comments."""
    pos = 0
    while True:
        start = text.find("{{", pos)
        if start < 0:
            return
        i = start + 2
        quote = None
        escaped = False
        comment = False
        while i < len(text):
            if comment:
                if text.startswith("*/", i):
                    comment = False
                    i += 2
                else:
                    i += 1
                continue
            char = text[i]
            if quote is not None:
                if quote == '"' and escaped:
                    escaped = False
                elif quote == '"' and char == "\\":
                    escaped = True
                elif char == quote:
                    quote = None
                i += 1
                continue
            if text.startswith("/*", i):
                comment = True
                i += 2
                continue
            if text.startswith("}}", i):
                yield start, text[start:i + 2]
                pos = i + 2
                break
            if char in ('"', '`'):
                quote = char
            i += 1
        else:
            lineno = text.count("\n", 0, start) + 1
            raise SystemExit(f"FAIL: template action không đóng tại {rel}:{lineno}")

asked = set()
capability_sites = []
dynamic_sites = []
lookup_sites = []

for path in sorted(src.rglob("*")):
    if not path.is_file():
        continue
    rel = path.relative_to(src)
    if "templates" not in rel.parts:
        continue
    try:
        text = path.read_text(encoding="utf-8")
    except UnicodeDecodeError:
        continue
    chart = rel.parts[0]
    # Parse toàn bộ {{ ... }} action thay vì từng dòng. Cách cũ bỏ sót khi
    # `lookup` hoặc argument của APIVersions.Has nằm ở dòng kế tiếp.
    for action_start, action in template_actions(text, rel):
        lineno = text.count("\n", 0, action_start) + 1
        display = " ".join(action.replace("\t", " ").split())
        if has_token.search(action):
            capability_sites.append(f"{chart}\t{rel}:{lineno}\t{display}")
            literals = list(has_literal.finditer(action))
            for match in literals:
                asked.add((chart, match.group(1)))
            if has_token.search(has_literal.sub("", action)):
                dynamic_sites.append(f"{rel}:{lineno}: {display}")
        elif api_versions_ref.search(action):
            # Alias/assignment như `$apis := .Capabilities.APIVersions` có thể
            # dẫn tới `$apis.Has ...` ở action khác; không đoán, bắt đọc tay.
            dynamic_sites.append(f"{rel}:{lineno}: tham chiếu APIVersions gián tiếp: {display}")
        if lookup_token.search(action):
            lookup_sites.append(f"{chart}\t{rel}:{lineno}\t{display}")

(out / "capability-sites.tsv").write_text(
    "\n".join(capability_sites) + ("\n" if capability_sites else ""), encoding="utf-8"
)
(out / "capabilities-asked.tsv").write_text(
    "".join(f"{chart}\t{api}\n" for chart, api in sorted(asked)), encoding="utf-8"
)
(out / "lookup-sites.tsv").write_text(
    "\n".join(lookup_sites) + ("\n" if lookup_sites else ""), encoding="utf-8"
)
if dynamic_sites:
    print("FAIL: có APIVersions.Has không dùng literal; không thể audit tự động:")
    print("\n".join(dynamic_sites))
    raise SystemExit(1)
print(f"API theo chart: {len(asked)}")
print(f"Điểm lookup: {len(lookup_sites)}")
PY

  cat "$RENDER_DIR/capabilities-asked.tsv"
  cat "$RENDER_DIR/lookup-sites.tsv"
)
```

Tạo hai bảng quyết định **một lần** rồi điền cột 3–4. Không xóa quyết định cũ khi chạy lại; gate sẽ báo dòng thiếu/thừa nếu exact chart đã đổi:

```bash
test -f "$RENDER_DIR/capability-expectations.tsv" || \
  awk -F '\t' 'BEGIN { OFS="\t" } { print $1, $2, "<EXPECTED-PRESENT|EXPECTED-ABSENT>", "<lý do theo đúng giai đoạn cài chart>" }' \
    "$RENDER_DIR/capabilities-asked.tsv" > "$RENDER_DIR/capability-expectations.tsv"

test -f "$RENDER_DIR/lookup-expectations.tsv" || \
  awk -F '\t' 'BEGIN { OFS="\t" } { print $1, $2, "<EXPECTED-EMPTY>", "<vì sao fresh install phải không tìm thấy object này>" }' \
    "$RENDER_DIR/lookup-sites.tsv" > "$RENDER_DIR/lookup-expectations.tsv"
```

Mỗi dòng capability dùng `EXPECTED-PRESENT` hoặc `EXPECTED-ABSENT`. Mỗi điểm `lookup` chỉ được dùng `EXPECTED-EMPTY` trong runbook tra cứu trước khi có cluster. Nếu fresh install cần `lookup` trả object, ghi `EXPECTED-PRESENT`: gate sẽ `BLOCKED-UNVERIFIED`, vì local render không thể chứng minh nhánh đó.

Hỏi chính exact Helm xem từng API **thực sự bật hay tắt** với mảng của đúng chart. Không có API được hỏi là trường hợp hợp lệ; file effective khi đó rỗng và vẫn được kiểm bằng phép so tập ở gate sau:

```bash
(
  set -euo pipefail
  : "${RENDER_DIR:?chạy Bước 11.1}"
  for name in CERT_MANAGER_API_VERSIONS TRAEFIK_API_VERSIONS RANCHER_API_VERSIONS; do
    declare -p "$name" >/dev/null 2>&1 || { echo "FAIL: chưa có $name — chạy lại Bước 0.4"; exit 1; }
  done
  declare -p CERT_MANAGER_API_VERSIONS TRAEFIK_API_VERSIONS RANCHER_API_VERSIONS \
    > "$RENDER_DIR/capability-arrays.snapshot"
  asked="$RENDER_DIR/capabilities-asked.tsv"
  test -f "$asked" || { echo 'FAIL: chưa có capabilities-asked.tsv'; exit 1; }

  probe="$RENDER_DIR/probe"
  test "$probe" = "$RENDER_DIR/probe" || { echo 'FAIL: đường dẫn probe ngoài dự kiến'; exit 1; }
  rm -rf -- "$probe"
  mkdir -p "$probe/templates"
  printf 'apiVersion: v2\nname: probe\nversion: 0.0.0\n' > "$probe/Chart.yaml"
  cat > "$probe/templates/probe.yaml" <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: probe
data:
{{- range $i, $a := .Values.apis }}
  "p{{ $i }}": "{{ $a }}={{ if $.Capabilities.APIVersions.Has $a }}ON{{ else }}OFF{{ end }}"
{{- end }}
EOF
  : > "$RENDER_DIR/capabilities-effective.tsv"

  for chart in cert-manager traefik rancher; do
    awk -F '\t' -v chart="$chart" '$1 == chart { print $2 }' "$asked" > "$probe/apis-$chart.txt"
    test -s "$probe/apis-$chart.txt" || continue
    { echo 'apis:'; sed 's/^/  - /' "$probe/apis-$chart.txt"; } > "$probe/values.yaml"
    case "$chart" in
      cert-manager) flags=("${CERT_MANAGER_API_VERSIONS[@]}") ;;
      traefik) flags=("${TRAEFIK_API_VERSIONS[@]}") ;;
      rancher) flags=("${RANCHER_API_VERSIONS[@]}") ;;
      *) echo "FAIL: chart không biết: $chart"; exit 1 ;;
    esac
    helm template probe "$probe" "${flags[@]}" > "$probe/rendered-$chart.yaml"
    grep -oE '[^"]+=(ON|OFF)' "$probe/rendered-$chart.yaml" \
      | while IFS= read -r result; do
          api="${result%=*}"; state="${result##*=}"
          printf '%s\t%s\t%s\n' "$chart" "$api" "$state"
        done >> "$RENDER_DIR/capabilities-effective.tsv"
  done
  sort -u -o "$RENDER_DIR/capabilities-effective.tsv" "$RENDER_DIR/capabilities-effective.tsv"
  cat "$RENDER_DIR/capabilities-effective.tsv"
)
```

Gate cuối đối chiếu tập asked/effective/expected theo khóa `(chart, api)` và lookup theo khóa `(chart, source:line)`. Dòng thiếu, thừa, trùng, placeholder, trạng thái sai hoặc lý do rỗng đều FAIL:

```bash
(
  set -euo pipefail
  : "${RENDER_DIR:?chạy Bước 11.1}"
  rm -f "$RENDER_DIR/capability-audit.ok"
  python3 - "$RENDER_DIR" <<'PY'
from pathlib import Path
import hashlib
import sys

root = Path(sys.argv[1])

def rows(name, columns):
    path = root / name
    if not path.is_file():
        raise SystemExit(f"FAIL: thiếu {path}")
    result = []
    for lineno, raw in enumerate(path.read_text(encoding="utf-8").splitlines(), 1):
        if not raw.strip() or raw.lstrip().startswith("#"):
            continue
        parts = raw.split("\t", columns - 1)
        if len(parts) != columns or any(not part.strip() for part in parts):
            raise SystemExit(f"FAIL: {name}:{lineno} sai định dạng TSV")
        result.append(tuple(part.strip() for part in parts))
    return result

def unique_map(items, key_len, name):
    out = {}
    for item in items:
        key = item[:key_len]
        if key in out:
            raise SystemExit(f"FAIL: {name} lặp khóa {key}")
        out[key] = item[key_len:]
    return out

asked = set(rows("capabilities-asked.tsv", 2))
effective = unique_map(rows("capabilities-effective.tsv", 3), 2, "capabilities-effective.tsv")
expected = unique_map(rows("capability-expectations.tsv", 4), 2, "capability-expectations.tsv")

if set(effective) != asked:
    raise SystemExit(f"FAIL: tập capability effective lệch asked; thiếu={sorted(asked-set(effective))}, thừa={sorted(set(effective)-asked)}")
if set(expected) != asked:
    raise SystemExit(f"FAIL: tập capability expectation lệch asked; thiếu={sorted(asked-set(expected))}, thừa={sorted(set(expected)-asked)}")

for key in sorted(asked):
    (state,) = effective[key]
    wanted, reason = expected[key]
    if wanted not in {"EXPECTED-PRESENT", "EXPECTED-ABSENT"}:
        raise SystemExit(f"FAIL: {key} có expectation không hợp lệ: {wanted}")
    if reason.startswith("<"):
        raise SystemExit(f"FAIL: {key} còn placeholder lý do")
    want_state = "ON" if wanted == "EXPECTED-PRESENT" else "OFF"
    if state != want_state:
        raise SystemExit(f"FAIL: {key} expected {want_state} nhưng Helm render là {state}; sửa mảng Bước 0.4 và quay lại Bước 3/7")

lookup_sites = {(chart, site) for chart, site, _ in rows("lookup-sites.tsv", 3)}
lookup_expected = unique_map(rows("lookup-expectations.tsv", 4), 2, "lookup-expectations.tsv")
if set(lookup_expected) != lookup_sites:
    raise SystemExit(f"FAIL: tập lookup expectation lệch sites; thiếu={sorted(lookup_sites-set(lookup_expected))}, thừa={sorted(set(lookup_expected)-lookup_sites)}")
for key in sorted(lookup_sites):
    wanted, reason = lookup_expected[key]
    if wanted != "EXPECTED-EMPTY":
        raise SystemExit(f"BLOCKED-UNVERIFIED: {key} cần lookup={wanted}; local render chỉ chứng minh được EXPECTED-EMPTY")
    if reason.startswith("<"):
        raise SystemExit(f"FAIL: {key} còn placeholder lý do")

fingerprint_files = [
    root / "capability-sites.tsv",
    root / "capabilities-asked.tsv",
    root / "capabilities-effective.tsv",
    root / "capability-expectations.tsv",
    root / "lookup-sites.tsv",
    root / "lookup-expectations.tsv",
    root / "capability-arrays.snapshot",
    root / "capability-versions.snapshot",
]
fingerprint_files.extend(sorted(path for path in (root / "charts").rglob("*") if path.is_file()))
digest = hashlib.sha256()
for path in fingerprint_files:
    digest.update(str(path.relative_to(root)).encode("utf-8"))
    digest.update(b"\0")
    digest.update(path.read_bytes())
    digest.update(b"\0")

print(f"PASS: capability audit ({len(asked)} chart/API), lookup audit ({len(lookup_sites)} site)")
(root / "capability-audit.ok").write_text(
    f"CAPABILITY_PAIRS={len(asked)}\nLOOKUP_SITES={len(lookup_sites)}\nAUDIT_SHA256={digest.hexdigest()}\n",
    encoding="utf-8",
)
PY
)
```

Nếu gate yêu cầu sửa một mảng API, **không được tiếp tục thẳng sang 11.3**. Chạy lại Bước 0.4, quay lại Bước 3 cho Rancher hoặc Bước 7 cho Traefik đối với **mọi candidate còn sống**, cập nhật lại bảng C/xếp hạng, rồi chạy lại Bước 10–11. Với cert-manager, chạy lại Bước 6 và Bước 10–11. Chỉ tiếp tục khi audit chạy lần hai mà không cần đổi mảng.

**Ghi:** vào dòng `Gate=FINAL-RENDER` số cặp chart/API, toàn bộ expectation `PRESENT`/`ABSENT` và số điểm `lookup`. Lưu kèm sáu artifact `capability-sites.tsv`, `capabilities-asked.tsv`, `capabilities-effective.tsv`, `capability-expectations.tsv`, `lookup-sites.tsv` và `lookup-expectations.tsv`.

### 11.3. Render

```bash
(
  set -euo pipefail
  : "${RENDER_DIR:?chạy Bước 11.1}"
  : "${SELECTED_ID:?}"
  : "${K8S_VERSION:?}"; : "${RANCHER_VERSION:?}"
  : "${HELM_VERSION:?}"; : "${CERT_MANAGER_VERSION:?}"
  : "${TRAEFIK_CHART_VERSION:?}"
  rm -f "$RENDER_DIR/render-content.ok"
  for name in CERT_MANAGER_API_VERSIONS TRAEFIK_API_VERSIONS RANCHER_API_VERSIONS; do
    declare -p "$name" >/dev/null 2>&1 || { echo "FAIL: chưa có $name — chạy lại Bước 0.4"; exit 1; }
  done
  test -s "$RENDER_DIR/capability-audit.ok" \
    || { echo 'FAIL: capability audit 11.2 chưa PASS'; exit 1; }
  current_arrays="$(declare -p CERT_MANAGER_API_VERSIONS TRAEFIK_API_VERSIONS RANCHER_API_VERSIONS)"
  saved_arrays="$(cat "$RENDER_DIR/capability-arrays.snapshot")"
  test "$current_arrays" = "$saved_arrays" \
    || { echo 'FAIL: mảng capability đã đổi sau audit; chạy lại 11.2'; exit 1; }
  current_versions="$(printf 'RANCHER_VERSION=%s\nTRAEFIK_CHART_VERSION=%s\nCERT_MANAGER_VERSION=%s' \
    "$RANCHER_VERSION" "$TRAEFIK_CHART_VERSION" "$CERT_MANAGER_VERSION")"
  saved_versions="$(cat "$RENDER_DIR/capability-versions.snapshot")"
  test "$current_versions" = "$saved_versions" \
    || { echo 'FAIL: exact chart version đã đổi sau audit; chạy lại 11.2'; exit 1; }
  actual_helm="$(helm version --template '{{.Version}}')"
  test "$actual_helm" = "$HELM_VERSION" \
    || { echo "FAIL: Helm thực tế $actual_helm khác HELM_VERSION=$HELM_VERSION"; exit 1; }
  printf 'SELECTED_ID=%s\nK8S_VERSION=%s\nRANCHER_VERSION=%s\nHELM_VERSION=%s\nCERT_MANAGER_VERSION=%s\nTRAEFIK_CHART_VERSION=%s\n' \
    "$SELECTED_ID" "$K8S_VERSION" "$RANCHER_VERSION" "$HELM_VERSION" \
    "$CERT_MANAGER_VERSION" "$TRAEFIK_CHART_VERSION" \
    > "$RENDER_DIR/render-inputs.snapshot"
  python3 - "$RENDER_DIR" <<'PY'
from pathlib import Path
import hashlib
import sys

root = Path(sys.argv[1])
ok = root / "capability-audit.ok"
expected = ""
for line in ok.read_text(encoding="utf-8").splitlines():
    if line.startswith("AUDIT_SHA256="):
        expected = line.split("=", 1)[1]
if not expected:
    raise SystemExit("FAIL: capability-audit.ok thiếu AUDIT_SHA256")

files = [
    root / "capability-sites.tsv",
    root / "capabilities-asked.tsv",
    root / "capabilities-effective.tsv",
    root / "capability-expectations.tsv",
    root / "lookup-sites.tsv",
    root / "lookup-expectations.tsv",
    root / "capability-arrays.snapshot",
    root / "capability-versions.snapshot",
]
files.extend(sorted(path for path in (root / "charts").rglob("*") if path.is_file()))
digest = hashlib.sha256()
for path in files:
    if not path.is_file():
        raise SystemExit(f"FAIL: thiếu artifact sau audit: {path}")
    digest.update(str(path.relative_to(root)).encode("utf-8"))
    digest.update(b"\0")
    digest.update(path.read_bytes())
    digest.update(b"\0")
actual = digest.hexdigest()
if actual != expected:
    raise SystemExit("FAIL: source/quyết định capability đã đổi sau audit; chạy lại 11.2")
print("PASS: fingerprint capability audit không đổi")
PY
  cd "$RENDER_DIR" || exit 1

  failures=0
  if helm template cert-manager "$RENDER_DIR/charts/cert-manager" \
    --namespace cert-manager \
    --kube-version "$K8S_VERSION" "${CERT_MANAGER_API_VERSIONS[@]}" \
    -f cert-manager-values.yaml > cert-manager-rendered.yaml; then
    echo 'PASS: cert-manager render'
  else
    echo 'FAIL: cert-manager render'; failures=$((failures + 1))
  fi

  if helm template traefik "$RENDER_DIR/charts/traefik" \
    --namespace traefik \
    --kube-version "$K8S_VERSION" "${TRAEFIK_API_VERSIONS[@]}" \
    -f traefik-values.yaml > traefik-rendered.yaml; then
    echo 'PASS: Traefik render'
  else
    echo 'FAIL: Traefik render'; failures=$((failures + 1))
  fi

  if helm template rancher "$RENDER_DIR/charts/rancher" \
    --namespace cattle-system \
    --kube-version "$K8S_VERSION" "${RANCHER_API_VERSIONS[@]}" \
    -f rancher-values.yaml > rancher-rendered.yaml; then
    echo 'PASS: Rancher render'
  else
    echo 'FAIL: Rancher render'; failures=$((failures + 1))
  fi

  test "$failures" -eq 0 || { echo "FAIL: $failures chart render lỗi"; exit 1; }
  echo 'PASS: cả ba chart render'
)
```

### 11.4. Kiểm nội dung render

Khối này gom `failures` giống 11.3. Đừng viết nó thành các lệnh rời rồi `echo PASS` ở cuối: `test` và `grep -q` im lặng khi fail, còn `grep` trên file thiếu hoặc rỗng trả về "không tìm thấy" — cộng lại sẽ in `PASS` trên một thư mục render hỏng hoàn toàn.

```bash
(
  set -euo pipefail
  : "${RENDER_DIR:?chạy Bước 11.1}"
  cd "$RENDER_DIR" || exit 1

  failures=0
  fail() { printf 'FAIL\t%s\n' "$1"; failures=$((failures + 1)); }
  pass() { printf 'PASS\t%s\n' "$1"; }

  rendered='cert-manager-rendered.yaml traefik-rendered.yaml rancher-rendered.yaml'
  for f in $rendered; do
    if test -s "$f"; then pass "file có nội dung: $f"; else fail "file thiếu hoặc rỗng: $f"; fi
  done

  n_ingress="$(grep -c '^kind: Ingress$' rancher-rendered.yaml || true)"
  if test "${n_ingress:-0}" -ge 1; then
    pass "Rancher đi nhánh Ingress (${n_ingress})"
  else
    fail 'Rancher không sinh Ingress nào'
  fi

  if grep -Eq '^[[:space:]]*ingressClassName:[[:space:]]*traefik[[:space:]]*$' rancher-rendered.yaml; then
    pass 'ingressClassName: traefik'
  else
    fail 'không thấy ingressClassName: traefik'
  fi

  if grep -Eq '^kind: (Gateway|GatewayClass|HTTPRoute)$' $rendered; then
    fail 'manifest lạc sang nhánh Gateway API'
  else
    pass 'không có Gateway API'
  fi

  if grep -Eqi "image:.*:(latest|stable|master)([\"']?[[:space:]]*)\$" $rendered; then
    fail 'còn image tag động'
  else
    pass 'không có image tag động'
  fi

  n_crd="$(grep -c '^kind: CustomResourceDefinition$' cert-manager-rendered.yaml || true)"
  if test "${n_crd:-0}" -ge 6; then
    pass "cert-manager CRDs (${n_crd})"
  else
    fail "cert-manager render thiếu CRD (${n_crd:-0}, cần ≥ 6)"
  fi

  test "$failures" -eq 0 || { echo "FAIL: $failures content gate lỗi"; exit 1; }
  python3 - "$RENDER_DIR" <<'PY'
from pathlib import Path
import hashlib
import sys

root = Path(sys.argv[1])
names = [
    "cert-manager-rendered.yaml",
    "traefik-rendered.yaml",
    "rancher-rendered.yaml",
    "cert-manager-values.yaml",
    "traefik-values.yaml",
    "rancher-values.yaml",
    "capability-audit.ok",
    "render-inputs.snapshot",
]
digest = hashlib.sha256()
for name in names:
    path = root / name
    if not path.is_file() or path.stat().st_size == 0:
        raise SystemExit(f"FAIL: thiếu/rỗng artifact trước khi khóa render: {path}")
    digest.update(name.encode("utf-8"))
    digest.update(b"\0")
    digest.update(path.read_bytes())
    digest.update(b"\0")
(root / "render-content.ok").write_text(
    f"RENDER_SHA256={digest.hexdigest()}\n", encoding="utf-8"
)
PY
  echo 'PASS: rendered content gates'
)
```

Hai gate "không có" (Gateway API, image tag động) dựa vào việc `grep` không tìm thấy, nên tự chúng không phân biệt được "sạch" với "file không tồn tại". Vòng `test -s` ở đầu khối là thứ bịt lỗ đó — đừng bỏ nó khi rút gọn.

Đối với Flannel và addon dạng manifest, dùng đúng file/tag đã kiểm ở Bước 8; không thay bằng URL `master` trong runbook cài đặt.

Nếu render fail, đây là lỗi của tổ hợp đã chọn. Đổi dòng `SELECTED` thành `ELIMINATED | Gate=FINAL-RENDER`, nâng dòng `RESERVE` kế tiếp thành `SELECTED` và chạy lại Bước 10–11. Bước 10 export `SELECTED_ID` mới nên 11.1 tạo `RENDER_DIR` riêng cho candidate đó; bằng chứng của candidate vừa loại được giữ nguyên để đối chiếu. Chỉ `BLOCKED` khi đã thử hết reserve theo thứ tự xếp hạng. Khi một candidate render đạt, đổi mọi `RESERVE` còn lại ở bảng B sang `State` = `ELIMINATED` và `Gate/lý do` = `Gate=RANKING — thua candidate <ID>`, đúng quy ước hai ô ở Bước 0.3. `RANKING` không phải gate kỹ thuật và không được thêm thành dòng ở bảng C.

**Ghi:** thêm dòng `Gate=FINAL-RENDER` cho candidate vừa render vào bảng C, kèm `RENDER_DIR` và kết quả 11.2–11.4. Chỉ candidate có dòng bằng chứng này mới được phát hành.

---

## Bước 12 — Semantic gate và phát hành

Bước 12 phải chạy trong **cùng shell** với Bước 10–11. Semantic gate đối chiếu các biến đã export (`SELECTED_ID`, `K8S_VERSION`, `RANCHER_VERSION`, `HELM_VERSION`, `CERT_MANAGER_VERSION`, `TRAEFIK_CHART_VERSION`, `FLANNEL_VERSION`) với bảng D và dòng `SELECTED` — đó là thứ chứng minh bạn đã render đúng bộ version sắp phát hành, chứ không phải một bộ khác. Trong shell mới các biến đó rỗng và mọi phép so ấy sẽ FAIL.

Nếu buộc phải mở shell mới, chạy lại Bước 0.1, 0.3, 0.4 và khối export ở Bước 10 — lấy giá trị từ chính bảng D — rồi mới chạy tiếp. Đừng gõ tay giá trị khác với phiếu để gate qua.

### 12.1. Ngày review

Ngày review là mốc sớm nhất trong:

1. lifecycle limit của candidate `SELECTED` trừ 90 ngày;
2. mọi `EVENT-PROXY`/`MONTH-PROXY` đã ghi;
3. ngày cài dự kiến trừ 30 ngày;
4. `BASELINE_TODAY + 90 ngày` để tránh giữ một snapshot version quá lâu dù EOL còn xa.

Mọi hard EOL phải sau ngày cài dự kiến. Nếu review đã trước ngày hiện tại, candidate không được phát hành.

Tính bằng script để không tự ép một lifecycle thành ngày giả, và để biết **thành phần nào** tạo mốc sớm nhất:

```bash
: "${BASELINE_TODAY:?chạy lại Bước 0.1 trong phiên hiện tại}"
: "${BASELINE_DIR:?chạy lại Bước 0.3}"
# Kết quả được ghi ra file để Bước 12.2 đối chiếu với giá trị bạn chép vào phiếu.
export REVIEW_COMPUTED="$BASELINE_DIR/review-computed.env"
rm -f "$REVIEW_COMPUTED"

export PLANNED_INSTALL_DATE='<YYYY-MM-DD>'
# Chỉ component có mốc DATE-EXACT của candidate SELECTED.
export DATE_EXACT_EOLS='<k8s=YYYY-MM-DD,ubuntu=YYYY-MM-DD,rancher=YYYY-MM-DD,containerd=YYYY-MM-DD>'
# Component có MONTH-PROXY/EVENT-PROXY đã chuẩn hóa ở Bước 9.2.
# Mỗi component chỉ xuất hiện ở đúng một trong hai biến. Năm key bắt buộc là k8s,
# ubuntu, rancher, containerd, cert-manager; bốn key tùy chọn là kube-packages,
# traefik, flannel, addon — bắt buộc khai khi dòng bảng D tương ứng có mốc khác N/A.
# Helm không có key vì lifecycle của nó luôn là N/A.
export PROXY_LIMITS='<cert-manager=YYYY-MM-DD>'

(
python3 - <<'PY'
import os
from datetime import date, timedelta

today = date.fromisoformat(os.environ["BASELINE_TODAY"])
if today != date.today():
    raise SystemExit("FAIL: BASELINE_TODAY không còn là ngày hiện tại; chạy lại Bước 0.1")

def one_date(name):
    raw = os.environ.get(name, "").strip()
    if not raw or raw.startswith("<"):
        raise SystemExit(f"FAIL: chưa điền {name}")
    return date.fromisoformat(raw)

def pairs(name, required):
    raw = os.environ.get(name, "").strip()
    if not raw or raw.startswith("<"):
        if required:
            raise SystemExit(f"FAIL: chưa điền {name}")
        return {}
    out = {}
    for item in raw.split(","):
        item = item.strip()
        if "=" not in item:
            raise SystemExit(f"FAIL: {name} sai định dạng tại '{item}'")
        key, value = (part.strip() for part in item.split("=", 1))
        if not key:
            raise SystemExit(f"FAIL: {name} có key rỗng tại '{item}'")
        if key in out:
            raise SystemExit(f"FAIL: {name} lặp key '{key}'")
        if value.startswith("<") or value.endswith(">"):
            raise SystemExit(f"FAIL: {name} còn placeholder tại '{item}'")
        out[key] = date.fromisoformat(value)
    return out

install = one_date("PLANNED_INSTALL_DATE")
hard = pairs("DATE_EXACT_EOLS", required=True)
proxy = pairs("PROXY_LIMITS", required=False)

duplicated = set(hard) & set(proxy)
if duplicated:
    raise SystemExit("FAIL: component xuất hiện đồng thời ở DATE_EXACT_EOLS và PROXY_LIMITS: " +
                     ", ".join(sorted(duplicated)))

required_limits = {"k8s", "ubuntu", "rancher", "containerd", "cert-manager"}
# Tùy chọn ở đây chỉ có nghĩa "không phải lúc nào artifact cũng công bố mốc".
# Bước 12.2 mới là chỗ ép: dòng bảng D nào có mốc khác N/A thì key phải có mặt.
optional_limits = {"kube-packages", "traefik", "flannel", "addon"}
missing = required_limits - (set(hard) | set(proxy))
if missing:
    raise SystemExit("FAIL: thiếu lifecycle limit bắt buộc: " + ", ".join(sorted(missing)))
unknown = (set(hard) | set(proxy)) - (required_limits | optional_limits)
if unknown:
    raise SystemExit("FAIL: lifecycle limit có component không được phép: " + ", ".join(sorted(unknown)))

expired = {k: v for k, v in hard.items() if v <= install}
if expired:
    detail = ", ".join(f"{k}={v}" for k, v in sorted(expired.items()))
    raise SystemExit(f"FAIL: EOL không sau ngày cài dự kiến: {detail}")

marks = {f"{k} (EOL-90)": v - timedelta(days=90) for k, v in hard.items()}
marks.update({f"{k} (PROXY)": v for k, v in proxy.items()})

install_minus_30 = install - timedelta(days=30)
marks["ngày cài -30"] = install if install_minus_30 < today else install_minus_30
marks["trần tuổi baseline"] = today + timedelta(days=90)

cause, review = min(marks.items(), key=lambda kv: (kv[1], kv[0]))
for name, value in sorted(marks.items(), key=lambda kv: (kv[1], kv[0])):
    print(f"  {value}  {name}")
print(f"BASELINE_TODAY={today}")
print(f"PLANNED_INSTALL_DATE={install}")
print(f"REVIEW_DATE={review}")
print(f"LIMITING_CAUSE={cause}")
if review < today:
    raise SystemExit("FAIL: baseline stale ngay khi lập")

# Chỉ ghi artifact khi baseline không stale, và ghi sau mọi phép kiểm — file này
# là thứ Bước 12.2 dùng để chứng minh giá trị trong phiếu đúng là giá trị vừa tính.
computed = os.environ["REVIEW_COMPUTED"]
with open(computed, "w", encoding="utf-8", newline="\n") as fh:
    fh.write(f"BASELINE_TODAY={today}\n")
    fh.write(f"PLANNED_INSTALL_DATE={install}\n")
    fh.write(f"DATE_EXACT_EOLS={os.environ['DATE_EXACT_EOLS'].strip()}\n")
    fh.write(f"PROXY_LIMITS={os.environ.get('PROXY_LIMITS', '').strip()}\n")
    fh.write(f"REVIEW_DATE={review}\n")
    fh.write(f"LIMITING_CAUSE={cause}\n")
print(f"WROTE={computed}")
print("OK: baseline chưa stale")
PY
)
```

Phá hòa bằng tên mốc để hai lần chạy trên cùng dữ liệu luôn ra cùng `LIMITING_CAUSE`, đúng tinh thần xếp hạng tất định ở Bước 9.2.

Baseline stale thì sửa **đúng thành phần trong `LIMITING_CAUSE`**: đổi version của nó, đổi nguồn cung cấp có lifecycle phù hợp, hoặc đổi ngày cài. Nếu nguyên nhân là chính Kubernetes hoặc giao của các dải tương thích, quay lại Bước 9.2 và lấy `RESERVE` kế tiếp. Không đổi component không liên quan cho tới khi phép tính tình cờ đạt.

Với cert-manager, cả hai minor đang supported thường mang EOL kiểu `EVENT` (`Release of 1.x`) không có lịch chắc chắn, nên `PROXY_LIMITS` của nó rơi về `BASELINE_TODAY + 90` và rất hay trở thành `LIMITING_CAUSE`. Đó là kết quả đúng, không phải lỗi nhập liệu.

**Ghi:** chép nguyên văn `REVIEW_DATE` vào dòng `- Ngày review:` và `LIMITING_CAUSE` vào dòng `- Limiting cause:` ở cuối phần D. Bảng D phải ghi các lifecycle đã truyền vào phép tính theo đúng dạng `DATE-EXACT=YYYY-MM-DD`, `MONTH-PROXY=YYYY-MM-DD` hoặc `EVENT-PROXY=YYYY-MM-DD`; ngày bắt đầu và ngày cài dự kiến cũng phải đúng với input của phép tính. Bước 12.2 đối chiếu tất cả các giá trị này với `review-computed.env`, nên gõ lại theo trí nhớ hoặc làm tròn ngày sẽ FAIL — đó là chủ đích.

### 12.2. Gate phiếu

```bash
(
  : "${BASELINE_SHEET:?chưa chạy Bước 0.3}"
  : "${BASELINE_DIR:?chưa chạy Bước 0.3}"
  : "${BASELINE_TODAY:?chưa chạy Bước 0.1}"
  test -f "$BASELINE_SHEET" \
    || { echo "FAIL: không mở được phiếu $BASELINE_SHEET"; exit 1; }

  failed=0
  fail() { printf 'FAIL\t%s\n' "$1"; failed=$((failed + 1)); }
  pass() { printf 'PASS\t%s\n' "$1"; }
  sheet_value() {
    sed -n "s/^- $1:[[:space:]]*//p" "$BASELINE_SHEET" | tr -d '\r' | sed 's/[[:space:]]*$//'
  }

  if grep -n '___' "$BASELINE_SHEET"; then
    fail 'còn placeholder'
  fi

  if grep -n '<BASELINE_TODAY>' "$BASELINE_SHEET"; then
    fail 'chưa thay placeholder ngày'
  fi

  if test "$(sheet_value 'Hướng')" = 'TECHNICALLY-COMPATIBLE'; then
    pass 'hướng phát hành đúng TECHNICALLY-COMPATIBLE'
  else
    fail "Hướng không hợp lệ: '$(sheet_value 'Hướng')'"
  fi
  if grep -Eq '^- OS: Ubuntu [^|[:space:]][^|]* \| Architecture: [^[:space:]].*$' "$BASELINE_SHEET"; then
    pass 'OS release và architecture đã khai'
  else
    fail 'dòng OS phải có Ubuntu release và architecture'
  fi
  if grep -Fxq -- '- Topology: kubeadm + Flannel + Traefik Ingress + cert-manager + Rancher trên cùng cluster' "$BASELINE_SHEET"; then
    pass 'topology khớp phạm vi runbook'
  else
    fail 'topology đã đổi khỏi phạm vi runbook'
  fi
  if grep -Fxq -- '- Network/CIDR: OUT-OF-SCOPE — kiểm tra không chồng lấn trong runbook cài đặt' "$BASELINE_SHEET"; then
    pass 'CIDR được bàn giao, không tham gia gate version'
  else
    fail 'dòng bàn giao Network/CIDR đã bị đổi hoặc thiếu'
  fi
  if test -n "$(sheet_value 'Addon')"; then pass 'Addon đã khai'; else fail 'Addon rỗng'; fi
  if test -n "$(sheet_value 'Candidate bị loại')"; then pass 'Danh sách candidate bị loại đã khai'; else fail 'Candidate bị loại rỗng'; fi
  if test "$(sheet_value 'Kết luận phiên')" = 'SELECTED-CONDITIONAL'; then
    pass 'Kết luận phiên là SELECTED-CONDITIONAL'
  else
    fail "Kết luận phiên phải là SELECTED-CONDITIONAL, hiện là '$(sheet_value 'Kết luận phiên')'"
  fi
  handoff_target="$(sed -n 's/^- HANDOFF: CNI VERSION test .*runbook nhận bàn giao:[[:space:]]*//p' "$BASELINE_SHEET" | tr -d '\r' | sed 's/[[:space:]]*$//')"
  if test -n "$handoff_target"; then pass 'đã chỉ rõ runbook nhận CNI VERSION handoff'; else fail 'runbook nhận CNI VERSION handoff rỗng'; fi

  # Gỡ token bàn giao hợp lệ trước khi quét trạng thái dở dang (quy tắc cứng số 4).
  if sed 's/DEFERRED-TO-INSTALL-TEST//g' "$BASELINE_SHEET" \
     | grep -nE 'PENDING|SURVIVE|RESERVE|DEFERRED|CHƯA XÁC MINH|BLOCKED-UNVERIFIED'; then
    fail 'còn trạng thái chưa hoàn tất'
  fi

  if ! grep -q 'HANDOFF: CNI VERSION test' "$BASELINE_SHEET"; then
    fail 'thiếu dòng bàn giao CNI VERSION test sang runbook cài đặt'
  fi

  # Cửa sổ phiên: phiếu được mở ở 'Ngày bắt đầu' và phát hành trong ngày hiện tại.
  # Bằng chứng chỉ hợp lệ khi ngày tra nằm trong cửa sổ đó, và cửa sổ không quá 14
  # ngày — dài hơn thì trang official đã kịp đổi và phải tra lại, không sửa ngày.
  got_start="$(sheet_value 'Ngày bắt đầu')"
  start_epoch="$(date -d "$got_start" +%s 2>/dev/null || true)"
  today_epoch="$(date -d "$BASELINE_TODAY" +%s 2>/dev/null || true)"
  if test -n "$start_epoch" && test -n "$today_epoch" \
     && test "$start_epoch" -le "$today_epoch" \
     && test "$(( (today_epoch - start_epoch) / 86400 ))" -le 14; then
    pass "cửa sổ phiên hợp lệ: $got_start → $BASELINE_TODAY"
  else
    fail "cửa sổ phiên không hợp lệ: Ngày bắt đầu='$got_start', hôm nay='$BASELINE_TODAY' (phải là ngày hợp lệ, cách nhau tối đa 14 ngày)"
  fi

  # Kiểm schema cho TOÀN BỘ bảng C, không chỉ ba cột đủ để đếm gate. Nếu bỏ
  # đoạn này, một dòng PASS nhưng URL/ngày/ghi chú rỗng vẫn được tính là evidence.
  invalid_evidence="$(awk -F'|' -v today="$BASELINE_TODAY" -v start="$got_start" '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    function gate_token(s, a) { split(s, a, /[[:space:]]+/); return a[1] }
    BEGIN {
      canonical["Gate=K8S-RELEASE"]=1; canonical["Gate=RANCHER"]=1
      canonical["Gate=UBUNTU"]=1; canonical["Gate=CONTAINERD"]=1
      canonical["Gate=KUBE-PACKAGES"]=1; canonical["Gate=HELM"]=1
      canonical["Gate=CERT-MANAGER"]=1; canonical["Gate=TRAEFIK"]=1
      canonical["Gate=FLANNEL"]=1; canonical["Gate=ADDON"]=1
      canonical["Gate=FINAL-RENDER"]=1
    }
    /^## C[.] / { inside=1; next }
    /^## D[.] / { inside=0 }
    inside && /^\|/ {
      step=trim($2); candidate=trim($3); check=trim($4); result=trim($5)
      artifact=trim($6); checked=trim($7); note=trim($8)
      if (step == "Bước" || step ~ /^---$/) next
      gate=gate_token(check)
      errors=""
      if (step == "") errors=errors " step-rỗng"
      if (candidate !~ /^(K[0-9][0-9]|ALL)$/) errors=errors " candidate-sai"
      if (!(gate in canonical)) errors=errors " gate-không-chuẩn"
      if (result !~ /^(PASS|N\/A|ELIMINATED)$/) errors=errors " result-sai"
      if (candidate == "ALL" && gate != "Gate=UBUNTU") errors=errors " ALL-chỉ-dành-cho-UBUNTU"
      if (result == "N/A" && gate != "Gate=ADDON") errors=errors " N/A-chỉ-dành-cho-ADDON"
      if (artifact == "") errors=errors " artifact-rỗng"
      if (start !~ /^[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]$/)
        errors=errors " ngày-bắt-đầu-của-phiếu-sai-định-dạng"
      else if (checked !~ /^[0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]$/)
        errors=errors " ngày-tra-sai-định-dạng"
      else if (checked < start || checked > today)
        errors=errors " ngày-tra-ngoài-cửa-sổ[" start ".." today "]"
      if (note == "") errors=errors " ghi-chú-rỗng"
      if (errors != "") print "line " NR ":" errors
    }
  ' "$BASELINE_SHEET")"
  if test -z "$invalid_evidence"; then
    pass 'mọi dòng bảng C đủ schema, artifact, ngày tra và ghi chú'
  else
    printf '%s\n' "$invalid_evidence"
    fail 'bảng C có evidence không hợp lệ'
  fi

  # Ngày cài, ngày review và limiting cause phải đúng là dữ liệu Bước 12.1 vừa dùng/tính,
  # không phải các giá trị được gõ độc lập vào phiếu. Bước 12.1 phải chạy trong chính
  # ngày phát hành, nên BASELINE_TODAY của nó phải bằng BASELINE_TODAY của shell này.
  computed="${BASELINE_DIR:-.}/review-computed.env"
  if test -f "$computed"; then
    want_today="$(sed -n 's/^BASELINE_TODAY=//p' "$computed" | tr -d '\r')"
    want_install="$(sed -n 's/^PLANNED_INSTALL_DATE=//p' "$computed" | tr -d '\r')"
    want_hard="$(sed -n 's/^DATE_EXACT_EOLS=//p' "$computed" | tr -d '\r')"
    want_proxy="$(sed -n 's/^PROXY_LIMITS=//p' "$computed" | tr -d '\r')"
    want_review="$(sed -n 's/^REVIEW_DATE=//p' "$computed" | tr -d '\r')"
    want_cause="$(sed -n 's/^LIMITING_CAUSE=//p' "$computed" | tr -d '\r')"
    test -n "$want_hard" || fail 'review-computed.env thiếu DATE_EXACT_EOLS'
    got_install="$(sheet_value 'Ngày cài dự kiến')"
    got_review="$(sheet_value 'Ngày review')"
    got_cause="$(sheet_value 'Limiting cause')"
    if test -n "$want_today" && test "$want_today" = "$BASELINE_TODAY"; then
      pass "Bước 12.1 đã chạy trong chính ngày phát hành ($want_today)"
    else
      fail "ngày phát hành: Bước 12.1 tính với '$want_today', shell hiện tại là '$BASELINE_TODAY'"
    fi
    if test -n "$want_install" && test "$got_install" = "$want_install"; then
      pass "Ngày cài dự kiến khớp Bước 12.1 ($want_install)"
    else
      fail "Ngày cài dự kiến: phiếu='$got_install' vs tính toán='$want_install'"
    fi
    if test -n "$want_review" && test "$got_review" = "$want_review"; then
      pass "Ngày review khớp Bước 12.1 ($want_review)"
    else
      fail "Ngày review: phiếu='$got_review' vs tính toán='$want_review'"
    fi
    if test -n "$want_cause" && test "$got_cause" = "$want_cause"; then
      pass "Limiting cause khớp Bước 12.1 ($want_cause)"
    else
      fail "Limiting cause: phiếu='$got_cause' vs tính toán='$want_cause'"
    fi
  else
    fail "chưa chạy Bước 12.1 — không thấy $computed"
    want_hard=''
    want_proxy=''
  fi

  invalid_candidates="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## B[.] / { inside=1; next }
    /^## C[.] / { inside=0 }
    inside && /^\|/ {
      id=trim($2)
      if (id == "ID" || id ~ /^---$/) next
      errors=""
      if (NF != 16) errors=errors " số-cột=" NF-2
      if (id !~ /^K[0-9][0-9]$/) errors=errors " ID-sai"
      for (column=2; column<=15; column++) if (trim($column) == "") errors=errors " cột-" column "-rỗng"
      state=trim($14)
      if (state != "SELECTED" && state != "ELIMINATED") errors=errors " state-sai=" state
      if (errors != "") print "line " NR ":" errors
    }
  ' "$BASELINE_SHEET")"
  if test -z "$invalid_candidates"; then
    pass 'mọi dòng bảng B đúng schema và trạng thái phát hành'
  else
    printf '%s\n' "$invalid_candidates"
    fail 'bảng B có candidate sai schema/trạng thái phát hành'
  fi

  selected="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## B[.] / { inside=1; next }
    /^## C[.] / { inside=0 }
    inside && /^\|/ && trim($14) == "SELECTED" { count++ }
    END { print count + 0 }
  ' "$BASELINE_SHEET")"
  if test "${selected:-0}" -ne 1; then
    fail "số dòng SELECTED=${selected:-0}"
  fi

  selected_id="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## B[.] / { inside=1; next }
    /^## C[.] / { inside=0 }
    inside && /^\|/ && trim($14) == "SELECTED" { print trim($2) }
  ' "$BASELINE_SHEET")"

  mapfile -t candidate_ids < <(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## B[.] / { inside=1; next }
    /^## C[.] / { inside=0 }
    inside && /^\|/ && trim($2) ~ /^K[0-9][0-9]$/ { print trim($2) }
  ' "$BASELINE_SHEET")
  if test "${#candidate_ids[@]}" -eq 0; then
    fail 'bảng B không có candidate'
  else
    dup_ids="$(printf '%s\n' "${candidate_ids[@]}" | sort | uniq -d | tr '\n' ' ')"
    if test -n "$dup_ids"; then
      fail "candidate ID trùng: $dup_ids"
    else
      pass "candidate ID duy nhất (${#candidate_ids[@]} dòng)"
    fi
  fi

  unknown_evidence="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## B[.] / { section="B"; next }
    /^## C[.] / { section="C"; next }
    /^## D[.] / { section="D"; next }
    section == "B" && /^\|/ && trim($2) ~ /^K[0-9][0-9]$/ { candidates[trim($2)]=1 }
    section == "C" && /^\|/ {
      candidate=trim($3)
      if (candidate ~ /^K[0-9][0-9]$/ && !(candidate in candidates)) print "line " NR ": " candidate
    }
  ' "$BASELINE_SHEET")"
  if test -z "$unknown_evidence"; then
    pass 'mọi evidence Kxx tham chiếu candidate có thật'
  else
    printf '%s\n' "$unknown_evidence"
    fail 'bảng C tham chiếu candidate không tồn tại'
  fi

  got_chosen="$(sheet_value 'Candidate được chọn')"
  if test -n "$selected_id" && test "$got_chosen" = "$selected_id"; then
    pass "Candidate được chọn khớp dòng SELECTED ($selected_id)"
  else
    fail "Candidate được chọn='$got_chosen' khác SELECTED='$selected_id'"
  fi

  # Mỗi candidate, kể cả candidate bị loại, phải có ít nhất một dòng bằng chứng.
  for id in "${candidate_ids[@]}"; do
    n="$(awk -F'|' -v wanted="$id" '
      function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
      /^## C[.] / { inside=1; next }
      /^## D[.] / { inside=0 }
      inside && /^\|/ && trim($3) == wanted { count++ }
      END { print count + 0 }
    ' "$BASELINE_SHEET")"
    if test "$n" -ge 1; then pass "candidate $id có bằng chứng"; else fail "candidate $id không có bằng chứng"; fi
  done

  # Candidate SELECTED phải có đúng một evidence cho mỗi gate. Chỉ Ubuntu được
  # dùng Candidate=ALL; chỉ addon được dùng kết quả N/A.
  required_evidence=(K8S-RELEASE RANCHER UBUNTU CONTAINERD KUBE-PACKAGES HELM CERT-MANAGER TRAEFIK FLANNEL ADDON FINAL-RENDER)
  if test -n "$selected_id"; then
    for gate in "${required_evidence[@]}"; do
      counts="$(awk -F'|' -v wanted_id="$selected_id" -v wanted_gate="Gate=$gate" -v gate_name="$gate" '
        function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
        function gate_token(s, a) { split(s, a, /[[:space:]]+/); return a[1] }
        /^## C[.] / { inside=1; next }
        /^## D[.] / { inside=0 }
        inside && /^\|/ {
          candidate=trim($3); check=trim($4); result=trim($5)
          same_candidate=(candidate == wanted_id || (gate_name == "UBUNTU" && candidate == "ALL"))
          if (same_candidate && gate_token(check) == wanted_gate) {
            total++
            if (result == "PASS" || (gate_name == "ADDON" && result == "N/A")) valid++
          }
        }
        END { print total + 0, valid + 0 }
      ' "$BASELINE_SHEET")"
      read -r total valid <<< "$counts"
      if test "$total" -eq 1 && test "$valid" -eq 1; then
        pass "$selected_id có đúng một Gate=$gate hợp lệ"
      else
        fail "$selected_id Gate=$gate: total=$total valid=$valid (cần 1/1)"
      fi
    done
  fi

  release_exact() {
    awk -F'|' -v wanted="$1" '
      function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
      /^## D[.] / { inside=1; next }
      inside && /^## / { inside=0 }
      inside && /^\|/ && trim($2) == wanted { print trim($3); exit }
    ' "$BASELINE_SHEET"
  }
  release_lifecycle() {
    awk -F'|' -v wanted="$1" '
      function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
      /^## D[.] / { inside=1; next }
      inside && /^## / { inside=0 }
      inside && /^\|/ && trim($2) == wanted { print trim($4); exit }
    ' "$BASELINE_SHEET"
  }
  selected_field() {
    awk -F'|' -v column="$1" '
      function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
      /^## B[.] / { inside=1; next }
      /^## C[.] / { inside=0 }
      inside && /^\|/ && trim($14) == "SELECTED" { print trim($column); exit }
    ' "$BASELINE_SHEET"
  }

  # Khóa các input lifecycle đã dùng để tính review date vào đúng cột Lifecycle của
  # bảng D. Nếu không có đối chiếu này, review-computed.env có thể thuộc một bộ ngày
  # khác với bộ đang được phát hành.
  lifecycle_component() {
    case "$1" in
      k8s) printf '%s\n' 'Kubernetes' ;;
      kube-packages) printf '%s\n' 'kubeadm/kubelet/kubectl' ;;
      ubuntu) printf '%s\n' 'Ubuntu' ;;
      containerd) printf '%s\n' 'containerd' ;;
      flannel) printf '%s\n' 'Flannel' ;;
      cert-manager) printf '%s\n' 'cert-manager' ;;
      traefik) printf '%s\n' 'Traefik' ;;
      rancher) printf '%s\n' 'Rancher' ;;
      addon) printf '%s\n' 'addon' ;;
      *) return 1 ;;
    esac
  }
  trim_ws() { printf '%s' "$1" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//'; }

  IFS=',' read -r -a hard_items <<< "$want_hard"
  for item in "${hard_items[@]}"; do
    test -n "$(trim_ws "$item")" || continue
    key="$(trim_ws "${item%%=*}")"
    value="$(trim_ws "${item#*=}")"
    if component="$(lifecycle_component "$key")"; then
      got_lifecycle="$(release_lifecycle "$component")"
      if test "$got_lifecycle" = "DATE-EXACT=$value"; then
        pass "$component lifecycle khớp DATE_EXACT_EOLS"
      else
        fail "$component lifecycle: D='$got_lifecycle' vs tính toán='DATE-EXACT=$value'"
      fi
    else
      fail "DATE_EXACT_EOLS có component không được phép '$key'"
    fi
  done

  IFS=',' read -r -a proxy_items <<< "$want_proxy"
  for item in "${proxy_items[@]}"; do
    test -n "$(trim_ws "$item")" || continue
    key="$(trim_ws "${item%%=*}")"
    value="$(trim_ws "${item#*=}")"
    if component="$(lifecycle_component "$key")"; then
      got_lifecycle="$(release_lifecycle "$component")"
      if [[ "$got_lifecycle" =~ ^(MONTH-PROXY|EVENT-PROXY)=$value$ ]]; then
        pass "$component lifecycle khớp PROXY_LIMITS"
      else
        fail "$component lifecycle: D='$got_lifecycle' vs proxy='$value'"
      fi
    else
      fail "PROXY_LIMITS có component không được phép '$key'"
    fi
  done

  # Chiều ngược lại: dòng bảng D nào có mốc lifecycle thì phải tham gia phép tính
  # review date. Thiếu chiều này, EOL của kube packages/Traefik/Flannel/addon vẫn được
  # phát hành nhưng không bao giờ kéo REVIEW_DATE về sớm hơn — xem Bước 9.2.
  computed_keys=" $(printf '%s,%s' "$want_hard" "$want_proxy" \
    | tr ',' '\n' | sed 's/=.*//; s/[[:space:]]//g' | grep -v '^$' | tr '\n' ' ')"
  for spec in k8s:Kubernetes kube-packages:kubeadm/kubelet/kubectl ubuntu:Ubuntu \
              containerd:containerd flannel:Flannel cert-manager:cert-manager \
              traefik:Traefik rancher:Rancher addon:addon; do
    key="${spec%%:*}"; component="${spec#*:}"
    got_lifecycle="$(release_lifecycle "$component")"
    case "$computed_keys" in
      *" $key "*) in_calc=1 ;;
      *) in_calc=0 ;;
    esac
    if test "$got_lifecycle" = 'N/A'; then
      if test "$in_calc" -eq 0; then
        pass "$component không công bố mốc lifecycle và không tham gia phép tính"
      else
        fail "$component ghi lifecycle N/A ở bảng D nhưng vẫn có key '$key' trong phép tính"
      fi
    elif test "$in_calc" -eq 1; then
      pass "$component có mốc lifecycle và đã tham gia phép tính"
    else
      fail "$component có lifecycle '$got_lifecycle' ở bảng D nhưng không có key '$key' trong DATE_EXACT_EOLS/PROXY_LIMITS"
    fi
  done
  if test "$(release_lifecycle Helm)" = 'N/A'; then
    pass 'Helm không có lifecycle riêng, bảng D ghi N/A'
  else
    fail "dòng Helm của bảng D phải ghi lifecycle N/A, hiện là '$(release_lifecycle Helm)'"
  fi

  required_components=(Kubernetes 'kubeadm/kubelet/kubectl' Ubuntu containerd Flannel Helm cert-manager Traefik Rancher addon)
  release_rows="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## D[.] / { inside=1; next }
    inside && /^## / { inside=0 }
    inside && /^\|/ {
      component=trim($2); version=trim($3)
      if (component != "" && version != "Exact version" && component !~ /^---$/) count++
    }
    END { print count + 0 }
  ' "$BASELINE_SHEET")"
  if test "$release_rows" -eq "${#required_components[@]}"; then
    pass "bảng D có đúng $release_rows component"
  else
    fail "bảng D có $release_rows dòng, cần đúng ${#required_components[@]}"
  fi

  for component in "${required_components[@]}"; do
    n="$(awk -F'|' -v wanted="$component" '
      function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
      /^## D[.] / { inside=1; next }
      inside && /^## / { inside=0 }
      inside && /^\|/ && trim($2) == wanted { count++ }
      END { print count + 0 }
    ' "$BASELINE_SHEET")"
    if test "$n" -eq 1; then pass "bảng D có đúng một dòng $component"; else fail "bảng D cần đúng một dòng $component, hiện có $n"; fi
  done

  invalid_release="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## D[.] / { inside=1; next }
    inside && /^## / { inside=0 }
    inside && /^\|/ {
      component=trim($2); version=trim($3)
      if (component == "" || version == "Exact version" || component ~ /^---$/) next
      for (column=2; column<=7; column++) if (trim($column) == "") bad++
      if (trim($6) != "TECHNICALLY-COMPATIBLE") bad++
    }
    END { print bad + 0 }
  ' "$BASELINE_SHEET")"
  if test "$invalid_release" -eq 0; then pass 'bảng D không có ô rỗng/sai nhãn'; else fail "bảng D có $invalid_release ô rỗng hoặc nhãn sai"; fi

  dynamic_release="$(awk -F'|' '
    function trim(s) { gsub(/^[[:space:]]+|[[:space:]]+$/, "", s); return s }
    /^## D[.] / { inside=1; next }
    inside && /^## / { inside=0 }
    inside && /^\|/ {
      component=trim($2); version=tolower(trim($3))
      if (component != "" && version != "exact version" && component !~ /^---$/ &&
          version ~ /(^|[^[:alnum:]_.-])(latest|stable|master)([^[:alnum:]_.-]|$)/) print component "=" version
    }
  ' "$BASELINE_SHEET")"
  if test -n "$dynamic_release"; then fail "bảng D chứa version động: $dynamic_release"; else pass 'bảng D không chứa version động'; fi

  # Các giá trị phát hành phải được chép nguyên từ đúng dòng SELECTED.
  check_equal_to_selected() {
    component="$1"; column="$2"
    release_value="$(release_exact "$component")"
    selected_value="$(selected_field "$column")"
    if test -n "$release_value" && test "$release_value" = "$selected_value"; then
      pass "$component khớp dòng SELECTED"
    else
      fail "$component lệch SELECTED: D='$release_value' B='$selected_value'"
    fi
  }
  check_equal_to_selected Kubernetes 4
  check_equal_to_selected Rancher 6
  check_equal_to_selected containerd 7
  check_equal_to_selected 'kubeadm/kubelet/kubectl' 8
  check_equal_to_selected Helm 9
  check_equal_to_selected cert-manager 10
  check_equal_to_selected Traefik 11

  flannel_addon="$(selected_field 12)"
  if [[ "$flannel_addon" == *' / '* ]]; then
    selected_flannel="${flannel_addon%% / *}"
    selected_addon="${flannel_addon#* / }"
    if test -n "$selected_flannel" && test "$(release_exact Flannel)" = "$selected_flannel"; then
      pass 'Flannel khớp chính xác phần thứ nhất của ô Flannel/addon'
    else
      fail "Flannel='$(release_exact Flannel)' khác '$selected_flannel'"
    fi
    if test -n "$selected_addon" && test "$(release_exact addon)" = "$selected_addon"; then
      pass 'addon khớp chính xác phần thứ hai của ô Flannel/addon'
    else
      fail "addon='$(release_exact addon)' khác '$selected_addon'"
    fi
  else
    fail "ô Flannel/addon không dùng đúng delimiter ' / ': '$flannel_addon'"
  fi

  # Khóa các biến đã thực sự dùng ở Bước 11 vào bảng D và candidate SELECTED.
  if test "${SELECTED_ID:-}" = "$selected_id"; then pass 'SELECTED_ID khớp phiếu'; else fail "SELECTED_ID='${SELECTED_ID:-}' khác '$selected_id'"; fi
  for spec in 'Kubernetes:K8S_VERSION' 'Rancher:RANCHER_VERSION' 'Helm:HELM_VERSION' \
              'cert-manager:CERT_MANAGER_VERSION' 'Traefik:TRAEFIK_CHART_VERSION' 'Flannel:FLANNEL_VERSION'; do
    component="${spec%%:*}"; variable="${spec#*:}"; value="${!variable:-}"
    release_value="$(release_exact "$component")"
    case "$component" in
      Rancher|Traefik)
        if [[ "$release_value" == *' / '* ]]; then
          expected_value="${release_value%% / *}"
        else
          expected_value=''
          fail "$component không dùng đúng định dạng 'chart / app': '$release_value'"
        fi
        ;;
      *) expected_value="$release_value" ;;
    esac
    if test -z "$value"; then
      fail "chưa export $variable"
    elif test -n "$expected_value" && test "$value" = "$expected_value"; then
      pass "$variable=$value khớp chính xác bảng D"
    else
      fail "$variable='$value' khác exact expected='$expected_value' từ bảng D '$release_value'"
    fi
  done

  expected_render_dir="${BASELINE_DIR:-}/render-check/$selected_id"
  if test -n "${BASELINE_DIR:-}" && test "${RENDER_DIR:-}" = "$expected_render_dir"; then
    pass "RENDER_DIR khớp candidate SELECTED ($selected_id)"
  else
    fail "RENDER_DIR='${RENDER_DIR:-}' khác '$expected_render_dir'"
  fi
  if test -s "${RENDER_DIR:-/nonexistent}/capability-audit.ok"; then
    pass 'capability audit artifact tồn tại'
  else
    fail 'thiếu capability-audit.ok của candidate SELECTED'
  fi
  if test -s "${RENDER_DIR:-/nonexistent}/render-content.ok"; then
    pass 'render content gate artifact tồn tại'
  else
    fail 'thiếu render-content.ok của candidate SELECTED'
  fi
  if python3 - "${RENDER_DIR:-/nonexistent}" <<'PY'
from pathlib import Path
import hashlib
import os
import sys

root = Path(sys.argv[1])
snapshot = {}
snapshot_path = root / "render-inputs.snapshot"
if snapshot_path.is_file():
    for line in snapshot_path.read_text(encoding="utf-8").splitlines():
        if "=" in line:
            key, value = line.split("=", 1)
            snapshot[key] = value
input_keys = [
    "SELECTED_ID", "K8S_VERSION", "RANCHER_VERSION", "HELM_VERSION",
    "CERT_MANAGER_VERSION", "TRAEFIK_CHART_VERSION",
]
for key in input_keys:
    if not snapshot.get(key) or snapshot[key] != os.environ.get(key, ""):
        raise SystemExit(1)

audit_ok = root / "capability-audit.ok"
audit_expected = ""
if audit_ok.is_file():
    for line in audit_ok.read_text(encoding="utf-8").splitlines():
        if line.startswith("AUDIT_SHA256="):
            audit_expected = line.split("=", 1)[1]
audit_files = [
    root / "capability-sites.tsv",
    root / "capabilities-asked.tsv", root / "capabilities-effective.tsv",
    root / "capability-expectations.tsv", root / "lookup-sites.tsv",
    root / "lookup-expectations.tsv", root / "capability-arrays.snapshot",
    root / "capability-versions.snapshot",
]
audit_files.extend(sorted(path for path in (root / "charts").rglob("*") if path.is_file()))
audit_digest = hashlib.sha256()
for path in audit_files:
    if not path.is_file():
        raise SystemExit(1)
    audit_digest.update(str(path.relative_to(root)).encode("utf-8"))
    audit_digest.update(b"\0")
    audit_digest.update(path.read_bytes())
    audit_digest.update(b"\0")
if not audit_expected or audit_digest.hexdigest() != audit_expected:
    raise SystemExit(1)

ok = root / "render-content.ok"
expected = ""
if ok.is_file():
    for line in ok.read_text(encoding="utf-8").splitlines():
        if line.startswith("RENDER_SHA256="):
            expected = line.split("=", 1)[1]
names = [
    "cert-manager-rendered.yaml", "traefik-rendered.yaml", "rancher-rendered.yaml",
    "cert-manager-values.yaml", "traefik-values.yaml", "rancher-values.yaml",
    "capability-audit.ok", "render-inputs.snapshot",
]
digest = hashlib.sha256()
for name in names:
    path = root / name
    if not path.is_file() or path.stat().st_size == 0:
        raise SystemExit(1)
    digest.update(name.encode("utf-8"))
    digest.update(b"\0")
    digest.update(path.read_bytes())
    digest.update(b"\0")
if not expected or digest.hexdigest() != expected:
    raise SystemExit(1)
PY
  then
    pass 'capability/render fingerprint khớp artifact đã qua gate 11.2–11.4'
  else
    fail 'render/value/capability artifact đã đổi sau gate 11.4'
  fi
  for rendered_file in cert-manager-rendered.yaml traefik-rendered.yaml rancher-rendered.yaml; do
    if test -s "${RENDER_DIR:-/nonexistent}/$rendered_file"; then
      pass "render artifact có nội dung: $rendered_file"
    else
      fail "render artifact thiếu/rỗng: $rendered_file"
    fi
  done

  test "$failed" -eq 0 || { echo "FAIL: semantic gate — $failed lỗi"; exit 1; }
  echo 'PASS: semantic gate'
)
```

Ba chi tiết bắt buộc, đừng rút gọn: `test -f` (grep trên file thiếu trả exit 2, mọi `if grep` sẽ im lặng bỏ qua và gate thành pass giả), `|| true` cộng `${selected:-0}` (chuỗi rỗng làm `test` lỗi cú pháp và cũng không vào nhánh FAIL), và `$BASELINE_SHEET` là đường dẫn tuyệt đối từ Bước 0.3.

### 12.3. Phát hành

Phát hành bảng D kèm:

- toàn bộ candidate table, kể cả dòng bị loại;
- Candidate ID được chọn;
- URL và ngày tra của từng gate;
- exact package/chart/app/image versions;
- lifecycle limit, review date và limiting cause;
- nhãn toàn bộ baseline: `TECHNICALLY-COMPATIBLE`.

Kết luận hợp lệ của runbook tra cứu:

```text
SELECTED-CONDITIONAL: candidate <ID>; mọi gate tra cứu/metadata/render PASS;
CONDITION: CNI VERSION test phải PASS trong runbook cài đặt trước khi mở cluster.
```

Hoặc, sau khi vét hết candidate:

```text
BLOCKED: không còn candidate; xem từng gate loại trong bảng B.
```

Hai chuỗi trên là nội dung **bản phát hành**. Dòng `- Kết luận phiên:` trong phiếu chỉ mang đúng token `SELECTED-CONDITIONAL` (hoặc `BLOCKED`) — Bước 12.2 so bằng chuỗi chính xác, nên chép cả câu dài vào đó sẽ FAIL. Candidate ID đã nằm ở dòng `- Candidate được chọn:`, không lặp lại.

---

## Phụ lục — Nguồn official bắt buộc

| Thành phần | Nguồn |
| --- | --- |
| Kubernetes lifecycle/patch | [Kubernetes Releases](https://kubernetes.io/releases/) |
| Kubernetes packages | [Installing kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) |
| Kubernetes runtime | [Container Runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) |
| Rancher channel | [Choosing a Rancher Version](https://ranchermanager.docs.rancher.com/getting-started/installation-and-upgrade/resources/choose-a-rancher-version) |
| Rancher support | [SUSE Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/) |
| Rancher lifecycle | [SUSE Product Lifecycle](https://www.suse.com/lifecycle/#rancher) |
| containerd | [RELEASES.md](https://github.com/containerd/containerd/blob/main/RELEASES.md) |
| Ubuntu lifecycle/security | [Release cycle](https://ubuntu.com/about/release-cycle), [`pro security-status`](https://documentation.ubuntu.com/pro-client/en/latest/explanations/how_to_interpret_the_security_status_command/) |
| Helm | [Version Support Policy](https://helm.sh/docs/v3/topics/version_skew/), [Releases](https://github.com/helm/helm/releases) |
| cert-manager | [Supported Releases](https://cert-manager.io/docs/releases/), [Helm installation](https://cert-manager.io/docs/installation/helm/) |
| Traefik | [Chart releases](https://github.com/traefik/traefik-helm-chart/releases), [Kubernetes docs](https://doc.traefik.io/traefik/setup/kubernetes/) |
| Flannel | [Releases](https://github.com/flannel-io/flannel/releases), [CNI Specification](https://www.cni.dev/docs/spec/) |
| local-path-provisioner | [Releases](https://github.com/rancher/local-path-provisioner/releases) |
| MetalLB | [Releases](https://github.com/metallb/metallb/releases), [Documentation](https://metallb.io/) |
| cloudflared | [Downloads](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/downloads/), [Releases](https://github.com/cloudflare/cloudflared/releases) |

Trang “current” thay đổi theo thời gian. Phiếu phải ghi ngày tra cạnh mỗi URL và lưu metadata của exact artifact; không sao chép version từ một phiếu đã hết hạn.
