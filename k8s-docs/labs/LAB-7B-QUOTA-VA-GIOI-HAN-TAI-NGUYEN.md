# Lab 7b — Quota và giới hạn tài nguyên

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do Lab 6a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** cleanup trả cluster về đúng `03-storage-ready`, **không tạo snapshot mới**.
> **Lab trước:** [Lab 7a — Lập lịch và eviction](LAB-7A-LAP-LICH-VA-EVICTION.md) cũng trả cluster
> về `03-storage-ready`. Cluster vào lab này phải ở đúng mốc đó: có provisioner và StorageClass
> mặc định, không workload, không object nào của lab trước.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[7b. Chính sách giới hạn tài nguyên](../00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên)
— sáu bài [132](../132-policies-vi.md), [133](../133-limit-range-vi.md),
[134](../134-resource-quotas-vi.md), [135](../135-pid-limiting-vi.md),
[74](../74-resource-managers-vi.md) và [154](../154-node-declared-features-vi.md). Nhóm 7b
**không có dòng "Thực hành"** trong lộ trình: loạt trang task về quota và limit range nằm ở
[giai đoạn 25](../00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace), tức
sau lab này, nên lab bám thẳng vào sáu bài khái niệm.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không
chép lại con số phiên bản nào** và **không cài thêm bất cứ thứ gì**. Thành phần duy nhất ngoài
baseline mà lab chạm tới là StorageClass mặc định do Lab 6a để lại; version của nó nằm ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00), và lab **đọc tên
class từ cluster** chứ không viết cứng.

**Lab 7b nói về chính sách ở cấp namespace và cấp node.** Phần `requests`/`limits` của từng
container, QoS class và cgroup đã làm xong ở
[Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md); lab này không lặp lại. Ở đây câu hỏi khác hẳn:
ai đặt trần cho **cả một namespace**, ai điền giá trị cho Pod **không khai gì**, và những chính
sách nào **không đi qua API server** mà nằm trên kubelet của từng node.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm hai lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Hai lệnh riêng của lab 7b: chưa namespace nào có chính sách tài nguyên.
kubectl get resourcequota -A
kubectl get limitrange -A
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; **hai lệnh cuối đều trả
`No resources found`** — cluster chưa có ResourceQuota và chưa có LimitRange nào.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Bài [132](../132-policies-vi.md) chia việc áp chính sách làm hai mặt phẳng, và chỉ ra được
  **mặt phẳng nào cưỡng chế ở API server** (có object API, `kubectl` sờ được) còn **mặt phẳng nào
  cưỡng chế trên node** (không có object API nào, phải đọc cấu hình kubelet).
- **LimitRange** là trần và mặc định cho **từng object**: chứng minh được Pod không khai tài
  nguyên vẫn được tạo và bị admission controller **chèn** `defaultRequest`/`default` vào, còn Pod
  vượt `max` hoặc dưới `min` bị **từ chối ngay lúc tạo** — và đọc đúng câu thông báo chỉ ra ràng
  buộc nào bị vi phạm.
- Cái bẫy của bài [133](../133-limit-range-vi.md): LimitRange **không kiểm tra tính nhất quán của
  chính giá trị mặc định nó áp**, nên một LimitRange hoàn toàn hợp lệ vẫn tạo ra Pod tự mâu
  thuẫn; và sửa `max` xuống thấp hơn Pod đang chạy thì Pod đó **không hề bị đụng tới**.
- **ResourceQuota** là trần **tổng** của cả namespace: đọc được `used` so với `hard`, chứng minh
  Pod vượt trần bị từ chối, và phân biệt được hai kiểu từ chối khác hẳn nhau — *thiếu khai báo*
  và *vượt trần*.
- Quota theo **số lượng object** và theo **lưu trữ** chặn được thứ mà trần CPU/memory không chạm
  tới, kể cả khi object đó không tiêu thụ một chút CPU nào.
- Bẫy Deployment của bài [134](../134-resource-quotas-vi.md): `kubectl apply` một Deployment vượt
  quota vẫn **thành công**, chỉ là Pod không sinh ra đủ — và biết tìm nguyên nhân ở đâu.
- **Mối liên hệ cốt lõi của nhóm 7b:** namespace có quota cho `cpu`/`memory` thì Pod **bắt buộc**
  phải khai `requests`/`limits`; thêm LimitRange là cách để người dùng không phải nhớ điều đó —
  và chứng minh được quota đếm **đúng giá trị mà LimitRange vừa chèn vào**.
- ResourceQuota **độc lập với dung lượng cluster** và **không tạo ràng buộc nào về node**: chứng
  minh bằng một trần lớn hơn cả cluster vẫn được API server chấp nhận, và bằng Pod của hai
  namespace cùng nằm trên một node.
- **Giới hạn PID** không khai được trong `.spec` của Pod: chứng minh bằng chính phản hồi của API
  server, rồi đọc `podPidsLimit` thật của kubelet trên cả ba node và thấy hệ quả của nó ở cgroup
  của một Pod đang chạy.
- **Resource manager của kubelet**: đọc được policy đang chạy của CPU manager, memory manager và
  topology manager, và chứng minh hệ quả — Pod `Guaranteed` xin CPU số nguyên **không** được cấp
  CPU độc quyền dưới policy mặc định.
- **Tính năng do Node khai báo** chưa tồn tại trên baseline: chứng minh bằng chính object Node.
- Cleanup đúng phạm vi: xóa hết object của lab, chứng minh **cấu hình kubelet không hề bị sửa**,
  và đưa cluster về đúng `03-storage-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 7b | Kiểm chứng ở |
| --- | --- |
| [132 — Chính sách](../132-policies-vi.md) | B1 — trang khung phân loại. B1.1 chứng minh nhánh API: LimitRange, ResourceQuota và NetworkPolicy đều là object API phạm vi namespace. B1.2 chứng minh nhánh kubelet không có object API nào. Toàn bộ B2–B5 là nhánh thứ nhất, B6–B7 là nhánh thứ hai |
| [133 — Khoảng giới hạn tài nguyên](../133-limit-range-vi.md) | B2 — `default`/`defaultRequest` được chèn vào Pod trần (B2.3), `max` và `min` từ chối ngay lúc tạo (B2.4, B2.5), cái bẫy `default` nhỏ hơn `requests` của client (B2.6), sửa LimitRange không đụng Pod đang chạy (B2.7); B5 — LimitRange đứng cạnh ResourceQuota |
| [134 — Hạn ngạch tài nguyên](../134-resource-quotas-vi.md) | B3 — trần tổng CPU/memory, `used` so với `hard`, hai kiểu từ chối, bẫy Deployment, quota không ràng buộc node, quota lớn hơn dung lượng cluster; B4 — quota theo số lượng object và theo lưu trữ; B5 — quota buộc mọi Pod phải khai tài nguyên |
| [135 — Giới hạn và dự trữ Process ID](../135-pid-limiting-vi.md) | B6 — `pids` không phải tài nguyên khai được trong Pod spec (B6.1), `podPidsLimit` thật của kubelet trên cả ba node (B6.2), hệ quả ở cgroup của Pod đang chạy (B6.3), `PIDPressure` và tín hiệu eviction `pid.available` (B6.4) |
| [74 — Các trình quản lý tài nguyên](../74-resource-managers-vi.md) | B7 — policy đang chạy của CPU/memory/topology manager đọc từ cấu hình kubelet thật (B7.1), hệ quả của policy `none` trên một Pod `Guaranteed` xin CPU số nguyên (B7.2), chính sách này không nằm trong object Node (B7.3), topology thật của node lab (B7.4) |
| [154 — Tính năng do Node khai báo](../154-node-declared-features-vi.md) | B8 — bài được lộ trình đánh dấu **đọc lướt**; lab kiểm chứng đúng phần đọc được: `.status.declaredFeatures` chưa tồn tại trên object Node của baseline |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [132](../132-policies-vi.md): admission controller tích hợp, `ValidatingAdmissionPolicy`, admission webhook | Cần chuỗi authn → authz → admission và cờ của kube-apiserver. Thuộc [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài [119](../119-controlling-access-vi.md) và [173](../173-admission-webhooks-vi.md) |
| Bài [133](../133-limit-range-vi.md): hai LimitRange trong cùng một namespace | Bài khẳng định kết quả là **không xác định (not deterministic)**. Không viết được gate `PASS:` nào đúng cho một hành vi không xác định, nên bước này bị loại theo đúng quy ước của thư mục lab |
| Bài [133](../133-limit-range-vi.md): `maxLimitRequestRatio`, và `min`/`max` cho `type: PersistentVolumeClaim` | Chính bài xếp hai phần này vào *Đọc lướt* và hoãn tới nhóm task [giai đoạn 25](../00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace) |
| Bài [134](../134-resource-quotas-vi.md): toàn bộ *Phạm vi hạn ngạch* — `BestEffort`, `Terminating`, `PriorityClass`, `CrossNamespacePodAffinity`, `VolumeAttributesClass` | Bài xếp vào *Đọc lướt*. Hai ví dụ mạnh nhất còn cần cờ `--admission-control-config-file` của kube-apiserver, tức sửa control plane — thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). PriorityClass thuộc nhóm 7a |
| Bài [134](../134-resource-quotas-vi.md): quota cho tài nguyên mở rộng, DRA resource claim, `hugepages-<size>`, `requests.ephemeral-storage` | DRA thuộc [giai đoạn 13](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao); device plugin thuộc giai đoạn 14; hai worker VM không cấu hình hugepages nên `hugepages-2Mi` của node bằng 0; quota lưu trữ tạm thời còn ở mức alpha |
| Bài [134](../134-resource-quotas-vi.md): `count/<resource>.<group>` cho custom resource | Cần CRD, thuộc [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng). B4 vẫn dùng cú pháp `count/` nhưng cho tài nguyên chuẩn |
| Bài [135](../135-pid-limiting-vi.md): đặt `podPidsLimit`, và dự trữ `pid=<number>` trong `--system-reserved` / `--kube-reserved` | Phải sửa cấu hình kubelet rồi restart kubelet trên từng node. Lab này **cấm sửa cấu hình kubelet**; việc đó thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài [224](../224-kubelet-config-file-vi.md) và [253](../253-reserve-compute-resources-vi.md). B6 chỉ **đọc** giá trị thật |
| Bài [135](../135-pid-limiting-vi.md): chạy fork bomb để thấy giới hạn chặn lại | Trên baseline **không có giới hạn PID nào đang bật** — B6.2 và B6.3 chứng minh điều đó — nên fork bomb chỉ làm mất ổn định chính `lab-k8s-worker2` mà không chứng minh được gì. Bài cũng nói rõ tín hiệu eviction `pid.available` **không cưỡng chế giới hạn**, nên không có đường vòng nào an toàn |
| Bài [74](../74-resource-managers-vi.md): đổi policy sang `static` để thấy CPU độc quyền | Đổi policy đòi sửa cấu hình kubelet, **restart kubelet và xóa state file** của manager — ba việc lab này cấm. Hai worker lại là VM 2 vCPU một NUMA node (B7.4) nên không có ranh giới NUMA/socket nào để căn chỉnh. Đọc tiếp ở [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) cho phần đổi cấu hình, và [giai đoạn 25](../00-ALO-TRINH-ADMIN.md#giai-đoạn-25--quản-trị-tài-nguyên-theo-namespace) cho ba bài [200](../200-cpu-management-policies-vi.md), [235](../235-memory-manager-vi.md), [259](../259-topology-manager-vi.md) |
| Bài [74](../74-resource-managers-vi.md): các tùy chọn của chính sách static, trình quản lý tài nguyên **cấp Pod**, và Device Manager | Các tùy chọn static đều là tinh chỉnh theo NUMA/socket/uncore cache của phần cứng thật; phần cấp Pod còn ở mức alpha và cần feature gate; Device Manager cần device plugin — [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng), bài [184](../184-device-plugins-vi.md) |
| Bài [154](../154-node-declared-features-vi.md): plugin `NodeDeclaredFeatures` của scheduler và admission controller `NodeDeclaredFeatureValidator` | Tính năng ra sau baseline và còn phải bật feature gate trên cả `kube-apiserver`, `kube-scheduler` và `kubelet` — thuộc [giai đoạn 8](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) bài [03](../03-control-plane-flags-vi.md) và [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) bài [196](../196-configure-feature-gates-vi.md). B8 kiểm chứng đúng phần đọc được |

> Trang tra cứu [313 — Khắc phục sự cố Topology Management](../313-debug-topology-vi.md) **không
> thuộc nhóm 7b**: lộ trình xếp nó vào dòng thực hành của
> [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) và nó chỉ dùng được khi
> Topology Manager chạy policy khác `none`. Đọc nó khi vận hành máy chủ vật lý nhiều socket, không
> phải ở lab này.

### 1.2. Thời lượng

2–3 giờ, tính từ lúc gate mở đầu đã PASS. Lab không cài gì và không có bước chờ controller lâu;
phần lớn thời gian nằm ở việc đọc kỹ từng thông báo từ chối. Các bước phải chờ kubelet hoặc
controller hội tụ đều viết dưới dạng vòng lặp có điều kiện thoát, không phải con số cố định.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi rõ
  node khác**. Lệnh cần `sudo` để đọc file trên node chạy trên chính node đó qua `ssh`.
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Gần như mọi gate so sánh với biến và hai
  hàm trợ giúp đặt ở B0 (`cpu_m`, `mem_mi`, `MIN_W_*`, `LR_*`, `Q_*`); mở shell mới giữa chừng là
  mất hết và các gate sau sẽ sai.
- **Lab này chỉ ĐỌC cấu hình node.** Tuyệt đối không sửa `/var/lib/kubelet/config.yaml`, không
  restart kubelet, không xóa state file trong `/var/lib/kubelet/`, không sửa manifest trong
  `/etc/kubernetes/manifests`. B0 ghi checksum cấu hình kubelet của cả ba node và B9 đối chiếu
  lại — đó là gate cuối chứng minh bạn không đụng vào.
- **Không cài thêm hạ tầng**, không tạo snapshot mới. Cluster phải trở về đúng `03-storage-ready`.
- Lab tạo đúng hai Namespace: `lab-7b` (namespace bị quản trị) và `lab-7b-free` (namespace đối
  chứng, cố ý **không** có chính sách nào). Cả hai bị xóa ở B9. Ngoài hai namespace đó, lab
  **không tạo object phạm vi cluster nào**.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Lab này không có bước phá hoại; hai bước ghim
  Pod vào `lab-k8s-worker2` (B6.3, B7.2) là bước **đọc** cgroup và CPU affinity, không phá gì.
- Kiến thức được phép dùng làm công cụ: Pod, ConfigMap, PVC/StorageClass, Deployment, và
  `nodeSelector` vừa học ở [Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md). **Không** dùng
  RBAC/Role/RoleBinding (giai đoạn 9), HPA và metrics-server (giai đoạn 11), DRA (giai đoạn 13),
  PriorityClass và topology spread constraint (nhóm 7a — lab này không lặp lại chúng). Lab
  **không dùng `kubectl top`**.
- Image dùng cho toàn bộ lab là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và đã có sẵn trên cả ba node từ
  Lab 00, nên lab không phụ thuộc mạng ra ngoài.
- **Mọi ngưỡng trong lab được tính từ `.status.allocatable` thật của hai worker**, đọc bằng
  `jsonpath` ở B0. Không có con số nào viết cứng trong manifest.
- Manifest tạm ghi vào `~/lab-work/7b/`; bằng chứng ghi vào `~/lab-evidence/7b/`. Không lưu token,
  key hay certificate.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.

**Cách quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `03-storage-ready` — không bao
giờ restore riêng một VM, xem ghi chú cuối
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) — rồi bật lại theo
thứ tự master → worker 1 → worker 2, chạy lại gate mở đầu và bắt đầu lại từ B0.

---

# Phần B — Thực hành kiến thức 7b

## B0. Chuẩn bị workspace, hai namespace và số liệu nền

**Mục đích:** dựng chỗ làm việc, tạo namespace bị quản trị và namespace đối chứng, rồi **tính mọi
ngưỡng của lab từ `.status.allocatable` thật** — không có con số nào bịa ra.

### B0.1. Workspace và hai namespace

```bash
mkdir -p ~/lab-work/7b ~/lab-evidence/7b
kubectl config current-context
kubectl get nodes -o wide
kubectl create namespace lab-7b
kubectl create namespace lab-7b-free
```

```bash
{
  echo "=== $(date -Is) — trang thai chinh sach truoc Lab 7b ==="
  echo '--- resourcequota ---'; kubectl get resourcequota -A 2>&1
  echo '--- limitrange ---';    kubectl get limitrange -A 2>&1
  echo '--- storageclass ---';  kubectl get storageclass 2>&1
  echo '--- persistentvolume ---'; kubectl get persistentvolume 2>&1
} | tee ~/lab-evidence/7b/b0-truoc.txt

