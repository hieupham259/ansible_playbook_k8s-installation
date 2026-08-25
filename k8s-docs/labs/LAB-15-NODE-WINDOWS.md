# Lab 15 — Node Windows

> **Điểm bắt đầu:** snapshot `04-metrics-ready` — mốc do
> [Lab 11a](LAB-11A-OBSERVABILITY.md) tạo, xem [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc — phụ thuộc nhánh, [B1](#b1-kiểm-tra-năng-lực-node-windows--chọn-nhánh) quyết định:**
> **nhánh A** (cluster có node Windows join được) **tạo** mốc mới `15-windows-ready`;
> **nhánh B** (không có VM Windows Server — trường hợp mặc định của cluster lab) cleanup **trả cluster
> về đúng `04-metrics-ready`**, không tạo mốc mới.
> **Lab tùy chọn.** [Lộ trình](../00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows)
> ghi thẳng: *bỏ qua hoàn toàn nếu cluster chỉ có Linux*. Nếu môi trường của bạn chỉ có Linux thì bỏ
> hẳn phần dựng node Windows là đúng — nhưng **phần đọc–hiểu vẫn đáng làm**, vì nó dạy ranh giới của
> chính mô hình Kubernetes: chỗ nào là API chung cho mọi node, chỗ nào là hệ điều hành lộ ra qua API.
> **Lab trước:** [Lab 14 — CRD và Operator](LAB-14-CRD-VA-OPERATOR.md), cũng trả cluster về
> `04-metrics-ready`.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[Giai đoạn 15 — Windows, nếu môi trường có node Windows](../00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows)
— bảy bài khái niệm [174](../174-windows-vi.md), [175](../175-windows-intro-vi.md),
[176](../176-windows-user-guide-vi.md), [89](../89-windows-networking-vi.md),
[106](../106-windows-storage-vi.md), [112](../112-windows-resource-management-vi.md),
[131](../131-windows-security-vi.md), cộng bốn bài thực hành
[273](../273-configure-gmsa-vi.md), [278](../278-configure-runasusername-vi.md),
[281](../281-create-hostprocess-pod-vi.md), [315](../315-debug-windows-vi.md).

**Xương sống của lab là một câu hỏi và một phép đo.** Câu hỏi: cluster của bạn có node Windows
không, và nếu không thì phần nào của giai đoạn 15 **vẫn kiểm chứng được** trên chính cluster Linux
đang có? Phép đo là [B1](#b1-kiểm-tra-năng-lực-node-windows--chọn-nhánh) — sáu phép kiểm chỉ đọc,
không suy đoán, rồi chốt nhánh.

Điểm khác biệt của giai đoạn 15 so với mười bốn giai đoạn trước: **hệ điều hành của node lộ ra
trong API**. `.spec.os.name` khóa một danh sách trường; `kubernetes.io/os` là điều kiện lập lịch
thật; `securityContext` tách làm hai nửa Linux và Windows; Secret là cùng một object nhưng nằm ở hai
chỗ khác nhau trên đĩa node. **Tất cả những ranh giới đó đều đo được từ một cluster toàn Linux** —
bằng cách nhìn vào phía Linux của ranh giới, và bằng cách xem API server từ chối cái gì. Đó là nội
dung của nhánh B, và nó không phải phiên bản rút gọn của nhánh A.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không chép
lại con số phiên bản nào**. Thành phần ngoài baseline đang chạy — CNI thay Flannel, ingress
controller, dynamic provisioner, metrics-server — đã khóa ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00); lab 15 chỉ đọc, không
đụng vào, và **không thêm dòng nào vào hai bảng đó**.

Quan hệ với các lab đã học, để không lặp lại:
[Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) đã chứng minh `limits` bị kernel ép và container
vượt memory limit bị OOM kill — B6 **không làm lại** phép thử đó, chỉ đọc ranh giới cgroup và cấu
hình `NodeAllocatable` để đối chiếu với bài 112.
[Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md) đã làm trọn `nodeSelector`, taint và toleration — B2 và B3
**không dạy lại cơ chế**, chỉ dùng nó đúng theo cách bài 176 khuyến nghị cho cluster hỗn hợp hệ
điều hành.
[Lab 9b](LAB-9B-POD-SECURITY-VA-HARDENING.md) đã làm `securityContext` nhìn từ bên trong container —
B5 dùng lại đúng kỹ thuật đó để đo **ranh giới Linux/Windows**, không để dạy lại security context.
[Lab 5a](LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md) đã làm Service và DNS — B9 chỉ chạm vào đúng những
điểm mà bài 89 nói là khác trên Windows.
[Lab 13](LAB-13-DRA.md) là lab đầu tiên dùng khuôn "kiểm tra năng lực trước rồi mới rẽ nhánh"; lab
15 dùng lại đúng khuôn đó.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm bốn lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Bon lenh rieng cua lab 15: dung moc 04-metrics-ready, chua co gi cua lab 15 trong cluster.
kubectl top node
kubectl get namespace lab-15 --ignore-not-found
kubectl get runtimeclass 2>/dev/null || echo '(cluster khong co kieu runtimeclass)'
kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; **`kubectl top node` in đủ ba dòng số
liệu** — đó là định nghĩa của mốc `04-metrics-ready`; lệnh thứ hai không in gì vì chưa có namespace
`lab-15`; `kubectl get runtimeclass` trả `No resources found` hoặc liệt kê đúng những RuntimeClass
mà bản phân phối cài sẵn — B0 chốt con số này làm ảnh nền; cột `TAINTS` của **hai worker** là
`<none>`, còn control plane vẫn giữ taint `node-role.kubernetes.io/control-plane:NoSchedule` đúng
như [A5.4.3](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) quy định.

Nếu `kubectl top node` báo lỗi, hoặc worker nào còn taint sót, thì cluster chưa ở mốc đầu vào —
restore cả ba VM về `04-metrics-ready` trước khi tiếp tục.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Đọc **năng lực node Windows thật** của cluster mình — node nào chạy hệ điều hành nào, ai mang
  label `kubernetes.io/os=windows`, bề mặt API của GMSA có tồn tại không, host có VM Windows Server
  nào không — và kết luận **bằng bằng chứng chứ không bằng phỏng đoán**, kể cả khi kết luận là
  "cluster này chỉ có Linux".
- Vì sao Kubernetes chỉ hỗ trợ Windows **ở vai trò worker node**, và vì sao chạy `kubectl` trên máy
  Windows **không** làm cluster của bạn thành cluster có Windows.
- **`.spec.os.name` không phải điều kiện lập lịch.** Chứng minh được điều đó trên chính cluster của
  mình, và chỉ ra được thứ *thật sự* quyết định Pod rơi vào node nào: `nodeSelector`
  `kubernetes.io/os`, cộng taint và toleration khi cần bảo vệ node khỏi hệ sinh thái manifest cũ.
- Danh sách trường mà `.spec.os.name: windows` **khóa cứng**, đo được bằng chính API server của bạn:
  cùng một manifest, bỏ dòng `os.name` thì được nhận, thêm vào thì bị từ chối.
- Ranh giới `securityContext`: những trường bạn đã dùng suốt giai đoạn 9 — `runAsUser`, `runAsGroup`,
  `fsGroup`, `capabilities`, `readOnlyRootFilesystem`, `privileged`, `seLinuxOptions`,
  `seccompProfile` — **có tác dụng thật trên Linux** và **không có tương đương trên Windows**, cộng
  hai thứ duy nhất còn hoạt động ở phía Windows là `runAsNonRoot` và `windowsOptions`.
- `runAsUserName` của bài 278 hành xử thế nào khi apply lên một cluster toàn Linux — API server nhận
  hay từ chối, kubelet làm gì với nó — và vì sao đó chính là bài học về **field chỉ có nghĩa trên
  node Windows**.
- Ranh giới quản lý tài nguyên: trên Linux ranh giới là **cgroup của Pod** và `NodeAllocatable` được
  cưỡng chế; trên Windows ranh giới là **job object của từng container**, không có OOM killer, và
  `--kube-reserved`/`--system-reserved` chỉ dùng để tính toán. Đo được cả hai vế phía Linux.
- Vì sao `kubernetes.io/os` là chưa đủ khi cluster có nhiều phiên bản Windows Server, label nào giải
  quyết, và **RuntimeClass đóng gói `nodeSelector` cùng toleration như thế nào** — chứng minh bằng
  chính Pod của bạn, trên cluster toàn Linux.
- Vì sao `kubectl logs` có thể **không thấy gì** với một workload Windows dù workload đó đang ghi log
  đầy đủ, và chứng minh được nguyên nhân gốc bằng một phép thử tương đương trên Linux.
- Bốn kiểu Service dùng được với node Windows, và danh sách hạn chế mạng của bài 89 — trong đó có
  cái bạn **kiểm chứng được vế Linux** ngay trên cluster của mình.
- Ranh giới lưu trữ của bài 106: `subPath`, `defaultMode`, `emptyDir.medium: Memory` và quyền theo
  uid/gid **chạy thật trên Linux** và **không có trên Windows**, cùng một nguyên nhân gốc duy nhất.
- Cleanup đúng phạm vi: xóa cả object phạm vi cluster, gỡ sạch taint đã đặt, và — ở nhánh B —
  chứng minh cluster trở về đúng `04-metrics-ready` bằng những phép **so sánh giá trị trước/sau**,
  trong đó có phép so chứng minh lab **không cài thêm gì**.

### Nhánh A hay nhánh B — quyết định bằng B1

[B1](#b1-kiểm-tra-năng-lực-node-windows--chọn-nhánh) chạy sáu phép kiểm chỉ đọc rồi đặt biến `NHANH`:

| Điều kiện đo ở B1 | Nhánh **A** | Nhánh **B** |
| --- | --- | --- |
| `WIN_N` — số Node có `.status.nodeInfo.operatingSystem` bằng `windows` | ≥ 1 | = 0 |
| `WIN_READY` — số Node Windows ở condition `Ready=True` | ≥ 1 | không áp dụng |
| `WIN_LBL` — số Node mang label `kubernetes.io/os=windows` | bằng `WIN_N` | = 0 |
| B11 — thêm node Windows và chạy workload Windows | **chạy** | bỏ qua |
| Điểm kết thúc | **tạo** `15-windows-ready` | **trả về** `04-metrics-ready` |

Ba điều kiện phải đúng **đồng thời** mới vào nhánh A. `WIN_LBL` khác `WIN_N` là dấu hiệu có người
gán label bằng tay: label `kubernetes.io/os` do kubelet tự đặt theo hệ điều hành thật, nên hai con
số lệch nhau nghĩa là cluster đang nói dối về chính nó — đó là `GHI NHAN:`, không phải nhánh A.

**Cluster lab của [Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm) có đúng ba VM Linux, nên nhánh B
là nhánh mặc định và đúng đắn — không phải nhánh thất bại.** Giai đoạn 15 được lộ trình đánh dấu
tùy chọn từ đầu chính vì lý do này. Nhánh B kiểm chứng thật, trên cluster thật, mười nhóm phép thử
có gate; nó chỉ không dựng được thứ mà cluster này về bản chất không có.

**STOP:** không dựng thêm VM Windows Server, không tải image Windows, không cài thêm bất cứ thứ gì
để ép sang nhánh A. Ba việc đó đổi hạ tầng của chuỗi snapshot chính, mà nhánh B khai báo **trả
cluster về `04-metrics-ready`**. Muốn chạy nhánh A thì đó là một quyết định về môi trường — thêm một
VM vào bộ VM của Lab 00 — chứ không phải một thao tác chữa cháy giữa lab.

Kết quả nhánh B **không** sinh ra món nợ nào trong [sổ nợ lab](README.md#5-sổ-nợ-lab). Sổ nợ ghi
những phần bị chặn bởi **kiến thức của giai đoạn sau**; ở đây không thiếu kiến thức nào, chỉ thiếu
một hệ điều hành — và đó đúng là lý do bản đồ lab đánh dấu lab 15 **tùy chọn** thay vì ghi nợ.

**Trước khi làm B4 và B5, đọc lại [bài 175](../175-windows-intro-vi.md)** — cụ thể là mục *Tương
thích API* và ba mục con *Tương thích các trường…*, vì toàn bộ danh sách trường mà B4 và B5 kiểm
nằm ở đó. **Trước khi làm B2, B3 và B7, đọc lại [bài 176](../176-windows-user-guide-vi.md)** mục
*Taint và toleration* cùng hai mục con của nó.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm giai đoạn 15 | Kiểm chứng ở |
| --- | --- |
| [174 — Windows trong Kubernetes](../174-windows-vi.md) | B1 — sáu phép kiểm năng lực: hệ điều hành thật của từng Node, vai trò worker so với control plane, ba label chuẩn mà bài nhắc tên, và phép phân biệt "cài kubectl trên Windows" với "có node Windows trong cluster" |
| [273 — Cấu hình GMSA](../273-configure-gmsa-vi.md) | B1.4 — bề mặt API của GMSA: CRD `GMSACredentialSpec`, nhóm API `windows.k8s.io`, hai webhook mutating và validating mà bài quy định phải cài. Phần cấu hình và dùng GMSA thật nằm ở bảng lý do bên dưới |
| [176 — Hướng dẫn chạy Windows container](../176-windows-user-guide-vi.md) | B2 (`nodeSelector` `kubernetes.io/os` là cơ chế thật, hai chiều `windows` và `linux`), B3 (taint `os=windows:NoSchedule` cộng toleration khớp — đúng giải pháp bài đề xuất cho hệ sinh thái manifest cũ), B7 (`node.kubernetes.io/windows-build` và RuntimeClass đóng gói `nodeSelector` + toleration), B8.1 (log không ra STDOUT thì `kubectl logs` không thấy gì), B11.A5–B11.A6 (manifest `win-webserver` và bảy phép kiểm chứng kết nối) |
| [175 — Windows containers trong Kubernetes](../175-windows-intro-vi.md) | B4 (`.spec.os.name` khóa danh sách trường; scheduler không dùng nó), B5 (bốn khác biệt khái niệm hệ điều hành; hai trường duy nhất còn hoạt động trong `securityContext` của Pod Windows), B6 (kubelet trên Windows: không ràng buộc bộ nhớ hay CPU, `--enforce-node-allocable` và `PIDPressure` chưa triển khai, không OOM eviction), B8.2 (pause container), B11.A1 (Windows Server 2022/2025 và cách ly tiến trình) |
| [131 — Bảo mật cho các node Windows](../131-windows-security-vi.md) | B5.3 — Secret trên node: phía Linux là tmpfs trong bộ nhớ, đo được bằng `findmnt`; phía Windows là văn bản thuần trên đĩa cục bộ phải bù bằng file ACL và BitLocker. B5.4 — danh sách cơ chế cô lập Linux không có trên Windows, và cái còn lại là gì |
| [278 — Cấu hình RunAsUserName](../278-configure-runasusername-vi.md) | B5.1 — apply `securityContext.windowsOptions.runAsUserName` lên cluster toàn Linux và đọc chính xác kết quả ở ba tình huống `os.name`; B5.2 — ràng buộc định dạng `DOMAIN\USER` mà bài liệt kê, kiểm bằng một giá trị hợp lệ và một giá trị vi phạm; B11.A7 — `echo $env:USERNAME` trên node Windows thật |
| [112 — Quản lý tài nguyên cho node Windows](../112-windows-resource-management-vi.md) | B6 — QoS class của ba Pod; ranh giới cgroup **của Pod** trên Linux so với job object **của từng container** trên Windows; `enforceNodeAllocatable` và condition `PIDPressure` đọc từ node thật; vì sao bài cảnh báo phải đặt `limits` cho mọi container Windows |
| [281 — Tạo một Windows HostProcess Pod](../281-create-hostprocess-pod-vi.md) | B7.3 — bốn điều khiển mà bảng *Yêu cầu cấu hình cho HostProcess Pod* quy định, đo bằng schema và bằng phản ứng của API server với manifest trích đoạn của chính bài. Phần chạy HostProcess thật nằm ở bảng lý do bên dưới |
| [315 — Mẹo debug Windows](../315-debug-windows-vi.md) | B8 — quy trình debug: `kubectl logs` chỉ thấy STDOUT; pause/sandbox image trên node; đối chiếu từng mục của bài với bước tương ứng trong quy trình debug Linux đã dùng từ Lab 3a tới Lab 12 |
| [89 — Mạng trên Windows](../89-windows-networking-vi.md) | B9 — bốn kiểu Service dùng được trên cluster có node Windows; NodePort gọi từ chính node đó chạy trên Linux nhưng nằm trong danh sách *Hạn chế* của Windows; `/etc/resolv.conf` là file trên Linux còn DNS trên Windows nằm trong registry nên CNI phải gọi HNS |
| [106 — Lưu trữ trên Windows](../106-windows-storage-vi.md) | B10 — `subPath`, `defaultMode`, `emptyDir.medium: Memory`, quyền theo uid/gid và volume `readOnly`: bốn thứ đầu chạy thật trên Linux và không có trên Windows, thứ năm chạy ở cả hai; cùng một nguyên nhân gốc là SAM không chia sẻ giữa host và container |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [273](../273-configure-gmsa-vi.md), **toàn bộ phần cấu hình và sử dụng GMSA** — cấp phát GMSA trong Active Directory, cho worker node quyền đọc mật khẩu GMSA, sinh credspec bằng `New-CredentialSpec`, cài CRD và hai webhook, ClusterRole verb `use`, `gmsaCredentialSpecName` trong Pod spec | Cần **một domain Active Directory thật** cộng **host Windows đã join domain đó**. Bài ghi ở mục *Trước khi bạn bắt đầu* rằng cluster phải có worker node Windows, và ở mục *Cấu hình GMSA và node Windows trong Active Directory* rằng các worker node Windows "cần được cấu hình trong Active Directory để có quyền truy cập thông tin xác thực bí mật gắn với GMSA". Domain controller nằm ngoài phạm vi của **cả nhánh A**: nhánh A chỉ thêm **một** VM Windows Server làm worker, không dựng domain. Cài CRD và hai webhook mà không có domain thì chỉ tạo ra object rỗng và một Pod webhook không ai gọi tới — đó là diễn, không phải kiểm chứng. B1.4 làm đúng phần đo được: **bề mặt API GMSA có tồn tại trên cluster này không** |
| Bài [281](../281-create-hostprocess-pod-vi.md), **chạy một HostProcess container thật** — truy cập network namespace, filesystem và event log của host, tạo local usergroup bằng `net localgroup`, phân quyền bằng `icacls`, biến `$CONTAINER_SANDBOX_MOUNT_POINT` | HostProcess container **chỉ tồn tại trên Windows**: nó chạy như một tiến trình trên host Windows dưới `LocalSystem`, `LocalService`, `NetworkService` hoặc một local usergroup. Không có node Windows thì không có host để chạy, và cluster toàn Linux không có khái niệm tương đương — bài 175 và 131 đều nói HostProcess là **thay thế** cho privileged container chứ không phải ánh xạ một–một. B7.3 kiểm phần đo được: bốn điều khiển trong bảng *Yêu cầu cấu hình* và phản ứng của API server với manifest của bài |
| Bài [176](../176-windows-user-guide-vi.md), **LogMonitor** — chép binary và cấu hình vào container, sửa entrypoint để đẩy ETW và event log ra STDOUT | LogMonitor là công cụ của Microsoft chạy **bên trong Windows container**; nó không có bản Linux và không có gì để giám sát trên Linux, vì ETW và event log là cơ chế của Windows. B8.1 kiểm chứng **nguyên nhân** mà LogMonitor sinh ra để giải quyết: `kubectl logs` chỉ đọc STDOUT của container, nên workload ghi log đi chỗ khác thì lệnh đó trả về rỗng — điều này đo được trên Linux |
| Bài [176](../176-windows-user-guide-vi.md), `--register-with-taints='os=windows:NoSchedule'` **đặt lúc kubelet đăng ký node** | Đây là **cờ dòng lệnh của kubelet**; đặt nó là sửa cấu hình kubelet trên node — việc đó thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), không phải lab này, và lab 15 khai báo không sửa cấu hình node nào. B3 kiểm chứng **cơ chế** bằng cách đặt đúng taint đó lên `lab-k8s-worker2` bằng `kubectl taint`, chứng minh hai vế chặn và cho qua, rồi gỡ sạch. Nhánh A ghi lại cờ này ở đúng chỗ nó thuộc về: lúc dựng node Windows |
| Bài [89](../89-windows-networking-vi.md), mục *Hạn chế* — trần **64 Pod backend** cho một Service, IPv6 giữa các Pod Windows trên mạng overlay, Local Traffic Policy khi không dùng DSR, ICMP đi xuyên mạng ở xa | Bốn hạn chế này là thuộc tính của **data plane Windows (VFP)** và của HNS. Trên cluster Linux không có trần 64 backend để chạm tới và không có VFP để quan sát, nên không phép thử nào cho ra kết luận về Windows. Dựng 65 Pod backend trên hai worker Linux chỉ chứng minh Linux **không** có trần đó — một câu đã biết trước, không phải kiểm chứng |
| Bài [89](../89-windows-networking-vi.md), **năm chế độ mạng** L2bridge, L2tunnel, Overlay, Transparent, NAT; IPAM; DSR; bảng *Các thiết lập Service trên Windows* | Mỗi chế độ gắn với một CNI plugin chỉ chạy trên Windows — win-bridge, win-overlay, Azure-CNI, ovn-kubernetes — và với HNS. Cài chúng cần node Windows; đọc bảng để **chọn** plugin thì thuộc bước dựng node ở B11.A2 của nhánh A |
| Bài [106](../106-windows-storage-vi.md), **ánh xạ thiết bị block**, **lưu trữ NFS**, **mở rộng volume đã mount (resizefs)**, **chiếu mount ngược về host** | Ba thứ đầu cần hạ tầng lab không có: StorageClass của [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md) cấp volume chế độ Filesystem chứ không phải Block; bộ VM không có NFS server; và [Lab 6b](LAB-6B-SNAPSHOT-VA-VOLUME-NANG-CAO.md) đã đo và ghi lại rằng provisioner của baseline **không phải CSI driver**, nên không có đường mở rộng volume. Thứ tư — chiếu file từ container ngược ra host — không phải tính năng của Kubernetes trên bất kỳ hệ điều hành nào; bài liệt kê nó để nói Windows không làm được, không phải để so với Linux |
| Bài [112](../112-windows-resource-management-vi.md), cờ `--windows-priorityclass` và khuyến nghị **dành riêng ít nhất 2GiB** bộ nhớ trên node Windows | Cờ chỉ tồn tại trên kubelet của Windows; đặt nó cần node Windows, mà đặt nó trên node Linux là sửa cấu hình kubelet — thuộc giai đoạn 20. Con số 2GiB là khuyến nghị vận hành cho node Windows, đo được trên node Windows thật chứ không suy ra từ node Linux |
| Bài [175](../175-windows-intro-vi.md), **pause image ký authenticode của Microsoft**, **Mirantis Container Runtime**, **Windows Operational Readiness**, **node problem detector cho Windows**, khuyến nghị phần cứng và kích thước image | Toàn bộ là lựa chọn và quy trình nghiệm thu **trên node Windows**. Node problem detector còn là add-on ngoài baseline; cài add-on mới là đổi hạ tầng, mà nhánh B khai báo trả cluster về `04-metrics-ready`. Nhánh A dùng chúng đúng chỗ: B11.A1 và B11.A2 |
| Bài [131](../131-windows-security-vi.md), **file ACL** bảo vệ vị trí file Secret và **mã hóa cấp volume bằng BitLocker** | Hai biện pháp của hệ điều hành Windows, thực hiện trên chính node Windows. Trên Linux không có ACL kiểu Windows và không có BitLocker. B5.3 kiểm chứng vế đối chiếu mà bài dựa vào để cảnh báo: **trên Linux, Secret nằm trên tmpfs** — và đo được điều đó bằng `findmnt` |
| Bài [315](../315-debug-windows-vi.md), các lệnh sửa bằng **PowerShell và HNS** — `Get-HnsNetwork`, `Get-NetAdapter`, `start.ps1`, chạy lại `flanneld.exe`, `SourceVip.json`, `ExceptionList` trong `cni.conf`, `wincat`, MAC spoofing, biến proxy máy | Mọi lệnh đều chạy **trên một node Windows** và thao tác trên HNS — thành phần không tồn tại trên Linux. B8.3 đối chiếu từng mục của bài với bước tương ứng trong quy trình debug Linux để bạn biết mục nào thay thế mục nào; nhánh A là nơi chạy chúng thật |
| `securityContext.seLinuxOptions` **có tác dụng thật trên Linux** | Baseline của [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) là Ubuntu Server, dùng **AppArmor** chứ không phải SELinux; không có SELinux thì trường này không cho ra hiệu ứng quan sát được ngay cả trên node Linux. B5.4 vẫn kiểm được vế cần cho giai đoạn 15: cùng trường đó bị API server **từ chối** khi `.spec.os.name` là `windows` |

### 1.2. Thời lượng

Ghi riêng cho từng nhánh, tính từ lúc gate mở đầu đã PASS:

| Nhánh | Thời lượng | Ghi chú |
| --- | --- | --- |
| **B** — mặc định trên cluster lab | **2–3 giờ** | B0 tới B10 rồi B12; không có bước chờ dựng máy |
| **A** | **4 giờ** | B0 tới B12 đầy đủ. **Chưa tính** thời gian cài Windows Server, cài container runtime và kéo Windows container image — ba việc đó phụ thuộc tốc độ mạng và cấu hình host, làm xong trước khi mở lab |

B2, B3 và B7 có bước phải chờ scheduler phản ứng; B6 có bước chờ Pod `Ready`. **Thời gian chờ phụ
thuộc cấu hình cluster**, nên mọi bước chờ đều viết dưới dạng `kubectl wait --timeout` hoặc vòng lặp
có điều kiện dừng, không phải một con số cố định.

---

## 2. Quy ước và an toàn

- Mọi lệnh chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi rõ node khác**.
  Lệnh cần đọc file trên worker chạy qua `ssh` từ master, đúng cách
  [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) và
  [Lab 12](LAB-12-VAN-HANH-VONG-DOI-NODE.md) đã dùng.
- Các khối PowerShell chạy trên **máy host Windows**, không phải trên VM. Chúng luôn được mở đầu
  bằng dòng "Chạy trên máy host".
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Rất nhiều gate so sánh với biến đặt ở bước
  trước (`EV`, `WK`, `NS`, `W1`, `W2`, `LINUX_N`, `WIN_N`, `NHANH`, `RC_BEFORE`, `API_BEFORE`,
  `CRD_BEFORE`, `NODE_N`, `TAINT_BEFORE`, `LBL_BEFORE`); mở shell mới giữa chừng là mất biến và mất
  luôn gate cuối.
- **Lab này không cài gì và không sửa cấu hình node nào:** không add-on, không CRD, không Helm chart,
  không bật feature gate, không sửa file trong `/etc/kubernetes/manifests`, không sửa
  `/var/lib/kubelet/config.yaml`, không đặt cờ `--register-with-taints`, và ở nhánh B không dựng
  thêm VM. Thiếu điều kiện thì ghi nhận thiếu, không đi vá.
- Lab tạo Namespace `lab-15` cùng object bên trong nó, cộng **một object phạm vi cluster**:
  RuntimeClass `lab-15-windows` ở B7. Cả hai bị xóa ở B12, và gate cuối so số lượng trước/sau để
  chứng minh điều đó.
- **Taint duy nhất lab đặt là `os=windows:NoSchedule` trên `lab-k8s-worker2`**, đặt ở B3 và gỡ ngay
  trong chính B3. Đây là bước fault injection duy nhất của lab, và nó tuân thủ quy ước chung: **chỉ
  `lab-k8s-worker2`**. Không đặt taint lên `lab-k8s-worker1`, tuyệt đối không đụng vào taint
  `node-role.kubernetes.io/control-plane:NoSchedule` của control plane.
- Image dùng cho toàn bộ nhánh B là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và đã có sẵn trên cả ba node từ
  Lab 00, nên nhánh B **không phụ thuộc mạng ra ngoài**. Nhánh A cần thêm Windows container image,
  và đó là một trong những lý do nhánh A phải chuẩn bị trước khi mở lab.
- Manifest tạm ghi vào `~/lab-work/15/`; bằng chứng ghi vào `~/lab-evidence/15/`. Không lưu token,
  key hay certificate.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng bắt đầu bằng `STOP:` nghĩa là
  dừng đúng bước đó và ghi lại, không đi vòng. Dòng bắt đầu bằng `GHI NHAN:` là kết quả **hợp lệ
  nhưng khác mặc định** — chép nguyên văn vào evidence rồi đọc lại phần *Ý nghĩa* trước khi kết
  luận.
- **Vài bước của lab cố ý đo phản ứng của API server thay vì khẳng định trước kết quả.** Chúng dùng
  `--dry-run=server` và ghi lại mã thoát cùng thông báo lỗi. Gate của những bước đó là *kết quả đã
  được ghi lại và rơi đúng vào một trong các lớp đã liệt kê* — không phải "phải ra đúng lớp này".
  Cách này giữ lab trung thực: bạn đọc câu trả lời của cluster mình, không đọc dự đoán của lab.
- **Cách quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `04-metrics-ready` — không bao
  giờ restore riêng một VM, xem ghi chú cuối
  [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) — rồi bật lại theo
  thứ tự master → worker 1 → worker 2, chạy lại gate mở đầu và bắt đầu lại từ B0.
- **Trước khi chạy B0, xác nhận trên máy host rằng cả ba VM đều còn mốc `04-metrics-ready`.** Không
  có mốc này thì không có đường lui. Chạy trên **máy host**, PowerShell:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmx = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
)
foreach ($f in $vmx) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '04-metrics-ready') { "PASS: $f" } else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** đúng ba dòng `PASS:`, không dòng `FAIL:` nào. Không mở B0 khi còn dòng `FAIL:`. Đường dẫn
`.vmx` ở trên theo máy host đang dùng; sửa lại nếu VM nằm chỗ khác. **Giữ nguyên cửa sổ PowerShell
này** — B1.6 và B12 dùng lại hai biến `$vmrun` và `$vmx`.

---
# Phần B — Thực hành kiến thức giai đoạn 15

## B0. Chuẩn bị workspace và ảnh nền của cluster

**Mục đích:** chốt các con số nền để B12 có cái đối chiếu. Lab này khẳng định "không cài gì và không
đổi gì", và khẳng định đó chỉ có giá trị khi có số đo trước và số đo sau.

```bash
mkdir -p ~/lab-work/15 ~/lab-evidence/15
EV="$HOME/lab-evidence/15"
WK="$HOME/lab-work/15"
NS='lab-15'

kubectl config current-context
MASTER="$(kubectl get node -l node-role.kubernetes.io/control-plane \
  -o jsonpath='{.items[0].metadata.name}')"
W1='lab-k8s-worker1'
W2='lab-k8s-worker2'
echo "MASTER=$MASTER | W1=$W1 | W2=$W2"

kubectl create namespace "$NS"
kubectl get namespace "$NS" -o jsonpath='{.status.phase}'; echo
```

Chốt các con số nền:

```bash
NODE_N="$(kubectl get nodes --no-headers | wc -l)"
API_BEFORE="$(kubectl api-resources -o name 2>/dev/null | sort -u | wc -l)"
CRD_BEFORE="$(kubectl get crd -o name 2>/dev/null | wc -l)"
RC_BEFORE="$(kubectl get runtimeclass -o name 2>/dev/null | wc -l)"
NS_BEFORE="$(kubectl get namespace -o name | sort | tr '\n' ' ')"
TAINT_BEFORE="$(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}={.spec.taints[*].key}{"\n"}{end}' | sort | tr '\n' ' ')"
LBL_BEFORE="$(kubectl get nodes --no-headers --show-labels | awk '{print $1, $NF}' | sort | md5sum)"

{
  echo "=== $(date -Is) — anh nen truoc Lab 15 ==="
  echo "MASTER=$MASTER W1=$W1 W2=$W2 NODE_N=$NODE_N"
  echo "API_BEFORE=$API_BEFORE CRD_BEFORE=$CRD_BEFORE RC_BEFORE=$RC_BEFORE"
  echo "NS_BEFORE=$NS_BEFORE"
  echo "TAINT_BEFORE=$TAINT_BEFORE"
  echo "LBL_BEFORE=$LBL_BEFORE"
} | tee "$EV/b0-anh-nen.txt"

test "$NODE_N" -ge 3 && echo "PASS: cluster co $NODE_N node"
test -s "$EV/b0-anh-nen.txt" && echo 'PASS: da ghi anh nen'
echo "$TAINT_BEFORE" | grep -q 'lab-k8s-worker2=' \
  && echo 'GHI NHAN: worker2 dang mang taint tu truoc — doc muc 4 truoc khi lam B3' \
  || echo 'PASS: worker2 chua mang taint nao — dung moc 04-metrics-ready'
```

**Ý nghĩa:** `TAINT_BEFORE` là trục quan trọng nhất của lab này. B3 đặt một taint lên
`lab-k8s-worker2` rồi gỡ, và cách duy nhất chứng minh đã gỡ sạch là so chuỗi này trước và sau.
`RC_BEFORE` là trục thứ hai: B7 tạo một RuntimeClass phạm vi cluster, và B12 phải chứng minh số
RuntimeClass trở về đúng con số ban đầu.

**PASS:** namespace `lab-15` ở phase `Active`; hai dòng `PASS:` đầu xuất hiện; dòng thứ ba là `PASS:`
hoặc `GHI NHAN:` và nội dung đã vào evidence; file `b0-anh-nen.txt` có đủ sáu dòng.

---

## B1. Kiểm tra năng lực node Windows — chọn nhánh

**Mục đích:** trả lời một câu hỏi duy nhất bằng bằng chứng: **cluster này có node Windows không?**
Bài [174](../174-windows-vi.md) mở đầu bằng đúng phép phân biệt mà mục này đo — "cài kubectl trên
Windows" khác "có worker node Windows trong cluster" — và bài
[176](../176-windows-user-guide-vi.md) mục *Đảm bảo workload đặc thù hệ điều hành được đặt lên đúng
host container* liệt kê chính xác những label mà mọi node đều có sẵn.

**Toàn bộ B1 chỉ đọc.** Không tạo, không sửa, không gán label, không đặt taint.

### B1.1. Hệ điều hành thật của từng Node

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,OS:.status.nodeInfo.operatingSystem,ARCH:.status.nodeInfo.architecture,OSIMAGE:.status.nodeInfo.osImage,KUBELET:.status.nodeInfo.kubeletVersion,RUNTIME:.status.nodeInfo.containerRuntimeVersion' \
  | tee "$EV/b1-nodeinfo.txt"

kubectl get nodes -o custom-columns='NODE:.metadata.name,ROLES:.metadata.labels.node-role\.kubernetes\.io/control-plane' \
  | tee "$EV/b1-roles.txt"
```

**Ý nghĩa:** `.status.nodeInfo.operatingSystem` là giá trị **kubelet tự báo lên** theo hệ điều hành
nó đang chạy trên đó; không ai gõ tay được vào đây. Đây là nguồn sự thật của B1, còn label ở B1.2
chỉ là bản sao mà kubelet đặt kèm. `osImage` cho biết bản phát hành cụ thể — với node Windows nó là
chuỗi Windows Server, và bài [175](../175-windows-intro-vi.md) mục *Tương thích phiên bản hệ điều
hành Windows* quy định chuỗi đó phải là **Windows Server 2022 hoặc Windows Server 2025**.

Cột `ROLES` trả lời câu hỏi vai trò của bài 174: Kubernetes hỗ trợ Windows **ở vai trò worker node**,
control plane chỉ chạy trên Linux. Nếu cluster của bạn có node Windows, nó phải nằm ở dòng có
`ROLES` là `<none>`.

**PASS:** hai file evidence được ghi; mỗi node có đúng một giá trị `OS`, không trống; đúng một node
mang giá trị ở cột `ROLES` — đó là control plane.

### B1.2. Đếm node theo label `kubernetes.io/os` và đọc ba label chuẩn

```bash
LINUX_N="$(kubectl get nodes -l kubernetes.io/os=linux --no-headers 2>/dev/null | wc -l)"
WIN_LBL="$(kubectl get nodes -l kubernetes.io/os=windows --no-headers 2>/dev/null | wc -l)"
echo "LINUX_N=$LINUX_N | WIN_LBL=$WIN_LBL"

kubectl get nodes -o custom-columns='NODE:.metadata.name,OS_LABEL:.metadata.labels.kubernetes\.io/os,ARCH_LABEL:.metadata.labels.kubernetes\.io/arch,WINBUILD:.metadata.labels.node\.kubernetes\.io/windows-build' \
  | tee "$EV/b1-labels.txt"

WINBUILD_N="$(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.labels.node\.kubernetes\.io/windows-build}{"\n"}{end}' \
  | sed '/^$/d' | wc -l)"
echo "WINBUILD_N=$WINBUILD_N"

test "$(( LINUX_N + WIN_LBL ))" -eq "$NODE_N" \
  && echo 'PASS: moi node deu mang label kubernetes.io/os — khong node nao thieu'
```

**Ý nghĩa:** bài 176 nói **tất cả các node** đều có sẵn hai label `kubernetes.io/os` và
`kubernetes.io/arch`. Phép cộng ở gate trên biến câu đó thành phép đo: tổng số node dán `linux` cộng
số node dán `windows` phải bằng tổng số node; lệch là có node thiếu label, và khi đó mọi
`nodeSelector` theo hệ điều hành đều mất tác dụng với node đó.

Label thứ ba, `node.kubernetes.io/windows-build`, **chỉ Kubernetes tự thêm cho node Windows**. Trên
cluster toàn Linux `WINBUILD_N` bằng 0, và đó là điều đúng — B7 quay lại chỗ này khi bàn cluster có
nhiều phiên bản Windows Server.

**PASS:** dòng `PASS: moi node deu mang label kubernetes.io/os…` xuất hiện; file `b1-labels.txt` có
đủ số dòng bằng số node.

### B1.3. Node Windows nào đang `Ready`

```bash
WIN_N="$(kubectl get nodes -o jsonpath='{range .items[*]}{.status.nodeInfo.operatingSystem}{"\n"}{end}' \
  | grep -cx 'windows' || true)"
WIN_READY="$(kubectl get nodes -o jsonpath='{range .items[*]}{.status.nodeInfo.operatingSystem}{" "}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}' \
  | grep -c '^windows True$' || true)"
echo "WIN_N=$WIN_N | WIN_READY=$WIN_READY | WIN_LBL=$WIN_LBL"

test "$WIN_LBL" -eq "$WIN_N" \
  && echo 'PASS: so node mang label windows khop so node co OS windows' \
  || echo 'GHI NHAN: label kubernetes.io/os khong khop .status.nodeInfo.operatingSystem'
```

**Ý nghĩa:** ba con số này là đầu vào của bảng quyết định nhánh ở [mục 1](#nhánh-a-hay-nhánh-b--quyết-định-bằng-b1).
Chúng phải nhất quán với nhau. `WIN_LBL` lớn hơn `WIN_N` nghĩa là có ai đó gán tay label
`kubernetes.io/os=windows` lên một node Linux — cluster khi đó sẽ lập lịch Pod Windows lên node
Linux và Pod chết ở tầng kubelet. Đây là lỗi cấu hình, không phải nhánh A.

**PASS:** ba biến đều có giá trị số; dòng `PASS:` hoặc `GHI NHAN:` của bước này xuất hiện và được
chép vào evidence.

### B1.4. Bề mặt API của GMSA

Bài [273](../273-configure-gmsa-vi.md) mở đầu bằng hai bước khởi tạo **một lần cho mỗi cluster**:
cài CRD `GMSACredentialSpec`, và cài hai webhook — một mutating để mở rộng tham chiếu GMSA thành
credspec đầy đủ, một validating để kiểm service account của Pod có được dùng GMSA đó không. Mục này
đo xem cluster của bạn đã có ba thứ đó chưa.

```bash
{
  echo '--- 1. API resource nao chua chu gmsa ---'
  kubectl api-resources 2>/dev/null | grep -i gmsa || echo '(khong co API resource nao chua chu gmsa)'

  echo '--- 2. CRD nao chua chu gmsa ---'
  kubectl get crd -o name 2>/dev/null | grep -i gmsa || echo '(khong co CRD nao chua chu gmsa)'

  echo '--- 3. nhom API windows.k8s.io ---'
  kubectl api-resources --api-group=windows.k8s.io 2>&1

  echo '--- 4. webhook cua GMSA ---'
  kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations -o name 2>/dev/null \
    | grep -i gmsa || echo '(khong co webhook nao chua chu gmsa)'
} | tee "$EV/b1-gmsa.txt"

GMSA_CRD="$(kubectl get crd -o name 2>/dev/null | grep -ci gmsa || true)"
GMSA_WH="$(kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations -o name 2>/dev/null \
  | grep -ci gmsa || true)"
echo "GMSA_CRD=$GMSA_CRD | GMSA_WH=$GMSA_WH" | tee -a "$EV/b1-gmsa.txt"

if [ "$GMSA_CRD" -eq 0 ] && [ "$GMSA_WH" -eq 0 ]; then
  echo 'PASS: cluster khong co be mat API GMSA — dung ky vong cua mot cluster kubeadm sach'
else
  echo "GHI NHAN: cluster da co mot phan be mat API GMSA (CRD=$GMSA_CRD, webhook=$GMSA_WH)"
fi
```

**Ý nghĩa:** GMSA **không đi kèm Kubernetes**. Bài 273 nói rõ CRD và hai webhook phải được cài thêm
bằng manifest hoặc script của dự án `windows-gmsa`. Một cluster kubeadm sạch không có gì trong bốn
phép kiểm trên, và đó là kết luận đúng — không phải thiếu sót.

Kết luận này quan trọng cho checkpoint: khi ai đó hỏi "cluster của anh dùng GMSA được không", câu
trả lời có ba tầng — (1) bề mặt API có chưa, (2) có node Windows không, (3) node Windows đã join
domain Active Directory chưa. B1.4 trả lời tầng một; B1.3 trả lời tầng hai; tầng ba nằm ngoài phạm
vi lab và đã ghi rõ ở [bảng lý do](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

**PASS:** file `b1-gmsa.txt` có đủ bốn mục đánh số cộng dòng `GMSA_CRD=… | GMSA_WH=…`; một dòng
`PASS:` hoặc `GHI NHAN:` của bước này xuất hiện.

### B1.5. RuntimeClass hiện có

```bash
kubectl get runtimeclass -o wide 2>&1 | tee "$EV/b1-runtimeclass.txt"
echo "RC_BEFORE=$RC_BEFORE" | tee -a "$EV/b1-runtimeclass.txt"
test -n "$RC_BEFORE" && echo "PASS: da chot so RuntimeClass nen — RC_BEFORE=$RC_BEFORE"
```

**Ý nghĩa:** bài 176 mục *Đơn giản hóa với RuntimeClass* dùng RuntimeClass để đóng gói sẵn
`nodeSelector` cùng toleration cho Windows. B7 tạo một RuntimeClass như vậy và B12 xóa nó; con số
`RC_BEFORE` là mốc so.

**PASS:** dòng `PASS: da chot so RuntimeClass nen…` xuất hiện.

### B1.6. Host có VM Windows Server nào không

**Chạy trên máy host**, PowerShell — dùng lại hai biến `$vmrun` và `$vmx` đã đặt ở [mục 2](#2-quy-ước-và-an-toàn):

```powershell
$lines = & $vmrun -T ws list
$lines
$paths = $lines | Select-Object -Skip 1
$extra = @($paths | Where-Object { $vmx -notcontains $_ })
if ($extra.Count -eq 0) {
  "PASS: chi ba VM Linux cua Lab 00 dang chay - khong VM nao khac"
} else {
  "GHI NHAN: co VM khac dang chay -> $($extra -join ', ')"
}
```

Chép nguyên văn output sang master và lưu vào evidence:

```bash
cat > "$EV/b1-host-vm.txt" <<'EOF'
# Chep nguyen van output cua khoi PowerShell B1.6 vao day.
EOF
echo 'PASS: da tao cho ghi ket qua vmrun list' && test -f "$EV/b1-host-vm.txt" \
  && echo 'PASS: file b1-host-vm.txt ton tai'
```

**Ý nghĩa:** `vmrun -T ws list` chỉ liệt kê **VM đang chạy**, không liệt kê VM đã tắt trong thư viện.
Vì vậy phép kiểm này trả lời câu hỏi hẹp và trung thực: *ngoài ba VM Linux của Lab 00, còn máy ảo
nào đang chạy trên host này không?* Đó là mức chắc chắn phù hợp — nó chứng minh không có node
Windows đang chạy để join, và nó không giả vờ biết trong thư viện VM của bạn có gì.

Kết hợp với B1.1: kể cả khi host có một VM Windows đang chạy, nó **chỉ** thành node Windows sau khi
join cluster và kubelet báo `operatingSystem=windows`. Đây chính là phép phân biệt của bài 174 —
máy Windows tồn tại là một chuyện, node Windows trong cluster là chuyện khác.

**PASS:** khối PowerShell in ra đúng một dòng `PASS:` hoặc `GHI NHAN:`; file `b1-host-vm.txt` đã có
nội dung chép về.

### B1.7. Tính toán và chốt nhánh

```bash
if [ "$WIN_N" -ge 1 ] && [ "$WIN_READY" -ge 1 ] && [ "$WIN_LBL" -eq "$WIN_N" ]; then
  NHANH=A
else
  NHANH=B
fi

{
  echo "=== $(date -Is) — kiem tra nang luc node Windows ==="
  echo "1. tong so Node                                : $NODE_N"
  echo "2. Node co .status.nodeInfo.operatingSystem=windows : $WIN_N"
  echo "3. Node Windows o condition Ready=True         : $WIN_READY"
  echo "4. Node mang label kubernetes.io/os=windows    : $WIN_LBL"
  echo "5. Node mang label node.kubernetes.io/windows-build : $WINBUILD_N"
  echo "6. be mat API GMSA (CRD / webhook)             : $GMSA_CRD / $GMSA_WH"
  echo "NHANH                                          : $NHANH"
} | tee "$EV/b1-nang-luc.txt"

echo "$NHANH" > "$EV/b1-nhanh.txt"

if [ "$NHANH" = A ]; then
  echo 'NHANH A: cluster co node Windows Ready — B11 chay, lab ket thuc bang moc 15-windows-ready'
else
  echo 'NHANH B: cluster toan Linux — bo qua B11, lab tra cluster ve 04-metrics-ready'
fi
```

**PASS:** in ra **đúng một** dòng bắt đầu bằng `NHANH A:` hoặc `NHANH B:`; file `b1-nang-luc.txt` có
đủ sáu dòng đánh số cộng dòng `NHANH`; file `b1-nhanh.txt` chứa đúng một ký tự `A` hoặc `B`. Sáu con
số trong file phải khớp với những gì B1.1 đến B1.6 in ra.

**STOP:** **không** dựng thêm VM Windows Server, **không** tải Windows container image, **không**
gán tay label `kubernetes.io/os=windows`, **không** cài CRD hay webhook GMSA để ép sang nhánh A.
Cluster lab của Lab 00 chỉ có ba VM Linux, nên **nhánh B là nhánh mặc định và đúng đắn, không phải
nhánh thất bại**. Mười mục còn lại của phần B đều chạy thật ở nhánh B, có gate thật, trên cluster
thật. Muốn có nhánh A thì phải chuẩn bị môi trường trước khi mở lab, không phải vá giữa chừng.

---

## B2. `nodeSelector kubernetes.io/os` — cơ chế thật để đặt workload lên đúng hệ điều hành

**Mục đích:** kiểm chứng câu trung tâm của bài [176](../176-windows-user-guide-vi.md): *"Nếu đặc tả
của Pod không chỉ định `nodeSelector` chẳng hạn như `"kubernetes.io/os": windows`, Pod đó có thể bị
lập lịch lên bất kỳ host nào"*, và *"thực hành tốt nhất … là sử dụng `nodeSelector`"*.

Mục này chạy ở **cả hai nhánh**, và trên một cluster toàn Linux nó là **kiểm chứng thật, không phải
mô phỏng**: bạn tạo một Pod đòi node Windows trên một cluster không có node Windows, rồi đọc lý do
scheduler đưa ra.

### B2.1. Pod chọn `windows` trên cluster không có node Windows

```bash
cat > "$WK/os-selector.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: os-selector-windows
  namespace: lab-15
  labels:
    app: os-probe
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/os: windows
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
YAML

kubectl apply -f "$WK/os-selector.yaml"

for i in $(seq 1 24); do
  kubectl -n "$NS" get events \
    --field-selector involvedObject.name=os-selector-windows,reason=FailedScheduling \
    -o name 2>/dev/null | grep -q . && break
  sleep 5
done

PHASE_W="$(kubectl -n "$NS" get pod os-selector-windows -o jsonpath='{.status.phase}')"
NODE_W="$(kubectl -n "$NS" get pod os-selector-windows -o jsonpath='{.spec.nodeName}')"
COND_W="$(kubectl -n "$NS" get pod os-selector-windows \
  -o jsonpath='{.status.conditions[?(@.type=="PodScheduled")].status}')"
MSG_W="$(kubectl -n "$NS" get events \
  --field-selector involvedObject.name=os-selector-windows,reason=FailedScheduling \
  -o jsonpath='{.items[-1:].message}')"

kubectl -n "$NS" describe pod os-selector-windows | tee "$EV/b2-windows-describe.txt"
echo "phase=$PHASE_W | nodeName=${NODE_W:-<chua gan>} | PodScheduled=${COND_W:-<chua co>}"
echo "message=$MSG_W" | tee "$EV/b2-windows-event.txt"

test "$PHASE_W" = 'Pending' && echo 'PASS: Pod chon kubernetes.io/os=windows nam Pending'
test -z "$NODE_W" && echo 'PASS: Pod chua duoc gan node nao — no dung o buoc lap lich'
test "$COND_W" = 'False' && echo 'PASS: condition PodScheduled=False — scheduler da xet va tu choi'
test -n "$MSG_W" && echo 'PASS: scheduler co ghi lai ly do trong Event reason=FailedScheduling'
if echo "$MSG_W" | grep -qiE 'selector|affinity'; then
  echo 'PASS: ly do scheduler dua ra noi thang ve node selector'
else
  echo "GHI NHAN: thong bao cua scheduler khong chua tu selector/affinity — chep nguyen van: $MSG_W"
fi
```

**Ý nghĩa:** bốn tín hiệu phải đọc cùng nhau. `Pending` một mình không nói gì — Pod đang kéo image
cũng `Pending`. `spec.nodeName` rỗng mới chứng minh Pod chưa qua được bước lập lịch. `PodScheduled`
bằng `False` là chữ ký của scheduler nói rằng nó **đã xét** và không xếp được. Và Event
`FailedScheduling` là chỗ scheduler viết ra **lý do**.

Đây chính là điều sẽ xảy ra trong cluster hỗn hợp nếu bạn viết `nodeSelector` sai chính tả hoặc đòi
một phiên bản Windows build mà không node nào có. Bạn vừa học cách chẩn đoán nó ở đúng ba chỗ.

**PASS:** bốn dòng `PASS:` đầu xuất hiện; dòng thứ năm là `PASS:` hoặc `GHI NHAN:` và nội dung đã
vào evidence.

### B2.2. Cùng manifest đó, đổi một giá trị

```bash
sed 's/kubernetes.io\/os: windows/kubernetes.io\/os: linux/; s/os-selector-windows/os-selector-linux/' \
  "$WK/os-selector.yaml" > "$WK/os-selector-linux.yaml"
grep -n 'name:\|kubernetes.io/os' "$WK/os-selector-linux.yaml"

kubectl apply -f "$WK/os-selector-linux.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/os-selector-linux --timeout=180s

NODE_L="$(kubectl -n "$NS" get pod os-selector-linux -o jsonpath='{.spec.nodeName}')"
OS_L="$(kubectl get node "$NODE_L" -o jsonpath='{.metadata.labels.kubernetes\.io/os}')"
echo "nodeName=$NODE_L | kubernetes.io/os cua node do=$OS_L"

test -n "$NODE_L" && echo 'PASS: Pod chon linux duoc gan node ngay'
test "$OS_L" = 'linux' && echo 'PASS: node nhan Pod dung la node mang label kubernetes.io/os=linux'
```

**Ý nghĩa:** hai Pod khác nhau **đúng một giá trị label**, và kết quả ngược hẳn nhau. Đó là bằng
chứng trực tiếp rằng `nodeSelector kubernetes.io/os` là điều kiện lập lịch **thật**, chứ không phải
một nhãn mô tả. B4 sẽ cho thấy `.spec.os.name` — cái tên trông giống hệt về ý nghĩa — hành xử ngược
lại hoàn toàn.

**PASS:** `kubectl wait` trả về thành công; hai dòng `PASS:` của bước này xuất hiện.

```bash
kubectl -n "$NS" delete pod os-selector-windows os-selector-linux --wait=true --timeout=120s
```

---

## B3. Taint và toleration theo hệ điều hành

**Mục đích:** kiểm chứng giải pháp thay thế mà bài [176](../176-windows-user-guide-vi.md) đề xuất khi
bạn **không thể** sửa `nodeSelector` cho toàn bộ manifest cũ, Helm chart cộng đồng và Pod do operator
sinh ra: *"Bằng cách thêm taint vào tất cả các node Windows, sẽ không có gì được lập lịch lên chúng
(bao gồm cả các Pod Linux hiện có). Để một Pod Windows được lập lịch lên một node Windows, Pod đó
cần cả `nodeSelector` lẫn toleration khớp tương ứng."*

> **Cảnh báo — mục này đặt taint lên một node.** Chỉ `lab-k8s-worker2`, đúng quy ước fault injection
> của [mục 6 README](README.md#6-quy-ước-chung-trong-mọi-lab). Taint được gỡ ngay trong B3.4 và gate
> cuối B12 so chuỗi taint trước/sau để chứng minh không còn sót. **Không** đặt taint lên
> `lab-k8s-worker1`, **không** đụng vào taint của control plane.

Cluster lab không có node Windows để taint, nên mục này taint `lab-k8s-worker2` bằng **đúng cặp
key/value/effect mà bài 176 dùng** — `os=windows:NoSchedule`. Thứ được kiểm chứng ở đây là **cơ chế
taint/toleration theo hệ điều hành**, không phải bản thân node Windows.

### B3.1. Đặt taint và ghi lại trạng thái trước

```bash
kubectl get node "$W2" -o jsonpath='{.spec.taints}{"\n"}' | tee "$EV/b3-taint-truoc.txt"
kubectl taint node "$W2" os=windows:NoSchedule
kubectl get node "$W2" -o jsonpath='{range .spec.taints[*]}{.key}={.value}:{.effect}{"\n"}{end}' \
  | tee "$EV/b3-taint-dat.txt"

grep -qx 'os=windows:NoSchedule' "$EV/b3-taint-dat.txt" \
  && echo 'PASS: taint os=windows:NoSchedule da nam tren worker2'
```

**PASS:** dòng `PASS: taint os=windows:NoSchedule da nam tren worker2` xuất hiện.

### B3.2. Pod không có toleration bị chặn

```bash
cat > "$WK/taint-khong-toleration.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: taint-no-tol
  namespace: lab-15
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
YAML

kubectl apply -f "$WK/taint-khong-toleration.yaml"

for i in $(seq 1 24); do
  kubectl -n "$NS" get events \
    --field-selector involvedObject.name=taint-no-tol,reason=FailedScheduling \
    -o name 2>/dev/null | grep -q . && break
  sleep 5
done

PHASE_T="$(kubectl -n "$NS" get pod taint-no-tol -o jsonpath='{.status.phase}')"
NODE_T="$(kubectl -n "$NS" get pod taint-no-tol -o jsonpath='{.spec.nodeName}')"
MSG_T="$(kubectl -n "$NS" get events \
  --field-selector involvedObject.name=taint-no-tol,reason=FailedScheduling \
  -o jsonpath='{.items[-1:].message}')"
echo "phase=$PHASE_T | nodeName=${NODE_T:-<chua gan>}"
echo "message=$MSG_T" | tee "$EV/b3-no-tol-event.txt"

test "$PHASE_T" = 'Pending' && test -z "$NODE_T" \
  && echo 'PASS: Pod ghim vao worker2 nhung khong co toleration thi nam Pending'
if echo "$MSG_T" | grep -qi 'taint'; then
  echo 'PASS: scheduler noi thang ly do la taint'
else
  echo "GHI NHAN: thong bao khong chua tu taint — chep nguyen van: $MSG_T"
fi
```

**PASS:** dòng `PASS: Pod ghim vao worker2 nhung khong co toleration…` xuất hiện; dòng thứ hai là
`PASS:` hoặc `GHI NHAN:` và nội dung đã vào evidence.

### B3.3. Thêm toleration khớp thì Pod chạy

```bash
cat > "$WK/taint-co-toleration.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: taint-with-tol
  namespace: lab-15
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  tolerations:
  - key: "os"
    operator: "Equal"
    value: "windows"
    effect: "NoSchedule"
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
YAML

kubectl apply -f "$WK/taint-co-toleration.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/taint-with-tol --timeout=180s

NODE_TT="$(kubectl -n "$NS" get pod taint-with-tol -o jsonpath='{.spec.nodeName}')"
echo "nodeName=$NODE_TT"
test "$NODE_TT" = "$W2" \
  && echo 'PASS: Pod co toleration khop duoc dat len chinh node dang mang taint'
```

**Ý nghĩa:** hai Pod chỉ khác nhau ở khối `tolerations` bốn dòng — lấy nguyên văn từ ví dụ của bài
176 — và kết quả ngược nhau. Ghép với B2: bài 176 nói Pod Windows cần **cả hai**, `nodeSelector`
**và** toleration. B2 chứng minh vế `nodeSelector`, B3 chứng minh vế toleration, và cả hai đều là
điều kiện cần chứ không phải điều kiện đủ.

Điểm quan trọng về mặt vận hành: taint bảo vệ node Windows khỏi **hệ sinh thái manifest Linux có
sẵn** mà bạn không sửa được. `nodeSelector` một mình không làm được việc đó, vì nó là thứ bạn phải
thêm vào từng Pod — chính là vấn đề mà bài 176 nêu.

**PASS:** `kubectl wait` thành công; dòng `PASS: Pod co toleration khop…` xuất hiện.

### B3.4. Gỡ taint và chứng minh worker2 sạch

```bash
kubectl -n "$NS" delete pod taint-no-tol taint-with-tol --wait=true --timeout=120s
kubectl taint node "$W2" os=windows:NoSchedule-

TAINT_SAU_B3="$(kubectl get node "$W2" -o jsonpath='{.spec.taints}')"
echo "taints cua $W2 sau khi go: ${TAINT_SAU_B3:-<rong>}" | tee "$EV/b3-taint-sau.txt"

test -z "$TAINT_SAU_B3" \
  && echo 'PASS: worker2 khong con taint nao — dung trang thai A5.4.3 quy dinh'

kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'
```

**PASS:** dòng `PASS: worker2 khong con taint nao…` xuất hiện; cột `TAINTS` của **cả hai worker** là
`<none>`; control plane **vẫn giữ** taint `node-role.kubernetes.io/control-plane:NoSchedule` — không
gỡ nó, gỡ là đổi hành vi lập lịch của mọi lab.

---
## B4. `.spec.os.name` — khai báo, không phải điều kiện lập lịch

**Mục đích:** kiểm chứng hai câu của bài [176](../176-windows-user-guide-vi.md) — *"Bộ lập lịch không
sử dụng giá trị của `.spec.os.name` khi gán Pod vào node"* và *"giá trị `.spec.os.name` không có tác
dụng đối với việc lập lịch các pod Windows"* — cùng câu kết của bài
[175](../175-windows-intro-vi.md): *"Nếu bất kỳ trường nào trong số này được chỉ định, Pod sẽ không
được API server chấp nhận."*

Hai câu đó nói về hai tầng khác nhau và mục này tách chúng ra bằng hai phép thử khác nhau: một ở
**tầng lập lịch**, một ở **tầng nhận manifest**.

### B4.1. Trường `.spec.os` có thật trong schema của cluster này

```bash
kubectl explain pod.spec.os | tee "$EV/b4-explain-os.txt"
kubectl explain pod.spec.os.name | tee -a "$EV/b4-explain-os.txt"
test -s "$EV/b4-explain-os.txt" && echo 'PASS: schema Pod cua cluster nay co truong .spec.os.name'
```

**PASS:** dòng `PASS: schema Pod cua cluster nay co truong .spec.os.name` xuất hiện.

### B4.2. `.spec.os.name: windows` **không** ngăn scheduler chọn node Linux

```bash
cat > "$WK/osname-windows.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: osname-windows
  namespace: lab-15
spec:
  restartPolicy: Never
  os:
    name: windows
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
YAML

kubectl apply -f "$WK/osname-windows.yaml"

for i in $(seq 1 24); do
  NODE_OS="$(kubectl -n "$NS" get pod osname-windows -o jsonpath='{.spec.nodeName}' 2>/dev/null)"
  test -n "$NODE_OS" && break
  kubectl -n "$NS" get events --field-selector involvedObject.name=osname-windows \
    -o name 2>/dev/null | grep -q . && break
  sleep 5
done

NODE_OS="$(kubectl -n "$NS" get pod osname-windows -o jsonpath='{.spec.nodeName}')"
PHASE_OS="$(kubectl -n "$NS" get pod osname-windows -o jsonpath='{.status.phase}')"
REASON_OS="$(kubectl -n "$NS" get pod osname-windows -o jsonpath='{.status.reason}')"
MSGP_OS="$(kubectl -n "$NS" get pod osname-windows -o jsonpath='{.status.message}')"
NODEOS_OS=''
test -n "$NODE_OS" && NODEOS_OS="$(kubectl get node "$NODE_OS" \
  -o jsonpath='{.status.nodeInfo.operatingSystem}')"

kubectl -n "$NS" describe pod osname-windows | tee "$EV/b4-osname-describe.txt"
{
  echo "nodeName        = ${NODE_OS:-<chua gan>}"
  echo "OS cua node do  = ${NODEOS_OS:-<khong xac dinh>}"
  echo "status.phase    = $PHASE_OS"
  echo "status.reason   = ${REASON_OS:-<rong>}"
  echo "status.message  = ${MSGP_OS:-<rong>}"
} | tee "$EV/b4-osname-ket-qua.txt"

if [ -n "$NODE_OS" ]; then
  echo 'PASS: scheduler VAN gan node cho Pod khai .spec.os.name=windows — no khong dung truong nay de loc node'
  test "$NODEOS_OS" = 'linux' \
    && echo 'PASS: node duoc chon chay linux — dung dieu bai 176 canh bao'
else
  echo 'GHI NHAN: Pod chua duoc gan node; chep nguyen van describe vao evidence roi doc phan Y nghia'
fi

case "$PHASE_OS" in
  Failed)  echo "GHI NHAN: Pod ket thuc o phase Failed voi reason='${REASON_OS:-<rong>}' — kubelet tu choi nhan Pod" ;;
  Running) echo 'GHI NHAN: Pod dang Running — chep describe vao evidence va doc phan Y nghia' ;;
  Pending) echo 'GHI NHAN: Pod van Pending — doc Event trong describe de biet no dung o dau' ;;
  *)       echo "GHI NHAN: phase=$PHASE_OS — chep nguyen van vao evidence" ;;
