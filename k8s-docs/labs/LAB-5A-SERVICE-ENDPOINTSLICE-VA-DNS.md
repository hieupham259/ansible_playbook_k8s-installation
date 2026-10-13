# Lab 5a — Service, EndpointSlice và DNS

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> Repo `~/bookstore` ở tag `lab-4b` (hoặc bundle `~/bookstore-lab-4b.bundle` nếu bạn vừa restore
> snapshot).
> **Điểm kết thúc:** teardown trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới. Repo
> `~/bookstore` có tag `lab-5a` và một bản bundle nằm ngoài VM.
> **Sprint dự án:** Sprint 6 — Nối các tầng của [dự án Bookstore](DU-AN-BOOKSTORE.md).
> **Lab trước:** [Lab 4b — StatefulSet, DaemonSet và Job](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md).
> **Lab sau:** [Lab 5b — NetworkPolicy, Ingress và CNI](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md).
>
> **Cập nhật và đối chiếu phiên bản:** 13/10/2026.

Lab này đi cùng phần **Service/DNS** của
[Giai đoạn 5 — Mạng nền tảng](../00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng): chín bài khái
niệm 81, 82, 83, 10, 57, 84, 86, 87, 88 và mười bài của dòng **Thực hành**. Phần NetworkPolicy
được thực thi thật, Ingress và CNI thuộc [Lab 5b](LAB-5B-NETWORKPOLICY-INGRESS-VA-CNI.md) — lab đó đổi CNI nên tạo
snapshot mới; lab này **không** đổi CNI, không cài MetalLB hay ingress controller, không sửa CoreDNS.

