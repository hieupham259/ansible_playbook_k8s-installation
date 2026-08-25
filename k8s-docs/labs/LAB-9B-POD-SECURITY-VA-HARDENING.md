# Lab 9b — Pod Security và hardening

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do Lab 6a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** cleanup trả cluster về đúng `03-storage-ready`, **không tạo snapshot mới**.
> Lab này **không cài thêm bất kỳ thành phần hạ tầng nào**, **không cài admission webhook**, và
> **không sửa cấu hình `kube-apiserver` hay kubelet** — kể cả để bật audit log.
> **Lab trước:** [Lab 9a — ServiceAccount, authn/authz và RBAC](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md),
> cũng bắt đầu và kết thúc ở `03-storage-ready`. Cluster vào lab này phải sạch: có StorageClass
> mặc định, không workload, không object nào của lab trước.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng **phần policy và hardening** của mục
[Giai đoạn 9 — Bảo mật và multi-tenancy](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy):
mười hai bài khái niệm [115](../115-pod-security-standards-vi.md),
[116](../116-pod-security-admission-vi.md), [121](../121-secrets-good-practices-vi.md),
[122](../122-multi-tenancy-vi.md), [123](../123-hardening-authentication-vi.md),
[126](../126-linux-security-vi.md), [127](../127-linux-kernel-security-vi.md),
[128](../128-api-server-bypass-risks-vi.md), [129](../129-security-checklist-vi.md),
[130](../130-application-security-checklist-vi.md), [166](../166-flow-control-vi.md),
[173](../173-admission-webhooks-vi.md), bài lịch sử [117](../117-pod-security-policy-vi.md),
cộng năm bài thực hành [254](../254-running-cloud-controller-vi.md),
[258](../258-sysctl-cluster-vi.md),
[282](../282-enforce-standards-admission-controller-vi.md),
[283](../283-enforce-standards-namespace-labels-vi.md), [286](../286-migrate-from-psp-vi.md),
[291](../291-security-context-vi.md).

**Phần truy cập của giai đoạn 9 không thuộc lab này.** ServiceAccount, ba chặng
authentication → authorization → admission, bốn tổ hợp Role/ClusterRole × RoleBinding/
ClusterRoleBinding và `kubectl auth can-i` đã làm ở
[Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md). Lab 9b **không lặp lại** chúng: nó chỉ
**dùng lại** RBAC ở đúng một chỗ — lớp thứ hai của mô hình tenant ở B8 — và ở đó cũng chỉ tạo
Role/RoleBinding trong phạm vi một namespace.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không
chép lại con số phiên bản nào**. Chỗ duy nhất cần số minor của Kubernetes là nhãn
`pod-security.kubernetes.io/<MODE>-version`, và B0.2 **suy nó ra từ chính API server** rồi gate
lại, thay vì gõ tay. Thành phần ngoài baseline đang chạy — CNI thay Flannel, ingress controller,
dynamic provisioner — đã khóa ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) và do Lab 5b, Lab 6a
cài; lab 9b chỉ **đọc** chúng, không đụng vào.

Topology vẫn là một control plane `lab-k8s-master` mang taint `NoSchedule` và hai worker
`lab-k8s-worker1`, `lab-k8s-worker2`. Lab dùng Pod, Deployment, Secret, ResourceQuota,
LimitRange, NetworkPolicy, Role và RoleBinding của các giai đoạn đã học làm công cụ. **Không**
dùng metrics-server và HPA ([giai đoạn 11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)),
DRA ([giai đoạn 13](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao)), CRD,
`ValidatingAdmissionPolicy` và admission webhook
([giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng)).

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm ba lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Ba lenh rieng cua lab 9b: dung diem bat dau, chua co gi cua lab, va CNI du suc cho B8.
kubectl get storageclass | grep -q '(default)' \
  && echo 'PASS: co StorageClass mac dinh — dung diem bat dau 03-storage-ready'
test -z "$(kubectl get namespace -o name | grep 'namespace/lab-9b' || true)" \
  && echo 'PASS: chua co namespace nao mang tien to lab-9b'
kubectl get pods -A -o wide | grep -qiE 'calico|cilium' \
  && echo 'PASS: CNI co thuc thi NetworkPolicy dang chay — B8 dua han vao dieu nay'
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; **ba dòng `PASS:` riêng của lab đều
xuất hiện**. Nếu dòng cuối fail, cluster của bạn chưa qua Lab 5b — mọi gate mạng của B8 sẽ vô
nghĩa, dừng lại trước khi đi tiếp.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- **Ranh giới giữa ba profile của [115](../115-pod-security-standards-vi.md) bằng thực nghiệm**:
  chỉ ra được đúng thứ nào làm một Pod trượt Baseline, đúng thứ nào Restricted **thêm** vào trên
  Baseline, và viết được một Pod đạt Restricted mà vẫn chạy.
- **Ba chế độ của [116](../116-pod-security-admission-vi.md) khác nhau ở đâu**, chứng minh bằng
  ba kết quả quan sát được: `enforce` chặn ngay lúc tạo Pod, `warn` cho tạo nhưng in cảnh báo,
  `audit` không chặn cũng không cảnh báo — và biết chính xác **phần nào của `audit` cluster này
  chưa kiểm chứng được**, vì sao, và nó nằm ở giai đoạn nào.
- Sự bất đối xứng dễ nhầm nhất của Pod Security Admission: `enforce` **không** áp cho workload
  resource mà chỉ áp cho Pod object, còn `warn` và `audit` thì có — chứng minh bằng một Deployment
  `apply` trót lọt nhưng không đẻ ra nổi một Pod nào.
- Ba chế độ đặt **đồng thời** trên cùng một namespace ở **ba mức khác nhau**, và đọc được bảng sự
  thật của tổ hợp đó.
- Vì sao ai patch được label của Namespace thì vô hiệu hóa được chính sách — và cách
  [283](../283-enforce-standards-namespace-labels-vi.md) dùng `--dry-run=server` để soi cả cluster
  trước khi siết.
- **Security context nhìn từ bên trong container** ([291](../291-security-context-vi.md),
  [127](../127-linux-kernel-security-vi.md)): đọc được `runAsUser`, `runAsGroup`, `fsGroup`,
  `allowPrivilegeEscalation`, `capabilities`, `seccompProfile`, AppArmor và
  `readOnlyRootFilesystem` **bằng trạng thái thật của tiến trình**, không bằng cách tin vào
  manifest.
- Một container `privileged: true` **xóa sạch** cả ba ràng buộc kernel cùng lúc — chứng minh bằng
  ba giá trị đọc được trong chính container đó.
- Phân biệt **sysctl an toàn và không an toàn** ([258](../258-sysctl-cluster-vi.md)): chỉ ra được
  cùng một Pod chết ở **hai mặt phẳng khác nhau** tùy namespace — bị admission từ chối, hay được
  lập lịch rồi kubelet không cho khởi chạy.
- **PodSecurityPolicy đã biến mất khỏi Kubernetes** ([117](../117-pod-security-policy-vi.md),
  [286](../286-migrate-from-psp-vi.md)): chứng minh bằng chính API của cluster này, và ánh xạ được
  lộ trình di trú của bài sang cluster đã qua mốc gỡ bỏ.
- **Chuỗi admission thật đang chạy**: liệt kê được các plugin đang hoạt động từ số liệu của chính
  API server, chỉ ra `PodSecurity` có mặt và `PodSecurityPolicy` thì không.
- **API Priority and Fairness** ([166](../166-flow-control-vi.md)): đọc được FlowSchema và
  PriorityLevelConfiguration thật, phân biệt đối tượng bắt buộc với đối tượng được đề xuất, và chỉ
  ra được request của chính bạn rơi vào FlowSchema nào.
- **Failure policy của admission webhook** ([173](../173-admission-webhooks-vi.md)): giải thích
  được vì sao `Fail` mặc định có thể làm chết cluster và vì sao bài khuyên mutating webhook
  "fail open" — kèm tồn kho webhook thật của cluster này.
- **Mô hình cô lập bốn lớp của [122](../122-multi-tenancy-vi.md)**: dựng một namespace tenant có đủ
  namespace + RBAC + quota + NetworkPolicy, rồi chứng minh **từng lớp** chặn đúng thứ nó phải chặn.
- Dùng được hai checklist [129](../129-security-checklist-vi.md) và
  [130](../130-application-security-checklist-vi.md) như **công cụ rà soát**: mỗi ô có câu trả lời
  lấy từ cluster này, kèm ô nào không áp dụng và vì sao.
- Cleanup đúng phạm vi: xóa hết Pod đặc quyền, chứng minh **cấu hình control plane và kubelet
  không hề bị sửa**, chứng minh tồn kho object phạm vi cluster trở về đúng con số cũ, và đưa
  cluster về đúng `03-storage-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong phần policy của giai đoạn 9 | Kiểm chứng ở |
| --- | --- |
| [115 — Chuẩn bảo mật Pod](../115-pod-security-standards-vi.md) | B1.1 ba profile là **định nghĩa**, không phải object; B2.2 ranh giới Baseline bằng năm lần từ chối liên tiếp; B2.3 Restricted **thêm** gì trên Baseline; B2.4 viết một Pod đạt Restricted; B5.3 danh sách sysctl an toàn chính là một ô trong bảng Baseline |
| [116 — Cơ chế admission bảo mật Pod](../116-pod-security-admission-vi.md) | B1.2 hai nhóm label mode × level; B2 toàn bộ chế độ `enforce`; B2.5 `enforce` không áp cho workload resource; B3.1 `warn`; B3.2 `warn` **có** áp cho workload resource; B3.3 `audit`; B3.4 ba chế độ ba mức trên một namespace; B3.5 đổi label làm plugin soi lại Pod đang chạy; B7.1 metric `pod_security_*` và plugin `PodSecurity` trong chuỗi admission |
| [117 — Chính sách bảo mật Pod](../117-pod-security-policy-vi.md) *(tài liệu lịch sử)* | B6.1–B6.2 — bài chỉ khẳng định một điều kiểm chứng được: API đã bị gỡ khỏi Kubernetes. Lab kiểm chứng đúng điều đó trên cluster này, không dựng lại PSP |
| [121 — Thực hành tốt cho Secret](../121-secrets-good-practices-vi.md) | B9.6 giới hạn Secret cho **một** container trong Pod hai container, và Secret volume là `tmpfs`; B9.1 ô "mã hóa khi lưu trữ" — đọc được là **chưa bật**, nợ #6 |
| [122 — Đa người thuê](../122-multi-tenancy-vi.md) | B8 toàn bộ — namespace, RBAC, quota, NetworkPolicy dựng thành một mô hình rồi kiểm từng lớp; B8.6 những lớp cô lập bài nêu mà cluster này không có |
| [123 — Hardening xác thực](../123-hardening-authentication-vi.md) | B9.3 — liệt kê **cơ chế xác thực nào thật sự đang bật** trên cluster này, và đối chiếu từng khuyến nghị của bài với cấu hình đọc được |
| [126 — Bảo mật cho node Linux](../126-linux-security-vi.md) | B9.4 — Secret mount vào Pod nằm trên `tmpfs`, và swap tắt trên cả ba node nên điều kiện kích hoạt rủi ro của bài không tồn tại |
| [127 — Ràng buộc bảo mật của Linux kernel](../127-linux-kernel-security-vi.md) | B4.4 `allowPrivilegeEscalation` và `no_new_privs`; B4.5 capability; B4.6 seccomp và AppArmor đọc từ `/proc`; B4.8 container đặc quyền ghi đè cả ba; B9.4 AppArmor bật sẵn trên node Ubuntu đúng như bài nói |
| [128 — Rủi ro vượt qua API server](../128-api-server-bypass-risks-vi.md) | B9.5 — bốn đường vòng, kiểm phần đọc được: thư mục static Pod manifest, cấu hình xác thực/phân quyền của kubelet, địa chỉ etcd đang lắng nghe, quyền trên socket container runtime |
| [129 — Danh sách kiểm tra bảo mật](../129-security-checklist-vi.md) | B9.1 — dùng như checklist đối chiếu: mỗi ô đọc được một câu trả lời từ cluster này và ghi vào evidence |
| [130 — Checklist bảo mật ứng dụng](../130-application-security-checklist-vi.md) | B9.2 — tự soi Pod của chính mình theo checklist: `securityContext` cấp Pod và cấp container, ServiceAccount, NetworkPolicy |
| [166 — Ưu tiên và công bằng cho API](../166-flow-control-vi.md) | B7.3 đọc FlowSchema và PriorityLevelConfiguration thật, phân biệt bốn đối tượng bắt buộc với sáu mức được đề xuất; B7.4 request của chính bạn rơi vào FlowSchema nào, đọc từ header phản hồi |
| [173 — Thực hành tốt cho admission webhook](../173-admission-webhooks-vi.md) | B7.2 — tồn kho webhook thật của cluster, bốn kind của nhóm API `admissionregistration.k8s.io`, và `failurePolicy` của từng webhook nếu có. **Lab không cài webhook**; phần còn lại ở bảng lý do bên dưới |
| [254 — Quản trị Cloud Controller Manager](../254-running-cloud-controller-vi.md) | B9.7 — chứng minh cluster này **không có** cloud provider: không có `cloud-controller-manager`, không có `--cloud-provider=external`, không node nào mang taint `node.cloudprovider.kubernetes.io/uninitialized`. Đó là toàn bộ phần bài có thể kiểm chứng ở đây |
| [258 — Dùng sysctl trong cluster](../258-sysctl-cluster-vi.md) | B5.1 `/proc/sys` chỉ đọc nên `sysctls` là đường duy nhất; B5.2 sysctl an toàn đổi giá trị **trong Pod** mà không đổi trên node; B5.3 sysctl không an toàn — cùng một Pod chết ở hai mặt phẳng khác nhau |
| [282 — Thực thi PSS bằng admission controller tích hợp](../282-enforce-standards-admission-controller-vi.md) | B1.3 — mặc định `privileged` cho namespace không có nhãn, chứng minh bằng chính namespace `lab-9b`; B7.1 apiserver **không** có `--admission-control-config-file`, nên bộ mặc định trong bài đang có hiệu lực nguyên vẹn |
| [283 — Thực thi PSS bằng nhãn Namespace](../283-enforce-standards-namespace-labels-vi.md) | B0.3 tạo namespace bằng đúng khuôn manifest của bài; B1.2 selector `!pod-security.kubernetes.io/enforce`; B3.5 `kubectl label --dry-run=server` cho một namespace và cho `--all` |
| [286 — Di chuyển từ PodSecurityPolicy](../286-migrate-from-psp-vi.md) | B6.1 API `policy/v1beta1` không còn; B6.2 manifest PSP của bài không apply được; B6.3 annotation `kubernetes.io/psp` không còn trên Pod nào; B6.4 ánh xạ năm bước di trú của bài sang cluster đã qua mốc gỡ bỏ; B8.2 bước 1 *Rà soát quyền trên namespace* |
| [291 — Cấu hình Security Context](../291-security-context-vi.md) | B2.4 `runAsUser`/`runAsGroup`/`fsGroup`/`supplementalGroups` và quyền trên volume; B4.2 `securityContext` cấp container ghi đè cấp Pod; B4.3 `runAsNonRoot` chặn image chạy bằng root; B4.4–B4.7 `allowPrivilegeEscalation`, `capabilities`, `seccompProfile`, `readOnlyRootFilesystem`; B5.1 masked path và read-only path của `/proc` |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [116](../116-pod-security-admission-vi.md), [282](../282-enforce-standards-admission-controller-vi.md): file `AdmissionConfiguration` / `PodSecurityConfiguration`, và **miễn trừ** theo username, RuntimeClass, namespace | Manifest đó phải truyền cho kube-apiserver qua cờ `--admission-control-config-file`. Lab **cấm sửa cấu hình control plane** — B0.4 và B10.3 biến điều đó thành gate. Thao tác thuộc [giai đoạn 8](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) bài [03](../03-control-plane-flags-vi.md) và [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). B7.1 kiểm chứng phần đọc được: cờ đó **không** có mặt |
| Bài [116](../116-pod-security-admission-vi.md): **nội dung** audit annotation mà chế độ `audit` ghi ra | Đọc được annotation đó cần một audit backend đang chạy, tức phải sửa cờ `--audit-policy-file` của kube-apiserver. B3.3 kiểm chứng đúng phần quan sát được từ ngoài — `audit` **không chặn và không cảnh báo** — cộng bộ đếm `pod_security_evaluations_total` của chính chế độ đó. Phần đọc log nằm ở [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu), bài [306](../306-audit-vi.md) |
| Bài [115](../115-pod-security-standards-vi.md): `windowsOptions.hostProcess`, mục *Trường OS của Pod*, các kiểm soát đặc thù hệ điều hành | Cluster lab chỉ có node Linux. Nội dung Windows nằm ở [giai đoạn 15](../00-ALO-TRINH-ADMIN.md#giai-đoạn-15--windows-nếu-môi-trường-có-node-windows) |
| Bài [115](../115-pod-security-standards-vi.md): user namespace, Pod sandbox, và ba policy engine bên thứ ba (Kyverno, OPA Gatekeeper, Kubewarden) | User namespace là bài [55](../55-user-namespaces-vi.md), sandbox cần RuntimeClass — bài [43](../43-runtime-class-vi.md). Ba policy engine là dự án bên thứ ba, cài chúng vi phạm quy ước "không cài thêm phần mềm" của lab |
| Bài [291](../291-security-context-vi.md): `seLinuxOptions`, gán lại nhãn SELinux hiệu quả, `SELinuxWarningController` | Node lab chạy Ubuntu, và bài [127](../127-linux-kernel-security-vi.md) nói hệ điều hành thường chỉ có **một** trong hai cơ chế MAC. B9.4 chứng minh cơ chế đang bật là **AppArmor**, nên không có chính sách SELinux nào để quan sát |
| Bài [291](../291-security-context-vi.md): `fsGroupChangePolicy`, ủy quyền cho CSI driver, `supplementalGroupsPolicy`, `procMount: Unmasked` | `fsGroupChangePolicy` chỉ có tác dụng với volume hỗ trợ quản lý quyền sở hữu, còn provisioner của baseline không phải CSI ([nợ #5](README.md#5-sổ-nợ-lab)). `procMount: Unmasked` đòi `hostUsers: false`, tức user namespace — bài [55](../55-user-namespaces-vi.md). `supplementalGroupsPolicy` cần feature gate, mà bật feature gate là sửa cấu hình control plane |
| Bài [291](../291-security-context-vi.md), [127](../127-linux-kernel-security-vi.md): viết seccomp profile hoặc AppArmor profile **tùy chỉnh** (`type: Localhost`) | Profile tùy chỉnh phải được nạp sẵn lên **mọi** node có thể nhận Pod, tức là copy file vào `<kubelet-root-dir>/seccomp/` hoặc nạp profile AppArmor trên từng node — đó là thay đổi cấu hình node, không phải object của cluster. Lab kiểm chứng nhánh `RuntimeDefault` và `Unconfined`, hai giá trị dùng được mà không đụng vào node |
| Bài [127](../127-linux-kernel-security-vi.md): Kubernetes Security Profiles Operator | Là operator bên thứ ba; Operator nằm ở [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) |
| Bài [258](../258-sysctl-cluster-vi.md): bật sysctl **không an toàn** bằng `--allowed-unsafe-sysctls` của kubelet | Là cờ của kubelet trên từng node. Lab chứng minh **mặt còn lại** của cùng một câu trong bài — Pod dùng sysctl không an toàn vẫn được lập lịch nhưng không khởi chạy được (B5.3) — mà không cần đổi cấu hình node nào |
| Bài [173](../173-admission-webhooks-vi.md): dựng máy chủ webhook, `matchConditions`, `reinvocationPolicy`, kiểm thử tính lũy đẳng của mutating webhook | Cần viết và triển khai một máy chủ webhook. Một `MutatingWebhookConfiguration` cấu hình sai làm chết **toàn cluster**, đúng thứ bài cảnh báo — nên lab **không cài webhook**. B7.2 kiểm chứng phần đọc được và giải thích `failurePolicy` trên tồn kho thật |
| Bài [130](../130-application-security-checklist-vi.md), [113](../113-security-vi.md): `ValidatingAdmissionPolicy` khuyến nghị cho workload nhạy cảm | Cần hiểu CEL và các điểm mở rộng API — [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng). Chính bài dịch xếp mục này vào phần *Đọc lướt* |
| Bài [166](../166-flow-control-vi.md): tạo hoặc sửa FlowSchema / PriorityLevelConfiguration, kịch bản server đệ quy, tinh chỉnh số ghế | FlowSchema và PriorityLevelConfiguration là object **phạm vi cluster** chi phối cách API server xử lý **mọi** request; một cấu hình sai làm đói chính kubelet và controller. Kịch bản đệ quy đòi một server thứ hai (webhook hoặc `APIService`). Cả hai đều ngoài phạm vi lab chỉ-đọc này |
| Bài [166](../166-flow-control-vi.md): metric APF ở mức BETA/ALPHA và mục *Khả năng quan sát* | Đọc và dựng biểu đồ từ `/metrics` cần Prometheus — [giai đoạn 11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability). B7.3 chỉ đọc **object** cấu hình, không dựng dashboard |
| Bài [122](../122-multi-tenancy-vi.md): container sandbox, control plane ảo cho mỗi tenant, service mesh, hạn chế tra cứu DNS liên namespace bằng policy của CoreDNS | Sandbox cần runtime thay thế (gVisor, Kata) và RuntimeClass — bài [43](../43-runtime-class-vi.md). Control plane ảo và service mesh là dự án bên thứ ba. Sửa cấu hình CoreDNS là sửa add-on hệ thống, thuộc [giai đoạn 21](../00-ALO-TRINH-ADMIN.md#giai-đoạn-21--dns-cni-và-kube-proxy) |
| Bài [122](../122-multi-tenancy-vi.md): cách ly node, QoS mạng bằng bandwidth plugin, độ ưu tiên và chiếm chỗ của Pod, StorageClass riêng cho mỗi tenant | Cách ly node bằng `nodeSelector`, taint và toleration đã thực hành ở [Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md); PriorityClass và preemption cũng vậy. StorageClass riêng cho tenant đã làm ở [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md). Bandwidth plugin được chính bài đánh dấu là **thử nghiệm**. B8 không làm lại, chỉ nhắc lại vị trí |
| Bài [121](../121-secrets-good-practices-vi.md): mã hóa Secret khi lưu trữ, Secrets Store CSI Driver, wipe/shred đĩa etcd | Mã hóa at rest là **nợ #6** trong [sổ nợ lab](README.md#5-sổ-nợ-lab), phải sửa cờ apiserver, trả ở [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu). Secrets Store CSI Driver là add-on bên thứ ba. Vận hành đĩa etcd thuộc [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa) |
| Bài [123](../123-hardening-authentication-vi.md): tích hợp OIDC, xác thực webhook, proxy xác thực | Cả ba cần một hệ thống danh tính bên ngoài cluster **và** sửa cờ của kube-apiserver. B9.3 kiểm chứng phần đọc được: cơ chế nào **đang** bật và cơ chế nào **không**, kèm lý do bài khuyên tránh từng cái |
| Bài [126](../126-linux-security-vi.md): rủi ro volume trong bộ nhớ bị ghi xuống đĩa khi node **có** swap | Ba node của lab tắt swap theo yêu cầu của kubeadm — B9.4 chứng minh điều đó, nên điều kiện kích hoạt rủi ro **không tồn tại**. Bật swap để tái hiện là đổi cấu hình node và thuộc [giai đoạn 12](../00-ALO-TRINH-ADMIN.md#giai-đoạn-12--quản-trị-cluster-nâng-cao), bài [170](../170-swap-memory-management-vi.md) |
| Bài [128](../128-api-server-bypass-risks-vi.md): tự dựng một static Pod ẩn, gọi thẳng API kubelet trên cổng 10250, dùng client certificate của etcd để đọc dữ liệu | Đây là **mô phỏng tấn công** trên chính cluster đang dùng cho mọi lab sau. Ba thao tác này để lại tiến trình không do API server quản lý, hoặc ghi thẳng vào kho dữ liệu. B9.5 kiểm chứng **biện pháp giảm thiểu** mà bài đề ra, tức đúng phần một quản trị viên phải rà |
| Bài [129](../129-security-checklist-vi.md), [130](../130-application-security-checklist-vi.md): quét image, ký image, tham chiếu image bằng `sha256` digest, thời hạn certificate, quy trình rà soát quyền định kỳ | Thuộc pipeline CI/CD và quy trình tổ chức, không có thao tác nào trên cluster để kiểm chứng. Vòng đời certificate là **nợ #7**, trả ở [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) |
| Bài [129](../129-security-checklist-vi.md): lọc truy cập tới cloud metadata API `169.254.169.254`, hạn chế `LoadBalancer` và `externalIPs` | Cluster lab chạy trên VM tự dựng, không có IaaS nên không có metadata API để lọc — B9.7 chứng minh điều đó. Không có cloud controller thì cũng không có Service `LoadBalancer` thật để hạn chế |
| Bài [254](../254-running-cloud-controller-vi.md): chạy và cấu hình `cloud-controller-manager`, ví dụ theo từng nhà cung cấp | Cần một tài khoản cloud và một cloud provider thỏa `cloudprovider.Interface`. B9.7 kiểm chứng đúng thứ kiểm chứng được trên cluster này: **không có** thành phần đó, và hệ quả của việc không có |
| Bài [286](../286-migrate-from-psp-vi.md): đơn giản hóa PSP mutating, gắn PSP privileged để vô hiệu hóa theo namespace, tắt admission controller PodSecurityPolicy | Không thể thực hiện vì **API PodSecurityPolicy không còn tồn tại** — B6 chứng minh đúng điều đó. Đây là lý do lộ trình xếp bài [117](../117-pod-security-policy-vi.md) là tài liệu lịch sử |

### 1.2. Thời lượng

3–4 giờ, tính từ lúc gate mở đầu đã PASS. Lab không cài gì, nhưng nó tạo và xóa nhiều Pod, và
phần lớn thời gian nằm ở chỗ **đọc kỹ câu thông báo từ chối** rồi đối chiếu với đúng ô trong bảng
đặc tả của bài [115](../115-pod-security-standards-vi.md). Nặng nhất là B2, B4 và B8. Các bước
phải chờ kubelet hoặc control plane hội tụ đều viết dưới dạng vòng lặp có điều kiện thoát, không
phải con số cố định.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi
  rõ node khác**. Lệnh cần `sudo` để đọc file trên node chạy trên chính node đó.
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Gần như mọi gate so sánh với biến đặt ở B0
  (`PSA_VER`, `W2`, `CR_N0`, `NP_N0`, các hàm `psa_try`, `proc_val`, `wait_field`); mở shell mới
  giữa chừng là mất hết và các gate sau sẽ sai.
- Manifest tạm ghi vào `~/lab-work/9b/`; bằng chứng ghi vào `~/lab-evidence/9b/`. Thư mục
  `~/lab-work/9b/` bị xóa ở B10.1, thư mục bằng chứng **giữ lại**.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.
- Image dùng cho toàn bộ lab là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và đã có sẵn trên cả ba node từ
  Lab 00, nên lab không phụ thuộc mạng ra ngoài. Image này **không có** `capsh`, nên mọi phép đo
  capability, seccomp và AppArmor đọc thẳng từ `/proc` — chính cách bài
  [291](../291-security-context-vi.md) làm.

**An toàn với Pod đặc quyền — quy ước bắt buộc của lab này:**

- Lab tạo Pod `privileged: true`, Pod chia sẻ namespace của host và Pod mount `hostPath`. Một Pod
  như thế **là** một lối thoát khỏi cơ chế cô lập container: bài
  [127](../127-linux-kernel-security-vi.md) nói nó được cấp **toàn bộ** Linux capability và ghi đè
  seccomp lẫn AppArmor. Vì vậy lab khai báo trước, đúng và đủ, những Pod loại này:

  | Pod | Đặc quyền gì | Node | Tạo ở | Xóa ở |
  | --- | --- | --- | --- | --- |
  | `dacquyen` (ns `lab-9b`) | `privileged: true`, kèm `seccompProfile: RuntimeDefault` để chứng minh nó bị ghi đè | **chỉ** `lab-k8s-worker2` | B4.8 | B4.9 và B10.1 |
  | `canh-bao` (ns `lab-9b-batche`) | `privileged: true`, dùng để kích hoạt cảnh báo của chế độ `warn` | **chỉ** `lab-k8s-worker2` | B3.1 | B10.1 |
  | `hostpath-tmp` (ns `lab-9b`) | mount `hostPath` `/tmp` của node, **chỉ đọc** | **chỉ** `lab-k8s-worker2` | B2.6 | B2.6 ngay sau khi đo |
  | `mac-dinh` (ns `lab-9b`) | thêm capability `SYS_ADMIN` và `seccompProfile: Unconfined` — chưa phải `privileged` nhưng đủ để leo thang | **chỉ** `lab-k8s-worker2` | B1.3 | B10.1 |

- **Mọi Pod đặc quyền đều ghim vào `lab-k8s-worker2`** bằng `nodeSelector`, đúng quy ước fault
  injection của [mục 6 trong README](README.md#6-quy-ước-chung-trong-mọi-lab). Không bao giờ chạy
  chúng trên `lab-k8s-master`: control plane của cluster nằm ở đó dưới dạng static Pod, và bài
  [128](../128-api-server-bypass-risks-vi.md) nói rõ ai chạm được vào filesystem của node đó thì
  chạm được vào control plane.
- **`hostPath` chỉ được mount `/tmp`, chỉ đọc, và bị xóa ngay trong cùng bước.** Không mount
  `/etc/kubernetes`, `/var/lib/kubelet`, `/run/containerd/containerd.sock` hay bất kỳ đường dẫn hệ
  thống nào khác — bài [128](../128-api-server-bypass-risks-vi.md) liệt kê đúng những thứ đó là
  đường vòng qua API server. Phần cần đọc từ các đường dẫn ấy, B9.5 đọc **trực tiếp trên node bằng
  `sudo`**, không mount vào container.
- B0.5 đếm số Pod `privileged: true` đang có trên cluster **trước** khi lab bắt đầu; B10.2 so lại
  đúng con số đó. Cluster baseline vốn đã có vài Pod đặc quyền của CNI và kube-proxy trong
  `kube-system` — gate là **so giá trị**, không phải đòi con số bằng không.

**Phạm vi và cách quay lui:**

- **Không cài thêm hạ tầng**, không tạo snapshot mới, **không sửa cấu hình `kube-apiserver`,
  kubelet hay manifest trong `/etc/kubernetes/manifests`.** Cụ thể: **không bật audit log**, không
  đổi danh sách admission plugin, không thêm `--admission-control-config-file`, không thêm
  `--allowed-unsafe-sysctls`. B0.4 ghi checksum, B10.3 đối chiếu — đó là gate chứng minh bạn không
  đụng vào.
- **Không cài admission webhook.** Một `MutatingWebhookConfiguration` hay
  `ValidatingWebhookConfiguration` sai phạm vi làm chết mọi request tạo object trên cluster, kể cả
  của chính control plane. Bài [173](../173-admission-webhooks-vi.md) tồn tại chính vì chuyện đó.
- Lab tạo đúng năm Namespace, tất cả mang tiền tố `lab-9b`: `lab-9b` (không nhãn PSA),
  `lab-9b-baseline`, `lab-9b-restricted`, `lab-9b-batche` và `lab-9b-tenant`. Toàn bộ bị xóa ở
  B10.1. Lab **không tạo object phạm vi cluster nào** — B10.2 gate điều đó bằng cách so tồn kho.
- **Không sửa nhãn PSA của bất kỳ namespace nào ngoài năm namespace trên.** B3.5 có chạy
  `kubectl label` trên `--all` namespace, nhưng **luôn kèm `--dry-run=server`** — đó là điểm mấu
  chốt của bước đó, không phải chi tiết phụ.

**Cách quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `03-storage-ready` — không bao
giờ restore riêng một VM, xem ghi chú cuối
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) — rồi bật lại theo
thứ tự master → worker 1 → worker 2, chạy lại gate mở đầu và bắt đầu lại từ B0.

---

# Phần B — Thực hành kiến thức 9b

## B0. Chuẩn bị workspace, năm namespace và ảnh chụp "trước"

**Mục đích:** dựng chỗ làm việc, **suy ra** nhãn phiên bản chính sách từ chính API server, tạo năm
namespace theo đúng khuôn manifest của bài [283](../283-enforce-standards-namespace-labels-vi.md),
định nghĩa các hàm trợ giúp mà toàn bộ lab dùng để **so giá trị** thay vì so chuỗi, và chụp lại
trạng thái "trước" của cả tồn kho object lẫn cấu hình control plane — hai thứ mà B10 sẽ đối chiếu.

### B0.1. Workspace và ba hàm trợ giúp

```bash
mkdir -p ~/lab-work/9b ~/lab-evidence/9b
chmod 700 ~/lab-work/9b
kubectl config current-context
kubectl get nodes -o wide