esac
```

**Ý nghĩa:** gate quan trọng nhất của bước này là **`spec.nodeName` có được điền hay không**. Nó
được điền nghĩa là scheduler đã chọn một node cho một Pod tự khai là Windows, trên một cluster không
có node Windows nào — đúng điều bài 176 nói: `.spec.os.name` **không** tham gia lọc node.

Số phận tiếp theo của Pod thuộc về **kubelet**, không thuộc về scheduler, và đó là lý do bước này ghi
lại `status.phase`, `status.reason`, `status.message` thay vì khẳng định trước. Kubelet là thành phần
duy nhất biết hệ điều hành thật của node nó đang chạy trên đó, nên nó là chốt chặn cuối. Chép nguyên
văn `reason` mà cluster của bạn trả về vào evidence — đó là chuỗi bạn sẽ gặp lại khi một Pod Windows
lạc sang node Linux trong cluster hỗn hợp thật.

Đây chính là bẫy mà bài 176 gọi tên: cái tên `os` khiến người ta tưởng nó là điều kiện lập lịch. Nó
không phải. Thứ lọc node là `nodeSelector` ở B2 và taint/toleration ở B3.

**PASS:** dòng `PASS: scheduler VAN gan node…` xuất hiện (hoặc một dòng `GHI NHAN:` kèm `describe`
đã vào evidence); đúng một dòng `GHI NHAN:` mô tả `phase` được in ra và đã chép vào evidence.

```bash
kubectl -n "$NS" delete pod osname-windows --wait=true --timeout=120s
```

### B4.3. `.spec.os.name: windows` khóa cứng một danh sách trường

Bài [175](../175-windows-intro-vi.md) liệt kê **hai mươi** trường không được đặt khi `.spec.os.name`
là `windows`. Mục này dựng một manifest cho mỗi trường — giống hệt nhau ngoại trừ đúng một trường —
rồi hỏi API server bằng `--dry-run=server`. Không object nào được tạo thật.

```bash
mkdir -p "$WK/oslock"