Mọi ticket dựa vào hợp đồng kỹ thuật ở [mục 4 của file dự án](DU-AN-BOOKSTORE.md#4-hợp-đồng-kỹ-thuật)
và bố cục repo ở [mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint). Kế hoạch gốc của sprint
nằm ở [mục 7.2](DU-AN-BOOKSTORE.md#sprint-6--lab-5a-nối-các-tầng).

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa). Image của dự án nằm
ở [bảng A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00); lab này không chép
lại con số phiên bản nào trong phần văn bản và **không cài thêm gì** vào cluster.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`. B0 bên dưới chạy lại đúng gate đó.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi hệ thống Bookstore đạt trạng thái sau sprint 6 — **trang web gọi được API qua
NodePort từ máy host, các tầng tìm nhau bằng tên** — **và** bạn chứng minh, giải thích được:

- `api` tìm `db` bằng `db-0.db`, `web` tìm `api` bằng `api`, `status-writer` tìm `node-agent` bằng
  `node-agent`; xóa `db-0` thì hệ thống tự lành mà không ai phải chạy lại `deploy.sh` (`TD-03` trả).
- Service chọn **Pod** bằng selector, không chọn controller; kết quả được ghi ra EndpointSlice;
  ClusterIP là địa chỉ ảo; `targetPort` theo tên được giải ra số thật, và một Service có thể có nhiều
  EndpointSlice nên phải gộp mới đọc đúng.
- Ba condition `ready`, `serving`, `terminating`: Pod NotReady **vẫn nằm** trong slice nhưng không
  nhận request; Pod đang tắt mang `terminating=true` trước khi container dừng.
- Định danh của `db-0` gồm phần nào đã có từ Lab 4b và phần nào Service quản trị headless thêm vào
  (nợ lab #3).
- DNS trong Pod: `nameserver`, `search`, `ndots:5`, FQDN, ranh giới namespace, `hostname` và
  `subdomain`; biến môi trường so với DNS khi khám phá Service.
- `NodePort` mở cùng một cổng trên mọi node; `LoadBalancer` treo `<pending>` trên bare metal;
  `ExternalName` chỉ là CNAME; `externalTrafficPolicy` và `internalTrafficPolicy` mỗi trường lọc tập
  endpoint nào, cho loại lưu lượng nào.
- Kustomize đầy đủ: bộ sinh ConfigMap có hậu tố hash biến việc đổi cấu hình thành một rollout; vì
  sao Secret vẫn giữ tên cố định; vì sao `labels` an toàn hơn `commonLabels`.
- Canary chia lưu lượng thật theo tỷ lệ bản sao nhờ một selector chung.
- NetworkPolicy theo ma trận của dự án được API chấp nhận nhưng không chặn gì trên Flannel.
- ClusterIP lấy từ đâu; định tuyến theo topology bật được mà không có tác dụng; `port-forward` là
  đường gỡ lỗi; hệ thống compose cũ ánh xạ sang object nào.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm Service/DNS | Ticket kiểm chứng |
| --- | --- |
| [81 — Service, cân bằng tải và mạng](../81-services-networking-vi.md) | B1 — Pod thấy đúng `.status.podIP` của mình, ClusterIP do kube-proxy chứ không do interface nào; B8 — API NetworkPolicy tồn tại nhưng hiện thực mạng Pod không thực thi |
| [82 — Service](../82-service-vi.md) | B1 — selector, `targetPort` theo tên, ClusterIP, headless; B3 — `NodePort` cố định; B6 — khám phá bằng biến môi trường và DNS; B9 — `LoadBalancer`, `ExternalName`; B10 ⭐ — Service không selector |
| [83 — EndpointSlices](../83-endpoint-slices-vi.md) | B1 — slice tự sinh, `managed-by`, owner; B5 — ba condition, nhiều slice cho một Service, gộp và khử trùng lặp; B10 ⭐ — slice tự viết |
| [10 — DNS cho Service và Pod](../10-dns-pod-service-vi.md) | B2 — các tầng gọi nhau bằng tên; B6 — `resolv.conf`, `search`, `ndots:5`, namespace, bản ghi cấp Pod |
| [57 — Hostname của Pod](../57-pod-hostname-vi.md) | B6 — `hostname`/`subdomain` của `db-0` và FQDN trong Pod; B12 — hostname xác định trước của Indexed Job |
| [84 — Chính sách mạng](../84-network-policies-vi.md) | B8 — NetworkPolicy theo ma trận mục 3.5, được API chấp nhận, không có hiệu lực trên Flannel (**nợ lab #4**) |
| [86 — Định tuyến nhận biết topology](../86-topology-aware-routing-vi.md) | B11 — bật annotation trên Service thử nghiệm, chứng minh cơ chế bảo vệ số 3 chặn: không node nào có zone nên không có `hints` |
| [87 — Chính sách lưu lượng nội bộ](../87-service-traffic-policy-vi.md) | B4 — `node-agent` với `internalTrafficPolicy: Local`, và cảnh báo "node không có endpoint thì như không có endpoint nào" đo trên `lab-k8s-worker2` |
| [88 — Cấp phát ClusterIP](../88-cluster-ip-allocation-vi.md) | B11 — dải Service thật, công thức băng tĩnh, ClusterIP tĩnh được chấp nhận và ngoài dải bị từ chối |
| [294 — Chuyển Docker Compose sang Kubernetes](../294-translate-compose-kubernetes-vi.md) | B13 — đối chiếu bằng tay file compose cũ ở mục 2.1 với hệ thống sau sprint 6 theo các quy tắc chuyển đổi của bài; **không** cài Kompose (binary ngoài baseline) |
| [320 — Lệnh imperative](../320-imperative-command-vi.md) | B1 — `kubectl expose` và `kubectl create service` ở chế độ dry-run so với manifest khai báo; B11 — pipeline `create --dry-run` → `set --local` → `create -f -` |
| [322 — Kustomize](../322-kustomization-vi.md) | B2 — kustomization gom cả namespace, `configMapGenerator`, `namespace`, `labels`; B3 — hậu tố hash làm đổi cấu hình thành rollout; B8 ⭐ — `labels` so với `commonLabels` |
| [340 — Xóa một StatefulSet](../340-delete-stateful-set-vi.md) | B12 — xóa StatefulSet thử nghiệm đúng trình tự của bài; Service quản trị không đi theo |
| [352 — Hàng đợi công việc mịn](../352-fine-parallel-work-queue-vi.md) | **Không thực hành.** Bài cần image Redis (ngoài A1.4) và một image worker tự build, tự push — lab không build image. B6 dùng đúng điểm của bài thuộc giai đoạn này: worker tìm hàng đợi bằng tên Service, đường lui là biến môi trường (câu hỏi thiết kế 3) |
| [354 — Job với giao tiếp Pod-đến-Pod](../354-job-pod-to-pod-communication-vi.md) | B12 — Indexed Job, headless Service chọn theo `job-name`, `subdomain`; Job chỉ xong khi mọi Pod gọi được nhau bằng tên |
| [362 — Cấu hình DNS cho cluster](../362-configure-dns-cluster-vi.md) | B6 — addon DNS là CoreDNS do kubeadm cài, Service tên `kube-dns`, chính là `nameserver` trong Pod; chỉ đọc, không sửa |
| [363 — Kết nối frontend với backend](../363-connecting-frontend-backend-vi.md) | B3 — nginx của `web` gọi Service `api` bằng tên, `web` ra ngoài bằng `NodePort` đúng như bài cho phép khi không có load balancer |
| [364 — Tạo bộ cân bằng tải bên ngoài](../364-create-external-load-balancer-vi.md) | B1 — `expose` dùng lại selector của Deployment; B9 — `LoadBalancer` treo `<pending>`, `externalTrafficPolicy` `Cluster` so với `Local`, `healthCheckNodePort` |
| [366 — Port forwarding](../366-port-forward-vi.md) | B10 — `port-forward` tới Service `db`, chứng minh đầu bên kia là PostgreSQL; chỉ nghe trên loopback |

Bốn phần **không kiểm chứng được trên cluster lab**, đọc để biết:

| Phần | Vì sao không có bước thực hành |
| --- | --- |
| Đường dữ liệu thật của `type: LoadBalancer` (bài 82, 363, 364) | Cần bộ cân bằng tải ngoài cluster và controller của nhà cung cấp. MetalLB nằm [ngoài baseline](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00); cài nó là đổi hạ tầng. B9 chỉ kiểm phần API |
| Hiệu ứng định tuyến của bài 86 | Cần cluster nhiều zone. B11 kiểm phần cấu hình được và chứng minh hint không được sinh |
| `hostnameOverride` của bài 57 | Tính năng beta ở baseline; chính bài nói nó không đổi bản ghi DNS của Pod — thứ lab này đo |
| Tùy chỉnh CoreDNS (bài 362 trỏ sang bài 204) | Sửa Corefile là đụng `kube-system`, thuộc giai đoạn 21; B6 chỉ đọc addon mặc định |

Lab này **trả** ba món nợ — đọc lại bài gốc trước ticket tương ứng:

| Nợ | Phát sinh ở | Trả ở | Đọc lại trước khi làm |
| --- | --- | --- | --- |
| Nợ lab #3 — Service headless quản trị cho StatefulSet | Lab 4b, bài 65 | B1 | Bài [65](../65-statefulset-vi.md), mục [*Định danh mạng ổn định*](../65-statefulset-vi.md#stable-network-id) và [*Hạn chế*](../65-statefulset-vi.md#limitations) |
| Readiness thất bại gỡ Pod khỏi EndpointSlice (nợ lab của 3a) | Lab 3a, bài 49 | B5 | Bài [49](../49-probes-vi.md), mục [*Readiness probe*](../49-probes-vi.md#readiness-probe) |
| Canary chia lưu lượng thật qua Service (nợ lab của 4a) | Lab 4a, bài 61 | B7 | Bài [61](../61-management-vi.md), mục [*Triển khai canary*](../61-management-vi.md#canary-deployments) |

Nợ còn **mở** sau sprint — nói được vì sao ở checkpoint:

| # | Phần hoãn | Vì cần | Trả ở |
| --- | --- | --- | --- |
| 4 | NetworkPolicy được thực thi thật (bài 84) | CNI hỗ trợ policy thay Flannel | Lab 5b |
| 1 | Thực hành HPA, VPA | metrics-server; VPA còn cần add-on ngoài A1.4 | HPA ở Lab 11b |
| 2 | `volumeClaimTemplates` cho `db` | StorageClass và provisioner | Lab 6a |

Nợ kỹ thuật của hệ thống sau sprint: **trả `TD-03`**; **phát sinh `TD-09`** (lưu lượng giữa các tầng
chưa bị giới hạn); `TD-04`, `TD-06`, `TD-07`, `TD-08` vẫn mở — xem
[sổ nợ của file dự án](DU-AN-BOOKSTORE.md#8-sổ-nợ-kỹ-thuật-dự-án).

### 1.2. Thời lượng

4–5 giờ. Phần cốt lõi (B0–B8, B13, B14) khoảng 4 giờ; B9–B12 là điều tra ngắn, phần ⭐ là tùy chọn.
Có thể chia hai phiên: phiên 1 làm B0–B6, phiên 2 làm B7–B14. Giữa hai phiên cứ để nguyên cluster;
đầu phiên 2 chạy lại gate ngắn A5.5 và `./scripts/deploy.sh`.

---

## 2. Quy ước và an toàn

- Mọi lệnh `kubectl` chạy trên `lab-k8s-master` trừ khi ghi rõ node khác. Bước nào gọi từ máy host
  Windows thì ghi rõ PowerShell.
- **Fault injection chỉ trên `lab-k8s-worker2`.** `db-0` nằm ở đó theo
  [thiết kế dự án](DU-AN-BOOKSTORE.md#36-ngân-sách-tài-nguyên-và-lập-lịch). Pod diễn tập `web-le`
  (B5) và client làm hỏng lời gọi (B4) đều ghim `lab-k8s-worker2`.
- Repo dự án: `~/bookstore`. Manifest thử nghiệm: `~/lab-work/5a/`. Bằng chứng: `~/lab-evidence/5a/`.
- Lab dùng kiến thức tới hết phần Service/DNS của giai đoạn 5. **Không** dựng Ingress, không đổi
  CNI, không cài load balancer, không sửa CoreDNS hay kube-proxy, không tạo PVC, Role hay HPA.
- Object thử nghiệm nằm trong namespace `bookstore` và mang label `lab: <tên>`. Chúng **không** mang
  label `app.kubernetes.io/part-of` ở `metadata` của chính object; riêng Pod phải lọt vào selector
  của Service `web` (B5, B7) thì buộc mang cặp label selector đó — vì thế xóa chúng ngay khi xong
  ticket. Lệnh liệt kê Pod `web` thật trong script dùng selector `'...component=web,!lab'`.
- Từ B2, mọi thay đổi hệ thống đi qua repo rồi `./scripts/deploy.sh`. Không `kubectl apply -k`, không
  `kubectl edit` hay `patch` lên object của dự án.
- Sau B2, `tests/smoke.sh` của Lab 4b báo `FAIL` ở ba dòng ConfigMap vì tên đã có hậu tố hash — đúng
  dự kiến, [mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label) cho phép điều đó. Bước đóng sprint thay
  bằng phiên bản đọc tên từ spec.
- NodePort `30080` thuộc dự án. Service thử nghiệm không xin cổng node cố định.
- Lab **chỉ đọc** `/etc/kubernetes/manifests/kube-apiserver.yaml` (B11, cần `sudo`).
- Node không có `jq`: script phân tích JSON bằng `python3`, gọi HTTP bằng `curl` trên master và
  `wget` của busybox trong Pod.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail. Dòng `FAIL:` trong script nghiệm
  thu nghĩa là ticket chưa xong.

---

# Phần B — Sprint 6: Nối các tầng

## B0. Nhận bàn giao sprint

**Mục đích:** xác nhận cluster ở đúng `01-cluster-ready`, repo ở đúng tag `lab-4b`, và hệ thống
của sprint 5 dựng lại được **chỉ từ repo**.

Gate ngắn A5.5, chạy nguyên văn:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

Repo, công cụ và thư mục làm việc. Nếu vừa restore snapshot, repo đã mất — khôi phục từ bundle theo
[mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm) trước:

```bash
mkdir -p ~/lab-work/5a ~/lab-evidence/5a
test -d ~/bookstore/.git || git clone ~/bookstore-lab-4b.bundle ~/bookstore
cd ~/bookstore
git describe --tags --exact-match HEAD 2>/dev/null | grep -qx 'lab-4b' && echo 'PASS: HEAD dung tag lab-4b'
test -z "$(git status --porcelain)" && echo 'PASS: repo khong co thay doi chua commit'
kubectl get namespace bookstore >/dev/null 2>&1 \
  && echo 'FAIL: namespace bookstore con sot tu lan truoc' || echo 'PASS: cluster chua co du an'
kubectl get services -A -o jsonpath='{range .items[*]}{range .spec.ports[*]}{.nodePort}{"\n"}{end}{end}' \
  | grep -qx 30080 && echo 'FAIL: NodePort 30080 dang bi chiem' || echo 'PASS: NodePort 30080 con trong'
curl --version >/dev/null && python3 --version && echo 'PASS: co curl va python3 tren master'
```

Dựng lại hệ thống của sprint 5 bằng script của chính sprint đó:

```bash
cd ~/bookstore
./scripts/deploy.sh > ~/lab-evidence/5a/b0-deploy.txt 2>&1; echo "deploy exit=$?"
./tests/smoke.sh > ~/lab-evidence/5a/b0-smoke-lab-4b.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/5a/b0-smoke-lab-4b.txt
```

Kéo sẵn image lên các node để thời gian kéo image không lẫn vào các phép đo ở B5. Tag lấy đúng từ
[A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa) và
[A1.4](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00):

```bash
sudo crictl pull docker.io/library/busybox:1.37 >/dev/null && echo 'PASS: lab-k8s-master co busybox'
for N in lab-k8s-worker1 lab-k8s-worker2; do
  for I in docker.io/library/busybox:1.37 docker.io/library/postgres:17.11 \
           docker.io/postgrest/postgrest:v14.18 docker.io/nginxinc/nginx-unprivileged:1.30.5-alpine; do
    ssh "$N" "sudo crictl pull $I" >/dev/null && echo "PASS: $N co $I" || echo "FAIL: $N khong keo duoc $I"
  done
done
```

**PASS:** gate A5.5 đạt như mô tả ở Lab 00; năm dòng `PASS:` về repo, cluster, cổng và công cụ;
`deploy exit=0`, `smoke exit=0` và file smoke không có dòng `FAIL:`; đủ chín dòng `PASS: ... co ...`.

`deploy.sh` hoặc `smoke.sh` fail nghĩa là repo của sprint 5 **chưa đủ** — sửa repo ở tag `lab-4b`
trước, không sửa tay trên cluster ([mục 6.1 của file dự án](DU-AN-BOOKSTORE.md#61-mở-đầu)).

## B1. [BS-5A-01] Service cho ba tầng và Service quản trị `db`

**Loại:** Tính năng — trả nợ lab #3.

**Bối cảnh:** tới sprint 5, mọi tầng gọi nhau bằng IP Pod chèn lúc triển khai (`TD-03`). StatefulSet
`db` đã khai `serviceName: db` từ Lab 4b nhưng Service đó chưa tồn tại, nên `db-0` có hostname mà
không có tên DNS (nợ lab #3). Ticket này cho mỗi tầng một địa chỉ ổn định; **chưa** đổi cách các
tầng gọi nhau — đó là B2. **Đọc lại bài [65](../65-statefulset-vi.md), mục
[*Định danh mạng ổn định*](../65-statefulset-vi.md#stable-network-id), trước khi làm ticket này.**

**Yêu cầu:**

1. Ba Service theo [mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe):
   `db` **headless**, cổng 5432; `api` cổng 3000; `web` cổng 80 (chỉ trong cluster ở ticket này, B3
   mở ra ngoài). Cổng đích tham chiếu **tên** cổng đã khai trên Pod, không tham chiếu số.
2. Selector và label theo [mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label). Manifest ở
   `k8s/app/<thành phần>/service.yaml`, thêm vào `k8s/app/kustomization.yaml`, triển khai bằng
   `./scripts/deploy.sh` hiện có của Lab 4b.
3. Tên `db-0.db.bookstore.svc.cluster.local` phân giải ra IP của `db-0`, mà manifest StatefulSet `db`
   không phải sửa dòng nào.
4. Chứng minh trên hệ thống thật: control plane tự sinh EndpointSlice cho mỗi Service, địa chỉ trong
   slice trùng IP Pod, tên cổng được giải ra số; ClusterIP không nằm trên interface nào của node mà
   TCP tới nó vẫn tới backend; một ClusterIP của `web` trải lưu lượng ra nhiều Pod.
5. Mô hình mạng của bài 81: địa chỉ một Pod tự thấy trùng `.status.podIP` mà API server báo.
6. Trước khi viết manifest, cho `kubectl` sinh thử Service bằng `expose` và bằng `create service` ở
   chế độ không gửi lên cluster; ghi khác biệt so với hợp đồng. Tạo **thật** một Service bằng
   `create service` để thấy nó không có endpoint nào, rồi xóa.

**Ràng buộc:** Service của dự án tạo bằng manifest trong repo. Không sửa StatefulSet `db`. Chưa đổi
địa chỉ DB của `api`, Job, CronJob — vẫn là IP do `deploy.sh` chèn.

**Đọc lại:**

- Bài [65](../65-statefulset-vi.md): [*Định danh mạng ổn định*](../65-statefulset-vi.md#stable-network-id),
  [*Hạn chế*](../65-statefulset-vi.md#limitations) — Service quản trị là thứ **bạn** phải tạo.
- Bài [82](../82-service-vi.md): [*Định nghĩa port*](../82-service-vi.md#field-spec-ports),
  [`type: ClusterIP`](../82-service-vi.md#type-clusterip),
  [*Headless Service*](../82-service-vi.md#headless-services).
- Bài [83](../83-endpoint-slices-vi.md): [*EndpointSlice API*](../83-endpoint-slices-vi.md#endpointslice-resource),
  [*Quản lý*](../83-endpoint-slices-vi.md#quản-lý-management),
  [*Quyền sở hữu*](../83-endpoint-slices-vi.md#quyền-sở-hữu-ownership).
- Bài [81](../81-services-networking-vi.md#the-kubernetes-network-model) — mô hình mạng và ranh giới
  trách nhiệm.
- Bài [320](../320-imperative-command-vi.md#cách-tạo-object-how-to-create-objects) — hai họ lệnh tạo
  object; bài [364](../364-create-external-load-balancer-vi.md#tạo-một-service-create-a-service) —
  `expose` dùng lại selector của tài nguyên được tham chiếu.

**Bàn giao:** `k8s/app/{db,api,web}/service.yaml`; `k8s/app/kustomization.yaml`;
`docs/adr/NNNN-service-ba-tang.md` (`NNNN` là số tiếp theo trong `docs/adr/`);
`~/lab-evidence/5a/b1-*`.

**Nghiệm thu:**

- `PASS:` `db` headless cổng 5432; `api` ClusterIP 3000 → `http`; `web` 80 → `http`; selector và
  label theo hợp đồng.
- `PASS:` EndpointSlice tự sinh, địa chỉ sẵn sàng trùng IP Pod: 2 `api`, 3 `web`, 1 `db`; slice ghi
  cổng số 3000 và 8080; do control plane quản lý, thuộc sở hữu Service.
- `PASS:` ClusterIP của `api` không nằm trên interface nào của master, gọi tới nó vẫn ra sách; Service
  `web` trải lưu lượng ra ít nhất hai Pod; địa chỉ Pod tự thấy trùng `.status.podIP`.
- `PASS:` `db-0.db.bookstore.svc.cluster.local` ra IP của `db-0`; tên Service `db` ra IP Pod chứ không
  ra ClusterIP; `statefulset.yaml` không đổi so với tag `lab-4b`.
- `PASS:` `expose` chép selector của Deployment `api`; `create service` tự đặt selector `app=<tên>`;
  Service tạo thật bằng lệnh đó không có endpoint nào và đã bị xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-service-ba-tang.md`:

1. Vì sao `db` là headless mà `api` thì không? Nếu `db` là ClusterIP thường thì tên `db-0.db` có tồn
   tại không, và dự án mất gì ở Lab 6a?
2. Manifest StatefulSet `db` không đổi một dòng mà `db-0` có tên DNS. Định danh của `db-0` gồm những
   phần nào đã có từ Lab 4b, phần nào Service quản trị vừa thêm vào?
3. Một ngày PostgREST đổi cổng chính. Nhờ `targetPort` theo tên, bạn sửa những file nào, và client
   nào **không** phải sửa?
4. `kubectl expose` và `kubectl create service` khác nhau ở chỗ lấy selector. Vì sao dự án vẫn viết
   manifest thay vì dùng một trong hai lệnh?

<details>
<summary>Gợi ý</summary>

1. Headless là Service `ClusterIP` có `.spec.clusterIP` đặt đúng chuỗi `None` — khác với bỏ trống.
2. Tên cổng trên Pod đã có từ 3a: `http` cho `web` và `api`, `postgres` cho `db`.
3. Gọi tên DNS từ một Pod có `nslookup`, `wget`, `ip`: container `status-writer` của `web`.
   `kubectl -n bookstore exec deploy/web -c status-writer -- nslookup <tên>`.
4. Dry-run:
   `kubectl -n bookstore expose deployment api --name=api-thu --port=3000 --target-port=http --dry-run=client -o yaml`;
   `kubectl -n bookstore create service clusterip api-thu --tcp=3000:http --dry-run=client -o yaml`.
5. Đọc slice của một Service: `kubectl -n bookstore get endpointslice -l kubernetes.io/service-name=api -o yaml`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a
cd ~/bookstore
svc() { kubectl -n "$NS" get service "$1" -o jsonpath="$2" 2>/dev/null; }
eps() { kubectl -n "$NS" get endpointslice -l "kubernetes.io/service-name=$1" -o jsonpath="$2" 2>/dev/null; }
ready_ips() { eps "$1" '{range .items[*].endpoints[*]}{.addresses[0]} {.conditions.ready}{"\n"}{end}' \
  | awk '$2=="true"{print $1}' | sort -u; }
pod_ips() { kubectl -n "$NS" get pod -l "app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=$1" \
  -o jsonpath='{range .items[*]}{.status.podIP}{"\n"}{end}' | sort -u; }

./scripts/deploy.sh > "$EV/b1-deploy.txt" 2>&1 && echo 'PASS: deploy.sh trien khai duoc ca ba Service'
kubectl -n "$NS" get service,endpointslice -o wide > "$EV/b1-services.txt"

# 1. Hợp đồng
{ [ "$(svc db '{.spec.clusterIP} {.spec.ports[0].port}')" = 'None 5432' ] && \
  [ "$(svc api '{.spec.type} {.spec.ports[0].port} {.spec.ports[0].targetPort}')" = 'ClusterIP 3000 http' ] && \
  [ "$(svc web '{.spec.ports[0].port} {.spec.ports[0].targetPort}')" = '80 http' ]; } \
  && echo 'PASS: db headless 5432, api ClusterIP 3000 -> http, web 80 -> http'
BAD=0
for S in db api web; do
  [ "$(svc "$S" '{.spec.selector.app\.kubernetes\.io/part-of}/{.spec.selector.app\.kubernetes\.io/component}')" = "bookstore/$S" ] \
    || { echo "FAIL: selector cua $S khong theo hop dong"; BAD=1; }
  [ "$(svc "$S" '{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ] || { echo "FAIL: $S thieu label du an"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: selector va label cua ba Service theo hop dong'

# 2. EndpointSlice
OK=1
for X in api:2 web:3 db:1; do
  S="${X%%:*}"; N="${X##*:}"
  { [ "$(ready_ips "$S")" = "$(pod_ips "$S")" ] && [ "$(ready_ips "$S" | grep -c .)" -eq "$N" ]; } \
    || { echo "FAIL: EndpointSlice cua $S khong khop Pod"; OK=0; }
done
[ "$OK" -eq 1 ] && echo 'PASS: EndpointSlice tu sinh, trung IP Pod: 2 api, 3 web, 1 db san sang'
{ [ "$(eps api '{.items[0].ports[0].port}')" = '3000' ] && [ "$(eps web '{.items[0].ports[0].port}')" = '8080' ]; } \
  && echo 'PASS: ten cong http duoc giai ra so that trong EndpointSlice (3000, 8080)'
{ [ "$(eps api '{.items[0].metadata.labels.endpointslice\.kubernetes\.io/managed-by}')" = 'endpointslice-controller.k8s.io' ] && \
  [ "$(eps api '{.items[0].metadata.ownerReferences[0].kind}/{.items[0].metadata.ownerReferences[0].name}')" = 'Service/api' ]; } \
  && echo 'PASS: slice do control plane quan ly va thuoc so huu Service api'

# 3. ClusterIP ảo, cân bằng tải, mô hình mạng
WEB="$(kubectl -n "$NS" get pod -l app.kubernetes.io/component=web -o jsonpath='{.items[0].metadata.name}')"
inpod() { kubectl -n "$NS" exec "$WEB" -c status-writer -- "$@"; }
CIP="$(svc api '{.spec.clusterIP}')"
ip -4 -o addr show | grep -qw "$CIP" && echo "FAIL: $CIP nam tren interface cua master" \
  || echo "PASS: ClusterIP $CIP cua api khong nam tren interface nao cua node"
T="$(inpod wget -qO- "http://$CIP:3000/books" | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && echo "PASS: TCP toi ClusterIP van toi api, /books tra $T cuon"
for i in $(seq 1 15); do inpod wget -qO- http://web/status.json; done > "$EV/b1-web-qua-service.txt" 2>/dev/null
N="$(python3 -c 'import json,sys; print(len({json.loads(l)["pod"] for l in sys.stdin if l.strip()}))' < "$EV/b1-web-qua-service.txt")"
[ "$N" -ge 2 ] && echo "PASS: mot ClusterIP cua web trai luu luong ra $N Pod"
IN="$(inpod ip -o -4 addr show dev eth0 | awk '{print $4}' | cut -d/ -f1)"
[ "$IN" = "$(kubectl -n "$NS" get pod "$WEB" -o jsonpath='{.status.podIP}')" ] \
  && echo 'PASS: dia chi Pod tu thay trung .status.podIP — khong co anh xa dia chi'

# 4. Nợ lab #3
DB_IP="$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.podIP}')"
for i in $(seq 1 30); do inpod nslookup db-0.db.bookstore.svc.cluster.local 2>/dev/null | grep -q "$DB_IP" && break; sleep 2; done
inpod nslookup db-0.db.bookstore.svc.cluster.local > "$EV/b1-dns-db-0.txt" 2>&1
grep -q "$DB_IP" "$EV/b1-dns-db-0.txt" && echo "PASS: NO LAB #3 DA TRA — db-0.db.bookstore.svc.cluster.local -> $DB_IP"
inpod nslookup db.bookstore.svc.cluster.local 2>/dev/null | grep -q "$DB_IP" \
  && echo 'PASS: ten Service headless db tra ve IP cua Pod, khong co ClusterIP nao'
git diff --quiet lab-4b -- k8s/app/db/statefulset.yaml && echo 'PASS: StatefulSet db khong phai sua dong nao'

# 5. Lệnh imperative so với khai báo
kubectl -n "$NS" expose deployment api --name=api-thu --port=3000 --target-port=http --dry-run=client -o yaml > "$EV/b1-expose.yaml"
kubectl -n "$NS" create service clusterip api-thu --tcp=3000:http --dry-run=client -o yaml > "$EV/b1-create-service.yaml"
grep -q 'app.kubernetes.io/component: api' "$EV/b1-expose.yaml" && echo 'PASS: expose chep selector cua Deployment api'
{ grep -q 'app: api-thu' "$EV/b1-create-service.yaml" && ! grep -q 'component' "$EV/b1-create-service.yaml"; } \
  && echo 'PASS: create service tu dat selector app=api-thu'
kubectl -n "$NS" create service clusterip api-thu --tcp=3000:http
sleep 5
[ -z "$(eps api-thu '{.items[*].endpoints[*].addresses[0]}')" ] \
  && echo 'PASS: selector khong khop Pod nao thi Service khong co endpoint nao'
kubectl -n "$NS" delete service api-thu && echo 'PASS: da xoa Service thu nghiem'
```

Nếu dòng `NO LAB #3` chậm: một truy vấn hỏi tên `db-0.db` **trước** khi Service tồn tại đã bị CoreDNS
nhớ kết quả thất bại — đúng cảnh báo negative caching của bài 65. Vòng chờ ở trên cho tối đa 60 giây.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/db/service.yaml — Service quản trị của StatefulSet db (trả nợ lab #3)
apiVersion: v1
kind: Service
metadata:
  name: db
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: db
    app.kubernetes.io/name: postgresql
spec:
  clusterIP: None                 # headless: DNS trả IP Pod và bản ghi db-0.db; kube-proxy không xử lý
  selector:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: db
  ports:
  - name: postgres
    port: 5432
    targetPort: postgres
---
# k8s/app/api/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: api
    app.kubernetes.io/name: postgrest
spec:
  selector:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: api
  ports:
  - name: http
    port: 3000
    targetPort: http
---
# k8s/app/web/service.yaml — B3 đổi sang NodePort 30080
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
    app.kubernetes.io/name: nginx
spec:
  selector:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
  ports:
  - name: http
    port: 80
    targetPort: http
```

Mỗi khối là một file riêng. Thêm ba dòng `db/service.yaml`, `api/service.yaml`, `web/service.yaml`
vào `resources` của `k8s/app/kustomization.yaml`, rồi `./scripts/deploy.sh`.

Ý chính cho ADR:

1. Bài 10: bản ghi cấp Pod chỉ có khi tồn tại một **headless** Service cùng namespace trùng tên
   `subdomain` của Pod. Với ClusterIP thường, `db` ra một IP ảo và không có `db-0.db` — mất định
   danh cấp Pod mà client cần để gọi đích danh và Lab 6a cần để gắn volume theo ordinal. Một bản sao
   thì IP ảo cũng không cân bằng được gì. `api` thì ngược lại: hai bản sao như nhau, client muốn một
   địa chỉ chung và để kube-proxy chia.
2. Có từ 4b: hostname `db-0`, label `statefulset.kubernetes.io/pod-name`, `subdomain: db` — do
   controller đặt. Service quản trị thêm **miền DNS** `db.bookstore.svc.cluster.local` và bản ghi
   `db-0.db.bookstore.svc.cluster.local` (bài 65). Vì `serviceName` đã khai trước, không phải sửa gì.
3. Chỉ sửa `containerPort` trong Deployment `api` (và cấu hình PostgREST). Service, nginx và mọi
   client gọi `api:3000` không đổi: EndpointSlice giải tên `http` ra số mới (bài 82).
4. `expose` lấy selector của Deployment `api` (bài 364) — gần hợp đồng nhất nhưng tên cổng và label
   theo ý lệnh; `create service` đặt selector `app=<tên>`, không khớp Pod nào nên Service rỗng. Dự án
   cần repo là nguồn sự thật duy nhất ([mục 5.3](DU-AN-BOOKSTORE.md#53-quy-tắc-làm-việc-với-repo)):
   manifest đọc được, review được và `deploy.sh` dựng lại được.

</details>

## B2. [BS-5A-02] Các tầng gọi nhau bằng tên: bỏ file `.tpl`, Kustomize đầy đủ

**Loại:** Tính năng — trả `TD-03`.

**Bối cảnh:** Service đã có, nhưng `api`, `db-migrate`, `db-backup` vẫn mang IP của `db-0` do
`deploy.sh` chèn lúc triển khai. Lab 4b B8 đã cho thấy hậu quả: `db-0` bị thay là `api` NotReady cho
tới khi có người chạy lại `deploy.sh`. Khi không còn gì phải chèn lúc triển khai, cả namespace đi
qua **một** kustomization — đúng hàng "5a" của [mục 5.5](DU-AN-BOOKSTORE.md#55-bố-cục-repo-theo-sprint).

**Yêu cầu:**

1. `api`, Job `db-migrate`, CronJob `db-backup` tìm DB bằng **tên DNS của Pod `db-0`** qua Service
   quản trị — dạng rút gọn `db-0.db` của [mục 3.1](DU-AN-BOOKSTORE.md#31-sơ-đồ-tổng-thể). Ba file
   thành `k8s/app/api/deployment.yaml`, `k8s/app/jobs/db-migrate.yaml`, `k8s/app/jobs/db-backup.yaml`;
   trong `k8s/` không còn file đuôi `.tpl`, trong `k8s/` và `scripts/` không còn chuỗi `__DB_HOST__`.
2. Job `db-migrate` **tự chờ** DB nhận kết nối trước khi chạy migration, để khi được apply cùng lúc
   với `db` nó vẫn xong ngay ở lần thử đầu — không tốn lượt thử lại nào.
3. `k8s/app/kustomization.yaml` gom **mọi** object của namespace, kể cả Job và CronJob. Ba ConfigMap
   `web-config`, `api-config`, `db-init` do bộ sinh của Kustomize tạo từ file nằm cạnh manifest — tên
   mang hậu tố hash, workload vẫn khai tên gốc. Tên và key của chúng giữ như hợp đồng; Secret
   `db-credentials` giữ **tên cố định**.
4. Label `app.kubernetes.io/part-of` đặt **một lần** ở kustomization cho mọi object sinh ra — kể cả
   ConfigMap — mà không chèn vào selector nào.
5. `scripts/deploy.sh` và `scripts/teardown.sh` thay bằng phiên bản Lab 5a — chép nguyên văn khối
   dưới.
6. `teardown.sh` rồi `deploy.sh` hai lần liên tiếp đều thành công; `/books` qua Service `api` trả ít
   nhất 8 cuốn sách.

**Ràng buộc:** không đổi key `index.html.tpl` mà init container `render-html` đọc — nhưng file nguồn
trong repo **không** được mang đuôi `.tpl`. Không sửa selector. Không `kubectl apply -k`. Lấy nội dung
ba ConfigMap cũ ra file **trước** khi teardown.

**Đọc lại:**

- Bài [322](../322-kustomization-vi.md): [*configMapGenerator*](../322-kustomization-vi.md#configmapgenerator),
  [*generatorOptions*](../322-kustomization-vi.md#generatoroptions),
  [*Thiết lập các trường xuyên suốt*](../322-kustomization-vi.md#thiết-lập-các-trường-xuyên-suốt-setting-cross-cutting-fields),
  [*Kết hợp*](../322-kustomization-vi.md#kết-hợp-composing),
  [*Danh sách tính năng*](../322-kustomization-vi.md#danh-sách-tính-năng-của-kustomize-kustomize-feature-list)
  — dòng `commonLabels` và dòng `labels`.
- Bài [10](../10-dns-pod-service-vi.md): [*Namespace của Service*](../10-dns-pod-service-vi.md#namespace-của-service-namespaces-of-services),
  [*Các trường hostname và subdomain của Pod*](../10-dns-pod-service-vi.md#pod-hostname-and-subdomain-field).
- Bài [82](../82-service-vi.md#dns), mục *DNS*.
- [Lab 3b](LAB-3B-CAU-HINH-UNG-DUNG.md) ticket B1 — ADR 0007 về Secret tên cố định và
  `--load-restrictor`; bài [275](../275-configure-pod-configmap-vi.md) — đặt key khác tên file.

**Bàn giao:** ba file manifest đổi tên; `k8s/app/{web,api,db}/` chứa file nguồn của ConfigMap;
`k8s/app/kustomization.yaml`; hai script mới; `docs/adr/NNNN-goi-nhau-bang-ten.md`;
`~/lab-evidence/5a/b2-*`.

**Nghiệm thu:**

- `PASS:` không còn file `.tpl` hay chuỗi `__DB_HOST__`; render mặc định vẫn bị từ chối vì `secrets/`
  nằm ngoài `k8s/app`.
- `PASS:` `teardown.sh` gỡ sạch; `deploy.sh` thành công hai lần liên tiếp.
- `PASS:` `db-migrate` xong mà không có lần thử thất bại nào.
- `PASS:` spec của `api`, `db-migrate`, `db-backup` chứa `db-0.db` và không chứa IP Pod nào.
- `PASS:` ba ConfigMap mang hậu tố hash, đúng tên workload tham chiếu, có label dự án; Secret
  `db-credentials` không hậu tố.
- `PASS:` gọi `api` bằng tên Service từ Pod `web`, `/books` trả ít nhất 8 cuốn.

Hai script, chép nguyên văn:

```bash
cat > ~/bookstore/scripts/deploy.sh <<'EOF'
#!/usr/bin/env bash
# Triển khai dự án Bookstore từ repo — phiên bản Lab 5a.
# - Các tầng tìm nhau bằng tên DNS qua Service (db-0.db, api, node-agent): không còn file .tpl,
#   không còn chèn IP lúc triển khai (TD-03 đã trả).
# - Mọi object của namespace bookstore, kể cả Job và CronJob, nằm trong kustomization k8s/app.
#   secrets/db.env nằm ngoài k8s/app nên phải render bằng --load-restrictor (Lab 3b, ADR 0007).
set -euo pipefail
cd "$(dirname "$0")/.."
NS=bookstore

# 1. Bí mật sinh tại chỗ khi thiếu; secrets/ không vào git.
mkdir -p secrets
if [ ! -s secrets/db.env ]; then
  ( umask 077
    printf 'postgres-password=%s\nauthenticator-password=%s\n' \
      "$(openssl rand -hex 24)" "$(openssl rand -hex 24)" > secrets/db.env )
  echo 'deploy: da sinh secrets/db.env moi'
fi

# 2. Cơ chế .tpl đã bỏ ở Lab 5a: còn file .tpl nghĩa là repo chưa chuyển xong.
TPL="$(find k8s -name '*.tpl')"
if [ -n "$TPL" ]; then echo 'deploy: con file .tpl, Lab 5a da bo co che nay:'; echo "$TPL"; exit 1; fi

# 3. Render một lần ra file tạm; lỗi Kustomize dừng ngay, trước khi đụng tới cluster.
RENDER="$(mktemp)"
trap 'rm -f "$RENDER"' EXIT
kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app > "$RENDER"

# 4. Object cấp cluster.
if compgen -G 'k8s/cluster/*.yaml' >/dev/null; then kubectl apply -f k8s/cluster/; fi

# 5. Job db-migrate chạy ở MỖI lần triển khai: Pod template của Job không sửa được sau khi tạo,
#    nên xóa Job cũ trước khi apply (Lab 4b). Lần đầu chưa có namespace thì bỏ qua.
if kubectl get namespace "$NS" >/dev/null 2>&1; then
  kubectl -n "$NS" delete job db-migrate --ignore-not-found --wait=true
fi
kubectl apply -f "$RENDER"

wait_job() {   # thành công khi Job Complete, thất bại ngay khi Job Failed
  local c=''
  for _ in $(seq 1 100); do
    c="$(kubectl -n "$NS" get job "$1" \
          -o jsonpath='{.status.conditions[?(@.status=="True")].type}' 2>/dev/null || true)"
    case " $c " in
      *' Complete '*) return 0 ;;
      *' Failed '*) kubectl -n "$NS" logs "job/$1" --all-containers --tail=20 || true; return 1 ;;
    esac
    sleep 3
  done
  return 1
}

# 6. Chờ theo chuỗi phụ thuộc. Init container của api và của db-migrate tự chờ db-0.db nhận kết nối,
#    nên apply mọi thứ cùng lúc vẫn an toàn.
kubectl -n "$NS" rollout status statefulset/db --timeout=300s
wait_job db-migrate || { echo 'deploy: db-migrate that bai'; exit 1; }
for D in api web; do kubectl -n "$NS" rollout status "deployment/$D" --timeout=300s; done
kubectl -n "$NS" rollout status daemonset/node-agent --timeout=300s
kubectl -n "$NS" get statefulset,deployment,daemonset,cronjob,job,service -o wide
EOF

cat > ~/bookstore/scripts/teardown.sh <<'EOF'
#!/usr/bin/env bash
# Gỡ toàn bộ dự án Bookstore khỏi cluster — phiên bản Lab 5a.
# Giữ nguyên secrets/db.env trên máy. Từ 5a dự án có Service (NodePort 30080), EndpointSlice và
# NetworkPolicy: đều đi theo namespace, nhưng script vẫn dọn và đếm theo label để chắc không sót.
set -euo pipefail
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
KINDS='statefulsets,deployments,daemonsets,cronjobs,jobs,replicasets,pods,poddisruptionbudgets,services,networkpolicies,configmaps,secrets'

# StatefulSet không bảo đảm Pod tắt có thứ tự khi bị xóa (bài 65, mục Hạn chế): thu db về 0 trước.
if kubectl -n "$NS" get statefulset db >/dev/null 2>&1; then
  kubectl -n "$NS" scale statefulset db --replicas=0
  kubectl -n "$NS" wait --for=delete pod/db-0 --timeout=180s || true
fi

# Xóa namespace kéo theo mọi object trong đó; dòng sau dọn object mang label dự án ở namespace khác.
kubectl delete namespace -l "$SEL" --ignore-not-found --wait=true --timeout=300s
kubectl delete "$KINDS" --all-namespaces -l "$SEL" --ignore-not-found --wait=true --timeout=120s
kubectl delete priorityclass -l "$SEL" --ignore-not-found

LEFT="$(kubectl get namespace,priorityclass -l "$SEL" --no-headers 2>/dev/null | wc -l)"
LEFT_NS="$(kubectl get "$KINDS,endpointslices" --all-namespaces -l "$SEL" --no-headers 2>/dev/null | wc -l)"
NP="$(kubectl get services -A -o jsonpath='{range .items[*]}{range .spec.ports[*]}{.nodePort}{"\n"}{end}{end}' \
      | grep -cx 30080 || true)"
if [ "$LEFT" -eq 0 ] && [ "$LEFT_NS" -eq 0 ] && [ "$NP" -eq 0 ]; then
  echo 'teardown: sach'
else
  echo "teardown: con $LEFT object cap cluster, $LEFT_NS object co label du an, $NP Service giu NodePort 30080"; exit 1
fi
EOF

chmod +x ~/bookstore/scripts/*.sh
```

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-goi-nhau-bang-ten.md`:

1. `TD-03` được trả bằng cơ chế nào? Lần tới `db-0` bị thay và đổi IP, thành phần nào cập nhật địa chỉ
   mà `api` dùng — và vì sao không ai phải chạy lại `deploy.sh`? (B5 đo điều này.)
2. ConfigMap mang hậu tố hash, Secret thì không. Lý do cũ ở ADR 0007 của Lab 3b (`api` nằm ngoài
   kustomization) đã hết. Lý do nào giữ Secret tên cố định bây giờ? Nghĩ tới mật khẩu đã nằm trong
   dữ liệu DB, tới Lab 6a khi dữ liệu bền, và tới việc PostgREST coi lỗi xác thực là lỗi không thử lại.
3. Bạn sửa `01-schema.sql` trong repo rồi `deploy.sh`. Chuyện gì xảy ra với `db-0` hôm nay, khi dữ
   liệu nằm trên `emptyDir`? Sau Lab 6a, khi dữ liệu bền, thay đổi đó còn tác dụng không?
4. Vì sao đặt label dự án bằng `labels` mà không bằng `commonLabels`? Dẫn đúng dòng trong bảng cuối
   bài 322; selector nào của dự án sẽ bị đụng.
5. Job chờ DB bằng init container hay để `backoffLimit` lo? So hai cách theo lượt thử, thời gian và
   log để lại.

<details>
<summary>Gợi ý</summary>

1. Lấy nội dung ConfigMap đang chạy ra file — làm **trước** khi teardown:
   `kubectl -n bookstore get configmap web-config -o go-template='{{index .data "default.conf"}}' > k8s/app/web/default.conf`.
   Với `api-config`, in mỗi key một dòng `KEY=value` bằng
   `-o go-template='{{range $k, $v := .data}}{{$k}}={{$v}}{{"\n"}}{{end}}'`.
2. Bộ sinh nhận ba dạng nguồn; `envs` biến mỗi dòng `KEY=value` thành một key — đúng hình dạng
   `api-config`. Muốn key khác tên file: `files: - <key>=<đường dẫn>`.
3. `labels` có cờ `includeSelectors`; để `false` thì chỉ đụng `metadata.labels`.
4. `namespace:` ở đầu kustomization đặt namespace cho mọi object, kể cả ConfigMap và Secret sinh ra.
5. `DB_HOST` của `api` xuất hiện ở hai chỗ: init container `wait-for-db` và container `postgrest`.
6. Init container chờ DB của Job: chép `wait-for-db` của `api`.
7. `git mv` để giữ lịch sử file; `git rm` ba `configmap.yaml` cũ. Đổi cơ chế triển khai giữa chừng:
   `./scripts/teardown.sh && ./scripts/deploy.sh`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong; script teardown rồi deploy lại</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a
cd ~/bookstore
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

{ [ -z "$(find k8s -name '*.tpl')" ] && ! grep -rq '__DB_HOST__' k8s scripts; } \
  && echo 'PASS: khong con file .tpl, khong con chuoi __DB_HOST__'
kubectl kustomize k8s/app > /dev/null 2>&1 && echo 'FAIL: render mac dinh doc duoc secrets/' \
  || echo 'PASS: van phai render voi --load-restrictor (secrets/ nam ngoai k8s/app)'

./scripts/teardown.sh > "$EV/b2-teardown.txt" 2>&1; grep -q 'teardown: sach' "$EV/b2-teardown.txt" \
  && echo 'PASS: teardown.sh moi go sach'
./scripts/deploy.sh > "$EV/b2-deploy-1.txt" 2>&1 && echo 'PASS: deploy.sh lan 1 thanh cong'
F1="$(jp job/db-migrate '{.status.failed}')"
./scripts/deploy.sh > "$EV/b2-deploy-2.txt" 2>&1 && echo 'PASS: deploy.sh lan 2 thanh cong'
[ "${F1:-0}" -eq 0 ] && echo 'PASS: db-migrate xong ngay lan thu dau du apply cung luc voi db'

BAD=0
for O in deployment/api job/db-migrate cronjob/db-backup; do
  Y="$(kubectl -n "$NS" get "$O" -o yaml)"; echo "$Y" > "$EV/b2-${O%%/*}.yaml"
  { echo "$Y" | grep -q 'db-0\.db' && ! echo "$Y" | grep -Eq '10\.244\.[0-9]+\.[0-9]+'; } \
    || { echo "FAIL: $O khong goi DB bang ten db-0.db"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: api, db-migrate, db-backup goi DB bang ten db-0.db, khong con IP Pod'

CM_WEB="$(jp deployment/web '{.spec.template.spec.volumes[*].configMap.name}' | tr ' ' '\n' | sort -u)"
CM_API="$(jp deployment/api '{.spec.template.spec.containers[*].envFrom[*].configMapRef.name}')"
CM_DB="$(jp statefulset/db '{.spec.template.spec.volumes[*].configMap.name}')"
echo "web=$CM_WEB api=$CM_API db=$CM_DB" | tee "$EV/b2-configmap.txt"
OK=1
for X in "web-config:$CM_WEB" "api-config:$CM_API" "db-init:$CM_DB"; do
  B="${X%%:*}"; N="${X#*:}"
  { echo "$N" | grep -Eqx "${B}-[a-z0-9]+" && \
    [ "$(kubectl -n "$NS" get configmap "$N" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}')" = 'bookstore' ]; } \
    || { echo "FAIL: ConfigMap $B duoc tham chieu la '$N'"; OK=0; }
done
[ "$OK" -eq 1 ] && echo 'PASS: ba ConfigMap mang hau to hash, workload tham chieu dung ten, co label du an'
[ "$(kubectl -n "$NS" get secret -o name | grep -c '^secret/db-credentials')" -eq 1 ] \
  && kubectl -n "$NS" get secret db-credentials >/dev/null 2>&1 \
  && echo 'PASS: Secret db-credentials giu ten co dinh, khong hau to'

WEB="$(kubectl -n "$NS" get pod -l 'app.kubernetes.io/component=web,!lab' -o jsonpath='{.items[0].metadata.name}')"
T="$(kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- http://api:3000/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && echo "PASS: goi api bang ten Service, /books tra $T cuon"
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

Chuyển file — chạy khi hệ thống của B1 **còn đang chạy**:

```bash
cd ~/bookstore
G='kubectl -n bookstore get configmap'
$G web-config -o go-template='{{index .data "default.conf"}}'        > k8s/app/web/default.conf
$G web-config -o go-template='{{index .data "index.html.tpl"}}'      > k8s/app/web/index.template.html
$G db-init    -o go-template='{{index .data "01-schema.sql"}}'       > k8s/app/db/01-schema.sql
$G db-init    -o go-template='{{index .data "02-authenticator.sh"}}' > k8s/app/db/02-authenticator.sh
$G api-config -o go-template='{{range $k, $v := .data}}{{$k}}={{$v}}{{"\n"}}{{end}}' > k8s/app/api/config.env
git rm -q k8s/app/web/configmap.yaml k8s/app/db/configmap.yaml k8s/app/api/configmap.yaml
git mv k8s/app/api/deployment.yaml.tpl k8s/app/api/deployment.yaml
git mv k8s/app/jobs/db-migrate.yaml.tpl k8s/app/jobs/db-migrate.yaml
git mv k8s/app/jobs/db-backup.yaml.tpl k8s/app/jobs/db-backup.yaml
sed -i 's/__DB_HOST__/db-0.db/g' k8s/app/api/deployment.yaml k8s/app/jobs/db-migrate.yaml k8s/app/jobs/db-backup.yaml
grep -n 'db-0.db' k8s/app/api/deployment.yaml k8s/app/jobs/*.yaml
```

Sau `sed`, hai biến `DB_HOST` của `api` (init `wait-for-db` và `postgrest`), `PGHOST` của `db-migrate`
và của `db-backup` đều là `db-0.db`; phần còn lại của ba file giữ như tag `lab-4b`. `db-migrate` thêm
một init container vào `spec.template.spec` — chép đúng `wait-for-db` của `api`:

```yaml
# k8s/app/jobs/db-migrate.yaml — phần thêm; backoffLimit 3, activeDeadlineSeconds 300 (tính cả lúc
# init chờ DB), TTL, container migrate với SQL bất biến kết quả: giữ nguyên như Lab 4b
    spec:
      restartPolicy: Never
      initContainers:
      - name: wait-for-db
        image: postgres:17.11
        command:
        - sh
        - -c
        - |
          until pg_isready -h db-0.db -p 5432 -t 2; do
            echo "wait-for-db: db-0.db:5432 chua nhan ket noi"
            sleep 2
          done
        resources:
          requests: {cpu: 10m, memory: 16Mi}
          limits: {cpu: 50m, memory: 32Mi}
      containers:
      - name: migrate
        env:
        - {name: PGHOST, value: db-0.db}     # sau sed; PGPORT, PGUSER, PGPASSWORD... giữ nguyên
```

```yaml
# k8s/app/kustomization.yaml — Lab 5a: mọi object của namespace bookstore đi qua Kustomize.
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: bookstore
labels:
- pairs:
    app.kubernetes.io/part-of: bookstore
  includeSelectors: false          # chỉ metadata.labels; selector đã đúng hợp đồng, không đụng tới

resources:
- 00-namespace.yaml
- db/statefulset.yaml
- db/service.yaml
- api/deployment.yaml
- api/service.yaml
- api/pdb.yaml                     # PDB maxUnavailable 1 từ Lab 3c, 4a — giữ đường dẫn của bạn
- web/deployment.yaml
- web/service.yaml
- web/pdb.yaml
- agent/daemonset.yaml
# - agent/service.yaml             # B4 thêm dòng này khi đã có file
- jobs/db-migrate.yaml
- jobs/db-backup.yaml

configMapGenerator:
- name: db-init
  files:
  - db/01-schema.sql
  - db/02-authenticator.sh
  options:
    labels:
      app.kubernetes.io/component: db
      app.kubernetes.io/name: postgresql
- name: api-config
  envs:
  - api/config.env
  options:
    labels:
      app.kubernetes.io/component: api
      app.kubernetes.io/name: postgrest
- name: web-config
  files:
  - web/default.conf
  - index.html.tpl=web/index.template.html   # key giữ nguyên, file nguồn không mang đuôi .tpl
  options:
    labels:
      app.kubernetes.io/component: web
      app.kubernetes.io/name: nginx

secretGenerator:
- name: db-credentials
  envs:
  - ../../secrets/db.env
  options:
    disableNameSuffixHash: true      # tên cố định — vì sao: câu hỏi thiết kế 2
    labels:
      app.kubernetes.io/component: db
```

```bash
cd ~/bookstore
kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app \
  | grep -E '^  name: (web-config|api-config|db-init|db-credentials)'
./scripts/teardown.sh && ./scripts/deploy.sh
```

Ý chính cho ADR:

1. Service quản trị cộng CoreDNS: địa chỉ được phân giải **lúc kết nối**. Khi `db-0` bị thay,
   EndpointSlice của `db` nhận IP mới, CoreDNS trả IP mới cho `db-0.db`, và PostgREST — vốn tự thử
   kết nối lại — kết nối tới đúng chỗ. Không ai phải render lại manifest.
2. Mật khẩu `authenticator` đã được ghi vào **dữ liệu** DB lúc khởi tạo. Secret mang hậu tố hash thì
   mỗi lần bí mật đổi là một tên mới, template của `api` và `db` đổi theo và **tự rollout** với mật
   khẩu mới — trong khi DB (từ 6a là dữ liệu bền) vẫn giữ mật khẩu cũ. PostgREST coi lỗi xác thực là
   lỗi không thử lại: `api` vào `CrashLoopBackOff`. Đổi bí mật phải là một quy trình có thứ tự (đổi
   trong DB trước), không phải tác dụng phụ của `deploy.sh`. Thêm vào đó
   [mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label) gọi Secret bằng tên cố định.
3. Hôm nay: nội dung đổi → tên `db-init-*` đổi → template StatefulSet đổi → `db-0` bị thay, `emptyDir`
   mới, DB khởi tạo lại với schema mới; dữ liệu migration mất cho tới khi `deploy.sh` chạy lại
   migration — đúng việc nó làm. Sau 6a: thư mục dữ liệu không rỗng, script khởi tạo **không chạy**;
   thay đổi chỉ gây một lần khởi động lại `db-0`. Thay đổi schema phải đi qua migration.
4. Bảng cuối bài 322: `commonLabels` thêm label vào **tài nguyên và selector**; `labels` (với
   `includeSelectors: false`) chỉ thêm label. Selector của Deployment, StatefulSet bất biến; còn
   NetworkPolicy ở B8 có selector chọn Pod CoreDNS ở `kube-system` — chèn `part-of` vào đó là chặn DNS
   khi policy có hiệu lực ở 5b. B8 ⭐ chứng minh.
5. Init container: Pod nằm `Init:0/1`, không tốn lượt thử, log ghi rõ đang chờ gì, và
   `activeDeadlineSeconds` vẫn chặn tổng thời gian. Để `backoffLimit` lo: mỗi lần `psql` hỏng là một
   Pod `Failed`, backoff 10s, 20s, 40s…; DB chậm hơn vài chục giây là Job hết lượt và `deploy.sh` fail.

</details>

## B3. [BS-5A-03] `web` thành cổng vào: proxy `/api/` và NodePort `30080`

**Loại:** Tính năng.

**Bối cảnh:** trang chủ đã có đoạn script gọi `/api/books` từ Lab 3a, nhưng nginx chưa biết chuyển
`/api/` đi đâu, và người dùng ngoài cluster chưa vào được. Hệ thống cũ mở `80:8080` trên một VM
([mục 2.1](DU-AN-BOOKSTORE.md#21-hiện-trạng)). Sprint này mở `web` ra ngoài bằng NodePort; Lab 5b thay
bằng Ingress có TLS. Ở Lab 4a, sửa `web-config` không tự gây rollout nên phải tăng label `version`
trong Pod template bằng tay; ticket này cho thấy bộ sinh ConfigMap của B2 thay cơ chế đó.

**Yêu cầu:**

1. nginx của `web` chuyển mọi request dưới `/api/` sang Service `api`, **bỏ tiền tố `/api`**:
   `GET /api/books` của `web` tương đương `GET /books` của `api`. Các `location` cũ giữ nguyên; thêm
   đúng chỗ dòng chú thích Lab 3b để lại.
2. Service `web` thành NodePort với cổng node `30080`; cổng Service 80 → `http` giữ nguyên
   ([mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe)). `api` vẫn chỉ trong
   cluster.
3. Từ master, `http://<IP node>:30080/` của **cả ba** node trả trang Bookstore; `/api/books` trả ít
   nhất 8 cuốn sách.
4. Trình duyệt trên máy host mở `http://<IP worker>:30080/` thấy danh sách sách thật, không còn dòng
   "Chưa kết nối được API".
5. Chứng minh đổi cấu hình là một rollout: chỉ sửa file trong repo rồi `./scripts/deploy.sh`, `web` có
   ReplicaSet mới trỏ tới ConfigMap tên mới, còn ConfigMap cũ vẫn tồn tại. Lần phát hành này vẫn ghi
   lý do vào lịch sử rollout như quy ước Lab 4a.

**Ràng buộc:** không địa chỉ IP nào trong cấu hình nginx. Không `kubectl edit`/`patch`. Ticket này
**không** teardown — yêu cầu 5 cần lịch sử rollout. Lỡ teardown thì sửa thêm một dòng chú thích trong
`default.conf` rồi `deploy.sh` lại.

**Đọc lại:**

- Bài [82](../82-service-vi.md): [`type: NodePort`](../82-service-vi.md#type-nodeport),
  [*Tự chọn port*](../82-service-vi.md#nodeport-custom-port),
  [*Dành riêng dải NodePort*](../82-service-vi.md#avoid-nodeport-collisions).
- Bài [363](../363-connecting-frontend-backend-vi.md): [*Trước khi bạn bắt đầu*](../363-connecting-frontend-backend-vi.md#trước-khi-bạn-bắt-đầu-before-you-begin),
  [*Tạo frontend*](../363-connecting-frontend-backend-vi.md#tạo-frontend-creating-the-frontend) —
  `server hello;` chỉ là tên Service.
- Bài [322](../322-kustomization-vi.md#generatoroptions) — vì sao có hậu tố hash.
- [Lab 3b](LAB-3B-CAU-HINH-UNG-DUNG.md) ticket B4 — ADR 0010: nginx chỉ đọc cấu hình lúc khởi động.
- nginx: [`proxy_pass`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) —
  đoạn về URI trong `proxy_pass` và về tên miền phân giải ra nhiều địa chỉ.

**Bàn giao:** `k8s/app/web/default.conf`; `k8s/app/web/service.yaml`;
`docs/adr/NNNN-cong-vao-web.md`; `~/lab-evidence/5a/b3-*`; tùy chọn: ảnh chụp trình duyệt.

**Nghiệm thu:**

- `PASS:` `web` NodePort 30080, cổng 80 → `http`; `api` vẫn ClusterIP.
- `PASS:` cả ba node trả `200` và trang Bookstore trên cổng 30080, kể cả master không chạy Pod `web`.
- `PASS:` `/api/books` qua NodePort trả ít nhất 8 cuốn; `/books` không tiền tố không phải đường vào
  API.
- `PASS:` `web` có ReplicaSet mới nhất trỏ ConfigMap khác ReplicaSet trước; ConfigMap cũ còn đó;
  revision mới nhất có lý do phát hành.
- `PASS:` (PowerShell trên host) nội dung `/api/books` có chuỗi `title`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-cong-vao-web.md`:

1. nginx phân giải tên `api` lúc nào? Vì sao điều đó an toàn khi đích là một Service ClusterIP, nhưng
   sẽ hỏng nếu đích là Service headless hay IP Pod? Nếu Service `api` bị xóa rồi tạo lại thì sao?
2. Vì sao 30080? Nó nằm ở băng nào của dải NodePort (bài 82), và vì sao không dùng cổng 80 như file
   compose cũ?
3. Ở Lab 3b, sửa `web-config` thì phải tạo lại Pod nginx; ở Lab 4a phải tăng label `version` bằng tay
   để có rollout. Hôm nay vì sao không phải? Hậu tố hash đã biến "sửa cấu hình" thành loại thao tác
   nào, label `version` và annotation `kubernetes.io/change-cause` còn dùng để làm gì, và ConfigMap cũ
   còn lại để làm gì?
4. Gọi `<IP master>:30080` thì master không chạy Pod `web` nào — vì sao vẫn `200`, và đổi lại mất gì
   (gợi ý: bài 364, IP nguồn)?

<details>
<summary>Gợi ý</summary>

1. `proxy_pass` có phần URI — dấu `/` ở cuối — thì nginx thay đoạn khớp `location` bằng URI đó.
2. Tên ngắn `api` được danh sách `search` trong `/etc/resolv.conf` của Pod `web` ghép thành FQDN.
3. Trường `nodePort` nằm trong phần tử `ports` của Service.
4. Lịch sử: `kubectl -n bookstore rollout history deployment/web`; ReplicaSet và ConfigMap của từng
   revision: `kubectl -n bookstore get rs -l app.kubernetes.io/component=web -o wide`.
5. Từ máy host Windows:
   `(Invoke-WebRequest -UseBasicParsing http://192.168.100.222:30080/api/books).Content`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a
nip() { kubectl get node "$1" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'; }
svc() { kubectl -n "$NS" get service "$1" -o jsonpath="$2" 2>/dev/null; }

[ "$(svc web '{.spec.type} {.spec.ports[0].port} {.spec.ports[0].targetPort} {.spec.ports[0].nodePort}')" = 'NodePort 80 http 30080' ] \
  && [ "$(svc api '{.spec.type}')" = 'ClusterIP' ] && echo 'PASS: web NodePort 30080, cong 80 -> http; api van ClusterIP'

: > "$EV/b3-nodeport.txt"
for N in lab-k8s-master lab-k8s-worker1 lab-k8s-worker2; do
  C="$(curl -s -o "$EV/b3-trang-$N.html" -m 5 -w '%{http_code}' "http://$(nip "$N"):30080/")"
  echo "$N -> $C" | tee -a "$EV/b3-nodeport.txt"
done
{ [ "$(grep -c ' -> 200$' "$EV/b3-nodeport.txt")" -eq 3 ] && grep -q 'Bookstore' "$EV/b3-trang-lab-k8s-master.html"; } \
  && echo 'PASS: ca ba node tra trang Bookstore tren cong 30080, ke ca master khong co Pod web'

W1="$(nip lab-k8s-worker1)"
curl -s -m 5 "http://$W1:30080/api/books" > "$EV/b3-api-books.json"
T="$(grep -o '"title"' "$EV/b3-api-books.json" | wc -l)"
[ "$T" -ge 8 ] && echo "PASS: /api/books qua NodePort tra $T cuon — nginx bo tien to /api"
curl -s -m 5 "http://$W1:30080/books" | grep -q '"title"' \
  && echo 'FAIL: /books khong tien to van ra du lieu api' || echo 'PASS: /books khong tien to khong phai duong vao api'

kubectl -n "$NS" get rs -l 'app.kubernetes.io/component=web,!lab' \
  -o jsonpath='{range .items[*]}{.metadata.annotations.deployment\.kubernetes\.io/revision} {.spec.template.spec.volumes[*].configMap.name}{"\n"}{end}' \
  | sort -n | awk '{print $1, $2}' > "$EV/b3-rs.txt"
cat "$EV/b3-rs.txt"
NEW="$(tail -1 "$EV/b3-rs.txt" | cut -d' ' -f2)"; OLD="$(tail -2 "$EV/b3-rs.txt" | head -1 | cut -d' ' -f2)"
CUR="$(kubectl -n "$NS" get deployment web -o jsonpath='{.spec.template.spec.volumes[*].configMap.name}' | cut -d' ' -f1)"
{ [ "$(grep -c . "$EV/b3-rs.txt")" -ge 2 ] && [ -n "$OLD" ] && [ "$NEW" != "$OLD" ] && [ "$CUR" = "$NEW" ] && \
  kubectl -n "$NS" get configmap "$OLD" >/dev/null 2>&1; } \
  && echo "PASS: doi default.conf sinh ConfigMap $NEW, web rollout sang ReplicaSet moi; $OLD van con"
kubectl -n "$NS" rollout history deployment/web | awk 'NF' | tail -1 | tee -a "$EV/b3-rs.txt" | grep -q '<none>' \
  && echo 'FAIL: revision moi nhat cua web khong co ly do' || echo 'PASS: revision moi nhat cua web co ly do phat hanh'
```

Trên máy host Windows, PowerShell — sửa IP cho đúng worker của bạn:

```powershell
$r = Invoke-WebRequest -UseBasicParsing http://192.168.100.222:30080/api/books
if ($r.StatusCode -eq 200 -and $r.Content -match '"title"') { 'PASS: may host goi duoc /api/books qua NodePort' }
```

Mở `http://192.168.100.222:30080/` bằng trình duyệt: danh sách sách hiện ra thay cho dòng "Đang tải".

</details>

<details>
<summary>Lời giải tham khảo</summary>

```nginx
# k8s/app/web/default.conf — cấu hình site của web (Bookstore).
# nginx chỉ đọc file này khi khởi động; từ Lab 5a, sửa file là sinh ConfigMap mới và web rollout.
server {
    listen      8080;
    server_name _;
    root        /usr/share/nginx/html;

    location = /healthz {
        access_log   off;
        default_type text/plain;
        return 200 "ok";
    }

    location = /status.json {
        add_header Cache-Control "no-store" always;
    }

    location / {
        index index.html;
    }

    # Lab 5a: reverse proxy sang Service api. "/" cuối proxy_pass thay "/api/" bằng "/":
    # /api/books -> http://api:3000/books. Tên "api" được phân giải một lần lúc nginx khởi động.
    location /api/ {
        proxy_pass http://api:3000/;
    }
}
```

```yaml
# k8s/app/web/service.yaml — sau B3
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
    app.kubernetes.io/name: nginx
spec:
  type: NodePort                  # Lab 5b trả về ClusterIP, người dùng đi qua Ingress
  selector:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: web
  ports:
  - name: http
    port: 80
    targetPort: http
    nodePort: 30080               # băng tĩnh 30000-30085 của dải mặc định
```

Trong `k8s/app/web/deployment.yaml`, chỉ đổi lý do phát hành — không phải tăng `version` để có rollout:

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "5a: nginx proxy /api/ sang Service api"
```

```bash
cd ~/bookstore && ./scripts/deploy.sh
kubectl -n bookstore rollout history deployment/web
kubectl -n bookstore get configmap -l app.kubernetes.io/component=web
```

Ý chính cho ADR:

1. nginx phân giải tên trong `proxy_pass` **một lần**, lúc nạp cấu hình. Với Service ClusterIP, địa
   chỉ đó không đổi suốt đời Service; Pod `api` đến và đi là việc của kube-proxy. Với headless hay IP
   Pod, nginx giữ IP cũ sau khi Pod bị thay — chính là `TD-03` tái diễn bên trong nginx. Service `api`
   bị xóa rồi tạo lại thì ClusterIP đổi, nginx giữ địa chỉ cũ tới khi Pod `web` khởi động lại; Service
   `api` không tồn tại lúc nginx khởi động thì nginx không lên được.
2. Bài 82: dải mặc định 30000–32767 chia băng; băng tĩnh 30000–30085 dành cho cổng chọn tay, cấp
   phát động dùng băng trên — ít rủi ro va chạm. Cổng 80 nằm ngoài dải NodePort; cổng chuẩn sẽ do
   Ingress ở 5b đảm nhận.
3. Nội dung đổi → tên ConfigMap đổi → template của Deployment đổi → RollingUpdate như một lần phát
   hành, có lịch sử, rollback được — không cần tăng `version` chỉ để kích rollout như ở Lab 4a.
   `version` vẫn là tên phiên bản người đọc thấy trong `status.json`, `change-cause` vẫn là lý do ghi
   vào lịch sử rollout; tăng chúng khi phát hành là quyết định của con người, không còn là mẹo kỹ thuật.
   Rollback cũng khác Lab 4a: `rollout undo` giờ đưa cả cấu hình cũ về, vì ReplicaSet cũ trỏ đúng
   ConfigMap cũ. ConfigMap cũ phải còn đó để ReplicaSet cũ mount được; xóa nó là phá rollback.
   `teardown.sh` dọn hết cuối sprint.
4. Mọi node lắng nghe cổng NodePort và chuyển tiếp tới một endpoint sẵn sàng bất kỳ
   (`externalTrafficPolicy: Cluster`, mặc định). Đổi lại: thêm một chặng sang node khác và backend
   không thấy IP nguồn thật của client (bài 364). B9 đo phương án `Local`.

</details>

## B4. [BS-5A-04] `status-writer` đọc `node-agent` cùng node

**Loại:** Tính năng, kèm điều tra trên `lab-k8s-worker2`.

**Bối cảnh:** Lab 4b dựng `node-agent` trên cả ba node để người vận hành biết request vừa rồi do
node nào phục vụ và node đó đang tải ra sao ([mục 3.2](DU-AN-BOOKSTORE.md#32-thành-phần-ứng-dụng)).
Chưa có Service nên chưa ai đọc nó. Thông tin chỉ có nghĩa khi `status-writer` đọc agent **cùng
node** với Pod `web` của nó — đọc nhầm agent của node khác là báo sai.

**Yêu cầu:**

1. Service `node-agent` cổng 9100 → `http` ([mục 4.2](DU-AN-BOOKSTORE.md#42-cổng-đường-dẫn-và-điểm-kiểm-tra-sức-khỏe)),
   chỉ định tuyến lưu lượng nội bộ tới endpoint nằm **trên cùng node** với client.
2. Mỗi lần ghi, `status-writer` đọc `/node.json` qua Service bằng tên, đưa nguyên đối tượng JSON đó
   vào `status.json` dưới trường mới `agent`. Bốn trường `pod`, `node`, `version`, `updated` giữ
   nguyên. Đọc không được thì `agent` là `null` và sidecar vẫn ghi tiếp.
3. Với **mọi** Pod `web`: gọi `node-agent` nhiều lần đều trả đúng node của Pod đó, và `agent.node`
   trong `status.json` trùng `node`.
4. Điều tra cảnh báo của bài 87 bằng object thử nghiệm ở `~/lab-work/5a/`: một backend chỉ chạy trên
   `lab-k8s-worker1`, một Service cùng chính sách lưu lượng như `node-agent`, và hai client ghim hai
   worker. Chứng minh client trên `lab-k8s-worker2` gọi không được dù tên vẫn phân giải.
5. Manifest ở `k8s/app/agent/service.yaml`; `status-writer` sửa trong `k8s/app/web/deployment.yaml`.

**Ràng buộc:** không gọi agent bằng IP Pod hay IP node; không `hostNetwork`, `hostPort`. Object thử
nghiệm xóa khi xong.

**Đọc lại:**

- Bài [87](../87-service-traffic-policy-vi.md): [*Sử dụng Service Internal Traffic Policy*](../87-service-traffic-policy-vi.md#sử-dụng-service-internal-traffic-policy-using-service-internal-traffic-policy)
  — đặc biệt ghi chú về node không có endpoint;
  [*Cách hoạt động*](../87-service-traffic-policy-vi.md#cách-hoạt-động-how-it-works).
- Bài [83](../83-endpoint-slices-vi.md#topology) — trường `nodeName` của endpoint.
- [Lab 4b](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md) ticket B7 — `node-agent` và `/node.json`;
  [Lab 3a](LAB-3A-POD-VA-VONG-DOI.md) ticket B5 — `status-writer` và cách ghi file nguyên tử.

**Bàn giao:** `k8s/app/agent/service.yaml`; `k8s/app/web/deployment.yaml`; `k8s/app/kustomization.yaml`;
`~/lab-work/5a/itp-thu.yaml`; `docs/adr/NNNN-agent-cung-node.md`; `~/lab-evidence/5a/b4-*`.

**Nghiệm thu:**

- `PASS:` Service `node-agent` ClusterIP 9100 → `http`, `internalTrafficPolicy: Local`.
- `PASS:` EndpointSlice của `node-agent` có endpoint sẵn sàng trên mọi node.
- `PASS:` từ mỗi Pod `web`, 10 lần gọi `node-agent` đều trả node của chính Pod đó; `status.json` có
  đúng năm trường và `agent.node` trùng `node`.
- `PASS:` client trên `lab-k8s-worker1` gọi được Service thử nghiệm; client trên `lab-k8s-worker2`
  không gọi được nhưng vẫn phân giải được tên; object thử nghiệm đã bị xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-agent-cung-node.md`:

1. Nếu Service `node-agent` để chính sách mặc định thì `status.json` sai thế nào, và vì sao lỗi đó
   khó phát hiện nếu `status.json` không ghi cả `node` lẫn `agent.node`?
2. Bài 87 nói cái giá của chính sách này. Kể những lúc một node chạy Pod `web` mà **không** có agent
   sẵn sàng (nghĩ tới agent khởi động lại, node mới, taint bảo trì ở Lab 7a, Lab 12). Thiết kế
   `agent: null` của bạn xử lý những lúc đó ra sao?
3. Chính sách này có đổi gì với lưu lượng vào NodePort `30080` không? Trường nào mới chi phối lưu
   lượng đó?
4. Vì sao DaemonSet cộng chính sách lưu lượng nội bộ `Local` là một cặp tự nhiên, còn Deployment hai
   bản sao cộng `Local` thì nguy hiểm?

<details>
<summary>Gợi ý</summary>

1. Trường cần đặt là `.spec.internalTrafficPolicy` của Service.
2. Trong vòng lặp của `status-writer`: `A="$(wget -qO- -T 2 http://node-agent:9100/node.json 2>/dev/null || true)"`,
   rỗng thì `A=null`, rồi đưa `$A` **không bọc nháy** vào `printf` — nó đã là JSON.
3. Kiểm JSON trên master: `... | python3 -c 'import json,sys; print(json.load(sys.stdin))'`.
4. Pod client thử nghiệm: `busybox` chạy `sleep 3600`, `nodeSelector` theo `kubernetes.io/hostname`,
   `terminationGracePeriodSeconds: 1` để xóa nhanh.

</details>

<details>
<summary>Script nghiệm thu — chạy sau `./scripts/deploy.sh` và sau khi tự viết `itp-thu.yaml`</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a; W=~/lab-work/5a
svc() { kubectl -n "$NS" get service "$1" -o jsonpath="$2" 2>/dev/null; }

[ "$(svc node-agent '{.spec.type} {.spec.ports[0].port} {.spec.ports[0].targetPort} {.spec.internalTrafficPolicy}')" = 'ClusterIP 9100 http Local' ] \
  && echo 'PASS: Service node-agent 9100 -> http, internalTrafficPolicy Local'
NODES="$(kubectl get nodes --no-headers | wc -l)"
EPN="$(kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=node-agent \
  -o jsonpath='{range .items[*].endpoints[*]}{.nodeName} {.conditions.ready}{"\n"}{end}' | awk '$2=="true"{print $1}' | sort -u | grep -c .)"
[ "$EPN" -eq "$NODES" ] && echo "PASS: node-agent co endpoint san sang tren ca $NODES node"

BAD=0; : > "$EV/b4-status.txt"
for P in $(kubectl -n "$NS" get pod -l 'app.kubernetes.io/component=web,!lab' -o jsonpath='{.items[*].metadata.name}'); do
  N="$(kubectl -n "$NS" get pod "$P" -o jsonpath='{.spec.nodeName}')"
  for i in $(seq 1 10); do
    kubectl -n "$NS" exec "$P" -c status-writer -- wget -qO- -T 2 http://node-agent:9100/node.json
  done > "$EV/b4-agent-$P.txt" 2>/dev/null
  SEEN="$(python3 -c 'import json,sys; print(" ".join(sorted({json.loads(l)["node"] for l in sys.stdin if l.strip()})))' < "$EV/b4-agent-$P.txt")"
  [ "$SEEN" = "$N" ] || { echo "FAIL: $P tren $N doc agent cua: $SEEN"; BAD=1; }
  S="$(kubectl -n "$NS" exec "$P" -c status-writer -- wget -qO- http://127.0.0.1:8080/status.json)"
  echo "$S" >> "$EV/b4-status.txt"
  R="$(echo "$S" | python3 -c 'import json,sys; d=json.load(sys.stdin); a=d.get("agent") or {}; print(" ".join(sorted(d)), d.get("pod"), d.get("node"), a.get("node"))')"
  [ "$R" = "agent node pod updated version $P $N $N" ] || { echo "FAIL: status.json cua $P: $R"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: moi Pod web chi doc agent cung node; status.json du nam truong, agent.node trung node'

# Điều tra bài 87 — lời gọi thất bại chạy trên lab-k8s-worker2
kubectl apply -f "$W/itp-thu.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/itp-be pod/hoi-w1 pod/hoi-w2 --timeout=180s
OK1=1; for i in $(seq 1 10); do kubectl -n "$NS" exec hoi-w1 -- wget -qO- -T 3 http://itp-thu/ > "$EV/b4-itp-w1.txt" 2>&1 && { OK1=0; break; }; sleep 2; done
[ "$OK1" -eq 0 ] && echo 'PASS: client tren worker1 (co endpoint) goi duoc itp-thu'
kubectl -n "$NS" exec hoi-w2 -- wget -qO- -T 3 http://itp-thu/ > "$EV/b4-itp-w2.txt" 2>&1 \
  && echo 'FAIL: client tren worker2 van goi duoc' \
  || echo 'PASS: client tren worker2 khong goi duoc — Service hanh xu nhu khong co endpoint nao'
kubectl -n "$NS" exec hoi-w2 -- nslookup itp-thu > "$EV/b4-itp-dns.txt" 2>&1 \
  && echo 'PASS: ten itp-thu van phan giai tu worker2 — loi nam o dinh tuyen, khong o DNS'
kubectl delete -f "$W/itp-thu.yaml" --wait=true >/dev/null \
  && [ "$(kubectl -n "$NS" get pod,svc -l lab=itp-thu --no-headers 2>/dev/null | wc -l)" -eq 0 ] \
  && echo 'PASS: da xoa object thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/agent/service.yaml — Lab 11a chuyển cùng node-agent sang namespace bookstore-system
apiVersion: v1
kind: Service
metadata:
  name: node-agent
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: agent
    app.kubernetes.io/name: busybox
spec:
  internalTrafficPolicy: Local    # chỉ dùng agent trên cùng node với Pod gọi
  selector:
    app.kubernetes.io/part-of: bookstore
    app.kubernetes.io/component: agent
  ports:
  - name: http
    port: 9100
    targetPort: http
```

Trong `k8s/app/web/deployment.yaml`, sidecar `status-writer` (mục trong `initContainers`) thay khối
`command`; `env`, `volumeMounts`, `resources`, `restartPolicy: Always` giữ nguyên:

```yaml
      - name: status-writer
        image: busybox:1.37
        restartPolicy: Always
        # env POD_NAME, NODE_NAME; volumeMounts html, podinfo; resources: giữ nguyên
        command:
        - sh
        - -c
        - |
          trap 'echo "status-writer: nhan TERM"; exit 0' TERM
          while true; do
            V="$(sed -n 's/^app\.kubernetes\.io\/version="\(.*\)"$/\1/p' /podinfo/labels)"
            A="$(wget -qO- -T 2 http://node-agent:9100/node.json 2>/dev/null || true)"
            [ -n "$A" ] || A=null
            printf '{"pod":"%s","node":"%s","version":"%s","updated":"%s","agent":%s}\n' \
              "$POD_NAME" "$NODE_NAME" "$V" "$(date -Iseconds)" "$A" > /html/status.json.tmp
            mv /html/status.json.tmp /html/status.json
            sleep 5 & wait $!
          done
```

```yaml
# ~/lab-work/5a/itp-thu.yaml — điều tra bài 87; xóa khi xong
apiVersion: v1
kind: Pod
metadata:
  name: itp-be
  namespace: bookstore
  labels: {lab: itp-thu, vai-tro: backend}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker1}
  terminationGracePeriodSeconds: 1
  containers:
  - name: httpd
    image: busybox:1.37
    command: ["sh", "-c", "mkdir -p /www && echo itp-be > /www/index.html && exec httpd -f -p 8080 -h /www"]
---
apiVersion: v1
kind: Service
metadata:
  name: itp-thu
  namespace: bookstore
  labels: {lab: itp-thu}
spec:
  internalTrafficPolicy: Local
  selector: {lab: itp-thu, vai-tro: backend}
  ports:
  - {name: http, port: 80, targetPort: 8080}
---
apiVersion: v1
kind: Pod
metadata:
  name: hoi-w1
  namespace: bookstore
  labels: {lab: itp-thu}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker1}
  terminationGracePeriodSeconds: 1
  containers:
  - {name: shell, image: "busybox:1.37", command: ["sleep", "3600"]}
---
apiVersion: v1
kind: Pod
metadata:
  name: hoi-w2
  namespace: bookstore
  labels: {lab: itp-thu}
spec:
  nodeSelector: {kubernetes.io/hostname: lab-k8s-worker2}
  terminationGracePeriodSeconds: 1
  containers:
  - {name: shell, image: "busybox:1.37", command: ["sleep", "3600"]}
```

`deploy.sh` chỉ áp những gì `k8s/app/kustomization.yaml` liệt kê: thêm dòng `- agent/service.yaml` vào
`resources`, ngay sau `- agent/daemonset.yaml`, trước khi triển khai.

```bash
cd ~/bookstore
grep -qx -- '- agent/service.yaml' k8s/app/kustomization.yaml \
  || sed -i 's|^- agent/daemonset.yaml$|&\n- agent/service.yaml|' k8s/app/kustomization.yaml
./scripts/deploy.sh
kubectl -n bookstore get service node-agent
kubectl -n bookstore exec deploy/web -c status-writer -- wget -qO- http://127.0.0.1:8080/status.json
```

Ý chính cho ADR:

1. Mặc định `Cluster`: kube-proxy chọn trong cả ba agent, nên khoảng hai phần ba số lần `status.json`
   báo tải của node khác. JSON vẫn hợp lệ, mọi thứ trông bình thường — chỉ so `agent.node` với `node`
   mới thấy sai. Đó là lý do ghi cả hai.
2. Bài 87: Pod trên node không có endpoint thấy Service như không có endpoint nào. Xảy ra khi agent
   trên node đó đang khởi động lại hoặc chưa Ready, khi node mới vào mà Pod DaemonSet chưa lên, khi
   một taint mới mà agent không dung thứ đuổi nó đi. Sidecar ghi `agent: null` và chạy tiếp; trang
   web vẫn phục vụ; kiểm thử khói bắt được.
3. Không. Bài 87 định nghĩa "nội bộ" là lưu lượng xuất phát từ Pod trong cluster; lưu lượng vào
   NodePort thuộc `externalTrafficPolicy` (B9).
4. DaemonSet bảo đảm **mỗi** node đủ điều kiện có đúng một Pod, nên mọi client đều có endpoint cục bộ.
   Deployment hai bản sao chỉ phủ tối đa hai node; client ở node thứ ba luôn thất bại — đúng kịch bản
   `hoi-w2`.

</details>

## B5. [BS-5A-05] Diễn tập: EndpointSlice theo selector, readiness và Pod đang tắt

**Loại:** Diễn tập sự cố — trên `lab-k8s-worker2`.

**Bối cảnh:** hai câu hỏi treo từ các sprint trước. Lab 3a để lại nợ: readiness thất bại thì Service
làm gì? Lab 3a B6 hỏi: vì sao nginx cần `preStop` để các nơi khác kịp thôi gửi request? Và sprint này
tuyên bố đã trả `TD-03` — phải chứng minh xóa `db-0` không còn cần ai chạy `deploy.sh`. **Đọc lại bài
[49](../49-probes-vi.md), mục [*Readiness probe*](../49-probes-vi.md#readiness-probe), trước khi làm
ticket này — đây là chỗ trả nợ.**

**Yêu cầu:**

Kịch bản A — một backend lạ trong Service `web`. Pod trần `web-le` ở `~/lab-work/5a/web-le.yaml`, ghim
`lab-k8s-worker2`, mang đúng cặp label selector của Service `web` cộng label `lab: web-le`; container
`busybox` phục vụ chuỗi `web-le` ở `/`, có cổng **tên `http` nhưng số 8081**, readiness HTTP đọc một
file, và `preStop` giữ Pod chừng 20 giây khi bị xóa. Chứng minh:

1. Service chọn **Pod**, không chọn controller: `web-le` không có chủ, Deployment `web` vẫn 3 bản sao,
   nhưng lưu lượng NodePort tới được `web-le`.
2. Cùng tên cổng mà khác số cổng: Service `web` có từ hai EndpointSlice trở lên; phải gộp mọi slice
   mới ra đủ bốn backend.
3. Làm `web-le` NotReady mà không giết container: endpoint của nó **vẫn nằm trong slice** với
   `ready=false`, và không request nào tới nó. Ready lại thì nhận lại.
4. Gỡ label thành phần: `web-le` rời Service; trả label: quay lại.
5. Xóa `web-le`: trong lúc `preStop` chạy, endpoint mang `terminating=true`, `ready=false`; Pod xóa
   xong thì endpoint biến mất.

Kịch bản B — DB bị thay:

6. Xóa `db-0` (xóa êm). Chứng minh: mọi endpoint của `api` chuyển `ready=false`; `/api/books` qua
   NodePort trả lỗi trong lúc đó; `postgrest` không bị khởi động lại.
7. **Không** chạy `deploy.sh`: `api` tự Ready lại, `/api/books` trả dữ liệu — `TD-03` đã trả. Ghi lại
   số sách lúc đó và giải thích.
8. Khôi phục dữ liệu migration bằng đúng một lệnh của repo.
9. Viết postmortem.

**Ràng buộc:** không xóa cưỡng bức. Không sửa repo trong lúc diễn tập. `web-le` bị xóa trong chính
kịch bản A.

**Đọc lại:**

- Bài [49](../49-probes-vi.md#readiness-probe) — câu "controller EndpointSlice sẽ loại bỏ địa chỉ IP
  của Pod khỏi các EndpointSlice"; so với thứ bạn đo được.
- Bài [83](../83-endpoint-slices-vi.md): [*Các condition*](../83-endpoint-slices-vi.md#conditions),
  [*Phân bổ các EndpointSlice*](../83-endpoint-slices-vi.md#phân-bổ-các-endpointslice-distribution-of-endpointslices),
  [*Các endpoint trùng lặp*](../83-endpoint-slices-vi.md#các-endpoint-trùng-lặp-duplicate-endpoints).
- Bài [47](../47-pod-lifecycle-vi.md#pod-termination-flow), *Luồng kết thúc Pod* — bước loại Pod khỏi
  EndpointSlice diễn ra song song với `preStop`.
- Bài [64](../64-replicaset-vi.md#cô-lập-pod-khỏi-một-replicaset-isolating-pods-from-a-replicaset) —
  đổi label để cô lập Pod; bài [63](../63-deployment-vi.md#selector) — cảnh báo về Pod trùng selector.

**Bàn giao:** `~/lab-work/5a/web-le.yaml`; `docs/postmortem/5a-endpointslice.md`;
`~/lab-evidence/5a/b5-*`.

**Nghiệm thu:**

- `PASS:` `web-le` trên `lab-k8s-worker2`, không có chủ; Deployment `web` vẫn 3 bản sao; NodePort đưa
  được request tới `web-le`.
- `PASS:` Service `web` có từ hai slice, cổng 8080 và 8081; gộp lại đủ 4 backend sẵn sàng.
- `PASS:` `web-le` NotReady: endpoint còn trong slice với `ready=false serving=false`; 0/40 request tới
  nó; Ready lại được.
- `PASS:` gỡ label thì rời mọi slice, trả label thì quay lại.
- `PASS:` lúc đang tắt: `ready=false`, `terminating=true`; xóa xong thì endpoint biến mất.
- `PASS:` xóa `db-0`: cả hai endpoint `api` từng `ready=false`, `/api/books` từng lỗi, `postgrest`
  không khởi động lại.
- `PASS:` không chạy `deploy.sh`, `api` tự lành và `/api/books` trả 5 cuốn — `TD-03` đã trả, `TD-04`
  còn đó.
- `PASS:` sau `./scripts/deploy.sh`, `/api/books` trả ít nhất 8 cuốn.

**Câu hỏi thiết kế** — trả lời trong postmortem:

1. Bài 49 nói readiness thất bại thì controller "loại bỏ địa chỉ IP của Pod khỏi các EndpointSlice".
   Bạn đo được gì trong slice? "Loại bỏ" đúng theo nghĩa nào?
2. Vì sao đọc một slice của `web` rồi kết luận số backend là sai — chứng minh bằng chính slice của
   `web-le`.
3. `terminating=true` xuất hiện lúc nào so với lúc container nhận tín hiệu dừng? Nối với `preStop` 5
   giây của nginx ở Lab 3a B6: vì sao vài giây đó giúp không request nào bị đứt?
4. So với Lab 4b B8: phần nào giờ tự lành, phần nào vẫn không — gọi đúng mã nợ.
5. Bài 64: đổi label để cô lập một Pod. Làm việc đó với một Pod `web` **thật** thì số bản sao và
   Service ra sao? Khi nào người trực nên dùng cách này?
6. `web-le` khớp cả selector của Deployment `web`. Bài 63 cảnh báo gì, và vì sao thí nghiệm ngắn này
   vẫn chấp nhận được còn đặt vào repo thì không?

<details>
<summary>Gợi ý</summary>

- `busybox` phục vụ thư mục bằng `httpd -f -p <cổng> -h <thư mục>`; dùng `exec` để `httpd` là PID 1.
- Readiness `httpGet` tới một file như `/healthz` trong thư mục đó; xóa file bằng `kubectl exec` là
  NotReady mà không giết container.
- `preStop` chạy `sleep 20`; ân hạn phải dài hơn.
- Gỡ một label: `kubectl label pod web-le app.kubernetes.io/component-`.
- Đọc condition của mọi endpoint:
  `-o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]} {.conditions.ready} {.conditions.serving} {.conditions.terminating}{"\n"}{end}'`.
- Xóa không chờ: `kubectl delete pod ... --wait=false`.

</details>

<details>
<summary>Script nghiệm thu — script này chính là cuộc diễn tập; chạy liền mạch từng kịch bản</summary>

Kịch bản A:

```bash
NS=bookstore; EV=~/lab-evidence/5a; W=~/lab-work/5a
nip() { kubectl get node "$1" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'; }
W1="$(nip lab-k8s-worker1)"
cond() { kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=web \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]} {.conditions.ready} {.conditions.serving} {.conditions.terminating}{"\n"}{end}'; }
uniq_ready() { cond | awk '$2=="true"{print $1}' | sort -u | grep -c .; }
hits() { H=0; for i in $(seq 1 "$1"); do curl -s -m 3 "http://$W1:30080/" | grep -qx 'web-le' && H=$((H+1)); done; echo "$H"; }

kubectl apply -f "$W/web-le.yaml"
kubectl -n "$NS" wait --for=condition=Ready pod/web-le --timeout=120s
LE_IP="$(kubectl -n "$NS" get pod web-le -o jsonpath='{.status.podIP}')"
{ [ "$(kubectl -n "$NS" get pod web-le -o jsonpath='{.spec.nodeName}')" = 'lab-k8s-worker2' ] && \
  [ -z "$(kubectl -n "$NS" get pod web-le -o jsonpath='{.metadata.ownerReferences}')" ] && \
  [ "$(kubectl -n "$NS" get deployment web -o jsonpath='{.status.replicas}')" = '3' ]; } \
  && echo 'PASS: web-le tren worker2, khong co chu; Deployment web van 3 ban sao'
for i in $(seq 1 30); do [ "$(uniq_ready)" -eq 4 ] && break; sleep 1; done
cond > "$EV/b5-slices.txt"
NSL="$(kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=web -o name | grep -c .)"
PORTS="$(kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=web -o jsonpath='{range .items[*]}{.ports[0].port}{"\n"}{end}' | sort -u | paste -sd' ' -)"
{ [ "$NSL" -ge 2 ] && [ "$PORTS" = '8080 8081' ] && [ "$(uniq_ready)" -eq 4 ]; } \
  && echo "PASS: Service web co $NSL EndpointSlice (cong $PORTS); gop lai moi du 4 backend"
H="$(hits 40)"; echo "san sang: web-le nhan $H/40" | tee "$EV/b5-hits.txt"
[ "$H" -ge 1 ] && echo 'PASS: NodePort dua duoc request toi Pod tran khop selector'

kubectl -n "$NS" exec web-le -- rm -f /www/healthz
kubectl -n "$NS" wait --for=condition=Ready=false pod/web-le --timeout=60s
for i in $(seq 1 30); do cond | grep -q "^$LE_IP false" && break; sleep 1; done
cond | grep "^$LE_IP " | tee "$EV/b5-notready.txt"
grep -q "^$LE_IP false false" "$EV/b5-notready.txt" \
  && echo 'PASS: Pod NotReady van nam trong slice voi ready=false serving=false'
H="$(hits 40)"; echo "NotReady: web-le nhan $H/40" | tee -a "$EV/b5-hits.txt"
[ "$H" -eq 0 ] && echo 'PASS: Pod NotReady khong nhan request nao'
kubectl -n "$NS" exec web-le -- sh -c 'echo ok > /www/healthz'
kubectl -n "$NS" wait --for=condition=Ready pod/web-le --timeout=60s && echo 'PASS: web-le Ready lai'

kubectl -n "$NS" label pod web-le app.kubernetes.io/component-
for i in $(seq 1 30); do cond | grep -q "^$LE_IP " || break; sleep 1; done
{ ! cond | grep -q "^$LE_IP " && [ "$(uniq_ready)" -eq 3 ]; } && echo 'PASS: go label thi web-le roi khoi moi slice'
kubectl -n "$NS" label pod web-le app.kubernetes.io/component=web
for i in $(seq 1 30); do [ "$(uniq_ready)" -eq 4 ] && break; sleep 1; done
[ "$(uniq_ready)" -eq 4 ] && echo 'PASS: tra label thi web-le quay lai Service'

kubectl -n "$NS" delete pod web-le --wait=false
SEEN=''
for i in $(seq 1 60); do
  L="$(cond | grep "^$LE_IP " || true)"
  case "$L" in *' true') SEEN="$L"; break ;; esac
  sleep 1
done
echo "trong luc tat: $SEEN" | tee "$EV/b5-terminating.txt"
case "$SEEN" in "$LE_IP false "*' true') echo 'PASS: endpoint dang tat mang ready=false, terminating=true' ;;
  *) echo 'FAIL: khong bat duoc cua so terminating' ;; esac
kubectl -n "$NS" wait --for=delete pod/web-le --timeout=90s
for i in $(seq 1 30); do cond | grep -q "^$LE_IP " || break; sleep 1; done
cond | grep -q "^$LE_IP " && echo 'FAIL: endpoint web-le van con' || echo 'PASS: Pod xoa xong thi endpoint bien mat'
```

Kịch bản B:

```bash
NS=bookstore; EV=~/lab-evidence/5a
cd ~/bookstore
nip() { kubectl get node "$1" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'; }
W1="$(nip lab-k8s-worker1)"
API='app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=api'
api_ready() { kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=api \
  -o jsonpath='{range .items[*].endpoints[*]}{.conditions.ready}{"\n"}{end}' | grep -c '^true$'; }
rc() { kubectl -n "$NS" get pod -l "$API" \
  -o jsonpath='{range .items[*]}{.status.containerStatuses[?(@.name=="postgrest")].restartCount}{"\n"}{end}' \
  | awk '{s+=$1} END{print s+0}'; }

R0="$(rc)"; DIP0="$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.podIP}')"
kubectl -n "$NS" delete pod db-0 --wait=false
MIN=9; LOI=0; : > "$EV/b5-db.txt"
for i in $(seq 1 120); do
  N="$(api_ready)"; [ "$N" -lt "$MIN" ] && MIN="$N"
  C="$(curl -s -o /dev/null -m 3 -w '%{http_code}' "http://$W1:30080/api/books")"
  [ "$C" != '200' ] && LOI=$((LOI + 1))
  echo "$(date +%T) api_ready=$N http=$C" >> "$EV/b5-db.txt"
  { [ "$MIN" -eq 0 ] && [ "$N" -eq 2 ] && [ "$C" = '200' ]; } && break
  sleep 2
done
[ "$MIN" -eq 0 ] && echo 'PASS: khi db-0 bi thay, ca hai endpoint api deu ready=false'
[ "$LOI" -ge 1 ] && echo "PASS: /api/books qua NodePort loi $LOI lan trong luc DB vang mat"
kubectl -n "$NS" wait --for=condition=Ready pod -l "$API" --timeout=180s
DIP1="$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.podIP}')"
echo "db-0 cu=$DIP0 moi=$DIP1" | tee -a "$EV/b5-db.txt"
[ "$(rc)" = "$R0" ] && echo 'PASS: postgrest khong bi khoi dong lai'
T="$(curl -s -m 5 "http://$W1:30080/api/books" | grep -o '"title"' | wc -l)"
[ "$T" -ge 5 ] && echo "PASS: TD-03 DA TRA — khong chay deploy.sh, api tu Ready lai, /api/books tra $T cuon"
N="$(kubectl -n "$NS" exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books')"
[ "$N" = '5' ] && echo 'PASS: con 5 cuon cua script khoi tao — du lieu migration mat cung emptyDir (TD-04)'
./scripts/deploy.sh > "$EV/b5-deploy.txt" 2>&1
T="$(curl -s -m 5 "http://$W1:30080/api/books" | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && echo "PASS: deploy.sh chay lai migration, /api/books tra $T cuon"
```

IP mới của `db-0` thường khác IP cũ; trùng thì kịch bản vẫn đúng — tên DNS không phụ thuộc IP.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/5a/web-le.yaml — Pod trần lọt vào Service web; xóa trong chính kịch bản A
apiVersion: v1
kind: Pod
metadata:
  name: web-le
  namespace: bookstore
  labels:
    app.kubernetes.io/part-of: bookstore    # bắt buộc: selector của Service web
    app.kubernetes.io/component: web        # bắt buộc: selector của Service web
    lab: web-le
spec:
  nodeSelector:
    kubernetes.io/hostname: lab-k8s-worker2
  terminationGracePeriodSeconds: 40
  containers:
  - name: httpd
    image: busybox:1.37
    command: ["sh", "-c", "mkdir -p /www && echo web-le > /www/index.html && echo ok > /www/healthz && exec httpd -f -p 8081 -h /www"]
    ports:
    - name: http                  # cùng tên với cổng của Pod web thật, khác số
      containerPort: 8081
    readinessProbe:
      httpGet: {path: /healthz, port: http}
      periodSeconds: 2
    lifecycle:
      preStop:
        exec: {command: ["sleep", "20"]}
    resources:
      requests: {cpu: 10m, memory: 8Mi}
      limits: {cpu: 50m, memory: 16Mi}
```

Ý chính cho postmortem:

1. Endpoint của Pod NotReady **vẫn có** trong slice, mang `ready=false`, `serving=false`; nó bị loại
   khỏi **tập đích được dùng** — kube-proxy không gửi request nào tới (0/40). "Loại bỏ" đúng theo
   nghĩa định tuyến, không theo nghĩa xóa dòng khỏi object.
2. Bài 83: endpoint được nhóm theo tổ hợp họ IP, giao thức, **số cổng**. `web-le` giải tên `http` ra
   8081 nên nằm ở slice riêng; slice 8080 chỉ có ba endpoint trong khi Service có bốn. Người đọc phải
   gộp mọi slice mang `kubernetes.io/service-name=web` rồi khử trùng lặp.
3. `terminating` được đặt khi Pod nhận timestamp xóa — trước khi container thoát (bài 83, 47). Việc
   các node thôi gửi request diễn ra **song song** với `preStop`, không trước nó; `preStop` giữ nginx
   còn phục vụ trong lúc kube-proxy trên mọi node cập nhật, nên request đang bay không rơi vào một
   tiến trình đã tắt.
4. Tự lành: StatefulSet tạo lại `db-0`, CoreDNS trả IP mới cho `db-0.db`, PostgREST kết nối lại —
   `TD-03` đã trả. Chưa: dữ liệu sau khởi tạo mất theo `emptyDir` (`TD-04`, Lab 6a), và trong vài chục
   giây `api` không phục vụ vì chỉ có một DB.
5. ReplicaSet thấy thiếu một Pod khớp selector và tạo Pod mới — số bản sao phục vụ giữ nguyên; Pod
   bị cô lập rời Service nhưng vẫn chạy để người trực `exec`, đọc log, chụp trạng thái, rồi xóa tay.
6. Bài 63: Deployment có thể coi Pod khớp selector là của mình; controller có selector chồng lấn sẽ
   tranh nhau. Ở đây ReplicaSet chọn thêm theo `pod-template-hash` nên không nhận `web-le`, và Pod chỉ
   sống vài phút. Trong repo, mọi Pod `web` phải thuộc đúng một controller.

</details>

## B6. [BS-5A-06] Điều tra: DNS trong Pod

**Loại:** Điều tra.

**Bối cảnh:** từ B2 cả hệ thống dựa vào DNS. Người trực cần biết một tên được mở rộng thế nào, vì
sao `api` gọi được từ namespace `bookstore` mà không gọi được từ namespace khác, vì sao một Pod vừa
tạo đôi khi chưa phân giải được, và còn cách khám phá Service nào ngoài DNS.

**Yêu cầu:** chứng minh từng điều dưới đây và ghi kết luận vào ADR.

1. `/etc/resolv.conf` của một Pod `web`: `nameserver` chính là ClusterIP của Service DNS của cluster;
   danh sách `search` bắt đầu bằng `bookstore.svc.cluster.local svc.cluster.local cluster.local`;
   có `ndots:5`; Pod dùng chính sách DNS mặc định. Addon DNS là CoreDNS do kubeadm cài, Service của
   nó tên `kube-dns` (bài 362) — chỉ đọc.
2. Bốn cách viết tên `api` — tên ngắn, có namespace, FQDN, FQDN có dấu chấm cuối — cùng ra một
   ClusterIP.
3. Từ một Pod tạm ở namespace `default`: tên ngắn `api` không phân giải; `api.bookstore` thì có.
4. Định danh DNS của `db-0`: hostname, FQDN trong `/etc/hosts` của chính nó, và tên `db-0.db` phân
   giải từ Pod khác. Pod `web` không khai `hostname`/`subdomain` thì không có tên cấp Pod.
5. Khám phá bằng biến môi trường: Pod `web` có biến cho Service `api` và `node-agent` theo quy tắc
   đặt tên của bài 82. Một Pod tạo **trước** một Service thử nghiệm không có biến của Service đó,
   nhưng vẫn phân giải được tên của nó; Pod tạo **sau** thì có biến.

**Ràng buộc:** chỉ đọc `kube-system`; không sửa CoreDNS. Pod tạm dùng `--rm`; namespace `default`
không còn Pod nào khi xong. Object thử nghiệm mang label `lab` và bị xóa.

**Đọc lại:**

- Bài [10](../10-dns-pod-service-vi.md): [*Namespace của Service*](../10-dns-pod-service-vi.md#namespace-của-service-namespaces-of-services),
  [*Service*](../10-dns-pod-service-vi.md#services),
  [*Các trường hostname và subdomain*](../10-dns-pod-service-vi.md#pod-hostname-and-subdomain-field),
  [*Chính sách DNS của Pod*](../10-dns-pod-service-vi.md#pod-s-dns-policy).
- Bài [57](../57-pod-hostname-vi.md#hostname-với-các-trường-hostname-và-subdomain-của-pod-hostname-with-pods-hostname-and-subdomain-fields).
- Bài [82](../82-service-vi.md): [*Biến môi trường*](../82-service-vi.md#biến-môi-trường-environment-variables),
  [*DNS*](../82-service-vi.md#dns).
- Bài [362](../362-configure-dns-cluster-vi.md) — cả bài.
- Bài [65](../65-statefulset-vi.md#stable-network-id) — negative caching.
- Bài [352](../352-fine-parallel-work-queue-vi.md): [*Nạp tác vụ vào hàng đợi*](../352-fine-parallel-work-queue-vi.md#nạp-tác-vụ-vào-hàng-đợi-filling-the-queue-with-tasks)
  và `worker.py` ở mục [*Tạo container image*](../352-fine-parallel-work-queue-vi.md#create-an-image)
  — đường lui bằng biến môi trường.

**Bàn giao:** `docs/adr/NNNN-dns-trong-pod.md`; `~/lab-evidence/5a/b6-*`.

**Nghiệm thu:**

- `PASS:` `nameserver` bằng ClusterIP của `kube-dns`, `search` bắt đầu bằng namespace của Pod, có
  `ndots:5`; `dnsPolicy` là `ClusterFirst`; addon là CoreDNS.
- `PASS:` bốn cách viết tên `api` cùng ra một ClusterIP.
- `PASS:` từ `default`, `api` không phân giải, `api.bookstore` thì có; `default` không còn Pod.
- `PASS:` `db-0` có hostname `db-0` và FQDN đầy đủ trong `/etc/hosts`; `db-0.db` phân giải ra IP của
  nó; tên cấp Pod của `web` không tồn tại.
- `PASS:` Pod `web` có `API_SERVICE_HOST` và `NODE_AGENT_SERVICE_PORT`; Pod tạo trước Service thử
  nghiệm không có biến nhưng vẫn tìm được bằng DNS; Pod tạo sau có biến.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-dns-trong-pod.md`:

1. Với `ndots:5`, tên `db-0.db`, tên `api.bookstore.svc.cluster.local` và cùng tên đó có dấu chấm cuối
   được thử theo thứ tự nào? Tên nào trúng ngay ở lần đầu? Vì sao dự án chọn `db-0.db` cho `api`?
2. Init container chờ DB của `api` và `db-migrate` có thể chờ lâu hơn thời gian `db-0` cần để Ready.
   Vì sao (bài 65)? Trường `publishNotReadyAddresses` (bài 10, 83) đổi điều đó thế nào, và bạn có bật
   nó cho Service `db` không?
3. Bài 352 để sẵn đường lui `REDIS_SERVICE_HOST` khi DNS hỏng. Đường lui đó đòi điều kiện gì về thứ tự
   tạo object, và nó còn đúng không nếu Service bị xóa rồi tạo lại?
4. Câu bẫy: tên ngắn `api` không gọi được từ namespace `default` — đó có phải cơ chế bảo vệ `api`
   không? Thứ gì mới thực sự cô lập được nó?

<details>
<summary>Gợi ý</summary>

- Đọc file từ một Pod `web`: `kubectl -n bookstore exec deploy/web -c status-writer -- cat /etc/resolv.conf`.
- Pod tạm ở namespace khác:
  `kubectl -n default run dns-ngoai --rm -i --restart=Never --image=busybox:1.37 -- sh -c 'nslookup api; echo ---; nslookup api.bookstore'`.
- Container `postgres` của `db-0` có `hostname`, `cat`.
- Biến môi trường của Service: tên Service viết hoa, `-` thành `_`, thêm `_SERVICE_HOST`,
  `_SERVICE_PORT`.
- Service thử nghiệm nhanh: `kubectl -n bookstore create service clusterip muon --tcp=3000:http`.

</details>

<details>
<summary>Script nghiệm thu — script này chính là cuộc điều tra</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a
WEB="$(kubectl -n "$NS" get pod -l 'app.kubernetes.io/component=web,!lab' -o jsonpath='{.items[0].metadata.name}')"
inpod() { kubectl -n "$NS" exec "$WEB" -c status-writer -- "$@"; }
DNS_IP="$(kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.clusterIP}')"
API_IP="$(kubectl -n "$NS" get svc api -o jsonpath='{.spec.clusterIP}')"

# 1. resolv.conf, chính sách DNS, addon
inpod cat /etc/resolv.conf > "$EV/b6-resolv.txt"
{ grep -qx "nameserver $DNS_IP" "$EV/b6-resolv.txt" && \
  grep -Eq '^search bookstore\.svc\.cluster\.local svc\.cluster\.local cluster\.local' "$EV/b6-resolv.txt" && \
  grep -q 'ndots:5' "$EV/b6-resolv.txt"; } \
  && echo "PASS: nameserver = ClusterIP kube-dns ($DNS_IP), search bat dau bang namespace, ndots:5"
[ "$(kubectl -n "$NS" get pod "$WEB" -o jsonpath='{.spec.dnsPolicy}')" = 'ClusterFirst' ] && echo 'PASS: dnsPolicy mac dinh ClusterFirst'
kubectl -n kube-system get deployment coredns -o jsonpath='{.spec.template.spec.containers[0].image}' | grep -q coredns \
  && echo 'PASS: addon DNS la CoreDNS (Deployment coredns), Service ten kube-dns'

# 2. Bốn cách viết tên
OK=1
for N in api api.bookstore api.bookstore.svc.cluster.local api.bookstore.svc.cluster.local.; do
  inpod nslookup "$N" 2>/dev/null | grep -q "$API_IP" || { echo "FAIL: $N"; OK=0; }
done
[ "$OK" -eq 1 ] && echo "PASS: bon cach viet ten api cung ra $API_IP"

# 3. Ranh giới namespace
kubectl -n default run dns-ngoai --rm -i --restart=Never --image=busybox:1.37 -- \
  sh -c 'nslookup api; echo ---; nslookup api.bookstore' > "$EV/b6-ngoai-ns.txt" 2>&1
{ ! sed -n '1,/^---$/p' "$EV/b6-ngoai-ns.txt" | grep -q "$API_IP" && \
  sed -n '/^---$/,$p' "$EV/b6-ngoai-ns.txt" | grep -q "$API_IP"; } \
  && echo 'PASS: tu namespace default, api khong phan giai, api.bookstore thi co'
[ "$(kubectl -n default get pods --no-headers 2>/dev/null | wc -l)" -eq 0 ] && echo 'PASS: namespace default khong con Pod'

# 4. Định danh DNS của db-0
DB_IP="$(kubectl -n "$NS" get pod db-0 -o jsonpath='{.status.podIP}')"
kubectl -n "$NS" exec db-0 -c postgres -- cat /etc/hosts > "$EV/b6-db-0-hosts.txt"
{ [ "$(kubectl -n "$NS" exec db-0 -c postgres -- hostname)" = 'db-0' ] && \
  grep -q 'db-0\.db\.bookstore\.svc\.cluster\.local' "$EV/b6-db-0-hosts.txt"; } \
  && echo 'PASS: db-0 co hostname db-0, FQDN db-0.db.bookstore.svc.cluster.local trong /etc/hosts'
inpod nslookup db-0.db 2>/dev/null | grep -q "$DB_IP" && echo "PASS: db-0.db phan giai ra $DB_IP"
inpod nslookup "$WEB.web" >/dev/null 2>&1 && echo 'FAIL: Pod web co ten cap Pod' \
  || echo 'PASS: Pod web khong khai hostname/subdomain nen khong co ten cap Pod'

# 5. Biến môi trường so với DNS
inpod env > "$EV/b6-env.txt"
{ grep -qx "API_SERVICE_HOST=$API_IP" "$EV/b6-env.txt" && grep -qx 'NODE_AGENT_SERVICE_PORT=9100' "$EV/b6-env.txt"; } \
  && echo 'PASS: Pod web co API_SERVICE_HOST va NODE_AGENT_SERVICE_PORT (viet hoa, - thanh _)'
kubectl -n "$NS" run dns-cu --image=busybox:1.37 --restart=Never --labels=lab=dns --command -- sleep 3600
kubectl -n "$NS" wait --for=condition=Ready pod/dns-cu --timeout=120s
kubectl -n "$NS" create service clusterip muon --tcp=3000:http
MUON_IP="$(kubectl -n "$NS" get svc muon -o jsonpath='{.spec.clusterIP}')"
kubectl -n "$NS" exec dns-cu -- env | grep -q '^MUON_SERVICE_HOST=' \
  && echo 'FAIL: Pod cu co bien cua Service tao sau' || echo 'PASS: dns-cu tao truoc Service muon nen khong co MUON_SERVICE_HOST'
kubectl -n "$NS" exec dns-cu -- nslookup muon 2>/dev/null | grep -q "$MUON_IP" && echo 'PASS: dns-cu van tim duoc muon bang DNS'
kubectl -n "$NS" run dns-moi --image=busybox:1.37 --restart=Never --labels=lab=dns --command -- sleep 3600
kubectl -n "$NS" wait --for=condition=Ready pod/dns-moi --timeout=120s
kubectl -n "$NS" exec dns-moi -- env | grep -qx "MUON_SERVICE_HOST=$MUON_IP" && echo 'PASS: dns-moi tao sau co MUON_SERVICE_HOST'
kubectl -n "$NS" delete pod -l lab=dns --grace-period=1 --wait=true
kubectl -n "$NS" delete service muon
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của ADR</summary>

1. Tên có ít hơn 5 dấu chấm được ghép lần lượt với từng hậu tố trong `search` trước, hết mới thử
   nguyên văn (bài 10). `db-0.db` (1 dấu chấm): lần đầu đã là `db-0.db.bookstore.svc.cluster.local` —
   trúng. `api.bookstore.svc.cluster.local` (4 dấu chấm): thử
   `api.bookstore.svc.cluster.local.bookstore.svc.cluster.local`, rồi các hậu tố còn lại, cuối cùng mới
   nguyên văn — vài truy vấn bỏ đi. Có dấu chấm cuối thì tên là tuyệt đối, một truy vấn. `db-0.db` ngắn,
   không gắn tên namespace vào cấu hình, và trúng ngay ở hậu tố đầu.
2. Vòng chờ hỏi `db-0.db` khi `db-0` chưa Ready: chưa có bản ghi, CoreDNS trả "không tồn tại" và kết
   quả thất bại đó được nhớ một lúc — bài 65 nói cache của CoreDNS là 30 giây — nên vòng chờ có thể trễ
   thêm tới chừng đó sau khi `db-0` đã Ready. `publishNotReadyAddresses: true` làm bản ghi có ngay cả
   khi Pod chưa Ready, nên không còn câu trả lời âm; `pg_isready` vẫn chặn cho tới khi DB nhận kết nối.
   Đánh đổi: client không kiểm như `pg_isready` sẽ kết nối vào một DB chưa sẵn sàng. Dự án giữ mặc
   định — cái trễ vài chục giây ở lần khởi động chấp nhận được.
3. Bài 82: biến chỉ được nạp cho Service đã tồn tại **trước** khi Pod ra đời — Service phải tạo trước
   Job. Biến được đặt một lần lúc Pod khởi động; Service bị xóa và tạo lại với ClusterIP khác thì biến
   trong Pod đang chạy trỏ sai, còn DNS thì tự cập nhật.
4. Không. Đó chỉ là hệ quả của danh sách `search`; bất kỳ Pod nào ở namespace nào cũng gọi được
   `api.bookstore`. Cô lập là việc của NetworkPolicy — B8 viết, Lab 5b làm cho có hiệu lực.

</details>

## B7. [BS-5A-07] Canary chia lưu lượng thật cho `web`

**Loại:** Điều tra — trả nợ lab của 4a.

**Bối cảnh:** Lab 4a dựng được cấu trúc canary — hai tập Pod chung bộ label, khác một label riêng —
nhưng chưa có Service nên chưa chia được lưu lượng thật. Trước khi phát hành phiên bản `web` tiếp theo,
đội muốn thử nó trên một phần nhỏ người dùng thật. **Đọc lại bài [61](../61-management-vi.md), mục
[*Triển khai canary*](../61-management-vi.md#canary-deployments), trước khi làm ticket này.**

**Yêu cầu:**

1. Deployment thử nghiệm `web-canary`, **một** bản sao, Pod template là bản sao của Pod template
   `web` đang chạy, chỉ khác: label `app.kubernetes.io/version` mang giá trị của bản ổn định thêm hậu
   tố `-canary`, thêm label riêng `track: canary` và `lab: canary`. Manifest ở `~/lab-work/5a/`.
2. Deployment `web` trong repo **không đổi một dòng**; Service `web` không đổi; canary vẫn nhận lưu
   lượng thật từ NodePort.
3. Đo: trong 60 request `/status.json` qua NodePort, có cả phiên bản ổn định lẫn phiên bản canary;
   ghi tỷ lệ đo được và so với tỷ lệ bản sao.
4. Rút canary: xóa nó, chứng minh không request nào còn tới phiên bản canary.

**Ràng buộc:** canary là thử nghiệm — không vào `k8s/`, xóa khi xong. Không `kubectl scale` hay
`patch` Deployment `web`.

**Đọc lại:**

- Bài [61](../61-management-vi.md#canary-deployments) — tỷ lệ lưu lượng do số bản sao quyết định.
- Bài [363](../363-connecting-frontend-backend-vi.md#tạo-đối-tượng-service-hello-creating-the-hello-service-object)
  — Service chọn tập con chung của label, `track` nằm ngoài selector.
- Bài [63](../63-deployment-vi.md#selector) — `.spec.selector` bất biến, và cảnh báo về Pod khác
  khớp selector.

**Bàn giao:** `~/lab-work/5a/web-canary.json` (hoặc `.yaml`); `docs/adr/NNNN-canary.md`;
`~/lab-evidence/5a/b7-*`.

**Nghiệm thu:**

- `PASS:` đúng một Pod canary Ready, version kết thúc bằng `-canary`; selector của canary có `track`,
  Pod canary thuộc ReplicaSet của canary; Deployment `web` vẫn 3/3, repo không đổi.
- `PASS:` Service `web` có 4 endpoint sẵn sàng khi canary chạy.
- `PASS:` 60 request qua NodePort chạm cả hai phiên bản; tỷ lệ canary được ghi lại.
- `PASS:` xóa canary: Service về 3 endpoint, 30 request không còn phiên bản canary; không còn object
  thử nghiệm, `k8s/` không chứa canary.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-canary.md`:

1. Vì sao Service `web` chia lưu lượng cho canary mà không phải sửa gì? Tỷ lệ kỳ vọng là bao nhiêu,
   tỷ lệ đo được là bao nhiêu, và bạn đổi tỷ lệ bằng cách nào?
2. Selector của Deployment `web` theo hợp đồng cũng khớp Pod canary. Bài 63 cảnh báo gì? Thiết kế
   đúng theo bài 61 đặt `track` vào selector của bản ổn định ngay từ đầu — vì sao dự án không sửa
   được điều đó bằng một lần `apply`?
3. Canary chạy tốt. Quy trình "lên chính thức" gồm những bước nào trên **repo**, và vì sao không được
   để canary sống trên cluster mà không có trong repo?
4. PodDisruptionBudget `web` đếm Pod nào trong lúc canary chạy?

<details>
<summary>Gợi ý</summary>

- Cách nhanh để có bản sao đúng Pod template đang chạy: lấy Deployment `web` dạng JSON
  (`kubectl -n bookstore get deployment web -o json`), dùng `python3` dựng một Deployment mới chỉ giữ
  `spec.template`, đổi label như yêu cầu, đặt `replicas: 1` và selector có thêm `track: canary`.
- Selector của canary phải khớp label template của chính nó; selector đó là bất biến sau khi tạo.
- Đọc `version` trong `status.json` bằng `python3 -c 'import json,sys; ...'`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi canary Ready</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a
cd ~/bookstore
nip() { kubectl get node "$1" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'; }
W1="$(nip lab-k8s-worker1)"; W2="$(nip lab-k8s-worker2)"
uniq_ready() { kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=web \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]} {.conditions.ready}{"\n"}{end}' | awk '$2=="true"{print $1}' | sort -u | grep -c .; }
versions() { python3 -c '
import json, sys
for l in sys.stdin:
    try: print(json.loads(l)["version"])
    except Exception: print("loi")'; }

STABLE="$(kubectl -n "$NS" get deployment web -o jsonpath='{.spec.template.metadata.labels.app\.kubernetes\.io/version}')"
CAN="$(kubectl -n "$NS" get pod -l 'app.kubernetes.io/component=web,track=canary' -o jsonpath='{.items[*].metadata.name}')"
CV="$(kubectl -n "$NS" get pod $CAN -o jsonpath='{.metadata.labels.app\.kubernetes\.io/version}' 2>/dev/null)"
{ [ "$(echo $CAN | wc -w)" -eq 1 ] && [ "$CV" = "${STABLE}-canary" ] && \
  [ "$(kubectl -n "$NS" get pod $CAN -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}')" = 'True' ]; } \
  && echo "PASS: mot Pod canary Ready, version $CV (on dinh: $STABLE)"
OWN="$(kubectl -n "$NS" get pod $CAN -o jsonpath='{.metadata.ownerReferences[0].name}' 2>/dev/null)"
{ [ "$(kubectl -n "$NS" get deployment web-canary -o jsonpath='{.spec.selector.matchLabels.track}')" = 'canary' ] && \
  case "$OWN" in web-canary-*) true ;; *) false ;; esac; } \
  && echo "PASS: selector cua canary co track=canary; Pod canary thuoc $OWN, khong bi web nhan"
{ [ "$(kubectl -n "$NS" get deployment web -o jsonpath='{.status.readyReplicas}/{.spec.replicas}')" = '3/3' ] && \
  ! grep -rqE 'web-canary|track: canary' k8s; } \
  && echo 'PASS: Deployment web van 3/3, repo khong co gi cua canary'
for i in $(seq 1 30); do [ "$(uniq_ready)" -eq 4 ] && break; sleep 1; done
[ "$(uniq_ready)" -eq 4 ] && echo 'PASS: Service web co 4 endpoint san sang (3 on dinh + 1 canary)'

for i in $(seq 1 30); do curl -s -m 3 "http://$W1:30080/status.json"; curl -s -m 3 "http://$W2:30080/status.json"; done \
  | versions > "$EV/b7-versions.txt"
C="$(grep -cx -- "${STABLE}-canary" "$EV/b7-versions.txt")"; S="$(grep -cx -- "$STABLE" "$EV/b7-versions.txt")"
echo "canary=$C on-dinh=$S tren 60 request (ty le ban sao 1:3, ky vong khoang 15 canary)" | tee "$EV/b7-ty-le.txt"
{ [ "$C" -ge 1 ] && [ "$S" -ge 1 ]; } && echo 'PASS: Service web chia luu luong that cho ca hai phien ban'

kubectl -n "$NS" delete deployment web-canary --wait=true
kubectl -n "$NS" wait --for=delete pod -l track=canary --timeout=120s
for i in $(seq 1 30); do [ "$(uniq_ready)" -eq 3 ] && break; sleep 1; done
for i in $(seq 1 30); do curl -s -m 3 "http://$W1:30080/status.json"; done | versions > "$EV/b7-sau-rut.txt"
{ [ "$(uniq_ready)" -eq 3 ] && ! grep -q -- '-canary$' "$EV/b7-sau-rut.txt"; } \
  && echo 'PASS: rut canary — Service ve 3 endpoint, khong request nao toi phien ban canary'
{ [ "$(kubectl -n "$NS" get deployment,pod -l lab=canary --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  ! grep -rq 'web-canary' k8s; } && echo 'PASS: khong con object canary, k8s/ khong chua canary'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
mkdir -p ~/lab-work/5a
kubectl -n bookstore get deployment web -o json > ~/lab-work/5a/web-live.json
python3 - ~/lab-work/5a/web-live.json > ~/lab-work/5a/web-canary.json <<'PY'
import json, sys
live = json.load(open(sys.argv[1]))
tpl = live["spec"]["template"]
lab = tpl["metadata"]["labels"]
lab["app.kubernetes.io/version"] = lab.get("app.kubernetes.io/version", "0.0.0") + "-canary"
lab["track"] = "canary"
lab["lab"] = "canary"
sel = dict(live["spec"]["selector"]["matchLabels"], track="canary")
json.dump({
    "apiVersion": "apps/v1", "kind": "Deployment",
    "metadata": {"name": "web-canary", "namespace": "bookstore", "labels": {"lab": "canary"}},
    "spec": {"replicas": 1, "selector": {"matchLabels": sel}, "template": tpl},
}, sys.stdout, indent=2)
PY
kubectl apply -f ~/lab-work/5a/web-canary.json
kubectl -n bookstore rollout status deployment/web-canary --timeout=180s
kubectl -n bookstore get pod -l app.kubernetes.io/component=web -L track,app.kubernetes.io/version
```

Ý chính cho ADR:

1. Selector của Service `web` chỉ có `part-of` và `component` — tập con chung của hai bản phát hành;
   `track` và `version` nằm ngoài nó (bài 61, cùng cách bài 363 tách bằng `tier`). Kube-proxy chọn
   gần như đều giữa bốn endpoint, nên kỳ vọng 1/4 ≈ 15/60; con số đo dao động quanh đó. Đổi tỷ lệ bằng
   số bản sao của từng bên.
2. Bài 63: Deployment có thể coi mọi Pod khớp selector là của mình; controller có selector chồng lấn
   sẽ tranh nhau. Chiều canary → ổn định không chồng: selector của `web-canary` có `track: canary`, Pod
   của `web` không mang label đó. Chiều ổn định → canary có chồng: selector `part-of` + `component` của
   `web` khớp cả Pod canary. Không thành tranh chấp vì Deployment làm việc qua ReplicaSet, selector của
   mỗi ReplicaSet có thêm `pod-template-hash` riêng, và Pod canary đã có `ownerReferences` trỏ về
   ReplicaSet của `web-canary` — ReplicaSet chỉ nhận Pod mồ côi (Lab 4a B2). Rủi ro còn lại: PDB `web`
   và mọi selector chọn theo hai label đó đếm cả canary. Đặt `track: stable` vào selector của `web` là
   đổi `.spec.selector` — bất biến trong `apps/v1` — tức phải xóa `--cascade=orphan` rồi tạo lại
   Deployment (Lab 4a B9); dự án chọn chấp nhận chồng lấn có kiểm soát vì canary chỉ sống vài phút và
   không bao giờ vào repo.
3. Lên chính thức: đổi `version` (và image nếu có) trong `k8s/app/web/deployment.yaml`, commit,
   `./scripts/deploy.sh` — RollingUpdate như mọi lần phát hành; rồi xóa canary. Repo là nguồn sự thật
   duy nhất ([mục 5.3](DU-AN-BOOKSTORE.md#53-quy-tắc-làm-việc-với-repo)): object chỉ sống trên cluster
   sẽ mất ở lần teardown và không ai review được.
4. PDB `web` chọn theo `part-of` + `component`, nên đếm cả Pod canary — bốn Pod. Một lần trục xuất tự
   nguyện có thể rơi vào Pod canary.

</details>

## B8. [BS-5A-08] NetworkPolicy theo ma trận — chưa có hiệu lực

**Loại:** Tính năng — phát sinh `TD-09`, nợ lab #4 vẫn mở.

**Bối cảnh:** hệ thống cũ chạy trên một mạng phẳng (`P7`); hôm nay cluster cũng vậy — Pod nào gọi Pod
nào cũng được. Dự án đã có ma trận lưu lượng ở
[mục 3.5](DU-AN-BOOKSTORE.md#35-thiết-kế-bảo-mật). Sprint này viết nó thành NetworkPolicy và đưa vào
repo, dù Flannel của baseline **không** thực thi gì; Lab 5b đổi CNI và cho chúng hiệu lực.

**Yêu cầu:**

1. NetworkPolicy ở `k8s/app/network/`, nằm trong kustomization. Mặc định chặn **cả hai chiều** cho mọi
   Pod trong `bookstore`; chỉ mở đúng các dòng của ma trận mục 3.5: Pod ở namespace `traefik` → `web`
   8080; `web` → `api` 3000; `api`, `db-migrate`, `db-backup` → `db` 5432; mọi Pod → CoreDNS ở
   `kube-system` cổng 53 UDP và TCP; `web` → `node-agent` 9100.
2. Selector dùng cặp `part-of` + `component` theo [mục 4.1](DU-AN-BOOKSTORE.md#41-tên-và-label).
   Namespace được chọn theo label tên do control plane đặt sẵn.
3. Một bộ mô phỏng (script nghiệm thu) đọc các policy trên cluster và xác nhận: mọi luồng của ma trận
   được phép, các luồng sai bị chặn.
4. Chứng minh trên cluster thật: một Pod không có label nào của dự án vẫn gọi được `api` — Flannel
   không thực thi (nợ lab #4).
5. Ghi `TD-09` vào `docs/tech-debt.md`.
6. ⭐ Chứng minh vì sao B2 chọn `labels` thay `commonLabels`: render policy DNS qua một kustomization
   thử nghiệm dùng `commonLabels` và chỉ ra selector bị chèn label.

**Ràng buộc:** không cài CNI khác; không dùng `ipBlock` cho Pod. Ma trận là hợp đồng — không mở thêm
luồng "cho chắc".

**Đọc lại:**

- Bài [84](../84-network-policies-vi.md): [*Điều kiện tiên quyết*](../84-network-policies-vi.md#điều-kiện-tiên-quyết-prerequisites),
  [*Hai kiểu cô lập pod*](../84-network-policies-vi.md#hai-kiểu-cô-lập-pod-the-two-sorts-of-pod-isolation),
  [*Tài nguyên NetworkPolicy*](../84-network-policies-vi.md#networkpolicy-resource),
  [*Hành vi của các bộ chọn to và from*](../84-network-policies-vi.md#hành-vi-của-các-bộ-chọn-to-và-from-behavior-of-to-and-from-selectors),
  [*Các chính sách mặc định*](../84-network-policies-vi.md#các-chính-sách-mặc-định-default-policies),
  [*Nhắm đến một Namespace theo tên*](../84-network-policies-vi.md#nhắm-đến-một-namespace-theo-tên-targeting-a-namespace-by-its-name).
- Bài [81](../81-services-networking-vi.md#the-kubernetes-network-model) — NetworkPolicy do hiện thực
  mạng Pod thực thi.
- Bài [322](../322-kustomization-vi.md#danh-sách-tính-năng-của-kustomize-kustomize-feature-list) —
  `commonLabels` so với `labels` (phần ⭐).

**Bàn giao:** `k8s/app/network/*.yaml`; `k8s/app/kustomization.yaml`; `docs/tech-debt.md`;
`docs/adr/NNNN-networkpolicy.md`; `~/lab-evidence/5a/b8-*`.

**Nghiệm thu:**

- `PASS:` từng luồng của ma trận được phép, từng luồng sai bị chặn — theo bộ mô phỏng.
- `PASS:` có policy chặn mặc định cả hai chiều cho mọi Pod.
- `PASS:` Pod lạ trong `bookstore` vẫn gọi được `api`: policy chưa có hiệu lực trên Flannel.
- `PASS:` hệ thống vẫn phục vụ `/api/books` qua NodePort; `docs/tech-debt.md` có `TD-09`.
- `PASS:` (⭐) với `commonLabels`, selector chọn CoreDNS bị chèn `part-of`; với `labels` thì không.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-networkpolicy.md`:

1. Luồng tới CoreDNS: viết `namespaceSelector` và `podSelector` trong **một** phần tử hay **hai** phần
   tử khác nhau ở chỗ nào? Bản sai mở ra những gì?
2. Vì sao bắt buộc có policy cho DNS khi chặn chiều đi? Probe của kubelet có cần policy không?
3. Nếu hôm nay policy có hiệu lực, trình duyệt gọi `http://<IP worker>:30080/` có còn vào được `web`
   không? Đọc đoạn về `ipBlock` và việc ghi đè IP nguồn của bài 84. Điều đó giải thích quyết định nào
   của Lab 5b?
4. Policy không có tác dụng, vậy vì sao vẫn đưa vào repo bây giờ? Lab 5b phải làm gì **trước** khi đổi
   CNI để không bị bất ngờ?

<details>
<summary>Gợi ý</summary>

- Bốn công thức mặc định của bài 84; `podSelector: {}` chọn mọi Pod trong namespace.
- Label `kubernetes.io/metadata.name` có sẵn trên mọi namespace; Pod CoreDNS mang `k8s-app: kube-dns`.
- Nhiều thành phần cùng được phép vào `db`: `matchExpressions` với toán tử `In`.
- `kubectl -n bookstore describe networkpolicy` cho thấy Kubernetes diễn giải policy thế nào.
- ⭐ Kustomization thử nghiệm ở `~/lab-work/5a/thu-commonlabels/` tham chiếu file policy DNS của bạn
  bằng đường dẫn tương đối, render với `--load-restrictor=LoadRestrictionsNone`; đổi YAML sang JSON
  bằng `kubectl create --dry-run=client -o json -f -`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau `./scripts/deploy.sh`</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a
cd ~/bookstore
kubectl -n "$NS" describe networkpolicy > "$EV/b8-describe.txt"
kubectl get namespace -o json > "$EV/b8-ns.json"
kubectl -n "$NS" get networkpolicy -o json > "$EV/b8-np.json"

python3 - "$EV/b8-np.json" "$EV/b8-ns.json" <<'PY' | tee "$EV/b8-mo-phong.txt"
import json, sys
pols = json.load(open(sys.argv[1]))["items"]
NSL = {n["metadata"]["name"]: n["metadata"].get("labels", {}) for n in json.load(open(sys.argv[2]))["items"]}
def nsl(ns): return NSL.get(ns, {"kubernetes.io/metadata.name": ns})
PORTS = {"web": {"http": 8080}, "api": {"http": 3000, "admin": 3001}, "db": {"postgres": 5432},
         "agent": {"http": 9100}, "dns": {"dns": 53, "dns-tcp": 53}}
def match(sel, labels):
    for k, v in (sel.get("matchLabels") or {}).items():
        if labels.get(k) != v: return False
    for e in sel.get("matchExpressions") or []:
        k, op, vals = e["key"], e["operator"], e.get("values") or []
        if (op == "In" and labels.get(k) not in vals) or (op == "NotIn" and labels.get(k) in vals) or \
           (op == "Exists" and k not in labels) or (op == "DoesNotExist" and k in labels): return False
    return True
def peer(p, ns, labels):
    nsel, psel = p.get("namespaceSelector"), p.get("podSelector")
    if nsel is None and psel is None: return False                # ipBlock: không dùng cho Pod
    if nsel is None and ns != "bookstore": return False
    if nsel is not None and not match(nsel, nsl(ns)): return False
    return psel is None or match(psel, labels)
def port_ok(ports, comp, port, proto):
    if not ports: return True
    for p in ports:
        if p.get("protocol", "TCP") != proto: continue
        v = p.get("port")
        if v is None: return True
        if isinstance(v, str): v = PORTS.get(comp, {}).get(v)
        if v is not None and v <= port <= p.get("endPort", v): return True
    return False
def types(p):
    return p["spec"].get("policyTypes") or (["Ingress"] + (["Egress"] if p["spec"].get("egress") else []))
def allowed(src, dst, port, proto):
    (sns, sl), (dns_, dl, comp) = src, dst
    if sns == "bookstore":
        eg = [p for p in pols if "Egress" in types(p) and match(p["spec"].get("podSelector") or {}, sl)]
        if eg and not any(port_ok(r.get("ports"), comp, port, proto) and
                          (not r.get("to") or any(peer(x, dns_, dl) for x in r["to"]))
                          for p in eg for r in p["spec"].get("egress") or []): return False
    if dns_ == "bookstore":
        ing = [p for p in pols if "Ingress" in types(p) and match(p["spec"].get("podSelector") or {}, dl)]
        if ing and not any(port_ok(r.get("ports"), comp, port, proto) and
                           (not r.get("from") or any(peer(x, sns, sl) for x in r["from"]))
                           for p in ing for r in p["spec"].get("ingress") or []): return False
    return True
def P(c): return ("bookstore", {"app.kubernetes.io/part-of": "bookstore", "app.kubernetes.io/component": c})
def D(c): return ("bookstore", P(c)[1], c)
DNS = ("kube-system", {"k8s-app": "kube-dns"}, "dns")
cases = [(True, ("traefik", {}), D("web"), 8080, "TCP", "traefik -> web 8080"),
         (True, P("web"), D("api"), 3000, "TCP", "web -> api 3000"),
         (True, P("web"), D("agent"), 9100, "TCP", "web -> node-agent 9100"),
         (True, P("api"), D("db"), 5432, "TCP", "api -> db 5432"),
         (True, P("migrate"), D("db"), 5432, "TCP", "db-migrate -> db 5432"),
         (True, P("backup"), D("db"), 5432, "TCP", "db-backup -> db 5432"),
         (False, P("web"), D("db"), 5432, "TCP", "web -> db 5432"),
         (False, P("agent"), D("api"), 3000, "TCP", "node-agent -> api 3000"),
         (False, P("api"), D("web"), 8080, "TCP", "api -> web 8080"),
         (False, P("db"), D("api"), 3000, "TCP", "db -> api 3000"),
         (False, P("web"), D("api"), 3001, "TCP", "web -> api admin 3001"),
         (False, ("bookstore", {"lab": "la"}), D("api"), 3000, "TCP", "Pod la trong bookstore -> api"),
         (False, ("default", {}), D("api"), 3000, "TCP", "Pod namespace default -> api"),
         (False, ("traefik", {}), D("api"), 3000, "TCP", "traefik -> api 3000"),
         (False, ("traefik", {}), D("db"), 5432, "TCP", "traefik -> db 5432"),
         (False, ("default", {}), D("web"), 8080, "TCP", "Pod namespace default -> web 8080"),
         (False, P("web"), ("kube-system", {"k8s-app": "khac"}, "dns"), 53, "UDP", "web -> Pod khac o kube-system 53")]
for c in ["web", "api", "db", "migrate", "backup", "agent"]:
    for proto in ["UDP", "TCP"]:
        cases.append((True, P(c), DNS, 53, proto, f"{c} -> CoreDNS 53/{proto}"))
bad = 0
for want, s, d, port, proto, name in cases:
    got = allowed(s, d, port, proto)
    print(("PASS: " if got == want else "FAIL: ") + name + (" duoc phep" if got else " bi chan"))
    bad += got != want
deny = any(not p["spec"].get("podSelector") and set(types(p)) == {"Ingress", "Egress"}
           and not p["spec"].get("ingress") and not p["spec"].get("egress") for p in pols)
print("PASS: co policy chan mac dinh hai chieu cho moi Pod" if deny else "FAIL: thieu policy chan mac dinh hai chieu")
print("PASS: ma tran 3.5 dung theo mo phong" if bad == 0 and deny else "FAIL: ma tran chua dung")
PY

kubectl -n "$NS" run np-la --rm -i --restart=Never --image=busybox:1.37 --labels=lab=np-la \
  -- wget -qO- -T 5 http://api:3000/books > "$EV/b8-flannel.txt" 2>&1
grep -q '"title"' "$EV/b8-flannel.txt" \
  && echo 'PASS: NO LAB #4 — tren Flannel, luong ma tran cam van thanh cong: policy chua co hieu luc'
W1="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
T="$(curl -s -m 5 "http://$W1:30080/api/books" | grep -o '"title"' | wc -l)"
[ "$T" -ge 8 ] && echo "PASS: he thong van phuc vu qua NodePort sau khi co policy ($T cuon)"
grep -q 'TD-09' docs/tech-debt.md && echo 'PASS: tech-debt.md co TD-09'
```

⭐ `labels` so với `commonLabels` — `F` là file policy DNS của bạn:

```bash
EV=~/lab-evidence/5a; F=allow-dns.yaml
mkdir -p ~/lab-work/5a/thu-commonlabels ~/lab-work/5a/thu-labels
printf 'resources:\n- ../../../bookstore/k8s/app/network/%s\ncommonLabels:\n  app.kubernetes.io/part-of: bookstore\n' "$F" \
  > ~/lab-work/5a/thu-commonlabels/kustomization.yaml
printf 'resources:\n- ../../../bookstore/k8s/app/network/%s\nlabels:\n- pairs:\n    app.kubernetes.io/part-of: bookstore\n' "$F" \
  > ~/lab-work/5a/thu-labels/kustomization.yaml
peer_labels() {
  kubectl kustomize --load-restrictor=LoadRestrictionsNone "$1" 2>/dev/null \
    | kubectl create --dry-run=client -o json -f - \
    | python3 -c 'import json,sys; p=json.load(sys.stdin); print([t.get("podSelector",{}).get("matchLabels",{}) for r in p["spec"]["egress"] for t in r["to"]])'
}
peer_labels ~/lab-work/5a/thu-commonlabels | tee "$EV/b8-commonlabels.txt"
peer_labels ~/lab-work/5a/thu-labels | tee "$EV/b8-labels.txt"
{ grep -q 'part-of' "$EV/b8-commonlabels.txt" && ! grep -q 'part-of' "$EV/b8-labels.txt"; } \
  && echo 'PASS: commonLabels chen part-of vao selector chon CoreDNS — labels thi khong'
rm -rf ~/lab-work/5a/thu-commonlabels ~/lab-work/5a/thu-labels
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# k8s/app/network/default-deny.yaml — mặc định chặn hai chiều cho mọi Pod trong bookstore
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: default-deny-all, namespace: bookstore}
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# k8s/app/network/allow-dns.yaml — chặn chiều đi là chặn luôn DNS (bài 84)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: allow-dns-egress, namespace: bookstore}
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - to:
    - namespaceSelector:                 # MỘT phần tử chứa cả hai bộ chọn: phép giao
        matchLabels: {kubernetes.io/metadata.name: kube-system}
      podSelector:
        matchLabels: {k8s-app: kube-dns}
    ports: [{protocol: UDP, port: 53}, {protocol: TCP, port: 53}]
---
# k8s/app/network/web.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: web, namespace: bookstore}
spec:
  podSelector:
    matchLabels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: web}
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels: {kubernetes.io/metadata.name: traefik}    # có ở Lab 5b
    ports: [{protocol: TCP, port: 8080}]
  egress:
  - to:
    - podSelector:
        matchLabels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: api}
    ports: [{protocol: TCP, port: 3000}]
  - to:
    - podSelector:
        matchLabels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: agent}
    ports: [{protocol: TCP, port: 9100}]
---
# k8s/app/network/api.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: api, namespace: bookstore}
spec:
  podSelector:
    matchLabels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: api}
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: web}
    ports: [{protocol: TCP, port: 3000}]
  egress:
  - to:
    - podSelector:
        matchLabels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: db}
    ports: [{protocol: TCP, port: 5432}]
---
# k8s/app/network/db.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: db, namespace: bookstore}
spec:
  podSelector:
    matchLabels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: db}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {app.kubernetes.io/part-of: bookstore}
        matchExpressions:
        - {key: app.kubernetes.io/component, operator: In, values: [api, migrate, backup]}
    ports: [{protocol: TCP, port: 5432}]
---
# k8s/app/network/db-jobs.yaml — chiều đi của db-migrate và db-backup
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: db-jobs, namespace: bookstore}
spec:
  podSelector:
    matchLabels: {app.kubernetes.io/part-of: bookstore}
    matchExpressions:
    - {key: app.kubernetes.io/component, operator: In, values: [migrate, backup]}
  policyTypes: [Egress]
  egress:
  - to:
    - podSelector:
        matchLabels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: db}
    ports: [{protocol: TCP, port: 5432}]
---
# k8s/app/network/node-agent.yaml — Lab 11a chuyển cùng agent sang bookstore-system
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: node-agent, namespace: bookstore}
spec:
  podSelector:
    matchLabels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: agent}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: web}
    ports: [{protocol: TCP, port: 9100}]
```

Mỗi khối là một file. Thêm bảy dòng `network/<file>.yaml` vào `resources`, rồi `./scripts/deploy.sh`.
Dòng thêm vào `docs/tech-debt.md`:

```markdown
| TD-09 | NetworkPolicy theo ma trận 3.5 đã có trong repo nhưng Flannel không thực thi: mọi Pod gọi được mọi Pod | 5a | 5b | mở | ~/lab-evidence/5a/b8-flannel.txt |
```

Ý chính cho ADR:

1. Một phần tử chứa cả hai bộ chọn là phép **giao**: Pod `k8s-app=kube-dns` **trong** `kube-system`.
   Tách thành hai phần tử là phép **hợp**: mọi Pod của `kube-system` **hoặc** Pod mang `k8s-app=kube-dns`
   trong chính `bookstore` — rộng hơn nhiều, sai một dấu gạch đầu dòng (bài 84).
2. Pod bị cô lập chiều đi chỉ được đi theo các rule `egress`; không có rule cho cổng 53 là tên nào cũng
   không phân giải (bài 84, ghi chú *Thận trọng*). Probe không cần: lưu lượng từ chính node chạy Pod
   luôn được phép.
3. Bài 84: khi IP nguồn bị ghi đè, việc NetworkPolicy thấy IP gốc hay IP của node là không xác định.
   Lưu lượng NodePort đi vào node khác rồi được chuyển sang Pod `web` thường mang IP của node đó —
   không nằm trong ma trận, nên bị chặn; chỉ lưu lượng từ chính node chạy Pod chắc chắn được qua.
   Vì vậy Lab 5b đưa người dùng vào qua Traefik — một Pod có trong ma trận — và trả `web` về ClusterIP.
4. Repo là nguồn sự thật; ma trận được review cùng code và có sẵn ngay khi CNI đổi. Lab 5b phải
   chứng minh trước rằng các policy hiện có không chặn gì trên Flannel, rồi mới đổi CNI và kiểm lại
   từng luồng — vì sau khi đổi, policy sai làm hỏng hệ thống ngay lập tức. ⭐: `commonLabels` thêm
   `part-of` vào selector chọn CoreDNS; Pod CoreDNS không mang label đó, nên ở 5b mọi Pod mất DNS.

</details>

## B9. [BS-5A-09] Điều tra: Service ra ngoài cluster — `LoadBalancer`, `externalTrafficPolicy`, `ExternalName`

**Loại:** Điều tra.

**Bối cảnh:** đội vận hành hỏi ba câu trước khi chốt cách mở hệ thống ra ngoài: sao không dùng
`type: LoadBalancer` như các hướng dẫn trên cloud; có nên giữ IP nguồn của khách hàng; và khi DB
chuyển ra dịch vụ ngoài cluster, ứng dụng có phải sửa không.

**Yêu cầu:** ba Service thử nghiệm ở `~/lab-work/5a/duong-vao.yaml`, mang label `lab: duong-vao`:
`web-lb` kiểu `LoadBalancer` với chính sách lưu lượng ngoài mặc định, `web-lb-local` kiểu
`LoadBalancer` với chính sách `Local` — cả hai chọn Pod `web` thật; `api-ngoai` kiểu `ExternalName`
trỏ tới FQDN của Service `api`. Chứng minh:

1. `web-lb` được cấp ClusterIP và NodePort nhưng `EXTERNAL-IP` treo `<pending>`, `status.loadBalancer`
   rỗng, không có finalizer dọn load balancer nào; cổng NodePort của nó vẫn phục vụ.
2. Chỉ `web-lb-local` được cấp `healthCheckNodePort`.
3. Với `web-lb-local`: gọi từ master tới IP worker có Pod `web` thì được; gọi từ `lab-k8s-worker1` tới
   IP của master — node không có Pod `web` — thì không, trong khi `web-lb` vẫn được.
4. `api-ngoai` không có ClusterIP, không có EndpointSlice, tên của nó chuyển hướng ở tầng DNS về
   ClusterIP của `api`, và gọi `/books` qua tên đó vẫn ra sách.
5. Xóa cả ba Service.

**Ràng buộc:** không cài load balancer nào. Không đặt `nodePort` cố định cho Service thử nghiệm.

**Đọc lại:**

- Bài [364](../364-create-external-load-balancer-vi.md):
  [*Giữ nguyên địa chỉ IP nguồn của client*](../364-create-external-load-balancer-vi.md#preserving-the-client-source-ip),
  [*Thu dọn các bộ cân bằng tải*](../364-create-external-load-balancer-vi.md#thu-dọn-garbage-collect-các-bộ-cân-bằng-tải-garbage-collecting-load-balancers),
  [*Các nhà cung cấp bộ cân bằng tải bên ngoài*](../364-create-external-load-balancer-vi.md#các-nhà-cung-cấp-bộ-cân-bằng-tải-bên-ngoài-external-load-balancer-providers).
- Bài [82](../82-service-vi.md): [`type: LoadBalancer`](../82-service-vi.md#loadbalancer),
  [`type: ExternalName`](../82-service-vi.md#externalname), [*Loại Service*](../82-service-vi.md#publishing-services-service-types)
  — thiết kế lồng nhau.

**Bàn giao:** `~/lab-work/5a/duong-vao.yaml`; `docs/adr/NNNN-duong-vao-tu-ngoai.md`;
`~/lab-evidence/5a/b9-*`.

**Nghiệm thu:**

- `PASS:` `web-lb` có ClusterIP, NodePort, `EXTERNAL-IP` `<pending>`, không finalizer; NodePort phục vụ.
- `PASS:` chỉ `web-lb-local` có `healthCheckNodePort`.
- `PASS:` `web-lb-local` phục vụ ở worker có Pod `web`, từ chối ở master; `web-lb` phục vụ ở master.
- `PASS:` `api-ngoai` không ClusterIP, không EndpointSlice, phân giải về ClusterIP của `api`, `/books`
  qua tên đó ra sách.
- `PASS:` không còn Service `lab=duong-vao`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-duong-vao-tu-ngoai.md`:

1. Với `web-lb`, Kubernetes đã làm xong những gì và còn thiếu ai? Việc không có finalizer nói gì về
   thành phần còn thiếu? Service đó có dùng được từ bên trong cluster không?
2. `Local` giữ được gì, mất gì? Vì sao Service `web` của dự án **không** dùng `Local` khi người dùng
   gõ IP của bất kỳ node nào? `healthCheckNodePort` sinh ra để ai dùng?
3. Một ngày `db` chuyển sang dịch vụ PostgreSQL bên ngoài. `ExternalName` giúp gì để `api` không phải
   sửa, và bài 82 cảnh báo gì khi giao thức dùng hostname — áp vào PostgreSQL có TLS kiểm tên máy chủ?

<details>
<summary>Gợi ý</summary>

- `kubectl get svc` hiện `<pending>` ở cột `EXTERNAL-IP`; trường thật là `.status.loadBalancer.ingress`.
- Node nào có Pod `web`:
  `kubectl -n bookstore get pod -l 'app.kubernetes.io/component=web,!lab' -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}'`.
- Gọi từ worker: `ssh lab-k8s-worker1 "curl -s -o /dev/null -m 3 -w '%{http_code}' http://<IP master>:<cổng>/"`.
  Gọi từ chính node tới IP của chính nó không đo được `Local` — chọn một node khác làm nguồn.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự viết `duong-vao.yaml`</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a; W=~/lab-work/5a
nip() { kubectl get node "$1" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'; }
svc() { kubectl -n "$NS" get service "$1" -o jsonpath="$2" 2>/dev/null; }
WEB="$(kubectl -n "$NS" get pod -l 'app.kubernetes.io/component=web,!lab' -o jsonpath='{.items[0].metadata.name}')"
API_IP="$(svc api '{.spec.clusterIP}')"

kubectl apply -f "$W/duong-vao.yaml"
sleep 10
kubectl -n "$NS" get svc -l lab=duong-vao -o wide | tee "$EV/b9-svc.txt"
NP="$(svc web-lb '{.spec.ports[0].nodePort}')"; NPL="$(svc web-lb-local '{.spec.ports[0].nodePort}')"
{ [ -n "$(svc web-lb '{.spec.clusterIP}')" ] && [ -n "$NP" ] && [ -z "$(svc web-lb '{.status.loadBalancer.ingress}')" ] && \
  grep -E '^web-lb ' "$EV/b9-svc.txt" | grep -q '<pending>' && [ -z "$(svc web-lb '{.metadata.finalizers}')" ]; } \
  && echo 'PASS: web-lb co ClusterIP, NodePort; EXTERNAL-IP <pending>; khong finalizer — khong ai lap trinh LB ngoai'
[ "$(curl -s -o /dev/null -m 5 -w '%{http_code}' "http://$(nip lab-k8s-worker1):$NP/")" = '200' ] \
  && echo 'PASS: tang NodePort ben duoi LoadBalancer van phuc vu'
{ [ -n "$(svc web-lb-local '{.spec.healthCheckNodePort}')" ] && [ -z "$(svc web-lb '{.spec.healthCheckNodePort}')" ]; } \
  && echo 'PASS: chi Service LoadBalancer + Local duoc cap healthCheckNodePort'

WN="$(kubectl -n "$NS" get pod -l 'app.kubernetes.io/component=web,!lab' -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u)"
BAD=0; : > "$EV/b9-etp.txt"
for N in $WN; do
  C="$(curl -s -o /dev/null -m 3 -w '%{http_code}' "http://$(nip "$N"):$NPL/")"
  echo "master -> $N:$NPL (co Pod web) = $C" | tee -a "$EV/b9-etp.txt"; [ "$C" = '200' ] || BAD=1
done
M="$(nip lab-k8s-master)"
CL="$(ssh lab-k8s-worker1 "curl -s -o /dev/null -m 3 -w '%{http_code}' http://$M:$NPL/")"
CC="$(ssh lab-k8s-worker1 "curl -s -o /dev/null -m 3 -w '%{http_code}' http://$M:$NP/")"
echo "worker1 -> master (khong co Pod web): Local=$CL Cluster=$CC" | tee -a "$EV/b9-etp.txt"
{ [ "$BAD" -eq 0 ] && [ "$CL" != '200' ] && [ "$CC" = '200' ]; } \
  && echo 'PASS: Local chi phuc vu o node co Pod web; Cluster phuc vu o moi node'

{ [ -z "$(svc api-ngoai '{.spec.clusterIP}')" ] && \
  [ "$(kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=api-ngoai -o name | grep -c .)" -eq 0 ]; } \
  && echo 'PASS: ExternalName khong co ClusterIP, khong co EndpointSlice'
kubectl -n "$NS" exec "$WEB" -c status-writer -- nslookup api-ngoai > "$EV/b9-extname.txt" 2>&1
grep -q "$API_IP" "$EV/b9-extname.txt" && echo 'PASS: api-ngoai chuyen huong o tang DNS toi ClusterIP cua api'
T="$(kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- http://api-ngoai:3000/books | grep -o '"title"' | wc -l)"
[ "$T" -ge 5 ] && echo "PASS: goi /books qua ten ExternalName van ra $T cuon"

kubectl delete -f "$W/duong-vao.yaml" --wait=true >/dev/null
[ "$(kubectl -n "$NS" get svc -l lab=duong-vao --no-headers 2>/dev/null | wc -l)" -eq 0 ] && echo 'PASS: da xoa ba Service thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/5a/duong-vao.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  namespace: bookstore
  labels: {lab: duong-vao}
spec:
  type: LoadBalancer
  selector: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: web}
  ports:
  - {name: http, port: 80, targetPort: http}
---
apiVersion: v1
kind: Service
metadata:
  name: web-lb-local
  namespace: bookstore
  labels: {lab: duong-vao}
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector: {app.kubernetes.io/part-of: bookstore, app.kubernetes.io/component: web}
  ports:
  - {name: http, port: 80, targetPort: http}
---
apiVersion: v1
kind: Service
metadata:
  name: api-ngoai
  namespace: bookstore
  labels: {lab: duong-vao}
spec:
  type: ExternalName
  externalName: api.bookstore.svc.cluster.local
```

Ý chính cho ADR:

1. Kubernetes làm xong phần trong cluster: ClusterIP, NodePort (thiết kế lồng nhau, bài 82). Thiếu
   một bộ cân bằng tải bên ngoài cùng controller lập trình nó và điền IP vào Service (bài 364). Không
   finalizer `service.kubernetes.io/load-balancer-cleanup` nghĩa là không có service controller nào
   nhận Service này. Bên trong cluster nó dùng được như một ClusterIP.
2. `Local` giữ IP nguồn của client và bỏ chặng thứ hai, đổi lại node không có endpoint cục bộ không
   phục vụ — đo được ở master. Người dùng của dự án gõ IP node bất kỳ, nên `Local` làm hỏng một phần
   các node. `healthCheckNodePort` dành cho bộ cân bằng tải bên ngoài, để nó bỏ qua node không có
   backend — trên bare metal không ai dùng.
3. Service `db` đổi sang `ExternalName` trỏ tên máy của dịch vụ ngoài: `api` vẫn gọi tên cũ, chuyển
   hướng xảy ra ở DNS, không proxy (bài 82). Cảnh báo của bài: tên client dùng khác tên máy chủ thật —
   TLS kiểm tên máy chủ (`verify-full` của PostgreSQL) sẽ thấy certificate không khớp `db-0.db`; HTTP
   gửi `Host:` mà server không nhận.

</details>

## B10. [BS-5A-10] Điều tra: `port-forward` tới `db` và ⭐ Service không selector

**Loại:** Điều tra — phần ⭐ là tùy chọn.

**Bối cảnh:** người trực DB cần một đường gỡ lỗi từ máy quản trị vào `db-0` mà không mở cổng nào ra
mạng. Đội cũng cần biết đường đó đứng ở đâu so với Service và — từ Lab 5b — so với NetworkPolicy.

**Yêu cầu:**

1. Chuyển tiếp một cổng cục bộ trên master tới cổng PostgreSQL qua Service `db`. Chứng minh đầu bên kia
   là **PostgreSQL thật** mà không cài client nào: gửi gói `SSLRequest` của giao thức PostgreSQL và
   nhận đúng một byte trả lời `N` hoặc `S`.
2. Chứng minh đường chuyển tiếp chỉ nghe trên loopback của master, đi tới đúng cổng 5432 của Pod, và
   dừng ngay khi tiến trình `kubectl` dừng.
3. Viết `docs/runbook-port-forward-db.md`: lệnh, giới hạn, rủi ro, ai được phép dùng.
4. ⭐ Service không selector `db-ngoai` (cổng 5432) cùng một EndpointSlice tự viết trỏ tới IP hiện tại
   của `db-0`, đánh dấu người quản lý là con người. Chứng minh: không có slice nào tự sinh trước khi
   bạn viết; sau đó `pg_isready` qua tên `db-ngoai` nhận kết nối; `port-forward` tới `db-ngoai` bị từ
   chối. Xóa cả hai.

**Ràng buộc:** không cài `psql` hay client nào lên master. Không mở `port-forward` trên địa chỉ khác
loopback.

**Đọc lại:**

- Bài [366](../366-port-forward-vi.md):
  [*Chuyển tiếp một port cục bộ tới một port trên Pod*](../366-port-forward-vi.md#forward-a-local-port-to-a-port-on-the-pod),
  [*Thảo luận*](../366-port-forward-vi.md#thảo-luận-discussion),
  [*Cân nhắc về ủy quyền và bảo mật*](../366-port-forward-vi.md#cân-nhắc-về-ủy-quyền-và-bảo-mật-authorization-and-security-considerations).
- Bài [82](../82-service-vi.md): [*Service không có selector*](../82-service-vi.md#services-without-selectors),
  [*Truy cập một Service không có selector*](../82-service-vi.md#service-no-selector-access) (⭐).
- Bài [83](../83-endpoint-slices-vi.md#quản-lý-management) — nhãn `managed-by` (⭐).

**Bàn giao:** `docs/runbook-port-forward-db.md`; ⭐ `~/lab-work/5a/db-ngoai.yaml`;
`~/lab-evidence/5a/b10-*`.

**Nghiệm thu:**

- `PASS:` `127.0.0.1:15432` trên master trả lời `SSLRequest` như PostgreSQL.
- `PASS:` cổng chỉ nghe trên loopback; log ghi đích 5432 và có kết nối đi qua; dừng `kubectl` thì cổng
  đóng.
- `PASS:` có `docs/runbook-port-forward-db.md`.
- `PASS:` (⭐) `db-ngoai` không có slice tự sinh; với slice tự viết, `pg_isready -h db-ngoai` nhận kết
  nối; `port-forward` tới `db-ngoai` thất bại; đã xóa.

**Câu hỏi thiết kế** — trả lời trong runbook:

1. `port-forward service/db` có đi qua kube-proxy và cân bằng tải không? Lưu lượng thật đi qua những
   thành phần nào để tới `db-0`?
2. Bài 366 nói `port-forward` có thể vượt qua các biện pháp kiểm soát ở tầng mạng. Khi Lab 5b bật
   NetworkPolicy chặn mọi luồng vào `db` trừ `api` và Job, đường này còn tới `db-0` không? Vậy ai được
   phép dùng nó, và lab nào siết quyền đó?
3. ⭐ EndpointSlice tự viết trỏ IP của `db-0` lặp lại đúng lỗi nào mà sprint này vừa trả? Khi nào Service
   không selector mới là lựa chọn đúng (bài 82)? Vì sao API server từ chối `port-forward` tới nó?

<details>
<summary>Gợi ý</summary>

- `kubectl -n bookstore port-forward svc/db 15432:5432` không trả lại dấu nhắc — chạy nền bằng `&`,
  ghi log ra file, nhớ PID để `kill`.
- `SSLRequest` là 8 byte: độ dài 8 và mã `80877103`, cả hai số nguyên 4 byte big-endian. `python3` có
  `socket` và `struct.pack("!ii", 8, 80877103)`.
- Cổng đang nghe trên master: `ss -ltn`.
- ⭐ `pg_isready` có trong image `postgres`:
  `kubectl -n bookstore run pg-thu --rm -i --restart=Never --image=postgres:17.11 -- pg_isready -h db-ngoai -p 5432`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự làm xong</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a; W=~/lab-work/5a
cd ~/bookstore
kubectl -n "$NS" port-forward svc/db 15432:5432 > "$EV/b10-pf.log" 2>&1 &
PF=$!
for i in $(seq 1 20); do grep -q 'Forwarding from 127.0.0.1:15432' "$EV/b10-pf.log" && break; sleep 1; done
R="$(python3 - <<'PY'
import socket, struct
s = socket.create_connection(("127.0.0.1", 15432), timeout=5)
s.sendall(struct.pack("!ii", 8, 80877103))
print(s.recv(1).decode())
PY
)"
case "$R" in N|S) echo "PASS: 127.0.0.1:15432 tra loi SSLRequest bang '$R' — dau ben kia la PostgreSQL" ;; esac
{ ss -ltn | grep -q '127.0.0.1:15432' && ! ss -ltn | grep -Eq '(0\.0\.0\.0|\*):15432'; } \
  && echo 'PASS: port-forward chi nghe tren loopback cua master'
{ grep -q -- '-> 5432' "$EV/b10-pf.log" && grep -q 'Handling connection' "$EV/b10-pf.log"; } \
  && echo 'PASS: duong chuyen tiep toi cong 5432 cua Pod, co ket noi di qua'
kill "$PF"; sleep 2
ss -ltn | grep -q '127.0.0.1:15432' && echo 'FAIL: cong van mo' || echo 'PASS: dung kubectl thi cong dong'
test -s docs/runbook-port-forward-db.md && echo 'PASS: co runbook port-forward'

# ⭐ Service không selector
if [ -f "$W/db-ngoai.yaml" ]; then
  kubectl apply -f "$W/db-ngoai.yaml"
  sleep 5
  [ "$(kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=db-ngoai \
       -o jsonpath='{range .items[*]}{.metadata.labels.endpointslice\.kubernetes\.io/managed-by}{"\n"}{end}')" = 'staff' ] \
    && echo 'PASS: db-ngoai chi co dung slice tu viet (managed-by=staff), khong co slice tu sinh'
  kubectl -n "$NS" run pg-thu --rm -i --restart=Never --image=postgres:17.11 -- pg_isready -h db-ngoai -p 5432 \
    > "$EV/b10-db-ngoai.txt" 2>&1
  grep -q 'accepting connections' "$EV/b10-db-ngoai.txt" && echo 'PASS: pg_isready qua ten db-ngoai nhan ket noi'
  timeout 10 kubectl -n "$NS" port-forward svc/db-ngoai 15433:5432 > "$EV/b10-pf-ngoai.txt" 2>&1 \
    && echo 'FAIL: port-forward toi Service khong selector thanh cong' \
    || echo 'PASS: port-forward toi Service khong selector bi tu choi'
  kubectl delete -f "$W/db-ngoai.yaml" --wait=true >/dev/null && echo 'PASS: da xoa db-ngoai'
fi
```

Phần ⭐ không có slice tự sinh nào để kiểm "trước khi viết": Service không selector thì control plane
không tạo slice (bài 82) — script chỉ thấy đúng một slice `managed-by=staff` của bạn.

</details>

<details>
<summary>Lời giải tham khảo</summary>

```bash
kubectl -n bookstore port-forward svc/db 15432:5432 &
python3 -c 'import socket,struct; s=socket.create_connection(("127.0.0.1",15432)); s.sendall(struct.pack("!ii",8,80877103)); print(s.recv(1))'
kill %1
```

⭐ `db-ngoai.yaml` — điền IP hiện tại của `db-0` trước khi apply:

```yaml
# ~/lab-work/5a/db-ngoai.yaml — Service không selector + EndpointSlice tự viết; xóa khi xong
apiVersion: v1
kind: Service
metadata:
  name: db-ngoai
  namespace: bookstore
  labels: {lab: db-ngoai}
spec:
  ports:
  - {name: postgres, port: 5432, targetPort: 5432}
---
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: db-ngoai-1
  namespace: bookstore
  labels:
    lab: db-ngoai
    kubernetes.io/service-name: db-ngoai
    endpointslice.kubernetes.io/managed-by: staff
addressType: IPv4
ports:
- {name: postgres, protocol: TCP, port: 5432}
endpoints:
- addresses: ["<IP của db-0>"]      # kubectl -n bookstore get pod db-0 -o jsonpath='{.status.podIP}'
  conditions: {ready: true}
```

Ý chính cho runbook:

1. Không. `port-forward` chỉ dùng tên Service để **chọn một Pod** (bài 366); lưu lượng đi từ `kubectl`
   qua API server tới kubelet trên node của `db-0` rồi vào Pod — không qua kube-proxy, không cân bằng
   tải, chỉ TCP.
2. Bài 366: đường này vượt qua kiểm soát tầng mạng — NetworkPolicy của 5b không chặn được nó. Kiểm soát
   nằm ở quyền: ai `create` được `pods/portforward` là chạm được `db`. Chỉ người trực DB; Lab 9a giới
   hạn quyền đó bằng RBAC.
3. ⭐ Trỏ cứng IP Pod — đúng `TD-03`: `db-0` bị thay là slice trỏ vào khoảng không. Service không selector
   hợp với backend **ngoài** cluster hay đang di chuyển dần (bài 82). API server không proxy tới endpoint
   không ánh xạ về Pod, để khỏi bị dùng làm proxy tới nơi bên gọi không được phép chạm.

</details>

## B11. [BS-5A-11] Điều tra: ClusterIP lấy từ đâu, và định tuyến theo topology

**Loại:** Điều tra.

**Bối cảnh:** một đồng nghiệp đề xuất "cho `api` một ClusterIP cố định cho dễ nhớ" và một người khác
muốn "bật định tuyến theo zone cho rẻ". Trả lời cả hai bằng số đo trên cluster thật.

**Yêu cầu:**

1. Đọc dải Service thật từ manifest của API server (chỉ đọc) và đối chiếu với `kubeadm-config`.
2. Áp công thức của bài 88 vào dải thật: độ lệch băng, băng tĩnh. Chứng minh ClusterIP của `kube-dns`
   là địa chỉ thứ 10 của dải và nằm trong băng tĩnh; ClusterIP của `api`, `web`, `node-agent` nằm trong
   dải và ở băng động.
3. Xin thử một ClusterIP tĩnh trong băng dưới — API server chấp nhận; xin một địa chỉ ngoài dải — bị
   từ chối. Cả hai bằng chế độ dry-run phía server, không để lại object nào.
4. Định tuyến theo topology: tạo một Service thử nghiệm cho `api` bằng pipeline
   `create --dry-run` → `set selector --local` → `create -f -` của bài 320, gắn chế độ topology `Auto`
   bằng `kubectl annotate`. Chứng minh không node nào có label zone và không endpoint nào được gắn
   `hints`. Gỡ annotation, xóa Service.

**Ràng buộc:** không sửa manifest control plane. Không gắn label zone cho node.

**Đọc lại:**

- Bài [88](../88-cluster-ip-allocation-vi.md): [*Làm thế nào để tránh xung đột*](../88-cluster-ip-allocation-vi.md#avoid-ClusterIP-conflict),
  [*Các ví dụ*](../88-cluster-ip-allocation-vi.md#allocation-examples).
- Bài [86](../86-topology-aware-routing-vi.md): [*Bật Topology Aware Routing*](../86-topology-aware-routing-vi.md#bật-topology-aware-routing-enabling-topology-aware-routing),
  [*EndpointSlice controller*](../86-topology-aware-routing-vi.md#implementation-control-plane),
  [*Các cơ chế bảo vệ*](../86-topology-aware-routing-vi.md#các-cơ-chế-bảo-vệ-safeguards),
  [*Các ràng buộc*](../86-topology-aware-routing-vi.md#các-ràng-buộc-constraints).
- Bài [320](../320-imperative-command-vi.md#dùng-lệnh-set-để-sửa-object-trước-khi-tạo-using-set-commands-to-modify-objects-before-creation).
- Bài [82](../82-service-vi.md#type-clusterip), mục *Tự chọn địa chỉ IP*.

**Bàn giao:** `docs/adr/NNNN-clusterip-va-topology.md`; `~/lab-evidence/5a/b11-*`.

**Nghiệm thu:**

- `PASS:` dải trong manifest API server khớp `kubeadm-config`.
- `PASS:` `kube-dns` là địa chỉ thứ 10, nằm trong băng tĩnh; ClusterIP của `api`, `web`, `node-agent`
  nằm trong dải, ở băng động.
- `PASS:` dry-run server chấp nhận ClusterIP tĩnh trong băng dưới, từ chối địa chỉ ngoài dải; không
  Service nào được tạo.
- `PASS:` annotation `topology-mode=Auto` được lưu; không node nào có zone; không endpoint nào có
  `hints`; Service thử nghiệm đã xóa.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-clusterip-va-topology.md`:

1. Có nên cho `api` một ClusterIP tĩnh không? Băng tĩnh có **đặt chỗ** cho bạn không (bài 88)?
2. Bài 86 nêu năm cơ chế bảo vệ. Cơ chế nào chặn trên cluster này? Ngoài ra, controller bỏ qua loại
   node nào khi tính tỷ lệ?
3. Vì sao không bao giờ bật được cả topology `Auto` lẫn `internalTrafficPolicy: Local` cho
   `node-agent` cùng lúc?

<details>
<summary>Gợi ý</summary>

- `sudo grep service-cluster-ip-range /etc/kubernetes/manifests/kube-apiserver.yaml`;
  `kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}'`.
- Đổi IP sang số nguyên trong bash: `IFS=. read -r a b c d <<<"$IP"; echo $(( (a<<24)+(b<<16)+(c<<8)+d ))`.
- Dry-run phía server: `kubectl create --dry-run=server -f <file>`.
- Pipeline của bài 320:
  `kubectl -n bookstore create service clusterip api-topo --tcp=3000:http --dry-run=client -o yaml | kubectl set selector --local -f - 'app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=api' -o yaml | kubectl create -f -`.

</details>

<details>
<summary>Script nghiệm thu — script này chính là cuộc điều tra</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a
ip2int() { local a b c d; IFS=. read -r a b c d <<<"$1"; echo $(( (a<<24)+(b<<16)+(c<<8)+d )); }
int2ip() { local n=$1; echo "$(( (n>>24)&255 )).$(( (n>>16)&255 )).$(( (n>>8)&255 )).$(( n&255 ))"; }

SVC_CIDR="$(sudo awk -F= '/--service-cluster-ip-range=/{print $2}' /etc/kubernetes/manifests/kube-apiserver.yaml | tr -d ' \r')"
KCFG="$(kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | awk -F': *' '/serviceSubnet:/{print $2}' | tr -d ' \r')"
echo "apiserver=$SVC_CIDR kubeadm-config=$KCFG" | tee "$EV/b11-dai.txt"
{ [ -n "$SVC_CIDR" ] && [ "$SVC_CIDR" = "$KCFG" ]; } && echo 'PASS: dai Service trong manifest apiserver khop kubeadm-config'

NET="$(ip2int "${SVC_CIDR%%/*}")"; BITS="${SVC_CIDR##*/}"; SIZE=$(( 1 << (32 - BITS) ))
MASK=$(( (0xFFFFFFFF << (32 - BITS)) & 0xFFFFFFFF ))
OFF=$(( SIZE / 16 )); [ "$OFF" -lt 16 ] && OFF=16; [ "$OFF" -gt 256 ] && OFF=256
SEND=$(( NET + OFF ))
echo "do lech bang=$OFF; bang tinh $(int2ip $((NET + 1))) .. $(int2ip $SEND)" | tee -a "$EV/b11-dai.txt"
in_range() { [ $(( $(ip2int "$1") & MASK )) -eq $(( NET & MASK )) ]; }