RQ_N="$(kubectl get resourcequota -A --no-headers 2>/dev/null | wc -l)"
LR_N="$(kubectl get limitrange -A --no-headers 2>/dev/null | wc -l)"
P1="$(kubectl get namespace lab-7b      -o jsonpath='{.status.phase}')"
P2="$(kubectl get namespace lab-7b-free -o jsonpath='{.status.phase}')"
echo "resourcequota=$RQ_N | limitrange=$LR_N | lab-7b=$P1 | lab-7b-free=$P2"

test "$RQ_N" -eq 0 && test "$LR_N" -eq 0 \
  && echo 'PASS: chua namespace nao co ResourceQuota hay LimitRange'
test "$P1" = 'Active' && test "$P2" = 'Active' \
  && echo 'PASS: hai namespace cua lab da Active'
```

**Ý nghĩa:** `lab-7b` là namespace sẽ nhận chính sách; `lab-7b-free` là **đối chứng** — cùng một
manifest, chạy ở đây thì lọt, chạy ở kia thì bị chặn. Bài [134](../134-resource-quotas-vi.md) nói
hạn ngạch chỉ được ép buộc trong namespace **thực sự có** một đối tượng ResourceQuota; hai
namespace cạnh nhau là cách chứng minh câu đó thay vì tin lời.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B0.2. Hai hàm trợ giúp và số liệu nền của hai worker

Quantity của Kubernetes có nhiều dạng viết cho cùng một giá trị (`1` và `1000m`, `1Gi` và
`1024Mi`), nên mọi so sánh trong lab đi qua hai hàm chuẩn hóa dưới đây. Định nghĩa chúng **một
lần** và giữ tới hết phần B:

```bash
cpu_m()  { case "$1" in ''|'<none>') echo 0 ;; *m) echo "${1%m}" ;; *) echo "$(( $1 * 1000 ))" ;; esac; }
mem_mi() { case "$1" in ''|'<none>') echo 0 ;; *Ki) echo "$(( ${1%Ki} / 1024 ))" ;; *Mi) echo "${1%Mi}" ;; *Gi) echo "$(( ${1%Gi} * 1024 ))" ;; *) echo 0 ;; esac; }

test "$(cpu_m 500m)" -eq 500 && test "$(cpu_m 2)" -eq 2000 \
  && test "$(mem_mi 1Gi)" -eq 1024 && test "$(mem_mi 2048Ki)" -eq 2 \
  && echo 'PASS: hai ham chuan hoa quantity hoat dong dung'
```

Chỉ đọc hai worker: `lab-k8s-master` mang taint `NoSchedule` nên không nhận Pod thường, và trần
của namespace không có nghĩa gì nếu tính cả phần không dùng được.

```bash
MIN_W_CPU_M=0; MIN_W_MEM_MI=0; SUM_W_CPU_M=0; SUM_W_MEM_MI=0
for n in lab-k8s-worker1 lab-k8s-worker2; do
  V_CPU="$(cpu_m  "$(kubectl get node "$n" -o jsonpath='{.status.allocatable.cpu}')")"
  V_MEM="$(mem_mi "$(kubectl get node "$n" -o jsonpath='{.status.allocatable.memory}')")"
  echo "$n allocatable: ${V_CPU}m CPU / ${V_MEM}Mi memory"
  SUM_W_CPU_M=$(( SUM_W_CPU_M + V_CPU ));  SUM_W_MEM_MI=$(( SUM_W_MEM_MI + V_MEM ))
  if [ "$MIN_W_CPU_M" -eq 0 ] || [ "$V_CPU" -lt "$MIN_W_CPU_M" ]; then MIN_W_CPU_M="$V_CPU"; fi
  if [ "$MIN_W_MEM_MI" -eq 0 ] || [ "$V_MEM" -lt "$MIN_W_MEM_MI" ]; then MIN_W_MEM_MI="$V_MEM"; fi
done
echo "worker nho nhat: ${MIN_W_CPU_M}m / ${MIN_W_MEM_MI}Mi"
echo "tong hai worker: ${SUM_W_CPU_M}m / ${SUM_W_MEM_MI}Mi"

test "$MIN_W_CPU_M" -gt 0 && test "$MIN_W_MEM_MI" -gt 0 \
  && echo 'PASS: doc duoc allocatable that cua ca hai worker'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện, và hai worker in ra con số khác 0.

### B0.3. Sinh bộ ngưỡng của lab

Ba ngưỡng của LimitRange lấy theo **worker nhỏ nhất** — vì LimitRange ràng buộc *từng* container,
mà một container phải vừa một node. Bốn ngưỡng của ResourceQuota lấy theo **bội của giá trị mặc
định**, để về sau đếm được chính xác bao nhiêu Pod thì đầy trần:

```bash
LR_MIN_CPU_M=$(( MIN_W_CPU_M / 50 ))
LR_DEF_CPU_M=$(( MIN_W_CPU_M / 10 ))
LR_MAX_CPU_M=$(( MIN_W_CPU_M / 4 ))
LR_MIN_MEM_MI=$(( MIN_W_MEM_MI / 200 ))
LR_DEF_MEM_MI=$(( MIN_W_MEM_MI / 50 ))
LR_MAX_MEM_MI=$(( MIN_W_MEM_MI / 10 ))

Q_REQ_CPU_M=$(( LR_DEF_CPU_M * 3 ));   Q_LIM_CPU_M=$(( LR_DEF_CPU_M * 6 ))
Q_REQ_MEM_MI=$(( LR_DEF_MEM_MI * 3 )); Q_LIM_MEM_MI=$(( LR_DEF_MEM_MI * 6 ))

{
  echo "=== $(date -Is) — nguong sinh tu allocatable that ==="
  echo "LimitRange  cpu:  min=${LR_MIN_CPU_M}m  default=${LR_DEF_CPU_M}m  max=${LR_MAX_CPU_M}m"
  echo "LimitRange  mem:  min=${LR_MIN_MEM_MI}Mi default=${LR_DEF_MEM_MI}Mi max=${LR_MAX_MEM_MI}Mi"
  echo "Quota  requests:  cpu=${Q_REQ_CPU_M}m  memory=${Q_REQ_MEM_MI}Mi"
  echo "Quota  limits:    cpu=${Q_LIM_CPU_M}m  memory=${Q_LIM_MEM_MI}Mi"
} | tee ~/lab-evidence/7b/b0-nguong.txt
```

```bash
test "$LR_MIN_CPU_M" -gt 0 && test "$LR_MIN_CPU_M" -lt "$LR_DEF_CPU_M" \
  && test "$LR_DEF_CPU_M" -lt "$LR_MAX_CPU_M" \
  && test "$LR_MIN_MEM_MI" -gt 0 && test "$LR_MIN_MEM_MI" -lt "$LR_DEF_MEM_MI" \
  && test "$LR_DEF_MEM_MI" -lt "$LR_MAX_MEM_MI" \
  && echo 'PASS: bo nguong LimitRange thoa min < default < max o ca hai truc'
test "$Q_REQ_CPU_M" -lt "$SUM_W_CPU_M" && test "$Q_REQ_MEM_MI" -lt "$SUM_W_MEM_MI" \
  && echo 'PASS: tran quota nam duoi tong allocatable, tuc quota se la thu chan truoc scheduler'
```

**Ý nghĩa:** đặt quota **thấp hơn** tổng allocatable là có chủ đích. Nếu quota cao hơn dung lượng
thật thì thứ chặn Pod sẽ là scheduler chứ không phải quota, và bạn sẽ học nhầm bài. B3.7 đảo
ngược tình huống này một cách có kiểm soát để thấy đúng ranh giới giữa hai cơ chế.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B0.4. Ảnh chụp "trước" của cấu hình kubelet

Lab này đọc cấu hình kubelet ở B6 và B7. Chụp lại ngay bây giờ để B9 chứng minh **không có gì bị
sửa**. Cấu hình *hiệu lực* — gồm cả giá trị mặc định mà file không viết ra — đọc qua endpoint
`configz` của kubelet, đi đúng đường control plane → kubelet mà
[tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet) đã kiểm:

```bash
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  kubectl get --raw "/api/v1/nodes/$n/proxy/configz" \
    > ~/lab-evidence/7b/b0-configz-$n.json 2>/dev/null || true
  echo "$n -> $(wc -c < ~/lab-evidence/7b/b0-configz-$n.json) byte"
done

OKZ=0
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  grep -q '"kubeletconfig"' ~/lab-evidence/7b/b0-configz-$n.json && OKZ=$(( OKZ + 1 ))
done
echo "configz doc duoc tren $OKZ/3 node"
test "$OKZ" -eq 3 && echo 'PASS: doc duoc cau hinh hieu luc cua ca ba kubelet'
```

Và checksum của **file** cấu hình trên từng node — thứ B9 sẽ so lại:

```bash
{
  echo "lab-k8s-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$n $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
} | tee ~/lab-evidence/7b/b0-kubelet-sha.txt

test "$(wc -l < ~/lab-evidence/7b/b0-kubelet-sha.txt)" -eq 3 \
  && test "$(awk '{print $2}' ~/lab-evidence/7b/b0-kubelet-sha.txt | grep -c '^[0-9a-f]\{64\}$')" -eq 3 \
  && echo 'PASS: ghi duoc checksum cau hinh kubelet cua ca ba node'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Nếu `configz` không đọc được, dừng lại và xem
[mục 4](#4-troubleshooting-của-lab-này) trước khi đi tiếp — B6 và B7 dựa hẳn vào file này.

---

## B1. Hai mặt phẳng cưỡng chế chính sách

**Mục đích:** dựng khung phân loại của bài [132](../132-policies-vi.md) bằng lệnh, để suốt phần
còn lại bạn luôn biết mình đang đứng ở mặt phẳng nào. Bài chia chính sách thành hai nhánh theo
**nơi cưỡng chế**: trong API server, và trên kubelet của từng worker node.

### B1.1. Nhánh API — chính sách là object, và object thì `kubectl` sờ được

```bash
kubectl api-resources --namespaced=true \
  | grep -E '^(limitranges|resourcequotas|networkpolicies) ' \
  | tee ~/lab-evidence/7b/b1-api-resources.txt

API_N="$(wc -l < ~/lab-evidence/7b/b1-api-resources.txt)"
echo "so object API dong vai tro chinh sach ma bai 132 liet ke = $API_N"
test "$API_N" -eq 3 \
  && echo 'PASS: ca ba deu la object API pham vi namespace'
```

```bash
kubectl explain limitrange.spec.limits  > ~/lab-evidence/7b/b1-explain-limitrange.txt
kubectl explain resourcequota.spec      > ~/lab-evidence/7b/b1-explain-resourcequota.txt

for f in default defaultRequest max min type; do
  grep -q "^ *$f" ~/lab-evidence/7b/b1-explain-limitrange.txt \
    || echo "FAIL: LimitRange khong co truong $f"
done
grep -q '^ *hard' ~/lab-evidence/7b/b1-explain-resourcequota.txt \
  && echo 'PASS: ResourceQuota co truong hard trong spec'
grep -q '^ *default' ~/lab-evidence/7b/b1-explain-limitrange.txt \
  && grep -q '^ *defaultRequest' ~/lab-evidence/7b/b1-explain-limitrange.txt \
  && echo 'PASS: LimitRange co du default va defaultRequest trong spec'
```

**Ý nghĩa:** ba object này là ví dụ mà bài [132](../132-policies-vi.md) đưa ra cho mục *Áp dụng
chính sách bằng các đối tượng API*. NetworkPolicy bạn đã làm ở giai đoạn 5; hai cái còn lại là
nội dung của B2–B5. Điểm chung: chúng **có schema, có namespace, có `kubectl explain`** — nghĩa là
cưỡng chế nằm trong API server.

**PASS:** không có dòng `FAIL:` nào; ba dòng `PASS:` xuất hiện.

### B1.2. Nhánh kubelet — chính sách không có object nào để `kubectl get`

```bash
kubectl api-resources 2>/dev/null \
  | grep -ciE 'pidlimit|cpumanager|memorymanager|topologymanager|kubeletconfig' \
  | tee ~/lab-evidence/7b/b1-api-node-policy.txt

NODE_API_N="$(cat ~/lab-evidence/7b/b1-api-node-policy.txt)"
echo "so object API cho chinh sach kubelet = $NODE_API_N"
test "$NODE_API_N" -eq 0 \
  && echo 'PASS: khong co object API nao cho chinh sach nam tren kubelet'
```

```bash
OKF=0
sudo test -f /var/lib/kubelet/config.yaml && OKF=$(( OKF + 1 ))
for n in lab-k8s-worker1 lab-k8s-worker2; do
  ssh "$n" 'sudo test -f /var/lib/kubelet/config.yaml' && OKF=$(( OKF + 1 ))
done
echo "so node co file cau hinh kubelet = $OKF"
test "$OKF" -eq 3 \
  && echo 'PASS: chinh sach nhanh thu hai song trong file tren tung node, khong trong etcd'
```

**Ý nghĩa:** đây là ranh giới quan trọng nhất của bài [132](../132-policies-vi.md). Hai bài
[135](../135-pid-limiting-vi.md) và [74](../74-resource-managers-vi.md) ở cuối nhóm 7b nằm trọn
trong nhánh này: muốn đổi thì phải đụng vào **từng node**, không có cách nào `kubectl apply` một
file YAML là xong. Hệ quả trực tiếp: một cluster hai worker cấu hình lệch nhau sẽ áp hai chính
sách khác nhau cho cùng một Pod, tùy nó rơi vào đâu — B6.2 kiểm chứng đúng điều đó.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B2. LimitRange — trần và mặc định cho từng object

**Mục đích:** chứng minh bốn khả năng mà bài [133](../133-limit-range-vi.md) liệt kê, theo đúng
thứ tự hai bước mà mục *Ràng buộc đối với limit và request tài nguyên* mô tả: **trước hết** điền
mặc định, **sau đó** mới soi min/max.

### B2.1. Mốc đối chiếu: namespace chưa có chính sách thì không ép gì

```bash
cat > ~/lab-work/7b/b2-tran.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: tran
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/7b/b2-tran.yaml -n lab-7b
kubectl wait --for=condition=Ready pod/tran -n lab-7b --timeout=120s
```

```bash
RES="$(kubectl get pod tran -n lab-7b -o jsonpath='{.spec.containers[0].resources}')"
QOS="$(kubectl get pod tran -n lab-7b -o jsonpath='{.status.qosClass}')"
echo "resources = ${RES:-<rong>} ; qosClass = $QOS"

test "$RES" = '{}' \
  && echo 'PASS: khong co LimitRange thi Pod tran giu nguyen resources rong'

