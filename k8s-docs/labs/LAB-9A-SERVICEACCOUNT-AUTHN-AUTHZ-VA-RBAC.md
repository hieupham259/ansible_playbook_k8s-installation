# Lab 9a — ServiceAccount, authn/authz và RBAC

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do Lab 6a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** cleanup trả cluster về đúng `03-storage-ready`, **không tạo snapshot mới**.
> Lab này không cài thêm bất kỳ thành phần hạ tầng nào và **không sửa cấu hình kube-apiserver**.
> **Lab trước:** Lab 8c — HA với external etcd (chưa viết, xem
> [bản đồ lab](README.md#4-bản-đồ-lab)) chạy trên **bộ VM riêng** và không đụng tới chuỗi chính,
> nên cluster vào lab này vẫn phải ở đúng `03-storage-ready`: có StorageClass mặc định, không
> workload, không object nào của lab trước.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng **phần truy cập** của mục
[Giai đoạn 9 — Bảo mật và multi-tenancy](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy):
năm bài khái niệm [113](../113-security-vi.md), [114](../114-cloud-native-security-vi.md),
[118](../118-service-accounts-vi.md), [119](../119-controlling-access-vi.md),
[120](../120-rbac-good-practices-vi.md), cộng ba bài thực hành
[279](../279-configure-service-account-vi.md), [338](../338-access-api-from-pod-vi.md),
[359](../359-access-cluster-vi.md).

**Phần policy của giai đoạn 9 không thuộc lab này.** Pod Security Standards, Pod Security
Admission, security context, seccomp/AppArmor, admission webhook và checklist hardening nằm ở
Lab 9b — Pod Security và hardening (chưa viết, xem [bản đồ lab](README.md#4-bản-đồ-lab)), cùng
với bốn bài thực hành [282](../282-enforce-standards-admission-controller-vi.md),
[283](../283-enforce-standards-namespace-labels-vi.md), [286](../286-migrate-from-psp-vi.md),
[291](../291-security-context-vi.md). Lab 9a dừng đúng ở ranh giới đó: nó đi hết ba chặng của
bài [119](../119-controlling-access-vi.md) nhưng **không** cấu hình bất kỳ admission controller
nào — chặng ba chỉ được kiểm chứng bằng các plugin đã bật sẵn trên baseline.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không
chép lại con số phiên bản nào** và **không cài thêm gì**. Thành phần ngoài baseline đang chạy —
CNI thay Flannel, ingress controller, dynamic provisioner — đã khóa ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) và do Lab 5b, Lab 6a
cài; lab 9a chỉ **đọc** chúng, không đụng vào.

Topology vẫn là một control plane `lab-k8s-master` mang taint `NoSchedule` và hai worker
`lab-k8s-worker1`, `lab-k8s-worker2`. Lab dùng Pod, ConfigMap, Secret, Deployment và
`nodeSelector` của các giai đoạn đã học làm công cụ. **Không** dùng Pod Security Admission và
security context (Lab 9b), metrics-server và HPA
([giai đoạn 11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)), DRA
([giai đoạn 13](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao)), CRD và
admission webhook ([giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng)).

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm ba lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Ba lenh rieng cua lab 9a: dung diem bat dau, va ban co phan quyen chua co gi cua lab.
kubectl get storageclass | grep -q '(default)' \
  && echo 'PASS: co StorageClass mac dinh — dung diem bat dau 03-storage-ready'
test -z "$(kubectl get namespace -o name | grep 'namespace/lab-9a' || true)" \
  && echo 'PASS: chua co namespace nao ten lab-9a'
test -z "$(kubectl get clusterrole,clusterrolebinding -l lab=9a -o name 2>/dev/null)" \
  && echo 'PASS: chua co ClusterRole hay ClusterRoleBinding nao mang label lab=9a'
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; **ba dòng `PASS:` riêng của lab đều
xuất hiện**. Nếu dòng cuối fail, cluster còn sót object phạm vi cluster của một lần chạy trước —
dọn theo B7.1 rồi chạy lại gate này trước khi đi tiếp.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- **Ba chặng của bài [119](../119-controlling-access-vi.md), đúng thứ tự**: chỉ ra được một
  request cụ thể chết ở **chặng nào**, bằng bằng chứng lấy từ chính API server — mã `401` cho
  xác thực, `403` cho phân quyền, và một câu thông báo hoàn toàn khác cho kiểm soát tiếp nhận.
- Ranh giới của chặng ba: chứng minh được admission **không** chạm vào request chỉ đọc, và
  ngược lại, chứng minh được admission **sửa** object trước khi nó được ghi — thứ mà cả hai
  chặng trước không làm được.
- **ServiceAccount là danh tính của Pod** ([118](../118-service-accounts-vi.md)): chỉ ra được
  token nằm ở đâu trong Pod, nó tới đó bằng cơ chế nào, và chứng minh được `sub` của token
  chính là danh tính mà API server dùng để quyết định.
- Pod chạy bằng ServiceAccount `default` **không làm được gì ngoài quyền tối thiểu**: chứng minh
  bằng chính Pod đó — gọi được endpoint discovery nhưng bị `403` khi đọc ConfigMap trong chính
  namespace của nó.
- Phân biệt **bốn** thứ của bài [120](../120-rbac-good-practices-vi.md) bằng thực nghiệm chứ
  không bằng định nghĩa: Role, ClusterRole, RoleBinding, ClusterRoleBinding — kể cả tổ hợp
  **không tồn tại**, và kể cả trường hợp ClusterRole bị RoleBinding **giới hạn lại** vào đúng
  một namespace.
- Nguyên tắc quyền tối thiểu, đo được: một ServiceAccount chỉ đọc ConfigMap trong đúng một
  namespace — chứng minh nó **không** đọc được ở namespace khác, **không** làm được verb khác,
  và **không** đụng được vào Secret.
- `kubectl auth can-i` với `--as`, `--as-group`, `--list`, và quan hệ của nó với object
  SubjectAccessReview mà bài [119](../119-controlling-access-vi.md) in ra dưới dạng JSON.
- Vì sao RBAC gắn được vào một user hay group **không hề tồn tại** trong API server, và vì sao
  điều đó không mâu thuẫn với việc Kubernetes không có object `User`.
- Cơ chế RBAC tự chặn leo thang đặc quyền: một chủ thể được phép tạo Role vẫn **không** tạo nổi
  một Role rộng hơn quyền nó đang có.
- **Client bên ngoài xác thực thế nào** ([359](../359-access-cluster-vi.md)): dựng được
  kubeconfig cho một ServiceAccount, chạy được `kubectl proxy`, và giải thích được vì sao quyền
  của proxy chính là quyền của kubeconfig mà nó dùng.
- Vì sao token dài hạn dạng Secret bị khuyến cáo tránh: chứng minh bằng chính payload của hai
  loại token đặt cạnh nhau.
- Cleanup đúng phạm vi: xóa hết **object phạm vi cluster** lab tạo ra, chứng minh **cấu hình
  control plane không hề bị sửa**, chứng minh **không token nào lọt vào thư mục bằng chứng**, và
  đưa cluster về đúng `03-storage-ready`.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong phần truy cập của giai đoạn 9 | Kiểm chứng ở |
| --- | --- |
| [113 — Bảo mật](../113-security-vi.md) | B1.2 — bài là trang mục lục năm nhóm cơ chế; lab đối chiếu từng nhóm với thứ **đọc được từ cluster này**: kiểm soát truy cập API (toàn bộ lab), TLS khi truyền (B1.1), Secret chỉ là "bảo vệ cơ bản" (B3.4), admission controller (B4), kiểm toán (chưa bật — ghi ở bảng lý do) |
| [114 — Bảo mật Cloud Native](../114-cloud-native-security-vi.md) | B1.3 — bốn giai đoạn vòng đời đối chiếu với những gì các lab trước đã làm; ba lĩnh vực của giai đoạn *Runtime* và chứng minh lĩnh vực **truy cập** là lab này: mọi Pod đang chạy đều mang một danh tính ServiceAccount, không Pod nào không có |
| [118 — Tài khoản dịch vụ](../118-service-accounts-vi.md) | B2.4 — token của ServiceAccount là JWT ký, `sub` là danh tính; B2.5 — claim audience; B2.6 — bound token chết theo object nó gắn vào; B5.1 — `default` và trần quyền discovery; B5.2 — token được chiếu vào Pod ở đâu; B5.4 — `automountServiceAccountToken` |
| [119 — Kiểm soát truy cập vào Kubernetes API](../119-controlling-access-vi.md) | B1.1 — bốn bước và tầng TLS; B2 — chặng 1 và mã `401`; B3 — chặng 2, mã `403` và SubjectAccessReview; B4 — chặng 3 và bước 4 validate rồi ghi |
| [120 — Thực hành tốt về RBAC](../120-rbac-good-practices-vi.md) | B3.2–B3.9 — bốn tổ hợp Role/ClusterRole nhân RoleBinding/ClusterRoleBinding và quyền tối thiểu; B3.4 — `list` trên Secret lộ nội dung; B3.10 — verb `escalate` và cơ chế chặn leo thang; B3.11 — rà soát binding mặc định, `cluster-admin` và `system:unauthenticated`; B6.4 — vì sao `get` trên `nodes/proxy` không phải quyền chỉ đọc |
| [279 — Cấu hình Service Account cho Pod](../279-configure-service-account-vi.md) | B2.4 — `kubectl create token`; B4.3 — plugin ServiceAccount chèn `serviceAccountName`, projected volume và `imagePullSecrets`; B5.4 — từ chối tự động mount ở hai cấp và thứ tự ưu tiên; B5.5 — chiếu token với audience riêng; B6.5 — token dài hạn dạng Secret; B6.6 — issuer discovery |
| [338 — Truy cập Kubernetes API từ một Pod](../338-access-api-from-pod-vi.md) | B5.1 — Pod gọi API bằng token của mình, không proxy; B5.2 — ba file và hai biến môi trường mà bài liệt kê, đối chiếu giá trị với cluster; B5.3 — cùng một Pod, hai namespace, hai kết quả |
| [359 — Truy cập cluster](../359-access-cluster-vi.md) | B2.1 — `kubectl config view` và client certificate; B6.1 — kubeconfig cho một ServiceAccount; B6.2, B6.3 — `kubectl proxy` và quyền của nó; B6.4 — bốn loại proxy, cái nào thật sự có trên cluster này; B6.5 — nhánh *Không dùng kubectl proxy* của bài |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [113](../113-security-vi.md): mã hóa khi lưu trữ, cấu hình audit logging | Phải sửa cờ của `kube-apiserver` và thêm file policy. Lab này **cấm sửa cấu hình control plane** — B0.3 và B7.3 biến điều đó thành gate. Nội dung thuộc [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu); mã hóa Secret at rest là **nợ #6** trong [sổ nợ lab](README.md#5-sổ-nợ-lab) |
| Bài [113](../113-security-vi.md): bảng bảo mật của nhà cung cấp cloud, `ValidatingAdmissionPolicy`, các hiện thực policy của hệ sinh thái | Cluster lab chạy trên VM tự dựng, không có IaaS. `ValidatingAdmissionPolicy` và policy engine bên thứ ba cần điểm mở rộng API — [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) |
| Bài [113](../113-security-vi.md), [114](../114-cloud-native-security-vi.md): Pod Security Standards, RuntimeClass, module bảo mật Linux (AppArmor, seccomp) | Thuộc **phần policy** của giai đoạn 9, tức Lab 9b. Lab 9a không được cài trước hạ tầng của lab sau, xem [nguyên tắc không nhảy cóc](README.md#2-nguyên-tắc-không-nhảy-cóc) |
| Bài [114](../114-cloud-native-security-vi.md): giai đoạn vòng đời *Phát triển* và *Phân phối* | Thuộc quy trình phát triển và chuỗi cung ứng, không có thao tác nào trên cluster để kiểm chứng. Chính bài dịch xếp hai giai đoạn này vào *Đọc lướt*; checklist tương ứng là bài [130](../130-application-security-checklist-vi.md) ở Lab 9b |
| Bài [114](../114-cloud-native-security-vi.md): *Bảo vệ runtime — tính toán* và *lưu trữ*, service mesh, khởi động đo lường bằng mật mã | Hai lĩnh vực tính toán và lưu trữ đã thực hành ở [Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md), [Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md), [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md), [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md) và [Lab 6b](LAB-6B-SNAPSHOT-VA-VOLUME-NANG-CAO.md); B1.3 chỉ đối chiếu lại, không làm lại. Service mesh và measured boot nằm ngoài lộ trình |
| Bài [118](../118-service-accounts-vi.md): *Giới hạn audience trên node* (`ServiceAccountNodeAudienceRestriction`) | Là hardening cho danh tính kubelet và cần quy tắc RBAC cho chính kubelet. Bài dịch hoãn tới [128](../128-api-server-bypass-risks-vi.md), tức Lab 9b |
| Bài [118](../118-service-accounts-vi.md): annotation `kubernetes.io/enforce-mountable-secrets` | Chính bài ghi **deprecated** và khuyên dùng namespace riêng thay thế. Lab không dạy cơ chế đã bị khai tử; phần thay thế nằm ở bài [121](../121-secrets-good-practices-vi.md) (Lab 9b) |
| Bài [118](../118-service-accounts-vi.md): TokenReview API, xác minh OIDC trong mã của bạn, SPIFFE, service mesh, IAM của cloud | Dành cho người **viết dịch vụ**, không phải quản trị viên cluster. So sánh các cơ chế xác thực nằm ở bài [123](../123-hardening-authentication-vi.md), thuộc Lab 9b |
| Bài [119](../119-controlling-access-vi.md): policy ABAC, module phân quyền Webhook, `--secure-port` / `--bind-address` / `--tls-cert-file` | Cluster kubeadm chạy RBAC; bật thêm module phân quyền hoặc đổi cờ TLS là **sửa cấu hình kube-apiserver**. Thuộc [giai đoạn 8](../00-ALO-TRINH-ADMIN.md#giai-đoạn-8--dựng-cluster-bằng-kubeadm) bài [03](../03-control-plane-flags-vi.md) và [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). B3.1 vẫn dựng đúng object SubjectAccessReview mà bài in ra, chỉ khác là chạy trên module RBAC |
| Bài [119](../119-controlling-access-vi.md): bật/tắt từng admission controller, admission webhook | Cần cờ `--enable-admission-plugins` và cấu hình webhook. Lab chỉ kiểm chứng các plugin **đã bật sẵn** trên baseline. Webhook nằm ở bài [173](../173-admission-webhooks-vi.md), Lab 9b |
| Bài [120](../120-rbac-good-practices-vi.md): thêm người dùng vào `system:masters`, cấp `cluster-admin`, verb `bind` và `impersonate`, quyền phê duyệt CSR, quyền tạo PersistentVolume | Đây là danh sách những thứ **không được làm**. Lab chứng minh chúng nguy hiểm bằng cách **đọc** binding có sẵn (B3.11) và bằng cách chứng minh ServiceAccount của lab không có chúng (B3.10), chứ không tạo ra một chủ thể nguy hiểm rồi thử. Xoay CA và CSR là **nợ #7**, trả ở [giai đoạn 18](../00-ALO-TRINH-ADMIN.md#giai-đoạn-18--vòng-đời-chứng-chỉ) |
| Bài [120](../120-rbac-good-practices-vi.md): *Kiểm soát các admission webhook*, *Sửa đổi Namespace* (patch label), hạn ngạch số lượng đối tượng | Webhook thuộc bài [173](../173-admission-webhooks-vi.md); rủi ro patch label namespace chỉ có nghĩa khi đã bật Pod Security Admission, tức bài [116](../116-pod-security-admission-vi.md) — cả hai ở Lab 9b. Hạn ngạch số lượng đối tượng đã làm ở [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) |
| Bài [279](../279-configure-service-account-vi.md): `imagePullSecrets` kéo image thật từ private registry | Cần một registry riêng và mạng ra ngoài. B4.3 kiểm chứng đúng phần thuộc chặng admission: Secret được **chèn** vào `.spec.imagePullSecrets` của Pod mới. Việc kéo image từ registry riêng nằm ngoài baseline |
| Bài [279](../279-configure-service-account-vi.md): `--bound-object-kind Node`, ghi đè `--service-account-jwks-uri` | Token gắn Node là công cụ của thành phần chạy trên node; ghi đè JWKS URI là cờ của `kube-apiserver`. B2.6 kiểm chứng cơ chế bound token bằng object mà lab tạo được, B6.6 chỉ **đọc** tài liệu discovery |
| Bài [338](../338-access-api-from-pod-vi.md): thư viện client Go/Python, `kubectl proxy` chạy dạng sidecar | Cần cài SDK hoặc một image có `kubectl`. Baseline chỉ khóa `busybox:1.37`, và lab **không cài thêm phần mềm**. B5 đi đúng nhánh *Không sử dụng proxy* mà chính bài mô tả bằng shell; nhánh proxy được kiểm chứng ở B6.2 trên `lab-k8s-master` |
| Bài [359](../359-access-cluster-vi.md): Proxy hoặc Load-balancer đặt trước apiserver, Cloud Load Balancer | Cluster lab có **một** control plane và không có nhà cung cấp cloud, nên hai loại proxy này không tồn tại để quan sát. Load balancer trước apiserver là nội dung của lab 8b và 8c trên bộ VM riêng |

### 1.2. Thời lượng

3–4 giờ, tính từ lúc gate mở đầu đã PASS. Lab không cài gì, nhưng số bước rất nhiều và mỗi bước
đều yêu cầu **đọc kỹ một câu thông báo** rồi đối chiếu với bài. Phần lớn thời gian nằm ở B3 và
B5. Các bước phải chờ control plane hội tụ đều viết dưới dạng vòng lặp có điều kiện thoát, không
phải con số cố định.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi
  rõ node khác**. Lệnh cần `sudo` để đọc file trên node chạy trên chính node đó.
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Gần như mọi gate so sánh với biến và các
  hàm trợ giúp đặt ở B0 (`SRV`, `CACERT`, `jwt_pay`, `jwt_claim`, `api_code`, `api_body`, các
  biến đếm `*_N0`); mở shell mới giữa chừng là mất hết và các gate sau sẽ sai.

**An toàn bí mật — quy ước bắt buộc của lab này:**

- **Không bao giờ ghi token thô vào `~/lab-evidence/9a/`.** Bằng chứng chỉ chứa: mã HTTP, câu
  thông báo của API server, output của `kubectl auth whoami`, `kubectl auth can-i`, và **các
  claim đã giải mã** của JWT (`sub`, `aud`, `iss`) — không bao giờ chứa chuỗi token. B7.4 là
  gate tự động kiểm điều này.
- Token chỉ tồn tại trong **biến shell** và trong file dưới `~/lab-work/9a/` với quyền `600`.
  Thư mục `~/lab-work/9a/` bị xóa ở B7.1; các biến bị `unset` ở cùng bước.
- Luôn truyền token **qua biến**, không gõ giá trị token vào dòng lệnh — nếu không, nó nằm lại
  trong `~/.bash_history`. Vẫn phải nhớ rằng `-H "Authorization: Bearer $TOK"` đặt giá trị vào
  `argv` của tiến trình `curl`; trên máy nhiều người dùng thì phải dùng file cấu hình của `curl`
  thay vì tham số. Lab chấp nhận đánh đổi này vì cluster lab là máy một người dùng.
- Nơi duy nhất lab **giải mã** token là payload JWT — phần công khai của token, không phải chữ
  ký. Không bước nào in cả chuỗi token ra màn hình.

**An toàn quyền hạn — quy ước bắt buộc của lab này:**

- Lab tạo **object phạm vi cluster**. Cấp quyền quá rộng ở phạm vi này là loại lỗi không tự lộ
  ra: không Pod nào crash, không gate nào đỏ, nhưng một ServiceAccount trong một namespace bất
  kỳ bỗng đọc được toàn cluster. Vì vậy lab khai báo trước, đúng và đủ, những gì nó tạo:

  | Object phạm vi cluster | Nội dung | Tạo ở | Xóa ở |
  | --- | --- | --- | --- |
  | ClusterRole `lab-9a-doc-node` | `get`, `list` trên `nodes` — **không** có `nodes/proxy`, không verb ghi | B3.6 | B7.1 |
  | ClusterRoleBinding `lab-9a-doc-node` | gắn ClusterRole trên cho đúng **một** ServiceAccount | B3.6 | B7.1 |

  Mọi object phạm vi cluster của lab mang label `lab=9a`. B7.1 xóa theo label, B7.2 gate rằng
  không còn cái nào và **số lượng ClusterRole/ClusterRoleBinding của cluster trở về đúng con số
  đọc ở B0**.
- **Tuyệt đối không cấp `cluster-admin`** cho bất kỳ chủ thể nào lab tạo ra, và **không thêm ai
  vào nhóm `system:masters`**. Bài [120](../120-rbac-good-practices-vi.md) nói rõ quyền của
  `system:masters` **không thu hồi được** bằng cách xóa binding — nghĩa là một sai lầm ở đó
  không dọn được bằng cleanup, chỉ dọn được bằng restore snapshot. B3.11 gate rằng không object
  nào của lab dính vào hai thứ đó.
- Không ClusterRole nào của lab chứa wildcard, `secrets`, `escalate`, `bind`, `impersonate`,
  `nodes/proxy`, hay quyền tạo PersistentVolume — đúng danh sách rủi ro leo thang đặc quyền mà
  bài [120](../120-rbac-good-practices-vi.md) liệt kê.
- Chỉ dùng `--as` và `--as-group` của `kubectl` để **kiểm chứng**. Đó là quyền mạo danh, và bạn
  có nó vì kubeconfig quản trị có nó — không phải vì lab cấp thêm cho ai.

**Phạm vi và cách quay lui:**

- **Không cài thêm hạ tầng**, không tạo snapshot mới, **không sửa cấu hình `kube-apiserver`,
  kubelet hay manifest trong `/etc/kubernetes/manifests`.** B0.3 ghi checksum, B7.3 đối chiếu —
  đó là gate chứng minh bạn không đụng vào.
- Lab tạo đúng hai Namespace: `lab-9a` (namespace chính) và `lab-9a-khac` (namespace phụ, dùng
  để chứng minh ranh giới namespace). Cả hai bị xóa ở B7.1.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Lab này không có bước phá hoại; các Pod cần
  ghim node đều ghim vào `lab-k8s-worker2` và chỉ **đọc** từ bên trong container.
- Image dùng cho toàn bộ lab là `busybox:1.37` đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và đã có sẵn trên cả ba node
  từ Lab 00, nên lab không phụ thuộc mạng ra ngoài.
- Manifest tạm và file chứa token ghi vào `~/lab-work/9a/`; bằng chứng ghi vào
  `~/lab-evidence/9a/`.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.

**Cách quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `03-storage-ready` — không bao
giờ restore riêng một VM, xem ghi chú cuối
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) — rồi bật lại theo
thứ tự master → worker 1 → worker 2, chạy lại gate mở đầu và bắt đầu lại từ B0.

---

# Phần B — Thực hành kiến thức 9a

## B0. Chuẩn bị workspace, hai namespace và ảnh chụp "trước"

**Mục đích:** dựng chỗ làm việc, tạo hai namespace, định nghĩa các hàm trợ giúp mà toàn bộ lab
dùng để **so giá trị** thay vì so chuỗi, và chụp lại trạng thái "trước" của cả tồn kho RBAC lẫn
cấu hình control plane — hai thứ mà B7 sẽ đối chiếu để chứng minh lab không để lại gì.

### B0.1. Workspace, hai namespace và ba hàm trợ giúp

```bash
mkdir -p ~/lab-work/9a ~/lab-evidence/9a
chmod 700 ~/lab-work/9a
kubectl config current-context
kubectl get nodes -o wide
kubectl create namespace lab-9a
kubectl create namespace lab-9a-khac
```

Toàn bộ lab đọc claim của JWT bằng hai hàm dưới đây. Chúng chỉ giải mã **payload** — phần công
khai của token — và không bao giờ in cả chuỗi token:

```bash
jwt_pay() {
  P="$(printf '%s' "$1" | cut -d. -f2)"
  case $(( ${#P} % 4 )) in 2) P="${P}==" ;; 3) P="${P}=" ;; esac
  printf '%s' "$P" | tr -- '-_' '+/' | base64 -d 2>/dev/null
}
jwt_claim() { jwt_pay "$1" | sed -n "s/.*\"$2\":\"\([^\"]*\)\".*/\1/p"; }
jwt_aud()   { jwt_pay "$1" | sed -n 's/.*"aud":\(\[[^]]*\]\).*/\1/p'; }
```

Kiểm chứng ba hàm bằng một chuỗi cố định, **không phải token thật**, để biết chúng chạy đúng
trước khi đem đi làm gate:

```bash
FAKE='aaa.eyJzdWIiOiJsYWItOWEtdGVzdCIsImF1ZCI6WyJsYWItOWEtYXVkIl19.bbb'
echo "sub=$(jwt_claim "$FAKE" sub) | aud=$(jwt_aud "$FAKE")"

test "$(jwt_claim "$FAKE" sub)" = 'lab-9a-test' \
  && test "$(jwt_aud "$FAKE")" = '["lab-9a-aud"]' \
  && echo 'PASS: ba ham giai ma claim cua JWT hoat dong dung'

P1="$(kubectl get namespace lab-9a      -o jsonpath='{.status.phase}')"
P2="$(kubectl get namespace lab-9a-khac -o jsonpath='{.status.phase}')"
test "$P1" = 'Active' && test "$P2" = 'Active' \
  && echo 'PASS: hai namespace cua lab da Active'
```

**Ý nghĩa:** `lab-9a` là namespace chính; `lab-9a-khac` là **đối chứng** — cùng một chủ thể,
cùng một verb, chạy ở đây thì được, chạy ở kia thì không. Bài
[120](../120-rbac-good-practices-vi.md) nói namespace là đơn vị tách các mức tin cậy, và hai
namespace cạnh nhau là cách chứng minh câu đó thay vì tin lời.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B0.2. Địa chỉ API server, CA, và hai hàm gọi API thô

Bài [359](../359-access-cluster-vi.md) nói kubectl **tự lo** việc định vị và xác thực với
apiserver. Cả lab này thì cố tình bỏ kubectl ra khỏi đường đi ở những chỗ cần đọc **mã HTTP**,
vì mã HTTP mới là thứ phân biệt chặng 1 với chặng 2. Hai giá trị dưới đây đọc từ chính
kubeconfig, không viết cứng:

```bash
SRV="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
CACERT=/etc/kubernetes/pki/ca.crt
echo "server = $SRV | ca = $CACERT"

api_code() {   # $1 = duong dan; $2 = token, de rong thi khong gui header nao
  if [ -n "$2" ]; then
    curl -s -o /dev/null -w '%{http_code}' --cacert "$CACERT" \
      -H "Authorization: Bearer $2" "${SRV}$1"
  else
    curl -s -o /dev/null -w '%{http_code}' --cacert "$CACERT" "${SRV}$1"
  fi
}
api_body() {
  if [ -n "$2" ]; then
    curl -s --cacert "$CACERT" -H "Authorization: Bearer $2" "${SRV}$1"
  else
    curl -s --cacert "$CACERT" "${SRV}$1"
  fi
}
```

```bash
test -s "$CACERT" && echo 'PASS: doc duoc certificate CA cua cluster'
case "$SRV" in https://*) echo 'PASS: kubeconfig tro toi mot endpoint https' ;; esac

curl -s -o /dev/null --cacert "$CACERT" "${SRV}/version"; RC_TLS=$?
echo "curl exit code khi co CA = $RC_TLS"
test "$RC_TLS" -eq 0 \
  && echo 'PASS: bat tay TLS thanh cong — certificate cua API server duoc chinh CA nay ky'
```

**Ý nghĩa:** đây chính là câu của bài [119](../119-controlling-access-vi.md): nếu cluster dùng
CA riêng thì client cần **một bản sao certificate CA đó** để tin được kết nối và chắc rằng nó
không bị chặn bắt giữa đường. `curl` trả về 0 nghĩa là chuỗi tin cậy khớp — chưa nói gì về việc
bạn là ai, vì xác thực còn chưa bắt đầu.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B0.3. Tồn kho "trước" — RBAC và cấu hình control plane

Lab tạo object phạm vi cluster, nên phải biết chính xác cluster đang có bao nhiêu cái **trước**
khi bắt đầu. B7.2 sẽ so lại đúng những con số này:

```bash
CR_N0="$(kubectl get clusterrole --no-headers | wc -l)"
CRB_N0="$(kubectl get clusterrolebinding --no-headers | wc -l)"
SA_N0="$(kubectl get serviceaccount -A --no-headers | wc -l)"
NS_N0="$(kubectl get namespace --no-headers | wc -l)"

{
  echo "=== $(date -Is) — ton kho RBAC truoc Lab 9a ==="
  echo "clusterrole=$CR_N0 clusterrolebinding=$CRB_N0 serviceaccount=$SA_N0 namespace=$NS_N0"
  echo '--- clusterrole/clusterrolebinding mang label lab=9a ---'
  kubectl get clusterrole,clusterrolebinding -l lab=9a 2>&1
} | tee ~/lab-evidence/9a/b0-truoc.txt

test "$CR_N0" -gt 0 && test "$CRB_N0" -gt 0 \
  && echo 'PASS: doc duoc so luong ClusterRole va ClusterRoleBinding hien co'
test -z "$(kubectl get clusterrole,clusterrolebinding -l lab=9a -o name 2>/dev/null)" \
  && echo 'PASS: chua object pham vi cluster nao mang label lab=9a'
```

Và checksum của cấu hình control plane cùng cấu hình kubelet — thứ B7.3 sẽ so lại:

```bash
{
  echo "apiserver-manifest $(sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml \
    | awk '{print $1}')"
  echo "kubelet-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "kubelet-$n $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
} | tee ~/lab-evidence/9a/b0-cauhinh-sha.txt

test "$(wc -l < ~/lab-evidence/9a/b0-cauhinh-sha.txt)" -eq 4 \
  && test "$(awk '{print $2}' ~/lab-evidence/9a/b0-cauhinh-sha.txt \
       | grep -c '^[0-9a-f]\{64\}$')" -eq 4 \
  && echo 'PASS: ghi duoc checksum cua manifest apiserver va cau hinh kubelet ba node'
```

**Ý nghĩa:** cám dỗ lớn nhất của một lab về authn/authz là "thêm một cờ vào apiserver để xem
module kia chạy thế nào". Bốn dòng checksum này biến lời hứa "chỉ đọc" thành thứ kiểm chứng
được, đúng như [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) đã làm với cấu hình kubelet.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B1. Bản đồ: bốn bước của một request, và những gì cluster này tự khai ra

**Mục đích:** dựng khung trước khi đi vào từng chặng. Bài [119](../119-controlling-access-vi.md)
vẽ **bốn bước**; bài [113](../113-security-vi.md) là mục lục **năm nhóm** cơ chế; bài
[114](../114-cloud-native-security-vi.md) chia **bốn giai đoạn** vòng đời. B1 đối chiếu cả ba
khung đó với cluster thật, chỉ bằng thao tác đọc.

### B1.1. Tầng 0 — bảo mật tầng truyền tải

Trước cả bước 1 là TLS. Ba phép thử dưới đây đều **không** mang credential nào, nên chúng chỉ
nói về tầng truyền tải:

```bash
API_HOSTPORT="${SRV#https://}"

# 1) Noi HTTP thuan toi cong TLS.
curl -s -o ~/lab-evidence/9a/b1-http-tho.txt -w 'http_code=%{http_code}\n' \
  "http://${API_HOSTPORT}/version" | tee ~/lab-evidence/9a/b1-http-code.txt
cat ~/lab-evidence/9a/b1-http-tho.txt

# 2) HTTPS nhung khong dua CA cho client.
curl -s -o /dev/null "${SRV}/version"; RC_NOCA=$?
echo "curl exit code khi KHONG co CA = $RC_NOCA"

# 3) HTTPS co CA — da lam o B0.2, nhac lai de doi chieu.
curl -s -o /dev/null --cacert "$CACERT" "${SRV}/version"; RC_CA=$?
echo "curl exit code khi CO CA = $RC_CA"
```

```bash
grep -qi 'HTTP request to an HTTPS server' ~/lab-evidence/9a/b1-http-tho.txt \
  && echo 'PASS: cong 6443 tu noi ro no chi phuc vu HTTPS'
test "$RC_NOCA" -ne 0 && test "$RC_CA" -eq 0 \
  && echo 'PASS: khong co CA thi client tu choi tin server; co CA thi bat tay xong'
```

**Ý nghĩa:** hai kết quả này ứng với hai câu của bài [119](../119-controlling-access-vi.md).
Thứ nhất: API server lắng nghe **được bảo vệ bằng TLS**, nên một client nói HTTP thuần bị chặn
ngay ở tầng dưới cùng, chưa hề chạm tới xác thực. Thứ hai: vì cluster dùng CA riêng, client
**phải có bản sao certificate CA** thì mới tin được kết nối. Và ở đúng giai đoạn này, client có
thể **xuất trình TLS client certificate** — đó là thứ kubeconfig quản trị của bạn đang làm, B2.1
sẽ mở ra xem.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B1.2. Năm nhóm cơ chế của bài 113 — cái nào đọc được từ cluster này

```bash
{
  echo "=== $(date -Is) — doi chieu nam nhom co che cua bai 113 ==="
  echo '--- 1. Kiem soat truy cap API: module RBAC co san sang khong ---'
  kubectl get --raw '/apis/rbac.authorization.k8s.io/v1' >/dev/null 2>&1 \
    && echo 'rbac.authorization.k8s.io/v1: co'
  echo "so luong ClusterRole dung san: $CR_N0"
  echo '--- 2. Ma hoa khi truyen: xem B1.1 ---'
  echo '--- 3. Secret API ---'
  kubectl api-resources --api-group='' --no-headers | awk '$1=="secrets"'
  echo '--- 4. Admission control: plugin duoc bat tren baseline ---'
  sudo grep -E 'admission-plugins' /etc/kubernetes/manifests/kube-apiserver.yaml || \
    echo '(khong co co --enable-admission-plugins tuong minh)'
  echo '--- 5. Kiem toan ---'
  sudo grep -cE 'audit-policy-file|audit-log-path' \
    /etc/kubernetes/manifests/kube-apiserver.yaml
} | tee ~/lab-evidence/9a/b1-nhom-co-che.txt
```

```bash
RBAC_OK="$(kubectl get --raw '/apis/rbac.authorization.k8s.io/v1' >/dev/null 2>&1 \
  && echo yes || echo no)"
SEC_OK="$(kubectl api-resources --api-group='' --no-headers | awk '$1=="secrets"' | wc -l)"
AUDIT_N="$(sudo grep -cE 'audit-policy-file|audit-log-path' \
  /etc/kubernetes/manifests/kube-apiserver.yaml || true)"
echo "rbac=$RBAC_OK secrets_api=$SEC_OK audit_flags=$AUDIT_N"

test "$RBAC_OK" = 'yes' && echo 'PASS: nhom 1 — API group RBAC dang phuc vu'
test "$SEC_OK" -eq 1   && echo 'PASS: nhom 3 — Secret API co mat'
test "$AUDIT_N" -eq 0  && echo 'PASS: nhom 5 — audit chua duoc bat tren baseline, dung du kien'
```

**Ý nghĩa:** bài [113](../113-security-vi.md) gọi kiểm soát quyền truy cập tới Kubernetes API là
**nhóm then chốt** — và đó chính là toàn bộ lab này. Ba nhóm còn lại được đối chiếu ở đúng mức
mà cluster cho phép: mã hóa khi truyền đã chứng minh ở B1.1, Secret sẽ được chứng minh ở B3.4 là
chỉ có **sự bảo vệ cơ bản**, còn admission sẽ chứng minh ở B4. Nhóm kiểm toán **chưa bật** —
đúng như bảng lý do ở [mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành) đã ghi; bật nó là việc
của giai đoạn 22.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B1.3. Bốn giai đoạn vòng đời của bài 114 — và vì sao lab này là lĩnh vực *truy cập*

Bài [114](../114-cloud-native-security-vi.md) chia vòng đời thành *Phát triển*, *Phân phối*,
*Triển khai*, *Runtime*; giai đoạn Runtime lại gồm ba lĩnh vực **truy cập**, **tính toán**,
**lưu trữ**. Bảng dưới đây là nơi lộ trình đã hoặc sẽ chạm vào từng ô:

| Giai đoạn / lĩnh vực | Ở đâu trong lộ trình |
| --- | --- |
| Phát triển, Phân phối | ngoài phạm vi thao tác trên cluster — xem bảng lý do ở [mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành) |
| Triển khai — namespace là cơ chế cô lập | B0.1 và toàn bộ B3 của lab này |
| Runtime — **truy cập** | **lab này**, từ B1 tới B6 |
| Runtime — tính toán | [Lab 3c](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md), [Lab 7a](LAB-7A-LAP-LICH-VA-EVICTION.md), [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md); phần Pod Security Standards ở Lab 9b |
| Runtime — lưu trữ | [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md), [Lab 6b](LAB-6B-SNAPSHOT-VA-VOLUME-NANG-CAO.md) |

Phần đọc được từ cluster: bài nói dùng **ServiceAccount để cung cấp và quản lý danh tính bảo mật
cho workload**. Nếu câu đó đúng thì trên cluster này **không Pod nào** đang chạy mà không có
danh tính:

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"|"}{.metadata.name}{"|"}{.spec.serviceAccountName}{"|"}{.metadata.annotations.kubernetes\.io/config\.mirror}{"\n"}{end}' \
  > ~/lab-evidence/9a/b1-pod-sa.txt
column -t -s'|' ~/lab-evidence/9a/b1-pod-sa.txt 2>/dev/null \
  || cat ~/lab-evidence/9a/b1-pod-sa.txt

POD_ALL="$(   grep -c '.'                 ~/lab-evidence/9a/b1-pod-sa.txt)"
POD_CO_SA="$( awk -F'|' '$3!=""'          ~/lab-evidence/9a/b1-pod-sa.txt | wc -l)"
POD_KHONG="$( awk -F'|' '$3==""'          ~/lab-evidence/9a/b1-pod-sa.txt | wc -l)"
POD_MIRROR="$(awk -F'|' '$3=="" && $4!=""' ~/lab-evidence/9a/b1-pod-sa.txt | wc -l)"
SA_ALL="$(kubectl get serviceaccount -A --no-headers | wc -l)"
echo "pod=$POD_ALL | co serviceAccountName=$POD_CO_SA | khong co=$POD_KHONG"
echo "trong so khong co, la mirror Pod cua static Pod: $POD_MIRROR"
echo "serviceaccount toan cluster=$SA_ALL"

test "$POD_CO_SA" -gt 0 && test "$POD_KHONG" -eq "$POD_MIRROR" \
  && echo 'PASS: moi Pod thuong deu mang danh tinh; chi mirror Pod la ngoai le'
test "$SA_ALL" -ge "$NS_N0" \
  && echo 'PASS: moi namespace deu co it nhat mot ServiceAccount'
```

**Ý nghĩa:** không Pod thường nào chạy "vô danh". Con số `SA_ALL` lớn hơn hoặc bằng số namespace
vì mỗi namespace được tạo ra đều **có sẵn** một ServiceAccount `default` — đúng câu của bài
[118](../118-service-accounts-vi.md), và hai namespace bạn vừa tạo ở B0.1 cũng vậy. Ai điền
`serviceAccountName` cho những Pod không khai gì thì B4.3 trả lời.

Ngoại lệ nằm ở **mirror Pod** — bản sao trong API của các static Pod control plane. Chúng do
kubelet đăng ký chứ không do bạn tạo, và cơ chế ở B4.3 cố ý không đụng vào chúng, nên chúng
không có `serviceAccountName`. Vì sao static Pod là một đường vòng đáng lo thì bài
[128](../128-api-server-bypass-risks-vi.md) trả lời, ở Lab 9b.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B2. Chặng 1 — Xác thực

**Mục đích:** chứng minh chặng 1 làm đúng một việc: biến một request thành một `username`, hoặc
từ chối bằng **401**. Và chứng minh nó **không** quan tâm bạn được phép làm gì — đó là việc của
chặng 2.

### B2.1. Bạn đang là ai, và điều đó tới từ đâu

```bash
kubectl auth whoami | tee ~/lab-evidence/9a/b2-whoami-admin.txt
kubectl config view --minify | tee ~/lab-evidence/9a/b2-kubeconfig.txt

ADMIN_USER="$(kubectl auth whoami -o jsonpath='{.status.userInfo.username}')"
ADMIN_GROUPS="$(kubectl auth whoami -o jsonpath='{.status.userInfo.groups}')"
echo "username = $ADMIN_USER"
echo "groups   = $ADMIN_GROUPS"
```

`kubectl config view` che phần nhạy cảm. Lấy đúng client certificate mà kubeconfig đang xuất
trình rồi đọc subject của nó:

```bash
kubectl config view --raw --minify \
  -o jsonpath='{.users[0].user.client-certificate-data}' \
  | base64 -d > ~/lab-work/9a/admin-client.crt
chmod 600 ~/lab-work/9a/admin-client.crt

openssl x509 -noout -subject -in ~/lab-work/9a/admin-client.crt \
  | tee ~/lab-evidence/9a/b2-client-cert-subject.txt

CN_CERT="$(openssl x509 -noout -subject -in ~/lab-work/9a/admin-client.crt \
  | sed -n 's/.*CN *= *\([^,\/]*\).*/\1/p' | sed 's/[[:space:]]*$//')"
echo "CN trong client certificate = $CN_CERT"
echo "username API server tra ve  = $ADMIN_USER"

test -n "$CN_CERT" && test "$CN_CERT" = "$ADMIN_USER" \
  && echo 'PASS: username o chang 1 duoc rut ra tu CN cua client certificate'
test -n "$ADMIN_GROUPS" \
  && echo 'PASS: authenticator con cung cap thong tin nhom cho chang sau'
```

**Ý nghĩa:** đúng ba câu của bài [119](../119-controlling-access-vi.md). Client xuất trình TLS
client certificate **ở giai đoạn TLS**; module xác thực bằng client certificate dùng nó ở
**bước 1**; kết quả là một `username`, và **một số authenticator còn cung cấp thông tin thành
viên nhóm**. Trường `O` trong subject chính là nhóm — hãy đọc lại
`b2-client-cert-subject.txt` và đối chiếu với dòng `groups`. Cả hai giá trị này là thứ chặng 2
sẽ dùng ở B3.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B2.2. Danh tính đó không phải một object

```bash
{
  echo '--- co resource type nao ten users hay groups khong ---'
  kubectl api-resources --no-headers | awk '$1=="users" || $1=="groups"'
  echo "(so dong ben tren la ket qua)"
  echo '--- thu get users ---'
  kubectl get users 2>&1 | head -3
} | tee ~/lab-evidence/9a/b2-khong-co-object-user.txt
```

```bash
U_RES="$(kubectl api-resources --no-headers | awk '$1=="users" || $1=="groups"' | wc -l)"
kubectl get users >/dev/null 2>&1; RC_U=$?
echo "resource type users/groups = $U_RES | exit code cua kubectl get users = $RC_U"

test "$U_RES" -eq 0 && echo 'PASS: khong co resource type nao ten users hay groups'
test "$RC_U" -ne 0  && echo 'PASS: kubectl khong liet ke duoc nguoi dung'
test "$SA_N0" -gt 0 && echo 'PASS: nguoc lai, ServiceAccount thi liet ke duoc — no la object that'
```

**Ý nghĩa:** bài [118](../118-service-accounts-vi.md) chốt điểm này bằng một bảng: ServiceAccount
nằm **trong** Kubernetes API, còn user hay group nằm **bên ngoài**. Bài
[119](../119-controlling-access-vi.md) nói cùng chuyện theo hướng khác: Kubernetes dùng username
để ra quyết định và ghi log, nhưng **không có đối tượng `User`** và không lưu username trong API.
Điều đó không cản RBAC gắn quyền vào một user hay group — B3.9 chứng minh.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B2.3. Ba cách trượt chặng 1 — và một cách không trượt

```bash
# 1) Bearer token khong phai token cua ai ca.
C_RAC="$(api_code /api 'day-khong-phai-mot-token')"

# 2) Khong gui credential nao, toi mot endpoint co du lieu.
C_AN_API="$(api_code /api '')"

# 3) Khong gui credential nao, toi endpoint thong tin cong khai.
C_AN_VER="$(api_code /version '')"

# Cluster nay co bat xac thuc an danh khong — doc tu manifest, khong doan.
if sudo grep -q 'anonymous-auth=false' /etc/kubernetes/manifests/kube-apiserver.yaml; then
  ANON=off; EXP_API=401; EXP_VER=401
else
  ANON=on;  EXP_API=403; EXP_VER=200
fi

{
  echo "anonymous-auth = $ANON"
  echo "token rac            -> $C_RAC (ky vong 401)"
  echo "khong credential /api    -> $C_AN_API (ky vong $EXP_API)"
  echo "khong credential /version -> $C_AN_VER (ky vong $EXP_VER)"
} | tee ~/lab-evidence/9a/b2-ma-http.txt
```

```bash
test "$C_RAC" = '401' \
  && echo 'PASS: token khong hop le bi chan o chang 1 voi ma 401'
test "$C_AN_API" = "$EXP_API" && test "$C_AN_VER" = "$EXP_VER" \
  && echo 'PASS: request khong credential cho ket qua dung voi cau hinh anonymous-auth doc duoc'
test "$C_AN_API" != '200' \
  && echo 'PASS: khong ai doc duoc du lieu API ma khong co danh tinh'
```

**Ý nghĩa:** đây là cái bẫy đáng nhớ nhất của chặng 1. Bài
[119](../119-controlling-access-vi.md) nói request **không xác thực được** thì bị từ chối với
**401**. Nhưng "không gửi credential" và "gửi credential sai" là **hai chuyện khác nhau**: token
rác làm mọi authenticator thất bại nên nhận `401`; còn khi `anonymous-auth` đang bật, request
trắng vẫn **qua được chặng 1** dưới danh tính `system:anonymous` rồi mới chết ở **chặng 2** với
`403`. Mã `403` cho một request không có credential nào là dấu hiệu bạn phải đi rà binding của
nhóm `system:unauthenticated` — đúng khuyến nghị gia cố của bài
[120](../120-rbac-good-practices-vi.md), làm ở B3.11.

Cũng vì thế `/version` trả `200` mà `/api` thì không: chúng là hai đường dẫn được phân quyền
khác nhau, không phải hai mức xác thực khác nhau.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B2.4. Một danh tính phi con người vượt qua chặng 1

Bài [118](../118-service-accounts-vi.md) mô tả ba bước dùng một service account: **tạo object**,
**cấp quyền**, **gán cho Pod**. B2 làm bước một; B3 làm bước hai; B5 làm bước ba.

```bash
kubectl create serviceaccount sa-doc-cm -n lab-9a
kubectl get serviceaccount sa-doc-cm -n lab-9a -o yaml \
  | tee ~/lab-evidence/9a/b2-sa-doc-cm.yaml

TOK_RO="$(kubectl create token sa-doc-cm -n lab-9a)"
```

Không in `TOK_RO` ra màn hình. Chỉ đọc các claim công khai của nó:

```bash
{
  echo "sub = $(jwt_claim "$TOK_RO" sub)"
  echo "iss = $(jwt_claim "$TOK_RO" iss)"
  echo "aud = $(jwt_aud "$TOK_RO")"
} | tee ~/lab-evidence/9a/b2-claim-token-sa.txt

SUB_RO="$(jwt_claim "$TOK_RO" sub)"
test "$SUB_RO" = 'system:serviceaccount:lab-9a:sa-doc-cm' \
  && echo 'PASS: claim sub cua token chinh la danh tinh cua ServiceAccount'
jwt_pay "$TOK_RO" | grep -q '"exp"' \
  && echo 'PASS: token co claim exp — day la token co thoi han'
```

Bây giờ dùng token đó gọi API server thật:

```bash
C_DISC="$(api_code /api "$TOK_RO")"
C_CM="$(  api_code /api/v1/namespaces/lab-9a/configmaps "$TOK_RO")"
api_body /api/v1/namespaces/lab-9a/configmaps "$TOK_RO" \
  | tee ~/lab-evidence/9a/b2-403-configmaps.txt
echo
echo "GET /api = $C_DISC | GET configmaps trong lab-9a = $C_CM"
```

```bash
test "$C_DISC" = '200' \
  && echo 'PASS: token duoc chap nhan o chang 1 — endpoint discovery tra ve 200'
test "$C_CM" = '403' \
  && echo 'PASS: cung token do bi chan o chang 2 voi ma 403'
grep -q 'system:serviceaccount:lab-9a:sa-doc-cm' ~/lab-evidence/9a/b2-403-configmaps.txt \
  && echo 'PASS: chinh API server goi ten danh tinh no da rut ra o chang 1'
```

**Ý nghĩa:** hai mã HTTP trên cùng một token là bằng chứng gọn nhất cho việc hai chặng là **hai
chặng**. Chặng 1 đã thành công — nếu không thì cả hai request đều `401`. Cái chết ở request thứ
hai là chặng 2, và câu thông báo nêu đúng tên danh tính mà chặng 1 vừa sinh ra. Đây cũng là bằng
chứng cho câu của bài [118](../118-service-accounts-vi.md): ServiceAccount mới tạo **không có
quyền hạn nào** ngoài các quyền API discovery mặc định.

**PASS:** năm dòng `PASS:` của bước này xuất hiện.

### B2.5. Chữ ký đúng, hạn còn, nhưng sai audience thì vẫn là 401

Bài [118](../118-service-accounts-vi.md) liệt kê năm việc API server làm với một bearer token:
kiểm chữ ký, kiểm hết hạn, kiểm tham chiếu object trong claim, kiểm token còn hiệu lực, và kiểm
**claim về audience**. Bốn cái đầu đều đạt ở token dưới đây; chỉ cái thứ năm sai:

```bash
TOK_AUD="$(kubectl create token sa-doc-cm -n lab-9a --audience=lab-9a-khong-ai-nghe)"

{
  echo "sub = $(jwt_claim "$TOK_AUD" sub)"
  echo "aud = $(jwt_aud "$TOK_AUD")"
} | tee ~/lab-evidence/9a/b2-claim-token-sai-aud.txt

C_AUD="$(api_code /api "$TOK_AUD")"
echo "GET /api voi token sai audience = $C_AUD"
```

```bash
test "$(jwt_claim "$TOK_AUD" sub)" = "$SUB_RO" \
  && echo 'PASS: van dung mot ServiceAccount, chi khac audience'
jwt_aud "$TOK_AUD" | grep -q 'lab-9a-khong-ai-nghe' \
  && echo 'PASS: token mang dung audience da yeu cau'
test "$C_AUD" = '401' \
  && echo 'PASS: sai audience thi chet o chang 1 — 401, khong phai 403'
test "$C_DISC" = '200' && test "$C_AUD" = '401' \
  && echo 'PASS: hai token cua cung mot SA, hai ket qua khac nhau chi vi audience'
```

**Ý nghĩa:** đây là chỗ dễ chẩn đoán sai nhất trong vận hành. Ứng dụng báo `401`, người ta đi
kiểm RBAC — nhưng RBAC nằm ở chặng 2 và request này chưa bao giờ tới đó. Audience là cách thu
hẹp phạm vi của token để nó **chỉ dùng được ở đúng nơi bạn định**, và một token phát cho hệ
thống khác thì với API server nó không phải token hợp lệ. B5.5 dựng lại đúng tình huống này bằng
projected volume trong một Pod.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B2.6. Token gắn với vòng đời của object — xóa object là token chết

```bash
kubectl create serviceaccount sa-tam -n lab-9a
TOK_TAM="$(kubectl create token sa-tam -n lab-9a)"

C_TAM_1="$(api_code /api "$TOK_TAM")"
echo "truoc khi xoa SA: GET /api = $C_TAM_1"
test "$C_TAM_1" = '200' && echo 'PASS: token cua sa-tam dang xac thuc duoc'
```

```bash
kubectl delete serviceaccount sa-tam -n lab-9a

i=0; C_TAM_2=""
while [ "$i" -lt 60 ]; do
  C_TAM_2="$(api_code /api "$TOK_TAM")"
  test "$C_TAM_2" = '401' && break
  i=$(( i + 1 )); sleep 1
done
echo "sau khi xoa SA: GET /api = $C_TAM_2 (sau $i vong lap)"

test "$C_TAM_2" = '401' \
  && echo 'PASS: token het hieu luc ngay khi object ma no gan vao bien mat'
unset TOK_TAM
```

**Ý nghĩa:** bài [118](../118-service-accounts-vi.md) gọi token do API `TokenRequest` phát hành
là **bound token** — nó gắn với vòng đời của client đang dùng ServiceAccount đó, và API server
còn kiểm rằng **tham chiếu object vẫn tồn tại**, so khớp bằng unique ID. Xóa ServiceAccount là
xóa mất tham chiếu, nên bước kiểm thứ ba trong danh sách năm bước thất bại — vẫn là chặng 1, vẫn
là `401`. Thời gian để thay đổi lan tới API server **phụ thuộc cấu hình**, nên bước này viết
dưới dạng vòng lặp có điều kiện thoát chứ không phải một con số.

Đây cũng là lý do bài khuyến nghị **TokenReview API** thay vì tự xác minh bằng OIDC: OIDC vẫn
coi token là hợp lệ cho tới khi nó chạm mốc hết hạn, còn API server thì vô hiệu ngay.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B3. Chặng 2 — Phân quyền và RBAC

**Mục đích:** chứng minh chặng 2 nhận vào đúng ba thứ mà bài
[119](../119-controlling-access-vi.md) liệt kê — **username**, **hành động**, **đối tượng chịu
tác động** — rồi trả lời có hoặc không. Và phân biệt bốn thứ của bài
[120](../120-rbac-good-practices-vi.md) bằng thực nghiệm.

### B3.1. Ba cách hỏi cùng một câu hỏi

`kubectl auth can-i` chính là cách gõ tay của object mà bài
[119](../119-controlling-access-vi.md) in ra dưới dạng JSON. Định nghĩa một hàm dựng object đó
thật, để về sau đối chiếu hai đường:

```bash
sar() {   # $1 = chu the, $2 = verb, $3 = resource, $4 = namespace (de rong neu pham vi cluster)
  kubectl create -o jsonpath='{.status.allowed}' -f - <<YAML
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  user: "$1"
  resourceAttributes:
    namespace: "$4"
    verb: "$2"
    resource: "$3"
YAML
}

SA_RO='system:serviceaccount:lab-9a:sa-doc-cm'
```

Ba cách hỏi, cùng một chủ thể chưa được cấp gì:

```bash
A1="$(kubectl auth can-i list configmaps -n lab-9a --as="$SA_RO")"
A2="$(sar "$SA_RO" list configmaps lab-9a)"
A3="$(api_code /api/v1/namespaces/lab-9a/configmaps "$TOK_RO")"
echo "can-i = $A1 | SubjectAccessReview.allowed = $A2 | ma HTTP that = $A3"

test "$A1" = 'no' && test "$A2" = 'false' && test "$A3" = '403' \
  && echo 'PASS: ba duong hoi cho cung mot cau tra loi: chua duoc phep'
```

**Ý nghĩa:** `kubectl auth can-i` không phải một heuristic phía client — nó gửi một
SubjectAccessReview và đọc `.status.allowed`, đúng object mà bài
[119](../119-controlling-access-vi.md) minh họa. Vì thế nó là **công cụ kiểm chứng chính** của
lab: mọi bước sau đây hỏi bằng `can-i`, rồi thỉnh thoảng đối chiếu lại bằng mã HTTP thật.

Cờ `--as` là quyền **mạo danh**. Bạn dùng được nó vì kubeconfig quản trị có quyền đó; bài
[120](../120-rbac-good-practices-vi.md) xếp verb `impersonate` vào danh sách rủi ro leo thang
đặc quyền chính vì lý do này. Lab không cấp `impersonate` cho bất kỳ chủ thể nào nó tạo ra.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B3.2. Role và RoleBinding — quyền tối thiểu trong đúng một namespace

```bash
cat > ~/lab-work/9a/b3-role.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: doc-configmap
  namespace: lab-9a
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: doc-configmap
  namespace: lab-9a
subjects:
- kind: ServiceAccount
  name: sa-doc-cm
  namespace: lab-9a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: doc-configmap
EOF

kubectl apply -f ~/lab-work/9a/b3-role.yaml
kubectl create configmap cau-hinh-mau -n lab-9a --from-literal=khoa=gia-tri
kubectl describe rolebinding doc-configmap -n lab-9a \
  | tee ~/lab-evidence/9a/b3-rolebinding.txt
```

```bash
i=0; C_OK=''
while [ "$i" -lt 30 ]; do
  C_OK="$(api_code /api/v1/namespaces/lab-9a/configmaps "$TOK_RO")"
  test "$C_OK" = '200' && break
  i=$(( i + 1 )); sleep 1
done
echo "GET configmaps trong lab-9a sau khi bind = $C_OK (sau $i vong lap)"

test "$C_OK" = '200' \
  && echo 'PASS: cung mot token, cung mot request — chi khac la da co RoleBinding'
test "$(kubectl auth can-i list configmaps -n lab-9a --as="$SA_RO")" = 'yes' \
  && test "$(sar "$SA_RO" list configmaps lab-9a)" = 'true' \
  && echo 'PASS: can-i va SubjectAccessReview deu doi sang cho phep'
api_body /api/v1/namespaces/lab-9a/configmaps "$TOK_RO" \
  | grep -q 'ConfigMapList' \
  && echo 'PASS: du lieu tra ve dung la danh sach ConfigMap'
```

**Ý nghĩa:** ba bước của bài [118](../118-service-accounts-vi.md) vừa xong bước hai. Không có gì
trong Pod, trong token hay trong kubeconfig thay đổi — chỉ có một cặp Role và RoleBinding xuất
hiện trong namespace. Quyết định của chặng 2 là **trạng thái của cluster**, không phải thuộc
tính của credential.

`RoleBinding` gắn ai với cái gì; `Role` mô tả cái gì. Hai object tách nhau vì cùng một Role
thường được gắn cho nhiều chủ thể.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B3.3. Ranh giới namespace: cùng chủ thể, cùng verb, namespace khác

```bash
kubectl create configmap cau-hinh-mau -n lab-9a-khac --from-literal=khoa=gia-tri

C_KHAC="$(api_code /api/v1/namespaces/lab-9a-khac/configmaps "$TOK_RO")"
A_KHAC="$(kubectl auth can-i list configmaps -n lab-9a-khac --as="$SA_RO")"
S_KHAC="$(sar "$SA_RO" list configmaps lab-9a-khac)"
api_body /api/v1/namespaces/lab-9a-khac/configmaps "$TOK_RO" \
  | tee ~/lab-evidence/9a/b3-403-namespace-khac.txt
echo
echo "lab-9a = $C_OK | lab-9a-khac = $C_KHAC | can-i = $A_KHAC | SAR = $S_KHAC"
```

```bash
test "$C_OK" = '200' && test "$C_KHAC" = '403' \
  && echo 'PASS: Role chi co tac dung trong namespace chua no'
test "$A_KHAC" = 'no' && test "$S_KHAC" = 'false' \
  && echo 'PASS: can-i va SubjectAccessReview deu tu choi o namespace khac'
grep -q 'lab-9a-khac' ~/lab-evidence/9a/b3-403-namespace-khac.txt \
  && echo 'PASS: thong bao goi dung ten namespace bi tu choi'
```

**Ý nghĩa:** đây là hình ảnh trực quan nhất của "phạm vi namespace". Object Role nằm trong
`lab-9a`; RoleBinding cũng nằm trong `lab-9a`; quyền vì thế **dừng lại ở biên của `lab-9a`**.
Bài [120](../120-rbac-good-practices-vi.md) khuyên gán quyền ở cấp namespace **nếu có thể**, và
đây là lý do: phạm vi ảnh hưởng của một sai lầm bị chặn lại bởi chính cấu trúc của object.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B3.4. Ranh giới verb và ranh giới loại tài nguyên

```bash
{
  echo "=== quyen that su cua $SA_RO trong lab-9a ==="
  kubectl auth can-i --list -n lab-9a --as="$SA_RO"
} | tee ~/lab-evidence/9a/b3-can-i-list.txt

for v in get list watch create update delete; do
  printf '%-7s configmaps -> %s\n' "$v" \
    "$(kubectl auth can-i "$v" configmaps -n lab-9a --as="$SA_RO")"
done | tee ~/lab-evidence/9a/b3-verb-matrix.txt

for r in secrets pods serviceaccounts; do
  printf 'list %-16s -> %s\n' "$r" \
    "$(kubectl auth can-i list "$r" -n lab-9a --as="$SA_RO")"
done | tee -a ~/lab-evidence/9a/b3-verb-matrix.txt
```

```bash
test "$(kubectl auth can-i get    configmaps -n lab-9a --as="$SA_RO")" = 'yes' \
  && test "$(kubectl auth can-i list   configmaps -n lab-9a --as="$SA_RO")" = 'yes' \
  && echo 'PASS: dung hai verb da cap thi duoc'
CHAN=0
for v in watch create update delete; do
  test "$(kubectl auth can-i "$v" configmaps -n lab-9a --as="$SA_RO")" = 'no' \
    && CHAN=$(( CHAN + 1 ))
done
test "$CHAN" -eq 4 && echo 'PASS: bon verb khong cap thi deu bi tu choi, ke ca watch'
test "$(kubectl auth can-i list secrets -n lab-9a --as="$SA_RO")" = 'no' \
  && echo 'PASS: Role chi noi configmaps thi khong dung toi Secret'
```

Bài [120](../120-rbac-good-practices-vi.md) cảnh báo rằng `list` trên Secret **cũng làm lộ nội
dung**, chứ không riêng `get`. Chứng minh bằng một Secret vô hại do bạn tự tạo:

```bash
kubectl create secret generic vi-du-khong-bi-mat -n lab-9a \
  --from-literal=ghi-chu=day-la-du-lieu-mau

EXP_B64="$(printf '%s' 'day-la-du-lieu-mau' | base64 -w0)"
HIT="$(kubectl get secrets -n lab-9a -o yaml | grep -c "$EXP_B64")"
echo "so lan gia tri xuat hien trong phan hoi List = $HIT"

test "$HIT" -ge 1 \
  && echo 'PASS: mot phan hoi List chua luon noi dung cua Secret'
test "$(kubectl auth can-i list secrets -n lab-9a --as="$SA_RO")" = 'no' \
  && echo 'PASS: va vi the sa-doc-cm khong duoc phep list Secret'
```

**Ý nghĩa:** trực giác "list chỉ ra tên, get mới ra nội dung" là sai trong Kubernetes. Phản hồi
List **bao gồm nội dung của tất cả các Secret**, nên `list` và `watch` ngang với `get` về mức lộ
dữ liệu. Đây cũng là nội dung mà bài [113](../113-security-vi.md) gọi là **sự bảo vệ cơ bản**:
Secret API giữ giá trị tách khỏi ConfigMap và tách khỏi manifest, nhưng ai đọc được nó thì đọc
được thật — biện pháp kế tiếp là mã hóa khi lưu trữ, thuộc giai đoạn 22.

**PASS:** năm dòng `PASS:` của bước này xuất hiện.

### B3.5. ClusterRole gắn bằng RoleBinding — quyền bị giới hạn lại vào namespace của binding

Dùng một ClusterRole **có sẵn** của cluster, không tạo thêm object phạm vi cluster nào:

```bash
kubectl get clusterrole view -o yaml | head -20 \
  | tee ~/lab-evidence/9a/b3-clusterrole-view.txt

kubectl create serviceaccount sa-view -n lab-9a-khac
cat > ~/lab-work/9a/b3-view-binding.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: xem-trong-lab-9a-khac
  namespace: lab-9a-khac
subjects:
- kind: ServiceAccount
  name: sa-view
  namespace: lab-9a-khac
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
EOF
kubectl apply -f ~/lab-work/9a/b3-view-binding.yaml

SA_VIEW='system:serviceaccount:lab-9a-khac:sa-view'
```

```bash
V1="$(kubectl auth can-i list configmaps -n lab-9a-khac --as="$SA_VIEW")"
V2="$(kubectl auth can-i list configmaps -n lab-9a      --as="$SA_VIEW")"
V3="$(kubectl auth can-i list secrets    -n lab-9a-khac --as="$SA_VIEW")"
V4="$(kubectl auth can-i list nodes                     --as="$SA_VIEW")"
{
  echo "configmaps trong lab-9a-khac (namespace cua binding) = $V1"
  echo "configmaps trong lab-9a      (namespace khac)        = $V2"
  echo "secrets    trong lab-9a-khac                         = $V3"
  echo "nodes      (pham vi cluster)                         = $V4"
} | tee ~/lab-evidence/9a/b3-clusterrole-qua-rolebinding.txt

test "$V1" = 'yes' && test "$V2" = 'no' \
  && echo 'PASS: ClusterRole gan bang RoleBinding chi co tac dung trong namespace cua binding'
test "$V3" = 'no' \
  && echo 'PASS: ClusterRole view khong bao gom Secret — quyen la thu Role mo ta, khong phai pham vi'
test "$V4" = 'no' \
  && echo 'PASS: RoleBinding khong voi toi duoc tai nguyen pham vi cluster'
```

**Ý nghĩa:** đây là chỗ hay nhầm nhất trong bốn thứ. `ClusterRole` chỉ nói **quyền gì**; thứ
quyết định **ở đâu** là loại binding. Gắn bằng `RoleBinding` thì một ClusterRole rộng bị **thu
lại** đúng bằng namespace chứa binding. Nhờ đó bạn viết một ClusterRole `view` duy nhất rồi tái
sử dụng cho từng namespace, thay vì nhân bản Role ở mọi nơi.

Hai kết quả còn lại tách bạch hai chuyện khác nhau: `secrets` bị từ chối vì **Role không chứa
nó**, còn `nodes` bị từ chối vì **binding sai loại** — tài nguyên phạm vi cluster không thuộc
namespace nào để một RoleBinding cấp.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B3.6. ClusterRole gắn bằng ClusterRoleBinding — hiệu lực toàn cluster

Đây là hai object phạm vi cluster duy nhất mà lab tạo ra. Chúng đã được khai báo ở
[mục 2](#2-quy-ước-và-an-toàn): chỉ `get` và `list` trên `nodes`, không `nodes/proxy`, không verb
ghi, và mang label `lab=9a` để B7 dọn được theo label.

```bash
kubectl create serviceaccount sa-doc-node -n lab-9a

cat > ~/lab-work/9a/b3-clusterrole-node.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: lab-9a-doc-node
  labels:
    lab: "9a"
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: lab-9a-doc-node
  labels:
    lab: "9a"
subjects:
- kind: ServiceAccount
  name: sa-doc-node
  namespace: lab-9a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: lab-9a-doc-node
EOF
kubectl apply -f ~/lab-work/9a/b3-clusterrole-node.yaml

SA_NODE='system:serviceaccount:lab-9a:sa-doc-node'
TOK_NODE="$(kubectl create token sa-doc-node -n lab-9a)"
```

```bash
N1="$(kubectl auth can-i list nodes --as="$SA_NODE")"
N2="$(kubectl auth can-i get  nodes --subresource=proxy --as="$SA_NODE")"
N3="$(kubectl auth can-i delete nodes --as="$SA_NODE")"
N4="$(kubectl auth can-i list configmaps -n lab-9a --as="$SA_NODE")"
C_NODE="$(api_code /api/v1/nodes "$TOK_NODE")"
{
  echo "list nodes            = $N1"
  echo "get nodes/proxy       = $N2"
  echo "delete nodes          = $N3"
  echo "list configmaps lab-9a = $N4"
  echo "ma HTTP GET /api/v1/nodes = $C_NODE"
} | tee ~/lab-evidence/9a/b3-clusterrolebinding.txt

test "$N1" = 'yes' && test "$C_NODE" = '200' \
  && echo 'PASS: ClusterRoleBinding cap duoc tai nguyen pham vi cluster'
test "$N2" = 'no' \
  && echo 'PASS: khong cap nodes/proxy — dung danh sach ru ro cua bai 120'
test "$N3" = 'no' && test "$N4" = 'no' \
  && echo 'PASS: hieu luc toan cluster khong co nghia la lam duoc moi thu'
```

**Ý nghĩa:** `ClusterRoleBinding` mở quyền ra **toàn cluster**: mọi namespace, cộng các tài
nguyên không thuộc namespace nào. Vì thế bài [120](../120-rbac-good-practices-vi.md) xếp nó sau
`RoleBinding` trong thứ tự ưu tiên — dùng khi thật sự cần, và với một ClusterRole hẹp nhất có
thể.

Chú ý `N2`. Bài [120](../120-rbac-good-practices-vi.md) cảnh báo rằng verb **get** trên
`nodes/proxy` **không phải quyền chỉ đọc**: nó mở đường vào Kubelet API, cho phép lấy log và
thực thi lệnh trong mọi Pod của node đó, và **bỏ qua cả audit logging lẫn admission control**.
ClusterRole của lab cố ý dừng ở `nodes`; B6.4 sẽ dùng chính đường `nodes/proxy` bằng quyền quản
trị để bạn thấy nó làm được gì.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B3.7. Cùng ClusterRole đó, gắn bằng RoleBinding thì không còn gì

```bash
kubectl create serviceaccount sa-doc-node-ns -n lab-9a
cat > ~/lab-work/9a/b3-node-rolebinding.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: doc-node-trong-namespace
  namespace: lab-9a
subjects:
- kind: ServiceAccount
  name: sa-doc-node-ns
  namespace: lab-9a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: lab-9a-doc-node
EOF
kubectl apply -f ~/lab-work/9a/b3-node-rolebinding.yaml

SA_NODE_NS='system:serviceaccount:lab-9a:sa-doc-node-ns'
```

```bash
M1="$(kubectl auth can-i list nodes --as="$SA_NODE_NS")"
M2="$(kubectl auth can-i list nodes -n lab-9a --as="$SA_NODE_NS")"
M3="$(sar "$SA_NODE_NS" list nodes '')"
echo "list nodes = $M1 | list nodes -n lab-9a = $M2 | SubjectAccessReview = $M3"

test "$M1" = 'no' && test "$M2" = 'no' && test "$M3" = 'false' \
  && echo 'PASS: cung mot ClusterRole, doi loai binding la mat sach quyen'
test "$(kubectl auth can-i list nodes --as="$SA_NODE")" = 'yes' \
  && echo 'PASS: doi chung — SA duoc gan bang ClusterRoleBinding thi van duoc'
```

**Ý nghĩa:** bốn thứ của bài [120](../120-rbac-good-practices-vi.md) không phải bốn mức mạnh
yếu. Chúng là **hai trục**: Role và ClusterRole trả lời "quyền gì và định nghĩa ở đâu";
RoleBinding và ClusterRoleBinding trả lời "hiệu lực ở đâu". Ô giao nhau `ClusterRole × RoleBinding`
vừa cho ra một kết quả không trực giác: quyền đã viết sẵn trong ClusterRole, mà chủ thể vẫn
không dùng được, vì `nodes` không nằm trong bất kỳ namespace nào để binding này với tới.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B3.8. Ô còn lại của bảng không tồn tại

```bash
cat > ~/lab-work/9a/b3-to-hop-sai.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: lab-9a-to-hop-sai
  labels:
    lab: "9a"
subjects:
- kind: ServiceAccount
  name: sa-doc-cm
  namespace: lab-9a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: doc-configmap
EOF

kubectl apply -f ~/lab-work/9a/b3-to-hop-sai.yaml 2>&1 \
  | tee ~/lab-evidence/9a/b3-to-hop-sai.txt
```

```bash
grep -qi 'roleRef' ~/lab-evidence/9a/b3-to-hop-sai.txt \
  && echo 'PASS: API server tu choi va chi thang vao truong roleRef'
grep -qi 'ClusterRole' ~/lab-evidence/9a/b3-to-hop-sai.txt \
  && echo 'PASS: thong bao noi ro gia tri duy nhat duoc chap nhan la ClusterRole'
test -z "$(kubectl get clusterrolebinding lab-9a-to-hop-sai --ignore-not-found -o name)" \
  && echo 'PASS: khong object nao duoc tao'
```

**Ý nghĩa:** một `ClusterRoleBinding` **không thể** trỏ tới một `Role`, vì Role sống trong một
namespace còn binding thì không. Chỗ chết của request này đáng chú ý: nó **không** phải chặng 2
— bạn là quản trị viên, `kubectl auth can-i create clusterrolebindings` trả `yes`. Nó cũng không
phải chặng 3 theo nghĩa chính sách. Nó là **bước 4** trong sơ đồ của bài
[119](../119-controlling-access-vi.md): sau khi vượt hết admission controller, object được
**kiểm tra tính hợp lệ bằng các thủ tục dành cho đúng loại API** rồi mới ghi vào object store.
Đây là bước thứ tư, và nó tồn tại để chặn những object vô nghĩa về cấu trúc.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B3.9. Chủ thể có thể là một danh tính không hề tồn tại

```bash
cat > ~/lab-work/9a/b3-binding-nhom.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: doc-configmap-cho-nhom
  namespace: lab-9a
subjects:
- kind: Group
  name: lab-9a-doc
  apiGroup: rbac.authorization.k8s.io
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: doc-configmap
EOF
kubectl apply -f ~/lab-work/9a/b3-binding-nhom.yaml
```

```bash
G1="$(kubectl auth can-i list configmaps -n lab-9a \
       --as=nguoi-chua-bao-gio-ton-tai --as-group=lab-9a-doc)"
G2="$(kubectl auth can-i list configmaps -n lab-9a \
       --as=nguoi-chua-bao-gio-ton-tai)"
G3="$(kubectl auth can-i list configmaps -n lab-9a \
       --as=nguoi-khac --as-group=lab-9a-doc)"
{
  echo "user bat ky + group lab-9a-doc  = $G1"
  echo "cung user do, khong co group    = $G2"
  echo "user khac  + group lab-9a-doc   = $G3"
} | tee ~/lab-evidence/9a/b3-subject-nhom.txt

test "$G1" = 'yes' && test "$G2" = 'no' && test "$G3" = 'yes' \
  && echo 'PASS: quyet dinh phu thuoc vao nhom, khong phu thuoc vao ten nguoi dung'
test "$U_RES" -eq 0 \
  && echo 'PASS: va ca hai danh tinh do van khong ton tai duoi dang object nao'
```

**Ý nghĩa:** chỗ này khép lại mâu thuẫn có vẻ như của B2.2. Kubernetes **không có object
`User`**, nhưng RBAC vẫn gắn quyền cho user và group được — vì binding chỉ lưu **tên**, còn tên
đó đến từ đâu là việc của chặng 1. Đổi authenticator, đổi CA, đổi nhà cung cấp danh tính, các
binding không phải viết lại. Với cluster kubeadm của bạn, group đến từ trường `O` trong client
certificate mà B2.1 đã đọc.

Hệ quả vận hành mà bài [120](../120-rbac-good-practices-vi.md) nêu ở mục *Rà soát định kỳ*: nếu
ai đó tạo lại được một danh tính **trùng tên** với danh tính đã xóa, họ **thừa hưởng nguyên** các
quyền cũ — vì binding vẫn còn nguyên và vẫn trỏ đúng cái tên đó.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B3.10. RBAC tự chặn leo thang đặc quyền

Cấp cho một ServiceAccount quyền quản lý Role — nhưng **chỉ** quyền đó:

```bash
kubectl create serviceaccount sa-rbac -n lab-9a
cat > ~/lab-work/9a/b3-role-quan-ly-role.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: quan-ly-role
  namespace: lab-9a
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles"]
  verbs: ["get", "list", "create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: quan-ly-role
  namespace: lab-9a
subjects:
- kind: ServiceAccount
  name: sa-rbac
  namespace: lab-9a
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: quan-ly-role
EOF
kubectl apply -f ~/lab-work/9a/b3-role-quan-ly-role.yaml

SA_RBAC='system:serviceaccount:lab-9a:sa-rbac'
```

Thử tạo một Role **không rộng hơn** quyền đang có:

```bash
cat > ~/lab-work/9a/b3-role-hep.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: chi-doc-role
  namespace: lab-9a
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles"]
  verbs: ["get"]
EOF

kubectl create -f ~/lab-work/9a/b3-role-hep.yaml --as="$SA_RBAC" 2>&1 \
  | tee ~/lab-evidence/9a/b3-escalate-hep.txt
```

Rồi thử tạo một Role **rộng hơn** — quyền đọc Secret, thứ `sa-rbac` không hề có:

```bash
cat > ~/lab-work/9a/b3-role-rong.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: doc-secret
  namespace: lab-9a
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
EOF

kubectl create -f ~/lab-work/9a/b3-role-rong.yaml --as="$SA_RBAC" 2>&1 \
  | tee ~/lab-evidence/9a/b3-escalate-rong.txt
```

```bash
test -n "$(kubectl get role chi-doc-role -n lab-9a --ignore-not-found -o name)" \
  && echo 'PASS: tao duoc Role nam trong pham vi quyen dang co'
test -z "$(kubectl get role doc-secret -n lab-9a --ignore-not-found -o name)" \
  && echo 'PASS: Role rong hon quyen dang co thi khong tao duoc'
grep -qi 'forbidden' ~/lab-evidence/9a/b3-escalate-rong.txt \
  && echo 'PASS: bi chan o chang 2 voi Forbidden'
grep -qi 'escalate\|not currently held\|privilege' ~/lab-evidence/9a/b3-escalate-rong.txt \
  && echo 'PASS: thong bao noi ro ly do la leo thang dac quyen'

E1="$(kubectl auth can-i create roles       -n lab-9a --as="$SA_RBAC")"
E2="$(kubectl auth can-i escalate  roles    -n lab-9a --as="$SA_RBAC")"
E3="$(kubectl auth can-i bind      roles    -n lab-9a --as="$SA_RBAC")"
E4="$(kubectl auth can-i impersonate users             --as="$SA_RBAC")"
{
  echo "create roles = $E1"
  echo "escalate roles = $E2"
  echo "bind roles = $E3"
  echo "impersonate users = $E4"
} | tee ~/lab-evidence/9a/b3-verb-nguy-hiem.txt

test "$E1" = 'yes' && test "$E2" = 'no' && test "$E3" = 'no' && test "$E4" = 'no' \
  && echo 'PASS: sa-rbac khong co bat ky verb nguy hiem nao trong danh sach cua bai 120'
```

**Ý nghĩa:** RBAC có một quy tắc nền: **không ai tạo được một role có nhiều quyền hơn chính
mình**. Nhờ đó, cấp quyền quản lý Role cho một chủ thể không tự động biến nó thành quản trị viên.

Bài [120](../120-rbac-good-practices-vi.md) liệt kê những thứ **phá vỡ** quy tắc đó: verb
`escalate` cho phép tạo role vượt quyền mình; verb `bind` cho phép gắn chủ thể vào role mà mình
chưa có; verb `impersonate` cho phép mượn danh tính khác. Bốn dòng cuối chứng minh `sa-rbac`
không có cái nào — và lab không cấp cho ai cả.

**PASS:** năm dòng `PASS:` của bước này xuất hiện.

### B3.11. Rà soát binding mặc định — chỉ đọc

```bash
{
  echo "=== $(date -Is) — ra soat binding pham vi cluster ==="
  echo '--- ai duoc gan cluster-admin ---'
  kubectl get clusterrolebinding \
    -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.roleRef.name}{"\n"}{end}' \
    | awk -F' -> ' '$2=="cluster-admin"'
  echo '--- binding co nhac toi system:unauthenticated ---'
  kubectl get clusterrolebinding \
    -o jsonpath='{range .items[*]}{.metadata.name}{" | "}{range .subjects[*]}{.kind}{":"}{.name}{" "}{end}{"\n"}{end}' \
    | grep 'system:unauthenticated' || echo '(khong co)'
  echo '--- binding co nhac toi system:masters ---'
  kubectl get clusterrolebinding \
    -o jsonpath='{range .items[*]}{.metadata.name}{" | "}{range .subjects[*]}{.kind}{":"}{.name}{" "}{end}{"\n"}{end}' \
    | grep 'system:masters' || echo '(khong co)'
} | tee ~/lab-evidence/9a/b3-ra-soat-binding.txt
```

```bash
kubectl get clusterrolebinding \
  -o jsonpath='{range .items[*]}{.metadata.name}{" | "}{range .subjects[*]}{.name}{" "}{end}{"\n"}{end}' \
  > ~/lab-work/9a/b3-crb-subjects.txt

LAB_TRONG_ADMIN="$(kubectl get clusterrolebinding \
  -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.roleRef.name}{"\n"}{end}' \
  | awk -F' -> ' '$2=="cluster-admin"' | grep -c 'lab-9a' || true)"
LAB_TRONG_MASTERS="$(grep 'system:masters' ~/lab-work/9a/b3-crb-subjects.txt \
  | grep -c 'lab-9a' || true)"
CRB_LAB="$(kubectl get clusterrolebinding -l lab=9a --no-headers | wc -l)"
echo "binding cua lab tro toi cluster-admin = $LAB_TRONG_ADMIN"
echo "binding cua lab dinh system:masters   = $LAB_TRONG_MASTERS"
echo "tong binding pham vi cluster cua lab  = $CRB_LAB"

test "$LAB_TRONG_ADMIN" -eq 0 \
  && echo 'PASS: khong object nao cua lab duoc gan cluster-admin'
test "$LAB_TRONG_MASTERS" -eq 0 \
  && echo 'PASS: khong object nao cua lab dinh toi nhom system:masters'
test "$CRB_LAB" -eq 1 \
  && echo 'PASS: lab tao dung mot ClusterRoleBinding, dung nhu khai bao o muc 2'
```

**Ý nghĩa:** ba việc rà soát này là phần *Gia cố* và *Đặc quyền tối thiểu* của bài
[120](../120-rbac-good-practices-vi.md), làm ở mức **đọc**. Bạn sẽ thấy `cluster-admin` được gắn
cho một nhóm do kubeadm dựng sẵn — đó là danh tính trong kubeconfig quản trị của bạn, và đó là
lý do mọi lệnh `--as` ở trên chạy được.

Ba gate cuối là gate an toàn của chính lab, không phải bài học: chúng chứng minh lab giữ đúng
lời hứa ở [mục 2](#2-quy-ước-và-an-toàn). `system:masters` đáng sợ hơn `cluster-admin` ở chỗ nó
**bỏ qua toàn bộ kiểm tra RBAC** và **không thu hồi được bằng cách xóa binding** — nghĩa là một
sai lầm ở đó cleanup không dọn nổi.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B4. Chặng 3 — Kiểm soát tiếp nhận, và bước 4 phía sau nó

**Mục đích:** chứng minh ba tính chất mà bài [119](../119-controlling-access-vi.md) gán riêng cho
chặng 3: nó chặn được thứ mà chặng 2 đã cho qua; nó **không** chạm tới request chỉ đọc; và nó là
chặng duy nhất **sửa** được object. Lab **không bật thêm plugin nào** — mọi thứ dưới đây do các
admission controller đã có sẵn trên baseline làm.

### B4.1. Chặng 2 nói được, chặng 3 nói không

```bash
kubectl auth can-i create pods -n lab-9a | tee ~/lab-evidence/9a/b4-can-i-create-pod.txt

cat > ~/lab-work/9a/b4-sa-khong-ton-tai.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: sa-ma
  namespace: lab-9a
spec:
  serviceAccountName: khong-he-ton-tai
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/9a/b4-sa-khong-ton-tai.yaml 2>&1 \
  | tee ~/lab-evidence/9a/b4-admission-tu-choi.txt
```

```bash
test "$(kubectl auth can-i create pods -n lab-9a)" = 'yes' \
  && echo 'PASS: chang 2 cho phep — ban duoc quyen tao Pod trong namespace nay'
grep -q 'error looking up service account' ~/lab-evidence/9a/b4-admission-tu-choi.txt \
  && echo 'PASS: bi tu choi vi noi dung object, khong phai vi ban la ai'
test -z "$(kubectl get pod sa-ma -n lab-9a --ignore-not-found -o name)" \
  && echo 'PASS: khong Pod nao duoc tao'
```

**Ý nghĩa:** đọc kỹ câu thông báo. Nó **không** nói bạn thiếu quyền gì; nó nói cái
`serviceAccountName` bạn ghi trong `.spec` không tồn tại. Đó chính là điều bài
[119](../119-controlling-access-vi.md) mô tả: ngoài những thuộc tính mà module phân quyền thấy
được, admission controller còn truy cập được **nội dung của đối tượng đang được tạo**.

Đây cũng là cái bẫy thực dụng nhất của giai đoạn 9: mã trả về vẫn là `Forbidden`. `403` **không**
đồng nghĩa "thiếu quyền RBAC". Cách phân biệt là hỏi lại chặng 2 bằng `kubectl auth can-i` —
nếu nó nói `yes` mà request vẫn chết, thứ giết request nằm ở chặng 3, và câu thông báo sẽ chỉ
vào một trường cụ thể trong object.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B4.2. Cùng một namespace không tồn tại: đọc thì lọt, ghi thì không

```bash
NS_MA=lab-9a-khong-ton-tai
kubectl get namespace "$NS_MA" --ignore-not-found -o name

kubectl get configmaps -n "$NS_MA" > ~/lab-evidence/9a/b4-doc-ns-ma.txt 2>&1
RC_DOC=$?
kubectl create configmap thu -n "$NS_MA" --from-literal=a=b \
  > ~/lab-evidence/9a/b4-ghi-ns-ma.txt 2>&1
RC_GHI=$?
cat ~/lab-evidence/9a/b4-doc-ns-ma.txt ~/lab-evidence/9a/b4-ghi-ns-ma.txt
```

```bash
echo "exit code khi doc = $RC_DOC | exit code khi ghi = $RC_GHI"

test "$RC_DOC" -eq 0 \
  && echo 'PASS: request chi doc di qua tron tru du namespace khong ton tai'
test "$RC_GHI" -ne 0 \
  && echo 'PASS: request tao thi bi chan'
grep -qi 'not found' ~/lab-evidence/9a/b4-ghi-ns-ma.txt \
  && echo 'PASS: thong bao chi ra namespace khong ton tai'
test "$(kubectl auth can-i create configmaps -n "$NS_MA")" = 'yes' \
  && echo 'PASS: va chang 2 van tra loi yes cho chinh request bi chan do'
```

**Ý nghĩa:** hai request khác nhau đúng một verb, đi tới cùng một namespace không tồn tại, cho
hai kết quả khác nhau. Bài [119](../119-controlling-access-vi.md) giải thích gọn: admission
controller tác động lên request **tạo, sửa đổi, xóa hoặc proxy** một đối tượng, và **không tác
động lên request chỉ đọc**. `kubectl get` dừng lại sau chặng 2, nên nó không gặp ai để bị chặn
và trả về một danh sách rỗng.

Đây là câu trả lời cho câu hỏi "`kubectl get pods` có đi qua admission controller không" — không.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B4.3. Chặng 3 không chỉ từ chối, nó còn sửa object

Ba thứ dưới đây **không có** trong manifest bạn viết, nhưng có trong object được lưu:

```bash
cat > ~/lab-work/9a/b4-pod-tran.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: tran
  namespace: lab-9a
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF

kubectl apply -f ~/lab-work/9a/b4-pod-tran.yaml
kubectl wait --for=condition=Ready pod/tran -n lab-9a --timeout=180s
```

```bash
P_SA="$(kubectl get pod tran -n lab-9a -o jsonpath='{.spec.serviceAccountName}')"
P_VOL="$(kubectl get pod tran -n lab-9a -o jsonpath='{.spec.volumes[*].name}')"
P_MNT="$(kubectl get pod tran -n lab-9a \
  -o jsonpath='{.spec.containers[0].volumeMounts[*].mountPath}')"
{
  echo "serviceAccountName sau admission = $P_SA"
  echo "volumes  = $P_VOL"
  echo "mounts   = $P_MNT"
} | tee ~/lab-evidence/9a/b4-mutating.txt

grep -q 'serviceAccountName' ~/lab-work/9a/b4-pod-tran.yaml \
  && echo 'FAIL: manifest goc da co serviceAccountName' \
  || echo 'PASS: manifest goc khong he co serviceAccountName'
grep -q 'volumes' ~/lab-work/9a/b4-pod-tran.yaml \
  && echo 'FAIL: manifest goc da co volumes' \
  || echo 'PASS: manifest goc khong he co volumes'
test "$P_SA" = 'default' \
  && echo 'PASS: admission dien ServiceAccount default cua namespace vao Pod'
case "$P_VOL" in kube-api-access-*) \
  echo 'PASS: admission them mot projected volume mang credential vao Pod' ;; esac
case "$P_MNT" in */var/run/secrets/kubernetes.io/serviceaccount*) \
  echo 'PASS: volume do duoc mount vao dung duong dan chuan' ;; esac
```

Thứ thứ ba: `imagePullSecrets` gắn ở ServiceAccount cũng được chèn vào Pod mới. Đây là nhánh
*Thêm ImagePullSecrets vào một service account* của bài
[279](../279-configure-service-account-vi.md), làm với một Secret giả để không phụ thuộc registry:

```bash
kubectl create secret docker-registry registry-gia -n lab-9a \
  --docker-server=registry.vi-du.local \
  --docker-username=DUMMY_USERNAME \
  --docker-password=DUMMY_DOCKER_PASSWORD \
  --docker-email=DUMMY@vi-du.local

kubectl patch serviceaccount sa-doc-cm -n lab-9a \
  -p '{"imagePullSecrets":[{"name":"registry-gia"}]}'

cat > ~/lab-work/9a/b4-pod-pullsecret.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: co-pull-secret
  namespace: lab-9a
spec:
  serviceAccountName: sa-doc-cm
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    imagePullPolicy: IfNotPresent
    command: ["sh", "-c", "sleep 3600"]
EOF
kubectl apply -f ~/lab-work/9a/b4-pod-pullsecret.yaml
kubectl wait --for=condition=Ready pod/co-pull-secret -n lab-9a --timeout=180s
```

```bash
IPS="$(kubectl get pod co-pull-secret -n lab-9a \
  -o jsonpath='{.spec.imagePullSecrets[0].name}')"
echo "imagePullSecrets cua Pod = $IPS"

grep -q 'imagePullSecrets' ~/lab-work/9a/b4-pod-pullsecret.yaml \
  && echo 'FAIL: manifest goc da co imagePullSecrets' \
  || echo 'PASS: manifest goc khong he co imagePullSecrets'
test "$IPS" = 'registry-gia' \
  && echo 'PASS: admission chep imagePullSecrets tu ServiceAccount sang Pod'
```

**Ý nghĩa:** file trên đĩa không đổi, object trong etcd thì có. Bài
[119](../119-controlling-access-vi.md) nói ngoài việc từ chối, admission controller còn **thiết
lập các giá trị mặc định phức tạp cho các trường** — và ba lần chèn vừa rồi đều do plugin
`ServiceAccount` làm. Chính plugin đó là thứ giết request ở B4.1: cùng một plugin, một hướng
sửa, một hướng từ chối, tùy vào nội dung object.

Đây cũng là câu trả lời cho B1.3: gần như mọi Pod trên cluster mang `serviceAccountName` kể cả
khi manifest không hề khai — chặng 3 điền. Ngoại lệ duy nhất là **mirror Pod** của static Pod:
plugin này cố ý bỏ qua chúng, vì chúng thuộc quyền quản của kubelet chứ không của bạn.

**PASS:** bảy dòng `PASS:` của bước này xuất hiện.

### B4.4. Bước 4 — validate rồi ghi

```bash
kubectl patch pod tran -n lab-9a \
  -p '{"spec":{"serviceAccountName":"sa-doc-cm"}}' 2>&1 \
  | tee ~/lab-evidence/9a/b4-sua-sa-cua-pod.txt

kubectl patch rolebinding doc-configmap -n lab-9a \
  -p '{"roleRef":{"apiGroup":"rbac.authorization.k8s.io","kind":"Role","name":"chi-doc-role"}}' 2>&1 \
  | tee ~/lab-evidence/9a/b4-sua-roleref.txt
```

```bash
SA_SAU="$(kubectl get pod tran -n lab-9a -o jsonpath='{.spec.serviceAccountName}')"
REF_SAU="$(kubectl get rolebinding doc-configmap -n lab-9a -o jsonpath='{.roleRef.name}')"
echo "serviceAccountName cua Pod tran = $SA_SAU (truoc do: $P_SA)"
echo "roleRef cua RoleBinding         = $REF_SAU"

test "$SA_SAU" = "$P_SA" \
  && echo 'PASS: serviceAccountName cua Pod dang chay khong sua duoc'
test "$REF_SAU" = 'doc-configmap' \
  && echo 'PASS: roleRef cua mot binding da ton tai khong sua duoc'
grep -qi 'forbidden\|invalid\|may not\|immutable' ~/lab-evidence/9a/b4-sua-sa-cua-pod.txt \
  && echo 'PASS: API server noi ro day la truong khong duoc thay doi'
```

**Ý nghĩa:** hai request này qua hết ba chặng — bạn có quyền, nội dung object không vi phạm chính
sách nào — rồi chết ở **bước 4**, nơi object được kiểm tra hợp lệ theo thủ tục riêng của từng
loại API trước khi ghi vào object store.

Hai trường bất biến này có lý do bảo mật, không phải kỹ thuật. `serviceAccountName` bất biến vì
danh tính của một Pod đang chạy không được đổi giữa chừng — bài
[279](../279-configure-service-account-vi.md) nói rõ bạn chỉ đặt được field này **khi tạo Pod
hoặc trong template của một Pod mới**. `roleRef` bất biến để không ai âm thầm trỏ một binding
hiện có sang một role mạnh hơn.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B5. ServiceAccount là danh tính của Pod

**Mục đích:** làm bước ba của bài [118](../118-service-accounts-vi.md) — **gán ServiceAccount cho
Pod** — rồi chứng minh từ bên trong container mọi điều bài
[338](../338-access-api-from-pod-vi.md) khẳng định.

### B5.1. ServiceAccount `default` và trần quyền của nó

Pod `tran` ở B4.3 đang chạy trên `lab-k8s-worker2` dưới ServiceAccount `default`. Cho nó tự gọi
API server bằng đúng cách mà bài [338](../338-access-api-from-pod-vi.md) mô tả ở mục *Không sử
dụng proxy*:

```bash
kubectl exec -n lab-9a tran -- sh -c '
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  wget -O- --no-check-certificate \
    --header "Authorization: Bearer $(cat $SA/token)" \
    "https://kubernetes.default.svc/api" 2>&1
' | tee ~/lab-evidence/9a/b5-default-discovery.txt

kubectl exec -n lab-9a tran -- sh -c '
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  NS=$(cat $SA/namespace)
  wget -O- --no-check-certificate \
    --header "Authorization: Bearer $(cat $SA/token)" \
    "https://kubernetes.default.svc/api/v1/namespaces/$NS/configmaps" 2>&1
' | tee ~/lab-evidence/9a/b5-default-configmaps.txt
```

```bash
grep -q 'APIVersions' ~/lab-evidence/9a/b5-default-discovery.txt \
  && echo 'PASS: Pod goi duoc API server bang token cua chinh no'
grep -q '403' ~/lab-evidence/9a/b5-default-configmaps.txt \
  && echo 'PASS: nhung khong doc noi ConfigMap ngay trong namespace cua no'

D1="$(kubectl auth can-i get /api --as=system:serviceaccount:lab-9a:default)"
D2="$(kubectl auth can-i list configmaps -n lab-9a --as=system:serviceaccount:lab-9a:default)"
D3="$(kubectl auth can-i list secrets    -n lab-9a --as=system:serviceaccount:lab-9a:default)"
{
  echo "get /api          = $D1"
  echo "list configmaps   = $D2"
  echo "list secrets      = $D3"
} | tee ~/lab-evidence/9a/b5-quyen-cua-default.txt

test "$D1" = 'yes' && test "$D2" = 'no' && test "$D3" = 'no' \
  && echo 'PASS: default chi co quyen API discovery, dung nhu bai 118 noi'
```

**Ý nghĩa:** đây là câu quan trọng nhất về ServiceAccount `default`: nó **không có quyền hạn nào**
ngoài các quyền API discovery mặc định mà Kubernetes cấp cho mọi principal đã xác thực khi RBAC
được bật. Một Pod không khai gì vẫn **xác thực được** với API server — và đúng ra thì chỉ thế.
Mọi thứ nó làm thêm được đều là do ai đó đã bind quyền cho `default`, và đó là điều bài
[120](../120-rbac-good-practices-vi.md) khuyên tránh: đừng cấp quyền cho `default`, hãy tạo
ServiceAccount riêng cho từng workload.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B5.2. Token được chiếu vào Pod ở đâu, và bằng cơ chế nào

```bash
kubectl get pod tran -n lab-9a -o jsonpath='{.spec.volumes[0].projected.sources}' \
  | tee ~/lab-evidence/9a/b5-projected-sources.json
echo
kubectl exec -n lab-9a tran -- ls -l /var/run/secrets/kubernetes.io/serviceaccount/ \
  | tee ~/lab-evidence/9a/b5-ba-file.txt
```

```bash
SRC="$(kubectl get pod tran -n lab-9a -o jsonpath='{.spec.volumes[0].projected.sources}')"
echo "$SRC" | grep -q 'serviceAccountToken' \
  && echo 'PASS: nguon thu nhat cua projected volume la serviceAccountToken'
echo "$SRC" | grep -q 'kube-root-ca.crt' \
  && echo 'PASS: nguon thu hai la ConfigMap chua certificate CA cua cluster'
echo "$SRC" | grep -q 'downwardAPI' \
  && echo 'PASS: nguon thu ba la downwardAPI, mang ten namespace vao'

for f in token ca.crt namespace; do
  kubectl exec -n lab-9a tran -- \
    test -f "/var/run/secrets/kubernetes.io/serviceaccount/$f" \
    && echo "PASS: co file $f trong Pod"
done
```

Ba giá trị dưới đây phải khớp với cluster, không phải "trông giống":

```bash
NS_TRONG_POD="$(kubectl exec -n lab-9a tran -- \
  cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)"
SVC_IP="$(kubectl get svc kubernetes -n default -o jsonpath='{.spec.clusterIP}')"
SVC_PORT="$(kubectl get svc kubernetes -n default -o jsonpath='{.spec.ports[0].port}')"
ENV_HOST="$(kubectl exec -n lab-9a tran -- printenv KUBERNETES_SERVICE_HOST)"
ENV_PORT="$(kubectl exec -n lab-9a tran -- printenv KUBERNETES_SERVICE_PORT_HTTPS)"
{
  echo "namespace trong Pod = $NS_TRONG_POD"
  echo "KUBERNETES_SERVICE_HOST = $ENV_HOST (ClusterIP that = $SVC_IP)"
  echo "KUBERNETES_SERVICE_PORT_HTTPS = $ENV_PORT (port that = $SVC_PORT)"
} | tee ~/lab-evidence/9a/b5-doi-chieu-gia-tri.txt

test "$NS_TRONG_POD" = 'lab-9a' \
  && echo 'PASS: file namespace mang dung namespace cua Pod'
test "$ENV_HOST" = "$SVC_IP" && test "$ENV_PORT" = "$SVC_PORT" \
  && echo 'PASS: hai bien moi truong tro dung Service kubernetes trong namespace default'
```

Và certificate CA trong Pod phải là **chính** CA của cluster:

```bash
kubectl get configmap kube-root-ca.crt -n lab-9a -o jsonpath='{.data.ca\.crt}' \
  > ~/lab-work/9a/ca-tu-configmap.crt

H_CM="$(sha256sum ~/lab-work/9a/ca-tu-configmap.crt | awk '{print $1}')"
H_POD="$(kubectl exec -n lab-9a tran -- \
  sha256sum /var/run/secrets/kubernetes.io/serviceaccount/ca.crt | awk '{print $1}')"
F_CM="$(openssl x509 -noout -fingerprint -sha256 -in ~/lab-work/9a/ca-tu-configmap.crt \
  | cut -d= -f2)"
F_NODE="$(sudo openssl x509 -noout -fingerprint -sha256 -in /etc/kubernetes/pki/ca.crt \
  | cut -d= -f2)"
{
  echo "sha256 ca.crt trong Pod        = $H_POD"
  echo "sha256 ca.crt trong ConfigMap  = $H_CM"
  echo "fingerprint tu ConfigMap       = $F_CM"
  echo "fingerprint tu /etc/kubernetes = $F_NODE"
} | tee ~/lab-evidence/9a/b5-ca-doi-chieu.txt

test "$H_POD" = "$H_CM" \
  && echo 'PASS: file ca.crt trong Pod dung la noi dung ConfigMap kube-root-ca.crt'
test "$F_CM" = "$F_NODE" \
  && echo 'PASS: va do chinh la CA dang ky certificate cua API server'
```

Cuối cùng, danh tính nằm trong chính token — giải mã **bên trong** Pod để token không rời khỏi
container:

```bash
PAY_POD="$(kubectl exec -n lab-9a tran -- sh -c '
  P="$(cut -d. -f2 < /var/run/secrets/kubernetes.io/serviceaccount/token)"
  case $(( ${#P} % 4 )) in 2) P="${P}==" ;; 3) P="${P}=" ;; esac
  printf "%s" "$P" | tr -- "-_" "+/" | base64 -d
')"
SUB_POD="$(printf '%s' "$PAY_POD" | sed -n 's/.*"sub":"\([^"]*\)".*/\1/p')"
echo "sub cua token trong Pod = $SUB_POD" | tee ~/lab-evidence/9a/b5-sub-trong-pod.txt

test "$SUB_POD" = 'system:serviceaccount:lab-9a:default' \
  && echo 'PASS: sub cua token trong Pod chinh la ServiceAccount cua Pod'
printf '%s' "$PAY_POD" | grep -q '"pod"' \
  && echo 'PASS: payload con ghi ca tham chieu toi chinh Pod — day la mot bound token'
```

**Ý nghĩa:** toàn bộ B5.2 là bài [338](../338-access-api-from-pod-vi.md) được kiểm chứng từng
câu. Ba file, hai biến môi trường, một tên DNS `kubernetes.default.svc` — không cái nào là quy
ước lỏng lẻo, tất cả đều có nguồn cụ thể trong object Pod.

Cơ chế đưa chúng vào là **projected volume** với ba nguồn, đúng như bài
[118](../118-service-accounts-vi.md) mô tả cho hành vi từ v1.22: token lấy qua API
`TokenRequest`, ngắn hạn, **tự động xoay vòng**. Khác hẳn cơ chế trước đó — token tĩnh, tồn tại
lâu dài, lưu dưới dạng Secret — mà B6.5 sẽ dựng lại để so.

Claim `pod` trong payload là thứ khiến token này là **bound token**: nó chết theo Pod, đúng như
`sa-tam` ở B2.6 chết theo ServiceAccount.

**PASS:** mười dòng `PASS:` của bước này xuất hiện.

### B5.3. Pod với ServiceAccount riêng: hai namespace, hai kết quả

Pod `co-pull-secret` ở B4.3 đang chạy dưới `sa-doc-cm` — ServiceAccount đã được cấp quyền đọc
ConfigMap trong `lab-9a` ở B3.2, và không được cấp gì ở `lab-9a-khac`.

```bash
kubectl exec -n lab-9a co-pull-secret -- sh -c '
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  wget -O- --no-check-certificate \
    --header "Authorization: Bearer $(cat $SA/token)" \
    "https://kubernetes.default.svc/api/v1/namespaces/lab-9a/configmaps" 2>&1
' | tee ~/lab-evidence/9a/b5-sa-rieng-lab9a.txt

kubectl exec -n lab-9a co-pull-secret -- sh -c '
  SA=/var/run/secrets/kubernetes.io/serviceaccount
  wget -O- --no-check-certificate \
    --header "Authorization: Bearer $(cat $SA/token)" \
    "https://kubernetes.default.svc/api/v1/namespaces/lab-9a-khac/configmaps" 2>&1
' | tee ~/lab-evidence/9a/b5-sa-rieng-lab9a-khac.txt
```

```bash
grep -q 'ConfigMapList' ~/lab-evidence/9a/b5-sa-rieng-lab9a.txt \
  && echo 'PASS: Pod doc duoc ConfigMap trong namespace cua no'
grep -q '403' ~/lab-evidence/9a/b5-sa-rieng-lab9a-khac.txt \
  && echo 'PASS: cung Pod do bi 403 o namespace ben canh'
SUB_SA="$(kubectl exec -n lab-9a co-pull-secret -- sh -c '
  P="$(cut -d. -f2 < /var/run/secrets/kubernetes.io/serviceaccount/token)"
  case $(( ${#P} % 4 )) in 2) P="${P}==" ;; 3) P="${P}=" ;; esac
  printf "%s" "$P" | tr -- "-_" "+/" | base64 -d
' | sed -n 's/.*"sub":"\([^"]*\)".*/\1/p')"
test "$SUB_SA" = 'system:serviceaccount:lab-9a:sa-doc-cm' \
  && echo 'PASS: Pod dang mang dung danh tinh ma spec.serviceAccountName khai'
```

**Ý nghĩa:** đây là toàn bộ nguyên tắc quyền tối thiểu, đo được: một danh tính, một namespace,
hai verb. Không có gì trong container biết về Role hay RoleBinding; nó chỉ có một token. Ranh
giới nằm ở phía API server, và nó là ranh giới **namespace** — đúng thứ bài
[120](../120-rbac-good-practices-vi.md) gọi là đơn vị tách các mức tin cậy.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B5.4. Từ chối tự động mount, ở hai cấp, và cấp nào thắng

```bash
cat > ~/lab-work/9a/b5-sa-khong-token.yaml <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-khong-token
  namespace: lab-9a
automountServiceAccountToken: false
EOF
kubectl apply -f ~/lab-work/9a/b5-sa-khong-token.yaml

cat > ~/lab-work/9a/b5-hai-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: khong-token
  namespace: lab-9a
spec:
  serviceAccountName: sa-khong-token
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-ep-token
  namespace: lab-9a
spec:
  serviceAccountName: sa-khong-token
  automountServiceAccountToken: true
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
EOF
kubectl apply -f ~/lab-work/9a/b5-hai-pod.yaml
kubectl wait --for=condition=Ready pod/khong-token pod/pod-ep-token \
  -n lab-9a --timeout=180s
```

```bash
V_KHONG="$(kubectl get pod khong-token  -n lab-9a -o jsonpath='{.spec.volumes[*].name}')"
V_EP="$(   kubectl get pod pod-ep-token -n lab-9a -o jsonpath='{.spec.volumes[*].name}')"
kubectl exec -n lab-9a khong-token -- \
  test -d /var/run/secrets/kubernetes.io/serviceaccount >/dev/null 2>&1; RC_K=$?
kubectl exec -n lab-9a pod-ep-token -- \
  test -d /var/run/secrets/kubernetes.io/serviceaccount >/dev/null 2>&1; RC_E=$?
{
  echo "pod khong-token  : volumes='$V_KHONG' thu muc credential ton tai? exit=$RC_K"
  echo "pod pod-ep-token : volumes='$V_EP' thu muc credential ton tai? exit=$RC_E"
} | tee ~/lab-evidence/9a/b5-automount.txt

test -z "$V_KHONG" && test "$RC_K" -ne 0 \
  && echo 'PASS: SA dat automountServiceAccountToken=false thi Pod khong nhan credential nao'
case "$V_EP" in kube-api-access-*) \
  test "$RC_E" -eq 0 \
    && echo 'PASS: Pod dat true de len tren thiet lap cua ServiceAccount' ;; esac
```

**Ý nghĩa:** hai Pod dùng **cùng một** ServiceAccount và cho hai kết quả trái ngược. Bài
[279](../279-configure-service-account-vi.md) chốt luật: nếu cả ServiceAccount lẫn `.spec` của
Pod cùng đặt `automountServiceAccountToken`, **spec của Pod được ưu tiên**.

Vế thứ nhất là khuyến nghị gia cố của bài [120](../120-rbac-good-practices-vi.md): tắt tự động
mount ở những chỗ workload không cần gọi API, để một container bị chiếm cũng không sẵn credential
trong tay. Vế thứ hai là lối thoát cho workload thật sự cần.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B5.5. Chiếu token với audience riêng — và hệ quả ở chặng 1

```bash
cat > ~/lab-work/9a/b5-pod-audience.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: token-rieng
  namespace: lab-9a
spec:
  serviceAccountName: sa-doc-cm
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: token-cho-dich-vu-khac
  volumes:
  - name: token-cho-dich-vu-khac
    projected:
      sources:
      - serviceAccountToken:
          path: token-dich-vu-khac
          expirationSeconds: 3600
          audience: lab-9a-dich-vu-khac
EOF
kubectl apply -f ~/lab-work/9a/b5-pod-audience.yaml
kubectl wait --for=condition=Ready pod/token-rieng -n lab-9a --timeout=180s
```

```bash
PAY_AUD="$(kubectl exec -n lab-9a token-rieng -- sh -c '
  P="$(cut -d. -f2 < /var/run/secrets/tokens/token-dich-vu-khac)"
  case $(( ${#P} % 4 )) in 2) P="${P}==" ;; 3) P="${P}=" ;; esac
  printf "%s" "$P" | tr -- "-_" "+/" | base64 -d
')"
AUD_POD="$(printf '%s' "$PAY_AUD" | sed -n 's/.*"aud":\(\[[^]]*\]\).*/\1/p')"
SUB_AUD="$(printf '%s' "$PAY_AUD" | sed -n 's/.*"sub":"\([^"]*\)".*/\1/p')"
{
  echo "aud = $AUD_POD"
  echo "sub = $SUB_AUD"
} | tee ~/lab-evidence/9a/b5-token-audience.txt

echo "$AUD_POD" | grep -q 'lab-9a-dich-vu-khac' \
  && echo 'PASS: token trong volume rieng mang dung audience da yeu cau'
test "$SUB_AUD" = 'system:serviceaccount:lab-9a:sa-doc-cm' \
  && echo 'PASS: van la danh tinh cua ServiceAccount gan cho Pod'
```

Dùng chính token đó gọi API server — nơi audience này vô nghĩa:

```bash
kubectl exec -n lab-9a token-rieng -- sh -c '
  wget -O- --no-check-certificate \
    --header "Authorization: Bearer $(cat /var/run/secrets/tokens/token-dich-vu-khac)" \
    "https://kubernetes.default.svc/api" 2>&1
' | tee ~/lab-evidence/9a/b5-audience-401.txt

grep -q '401' ~/lab-evidence/9a/b5-audience-401.txt \
  && echo 'PASS: API server tu choi o chang 1 vi token khong danh cho no'

kubectl exec -n lab-9a token-rieng -- sh -c '
  wget -O- --no-check-certificate \
    --header "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    "https://kubernetes.default.svc/api/v1/namespaces/lab-9a/configmaps" 2>&1
' | grep -q 'ConfigMapList' \
  && echo 'PASS: token mac dinh cua cung Pod do van dung binh thuong'
```

**Ý nghĩa:** một Pod có thể mang **nhiều** token cùng lúc, mỗi cái cho một bên tin cậy khác nhau,
và mỗi cái chỉ dùng được ở đúng nơi audience của nó chỉ tới. Đó là ý của bài
[118](../118-service-accounts-vi.md) khi nói ứng dụng nên **định nghĩa audience nó chấp nhận** —
để token bị lộ ở một dịch vụ không mở được cửa ở dịch vụ khác.

`expirationSeconds` là **yêu cầu**, không phải cam kết: thời hạn thực tế phụ thuộc cấu hình của
API server, và kubelet chủ động làm mới token trước khi hết hạn. Vì thế lab không gate lên bất
kỳ con số thời gian nào — chỉ gate lên audience và `sub`.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

---

## B6. Client bên ngoài cluster xác thực thế nào

**Mục đích:** đi hết bài [359](../359-access-cluster-vi.md) trên cluster thật — kubeconfig,
`kubectl proxy`, và nhánh "không dùng proxy" — rồi chốt lại hai loại token của bài
[279](../279-configure-service-account-vi.md) bằng chính payload của chúng.

### B6.1. Một kubeconfig cho ServiceAccount

Bài [359](../359-access-cluster-vi.md) nói để truy cập một cluster bạn cần **vị trí** và **thông
tin xác thực**. Dựng đủ hai thứ đó cho `sa-doc-cm`, không dùng lại gì của kubeconfig quản trị
ngoài địa chỉ server và CA:

```bash
KCFG=~/lab-work/9a/sa-doc-cm.kubeconfig
rm -f "$KCFG"; touch "$KCFG"; chmod 600 "$KCFG"

kubectl --kubeconfig="$KCFG" config set-cluster lab \
  --server="$SRV" --certificate-authority="$CACERT" --embed-certs=true
kubectl --kubeconfig="$KCFG" config set-credentials sa-doc-cm --token="$TOK_RO"
kubectl --kubeconfig="$KCFG" config set-context lab \
  --cluster=lab --user=sa-doc-cm --namespace=lab-9a
kubectl --kubeconfig="$KCFG" config use-context lab

kubectl --kubeconfig="$KCFG" config view | tee ~/lab-evidence/9a/b6-kubeconfig-sa.txt
```

```bash
SA_WHO="$(kubectl --kubeconfig="$KCFG" auth whoami \
  -o jsonpath='{.status.userInfo.username}')"
SA_GRP="$(kubectl --kubeconfig="$KCFG" auth whoami \
  -o jsonpath='{.status.userInfo.groups}')"
{
  echo "username = $SA_WHO"
  echo "groups   = $SA_GRP"
} | tee ~/lab-evidence/9a/b6-whoami-sa.txt

test "$SA_WHO" = "$SA_RO" \
  && echo 'PASS: kubeconfig nay xac thuc dung danh tinh ServiceAccount'
echo "$SA_GRP" | grep -q 'system:serviceaccounts:lab-9a' \
  && echo 'PASS: authenticator con gan them nhom theo namespace cho ServiceAccount'

kubectl --kubeconfig="$KCFG" get configmaps >/dev/null 2>&1 \
  && echo 'PASS: kubeconfig nay doc duoc ConfigMap trong lab-9a'
kubectl --kubeconfig="$KCFG" get secrets > ~/lab-evidence/9a/b6-sa-doc-secret.txt 2>&1
grep -qi 'forbidden' ~/lab-evidence/9a/b6-sa-doc-secret.txt \
  && echo 'PASS: va van khong dung toi Secret'
test "$(stat -c '%a' "$KCFG")" = '600' \
  && echo 'PASS: file kubeconfig chua token dang o quyen 600'
```

**Ý nghĩa:** kubeconfig **không** phải một cơ chế xác thực. Nó là một cái hộp đựng ba thứ: địa
chỉ, cách tin server (CA), và credential. Đổi credential trong hộp là đổi danh tính; mọi chặng
sau đó không hề biết bạn dùng file nào.

Nhóm `system:serviceaccounts:lab-9a` xuất hiện tự động cho mọi ServiceAccount của namespace đó.
Nó là công cụ mạnh — và cũng là rủi ro: bind một Role cho nhóm này là bind cho **mọi**
ServiceAccount trong namespace, hiện tại lẫn tương lai.

**PASS:** năm dòng `PASS:` của bước này xuất hiện.

### B6.2. `kubectl proxy` — client nói HTTP, proxy lo phần còn lại

```bash
ss -ltn | grep -q ':8001 ' \
  && echo 'FAIL: cong 8001 dang ban — doi sang cong khac truoc khi tiep tuc' \
  || echo 'PASS: cong 8001 dang ranh'

kubectl proxy --port=8001 > ~/lab-evidence/9a/b6-proxy-admin.log 2>&1 &
PROXY_PID=$!

i=0; C_PROXY=''
while [ "$i" -lt 30 ]; do
  C_PROXY="$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8001/api/ || true)"
  test "$C_PROXY" = '200' && break
  i=$(( i + 1 )); sleep 1
done
echo "ma HTTP qua proxy = $C_PROXY (sau $i vong lap)"

curl -s http://127.0.0.1:8001/api/ | tee ~/lab-evidence/9a/b6-proxy-api.json
echo
```

```bash
test "$C_PROXY" = '200' \
  && echo 'PASS: doc duoc API qua HTTP thuan tren localhost, khong header xac thuc nao'
grep -q 'APIVersions' ~/lab-evidence/9a/b6-proxy-api.json \
  && echo 'PASS: noi dung tra ve dung la tai lieu discovery cua API'
C_TRUC_TIEP="$(api_code /api '')"
test "$C_TRUC_TIEP" != '200' \
  && echo 'PASS: cung request do goi thang toi API server thi khong duoc — proxy da lam gi do'

kill "$PROXY_PID" 2>/dev/null
wait "$PROXY_PID" 2>/dev/null
unset PROXY_PID
```

**Ý nghĩa:** bốn việc mà bài [359](../359-access-cluster-vi.md) liệt kê cho `kubectl proxy`, đọc
lại theo đúng thứ tự vừa quan sát: nó **chạy trên máy người dùng**, **proxy từ một địa chỉ
localhost tới apiserver**, **client tới proxy dùng HTTP** còn **proxy tới apiserver dùng HTTPS**,
nó **định vị apiserver** và **thêm các header xác thực**. Ba dòng gate ở trên chứng minh đúng
mệnh đề cuối: bỏ proxy ra, cùng một request không còn đi tới đâu.

Vì proxy nghe trên `localhost`, ai chạm được vào máy đó là chạm được vào quyền của proxy. Đừng
bao giờ để nó lắng nghe trên interface ra ngoài.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B6.3. Proxy mang đúng quyền của kubeconfig nó dùng

```bash
kubectl --kubeconfig="$KCFG" proxy --port=8001 \
  > ~/lab-evidence/9a/b6-proxy-sa.log 2>&1 &
PROXY_PID=$!

i=0; C_UP=''
while [ "$i" -lt 30 ]; do
  C_UP="$(curl -s -o /dev/null -w '%{http_code}' \
    http://127.0.0.1:8001/api/v1/namespaces/lab-9a/configmaps || true)"
  test -n "$C_UP" && test "$C_UP" != '000' && break
  i=$(( i + 1 )); sleep 1
done

C_CM_PROXY="$(curl -s -o /dev/null -w '%{http_code}' \
  http://127.0.0.1:8001/api/v1/namespaces/lab-9a/configmaps)"
C_SEC_PROXY="$(curl -s -o /dev/null -w '%{http_code}' \
  http://127.0.0.1:8001/api/v1/namespaces/lab-9a/secrets)"
C_NODE_PROXY="$(curl -s -o /dev/null -w '%{http_code}' \
  http://127.0.0.1:8001/api/v1/nodes)"
{
  echo "configmaps qua proxy cua SA = $C_CM_PROXY"
  echo "secrets    qua proxy cua SA = $C_SEC_PROXY"
  echo "nodes      qua proxy cua SA = $C_NODE_PROXY"
} | tee ~/lab-evidence/9a/b6-proxy-quyen.txt
```

```bash
test "$C_CM_PROXY" = '200' \
  && echo 'PASS: proxy chay bang kubeconfig cua SA doc duoc dung thu SA duoc phep'
test "$C_SEC_PROXY" = '403' && test "$C_NODE_PROXY" = '403' \
  && echo 'PASS: va bi chan y het khi goi thang — proxy khong them quyen nao'

kill "$PROXY_PID" 2>/dev/null
wait "$PROXY_PID" 2>/dev/null
unset PROXY_PID
ss -ltn | grep -q ':8001 ' \
  && echo 'FAIL: van con tien trinh nghe tren 8001' \
  || echo 'PASS: da dong proxy, cong 8001 tra lai'
```

**Ý nghĩa:** proxy **xác thực hộ**, không **phân quyền hộ**. Nó gắn credential của kubeconfig nó
đang cầm rồi chuyển tiếp; ba chặng phía sau vẫn chạy nguyên vẹn ở API server. Đây là chỗ nhiều
người hiểu sai và mở `kubectl proxy` bằng kubeconfig quản trị cho cả một dashboard dùng chung —
tức là trao quyền quản trị cho bất kỳ ai gọi được cái cổng đó.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B6.4. Bốn loại proxy — cái nào thật sự có trên cluster này

| Loại proxy trong bài 359 | Trên cluster lab |
| --- | --- |
| `kubectl proxy` | có — B6.2 và B6.3 |
| apiserver proxy | có — chính đường `/api/v1/nodes/<node>/proxy/...` dưới đây |
| kube-proxy | có — DaemonSet trong `kube-system` |
| Proxy hoặc load balancer trước apiserver | **không** — cluster chỉ có một control plane |
| Cloud Load Balancer | **không** — không có nhà cung cấp cloud |

```bash
HZ="$(kubectl get --raw '/api/v1/nodes/lab-k8s-worker2/proxy/healthz')"
KP="$(kubectl -n kube-system get daemonset kube-proxy -o name 2>/dev/null)"
Q1="$(kubectl auth can-i get nodes --subresource=proxy --as="$SA_RO")"
Q2="$(kubectl auth can-i get nodes --subresource=proxy --as="$SA_NODE")"
{
  echo "apiserver proxy toi kubelet cua worker2 -> $HZ"
  echo "kube-proxy -> ${KP:-khong co}"
  echo "sa-doc-cm  co get nodes/proxy khong  -> $Q1"
  echo "sa-doc-node co get nodes/proxy khong -> $Q2"
} | tee ~/lab-evidence/9a/b6-cac-loai-proxy.txt

test "$HZ" = 'ok' \
  && echo 'PASS: apiserver proxy dua duoc request toi kubelet cua mot node'
test -n "$KP" \
  && echo 'PASS: kube-proxy dang chay dang DaemonSet'
test "$Q1" = 'no' && test "$Q2" = 'no' \
  && echo 'PASS: khong chu the nao cua lab cham duoc vao nodes/proxy'
```

**Ý nghĩa:** đường `/api/v1/nodes/<node>/proxy/...` mà bạn vừa dùng bằng quyền quản trị chính là
thứ bài [120](../120-rbac-good-practices-vi.md) cảnh báo. Nó là **bastion tích hợp sẵn trong
apiserver** theo mô tả của bài [359](../359-access-cluster-vi.md), và nó nối thẳng tới Kubelet
API. Ai có verb **get** trên `nodes/proxy` thì lấy được log và **thực thi lệnh trong mọi Pod**
của node đó, **bỏ qua cả audit logging lẫn admission control** — nên `get` ở đây **không phải
quyền chỉ đọc**. Đó là lý do ClusterRole của lab dừng lại ở `nodes`.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B6.5. Hai loại token đặt cạnh nhau

Bài [359](../359-access-cluster-vi.md) có nguyên một nhánh *Không dùng kubectl proxy* dựa trên
token dài hạn dạng Secret, còn bài [279](../279-configure-service-account-vi.md) thì khuyến cáo
tránh cách đó. Dựng lại để so bằng bằng chứng, rồi xóa ngay.

> Đây là credential **không hết hạn**. Nó tồn tại đúng trong bước này và bị xóa ở cuối bước.
> Không sao chép nó ra ngoài `~/lab-work/9a/`, không ghi vào bằng chứng.

```bash
kubectl create serviceaccount sa-cu -n lab-9a
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: sa-cu-token
  namespace: lab-9a
  annotations:
    kubernetes.io/service-account.name: sa-cu
type: kubernetes.io/service-account-token
EOF

i=0; TOK_CU=''
while [ "$i" -lt 60 ]; do
  TOK_CU="$(kubectl get secret sa-cu-token -n lab-9a -o jsonpath='{.data.token}' \
    | base64 -d 2>/dev/null)"
  test -n "$TOK_CU" && break
  i=$(( i + 1 )); sleep 1
done
echo "control plane dien token vao Secret sau $i vong lap"
```

```bash
test -n "$TOK_CU" \
  && echo 'PASS: control plane tu sinh token cho ServiceAccount va luu vao Secret'

{
  echo "sub cua token dai han = $(jwt_claim "$TOK_CU" sub)"
  echo "token dai han co claim exp khong:"
  jwt_pay "$TOK_CU" | grep -c '"exp"'
  echo "token ngan han co claim exp khong:"
  jwt_pay "$TOK_RO" | grep -c '"exp"'
} | tee ~/lab-evidence/9a/b6-hai-loai-token.txt

if jwt_pay "$TOK_CU" | grep -q '"exp"'; then
  echo 'FAIL: token dai han lai co claim exp — doc lai buoc nay'
else
  echo 'PASS: token dai han khong co claim exp, tuc khong bao gio het han'
fi
jwt_pay "$TOK_RO" | grep -q '"exp"' \
  && echo 'PASS: doi chung — token do TokenRequest phat hanh thi co exp'
test "$(api_code /api "$TOK_CU")" = '200' \
  && echo 'PASS: token dai han van xac thuc duoc binh thuong'
test -z "$(kubectl get serviceaccount sa-cu -n lab-9a -o jsonpath='{.secrets}')" \
  && echo 'PASS: Secret nay khong xuat hien trong truong .secrets cua ServiceAccount'
```

Xóa ServiceAccount và xem control plane dọn:

```bash
kubectl delete serviceaccount sa-cu -n lab-9a

i=0; C_CU=''
while [ "$i" -lt 60 ]; do
  C_CU="$(api_code /api "$TOK_CU")"
  test "$C_CU" = '401' && break
  i=$(( i + 1 )); sleep 1
done
echo "token dai han sau khi xoa SA -> $C_CU (sau $i vong lap)"

test "$C_CU" = '401' \
  && echo 'PASS: xoa ServiceAccount thi token dai han cung mat hieu luc'
kubectl delete secret sa-cu-token -n lab-9a --ignore-not-found
unset TOK_CU
```

**Ý nghĩa:** khác biệt nằm ở đúng một claim. Token do `TokenRequest` phát hành có `exp`, gắn với
object, tự xoay vòng, và chết ngay khi object biến mất. Token dạng Secret **không có `exp`** —
nó không bao giờ tự hết hạn, và nếu bị lộ thì nó dùng được cho tới khi có người nhớ ra phải xóa.
Đó là toàn bộ lý do bài [279](../279-configure-service-account-vi.md) xếp nó vào nhóm **không
khuyến nghị**, đặc biệt ở quy mô lớn.

Câu cuối là điểm dễ hiểu nhầm: trường `.secrets` của ServiceAccount **chỉ** được điền với các
Secret sinh **tự động**, nên một Secret bạn tạo tay không xuất hiện ở đó. Nghĩa là bạn không thể
rà token dài hạn bằng cách đọc object ServiceAccount — phải rà bằng loại Secret.

**PASS:** sáu dòng `PASS:` của bước này xuất hiện.

### B6.6. Ai xác minh được token này, và bằng gì

```bash
kubectl get --raw '/.well-known/openid-configuration' \
  > ~/lab-evidence/9a/b6-openid-configuration.json
kubectl get --raw '/openid/v1/jwks' > ~/lab-evidence/9a/b6-jwks.json

ISS_DOC="$(sed -n 's/.*"issuer":"\([^"]*\)".*/\1/p' \
  ~/lab-evidence/9a/b6-openid-configuration.json)"
ISS_TOK="$(jwt_claim "$TOK_RO" iss)"
DISC_ROLE="$(kubectl get clusterrole system:service-account-issuer-discovery \
  -o name 2>/dev/null)"
DISC_SUBJ="$(kubectl get clusterrolebinding system:service-account-issuer-discovery \
  -o jsonpath='{.subjects[*].name}' 2>/dev/null)"
{
  echo "issuer trong tai lieu discovery = $ISS_DOC"
  echo "claim iss trong token           = $ISS_TOK"
  echo "clusterrole discovery           = ${DISC_ROLE:-khong co}"
  echo "duoc gan cho                    = ${DISC_SUBJ:-khong co}"
} | tee ~/lab-evidence/9a/b6-issuer.txt
```

```bash
test -n "$ISS_DOC" && test "$ISS_DOC" = "$ISS_TOK" \
  && echo 'PASS: issuer cong bo trong tai lieu discovery khop voi claim iss cua token'
grep -q '"keys"' ~/lab-evidence/9a/b6-jwks.json \
  && echo 'PASS: cluster cong bo bo khoa cong khai de ben khac xac minh chu ky'
test -n "$DISC_ROLE" \
  && echo 'PASS: cluster co san ClusterRole cho viec kham pha issuer'
echo "$DISC_SUBJ" | grep -q 'system:serviceaccounts' \
  && echo 'PASS: role do duoc gan cho nhom chua moi ServiceAccount'
```

**Ý nghĩa:** hai tài liệu này là cách một **bên tin cậy** bên ngoài xác minh token của cluster mà
không cần hỏi API server từng lần: đọc `issuer`, lấy `jwks_uri`, rồi kiểm chữ ký bằng khóa công
khai. Bài [118](../118-service-accounts-vi.md) đặt cạnh đó một cảnh báo quan trọng — cách này
**không** phát hiện được token đã bị thu hồi, vì bên tin cậy vẫn coi token hợp lệ cho tới mốc hết
hạn. B2.6 đã cho thấy API server thì vô hiệu ngay. Đó là lý do dự án Kubernetes khuyến nghị
TokenReview API thay vì OIDC discovery khi bạn tự viết dịch vụ xác minh.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

---

## B7. Cleanup và gate cuối

**Mục đích:** xóa mọi thứ lab tạo ra — **kể cả object phạm vi cluster** — chứng minh cấu hình
control plane không hề bị sửa, chứng minh không token nào lọt vào thư mục bằng chứng, và chứng
minh cluster đã về đúng `03-storage-ready`.

### B7.1. Xóa object của bài học

```bash
kubectl delete namespace lab-9a lab-9a-khac --wait=true --timeout=300s
kubectl delete clusterrole,clusterrolebinding -l lab=9a --ignore-not-found
kubectl delete clusterrolebinding lab-9a-to-hop-sai --ignore-not-found
```

```bash
rm -f ~/lab-work/9a/admin-client.crt ~/lab-work/9a/ca-tu-configmap.crt \
      ~/lab-work/9a/sa-doc-cm.kubeconfig \
      ~/lab-work/9a/b3-role.yaml ~/lab-work/9a/b3-view-binding.yaml \
      ~/lab-work/9a/b3-clusterrole-node.yaml ~/lab-work/9a/b3-node-rolebinding.yaml \
      ~/lab-work/9a/b3-to-hop-sai.yaml ~/lab-work/9a/b3-binding-nhom.yaml \
      ~/lab-work/9a/b3-role-quan-ly-role.yaml ~/lab-work/9a/b3-role-hep.yaml \
      ~/lab-work/9a/b3-role-rong.yaml ~/lab-work/9a/b3-crb-subjects.txt \
      ~/lab-work/9a/b4-sa-khong-ton-tai.yaml ~/lab-work/9a/b4-pod-tran.yaml \
      ~/lab-work/9a/b4-pod-pullsecret.yaml \
      ~/lab-work/9a/b5-sa-khong-token.yaml ~/lab-work/9a/b5-hai-pod.yaml \
      ~/lab-work/9a/b5-pod-audience.yaml
rmdir ~/lab-work/9a

unset TOK_RO TOK_AUD TOK_NODE PAY_POD PAY_AUD KCFG
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến điều
đó thành gate thay vì im lặng bỏ qua. Nếu nó fail, **liệt kê thư mục trước khi xóa tay** — một
file còn sót ở đây rất có thể là file chứa token. Thư mục `~/lab-evidence/9a/` **giữ lại** — đó
là bằng chứng.

```bash
NS_LEFT="$(kubectl get namespace lab-9a lab-9a-khac --ignore-not-found -o name 2>/dev/null \
  | wc -l)"
LAB_OBJ="$(kubectl get clusterrole,clusterrolebinding -l lab=9a --no-headers 2>/dev/null \
  | wc -l)"
echo "namespace cua lab con = $NS_LEFT | object pham vi cluster mang label lab=9a = $LAB_OBJ"

test "$NS_LEFT" -eq 0 && echo 'PASS: hai namespace cua lab da bien mat'
test "$LAB_OBJ" -eq 0 && echo 'PASS: khong con ClusterRole hay ClusterRoleBinding nao cua lab'
test ! -e ~/lab-work/9a && echo 'PASS: thu muc chua manifest va token da xoa'
```

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B7.2. Gate quan trọng nhất về quyền: tồn kho RBAC trở về đúng con số cũ

```bash
CR_N9="$(kubectl get clusterrole --no-headers | wc -l)"
CRB_N9="$(kubectl get clusterrolebinding --no-headers | wc -l)"
SA_N9="$(kubectl get serviceaccount -A --no-headers | wc -l)"
NS_N9="$(kubectl get namespace --no-headers | wc -l)"
{
  echo "clusterrole:        truoc=$CR_N0  sau=$CR_N9"
  echo "clusterrolebinding: truoc=$CRB_N0 sau=$CRB_N9"
  echo "serviceaccount:     truoc=$SA_N0  sau=$SA_N9"
  echo "namespace:          truoc=$NS_N0  sau=$NS_N9"
} | tee ~/lab-evidence/9a/b7-ton-kho.txt

test "$CR_N9" -eq "$CR_N0" && test "$CRB_N9" -eq "$CRB_N0" \
  && echo 'PASS: so ClusterRole va ClusterRoleBinding tro ve dung con so truoc lab'
test "$SA_N9" -eq "$SA_N0" && test "$NS_N9" -eq "$NS_N0" \
  && echo 'PASS: so ServiceAccount va Namespace cung tro ve dung con so truoc lab'

kubectl get clusterrolebinding \
  -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.roleRef.name}{"\n"}{end}' \
  | grep -c '^lab-9a' > /dev/null; RC_LAB=$?
test "$RC_LAB" -ne 0 \
  && echo 'PASS: khong binding nao con mang tien to lab-9a'
```

**Ý nghĩa:** đây là gate mà một lab về quyền **bắt buộc** phải có. Xóa namespace là đủ để dọn
Role, RoleBinding và ServiceAccount — chúng thuộc namespace. Nhưng ClusterRole và
ClusterRoleBinding **không thuộc namespace nào**, nên chúng sống sót qua mọi lần `delete
namespace` và không ai nhận ra cho tới khi có sự cố. So **giá trị** với con số đọc ở B0.3 là cách
duy nhất chứng minh không còn gì.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B7.3. Cấu hình control plane và kubelet không đổi

```bash
{
  echo "apiserver-manifest $(sudo sha256sum /etc/kubernetes/manifests/kube-apiserver.yaml \
    | awk '{print $1}')"
  echo "kubelet-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in lab-k8s-worker1 lab-k8s-worker2; do
    echo "kubelet-$n $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
} | tee ~/lab-evidence/9a/b7-cauhinh-sha.txt

diff -u ~/lab-evidence/9a/b0-cauhinh-sha.txt ~/lab-evidence/9a/b7-cauhinh-sha.txt \
  && echo 'PASS: manifest apiserver va cau hinh kubelet ba node khong he doi trong suot lab' \
  || echo 'FAIL: cau hinh control plane da bi sua — xem muc 4'

RS_OK=0
for n in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  A="$(kubectl get node "$n" -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')"
  test "$A" = 'True' && RS_OK=$(( RS_OK + 1 ))
done
test "$RS_OK" -eq 3 && echo 'PASS: ca ba node van Ready sau khi lab ket thuc'
```

**Ý nghĩa:** cả lab này xoay quanh những thứ do cờ của `kube-apiserver` quyết định — module xác
thực, module phân quyền, danh sách admission controller, `anonymous-auth`, issuer của token. Cám
dỗ "sửa một cờ để xem module kia chạy thế nào" là có thật, và hậu quả của nó không dừng ở lab
này: mọi lab sau đều bắt đầu từ `03-storage-ready`. Gate này biến lời hứa ở
[mục 2](#2-quy-ước-và-an-toàn) thành thứ kiểm chứng được. Nếu `diff` báo khác, restore cả ba VM
trước khi sang lab sau.

**PASS:** hai dòng `PASS:` của bước này xuất hiện, không dòng `FAIL:` nào.

### B7.4. Gate về bí mật: không token nào lọt vào bằng chứng

```bash
grep -rl 'eyJhbGciOi' ~/lab-evidence/9a/ 2>/dev/null | tee ~/lab-evidence/9a/b7-quet-token.txt
LO="$(grep -rl 'eyJhbGciOi' ~/lab-evidence/9a/ 2>/dev/null | grep -vc 'b7-quet-token.txt' \
  || true)"
BEARER="$(grep -rl 'Authorization: Bearer ey' ~/lab-evidence/9a/ 2>/dev/null | wc -l)"
echo "file bang chung chua chuoi JWT = $LO | file chua header Bearer thuc = $BEARER"

test "$LO" -eq 0 && test "$BEARER" -eq 0 \
  && echo 'PASS: khong file bang chung nao chua token'
ls -l ~/lab-evidence/9a/ | tee -a ~/lab-evidence/9a/b7-quet-token.txt
```

**Ý nghĩa:** quy ước ở [mục 2](#2-quy-ước-và-an-toàn) nói bằng chứng chỉ chứa mã HTTP, thông báo
và **claim đã giải mã** — không bao giờ chứa chuỗi token. Đây là gate kiểm điều đó. Nếu nó fail,
xóa file vi phạm rồi chạy lại; và nhớ rằng thư mục bằng chứng là thứ người ta hay chép ra ngoài
để nộp báo cáo, nên một token nằm trong đó là một credential đã rời khỏi cluster.

**PASS:** dòng `PASS:` của bước này xuất hiện.

### B7.5. Gate ngắn A5.5 và kết thúc

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
kubectl get storageclass | grep -q '(default)' \
  && echo 'PASS: van con StorageClass mac dinh cua moc 03-storage-ready'

{
  echo "=== $(date -Is) — trang thai sau Lab 9a ==="
  kubectl get namespaces
  kubectl get clusterrole,clusterrolebinding -l lab=9a 2>&1
  kubectl get storageclass -o wide
  kubectl get pv
} | tee ~/lab-evidence/9a/b7-sau.txt

diff -u ~/lab-evidence/9a/b0-truoc.txt ~/lab-evidence/9a/b7-sau.txt \
  > ~/lab-evidence/9a/b7-diff.txt 2>&1 || true
```

**PASS:** ba node `Ready`; dòng `PASS: readyz ok` xuất hiện; lệnh field selector trả
`No resources found`; CoreDNS đủ replica `READY`; `default` không có Pod; dòng
`PASS: van con StorageClass mac dinh …` xuất hiện; danh sách namespace không còn `lab-9a` và
`lab-9a-khac`.

Cluster đã trở về `03-storage-ready`. **Lab 9a không tạo snapshot mới** — để ba VM nguyên trạng
đang chạy hoặc tắt tùy bạn; lab sau tự bật máy theo
[A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab).

---

## 3. Checkpoint 9a

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Sau khi TLS xong, một request đi qua **ba chặng** nào, theo đúng thứ tự? Với mỗi chặng, nói
      rõ nó từ chối **vì lý do gì**, trả **mã HTTP nào**, và bạn dựng lại tình huống đó bằng bước
      nào trong lab.
- [ ] Một client gửi request **không kèm credential nào** tới `/api` và nhận `403` chứ không phải
      `401`. Giải thích vì sao, và nói xem điều đó cho bạn biết gì về cấu hình của cluster. Với
      cùng client đó, `/version` trả `200` — vì sao hai đường dẫn lại khác nhau?
- [ ] Ứng dụng của bạn báo `401` khi gọi API server, nhưng token của nó vừa được cấp và
      ServiceAccount thì có đủ RoleBinding. Kể ra **ba** nguyên nhân khác nhau mà lab đã dựng lại
      được, và cách phân biệt chúng.
- [ ] Bạn có một ClusterRole `doc-cm` cho phép `get`/`list` trên ConfigMap. Mô tả kết quả khi gắn
      nó bằng **RoleBinding trong namespace `team-a`** so với khi gắn bằng **ClusterRoleBinding**.
      Rồi trả lời: có tổ hợp nào trong bốn tổ hợp là **không tồn tại** không, và request tạo nó
      chết ở bước nào?
- [ ] Bạn cấp cho một ServiceAccount `get` và `list` trên ConfigMap trong `team-a`. Liệt kê ít
      nhất bốn thứ nó **vẫn không làm được**, và với mỗi thứ nói rõ lý do là "Role không chứa" hay
      "binding không với tới".
- [ ] Đồng nghiệp muốn cấp `list` trên Secret cho một tài khoản giám sát vì "list chỉ ra tên, an
      toàn hơn get". Sai ở đâu? Bạn đã chứng minh bằng bước nào?
- [ ] Kubernetes **không có** object `User`. Vậy tại sao một RoleBinding gắn cho group
      `lab-9a-doc` lại có tác dụng, và cái tên `lab-9a-doc` đó đến từ đâu trên cluster kubeadm của
      bạn? Nêu một hệ quả vận hành nguy hiểm của việc binding chỉ lưu tên.
- [ ] Bạn `kubectl apply` một Pod và bị `Forbidden`, nhưng `kubectl auth can-i create pods` trả
      `yes`. Chuyện gì đang xảy ra? Cùng lúc đó, `kubectl get pods` trong một namespace **không
      tồn tại** lại chạy trơn tru — vì sao hai request này khác nhau?
- [ ] Kể ba thứ mà chặng 3 **thêm vào** một Pod bạn viết chỉ với `image` và `command`. Ai làm
      việc đó, và file manifest trên đĩa của bạn có đổi không?
- [ ] Một Pod không khai `spec.serviceAccountName` chạy trong `team-a`. Nó gọi API server dưới
      danh tính nào? Nó làm được gì và không làm được gì? Trên `lab-k8s-worker2`, ba file nào được
      chiếu vào container đó, ở đường dẫn nào, và bằng cơ chế gì?
- [ ] `kubectl proxy` làm hộ bạn việc gì và **không** làm hộ việc gì? Nếu bạn chạy nó bằng
      kubeconfig của một ServiceAccount chỉ đọc ConfigMap, người gọi vào cổng proxy đó làm được
      những gì?
- [ ] Đặt cạnh nhau hai token của cùng một ServiceAccount: một do `TokenRequest` phát hành, một
      lưu trong Secret dạng `kubernetes.io/service-account-token`. Khác nhau ở claim nào, ở vòng
      đời nào, và vì sao dự án Kubernetes khuyến cáo tránh loại thứ hai?

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Luồng một request `kubectl create -f pod.yaml` đi từ máy bạn tới etcd.** Bắt đầu từ lúc
   TCP mở tới cổng 6443. Kể đủ: cái gì được xuất trình ở tầng TLS và ai kiểm nó; chặng 1 rút ra
   những gì từ thứ đó và ghi vào đâu; chặng 2 nhận vào đúng ba thứ nào và trả lời thế nào khi có
   nhiều module; chặng 3 nhìn thấy thứ gì mà hai chặng trước không thấy, và quy tắc kết hợp nhiều
   module của nó **ngược** với chặng 2 ra sao; cuối cùng bước 4 làm gì trước khi ghi. Với mỗi
   chặng, nêu **một** cách bạn đã làm request đó chết ở đúng chặng đó, và **dấu hiệu** để nhận ra
   từ phía client.
2. **Luồng một Pod trở thành một chủ thể có quyền.** Bắt đầu từ một manifest chỉ có `image`. Kể
   đủ: ai gán danh tính cho nó và ở chặng nào; credential vào trong container bằng cơ chế gì và
   gồm mấy phần; container gọi API server bằng đường nào và tự xác minh server bằng gì; API
   server rút danh tính ra từ đâu trong token đó. Rồi kể tiếp phía quản trị: bạn thêm những object
   nào để nó đọc được ConfigMap trong đúng một namespace, vì sao **không** nên gán quyền đó cho
   `default`, và ba cách bạn kiểm chứng lại kết quả mà không cần chạy Pod. Kết thúc bằng câu hỏi
   ngược: nếu ngày mai bạn xóa ServiceAccount đó, chuyện gì xảy ra với token đang nằm trong Pod,
   và vì sao đó là một tính chất chứ không phải một sự cố.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm **401** với **403**, **xác thực** với **phân
quyền**, **Role** với **ClusterRole**, **RoleBinding** với **ClusterRoleBinding**, hay
**`Forbidden` của chặng 2** với **`Forbidden` của chặng 3** — phần truy cập của
[giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy) hoàn tất, và bạn
sang được phần policy ở Lab 9b.

Lab này **không phát sinh nợ mới** và **không trả nợ nào**; xem
[sổ nợ lab](README.md#5-sổ-nợ-lab). Riêng **nợ #6 — mã hóa Secret at rest** có liên quan trực
tiếp tới điều B3.4 vừa chứng minh: Secret API chỉ là **sự bảo vệ cơ bản**, và nợ đó vẫn **chưa
trả** — nó được trả ở [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu),
không phải ở đây. Những phần cố ý không làm khác đều nằm trong bảng lý do ở
[mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ liệt
kê sự cố phát sinh từ nội dung bài học 9a.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| B0.2: `curl` báo lỗi certificate dù đã truyền `--cacert` | `sudo openssl x509 -noout -subject -issuer -in /etc/kubernetes/pki/ca.crt`; `echo "$SRV"` | `SRV` có thể là hostname không nằm trong SAN của certificate API server. Dùng đúng giá trị `kubectl config view` in ra, đừng thay bằng IP tay |
| B2.1: `client-certificate-data` rỗng | `kubectl config view --raw --minify -o jsonpath='{.users[0].user}'` | Kubeconfig của bạn dùng đường dẫn file (`client-certificate`) thay vì dữ liệu nhúng. Đọc thẳng file đó bằng `openssl x509 -in <duong-dan>`; các gate còn lại giữ nguyên |
| B2.1: `CN_CERT` rỗng hoặc khác `ADMIN_USER` | `openssl x509 -noout -subject -in ~/lab-work/9a/admin-client.crt` | Định dạng subject của `openssl` khác nhau theo phiên bản. Đọc bằng mắt rồi so tay; **không** sửa cluster để làm gate xanh |
| B2.3: `C_AN_API` không khớp giá trị kỳ vọng | `sudo grep -n 'anonymous-auth' /etc/kubernetes/manifests/kube-apiserver.yaml` | Cluster của bạn cấu hình `anonymous-auth` khác baseline. Ghi giá trị thật vào evidence và đọc lại phần *Ý nghĩa* theo cấu hình đó. **Không sửa manifest** — đó là nội dung giai đoạn 20 |
| B2.6 hoặc B6.5: vòng lặp chạy hết 60 vòng mà mã vẫn `200` | `kubectl get serviceaccount -n lab-9a`; `kubectl -n kube-system logs -l component=kube-apiserver --tail=50` | ServiceAccount chưa thực sự bị xóa, hoặc thông tin chưa lan tới authenticator. Kiểm object đã biến mất chưa rồi tăng số vòng lặp. Thời gian lan truyền **phụ thuộc cấu hình**; không kết luận cơ chế hỏng chỉ vì một ngưỡng cố định |
| B3.2: vòng lặp không bao giờ đạt `200` | `kubectl describe rolebinding doc-configmap -n lab-9a`; `kubectl auth can-i list configmaps -n lab-9a --as="$SA_RO"` | Sai `subjects[0].namespace` trong RoleBinding (phải là `lab-9a`), hoặc `TOK_RO` thuộc phiên shell cũ. Tạo lại token bằng `kubectl create token sa-doc-cm -n lab-9a` rồi chạy lại gate |
| B3.6: `kubectl auth can-i ... --subresource=proxy` báo cờ không tồn tại | `kubectl auth can-i --help` | Bản `kubectl` đang dùng cũ hơn baseline. Kiểm lại version theo [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); **không** cài `kubectl` khác để đi vòng |
| B3.10: Role rộng lại tạo được | `kubectl auth can-i --list -n lab-9a --as="$SA_RBAC"` | `sa-rbac` đang có nhiều quyền hơn dự kiến — thường vì có binding sót từ lần chạy trước. Xóa namespace `lab-9a`, chạy lại từ B0.1 |
| B4.1: thông báo không chứa `error looking up service account` | `cat ~/lab-evidence/9a/b4-admission-tu-choi.txt` | Nếu thông báo là `namespaces "lab-9a" not found` thì namespace đã bị xóa; tạo lại rồi chạy lại từ B3.2. Nếu là lỗi RBAC, bạn đang chạy bằng kubeconfig không phải quản trị |
| B4.3: Pod kẹt `Pending` | `kubectl describe pod tran -n lab-9a`; `kubectl get nodes -o custom-columns='NODE:.metadata.name,TAINTS:.spec.taints'` | `lab-k8s-worker2` mang taint hoặc `NotReady`. Chạy lại gate mở đầu; nếu worker2 hỏng thật thì restore cả ba VM — **không** đổi `nodeSelector` sang worker1, vì quy ước lab ghim mọi thao tác đọc trong container vào worker2 |
| B5: `kubectl exec` chạy `wget` báo lỗi bắt tay TLS | `kubectl exec -n lab-9a tran -- wget --help 2>&1 \| head -5` | Bản `busybox` của bạn không hoàn tất được TLS với API server. Chạy đúng request đó từ `lab-k8s-master`: đọc token bằng `TOK_POD="$(kubectl exec -n lab-9a tran -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)"` rồi `api_code`/`api_body` với `$TOK_POD`, và `unset TOK_POD` ngay sau đó. Gate giữ nguyên: mã `200` cho discovery, `403` cho ConfigMap. **Không cài thêm image nào** |
| B5.2: `H_POD` khác `H_CM` | `kubectl exec -n lab-9a tran -- md5sum /var/run/secrets/kubernetes.io/serviceaccount/ca.crt` | Thường do khác một ký tự xuống dòng ở cuối. Dùng gate còn lại — so **fingerprint** của certificate bằng `openssl` — làm bằng chứng chính, và ghi cả hai giá trị vào evidence |
| B5.4: Pod `khong-token` vẫn có thư mục credential | `kubectl get pod khong-token -n lab-9a -o jsonpath='{.spec.volumes}'` | Pod được tạo **trước** khi ServiceAccount có `automountServiceAccountToken: false`. Xóa Pod, apply lại — quyết định này xảy ra ở chặng admission, tức lúc tạo Pod |
| B6.2: `kubectl proxy` không lên, vòng lặp trả `000` | `cat ~/lab-evidence/9a/b6-proxy-admin.log`; `ss -ltn \| grep 8001` | Cổng đã bận hoặc tiến trình proxy cũ còn sống. `kill` tiến trình cũ rồi chạy lại; nếu cần, đổi `--port` ở **tất cả** lệnh của B6.2 và B6.3 |
| B6.5: `.data.token` mãi rỗng | `kubectl describe secret sa-cu-token -n lab-9a`; `kubectl get sa sa-cu -n lab-9a` | Annotation `kubernetes.io/service-account.name` sai tên, hoặc ServiceAccount chưa tồn tại lúc tạo Secret. Xóa Secret, tạo lại ServiceAccount trước rồi mới tạo Secret |
| B7.1: namespace kẹt `Terminating` | `kubectl get pods -n lab-9a`; `kubectl describe namespace lab-9a` | Pod còn trong grace period. Chờ; **không** cưỡng chế finalizer của Namespace |
| B7.2: số ClusterRole hoặc ClusterRoleBinding không khớp | `kubectl get clusterrole,clusterrolebinding --sort-by=.metadata.creationTimestamp \| tail -20` | Còn object của lab không mang label `lab=9a`, hoặc bạn đã tạo thêm ngoài kịch bản. Tìm theo thời điểm tạo, xóa đúng cái đó. **Không** xóa ClusterRole nào có tiền tố `system:` |
| B7.3: `diff` báo checksum khác | `diff -u ~/lab-evidence/9a/b0-cauhinh-sha.txt ~/lab-evidence/9a/b7-cauhinh-sha.txt` | Cấu hình control plane đã bị sửa trong lúc chạy lab. Cluster lệch baseline: tắt cả ba VM, restore cả ba về `03-storage-ready`, và **không** sang lab sau trước khi gate này PASS |
| B7.4: có file bằng chứng chứa token | `grep -rl 'eyJhbGciOi' ~/lab-evidence/9a/` | Một lệnh đã `tee` cả token vào evidence. Xóa file đó, chạy lại đúng bước đã sinh ra nó theo phiên bản trong bài (chỉ ghi claim, không ghi token) |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Security](https://v1-35.docs.kubernetes.io/docs/concepts/security/)
- [Kubernetes v1.35 — Cloud Native Security and Kubernetes](https://v1-35.docs.kubernetes.io/docs/concepts/security/cloud-native-security/)
- [Kubernetes v1.35 — Service Accounts](https://v1-35.docs.kubernetes.io/docs/concepts/security/service-accounts/)
- [Kubernetes v1.35 — Controlling Access to the Kubernetes API](https://v1-35.docs.kubernetes.io/docs/concepts/security/controlling-access/)
- [Kubernetes v1.35 — Role Based Access Control Good Practices](https://v1-35.docs.kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Kubernetes v1.35 — Configure Service Accounts for Pods](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/)
- [Kubernetes v1.35 — Accessing the Kubernetes API from a Pod](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/access-api-from-pod/)
- [Kubernetes v1.35 — Accessing Clusters](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/access-cluster/)
- [Kubernetes v1.35 — Authenticating](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes v1.35 — Authorization](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [Kubernetes v1.35 — Using RBAC Authorization](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes v1.35 — Admission Controllers Reference](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Kubernetes v1.35 — Managing Service Accounts (administration)](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
- [Kubernetes v1.35 — TokenRequest API reference](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/)
- [Kubernetes v1.35 — SubjectAccessReview API reference](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/authorization-resources/subject-access-review-v1/)
