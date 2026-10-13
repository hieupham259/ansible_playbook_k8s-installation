# Lab 7a — Lập lịch và eviction

> **Điểm bắt đầu:** snapshot `03-storage-ready` — mốc do [Lab 6a](LAB-6A-PV-PVC-VA-STORAGECLASS.md)
> tạo, xem [chuỗi snapshot](README.md#3-chuỗi-snapshot). Repo `~/bookstore` ở tag `lab-6b` (hoặc
> bundle `~/bookstore-lab-6b.bundle` nếu bạn vừa restore snapshot).
> **Điểm kết thúc:** teardown trả cluster về đúng `03-storage-ready`, không tạo snapshot mới. Không
> node nào còn label hay taint của dự án. Repo `~/bookstore` có tag `lab-7a` và một bản bundle nằm
> ngoài VM.
> **Sprint dự án:** Sprint 10 — Đặt workload đúng chỗ của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 6b — Snapshot và volume nâng cao](LAB-6B-SNAPSHOT-VA-VOLUME-NANG-CAO.md).
> **Lab sau:** [Lab 7b — Quota và giới hạn tài nguyên](LAB-7B-QUOTA-VA-GIOI-HAN-TAI-NGUYEN.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng mục [7a. Scheduling và eviction](../00-ALO-TRINH-ADMIN.md#7a-scheduling-và-eviction)
của [Giai đoạn 7](../00-ALO-TRINH-ADMIN.md#giai-đoạn-7--lập-lịch-và-chính-sách-tài-nguyên): mười
ba bài khái niệm 136–148 cộng ba bài thực hành 210, 266, 267. Mọi ticket dựa vào hợp đồng kỹ thuật
ở [mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật), bố cục repo ở
[mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint) và ràng buộc lập lịch đích ở
[mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch). Kế hoạch gốc của sprint nằm ở
[mục 7.2](DU-AN-BOOKSTORE.md#sprint-10--lab-7a-đặt-workload-đúng-chỗ).

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Thành phần ngoài
baseline đang chạy — Calico, Traefik, local-path-provisioner — đã khóa ở
[bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) và do Lab 5b, Lab 6a cài;
lab này chỉ **đọc** chúng, không chép lại con số phiên bản nào và **không cài thêm gì**.

Topology vẫn là một control plane `lab-k8s-master` mang taint `NoSchedule` và hai worker
`lab-k8s-worker1`, `lab-k8s-worker2`, mỗi worker 2 vCPU
([A1.2](LAB-00-MOI-TRUONG-1.35.7.md#a12-ba-vm)). **Hai worker là hai miền topology duy nhất nhận
workload của dự án** — đó là dữ kiện quyết định của B3, B5 và B7.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi hệ thống Bookstore đạt trạng thái sau sprint 10 — **mỗi workload nằm đúng chỗ
theo một ràng buộc khai báo trong repo** — **và** bạn chứng minh, giải thích được:

- Chu trình lọc rồi chấm điểm của scheduler: đọc được bước lọc trong thông điệp của một Pod
  `Pending`, biết khi nào bước chấm điểm không chạy, và vì sao `nodeName` bỏ qua cả hai bước.
- `db` đi theo label dữ liệu của dự án bằng node affinity bắt buộc. Label đó và `nodeAffinity` của
  PV là hai ràng buộc khác nhau cùng trỏ về một node. `IgnoredDuringExecution` nghĩa là gỡ label
  không đuổi `db-0` đang chạy, nhưng `db-0` bị thay thì không lên lại được.
- `web` trải đều hai worker bằng `maxSkew`; `api` dùng anti-affinity mềm; vì sao không dùng
  anti-affinity cứng cho cả hai; node thiếu `topologyKey` bị bỏ qua hoàn toàn.
- PriorityClass `bookstore-batch` mang giá trị âm: tác vụ nền xếp hàng sau và là nạn nhân đầu tiên.
  Preemption chỉ lấy vừa đủ chỗ, ghi `nominatedNodeName`, và để lại condition `DisruptionTarget`.
- Báo cáo tồn kho chờ duyệt bằng `schedulingGates`: `SchedulingGated` khác `Unschedulable`; chỉ gỡ
  được cổng, chỉ siết được chỉ thị lập lịch.
- Taint khi bảo trì: `NoSchedule` không đuổi Pod đang chạy, `NoExecute` thì có, `tolerationSeconds`
  quyết định ở lại bao lâu; toleration chỉ cho phép chứ không kéo Pod về. Ràng buộc phân bố vẫn đếm
  node đang bị taint trừ khi đặt `nodeTaintsPolicy: Honor`. Bảo trì node dữ liệu là downtime của DB.
- Hai thứ cùng tên eviction: qua API thì tôn trọng PDB và grace period, trả `429` khi PDB hết ngân
  sách; do áp lực node thì kubelet tự làm theo ngưỡng thật đọc từ cấu hình của nó. Xếp hạng được Pod
  của dự án trên `lab-k8s-worker2` theo đúng ba tham số mà kubelet dùng.
- Những gì chỉ đọc được trên cluster này: scheduler chạy cấu hình mặc định một profile,
  `percentageOfNodesToScore` không có tác dụng, bin packing không bật. Pod overhead được cộng vào
  phép tính lập lịch — thử được bằng một RuntimeClass trỏ vào handler `runc` có sẵn.
- Mọi thay đổi nằm trong repo; `teardown.sh` gỡ cả label và taint mà dự án đặt lên node.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm 7a | Ticket kiểm chứng |
| --- | --- |
| [136 — Lập lịch, Preemption và Eviction](../136-scheduling-eviction-vi.md) | B1 — ba từ khóa ứng với ba hiện vật thật: Pod chưa lập lịch, kind `PriorityClass`, subresource `pods/eviction`; B5, B7, B8 — bốn con đường chấm dứt Pod đang chạy, phân biệt bằng `reason` của condition `DisruptionTarget` |
| [137 — Bộ lập lịch của Kubernetes](../137-kube-scheduler-vi.md) | B1 — binding do `default-scheduler` ghi, thông điệp lọc của Pod `qua-co`, danh sách khả thi rỗng thì Pod chưa được lập lịch; B9 — hai cách cấu hình bước lọc và bước chấm điểm |
| [138 — Gán Pod cho Node](../138-assign-pod-node-vi.md) | B1 — `nodeName` bỏ qua scheduler; B2 — node affinity `required`, quy tắc AND, `IgnoredDuringExecution`; B3 — inter-pod anti-affinity `required` so với `preferred`; B4 — node affinity `preferred`, `weight`, toán tử `DoesNotExist` |
| [267 — Gán Pod vào Node](../267-assign-pods-nodes-vi.md) | B2 — gắn label cho node rồi dùng `nodeSelector` khớp label đó (Pod `chon-nhan`); B1 — `nodeName` |
| [266 — Gán Pod vào Node bằng Node Affinity](../266-assign-pods-nodes-node-affinity-vi.md) | B2 — `required` cho `db`; B4 — `preferred` cho tác vụ nền, chứng minh nó mềm bằng một Pod bị ép sang node không được ưu tiên |
| [139 — Taint và Toleration](../139-taint-and-toleration-vi.md) | B1 — taint control plane trong thông điệp lọc; B7 — `NoSchedule`, `NoExecute`, `tolerationSeconds`, `effect` rỗng, `nodeName` trên node bị taint, Pod DaemonSet bị taint tùy biến đuổi; B8 — toleration `300` giây tự thêm |
| [140 — Ràng buộc phân bố Pod theo topology](../140-topology-spread-constraints-vi.md) | B3 — `maxSkew` theo `kubernetes.io/hostname`, `DoNotSchedule`, 3 và 5 bản sao, `topologyKey` gõ sai; B7 — node bị taint vẫn được đếm (*Các hạn chế đã biết*), sửa bằng `nodeTaintsPolicy: Honor`, phân bố không tự cân bằng lại |
| [141 — Độ ưu tiên và Preemption của Pod](../141-pod-priority-preemption-vi.md) | B4 — PriorityClass `bookstore-batch`, thứ tự giá trị; B5 — preemption trên `lab-k8s-worker2`, `nominatedNodeName`, nạn nhân tối thiểu, Pod batch thay thế nằm chờ; ⭐ PDB chỉ được tôn trọng ở mức nỗ lực tốt nhất |
| [210 — Bảo đảm lập lịch cho Pod add-on quan trọng](../210-guaranteed-scheduling-critical-addon-pods-vi.md) | B4 — CoreDNS, kube-proxy dùng hai PriorityClass hệ thống; `bookstore-critical` luôn thấp hơn chúng |
| [142 — Eviction do áp lực node](../142-node-pressure-eviction-vi.md) | B8 — đọc `evictionHard` thật của kubelet trên `lab-k8s-worker2`, khoảng cách tới ngưỡng, điều kiện node và taint áp lực, xếp hạng Pod theo ba tham số. Phần kích hoạt eviction thật xem bảng dưới |
| [143 — Eviction khởi phát qua API](../143-api-eviction-vi.md) | B8 — tạo `Eviction` cho Pod `web`: được chấp nhận, grace period được tôn trọng, Pod rời EndpointSlice trước khi container chết, `429` khi PDB hết ngân sách |
| [144 — Pod Overhead](../144-pod-overhead-vi.md) | B9 — RuntimeClass thử nghiệm khai `overhead` trên handler `runc`: admission điền `spec.overhead`, node cộng overhead vào request, tự khai `overhead` bị từ chối; ⭐ cgroup cấp Pod; ⭐ Pod `Pending` chỉ vì overhead |
| [145 — Mức sẵn sàng lập lịch của Pod](../145-pod-scheduling-readiness-vi.md) | B6 — Job báo cáo chờ duyệt: `SchedulingGated`, không thêm cổng mới, siết `nodeSelector` được nhưng nới thì không, gỡ cổng thì chạy |
| [146 — Tinh chỉnh hiệu năng bộ lập lịch](../146-scheduler-perf-tuning-vi.md) | B9 — kube-scheduler không nạp `--config`, cluster ba node nằm dưới ngưỡng 100 node; ADR không tinh chỉnh |
| [147 — Scheduling Framework](../147-scheduling-framework-vi.md) | B9 — ánh xạ bằng chứng của B1, B4, B5, B6 vào `PreEnqueue`, `QueueSort`, `Filter`, `Score`, `PostFilter`, `Bind`; một profile `default-scheduler` |
| [148 — Đóng gói tài nguyên](../148-resource-bin-packing-vi.md) | B9 — `NodeResourcesFit` dùng chiến lược chấm điểm mặc định; ADR vì sao không bật `MostAllocated` cho cluster hai worker |

Các phần dưới đây **không kiểm chứng được trong lab này**, ghi rõ lý do thay vì bỏ trống:

| Phần | Vì sao không thực hành ở đây |
| --- | --- |
| Bài [142](../142-node-pressure-eviction-vi.md) — kích hoạt eviction do áp lực node thật và quan sát thứ tự trục xuất | Trên cluster lab, `nodefs` **chính là filesystem gốc của node**: ép `nodefs.available` xuống dưới ngưỡng là làm đầy đĩa host, việc [mục 2](#2-quy-ước-và-an-toàn) cấm. Ép `memory.available` thì nạn nhân có thể là chính `kubelet`, `containerd`. B8 thay bằng đọc ngưỡng thật, tính khoảng cách, và xếp hạng theo số liệu thật |
| Bài 142 — toleration `node.kubernetes.io/memory-pressure` tự thêm cho Pod khác `BestEffort` | Chỉ thấy hệ quả khi node thật sự ở `MemoryPressure`. Cùng lý do an toàn như dòng trên |
| Bài [141](../141-pod-priority-preemption-vi.md) — `preemptionPolicy: Never` | Bài xếp vào phần đọc lướt, thực hành ở [giai đoạn 13](../00-ALO-TRINH-ADMIN.md#giai-đoạn-13--lập-lịch-và-workload-nâng-cao). B4 giải thích vì sao giá trị âm đã đủ cho `bookstore-batch` |
| Bài [144](../144-pod-overhead-vi.md) — overhead thật của runtime ảo hóa | Cần Kata/Firecracker, nằm ngoài A1.3. B9 chỉ **mô phỏng phép tính** bằng RuntimeClass trỏ vào handler `runc` có sẵn |
| Bài [146](../146-scheduler-perf-tuning-vi.md), [148](../148-resource-bin-packing-vi.md) — đổi `percentageOfNodesToScore`, bật `MostAllocated` hay `RequestedToCapacityRatio` | Phải sửa `KubeSchedulerConfiguration` của control plane, thuộc [giai đoạn 20](../00-ALO-TRINH-ADMIN.md#giai-đoạn-20--cấu-hình-lại-cluster-đang-chạy). Riêng bài 146 không có gì để đo: ba node luôn dưới ngưỡng 100 node khả thi |
| Bài [147](../147-scheduling-framework-vi.md) — viết plugin, chạy scheduler thứ hai | Phải biên dịch hoặc triển khai scheduler riêng; [giai đoạn 14](../00-ALO-TRINH-ADMIN.md#giai-đoạn-14--khả-năng-mở-rộng) |
| Bài [140](../140-topology-spread-constraints-vi.md) — `minDomains`, `matchLabelKeys`, ràng buộc mặc định cấp cluster | Bài xếp vào phần đọc lướt; hai cái đầu cần node pool co giãn hoặc workload nhiều revision cùng lúc, cái sau phải sửa cấu hình scheduler |
| Bài [138](../138-assign-pod-node-vi.md) — `NodeRestriction`, tiền tố `node-restriction.kubernetes.io/` | Thao tác hardening, cần Node authorizer và admission; [giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy). B2 chỉ hỏi label dữ liệu của dự án có được bảo vệ không |
| Bài [139](../139-taint-and-toleration-vi.md) — toán tử `Gt`/`Lt` cho toleration | Alpha ở baseline, phải bật feature gate — làm lệch baseline |
| Bài [143](../143-api-eviction-vi.md) — `kubectl drain` như một quy trình bảo trì | `cordon`, `drain`, `uncordon` thuộc [giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node), làm ở Lab 12. B8 gọi thẳng subresource `eviction` — đúng cơ chế nằm dưới `drain` |

Đây **không phải nợ lộ trình mới**: chúng là giới hạn cố định của môi trường lab hoặc đã có chỗ ở
giai đoạn sau. [Sổ nợ lab](README.md#5-sổ-nợ-lab) không thay đổi vì lab 7a.

Nợ kỹ thuật của hệ thống sau sprint: **không trả nợ nào**; **phát sinh `TD-12`** — sau khi một
worker quay lại, `web` và `api` không tự trải đều lại (B7). Các nợ còn mở từ sprint trước giữ nguyên
— xem `docs/tech-debt.md` trong repo và [sổ nợ của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án).

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B10, không tính mục ⭐) khoảng 4 giờ. B5, B7 và B8 có bước chờ grace
period và chờ controller hội tụ; thời gian đó **phụ thuộc cấu hình**, nên mọi bước chờ viết dưới
dạng vòng lặp có điều kiện dừng. Có thể chia hai phiên: phiên 1 làm B0–B4, phiên 2 làm B5–B10. Giữa
hai phiên cứ để nguyên cluster, **trừ taint**: B7 phải xong trong một phiên. Đầu phiên 2 chạy lại
gate ngắn A5.5 và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

> **Cảnh báo — lab này KHÔNG làm đầy đĩa hay làm cạn RAM để kích hoạt eviction, và bạn cũng đừng
> tự thêm bước đó.** Trên cluster lab, `nodefs` của kubelet nằm trên filesystem gốc của node; đẩy
> `nodefs.available` xuống dưới ngưỡng là làm đầy đĩa host, và một node hết đĩa thì `containerd`,
> `kubelet` cùng hỏng chứ không chỉ Pod bị trục xuất. B8 kiểm chứng phần **đọc được**: ngưỡng thật,
> tín hiệu thật, khoảng cách giữa hai thứ đó. Muốn thấy eviction thật thì làm trên cluster dùng một
> lần, không làm trên cluster mang chuỗi snapshot của lộ trình.

- Mọi lệnh chạy trên `lab-k8s-master` trừ khi ghi rõ node khác. Lệnh trên worker đi qua
  `ssh <node> 'sudo ...'` như các lab trước.
- **Fault injection chỉ trên `lab-k8s-worker2`.** Taint bảo trì (B7), lấp đầy CPU để ép preemption
  (B5) và mọi diễn tập chạm vào `db-0` (B2, B7) đều nhắm đúng node này. `lab-k8s-worker1` chỉ nhận
  Pod bình thường.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/7a/`. Bằng chứng: `~/lab-evidence/7a/`.
- Lab dùng kiến thức nhóm 7a và mọi nhóm trước (tới Lab 6b). **Không** dùng ResourceQuota,
  LimitRange (nhóm [7b](../00-ALO-TRINH-ADMIN.md#7b-chính-sách-giới-hạn-tài-nguyên)), RBAC, Pod
  Security ([giai đoạn 9](../00-ALO-TRINH-ADMIN.md#giai-đoạn-9--bảo-mật-và-multi-tenancy)),
  metrics-server, HPA ([giai đoạn 11](../00-ALO-TRINH-ADMIN.md#giai-đoạn-11--observability)), và
  **không** chạy `kubectl cordon`, `drain`, `uncordon` — ba lệnh đó thuộc
  [giai đoạn 16](../00-ALO-TRINH-ADMIN.md#giai-đoạn-16--vòng-đời-node).
- Object thử nghiệm (Pod, Job, Deployment trong `~/lab-work/7a/`) nằm trong namespace `bookstore`
  nhưng **không** mang label `app.kubernetes.io/part-of`; xóa khi xong ticket. Ngoại lệ duy nhất cấp
  cluster là RuntimeClass `bookstore-overhead-thu` ở B9 — xóa ngay trong B9.
- Repo là nguồn sự thật. Lệnh imperative — `kubectl label node`, `kubectl taint`, `kubectl scale`,
  `kubectl rollout restart`, `kubectl patch pod` — chỉ dùng trong diễn tập và điều tra; trạng thái
  cuối luôn dựng lại được bằng `./scripts/deploy.sh`.
- Label `bookstore.example.com/data` trên node không nằm trong manifest nào: **chỉ** `deploy.sh` gắn
  nó, **chỉ** `teardown.sh` gỡ nó. Taint `bookstore.example.com/bao-tri` của B7 phải được gỡ ngay
  trong B7; `teardown.sh` gỡ thêm một lần làm lưới an toàn.
- **Không gỡ taint của control plane.** `node-role.kubernetes.io/control-plane:NoSchedule` trên
  `lab-k8s-master` là dữ kiện của bài học.
- **Không sửa cấu hình kube-scheduler và kubelet.** B8, B9 chỉ **đọc** spec của static Pod
  kube-scheduler và endpoint `configz` của kubelet.
- B7 đặt taint `NoExecute` lên `lab-k8s-worker2`. Pod hạ tầng không dung thứ taint đó (Traefik,
  local-path-provisioner, có thể cả CoreDNS) sẽ **chuyển sang node khác** và không tự quay lại sau
  khi gỡ taint. Đó là hành vi đúng của `NoExecute`; gate cuối chỉ yêu cầu chúng `Running`.
- Mỗi script nghiệm thu tự định nghĩa hàm của nó; chạy mỗi khối trong một shell liền mạch.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 10: Đặt workload đúng chỗ

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `03-storage-ready`, bàn cờ lập lịch còn sạch (không taint
trên worker, không label của dự án trên node), repo ở đúng tag `lab-6b`, và hệ thống của sprint 9
dựng lại được **chỉ từ repo**.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Kiểm tra riêng của điểm bắt đầu này:

```bash
kubectl get storageclass local-path \
  -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}' | grep -qx true \
  && echo 'PASS: local-path la StorageClass mac dinh — dung diem bat dau 03-storage-ready'
kubectl -n traefik get deployment traefik >/dev/null 2>&1 && echo 'PASS: co Traefik cua 02-net-ready'
test -z "$(kubectl get nodes lab-k8s-worker1 lab-k8s-worker2 -o jsonpath='{.items[*].spec.taints}')" \
  && echo 'PASS: hai worker chua mang taint nao'
test -z "$(kubectl get nodes -l bookstore.example.com/data -o name)" \
  && echo 'PASS: chua node nao mang label du lieu cua du an'
for N in lab-k8s-worker1 lab-k8s-worker2; do
  ssh "$N" 'sudo -n true' && echo "PASS: ssh va sudo khong hoi mat khau toi $N"
done
```

Repo và thư mục làm việc. Nếu vừa restore snapshot, repo đã mất — khôi phục từ bundle theo
[mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) trước:

```bash
mkdir -p ~/lab-work/7a ~/lab-evidence/7a
test -d ~/bookstore/.git || git clone ~/bookstore-lab-6b.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-6b' && echo 'PASS: HEAD dung tag lab-6b'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
```

Dựng lại hệ thống của sprint 9 bằng script của chính sprint đó, rồi chụp ảnh nền của bàn cờ:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/7a/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/7a/b0-smoke-lab-6b.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/7a/b0-smoke-lab-6b.txt

{
  echo "=== $(date -Is) — ban co lap lich luc nhan ban giao ==="
  kubectl get nodes -o custom-columns='NODE:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory,TAINTS:.spec.taints'
  kubectl get nodes --show-labels
  kubectl get priorityclass
  kubectl -n bookstore get pods -o wide
} > ~/lab-evidence/7a/b0-ban-co.txt
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; sáu dòng `PASS:` của điểm bắt đầu (kể cả hai dòng
`ssh`); ba dòng `PASS:` về repo và cluster; `deploy exit=0`, `smoke exit=0` và file smoke không có
dòng `FAIL:`.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 9 **chưa đủ** — có thứ chỉ tồn tại trên
cluster cũ. Sửa repo ở tag `lab-6b` trước, không sửa tay trên cluster
([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)). Không có `ssh`/`sudo` không mật khẩu thì
**dừng**: `teardown.sh` của sprint này cần nó để dọn dữ liệu trên node.

## B1. [BS-7A-01] Điều tra: scheduler đang đặt Bookstore thế nào

**Loại:** Điều tra.

**Bối cảnh:** yêu cầu R10 của dự án là bản sao trải đều trên các node và tác vụ nền không chiếm chỗ
của dịch vụ chính ([mục 2.2](DU-AN-BOOKSTORE.md#22-yêu-cầu-của-hệ-thống-mới)). Từ Lab 3a tới Lab 6b,
ràng buộc lập lịch duy nhất của dự án là `nodeSelector` ghim `db`. Trước khi thêm ràng buộc, đội cần
biết hiện trạng: ai đặt Pod, theo quy tắc nào, và phần nào hiện nay chỉ là may mắn.

**Yêu cầu:**

1. Ghi bảng hiện trạng vào bằng chứng: mỗi Pod của dự án đang nằm trên node nào, mang độ ưu tiên bao
   nhiêu, do scheduler nào đặt; và ai đã ghi quyết định đặt Pod (đọc từ sự kiện).
2. Pod thử nghiệm `qua-co` (`~/lab-work/7a/qua-co.yaml`) xin **nhiều CPU hơn `Allocatable` của node
   lớn nhất** — con số đọc từ cluster, không đoán. Chứng minh Pod nằm `Pending`, chưa được gán node,
   và thông điệp của bước lọc nêu lý do cho **từng nhóm node**: worker thiếu CPU, control plane có
   taint không được dung thứ.
3. Pod thử nghiệm `ghim-ten` (`~/lab-work/7a/ghim-ten.yaml`) đặt thẳng lên `lab-k8s-worker2` mà
   **không đi qua scheduler**. Chứng minh nó chạy mà không có sự kiện lập lịch nào.
4. Tìm trong API ba hiện vật ứng với ba từ khóa của bài 136: một kind riêng cho độ ưu tiên, một
   subresource cho eviction, và chính Pod `qua-co` cho trạng thái chưa được lập lịch.
5. Viết ADR: hiện trạng có đáp ứng R10 không, phần nào là may mắn.

**Ràng buộc:** không sửa repo trong ticket này. Hai Pod thử nghiệm không mang label dự án và bị xóa
khi xong.

**Đọc lại:**

- Bài [137](../137-kube-scheduler-vi.md): [*Tổng quan về lập lịch*](../137-kube-scheduler-vi.md#scheduling),
  [*Chọn node trong kube-scheduler*](../137-kube-scheduler-vi.md#kube-scheduler-implementation).
- Bài [136](../136-scheduling-eviction-vi.md) — đoạn mở đầu và
  [*Sự gián đoạn Pod*](../136-scheduling-eviction-vi.md#sự-gián-đoạn-pod-pod-disruption).
- Bài [138](../138-assign-pod-node-vi.md#nodename), mục *nodeName*; bài
  [267](../267-assign-pods-nodes-vi.md#tạo-một-pod-được-lên-lịch-vào-một-node-cụ-thể-create-a-pod-that-gets-scheduled-to-specific-node),
  mục *Tạo một Pod được lên lịch vào một node cụ thể*.
- Bài [139](../139-taint-and-toleration-vi.md#concepts), mục *Khái niệm* — vì sao Pod thường không
  lên control plane.

**Bàn giao:** `docs/adr/NNNN-hien-trang-lap-lich.md` (`NNNN` là số tiếp theo trong `docs/adr/`);
`~/lab-evidence/7a/b1-*.txt`.

**Nghiệm thu:**

- `PASS:` có sự kiện `Scheduled` cho Pod của dự án do `default-scheduler` ghi.
- `PASS:` `qua-co` xin nhiều CPU hơn node lớn nhất; phase `Pending`, `PodScheduled=False` với lý do
  `Unschedulable`, `nodeName` rỗng.
- `PASS:` thông điệp lọc chứa cả `Insufficient cpu` lẫn `untolerated taint`.
- `PASS:` `ghim-ten` chạy trên `lab-k8s-worker2` mà không có sự kiện `Scheduled` nào.
- `PASS:` API có kind `priorityclasses` và subresource `pods/eviction`.
- `PASS:` hai Pod thử nghiệm đã bị xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-hien-trang-lap-lich.md`:

1. Với `qua-co`, bước lọc trả về tập rỗng. Bước chấm điểm có chạy không? Pod này khác một Pod hỏng
   ở đâu? Dẫn đúng câu của bài 137.
2. Đọc bảng hiện trạng: ba bản sao `web` có **được bảo đảm** nằm trên hai worker không? Khi nhiều
   node cùng điểm cao nhất, scheduler chọn thế nào — và điều đó nói gì về R10?
3. Từ 3a tới 6b `db` được ghim bằng `nodeSelector` theo hostname; `ghim-ten` dùng `nodeName`. Cách
   nào đi qua scheduler? Nếu `lab-k8s-worker2` hết CPU thì mỗi cách cho kết quả gì?

<details>
<summary>Gợi ý</summary>

- `Allocatable` của từng node:
  `kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name} {.status.allocatable.cpu}{"\n"}{end}'`.
  Đơn vị có thể là số nguyên CPU (`2`) hoặc millicore (`1900m`).
- Thông điệp bước lọc nằm ở `status.conditions[?(@.type=="PodScheduled")].message` và ở sự kiện
  `FailedScheduling` của Pod.
- Sự kiện lập lịch: `kubectl -n bookstore get events --field-selector reason=Scheduled`. Sự kiện hết
  hạn sau một khoảng thời gian phụ thuộc cấu hình apiserver; Pod của Job `db-migrate` được tạo lại ở
  mỗi lần `deploy.sh` nên luôn có sự kiện mới.
- API discovery: `kubectl api-resources --api-group=scheduling.k8s.io`;
  `kubectl get --raw /api/v1 | tr ',' '\n' | grep eviction`.
- `nodeName` là một trường của `spec`, không cần label nào.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã tạo hai Pod thử nghiệm</summary>

```bash
NS=bookstore; EV=~/lab-evidence/7a
cpu2m() { case "$1" in '') echo 0 ;; *m) echo "${1%m}" ;; *) awk -v x="$1" 'BEGIN{printf "%d\n", x*1000}' ;; esac; }
jp() { kubectl -n "$NS" get pod "$1" -o jsonpath="$2" 2>/dev/null; }

kubectl -n "$NS" get pods \
  -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,PRIORITY:.spec.priority,SCHEDULER:.spec.schedulerName' \
  > "$EV/b1-hien-trang.txt"
kubectl -n "$NS" get events --field-selector reason=Scheduled \
  -o custom-columns='POD:.involvedObject.name,SOURCE:.source.component,REPORTER:.reportingComponent' \
  > "$EV/b1-scheduled.txt"
grep -q 'default-scheduler' "$EV/b1-scheduled.txt" && echo 'PASS: quyet dinh dat Pod do default-scheduler ghi'

MAX=0
for N in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  M="$(cpu2m "$(kubectl get node "$N" -o jsonpath='{.status.allocatable.cpu}')")"
  [ "$M" -gt "$MAX" ] && MAX="$M"
done
REQ="$(cpu2m "$(jp qua-co '{.spec.containers[0].resources.requests.cpu}')")"
echo "allocatable lon nhat=${MAX}m ; qua-co xin=${REQ}m" > "$EV/b1-qua-co.txt"
[ "$REQ" -gt "$MAX" ] && echo 'PASS: qua-co xin nhieu CPU hon node lon nhat'
for i in $(seq 1 30); do
  [ "$(jp qua-co '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ] && break
  sleep 2
done
{ [ "$(jp qua-co '{.status.phase}')" = 'Pending' ] && \
  [ "$(jp qua-co '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ] && \
  [ -z "$(jp qua-co '{.spec.nodeName}')" ]; } && echo 'PASS: qua-co Pending, Unschedulable, chua gan node'
jp qua-co '{.status.conditions[?(@.type=="PodScheduled")].message}' >> "$EV/b1-qua-co.txt"; echo >> "$EV/b1-qua-co.txt"
cat "$EV/b1-qua-co.txt"
{ grep -q 'Insufficient cpu' "$EV/b1-qua-co.txt" && grep -q 'untolerated taint' "$EV/b1-qua-co.txt"; } \
  && echo 'PASS: buoc loc neu ly do cho tung nhom node'

kubectl -n "$NS" wait --for=condition=Ready pod/ghim-ten --timeout=120s >/dev/null
SCHED="$(kubectl -n "$NS" get events --field-selector involvedObject.name=ghim-ten,reason=Scheduled --no-headers 2>/dev/null | wc -l)"
{ [ "$(jp ghim-ten '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && [ "$SCHED" -eq 0 ]; } \
  && echo 'PASS: ghim-ten chay tren worker2 ma khong co su kien Scheduled'

{ kubectl api-resources --api-group=scheduling.k8s.io
  kubectl get --raw /api/v1 | tr ',' '\n' | grep 'pods/eviction'; } > "$EV/b1-ba-tu-khoa.txt"
{ grep -q 'priorityclasses' "$EV/b1-ba-tu-khoa.txt" && grep -q 'pods/eviction' "$EV/b1-ba-tu-khoa.txt"; } \
  && echo 'PASS: co kind PriorityClass va subresource pods/eviction'

kubectl -n "$NS" delete pod qua-co ghim-ten --ignore-not-found --wait=true >/dev/null
[ -z "$(kubectl -n "$NS" get pods -o name | grep -E '^pod/(qua-co|ghim-ten)$')" ] && echo 'PASS: da xoa Pod thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
cpu2m() { case "$1" in '') echo 0 ;; *m) echo "${1%m}" ;; *) awk -v x="$1" 'BEGIN{printf "%d\n", x*1000}' ;; esac; }
MAX=0
for N in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  M="$(cpu2m "$(kubectl get node "$N" -o jsonpath='{.status.allocatable.cpu}')")"
  [ "$M" -gt "$MAX" ] && MAX="$M"
done

cat > ~/lab-work/7a/qua-co.yaml <<EOF
apiVersion: v1
kind: Pod
metadata: {name: qua-co, namespace: bookstore}
spec:
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources: {requests: {cpu: "$((MAX + 500))m", memory: 16Mi}}   # hơn node lớn nhất 500m
EOF

cat > ~/lab-work/7a/ghim-ten.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata: {name: ghim-ten, namespace: bookstore}
spec:
  nodeName: lab-k8s-worker2          # không đi qua scheduler
  terminationGracePeriodSeconds: 5
  containers:
  - {name: app, image: "busybox:1.37", command: ["sh", "-c", "sleep 3600"], resources: {requests: {cpu: 10m, memory: 16Mi}}}
EOF

kubectl apply -f ~/lab-work/7a/qua-co.yaml -f ~/lab-work/7a/ghim-ten.yaml
```

Ý chính cho ADR:

1. Không. Bước chấm điểm chỉ nhận **tập khả thi** của bước lọc; tập rỗng thì không có gì để chấm.
   Bài 137: nếu không có node nào phù hợp, Pod ở trạng thái chưa được lập lịch **cho đến khi**
   scheduler sắp đặt được nó — Pod không hỏng, nó nằm trong hàng đợi và được thử lại khi cluster đổi.
2. Không được bảo đảm. Hiện chưa có ràng buộc nào ép trải; scheduler chấm điểm bằng các hàm mặc
   định, và khi nhiều node cùng điểm cao nhất thì **chọn ngẫu nhiên**. Phân bố đẹp hôm nay là may
   mắn; R10 phải được khai báo thành ràng buộc — đó là B3.
3. `nodeSelector` đi qua scheduler: worker2 hết CPU thì Pod `Pending` với lý do lọc và chờ chỗ.
   `nodeName` bỏ qua scheduler: kubelet trên worker2 tự nhận Pod và từ chối với lý do `OutOfcpu`,
   Pod thất bại chứ không được sắp xếp lại (bài 138, mục *nodeName*). Bài 138 dặn dùng node affinity
   hoặc `nodeSelector` để ghim Pod mà không bỏ qua scheduler.

</details>

## B2. [BS-7A-02] `db` đi theo label dữ liệu của dự án

**Loại:** Tính năng.

**Bối cảnh:** `db` đang ghim bằng `kubernetes.io/hostname: lab-k8s-worker2` — tên một máy cụ thể nằm
trong manifest. Kế hoạch dự án ([mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch))
chuyển sang một label do dự án quản lý: "node dữ liệu" là một **vai trò**, không phải một tên máy. Từ
Lab 6a dữ liệu nằm trên PV `local-path` gắn chặt với worker2, nên ràng buộc mới phải khớp với ràng
buộc của volume chứ không được mâu thuẫn với nó.

**Yêu cầu:**

1. `scripts/deploy.sh` và `scripts/teardown.sh` thay bằng phiên bản Lab 7a — chép nguyên văn từ khối
   bên dưới. `deploy.sh` gắn label `bookstore.example.com/data=true` cho **đúng** `lab-k8s-worker2`
   trước khi apply; `teardown.sh` gỡ label đó và mọi taint `bookstore.example.com/*` khỏi mọi node.
2. `db` chọn node bằng **node affinity bắt buộc** theo label trên; không còn ghim theo hostname.
3. Đọc ràng buộc node mà provisioner đã ghi trên PV của `data-db-0`, đặt cạnh ràng buộc của `db-0`.
   Ghi vào ADR: hai ràng buộc, ai đặt mỗi cái, lúc nào mỗi cái được đánh giá.
4. Pod thử nghiệm `chon-nhan` (`~/lab-work/7a/chon-nhan.yaml`) dùng **`nodeSelector`** khớp đúng label
   đó — cách làm của bài 267 — và phải rơi vào `lab-k8s-worker2`.
5. Chứng minh `IgnoredDuringExecution` trên `db` thật: gỡ label khỏi worker2 thì `db-0` đang chạy
   **không** bị đuổi; nhưng nếu `db-0` bị thay trong lúc thiếu label thì Pod mới không lập lịch được.
   Khôi phục bằng `./scripts/deploy.sh`; dữ liệu còn nguyên.
6. ⭐ Gắn nhầm label sang `lab-k8s-worker1` (và gỡ khỏi worker2) rồi thay `db-0`: đọc thông điệp lọc,
   chỉ ra ràng buộc nào loại node nào.

**Ràng buộc:** không sửa `volumeClaimTemplates` (trường bất biến của StatefulSet). Không xóa PVC hay
PV. Label của node không nằm trong manifest nào — `deploy.sh` là chỗ duy nhất gắn nó. Diễn tập ở yêu
cầu 5, 6 làm `db` mất sẵn sàng trong vài phút; chỉ làm trên worker2.

**Đọc lại:**

- Bài [138](../138-assign-pod-node-vi.md): [*Node affinity*](../138-assign-pod-node-vi.md#node-affinity)
  — cả hai ghi chú: `IgnoredDuringExecution` và quy tắc AND/OR;
  [*Toán tử*](../138-assign-pod-node-vi.md#operators);
  [*Cô lập/hạn chế node*](../138-assign-pod-node-vi.md#node-isolation-restriction) (chỉ để trả lời câu
  hỏi 4).
- Bài [266](../266-assign-pods-nodes-node-affinity-vi.md#lên-lịch-một-pod-bằng-node-affinity-bắt-buộc-schedule-a-pod-using-required-node-affinity)
  — *Lên lịch một Pod bằng node affinity bắt buộc*.
- Bài [267](../267-assign-pods-nodes-vi.md#add-a-label-to-a-node) — *Thêm label cho một node* và
  [*Tạo một Pod được lên lịch vào node bạn đã chọn*](../267-assign-pods-nodes-vi.md#tạo-một-pod-được-lên-lịch-vào-node-bạn-đã-chọn-create-a-pod-that-gets-scheduled-to-your-chosen-node).
- Bài [96](../96-storage-classes-vi.md#volume-binding-mode) (giai đoạn 6), mục *Chế độ gắn kết volume*
  — vì sao PV `local-path` mang `nodeAffinity`.

**Bàn giao:** `k8s/app/db/statefulset.yaml`; hai script mới; `docs/adr/NNNN-db-node-affinity.md`;
`~/lab-evidence/7a/b2-*.txt`.

**Nghiệm thu:**

- `PASS:` `teardown.sh` in `teardown: sach` và không node nào còn label dữ liệu; `deploy.sh` chạy hai
  lần liên tiếp đều thành công.
- `PASS:` đúng một node mang label dữ liệu, và đó là `lab-k8s-worker2`.
- `PASS:` `db-0` chạy trên `lab-k8s-worker2`; StatefulSet chọn node bằng node affinity bắt buộc trên
  label dữ liệu; không còn `nodeSelector`.
- `PASS:` PV của `data-db-0` mang `nodeAffinity` trỏ `lab-k8s-worker2`.
- `PASS:` `chon-nhan` chạy trên `lab-k8s-worker2`.
- `PASS:` gỡ label thì `db-0` giữ nguyên UID và vẫn Ready.
- `PASS:` thay `db-0` lúc thiếu label thì Pod mới `Pending` với lý do node affinity; `deploy.sh` khôi
  phục, `db-0` Ready trên worker2 với ít nhất 8 cuốn sách.
- `PASS:` (⭐) gắn nhầm label: thông điệp lọc có cả `volume node affinity conflict` lẫn
  `node affinity/selector`.

Hai script, chép nguyên văn:

```bash
cat > ~/bookstore/scripts/deploy.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai dự án Bookstore từ repo — phiên bản Lab 7a.
# Từ 5a các tầng gọi nhau bằng tên DNS, không còn file .tpl. Lab 7a thêm: gắn label node dữ liệu
# trước khi apply, và tách Job db-migrate ra để chạy sau khi db-0 Ready.
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore
HOST=bookstore.lab.local
DATA_NODE=lab-k8s-worker2
DATA_LABEL=bookstore.example.com/data

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

# 1. Bí mật sinh tại chỗ khi thiếu; secrets/ không vào git (3b, 5b). Dữ liệu DB cũ còn trên
#    cluster thì không sinh mới: mật khẩu thật nằm trong chính dữ liệu đó (6a).
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
    openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
      -keyout secrets/tls.key -out secrets/tls.crt \
      -subj '/CN=bookstore.lab.local' -addext 'subjectAltName=DNS:bookstore.lab.local' 2>/dev/null )
  echo 'deploy: da sinh certificate tu ky moi'
fi

# 2. Node dữ liệu (7a): đúng một node mang label của dự án, gắn trước khi db-0 cần lập lịch.
kubectl label node "$DATA_NODE" "${DATA_LABEL}=true" --overwrite
for N in $(kubectl get nodes -l "$DATA_LABEL" -o jsonpath='{.items[*].metadata.name}'); do
  [ "$N" = "$DATA_NODE" ] || kubectl label node "$N" "${DATA_LABEL}-"
done

# 3. Object cấp cluster, rồi toàn bộ k8s/app trừ Job db-migrate. secrets/ nằm ngoài thư mục gốc
#    k8s/app nên phải render với --load-restrictor (3b, ADR 0007).
if compgen -G 'k8s/cluster/*.yaml' >/dev/null; then kubectl apply -f k8s/cluster/; fi
RENDER="$(mktemp)"
trap 'rm -f "$RENDER"' EXIT
kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app > "$RENDER"
kubectl apply -f "$RENDER" -l 'app.kubernetes.io/component!=migrate'

# 4. Chờ db-0. PVC data-db-0 được cấp phát khi db-0 được lập lịch (WaitForFirstConsumer, 6a).
kubectl -n "$NS" rollout status statefulset/db --timeout=300s
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s

wait_job() {   # thành công khi Job Complete, thất bại ngay khi Job Failed
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

# 5. Migration chạy ở MỖI lần triển khai nên phải bất biến kết quả (4b). Pod template của Job không
#    sửa được sau khi tạo, nên xóa Job cũ rồi tạo lại.
kubectl -n "$NS" delete job db-migrate --ignore-not-found --wait=true
if awk '/^---/{k=""} /^kind: /{k=$2} k=="Job" && /^  name: db-migrate$/{f=1} END{exit !f}' "$RENDER"; then
  kubectl apply -f "$RENDER" -l 'app.kubernetes.io/component=migrate'
elif [ -f k8s/app/jobs/db-migrate.yaml ]; then
  kubectl apply -f k8s/app/jobs/db-migrate.yaml
fi
if kubectl -n "$NS" get job db-migrate >/dev/null 2>&1; then
  wait_job db-migrate || { echo 'deploy: db-migrate that bai'; exit 1; }
fi

# 6. Chờ các controller còn lại.
for D in web api; do
  kubectl -n "$NS" rollout status "deployment/$D" --timeout=300s
done
if kubectl -n "$NS" get daemonset node-agent >/dev/null 2>&1; then
  kubectl -n "$NS" rollout status daemonset/node-agent --timeout=300s
fi

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

kubectl -n "$NS" get statefulset,deployment,daemonset,cronjob,job,pvc -o wide
EOF

cat > ~/bookstore/scripts/teardown.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ toàn bộ dự án Bookstore khỏi cluster — phiên bản Lab 7a.
set -euo pipefail
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
DATA_LABEL=bookstore.example.com/data
WORKERS='lab-k8s-worker1 lab-k8s-worker2'
# Kind namespaced được dọn theo label ở mọi namespace (3b, 4a, 5a, 5b, 6a).
KINDS='statefulsets,deployments,daemonsets,cronjobs,jobs,replicasets,pods,poddisruptionbudgets,services,networkpolicies,ingresses,configmaps,secrets,persistentvolumeclaims'

# 1. db tắt êm trước: xóa StatefulSet không bảo đảm Pod tắt có thứ tự (bài 65, mục Hạn chế).
if kubectl -n "$NS" get statefulset db >/dev/null 2>&1; then
  kubectl -n "$NS" scale statefulset db --replicas=0
  kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s || true
fi

# 2. Namespace và mọi object mang label dự án, kể cả cấp cluster; StorageClass của dự án xóa cả
#    theo tên (6a).
kubectl delete namespace -l "$SEL" --ignore-not-found --wait=true --timeout=300s
kubectl delete "$KINDS" --all-namespaces -l "$SEL" --ignore-not-found --wait=true --timeout=120s
kubectl delete priorityclass,storageclass -l "$SEL" --ignore-not-found
kubectl delete storageclass bookstore-retain --ignore-not-found

# 3. PV của dự án: PV do provisioner tạo không mang label dự án, nhận diện qua class và claim;
#    PV tạo tay thì mang label dự án (6a).
pv_rows() {
  kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}|{.spec.storageClassName}|{.spec.claimRef.namespace}|{.spec.persistentVolumeReclaimPolicy}|{.metadata.labels.app\.kubernetes\.io/part-of}{"\n"}{end}'
}
for _ in $(seq 1 30); do   # PV Delete do provisioner tự thu hồi; chờ nó một lúc
  pv_rows | awk -F'|' -v ns="$NS" '$3==ns && $4=="Delete"' | grep -q . || break
  sleep 2
done
for PV in $(pv_rows | awk -F'|' -v ns="$NS" '$2=="bookstore-retain" || $3==ns || $5=="bookstore" {print $1}'); do
  kubectl delete pv "$PV" --ignore-not-found --wait=true --timeout=120s
done

# 4. Dữ liệu sót trên node: thư mục <pv>_bookstore_<pvc> của local-path-provisioner (6a).
for N in $WORKERS; do
  ssh "$N" "sudo find /opt/local-path-provisioner -mindepth 1 -maxdepth 1 -name '*_bookstore_*' -exec rm -rf {} +" || true
done

# 5. Label và taint mà dự án đặt lên node (7a).
for N in $(kubectl get nodes -l "$DATA_LABEL" -o jsonpath='{.items[*].metadata.name}'); do
  kubectl label node "$N" "${DATA_LABEL}-"
done
for N in $(kubectl get nodes -o jsonpath='{.items[*].metadata.name}'); do
  for K in $(kubectl get node "$N" -o jsonpath='{range .spec.taints[*]}{.key}{"\n"}{end}' \
             | grep '^bookstore\.example\.com/' | sort -u || true); do
    kubectl taint node "$N" "${K}-"
  done
done

# 6. Tự kiểm: không còn gì của dự án.
LEFT_CLUSTER="$( { kubectl get namespace,priorityclass,storageclass -l "$SEL" --no-headers
                   kubectl get storageclass bookstore-retain --no-headers --ignore-not-found; } 2>/dev/null | wc -l)"
LEFT_NS="$( { kubectl get all -A -l "$SEL" --no-headers
              kubectl get "$KINDS,endpointslices" -A -l "$SEL" --no-headers; } 2>/dev/null | wc -l)"
LEFT_PV="$(pv_rows | awk -F'|' -v ns="$NS" '$2=="bookstore-retain" || $3==ns || $5=="bookstore"' | wc -l)"
LEFT_NODE="$(kubectl get nodes -l "$DATA_LABEL" --no-headers 2>/dev/null | wc -l)"
LEFT_TAINT="$(kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' \
              | grep -c '^bookstore\.example\.com/' || true)"
LEFT_DIR=0
for N in $WORKERS; do
  C="$(ssh "$N" "sudo find /opt/local-path-provisioner -mindepth 1 -maxdepth 1 -name '*_bookstore_*' 2>/dev/null | wc -l")"
  LEFT_DIR=$((LEFT_DIR + ${C:-0}))
done
TOTAL=$((LEFT_CLUSTER + LEFT_NS + LEFT_PV + LEFT_NODE + LEFT_TAINT + LEFT_DIR))
if [ "$TOTAL" -eq 0 ]; then
  echo 'teardown: sach'
else
  echo "teardown: con cap-cluster=$LEFT_CLUSTER namespaced=$LEFT_NS pv=$LEFT_PV label-node=$LEFT_NODE taint=$LEFT_TAINT thu-muc=$LEFT_DIR"
  exit 1
fi
# Hạ tầng nền không mang label dự án và phải còn nguyên (5b).
{ kubectl get ingressclass traefik >/dev/null 2>&1 && kubectl -n kube-system get daemonset calico-node >/dev/null 2>&1; } \
  || echo 'teardown: CANH BAO - ha tang 02-net-ready khong con nguyen (khong do script nay go)'
EOF

chmod +x ~/bookstore/scripts/*.sh
```

Ba điểm mới so với phiên bản của các sprint trước:

- `deploy.sh` apply bản render **hai lần có lọc label**: lần đầu mọi thứ trừ thành phần `migrate`,
  lần sau chỉ Job `db-migrate` khi `db-0` đã Ready. Nếu sprint 6 của bạn để manifest migration ngoài
  `kustomization.yaml` ở `k8s/app/jobs/db-migrate.yaml`, nhánh `elif` dùng file đó.
- `deploy.sh` gắn label node dữ liệu và gỡ nó khỏi mọi node khác — chạy lại `deploy.sh` cũng là cách
  sửa label gắn nhầm.
- `teardown.sh` giữ nguyên phần dọn PV và thư mục dữ liệu trên node của sprint 8, thêm phần gỡ label
  và taint của dự án, và đếm cả hai thứ đó trong phần tự kiểm.

Các bước của sprint 7, 8 vẫn giữ: kiểm hạ tầng nền, không sinh `secrets/db.env` mới khi dữ liệu cũ còn,
kiểm HTTPS qua Traefik ở cuối `deploy.sh`.

Nếu `deploy.sh` ở tag `lab-6b` của bạn sinh bí mật khác đi (tên file, tham số certificate), giữ
nguyên khối số 1 của bản cũ và chỉ thay phần còn lại.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-db-node-affinity.md`:

1. `db-0` giờ chịu hai ràng buộc node: node affinity của Pod và `nodeAffinity` của PV. Ai đặt từng
   cái, lúc nào? Scheduler kết hợp chúng theo quy tắc nào — dẫn ghi chú của bài 138.
2. Vì sao chọn node affinity `required` chứ không phải `nodeSelector` với label mới, dù cả hai đều
   là ràng buộc cứng? Khi nào bạn cần một `operator` khác `In`?
3. Ở yêu cầu 5, gỡ label không đuổi `db-0` nhưng làm `db-0` không quay lại được. Giải thích bằng
   `IgnoredDuringExecution`. Cơ chế nào mới đuổi được Pod đang chạy khỏi một node?
4. Label `bookstore.example.com/data` có chống được một kubelet bị chiếm tự gắn label đó cho node
   của nó không? Bài 138 gợi ý gì? Vì sao trên cluster này `db` vẫn không chạy sang node đó?

<details>
<summary>Gợi ý</summary>

1. Khối cần viết nằm ở `spec.template.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution`:
   `nodeSelectorTerms` là danh sách term, mỗi term có `matchExpressions` với `key`, `operator`,
   `values`. Giá trị label là chuỗi — viết `"true"` có nháy.
2. Template của StatefulSet đổi thì `db-0` bị thay theo rolling update; PV giữ dữ liệu.
3. Tên PV: `kubectl -n bookstore get pvc data-db-0 -o jsonpath='{.spec.volumeName}'`; ràng buộc của
   nó: `kubectl get pv <tên> -o jsonpath='{.spec.nodeAffinity}'`.
4. Gỡ label: `kubectl label node lab-k8s-worker2 bookstore.example.com/data-`.
5. Trình tự an toàn: chép hai script, sửa manifest, rồi
   `./scripts/teardown.sh && ./scripts/deploy.sh`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; teardown, deploy, rồi diễn tập trên `db`</summary>

```bash
NS=bookstore; EV=~/lab-evidence/7a; DL=bookstore.example.com/data
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
books() { kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null; }
wait_unsched() {
  for i in $(seq 1 40); do
    [ "$(jp pod/db-0 '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ] && return 0
    sleep 3
  done
  return 1
}

./scripts/teardown.sh > "$EV/b2-teardown.txt" 2>&1
{ grep -q 'teardown: sach' "$EV/b2-teardown.txt" && [ -z "$(kubectl get nodes -l "$DL" -o name)" ]; } \
  && echo 'PASS: teardown sach, khong node nao con label du lieu'
./scripts/deploy.sh > "$EV/b2-deploy-1.txt" 2>&1 && ./scripts/deploy.sh > "$EV/b2-deploy-2.txt" 2>&1 \
  && echo 'PASS: deploy.sh chay hai lan lien tiep thanh cong'

[ "$(kubectl get nodes -l "$DL=true" -o jsonpath='{.items[*].metadata.name}')" = 'lab-k8s-worker2' ] \
  && echo 'PASS: dung mot node mang label du lieu: lab-k8s-worker2'
NA='{.spec.template.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[*].matchExpressions[*].key}'
{ [ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && \
  jp statefulset/db "$NA" | grep -q "$DL" && \
  [ -z "$(jp statefulset/db '{.spec.template.spec.nodeSelector}')" ]; } \
  && echo 'PASS: db-0 tren worker2 nho node affinity bat buoc, khong con nodeSelector'
PV="$(jp pvc/data-db-0 '{.spec.volumeName}')"
kubectl get pv "$PV" -o jsonpath='{.spec.nodeAffinity}' > "$EV/b2-pv-affinity.txt"
grep -q 'lab-k8s-worker2' "$EV/b2-pv-affinity.txt" && echo "PASS: PV $PV gan voi lab-k8s-worker2"

kubectl apply -f ~/lab-work/7a/chon-nhan.yaml >/dev/null
kubectl -n "$NS" wait --for=condition=Ready pod/chon-nhan --timeout=120s >/dev/null
[ "$(jp pod/chon-nhan '{.spec.nodeName}')" = 'lab-k8s-worker2' ] \
  && echo 'PASS: nodeSelector tren label du lieu dua Pod ve worker2'
kubectl -n "$NS" delete pod chon-nhan --wait=true >/dev/null

# Diễn tập trên worker2: IgnoredDuringExecution
U0="$(jp pod/db-0 '{.metadata.uid}')"
kubectl label node lab-k8s-worker2 "$DL-"
sleep 30
{ [ "$(jp pod/db-0 '{.metadata.uid}')" = "$U0" ] && \
  [ "$(jp pod/db-0 '{.status.conditions[?(@.type=="Ready")].status}')" = 'True' ]; } \
  && echo 'PASS: go label khong duoi db-0 dang chay'
kubectl -n "$NS" delete pod db-0 --wait=true
wait_unsched
jp pod/db-0 '{.status.conditions[?(@.type=="PodScheduled")].message}' > "$EV/b2-thieu-label.txt"
{ [ "$(jp pod/db-0 '{.status.phase}')" = 'Pending' ] && grep -q 'node affinity' "$EV/b2-thieu-label.txt"; } \
  && echo 'PASS: db-0 moi khong lap lich duoc khi thieu label'
./scripts/deploy.sh > "$EV/b2-deploy-khoi-phuc.txt" 2>&1
N="$(books)"
{ [ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && [ "${N:-0}" -ge 8 ]; } \
  && echo "PASS: deploy.sh khoi phuc, db-0 tren worker2 voi $N cuon sach"
```

⭐ Gắn nhầm label:

```bash
NS=bookstore; EV=~/lab-evidence/7a; DL=bookstore.example.com/data
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
kubectl label node lab-k8s-worker2 "$DL-"
kubectl label node lab-k8s-worker1 "$DL=true"
kubectl -n "$NS" delete pod db-0 --wait=true
for i in $(seq 1 40); do
  [ "$(jp pod/db-0 '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ] && break
  sleep 3
done
jp pod/db-0 '{.status.conditions[?(@.type=="PodScheduled")].message}' | tee "$EV/b2-nhan-nham.txt"; echo
{ grep -q 'volume node affinity conflict' "$EV/b2-nhan-nham.txt" && \
  grep -q 'node affinity/selector' "$EV/b2-nhan-nham.txt"; } \
  && echo 'PASS: worker1 bi PV loai, worker2 bi node affinity cua Pod loai'
cd ~/bookstore && ./scripts/deploy.sh > "$EV/b2-deploy-sau-nhan-nham.txt" 2>&1 \
  && [ "$(kubectl get nodes -l "$DL=true" -o jsonpath='{.items[*].metadata.name}')" = 'lab-k8s-worker2' ] \
  && echo 'PASS: deploy.sh go label nham va dua db-0 ve worker2'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Trong `k8s/app/db/statefulset.yaml`, ở `spec.template.spec`: **xóa** khối `nodeSelector` theo
hostname, **thêm** khối `affinity`. Phần còn lại của StatefulSet (của bạn từ Lab 6a) giữ nguyên.

```yaml
# k8s/app/db/statefulset.yaml — chỉ phần thay đổi, trong spec.template.spec
      # nodeSelector:                             # bỏ từ Lab 7a
      #   kubernetes.io/hostname: lab-k8s-worker2
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: bookstore.example.com/data   # do scripts/deploy.sh gắn cho lab-k8s-worker2
                operator: In
                values: ["true"]
```

```yaml
# ~/lab-work/7a/chon-nhan.yaml
apiVersion: v1
kind: Pod
metadata: {name: chon-nhan, namespace: bookstore}
spec:
  nodeSelector:
    bookstore.example.com/data: "true"
  terminationGracePeriodSeconds: 5
  containers:
  - {name: app, image: "busybox:1.37", command: ["sh", "-c", "sleep 3600"], resources: {requests: {cpu: 10m, memory: 16Mi}}}
```

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh
kubectl get nodes -L bookstore.example.com/data
```

Ý chính cho ADR:

1. Node affinity của Pod do **bạn** viết trong template; scheduler đánh giá nó ở bước lọc mỗi lần
   lập lịch `db-0`. `nodeAffinity` của PV do **provisioner** ghi khi cấp phát volume lần đầu (lúc
   `db-0` đầu tiên được lập lịch, nhờ `WaitForFirstConsumer`); từ đó scheduler cũng lọc theo nó cho
   mọi Pod dùng PVC này. Hai ràng buộc cùng phải thỏa — tinh thần ghi chú "đặt cả `nodeSelector` lẫn
   `nodeAffinity` thì cả hai phải được thỏa mãn" của bài 138: các bộ lọc cộng dồn, không thay nhau.
2. Hiệu lực như nhau; node affinity diễn đạt được nhiều hơn: nhiều giá trị (`In`), phủ định (`NotIn`,
   `DoesNotExist` — B4 dùng), nhiều term OR với nhau, và có dạng mềm. Kế hoạch dự án chọn node
   affinity để mọi ràng buộc node của dự án viết một kiểu. `nodeSelector` vẫn đủ cho Pod thử nghiệm
   `chon-nhan`.
3. Affinity chỉ được đánh giá **lúc lập lịch**: label đổi sau đó thì Pod vẫn chạy tiếp. `db-0` mới
   lại phải qua bước lọc nên bị chặn. Muốn đuổi Pod đang chạy thì dùng taint `NoExecute` (B7).
4. Không. Bài 138: label dùng để cô lập node phải là label mà kubelet **không** sửa được — tiền tố
   `node-restriction.kubernetes.io/` cùng admission `NodeRestriction` (giai đoạn 9). Trên cluster này
   `db-0` vẫn an toàn vì PV `local-path` ghim nó vào worker2 bằng hostname — nhưng thành phần mới nào
   chỉ dựa vào label dự án thì không có lớp bảo vệ đó.

Thông điệp ⭐ có dạng `0/3 nodes are available: 1 node(s) didn't match Pod's node affinity/selector,
1 node(s) had untolerated taint ..., 1 node(s) had volume node affinity conflict` — mỗi node bị loại
bởi đúng một bộ lọc, đọc được ràng buộc nào loại node nào.

</details>

## B3. [BS-7A-03] `web` trải đều, `api` tránh dồn một node

**Loại:** Tính năng, kèm điều tra trên Deployment thử nghiệm.

**Bối cảnh:** B1 cho thấy phân bố hiện tại của `web` và `api` là may mắn. Mất một worker mà cả ba bản
sao `web` nằm trên đó là mất trang chủ. `web` hiện có 3 bản sao, từ Lab 11b sẽ co giãn 2–5 bản sao
theo tải — ràng buộc phải đúng với **mọi** số bản sao. `api` có 2 bản sao và phải tiếp tục chạy đủ
khi chỉ còn một worker nhận Pod (bảo trì ở Lab 12).

**Yêu cầu:**

1. `web`: bản sao chia cho hai worker, chênh lệch giữa hai node **không quá 1**, với bất kỳ số bản sao
   nào. Khi không giữ được điều đó, scheduler **để Pod chờ** chứ không đặt lệch. Miền phân bố là từng
   node.
2. Pod `web` phải được đếm đúng: chính các Pod của `web` khớp điều kiện đếm của ràng buộc — không có
   "pod ma".
3. `api`: hai bản sao **ưu tiên** nằm trên hai node khác nhau, nhưng vẫn chạy cả hai khi chỉ còn một
   node nhận được Pod.
4. Điều tra trên hai Deployment thử nghiệm ở `~/lab-work/7a/`: `chong-don-cung` (3 bản sao,
   anti-affinity cứng theo node) và `chong-don-mem` (cùng quy tắc ở dạng mềm). Đếm Pod chạy và Pod
   chờ của mỗi cái.
5. Điều tra: Deployment thử nghiệm `sai-khoa` (2 bản sao) có ràng buộc phân bố với `topologyKey` mà
   không node nào có — mô phỏng một lỗi gõ label. Ghi lại chuyện gì xảy ra.
6. Tạm scale `web` lên 5 bản sao bằng lệnh để kiểm phân bố, rồi trả về bằng `./scripts/deploy.sh`.

**Ràng buộc:** chỉ dùng `kubernetes.io/hostname` làm miền — cluster không có label zone. Không dùng
`nodeName` hay `nodeSelector` cho `web`, `api`. Trường hợp một node đang bị taint để B7 xử lý.
Deployment thử nghiệm dùng `busybox`, label riêng, không mang label dự án.

**Đọc lại:**

- Bài [140](../140-topology-spread-constraints-vi.md):
  [*Định nghĩa ràng buộc phân bố*](../140-topology-spread-constraints-vi.md#định-nghĩa-ràng-buộc-phân-bố-spread-constraint-definition),
  [*Ví dụ: một ràng buộc phân bố theo topology*](../140-topology-spread-constraints-vi.md#example-one-topologyspreadconstraint),
  [*Các quy ước ngầm định*](../140-topology-spread-constraints-vi.md#các-quy-ước-ngầm-định-implicit-conventions),
  [*So sánh với podAffinity và podAntiAffinity*](../140-topology-spread-constraints-vi.md#comparison-with-podaffinity-podantiaffinity),
  [*Các hạn chế đã biết*](../140-topology-spread-constraints-vi.md#các-hạn-chế-đã-biết-known-limitations).
- Bài [138](../138-assign-pod-node-vi.md):
  [*Inter-pod affinity và anti-affinity*](../138-assign-pod-node-vi.md#inter-pod-affinity-and-anti-affinity),
  [*Các trường hợp sử dụng thực tế hơn*](../138-assign-pod-node-vi.md#các-trường-hợp-sử-dụng-thực-tế-hơn-more-practical-use-cases).

**Bàn giao:** `k8s/app/web/deployment.yaml`; `k8s/app/api/deployment.yaml`;
`docs/adr/NNNN-trai-web-api.md`; `~/lab-evidence/7a/b3-*.txt`.

**Nghiệm thu:**

- `PASS:` `web` 3/3 nằm trên cả hai worker; ràng buộc cứng theo node với `maxSkew` 1.
- `PASS:` selector đếm của ràng buộc chọn đúng các Pod `web` — không có pod ma.
- `PASS:` scale lên 5 thì phân bố là 3/2; `deploy.sh` đưa `web` về 3 bản sao.
- `PASS:` `api` 2/2 trên hai node khác nhau; quy tắc anti-affinity là mềm.
- `PASS:` `chong-don-cung`: 2 Pod chạy, 1 Pod chờ với lý do anti-affinity; `chong-don-mem`: 3 Pod chạy
  trên hai node.
- `PASS:` `sai-khoa`: mọi Pod chờ, lý do là ràng buộc phân bố.
- `PASS:` ba Deployment thử nghiệm đã bị xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-trai-web-api.md`:

1. Với `maxSkew: 1` trên hai worker, 3 bản sao và 5 bản sao phân bố ra sao? Vì sao 4/1 bị loại? Viết
   phép tính theo đúng định nghĩa *mức tối thiểu toàn cục* của bài 140.
2. Vì sao không dùng anti-affinity cứng cho `web`? Dùng kết quả của `chong-don-cung`, và nghĩ tới 5
   bản sao của HPA lẫn Pod dôi ra khi rollout.
3. Vì sao `api` dùng dạng mềm? Nếu dạng cứng, chuyện gì xảy ra khi một worker không nhận Pod (bảo trì)
   và khi rollout cần thêm một Pod tạm?
4. `sai-khoa` cho thấy gì? Vì sao một `topologyKey` không node nào có **không** làm ràng buộc vô hiệu?
   Lỗi này ngoài thực tế trông thế nào trên `kubectl get nodes`?

<details>
<summary>Gợi ý</summary>

1. `topologySpreadConstraints` nằm ở `spec.template.spec` của Deployment: `maxSkew`, `topologyKey`,
   `whenUnsatisfiable`, `labelSelector`.
2. Cho `labelSelector` của ràng buộc dùng `matchLabels` là cặp label hợp đồng `part-of` + `component`
   — giống selector của chính Deployment. Script nghiệm thu đọc `matchLabels` để kiểm pod ma.
3. Anti-affinity mềm: `podAntiAffinity.preferredDuringSchedulingIgnoredDuringExecution`, mỗi phần tử
   gồm `weight` và `podAffinityTerm` (bên trong là `labelSelector` và `topologyKey`). Dạng cứng là danh
   sách term trực tiếp, không có `weight`.
4. Scale tạm: `kubectl -n bookstore scale deployment web --replicas=5`. `kubectl apply` của
   `deploy.sh` sẽ đưa số bản sao về giá trị trong manifest.
5. Deployment thử nghiệm: `busybox:1.37` chạy `sleep 3600`, `requests` nhỏ, label `app: <tên>`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã sửa repo và tạo ba Deployment thử nghiệm</summary>

```bash
NS=bookstore; EV=~/lab-evidence/7a; SEL='app.kubernetes.io/part-of=bookstore'
WEB="$SEL,app.kubernetes.io/component=web"; API="$SEL,app.kubernetes.io/component=api"
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
nodes() { kubectl -n "$NS" get pods -l "$1" --field-selector=status.phase=Running \
            -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}'; }
on() { nodes "$1" | grep -cx "$2"; }
cnt() { kubectl -n "$NS" get pods -l "$1" --field-selector=status.phase="$2" --no-headers 2>/dev/null | wc -l; }

./scripts/deploy.sh > "$EV/b3-deploy.txt" 2>&1
A="$(on "$WEB" lab-k8s-worker1)"; B="$(on "$WEB" lab-k8s-worker2)"
echo "web: worker1=$A worker2=$B" | tee "$EV/b3-web.txt"
{ [ "$A" -ge 1 ] && [ "$B" -ge 1 ] && [ $((A + B)) -eq 3 ]; } && echo "PASS: web 3/3 tren hai worker ($A/$B)"
C='{.spec.template.spec.topologySpreadConstraints[0]'
{ [ "$(jp deployment/web "$C.topologyKey}")" = 'kubernetes.io/hostname' ] && \
  [ "$(jp deployment/web "$C.whenUnsatisfiable}")" = 'DoNotSchedule' ] && \
  [ "$(jp deployment/web "$C.maxSkew}")" = '1' ]; } && echo 'PASS: rang buoc cung theo node, maxSkew 1'
LS="$(jp deployment/web "$C.labelSelector.matchLabels}" \
      | python3 -c 'import json,sys; print(",".join(k+"="+v for k,v in json.load(sys.stdin).items()))' 2>/dev/null)"
{ [ -n "$LS" ] && [ "$(kubectl -n "$NS" get pods -l "$LS" --no-headers | wc -l)" -eq \
                    "$(kubectl -n "$NS" get pods -l "$WEB" --no-headers | wc -l)" ]; } \
  && echo "PASS: selector dem ($LS) chon dung cac Pod web — khong co pod ma"

kubectl -n "$NS" scale deployment web --replicas=5
kubectl -n "$NS" rollout status deployment/web --timeout=180s >/dev/null
A="$(on "$WEB" lab-k8s-worker1)"; B="$(on "$WEB" lab-k8s-worker2)"
echo "web x5: worker1=$A worker2=$B" | tee -a "$EV/b3-web.txt"
{ [ $((A + B)) -eq 5 ] && [ $((A - B)) -le 1 ] && [ $((B - A)) -le 1 ]; } && echo "PASS: 5 ban sao chia $A/$B"
./scripts/deploy.sh >> "$EV/b3-deploy.txt" 2>&1
[ "$(jp deployment/web '{.status.readyReplicas}')" = '3' ] && echo 'PASS: deploy.sh dua web ve 3 ban sao'
echo "web sau khi thu ve 3: worker1=$(on "$WEB" lab-k8s-worker1) worker2=$(on "$WEB" lab-k8s-worker2)" \
  | tee -a "$EV/b3-web.txt"

AN="$(nodes "$API" | sort -u | wc -l)"
P='{.spec.template.spec.affinity.podAntiAffinity'
{ [ "$(jp deployment/api '{.status.readyReplicas}')" = '2' ] && [ "$AN" -eq 2 ] && \
  [ -n "$(jp deployment/api "$P.preferredDuringSchedulingIgnoredDuringExecution}")" ] && \
  [ -z "$(jp deployment/api "$P.requiredDuringSchedulingIgnoredDuringExecution}")" ]; } \
  && echo 'PASS: api 2/2 tren hai node, anti-affinity mem'

for i in $(seq 1 40); do
  [ "$(cnt app=chong-don-cung Running)" -eq 2 ] && [ "$(cnt app=chong-don-cung Pending)" -eq 1 ] && \
  [ "$(cnt app=chong-don-mem Running)" -eq 3 ] && [ "$(cnt app=sai-khoa Pending)" -eq 2 ] && break
  sleep 3
done
kubectl -n "$NS" get pods -l 'app in (chong-don-cung,chong-don-mem,sai-khoa)' -o wide | tee "$EV/b3-thu-nghiem.txt"
msg() { kubectl -n "$NS" get pods -l "app=$1" --field-selector=status.phase=Pending \
          -o jsonpath='{.items[0].status.conditions[?(@.type=="PodScheduled")].message}'; }
msg chong-don-cung > "$EV/b3-chong-don-cung.txt"
{ [ "$(cnt app=chong-don-cung Running)" -eq 2 ] && [ "$(cnt app=chong-don-cung Pending)" -eq 1 ] && \
  grep -q 'anti-affinity' "$EV/b3-chong-don-cung.txt"; } && echo 'PASS: anti-affinity cung: 2 chay, 1 cho'
{ [ "$(cnt app=chong-don-mem Running)" -eq 3 ] && [ "$(nodes app=chong-don-mem | sort -u | wc -l)" -eq 2 ]; } \
  && echo 'PASS: anti-affinity mem: ca 3 chay, trai tren hai node'
msg sai-khoa > "$EV/b3-sai-khoa.txt"
{ [ "$(cnt app=sai-khoa Running)" -eq 0 ] && [ "$(cnt app=sai-khoa Pending)" -eq 2 ] && \
  grep -q 'topology spread' "$EV/b3-sai-khoa.txt"; } && echo 'PASS: sai-khoa: moi Pod cho vi rang buoc phan bo'

kubectl -n "$NS" delete deployment chong-don-cung chong-don-mem sai-khoa --ignore-not-found --wait=true >/dev/null
[ -z "$(kubectl -n "$NS" get deployment -o name | grep -E 'chong-don|sai-khoa')" ] && echo 'PASS: da xoa Deployment thu nghiem'
```

Dòng cuối của khối scale chỉ **ghi lại** phân bố sau khi thu về 3 bản sao, không kiểm: bài 140,
mục *Các hạn chế đã biết*, nói scale down không bảo đảm phân bố còn cân bằng.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/web/deployment.yaml — thêm vào spec.template.spec
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app.kubernetes.io/part-of: bookstore
            app.kubernetes.io/component: web
```

```yaml
# k8s/app/api/deployment.yaml — thêm vào spec.template.spec
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels:
                  app.kubernetes.io/part-of: bookstore
                  app.kubernetes.io/component: api
```

```bash
dep() {   # $1 = tên, $2 = số bản sao, $3 = khối ràng buộc lập lịch đặt trong spec.template.spec
  cat > ~/lab-work/7a/$1.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata: {name: $1, namespace: bookstore}
spec:
  replicas: $2
  selector: {matchLabels: {app: $1}}
  template:
    metadata: {labels: {app: $1}}
    spec:
      terminationGracePeriodSeconds: 5
$3
      containers:
      - {name: app, image: "busybox:1.37", command: ["sh", "-c", "sleep 3600"], resources: {requests: {cpu: 10m, memory: 16Mi}}}
EOF
}
dep chong-don-cung 3 '      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector: {matchLabels: {app: chong-don-cung}}'
dep chong-don-mem 3 '      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector: {matchLabels: {app: chong-don-mem}}'
dep sai-khoa 2 '      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: bookstore.example.com/zon     # cố ý gõ sai: không node nào có key này
        whenUnsatisfiable: DoNotSchedule
        labelSelector: {matchLabels: {app: sai-khoa}}'

kubectl apply -f ~/lab-work/7a/chong-don-cung.yaml -f ~/lab-work/7a/chong-don-mem.yaml -f ~/lab-work/7a/sai-khoa.yaml
```

Ý chính cho ADR:

1. 3 bản sao → 2/1; 5 bản sao → 3/2. Độ lệch là số Pod khớp ở miền đích **sau khi** đặt trừ mức
   tối thiểu toàn cục — cách bài 140 tính ví dụ `[3, 1]`. Từ 2/1, Pod thứ tư vào node đang có 2 thì
   `3 - 1 = 2 > 1`, bị loại; vào node có 1 thì `2 - 1 = 1`, hợp lệ → 2/2. Pod thứ năm vào node nào
   cũng `3 - 2 = 1` → 3/2. Muốn có 4/1 thì phải đặt Pod vào node đang nhiều hơn — luôn lệch quá 1.
2. Anti-affinity cứng theo node nghĩa là **mỗi node tối đa một bản sao**: `chong-don-cung` có Pod thứ
   ba nằm chờ. Với `web` 3–5 bản sao trên hai worker, một nửa số bản sao sẽ không chạy; Pod dôi ra
   lúc rollout cũng không có chỗ. Bài 140 gọi topology spread là "kiểm soát tinh vi hơn": mức lệch
   định lượng được, vẫn cưỡng chế được bằng `DoNotSchedule`.
3. Dạng mềm thì khi một worker không nhận Pod, bản sao thay thế vẫn lên worker còn lại — `api` giữ đủ
   năng lực. Dạng cứng thì bản thay thế nằm chờ, `api` mất một nửa năng lực đúng lúc bảo trì, và Pod
   tạm của rollout cũng không có chỗ. Cái giá của dạng mềm: mất khả năng cưỡng chế, nên hai bản sao
   **có thể** cùng một node — B7 cho thấy khi nào.
4. Bài 140, *Các quy ước ngầm định*: scheduler chỉ xét node có **đủ mọi** `topologyKey`; node thiếu
   key bị bỏ qua và Pod mới không có cơ hội lên đó. Không node nào có key thì mọi Pod nằm chờ dù
   node vẫn `Ready` và còn tài nguyên — đúng bẫy `zone-typo` của bài. Trên `kubectl get nodes -L` nó
   hiện ra là một cột rỗng.

</details>

## B4. [BS-7A-04] Tác vụ nền nhường chỗ cho dịch vụ chính

**Loại:** Tính năng.

**Bối cảnh:** `db-migrate`, `db-backup` đang chạy với độ ưu tiên 0 — ngang `web` — và có thể rơi vào
`lab-k8s-worker2`, nơi `db` cần tài nguyên ổn định. Hợp đồng
([mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label)) dành tên PriorityClass `bookstore-batch` cho tác vụ
nền từ sprint này; [mục 3.6](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch) chốt Job và
CronJob dùng nó.

**Yêu cầu:**

1. PriorityClass `bookstore-batch` ở `k8s/cluster/priorityclass-batch.yaml`, label theo hợp đồng,
   **không** phải giá trị mặc định toàn cluster, có mô tả. Chọn giá trị sao cho Pod batch xếp hàng
   **sau mọi Pod không khai PriorityClass** (như `web` và phần lớn Pod hạ tầng) và **không bao giờ**
   chiếm chỗ của chúng.
2. Pod của `db-migrate` và của `db-backup` dùng `bookstore-batch`.
3. Pod batch **ưu tiên** node không mang label dữ liệu, nhưng vẫn chạy được trên node dữ liệu khi cần
   — ví dụ khi volume của nó nằm ở đó.
4. Chứng minh quy tắc ở yêu cầu 3 là mềm, bằng hai Pod thử nghiệm mang đúng quy tắc lập lịch của tác
   vụ nền: `uu-tien-thu` (chỉ quy tắc đó) và `uu-tien-ep` (thêm ràng buộc cứng sang
   `lab-k8s-worker2`).
5. Đọc: add-on của cluster dùng PriorityClass hệ thống nào; xếp mọi giá trị đang có từ cao xuống thấp.

**Ràng buộc:** không đổi `bookstore-critical`. Không dùng `preemptionPolicy` — bài 141 để mục đó cho
giai đoạn 13. Không đặt `spec.priority` trực tiếp.

**Đọc lại:**

- Bài [141](../141-pod-priority-preemption-vi.md): [*PriorityClass*](../141-pod-priority-preemption-vi.md#priorityclass),
  [*Độ ưu tiên của Pod*](../141-pod-priority-preemption-vi.md#pod-priority),
  [*Ảnh hưởng của độ ưu tiên Pod đến thứ tự lập lịch*](../141-pod-priority-preemption-vi.md#ảnh-hưởng-của-độ-ưu-tiên-pod-đến-thứ-tự-lập-lịch-effect-of-pod-priority-on-scheduling-order),
  [*Preemption*](../141-pod-priority-preemption-vi.md#preemption).
- Bài [210](../210-guaranteed-scheduling-critical-addon-pods-vi.md#marking-pod-as-critical) — cả bài,
  rất ngắn.
- Bài [138](../138-assign-pod-node-vi.md#trọng-số-node-affinity-node-affinity-weight), mục *Trọng số
  node affinity*, và mục [*Toán tử*](../138-assign-pod-node-vi.md#operators).
- Bài [266](../266-assign-pods-nodes-node-affinity-vi.md#lên-lịch-một-pod-bằng-node-affinity-ưu-tiên-schedule-a-pod-using-preferred-node-affinity)
  — *Lên lịch một Pod bằng node affinity ưu tiên*.

**Bàn giao:** `k8s/cluster/priorityclass-batch.yaml`; manifest của Job `db-migrate` và CronJob
`db-backup`; `docs/adr/NNNN-priorityclass-batch.md`; `~/lab-evidence/7a/b4-*.txt`.

**Nghiệm thu:**

- `PASS:` `deploy.sh` chạy hai lần liên tiếp thành công.
- `PASS:` `bookstore-batch` mang label dự án, không `globalDefault`, giá trị âm.
- `PASS:` Pod của `db-migrate` và Pod của một lần chạy `db-backup` mang `spec.priority` bằng giá trị
  của `bookstore-batch`.
- `PASS:` quy tắc node của cả Job lẫn CronJob là `preferred`, không có `required`.
- `PASS:` `uu-tien-thu` lên `lab-k8s-worker1`; `uu-tien-ep` vẫn chạy trên `lab-k8s-worker2`.
- `PASS:` CoreDNS dùng `system-cluster-critical`, kube-proxy dùng `system-node-critical`; thứ tự:
  `system-node-critical` > `system-cluster-critical` > 1 tỷ ≥ `bookstore-critical` > 0 >
  `bookstore-batch`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-priorityclass-batch.md`:

1. Vì sao giá trị âm? Một Pod `bookstore-batch` có thể chiếm chỗ của ai? Nếu chọn giá trị dương (ví
   dụ `1000`), chuyện gì xảy ra với `web` khi node chật?
2. Bài 141 xếp `preemptionPolicy: Never` cho workload kiểu batch. Vì sao với giá trị âm, dự án chưa
   cần trường đó?
3. `db-backup` ghi vào PVC `backup-store` — volume `local-path` gắn với đúng một node. Quy tắc "ưu
   tiên node không phải node dữ liệu" còn tác dụng gì nếu PV đó nằm trên worker2? Vì sao `preferred`
   đúng ở đây còn `required` thì nguy hiểm?
4. Bài 210: CoreDNS dùng PriorityClass nào? `bookstore-critical` có chiếm chỗ được CoreDNS không? Đánh
   dấu critical bảo đảm gì và **không** bảo đảm gì?

<details>
<summary>Gợi ý</summary>

1. Khoảng giá trị hợp lệ của PriorityClass ở mục *PriorityClass* của bài 141 — có cả số âm.
2. Đường dẫn trong manifest: Job là `spec.template.spec`; CronJob là
   `spec.jobTemplate.spec.template.spec`.
3. `operator: DoesNotExist` không cần `values`. `NotIn ["true"]` cũng được — node không có key được
   coi là không nằm trong tập.
4. Đổi pod template của CronJob không ảnh hưởng Job đã tạo trước đó; chạy thử bằng
   `kubectl -n bookstore create job <tên> --from=cronjob/db-backup`.
5. Thứ tự giá trị: `kubectl get priorityclass --sort-by=.value`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong và đã tạo hai Pod thử nghiệm</summary>

```bash
NS=bookstore; EV=~/lab-evidence/7a
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
pc() { kubectl get priorityclass "$1" -o jsonpath="$2" 2>/dev/null; }

./scripts/deploy.sh > "$EV/b4-deploy-1.txt" 2>&1 && ./scripts/deploy.sh > "$EV/b4-deploy-2.txt" 2>&1 \
  && echo 'PASS: deploy.sh chay hai lan lien tiep thanh cong'
V="$(pc bookstore-batch '{.value}')"
{ [ "$(pc bookstore-batch '{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ] && \
  [ "$(pc bookstore-batch '{.globalDefault}')" != 'true' ] && [ "${V:-0}" -lt 0 ]; } \
  && echo "PASS: bookstore-batch mang label, khong globalDefault, value=$V"

MP="$(kubectl -n "$NS" get pods -l job-name=db-migrate -o jsonpath='{.items[0].spec.priority}')"
J="b4-backup-$(date +%s)"
kubectl -n "$NS" create job "$J" --from=cronjob/db-backup >/dev/null
kubectl -n "$NS" wait --for=condition=Complete "job/$J" --timeout=180s >/dev/null
BP="$(kubectl -n "$NS" get pods -l job-name="$J" -o jsonpath='{.items[0].spec.priority}')"
echo "db-migrate priority=$MP ; db-backup priority=$BP" | tee "$EV/b4-priority.txt"
{ [ "$MP" = "$V" ] && [ "$BP" = "$V" ]; } && echo 'PASS: Pod db-migrate va db-backup mang priority bookstore-batch'
kubectl -n "$NS" delete job "$J" --wait=true >/dev/null

for O in job/db-migrate cronjob/db-backup; do
  case "$O" in job/*) A='{.spec.template.spec.affinity.nodeAffinity' ;;
               *)     A='{.spec.jobTemplate.spec.template.spec.affinity.nodeAffinity' ;; esac
  { [ -n "$(jp "$O" "$A.preferredDuringSchedulingIgnoredDuringExecution}")" ] && \
    [ -z "$(jp "$O" "$A.requiredDuringSchedulingIgnoredDuringExecution}")" ]; } \
    && echo "PASS: $O uu tien node, khong ep node"
done

kubectl apply -f ~/lab-work/7a/uu-tien-thu.yaml -f ~/lab-work/7a/uu-tien-ep.yaml >/dev/null
kubectl -n "$NS" wait --for=condition=Ready pod/uu-tien-thu pod/uu-tien-ep --timeout=120s >/dev/null
{ [ "$(jp pod/uu-tien-thu '{.spec.nodeName}')" = 'lab-k8s-worker1' ] && \
  [ "$(jp pod/uu-tien-ep '{.spec.nodeName}')" = 'lab-k8s-worker2' ]; } \
  && echo 'PASS: quy tac uu tien keo Pod ve worker1 nhung khong chan Pod bi ep sang worker2'
kubectl -n "$NS" delete pod uu-tien-thu uu-tien-ep --wait=true >/dev/null

DNS="$(kubectl -n kube-system get deployment coredns -o jsonpath='{.spec.template.spec.priorityClassName}')"
KP="$(kubectl -n kube-system get daemonset kube-proxy -o jsonpath='{.spec.template.spec.priorityClassName}')"
NC="$(pc system-node-critical '{.value}')"; CC="$(pc system-cluster-critical '{.value}')"
BC="$(pc bookstore-critical '{.value}')"
kubectl get priorityclass --sort-by=.value -o custom-columns='NAME:.metadata.name,VALUE:.value' | tee "$EV/b4-thu-tu.txt"
{ [ "$DNS" = 'system-cluster-critical' ] && [ "$KP" = 'system-node-critical' ] && \
  [ "$NC" -gt "$CC" ] && [ "$CC" -gt 1000000000 ] && [ "$BC" -le 1000000000 ] && \
  [ "$BC" -gt 0 ] && [ "$V" -lt 0 ]; } \
  && echo 'PASS: node-critical > cluster-critical > 1 ty >= bookstore-critical > 0 > bookstore-batch'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/cluster/priorityclass-batch.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: bookstore-batch
  labels:
    app.kubernetes.io/part-of: bookstore
value: -1000
globalDefault: false
description: "Tac vu nen cua Bookstore (Job, CronJob): xep sau moi Pod mac dinh, la nan nhan dau tien"
```

Thêm vào pod template của Job `db-migrate` (`spec.template.spec`) và CronJob `db-backup`
(`spec.jobTemplate.spec.template.spec`):

```yaml
      priorityClassName: bookstore-batch
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: bookstore.example.com/data
                operator: DoesNotExist
```

```yaml
# ~/lab-work/7a/uu-tien-thu.yaml — bản uu-tien-ep.yaml đổi tên và thêm
#   nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
apiVersion: v1
kind: Pod
metadata: {name: uu-tien-thu, namespace: bookstore}
spec:
  priorityClassName: bookstore-batch
  terminationGracePeriodSeconds: 5
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: bookstore.example.com/data
            operator: DoesNotExist
  containers:
  - {name: app, image: "busybox:1.37", command: ["sh", "-c", "sleep 3600"], resources: {requests: {cpu: 10m, memory: 16Mi}}}
```

```bash
cd ~/bookstore && ./scripts/deploy.sh
kubectl get priorityclass --sort-by=.value
```

Ý chính cho ADR:

1. Pod không khai PriorityClass có độ ưu tiên 0 (bài 141). Giá trị âm đặt tác vụ nền **dưới** mọi Pod
   như thế: trong hàng đợi chúng đứng sau, và khi cần chỗ chúng là nạn nhân đầu tiên. Ngược lại một Pod
   batch chỉ chiếm chỗ được Pod có độ ưu tiên **thấp hơn** nó — không có Pod nào như vậy, nên nó không
   bao giờ đuổi ai. Nếu dương (`1000`), một Job đang chờ có thể **chiếm chỗ của `web`** (độ ưu tiên 0)
   khi node chật — đúng điều R10 cấm.
2. `preemptionPolicy: Never` dành cho Pod muốn **xếp trước** nhưng không đuổi ai. `bookstore-batch`
   không muốn xếp trước; với giá trị thấp nhất, logic preemption của nó không bao giờ tìm được nạn nhân.
3. Volume `local-path` mang `nodeAffinity` (B2): nếu `backup-store` nằm trên worker2 thì Pod backup
   **phải** lên worker2. Quy tắc `preferred` chỉ cộng điểm nên vẫn nhường cho ràng buộc cứng của
   volume. Nếu viết `required` ("không được lên node dữ liệu"), hai ràng buộc cứng mâu thuẫn và Pod
   backup nằm chờ mãi.
4. CoreDNS dùng `system-cluster-critical`, giá trị lớn hơn 1 tỷ — trần của PriorityClass do người
   dùng tạo — nên `bookstore-critical` không bao giờ chiếm chỗ được nó. Bài 210: đánh dấu critical
   **không** ngăn trục xuất hoàn toàn; nó chỉ ngăn Pod bị mất vĩnh viễn — Pod critical không phải
   static luôn được lập lịch lại.

</details>

## B5. [BS-7A-05] Diễn tập: preemption trên `lab-k8s-worker2`

**Loại:** Diễn tập sự cố — chỉ trên `lab-k8s-worker2`.

**Bối cảnh:** B4 mới **khai báo** thứ tự ưu tiên. Phải thấy nó hoạt động khi node thật sự chật. Kịch
bản: một đợt tác vụ nền nặng lấp đầy worker2; đúng lúc đó một thành phần critical cần chỗ trên chính
worker2 — node duy nhất có dữ liệu.

**Yêu cầu:**

1. Tính chỗ trống CPU của `lab-k8s-worker2` từ `Allocatable` và tổng `requests` đang có trên node —
   ghi phép tính vào postmortem. Không đoán con số.
2. Lấp worker2 bằng Job thử nghiệm `lap-cho`: `bookstore-batch`, 2 Pod chạy song song, mỗi Pod xin
   khoảng 45% chỗ trống, ghim worker2, và tắt chậm khi nhận tín hiệu dừng.
3. Tạo Pod thử nghiệm `can-cho-gap`: `bookstore-critical`, xin khoảng 50% chỗ trống, ghim worker2.
   Chứng minh: scheduler đề cử worker2 cho Pod này; **đúng một** Pod `lap-cho` bị chiếm chỗ, và cả sự
   kiện lẫn condition của Pod đó nói rõ lý do; `can-cho-gap` chạy trên worker2.
4. Chứng minh không Pod nào khác trên worker2 — của Bookstore hay của hạ tầng — bị chọn làm nạn nhân.
5. Pod `lap-cho` thay thế do Job tạo lại nằm chờ và không chiếm chỗ được ai: công việc nền bị
   **hoãn**, không mất.
6. ⭐ Lặp lại với một PDB che hai Pod `lap-cho`: preemption vẫn xảy ra.
7. Dọn sạch, viết postmortem.

**Ràng buộc:** chỉ worker2. Không sửa repo. Không đổi `Allocatable`, không tạo Pod lớn trên node
khác.

**Đọc lại:**

- Bài [141](../141-pod-priority-preemption-vi.md): [*Preemption*](../141-pod-priority-preemption-vi.md#preemption),
  [*Thông tin hiển thị cho người dùng*](../141-pod-priority-preemption-vi.md#thông-tin-hiển-thị-cho-người-dùng-user-exposed-information),
  [*Chấm dứt nhẹ nhàng cho các nạn nhân của preemption*](../141-pod-priority-preemption-vi.md#chấm-dứt-nhẹ-nhàng-cho-các-nạn-nhân-của-preemption-graceful-termination-of-preemption-victims),
  [*PodDisruptionBudget được hỗ trợ, nhưng không được đảm bảo*](../141-pod-priority-preemption-vi.md#poddisruptionbudget-được-hỗ-trợ-nhưng-không-được-đảm-bảo-poddisruptionbudget-is-supported-but-not-guaranteed),
  và ghi chú "không nhất thiết loại bỏ tất cả" trong mục
  [*Inter-Pod affinity trên các Pod có độ ưu tiên thấp hơn*](../141-pod-priority-preemption-vi.md#inter-pod-affinity-trên-các-pod-có-độ-ưu-tiên-thấp-hơn-inter-pod-affinity-on-lower-priority-pods).
- Bài [147](../147-scheduling-framework-vi.md#post-filter), mục *PostFilter*.
- Bài [53](../53-disruptions-vi.md#pod-disruption-conditions) (nhóm 3c), mục *Các condition gián đoạn
  của Pod* — giá trị `reason` của `DisruptionTarget`.

**Bàn giao:** `docs/postmortem/7a-preemption-worker2.md`; `~/lab-evidence/7a/b5-*.txt`.

**Nghiệm thu:**

- `PASS:` phép tính chỗ trống của worker2 nằm trong bằng chứng, và chỗ trống đủ (ít nhất `400m`) để
  dựng tình huống.
- `PASS:` hai Pod `lap-cho` chạy trên worker2 với độ ưu tiên của `bookstore-batch`.
- `PASS:` `can-cho-gap` được đề cử `lab-k8s-worker2` (`nominatedNodeName`) rồi chạy trên worker2.
- `PASS:` nạn nhân mang condition `DisruptionTarget` lý do `PreemptionByScheduler`; đúng một Pod
  `lap-cho` có sự kiện `Preempted`.
- `PASS:` UID của mọi Pod khác trên worker2 không đổi.
- `PASS:` Pod `lap-cho` thay thế nằm `Pending` vì thiếu CPU; Job vẫn còn Pod đang hoạt động.
- `PASS:` (⭐) với PDB, preemption vẫn xảy ra.
- `PASS:` Job `lap-cho`, Pod `can-cho-gap` (và PDB ⭐) đã bị xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/postmortem/7a-preemption-worker2.md`:

1. Vì sao nạn nhân là `lap-cho` mà không phải Pod `web` hay `node-agent` trên worker2, dù chúng cũng
   thấp hơn `bookstore-critical`? Vì sao chỉ **một** Pod `lap-cho`?
2. `nominatedNodeName` có phải lời hứa không? Khoảng trễ giữa lúc đề cử và lúc `can-cho-gap` chạy do
   đâu mà có, phụ thuộc vào gì?
3. Giả sử worker2 không có Pod batch nào, chỉ có `web`, `node-agent` và `db-0`; một Pod critical
   (chẳng hạn `db-0` vừa bị thay) cần chỗ. Ai bị chiếm chỗ? Điều đó nói gì về việc `web` không mang
   PriorityClass nào?
4. Pod `lap-cho` thay thế không chiếm chỗ được ai. Vì sao? Job đó tiếp tục khi nào?

<details>
<summary>Gợi ý</summary>

1. Tổng `requests` trên node nằm ở phần *Allocated resources* của `kubectl describe node`.
2. `trap '' TERM; while true; do sleep 1; done` cộng `terminationGracePeriodSeconds: 30` giữ Pod bị
   chiếm chỗ sống thêm một lúc — đủ để bạn kịp đọc condition và `nominatedNodeName`.
3. `status.nominatedNodeName` nằm trên Pod **đang chờ**, không nằm trên nạn nhân. Sự kiện:
   `kubectl -n bookstore get events --field-selector reason=Preempted`.
4. Job song song: `parallelism: 2`, `completions: 2`.

</details>

<details>
<summary>Script nghiệm thu — script này chính là cuộc diễn tập</summary>

```bash
NS=bookstore; EV=~/lab-evidence/7a; W2=lab-k8s-worker2; LW=~/lab-work/7a
cpu2m() { case "$1" in '') echo 0 ;; *m) echo "${1%m}" ;; *) awk -v x="$1" 'BEGIN{printf "%d\n", x*1000}' ;; esac; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
uids() {   # Pod đang chạy trên worker2, trừ Pod của cuộc diễn tập và Pod do Job/CronJob tự sinh
  kubectl get pods -A --field-selector spec.nodeName=$W2,status.phase=Running \
    -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}={.metadata.uid}{"\n"}{end}' \
    | grep -v -e '^bookstore/lap-cho-' -e '^bookstore/can-cho-gap=' -e '^bookstore/db-backup-' -e '^bookstore/db-migrate-' \
    | sort
}

kubectl -n "$NS" delete job lap-cho --ignore-not-found --wait=true >/dev/null
kubectl -n "$NS" delete pod can-cho-gap --ignore-not-found --wait=true >/dev/null
ALLOC="$(cpu2m "$(kubectl get node $W2 -o jsonpath='{.status.allocatable.cpu}')")"
USED="$(cpu2m "$(kubectl describe node $W2 | awk '/^Allocated resources/,/^Events/' | awk '$1=="cpu"{print $2; exit}')")"
FREE=$((ALLOC - USED)); LOW=$((FREE * 45 / 100)); HIGH=$((FREE * 50 / 100))
echo "worker2: allocatable=${ALLOC}m requested=${USED}m free=${FREE}m | lap-cho=${LOW}m x2 | can-cho-gap=${HIGH}m" \
  | tee "$EV/b5-so-lieu.txt"
[ "$FREE" -ge 400 ] && echo 'PASS: worker2 du cho trong de dung tinh huong'
uids > "$EV/b5-truoc.txt"

cat > "$LW/lap-cho.yaml" <<EOF
apiVersion: batch/v1
kind: Job
metadata: {name: lap-cho, namespace: bookstore}
spec:
  parallelism: 2
  completions: 2
  backoffLimit: 6
  template:
    metadata:
      labels: {app: lap-cho}
    spec:
      priorityClassName: bookstore-batch
      restartPolicy: Never
      terminationGracePeriodSeconds: 30
      nodeSelector: {kubernetes.io/hostname: $W2}
      containers:
      - name: lam
        image: busybox:1.37
        command: ["sh", "-c", "trap '' TERM; while true; do sleep 1; done"]
        resources:
          requests: {cpu: "${LOW}m", memory: 16Mi}
EOF
cat > "$LW/can-cho-gap.yaml" <<EOF
apiVersion: v1
kind: Pod
metadata: {name: can-cho-gap, namespace: bookstore}
spec:
  priorityClassName: bookstore-critical
  terminationGracePeriodSeconds: 5
  nodeSelector: {kubernetes.io/hostname: $W2}
  containers:
  - {name: app, image: "busybox:1.37", command: ["sh", "-c", "sleep 3600"], resources: {requests: {cpu: "${HIGH}m", memory: 16Mi}}}
EOF

kubectl apply -f "$LW/lap-cho.yaml" >/dev/null
for i in $(seq 1 40); do
  [ "$(kubectl -n "$NS" get pods -l app=lap-cho --field-selector=status.phase=Running --no-headers 2>/dev/null | wc -l)" -eq 2 ] && break
  sleep 3
done
V="$(kubectl get priorityclass bookstore-batch -o jsonpath='{.value}')"
{ [ "$(kubectl -n "$NS" get pods -l app=lap-cho --field-selector=status.phase=Running --no-headers | wc -l)" -eq 2 ] && \
  [ "$(kubectl -n "$NS" get pods -l app=lap-cho -o jsonpath='{.items[0].spec.priority}')" = "$V" ]; } \
  && echo 'PASS: hai Pod lap-cho giu gan het cho trong cua worker2'
LAP0="$(kubectl -n "$NS" get pods -l app=lap-cho -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}')"

kubectl apply -f "$LW/can-cho-gap.yaml" >/dev/null
NOM=''; VIC=''; DT=''
for i in $(seq 1 30); do
  [ -z "$NOM" ] && NOM="$(jp pod/can-cho-gap '{.status.nominatedNodeName}')"
  [ -z "$VIC" ] && VIC="$(kubectl -n "$NS" get pods -l app=lap-cho \
        -o jsonpath='{range .items[?(@.metadata.deletionTimestamp)]}{.metadata.name}{"\n"}{end}' | head -1)"
  [ -n "$VIC" ] && [ -z "$DT" ] && DT="$(jp "pod/$VIC" '{.status.conditions[?(@.type=="DisruptionTarget")].reason}')"
  [ -n "$NOM" ] && [ -n "$DT" ] && break
  sleep 1
done
echo "nominatedNodeName=$NOM ; nan nhan=$VIC ; DisruptionTarget=$DT" | tee -a "$EV/b5-so-lieu.txt"
[ "$NOM" = "$W2" ] && echo 'PASS: scheduler de cu worker2 cho can-cho-gap'
[ "$DT" = 'PreemptionByScheduler' ] && echo 'PASS: nan nhan mang DisruptionTarget=PreemptionByScheduler'

kubectl -n "$NS" wait --for=condition=Ready pod/can-cho-gap --timeout=180s >/dev/null
[ "$(jp pod/can-cho-gap '{.spec.nodeName}')" = "$W2" ] && echo 'PASS: can-cho-gap chay tren worker2'
kubectl -n "$NS" get events --field-selector reason=Preempted \
  -o custom-columns='POD:.involvedObject.name,MSG:.message' > "$EV/b5-preempted.txt"
N_PRE="$(for P in $LAP0; do grep -c "^$P " "$EV/b5-preempted.txt"; done | awk '{s+=$1} END{print s+0}')"
[ "$N_PRE" -eq 1 ] && echo 'PASS: dung mot Pod lap-cho bi chiem cho'

uids > "$EV/b5-sau.txt"
diff "$EV/b5-truoc.txt" "$EV/b5-sau.txt" >/dev/null && echo 'PASS: khong Pod nao khac tren worker2 bi chon lam nan nhan'

PEND=''
for i in $(seq 1 20); do
  PEND="$(kubectl -n "$NS" get pods -l app=lap-cho --field-selector=status.phase=Pending -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
  [ -n "$PEND" ] && break
  sleep 3
done
sleep 15
jp "pod/$PEND" '{.status.conditions[?(@.type=="PodScheduled")].message}' > "$EV/b5-thay-the.txt"
N_ACT="$(jp job/lap-cho '{.status.active}')"
{ [ -n "$PEND" ] && [ "$(jp "pod/$PEND" '{.status.phase}')" = 'Pending' ] && \
  grep -q 'Insufficient cpu' "$EV/b5-thay-the.txt" && [ "${N_ACT:-0}" -ge 1 ]; } \
  && echo "PASS: Pod thay the $PEND nam cho, Job lap-cho van con $N_ACT Pod hoat dong"

kubectl -n "$NS" delete job lap-cho --wait=true >/dev/null
kubectl -n "$NS" delete pod can-cho-gap --wait=true >/dev/null
[ -z "$(kubectl -n "$NS" get pods -l app=lap-cho -o name)" ] && echo 'PASS: da don Job lap-cho va can-cho-gap'
```

⭐ Lặp lại với PDB — chạy trong cùng shell sau khối trên (dùng lại `LW`, `NS`, `jp`):

```bash
kubectl apply -f "$LW/lap-cho.yaml" >/dev/null
kubectl -n "$NS" create poddisruptionbudget lap-cho --selector=app=lap-cho --min-available=2 >/dev/null
for i in $(seq 1 40); do [ "$(jp pdb/lap-cho '{.status.currentHealthy}')" = '2' ] && break; sleep 3; done
echo "PDB lap-cho: disruptionsAllowed=$(jp pdb/lap-cho '{.status.disruptionsAllowed}')"
kubectl apply -f "$LW/can-cho-gap.yaml" >/dev/null
kubectl -n "$NS" wait --for=condition=Ready pod/can-cho-gap --timeout=180s >/dev/null \
  && echo 'PASS: preemption van xay ra du PDB khong cho phep gian doan nao'
kubectl -n "$NS" delete pdb lap-cho --wait=true >/dev/null
kubectl -n "$NS" delete job lap-cho --wait=true >/dev/null
kubectl -n "$NS" delete pod can-cho-gap --wait=true >/dev/null
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của postmortem</summary>

1. Logic preemption tìm node mà việc gỡ các Pod **thấp hơn** Pod đang chờ sẽ đủ chỗ, rồi chọn tập
   nạn nhân có độ ưu tiên **thấp nhất** và **nhỏ nhất** có thể (bài 141: preemption không nhất thiết
   loại bỏ tất cả). `lap-cho` ở `-1000`, thấp hơn `web`, `node-agent` và Pod hạ tầng (0 trở lên), nên
   chúng được giữ lại trước. Gỡ một `lap-cho` là đã trả lại 45% chỗ trống, cộng 10% còn dư là đủ cho
   50% của `can-cho-gap` — Pod `lap-cho` thứ hai không cần đi.
2. Không. Bài 141: `nominatedNodeName` giúp scheduler giữ chỗ và cho người dùng thấy preemption; Pod
   **không chắc** lên đúng node đó — node khác có thể trống ra trước, hoặc một Pod ưu tiên cao hơn tới
   giành. Khoảng trễ là thời gian nạn nhân dùng grace period của nó: ở đây `lap-cho` bỏ qua `SIGTERM`
   nên dùng trọn 30 giây. Khoảng trễ này **phụ thuộc cấu hình** grace period của nạn nhân.
3. Khi đó nạn nhân là `web` hoặc `node-agent` (độ ưu tiên 0) — dịch vụ chính bị đuổi để chỗ cho
   critical. Điều này chấp nhận được với `db`, nhưng cho thấy `web` không có lớp bảo vệ nào; đó là lý
   do tác vụ nền phải ở **dưới** `web` chứ không ngang hàng.
4. Một Pod chỉ chiếm chỗ được Pod có độ ưu tiên thấp hơn nó; `bookstore-batch` là thấp nhất cluster.
   Thông điệp của Pod chờ có `Insufficient cpu` kèm phần preemption không tìm được nạn nhân. Job tiếp
   tục khi worker2 có chỗ một cách tự nhiên — ở đây là khi `can-cho-gap` bị xóa.

Với ⭐: PDB `minAvailable: 2` trên đúng 2 Pod nên ngân sách bằng 0; scheduler cố tìm nạn nhân không vi
phạm PDB, không có, và **vẫn** chiếm chỗ. So với Eviction API ở B8 — nơi PDB chặn đứng yêu cầu — PDB
chỉ là nỗ lực tốt nhất trước preemption.

</details>

## B6. [BS-7A-06] Báo cáo tồn kho chờ duyệt

**Loại:** Tính năng.

**Bối cảnh:** phòng kinh doanh muốn một báo cáo tồn kho chạy theo yêu cầu: số đầu sách, tổng số cuốn
tồn, tổng giá trị tồn kho. Báo cáo đọc toàn bộ `api.books` — một tác vụ nền. Quy trình: ai cũng tạo
được yêu cầu bất kỳ lúc nào, nhưng báo cáo chỉ chạy khi người vận hành **duyệt**, và người duyệt có
thể chỉ định node chạy. Trong lúc chờ duyệt, Pod không được bắt scheduler thử đi thử lại vô ích.

**Yêu cầu:**

1. Job `bao-cao-ton-kho` ở `docs/bao-cao-ton-kho.job.yaml` — manifest **theo yêu cầu**: không nằm
   trong `k8s/app/`, không do `deploy.sh` tạo. Giải thích lựa chọn này trong ADR.
2. Pod của Job vừa tạo ở trạng thái **chờ duyệt**: scheduler chưa xét tới nó — khác với một Pod
   `Pending` vì thiếu chỗ.
3. Pod báo cáo là tác vụ nền như B4: `bookstore-batch`, ưu tiên node không phải node dữ liệu. Label
   theo hợp đồng của thành phần đọc DB `backup` — để đi đúng đường mạng đã mở cho `backup` ở Lab 5b
   ([ma trận NetworkPolicy](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật)), không thêm dòng mới vào ma trận.
4. Khi thành công, log có đúng một dòng dạng
   `bao-cao: OK dau_sach=<n> ton_kho=<n> gia_tri_vnd=<n>`.
5. Người duyệt **chỉ thu hẹp** được nơi chạy: thêm ràng buộc sang `lab-k8s-worker1` trong lúc Pod còn
   chờ duyệt thì được; gỡ ràng buộc đã có thì không. Không thêm được cổng chờ mới vào một Pod đã tạo.
6. Duyệt = gỡ cổng. Pod chạy, Job hoàn tất.
7. `docs/runbook-bao-cao-ton-kho.md`: tạo yêu cầu, kiểm trạng thái chờ duyệt, chỉ định node, duyệt,
   đọc kết quả, hủy yêu cầu.

**Ràng buộc:** không dùng `suspend` của Job — trọng tâm là bài 145. Không sửa NetworkPolicy. Pod báo
cáo chỉ đọc DB.

**Đọc lại:**

- Bài [145](../145-pod-scheduling-readiness-vi.md):
  [*Cấu hình schedulingGates của Pod*](../145-pod-scheduling-readiness-vi.md#cấu-hình-schedulinggates-của-pod-configuring-pod-schedulinggates),
  [*Ví dụ sử dụng*](../145-pod-scheduling-readiness-vi.md#ví-dụ-sử-dụng-usage-example),
  [*Các chỉ thị lập lịch có thể thay đổi của Pod*](../145-pod-scheduling-readiness-vi.md#các-chỉ-thị-lập-lịch-có-thể-thay-đổi-của-pod-mutable-pod-scheduling-directives).
- Bài [147](../147-scheduling-framework-vi.md#pre-enqueue), mục *PreEnqueue*.
- Bài [67](../67-job-vi.md#chỉ-thị-lập-lịch-có-thể-thay-đổi-mutable-scheduling-directives) (nhóm 4b),
  mục *Chỉ thị lập lịch có thể thay đổi* — pod template của Job được mang scheduling gate.

**Bàn giao:** `docs/bao-cao-ton-kho.job.yaml`; `docs/runbook-bao-cao-ton-kho.md`;
`docs/adr/NNNN-bao-cao-cho-duyet.md`; `~/lab-evidence/7a/b6-*.txt`.

**Nghiệm thu:**

- `PASS:` manifest nằm trong `docs/`; bản render của `k8s/app` không chứa nó.
- `PASS:` Pod của Job ở `SchedulingGated`: chưa gán node, Job đang có 1 Pod hoạt động.
- `PASS:` thêm cổng mới vào Pod bị từ chối.
- `PASS:` thêm `nodeSelector` sang `lab-k8s-worker1` được chấp nhận; gỡ `nodeSelector` bị từ chối.
- `PASS:` gỡ cổng thì Pod chạy trên `lab-k8s-worker1`, Job `Complete`, log có ít nhất 8 đầu sách.
- `PASS:` Pod báo cáo mang độ ưu tiên của `bookstore-batch`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-bao-cao-cho-duyet.md`:

1. `SchedulingGated` và `Pending` vì thiếu chỗ khác nhau thế nào từ góc nhìn scheduler? Pod bị giữ ở
   điểm mở rộng nào của framework, và vì sao nó không mang lý do `Unschedulable`?
2. Vì sao manifest không nằm trong `k8s/app/`? Nếu nằm đó, mỗi lần `deploy.sh` và `smoke.sh` chạy sẽ
   gặp chuyện gì?
3. Vì sao người duyệt chỉ được thu hẹp mà không được nới nơi chạy? Dẫn lý do OR/AND của bài 145.
4. So với `suspend` của Job: Pod có tồn tại không, việc giữ lại xảy ra ở cấp nào?

<details>
<summary>Gợi ý</summary>

1. `spec.template.spec.schedulingGates` là danh sách các `name`. Đặt tên cổng theo domain của dự án,
   ví dụ `bookstore.example.com/duyet-bao-cao`.
2. Pod do Job tạo mang label `job-name=bao-cao-ton-kho`.
3. Gỡ cổng: `kubectl -n bookstore patch pod <tên> --type=json -p '[{"op":"remove","path":"/spec/schedulingGates"}]'`.
4. Thêm `nodeSelector`: `kubectl -n bookstore patch pod <tên> -p '{"spec":{"nodeSelector":{"kubernetes.io/hostname":"lab-k8s-worker1"}}}'`.
5. Kết nối DB như Job `db-migrate`: host `db-0.db`, user `postgres`, mật khẩu từ Secret
   `db-credentials`. `psql -tA -F ' ' -c '...'` trả một dòng, các cột cách nhau bởi dấu cách.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script tạo yêu cầu, siết, duyệt</summary>

```bash
NS=bookstore; EV=~/lab-evidence/7a; F=~/bookstore/docs/bao-cao-ton-kho.job.yaml
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

{ test -f "$F" && ! kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | grep -q 'name: bao-cao-ton-kho'; } \
  && echo 'PASS: manifest bao cao nam trong docs/, khong thuoc k8s/app'
kubectl -n "$NS" delete job bao-cao-ton-kho --ignore-not-found --wait=true >/dev/null
kubectl apply -f "$F" >/dev/null
P=''
for i in $(seq 1 20); do
  P="$(kubectl -n "$NS" get pods -l job-name=bao-cao-ton-kho -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
  [ -n "$P" ] && break
  sleep 2
done
sleep 5
kubectl -n "$NS" get pod "$P" > "$EV/b6-cho-duyet.txt"
{ [ "$(jp "pod/$P" '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'SchedulingGated' ] && \
  [ -z "$(jp "pod/$P" '{.spec.nodeName}')" ] && [ "$(jp job/bao-cao-ton-kho '{.status.active}')" = '1' ]; } \
  && echo "PASS: $P o trang thai SchedulingGated, Job dang cho duyet"

kubectl -n "$NS" patch pod "$P" --type=json \
  -p '[{"op":"add","path":"/spec/schedulingGates/-","value":{"name":"bookstore.example.com/them-sau"}}]' \
  > "$EV/b6-them-cong.txt" 2>&1 && echo 'FAIL: them duoc cong moi' || echo 'PASS: khong them duoc cong moi vao Pod da tao'
kubectl -n "$NS" patch pod "$P" -p '{"spec":{"nodeSelector":{"kubernetes.io/hostname":"lab-k8s-worker1"}}}' \
  > "$EV/b6-siet.txt" 2>&1 && echo 'PASS: them nodeSelector khi Pod con cho duyet duoc chap nhan'
kubectl -n "$NS" patch pod "$P" --type=json -p '[{"op":"remove","path":"/spec/nodeSelector"}]' \
  > "$EV/b6-noi.txt" 2>&1 && echo 'FAIL: go duoc nodeSelector' || echo 'PASS: khong go duoc nodeSelector — chi siet, khong noi'

kubectl -n "$NS" patch pod "$P" --type=json -p '[{"op":"remove","path":"/spec/schedulingGates"}]' >/dev/null
kubectl -n "$NS" wait --for=condition=Complete job/bao-cao-ton-kho --timeout=180s >/dev/null
kubectl -n "$NS" logs job/bao-cao-ton-kho > "$EV/b6-ket-qua.txt" 2>&1
cat "$EV/b6-ket-qua.txt"
DS="$(sed -n 's/^bao-cao: OK dau_sach=\([0-9]*\) .*/\1/p' "$EV/b6-ket-qua.txt")"
{ [ "$(jp "pod/$P" '{.spec.nodeName}')" = 'lab-k8s-worker1' ] && [ "${DS:-0}" -ge 8 ]; } \
  && echo "PASS: duyet xong, bao cao chay tren worker1, $DS dau sach"
[ "$(jp "pod/$P" '{.spec.priority}')" = "$(kubectl get priorityclass bookstore-batch -o jsonpath='{.value}')" ] \
  && echo 'PASS: Pod bao cao la tac vu nen bookstore-batch'
kubectl -n "$NS" delete job bao-cao-ton-kho --wait=true >/dev/null
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# docs/bao-cao-ton-kho.job.yaml — Job THEO YÊU CẦU, không thuộc k8s/app, không do deploy.sh tạo.
# Quy trình: docs/runbook-bao-cao-ton-kho.md
apiVersion: batch/v1
kind: Job
metadata:
  name: bao-cao-ton-kho
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: backup        # thành phần đọc DB: dùng đường mạng đã mở cho backup (5b)
    app.kubernetes.io/name: postgresql
spec:
  backoffLimit: 1
  ttlSecondsAfterFinished: 3600
  template:
    metadata:
      labels:
        app.kubernetes.io/part-of: bookstore
        app.kubernetes.io/component: backup
        app.kubernetes.io/name: postgresql
    spec:
      schedulingGates:
      - name: bookstore.example.com/duyet-bao-cao
      restartPolicy: Never
      priorityClassName: bookstore-batch
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: bookstore.example.com/data
                operator: DoesNotExist
      containers:
      - name: bao-cao
        image: postgres:17.11
        env:
        - {name: PGHOST, value: db-0.db}
        - {name: PGUSER, value: postgres}
        - {name: PGDATABASE, value: bookstore}
        - {name: PGCONNECT_TIMEOUT, value: "5"}
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef: {name: db-credentials, key: postgres-password}
        command:
        - sh
        - -c
        - |
          set -e
          R="$(psql -v ON_ERROR_STOP=1 -tA -F ' ' -c \
            'SELECT count(*), coalesce(sum(stock),0), coalesce(sum(stock::bigint * price_vnd),0) FROM api.books')"
          set -- $R
          echo "bao-cao: OK dau_sach=$1 ton_kho=$2 gia_tri_vnd=$3"
        resources:
          requests: {cpu: 50m, memory: 64Mi}
          limits: {cpu: 200m, memory: 128Mi}
```

Khung `docs/runbook-bao-cao-ton-kho.md`:

```markdown
# Runbook — báo cáo tồn kho chờ duyệt

1. Tạo yêu cầu: `kubectl apply -f docs/bao-cao-ton-kho.job.yaml`
2. Kiểm chờ duyệt: `kubectl -n bookstore get pods -l job-name=bao-cao-ton-kho` — STATUS `SchedulingGated`.
3. (Tùy chọn) Chỉ định node, chỉ khi còn chờ duyệt:
   `kubectl -n bookstore patch pod <pod> -p '{"spec":{"nodeSelector":{"kubernetes.io/hostname":"<node>"}}}'`
4. Duyệt: `kubectl -n bookstore patch pod <pod> --type=json -p '[{"op":"remove","path":"/spec/schedulingGates"}]'`
5. Kết quả: `kubectl -n bookstore logs job/bao-cao-ton-kho` — dòng `bao-cao: OK ...`.
6. Hủy yêu cầu: `kubectl -n bookstore delete job bao-cao-ton-kho`.
```

Ý chính cho ADR:

1. `SchedulingGated`: Pod **chưa hề được thử** — plugin `PreEnqueue` không cho nó vào hàng đợi active,
   nên không có lý do `Unschedulable` (bài 147). `Pending` thiếu chỗ: Pod **đã được thử** ở bước lọc và
   bị kết luận không lập lịch được. Chẩn đoán ngược nhau: một bên đi tìm người chưa duyệt, một bên sửa
   tài nguyên hay ràng buộc.
2. Manifest trong `k8s/app/` thì mỗi lần `deploy.sh` lại tạo một yêu cầu báo cáo chờ duyệt — một
   hành động vận hành biến thành trạng thái thường trực; `smoke.sh` phải bỏ qua một Pod mãi không
   chạy; và Job đã chạy xong rồi bị TTL dọn sẽ được tạo lại ở lần deploy sau. Báo cáo là **thao
   tác**, không phải **trạng thái mong muốn** — cùng cách Lab 4b để `docs/hpa-web.draft.yaml` ngoài
   `k8s/app`.
3. Bài 145: các term trong `nodeSelectorTerms` được OR, các biểu thức trong `matchExpressions` được
   AND; chỉ "thêm" mới bảo đảm tập node mới là **tập con** của tập cũ. Nhờ vậy một controller đã duyệt
   chính sách cho Pod không bị qua mặt bằng một chỉnh sửa nới ra.
4. `suspend` giữ ở **cấp Job**: Job chưa tạo Pod nào. Cổng lập lịch giữ ở **cấp Pod**: Pod đã tồn tại,
   đã qua admission, người duyệt đọc và siết được spec cụ thể của nó trước khi thả vào hàng đợi.

</details>

## B7. [BS-7A-07] Diễn tập: bảo trì `lab-k8s-worker2` bằng taint

**Loại:** Diễn tập sự cố — chỉ trên `lab-k8s-worker2`.

**Bối cảnh:** trước khi học quy trình bảo trì đầy đủ ở Lab 12, đội cần biết taint làm gì với hệ thống
thật trong hai pha của một lần bảo trì: **đóng cửa** node (không nhận Pod mới) rồi **dọn** node (đuổi
Pod đang chạy). Worker2 là node dữ liệu — đây là kịch bản khó nhất của dự án.

**Yêu cầu — pha 1, đóng cửa:**

1. Đặt taint key `bookstore.example.com/bao-tri`, value `true`, effect **chặn Pod mới** lên worker2.
   Chứng minh Pod đang chạy ở đó không bị đuổi.
2. Pod thử nghiệm ghim worker2: `khong-chiu` không có toleration thì nằm chờ, lý do nêu đúng taint bảo
   trì; `chiu-dong-cua` có toleration thì chạy. Pod `ghim-ten` của B1 (`nodeName: lab-k8s-worker2`)
   tạo lại và chạy dù không có toleration nào.
3. Trong lúc worker2 đóng cửa, phát hành lại `web` (`kubectl rollout restart`). Nếu phát hành kẹt,
   đọc lý do, rồi sửa ràng buộc phân bố **trong repo** sao cho phát hành hoàn tất trong lúc bảo trì mà
   vẫn trải đều khi hai worker cùng nhận Pod; triển khai bằng `./scripts/deploy.sh`.

**Pha 2, dọn node:**

4. Tạo thêm hai Pod thử nghiệm ghim worker2: `chiu-60s` chịu được việc bị đuổi trong 60 giây rồi đi;
   `chiu-mai` chịu được mãi — viết toleration của `chiu-mai` **không ghi `effect`**.
5. Thêm effect **đuổi Pod đang chạy** cho cùng key. Ghi nhận: ai đánh dấu xóa `db-0` và vì sao `db-0`
   mới không lên lại được ở đâu; `api` còn sẵn sàng không; `node-agent` còn mấy Pod; `ghim-ten`,
   `chiu-60s`, `chiu-mai` mỗi Pod ra sao.

**Kết thúc bảo trì:**

6. Gỡ mọi taint bảo trì. `db-0` quay lại worker2 với dữ liệu cũ; `node-agent` đủ ba Pod. Đưa `web` và
   `api` về trải đều bằng phát hành lại; `./tests/smoke.sh` đang có trong repo chạy không còn dòng
   `FAIL:`. Xóa Pod thử nghiệm.
7. Viết postmortem.

**Ràng buộc:** chỉ worker2. Không `cordon`/`drain` (giai đoạn 16). Không gỡ taint control plane. Không
để taint bảo trì tồn tại sau ticket. `db` mất sẵn sàng trong pha 2 là chấp nhận được trong diễn tập.

**Đọc lại:**

- Bài [139](../139-taint-and-toleration-vi.md): [*Khái niệm*](../139-taint-and-toleration-vi.md#concepts)
  — ba effect, quy tắc khớp, `effect` rỗng, nhiều taint như bộ lọc, `nodeName` gặp `NoExecute`;
  [*Eviction dựa trên taint*](../139-taint-and-toleration-vi.md#eviction-dựa-trên-taint-taint-based-evictions);
  [*Taint Node theo điều kiện*](../139-taint-and-toleration-vi.md#taint-nodes-by-condition) — toleration
  mà DaemonSet controller tự thêm.
- Bài [140](../140-topology-spread-constraints-vi.md):
  [*Định nghĩa ràng buộc phân bố*](../140-topology-spread-constraints-vi.md#định-nghĩa-ràng-buộc-phân-bố-spread-constraint-definition)
  — `nodeTaintsPolicy`;
  [*Các hạn chế đã biết*](../140-topology-spread-constraints-vi.md#các-hạn-chế-đã-biết-known-limitations).
- Bài [66](../66-daemonset-vi.md#taint-và-toleration-taints-and-tolerations) (nhóm 4b), mục *Taint và
  toleration*.
- Bài [53](../53-disruptions-vi.md#pod-disruption-conditions), mục *Các condition gián đoạn của Pod*.

**Bàn giao:** `k8s/app/web/deployment.yaml` (nếu phải sửa); `docs/postmortem/7a-bao-tri-worker2.md`;
`~/lab-evidence/7a/b7-*.txt`.

**Nghiệm thu:**

- `PASS:` taint chặn Pod mới không đuổi Pod nào đang chạy trên worker2.
- `PASS:` `khong-chiu` chờ với lý do taint bảo trì; `chiu-dong-cua` chạy trên worker2; `ghim-ten` chạy
  trên worker2 dù không có toleration.
- `PASS:` phát hành `web` trong lúc worker2 đóng cửa hoàn tất (sau khi sửa repo nếu cần); mọi Pod
  `web` nằm trên worker1; worker2 vẫn mang taint.
- `PASS:` sau effect đuổi: có sự kiện `TaintManagerEviction` cho `db-0`; `db-0` mới `Pending`; `api`
  không còn bản sao sẵn sàng; `node-agent` còn đúng 2 Pod, không Pod nào trên worker2.
- `PASS:` `ghim-ten` bị đuổi; `chiu-mai` vẫn chạy; `chiu-60s` bị đuổi sau thời hạn của nó.
- `PASS:` gỡ taint: `db-0` chạy trên worker2 với ít nhất 8 cuốn sách; `node-agent` đủ 3 Pod.
- `PASS:` sau khi phát hành lại, `web` nằm trên cả hai worker và `api` trên hai node; không còn taint
  `bookstore.example.com/*`; Pod thử nghiệm đã xóa.
- `PASS:` `./tests/smoke.sh` không còn dòng `FAIL:`.

**Câu hỏi thiết kế** — trả lời trong `docs/postmortem/7a-bao-tri-worker2.md`:

1. Lập bảng: với `db-0`, Pod `web`, `node-agent`, `ghim-ten`, `chiu-60s`, `chiu-mai` — pha 1 và pha 2
   làm gì với từng Pod, và vì sao (dẫn danh sách ba effect của bài 139).
2. Vì sao phát hành `web` kẹt trong pha 1? Bài 140 nói gì về node bị taint trong phép tính phân bố?
   Vì sao chọn `nodeTaintsPolicy: Honor` chứ không đổi sang `ScheduleAnyway`?
3. `db-0` không thể sang worker1 vì **hai** lý do độc lập. Kể ra. Kết luận gì cho việc bảo trì node dữ
   liệu ở Lab 12?
4. Vì sao `node-agent` bị đuổi còn `calico-node` và `kube-proxy` thì không? `node-agent` có nên dung
   thứ taint bảo trì không?
5. Sau khi gỡ taint, `web` và `api` có tự trải lại không? Vì sao? Đây là nợ `TD-12` — đề xuất bước nào
   cho quy trình bảo trì của Lab 12.

<details>
<summary>Gợi ý</summary>

1. Thêm taint: `kubectl taint node lab-k8s-worker2 bookstore.example.com/bao-tri=true:NoSchedule`.
   Gỡ mọi effect của key: `kubectl taint node lab-k8s-worker2 bookstore.example.com/bao-tri-`.
2. Toleration có `effect` rỗng khớp **mọi** effect của key đó (ghi chú trong mục *Khái niệm*).
3. `chiu-60s` phải dung thứ cả effect chặn Pod mới (để lên được node đang đóng cửa) lẫn effect đuổi
   kèm `tolerationSeconds: 60`.
4. Khi phát hành kẹt, đọc thông điệp của Pod `web` đang chờ: nó có hai lý do cho hai worker.
5. Sự kiện của taint manager: `kubectl -n bookstore get events --field-selector reason=TaintManagerEviction`.
6. Phát hành lại: `kubectl -n bookstore rollout restart deployment/web deployment/api`.

</details>

<details>
<summary>Script nghiệm thu — pha 1 (chạy sau khi đã viết bốn manifest thử nghiệm)</summary>

```bash
NS=bookstore; EV=~/lab-evidence/7a; W2=lab-k8s-worker2; LW=~/lab-work/7a; TK=bookstore.example.com/bao-tri
WEB='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web'
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
uids() { kubectl -n "$NS" get pods -l app.kubernetes.io/part-of=bookstore --field-selector spec.nodeName=$W2,status.phase=Running \
           -o jsonpath='{range .items[*]}{.metadata.name}={.metadata.uid}{"\n"}{end}' | grep -v -e '^db-backup-' -e '^db-migrate-' | sort; }

uids > "$EV/b7-truoc.txt"
kubectl get pods -A -o wide --field-selector spec.nodeName=$W2 > "$EV/b7-worker2-truoc.txt"
kubectl taint node $W2 "$TK=true:NoSchedule"
sleep 20
uids > "$EV/b7-sau-dong-cua.txt"
diff "$EV/b7-truoc.txt" "$EV/b7-sau-dong-cua.txt" >/dev/null && echo 'PASS: taint chan Pod moi khong duoi Pod dang chay'

kubectl apply -f "$LW/khong-chiu.yaml" -f "$LW/chiu-dong-cua.yaml" -f "$LW/ghim-ten.yaml" >/dev/null
for i in $(seq 1 30); do
  [ "$(jp pod/khong-chiu '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ] && break; sleep 2
done
jp pod/khong-chiu '{.status.conditions[?(@.type=="PodScheduled")].message}' > "$EV/b7-khong-chiu.txt"
grep -q "$TK" "$EV/b7-khong-chiu.txt" && echo 'PASS: khong-chiu cho vi taint bao tri'
kubectl -n "$NS" wait --for=condition=Ready pod/chiu-dong-cua pod/ghim-ten --timeout=120s >/dev/null
{ [ "$(jp pod/chiu-dong-cua '{.spec.nodeName}')" = "$W2" ] && [ "$(jp pod/ghim-ten '{.spec.nodeName}')" = "$W2" ] && \
  [ -z "$(jp pod/ghim-ten '{.spec.tolerations[?(@.key=="bookstore.example.com/bao-tri")].key}')" ]; } \
  && echo 'PASS: toleration cho phep chiu-dong-cua; ghim-ten vao bang nodeName khong can toleration'

kubectl -n "$NS" rollout restart deployment/web
if kubectl -n "$NS" rollout status deployment/web --timeout=120s > "$EV/b7-phat-hanh-1.txt" 2>&1; then
  echo 'PASS: phat hanh web hoan tat ngay — rang buoc phan bo da tinh toi taint'
else
  P="$(kubectl -n "$NS" get pods -l "$WEB" --field-selector=status.phase=Pending -o jsonpath='{.items[0].metadata.name}')"
  jp "pod/$P" '{.status.conditions[?(@.type=="PodScheduled")].message}' | tee "$EV/b7-phat-hanh-ket.txt"; echo
  grep -q 'topology spread' "$EV/b7-phat-hanh-ket.txt" \
    && echo 'PASS: da tai hien — phat hanh web ket vi rang buoc phan bo van dem worker2 dang dong cua'
  echo '>>> Sua rang buoc trong repo roi chay khoi "sau khi sua" ben duoi'
fi
```

Sau khi sửa repo (bỏ qua nếu phát hành đã hoàn tất ở trên):

```bash
NS=bookstore; EV=~/lab-evidence/7a; W2=lab-k8s-worker2; TK=bookstore.example.com/bao-tri
WEB='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web'
cd ~/bookstore && ./scripts/deploy.sh > "$EV/b7-deploy-sau-sua.txt" 2>&1 && echo 'PASS: deploy.sh hoan tat trong luc worker2 dong cua'
NW="$(kubectl -n "$NS" get pods -l "$WEB" --field-selector=status.phase=Running -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u)"
{ [ "$NW" = 'lab-k8s-worker1' ] && [ "$(kubectl -n "$NS" get deployment web -o jsonpath='{.status.readyReplicas}')" = '3' ] && \
  kubectl get node $W2 -o jsonpath='{.spec.taints[*].key}' | grep -q "$TK"; } \
  && echo 'PASS: web 3/3 tren worker1, worker2 van dong cua'
```

</details>

<details>
<summary>Script nghiệm thu — pha 2 và kết thúc bảo trì (chạy liền sau pha 1)</summary>

```bash
NS=bookstore; EV=~/lab-evidence/7a; W2=lab-k8s-worker2; LW=~/lab-work/7a; TK=bookstore.example.com/bao-tri
SEL='app.kubernetes.io/part-of=bookstore'
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
gone() { ! kubectl -n "$NS" get pod "$1" >/dev/null 2>&1; }

kubectl apply -f "$LW/chiu-60s.yaml" -f "$LW/chiu-mai.yaml" >/dev/null
kubectl -n "$NS" wait --for=condition=Ready pod/chiu-60s pod/chiu-mai --timeout=120s >/dev/null
kubectl taint node $W2 "$TK=true:NoExecute"
T0="$(date +%s)"

for i in $(seq 1 30); do gone ghim-ten && break; sleep 2; done
gone ghim-ten && echo 'PASS: ghim-ten bi duoi — NoExecute khong song o buoc loc nen nodeName khong thoat'
for i in $(seq 1 40); do
  [ "$(jp pod/db-0 '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ] && break; sleep 3
done
kubectl -n "$NS" get events --field-selector reason=TaintManagerEviction \
  -o custom-columns='POD:.involvedObject.name,MSG:.message' > "$EV/b7-taint-manager.txt"
jp pod/db-0 '{.status.conditions[?(@.type=="PodScheduled")].message}' > "$EV/b7-db-cho.txt"
{ grep -q '^db-0 ' "$EV/b7-taint-manager.txt" && [ "$(jp pod/db-0 '{.status.phase}')" = 'Pending' ]; } \
  && echo 'PASS: taint manager danh dau xoa db-0; db-0 moi khong len duoc o dau'
for i in $(seq 1 30); do [ -z "$(jp deployment/api '{.status.readyReplicas}')" ] && break; sleep 3; done
[ -z "$(jp deployment/api '{.status.readyReplicas}')" ] && echo 'PASS: api khong con ban sao san sang khi db mat'
for i in $(seq 1 20); do
  AG="$(kubectl -n "$NS" get pods -l "$SEL,app.kubernetes.io/component=agent" -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}')"
  [ "$(echo "$AG" | grep -c .)" -eq 2 ] && ! echo "$AG" | grep -qx "$W2" && break
  sleep 3
done
{ [ "$(echo "$AG" | grep -c .)" -eq 2 ] && ! echo "$AG" | grep -qx "$W2"; } \
  && echo 'PASS: node-agent con 2 Pod, khong Pod nao tren worker2'
kubectl get pods -A -o wide --field-selector spec.nodeName=$W2 > "$EV/b7-worker2-pha2.txt"

for i in $(seq 1 60); do gone chiu-60s && break; sleep 3; done
T1="$(date +%s)"
{ gone chiu-60s && [ $((T1 - T0)) -ge 60 ] && [ "$(jp pod/chiu-mai '{.status.phase}')" = 'Running' ]; } \
  && echo "PASS: chiu-60s bi duoi sau $((T1 - T0)) giay, chiu-mai van chay"

# Kết thúc bảo trì
kubectl taint node $W2 "$TK-"
kubectl -n "$NS" wait --for=condition=Ready pod/db-0 --timeout=300s >/dev/null
N="$(kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
{ [ "$(jp pod/db-0 '{.spec.nodeName}')" = "$W2" ] && [ "${N:-0}" -ge 8 ]; } \
  && echo "PASS: db-0 quay lai worker2 voi $N cuon sach"
kubectl -n "$NS" rollout status daemonset/node-agent --timeout=180s >/dev/null
[ "$(jp daemonset/node-agent '{.status.numberReady}')" = '3' ] && echo 'PASS: node-agent du 3 Pod'

kubectl -n "$NS" rollout restart deployment/web deployment/api
kubectl -n "$NS" rollout status deployment/web --timeout=300s >/dev/null
kubectl -n "$NS" rollout status deployment/api --timeout=300s >/dev/null
WN="$(kubectl -n "$NS" get pods -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"
AN="$(kubectl -n "$NS" get pods -l "$SEL,app.kubernetes.io/component=api" --field-selector=status.phase=Running -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u | wc -l)"
kubectl -n "$NS" delete pod khong-chiu chiu-dong-cua chiu-mai --ignore-not-found --wait=true >/dev/null
{ [ "$WN" -eq 2 ] && [ "$AN" -eq 2 ] && \
  ! kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' | grep -q '^bookstore\.example\.com/'; } \
  && echo 'PASS: web va api trai lai hai node, khong con taint bao tri'
cd ~/bookstore && ./tests/smoke.sh > "$EV/b7-smoke-sau-bao-tri.txt" 2>&1 \
  && echo 'PASS: smoke.sh cua repo khong con dong FAIL sau bao tri'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Sửa repo cho pha 1 — thêm một dòng vào ràng buộc của `web`:

```yaml
# k8s/app/web/deployment.yaml — trong spec.template.spec.topologySpreadConstraints[0]
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        nodeTaintsPolicy: Honor          # node bị taint mà Pod không dung thứ thì không tính là một miền
        labelSelector:
          matchLabels:
            app.kubernetes.io/part-of: bookstore
            app.kubernetes.io/component: web
```

Bốn Pod thử nghiệm (`ghim-ten` dùng lại manifest của B1) sinh từ một khung chung, chỉ khác tên và
khối `tolerations`:

```bash
pod() {   # $1 = tên Pod, $2 = khối tolerations (có thể rỗng)
  cat > ~/lab-work/7a/$1.yaml <<EOF
apiVersion: v1
kind: Pod
metadata: {name: $1, namespace: bookstore}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  terminationGracePeriodSeconds: 5
$2
  containers:
  - {name: app, image: "busybox:1.37", command: ["sh", "-c", "sleep 3600"], resources: {requests: {cpu: 10m, memory: 16Mi}}}
EOF
}
K=bookstore.example.com/bao-tri
pod khong-chiu ''
pod chiu-dong-cua "  tolerations: [{key: $K, operator: Exists, effect: NoSchedule}]"
pod chiu-60s "  tolerations: [{key: $K, operator: Exists, effect: NoSchedule}, {key: $K, operator: Exists, effect: NoExecute, tolerationSeconds: 60}]"
pod chiu-mai "  tolerations: [{key: $K, operator: Exists}]"   # effect rỗng: khớp mọi effect của key, ở lại mãi
```

Ý chính cho postmortem:

1. Pha 1 (`NoSchedule`): không Pod đang chạy nào bị đuổi; chỉ Pod **mới** bị chặn — `khong-chiu`
   chờ, `chiu-dong-cua` lên được, `ghim-ten` lên được vì `nodeName` bỏ qua scheduler. Pha 2
   (`NoExecute`): Pod không dung thứ bị thu hồi ngay — `db-0`, Pod `api` trên worker2, `node-agent`,
   `ghim-ten`; `chiu-60s` ở lại đúng thời hạn rồi bị thu hồi; `chiu-mai` dung thứ không kèm
   `tolerationSeconds` nên ở lại mãi. Bài 139 nói `NoExecute` do kubelet và taint-eviction-controller
   thi hành chứ không sống ở bước lọc — vì vậy `nodeName` không cứu được `ghim-ten`. Pod bị đuổi mang
   `DisruptionTarget` với lý do `DeletionByTaintManager` (bài 53).
2. Ràng buộc mặc định `nodeTaintsPolicy: Ignore` vẫn tính worker2 là một miền, cùng các Pod `web` cũ
   trên đó — bài 140 ghi đúng điều này trong *Các hạn chế đã biết*. Pod mới chỉ lên được worker1, mà
   càng đặt vào worker1 thì càng lệch so với worker2 → `DoNotSchedule` giữ Pod chờ, phát hành kẹt.
   `Honor` bỏ node bị taint mà Pod không dung thứ khỏi phép tính: lúc bảo trì chỉ còn một miền, lúc
   bình thường vẫn trải đều cứng. `ScheduleAnyway` thì mất cưỡng chế cả lúc bình thường.
3. PV `data-db-0` ghim worker2 bằng hostname (B2) **và** node affinity của `db` đòi label dữ liệu —
   chỉ worker2 có. Dữ liệu nằm cục bộ nên **bảo trì worker2 là cửa sổ downtime của DB**; Lab 12 phải
   quyết định chấp nhận cửa sổ đó hay đổi kiến trúc lưu trữ, và ghi thành ADR.
4. DaemonSet controller chỉ tự thêm toleration cho các taint hệ thống (`not-ready`, `unreachable`,
   các taint áp lực, `unschedulable`); taint tùy biến của dự án không nằm trong đó. `calico-node` và
   `kube-proxy` khai toleration `operator: Exists` không key — dung thứ mọi taint. `node-agent` báo
   trạng thái của chính node; trong cửa sổ bảo trì không có Pod `web` nào ở đó đọc nó, nên để nó bị
   đuổi là chấp nhận được — hoặc dung thứ riêng key bảo trì nếu muốn giữ heartbeat.
5. Không. Bài 140: ràng buộc chỉ được đánh giá lúc lập lịch, không có gì bảo đảm chúng còn đúng sau
   đó; tái cân bằng cần công cụ ngoài như Descheduler. Anti-affinity mềm của `api` cũng vậy. `TD-12`:
   quy trình bảo trì phải có bước phát hành lại `web`, `api` sau khi node quay lại — đề xuất đưa vào
   runbook của Lab 12.

</details>

## B8. [BS-7A-08] Điều tra: hai loại eviction

**Loại:** Điều tra.

**Bối cảnh:** B5 và B7 cho thấy hai cách Pod đang chạy bị chấm dứt: preemption và taint. Còn hai cách
mang cùng tên *eviction*: qua API — công cụ nằm dưới `kubectl drain` mà Lab 12 sẽ dùng — và do kubelet
khi node chịu áp lực. Đội cần biết cái nào tôn trọng PDB và grace period của `web`, và nếu worker2 cạn
RAM thì Pod nào của Bookstore đi trước.

**Yêu cầu — A, eviction qua API trên `web`:**

1. Gửi yêu cầu eviction cho một Pod `web` trên worker2 qua subresource `eviction` — không dùng
   `kubectl drain` hay `kubectl delete`. Chứng minh: yêu cầu được chấp nhận; Pod mang đúng grace period
   của chính nó; condition `DisruptionTarget` cho biết lý do; **trong lúc container còn chạy**, Pod đã
   không còn là endpoint sẵn sàng của Service `web`.
2. Ngay sau đó gửi eviction cho các Pod `web` còn lại, từng Pod một, tới khi bị từ chối. Số lần được
   chấp nhận phải đúng bằng ngân sách của PDB `web` lúc bắt đầu; lần bị từ chối nêu lý do PDB; Pod bị
   từ chối vẫn nguyên.
3. `web` tự về đủ bản sao.

**B, eviction do áp lực node trên worker2 — chỉ đọc:**

4. Đọc ngưỡng eviction mà kubelet trên worker2 **đang thực sự dùng**, và chu kỳ chống dao động điều
   kiện node.
5. Tính khoảng cách hiện tại tới ngưỡng `memory.available`, `nodefs.available`, `nodefs.inodesFree`.
6. Xếp hạng mọi Pod trên worker2 theo ba tham số kubelet dùng khi thiếu bộ nhớ; đánh dấu Pod của
   Bookstore; `db-0` thuộc nhóm nào.
7. Kiểm không node nào đang chịu áp lực và không có taint áp lực. Đọc toleration `not-ready`,
   `unreachable` trên Pod `web` — thứ bạn không hề viết.
8. ADR: so sánh **bốn** con đường chấm dứt Pod đang chạy đã gặp trong sprint — preemption (B5), taint
   `NoExecute` (B7), eviction qua API, eviction do áp lực node — theo: ai quyết định, PDB có được tôn
   trọng không, grace period, `reason` của `DisruptionTarget`.

**Ràng buộc:** phần B **không** gây áp lực thật ([mục 2](#2-quy-ước-và-an-toàn)). Không sửa cấu hình
kubelet. Phần A chỉ evict Pod `web`.

**Đọc lại:**

- Bài [143](../143-api-eviction-vi.md):
  [*Cách hoạt động của eviction khởi phát qua API*](../143-api-eviction-vi.md#cách-hoạt-động-của-eviction-khởi-phát-qua-api-how-api-initiated-eviction-works),
  [*Xử lý sự cố eviction bị kẹt*](../143-api-eviction-vi.md#xử-lý-sự-cố-eviction-bị-kẹt-troubleshooting-stuck-evictions).
- Bài [142](../142-node-pressure-eviction-vi.md): [*Tín hiệu eviction*](../142-node-pressure-eviction-vi.md#eviction-signals),
  [*Ngưỡng eviction cứng*](../142-node-pressure-eviction-vi.md#hard-eviction-thresholds),
  [*Điều kiện node*](../142-node-pressure-eviction-vi.md#node-conditions),
  [*Dao động điều kiện node*](../142-node-pressure-eviction-vi.md#dao-động-điều-kiện-node-node-condition-oscillation),
  [*Lựa chọn pod cho eviction của kubelet*](../142-node-pressure-eviction-vi.md#pod-selection-for-kubelet-eviction).
- Bài [139](../139-taint-and-toleration-vi.md#eviction-dựa-trên-taint-taint-based-evictions), mục
  *Eviction dựa trên taint* — toleration 300 giây tự thêm.
- Bài [141](../141-pod-priority-preemption-vi.md#interactions-of-pod-priority-and-qos), mục *Tương tác
  giữa độ ưu tiên của Pod và chất lượng dịch vụ*.
- Bài [53](../53-disruptions-vi.md#pod-disruption-conditions), mục *Các condition gián đoạn của Pod*.

**Bàn giao:** `docs/adr/NNNN-hai-loai-eviction.md`; `~/lab-evidence/7a/b8-*`.

**Nghiệm thu:**

- `PASS:` eviction đầu tiên được chấp nhận; Pod có `deletionTimestamp`, `deletionGracePeriodSeconds`
  bằng `terminationGracePeriodSeconds` của `web`, và `DisruptionTarget` lý do `EvictionByEvictionAPI`.
- `PASS:` trong lúc Pod đó còn tồn tại, endpoint của nó trong EndpointSlice của `web` không sẵn sàng.
- `PASS:` số eviction được chấp nhận bằng `disruptionsAllowed` ban đầu của PDB `web`; lần tiếp theo
  bị từ chối với lý do PDB và Pod đó vẫn nguyên.
- `PASS:` `web` về đủ 3 bản sao Ready.
- `PASS:` `evictionHard` của kubelet trên worker2 là bộ mặc định (`memory.available` `100Mi`,
  `nodefs.available` `10%`) và có `evictionPressureTransitionPeriod`.
- `PASS:` ba tín hiệu đều đang trên ngưỡng; khoảng cách ghi trong bằng chứng.
- `PASS:` có bảng xếp hạng Pod trên worker2; `db-0` thuộc nhóm dùng dưới `requests`.
- `PASS:` không node nào có điều kiện `*Pressure` là `True` hay taint áp lực; Pod `web` có toleration
  `not-ready` và `unreachable` 300 giây.

<details>
<summary>Gợi ý</summary>

1. Đối tượng `Eviction` là JSON `{"apiVersion":"policy/v1","kind":"Eviction","metadata":{"name":...,"namespace":...}}`;
   gửi bằng `kubectl create --raw /api/v1/namespaces/bookstore/pods/<pod>/eviction -f <file>`.
2. EndpointSlice của Service: `kubectl -n bookstore get endpointslice -l kubernetes.io/service-name=web`.
   Endpoint đang tắt có thể biến mất khỏi slice hoặc ở lại với `ready=false` — cả hai cùng một kết luận.
3. `preStop` của nginx (Lab 3a) giữ Pod `web` sống thêm vài giây — đủ để kịp đọc EndpointSlice.
4. Cấu hình kubelet đang chạy: `kubectl get --raw /api/v1/nodes/lab-k8s-worker2/proxy/configz`. Số liệu
   của node và từng Pod: `.../proxy/stats/summary`. Node không có `jq`; dùng `python3`.

</details>

<details>
<summary>Script nghiệm thu — phần A</summary>

```bash
NS=bookstore; EV=~/lab-evidence/7a; W2=lab-k8s-worker2; LW=~/lab-work/7a
WEB='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=web'
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
evict() {
  printf '{"apiVersion":"policy/v1","kind":"Eviction","metadata":{"name":"%s","namespace":"%s"}}\n' "$1" "$NS" > "$LW/eviction.json"
  kubectl create --raw "/api/v1/namespaces/$NS/pods/$1/eviction" -f "$LW/eviction.json"
}

kubectl -n "$NS" rollout status deployment/web --timeout=180s >/dev/null
for i in $(seq 1 20); do [ "$(jp pdb/web '{.status.disruptionsAllowed}')" -ge 1 ] 2>/dev/null && break; sleep 3; done
ALLOWED="$(jp pdb/web '{.status.disruptionsAllowed}')"
GRACE="$(jp deployment/web '{.spec.template.spec.terminationGracePeriodSeconds}')"
PODS="$( { kubectl -n "$NS" get pods -l "$WEB" --field-selector spec.nodeName=$W2 -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
           kubectl -n "$NS" get pods -l "$WEB" -o jsonpath='{range .items[*]}{.metadata.name} {.spec.nodeName}{"\n"}{end}' \
             | awk -v w=$W2 '$2!=w {print $1}'; } )"
echo "PDB web disruptionsAllowed=$ALLOWED ; grace=$GRACE ; thu tu: $(echo $PODS)" | tee "$EV/b8-a.txt"

FIRST="$(echo "$PODS" | head -1)"; IP="$(jp "pod/$FIRST" '{.status.podIP}')"
# Gửi dồn: Pod đầu, rồi lần lượt các Pod còn lại tới khi bị từ chối — trước khi Pod thay thế kịp Ready.
N_OK=0; REFUSED=''
for P in $PODS; do
  if evict "$P" > "$EV/b8-evict-$P.txt" 2>&1; then N_OK=$((N_OK + 1)); else REFUSED="$P"; break; fi
done
[ "$N_OK" -ge 1 ] && echo "PASS: eviction $FIRST duoc chap nhan"
DG="$(jp "pod/$FIRST" '{.metadata.deletionGracePeriodSeconds}')"
DT="$(jp "pod/$FIRST" '{.status.conditions[?(@.type=="DisruptionTarget")].reason}')"
echo "$FIRST: deletionGracePeriodSeconds=$DG DisruptionTarget=$DT" | tee -a "$EV/b8-a.txt"
{ [ -n "$(jp "pod/$FIRST" '{.metadata.deletionTimestamp}')" ] && [ "$DG" = "$GRACE" ] && \
  [ "$DT" = 'EvictionByEvictionAPI' ]; } && echo 'PASS: Pod mang grace period cua chinh no va DisruptionTarget=EvictionByEvictionAPI'
OK_EP=0
for i in 1 2 3 4 5 6; do
  kubectl -n "$NS" get pod "$FIRST" >/dev/null 2>&1 || break
  kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=web \
    -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{" ready="}{.conditions.ready}{"\n"}{end}' > "$EV/b8-endpoints.txt"
  grep -q "^${IP} ready=true" "$EV/b8-endpoints.txt" || { OK_EP=1; break; }
  sleep 1
done
[ "$OK_EP" -eq 1 ] && echo 'PASS: Pod con song nhung da khong con la endpoint san sang cua web'

echo "duoc chap nhan=$N_OK ; bi tu choi=$REFUSED" | tee -a "$EV/b8-a.txt"
{ [ "$N_OK" -eq "$ALLOWED" ] && [ -n "$REFUSED" ] && grep -qi 'disruption budget' "$EV/b8-evict-$REFUSED.txt" && \
  [ -z "$(jp "pod/$REFUSED" '{.metadata.deletionTimestamp}')" ]; } \
  && echo "PASS: ngan sach PDB = $ALLOWED lan; lan tiep theo bi tu choi va Pod $REFUSED con nguyen"

kubectl -n "$NS" rollout status deployment/web --timeout=180s >/dev/null
[ "$(jp deployment/web '{.status.readyReplicas}')" = '3' ] && echo 'PASS: web tu ve du 3 ban sao'
```

</details>

<details>
<summary>Script nghiệm thu — phần B</summary>

```bash
NS=bookstore; EV=~/lab-evidence/7a; W2=lab-k8s-worker2
kubectl get --raw "/api/v1/nodes/$W2/proxy/configz" > "$EV/b8-configz.json"
kubectl get --raw "/api/v1/nodes/$W2/proxy/stats/summary" > "$EV/b8-summary.json"
kubectl get pods -A --field-selector spec.nodeName=$W2 -o json > "$EV/b8-pods-w2.json"

python3 - "$EV/b8-configz.json" "$EV/b8-summary.json" "$EV/b8-pods-w2.json" > "$EV/b8-ap-luc.txt" <<'PY'
import json, sys
cfg = json.load(open(sys.argv[1]))['kubeletconfig']
summ = json.load(open(sys.argv[2]))
pods = json.load(open(sys.argv[3]))['items']
U = {'Ki': 2**10, 'Mi': 2**20, 'Gi': 2**30, 'Ti': 2**40, 'k': 10**3, 'K': 10**3, 'M': 10**6, 'G': 10**9, 'T': 10**12}
def qty(s):
    for u in sorted(U, key=len, reverse=True):
        if s.endswith(u):
            return float(s[:-len(u)]) * U[u]
    return float(s)
eh = cfg.get('evictionHard', {})
print('evictionHard', json.dumps(eh, sort_keys=True))
print('evictionSoft', json.dumps(cfg.get('evictionSoft', {}), sort_keys=True))
print('evictionPressureTransitionPeriod', cfg.get('evictionPressureTransitionPeriod'))
def nguong(sig, tong):
    v = eh.get(sig, '0')
    return float(v[:-1]) / 100 * tong if v.endswith('%') else qty(v)
node = summ['node']; fs = node['fs']
for ten, hien, tong in (('memory.available', node['memory']['availableBytes'], 0),
                        ('nodefs.available', fs['availableBytes'], fs['capacityBytes']),
                        ('nodefs.inodesFree', fs['inodesFree'], fs['inodes'])):
    t = nguong(ten, tong)
    print(f"TIN_HIEU {ten} hien_tai={int(hien)} nguong={int(t)} {'TREN' if hien > t else 'DUOI'}_NGUONG")
dung = {(p['podRef']['namespace'], p['podRef']['name']): p.get('memory', {}).get('workingSetBytes', 0)
        for p in summ.get('pods', [])}
hang = []
for p in pods:
    if p['status'].get('phase') != 'Running':
        continue
    ns, ten, spec = p['metadata']['namespace'], p['metadata']['name'], p['spec']
    cs = spec.get('containers', []) + [c for c in spec.get('initContainers', []) if c.get('restartPolicy') == 'Always']
    req = sum(qty(c.get('resources', {}).get('requests', {}).get('memory', '0')) for c in cs)
    u = dung.get((ns, ten), 0)
    hang.append((0 if u > req else 1, spec.get('priority', 0), -(u - req), ns, ten, u, req))
hang.sort()
print('HANG NHOM       PRIORITY   DUNG_MI   REQ_MI  POD')
for i, r in enumerate(hang, 1):
    print(f"{i:>4} {'VUOT_REQ' if r[0] == 0 else 'DUOI_REQ':<9} {r[1]:>11} {r[5]/2**20:>8.1f} {r[6]/2**20:>8.1f}  {r[3]}/{r[4]}")
PY
cat "$EV/b8-ap-luc.txt"

{ grep -q '"memory.available": "100Mi"' "$EV/b8-ap-luc.txt" && grep -q '"nodefs.available": "10%"' "$EV/b8-ap-luc.txt" && \
  grep -q '^evictionPressureTransitionPeriod [0-9]' "$EV/b8-ap-luc.txt"; } \
  && echo 'PASS: kubelet worker2 dung bo nguong cung mac dinh, co chu ky chong dao dong'
[ "$(grep -c '^TIN_HIEU .*TREN_NGUONG' "$EV/b8-ap-luc.txt")" -eq 3 ] && echo 'PASS: ba tin hieu deu tren nguong'
grep -E ' bookstore/db-0$' "$EV/b8-ap-luc.txt" | grep -q 'DUOI_REQ' && echo 'PASS: db-0 thuoc nhom dung duoi requests'

kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" "}{range .status.conditions[?(@.status=="True")]}{.type}{" "}{end}{"\n"}{end}' \
  > "$EV/b8-dieu-kien.txt"
{ ! grep -q 'Pressure' "$EV/b8-dieu-kien.txt" && \
  ! kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' | grep -q 'pressure'; } \
  && echo 'PASS: khong node nao chiu ap luc, khong co taint ap luc'
WP="$(kubectl -n "$NS" get pods -l app.kubernetes.io/component=web -o jsonpath='{.items[0].metadata.name}')"
T='{.spec.tolerations[?(@.key=="node.kubernetes.io/%s")].tolerationSeconds}'
{ [ "$(kubectl -n "$NS" get pod "$WP" -o jsonpath="$(printf "$T" not-ready)")" = '300' ] && \
  [ "$(kubectl -n "$NS" get pod "$WP" -o jsonpath="$(printf "$T" unreachable)")" = '300' ]; } \
  && echo 'PASS: Pod web co toleration not-ready va unreachable 300 giay do Kubernetes tu them'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của ADR</summary>

| Con đường | Ai quyết định | PDB | Grace period của Pod | `DisruptionTarget.reason` |
| --- | --- | --- | --- | --- |
| Preemption (B5) | kube-scheduler, ở `PostFilter` | chỉ nỗ lực tốt nhất | được tôn trọng | `PreemptionByScheduler` |
| Taint `NoExecute` (B7) | taint-eviction-controller | không hỏi | được tôn trọng | `DeletionByTaintManager` |
| Eviction qua API (B8-A) | API server, theo yêu cầu của client (`kubectl drain`) | **tôn trọng** — `429` khi hết ngân sách | được tôn trọng | `EvictionByEvictionAPI` |
| Áp lực node (B8-B, chỉ đọc) | kubelet | **không** tôn trọng | **không** — ngưỡng cứng dùng `0s` | `TerminationByKubelet` |

- Bài 143, trình tự sáu bước: bước 3 — control plane gỡ Pod khỏi EndpointSlice **trong lúc** kubelet
  còn đang tắt nó. `preStop` của `web` cho đúng khoảng đó: Pod thôi nhận request mới trước khi ngừng
  xử lý request đang dở. `429` là "chờ đi"; `500` (nhiều PDB cùng trỏ một Pod) là "cấu hình sai".
- Ngân sách dùng một lần: API server trừ `disruptionsAllowed` ngay khi chấp nhận eviction, nên yêu cầu
  thứ hai đến trước khi Pod thay thế Ready thì bị từ chối — đúng tình huống *eviction bị kẹt* nếu Pod
  mới không bao giờ Ready.
- Bài 142: kubelet xếp hạng theo (1) dùng có vượt `requests` không, (2) Priority, (3) mức vượt. Không
  dùng QoS class — QoS chỉ để **ước lượng**. `db-0` là Guaranteed, dùng dưới `requests`, độ ưu tiên
  `bookstore-critical`: nằm ở nhóm cuối, sau mọi Pod độ ưu tiên thấp hơn cùng nhóm; chỉ bị chọn khi
  daemon hệ thống tự ăn quá phần dự trữ. Pod hệ thống `system-node-critical` còn đứng sau nó nữa.
- Bộ ngưỡng đang dùng là mặc định; nhớ bẫy của bài 142: đổi **một** ngưỡng cứng thì các ngưỡng còn
  lại về 0 chứ không giữ mặc định. Sửa cấu hình kubelet thuộc giai đoạn 20.
- Toleration `not-ready`/`unreachable` 300 giây do admission tự thêm: khi worker2 mất liên lạc, Pod
  `web` ở đó còn được giữ khoảng thời gian đó trước khi bị thu hồi — Lab 12 đo hệ quả.

</details>

## B9. [BS-7A-09] Điều tra: cấu hình scheduler, Pod overhead và bin packing

**Loại:** Điều tra.

**Bối cảnh:** ba câu đội vận hành sẽ hỏi khi cluster lớn lên: có nên tinh chỉnh scheduler cho nhanh,
có nên dồn Pod (bin packing) để bớt node, và một runtime ảo hóa sẽ làm sai phép tính tài nguyên ra
sao. Trên cluster này hai câu đầu chỉ **đọc** được — sửa cấu hình scheduler là việc của giai đoạn 20.
Câu thứ ba thì thử được phép tính: một RuntimeClass trỏ vào handler `runc` có sẵn trong containerd
(Lab 2) mà vẫn khai `overhead`.

**Yêu cầu:**

1. Đọc cấu hình kube-scheduler đang chạy: có nạp file cấu hình không; mọi Pod của cluster dùng
   scheduler tên gì. Kết luận về `percentageOfNodesToScore` với số node hiện có.
2. Trong ADR, lập bảng: mỗi điểm mở rộng `PreEnqueue`, `QueueSort`, `Filter`, `Score`, `PostFilter`,
   `Bind` — bằng chứng nào trong `~/lab-evidence/7a/` của sprint này cho thấy nó làm việc.
3. RuntimeClass thử nghiệm `bookstore-overhead-thu` (handler `runc`, overhead `250m` CPU và `120Mi`
   bộ nhớ) và Pod `overhead-thu` dùng nó (container xin `100m`/`64Mi`, `requests` bằng `limits`, ghim
   worker2). Chứng minh: `spec.overhead` được điền dù bạn không viết; node tính request của Pod bằng
   tổng container **cộng** overhead; một Pod tự khai `overhead` mà không có RuntimeClass bị từ chối.
4. ⭐ Đọc `memory.max` của cgroup cấp Pod `overhead-thu` trên worker2.
5. ⭐ Pod `overhead-chat`: container xin đúng chỗ trống CPU của worker2 trừ `100m`, dùng RuntimeClass
   trên → `Pending` vì thiếu CPU; cùng Pod bỏ RuntimeClass (`khong-overhead`) thì chạy.
6. ADR: không tinh chỉnh scheduler, không bật bin packing — và khi nào sẽ cân nhắc lại.
7. Xóa RuntimeClass và mọi Pod thử nghiệm.

**Ràng buộc:** chỉ đọc `/etc/kubernetes/manifests/` và spec của static Pod; không sửa. RuntimeClass
thử nghiệm là object cấp cluster ngoài dự án — phải xóa trong ticket.

**Đọc lại:**

- Bài [146](../146-scheduler-perf-tuning-vi.md):
  [*Ngưỡng chấm điểm node*](../146-scheduler-perf-tuning-vi.md#percentage-of-nodes-to-score),
  [*Ngưỡng mặc định*](../146-scheduler-perf-tuning-vi.md#ngưỡng-mặc-định-default-threshold),
  [*Tinh chỉnh percentageOfNodesToScore*](../146-scheduler-perf-tuning-vi.md#tinh-chỉnh-percentageofnodestoscore-tuning-percentageofnodestoscore).
- Bài [147](../147-scheduling-framework-vi.md):
  [*Chu trình lập lịch & chu trình binding*](../147-scheduling-framework-vi.md#chu-trình-lập-lịch--chu-trình-binding-scheduling-cycle--binding-cycle),
  [*PreEnqueue*](../147-scheduling-framework-vi.md#pre-enqueue),
  [*QueueSort*](../147-scheduling-framework-vi.md#queue-sort),
  [*Filter*](../147-scheduling-framework-vi.md#filter),
  [*PostFilter*](../147-scheduling-framework-vi.md#post-filter),
  [*Score*](../147-scheduling-framework-vi.md#scoring),
  [*Bind*](../147-scheduling-framework-vi.md#bind).
- Bài [148](../148-resource-bin-packing-vi.md):
  [*Bật đóng gói tài nguyên bằng chiến lược MostAllocated*](../148-resource-bin-packing-vi.md#bật-đóng-gói-tài-nguyên-bằng-chiến-lược-mostallocated-enabling-bin-packing-using-mostallocated-strategy),
  [*Tinh chỉnh hàm chấm điểm*](../148-resource-bin-packing-vi.md#tinh-chỉnh-hàm-chấm-điểm-tuning-the-score-function).
- Bài [144](../144-pod-overhead-vi.md): [*Cấu hình Pod overhead*](../144-pod-overhead-vi.md#set-up),
  [*Ví dụ sử dụng*](../144-pod-overhead-vi.md#ví-dụ-sử-dụng-usage-example).
- Bài [43](../43-runtime-class-vi.md) (giai đoạn 2) — RuntimeClass chọn một handler đã có.

**Bàn giao:** `docs/adr/NNNN-cau-hinh-scheduler.md`; `~/lab-evidence/7a/b9-*.txt`.

**Nghiệm thu:**

- `PASS:` kube-scheduler không nạp `--config`; mọi Pod dùng `default-scheduler`; cluster dưới 100
  node.
- `PASS:` ADR có đủ sáu điểm mở rộng.
- `PASS:` `overhead-thu` có `spec.overhead` `250m`/`120Mi`; node báo request `350m` và `184Mi` cho Pod
  này.
- `PASS:` Pod tự khai `overhead` không kèm RuntimeClass bị từ chối.
- `PASS:` (⭐) `memory.max` của cgroup cấp Pod là `192937984` byte (184Mi).
- `PASS:` (⭐) `overhead-chat` chờ vì `Insufficient cpu`; `khong-overhead` cùng request thì chạy.
- `PASS:` RuntimeClass và Pod thử nghiệm đã bị xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-cau-hinh-scheduler.md`:

1. Đặt `percentageOfNodesToScore: 10` trên cluster này thì được gì? Núm đó đổi cái gì lấy cái gì?
2. Bảng ánh xạ ở yêu cầu 2: vì sao preemption không phải "tính năng rời" mà là một plugin ở
   `PostFilter`, và vì sao cổng lập lịch của B6 không làm Pod mang lý do `Unschedulable`?
3. Overhead được cộng vào **ba** chỗ nào? Nếu dự án chuyển `db` sang runtime ảo hóa với overhead
   `250m`, bài toán B5 và ngân sách CPU của worker2 đổi thế nào?
4. Bật `MostAllocated` trên cluster hai worker này có lợi gì, hại gì? R10 nói gì?

<details>
<summary>Gợi ý</summary>

1. Static Pod kube-scheduler mang label `component=kube-scheduler`:
   `kubectl -n kube-system get pod -l component=kube-scheduler -o jsonpath='{.items[0].spec.containers[0].command}'`.
2. Tên scheduler của mọi Pod: trường `spec.schedulerName`.
3. RuntimeClass thuộc API group `node.k8s.io/v1`; trường `overhead.podFixed`. Tạo RuntimeClass trước,
   chờ vài giây rồi mới tạo Pod dùng nó.
4. Request mà node tính: dòng của Pod trong `kubectl describe node lab-k8s-worker2`.
5. Cgroup cấp Pod Guaranteed với driver systemd:
   `/sys/fs/cgroup/kubepods.slice/kubepods-pod<UID, gạch ngang đổi thành gạch dưới>.slice/` (Lab 3c).

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi đã viết ADR và hai manifest overhead</summary>

```bash
NS=bookstore; EV=~/lab-evidence/7a; W2=lab-k8s-worker2; LW=~/lab-work/7a
cpu2m() { case "$1" in '') echo 0 ;; *m) echo "${1%m}" ;; *) awk -v x="$1" 'BEGIN{printf "%d\n", x*1000}' ;; esac; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

kubectl -n kube-system get pod -l component=kube-scheduler -o jsonpath='{.items[0].spec.containers[0].command}' \
  | tr ',' '\n' > "$EV/b9-scheduler-cmd.txt"
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.schedulerName}{"\n"}{end}' | sort | uniq -c > "$EV/b9-scheduler-name.txt"
NODES="$(kubectl get nodes --no-headers | wc -l)"
{ ! grep -q -- '--config' "$EV/b9-scheduler-cmd.txt" && \
  [ "$(grep -vc 'default-scheduler' "$EV/b9-scheduler-name.txt")" -eq 0 ] && [ "$NODES" -lt 100 ]; } \
  && echo "PASS: scheduler cau hinh mac dinh, moi Pod dung default-scheduler, $NODES node < 100"
ADR="$(ls ~/bookstore/docs/adr/*-cau-hinh-scheduler.md 2>/dev/null | head -1)"
BAD=0
for E in PreEnqueue QueueSort Filter Score PostFilter Bind; do grep -q "$E" "$ADR" 2>/dev/null || { echo "FAIL: ADR thieu $E"; BAD=1; }; done
{ [ -n "$ADR" ] && [ "$BAD" -eq 0 ]; } && echo 'PASS: ADR anh xa du sau diem mo rong'

kubectl get runtimeclass > "$EV/b9-runtimeclass-truoc.txt" 2>&1
kubectl apply -f "$LW/rc-overhead.yaml" >/dev/null; sleep 5
kubectl apply -f "$LW/overhead-thu.yaml" >/dev/null
kubectl -n "$NS" wait --for=condition=Ready pod/overhead-thu --timeout=120s >/dev/null
{ [ "$(jp pod/overhead-thu '{.spec.overhead.cpu}')" = '250m' ] && [ "$(jp pod/overhead-thu '{.spec.overhead.memory}')" = '120Mi' ]; } \
  && echo 'PASS: admission dien spec.overhead tu RuntimeClass'
kubectl describe node "$W2" | grep 'overhead-thu' | tee "$EV/b9-node-request.txt"
grep -Eq 'overhead-thu +350m .*184Mi' "$EV/b9-node-request.txt" && echo 'PASS: node tinh request = container + overhead (350m, 184Mi)'
kubectl apply --dry-run=server -f "$LW/tu-khai-overhead.yaml" > "$EV/b9-tu-khai.txt" 2>&1 \
  && echo 'FAIL: tu khai overhead duoc chap nhan' \
  || { grep -qi 'overhead' "$EV/b9-tu-khai.txt" && echo 'PASS: Pod tu khai overhead khong kem RuntimeClass bi tu choi'; }

# ⭐ cgroup cấp Pod
U="$(jp pod/overhead-thu '{.metadata.uid}' | tr '-' '_')"
MM="$(ssh "$W2" "cat /sys/fs/cgroup/kubepods.slice/kubepods-pod${U}.slice/memory.max")"
echo "memory.max = $MM" | tee -a "$EV/b9-node-request.txt"
[ "$MM" = '192937984' ] && echo 'PASS: cgroup cap Pod gioi han o 64Mi + 120Mi = 184Mi'

# ⭐ Pod chờ chỉ vì overhead
ALLOC="$(cpu2m "$(kubectl get node $W2 -o jsonpath='{.status.allocatable.cpu}')")"
USED="$(cpu2m "$(kubectl describe node $W2 | awk '/^Allocated resources/,/^Events/' | awk '$1=="cpu"{print $2; exit}')")"
REQ=$((ALLOC - USED - 100))
for X in overhead-chat khong-overhead; do
  RC=''; [ "$X" = overhead-chat ] && RC='  runtimeClassName: bookstore-overhead-thu'
  cat > "$LW/$X.yaml" <<EOF
apiVersion: v1
kind: Pod
metadata: {name: $X, namespace: bookstore}
spec:
$RC
  nodeSelector: {kubernetes.io/hostname: $W2}
  terminationGracePeriodSeconds: 5
  containers:
  - {name: app, image: "busybox:1.37", command: ["sh", "-c", "sleep 3600"], resources: {requests: {cpu: "${REQ}m", memory: 16Mi}}}
EOF
done
kubectl apply -f "$LW/overhead-chat.yaml" >/dev/null
for i in $(seq 1 20); do [ "$(jp pod/overhead-chat '{.status.conditions[?(@.type=="PodScheduled")].reason}')" = 'Unschedulable' ] && break; sleep 3; done
jp pod/overhead-chat '{.status.conditions[?(@.type=="PodScheduled")].message}' > "$EV/b9-overhead-chat.txt"
grep -q 'Insufficient cpu' "$EV/b9-overhead-chat.txt" && echo "PASS: overhead-chat xin ${REQ}m + 250m overhead nen khong vua worker2"
kubectl -n "$NS" delete pod overhead-chat --wait=true >/dev/null
kubectl apply -f "$LW/khong-overhead.yaml" >/dev/null
kubectl -n "$NS" wait --for=condition=Ready pod/khong-overhead --timeout=120s >/dev/null \
  && echo "PASS: cung ${REQ}m ma khong co overhead thi Pod chay"

kubectl -n "$NS" delete pod overhead-thu khong-overhead overhead-chat --ignore-not-found --wait=true >/dev/null
kubectl delete runtimeclass bookstore-overhead-thu --ignore-not-found >/dev/null
{ ! kubectl get runtimeclass bookstore-overhead-thu >/dev/null 2>&1 && \
  [ -z "$(kubectl -n "$NS" get pods -o name | grep -E 'overhead')" ]; } && echo 'PASS: da xoa RuntimeClass va Pod thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/7a/rc-overhead.yaml — handler runc là cấu hình runtime mặc định có sẵn trong containerd (Lab 2)
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: bookstore-overhead-thu
handler: runc
overhead:
  podFixed:
    cpu: 250m
    memory: 120Mi
```

```yaml
# ~/lab-work/7a/overhead-thu.yaml — requests bằng limits: Pod Guaranteed
apiVersion: v1
kind: Pod
metadata: {name: overhead-thu, namespace: bookstore}
spec:
  runtimeClassName: bookstore-overhead-thu
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  terminationGracePeriodSeconds: 5
  containers:
  - name: app
    image: busybox:1.37
    command: ["sh", "-c", "sleep 3600"]
    resources: {requests: {cpu: 100m, memory: 64Mi}, limits: {cpu: 100m, memory: 64Mi}}
---
# ~/lab-work/7a/tu-khai-overhead.yaml — chỉ dùng với --dry-run=server
apiVersion: v1
kind: Pod
metadata: {name: tu-khai-overhead, namespace: bookstore}
spec:
  overhead: {cpu: 10m}
  containers:
  - {name: app, image: "busybox:1.37", command: ["sh", "-c", "sleep 3600"]}
```

Bảng ánh xạ cho ADR:

| Điểm mở rộng | Bằng chứng trong sprint |
| --- | --- |
| `PreEnqueue` | B6 — Pod báo cáo `SchedulingGated`, không có lý do `Unschedulable` (`b6-cho-duyet.txt`) |
| `QueueSort` | B4 — `spec.priority` quyết định thứ tự hàng đợi (`b4-thu-tu.txt`) |
| `Filter` | B1, B2, B3 — thông điệp lọc nêu lý do cho từng node (`b1-qua-co.txt`, `b2-thieu-label.txt`, `b3-sai-khoa.txt`) |
| `Score` | B4 — `uu-tien-thu` về worker1 nhờ `weight` của quy tắc `preferred` |
| `PostFilter` | B5 — `nominatedNodeName` chỉ xuất hiện sau khi bước lọc trả về rỗng (`b5-so-lieu.txt`) |
| `Bind` | B1 — sự kiện `Scheduled` do `default-scheduler` ghi (`b1-scheduled.txt`) |

Ý chính cho ADR:

1. Không được gì: bài 146 nói dưới 100 node khả thi thì scheduler vẫn kiểm tra tất cả, và cluster vài
   trăm node trở xuống nên để mặc định. Núm đổi **độ chính xác** lấy **độ trễ**: node bị bỏ qua ở bước
   lọc không bao giờ được chấm điểm.
2. Bài 147: preemption là triển khai điển hình của `PostFilter`, và `PostFilter` **chỉ chạy khi bước
   lọc không tìm thấy node nào** — đúng thứ tự B5 thấy. Cổng lập lịch chặn ở `PreEnqueue`, **trước**
   hàng đợi active: Pod chưa hề được thử nên không có kết luận `Unschedulable` nào để ghi.
3. Bài 144: scheduler cộng overhead vào tổng request khi tìm node; kubelet cộng nó vào kích thước
   cgroup cấp Pod; và nó tham gia xếp hạng eviction (cộng thêm: ResourceQuota ở nhóm 7b). `db` với
   overhead `250m` sẽ chiếm `750m` của worker2 thay vì `500m` — chỗ trống của B5 nhỏ đi đúng chừng đó,
   và mọi phép tính ngân sách ở mục 3.6 phải tính lại.
4. Bài 148: `MostAllocated` dồn Pod lên ít node nhất — lợi khi có autoscaler thu hẹp node rảnh.
   Cluster lab không co giãn node, nên chỉ còn cái hại: workload dồn về một máy, đi ngược R10. Chiến
   lược mặc định đang trải Pod là phù hợp. Cân nhắc lại khi có node autoscaling và nhiều node hơn.

</details>

## B10. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 10 — mỗi workload đúng chỗ
— rồi trả cluster về đúng `03-storage-ready`, không còn label hay taint của dự án trên node nào.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 7a — kiểm thử khói **tích lũy**, chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 7a. Kiểm mọi hợp đồng còn hiệu lực tới sprint 10: controller,
# cấu hình, Service và DNS, NetworkPolicy, Ingress TLS, dữ liệu bền, tác vụ nền, và vị trí đặt
# workload (label dữ liệu, node affinity, trải đều, anti-affinity, PriorityClass).
set -u
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
DL='bookstore.example.com/data'
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
nodes_of() { k get pods -l "$SEL,app.kubernetes.io/component=$1" --field-selector=status.phase=Running \
               -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' 2>/dev/null; }

# 1. Namespace, object cấp cluster, node dữ liệu
[ "$(kubectl get namespace "$NS" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = 'bookstore' ] \
  && pass 'namespace bookstore mang label du an' || fail 'namespace bookstore thieu hoac thieu label'
for PC in bookstore-critical bookstore-batch; do
  [ "$(kubectl get priorityclass "$PC" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = 'bookstore' ] \
    && pass "PriorityClass $PC mang label du an" || fail "thieu PriorityClass $PC hoac thieu label"
done
CRIT="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
BATCH="$(kubectl get priorityclass bookstore-batch -o jsonpath='{.value}' 2>/dev/null)"
{ [ "${BATCH:-0}" -lt 0 ] && [ "${CRIT:-0}" -gt 0 ]; } \
  && pass "thu tu uu tien: bookstore-batch ($BATCH) < mac dinh (0) < bookstore-critical ($CRIT)" \
  || fail 'gia tri PriorityClass sai thu tu'
[ "$(kubectl get storageclass bookstore-retain -o jsonpath='{.reclaimPolicy}' 2>/dev/null)" = 'Retain' ] \
  && pass 'StorageClass bookstore-retain Retain' || fail 'thieu StorageClass bookstore-retain Retain'
[ "$(kubectl get nodes -l "$DL=true" -o jsonpath='{.items[*].metadata.name}')" = 'lab-k8s-worker2' ] \
  && pass 'chi lab-k8s-worker2 mang label du lieu' || fail 'label du lieu khong nam dung tren lab-k8s-worker2'

# 2. Cấu hình, bí mật, PDB
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials du hai key' || fail 'Secret db-credentials thieu key'
[ "$(jp secret/bookstore-tls '{.type}')" = 'kubernetes.io/tls' ] && pass 'Secret bookstore-tls kieu TLS' || fail 'thieu Secret TLS bookstore-tls'
for P in web api; do
  k get pdb "$P" >/dev/null 2>&1 && pass "co PodDisruptionBudget $P" || fail "thieu PodDisruptionBudget $P"
done

# 3. db: StatefulSet, dữ liệu bền, đặt theo label dữ liệu
[ "$(jp statefulset/db '{.status.readyReplicas}')" = '1' ] && pass 'StatefulSet db 1/1 Ready' || fail 'StatefulSet db chua Ready'
[ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && pass 'db-0 tren lab-k8s-worker2' || fail 'db-0 khong o lab-k8s-worker2'
jp statefulset/db '{.spec.template.spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms[*].matchExpressions[*].key}' \
  | grep -q "$DL" && pass 'db dat bang node affinity bat buoc tren label du lieu' || fail 'db khong dung node affinity theo label du lieu'
[ "$(jp pod/db-0 '{.status.qosClass}')" = 'Guaranteed' ] && pass 'db-0 QoS Guaranteed' || fail 'db-0 khong Guaranteed'
{ [ -n "$CRIT" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$CRIT" ]; } && pass 'db-0 dung bookstore-critical' || fail 'db-0 sai priority'
{ [ "$(jp pvc/data-db-0 '{.status.phase}')" = 'Bound' ] && [ "$(jp pvc/data-db-0 '{.spec.storageClassName}')" = 'bookstore-retain' ]; } \
  && pass 'PVC data-db-0 Bound, class bookstore-retain' || fail 'PVC data-db-0 sai'
PV="$(jp pvc/data-db-0 '{.spec.volumeName}')"
{ [ -n "$PV" ] && [ "$(kubectl get pv "$PV" -o jsonpath='{.spec.persistentVolumeReclaimPolicy}' 2>/dev/null)" = 'Retain' ]; } \
  && pass "PV $PV reclaimPolicy Retain" || fail 'PV cua data-db-0 khong Retain'
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach" || fail "db co ${N:-0} cuon sach, can >= 8"

# 4. web, api: controller, trải đều, anti-affinity mềm, tài nguyên
for X in web:3 api:2; do
  D="${X%%:*}"; WANT="${X##*:}"
  [ "$(jp "deployment/$D" '{.status.readyReplicas}')" = "$WANT" ] && pass "Deployment $D $WANT/$WANT Ready" \
    || fail "Deployment $D chua du $WANT ban sao Ready"
done
W1="$(nodes_of web | grep -cx lab-k8s-worker1)"; W2="$(nodes_of web | grep -cx lab-k8s-worker2)"
{ [ "$W1" -ge 1 ] && [ "$W2" -ge 1 ] && [ $((W1 - W2)) -le 1 ] && [ $((W2 - W1)) -le 1 ]; } \
  && pass "web trai deu hai worker ($W1/$W2)" || fail "web lech: worker1=$W1 worker2=$W2"
# api dùng anti-affinity mềm theo hợp đồng: trải hai node là mong muốn, không bảo đảm — chỉ báo INFO.
NAPI="$(nodes_of api | sort -u | wc -l)"
[ "$NAPI" -eq 2 ] && echo 'INFO: api tren hai node khac nhau' \
  || echo "INFO: api tren $NAPI node — anti-affinity mem; can bang lai truoc khi bao tri node do"
AP="$(k get pod -l "$SEL,app.kubernetes.io/component=api" -o jsonpath='{.items[0].spec.priority}' 2>/dev/null)"
{ [ -n "$CRIT" ] && [ "$AP" = "$CRIT" ]; } && pass 'api dung bookstore-critical' || fail 'api sai priority'
ORPHAN="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"
MISS="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container co requests va limits' || fail "container thieu tai nguyen: $(echo $MISS)"

# 5. Service, DNS, proxy /api/, node-agent cùng node
[ "$(jp service/db '{.spec.clusterIP}')" = 'None' ] && pass 'Service db headless' || fail 'Service db khong headless'
[ "$(jp service/web '{.spec.ports[0].port}')" = '80' ] && pass 'Service web cong 80' || fail 'Service web sai cong'
[ "$(jp service/api '{.spec.ports[0].port}')" = '3000' ] && pass 'Service api cong 3000' || fail 'Service api sai cong'
[ "$(jp service/node-agent '{.spec.internalTrafficPolicy}')" = 'Local' ] \
  && pass 'Service node-agent internalTrafficPolicy Local' || fail 'Service node-agent sai'
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
WEB_NODE="$(jp "pod/$WEB" '{.spec.nodeName}')"
get() { k exec "$WEB" -c status-writer -- wget -q -T 5 -O- "$1" 2>/dev/null; }
get http://127.0.0.1:8080/ | grep -q 'Bookstore' && pass 'web phuc vu trang chu' || fail 'web khong phuc vu trang chu'
get http://127.0.0.1:8080/status.json | grep -Eq "\"pod\" *: *\"$WEB\"" \
  && pass 'web phuc vu status.json dung ten Pod' || fail 'status.json sai hoac thieu'
T="$(get http://127.0.0.1:8080/api/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "nginx proxy /api/books tra $T cuon sach" || fail "nginx proxy /api/books tra $T cuon sach"
T="$(get http://api:3000/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && pass "Service api theo ten DNS tra $T cuon sach" || fail "Service api tra $T cuon sach"
get http://node-agent:9100/node.json | grep -Eq "\"node\" *: *\"$WEB_NODE\"" \
  && pass 'node-agent qua Service tra dung node cua Pod web' || fail 'node-agent qua Service sai node'

# 6. NetworkPolicy: Pod lạ không gọi được api
OUT="$(k run "smoke-la-$$" --rm -i --quiet --restart=Never --image=busybox:1.37 --command -- \
        sh -c 'wget -q -T 3 -O- http://api:3000/books >/dev/null 2>&1 && echo MO || echo CHAN' 2>/dev/null)"
echo "$OUT" | grep -q 'CHAN' && pass 'Pod la khong goi duoc api (NetworkPolicy)' || fail "Pod la goi duoc api hoac khong chay: $OUT"

# 7. Ingress HTTPS qua Traefik
NP="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="websecure")].nodePort}' 2>/dev/null)"
[ -n "$NP" ] || NP="$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.port==443)].nodePort}' 2>/dev/null)"
IP1="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
[ "$(jp ingress/bookstore '{.spec.rules[0].host}')" = 'bookstore.lab.local' ] \
  && pass 'Ingress bookstore host bookstore.lab.local' || fail 'Ingress bookstore sai host'
curl -sk --max-time 10 --resolve "bookstore.lab.local:${NP}:${IP1}" "https://bookstore.lab.local:${NP}/" | grep -q 'Bookstore' \
  && pass 'trang chu qua Ingress HTTPS' || fail "khong vao duoc trang chu qua Ingress HTTPS (nodePort='$NP')"

# 8. Tác vụ nền: CronJob, Job, PriorityClass bookstore-batch, PVC backup
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = 'Forbid' ] && pass 'db-backup Forbid' || fail 'db-backup khong Forbid'
[ "$(jp cronjob/db-backup '{.spec.jobTemplate.spec.template.spec.priorityClassName}')" = 'bookstore-batch' ] \
  && pass 'db-backup dung bookstore-batch' || fail 'db-backup khong dung bookstore-batch'
if k get job db-migrate >/dev/null 2>&1; then
  { [ "$(jp job/db-migrate '{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ] && \
    [ "$(jp job/db-migrate '{.spec.template.spec.priorityClassName}')" = 'bookstore-batch' ]; } \
    && pass 'Job db-migrate Complete, dung bookstore-batch' || fail 'Job db-migrate chua Complete hoac sai PriorityClass'
else
  pass 'Job db-migrate da duoc TTL don, ket qua nam trong db'
fi
J="smoke-backup-$(date +%s)"
if k create job "$J" --from=cronjob/db-backup >/dev/null 2>&1 \
   && k wait --for=condition=Complete "job/$J" --timeout=180s >/dev/null 2>&1 \
   && k logs "job/$J" 2>/dev/null | grep -q '^db-backup: OK'; then
  pass 'chay thu db-backup thanh cong'
else
  fail 'chay thu db-backup that bai'
fi
k delete job "$J" --ignore-not-found --wait=true >/dev/null 2>&1
[ "$(jp pvc/backup-store '{.status.phase}')" = 'Bound' ] && pass 'PVC backup-store Bound' || fail 'PVC backup-store chua Bound'

# 9. DaemonSet node-agent trên mọi node
NODES="$(kubectl get nodes --no-headers | wc -l)"
R="$(jp daemonset/node-agent '{.status.numberReady}')"
[ "${R:-0}" -eq "$NODES" ] && pass "node-agent Ready tren $R/$NODES node" || fail "node-agent ${R:-0}/$NODES node"

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

2. Cập nhật `docs/tech-debt.md`: thêm dòng nợ mới của sprint với mã `TD-12` — mã cố định theo
   [mục 8 của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án):

```markdown
| TD-12 | Sau khi một worker quay lại, web và api không tự trải đều lại; phải phát hành lại bằng tay | 7a | 12 (bước trải lại trong quy trình bảo trì) | mở | docs/postmortem/7a-bao-tri-worker2.md |
```

3. Dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/7a/b10-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/7a/b10-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/7a/b10-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-7a: sprint 10 — dat workload dung cho'
git tag lab-7a
git bundle create ~/bookstore-lab-7a.bundle --all && git bundle verify ~/bookstore-lab-7a.bundle
```

Chép `~/bookstore-lab-7a.bundle` về máy host bằng `scp` trước khi đi tiếp.

5. Dọn object thử nghiệm và trả cluster:

```bash
kubectl -n bookstore delete job,deployment,pod -l '!app.kubernetes.io/part-of' --ignore-not-found --wait=true
kubectl delete runtimeclass bookstore-overhead-thu --ignore-not-found
kubectl taint node lab-k8s-worker2 bookstore.example.com/bao-tri- 2>/dev/null || true
rm -rf ~/lab-work/7a
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối:

```bash
cd ~/bookstore
SEL=app.kubernetes.io/part-of=bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-7a' && echo 'PASS: HEAD mang tag lab-7a'
test -s ~/bookstore-lab-7a.bundle && echo 'PASS: co bundle'
{ test -f k8s/cluster/priorityclass-batch.yaml && test -f docs/bao-cao-ton-kho.job.yaml && \
  test -f docs/runbook-bao-cao-ton-kho.md; } && echo 'PASS: bo cuc repo dung muc 5.5 va ticket B6'
BAD=0
for F in 'adr/*-hien-trang-lap-lich.md' 'adr/*-db-node-affinity.md' 'adr/*-trai-web-api.md' \
         'adr/*-priorityclass-batch.md' 'adr/*-bao-cao-cho-duyet.md' 'adr/*-hai-loai-eviction.md' \
         'adr/*-cau-hinh-scheduler.md' 'postmortem/7a-preemption-worker2.md' 'postmortem/7a-bao-tri-worker2.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR va postmortem cua sprint 10'
grep -q '| 7a |' docs/tech-debt.md && echo 'PASS: tech-debt.md ghi no phat sinh o 7a'
{ [ "$(kubectl get namespace,priorityclass,storageclass -l "$SEL" --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  [ "$(kubectl get all -A -l "$SEL" --no-headers 2>/dev/null | wc -l)" -eq 0 ]; } && echo 'PASS: khong con object nao cua du an'
[ -z "$(kubectl get pv -o jsonpath='{range .items[*]}{.spec.claimRef.namespace}{"\n"}{end}' | grep -x bookstore)" ] \
  && echo 'PASS: khong con PV nao cua du an'
{ [ -z "$(kubectl get nodes -l bookstore.example.com/data -o name)" ] && \
  ! kubectl get nodes -o jsonpath='{range .items[*]}{range .spec.taints[*]}{.key}{"\n"}{end}{end}' | grep -q '^bookstore\.example\.com/'; } \
  && echo 'PASS: khong node nao con label hay taint cua du an'
test -z "$(kubectl get nodes lab-k8s-worker1 lab-k8s-worker2 -o jsonpath='{.items[*].spec.taints}')" && echo 'PASS: hai worker sach taint'
[ "$(kubectl get node lab-k8s-master -o jsonpath='{.spec.taints[*].key}')" = 'node-role.kubernetes.io/control-plane' ] \
  && echo 'PASS: taint control plane con nguyen'
kubectl get runtimeclass bookstore-overhead-thu >/dev/null 2>&1 && echo 'FAIL: con RuntimeClass thu nghiem' \
  || echo 'PASS: khong con RuntimeClass thu nghiem'
test ! -e ~/lab-work/7a && echo 'PASS: da don lab-work'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
kubectl get storageclass local-path \
  -o jsonpath='{.metadata.annotations.storageclass\.kubernetes\.io/is-default-class}' | grep -qx true \
  && echo 'PASS: StorageClass mac dinh local-path con nguyen'
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; mười ba dòng `PASS:` đầu của gate
cuối xuất hiện, không có dòng `FAIL:`; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector
trả `No resources found`; CoreDNS đủ replica; namespace `default` không có Pod; dòng `PASS:
StorageClass mac dinh…`. Thư mục `~/lab-evidence/7a/` **giữ lại**. Cluster trở về
`03-storage-ready`; **không tạo snapshot mới**.

Một khác biệt được phép so với lúc bắt đầu: **vị trí** của vài Pod hạ tầng (Traefik,
local-path-provisioner, có thể cả CoreDNS) có thể đã chuyển khỏi `lab-k8s-worker2` do taint
`NoExecute` ở B7. Snapshot khai báo *hạ tầng nào có mặt*, không khai báo *Pod nằm ở node nào*, nên đây
không phải sai lệch.

---

## 3. Checkpoint 7a

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Kể hai bước của kube-scheduler bằng Pod `qua-co`: mỗi bước nhận gì, trả gì; bước nào không
      chạy và vì sao. `ghim-ten` khác ở chỗ nào, và nếu worker2 hết CPU thì `ghim-ten` gặp chuyện gì?
- [ ] `db-0` chịu hai ràng buộc node. Ai đặt từng cái, lúc nào được đánh giá? Gỡ label dữ liệu khi
      `db-0` đang chạy thì sao, và khi `db-0` bị thay thì sao? Gắn nhầm label sang worker1 thì sao?
- [ ] `web` với `maxSkew: 1`: phân bố 3 và 5 bản sao, phép tính loại 4/1. Vì sao không dùng
      anti-affinity cứng? Một `topologyKey` không node nào có dẫn tới gì?
- [ ] `api` dùng anti-affinity mềm: khi nào hai bản sao nằm chung một node, và vì sao dự án chấp nhận?
- [ ] `bookstore-batch` mang giá trị âm: trong hàng đợi nó đứng ở đâu, nó chiếm chỗ được ai, ai chiếm
      chỗ được nó? Vì sao chưa cần `preemptionPolicy: Never`? `bookstore-critical` có chiếm chỗ được
      CoreDNS không?
- [ ] Kể lại B5: vì sao nạn nhân là `lap-cho` và chỉ một Pod; `nominatedNodeName` nằm trên Pod nào và
      có phải lời hứa không; Pod `lap-cho` thay thế vì sao nằm chờ mà không đuổi được ai?
- [ ] `SchedulingGated` khác `Pending` thiếu chỗ ở đâu, bị giữ ở điểm mở rộng nào? Với Pod đang chờ
      duyệt bạn làm được gì và không làm được gì?
- [ ] Pha 1 và pha 2 của B7 làm gì với `db-0`, Pod `web`, `node-agent`, `ghim-ten`, `chiu-60s`,
      `chiu-mai`? Toleration không ghi `effect` khớp những gì?
- [ ] Vì sao phát hành `web` kẹt khi worker2 đóng cửa, `nodeTaintsPolicy: Honor` sửa điều gì, và vì sao
      sau bảo trì `web` không tự trải lại (`TD-12`)?
- [ ] Eviction qua API và eviction do áp lực node đối xử với PDB, grace period thế nào? Ý nghĩa của
      `200`, `429`, `500`. Trong sáu bước xóa Pod, bước nào Pod rời EndpointSlice?
- [ ] Ba tham số kubelet dùng để xếp hạng khi thiếu bộ nhớ, đúng thứ tự. Trên worker2 của bạn Pod nào
      của Bookstore đi trước, `db-0` đứng đâu? Kubelet có dùng QoS class không? Đổi đúng một ngưỡng
      cứng trong cấu hình kubelet thì các ngưỡng còn lại ra sao?
- [ ] Overhead được cộng vào ba chỗ nào? Vì sao `percentageOfNodesToScore` và `MostAllocated` không
      đáng đụng tới trên cluster này?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Lab 7a **không phát sinh và không trả** nợ lab nào; các phần không thực hành được ở mục 1.1 là giới
  hạn cố định của môi trường hoặc đã có chỗ ở giai đoạn sau. Nợ lab #1 (phần VPA), #5, #6, #7, #8 vẫn
  mở như [sổ nợ lab](README.md#5-sổ-nợ-lab).
- Nợ kỹ thuật: **`TD-12` mới** — `web`, `api` không tự trải lại sau bảo trì, trả ở Lab 12 bằng bước
  phát hành lại trong quy trình bảo trì. Các nợ còn mở từ sprint trước (ví dụ `TD-06` — `db` khởi
  động bằng root, trả ở Lab 9b) giữ nguyên.

### Bài giải thích cuối cùng

Trong vài phút, kể lại cuộc đời **chỗ ngồi** của `db-0` trong sprint này. Bắt đầu từ lúc
`scripts/deploy.sh` gắn label dữ liệu cho worker2 rồi apply StatefulSet: `db-0` được scheduler lọc qua
những bộ lọc nào (taint control plane, node affinity của Pod, `nodeAffinity` của PV), có được chấm
điểm không, ai ghi binding. Rồi đi qua ba biến cố của sprint: B5 — một Pod critical cần chỗ trên
worker2, ai bị hy sinh và vì sao không phải `web`; B7 — worker2 đóng cửa rồi bị dọn, `db-0` bị ai
đuổi, vì sao không đi đâu được, và nó quay lại thế nào với dữ liệu còn nguyên; B8 — nếu worker2 cạn
RAM, kubelet xếp `db-0` ở đâu so với các Pod khác. Kết thúc bằng việc chỉ ra Lab 7b (hạn mức namespace) và Lab 12
(bảo trì node có quy trình) sẽ thay đổi câu chuyện này ở điểm nào.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B0: một worker đã mang taint | Một phiên 7a dở dang còn sót taint bảo trì | `./scripts/teardown.sh` gỡ mọi taint `bookstore.example.com/*`; taint lạ khác thì restore cả ba VM về `03-storage-ready` |
| B0: `ssh ... 'sudo -n true'` hỏi mật khẩu | Worker chưa cho `sudo` không mật khẩu qua `ssh` | Dừng. Cấu hình `sudo` là quyết định của bạn; `teardown.sh` cần nó để dọn dữ liệu trên node |
| B1: không có sự kiện `Scheduled` | Sự kiện đã hết hạn — thời gian giữ phụ thuộc cấu hình apiserver | `./scripts/deploy.sh` lại: Job `db-migrate` tạo Pod mới nên có sự kiện mới |
| B1: thông điệp lọc không có `Insufficient cpu` | `qua-co` xin chưa đủ lớn, hoặc đọc nhầm đơn vị `m` | In lại `Allocatable` của từng node; xin hơn node lớn nhất ít nhất `500m` |
| B2: `deploy.sh` treo ở `rollout status statefulset/db` | `db-0` `Pending`: label dữ liệu chưa gắn, hoặc giá trị trong `values` không khớp `"true"` | `kubectl -n bookstore describe pod db-0`; `kubectl get nodes -L bookstore.example.com/data` |
| B2: apply StatefulSet báo lỗi kiểu dữ liệu ở `values` | Viết `values: [true]` không nháy — YAML hiểu là boolean | Viết `values: ["true"]` |
| B2: `db-0` `Pending` với `volume node affinity conflict` | Label dữ liệu nằm nhầm node; PV vẫn ghim worker2 | `./scripts/deploy.sh` — nó gắn label cho worker2 và gỡ khỏi node khác |
| B2: `teardown.sh` báo `thu-muc=` khác 0 | `ssh`/`sudo` tới worker lỗi, hoặc thư mục bị khóa | Chạy tay `ssh lab-k8s-worker2 'sudo ls /opt/local-path-provisioner'`; chỉ xóa thư mục mang `_bookstore_` |
| B3: ba Pod `web` cùng một node | Thiếu ràng buộc, `whenUnsatisfiable` là `ScheduleAnyway`, hoặc selector không khớp label Pod (pod ma) | So `labelSelector` của ràng buộc với label của Pod template |
| B3: `chong-don-cung` chạy đủ 3 Pod | Selector của anti-affinity không khớp label của chính nó | Sửa `matchLabels` cho trùng `app: chong-don-cung` |
| B4: `uu-tien-thu` lên worker2 | Viết nhầm `Exists` thay vì `DoesNotExist`, hoặc worker1 đang quá tải | `kubectl -n bookstore get pod uu-tien-thu -o yaml`; đọc phần *Allocated resources* của hai worker |
| B5: `FREE` nhỏ hơn `400m` | Còn Pod thử nghiệm của ticket trước trên worker2 | `kubectl get pods -A -o wide --field-selector spec.nodeName=lab-k8s-worker2`; xóa Pod thử nghiệm |
| B5: không bắt được `nominatedNodeName` | Nạn nhân chết quá nhanh | `lap-cho` phải có `trap '' TERM` và `terminationGracePeriodSeconds: 30` |
| B5: nạn nhân là Pod `web` hay Pod hạ tầng | `bookstore-batch` không mang giá trị âm | Sửa giá trị trong `k8s/cluster/`, `./scripts/deploy.sh`, chạy lại B5 |
| B5: `can-cho-gap` chờ mãi, không ai bị chiếm chỗ | `HIGH` lớn hơn cả chỗ trống cộng một Pod `lap-cho`, hoặc `lap-cho` không nằm trên worker2 | In lại `b5-so-lieu.txt`; kiểm `nodeSelector` của hai manifest |
| B6: Pod hiện `Pending` chứ không `SchedulingGated` | `schedulingGates` sai thụt lề, không vào được template | `kubectl apply --dry-run=server -f docs/bao-cao-ton-kho.job.yaml -o yaml` rồi đọc template |
| B6: Job `Failed`, log báo timeout kết nối | NetworkPolicy của Lab 5b không nhận Pod báo cáo — label không khớp thành phần `backup` | So label của Pod với `podSelector` của policy cho `db`; không sửa policy, sửa label của Pod |
| B6: thêm `nodeSelector` bị từ chối | Cổng đã được gỡ — Pod không còn chờ duyệt | Đúng hành vi; tạo lại yêu cầu và siết **trước** khi duyệt |
| B7: phát hành `web` ở pha 1 hoàn tất ngay | Ràng buộc đã có `nodeTaintsPolicy: Honor` từ trước | Ghi vào postmortem; bỏ qua khối "sau khi sửa" |
| B7: `db-0` không quay lại sau khi gỡ taint | Còn sót một effect của key bảo trì | `kubectl taint node lab-k8s-worker2 bookstore.example.com/bao-tri-` gỡ mọi effect |
| B7: Pod hạ tầng `Pending` trong pha 2 | Worker1 hết chỗ khi nhận Pod chuyển sang | Gỡ taint trước; **không** gỡ taint control plane để lấy chỗ |
| B7: `smoke.sh` báo `web` lệch sau bảo trì | Phân bố không tự cân bằng lại (`TD-12`) | `kubectl -n bookstore rollout restart deployment/web deployment/api`; hoặc `teardown.sh && deploy.sh` |
| B8: eviction đầu tiên bị từ chối | PDB `web` đang có ngân sách 0 — chưa đủ bản sao Ready | Chờ `web` đủ Ready rồi chạy lại |
| B8: mọi eviction đều được chấp nhận | PDB `web` của Lab 3c cho phép gián đoạn bằng hoặc hơn số bản sao | Ghi vào ADR; xem lại PDB — một PDB như vậy không bảo vệ gì trong `drain` ở Lab 12 |
| B8: `kubectl get --raw .../proxy/configz` bị từ chối | Không dùng kubeconfig quản trị | Dùng kubeconfig quản trị của Lab 00 trên `lab-k8s-master`; không tạo thêm quyền |
| B8: `evictionHard` khác bộ mặc định | Kubelet đã bị sửa ngoài quy trình Lab 00 | Không sửa ở đây; ghi vào ADR và đọc B8 với đúng con số của bạn — sửa kubelet thuộc giai đoạn 20 |
| B9: tạo `overhead-thu` bị từ chối `RuntimeClass ... not found` | Chưa tạo RuntimeClass, hoặc gõ khác tên | Apply `rc-overhead.yaml` trước; so `runtimeClassName` với `kubectl get runtimeclass` |
| B9: `overhead-thu` kẹt `ContainerCreating`, sự kiện nhắc handler | Handler `runc` không có trong cấu hình containerd | Đọc lại cấu hình runtime ở B7 của [Lab 2](LAB-2-CONTAINER-IMAGE-CRI-VA-CGROUP.md); đổi `handler` cho đúng tên đang có |
| B9: không đọc được `memory.max` | Pod không phải Guaranteed nên cgroup nằm dưới `kubepods-burstable.slice` | Đặt `requests` bằng `limits` cho cả CPU lẫn bộ nhớ |
| B10: `smoke.sh` FAIL ở Ingress với `nodePort=''` | Service của Traefik đặt tên port khác | `kubectl -n traefik get svc traefik -o yaml`; script đã thử thêm theo cổng 443 |
| B10: namespace kẹt `Terminating` khi teardown | `db-0` đang dùng ân hạn khi tắt | Chờ; `teardown.sh` chờ tối đa 180 giây cho `db-0` và 300 giây cho namespace |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Scheduling, Preemption and Eviction](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/)
- [Kubernetes v1.35 — Kubernetes Scheduler](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
- [Kubernetes v1.35 — Assigning Pods to Nodes](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
- [Kubernetes v1.35 — Taints and Tolerations](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
- [Kubernetes v1.35 — Pod Topology Spread Constraints](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
- [Kubernetes v1.35 — Pod Priority and Preemption](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [Kubernetes v1.35 — Node-pressure Eviction](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [Kubernetes v1.35 — API-initiated Eviction](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/api-eviction/)
- [Kubernetes v1.35 — Pod Overhead](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/pod-overhead/)
- [Kubernetes v1.35 — Pod Scheduling Readiness](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/)
- [Kubernetes v1.35 — Scheduler Performance Tuning](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/)
- [Kubernetes v1.35 — Scheduling Framework](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Kubernetes v1.35 — Resource Bin Packing](https://v1-35.docs.kubernetes.io/docs/concepts/scheduling-eviction/resource-bin-packing/)
- [Kubernetes v1.35 — Assign Pods to Nodes](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes/)
- [Kubernetes v1.35 — Assign Pods to Nodes using Node Affinity](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/)
- [Kubernetes v1.35 — Guaranteed Scheduling For Critical Add-On Pods](https://v1-35.docs.kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)
- [Kubernetes v1.35 — Disruptions, mục Pod disruption conditions](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-conditions)
- [Kubernetes v1.35 — Runtime Class](https://v1-35.docs.kubernetes.io/docs/concepts/containers/runtime-class/)
- [Kubernetes v1.35 — Jobs, mục Mutable Scheduling Directives](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/job/)
- [Kubernetes — Eviction API reference](https://kubernetes.io/docs/reference/kubernetes-api/policy-resources/eviction-v1/)
- [Kubernetes — kube-scheduler configuration (v1)](https://kubernetes.io/docs/reference/config-api/kube-scheduler-config.v1/)
- [Kubernetes — Kubelet configuration (v1beta1)](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [Kubernetes — Well-Known Labels, Annotations and Taints](https://kubernetes.io/docs/reference/labels-annotations-taints/)
- [PostgreSQL 17 — psql](https://www.postgresql.org/docs/17/app-psql.html)