kubectl delete pod tran -n lab-7b --wait=true --timeout=120s
```

**Ý nghĩa:** bài [133](../133-limit-range-vi.md) mở đầu bằng đúng câu này — theo mặc định container
chạy với tài nguyên tính toán **không bị giới hạn**. Đây là trạng thái xuất phát; mọi thứ B2 làm
sau đây là thay đổi so với nó.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B2.2. Một LimitRange với đủ bốn nhóm giá trị

```bash
cat > ~/lab-work/7b/b2-limitrange.yaml <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: lab-7b-limits
  namespace: lab-7b
spec:
  limits:
  - type: Container
    default:
      cpu: "${LR_DEF_CPU_M}m"
      memory: "${LR_DEF_MEM_MI}Mi"
    defaultRequest:
      cpu: "${LR_DEF_CPU_M}m"
      memory: "${LR_DEF_MEM_MI}Mi"
    max:
      cpu: "${LR_MAX_CPU_M}m"
      memory: "${LR_MAX_MEM_MI}Mi"
    min:
      cpu: "${LR_MIN_CPU_M}m"
      memory: "${LR_MIN_MEM_MI}Mi"
EOF

kubectl apply -f ~/lab-work/7b/b2-limitrange.yaml
kubectl describe limitrange lab-7b-limits -n lab-7b | tee ~/lab-evidence/7b/b2-limitrange.txt
```

```bash
G_TYPE="$(kubectl get limitrange lab-7b-limits -n lab-7b -o jsonpath='{.spec.limits[0].type}')"
G_MAX="$( kubectl get limitrange lab-7b-limits -n lab-7b -o jsonpath='{.spec.limits[0].max.cpu}')"
G_MIN="$( kubectl get limitrange lab-7b-limits -n lab-7b -o jsonpath='{.spec.limits[0].min.cpu}')"
G_DEF="$( kubectl get limitrange lab-7b-limits -n lab-7b -o jsonpath='{.spec.limits[0].default.cpu}')"
G_DRQ="$( kubectl get limitrange lab-7b-limits -n lab-7b -o jsonpath='{.spec.limits[0].defaultRequest.cpu}')"
echo "type=$G_TYPE min=$G_MIN defaultRequest=$G_DRQ default=$G_DEF max=$G_MAX"

test "$G_TYPE" = 'Container' \
  && test "$(cpu_m "$G_MIN")" -eq "$LR_MIN_CPU_M" \
  && test "$(cpu_m "$G_DEF")" -eq "$LR_DEF_CPU_M" \
  && test "$(cpu_m "$G_DRQ")" -eq "$LR_DEF_CPU_M" \
  && test "$(cpu_m "$G_MAX")" -eq "$LR_MAX_CPU_M" \
  && echo 'PASS: LimitRange luu dung bon nhom gia tri, pham vi type Container'
```

**Ý nghĩa:** `type: Container` nghĩa là ràng buộc áp cho **từng container**, không phải cho Pod.
Một Pod ba container thì mỗi container phải tự lọt khoảng `min`–`max`. Đây là chỗ dễ nhầm nhất
giữa LimitRange và ResourceQuota, và B5.4 sẽ dùng đúng khác biệt đó.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B2.3. Pod trần được tạo — và bị chèn giá trị vào

```bash
kubectl apply -f ~/lab-work/7b/b2-tran.yaml -n lab-7b
kubectl wait --for=condition=Ready pod/tran -n lab-7b --timeout=120s

R_CPU="$(kubectl get pod tran -n lab-7b -o jsonpath='{.spec.containers[0].resources.requests.cpu}')"
L_CPU="$(kubectl get pod tran -n lab-7b -o jsonpath='{.spec.containers[0].resources.limits.cpu}')"
R_MEM="$(kubectl get pod tran -n lab-7b -o jsonpath='{.spec.containers[0].resources.requests.memory}')"
L_MEM="$(kubectl get pod tran -n lab-7b -o jsonpath='{.spec.containers[0].resources.limits.memory}')"
echo "requests: cpu=$R_CPU memory=$R_MEM | limits: cpu=$L_CPU memory=$L_MEM"
```

```bash
grep -q 'resources' ~/lab-work/7b/b2-tran.yaml \
  && echo 'FAIL: manifest goc co truong resources' \
  || echo 'PASS: manifest goc khong he co truong resources'

test "$(cpu_m  "$R_CPU")" -eq "$LR_DEF_CPU_M" \
  && test "$(cpu_m  "$L_CPU")" -eq "$LR_DEF_CPU_M" \
  && test "$(mem_mi "$R_MEM")" -eq "$LR_DEF_MEM_MI" \
  && test "$(mem_mi "$L_MEM")" -eq "$LR_DEF_MEM_MI" \
  && echo 'PASS: admission controller LimitRange da chen defaultRequest va default vao container'
```

**Ý nghĩa:** file trên đĩa không đổi, object trong etcd thì có. Bài
[133](../133-limit-range-vi.md) mô tả đúng trình tự: *trước hết* admission controller LimitRange
áp giá trị request và limit mặc định cho mọi Pod chưa đặt yêu cầu tài nguyên tính toán, *sau đó*
mới theo dõi mức sử dụng so với min/max. Việc chèn xảy ra **ở giai đoạn admission, trước khi Pod
được lưu** — nên Pod cuối cùng chạy với giá trị đã bị chèn chứ không phải với ô trống.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B2.4. Vượt `max` bị từ chối ngay lúc tạo

```bash
LR_OVER_CPU_M=$(( LR_MAX_CPU_M * 2 ))
echo "se xin limit cpu = ${LR_OVER_CPU_M}m trong khi max = ${LR_MAX_CPU_M}m"

cat > ~/lab-work/7b/b2-qua-max.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: qua-max
  namespace: lab-7b
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${LR_DEF_CPU_M}m"
      limits:
        cpu: "${LR_OVER_CPU_M}m"
EOF

kubectl apply -f ~/lab-work/7b/b2-qua-max.yaml 2>&1 \
  | tee ~/lab-evidence/7b/b2-qua-max.txt
```

```bash
grep -qi 'forbidden' ~/lab-evidence/7b/b2-qua-max.txt \
  && echo 'PASS: API server tra ve Forbidden ngay luc tao'
grep -q 'maximum cpu usage per Container' ~/lab-evidence/7b/b2-qua-max.txt \
  && echo 'PASS: thong bao chi dung rang buoc max bi vi pham'
test -z "$(kubectl get pod qua-max -n lab-7b --ignore-not-found -o name)" \
  && echo 'PASS: khong co Pod nao duoc tao'
```

**Ý nghĩa:** đọc kỹ câu thông báo — nó nói rõ **ràng buộc nào**, **ngưỡng bao nhiêu** và **giá trị
nào của bạn** vi phạm. Bài [133](../133-limit-range-vi.md) mô tả chính xác hành vi này: yêu cầu
gửi tới API server thất bại với mã `403 Forbidden` kèm thông báo giải thích ràng buộc đã bị vi
phạm. Cụm `per Container` trong câu đó là bằng chứng ràng buộc áp cho **từng container**.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B2.5. Dưới `min` cũng bị từ chối — nhưng bằng một câu khác

```bash
LR_UNDER_CPU_M=$(( LR_MIN_CPU_M / 2 ))
test "$LR_UNDER_CPU_M" -gt 0 \
  && echo "PASS: nguong thu duoi min la ${LR_UNDER_CPU_M}m, van lon hon 0"

cat > ~/lab-work/7b/b2-duoi-min.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: duoi-min
  namespace: lab-7b
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${LR_UNDER_CPU_M}m"
      limits:
        cpu: "${LR_UNDER_CPU_M}m"
EOF

kubectl apply -f ~/lab-work/7b/b2-duoi-min.yaml 2>&1 \
  | tee ~/lab-evidence/7b/b2-duoi-min.txt
```

```bash
grep -q 'minimum cpu usage per Container' ~/lab-evidence/7b/b2-duoi-min.txt \
  && echo 'PASS: thong bao chi dung rang buoc min bi vi pham'
test -z "$(kubectl get pod duoi-min -n lab-7b --ignore-not-found -o name)" \
  && echo 'PASS: Pod duoi min khong duoc tao'
diff -q ~/lab-evidence/7b/b2-qua-max.txt ~/lab-evidence/7b/b2-duoi-min.txt >/dev/null \
  && echo 'FAIL: hai thong bao giong nhau' \
  || echo 'PASS: hai vi pham khac nhau cho hai thong bao khac nhau'
```

**Ý nghĩa:** `min` không phải để "bắt người dùng xin nhiều hơn" mà để chặn những Pod xin quá ít
rồi bị throttle tới mức vô dụng, hoặc để giữ mật độ Pod trên node ở mức quản trị được. Đây vẫn là
ràng buộc **cho từng container**, cùng họ với `max`.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B2.6. Cái bẫy: LimitRange hợp lệ vẫn đẻ ra Pod tự mâu thuẫn

Bài [133](../133-limit-range-vi.md) có nguyên một mục cho chuyện này: LimitRange **không** kiểm tra
tính nhất quán của các giá trị mặc định mà nó áp. Dựng lại đúng tình huống đó bằng số của lab —
một request nằm **giữa** `default` và `max`, và Pod **không khai limit**:

```bash
LR_TRAP_CPU_M=$(( MIN_W_CPU_M / 5 ))
echo "request=${LR_TRAP_CPU_M}m ; default limit=${LR_DEF_CPU_M}m ; max=${LR_MAX_CPU_M}m"
test "$LR_TRAP_CPU_M" -gt "$LR_DEF_CPU_M" && test "$LR_TRAP_CPU_M" -lt "$LR_MAX_CPU_M" \
  && echo 'PASS: request nam giua default va max, tuc LimitRange se khong chan no'

cat > ~/lab-work/7b/b2-bay.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: bay
  namespace: lab-7b
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${LR_TRAP_CPU_M}m"
EOF

kubectl apply -f ~/lab-work/7b/b2-bay.yaml 2>&1 | tee ~/lab-evidence/7b/b2-bay.txt
```

```bash
grep -q 'must be less than or equal to cpu limit' ~/lab-evidence/7b/b2-bay.txt \
  && echo 'PASS: Pod bi tu choi vi request lon hon limit ma LimitRange vua chen vao'
grep -qi 'forbidden' ~/lab-evidence/7b/b2-bay.txt \
  && echo 'FAIL: day la loi Forbidden cua LimitRange' \
  || echo 'PASS: day KHONG phai loi Forbidden — no la loi tinh hop le cua object'
test -z "$(kubectl get pod bay -n lab-7b --ignore-not-found -o name)" \
  && echo 'PASS: khong co Pod bay nao ton tai'
```

**Ý nghĩa:** hai gate đầu là cả bài học. LimitRange **không vi phạm gì cả** — nó chèn `default`
limit vào đúng như được bảo. Nhưng giá trị chèn vào nhỏ hơn `requests` mà client gửi lên, và một
Pod có request lớn hơn limit là object **không hợp lệ**. Vì thế thông báo lần này không phải
`Forbidden` của admission mà là lỗi validate object — đúng câu mà bài
[133](../133-limit-range-vi.md) in ra trong ví dụ của nó.

Cách sửa **từ phía Pod**, không đụng LimitRange: khai cả `requests` lẫn `limits`.

```bash
cat > ~/lab-work/7b/b2-bay-sua.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: bay-sua
  namespace: lab-7b
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${LR_TRAP_CPU_M}m"
      limits:
        cpu: "${LR_TRAP_CPU_M}m"
EOF

kubectl apply -f ~/lab-work/7b/b2-bay-sua.yaml
kubectl wait --for=condition=Ready pod/bay-sua -n lab-7b --timeout=120s

S_REQ="$(kubectl get pod bay-sua -n lab-7b -o jsonpath='{.spec.containers[0].resources.requests.cpu}')"
S_LIM="$(kubectl get pod bay-sua -n lab-7b -o jsonpath='{.spec.containers[0].resources.limits.cpu}')"
echo "bay-sua: requests.cpu=$S_REQ limits.cpu=$S_LIM"
test "$(cpu_m "$S_REQ")" -eq "$LR_TRAP_CPU_M" && test "$(cpu_m "$S_LIM")" -eq "$LR_TRAP_CPU_M" \
  && echo 'PASS: khai ca hai thi khong con cho trong de gia tri mac dinh chen vao'
```

**PASS:** năm dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B2.7. Sửa LimitRange không đụng tới Pod đang chạy

Hạ `max.cpu` xuống **dưới cả giá trị mặc định** — tức từ giờ ngay cả Pod trần cũng không qua được:

```bash
LR_TIGHT_CPU_M=$(( LR_DEF_CPU_M / 2 ))
kubectl patch limitrange lab-7b-limits -n lab-7b --type=json \
  -p="[{\"op\":\"replace\",\"path\":\"/spec/limits/0/max/cpu\",\"value\":\"${LR_TIGHT_CPU_M}m\"}]"

kubectl get limitrange lab-7b-limits -n lab-7b -o jsonpath='{.spec.limits[0].max.cpu}'; echo
```

```bash
T_PHASE="$(kubectl get pod tran -n lab-7b -o jsonpath='{.status.phase}')"
T_REQ="$(  kubectl get pod tran -n lab-7b -o jsonpath='{.spec.containers[0].resources.requests.cpu}')"
T_RST="$(  kubectl get pod tran -n lab-7b -o jsonpath='{.status.containerStatuses[0].restartCount}')"
echo "Pod tran: phase=$T_PHASE requests.cpu=$T_REQ restarts=$T_RST"

test "$T_PHASE" = 'Running' && test "$(cpu_m "$T_REQ")" -eq "$LR_DEF_CPU_M" \
  && test "$T_RST" -eq 0 \
  && echo 'PASS: Pod dang chay khong bi tu choi, khong bi sua, khong bi khoi dong lai'
```

```bash
sed 's/name: tran/name: tran-2/' ~/lab-work/7b/b2-tran.yaml > ~/lab-work/7b/b2-tran-2.yaml
kubectl apply -f ~/lab-work/7b/b2-tran-2.yaml -n lab-7b 2>&1 \
  | tee ~/lab-evidence/7b/b2-tran-2.txt

grep -q 'maximum cpu usage per Container' ~/lab-evidence/7b/b2-tran-2.txt \
  && echo 'PASS: Pod moi bi chan boi chinh gia tri mac dinh ma LimitRange tu chen vao'
test -z "$(kubectl get pod tran-2 -n lab-7b --ignore-not-found -o name)" \
  && echo 'PASS: tran-2 khong duoc tao'
```

**Ý nghĩa:** hai kết quả trái ngược nhau trên cùng một namespace, cùng một thời điểm. Bài
[133](../133-limit-range-vi.md) giải thích: việc kiểm tra LimitRange **chỉ diễn ra ở giai đoạn
admission của Pod**, không áp dụng trên Pod đang chạy — thêm hoặc sửa LimitRange thì Pod đã tồn
tại từ trước vẫn tiếp tục chạy không thay đổi. Và đây cũng là lần thứ hai bạn gặp việc LimitRange
không tự kiểm tra giá trị mặc định của chính nó: `default` `${LR_DEF_CPU_M}m` giờ đã lớn hơn `max`
`${LR_TIGHT_CPU_M}m`, khiến **mọi** Pod trần đều chết ở admission.

Trả LimitRange về nguyên trạng trước khi đi tiếp:

```bash
kubectl apply -f ~/lab-work/7b/b2-limitrange.yaml
BACK="$(kubectl get limitrange lab-7b-limits -n lab-7b -o jsonpath='{.spec.limits[0].max.cpu}')"
test "$(cpu_m "$BACK")" -eq "$LR_MAX_CPU_M" && echo 'PASS: max.cpu da tro ve gia tri ban dau'
```

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B2.8. Gỡ LimitRange để B3 chứng minh quota một mình

```bash
kubectl delete pod --all -n lab-7b --wait=true --timeout=180s
kubectl delete limitrange lab-7b-limits -n lab-7b