DNS_IP="$(kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.clusterIP}')"
{ [ "$DNS_IP" = "$(int2ip $((NET + 10)))" ] && [ "$(ip2int "$DNS_IP")" -le "$SEND" ]; } \
  && echo "PASS: kube-dns ($DNS_IP) la dia chi thu 10, nam trong bang tinh"
BAD=0
for S in api web node-agent; do
  IP="$(kubectl -n "$NS" get svc "$S" -o jsonpath='{.spec.clusterIP}')"
  echo "$S $IP" >> "$EV/b11-dai.txt"
  { in_range "$IP" && [ "$(ip2int "$IP")" -gt "$SEND" ]; } || { echo "FAIL: $S $IP"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: ClusterIP cua api, web, node-agent nam trong dai, o bang dong'

thu() {   # $1 = ten, $2 = clusterIP; dry-run phía server, không tạo object
  printf 'apiVersion: v1\nkind: Service\nmetadata: {name: %s, namespace: bookstore, labels: {lab: cip}}\nspec:\n  clusterIP: %s\n  ports: [{port: 3000}]\n' "$1" "$2" \
    | kubectl create --dry-run=server -f - 2>&1
}
thu cip-tinh "$(int2ip $((NET + 99)))" > "$EV/b11-tinh.txt" && echo "PASS: dry-run server chap nhan ClusterIP tinh $(int2ip $((NET + 99)))"
thu cip-ngoai "$(int2ip $((NET + SIZE + 1)))" > "$EV/b11-ngoai.txt" \
  && echo 'FAIL: chap nhan dia chi ngoai dai' || { cat "$EV/b11-ngoai.txt"; echo 'PASS: dia chi ngoai dai bi API server tu choi'; }
[ "$(kubectl -n "$NS" get svc -l lab=cip --no-headers 2>/dev/null | wc -l)" -eq 0 ] && echo 'PASS: khong Service nao duoc tao'

# Định tuyến theo topology
kubectl -n "$NS" create service clusterip api-topo --tcp=3000:http --dry-run=client -o yaml \
  | kubectl set selector --local -f - 'app.kubernetes.io/part-of=bookstore,app.kubernetes.io/component=api' -o yaml \
  | kubectl create -f -
kubectl -n "$NS" annotate service api-topo service.kubernetes.io/topology-mode=Auto --overwrite
[ "$(kubectl -n "$NS" get svc api-topo -o jsonpath='{.metadata.annotations.service\.kubernetes\.io/topology-mode}')" = 'Auto' ] \
  && echo 'PASS: annotation topology-mode=Auto da duoc luu'
sleep 10
ZONES="$(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.labels.topology\.kubernetes\.io/zone}{"\n"}{end}' | grep -c . || true)"
EPN="$(kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=api-topo -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\n"}{end}' | grep -c .)"
HINTS="$(kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=api-topo -o jsonpath='{range .items[*].endpoints[*]}{.hints}{"\n"}{end}' | grep -c . || true)"
echo "node co zone=$ZONES ; endpoint=$EPN ; endpoint co hints=$HINTS" | tee "$EV/b11-topology.txt"
{ [ "$ZONES" -eq 0 ] && [ "$EPN" -ge 1 ] && [ "$HINTS" -eq 0 ]; } \
  && echo 'PASS: khong node nao co zone — co che bao ve so 3 chan, khong endpoint nao co hints'