mk_oslock () {   # $1 = ten; $2 = doan YAML muc pod (thut 2); $3 = doan YAML muc container (thut 4)
  cat > "$WK/oslock/$1.yaml" <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: oslock-$1
  namespace: lab-15
spec:
  os:
    name: windows
  nodeSelector:
    kubernetes.io/os: windows
$2
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
$3
EOF
}

# Muoi truong o cap Pod
mk_oslock hostpid              '  hostPID: true' ''
mk_oslock hostipc              '  hostIPC: true' ''
mk_oslock shareprocessns       '  shareProcessNamespace: true' ''
mk_oslock selinuxoptions       '  securityContext:
    seLinuxOptions:
      level: "s0:c1,c2"' ''
mk_oslock seccompprofile       '  securityContext:
    seccompProfile:
      type: RuntimeDefault' ''
mk_oslock fsgroup              '  securityContext:
    fsGroup: 2000' ''
mk_oslock sysctls              '  securityContext:
    sysctls:
    - name: kernel.shm_rmid_forced
      value: "1"' ''
mk_oslock pod-runasuser        '  securityContext:
    runAsUser: 1000' ''
mk_oslock pod-runasgroup       '  securityContext:
    runAsGroup: 3000' ''
mk_oslock supplementalgroups   '  securityContext:
    supplementalGroups: [4000]' ''

# Bon truong o cap container
mk_oslock ctr-capabilities     '' '    securityContext:
      capabilities:
        drop: ["NET_RAW"]'
mk_oslock ctr-readonlyrootfs   '' '    securityContext:
      readOnlyRootFilesystem: true'
mk_oslock ctr-privileged       '' '    securityContext:
      privileged: true'
mk_oslock ctr-allowprivesc     '' '    securityContext:
      allowPrivilegeEscalation: false'

ls -1 "$WK/oslock" | wc -l
```

Hỏi API server từng cái một:

```bash
OSLOCK_N=0
OSLOCK_REJECT=0
: > "$EV/b4-oslock.txt"