W2='lab-k8s-worker2'
echo "node danh cho moi Pod dac quyen cua lab = $W2"
```

Ba hàm dưới đây được dùng ở gần như mọi bước sau:

```bash
# Apply mot manifest va giu lai CA stdout lan stderr — vi PSA tra loi tu choi qua stderr
# con canh bao cua che do warn cung qua stderr.
psa_try() {   # $1 = namespace  $2 = file manifest  $3 = file ket qua
  kubectl apply -n "$1" -f "$2" > "$3" 2>&1
}

# Doc mot truong trong /proc/1/status cua container — tien trinh chinh cua Pod.
proc_val() {  # $1 = namespace  $2 = pod  $3 = ten truong, khong ke dau hai cham
  kubectl -n "$1" exec "$2" -- awk -v f="$3:" '$1==f {print $2; exit}' /proc/1/status
}

# Cho toi khi mot truong cua Pod dat gia tri mong doi; in gia tri cuoi cung doc duoc.
wait_field() {  # $1 = ns  $2 = pod  $3 = jsonpath  $4 = gia tri mong doi
  i=0; V=''
  while [ "$i" -lt 90 ]; do
    V="$(kubectl -n "$1" get pod "$2" -o jsonpath="$3" 2>/dev/null)"
    [ "$V" = "$4" ] && { echo "$V"; return 0; }
    i=$(( i + 1 )); sleep 2
  done
  echo "$V"; return 1
}
```

```bash
test -d ~/lab-work/9b && test -d ~/lab-evidence/9b \
  && echo 'PASS: hai thu muc lam viec da san sang'
kubectl get node "$W2" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' \
  | grep -qx 'True' \
  && echo 'PASS: node dich cho Pod dac quyen ton tai va dang Ready'