POD_LEFT="$(kubectl get pods -n lab-7b --no-headers 2>/dev/null | wc -l)"
LR_LEFT="$( kubectl get limitrange -n lab-7b --no-headers 2>/dev/null | wc -l)"
echo "pod con lai=$POD_LEFT | limitrange con lai=$LR_LEFT"
test "$POD_LEFT" -eq 0 && test "$LR_LEFT" -eq 0 \
  && echo 'PASS: lab-7b tro lai trang, san sang cho phan quota'
```

**Ý nghĩa:** B3 phải chạy trên một namespace **chỉ có quota**, vì mục tiêu của nó là chứng minh
quota một mình bắt người dùng phải khai tài nguyên. B5 sẽ đưa LimitRange quay lại — và đó chính là
điểm nối của cả nhóm bài.

**PASS:** dòng `PASS:` của bước này xuất hiện. File manifest trong `~/lab-work/7b/` **giữ lại**;
B3 và B5 dùng lại `b2-tran.yaml` và `b2-limitrange.yaml`.

---

## B3. ResourceQuota — trần tổng cho tài nguyên tính toán

**Mục đích:** chuyển từ trần *của từng object* sang trần *của cả namespace*, và tách bạch **hai
kiểu từ chối hoàn toàn khác nhau** mà bài [134](../134-resource-quotas-vi.md) mô tả.

### B3.1. Đặt trần và đọc `used` so với `hard`

```bash
cat > ~/lab-work/7b/b3-quota.yaml <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab-7b-compute
  namespace: lab-7b
spec:
  hard:
    requests.cpu: "${Q_REQ_CPU_M}m"
    requests.memory: "${Q_REQ_MEM_MI}Mi"
    limits.cpu: "${Q_LIM_CPU_M}m"
    limits.memory: "${Q_LIM_MEM_MI}Mi"
EOF

kubectl apply -f ~/lab-work/7b/b3-quota.yaml
kubectl describe quota lab-7b-compute -n lab-7b | tee ~/lab-evidence/7b/b3-quota-truoc.txt
```

```bash
H_CPU="$(kubectl get resourcequota lab-7b-compute -n lab-7b -o jsonpath='{.status.hard.requests\.cpu}')"
U_CPU="$(kubectl get resourcequota lab-7b-compute -n lab-7b -o jsonpath='{.status.used.requests\.cpu}')"
H_MEM="$(kubectl get resourcequota lab-7b-compute -n lab-7b -o jsonpath='{.status.hard.requests\.memory}')"
U_MEM="$(kubectl get resourcequota lab-7b-compute -n lab-7b -o jsonpath='{.status.used.requests\.memory}')"
echo "requests.cpu used=$U_CPU hard=$H_CPU | requests.memory used=$U_MEM hard=$H_MEM"

test "$(cpu_m  "$H_CPU")" -eq "$Q_REQ_CPU_M" \
  && test "$(mem_mi "$H_MEM")" -eq "$Q_REQ_MEM_MI" \
  && echo 'PASS: hard trong status khop dung tran vua dat'
test "$(cpu_m "$U_CPU")" -eq 0 && test "$(mem_mi "$U_MEM")" -eq 0 \
  && echo 'PASS: used bang 0 vi namespace chua co Pod nao'
```

**Ý nghĩa:** `status` của ResourceQuota là chỗ duy nhất trả lời được câu "namespace này còn bao
nhiêu". `hard` là thứ bạn khai, `used` là thứ quota controller đếm được — và mọi quyết định chấp
nhận hay từ chối ở admission đều so hai con số này. Từ giây phút object này tồn tại, hạn ngạch
**được ép buộc** trong `lab-7b`; namespace không có object nào thì không có ép buộc nào.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B3.2. Kiểu từ chối thứ nhất — Pod trần thiếu khai báo

```bash
kubectl apply -f ~/lab-work/7b/b2-tran.yaml -n lab-7b 2>&1 \
  | tee ~/lab-evidence/7b/b3-tran-bi-tu-choi.txt
```

```bash
grep -qi 'forbidden' ~/lab-evidence/7b/b3-tran-bi-tu-choi.txt \
  && echo 'PASS: Pod tran bi tu choi trong namespace co quota cpu/memory'
grep -q 'lab-7b-compute' ~/lab-evidence/7b/b3-tran-bi-tu-choi.txt \
  && echo 'PASS: thong bao goi dich danh quota nao dang chan'
grep -q 'exceeded quota' ~/lab-evidence/7b/b3-tran-bi-tu-choi.txt \
  && echo 'FAIL: day la loi vuot tran' \
  || echo 'PASS: day KHONG phai loi vuot tran — namespace dang trong nguyen'
test -z "$(kubectl get pod tran -n lab-7b --ignore-not-found -o name)" \
  && echo 'PASS: khong co Pod nao duoc tao'
```

**Ý nghĩa:** đọc kỹ nội dung thông báo: nó **liệt kê đúng những trường còn thiếu**. Namespace
đang trống trơn, quota còn nguyên vẹn, mà Pod vẫn bị chặn — vì bài
[134](../134-resource-quotas-vi.md) nói rõ: khi hạn ngạch bật cho `cpu` hoặc `memory`, bạn và mọi
client khác **phải** chỉ định `requests` hoặc `limits` cho tài nguyên đó với **mọi Pod mới**, nếu
không control plane có thể từ chối admission. Đây là kiểu từ chối thứ nhất — *thiếu khai báo*. Ghi
nhớ nó, vì B3.4 sẽ cho bạn kiểu thứ hai và hai thông báo trông khác hẳn nhau.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B3.3. Namespace đối chứng: cùng manifest, không quota, chạy ngon

```bash
kubectl apply -f ~/lab-work/7b/b2-tran.yaml -n lab-7b-free
kubectl wait --for=condition=Ready pod/tran -n lab-7b-free --timeout=120s

F_PHASE="$(kubectl get pod tran -n lab-7b-free -o jsonpath='{.status.phase}')"
F_RES="$(  kubectl get pod tran -n lab-7b-free -o jsonpath='{.spec.containers[0].resources}')"
echo "lab-7b-free/tran: phase=$F_PHASE resources=${F_RES:-<rong>}"

test "$F_PHASE" = 'Running' && test "$F_RES" = '{}' \
  && echo 'PASS: han ngach chi duoc ep buoc trong namespace thuc su co doi tuong ResourceQuota'
```

**Ý nghĩa:** cùng một byte manifest, hai kết quả trái ngược. Đây là bằng chứng cho câu của bài
[134](../134-resource-quotas-vi.md): *một hạn ngạch tài nguyên được ép buộc trong một namespace cụ
thể khi có một ResourceQuota trong namespace đó*. Không có object thì không có chính sách; ranh
giới đúng bằng ranh giới namespace.

**PASS:** dòng `PASS:` của bước này xuất hiện. **Giữ Pod này lại tới B9** — nó là bằng chứng đứng
sẵn suốt phần còn lại của lab rằng một namespace không có chính sách thì Pod trần sống bình thường.

### B3.4. Kiểu từ chối thứ hai — đầy trần

```bash
POD_CPU_M=$(( Q_REQ_CPU_M / 2 ));   POD_LIM_CPU_M=$(( POD_CPU_M * 2 ))
POD_MEM_MI=$(( Q_REQ_MEM_MI / 2 )); POD_LIM_MEM_MI=$(( POD_MEM_MI * 2 ))
echo "moi Pod xin: requests ${POD_CPU_M}m/${POD_MEM_MI}Mi, limits ${POD_LIM_CPU_M}m/${POD_LIM_MEM_MI}Mi"

for i in 1 2 3; do
  cat > ~/lab-work/7b/b3-khai-$i.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: khai-$i
  namespace: lab-7b
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${POD_CPU_M}m"
        memory: "${POD_MEM_MI}Mi"
      limits:
        cpu: "${POD_LIM_CPU_M}m"
        memory: "${POD_LIM_MEM_MI}Mi"
EOF
done

kubectl apply -f ~/lab-work/7b/b3-khai-1.yaml
kubectl apply -f ~/lab-work/7b/b3-khai-2.yaml
kubectl wait --for=condition=Ready pod/khai-1 pod/khai-2 -n lab-7b --timeout=180s
```

```bash
for i in $(seq 1 24); do
  U_CPU="$(kubectl get resourcequota lab-7b-compute -n lab-7b -o jsonpath='{.status.used.requests\.cpu}')"
  test "$(cpu_m "$U_CPU")" -eq $(( POD_CPU_M * 2 )) && break
  sleep 5
done
kubectl describe quota lab-7b-compute -n lab-7b | tee ~/lab-evidence/7b/b3-quota-day.txt

test "$(cpu_m "$U_CPU")" -eq $(( POD_CPU_M * 2 )) \
  && echo 'PASS: used cong don dung tong requests cua hai Pod'
test $(( Q_REQ_CPU_M - POD_CPU_M * 2 )) -lt "$POD_CPU_M" \
  && echo 'PASS: cho con lai khong du cho Pod thu ba'
```

```bash
kubectl apply -f ~/lab-work/7b/b3-khai-3.yaml 2>&1 | tee ~/lab-evidence/7b/b3-khai-3.txt

grep -q 'exceeded quota' ~/lab-evidence/7b/b3-khai-3.txt \
  && echo 'PASS: Pod thu ba bi tu choi vi vuot tran tong'
grep -q 'requested:' ~/lab-evidence/7b/b3-khai-3.txt \
  && grep -q 'limited:' ~/lab-evidence/7b/b3-khai-3.txt \
  && echo 'PASS: thong bao noi ro xin bao nhieu, dang dung bao nhieu, tran bao nhieu'
diff -q ~/lab-evidence/7b/b3-tran-bi-tu-choi.txt ~/lab-evidence/7b/b3-khai-3.txt >/dev/null \
  && echo 'FAIL: hai kieu tu choi cho cung mot thong bao' \
  || echo 'PASS: hai kieu tu choi cho hai thong bao khac han nhau'
```

**Ý nghĩa:** so hai file `b3-tran-bi-tu-choi.txt` và `b3-khai-3.txt` cạnh nhau. Cả hai đều là
`403 Forbidden`, cả hai đều do ResourceQuota, nhưng nguyên nhân khác hẳn: cái đầu là **Pod không
khai đủ trường**, cái sau là **namespace hết chỗ**. Nhầm hai cái này là nguồn gốc của rất nhiều
lần sửa sai chỗ — thêm node không cứu được cái thứ nhất, còn sửa manifest không cứu được cái thứ
hai.

**PASS:** năm dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B3.5. Bẫy Deployment — `apply` thành công mà Pod không sinh ra đủ

```bash
kubectl delete pod khai-1 khai-2 -n lab-7b --wait=true --timeout=180s
for i in $(seq 1 24); do
  U_CPU="$(kubectl get resourcequota lab-7b-compute -n lab-7b -o jsonpath='{.status.used.requests\.cpu}')"
  test "$(cpu_m "$U_CPU")" -eq 0 && break
  sleep 5
done
test "$(cpu_m "$U_CPU")" -eq 0 && echo 'PASS: used tro ve 0, quota lai trong'
```

```bash
cat > ~/lab-work/7b/b3-deploy.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: doi-quota
  namespace: lab-7b
spec:
  replicas: 5
  selector:
    matchLabels:
      app: doi-quota
  template:
    metadata:
      labels:
        app: doi-quota
    spec:
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh", "-c", "sleep 3600"]
        resources:
          requests:
            cpu: "${POD_CPU_M}m"
            memory: "${POD_MEM_MI}Mi"
          limits:
            cpu: "${POD_LIM_CPU_M}m"
            memory: "${POD_LIM_MEM_MI}Mi"
EOF

kubectl apply -f ~/lab-work/7b/b3-deploy.yaml 2>&1 | tee ~/lab-evidence/7b/b3-deploy-apply.txt

grep -q 'created' ~/lab-evidence/7b/b3-deploy-apply.txt \
  && echo 'PASS: lenh apply Deployment THANH CONG du no doi gap nhieu lan tran'
```

```bash
for i in $(seq 1 36); do
  RF="$(kubectl get deploy doi-quota -n lab-7b \
    -o jsonpath='{.status.conditions[?(@.type=="ReplicaFailure")].reason}' 2>/dev/null)"
  test "$RF" = 'FailedCreate' && break
  sleep 5
done

RS="$(kubectl get rs -n lab-7b -l app=doi-quota -o jsonpath='{.items[0].metadata.name}')"
kubectl describe rs "$RS" -n lab-7b | tee ~/lab-evidence/7b/b3-rs-describe.txt
WANT="$(kubectl get deploy doi-quota -n lab-7b -o jsonpath='{.spec.replicas}')"
GOT="$( kubectl get deploy doi-quota -n lab-7b -o jsonpath='{.status.readyReplicas}')"
echo "replicas mong muon=$WANT | dang Ready=${GOT:-0} | ReplicaFailure=$RF"

test "${GOT:-0}" -lt "$WANT" \
  && echo 'PASS: Deployment ton tai nhung khong dua du so Pod vao ton tai'
test "$RF" = 'FailedCreate' \
  && echo 'PASS: nguyen nhan doc duoc o condition ReplicaFailure cua Deployment'
grep -q 'exceeded quota' ~/lab-evidence/7b/b3-rs-describe.txt \
  && echo 'PASS: su kien cua ReplicaSet moi la cho ghi ro quota da chan'
```

**Ý nghĩa:** đây đúng là cảnh báo của bài [134](../134-resource-quotas-vi.md) — bạn thường không
tạo Pod trực tiếp, và nếu tạo một Deployment đòi nhiều tài nguyên hơn mức khả dụng thì việc tạo
Deployment **thành công**, nhưng nó không đưa được toàn bộ số Pod nó quản lý vào tồn tại. Lệnh
`apply` không hề báo lỗi vì object bị quota chặn **không phải Deployment** mà là các Pod do
ReplicaSet tạo ra. Vì thế nơi tìm nguyên nhân là `kubectl describe` trên đối tượng quản lý
workload, không phải ở dòng output của `apply`.

**PASS:** năm dòng `PASS:` của bước này xuất hiện.

```bash
kubectl delete -f ~/lab-work/7b/b3-deploy.yaml --wait=true --timeout=180s
```

### B3.6. Quota không tạo ràng buộc nào về node

```bash
cat > ~/lab-work/7b/b3-ghim-quota.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ghim-quota
  namespace: lab-7b
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${POD_CPU_M}m"
        memory: "${POD_MEM_MI}Mi"
      limits:
        cpu: "${POD_LIM_CPU_M}m"
        memory: "${POD_LIM_MEM_MI}Mi"
EOF

cat > ~/lab-work/7b/b3-ghim-free.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: ghim-free
  namespace: lab-7b-free
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker1
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/7b/b3-ghim-quota.yaml
kubectl apply -f ~/lab-work/7b/b3-ghim-free.yaml
kubectl wait --for=condition=Ready pod/ghim-quota -n lab-7b      --timeout=180s
kubectl wait --for=condition=Ready pod/ghim-free  -n lab-7b-free --timeout=180s
```

```bash
N1="$(kubectl get pod ghim-quota -n lab-7b      -o jsonpath='{.spec.nodeName}')"
N2="$(kubectl get pod ghim-free  -n lab-7b-free -o jsonpath='{.spec.nodeName}')"
echo "lab-7b/ghim-quota -> $N1 ; lab-7b-free/ghim-free -> $N2"

test "$N1" = 'lab-k8s-worker1' && test "$N2" = 'lab-k8s-worker1' \
  && echo 'PASS: Pod cua namespace co quota va cua namespace khong quota cung nam mot node'

