# Lab 14 — CRD và Operator

> **Điểm bắt đầu:** snapshot `04-metrics-ready` — mốc do Lab 11a tạo, xem
> [chuỗi snapshot](README.md#3-chuỗi-snapshot). Repo `~/bookstore` ở tag `lab-13`, hoặc `lab-12` nếu
> bạn bỏ [Lab 13](LAB-13-DRA.md) tùy chọn. Vừa restore snapshot thì khôi phục repo từ bundle mới nhất.
> **Điểm kết thúc:** teardown trả cluster về đúng `04-metrics-ready`, **không tạo snapshot mới**.
> Repo `~/bookstore` có tag `lab-14` — **tag cuối của dự án** — và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 21 — Operator sao lưu và nghiệm thu dự án, sprint cuối của
> [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 13 — DRA](LAB-13-DRA.md) (tùy chọn), hoặc
> [Lab 12 — Vận hành vòng đời node](LAB-12-VAN-HANH-VONG-DOI-NODE.md).
> **Sau lab này:** dự án đóng. [Lab 15](LAB-15-NODE-WINDOWS.md) nằm ngoài dự án; phần tiếp theo của
> lộ trình là [Phần II — Vận hành cluster](../00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng mục
[Giai đoạn 14 — Khả năng mở rộng](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng): bảy bài
khái niệm [177](../177-extend-kubernetes-vi.md), [178](../178-api-extension-vi.md),
[179](../179-custom-resources-vi.md), [180](../180-apiserver-aggregation-vi.md),
[181](../181-operator-vi.md), [182](../182-compute-storage-net-vi.md),
[184](../184-device-plugins-vi.md), cộng hai bài thực hành [313](../313-debug-topology-vi.md) và
[323](../323-storage-version-migration-vi.md). Mọi ticket dựa vào hợp đồng kỹ thuật ở
[mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật), bố cục repo ở
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint) và kế hoạch gốc ở
[mục 7.2](DU-AN-BOOKSTORE.md#sprint-21--lab-14-operator-sao-lưu).

Sprint này làm hai việc. Một: đưa sao lưu theo yêu cầu lên **API khai báo của chính dự án** —
custom resource `DbBackup` cộng một controller `backup-operator` (yêu cầu `R19`). Hai: **nghiệm thu
toàn bộ dự án** — đối chiếu từng yêu cầu `R1`–`R19` với bằng chứng, đóng sổ nợ kỹ thuật, viết retro.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Image của dự án nằm ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00); lab này không chép lại
con số phiên bản nào trong phần văn bản, **không cài thêm gì** vào cluster, không cài toolchain lập
trình hay operator framework.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi hệ thống Bookstore đạt trạng thái cuối của dự án — **sao lưu theo yêu cầu đi qua
API của chính dự án** — **và** bạn chứng minh, giải thích được:

- Bảy điểm mở rộng của bài 177, và mỗi thứ cluster Bookstore đang dùng (Calico, Traefik,
  local-path-provisioner, metrics-server) nằm ở điểm nào; vì sao provisioner của StorageClass là
  *controller* chứ không phải *storage plugin*.
- Có **đúng hai cách** thêm custom resource, đọc ra được cả hai đang chạy trên cluster: một API group
  do kube-apiserver tự phục vụ và lưu, một API group được proxy sang server khác.
- Vì sao `DbBackup` là CRD chứ không phải aggregated API, và là custom resource chứ không phải thêm
  một key vào ConfigMap `backup-status`.
- Schema của CRD là **hợp đồng do API server cưỡng chế**: object sai bị từ chối, trường lạ bị cắt
  tỉa, giá trị mặc định được điền; subresource `status` tách hai đường ghi.
- Kind mới **không** tự có trong role dựng sẵn: `view` không đọc được `DbBackup`; quyền phải cấp tường
  minh, `dbbackups` và `dbbackups/status` là hai resource khác nhau trong RBAC; ResourceQuota đếm được
  số object của CRD.
- **CRD không có controller chỉ là kho dữ liệu** — chứng minh bằng chính operator của bạn khi nó
  dừng, và chứng minh vòng lặp **hội tụ** khi nó chạy lại mà không mất yêu cầu nào.
- `backup-operator` đọc `spec`, tạo Job, ghi `status`; Job của nó đi qua đủ bốn lớp chính sách của
  các sprint trước: Pod Security `restricted`, NetworkPolicy, quota, PriorityClass `bookstore-batch`.
- Phần dọn dẹp theo `ownerReferences` là công của garbage collector, không phải của controller; việc
  phải xảy ra **trước** khi xóa thì cần cơ chế khác.
- `status.storedVersions`: vì sao API server từ chối gỡ một version còn nằm trong đó, và quy trình ghi
  lại object để chuyển storage version — cái `StorageVersionMigration` của bài 323 tự động hóa.
- Cluster không có device plugin nào và không thể kích hoạt nhánh debug Topology Manager — đọc đúng
  bằng chứng, và nói được một tính năng dùng GPU sẽ cần gì trên hệ thống này.
- Hệ thống cuối đáp ứng từng yêu cầu `R1`–`R19` bằng bằng chứng nào; nợ kỹ thuật nào đã trả, nào
  chấp nhận có điều kiện, nào chuyển sang Phần II, và nợ lab nào còn mở.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 14 | Ticket kiểm chứng |
| --- | --- |
| [177 — Mở rộng Kubernetes](../177-extend-kubernetes-vi.md) | B1 — bảy điểm của *Chú giải cho hình vẽ* đọc trên cluster thật, webhook so với binary plugin, ADR *Thay đổi các resource có sẵn*; B4 — mẫu controller: đọc `.spec`, làm, cập nhật `.status`; B5 — controller là một điểm lỗi nhưng nằm ngoài đường request |
| [178 — Mở rộng Kubernetes API](../178-api-extension-vi.md) | B1 — đúng hai cách, đếm trên cluster; B2 — với CRD, control plane phục vụ và lưu, không thêm API server nào |
| [179 — Tài nguyên tùy chỉnh](../179-custom-resources-vi.md) | B1 — ADR chọn CRD so với aggregated API, ConfigMap so với custom resource; B2 — CRD, validation, defaulting, cắt tỉa, status subresource, cột in; B3 — RBAC không tự cấp quyền cho kind mới, mục *Lưu trữ* qua quota; B5 — *Custom controller*: thiếu controller thì chỉ là kho dữ liệu; B6 — storage version |
| [180 — Tầng tổng hợp API](../180-apiserver-aggregation-vi.md) | B1 — APIService của metrics-server có `.spec.service`, API group của CRD thì không; B3 — câu hỏi vì sao quota đếm được `DbBackup` mà không đếm được object của aggregated API |
| [181 — Mẫu Operator](../181-operator-vi.md) | B4 — `backup-operator` dưới ServiceAccount riêng, đối chiếu bảy bước của `SampleDB`; B5 — diễn tập controller vắng mặt, Job mất, owner bị xóa |
| [182 — Phần mở rộng tính toán, lưu trữ, mạng](../182-compute-storage-net-vi.md) | B1 — ba nhóm đối chiếu với hạ tầng thật: CNI là nhóm bắt buộc, provisioner không phải CSI; B7 — nhóm thiết bị |
| [184 — Device Plugin](../184-device-plugins-vi.md) | B7 — `Capacity`/`Allocatable` không có resource `vendor-domain/resourcetype`, socket đăng ký có nhưng không ai đăng ký, đối chiếu DRA, ADR GPU cho dự án |
| [313 — Khắc phục sự cố Topology Management](../313-debug-topology-vi.md) | B7 — đọc `topologyManagerPolicy`, `memoryManagerPolicy` hiệu lực qua `configz` và số NUMA node để chứng minh nhánh debug không kích hoạt được |
| [323 — Storage Version Migration](../323-storage-version-migration-vi.md) | B6 — thêm `v1` cho `DbBackup`, `status.storedVersions`, API server từ chối gỡ version còn lưu, ghi lại object rồi cập nhật `storedVersions` |

Các phần dưới đây **không kiểm chứng được trong lab này** — ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài 181 — viết operator thật bằng Go, kubebuilder, Kopf hay Operator Framework, và chạy nó như một Deployment trong cluster | Cần cài toolchain, framework và **build image** — ngoài baseline. B4 làm phần **cơ chế** bằng công cụ có sẵn: vòng lặp `kubectl` chạy trên master dưới danh tính ServiceAccount có quyền tối thiểu. Thiếu nơi chạy thường trực trong cluster là nợ kỹ thuật mới *operator ngoài cluster* |
| Bài 180 — dựng một extension API server của riêng bạn | Phải viết và build binary cùng image, rồi cấu hình cụm cờ `--requestheader-*` cho kube-apiserver. Đó là bài [374](../374-configure-aggregation-layer-vi.md) và [380](../380-setup-extension-api-server-vi.md) của [giai đoạn 28](../00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes). B1 chỉ đọc phần **đọc được** trên extension API server đang chạy thật là metrics-server |
| Bài 179 — conversion webhook, validating/mutating webhook cho custom resource | Webhook đòi một dịch vụ HTTPS trong cluster kèm certificate và `caBundle` — thêm hạ tầng. B6 dùng `conversion.strategy: None`, an toàn khi mọi version chung một schema |
| Bài 184 — chạy device plugin thật và cấp GPU cho Pod; ba endpoint gRPC của Pod Resources API | Ba VM không có GPU, NIC hiệu năng cao hay FPGA — giới hạn phần cứng. Gọi gRPC cần viết client protobuf. B7 kiểm chứng phần đọc được |
| Bài 313 — dựng lỗi `TopologyAffinityError`, đọc `memory_manager_state` | Cần policy khác `none` (sửa `KubeletConfiguration` — [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy)) **và** máy nhiều NUMA domain. Lab không sửa cấu hình kubelet; gate cuối so checksum |
| Bài 323 — object `StorageVersionMigration`, luồng mã hóa lại Secret, đọc etcd bằng `etcdctl … \| hexdump -C` | Cần feature gate `StorageVersionMigrator` và runtime config trên API server (giai đoạn 20), `EncryptionConfiguration` ([nợ lab #6](README.md#5-sổ-nợ-lab), giai đoạn 22), và quy trình etcd của giai đoạn 19. B6 làm phần **ghi lại thủ công** — không cần feature gate nào |
| Bài 179 — `selectableFields`, subresource `scale` | Tinh chỉnh sau khi CRD đã chạy; `scale` chỉ có nghĩa khi ghép với HPA, mà `DbBackup` là yêu cầu một lần, không có bản sao để co giãn |

Lab này **không thay thế** bài [378](../378-custom-resource-definitions-vi.md) — trang xương sống về
cú pháp CRD, đọc ở [giai đoạn 28](../00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes). Ở đây
bạn chỉ dùng những trường mà bài 179 đã nêu tên trong hai bảng so sánh và ví dụ YAML của nó.

Lab **không phát sinh nợ lab mới** và **không trả nợ lab nào**. Nó phát sinh một nợ kỹ thuật mới —
*operator ngoài cluster* (B4), mã `TD-20` theo [mục 8 của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án) — và **đóng sổ** toàn bộ nợ kỹ thuật của dự án
ở B8.

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B6, B8, B9 và phần chính của B7) khoảng 4 giờ; các mục ⭐ là tùy chọn.
B1 và B7 chỉ đọc. B8 là việc viết tài liệu nghiệm thu — đừng dồn nó vào cuối phiên mệt. Có thể chia
hai phiên: phiên 1 làm B0–B4, phiên 2 làm B5–B9. Giữa hai phiên cứ để nguyên cluster; đầu phiên 2
chạy lại gate ngắn A5.5, `./scripts/deploy.sh` (nó khởi động lại `backup-operator`) và
`./scripts/backup-operator.sh status`.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` bằng kubeconfig quản trị, trừ khi ghi rõ node khác
  hoặc ghi rõ kubeconfig của `backup-operator`. Lệnh đọc file trên worker chạy qua `ssh` từ master.
- **Lab không có fault injection trên node.** Diễn tập ở B5 chỉ dừng tiến trình `backup-operator` của
  dự án và xóa object của dự án. B1 và B7 chỉ **đọc** trên `lab-k8s-worker2`.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/14/`. Bằng chứng: `~/lab-evidence/14/`.
- **Không bao giờ xóa một CRD bạn không tạo.** Cluster ở mốc `04-metrics-ready` đã có CRD của Calico
  và của Traefik; xóa CRD là xóa **mọi** object đang lưu trong nó — với CRD của CNI là xóa cấu hình
  mạng của cluster. Mọi lệnh xóa CRD trong lab đi qua selector `app.kubernetes.io/part-of=bookstore`.
- Lab thêm hai object cấp cluster vào dự án: CRD `dbbackups.bookstore.example.com` và ClusterRole
  `bookstore-dbbackup-viewer`. `teardown.sh` mới gỡ chúng theo label; gate cuối so **số lượng**
  CRD, ClusterRole, ClusterRoleBinding, APIService với tồn kho ghi ở B0.
- `backup-operator` là **tiến trình nền trên master**, không phải Pod: nó chết khi master khởi động
  lại. Sau khi bật lại VM, chạy `./scripts/deploy.sh` hoặc `./scripts/backup-operator.sh start`.
- Token của ServiceAccount nằm trong `secrets/*.kubeconfig`, quyền `600`, không vào git, **không bao
  giờ** chép vào `~/lab-evidence/`. Gate cuối quét bằng chứng tìm chuỗi giống JWT.
- Lab chỉ **đọc** cấu hình kubelet. B0 ghi checksum `/var/lib/kubelet/config.yaml` của ba node, gate
  cuối so lại.
- Kiến thức được dùng: giai đoạn 1–13 — ServiceAccount, Role/ClusterRole, `kubectl auth can-i`,
  `kubectl create token`, `ownerReferences`, Pod Security, NetworkPolicy, ResourceQuota,
  PriorityClass, Job, PVC. Giai đoạn 14 gần cuối Phần I nên **không có món nào phải hoãn vì chưa học**.
- `kubectl scale`, `kubectl patch`, `kubectl delete` trên object của dự án chỉ dùng trong script nghiệm
  thu hoặc diễn tập. Repo là nguồn sự thật duy nhất.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 21: Operator sao lưu và nghiệm thu dự án

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `04-metrics-ready`, repo ở đúng tag của sprint trước, hệ thống
sau sprint 19/20 dựng lại được **chỉ từ repo**, và ghi tồn kho object cấp cluster để gate cuối so lại.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Mốc `04-metrics-ready` và cluster chưa có dấu vết của dự án:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
test "$(sl kubectl top node --no-headers)" -eq 3 && echo 'PASS: kubectl top node du ba node'
test "$(kubectl get apiservice v1beta1.metrics.k8s.io \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')" = 'True' \
  && echo 'PASS: APIService metrics.k8s.io Available'
test "$(kubectl get crd --no-headers | wc -l)" -gt 0 && echo 'PASS: cluster da co CRD cua ha tang nen'
test "$(kubectl get crd,clusterrole,clusterrolebinding,priorityclass,storageclass \
  -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 \
  && echo 'PASS: chua co object cap cluster nao cua du an'
```

Repo, thư mục làm việc và tồn kho "trước". Nếu vừa restore snapshot, repo đã mất — khôi phục từ bundle
theo [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
mkdir -p ~/lab-work/14 ~/lab-evidence/14
EV=~/lab-evidence/14
test -d ~/bookstore/.git || git clone "$(ls -t ~/bookstore-lab-1[23].bundle | head -n 1)" ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qxE 'lab-1[23]' && echo 'PASS: HEAD dung tag lab-13 hoac lab-12'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'

{ for R in crd clusterrole clusterrolebinding apiservice; do
    echo "$R=$(kubectl get "$R" --no-headers | wc -l)"
  done; } | tee "$EV/b0-ton-kho.txt"
{ echo "lab-k8s-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for N in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$N $(ssh "$N" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done; } | tee "$EV/b0-kubelet-sha.txt"
test "$(grep -c ' [0-9a-f]\{64\}$' "$EV/b0-kubelet-sha.txt")" -eq 3 && echo 'PASS: ghi duoc checksum kubelet cua ba node'
```

Dựng lại hệ thống của sprint trước bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > "$EV/b0-deploy.txt" 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > "$EV/b0-smoke-sprint-truoc.txt" 2>&1; echo "smoke exit=$?"
grep -c '^PASS:' "$EV/b0-smoke-sprint-truoc.txt"; grep '^FAIL:' "$EV/b0-smoke-sprint-truoc.txt"
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; bốn dòng `PASS:` của mốc `04-metrics-ready`; ba dòng
`PASS:` về repo và cluster; tồn kho bốn con số được ghi; dòng `PASS:` checksum; `deploy exit=0`,
`smoke exit=0` và file smoke không có dòng `FAIL:`.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint trước **chưa đủ** — có thứ chỉ tồn tại trên
cluster cũ. Sửa repo ở tag đó trước, không sửa tay trên cluster
([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-14-01] Điều tra: bản đồ điểm mở rộng của cluster

**Loại:** Điều tra.

**Bối cảnh:** trước khi thêm API của riêng mình, đội phải biết cluster **đã** được mở rộng ở đâu. Từ
Lab 5b tới Lab 11a bạn đã cài Calico, Traefik, local-path-provisioner và metrics-server mà chưa gọi
tên chúng theo bản đồ điểm mở rộng. Ticket này gọi tên từng thứ, rồi quyết định `DbBackup` — yêu cầu
sao lưu theo nhu cầu, hiện được làm bằng `kubectl create job --from=cronjob/db-backup` — sẽ đứng ở ô
nào trên bản đồ.

**Yêu cầu:**

1. Với mỗi điểm trong bảy điểm của *Chú giải cho hình vẽ* (bài 177), chỉ ra **một hiện vật có thật**
   trên cluster đang chạy Bookstore, hoặc chứng minh điểm đó đang trống.
2. Chứng minh có đúng hai cách thêm API đang chạy song song trên cluster: một API group do
   kube-apiserver tự phục vụ và lưu (từ CRD), một API group được chuyển tiếp sang server khác. Gọi
   thẳng vào đường dẫn của API group thứ hai và cho thấy dữ liệu được tính tại thời điểm hỏi.
3. Phân loại provisioner của StorageClass `bookstore-retain`: storage plugin hay controller? Phân loại
   CNI của cluster: webhook hay binary plugin, ai gọi nó?
4. Ghi ADR quyết định cơ chế cho `DbBackup`.

**Ràng buộc:** chỉ đọc. Không tạo, sửa, xóa object nào; không sửa file nào trên node.

**Đọc lại:**

- Bài [177](../177-extend-kubernetes-vi.md):
  [*Các mẫu thiết kế phần mở rộng*](../177-extend-kubernetes-vi.md#extension-patterns),
  [*Chú giải cho hình vẽ*](../177-extend-kubernetes-vi.md#chú-giải-cho-hình-vẽ-key-to-the-figure),
  [*Thay đổi các resource có sẵn*](../177-extend-kubernetes-vi.md#changing-built-in-resources),
  [*Kết hợp API mới với tự động hóa*](../177-extend-kubernetes-vi.md#combining-new-apis-with-automation).
- Bài [178](../178-api-extension-vi.md) — cả bài, hai gạch đầu dòng.
- Bài [180](../180-apiserver-aggregation-vi.md), mục
  [*Tầng tổng hợp*](../180-apiserver-aggregation-vi.md#tầng-tổng-hợp-aggregation-layer).
- Bài [179](../179-custom-resources-vi.md):
  [*Tôi nên dùng ConfigMap hay custom resource?*](../179-custom-resources-vi.md#tôi-nên-dùng-configmap-hay-custom-resource-should-i-use-a-configmap-or-a-custom-resource),
  [*Chọn phương pháp thêm custom resource*](../179-custom-resources-vi.md#chọn-phương-pháp-thêm-custom-resource-choosing-a-method-for-adding-custom-resources)
  và hai bảng con của nó.
- Bài [182](../182-compute-storage-net-vi.md) — cả bài.

**Bàn giao:** `docs/adr/NNNN-chon-co-che-mo-rong.md` (`NNNN` là số tiếp theo trong `docs/adr/`);
`~/lab-evidence/14/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` cluster đã có ít nhất một API group sinh ra từ CRD trước khi dự án thêm CRD nào.
- `PASS:` `metrics.k8s.io` là API được chuyển tiếp sang một Service trong `kube-system`; gọi thẳng đường
  dẫn của nó trả `NodeMetricsList`.
- `PASS:` không API group nào sinh ra từ CRD trỏ tới một Service.
- `PASS:` provisioner của `bookstore-retain` không có trong danh sách CSIDriver; nó chạy như một
  Deployment thường.
- `PASS:` `lab-k8s-worker2` có file cấu hình CNI và binary CNI trên đĩa.
- `PASS:` mọi Pod của dự án dùng scheduler mặc định; không có kubectl plugin nào.
- `PASS:` có ADR chọn cơ chế mở rộng.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-chon-co-che-mo-rong.md`:

1. `DbBackup` nên là CRD hay aggregated API? Dẫn đúng các dòng trong bảng *So sánh mức độ dễ dùng* và
   bảng *Tính năng nâng cao và tính linh hoạt* của bài 179 quyết định lựa chọn đó. Nếu sau này cần một
   thao tác `restore` ngoài CRUD thì quyết định còn đứng không?
2. Sprint 15 đã có ConfigMap `backup-status` do CronJob `db-backup` ghi kết quả. Vì sao yêu cầu backup
   theo nhu cầu lại là custom resource chứ không phải thêm một key vào ConfigMap đó? Dẫn mục *Tôi nên
   dùng ConfigMap hay custom resource?*
3. Đội muốn "mọi CronJob của dự án phải `concurrencyPolicy: Forbid`". Thêm một CRD có làm được việc đó
   không? Bài 177 nói gì, và điểm mở rộng nào mới làm được?
4. `backup-operator` sẽ nằm ở điểm số mấy? Bài 177 nói controller cũng thêm một điểm lỗi — nếu nó chết
   thì request tới `api` của Bookstore có bị ảnh hưởng không?

<details>
<summary>Gợi ý</summary>

- APIService có `.spec.service` là API được chuyển tiếp; `.spec.service` rỗng là API do chính
  kube-apiserver phục vụ. In ba cột bằng
  `-o custom-columns='NAME:.metadata.name,SVCNS:.spec.service.namespace,SVC:.spec.service.name'`.
- Gọi thẳng một đường dẫn API, không qua `kubectl get <kind>`: `kubectl get --raw /apis/<group>/<version>`.
- Danh sách storage plugin đã đăng ký: `kubectl get csidrivers`.
- CNI trên node: `/etc/cni/net.d/` và `/opt/cni/bin/` — đọc qua `ssh lab-k8s-worker2`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự điều tra xong; chỉ đọc</summary>

```bash
EV=~/lab-evidence/14
cd ~/bookstore

# Điểm 3 — loại resource: API group sinh ra từ CRD, và APIService
kubectl get crd -o jsonpath='{range .items[*]}{.spec.group}{"\n"}{end}' | sort | uniq -c \
  | tee "$EV/b1-crd-groups.txt"
test "$(wc -l < "$EV/b1-crd-groups.txt")" -ge 1 && echo 'PASS: cluster da co API group sinh ra tu CRD'
kubectl get apiservice --no-headers \
  -o custom-columns='NAME:.metadata.name,SVCNS:.spec.service.namespace,SVC:.spec.service.name' \
  > "$EV/b1-apiservice.txt"
awk '$3 != "<none>"' "$EV/b1-apiservice.txt" | tee "$EV/b1-apiservice-proxy.txt"
grep -q '^v1beta1.metrics.k8s.io *kube-system ' "$EV/b1-apiservice-proxy.txt" \
  && echo 'PASS: metrics.k8s.io duoc chuyen tiep sang Service trong kube-system'
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes > "$EV/b1-metrics-nodes.json"
grep -q 'NodeMetricsList' "$EV/b1-metrics-nodes.json" && echo 'PASS: du lieu metric tinh tai thoi diem hoi'
N_CRD_PROXY="$(awk '{print $1}' "$EV/b1-apiservice-proxy.txt" \
  | grep -cFf <(awk '{print $2}' "$EV/b1-crd-groups.txt") || true)"
test "$N_CRD_PROXY" -eq 0 && echo 'PASS: khong API group nao cua CRD can API server rieng'

# Điểm 5 — controller, và điểm 7 — storage plugin
PROV="$(kubectl get storageclass bookstore-retain -o jsonpath='{.provisioner}')"
kubectl get csidrivers > "$EV/b1-csidrivers.txt" 2>&1
kubectl get csidriver "$PROV" >/dev/null 2>&1 \
  && echo "FAIL: $PROV la CSIDriver" || echo "PASS: $PROV khong phai CSIDriver — no la controller"
test "$(kubectl -n local-path-storage get deployment --no-headers | wc -l)" -ge 1 \
  && echo 'PASS: provisioner chay nhu mot Deployment thuong, ngoai control plane'

# Điểm 6 — network plugin: binary plugin do kubelet thực thi
ssh lab-k8s-worker2 'ls -l /etc/cni/net.d/ /opt/cni/bin/' > "$EV/b1-cni-worker2.txt"
{ ssh lab-k8s-worker2 'ls /etc/cni/net.d/' | grep -q 'conflist$' \
  && ssh lab-k8s-worker2 'test -x /opt/cni/bin/calico'; } \
  && echo 'PASS: worker2 co cau hinh CNI va binary calico tren dia'

# Điểm 1, 2, 4 — client, truy cập API, lập lịch
PLUGIN_N="$(kubectl plugin list 2>/dev/null | grep -c '^/')"
VWH_N="$(kubectl get validatingwebhookconfigurations --no-headers 2>/dev/null | wc -l)"
MWH_N="$(kubectl get mutatingwebhookconfigurations --no-headers 2>/dev/null | wc -l)"
SCHED="$(kubectl -n bookstore get pods -o jsonpath='{range .items[*]}{.spec.schedulerName}{"\n"}{end}' | sort -u)"
echo "kubectl plugin=$PLUGIN_N validating=$VWH_N mutating=$MWH_N scheduler=$SCHED" | tee "$EV/b1-diem-1-2-4.txt"
{ test "$PLUGIN_N" -eq 0 && test "$SCHED" = 'default-scheduler'; } \
  && echo 'PASS: khong kubectl plugin; moi Pod du an dung scheduler mac dinh'

ls docs/adr/*-chon-co-che-mo-rong.md >/dev/null 2>&1 && echo 'PASS: co ADR chon co che mo rong'
```

Nếu `VWH_N` hoặc `MWH_N` khác 0, đọc tên chúng: đó là add-on đang dùng điểm số 2. Bằng 0 cũng hợp lệ
— tùy add-on bạn đã cài.

</details>

<details>
<summary>Lời giải tham khảo</summary>

Bản đồ trên cluster Bookstore (điền bằng output thật của bạn):

| Điểm | Hiện vật trên cluster | Mô hình gọi |
| --- | --- | --- |
| 1 — client | trống: không kubectl plugin nào | binary plugin do kubectl thực thi |
| 2 — truy cập API | admission webhook nếu có; Pod Security admission là plugin dựng sẵn, không phải phần mở rộng | webhook — kube-apiserver gọi ra |
| 3 — loại resource | API group của Calico, Traefik (CRD, kube-apiserver tự phục vụ); `metrics.k8s.io` (chuyển tiếp) | CRD: không gọi ra; tầng tổng hợp: request mạng |
| 4 — lập lịch | chỉ scheduler mặc định | — |
| 5 — controller | local-path-provisioner; sắp tới là `backup-operator` | controller là **client** của API server |
| 6 — network plugin | Calico: conflist trong `/etc/cni/net.d/`, binary trong `/opt/cni/bin/` | binary plugin do kubelet thực thi |
| 7 — device/storage plugin | trống: không CSIDriver nào, không device plugin nào (B7) | binary plugin do kubelet gọi |

Ý chính cho ADR:

1. **CRD.** Bảng *So sánh mức độ dễ dùng*: không lập trình, không dịch vụ bổ sung nào phải chạy, vá lỗi
   theo đợt nâng cấp Kubernetes. Bảng *Tính năng nâng cao*: `DbBackup` cần validation, defaulting,
   status subresource, multi-versioning — CRD đều có. Nó không cần custom storage, Protocol Buffers
   hay subresource ngoài CRUD. Một thao tác `restore` dạng subresource thì **không** làm được bằng CRD;
   khi đó hoặc mô hình hóa thành kind thứ hai (`DbRestore`) — vẫn CRD — hoặc chuyển sang aggregated API.
2. ConfigMap hợp khi chương trình trong Pod đọc file cấu hình để tự cấu hình. `DbBackup` là một **yêu
   cầu** có vòng đời riêng: muốn `kubectl get dbbackups` ở mức cao nhất, muốn watch thay đổi để tự động
   hóa, muốn quy ước `.spec`/`.status` — đúng các dấu hiệu "dùng custom resource" của bài 179.
   `backup-status` vẫn giữ vai trò cũ: kết quả của lịch backup định kỳ.
3. **Không.** Custom resource luôn ở API group mới; không thay đổi được hành vi của API có sẵn như
   CronJob. Việc đó cần *phần mở rộng truy cập API* — admission webhook ở điểm số 2.
4. Điểm số 5. Nó là client của API server, **không nằm trên đường request**: `api` vẫn phục vụ bình
   thường khi nó chết; chỉ yêu cầu backup mới không được ai xử lý. B5 chứng minh điều này.

</details>

## B2. [BS-14-02] CRD `DbBackup`

**Loại:** Tính năng.

**Bối cảnh:** muốn có một bản sao lưu ngay — trước khi phát hành, trước khi bảo trì `lab-k8s-worker2`
— người vận hành đang phải nhớ lệnh `kubectl create job --from=cronjob/db-backup`, và kết quả chỉ nằm
trong log của một Pod sớm bị dọn. Dự án cần một **kind** để khai báo "tôi muốn một bản backup" và đọc
lại kết quả ngay trên object đó. Ticket này tạo kind; B4 tạo thứ làm cho nó có hành vi.

**Yêu cầu:**

1. CRD tên `dbbackups.bookstore.example.com` ở `k8s/cluster/crd-dbbackup.yaml`: API group
   `bookstore.example.com`, kind `DbBackup`, số nhiều `dbbackups`, số ít `dbbackup`, tên ngắn `dbb`,
   thuộc category `bookstore`. Mang label `app.kubernetes.io/part-of=bookstore` và
   `app.kubernetes.io/component=operator`. Object thuộc về một namespace.
2. Đúng một version, `v1alpha1` — B6 sẽ thêm `v1`.
3. Hợp đồng `spec`, được **API server** cưỡng chế:
   - `database` bắt buộc, chỉ nhận `bookstore`;
   - `format` là `plain` hoặc `custom`, bỏ trống thì thành `plain`;
   - `note` tùy chọn, tối đa 120 ký tự;
   - trường không khai báo không được lưu;
   - **tên** của `DbBackup` là một DNS label (chữ thường, số, gạch nối; không dấu chấm), tối đa 54 ký
     tự. Mặc định API server nhận mọi DNS subdomain tới 253 ký tự, nhưng B4 đặt tên Job là
     `dbbackup-<tên>` và đưa tên vào một label — cả hai giới hạn 63 ký tự. Tên vượt giới hạn phải bị
     từ chối ngay khi tạo, không được để controller gặp lỗi về sau.
4. `status` gồm `phase` (`Running`, `Succeeded`, `Failed`), `jobName`, `file`, `sizeBytes` (số
   nguyên), `startTime`, `completionTime` (thời điểm), `observedGeneration`, `message`. `status` chỉ
   ghi được qua đường riêng, và ghi `status` không làm tăng `metadata.generation`.
5. `kubectl get dbb` hiện các cột `PHASE`, `SIZE`, `FILE`, `AGE`, mọi cột hiện ở output thường.
6. `scripts/deploy.sh` và `scripts/teardown.sh` thay bằng phiên bản Lab 14 — chép nguyên văn khối bên
   dưới. Đây là **bản cuối** của hai script: chúng đã có sẵn các bước cho ticket B3, B4 và tự bỏ qua
   khi file tương ứng chưa có.
7. Chứng minh trên một namespace dùng một lần: `DbBackup` mất theo namespace, CRD thì không.

**Ràng buộc:** không viết webhook. Không dùng trường nào ngoài những trường bài 179 đã nêu tên trong
hai bảng và ví dụ YAML — riêng từ khóa schema (`required`, `enum`, `default`, `maxLength`, `pattern`)
là phần *validation theo OpenAPI v3.0* mà bảng đó dẫn tới. Không dùng `x-kubernetes-validations`
(CEL): bài 378 dạy nó ở giai đoạn 28. Mọi thử nghiệm làm trong namespace `bookstore-thu`,
**không** trong `bookstore`: chưa có controller nào, và B4 sẽ xử lý mọi `DbBackup` còn sót trong
`bookstore`.

**Đọc lại:**

- Bài [179](../179-custom-resources-vi.md):
  [*Custom controller*](../179-custom-resources-vi.md#custom-controller),
  [*CustomResourceDefinition*](../179-custom-resources-vi.md#customresourcedefinitions),
  [*Tính năng nâng cao và tính linh hoạt*](../179-custom-resources-vi.md#advanced-features-and-flexibility)
  — các dòng Validation, Defaulting, Status Subresource, OpenAPI Schema, Instance Name;
  [*Các tính năng chung*](../179-custom-resources-vi.md#common-features); ví dụ YAML ở
  [*Các trường có thể chọn lọc*](../179-custom-resources-vi.md#crd-selectable-fields) cho khung của
  một CRD có `additionalPrinterColumns`.
- Bài [178](../178-api-extension-vi.md) — với CRD, control plane phục vụ và lưu.
- Bài [378](../378-custom-resource-definitions-vi.md), chỉ hai đoạn mà dòng Validation của bài 179 dẫn
  tới: [*Kiểm tra hợp lệ*](../378-custom-resource-definitions-vi.md#validation) (từ khóa OpenAPI v3.0,
  ví dụ `pattern`) và quy tắc 4 của
  [*structural schema*](../378-custom-resource-definitions-vi.md#specifying-a-structural-schema) — schema
  chỉ được ràng buộc `metadata.name` và `metadata.generateName`.
- Bài [17 — Tên và ID](../17-names-vi.md), mục
  [*Tên DNS Subdomain*](../17-names-vi.md#dns-subdomain-names) và
  [*Tên label theo RFC 1123*](../17-names-vi.md#dns-label-names) — hai giới hạn 253 và 63 ký tự.
- [Mục 5.5 của file dự án](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint) — hàng "14".

**Bàn giao:** `k8s/cluster/crd-dbbackup.yaml`; hai script mới; `docs/adr/NNNN-crd-dbbackup.md`;
`~/lab-evidence/14/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` `teardown.sh` gỡ sạch; `deploy.sh` hai lần liên tiếp đều thành công và chờ CRD `Established`.
- `PASS:` CRD `Established` và `NamesAccepted`, mang label dự án, `Namespaced`, đúng tên, tên ngắn,
  category; chỉ có `v1alpha1`, `storedVersions` là `["v1alpha1"]`.
- `PASS:` `/apis/bookstore.example.com/v1alpha1` do chính kube-apiserver phục vụ, không Service nào.
- `PASS:` ba object sai bị API server từ chối: thiếu `database`, `database` lạ, `note` quá dài.
- `PASS:` tên ở biên: 54 ký tự được nhận; 55 ký tự và tên có dấu chấm bị từ chối.
- `PASS:` trường lạ bị từ chối ở chế độ `strict`; qua `ignore` thì object được tạo nhưng trường lạ
  không được lưu; `format` bỏ trống đọc ra `plain`.
- `PASS:` `kubectl get dbb` có bốn cột yêu cầu; `kubectl get bookstore` liệt kê `DbBackup`.
- `PASS:` ghi `status` qua resource chính bị bỏ qua; qua subresource thì được; `generation` chỉ tăng
  theo `spec`; `phase` sai giá trị bị từ chối ngay cả trên đường `status`.
- `PASS:` xóa namespace thử thì `DbBackup` trong đó mất, CRD còn nguyên.

Hai script, chép nguyên văn. Khung kế thừa `deploy.sh`, `teardown.sh` của
[Lab 11a](LAB-11A-OBSERVABILITY.md) (sprint 18–20 không đổi cơ chế); khối đánh dấu `[14]` là phần mới.
Nếu repo của bạn có công cụ riêng mà bản dưới không gọi — ví dụ `scripts/ci-kubeconfig.sh` của Lab 9a
— chúng vẫn chạy độc lập như cũ.

```bash
cat > ~/bookstore/scripts/deploy.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai dự án Bookstore từ repo — phiên bản Lab 14, bản cuối của dự án.
# Thứ tự: kiểm hạ tầng nền -> bí mật tại chỗ -> object cấp cluster (CRD phải Established) và label node
# -> k8s/system -> k8s/app (db trước, migration sau) -> k8s/tenants/* -> chờ controller
# -> HTTPS qua Traefik -> danh tính có thời hạn của backup-operator -> backup-operator -> DbBackup nền.
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore
SYS=bookstore-system
CRD=dbbackups.bookstore.example.com
HOST=bookstore.lab.local
DATA_NODE=lab-k8s-worker2
DATA_LABEL=bookstore.example.com/data
ERR="$(mktemp)"
APP="$(mktemp)"
trap 'rm -f "$ERR" "$APP"' EXIT
# secrets/ nằm ngoài thư mục gốc của kustomization nên phải nới load restrictor (Lab 3b, ADR 0007).
render() { kubectl kustomize --load-restrictor=LoadRestrictionsNone "$1"; }
# Cảnh báo "would violate PodSecurity" lúc apply là lỗi triển khai (9b): namespace enforce sẽ không tạo
# được Pod của workload đó, nên dừng ngay thay vì chờ rollout hết hạn.
apply_ps() {   # tham số chuyển nguyên cho kubectl apply
  if ! kubectl apply "$@" 2>"$ERR"; then cat "$ERR" >&2; return 1; fi
  cat "$ERR" >&2
  if grep -q 'would violate PodSecurity' "$ERR"; then
    echo 'deploy: co workload vi pham Pod Security — xem canh bao o tren' >&2
    return 1
  fi
}

wait_job() {        # thành công khi Job Complete, thất bại ngay khi Job Failed (Lab 4b)
  local c=''
  for _ in $(seq 1 100); do
    c="$(kubectl -n "$NS" get job "$1" \
          -o jsonpath='{.status.conditions[?(@.status=="True")].type}' 2>/dev/null || true)"
    case " $c " in
      *' Complete '*) return 0 ;;
      *' Failed '*) kubectl -n "$NS" logs "job/$1" --tail=20 || true; return 1 ;;
    esac
    sleep 3
  done
  return 1
}
wait_dbbackup() {   # [14] thành công khi DbBackup Succeeded, thất bại ngay khi Failed
  local p=''
  for _ in $(seq 1 100); do
    p="$(kubectl -n "$NS" get dbbackup "$1" -o jsonpath='{.status.phase}' 2>/dev/null || true)"
    case "$p" in Succeeded) return 0 ;; Failed) return 1 ;; esac
    sleep 3
  done
  return 1
}
mk_kubeconfig() {   # [14] $1 = ServiceAccount trong bookstore, $2 = file đích, $3 = thời hạn token
  local srv tmp tok
  srv="$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
  tmp="$(mktemp -d)"
  kubectl config view --raw --minify -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
    | base64 -d > "$tmp/ca.crt"
  tok="$(kubectl -n "$NS" create token "$1" --duration="$3")"
  rm -f "$2"; ( umask 077; : > "$2" )
  kubectl --kubeconfig="$2" config set-cluster bookstore-lab --server="$srv" \
    --certificate-authority="$tmp/ca.crt" --embed-certs=true >/dev/null
  kubectl --kubeconfig="$2" config set-credentials "$1" --token="$tok" >/dev/null
  kubectl --kubeconfig="$2" config set-context "$1" --cluster=bookstore-lab --user="$1" \
    --namespace="$NS" >/dev/null
  kubectl --kubeconfig="$2" config use-context "$1" >/dev/null
  rm -rf "$tmp"
}

# 0. Hạ tầng nền thuộc cluster, không thuộc repo: thiếu thì dừng, không tự cài.
#    Calico, IngressClass traefik, Service traefik (02-net-ready, 5b); provisioner
#    rancher.io/local-path đang chạy (03-storage-ready, 6a).
for X in 'daemonset/calico-node -n kube-system' 'ingressclass/traefik' 'service/traefik -n traefik'; do
  kubectl get $X >/dev/null 2>&1 || { echo "deploy: thieu $X - cluster chua o moc 02-net-ready"; exit 1; }
done
kubectl get storageclass -o jsonpath='{range .items[*]}{.provisioner}{"\n"}{end}' \
  | grep -qx 'rancher.io/local-path' \
  || { echo 'deploy: cluster chua co provisioner rancher.io/local-path (moc 03-storage-ready)'; exit 1; }
kubectl -n local-path-storage rollout status deployment/local-path-provisioner --timeout=120s

# 1. Bí mật sinh tại chỗ khi thiếu; secrets/ không vào git (3b, 5b). Dữ liệu DB cũ còn trên cluster
#    thì không sinh mới: mật khẩu thật nằm trong chính dữ liệu đó (6a).
mkdir -p secrets
if [ ! -s secrets/db.env ]; then
  CU="$( { kubectl -n "$NS" get pvc data-db-0 -o name 2>/dev/null || true
           kubectl get pv -o jsonpath='{range .items[*]}{.spec.claimRef.namespace}/{.spec.claimRef.name}{"\n"}{end}' \
             | grep -x "$NS/data-db-0" || true; } | wc -l)"
  if [ "$CU" -gt 0 ]; then
    echo 'deploy: thieu secrets/db.env trong khi du lieu db cu con tren cluster.'
    echo 'deploy: chep lai secrets/db.env cu, hoac chay scripts/teardown.sh de xoa du lieu cu.'
    exit 1
  fi
  ( umask 077
    printf 'postgres-password=%s\nauthenticator-password=%s\n' \
      "$(openssl rand -hex 24)" "$(openssl rand -hex 24)" > secrets/db.env )
  echo 'deploy: da sinh secrets/db.env moi'
fi
if [ ! -s secrets/tls.crt ] || [ ! -s secrets/tls.key ]; then
  ( umask 077
    openssl req -x509 -newkey rsa:2048 -nodes -days 365 -subj '/CN=bookstore.lab.local' \
      -addext 'subjectAltName=DNS:bookstore.lab.local' \
      -keyout secrets/tls.key -out secrets/tls.crt 2>/dev/null )
  echo 'deploy: da sinh certificate tu ky moi'
fi

# 2. Object cấp cluster, rồi label node giữ dữ liệu của db: đúng một node mang label (7a).
#    [14] CRD phải Established trước khi bất kỳ custom resource nào được apply ở bước 4.
if compgen -G 'k8s/cluster/*.yaml' >/dev/null; then kubectl apply -f k8s/cluster/; fi
if [ -f k8s/cluster/crd-dbbackup.yaml ]; then
  kubectl wait --for=condition=Established "crd/$CRD" --timeout=60s
fi
kubectl label node "$DATA_NODE" "${DATA_LABEL}=true" --overwrite
for N in $(kubectl get nodes -l "$DATA_LABEL" -o jsonpath='{.items[*].metadata.name}'); do
  [ "$N" = "$DATA_NODE" ] || kubectl label node "$N" "${DATA_LABEL}-"
done

# 3. Namespace hệ thống và node-agent, áp trước ứng dụng; node-agent cũ ở namespace bookstore
#    nhường chỗ (11a).
render k8s/system | kubectl apply -f -
kubectl -n "$NS" delete daemonset,service,serviceaccount node-agent --ignore-not-found --wait=true >/dev/null 2>&1 || true

# 4. Ứng dụng. Render một lần; áp mọi thứ trừ migration, chờ db-0, rồi mới chạy migration.
#    Job db-migrate chạy lại ở MỖI lần triển khai nên phải bất biến kết quả; pod template của Job
#    không sửa được, nên xóa Job cũ trước (4b).
render k8s/app > "$APP"
apply_ps -f "$APP" -l 'app.kubernetes.io/component!=migrate'
kubectl -n "$NS" rollout status statefulset/db --timeout=300s
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s
kubectl -n "$NS" delete job db-migrate --ignore-not-found --wait=true
OUT="$(kubectl apply -f "$APP" -l 'app.kubernetes.io/component=migrate' 2>&1)" \
  || echo "$OUT" | grep -q 'no objects passed to apply' || { echo "$OUT"; exit 1; }
echo "$OUT"
if grep -q 'would violate PodSecurity' <<<"$OUT"; then   # (9b)
  echo 'deploy: db-migrate vi pham Pod Security — xem canh bao o tren' >&2; exit 1
fi
if kubectl -n "$NS" get job db-migrate >/dev/null 2>&1; then
  wait_job db-migrate || { echo 'deploy: db-migrate that bai'; exit 1; }
fi

# 5. Tenant (9b): mỗi thư mục con của k8s/tenants là một tenant, namespace bookstore-<tên>, có ít
#    nhất một Deployment. Apply qua kiểm Pod Security, rồi chờ Available.
for D in k8s/tenants/*/; do
  [ -d "$D" ] || continue
  if [ -f "${D}kustomization.yaml" ]; then render "$D" | apply_ps -f -; else apply_ps -f "$D"; fi
  kubectl -n "bookstore-$(basename "$D")" wait --for=condition=Available deployment --all --timeout=180s
done

# 6. Chờ các controller còn lại.
for D in web api; do
  kubectl -n "$NS" rollout status "deployment/$D" --timeout=300s
done
kubectl -n "$SYS" rollout status daemonset/node-agent --timeout=300s

# 7. Đường vào (5b): Traefik phải phục vụ HTTPS bằng đúng certificate của dự án.
NP="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}')"
IP="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
OK=0
for _ in $(seq 1 30); do
  if curl -fsS --max-time 5 --cacert secrets/tls.crt --resolve "${HOST}:${NP}:${IP}" \
       "https://${HOST}:${NP}/healthz" >/dev/null 2>&1; then OK=1; break; fi
  sleep 2
done
[ "$OK" -eq 1 ] || { echo "deploy: https://${HOST}:${NP}/ chua phuc vu qua Traefik"; exit 1; }

# 8. [14] backup-operator: token có thời hạn vào secrets/, chạy nền trên master (nợ "operator
#    ngoài cluster"), và DbBackup nền trong repo phải thành công.
if kubectl -n "$NS" get serviceaccount backup-operator >/dev/null 2>&1; then
  mk_kubeconfig backup-operator secrets/backup-operator.kubeconfig 24h
fi
if [ -x scripts/backup-operator.sh ] && [ -s secrets/backup-operator.kubeconfig ]; then
  ./scripts/backup-operator.sh restart
  if kubectl -n "$NS" get dbbackup baseline >/dev/null 2>&1; then
    wait_dbbackup baseline || { echo 'deploy: DbBackup baseline khong Succeeded'; exit 1; }
  fi
fi

kubectl -n "$NS" get statefulset,deployment,hpa,cronjob,job,pvc -o wide
kubectl -n "$SYS" get daemonset,service -o wide
if kubectl get crd "$CRD" >/dev/null 2>&1; then kubectl -n "$NS" get dbbackups; fi
EOF

cat > ~/bookstore/scripts/teardown.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ toàn bộ dự án Bookstore khỏi cluster — phiên bản Lab 14, bản cuối của dự án.
# Gỡ mọi object mang label dự án (cả bookstore-system và namespace thử nghiệm), object cấp cluster
# (CRD, ClusterRole, PriorityClass, StorageClass), PV cùng thư mục dữ liệu trên node, label và taint
# node, kubeconfig chứa token và tiến trình backup-operator trên master; rồi tự kiểm. KHÔNG đụng hạ
# tầng nền (Calico, Traefik, local-path-provisioner, metrics-server): chúng không mang label dự án.
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
CRD=dbbackups.bookstore.example.com
LP_ROOT=/opt/local-path-provisioner
WORKERS='lab-k8s-worker1 lab-k8s-worker2'
# Kind namespaced được dọn theo label ở mọi namespace (3b, 4a, 5a, 5b, 6a, 9b, 11a).
KINDS='statefulsets,deployments,daemonsets,cronjobs,jobs,replicasets,pods,poddisruptionbudgets,services,networkpolicies,ingresses,configmaps,secrets,persistentvolumeclaims,roles,rolebindings,resourcequotas,limitranges,serviceaccounts'
# Thư mục volume tên <PV>_<namespace>_<PVC>: khớp namespace bookstore và bookstore-* (6a).
LP_FIND="sudo find $LP_ROOT -mindepth 1 -maxdepth 1 \( -name '*_bookstore_*' -o -name '*_bookstore-*_*' \)"
# Lệnh chạy trên worker. Thư mục gốc chưa từng có nghĩa là không có gì để dọn; lỗi ssh, sudo hay find
# thì trả mã khác 0 — "không kiểm được" không bao giờ được hiểu thành "không còn gì".
LP_ON_NODE="sudo -n test ! -e $LP_ROOT || $LP_FIND"

# PV do provisioner sinh không mang label dự án: nhận diện theo namespace của claim hoặc theo class;
# PV tạo tay thì mang label dự án (6a).
# Hai truy vấn chạy riêng: một cái lỗi thì hàm trả lỗi, không trả về một danh sách thiếu.
project_pvs() {
  local a b
  a="$(kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name} {.spec.claimRef.namespace} {.spec.storageClassName}{"\n"}{end}')" || return 1
  b="$(kubectl get pv -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}')" || return 1
  { printf '%s\n' "$a" | awk '$2 == "bookstore" || $2 ~ /^bookstore-/ || $3 == "bookstore-retain" {print $1}'
    printf '%s\n' "$b"; } | awk 'NF' | sort -u
}
# Liệt kê thư mục dữ liệu còn lại trên một worker; mã thoát khác 0 = không kiểm được worker đó.
lp_left() { ssh -o BatchMode=yes "$1" "$LP_ON_NODE"; }

# 1. [14] Dừng controller trước, để nó không tạo Job mới trong lúc gỡ.
if [ -x scripts/backup-operator.sh ]; then ./scripts/backup-operator.sh stop; fi

# 2. [14] Custom resource trước CRD; Job của chúng đi theo ownerReferences.
if kubectl get crd "$CRD" >/dev/null 2>&1; then
  kubectl delete dbbackups --all --all-namespaces --ignore-not-found --wait=true --timeout=120s
fi

# 3. db tắt êm trước khi xóa namespace (4b).
if kubectl -n "$NS" get statefulset db >/dev/null 2>&1; then
  kubectl -n "$NS" scale statefulset db --replicas=0
  kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s || true
fi

# 4. Namespace của dự án: bookstore, bookstore-staging, bookstore-system, namespace thử có label.
kubectl delete namespace -l "$SEL" --ignore-not-found --wait=true --timeout=300s
kubectl delete "$KINDS" --all-namespaces -l "$SEL" --ignore-not-found --wait=true --timeout=120s

# 5. PV còn lại và thư mục dữ liệu trên node — reclaimPolicy Retain giữ lại cả hai (6a).
PVS="$(project_pvs)" || { echo 'teardown: STOP - khong liet ke duoc PV cua du an'; exit 1; }
for PV in $PVS; do
  P="$(kubectl get pv "$PV" -o jsonpath='{.spec.hostPath.path}{.spec.local.path}' 2>/dev/null || true)"
  N="$(kubectl get pv "$PV" -o jsonpath='{.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]}' 2>/dev/null || true)"
  kubectl delete pv "$PV" --ignore-not-found --wait=true --timeout=120s
  case "$P" in
    "$LP_ROOT"/?*)
      if [ "$N" = "$(hostname)" ]; then sudo rm -rf -- "$P"
      elif [ -n "$N" ]; then ssh "$N" "sudo rm -rf -- '$P'"; fi ;;
  esac
done
#    Thư mục còn sót trên mọi worker, kể cả của PV đã mất từ trước (6a).
for W in $WORKERS; do
  ssh -o BatchMode=yes "$W" "$LP_ON_NODE -exec rm -rf {} +" \
    || { echo "teardown: STOP - khong xoa duoc thu muc du lieu tren $W (ssh, sudo hoac find loi)"; exit 1; }
done

# 6. Object cấp cluster mang label dự án — [14] CRD xóa sau namespace để namespace không kẹt
#    Terminating; StorageClass của dự án xóa cả theo tên (6a) — rồi label và taint node (7a).
kubectl delete crd -l "$SEL" --ignore-not-found --wait=true --timeout=120s
kubectl delete clusterrolebinding,clusterrole,priorityclass,storageclass -l "$SEL" --ignore-not-found
kubectl delete storageclass bookstore-retain --ignore-not-found
kubectl label node --all bookstore.example.com/data- >/dev/null 2>&1 || true
for N in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  for K in $(kubectl get node "$N" -o jsonpath='{range .spec.taints[*]}{.key}{"\n"}{end}' \
             | grep '^bookstore\.example\.com/' | sort -u || true); do
    kubectl taint node "$N" "${K}-"
  done
done

# 7. [14] Token trong kubeconfig đã chết theo ServiceAccount; xóa file cho khỏi nhầm.
rm -f secrets/*.kubeconfig

# 8. Tự kiểm: cluster, thư mục dữ liệu trên worker, tiến trình trên master.
# Mỗi truy vấn kiểm kê chạy riêng và phải thành công: API không trả lời hay Forbidden thì dừng —
# "không hỏi được" không bao giờ được đếm thành "không còn gì". dem <biến> <lệnh...> cộng số dòng
# output của lệnh vào biến; hàm chạy trong shell chính nên exit dừng cả script.
dem() {
  local v="$1" out; shift
  out="$("$@" 2>/dev/null)" || { echo "teardown: STOP - truy van loi, chua ket luan sach: $*"; exit 1; }
  printf -v "$v" '%d' "$(( ${!v:-0} + $(printf '%s\n' "$out" | awk 'NF' | wc -l) ))"
}
taints() {    # key của taint mà dự án đặt lên node (7a); lấy dữ liệu trước, lọc sau
  local t; t="$(kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}')" || return 1
  printf '%s\n' "$t" | awk '/^bookstore\.example\.com\//'
}
LEFT=0; LEFT_NS=0; LEFT_PV=0; LEFT_NODE=0; LEFT_TAINT=0
dem LEFT kubectl get namespace,priorityclass,storageclass,crd,clusterrole,clusterrolebinding -l "$SEL" --no-headers
dem LEFT kubectl get storageclass bookstore-retain --no-headers --ignore-not-found
dem LEFT_NS kubectl get all -A -l "$SEL" --no-headers
dem LEFT_NS kubectl get "$KINDS,endpointslices" -A -l "$SEL" --no-headers
dem LEFT_PV project_pvs
dem LEFT_NODE kubectl get nodes -l bookstore.example.com/data --no-headers
dem LEFT_TAINT taints
#    Mỗi worker cho một trong ba kết quả: còn thư mục, đã xác nhận không còn, hoặc không kiểm được.
#    Kết quả thứ ba dừng script: chưa kết luận sạch thì không được chụp snapshot.
LEFT_DIR=0
for W in $WORKERS; do
  L="$(lp_left "$W")" \
    || { echo "teardown: STOP - khong kiem duoc thu muc du lieu tren $W; chua ket luan sach, KHONG chup snapshot"; exit 1; }
  LEFT_DIR=$(( LEFT_DIR + $(printf '%s\n' "$L" | grep -c . || true) ))
done
LEFT_OP=0
if [ -x scripts/backup-operator.sh ] && ./scripts/backup-operator.sh status >/dev/null 2>&1; then LEFT_OP=1; fi
if [ $(( LEFT + LEFT_NS + LEFT_PV + LEFT_NODE + LEFT_TAINT + LEFT_DIR + LEFT_OP )) -eq 0 ]; then
  echo 'teardown: sach'
else
  echo "teardown: con $LEFT object cap cluster, $LEFT_NS object co label, $LEFT_PV PV, $LEFT_NODE node co label, $LEFT_TAINT taint, $LEFT_DIR thu muc du lieu, operator=$LEFT_OP"
  exit 1
fi
# Hạ tầng nền không mang label dự án và phải còn nguyên (5b).
{ kubectl get ingressclass traefik >/dev/null 2>&1 && kubectl -n kube-system get daemonset calico-node >/dev/null 2>&1; } \
  || echo 'teardown: CANH BAO - ha tang 02-net-ready khong con nguyen (khong do script nay go)'
EOF

chmod +x ~/bookstore/scripts/*.sh
```

Từ ticket này mọi triển khai đi qua `deploy.sh` — cả CRD. Hai giả định kế thừa từ Lab 11a: mọi
manifest của `k8s/app` nằm trong `k8s/app/kustomization.yaml`, và Job `db-migrate` mang label
`app.kubernetes.io/component=migrate`. Repo của bạn khác ở điểm nào thì sửa repo cho khớp
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint), không sửa script.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-crd-dbbackup.md`:

1. `DbBackup` thuộc namespace hay phạm vi cluster? Nghĩ tới tenant `bookstore-staging`, tới RBAC theo
   namespace, và tới việc mỗi `DbBackup` sao lưu DB **của namespace nào**.
2. `database` chỉ có một giá trị hợp lệ — vì sao vẫn khai `enum` thay vì để operator tự kiểm? Câu nào
   của bài 179 về validation trả lời điều này?
3. Vì sao bật subresource `status` ngay ở version đầu, trước khi có controller?
4. `deploy.sh` chờ CRD `Established` trước khi apply `k8s/app`. Nếu `k8s/app` có một `DbBackup` mà
   không chờ thì lần triển khai đầu trên cluster sạch hỏng thế nào? Xóa namespace có dọn CRD không, và
   `teardown.sh` phải làm gì vì điều đó?

<details>
<summary>Gợi ý</summary>

1. Khung CRD: `group`, `scope`, `names`, `versions[]`. Mỗi phần tử `versions` có `served`, `storage`,
   `schema.openAPIV3Schema`, `subresources`, `additionalPrinterColumns`. Ví dụ YAML trong bài 179 là
   khung đủ dùng.
2. Từ khóa schema: `required: [...]` ở object cha, `enum: [...]`, `default: ...`, `maxLength: ...`;
   kiểu thời điểm là `type: string` với `format: date-time`. Ràng buộc tên: khai `metadata` cạnh
   `spec` trong `properties` của gốc schema, bên trong chỉ có `name` với `maxLength` và `pattern`.
   54 = 63 − độ dài tiền tố `dbbackup-`.
3. Thử object sai mà không ghi gì: `kubectl create --dry-run=server -f ...`.
4. Trường lạ: so `kubectl create --validate=strict` với `--validate=ignore`.
5. Ghi `status` qua đường riêng: `kubectl patch ... --subresource=status --type=merge -p '...'`.
6. Nếu `kubectl get dbb` báo không có resource type mà
   `kubectl get --raw /apis/bookstore.example.com/v1alpha1` trả danh sách thì server đã đúng — xem
   [mục 4](#4-troubleshooting-của-lab-này).

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
EV=~/lab-evidence/14; T=bookstore-thu; GRP=bookstore.example.com; CRD=dbbackups.$GRP
cd ~/bookstore

./scripts/teardown.sh > "$EV/b2-teardown.txt" 2>&1
grep -q 'teardown: sach' "$EV/b2-teardown.txt" && echo 'PASS: teardown.sh go sach du an'
./scripts/deploy.sh > "$EV/b2-deploy-1.txt" 2>&1 && echo 'PASS: deploy.sh lan 1 thanh cong'
./scripts/deploy.sh > "$EV/b2-deploy-2.txt" 2>&1 && echo 'PASS: deploy.sh lan 2 thanh cong'
grep -q "$CRD condition met" "$EV/b2-deploy-1.txt" && echo 'PASS: deploy.sh cho CRD Established'

c() { kubectl get crd "$CRD" -o jsonpath="$1"; }
[ "$(c '{.status.conditions[?(@.type=="Established")].status}/{.status.conditions[?(@.type=="NamesAccepted")].status}')" = 'True/True' ] \
  && echo 'PASS: CRD Established va NamesAccepted'
[ "$(c '{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ] && echo 'PASS: CRD mang label du an'
[ "$(c '{.spec.scope}|{.spec.names.kind}|{.spec.names.plural}|{.spec.names.shortNames[*]}|{.spec.names.categories[*]}')" \
  = 'Namespaced|DbBackup|dbbackups|dbb|bookstore' ] && echo 'PASS: scope, ten, ten ngan, category dung'
[ "$(c '{.spec.versions[*].name}|{.status.storedVersions}')" = 'v1alpha1|["v1alpha1"]' ] \
  && echo 'PASS: chi co v1alpha1, storedVersions ["v1alpha1"]'
{ [ -z "$(kubectl get apiservice "v1alpha1.$GRP" -o jsonpath='{.spec.service.name}' 2>/dev/null)" ] \
  && kubectl get --raw "/apis/$GRP/v1alpha1" | grep -q '"name":"dbbackups"'; } \
  && echo 'PASS: kube-apiserver tu phuc vu /apis/bookstore.example.com/v1alpha1'

kubectl create namespace "$T" >/dev/null
dbb() { printf 'apiVersion: %s/v1alpha1\nkind: DbBackup\nmetadata:\n  name: %s\n  namespace: %s\nspec: %s\n' \
          "$GRP" "$1" "$T" "$2"; }
dbb thieu-db '{note: thu}'        | kubectl create --dry-run=server -f - > "$EV/b2-sai-1.txt" 2>&1
dbb sai-db   '{database: khac}'   | kubectl create --dry-run=server -f - > "$EV/b2-sai-2.txt" 2>&1
dbb note-dai "{database: bookstore, note: $(printf 'x%.0s' $(seq 1 130))}" \
                                  | kubectl create --dry-run=server -f - > "$EV/b2-sai-3.txt" 2>&1
grep -q 'spec.database: Required value' "$EV/b2-sai-1.txt" && echo 'PASS: thieu database bi tu choi'
grep -q 'spec.database: Unsupported value' "$EV/b2-sai-2.txt" && echo 'PASS: database la bi tu choi'
grep -q 'spec.note: Too long' "$EV/b2-sai-3.txt" && echo 'PASS: note qua 120 ky tu bi tu choi'
dbb "$(printf 'a%.0s' $(seq 1 54))" '{database: bookstore}' \
  | kubectl create --dry-run=server -f - > "$EV/b2-ten-54.txt" 2>&1 && echo 'PASS: ten 54 ky tu duoc nhan'
dbb "$(printf 'a%.0s' $(seq 1 55))" '{database: bookstore}' \
  | kubectl create --dry-run=server -f - > "$EV/b2-ten-55.txt" 2>&1
grep -q 'metadata.name: Too long' "$EV/b2-ten-55.txt" && echo 'PASS: ten 55 ky tu bi tu choi'
dbb co.dau-cham '{database: bookstore}' | kubectl create --dry-run=server -f - > "$EV/b2-ten-cham.txt" 2>&1
{ grep -q 'metadata.name' "$EV/b2-ten-cham.txt" && grep -q 'should match' "$EV/b2-ten-cham.txt"; } \
  && echo 'PASS: ten co dau cham bi tu choi du la DNS subdomain hop le'

dbb la '{database: bookstore, mauNen: xanh}' | kubectl create --validate=strict -f - > "$EV/b2-strict.txt" 2>&1
grep -qi 'unknown field' "$EV/b2-strict.txt" && echo 'PASS: che do strict tu choi truong la'
dbb la '{database: bookstore, mauNen: xanh}' | kubectl create --validate=ignore -f - >/dev/null
[ -z "$(kubectl -n "$T" get dbbackup la -o jsonpath='{.spec.mauNen}')" ] && echo 'PASS: truong la bi cat tia, khong duoc luu'
[ "$(kubectl -n "$T" get dbbackup la -o jsonpath='{.spec.format}')" = 'plain' ] && echo 'PASS: format mac dinh la plain'

kubectl -n "$T" get dbb > "$EV/b2-get.txt"
head -n 1 "$EV/b2-get.txt" | grep -q '^NAME *PHASE *SIZE *FILE *AGE' && echo 'PASS: kubectl get dbb co du bon cot'
kubectl -n "$T" get bookstore --no-headers 2>/dev/null | grep -q '^la ' && echo 'PASS: category bookstore liet ke DbBackup'

g() { kubectl -n "$T" get dbbackup la -o jsonpath="$1"; }
G0="$(g '{.metadata.generation}')"
kubectl -n "$T" patch dbbackup la --type=merge -p '{"status":{"phase":"Succeeded"}}' >/dev/null
[ -z "$(g '{.status.phase}')" ] && echo 'PASS: ghi status qua resource chinh bi bo qua'
kubectl -n "$T" patch dbbackup la --subresource=status --type=merge -p '{"status":{"phase":"Running"}}' >/dev/null
[ "$(g '{.status.phase}')" = 'Running' ] && echo 'PASS: chi duong /status ghi duoc status'
G1="$(g '{.metadata.generation}')"
kubectl -n "$T" patch dbbackup la --type=merge -p '{"spec":{"note":"doi spec"}}' >/dev/null
G2="$(g '{.metadata.generation}')"
{ [ "$G1" = "$G0" ] && [ "$G2" -gt "$G1" ]; } && echo "PASS: generation $G0 -> $G1 (ghi status) -> $G2 (ghi spec)"
kubectl -n "$T" patch dbbackup la --subresource=status --type=merge -p '{"status":{"phase":"XongRoi"}}' \
  > "$EV/b2-status-sai.txt" 2>&1
grep -q 'Unsupported value' "$EV/b2-status-sai.txt" && echo 'PASS: schema ap ca len duong status'

kubectl delete namespace "$T" --wait=true --timeout=120s >/dev/null
[ "$(kubectl get dbbackups -A --no-headers 2>/dev/null | grep -c "^$T ")" -eq 0 ] \
  && kubectl get crd "$CRD" >/dev/null && echo 'PASS: DbBackup mat theo namespace, CRD van con'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/cluster/crd-dbbackup.yaml — B6 thêm version v1
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: dbbackups.bookstore.example.com      # bắt buộc là <plural>.<group>
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: operator
spec:
  group: bookstore.example.com
  scope: Namespaced
  names:
    plural: dbbackups
    singular: dbbackup
    kind: DbBackup
    shortNames: [dbb]
    categories: [bookstore]
  versions:
  - name: v1alpha1
    served: true
    storage: true
    subresources:
      status: {}
    additionalPrinterColumns:
    - {name: Phase, type: string, jsonPath: .status.phase}
    - {name: Size, type: integer, jsonPath: .status.sizeBytes}
    - {name: File, type: string, jsonPath: .status.file}
    - {name: Age, type: date, jsonPath: .metadata.creationTimestamp}
    schema:
      openAPIV3Schema:
        type: object
        required: [spec]
        properties:
          metadata:                 # schema chỉ được ràng buộc name và generateName (bài 378)
            type: object
            properties:
              name: {type: string, maxLength: 54, pattern: '^[a-z0-9]([-a-z0-9]*[a-z0-9])?$'}
          spec:
            type: object
            required: [database]
            properties:
              database: {type: string, enum: [bookstore]}
              format: {type: string, enum: [plain, custom], default: plain}
              note: {type: string, maxLength: 120}
          status:
            type: object
            properties:
              phase: {type: string, enum: [Running, Succeeded, Failed]}
              jobName: {type: string}
              file: {type: string}
              sizeBytes: {type: integer, format: int64}
              startTime: {type: string, format: date-time}
              completionTime: {type: string, format: date-time}
              observedGeneration: {type: integer, format: int64}
              message: {type: string}
```

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh
kubectl get crd dbbackups.bookstore.example.com
```

Ý chính cho ADR:

1. **Namespace.** Một `DbBackup` sao lưu DB của chính namespace chứa nó; RoleBinding trong `bookstore`
   giới hạn ai được yêu cầu backup DB nào; tenant `bookstore-staging` sau này có `DbBackup` của riêng
   nó mà không đụng tới production. Phạm vi cluster sẽ buộc dùng ClusterRoleBinding — rộng hơn cần
   thiết.
2. Validation ở API server chặn **mọi** client như nhau, kể cả client gọi thẳng REST, và chặn **trước
   khi ghi**. Bài 179: validation "cho phép bạn phát triển API độc lập với các client của mình". Một
   yêu cầu sai bị từ chối ngay trên `kubectl apply`, thay vì nằm trong etcd chờ operator báo `Failed`.
   Tên cũng vậy: một tên DNS subdomain 65 ký tự hợp lệ với API server nhưng không làm tên Job hay
   giá trị label được. Chặn ở CRD thì lỗi hiện ra với người gõ lệnh, không phải trong log controller.
3. Không có subresource thì ai sửa được object là sửa được `status` — người dùng có thể tự khai
   `Succeeded` cho một bản backup chưa từng chạy. Có subresource thì RBAC tách được quyền ghi `spec`
   (người dùng) và quyền ghi `status` (controller), và `generation` thành tín hiệu "có việc mới".
4. Chưa `Established` thì endpoint chưa tồn tại: `kubectl apply` báo không tìm được mapping cho kind
   `DbBackup` và trả mã lỗi — `set -e` dừng `deploy.sh` giữa chừng. Xóa namespace **không** dọn CRD
   (object cấp cluster); `teardown.sh` phải xóa CRD theo label, và xóa **sau** namespace.

</details>

## B3. [BS-14-03] Danh tính, quyền và hạn mức cho kind mới

**Loại:** Tính năng.

**Bối cảnh:** một kind mới ra đời sau mọi role dựng sẵn, nên không role nào biết tới nó. Sprint này cần
ba thứ trước khi có controller: một **danh tính** cho controller với quyền vừa đủ, quyền **đọc**
`DbBackup` cho đội phát triển (sprint 15 chỉ cho họ `view`), và một **trần số lượng** — custom resource
nằm trong kho lưu trữ của API server như ConfigMap, tạo vô hạn là làm hại control plane.

**Yêu cầu:**

1. ServiceAccount `backup-operator` trong `bookstore`, label theo hợp đồng
   (`app.kubernetes.io/component=operator`). Nó có **đúng** những quyền mà vòng lặp ở B4 cần trong
   namespace `bookstore`: đọc, liệt kê, theo dõi `DbBackup`; ghi `status` của `DbBackup`; tạo và đọc
   Job; đọc Pod và log của Pod. **Không** sửa `spec` hay xóa `DbBackup`, không xóa Job, không tạo Pod
   trực tiếp, không đọc Secret, không có quyền gì ở namespace khác.
2. Không Pod nào chạy dưới ServiceAccount này. Danh tính được dùng qua
   `secrets/backup-operator.kubeconfig` — token có thời hạn mà `deploy.sh` (đã chép ở B2) sinh ra;
   file quyền `600`, git bỏ qua.
3. Nhóm `bookstore-devs` đọc được `DbBackup` trong `bookstore` nhưng không tạo, sửa, xóa. Quyền đọc
   này được định nghĩa **một lần** ở cấp cluster — ClusterRole `bookstore-dbbackup-viewer` ở
   `k8s/cluster/`, mang label dự án — và chỉ được **gán** trong namespace `bookstore`.
4. `bookstore-quota` đặt trần 20 `DbBackup` trong `bookstore`. Sprint 11 đặt `count/jobs.batch: "12"`
   và trạng thái thường đã dùng khoảng 9–10 Job; mỗi `DbBackup` sẽ sinh một Job sống tới khi TTL dọn
   (B4). Nâng **đúng** key `count/jobs.batch` vừa đủ để có ít nhất ba chỗ trống cho Job của operator,
   giữ nguyên mọi key khác, ghi lý do vào ADR.
5. Manifest trong namespace đặt ở `k8s/app/rbac/` và được kustomization của `k8s/app` gom vào.
6. Chứng minh trên namespace thử `bookstore-thu`: ClusterRole `view` dựng sẵn đọc được Pod nhưng không
   đọc được `DbBackup`; quota đếm `DbBackup` và từ chối object vượt trần.

**Ràng buộc:** không dùng wildcard trong rule. Không dùng ClusterRoleBinding. Không cấp quyền cho
`bookstore-devs` bằng cách sửa ClusterRole `view` dựng sẵn.

**Đọc lại:**

- Bài [179](../179-custom-resources-vi.md), mục
  [*Xác thực, phân quyền và kiểm toán*](../179-custom-resources-vi.md#xác-thực-phân-quyền-và-kiểm-toán-authentication-authorization-and-auditing)
  và [*Lưu trữ*](../179-custom-resources-vi.md#storage).
- Bài [120](../120-rbac-good-practices-vi.md):
  [*Đặc quyền tối thiểu*](../120-rbac-good-practices-vi.md#least-privilege),
  [*Tạo workload*](../120-rbac-good-practices-vi.md#workload-creation),
  [*Từ chối dịch vụ qua việc tạo đối tượng*](../120-rbac-good-practices-vi.md#object-creation-dos).
- Bài [134](../134-resource-quotas-vi.md), mục
  [*Hạn ngạch theo số lượng đối tượng*](../134-resource-quotas-vi.md#quota-on-object-count) — đoạn về
  custom resource và về API aggregation.
- Bài [279](../279-configure-service-account-vi.md) — token có thời hạn qua `--duration`.

**Bàn giao:** `k8s/cluster/clusterrole-dbbackup-viewer.yaml`; `k8s/app/rbac/backup-operator.yaml`;
kustomization và manifest quota đã cập nhật; `docs/adr/NNNN-rbac-dbbackup.md`;
`~/lab-evidence/14/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` `view` dựng sẵn không nhắc tới `bookstore.example.com`; gán `view` thì đọc được Pod nhưng không
  đọc được `DbBackup`.
- `PASS:` `bookstore-devs` liệt kê được `DbBackup` trong `bookstore`; không tạo, không xóa; vẫn không
  đọc Secret.
- `PASS:` `backup-operator` làm được đúng bảy việc liệt kê ở yêu cầu 1 và bị từ chối ở tám việc còn lại
  được kiểm.
- `PASS:` kubeconfig của operator quyền `600`, git bỏ qua, xác thực đúng danh tính ServiceAccount, bị
  từ chối khi đọc Secret.
- `PASS:` ClusterRole mang label dự án; ServiceAccount mang `component=operator`.
- `PASS:` `bookstore-quota` có trần `DbBackup`; trên namespace thử, object vượt trần bị từ chối với
  `exceeded quota`.
- `PASS:` `count/jobs.batch` còn ít nhất ba chỗ trống sau khi triển khai.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-rbac-dbbackup.md`:

1. Vì sao `view` không đọc được `DbBackup`, và câu nào của bài 179 nói về việc đóng gói CRD kèm role?
   Vì sao định nghĩa quyền đọc bằng ClusterRole mà gán bằng RoleBinding, chứ không viết một Role?
2. RBAC coi `dbbackups` và `dbbackups/status` là hai resource. Điều đó cho dự án ranh giới gì giữa
   người dùng và controller?
3. Operator không đọc được Secret nhưng tạo được Job. Theo mục *Tạo workload* của bài 120, thực tế nó
   chạm tới được những gì trong `bookstore`? Lớp nào của dự án giới hạn thiệt hại nếu kubeconfig của
   nó bị lộ?
4. Token có thời hạn: hết hạn thì operator ra sao, ai cấp lại, và vì sao vẫn chọn token có hạn thay
   vì token dài hạn dạng Secret?
5. Vì sao quota đếm được `DbBackup` mà không đếm được object của một aggregated API? Trần 20 bảo vệ cái
   gì?

<details>
<summary>Gợi ý</summary>

- Một rule RBAC có `apiGroups`, `resources`, `verbs`. Subresource viết dạng `dbbackups/status`,
  `pods/log`.
- Group trong `subjects` cần `apiGroup: rbac.authorization.k8s.io`.
- Kiểm quyền: `kubectl auth can-i <verb> <resource> -n <ns> --as=system:serviceaccount:<ns>:<sa>`;
  với group: `--as=<user bất kỳ> --as-group=bookstore-devs`; subresource: `--subresource=status`.
- Quota cho custom resource: khóa `count/<plural>.<group>` trong `spec.hard`.
- Quota vừa tạo cần controller tính `status` trước; tạo object ngay lúc đó có thể bị từ chối vì chưa
  biết mức dùng — chờ `status.hard` có giá trị.

</details>

<details>
<summary>Script nghiệm thu — chạy sau `./scripts/deploy.sh`</summary>

```bash
NS=bookstore; T=bookstore-thu; EV=~/lab-evidence/14; GRP=bookstore.example.com
OP="system:serviceaccount:$NS:backup-operator"
cd ~/bookstore
./scripts/deploy.sh > "$EV/b3-deploy.txt" 2>&1 && echo 'PASS: deploy.sh thanh cong'
want() { local w="$1" m="$2"; shift 2
         [ "$(kubectl auth can-i "$@" 2>/dev/null)" = "$w" ] && echo "PASS: $m" || echo "FAIL: $m"; }

kubectl get clusterrole view -o yaml > "$EV/b3-clusterrole-view.yaml"
grep -q "$GRP" "$EV/b3-clusterrole-view.yaml" \
  && echo 'FAIL: view nhac toi API group cua du an' || echo 'PASS: ClusterRole view khong biet bookstore.example.com'
kubectl create namespace "$T" >/dev/null
kubectl -n "$T" create rolebinding chi-view --clusterrole=view --user=nguoi-thu >/dev/null
want yes 'view doc duoc Pod'                     list pods -n "$T" --as=nguoi-thu
want no  'view KHONG doc duoc DbBackup'          list "dbbackups.$GRP" -n "$T" --as=nguoi-thu

want yes 'bookstore-devs liet ke duoc DbBackup'  list "dbbackups.$GRP" -n "$NS" --as=dev-thu --as-group=bookstore-devs
want no  'bookstore-devs khong tao DbBackup'     create "dbbackups.$GRP" -n "$NS" --as=dev-thu --as-group=bookstore-devs
want no  'bookstore-devs khong xoa DbBackup'     delete "dbbackups.$GRP" -n "$NS" --as=dev-thu --as-group=bookstore-devs
want no  'bookstore-devs van khong doc Secret'   get secrets -n "$NS" --as=dev-thu --as-group=bookstore-devs

want yes 'operator get DbBackup'                 get "dbbackups.$GRP" -n "$NS" --as="$OP"
want yes 'operator list DbBackup'                list "dbbackups.$GRP" -n "$NS" --as="$OP"
want yes 'operator watch DbBackup'               watch "dbbackups.$GRP" -n "$NS" --as="$OP"
want yes 'operator ghi dbbackups/status'         patch "dbbackups.$GRP" --subresource=status -n "$NS" --as="$OP"
want yes 'operator tao Job'                      create jobs.batch -n "$NS" --as="$OP"
want yes 'operator doc Pod'                      list pods -n "$NS" --as="$OP"
want yes 'operator doc log Pod'                  get pods --subresource=log -n "$NS" --as="$OP"
want no  'operator khong sua spec DbBackup'      patch "dbbackups.$GRP" -n "$NS" --as="$OP"
want no  'operator khong xoa DbBackup'           delete "dbbackups.$GRP" -n "$NS" --as="$OP"
want no  'operator khong xoa Job'                delete jobs.batch -n "$NS" --as="$OP"
want no  'operator khong tao Pod truc tiep'      create pods -n "$NS" --as="$OP"
want no  'operator khong doc Secret'             get secrets -n "$NS" --as="$OP"
want no  'operator khong exec vao Pod'           create pods --subresource=exec -n "$NS" --as="$OP"
want no  'operator khong co quyen o staging'     list "dbbackups.$GRP" -n bookstore-staging --as="$OP"
want no  'operator khong co quyen cap cluster'   list "dbbackups.$GRP" -A --as="$OP"

KC=secrets/backup-operator.kubeconfig
[ "$(stat -c '%a' "$KC" 2>/dev/null)" = '600' ] && echo 'PASS: kubeconfig cua operator quyen 600'
git check-ignore -q "$KC" && echo 'PASS: git bo qua kubeconfig cua operator'
[ "$(kubectl --kubeconfig="$KC" auth whoami -o jsonpath='{.status.userInfo.username}' 2>/dev/null)" = "$OP" ] \
  && echo 'PASS: kubeconfig xac thuc dung danh tinh ServiceAccount'
kubectl --kubeconfig="$KC" -n "$NS" get secrets >/dev/null 2>&1 \
  && echo 'FAIL: kubeconfig cua operator doc duoc Secret' || echo 'PASS: kubeconfig cua operator bi tu choi doc Secret'
[ "$(kubectl get clusterrole bookstore-dbbackup-viewer -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ] \
  && [ "$(kubectl -n "$NS" get sa backup-operator -o jsonpath='{.metadata.labels.app\.kubernetes\.io/component}')" = 'operator' ] \
  && echo 'PASS: ClusterRole va ServiceAccount mang label theo hop dong'

Q="$(kubectl -n "$NS" get resourcequota bookstore-quota -o jsonpath='{.spec.hard.count/dbbackups\.bookstore\.example\.com}')"
[ "${Q:-0}" -ge 1 ] && echo "PASS: bookstore-quota dat tran $Q DbBackup"
JH="$(kubectl -n "$NS" get resourcequota bookstore-quota -o jsonpath='{.status.hard.count/jobs\.batch}')"
JU="$(kubectl -n "$NS" get resourcequota bookstore-quota -o jsonpath='{.status.used.count/jobs\.batch}')"
[ $(( ${JH:-0} - ${JU:-0} )) -ge 3 ] && echo "PASS: count/jobs.batch $JU/$JH, con du cho cho Job cua operator"
kubectl -n "$T" create quota tran-dbb --hard="count/dbbackups.$GRP=1" >/dev/null
for _ in $(seq 1 30); do [ -n "$(kubectl -n "$T" get quota tran-dbb -o jsonpath='{.status.hard}')" ] && break; sleep 2; done
mk() { printf 'apiVersion: %s/v1alpha1\nkind: DbBackup\nmetadata:\n  name: %s\n  namespace: %s\nspec: {database: bookstore}\n' \
         "$GRP" "$1" "$T"; }
mk mot | kubectl create -f - >/dev/null 2>&1 && echo 'PASS: DbBackup thu nhat nam trong tran'
mk hai | kubectl create -f - > "$EV/b3-quota.txt" 2>&1
grep -q 'exceeded quota' "$EV/b3-quota.txt" && echo 'PASS: DbBackup thu hai bi quota tu choi'
kubectl delete namespace "$T" --wait=true --timeout=120s >/dev/null && echo 'PASS: da xoa namespace thu'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/cluster/clusterrole-dbbackup-viewer.yaml — "role đóng gói kèm CRD" của dự án
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: bookstore-dbbackup-viewer
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: operator
rules:
- apiGroups: [bookstore.example.com]
  resources: [dbbackups]
  verbs: [get, list, watch]
```

```yaml
# k8s/app/rbac/backup-operator.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: backup-operator
  namespace: bookstore
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: operator}
automountServiceAccountToken: false      # không Pod nào chạy dưới danh tính này
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: backup-operator
  namespace: bookstore
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: operator}
rules:
- {apiGroups: [bookstore.example.com], resources: [dbbackups], verbs: [get, list, watch]}
- {apiGroups: [bookstore.example.com], resources: [dbbackups/status], verbs: [get, patch, update]}
- {apiGroups: [batch], resources: [jobs], verbs: [get, list, watch, create]}
- {apiGroups: [""], resources: [pods], verbs: [get, list]}
- {apiGroups: [""], resources: [pods/log], verbs: [get]}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: backup-operator
  namespace: bookstore
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: operator}
roleRef: {apiGroup: rbac.authorization.k8s.io, kind: Role, name: backup-operator}
subjects:
- {kind: ServiceAccount, name: backup-operator, namespace: bookstore}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bookstore-devs-dbbackup-viewer
  namespace: bookstore
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: operator}
roleRef: {apiGroup: rbac.authorization.k8s.io, kind: ClusterRole, name: bookstore-dbbackup-viewer}
subjects:
- {kind: Group, name: bookstore-devs, apiGroup: rbac.authorization.k8s.io}
```

Thêm `rbac/backup-operator.yaml` vào danh sách `resources` của kustomization đang gom `k8s/app/rbac/`
(của `k8s/app` hoặc của chính thư mục `rbac/` nếu sprint 15 tách riêng). Trong `spec.hard` của
`k8s/app/policy/resourcequota.yaml` thêm một dòng và sửa đúng một dòng, giữ nguyên các key khác:

```yaml
    count/jobs.batch: "16"     # Lab 14: +4 cho Job của backup-operator, mỗi Job sống tới TTL 600 giây
    count/dbbackups.bookstore.example.com: "20"
```

```bash
cd ~/bookstore && ./scripts/deploy.sh
ls -l secrets/backup-operator.kubeconfig
```

Ý chính cho ADR:

1. Role dựng sẵn liệt kê **tường minh** từng API group; `bookstore.example.com` ra đời sau nên không
   role nào biết. Bài 179: CRD và aggregated API "thường được đóng gói kèm các định nghĩa role mới cho
   những kiểu dữ liệu mà chúng bổ sung". ClusterRole là định nghĩa dùng lại được — sau này gán thêm cho
   `bookstore-staging` bằng một RoleBinding nữa; RoleBinding giữ phạm vi trong một namespace, đúng
   *Đặc quyền tối thiểu* của bài 120 ("RoleBinding thay vì ClusterRoleBinding").
2. Quyền trên `dbbackups` không kéo theo quyền trên `dbbackups/status`. Operator chỉ ghi được `status`,
   người dùng chỉ ghi được `spec`: operator không tự viết lại yêu cầu rồi báo "xong", người dùng không
   tự khai `Succeeded`.
3. Quyền tạo workload ngầm cấp quyền tới Secret, ConfigMap, PVC mount được và tới mọi ServiceAccount
   trong namespace — tức operator **gián tiếp** đọc được `db-credentials` qua một Job tự viết. Thiệt hại
   bị giới hạn bởi: Pod Security `restricted` (không Pod đặc quyền, không `hostPath`), quota (không tạo
   vô hạn), NetworkPolicy (Pod mới vẫn chỉ đi được theo ma trận), token có hạn, và RBAC chỉ trong
   `bookstore`. Ghi điều này vào `R13` khi nghiệm thu: đạt có điều kiện.
4. Hết hạn thì mọi lệnh `kubectl` của operator trả `Unauthorized`, vòng lặp ghi lỗi và chờ — không làm
   hỏng gì, chỉ ngừng xử lý yêu cầu mới. `deploy.sh` cấp token mới; operator đọc lại file ở lần gọi
   sau. Token có hạn giới hạn thời gian một token bị lộ còn dùng được; token dạng Secret sống mãi và
   nằm trong etcd. Gánh nặng cấp lại là một phần của nợ *operator ngoài cluster* (B4).
5. Quota được control plane cưỡng chế cho API do chính API server phục vụ, **kể cả custom resource từ
   CRD**; aggregated API do server khác phục vụ nên phải tự cưỡng chế (bài 134). Trần 20 bảo vệ kho
   lưu trữ của API server (bài 179, *Lưu trữ*) khỏi một script lỗi tạo `DbBackup` liên tục — đúng rủi ro
   *Từ chối dịch vụ qua việc tạo đối tượng* của bài 120. Trần `count/jobs.batch` thì phải nâng: mỗi
   `DbBackup` sinh một Job, và ở mức dùng thường 9–10/12, chỉ hai yêu cầu backup liền nhau đã đủ làm
   Job của CronJob `db-backup` bị quota từ chối. Nâng đúng key đó, không nới các trần khác.

</details>

## B4. [BS-14-04] `backup-operator`: vòng lặp điều khiển cho `DbBackup`

**Loại:** Tính năng.

**Bối cảnh:** CRD và quyền đã có, nhưng một `DbBackup` vẫn chỉ là dữ liệu nằm trong kho. Ticket này
viết thứ biến dữ liệu đó thành hành vi: một controller đọc `spec`, tạo Job `pg_dump`, theo dõi Job và
ghi kết quả vào `status`. Dự án không có toolchain hay quy trình build image, nên controller là một
vòng lặp shell chạy trên master dưới ServiceAccount `backup-operator` — cùng cơ chế, khác nơi chạy.

**Yêu cầu:**

1. `scripts/backup-operator.sh` theo khung dưới đây. **Phần công cụ** (quản lý tiến trình, `reconcile`)
   chép nguyên văn; bạn viết hai hàm `job_manifest` và `reconcile_one`.
2. Hợp đồng của vòng lặp, cho mỗi `DbBackup` trong `bookstore`:
   - `phase` đã là `Succeeded` hoặc `Failed`: **không làm gì**, không ghi gì.
   - Chưa có Job `dbbackup-<tên>`: tạo Job, ghi `phase: Running`, `jobName`, `startTime`,
     `observedGeneration`.
   - Job `Complete`: đọc dòng `db-backup: OK <file> <bytes> byte` trong log của Pod thành công, ghi
     `phase: Succeeded`, `file`, `sizeBytes`, `completionTime`.
   - Job `Failed`: ghi `phase: Failed`, `completionTime`, `message`.
   - `phase` là `Running` mà Job không còn: ghi `phase: Failed` với `message` nói rõ — **không** tạo lại.
   - Không tạo được Job: lỗi **không tự khỏi** (API server báo object Job không hợp lệ) thì ghi
     `phase: Failed` với `message`, không thử lại; lỗi **tạm thời** (quota đang đầy, API server không trả
     lời) thì không ghi gì, thử lại ở vòng sau.
   - `DbBackup` là yêu cầu **một lần**: đã `Succeeded` hay `Failed` thì sửa `spec` cũng không chạy lại.
     Muốn bản backup mới thì tạo `DbBackup` mới.
3. Job do operator tạo:
   - tên `dbbackup-<tên DbBackup>`; label `app.kubernetes.io/part-of=bookstore`,
     `app.kubernetes.io/component=backup`, `app.kubernetes.io/name=postgresql`,
     `app.kubernetes.io/managed-by=backup-operator`, `bookstore.example.com/dbbackup=<tên>`;
     `ownerReferences` trỏ về đúng `DbBackup` đó với vai trò controller;
   - `pg_dump` database `bookstore` qua `db-0.db`, mật khẩu từ Secret `db-credentials`, định dạng theo
     `spec.format`; ghi một file mới vào PVC `backup-store` ở `/backup` theo cùng hợp đồng ghi file của
     Lab 6a — ghi tạm, kiểm file có bảng `api.books`, rồi mới đổi sang tên chính thức; giữ tối đa 10
     bản `dbbackup-*`; dọn file tạm bỏ dở — và in **đúng hợp đồng** của `db-backup`
     ([mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe)): một dòng
     `db-backup: OK <file> <bytes> byte`;
   - đạt Pod Security `restricted`; tới được `db` qua NetworkPolicy đang có; có `requests`/`limits`
     nằm trong quota; PriorityClass `bookstore-batch`; không mount token; tự được dọn sau một thời gian.
4. `k8s/app/backup/dbbackup-baseline.yaml`: `DbBackup` tên `baseline` — bản sao lưu nền sau mỗi lần dựng
   hệ thống từ repo — được kustomization gom vào. `deploy.sh` của B2 tự khởi động operator và chờ
   `baseline` thành công.
5. `docs/runbook-backup-operator.md`: chạy, dừng, xem log, cấp lại token, yêu cầu một bản backup, đọc
   kết quả, chép file dump ra máy host.
6. Ghi nợ mới *operator ngoài cluster* vào `docs/tech-debt.md` với mã `TD-20` — mã cố định theo
   sổ nợ của file dự án.

**Ràng buộc:** chỉ `bash`, `kubectl`, coreutils — không `jq`, không ngôn ngữ khác. Operator **không**
dùng kubeconfig quản trị. Không nới Pod Security, NetworkPolicy hay quota để Job chạy được — Job bị chặn
thì sửa Job. Nếu NetworkPolicy của bạn không chọn được Pod của Job bằng label hợp đồng thì bổ sung
đúng ô ma trận "Job của `backup-operator` → `db`" ở [mục 3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật),
không mở rộng hơn.

Khung `scripts/backup-operator.sh`, chép nguyên văn rồi viết hai hàm trong khối đánh dấu:

```bash
cat > ~/bookstore/scripts/backup-operator.sh <<'EOF'
#!/usr/bin/env bash
# backup-operator — controller của custom resource DbBackup (dự án Bookstore, Lab 14).
# Chạy trên lab-k8s-master dưới ServiceAccount bookstore/backup-operator, ngoài cluster (nợ kỹ thuật).
# Cách dùng: backup-operator.sh run | once | start | stop | restart | status
set -uo pipefail
SELF="$(cd "$(dirname "$0")" && pwd)/$(basename "$0")"
cd "$(dirname "$SELF")/.."
NS=bookstore
RES=dbbackups.bookstore.example.com
KCFG="${KCFG:-secrets/backup-operator.kubeconfig}"
PIDFILE=secrets/backup-operator.pid
LOGFILE=secrets/backup-operator.log
INTERVAL="${INTERVAL:-5}"

k()   { kubectl --kubeconfig="$KCFG" -n "$NS" "$@"; }
now() { date -u +%Y-%m-%dT%H:%M:%SZ; }
log() { echo "$(now) $*"; }
set_status() {      # $1 = tên DbBackup, $2 = object JSON gộp vào status
  k patch "$RES" "$1" --subresource=status --type=merge -p "{\"status\":$2}" >/dev/null
}

# ==== PHẦN BẠN VIẾT (BS-14-04) ==================================================================
job_manifest() {    # $1 = tên DbBackup, $2 = uid, $3 = apiVersion, $4 = plain|custom → in YAML của Job
  echo 'job_manifest: chua viet' >&2; return 1
}
reconcile_one() {   # $1 = tên một DbBackup → đưa thực tế về đúng spec, ghi status; lỗi thì return 1
  echo 'reconcile_one: chua viet' >&2; return 1
}
# ================================================================================================

reconcile() {       # một vòng: mọi DbBackup trong namespace
  local rc=0 n names
  names="$(k get "$RES" -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}')" || return 1
  for n in $names; do
    reconcile_one "$n" || { log "$n: loi, thu lai o vong sau"; rc=1; }
  done
  return "$rc"
}

is_running() {
  local p
  p="$(cat "$PIDFILE" 2>/dev/null)" || return 1
  [ -n "$p" ] && kill -0 "$p" 2>/dev/null && grep -qa 'backup-operator' "/proc/$p/cmdline" 2>/dev/null
}
do_start() {
  if is_running; then echo "backup-operator: dang chay (pid $(cat "$PIDFILE"))"; return 0; fi
  [ -s "$KCFG" ] || { echo "backup-operator: thieu $KCFG — chay scripts/deploy.sh"; return 1; }
  setsid nohup "$SELF" run >> "$LOGFILE" 2>&1 < /dev/null &
  for _ in $(seq 1 10); do is_running && break; sleep 1; done
  if is_running; then echo "backup-operator: da chay nen (pid $(cat "$PIDFILE"), log $LOGFILE)"
  else echo "backup-operator: khong khoi dong duoc, xem $LOGFILE"; return 1; fi
}
do_stop() {
  if is_running; then
    kill -TERM "$(cat "$PIDFILE")"
    for _ in $(seq 1 30); do is_running || break; sleep 1; done
    if is_running; then kill -KILL "$(cat "$PIDFILE")"; fi
  fi
  rm -f "$PIDFILE"
  echo 'backup-operator: da dung'
}

case "${1:-run}" in
  run)
    if is_running && [ "$(cat "$PIDFILE")" != "$$" ]; then echo 'backup-operator: da co ban dang chay'; exit 1; fi
    echo $$ > "$PIDFILE"
    trap 'log "nhan tin hieu dung, thoat"; rm -f "$PIDFILE"; exit 0' TERM INT
    log "bat dau: namespace $NS, chu ky ${INTERVAL}s, kubeconfig $KCFG"
    while true; do
      reconcile || true
      sleep "$INTERVAL" & wait $!
    done ;;
  once)    reconcile ;;
  start)   do_start ;;
  stop)    do_stop ;;
  restart) do_stop; do_start ;;
  status)  if is_running; then echo "backup-operator: dang chay (pid $(cat "$PIDFILE"))"
           else echo 'backup-operator: khong chay'; exit 3; fi ;;
  *)       echo "cach dung: $0 run|once|start|stop|restart|status"; exit 2 ;;
esac
EOF
chmod +x ~/bookstore/scripts/backup-operator.sh
```

**Đọc lại:**

- Bài [181](../181-operator-vi.md):
  [*Operator trong Kubernetes*](../181-operator-vi.md#operator-trong-kubernetes-operators-in-kubernetes),
  [*Một ví dụ về operator*](../181-operator-vi.md#example),
  [*Triển khai operator*](../181-operator-vi.md#triển-khai-operator-deploying-operators),
  [*Sử dụng một operator*](../181-operator-vi.md#using-operators).
- Bài [177](../177-extend-kubernetes-vi.md), mục
  [*Các mẫu thiết kế phần mở rộng*](../177-extend-kubernetes-vi.md#extension-patterns) — controller đọc
  `.spec`, làm việc, cập nhật `.status`.
- Bài [179](../179-custom-resources-vi.md), mục
  [*Custom controller*](../179-custom-resources-vi.md#custom-controller).
- Bài [25 — Controller](../25-controllers-vi.md) — vòng lặp điều khiển.
- [Mục 3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật) và
  [mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch) của file dự án — ma trận
  NetworkPolicy, PriorityClass cho Job.

**Bàn giao:** `scripts/backup-operator.sh`; `k8s/app/backup/dbbackup-baseline.yaml`; kustomization đã
cập nhật; `docs/runbook-backup-operator.md`; `docs/adr/NNNN-backup-operator.md`; `docs/tech-debt.md`;
`~/lab-evidence/14/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` `teardown.sh` gỡ sạch (kể cả dừng operator); `deploy.sh` dựng lại, khởi động operator và chờ
  được `baseline` thành công; chạy lại `deploy.sh` lần hai vẫn thành công.
- `PASS:` `baseline` ở `Succeeded`, đủ sáu trường `status`, `observedGeneration` bằng `generation`,
  file nằm dưới `/backup`, kích thước lớn hơn 0.
- `PASS:` một Pod đọc PVC `backup-store` thấy file đúng kích thước mà `status` ghi.
- `PASS:` `DbBackup` với `format: custom` thành công, file đuôi `.dump`.
- `PASS:` Job đúng tên, label, `ownerReferences` (đúng `uid`, `controller: true`), PriorityClass
  `bookstore-batch`, không mount token, có TTL, có `requests`/`limits`.
- `PASS:` thử gắn nhãn `enforce=restricted` cho `bookstore` ở chế độ dry-run server không ra cảnh báo
  nào.
- `PASS:` chạy `once` hai lần khi mọi thứ đã khớp: không `DbBackup` nào bị ghi, không Job nào được tạo.
- `PASS:` `DbBackup` tên dài đúng 54 ký tự: Job `dbbackup-<tên>` 63 ký tự được tạo, backup `Succeeded`.
- `PASS:` có runbook, ADR, và một dòng nợ về `backup-operator` trong `docs/tech-debt.md`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-backup-operator.md`:

1. Ánh xạ vòng lặp của bạn vào mẫu controller của bài 177: đọc gì, làm gì, ghi gì. Vì sao mọi trạng
   thái nằm trong `status` của `DbBackup` thay vì trong bộ nhớ hay một file của operator?
2. Job mang `component=backup` chứ không phải `operator`. Nó tái dùng những hợp đồng nào của các
   sprint trước nhờ label đó, và lớp nào sẽ chặn nó nếu gắn sai?
3. Đối chiếu bảy bước của ví dụ `SampleDB` (bài 181) với `backup-operator`: bước nào có, bước nào khác,
   bước nào không có. Khác biệt lớn nhất là gì, và nó thành nợ nào?
4. TTL của Job và `ownerReferences` cùng dọn Job — mỗi cơ chế dọn trong trường hợp nào? Vì sao TTL
   không làm mất kết quả backup?
5. Vì sao đọc kết quả từ log của **Pod thành công** thay vì `kubectl logs job/<tên>`?
6. Tạo Job thất bại: lỗi nào thử lại là đúng, lỗi nào thử lại là vô ích? CRD đã chặn tên dài ở B2 —
   vì sao controller vẫn cần nhánh ghi `Failed`? Và vì sao sửa `spec` của một `DbBackup` đã xong không
   làm nó chạy lại?

<details>
<summary>Gợi ý</summary>

1. Đọc nhiều trường của một object trong một lệnh:
   `k get "$RES" "$1" -o jsonpath='{.status.phase}|{.metadata.uid}|{.apiVersion}|{.metadata.generation}|{.spec.format}'`
   rồi tách bằng `IFS='|' read -r ...`. Trường chưa có thì jsonpath in rỗng.
2. `apiVersion` dùng cho `ownerReferences` lấy từ chính object vừa đọc — B6 sẽ đổi nó mà bạn không phải
   sửa script.
3. Tạo Job bằng `job_manifest ... | k create -f -` (operator có quyền `create`, không có `patch`).
4. Condition của Job: `-o jsonpath='{.status.conditions[?(@.status=="True")].type}'`, so khớp
   ` Complete ` và ` Failed `.
5. Pod thành công của một Job:
   `-l batch.kubernetes.io/job-name=<job> --field-selector=status.phase=Succeeded`.
6. Pod Security `restricted` cho image `postgres` không khai `USER`: `runAsNonRoot`, `runAsUser: 999`,
   `seccompProfile: RuntimeDefault`, `allowPrivilegeEscalation: false`, drop mọi capability.
7. Job phải tự được dọn sau vài phút (`ttlSecondsAfterFinished`): trần `count/jobs.batch` đã nâng ở B3
   chỉ đủ cho vài Job của operator cùng lúc.
8. Trong heredoc không đặt nháy (`<<YAML`), biến của shell trong Pod phải viết `\$` để không bị mở rộng
   ngay trên master.
9. Chạy thử từng vòng: `./scripts/backup-operator.sh stop; ./scripts/backup-operator.sh once`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/14; W=~/lab-work/14; GRP=bookstore.example.com
cd ~/bookstore
s() { kubectl -n "$NS" get dbbackup "$1" -o jsonpath="$2" 2>/dev/null; }
wait_phase() { for _ in $(seq 1 80); do case "$(s "$1" '{.status.phase}')" in Succeeded|Failed) return 0;; esac; sleep 3; done; return 1; }

./scripts/teardown.sh > "$EV/b4-teardown.txt" 2>&1
grep -q 'teardown: sach' "$EV/b4-teardown.txt" && echo 'PASS: teardown.sh go sach, operator da dung'
./scripts/deploy.sh > "$EV/b4-deploy-1.txt" 2>&1 && echo 'PASS: deploy.sh dung lai he thong va baseline Succeeded'
./scripts/deploy.sh > "$EV/b4-deploy-2.txt" 2>&1 && echo 'PASS: deploy.sh lan 2 thanh cong'
./scripts/backup-operator.sh status && grep -q 'bat dau' secrets/backup-operator.log && echo 'PASS: operator dang chay nen'

[ "$(s baseline '{.status.phase}')" = 'Succeeded' ] && echo 'PASS: baseline Succeeded'
BAD=0
for F in jobName file sizeBytes startTime completionTime observedGeneration; do
  [ -n "$(s baseline "{.status.$F}")" ] || { echo "FAIL: baseline thieu status.$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: status cua baseline du sau truong'
[ "$(s baseline '{.status.observedGeneration}')" = "$(s baseline '{.metadata.generation}')" ] \
  && echo 'PASS: observedGeneration bang generation'
FILE="$(s baseline '{.status.file}')"; SIZE="$(s baseline '{.status.sizeBytes}')"
case "$FILE" in /backup/*.sql) [ "${SIZE:-0}" -gt 0 ] && echo "PASS: $FILE, $SIZE byte" ;; esac

cat > "$W/doc-backup.yaml" <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: doc-backup
  namespace: bookstore
spec:
  restartPolicy: Never
  terminationGracePeriodSeconds: 1
  automountServiceAccountToken: false
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    runAsGroup: 65534
    seccompProfile: {type: RuntimeDefault}
  volumes:
  - name: backup
    persistentVolumeClaim: {claimName: backup-store, readOnly: true}
  containers:
  - name: doc
    image: busybox:1.37
    command: ["sh", "-c", "sleep 600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities: {drop: [ALL]}
    resources:
      requests: {cpu: 10m, memory: 8Mi}
      limits: {cpu: 50m, memory: 16Mi}
    volumeMounts:
    - {name: backup, mountPath: /backup, readOnly: true}
YAML
kubectl apply -f "$W/doc-backup.yaml" >/dev/null
kubectl -n "$NS" wait --for=condition=Ready pod/doc-backup --timeout=180s >/dev/null
[ "$(kubectl -n "$NS" exec doc-backup -- stat -c %s "$FILE")" = "$SIZE" ] \
  && echo 'PASS: file tren PVC backup-store dung kich thuoc status ghi'
kubectl -n "$NS" exec doc-backup -- ls -l /backup > "$EV/b4-backup-store.txt"

kubectl apply -f - >/dev/null <<YAML
apiVersion: $GRP/v1alpha1
kind: DbBackup
metadata:
  name: b4-custom
  namespace: $NS
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: backup}
spec: {database: bookstore, format: custom, note: thu dinh dang custom}
YAML
wait_phase b4-custom
{ [ "$(s b4-custom '{.status.phase}')" = 'Succeeded' ] && s b4-custom '{.status.file}' | grep -q '\.dump$'; } \
  && echo 'PASS: format custom cho file .dump'
J="$(s b4-custom '{.status.jobName}')"
jj() { kubectl -n "$NS" get job "$J" -o jsonpath="$1" 2>/dev/null; }
kubectl -n "$NS" get job "$J" -o yaml > "$EV/b4-job.yaml"
[ "$J" = 'dbbackup-b4-custom' ] && echo 'PASS: ten Job theo hop dong'
[ "$(jj '{.metadata.labels.app\.kubernetes\.io/component}|{.metadata.labels.app\.kubernetes\.io/managed-by}|{.metadata.labels.bookstore\.example\.com/dbbackup}')" \
  = 'backup|backup-operator|b4-custom' ] && echo 'PASS: label cua Job theo hop dong'
[ "$(jj '{.metadata.ownerReferences[0].kind}|{.metadata.ownerReferences[0].uid}|{.metadata.ownerReferences[0].controller}')" \
  = "DbBackup|$(s b4-custom '{.metadata.uid}')|true" ] && echo 'PASS: Job thuoc dung DbBackup qua ownerReferences'
[ "$(jj '{.spec.template.spec.priorityClassName}|{.spec.template.spec.automountServiceAccountToken}')" = 'bookstore-batch|false' ] \
  && echo 'PASS: PriorityClass bookstore-batch, khong mount token'
{ [ -n "$(jj '{.spec.ttlSecondsAfterFinished}')" ] && [ -n "$(jj '{.spec.template.spec.containers[0].resources.requests.cpu}')" ] \
  && [ -n "$(jj '{.spec.template.spec.containers[0].resources.limits.memory}')" ]; } && echo 'PASS: co TTL, requests, limits'
# Đặt lại đúng mức enforce đang có thì policy không đổi, Pod đang chạy không được đánh giá lại;
# đổi enforce-version mới buộc đánh giá (cùng cách với smoke 9b).
# Lệnh dry-run phải thành công thì "không có cảnh báo" mới có nghĩa; lỗi công cụ là FAIL.
if W1="$(kubectl label --dry-run=server --overwrite namespace "$NS" pod-security.kubernetes.io/enforce-version=latest 2>&1)"; then
  echo "$W1" | grep -q 'violate' && echo "FAIL: $W1" || echo 'PASS: moi Pod trong bookstore dat restricted, ke ca Pod cua operator'
else
  echo "FAIL: khong hoi duoc Pod Security bang dry-run: $W1"
fi

RV0="$(s baseline '{.metadata.resourceVersion}')"
JN0="$(kubectl -n "$NS" get jobs -l app.kubernetes.io/managed-by=backup-operator --no-headers | wc -l)"
./scripts/backup-operator.sh once >/dev/null; ./scripts/backup-operator.sh once >/dev/null
RV1="$(s baseline '{.metadata.resourceVersion}')"
JN1="$(kubectl -n "$NS" get jobs -l app.kubernetes.io/managed-by=backup-operator --no-headers | wc -l)"
{ [ "$RV0" = "$RV1" ] && [ "$JN0" = "$JN1" ]; } && echo 'PASS: chay lai vong lap khi da khop khong ghi gi, khong tao Job'

# Tên ở biên: 54 ký tự là dài nhất CRD nhận; Job dbbackup-<tên> dài đúng 63 ký tự vẫn phải tạo được.
kubectl -n "$NS" delete dbbackup b4-custom --wait=true >/dev/null   # trả chỗ trong count/jobs.batch
NB="$(printf 'b%.0s' $(seq 1 54))"
kubectl apply -f - >/dev/null <<YAML
apiVersion: $GRP/v1alpha1
kind: DbBackup
metadata:
  name: $NB
  namespace: $NS
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: backup}
spec: {database: bookstore, note: ten o bien}
YAML
wait_phase "$NB"
{ [ "$(s "$NB" '{.status.phase}')" = 'Succeeded' ] && [ "$(s "$NB" '{.status.jobName}')" = "dbbackup-$NB" ] \
  && [ "$(printf 'dbbackup-%s' "$NB" | wc -c)" -eq 63 ]; } \
  && echo 'PASS: ten 54 ky tu: Job 63 ky tu duoc tao, backup Succeeded'
kubectl -n "$NS" delete dbbackup "$NB" --wait=true >/dev/null

{ test -s docs/runbook-backup-operator.md && grep -qE '^\| *TD-[0-9]+ .*backup-operator' docs/tech-debt.md \
  && ls docs/adr/*-backup-operator.md >/dev/null 2>&1; } && echo 'PASS: co runbook, ADR va no operator ngoai cluster'
kubectl -n "$NS" delete pod doc-backup --wait=true >/dev/null
kubectl -n "$NS" delete dbbackup b4-custom --ignore-not-found --wait=true >/dev/null
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Hai hàm thay vào khối `PHẦN BẠN VIẾT`:

```bash
job_manifest() {    # $1 = tên DbBackup, $2 = uid, $3 = apiVersion, $4 = plain|custom → in YAML của Job
  local fmt=p ext=sql
  if [ "$4" = custom ]; then fmt=c; ext=dump; fi
  cat <<YAML
apiVersion: batch/v1
kind: Job
metadata:
  name: dbbackup-$1
  namespace: $NS
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: backup
    app.kubernetes.io/name: postgresql
    app.kubernetes.io/managed-by: backup-operator
    bookstore.example.com/dbbackup: "$1"
  ownerReferences:
  - apiVersion: $3
    kind: DbBackup
    name: "$1"
    uid: "$2"
    controller: true
    blockOwnerDeletion: false
spec:
  backoffLimit: 1
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 600
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: backup
        app.kubernetes.io/name: postgresql
        app.kubernetes.io/managed-by: backup-operator
        bookstore.example.com/dbbackup: "$1"
    spec:
      restartPolicy: Never
      priorityClassName: bookstore-batch
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 999
        runAsGroup: 999
        seccompProfile:
          type: RuntimeDefault
      volumes:
      - name: backup
        persistentVolumeClaim:
          claimName: backup-store
      containers:
      - name: pg-dump
        image: postgres:17.11
        env:
        - {name: PGHOST, value: db-0.db}
        - {name: PGUSER, value: postgres}
        - {name: PGDATABASE, value: bookstore}
        - {name: PGCONNECT_TIMEOUT, value: "5"}
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef: {name: db-credentials, key: postgres-password}
        - {name: FMT, value: $fmt}
        - {name: EXT, value: $ext}
        - {name: OUT, value: /backup/dbbackup-$1}
        command:
        - sh
        - -c
        - |
          set -e
          find /backup -maxdepth 1 -name 'dbbackup-*.tmp' -mmin +60 -delete
          F="\$OUT-\$(date -u +%Y%m%dT%H%M%SZ).\$EXT"
          pg_dump --no-owner --format="\$FMT" --file="\$F.tmp"
          if [ "\$FMT" = c ]; then pg_restore --list "\$F.tmp" | grep -q 'TABLE api books'
          else grep -q 'CREATE TABLE api.books' "\$F.tmp"; fi
          mv "\$F.tmp" "\$F"
          ls -1t /backup/dbbackup-*.sql /backup/dbbackup-*.dump 2>/dev/null | tail -n +11 | xargs -r rm -f
          echo "db-backup: OK \$F \$(wc -c < "\$F") byte"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: [ALL]
        resources:
          requests: {cpu: 50m, memory: 64Mi}
          limits: {cpu: 200m, memory: 128Mi}
        volumeMounts:
        - {name: backup, mountPath: /backup}
YAML
}

reconcile_one() {   # $1 = tên một DbBackup → đưa thực tế về đúng spec, ghi status; lỗi thì return 1
  local name="$1" line phase uid av gen fmt job conds pod out file size err
  line="$(k get "$RES" "$name" \
    -o jsonpath='{.status.phase}|{.metadata.uid}|{.apiVersion}|{.metadata.generation}|{.spec.format}')" || return 1
  IFS='|' read -r phase uid av gen fmt <<< "$line"
  case "$phase" in Succeeded|Failed) return 0 ;; esac            # yêu cầu đã xong: không làm gì
  job="dbbackup-$name"

  if ! k get job "$job" >/dev/null 2>&1; then
    if [ "$phase" = Running ]; then                              # Job mất giữa chừng: báo thật
      set_status "$name" "{\"phase\":\"Failed\",\"completionTime\":\"$(now)\",\"message\":\"Job $job bien mat truoc khi xong\"}"
      log "$name: Failed — Job bien mat"; return 0
    fi
    if ! err="$(job_manifest "$name" "$uid" "$av" "${fmt:-plain}" | k create -f - 2>&1 >/dev/null)"; then
      case "$err" in
        *' is invalid'*)                                         # thử lại bao nhiêu lần cũng vậy
          set_status "$name" "{\"phase\":\"Failed\",\"completionTime\":\"$(now)\",\"message\":\"Job $job khong hop le, khong tao duoc\"}"
          log "$name: Failed — $err"; return 0 ;;
      esac
      log "$name: chua tao duoc Job — $err"; return 1            # quota, mạng: vòng sau thử lại
    fi
    set_status "$name" "{\"phase\":\"Running\",\"jobName\":\"$job\",\"startTime\":\"$(now)\",\"observedGeneration\":$gen}"
    log "$name: tao Job $job"; return 0
  fi

  conds="$(k get job "$job" -o jsonpath='{.status.conditions[?(@.status=="True")].type}')"
  case " $conds " in
    *' Complete '*)
      pod="$(k get pods -l "batch.kubernetes.io/job-name=$job" --field-selector=status.phase=Succeeded \
               -o jsonpath='{.items[0].metadata.name}')"
      out="$(k logs "$pod" 2>/dev/null | grep '^db-backup: OK' | tail -n 1)"
      file="$(echo "$out" | awk '{print $3}')"; size="$(echo "$out" | awk '{print $4}')"
      case "$size" in
        ''|*[!0-9]*)
          set_status "$name" "{\"phase\":\"Failed\",\"completionTime\":\"$(now)\",\"message\":\"Job xong nhung khong doc duoc dong db-backup: OK\"}"
          log "$name: Failed — khong doc duoc ket qua"; return 0 ;;
      esac
      set_status "$name" "{\"phase\":\"Succeeded\",\"file\":\"$file\",\"sizeBytes\":$size,\"completionTime\":\"$(now)\",\"observedGeneration\":$gen,\"message\":\"\"}"
      log "$name: Succeeded $file $size byte" ;;
    *' Failed '*)
      set_status "$name" "{\"phase\":\"Failed\",\"completionTime\":\"$(now)\",\"message\":\"Job $job that bai, xem log cua Pod\"}"
      log "$name: Failed — Job that bai" ;;
    *)                                                           # Job đang chạy
      [ "$phase" = Running ] || set_status "$name" "{\"phase\":\"Running\",\"jobName\":\"$job\",\"startTime\":\"$(now)\",\"observedGeneration\":$gen}" ;;
  esac
}
```

```yaml
# k8s/app/backup/dbbackup-baseline.yaml — B6 đổi apiVersion sang v1
apiVersion: bookstore.example.com/v1alpha1
kind: DbBackup
metadata:
  name: baseline
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: backup
    app.kubernetes.io/name: postgresql
spec:
  database: bookstore
  note: Ban sao luu nen sau moi lan dung he thong tu repo
```

Thêm `backup/dbbackup-baseline.yaml` vào `resources` của `k8s/app/kustomization.yaml`. Dòng mới của
`docs/tech-debt.md`:

```markdown
| TD-20 | backup-operator chạy ngoài cluster như tiến trình nền trên master: không HA, chết khi master khởi động lại, token 24 giờ phải cấp lại bằng deploy.sh | 14 | Phần II, giai đoạn 28 | mở | docs/runbook-backup-operator.md |
```


Khung `docs/runbook-backup-operator.md` — năm mục: *Chạy và dừng* (`start`, `stop`, `status`, log ở
`secrets/backup-operator.log`); *Sau khi master khởi động lại* (`./scripts/deploy.sh`); *Token hết hạn*
(dấu hiệu `Unauthorized` trong log, cấp lại bằng `./scripts/deploy.sh`); *Yêu cầu một bản backup* (một
manifest `DbBackup`, `kubectl -n bookstore get dbb -w`); *Chép bản dump ra máy host* (Pod đọc
`backup-store` như `doc-backup`, `kubectl cp`, rồi `scp` — điều kiện của `TD-10`).

Ý chính cho ADR:

1. Đọc `spec` (và `phase`), làm cho thực tế khớp (tạo Job), cập nhật `status` — đúng câu của bài 177:
   controller "đọc `.spec` của một object, có thể thực hiện một số việc, rồi cập nhật `.status`". Mọi
   trạng thái nằm trong API server nên operator **không có trạng thái riêng**: chết, khởi động lại,
   chạy hai bản `once` song song — mỗi vòng đọc lại sự thật từ đầu. B5 chứng minh.
2. `component=backup` cho Job đi qua đúng ô ma trận NetworkPolicy tới `db` cổng 5432 mà sprint 7 đã mở
   cho Pod sao lưu, cùng nhóm với CronJob `db-backup`. Gắn `component=operator` thì default deny chặn
   kết nối tới `db`: `pg_dump` hết thời gian chờ kết nối, Job `Failed`. Pod Security và quota không xét
   label — chúng xét spec: thiếu `seccompProfile` thì bị từ chối ngay lúc tạo Pod, thiếu `requests`
   thì quota từ chối.
3. Có: bước 1 (custom resource), 4 (truy vấn control plane — bằng `list` mỗi vòng thay cho `watch`), 5
   (làm thực tế khớp — một Job thay cho PVC, StatefulSet, Job). Khác: bước 5 phần xóa — garbage
   collector xóa Job, không có bước "trước khi xóa" (B5). Không có: bước 2 và 3 (Deployment, image —
   operator chạy trên master), bước 6 (lịch — đã có CronJob `db-backup`), bước 7 (nâng cấp). Khác biệt
   lớn nhất là **nơi chạy**: không Deployment nên không ai khởi động lại nó — nợ *operator ngoài cluster*.
4. TTL dọn Job đã xong sau mười phút dù `DbBackup` còn — giữ `count/jobs.batch` dưới trần;
   `ownerReferences` dọn Job khi `DbBackup` bị xóa, kể cả Job chưa xong. Kết quả nằm trong `status` của `DbBackup` và file trên PVC, không nằm trong Job —
   miễn operator ghi `status` trước khi TTL hết (B5 cho thấy điều gì xảy ra nếu không).
5. `backoffLimit: 1` cho phép một Pod thất bại rồi một Pod thành công; `kubectl logs job/<tên>` chọn
   **một** Pod của Job, có thể đúng Pod thất bại — operator sẽ ghi `Failed` cho một bản backup thực ra
   đã xong.
6. Quota đầy hay API server không trả lời là lỗi **tạm thời**: TTL dọn Job cũ, mạng trở lại, vòng sau
   tạo được. API server báo Job `is invalid` là lỗi **của đầu vào**: thử lại bao nhiêu vòng cũng cùng một
   kết quả, nên ghi `Failed` để người yêu cầu thấy ngay trên object. Chặn tên ở CRD (B2) làm nhánh này
   hiếm, nhưng controller không được giả định mọi đầu vào đều đã qua kiểm tra — schema có thể đổi, hoặc
   object cũ được tạo trước khi có ràng buộc. `DbBackup` là yêu cầu một lần: `phase` đã kết thúc thì
   vòng lặp bỏ qua, nên sửa `spec` chỉ tăng `generation` — `observedGeneration` ở `status` cho thấy
   kết quả ứng với `spec` nào. Muốn bản mới thì tạo `DbBackup` mới.

</details>

## B5. [BS-14-05] Diễn tập: CRD không có controller

**Loại:** Diễn tập sự cố — chỉ trên tiến trình `backup-operator` và object của dự án; không node nào bị
chạm tới.

**Bối cảnh:** bài 177 nói controller tùy chỉnh thêm một **điểm lỗi**. Trước khi đưa `R19` vào nghiệm
thu, đội phải thấy điểm lỗi đó hỏng ra sao và phần còn lại của hệ thống chịu thế nào: controller vắng
mặt, Job biến mất giữa chừng, yêu cầu bị xóa, tiến trình bị giết thô.

**Yêu cầu:**

1. **Kịch bản A — không có controller.** Dừng operator, tạo `DbBackup` `kho-du-lieu`. Chứng minh trong
   một khoảng quan sát có giới hạn: không Job, không Pod, `status` rỗng, không Event nào về object đó —
   nhưng object đọc được đầy đủ, kể cả bằng danh tính của operator, và ứng dụng vẫn phục vụ. Chạy lại
   operator: yêu cầu thành công mà không ai tạo lại nó.
2. **Kịch bản B — Job mất giữa chừng.** Với operator dừng, tạo `mat-job`, chạy **một** vòng để có Job và
   `phase: Running`, xóa Job, chạy vòng tiếp: `phase: Failed` có `message`, không Job mới; vòng sau đó
   không ghi gì nữa.
3. **Kịch bản C — xóa yêu cầu.** Xóa `kho-du-lieu`: Job của nó biến mất dù operator không có quyền xóa
   Job; file dump vẫn còn trên PVC.
4. ⭐ **Kịch bản D — tiến trình bị giết thô.** `kill -9` tiến trình operator: `status` phát hiện nó đã
   chết dù file PID còn; `start` chạy lại được; không `DbBackup` nào có hai Job.
5. Viết postmortem `docs/postmortem/14-controller-vang-mat.md`.

**Ràng buộc:** không sửa script operator trong lúc diễn tập. Không xóa `baseline`. Xóa mọi `DbBackup`
diễn tập khi xong.

**Đọc lại:**

- Bài [179](../179-custom-resources-vi.md), mục
  [*Custom controller*](../179-custom-resources-vi.md#custom-controller).
- Bài [181](../181-operator-vi.md), [*Một ví dụ về operator*](../181-operator-vi.md#example) — bước 5,
  phần "nếu bạn xóa nó";
  [*Triển khai operator*](../181-operator-vi.md#triển-khai-operator-deploying-operators).
- Bài [177](../177-extend-kubernetes-vi.md), mục
  [*Các mẫu thiết kế phần mở rộng*](../177-extend-kubernetes-vi.md#extension-patterns) — điểm lỗi.
- [Lab 1c](LAB-1C-VONG-DOI-VA-CO-CHE-NEN-CUA-OBJECT.md) — owner reference, garbage collection,
  finalizer.

**Bàn giao:** `docs/postmortem/14-controller-vang-mat.md`; `~/lab-evidence/14/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` A — 30 giây không Job, `status` rỗng, không Event; `spec` đủ cả giá trị mặc định; danh tính
  operator đọc được object; `/books` vẫn trả dữ liệu; chạy lại operator thì `Succeeded`.
- `PASS:` B — vòng 1 cho `Running`; sau khi Job bị xóa, vòng 2 cho `Failed` có `message`, không Job mới;
  vòng 3 không đổi `resourceVersion`.
- `PASS:` C — Job biến mất theo `DbBackup`; operator không có quyền xóa Job; file dump còn trên PVC.
- `PASS:` ⭐ D — `status` báo không chạy sau `kill -9`; `start` chạy lại được; không `DbBackup` nào có
  hai Job.
- `PASS:` có postmortem; không còn `DbBackup` diễn tập.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Ở kịch bản A, controller vắng mặt đúng lúc yêu cầu được tạo, vậy mà yêu cầu không mất. Vì sao?
   Lúc controller dừng, bạn đang có gì và chưa có gì?
2. Ở kịch bản B, vì sao ghi `Failed` thay vì tạo lại Job? Khi nào tạo lại mới là đúng?
3. Ở kịch bản C, ai xóa Job, dựa vào trường nào? Vì sao file dump còn — có nên xóa theo không? Bước 5
   của `SampleDB` làm một việc **trước** khi xóa: cơ chế nào cho phép điều đó?
4. Operator chết ba ngày mà không ai biết. Hệ thống hỏng ở đâu, **không** hỏng ở đâu? Yêu cầu nào trong
   `R1`–`R19` bị ảnh hưởng?

<details>
<summary>Gợi ý</summary>

- Một vòng duy nhất, không nền: `./scripts/backup-operator.sh once` khi operator đang dừng.
- Event về một object:
  `kubectl get events --field-selector involvedObject.kind=DbBackup,involvedObject.name=<tên>`.
- Đọc file trên PVC: dùng lại Pod `doc-backup` mà script nghiệm thu B4 đã ghi ở
  `~/lab-work/14/doc-backup.yaml`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đọc yêu cầu; script này chính là cuộc diễn tập</summary>

```bash
NS=bookstore; EV=~/lab-evidence/14; W=~/lab-work/14; GRP=bookstore.example.com
OP="system:serviceaccount:$NS:backup-operator"
cd ~/bookstore
s() { kubectl -n "$NS" get dbbackup "$1" -o jsonpath="$2" 2>/dev/null; }
mk() { kubectl apply -f - >/dev/null <<YAML
apiVersion: $GRP/v1alpha1
kind: DbBackup
metadata:
  name: $1
  namespace: $NS
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: backup}
spec: {database: bookstore, note: dien tap B5}
YAML
}
wait_phase() { for _ in $(seq 1 80); do [ "$(s "$1" '{.status.phase}')" = "$2" ] && return 0; sleep 3; done; return 1; }

# Kịch bản A — không có controller
./scripts/backup-operator.sh stop >/dev/null
mk kho-du-lieu
sleep 30
JN="$(kubectl -n "$NS" get jobs -l bookstore.example.com/dbbackup=kho-du-lieu --no-headers 2>/dev/null | wc -l)"
EN="$(kubectl -n "$NS" get events --field-selector involvedObject.kind=DbBackup,involvedObject.name=kho-du-lieu \
      --no-headers 2>/dev/null | wc -l)"
kubectl -n "$NS" get dbb kho-du-lieu -o yaml > "$EV/b5-a-kho-du-lieu.yaml"
[ "$JN" -eq 0 ] && echo 'PASS: A — 30 giay khong Job nao duoc tao'
[ -z "$(s kho-du-lieu '{.status.phase}')" ] && echo 'PASS: A — status rong, khong ai ghi'
[ "$EN" -eq 0 ] && echo 'PASS: A — khong thanh phan nao phat Event ve DbBackup'
[ "$(s kho-du-lieu '{.spec.format}')" = 'plain' ] && echo 'PASS: A — du lieu day du, ke ca gia tri mac dinh'
kubectl --kubeconfig=secrets/backup-operator.kubeconfig -n "$NS" get dbbackup kho-du-lieu >/dev/null 2>&1 \
  && echo 'PASS: A — danh tinh controller doc duoc: thieu vong lap, khong thieu quyen'
WEB="$(kubectl -n "$NS" get pod -l app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web \
       --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')"
T="$(kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -q -T 5 -O- http://api:3000/books 2>/dev/null | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && echo "PASS: A — ung dung van phuc vu $T cuon sach"
./scripts/backup-operator.sh start >/dev/null
wait_phase kho-du-lieu Succeeded && echo 'PASS: A — controller chay lai thi hoi tu, khong mat yeu cau'

# Kịch bản B — Job mất giữa chừng
./scripts/backup-operator.sh stop >/dev/null
mk mat-job
./scripts/backup-operator.sh once > "$EV/b5-b-vong-1.txt" 2>&1
[ "$(s mat-job '{.status.phase}')" = 'Running' ] && echo 'PASS: B — vong 1 tao Job, phase Running'
kubectl -n "$NS" delete job dbbackup-mat-job --wait=true >/dev/null
./scripts/backup-operator.sh once > "$EV/b5-b-vong-2.txt" 2>&1
{ [ "$(s mat-job '{.status.phase}')" = 'Failed' ] && [ -n "$(s mat-job '{.status.message}')" ]; } \
  && echo 'PASS: B — Job mat khi Running: controller ghi Failed kem message'
kubectl -n "$NS" get job dbbackup-mat-job >/dev/null 2>&1 \
  && echo 'FAIL: B — controller tao lai Job' || echo 'PASS: B — khong Job moi nao duoc tao lai'
RV0="$(s mat-job '{.metadata.resourceVersion}')"
./scripts/backup-operator.sh once >/dev/null 2>&1
[ "$(s mat-job '{.metadata.resourceVersion}')" = "$RV0" ] && echo 'PASS: B — phase cuoi khong bi ghi lai'
./scripts/backup-operator.sh start >/dev/null

# Kịch bản C — xóa yêu cầu
F="$(s kho-du-lieu '{.status.file}')"; JC=dbbackup-kho-du-lieu
kubectl -n "$NS" get job "$JC" >/dev/null 2>&1 && echo 'PASS: C — truoc khi xoa, Job cua kho-du-lieu con'
kubectl -n "$NS" delete dbbackup kho-du-lieu --wait=true >/dev/null
for _ in $(seq 1 30); do kubectl -n "$NS" get job "$JC" >/dev/null 2>&1 || break; sleep 2; done
kubectl -n "$NS" get job "$JC" >/dev/null 2>&1 \
  && echo 'FAIL: C — Job con sau khi xoa DbBackup' || echo 'PASS: C — Job bien mat theo DbBackup'
[ "$(kubectl auth can-i delete jobs.batch -n "$NS" --as="$OP")" = 'no' ] \
  && echo 'PASS: C — operator khong co quyen xoa Job: nguoi xoa la garbage collector'
kubectl apply -f "$W/doc-backup.yaml" >/dev/null
kubectl -n "$NS" wait --for=condition=Ready pod/doc-backup --timeout=180s >/dev/null
kubectl -n "$NS" exec doc-backup -- test -s "$F" && echo "PASS: C — file $F van con tren PVC"
kubectl -n "$NS" delete pod doc-backup --wait=true >/dev/null

# ⭐ Kịch bản D — tiến trình bị giết thô
kill -9 "$(cat secrets/backup-operator.pid)"; sleep 1
./scripts/backup-operator.sh status >/dev/null 2>&1 \
  && echo 'FAIL: D — status van bao dang chay' || echo 'PASS: D — status phat hien tien trinh da chet'
./scripts/backup-operator.sh start >/dev/null && ./scripts/backup-operator.sh status >/dev/null \
  && echo 'PASS: D — start chay lai duoc'
DUP="$(kubectl -n "$NS" get jobs -l app.kubernetes.io/managed-by=backup-operator \
       -o jsonpath='{range .items[*]}{.metadata.labels.bookstore\.example\.com/dbbackup}{"\n"}{end}' | sort | uniq -d | wc -l)"
[ "$DUP" -eq 0 ] && echo 'PASS: D — khong DbBackup nao co hai Job'

kubectl -n "$NS" delete dbbackup mat-job --ignore-not-found --wait=true >/dev/null
[ "$(kubectl -n "$NS" get dbbackups --no-headers | grep -vc '^baseline ')" -eq 0 ] && echo 'PASS: chi con baseline'
test -s docs/postmortem/14-controller-vang-mat.md && echo 'PASS: co postmortem'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. Yêu cầu nằm trong **API server**, không nằm trong một sự kiện mà controller phải bắt kịp. Mỗi vòng
   lặp đọc lại toàn bộ trạng thái mong muốn rồi so với thực tế — bài 25 gọi là đưa trạng thái hiện tại
   về trạng thái mong muốn — nên chạy lại lúc nào cũng tìm thấy việc còn dở. Khi controller dừng, bạn có
   đủ một API: schema, validation, defaulting, `kubectl get`, RBAC, quota. Bạn **chưa có** API khai báo:
   bài 179 — "bản thân custom resource chỉ cho phép bạn lưu trữ và truy xuất dữ liệu có cấu trúc".
2. Controller không biết Job đã làm tới đâu: `pg_dump` có thể đã ghi xong file, cũng có thể đã ghi dở.
   `DbBackup` là **yêu cầu một lần**; tự tạo lại là quyết định thay người vận hành. Báo thật rồi để người
   tạo `DbBackup` mới. Tạo lại chỉ đúng khi tài nguyên là **trạng thái cần duy trì** (như ReplicaSet giữ
   số Pod), không phải một hành động đã diễn ra.
3. Garbage collector của control plane, dựa vào `metadata.ownerReferences` của Job trỏ về `DbBackup`
   (Lab 1c). File không phải object Kubernetes nên không ai xóa nó — và đó là điều đúng: bản backup phải
   sống lâu hơn yêu cầu tạo ra nó. Việc phải làm **trước** khi xóa (như tạo snapshot của `SampleDB`) cần
   một **finalizer** do controller đặt rồi tự gỡ khi xong việc; garbage collector không làm được.
4. Hỏng: `DbBackup` mới tồn đọng, không ai báo — `R19` ngừng, và thứ phát hiện được duy nhất là
   `PHASE` rỗng trong `kubectl get dbb`. Không hỏng: `web`, `api`, `db` phục vụ bình thường (controller
   không nằm trên đường request), CronJob `db-backup` vẫn sao lưu theo lịch (`R5`, `R8`). Hệ quả cho vận
   hành: cần một kiểm tra định kỳ "operator còn sống" — chính là dòng `R19` của `smoke.sh` — và một nơi
   chạy tự khởi động lại (nợ *operator ngoài cluster*).

</details>

## B6. [BS-14-06] Thêm version `v1` và chuyển storage version

**Loại:** Tính năng.

**Bối cảnh:** API `DbBackup` đã dùng thật qua hai ticket mà không đổi schema — đủ để tuyên bố `v1`.
Nhưng đổi version của CRD **không** tự ghi lại dữ liệu cũ: object tạo ở `v1alpha1` vẫn nằm trong kho
ở `v1alpha1` cho tới khi có thứ ghi lại nó (bài 323). Sprint này đổi storage version, đưa mọi object cũ
sang `v1` mà không mất `status`, không phải sửa operator.

**Yêu cầu:**

1. CRD phục vụ cả `v1` và `v1alpha1`, schema, subresource, cột in giống hệt nhau; `v1` là storage
   version; chiến lược chuyển đổi để mặc định.
2. Có ít nhất một `DbBackup` tạo ở `v1alpha1` **trước** khi đổi.
3. Chứng minh sau khi đổi: `status.storedVersions` có cả hai; cùng một object đọc qua hai version chỉ
   khác `apiVersion`; API server **từ chối** bỏ `v1alpha1` khỏi `spec.versions` và nói lý do.
4. Đưa mọi `DbBackup` về `v1` trong kho, rồi gỡ `v1alpha1` khỏi `status.storedVersions`. Chứng minh
   `status` của object không mất và API server bây giờ **chấp nhận** bỏ `v1alpha1` — chỉ thử, không bỏ
   thật.
5. `dbbackup-baseline.yaml` trong repo dùng `v1`; không manifest hay script nào của repo còn dùng
   `v1alpha1`; script operator không phải sửa; Job tạo sau khi đổi có `ownerReferences` trỏ `v1`.
6. `v1alpha1` vẫn được phục vụ nhưng không còn là storage. Ghi vào ADR lý do và điều kiện gỡ hẳn.

**Ràng buộc:** không dùng conversion webhook, không bật feature gate, không tạo `StorageVersionMigration`.
Sửa `k8s/cluster/crd-dbbackup.yaml` và `dbbackup-baseline.yaml` trong repo **rồi mới** chạy script
nghiệm thu — script tự apply CRD, nên đừng apply trước.

**Đọc lại:**

- Bài [323](../323-storage-version-migration-vi.md): đoạn mở đầu về việc dữ liệu phải được chủ động ghi
  lại; [*Trước khi bạn bắt đầu*](../323-storage-version-migration-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin)
  — vì sao cách tự động không dùng được trên cluster lab;
  [*Cập nhật storage schema ưu tiên của một CRD*](../323-storage-version-migration-vi.md#cập-nhật-storage-schema-ưu-tiên-của-một-crd-update-the-preferred-storage-schema-of-a-crd)
  — chuỗi `storage: true` → object cũ vẫn ở version cũ → `storedVersions` chỉ còn version mới.
- Bài [179](../179-custom-resources-vi.md), mục [*Lưu trữ*](../179-custom-resources-vi.md#storage).
- ⭐ Bài [377](../377-custom-resource-definition-versioning-vi.md), mục
  [*Nâng cấp các object hiện có lên storage version mới*](../377-custom-resource-definition-versioning-vi.md#upgrade-existing-objects-to-a-new-stored-version)
  — trang đó thuộc giai đoạn 28; ở đây chỉ cần đúng ba bước của *Phương án 2*, có sẵn trong gợi ý.

**Bàn giao:** `k8s/cluster/crd-dbbackup.yaml`, `k8s/app/backup/dbbackup-baseline.yaml` đã cập nhật;
`docs/adr/NNNN-dbbackup-v1.md`; `~/lab-evidence/14/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` trước khi đổi, `storedVersions` là `["v1alpha1"]` và `truoc-v1` thành công.
- `PASS:` sau khi áp CRD mới: storage là `v1`, `v1alpha1` vẫn phục vụ, `storedVersions` có cả hai;
  đọc qua hai version chỉ khác `apiVersion`.
- `PASS:` API server từ chối bỏ `v1alpha1`, lý do nằm ở `status.storedVersions`.
- `PASS:` mọi `DbBackup` được ghi lại (`resourceVersion` đổi), `status` giữ nguyên; `storedVersions`
  chỉ còn `["v1"]`; bây giờ bỏ `v1alpha1` được chấp nhận.
- `PASS:` repo không còn `v1alpha1` ngoài file CRD; Job mới trỏ `v1`, Job cũ vẫn trỏ `v1alpha1`.
- `PASS:` teardown rồi deploy: CRD dựng mới có `storedVersions` là `["v1"]`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-dbbackup-v1.md`:

1. `spec.versions` và `status.storedVersions` khác nhau thế nào? Vì sao API server từ chối bỏ
   `v1alpha1` lúc đầu?
2. Đặt `storage: true` cho `v1` có chuyển object cũ không? Trên cluster có hàng nghìn object thì bước nào
   tốn công nhất, công cụ nào sinh ra để làm nó, và vì sao lab không dùng công cụ đó?
3. Vì sao chiến lược chuyển đổi mặc định an toàn ở đây? Nếu `v1` đổi tên trường `note` thì sao?
4. API server đã cho bỏ `v1alpha1` — vì sao vẫn giữ nó? Kể ít nhất ba thứ có thể còn tham chiếu
   `v1alpha1`, và điều kiện để gỡ hẳn.
5. Đổi version có phải sửa Role, ClusterRole, quota nào không? Vì sao?

<details>
<summary>Gợi ý</summary>

- Hai phần tử trong `versions`, chỉ một phần tử có `storage: true`.
- Đọc một object qua version cụ thể: `kubectl get dbbackups.v1alpha1.bookstore.example.com <tên>` và
  `kubectl get dbbackups.v1.bookstore.example.com <tên>`.
- Ba bước ghi lại thủ công: (1) đặt version mới làm storage và apply; (2) liệt kê mọi object, ghi lại
  từng object với đúng nội dung cũ — `kubectl get ... -o yaml | kubectl replace -f -`; (3) vá `status`
  của chính CRD:
  `kubectl patch crd <tên> --subresource=status --type=merge -p '{"status":{"storedVersions":["v1"]}}'`.
- Thử bỏ một version mà không bỏ thật: `kubectl patch crd <tên> --type=json --dry-run=server -p
  '[{"op":"remove","path":"/spec/versions/<chỉ số>"}]'`.
- Đừng chạy bước (2) khi có `DbBackup` đang `Running` — operator đang ghi `status` của nó, `replace`
  sẽ gặp xung đột `resourceVersion`.

</details>

<details>
<summary>Script nghiệm thu — chạy ngay sau khi sửa hai file trong repo, CHƯA apply</summary>

```bash
NS=bookstore; EV=~/lab-evidence/14; GRP=bookstore.example.com; CRD=dbbackups.$GRP
cd ~/bookstore
sv() { kubectl get crd "$CRD" -o jsonpath='{.status.storedVersions}'; }
s() { kubectl -n "$NS" get dbbackup "$1" -o jsonpath="$2" 2>/dev/null; }
mk() { kubectl apply -f - >/dev/null <<YAML
apiVersion: $GRP/$2
kind: DbBackup
metadata:
  name: $1
  namespace: $NS
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: backup}
spec: {database: bookstore, note: tao o $2}
YAML
}
wait_ok() { for _ in $(seq 1 80); do [ "$(s "$1" '{.status.phase}')" = 'Succeeded' ] && return 0; sleep 3; done; return 1; }
idx() { local i=0 v; for v in $(kubectl get crd "$CRD" -o jsonpath='{.spec.versions[*].name}'); do
          [ "$v" = v1alpha1 ] && { echo "$i"; return; }; i=$((i + 1)); done; }

# 0. Trước khi đổi
[ "$(sv)" = '["v1alpha1"]' ] && echo 'PASS: truoc khi doi, storedVersions ["v1alpha1"]'
mk truoc-v1 v1alpha1
wait_ok truoc-v1 && echo 'PASS: truoc-v1 Succeeded khi CRD chi co v1alpha1'
OWNER0="$(kubectl -n "$NS" get job dbbackup-truoc-v1 -o jsonpath='{.metadata.ownerReferences[0].apiVersion}')"

# 1. Áp CRD mới từ repo
kubectl apply -f k8s/cluster/crd-dbbackup.yaml >/dev/null
kubectl wait --for=condition=Established "crd/$CRD" --timeout=60s >/dev/null
{ [ "$(kubectl get crd "$CRD" -o jsonpath='{.spec.versions[?(@.storage==true)].name}')" = 'v1' ] \
  && [ "$(kubectl get crd "$CRD" -o jsonpath='{.spec.versions[?(@.name=="v1alpha1")].served}')" = 'true' ]; } \
  && echo 'PASS: storage la v1, v1alpha1 van duoc phuc vu'
for _ in $(seq 1 15); do sv | grep -q '"v1"' && break; sleep 2; done
sv > "$EV/b6-stored-1.txt"; cat "$EV/b6-stored-1.txt"; echo
grep -q 'v1alpha1' "$EV/b6-stored-1.txt" && grep -q '"v1"' "$EV/b6-stored-1.txt" \
  && echo 'PASS: storedVersions co ca hai — object cu van nam o v1alpha1'
kubectl -n "$NS" get "dbbackups.v1alpha1.$GRP" truoc-v1 -o yaml > "$EV/b6-doc-v1alpha1.yaml"
kubectl -n "$NS" get "dbbackups.v1.$GRP" truoc-v1 -o yaml > "$EV/b6-doc-v1.yaml"
diff "$EV/b6-doc-v1alpha1.yaml" "$EV/b6-doc-v1.yaml" > "$EV/b6-diff.txt"
{ [ "$(grep -c '^[<>]' "$EV/b6-diff.txt")" -eq 2 ] && grep -q "^> apiVersion: $GRP/v1$" "$EV/b6-diff.txt"; } \
  && echo 'PASS: doc qua hai version chi khac apiVersion'

# 2. Bỏ v1alpha1 khi nó còn trong storedVersions
I="$(idx)"
kubectl patch crd "$CRD" --type=json --dry-run=server -p "[{\"op\":\"remove\",\"path\":\"/spec/versions/$I\"}]" \
  > "$EV/b6-tu-choi.txt" 2>&1 \
  && echo 'FAIL: API server cho bo v1alpha1 khi con trong storedVersions' \
  || { grep -q 'storedVersions' "$EV/b6-tu-choi.txt" && echo 'PASS: API server tu choi, ly do o status.storedVersions'; }

# 3. Ghi lại mọi DbBackup, rồi gỡ v1alpha1 khỏi storedVersions
RV0="$(s truoc-v1 '{.metadata.resourceVersion}')"; ST0="$(s truoc-v1 '{.status.phase}|{.status.file}|{.status.sizeBytes}')"
kubectl get dbbackups -A -o jsonpath='{range .items[*]}{.metadata.namespace} {.metadata.name}{"\n"}{end}' \
  | while read -r N O; do
      kubectl -n "$N" get dbbackup "$O" -o yaml | kubectl replace -f - >/dev/null && echo "da ghi lai $N/$O"
    done | tee "$EV/b6-ghi-lai.txt"
[ "$(s truoc-v1 '{.metadata.resourceVersion}')" != "$RV0" ] && echo 'PASS: object da duoc ghi lai o storage version moi'
[ "$(s truoc-v1 '{.status.phase}|{.status.file}|{.status.sizeBytes}')" = "$ST0" ] \
  && echo 'PASS: status giu nguyen — ghi resource chinh khong dung toi status'
kubectl patch crd "$CRD" --subresource=status --type=merge -p '{"status":{"storedVersions":["v1"]}}' >/dev/null
[ "$(sv)" = '["v1"]' ] && echo 'PASS: storedVersions chi con v1'
kubectl patch crd "$CRD" --type=json --dry-run=server -p "[{\"op\":\"remove\",\"path\":\"/spec/versions/$I\"}]" \
  >/dev/null 2>&1 && echo 'PASS: bay gio API server chap nhan bo v1alpha1 (chi thu, chua bo)'

# 4. Repo và operator
grep -q "^apiVersion: $GRP/v1$" k8s/app/backup/dbbackup-baseline.yaml && echo 'PASS: baseline trong repo dung v1'
grep -rl "$GRP/v1alpha1" k8s/app k8s/system scripts tests 2>/dev/null | grep -q . \
  && echo 'FAIL: con file dung v1alpha1' || echo 'PASS: repo khong con manifest, script nao dung v1alpha1'
mk sau-v1 v1
wait_ok sau-v1
{ [ "$(kubectl -n "$NS" get job dbbackup-sau-v1 -o jsonpath='{.metadata.ownerReferences[0].apiVersion}')" = "$GRP/v1" ] \
  && [ "$OWNER0" = "$GRP/v1alpha1" ]; } \
  && echo 'PASS: Job moi tro v1, Job cu van tro v1alpha1 — script operator khong doi'

# 5. Dựng lại sạch
kubectl -n "$NS" delete dbbackup truoc-v1 sau-v1 --wait=true >/dev/null
./scripts/teardown.sh > "$EV/b6-teardown.txt" 2>&1 && ./scripts/deploy.sh > "$EV/b6-deploy.txt" 2>&1 \
  && echo 'PASS: teardown + deploy sau khi doi version'
[ "$(sv)" = '["v1"]' ] && echo 'PASS: CRD dung moi co storedVersions ["v1"]'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/cluster/crd-dbbackup.yaml — bản cuối: v1 là storage, v1alpha1 còn phục vụ
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: dbbackups.bookstore.example.com
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: operator
spec:
  group: bookstore.example.com
  scope: Namespaced
  names:
    plural: dbbackups
    singular: dbbackup
    kind: DbBackup
    shortNames: [dbb]
    categories: [bookstore]
  versions:
  - name: v1
    served: true
    storage: true
    subresources: &sub              # &tên: neo YAML; *tên ở dưới dùng lại đúng khối này
      status: {}
    additionalPrinterColumns: &cols
    - {name: Phase, type: string, jsonPath: .status.phase}
    - {name: Size, type: integer, jsonPath: .status.sizeBytes}
    - {name: File, type: string, jsonPath: .status.file}
    - {name: Age, type: date, jsonPath: .metadata.creationTimestamp}
    schema: &schema
      openAPIV3Schema:
        type: object
        required: [spec]
        properties:
          metadata:                 # schema chỉ được ràng buộc name và generateName (bài 378)
            type: object
            properties:
              name: {type: string, maxLength: 54, pattern: '^[a-z0-9]([-a-z0-9]*[a-z0-9])?$'}
          spec:
            type: object
            required: [database]
            properties:
              database: {type: string, enum: [bookstore]}
              format: {type: string, enum: [plain, custom], default: plain}
              note: {type: string, maxLength: 120}
          status:
            type: object
            properties:
              phase: {type: string, enum: [Running, Succeeded, Failed]}
              jobName: {type: string}
              file: {type: string}
              sizeBytes: {type: integer, format: int64}
              startTime: {type: string, format: date-time}
              completionTime: {type: string, format: date-time}
              observedGeneration: {type: integer, format: int64}
              message: {type: string}
  - name: v1alpha1                 # còn phục vụ cho tới khi không còn gì tham chiếu — xem ADR
    served: true
    storage: false
    subresources: *sub             # schema, subresource, cột in giống hệt v1
    additionalPrinterColumns: *cols
    schema: *schema
```

Trong `k8s/app/backup/dbbackup-baseline.yaml` đổi dòng đầu thành
`apiVersion: bookstore.example.com/v1`.

Ý chính cho ADR:

1. `spec.versions` là những version **được phục vụ**; `status.storedVersions` là những version **từng
   được dùng để lưu**, do API server ghi. Còn `v1alpha1` trong `storedVersions` nghĩa là kho có thể còn
   object ở định dạng đó; bỏ nó khỏi `spec.versions` thì không còn cách đọc chúng lên.
2. **Không.** `storage: true` chỉ quyết định định dạng của những lần ghi **từ nay**. Bài 323: đổi
   storage schema chỉ tác động tới bản ghi được ghi mới — object cũ nằm im cho tới khi có thứ ghi lại
   nó. Bước tốn công là **ghi lại mọi object**; `StorageVersionMigration` sinh ra để tự động hóa đúng
   bước đó, nhưng cần feature gate và runtime config trên API server — sửa control plane, ngoài phạm vi
   lab.
3. Chiến lược mặc định chỉ đặt lại `apiVersion`, nên đúng khi mọi version **chung một schema** — ở đây
   giống hệt. Đổi tên `note` thì object `v1alpha1` đọc qua `v1` sẽ mất trường đó (bị cắt tỉa); khi ấy
   bắt buộc có conversion webhook.
4. Còn tham chiếu `v1alpha1`: `ownerReferences` của các Job tạo trước khi đổi; manifest và script ở
   ngoài repo (của người dùng, của CI) còn ghi `v1alpha1`; client đang chạy. Điều kiện gỡ hẳn:
   `storedVersions` chỉ còn `v1` (đã xong), không object nào còn `ownerReferences` trỏ `v1alpha1` (TTL đã
   dọn hết Job cũ), không client nào còn dùng — rồi mới bỏ khỏi `spec.versions` ở một release sau.
5. **Không.** Rule RBAC và khóa quota kể **group** và **resource** (`dbbackups`), không kể version.

</details>

## B7. [BS-14-07] Điều tra: thiết bị chuyên dụng và topology của node

**Loại:** Điều tra.

**Bối cảnh:** `R18` — đánh giá khả năng dùng thiết bị chuyên dụng cho một tính năng tương lai, "gợi ý
sách bằng GPU". Nếu đã làm [Lab 13](LAB-13-DRA.md), dự án có ADR về DRA; sprint này đóng nốt phía
**device plugin** và **Topology Manager** bằng bằng chứng đọc từ cluster thật, rồi chốt ADR.

**Yêu cầu:**

1. Chứng minh không node nào công bố resource dạng `vendor-domain/resourcetype`; trên
   `lab-k8s-worker2`, socket đăng ký của kubelet có mặt nhưng không plugin nào đăng ký.
2. Đối chiếu DRA: nhóm API `resource.k8s.io` có được phục vụ không, có `ResourceSlice` nào không.
3. Đọc `topologyManagerPolicy` và `memoryManagerPolicy` **hiệu lực** của ba kubelet cùng số NUMA node
   của máy, đủ để kết luận nhánh debug của bài 313 không thể kích hoạt trên cluster này.
4. ADR: để chạy tính năng dùng GPU trên hệ thống Bookstore cần những gì — phần cứng, plugin chạy ở
   namespace nào (nhớ Pod Security của ba namespace), Pod xin thiết bị ra sao, quota, và so với DRA.

**Ràng buộc:** chỉ đọc. Không sửa `KubeletConfiguration`, không đổi policy "để thử cho biết". Script
so checksum kubelet với B0.

**Đọc lại:**

- Bài [184](../184-device-plugins-vi.md):
  [*Đăng ký device plugin*](../184-device-plugins-vi.md#đăng-ký-device-plugin-device-plugin-registration),
  [*Hiện thực device plugin*](../184-device-plugins-vi.md#hiện-thực-device-plugin-device-plugin-implementation),
  [*Triển khai device plugin*](../184-device-plugins-vi.md#triển-khai-device-plugin-device-plugin-deployment),
  [*Tích hợp với Topology Manager*](../184-device-plugins-vi.md#device-plugin-integration-with-the-topology-manager).
- Bài [313](../313-debug-topology-vi.md):
  [*Các nguồn thông tin để khắc phục sự cố*](../313-debug-topology-vi.md#các-nguồn-thông-tin-để-khắc-phục-sự-cố-sources-of-troubleshooting-information),
  [*Khắc phục lỗi `TopologyAffinityError`*](../313-debug-topology-vi.md#TopologyAffinityError).
- Bài [182](../182-compute-storage-net-vi.md) — nhóm device plugin "bổ sung cho" `cpu` và `memory`.
- Bài [134](../134-resource-quotas-vi.md), mục *Hạn ngạch cho tài nguyên mở rộng*.

**Bàn giao:** `docs/adr/NNNN-thiet-bi-va-topology.md` — trỏ tới `docs/adr/*-dra-bookstore.md` nếu đã
làm Lab 13;
`~/lab-evidence/14/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` không node nào công bố resource `vendor-domain/resourcetype`.
- `PASS:` `kubelet.sock` có trên `lab-k8s-worker2`; không socket plugin nào khác.
- `PASS:` một dòng của nhánh DRA tương ứng với cluster.
- `PASS:` ba kubelet ở Topology Manager `none` và Memory Manager `None`; máy có một NUMA node.
- `PASS:` checksum kubelet không đổi so với B0; có ADR.

**Câu hỏi thiết kế** — trả lời trong ADR:

1. Luồng đăng ký device plugin có hai chặng. Chặng nào chưa từng xảy ra trên cluster của bạn, và bằng
   chứng nào cho từng chặng?
2. Device plugin dạng DaemonSet cần security context privileged và mount
   `/var/lib/kubelet/device-plugins`. Nó chạy được ở namespace nào của dự án, không được ở namespace
   nào, vì sao?
3. Pod gợi ý sách xin GPU bằng gì, chịu hai ràng buộc nào? Quota giới hạn GPU của namespace bằng khóa
   nào? DRA thay đổi điều gì so với cách đó?
4. Vì sao không bật thử policy Topology Manager khác `none` trên cluster lab, kể cả để quan sát?

<details>
<summary>Script nghiệm thu — chỉ đọc</summary>

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
EV=~/lab-evidence/14
cd ~/bookstore
for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  echo "$N capacity: $(kubectl get node "$N" -o jsonpath='{.status.capacity}')"
  echo "$N allocatable: $(kubectl get node "$N" -o jsonpath='{.status.allocatable}')"
done > "$EV/b7-capacity.txt"
EXT_N="$(grep -o '"[a-z0-9.-]\{1,\}/[a-z0-9.-]\{1,\}"' "$EV/b7-capacity.txt" | wc -l)"
[ "$EXT_N" -eq 0 ] && echo 'PASS: khong node nao cong bo resource vendor-domain/resourcetype'

ssh lab-k8s-worker2 'sudo ls -l /var/lib/kubelet/device-plugins/' > "$EV/b7-device-plugins.txt"
SOCK="$(ssh lab-k8s-worker2 'sudo ls -1 /var/lib/kubelet/device-plugins/' | grep '\.sock$')"
[ "$SOCK" = 'kubelet.sock' ] && echo 'PASS: chi co kubelet.sock — dang ky san sang, khong plugin nao dang ky'

kubectl api-resources --api-group=resource.k8s.io > "$EV/b7-dra.txt" 2>&1
if [ "$(grep -c resource.k8s.io "$EV/b7-dra.txt")" -gt 0 ]; then
  [ "$(sl kubectl get resourceslices --no-headers)" -eq 0 ] \
    && echo 'PASS: API DRA duoc phuc vu nhung khong driver nao cong bo thiet bi'
else
  echo 'PASS: nhom API DRA khong duoc phuc vu tren cluster nay'
fi

TM=0; MM=0
for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  kubectl get --raw "/api/v1/nodes/$N/proxy/configz" > "$EV/b7-configz-$N.json"
  T="$(grep -o '"topologyManagerPolicy":"[^"]*"' "$EV/b7-configz-$N.json" | sed 's/.*:"//;s/"$//')"
  M="$(grep -o '"memoryManagerPolicy":"[^"]*"' "$EV/b7-configz-$N.json" | sed 's/.*:"//;s/"$//')"
  echo "$N topologyManagerPolicy=${T:-<mac dinh none>} memoryManagerPolicy=${M:-<mac dinh None>}" | tee -a "$EV/b7-policy.txt"
  case "${T:-none}" in none) TM=$((TM + 1)) ;; esac
  case "${M:-None}" in None|none) MM=$((MM + 1)) ;; esac
done
NUMA="$(ssh lab-k8s-worker2 'ls -d /sys/devices/system/node/node[0-9]* | wc -l')"
{ [ "$TM" -eq 3 ] && [ "$MM" -eq 3 ] && [ "$NUMA" -eq 1 ]; } \
  && echo 'PASS: Topology Manager none, Memory Manager None, mot NUMA node — khong co ranh gioi nao de can'

{ echo "lab-k8s-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for N in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$N $(ssh "$N" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done; } > "$EV/b7-kubelet-sha.txt"
diff -q "$EV/b0-kubelet-sha.txt" "$EV/b7-kubelet-sha.txt" >/dev/null && echo 'PASS: cau hinh kubelet khong doi tu B0'
ls docs/adr/*thiet-bi* >/dev/null 2>&1 && echo 'PASS: co ADR thiet bi'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của ADR</summary>

1. Chặng một: plugin tự đăng ký với **kubelet** qua `kubelet.sock`, gửi `ResourceName` dạng
   `vendor-domain/resourcetype`. Chặng hai: **kubelet** công bố resource lên API server khi cập nhật
   trạng thái node. Không có tên nào dạng đó trong `capacity`/`allocatable` — chặng hai chưa xảy ra; thư
   mục đăng ký chỉ có `kubelet.sock` — chặng một cũng chưa. Socket có sẵn vì kubelet luôn phục vụ dịch
   vụ `Registration`.
2. Không ở `bookstore` hay `bookstore-staging`: cả hai `enforce: restricted`, cấm privileged và
   `hostPath`. `bookstore-system` (`privileged`) thì được về mặt Pod Security, nhưng đó là namespace của
   agent dự án; plugin của nhà cung cấp nên có namespace hạ tầng riêng như Calico, Traefik — tách quyền
   và vòng đời khỏi ứng dụng (bài 120, *Giảm thiểu việc phân phối các token đặc quyền*).
3. Bằng một **extended resource** trong `resources.limits` (ví dụ `nvidia.com/gpu: 1`): chỉ số nguyên,
   không overcommit, không chia sẻ giữa container. Quota dùng khóa `requests.<tên resource>` cho tài
   nguyên mở rộng (bài 134). DRA đưa thiết bị vào object API riêng của `resource.k8s.io`, mô tả được
   thuộc tính và cho Pod xin bằng claim — điều mà một cái tên và một con số không diễn đạt được.
4. Đổi policy là sửa `KubeletConfiguration` của node đang chạy — giai đoạn 20 — và với một NUMA node
   thì mọi cách sắp đặt đều thỏa, không có gì để quan sát. Cluster sẽ lệch mốc `04-metrics-ready`.

</details>

## B8. [BS-14-08] Nghiệm thu cuối dự án

**Loại:** Điều tra.

**Bối cảnh:** dự án bắt đầu từ một file `docker compose` với tám vấn đề `P1`–`P8` và mười chín yêu cầu
`R1`–`R19` ([mục 2 của file dự án](DU-AN-BOOKSTORE.md#2-bài-toán)). Hai mươi mốt sprint sau, đội phải
trả lời bằng **bằng chứng**, không bằng cảm nhận: yêu cầu nào đạt, đạt có điều kiện gì, nợ nào đã trả,
nợ nào còn và đi đâu. Đây là biên bản bàn giao cuối của dự án.

**Yêu cầu:**

1. `tests/smoke.sh` thay bằng **bản cuối của dự án** — chép nguyên văn khối dưới đây. Nó kiểm mọi hợp
   đồng còn hiệu lực ở mục 3 và 4 của file dự án; mỗi dòng ghi mã `R<n>` mà nó làm bằng chứng. Chạy sau
   một lần dựng sạch, phải PASS toàn bộ.
2. `docs/nghiem-thu.md`: một bảng `| Mã | Yêu cầu | Bằng chứng | Kết quả |` đủ `R1` → `R19`. Kết quả là
   `ĐẠT`, `ĐẠT CÓ ĐIỀU KIỆN` (ghi rõ điều kiện) hoặc `KHÔNG ÁP DỤNG` (ghi rõ lý do). Bằng chứng là dòng
   `PASS: R<n>` của smoke, hoặc file trong repo — ADR, postmortem, runbook — với đường dẫn **có thật**.
   `R9`, `R12`, `R18` không kiểm được trên một cluster đang chạy, và `R17` chỉ kiểm được một phần: bằng
   chứng của chúng là tài liệu của sprint tương ứng.
3. `docs/tech-debt.md`: **mọi** dòng `TD-nn` có trạng thái cuối — `đã trả`, `chấp nhận có điều kiện`
   hoặc `chuyển Phần II` — kèm bằng chứng hoặc lý do. Không còn dòng nào `mở`. Bảng có đủ `TD-01`
   → `TD-20` của sổ nợ trong file dự án, `TD-20` là nợ *operator ngoài cluster* của B4.
4. `docs/retro.md`: năm mục `## Đã làm tốt`, `## Chưa tốt`, `## Bài học`, `## Nợ còn lại`,
   `## Bước tiếp theo`. Mục nợ nêu **nợ lab** còn mở — `#1` (phần VPA), `#5`, `#6`, `#7`, `#8` — và nơi
   trả trong lộ trình.
5. `README.md` của repo mô tả trạng thái cuối và trỏ tới `docs/nghiem-thu.md`.
6. Vệ sinh repo: `secrets/` không có gì trong git ngoài `.gitkeep`; không còn file `.tpl`; không còn mật
   khẩu trong manifest hay script; overlay `k8s/overlays/ha-lab` vẫn render được; `platform/` còn ghi
   chép dựng cluster.

**Ràng buộc:** không sửa `smoke.sh` để nó PASS — hệ thống fail thì sửa hệ thống (trong repo), hoặc ghi
yêu cầu đó là `ĐẠT CÓ ĐIỀU KIỆN` với lý do thật. Không viết kết luận `ĐẠT` cho yêu cầu không có bằng
chứng.

**Đọc lại:**

- [Mục 2](DU-AN-BOOKSTORE.md#2-bài-toán), [mục 3](DU-AN-BOOKSTORE.md#3-kiến-trúc-đích),
  [mục 4](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật) và [mục 8](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án)
  của file dự án; `docs/tech-debt.md` của repo là bản đang sống của mục 8.
- [Sổ nợ lab](README.md#5-sổ-nợ-lab) và [sổ nợ lộ trình](../00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình).
- [Phần II — Vận hành cluster](../00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster) — giai đoạn 18, 19,
  20, 22, 23, 28.
- Bài [120](../120-rbac-good-practices-vi.md#workload-creation) — mục *Tạo workload*, cho điều kiện của
  `R13`.

**Bàn giao:** `tests/smoke.sh`; `docs/nghiem-thu.md`; `docs/tech-debt.md`; `docs/retro.md`;
`README.md`; `~/lab-evidence/14/b8-*.txt`.

**Nghiệm thu:**

- `PASS:` `smoke.sh` bản cuối PASS toàn bộ; có ít nhất một dòng `PASS:` cho mỗi `R1`–`R8`, `R10`,
  `R11`, `R13`–`R17`, `R19`.
- `PASS:` `nghiem-thu.md` đủ `R1`–`R19`, mỗi dòng có kết luận, mọi đường dẫn trong bảng tồn tại.
- `PASS:` `tech-debt.md` có `TD-01` → `TD-20`, `TD-20` là nợ `backup-operator`; không dòng nào còn `mở`.
- `PASS:` `retro.md` đủ năm mục, nêu nợ lab `#1`, `#5`, `#6`, `#7`, `#8` và giai đoạn 18, 19, 22.
- `PASS:` repo sạch theo yêu cầu 6; `README.md` trỏ tới biên bản nghiệm thu.

`tests/smoke.sh` bản cuối, chép nguyên văn. Nó kế thừa cách kiểm của bản Lab 11b và Lab 12 — chờ HPA
đưa `web` lên `minReplicas` trước khi kiểm, đọc tên ConfigMap từ spec của workload, vào Ingress qua IP
của `lab-k8s-master` (đặt `ENTRY_IP` để đổi) — và thêm các dòng `R19` của sprint này:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái CUỐI của dự án Bookstore (Lab 14, sprint 21).
# Tích lũy: kiểm mọi hợp đồng còn hiệu lực ở DU-AN-BOOKSTORE.md mục 3 và 4. Mỗi dòng ghi mã yêu cầu
# R<n> mà nó làm bằng chứng. R9, R12, R18 có bằng chứng ở tài liệu, không kiểm trên cluster đang chạy.
set -u
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
cd "$(dirname "$0")/.."
NS=bookstore; SYS=bookstore-system; STG=bookstore-staging
SEL='app.kubernetes.io/part-of=bookstore'
GRP=bookstore.example.com
HOST=bookstore.lab.local
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
ok()   { if [ "$1" = "$2" ]; then pass "$3"; else fail "$3 (thay '$1')"; fi; }
k()    { kubectl -n "$NS" "$@"; }
jp()   { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
pods() { k get pod -l "$SEL,app.kubernetes.io/component=$1" --field-selector=status.phase=Running \
           -o jsonpath="$2" 2>/dev/null; }

# 0. HPA cần một chu kỳ để đưa web lên minReplicas và một cửa sổ đo để có số liệu (11b).
MIN="$(jp hpa/web '{.spec.minReplicas}')"
for _ in $(seq 1 48); do
  R="$(jp deployment/web '{.spec.replicas}')"; RR="$(jp deployment/web '{.status.readyReplicas}')"
  U="$(jp hpa/web '{.status.currentMetrics[0].resource.current.averageUtilization}')"
  [ -n "$MIN" ] && [ "${R:-0}" -ge "$MIN" ] && [ "${RR:-0}" = "${R:-x}" ] && [ -n "$U" ] && break
  sleep 5
done

# 1. Namespace và Pod Security (R14)
for X in "$NS:restricted" "$STG:restricted" "$SYS:privileged"; do
  N="${X%%:*}"; L="${X##*:}"
  ok "$(kubectl get namespace "$N" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}|{.metadata.labels.pod-security\.kubernetes\.io/enforce}' 2>/dev/null)" \
     "bookstore|$L" "R14 namespace $N mang label du an, enforce=$L"
done
# Đặt lại đúng mức enforce đang có thì policy không đổi và Pod đang chạy không được đánh giá lại;
# đổi enforce-version mới buộc đánh giá (cùng cách với smoke 9b). Lệnh dry-run phải thành công thì
# "không có cảnh báo" mới có nghĩa: Forbidden, timeout hay lỗi chứng chỉ là FAIL, không phải "sạch".
if PSW="$(kubectl label --dry-run=server --overwrite namespace "$NS" pod-security.kubernetes.io/enforce-version=latest 2>&1)"; then
  if echo "$PSW" | grep -q 'violate'; then fail "R14 co Pod vi pham restricted: $PSW"; else pass 'R14 moi Pod trong bookstore dat restricted'; fi
else
  fail "R14 khong hoi duoc Pod Security bang dry-run: $PSW"
fi
# Đối chứng: cùng đường admission phải từ chối một Pod đặc quyền ở hai namespace restricted.
for N in "$NS" "$STG"; do
  printf '%s\n' 'apiVersion: v1' 'kind: Pod' 'metadata: {name: smoke-dac-quyen}' 'spec:' \
    '  containers: [{name: c, image: busybox:1.37, securityContext: {privileged: true}}]' \
    | kubectl -n "$N" create --dry-run=server -f - 2>&1 | grep -q 'violates PodSecurity "restricted' \
    && pass "R14 doi chung: $N tu choi Pod dac quyen" || fail "R14 doi chung: $N khong tu choi Pod dac quyen"
done

# 2. Object cấp cluster và node (R8, R10, R13, R17, R19)
PRIO="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
[ -n "$PRIO" ] && pass "R10 PriorityClass bookstore-critical ($PRIO)" || fail 'R10 thieu PriorityClass bookstore-critical'
[ -n "$(kubectl get priorityclass bookstore-batch -o jsonpath='{.value}' 2>/dev/null)" ] \
  && pass 'R10 PriorityClass bookstore-batch' || fail 'R10 thieu PriorityClass bookstore-batch'
ok "$(kubectl get storageclass bookstore-retain -o jsonpath='{.reclaimPolicy}' 2>/dev/null)" 'Retain' \
   'R8 StorageClass bookstore-retain giu PV khi xoa PVC'
ok "$(kubectl get nodes -l bookstore.example.com/data=true -o jsonpath='{.items[*].metadata.name}')" 'lab-k8s-worker2' \
   'R10 chi lab-k8s-worker2 mang label du lieu'
UNS="$(kubectl get nodes -o jsonpath='{range .items[*]}{.spec.unschedulable}{"\n"}{end}' | grep -c '^true$')"
[ "$UNS" -eq 0 ] && pass 'R17 khong node nao con bi cordon' || fail "R17 con $UNS node bi cordon"
ok "$(kubectl get crd "dbbackups.$GRP" -o jsonpath='{.status.conditions[?(@.type=="Established")].status}|{.metadata.labels.app\.kubernetes\.io/part-of}|{.status.storedVersions}' 2>/dev/null)" \
   'True|bookstore|["v1"]' 'R19 CRD DbBackup Established, mang label, storedVersions ["v1"]'
ok "$(kubectl get clusterrole bookstore-dbbackup-viewer -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" \
   'bookstore' 'R13 ClusterRole bookstore-dbbackup-viewer mang label du an'

# 3. Cấu hình và bí mật (R2, R7). ConfigMap có hậu tố hash từ 5a: đọc tên từ spec của workload.
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'R2 Secret db-credentials du hai key' || fail 'R2 Secret db-credentials thieu key'
ok "$(jp secret/bookstore-tls '{.type}')" 'kubernetes.io/tls' 'R7 Secret bookstore-tls kieu TLS'
WL="$(k get deployment,statefulset -o json 2>/dev/null)"
for B in web-config api-config db-init; do
  CM="$(echo "$WL" | grep -o "\"$B\(-[a-z0-9]*\)\?\"" | head -n 1 | tr -d '"')"
  [ -n "$CM" ] && k get configmap "$CM" >/dev/null 2>&1 && pass "R2 ConfigMap $CM duoc workload dung" \
    || fail "R2 thieu ConfigMap $B duoc workload tham chieu"
done
ok "$(jp deployment/api '{.spec.template.spec.containers[?(@.name=="postgrest")].env[?(@.name=="DB_PASSWORD")].valueFrom.secretKeyRef.name}')" \
   'db-credentials' 'R2 mat khau cua api lay tu Secret'

# 4. db: định danh, dữ liệu bền, non-root (R3, R5, R8, R10, R14)
ok "$(jp statefulset/db '{.status.readyReplicas}')" '1' 'R5 StatefulSet db 1/1 Ready'
ok "$(jp pod/db-0 '{.metadata.ownerReferences[0].kind}/{.spec.nodeName}/{.status.qosClass}')" \
   'StatefulSet/lab-k8s-worker2/Guaranteed' 'R10 db-0 thuoc StatefulSet, tren lab-k8s-worker2, R3 QoS Guaranteed'
jp statefulset/db '{.spec.template.spec.affinity}' | grep -q 'bookstore.example.com/data' \
  && pass 'R10 db chon node bang affinity theo label du an' || fail 'R10 db khong dung affinity theo label du an'
[ -n "$PRIO" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$PRIO" ] && pass 'R10 db-0 dung bookstore-critical' \
  || fail 'R10 db-0 sai priority'
ok "$(k exec db-0 -c postgres -- id -u 2>/dev/null)" '999' 'R14 postgres chay uid 999'
ok "$(jp pvc/data-db-0 '{.status.phase}|{.spec.storageClassName}')" 'Bound|bookstore-retain' \
   'R8 PVC data-db-0 Bound tren bookstore-retain'
ok "$(jp pvc/backup-store '{.status.phase}')" 'Bound' 'R8 PVC backup-store Bound'
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "R5 db co $N cuon sach" || fail "R5 db co ${N:-0} cuon sach, can >= 8"

# 5. Controller, probe, tài nguyên, phát hành, co giãn, chịu bảo trì (R1, R3, R4, R10, R16, R17)
ok "$(jp deployment/api '{.status.readyReplicas}')" '2' 'R1 Deployment api 2/2 Ready'
C='{.spec.template.spec.containers[?(@.name=="postgrest")]'
ok "$(jp deployment/api "$C.readinessProbe.httpGet.path}|$C.livenessProbe.httpGet.path}")" '/ready|/live' \
   'R1 api: readiness /ready, liveness /live'
[ -n "$(jp deployment/api "$C.startupProbe}")" ] && pass 'R1 api co startup probe' || fail 'R1 api thieu startup probe'
[ -n "$(jp deployment/web '{.spec.template.spec.containers[?(@.name=="nginx")].readinessProbe}')" ] \
  && pass 'R1 nginx co readiness probe' || fail 'R1 nginx thieu readiness probe'
ok "$(jp deployment/web '{.spec.strategy.type}')" 'RollingUpdate' 'R4 web phat hanh RollingUpdate'
ok "$(jp hpa/web '{.spec.scaleTargetRef.kind}/{.spec.scaleTargetRef.name} {.spec.minReplicas}-{.spec.maxReplicas}')" \
   'Deployment/web 2-5' 'R16 HPA web dieu khien Deployment web trong khoang 2-5'
ok "$(jp hpa/web '{.status.conditions[?(@.type=="ScalingActive")].status}')" 'True' 'R16 HPA doc duoc metric (ScalingActive=True)'
R="$(jp deployment/web '{.spec.replicas}')"; RR="$(jp deployment/web '{.status.readyReplicas}')"
{ [ "${R:-0}" -ge 2 ] && [ "${RR:-0}" = "$R" ]; } && pass "R16 web $RR/$R ban sao Ready trong khoang HPA" \
  || fail "R16 web ${RR:-0}/${R:-0} ban sao Ready"
for D in web api; do
  NN="$(pods "$D" '{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | grep -c .)"
  if [ "$D" = web ]; then   # web: topology spread DoNotSchedule (7a) — ràng buộc cứng
    [ "$NN" -ge 2 ] && pass "R10 $D trai tren $NN node" || fail "R10 $D chi tren $NN node"
  else                      # api: anti-affinity mềm (7a) — mong muốn, không bảo đảm
    echo "INFO: R10 $D tren $NN node (anti-affinity mem)"
  fi
  k get pdb "$D" >/dev/null 2>&1 && pass "R3 PodDisruptionBudget $D" || fail "R3 thieu PodDisruptionBudget $D"
  [ "$(jp "pdb/$D" '{.status.disruptionsAllowed}')" -ge 1 ] 2>/dev/null \
    && pass "R17 PDB $D cho phep mot gian doan — drain mot worker khong ket" || fail "R17 PDB $D khong cho gian doan nao"
done
MISS="$(k get pods -l "$SEL" --field-selector=status.phase=Running \
  -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' \
  2>/dev/null | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'R3 moi container dang chay co requests va limits' || fail "R3 thieu tai nguyen: $(echo $MISS)"
ORPHAN="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' \
  2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'R1 moi Pod cua du an co controller' || fail "R1 co $ORPHAN Pod tran"
[ "$(sl k get job,networkpolicy,pod -l app.kubernetes.io/component=loadtest --no-headers)" -eq 0 ] \
  && pass 'R16 khong con object kiem thu tai' || fail 'R16 con object loadtest tren cluster'

# 6. Tác vụ nền (R5, R8, R10, R13, R15)
if k get job db-migrate >/dev/null 2>&1; then
  ok "$(jp job/db-migrate '{.status.conditions[?(@.type=="Complete")].status}')" 'True' 'R5 Job db-migrate Complete'
else pass 'R5 Job db-migrate da duoc TTL don, ket qua nam trong db'; fi
ok "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}|{.spec.jobTemplate.spec.template.spec.priorityClassName}')" \
   'Forbid|bookstore-batch' 'R5 db-backup Forbid, R10 chay o bookstore-batch'
jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.volumes[*].persistentVolumeClaim.claimName}' | grep -qw backup-store \
  && pass 'R8 db-backup ghi vao PVC backup-store' || fail 'R8 db-backup khong dung PVC backup-store'
J="smoke-backup-$(date +%s)"
if k create job "$J" --from=cronjob/db-backup >/dev/null 2>&1 \
   && k wait --for=condition=Complete "job/$J" --timeout=240s >/dev/null 2>&1 \
   && k logs "job/$J" --all-containers=true 2>/dev/null | grep -q '^db-backup: OK'; then
  pass 'R5 chay thu db-backup thanh cong'
else fail 'R5 chay thu db-backup that bai'; fi
k delete job "$J" --ignore-not-found --wait=true >/dev/null 2>&1
# Hợp đồng ghi file backup (6a): đọc thẳng PVC, không tin log — tối đa 10 bản mỗi loại,
# không còn file tạm cũ hơn một giờ.
RD="smoke-doc-backup-$(date +%s)"
k apply -f - >/dev/null 2>&1 <<POD
apiVersion: v1
kind: Pod
metadata:
  name: $RD
spec:
  restartPolicy: Never
  securityContext:
    runAsNonRoot: true
    runAsUser: 65534
    seccompProfile: {type: RuntimeDefault}
  containers:
  - name: doc
    image: busybox:1.37
    command: ["sh", "-c", "ls -1 /backup; echo ---; find /backup -maxdepth 1 -name '*.tmp' -mmin +60"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities: {drop: [ALL]}
    resources:
      requests: {cpu: 10m, memory: 16Mi}
      limits: {cpu: 50m, memory: 32Mi}
    volumeMounts:
    - {name: b, mountPath: /backup, readOnly: true}
  volumes:
  - name: b
    persistentVolumeClaim: {claimName: backup-store, readOnly: true}
POD
if k wait --for=jsonpath='{.status.phase}'=Succeeded "pod/$RD" --timeout=120s >/dev/null 2>&1; then
  OUT="$(k logs "$RD" 2>/dev/null)"
  NB="$(echo "$OUT" | sed '/^---$/,$d' | grep -c '^bookstore-.*\.sql$')"
  NO="$(echo "$OUT" | sed '/^---$/,$d' | grep -Ec '^dbbackup-.*\.(sql|dump)$')"
  NT="$(echo "$OUT" | sed '1,/^---$/d' | grep -c .)"
  { [ "$NB" -le 10 ] && [ "$NO" -le 10 ] && [ "$NT" -eq 0 ]; } \
    && pass "R8 backup-store: $NB ban db-backup, $NO ban operator, khong file tam cu" \
    || fail "R8 backup-store sai hop dong 6a ($NB ban db-backup, $NO ban operator, $NT file tam cu)"
else
  fail 'R8 khong doc duoc noi dung backup-store'
fi
k delete pod "$RD" --ignore-not-found --wait=false >/dev/null 2>&1
k get configmap backup-status >/dev/null 2>&1 && pass 'R13 db-backup bao cao qua ConfigMap backup-status' \
  || fail 'R13 thieu ConfigMap backup-status'
NODES="$(kubectl get nodes --no-headers | wc -l)"
AR="$(kubectl -n "$SYS" get daemonset node-agent -o jsonpath='{.status.numberReady}' 2>/dev/null)"
[ "${AR:-0}" -eq "$NODES" ] && pass "R15 node-agent Ready tren $AR/$NODES node" || fail "R15 node-agent ${AR:-0}/$NODES node"

# 7. Tìm nhau bằng tên, zero-trust, HTTPS (R1, R6, R7)
ok "$(jp service/db '{.spec.clusterIP}')" 'None' 'R6 Service db headless'
ok "$(jp service/api '{.spec.ports[0].port}')" '3000' 'R6 Service api cong 3000'
ok "$(jp service/web '{.spec.type}|{.spec.ports[0].port}')" 'ClusterIP|80' 'R7 Service web ClusterIP cong 80'
WEB="$(pods web '{.items[0].metadata.name}')"
get() { k exec "$WEB" -c status-writer -- timeout 10 wget -q -T 5 -O- "$1" 2>/dev/null; }
T="$(get http://api:3000/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "R6 web goi api bang ten: $T cuon" || fail "R6 web goi api bang ten: $T cuon"
T="$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass 'R6 nginx chuyen /api/ sang api' || fail 'R6 nginx khong chuyen /api/'
get http://127.0.0.1:8080/status.json | grep -Eq "\"pod\" *: *\"$WEB\"" \
  && pass 'R1 status.json dung ten Pod' || fail 'R1 status.json sai'
WN="$(jp "pod/$WEB" '{.spec.nodeName}')"
{ [ "$(kubectl -n "$SYS" get service node-agent -o jsonpath='{.spec.internalTrafficPolicy}' 2>/dev/null)" = 'Local' ] \
  && get "http://node-agent.$SYS.svc.cluster.local:9100/node.json" | grep -Eq "\"node\" *: *\"$WN\""; } \
  && pass 'R6 web doc node-agent cung node qua Service Local' || fail 'R6 node-agent qua Service Local loi'
NPOL="$(k get networkpolicy -o jsonpath='{range .items[*]}{.spec.podSelector}|{.spec.policyTypes}{"\n"}{end}' 2>/dev/null | grep '^{}|')"
{ echo "$NPOL" | grep -q 'Ingress' && echo "$NPOL" | grep -q 'Egress'; } \
  && pass 'R7 bookstore co default deny hai chieu' || fail 'R7 thieu default deny hai chieu'
# Pod ngoài ma trận phải có egress mở thì phép thử mới chạm tới ingress của api: node-agent không
# dùng được vì bookstore-system chặn mọi egress, kể cả DNS (11a). Dùng namespace thử bookstore-thu
# (không NetworkPolicy); chỉ xóa nó nếu smoke tự tạo. DNS-OK là đối chứng: Pod có mạng.
PNS=bookstore-thu; MADE=0
if ! kubectl get namespace "$PNS" >/dev/null 2>&1; then
  kubectl create namespace "$PNS" >/dev/null 2>&1 \
    && kubectl label namespace "$PNS" app.kubernetes.io/part-of=bookstore >/dev/null 2>&1 && MADE=1
fi
OUT="$(kubectl -n "$PNS" run "smoke-r7-$(date +%s)" --rm -i --restart=Never --image=busybox:1.37 \
         --command -- sh -c \
         "nslookup api.$NS.svc.cluster.local >/dev/null 2>&1 && echo DNS-OK; timeout 10 wget -q -T 3 -O /dev/null http://api.$NS:3000/books && echo LOT-QUA || echo BI-CHAN" 2>/dev/null)"
[ "$MADE" -eq 1 ] && kubectl delete namespace "$PNS" --ignore-not-found --wait=true --timeout=120s >/dev/null 2>&1
{ echo "$OUT" | grep -q 'DNS-OK' && echo "$OUT" | grep -q 'BI-CHAN'; } \
  && pass 'R7 Pod ngoai ma tran phan giai duoc api nhung khong goi duoc' \
  || fail "R7 Pod ngoai ma tran: $(echo $OUT)"
ok "$(jp ingress/bookstore '{.spec.rules[0].host}|{.spec.tls[0].secretName}')" "$HOST|bookstore-tls" 'R7 Ingress bookstore co host va TLS'
NP="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}' 2>/dev/null)"
[ -n "$NP" ] || NP="$(kubectl -n traefik get service traefik -o jsonpath='{.spec.ports[?(@.port==443)].nodePort}' 2>/dev/null)"
IP="${ENTRY_IP:-$(kubectl get node lab-k8s-master -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')}"
GET() { curl -sk --max-time 10 --resolve "$HOST:$NP:$IP" "https://$HOST:$NP$1"; }
GET / | grep -q 'Bookstore' && pass 'R7 HTTPS qua Ingress: trang chu' || fail 'R7 HTTPS qua Ingress: trang chu hong'
T="$(GET /api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "R7 HTTPS qua Ingress: /api/books $T cuon" || fail "R7 HTTPS qua Ingress: /api/books $T cuon"
curl -skv --max-time 10 --resolve "$HOST:$NP:$IP" "https://$HOST:$NP/healthz" 2>&1 | grep -qi "subject:.*$HOST" \
  && pass "R7 certificate cap cho $HOST" || fail 'R7 certificate sai ten'

# 8. Quyền tối thiểu (R13), giới hạn quyền debug (TD-07)
SAN() { echo "system:serviceaccount:$NS:$1"; }
can() { kubectl auth can-i "$@" -n "$NS" 2>/dev/null; }
want() { local w="$1" m="$2"; shift 2; ok "$(can "$@")" "$w" "R13 $m"; }
{ [ "$(can patch deployments.apps --as="$(SAN ci-deployer)")" = 'yes' ] \
  || [ "$(can update deployments.apps --as="$(SAN ci-deployer)")" = 'yes' ]; } \
  && pass 'R13 ci-deployer cap nhat duoc Deployment' || fail 'R13 ci-deployer khong cap nhat duoc Deployment'
want no  'ci-deployer khong doc Secret'                   get secrets --as="$(SAN ci-deployer)"
want yes 'bookstore-devs xem duoc Pod'                    list pods --as=smoke-dev --as-group=bookstore-devs
want no  'bookstore-devs khong doc Secret'                get secrets --as=smoke-dev --as-group=bookstore-devs
want no  'bookstore-devs khong exec vao Pod (TD-07)'      create pods --subresource=exec --as=smoke-dev --as-group=bookstore-devs
want no  'bookstore-devs khong gan debug container (TD-07)' patch pods --subresource=ephemeralcontainers --as=smoke-dev --as-group=bookstore-devs
want yes 'bookstore-devs xem duoc DbBackup'               list "dbbackups.$GRP" --as=smoke-dev --as-group=bookstore-devs
want no  'bookstore-devs khong tao DbBackup'              create "dbbackups.$GRP" --as=smoke-dev --as-group=bookstore-devs
want yes 'db-backup sua duoc ConfigMap backup-status'     patch configmaps/backup-status --as="$(SAN db-backup)"
want no  'db-backup khong doc Secret'                     get secrets --as="$(SAN db-backup)"
want yes 'backup-operator ghi duoc dbbackups/status'      patch "dbbackups.$GRP" --subresource=status --as="$(SAN backup-operator)"
want no  'backup-operator khong sua spec DbBackup'        patch "dbbackups.$GRP" --as="$(SAN backup-operator)"
want yes 'backup-operator tao duoc Job'                   create jobs.batch --as="$(SAN backup-operator)"
want no  'backup-operator khong xoa Job'                  delete jobs.batch --as="$(SAN backup-operator)"
want no  'backup-operator khong doc Secret'               get secrets --as="$(SAN backup-operator)"
TOK="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.labels.app\.kubernetes\.io/component}:{.spec.volumes[*].name}{"\n"}{end}' \
       2>/dev/null | grep -E '^(web|api|db):' | grep -c 'kube-api-access')"
[ "$TOK" -eq 0 ] && pass 'R13 web, api, db khong mount token ServiceAccount' || fail "R13 co $TOK Pod web/api/db mount token"

# 9. Quan sát (R15)
TOPN="$(kubectl top pod -n "$NS" --no-headers 2>/dev/null | wc -l)"
[ "$TOPN" -ge 4 ] && pass "R15 kubectl top pod co metric cho $TOPN Pod" || fail 'R15 kubectl top pod khong co du lieu'
kubectl -n "$NS" port-forward deployment/api 13001:3001 >/dev/null 2>&1 & PF=$!
for _ in $(seq 1 20); do curl -s -o /dev/null http://127.0.0.1:13001/live && break; sleep 0.5; done
curl -s --max-time 5 http://127.0.0.1:13001/metrics | grep -q '^pgrst_' \
  && pass 'R15 api /metrics dang Prometheus' || fail 'R15 api /metrics hong'
kill "$PF" 2>/dev/null; wait "$PF" 2>/dev/null

# 10. Hạn mức và tenant (R11, R14)
for X in resourcequota/bookstore-quota limitrange/bookstore-limits; do
  k get "$X" >/dev/null 2>&1 && pass "R11 co $X" || fail "R11 thieu $X"
done
[ -n "$(jp resourcequota/bookstore-quota '{.status.used}')" ] && pass 'R11 quota dang tinh muc dung' || fail 'R11 quota chua co status.used'
[ -n "$(jp resourcequota/bookstore-quota '{.spec.hard.count/dbbackups\.bookstore\.example\.com}')" ] \
  && pass 'R11 quota dat tran so DbBackup' || fail 'R11 quota chua dat tran DbBackup'
for KND in resourcequota networkpolicy rolebinding; do
  [ "$(kubectl -n "$STG" get "$KND" --no-headers 2>/dev/null | wc -l)" -ge 1 ] \
    && pass "R14 tenant staging co $KND" || fail "R14 tenant staging thieu $KND"
done

# 11. Sao lưu bằng API khai báo của dự án (R10, R19)
./scripts/backup-operator.sh status >/dev/null 2>&1 && pass 'R19 backup-operator dang chay' || fail 'R19 backup-operator khong chay'
ok "$(jp dbbackup/baseline '{.status.phase}')" 'Succeeded' 'R19 DbBackup baseline Succeeded'
B="smoke-$(date +%s)"
kubectl apply -f - >/dev/null 2>&1 <<YAML
apiVersion: $GRP/v1
kind: DbBackup
metadata:
  name: $B
  namespace: $NS
  labels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: backup}
spec: {database: bookstore, note: smoke test}
YAML
P=''
for _ in $(seq 1 80); do
  P="$(jp "dbbackup/$B" '{.status.phase}')"; case "$P" in Succeeded|Failed) break ;; esac; sleep 3
done
SZ="$(jp "dbbackup/$B" '{.status.sizeBytes}')"
{ [ "$P" = 'Succeeded' ] && [ "${SZ:-0}" -gt 0 ]; } && pass "R19 DbBackup theo yeu cau Succeeded, $SZ byte" \
  || fail "R19 DbBackup $B phase='$P'"
JB="$(jp "dbbackup/$B" '{.status.jobName}')"
ok "$(jp "job/$JB" '{.metadata.ownerReferences[0].kind}|{.metadata.labels.app\.kubernetes\.io/component}|{.spec.template.spec.priorityClassName}')" \
   'DbBackup|backup|bookstore-batch' 'R19 Job thuoc DbBackup, component backup, R10 chay o bookstore-batch'
k delete dbbackup "$B" --wait=true >/dev/null 2>&1
for _ in $(seq 1 30); do k get job "$JB" >/dev/null 2>&1 || break; sleep 2; done
k get job "$JB" >/dev/null 2>&1 && fail 'R19 xoa DbBackup khong keo theo Job' || pass 'R19 xoa DbBackup keo theo Job'

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

**Câu hỏi thiết kế** — trả lời trong `docs/retro.md`:

1. Yêu cầu nào bạn ghi `ĐẠT CÓ ĐIỀU KIỆN`? Với mỗi cái: điều kiện là gì, ai phải theo dõi, và dấu hiệu
   nào cho biết điều kiện bị vi phạm?
2. Nợ nào chuyển sang Phần II, tới giai đoạn nào, và vì sao không trả trong Phần I?
3. Trong ba script dùng chung, nếu chỉ được giữ một, giữ cái nào? Vì sao?
4. Sprint nào sinh nhiều nợ nhất? Nhìn lại, có nợ nào lẽ ra tránh được bằng một quyết định khác ở sprint
   đó không?

<details>
<summary>Gợi ý</summary>

- Bảng nghiệm thu là để người khác **kiểm lại** được: mỗi ô bằng chứng nên là một dòng `PASS:` cụ thể
  hoặc một đường dẫn trong backtick, không phải một câu kể.
- Trạng thái của `TD-07`, `TD-10`: đọc lại sổ nợ của file dự án — "giảm", "giới hạn", "chấp nhận có
  điều kiện" là những chữ đã được chọn có chủ đích.
- `R18` phụ thuộc bạn có làm Lab 13 hay không; B7 của lab này đã có ADR thiết bị.
- Viết đường dẫn trong bảng không kèm dấu chấm cuối câu — script so đường dẫn theo đúng ký tự.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
EV=~/lab-evidence/14
cd ~/bookstore
./scripts/teardown.sh > "$EV/b8-teardown.txt" 2>&1 && ./scripts/deploy.sh > "$EV/b8-deploy.txt" 2>&1 \
  && echo 'PASS: dung lai sach tu repo'
./tests/smoke.sh > "$EV/b8-smoke.txt" 2>&1; echo "smoke exit=$?"
echo "so dong PASS: $(grep -c '^PASS:' "$EV/b8-smoke.txt")"
grep -q '^FAIL:' "$EV/b8-smoke.txt" && grep '^FAIL:' "$EV/b8-smoke.txt" || echo 'PASS: smoke ban cuoi toan PASS'
BAD=0
for R in 1 2 3 4 5 6 7 8 10 11 13 14 15 16 17 19; do
  grep -q "^PASS: R$R " "$EV/b8-smoke.txt" || { echo "FAIL: smoke khong co dong PASS nao cho R$R"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: smoke co bang chung cho moi yeu cau kiem duoc tren cluster'

BAD=0
for R in $(seq 1 19); do
  L="$(grep -E "^\| *R$R *\|" docs/nghiem-thu.md | head -n 1)"
  [ -n "$L" ] || { echo "FAIL: nghiem-thu.md thieu R$R"; BAD=1; continue; }
  echo "$L" | grep -qE 'ĐẠT|KHÔNG ÁP DỤNG' || { echo "FAIL: R$R chua co ket luan"; BAD=1; }
  for P in $(echo "$L" | grep -oE '(docs|platform|k8s|scripts|tests)/[A-Za-z0-9._/-]+'); do
    P="${P%.}"; [ -e "$P" ] || { echo "FAIL: R$R tro toi $P khong ton tai"; BAD=1; }
  done
done
[ "$BAD" -eq 0 ] && echo 'PASS: nghiem-thu.md du R1-R19, moi dong co ket luan, moi duong dan ton tai'

grep -E '^\| *TD-[0-9]+ *\|' docs/tech-debt.md > "$EV/b8-td.txt"
BAD=0
for T in $(seq -w 1 20); do grep -q "TD-$T" "$EV/b8-td.txt" || { echo "FAIL: thieu TD-$T"; BAD=1; }; done
grep -q 'backup-operator' "$EV/b8-td.txt" || { echo 'FAIL: thieu no operator ngoai cluster'; BAD=1; }
while IFS= read -r L; do
  echo "$L" | grep -qE '\| *(đã trả|chấp nhận có điều kiện|chuyển Phần II) *\|' || { echo "FAIL: chua dong: $L"; BAD=1; }
done < "$EV/b8-td.txt"
[ "$BAD" -eq 0 ] && echo "PASS: $(wc -l < "$EV/b8-td.txt") no ky thuat deu co trang thai cuoi"

BAD=0
for H in 'Đã làm tốt' 'Chưa tốt' 'Bài học' 'Nợ còn lại' 'Bước tiếp theo'; do
  grep -q "^## $H" docs/retro.md || { echo "FAIL: retro thieu muc $H"; BAD=1; }
done
for N in 1 5 6 7 8; do grep -q "#$N" docs/retro.md || { echo "FAIL: retro khong neu no lab #$N"; BAD=1; }; done
for G in 18 19 22; do grep -q "giai đoạn $G" docs/retro.md || { echo "FAIL: retro khong neu giai doan $G"; BAD=1; }; done
[ "$BAD" -eq 0 ] && echo 'PASS: retro du nam muc, neu no lab con mo va noi tra'

[ "$(git ls-files secrets)" = 'secrets/.gitkeep' ] && echo 'PASS: R2 secrets/ khong co gi trong git ngoai .gitkeep'
[ -z "$(git ls-files 'k8s/*.tpl')" ] && echo 'PASS: TD-03 khong con file .tpl'
git grep -qE 'lab-3a-mat-khau|lab-3a-authenticator|doi-mat-khau-nay' -- k8s scripts \
  && echo 'FAIL: TD-02 con mat khau trong repo' || echo 'PASS: TD-02 khong con mat khau trong manifest va script'
kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/overlays/ha-lab >/dev/null 2>&1 \
  && echo 'PASS: R12 overlay ha-lab van render duoc'
{ test -s platform/README.md && ls platform/kubeadm/* >/dev/null 2>&1; } && echo 'PASS: R12 platform/ con ghi chep dung cluster'
grep -q 'docs/nghiem-thu.md' README.md && echo 'PASS: README tro toi bien ban nghiem thu'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Khung `docs/nghiem-thu.md` — điền đường dẫn **thật** trong repo của bạn vào chỗ `<...>`:

```markdown
# Biên bản nghiệm thu — dự án Bookstore

Tag `lab-14`; smoke: `~/lab-evidence/14/b8-smoke.txt` (cluster mốc `04-metrics-ready`).

| Mã | Yêu cầu | Bằng chứng | Kết quả |
| --- | --- | --- | --- |
| R1 | Health check đúng vai trò; container hỏng tự được thay | smoke `PASS: R1 ...`; `docs/postmortem/3a-db-bien-mat.md` | ĐẠT |
| R2 | Cấu hình, bí mật tách khỏi manifest và image | smoke `PASS: R2 ...`; `git ls-files secrets` chỉ `.gitkeep` | ĐẠT |
| R3 | Tài nguyên khai báo; chịu gián đoạn tự nguyện | smoke `PASS: R3 ...` (requests/limits, QoS, PDB) | ĐẠT |
| R4 | Phát hành không gián đoạn, rollback được | smoke `PASS: R4 ...` (chiến lược, PDB); `<ADR cong-vao-web của 5a>` — rollback rồi phát hành lại `web` dưới phép đo liên tục, 0 lỗi; `docs/postmortem/12-bao-tri-worker1.md`, mục *Đã sửa* — thay toàn bộ Pod `api` dưới probe, 0 lỗi trên `/api/books`; `<postmortem rollout hỏng của 4a>` | ĐẠT — số đo ở mức request của 5a B3 và Lab 12 B6, không chỉ trạng thái cuối Ready |
| R5 | DB định danh ổn định; migration, backup, agent là workload riêng | smoke `PASS: R5 ...` | ĐẠT |
| R6 | Các tầng tìm nhau bằng tên | smoke `PASS: R6 ...` | ĐẠT |
| R7 | Zero-trust giữa các tầng; HTTPS từ ngoài | smoke `PASS: R7 ...`; `<ADR https-ingress của 5b>` | ĐẠT CÓ ĐIỀU KIỆN — certificate tự ký, client không tin; Traefik → web là plaintext |
| R8 | Dữ liệu và backup sống sót qua xóa Pod, xóa workload | smoke `PASS: R8 ...`; `<postmortem 6a>` | ĐẠT |
| R9 | Quy trình khôi phục đã diễn tập | `<postmortem 6b>`; `<postmortem 8a>` | ĐẠT |
| R10 | Trải đều; tác vụ nền không chiếm chỗ dịch vụ chính | smoke `PASS: R10 ...` | ĐẠT |
| R11 | Hạn mức namespace | smoke `PASS: R11 ...` | ĐẠT |
| R12 | Dựng lại toàn bộ từ repo; đánh giá HA | `platform/README.md`; `k8s/overlays/ha-lab`; `<ADR topology HA của 8c>` | ĐẠT |
| R13 | Quyền tối thiểu cho thành phần, CI, đội phát triển | smoke `PASS: R13 ...`; `<ADR rbac-dbbackup>` | ĐẠT CÓ ĐIỀU KIỆN — ci-deployer và backup-operator tạo được workload nên gián tiếp chạm Secret (bài 120, *Tạo workload*) |
| R14 | Pod Security `restricted`; nhiều tenant | smoke `PASS: R14 ...` | ĐẠT |
| R15 | Metric, log theo node, quy trình chẩn đoán | smoke `PASS: R15 ...`; `<runbook, postmortem của 11a>` | ĐẠT CÓ ĐIỀU KIỆN — chưa có nơi lưu metric, log dài hạn |
| R16 | `web` tự co giãn | smoke `PASS: R16 ...`; `<evidence, ADR của 11b>` | ĐẠT |
| R17 | Bảo trì node có kế hoạch: worker1 không rơi request ở tầng ứng dụng trên đường đo; worker2 trong cửa sổ, downtime đo được, dữ liệu giữ nguyên | smoke `PASS: R17 ...`; `docs/runbook-bao-tri-node.md` (đoạn đánh giá `R17`); `docs/postmortem/12-bao-tri-worker1.md` | ĐẠT — đúng hai vế đã định, số đo ở Lab 12. Ngoài `R17` và còn là nợ: đường vào chưa HA — Traefik một bản sao, file hosts (`TD-18`); bảo trì worker2 không downtime (`TD-19`) |
| R18 | Đánh giá thiết bị chuyên dụng | `<ADR thiết bị của B7>`; `<ADR dra-bookstore nếu làm Lab 13>` | ĐẠT |
| R19 | Backup bằng API khai báo của dự án | smoke `PASS: R19 ...`; `docs/runbook-backup-operator.md` | ĐẠT CÓ ĐIỀU KIỆN — operator chạy ngoài cluster |
```

Trạng thái cuối của sổ nợ, mã theo [mục 8 của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án):

| Nợ | Trạng thái cuối | Lý do, bằng chứng |
| --- | --- | --- |
| `TD-01` → `TD-06`, `TD-08`, `TD-09` | `đã trả` | dòng `PASS:` tương ứng của smoke: controller, Secret, tên DNS, PVC, tài nguyên, uid 999, `backup-store`, default deny |
| `TD-07` — debug, `exec` đọc được mật khẩu | `chấp nhận có điều kiện` | mật khẩu của `api` vẫn nằm trong biến môi trường; `bookstore-devs` không có `pods/exec`, `pods/ephemeralcontainers` — smoke `R13 ... (TD-07)`; chỉ nhóm trực có quyền debug |
| `TD-10` — bản backup ở cùng cluster với DB | `chấp nhận có điều kiện` | điều kiện: chép bản dump ra máy host theo `docs/runbook-backup-operator.md` sau mỗi đợt quan trọng; khôi phục thảm họa của cả cluster là [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa) |
| `TD-11` — certificate tự ký (5b) | `chấp nhận có điều kiện` | công cụ cấp certificate nằm ngoài chuỗi lab; điều kiện: theo dõi hạn 365 ngày, `deploy.sh` sinh lại khi thiếu |
| `TD-12` — `web`, `api` không tự trải lại sau bảo trì (7a) | `đã trả` | quy trình bảo trì của Lab 12 có bước cân bằng lại; `docs/runbook-bao-tri-node.md` |
| `TD-13` — kubelet chưa giới hạn PID (7b) | `chuyển Phần II` | [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) — cấu hình kubelet |
| `TD-14` — cluster dựng lại bằng tay, không backup etcd (8a) | `chuyển Phần II` | [giai đoạn 19](../00-ALO-TRINH-ADMIN.md#giai-đoạn-19--etcd-backup-và-khôi-phục-thảm-họa) — nợ lab #8 |
| `TD-15` — `ci-deployer` sửa được pod template (9a) | `chấp nhận có điều kiện` | token CI ngắn hạn, thu hồi bằng xóa ServiceAccount. Giai đoạn 14 chỉ gọi tên điểm mở rộng phù hợp — admission webhook, điểm số 2 — chưa dạy viết nó |
| `TD-16` — metrics-server bỏ kiểm certificate của kubelet (11a) | `chuyển Phần II` | [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy) — `serverTLSBootstrap` và duyệt CSR |
| `TD-17` — chưa có nơi lưu metric, log dài hạn (11a) | `chuyển Phần II` | [giai đoạn 23](../00-ALO-TRINH-ADMIN.md#giai-đoạn-23--giám-sát-và-cảnh-báo) |
| `TD-18` — đường vào chưa HA (12) | `chấp nhận có điều kiện` | hosts trỏ một IP node, Traefik một bản sao; điều kiện: bảo trì node đó theo runbook, đổi IP trong hosts |
| `TD-19` — `db` một instance trên PV local (12) | `chấp nhận có điều kiện` | cửa sổ bảo trì theo `docs/runbook-bao-tri-node.md` |
| `TD-20` — *operator ngoài cluster* (14) | `chuyển Phần II` | cần đóng gói controller thành image và chạy như Deployment — [giai đoạn 28](../00-ALO-TRINH-ADMIN.md#giai-đoạn-28--mở-rộng-kubernetes); dự án không có toolchain, không build image |

Khung `docs/retro.md`:

```markdown
# Retro — dự án Bookstore (sprint 1 → 21)

## Đã làm tốt
- Dựng lại toàn bộ hệ thống chỉ từ repo ở đầu mọi sprint; ...

## Chưa tốt
- ...

## Bài học
- ...

## Nợ còn lại
- Nợ kỹ thuật chấp nhận có điều kiện: TD-07, TD-10, TD-11, TD-15, TD-18, TD-19 — điều kiện ghi ở
  docs/tech-debt.md.
- Nợ kỹ thuật chuyển Phần II: TD-13 PID và TD-16 metrics-server (giai đoạn 20), TD-14 etcd (giai
  đoạn 19), TD-17 metric và log dài hạn (giai đoạn 23), TD-20 operator ngoài cluster (giai đoạn 28).
- Nợ lab #1 (phần VPA): chưa có lab trả — cần chốt version add-on VPA ở bảng A1.4 rồi làm lại phần
  VPA của Lab 11b.
- Nợ lab #5 (snapshot volume): chạy lại Lab 6b nhánh A khi cluster có CSI driver hỗ trợ snapshot.
- Nợ lab #6 (mã hóa Secret at rest): giai đoạn 22.
- Nợ lab #7 (vòng đời certificate): giai đoạn 18.
- Nợ lab #8 (backup, restore etcd): giai đoạn 19.

## Bước tiếp theo
- Phần II của lộ trình, theo thứ tự giai đoạn 16 → 30.
```

Ý chính cho bốn câu hỏi:

1. Điển hình: `R7` (certificate tự ký — dấu hiệu: ai đó tắt kiểm tra certificate trên client để "cho
   chạy"), `R13` (quyền tạo workload của CI và operator — dấu hiệu: Job, Pod lạ mang
   `managed-by=backup-operator`), `R15` (log mất khi xoay vòng), `R19` (operator ngoài cluster — dấu
   hiệu: smoke báo `R19 backup-operator khong chay`, `DbBackup` có `PHASE` rỗng lâu). `R17` không nằm
   trong danh sách: cửa sổ DB và đường đo đã là một phần của chính yêu cầu, nên kết luận là `ĐẠT`; phần
   vượt ra ngoài — đường vào chưa HA, bảo trì worker2 không downtime — là nợ `TD-18`, `TD-19`, theo
   dõi bằng số đo của Lab 12.
2. *Operator ngoài cluster* sang giai đoạn 28 vì cần đóng gói controller thành image và chạy như
   Deployment — thứ dự án không có toolchain. metrics-server sang giai đoạn 20, metric/log dài hạn sang
   giai đoạn 23 — cả hai cần sửa cấu hình cluster hoặc thêm hạ tầng. Nợ lab `#6`, `#7`, `#8` vốn đã xếp
   ở giai đoạn 22, 18, 19.
3. `deploy.sh` — nó là bản mô tả **thực thi được** của cả hệ thống: dựng lại được từ repo thì có thể
   viết lại hai script kia; mất nó thì repo chỉ còn là một đống manifest không thứ tự.
4. Sprint 1 (Pod trần) sinh nhiều nợ nhất, và đó là chủ đích của lộ trình: mỗi nợ là một cơ chế chưa
   học. Nợ tránh được thật sự thường là nợ về **quy trình** (bí mật sinh tại chỗ, bundle ra ngoài VM),
   không phải nợ về cơ chế.

</details>

## B9. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao repo ở tag cuối của dự án, rồi trả cluster về đúng `04-metrics-ready` với tồn
kho object cấp cluster bằng đúng con số ở B0.

1. `tests/smoke.sh` của Lab 14 là bản đã chép ở B8 — bản cuối của dự án. `docs/tech-debt.md` đã đóng ở
   B8. Không sửa gì thêm ở hai file này.

2. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/14/b9-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/14/b9-smoke.txt 2>&1; echo "smoke exit=$?"
grep -c '^PASS:' ~/lab-evidence/14/b9-smoke.txt; grep '^FAIL:' ~/lab-evidence/14/b9-smoke.txt
```

3. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-14: sprint 21 — CRD DbBackup, backup-operator, nghiem thu cuoi du an'
git tag lab-14
git bundle create ~/bookstore-lab-14.bundle --all && git bundle verify ~/bookstore-lab-14.bundle
```

Chép `~/bookstore-lab-14.bundle` về máy host bằng `scp`. Đây là bundle **cuối** của dự án — giữ lâu
dài; bundle của các sprint trước có thể xóa.

4. Dọn object thử nghiệm và trả cluster:

```bash
kubectl -n bookstore delete pod,job -l '!app.kubernetes.io/part-of' --ignore-not-found --wait=true
kubectl delete namespace bookstore-thu --ignore-not-found --wait=true
rm -rf ~/lab-work/14
cd ~/bookstore && ./scripts/teardown.sh
```

5. Gate cuối:

```bash
# sl <lệnh...>: đếm dòng output; lệnh lỗi thì in "loi" để mọi phép so sánh số đều sai, không ra PASS.
sl() { local o; o="$("$@" 2>/dev/null)" || { echo loi; return 1; }; printf '%s\n' "$o" | awk 'NF' | wc -l; }
EV=~/lab-evidence/14
cd ~/bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-14' && echo 'PASS: HEAD mang tag lab-14'
test -s ~/bookstore-lab-14.bundle && echo 'PASS: co bundle cuoi cua du an'
{ test -f k8s/cluster/crd-dbbackup.yaml && test -f k8s/cluster/clusterrole-dbbackup-viewer.yaml \
  && test -x scripts/backup-operator.sh && test -f k8s/app/backup/dbbackup-baseline.yaml \
  && test -f k8s/app/rbac/backup-operator.yaml; } && echo 'PASS: bo cuc repo dung muc 5.5'
BAD=0
for F in 'adr/*-chon-co-che-mo-rong.md' 'adr/*-crd-dbbackup.md' 'adr/*-rbac-dbbackup.md' \
         'adr/*-backup-operator.md' 'adr/*-dbbackup-v1.md' 'adr/*thiet-bi*' \
         'postmortem/14-controller-vang-mat.md' 'runbook-backup-operator.md' 'nghiem-thu.md' 'retro.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR, postmortem, runbook, bien ban nghiem thu va retro'
{ test -s docs/tech-debt.md && grep -qE '^\| *TD-' docs/tech-debt.md \
  && ! grep -E '^\| *TD-' docs/tech-debt.md | grep -qE '\| *mở *\|'; } \
  && echo 'PASS: so no ky thuat da dong' || echo 'FAIL: con no ky thuat mo, hoac khong doc duoc docs/tech-debt.md'

{ [ "$(kubectl get namespace,priorityclass,storageclass,crd,clusterrole,clusterrolebinding \
       -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 ] \
  && [ "$(sl kubectl get all -A -l app.kubernetes.io/part-of=bookstore --no-headers)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an'
{ test -x scripts/backup-operator.sh && ! ./scripts/backup-operator.sh status >/dev/null 2>&1; } \
  && echo 'PASS: backup-operator da dung' || echo 'FAIL: operator con chay, hoac thieu scripts/backup-operator.sh'
ls secrets/*.kubeconfig >/dev/null 2>&1 && echo 'FAIL: con kubeconfig chua token' || echo 'PASS: khong con kubeconfig chua token'
test ! -e ~/lab-work/14 && echo 'PASS: da don lab-work'

for _ in $(seq 1 30); do
  { for R in crd clusterrole clusterrolebinding apiservice; do
      echo "$R=$(kubectl get "$R" --no-headers | wc -l)"; done; } > "$EV/b9-ton-kho.txt"
  diff -q "$EV/b0-ton-kho.txt" "$EV/b9-ton-kho.txt" >/dev/null && break
  sleep 2
done
diff "$EV/b0-ton-kho.txt" "$EV/b9-ton-kho.txt" \
  && echo 'PASS: CRD, ClusterRole, ClusterRoleBinding, APIService ve dung so luong cua B0'
{ echo "lab-k8s-master $(sudo sha256sum /var/lib/kubelet/config.yaml | awk '{print $1}')"
  for N in lab-k8s-worker1 lab-k8s-worker2; do
    echo "$N $(ssh "$N" 'sudo sha256sum /var/lib/kubelet/config.yaml' | awk '{print $1}')"
  done; } > "$EV/b9-kubelet-sha.txt"
diff -q "$EV/b0-kubelet-sha.txt" "$EV/b9-kubelet-sha.txt" >/dev/null && echo 'PASS: cau hinh kubelet khong doi suot lab'
[ -z "$(grep -rl 'eyJ' "$EV" 2>/dev/null)" ] && echo 'PASS: bang chung khong chua chuoi giong JWT'

test "$(sl kubectl top node --no-headers)" -eq 3 && echo 'PASS: kubectl top node du ba node'
test "$(kubectl get apiservice v1beta1.metrics.k8s.io \
  -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')" = 'True' && echo 'PASS: metrics.k8s.io Available'
test "$(kubectl get storageclass \
  -o jsonpath='{range .items[*]}{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}{"\n"}{end}' \
  | grep -c '^true$')" -eq 1 && echo 'PASS: dung mot StorageClass mac dinh'
test "$(kubectl get ingressclass --no-headers 2>/dev/null | wc -l)" -ge 1 && echo 'PASS: IngressClass van con'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** `smoke exit=0` và file smoke ở bước 2 in toàn `PASS:`; mười tám dòng `PASS:` của gate cuối
xuất hiện, không dòng `FAIL:`; ba node `Ready`; lệnh field selector trả `No resources found`; CoreDNS
đủ replica; namespace `default` không có Pod. Thư mục `~/lab-evidence/14/` **giữ lại**. Cluster trở về
`04-metrics-ready`; **không tạo snapshot mới**.

Dự án Bookstore kết thúc ở tag `lab-14`. [Lab 15](LAB-15-NODE-WINDOWS.md) (tùy chọn) không thuộc dự án
và bắt đầu từ chính mốc này; phần tiếp theo của lộ trình là
[Phần II — Vận hành cluster](../00-ALO-TRINH-ADMIN.md#phần-ii--vận-hành-cluster).

---

## 3. Checkpoint 14

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Bạn `kubectl apply` một `DbBackup` khi operator đang dừng và không có gì xảy ra. Bạn **đang có**
      gì và **chưa có** gì? Vì sao `kubectl get`, validation, RBAC, quota vẫn chạy đủ? Khi operator
      chạy lại, vì sao yêu cầu đó không bị bỏ sót?
- [ ] Câu bẫy: `metrics.k8s.io` và `bookstore.example.com` đều "mở rộng Kubernetes API". Đọc trường nào
      của object nào để phân biệt? Ai phục vụ, ai lưu dữ liệu ở mỗi bên? Vì sao `metrics.k8s.io` không
      thể là CRD, và vì sao quota đếm được `DbBackup` mà không đếm được object của nó?
- [ ] Bảy điểm mở rộng của bài 177: Calico, Traefik, local-path-provisioner, metrics-server,
      `backup-operator` nằm ở điểm nào, mô hình gọi nào? Vì sao provisioner không phải storage plugin?
- [ ] Một `DbBackup` có `database: khac`, một cái có trường lạ, một cái bỏ trống `format`. API server làm
      gì với từng cái, ở tầng nào? Vì sao validation ở API server tốt hơn để operator tự kiểm?
- [ ] Trước và sau khi bật subresource `status`: ai ghi được `status`, `generation` tăng khi nào, và
      RBAC tách hai đường ghi đó bằng resource nào?
- [ ] Nhóm `bookstore-devs` có `view` mà không đọc được `DbBackup`. Vì sao? Bạn sửa bằng object nào, ở
      phạm vi nào, và vì sao không dùng ClusterRoleBinding?
- [ ] `backup-operator` không đọc được Secret nhưng tạo được Job. Theo bài 120 nó thực tế chạm tới được
      gì? Bốn lớp chính sách nào của các sprint trước giới hạn Job của nó, và label nào làm NetworkPolicy
      cho Job tới được `db`?
- [ ] Kể vòng lặp của operator cho một `DbBackup` từ lúc tạo tới `Succeeded`: đọc gì, tạo gì, đọc kết
      quả ở đâu, ghi gì. Vì sao Job biến mất giữa chừng thì ghi `Failed` thay vì tạo lại? Vì sao chạy
      lại vòng lặp khi đã khớp không ghi gì?
- [ ] Xóa một `DbBackup`: ai xóa Job, dựa vào trường nào, file dump còn không? Muốn làm một việc
      **trước** khi xóa — như bước 5 của `SampleDB` — cần cơ chế gì?
- [ ] `status.storedVersions` là gì? Vì sao API server từ chối bỏ `v1alpha1` lúc đầu, ba bước nào đưa
      bạn tới chỗ bỏ được, và vì sao dự án vẫn giữ `v1alpha1` được phục vụ?
- [ ] Node của bạn không công bố resource `vendor-domain/resourcetype`. Hai chặng đăng ký nào chưa xảy
      ra? Một device plugin GPU sẽ chạy ở namespace nào của dự án, và Pod gợi ý sách xin GPU bằng gì?
- [ ] Nghiệm thu: yêu cầu nào trong `R1`–`R19` đạt có điều kiện, điều kiện gì? Nợ kỹ thuật nào chuyển
      sang Phần II, tới giai đoạn nào?

Nợ còn mở khi đóng dự án — nói được **vì sao chưa trả** và trả ở đâu:

- **Nợ lab:** `#1` phần VPA (add-on ngoài A1.4, chưa có lab trả), `#5` (snapshot volume — cần CSI
  driver hỗ trợ snapshot, chạy lại Lab 6b nhánh A), `#6` (mã hóa Secret at rest — giai đoạn 22), `#7`
  (vòng đời certificate — giai đoạn 18), `#8` (backup, restore etcd — giai đoạn 19). Lab 14 không phát
  sinh và không trả nợ lab nào.
- **Nợ kỹ thuật:** sau B8 không còn dòng `mở`. Đã trả: `TD-12`. Chấp nhận có điều kiện: `TD-07`,
  `TD-10`, `TD-11`, `TD-15`, `TD-18`, `TD-19`. Chuyển Phần II: `TD-13`, `TD-16` (giai đoạn 20), `TD-14`
  (giai đoạn 19), `TD-17` (giai đoạn 23) và nợ mới của sprint này — `TD-20` *operator ngoài cluster*
  (giai đoạn 28).

### Bài giải thích cuối cùng

Trong vài phút, kể lại bằng lời hai luồng, không nhìn tài liệu:

1. **Một yêu cầu backup đi qua hệ thống.** Bắt đầu từ lúc người trực `kubectl apply` một `DbBackup`:
   API server xác thực, phân quyền (RBAC cho kind mới), admission (quota đếm `DbBackup`), validation và
   defaulting theo schema, rồi lưu ở storage version nào. Operator — chạy ở đâu, dưới danh tính nào, token
   gì — thấy object ra sao, tạo Job thế nào, Job đi qua Pod Security, NetworkPolicy, quota, PriorityClass
   ra sao để tới `db-0.db:5432` và ghi vào `backup-store`. Kết quả quay về `status` bằng đường nào. Cuối
   cùng: xóa yêu cầu thì ai dọn gì, cái gì còn lại. Kết bằng: nếu operator chết ba ngày thì hệ thống hỏng
   ở đâu, **không** hỏng ở đâu.
2. **Dự án từ `docker compose` tới `lab-14`.** Đi qua tám vấn đề `P1`–`P8` của hệ thống cũ; với mỗi vấn
   đề nói sprint nào giải nó bằng cơ chế Kubernetes nào, và bằng chứng nằm ở dòng `PASS:` nào của
   `smoke.sh` hoặc tài liệu nào trong repo. Rồi nói thật ba chỗ hệ thống **vẫn** yếu khi đóng dự án và
   chúng sẽ được xử lý ở đâu trong Phần II.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm *CRD* với *aggregated API*, *custom resource* với
*custom controller*, *`spec.versions`* với *`status.storedVersions`*, hay *garbage collection theo owner
reference* với *việc của controller* — Lab 14,
[giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) và dự án Bookstore hoàn tất.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: `deploy.sh` của sprint trước fail | Repo ở tag cũ thiếu thứ chỉ tồn tại trên cluster cũ | Sửa repo ở tag đó; không sửa tay trên cluster |
| B2: `deploy.sh` dừng ở `kubectl apply` với lỗi không tìm được mapping cho kind `DbBackup` | CRD chưa `Established` khi apply custom resource, hoặc file CRD không nằm trong `k8s/cluster/` | Giữ đúng thứ tự của `deploy.sh`: `k8s/cluster` → chờ `Established` → `k8s/app` |
| B2: CRD có `NamesAccepted=False` | Tên ngắn, category hoặc số nhiều trùng resource khác | `kubectl get crd dbbackups.bookstore.example.com -o jsonpath='{.status.conditions}'`; đổi tên trùng rồi apply lại |
| B2: apply CRD báo lỗi schema, condition `NonStructuralSchema` | Thiếu `type` ở một mức của schema, hoặc `default` đặt ở chỗ không có `type` | Mỗi `properties` con phải có `type`; `default` phải hợp lệ với chính schema của trường đó |
| B2: `kubectl get dbb` báo không có resource type trong khi CRD đã tạo | Cache discovery phía client chưa làm mới | `kubectl get --raw /apis/bookstore.example.com` trả danh sách thì server đã đúng; `rm -rf ~/.kube/cache/discovery` rồi thử lại |
| B2: patch `--subresource=status` báo không tìm thấy resource | `subresources` đặt sai cấp | `subresources: {status: {}}` nằm **trong** một phần tử của `versions`, cùng cấp `schema` |
| B3: `bookstore-devs` vẫn không đọc được `DbBackup` | Sai tên group, RoleBinding sai namespace, `apiGroups` của ClusterRole sai | `kubectl -n bookstore get rolebinding bookstore-devs-dbbackup-viewer -o yaml`; group là `bookstore.example.com`, không có `/v1alpha1` |
| B3: tạo `DbBackup` đầu tiên trên namespace thử bị từ chối vì quota | Quota vừa tạo chưa được controller tính `status` | Chờ `status.hard` có giá trị như script làm |
| B4: log operator `Unauthorized` | Token hết hạn, hoặc ServiceAccount bị xóa rồi tạo lại | `./scripts/deploy.sh` cấp token mới; operator đọc lại file ở lần gọi sau |
| B4: log operator `forbidden ... dbbackups/status` hoặc `pods/log` | Role thiếu subresource | Thêm đúng rule còn thiếu vào `k8s/app/rbac/backup-operator.yaml`, `deploy.sh` |
| B4: Job tạo xong nhưng không có Pod, sự kiện `FailedCreate ... violates PodSecurity "restricted"` | Thiếu `seccompProfile`, `runAsNonRoot`, `allowPrivilegeEscalation: false` hoặc drop capability | Sửa `job_manifest`; đọc `kubectl -n bookstore describe job dbbackup-<tên>` |
| B4: sự kiện `FailedCreate ... exceeded quota` | `count/jobs.batch` đầy, hoặc Pod thiếu `requests`/`limits` | `kubectl -n bookstore describe quota bookstore-quota`; TTL của Job đủ ngắn, trần nâng ở B3 |
| B4: Pod của Job lỗi `pg_dump: error: connection to server ... timeout` | Pod không mang `app.kubernetes.io/component=backup`, NetworkPolicy chặn | Label hợp đồng trên **template** của Pod, không chỉ trên Job |
| B4: Pod của Job lỗi `password authentication failed` | `secrets/db.env` sinh lại trong khi PV `Retain` còn dữ liệu cũ | Xem [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm); teardown sạch rồi deploy |
| B4: Pod của Job lỗi `could not open output file ... Permission denied` | Thư mục của PVC không cho uid 999 ghi | `kubectl exec` Pod `doc-backup` xem `ls -ld /backup`; so với CronJob `db-backup` đang ghi được ở cùng PVC |
| B4: Job `Complete` mà `status` không bao giờ `Succeeded` | Operator không chạy, hoặc dòng log khác hợp đồng `db-backup: OK <file> <bytes> byte` | `./scripts/backup-operator.sh status`; `kubectl -n bookstore logs job/dbbackup-<tên>` |
| B4: `k create -f -` báo Job đã tồn tại | `DbBackup` mới trùng tên một `DbBackup` cũ vừa xóa, Job cũ chưa được dọn | Chờ garbage collector, hoặc dùng tên khác |
| B5: `stop` không dừng được operator | File PID thiếu hoặc lệch | `pgrep -af backup-operator.sh`, `kill` đúng PID, rồi `start` |
| B6: bỏ `v1alpha1` không bị từ chối | Đã chạy bước ghi lại và vá `storedVersions` trước | Thứ tự sai nhưng vô hại; ghi vào bằng chứng rồi đi tiếp |
| B6: `kubectl replace` báo object đã bị sửa | Operator đang ghi `status` của một `DbBackup` đang chạy | Chờ mọi `DbBackup` ở pha cuối rồi chạy lại vòng ghi lại |
| B6: `deploy.sh` báo lỗi khi apply CRD chỉ có `v1` | `storedVersions` vẫn còn `v1alpha1` | Giữ `v1alpha1` trong file CRD cho tới khi di trú xong |
| B8: smoke `R16` fail ngay sau `deploy.sh` | HPA chưa có số đo đầu tiên | Chạy lại smoke sau một chu kỳ đo; vòng chờ của smoke cho tối đa 240 giây |
| B8: smoke `R7` HTTPS fail | Tên cổng Service của Traefik khác, hoặc IP vào không phải node đang chạy | Đặt `ENTRY_IP` thành IP một worker; kiểm `kubectl -n traefik get svc traefik` |
| B9: `teardown.sh` kẹt ở xóa namespace | CRD bị xóa trước namespace | Giữ thứ tự của script: custom resource → namespace → CRD |
| B9: tồn kho APIService lệch B0 | APIService tự đăng ký của API group chưa bị dọn | Chờ thêm; không xóa tay APIService nào ngoài group `bookstore.example.com` |
| Lỡ xóa một CRD không thuộc dự án | Selector sai | Dừng hẳn; restore cả ba VM về `04-metrics-ready` — xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường) |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Extending Kubernetes](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/)
- [Kubernetes v1.35 — Extending the Kubernetes API](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/api-extension/)
- [Kubernetes v1.35 — Custom Resources](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Kubernetes v1.35 — Kubernetes API Aggregation Layer](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)
- [Kubernetes v1.35 — Operator pattern](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Kubernetes v1.35 — Compute, Storage, and Networking Extensions](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/)
- [Kubernetes v1.35 — Device Plugins](https://v1-35.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [Kubernetes v1.35 — Troubleshooting Topology Management](https://v1-35.docs.kubernetes.io/docs/tasks/debug/debug-cluster/topology/)
- [Kubernetes v1.35 — Migrate Kubernetes Objects Using Storage Version Migration](https://v1-35.docs.kubernetes.io/docs/tasks/manage-kubernetes-objects/storage-version-migration/)
- [Kubernetes v1.35 — Extend the Kubernetes API with CustomResourceDefinitions](https://v1-35.docs.kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)
- [Kubernetes v1.35 — Versions in CustomResourceDefinitions](https://v1-35.docs.kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)
- [Kubernetes v1.35 — Controllers](https://v1-35.docs.kubernetes.io/docs/concepts/architecture/controller/)
- [Kubernetes v1.35 — Owners and Dependents](https://v1-35.docs.kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/)
- [Kubernetes v1.35 — Role Based Access Control Good Practices](https://v1-35.docs.kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Kubernetes v1.35 — Resource Quotas](https://v1-35.docs.kubernetes.io/docs/concepts/policy/resource-quotas/)
- [Kubernetes v1.35 — Automatic Cleanup for Finished Jobs](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)
- [Kubernetes v1.35 — Pod Security Standards](https://v1-35.docs.kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes v1.35 — `kubectl create token`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_token/)
- [Kubernetes v1.35 — `kubectl auth can-i`](https://v1-35.docs.kubernetes.io/docs/reference/kubectl/generated/kubectl_auth/kubectl_auth_can-i/)
- [Kubernetes API — CustomResourceDefinition v1](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/extend-resources/custom-resource-definition-v1/)
- [Kubernetes API — APIService v1](https://v1-35.docs.kubernetes.io/docs/reference/kubernetes-api/cluster-resources/api-service-v1/)
- [PostgreSQL 17 — pg_dump](https://www.postgresql.org/docs/17/app-pgdump.html)
- [PostgreSQL 17 — pg_restore](https://www.postgresql.org/docs/17/app-pgrestore.html)
- [PostgREST v14 — Observability, mục Metrics](https://docs.postgrest.org/en/v14/references/observability.html)