kubectl -n "$NS" annotate service api-topo service.kubernetes.io/topology-mode-
kubectl -n "$NS" delete service api-topo && echo 'PASS: da go annotation va xoa Service thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của ADR</summary>

1. Không. Client gọi `api` bằng tên; ClusterIP tĩnh chỉ dành cho Service phải ở địa chỉ biết trước —
   DNS của cluster là ví dụ của bài 88 — hay hệ thống cũ khó cấu hình lại (bài 82). Băng tĩnh **không
   đặt chỗ**: cấp phát động chỉ ưu tiên băng trên, cạn thì lấn xuống; địa chỉ bạn chọn vẫn có thể bị
   chiếm trước.
2. Cơ chế số 3: có node thiếu label `topology.kubernetes.io/zone` thì control plane không đặt hint nào —
   cả ba node đều thiếu. Ngoài ra controller bỏ qua node chưa sẵn sàng và node mang label
   `node-role.kubernetes.io/control-plane` khi tính tỷ lệ (bài 86, *Các ràng buộc*).
3. Bài 86: topology hint không được dùng khi Service đặt `internalTrafficPolicy: Local`. Hai tính năng
   dùng chung được trong một cluster nhưng không trên cùng một Service.

</details>

## B12. [BS-5A-12] Điều tra: headless Service ngoài `db` — xóa StatefulSet đúng cách, Job gọi nhau bằng tên