kubectl delete pod ghim-quota -n lab-7b      --wait=true --timeout=120s
kubectl delete pod ghim-free  -n lab-7b-free --wait=true --timeout=120s
```

**Ý nghĩa:** bài [134](../134-resource-quotas-vi.md) đóng mục *Hạn ngạch và dung lượng cluster*
bằng đúng câu này: hạn ngạch phân chia tổng tài nguyên của cluster nhưng **không tạo ra ràng buộc
nào liên quan đến node** — Pod từ nhiều namespace vẫn chạy chung trên một node. Quota là bài toán
kế toán, không phải bài toán chỗ ngồi. Muốn tách Pod theo node thì cần taint/affinity của nhóm
7a, không phải quota.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B3.7. Quota là số tuyệt đối, không nhìn dung lượng cluster

```bash
HUGE_CPU_M=$(( SUM_W_CPU_M * 10 ))
echo "tong allocatable hai worker=${SUM_W_CPU_M}m ; se dat tran=${HUGE_CPU_M}m"

cat > ~/lab-work/7b/b3-quota-huge.yaml <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab-7b-free-huge
  namespace: lab-7b-free
spec:
  hard:
    requests.cpu: "${HUGE_CPU_M}m"
EOF

kubectl apply -f ~/lab-work/7b/b3-quota-huge.yaml
HH="$(kubectl get resourcequota lab-7b-free-huge -n lab-7b-free -o jsonpath='{.status.hard.requests\.cpu}')"
echo "hard.requests.cpu = $HH"

test "$(cpu_m "$HH")" -eq "$HUGE_CPU_M" \
  && echo 'PASS: API server chap nhan tran lon gap nhieu lan dung luong that cua cluster'
```

```bash
cat > ~/lab-work/7b/b3-to-qua-node.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: to-qua-node
  namespace: lab-7b-free
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${SUM_W_CPU_M}m"
EOF

kubectl apply -f ~/lab-work/7b/b3-to-qua-node.yaml 2>&1 | tee ~/lab-evidence/7b/b3-to-qua-node.txt

for i in $(seq 1 24); do
  RSN="$(kubectl get pod to-qua-node -n lab-7b-free \
    -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].reason}' 2>/dev/null)"
  test "$RSN" = 'Unschedulable' && break
  sleep 5
done
PH="$(kubectl get pod to-qua-node -n lab-7b-free -o jsonpath='{.status.phase}')"
echo "phase=$PH ; PodScheduled reason=$RSN"

test -n "$(kubectl get pod to-qua-node -n lab-7b-free --ignore-not-found -o name)" \
  && echo 'PASS: Pod QUA duoc admission cua quota — object da ton tai trong etcd'
test "$PH" = 'Pending' && test "$RSN" = 'Unschedulable' \
  && echo 'PASS: nhung scheduler khong xep duoc — hai co che khac nhau, hai thoi diem khac nhau'
```

**Ý nghĩa:** bài [134](../134-resource-quotas-vi.md) viết: các ResourceQuota **độc lập với dung
lượng cluster**, chúng được biểu diễn bằng **đơn vị tuyệt đối**, nên thêm node cũng *không* tự cho
namespace tiêu thụ nhiều hơn — và ngược lại, tổng hạn ngạch hoàn toàn có thể vượt dung lượng thật.
Bước này cho thấy hệ quả cụ thể: quota gật đầu ở **admission**, còn việc có chỗ chạy hay không là
chuyện của **scheduler** ở bước sau. Hai cửa ải, hai loại lỗi, hai chỗ phải nhìn.

Trả `lab-7b-free` về đúng vai đối chứng trước khi đi tiếp:

```bash
kubectl delete pod to-qua-node -n lab-7b-free --wait=true --timeout=120s
kubectl delete resourcequota lab-7b-free-huge -n lab-7b-free

RQF="$(kubectl get resourcequota -n lab-7b-free --no-headers 2>/dev/null | wc -l)"
test "$RQF" -eq 0 && echo 'PASS: lab-7b-free khong con chinh sach nao'
```

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

---

## B4. ResourceQuota — số lượng object và lưu trữ

**Mục đích:** chứng minh họ giới hạn thứ hai và thứ ba mà bài
[134](../134-resource-quotas-vi.md) liệt kê. Chúng chặn được thứ mà trần CPU/memory hoàn toàn
không chạm tới: một đống object bé xíu không tốn CPU nào nhưng vẫn phá được control plane.

### B4.1. Quota theo số lượng object

`lab-7b` đã có sẵn một ConfigMap do hệ thống tạo, nên trần phải tính **từ `used` thật**, không đoán:

```bash
CM_USED="$(kubectl get configmap -n lab-7b --no-headers 2>/dev/null | wc -l)"
CM_HARD=$(( CM_USED + 1 ))
echo "configmap dang co=$CM_USED ; se dat tran=$CM_HARD"

cat > ~/lab-work/7b/b4-quota-object.yaml <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab-7b-objects
  namespace: lab-7b
spec:
  hard:
    configmaps: "${CM_HARD}"
    count/deployments.apps: "1"
EOF

kubectl apply -f ~/lab-work/7b/b4-quota-object.yaml

for i in $(seq 1 24); do
  UCM="$(kubectl get resourcequota lab-7b-objects -n lab-7b -o jsonpath='{.status.used.configmaps}')"
  test -n "$UCM" && break
  sleep 2
done
kubectl describe quota lab-7b-objects -n lab-7b | tee ~/lab-evidence/7b/b4-quota-object.txt
echo "used.configmaps = $UCM"

test "$UCM" -eq "$CM_USED" \
  && echo 'PASS: quota dem duoc ca object da co truoc khi no ra doi'
grep -q 'count/deployments.apps' ~/lab-evidence/7b/b4-quota-object.txt \
  && echo 'PASS: cu phap count/<resource>.<group> hien dung trong status'
```

```bash
kubectl create configmap cm-1 -n lab-7b --from-literal=k=v
kubectl create configmap cm-2 -n lab-7b --from-literal=k=v 2>&1 \
  | tee ~/lab-evidence/7b/b4-cm-2.txt

grep -q 'exceeded quota' ~/lab-evidence/7b/b4-cm-2.txt \
  && echo 'PASS: ConfigMap thu hai bi chan du no khong ton mot chut CPU nao'
test "$(kubectl get configmap -n lab-7b --no-headers | wc -l)" -eq "$CM_HARD" \
  && echo 'PASS: so ConfigMap dung bang tran'
```

```bash
cat > ~/lab-work/7b/b4-deploy-0.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dep-1
  namespace: lab-7b
spec:
  replicas: 0
  selector:
    matchLabels:
      app: dep-1
  template:
    metadata:
      labels:
        app: dep-1
    spec:
      containers:
      - name: app
        image: busybox:1.37
        command: ["sh", "-c", "sleep 3600"]
EOF

sed 's/dep-1/dep-2/g' ~/lab-work/7b/b4-deploy-0.yaml > ~/lab-work/7b/b4-deploy-0b.yaml

kubectl apply -f ~/lab-work/7b/b4-deploy-0.yaml
kubectl apply -f ~/lab-work/7b/b4-deploy-0b.yaml 2>&1 | tee ~/lab-evidence/7b/b4-dep-2.txt

grep -q 'exceeded quota' ~/lab-evidence/7b/b4-dep-2.txt \
  && echo 'PASS: Deployment thu hai bi tu choi NGAY LUC APPLY'
test "$(kubectl get deploy -n lab-7b --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: chi mot Deployment ton tai'
```

**Ý nghĩa:** so bước này với B3.5 — cùng là Deployment, hai kết cục ngược nhau. Quota **tính toán**
không đụng tới object Deployment nên `apply` lọt; quota **số lượng object** đếm thẳng chính object
Deployment nên `apply` chết ngay. Bài [134](../134-resource-quotas-vi.md) nói rõ họ giới hạn này
tồn tại để **bảo vệ bộ lưu trữ của control plane**: quá nhiều Secret có thể khiến server và
controller không khởi động được, còn một CronJob cấu hình kém đẻ ra vô số Job thì thành từ chối
dịch vụ. Replica bằng 0 ở đây là cố ý — nó chứng minh việc chặn hoàn toàn không liên quan tới tài
nguyên tính toán.

**PASS:** sáu dòng `PASS:` của bước này xuất hiện.

### B4.2. Quota theo lưu trữ và theo StorageClass

Tên StorageClass đọc từ cluster, **không viết cứng** — nó do Lab 6a để lại ở mốc
`03-storage-ready`:

```bash
SC_N="$(kubectl get storageclass --no-headers | wc -l)"
SC_NAME="$(kubectl get storageclass -o jsonpath='{.items[0].metadata.name}')"
echo "so StorageClass=$SC_N ; ten=$SC_NAME"
test "$SC_N" -eq 1 && test -n "$SC_NAME" \
  && echo 'PASS: doc duoc dung mot StorageClass cua moc 03-storage-ready'

cat > ~/lab-work/7b/b4-quota-storage.yaml <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab-7b-storage
  namespace: lab-7b
spec:
  hard:
    persistentvolumeclaims: "1"
    requests.storage: "2Gi"
    ${SC_NAME}.storageclass.storage.k8s.io/requests.storage: "2Gi"
EOF

kubectl apply -f ~/lab-work/7b/b4-quota-storage.yaml
kubectl describe quota lab-7b-storage -n lab-7b | tee ~/lab-evidence/7b/b4-quota-storage.txt

grep -q "${SC_NAME}.storageclass.storage.k8s.io/requests.storage" ~/lab-evidence/7b/b4-quota-storage.txt \
  && echo 'PASS: quota tach rieng duoc theo tung StorageClass'
```

```bash
for i in 1 2; do
  cat > ~/lab-work/7b/b4-pvc-$i.yaml <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-$i
  namespace: lab-7b
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ${SC_NAME}
  resources:
    requests:
      storage: 1Gi
EOF
done

kubectl apply -f ~/lab-work/7b/b4-pvc-1.yaml
kubectl apply -f ~/lab-work/7b/b4-pvc-2.yaml 2>&1 | tee ~/lab-evidence/7b/b4-pvc-2.txt

for i in $(seq 1 24); do
  UPVC="$(kubectl get resourcequota lab-7b-storage -n lab-7b \
    -o jsonpath='{.status.used.persistentvolumeclaims}')"
  test "${UPVC:-0}" -eq 1 && break
  sleep 2
done
USTO="$(kubectl get resourcequota lab-7b-storage -n lab-7b -o jsonpath='{.status.used.requests\.storage}')"
echo "used.persistentvolumeclaims=$UPVC ; used.requests.storage=$USTO"

test "${UPVC:-0}" -eq 1 \
  && echo 'PASS: PVC dau tien duoc tinh vao quota du no chua bind vao PV nao'
grep -q 'exceeded quota' ~/lab-evidence/7b/b4-pvc-2.txt \
  && echo 'PASS: PVC thu hai bi tu choi'
test "$(kubectl get pvc -n lab-7b --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: chi mot PVC ton tai'
```

**Ý nghĩa:** PVC thứ nhất đang ở `Pending` vì StorageClass của lab dùng
`volumeBindingMode: WaitForFirstConsumer` — chưa có Pod tiêu thụ thì chưa có PV nào được sinh ra.
Quota vẫn tính nó. Điều đó cho thấy quota đếm **object và giá trị khai trong `spec`**, không đếm
dung lượng đĩa thật đã bị chiếm. Bài [134](../134-resource-quotas-vi.md) tách hẳn hai tên tài
nguyên cho hai chuyện: `persistentvolumeclaims` là *số lượng claim*, còn `requests.storage` là
*tổng dung lượng xin*; và cả hai còn tách được theo từng StorageClass.

**PASS:** năm dòng `PASS:` của bước này xuất hiện.

### B4.3. Dọn phần object và lưu trữ

```bash
kubectl delete pvc pvc-1 -n lab-7b --wait=true --timeout=180s
kubectl delete deploy dep-1 -n lab-7b --wait=true --timeout=180s
kubectl delete configmap cm-1 -n lab-7b
kubectl delete resourcequota lab-7b-objects lab-7b-storage -n lab-7b

PV_N="$(kubectl get pv --no-headers 2>/dev/null | wc -l)"
PVC_N="$(kubectl get pvc -n lab-7b --no-headers 2>/dev/null | wc -l)"
RQ_LEFT="$(kubectl get resourcequota -n lab-7b --no-headers 2>/dev/null | wc -l)"
echo "pv=$PV_N | pvc lab-7b=$PVC_N | resourcequota lab-7b=$RQ_LEFT"

test "$PV_N" -eq 0 && test "$PVC_N" -eq 0 && test "$RQ_LEFT" -eq 1 \
  && echo 'PASS: tang luu tru sach, chi con lai quota tinh toan cua B3'
```

**Ý nghĩa:** `RQ_LEFT` phải bằng **1**, không phải 0 — `lab-7b-compute` còn nguyên vì B5 dựa hẳn
vào nó.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B5. LimitRange gặp ResourceQuota — mối liên hệ cốt lõi của nhóm 7b

**Mục đích:** đây là phần quan trọng nhất của lab. Hai object vừa học độc lập nhau về cơ chế nhưng
gần như luôn đi thành cặp trong thực tế, và lý do nằm gọn trong ba bước dưới đây.

### B5.1. Điểm xuất phát: quota một mình bắt người dùng phải nhớ

```bash
LR_NOW="$(kubectl get limitrange -n lab-7b --no-headers 2>/dev/null | wc -l)"
RQ_NOW="$(kubectl get resourcequota -n lab-7b --no-headers 2>/dev/null | wc -l)"
POD_NOW="$(kubectl get pods -n lab-7b --no-headers 2>/dev/null | wc -l)"
echo "limitrange=$LR_NOW quota=$RQ_NOW pod=$POD_NOW"
test "$LR_NOW" -eq 0 && test "$RQ_NOW" -eq 1 && test "$POD_NOW" -eq 0 \
  && echo 'PASS: lab-7b dang co dung mot quota, khong LimitRange, khong Pod'

kubectl apply -f ~/lab-work/7b/b2-tran.yaml -n lab-7b 2>&1 \
  | tee ~/lab-evidence/7b/b5-truoc.txt
grep -qi 'forbidden' ~/lab-evidence/7b/b5-truoc.txt \
  && test -z "$(kubectl get pod tran -n lab-7b --ignore-not-found -o name)" \
  && echo 'PASS: Pod tran van bi tu choi — day la diem xuat phat'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B5.2. Thêm LimitRange — cùng manifest đó, giờ chạy được

```bash
kubectl apply -f ~/lab-work/7b/b2-limitrange.yaml
kubectl apply -f ~/lab-work/7b/b2-tran.yaml -n lab-7b
kubectl wait --for=condition=Ready pod/tran -n lab-7b --timeout=180s
```

```bash
R_CPU="$(kubectl get pod tran -n lab-7b -o jsonpath='{.spec.containers[0].resources.requests.cpu}')"
R_MEM="$(kubectl get pod tran -n lab-7b -o jsonpath='{.spec.containers[0].resources.requests.memory}')"

for i in $(seq 1 24); do
  U_CPU="$(kubectl get resourcequota lab-7b-compute -n lab-7b -o jsonpath='{.status.used.requests\.cpu}')"
  test "$(cpu_m "$U_CPU")" -eq "$LR_DEF_CPU_M" && break
  sleep 5
done
U_MEM="$(kubectl get resourcequota lab-7b-compute -n lab-7b -o jsonpath='{.status.used.requests\.memory}')"
kubectl describe quota lab-7b-compute -n lab-7b | tee ~/lab-evidence/7b/b5-quota-sau.txt
echo "Pod nhan requests cpu=$R_CPU memory=$R_MEM ; quota used cpu=$U_CPU memory=$U_MEM"

test "$(cpu_m "$R_CPU")" -eq "$LR_DEF_CPU_M" \
  && echo 'PASS: LimitRange chen defaultRequest vao dung nhu B2.3'
test "$(cpu_m  "$U_CPU")" -eq "$LR_DEF_CPU_M" \
  && test "$(mem_mi "$U_MEM")" -eq "$LR_DEF_MEM_MI" \
  && echo 'PASS: quota dem DUNG gia tri ma LimitRange vua chen vao, khong phai 0'
```