for f in "$WK"/oslock/*.yaml; do
  n="$(basename "$f" .yaml)"
  OSLOCK_N=$(( OSLOCK_N + 1 ))
  if out="$(kubectl apply --dry-run=server -f "$f" 2>&1)"; then
    printf '%-22s : CHAP NHAN -> %s\n' "$n" "$(echo "$out" | tr '\n' ' ')" >> "$EV/b4-oslock.txt"
  else
    OSLOCK_REJECT=$(( OSLOCK_REJECT + 1 ))
    printf '%-22s : TU CHOI   -> %s\n' "$n" \
      "$(echo "$out" | head -3 | tr '\n' ' ')" >> "$EV/b4-oslock.txt"
  fi
done

cat "$EV/b4-oslock.txt"
echo "API server tu choi $OSLOCK_REJECT/$OSLOCK_N truong" | tee -a "$EV/b4-oslock.txt"

if [ "$OSLOCK_REJECT" -eq "$OSLOCK_N" ]; then
  echo 'PASS: API server tu choi tat ca cac truong ma bai 175 liet ke'
else
  echo "GHI NHAN: chi $OSLOCK_REJECT/$OSLOCK_N truong bi tu choi — doc b4-oslock.txt va ghi lai cai nao lot"
fi
```

Phép đối chứng — **bỏ đúng ba dòng khai `os`**, giữ nguyên mọi thứ còn lại:

```bash
sed '/^  os:$/,+1d' "$WK/oslock/pod-runasuser.yaml" \
  | sed 's/kubernetes.io\/os: windows/kubernetes.io\/os: linux/' \
  > "$WK/oslock-doi-chung.yaml"
cat "$WK/oslock-doi-chung.yaml"

if kubectl apply --dry-run=server -f "$WK/oslock-doi-chung.yaml" > "$EV/b4-doi-chung.txt" 2>&1; then
  echo 'PASS: cung manifest do, bo dong os.name thi API server CHAP NHAN — khac biet den tu dung .spec.os.name'
else
  echo 'GHI NHAN: manifest doi chung cung bi tu choi — doc b4-doi-chung.txt truoc khi ket luan'
  cat "$EV/b4-doi-chung.txt"
fi
```

**Ý nghĩa:** phép đối chứng mới là thứ biến bước này thành kiểm chứng. Nếu chỉ chạy vòng lặp và thấy
mọi manifest bị từ chối, bạn chưa biết nguyên nhân là `os.name` hay là một lỗi cú pháp chung. Bỏ
đúng hai dòng `os:`/`name: windows`, cùng manifest đó được nhận — khi ấy **`.spec.os.name` là biến
duy nhất còn lại**.

Rút ra: `.spec.os.name` không phải nhãn mô tả vô hại. Nó **thu hẹp tập trường hợp lệ của Pod**, và
đó là cơ chế Kubernetes dùng để chặn từ sớm những manifest chỉ có nghĩa trên Linux. Ghép với B4.2:
trường này **rất mạnh ở tầng nhận manifest** và **hoàn toàn vô hình ở tầng lập lịch**. Nhớ đúng hai
vế đó là hiểu xong `.spec.os.name`.

Sáu trường còn lại trong danh sách hai mươi của bài 175 —
`spec.securityContext.fsGroupChangePolicy`, `spec.containers[*].securityContext.seLinuxOptions`,
`spec.containers[*].securityContext.seccompProfile`,
`spec.containers[*].securityContext.procMount`, `spec.containers[*].securityContext.runAsUser`,
`spec.containers[*].securityContext.runAsGroup` — cùng cơ chế và cùng một câu kết luận; thêm chúng
vào vòng lặp trên bằng `mk_oslock` là bài tập tự làm nếu bạn muốn phủ hết hai mươi.

**PASS:** file `b4-oslock.txt` có đúng `OSLOCK_N` dòng kết quả cộng dòng tổng kết; một dòng `PASS:`
hoặc `GHI NHAN:` cho vòng lặp; một dòng `PASS:` hoặc `GHI NHAN:` cho phép đối chứng, và nội dung đã
vào evidence.

---

## B5. `securityContext`: ranh giới giữa Linux và Windows

**Mục đích:** bài [175](../175-windows-intro-vi.md) kết luận ở mục *Tương thích các trường trong
security context của Pod*: **chỉ `securityContext.runAsNonRoot` và `securityContext.windowsOptions`
là hoạt động trên Windows**. Bài [131](../131-windows-security-vi.md) bổ sung vế bảo mật: SELinux,
AppArmor, Seccomp và POSIX capability tùy chỉnh **không được hỗ trợ** trên node Windows. Mục này đo
cả hai phía của ranh giới đó từ một cluster toàn Linux.

### B5.1. `runAsUserName` trên cluster toàn Linux — ba tình huống `os.name`

Bài [278](../278-configure-runasusername-vi.md) đặt `runAsUserName` bên trong
`securityContext.windowsOptions`. Ba manifest dưới đây khác nhau **đúng phần khai `spec.os`**:

```bash
mkdir -p "$WK/runasusername"

cat > "$WK/runasusername/khong-khai-os.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: runasusername-noos
  namespace: lab-15
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/os: linux
  securityContext:
    windowsOptions:
      runAsUserName: "ContainerUser"
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
YAML

sed 's/^spec:$/spec:\n  os:\n    name: linux/; s/runasusername-noos/runasusername-oslinux/' \
  "$WK/runasusername/khong-khai-os.yaml" > "$WK/runasusername/os-linux.yaml"
sed 's/^spec:$/spec:\n  os:\n    name: windows/; s/runasusername-noos/runasusername-oswindows/; s/kubernetes.io\/os: linux/kubernetes.io\/os: windows/' \
  "$WK/runasusername/khong-khai-os.yaml" > "$WK/runasusername/os-windows.yaml"

grep -n 'name:\|os:\|runAsUserName' "$WK/runasusername/os-linux.yaml"
grep -n 'name:\|os:\|runAsUserName' "$WK/runasusername/os-windows.yaml"
```

Hỏi API server cả ba, **chỉ dry-run**, không tạo gì:

```bash
: > "$EV/b5-runasusername.txt"
for f in "$WK"/runasusername/*.yaml; do
  n="$(basename "$f" .yaml)"
  if out="$(kubectl apply --dry-run=server -f "$f" 2>&1)"; then
    printf '%-16s : CHAP NHAN -> %s\n' "$n" "$(echo "$out" | tr '\n' ' ')" >> "$EV/b5-runasusername.txt"
  else
    printf '%-16s : TU CHOI   -> %s\n' "$n" "$(echo "$out" | head -3 | tr '\n' ' ')" >> "$EV/b5-runasusername.txt"
  fi
done
cat "$EV/b5-runasusername.txt"

test "$(wc -l < "$EV/b5-runasusername.txt")" -eq 3 \
  && echo 'PASS: da ghi lai cau tra loi cua API server cho ca ba tinh huong os.name'
grep -qE 'CHAP NHAN|TU CHOI' "$EV/b5-runasusername.txt" \
  && echo 'PASS: moi dong deu roi vao dung mot trong hai lop CHAP NHAN / TU CHOI'
```

**Ý nghĩa của từng lớp kết quả** — đọc bảng này *sau khi* đã có kết quả thật, đừng đọc trước:

| Kết quả cho `khong-khai-os` | Nghĩa là |
| --- | --- |
| CHAP NHAN | API server **không** ràng buộc `windowsOptions` khi Pod không khai hệ điều hành. Trường được lưu vào object và đi tới kubelet; phần B5.1b bên dưới đo xem kubelet Linux làm gì với nó |
| TU CHOI | API server của cluster bạn ràng buộc chặt hơn. Chép nguyên văn thông báo vào evidence — đó là câu trả lời của **cluster bạn**, và checkpoint hỏi đúng câu đó |

| Kết quả cho `os-linux` | Nghĩa là |
| --- | --- |
| TU CHOI | Đúng đối xứng với B4.3: khai `os.name: linux` thì tập trường hợp lệ **loại bỏ phần Windows**, y như khai `windows` thì loại bỏ phần Linux. `.spec.os.name` cắt cả hai chiều |
| CHAP NHAN | Ghi lại; nó nghĩa là ràng buộc chỉ đi một chiều trên cluster bạn |

Nếu `khong-khai-os` được chấp nhận, chạy tiếp phần đo hành vi kubelet:

```bash
if kubectl apply --dry-run=server -f "$WK/runasusername/khong-khai-os.yaml" >/dev/null 2>&1; then
  kubectl apply -f "$WK/runasusername/khong-khai-os.yaml"
  kubectl -n "$NS" wait --for=condition=Ready pod/runasusername-noos --timeout=180s

  STORED="$(kubectl -n "$NS" get pod runasusername-noos \
    -o jsonpath='{.spec.securityContext.windowsOptions.runAsUserName}')"
  WHOAMI="$(kubectl -n "$NS" exec runasusername-noos -- id -un)"
  UIDNUM="$(kubectl -n "$NS" exec runasusername-noos -- id -u)"
  echo "runAsUserName luu trong object = ${STORED:-<rong>} | user that trong container = $WHOAMI (uid $UIDNUM)" \
    | tee "$EV/b5-runasusername-runtime.txt"

  test "$STORED" = 'ContainerUser' \
    && echo 'PASS: API server LUU nguyen gia tri runAsUserName vao object'
  test "$WHOAMI" != 'ContainerUser' \
    && echo 'PASS: tien trinh trong container KHONG chay duoi ContainerUser — kubelet Linux bo qua truong nay'
  kubectl -n "$NS" delete pod runasusername-noos --wait=true --timeout=120s
else
  echo 'GHI NHAN: API server tu choi manifest khong khai os — bo qua phan do hanh vi kubelet, ghi ly do vao evidence'
fi
```

**Ý nghĩa:** đây là bài học trung tâm của mục B5, và nó chỉ nhìn thấy được trên một cluster toàn
Linux. Object mang đầy đủ trường, `kubectl get -o yaml` in ra `runAsUserName: ContainerUser`, không
lỗi, không cảnh báo — **nhưng tiến trình bên trong container không chạy dưới user đó**. Trường này
chỉ được **container runtime của Windows** đọc; runtime Linux không có gì để ánh xạ nó tới, vì như
bài 175 nói ở mục *Tương thích API*, Linux định danh bằng **UID/GID số nguyên** còn Windows dùng
**SID nhị phân trong cơ sở dữ liệu SAM**.

Hệ quả vận hành: một manifest sai hệ điều hành có thể **im lặng chạy** trên cluster hỗn hợp. Không
có gì báo lỗi cho bạn ngoài chính hành vi sai của ứng dụng. Đó là lý do bài 176 khuyên khai
`.spec.os.name` cho **mọi** Pod — để tầng API bắt lỗi thay bạn, đúng như B4.3 đã đo.

**PASS:** hai dòng `PASS:` của phần dry-run xuất hiện; phần đo hành vi kubelet in ra hai dòng `PASS:`
hoặc một dòng `GHI NHAN:` kèm lý do đã vào evidence.

### B5.2. Ràng buộc định dạng `DOMAIN\USER` của bài 278

Bài 278 mục *Các giới hạn về username trên Windows* liệt kê ràng buộc rất cụ thể: `USER` tối đa 20
ký tự và **không được chứa** `" / \ [ ] : ; | = , + * ? < > @`. Ràng buộc này thuộc về **API
server**, nên nó đo được ngay trên cluster Linux.

```bash
mk_runas () {   # $1 = ten; $2 = gia tri runAsUserName (da escape cho YAML)
  cat > "$WK/runasusername/fmt-$1.yaml" <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: runasfmt-$1
  namespace: lab-15
spec:
  restartPolicy: Never
  securityContext:
    windowsOptions:
      runAsUserName: "$2"
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
EOF
}

mk_runas hop-le-1 'ContainerUser'
mk_runas hop-le-2 'NT AUTHORITY\\NETWORK SERVICE'
mk_runas vi-pham  'bad/user'
mk_runas rong     ''

: > "$EV/b5-runas-format.txt"
for f in "$WK"/runasusername/fmt-*.yaml; do
  n="$(basename "$f" .yaml)"
  if out="$(kubectl apply --dry-run=server -f "$f" 2>&1)"; then
    printf '%-16s : CHAP NHAN\n' "$n" >> "$EV/b5-runas-format.txt"
  else
    printf '%-16s : TU CHOI   -> %s\n' "$n" \
      "$(echo "$out" | head -2 | tr '\n' ' ')" >> "$EV/b5-runas-format.txt"
  fi
done
cat "$EV/b5-runas-format.txt"

test "$(wc -l < "$EV/b5-runas-format.txt")" -eq 4 \
  && echo 'PASS: da hoi API server du bon gia tri runAsUserName'
if grep -q '^fmt-vi-pham .*TU CHOI' "$EV/b5-runas-format.txt"; then
  echo 'PASS: gia tri chua dau / bi tu choi — dung rang buoc USER cua bai 278'
else
  echo 'GHI NHAN: gia tri chua dau / khong bi tu choi — chep ket qua vao evidence'
fi
```

**Ý nghĩa:** đây là phần của giai đoạn 15 mà cluster toàn Linux kiểm chứng được **trọn vẹn**, không
cần rẽ nhánh: quy tắc đặt tên `DOMAIN\USER` được cưỡng chế ở tầng API server, không phải ở node.
Chú ý sự đối lập với B5.1: cùng một trường, **định dạng** thì API server kiểm được ngay, còn **hiệu
lực** thì phải có node Windows mới thấy. Biết trường nào được kiểm ở tầng nào là biết lỗi sẽ hiện ra
lúc nào — lúc `kubectl apply`, hay ba tuần sau trong production.

**PASS:** dòng `PASS: da hoi API server du bon gia tri runAsUserName` xuất hiện; dòng thứ hai là
`PASS:` hoặc `GHI NHAN:` và nội dung đã vào evidence.

### B5.3. Secret trên node: tmpfs của Linux, đĩa cục bộ của Windows

Bài [131](../131-windows-security-vi.md) mở đầu bằng cảnh báo: *"Trên Windows, dữ liệu từ Secret
được ghi ra dưới dạng văn bản thuần trên bộ lưu trữ cục bộ của node (khác với việc dùng tmpfs /
filesystem trong bộ nhớ trên Linux)."* Vế Linux của câu đó đo được ngay:

```bash
kubectl -n "$NS" create secret generic lab-15-secret --from-literal=token=khong-phai-bi-mat-that

cat > "$WK/secret-tmpfs.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: secret-tmpfs
  namespace: lab-15
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
    volumeMounts:
    - name: sec
      mountPath: /etc/lab-secret
      readOnly: true
  volumes:
  - name: sec
    secret:
      secretName: lab-15-secret
YAML

kubectl apply -f "$WK/secret-tmpfs.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/secret-tmpfs --timeout=180s

SEC_UID="$(kubectl -n "$NS" get pod secret-tmpfs -o jsonpath='{.metadata.uid}')"
echo "pod uid = $SEC_UID"

# Nhin tu ben trong container
kubectl -n "$NS" exec secret-tmpfs -- sh -c 'grep " /etc/lab-secret " /proc/mounts' \
  | tee "$EV/b5-secret-mount-trong-container.txt"
grep -q 'tmpfs' "$EV/b5-secret-mount-trong-container.txt" \
  && echo 'PASS: trong container, thu muc Secret duoc mount bang tmpfs'

# Nhin tu chinh node
ssh "$W2" "sudo findmnt -n -o TARGET,FSTYPE -T /var/lib/kubelet/pods/$SEC_UID/volumes/kubernetes.io~secret/sec" \
  | tee "$EV/b5-secret-mount-tren-node.txt"
grep -q 'tmpfs' "$EV/b5-secret-mount-tren-node.txt" \
  && echo 'PASS: tren node, thu muc Secret cung la tmpfs — du lieu khong cham dia'
```

**Ý nghĩa:** hai phép nhìn cùng một chỗ từ hai phía cho cùng một câu trả lời `tmpfs` — filesystem
nằm trong bộ nhớ. Tắt node là mất, không có gì đọng lại trên đĩa. Đó là mặc định của Linux mà bạn
chưa từng phải nghĩ tới suốt giai đoạn 3 và 9.

Trên node Windows, **cùng object Secret đó** nằm dưới dạng văn bản thuần trên bộ lưu trữ cục bộ. Object
trong API không đổi, RBAC không đổi, `kubectl get secret` không đổi — chỉ mức bảo vệ trên node là
khác. Vì vậy bài 131 yêu cầu người vận hành bù bằng **cả hai** biện pháp: file ACL cho vị trí file,
và mã hóa cấp volume bằng BitLocker. Đây là ví dụ rõ nhất trong cả giai đoạn về việc *"cùng một API,
khác hệ điều hành, khác mô hình rủi ro"*.

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Nếu `ssh` tới `lab-k8s-worker2` không dùng được,
xem [mục 4](#4-troubleshooting-của-lab-này) — phép nhìn từ trong container vẫn đủ để gate, nhưng ghi
rõ ở checkpoint rằng bạn chỉ đo được một phía.

```bash
kubectl -n "$NS" delete pod secret-tmpfs --wait=true --timeout=120s
kubectl -n "$NS" delete secret lab-15-secret
```

### B5.4. Các cơ chế cô lập của Linux: có tác dụng thật ở đây, không có ở kia

Bài [175](../175-windows-intro-vi.md) và [131](../131-windows-security-vi.md) cùng chỉ ra một danh
sách. Bảng dưới ghi lại đúng những gì hai bài viết, cột cuối là chỗ mục này đo:

| Trường `securityContext` | Trên Windows, theo bài 175 và 131 | Đo ở đây |
| --- | --- | --- |
| `runAsUser` | không khả thi — Windows không có UID số nguyên; dùng `runAsUserName` thay thế | B5.4a |
| `runAsGroup` | không khả thi vì không có hỗ trợ GID | B5.4a |
| `fsGroup` | nằm trong danh sách trường bị cấm khi `os.name` là `windows`; bài 106 nói quyền theo uid/gid không khả dụng | B5.4a |
| `capabilities` | POSIX capability **chưa được triển khai** trên Windows | B5.4a |
| `readOnlyRootFilesystem` | không khả thi — registry và tiến trình hệ thống trong container **bắt buộc** ghi được | B5.4a |
| `privileged` | Windows **không hỗ trợ** container đặc quyền; thay bằng HostProcess container | B4.3 |
| `allowPrivilegeEscalation` | không khả thi — không có capability nào được kết nối | B4.3 |
| `seLinuxOptions` | SELinux là đặc thù của Linux | B4.3; xem [bảng lý do](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành) |
| `seccompProfile` | Seccomp không được hỗ trợ trên node Windows | B4.3 |
| `procMount` | Windows không có filesystem `/proc` | bài tập tự làm ở B4.3 |
| `runAsNonRoot` | **hoạt động** — nó ngăn container chạy dưới `ContainerAdministrator` | B5.4b |
| `windowsOptions` | **hoạt động** — đây là nửa Windows của security context | B5.1, B5.2 |

#### B5.4a. Năm trường đó có tác dụng thật trên Linux

```bash
cat > "$WK/sc-linux.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: sc-linux
  namespace: lab-15
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/os: linux
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
    securityContext:
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    emptyDir: {}
YAML

kubectl apply -f "$WK/sc-linux.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/sc-linux --timeout=180s

SC_UID="$(kubectl -n "$NS" exec sc-linux -- id -u)"
SC_GID="$(kubectl -n "$NS" exec sc-linux -- id -g)"
SC_VOLGID="$(kubectl -n "$NS" exec sc-linux -- stat -c '%g' /data)"
SC_CAPEFF="$(kubectl -n "$NS" exec sc-linux -- sh -c 'grep ^CapEff /proc/self/status' | awk '{print $2}')"
kubectl -n "$NS" exec sc-linux -- sh -c 'touch /probe-rootfs' >/dev/null 2>&1; SC_ROOTFS=$?
kubectl -n "$NS" exec sc-linux -- sh -c 'touch /data/probe-vol' >/dev/null 2>&1; SC_VOL=$?

{
  echo "runAsUser  -> id -u        = $SC_UID"
  echo "runAsGroup -> id -g        = $SC_GID"
  echo "fsGroup    -> gid cua /data= $SC_VOLGID"
  echo "capabilities drop ALL -> CapEff = $SC_CAPEFF"
  echo "readOnlyRootFilesystem -> ma thoat cua touch / = $SC_ROOTFS"
  echo "volume ghi duoc        -> ma thoat cua touch /data = $SC_VOL"
} | tee "$EV/b5-sc-linux.txt"

test "$SC_UID" = '1000'    && echo 'PASS: runAsUser co tac dung — tien trinh chay duoi uid 1000'
test "$SC_GID" = '3000'    && echo 'PASS: runAsGroup co tac dung — gid chinh la 3000'
test "$SC_VOLGID" = '2000' && echo 'PASS: fsGroup co tac dung — volume thuoc group 2000'
test "$SC_CAPEFF" = '0000000000000000' \
  && echo 'PASS: capabilities drop ALL co tac dung — CapEff bang 0'
test "$SC_ROOTFS" -ne 0    && echo 'PASS: readOnlyRootFilesystem co tac dung — khong ghi duoc vao /'
test "$SC_VOL" -eq 0       && echo 'PASS: volume duoc mount van ghi duoc — root FS chi doc khac volume chi doc'
```

**Ý nghĩa:** sáu phép đo, sáu con số. Không có "quan sát thấy" nào ở đây — mỗi trường được đọc ngược
lại từ bên trong container bằng một giá trị so sánh được. Đây là **nửa Linux** của ranh giới: những
trường này là công cụ hàng ngày của bạn từ giai đoạn 9.

Hai dòng cuối tách đúng chỗ mà bài [106](../106-windows-storage-vi.md) nhấn mạnh: *"Hệ thống file chỉ
đọc không được hỗ trợ … Tuy nhiên, volume chỉ đọc vẫn được hỗ trợ."* Trên Linux bạn có cả hai và vừa
đo được cả hai; trên Windows bạn chỉ còn vế thứ hai.

**PASS:** đủ sáu dòng `PASS:` của bước này.

#### B5.4b. Cùng manifest đó, thêm `os.name: windows`

```bash
sed 's/^spec:$/spec:\n  os:\n    name: windows/; s/name: sc-linux/name: sc-windows/; s/kubernetes.io\/os: linux/kubernetes.io\/os: windows/' \
  "$WK/sc-linux.yaml" > "$WK/sc-windows.yaml"
grep -n 'os:\|name:' "$WK/sc-windows.yaml" | head -8

if kubectl apply --dry-run=server -f "$WK/sc-windows.yaml" > "$EV/b5-sc-windows.txt" 2>&1; then
  echo 'GHI NHAN: API server CHAP NHAN manifest — chep b5-sc-windows.txt vao evidence va doc lai bai 175'
else
  echo 'PASS: API server TU CHOI cung manifest khi os.name la windows'
  head -5 "$EV/b5-sc-windows.txt"
fi

FIELD_HITS="$(grep -oiE 'runAsUser|runAsGroup|fsGroup|capabilities|readOnlyRootFilesystem' \
  "$EV/b5-sc-windows.txt" | sort -u | wc -l)"
echo "so ten truong xuat hien trong thong bao loi: $FIELD_HITS"
test "$FIELD_HITS" -ge 3 \
  && echo 'PASS: thong bao loi goi ten it nhat ba truong bi cam — API server noi ro cai gi sai' \
  || echo "GHI NHAN: thong bao chi goi ten $FIELD_HITS truong — chep nguyen van vao evidence"
```

**Ý nghĩa:** đây là câu trả lời trực tiếp cho câu hỏi *"những gì tôi đã dựng ở giai đoạn 9 còn tác
dụng bao nhiêu trên node Windows?"* mà bài 131 đặt ra. Cùng một manifest: trên Linux nó chạy và sáu
phép đo đều đúng; khai `os.name: windows` thì nó **không tới được node nào**, vì API server chặn từ
đầu. Không phải "chạy nhưng vô hiệu" — mà là "không được nhận".

Thứ còn lại ở phía Windows chỉ gồm những tầng không phụ thuộc kernel Linux: xác thực và phân quyền
của API server, RBAC, ServiceAccount, namespace, cộng `runAsNonRoot`, `windowsOptions`, và các cơ
chế riêng của Windows — `RunAsUserName`, GMSA, file ACL, BitLocker.

**PASS:** một dòng `PASS:` hoặc `GHI NHAN:` cho phản ứng của API server, và một dòng nữa cho số tên
trường trong thông báo lỗi; cả hai đã vào evidence.

```bash
kubectl -n "$NS" delete pod sc-linux --wait=true --timeout=120s
```

---

## B6. Quản lý tài nguyên: cgroup của Pod so với job object của container

**Mục đích:** bài [112](../112-windows-resource-management-vi.md) mở đầu bằng đúng phép so sánh này:
trên Linux **cgroup là ranh giới của pod**; trên Windows là **một job object cho mỗi container** cộng
bộ lọc namespace hệ thống. Mục này đo vế Linux của cả ba khác biệt mà bài nêu — ranh giới, bộ nhớ,
và tài nguyên dành riêng — để bạn biết chính xác mình mất gì khi chuyển sang node Windows.

[Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md) đã chứng minh container vượt memory limit bị kernel
OOM kill; mục này **không làm lại** phép thử đó.

### B6.1. Ba QoS class, đọc từ chính cluster

```bash
cat > "$WK/qos.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
  namespace: lab-15
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "64Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
  namespace: lab-15
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: "50m"
        memory: "32Mi"
      limits:
        memory: "64Mi"
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
  namespace: lab-15
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
YAML

kubectl apply -f "$WK/qos.yaml"
kubectl -n "$NS" wait --for=condition=Ready \
  pod/qos-guaranteed pod/qos-burstable pod/qos-besteffort --timeout=180s

kubectl -n "$NS" get pod -o custom-columns='POD:.metadata.name,QOS:.status.qosClass,NODE:.spec.nodeName' \
  | tee "$EV/b6-qos.txt"

test "$(kubectl -n "$NS" get pod qos-guaranteed -o jsonpath='{.status.qosClass}')" = 'Guaranteed' \
  && echo 'PASS: Pod requests bang limits duoc xep Guaranteed'
test "$(kubectl -n "$NS" get pod qos-burstable -o jsonpath='{.status.qosClass}')" = 'Burstable' \
  && echo 'PASS: Pod co requests khac limits duoc xep Burstable'
test "$(kubectl -n "$NS" get pod qos-besteffort -o jsonpath='{.status.qosClass}')" = 'BestEffort' \
  && echo 'PASS: Pod khong khai gi duoc xep BestEffort'
```

**Ý nghĩa:** ba class này là ngôn ngữ chung của API — chúng tồn tại bất kể node chạy hệ điều hành
nào. Cái **không** chung là ý nghĩa vận hành của chúng. Bài 112 nói Windows *"có thể giới hạn lượng
thời gian CPU … nhưng không thể bảo đảm một lượng thời gian CPU tối thiểu"*, và bài 175 nói trên
Windows `requests` chỉ dùng để **tránh cấp phát quá mức**, không dùng để bảo đảm tài nguyên. Nghĩa là
trên node Windows, nửa `limits` của Guaranteed còn nguyên, còn nửa `requests` thì mất phần "bảo
đảm" — Guaranteed ở đó không còn là lời hứa mà chỉ là một nhãn.

**PASS:** ba dòng `PASS:` của bước này xuất hiện; cả ba Pod nằm trên `lab-k8s-worker2`.

### B6.2. Ranh giới trên Linux là cgroup **của Pod**

```bash
G_UID="$(kubectl -n "$NS" get pod qos-guaranteed -o jsonpath='{.metadata.uid}')"
G_ESC="$(echo "$G_UID" | tr '-' '_')"
echo "pod uid = $G_UID"

SLICE="$(ssh "$W2" "sudo find /sys/fs/cgroup/kubepods.slice -maxdepth 3 -type d -name '*${G_ESC}*'" \
  | head -1)"
echo "cgroup cua Pod = ${SLICE:-<khong tim thay>}" | tee "$EV/b6-cgroup.txt"

test -n "$SLICE" && echo 'PASS: tim thay dung mot thu muc cgroup mang uid cua Pod'

SCOPE_N=0
POD_MAX=''
if [ -n "$SLICE" ]; then
  SCOPE_N="$(ssh "$W2" "sudo ls -d '${SLICE}'/*.scope 2>/dev/null | wc -l")"
  POD_MAX="$(ssh "$W2" "sudo cat '${SLICE}/memory.max' 2>/dev/null")"
  ssh "$W2" "sudo ls -1 '${SLICE}'" | tee -a "$EV/b6-cgroup.txt"
fi
{
  echo "so container scope nam trong cgroup cua Pod : $SCOPE_N"
  echo "memory.max o muc Pod                        : ${POD_MAX:-<khong doc duoc>}"
} | tee -a "$EV/b6-cgroup.txt"

test "$SCOPE_N" -ge 1 \
  && echo 'PASS: cac container cua Pod nam LONG BEN TRONG mot cgroup chung cua Pod'
test "$POD_MAX" = '67108864' \
  && echo 'PASS: memory.max o muc Pod bang dung 64Mi — gioi han duoc dat o RANH GIOI POD'
```

**Ý nghĩa:** đây là hình ảnh mà bài 112 dùng làm mốc so sánh. Trên Linux có **một** cgroup cho cả
Pod, `memory.max` được đặt ở đó, và mọi container — kể cả container hạ tầng "pause" mà bài 175 mô tả
— nằm bên trong nó. Ranh giới kiểm soát tài nguyên **là Pod**.

Trên Windows không có tầng đó. Bài 112 nói Windows dùng *"một job object cho mỗi container"*, nên
ranh giới lùi xuống **một cấp**: từng container tự chịu giới hạn của mình, không có ranh giới chung
cho cả Pod. Kèm theo là hai hệ quả bài nêu ngay sau: không thể chạy container Windows mà không có bộ
lọc namespace, nên **đặc quyền hệ thống không thực thi được trong ngữ cảnh host** và **container đặc
quyền không khả dụng**. Đó chính là dòng `privileged` mà B4.3 vừa thấy API server từ chối.

Và một bẫy từ vựng bài phải ghi chú riêng: **job object của Windows không liên quan gì tới Kubernetes
Job** của giai đoạn 4.

**PASS:** ba dòng `PASS:` của bước này xuất hiện. Nếu `POD_MAX` khác `67108864`, ghi con số thật vào
evidence và đọc [mục 4](#4-troubleshooting-của-lab-này) — cgroup driver hoặc đường dẫn cgroup của
node bạn khác baseline, và đó là điều cần báo chứ không phải cần vá.

### B6.3. `NodeAllocatable` được cưỡng chế, và `PIDPressure` tồn tại

Bài [175](../175-windows-intro-vi.md) mục *Các tùy chọn dòng lệnh cho kubelet* nói ba điều về node
Windows: `--enforce-node-allocable` **chưa được triển khai**, condition `PIDPressure` **chưa được
triển khai**, và kubelet **không thực hiện eviction do OOM**. Đo vế Linux của cả ba:

```bash
kubectl get --raw "/api/v1/nodes/$W2/proxy/configz" > "$EV/b6-configz.json" 2>/dev/null || true
test -s "$EV/b6-configz.json" && echo 'PASS: doc duoc configz cua kubelet tren worker2'

for k in enforceNodeAllocatable kubeReserved systemReserved evictionHard; do
  echo "--- $k ---"
  grep -o "\"$k\":[^,}]*[]}]*" "$EV/b6-configz.json" | head -1 || echo '(khong thay)'
done | tee "$EV/b6-kubelet-allocatable.txt"

grep -q 'enforceNodeAllocatable' "$EV/b6-configz.json" \
  && echo 'PASS: kubelet Linux CO khai enforceNodeAllocatable — co che cuong che nay ton tai o day'

PIDP="$(kubectl get node "$W2" -o jsonpath='{.status.conditions[?(@.type=="PIDPressure")].status}')"
MEMP="$(kubectl get node "$W2" -o jsonpath='{.status.conditions[?(@.type=="MemoryPressure")].status}')"
echo "PIDPressure=$PIDP | MemoryPressure=$MEMP" | tee -a "$EV/b6-kubelet-allocatable.txt"
test -n "$PIDP" && echo 'PASS: node Linux CO condition PIDPressure — thu ma node Windows chua co'
test "$PIDP" = 'False' && test "$MEMP" = 'False' \
  && echo 'PASS: node khong o trang thai ap luc — lab khong ep node vao ap luc nao'

kubectl get node "$W2" -o jsonpath='{.status.allocatable}{"\n"}' | tee -a "$EV/b6-kubelet-allocatable.txt"
```

**Ý nghĩa:** trên Linux, `--kube-reserved` và `--system-reserved` vừa **trừ vào `NodeAllocatable`**
vừa được **cưỡng chế** qua `enforceNodeAllocatable`. Bài 112 nói rõ trên Windows *"các giá trị này
chỉ được dùng để tính toán tài nguyên có thể cấp phát của node"* — tức chỉ còn vế kế toán, mất vế
cưỡng chế.

Ghép ba điều đó lại thì ra khuyến cáo mạnh nhất của bài 112: trên node Windows, thứ duy nhất còn tác
dụng thật là **limit đặt trên container**. *"Lập lịch các pod không có limit có thể khiến các node
Windows bị cấp phát quá mức và trong những trường hợp cực đoan có thể làm cho node trở nên không
lành mạnh."* Nguy hiểm hơn Linux vì không có OOM killer dọn dẹp giúp: bài 112 nói Windows *"không có
cơ chế kết thúc tiến trình khi hết bộ nhớ"*, tiến trình **chuyển trang xuống đĩa và chậm đi** thay vì
bị kill. Hỏng âm thầm, khó phát hiện hơn nhiều so với một Pod `OOMKilled`.

**PASS:** dòng `PASS: doc duoc configz…` cùng ba dòng `PASS:` còn lại của bước này xuất hiện; nếu
`configz` trả `403`, xem [mục 4](#4-troubleshooting-của-lab-này).

```bash
kubectl -n "$NS" delete pod qos-guaranteed qos-burstable qos-besteffort --wait=true --timeout=180s
```

---
## B7. Nhiều phiên bản Windows trong một cluster, RuntimeClass và HostProcess

**Mục đích:** bài [176](../176-windows-user-guide-vi.md) mục *Xử lý nhiều phiên bản Windows trong
cùng một cluster* nói `kubernetes.io/os` là **chưa đủ** khi cluster có cả Server 2022 lẫn 2025, và
mục *Đơn giản hóa với RuntimeClass* đưa ra cách gói `nodeSelector` cùng toleration lại một chỗ. Mục
này kiểm chứng cả hai trên cluster toàn Linux, cộng phần đo được của HostProcess Pod ở bài
[281](../281-create-hostprocess-pod-vi.md).

### B7.1. Label `node.kubernetes.io/windows-build`

```bash
kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,OS:.metadata.labels.kubernetes\.io/os,WINBUILD:.metadata.labels.node\.kubernetes\.io/windows-build' \
  | tee "$EV/b7-winbuild.txt"

test "$WINBUILD_N" -eq "$WIN_N" \
  && echo "PASS: so node mang label windows-build ($WINBUILD_N) khop so node Windows ($WIN_N)"
```

Bảng giá trị mà bài 176 quy định, dùng để viết `nodeSelector` khi cluster có nhiều phiên bản:

| Tên sản phẩm | Giá trị `node.kubernetes.io/windows-build` |
| --- | --- |
| Windows Server 2022 | `10.0.20348` |
| Windows Server 2025 | `10.0.26100` |

**Ý nghĩa:** label này **do Kubernetes tự thêm**, chỉ cho node Windows, và nó tồn tại vì một ràng
buộc cứng của Windows mà bài 175 nêu ở mục *Tương thích phiên bản hệ điều hành Windows*: *"phiên bản
hệ điều hành của host phải khớp với phiên bản hệ điều hành của image cơ sở của container"*. Trên
Linux không có ràng buộc tương đương — một image `busybox` chạy được trên mọi bản phân phối Linux có
kernel đủ mới — nên bạn chưa từng phải nghĩ tới nó.

Hệ quả: trong cluster hỗn hợp nhiều phiên bản, `nodeSelector` phải mang **hai** khóa, và Pod nào
thiếu khóa thứ hai sẽ rơi vào node sai phiên bản rồi báo `ErrImgPull`/`ImagePullBackOff` — đúng triệu
chứng số 2 ở mục *Khắc phục sự cố cấp node* của bài [315](../315-debug-windows-vi.md).

**PASS:** dòng `PASS: so node mang label windows-build…` xuất hiện. Trên cluster toàn Linux cả hai
con số bằng 0, và đó là kết quả đúng.

### B7.2. RuntimeClass đóng gói `nodeSelector` và toleration

Tạo RuntimeClass theo đúng khuôn của bài 176, chỉ đổi tên và giá trị build cho khớp bảng trên:

```bash
cat > "$WK/runtimeclass.yaml" <<'YAML'
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: lab-15-windows
handler: example-container-runtime-handler
scheduling:
  nodeSelector:
    kubernetes.io/os: 'windows'
    kubernetes.io/arch: 'amd64'
    node.kubernetes.io/windows-build: '10.0.20348'
  tolerations:
  - effect: NoSchedule
    key: os
    operator: Equal
    value: "windows"
YAML

kubectl apply -f "$WK/runtimeclass.yaml"
kubectl get runtimeclass lab-15-windows -o yaml | tee "$EV/b7-runtimeclass.yaml"

RC_SEL="$(kubectl get runtimeclass lab-15-windows \
  -o jsonpath='{.scheduling.nodeSelector.kubernetes\.io/os}')"
RC_TOL="$(kubectl get runtimeclass lab-15-windows \
  -o jsonpath='{.scheduling.tolerations[0].key}')"
echo "nodeSelector.kubernetes.io/os=$RC_SEL | tolerations[0].key=$RC_TOL"

test "$RC_SEL" = 'windows' && echo 'PASS: RuntimeClass giu nguyen nodeSelector danh cho Windows'
test "$RC_TOL" = 'os'      && echo 'PASS: RuntimeClass giu nguyen toleration khop taint os=windows'
```

Pod chỉ khai đúng một dòng `runtimeClassName`, không khai `nodeSelector`, không khai `tolerations`:

```bash
cat > "$WK/pod-runtimeclass.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: rc-consumer
  namespace: lab-15
spec:
  restartPolicy: Never
  runtimeClassName: lab-15-windows
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
YAML

kubectl apply -f "$WK/pod-runtimeclass.yaml"

for i in $(seq 1 24); do
  kubectl -n "$NS" get events --field-selector involvedObject.name=rc-consumer \
    -o name 2>/dev/null | grep -q . && break
  sleep 5
done

POD_SEL="$(kubectl -n "$NS" get pod rc-consumer -o jsonpath='{.spec.nodeSelector}')"
POD_TOL="$(kubectl -n "$NS" get pod rc-consumer -o jsonpath='{.spec.tolerations[*].key}')"
POD_PH="$(kubectl -n "$NS" get pod rc-consumer -o jsonpath='{.status.phase}')"
POD_ND="$(kubectl -n "$NS" get pod rc-consumer -o jsonpath='{.spec.nodeName}')"
{
  echo "nodeSelector duoc tiem vao Pod : ${POD_SEL:-<rong>}"
  echo "toleration keys tren Pod       : ${POD_TOL:-<rong>}"
  echo "phase                          : $POD_PH"
  echo "nodeName                       : ${POD_ND:-<chua gan>}"
} | tee "$EV/b7-rc-consumer.txt"

if echo "$POD_SEL" | grep -q 'windows'; then
  echo 'PASS: admission da TIEM nodeSelector cua RuntimeClass vao Pod — Pod khong he khai dong nao'
else
  echo 'GHI NHAN: Pod khong duoc tiem nodeSelector — chep b7-rc-consumer.txt vao evidence'
fi
echo "$POD_TOL" | grep -qw 'os' \
  && echo 'PASS: toleration cua RuntimeClass cung duoc tiem vao Pod'
test "$POD_PH" = 'Pending' && test -z "$POD_ND" \
  && echo 'PASS: Pod nam Pending vi khong node nao khop — dung ket qua tren cluster toan Linux'
```

**Ý nghĩa:** Pod khai đúng **một** dòng, và trong object đã tạo lại có cả `nodeSelector` ba khóa lẫn
toleration. Đó chính là điều bài 176 gọi là "đơn giản hóa": quản trị viên cluster định nghĩa
RuntimeClass một lần, người triển khai workload chỉ cần biết tên nó. Không ai phải nhớ chuỗi
`10.0.20348`, và không manifest nào bị quên toleration.

Đây là kiểm chứng thật chứ không phải mô phỏng: cơ chế tiêm này là của **admission**, hoàn toàn độc
lập với việc cluster có node Windows hay không. Bạn vừa thấy nó chạy trên cluster của mình.

Chú ý trường `handler`: nó trỏ tới một handler mà container runtime phải hiểu. Pod này không bao giờ
được lập lịch nên handler không bao giờ bị gọi tới — trên cluster có node Windows thật, giá trị đó
phải khớp với handler mà containerd trên node Windows khai báo.

**PASS:** hai dòng `PASS:` của phần RuntimeClass, cộng ba dòng `PASS:`/`GHI NHAN:` của phần Pod;
`b7-rc-consumer.txt` có đủ bốn dòng.

```bash
kubectl -n "$NS" delete pod rc-consumer --wait=true --timeout=120s
```

### B7.3. HostProcess Pod: bốn điều khiển trong bảng của bài 281

Bài [281](../281-create-hostprocess-pod-vi.md) mục *Yêu cầu cấu hình cho HostProcess Pod* liệt kê
đúng bốn điều khiển: `securityContext.windowsOptions.hostProcess` phải là `true`, `hostNetwork` phải
là `true`, `securityContext.windowsOptions.runAsUserName` **bắt buộc** phải chỉ định, và
`runAsNonRoot` **không được** là `true`.

```bash
kubectl explain pod.spec.securityContext.windowsOptions.hostProcess | tee "$EV/b7-explain-hostprocess.txt"
test -s "$EV/b7-explain-hostprocess.txt" \
  && echo 'PASS: schema Pod cua cluster nay co truong windowsOptions.hostProcess'
```

Ba manifest: một bản **đúng theo trích đoạn của bài**, hai bản vi phạm đúng một điều kiện:

```bash
mkdir -p "$WK/hostprocess"

cat > "$WK/hostprocess/dung-bai.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: hpc-dung-bai
  namespace: lab-15
spec:
  securityContext:
    windowsOptions:
      hostProcess: true
      runAsUserName: "NT AUTHORITY\\Local service"
  hostNetwork: true
  containers:
  - name: test
    image: busybox:1.37
    command:
      - ping
      - -t
      - 127.0.0.1
  nodeSelector:
    "kubernetes.io/os": windows
YAML

cat > "$WK/hostprocess/thieu-hostnetwork.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: hpc-thieu-hostnetwork
  namespace: lab-15
spec:
  securityContext:
    windowsOptions:
      hostProcess: true
      runAsUserName: "NT AUTHORITY\\Local service"
  containers:
  - name: test
    image: busybox:1.37
    command:
      - ping
      - -t
      - 127.0.0.1
  nodeSelector:
    "kubernetes.io/os": windows
YAML

cat > "$WK/hostprocess/runasnonroot.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: hpc-runasnonroot
  namespace: lab-15
spec:
  securityContext:
    runAsNonRoot: true
    windowsOptions:
      hostProcess: true
      runAsUserName: "NT AUTHORITY\\Local service"
  hostNetwork: true
  containers:
  - name: test
    image: busybox:1.37
    command:
      - ping
      - -t
      - 127.0.0.1
  nodeSelector:
    "kubernetes.io/os": windows
YAML

grep -n 'hostNetwork\|runAsNonRoot\|hostProcess' "$WK/hostprocess/thieu-hostnetwork.yaml"
grep -n 'hostNetwork\|runAsNonRoot\|hostProcess' "$WK/hostprocess/runasnonroot.yaml"

: > "$EV/b7-hostprocess.txt"
for f in "$WK"/hostprocess/*.yaml; do
  n="$(basename "$f" .yaml)"
  if out="$(kubectl apply --dry-run=server -f "$f" 2>&1)"; then
    printf '%-20s : CHAP NHAN\n' "$n" >> "$EV/b7-hostprocess.txt"
  else
    printf '%-20s : TU CHOI   -> %s\n' "$n" \
      "$(echo "$out" | head -3 | tr '\n' ' ')" >> "$EV/b7-hostprocess.txt"
  fi
done
cat "$EV/b7-hostprocess.txt"

test "$(wc -l < "$EV/b7-hostprocess.txt")" -eq 3 \
  && echo 'PASS: da hoi API server du ba bien the manifest HostProcess'
grep -qE 'CHAP NHAN|TU CHOI' "$EV/b7-hostprocess.txt" \
  && echo 'PASS: moi dong roi vao dung mot trong hai lop CHAP NHAN / TU CHOI'
```

**Ý nghĩa:** ba dòng kết quả trong `b7-hostprocess.txt` cho biết **bao nhiêu phần của bảng yêu cầu ở
bài 281 được API server cưỡng chế** trên cluster của bạn, và bao nhiêu phần chỉ là quy ước mà node
Windows mới kiểm. Đó là câu hỏi vận hành thật: lỗi hiện ra lúc `kubectl apply`, hay lúc Pod đã lên
node?

Chép nguyên văn ba dòng đó vào evidence. Đừng suy ra kết luận trước khi chạy — và đừng sửa manifest
cho khớp một kết quả mong đợi.

Ba điều còn lại của bài 281 **không đo được ở đây** và đã ghi rõ ở
[bảng lý do](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành): HostProcess pod chỉ chứa được HostProcess
container; nó chạy như tiến trình trên host Windows dưới `LocalSystem`, `LocalService`,
`NetworkService` hoặc một local usergroup; và nó cần containerd 1.6 trở lên, khuyến nghị 1.7. Cả ba
đều là thuộc tính của node Windows.

Điều đáng nhớ nhất cho checkpoint: bài 281 xếp HostProcess pod vào **profile privileged** của
[Pod Security Standards](../115-pod-security-standards-vi.md) — chính sách `baseline` và `restricted`
**cấm** nó. Đó là mối nối trực tiếp với [Lab 9b](LAB-9B-POD-SECURITY-VA-HARDENING.md): một namespace
đang bật `enforce=restricted` sẽ chặn HostProcess pod, bất kể node là Windows hay Linux.

**PASS:** dòng `PASS: schema Pod…hostProcess` cùng hai dòng `PASS:` của vòng lặp xuất hiện;
`b7-hostprocess.txt` có đúng ba dòng.

---

## B8. Debug: quy trình Linux đã quen và những gì đổi trên Windows

**Mục đích:** bài [315](../315-debug-windows-vi.md) là danh mục sự cố của node Windows. Phần lớn cần
node Windows để chạy, nhưng **nguyên nhân gốc** của ba mục quan trọng nhất đo được ngay trên Linux.
Mục này đo chúng, rồi đối chiếu từng bước với quy trình debug bạn đã dùng từ Lab 3a tới Lab 12.

### B8.1. `kubectl logs` chỉ thấy STDOUT

Bài [176](../176-windows-user-guide-vi.md) mục *Thu thập log từ các workload* nêu vấn đề: workload
Windows *"thường được cấu hình để ghi log vào ETW … hoặc đẩy các mục log vào event log của ứng
dụng"*, nên `kubectl logs <pod>` không đọc được. Nguyên nhân gốc không phải Windows — mà là
**`kubectl logs` chỉ đọc STDOUT của container**. Chứng minh bằng một Pod hai container:

```bash
cat > "$WK/log-duong-di.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: log-duong-di
  namespace: lab-15
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/os: linux
  containers:
  - name: ra-stdout
    image: busybox:1.37
    command: ["sh", "-c", "while true; do echo LAB15-STDOUT-OK; sleep 5; done"]
  - name: ra-file
    image: busybox:1.37
    command: ["sh", "-c", "mkdir -p /var/log/app; while true; do echo LAB15-FILE-OK >> /var/log/app/app.log; sleep 5; done"]
YAML

kubectl apply -f "$WK/log-duong-di.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/log-duong-di --timeout=180s

for i in $(seq 1 12); do
  kubectl -n "$NS" logs log-duong-di -c ra-stdout 2>/dev/null | grep -q 'LAB15-STDOUT-OK' && break
  sleep 5
done

STDOUT_N="$(kubectl -n "$NS" logs log-duong-di -c ra-stdout | grep -c 'LAB15-STDOUT-OK' || true)"
FILELOG_N="$(kubectl -n "$NS" logs log-duong-di -c ra-file | wc -l)"
INFILE_N="$(kubectl -n "$NS" exec log-duong-di -c ra-file -- sh -c 'wc -l < /var/log/app/app.log')"
{
  echo "kubectl logs -c ra-stdout : $STDOUT_N dong khop LAB15-STDOUT-OK"
  echo "kubectl logs -c ra-file   : $FILELOG_N dong"
  echo "file /var/log/app/app.log : $INFILE_N dong"
} | tee "$EV/b8-logs.txt"

test "$STDOUT_N" -ge 1 && echo 'PASS: container ghi ra STDOUT thi kubectl logs doc duoc'
test "$FILELOG_N" -eq 0 && echo 'PASS: container ghi vao file thi kubectl logs KHONG doc duoc gi'
test "$INFILE_N" -ge 1 \
  && echo 'PASS: container do VAN dang ghi log day du — van de nam o duong di cua log, khong o ung dung'
```

**Ý nghĩa:** ba con số này là toàn bộ câu chuyện. Container thứ hai ghi log đều đặn, file có nội
dung, ứng dụng khỏe mạnh — và `kubectl logs` trả về rỗng. Không có lỗi nào để đọc, không có Event
nào, không có `RESTARTS` tăng. Đây là kiểu hỏng khó chịu nhất: **im lặng**.

Trên Windows, tình huống này là **mặc định** chứ không phải trường hợp hiếm, vì ETW và event log là
nơi ứng dụng Windows ghi log theo thói quen. Đó là lý do bài 176 khuyến nghị **LogMonitor**: nó
không sửa ứng dụng, nó chỉ **chuyển tiếp** event log và ETW provider ra STDOUT để `kubectl logs` đọc
được. Hiểu đúng nguyên nhân gốc rồi thì giải pháp trở nên hiển nhiên.

Bài học chuyển được sang Linux: một sidecar chạy agent gom log rồi đẩy đi nơi khác cũng làm
`kubectl logs` của container chính im lặng y hệt.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

```bash
kubectl -n "$NS" delete pod log-duong-di --wait=true --timeout=120s
```

### B8.2. Pause image trên node

Triệu chứng số 1 ở mục *Khắc phục sự cố cấp node* của bài 315 — Pod kẹt ở "Container Creating" hoặc
restart lặp lại — quy về **pause image không tương thích phiên bản Windows**, và bài chỉ đúng chỗ
khai báo nó với containerd. Xem node Linux của bạn đang có gì:

```bash
ssh "$W2" 'sudo crictl images 2>/dev/null | grep -i pause' | tee "$EV/b8-pause-image.txt"

if ssh "$W2" "sudo grep -n 'sandbox' /etc/containerd/config.toml" > "$WK/sandbox.txt" 2>/dev/null \
   && [ -s "$WK/sandbox.txt" ]; then
  cat "$WK/sandbox.txt" | tee -a "$EV/b8-pause-image.txt"
else
  echo '(config.toml khong khai sandbox image — containerd dung gia tri mac dinh)' \
    | tee -a "$EV/b8-pause-image.txt"
fi
rm -f "$WK/sandbox.txt"

test -s "$EV/b8-pause-image.txt" && echo 'PASS: da ghi lai tinh trang pause image cua node'
grep -qi 'pause' "$EV/b8-pause-image.txt" \
  && echo 'PASS: node co pause image trong kho image cuc bo' \
  || echo 'GHI NHAN: khong thay pause image trong crictl images — chep nguyen van vao evidence'
```

**Ý nghĩa:** bài [175](../175-windows-intro-vi.md) mục *Pause container* giải thích vì sao vật này
tồn tại: trên Linux, cgroup và namespace của Pod cần **một tiến trình** giữ cho chúng sống, và tiến
trình pause làm việc đó; mọi container trong Pod chia sẻ một điểm cuối mạng, nên container ứng dụng
crash hay restart mà cấu hình mạng không mất. B6.2 đã nhìn thấy chính nó — nó là một trong những
`*.scope` nằm trong cgroup của Pod.

Khác biệt về vận hành: trên Linux, pause image được kéo một lần rồi dùng chung cho mọi Pod và bạn
gần như không bao giờ phải nghĩ tới. Trên Windows, nó nằm trong ràng buộc **phiên bản host phải khớp
phiên bản base image** ở B7.1 — nên nó trở thành nguyên nhân sự cố hàng đầu, và bài 315 xếp nó ở
mục số 1. Dự án Kubernetes còn khuyến nghị dùng bản do Microsoft duy trì cho môi trường yêu cầu file
nhị phân được ký authenticode.

**PASS:** hai dòng `PASS:` (hoặc một `PASS:` cộng một `GHI NHAN:`) của bước này xuất hiện.

### B8.3. `kubectl port-forward` và bảng đối chiếu quy trình debug

Triệu chứng số 8 của bài 315 là `kubectl port-forward` thất bại vì thiếu `wincat` trong pause
container. Kiểm chứng vế Linux, dùng lại đúng kỹ thuật của
[tầng 6 gate A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready):

```bash
kubectl -n "$NS" create deployment web-lab15 --image=busybox:1.37 --replicas=2 \
  -- sh -c 'mkdir -p /www && echo lab15-web-ok > /www/index.html && httpd -f -p 8080 -h /www'
kubectl -n "$NS" rollout status deploy/web-lab15 --timeout=180s
kubectl -n "$NS" expose deploy web-lab15 --name=web-clusterip --port=80 --target-port=8080

kubectl -n "$NS" port-forward svc/web-clusterip 18150:80 >/tmp/lab15-pf.log 2>&1 &
PF_PID=$!
for i in $(seq 1 20); do
  curl -fsS http://127.0.0.1:18150 >/dev/null 2>&1 && break
  sleep 1
done
curl -s http://127.0.0.1:18150 | grep -qx 'lab15-web-ok' \
  && echo 'PASS: kubectl port-forward chay tren cluster nay'
kill "$PF_PID" 2>/dev/null
rm -f /tmp/lab15-pf.log

test "$(kubectl -n "$NS" exec deploy/web-lab15 -- hostname)" != '' \
  && echo 'PASS: kubectl exec chay tren cluster nay'
```

Bảng đối chiếu — cột giữa là bước bạn đã dùng suốt các lab trước, cột phải là thứ thay thế nó trên
node Windows theo bài 315:

| Câu hỏi khi debug | Trên node Linux (đã dùng từ Lab 3a) | Trên node Windows (bài 315) |
| --- | --- | --- |
| Ứng dụng in ra gì? | `kubectl logs` — đọc STDOUT | `kubectl logs` **rỗng** nếu log đi vào ETW hoặc event log; cần LogMonitor chuyển tiếp ra STDOUT |
| Pod kẹt ở `ContainerCreating` | `kubectl describe pod`, đọc Event của kubelet | Kiểm **pause image có tương thích phiên bản Windows** không; với containerd, xem `plugins.plugins.cri.sandbox_image` trong `config.toml` |
| `ErrImagePull` / `ImagePullBackOff` | sai tên image, thiếu credential, mất egress | Pod bị lập lịch lên node Windows **không khớp phiên bản** — quay về `nodeSelector` với `node.kubernetes.io/windows-build` ở B7.1 |
| Pod không có mạng | Pod IP, CNI Pod trên node, sysctl, module kernel | Trên máy ảo phải **bật MAC spoofing** trên mọi adapter mạng của VM |
| Kiểm kết nối ra ngoài | `ping` rồi `curl` | **Không dùng `ping`**: Pod Windows không có rule outbound cho ICMP. Dùng `curl`, vì TCP/UDP được hỗ trợ |
| `curl` ra ngoài vẫn hỏng | firewall, DNS, egress của LAN | Xem `ExceptionList` trong `cni.conf` theo hướng **bớt**: IP đích phải **không** nằm trong danh sách thì traffic mới được SNAT đúng |
| Gọi NodePort ngay trên node | chạy được — B9.2 đo | **Giới hạn đã biết**: từ chính node đó thì hỏng; từ node khác hoặc client ngoài thì được |
| Gọi Service IP từ chính node | chạy được | **Giới hạn đã biết**: node không truy cập được service IP; Pod Windows trên node đó thì được |
| Mở shell vào container | `kubectl exec` | `kubectl exec … powershell` — bài 278 và 273 đều dùng đường này |
| Chuyển tiếp cổng để test | `kubectl port-forward` | Cần `wincat` trong pause container; thiếu thì báo `wincat not found` |
| Node vừa join lại mà mạng hỏng | kiểm PodCIDR, CNI Pod | Với Flannel: xóa `C:\k\SourceVip.json` và `C:\k\SourceVipRequest.json` |
| Pod không khởi chạy được | đọc `describe` và log kubelet | Thiếu `/run/flannel/subnet.env` nghĩa là flanneld chưa chạy đúng |

**Ý nghĩa:** đọc bảng theo cột, không theo hàng, thì thấy quy luật: **những bước dựa vào API của
Kubernetes giữ nguyên** — `describe`, Event, `exec`, `nodeSelector`; **những bước dựa vào hành vi của
hệ điều hành thì đổi** — đường đi của log, ICMP, NodePort cục bộ, port-forward. Đó cũng đúng là ranh
giới mà cả giai đoạn 15 vẽ ra.

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Deployment `web-lab15` và Service `web-clusterip`
**giữ lại** cho B9; chúng bị xóa cùng namespace ở B12.

---

## B9. Mạng: bốn kiểu Service và ranh giới của bài 89

**Mục đích:** bài [89](../89-windows-networking-vi.md) mục *Cân bằng tải và Service* liệt kê bốn kiểu
Service dùng được trong cluster có node Windows, và mục *Hạn chế* liệt kê những gì không chạy. Mục
này kiểm chứng phần đo được trên Linux và ghi rõ phần không đo được.

### B9.1. Bốn kiểu Service

```bash
kubectl -n "$NS" expose deploy web-lab15 --name=web-nodeport --type=NodePort \
  --port=80 --target-port=8080
kubectl -n "$NS" create service loadbalancer web-lb --tcp=80:8080
kubectl -n "$NS" create service externalname web-ext --external-name=example.internal

kubectl -n "$NS" get svc -o custom-columns='SVC:.metadata.name,TYPE:.spec.type,CLUSTERIP:.spec.clusterIP' \
  | tee "$EV/b9-services.txt"

TYPES="$(kubectl -n "$NS" get svc -o jsonpath='{range .items[*]}{.spec.type}{"\n"}{end}' | sort -u | tr '\n' ' ')"
echo "cac kieu Service dang ton tai: $TYPES"
for t in ClusterIP NodePort LoadBalancer ExternalName; do
  echo "$TYPES" | grep -qw "$t" && echo "PASS: cluster nhan Service kieu $t"
done
```

**Ý nghĩa:** bốn kiểu này là bốn kiểu bài 89 nói dùng được trên cluster có node Windows — nói cách
khác, **thêm node Windows không cắt mất kiểu Service nào**. Service `web-lb` sẽ đứng mãi ở
`EXTERNAL-IP: <pending>` vì cluster lab không có bộ cấp phát IP ngoài; đó là kết quả đúng và đã biết
từ [Lab 5a](LAB-5A-SERVICE-ENDPOINTSLICE-VA-DNS.md#b6-nodeport-externalname-và-loadbalancer), không
phải lỗi của lab này.

**PASS:** bốn dòng `PASS: cluster nhan Service kieu …` xuất hiện.

### B9.2. NodePort gọi từ chính node — chạy trên Linux, không chạy trên Windows

```bash
NP="$(kubectl -n "$NS" get svc web-nodeport -o jsonpath='{.spec.ports[0].nodePort}')"
echo "nodePort = $NP"

RC_MASTER="$(curl -s -o /dev/null -w '%{http_code}' "http://127.0.0.1:$NP")"
RC_W2_LOCAL="$(ssh "$W2" "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:$NP")"
W2_IP="$(kubectl get node "$W2" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
RC_W2_FROM_MASTER="$(curl -s -o /dev/null -w '%{http_code}' "http://$W2_IP:$NP")"
{
  echo "tu chinh master, goi 127.0.0.1:$NP          -> $RC_MASTER"
  echo "tu chinh worker2, goi 127.0.0.1:$NP         -> $RC_W2_LOCAL"
  echo "tu master, goi IP cua worker2 :$NP          -> $RC_W2_FROM_MASTER"
} | tee "$EV/b9-nodeport.txt"

test "$RC_MASTER" = '200' \
  && echo 'PASS: goi NodePort ngay tren chinh node do CHAY duoc — day la node Linux'
test "$RC_W2_LOCAL" = '200' \
  && echo 'PASS: worker2 cung goi duoc NodePort cua chinh no'
test "$RC_W2_FROM_MASTER" = '200' \
  && echo 'PASS: goi NodePort tu node khac cung chay'
```

**Ý nghĩa:** ba con số `200`. Trên node Windows, **con số đầu và con số thứ hai sẽ hỏng**: mục *Hạn
chế* của bài 89 ghi *"truy cập NodePort cục bộ từ chính node đó"* trong danh sách không được hỗ trợ,
và nói thêm trong ngoặc rằng nó *"vẫn hoạt động với các node khác hoặc client bên ngoài"*. Bài 315
lặp lại y hệt ở triệu chứng số 3 và gọi đó là **giới hạn đã biết**.

Vì sao điều này quan trọng với người vận hành: nó biến một **phép thử** thành một **báo động giả**.
Bạn ssh vào node Windows, `curl` NodePort của chính nó, thấy hỏng, và kết luận Service hỏng — trong
khi Service hoàn toàn khỏe. Đứng đúng chỗ để thử là một phần của kỹ năng debug, và đây là ví dụ
điển hình nhất.

**PASS:** ba dòng `PASS:` của bước này xuất hiện. Nếu `ssh` tới worker2 không dùng được, bỏ con số
thứ hai và ghi rõ ở checkpoint; hai con số còn lại vẫn đủ để gate.

### B9.3. Cấu hình DNS: file trên Linux, registry trên Windows

```bash
kubectl -n "$NS" exec deploy/web-lab15 -- cat /etc/resolv.conf | tee "$EV/b9-resolv.txt"
DNSIP="$(kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.clusterIP}')"
echo "ClusterIP cua kube-dns = $DNSIP"

grep -q "$DNSIP" "$EV/b9-resolv.txt" \
  && echo 'PASS: /etc/resolv.conf trong Pod tro dung ClusterIP cua CoreDNS'
grep -q 'search' "$EV/b9-resolv.txt" \
  && echo 'PASS: /etc/resolv.conf co dong search — day la mot FILE, do kubelet ghi vao Pod'
```

**Ý nghĩa:** trên Linux, cấu hình DNS của Pod là **một file** mà kubelet ghi vào và CNI có thể tác
động tới. Bài 89 mục *Mạng container trên Windows* nói vế còn lại: trên Windows, DNS, route và metric
nằm trong **cơ sở dữ liệu registry**, không phải file trong `/etc`; hơn nữa **registry của container
tách biệt với registry của host**, nên "ánh xạ `/etc/resolv.conf` từ host vào container" không có
cùng tác dụng. Hệ quả trực tiếp: các hiện thực **CNI phải gọi HNS** thay vì dựa vào ánh xạ file.

Đó là khác biệt **ở gốc**, không phải thiếu sót của một plugin cụ thể — và nó giải thích vì sao
không thể lấy một CNI Linux rồi biên dịch lại cho Windows, mà phải có win-bridge, win-overlay hay
Azure-CNI. Về mặt mạng, bài mô tả container Windows *"hoạt động tương tự như máy ảo"*: mỗi container
có một **vNIC** gắn vào một **vSwitch Hyper-V**, **HCS** quản lý container còn **HNS** quản lý mạng
ảo, endpoint, namespace và các policy đóng gói gói tin, cân bằng tải, ACL, NAT.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

```bash
kubectl -n "$NS" delete svc web-lb web-ext --wait=true --timeout=120s
```

---

## B10. Lưu trữ: bốn thứ chạy trên Linux, không có trên Windows

**Mục đích:** bài [106](../106-windows-storage-vi.md) gần như toàn bộ là một **danh sách phủ định**.
Mục này chạy bốn mục trong danh sách đó trên Linux để bạn thấy chính xác mình mất gì, rồi đo xem
**ai** cưỡng chế những hạn chế ấy — API server hay node.

### B10.1. Bốn cơ chế, một Pod

```bash
kubectl -n "$NS" create configmap lab-15-cm \
  --from-literal=only.conf='lab15-subpath-ok' --from-literal=khac.conf='khong-mount'
kubectl -n "$NS" create secret generic lab-15-mode --from-literal=token=lab15-mode-ok

cat > "$WK/storage-linux.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: storage-linux
  namespace: lab-15
spec:
  restartPolicy: Never
  nodeSelector:
    kubernetes.io/os: linux
  containers:
  - name: probe
    image: busybox:1.37
    command: ["sleep", "3600"]
    volumeMounts:
    - name: cm
      mountPath: /etc/app/only.conf
      subPath: only.conf
    - name: sec
      mountPath: /etc/sec
      readOnly: true
    - name: memdir
      mountPath: /memdir
    - name: rodir
      mountPath: /rodir
      readOnly: true
  volumes:
  - name: cm
    configMap:
      name: lab-15-cm
  - name: sec
    secret:
      secretName: lab-15-mode
      defaultMode: 0400
  - name: memdir
    emptyDir:
      medium: Memory
  - name: rodir
    emptyDir: {}
YAML

kubectl apply -f "$WK/storage-linux.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/storage-linux --timeout=180s

ST_SUBPATH="$(kubectl -n "$NS" exec storage-linux -- cat /etc/app/only.conf)"
ST_DIRN="$(kubectl -n "$NS" exec storage-linux -- sh -c 'ls -1 /etc/app | wc -l')"
ST_MODE="$(kubectl -n "$NS" exec storage-linux -- stat -c '%a' /etc/sec/token)"
ST_MEM="$(kubectl -n "$NS" exec storage-linux -- sh -c 'grep " /memdir " /proc/mounts')"
kubectl -n "$NS" exec storage-linux -- sh -c 'touch /rodir/probe' >/dev/null 2>&1; ST_RO=$?
kubectl -n "$NS" exec storage-linux -- sh -c 'touch /memdir/probe' >/dev/null 2>&1; ST_MEMW=$?

{
  echo "subPath      -> noi dung /etc/app/only.conf = $ST_SUBPATH"
  echo "subPath      -> so muc trong /etc/app        = $ST_DIRN"
  echo "defaultMode  -> quyen cua /etc/sec/token     = $ST_MODE"
  echo "medium Memory-> dong mount cua /memdir       = $ST_MEM"
  echo "volume readOnly -> ma thoat touch /rodir     = $ST_RO"
  echo "volume ghi duoc -> ma thoat touch /memdir    = $ST_MEMW"
} | tee "$EV/b10-storage-linux.txt"

test "$ST_SUBPATH" = 'lab15-subpath-ok' \
  && echo 'PASS: subPath mount dung MOT key cua ConfigMap thanh mot file'
test "$ST_DIRN" -eq 1 \
  && echo 'PASS: thu muc chua no chi co dung mot muc — day la subPath, khong phai mount ca volume'
test "$ST_MODE" = '400' \
  && echo 'PASS: defaultMode 0400 co tac dung — quyen file dung nhu khai bao'
echo "$ST_MEM" | grep -q 'tmpfs' \
  && echo 'PASS: emptyDir medium Memory duoc mount bang tmpfs'
test "$ST_RO" -ne 0 \
  && echo 'PASS: volume mount readOnly chan duoc ghi'
test "$ST_MEMW" -eq 0 \
  && echo 'PASS: volume khong readOnly van ghi binh thuong'
```

**Ý nghĩa:** sáu phép đo, và **bốn** trong số đó nằm thẳng trong danh sách "không được hỗ trợ trên
node Windows" của bài 106: mount subPath của volume, mount volume theo subPath cho Secret, thiết lập
quyền cho Secret bằng `defaultMode` (do phụ thuộc UID/GID), và dùng bộ nhớ làm phương tiện lưu trữ
(`emptyDir.medium: Memory`).

Hai phép đo cuối tách đúng chỗ dễ nhầm nhất của bài: **hệ thống file gốc chỉ đọc không được hỗ trợ,
nhưng volume chỉ đọc thì vẫn được**. Bạn vừa đo cả hai vế ở B5.4a và ở đây.

Và tất cả quy về **một nguyên nhân gốc** mà bài nói thẳng: SAM **không được chia sẻ** giữa host và
container, nên "không tồn tại ánh xạ nào giữa hai bên" và "mọi quyền đều được phân giải trong ngữ
cảnh của container". Không có ánh xạ uid/gid thì `defaultMode` mất nghĩa; quyền ghi bắt buộc cho
registry và SAM thì root filesystem không thể chỉ đọc. Một nguyên nhân, cả danh sách hệ quả — đó là
cách nhớ bài 106.

**PASS:** đủ sáu dòng `PASS:` của bước này.

### B10.2. Ai cưỡng chế hạn chế lưu trữ — API server hay node?

```bash
sed 's/^spec:$/spec:\n  os:\n    name: windows/; s/name: storage-linux/name: storage-windows/; s/kubernetes.io\/os: linux/kubernetes.io\/os: windows/' \
  "$WK/storage-linux.yaml" > "$WK/storage-windows.yaml"
grep -n 'os:\|subPath\|defaultMode\|medium' "$WK/storage-windows.yaml"

if kubectl apply --dry-run=server -f "$WK/storage-windows.yaml" > "$EV/b10-storage-windows.txt" 2>&1; then
  echo 'GHI NHAN: API server CHAP NHAN manifest luu tru nay du os.name la windows'
else
  echo 'GHI NHAN: API server TU CHOI manifest luu tru nay khi os.name la windows'
fi
head -5 "$EV/b10-storage-windows.txt"
test -s "$EV/b10-storage-windows.txt" \
  && echo 'PASS: da ghi lai cau tra loi cua API server cho manifest luu tru khai os.name=windows'
```

**Ý nghĩa:** câu hỏi ở đây khác câu hỏi của B4.3, và đó là điểm cần nhớ.

- Danh sách của **bài 175** — `hostPID`, `fsGroup`, `capabilities`, `privileged`… — được **API
  server** cưỡng chế. B4.3 đã đo: manifest bị từ chối ngay lúc `apply`.
- Danh sách của **bài 106** — `subPath`, `defaultMode`, `emptyDir.medium: Memory` — là hạn chế của
  **driver hệ thống file phân lớp trên NTFS** và của mô hình quyền Windows. Chúng không nằm trong
  danh sách trường bị `.spec.os.name` khóa.

Chép nguyên văn kết quả bạn nhận được vào evidence. Nếu API server **chấp nhận** manifest, bạn vừa
chứng minh được điều quan trọng nhất của mục này: **có những hạn chế của Windows mà tầng API không
bắt giúp bạn**. Manifest đi qua trót lọt, Pod được lập lịch lên node Windows, và sự cố xuất hiện ở
tầng kubelet hoặc tầng runtime — muộn hơn nhiều và khó truy ngược hơn nhiều.

Hệ quả thực hành: với workload Windows, `kubectl apply` chạy sạch **không** đồng nghĩa manifest đúng.
Danh sách của bài 106 phải được rà bằng mắt hoặc bằng policy riêng, không có ai rà giúp.

**PASS:** một dòng `GHI NHAN:` mô tả phản ứng của API server, cộng dòng
`PASS: da ghi lai cau tra loi…`; nội dung đã vào evidence.

```bash
kubectl -n "$NS" delete pod storage-linux --wait=true --timeout=120s
```

---
## B11. Thêm node Windows và chạy workload Windows *(chỉ nhánh A)*

> **Toàn bộ B11 chỉ chạy khi `NHANH` bằng `A`.** Ở nhánh B, đọc mục này rồi đi thẳng tới
> [B12](#b12-cleanup-và-gate-cuối). Đọc chứ đừng bỏ qua: nó là bản đồ những điều kiện bạn phải thu
> xếp nếu ngày mai môi trường thật của bạn có node Windows.

**Phạm vi của mục này, nói rõ ngay từ đầu:** quy trình chi tiết để thêm một worker node Windows vào
cluster nằm ở bài [216 — Thêm node worker Windows](../216-adding-windows-nodes-vi.md), thuộc
[giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node) — **giai đoạn sau**. Lab 15
không chép lại quy trình đó, vì [nguyên tắc không nhảy cóc](README.md#2-nguyên-tắc-không-nhảy-cóc)
cấm dùng nội dung chưa học để làm cho bài chạy được.

Cái B11 cung cấp là ba thứ **lấy từ đúng các bài của giai đoạn 15**: (1) danh sách **điều kiện** node
Windows phải thỏa, (2) các **gate `PASS:`** để biết node đã join đúng chưa, và (3) **bài thực hành
workload Windows** của bài 176 cùng 278. Chỗ nào bảy bài của giai đoạn 15 không nói tới, mục này ghi
thẳng là *phải tra tài liệu của đúng phiên bản Windows Server và đúng phiên bản Kubernetes bạn đang
chạy* — chứ không đoán hộ bạn.

### B11.A0. Cổng vào nhánh A

```bash
test "$NHANH" = 'A' || echo 'STOP: NHANH khac A — bo qua toan bo B11, di thang toi B12'
if [ "$NHANH" = 'A' ]; then
  kubectl get nodes -o custom-columns='NODE:.metadata.name,OS:.status.nodeInfo.operatingSystem,OSIMAGE:.status.nodeInfo.osImage,KUBELET:.status.nodeInfo.kubeletVersion,RUNTIME:.status.nodeInfo.containerRuntimeVersion' \
    | tee "$EV/b11-nodes-truoc.txt"
  WINNODE="$(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.nodeInfo.operatingSystem}{"\n"}{end}' \
    | awk '$2=="windows"{print $1; exit}')"
  echo "node Windows dau tien = ${WINNODE:-<khong co>}"
  test -n "$WINNODE" && echo 'PASS: da xac dinh duoc ten node Windows de dung cho cac buoc sau'
fi
```

**PASS:** biến `WINNODE` có giá trị; file `b11-nodes-truoc.txt` liệt kê node Windows kèm `osImage`.

### B11.A1. Điều kiện của chính host Windows

Lấy từ bài [175](../175-windows-intro-vi.md) và bài [315](../315-debug-windows-vi.md):

| Điều kiện | Nguồn | Cách xác nhận |
| --- | --- | --- |
| Hệ điều hành là **Windows Server 2022** hoặc **Windows Server 2025** | 175, mục *Tương thích phiên bản hệ điều hành Windows* | `osImage` ở `b11-nodes-truoc.txt` |
| Chỉ dùng **cách ly tiến trình**; Kubernetes **không hỗ trợ** cách ly Hyper-V | 175, mục *Node Windows trong Kubernetes* | cấu hình của container runtime trên host |
| Vai trò **worker**, không phải control plane | 174 | cột `ROLES` ở [B1.1](#b11-hệ-điều-hành-thật-của-từng-node) |
| Bộ xử lý 64-bit ≥ 4 nhân, ≥ 8GB RAM, ≥ 50GB đĩa trống | 175, mục *Khuyến nghị và lưu ý về phần cứng* | cấu hình VM trên host |
| Ưu tiên bản cài **không có Desktop Experience** để tiết kiệm tài nguyên | 175, cùng mục | lựa chọn lúc cài |
| **MAC spoofing bật** trên mọi adapter mạng của VM | 315, mục *Khắc phục sự cố mạng*, triệu chứng 1 | thiết lập adapter trong VMware |
| Dự trù dung lượng: Windows container image từ **300MB tới hơn 10GB**; ổ `C:` trong container mặc định hiện dung lượng ảo 20GB | 175, cùng mục | dung lượng đĩa VM |

Kiểm phiên bản hệ điều hành thật của node, và đối chiếu với bảng build ở [B7.1](#b71-label-nodekubernetesiowindows-build):

```bash
if [ "$NHANH" = 'A' ]; then
  WIN_OSIMAGE="$(kubectl get node "$WINNODE" -o jsonpath='{.status.nodeInfo.osImage}')"
  WIN_BUILD="$(kubectl get node "$WINNODE" \
    -o jsonpath='{.metadata.labels.node\.kubernetes\.io/windows-build}')"
  echo "osImage=$WIN_OSIMAGE | windows-build=$WIN_BUILD" | tee "$EV/b11-osimage.txt"

  echo "$WIN_OSIMAGE" | grep -qE '2022|2025' \
    && echo 'PASS: osImage la Windows Server 2022 hoac 2025 — dung pham vi bai 175 ho tro' \
    || echo "STOP: osImage la '$WIN_OSIMAGE', ngoai pham vi bai 175 ho tro — dung tai day"
  case "$WIN_BUILD" in
    10.0.20348) echo 'PASS: label windows-build khop Windows Server 2022 theo bang bai 176' ;;
    10.0.26100) echo 'PASS: label windows-build khop Windows Server 2025 theo bang bai 176' ;;
    '')         echo 'GHI NHAN: node khong mang label node.kubernetes.io/windows-build' ;;
    *)          echo "GHI NHAN: windows-build=$WIN_BUILD khong co trong bang bai 176 — ghi lai va tra tai lieu Microsoft" ;;
  esac
fi
```

**PASS:** dòng `PASS: osImage la Windows Server 2022 hoac 2025…` xuất hiện, và một dòng `PASS:` hoặc
`GHI NHAN:` cho label build; cả hai đã vào evidence.

### B11.A2. Container runtime trên node Windows

Bài [175](../175-windows-intro-vi.md) mục *Các container runtime* nêu hai lựa chọn: **ContainerD**
(ổn định từ Kubernetes v1.20, dùng ContainerD 1.4.0 trở lên) và **Mirantis Container Runtime**. Bài
[281](../281-create-hostprocess-pod-vi.md) siết thêm: HostProcess container **yêu cầu containerd 1.6
trở lên, khuyến nghị 1.7**, và chạy HostProcess dưới tài khoản user cục bộ **yêu cầu containerd
1.7+**.

Bài 175 cũng ghi một hạn chế đã biết: dùng **GMSA cùng containerd** để truy cập network share của
Windows đòi hỏi một bản vá kernel.

Bảy bài của giai đoạn 15 **không** đưa lệnh cài container runtime trên Windows. **Tra tài liệu chính
thức của đúng phiên bản Kubernetes bạn đang chạy** — trang *Container runtimes*, mục ContainerD, cùng
tài liệu của Microsoft cho phiên bản Windows Server tương ứng. Không suy từ quy trình Linux ở
[A4.2 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a42-cài-containerd-và-runc-đúng-version): đó là gói
Ubuntu, không dùng được ở đây.

Sau khi node đã join, đọc lại runtime thật:

```bash
if [ "$NHANH" = 'A' ]; then
  WIN_RUNTIME="$(kubectl get node "$WINNODE" \
    -o jsonpath='{.status.nodeInfo.containerRuntimeVersion}')"
  echo "containerRuntimeVersion = $WIN_RUNTIME" | tee "$EV/b11-runtime.txt"
  test -n "$WIN_RUNTIME" \
    && echo 'PASS: node Windows co bao len container runtime va phien ban cua no'
  echo "$WIN_RUNTIME" | grep -qi 'containerd' \
    && echo 'PASS: runtime la containerd — dieu kien cua HostProcess container o bai 281' \
    || echo "GHI NHAN: runtime khong phai containerd ($WIN_RUNTIME) — HostProcess container khong dung duoc"
fi
```

**PASS:** dòng `PASS: node Windows co bao len container runtime…` xuất hiện; dòng thứ hai là `PASS:`
hoặc `GHI NHAN:` và đã vào evidence.

### B11.A3. CNI phải tương thích **cả** Windows lẫn Linux

Bài [89](../89-windows-networking-vi.md) mục *Các chế độ mạng* nói thẳng: *"Trong một cluster hỗn hợp
với các worker node Windows và Linux, bạn cần chọn một giải pháp mạng tương thích trên cả Windows
lẫn Linux."* Bảng năm chế độ của bài là bảng để chọn, và nó cũng ghi rõ chế độ **NAT không dùng trong
Kubernetes**.

Đọc CNI **đang chạy** trên cluster của bạn rồi đối chiếu bảng đó trước khi join node Windows:

```bash
if [ "$NHANH" = 'A' ]; then
  kubectl get daemonset -A -o custom-columns='NS:.metadata.namespace,NAME:.metadata.name,DESIRED:.status.desiredNumberScheduled,READY:.status.numberReady' \
    | tee "$EV/b11-cni-daemonset.txt"
  kubectl get pods -A -o wide --field-selector "spec.nodeName=$WINNODE" \
    | tee "$EV/b11-pods-tren-node-windows.txt"

  CNI_ON_WIN="$(kubectl get pods -A --no-headers --field-selector "spec.nodeName=$WINNODE" 2>/dev/null | wc -l)"
  echo "so Pod he thong dang chay tren node Windows: $CNI_ON_WIN"
  test "$CNI_ON_WIN" -ge 1 \
    && echo 'PASS: co it nhat mot Pod he thong chay tren node Windows — CNI va kube-proxy da co mat' \
    || echo 'STOP: khong Pod he thong nao tren node Windows — CNI chua phu node nay, dung tai day'
fi
```

**Ý nghĩa:** đây là gate quan trọng nhất trước khi chạy workload. CNI của cluster lab được thay ở
[Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md); nếu bản Windows của nó không tồn tại thì node
Windows sẽ `Ready` nhưng Pod trên đó không có mạng — đúng triệu chứng số 1 mục *Khắc phục sự cố
mạng* của bài 315.

Bài 89 ghi hai đường Flannel hỗ trợ trên Windows: **VXLAN backend** ủy quyền cho `win-overlay` (hỗ
trợ **beta**) và **host-gateway backend** ủy quyền cho `win-bridge` (hỗ trợ **stable**), cộng hạn
chế Flannel chỉ dùng VNI 4096 và UDP port 4789. Ghi lại lựa chọn của bạn vào evidence — checkpoint
hỏi đúng câu đó.

**PASS:** dòng `PASS: co it nhat mot Pod he thong chay tren node Windows…` xuất hiện.

### B11.A4. Node đã join đúng chưa — sáu gate

Lệnh join sinh trên `lab-k8s-master` bằng đúng cách [A5.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a53-join-hai-worker)
đã dùng:

```bash
if [ "$NHANH" = 'A' ]; then kubeadm token create --print-join-command; fi
```

Phần chạy lệnh đó **trên host Windows** cùng các bước đặc thù Windows đi kèm nằm ở bài
[216](../216-adding-windows-nodes-vi.md) của giai đoạn 16 — ngoài phạm vi lab này. Bài
[176](../176-windows-user-guide-vi.md) chỉ đóng góp đúng một chi tiết ở bước đăng ký, và nó quan
trọng: kubelet trên node Windows nên đăng ký kèm taint

```text
--register-with-taints='os=windows:NoSchedule'
```

để **không gì được lập lịch lên node Windows**, kể cả những Pod Linux có sẵn không mang
`nodeSelector`. Cơ chế của taint này bạn đã kiểm chứng ở [B3](#b3-taint-và-toleration-theo-hệ-điều-hành).

Sáu gate sau khi node đã join:

```bash
if [ "$NHANH" = 'A' ]; then
  kubectl wait --for=condition=Ready "node/$WINNODE" --timeout=600s

  G_OS="$(kubectl get node "$WINNODE" -o jsonpath='{.status.nodeInfo.operatingSystem}')"
  G_LBL="$(kubectl get node "$WINNODE" -o jsonpath='{.metadata.labels.kubernetes\.io/os}')"
  G_ARCH="$(kubectl get node "$WINNODE" -o jsonpath='{.metadata.labels.kubernetes\.io/arch}')"
  G_ROLE="$(kubectl get node "$WINNODE" \
    -o jsonpath='{.metadata.labels.node-role\.kubernetes\.io/control-plane}')"
  G_TAINT="$(kubectl get node "$WINNODE" \
    -o jsonpath='{range .spec.taints[*]}{.key}={.value}:{.effect}{"\n"}{end}')"
  G_CIDR="$(kubectl get node "$WINNODE" -o jsonpath='{.spec.podCIDR}')"
  {
    echo "operatingSystem   = $G_OS"
    echo "kubernetes.io/os  = $G_LBL"
    echo "kubernetes.io/arch= $G_ARCH"
    echo "role control-plane= ${G_ROLE:-<khong co, dung la worker>}"
    echo "taints            = ${G_TAINT:-<khong co>}"
    echo "podCIDR           = ${G_CIDR:-<chua cap>}"
  } | tee "$EV/b11-node-windows.txt"

  test "$G_OS" = 'windows'  && echo 'PASS: kubelet bao len operatingSystem=windows'
  test "$G_LBL" = 'windows' && echo 'PASS: node mang label kubernetes.io/os=windows'
  test -n "$G_ARCH"         && echo 'PASS: node mang label kubernetes.io/arch'
  test -z "$G_ROLE"         && echo 'PASS: node Windows KHONG mang role control-plane — dung vai tro worker cua bai 174'
  test -n "$G_CIDR"         && echo 'PASS: node duoc cap podCIDR — CNI da nhan node nay'
  echo "$G_TAINT" | grep -qx 'os=windows:NoSchedule' \
    && echo 'PASS: node dang ky kem taint os=windows:NoSchedule theo khuyen nghi bai 176' \
    || echo 'GHI NHAN: node khong mang taint os=windows:NoSchedule — moi Pod khong co nodeSelector deu co the roi vao day'
fi
```

**PASS:** năm dòng `PASS:` đầu xuất hiện; dòng thứ sáu là `PASS:` hoặc `GHI NHAN:` và đã vào evidence.

### B11.A5. Workload Windows có Service

Dùng **nguyên văn manifest `win-webserver.yaml` của bài
[176](../176-windows-user-guide-vi.md)**, mục *Bắt đầu: Triển khai một workload Windows* — không tự viết
lại lệnh PowerShell trong `command`. Sửa đúng ba chỗ, và **chỉ ba chỗ**:

1. **Tag của image.** Ví dụ trong bài dùng `mcr.microsoft.com/windows/servercore:ltsc2019`, nhưng bài
   [175](../175-windows-intro-vi.md) quy định *"phiên bản hệ điều hành của host phải khớp với phiên
   bản hệ điều hành của image cơ sở của container"*, và phạm vi hỗ trợ là Server 2022 hoặc 2025. Đổi
   tag cho khớp `osImage` bạn đọc được ở B11.A1; tra tài liệu Microsoft để lấy đúng tag của phiên bản
   đó.
2. **Thêm `namespace: lab-15`** vào `metadata` của cả Service lẫn Deployment, để cleanup ở B12 gom
   được.
3. **Thêm toleration** khớp taint đã đăng ký ở B11.A4 — vì bài 176 nói Pod Windows cần **cả**
   `nodeSelector` **lẫn** toleration:

```yaml
     tolerations:
     - key: "os"
       operator: "Equal"
       value: "windows"
       effect: "NoSchedule"
```

Giữ nguyên `nodeSelector: kubernetes.io/os: windows` mà bài đã có. Nếu cluster của bạn có nhiều
phiên bản Windows Server, thêm khóa thứ hai `node.kubernetes.io/windows-build` theo bảng ở
[B7.1](#b71-label-nodekubernetesiowindows-build).

```bash
if [ "$NHANH" = 'A' ]; then
  kubectl apply -f "$WK/win-webserver.yaml"
  kubectl -n "$NS" rollout status deploy/win-webserver --timeout=1800s
  kubectl -n "$NS" get pods -l app=win-webserver -o wide | tee "$EV/b11-win-pods.txt"

  WPODS_READY="$(kubectl -n "$NS" get deploy win-webserver -o jsonpath='{.status.readyReplicas}')"
  WPODS_NODE="$(kubectl -n "$NS" get pods -l app=win-webserver \
    -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | tr '\n' ' ')"
  echo "readyReplicas=$WPODS_READY | node=$WPODS_NODE"

  test "$WPODS_READY" = '2' && echo 'PASS: ca hai Pod Windows deu Ready'
  echo "$WPODS_NODE" | grep -qw "$WINNODE" \
    && echo 'PASS: Pod Windows nam dung tren node Windows'
fi
```

`--timeout=1800s` ở đây phản ánh việc **kéo Windows container image mất lâu hơn nhiều** so với image
Linux — bài 175 ghi kích thước từ 300MB tới hơn 10GB. Đây là timeout của lệnh chờ, không phải cam
kết về thời gian.

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Pod kẹt ở `ContainerCreating` hoặc restart lặp
lại thì đọc triệu chứng số 1 của bài 315 — pause image không khớp phiên bản; `ErrImgPull` hoặc
`ImagePullBackOff` thì đọc triệu chứng số 2 — Pod lên node không tương thích phiên bản, quay lại
điểm 1 ở đầu B11.A5.

### B11.A6. Bảy phép kiểm chứng kết nối của bài 176

Bài 176 liệt kê đúng bảy phép; chạy tất cả, mỗi phép một gate. **Chạy từ `lab-k8s-master`**, đúng như
bài quy định là "từ node control plane Linux":

```bash
if [ "$NHANH" = 'A' ]; then
  WSVC_IP="$(kubectl -n "$NS" get svc win-webserver -o jsonpath='{.spec.clusterIP}')"
  WSVC_NP="$(kubectl -n "$NS" get svc win-webserver -o jsonpath='{.spec.ports[0].nodePort}')"
  WPOD_IPS="$(kubectl -n "$NS" get pods -l app=win-webserver \
    -o jsonpath='{range .items[*]}{.status.podIP}{"\n"}{end}')"
  WPOD1="$(echo "$WPOD_IPS" | head -1)"
  WPOD_NAME="$(kubectl -n "$NS" get pods -l app=win-webserver \
    -o jsonpath='{.items[0].metadata.name}')"
  WNODE_IP="$(kubectl get node "$WINNODE" \
    -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
  echo "svcIP=$WSVC_IP nodePort=$WSVC_NP podIP=$WPOD1 nodeIP=$WNODE_IP"

  # 1. Liet ke duoc nhieu Pod tu node control plane Linux
  test "$(kubectl -n "$NS" get pods -l app=win-webserver --no-headers | wc -l)" -ge 2 \
    && echo 'PASS: 1/7 liet ke duoc cac Pod Windows tu control plane Linux'

  # 2. Node toi Pod: curl vao port 80 cua IP Pod, tu control plane Linux
  test "$(curl -s -o /dev/null -w '%{http_code}' "http://$WPOD1:80")" = '200' \
    && echo 'PASS: 2/7 node toi Pod'

  # 3. Pod toi Pod: ping giua cac Pod bang kubectl exec
  kubectl -n "$NS" exec "$WPOD_NAME" -- ping -n 2 "$(echo "$WPOD_IPS" | tail -1)" \
    | tee "$EV/b11-pod-to-pod.txt"
  grep -qiE 'reply from|TTL=' "$EV/b11-pod-to-pod.txt" \
    && echo 'PASS: 3/7 Pod toi Pod — ICMP trong cung mang hoat dong dung nhu bai 89 mo ta'

  # 4. Service toi Pod: curl IP ao cua Service, tu control plane Linux VA tu chinh Pod
  test "$(curl -s -o /dev/null -w '%{http_code}' "http://$WSVC_IP:80")" = '200' \
    && echo 'PASS: 4a/7 service toi Pod, goi tu control plane Linux'
  kubectl -n "$NS" exec "$WPOD_NAME" -- powershell -command \
    "(Invoke-WebRequest -UseBasicParsing http://$WSVC_IP).StatusCode" | tee "$EV/b11-svc-tu-pod.txt"
  grep -q '200' "$EV/b11-svc-tu-pod.txt" \
    && echo 'PASS: 4b/7 service toi Pod, goi tu chinh Pod Windows'

  # 5. Kham pha service bang ten kem hau to DNS mac dinh
  kubectl -n "$NS" exec "$WPOD_NAME" -- powershell -command \
    "(Invoke-WebRequest -UseBasicParsing http://win-webserver.$NS.svc.cluster.local).StatusCode" \
    | tee "$EV/b11-dns.txt"
  grep -q '200' "$EV/b11-dns.txt" && echo 'PASS: 5/7 kham pha service bang ten DNS'

  # 6. Ket noi chieu vao: NodePort tu control plane Linux
  test "$(curl -s -o /dev/null -w '%{http_code}' "http://$WNODE_IP:$WSVC_NP")" = '200' \
    && echo 'PASS: 6/7 NodePort goi tu node khac — dung vi tri ma bai 89 noi la hoat dong'

  # 7. Ket noi chieu ra: dung curl chu KHONG dung ping, theo bai 89 va 315
  kubectl -n "$NS" exec "$WPOD_NAME" -- powershell -command \
    "(Invoke-WebRequest -UseBasicParsing https://kubernetes.io).StatusCode" | tee "$EV/b11-egress.txt"
  grep -q '200' "$EV/b11-egress.txt" \
    && echo 'PASS: 7/7 ket noi chieu ra bang TCP — khong dung ping vi ICMP xuyen mang xa khong duoc ho tro'
fi
```

Phép kiểm thứ tám, **cố ý kỳ vọng thất bại**, để bạn tận mắt thấy giới hạn đã biết của bài 89 và bài
315 thay vì chỉ đọc về nó:

```bash
if [ "$NHANH" = 'A' ]; then
  echo "Tren chinh node Windows ($WINNODE), mo PowerShell va chay:"
  echo "  curl.exe http://127.0.0.1:$WSVC_NP"
  echo "  curl.exe http://$WSVC_IP"
  cat > "$EV/b11-gioi-han-tren-node.txt" <<'EOF'
# Chep nguyen van ket qua hai lenh chay TREN CHINH node Windows vao day.
# Bai 89 muc Han che va bai 315 trieu chung 3 va 5 deu noi hai lenh nay that bai,
# trong khi tu node khac hoac client ngoai thi chay - phep kiem 6/7 o tren da chung minh.
EOF
  test -f "$EV/b11-gioi-han-tren-node.txt" \
    && echo 'PASS: da tao cho ghi ket qua hai phep thu tren chinh node Windows'
fi
```

**Ý nghĩa:** bảy phép kiểm của bài 176 phủ đủ bốn hướng lưu lượng — node tới Pod, Pod tới Pod, Service
tới Pod, và ra ngoài — cộng khám phá bằng DNS và đường vào bằng NodePort. Phép thứ tám thì phủ **cái
không chạy**, và đó là phần dễ chẩn đoán nhầm nhất: đứng trên node Windows tự gọi NodePort hay
service IP của chính mình sẽ hỏng, **không phải vì Service hỏng**.

Chú ý phép 7 dùng `Invoke-WebRequest` chứ không dùng `ping`. Bài 89 giải thích: data plane Windows
(**VFP**) không chuyển vị được gói ICMP, nên ICMP tới đích **trong cùng mạng** thì chạy — phép 3 vừa
chứng minh — còn ICMP **xuyên mạng ở xa** thì không định tuyến ngược về nguồn được. TCP/UDP vẫn
chuyển vị bình thường, nên bài khuyên thay `ping <đích>` bằng `curl <đích>`.

**PASS:** tám dòng `PASS:` (1/7 tới 7/7 cộng 4b) xuất hiện; file `b11-gioi-han-tren-node.txt` đã có
nội dung chép về từ node Windows.

### B11.A7. `runAsUserName` trên node Windows thật

Đây là nơi trả nốt phần mà [B5.1](#b51-runasusername-trên-cluster-toàn-linux--ba-tình-huống-osname)
không đo được. Dùng **đúng manifest của bài [278](../278-configure-runasusername-vi.md)**, đổi tag
image cho khớp phiên bản host và thêm `namespace: lab-15` cùng toleration như ở B11.A5:

```bash
if [ "$NHANH" = 'A' ]; then
  kubectl apply -f "$WK/run-as-username-pod.yaml"
  kubectl -n "$NS" wait --for=condition=Ready pod/run-as-username-pod-demo --timeout=1800s

  RAU="$(kubectl -n "$NS" exec run-as-username-pod-demo -- powershell -command 'echo $env:USERNAME' \
    | tr -d '\r')"
  echo "USERNAME trong container = $RAU" | tee "$EV/b11-runasusername.txt"
  test "$RAU" = 'ContainerUser' \
    && echo 'PASS: tien trinh chay duoi ContainerUser — runAsUserName CO tac dung tren node Windows'

  kubectl apply -f "$WK/run-as-username-container.yaml"
  kubectl -n "$NS" wait --for=condition=Ready pod/run-as-username-container-demo --timeout=1800s
  RAU2="$(kubectl -n "$NS" exec run-as-username-container-demo -- powershell -command 'echo $env:USERNAME' \
    | tr -d '\r')"
  echo "USERNAME khi dat o CA hai cap = $RAU2" | tee -a "$EV/b11-runasusername.txt"
  test "$RAU2" = 'ContainerAdministrator' \
    && echo 'PASS: gia tri o cap container GHI DE gia tri o cap Pod — dung dieu bai 278 noi'
fi
```

**Ý nghĩa:** đặt cạnh kết quả của B5.1, cặp số liệu này khép lại bài học của cả mục B5. Cùng một
trường, cùng một object: trên node Linux nó được lưu và bị bỏ qua; trên node Windows nó quyết định
danh tính của tiến trình. Và như bài 131 nhắc, mặc định phụ thuộc base image — image dựa trên **Nano
Server** chạy dưới `ContainerUser`, image dựa trên **Server Core** chạy dưới
`ContainerAdministrator`. Đổi base image là đổi quyền mặc định mà manifest không hề thay đổi;
`runAsUserName` là cách ghi đè điều đó một cách tường minh.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B12. Cleanup và gate cuối

**Mục đích:** xóa mọi thứ lab tạo ra — trong namespace và ở phạm vi cluster — rồi **chứng minh bằng
so sánh giá trị** rằng cluster ở đúng trạng thái mà [blockquote đầu file](#lab-15--node-windows) khai
báo cho nhánh của bạn.

### B12.1. Xóa object của lab

```bash
kubectl -n "$NS" delete pod --all --wait=true --timeout=300s
kubectl -n "$NS" get all
kubectl delete namespace "$NS" --wait=true --timeout=300s

kubectl delete runtimeclass lab-15-windows --ignore-not-found

# Taint cua B3 phai da duoc go o B3.4; kiem lai lan nua cho chac.
if kubectl get node "$W2" -o jsonpath='{range .spec.taints[*]}{.key}{"\n"}{end}' | grep -qx 'os'; then
  echo 'GHI NHAN: worker2 van con taint os — B3.4 chua chay hoac chay khong xong, go ngay bay gio'
  kubectl taint node "$W2" os=windows:NoSchedule-
else
  echo 'PASS: worker2 khong con taint os nao — B3.4 da lam dung'
fi

rm -f "$WK"/*.yaml
rm -rf "$WK/oslock" "$WK/runasusername" "$WK/hostprocess"
rmdir "$WK"
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến điều đó
thành gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/15/` **giữ lại** — đó là bằng chứng, và
`b1-nang-luc.txt` là hồ sơ để lần sau đối chiếu.

**PASS:** không có lỗi nào khi xóa; `rmdir` thành công.

### B12.2. Gate cuối chung cho cả hai nhánh

```bash
kubectl get namespace "$NS" >/dev/null 2>&1 \
  && echo 'FAIL: namespace lab-15 van con' \
  || echo 'PASS: namespace lab-15 da xoa'

kubectl get runtimeclass lab-15-windows >/dev/null 2>&1 \
  && echo 'FAIL: RuntimeClass cua lab van con' \
  || echo 'PASS: RuntimeClass cua lab da xoa'

RC_AFTER="$(kubectl get runtimeclass -o name 2>/dev/null | wc -l)"
API_AFTER="$(kubectl api-resources -o name 2>/dev/null | sort -u | wc -l)"
CRD_AFTER="$(kubectl get crd -o name 2>/dev/null | wc -l)"
NS_AFTER="$(kubectl get namespace -o name | sort | tr '\n' ' ')"
TAINT_AFTER="$(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}={.spec.taints[*].key}{"\n"}{end}' | sort | tr '\n' ' ')"
LBL_AFTER="$(kubectl get nodes --no-headers --show-labels | awk '{print $1, $NF}' | sort | md5sum)"

test "$RC_AFTER"  -eq "$RC_BEFORE" \
  && echo "PASS: so RuntimeClass tro ve dung muc dau lab ($RC_BEFORE)"
test "$API_AFTER" -eq "$API_BEFORE" \
  && echo "PASS: so API resource khong doi ($API_BEFORE) — lab khong bat nhom API nao"
test "$CRD_AFTER" -eq "$CRD_BEFORE" \
  && echo "PASS: so CRD khong doi ($CRD_BEFORE) — lab khong cai CRD nao, ke ca CRD cua GMSA"
test "$NS_AFTER"  =   "$NS_BEFORE" \
  && echo 'PASS: danh sach namespace khong doi'

test ! -e "$WK" && echo 'PASS: manifest tam da xoa'

kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'
kubectl wait --for=condition=Ready node --all --timeout=180s
kubectl get pods -n default
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl top node
```

**PASS:** không có dòng `FAIL:` nào; sáu dòng `PASS:` của bước này xuất hiện; mọi node `Ready`;
`kubectl get pods -n default` trả `No resources found in default namespace.`; lệnh field selector trả
`No resources found`; CoreDNS đủ replica `READY`; `kubectl top node` in đủ số dòng bằng số node.

### B12.3. Nhánh B — chứng minh cluster về đúng `04-metrics-ready`

```bash
if [ "$NHANH" = 'B' ]; then
  test "$TAINT_AFTER" = "$TAINT_BEFORE" \
    && echo 'PASS: chuoi taint cua ca ba node giong het truoc lab — taint cua B3 da go sach' \
    || { echo 'FAIL: chuoi taint khac truoc lab'; echo "truoc: $TAINT_BEFORE"; echo "sau  : $TAINT_AFTER"; }

  test "$LBL_AFTER" = "$LBL_BEFORE" \
    && echo 'PASS: nhan cua ca ba node khong doi — lab khong gan nhan nao'

  NODE_AFTER="$(kubectl get nodes --no-headers | wc -l)"
  test "$NODE_AFTER" -eq "$NODE_N" \
    && echo "PASS: so node khong doi ($NODE_N) — lab khong them VM nao vao cluster"

  WIN_AFTER="$(kubectl get nodes -o jsonpath='{range .items[*]}{.status.nodeInfo.operatingSystem}{"\n"}{end}' \
    | grep -cx 'windows' || true)"
  test "$WIN_AFTER" -eq 0 \
    && echo 'PASS: cluster van toan Linux — lab khong dung them node Windows nao'

  {
    echo "=== $(date -Is) — Lab 15 nhanh B: cluster tro ve 04-metrics-ready ==="
    echo "node                 : $NODE_N -> $NODE_AFTER"
    echo "node Windows         : $WIN_N -> $WIN_AFTER"
    echo "API resource         : $API_BEFORE -> $API_AFTER"
    echo "CRD                  : $CRD_BEFORE -> $CRD_AFTER"
    echo "RuntimeClass         : $RC_BEFORE -> $RC_AFTER"
    echo "taint (chuoi day du) : $TAINT_BEFORE -> $TAINT_AFTER"
    echo "nhan node (md5)      : $LBL_BEFORE -> $LBL_AFTER"
  } | tee "$EV/b12-truoc-sau.txt"
fi
```

Rồi chạy **toàn bộ bảy tầng của [gate A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready)**
theo đúng file Lab 00 — không chép lại nội dung ở đây. Lab 15 đã đụng vào taint của một node và tạo
Service kiểu NodePort, nên chạy đủ bảy tầng là cách chắc chắn nhất để biết cluster còn nguyên vẹn.
Nhớ xóa sạch resource test của A5.4.8 sau khi chạy xong.

**PASS:** bốn dòng `PASS:` của B12.3 xuất hiện, không dòng `FAIL:` nào; bảy tầng A5.4 đều PASS; file
`b12-truoc-sau.txt` có đủ bảy dòng so sánh và **không dòng nào lệch**. Cluster trở về
`04-metrics-ready`; **không chụp snapshot mới** và **không cần restore** — lab sau bắt đầu ngay từ
trạng thái này.

Nếu `TAINT_AFTER` khác `TAINT_BEFORE`, xem [mục 4](#4-troubleshooting-của-lab-này) trước khi đi tiếp.
Đừng chấp nhận một cluster còn taint sót: mọi lab sau đều giả định hai worker sạch taint.

### B12.4. Nhánh A — chụp mốc `15-windows-ready`

```bash
if [ "$NHANH" = 'A' ]; then
  kubectl get nodes -o custom-columns='NODE:.metadata.name,OS:.status.nodeInfo.operatingSystem,STATUS:.status.conditions[-1].type,TAINTS:.spec.taints' \
    | tee "$EV/b12-nodes-cuoi.txt"
  kubectl wait --for=condition=Ready node --all --timeout=600s
  echo 'PASS: moi node — ke ca node Windows — deu Ready truoc khi chup moc'
fi
```

Tắt **tất cả** VM sạch sẽ rồi mới chụp: node Windows trước, rồi `lab-k8s-worker2` →
`lab-k8s-worker1` → `lab-k8s-master`, ngược thứ tự bật ở
[A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab). Chờ VMware Workstation hiển thị
tất cả ở trạng thái *Powered off*.

Chụp trên **từng VM**, kể cả VM Windows: chuột phải VM → **Snapshot → Take Snapshot** → ô *Name*
điền đúng nguyên văn:

```text
15-windows-ready
```

Ô *Description* ghi lab đã dựng mốc này và ngày chụp, ví dụ
`dựng bằng LAB-15-NODE-WINDOWS.md, chụp <ngày>`.

Quy tắc tên giống hệt Lab 00 và Lab 11a: đúng nguyên văn `15-windows-ready` trên mọi VM, không hậu tố
theo VM, không thêm ngày, không đổi hoa thường. **Giữ nguyên mốc `04-metrics-ready`** trên ba VM
Linux — đừng xóa nó, đó vẫn là điểm quay lui và là điểm bắt đầu của các lab khác.

Verify từ PowerShell trên máy host — thêm đường dẫn `.vmx` của VM Windows vào mảng:

```powershell
$vmrun = 'C:\Program Files\VMware\VMware Workstation\vmrun.exe'
$vmxAll = @(
  'E:\Virtual Machines\lab-k8s-master\lab-k8s-master.vmx'
  'E:\Virtual Machines\lab-k8s-worker1\lab-k8s-worker1.vmx'
  'E:\Virtual Machines\lab-k8s-worker2\lab-k8s-worker2.vmx'
  'E:\Virtual Machines\lab-k8s-win1\lab-k8s-win1.vmx'
)
foreach ($f in $vmxAll) {
  $names = & $vmrun -T ws listSnapshots $f | Select-Object -Skip 1
  if ($names -ccontains '15-windows-ready') { "PASS: $f" }
  else { "FAIL: $f -> $($names -join ', ')" }
}
```

**PASS:** đúng một dòng `PASS:` cho mỗi VM, không dòng `FAIL:` nào — `-ccontains` phân biệt hoa
thường nên gate này bắt được cả lỗi gõ sai tên. Đường dẫn `.vmx` của VM Windows theo máy host đang
dùng; sửa lại cho đúng.

---

## 3. Checkpoint 15

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu. **Mọi câu dưới
đây trả lời được kể cả khi bạn chạy nhánh B** — đó là điều kiện thiết kế của checkpoint này.

- [ ] Bạn chạy `kubectl` từ một laptop Windows vào cluster ba VM Ubuntu. Cluster của bạn đã là
      cluster "có Windows" chưa? Kubernetes hỗ trợ Windows ở **vai trò nào**, và vai trò nào **không**
      được hỗ trợ? Sáu phép kiểm năng lực của B1 là gì, cluster của bạn thiếu điều kiện nào và
      **thiếu vì lý do gì** — phần cứng, hệ điều hành, hay cấu hình? Vì sao kết quả đó **không** tạo
      ra một món nợ trong [sổ nợ lab](README.md#5-sổ-nợ-lab)?
- [ ] Bạn đặt `.spec.os.name: windows` cho một Pod và không đặt gì thêm. Scheduler có nhờ đó mà đưa
      Pod lên node Windows không? Bạn đã chứng minh điều đó bằng **trường nào** trên object? Vậy
      `.spec.os.name` dùng để làm gì, và nó **rất mạnh** ở tầng nào?
- [ ] Kể ba cơ chế quyết định một Pod rơi vào node nào trong cluster hỗn hợp hệ điều hành, và nói
      cái nào là điều kiện cần, cái nào là điều kiện đủ. Vì sao bài 176 khuyên dùng **taint** thay vì
      chỉ sửa `nodeSelector` cho mọi manifest? Bạn đã chứng minh hai vế chặn và cho qua bằng hai Pod
      khác nhau đúng bao nhiêu dòng?
- [ ] `runAsUser: 1000` trên node Ubuntu làm gì, và bạn đọc kết quả đó ra bằng lệnh nào? Cùng trường
      đó khi `.spec.os.name` là `windows` thì sao? Windows dùng gì thay thế, và **vì sao** không thể
      ánh xạ một–một — câu trả lời nằm ở khác biệt khái niệm nào của bài 175?
- [ ] Bạn apply một Pod có `securityContext.windowsOptions.runAsUserName` lên cluster toàn Linux của
      mình. Kết quả **thật** bạn quan sát được là gì ở ba tình huống `os.name` — không khai, khai
      `linux`, khai `windows`? Nếu API server chấp nhận, tiến trình trong container chạy dưới user
      nào, và điều đó dạy bạn gì về nguy cơ của manifest sai hệ điều hành trong cluster hỗn hợp?
- [ ] Trên node Linux, ranh giới kiểm soát tài nguyên là gì và ở **mức nào**? Bạn đã nhìn thấy nó ở
      đường dẫn nào trên `lab-k8s-worker2`? Trên node Windows ranh giới là gì, ở mức nào, và
      "job object" có liên quan gì tới Kubernetes Job không?
- [ ] Container vượt memory limit trên node Linux bị xử lý thế nào? Trên node Windows thì sao? Ghép
      với việc `--enforce-node-allocable` và `PIDPressure` chưa được triển khai, vì sao bài 112 cảnh
      báo **mạnh** về Pod không có `limits` trên node Windows — dù bạn đã đặt `--kube-reserved` và
      `--system-reserved`?
- [ ] Cluster có cả Windows Server 2022 và 2025. Vì sao `kubernetes.io/os: windows` là chưa đủ, và
      label nào giải quyết? RuntimeClass giúp gì, và bạn đã **chứng minh** nó giúp bằng cách nào —
      Pod của bạn khai bao nhiêu dòng, và object tạo ra có thêm những gì?
- [ ] Ứng dụng .NET của bạn ghi log đầy đủ vào event log của Windows. `kubectl logs` thấy gì? Bạn đã
      tái hiện đúng nguyên nhân gốc đó trên Linux bằng phép thử nào, và ba con số nào chứng minh
      "ứng dụng khỏe, đường đi của log sai"? Bài 176 khuyến nghị công cụ nào, và nó sửa cái gì?
- [ ] Bạn ssh vào node Windows, `curl` NodePort của chính nó và thấy hỏng. Cluster có đang hỏng
      không? Từ node khác thì sao, và từ Pod Windows trên chính node đó thì sao? Bạn đã đo ba con số
      tương ứng trên cluster Linux của mình ở bước nào? Còn `ping 8.8.8.8` từ Pod Windows không có
      phản hồi thì kết luận được gì — và phải kiểm bằng lệnh nào thay thế?
- [ ] Trên Linux bạn dùng `subPath`, `defaultMode`, `emptyDir.medium: Memory` và quyền theo uid/gid.
      Bốn thứ đó trên node Windows ra sao, và **một nguyên nhân gốc chung** là gì? Quan trọng hơn:
      ai cưỡng chế danh sách của bài 175, ai cưỡng chế danh sách của bài 106, và hệ quả thực hành của
      sự khác biệt đó là gì?
- [ ] Lab này kết thúc theo nhánh. Nhánh của bạn là gì, và bạn đã dùng **những phép so sánh nào** để
      chứng minh cluster đang ở đúng trạng thái mà nhánh đó khai báo?

### Bài giải thích cuối cùng

Trong tối đa 10 phút, kể lại **một Pod đi từ `kubectl apply` tới lúc chạy trong một cluster có cả
node Linux và node Windows, và nó có thể chết ở những đâu**:

1. Người vận hành gõ `kubectl apply`. Câu hỏi đầu tiên không phải "viết YAML thế nào" mà là "cluster
   này có node Windows không" — và bạn trả lời bằng cột `operatingSystem` của
   `kubectl get nodes -o custom-columns=…`, không bằng trí nhớ. Bạn còn phân biệt được máy Windows
   trên host với node Windows trong cluster, và biết `kubectl` chạy ở đâu thì không liên quan.
2. **Chốt chặn thứ nhất là API server.** Nếu Pod khai `.spec.os.name: windows` thì hai mươi trường
   của bài 175 bị khóa; khai `linux` thì `windowsOptions` bị khóa. Bạn đã đo cả hai chiều và bạn có
   phép đối chứng chứng minh biến duy nhất là dòng `os.name`. Nhưng bạn cũng biết chốt chặn này
   **không** phủ danh sách lưu trữ của bài 106 — chỗ đó không ai gác giúp bạn.
3. **Chốt chặn thứ hai là scheduler**, và nó **không nhìn `.spec.os.name`**. Thứ nó nhìn là
   `nodeSelector kubernetes.io/os`, taint và toleration, cộng `node.kubernetes.io/windows-build` khi
   cluster có nhiều phiên bản Windows. Ba thứ đó bạn đã kiểm chứng bằng ba cặp Pod, mỗi cặp khác
   nhau đúng vài dòng và cho kết quả ngược nhau. RuntimeClass gói cả `nodeSelector` lẫn toleration
   lại một chỗ, và bạn đã thấy admission tiêm chúng vào một Pod chỉ khai đúng một dòng.
4. **Chốt chặn thứ ba là kubelet**, thành phần duy nhất biết hệ điều hành thật của node. Đây là nơi
   một Pod đã có `nodeName` vẫn có thể không bao giờ chạy, và bạn đã ghi lại đúng `reason` mà cluster
   mình trả về.
5. Pod chạy rồi thì **ranh giới tài nguyên khác nhau**: cgroup của Pod trên Linux, job object của
   từng container trên Windows; kernel OOM kill trên Linux, chuyển trang xuống đĩa trên Windows;
   `NodeAllocatable` được cưỡng chế trên Linux, chỉ để tính toán trên Windows. Bạn đã nhìn thấy
   cgroup thật với `memory.max` thật và condition `PIDPressure` thật.
6. Khi có sự cố, **quy trình debug chia làm hai nửa**: nửa dựa vào API của Kubernetes giữ nguyên —
   `describe`, Event, `exec`, `nodeSelector`; nửa dựa vào hệ điều hành thì đổi — đường đi của log,
   ICMP, NodePort gọi từ chính node, `port-forward`. Bạn có bảng đối chiếu và bạn đã đo vế Linux của
   từng dòng đo được.
7. Và điều quan trọng nhất của lab: khi cluster không có node Windows, việc đúng là **đo, ghi lại,
   nói rõ dừng ở đâu và vì sao** — không phải dựng thêm VM giữa chừng hay bịa ra dữ liệu để bảng kết
   quả trông đầy đủ hơn. Giai đoạn 15 là giai đoạn cuối của Phần I, và bài học nó để lại không phải
   là Windows, mà là: **API của Kubernetes chung cho mọi node, hệ điều hành thì không — và bạn phải
   biết đường ranh giới nằm ở đâu**.

Khi mọi checkbox được đánh dấu và không còn nhầm `.spec.os.name` với `nodeSelector`, "API server
chấp nhận" với "workload chạy đúng", "cgroup của Pod" với "job object của container", hay "cluster
không có node Windows" với "lab không kiểm chứng được gì", Lab 15 và
[giai đoạn 15](../00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows) hoàn
tất — với ghi chú rõ ràng rằng phần dựng node Windows và phần GMSA cần một môi trường khác.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ liệt kê
sự cố phát sinh từ nội dung bài học giai đoạn 15.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| `WIN_N` bằng 0 và bạn muốn chạy nhánh A | `cat ~/lab-evidence/15/b1-nang-luc.txt` | **Không phải lỗi.** Cluster lab của Lab 00 có ba VM Linux; nhánh B là kết quả đúng. **Không** dựng VM Windows giữa chừng lab: thêm một node là đổi hạ tầng chuỗi snapshot chính. Muốn nhánh A thì chuẩn bị VM Windows Server **trước khi mở lab**, rồi chạy lại từ B0 |
| `WIN_LBL` khác `WIN_N` | `kubectl get nodes --show-labels`; `~/lab-evidence/15/b1-labels.txt` | Có người gán tay label `kubernetes.io/os`. Label này do kubelet đặt; sửa nó bằng tay làm scheduler tin sai và Pod chết ở tầng kubelet. Ghi lại, **không** gán thêm label; tìm ai đã đổi, hoặc restore cả ba VM về `04-metrics-ready` |
| B2.1: Pod `os-selector-windows` **không** ở `Pending` | `kubectl -n lab-15 describe pod os-selector-windows`; `~/lab-evidence/15/b2-windows-describe.txt` | Ghi nguyên văn `describe` vào evidence. Nếu Pod được gán node, kiểm ngay `WIN_LBL` ở dòng trên — gần như chắc chắn có node Linux đang mang label `windows`. **Không** sửa lab cho khớp kết quả |
| B2.1: Event `FailedScheduling` không xuất hiện sau vòng lặp | `kubectl -n lab-15 get events --sort-by=.lastTimestamp` | Event có thể đã bị dọn theo TTL nếu bạn quay lại bước này muộn. Xóa Pod, apply lại và đọc Event ngay. `describe` vẫn hiện Event gần nhất |
| B3: `kubectl taint` báo taint đã tồn tại | `kubectl get node lab-k8s-worker2 -o jsonpath='{.spec.taints}'`; `~/lab-evidence/15/b0-anh-nen.txt` | Một lần chạy trước để sót. Gỡ bằng `kubectl taint node lab-k8s-worker2 os=windows:NoSchedule-` rồi chạy lại B3.1. Nếu `TAINT_BEFORE` ở B0 đã có taint thì cluster lệch mốc đầu vào — restore cả ba VM |
| B3.4: gỡ taint xong `TAINT_AFTER` vẫn khác `TAINT_BEFORE` | `kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'` | So từng node với `b0-anh-nen.txt`. Gỡ đúng taint đã thêm. **Không** gỡ taint `node-role.kubernetes.io/control-plane:NoSchedule` của control plane — đó là taint của baseline. Không xác định được thì restore cả ba VM |
| B4.3: vòng lặp có manifest **không** bị từ chối | `cat ~/lab-evidence/15/b4-oslock.txt` | Ghi lại đúng tên trường lọt lưới và thông báo kèm theo. Đây là dữ liệu về **cluster của bạn**, cần báo chứ không cần vá; nói rõ ở checkpoint. Kiểm phép đối chứng ở cuối B4.3 vẫn `CHAP NHAN` — nếu nó cũng bị từ chối thì lỗi nằm ở manifest chứ không ở `os.name` |
| B4.3: phép đối chứng cũng bị từ chối | `cat ~/lab-evidence/15/b4-doi-chung.txt`; `cat ~/lab-work/15/oslock-doi-chung.yaml` | Lệnh `sed` cắt nhầm dòng. Mở file ra xem, sửa tay cho đúng rồi chạy lại. Không kết luận gì về `os.name` khi phép đối chứng chưa `CHAP NHAN` |
| B5.1: `kubectl exec` vào `runasusername-noos` báo lỗi | `kubectl -n lab-15 get pod runasusername-noos -o wide` | Pod chưa `Ready`. Chờ `kubectl wait` xong rồi mới `exec`. Nếu Pod không bao giờ `Ready`, đọc `describe` và ghi vào evidence — đó chính là câu trả lời cho tình huống này trên cluster bạn |
| B5.3 hoặc B6.2 hoặc B8.2: `ssh lab-k8s-worker2` không dùng được | `ssh lab-k8s-worker2 hostname` | Lab 7b và Lab 12 dùng cùng đường này. Nếu chưa cấu hình `ssh` không mật khẩu từ master, làm điều đó trước, hoặc bỏ phần đo trên node và ghi rõ ở checkpoint rằng bạn chỉ đo được từ trong container. **Không** thay bằng cách chạy Pod đặc quyền để đọc host |
| B6.2: `find` không tìm thấy cgroup mang uid của Pod | `ssh lab-k8s-worker2 'stat -fc %T /sys/fs/cgroup'`; `ssh lab-k8s-worker2 'sudo ls /sys/fs/cgroup/kubepods.slice'` | Cgroup driver hoặc bố cục thư mục khác baseline. Ghi lại đường dẫn thật vào evidence và đọc `memory.max` ở đó. Nếu `stat -fc %T` không trả `cgroup2fs` thì node lệch [A4.1](LAB-00-MOI-TRUONG-1.35.7.md#a41-cập-nhật-os-tắt-swap-và-bật-kernel-prerequisites) — đó là sự cố môi trường, xem mục 4 của Lab 00 |
| B6.2: `POD_MAX` khác `67108864` | `kubectl -n lab-15 get pod qos-guaranteed -o jsonpath='{.spec.containers[0].resources}'` | Đọc lại limit thật của Pod rồi so với `memory.max`. Hai số phải bằng nhau; ghi cả hai vào evidence. Đừng sửa con số trong lab cho khớp — đó là dữ liệu |
| B6.3: `configz` trả `403` | `kubectl config current-context` | Chạy trên `lab-k8s-master` bằng kubeconfig quản trị. **Không** tạo ClusterRoleBinding để chữa — cách đúng đã có ở [Lab 11a B3](LAB-11A-OBSERVABILITY.md#b3-đọc-metric-là-một-hành-động-được-ủy-quyền) |
| B7.2: Pod `rc-consumer` **không** được tiêm `nodeSelector` | `kubectl -n lab-15 get pod rc-consumer -o yaml`; `kubectl get runtimeclass lab-15-windows -o yaml` | Kiểm `scheduling` trong RuntimeClass có đủ hai khối `nodeSelector` và `tolerations` không. Nếu có mà Pod vẫn không được tiêm thì admission plugin RuntimeClass đang tắt trên cluster bạn — ghi lại, **không** sửa cấu hình apiserver |
| B7.2: `kubectl apply` Pod báo xung đột `nodeSelector` | `kubectl -n lab-15 get pod rc-consumer -o yaml` | Pod tự khai một khóa `nodeSelector` trùng khóa của RuntimeClass với giá trị khác. Manifest ở B7.2 cố ý **không** khai `nodeSelector`; đảm bảo bạn dùng đúng file đó |
| B8.1: `kubectl logs -c ra-file` **có** nội dung | `kubectl -n lab-15 logs log-duong-di -c ra-file` | Container đang in ra STDOUT chứ không chỉ ghi file — kiểm lại `command` trong manifest có `>>` không. Sửa manifest rồi chạy lại; phép thử chỉ có nghĩa khi đúng một container ghi ra STDOUT |
| B9.2: một trong ba mã HTTP khác `200` | `kubectl -n lab-15 get endpointslice`; `kubectl -n lab-15 get pods -o wide` | Đây là sự cố mạng của cluster, không phải nội dung giai đoạn 15. Chạy lại [tầng 3 và tầng 5 của A5.4](LAB-00-MOI-TRUONG-1.35.7.md#a54-gate-01-cluster-ready) trước khi kết luận gì về NodePort |
| B10.1: `stat -c %a /etc/sec/token` khác `400` | `kubectl -n lab-15 get pod storage-linux -o jsonpath='{.spec.volumes}'` | Kiểm `defaultMode` trong manifest có bị YAML đọc thành số thập phân không — viết `0400` chứ không viết `400`. Sửa rồi tạo lại Pod |
| B11.A5: Pod Windows kẹt `ContainerCreating` hoặc restart lặp lại | `kubectl -n lab-15 describe pod <ten>`; cấu hình containerd trên node Windows | Triệu chứng số 1 của bài 315: **pause image không tương thích phiên bản Windows**. Kiểm `plugins.plugins.cri.sandbox_image` trong `config.toml` của node Windows |
| B11.A5: Pod Windows báo `ErrImgPull` hoặc `ImagePullBackOff` | `kubectl -n lab-15 describe pod <ten>`; `~/lab-evidence/15/b11-osimage.txt` | Triệu chứng số 2 của bài 315: Pod lên node Windows **không tương thích phiên bản**. Đổi tag image cho khớp `osImage`, hoặc thêm `node.kubernetes.io/windows-build` vào `nodeSelector` |
| B11.A6: Pod Windows không có kết nối mạng | thiết lập adapter của VM Windows trên host | Triệu chứng số 1 mục *Khắc phục sự cố mạng* của bài 315: **bật MAC spoofing** trên **tất cả** adapter mạng của VM |
| B11.A6: phép 7/7 thất bại nhưng phép 3/7 đạt | dùng `curl`/`Invoke-WebRequest`, không dùng `ping` | Nếu bạn đã dùng TCP mà vẫn hỏng, xem `ExceptionList` trong `cni.conf` theo hướng **bớt**: IP đích phải **không** nằm trong danh sách thì traffic mới được SNAT đúng |
| B12.2: `API_AFTER`, `CRD_AFTER` hoặc `RC_AFTER` khác giá trị trước lab | `~/lab-evidence/15/b0-anh-nen.txt` | Có thứ gì đó được cài hoặc để sót trong lúc chạy lab — đó là điều lab cấm. Xóa đúng object thừa; không xác định được thì restore cả ba VM về `04-metrics-ready` và chạy lại từ B0 |
| B12.1: namespace `lab-15` kẹt `Terminating` | `kubectl -n lab-15 get pods,svc`; `kubectl -n lab-15 get pod -o yaml \| grep finalizers -A3` | Còn Pod chưa kết thúc. Xóa Pod trước rồi chờ; **không** cưỡng chế finalizer |

---

## 5. Nguồn chính thức

- [Windows in Kubernetes](https://kubernetes.io/docs/concepts/windows/)
- [Windows containers in Kubernetes](https://kubernetes.io/docs/concepts/windows/intro/)
- [Guide for Running Windows Containers in Kubernetes](https://kubernetes.io/docs/concepts/windows/user-guide/)
- [Networking on Windows](https://kubernetes.io/docs/concepts/services-networking/windows-networking/)
- [Windows Storage](https://kubernetes.io/docs/concepts/storage/windows-storage/)
- [Resource Management for Windows nodes](https://kubernetes.io/docs/concepts/configuration/windows-resource-management/)
- [Security For Windows Nodes](https://kubernetes.io/docs/concepts/security/windows-security/)
- [Configure GMSA for Windows Pods and containers](https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa/)
- [Configure RunAsUserName for Windows pods and containers](https://kubernetes.io/docs/tasks/configure-pod-container/configure-runasusername/)
- [Create a Windows HostProcess Pod](https://kubernetes.io/docs/tasks/configure-pod-container/create-hostprocess-pod/)
- [Windows debugging tips](https://kubernetes.io/docs/tasks/debug/debug-cluster/windows/)
- [Labels, Annotations and Taints — `node.kubernetes.io/windows-build`](https://kubernetes.io/docs/reference/labels-annotations-taints/#nodekubernetesiowindows-build)
- [RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
- [Adding Windows worker nodes — đọc ở giai đoạn 16, là nơi có quy trình join đầy đủ](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/adding-windows-nodes/)