**Loại:** Điều tra.

**Bối cảnh:** headless Service không chỉ phục vụ `db`. Đội cần biết hai điều trước khi dùng nó ở chỗ
khác: dọn một StatefulSet thì Service quản trị của nó ra sao, và một Job song song có thể cho các Pod
của mình gọi nhau bằng tên mà không hỏi API server — hướng đi cho báo cáo tồn kho song song của Lab 4b
nếu một ngày các phân vùng cần trao đổi kết quả.

**Yêu cầu:**

1. StatefulSet thử nghiệm `sts-thu` hai bản sao cùng Service quản trị headless `sts-thu`, ở
   `~/lab-work/5a/sts-thu.yaml`. Mỗi Pod phục vụ hostname của chính nó. Chứng minh `sts-thu-1.sts-thu`
   phân giải và trả đúng `sts-thu-1`.
2. Xóa StatefulSet theo đúng trình tự của bài 340 — đọc ân hạn của Pod, xóa theo label, chờ hết ân
   hạn. Chứng minh Pod đi theo nhưng Service quản trị **còn lại**; rồi tự xóa Service. Không có claim
   nào để xóa — nợ lab #2 vẫn mở.
3. Indexed Job `kiem-ket-noi` ba Pod song song cùng headless Service `kiem-ket-noi`, ở
   `~/lab-work/5a/kiem-ket-noi.yaml`: mỗi Pod phục vụ hostname của nó và chỉ thoát thành công sau khi
   gọi được cả ba Pod bằng tên `kiem-ket-noi-<i>.kiem-ket-noi`. Job `Complete`; mỗi Pod mang hostname
   và subdomain đúng; log ghi đủ chín lần gọi thành công.