**Ý nghĩa:** gate thứ hai là kết luận của cả nhóm bài. Bạn không sửa một ký tự nào trong manifest
Pod, không nới quota, chỉ thêm một LimitRange — và Pod đi qua. Trình tự bên trong API server:
LimitRange (mutating admission) **chèn** `requests`/`limits` mặc định vào trước, ResourceQuota
(validating admission) **soi** Pod **sau đó** và cộng đúng con số vừa được chèn vào `used`. Bài
[134](../134-resource-quotas-vi.md) ghi thẳng lối thoát này trong ghi chú của nó: định nghĩa một
LimitRange để ép giá trị mặc định cho những Pod không đặt yêu cầu tài nguyên tính toán, *để người
dùng không phải nhớ tự làm việc đó*.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B5.3. Trần tổng vẫn là trần tổng

`Q_REQ_CPU_M` đúng bằng ba lần giá trị mặc định, nên Pod trần thứ tư phải chết:

```bash
for i in 2 3 4; do
  sed "s/name: tran/name: tran-$i/" ~/lab-work/7b/b2-tran.yaml > ~/lab-work/7b/b5-tran-$i.yaml
done

kubectl apply -f ~/lab-work/7b/b5-tran-2.yaml -n lab-7b
kubectl apply -f ~/lab-work/7b/b5-tran-3.yaml -n lab-7b
kubectl wait --for=condition=Ready pod/tran-2 pod/tran-3 -n lab-7b --timeout=180s

for i in $(seq 1 24); do
  U_CPU="$(kubectl get resourcequota lab-7b-compute -n lab-7b -o jsonpath='{.status.used.requests\.cpu}')"
  test "$(cpu_m "$U_CPU")" -eq "$Q_REQ_CPU_M" && break
  sleep 5
done
echo "used.requests.cpu=$U_CPU ; hard=${Q_REQ_CPU_M}m"
test "$(cpu_m "$U_CPU")" -eq "$Q_REQ_CPU_M" \
  && echo 'PASS: ba Pod tran lap day chinh xac tran tong'

kubectl apply -f ~/lab-work/7b/b5-tran-4.yaml -n lab-7b 2>&1 \
  | tee ~/lab-evidence/7b/b5-tran-4.txt
grep -q 'exceeded quota' ~/lab-evidence/7b/b5-tran-4.txt \
  && echo 'PASS: Pod thu tu bi quota chan, khong phai bi LimitRange chan'
grep -q 'maximum cpu usage per Container' ~/lab-evidence/7b/b5-tran-4.txt \
  && echo 'FAIL: LimitRange moi la thu chan' \
  || echo 'PASS: thong bao khong nhac toi rang buoc cua LimitRange'
```

**Ý nghĩa:** ba Pod hoàn toàn hợp lệ với LimitRange — mỗi cái đúng bằng giá trị mặc định, nằm gọn
giữa `min` và `max`. Chúng chết ở cửa thứ hai. Đây là câu phân biệt phải thuộc: **LimitRange là
trần của từng object, ResourceQuota là trần của tổng cả namespace**, và một Pod phải qua **cả
hai**.

**PASS:** ba dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B5.4. Hai cửa ải, hai lý do từ chối khác nhau

Giải phóng một suất rồi thử hai hướng vi phạm ngược nhau:

```bash
kubectl delete pod tran-3 -n lab-7b --wait=true --timeout=120s
for i in $(seq 1 24); do
  U_CPU="$(kubectl get resourcequota lab-7b-compute -n lab-7b -o jsonpath='{.status.used.requests\.cpu}')"
  test "$(cpu_m "$U_CPU")" -eq $(( LR_DEF_CPU_M * 2 )) && break
  sleep 5
done
echo "cho con lai = $(( Q_REQ_CPU_M - LR_DEF_CPU_M * 2 ))m ; max cua LimitRange = ${LR_MAX_CPU_M}m"
```

```bash
cat > ~/lab-work/7b/b5-vi-pham-max.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: vi-pham-max
  namespace: lab-7b
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${LR_MIN_CPU_M}m"
      limits:
        cpu: "$(( LR_MAX_CPU_M * 2 ))m"
EOF

cat > ~/lab-work/7b/b5-vi-pham-quota.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: vi-pham-quota
  namespace: lab-7b
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "${LR_MAX_CPU_M}m"
      limits:
        cpu: "${LR_MAX_CPU_M}m"
EOF

kubectl apply -f ~/lab-work/7b/b5-vi-pham-max.yaml 2>&1   | tee ~/lab-evidence/7b/b5-vi-pham-max.txt
kubectl apply -f ~/lab-work/7b/b5-vi-pham-quota.yaml 2>&1 | tee ~/lab-evidence/7b/b5-vi-pham-quota.txt
```

```bash
test "$LR_MAX_CPU_M" -gt $(( Q_REQ_CPU_M - LR_DEF_CPU_M * 2 )) \
  && echo 'PASS: request cua vi-pham-quota nam trong max nhung vuot cho con lai'

grep -q 'maximum cpu usage per Container' ~/lab-evidence/7b/b5-vi-pham-max.txt \
  && ! grep -q 'exceeded quota' ~/lab-evidence/7b/b5-vi-pham-max.txt \
  && echo 'PASS: vi-pham-max chet o cua LimitRange du quota con cho'
grep -q 'exceeded quota' ~/lab-evidence/7b/b5-vi-pham-quota.txt \
  && ! grep -q 'maximum cpu usage per Container' ~/lab-evidence/7b/b5-vi-pham-quota.txt \
  && echo 'PASS: vi-pham-quota chet o cua ResourceQuota du LimitRange cho qua'
test -z "$(kubectl get pod vi-pham-max vi-pham-quota -n lab-7b --ignore-not-found -o name)" \
  && echo 'PASS: khong Pod nao trong hai duoc tao'
```

**Ý nghĩa:** hai Pod, hai đường chết khác nhau, và mỗi thông báo chỉ đúng vào cơ chế đã chặn nó.
Khi vận hành thật, đọc được câu thông báo là biết ngay phải sửa ở đâu: `maximum ... per Container`
thì sửa manifest của người dùng, `exceeded quota` thì hoặc dọn bớt Pod cũ hoặc xin quản trị viên
nới trần.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B5.5. Dọn phần workload của B5

```bash
kubectl delete pod --all -n lab-7b --wait=true --timeout=180s
for i in $(seq 1 24); do
  U_CPU="$(kubectl get resourcequota lab-7b-compute -n lab-7b -o jsonpath='{.status.used.requests\.cpu}')"
  test "$(cpu_m "$U_CPU")" -eq 0 && break
  sleep 5
done
echo "used.requests.cpu sau khi don = $U_CPU"
test "$(cpu_m "$U_CPU")" -eq 0 \
  && echo 'PASS: quota controller da tru lai het, used ve 0'
```

**Ý nghĩa:** `used` không phải con số cộng dồn vĩnh viễn — nó là ảnh chụp mức tiêu thụ hiện tại,
do quota controller cập nhật khi object đến và đi. Giữ nguyên quota và LimitRange trong `lab-7b`;
B9 xóa chúng cùng namespace.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B6. Giới hạn PID — chính sách nằm trên kubelet

**Mục đích:** đổi mặt phẳng. Từ đây tới hết phần B, chính sách không còn là object trong etcd nữa.
Bài [135](../135-pid-limiting-vi.md) là ví dụ rõ nhất: PID là *người anh em* của `requests`/`limits`
nhưng **chỉ định theo một cách khác hẳn**.

### B6.1. `pids` không phải thứ khai được trong `.spec` của Pod

Chạy trong `lab-7b-free` để không có chính sách nào khác chen vào:

```bash
cat > ~/lab-work/7b/b6-pids.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pid-thu
  namespace: lab-7b-free
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      limits:
        pids: "100"
EOF

kubectl apply -f ~/lab-work/7b/b6-pids.yaml 2>&1 | tee ~/lab-evidence/7b/b6-pids.txt
```

```bash
grep -qi 'invalid' ~/lab-evidence/7b/b6-pids.txt \
  && echo 'PASS: API server tu choi vi ten tai nguyen khong hop le'
grep -q 'pids' ~/lab-evidence/7b/b6-pids.txt \
  && echo 'PASS: thong bao goi dich danh khoa pids'
test -z "$(kubectl get pod pid-thu -n lab-7b-free --ignore-not-found -o name)" \
  && echo 'PASS: khong co Pod nao duoc tao'
```

**Ý nghĩa:** đây là bằng chứng trực tiếp cho câu quan trọng nhất của bài
[135](../135-pid-limiting-vi.md): *thay vì định nghĩa giới hạn tài nguyên của Pod trong `.spec` của
Pod, bạn cấu hình giới hạn này như một thiết lập trên kubelet; giới hạn PID định nghĩa ở cấp Pod
hiện chưa được hỗ trợ*. Không có chỗ nào trong Pod spec nhận `pids`, và API server nói thẳng điều
đó chứ không im lặng bỏ qua.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B6.2. `podPidsLimit` thật của kubelet trên cả ba node

Trước hết đọc **file cấu hình** — thứ quản trị viên viết ra:

```bash
DECL=0
LINE="$(sudo grep -i 'podPidsLimit' /var/lib/kubelet/config.yaml || true)"
test -n "$LINE" && DECL=$(( DECL + 1 ))
echo "lab-k8s-master: ${LINE:-<khong khai>}"
for n in lab-k8s-worker1 lab-k8s-worker2; do
  LINE="$(ssh "$n" 'sudo grep -i podPidsLimit /var/lib/kubelet/config.yaml' 2>/dev/null || true)"
  test -n "$LINE" && DECL=$(( DECL + 1 ))
  echo "$n: ${LINE:-<khong khai>}"
done
echo "so node co khai podPidsLimit = $DECL"

test "$DECL" -eq 0 \
  && echo 'PASS: khong node nao khai podPidsLimit trong file cau hinh kubelet'
```

Rồi đọc **giá trị hiệu lực** — thứ kubelet thực sự đang chạy, gồm cả mặc định mà file không viết ra:

```bash
PPL_OK=0
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  PPL="$(tr ',' '\n' < ~/lab-evidence/7b/b0-configz-$n.json \
    | sed -n 's/.*"podPidsLimit":\(-\{0,1\}[0-9][0-9]*\).*/\1/p' | head -n 1)"
  echo "$n podPidsLimit hieu luc = ${PPL:-<khong doc duoc>}"
  test "$PPL" = '-1' && PPL_OK=$(( PPL_OK + 1 ))
done

test "$PPL_OK" -eq 3 \
  && echo 'PASS: gia tri hieu luc la -1 tren ca ba node — khong co tran PID nao cho Pod'
```