```

**Ý nghĩa:** `wait_field` thay cho mọi con số chờ cố định. Thời gian kubelet chuyển một Pod sang
trạng thái lỗi **phụ thuộc cấu hình** — số vòng lặp chỉ là trần an toàn, không phải cam kết.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B0.2. Suy nhãn phiên bản chính sách từ chính cluster

Bài [116](../116-pod-security-admission-vi.md) cho phép ghim chính sách vào một phiên bản minor cụ
thể bằng nhãn `pod-security.kubernetes.io/<MODE>-version`, và bài
[283](../283-enforce-standards-namespace-labels-vi.md) khuyên ghim thay vì để `latest`. Con số
minor **không được chép tay** vào lab này — nó đọc từ API server:

```bash
K8S_MAJOR="$(kubectl get --raw /version | sed -n 's/.*"major": *"\([^"]*\)".*/\1/p')"
K8S_MINOR="$(kubectl get --raw /version | sed -n 's/.*"minor": *"\([^"]*\)".*/\1/p')"
GITV="$(kubectl get --raw /version | sed -n 's/.*"gitVersion": *"\([^"]*\)".*/\1/p')"
PSA_VER="v${K8S_MAJOR}.${K8S_MINOR}"
echo "gitVersion = $GITV | nhan phien ban chinh sach = $PSA_VER"
echo "$PSA_VER" > ~/lab-evidence/9b/b0-psa-version.txt
```

```bash
echo "$PSA_VER" | grep -qE '^v[0-9]+\.[0-9]+$' \
  && echo 'PASS: PSA_VER co dang vX.Y — khong phai placeholder'
case "$GITV" in
  "${PSA_VER}."*) echo 'PASS: PSA_VER dung la minor cua server dang chay' ;;
  *)              echo 'FAIL: PSA_VER khong khop gitVersion cua server' ;;
esac
```

**Ý nghĩa:** gate thứ hai là thứ ngăn lab trôi khỏi baseline. Nếu bạn khôi phục nhầm snapshot của
một cluster khác, `PSA_VER` sẽ khác và bạn biết ngay — thay vì phát hiện ở B2 khi câu thông báo
từ chối ghi một phiên bản chính sách lạ. Con số phiên bản của cluster nằm ở
[bảng A1.3 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa), không nằm ở đây.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B0.3. Năm namespace, bốn cấu hình chính sách khác nhau

Manifest dưới đây dùng đúng khuôn của bài [283](../283-enforce-standards-namespace-labels-vi.md) và
mục *Khởi tạo chính sách* của bài [115](../115-pod-security-standards-vi.md). Heredoc **không**
đóng ngoặc nháy, nên `$PSA_VER` được thay bằng giá trị vừa suy ra:

```bash
cat > ~/lab-work/9b/b0-namespace.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: lab-9b
  labels:
    lab: "9b"
---
apiVersion: v1
kind: Namespace
metadata:
  name: lab-9b-baseline
  labels:
    lab: "9b"
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: $PSA_VER
---
apiVersion: v1
kind: Namespace
metadata:
  name: lab-9b-restricted
  labels:
    lab: "9b"
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: $PSA_VER
---
apiVersion: v1
kind: Namespace
metadata:
  name: lab-9b-batche
  labels:
    lab: "9b"
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: $PSA_VER
    pod-security.kubernetes.io/warn: baseline
    pod-security.kubernetes.io/warn-version: $PSA_VER
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: $PSA_VER
---
apiVersion: v1
kind: Namespace
metadata:
  name: lab-9b-tenant
  labels:
    lab: "9b"
    tenant: doi-a
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: $PSA_VER
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: $PSA_VER
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: $PSA_VER
EOF

kubectl apply -f ~/lab-work/9b/b0-namespace.yaml
kubectl get namespace -l lab=9b --show-labels | tee ~/lab-evidence/9b/b0-namespace.txt
```

```bash
NS_OK=0
for n in lab-9b lab-9b-baseline lab-9b-restricted lab-9b-batche lab-9b-tenant; do
  test "$(kubectl get namespace "$n" -o jsonpath='{.status.phase}')" = 'Active' \
    && NS_OK=$(( NS_OK + 1 ))
done
echo "namespace Active = $NS_OK/5"
test "$NS_OK" -eq 5 && echo 'PASS: nam namespace cua lab da Active'

test -z "$(kubectl get namespace lab-9b \
  -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}')" \
  && echo 'PASS: lab-9b co chu dich KHONG mang nhan enforce'
test "$(kubectl get namespace lab-9b-batche \
  -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/warn}')" = 'baseline' \
  && test "$(kubectl get namespace lab-9b-batche \
       -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/audit}')" = 'restricted' \
  && echo 'PASS: lab-9b-batche mang ba che do o ba muc khac nhau'
```

**Ý nghĩa:** bốn cấu hình này là toàn bộ không gian thí nghiệm của lab. `lab-9b` cố ý **không có
nhãn nào** để B1.3 chứng minh mặc định mà bài
[282](../282-enforce-standards-admission-controller-vi.md) khai báo. `lab-9b-batche` là namespace
duy nhất mang cả ba chế độ, mỗi chế độ một mức — thứ mà bài
[116](../116-pod-security-admission-vi.md) nói là được phép nhưng ít người thử.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B0.4. Checksum cấu hình control plane và kubelet

Cám dỗ lớn nhất của một lab về policy là "bật tạm audit log để xem chế độ `audit` ghi gì". Bốn
dòng checksum dưới đây biến lời hứa ở [mục 2](#2-quy-ước-và-an-toàn) thành thứ kiểm chứng được,
đúng như [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) và
[Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) đã làm:

```bash
{
  echo "apiserver-manifest $(sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml \
    | awk '{print $1}')"
  echo "kubelet-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "kubelet-$n $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
} | tee ~/lab-evidence/9b/b0-cauhinh-sha.txt

test "$(wc -l < ~/lab-evidence/9b/b0-cauhinh-sha.txt)" -eq 4 \
  && test "$(awk '{print $2}' ~/lab-evidence/9b/b0-cauhinh-sha.txt \
       | grep -c '^[0-9a-f]\{64\}$')" -eq 4 \
  && echo 'PASS: ghi duoc checksum cua manifest apiserver va cau hinh kubelet ba node'
```

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B0.5. Tồn kho "trước": object phạm vi cluster, NetworkPolicy và Pod đặc quyền

Lab này **không** tạo object phạm vi cluster nào. Đếm trước để B10.2 chứng minh điều đó, và đếm
luôn số container đặc quyền đang có — cluster baseline vốn đã có vài cái trong `kube-system`:

```bash
CR_N0="$(kubectl get clusterrole --no-headers | wc -l)"
CRB_N0="$(kubectl get clusterrolebinding --no-headers | wc -l)"
NS_N0="$(kubectl get namespace --no-headers | wc -l)"
NP_N0="$(kubectl get networkpolicy -A --no-headers 2>/dev/null | wc -l)"
PRIV_N0="$(kubectl get pods -A \
  -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.securityContext.privileged}{"\n"}{end}{end}' \
  | grep -c '^true$' || true)"

{
  echo "=== $(date -Is) — ton kho truoc Lab 9b ==="
  echo "clusterrole=$CR_N0 clusterrolebinding=$CRB_N0 namespace=$NS_N0"
  echo "networkpolicy=$NP_N0 container_privileged=$PRIV_N0"
  echo '--- Pod co container privileged, theo namespace/ten ---'
  kubectl get pods -A \
    -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{" "}{range .spec.containers[*]}{.securityContext.privileged}{" "}{end}{"\n"}{end}' \
    | grep 'true' || echo '(khong co)'
} | tee ~/lab-evidence/9b/b0-ton-kho.txt
```

```bash
test "$CR_N0" -gt 0 && test "$CRB_N0" -gt 0 \
  && echo 'PASS: doc duoc so luong ClusterRole va ClusterRoleBinding hien co'
echo "$PRIV_N0" | grep -qE '^[0-9]+$' \
  && echo "PASS: dem duoc so container privileged truoc lab = $PRIV_N0"
test -z "$(kubectl get clusterrole,clusterrolebinding -l lab=9b -o name 2>/dev/null)" \
  && echo 'PASS: chua object pham vi cluster nao mang label lab=9b'
```

**Ý nghĩa:** con số `PRIV_N0` là thứ B10.2 so lại. Gate không đòi nó bằng 0 — CNI và kube-proxy
của baseline **phải** chạy đặc quyền, đó là ví dụ sống của câu trong bài
[115](../115-pod-security-standards-vi.md): profile *Privileged* dành cho workload cấp hệ thống và
hạ tầng, do người dùng đặc quyền quản lý. Điều lab phải chứng minh là **con số đó không tăng thêm
sau khi lab kết thúc**.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B1. Ba profile là định nghĩa, ba chế độ là label

**Mục đích:** dựng khung trước khi thử. Bài [115](../115-pod-security-standards-vi.md) định nghĩa
**ba profile**; bài [116](../116-pod-security-admission-vi.md) **áp** chúng bằng label; bài
[282](../282-enforce-standards-admission-controller-vi.md) nói điều gì xảy ra khi **không có**
label nào. B1 kiểm chứng cả ba khẳng định đó chỉ bằng thao tác đọc.

### B1.1. Không có object nào tên là "Pod Security Standard"

```bash
kubectl api-resources 2>/dev/null | grep -iE 'podsecurity|securitystandard' \
  > ~/lab-evidence/9b/b1-api-resource-podsecurity.txt || true
cat ~/lab-evidence/9b/b1-api-resource-podsecurity.txt

RS_N="$(grep -c . ~/lab-evidence/9b/b1-api-resource-podsecurity.txt || true)"
echo "so tai nguyen API co ten chua podsecurity = $RS_N"
test "$RS_N" -eq 0 \
  && echo 'PASS: khong tai nguyen API nao hien thuc ba profile — chung la dinh nghia, khong phai object'

kubectl explain pod.spec.securityContext --recursive \
  > ~/lab-evidence/9b/b1-explain-podsc.txt 2>&1
grep -q 'seccompProfile' ~/lab-evidence/9b/b1-explain-podsc.txt \
  && echo 'PASS: cac truong ma ba profile rang buoc thi co that trong API, o securityContext'
```

**Ý nghĩa:** đây là câu trong FAQ của bài [115](../115-pod-security-standards-vi.md) mà người học
hay bỏ qua. **Security context** là cấu hình trong manifest Pod, tức tham số truyền cho container
runtime — nó có mặt trong API. **Security profile** là cơ chế của *control plane* để thực thi
những thiết lập đó cùng các tham số liên quan nằm ngoài security context; nó **không** có object
riêng. Việc tách định nghĩa khỏi cách khởi tạo chính là mục *Khởi tạo chính sách* của bài: cùng ba
profile ấy, cơ chế thực thi có thể là Pod Security Admission tích hợp hoặc một policy engine bên
thứ ba.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B1.2. Hai nhóm label: mode và level

Bài [116](../116-pod-security-admission-vi.md) định nghĩa hai nhóm label — nhãn **mức theo từng
chế độ** và nhãn **phiên bản theo từng chế độ**. Đọc lại chính xác những gì B0.3 đã đặt:

```bash
kubectl get namespace -l lab=9b \
  -o custom-columns='NS:.metadata.name,ENFORCE:.metadata.labels.pod-security\.kubernetes\.io/enforce,WARN:.metadata.labels.pod-security\.kubernetes\.io/warn,AUDIT:.metadata.labels.pod-security\.kubernetes\.io/audit' \
  | tee ~/lab-evidence/9b/b1-ma-tran-nhan.txt
```

```bash
test "$(kubectl get namespace lab-9b-baseline \
  -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}')" = 'baseline' \
  && echo 'PASS: lab-9b-baseline enforce=baseline'
test "$(kubectl get namespace lab-9b-restricted \
  -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}')" = 'restricted' \
  && echo 'PASS: lab-9b-restricted enforce=restricted'
V_ENF="$(kubectl get namespace lab-9b-restricted \
  -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce-version}')"
test "$V_ENF" = "$PSA_VER" \
  && echo 'PASS: nhan phien ban da ghim dung minor cua cluster, khong phai latest'
```

**Ý nghĩa:** ghim phiên bản là khuyến nghị của bài
[283](../283-enforce-standards-namespace-labels-vi.md). Để `latest` nghĩa là nội dung của một mức
chính sách **tự thay đổi dưới chân bạn** khi cluster được nâng cấp lên minor mới có định nghĩa
chặt hơn — Pod đang chạy vẫn chạy, nhưng lần deploy sau đột nhiên bị từ chối mà không ai đổi gì
trong manifest.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B1.3. Namespace không có nhãn thì mặc định là gì

Bài [282](../282-enforce-standards-admission-controller-vi.md) ghi rõ bộ mặc định của admission
controller: `enforce`, `audit` và `warn` đều là `privileged`, phiên bản `latest`. Lệnh của bài
[283](../283-enforce-standards-namespace-labels-vi.md) liệt kê những namespace **chưa được đánh
giá tường minh**:

```bash
kubectl get namespaces --selector='!pod-security.kubernetes.io/enforce' \
  | tee ~/lab-evidence/9b/b1-ns-chua-danh-gia.txt

kubectl get namespaces --selector='!pod-security.kubernetes.io/enforce' -o name \
  | grep -qx 'namespace/lab-9b' \
  && echo 'PASS: lab-9b nam trong nhom namespace chua co enforce tuong minh'
```

Chứng minh mặc định ấy bằng một Pod, thay vì tin lời. Pod dưới đây **trượt cả Baseline lẫn
Restricted** — nó xin `SYS_ADMIN` và tắt seccomp:

```bash
cat > ~/lab-work/9b/b1-mac-dinh.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: mac-dinh
  labels:
    lab: "9b"
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        seccompProfile:
          type: Unconfined
        capabilities:
          add: ["SYS_ADMIN"]
YAMLEOF

psa_try lab-9b ~/lab-work/9b/b1-mac-dinh.yaml ~/lab-evidence/9b/b1-mac-dinh-ket-qua.txt
cat ~/lab-evidence/9b/b1-mac-dinh-ket-qua.txt
kubectl -n lab-9b wait --for=condition=Ready pod/mac-dinh --timeout=180s
```

```bash
if grep -qi 'violate' ~/lab-evidence/9b/b1-mac-dinh-ket-qua.txt; then
  echo 'FAIL: co canh bao hoac tu choi — lab-9b khong con o mac dinh privileged'
else
  echo 'PASS: khong mot canh bao nao — mac dinh cua ca ba che do dung la privileged'
fi
test "$(kubectl -n lab-9b get pod mac-dinh -o jsonpath='{.status.phase}')" = 'Running' \
  && echo 'PASS: Pod vi pham ca Baseline lan Restricted van chay trong namespace khong nhan'
```

**Ý nghĩa:** đây là mặt trái của thiết kế. Pod Security Admission **bật sẵn**, nhưng namespace
không có nhãn thì nó không cấm gì cả — bảo vệ chỉ bắt đầu khi ai đó dán nhãn. Vì thế lệnh
`--selector='!pod-security.kubernetes.io/enforce'` ở trên là một trong những lệnh rà soát đáng giá
nhất của giai đoạn này: nó liệt kê đúng những chỗ chính sách chưa với tới. Ô tương ứng trong
checklist [129](../129-security-checklist-vi.md) — *chính sách Pod Security Standards phù hợp được
áp dụng và thực thi cho tất cả namespace* — được rà lại đầy đủ ở B9.1.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`. Giữ Pod `mac-dinh` lại — B4.5 dùng nó
làm mốc đối chiếu capability mặc định.

---

## B2. Chế độ `enforce` — chặn ngay lúc tạo Pod

**Mục đích:** đi hết ranh giới giữa ba profile bằng thực nghiệm. Mỗi bước ở đây là **một câu thông
báo từ chối** phải đọc kỹ: nó ghi rõ mức chính sách, phiên bản chính sách, và **đúng trường** nào
gây trượt. Bảng đặc tả trong bài [115](../115-pod-security-standards-vi.md) là tài liệu tra cứu
cho những câu đó, không phải thứ để học thuộc.

### B2.1. Pod trần: Baseline nhận, và vì sao

Bài [115](../115-pod-security-standards-vi.md) mô tả Baseline là chính sách **hạn chế ở mức tối
thiểu, vẫn cho phép cấu hình Pod mặc định**. Kiểm chứng bằng một Pod không khai `securityContext`
nào:

```bash
cat > ~/lab-work/9b/b2-tran.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: tran
  labels:
    lab: "9b"
spec:
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
YAMLEOF

psa_try lab-9b-baseline ~/lab-work/9b/b2-tran.yaml ~/lab-evidence/9b/b2-tran-baseline.txt
cat ~/lab-evidence/9b/b2-tran-baseline.txt
kubectl -n lab-9b-baseline wait --for=condition=Ready pod/tran --timeout=180s
```

```bash
test "$(kubectl -n lab-9b-baseline get pod tran -o jsonpath='{.status.phase}')" = 'Running' \
  && echo 'PASS: Pod tran chay duoc trong namespace enforce=baseline'
grep -qi 'violate' ~/lab-evidence/9b/b2-tran-baseline.txt \
  || echo 'PASS: khong canh bao nao — cau hinh Pod mac dinh dat Baseline'
```

**Ý nghĩa:** đây là điểm bán hàng của Baseline. Bạn dán nhãn `enforce=baseline` cho một namespace
đang có workload viết theo kiểu thông thường, và phần lớn chúng **không phải sửa gì**. Restricted
thì ngược lại — bài nói thẳng nó đánh đổi bằng **tính tương thích**, và B2.3 sẽ cho thấy cái giá
đó bằng con số.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B2.2. Năm ranh giới của Baseline, năm lần bị từ chối

Năm Pod dưới đây mỗi cái vi phạm **đúng một** kiểm soát trong bảng Baseline: chia sẻ namespace của
host, container đặc quyền, volume `hostPath`, thêm capability ngoài danh sách cho phép, và đặt
seccomp profile thành `Unconfined`.

```bash
cat > ~/lab-work/9b/b2-vi-pham-baseline.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: vp-hostnet
  labels:
    lab: "9b"
spec:
  hostNetwork: true
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: vp-privileged
  labels:
    lab: "9b"
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        privileged: true
---
apiVersion: v1
kind: Pod
metadata:
  name: vp-hostpath
  labels:
    lab: "9b"
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  volumes:
    - name: tmp-cua-node
      hostPath:
        path: /tmp
        type: Directory
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: tmp-cua-node
          mountPath: /host-tmp
          readOnly: true
---
apiVersion: v1
kind: Pod
metadata:
  name: vp-cap
  labels:
    lab: "9b"
spec:
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        capabilities:
          add: ["SYS_ADMIN"]
---
apiVersion: v1
kind: Pod
metadata:
  name: vp-seccomp
  labels:
    lab: "9b"
spec:
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        seccompProfile:
          type: Unconfined
YAMLEOF

psa_try lab-9b-baseline ~/lab-work/9b/b2-vi-pham-baseline.yaml \
  ~/lab-evidence/9b/b2-vi-pham-baseline.txt
cat ~/lab-evidence/9b/b2-vi-pham-baseline.txt
```

Gate kiểm **năm lý do khác nhau**, chứ không chỉ kiểm "có lỗi":

```bash
LY_DO=0
for k in 'host namespaces' 'securityContext.privileged' 'hostPath volumes' \
         'non-default capabilities' 'Unconfined'; do
  if grep -q "$k" ~/lab-evidence/9b/b2-vi-pham-baseline.txt; then
    echo "co ly do: $k"; LY_DO=$(( LY_DO + 1 ))
  else
    echo "THIEU ly do: $k"
  fi
done
echo "so ly do tu choi doc duoc = $LY_DO/5"
test "$LY_DO" -eq 5 && echo 'PASS: du nam kiem soat cua Baseline deu tu tu choi mot Pod'

grep -c 'violates PodSecurity' ~/lab-evidence/9b/b2-vi-pham-baseline.txt
test "$(grep -c 'violates PodSecurity' ~/lab-evidence/9b/b2-vi-pham-baseline.txt)" -eq 5 \
  && echo 'PASS: nam cau tu choi rieng biet, moi Pod mot cau'

grep -q "baseline:$PSA_VER" ~/lab-evidence/9b/b2-vi-pham-baseline.txt \
  && echo 'PASS: cau tu choi ghi dung muc va dung phien ban chinh sach da ghim o B0.3'

test "$(kubectl -n lab-9b-baseline get pods --no-headers 2>/dev/null | wc -l)" -eq 1 \
  && echo 'PASS: khong Pod vi pham nao duoc tao — namespace van chi co Pod tran cua B2.1'
```

**Ý nghĩa:** dòng gate thứ ba là thứ nối B2 với B0.2. Câu từ chối ghi `baseline:<phiên bản>` — đó
chính là nhãn `enforce-version` bạn đã ghim, chứ không phải `latest`. Nếu nó ghi một minor khác,
namespace của bạn đang chịu một định nghĩa profile khác với định nghĩa bạn nghĩ.

Dòng gate cuối là điểm quan trọng nhất của cả chế độ `enforce`: Pod **không được tạo**. Nó không
`Pending`, không `CrashLoopBackOff`, không tồn tại. `kubectl get pods` sẽ không bao giờ cho bạn
thấy nó — bằng chứng duy nhất nằm ở câu thông báo mà client nhận được.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện, và không dòng `THIEU ly do:` nào.

### B2.3. Restricted thêm gì trên Baseline

Chính Pod `tran` của B2.1 — cái vừa được Baseline nhận không kêu ca — đem sang namespace
`enforce=restricted`:

```bash
psa_try lab-9b-restricted ~/lab-work/9b/b2-tran.yaml \
  ~/lab-evidence/9b/b2-tran-restricted.txt
cat ~/lab-evidence/9b/b2-tran-restricted.txt
```

```bash
THEM=0
for k in 'allowPrivilegeEscalation != false' 'unrestricted capabilities' \
         'runAsNonRoot != true' 'seccompProfile'; do
  if grep -q "$k" ~/lab-evidence/9b/b2-tran-restricted.txt; then
    echo "Restricted them: $k"; THEM=$(( THEM + 1 ))
  else
    echo "THIEU: $k"
  fi
done
echo "so rang buoc Restricted them vao = $THEM/4"
test "$THEM" -eq 4 \
  && echo 'PASS: dung mot manifest, Baseline nhan con Restricted tu choi vi bon ly do'
grep -q "restricted:$PSA_VER" ~/lab-evidence/9b/b2-tran-restricted.txt \
  && echo 'PASS: cau tu choi ghi muc restricted va dung phien ban da ghim'
test "$(kubectl -n lab-9b-restricted get pods --no-headers 2>/dev/null | wc -l)" -eq 0 \
  && echo 'PASS: namespace restricted van khong co Pod nao'
```

**Ý nghĩa:** bốn dòng vừa đếm được **là** câu trả lời cho câu hỏi "Restricted khác Baseline ở
đâu", và chúng đến từ chính API server chứ không từ trí nhớ. Chú ý điều bài
[115](../115-pod-security-standards-vi.md) nhấn mạnh về seccomp: ở Baseline chỉ cấm đặt tường minh
`Unconfined`; ở Restricted thì **không đặt cũng là vi phạm** — profile phải được đặt tường minh
thành `RuntimeDefault` hoặc `Localhost`. Đó là lý do một Pod chưa từng nhắc tới seccomp vẫn trượt.

**PASS:** ba dòng `PASS:` xuất hiện, và không dòng `THIEU:` nào.

### B2.4. Viết một Pod đạt Restricted, rồi đọc lại từ bên trong

Sửa đúng bốn thứ vừa đếm, cộng phần `runAsUser`, `runAsGroup`, `fsGroup`, `supplementalGroups` mà
bài [291](../291-security-context-vi.md) minh họa:

```bash
cat > ~/lab-work/9b/b2-dat-restricted.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: dat-restricted
  labels:
    lab: "9b"
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    supplementalGroups: [4000]
    seccompProfile:
      type: RuntimeDefault
  volumes:
    - name: sec-ctx-vol
      emptyDir: {}
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: sec-ctx-vol
          mountPath: /data/demo
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
YAMLEOF

psa_try lab-9b-restricted ~/lab-work/9b/b2-dat-restricted.yaml \
  ~/lab-evidence/9b/b2-dat-restricted.txt
cat ~/lab-evidence/9b/b2-dat-restricted.txt
kubectl -n lab-9b-restricted wait --for=condition=Ready pod/dat-restricted --timeout=180s
```

Đọc trạng thái thật của tiến trình, đúng ba phép đo của bài [291](../291-security-context-vi.md):

```bash
kubectl -n lab-9b-restricted exec dat-restricted -- id \
  | tee ~/lab-evidence/9b/b2-id.txt
kubectl -n lab-9b-restricted exec dat-restricted -- \
  sh -c 'echo hello > /data/demo/testfile; ls -l /data/demo' \
  | tee -a ~/lab-evidence/9b/b2-id.txt

U="$(kubectl -n lab-9b-restricted exec dat-restricted -- id -u)"
G="$(kubectl -n lab-9b-restricted exec dat-restricted -- id -g)"
GRPS="$(kubectl -n lab-9b-restricted exec dat-restricted -- id -G)"
DIR_G="$(kubectl -n lab-9b-restricted exec dat-restricted -- stat -c '%g' /data/demo)"
FILE_UG="$(kubectl -n lab-9b-restricted exec dat-restricted -- stat -c '%u:%g' /data/demo/testfile)"
echo "uid=$U gid=$G groups=$GRPS | /data/demo gid=$DIR_G | testfile=$FILE_UG"
```

```bash
test "$U" -eq 1000 && echo 'PASS: runAsUser co hieu luc — tien trinh chay duoi uid 1000'
test "$G" -eq 3000 && echo 'PASS: runAsGroup co hieu luc — gid chinh la 3000, khong phai 0'
echo "$GRPS" | grep -qw 2000 && echo "$GRPS" | grep -qw 4000 \
  && echo 'PASS: fsGroup va supplementalGroups deu nam trong danh sach group bo sung'
test "$DIR_G" -eq 2000 \
  && echo 'PASS: thu muc volume thuoc so huu cua group fsGroup'
test "$FILE_UG" = '1000:2000' \
  && echo 'PASS: file moi tao thuoc uid runAsUser va gid fsGroup'
```

**Ý nghĩa:** đây là Pod tham chiếu của cả lab — mọi workload trong namespace tenant ở B8 viết theo
đúng khuôn này. Ghi nhớ điều bài [291](../291-security-context-vi.md) nói: nếu bỏ `runAsGroup` thì
`gid` **vẫn là 0 (root)** và tiến trình tương tác được với mọi file thuộc group root. Bỏ một trường
không có nghĩa là "an toàn hơn theo mặc định".

**PASS:** năm dòng `PASS:` của bước này xuất hiện.

### B2.5. `enforce` không áp cho workload resource

Đây là điểm bất đối xứng dễ nhầm nhất của bài [116](../116-pod-security-admission-vi.md): chế độ
`audit` và `warn` áp cho cả workload resource, còn `enforce` **chỉ** áp cho Pod object được tạo ra.

```bash
cat > ~/lab-work/9b/b2-deploy-tho.yaml <<'YAMLEOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tho
  labels:
    lab: "9b"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tho
  template:
    metadata:
      labels:
        app: tho
        lab: "9b"
    spec:
      containers:
        - name: c
          image: busybox:1.37
          command: ["sh", "-c", "sleep 3600"]
YAMLEOF

psa_try lab-9b-restricted ~/lab-work/9b/b2-deploy-tho.yaml \
  ~/lab-evidence/9b/b2-deploy-tho.txt
cat ~/lab-evidence/9b/b2-deploy-tho.txt
```

```bash
grep -q 'deployment.apps/tho created' ~/lab-evidence/9b/b2-deploy-tho.txt \
  && echo 'PASS: Deployment duoc API server nhan binh thuong'
grep -qi 'violate' ~/lab-evidence/9b/b2-deploy-tho.txt \
  || echo 'PASS: khong canh bao nao luc apply — namespace nay chi bat enforce, khong bat warn'

RSN=''; i=0
while [ "$i" -lt 60 ]; do
  RSN="$(kubectl -n lab-9b-restricted get rs -l app=tho -o name 2>/dev/null | head -1)"
  [ -n "$RSN" ] && break
  i=$(( i + 1 )); sleep 2
done
MSG=''; i=0
while [ "$i" -lt 60 ]; do
  MSG="$(kubectl -n lab-9b-restricted get "$RSN" \
    -o jsonpath='{.status.conditions[?(@.type=="ReplicaFailure")].message}' 2>/dev/null)"
  [ -n "$MSG" ] && break
  i=$(( i + 1 )); sleep 2
done
echo "$MSG" | tee ~/lab-evidence/9b/b2-replicafailure.txt

echo "$MSG" | grep -q 'violates PodSecurity' \
  && echo 'PASS: ReplicaSet moi la cho bao loi — Pod bi tu choi o buoc controller tao Pod'
POD_N="$(kubectl -n lab-9b-restricted get pods -l app=tho --no-headers 2>/dev/null | wc -l)"
READY="$(kubectl -n lab-9b-restricted get deployment tho -o jsonpath='{.status.readyReplicas}')"
echo "pod cua deployment = $POD_N | readyReplicas = '${READY:-0}'"
test "$POD_N" -eq 0 && test -z "$READY" \
  && echo 'PASS: Deployment ton tai voi 0 Pod va 0 replica san sang'
```

**Ý nghĩa:** với một người vận hành, đây là kịch bản đau nhất. `kubectl apply` báo thành công,
CI/CD báo xanh, và không có gì chạy. Nếu namespace cũng bật `warn` ở cùng mức thì bạn nhận cảnh
báo **ngay lúc apply** — đó chính là lý do khuôn manifest của bài
[283](../283-enforce-standards-namespace-labels-vi.md) luôn đặt `warn` kèm `enforce`, và cũng là
lý do B3 tồn tại.

Dọn ngay để B10 không phải nhớ:

```bash
kubectl -n lab-9b-restricted delete deployment tho --wait=true
test "$(kubectl -n lab-9b-restricted get deployment --no-headers 2>/dev/null | wc -l)" -eq 0 \
  && echo 'PASS: da xoa Deployment tho'
```

**PASS:** năm dòng `PASS:` của bước này xuất hiện.

### B2.6. Cùng một manifest, hai namespace, hai kết quả

Chứng minh rằng thứ từ chối Pod ở B2.2 là **nhãn của namespace**, không phải manifest sai. Lấy
đúng Pod `hostPath` đã bị Baseline từ chối, đem sang `lab-9b` — namespace không nhãn:

```bash
cat > ~/lab-work/9b/b2-hostpath.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-tmp
  labels:
    lab: "9b"
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  volumes:
    - name: tmp-cua-node
      hostPath:
        path: /tmp
        type: Directory
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: tmp-cua-node
          mountPath: /host-tmp
          readOnly: true
YAMLEOF

psa_try lab-9b ~/lab-work/9b/b2-hostpath.yaml ~/lab-evidence/9b/b2-hostpath.txt
cat ~/lab-evidence/9b/b2-hostpath.txt
kubectl -n lab-9b wait --for=condition=Ready pod/hostpath-tmp --timeout=180s
```

```bash
test "$(kubectl -n lab-9b get pod hostpath-tmp -o jsonpath='{.status.phase}')" = 'Running' \
  && echo 'PASS: cung manifest do, namespace khong nhan thi Pod chay'
test "$(kubectl -n lab-9b get pod hostpath-tmp -o jsonpath='{.spec.nodeName}')" = "$W2" \
  && echo 'PASS: Pod mount hostPath nam dung tren lab-k8s-worker2'

kubectl -n lab-9b exec hostpath-tmp -- \
  sh -c 'touch /host-tmp/lab-9b-thu-ghi 2>&1; echo "rc=$?"' \
  | tee ~/lab-evidence/9b/b2-hostpath-ghi.txt
grep -q 'Read-only file system' ~/lab-evidence/9b/b2-hostpath-ghi.txt \
  && echo 'PASS: mount readOnly chan duoc chieu ghi vao filesystem cua node'
```

Xóa ngay, đúng quy ước ở [mục 2](#2-quy-ước-và-an-toàn):

```bash
kubectl -n lab-9b delete pod hostpath-tmp --wait=true
test -z "$(kubectl -n lab-9b get pod hostpath-tmp --ignore-not-found -o name)" \
  && echo 'PASS: Pod mount hostPath da bi xoa ngay trong buoc tao ra no'
ssh "$W2" 'test ! -e /tmp/lab-9b-thu-ghi && echo "PASS: khong file nao bi ghi vao /tmp cua node"'
```

**Ý nghĩa:** `readOnly: true` trên mọi `hostPath` mount là đúng khuyến nghị của bài
[128](../128-api-server-bypass-risks-vi.md) — nó nói `hostPath` mount phải là chỉ-đọc để giảm rủi
ro kẻ tấn công vượt qua các hạn chế về thư mục, và phải hạn chế hoặc cấm hẳn việc mount socket của
container runtime, dù trực tiếp hay qua thư mục cha. Baseline cấm `hostPath` **hoàn toàn**, và đó
là cách bảo vệ chắc chắn hơn nhiều so với việc tự nhớ đặt `readOnly`.

**PASS:** năm dòng `PASS:` của bước này xuất hiện.

---

## B3. Chế độ `warn` và `audit` — thấy vi phạm mà không chặn

**Mục đích:** dựng lại đủ ba hành vi mà bảng chế độ của bài
[116](../116-pod-security-admission-vi.md) mô tả, trên **một** namespace duy nhất mang ba chế độ ở
**ba mức khác nhau**: `enforce=privileged`, `warn=baseline`, `audit=restricted`.

### B3.1. `warn`: Pod được tạo, và bạn được cảnh báo

```bash
cat > ~/lab-work/9b/b3-canh-bao.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: canh-bao
  labels:
    lab: "9b"
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        privileged: true
YAMLEOF

psa_try lab-9b-batche ~/lab-work/9b/b3-canh-bao.yaml ~/lab-evidence/9b/b3-canh-bao.txt
cat ~/lab-evidence/9b/b3-canh-bao.txt
kubectl -n lab-9b-batche wait --for=condition=Ready pod/canh-bao --timeout=180s
```

```bash
grep -q 'would violate PodSecurity' ~/lab-evidence/9b/b3-canh-bao.txt \
  && echo 'PASS: che do warn in canh bao "would violate", khac han cau "violates" cua enforce'
grep -q "baseline:$PSA_VER" ~/lab-evidence/9b/b3-canh-bao.txt \
  && echo 'PASS: canh bao ghi dung muc baseline — dung muc dat cho warn, khong phai enforce'
test "$(kubectl -n lab-9b-batche get pod canh-bao -o jsonpath='{.status.phase}')" = 'Running' \
  && echo 'PASS: Pod van duoc tao va dang chay — warn khong chan'
test "$(kubectl -n lab-9b-batche get pod canh-bao -o jsonpath='{.spec.nodeName}')" = "$W2" \
  && echo 'PASS: Pod dac quyen nam dung tren lab-k8s-worker2'
```

**Ý nghĩa:** hai câu văn khác nhau ở đúng một thì của động từ — `violates` với `would violate` — và
đó là toàn bộ khác biệt giữa "đã bị chặn" và "sẽ bị chặn nếu bạn siết". Nhận ra được sự khác nhau
đó khi đọc log CI là kỹ năng thực dụng nhất của mục này.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B3.2. `warn` **có** áp cho workload resource

Cùng một Deployment không hợp lệ như B2.5, nhưng lần này trong namespace có bật `warn`:

```bash
sed 's/name: tho/name: tho-batche/; s/app: tho/app: tho-batche/' \
  ~/lab-work/9b/b2-deploy-tho.yaml > ~/lab-work/9b/b3-deploy-batche.yaml
grep -n 'name: tho-batche\|app: tho-batche' ~/lab-work/9b/b3-deploy-batche.yaml

psa_try lab-9b-batche ~/lab-work/9b/b3-deploy-batche.yaml \
  ~/lab-evidence/9b/b3-deploy-batche.txt
cat ~/lab-evidence/9b/b3-deploy-batche.txt
```

```bash
grep -q 'would violate PodSecurity' ~/lab-evidence/9b/b3-deploy-batche.txt \
  && echo 'PASS: warn canh bao NGAY LUC APPLY mot Deployment — dieu enforce khong lam'
grep -q 'created' ~/lab-evidence/9b/b3-deploy-batche.txt \
  && echo 'PASS: Deployment van duoc tao'

POD_N="$(kubectl -n lab-9b-batche get pods -l app=tho-batche --no-headers 2>/dev/null | wc -l)"
i=0
while [ "$i" -lt 60 ] && [ "$POD_N" -lt 2 ]; do
  POD_N="$(kubectl -n lab-9b-batche get pods -l app=tho-batche --no-headers 2>/dev/null | wc -l)"
  i=$(( i + 1 )); sleep 2
done
echo "so Pod cua tho-batche = $POD_N"
test "$POD_N" -eq 2 \
  && echo 'PASS: hai Pod van sinh ra — enforce=privileged khong chan gi ca'

kubectl -n lab-9b-batche delete deployment tho-batche --wait=true
test "$(kubectl -n lab-9b-batche get deployment --no-headers 2>/dev/null | wc -l)" -eq 0 \
  && echo 'PASS: da xoa Deployment tho-batche'
```

**Ý nghĩa:** đặt B2.5 cạnh B3.2 là có đủ bảng đối chiếu. `enforce` không nhìn thấy Pod template,
chỉ nhìn thấy Pod; `warn` nhìn thấy cả hai. Vì vậy `warn` là công cụ **phát hiện sớm**, còn
`enforce` là công cụ **ngăn chặn** — và một namespace production cần cả hai, đặt ở hai mức: siết
theo mức đang thực thi được, cảnh báo theo mức mình **mong muốn** đạt tới.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B3.3. `audit`: không chặn, không cảnh báo

Bước này cần một Pod **đạt Baseline nhưng trượt Restricted** — chính là Pod `tran` của B2.1. Trong
`lab-9b-batche`, nó không đụng `enforce` (privileged), không đụng `warn` (baseline), nhưng đụng
`audit` (restricted).

Đọc bộ đếm của chính chế độ `audit` trước và sau, dùng metric mà bài
[116](../116-pod-security-admission-vi.md) liệt kê:

```bash
psa_audit_deny() {
  kubectl get --raw /metrics \
    | awk '/^pod_security_evaluations_total\{/ && /mode="audit"/ && /decision="deny"/ {s+=$2} END {print s+0}'
}

AUD0="$(psa_audit_deny)"
echo "bo dem pod_security_evaluations_total mode=audit decision=deny truoc = $AUD0"

psa_try lab-9b-batche ~/lab-work/9b/b2-tran.yaml ~/lab-evidence/9b/b3-audit.txt
cat ~/lab-evidence/9b/b3-audit.txt
kubectl -n lab-9b-batche wait --for=condition=Ready pod/tran --timeout=180s

AUD1="$(psa_audit_deny)"
echo "bo dem sau = $AUD1"
```

```bash
grep -qi 'violate' ~/lab-evidence/9b/b3-audit.txt \
  || echo 'PASS: khong mot canh bao nao — audit im lang, dung nhu bang che do cua bai 116'
test "$(kubectl -n lab-9b-batche get pod tran -o jsonpath='{.status.phase}')" = 'Running' \
  && echo 'PASS: Pod van duoc tao va chay — audit khong chan'
test "$AUD1" -gt "$AUD0" \
  && echo 'PASS: bo dem danh gia o che do audit da tang — vi pham that su duoc ghi nhan'

{
  echo "=== $(date -Is) — phan cua che do audit KHONG kiem chung duoc o lab nay ==="
  echo 'Noi dung audit annotation can mot audit backend dang chay.'
  sudo grep -c 'audit-policy-file' /etc/kubernetes/manifests/kube-apiserver.yaml \
    | sed 's/^/co bao nhieu dong --audit-policy-file trong manifest apiserver: /'
} | tee ~/lab-evidence/9b/b3-audit-chua-kiem-chung.txt

test "$(sudo grep -c 'audit-policy-file' /etc/kubernetes/manifests/kube-apiserver.yaml)" -eq 0 \
  && echo 'PASS: apiserver KHONG bat audit log — ghi ro day la phan chua kiem chung duoc'
```

**Ý nghĩa:** ba thứ quan sát được đã đủ để phân biệt `audit` với hai chế độ kia: **không chặn**,
**không cảnh báo**, mà vẫn **có đánh giá xảy ra**. Thứ duy nhất còn thiếu là đọc chính audit
annotation, và nó cần một audit backend — tức phải thêm cờ `--audit-policy-file` cho
`kube-apiserver`. Lab này **cấm** làm việc đó, xem gate cuối cùng vừa chạy. Phần đó nằm ở
[giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu), bài
[306](../306-audit-vi.md).

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B3.4. Bảng sự thật của một namespace ba chế độ

Gom ba quan sát trên thành bảng, rồi ghi lại:

```bash
{
  echo "=== $(date -Is) — lab-9b-batche: enforce=privileged, warn=baseline, audit=restricted ==="
  printf '%-14s %-22s %-10s %-10s\n' 'POD' 'VI PHAM MUC NAO' 'DUOC TAO' 'CO CANH BAO'
  printf '%-14s %-22s %-10s %-10s\n' 'canh-bao' 'baseline + restricted' \
    "$(kubectl -n lab-9b-batche get pod canh-bao -o jsonpath='{.status.phase}')" 'co'
  printf '%-14s %-22s %-10s %-10s\n' 'tran' 'chi restricted' \
    "$(kubectl -n lab-9b-batche get pod tran -o jsonpath='{.status.phase}')" 'khong'
  echo '--- nhan that su dang co tren namespace ---'
  kubectl get namespace lab-9b-batche -o jsonpath='{.metadata.labels}' ; echo
} | tee ~/lab-evidence/9b/b3-bang-su-that.txt
```

```bash
test "$(grep -c 'Running' ~/lab-evidence/9b/b3-bang-su-that.txt)" -eq 2 \
  && echo 'PASS: ca hai Pod deu chay — khong Pod nao bi enforce=privileged chan'
grep -q '"pod-security.kubernetes.io/warn":"baseline"' ~/lab-evidence/9b/b3-bang-su-that.txt \
  && grep -q '"pod-security.kubernetes.io/audit":"restricted"' \
       ~/lab-evidence/9b/b3-bang-su-that.txt \
  && echo 'PASS: ba che do o ba muc cung ton tai tren mot namespace'
```

**Ý nghĩa:** đây là mô hình triển khai thực tế mà bài
[286](../286-migrate-from-psp-vi.md) mô tả ở bước *Kiểm chứng mức Pod Security*: siết `enforce` ở
mức bạn chắc chắn chạy được, đặt `warn` và `audit` ở mức bạn **muốn** đạt tới, rồi theo dõi một
thời gian trước khi nâng `enforce`. Chỉ có ba chế độ ở ba mức mới làm được điều đó.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B3.5. Thử trước khi siết: `--dry-run=server`

Bài [283](../283-enforce-standards-namespace-labels-vi.md) ghi chú rằng khi nhãn `enforce` được
thêm hoặc thay đổi, admission plugin **kiểm tra từng Pod trong namespace đó theo chính sách mới**
và trả vi phạm về dưới dạng cảnh báo. Với `--dry-run=server`, các kiểm tra vẫn chạy nhưng chính
sách **không** được cập nhật:

```bash
kubectl label --dry-run=server --overwrite ns lab-9b-batche \
  pod-security.kubernetes.io/enforce=baseline \
  > ~/lab-evidence/9b/b3-dryrun-mot-ns.txt 2>&1
cat ~/lab-evidence/9b/b3-dryrun-mot-ns.txt
```

```bash
grep -q 'violate the new PodSecurity enforce level' ~/lab-evidence/9b/b3-dryrun-mot-ns.txt \
  && echo 'PASS: dry-run soi lai Pod dang chay va bao chinh xac cai nao se bi chan'
grep -q 'canh-bao' ~/lab-evidence/9b/b3-dryrun-mot-ns.txt \
  && echo 'PASS: canh bao goi ten dung Pod dac quyen dang chay'
test "$(kubectl get namespace lab-9b-batche \
  -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}')" = 'privileged' \
  && echo 'PASS: nhan enforce KHONG doi — dry-run danh gia ma khong cap nhat chinh sach'
```

Rồi soi cả cluster, đúng lệnh của bài — vẫn `--dry-run=server`, nên **không namespace nào bị đổi**:

```bash
kubectl label --dry-run=server --overwrite ns --all \
  pod-security.kubernetes.io/enforce=baseline \
  > ~/lab-evidence/9b/b3-dryrun-tat-ca.txt 2>&1
grep 'violate the new PodSecurity enforce level' ~/lab-evidence/9b/b3-dryrun-tat-ca.txt \
  | tee ~/lab-evidence/9b/b3-dryrun-ns-vi-pham.txt
```

```bash
test "$(grep -c . ~/lab-evidence/9b/b3-dryrun-ns-vi-pham.txt)" -gt 0 \
  && echo 'PASS: it nhat mot namespace cua cluster co Pod se truot baseline'
ENF_SAU="$(kubectl get namespace lab-9b-baseline \
  -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}')"
test "$ENF_SAU" = 'baseline' && echo 'PASS: nhan cua lab-9b-baseline khong bi thay doi'
test -z "$(kubectl get namespace lab-9b \
  -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}')" \
  && echo 'PASS: lab-9b van khong co nhan enforce — --all voi dry-run khong ghi gi'
```

**Ý nghĩa:** lệnh vừa chạy là công cụ đánh giá tốt nhất trước khi siết một cluster đang chạy: nó
nói cho bạn biết **chính xác** namespace nào và Pod nào sẽ gãy, mà không gãy cái gì cả. Nhớ giới
hạn của nó, chính bài [286](../286-migrate-from-psp-vi.md) nêu: dry-run chỉ nhìn thấy Pod **đang
tồn tại**, nên nó bỏ sót CronJob chưa tới giờ, workload đã scale về 0 và những thứ chưa deploy. Đó
là lý do bài khuyên dùng thêm chế độ `audit` trong một khoảng theo dõi.

Và đây cũng là chỗ hiểu bước 1 của bài [286](../286-migrate-from-psp-vi.md) — *Rà soát quyền trên
namespace*: chính sách này được điều khiển bằng **label của Namespace**, nên ai `update`, `patch`
hay `create` được Namespace thì đổi được mức Pod Security của nó. B8.2 biến nhận xét đó thành một
gate RBAC thật.

**PASS:** sáu dòng `PASS:` của hai khối trên xuất hiện.

---

## B4. Security context nhìn từ bên trong container

**Mục đích:** mọi thứ ba profile ràng buộc đều là **trường trong manifest**, nhưng thứ thật sự bảo
vệ bạn là **trạng thái của tiến trình trong kernel**. B4 đo trạng thái đó. Image `busybox:1.37`
của baseline không có `capsh`, nên mọi phép đo đọc thẳng `/proc` — đúng cách bài
[291](../291-security-context-vi.md) làm.

Bốn trường được đọc suốt mục này:

| Trường trong `/proc/1/status` | Nói lên điều gì |
| --- | --- |
| `CapPrm` | bitmap capability tiến trình **được phép** dùng; bit thứ *n* ứng với capability số *n* |
| `NoNewPrivs` | cờ `no_new_privs` — `allowPrivilegeEscalation` quyết định trực tiếp giá trị này |
| `Seccomp` | `0` không lọc, `2` đang chạy ở chế độ filter |
| `/proc/self/attr/current` | profile AppArmor đang áp cho tiến trình |

### B4.1. Mốc đối chiếu: một container không khai gì cả

Pod `tran` ở `lab-9b-baseline` (tạo từ B2.1) là container mặc định — không một dòng
`securityContext` nào:

```bash
CAP_TRAN="$(proc_val lab-9b-baseline tran CapPrm)"
NNP_TRAN="$(proc_val lab-9b-baseline tran NoNewPrivs)"
SEC_TRAN="$(proc_val lab-9b-baseline tran Seccomp)"
AA_TRAN="$(kubectl -n lab-9b-baseline exec tran -- cat /proc/self/attr/current)"
UID_TRAN="$(kubectl -n lab-9b-baseline exec tran -- id -u)"

{
  echo "=== $(date -Is) — moc doi chieu: container khong khai securityContext ==="
  echo "CapPrm      = $CAP_TRAN  (thap phan $(( 0x$CAP_TRAN )))"
  echo "NoNewPrivs  = $NNP_TRAN"
  echo "Seccomp     = $SEC_TRAN"
  echo "AppArmor    = $AA_TRAN"
  echo "uid         = $UID_TRAN"
} | tee ~/lab-evidence/9b/b4-moc-doi-chieu.txt
```

```bash
echo "$CAP_TRAN" | grep -qE '^[0-9a-f]{16}$' \
  && echo 'PASS: doc duoc bitmap capability cua container mac dinh'
test "$(( 0x$CAP_TRAN ))" -gt 0 \
  && echo 'PASS: container mac dinh VAN co mot tap capability — no khong phai zero'
test "$NNP_TRAN" -eq 0 \
  && echo 'PASS: NoNewPrivs = 0 — mac dinh KHONG chan leo thang dac quyen'
test "$UID_TRAN" -eq 0 \
  && echo 'PASS: container mac dinh chay bang root trong container'
```

**Ý nghĩa:** bốn con số này là lý do Restricted tồn tại. Một container "bình thường", không khai
gì, vẫn chạy bằng root, vẫn giữ một tập capability, và vẫn cho phép leo thang đặc quyền. Baseline
chấp nhận trạng thái đó — nó chỉ chặn những cách leo thang **đã biết**. Restricted thì bắt bạn đóng
từng cái một, và mỗi bước dưới đây đóng đúng một cái.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B4.2. `securityContext` cấp container ghi đè cấp Pod

```bash
cat > ~/lab-work/9b/b4-ghi-de.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: ghi-de
  labels:
    lab: "9b"
spec:
  securityContext:
    runAsUser: 1000
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        runAsUser: 2000
        allowPrivilegeEscalation: false
YAMLEOF

kubectl apply -n lab-9b-baseline -f ~/lab-work/9b/b4-ghi-de.yaml
kubectl -n lab-9b-baseline wait --for=condition=Ready pod/ghi-de --timeout=180s

U_POD="$(kubectl -n lab-9b-baseline get pod ghi-de \
  -o jsonpath='{.spec.securityContext.runAsUser}')"
U_CTR="$(kubectl -n lab-9b-baseline get pod ghi-de \
  -o jsonpath='{.spec.containers[0].securityContext.runAsUser}')"
U_THAT="$(kubectl -n lab-9b-baseline exec ghi-de -- id -u)"
echo "manifest: pod=$U_POD container=$U_CTR | tien trinh that su chay duoi uid=$U_THAT"
```

```bash
test "$U_POD" -eq 1000 && test "$U_CTR" -eq 2000 \
  && echo 'PASS: manifest khai hai gia tri khac nhau o hai cap'
test "$U_THAT" -eq 2000 \
  && echo 'PASS: gia tri cap container thang — no ghi de gia tri cap Pod'
```

**Ý nghĩa:** quy tắc của bài [291](../291-security-context-vi.md): thiết lập cấp Container chỉ áp
cho Container đó và **ghi đè** thiết lập cấp Pod khi trùng lặp; đổi lại, nó **không** ảnh hưởng tới
Volume của Pod. Đó là lý do `fsGroup` và `seLinuxOptions` chỉ tồn tại ở cấp Pod: chúng tác động lên
volume, thứ dùng chung cho cả Pod.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B4.3. `runAsNonRoot` chặn một image vốn chạy bằng root

`runAsNonRoot: true` không **đổi** user; nó **từ chối khởi chạy** nếu image sẽ chạy bằng root:

```bash
cat > ~/lab-work/9b/b4-nonroot-loi.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: nonroot-loi
  labels:
    lab: "9b"
spec:
  securityContext:
    runAsNonRoot: true
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
YAMLEOF

kubectl apply -n lab-9b -f ~/lab-work/9b/b4-nonroot-loi.yaml

wait_field lab-9b nonroot-loi \
  '{.status.containerStatuses[0].state.waiting.reason}' 'CreateContainerConfigError'
kubectl -n lab-9b get pod nonroot-loi -o wide
kubectl -n lab-9b describe pod nonroot-loi | tail -20 \
  | tee ~/lab-evidence/9b/b4-nonroot-loi.txt
```

```bash
RS_NR="$(kubectl -n lab-9b get pod nonroot-loi \
  -o jsonpath='{.status.containerStatuses[0].state.waiting.reason}')"
MSG_NR="$(kubectl -n lab-9b get pod nonroot-loi \
  -o jsonpath='{.status.containerStatuses[0].state.waiting.message}')"
echo "reason=$RS_NR"; echo "message=$MSG_NR"

test "$RS_NR" = 'CreateContainerConfigError' \
  && echo 'PASS: container khong khoi chay duoc — loi o buoc dung cau hinh container'
echo "$MSG_NR" | grep -q 'runAsNonRoot' \
  && echo 'PASS: thong bao chi dung nguyen nhan la runAsNonRoot'
test -n "$(kubectl -n lab-9b get pod nonroot-loi -o jsonpath='{.spec.nodeName}')" \
  && echo 'PASS: Pod VAN duoc lap lich len mot node — no chet o kubelet, khong o admission'
```

**Ý nghĩa:** ba dòng gate trên chia ranh giới cho cả phần còn lại của lab. Pod Security Admission
chặn ở **API server**, trước khi object tồn tại. `runAsNonRoot` chặn ở **kubelet**, sau khi Pod đã
có tên, đã được lập lịch, đã hiện ra trong `kubectl get pods`. Hai mặt phẳng khác nhau, hai cách
điều tra khác nhau. B5.3 sẽ dùng lại đúng sự phân biệt này với sysctl.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B4.4. `allowPrivilegeEscalation` và cờ `no_new_privs`

```bash
cat > ~/lab-work/9b/b4-pe-tat.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: pe-tat
  labels:
    lab: "9b"
spec:
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        allowPrivilegeEscalation: false
YAMLEOF

kubectl apply -n lab-9b-baseline -f ~/lab-work/9b/b4-pe-tat.yaml
kubectl -n lab-9b-baseline wait --for=condition=Ready pod/pe-tat --timeout=180s

NNP_TAT="$(proc_val lab-9b-baseline pe-tat NoNewPrivs)"
echo "NoNewPrivs: tran=$NNP_TRAN  pe-tat=$NNP_TAT" \
  | tee ~/lab-evidence/9b/b4-nonewprivs.txt
```

```bash
test "$NNP_TAT" -eq 1 \
  && echo 'PASS: allowPrivilegeEscalation=false dat co no_new_privs len 1 trong kernel'
test "$NNP_TRAN" -eq 0 && test "$NNP_TAT" -ne "$NNP_TRAN" \
  && echo 'PASS: hai container chi khac nhau mot truong, va gia tri trong kernel khac nhau'
```

**Ý nghĩa:** bài [291](../291-security-context-vi.md) nói trường boolean này **trực tiếp** quyết
định cờ `no_new_privs` của tiến trình. Bài [127](../127-linux-kernel-security-vi.md) bổ sung nó
chặn đúng hai việc: tiến trình **không giành thêm capability mới**, và người dùng không có đặc
quyền **không đổi được seccomp profile đang áp sang một profile dễ dãi hơn**. Cũng nhớ mặt còn lại
của cùng câu đó: `allowPrivilegeEscalation` **luôn** là `true` khi container chạy `privileged`
hoặc có `CAP_SYS_ADMIN` — B4.8 sẽ cho thấy điều đó.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B4.5. Capability: drop `ALL` rồi thêm lại đúng một cái

```bash
cat > ~/lab-work/9b/b4-cap.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: cap-drop
  labels:
    lab: "9b"
spec:
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        capabilities:
          drop: ["ALL"]
---
apiVersion: v1
kind: Pod
metadata:
  name: cap-add
  labels:
    lab: "9b"
spec:
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        capabilities:
          drop: ["ALL"]
          add: ["NET_BIND_SERVICE"]
YAMLEOF

kubectl apply -n lab-9b-baseline -f ~/lab-work/9b/b4-cap.yaml
kubectl -n lab-9b-baseline wait --for=condition=Ready pod/cap-drop pod/cap-add --timeout=180s

CAP_DROP="$(proc_val lab-9b-baseline cap-drop CapPrm)"
CAP_ADD="$(proc_val lab-9b-baseline cap-add CapPrm)"
{
  echo "mac dinh   CapPrm=$CAP_TRAN -> $(( 0x$CAP_TRAN ))"
  echo "drop ALL   CapPrm=$CAP_DROP -> $(( 0x$CAP_DROP ))"
  echo "them 1 cai CapPrm=$CAP_ADD  -> $(( 0x$CAP_ADD ))"
} | tee ~/lab-evidence/9b/b4-capability.txt
```

```bash
test "$(( 0x$CAP_DROP ))" -eq 0 \
  && echo 'PASS: drop ALL lam bitmap capability ve dung 0'
test "$(( 0x$CAP_ADD ))" -eq 1024 \
  && echo 'PASS: chi con dung bit 10 duoc bat — 2^10 = 1024, tuc CAP_NET_BIND_SERVICE'
test "$(( 0x$CAP_TRAN ))" -gt "$(( 0x$CAP_ADD ))" \
  && echo 'PASS: container mac dinh giu NHIEU capability hon container da drop ALL'
```

**Ý nghĩa:** so **giá trị** chứ không so chuỗi là điểm mấu chốt ở đây. Bài
[291](../291-security-context-vi.md) so hai bitmap bằng mắt và chỉ ra bit 12 là `CAP_NET_ADMIN`,
bit 25 là `CAP_SYS_TIME`; lab làm cùng việc đó bằng số học, nên gate không phụ thuộc độ rộng chuỗi
hex mà kernel in ra. `NET_BIND_SERVICE` cũng chính là **capability duy nhất** Restricted cho phép
thêm lại sau khi drop `ALL` — nó cần cho tiến trình non-root bind vào cổng dưới 1024.

Nhớ quy ước đặt tên: hằng số kernel là `CAP_NET_BIND_SERVICE`, còn trong manifest phải bỏ tiền tố
`CAP_`.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B4.6. seccomp và AppArmor: hai cơ chế, hai chỗ đọc

```bash
cat > ~/lab-work/9b/b4-seccomp-rd.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: sec-rd
  labels:
    lab: "9b"
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
YAMLEOF

cat > ~/lab-work/9b/b4-seccomp-unc.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: sec-unc
  labels:
    lab: "9b"
spec:
  securityContext:
    seccompProfile:
      type: Unconfined
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
YAMLEOF

kubectl apply -n lab-9b-baseline -f ~/lab-work/9b/b4-seccomp-rd.yaml
kubectl apply -n lab-9b -f ~/lab-work/9b/b4-seccomp-unc.yaml
kubectl -n lab-9b-baseline wait --for=condition=Ready pod/sec-rd --timeout=180s
kubectl -n lab-9b wait --for=condition=Ready pod/sec-unc --timeout=180s

S_RD="$(proc_val lab-9b-baseline sec-rd Seccomp)"
S_UNC="$(proc_val lab-9b sec-unc Seccomp)"
AA_RD="$(kubectl -n lab-9b-baseline exec sec-rd -- cat /proc/self/attr/current)"
{
  echo "seccompProfile RuntimeDefault -> Seccomp=$S_RD"
  echo "seccompProfile Unconfined     -> Seccomp=$S_UNC"
  echo "AppArmor cua container thuong -> $AA_RD"
} | tee ~/lab-evidence/9b/b4-seccomp-apparmor.txt
```

```bash
test "$S_RD" -eq 2 \
  && echo 'PASS: RuntimeDefault dat tien trinh vao che do seccomp filter'
test "$S_UNC" -eq 0 \
  && echo 'PASS: Unconfined tat han bo loc syscall'
test "$S_RD" -ne "$S_UNC" \
  && echo 'PASS: hai gia tri khac nhau — profile trong manifest that su toi duoc kernel'
echo "$AA_RD" | grep -q '(enforce)' \
  && echo 'PASS: container thuong dang chay duoi mot AppArmor profile o che do enforce'
```

**Ý nghĩa:** hai cơ chế nhắm hai đối tượng khác nhau, đúng như bài
[127](../127-linux-kernel-security-vi.md) phân biệt: **seccomp** lọc từng **syscall**; **AppArmor**
hạn chế **quyền truy cập tài nguyên của một chương trình** và định nghĩa tài nguyên bằng **đường
dẫn file**. Profile AppArmor bạn vừa đọc là profile mặc định do container runtime cung cấp — đúng
khuyến nghị của bài: **dùng profile mặc định của runtime**, đừng tự viết trừ khi thật sự cần kiểm
soát chi tiết, vì profile tự viết hỏng khi ứng dụng cập nhật và rất khó quản lý ở quy mô lớn.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B4.7. `readOnlyRootFilesystem`

```bash
cat > ~/lab-work/9b/b4-ro-root.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: ro-root
  labels:
    lab: "9b"
spec:
  volumes:
    - name: cho-ghi
      emptyDir: {}
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: cho-ghi
          mountPath: /tmp
      securityContext:
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
YAMLEOF

kubectl apply -n lab-9b-baseline -f ~/lab-work/9b/b4-ro-root.yaml
kubectl -n lab-9b-baseline wait --for=condition=Ready pod/ro-root --timeout=180s

kubectl -n lab-9b-baseline exec ro-root -- \
  sh -c 'touch /thu-ghi-root 2>&1; echo "rc_root=$?"; touch /tmp/thu-ghi-tmp 2>&1; echo "rc_tmp=$?"' \
  | tee ~/lab-evidence/9b/b4-ro-root.txt
```

```bash
grep -q 'Read-only file system' ~/lab-evidence/9b/b4-ro-root.txt \
  && echo 'PASS: khong ghi duoc vao root filesystem cua container'
grep -q 'rc_tmp=0' ~/lab-evidence/9b/b4-ro-root.txt \
  && echo 'PASS: van ghi duoc vao emptyDir mount o /tmp — dung cach lam viec that te'
grep -q 'rc_root=0' ~/lab-evidence/9b/b4-ro-root.txt \
  && echo 'FAIL: root filesystem van ghi duoc — readOnlyRootFilesystem khong co hieu luc'
```

**Ý nghĩa:** đây là ô *Cấu hình root filesystem ở chế độ chỉ đọc* trong checklist
[130](../130-application-security-checklist-vi.md). Chú ý điều Pod này chứng minh: bật cờ đó gần
như luôn phải đi kèm một volume ghi được cho những thư mục ứng dụng thật sự cần — nếu không, ứng
dụng chết ngay lần ghi file tạm đầu tiên. Đó cũng là lý do **Restricted không đòi**
`readOnlyRootFilesystem`: nó là hardening tốt nhưng phá tương thích quá mạnh để đưa vào một profile
chuẩn.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B4.8. Container đặc quyền xóa sạch cả ba ràng buộc

Pod dưới đây khai `seccompProfile: RuntimeDefault` **và** `privileged: true` cùng lúc. Đây là thí
nghiệm quan trọng nhất của B4: nó cho biết khi hai thứ mâu thuẫn thì cái nào thắng.

```bash
cat > ~/lab-work/9b/b4-dacquyen.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: dacquyen
  labels:
    lab: "9b"
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        privileged: true
YAMLEOF

kubectl apply -n lab-9b -f ~/lab-work/9b/b4-dacquyen.yaml
kubectl -n lab-9b wait --for=condition=Ready pod/dacquyen --timeout=180s

CAP_PRIV="$(proc_val lab-9b dacquyen CapPrm)"
S_PRIV="$(proc_val lab-9b dacquyen Seccomp)"
NNP_PRIV="$(proc_val lab-9b dacquyen NoNewPrivs)"
AA_PRIV="$(kubectl -n lab-9b exec dacquyen -- cat /proc/self/attr/current)"
{
  echo "=== $(date -Is) — container privileged=true, khai seccompProfile RuntimeDefault ==="
  echo "CapPrm     = $CAP_PRIV -> $(( 0x$CAP_PRIV ))   (mac dinh: $(( 0x$CAP_TRAN )))"
  echo "Seccomp    = $S_PRIV                            (khai RuntimeDefault, sec-rd do duoc $S_RD)"
  echo "NoNewPrivs = $NNP_PRIV"
  echo "AppArmor   = $AA_PRIV"
  echo "node       = $(kubectl -n lab-9b get pod dacquyen -o jsonpath='{.spec.nodeName}')"
} | tee ~/lab-evidence/9b/b4-dacquyen.txt
```

```bash
test "$(( 0x$CAP_PRIV ))" -gt "$(( 0x$CAP_TRAN ))" \
  && echo 'PASS: container dac quyen giu NHIEU capability hon container mac dinh'
test "$S_PRIV" -eq 0 \
  && echo 'PASS: seccomp bi TAT du manifest khai RuntimeDefault — privileged ghi de'
echo "$AA_PRIV" | grep -q 'unconfined' \
  && echo 'PASS: AppArmor profile bi bo qua — tien trinh chay unconfined'
test "$(kubectl -n lab-9b get pod dacquyen -o jsonpath='{.spec.nodeName}')" = "$W2" \
  && echo 'PASS: Pod dac quyen nam dung tren lab-k8s-worker2 theo quy uoc muc 2'
```

**Ý nghĩa:** ba dòng gate đầu là ba gạch đầu dòng của bài
[127](../127-linux-kernel-security-vi.md), đo được bằng số: container đặc quyền chạy với seccomp
`Unconfined` **ghi đè** profile bạn chỉ định, **bỏ qua** mọi AppArmor profile, và được cấp toàn bộ
Linux capability **kể cả những cái nó không cần**. Bài khuyên đúng một việc: cấp riêng capability
cần thiết qua trường `capabilities`, và chỉ dùng chế độ đặc quyền khi cần một capability không cấp
được qua `securityContext`.

Với người vận hành, hệ quả rất cụ thể: một manifest có `privileged: true` thì mọi dòng hardening
khác trong cùng manifest chỉ còn là trang trí. Đó là lý do Baseline cấm nó ngay từ mức thấp nhất.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B4.9. Dọn phần B4

Xóa Pod đặc quyền **ngay khi đo xong**, không để nó sống tới cuối lab:

```bash
kubectl -n lab-9b delete pod dacquyen nonroot-loi sec-unc --wait=true --ignore-not-found
kubectl -n lab-9b-baseline delete pod ghi-de pe-tat cap-drop cap-add sec-rd ro-root \
  --wait=true --ignore-not-found

PRIV_CON="$(kubectl get pods -n lab-9b \
  -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.securityContext.privileged}{"\n"}{end}{end}' \
  | grep -c '^true$' || true)"
echo "container privileged con lai trong lab-9b = $PRIV_CON"
test "$PRIV_CON" -eq 0 \
  && echo 'PASS: khong con container dac quyen nao trong lab-9b'
test "$(kubectl -n lab-9b-baseline get pods --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: lab-9b-baseline tro ve dung mot Pod tran cua B2.1'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B5. `sysctl` trong cluster: an toàn, không an toàn, và hai mặt phẳng cưỡng chế

**Mục đích:** bài [258](../258-sysctl-cluster-vi.md) chia sysctl thành hai nhóm và nói rõ hậu quả
của từng nhóm. B5 dựng lại cả hai, cộng phần `/proc` mà bài
[291](../291-security-context-vi.md) mô tả — thứ giải thích **vì sao** phải có trường `sysctls`
thay vì ghi thẳng vào `/proc/sys`.

### B5.1. Vì sao không ghi thẳng vào `/proc/sys`

Dựng lại Pod `ro-root` của B4.7 trong `lab-9b-baseline` để có chỗ thử:

```bash
kubectl apply -n lab-9b-baseline -f ~/lab-work/9b/b4-ro-root.yaml
kubectl -n lab-9b-baseline wait --for=condition=Ready pod/ro-root --timeout=180s

kubectl -n lab-9b-baseline exec ro-root -- sh -c '
  echo "doc duoc: $(cat /proc/sys/net/ipv4/ip_unprivileged_port_start)"
  echo 100 > /proc/sys/net/ipv4/ip_unprivileged_port_start 2>&1
  echo "rc_ghi=$?"
  echo "kich thuoc /proc/kcore = $(stat -c %s /proc/kcore)"
' | tee ~/lab-evidence/9b/b5-proc.txt
```

```bash
grep -q 'doc duoc: [0-9]' ~/lab-evidence/9b/b5-proc.txt \
  && echo 'PASS: /proc/sys doc duoc binh thuong tu trong container'
grep -qE 'Read-only file system|rc_ghi=[1-9]' ~/lab-evidence/9b/b5-proc.txt \
  && echo 'PASS: /proc/sys KHONG ghi duoc — no nam trong danh sach duong dan chi doc cua OCI'
test "$(grep -o 'kich thuoc /proc/kcore = [0-9]*' ~/lab-evidence/9b/b5-proc.txt \
  | awk '{print $NF}')" -eq 0 \
  && echo 'PASS: /proc/kcore bi che — dung danh sach masked path cua bai 291'
```

**Ý nghĩa:** bài [291](../291-security-context-vi.md) liệt kê đúng hai danh sách này. `/proc/sys`
nằm trong nhóm **chỉ đọc**, `/proc/kcore` nằm trong nhóm **bị che**. Chúng có mặt trong mount
namespace của container nên trông như thật, nhưng tiến trình không ghi được vào — đó là lý do
Kubernetes phải có một đường riêng, `spec.securityContext.sysctls`, để đặt tham số kernel cho Pod.
Trường `procMount: Unmasked` mở được hai danh sách này, nhưng nó đòi `hostUsers: false`, tức phải
nằm trong user namespace — nằm ngoài lab này, xem bảng lý do ở
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B5.2. Sysctl an toàn: đổi trong Pod, không đổi trên node

```bash
cat > ~/lab-work/9b/b5-antoan.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-antoan
  labels:
    lab: "9b"
spec:
  securityContext:
    sysctls:
      - name: net.ipv4.ip_unprivileged_port_start
        value: "80"
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
YAMLEOF

psa_try lab-9b-baseline ~/lab-work/9b/b5-antoan.yaml ~/lab-evidence/9b/b5-antoan.txt
cat ~/lab-evidence/9b/b5-antoan.txt
kubectl -n lab-9b-baseline wait --for=condition=Ready pod/sysctl-antoan --timeout=180s

NODE_CUA_POD="$(kubectl -n lab-9b-baseline get pod sysctl-antoan -o jsonpath='{.spec.nodeName}')"
TRONG_POD="$(kubectl -n lab-9b-baseline exec sysctl-antoan -- \
  cat /proc/sys/net/ipv4/ip_unprivileged_port_start)"
TREN_NODE="$(ssh "$NODE_CUA_POD" 'sysctl -n net.ipv4.ip_unprivileged_port_start')"
echo "node=$NODE_CUA_POD | trong Pod=$TRONG_POD | tren node=$TREN_NODE" \
  | tee ~/lab-evidence/9b/b5-antoan-gia-tri.txt
```

```bash
test "$TRONG_POD" -eq 80 \
  && echo 'PASS: sysctl an toan da co hieu luc BEN TRONG Pod'
test "$TREN_NODE" -ne "$TRONG_POD" \
  && echo 'PASS: gia tri tren node KHONG bi doi — sysctl nay duoc namespace hoa'
grep -qi 'violate' ~/lab-evidence/9b/b5-antoan.txt \
  || echo 'PASS: Baseline chap nhan — ten sysctl nay nam trong danh sach cho phep cua no'
```

**Ý nghĩa:** đây là đúng định nghĩa "an toàn" của bài [258](../258-sysctl-cluster-vi.md): sysctl
được namespace hóa và **cách ly đủ mức giữa các Pod trên cùng một node** — đặt cho Pod này không
ảnh hưởng Pod khác, không hại sức khỏe node, không giành thêm CPU hay bộ nhớ ngoài giới hạn của
Pod. Hai giá trị khác nhau bạn vừa đo **là** bằng chứng của tính cách ly đó.

Chú ý cách hai bài khớp nhau: danh sách sysctl an toàn của bài
[258](../258-sysctl-cluster-vi.md) chính là danh sách giá trị được phép trong ô *Sysctls* của bảng
Baseline ở bài [115](../115-pod-security-standards-vi.md). Và nhớ ngoại lệ bài nêu: sysctl `net.*`
**không được dùng khi bật host networking** — Pod này không dùng `hostNetwork`, nên không vướng.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B5.3. Sysctl không an toàn: cùng một Pod, hai cách chết

```bash
cat > ~/lab-work/9b/b5-khong-antoan.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: sysctl-khong-antoan
  labels:
    lab: "9b"
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  securityContext:
    sysctls:
      - name: net.core.somaxconn
        value: "1024"
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
YAMLEOF
```

Mặt phẳng thứ nhất — namespace **có** nhãn Baseline, Pod chết ở admission:

```bash
psa_try lab-9b-baseline ~/lab-work/9b/b5-khong-antoan.yaml \
  ~/lab-evidence/9b/b5-khong-antoan-admission.txt
cat ~/lab-evidence/9b/b5-khong-antoan-admission.txt
```

```bash
grep -q 'violates PodSecurity' ~/lab-evidence/9b/b5-khong-antoan-admission.txt \
  && echo 'PASS: Baseline tu choi ngay o admission'
grep -q 'forbidden sysctls' ~/lab-evidence/9b/b5-khong-antoan-admission.txt \
  && echo 'PASS: ly do ghi ro la sysctl bi cam, kem ten sysctl'
test -z "$(kubectl -n lab-9b-baseline get pod sysctl-khong-antoan \
  --ignore-not-found -o name)" \
  && echo 'PASS: Pod khong he ton tai trong namespace do'
```

Mặt phẳng thứ hai — namespace **không** nhãn, Pod được tạo, được lập lịch, rồi kubelet từ chối:

```bash
kubectl apply -n lab-9b -f ~/lab-work/9b/b5-khong-antoan.yaml
wait_field lab-9b sysctl-khong-antoan '{.status.reason}' 'SysctlForbidden'
kubectl -n lab-9b get pod sysctl-khong-antoan -o wide \
  | tee ~/lab-evidence/9b/b5-khong-antoan-kubelet.txt
kubectl -n lab-9b get pod sysctl-khong-antoan \
  -o jsonpath='{.status.reason}{"|"}{.status.message}{"|"}{.spec.nodeName}{"\n"}' \
  | tee -a ~/lab-evidence/9b/b5-khong-antoan-kubelet.txt
```

```bash
LY="$(kubectl -n lab-9b get pod sysctl-khong-antoan -o jsonpath='{.status.reason}')"
PHA="$(kubectl -n lab-9b get pod sysctl-khong-antoan -o jsonpath='{.status.phase}')"
ND="$(kubectl -n lab-9b get pod sysctl-khong-antoan -o jsonpath='{.spec.nodeName}')"
echo "reason=$LY phase=$PHA node=$ND"

test -n "$ND" \
  && echo 'PASS: Pod VAN duoc lap lich — dung nhu bai 258 mo ta'
test "$LY" = 'SysctlForbidden' \
  && echo 'PASS: kubelet tu choi khoi chay voi ly do SysctlForbidden'
test "$PHA" = 'Failed' \
  && echo 'PASS: Pod ket thuc o pha Failed, khong bao gio Running'
test "$ND" = "$W2" \
  && echo 'PASS: thi nghiem nay chay dung tren lab-k8s-worker2'
```

Dọn phần B5:

```bash
kubectl -n lab-9b delete pod sysctl-khong-antoan --wait=true --ignore-not-found
kubectl -n lab-9b-baseline delete pod sysctl-antoan ro-root --wait=true --ignore-not-found
test "$(kubectl -n lab-9b-baseline get pods --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: lab-9b-baseline lai chi con Pod tran'
```

**Ý nghĩa:** đây là bài học vận hành đắt nhất của mục này. Cùng một manifest sai, hai namespace,
hai kiểu triệu chứng hoàn toàn khác:

| Namespace | Chặn ở đâu | Bạn thấy gì | Cách điều tra |
| --- | --- | --- | --- |
| có nhãn `enforce` | API server, chặng admission | một câu lỗi ở client, **không có object nào** | đọc câu thông báo trả về |
| không nhãn | kubelet trên node | Pod tồn tại, `Failed`, `SysctlForbidden` | `kubectl get pod -o wide`, `describe`, log kubelet |

Bài [258](../258-sysctl-cluster-vi.md) nói rõ điều thứ hai: Pod dùng sysctl không an toàn đang bị
tắt **vẫn được lập lịch, nhưng không khởi chạy được**. Bài cũng đưa cách xử lý đúng nếu bạn thật sự
cần chúng: bật riêng trên từng node bằng cờ kubelet, rồi dùng taint và toleration để chỉ những Pod
cần mới lên đúng node đó — lab này không đổi cờ kubelet, xem bảng lý do ở
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

**PASS:** tám dòng `PASS:` của ba khối trên xuất hiện.

---

## B6. PodSecurityPolicy đã biến mất — chứng minh bằng chính API

**Mục đích:** bài [117](../117-pod-security-policy-vi.md) là **tài liệu lịch sử** và chỉ khẳng
định đúng một điều kiểm chứng được: PodSecurityPolicy bị đánh dấu lỗi thời ở v1.21 và **bị gỡ bỏ**
ở v1.25. Bài [286](../286-migrate-from-psp-vi.md) là lộ trình di trú. B6 kiểm chứng khẳng định đó,
rồi ánh xạ lộ trình sang một cluster đã vượt mốc gỡ bỏ rất xa.

### B6.1. Nhóm API `policy/v1beta1` không còn tồn tại

```bash
kubectl api-versions | grep '^policy/' | tee ~/lab-evidence/9b/b6-policy-apiversions.txt
kubectl api-resources --api-group=policy -o name \
  | tee ~/lab-evidence/9b/b6-policy-resources.txt

kubectl get --raw /apis/policy/v1beta1 > ~/lab-evidence/9b/b6-raw-v1beta1.txt 2>&1
RC_RAW=$?
echo "exit code khi goi /apis/policy/v1beta1 = $RC_RAW"
cat ~/lab-evidence/9b/b6-raw-v1beta1.txt
```

```bash
test "$(grep -c '^policy/v1beta1$' ~/lab-evidence/9b/b6-policy-apiversions.txt)" -eq 0 \
  && echo 'PASS: khong con phien ban policy/v1beta1 trong danh sach API cua cluster'
test "$(grep -c 'podsecuritypolic' ~/lab-evidence/9b/b6-policy-resources.txt)" -eq 0 \
  && echo 'PASS: nhom API policy khong con tai nguyen podsecuritypolicies'
test "$RC_RAW" -ne 0 \
  && echo 'PASS: goi thang duong dan /apis/policy/v1beta1 that bai — duong dan do khong ton tai'
grep -q '^policy/v1$' ~/lab-evidence/9b/b6-policy-apiversions.txt \
  && echo 'PASS: nhom policy VAN con o phien ban v1 — chi rieng PodSecurityPolicy bi go'
```

**Ý nghĩa:** dòng gate cuối là chỗ dễ nhầm. Nhóm API `policy` **không** biến mất — nó vẫn phục vụ
`poddisruptionbudgets`. Thứ bị gỡ là **một kind** cùng với **một phiên bản** của nhóm đó. Phân biệt
"deprecated" với "removed" cũng vậy: từ v1.21 đến trước v1.25, API vẫn còn và vẫn chạy, chỉ kèm
cảnh báo; sau mốc gỡ bỏ thì manifest cũ không apply được nữa và **mọi chính sách dựa vào nó im
lặng biến mất** khi cluster được nâng cấp qua mốc.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B6.2. Manifest PSP của bài không apply được

Đây là đúng file `privileged-psp.yaml` mà bài [286](../286-migrate-from-psp-vi.md) dùng ở bước
*Bỏ qua PodSecurityPolicy*:

```bash
cat > ~/lab-work/9b/b6-privileged-psp.yaml <<'YAMLEOF'
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: privileged
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
spec:
  privileged: true
  allowPrivilegeEscalation: true
  allowedCapabilities:
    - '*'
  volumes:
    - '*'
  hostNetwork: true
  hostPorts:
    - min: 0
      max: 65535
  hostIPC: true
  hostPID: true
  runAsUser:
    rule: 'RunAsAny'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
YAMLEOF

kubectl apply -f ~/lab-work/9b/b6-privileged-psp.yaml \
  > ~/lab-evidence/9b/b6-apply-psp.txt 2>&1
cat ~/lab-evidence/9b/b6-apply-psp.txt
```

```bash
grep -q 'no matches for kind' ~/lab-evidence/9b/b6-apply-psp.txt \
  && echo 'PASS: API server khong nhan ra kind PodSecurityPolicy'
grep -q 'policy/v1beta1' ~/lab-evidence/9b/b6-apply-psp.txt \
  && echo 'PASS: thong bao goi dung phien ban khong con ton tai'
```

**Ý nghĩa:** câu lỗi này khác hẳn `Forbidden`. Nó không phải chuyện quyền, mà là chuyện **API
không có kind đó**. Khi tiếp quản một cluster cũ và định nâng cấp, đây chính là thứ sẽ xảy ra với
mọi manifest PSP trong kho GitOps của bạn ngay sau lần nâng cấp qua mốc gỡ bỏ.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B6.3. Không Pod nào còn mang dấu vết PSP

Bước 2.c và 3.a của bài [286](../286-migrate-from-psp-vi.md) tìm Pod theo annotation
`kubernetes.io/psp`. Chạy đúng lệnh đó trên cluster này:

```bash
kubectl get pods --all-namespaces \
  -o jsonpath="{range .items[*]}{.metadata.namespace}{' '}{.metadata.name}{' '}{.metadata.annotations.kubernetes\.io/psp}{'\n'}{end}" \
  | tee ~/lab-evidence/9b/b6-annotation-psp.txt | head -20

CO_PSP="$(awk 'NF>=3 {print}' ~/lab-evidence/9b/b6-annotation-psp.txt | wc -l)"
TONG_POD="$(kubectl get pods -A --no-headers | wc -l)"
echo "pod mang annotation kubernetes.io/psp = $CO_PSP / tong so pod = $TONG_POD"
```

```bash
test "$TONG_POD" -gt 0 \
  && echo 'PASS: cluster co Pod dang chay de doi chieu'
test "$CO_PSP" -eq 0 \
  && echo 'PASS: khong Pod nao mang annotation kubernetes.io/psp — khong co PSP nao tung tac dong'
```

**Ý nghĩa:** trên cluster thật đang di trú, con số này là thước đo tiến độ: sau khi triển khai
chính sách mới và **tạo lại** Pod, số Pod còn mang annotation của PSP gốc phải giảm về 0. Trên
cluster của bạn nó bằng 0 vì cơ chế đó chưa từng tồn tại.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B6.4. Năm bước di trú của bài, đối chiếu với cluster này

Ghi lại kết luận để dùng khi thật sự phải tiếp quản một cluster cũ:

```bash
{
  echo "=== $(date -Is) — anh xa lo trinh 286 sang cluster da qua moc go bo ==="
  echo '0. Pod Security Admission co phu hop khong?  -> can quyet dinh; PSA khong mutating,'
  echo '   chi co 3 muc chuan, va khong phan quyen duoi cap namespace'
  echo '1. Ra soat quyen tren namespace              -> VAN CAN LAM, kiem chung o B8.2'
  echo '2. Don gian hoa va chuan hoa cac PSP         -> khong ap dung, khong co PSP nao'
  echo '3.a Xac dinh muc Pod Security phu hop        -> VAN CAN LAM, dua tren Pod hien co'
  echo '3.b Kiem chung muc                           -> VAN CAN LAM, da lam o B3.5 va B3.3'
  echo '3.c Thuc thi muc                             -> VAN CAN LAM, da lam o B2'
  echo '3.d Bo qua PodSecurityPolicy                 -> khong ap dung, B6.2 da chung minh'
  echo '4. Ra soat quy trinh tao namespace moi       -> VAN CAN LAM, xem B1.3'
  echo '5. Tat admission controller PodSecurityPolicy-> khong ap dung, xac nhan lai o B7.1'
} | tee ~/lab-evidence/9b/b6-anh-xa-di-tru.txt

test "$(grep -c 'VAN CAN LAM' ~/lab-evidence/9b/b6-anh-xa-di-tru.txt)" -eq 5 \
  && echo 'PASS: ghi lai duoc nam buoc con nghia va bon buoc khong con nghia'
```

**Ý nghĩa:** phần còn giá trị của bài [286](../286-migrate-from-psp-vi.md) không phải là PSP, mà là
**phương pháp**: rà quyền namespace trước, chọn mức bằng cách nhìn Pod hiện có, kiểm chứng bằng
`audit`/`warn` và dry-run trước khi siết `enforce`, rồi mới cập nhật quy trình tạo namespace mới.
Phương pháp đó áp dụng nguyên vẹn cho một cluster chưa từng có PSP.

Cũng nhớ ba giới hạn bài nêu ở bước 0, vì chúng giải thích vì sao Pod Security Admission **không**
thay được mọi thứ: nó **không mutating**, nên không đặt giá trị mặc định cho bạn — đó là lý do B2.4
phải viết đủ bốn trường bằng tay; nó chỉ có **ba mức chuẩn**, không cho ràng buộc chi tiết hơn; và
nó phân quyền ở **cấp namespace**, không cho hai ServiceAccount trong cùng namespace chịu hai chính
sách khác nhau.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B7. Chuỗi admission thật, webhook, và API Priority and Fairness

**Mục đích:** ba thứ ở tầng API server mà một quản trị viên phải đọc được mà không cần cài thêm gì:
danh sách admission plugin đang chạy, tồn kho admission webhook, và cấu hình APF.

### B7.1. Những admission plugin thật sự đang chạy

Bài [286](../286-migrate-from-psp-vi.md) kiểm chuỗi admission bằng cách đọc log lúc apiserver khởi
động. Cách bền hơn là đọc số liệu của chính API server — mỗi plugin từng chạy đều có một dòng
counter mang tên nó:

```bash
kubectl get --raw /metrics \
  | grep -o 'apiserver_admission_controller_admission_duration_seconds_count{name="[^"]*"' \
  | sed 's/.*name="//; s/"$//' \
  | sort -u | tee ~/lab-evidence/9b/b7-admission-plugin.txt

echo "so plugin doc duoc = $(grep -c . ~/lab-evidence/9b/b7-admission-plugin.txt)"
```

```bash
test "$(grep -c . ~/lab-evidence/9b/b7-admission-plugin.txt)" -gt 5 \
  && echo 'PASS: doc duoc danh sach admission plugin dang hoat dong'
grep -qx 'PodSecurity' ~/lab-evidence/9b/b7-admission-plugin.txt \
  && echo 'PASS: plugin PodSecurity co mat trong chuoi admission'
test "$(grep -cx 'PodSecurityPolicy' ~/lab-evidence/9b/b7-admission-plugin.txt)" -eq 0 \
  && echo 'PASS: PodSecurityPolicy KHONG co trong chuoi — dung buoc 5 cua bai 286'
grep -qx 'ResourceQuota' ~/lab-evidence/9b/b7-admission-plugin.txt \
  && grep -qx 'LimitRanger' ~/lab-evidence/9b/b7-admission-plugin.txt \
  && echo 'PASS: ResourceQuota va LimitRanger cung dang bat — B8.3 dua vao ca hai'
```

Và kiểm chứng rằng bộ mặc định của bài
[282](../282-enforce-standards-admission-controller-vi.md) đang có hiệu lực nguyên vẹn, tức là
**không** có file cấu hình nào ghi đè nó:

```bash
{
  echo "--- co bao nhieu dong --admission-control-config-file trong manifest apiserver ---"
  sudo grep -c 'admission-control-config-file' /etc/kubernetes/manifests/kube-apiserver.yaml
  echo "--- cac co lien quan admission dang co ---"
  sudo grep -nE 'admission|feature-gates' /etc/kubernetes/manifests/kube-apiserver.yaml \
    || echo '(khong co dong nao)'
} | tee ~/lab-evidence/9b/b7-admission-config.txt

test "$(sudo grep -c 'admission-control-config-file' \
  /etc/kubernetes/manifests/kube-apiserver.yaml)" -eq 0 \
  && echo 'PASS: khong co file cau hinh admission — bo mac dinh privileged/latest dang co hieu luc'
```

**Ý nghĩa:** hai điều vừa chứng minh nối B1.3 với B6. Namespace không nhãn mặc định là
`privileged` **vì** không ai ghi đè bộ mặc định đó; và PSP không còn trong chuỗi **vì** plugin đã
bị gỡ khỏi binary, không phải vì ai đó tắt nó. Bài [129](../129-security-checklist-vi.md) có một ô
đúng cho việc này — *một tập hợp admission controller phù hợp được bật* — và file bạn vừa ghi là
câu trả lời cho ô đó.

**PASS:** năm dòng `PASS:` của hai khối trên xuất hiện.

### B7.2. Tồn kho admission webhook và `failurePolicy`

```bash
kubectl api-resources --api-group=admissionregistration.k8s.io -o name \
  | tee ~/lab-evidence/9b/b7-admissionregistration-kind.txt

MWC="$(kubectl get mutatingwebhookconfigurations --no-headers 2>/dev/null | wc -l)"
VWC="$(kubectl get validatingwebhookconfigurations --no-headers 2>/dev/null | wc -l)"
{
  echo "=== $(date -Is) — ton kho admission webhook ==="
  echo "mutatingwebhookconfigurations=$MWC"
  echo "validatingwebhookconfigurations=$VWC"
  echo '--- ten webhook, failurePolicy, timeoutSeconds ---'
  kubectl get mutatingwebhookconfigurations,validatingwebhookconfigurations \
    -o jsonpath='{range .items[*]}{.kind}{" "}{.metadata.name}{"\n"}{range .webhooks[*]}{"    hook="}{.name}{" failurePolicy="}{.failurePolicy}{" timeoutSeconds="}{.timeoutSeconds}{"\n"}{end}{end}' \
    2>/dev/null || true
  echo '(het)'
} | tee ~/lab-evidence/9b/b7-webhook.txt
```

```bash
test "$(grep -c 'mutatingwebhookconfigurations' \
  ~/lab-evidence/9b/b7-admissionregistration-kind.txt)" -ge 1 \
  && test "$(grep -c 'validatingwebhookconfigurations' \
       ~/lab-evidence/9b/b7-admissionregistration-kind.txt)" -ge 1 \
  && echo 'PASS: hai kind cau hinh webhook co that trong API cua cluster nay'
grep -q "mutatingwebhookconfigurations=$MWC" ~/lab-evidence/9b/b7-webhook.txt \
  && grep -q "validatingwebhookconfigurations=$VWC" ~/lab-evidence/9b/b7-webhook.txt \
  && echo "PASS: ghi duoc ton kho webhook — mutating=$MWC validating=$VWC"
grep -q '(het)' ~/lab-evidence/9b/b7-webhook.txt \
  && echo 'PASS: file ton kho hoan chinh, khong bi cat giua chung'
```

**Ý nghĩa và phần chỉ đọc được, không làm được:** bài
[173](../173-admission-webhooks-vi.md) nói `failurePolicy` mặc định là `Fail`, nghĩa là **API
server từ chối request nếu webhook gặp lỗi** — timeout hay lỗi logic đều tính. Hệ quả: trong thời
gian webhook chết, mọi request khớp phạm vi của nó đều hỏng, kể cả request hợp lệ. Vì thế bài
khuyên mutating webhook nên **"fail open"** bằng `failurePolicy: Ignore`, và để một validating
controller kiểm trạng thái cuối cùng — chính sách vẫn được thực thi ở giai đoạn validating, còn
thời gian webhook chết không chặn việc triển khai tài nguyên hợp lệ.

Ba khuyến nghị nữa của bài mà bạn đọc được ngay từ tồn kho vừa ghi, nếu cluster có webhook:
`timeoutSeconds` phải **nhỏ**, vì webhook cộng thẳng độ trễ vào mọi request API; phạm vi phải hẹp —
tránh khớp `kube-system`, **không** biến đổi Lease trong `kube-node-lease` vì nó làm hỏng nâng cấp
node, và **không** đụng vào TokenReview hay SubjectAccessReview vốn luôn là request chỉ đọc; và
quyền `create`/`update`/`patch`/`delete` trên `MutatingWebhookConfigurations` phải chỉ dành cho
thực thể đáng tin cậy, vì ai sửa được nó thì sửa được mọi object sắp được ghi.

**Lab dừng ở đây, không cài webhook** — lý do nằm ở bảng tại
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành), và nó chính là cảnh báo mở đầu của bài 173:
webhook thiết kế tồi làm gián đoạn workload vì nó nắm quá nhiều quyền kiểm soát đối với object
trong cluster.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B7.3. FlowSchema và PriorityLevelConfiguration thật đang có

```bash
kubectl get flowschema | tee ~/lab-evidence/9b/b7-flowschema.txt
kubectl get prioritylevelconfiguration | tee ~/lab-evidence/9b/b7-prioritylevel.txt
```

Bốn đối tượng **bắt buộc** mà bài [166](../166-flow-control-vi.md) mô tả:

```bash
BB=0
for x in 'flowschema exempt' 'flowschema catch-all' \
         'prioritylevelconfiguration exempt' 'prioritylevelconfiguration catch-all'; do
  if kubectl get $x -o name >/dev/null 2>&1; then
    echo "co doi tuong bat buoc: $x"; BB=$(( BB + 1 ))
  else
    echo "THIEU doi tuong bat buoc: $x"
  fi
done
test "$BB" -eq 4 && echo 'PASS: du bon doi tuong cau hinh bat buoc cua APF'

G_EXEMPT="$(kubectl get flowschema exempt \
  -o jsonpath='{.spec.rules[0].subjects[0].group.name}')"
T_EXEMPT="$(kubectl get prioritylevelconfiguration exempt -o jsonpath='{.spec.type}')"
T_CATCH="$(kubectl get prioritylevelconfiguration catch-all -o jsonpath='{.spec.type}')"
echo "flowschema exempt khop group=$G_EXEMPT | plc exempt type=$T_EXEMPT | catch-all type=$T_CATCH"

test "$G_EXEMPT" = 'system:masters' \
  && echo 'PASS: FlowSchema exempt phan loai moi request tu nhom system:masters'
test "$T_EXEMPT" = 'Exempt' && test "$T_CATCH" = 'Limited' \
  && echo 'PASS: exempt khong chiu gioi han, catch-all thi co — dung hai vai tro bai mo ta'
```

Sáu mức ưu tiên **được đề xuất**:

```bash
DX=0
for p in node-high system leader-election workload-high workload-low global-default; do
  if kubectl get prioritylevelconfiguration "$p" -o name >/dev/null 2>&1; then
    echo "co muc de xuat: $p"; DX=$(( DX + 1 ))
  else
    echo "THIEU muc de xuat: $p"
  fi
done
echo "so muc uu tien duoc de xuat co mat = $DX/6"
test "$DX" -eq 6 && echo 'PASS: du sau muc uu tien cua cau hinh duoc de xuat'

kubectl get flowschema \
  -o custom-columns='FS:.metadata.name,PRECEDENCE:.spec.matchingPrecedence,PLC:.spec.priorityLevelConfiguration.name,DISTINGUISHER:.spec.distinguisherMethod.type' \
  --sort-by=.spec.matchingPrecedence \
  | tee ~/lab-evidence/9b/b7-flowschema-thu-tu.txt
test "$(grep -c . ~/lab-evidence/9b/b7-flowschema-thu-tu.txt)" -gt 6 \
  && echo 'PASS: ghi duoc bang FlowSchema xep theo matchingPrecedence tang dan'
```

**Ý nghĩa:** bảng cuối cùng là thứ đáng đọc kỹ. Mọi request đến được đối chiếu với các FlowSchema
theo `matchingPrecedence` **từ nhỏ tới lớn**, và **FlowSchema khớp đầu tiên thắng** — nên đọc bảng
từ trên xuống chính là đọc thứ tự phân loại thật. Cột `DISTINGUISHER` cho biết trong cùng một mức
ưu tiên, request được tách thành flow theo tiêu chí nào: `ByUser` thì một người dùng không làm đói
người khác, `ByNamespace` thì một namespace không làm đói namespace khác, để trống thì mọi request
khớp schema đó nằm chung một flow duy nhất.

Với một cluster đa người thuê — chủ đề của B8 — đây là lớp bảo vệ **control plane** tương đương với
ResourceQuota ở lớp tài nguyên node: nó ngăn một controller hỏng của tenant này làm ngập API server
và làm đói bầu chọn leader hay chính kubelet.

**PASS:** năm dòng `PASS:` của bước này xuất hiện, và không dòng `THIEU` nào.

### B7.4. Request của chính bạn rơi vào FlowSchema nào

API server trả về hai header cho biết nó đã phân loại request vào đâu:

```bash
kubectl get namespace -v=8 > ~/lab-evidence/9b/b7-pf-header.txt 2>&1
grep -i 'X-Kubernetes-Pf' ~/lab-evidence/9b/b7-pf-header.txt

FS_UID="$(grep -io 'flowschema-uid: *[0-9a-f-]*' ~/lab-evidence/9b/b7-pf-header.txt \
  | head -1 | sed 's/.*: *//')"
PL_UID="$(grep -io 'prioritylevel-uid: *[0-9a-f-]*' ~/lab-evidence/9b/b7-pf-header.txt \
  | head -1 | sed 's/.*: *//')"

FS_NAME="$(kubectl get flowschema \
  -o jsonpath='{range .items[*]}{.metadata.uid}{" "}{.metadata.name}{"\n"}{end}' \
  | awk -v u="$FS_UID" '$1==u {print $2}')"
PL_NAME="$(kubectl get prioritylevelconfiguration \
  -o jsonpath='{range .items[*]}{.metadata.uid}{" "}{.metadata.name}{"\n"}{end}' \
  | awk -v u="$PL_UID" '$1==u {print $2}')"

{
  echo "=== $(date -Is) — request cua kubeconfig quan tri di qua dau ==="
  echo "FlowSchema               = $FS_NAME  (uid $FS_UID)"
  echo "PriorityLevelConfiguration = $PL_NAME  (uid $PL_UID)"
} | tee ~/lab-evidence/9b/b7-pf-ket-qua.txt
```

```bash
test -n "$FS_UID" && test -n "$PL_UID" \
  && echo 'PASS: API server tra ve ca hai header phan loai — APF dang bat'
test -n "$FS_NAME" \
  && kubectl get flowschema "$FS_NAME" -o name >/dev/null 2>&1 \
  && echo "PASS: giai duoc uid thanh ten FlowSchema co that: $FS_NAME"
test -n "$PL_NAME" \
  && kubectl get prioritylevelconfiguration "$PL_NAME" -o name >/dev/null 2>&1 \
  && echo "PASS: giai duoc uid thanh ten PriorityLevelConfiguration co that: $PL_NAME"
```

**Ý nghĩa:** hai header này biến APF từ lý thuyết thành thứ quan sát được cho **từng** request. Đối
chiếu tên FlowSchema vừa ra với nhóm mà kubeconfig quản trị của bạn thuộc về — Lab 9a đã đọc
`O` trong client certificate ở B2.1 — rồi tự trả lời vì sao request rơi đúng vào schema đó chứ
không phải schema khác. Nếu nó là `exempt`, request của bạn **không chịu bất kỳ giới hạn nào** và
luôn được điều phối ngay; nếu là một mức `Limited`, nó chia sẻ ngân sách ghế với những request cùng
mức.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B8. Một namespace tenant hoàn chỉnh: bốn lớp cùng hoạt động

**Mục đích:** bài [122](../122-multi-tenancy-vi.md) không giới thiệu cơ chế mới nào — nó **ghép**
những cơ chế bạn đã học thành một mô hình cô lập. B8 dựng đúng mô hình đó cho namespace
`lab-9b-tenant` rồi kiểm **từng lớp** bằng một phép thử riêng.

Bốn lớp, theo đúng cách bài chia:

| Lớp | Mặt phẳng | Cơ chế | Đã học ở |
| --- | --- | --- | --- |
| 1 | control plane | Namespace + Pod Security Standards | B0.3 và B2 của lab này |
| 2 | control plane | RBAC — Role và RoleBinding phạm vi namespace | [Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md) |
| 3 | control plane | ResourceQuota và LimitRange | [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) |
| 4 | data plane | NetworkPolicy | [Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) |

### B8.1. Lớp 1 — namespace và chính sách Pod

Namespace `lab-9b-tenant` đã mang ba chế độ ở mức `restricted` từ B0.3. Kiểm chứng lớp này bằng
đúng Pod đặc quyền đã dùng ở B3.1:

```bash
kubectl get namespace lab-9b-tenant --show-labels | tee ~/lab-evidence/9b/b8-lop1-nhan.txt

psa_try lab-9b-tenant ~/lab-work/9b/b3-canh-bao.yaml ~/lab-evidence/9b/b8-lop1-tu-choi.txt
cat ~/lab-evidence/9b/b8-lop1-tu-choi.txt
```

```bash
grep -q 'violates PodSecurity' ~/lab-evidence/9b/b8-lop1-tu-choi.txt \
  && echo 'PASS: lop 1 chan — Pod dac quyen khong vao duoc namespace tenant'
grep -q "restricted:$PSA_VER" ~/lab-evidence/9b/b8-lop1-tu-choi.txt \
  && echo 'PASS: chan o dung muc restricted da ghim'
test -z "$(kubectl -n lab-9b-tenant get pod canh-bao --ignore-not-found -o name)" \
  && echo 'PASS: khong object nao duoc tao'
```

**Ý nghĩa:** đây là điểm bài [122](../122-multi-tenancy-vi.md) gọi là "mỗi workload một namespace,
mỗi namespace một chính sách bảo mật phù hợp". Namespace cho hai thứ: **không gian tên riêng** để
tenant đặt tên tài nguyên không phải quan tâm tenant khác, và **phạm vi** để gắn chính sách — vì
Role và NetworkPolicy đều là tài nguyên thuộc namespace.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B8.2. Lớp 2 — RBAC giới hạn tenant trong đúng namespace của nó

```bash
cat > ~/lab-work/9b/b8-rbac.yaml <<'YAMLEOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tenant-sa
  labels:
    lab: "9b"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: tenant-doc
  labels:
    lab: "9b"
rules:
  - apiGroups: [""]
    resources: ["pods", "configmaps"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-doc
  labels:
    lab: "9b"
subjects:
  - kind: ServiceAccount
    name: tenant-sa
    namespace: lab-9b-tenant
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: tenant-doc
YAMLEOF

kubectl apply -n lab-9b-tenant -f ~/lab-work/9b/b8-rbac.yaml
SA='system:serviceaccount:lab-9b-tenant:tenant-sa'

{
  echo "=== $(date -Is) — lop 2: pham vi quyen cua $SA ==="
  for q in 'list pods -n lab-9b-tenant' 'list configmaps -n lab-9b-tenant' \
           'list pods -n lab-9b' 'list secrets -n lab-9b-tenant' \
           'delete pods -n lab-9b-tenant' 'patch namespaces'; do
    printf '%-34s -> %s\n' "$q" "$(kubectl auth can-i $q --as="$SA" 2>/dev/null)"
  done
} | tee ~/lab-evidence/9b/b8-lop2-quyen.txt
```

```bash
test "$(kubectl auth can-i list pods -n lab-9b-tenant --as="$SA")" = 'yes' \
  && echo 'PASS: tenant doc duoc Pod trong namespace cua chinh no'
test "$(kubectl auth can-i list pods -n lab-9b --as="$SA")" = 'no' \
  && echo 'PASS: lop 2 chan — cung mot verb, namespace khac thi khong'
test "$(kubectl auth can-i list secrets -n lab-9b-tenant --as="$SA")" = 'no' \
  && echo 'PASS: Role khong chua secrets, nen list Secret bi tu choi'
test "$(kubectl auth can-i delete pods -n lab-9b-tenant --as="$SA")" = 'no' \
  && echo 'PASS: chi ba verb doc duoc cap, khong co verb ghi'
test "$(kubectl auth can-i patch namespaces --as="$SA")" = 'no' \
  && echo 'PASS: tenant KHONG patch duoc Namespace — nen no khong tu ha muc Pod Security'
```

**Ý nghĩa:** gate cuối là bước 1 của bài [286](../286-migrate-from-psp-vi.md) — *Rà soát quyền
trên namespace* — biến thành phép thử. Pod Security Admission được điều khiển bằng **label của
Namespace**, nên **bất kỳ ai `update`, `patch` hay `create` được Namespace đều đổi được mức Pod
Security của nó**, và đó là một đường vòng qua toàn bộ lớp 1. Nếu tenant có quyền đó, ba lớp còn
lại không cứu được bạn. Bài [130](../130-application-security-checklist-vi.md) nhắc cùng ý ở ô về
verb **patch** trên Namespace.

Lưu ý phạm vi: lab **cố ý** chỉ tạo Role và RoleBinding — hai object **thuộc namespace**, nên
`kubectl delete namespace` ở B10.1 dọn sạch chúng. Đó cũng là khuyến nghị của bài
[122](../122-multi-tenancy-vi.md): tài nguyên phạm vi cluster ít hữu ích cho cluster đa người thuê.

**PASS:** năm dòng `PASS:` của bước này xuất hiện.

### B8.3. Lớp 3 — quota và giới hạn tài nguyên

```bash
cat > ~/lab-work/9b/b8-quota.yaml <<'YAMLEOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-mac-dinh
  labels:
    lab: "9b"
spec:
  limits:
    - type: Container
      default:
        cpu: 100m
        memory: 64Mi
      defaultRequest:
        cpu: 50m
        memory: 32Mi
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  labels:
    lab: "9b"
spec:
  hard:
    pods: "2"
    requests.cpu: 500m
    requests.memory: 512Mi
    limits.cpu: "1"
    limits.memory: 1Gi
YAMLEOF

kubectl apply -n lab-9b-tenant -f ~/lab-work/9b/b8-quota.yaml
kubectl -n lab-9b-tenant describe resourcequota tenant-quota \
  | tee ~/lab-evidence/9b/b8-lop3-quota.txt
```

Hai workload của tenant, viết theo đúng khuôn Restricted của B2.4:

```bash
cat > ~/lab-work/9b/b8-tenant-pod.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: tenant-web
  labels:
    lab: "9b"
    app: web
spec:
  serviceAccountName: tenant-sa
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: c
      image: busybox:1.37
      command:
        - sh
        - -c
        - "mkdir -p /tmp/www && echo tenant-ok > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
---
apiVersion: v1
kind: Pod
metadata:
  name: tenant-client
  labels:
    lab: "9b"
    app: client
spec:
  serviceAccountName: tenant-sa
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
YAMLEOF

psa_try lab-9b-tenant ~/lab-work/9b/b8-tenant-pod.yaml ~/lab-evidence/9b/b8-tenant-pod.txt
cat ~/lab-evidence/9b/b8-tenant-pod.txt
kubectl -n lab-9b-tenant wait --for=condition=Ready pod/tenant-web pod/tenant-client \
  --timeout=180s
```

```bash
grep -qi 'violate' ~/lab-evidence/9b/b8-tenant-pod.txt \
  || echo 'PASS: hai Pod cua tenant dat muc restricted — khong canh bao, khong tu choi'

R_CPU="$(kubectl -n lab-9b-tenant get pod tenant-web \
  -o jsonpath='{.spec.containers[0].resources.requests.cpu}')"
L_MEM="$(kubectl -n lab-9b-tenant get pod tenant-web \
  -o jsonpath='{.spec.containers[0].resources.limits.memory}')"
echo "manifest khong khai resources, nhung Pod co requests.cpu=$R_CPU limits.memory=$L_MEM"
test -n "$R_CPU" && test -n "$L_MEM" \
  && echo 'PASS: LimitRange da chen gia tri mac dinh — dieu kien de ResourceQuota chap nhan Pod'

DUNG="$(kubectl -n lab-9b-tenant get resourcequota tenant-quota -o jsonpath='{.status.used.pods}')"
TRAN="$(kubectl -n lab-9b-tenant get resourcequota tenant-quota -o jsonpath='{.status.hard.pods}')"
echo "quota pods: used=$DUNG hard=$TRAN"
test "$DUNG" = "$TRAN" && echo 'PASS: tenant da dung het tran so luong Pod'
```

Pod thứ ba — lớp 3 chặn:

```bash
cat > ~/lab-work/9b/b8-tenant-thua.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: tenant-thua
  labels:
    lab: "9b"
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: c
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: ["ALL"]
YAMLEOF

psa_try lab-9b-tenant ~/lab-work/9b/b8-tenant-thua.yaml ~/lab-evidence/9b/b8-lop3-tu-choi.txt
cat ~/lab-evidence/9b/b8-lop3-tu-choi.txt
```

```bash
grep -q 'exceeded quota' ~/lab-evidence/9b/b8-lop3-tu-choi.txt \
  && echo 'PASS: lop 3 chan — Pod hop le ve bao mat nhung vuot tran so luong'
grep -q 'violates PodSecurity' ~/lab-evidence/9b/b8-lop3-tu-choi.txt \
  && echo 'FAIL: Pod nay le ra phai dat restricted — kiem lai manifest'
test "$(kubectl -n lab-9b-tenant get pods --no-headers | wc -l)" -eq 2 \
  && echo 'PASS: namespace tenant van dung hai Pod'
```

**Ý nghĩa:** hai câu từ chối của B8.1 và B8.3 nói hai chuyện hoàn toàn khác nhau — `violates
PodSecurity` là **bảo mật**, `exceeded quota` là **công bằng tài nguyên**. Bài
[122](../122-multi-tenancy-vi.md) nhấn mạnh quota là thứ chống "hàng xóm ồn ào", và nó cũng nhắc
một điều quan trọng: **khi có quota, Kubernetes yêu cầu mọi container phải khai request và
limit**. LimitRange là thứ giữ cho yêu cầu đó không rơi vào đầu tenant — bạn vừa chứng minh bằng hai
giá trị được chèn vào Pod mà manifest không hề khai.

**PASS:** bốn dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B8.4. Lớp 4 — NetworkPolicy, đo trước rồi mới siết

Trước hết dựng một Pod **ngoài** tenant, trong `lab-9b`, và đo đường cơ sở. Không có đường cơ sở
thì mọi timeout ở bước sau đều mơ hồ:

```bash
cat > ~/lab-work/9b/b8-ngoai.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: ngoai
  labels:
    lab: "9b"
spec:
  containers:
    - name: c
      image: busybox:1.37
      command:
        - sh
        - -c
        - "mkdir -p /tmp/www && echo ngoai-ok > /tmp/www/index.html && httpd -f -p 8080 -h /tmp/www"
YAMLEOF

kubectl apply -n lab-9b -f ~/lab-work/9b/b8-ngoai.yaml
kubectl -n lab-9b wait --for=condition=Ready pod/ngoai --timeout=180s

WEB_IP="$(kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath='{.status.podIP}')"
NGOAI_IP="$(kubectl -n lab-9b get pod ngoai -o jsonpath='{.status.podIP}')"
echo "tenant-web=$WEB_IP  ngoai=$NGOAI_IP"

{
  echo "=== $(date -Is) — duong co so, TRUOC khi co NetworkPolicy ==="
  echo -n 'ngoai -> tenant-web        : '
  kubectl -n lab-9b exec ngoai -- wget -q -T 3 -O- "http://$WEB_IP:8080" 2>&1 || echo 'KHONG THONG'
  echo -n 'tenant-client -> tenant-web: '
  kubectl -n lab-9b-tenant exec tenant-client -- wget -q -T 3 -O- "http://$WEB_IP:8080" 2>&1 \
    || echo 'KHONG THONG'
  echo -n 'tenant-client -> ngoai     : '
  kubectl -n lab-9b-tenant exec tenant-client -- wget -q -T 3 -O- "http://$NGOAI_IP:8080" 2>&1 \
    || echo 'KHONG THONG'
} | tee ~/lab-evidence/9b/b8-lop4-truoc.txt
```

```bash
test "$(grep -c 'tenant-ok' ~/lab-evidence/9b/b8-lop4-truoc.txt)" -eq 2 \
  && echo 'PASS: duong co so — ca Pod ngoai lan Pod trong tenant deu goi duoc tenant-web'
grep -q 'ngoai-ok' ~/lab-evidence/9b/b8-lop4-truoc.txt \
  && echo 'PASS: duong co so — tenant-client goi duoc ra ngoai namespace'
```

Áp policy cô lập cho tenant:

```bash
cat > ~/lab-work/9b/b8-networkpolicy.yaml <<'YAMLEOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: tenant-co-lap
  labels:
    lab: "9b"
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}
  egress:
    - to:
        - podSelector: {}
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
YAMLEOF

kubectl apply -n lab-9b-tenant -f ~/lab-work/9b/b8-networkpolicy.yaml
kubectl -n lab-9b-tenant describe networkpolicy tenant-co-lap \
  | tee ~/lab-evidence/9b/b8-lop4-policy.txt
```

Đo lại đúng ba đường đó, cộng DNS:

```bash
{
  echo "=== $(date -Is) — SAU khi ap NetworkPolicy tenant-co-lap ==="
  echo -n 'ngoai -> tenant-web        : '
  kubectl -n lab-9b exec ngoai -- wget -q -T 3 -O- "http://$WEB_IP:8080" 2>&1 || echo 'KHONG THONG'
  echo -n 'tenant-client -> tenant-web: '
  kubectl -n lab-9b-tenant exec tenant-client -- wget -q -T 3 -O- "http://$WEB_IP:8080" 2>&1 \
    || echo 'KHONG THONG'
  echo -n 'tenant-client -> ngoai     : '
  kubectl -n lab-9b-tenant exec tenant-client -- wget -q -T 3 -O- "http://$NGOAI_IP:8080" 2>&1 \
    || echo 'KHONG THONG'
  echo -n 'tenant-client -> DNS       : '
  kubectl -n lab-9b-tenant exec tenant-client -- nslookup kubernetes.default >/dev/null 2>&1 \
    && echo 'THONG' || echo 'KHONG THONG'
} | tee ~/lab-evidence/9b/b8-lop4-sau.txt
```

```bash
NGOAI_VAO="$(kubectl -n lab-9b exec ngoai -- \
  wget -q -T 3 -O- "http://$WEB_IP:8080" >/dev/null 2>&1; echo $?)"
TRONG_VAO="$(kubectl -n lab-9b-tenant exec tenant-client -- \
  wget -q -T 3 -O- "http://$WEB_IP:8080" >/dev/null 2>&1; echo $?)"
TRONG_RA="$(kubectl -n lab-9b-tenant exec tenant-client -- \
  wget -q -T 3 -O- "http://$NGOAI_IP:8080" >/dev/null 2>&1; echo $?)"
DNS_OK="$(kubectl -n lab-9b-tenant exec tenant-client -- \
  nslookup kubernetes.default >/dev/null 2>&1; echo $?)"
echo "ngoai->web=$NGOAI_VAO  trong->web=$TRONG_VAO  trong->ngoai=$TRONG_RA  dns=$DNS_OK"

test "$NGOAI_VAO" -ne 0 \
  && echo 'PASS: lop 4 chan chieu den — Pod ngoai namespace khong goi duoc tenant-web'
test "$TRONG_VAO" -eq 0 \
  && echo 'PASS: giao tiep trong noi bo namespace van thong'
test "$TRONG_RA" -ne 0 \
  && echo 'PASS: lop 4 chan chieu di — tenant khong goi ra ngoai namespace duoc'
test "$DNS_OK" -eq 0 \
  && echo 'PASS: ngoai le cho DNS hoat dong — tenant van phan giai duoc ten'
```

**Ý nghĩa:** đây đúng khuôn bài [122](../122-multi-tenancy-vi.md) khuyến nghị cho môi trường đòi
cô lập nghiêm ngặt: **bắt đầu bằng chính sách mặc định từ chối** giao tiếp giữa các Pod, **kèm một
quy tắc cho phép mọi Pod truy vấn DNS**, rồi mới thêm dần quy tắc cho phép trong nội bộ namespace.
Bài cũng khuyên **không** dùng `namespaceSelector: {}` rỗng khi phải cho phép lưu lượng liên
namespace — chính sách trên dùng selector có nhãn cụ thể `kubernetes.io/metadata.name: kube-system`
thay vì để rỗng.

Và nhớ điều kiện tiên quyết mà cả bài [122](../122-multi-tenancy-vi.md) lẫn
[129](../129-security-checklist-vi.md) đều cảnh báo: NetworkPolicy cần **một CNI plugin có hiện
thực nó**, nếu không object vẫn được lưu mà **bị bỏ qua hoàn toàn**. Cluster của bạn thỏa điều kiện
đó từ Lab 5b — gate mở đầu của lab này đã kiểm.

**PASS:** sáu dòng `PASS:` của hai khối trên xuất hiện.

### B8.5. Bốn lớp, bốn câu từ chối khác nhau

```bash
{
  echo "=== $(date -Is) — mo hinh co lap cua lab-9b-tenant ==="
  printf '%-6s %-26s %-42s\n' 'LOP' 'CO CHE' 'CAU TU CHOI QUAN SAT DUOC'
  printf '%-6s %-26s %-42s\n' '1' 'Pod Security Admission' 'violates PodSecurity "restricted:..."'
  printf '%-6s %-26s %-42s\n' '2' 'RBAC' 'kubectl auth can-i tra ve no'
  printf '%-6s %-26s %-42s\n' '3' 'ResourceQuota' 'exceeded quota: tenant-quota'
  printf '%-6s %-26s %-42s\n' '4' 'NetworkPolicy' 'wget het thoi gian cho, exit code khac 0'
  echo '--- object thuoc namespace ma tenant dang co ---'
  kubectl -n lab-9b-tenant get serviceaccount,role,rolebinding,resourcequota,limitrange,networkpolicy,pod
} | tee ~/lab-evidence/9b/b8-bon-lop.txt
```

```bash
OBJ=0
for k in serviceaccount/tenant-sa role/tenant-doc rolebinding/tenant-doc \
         resourcequota/tenant-quota limitrange/tenant-mac-dinh networkpolicy/tenant-co-lap; do
  kubectl -n lab-9b-tenant get "$k" -o name >/dev/null 2>&1 && OBJ=$(( OBJ + 1 )) \
    || echo "THIEU: $k"
done
echo "object cua mo hinh tenant = $OBJ/6"
test "$OBJ" -eq 6 \
  && echo 'PASS: du sau object dung nen mo hinh, tat ca deu thuoc pham vi namespace'
test "$(grep -c 'violates PodSecurity\|exceeded quota' ~/lab-evidence/9b/b8-bon-lop.txt)" -eq 2 \
  && echo 'PASS: ghi lai duoc bang bon lop kem cau tu choi tuong ung'
```

**Ý nghĩa:** giá trị của mô hình nằm ở chỗ bốn lớp **độc lập** với nhau. Thủng một lớp không mở
toang cả namespace: tenant vượt được RBAC vẫn vướng Pod Security, vượt được Pod Security vẫn vướng
quota, và dù chạy được Pod thì vẫn không nói chuyện ra ngoài namespace được. Đó là "phòng thủ theo
chiều sâu" mà checklist [130](../130-application-security-checklist-vi.md) nhắc tới, viết bằng
object thật.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `THIEU:` nào.

### B8.6. Những lớp cô lập bài nêu mà cluster này không có

Ghi lại để không nhầm mô hình vừa dựng là "đa người thuê cứng":

```bash
{
  echo "=== $(date -Is) — lop co lap con thieu so voi bai 122 ==="
  echo 'Sandbox container (gVisor, Kata): can RuntimeClass va mot runtime thay the.'
  kubectl get runtimeclass 2>&1 | head -3
  echo 'Cach ly node: thuc hien bang nodeSelector/taint — da lam o Lab 7a, khong lam lai o day.'
  kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'
  echo 'Control plane ao cho moi tenant: can extension ngoai cluster, ngoai lo trinh.'
  echo 'Service mesh va ma hoa mTLS trong cluster: du an ben thu ba, ngoai lo trinh.'
  echo 'Han che tra cuu DNS lien namespace: phai sua cau hinh CoreDNS — giai doan 21.'
} | tee ~/lab-evidence/9b/b8-lop-con-thieu.txt

test "$(kubectl get runtimeclass --no-headers 2>/dev/null | wc -l)" -eq 0 \
  && echo 'PASS: cluster khong co RuntimeClass nao — khong co lua chon sandbox de dung'
test "$(grep -c 'Control plane ao' ~/lab-evidence/9b/b8-lop-con-thieu.txt)" -eq 1 \
  && echo 'PASS: ghi lai duoc danh sach lop con thieu vao bang chung'
```

**Ý nghĩa:** bài [122](../122-multi-tenancy-vi.md) nói rất rõ rằng "cứng" và "mềm" là một **phổ**,
không phải hai trạng thái. Mô hình bạn vừa dựng cô lập tốt ở **control plane**, nhưng ở **data
plane** thì các tenant vẫn dùng chung kernel của node — container là ảo hóa cấp hệ điều hành, ranh
giới yếu hơn máy ảo. Khi tenant chạy mã **không đáng tin cậy**, bài khuyên dùng sandbox hoặc
cluster riêng. Biết mình đang ở đâu trên phổ đó quan trọng hơn việc dựng thêm một lớp nữa.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B9. Rà cluster theo hai checklist, và bốn bài hardening còn lại

**Mục đích:** bài [129](../129-security-checklist-vi.md) và
[130](../130-application-security-checklist-vi.md) không phải bài đọc — chúng là **công cụ rà
soát**. B9 chạy đúng vai trò đó: mỗi ô có một câu trả lời lấy từ cluster này, ghi vào evidence.
Ba bài [123](../123-hardening-authentication-vi.md), [126](../126-linux-security-vi.md),
[128](../128-api-server-bypass-risks-vi.md) và bài [121](../121-secrets-good-practices-vi.md) được
kiểm ở đúng phần chúng có thể kiểm.

Nhớ cảnh báo mở đầu của cả hai checklist: **bản thân checklist không đủ** để có một thế trận bảo
mật tốt, thứ tự các ô **không phản ánh mức ưu tiên**, và mỗi nhóm ô phải được đánh giá theo giá trị
riêng — có ô quá chặt, có ô quá lỏng so với nhu cầu của bạn.

### B9.1. Checklist 129 — rà phần đọc được từ cluster

```bash
{
  echo "=== $(date -Is) — ra cluster theo checklist 129 ==="
  echo '--- Xac thuc va phan quyen ---'
  echo '[1] binding toi nhom system:masters:'
  kubectl get clusterrolebinding \
    -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{range .subjects[*]}{.kind}{"/"}{.name}{" "}{end}{"\n"}{end}' \
    | grep 'system:masters' || echo '    (khong binding nao)'
  echo '[2] kube-controller-manager co --use-service-account-credentials:'
  sudo grep -c 'use-service-account-credentials' \
    /etc/kubernetes/manifests/kube-controller-manager.yaml

  echo '--- Bao mat mang ---'
  echo '[3] CNI co thuc thi NetworkPolicy: da chung minh o B8.4'
  echo '[4] namespace co NetworkPolicy:'
  kubectl get networkpolicy -A --no-headers 2>/dev/null | awk '{print "    "$1"/"$2}' \
    || echo '    (khong co)'
  echo '[5] so namespace KHONG co NetworkPolicy nao:'
  kubectl get namespace -o name | wc -l
  kubectl get networkpolicy -A --no-headers 2>/dev/null | awk '{print $1}' | sort -u | wc -l

  echo '--- Bao mat Pod ---'
  echo '[6] namespace chua co nhan enforce tuong minh:'
  kubectl get namespaces --selector='!pod-security.kubernetes.io/enforce' --no-headers \
    | awk '{print "    "$1}'
  echo '[7] container KHONG dat limits.memory:'
  kubectl get pods -A \
    -o jsonpath='{range .items[*]}{range .spec.containers[*]}{$.metadata.namespace}{"/"}{$.metadata.name}{" mem="}{.resources.limits.memory}{"\n"}{end}{end}' \
    | grep 'mem=$' | wc -l
  echo '[8] Pod co khai seccompProfile o cap Pod:'
  kubectl get pods -A \
    -o jsonpath='{range .items[*]}{.spec.securityContext.seccompProfile.type}{"\n"}{end}' \
    | grep -c . || true

  echo '--- Log va kiem toan ---'
  echo '[9] apiserver co --audit-policy-file:'
  sudo grep -c 'audit-policy-file' /etc/kubernetes/manifests/kube-apiserver.yaml

  echo '--- Secret ---'
  echo '[10] apiserver co --encryption-provider-config:'
  sudo grep -c 'encryption-provider-config' /etc/kubernetes/manifests/kube-apiserver.yaml
  echo '[11] Secret kieu service-account-token dai han dang ton tai:'
  kubectl get secret -A --field-selector type=kubernetes.io/service-account-token \
    --no-headers 2>/dev/null | wc -l

  echo '--- Image ---'
  echo '[12] container dang dung tag latest:'
  kubectl get pods -A \
    -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
    | grep -c ':latest$' || true

  echo '--- Admission controller ---'
  echo '[13] danh sach plugin dang chay: xem b7-admission-plugin.txt'
  grep -c . ~/lab-evidence/9b/b7-admission-plugin.txt
} | tee ~/lab-evidence/9b/b9-checklist-129.txt
```

```bash
test "$(grep -c '^\[' ~/lab-evidence/9b/b9-checklist-129.txt)" -eq 13 \
  && echo 'PASS: du muoi ba o cua checklist 129 co cau tra loi tu cluster nay'

test "$(sudo grep -c 'use-service-account-credentials' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml)" -ge 1 \
  && echo 'PASS: o [2] dat — kube-controller-manager chay voi --use-service-account-credentials'
test "$(sudo grep -c 'encryption-provider-config' \
  /etc/kubernetes/manifests/kube-apiserver.yaml)" -eq 0 \
  && echo 'PASS: o [10] KHONG dat va lab ghi ro dieu do — day la no #6, tra o giai doan 22'
LATEST="$(kubectl get pods -A \
  -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
  | grep -c ':latest$' || true)"
test "$LATEST" -eq 0 \
  && echo 'PASS: o [12] dat — khong container nao dang chay bang tag latest'
```

**Ý nghĩa:** file bạn vừa ghi là sản phẩm thật của mục này — không phải dòng `PASS:`. Hai ô đáng chú
ý nhất:

- Ô **[10]** không đạt, và lab **không sửa**. Bài [121](../121-secrets-good-practices-vi.md) nói
  Secret mặc định được lưu **không mã hóa** trong etcd, và base64 **không phải** mã hóa. Đây là
  **nợ #6** trong [sổ nợ lab](README.md#5-sổ-nợ-lab), trả ở
  [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) — vẫn **chưa
  trả** sau lab này.
- Ô **[6]** liệt kê chính xác những namespace mà chính sách Pod chưa với tới. Trên cluster
  production, danh sách đó là việc cần làm tiếp theo, và B3.5 đã cho bạn công cụ đánh giá trước khi
  siết.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B9.2. Checklist 130 — tự soi Pod của chính mình

Đối tượng soi là `tenant-web` ở B8, đọc lại **từ object đã lưu trong API**, không từ file manifest:

```bash
{
  echo "=== $(date -Is) — soi tenant-web theo checklist 130 ==="
  printf '%-42s %s\n' 'serviceAccountName' \
    "$(kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath='{.spec.serviceAccountName}')"
  printf '%-42s %s\n' 'automountServiceAccountToken' \
    "$(kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath='{.spec.automountServiceAccountToken}')"
  printf '%-42s %s\n' 'securityContext.runAsNonRoot (Pod)' \
    "$(kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath='{.spec.securityContext.runAsNonRoot}')"
  printf '%-42s %s\n' 'securityContext.runAsUser (Pod)' \
    "$(kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath='{.spec.securityContext.runAsUser}')"
  printf '%-42s %s\n' 'allowPrivilegeEscalation (container)' \
    "$(kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath='{.spec.containers[0].securityContext.allowPrivilegeEscalation}')"
  printf '%-42s %s\n' 'privileged (container)' \
    "$(kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath='{.spec.containers[0].securityContext.privileged}')"
  printf '%-42s %s\n' 'capabilities.drop (container)' \
    "$(kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath='{.spec.containers[0].securityContext.capabilities.drop[*]}')"
  printf '%-42s %s\n' 'readOnlyRootFilesystem (container)' \
    "$(kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath='{.spec.containers[0].securityContext.readOnlyRootFilesystem}')"
  printf '%-42s %s\n' 'resources.limits.memory' \
    "$(kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath='{.spec.containers[0].resources.limits.memory}')"
  printf '%-42s %s\n' 'image' \
    "$(kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath='{.spec.containers[0].image}')"
  printf '%-42s %s\n' 'NetworkPolicy phu namespace' \
    "$(kubectl -n lab-9b-tenant get networkpolicy -o jsonpath='{.items[*].metadata.name}')"
} | tee ~/lab-evidence/9b/b9-checklist-130.txt
```

```bash
G() { kubectl -n lab-9b-tenant get pod tenant-web -o jsonpath="$1"; }

test "$(G '{.spec.serviceAccountName}')" = 'tenant-sa' \
  && echo 'PASS: khong dung ServiceAccount default — moi workload mot danh tinh rieng'
test "$(G '{.spec.automountServiceAccountToken}')" = 'false' \
  && echo 'PASS: khong tu dong mount token vi Pod nay khong goi Kubernetes API'
test "$(G '{.spec.securityContext.runAsNonRoot}')" = 'true' \
  && test "$(G '{.spec.securityContext.runAsUser}')" = '1000' \
  && echo 'PASS: chay bang nguoi dung khong dac quyen, uid tuong minh'
test "$(G '{.spec.containers[0].securityContext.allowPrivilegeEscalation}')" = 'false' \
  && echo 'PASS: da vo hieu hoa leo thang dac quyen'
test -z "$(G '{.spec.containers[0].securityContext.privileged}')" \
  && echo 'PASS: khong dat privileged'
echo "$(G '{.spec.containers[0].securityContext.capabilities.drop[*]}')" | grep -qw 'ALL' \
  && echo 'PASS: da drop toan bo capability'
test -n "$(G '{.spec.containers[0].resources.limits.memory}')" \
  && echo 'PASS: co gioi han bo nho'
test -n "$(kubectl -n lab-9b-tenant get networkpolicy -o name)" \
  && echo 'PASS: co NetworkPolicy phu Pod nay'

test -z "$(G '{.spec.containers[0].securityContext.readOnlyRootFilesystem}')" \
  && echo 'GHI NHAN: readOnlyRootFilesystem CHUA dat — xem ghi chu ben duoi'
```

**Ý nghĩa:** dòng cuối cùng cố ý **không** là `PASS:`. `tenant-web` ghi `index.html` vào `/tmp` nằm
trên root filesystem của container, nên bật `readOnlyRootFilesystem: true` sẽ làm nó chết ngay —
đúng như B4.7 đã chỉ ra. Cách sửa đúng là thêm một volume ghi được rồi mới bật cờ, và đó là công
việc của người viết ứng dụng chứ không phải một dòng thêm vào manifest.

Ghi nhận một ô **chưa đạt** kèm lý do là cách dùng checklist đúng đắn: nó là danh sách để đối chiếu
và cải thiện liên tục, không phải bài kiểm tra để tô xanh hết mọi ô.

**PASS:** tám dòng `PASS:` xuất hiện, cộng đúng một dòng `GHI NHAN:`.

### B9.3. Bài 123 — cơ chế xác thực nào thật sự đang bật

```bash
{
  echo "=== $(date -Is) — co che xac thuc dang bat tren cluster nay ==="
  for f in client-ca-file token-auth-file oidc-issuer-url \
           authentication-token-webhook-config-file requestheader-client-ca-file \
           service-account-key-file service-account-issuer anonymous-auth; do
    printf '%-42s %s\n' "--$f" \
      "$(sudo grep -c -- "--$f" /etc/kubernetes/manifests/kube-apiserver.yaml)"
  done
  echo '--- bootstrap token con song trong kube-system ---'
  kubectl -n kube-system get secret \
    --field-selector type=bootstrap.kubernetes.io/token --no-headers 2>/dev/null | wc -l
} | tee ~/lab-evidence/9b/b9-xac-thuc.txt
```

```bash
test "$(sudo grep -c -- '--token-auth-file' \
  /etc/kubernetes/manifests/kube-apiserver.yaml)" -eq 0 \
  && echo 'PASS: KHONG dung file token tinh — dung khuyen nghi cua bai 123'
test "$(sudo grep -c -- '--oidc-issuer-url' \
  /etc/kubernetes/manifests/kube-apiserver.yaml)" -eq 0 \
  && echo 'PASS: khong tich hop OIDC — cluster lab khong co identity provider ben ngoai'
test "$(sudo grep -c -- '--client-ca-file' \
  /etc/kubernetes/manifests/kube-apiserver.yaml)" -ge 1 \
  && echo 'PASS: xac thuc bang client certificate X.509 dang bat — day la co che ban dung'
test "$(sudo grep -c -- '--requestheader-client-ca-file' \
  /etc/kubernetes/manifests/kube-apiserver.yaml)" -ge 1 \
  && echo 'PASS: co ca cau hinh proxy xac thuc, dung cho tang aggregation'
```

**Ý nghĩa:** cluster này xác thực người dùng bằng **client certificate X.509** — đúng cơ chế bài
[123](../123-hardening-authentication-vi.md) nói là dùng cho thành phần hệ thống nhưng **không phù
hợp cho production nhiều người dùng**, vì sáu lý do bài liệt kê. Ba lý do đáng nhớ nhất, và cả ba
đều đúng với kubeconfig bạn đang cầm: certificate **không thu hồi được từng cái** — một khi lộ, nó
dùng được tới khi hết hạn; **không có bản ghi lưu vĩnh viễn** về những certificate đã cấp; và **tư
cách thành viên nhóm nằm trong trường `O`** của certificate, nên không đổi được trong suốt vòng đời
của nó.

Ghi nhớ nguyên tắc bao trùm của bài: **bật càng ít cơ chế xác thực càng tốt**, vì Kubernetes không
có cơ sở dữ liệu người dùng — muốn kiểm toán quyền truy cập thì phải rà **mọi** nguồn xác thực đã
cấu hình.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B9.4. Bài 126 — Secret trên node, swap, và AppArmor

Tạo một Secret và một Pod hai container; container `phu` **không** mount Secret:

```bash
kubectl -n lab-9b create secret generic db-cred \
  --from-literal=mat-khau='lab-9b-bi-mat' --dry-run=client -o yaml \
  | kubectl apply -n lab-9b -f -

cat > ~/lab-work/9b/b9-hai-container.yaml <<'YAMLEOF'
apiVersion: v1
kind: Pod
metadata:
  name: hai-container
  labels:
    lab: "9b"
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  volumes:
    - name: db
      secret:
        secretName: db-cred
  containers:
    - name: app
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: db
          mountPath: /etc/db
          readOnly: true
    - name: phu
      image: busybox:1.37
      command: ["sh", "-c", "sleep 3600"]
YAMLEOF

kubectl apply -n lab-9b -f ~/lab-work/9b/b9-hai-container.yaml
kubectl -n lab-9b wait --for=condition=Ready pod/hai-container --timeout=180s

kubectl -n lab-9b exec hai-container -c app -- mount \
  | grep '/etc/db' | tee ~/lab-evidence/9b/b9-secret-tmpfs.txt

{
  echo '--- swap tren ca ba node ---'
  for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
    printf '%-18s swap_dong=%s\n' "$n" "$(ssh "$n" 'swapon --show' | grep -c . || true)"
  done
  echo '--- AppArmor bat tren ca ba node ---'
  for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
    printf '%-18s apparmor=%s\n' "$n" \
      "$(ssh "$n" 'cat /sys/module/apparmor/parameters/enabled 2>/dev/null || echo N')"
  done
} | tee ~/lab-evidence/9b/b9-node-linux.txt
```

```bash
grep -q 'type tmpfs' ~/lab-evidence/9b/b9-secret-tmpfs.txt \
  && echo 'PASS: volume Secret duoc hien thuc bang tmpfs — dung nhu bai 126 mo ta'
test "$(grep -c 'swap_dong=0' ~/lab-evidence/9b/b9-node-linux.txt)" -eq 3 \
  && echo 'PASS: ca ba node deu khong co swap — dieu kien kich hoat rui ro cua bai khong ton tai'
test "$(grep -c 'apparmor=Y' ~/lab-evidence/9b/b9-node-linux.txt)" -eq 3 \
  && echo 'PASS: AppArmor bat tren ca ba node Ubuntu — khop voi B4.6 va voi bai 127'
```

**Ý nghĩa:** hai câu của bài [126](../126-linux-security-vi.md) vừa được nối lại. Volume lưu trong
bộ nhớ — mount kiểu `secret`, hoặc `emptyDir` với `medium: Memory` — dùng `tmpfs`. Nhưng `tmpfs`
**không** đảm bảo dữ liệu không chạm đĩa: nếu node **có** swap và kernel cũ (hoặc kernel mới nhưng
cấu hình Kubernetes không được hỗ trợ), trang bộ nhớ vẫn có thể bị đẩy xuống lưu trữ bền vững. Ba
node của bạn tắt swap theo yêu cầu của kubeadm, nên điều kiện đó không phát sinh. Nếu sau này bật
swap, bài đòi kernel từ 6.3 trở lên — phiên bản hỗ trợ chính thức tùy chọn `noswap` — hoặc kernel
có backport.

Kết quả AppArmor cũng chốt lại lựa chọn ở [mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành): node
Ubuntu bật AppArmor, và bài [127](../127-linux-kernel-security-vi.md) nói hệ điều hành thường chỉ
gồm **một trong hai** cơ chế MAC — nên trên cluster này không có chính sách SELinux nào để quan sát.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B9.5. Bài 128 — bốn đường vòng, kiểm phần đọc được

```bash
{
  echo "=== $(date -Is) — bien phap giam thieu bon duong vong qua API server ==="
  echo '[A] thu muc static Pod manifest tren lab-k8s-master:'
  sudo stat -c '    chu so huu=%U:%G quyen=%a %n' /etc/kubernetes/manifests
  sudo ls -1 /etc/kubernetes/manifests | sed 's/^/    /'

  echo '[B] cong chi-doc cua kubelet (10255) va cong co xac thuc (10250):'
  for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
    IP="$(kubectl get node "$n" \
      -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
    RO="$(curl -s -o /dev/null -m 3 -w '%{http_code}' "http://$IP:10255/pods" || echo 000)"
    AU="$(curl -sk -o /dev/null -m 5 -w '%{http_code}' "https://$IP:10250/pods" || echo 000)"
    printf '    %-18s ip=%-15s readonly_10255=%s anonymous_10250=%s\n' "$n" "$IP" "$RO" "$AU"
  done

  echo '[C] etcd dang lang nghe o dau:'
  sudo ss -ltn 2>/dev/null | grep ':2379' | sed 's/^/    /' || echo '    (khong doc duoc)'

  echo '[D] socket cua container runtime:'
  sudo stat -c '    chu so huu=%U:%G quyen=%a %n' /run/containerd/containerd.sock
} | tee ~/lab-evidence/9b/b9-bypass-128.txt
```

```bash
test "$(grep -c '^\[' ~/lab-evidence/9b/b9-bypass-128.txt)" -eq 4 \
  && echo 'PASS: da ra du bon duong vong ma bai 128 liet ke'

sudo stat -c '%U' /etc/kubernetes/manifests | grep -qx 'root' \
  && echo 'PASS: [A] thu muc static Pod manifest thuoc so huu root'
test "$(grep -c 'readonly_10255=000' ~/lab-evidence/9b/b9-bypass-128.txt)" -eq 3 \
  && echo 'PASS: [B] cong chi-doc 10255 cua kubelet KHONG mo tren ca ba node'
test "$(grep -cE 'anonymous_10250=(401|403)' ~/lab-evidence/9b/b9-bypass-128.txt)" -eq 3 \
  && echo 'PASS: [B] API kubelet doi xac thuc — truy cap an danh bi tu choi tren ca ba node'
test "$(grep -c '0.0.0.0:2379' ~/lab-evidence/9b/b9-bypass-128.txt)" -eq 0 \
  && echo 'PASS: [C] etcd khong lang nghe tren moi dia chi'
sudo stat -c '%U' /run/containerd/containerd.sock | grep -qx 'root' \
  && echo 'PASS: [D] socket container runtime thuoc so huu root'
```

**Ý nghĩa:** bốn đường này có một điểm chung khiến chúng nguy hiểm hơn một request đặc quyền cao đi
qua API server: chúng **không chịu admission control** và **không được audit logging của Kubernetes
ghi lại**. Kẻ đi đường này không để lại dấu vết trong audit log của cluster.

Hai điều đáng nhớ nhất từ bài [128](../128-api-server-bypass-risks-vi.md), gắn với đúng những gì
bạn vừa đọc:

- Thư mục `[A]` chứa chính các thành phần control plane. Ai ghi được vào đó thì **sửa được static
  Pod hiện có hoặc đưa vào static Pod mới**, và Pod mới đó vẫn dùng được `hostPath` mount từ node.
  Tệ hơn: một static Pod **không vượt qua admission control vẫn chạy trên node**, kubelet chỉ không
  đăng ký nó với API server — nghĩa là Pod chạy mà `kubectl get pods` không thấy.
- Với `[C]`, bài nói etcd **chưa có mô hình cấp quyền được chấp nhận rộng rãi**: bất kỳ certificate
  nào do CA mà etcd tin tưởng phát hành đều cho **toàn quyền đọc ghi**, kể cả certificate vốn chỉ
  dùng để health check. Vì vậy hạn chế ở tầng mạng và kiểm soát private key là biện pháp chính.

**PASS:** sáu dòng `PASS:` của bước này xuất hiện.

### B9.6. Bài 121 — Secret chỉ tới đúng container cần nó

```bash
kubectl -n lab-9b exec hai-container -c app -- cat /etc/db/mat-khau \
  > ~/lab-evidence/9b/b9-secret-app.txt 2>&1
kubectl -n lab-9b exec hai-container -c phu -- ls /etc/db \
  > ~/lab-evidence/9b/b9-secret-phu.txt 2>&1; RC_PHU=$?
cat ~/lab-evidence/9b/b9-secret-app.txt
cat ~/lab-evidence/9b/b9-secret-phu.txt
echo "exit code cua container phu = $RC_PHU"
```

```bash
grep -q 'lab-9b-bi-mat' ~/lab-evidence/9b/b9-secret-app.txt \
  && echo 'PASS: container app doc duoc Secret qua volume mount'
test "$RC_PHU" -ne 0 \
  && echo 'PASS: container phu KHONG co duong dan do — Secret khong toi duoc no'
test "$(kubectl -n lab-9b get pod hai-container \
  -o jsonpath='{.spec.containers[1].volumeMounts[*].name}' | grep -c 'db')" -eq 0 \
  && echo 'PASS: object trong API xac nhan container thu hai khong mount volume Secret'
```

**Ý nghĩa:** đây là khuyến nghị *Giới hạn quyền truy cập Secret cho các container cụ thể* của bài
[121](../121-secrets-good-practices-vi.md). Volume khai ở **cấp Pod**, nhưng `volumeMounts` khai ở
**cấp container** — nếu bạn mount cho mọi container thì một lỗ hổng ở container không liên quan
cũng thành đường lấy mật khẩu cơ sở dữ liệu.

Nhớ thêm hai điều bài nêu mà lab không cần dựng lại vì Lab 9a đã chứng minh phần RBAC: **`list`
trên Secret ngầm cho phép lấy được nội dung**, và **ai tạo được Pod dùng một Secret thì cũng xem
được giá trị của Secret đó** — RBAC không chặn được đường thứ hai, chỉ có Secret ngắn hạn và quy
tắc kiểm toán mới giảm thiểu được.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B9.7. Bài 254 — cluster này không có cloud provider

```bash
{
  echo "=== $(date -Is) — cloud provider tren cluster nay ==="
  echo '[1] pod cloud-controller-manager:'
  kubectl get pods -A -o name | grep -i 'cloud-controller' || echo '    (khong co)'
  echo '[2] co --cloud-provider trong manifest control plane:'
  sudo grep -c -- '--cloud-provider' /etc/kubernetes/manifests/kube-apiserver.yaml
  sudo grep -c -- '--cloud-provider' /etc/kubernetes/manifests/kube-controller-manager.yaml
  echo '[3] node mang taint node.cloudprovider.kubernetes.io/uninitialized:'
  kubectl get nodes \
    -o jsonpath='{range .items[*]}{.metadata.name}{" "}{range .spec.taints[*]}{.key}{" "}{end}{"\n"}{end}' \
    | grep -c 'cloudprovider' || true
  echo '[4] providerID cua tung node:'
  kubectl get nodes \
    -o custom-columns='NODE:.metadata.name,PROVIDER:.spec.providerID' | sed 's/^/    /'
} | tee ~/lab-evidence/9b/b9-cloud-provider.txt
```

```bash
test "$(kubectl get pods -A -o name | grep -ci 'cloud-controller' || true)" -eq 0 \
  && echo 'PASS: khong co cloud-controller-manager nao chay tren cluster'
test "$(sudo grep -c -- '--cloud-provider' \
  /etc/kubernetes/manifests/kube-controller-manager.yaml)" -eq 0 \
  && echo 'PASS: kube-controller-manager khong khai --cloud-provider'
test "$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' \
  | grep -c 'cloudprovider' || true)" -eq 0 \
  && echo 'PASS: khong node nao ket o taint uninitialized cua cloud provider'
```

**Ý nghĩa:** kết quả này khép lại ba ô cùng lúc. Bài
[254](../254-running-cloud-controller-vi.md) nói thành phần chỉ định `--cloud-provider=external` sẽ
thêm taint `node.cloudprovider.kubernetes.io/uninitialized` với effect `NoSchedule` lúc khởi tạo,
và node mới sẽ **kẹt ở trạng thái không lập lịch được** nếu cloud controller manager không khả
dụng. Cluster của bạn không có thành phần đó, nên không có rủi ro đó — và cũng vì thế, hai ô của
checklist [129](../129-security-checklist-vi.md) về **lọc truy cập tới cloud metadata API
`169.254.169.254`** và **hạn chế `LoadBalancer` cùng `externalIPs`** không áp dụng: không có
metadata API để lọc, và không có cloud load balancer để cấp phát.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B10. Cleanup và gate cuối

**Mục đích:** xóa mọi thứ lab tạo ra — **kể cả Pod đặc quyền** — chứng minh tồn kho object trở về
đúng con số cũ, chứng minh cấu hình control plane và kubelet không hề bị sửa, và chứng minh cluster
đã về đúng `03-storage-ready`.

### B10.1. Xóa object của bài học

```bash
kubectl delete namespace lab-9b lab-9b-baseline lab-9b-restricted lab-9b-batche lab-9b-tenant \
  --wait=true --timeout=300s
```

```bash
rm -f ~/lab-work/9b/*.yaml
rmdir ~/lab-work/9b

unset PSA_VER K8S_MAJOR K8S_MINOR GITV W2 \
      CAP_TRAN NNP_TRAN SEC_TRAN AA_TRAN UID_TRAN \
      CAP_DROP CAP_ADD CAP_PRIV S_RD S_UNC S_PRIV NNP_TAT NNP_PRIV AA_RD AA_PRIV \
      WEB_IP NGOAI_IP SA FS_UID PL_UID FS_NAME PL_NAME
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến điều
đó thành gate thay vì im lặng bỏ qua. Nếu nó fail, **liệt kê thư mục trước khi xóa tay**. Thư mục
`~/lab-evidence/9b/` **giữ lại** — đó là bằng chứng, và với lab này nó còn là hồ sơ rà soát bảo mật
của cluster.

```bash
NS_LEFT="$(kubectl get namespace -l lab=9b --no-headers 2>/dev/null | wc -l)"
POD_LEFT="$(kubectl get pods -A -l lab=9b --no-headers 2>/dev/null | wc -l)"
echo "namespace cua lab con = $NS_LEFT | pod mang label lab=9b con = $POD_LEFT"

test "$NS_LEFT" -eq 0 && echo 'PASS: nam namespace cua lab da bien mat'
test "$POD_LEFT" -eq 0 && echo 'PASS: khong Pod nao cua lab con song'
test ! -e ~/lab-work/9b && echo 'PASS: thu muc manifest tam da xoa'
```

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B10.2. Tồn kho trở về đúng con số cũ

```bash
CR_N9="$(kubectl get clusterrole --no-headers | wc -l)"
CRB_N9="$(kubectl get clusterrolebinding --no-headers | wc -l)"
NS_N9="$(kubectl get namespace --no-headers | wc -l)"
NP_N9="$(kubectl get networkpolicy -A --no-headers 2>/dev/null | wc -l)"
PRIV_N9="$(kubectl get pods -A \
  -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.securityContext.privileged}{"\n"}{end}{end}' \
  | grep -c '^true$' || true)"

{
  echo "clusterrole:         truoc=$CR_N0   sau=$CR_N9"
  echo "clusterrolebinding:  truoc=$CRB_N0  sau=$CRB_N9"
  echo "namespace:           truoc=$NS_N0   sau=$NS_N9"
  echo "networkpolicy:       truoc=$NP_N0   sau=$NP_N9"
  echo "container privileged: truoc=$PRIV_N0 sau=$PRIV_N9"
} | tee ~/lab-evidence/9b/b10-ton-kho.txt

test "$CR_N9" -eq "$CR_N0" && test "$CRB_N9" -eq "$CRB_N0" \
  && echo 'PASS: lab khong de lai object pham vi cluster nao'
test "$NS_N9" -eq "$NS_N0" && test "$NP_N9" -eq "$NP_N0" \
  && echo 'PASS: so namespace va so NetworkPolicy tro ve dung con so truoc lab'
test "$PRIV_N9" -eq "$PRIV_N0" \
  && echo 'PASS: so container dac quyen tro ve dung con so truoc lab — khong cai nao sot lai'
```

**Ý nghĩa:** gate cuối cùng là gate **bắt buộc** của một lab có tạo Pod đặc quyền. Một container
`privileged: true` bị bỏ quên không làm gate nào đỏ, không làm Pod nào crash, và không ai nhận ra
cho tới khi có sự cố — nhưng nó vô hiệu hóa seccomp, AppArmor và toàn bộ giới hạn capability trên
đúng node nó đang chạy. So **giá trị** với con số đọc ở B0.5 là cách duy nhất chứng minh không còn
cái nào.

Cũng nhớ vì sao gate không đòi con số bằng 0: CNI và kube-proxy trong `kube-system` **phải** chạy
đặc quyền. Đó là workload cấp hạ tầng, đúng đối tượng của profile *Privileged*.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B10.3. Cấu hình control plane và kubelet không đổi

```bash
{
  echo "apiserver-manifest $(sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml \
    | awk '{print $1}')"
  echo "kubelet-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "kubelet-$n $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
} | tee ~/lab-evidence/9b/b10-cauhinh-sha.txt

diff -u ~/lab-evidence/9b/b0-cauhinh-sha.txt ~/lab-evidence/9b/b10-cauhinh-sha.txt \
  && echo 'PASS: manifest apiserver va cau hinh kubelet ba node khong he doi trong suot lab' \
  || echo 'FAIL: cau hinh control plane hoac kubelet da bi sua — xem muc 4'

RS_OK=0
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  A="$(kubectl get node "$n" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
  test "$A" = 'True' && RS_OK=$(( RS_OK + 1 ))
done
test "$RS_OK" -eq 3 && echo 'PASS: ca ba node van Ready sau khi lab ket thuc'
```

**Ý nghĩa:** cả lab này xoay quanh những thứ do cấu hình control plane quyết định — bộ mặc định của
Pod Security Admission, danh sách admission plugin, audit log, danh sách sysctl không an toàn được
phép, cơ chế xác thực đang bật. Cám dỗ "bật tạm một cờ để xem cơ chế kia chạy thế nào" là có thật,
và hậu quả không dừng ở lab này: **mọi lab sau đều bắt đầu từ `03-storage-ready`**. Nếu `diff` báo
khác, restore cả ba VM trước khi sang lab sau.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B10.4. Gate ngắn A5.5 và kết thúc

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
kubectl get storageclass | grep -q '(default)' \
  && echo 'PASS: van con StorageClass mac dinh cua moc 03-storage-ready'

{
  echo "=== $(date -Is) — trang thai sau Lab 9b ==="
  kubectl get namespaces --show-labels
  kubectl get pods -A -o wide
  kubectl get storageclass -o wide
  kubectl get pv
} | tee ~/lab-evidence/9b/b10-sau.txt

ls -l ~/lab-evidence/9b/ | tee ~/lab-evidence/9b/b10-danh-sach-bang-chung.txt
test "$(ls -1 ~/lab-evidence/9b/ | wc -l)" -gt 20 \
  && echo 'PASS: ho so ra soat bao mat cua cluster da duoc luu day du'
```

**PASS:** ba node `Ready`; dòng `PASS: readyz ok` xuất hiện; lệnh field selector trả
`No resources found`; CoreDNS đủ replica `READY`; `default` không có Pod; hai dòng `PASS:` còn lại
xuất hiện; danh sách namespace không còn namespace nào mang tiền tố `lab-9b`.

Cluster đã trở về `03-storage-ready`. **Lab 9b không tạo snapshot mới** — để ba VM nguyên trạng
đang chạy hoặc tắt tùy bạn; lab sau tự bật máy theo
[A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab).

---

## 3. Checkpoint 9b

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Ba profile Pod Security Standards tên là gì, và **thứ tự tích lũy** của chúng ra sao? Kể **ít
      nhất bốn** kiểm soát mà Baseline cấm, rồi kể **bốn** thứ Restricted **thêm** vào trên
      Baseline. Với mỗi nhóm, nói rõ bạn dựng lại bằng bước nào trong lab.
- [ ] Ba chế độ `enforce`, `audit`, `warn` làm gì khi phát hiện vi phạm? Bạn quan sát được sự khác
      nhau của chúng bằng **ba dấu hiệu** nào, và **dấu hiệu nào của `audit`** cluster này **không**
      quan sát được — vì sao, và nó nằm ở giai đoạn nào?
- [ ] Bạn `kubectl apply` một Deployment có container `privileged: true` vào một namespace đặt
      `pod-security.kubernetes.io/enforce: restricted`. Lệnh apply thành công hay thất bại? Pod có
      chạy không? Bạn nhìn ở **đâu** để biết chuyện gì đã xảy ra? Nếu namespace đó cũng bật `warn`
      thì khác gì?
- [ ] Một namespace có `enforce=privileged`, `warn=baseline`, `audit=restricted`. Với ba Pod — một
      cái đặc quyền, một cái chỉ trượt Restricted, một cái đạt Restricted — điều gì xảy ra với từng
      cái? Vì sao cấu hình ba mức như thế lại là bước chuẩn khi siết một cluster đang chạy?
- [ ] Đồng nghiệp nói: "namespace của tôi chưa dán nhãn nào, nên Pod Security Admission chưa can
      thiệp — coi như nó chặt nhất." Sai ở đâu? Cluster mặc định là mức nào, ai quyết định con số
      đó, và bạn dùng **lệnh nào** để liệt kê mọi namespace đang ở tình trạng ấy?
- [ ] Bạn có một Pod chạy được và muốn biết nó thật sự bị ràng buộc tới đâu. Kể **bốn** chỗ đọc
      được trạng thái thật từ bên trong container, và mỗi chỗ trả lời cho **trường nào** trong
      `securityContext`. Vì sao đọc manifest là chưa đủ?
- [ ] Một manifest khai `seccompProfile: RuntimeDefault`, một AppArmor profile mặc định, và
      `privileged: true`. Ba ràng buộc kernel còn tác dụng gì? Bạn chứng minh bằng ba giá trị nào?
      Rồi trả lời tiếp: cách làm đúng khi ứng dụng thật sự cần một quyền quản trị hệ điều hành là
      gì?
- [ ] `runAsNonRoot: true` chặn Pod ở **mặt phẳng nào**, và Pod Security Admission chặn ở **mặt
      phẳng nào**? Với mỗi cái, nói rõ bạn thấy gì trong `kubectl get pods` và bạn điều tra bằng
      lệnh nào. Cùng câu hỏi đó cho một Pod dùng sysctl **không an toàn** trong hai namespace khác
      nhau.
- [ ] Phân biệt sysctl an toàn và không an toàn theo đúng ba tiêu chí của tài liệu. Vì sao không
      thể đặt sysctl bằng cách `echo` vào `/proc/sys` từ trong container? Nếu bạn thật sự cần một
      sysctl không an toàn thì quy trình đúng gồm những bước nào?
- [ ] PodSecurityPolicy bị **deprecated** ở phiên bản nào và bị **gỡ bỏ** ở phiên bản nào? Hai mốc
      đó khác nhau ra sao về hậu quả với người vận hành? Trong năm bước của lộ trình di trú, những
      bước nào **vẫn còn giá trị** cho một cluster chưa từng có PSP?
- [ ] `failurePolicy` mặc định của một admission webhook là gì, và điều đó gây ra chuyện gì khi
      webhook chết? Bài khuyên mutating webhook nên đặt giá trị nào, và **cơ chế nào bù lại** phần
      thực thi chính sách bị mất? Kể thêm hai phạm vi mà bài dặn tuyệt đối không cho webhook đụng
      vào.
- [ ] Bốn lớp cô lập của một namespace tenant là gì, mỗi lớp đứng ở mặt phẳng nào, và **câu từ chối
      đặc trưng** của từng lớp trông ra sao? Nếu tenant có quyền `patch` trên Namespace thì lớp nào
      sụp trước, và vì sao ba lớp còn lại không cứu được?

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Luồng một Pod vi phạm chính sách, từ `kubectl apply` tới lúc bạn biết mình sai.** Bắt đầu từ
   một manifest có `privileged: true`. Kể đủ: request đi qua ba chặng nào trước khi tới Pod
   Security Admission và nó nằm ở chặng nào; plugin đó tra **cái gì** trên Namespace để biết phải
   áp mức nào và phiên bản nào; ba chế độ cho ra ba hành vi gì; vì sao cùng manifest ấy gói trong
   một Deployment lại cho kết quả khác hẳn, và bạn nhìn vào **object nào** để tìm ra nguyên nhân.
   Kết thúc bằng: nếu Pod **được** tạo, kể tiếp đường nó đi tới kubelet, ba thứ mà kubelet còn có
   thể từ chối, và cách phân biệt từ chối của kubelet với từ chối của admission chỉ bằng
   `kubectl get pods`.
2. **Luồng dựng một namespace cho một đội mới trên cluster dùng chung.** Bắt đầu từ lúc chưa có gì.
   Kể đủ: bốn lớp bạn dựng theo thứ tự nào và vì sao thứ tự đó; với mỗi lớp, object nào được tạo,
   nó **thuộc namespace** hay **phạm vi cluster**, và điều đó ảnh hưởng thế nào tới việc dọn dẹp;
   bạn kiểm chứng từng lớp bằng phép thử nào. Rồi kể phần đánh giá trước khi siết: dùng chế độ nào
   và lệnh nào để biết trước điều gì sẽ gãy, và giới hạn của cách đó. Kết thúc bằng câu hỏi ngược:
   mô hình này **không** bảo vệ được điều gì, tenant chạy mã không đáng tin cậy thì bạn phải thêm
   lớp nào, và vì sao lớp đó không thuộc lộ trình ở giai đoạn này.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm **profile** với **security context**,
**`violates`** với **`would violate`**, **`enforce`** với **`warn`**, **chặn ở admission** với
**chặn ở kubelet**, hay **`deprecated`** với **`removed`** — phần policy của
[giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) hoàn tất.

Lab này **không phát sinh nợ mới** và **không trả nợ nào**; xem
[sổ nợ lab](README.md#5-sổ-nợ-lab). Riêng **nợ #6 — mã hóa Secret at rest** liên quan trực tiếp tới
ô `[10]` mà B9.1 vừa rà: cluster **chưa** bật `--encryption-provider-config`, và nợ đó vẫn **chưa
trả** — nó được trả ở
[giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu), không phải ở đây.
Phần đọc audit annotation của chế độ `audit` cũng thuộc giai đoạn đó. Những phần cố ý không làm
khác đều nằm trong bảng lý do ở [mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ liệt kê
sự cố phát sinh từ nội dung bài học 9b.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| B0.2: gate `PSA_VER` khớp `gitVersion` báo `FAIL` | `kubectl get --raw /version` | Bạn đang nối vào một cluster khác baseline, hoặc `minor` có hậu tố. Đối chiếu [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và restore đúng snapshot; **không** sửa `PSA_VER` bằng tay để gate xanh |
| B0.3: namespace tạo ra nhưng nhãn `enforce-version` rỗng | `kubectl get ns lab-9b-baseline -o yaml` | Heredoc đã bị đóng ngoặc nháy nên `$PSA_VER` không nở. Dùng đúng `<<EOF` (không nháy) cho file namespace, và `<<'YAMLEOF'` cho mọi manifest Pod |
| B1.3: Pod `mac-dinh` bị từ chối | `kubectl get ns lab-9b --show-labels` | `lab-9b` đã bị dán nhãn PSA từ lần chạy trước hoặc do B3.5 chạy thiếu `--dry-run`. Xóa namespace, tạo lại từ B0.3, và kiểm lại rằng mọi lệnh `kubectl label` trong lab đều kèm `--dry-run=server` |
| B2.2: số câu `violates PodSecurity` ít hơn 5 | `cat ~/lab-evidence/9b/b2-vi-pham-baseline.txt` | Một Pod trong bộ năm cái đã tồn tại từ lần chạy trước nên `apply` chỉ cập nhật. Xóa hết Pod trong `lab-9b-baseline` rồi chạy lại bước đó |
| B2.2: câu từ chối ghi một phiên bản khác `$PSA_VER` | `kubectl get ns lab-9b-baseline -o jsonpath='{.metadata.labels}'` | Nhãn `enforce-version` bị thiếu nên plugin dùng `latest`. Áp lại manifest namespace của B0.3 |
| B2.5: `ReplicaFailure` mãi không xuất hiện | `kubectl -n lab-9b-restricted get rs -o yaml` | ReplicaSet chưa kịp thử tạo Pod, hoặc selector sai nên `$RSN` rỗng. Tăng số vòng lặp; thời gian controller thử lại **phụ thuộc cấu hình**, đừng kết luận cơ chế hỏng vì một ngưỡng cố định |
| B3.3: bộ đếm `pod_security_evaluations_total` không tăng | `kubectl get --raw /metrics \| grep pod_security_evaluations_total \| head` | Bản dựng của bạn đặt tên nhãn khác cho `mode`/`decision`. Ghi nguyên các dòng metric vào evidence, và dùng hai bằng chứng còn lại của B3.3 — không cảnh báo, không bị chặn — làm kết luận. **Không bật audit log để đi vòng** |
| B4.1: `proc_val` trả chuỗi rỗng | `kubectl -n lab-9b-baseline exec tran -- head -30 /proc/1/status` | Pod chưa `Running`, hoặc tên trường viết sai hoa thường. Trường phân biệt hoa thường: `CapPrm`, `NoNewPrivs`, `Seccomp` |
| B4.3: Pod `nonroot-loi` lại `Running` | `kubectl -n lab-9b get pod nonroot-loi -o jsonpath='{.spec.securityContext}'` | Manifest bị apply đè bởi phiên bản cũ có `runAsUser`. Xóa Pod rồi apply lại đúng file của bước đó |
| B4.5: `$(( 0x$CAP_ADD ))` báo lỗi cú pháp | `echo "$CAP_ADD"` | Biến rỗng hoặc dính ký tự lạ. Chạy lại `proc_val`, và bảo đảm bạn đang ở **cùng một phiên shell** với B4.1 |
| B4.6: `Seccomp` của Pod `sec-rd` bằng `0` | `kubectl -n lab-9b-baseline get pod sec-rd -o jsonpath='{.spec.securityContext.seccompProfile}'` | Trường không được ghi vào Pod, thường do apply nhầm file. So lại object trong API với manifest; **không** đổi cấu hình kubelet để làm gate xanh |
| B4.6: `/proc/self/attr/current` rỗng hoặc báo lỗi | `ssh lab-k8s-worker2 'cat /sys/module/apparmor/parameters/enabled'` | Node chưa bật AppArmor. Ghi giá trị thật vào evidence và đọc lại B9.4 theo cấu hình đó; đây là tính chất của node, không phải lỗi của Pod |
| B5.2: giá trị trong Pod bằng giá trị trên node | `kubectl -n lab-9b-baseline get pod sysctl-antoan -o jsonpath='{.spec.securityContext.sysctls}'` | Trường `sysctls` không vào được Pod, hoặc bạn đọc nhầm node. Lấy node từ `.spec.nodeName` chứ đừng đoán |
| B5.3: Pod `sysctl-khong-antoan` kẹt `Pending` | `kubectl -n lab-9b describe pod sysctl-khong-antoan`; `kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'` | `lab-k8s-worker2` mang taint hoặc `NotReady`. Chạy lại gate mở đầu; nếu worker2 hỏng thật thì restore cả ba VM — **không** đổi `nodeSelector` sang worker1, vì quy ước lab ghim mọi Pod đặc quyền vào worker2 |
| B7.1: danh sách admission plugin rỗng | `kubectl get --raw /metrics \| head -5` | Bạn không có quyền đọc `/metrics`, hoặc chưa có admission nào chạy trong phiên này. Tạo một Pod bất kỳ rồi đọc lại; kiểm kubeconfig đang dùng có phải kubeconfig quản trị không |
| B7.4: không có header `X-Kubernetes-Pf-*` | `kubectl get ns -v=8 2>&1 \| grep -i 'response headers' -A 20` | APF bị tắt bằng `--enable-priority-and-fairness=false`, hoặc `-v=8` không in header ở bản `kubectl` của bạn. Đối chiếu version theo [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); **không** cài `kubectl` khác để đi vòng |
| B8.3: hai Pod tenant bị từ chối vì thiếu resources | `kubectl -n lab-9b-tenant get limitrange tenant-mac-dinh -o yaml` | LimitRange được apply **sau** khi Pod được tạo. Xóa Pod, bảo đảm LimitRange tồn tại trước, rồi apply lại — giá trị mặc định chỉ được chèn lúc tạo Pod |
| B8.4: đường cơ sở đã không thông | `kubectl -n lab-9b-tenant get pod tenant-web -o wide`; `kubectl -n lab-9b-tenant logs tenant-web` | `httpd` chưa lên, hoặc IP đọc trước khi Pod có IP. Đọc lại `WEB_IP` sau khi Pod `Ready`. Không áp NetworkPolicy khi đường cơ sở còn đỏ — mọi kết luận sau đó sẽ sai |
| B8.4: sau khi áp policy mà lưu lượng vẫn qua | `kubectl get pods -A -o wide \| grep -iE 'calico\|cilium'` | CNI đang chạy không thực thi NetworkPolicy — cluster không ở đúng `03-storage-ready`. Restore cả ba VM; xem [nợ #4](README.md#5-sổ-nợ-lab) |
| B8.4: DNS trong tenant không thông sau khi áp policy | `kubectl get ns kube-system --show-labels`; `kubectl -n kube-system get svc kube-dns` | Namespace `kube-system` thiếu nhãn `kubernetes.io/metadata.name`, hoặc DNS Service nằm ở namespace khác. Sửa `namespaceSelector` cho khớp nhãn thật; **không** gỡ luôn quy tắc egress |
| B9.5: cổng `10250` trả `000` trên cả ba node | `ssh lab-k8s-worker2 'sudo ss -ltn \| grep 10250'` | `curl` không tới được node do firewall của chính máy bạn, chứ chưa chắc kubelet đóng cổng. Chạy lại từ `lab-k8s-master`; `000` nghĩa là không kết nối được, khác hẳn `401` |
| B10.1: namespace kẹt `Terminating` | `kubectl get pods -n <ns>`; `kubectl describe namespace <ns>` | Pod còn trong grace period. Chờ; **không** cưỡng chế finalizer của Namespace |
| B10.2: số container đặc quyền lớn hơn con số ở B0.5 | `kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{" "}{range .spec.containers[*]}{.securityContext.privileged}{" "}{end}{"\n"}{end}' \| grep true` | Còn Pod đặc quyền của lab sót lại ngoài năm namespace, hoặc bạn đã tạo thêm ngoài kịch bản. Tìm theo output trên và xóa đúng cái đó. **Không** xóa Pod nào trong `kube-system` |
| B10.3: `diff` báo checksum khác | `diff -u ~/lab-evidence/9b/b0-cauhinh-sha.txt ~/lab-evidence/9b/b10-cauhinh-sha.txt` | Cấu hình control plane hoặc kubelet đã bị sửa trong lúc chạy lab. Cluster lệch baseline: tắt cả ba VM, restore cả ba về `03-storage-ready`, và **không** sang lab sau trước khi gate này PASS |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Pod Security Standards](https://v1-35.docs.kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes v1.35 — Pod Security Admission](https://v1-35.docs.kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Kubernetes v1.35 — Enforce Pod Security Standards with Namespace Labels](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)
- [Kubernetes v1.35 — Enforce Pod Security Standards by Configuring the Built-in Admission Controller](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/)
- [Kubernetes v1.35 — Migrate from PodSecurityPolicy to the Built-In PodSecurity Admission Controller](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/migrate-from-psp/)
- [Kubernetes v1.35 — Pod Security Policy (removed feature)](https://v1-35.docs.kubernetes.io/docs/concepts/security/pod-security-policy/)
- [Kubernetes v1.35 — Configure a Security Context for a Pod or Container](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/security-context/)
- [Kubernetes v1.35 — Linux kernel security constraints for Pods and containers](https://v1-35.docs.kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/)
- [Kubernetes v1.35 — Security for Linux nodes](https://v1-35.docs.kubernetes.io/docs/concepts/security/linux-security/)
- [Kubernetes v1.35 — Using sysctls in a Kubernetes Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/)
- [Kubernetes v1.35 — Multi-tenancy](https://v1-35.docs.kubernetes.io/docs/concepts/security/multi-tenancy/)
- [Kubernetes v1.35 — Good practices for Kubernetes Secrets](https://v1-35.docs.kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [Kubernetes v1.35 — Hardening Guide: Authentication Mechanisms](https://v1-35.docs.kubernetes.io/docs/concepts/security/hardening-guide/authentication-mechanisms/)
- [Kubernetes v1.35 — Kubernetes API Server Bypass Risks](https://v1-35.docs.kubernetes.io/docs/concepts/security/api-server-bypass-risks/)
- [Kubernetes v1.35 — Security Checklist](https://v1-35.docs.kubernetes.io/docs/concepts/security/security-checklist/)
- [Kubernetes v1.35 — Application Security Checklist](https://v1-35.docs.kubernetes.io/docs/concepts/security/application-security-checklist/)
- [Kubernetes v1.35 — API Priority and Fairness](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/flow-control/)
- [Kubernetes v1.35 — Admission Webhook Good Practices](https://v1-35.docs.kubernetes.io/docs/concepts/cluster-administration/admission-webhooks-good-practices/)
- [Kubernetes v1.35 — Cloud Controller Manager Administration](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/running-cloud-controller/)
- [Kubernetes v1.35 — Admission Controllers Reference](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Kubernetes v1.35 — Network Policies](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes v1.35 — Resource Quotas](https://v1-35.docs.kubernetes.io/docs/concepts/policy/resource-quotas/)