4. Xóa mọi object thử nghiệm.

**Ràng buộc:** không đụng StatefulSet `db`. Pod của Job không hỏi API server.

**Đọc lại:**

- Bài [340](../340-delete-stateful-set-vi.md): [*Xóa một StatefulSet*](../340-delete-stateful-set-vi.md#xóa-một-statefulset-deleting-a-statefulset),
  [*Xóa hoàn toàn một StatefulSet*](../340-delete-stateful-set-vi.md#xóa-hoàn-toàn-một-statefulset-complete-deletion-of-a-statefulset).
- Bài [65](../65-statefulset-vi.md#limitations), *Hạn chế* — xóa StatefulSet không bảo đảm thứ tự tắt.
- Bài [354](../354-job-pod-to-pod-communication-vi.md): [*Khởi chạy một Job với giao tiếp Pod-đến-Pod*](../354-job-pod-to-pod-communication-vi.md#khởi-chạy-một-job-với-giao-tiếp-pod-đến-pod-starting-a-job-with-pod-to-pod-communication),
  [*Ví dụ*](../354-job-pod-to-pod-communication-vi.md#ví-dụ-example).
- Bài [57](../57-pod-hostname-vi.md#hostname-với-các-trường-hostname-và-subdomain-của-pod-hostname-with-pods-hostname-and-subdomain-fields).

**Bàn giao:** hai manifest ở `~/lab-work/5a/`; `docs/adr/NNNN-headless-ngoai-db.md`;
`~/lab-evidence/5a/b12-*`.

**Nghiệm thu:**

- `PASS:` `sts-thu-1.sts-thu` phân giải và trả `sts-thu-1`.
- `PASS:` sau khi xóa theo trình tự bài 340: không còn Pod `sts-thu`, Service `sts-thu` vẫn còn; không
  có PVC nào.
- `PASS:` Job `kiem-ket-noi` `Complete`; hostname/subdomain của ba Pod là `kiem-ket-noi-{0,1,2}.kiem-ket-noi`;
  log có đủ 9 lần gọi thành công.
- `PASS:` không còn object `lab=sts-thu` hay `lab=kiem-ket-noi`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-headless-ngoai-db.md`:

1. Vì sao Service quản trị không bị xóa theo StatefulSet? Với dự án, `teardown.sh` xử lý điều đó thế
   nào, và vì sao nó thu `db` về 0 bản sao trước khi xóa namespace?
2. Ba Pod của Job tính được tên của nhau trước khi chạy nhờ đâu? Quên `subdomain` trong template thì
   Job ra sao? Đặt `dnsPolicy: Default` thì sao?
3. Vì sao script của mỗi Pod phải tiếp tục phục vụ thêm một lúc **sau khi** đã gọi xong cả ba Pod?

<details>
<summary>Gợi ý</summary>

- Pod `busybox` phục vụ hostname: `mkdir -p /www && hostname > /www/index.html && exec httpd -f -p 8080 -h /www`.
- Selector đơn giản nhất cho headless Service của Job là label `job-name` mà Kubernetes tự thêm.
- Trong Pod của Job, `httpd -p 8080 -h /www` không có `-f` thì tự chạy nền; vòng
  `until wget -q -T 2 -O- http://kiem-ket-noi-$i.kiem-ket-noi:8080/ ...; do sleep 2; done`.
- Log của mọi Pod một Job, không cắt: `kubectl logs -l job-name=kiem-ket-noi --tail=-1`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi tự viết hai manifest</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a; W=~/lab-work/5a
WEB="$(kubectl -n "$NS" get pod -l 'app.kubernetes.io/component=web,!lab' -o jsonpath='{.items[0].metadata.name}')"

kubectl apply -f "$W/sts-thu.yaml"
kubectl -n "$NS" rollout status statefulset/sts-thu --timeout=180s
OUT=''; for i in $(seq 1 30); do
  OUT="$(kubectl -n "$NS" exec "$WEB" -c status-writer -- wget -qO- -T 2 http://sts-thu-1.sts-thu:8080/ 2>/dev/null)"
  [ "$OUT" = 'sts-thu-1' ] && break; sleep 2
done
[ "$OUT" = 'sts-thu-1' ] && echo 'PASS: sts-thu-1.sts-thu phan giai va tra dung sts-thu-1'

GRACE="$(kubectl -n "$NS" get pod sts-thu-0 --template '{{.spec.terminationGracePeriodSeconds}}')"
kubectl -n "$NS" delete statefulset -l lab=sts-thu
sleep "$GRACE"
kubectl -n "$NS" wait --for=delete pod -l lab=sts-thu --timeout=60s
{ [ "$(kubectl -n "$NS" get pod -l lab=sts-thu --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  kubectl -n "$NS" get service sts-thu >/dev/null 2>&1 && \
  [ "$(kubectl -n "$NS" get pvc --no-headers 2>/dev/null | wc -l)" -eq 0 ]; } \
  && echo 'PASS: Pod di theo StatefulSet, Service quan tri van con, khong co PVC (no lab #2)'
kubectl -n "$NS" delete service sts-thu

kubectl apply -f "$W/kiem-ket-noi.yaml"
kubectl -n "$NS" wait --for=condition=Complete job/kiem-ket-noi --timeout=300s && echo 'PASS: Job kiem-ket-noi Complete'
kubectl -n "$NS" get pod -l job-name=kiem-ket-noi -o jsonpath='{range .items[*]}{.spec.hostname}.{.spec.subdomain}{"\n"}{end}' \
  | sort > "$EV/b12-hostname.txt"
[ "$(paste -sd' ' "$EV/b12-hostname.txt")" = 'kiem-ket-noi-0.kiem-ket-noi kiem-ket-noi-1.kiem-ket-noi kiem-ket-noi-2.kiem-ket-noi' ] \
  && echo 'PASS: hostname va subdomain cua ba Pod dung mau ${jobName}-${index}'
kubectl -n "$NS" logs -l job-name=kiem-ket-noi --tail=-1 > "$EV/b12-job.log"
[ "$(grep -c '^da goi duoc' "$EV/b12-job.log")" -eq 9 ] && echo 'PASS: du 9 lan goi thanh cong giua ba Pod'

kubectl delete -f "$W/kiem-ket-noi.yaml" --wait=true >/dev/null
[ "$(kubectl -n "$NS" get statefulset,job,service,pod -l 'lab in (sts-thu,kiem-ket-noi)' --no-headers 2>/dev/null | wc -l)" -eq 0 ] \
  && echo 'PASS: khong con object thu nghiem'
```

</details>

<details>
<summary>Lời giải tham khảo</summary>

```yaml
# ~/lab-work/5a/sts-thu.yaml
apiVersion: v1
kind: Service
metadata:
  name: sts-thu
  namespace: bookstore
  labels: {lab: sts-thu}
spec:
  clusterIP: None
  selector: {lab: sts-thu}
  ports:
  - {name: http, port: 8080, targetPort: 8080}
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sts-thu
  namespace: bookstore
  labels: {lab: sts-thu}
spec:
  serviceName: sts-thu
  replicas: 2
  selector:
    matchLabels: {lab: sts-thu}
  template:
    metadata:
      labels: {lab: sts-thu}
    spec:
      terminationGracePeriodSeconds: 5
      containers:
      - name: httpd
        image: busybox:1.37
        command: ["sh", "-c", "mkdir -p /www && hostname > /www/index.html && exec httpd -f -p 8080 -h /www"]
        ports:
        - {name: http, containerPort: 8080}
```

```yaml
# ~/lab-work/5a/kiem-ket-noi.yaml
apiVersion: v1
kind: Service
metadata:
  name: kiem-ket-noi
  namespace: bookstore
  labels: {lab: kiem-ket-noi}
spec:
  clusterIP: None
  selector:
    job-name: kiem-ket-noi
  ports:
  - {name: http, port: 8080, targetPort: 8080}
---
apiVersion: batch/v1
kind: Job
metadata:
  name: kiem-ket-noi
  namespace: bookstore
  labels: {lab: kiem-ket-noi}
spec:
  completions: 3
  parallelism: 3
  completionMode: Indexed
  backoffLimit: 2
  activeDeadlineSeconds: 300
  template:
    metadata:
      labels: {lab: kiem-ket-noi}
    spec:
      subdomain: kiem-ket-noi
      restartPolicy: Never
      containers:
      - name: peer
        image: busybox:1.37
        command:
        - sh
        - -c
        - |
          mkdir -p /www && hostname > /www/index.html
          httpd -p 8080 -h /www
          for i in 0 1 2; do
            until wget -q -T 2 -O- "http://kiem-ket-noi-$i.kiem-ket-noi:8080/" 2>/dev/null | grep -qx "kiem-ket-noi-$i"; do
              sleep 2
            done
            echo "da goi duoc kiem-ket-noi-$i"
          done
          sleep 20   # phục vụ thêm cho Pod nào còn chưa gọi tới mình
```

Ý chính cho ADR:

1. Service quản trị là object bạn tạo và sở hữu; StatefulSet chỉ nhắc tên nó qua `serviceName` — nên
   bài 340 bảo xóa Service riêng. `teardown.sh` xóa cả namespace nên Service đi theo; nó thu `db` về 0
   trước vì bài 65 nói xóa StatefulSet không bảo đảm Pod tắt có thứ tự và êm.
2. Chế độ Indexed đặt hostname theo `${jobName}-${completionIndex}` (bài 354): tên Job biết trước, chỉ
   số là 0..2. Thiếu `subdomain`: Job vẫn chạy nhưng không bản ghi nào gắn Pod vào miền
   `kiem-ket-noi`, các vòng `until` lặp mãi tới khi hết `activeDeadlineSeconds`. `dnsPolicy: Default`
   thì Pod dùng DNS của node, không có danh sách `search` của cluster — dạng tên rút gọn không chạy.
3. Pod xong sớm mà thoát ngay thì IP và bản ghi DNS của nó biến mất trong khi Pod chậm hơn còn đang chờ
   gọi tới nó — Pod chậm sẽ chờ mãi. Phục vụ thêm một lúc trước khi thoát cho mọi Pod kịp gọi xong.

</details>

## B13. [BS-5A-13] Điều tra: đối chiếu hệ thống compose cũ

**Loại:** Điều tra.

**Bối cảnh:** sau sáu sprint, chủ hiệu sách hỏi: "nếu ngày đầu chạy một công cụ chuyển đổi tự động
file compose thì có nhanh hơn không?". Trả lời bằng cách đối chiếu từng dòng của
[file compose ở mục 2.1](DU-AN-BOOKSTORE.md#21-hiện-trạng) với hệ thống hiện tại, theo đúng các quy
tắc chuyển đổi bài 294 mô tả — **không** cài Kompose.

**Yêu cầu:**

1. ADR có một bảng: mỗi phần tử của file compose — bốn service `db`, `api`, `web`, `backup`, các khóa
   `image`, `environment`, `volumes`, `ports`, `depends_on`, `restart` ngầm định, và volume `dbdata` —
   ứng với object, trường nào trong hệ thống sau sprint 6, và vấn đề `P1`…`P8` nào nó gỡ hay còn treo.
2. Cột thứ hai của bảng: theo quy tắc của bài 294, Kompose sẽ sinh ra gì cho dòng đó — và vì sao kết
   quả đó sai hoặc thiếu so với hệ thống của dự án. Ít nhất nêu: service không khai `ports`, `restart`
   bỏ trống, service có volume, image không tag.
3. Script nghiệm thu kiểm vế Kubernetes của từng dòng chính trên cluster đang chạy.

**Ràng buộc:** không tải, không cài Kompose. Bảng viết bằng lời của bạn.

**Đọc lại:**

- Bài [294](../294-translate-compose-kubernetes-vi.md): [*Sử dụng Kompose*](../294-translate-compose-kubernetes-vi.md#sử-dụng-kompose-use-kompose),
  [*Các kiểu chuyển đổi thay thế*](../294-translate-compose-kubernetes-vi.md#các-kiểu-chuyển-đổi-thay-thế-alternative-conversions),
  [*Label*](../294-translate-compose-kubernetes-vi.md#label-labels),
  [*Restart*](../294-translate-compose-kubernetes-vi.md#restart),
  [*Cảnh báo về cấu hình Deployment*](../294-translate-compose-kubernetes-vi.md#cảnh-báo-về-cấu-hình-deployment-warning-about-deployment-configurations),
  và dòng `WARN ... Service cannot be created because of missing port` ở
  [*Ví dụ với OpenShift*](../294-translate-compose-kubernetes-vi.md#ví-dụ-kompose-convert-với-openshift-openshift-kompose-convert-example).
- [Mục 2.1 và 2.2 của file dự án](DU-AN-BOOKSTORE.md#21-hiện-trạng).

**Bàn giao:** `docs/adr/NNNN-doi-chieu-compose.md`; `~/lab-evidence/5a/b13-*`.

**Nghiệm thu:**

- `PASS:` mọi image của dự án ghim tag, không `latest` (`P2`).
- `PASS:` chuỗi mật khẩu của file compose không có trong bất cứ thứ gì Kustomize render ra (`P3`).
- `PASS:` `depends_on` thành init container chờ DB của `api`; `ports: 80:8080` thành NodePort 30080
  tới cổng 8080; `./sql` thành ConfigMap mount ở `/docker-entrypoint-initdb.d`.
- `PASS:` volume `dbdata` hiện vẫn là `emptyDir` (`TD-04`); vòng lặp `backup` thành CronJob không chạy
  chồng (`P6` mới gỡ một nửa — `TD-08`).
- `PASS:` hostname `db` trong URI cũ thành Service headless `db` và tên `db-0.db`.
- `PASS:` ADR nhắc đủ bốn service `db`, `api`, `web`, `backup`.

**Câu hỏi thiết kế** — trả lời trong `docs/adr/NNNN-doi-chieu-compose.md`:

1. Service `db` trong compose không khai `ports`. Theo bài 294, Kompose sinh gì cho nó, và `api` — vốn
   gọi `@db:5432` — có còn tìm được DB không?
2. Service `backup` không khai `restart`. Kompose sinh object gì, với `restartPolicy` nào, và vì sao đó
   là hình dạng sai cho việc sao lưu định kỳ?
3. `db` có volume. Kompose đổi chiến lược của Deployment thành gì, vì sao, và dự án đã chọn object khác
   ở sprint nào để có cùng bảo đảm ấy một cách đúng nghĩa?
4. Kết luận cho chủ hiệu sách: công cụ chuyển đổi tiết kiệm được phần nào, và phần nào sáu sprint vừa
   qua mới làm được?

<details>
<summary>Gợi ý</summary>

- Liệt kê image của mọi container, kể cả init container:
  `-o jsonpath='{range .items[*]}{range .spec.initContainers[*]}{.image}{"\n"}{end}{range .spec.containers[*]}{.image}{"\n"}{end}{end}'`.
- Render toàn bộ để tìm chuỗi:
  `kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | grep -c doi-mat-khau-nay`.

</details>

<details>
<summary>Script nghiệm thu — chạy sau khi viết ADR</summary>

```bash
NS=bookstore; EV=~/lab-evidence/5a
cd ~/bookstore
SEL='app.kubernetes.io/part-of=bookstore'
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }

kubectl -n "$NS" get pod -l "$SEL" \
  -o jsonpath='{range .items[*]}{range .spec.initContainers[*]}{.image}{"\n"}{end}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
  | sort -u > "$EV/b13-images.txt"
{ ! grep -Ev ':[^:/]+$' "$EV/b13-images.txt" | grep -q . && ! grep -q ':latest$' "$EV/b13-images.txt"; } \
  && echo "PASS: $(grep -c . "$EV/b13-images.txt") image deu ghim tag, khong latest (P2)"
kubectl kustomize --load-restrictor=LoadRestrictionsNone k8s/app | grep -q 'doi-mat-khau-nay' \
  && echo 'FAIL: mat khau cua compose con trong manifest' || echo 'PASS: mat khau cua file compose khong co trong manifest (P3)'

[ -n "$(jp deployment/api '{.spec.template.spec.initContainers[*].name}')" ] \
  && echo 'PASS: depends_on [db] thanh init container cho DB cua api'
{ [ "$(jp service/web '{.spec.type} {.spec.ports[0].nodePort}')" = 'NodePort 30080' ] && \
  [ "$(kubectl -n "$NS" get endpointslice -l kubernetes.io/service-name=web -o jsonpath='{.items[0].ports[0].port}')" = '8080' ]; } \
  && echo 'PASS: ports 80:8080 thanh NodePort 30080 toi cong 8080 cua Pod'
M="$(jp statefulset/db '{.spec.template.spec.containers[?(@.name=="postgres")].volumeMounts[?(@.mountPath=="/docker-entrypoint-initdb.d")].name}')"
jp statefulset/db "{.spec.template.spec.volumes[?(@.name==\"$M\")].configMap.name}" | grep -q '^db-init' \
  && echo 'PASS: ./sql thanh ConfigMap db-init mount o /docker-entrypoint-initdb.d'
{ jp statefulset/db '{.spec.template.spec.volumes[*].emptyDir}' | grep -q '{}' && \
  [ "$(kubectl -n "$NS" get pvc --no-headers 2>/dev/null | wc -l)" -eq 0 ]; } \
  && echo 'PASS: volume dbdata hien van la emptyDir — TD-04 con mo'
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = 'Forbid' ] \
  && echo 'PASS: vong lap backup thanh CronJob khong chay chong (ban dump van chua ben — TD-08)'
{ [ "$(jp service/db '{.spec.clusterIP}')" = 'None' ] && jp deployment/api '{.spec.template.spec}' | grep -q 'db-0\.db'; } \
  && echo 'PASS: hostname db trong URI cu thanh Service headless db va ten db-0.db'
F="$(ls docs/adr/*-doi-chieu-compose.md 2>/dev/null | head -1)"
MISS=0; for S in db api web backup; do [ -n "$F" ] && grep -qw "$S" "$F" || MISS=1; done
[ "$MISS" -eq 0 ] && echo "PASS: $F nhac du bon service cua file compose"
```

</details>

<details>
<summary>Lời giải tham khảo — ý chính của ADR</summary>

| Compose | Hệ thống sau sprint 6 | Kompose (bài 294) sẽ sinh |
| --- | --- | --- |
| `db.image: postgres`, không tag | StatefulSet `db`, image ghim tag ở A1.4 — gỡ `P2` | Deployment với đúng `postgres` không tag — `P2` còn nguyên |
| `db.environment` có mật khẩu | Secret `db-credentials` sinh tại chỗ, đọc qua file — gỡ `P3` | biến môi trường chứa mật khẩu trong manifest |
| `db.volumes: dbdata`, `./sql` | `emptyDir` (`TD-04`, Lab 6a); ConfigMap `db-init` | PVC; chiến lược Deployment đổi thành `Recreate` |
| `db` không có `ports` | Service headless `db`, tên `db-0.db` | **không** sinh Service (`Service cannot be created because of missing port`) |
| `api.PGRST_DB_URI ...@db:5432` | URI ghép từ biến, host `db-0.db`, mật khẩu từ Secret | giữ nguyên chuỗi — `db` không phân giải vì không có Service |
| `api.depends_on: [db]` | init container `wait-for-db` + readiness `/ready` — gỡ `P4` | Kubernetes không có `depends_on` |
| `web.ports: "80:8080"` | Service `web` NodePort 30080 → `http` 8080 | Service cổng 80 → 8080, `ClusterIP` mặc định |
| `web.volumes` cấu hình nginx, HTML | ConfigMap `web-config` sinh bằng Kustomize, đổi là rollout | không chuyển được bind mount từ máy |
| `backup` vòng lặp, `restart` bỏ trống | CronJob `db-backup` `Forbid` — gỡ nửa `P6` (`TD-08`) | object controller `restartPolicy: Always` chạy vòng lặp mãi |
| một VM, mạng phẳng | ba node, nhiều bản sao (gỡ một phần `P1`); NetworkPolicy đã viết (`TD-09`) | không nói gì về số bản sao hay mạng |

1. Không có Service cho `db` vì service đó không khai `ports` (cảnh báo trong ví dụ OpenShift của bài),
   nên tên `db` không phân giải — `api` không tìm được DB. Dự án giải bằng Service headless do mình
   viết.
2. Bảng *Restart*: `restart` bỏ trống → object controller với `restartPolicy: Always`. Một Deployment
   chạy vòng `while true; sleep 3600` — không có lịch, không có lịch sử lần chạy, không có
   `concurrencyPolicy`; đúng hình dạng là CronJob (Lab 4b).
3. `Recreate`, để hai instance không cùng ghi một volume (bài 294). Dự án có cùng bảo đảm "nhiều nhất
   một" bằng StatefulSet ở Lab 4b — đúng nghĩa hơn, kèm định danh ổn định.
4. Công cụ tiết kiệm phần gõ YAML ban đầu. Những thứ làm hệ thống chạy được và vận hành được — bí mật,
   probe, tài nguyên, controller đúng loại, tên ổn định, tắt êm, migration, backup, cấu hình đổi là
   rollout — đều là quyết định thiết kế mà sáu sprint mới đưa vào.

</details>

## B14. Đóng sprint: cleanup và gate cuối

**Mục đích:** bàn giao một repo dựng lại được toàn bộ hệ thống của sprint 6, rồi trả cluster về đúng
`01-cluster-ready`.

1. Thay `tests/smoke.sh` bằng phiên bản của Lab 5a — kiểm thử khói **tích lũy**, chép nguyên văn:

```bash
cat > ~/bookstore/tests/smoke.sh <<'EOF'
#!/usr/bin/env bash
# Kiểm thử khói — trạng thái sau Lab 5a: các tầng gọi nhau bằng tên qua Service và DNS, web mở
# NodePort 30080, mọi object đi qua Kustomize. Kiểm mọi hợp đồng còn hiệu lực tới sprint 6.
set -u
cd "$(dirname "$0")/.."
NS=bookstore
SEL='app.kubernetes.io/part-of=bookstore'
FAIL=0
pass() { echo "PASS: $*"; }
fail() { echo "FAIL: $*"; FAIL=1; }
k() { kubectl -n "$NS" "$@"; }
jp() { kubectl -n "$NS" get "$1" -o jsonpath="$2" 2>/dev/null; }
nip() { kubectl get node "$1" -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'; }
ready_eps() { k get endpointslice -l "kubernetes.io/service-name=$1" \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]} {.conditions.ready}{"\n"}{end}' 2>/dev/null \
  | awk '$2=="true"{print $1}' | sort -u | grep -c .; }

# 1. Namespace, object cấp cluster, repo
[ "$(kubectl get namespace "$NS" -o jsonpath='{.metadata.labels.app\.kubernetes\.io/part-of}' 2>/dev/null)" = 'bookstore' ] \
  && pass 'namespace bookstore mang label du an' || fail 'namespace bookstore thieu hoac thieu label'
PRIO="$(kubectl get priorityclass bookstore-critical -o jsonpath='{.value}' 2>/dev/null)"
[ -n "$PRIO" ] && pass "co PriorityClass bookstore-critical ($PRIO)" || fail 'thieu PriorityClass bookstore-critical'
[ -z "$(find k8s -name '*.tpl')" ] && pass 'repo khong con file .tpl (TD-03)' || fail 'repo con file .tpl'

# 2. Cấu hình (tên có hậu tố hash từ 5a — đọc từ spec), bí mật, PDB
CM_WEB="$(jp deployment/web '{.spec.template.spec.volumes[*].configMap.name}' | tr ' ' '\n' | grep -m1 '^web-config')"
CM_API="$(jp deployment/api '{.spec.template.spec.containers[*].envFrom[*].configMapRef.name}' | tr ' ' '\n' | grep -m1 '^api-config')"
CM_DB="$(jp statefulset/db '{.spec.template.spec.volumes[*].configMap.name}' | tr ' ' '\n' | grep -m1 '^db-init')"
for C in "web-config:$CM_WEB" "api-config:$CM_API" "db-init:$CM_DB"; do
  N="${C#*:}"
  { [ -n "$N" ] && k get configmap "$N" >/dev/null 2>&1; } && pass "ConfigMap ${C%%:*} -> $N ton tai" \
    || fail "ConfigMap ${C%%:*} khong duoc tham chieu hoac khong ton tai"
done
D="$(jp secret/db-credentials '{.data}')"
{ echo "$D" | grep -q '"postgres-password"' && echo "$D" | grep -q '"authenticator-password"'; } \
  && pass 'Secret db-credentials ten co dinh, du hai key' || fail 'Secret db-credentials thieu'
for P in web api; do
  [ "$(jp "pdb/$P" '{.spec.maxUnavailable}/{.spec.minAvailable}')" = '1/' ] \
    && pass "PDB $P maxUnavailable 1" || fail "PDB $P khong theo hop dong maxUnavailable 1"
done

# 3. Controller cho mọi thành phần
[ "$(jp statefulset/db '{.status.readyReplicas}')" = '1' ] && pass 'StatefulSet db 1/1 Ready' || fail 'StatefulSet db chua Ready'
[ "$(jp pod/db-0 '{.metadata.ownerReferences[0].kind}/{.metadata.ownerReferences[0].name}')" = 'StatefulSet/db' ] \
  && pass 'db-0 thuoc StatefulSet db' || fail 'db-0 khong thuoc StatefulSet db'
[ "$(jp pod/db-0 '{.spec.nodeName}')" = 'lab-k8s-worker2' ] && pass 'db-0 tren lab-k8s-worker2' || fail 'db-0 khong o lab-k8s-worker2'
[ "$(jp pod/db-0 '{.status.qosClass}')" = 'Guaranteed' ] && pass 'db-0 QoS Guaranteed' || fail 'db-0 khong Guaranteed'
[ -n "$PRIO" ] && [ "$(jp pod/db-0 '{.spec.priority}')" = "$PRIO" ] && pass 'db-0 dung bookstore-critical' || fail 'db-0 sai priority'
for X in web:3 api:2; do
  N="${X%%:*}"; WANT="${X##*:}"
  [ "$(jp "deployment/$N" '{.status.readyReplicas}')" = "$WANT" ] && pass "Deployment $N $WANT/$WANT Ready" \
    || fail "Deployment $N chua du $WANT ban sao Ready"
  [ "$(k get deployment "$N" -o go-template='{{range $k, $v := .spec.selector.matchLabels}}{{$k}}={{$v}} {{end}}' 2>/dev/null)" \
    = "app.kubernetes.io/component=$N app.kubernetes.io/part-of=bookstore " ] \
    && pass "selector cua $N dung hop dong" || fail "selector cua $N sai hop dong"
  [ -n "$(jp "deployment/$N" '{.metadata.annotations.kubernetes\.io/change-cause}')" ] \
    && pass "Deployment $N co ly do phat hanh" || fail "Deployment $N thieu kubernetes.io/change-cause"
done
NODES="$(kubectl get nodes --no-headers | wc -l)"
[ "$(jp daemonset/node-agent '{.status.numberReady}')" = "$NODES" ] && pass "node-agent Ready tren $NODES/$NODES node" \
  || fail 'node-agent chua Ready tren moi node'
ORPHAN="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{.metadata.name}:{.metadata.ownerReferences[0].kind}{"\n"}{end}' 2>/dev/null | grep -c ':$')"
[ "$ORPHAN" -eq 0 ] && pass 'moi Pod cua du an deu co controller' || fail "$ORPHAN Pod tran"
MISS="$(k get pods -l "$SEL" -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{range .spec.initContainers[?(@.restartPolicy=="Always")]}{.name}={.resources.requests.cpu},{.resources.limits.memory}{"\n"}{end}{end}' 2>/dev/null \
  | grep -E '=,|,$' | sort -u)"
[ -z "$MISS" ] && pass 'moi container chinh va sidecar co requests va limits' || fail "thieu tai nguyen: $(echo $MISS)"

# 4. Service và EndpointSlice theo hợp đồng 4.2
[ "$(jp service/db '{.spec.clusterIP} {.spec.ports[0].port}')" = 'None 5432' ] && pass 'Service db headless, cong 5432' || fail 'Service db sai'
[ "$(jp service/api '{.spec.type} {.spec.ports[0].port} {.spec.ports[0].targetPort}')" = 'ClusterIP 3000 http' ] \
  && pass 'Service api ClusterIP 3000 -> http' || fail 'Service api sai'
[ "$(jp service/web '{.spec.type} {.spec.ports[0].port} {.spec.ports[0].targetPort} {.spec.ports[0].nodePort}')" = 'NodePort 80 http 30080' ] \
  && pass 'Service web NodePort 30080, 80 -> http' || fail 'Service web sai'
[ "$(jp service/node-agent '{.spec.ports[0].port} {.spec.ports[0].targetPort} {.spec.internalTrafficPolicy}')" = '9100 http Local' ] \
  && pass 'Service node-agent 9100 -> http, internalTrafficPolicy Local' || fail 'Service node-agent sai'
for X in api:2 web:3 db:1 "node-agent:$NODES"; do
  S="${X%%:*}"; WANT="${X##*:}"
  [ "$(ready_eps "$S")" -eq "$WANT" ] && pass "EndpointSlice $S co $WANT endpoint san sang" || fail "EndpointSlice $S khong du $WANT endpoint"
done

# 5. Dữ liệu, migration, gọi nhau bằng tên
N="$(k exec db-0 -c postgres -- psql -U postgres -d bookstore -tAc 'SELECT count(*) FROM api.books' 2>/dev/null)"
[ "${N:-0}" -ge 8 ] && pass "db co $N cuon sach (khoi tao + migration)" || fail "db co ${N:-0} cuon sach, can >= 8"
if k get job db-migrate >/dev/null 2>&1; then
  [ "$(jp job/db-migrate '{.status.conditions[?(@.type=="Complete")].status}')" = 'True' ] \
    && pass 'Job db-migrate Complete' || fail 'Job db-migrate chua Complete'
else
  pass 'Job db-migrate da duoc TTL don, ket qua nam trong db'
fi
BAD=0
for O in deployment/api job/db-migrate cronjob/db-backup; do
  k get "$O" >/dev/null 2>&1 || continue
  Y="$(k get "$O" -o yaml)"
  { echo "$Y" | grep -q 'db-0\.db' && ! echo "$Y" | grep -Eq '10\.244\.[0-9]+\.[0-9]+'; } || { echo "FAIL: $O khong goi DB bang ten"; BAD=1; }
done
[ "$BAD" -eq 0 ] && pass 'api, db-migrate, db-backup goi DB bang ten db-0.db, khong IP' || FAIL=1
WEB="$(k get pod -l "$SEL,app.kubernetes.io/component=web,!lab" --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)"
DB_IP="$(jp pod/db-0 '{.status.podIP}')"
k exec "$WEB" -c status-writer -- nslookup db-0.db 2>/dev/null | grep -q "$DB_IP" \
  && pass "db-0.db phan giai ra IP cua db-0 ($DB_IP)" || fail 'db-0.db khong phan giai dung'

# 6. Đường vào qua NodePort, gọi từ master
for NODE in lab-k8s-worker1 lab-k8s-worker2; do
  IP="$(nip "$NODE")"
  curl -s -m 5 "http://$IP:30080/" | grep -q 'Bookstore' && pass "$NODE:30080 tra trang Bookstore" || fail "$NODE:30080 khong tra trang chu"
  T="$(curl -s -m 5 "http://$IP:30080/api/books" | grep -o '"title"' | wc -l)"
  [ "$T" -ge 8 ] && pass "$NODE:30080/api/books tra $T cuon sach" || fail "$NODE:30080/api/books tra $T cuon sach"
done
[ "$(curl -s -m 5 "http://$(nip lab-k8s-worker1):30080/healthz")" = 'ok' ] && pass 'healthz qua NodePort tra ok' || fail 'healthz qua NodePort sai'

# 7. status.json của mọi Pod web: năm trường, agent cùng node
BAD=0
for P in $(k get pod -l "$SEL,app.kubernetes.io/component=web,!lab" -o jsonpath='{.items[*].metadata.name}' 2>/dev/null); do
  NODE="$(jp "pod/$P" '{.spec.nodeName}')"
  R="$(k exec "$P" -c status-writer -- wget -qO- http://127.0.0.1:8080/status.json 2>/dev/null | python3 -c '
import json, sys
try:
    d = json.load(sys.stdin); a = d.get("agent") or {}
    print(" ".join(sorted(d)), d.get("pod"), d.get("node"), a.get("node"))
except Exception:
    print("loi")')"
  [ "$R" = "agent node pod updated version $P $NODE $NODE" ] || { echo "FAIL: status.json cua $P: $R"; BAD=1; }
done
[ "$BAD" -eq 0 ] && pass 'moi Pod web: status.json du nam truong, agent la node-agent cung node' || FAIL=1

# 8. CronJob db-backup: chính sách và một lần chạy thử (gọi db-0.db)
[ "$(jp cronjob/db-backup '{.spec.concurrencyPolicy}')" = 'Forbid' ] && pass 'db-backup Forbid' || fail 'db-backup khong Forbid'
SDS="$(jp cronjob/db-backup '{.spec.startingDeadlineSeconds}')"
[ "${SDS:-0}" -ge 10 ] && pass "db-backup startingDeadlineSeconds=$SDS" || fail 'db-backup thieu startingDeadlineSeconds >= 10'
[ "$(jp cronjob/db-backup '{.spec.suspend}')" != 'true' ] && pass 'db-backup khong bi tam ngung' || fail 'db-backup dang suspend'
J="smoke-backup-$(date +%s)"
if k create job "$J" --from=cronjob/db-backup >/dev/null 2>&1 \
   && k wait --for=condition=Complete "job/$J" --timeout=180s >/dev/null 2>&1 \
   && k logs "job/$J" 2>/dev/null | grep -q '^db-backup: OK'; then
  pass 'chay thu db-backup thanh cong'
else
  fail 'chay thu db-backup that bai'
fi
k delete job "$J" --ignore-not-found --wait=true >/dev/null 2>&1

# 9. NetworkPolicy đã viết (chưa có hiệu lực trên Flannel — TD-09)
k get networkpolicy -o json 2>/dev/null | python3 -c '
import json, sys
ok = any(not (p["spec"].get("podSelector") or {}) and set(p["spec"].get("policyTypes") or []) == {"Ingress", "Egress"}
         and not p["spec"].get("ingress") and not p["spec"].get("egress") for p in json.load(sys.stdin)["items"])
sys.exit(0 if ok else 1)' && pass 'co NetworkPolicy chan mac dinh hai chieu (TD-09: chua co hieu luc tren Flannel)' \
  || fail 'thieu NetworkPolicy chan mac dinh hai chieu'

exit "$FAIL"
EOF
chmod +x ~/bookstore/tests/smoke.sh
```

2. Cập nhật `docs/tech-debt.md`: `TD-03` đổi trạng thái thành `đã trả` (5a, bằng chứng
   `~/lab-evidence/5a/b5-db.txt`); `TD-09` đã thêm ở B8.

3. Dọn object thử nghiệm còn sót, dựng lại sạch từ repo và chạy kiểm thử khói:

```bash
kubectl -n bookstore delete deployment,statefulset,job,service,pod -l lab --ignore-not-found --wait=true
cd ~/bookstore
./scripts/teardown.sh && ./scripts/deploy.sh > ~/lab-evidence/5a/b14-deploy.txt 2>&1
./tests/smoke.sh > ~/lab-evidence/5a/b14-smoke.txt 2>&1; echo "smoke exit=$?"
cat ~/lab-evidence/5a/b14-smoke.txt
```

4. Commit, tag, đóng gói, chép ra ngoài VM theo
   [mục 5.4 của file dự án](DU-AN-BOOKSTORE.md#54-sao-lưu-repo-ra-ngoài-vm):

```bash
cd ~/bookstore
git status --short
git add -A && git commit -m 'lab-5a: sprint 6 — Service, DNS, Kustomize day du, NodePort, NetworkPolicy'
git tag lab-5a
git bundle create ~/bookstore-lab-5a.bundle --all && git bundle verify ~/bookstore-lab-5a.bundle
```

```powershell
# Trên máy host Windows; sửa user và thư mục đích cho đúng máy của bạn
scp <user>@192.168.100.221:~/bookstore-lab-5a.bundle "$HOME\lab-backup\"
```

5. Dọn và trả cluster:

```bash
kubectl -n bookstore delete deployment,statefulset,job,service,pod -l lab --ignore-not-found --wait=true
rm -rf ~/lab-work/5a
cd ~/bookstore && ./scripts/teardown.sh
```

6. Gate cuối:

```bash
cd ~/bookstore
test -z "$(git status --porcelain)" && echo 'PASS: repo khong con thay doi chua commit'
git describe --tags --exact-match HEAD | grep -qx 'lab-5a' && echo 'PASS: HEAD mang tag lab-5a'
test -s ~/bookstore-lab-5a.bundle && echo 'PASS: co bundle'
BAD=0
for F in db/service.yaml api/service.yaml web/service.yaml agent/service.yaml api/deployment.yaml \
         jobs/db-migrate.yaml jobs/db-backup.yaml web/default.conf api/config.env db/01-schema.sql; do
  test -f "k8s/app/$F" || { echo "FAIL: thieu k8s/app/$F"; BAD=1; }
done
{ [ "$BAD" -eq 0 ] && [ -z "$(find k8s -name '*.tpl')" ] && ls k8s/app/network/*.yaml >/dev/null 2>&1; } \
  && echo 'PASS: bo cuc repo dung hang 5a cua muc 5.5'
grep -E 'TD-03.*đã trả' docs/tech-debt.md >/dev/null && grep -q 'TD-09' docs/tech-debt.md \
  && echo 'PASS: tech-debt.md cap nhat TD-03, TD-09'
BAD=0
for F in 'adr/*-service-ba-tang.md' 'adr/*-goi-nhau-bang-ten.md' 'adr/*-cong-vao-web.md' 'adr/*-agent-cung-node.md' \
         'postmortem/5a-endpointslice.md' 'adr/*-dns-trong-pod.md' 'adr/*-canary.md' 'adr/*-networkpolicy.md' \
         'adr/*-duong-vao-tu-ngoai.md' 'runbook-port-forward-db.md' 'adr/*-clusterip-va-topology.md' \
         'adr/*-headless-ngoai-db.md' 'adr/*-doi-chieu-compose.md'; do
  ls docs/$F >/dev/null 2>&1 || { echo "FAIL: thieu docs/$F"; BAD=1; }
done
[ "$BAD" -eq 0 ] && echo 'PASS: du ADR, postmortem va runbook cua sprint 6'
{ [ "$(kubectl get namespace,priorityclass -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 ] && \
  [ "$(kubectl get all -A -l app.kubernetes.io/part-of=bookstore --no-headers 2>/dev/null | wc -l)" -eq 0 ]; } \
  && echo 'PASS: khong con object nao cua du an'
kubectl get services -A -o jsonpath='{range .items[*]}{range .spec.ports[*]}{.nodePort}{"\n"}{end}{end}' | grep -qx 30080 \
  && echo 'FAIL: NodePort 30080 van bi giu' || echo 'PASS: NodePort 30080 da duoc tra'
test ! -e ~/lab-work/5a && echo 'PASS: da don lab-work'

kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** `smoke exit=0` và file smoke ở bước 3 in toàn `PASS:`; chín dòng `PASS:` đầu của gate cuối
xuất hiện; dòng `PASS: readyz ok`; ba node `Ready`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica; namespace `default` không có Pod. Thư mục `~/lab-evidence/5a/` **giữ lại**.
Cluster trở về `01-cluster-ready`; **không tạo snapshot mới**.

---

## 3. Checkpoint 5a

Không đánh dấu lab hoàn tất chỉ vì script nghiệm thu PASS. Tự trả lời, không nhìn tài liệu:

- [ ] Kể đường đi của `GET http://<IP worker>:30080/api/books` từ trình duyệt tới bảng `api.books`:
      ai lắng nghe cổng 30080, ai chọn Pod `web`, nginx tìm `api` thế nào, `api` tìm `db-0` thế nào.
      Chặng nào dùng ClusterIP ảo, chặng nào dùng IP Pod thật?
- [ ] Định danh của `db-0` gồm phần nào có từ Lab 4b, phần nào Service quản trị headless thêm vào? Vì
      sao `db` là headless mà `api` thì không? Nợ lab #3 đã trả ra sao?
- [ ] `port`, `targetPort` theo tên và cổng ghi trong EndpointSlice khác nhau ở đâu? Vì sao Service
      `web` từng có hai slice khi `web-le` chạy, và vì sao đọc một slice là kết luận sai?
- [ ] Ba condition `ready`, `serving`, `terminating`: giá trị của mỗi cái khi Pod NotReady và khi Pod
      đang tắt? Bài 49 nói "loại bỏ khỏi EndpointSlice" — bạn đo được gì?
- [ ] Xóa `db-0` ở Lab 4b và ở lab này khác nhau thế nào? Cái gì giờ tự lành, cái gì vẫn mất? Gọi đúng
      mã nợ.
- [ ] Đọc `/etc/resolv.conf` của một Pod `web`, giải thích từng dòng. `db-0.db` và
      `api.bookstore.svc.cluster.local` được thử theo thứ tự nào? Vì sao init chờ DB có thể chờ lâu
      hơn thời gian `db-0` cần để Ready?
- [ ] Hậu tố hash của ConfigMap biến việc sửa `default.conf` thành gì? ConfigMap cũ còn lại để làm gì?
      Vì sao Secret `db-credentials` vẫn giữ tên cố định, và vì sao dự án dùng `labels` chứ không
      `commonLabels`?
- [ ] `NodePort`, `LoadBalancer`, `ExternalName` lồng nhau thế nào, và trên cluster này mỗi loại dừng ở
      đâu? `externalTrafficPolicy: Local` và `internalTrafficPolicy: Local` mỗi cái lọc endpoint cho
      loại lưu lượng nào, cái giá là gì?
- [ ] Vì sao `status-writer` luôn đọc đúng agent cùng node? Khi nào node có Pod `web` mà không có agent,
      và hệ thống phản ứng ra sao?
- [ ] Canary: vì sao Service chia lưu lượng mà không phải sửa gì, tỷ lệ do đâu quyết định, và
      selector chồng lấn với Deployment `web` là rủi ro gì?
- [ ] NetworkPolicy của dự án: ma trận gồm những luồng nào, vì sao phải mở DNS, vì sao probe không cần
      rule, và vì sao hôm nay chúng không chặn gì? Lưu lượng NodePort sẽ ra sao khi chúng có hiệu lực?
- [ ] ClusterIP của `kube-dns` và của `api` nằm ở băng nào, vì sao? Vì sao định tuyến theo topology
      không có tác dụng ở đây, và vì sao `port-forward` là đường gỡ lỗi chứ không phải đường cho người
      dùng?

Nợ còn mở sau lab này — nói được **vì sao chưa trả**:

- Nợ lab #4 (NetworkPolicy được thực thi — Lab 5b), #2 (`volumeClaimTemplates` — Lab 6a), #1 (HPA ở
  Lab 11b; VPA chưa có lab).
- Nợ kỹ thuật: `TD-09` mới (lưu lượng giữa các tầng chưa bị giới hạn — Lab 5b), `TD-04` (dữ liệu trên
  `emptyDir` — Lab 6a), `TD-08` (bản dump chưa có nơi bền — Lab 6a), `TD-06` (`db` khởi động bằng root
  — Lab 9b), `TD-07` (quyền debug — Lab 9a). **Đã trả ở lab này:** nợ lab #3, nợ readiness →
  EndpointSlice của Lab 3a, nợ canary của Lab 4a, `TD-03`.

### Bài giải thích cuối cùng

Trong vài phút, kể lại một request: bạn mở `http://<IP worker>:30080/` trên máy host. Đi qua cổng
NodePort trên node, kube-proxy chọn một endpoint của Service `web` trong EndpointSlice, nginx trả trang
và đoạn script gọi `/api/books`; nginx chuyển sang `api` bằng tên đã phân giải lúc khởi động, PostgREST
gọi `db-0.db` qua CoreDNS và Service quản trị. Rồi kể kịch bản B5: `db-0` bị xóa — condition nào đổi
trong slice nào, nginx trả gì trong lúc đó, ai tạo lại `db-0`, vì sao `api` tự Ready lại, và vì sao vẫn
chỉ còn năm cuốn sách. Kết thúc bằng việc chỉ ra Lab 5b (Calico, Ingress, NetworkPolicy có hiệu lực) và
Lab 6a (dữ liệu bền) sẽ thay đổi câu chuyện này ở điểm nào.

---

## 4. Troubleshooting của lab này

Sự cố dựng môi trường xem [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Dưới đây chỉ là sự cố phát sinh trong nội dung bài học.

| Triệu chứng | Nguyên nhân thường gặp | Xử lý |
| --- | --- | --- |
| B1: `db-0.db.bookstore.svc.cluster.local` không phân giải dù Service `db` đã có | Đã có truy vấn hỏi tên đó trước khi Service tồn tại; CoreDNS nhớ kết quả thất bại (bài 65) | Chờ hết thời gian cache rồi hỏi lại; vòng chờ của script cho tối đa 60 giây |
| B1: `apply` Service `db` báo `field is immutable` ở `clusterIP` | Service `db` từng được tạo là ClusterIP thường rồi mới đổi sang `None` | Xóa Service `db` rồi `./scripts/deploy.sh`; `clusterIP` không đổi được tại chỗ |
| B1: EndpointSlice của `api` rỗng | Selector của Service sai một label, hoặc `targetPort` mang tên không có trên Pod | So `kubectl get svc api -o yaml` với label và `ports[].name` của Pod |
| B2: `deploy.sh` báo `security; file ... is not in or below` | Render bằng `kubectl apply -k` hay `kubectl kustomize` không có `--load-restrictor` | Mọi triển khai đi qua `./scripts/deploy.sh` (ADR 0007) |
| B2: `deploy.sh` báo `con file .tpl` | Còn file `.tpl` cũ, hoặc file nguồn trang mẫu mang đuôi `.tpl` | `git mv` sang tên mới; key `index.html.tpl` đặt bằng cú pháp `key=đường dẫn` |
| B2: Pod `web` `CreateContainerConfigError` hoặc `FailedMount` sau khi đổi sang bộ sinh | Volume tham chiếu tên ConfigMap không có trong kustomization, nên Kustomize không thay bằng tên có hậu tố | Tên trong manifest phải trùng `name` của `configMapGenerator`; xem bản render |
| B2: `db-migrate` có lần thử `Failed`, log `could not translate host name "db-0.db"` | Thiếu init container chờ DB; Job chạy trước khi `db-0` Ready | Thêm init `wait-for-db` như yêu cầu 2 |
| B2, B5: `api` `CrashLoopBackOff`, log lỗi xác thực | `secrets/db.env` bị sinh lại trong khi `db-0` còn dữ liệu với mật khẩu cũ | Xóa `db-0` để khởi tạo lại với Secret hiện tại, rồi `./scripts/deploy.sh` |
| B3: Pod `web` `CrashLoopBackOff`, log nginx `host not found in upstream "api"` | nginx khởi động khi Service `api` chưa có, hoặc gõ sai tên | Kiểm `kubectl get svc api`; nginx tự khởi động lại và lên khi tên phân giải được |
| B3: `/api/books` trả `404` của PostgREST | `proxy_pass` thiếu `/` cuối nên tiền tố `/api` vẫn bị gửi sang | `proxy_pass http://api:3000/;` |
| B3: chỉ một ReplicaSet `web`, gate rollout fail | Đã teardown giữa chừng nên mất lịch sử | Sửa thêm một dòng chú thích trong `default.conf`, `./scripts/deploy.sh` lại |
| B3: trình duyệt trên host không mở được 30080 | Tường lửa trên node, hoặc IP sai | `curl` từ master trước; nếu master gọi được mà host không, xem A4.4 của Lab 00 |
| B4: `agent` luôn `null` | Thiếu `-T` nên `wget` treo, Service `node-agent` sai cổng, hoặc agent chưa Ready trên node đó | `kubectl exec deploy/web -c status-writer -- wget -qO- -T 2 http://node-agent:9100/node.json` |
| B4: `status.json` không phải JSON hợp lệ | `$A` bị bọc trong nháy kép trong `printf` | Đưa `$A` vào chuỗi định dạng không nháy — nó đã là JSON |
| B5: không bắt được cửa sổ `terminating` | Thiếu `preStop` hoặc ân hạn ngắn hơn `preStop` | Đúng manifest `web-le`: `preStop` 20 giây, ân hạn 40 giây |
| B5: `api` không bao giờ `ready=false` khi xóa `db-0` | `db-0` mới lên rất nhanh, trước hai lần readiness thất bại | Chạy lại kịch bản B; nếu vẫn vậy, ghi vào postmortem kèm `~/lab-evidence/5a/b5-db.txt` |
| B6: tên ngắn `api` phân giải được từ namespace `default` | Có Service `api` khác ở namespace `default` | `kubectl get svc -n default`; đó không phải Service của dự án |
| B7: không request nào tới canary | Selector của Service khác selector chung, hoặc Pod canary chưa Ready | `kubectl -n bookstore get endpointslice -l kubernetes.io/service-name=web -o wide` |
| B8: bộ mô phỏng báo DNS bị chặn | Hai bộ chọn của luồng DNS tách làm hai phần tử, hoặc thiếu `UDP` | Một phần tử chứa cả `namespaceSelector` và `podSelector`; khai cả UDP và TCP |
| B9: `EXTERNAL-IP` của `web-lb` có địa chỉ thật | Có một hiện thực load balancer đang chạy — cluster lệch baseline | Dừng lại, đối chiếu [A1.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) |
| B9: master trả `200` với `web-lb-local` | Gọi từ chính master tới IP của master — kube-proxy xử lý lưu lượng phát từ chính node theo cách khác | Gọi từ `lab-k8s-worker1` như script |
| B10: `SSLRequest` không có trả lời | `port-forward` chưa sẵn sàng, hoặc cổng 15432 đang bị chiếm | Chờ dòng `Forwarding from`; đổi cổng cục bộ |
| B11: `SVC_CIDR` rỗng | Thiếu `sudo` khi đọc manifest API server | Chạy lại với `sudo` |
| B12: Job `kiem-ket-noi` không `Complete` | Thiếu `subdomain`, selector `job-name` sai tên Job, hoặc Pod xong sớm rồi thoát | Đối chiếu manifest; giữ `sleep` cuối script |
| B14: `teardown.sh` báo còn Service giữ NodePort 30080 | Service thử nghiệm ngoài namespace `bookstore` xin 30080 | Tìm bằng lệnh trong teardown, xóa Service đó |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Services, Load Balancing, and Networking](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/)
- [Kubernetes v1.35 — Service](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes v1.35 — EndpointSlices](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes v1.35 — DNS for Services and Pods](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes v1.35 — Pod Hostname](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/pod-hostname/)
- [Kubernetes v1.35 — Network Policies](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes v1.35 — Topology Aware Routing](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)
- [Kubernetes v1.35 — Service Internal Traffic Policy](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/service-traffic-policy/)
- [Kubernetes v1.35 — Service ClusterIP allocation](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/cluster-ip-allocation/)
- [Kubernetes v1.35 — Virtual IPs and Service Proxies](https://v1-35.docs.kubernetes.io/docs/reference/networking/virtual-ips/)
- [Kubernetes v1.35 — StatefulSets](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes v1.35 — Translate a Docker Compose File to Kubernetes Resources](https://v1-35.docs.kubernetes.io/docs/tasks/configure-pod-container/translate-compose-kubernetes/)
- [Kubernetes v1.35 — Managing Kubernetes Objects Using Imperative Commands](https://v1-35.docs.kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-command/)
- [Kubernetes v1.35 — Declarative Management of Kubernetes Objects Using Kustomize](https://v1-35.docs.kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/)
- [Kubernetes v1.35 — Delete a StatefulSet](https://v1-35.docs.kubernetes.io/docs/tasks/run-application/delete-stateful-set/)
- [Kubernetes v1.35 — Fine Parallel Processing Using a Work Queue](https://v1-35.docs.kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/)
- [Kubernetes v1.35 — Job with Pod-to-Pod Communication](https://v1-35.docs.kubernetes.io/docs/tasks/job/job-with-pod-to-pod-communication/)
- [Kubernetes v1.35 — Configure DNS for a Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/configure-dns-cluster/)
- [Kubernetes v1.35 — Connect a Frontend to a Backend Using Services](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/connecting-frontend-backend/)
- [Kubernetes v1.35 — Create an External Load Balancer](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/)
- [Kubernetes v1.35 — Use Port Forwarding to Access Applications in a Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)
- [Kubernetes — `kubectl create service clusterip`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_create/kubectl_create_service_clusterip/)
- [Kustomize — configMapGenerator](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/configmapgenerator/)
- [Kustomize — labels](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/labels/)
- [nginx — `proxy_pass`](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)
- [PostgreSQL 17 — Message Flow, mục SSL Session Encryption](https://www.postgresql.org/docs/17/protocol-flow.html#PROTOCOL-FLOW-SSL)