**Ý nghĩa:** hai lệnh này trả lời hai câu hỏi khác nhau. File cấu hình cho biết **admin đã quyết
gì**; `configz` cho biết **kubelet đang chạy với gì**. Cluster lab đang ở mặc định, tức chưa bật
giới hạn PID nào — và đó là dữ kiện quyết định cách đọc B6.3. Bài
[135](../135-pid-limiting-vi.md) cảnh báo ngay chỗ này: vì mỗi Node có thể có một giới hạn khác
nhau, **giới hạn thực tế áp lên một Pod phụ thuộc nơi nó được lập lịch tới**; cách dễ nhất là để
tất cả các Node dùng cùng một mức. Ở đây cả ba node cùng `-1`, nên ít nhất bạn đang ở trạng thái
đồng nhất.

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Nếu một node trả về giá trị khác `-1`, xem
[mục 4](#4-troubleshooting-của-lab-này) trước khi đọc tiếp B6.3.

### B6.3. Hệ quả ở cgroup của một Pod đang chạy

```bash
cat > ~/lab-work/7b/b6-pid-doc.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: pid-doc
  namespace: lab-7b-free
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/7b/b6-pid-doc.yaml
kubectl wait --for=condition=Ready pod/pid-doc -n lab-7b-free --timeout=180s
```

```bash
PMAX="$(kubectl exec pid-doc -n lab-7b-free -- cat /sys/fs/cgroup/pids.max)"
PCUR="$(kubectl exec pid-doc -n lab-7b-free -- cat /sys/fs/cgroup/pids.current)"
KPM="$( ssh lab-k8s-worker2 'cat /proc/sys/kernel/pid_max')"
echo "pids.max=$PMAX pids.current=$PCUR ; kernel pid_max tren worker2=$KPM"

test "$PMAX" = 'max' \
  && echo 'PASS: container khong co tran PID rieng, dung nhu podPidsLimit=-1'
test "$PCUR" -ge 1 \
  && echo 'PASS: cgroup van dem so task dang chay du khong ap tran'
test "$KPM" -gt 0 \
  && echo 'PASS: doc duoc gioi han PID cua he dieu hanh'
```

**Ý nghĩa:** ba con số này ghép lại thành bức tranh của bài [135](../135-pid-limiting-vi.md).
`pids.max` là `max` — không có trần nào cho Pod này, nên nếu nó fork loạn thì nó ăn vào **ngân
sách chung của node**, tức `pid_max` của hệ điều hành mà bạn vừa đọc. Đúng tình huống mà bài mô
tả: rất dễ chạm giới hạn số task mà chưa chạm bất kỳ giới hạn tài nguyên nào khác, và điều đó gây
mất ổn định cho host.

Lab **không** chạy fork bomb để chứng minh: không có giới hạn nào đang bật thì thí nghiệm đó chỉ
làm hỏng `lab-k8s-worker2` chứ không chứng minh được gì — xem bảng lý do ở
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B6.4. Eviction không thay thế được giới hạn cứng

```bash
PP_OK=0
for n in lab-k8s-worker1 lab-k8s-worker2; do
  C="$(kubectl get node "$n" -o jsonpath='{.status.conditions[?(@.type=="PIDPressure")].status}')"
  echo "$n PIDPressure=$C"
  test "$C" = 'False' && PP_OK=$(( PP_OK + 1 ))
done
test "$PP_OK" -eq 2 && echo 'PASS: hai worker khong chiu ap luc PID'

EH_OK=0
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  grep -qF '"pid.available"' ~/lab-evidence/7b/b0-configz-$n.json \
    || EH_OK=$(( EH_OK + 1 ))
done
tr ',' '\n' < ~/lab-evidence/7b/b0-configz-lab-k8s-worker2.json \
  | grep -i 'eviction' | tee ~/lab-evidence/7b/b6-eviction.txt

test "$EH_OK" -eq 3 \
  && echo 'PASS: khong node nao cau hinh nguong eviction pid.available'
```

**Ý nghĩa:** hai thứ này thường bị gộp làm một, nhưng bài [135](../135-pid-limiting-vi.md) tách
rất rõ. `PIDPressure` là **condition quan sát được** của Node, và tín hiệu eviction `pid.available`
— nếu bạn cấu hình nó — chỉ được **tính toán định kỳ và KHÔNG cưỡng chế giới hạn**; nếu số PID
tăng rất nhanh thì node vẫn mất ổn định dù đã đặt eviction cứng. Thứ đặt ra **giới hạn cứng** thật
sự là `podPidsLimit` (theo Pod) và phần dự trữ `pid=<number>` trong `--system-reserved` /
`--kube-reserved` (theo Node) — và cả ba đều nằm trong cấu hình kubelet, không phải trong API.

Cơ chế eviction do áp lực node đã được mổ xẻ ở
[Lab 7a B8](LAB-7A-LAP-LICH-VA-EVICTION.md#b8-eviction-do-áp-lực-node--đọc-cấu-hình-thật-không-ép-node-vào-áp-lực);
ở đây bạn chỉ dùng lại đúng một dữ kiện của nó — danh sách `evictionHard` — để trả lời câu hỏi của
bài 135: trục PID có nằm trong đó không.

Trên cluster này, cả hai lớp bảo vệ đều đang **tắt**. Đó là trạng thái mặc định của kubeadm, và
việc bật chúng thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy).

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B7. Resource manager của kubelet

**Mục đích:** đọc policy đang chạy của bốn trình quản lý mà bài [74](../74-resource-managers-vi.md)
mô tả, rồi chứng minh hệ quả của policy mặc định bằng một Pod thật. Lab **không đổi policy** —
việc đó đòi restart kubelet và xóa state file, xem bảng lý do ở
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

### B7.1. Policy đang chạy trên hai worker

```bash
: > ~/lab-evidence/7b/b7-policy.txt
OKP=0
for n in lab-k8s-worker1 lab-k8s-worker2; do
  F=~/lab-evidence/7b/b0-configz-$n.json
  CMP="$(   tr ',' '\n' < "$F" | sed -n 's/.*"cpuManagerPolicy":"\([^"]*\)".*/\1/p'      | head -n 1)"
  MMP="$(   tr ',' '\n' < "$F" | sed -n 's/.*"memoryManagerPolicy":"\([^"]*\)".*/\1/p'   | head -n 1)"
  TPOL="$(  tr ',' '\n' < "$F" | sed -n 's/.*"topologyManagerPolicy":"\([^"]*\)".*/\1/p' | head -n 1)"
  TSCOPE="$(tr ',' '\n' < "$F" | sed -n 's/.*"topologyManagerScope":"\([^"]*\)".*/\1/p'  | head -n 1)"
  echo "$n cpuManager=$CMP memoryManager=$MMP topologyManager=$TPOL scope=$TSCOPE" \
    | tee -a ~/lab-evidence/7b/b7-policy.txt
  test "$(printf '%s' "$CMP"    | tr 'A-Z' 'a-z')" = 'none' \
    && test "$(printf '%s' "$MMP"    | tr 'A-Z' 'a-z')" = 'none' \
    && test "$(printf '%s' "$TPOL"   | tr 'A-Z' 'a-z')" = 'none' \
    && test "$(printf '%s' "$TSCOPE" | tr 'A-Z' 'a-z')" = 'container' \
    && OKP=$(( OKP + 1 ))
done

echo "so worker chay dung policy mac dinh = $OKP"
test "$OKP" -eq 2 \
  && echo 'PASS: ca hai worker chay policy mac dinh cua ca ba trinh quan ly'
```

Vòng lặp cố ý **không** bị bọc trong một pipeline: `| tee` đặt sau `done` sẽ đẩy cả vòng lặp vào
subshell và `OKP` mất giá trị. Ghi evidence bằng `tee -a` bên trong thân vòng lặp thay vì vậy.

**Ý nghĩa:** bốn giá trị này là điểm khởi đầu của bài [74](../74-resource-managers-vi.md).
`cpuManagerPolicy: none` nghĩa là kubelet **bật tường minh cơ chế CPU affinity mặc định**, không
cung cấp affinity nào ngoài những gì bộ lập lịch của hệ điều hành tự làm, và giới hạn CPU được
cưỡng chế bằng **CFS quota**. `memoryManagerPolicy: None` và `topologyManagerPolicy: none` nghĩa
là không có ai sinh gợi ý NUMA và không có ai điều phối. B7.2 nhìn thẳng vào hệ quả.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B7.2. Pod `Guaranteed` xin CPU số nguyên vẫn nằm trong pool dùng chung

```bash
NCPU="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.capacity.cpu}')"
if [ "$NCPU" -eq 1 ]; then EXPECT='0'; else EXPECT="0-$(( NCPU - 1 ))"; fi
echo "worker2 co $NCPU CPU ; ky vong Cpus_allowed_list = $EXPECT"

cat > ~/lab-work/7b/b7-guaranteed.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: cpu-nguyen
  namespace: lab-7b-free
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources:
      requests:
        cpu: "1"
        memory: "64Mi"
      limits:
        cpu: "1"
        memory: "64Mi"
EOF

kubectl apply -f ~/lab-work/7b/b7-guaranteed.yaml
kubectl wait --for=condition=Ready pod/cpu-nguyen -n lab-7b-free --timeout=180s
```

```bash
QOS="$(kubectl get pod cpu-nguyen -n lab-7b-free -o jsonpath='{.status.qosClass}')"
ALLOWED="$(kubectl exec cpu-nguyen -n lab-7b-free -- awk '/Cpus_allowed_list/{print $2}' /proc/self/status)"
CPUMAX="$( kubectl exec cpu-nguyen -n lab-7b-free -- cat /sys/fs/cgroup/cpu.max)"
echo "qosClass=$QOS Cpus_allowed_list=$ALLOWED cpu.max=$CPUMAX"

test "$QOS" = 'Guaranteed' \
  && echo 'PASS: Pod dat du hai dieu kien ma bai 74 neu — Guaranteed va CPU requests so nguyen'
test "$ALLOWED" = "$EXPECT" \
  && echo 'PASS: container van thay TOAN BO CPU cua node — khong duoc cap CPU doc quyen'
test "$CPUMAX" != 'max' \
  && echo 'PASS: gioi han CPU van duoc cuong che bang CFS quota, dung nhu policy none'
```

**Ý nghĩa:** Pod này thỏa **đúng** hai điều kiện hẹp mà bài [74](../74-resource-managers-vi.md)
đặt ra cho việc được cấp CPU độc quyền — QoS `Guaranteed` và CPU `requests` là **số nguyên**. Nó
vẫn không được cấp CPU nào riêng, vì điều kiện thứ ba không thỏa: chính sách phải là `static`.
Dưới `none`, mọi container đều nằm trong **pool dùng chung** và giới hạn CPU được cưỡng chế bằng
CFS quota — hai giá trị bạn vừa đọc chính là hai mặt của câu đó.

Nếu policy là `static`, `Cpus_allowed_list` sẽ thu về đúng một CPU và `cpu.max` không còn là công
cụ cưỡng chế nữa, vì mức sử dụng đã bị giới hạn bởi chính miền lập lịch. Đừng thử đổi để xem: xem
bảng lý do ở [mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B7.3. Chính sách này không nằm trong object Node

```bash
kubectl get node lab-k8s-worker2 -o yaml > ~/lab-evidence/7b/b7-node.yaml
HIT="$(grep -ci 'cpuManagerPolicy\|memoryManagerPolicy\|topologyManagerPolicy\|podPidsLimit' \
  ~/lab-evidence/7b/b7-node.yaml || true)"
echo "so lan cac policy xuat hien trong object Node = $HIT"
test "$HIT" -eq 0 \
  && echo 'PASS: object Node khong he mang chinh sach cua kubelet'

ssh lab-k8s-worker2 'sudo ls -1 /var/lib/kubelet/' | tee ~/lab-evidence/7b/b7-kubelet-dir.txt
grep -qx 'config.yaml' ~/lab-evidence/7b/b7-kubelet-dir.txt \
  && echo 'PASS: cau hinh song trong thu muc cua kubelet tren chinh node do'
```

**Ý nghĩa:** đây là vòng khép lại của B1.2. Bạn có thể `kubectl get node` cả ngày mà không bao giờ
biết worker đang chạy policy nào — thông tin đó chỉ có trong file trên node và trong `configz` của
chính kubelet. Trong danh sách thư mục vừa in, nếu thấy `cpu_manager_state` hoặc
`memory_manager_state` thì đó chính là các **state file** mà bài [74](../74-resource-managers-vi.md)
nhắc tới: đổi policy đòi phải xóa chúng rồi restart kubelet, và đó là lý do lab này không đổi.
**Không xóa, không sửa gì trong thư mục này.**

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B7.4. Vì sao topology manager không có gì để căn chỉnh ở đây

```bash
NUMA_N="$(ssh lab-k8s-worker2 'ls -d /sys/devices/system/node/node[0-9]* 2>/dev/null | wc -l')"
echo "so NUMA node tren lab-k8s-worker2 = $NUMA_N"
test "$NUMA_N" -eq 1 \
  && echo 'PASS: worker chi co mot NUMA node, khong co ranh gioi nao de can chinh'
```

**Ý nghĩa:** Topology Manager là trình điều phối trung tâm; CPU Manager, Memory Manager và Device
Manager đều tham vấn nó. Nhưng toàn bộ giá trị của nó nằm ở việc đặt CPU và bộ nhớ về **cùng một
phía của ranh giới NUMA** — và một VM một NUMA node thì không có ranh giới nào. Đây là lý do kỹ
thuật, không phải lý do lười: bài [74](../74-resource-managers-vi.md) vốn viết cho phần cứng
bare-metal nhiều socket.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B8. Tính năng do Node khai báo

**Mục đích:** lộ trình đánh dấu bài [154](../154-node-declared-features-vi.md) là **đọc lướt**, và
lab giữ đúng mức đó: chỉ kiểm chứng phần đọc được từ object Node.

```bash
kubectl explain node.status > ~/lab-evidence/7b/b8-explain-node-status.txt 2>&1 || true

DF_HIT=0
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  C="$(kubectl get node "$n" -o json | grep -c 'declaredFeatures' || true)"
  echo "$n: so lan xuat hien declaredFeatures = $C"
  DF_HIT=$(( DF_HIT + C ))
done

test "$DF_HIT" -eq 0 \
  && echo 'PASS: khong node nao khai .status.declaredFeatures tren baseline nay'
```

**Ý nghĩa:** kubelet chỉ báo cáo vào `.status.declaredFeatures` những tính năng đang trong quá
trình phát triển tích cực, và cả cơ chế chỉ hoạt động khi feature gate được bật trên
`kube-apiserver`, `kube-scheduler` lẫn `kubelet`. Cluster này không có gì trong trường đó — đúng
như dự kiến. Bài [154](../154-node-declared-features-vi.md) cũng nói rõ đây là framework **nội bộ**,
dành cho nhà phát triển tính năng của Kubernetes, và người triển khai Pod **không cần tương tác
trực tiếp**: bạn không khai gì thêm trong Pod spec, scheduler tự suy ra tập tính năng mà Pod cần.

Mục tiêu duy nhất của lần đọc này là **nhận ra cái tên** khi sau này gặp một Pod `Pending` khó hiểu
trên cluster đang nâng cấp dở — tình huống chênh lệch phiên bản giữa các node mà cơ chế này sinh
ra để xử lý. Cluster ba máy cùng một phiên bản của bạn theo định nghĩa không có sự lệch đó.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B9. Cleanup và gate cuối

**Mục đích:** xóa mọi thứ lab tạo ra, chứng minh **cấu hình node không hề bị sửa**, và chứng minh
cluster đã về đúng `03-storage-ready`.

### B9.1. Xóa object của bài học

```bash
kubectl delete namespace lab-7b lab-7b-free --wait=true --timeout=300s

rm -f ~/lab-work/7b/b2-tran.yaml ~/lab-work/7b/b2-tran-2.yaml \
      ~/lab-work/7b/b2-limitrange.yaml ~/lab-work/7b/b2-qua-max.yaml \
      ~/lab-work/7b/b2-duoi-min.yaml ~/lab-work/7b/b2-bay.yaml \
      ~/lab-work/7b/b2-bay-sua.yaml \
      ~/lab-work/7b/b3-quota.yaml ~/lab-work/7b/b3-khai-1.yaml \
      ~/lab-work/7b/b3-khai-2.yaml ~/lab-work/7b/b3-khai-3.yaml \
      ~/lab-work/7b/b3-deploy.yaml ~/lab-work/7b/b3-ghim-quota.yaml \
      ~/lab-work/7b/b3-ghim-free.yaml ~/lab-work/7b/b3-quota-huge.yaml \
      ~/lab-work/7b/b3-to-qua-node.yaml \
      ~/lab-work/7b/b4-quota-object.yaml ~/lab-work/7b/b4-deploy-0.yaml \
      ~/lab-work/7b/b4-deploy-0b.yaml ~/lab-work/7b/b4-quota-storage.yaml \
      ~/lab-work/7b/b4-pvc-1.yaml ~/lab-work/7b/b4-pvc-2.yaml \
      ~/lab-work/7b/b5-tran-2.yaml ~/lab-work/7b/b5-tran-3.yaml \
      ~/lab-work/7b/b5-tran-4.yaml ~/lab-work/7b/b5-vi-pham-max.yaml \
      ~/lab-work/7b/b5-vi-pham-quota.yaml \
      ~/lab-work/7b/b6-pids.yaml ~/lab-work/7b/b6-pid-doc.yaml \
      ~/lab-work/7b/b7-guaranteed.yaml
rmdir ~/lab-work/7b
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến điều
đó thành gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/7b/` **giữ lại** — đó là bằng chứng.

```bash
NS_LEFT="$(kubectl get namespace lab-7b lab-7b-free --ignore-not-found -o name 2>/dev/null | wc -l)"
RQ_LEFT="$(kubectl get resourcequota -A --no-headers 2>/dev/null | wc -l)"
LR_LEFT="$(kubectl get limitrange -A --no-headers 2>/dev/null | wc -l)"
echo "namespace cua lab con=$NS_LEFT | resourcequota=$RQ_LEFT | limitrange=$LR_LEFT"

test "$NS_LEFT" -eq 0 && echo 'PASS: hai namespace cua lab da bien mat'
test "$RQ_LEFT" -eq 0 && test "$LR_LEFT" -eq 0 \
  && echo 'PASS: khong con ResourceQuota hay LimitRange nao trong cluster'
test ! -e ~/lab-work/7b && echo 'PASS: manifest tam da xoa'
```

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B9.2. Gate quan trọng nhất của lab: cấu hình node không đổi

```bash
{
  echo "lab-k8s-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$n $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
} | tee ~/lab-evidence/7b/b9-kubelet-sha.txt

diff -u ~/lab-evidence/7b/b0-kubelet-sha.txt ~/lab-evidence/7b/b9-kubelet-sha.txt \
  && echo 'PASS: cau hinh kubelet cua ca ba node khong he doi trong suot lab' \
  || echo 'FAIL: cau hinh kubelet da bi sua — xem muc 4'

RS_OK=0
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  A="$(kubectl get node "$n" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
  test "$A" = 'True' && RS_OK=$(( RS_OK + 1 ))
done
test "$RS_OK" -eq 3 && echo 'PASS: ca ba kubelet van Ready sau khi lab ket thuc'
```

**Ý nghĩa:** B6 và B7 đọc rất nhiều thứ trên node, và cám dỗ "sửa thử một dòng để xem policy
static chạy thế nào" là có thật. Gate này biến lời hứa "chỉ đọc" thành thứ kiểm chứng được. Nếu
`diff` báo khác, cluster của bạn đã lệch khỏi baseline: restore cả ba VM về `03-storage-ready`
trước khi sang lab sau.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B9.3. Tầng lưu trữ trở về đúng định nghĩa của mốc

```bash
PV_ALL="$(kubectl get pv --no-headers 2>/dev/null | wc -l)"
PVC_ALL="$(kubectl get pvc -A --no-headers 2>/dev/null | wc -l)"
SC_ALL="$(kubectl get sc --no-headers | wc -l)"
SC_NOW="$(kubectl get sc -o jsonpath='{.items[0].metadata.name}')"
SC_DEF="$(kubectl get sc "$SC_NOW" \
  -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}')"
echo "pv=$PV_ALL pvc=$PVC_ALL storageclass=$SC_ALL ($SC_NOW, default=$SC_DEF)"

test "$PV_ALL" -eq 0 && test "$PVC_ALL" -eq 0 \
  && echo 'PASS: khong con PV hay PVC nao — PVC cua B4 khong de lai gi'
test "$SC_ALL" -eq 1 && test "$SC_NOW" = "$SC_NAME" && test "$SC_DEF" = 'true' \
  && echo 'PASS: van dung mot StorageClass mac dinh nhu moc 03-storage-ready quy dinh'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B9.4. Gate ngắn A5.5 và kết thúc

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
kubectl get pods -A -o wide | tee ~/lab-evidence/7b/b9-pods.txt

{
  echo "=== $(date -Is) — trang thai sau Lab 7b ==="
  kubectl get resourcequota -A 2>&1
  kubectl get limitrange -A 2>&1
  kubectl get sc -o wide
  kubectl get pv
  kubectl get namespaces
} | tee ~/lab-evidence/7b/b9-sau.txt

diff -u ~/lab-evidence/7b/b0-truoc.txt ~/lab-evidence/7b/b9-sau.txt \
  > ~/lab-evidence/7b/b9-diff.txt 2>&1 || true
```

**PASS:** ba node `Ready`; dòng `PASS: readyz ok` xuất hiện; lệnh field selector trả
`No resources found`; CoreDNS đủ replica `READY`; `default` không có Pod; danh sách namespace
không còn `lab-7b` và `lab-7b-free`.

Cluster đã trở về `03-storage-ready`. **Lab 7b không tạo snapshot mới** — để ba VM nguyên trạng
đang chạy hoặc tắt tùy bạn; lab sau tự bật máy theo
[A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab).

---

## 3. Checkpoint 7b

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Bài 132 chia việc áp chính sách thành hai mặt phẳng theo **nơi cưỡng chế**. Kể tên hai mặt
      phẳng đó, và với mỗi cái nói rõ bạn kiểm tra nó bằng lệnh gì. Vì sao `kubectl get` không bao
      giờ cho bạn biết một worker đang chạy `cpuManagerPolicy` nào?
- [ ] Namespace có đúng một LimitRange đặt `default` và `defaultRequest`. Bạn apply một Pod không
      khai `resources`. Pod có bị từ chối không? Ai điền giá trị vào, ở bước nào của vòng đời yêu
      cầu, và file manifest trên đĩa của bạn có đổi không?
- [ ] Một LimitRange hoàn toàn hợp lệ vẫn khiến một Pod không tạo được, và thông báo **không phải**
      `Forbidden`. Kể lại tình huống, giải thích cơ chế, và nêu cách sửa từ phía Pod mà không đụng
      LimitRange.
- [ ] Bạn hạ `max` của LimitRange xuống dưới mức Pod đang chạy đang dùng. Chuyện gì xảy ra với Pod
      đó, và chuyện gì xảy ra với Pod tiếp theo có cùng manifest? Một câu trả lời của bài giải
      thích cả hai — câu nào?
- [ ] Namespace `team-a` có ResourceQuota `requests.cpu` và LimitRange `max.cpu`. Mỗi cái chặn
      điều gì? Với một Pod cụ thể, làm sao đọc thông báo để biết **cái nào** đã chặn nó?
- [ ] Namespace có quota cho `memory` nhưng không có LimitRange. Bạn apply một Pod trần: bị từ
      chối kiểu gì? Rồi bạn apply một Pod khai đủ nhưng namespace đã đầy: bị từ chối kiểu gì? Hai
      thông báo khác nhau ở đâu và mỗi cái đòi bạn sửa ở chỗ nào?
- [ ] Bạn thêm LimitRange vào namespace đang có quota, không sửa gì khác. Vì sao Pod trần đột
      nhiên chạy được, và vì sao `used` của quota tăng đúng bằng giá trị mặc định chứ không phải 0?
      Trật tự hai admission controller là gì?
- [ ] Bạn `kubectl apply` một Deployment 5 replica vào namespace mà quota chỉ đủ cho 2. Lệnh apply
      báo gì? Bao nhiêu Pod tồn tại? Tìm nguyên nhân ở object nào? Câu trả lời đổi thế nào nếu
      quota là `count/deployments.apps` thay vì `requests.cpu`?
- [ ] Hạn ngạch theo **số lượng object** và theo **lưu trữ** bảo vệ được điều gì mà trần CPU/memory
      không bảo vệ được? Nêu một ví dụ bài đưa ra. Một PVC đang `Pending` chưa bind vào PV nào có
      bị tính vào quota không, và vì sao?
- [ ] Bạn thêm một worker thứ ba vào cluster. Hạn ngạch của `lab-7b` có tự nới ra không?
      ResourceQuota có buộc Pod của namespace đó chỉ chạy trên một số node nhất định không? Bạn đã
      chứng minh hai điều này bằng bước nào?
- [ ] Giới hạn PID của một Pod khai ở đâu, và chuyện gì xảy ra nếu bạn thử khai nó trong
      `.spec.containers[].resources`? Trên cluster lab, `pids.max` của một container là bao nhiêu
      và điều đó nói lên gì? Tín hiệu eviction `pid.available` có thay thế được giới hạn cứng
      không?
- [ ] Một Pod `Guaranteed` xin `cpu: "1"` chạy trên `lab-k8s-worker2`. Nó có được cấp một CPU độc
      quyền không? Bạn đọc hai giá trị nào để trả lời, và điều kiện còn thiếu là gì? Vì sao lab
      không đổi điều kiện đó để thử?

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Luồng một yêu cầu tạo Pod đi qua hai chính sách cấp namespace.** Bắt đầu từ lúc bạn gõ
   `kubectl apply` một Pod **không khai tài nguyên** vào một namespace có cả LimitRange lẫn
   ResourceQuota. Kể đủ: ai chạm vào object trước và **sửa** gì; ai chạm sau và **soi** gì; con số
   nào được cộng vào đâu; ba lý do khác nhau khiến yêu cầu đó có thể chết, kèm câu thông báo đặc
   trưng của từng lý do. Rồi kể tiếp: nếu Pod qua được cả hai cửa mà vẫn `Pending`, thứ chặn nó
   lúc này là ai và vì sao đó là một cơ chế hoàn toàn khác.
2. **Luồng chính sách không đi qua API server.** Kể: hai loại chính sách của bài 135 và 74 sống ở
   đâu, đọc chúng bằng cách nào trên một cluster đang chạy, và vì sao hai bản sao giống hệt nhau
   của cùng một Pod lại có thể chịu hai chính sách khác nhau. Sau đó chỉ ra, với mỗi loại, **điều
   gì bạn quan sát được trên cluster lab** và **điều gì thì không**, kèm lý do — không có giới hạn
   nào đang bật, chỉ có một NUMA node, và việc bật chúng đòi những thao tác nào mà lab cố ý không
   làm.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm **LimitRange** với **ResourceQuota**,
*thiếu khai báo* với *vượt trần*, quota **tính toán** với quota **số lượng object**, hay chính sách
**trong API server** với chính sách **trên kubelet** — Lab 7b và
[giai đoạn 7](../00-ALO-TRINH-ADMIN.md#giai-đoạn-7--lập-lịch-và-chính-sách-tài-nguyên) hoàn tất.

Lab này **không phát sinh nợ mới** và **không trả nợ nào**; xem
[sổ nợ lab](README.md#5-sổ-nợ-lab). Những phần cố ý không làm đều nằm trong bảng lý do ở
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành), và chúng thuộc đúng thứ tự lộ trình đã định chứ
không phải nợ.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ liệt
kê sự cố phát sinh từ nội dung bài học 7b.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| B0.4: `configz` không đọc được trên một node | `kubectl get --raw "/api/v1/nodes/<node>/proxy/healthz"`; [tầng 6 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a547-tầng-6--đường-control-plane--kubelet) | Đường control plane → kubelet đang hỏng, và nó ảnh hưởng cả `logs`/`exec`. Chạy lại tầng 6 của A5.4; đừng đọc vòng bằng cách sửa gì trên node. Nếu vẫn hỏng, restore cả ba VM về `03-storage-ready` |
| B0.2: `allocatable` in ra `0Mi` | `kubectl get node <n> -o jsonpath='{.status.allocatable.memory}'` | Đơn vị trả về không phải `Ki`/`Mi`/`Gi`. Bổ sung nhánh tương ứng vào hàm `mem_mi` **trong phiên shell**, không sửa cluster |
| B0.3: gate `min < default < max` fail | In lại `MIN_W_CPU_M` và `MIN_W_MEM_MI` | Worker quá nhỏ nên phép chia nguyên ra 0. Giảm mẫu số trong công thức của B0.3 (ví dụ `/ 20` thay `/ 50`) rồi chạy lại B0.3; các bước sau tự theo giá trị mới |
| B2.4 hoặc B2.5 không thấy câu `maximum`/`minimum ... per Container` | `cat` lại file evidence tương ứng | Nếu thông báo là `exceeded quota` thì bạn đang chạy nhầm thứ tự — B2 phải chạy khi namespace **chưa** có ResourceQuota. Xóa quota, chạy lại B2 từ B2.2 |
| B2.6 báo `forbidden` thay vì `must be less than or equal to cpu limit` | So `LR_TRAP_CPU_M` với `LR_MIN_CPU_M` và `LR_MAX_CPU_M` | Request rơi ra ngoài khoảng `min`–`max` nên LimitRange chặn trước khi lỗi hợp lệ kịp xuất hiện. Chọn lại `LR_TRAP_CPU_M` nằm hẳn giữa `default` và `max` |
| B3.2 lại thấy `exceeded quota` | `kubectl get pods -n lab-7b`; `kubectl describe quota lab-7b-compute -n lab-7b` | Namespace còn Pod sót từ B2. Xóa hết Pod, chờ `used` về 0 rồi chạy lại — kiểu từ chối *thiếu khai báo* chỉ hiện ra khi quota còn trống |
| `used` không về 0 sau khi xóa Pod | `kubectl get pods -n lab-7b`; `kubectl describe quota <tên> -n lab-7b` | Còn Pod trong grace period, hoặc quota controller chưa kịp cập nhật. Chờ hết vòng lặp trong bước tương ứng. Không xóa rồi tạo lại object quota — làm thế là mất luôn phép so `used`/`hard` mà bước sau cần |
| B3.5: Deployment lên đủ 5 replica | In `POD_CPU_M` và `Q_REQ_CPU_M`; `kubectl describe quota lab-7b-compute -n lab-7b` | Trần quá rộng so với request. Kiểm tra quota còn nguyên trong namespace, và `used` bằng 0 trước khi apply. Đừng tăng `replicas` để "ép" — sửa lại quan hệ giữa hai con số ở B0.3 |
| B4.1: `cm-1` đã bị chặn | `kubectl get configmap -n lab-7b`; `kubectl describe quota lab-7b-objects -n lab-7b` | `CM_USED` đọc sai thời điểm (bạn tạo ConfigMap trước khi đọc). Xóa quota `lab-7b-objects`, đọc lại `CM_USED`, apply lại |
| B4.2: PVC `pvc-1` kẹt `Pending` | `kubectl describe pvc pvc-1 -n lab-7b` rồi đọc Events | Nếu event là `WaitForFirstConsumer` thì **đúng như thiết kế** — bước này cố ý không tạo Pod tiêu thụ. Đi tiếp, đừng tạo Pod để "chữa" |
| B5.2: Pod chạy được nhưng `used` vẫn 0 | `kubectl get pod tran -n lab-7b -o jsonpath='{.spec.containers[0].resources}'` | Nếu `resources` rỗng thì LimitRange chưa được apply lại sau B2.8. Apply `b2-limitrange.yaml`, xóa Pod, tạo lại |
| B6.2: một node trả `podPidsLimit` khác `-1` | `ssh <node> 'sudo grep -i podPidsLimit /var/lib/kubelet/config.yaml'` | Node đó đã được cấu hình trần PID từ trước — cluster của bạn lệch baseline. **Không sửa file để "đưa về chuẩn"**; ghi giá trị đọc được vào evidence, và ở B6.3 kỳ vọng `pids.max` bằng đúng con số đó thay vì `max`. Việc đưa ba node về cùng một mức là nội dung của giai đoạn 20 |
| B6.3: container không có `/sys/fs/cgroup/pids.max` | `stat -fc %T /sys/fs/cgroup/` trên `lab-k8s-worker2` phải là `cgroup2fs` — xem [Lab 2 B2](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md#b2-cgroup-v2-và-cgroup-driver) | Nếu node đúng cgroup v2 mà container không thấy file, đọc `PidsLimit` từ node bằng `sudo crictl inspect <container-id>` và ghi vào evidence. **Không** sửa cấu hình containerd hay systemd để bật thêm controller |
| B7.1: `OKP` in ra 0 dù màn hình thấy `none` | Kiểm tra bạn có tự thêm `\| tee` sau `done` không | Bọc vòng lặp trong pipeline đẩy nó vào subshell và biến đếm không thoát ra được. Chạy lại đúng khối như trong bài, ghi evidence bằng `tee -a` bên trong thân vòng lặp |
| B7.2: `Cpus_allowed_list` khác dải đầy đủ | `kubectl get node lab-k8s-worker2 -o jsonpath='{.status.capacity.cpu}'`; đọc lại `cpuManagerPolicy` ở B7.1 | Nếu policy đã là `static` thì cluster lệch baseline — xem dòng B6.2 ở trên. Nếu policy vẫn `none` mà dải vẫn hẹp, kiểm tra Pod có `nodeSelector` đúng node không |
| B7.4: số NUMA node khác 1 | `ssh lab-k8s-worker2 'lscpu \| grep -i numa'` | VM được cấu hình nhiều socket. Ghi con số thật vào evidence và đọc lại phần *Ý nghĩa* theo topology của bạn; không đổi cấu hình VM giữa chừng lab |
| B9.1: namespace kẹt `Terminating` | `kubectl get pods,pvc -n lab-7b`; `kubectl get pv` | Thường là PVC của B4 chưa được dọn xong hoặc Pod còn trong grace period. Chờ; **không** cưỡng chế finalizer của Namespace |
| B9.2: `diff` báo checksum khác | `diff -u ~/lab-evidence/7b/b0-kubelet-sha.txt ~/lab-evidence/7b/b9-kubelet-sha.txt` | Cấu hình kubelet đã bị sửa trong lúc chạy lab. Cluster lệch baseline: tắt cả ba VM, restore cả ba về `03-storage-ready`, và **không** sang lab sau trước khi gate này PASS |
| B9.3: còn PV sót lại | `kubectl get pv -o wide`; log của provisioner | PVC của B4 chưa được thu hồi hết. Chờ provisioner dọn; nếu PV kẹt `Terminating`, xử lý theo [mục 4 của Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md#4-troubleshooting-của-lab-này) — mốc `03-storage-ready` được định nghĩa là **không có PV nào** |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Policies](https://v1-35.docs.kubernetes.io/docs/concepts/policy/)
- [Kubernetes v1.35 — Limit Ranges](https://v1-35.docs.kubernetes.io/docs/concepts/policy/limit-range/)
- [Kubernetes v1.35 — Resource Quotas](https://v1-35.docs.kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes v1.35 — Process ID Limits And Reservations](https://v1-35.docs.kubernetes.io/docs/concepts/policy/pid-limiting/)
- [Kubernetes v1.35 — Resource Managers](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/resource-managers/)
- [Kubernetes — Node Declared Features](https://kubernetes.io/docs/concepts/scheduling-eviction/node-declared-features/)
  — trang này mô tả tính năng ra sau baseline của Lab 00, nên dùng link không gắn version
- [Kubernetes v1.35 — ResourceQuota API reference](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/policy-resources/resource-quota-v1/)
- [Kubernetes v1.35 — LimitRange API reference](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/policy-resources/limit-range-v1/)
- [Kubernetes v1.35 — Kubelet Configuration (v1beta1)](https://v1-35.docs.kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [Kubernetes v1.35 — Set Kubelet Parameters Via A Configuration File](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)
