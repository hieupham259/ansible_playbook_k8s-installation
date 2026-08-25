# Lab 14 — CRD và Operator

> **Điểm bắt đầu:** snapshot `04-metrics-ready` — mốc do Lab 11a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot).
> **Điểm kết thúc:** cleanup trả cluster về đúng `04-metrics-ready`, **không tạo snapshot mới**.
> Lab này không cài thêm bất kỳ thành phần hạ tầng nào, không cài operator framework, không cài
> toolchain lập trình, và không sửa cấu hình node nào.
> **Lab trước:** [Lab 13 — DRA](LAB-13-DRA.md) (tùy chọn) cũng trả cluster về `04-metrics-ready`;
> bỏ qua lab 13 thì lab trước là [Lab 12 — Vận hành vòng đời node](LAB-12-VAN-HANH-VONG-DOI-NODE.md).
> Cluster vào lab này phải ở đúng mốc đó: có CNI thực thi
> NetworkPolicy, ingress controller, StorageClass mặc định và metrics-server; không workload.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng mục
[Giai đoạn 14 — Khả năng mở rộng](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng)
— bảy bài [177](../177-extend-kubernetes-vi.md), [178](../178-api-extension-vi.md),
[179](../179-custom-resources-vi.md), [180](../180-apiserver-aggregation-vi.md),
[181](../181-operator-vi.md), [182](../182-compute-storage-net-vi.md),
[184](../184-device-plugins-vi.md), cộng hai bài thực hành
[313](../313-debug-topology-vi.md) và [323](../323-storage-version-migration-vi.md).

**Xương sống của lab là một vòng đời CRD đầy đủ**: tạo `CustomResourceDefinition`, apply custom
resource, siết schema, mở subresource `status`, rồi tự tay đóng vai controller để chứng minh câu
mà [checkpoint của giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) bắt bạn
giải thích được: **CRD không có controller thì chỉ là kho lưu dữ liệu**. Ba mục còn lại là phần
**đọc bản đồ điểm mở rộng trên chính cluster của bạn** — bốn điểm mở rộng bạn đã dùng từ Lab 5b,
6a và 11a mà chưa gọi tên chúng.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không
chép lại con số phiên bản nào** và cũng **không cần con số nào**, vì nó không cài gì. Thành phần
ngoài baseline mà lab thấy đang chạy — CNI thay Flannel, ingress controller, dynamic provisioner,
metrics-server — đã được khóa ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) và do Lab 5b, Lab 6a,
Lab 11a cài; lab 14 chỉ **đọc** và **phân loại** chúng, không đụng vào.

Quan hệ với ba lab đã học, cả ba đều được dùng lại nguyên vẹn:

- [Lab 1c](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md#b2-owner-reference-và-garbage-collection)
  đã dạy `ownerReferences` và garbage collection. B8 gắn owner reference từ ConfigMap lên custom
  resource, rồi chứng minh phần dọn dẹp đó **không phải công của controller**.
- [Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md#b3-chặng-2--phân-quyền-và-rbac) đã dạy
  ServiceAccount, Role, ClusterRole và `kubectl auth can-i`. B6 và B8 dùng lại y hệt, lần này cho
  một kind **do bạn định nghĩa**.
- [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md#b92-gate-quan-trọng-nhất-của-lab-cấu-hình-node-không-đổi)
  đã dạy cách chụp checksum cấu hình ở B0 rồi `diff` ở gate cuối. B0.4 và B11.4 làm đúng như vậy.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`, rồi thêm ba lệnh riêng của lab này:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

# Ba lenh rieng cua lab 14: dung moc 04-metrics-ready, va cluster chua co dau vet cua lab nay.
kubectl top node
kubectl get crd --no-headers | wc -l
kubectl get crd,clusterrole,clusterrolebinding -l lab=14
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod; **`kubectl top node` in đủ ba dòng số
liệu** (đó là định nghĩa của mốc `04-metrics-ready`); lệnh đếm CRD in ra một số **lớn hơn 0** — nếu
nó in `0` thì cluster của bạn không ở mốc đầu vào, vì cả Lab 5b lẫn Lab 6a đều cài thành phần mang
theo CRD; lệnh cuối trả `No resources found`.

Nếu `kubectl top node` báo lỗi, hoặc lệnh cuối trả về object nào, thì cluster đã lệch — restore cả
ba VM về `04-metrics-ready` trước khi tiếp tục. B11 dựa hẳn vào việc **chưa có object cluster-scoped
nào mang label `lab=14`** ở thời điểm bắt đầu.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Bảy điểm mở rộng của bài [177](../177-extend-kubernetes-vi.md), và với mỗi điểm chỉ ra được
  **một hiện vật có thật** trong cluster của mình — hoặc chứng minh được điểm đó đang trống.
- Cluster của bạn đã dùng **bốn điểm mở rộng khác nhau** từ trước lab này, và gọi đúng tên từng
  cái: network plugin, tầng tổng hợp API, custom resource, và mẫu controller.
- Ranh giới **webhook so với binary plugin**: cái nào là request mạng, cái nào là chương trình
  nhị phân do kubelet thực thi, và mỗi loại đang tồn tại ở đâu trên cluster này.
- Có **đúng hai cách** thêm custom resource, và trên cluster này đọc ra được cả hai đang chạy song
  song: một API group do kube-apiserver tự phục vụ, một API group được proxy sang server khác.
- Tạo một CRD, apply một custom resource và đọc lại bằng `kubectl get` — **đúng checkpoint của
  giai đoạn 14** — kèm giải thích endpoint REST mới nằm ở đường dẫn nào và ai phục vụ nó.
- Schema OpenAPI trong CRD là một **hợp đồng được API server cưỡng chế**: object sai bị từ chối,
  đọc đúng câu từ chối, và trường lạ bị **cắt tỉa** chứ không được lưu.
- `shortNames`, `categories` và `additionalPrinterColumns` đổi **bề mặt CLI** của kind mới ra sao,
  và vì sao chúng là thuộc tính của server chứ không phải của `kubectl`.
- Khác biệt thật giữa `scope: Namespaced` và `scope: Cluster`, chứng minh bằng thực nghiệm, cộng
  hệ quả nguy hiểm nhất của nó: **xóa namespace không dọn object cluster-scoped**.
- Vì sao một ServiceAccount mang ClusterRole `view` dựng sẵn **vẫn không đọc được** kind mới, và
  vì sao phải cấp quyền tường minh cho API group mới.
- Subresource `status` đổi điều gì: trước khi bật, `status` chỉ là một trường ai cũng ghi được;
  sau khi bật, **người dùng ghi `spec`, controller ghi `status`**, và `metadata.generation` chỉ
  tăng theo `spec`.
- **CRD không có controller thì chỉ là kho lưu dữ liệu**: chứng minh bằng một custom resource
  không làm gì cả, rồi bằng một vòng lặp điều khiển thủ công biến chính dữ liệu đó thành hành vi.
- Ba tính chất của một vòng lặp điều khiển, chứng minh riêng từng cái: **tạo** object phụ thuộc,
  **hội tụ** khi `spec` đổi, và **tự chữa** khi object phụ thuộc bị xóa.
- Phần nào của "operator" **không** phải công của vòng lặp: garbage collection theo
  `ownerReferences` là công của API machinery, không phải của controller.
- `status.storedVersions` là gì, vì sao API server **từ chối** gỡ một version còn nằm trong đó, và
  chạy trọn quy trình nâng cấp storage version thủ công của bài
  [377](../377-custom-resource-definition-versioning-vi.md).
- Vì sao cluster lab **không** có device plugin nào, đọc đúng bằng chứng cho điều đó, và nói được
  device plugin khác DRA của giai đoạn 13 ở điểm nào.
- Cleanup đúng phạm vi: namespace biến mất **không** kéo theo CRD và ClusterRole, và số CRD,
  ClusterRole, ClusterRoleBinding, APIService trở về **đúng con số cũ**.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 14 | Phần lab kiểm chứng |
| --- | --- |
| [177 — Mở rộng Kubernetes](../177-extend-kubernetes-vi.md) | B1 — dựng lại bảy điểm mở rộng của mục *Chú giải cho hình vẽ* bằng lệnh, tìm hiện vật cho từng điểm trên cluster thật; B1.3 phân loại webhook so với binary plugin; B1.4 chốt bốn điểm mở rộng cluster đã dùng từ trước lab này |
| [182 — Các phần mở rộng về Tính toán, Lưu trữ và Mạng](../182-compute-storage-net-vi.md) | B1.2 — ba nhóm hạ tầng đối chiếu với add-on thật: provisioner của StorageClass **không** phải CSI driver nên nó nằm ở điểm mở rộng *controller* chứ không phải *storage plugin*; CNI là nhóm duy nhất thiếu thì cluster không chạy |
| [178 — Mở rộng Kubernetes API](../178-api-extension-vi.md) | B2 — đúng hai cách, đo trên cluster thật: đếm APIService do kube-apiserver tự phục vụ so với APIService được proxy sang một Service |
| [180 — Tầng tổng hợp API](../180-apiserver-aggregation-vi.md) | B2.2 — `v1beta1.metrics.k8s.io` là APIService duy nhất có `.spec.service`, gọi thẳng `/apis/metrics.k8s.io/v1beta1` để thấy đường proxy; B3.5 — API group của CRD cũng có APIService nhưng `.spec.service` **rỗng**, tức không có API server thứ hai nào |
| [179 — Tài nguyên tùy chỉnh](../179-custom-resources-vi.md) | B3 (tạo CRD, apply CR, đọc lại — checkpoint giai đoạn 14), B4 (validation và cắt tỉa trường), B5 (`shortNames`, `categories`, `additionalPrinterColumns`), B6 (`scope` và RBAC cho kind mới), B7 (subresource `status`), B9 (multi-versioning và storage version) — sáu mục ứng với sáu ô "Có" của cột CRD trong bảng *Tính năng nâng cao và tính linh hoạt* |
| [181 — Mẫu Operator](../181-operator-vi.md) | B8 — custom resource một mình không làm gì; rồi một vòng lặp điều khiển thủ công chạy bằng ServiceAccount có quyền tối thiểu, chứng minh ba tính chất tạo/hội tụ/tự chữa; B8.6 ánh xạ từng bước sang bảy bước của ví dụ `SampleDB` |
| [184 — Device Plugin](../184-device-plugins-vi.md) | B10.1–B10.2 — `Capacity`/`Allocatable` của ba node không có resource nào dạng `vendor-domain/resourcetype`; socket đăng ký `kubelet.sock` tồn tại nhưng không plugin nào đăng ký; B10.3 đối chiếu với DRA của giai đoạn 13 |
| [313 — Khắc phục sự cố Topology Management](../313-debug-topology-vi.md) | B10.4 — đọc `topologyManagerPolicy` và `memoryManagerPolicy` **hiệu lực** của cả ba kubelet qua `configz`, cộng số NUMA node thật của máy, để chứng minh nhánh debug của bài không thể kích hoạt trên cluster này |
| [323 — Storage Version Migration](../323-storage-version-migration-vi.md) | B9 — `status.storedVersions` của một CRD thật, API server **từ chối** gỡ version còn nằm trong `storedVersions`, và quy trình nâng cấp thủ công ba bước của bài [377](../377-custom-resource-definition-versioning-vi.md) chạy trọn bằng `kubectl` |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [181](../181-operator-vi.md) — viết một operator thật bằng Go, kubebuilder, Kopf hay Operator Framework | Cần cài toolchain ngôn ngữ và framework lên node lab, tức thêm phần mềm ngoài baseline. B8 làm phần **cơ chế** bằng công cụ có sẵn: vòng lặp `kubectl` chạy dưới danh tính một ServiceAccount có quyền tối thiểu. Khác biệt so với operator thật nằm ở **nơi chạy và cách kích hoạt** (một Deployment trong cluster, `watch` thay cho vòng lặp), không phải ở bản chất vòng lặp — B8.6 liệt kê từng khác biệt |
| Bài [180](../180-apiserver-aggregation-vi.md) — dựng một extension API server của riêng bạn | Phải viết và build binary cùng image, rồi cấu hình cụm cờ `--requestheader-*` cho kube-apiserver. Đó là bài [374](../374-configure-aggregation-layer-vi.md) và [380](../380-setup-extension-api-server-vi.md) của [giai đoạn 28](../00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes). B2 kiểm chứng **phần đọc được** trên một extension API server đang chạy thật là metrics-server |
| Bài [179](../179-custom-resources-vi.md) — conversion webhook, validating/mutating webhook cho custom resource | Webhook đòi một dịch vụ HTTPS trong cluster kèm certificate và `caBundle`. Dựng nó là dựng thêm hạ tầng, và bản thân admission webhook thuộc [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy), bài [173](../173-admission-webhooks-vi.md). B9 dùng `conversion.strategy: None` — đúng trường hợp bài [377](../377-custom-resource-definition-versioning-vi.md) mô tả là an toàn khi mọi version dùng chung một schema |
| Bài [184](../184-device-plugins-vi.md) — chạy một device plugin thật và cấp GPU cho Pod | Cluster lab là ba VM không có GPU, NIC hiệu năng cao hay FPGA. Đây là giới hạn **phần cứng**, không phải thiếu cấu hình. B10 kiểm chứng phần đọc được, còn cơ chế **extended resource** mà device plugin dùng để công bố thiết bị thì bạn đã làm bằng tay ở [Lab 3c phần B5](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md#b5-extended-resource-tài-nguyên-do-bạn-tự-đặt-tên) |
| Bài [184](../184-device-plugins-vi.md) — ba endpoint gRPC `List`, `GetAllocatableResources`, `Get` của Pod Resources API | Phải viết một client gRPC nói đúng protobuf của kubelet; không công cụ nào trong baseline làm được. Chính bài xếp phần này vào mục đọc lướt |
| Bài [313](../313-debug-topology-vi.md) — dựng lỗi `TopologyAffinityError` và đọc `numaAffinity` trong `/var/lib/kubelet/memory_manager_state` | Chỉ xuất hiện khi Topology Manager chạy policy khác `none` **và** máy có nhiều NUMA domain. Bật policy khác là sửa `KubeletConfiguration` trên node đang chạy — thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), bài [224](../224-kubelet-config-file-vi.md); và VM một NUMA node thì không có ranh giới nào để căn. **Lab 14 tuyệt đối không sửa cấu hình kubelet** — B11.4 chứng minh điều đó bằng checksum |
| Bài [323](../323-storage-version-migration-vi.md) — object `StorageVersionMigration` và luồng mã hóa lại Secret | Cần bật feature gate `StorageVersionMigrator` và runtime config `storagemigration.k8s.io/v1beta1` trên API server, cộng một `EncryptionConfiguration`. Cả hai là sửa control plane: cái đầu thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy), cái sau thuộc [nợ #6](README.md#5-sổ-nợ-lab) trả ở [giai đoạn 22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu). B9 chạy **phương án 2 của bài [377](../377-custom-resource-definition-versioning-vi.md)** — nâng cấp thủ công — vốn không cần feature gate nào |
| Bài [323](../323-storage-version-migration-vi.md) — đọc thẳng etcd bằng `etcdctl … \| hexdump -C` để thấy định dạng byte của object cũ | Cần đối số kết nối etcd và quy trình vận hành etcd của [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa), bài [197](../197-configure-upgrade-etcd-vi.md). B9 chứng minh cùng luận điểm từ phía API bằng `status.storedVersions` — thứ chính API server ghi ra để trả lời đúng câu hỏi đó |
| Bài [179](../179-custom-resources-vi.md) — `selectableFields` và field selector cho custom resource; subresource `scale` | Cả hai là tinh chỉnh sau khi CRD đã chạy. Riêng `scale` chỉ có ý nghĩa khi ghép với HorizontalPodAutoscaler, và HPA là nội dung của [Lab 11b](LAB-11B-HPA-VA-VPA.md) — lab 14 không lặp lại lab khác. B5 dừng ở phần bề mặt CLI mà mọi CRD đều dùng |

Đây **không phải nợ lộ trình mới**. Toàn bộ phần cố ý không làm ở trên hoặc là giới hạn phần cứng
của môi trường lab, hoặc là nội dung đã có chỗ đứng ở giai đoạn sau —
[giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy),
[22](../00-ALO-TRINH-ADMIN.md#giai-đoạn-22--audit-và-mã-hóa-dữ-liệu) và
[28](../00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes). Lab 14 **không phát sinh nợ mới**
và **không trả nợ nào**; xem [sổ nợ lab](README.md#5-sổ-nợ-lab).

Lab này **không** thay thế bài [378](../378-custom-resource-definitions-vi.md) — trang xương sống
về cú pháp CRD, đọc ở [giai đoạn 28](../00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes),
vốn là nhóm thực hành của chính giai đoạn 14. Ở đây bạn chỉ dùng đúng những trường mà bài
[179](../179-custom-resources-vi.md) đã nêu tên trong hai bảng so sánh và trong ví dụ YAML của
chính nó; mọi tinh chỉnh sâu hơn để dành cho giai đoạn 28.

### 1.2. Thời lượng

2–3 giờ, tính từ lúc gate mở đầu đã PASS. Phần lớn thời gian nằm ở B3–B9; B1, B2 và B10 chỉ đọc.

Lab có ba chỗ phải chờ: CRD chuyển sang `Established`, discovery của `kubectl` nhận kiểu mới, và
APIService tự đăng ký biến mất sau khi xóa CRD. **Cả ba đều phụ thuộc cấu hình cluster**, nên
chúng được viết dưới dạng `kubectl wait --timeout` hoặc vòng lặp có điều kiện dừng, không phải một
con số giây. Riêng discovery của `kubectl` là **cache phía client**, xử lý bằng cách làm mới cache
chứ không phải bằng cách chờ — xem B3.2 và [mục 4](#4-troubleshooting-của-lab-này).

---

## 2. Quy ước và an toàn

> **Cảnh báo — lab này tạo object phạm vi cluster.** `CustomResourceDefinition`, `ClusterRole` và
> `ClusterRoleBinding` **không thuộc namespace nào**. Xóa namespace `lab-14` sẽ **không** dọn
> chúng. Đây là bẫy lớn nhất của giai đoạn 14 và cũng là lý do B11 tồn tại: nó xóa theo **label**
> rồi **so số lượng trước và sau** để bắt object sót lại.

**Bảng object phạm vi cluster mà lab này tạo ra.** Không object cluster-scoped nào khác được tạo;
B11.2 kiểm chứng đúng bảng này:

| Tên | Kind | Tạo ở | Xóa ở | Label |
| --- | --- | --- | --- | --- |
| `webpages.lab14.example.com` | CustomResourceDefinition | B3.1 | B11.2 | `lab=14` |
| `clusterwidgets.lab14.example.com` | CustomResourceDefinition | B6.4 | B11.2 | `lab=14` |
| `lab-14-doc-clusterwidget` | ClusterRole | B6.5 | B11.2 | `lab=14` |
| `lab-14-doc-clusterwidget` | ClusterRoleBinding | B6.5 | B11.2 | `lab=14` |

Ngoài bảng trên, kube-apiserver **tự** tạo thêm object `APIService` cho mỗi API group của CRD
(B2.3 giải thích cơ chế). Bạn không tạo và cũng không được xóa chúng bằng tay: chúng biến mất theo
CRD. B11.3 chờ và đối chiếu số lượng.

> **Cảnh báo thứ hai — không bao giờ xóa một CRD bạn không tạo.** Cluster ở mốc `04-metrics-ready`
> đã có sẵn CRD của CNI và của ingress controller. Xóa một CRD sẽ **xóa toàn bộ custom object
> đang lưu trong đó** (bài [378](../378-custom-resource-definitions-vi.md), mục *Xóa một
> CustomResourceDefinition*) — với CRD của CNI, đó là xóa cấu hình mạng của cluster. Mọi lệnh xóa
> CRD trong lab này đều đi qua selector `-l lab=14` và có gate đếm **trước khi** xóa.

Các quy ước còn lại:

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, **trừ khi ghi rõ
  node khác**. Lệnh cần đọc file trên worker chạy qua `ssh` từ master, đúng cách
  [Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md) và [Lab 12](LAB-12-VAN-HANH-VONG-DOI-NODE.md)
  đã dùng.
- **Lab này không có fault injection.** Không node nào bị tắt, không tiến trình nào bị dừng. Thao
  tác duy nhất chạm tới node là **đọc** hai đường dẫn ở B10, và chúng chỉ đọc trên
  `lab-k8s-worker2` — giữ đúng quy ước "chỉ đụng vào worker2" của
  [mục 6 trong README](README.md#6-quy-ước-chung-trong-mọi-lab).
- **Lab này chỉ ĐỌC cấu hình node và cấu hình control plane.** Tuyệt đối không sửa
  `/var/lib/kubelet/config.yaml`, không sửa file nào trong `/etc/kubernetes/manifests`, không đổi
  feature gate, không restart kubelet. B0.4 ghi checksum của sáu file đó và B11.4 đối chiếu lại.
- **Lab không cài thêm gì.** Không Helm chart, không operator framework, không image mới, không
  binary mới. Vì thế lab **không cần con số phiên bản nào** và không chép lại bảng
  [A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) hay
  [A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00). Toàn bộ phần B chạy bằng
  `kubectl`, `curl`, `ssh` và shell có sẵn trên node.
- **Lab không tạo image và không chạy container nào của riêng nó.** Custom resource của lab được
  vật chất hóa thành ConfigMap chứ không phải Deployment — B8.6 nói rõ vì sao lựa chọn đó không
  làm mất bài học.
- Kiến thức được phép dùng làm công cụ: toàn bộ giai đoạn 1–13, gồm ConfigMap, ServiceAccount,
  Role/ClusterRole, `ownerReferences`, `kubectl auth can-i`, `kubectl create token`, metrics-server
  và StorageClass. Giai đoạn 14 là gần cuối Phần I nên **không có món nào phải hoãn vì chưa học**.
- **Mọi con số của cluster đều đọc từ cluster thật** — số CRD, số ClusterRole, số APIService, tên
  StorageClass, tên provisioner, số NUMA node. Không con số nào của cluster viết cứng vào gate.
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Gần như mọi gate so sánh với biến đặt ở B0
  (`WK`, `EV`, `NS`, `LAB_GROUP`, `CRD_N0`, `CR_N0`, `CRB_N0`, `APISVC_N0`) và với hàm `kctl` định
  nghĩa ở B8; mở shell mới giữa chừng là mất hết.
- Manifest tạm ghi vào `~/lab-work/14/`; bằng chứng ghi vào `~/lab-evidence/14/`. Token sinh ra ở
  B8 chỉ tồn tại trong biến shell và trong một file kubeconfig quyền `600`, **không ghi vào
  bằng chứng**.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.

**Cách quay lui khi hỏng:** tắt **cả ba VM**, restore **cả ba** về `04-metrics-ready` — không bao
giờ restore riêng một VM, xem ghi chú cuối
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) — rồi bật lại theo
thứ tự master → worker 1 → worker 2, chạy lại gate mở đầu và bắt đầu lại từ B0.

**Trước khi chạy B0, xác nhận trên máy host rằng cả ba VM đều còn mốc `04-metrics-ready`.** Không
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
  if ($names -ccontains '04-metrics-ready') { "PASS: $f" } else { "FAIL: $f" }
}
```

**PASS:** đúng ba dòng `PASS:`. Không mở B0 khi còn dòng `FAIL:`. Đường dẫn `.vmx` ở trên theo máy
host đang dùng; sửa lại nếu VM nằm chỗ khác.

---

# Phần B — Thực hành kiến thức giai đoạn 14

## B0. Chuẩn bị workspace, namespace và tồn kho "trước"

**Mục đích:** dựng chỗ làm việc, khóa tên và API group của lab vào biến, đếm chính xác số object
phạm vi cluster **trước** khi bắt đầu, và chụp checksum cấu hình để B11 chứng minh lab không sửa gì.

### B0.1. Workspace, biến của lab và namespace

```bash
mkdir -p ~/lab-work/14 ~/lab-evidence/14
WK=~/lab-work/14
EV=~/lab-evidence/14
NS=lab-14
LAB_GROUP=lab14.example.com
MASTER=lab-k8s-master
W1=lab-k8s-worker1
W2=lab-k8s-worker2

kubectl config current-context
kubectl create namespace "$NS"
kubectl label namespace "$NS" lab=14 --overwrite
```

```bash
echo "WK=$WK | EV=$EV | NS=$NS | LAB_GROUP=$LAB_GROUP"
test -d "$WK" && test -d "$EV" && echo 'PASS: hai thu muc lam viec da ton tai'
test "$(kubectl get namespace "$NS" -o jsonpath='{.status.phase}')" = 'Active' \
  && echo 'PASS: namespace lab-14 da Active'
case "$LAB_GROUP" in
  *.*.*) echo "PASS: LAB_GROUP = $LAB_GROUP co dang DNS subdomain nhieu nhan" ;;
  *)     echo 'FAIL: LAB_GROUP phai la mot ten DNS subdomain, vi du lab14.example.com' ;;
esac
```

**Ý nghĩa:** `LAB_GROUP` là **API group mới** mà lab sẽ tạo ra. Bài
[177](../177-extend-kubernetes-vi.md) nói rõ ở mục *Thay đổi các resource có sẵn*: custom resource
luôn nằm trong một API group mới, và bạn **không thay thế hay thay đổi được API group hiện có**.
Cả lab này quay quanh đúng câu đó — cuối B3 bạn sẽ thấy `lab14.example.com` xuất hiện cạnh `apps`
và `batch` trong `kubectl api-resources`, còn nội dung của `apps` thì không hề đổi.

Tên miền `example.com` được dùng vì nó là miền dành riêng cho ví dụ, không ai sở hữu thật. Trên
cluster thật, dùng miền của tổ chức bạn.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B0.2. Tồn kho "trước" của object phạm vi cluster

Đây là bốn con số mà B11 sẽ so lại. Đọc chúng **trước khi** tạo bất cứ thứ gì:

```bash
CRD_N0="$(kubectl get crd --no-headers | wc -l)"
CR_N0="$(kubectl get clusterrole --no-headers | wc -l)"
CRB_N0="$(kubectl get clusterrolebinding --no-headers | wc -l)"
APISVC_N0="$(kubectl get apiservice --no-headers | wc -l)"
DEPLOY_N0="$(kubectl get deployments -A --no-headers | wc -l)"
POD_N0="$(kubectl get pods -A --no-headers | wc -l)"

{
  echo "=== $(date -Is) — ton kho pham vi cluster truoc Lab 14 ==="
  echo "crd=$CRD_N0 clusterrole=$CR_N0 clusterrolebinding=$CRB_N0 apiservice=$APISVC_N0"
  echo "deployment toan cluster=$DEPLOY_N0 | pod toan cluster=$POD_N0"
  echo '--- crd dang co, kem API group va scope ---'
  kubectl get crd -o custom-columns='NAME:.metadata.name,GROUP:.spec.group,SCOPE:.spec.scope' \
    --no-headers | sort
  echo '--- object pham vi cluster mang label lab=14 ---'
  kubectl get crd,clusterrole,clusterrolebinding -l lab=14 2>&1
} | tee "$EV/b0-truoc.txt"
```

```bash
test "$CRD_N0" -gt 0 \
  && echo 'PASS: cluster da co CRD tu truoc lab nay — B1 se goi ten chung'
test "$CR_N0" -gt 0 && test "$CRB_N0" -gt 0 && test "$APISVC_N0" -gt 0 \
  && echo 'PASS: doc duoc ca ba con so ClusterRole, ClusterRoleBinding va APIService'
test -z "$(kubectl get crd,clusterrole,clusterrolebinding -l lab=14 -o name 2>/dev/null)" \
  && echo 'PASS: chua object pham vi cluster nao mang label lab=14'
```

**Ý nghĩa:** con số `CRD_N0` lớn hơn 0 **trước khi bạn làm gì** chính là bài học đầu tiên của giai
đoạn 14: bạn đã dùng cơ chế CRD từ Lab 5b và Lab 6a mà chưa gọi tên nó. B1.1 sẽ mở danh sách này
ra xem từng cái đến từ đâu.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B0.3. Ảnh chụp "trước" của hạ tầng mốc `04-metrics-ready`

```bash
{
  echo "=== $(date -Is) — ha tang moc 04-metrics-ready truoc Lab 14 ==="
  echo '--- storageclass';  kubectl get storageclass -o wide 2>&1
  echo '--- ingressclass';  kubectl get ingressclass 2>&1
  echo '--- apiservice metrics'; kubectl get apiservice v1beta1.metrics.k8s.io 2>&1
  echo '--- persistentvolume'; kubectl get pv 2>&1
  echo '--- namespaces';    kubectl get namespaces
} | tee "$EV/b0-hatang-truoc.txt"

SC_NAME="$(kubectl get storageclass \
  -o jsonpath='{range .items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")]}{.metadata.name}{"\n"}{end}')"
SC_PROV="$(kubectl get storageclass "$SC_NAME" -o jsonpath='{.provisioner}')"
MS_AVAIL0="$(kubectl get apiservice v1beta1.metrics.k8s.io \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
echo "storageclass mac dinh=$SC_NAME | provisioner=$SC_PROV | metrics Available=$MS_AVAIL0"

test -n "$SC_NAME" && test -n "$SC_PROV" \
  && echo 'PASS: doc duoc ten StorageClass mac dinh va ten provisioner cua no'
test "$MS_AVAIL0" = 'True' \
  && echo 'PASS: APIService cua metrics-server dang Available — dung moc dau vao'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện. Nếu `MS_AVAIL0` không phải `True` thì cluster
chưa ở `04-metrics-ready`; dừng lại và restore trước khi đi tiếp.

### B0.4. Checksum cấu hình node và control plane

Lab 14 hứa **chỉ đọc** cấu hình kubelet và control plane. Chụp checksum ngay bây giờ để B11.4 biến
lời hứa đó thành thứ kiểm chứng được, đúng cách
[Lab 7b](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md#b92-gate-quan-trọng-nhất-của-lab-cấu-hình-node-không-đổi)
đã làm:

```bash
{
  echo "$MASTER kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in "$W1" "$W2"; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "$MASTER $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} | tee "$EV/b0-config-sha.txt"

test "$(wc -l < "$EV/b0-config-sha.txt")" -eq 6 \
  && test "$(awk '{print $3}' "$EV/b0-config-sha.txt" | grep -c '^[0-9a-f]\{64\}$')" -eq 6 \
  && echo 'PASS: ghi duoc checksum cua sau file cau hinh'
```

**Ý nghĩa:** cám dỗ lớn nhất của B10 là "bật thử Topology Manager policy khác `none` để xem bài
[313](../313-debug-topology-vi.md) nói gì". Sáu dòng checksum này chặn đúng cám dỗ đó.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B1. Bản đồ điểm mở rộng, đọc trên chính cluster của bạn

**Mục đích:** dựng lại bảy điểm mở rộng của mục *Chú giải cho hình vẽ* trong bài
[177](../177-extend-kubernetes-vi.md) bằng lệnh, rồi tìm cho mỗi điểm một hiện vật có thật trên
cluster này. Đây là mục **tổng hợp**: cuối B1 bạn sẽ chỉ ra được bốn điểm mở rộng khác nhau mà
cluster của bạn đã dùng suốt từ Lab 5b tới Lab 11a.

Toàn bộ B1 **chỉ đọc**. Không lệnh nào ở đây tạo, sửa hay xóa object.

### B1.1. Điểm 1, 2 và 3 — client, truy cập API, và loại resource

```bash
{
  echo '=== diem 1 — phan mo rong phia client (kubectl plugin) ==='
  kubectl plugin list 2>&1
  echo '=== diem 2 — phan mo rong truy cap API (admission webhook) ==='
  kubectl get validatingwebhookconfigurations 2>&1
  kubectl get mutatingwebhookconfigurations 2>&1
  echo '=== diem 3 — loai resource: CRD dang co ==='
  kubectl get crd -o custom-columns='NAME:.metadata.name,GROUP:.spec.group,SCOPE:.spec.scope' \
    --no-headers | sort
} | tee "$EV/b1-diem-1-2-3.txt"

PLUGIN_N="$(kubectl plugin list 2>/dev/null | grep -c '^/')"
VWH_N="$(kubectl get validatingwebhookconfigurations --no-headers 2>/dev/null | wc -l)"
MWH_N="$(kubectl get mutatingwebhookconfigurations --no-headers 2>/dev/null | wc -l)"
GROUP_N="$(kubectl get crd -o jsonpath='{range .items[*]}{.spec.group}{"\n"}{end}' | sort -u | wc -l)"
echo "kubectl plugin=$PLUGIN_N | validating webhook=$VWH_N | mutating webhook=$MWH_N"
echo "so API group do CRD tao ra=$GROUP_N"
```

```bash
test "$PLUGIN_N" -eq 0 \
  && echo 'PASS: diem mo rong phia client dang trong — chua ai cai kubectl plugin nao'
test "$GROUP_N" -ge 1 \
  && echo 'PASS: diem mo rong loai resource DA duoc dung — cluster co API group do CRD tao'
kubectl get crd -o jsonpath='{range .items[*]}{.spec.group}{"\n"}{end}' | sort -u \
  | tee "$EV/b1-crd-groups.txt"
test "$(wc -l < "$EV/b1-crd-groups.txt")" -eq "$GROUP_N" \
  && echo 'PASS: ghi duoc danh sach API group do CRD tao ra'
```

**Ý nghĩa:** ba điểm đầu của chú giải hình vẽ ứng với ba câu hỏi khác nhau. Điểm 1 hỏi *client có
được tùy biến không* — ở đây là không, và bài nói rõ phần mở rộng phía client "chỉ ảnh hưởng đến
môi trường cục bộ của từng người dùng, do đó không thể áp đặt các policy cho toàn hệ thống". Điểm
2 hỏi *có ai chặn hay sửa request theo nội dung không* — đây là chỗ duy nhất tác động được lên
hành vi của các API **có sẵn** như Pod. Điểm 3 hỏi *API server đang phục vụ những loại resource
nào*, và danh sách CRD ở trên là câu trả lời: mỗi API group trong `b1-crd-groups.txt` là một
add-on đã thêm kiểu resource của riêng nó vào cluster của bạn.

Nếu `VWH_N` hoặc `MWH_N` khác 0, mở `b1-diem-1-2-3.txt` và đọc tên chúng: đó là add-on đang dùng
điểm mở rộng số 2. Nếu cả hai bằng 0 thì cluster này chưa dùng điểm đó — cả hai kết quả đều hợp
lệ, tùy add-on bạn đã cài ở Lab 5b và Lab 8.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B1.2. Điểm 5, 6 và 7 — controller, network plugin, device và storage plugin

Đây là ba nhóm mà bài [182](../182-compute-storage-net-vi.md) gọi là *phần mở rộng hạ tầng*. Đọc
từng nhóm trên cluster thật:

```bash
{
  echo '=== diem 6 — network plugin: cau hinh CNI that tren node ==='
  ssh "$W2" 'ls -1 /etc/cni/net.d/'
  ssh "$W2" 'ls -1 /opt/cni/bin/ | head -20'
  echo '=== diem 7a — device plugin: thu muc dang ky cua kubelet ==='
  ssh "$W2" 'sudo ls -l /var/lib/kubelet/device-plugins/'
  echo '=== diem 7b — storage plugin: CSI driver dang dang ky ==='
  kubectl get csidrivers 2>&1
  echo '=== diem 5 — controller: provisioner cua StorageClass mac dinh ==='
  kubectl get storageclass "$SC_NAME" -o yaml | grep -E '^provisioner:|^volumeBindingMode:'
} | tee "$EV/b1-diem-5-6-7.txt"
```

Phân loại provisioner: nó là **CSI driver** (điểm 7) hay chỉ là một **controller** (điểm 5)?

```bash
CSI_N="$(kubectl get csidrivers --no-headers 2>/dev/null | wc -l)"
if kubectl get csidriver "$SC_PROV" >/dev/null 2>&1; then IS_CSI=co; else IS_CSI=khong; fi
echo "so CSIDriver dang dang ky=$CSI_N | provisioner '$SC_PROV' co la CSIDriver khong: $IS_CSI"

test "$IS_CSI" = 'khong' \
  && echo "PASS: '$SC_PROV' khong nam trong danh sach CSIDriver — no la controller, khong phai storage plugin"
```

```bash
CNI_CONF="$(ssh "$W2" 'ls -1 /etc/cni/net.d/ | wc -l')"
DP_SOCK="$(ssh "$W2" 'sudo test -S /var/lib/kubelet/device-plugins/kubelet.sock && echo co || echo khong')"
echo "so file cau hinh CNI tren $W2=$CNI_CONF | socket dang ky device plugin: $DP_SOCK"

test "$CNI_CONF" -ge 1 \
  && echo 'PASS: diem mo rong network plugin DA duoc dung — node co cau hinh CNI that'
test "$DP_SOCK" = 'co' \
  && echo 'PASS: kubelet co san socket dang ky device plugin, nhung chua plugin nao dung toi no'
```

**Ý nghĩa:** ba kết quả này tách bạch đúng ba nhóm của bài [182](../182-compute-storage-net-vi.md).

- **Network plugin là nhóm bắt buộc.** Bài viết cluster "cần" một network plugin để có mạng Pod
  hoạt động. File conflist trên node là hiện vật; gỡ nó ra thì Pod không có mạng.
- **Device plugin là nhóm bổ sung.** Socket `kubelet.sock` có sẵn vì kubelet luôn phơi ra dịch vụ
  gRPC `Registration`; nhưng chưa plugin nào tự đăng ký, nên node không công bố thiết bị nào. B10
  quay lại chỗ này với `Capacity` và `Allocatable`.
- **Provisioner của bạn không phải storage plugin.** Đây là chỗ dễ nhầm nhất của B1: bài
  [177](../177-extend-kubernetes-vi.md) xếp CSI vào điểm 7 — binary plugin do **kubelet** thực thi
  để mount volume. Provisioner mặc định của cluster này không có mặt trong `kubectl get csidrivers`,
  nghĩa là nó không phải CSI driver; nó là một **Pod thường xuyên watch PersistentVolumeClaim rồi
  tạo PersistentVolume** — tức mẫu **controller**, điểm mở rộng số 5. Đây cũng chính là lý do
  [nợ #5](README.md#5-sổ-nợ-lab) — snapshot volume — chưa trả được: không có CSI driver thì không
  có snapshot.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B1.3. Ranh giới webhook so với binary plugin

Bài [177](../177-extend-kubernetes-vi.md) chia mọi lời gọi ra ngoài thành hai mô hình, khác nhau ở
chỗ **Kubernetes gọi qua mạng hay thực thi một chương trình nhị phân**. Điền bảng dưới đây bằng
chính output bạn vừa ghi:

| Thành phần trên cluster này | Mô hình | Bên gọi | Điểm mở rộng |
| --- | --- | --- | --- |
| CNI plugin trong `/opt/cni/bin/` | binary plugin | kubelet | 6 |
| Admission webhook (nếu `VWH_N` hoặc `MWH_N` khác 0) | webhook | kube-apiserver | 2 |
| Extension API server của metrics-server | webhook (request mạng qua tầng tổng hợp) | kube-apiserver | 3 |
| Provisioner của StorageClass | không phải cả hai — nó là **client** của API server | chính nó gọi vào API server | 5 |

```bash
CNI_BIN="$(ssh "$W2" 'ls -1 /opt/cni/bin/ | wc -l')"
echo "so binary CNI tren $W2=$CNI_BIN"
test "$CNI_BIN" -ge 1 \
  && echo 'PASS: mo hinh binary plugin ton tai that — co file thuc thi tren dia cua node'
test "$MS_AVAIL0" = 'True' \
  && echo 'PASS: mo hinh goi qua mang ton tai that — metrics-server tra loi qua tang tong hop'
```

**Ý nghĩa:** hàng thứ tư là hàng đáng nhớ nhất. Controller **không** nằm trong hai mô hình kia:
Kubernetes không gọi nó, nó gọi Kubernetes. Đó là lý do bài xếp controller thành một điểm mở rộng
riêng và cũng là lý do một controller **không thêm điểm lỗi vào đường xử lý request** — nếu
provisioner chết, API server vẫn nhận PVC bình thường, chỉ là không ai đáp ứng. Ngược lại, một
admission webhook chết có thể chặn đứng mọi thao tác ghi.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B1.4. Chốt: bốn điểm mở rộng cluster của bạn đã dùng

```bash
{
  echo "=== $(date -Is) — bon diem mo rong cluster nay dang dung ==="
  echo "1) diem 6 — network plugin (CNI): $CNI_CONF file cau hinh, $CNI_BIN binary tren $W2"
  echo "2) diem 3 — tang tong hop API: apiservice v1beta1.metrics.k8s.io, Available=$MS_AVAIL0"
  echo "3) diem 3 — custom resource: $CRD_N0 CRD thuoc $GROUP_N API group"
  echo "4) diem 5 — mau controller: provisioner '$SC_PROV' cua storageclass '$SC_NAME'"
  echo "diem 1 (client) = trong | diem 4 (scheduler) = chi co scheduler mac dinh"
  echo "diem 7 (device plugin) = trong, socket dang ky co san"
} | tee "$EV/b1-bon-diem.txt"

SCHED_N="$(kubectl -n kube-system get pods -l component=kube-scheduler --no-headers 2>/dev/null | wc -l)"
echo "so Pod kube-scheduler=$SCHED_N"

test "$CNI_CONF" -ge 1 && test "$MS_AVAIL0" = 'True' \
  && test "$CRD_N0" -gt 0 && test -n "$SC_PROV" \
  && echo 'PASS: chung minh duoc bon diem mo rong khac nhau dang duoc dung tren cluster nay'
test "$SCHED_N" -ge 1 \
  && echo 'PASS: diem mo rong lap lich chi co scheduler mac dinh — khong scheduler thu hai nao'
```

**Ý nghĩa:** đây là câu trả lời cho câu hỏi mở đầu bài [177](../177-extend-kubernetes-vi.md): "mỗi
thứ bạn đã cài nằm ở ô nào trên bản đồ". Bạn chưa viết dòng mã mở rộng nào, nhưng cluster của bạn
đã dùng bốn ô khác nhau. Phần còn lại của lab đi vào ô số ba — ô mà bạn **tự** thêm được mà không
cần lập trình.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B2. Đúng hai cách thêm custom resource, đo trên cluster thật

**Mục đích:** kiểm chứng câu trung tâm của bài [178](../178-api-extension-vi.md) — có **đúng hai**
cách — và câu phân biệt của bài [180](../180-apiserver-aggregation-vi.md): CRD làm kube-apiserver
**tự phục vụ** kind mới, còn tầng tổng hợp làm kube-apiserver **chuyển tiếp** sang server khác.

### B2.1. Phân loại toàn bộ APIService của cluster

```bash
kubectl get apiservice \
  -o custom-columns='NAME:.metadata.name,SVC:.spec.service.name,SVCNS:.spec.service.namespace' \
  --no-headers | tee "$EV/b2-apiservice.txt"

PROXY_N="$(awk '$2 != "<none>"' "$EV/b2-apiservice.txt" | wc -l)"
LOCAL_N="$(awk '$2 == "<none>"' "$EV/b2-apiservice.txt" | wc -l)"
echo "apiservice duoc proxy sang mot Service=$PROXY_N | apiservice do kube-apiserver tu phuc vu=$LOCAL_N"
awk '$2 != "<none>"' "$EV/b2-apiservice.txt" | tee "$EV/b2-apiservice-proxy.txt"
```

```bash
test "$PROXY_N" -ge 1 \
  && echo 'PASS: co it nhat mot API duoc proxy — tang tong hop dang lam viec that'
test "$LOCAL_N" -gt "$PROXY_N" \
  && echo 'PASS: phan lon API van do chinh kube-apiserver phuc vu'
grep -q 'metrics.k8s.io' "$EV/b2-apiservice-proxy.txt" \
  && echo 'PASS: API duoc proxy chinh la cua metrics-server'
```

**Ý nghĩa:** bài [180](../180-apiserver-aggregation-vi.md) nói tầng tổng hợp "chạy trong cùng tiến
trình với kube-apiserver" và "cho tới khi có một extension resource được đăng ký, nó không làm gì
cả". Cột `SVC` là chỗ phân biệt: `<none>` nghĩa là APIService đó chỉ **khai báo** một đường dẫn do
chính kube-apiserver phục vụ; có tên Service nghĩa là mọi request tới đường dẫn ấy bị **chuyển
tiếp** sang Pod đứng sau Service đó.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B2.2. Đi thẳng vào đường proxy

```bash
MS_SVC="$(awk '$1 ~ /metrics.k8s.io/ {print $3"/"$2}' "$EV/b2-apiservice.txt")"
echo "metrics-server dung sau Service: $MS_SVC"

kubectl get --raw '/apis/metrics.k8s.io/v1beta1' | tee "$EV/b2-metrics-discovery.json"
echo
kubectl get --raw '/apis/metrics.k8s.io/v1beta1/nodes' > "$EV/b2-metrics-nodes.json"
wc -c "$EV/b2-metrics-nodes.json"
```

```bash
grep -q 'APIResourceList' "$EV/b2-metrics-discovery.json" \
  && echo 'PASS: duong dan /apis/metrics.k8s.io/v1beta1 tra ve danh sach resource that'
grep -q 'NodeMetricsList' "$EV/b2-metrics-nodes.json" \
  && echo 'PASS: du lieu tra ve duoc sinh ra tai thoi diem hoi, boi mot server rieng'
MS_POD_N="$(kubectl -n kube-system get pods -l k8s-app=metrics-server --no-headers 2>/dev/null | wc -l)"
echo "so Pod metrics-server dang chay=$MS_POD_N"
test "$MS_POD_N" -ge 1 \
  && echo 'PASS: co tien trinh that dung sau API nay — do la "dich vu co the hong" cua bang so sanh'
```

**Ý nghĩa:** đây là ô "Aggregated API" trong bảng *So sánh mức độ dễ dùng* của bài
[179](../179-custom-resources-vi.md), nhìn thấy được: "có thêm một dịch vụ phải tạo ra và dịch vụ
đó có thể hỏng". Bạn đã chứng kiến đúng điều đó ở
[Lab 11a phần B5](LAB-11A-OBSERVABILITY.md#b5-cài-metrics-server-và-chữa-lỗi-certificate), khi
APIService ở trạng thái `Available=False` cho tới lúc Pod metrics-server chạy được.

Chú ý điều `metrics.k8s.io` **không** làm: nó không lưu gì cả. Metric là dữ liệu được tính ra tại
thời điểm hỏi. Đó là lý do nó không thể là một CRD — CRD chỉ giúp kube-apiserver **nhận biết và
lưu trữ** kind mới.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B2.3. API group của CRD cũng có APIService — nhưng không có server nào phía sau

```bash
for g in $(cat "$EV/b1-crd-groups.txt"); do
  echo "--- API group $g ---"
  grep -E "\.${g}\b" "$EV/b2-apiservice.txt" || echo "  (khong co apiservice nao cho group nay)"
done | tee "$EV/b2-crd-group-apiservice.txt"

CRD_PROXY_N="$(awk '$2 != "<none>"' "$EV/b2-apiservice.txt" \
  | grep -cFf "$EV/b1-crd-groups.txt" || true)"
echo "so API group cua CRD duoc proxy sang mot Service = $CRD_PROXY_N"

test "$CRD_PROXY_N" -eq 0 \
  && echo 'PASS: khong API group nao cua CRD can toi mot API server rieng'
```

**Ý nghĩa:** kube-apiserver **tự** đăng ký một APIService cho mỗi API group mà nó phục vụ, kể cả
group do CRD sinh ra. Vì thế tên group của CRD có thể xuất hiện trong `kubectl get apiservice` —
nhưng luôn với `.spec.service` **rỗng**. Đó chính là ranh giới bài
[178](../178-api-extension-vi.md) mô tả bằng hai vế: với CRD, "control plane của Kubernetes phục vụ
và đảm nhận việc lưu trữ"; với tầng tổng hợp, "bạn viết và triển khai API server của riêng mình".

Nếu output ghi "(khong co apiservice nao cho group nay)" cho mọi group thì kết luận **không đổi**:
vẫn không có server riêng nào. Gate ở trên đúng trong cả hai trường hợp.

**PASS:** dòng `PASS:` của bước này xuất hiện.

---

## B3. CRD đầu tiên: tạo, đăng ký, đọc lại

**Mục đích:** làm đúng checkpoint của
[giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) — tạo một CRD, apply một
custom resource, đọc lại bằng `kubectl get` — và hiểu ba thứ xảy ra sau lưng: endpoint REST mới,
điều kiện `Established`, và cache discovery của `kubectl`.

### B3.1. Viết và apply CustomResourceDefinition

CRD ở bước này cố tình **tối giản**: chỉ có group, scope, names và schema. B4, B5 và B7 lần lượt
thêm ba lớp còn lại, mỗi lớp một mục để đo được nó thay đổi gì.

```bash
cat > "$WK/b3-crd-webpage.yaml" <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name bat buoc phai la <plural>.<group>
  name: webpages.${LAB_GROUP}
  labels:
    lab: "14"
spec:
  group: ${LAB_GROUP}
  scope: Namespaced
  names:
    plural: webpages
    singular: webpage
    kind: WebPage
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              message:
                type: string
              size:
                type: integer
          status:
            type: object
            properties:
              phase:
                type: string
              observedGeneration:
                type: integer
              configMap:
                type: string
EOF

kubectl apply --dry-run=server --validate=strict -f "$WK/b3-crd-webpage.yaml"
kubectl apply -f "$WK/b3-crd-webpage.yaml"
```

```bash
kubectl wait --for=condition=Established --timeout=120s "crd/webpages.${LAB_GROUP}"
kubectl get "crd/webpages.${LAB_GROUP}" \
  -o jsonpath='Established={.status.conditions[?(@.type=="Established")].status}{"\n"}NamesAccepted={.status.conditions[?(@.type=="NamesAccepted")].status}{"\n"}acceptedKind={.status.acceptedNames.kind}{"\n"}acceptedPlural={.status.acceptedNames.plural}{"\n"}storedVersions={.status.storedVersions}{"\n"}' \
  | tee "$EV/b3-crd-status.txt"

EST="$(kubectl get "crd/webpages.${LAB_GROUP}" \
  -o jsonpath='{.status.conditions[?(@.type=="Established")].status}')"
NAMES_OK="$(kubectl get "crd/webpages.${LAB_GROUP}" \
  -o jsonpath='{.status.conditions[?(@.type=="NamesAccepted")].status}')"
test "$EST" = 'True' && echo 'PASS: CRD da Established — endpoint REST moi da san sang'
test "$NAMES_OK" = 'True' \
  && echo 'PASS: NamesAccepted=True — khong ten nao xung dot voi resource dang co'
```

**Ý nghĩa:** `--dry-run=server --validate=strict` gửi manifest lên API server để kiểm mà không ghi
gì; đây là thói quen đúng với mọi object phạm vi cluster. Hai condition trong `status` trả lời hai
câu hỏi khác nhau: `NamesAccepted` nói *tên bạn xin có bị trùng không*, còn `Established` nói
*endpoint đã phục vụ được chưa*. Chỉ khi `Established=True` mới có gì để `kubectl get`.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B3.2. Endpoint REST mới, và cache discovery của `kubectl`

Hỏi thẳng API server — đường này **không** đi qua cache nào của client:

```bash
kubectl get --raw "/apis/${LAB_GROUP}" | tee "$EV/b3-group-discovery.json"
echo
kubectl get --raw "/apis/${LAB_GROUP}/v1" | tee "$EV/b3-version-discovery.json"
echo

grep -q "\"name\":\"webpages\"" "$EV/b3-version-discovery.json" \
  && echo 'PASS: API server da phuc vu resource webpages tai /apis/'"$LAB_GROUP"'/v1'
grep -q '"namespaced":true' "$EV/b3-version-discovery.json" \
  && echo 'PASS: resource moi duoc khai bao la namespaced, dung nhu spec.scope'
```

Bây giờ mới tới lượt `kubectl`. Nó giữ một **cache discovery phía client** ở `~/.kube/cache`, nên
kiểu vừa tạo có thể chưa xuất hiện ngay. Cách xử lý là **làm mới cache**, không phải chờ:

```bash
# Cache discovery la file cua client, xoa duoc an toan; kubectl tu dung lai o lan goi sau.
rm -rf ~/.kube/cache/discovery
kubectl api-resources --api-group="$LAB_GROUP" | tee "$EV/b3-api-resources.txt"

test "$(grep -c 'WebPage' "$EV/b3-api-resources.txt")" -ge 1 \
  && echo 'PASS: kubectl da nhin thay kind WebPage trong API group moi'
```

**Ý nghĩa:** hai lệnh `--raw` hỏi trực tiếp API server; `kubectl api-resources` đi qua cache. Phân
biệt được hai đường này là thứ cứu bạn khỏi kết luận sai "CRD chưa được tạo" khi thực ra chỉ là
client chưa làm mới danh sách kiểu. **Thời gian sống của cache phụ thuộc cấu hình `kubectl`**, nên
lab không nêu con số; nó xóa cache để kết quả xác định. Xem thêm
[mục 4](#4-troubleshooting-của-lab-này).

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B3.3. Apply custom resource đầu tiên và đọc lại

```bash
cat > "$WK/b3-cr-trang-a.yaml" <<EOF
apiVersion: ${LAB_GROUP}/v1
kind: WebPage
metadata:
  name: trang-a
  namespace: ${NS}
  labels:
    lab: "14"
spec:
  message: Xin chao tu custom resource
  size: 2
EOF

kubectl apply --dry-run=server --validate=strict -f "$WK/b3-cr-trang-a.yaml"
kubectl apply -f "$WK/b3-cr-trang-a.yaml"

kubectl get webpages -n "$NS" | tee "$EV/b3-get-webpages.txt"
kubectl get webpage trang-a -n "$NS" -o yaml | tee "$EV/b3-cr-doc-lai.yaml"
```

```bash
MSG_BACK="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.spec.message}')"
SIZE_BACK="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.spec.size}')"
UID_BACK="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.metadata.uid}')"
echo "message=$MSG_BACK | size=$SIZE_BACK | uid co dai=$(printf '%s' "$UID_BACK" | wc -c)"

test "$MSG_BACK" = 'Xin chao tu custom resource' \
  && echo 'PASS: doc lai dung gia tri da apply — API server luu tru custom resource that'
test "$SIZE_BACK" -eq 2 \
  && echo 'PASS: truong so nguyen giu dung kieu, khong bi bien thanh chuoi'
test -n "$UID_BACK" \
  && echo 'PASS: object co uid do API server cap — no la object day du, khong phai mot manh YAML'
```

**Ý nghĩa:** đây là **checkpoint của giai đoạn 14, làm xong**. Chú ý những gì bạn nhận được miễn
phí, đúng bảng *Các tính năng chung* của bài [179](../179-custom-resources-vi.md): CRUD qua HTTP và
`kubectl`, discovery, HTTPS, xác thực và phân quyền dựng sẵn, `metadata` với `uid` và
`resourceVersion`, label và annotation. Bạn không viết một dòng mã nào cho những thứ đó.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B3.4. Đọc theo ba bí danh, và `kubectl explain`

```bash
kubectl get webpages -n "$NS" --no-headers | wc -l
kubectl get webpage  -n "$NS" --no-headers | wc -l
kubectl get "webpages.${LAB_GROUP}" -n "$NS" --no-headers | wc -l
kubectl explain webpage.spec | tee "$EV/b3-explain.txt"
```

```bash
N1="$(kubectl get webpages -n "$NS" --no-headers | wc -l)"
N2="$(kubectl get webpage  -n "$NS" --no-headers | wc -l)"
N3="$(kubectl get "webpages.${LAB_GROUP}" -n "$NS" --no-headers | wc -l)"
echo "so nhieu=$N1 | so it=$N2 | day du <plural>.<group>=$N3"

test "$N1" -eq 1 && test "$N2" -eq 1 && test "$N3" -eq 1 \
  && echo 'PASS: ba cach goi ten cung tro toi mot resource'
grep -q 'message' "$EV/b3-explain.txt" \
  && echo 'PASS: kubectl explain doc duoc schema — API server cong bo schema OpenAPI cua kind moi'
```

**Ý nghĩa:** dạng đầy đủ `<plural>.<group>` là dạng **không bao giờ nhập nhằng**; hãy dùng nó trong
script và khi hai add-on cùng đặt tên một kind. `kubectl explain` chạy được là bằng chứng cho ô
*OpenAPI Schema* trong bảng *Tính năng nâng cao* của bài [179](../179-custom-resources-vi.md):
schema lấy động được từ server, nên client bảo vệ được người dùng khỏi gõ sai tên trường.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B3.5. CRD tạo API group mới mà không cần API server nào

```bash
DEPLOY_N1="$(kubectl get deployments -A --no-headers | wc -l)"
POD_N1="$(kubectl get pods -A --no-headers | wc -l)"
SVC_OF_CRD="$(kubectl get "apiservice/v1.${LAB_GROUP}" --ignore-not-found \
  -o jsonpath='{.spec.service.name}')"
echo "deployment: truoc=$DEPLOY_N0 sau=$DEPLOY_N1 | pod: truoc=$POD_N0 sau=$POD_N1"
echo "apiservice cho group moi tro toi Service ten: '${SVC_OF_CRD:-<khong co>}'"

test "$DEPLOY_N1" -eq "$DEPLOY_N0" && test "$POD_N1" -eq "$POD_N0" \
  && echo 'PASS: them mot API vao cluster ma khong them Deployment hay Pod nao'
test -z "$SVC_OF_CRD" \
  && echo 'PASS: API group moi khong tro toi Service nao — khong co API server thu hai'
kubectl get --raw "/apis/${LAB_GROUP}/v1/namespaces/${NS}/webpages" \
  | grep -q 'WebPageList' \
  && echo 'PASS: chinh kube-apiserver tra loi tren duong dan cua group moi'
```

**Ý nghĩa:** đây là **ranh giới CRD so với aggregated API**, chứng minh xong bằng ba dữ kiện. Một,
API group mới đã tồn tại và phục vụ được. Hai, không APIService nào của nó trỏ tới một Service —
đối chiếu với `metrics.k8s.io` ở B2.2 thì khác biệt hiện ra ngay. Ba, không Pod và không Deployment
nào được thêm vào cluster.

Đúng bảng *So sánh mức độ dễ dùng* của bài [179](../179-custom-resources-vi.md), cột CRD: "không
đòi hỏi lập trình", "không có dịch vụ bổ sung nào phải chạy; CRD được API server xử lý". Bạn vừa
thêm một API vào Kubernetes bằng đúng một file YAML.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B4. Schema là hợp đồng được API server cưỡng chế

**Mục đích:** kiểm chứng ô *Validation* và ô *OpenAPI Schema* của bảng *Tính năng nâng cao và tính
linh hoạt* trong bài [179](../179-custom-resources-vi.md) — hai ô mà CRD ghi "Có" và người ta hay
tưởng là "Không". Sau mục này bạn sẽ đọc được đúng câu từ chối của API server và biết trường lạ đi
đâu.

### B4.1. Siết schema: bắt buộc, giới hạn và biểu thức chính quy

Sửa CRD của B3 bằng cách thêm ràng buộc vào đúng schema cũ. Đây là một lần **cập nhật** CRD; object
`trang-a` đã lưu **không bị kiểm lại**, vì validation chỉ chạy lúc ghi:

```bash
cat > "$WK/b4-crd-webpage.yaml" <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webpages.${LAB_GROUP}
  labels:
    lab: "14"
spec:
  group: ${LAB_GROUP}
  scope: Namespaced
  names:
    plural: webpages
    singular: webpage
    kind: WebPage
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: ["message"]
            properties:
              message:
                type: string
                minLength: 1
                maxLength: 80
                pattern: '^[A-Za-z0-9 .,-]+$'
              size:
                type: integer
                minimum: 1
                maximum: 5
          status:
            type: object
            properties:
              phase:
                type: string
              observedGeneration:
                type: integer
              configMap:
                type: string
EOF

kubectl apply -f "$WK/b4-crd-webpage.yaml"
kubectl wait --for=condition=Established --timeout=120s "crd/webpages.${LAB_GROUP}"
```

```bash
REQ_OK="$(kubectl get "crd/webpages.${LAB_GROUP}" \
  -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.required[0]}')"
MAXI="$(kubectl get "crd/webpages.${LAB_GROUP}" \
  -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.spec.properties.size.maximum}')"
echo "truong bat buoc=$REQ_OK | size.maximum=$MAXI"

test "$REQ_OK" = 'message' && test "$MAXI" -eq 5 \
  && echo 'PASS: rang buoc moi da nam trong CRD tren server'
test "$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.spec.message}')" \
     = 'Xin chao tu custom resource' \
  && echo 'PASS: object da luu khong bi kiem lai — validation chi chay luc ghi'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B4.2. Ba kiểu vi phạm, ba câu từ chối khác nhau

```bash
cat > "$WK/b4-thieu-message.yaml" <<EOF
apiVersion: ${LAB_GROUP}/v1
kind: WebPage
metadata:
  name: thieu-message
  namespace: ${NS}
spec:
  size: 2
EOF

cat > "$WK/b4-size-qua-lon.yaml" <<EOF
apiVersion: ${LAB_GROUP}/v1
kind: WebPage
metadata:
  name: size-qua-lon
  namespace: ${NS}
spec:
  message: Trang thu nghiem
  size: 9
EOF

cat > "$WK/b4-sai-kieu.yaml" <<EOF
apiVersion: ${LAB_GROUP}/v1
kind: WebPage
metadata:
  name: sai-kieu
  namespace: ${NS}
spec:
  message: Trang thu nghiem
  size: "hai"
EOF

for f in b4-thieu-message b4-size-qua-lon b4-sai-kieu; do
  echo "=== $f ==="
  kubectl apply --dry-run=server -f "$WK/$f.yaml" 2>&1
done | tee "$EV/b4-tu-choi.txt"
```

```bash
CREATED_N="$(kubectl get webpages -n "$NS" --no-headers | wc -l)"
echo "so WebPage dang co sau ba lan thu=$CREATED_N"

grep -qi 'required' "$EV/b4-tu-choi.txt" \
  && echo 'PASS: thieu truong bat buoc bi tu choi, thong bao noi ro Required'
grep -q 'spec.size' "$EV/b4-tu-choi.txt" \
  && echo 'PASS: thong bao goi dung duong dan truong sai la spec.size'
grep -qi 'integer' "$EV/b4-tu-choi.txt" \
  && echo 'PASS: sai kieu du lieu bi bat, thong bao nhac dung kieu mong doi la integer'
test "$CREATED_N" -eq 1 \
  && echo 'PASS: khong object hong nao duoc tao — --dry-run=server kiem ma khong ghi'
```

**Ý nghĩa:** cả ba câu từ chối đến từ **API server**, không phải từ `kubectl`. Đó là điều làm
validation của CRD khác hẳn việc tự kiểm trong ứng dụng: mọi client đều bị chặn như nhau, kể cả
client gọi thẳng REST. Bài [179](../179-custom-resources-vi.md) đặt điểm này ở ngay đầu ô
*Validation*: nó "cho phép bạn phát triển API độc lập với các client của mình".

Chú ý `--dry-run=server`: request đi trọn đường qua xác thực, phân quyền, admission và validation
rồi bị vứt bỏ trước khi ghi. Đây là cách an toàn để kiểm một manifest trên cluster thật.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B4.3. Trường lạ: bị từ chối hay bị cắt tỉa?

```bash
cat > "$WK/b4-truong-la.yaml" <<EOF
apiVersion: ${LAB_GROUP}/v1
kind: WebPage
metadata:
  name: trang-la
  namespace: ${NS}
  labels:
    lab: "14"
spec:
  message: Trang co truong la
  size: 1
  mauNen: xanh
EOF

echo '=== apply mac dinh (validate=strict) ==='
kubectl apply -f "$WK/b4-truong-la.yaml" 2>&1 | tee "$EV/b4-truong-la-strict.txt"
echo '=== apply voi validate=ignore ==='
kubectl apply --validate=ignore -f "$WK/b4-truong-la.yaml" 2>&1 \
  | tee "$EV/b4-truong-la-ignore.txt"
```

```bash
MAU="$(kubectl get webpage trang-la -n "$NS" -o jsonpath='{.spec.mauNen}')"
MSG_LA="$(kubectl get webpage trang-la -n "$NS" -o jsonpath='{.spec.message}')"
echo "spec.mauNen doc lai duoc: '${MAU:-<rong>}' | spec.message='$MSG_LA'"

grep -qi 'unknown field' "$EV/b4-truong-la-strict.txt" \
  && echo 'PASS: che do strict tu choi truong la thay vi im lang bo qua'
test -n "$MSG_LA" \
  && echo 'PASS: voi validate=ignore, object van duoc tao'
test -z "$MAU" \
  && echo 'PASS: truong la KHONG duoc luu — API server da cat tia no khoi object'
```

**Ý nghĩa:** hai hành vi khác nhau ở hai tầng khác nhau, và lẫn lộn chúng là lỗi kinh điển.

- `--validate=strict` là **kiểm tra trường**: request bị từ chối vì bạn gõ một tên không có trong
  schema. Đây là lá chắn chống gõ sai — đúng câu hỏi mà ô *OpenAPI Schema* của bài
  [179](../179-custom-resources-vi.md) đặt ra: "người dùng có được bảo vệ khỏi việc gõ sai tên
  trường bằng cách chỉ cho phép đặt các trường hợp lệ hay không".
- **Cắt tỉa trường** là hành vi lưu trữ: nếu request lọt qua, mọi trường không khai trong schema bị
  **bỏ đi trước khi ghi**. Custom resource **không** phải chỗ nhét dữ liệu tùy ý; schema là ranh
  giới của những gì tồn tại.

Hệ quả vận hành: một trường bạn quên khai trong schema sẽ **biến mất im lặng** với client dùng
`--validate=ignore`. Đó là lý do lúc thiết kế CRD, schema phải viết trước, không phải viết sau.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

```bash
kubectl delete webpage trang-la -n "$NS"
test "$(kubectl get webpages -n "$NS" --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: da don object thu nghiem cua B4, chi con trang-a'
```

**PASS:** dòng `PASS:` trên xuất hiện.

---

## B5. Bề mặt CLI của một kind mới

**Mục đích:** chứng minh `shortNames`, `categories` và `additionalPrinterColumns` là thuộc tính của
**server**, không phải mẹo của `kubectl`, và đo được chúng đổi gì. Ba trường này là thứ khiến một
custom resource "trông như resource dựng sẵn" — đúng dòng của bài
[179](../179-custom-resources-vi.md): "bạn muốn `kubectl` hỗ trợ ở mức cao nhất".

### B5.1. Ảnh chụp "trước" của output `kubectl get`

```bash
kubectl get webpages -n "$NS" | tee "$EV/b5-get-truoc.txt"
COL_TRUOC="$(head -1 "$EV/b5-get-truoc.txt" | wc -w)"
echo "so cot truoc khi them printer column=$COL_TRUOC"

test "$COL_TRUOC" -eq 2 \
  && echo 'PASS: mac dinh chi co hai cot NAME va AGE'
kubectl get wp -n "$NS" 2>&1 | tee "$EV/b5-shortname-truoc.txt"
grep -qi 'error\|the server doesn' "$EV/b5-shortname-truoc.txt" \
  && echo 'PASS: bi danh wp chua ton tai — chua khai shortNames'
```

**Ý nghĩa:** `kubectl` **không** tự bịa ra cột. Bài [378](../378-custom-resource-definitions-vi.md)
nói thẳng ở mục *Cột hiển thị bổ sung*: "API server của cluster quyết định những cột nào được hiển
thị bởi lệnh `kubectl get`". Hai cột `NAME` và `AGE` là mặc định của server cho mọi kind chưa khai
cột riêng.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B5.2. Thêm `shortNames`, `categories` và `additionalPrinterColumns`

```bash
cat > "$WK/b5-crd-webpage.yaml" <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webpages.${LAB_GROUP}
  labels:
    lab: "14"
spec:
  group: ${LAB_GROUP}
  scope: Namespaced
  names:
    plural: webpages
    singular: webpage
    kind: WebPage
    shortNames:
    - wp
    categories:
    - lab14
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: ["message"]
            properties:
              message:
                type: string
                minLength: 1
                maxLength: 80
                pattern: '^[A-Za-z0-9 .,-]+$'
              size:
                type: integer
                minimum: 1
                maximum: 5
          status:
            type: object
            properties:
              phase:
                type: string
              observedGeneration:
                type: integer
              configMap:
                type: string
    additionalPrinterColumns:
    - name: Message
      type: string
      description: Noi dung se duoc controller ghi ra ConfigMap
      jsonPath: .spec.message
    - name: Size
      type: integer
      jsonPath: .spec.size
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
EOF

kubectl apply -f "$WK/b5-crd-webpage.yaml"
kubectl wait --for=condition=Established --timeout=120s "crd/webpages.${LAB_GROUP}"

# Ba truong vua them nam trong thong tin discovery, nen phai lam moi cache cua client.
rm -rf ~/.kube/cache/discovery
kubectl api-resources --api-group="$LAB_GROUP" -o wide | tee "$EV/b5-api-resources.txt"
```

```bash
kubectl get webpages -n "$NS" | tee "$EV/b5-get-sau.txt"
COL_SAU="$(head -1 "$EV/b5-get-sau.txt" | wc -w)"
echo "so cot sau khi them printer column=$COL_SAU"

test "$COL_SAU" -gt "$COL_TRUOC" \
  && echo 'PASS: output kubectl get da nhieu cot hon truoc'
head -1 "$EV/b5-get-sau.txt" | grep -q 'MESSAGE' \
  && echo 'PASS: cot MESSAGE lay tu .spec.message da xuat hien'
grep -q 'wp' "$EV/b5-api-resources.txt" \
  && echo 'PASS: shortName wp da nam trong thong tin discovery cua server'
```

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B5.3. Ba lối gọi mới, và một lối gọi theo category

```bash
kubectl get wp -n "$NS" | tee "$EV/b5-shortname-sau.txt"
kubectl get lab14 -n "$NS" | tee "$EV/b5-category.txt"
kubectl get all -n "$NS" 2>&1 | tee "$EV/b5-category-all.txt"
```

```bash
SHORT_N="$(kubectl get wp -n "$NS" --no-headers | wc -l)"
CAT_N="$(kubectl get lab14 -n "$NS" --no-headers | wc -l)"
ALL_N="$(kubectl get all -n "$NS" --no-headers 2>/dev/null | grep -c 'webpage' || true)"
echo "wp=$SHORT_N | category lab14=$CAT_N | xuat hien trong 'kubectl get all'=$ALL_N"

test "$SHORT_N" -eq 1 \
  && echo 'PASS: shortName wp goi duoc custom resource'
test "$CAT_N" -eq 1 \
  && echo 'PASS: kubectl get lab14 tra ve custom resource — category hoat dong'
test "$ALL_N" -eq 0 \
  && echo 'PASS: khong lot vao category all vi CRD nay khong khai all'
```

**Ý nghĩa:** category là **danh sách nhóm resource** mà kind của bạn thuộc về. `all` chỉ là một
category có sẵn tên đẹp, không phải "mọi thứ" — CRD này khai `lab14` chứ không khai `all`, nên
`kubectl get all` không thấy nó. Trên cluster thật, đây là cách gom mọi kind của một sản phẩm lại
để `kubectl get <ten-san-pham>` liệt kê được hết trong một lệnh.

Nếu `kubectl get lab14` báo không nhận kiểu, đó là cache discovery chứ không phải CRD sai — làm mới
cache như B5.2 rồi chạy lại; xem [mục 4](#4-troubleshooting-của-lab-này).

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B6. `scope`, và vì sao RBAC không tự cấp quyền cho kind mới

**Mục đích:** chứng minh bằng thực nghiệm khác biệt giữa `scope: Namespaced` và `scope: Cluster`,
rồi kiểm chứng câu cảnh báo vận hành của bài [179](../179-custom-resources-vi.md) ở mục *Xác thực,
phân quyền và kiểm toán*: "hầu hết các role RBAC sẽ không cấp quyền truy cập tới resource mới".

Đây cũng là mục **tạo hai object phạm vi cluster** đã khai ở
[bảng mục 2](#2-quy-ước-và-an-toàn). Từ đây tới hết lab, mọi object mới đều mang label `lab=14`.

### B6.1. Ranh giới namespace của kind namespaced

```bash
kubectl get webpages -n "$NS" --no-headers | wc -l
kubectl get webpages -n default 2>&1 | tee "$EV/b6-webpage-default.txt"
kubectl get webpages -A | tee "$EV/b6-webpage-all-ns.txt"
kubectl api-resources --namespaced=true --api-group="$LAB_GROUP" \
  | tee "$EV/b6-namespaced-true.txt"
```

```bash
IN_NS="$(kubectl get webpages -n "$NS" --no-headers | wc -l)"
IN_DEF="$(kubectl get webpages -n default --no-headers 2>/dev/null | wc -l)"
echo "webpage trong $NS=$IN_NS | trong default=$IN_DEF"

test "$IN_NS" -eq 1 && test "$IN_DEF" -eq 0 \
  && echo 'PASS: cung mot kind, hai namespace, hai ket qua — object thuoc ve namespace'
grep -q 'webpages' "$EV/b6-namespaced-true.txt" \
  && echo 'PASS: server tu khai bao webpages la kind namespaced'
```

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B6.2. ServiceAccount mang ClusterRole `view` vẫn không đọc được kind mới

```bash
kubectl create serviceaccount sa-doc-wp -n "$NS"
kubectl create rolebinding doc-view -n "$NS" \
  --clusterrole=view --serviceaccount="${NS}:sa-doc-wp"
SA_WP="system:serviceaccount:${NS}:sa-doc-wp"

kubectl get clusterrole view -o yaml > "$EV/b6-clusterrole-view.yaml"
grep -c "$LAB_GROUP" "$EV/b6-clusterrole-view.yaml" || true
```

```bash
CAN_CM="$(kubectl auth can-i list configmaps -n "$NS" --as="$SA_WP")"
CAN_WP="$(kubectl auth can-i list "webpages.${LAB_GROUP}" -n "$NS" --as="$SA_WP")"
VIEW_MENTION="$(grep -c "$LAB_GROUP" "$EV/b6-clusterrole-view.yaml" || true)"
echo "view co doc duoc configmap khong=$CAN_CM | co doc duoc webpage khong=$CAN_WP"
echo "so lan ClusterRole view nhac toi $LAB_GROUP=$VIEW_MENTION"

test "$CAN_CM" = 'yes' \
  && echo 'PASS: ClusterRole view that su dang co hieu luc — no doc duoc ConfigMap'
test "$CAN_WP" = 'no' \
  && echo 'PASS: nhung chinh no KHONG doc duoc kind moi'
test "$VIEW_MENTION" -eq 0 \
  && echo 'PASS: ly do nam trong chinh noi dung role — no khong he nhac toi API group moi'
```

**Ý nghĩa:** đây là câu bẫy lớn nhất về vận hành CRD, và ba dòng trên chứng minh trọn vẹn. Role
dựng sẵn liệt kê **tường minh** từng API group và từng resource; API group của bạn ra đời sau, nên
không role nào biết tới nó. Bài [179](../179-custom-resources-vi.md) chốt: "Bạn sẽ cần cấp quyền
truy cập một cách tường minh cho resource mới. CRD và Aggregated API thường được đóng gói kèm các
định nghĩa role mới cho những kiểu dữ liệu mà chúng bổ sung."

Đó cũng là lý do một Helm chart cài operator luôn kèm sẵn một bộ ClusterRole: không phải để bản
thân operator chạy, mà để **người dùng** đọc được kind mới.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

```bash
kubectl create role doc-webpage -n "$NS" \
  --verb=get,list,watch --resource="webpages.${LAB_GROUP}"
kubectl create rolebinding doc-webpage -n "$NS" \
  --role=doc-webpage --serviceaccount="${NS}:sa-doc-wp"

CAN_WP2="$(kubectl auth can-i list "webpages.${LAB_GROUP}" -n "$NS" --as="$SA_WP")"
echo "sau khi cap quyen tuong minh=$CAN_WP2"
test "$CAN_WP2" = 'yes' \
  && echo 'PASS: cap quyen tuong minh cho API group moi thi doc duoc'
```

**PASS:** dòng `PASS:` trên xuất hiện. Hai object vừa tạo là Role và RoleBinding **trong namespace**
`lab-14`, nên chúng biến mất theo namespace ở B11.1 — khác hẳn hai object ở B6.5.

### B6.3. CRD thứ hai, lần này `scope: Cluster`

```bash
cat > "$WK/b6-crd-clusterwidget.yaml" <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: clusterwidgets.${LAB_GROUP}
  labels:
    lab: "14"
spec:
  group: ${LAB_GROUP}
  scope: Cluster
  names:
    plural: clusterwidgets
    singular: clusterwidget
    kind: ClusterWidget
    shortNames:
    - cw
    categories:
    - lab14
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              note:
                type: string
                maxLength: 120
    additionalPrinterColumns:
    - name: Note
      type: string
      jsonPath: .spec.note
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
EOF

kubectl apply --dry-run=server --validate=strict -f "$WK/b6-crd-clusterwidget.yaml"
kubectl apply -f "$WK/b6-crd-clusterwidget.yaml"
kubectl wait --for=condition=Established --timeout=120s "crd/clusterwidgets.${LAB_GROUP}"
rm -rf ~/.kube/cache/discovery
kubectl api-resources --namespaced=false --api-group="$LAB_GROUP" \
  | tee "$EV/b6-namespaced-false.txt"

cat > "$WK/b6-cr-widget.yaml" <<EOF
apiVersion: ${LAB_GROUP}/v1
kind: ClusterWidget
metadata:
  name: widget-toan-cluster
  labels:
    lab: "14"
spec:
  note: Object nay khong thuoc namespace nao
EOF

kubectl apply -f "$WK/b6-cr-widget.yaml"
kubectl get clusterwidgets | tee "$EV/b6-get-clusterwidget.txt"
```

```bash
CW_N="$(kubectl get clusterwidgets --no-headers | wc -l)"
CW_NS="$(kubectl get clusterwidget widget-toan-cluster -o jsonpath='{.metadata.namespace}')"
CW_TRONG_NS="$(kubectl get clusterwidgets -n "$NS" --no-headers | wc -l)"
echo "clusterwidget=$CW_N | metadata.namespace='${CW_NS:-<rong>}' | goi kem -n $NS ra=$CW_TRONG_NS"

test "$CW_N" -eq 1 \
  && echo 'PASS: doc duoc custom resource pham vi cluster ma khong can -n'
test -z "$CW_NS" \
  && echo 'PASS: object khong co metadata.namespace — no khong thuoc namespace nao'
test "$CW_TRONG_NS" -eq 1 \
  && echo 'PASS: them -n cung ra dung ket qua do — co -n bi bo qua voi kind cluster-scoped'
grep -q 'clusterwidgets' "$EV/b6-namespaced-false.txt" \
  && echo 'PASS: server khai bao clusterwidgets la kind KHONG namespaced'
```

**Ý nghĩa:** `-n` bị bỏ qua chứ không báo lỗi — đây là nguồn nhầm lẫn thường trực khi vận hành.
Bạn tưởng mình đang xóa object của một namespace, thực ra bạn đang xóa object của **cả cluster**.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B6.4. Kind cluster-scoped đòi ClusterRole, Role không đủ

`sa-doc-wp` đang có một Role đọc `webpages` trong `lab-14`. Nó có đọc được `clusterwidgets` không?

```bash
CAN_CW="$(kubectl auth can-i list "clusterwidgets.${LAB_GROUP}" --as="$SA_WP")"
echo "Role trong lab-14 co doc duoc clusterwidget khong=$CAN_CW"
test "$CAN_CW" = 'no' \
  && echo 'PASS: quyen gan bang Role khong voi toi object pham vi cluster'
```

**PASS:** dòng `PASS:` trên xuất hiện.

Cấp quyền đúng cách — đây là **hai object cluster-scoped** duy nhất lab tạo ngoài hai CRD:

```bash
cat > "$WK/b6-rbac-clusterwidget.yaml" <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: lab-14-doc-clusterwidget
  labels:
    lab: "14"
rules:
- apiGroups: ["${LAB_GROUP}"]
  resources: ["clusterwidgets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: lab-14-doc-clusterwidget
  labels:
    lab: "14"
subjects:
- kind: ServiceAccount
  name: sa-doc-wp
  namespace: ${NS}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: lab-14-doc-clusterwidget
EOF

kubectl apply --dry-run=server --validate=strict -f "$WK/b6-rbac-clusterwidget.yaml"
kubectl apply -f "$WK/b6-rbac-clusterwidget.yaml"
```

```bash
CAN_CW2="$(kubectl auth can-i list "clusterwidgets.${LAB_GROUP}" --as="$SA_WP")"
CAN_WP3="$(kubectl auth can-i list "webpages.${LAB_GROUP}" -n default --as="$SA_WP")"
LABELED_N="$(kubectl get crd,clusterrole,clusterrolebinding -l lab=14 --no-headers 2>/dev/null | wc -l)"
echo "sau ClusterRoleBinding: clusterwidget=$CAN_CW2 | webpage trong default=$CAN_WP3"
echo "so object pham vi cluster mang label lab=14=$LABELED_N"

test "$CAN_CW2" = 'yes' \
  && echo 'PASS: ClusterRole gan bang ClusterRoleBinding moi voi toi kind cluster-scoped'
test "$CAN_WP3" = 'no' \
  && echo 'PASS: quyen moi khong lan sang kind khac — no chi ke ten clusterwidgets'
test "$LABELED_N" -eq 4 \
  && echo 'PASS: dung bon object pham vi cluster mang label lab=14, khop bang o muc 2'
```

**Ý nghĩa:** cùng một bài học của
[Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md#b3-chặng-2--phân-quyền-và-rbac), lần này áp
lên một kind bạn tự định nghĩa: **phạm vi của quyền phải khớp phạm vi của object**. Và vì CRD "luôn
dùng cùng cơ chế xác thực, phân quyền và ghi log kiểm toán như các resource dựng sẵn", bạn không
phải học một mô hình phân quyền thứ hai — đó chính là ưu điểm bài
[179](../179-custom-resources-vi.md) nêu ở cột CRD.

Con số `4` ở gate cuối là con số B11.2 sẽ đưa về `0`.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B6.5. Bẫy: xóa namespace **không** dọn object phạm vi cluster

Thí nghiệm có kiểm soát, trên một namespace dùng một lần:

```bash
kubectl create namespace lab-14-tam
kubectl label namespace lab-14-tam lab=14 --overwrite

cat > "$WK/b6-cr-tam.yaml" <<EOF
apiVersion: ${LAB_GROUP}/v1
kind: WebPage
metadata:
  name: trang-tam
  namespace: lab-14-tam
  labels:
    lab: "14"
spec:
  message: Trang nay se chet theo namespace
  size: 1
EOF

kubectl apply -f "$WK/b6-cr-tam.yaml"
WP_TAM_TRUOC="$(kubectl get webpages -n lab-14-tam --no-headers | wc -l)"
CW_TRUOC="$(kubectl get clusterwidgets --no-headers | wc -l)"
CRD_TRUOC="$(kubectl get crd -l lab=14 --no-headers | wc -l)"
echo "truoc khi xoa namespace: webpage trong lab-14-tam=$WP_TAM_TRUOC clusterwidget=$CW_TRUOC crd=$CRD_TRUOC"

kubectl delete namespace lab-14-tam --wait=true --timeout=300s
```

```bash
NS_TAM="$(kubectl get namespace lab-14-tam --ignore-not-found -o name | wc -l)"
CW_SAU="$(kubectl get clusterwidgets --no-headers | wc -l)"
CRD_SAU="$(kubectl get crd -l lab=14 --no-headers | wc -l)"
WP_CON="$(kubectl get webpages -A --no-headers | wc -l)"
echo "namespace lab-14-tam con=$NS_TAM | clusterwidget=$CW_SAU | crd lab=14=$CRD_SAU"
echo "tong so webpage con lai tren toan cluster=$WP_CON"

test "$NS_TAM" -eq 0 \
  && echo 'PASS: namespace da bien mat'
test "$WP_CON" -eq 1 \
  && echo 'PASS: WebPage trong namespace do chet theo namespace — dung nhu kind namespaced'
test "$CW_SAU" -eq "$CW_TRUOC" \
  && echo 'PASS: ClusterWidget KHONG chet theo namespace — no khong thuoc namespace nao'
test "$CRD_SAU" -eq "$CRD_TRUOC" \
  && echo 'PASS: hai CRD van con nguyen — CRD la object pham vi cluster'
```

**Ý nghĩa:** bài [378](../378-custom-resource-definitions-vi.md) viết đúng hai câu này ở mục *Tạo
một CustomResourceDefinition*: "việc xóa một namespace sẽ xóa toàn bộ custom object trong namespace
đó" và "Bản thân các CustomResourceDefinition thì không thuộc namespace nào và có sẵn cho mọi
namespace". Bạn vừa chứng minh cả hai trong một thí nghiệm.

Hệ quả vận hành: `kubectl delete namespace` **không phải** lệnh dọn dẹp đủ cho một lab hay một sản
phẩm dùng CRD. Đó là lý do B11 tồn tại và vì sao nó đếm.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

---

## B7. Subresource `status`: hai đường ghi tách nhau

**Mục đích:** đo chính xác điều mà `subresources.status` đổi. Đây là ô *Status Subresource* của
bảng *Tính năng nâng cao và tính linh hoạt* trong bài [179](../179-custom-resources-vi.md), và nó
là điều kiện kỹ thuật để B8 chia được vai "người dùng ghi `spec`, controller ghi `status`".

### B7.1. Trước khi bật: `status` chỉ là một trường như mọi trường khác

CRD hiện tại **có** `status` trong schema nhưng **chưa** bật subresource. Hai thứ đó khác nhau:

```bash
SUBRES="$(kubectl get "crd/webpages.${LAB_GROUP}" \
  -o jsonpath='{.spec.versions[0].subresources}')"
echo "subresources hien tai='${SUBRES:-<khong co>}'"

kubectl patch webpage trang-a -n "$NS" --type=merge \
  -p '{"status":{"phase":"nguoi-dung-ghi"}}'
PHASE_1="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.status.phase}')"
echo "status.phase sau khi patch tren resource chinh='${PHASE_1:-<rong>}'"

test -z "$SUBRES" \
  && echo 'PASS: CRD chua bat subresource nao'
test "$PHASE_1" = 'nguoi-dung-ghi' \
  && echo 'PASS: chua bat subresource thi ai co quyen ghi object la ghi duoc ca status'
```

**Ý nghĩa:** có `status` trong schema chỉ nghĩa là **cho phép lưu** một object con tên `status`.
Không có gì bảo vệ nó. Bất kỳ ai sửa được object là sửa được `status` — tức là một người dùng có
thể tự khai "tôi Ready" mà không ai làm gì.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B7.2. Bật subresource `status`

CRD dưới đây giống hệt bản ở B5.2, **chỉ thêm hai dòng** `subresources:` và `status: {}` — chúng
là anh em cùng cấp với `schema:` và `additionalPrinterColumns:` bên trong một phần tử của
`versions`:

```bash
cat > "$WK/b7-crd-webpage.yaml" <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webpages.${LAB_GROUP}
  labels:
    lab: "14"
spec:
  group: ${LAB_GROUP}
  scope: Namespaced
  names:
    plural: webpages
    singular: webpage
    kind: WebPage
    shortNames:
    - wp
    categories:
    - lab14
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: ["message"]
            properties:
              message:
                type: string
                minLength: 1
                maxLength: 80
                pattern: '^[A-Za-z0-9 .,-]+$'
              size:
                type: integer
                minimum: 1
                maximum: 5
          status:
            type: object
            properties:
              phase:
                type: string
              observedGeneration:
                type: integer
              configMap:
                type: string
    subresources:
      status: {}
    additionalPrinterColumns:
    - name: Message
      type: string
      description: Noi dung se duoc controller ghi ra ConfigMap
      jsonPath: .spec.message
    - name: Size
      type: integer
      jsonPath: .spec.size
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
EOF

diff -u "$WK/b5-crd-webpage.yaml" "$WK/b7-crd-webpage.yaml" | tee "$EV/b7-diff-crd.txt"
test "$(grep -c '^+[^+]' "$EV/b7-diff-crd.txt")" -eq 2 \
  && test "$(grep -c '^-[^-]' "$EV/b7-diff-crd.txt")" -eq 0 \
  && echo 'PASS: so voi ban B5, CRD chi them dung hai dong va khong bo dong nao'

kubectl apply --dry-run=server --validate=strict -f "$WK/b7-crd-webpage.yaml"
kubectl apply -f "$WK/b7-crd-webpage.yaml"
kubectl wait --for=condition=Established --timeout=120s "crd/webpages.${LAB_GROUP}"
```

```bash
SUBRES2="$(kubectl get "crd/webpages.${LAB_GROUP}" \
  -o jsonpath='{.spec.versions[0].subresources.status}')"
echo "subresources.status='${SUBRES2:-<khong co>}'"
kubectl get --raw "/apis/${LAB_GROUP}/v1" | tr ',' '\n' | grep -q 'webpages/status' \
  && echo 'PASS: duong dan webpages/status da xuat hien trong discovery cua API group'
test -n "$SUBRES2" \
  && echo 'PASS: CRD da khai subresources.status'
```

**Ý nghĩa:** `diff` là gate ở đây vì nó chứng minh **thay đổi duy nhất** so với B5 là hai dòng bật
subresource. Nhờ vậy mọi khác biệt hành vi quan sát được ở B7.3 và B7.4 chỉ có thể do hai dòng đó,
không do thứ gì khác bạn vô tình sửa. Đường dẫn `webpages/status` xuất hiện trong discovery là bằng
chứng phía server: API server vừa mở thêm một endpoint con cho kind này.

**PASS:** ba dòng `PASS:` của bước này xuất hiện — một của `diff` và hai của khối gate.

### B7.3. Sau khi bật: cùng lệnh patch đó không còn tác dụng

```bash
kubectl patch webpage trang-a -n "$NS" --type=merge \
  -p '{"status":{"phase":"co-gang-ghi-lai"}}'
PHASE_2="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.status.phase}')"
echo "status.phase sau khi patch tren resource chinh='${PHASE_2:-<rong>}'"

test "$PHASE_2" = 'nguoi-dung-ghi' \
  && echo 'PASS: gia tri cu van nguyen — thay doi len status qua resource chinh bi bo qua'
```

```bash
kubectl patch webpage trang-a -n "$NS" --subresource=status --type=merge \
  -p '{"status":{"phase":"controller-ghi"}}'
PHASE_3="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.status.phase}')"
echo "status.phase sau khi patch qua subresource='${PHASE_3:-<rong>}'"

test "$PHASE_3" = 'controller-ghi' \
  && echo 'PASS: chi duong /status moi ghi duoc status'
```

```bash
MSG_TRUOC_STATUS="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.spec.message}')"
kubectl patch webpage trang-a -n "$NS" --subresource=status --type=merge \
  -p '{"spec":{"message":"Ghi spec qua duong status"}}' 2>&1 \
  | tee "$EV/b7-ghi-spec-qua-status.txt"
MSG_SAU_STATUS="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.spec.message}')"
echo "spec.message truoc='$MSG_TRUOC_STATUS' | sau='$MSG_SAU_STATUS'"

test "$MSG_SAU_STATUS" = "$MSG_TRUOC_STATUS" \
  && echo 'PASS: duong /status khong ghi duoc spec — hai chieu deu bi chan'
```

**Ý nghĩa:** đây là toàn bộ giá trị của subresource `status`, và nó là **cơ chế phân quyền chứ
không phải cú pháp**. Bài [179](../179-custom-resources-vi.md) mô tả đúng một câu: "cho phép kiểm
soát truy cập chi tiết, trong đó người dùng ghi phần spec còn controller ghi phần status". B8 sẽ
dùng đúng ranh giới này: ServiceAccount của controller được cấp `webpages/status` chứ **không** được
cấp quyền sửa `webpages`.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B7.4. `metadata.generation` chỉ tăng theo `spec`

```bash
gen_wp() { kubectl get webpage trang-a -n "$NS" -o jsonpath='{.metadata.generation}'; }

GEN_0="$(gen_wp)"; GEN_0="${GEN_0:-0}"
kubectl patch webpage trang-a -n "$NS" --subresource=status --type=merge \
  -p '{"status":{"phase":"lan-hai"}}' >/dev/null
GEN_1="$(gen_wp)"; GEN_1="${GEN_1:-0}"
kubectl patch webpage trang-a -n "$NS" --type=merge \
  -p '{"spec":{"size":3}}' >/dev/null
GEN_2="$(gen_wp)"; GEN_2="${GEN_2:-0}"
echo "generation: ban dau=$GEN_0 | sau khi ghi status=$GEN_1 | sau khi ghi spec=$GEN_2"

test "$GEN_1" -eq "$GEN_0" \
  && echo 'PASS: ghi status khong lam tang generation'
test "$GEN_2" -gt "$GEN_1" \
  && echo 'PASS: ghi spec thi generation tang — day la tin hieu "co viec moi cho controller"'
```

**Ý nghĩa:** cặp `metadata.generation` và `status.observedGeneration` là **giao thức tiêu chuẩn**
giữa người dùng và controller: người dùng đổi `spec` thì `generation` tăng; controller làm xong
thì ghi `observedGeneration` bằng `generation`. Hai số bằng nhau nghĩa là controller đã bắt kịp;
khác nhau nghĩa là còn việc chưa làm. B8 hiện thực đúng giao thức đó, và bạn sẽ đọc được nó bằng
mắt trong cột `PHASE` của `kubectl get wp`.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

---

## B8. CRD không có controller thì chỉ là kho lưu dữ liệu

**Mục đích:** đây là **mục đích cuối cùng của cả giai đoạn 14**, và là câu mà
[checkpoint của lộ trình](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) bắt bạn giải
thích được. Bài [179](../179-custom-resources-vi.md) viết: "Bản thân custom resource chỉ cho phép
bạn lưu trữ và truy xuất dữ liệu có cấu trúc. Khi bạn kết hợp một custom resource với một *custom
controller*, custom resource sẽ cung cấp một API khai báo thực thụ." B8 chứng minh **cả hai vế**:
vế đầu bằng một custom resource không làm gì cả, vế sau bằng một vòng lặp điều khiển do bạn tự chạy.

> **Lab này KHÔNG viết controller bằng Go và KHÔNG cài operator framework** — cả hai đòi toolchain
> ngoài baseline, xem [bảng lý do ở mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành). Thay vào đó
> bạn đóng vai controller bằng một hàm shell chạy `kubectl` dưới danh tính một ServiceAccount có
> quyền tối thiểu. B8.6 liệt kê chính xác những gì khác so với một operator thật.

### B8.1. Một custom resource, và không có gì xảy ra

Đếm mọi thứ trong namespace **trước**:

```bash
CM_N0="$(kubectl get configmaps -n "$NS" --no-headers | wc -l)"
POD_NS0="$(kubectl get pods -n "$NS" --no-headers 2>/dev/null | wc -l)"
DEP_NS0="$(kubectl get deployments -n "$NS" --no-headers 2>/dev/null | wc -l)"
echo "truoc: configmap=$CM_N0 pod=$POD_NS0 deployment=$DEP_NS0"

cat > "$WK/b8-cr-trang-b.yaml" <<EOF
apiVersion: ${LAB_GROUP}/v1
kind: WebPage
metadata:
  name: trang-b
  namespace: ${NS}
  labels:
    lab: "14"
spec:
  message: Trang B chua co ai cham toi
  size: 1
EOF

kubectl apply -f "$WK/b8-cr-trang-b.yaml"
kubectl get wp -n "$NS" | tee "$EV/b8-truoc-controller.txt"
```

Quan sát có giới hạn — vòng lặp dừng ngay khi thấy bất kỳ object nào xuất hiện:

```bash
i=0; DOI=khong
while [ "$i" -lt 15 ]; do
  CM_NOW="$(kubectl get configmaps -n "$NS" --no-headers | wc -l)"
  POD_NOW="$(kubectl get pods -n "$NS" --no-headers 2>/dev/null | wc -l)"
  if [ "$CM_NOW" -ne "$CM_N0" ] || [ "$POD_NOW" -ne "$POD_NS0" ]; then DOI=co; break; fi
  i=$(( i + 1 )); sleep 2
done
echo "sau $i vong quan sat: co gi doi khong = $DOI"

EV_N="$(kubectl get events -n "$NS" --field-selector involvedObject.kind=WebPage \
  --no-headers 2>/dev/null | wc -l)"
PHASE_B="$(kubectl get webpage trang-b -n "$NS" -o jsonpath='{.status.phase}')"
echo "so Event lien quan toi WebPage=$EV_N | status.phase cua trang-b='${PHASE_B:-<rong>}'"
```

```bash
test "$DOI" = 'khong' \
  && echo 'PASS: qua ca vong quan sat, khong object nao xuat hien trong namespace'
test "$EV_N" -eq 0 \
  && echo 'PASS: khong thanh phan nao phat Event ve WebPage — khong ai dang lang nghe kind nay'
test -z "$PHASE_B" \
  && echo 'PASS: status van rong — khong ai ghi vao no'
test "$(kubectl get webpage trang-b -n "$NS" -o jsonpath='{.spec.message}')" \
     = 'Trang B chua co ai cham toi' \
  && echo 'PASS: du lieu thi van con nguyen — day dung la mot kho luu du lieu'
```

**Ý nghĩa:** bốn gate trên là **vế thứ nhất của luận điểm**, chứng minh xong. Bạn đã có một API
đầy đủ: schema, validation, `kubectl get`, RBAC, watch, audit log. Bạn **chưa có** một API khai báo,
vì không có ai đọc `spec` rồi làm cho thực tế khớp với nó.

`kubectl` vẫn chạy trơn tru dù thiếu controller — đúng như bài [181](../181-operator-vi.md) mô tả ở
mục *Triển khai operator*: cách phổ biến nhất là thêm **Custom Resource Definition và Controller
tương ứng**, hai vế chứ không phải một. Bạn vừa cài đúng một vế.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B8.2. Danh tính và quyền tối thiểu cho controller

Controller là một **client của Kubernetes API**, nên nó cần một danh tính và một bộ quyền. Dùng lại
đúng ba bước của [Lab 9a](LAB-9A-SERVICEACCOUNT-AUTHN-AUTHZ-VA-RBAC.md#b5-serviceaccount-là-danh-tính-của-pod):
tạo ServiceAccount, cấp quyền, rồi gán danh tính đó cho thứ đang chạy.

```bash
kubectl create serviceaccount sa-lab14-controller -n "$NS"

cat > "$WK/b8-role-thieu-status.yaml" <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: lab14-controller
  namespace: ${NS}
rules:
- apiGroups: ["${LAB_GROUP}"]
  resources: ["webpages"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: lab14-controller
  namespace: ${NS}
subjects:
- kind: ServiceAccount
  name: sa-lab14-controller
  namespace: ${NS}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: lab14-controller
EOF

kubectl apply -f "$WK/b8-role-thieu-status.yaml"
```

Dựng kubeconfig cho danh tính đó — địa chỉ và CA lấy từ chính kubeconfig quản trị, credential thì
không:

```bash
SRV="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
CACERT=/etc/kubernetes/pki/ca.crt
KCFG="$WK/ctrl.kubeconfig"
rm -f "$KCFG"; touch "$KCFG"; chmod 600 "$KCFG"

TOK_CTRL="$(kubectl create token sa-lab14-controller -n "$NS")"
kubectl --kubeconfig="$KCFG" config set-cluster lab \
  --server="$SRV" --certificate-authority="$CACERT" --embed-certs=true >/dev/null
kubectl --kubeconfig="$KCFG" config set-credentials ctrl --token="$TOK_CTRL" >/dev/null
kubectl --kubeconfig="$KCFG" config set-context lab \
  --cluster=lab --user=ctrl --namespace="$NS" >/dev/null
kubectl --kubeconfig="$KCFG" config use-context lab >/dev/null

kctl() { kubectl --kubeconfig="$KCFG" -n "$NS" "$@"; }
```

```bash
WHO="$(kctl auth whoami -o jsonpath='{.status.userInfo.username}')"
echo "controller dang la: $WHO"
test "$WHO" = "system:serviceaccount:${NS}:sa-lab14-controller" \
  && echo 'PASS: kubeconfig cua controller xac thuc dung danh tinh ServiceAccount'
test "$(stat -c '%a' "$KCFG")" = '600' \
  && echo 'PASS: file kubeconfig chua token dang o quyen 600'
test "$(kctl auth can-i list webpages)" = 'yes' \
  && test "$(kctl auth can-i create configmaps)" = 'yes' \
  && echo 'PASS: controller doc duoc webpage va tao duoc configmap'
test "$(kctl auth can-i update webpages)" = 'no' \
  && test "$(kctl auth can-i delete configmaps)" = 'no' \
  && test "$(kctl auth can-i create pods)" = 'no' \
  && echo 'PASS: va khong lam duoc gi ngoai ba viec do — quyen dung la toi thieu'
```

**Ý nghĩa:** chú ý điều controller **không** được cấp: quyền `update` trên `webpages`. Đó là ranh
giới B7 vừa dựng, nay được RBAC cưỡng chế: người dùng ghi `spec`, controller ghi `status`, và không
bên nào lấn sân. Nếu controller sửa được `spec`, nó có thể tự viết lại yêu cầu của người dùng rồi
báo cáo "đã xong" — một operator viết ẩu hoàn toàn có thể làm thế.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện. Không ghi `TOK_CTRL` ra bằng chứng.

### B8.3. Vòng lặp điều khiển, viết bằng shell

Đây là toàn bộ "operator" của lab. Ba việc, đúng thứ tự của mọi controller: **đọc trạng thái mong
muốn**, **làm cho thực tế khớp**, **báo cáo lại bằng `status`**.

```bash
reconcile() {
  local name msg gen uid cm rc=0
  for name in $(kctl get webpages -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); do
    msg="$(kctl get webpage "$name" -o jsonpath='{.spec.message}')"
    gen="$(kctl get webpage "$name" -o jsonpath='{.metadata.generation}')"
    uid="$(kctl get webpage "$name" -o jsonpath='{.metadata.uid}')"
    cm="web-${name}"
    cat > "$WK/reconcile-${name}.yaml" <<YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: ${cm}
  namespace: ${NS}
  labels:
    lab: "14"
  ownerReferences:
  - apiVersion: ${LAB_GROUP}/v1
    kind: WebPage
    name: ${name}
    uid: ${uid}
    controller: true
    blockOwnerDeletion: false
data:
  message: "${msg}"
YAML
    kctl apply -f "$WK/reconcile-${name}.yaml" >/dev/null || rc=1
    kctl patch webpage "$name" --subresource=status --type=merge \
      -p "{\"status\":{\"phase\":\"Ready\",\"observedGeneration\":${gen},\"configMap\":\"${cm}\"}}" \
      >/dev/null 2>&1 || rc=1
    echo "reconcile $name -> configmap=$cm generation=$gen"
  done
  return "$rc"
}
```

Chạy lần đầu — và nó **fail một nửa**, đúng như thiết kế:

```bash
reconcile; echo "ma tra ve cua reconcile=$?"
kctl patch webpage trang-a --subresource=status --type=merge \
  -p '{"status":{"phase":"Ready"}}' 2>&1 | tee "$EV/b8-status-bi-tu-choi.txt"
```

```bash
CM_SAU1="$(kubectl get configmaps -n "$NS" --no-headers | wc -l)"
PHASE_A1="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.status.phase}')"
echo "configmap trong namespace=$CM_SAU1 | status.phase cua trang-a='$PHASE_A1'"

test "$CM_SAU1" -gt "$CM_N0" \
  && echo 'PASS: phan tao ConfigMap chay duoc — controller co quyen do'
grep -qi 'forbidden' "$EV/b8-status-bi-tu-choi.txt" \
  && echo 'PASS: nhung ghi status bi tu choi — Role chua ke ten webpages/status'
grep -q 'webpages/status' "$EV/b8-status-bi-tu-choi.txt" \
  && echo 'PASS: thong bao goi dung ten subresource con thieu quyen'
```

**Ý nghĩa:** quyền trên `webpages` **không** kéo theo quyền trên `webpages/status`. RBAC coi
subresource là một resource riêng, và đó chính là thứ làm cơ chế "hai đường ghi" của B7 có giá trị
thực tế — nếu chúng dùng chung một quyền thì tách đường cũng vô nghĩa.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

Bổ sung đúng một rule rồi chạy lại:

```bash
cat > "$WK/b8-role-du.yaml" <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: lab14-controller
  namespace: ${NS}
rules:
- apiGroups: ["${LAB_GROUP}"]
  resources: ["webpages"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["${LAB_GROUP}"]
  resources: ["webpages/status"]
  verbs: ["get", "patch", "update"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
EOF

kubectl apply -f "$WK/b8-role-du.yaml"

i=0; CAN_ST=no
while [ "$i" -lt 15 ]; do
  CAN_ST="$(kctl auth can-i patch "webpages.${LAB_GROUP}" --subresource=status)"
  test "$CAN_ST" = 'yes' && break
  i=$(( i + 1 )); sleep 2
done
echo "controller co patch duoc webpages/status khong=$CAN_ST (sau $i vong)"

reconcile; echo "ma tra ve cua reconcile=$?"
kubectl get wp -n "$NS" | tee "$EV/b8-sau-controller.txt"
```

```bash
PHASE_A2="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.status.phase}')"
PHASE_B2="$(kubectl get webpage trang-b -n "$NS" -o jsonpath='{.status.phase}')"
CM_A="$(kubectl get configmap web-trang-a -n "$NS" -o jsonpath='{.data.message}')"
SPEC_A="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.spec.message}')"
echo "phase: trang-a=$PHASE_A2 trang-b=$PHASE_B2"
echo "configmap web-trang-a data.message='$CM_A' | spec.message='$SPEC_A'"

test "$CAN_ST" = 'yes' \
  && echo 'PASS: them dung mot rule la mo duoc duong ghi status'
test "$PHASE_A2" = 'Ready' && test "$PHASE_B2" = 'Ready' \
  && echo 'PASS: ca hai custom resource da co status do controller ghi'
test "$CM_A" = "$SPEC_A" \
  && echo 'PASS: noi dung ConfigMap khop dung spec.message — du lieu da bien thanh hanh vi'
```

**Ý nghĩa:** đây là **vế thứ hai của luận điểm**, chứng minh xong. Không có gì trong CRD thay đổi
giữa B8.1 và bây giờ: cùng schema, cùng object, cùng API. Thứ duy nhất được thêm vào là **một vòng
lặp đọc `spec` rồi hành động**. Đó là toàn bộ khác biệt giữa "kho lưu dữ liệu" và "API khai báo".

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B8.4. Ba tính chất của một vòng lặp điều khiển

Một controller không phải là "cái chạy khi tôi apply". Ba thí nghiệm dưới đây tách bạch điều đó.

**Một — hội tụ.** Đổi `spec`, chưa chạy vòng lặp:

```bash
kubectl patch webpage trang-a -n "$NS" --type=merge \
  -p '{"spec":{"message":"Noi dung da doi lan hai"}}' >/dev/null

GEN_A="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.metadata.generation}')"
OBS_A="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.status.observedGeneration}')"
CM_A2="$(kubectl get configmap web-trang-a -n "$NS" -o jsonpath='{.data.message}')"
echo "generation=$GEN_A observedGeneration=$OBS_A | configmap van dang giu='$CM_A2'"

test "$GEN_A" -gt "$OBS_A" \
  && echo 'PASS: generation da vuot observedGeneration — tin hieu "con viec chua lam"'
test "$CM_A2" != 'Noi dung da doi lan hai' \
  && echo 'PASS: thuc te chua doi theo, vi vong lap chua chay'
```

```bash
reconcile >/dev/null
GEN_A2="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.metadata.generation}')"
OBS_A2="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.status.observedGeneration}')"
CM_A3="$(kubectl get configmap web-trang-a -n "$NS" -o jsonpath='{.data.message}')"
echo "sau reconcile: generation=$GEN_A2 observedGeneration=$OBS_A2 configmap='$CM_A3'"

test "$OBS_A2" -eq "$GEN_A2" \
  && echo 'PASS: observedGeneration da bat kip generation'
test "$CM_A3" = 'Noi dung da doi lan hai' \
  && echo 'PASS: thuc te da hoi tu ve trang thai mong muon moi'
```

**Hai — tự chữa.** Xóa object phụ thuộc bằng tay, với tư cách quản trị:

```bash
kubectl delete configmap web-trang-a -n "$NS"
CM_MAT="$(kubectl get configmap web-trang-a -n "$NS" --ignore-not-found -o name | wc -l)"
echo "configmap sau khi xoa: con $CM_MAT"

reconcile >/dev/null
CM_VE="$(kubectl get configmap web-trang-a -n "$NS" --ignore-not-found -o name | wc -l)"
CM_A4="$(kubectl get configmap web-trang-a -n "$NS" -o jsonpath='{.data.message}')"
echo "configmap sau khi chay lai vong lap: con $CM_VE, noi dung='$CM_A4'"

test "$CM_MAT" -eq 0 \
  && echo 'PASS: object phu thuoc da that su bi xoa'
test "$CM_VE" -eq 1 && test "$CM_A4" = 'Noi dung da doi lan hai' \
  && echo 'PASS: vong lap dung lai no voi dung noi dung — day la tu chua, khong phai phat lai lenh'
```

**Ba — vòng lặp là bất biến theo số lần chạy.** Chạy thêm hai lần, không gì đổi:

```bash
RV_1="$(kubectl get configmap web-trang-a -n "$NS" -o jsonpath='{.metadata.resourceVersion}')"
reconcile >/dev/null; reconcile >/dev/null
RV_2="$(kubectl get configmap web-trang-a -n "$NS" -o jsonpath='{.metadata.resourceVersion}')"
echo "resourceVersion cua ConfigMap: truoc=$RV_1 sau hai lan reconcile=$RV_2"

test "$RV_1" = "$RV_2" \
  && echo 'PASS: chay lai vong lap khi da khop thi khong ghi gi — thao tac la idempotent'
```

**Ý nghĩa:** ba tính chất này là định nghĩa thật của mẫu controller trong bài
[25](../25-controllers-vi.md) mà bài [181](../181-operator-vi.md) nối tiếp: vòng lặp **so sánh** hai
trạng thái rồi **thu hẹp khoảng cách**, chứ không "phản ứng với lệnh của bạn". Đó là lý do nó tự
chữa được và chạy lại được bao nhiêu lần cũng không hỏng — trong khi một script chạy một lần thì
không có tính chất nào trong ba tính chất trên.

**PASS:** bảy dòng `PASS:` của ba thí nghiệm xuất hiện.

### B8.5. Phần dọn dẹp **không** phải công của controller

```bash
CM_TRUOC_XOA="$(kubectl get configmaps -n "$NS" -l lab=14 --no-headers | wc -l)"
kubectl get configmap web-trang-b -n "$NS" \
  -o jsonpath='owner={.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}{"\n"}uid={.metadata.ownerReferences[0].uid}{"\n"}' \
  | tee "$EV/b8-owner-ref.txt"

kubectl delete webpage trang-b -n "$NS"

i=0
while [ "$i" -lt 30 ]; do
  kubectl get configmap web-trang-b -n "$NS" >/dev/null 2>&1 || break
  i=$(( i + 1 )); sleep 2
done
CM_B_CON="$(kubectl get configmap web-trang-b -n "$NS" --ignore-not-found -o name | wc -l)"
echo "sau $i vong cho: configmap web-trang-b con $CM_B_CON"
```

```bash
test "$CM_TRUOC_XOA" -eq 2 \
  && echo 'PASS: truoc khi xoa co dung hai ConfigMap do vong lap tao ra'
test "$CM_B_CON" -eq 0 \
  && echo 'PASS: ConfigMap bien mat theo custom resource — vong lap KHONG he chay lan nao'
grep -q 'WebPage/trang-b' "$EV/b8-owner-ref.txt" \
  && echo 'PASS: ly do nam o ownerReferences tro tu ConfigMap len WebPage'
```

**Ý nghĩa:** đây là chỗ dễ gán công nhầm nhất. Việc xóa theo dây chuyền do **garbage collector của
control plane** làm, dựa trên `ownerReferences` mà bạn đã học ở
[Lab 1c phần B2](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md#b2-owner-reference-và-garbage-collection).
Controller chỉ có công **gắn** owner reference lúc tạo. Một operator thật khai thác đúng điều này:
gắn owner reference cho mọi object nó tạo, để phần dọn dẹp thành việc của Kubernetes chứ không phải
việc nó phải nhớ.

Điều garbage collector **không** làm được là những việc phải xảy ra **trước** khi xóa — ví dụ bước
"tạo một snapshot rồi mới xóa StatefulSet và Volume" trong ví dụ `SampleDB` của bài
[181](../181-operator-vi.md). Muốn thế thì controller phải đặt **finalizer** như
[Lab 1c phần B1](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md#b1-finalizer-giữ-object-ở-trạng-thái-đang-xóa)
đã làm, và tự gỡ finalizer khi dọn xong.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B8.6. Đối chiếu với bảy bước của `SampleDB`, và những gì lab cố ý không có

Ánh xạ từng bước ví dụ *Một ví dụ về operator* trong bài [181](../181-operator-vi.md) sang thứ bạn
vừa chạy:

| Bước của `SampleDB` | Trong lab này | Khác biệt |
| --- | --- | --- |
| 1. Một custom resource `SampleDB` cấu hình vào cluster | CRD `webpages.lab14.example.com` và hai object của nó | giống hệt về bản chất |
| 2. Một Deployment đảm bảo có Pod chạy phần controller | hàm `reconcile` chạy trong shell trên `lab-k8s-master` | **khác**: không có Deployment, không có Pod. Vòng lặp do bạn gọi tay |
| 3. Một container image chứa mã của operator | không có; mã là mấy dòng `kubectl` | **khác**: không cần build image |
| 4. Mã truy vấn control plane tìm các resource đang được cấu hình | `kctl get webpages` ở đầu hàm | giống; operator thật dùng `watch` thay vì hỏi lại từ đầu |
| 5. Làm cho thực tế khớp cấu hình: tạo PVC, StatefulSet, Job | tạo ConfigMap mang `ownerReferences` | giống về cơ chế, khác về loại object được tạo |
| 5. Khi xóa: tạo snapshot **trước**, rồi xóa StatefulSet và Volume | garbage collector xóa ConfigMap theo owner reference | **khác**: lab không có bước "trước khi xóa", vì không dùng finalizer |
| 6. Sao lưu định kỳ theo lịch | không có | **khác**: cần một vòng lặp thường trực, không phải lời gọi tay |
| 7. Mã hỗ trợ: phát hiện phiên bản cũ và tạo Job nâng cấp | không có | **khác**: đây là phần "tri thức nghiệp vụ", thứ chỉ có ý nghĩa với ứng dụng thật |

Bốn khác biệt đó gói gọn thành đúng một câu: **lab có vòng lặp nhưng không có nơi chạy thường
trực**. Trên cluster thật, chỗ chạy đó là một Deployment trong cluster — bài
[181](../181-operator-vi.md) nói rõ ở mục *Triển khai operator*: "Controller thường chạy bên ngoài
control plane, giống như cách bạn chạy bất kỳ ứng dụng container hóa nào."

```bash
{
  echo "=== $(date -Is) — tong ket B8 ==="
  kubectl get wp -n "$NS" -o wide
  kubectl get configmaps -n "$NS" -l lab=14
  kubectl get webpage trang-a -n "$NS" \
    -o jsonpath='generation={.metadata.generation} observedGeneration={.status.observedGeneration} phase={.status.phase} configMap={.status.configMap}{"\n"}'
} | tee "$EV/b8-tong-ket.txt"

OBS_F="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.status.observedGeneration}')"
GEN_F="$(kubectl get webpage trang-a -n "$NS" -o jsonpath='{.metadata.generation}')"
test "$OBS_F" -eq "$GEN_F" \
  && echo 'PASS: ket thuc B8 voi controller da bat kip trang thai mong muon'
```

**PASS:** dòng `PASS:` trên xuất hiện.

---

## B9. Storage version và `status.storedVersions`

**Mục đích:** trả lời câu hỏi mở đầu của bài [323](../323-storage-version-migration-vi.md) — vì sao
phải **chủ động ghi lại** dữ liệu trong etcd — bằng thứ đọc được từ chính API server. Object
`StorageVersionMigration` cần feature gate nên nằm ngoài lab (xem
[bảng lý do](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành)); ở đây bạn chạy **phương án 2** của bài
[377](../377-custom-resource-definition-versioning-vi.md): nâng cấp thủ công, không cần feature gate.

Toàn bộ B9 thao tác trên CRD `clusterwidgets` của B6, để `webpages` và vòng lặp B8 không bị động tới.

### B9.1. Một version, một storage version

```bash
SV_1="$(kubectl get "crd/clusterwidgets.${LAB_GROUP}" -o jsonpath='{.status.storedVersions}')"
VER_1="$(kubectl get "crd/clusterwidgets.${LAB_GROUP}" -o jsonpath='{.spec.versions[*].name}')"
CONV="$(kubectl get "crd/clusterwidgets.${LAB_GROUP}" -o jsonpath='{.spec.conversion.strategy}')"
echo "spec.versions=$VER_1 | status.storedVersions=$SV_1 | conversion.strategy=$CONV"

test "$SV_1" = '["v1"]' \
  && echo 'PASS: moi object dang duoc luu o v1 — do la storage version duy nhat tung ton tai'
test "$CONV" = 'None' \
  && echo 'PASS: chien luoc chuyen doi mac dinh la None'
```

**Ý nghĩa:** `spec.versions` là những version **được phục vụ**; `status.storedVersions` là những
version **từng được dùng để lưu**. Hai danh sách khác nhau, và bài
[377](../377-custom-resource-definition-versioning-vi.md) chốt một câu quan trọng: "Không thể có
object nào tồn tại trong bộ lưu trữ ở một phiên bản chưa từng là storage version."

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B9.2. Thêm `v2` và chuyển storage version sang nó

```bash
cat > "$WK/b9-crd-v2.yaml" <<EOF
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: clusterwidgets.${LAB_GROUP}
  labels:
    lab: "14"
spec:
  group: ${LAB_GROUP}
  scope: Cluster
  conversion:
    # None gia dinh moi phien ban dung chung mot schema va chi dat lai truong apiVersion.
    strategy: None
  names:
    plural: clusterwidgets
    singular: clusterwidget
    kind: ClusterWidget
    shortNames:
    - cw
    categories:
    - lab14
  versions:
  - name: v1
    served: true
    storage: false
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              note:
                type: string
                maxLength: 120
    additionalPrinterColumns:
    - name: Note
      type: string
      jsonPath: .spec.note
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
  - name: v2
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              note:
                type: string
                maxLength: 120
    additionalPrinterColumns:
    - name: Note
      type: string
      jsonPath: .spec.note
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
EOF

kubectl apply --dry-run=server --validate=strict -f "$WK/b9-crd-v2.yaml"
kubectl apply -f "$WK/b9-crd-v2.yaml"
kubectl wait --for=condition=Established --timeout=120s "crd/clusterwidgets.${LAB_GROUP}"
rm -rf ~/.kube/cache/discovery
```

```bash
i=0; SV_2=''
while [ "$i" -lt 15 ]; do
  SV_2="$(kubectl get "crd/clusterwidgets.${LAB_GROUP}" -o jsonpath='{.status.storedVersions}')"
  case "$SV_2" in *v2*) break ;; esac
  i=$(( i + 1 )); sleep 2
done
VER_2="$(kubectl get "crd/clusterwidgets.${LAB_GROUP}" -o jsonpath='{.spec.versions[*].name}')"
echo "spec.versions=$VER_2 | status.storedVersions=$SV_2 (sau $i vong)"

case "$SV_2" in
  *v1*) echo 'PASS: v1 van nam trong storedVersions — object cu VAN dang o dinh dang v1' ;;
esac
case "$SV_2" in
  *v2*) echo 'PASS: v2 da duoc them vao storedVersions ngay khi no thanh storage version' ;;
esac
```

Đọc cùng một object qua hai version:

```bash
kubectl get "clusterwidgets.v1.${LAB_GROUP}" widget-toan-cluster -o yaml \
  > "$EV/b9-doc-v1.yaml"
kubectl get "clusterwidgets.v2.${LAB_GROUP}" widget-toan-cluster -o yaml \
  > "$EV/b9-doc-v2.yaml"
diff -u "$EV/b9-doc-v1.yaml" "$EV/b9-doc-v2.yaml" | tee "$EV/b9-diff-hai-version.txt"

NOTE_V1="$(kubectl get "clusterwidgets.v1.${LAB_GROUP}" widget-toan-cluster -o jsonpath='{.spec.note}')"
NOTE_V2="$(kubectl get "clusterwidgets.v2.${LAB_GROUP}" widget-toan-cluster -o jsonpath='{.spec.note}')"
echo "note doc qua v1='$NOTE_V1' | note doc qua v2='$NOTE_V2'"

test "$NOTE_V1" = "$NOTE_V2" \
  && echo 'PASS: noi dung doc ra qua hai version giong het nhau'
grep -q "^-apiVersion: ${LAB_GROUP}/v1" "$EV/b9-diff-hai-version.txt" \
  && grep -q "^+apiVersion: ${LAB_GROUP}/v2" "$EV/b9-diff-hai-version.txt" \
  && ! grep -qE '^[+-]  note:' "$EV/b9-diff-hai-version.txt" \
  && echo 'PASS: chi truong apiVersion doi, phan spec khong doi — dung nhu chien luoc None mo ta'
```

**Ý nghĩa:** đây là hai câu của bài [377](../377-custom-resource-definition-versioning-vi.md) hiện
ra bằng số. Một, `storedVersions` **mọc thêm** ngay khi bạn đổi storage version, và v1 **không tự
biến mất** — vì các object cũ vẫn nằm nguyên trong bộ lưu trữ theo định dạng v1. Hai, chuyển đổi
`None` chỉ đặt lại `apiVersion`, nên nó **chỉ an toàn khi mọi version dùng chung một schema**;
schema của v1 và v2 ở trên giống hệt nhau đúng vì lý do đó.

**PASS:** bốn dòng `PASS:` của bước này xuất hiện.

### B9.3. API server từ chối gỡ một version còn nằm trong `storedVersions`

```bash
sed '/^  - name: v1$/,/^      jsonPath: .metadata.creationTimestamp$/d' "$WK/b9-crd-v2.yaml" \
  > "$WK/b9-crd-bo-v1.yaml"
grep -cE '^  - name: v[12]$' "$WK/b9-crd-bo-v1.yaml"

kubectl apply -f "$WK/b9-crd-bo-v1.yaml" 2>&1 | tee "$EV/b9-tu-choi-bo-v1.txt"
```

```bash
VER_CON="$(kubectl get "crd/clusterwidgets.${LAB_GROUP}" -o jsonpath='{.spec.versions[*].name}')"
echo "spec.versions sau khi thu bo v1=$VER_CON"

test "$(grep -cE '^  - name: v[12]$' "$WK/b9-crd-bo-v1.yaml")" -eq 1 \
  && echo 'PASS: file moi chi con mot version'
grep -qi 'storedVersions' "$EV/b9-tu-choi-bo-v1.txt" \
  && echo 'PASS: API server tu choi, va noi thang ly do nam o status.storedVersions'
echo "$VER_CON" | grep -q 'v1' \
  && echo 'PASS: CRD tren server khong he doi — request bi tu choi truoc khi ghi'
```

**Ý nghĩa:** đây là **lý do tồn tại của storage version migration**, do chính API server nói ra.
Nó không cho bạn gỡ v1 chừng nào còn khả năng có object đang nằm ở v1 trong bộ lưu trữ — vì gỡ rồi
thì không còn cách nào đọc chúng lên. Bài [323](../323-storage-version-migration-vi.md) mô tả đúng
tình huống này ở câu mở: đổi storage schema "chỉ tác động tới bản ghi **được ghi mới** — object cũ
nằm im theo định dạng cũ cho tới khi có thứ ghi lại nó".

Nếu `grep` không thấy chữ `storedVersions`, mở `b9-tu-choi-bo-v1.txt` đọc nguyên văn thông báo
trước khi đi tiếp; xem [mục 4](#4-troubleshooting-của-lab-này).

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B9.4. Nâng cấp thủ công ba bước, rồi gỡ được v1

Bước 1 đã xong ở B9.2 (đặt v2 làm storage). Bước 2: **ghi lại mọi object hiện có**, ép bộ lưu trữ
dùng storage version hiện hành:

```bash
RV_TRUOC="$(kubectl get clusterwidget widget-toan-cluster \
  -o jsonpath='{.metadata.resourceVersion}')"

for w in $(kubectl get clusterwidgets -o name); do
  kubectl get "$w" -o yaml | kubectl replace -f - >/dev/null
  echo "da ghi lai $w"
done

RV_SAU="$(kubectl get clusterwidget widget-toan-cluster \
  -o jsonpath='{.metadata.resourceVersion}')"
echo "resourceVersion: truoc=$RV_TRUOC sau=$RV_SAU"

test "$RV_SAU" != "$RV_TRUOC" \
  && echo 'PASS: object da that su duoc ghi lai — gio no nam o storage version hien hanh'
```

Bước 3: gỡ v1 khỏi `status.storedVersions`:

```bash
kubectl patch "customresourcedefinitions/clusterwidgets.${LAB_GROUP}" \
  --subresource='status' --type='merge' -p '{"status":{"storedVersions":["v2"]}}'

SV_3="$(kubectl get "crd/clusterwidgets.${LAB_GROUP}" -o jsonpath='{.status.storedVersions}')"
echo "status.storedVersions sau khi vá=$SV_3"
test "$SV_3" = '["v2"]' \
  && echo 'PASS: storedVersions chi con v2'
```

Bây giờ mới gỡ được v1 khỏi `spec.versions`:

```bash
kubectl apply -f "$WK/b9-crd-bo-v1.yaml"
kubectl wait --for=condition=Established --timeout=120s "crd/clusterwidgets.${LAB_GROUP}"
rm -rf ~/.kube/cache/discovery

VER_3="$(kubectl get "crd/clusterwidgets.${LAB_GROUP}" -o jsonpath='{.spec.versions[*].name}')"
CW_CON="$(kubectl get clusterwidgets --no-headers | wc -l)"
CAN_CW3="$(kubectl auth can-i list "clusterwidgets.${LAB_GROUP}" --as="$SA_WP")"
echo "spec.versions=$VER_3 | so object con lai=$CW_CON | sa-doc-wp con doc duoc khong=$CAN_CW3"

test "$VER_3" = 'v2' \
  && echo 'PASS: da go duoc v1 sau khi hoan tat nang cap thu cong'
test "$CW_CON" -eq 1 \
  && echo 'PASS: object cu van doc duoc binh thuong qua v2 — khong mat du lieu'
kubectl get --raw "/apis/${LAB_GROUP}/v1/clusterwidgets" >/dev/null 2>&1 \
  || echo 'PASS: duong dan v1 khong con duoc phuc vu'
test "$CAN_CW3" = 'yes' \
  && echo 'PASS: ClusterRole van con hieu luc — RBAC ke ten resource, khong ke ten version'
```

**Ý nghĩa:** ba bước bạn vừa chạy chính là *Phương án 2* trong mục *Nâng cấp các object hiện có lên
storage version mới* của bài [377](../377-custom-resource-definition-versioning-vi.md). Chúng chạy
được ở đây vì cluster chỉ có **một** object; trên cluster thật với hàng nghìn object, bước 2 là
công việc cần tự động hóa — và đó đúng là thứ `StorageVersionMigration` của bài
[323](../323-storage-version-migration-vi.md) sinh ra để làm.

Gate cuối là một chi tiết đáng nhớ: **RBAC không có khái niệm version**. Rule của bạn ghi
`apiGroups` và `resources`, nên nó sống sót qua mọi lần đổi version của CRD.

**PASS:** sáu dòng `PASS:` của bước này xuất hiện.

---

## B10. Device plugin, extended resource và Topology Manager — ranh giới đọc được

**Mục đích:** đóng nốt hai điểm mở rộng còn lại của bản đồ B1 bằng bằng chứng, không bằng phỏng
đoán. Cluster lab không có GPU và không có node nhiều NUMA domain, nên bài
[184](../184-device-plugins-vi.md) và bài [313](../313-debug-topology-vi.md) chỉ kiểm chứng được
phần đọc được — nhưng phần đó đủ để trả lời checkpoint.

Toàn bộ B10 **chỉ đọc**. Không lệnh nào sửa cấu hình kubelet.

### B10.1. `Capacity` và `Allocatable`: không thiết bị nào được công bố

```bash
for n in "$MASTER" "$W1" "$W2"; do
  echo "$n capacity:    $(kubectl get node "$n" -o jsonpath='{.status.capacity}')"
  echo "$n allocatable: $(kubectl get node "$n" -o jsonpath='{.status.allocatable}')"
done | tee "$EV/b10-capacity.txt"

EXT_N="$(grep -o '"[a-z0-9.-]\{1,\}/[a-z0-9.-]\{1,\}"' "$EV/b10-capacity.txt" | wc -l)"
echo "so resource dang vendor-domain/resourcetype tren ba node=$EXT_N"

test "$EXT_N" -eq 0 \
  && echo 'PASS: khong node nao cong bo extended resource — khong device plugin nao dang chay'
grep -q 'hugepages' "$EV/b10-capacity.txt" \
  && echo 'PASS: chi co resource dung san (cpu, memory, pods, ephemeral-storage, hugepages-*)'
```

**Ý nghĩa:** bài [184](../184-device-plugins-vi.md) mô tả luồng đăng ký gồm hai chặng: plugin tự
đăng ký với **kubelet**, rồi **kubelet** mới công bố resource lên API server "như một phần của việc
cập nhật trạng thái node". Danh sách trên là đầu ra của chặng thứ hai. Không có tên nào dạng
`vendor-domain/resourcetype` nghĩa là chặng thứ nhất chưa từng xảy ra.

Bạn đã tự tay đi qua chặng thứ hai ở
[Lab 3c phần B5](LAB-3C-TAI-NGUYEN-QOS-VA-GIAN-DOAN.md#b5-extended-resource-tài-nguyên-do-bạn-tự-đặt-tên):
PATCH thẳng vào `/status/capacity` của Node để quảng bá `example.com/dongle`, rồi Pod xin nó bằng
`resources.limits`. Điều một device plugin làm thêm so với bạn ở đó là **tự động phát hiện thiết bị
và tự báo cáo sức khỏe của chúng** — chứ không phải một cơ chế API khác.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B10.2. Điểm đăng ký có sẵn, nhưng không ai dùng

```bash
ssh "$W2" 'sudo ls -l /var/lib/kubelet/device-plugins/' | tee "$EV/b10-device-plugins-dir.txt"

DP_SOCK2="$(ssh "$W2" 'sudo test -S /var/lib/kubelet/device-plugins/kubelet.sock && echo co || echo khong')"
DP_KHAC="$(ssh "$W2" 'sudo ls -1 /var/lib/kubelet/device-plugins/ | grep -c "\.sock$"')"
echo "kubelet.sock ton tai=$DP_SOCK2 | tong so file .sock trong thu muc=$DP_KHAC"

test "$DP_SOCK2" = 'co' \
  && echo 'PASS: kubelet dang phuc vu dich vu gRPC Registration tai duong dan hard-code'
test "$DP_KHAC" -ge 1 \
  && echo 'PASS: doc duoc so socket trong thu muc dang ky'
```

**Ý nghĩa:** đường dẫn `/var/lib/kubelet/device-plugins/` là **hard-code trong kubelet**, không đổi
theo `--root-dir` hay cấu hình nào khác — đó là lý do bài
[184](../184-device-plugins-vi.md) dặn khi triển khai plugin dạng DaemonSet thì phải mount đúng
đường dẫn này vào Pod. `kubelet.sock` là đầu kia của cuộc đăng ký: plugin gọi vào đó để khai tên
socket của nó, phiên bản API và `ResourceName`.

Nếu `DP_KHAC` lớn hơn 1, mở `b10-device-plugins-dir.txt` xem socket thừa là của ai — trên cluster
lab sạch thì chỉ có `kubelet.sock`.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B10.3. Đối chiếu với DRA của giai đoạn 13

```bash
DRA_N="$(kubectl api-resources --api-group=resource.k8s.io --no-headers 2>/dev/null | wc -l)"
kubectl api-resources --api-group=resource.k8s.io 2>&1 | tee "$EV/b10-dra.txt"
echo "so resource cua nhom resource.k8s.io duoc phuc vu=$DRA_N"

if [ "$DRA_N" -gt 0 ]; then
  SLICE_N="$(kubectl get resourceslices --no-headers 2>/dev/null | wc -l)"
  echo "so ResourceSlice=$SLICE_N"
  test "$SLICE_N" -eq 0 \
    && echo 'PASS: API DRA duoc phuc vu nhung khong driver nao cong bo thiet bi'
else
  echo 'PASS: nhom API DRA khong duoc phuc vu tren cluster nay'
fi
```

**Ý nghĩa:** hai cơ chế, hai chỗ dữ liệu nằm. Device plugin đẩy thiết bị vào **`status` của Node**
dưới dạng extended resource — số nguyên, không overcommit được, không chia sẻ được giữa các
container, và Pod xin bằng `resources.limits`. DRA đặt thiết bị vào **object API riêng** của nhóm
`resource.k8s.io`, nơi mô tả được thuộc tính và ràng buộc của từng thiết bị, và Pod xin bằng một
claim. Cả hai cùng tồn tại: device plugin là cơ chế đang chạy trong hầu hết cluster có GPU hiện nay,
DRA là hướng thay thế.

Đó là toàn bộ nội dung mà
[checkpoint giai đoạn 13](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao) yêu
cầu giải thích khi cluster không có GPU. [Lab 13 — DRA](LAB-13-DRA.md) đi sâu vào phía DRA; ở đây
bạn chỉ cần đối chiếu đủ để gọi tên khác biệt.

**PASS:** một dòng `PASS:` của nhánh tương ứng xuất hiện.

### B10.4. Topology Manager đang ở policy nào

```bash
for n in "$MASTER" "$W1" "$W2"; do
  kubectl get --raw "/api/v1/nodes/$n/proxy/configz" > "$EV/b10-configz-$n.json" 2>/dev/null \
    || echo "khong doc duoc configz cua $n"
done

pol() { grep -o "\"$2\":\"[^\"]*\"" "$EV/b10-configz-$1.json" | head -1 | sed 's/.*:"//;s/"$//'; }

TM_OK=0; MM_OK=0
# Khong dung pipe o day: vong lap phai chay trong chinh shell nay thi hai bien dem moi giu lai.
for n in "$MASTER" "$W1" "$W2"; do
  TM="$(pol "$n" topologyManagerPolicy)"
  MM="$(pol "$n" memoryManagerPolicy)"
  echo "$n topologyManagerPolicy='${TM:-<khong khai, mac dinh none>}' memoryManagerPolicy='${MM:-<khong khai, mac dinh None>}'"
  case "${TM:-none}" in none) TM_OK=$(( TM_OK + 1 )) ;; esac
  case "${MM:-None}" in None|none) MM_OK=$(( MM_OK + 1 )) ;; esac
done > "$EV/b10-topology-policy.txt"
cat "$EV/b10-topology-policy.txt"

NUMA_N="$(ssh "$W2" 'ls -d /sys/devices/system/node/node[0-9]* 2>/dev/null | wc -l')"
echo "so NUMA node tren $W2=$NUMA_N"
```

```bash
echo "so node co topologyManagerPolicy=none: $TM_OK/3 | memoryManagerPolicy=None: $MM_OK/3"

test "$TM_OK" -eq 3 \
  && echo 'PASS: ca ba kubelet deu o policy none — Topology Manager khong can chinh gi'
test "$MM_OK" -eq 3 \
  && echo 'PASS: ca ba kubelet deu o Memory Manager policy None'
test "$NUMA_N" -eq 1 \
  && echo 'PASS: may chi co mot NUMA domain — khong co ranh gioi nao de can'
```

**Ý nghĩa:** bài [313](../313-debug-topology-vi.md) là trang **tra cứu khi hỏng**, và ba con số
trên nói rõ vì sao nó không hỏng được ở đây. Với policy `none`, kubelet không sinh hint và không từ
chối Pod nào vì lý do topology, nên `TopologyAffinityError` không xuất hiện trong status của Pod.
Với một NUMA domain, mọi cách sắp đặt đều thỏa mãn, nên cả khi bật policy khác thì cũng chẳng có gì
để quan sát.

Giá trị của bài với bạn là **biết tra ở đâu** khi vận hành máy chủ vật lý nhiều socket: bốn nguồn
thông tin là status của Pod, log hệ thống, file trạng thái của kubelet, và device plugin resource
API — chính là điểm mở rộng số 7 mà B10.2 vừa đọc.

**Không đổi policy để "thử cho biết".** Đó là sửa `KubeletConfiguration` trên node đang chạy, thuộc
giai đoạn 20, và B11.4 sẽ bắt được nếu bạn làm.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

---

## B11. Cleanup và gate cuối

**Mục đích:** xóa mọi thứ lab tạo ra — kể cả thứ **nằm ngoài namespace** — rồi chứng minh bốn con
số tồn kho của B0.2 trở về đúng như cũ và cluster về đúng `04-metrics-ready`.

### B11.1. Xóa namespace, và xem thứ gì **không** chết theo

```bash
CRD_LAB_TRUOC="$(kubectl get crd -l lab=14 --no-headers | wc -l)"
CR_LAB_TRUOC="$(kubectl get clusterrole,clusterrolebinding -l lab=14 --no-headers | wc -l)"
CW_TRUOC_XOA_NS="$(kubectl get clusterwidgets --no-headers | wc -l)"
echo "truoc khi xoa namespace: crd lab=14 la $CRD_LAB_TRUOC, rbac lab=14 la $CR_LAB_TRUOC, clusterwidget la $CW_TRUOC_XOA_NS"

kubectl delete namespace "$NS" --wait=true --timeout=300s
```

```bash
NS_LEFT="$(kubectl get namespace "$NS" --ignore-not-found -o name 2>/dev/null | wc -l)"
NS_TAM_LEFT="$(kubectl get namespace lab-14-tam --ignore-not-found -o name 2>/dev/null | wc -l)"
CRD_LAB_SAU="$(kubectl get crd -l lab=14 --no-headers | wc -l)"
CR_LAB_SAU="$(kubectl get clusterrole,clusterrolebinding -l lab=14 --no-headers | wc -l)"
CW_SAU_XOA_NS="$(kubectl get clusterwidgets --no-headers | wc -l)"
echo "sau khi xoa namespace: crd lab=14 la $CRD_LAB_SAU, rbac lab=14 la $CR_LAB_SAU, clusterwidget la $CW_SAU_XOA_NS"

test "$NS_LEFT" -eq 0 && test "$NS_TAM_LEFT" -eq 0 \
  && echo 'PASS: khong con namespace nao cua lab'
test "$CRD_LAB_SAU" -eq "$CRD_LAB_TRUOC" \
  && test "$CR_LAB_SAU" -eq "$CR_LAB_TRUOC" \
  && test "$CW_SAU_XOA_NS" -eq "$CW_TRUOC_XOA_NS" \
  && echo 'PASS: xoa namespace KHONG don CRD, ClusterRole, ClusterRoleBinding hay object cluster-scoped'
```

**Ý nghĩa:** đây là lý do lab này có mục cleanup dài hơn mọi lab trước. Nếu bạn dừng ở đây và
chuyển sang lab sau, cluster của bạn mang theo hai CRD, một ClusterRole, một ClusterRoleBinding và
một custom object — tất cả đều **vô hình** với `kubectl get all` và với mọi lệnh có `-n`. Đó là
cách một cluster tích tụ rác trong đời thật.

**PASS:** hai dòng `PASS:` của bước này xuất hiện.

### B11.2. Xóa theo label, và so số lượng trước với sau

Kiểm **trước khi** xóa: đúng bốn object, đúng bốn cái tên đã khai ở
[bảng mục 2](#2-quy-ước-và-an-toàn):

```bash
kubectl get crd,clusterrole,clusterrolebinding -l lab=14 -o name | sort \
  | tee "$EV/b11-se-xoa.txt"

SE_XOA_N="$(wc -l < "$EV/b11-se-xoa.txt")"
echo "so object pham vi cluster se bi xoa=$SE_XOA_N"

test "$SE_XOA_N" -eq 4 \
  && echo 'PASS: dung bon object, khop bang khai bao o muc 2'
grep -q "webpages.${LAB_GROUP}" "$EV/b11-se-xoa.txt" \
  && grep -q "clusterwidgets.${LAB_GROUP}" "$EV/b11-se-xoa.txt" \
  && grep -q 'lab-14-doc-clusterwidget' "$EV/b11-se-xoa.txt" \
  && echo 'PASS: ba cai ten deu dung nhu du kien — khong CRD la nao lot vao selector'
```

**PASS:** hai dòng `PASS:` của bước kiểm này xuất hiện.

> Không đi tiếp khi hai gate trên chưa PASS. Nếu selector bắt nhầm một CRD của CNI hay của ingress
> controller thì lệnh xóa bên dưới sẽ **xóa luôn cấu hình mạng hoặc cấu hình ingress** của cluster.
> Trường hợp đó: dừng hẳn, restore cả ba VM về `04-metrics-ready`.

```bash
kubectl delete crd -l lab=14 --wait=true --timeout=300s
kubectl delete clusterrolebinding -l lab=14 --wait=true --timeout=120s
kubectl delete clusterrole -l lab=14 --wait=true --timeout=120s
```

```bash
CON_LAI="$(kubectl get crd,clusterrole,clusterrolebinding -l lab=14 --no-headers 2>/dev/null | wc -l)"
echo "so object pham vi cluster mang label lab=14 con lai=$CON_LAI"

test "$CON_LAI" -eq 0 \
  && echo 'PASS: khong con object pham vi cluster nao cua lab'
kubectl get --raw "/apis/${LAB_GROUP}/v2/clusterwidgets" >/dev/null 2>&1 \
  || echo 'PASS: duong dan cua API group da khong con duoc phuc vu'
kubectl get clusterwidgets 2>&1 | grep -qi 'server doesn\|not find\|unable to' \
  && echo 'PASS: kind cu khong con ton tai — xoa CRD xoa luon moi custom object cua no'
```

**Ý nghĩa:** ba gate này chứng minh đúng câu của bài
[378](../378-custom-resource-definitions-vi.md): "Khi bạn xóa một CustomResourceDefinition, server
sẽ gỡ cài đặt endpoint API RESTful và xóa toàn bộ custom object đang được lưu trong đó." Bạn không
phải xóa `widget-toan-cluster` riêng — nó biến mất cùng CRD. Đây cũng chính là lý do cảnh báo thứ
hai ở [mục 2](#2-quy-ước-và-an-toàn) tồn tại.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B11.3. Bốn con số tồn kho trở về đúng như B0.2

```bash
i=0
while [ "$i" -lt 30 ]; do
  APISVC_N1="$(kubectl get apiservice --no-headers | wc -l)"
  test "$APISVC_N1" -eq "$APISVC_N0" && break
  i=$(( i + 1 )); sleep 2
done

CRD_N1="$(kubectl get crd --no-headers | wc -l)"
CR_N1="$(kubectl get clusterrole --no-headers | wc -l)"
CRB_N1="$(kubectl get clusterrolebinding --no-headers | wc -l)"
APISVC_N1="$(kubectl get apiservice --no-headers | wc -l)"

{
  echo "=== $(date -Is) — ton kho pham vi cluster sau Lab 14 ==="
  echo "crd:                truoc=$CRD_N0    sau=$CRD_N1"
  echo "clusterrole:        truoc=$CR_N0     sau=$CR_N1"
  echo "clusterrolebinding: truoc=$CRB_N0    sau=$CRB_N1"
  echo "apiservice:         truoc=$APISVC_N0 sau=$APISVC_N1 (sau $i vong cho)"
} | tee "$EV/b11-ton-kho.txt"

test "$CRD_N1" -eq "$CRD_N0" \
  && echo 'PASS: so CRD tro ve dung con so cu'
test "$CR_N1" -eq "$CR_N0" && test "$CRB_N1" -eq "$CRB_N0" \
  && echo 'PASS: so ClusterRole va ClusterRoleBinding tro ve dung con so cu'
test "$APISVC_N1" -eq "$APISVC_N0" \
  && echo 'PASS: APIService tu dang ky cua CRD da bien mat theo CRD'
```

**Ý nghĩa:** đây là **gate quan trọng nhất của Lab 14**. Xóa theo label là đủ khi bạn nhớ gắn label;
so số lượng là thứ bắt được cái bạn quên gắn. Hai kỹ thuật bổ sung cho nhau, và trên object phạm vi
cluster thì thiếu cái nào cũng không đủ.

Con số `apiservice` cần vòng chờ vì object đó do kube-apiserver tự dọn — **thời gian phụ thuộc cấu
hình control plane**. Nếu sau cả vòng chờ nó vẫn lệch, mở `b2-apiservice.txt` và
`kubectl get apiservice` để tìm dòng thừa trước khi kết luận.

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B11.4. Cấu hình node và control plane không hề đổi

```bash
{
  echo "$MASTER kubelet $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for n in "$W1" "$W2"; do
    echo "$n kubelet $(ssh "$n" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done
  for f in kube-apiserver kube-scheduler kube-controller-manager; do
    echo "$MASTER $f $(sudo sha256sum /etc/kubernetes/manifests/$f.yaml | awk '{print $1}')"
  done
} | tee "$EV/b11-config-sha.txt"

diff -u "$EV/b0-config-sha.txt" "$EV/b11-config-sha.txt" \
  && echo 'PASS: sau file cau hinh khong he doi trong suot lab' \
  || echo 'FAIL: cau hinh da bi sua — xem muc 4'
```

**Ý nghĩa:** gate này khóa lời hứa của B10: lab đọc `topologyManagerPolicy` chứ không đổi nó. Nếu
`diff` báo khác, cluster đã lệch khỏi `04-metrics-ready` và **không được** mang sang lab sau.

**PASS:** dòng `PASS:` xuất hiện, không dòng `FAIL:` nào.

### B11.5. Hạ tầng của mốc `04-metrics-ready` còn nguyên

```bash
SC_N1="$(kubectl get storageclass --no-headers | wc -l)"
SC_DEF1="$(kubectl get storageclass \
  -o jsonpath='{range .items[*]}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' \
  | grep -c '^true$')"
IC_N1="$(kubectl get ingressclass --no-headers 2>/dev/null | wc -l)"
MS_AVAIL1="$(kubectl get apiservice v1beta1.metrics.k8s.io \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')"
PV_N1="$(kubectl get pv --no-headers 2>/dev/null | wc -l)"
PVC_N1="$(kubectl get pvc -A --no-headers 2>/dev/null | wc -l)"
TOP_N="$(kubectl top node --no-headers 2>/dev/null | wc -l)"
echo "storageclass=$SC_N1 (mac dinh=$SC_DEF1) | ingressclass=$IC_N1 | metrics=$MS_AVAIL1"
echo "pv=$PV_N1 pvc=$PVC_N1 | so dong kubectl top node=$TOP_N"

test "$SC_DEF1" -eq 1 && test "$IC_N1" -ge 1 && test "$MS_AVAIL1" = 'True' \
  && echo 'PASS: StorageClass mac dinh, ingress controller va metrics-server deu con nguyen'
test "$PV_N1" -eq 0 && test "$PVC_N1" -eq 0 \
  && echo 'PASS: khong PV hay PVC nao — lab 14 khong dung toi tang luu tru'
test "$TOP_N" -eq 3 \
  && echo 'PASS: kubectl top node van in du ba node — dung dinh nghia moc 04-metrics-ready'
```

**PASS:** ba dòng `PASS:` của bước này xuất hiện.

### B11.6. Dọn workspace và gate ngắn A5.5

```bash
rm -f "$WK"/b3-*.yaml "$WK"/b4-*.yaml "$WK"/b5-*.yaml "$WK"/b6-*.yaml \
      "$WK"/b7-*.yaml "$WK"/b8-*.yaml "$WK"/b9-*.yaml \
      "$WK"/reconcile-*.yaml "$WK"/ctrl.kubeconfig
rmdir "$WK"
unset TOK_CTRL
```

`rmdir` cố ý không dùng `-rf`: nó fail nếu thư mục còn file lạ, và `test ! -e` bên dưới biến điều
đó thành gate thay vì im lặng bỏ qua. Thư mục `~/lab-evidence/14/` **giữ lại** — đó là bằng chứng.

```bash
test ! -e "$WK" && echo 'PASS: manifest tam va kubeconfig chua token da xoa het'
grep -rl 'eyJ' "$EV" 2>/dev/null | tee "$EV/b11-quet-token.txt"
test ! -s "$EV/b11-quet-token.txt" \
  && echo 'PASS: khong file bang chung nao chua chuoi giong JWT'
```

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default

READY_N="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}' \
  | grep -c '^True$')"
echo "so node Ready=True: $READY_N"
test "$READY_N" -eq 3 && echo 'PASS: ca ba node Ready'

{
  echo "=== $(date -Is) — trang thai sau Lab 14 ==="
  echo '--- ton kho pham vi cluster'
  echo "crd=$CRD_N1 clusterrole=$CR_N1 clusterrolebinding=$CRB_N1 apiservice=$APISVC_N1"
  echo '--- crd dang co, kem API group va scope'
  kubectl get crd -o custom-columns='NAME:.metadata.name,GROUP:.spec.group,SCOPE:.spec.scope' \
    --no-headers | sort
  echo '--- storageclass'; kubectl get storageclass -o wide
  echo '--- ingressclass'; kubectl get ingressclass 2>&1
  echo '--- apiservice metrics'; kubectl get apiservice v1beta1.metrics.k8s.io 2>&1
  echo '--- persistentvolume'; kubectl get pv 2>&1
  echo '--- namespaces'; kubectl get namespaces
} | tee "$EV/b11-sau.txt"

diff -u "$EV/b0-truoc.txt" "$EV/b11-sau.txt" > "$EV/b11-diff.txt" 2>&1 || true
grep -c '^[+-]' "$EV/b11-diff.txt"
```

**PASS:** ba node `Ready`; dòng `PASS: readyz ok` xuất hiện; lệnh field selector trả
`No resources found`; CoreDNS đủ replica `READY`; `default` không có Pod; dòng
`PASS: ca ba node Ready` xuất hiện; danh sách namespace không còn `lab-14` lẫn `lab-14-tam`; danh
sách CRD trong `b11-sau.txt` **không còn** dòng nào thuộc API group của lab.

Cluster đã trở về `04-metrics-ready`. **Lab 14 không tạo snapshot mới** — để ba VM nguyên trạng
đang chạy hoặc tắt tùy bạn; lab sau tự bật máy theo
[A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab).

---

## 3. Checkpoint 14

Không đánh dấu lab hoàn tất chỉ vì command chạy hết. Tự trả lời, không nhìn tài liệu:

- [ ] Bạn `kubectl apply` một custom resource và không có gì xảy ra trong cluster. Kubernetes hỏng
      à? Bạn **đang có** thứ gì và **chưa có** thứ gì? Nêu đúng thành phần còn thiếu, và nói vì sao
      `kubectl get`, `kubectl edit` và RBAC vẫn hoạt động bình thường dù thiếu nó.
- [ ] Kể ba tính chất phân biệt một **vòng lặp điều khiển** với một script chạy một lần, và với mỗi
      tính chất nói bạn đã chứng minh nó bằng thí nghiệm nào ở B8.4. Nếu chạy vòng lặp mười lần
      liên tiếp khi mọi thứ đã khớp thì object bị ghi mấy lần, và bạn đọc trường nào để biết?
- [ ] Bạn xóa custom resource và object phụ thuộc của nó cũng biến mất, **dù vòng lặp không chạy**.
      Ai đã xóa nó, dựa vào cái gì, và điều đó nằm ở trường nào của object phụ thuộc? Việc gì mà
      cơ chế đó **không** làm được, và một operator thật dùng cơ chế nào để làm việc đó?
- [ ] Câu bẫy: cluster của bạn có `metrics.k8s.io` và có API group của CRD, cả hai đều "mở rộng
      Kubernetes API". Bạn đọc **trường nào** của object nào để phân biệt chúng, và với mỗi cái thì
      **ai** phục vụ và **ai** lưu trữ dữ liệu? Vì sao `metrics.k8s.io` **không thể** là một CRD?
- [ ] Bạn cần thêm bốn trường vào một API dùng nội bộ công ty, không cần kho lưu trữ riêng. Chọn CRD
      hay aggregated API, và vì sao? Nếu sáu tháng sau cần một subresource `logs` thì quyết định đó
      còn đứng vững không? Kể thêm ba khả năng nữa mà aggregated API có còn CRD không có.
- [ ] Bạn khai một trường trong custom resource mà schema không có. Với `--validate=strict` thì sao,
      với `--validate=ignore` thì sao, và sau khi object được lưu thì đọc lại trường đó ra gì? Hai
      hành vi ấy xảy ra ở **hai tầng khác nhau** — tầng nào là tầng nào?
- [ ] Trước khi bật subresource `status`, ai ghi được `status`? Sau khi bật thì lệnh `kubectl patch`
      cũ đi đâu về đâu? Nêu **hai** thứ mà việc bật subresource thay đổi, và nói vì sao RBAC lại là
      lý do chính khiến người ta bật nó.
- [ ] `metadata.generation` và `status.observedGeneration` tạo thành một giao thức. Ai tăng số nào,
      khi nào, và khi hai số lệch nhau thì bạn kết luận điều gì về cluster? Ghi `status` có làm tăng
      `generation` không?
- [ ] Bạn cấp cho một ServiceAccount ClusterRole `view` dựng sẵn rồi bảo nó đọc custom resource của
      bạn. Nó đọc được không, vì sao, và bạn đọc gì trong chính nội dung của role để chứng minh?
      Điều này nói gì về việc đóng gói một operator để phát hành?
- [ ] Bạn xóa namespace chứa toàn bộ đồ của một sản phẩm dùng CRD. Kể **ba loại object** vẫn còn lại
      trong cluster sau lệnh đó. Chúng có hiện ra trong `kubectl get all` không? Bạn dùng hai kỹ
      thuật nào để cleanup cho đủ, và vì sao thiếu một trong hai là chưa đủ?
- [ ] API server từ chối không cho bạn gỡ version `v1` khỏi một CRD. Nó viện lý do ở trường nào, vì
      sao ràng buộc đó tồn tại, và ba bước nào đưa bạn tới chỗ gỡ được? Trên cluster có hàng nghìn
      object thì bước nào là bước tốn công nhất và công cụ nào sinh ra để làm nó?
- [ ] Node của bạn không công bố resource nào dạng `vendor-domain/resourcetype`. Bạn kết luận gì, và
      **hai chặng** nào của luồng đăng ký device plugin đã không xảy ra? Device plugin đặt dữ liệu
      thiết bị ở đâu, DRA đặt ở đâu, và vì sao khác biệt đó quan trọng?

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng sau, không nhìn tài liệu:

1. **Từ một file YAML tới một API khai báo.** Bắt đầu từ lúc bạn `kubectl apply` file CRD. Kể đủ
   sáu chặng: API server làm gì để endpoint mới tồn tại và bạn đọc **hai condition nào** để biết nó
   xong; vì sao `kubectl` có thể vẫn chưa thấy kiểu mới và bạn xử lý bằng cách nào; bạn tạo một
   custom object thì nó được **ai** kiểm và **ai** lưu; ở thời điểm này bạn đang có chính xác cái
   gì; bạn thêm gì để nó thành API khai báo và thứ đó cần **danh tính** cùng **quyền** nào; và cuối
   cùng, chuyện gì xảy ra khi ai đó xóa object phụ thuộc mà controller tạo ra. Kết bằng câu trả lời
   cho: nếu vòng lặp của bạn chết trong ba ngày thì cluster hỏng ở đâu, và **không** hỏng ở đâu.
2. **Cluster này đứng ở đâu trên bản đồ mở rộng.** Đi qua bảy điểm mở rộng của bài 177 và với mỗi
   điểm nói một hiện vật bạn đã sờ được trong lab, hoặc nói rõ điểm đó đang trống và bạn biết bằng
   bằng chứng gì. Rồi trả lời ba câu ranh giới: `metrics.k8s.io` và API group của bạn khác nhau ở
   **ai phục vụ và ai lưu**, đọc ở đâu ra; provisioner của StorageClass nằm ở điểm mở rộng nào và
   vì sao **không** phải điểm storage plugin; cluster có device plugin không, và nếu ngày mai cắm
   một GPU vào `lab-k8s-worker2` thì cần thêm chính xác thứ gì để Pod xin được nó. Cuối cùng, nói
   rõ **một chủ đề của giai đoạn 14 mà lab này cố ý không làm** và nó được trả ở đâu.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm *CRD* với *aggregated API*, *custom resource*
với *custom controller*, *`spec` trong schema* với *subresource `status`*, *`spec.versions`* với
*`status.storedVersions`*, hay *garbage collection theo owner reference* với *việc của controller* —
Lab 14 và [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) hoàn tất.

Lab này **không phát sinh nợ mới** và **không trả nợ nào**; xem
[sổ nợ lab](README.md#5-sổ-nợ-lab). Subresource `scale` không nằm trong lab vì nó chỉ có ý nghĩa
khi ghép với HorizontalPodAutoscaler, và HPA là nội dung của
[Lab 11b](LAB-11B-HPA-VA-VPA.md). Nhóm thực hành đầy đủ của giai đoạn 14 nằm ở
[giai đoạn 28](../00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes), với bài
[378](../378-custom-resource-definitions-vi.md) là trang xương sống; những phần cố ý không làm còn
lại đều nằm trong bảng lý do ở [mục 1.1](#11-ánh-xạ-tài-liệu-sang-bài-thực-hành).

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường nằm ở
[mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Bảng dưới chỉ liệt kê
sự cố phát sinh từ nội dung bài học 14.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| **B3.2 / B5.3 / B6.3: CRD đã tạo nhưng `kubectl get webpages` báo `the server doesn't have a resource type`** | `kubectl get crd -l lab=14`; `kubectl get --raw "/apis/$LAB_GROUP/v1"`; `ls ~/.kube/cache/discovery` | Phân biệt hai tầng trước khi sửa. **(1)** Nếu `--raw` trả về `APIResourceList` thì **server đã đúng**, vấn đề nằm ở **cache discovery phía client**: `kubectl` lưu danh sách kiểu trong `~/.kube/cache/discovery/<host>/` và dùng lại bản đã lưu, nên kiểu vừa tạo có thể chưa có trong đó — thời gian sống của cache **phụ thuộc cấu hình `kubectl`**, đừng ngồi chờ. Chạy `rm -rf ~/.kube/cache/discovery` rồi `kubectl api-resources --api-group="$LAB_GROUP"`, sau đó thử lại. **(2)** Nếu `--raw` trả `404` thì CRD chưa `Established`: đọc `kubectl get crd <ten> -o jsonpath='{.status.conditions}'`; condition `NamesAccepted=False` nghĩa là tên bạn xin đang xung đột với resource khác — đổi `plural`, `singular`, `kind`, `shortNames` hoặc `categories` rồi apply lại. **(3)** Nếu bạn gõ thiếu group ở dạng đầy đủ, dùng `kubectl get webpages.$LAB_GROUP` — dạng `<plural>.<group>` không bao giờ nhập nhằng |
| B5.3: `kubectl get lab14` không trả gì trong khi `kubectl get wp` chạy được | `kubectl api-resources -o wide \| grep webpages`; kiểm cột `CATEGORIES` | Category cũng nằm trong thông tin discovery, nên nó dính đúng cache ở dòng trên. Làm mới cache rồi thử lại. Nếu cột `CATEGORIES` trống thì `categories:` chưa vào CRD — kiểm nó nằm dưới `spec.names`, không phải dưới `spec.versions` |
| B4.3: `kubectl apply` báo `unknown field` trong khi bạn muốn thấy cơ chế cắt tỉa | đọc `$EV/b4-truong-la-strict.txt` | Đây **không phải lỗi**, đó là kết quả mong đợi của `--validate=strict`. Muốn thấy cắt tỉa thì chạy đúng lệnh thứ hai của B4.3 với `--validate=ignore`. Đừng sửa schema để "cho nó qua" — thêm trường vào schema là đổi hợp đồng API, không phải cách chữa lỗi gõ sai |
| B7.3: patch `--subresource=status` báo `the server could not find the requested resource` | `kubectl get crd <ten> -o jsonpath='{.spec.versions[0].subresources}'`; `kubectl get --raw "/apis/$LAB_GROUP/v1" \| tr ',' '\n' \| grep status` | Subresource chưa được bật. Kiểm hai dòng `subresources:` và `status: {}` nằm **bên trong một phần tử của `versions`**, cùng cấp với `schema:` — đặt nhầm lên `spec` là lỗi thường gặp nhất. Chạy lại `diff` của B7.2 để thấy chính xác bạn thêm gì |
| B8.3: `reconcile` trả mã khác 0 và status không đổi | `kctl auth can-i patch "webpages.$LAB_GROUP" --subresource=status`; đọc `$EV/b8-status-bi-tu-choi.txt` | Nếu trả `no`: Role chưa có rule cho `webpages/status` — apply `$WK/b8-role-du.yaml` rồi chạy lại vòng chờ ở B8.3. Nếu trả `yes` mà vẫn hỏng: token trong `$KCFG` thuộc phiên shell cũ; tạo lại bằng `kubectl create token sa-lab14-controller -n "$NS"` và dựng lại kubeconfig theo B8.2 |
| B8.3: `kctl` báo `Unauthorized` hoặc `error: You must be logged in` | `kctl auth whoami`; `stat -c '%a' "$KCFG"` | Token của ServiceAccount **có thời hạn**. Sinh token mới và đặt lại `set-credentials` theo B8.2; đừng nới quyền để "cho chạy". Nếu `auth whoami` ra đúng tên mà vẫn lỗi thì `--certificate-authority` trỏ sai file — dùng đúng `/etc/kubernetes/pki/ca.crt` |
| B8.5: xóa custom resource nhưng ConfigMap **không** biến mất | `kubectl get configmap web-<ten> -o jsonpath='{.metadata.ownerReferences}'` | So `uid` trong owner reference với `uid` thật của custom resource. Sai `uid` thì garbage collector coi owner là đã biến mất theo kiểu khác và có thể không xóa — đây đúng là cái bẫy [Lab 1c](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md#b2-owner-reference-và-garbage-collection) đã cảnh báo. Xóa ConfigMap bằng tay rồi chạy lại `reconcile` để nó gắn owner reference đúng |
| B9.3: `kubectl apply` bỏ v1 **không** bị từ chối như mong đợi | `kubectl get crd <ten> -o jsonpath='{.status.storedVersions}'`; `grep -cE '^  - name: v[12]$' "$WK/b9-crd-bo-v1.yaml"` | Nếu `grep` ra `2` thì `sed` chưa cắt được khối v1 — mở file và xóa tay khối `- name: v1` cho tới hết `additionalPrinterColumns` của nó. Nếu `storedVersions` đã là `["v2"]` thì bạn đã lỡ chạy B9.4 trước B9.3; đó là thứ tự sai nhưng vô hại, ghi vào evidence rồi đi tiếp |
| B9.4: `kubectl replace` báo lỗi về `resourceVersion` | `kubectl get <object> -o yaml \| head -20` | Object đã bị ai đó ghi giữa lúc bạn `get` và lúc bạn `replace`. Chạy lại đúng vòng lặp của B9.4 — nó `get` lại từ đầu mỗi vòng |
| **Bạn lỡ xóa một CRD không thuộc lab** | `kubectl get crd`; `kubectl -n kube-system get pods`; `kubectl get pods -A` | Dừng mọi thứ. Xóa CRD xóa luôn toàn bộ custom object trong đó, và với CRD của CNI hay ingress controller thì đó là mất cấu hình, không phải mất một object. **Tắt cả ba VM và restore cả ba về `04-metrics-ready`** — xem ghi chú cuối [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường). Đừng cố cài lại add-on bằng tay: bạn sẽ dựng ra một cluster không còn khớp mốc snapshot |
| B11.1: `kubectl delete namespace` treo ở `Terminating` | `kubectl get namespace <ten> -o jsonpath='{.spec.finalizers}{"\n"}{.status.conditions}'`; `kubectl api-resources --verbs=list --namespaced -o name` | Namespace chỉ biến mất khi mọi kind namespaced trong nó đã dọn xong. Nếu bạn còn một CRD **đã bị xóa** trong khi API group của nó vẫn nằm trong discovery, namespace sẽ chờ. Trong lab này thứ tự đúng là **xóa namespace trước, CRD sau** — đúng như B11.1 rồi B11.2. Nếu đã lỡ đảo thứ tự, đọc `status.conditions` để biết kind nào đang chặn |
| B11.3: `apiservice` sau lab nhiều hơn trước | `kubectl get apiservice \| grep "$LAB_GROUP"`; `kubectl get crd -l lab=14` | APIService tự đăng ký chỉ biến mất khi CRD của nó đã biến mất. Kiểm CRD đã xóa hết chưa; nếu đã hết mà APIService vẫn còn sau cả vòng chờ, đọc `kubectl describe apiservice <ten>` — **không** xóa nó bằng tay trừ khi nó thuộc API group của lab |
| B11.4: `diff` báo checksum kubelet đã đổi | `sudo cat /var/lib/kubelet/config.yaml \| grep -iE 'topology\|memoryManager'` | Bạn đã sửa cấu hình kubelet, gần như chắc chắn ở B10.4. Lab 14 chỉ đọc. Cluster đã lệch mốc: restore cả ba VM về `04-metrics-ready` trước khi sang lab sau, và ghi vào evidence rằng baseline đã lệch |

---

## 5. Nguồn chính thức

Các phần giải thích command trong thân bài ưu tiên snapshot tài liệu Kubernetes v1.35
(`https://v1-35.docs.kubernetes.io/`) để hành vi, trường và flag khớp minor version của cluster.

- [Kubernetes v1.35 — Extending Kubernetes](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/)
- [Kubernetes v1.35 — Extending the Kubernetes API](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/api-extension/)
- [Kubernetes v1.35 — Custom Resources](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Kubernetes v1.35 — Kubernetes API Aggregation Layer](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)
- [Kubernetes v1.35 — Operator pattern](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Kubernetes v1.35 — Compute, Storage, and Networking Extensions](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/)
- [Kubernetes v1.35 — Device Plugins](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Kubernetes v1.35 — Extend the Kubernetes API with CustomResourceDefinitions](https://v1-35.docs.kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [Kubernetes v1.35 — Versions in CustomResourceDefinitions](https://v1-35.docs.kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)
- [Kubernetes v1.35 — Migrate Kubernetes Objects Using Storage Version Migration](https://v1-35.docs.kubernetes.io/docs/tasks/manage-kubernetes-objects/storage-version-migration/)
- [Kubernetes v1.35 — Troubleshooting Topology Management](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-cluster/topology/)
- [Kubernetes v1.35 — Controllers](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/controller/)
- [Kubernetes v1.35 — Owners and Dependents](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/)
- [Kubernetes v1.35 — Using RBAC Authorization](https://v1-35.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Kubernetes v1.35 — Field Validation](https://v1-35.docs.kubernetes.io/docs/reference/using-api/api-concepts/#field-validation)
- [Kubernetes v1.35 — `kubectl api-resources`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_api-resources/)
- [Kubernetes v1.35 — `kubectl patch`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_patch/)
- [Kubernetes v1.35 — `kubectl auth can-i`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/)
- [Kubernetes API — CustomResourceDefinition v1](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/extend-resources/custom-resource-definition-v1/)
- [Kubernetes API — APIService v1](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/cluster-resources/api-service-v1/)
