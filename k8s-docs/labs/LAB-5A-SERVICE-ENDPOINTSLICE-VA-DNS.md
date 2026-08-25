# Lab 5a — Service, EndpointSlice và DNS

> **Điểm bắt đầu:** snapshot `01-cluster-ready` — xem
> [Lab 00 — Môi trường](LAB-00-MOI-TRUONG-1.35.7.md).
> **Điểm kết thúc:** cleanup trả cluster về đúng `01-cluster-ready`, không tạo snapshot mới.
> **Lab trước:** [Lab 4b — StatefulSet, DaemonSet và Job](LAB-4B-STATEFULSET-DAEMONSET-VA-JOB.md)
> đã cleanup namespace `lab-4b` và trả cluster về `01-cluster-ready`, nên điểm bắt đầu của lab này
> là một cluster sạch không workload.
>
> **Cập nhật và đối chiếu phiên bản:** 25/08/2026.

Lab này đi cùng phần **Service/DNS** của mục
[Giai đoạn 5 — Mạng nền tảng](../00-ALO-TRINH-ADMIN.md#giai-đoạn-5--mạng-nền-tảng). Phần
NetworkPolicy, Ingress và CNI của cùng giai đoạn thuộc **Lab 5b** — lab đó đổi CNI và cài ingress
controller nên nó tạo snapshot mới; lab này **không** đụng vào cả hai thứ đó.

Cluster giữ nguyên baseline của
[bảng A1.3 trong Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); lab này **không chép
lại con số phiên bản nào** và **không cài thêm gì** — không MetalLB, không ingress controller,
không đổi CNI, không sửa CoreDNS.

Lab dùng Deployment và ReplicaSet của giai đoạn 4, StatefulSet và Job của
[nhóm 4b](../00-ALO-TRINH-ADMIN.md#4b-statefulset-daemonset-job-và-autoscaling), Pod và probe của
giai đoạn 3 làm công cụ. **Không** dùng NetworkPolicy, Ingress, PVC, StorageClass, RBAC hay
metrics-server — tất cả thuộc giai đoạn sau.

Trước khi bắt đầu, chạy [quy trình mở đầu A5.5](LAB-00-MOI-TRUONG-1.35.7.md#a55-quy-trình-mở-đầu-mỗi-lab)
trên `lab-k8s-master`:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns
kubectl get pods -n default
```

**PASS:** ba node `Ready`; có dòng `PASS: readyz ok`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica `READY`; namespace `default` không có Pod.

---

## 1. Kết quả phải đạt

Hoàn thành lab khi có thể tự chứng minh và giải thích được:

- Mô hình mạng Kubernetes ở dạng đo được: hai Pod trên hai node khác nhau, thuộc hai PodCIDR khác
  nhau, gọi thẳng nhau **bằng Pod IP**, không proxy và không NAT; Pod nhìn thấy đúng địa chỉ mà
  API server báo cáo trong `.status.podIP`.
- Service chọn Pod bằng **selector**, và kết quả của việc chọn đó được ghi ra **EndpointSlice**;
  chỉ ra được cả hai chiều — đổi label của Pod thì EndpointSlice đổi theo, và ngược lại.
- ClusterIP là **địa chỉ IP ảo**: nó không nằm trên bất kỳ interface nào của node, không có tiến
  trình nào lắng nghe trên nó, nhưng TCP tới nó vẫn tới được backend.
- Ánh xạ `port` → `targetPort`, kể cả khi `targetPort` là **tên port** đặt trên Pod; đọc được số
  port thật mà EndpointSlice ghi lại.
- Ba condition `ready`, `serving`, `terminating` của một endpoint và thời điểm mỗi cái đổi giá trị.
- Bốn `type` của Service và thiết kế **lồng nhau**: `ClusterIP`, `NodePort` (vẫn có cluster IP),
  `LoadBalancer` (vẫn có cả hai, nhưng treo vô thời hạn trên cluster bare metal), `ExternalName`
  (chỉ là CNAME, **không** có cluster IP và **không** có proxy nào).
- DNS trong Pod: `nameserver`, danh sách `search` và `options ndots:5` trong `/etc/resolv.conf`;
  phân biệt tên ngắn, tên có namespace và FQDN, và nói được vì sao tên ngắn chỉ chạy trong
  namespace của chính Pod đó.
- Hai cách khám phá Service từ trong Pod — biến môi trường và DNS — và chứng minh được vì sao cách
  thứ nhất phụ thuộc thứ tự tạo object còn cách thứ hai thì không.
- `externalTrafficPolicy` và `internalTrafficPolicy`: đo được hệ quả của giá trị `Local` bằng
  chính khác biệt "node có endpoint thì gọi được, node không có thì không".
- Định tuyến nhận biết topology bật được bằng annotation nhưng **không có tác dụng** trên cluster
  một zone, và chỉ ra được đúng cơ chế bảo vệ nào đã chặn nó.
- ClusterIP đến từ đâu: đọc dải `service-cluster-ip-range` thật, tính băng tĩnh bằng công thức,
  xin một ClusterIP tĩnh thành công và một ClusterIP ngoài dải bị API server từ chối.
- Headless Service cho **DNS cấp Pod**: mỗi Pod có bản ghi riêng
  `<pod>.<service>.<ns>.svc.cluster.local`, và điều kiện để bản ghi đó tồn tại.
- **Nợ #3 đã trả**: StatefulSet có Service quản trị headless, và
  `web-0.web-headless.lab-5a.svc.cluster.local` phân giải được — đúng thứ Lab 4b cố ý bỏ trống.
- Cleanup toàn bộ object lab và đưa cluster về `01-cluster-ready`.

### Lab này trả nợ #3

| # | Nợ | Phát sinh ở | Trả ở |
| --- | --- | --- | --- |
| 3 | Service headless quản trị cho StatefulSet | [giai đoạn 4](../00-ALO-TRINH-ADMIN.md#giai-đoạn-4--workload-controller), bài [65](../65-statefulset-vi.md) | **B11 của lab này** |

**Đọc lại bài [65 — StatefulSets](../65-statefulset-vi.md), mục *Định danh mạng ổn định*, trước khi
làm B11.** Lab 4b đã dựng StatefulSet `web` với `serviceName: web-headless` trỏ tới một Service
**chưa tồn tại**, và chứng minh phần định danh còn lại vẫn đủ. B11 dựng nốt Service đó và đo phần
duy nhất còn thiếu: miền DNS `$(podname).$(governing service domain)`.

Hai món nợ khác của bài 65 **vẫn treo** sau lab này: nợ #2 (`volumeClaimTemplates`, trả ở Lab 6a)
và nợ #1 (HPA/VPA, trả ở Lab 11b). StatefulSet ở B11 vì vậy vẫn **không có**
`volumeClaimTemplates`. Xem [sổ nợ lab](README.md#5-sổ-nợ-lab) và
[Sổ nợ lộ trình](../00-ALO-TRINH-ADMIN.md#sổ-nợ-lộ-trình).

Lab này cũng **không** trả nợ #4 (NetworkPolicy được thực thi thật) — đó là việc của Lab 5b.

### 1.1. Ánh xạ tài liệu sang bài thực hành

| Bài trong nhóm Service/DNS | Kiểm chứng ở |
| --- | --- |
| [81 — Service, cân bằng tải và mạng](../81-services-networking-vi.md) | B1 — Pod xuyên node gọi nhau bằng Pod IP, không proxy không NAT; B1.3 ranh giới trách nhiệm giữa Kubernetes và CNI |
| [82 — Service](../82-service-vi.md) | B2 (selector, `port`/`targetPort` có tên, cluster IP ảo), B4.4 (biến môi trường), B6 (`NodePort`, `ExternalName`, `LoadBalancer` treo), B9 (Service không selector), B10 (headless) |
| [83 — EndpointSlices](../83-endpoint-slices-vi.md) | B2.3 (slice tự sinh, `kubernetes.io/service-name`, `managed-by`), B3 (bám selector, ba condition, gộp và khử trùng lặp nhiều slice), B9.2 (slice tự viết tay) |
| [10 — DNS cho Service và Pod](../10-dns-pod-service-vi.md) | B4 (`resolv.conf`, `search`, `ndots:5`, tên ngắn so với FQDN, namespace), B10 (bản ghi cấp Pod), B11 (miền DNS của StatefulSet) |
| [57 — Hostname của Pod](../57-pod-hostname-vi.md) | B10.3 — `spec.hostname` thắng `metadata.name`, cặp `hostname` + `subdomain` sinh bản ghi A, `setHostnameAsFQDN` |
| [86 — Định tuyến nhận biết topology](../86-topology-aware-routing-vi.md) | B8 — đặt annotation `service.kubernetes.io/topology-mode: Auto`, đọc lại, rồi chứng minh cơ chế bảo vệ số 3 đã chặn: không node nào có label zone nên EndpointSlice **không** có `hints` |
| [87 — Chính sách lưu lượng nội bộ của Service](../87-service-traffic-policy-vi.md) | B7.2 — `internalTrafficPolicy: Local`, đo trực tiếp trên hai worker: node có endpoint gọi được, node không có endpoint thì Service hành xử như không có endpoint nào |
| [88 — Cấp phát ClusterIP cho Service](../88-cluster-ip-allocation-vi.md) | B5 — đọc `--service-cluster-ip-range` từ manifest apiserver, kiểm ClusterIP nằm trong dải, tính băng tĩnh bằng công thức, xin IP tĩnh, và IP ngoài dải bị từ chối |
| [362 — Cấu hình DNS cho một cluster](../362-configure-dns-cluster-vi.md) | B0.3 và B4.1 — xác nhận addon DNS của cluster là CoreDNS do kubeadm cài, Service của nó là `kube-dns`, và `nameserver` trong Pod trỏ đúng ClusterIP đó |
| [363 — Kết nối Frontend với Backend bằng Service](../363-connecting-frontend-backend-vi.md) | B12 — backend sau một Service ClusterIP, frontend tìm backend **bằng tên DNS**, frontend expose bằng NodePort đúng như bài cho phép khi môi trường không có load balancer |
| [364 — Tạo bộ cân bằng tải bên ngoài](../364-create-external-load-balancer-vi.md) | B6.3 (Service `type: LoadBalancer` giữ nguyên ClusterIP và NodePort, `EXTERNAL-IP` treo `<pending>`) và B7.1 (`externalTrafficPolicy` `Cluster` so với `Local`) |
| [354 — Job với giao tiếp Pod-đến-Pod](../354-job-pod-to-pod-communication-vi.md) | B11.3 — headless Service selector `job-name`, `completionMode: Indexed`, `subdomain`; Job chỉ `Complete` khi mọi Pod gọi được nhau bằng hostname |
| [366 — Sử dụng Port Forwarding để truy cập ứng dụng](../366-port-forward-vi.md) | B9.3 — `port-forward` tới `service/` của Service **có** selector chạy được, tới Service **không** selector bị API server từ chối |

Bốn phần dưới đây **không kiểm chứng được trên cluster lab**, đọc để biết:

| Phần | Vì sao không có bước thực hành |
| --- | --- |
| Đường dữ liệu thật của `type: LoadBalancer` — bài [82](../82-service-vi.md#loadbalancer), [363](../363-connecting-frontend-backend-vi.md), [364](../364-create-external-load-balancer-vi.md) | Bài nói rõ đường dữ liệu do một load balancer **nằm ngoài cluster** cung cấp, và Kubernetes chỉ lập trình nó qua cloud-controller-manager. Cluster lab là bare metal, không có cloud provider; MetalLB nằm [ngoài baseline](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) và cài nó là đổi hạ tầng, tức phải là lab tạo snapshot mới. B6.3 chỉ kiểm chứng **phần API**: ClusterIP và NodePort vẫn được cấp, `status.loadBalancer` rỗng vĩnh viễn |
| Hiệu ứng định tuyến của bài [86](../86-topology-aware-routing-vi.md) | Cần cluster nhiều zone. Ba node lab nằm trên một mạng phẳng và không node nào mang label `topology.kubernetes.io/zone`, nên cơ chế bảo vệ số 3 luôn chặn. B8 kiểm chứng đúng phần **cấu hình được** — annotation vào được, đọc lại được — và chứng minh hint **không** được sinh, đó là hành vi đúng chứ không phải lỗi |
| `hostnameOverride` của bài [57](../57-pod-hostname-vi.md) | Bài ghi trạng thái tính năng là beta ở phiên bản baseline, nên không chắc feature gate đã bật trên cluster lab; và chính bài nói trường này **không** ảnh hưởng tới bản ghi A/AAAA của Pod trong DNS server, tức nó không đổi thứ mà lab này đo. Đọc để nhận ra khi gặp, không đặt vào manifest |
| Tùy chỉnh CoreDNS của bài [362](../362-configure-dns-cluster-vi.md) | Bài 362 là trang giới thiệu và trỏ toàn bộ phần cấu hình sang bài [204](../204-dns-custom-nameservers-vi.md), không nằm trong nhóm bài này. Sửa ConfigMap của CoreDNS là đụng vào `kube-system` — lab không làm. B0.3 và B4.1 chỉ **đọc** addon mặc định |

### 1.2. Thời lượng

3–4 giờ, tính từ lúc gate `01-cluster-ready` đã PASS. B3, B11 và B12 có bước chờ: B3 phải bắt cho
được cửa sổ `terminating`, còn B11 và B12 phải chờ DNS hội tụ — thời gian phụ thuộc cấu hình cache
của CoreDNS trên cluster của bạn.

---

## 2. Quy ước và an toàn

- Mọi lệnh chạy trên `lab-k8s-master` bằng user quản trị có kubeconfig, trừ khi ghi rõ node khác.
- **Chạy toàn bộ phần B trong cùng một phiên shell.** Rất nhiều gate so sánh biến đặt ở bước trước
  (`W1_IP`, `DNS_IP`, `CIP`, `SVC_CIDR`, `EXTRA_IP`…); mở shell mới giữa chừng là mất biến và phải
  chạy lại từ đầu mục B chứa bước đang dở.
- Mọi object lab tạo ra nằm trong namespace `lab-5a` và luôn được gọi kèm `-n lab-5a`. Lab không
  đổi namespace mặc định của context.
- Lab chỉ tạo Namespace, Pod, Deployment, StatefulSet, Job, Service và EndpointSlice. **Không** tạo
  NetworkPolicy, Ingress, IngressClass, PersistentVolumeClaim, StorageClass, Role, RoleBinding hay
  HorizontalPodAutoscaler. **Không** cài CNI khác, ingress controller, MetalLB hay metrics-server.
- Lab **chỉ đọc** `/etc/kubernetes/manifests/kube-apiserver.yaml` và các object trong `kube-system`.
  Không sửa manifest control plane, không sửa ConfigMap của CoreDNS hay kube-proxy, không sửa cấu
  hình kubelet hay containerd.
- **Fault injection chỉ trên `lab-k8s-worker2`** — B7.2 là bước duy nhất cố ý làm một lời gọi
  **thất bại**, và nó được chạy từ Pod nằm trên đúng node này.
- Manifest tạm ghi vào `~/lab-work/5a/`; bằng chứng ghi vào `~/lab-evidence/5a/`. Không lưu token,
  key hay certificate.
- Lab cần kéo được image `busybox` từ internet. Image dùng đúng tag đã khóa ở
  [bảng A1.3](LAB-00-MOI-TRUONG-1.35.7.md#a13-phiên-bản-được-khóa); nếu môi trường cô lập, xem mục 4.
- Dòng bắt đầu bằng `PASS:` là gate; không đi tiếp khi gate fail.

---

# Phần B — Thực hành kiến thức 5a

## B0. Chuẩn bị workspace, namespace và dữ liệu nền của cluster

Lab này so sánh rất nhiều giá trị với **số thật của cluster**, không với số in trong tài liệu.
B0 đọc hết những số đó một lần rồi giữ trong biến shell.

### B0.1. Workspace và namespace

```bash
mkdir -p ~/lab-work/5a ~/lab-evidence/5a
kubectl config current-context
kubectl create namespace lab-5a
kubectl get namespace lab-5a -o jsonpath='{.status.phase}'; echo
```

**PASS:** context trỏ đúng cluster lab; namespace `lab-5a` ở phase `Active`.

### B0.2. Tên node, IP node và PodCIDR

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[?(@.type=="InternalIP")].address}{"\t"}{.spec.podCIDR}{"\n"}{end}' \
  | tee ~/lab-evidence/5a/b0-nodes.txt

M_IP="$(kubectl get node lab-k8s-master  -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
W1_IP="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
W2_IP="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"
W1_CIDR="$(kubectl get node lab-k8s-worker1 -o jsonpath='{.spec.podCIDR}')"
W2_CIDR="$(kubectl get node lab-k8s-worker2 -o jsonpath='{.spec.podCIDR}')"
echo "master=$M_IP worker1=$W1_IP worker2=$W2_IP"
echo "podCIDR worker1=$W1_CIDR worker2=$W2_CIDR"

test -n "$M_IP" && test -n "$W1_IP" && test -n "$W2_IP" \
  && test "$W1_IP" != "$W2_IP" \
  && test -n "$W1_CIDR" && test -n "$W2_CIDR" && test "$W1_CIDR" != "$W2_CIDR" \
  && echo 'PASS: doc duoc IP ba node va hai PodCIDR khac nhau'
```

**Ý nghĩa:** hai worker mang hai PodCIDR khác nhau — đó là điều kiện để B1 thật sự đo được đường đi
**xuyên node**, chứ không phải hai Pod tình cờ nằm cùng subnet.

**PASS:** dòng `PASS: doc duoc IP ba node va hai PodCIDR khac nhau` xuất hiện.

### B0.3. Addon DNS của cluster

Bài [362](../362-configure-dns-cluster-vi.md) nói CoreDNS là lựa chọn được khuyến nghị và **được
cài mặc định cùng kubeadm**. Kiểm tra điều đó trên cluster của bạn, không suy đoán:

```bash
kubectl -n kube-system get deployment coredns -o wide
kubectl -n kube-system get svc kube-dns -o wide | tee ~/lab-evidence/5a/b0-dns-addon.txt

DNS_IP="$(kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.clusterIP}')"
DNS_IMG="$(kubectl -n kube-system get deployment coredns \
  -o jsonpath='{.spec.template.spec.containers[0].image}')"
DNS_READY="$(kubectl -n kube-system get deployment coredns -o jsonpath='{.status.readyReplicas}')"
echo "kube-dns ClusterIP = $DNS_IP"
echo "image = $DNS_IMG"

test -n "$DNS_IP" \
  && test "${DNS_READY:-0}" -ge 1 \
  && case "$DNS_IMG" in *coredns*) true;; *) false;; esac \
  && echo 'PASS: addon DNS la CoreDNS, Service ten kube-dns, co ClusterIP va dang READY'
```

**Ý nghĩa:** tên Deployment là `coredns` nhưng tên Service vẫn là `kube-dns` — đó là di sản của
addon cũ và là lý do bạn thấy hai cái tên khác nhau cho cùng một thứ. Địa chỉ `$DNS_IP` sẽ được đối
chiếu với `nameserver` trong `/etc/resolv.conf` của Pod ở B4.1.

**PASS:** dòng `PASS: addon DNS la CoreDNS, ...` xuất hiện.

### B0.4. Chế độ của kube-proxy

Vài gate phía sau phụ thuộc việc kube-proxy đang chạy ở chế độ nào. Ghi lại giá trị thật:

```bash
kubectl -n kube-system get cm kube-proxy -o jsonpath='{.data.config\.conf}' \
  | grep -E '^\s*mode:' | tee ~/lab-evidence/5a/b0-kube-proxy-mode.txt
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
```

**Ý nghĩa:** baseline để `mode` rỗng, nghĩa là kube-proxy tự chọn chế độ mặc định trên Linux. Nếu
cluster của bạn đang chạy `mode: ipvs` thì gate B2.4 hành xử khác — xem mục 4.

**PASS:** kube-proxy có đúng ba Pod `Running`, mỗi node một Pod.

## B1. Mô hình mạng: hai Pod trên hai node nói chuyện trực tiếp

Bài [81](../81-services-networking-vi.md) đặt ra một yêu cầu rất cứng: mọi Pod giao tiếp được với
mọi Pod khác, **trực tiếp, không qua proxy và không qua NAT**. Phần này biến câu đó thành phép đo.
Chưa có Service nào ở đây — Service chỉ xuất hiện từ B2.

### B1.1. Hai Pod, hai node

```bash
cat > ~/lab-work/5a/nettest.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: net-w1
  namespace: lab-5a
  labels:
    app: nettest
spec:
  nodeName: lab-k8s-worker1
  containers:
  - name: shell
    image: busybox:1.37
    command: ["sh", "-c", "sleep 36000"]
---
apiVersion: v1
kind: Pod
metadata:
  name: net-w2
  namespace: lab-5a
  labels:
    app: nettest
spec:
  nodeName: lab-k8s-worker2
  containers:
  - name: httpd
    image: busybox:1.37
    command:
      - sh
      - -c
      - 'mkdir -p /www && echo net-w2-ok > /www/index.html && httpd -f -p 8080 -h /www'
    ports:
      - containerPort: 8080
        name: http-w2
YAML

kubectl apply -f ~/lab-work/5a/nettest.yaml
kubectl wait --for=condition=Ready pod/net-w1 pod/net-w2 -n lab-5a --timeout=180s
kubectl get pods -n lab-5a -o wide | tee ~/lab-evidence/5a/b1-pods.txt
```

**Ý nghĩa:** `nodeName` ghim mỗi Pod lên đúng một worker. Để scheduler tự chọn thì hai Pod có thể
rơi vào cùng một node và phép thử mất ý nghĩa.

**PASS:** hai Pod `1/1 Running`, `net-w1` trên `lab-k8s-worker1`, `net-w2` trên `lab-k8s-worker2`.

### B1.2. Pod IP là địa chỉ thật của chính Pod

```bash
A_IP="$(kubectl get pod net-w1 -n lab-5a -o jsonpath='{.status.podIP}')"
B_IP="$(kubectl get pod net-w2 -n lab-5a -o jsonpath='{.status.podIP}')"
echo "net-w1 = $A_IP   net-w2 = $B_IP"

A_IN="$(kubectl exec -n lab-5a net-w1 -- ip -o -4 addr show dev eth0 \
  | awk '{print $4}' | cut -d/ -f1)"
echo "net-w1 tu thay dia chi cua no la: $A_IN"

test "$A_IP" = "$A_IN" \
  && echo 'PASS: dia chi ben trong Pod trung voi .status.podIP, khong co anh xa dia chi'
```

**Ý nghĩa:** đây là nửa đầu của "không NAT". Trong các hệ container cũ, container thấy một địa chỉ
riêng còn thế giới bên ngoài thấy một địa chỉ khác. Ở đây hai giá trị bằng nhau: Pod được đối xử
gần như một VM hay host vật lý.

**PASS:** dòng `PASS: dia chi ben trong Pod trung voi .status.podIP, ...` xuất hiện.

### B1.3. Gọi nhau bằng Pod IP, xuyên node, không cổng trung gian

```bash
kubectl exec -n lab-5a net-w1 -- wget -q -T 5 -O- "http://$B_IP:8080/" \
  | tee ~/lab-evidence/5a/b1-cross-node.txt

kubectl exec -n lab-5a net-w1 -- wget -q -T 5 -O- "http://$B_IP:8080/" \
  | grep -qx 'net-w2-ok' \
  && echo 'PASS: Pod tren worker1 goi thang Pod tren worker2 bang Pod IP'

case "$B_IP" in
  "${W2_CIDR%%/*}"*) echo "net-w2 IP nam trong PodCIDR cua worker2 ($W2_CIDR)";;
esac

kubectl exec -n lab-5a net-w1 -- wget -q -T 5 -O- "http://$B_IP:8081/" >/dev/null 2>&1 \
  && echo 'FAIL: port 8081 khong duoc mo ma van tra loi' \
  || echo 'PASS: chi dung port container mo la tra loi, khong co anh xa port nao khac'
```

**Ý nghĩa:** ba điều được chứng minh cùng lúc. Một, đích đến là **Pod IP** chứ không phải IP node —
không ai phải ánh xạ port container sang port host. Hai, địa chỉ đó thuộc PodCIDR của node **kia**,
nên gói tin thật sự đi xuyên node. Ba, chỉ đúng port mà container mở là trả lời — không có lớp
proxy nào chen vào giữa dịch port hộ bạn.

Ai làm cho đường đi này chạy được? **Không phải Kubernetes.** Bài 81 nói rõ Kubernetes chỉ định
nghĩa API, còn mạng Pod do hiện thực mạng Pod — CNI plugin đang cài trên cluster — hiện thực. Xác
nhận nó đang chạy, nhưng **không** đụng vào:

```bash
kubectl get pods -n kube-flannel -o wide | tee ~/lab-evidence/5a/b1-cni.txt
```

**PASS:** dòng `PASS: Pod tren worker1 goi thang Pod tren worker2 bang Pod IP` và dòng
`PASS: chi dung port container mo la tra loi, ...` cùng xuất hiện; CNI có một Pod trên mỗi node,
tất cả `Running`.

## B2. Service ClusterIP đầu tiên

### B2.1. Backend nói ra tên của chính nó

```bash
cat > ~/lab-work/5a/web.yaml <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: lab-5a
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: httpd
        image: busybox:1.37
        command:
          - sh
          - -c
          - 'mkdir -p /www && hostname > /www/index.html && httpd -f -p 8080 -h /www'
        ports:
        - containerPort: 8080
          name: http-web
YAML

kubectl apply -f ~/lab-work/5a/web.yaml
kubectl rollout status deployment/web -n lab-5a --timeout=180s
kubectl get pods -n lab-5a -l app=web -o wide | tee ~/lab-evidence/5a/b2-pods.txt

WEB_PODS="$(kubectl get pods -n lab-5a -l app=web --no-headers | wc -l)"
test "$WEB_PODS" -eq 3 && echo 'PASS: ba Pod backend dang chay'
```

**Ý nghĩa:** mỗi Pod trả về `hostname` của chính nó, nên ở B2.5 bạn phân biệt được lời gọi nào rơi
vào backend nào. Port container được **đặt tên** `http-web`; B2.2 sẽ dùng chính cái tên đó.

**PASS:** dòng `PASS: ba Pod backend dang chay` xuất hiện.

### B2.2. Service trỏ tới `targetPort` bằng tên

```bash
cat > ~/lab-work/5a/svc-web.yaml <<'YAML'
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: lab-5a
spec:
  selector:
    app: web
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http-web
YAML

kubectl apply -f ~/lab-work/5a/svc-web.yaml
kubectl get svc web -n lab-5a -o wide | tee ~/lab-evidence/5a/b2-svc.txt

CIP="$(kubectl get svc web -n lab-5a -o jsonpath='{.spec.clusterIP}')"
STYPE="$(kubectl get svc web -n lab-5a -o jsonpath='{.spec.type}')"
echo "ClusterIP = $CIP   type = $STYPE"

test -n "$CIP" && test "$CIP" != 'None' && test "$STYPE" = 'ClusterIP' \
  && echo 'PASS: Service duoc cap mot cluster IP va type mac dinh la ClusterIP'
```

**Ý nghĩa:** manifest **không** khai `type`, và Kubernetes điền `ClusterIP` — đúng giá trị mặc định
bài [82](../82-service-vi.md) mô tả. `targetPort` là **tên**, không phải số: nhờ đó backend đổi số
port ở phiên bản sau mà client không phải sửa gì.

**PASS:** dòng `PASS: Service duoc cap mot cluster IP va type mac dinh la ClusterIP` xuất hiện.

### B2.3. EndpointSlice là nơi kết quả của selector được ghi ra

```bash
kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web -o wide \
  | tee ~/lab-evidence/5a/b2-eps.txt

kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web \
  -o jsonpath='{range .items[*]}{"slice="}{.metadata.name}{" managed-by="}{.metadata.labels.endpointslice\.kubernetes\.io/managed-by}{" ownerKind="}{.metadata.ownerReferences[0].kind}{" ownerName="}{.metadata.ownerReferences[0].name}{" port="}{.ports[0].port}{" portName="}{.ports[0].name}{"\n"}{end}'

EP_PORT="$(kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web \
  -o jsonpath='{.items[0].ports[0].port}')"
EP_OWNER="$(kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web \
  -o jsonpath='{.items[0].metadata.ownerReferences[0].name}')"

EP_IPS="$(kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\n"}{end}' | sort -u)"
POD_IPS="$(kubectl get pods -n lab-5a -l app=web \
  -o jsonpath='{range .items[*]}{.status.podIP}{"\n"}{end}' | sort -u)"

echo "port trong EndpointSlice = $EP_PORT"
echo "owner = $EP_OWNER"

test "$EP_PORT" -eq 8080 \
  && test "$EP_OWNER" = 'web' \
  && test "$EP_IPS" = "$POD_IPS" \
  && echo 'PASS: EndpointSlice tu sinh, thuoc so huu Service web, va giai ten port ra so 8080'
```

**Ý nghĩa:** control plane tự tạo EndpointSlice cho **mọi Service có selector**. Ba dấu vết cho
biết slice này thuộc về ai: label `kubernetes.io/service-name`, owner reference trỏ về Service, và
label `endpointslice.kubernetes.io/managed-by` cho biết ai đang quản lý nó. Con số `8080` trong
slice chính là chỗ tên `http-web` được giải ra số thật.

**PASS:** dòng `PASS: EndpointSlice tu sinh, ...` xuất hiện.

### B2.4. ClusterIP là một địa chỉ IP ảo

```bash
ip -4 addr show | grep -w "$CIP" \
  && echo "FAIL: $CIP xuat hien tren mot interface cua node" \
  || echo "PASS: $CIP khong nam tren bat ky interface nao cua node"

kubectl exec -n lab-5a net-w1 -- wget -q -T 5 -O- "http://$CIP:80/" >/dev/null \
  && echo 'PASS: TCP toi ClusterIP van toi duoc backend'
```

**Ý nghĩa:** không có tiến trình nào lắng nghe trên `$CIP`, và địa chỉ đó không tồn tại trên bất kỳ
card mạng nào. Nó chỉ là một mục trong bảng quy tắc mà kube-proxy lập trình trên từng node. Đó là
toàn bộ nội dung của cụm từ "địa chỉ IP ảo" ở bài 82.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B2.5. Một địa chỉ, nhiều backend

```bash
: > ~/lab-evidence/5a/b2-loadbalance.txt
for i in $(seq 1 12); do
  kubectl exec -n lab-5a net-w1 -- wget -q -T 5 -O- "http://$CIP:80/" \
    >> ~/lab-evidence/5a/b2-loadbalance.txt
done
sort ~/lab-evidence/5a/b2-loadbalance.txt | uniq -c

HITS="$(sort -u ~/lab-evidence/5a/b2-loadbalance.txt | grep -c .)"
echo "so backend khac nhau da tra loi: $HITS"
test "$HITS" -ge 2 \
  && echo 'PASS: mot ClusterIP duy nhat trai luu luong ra nhieu Pod backend'
```

**Ý nghĩa:** client chỉ biết một địa chỉ và một port. Tập Pod phía sau đổi lúc nào cũng được — đúng
vấn đề mà bài 82 mở đầu: "làm sao frontend tìm ra và theo dõi được địa chỉ IP nào cần kết nối tới".

**PASS:** dòng `PASS: mot ClusterIP duy nhat trai luu luong ra nhieu Pod backend` xuất hiện.

## B3. EndpointSlice bám theo selector

### B3.1. Một Pod trần khớp selector cũng là backend

Service chọn **Pod**, không chọn Deployment. Chứng minh bằng một Pod không thuộc controller nào:

```bash
cat > ~/lab-work/5a/web-extra.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: web-extra
  namespace: lab-5a
  labels:
    app: web
spec:
  nodeName: lab-k8s-worker1
  terminationGracePeriodSeconds: 60
  containers:
  - name: httpd
    image: busybox:1.37
    command:
      - sh
      - -c
      - 'mkdir -p /www && hostname > /www/index.html && httpd -f -p 8080 -h /www'
    ports:
    - containerPort: 8080
      name: http-web
    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 30"]
YAML

kubectl apply -f ~/lab-work/5a/web-extra.yaml
kubectl wait --for=condition=Ready pod/web-extra -n lab-5a --timeout=180s

EXTRA_IP="$(kubectl get pod web-extra -n lab-5a -o jsonpath='{.status.podIP}')"
echo "ownerReferences cua web-extra: $(kubectl get pod web-extra -n lab-5a \
  -o jsonpath='{.metadata.ownerReferences}')"

count_eps() {
  kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web \
    -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\n"}{end}' | sort -u | grep -c .
}

for i in $(seq 1 30); do
  test "$(count_eps)" -eq 4 && break
  sleep 1
done
echo "so endpoint duy nhat = $(count_eps)"

test "$(count_eps)" -eq 4 \
  && kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web \
       -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\n"}{end}' \
     | grep -qx "$EXTRA_IP" \
  && echo 'PASS: Pod tran khop selector duoc dua vao EndpointSlice, khong can controller nao'
```

**Ý nghĩa:** `ownerReferences` của `web-extra` rỗng — nó không thuộc ReplicaSet nào, và ReplicaSet
của Deployment cũng **không** nhận nuôi nó vì selector của ReplicaSet còn đòi thêm nhãn
`pod-template-hash`. Nhưng Service `web` chỉ đòi `app: web`, nên nó vẫn là backend.

**PASS:** dòng `PASS: Pod tran khop selector duoc dua vao EndpointSlice, ...` xuất hiện.

### B3.2. Đổi label thì EndpointSlice đổi theo, cả hai chiều

```bash
kubectl label pod web-extra -n lab-5a app=web-tam --overwrite
for i in $(seq 1 30); do test "$(count_eps)" -eq 3 && break; sleep 1; done
echo "sau khi doi label: $(count_eps) endpoint"
test "$(count_eps)" -eq 3 && echo 'PASS: Pod khong con khop selector thi roi khoi EndpointSlice'

kubectl label pod web-extra -n lab-5a app=web --overwrite
for i in $(seq 1 30); do test "$(count_eps)" -eq 4 && break; sleep 1; done
echo "sau khi tra label: $(count_eps) endpoint"
test "$(count_eps)" -eq 4 && echo 'PASS: tra label ve thi Pod quay lai EndpointSlice'
```

**Ý nghĩa:** không có lệnh nào tên là "thêm backend vào Service". Thứ duy nhất bạn sửa là **label
của Pod**; controller của Service quét lại và ghi lại tập EndpointSlice. Đây là vòng lặp điều khiển
của giai đoạn 1, áp vào mạng.

**PASS:** cả hai dòng `PASS:` xuất hiện.

### B3.3. Ba condition của một endpoint

```bash
read_cond() {
  kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web \
    -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{" ready="}{.conditions.ready}{" serving="}{.conditions.serving}{" terminating="}{.conditions.terminating}{"\n"}{end}' \
    | grep "^$EXTRA_IP " || true
}

echo "truoc khi xoa: $(read_cond)"
kubectl delete pod web-extra -n lab-5a --wait=false

SEEN=''
for i in $(seq 1 90); do
  LINE="$(read_cond)"
  case "$LINE" in *terminating=true*) SEEN="$LINE"; break;; esac
  kubectl get pod web-extra -n lab-5a >/dev/null 2>&1 || break
  sleep 1
done
echo "trong luc ket thuc: $SEEN" | tee ~/lab-evidence/5a/b3-conditions.txt

case "$SEEN" in
  *ready=false*terminating=true*)
    echo 'PASS: endpoint dang ket thuc mang ready=false va terminating=true';;
  *)
    echo 'FAIL: khong bat duoc cua so terminating, xem muc 4';;
esac

kubectl wait --for=delete pod/web-extra -n lab-5a --timeout=120s
for i in $(seq 1 30); do test "$(count_eps)" -eq 3 && break; sleep 1; done
test "$(count_eps)" -eq 3 && echo 'PASS: endpoint bien mat han sau khi Pod da xoa xong'
```

**Ý nghĩa:** condition `terminating` được đặt **ngay khi Pod nhận timestamp xóa**, thường trước khi
container thoát — chính vì thế `preStop` và `terminationGracePeriodSeconds` ở B3.1 mới mở đủ rộng
cửa sổ để bạn nhìn thấy. `ready` là cách viết tắt cho "`serving` và không `terminating`", nên nó
chuyển `false` trước, trong khi `serving` vẫn còn `true` vì tiến trình vẫn đang phục vụ. Service
proxy thông thường bỏ qua endpoint đang `terminating`, trừ khi **mọi** endpoint khả dụng đều đang
terminating — đó là cơ chế giữ cho traffic không mất trong lúc rolling update ở Lab 4a.

**PASS:** dòng `PASS: endpoint dang ket thuc mang ready=false va terminating=true` và dòng
`PASS: endpoint bien mat han ...` xuất hiện.

### B3.4. Đọc một Service phải gộp tất cả slice của nó

```bash
kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web -o name \
  | tee ~/lab-evidence/5a/b3-slices.txt
N_SLICE="$(kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web -o name | grep -c .)"
N_RAW="$(kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{"\n"}{end}' | grep -c .)"
N_UNIQ="$(count_eps)"
echo "so slice=$N_SLICE  so dong endpoint=$N_RAW  so dia chi duy nhat=$N_UNIQ"

test "$N_UNIQ" -le "$N_RAW" && test "$N_SLICE" -ge 1 \
  && echo 'PASS: da gop moi slice roi khu trung lap truoc khi ket luan'
```

**Ý nghĩa:** ở quy mô lab bạn chỉ thấy một slice, nhưng cách đọc phải đúng ngay từ đầu: một Service
có thể có nhiều EndpointSlice, và **cùng một endpoint có thể xuất hiện ở nhiều slice**. Kết luận
"slice này có 3 endpoint nên Service có 3 backend" là sai về nguyên tắc. Hàm `count_eps` dùng suốt
lab đã gộp và `sort -u` đúng theo yêu cầu đó.

**PASS:** dòng `PASS: da gop moi slice roi khu trung lap truoc khi ket luan` xuất hiện.

## B4. DNS trong Pod và hai cách khám phá Service

### B4.1. `/etc/resolv.conf` do kubelet ghi

```bash
kubectl exec -n lab-5a net-w1 -- cat /etc/resolv.conf \
  | tee ~/lab-evidence/5a/b4-resolv.txt

kubectl exec -n lab-5a net-w1 -- cat /etc/resolv.conf \
  | grep -qx "nameserver $DNS_IP" \
  && echo 'PASS: nameserver trong Pod chinh la ClusterIP cua Service kube-dns'

kubectl exec -n lab-5a net-w1 -- cat /etc/resolv.conf \
  | grep -qE '^search lab-5a\.svc\.cluster\.local svc\.cluster\.local cluster\.local' \
  && echo 'PASS: search bat dau bang namespace cua Pod, roi svc.cluster.local, roi cluster.local'

kubectl exec -n lab-5a net-w1 -- cat /etc/resolv.conf \
  | grep -q 'ndots:5' \
  && echo 'PASS: options ndots:5 co mat'

DNSPOL="$(kubectl get pod net-w1 -n lab-5a -o jsonpath='{.spec.dnsPolicy}')"
echo "dnsPolicy = $DNSPOL"
test "$DNSPOL" = 'ClusterFirst' \
  && echo 'PASS: dnsPolicy mac dinh la ClusterFirst chu khong phai Default'
```

**Ý nghĩa:** ba dòng trong file này quyết định toàn bộ hành vi phân giải tên của Pod.
`nameserver` trỏ về CoreDNS qua **ClusterIP của Service `kube-dns`** — tức chính Pod cũng đang đi
qua một Service để tìm DNS. Danh sách `search` bắt đầu bằng namespace của chính Pod, đó là lý do
tên ngắn chỉ chạy trong namespace của nó. Và `Default` **không phải** giá trị mặc định: không khai
gì thì Kubernetes dùng `ClusterFirst`.

`ndots:5` nghĩa là một tên có **ít hơn 5 dấu chấm** được coi là tên tương đối và thử ghép lần lượt
với từng hậu tố trong `search` trước, chỉ khi hết mới thử nguyên văn. Số lần truy vấn thật sự đi
tới CoreDNS không đo được từ trong Pod bằng công cụ có sẵn, nên lab dừng ở chỗ kiểm chứng cấu hình
và kiểm chứng **hệ quả** của nó ở B4.2 và B4.3.

**PASS:** bốn dòng `PASS:` xuất hiện.

### B4.2. Tên ngắn, tên có namespace, và FQDN

```bash
kubectl exec -n lab-5a net-w1 -- nslookup web | tee ~/lab-evidence/5a/b4-nslookup-web.txt
kubectl exec -n lab-5a net-w1 -- nslookup web | grep -q "$CIP" \
  && echo 'PASS: ten ngan web phan giai duoc trong chinh namespace cua Pod'

kubectl exec -n lab-5a net-w1 -- nslookup web.lab-5a | grep -q "$CIP" \
  && echo 'PASS: web.lab-5a phan giai ra cung mot ClusterIP'

kubectl exec -n lab-5a net-w1 -- nslookup web.lab-5a.svc.cluster.local | grep -q "$CIP" \
  && echo 'PASS: FQDN day du phan giai ra cung mot ClusterIP'
```

**Ý nghĩa:** ba tên khác nhau, một địa chỉ. Dạng chuẩn là
`my-svc.my-namespace.svc.cluster-domain.example`; hai dạng còn lại chỉ là dạng chuẩn bị cắt bớt và
được danh sách `search` ghép lại.

**PASS:** ba dòng `PASS:` xuất hiện.

### B4.3. Tên ngắn không vượt được ranh giới namespace

Service `kubernetes` nằm ở namespace `default`, không phải `lab-5a`:

```bash
K8S_IP="$(kubectl get svc kubernetes -n default -o jsonpath='{.spec.clusterIP}')"
echo "ClusterIP cua Service kubernetes trong namespace default = $K8S_IP"

if kubectl exec -n lab-5a net-w1 -- nslookup kubernetes >/dev/null 2>&1; then
  echo 'FAIL: ten ngan lai phan giai duoc sang namespace khac'
else
  echo 'PASS: ten ngan kubernetes khong phan giai duoc tu namespace lab-5a'
fi

kubectl exec -n lab-5a net-w1 -- nslookup kubernetes.default | grep -q "$K8S_IP" \
  && echo 'PASS: ghi ro namespace thi phan giai duoc sang namespace khac'
```

**Ý nghĩa:** truy vấn không ghi namespace bị giới hạn trong namespace của Pod truy vấn. Đây không
phải cơ chế bảo mật — nó chỉ là hệ quả của danh sách `search`. Muốn sang namespace khác thì phải
nói ra tên namespace.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B4.4. Biến môi trường phụ thuộc thứ tự, DNS thì không

`net-w1` ra đời ở B1, **trước** khi Service `web` được tạo ở B2:

```bash
kubectl exec -n lab-5a net-w1 -- env | grep -E '^WEB_SERVICE_' \
  && echo 'FAIL: Pod cu lai co bien moi truong cua Service tao sau' \
  || echo 'PASS: Pod tao truoc Service khong duoc nap bien moi truong WEB_SERVICE_*'

kubectl run late-pod -n lab-5a --image=busybox:1.37 --restart=Never \
  --command -- sh -c 'sleep 36000'
kubectl wait --for=condition=Ready pod/late-pod -n lab-5a --timeout=180s

kubectl exec -n lab-5a late-pod -- env | grep -E '^WEB_SERVICE_' \
  | tee ~/lab-evidence/5a/b4-env.txt

ENV_HOST="$(kubectl exec -n lab-5a late-pod -- sh -c 'echo $WEB_SERVICE_HOST')"
ENV_PORT="$(kubectl exec -n lab-5a late-pod -- sh -c 'echo $WEB_SERVICE_PORT')"
echo "WEB_SERVICE_HOST=$ENV_HOST  WEB_SERVICE_PORT=$ENV_PORT"

test "$ENV_HOST" = "$CIP" && test "$ENV_PORT" = '80' \
  && echo 'PASS: Pod tao sau Service duoc nap dung ClusterIP va port vao bien moi truong'

kubectl exec -n lab-5a net-w1 -- nslookup web >/dev/null 2>&1 \
  && echo 'PASS: cung Pod cu do van tim duoc Service bang DNS, khong phu thuoc thu tu'
```

**Ý nghĩa:** kubelet nạp biến môi trường cho một Pod **tại thời điểm Pod ra đời**, và chỉ nạp các
Service đã tồn tại lúc đó. Nếu ứng dụng của bạn đọc `{SVCNAME}_SERVICE_HOST` thì bạn phải tạo
Service **trước** Pod client. DNS không có ràng buộc này — cùng một Pod `net-w1` không có biến môi
trường nhưng vẫn gọi được Service bằng tên.

**PASS:** ba dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

## B5. ClusterIP đến từ đâu

### B5.1. Đọc dải Service thật của cluster

```bash
sudo grep -n 'service-cluster-ip-range' /etc/kubernetes/manifests/kube-apiserver.yaml \
  | tee ~/lab-evidence/5a/b5-apiserver-flag.txt

SVC_CIDR="$(sudo awk -F= '/--service-cluster-ip-range=/{print $2}' \
  /etc/kubernetes/manifests/kube-apiserver.yaml | tr -d ' \r')"
echo "service-cluster-ip-range = $SVC_CIDR"

kubectl -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' \
  | grep -A2 'networking:' | tee -a ~/lab-evidence/5a/b5-apiserver-flag.txt

KUBEADM_SUBNET="$(kubectl -n kube-system get cm kubeadm-config \
  -o jsonpath='{.data.ClusterConfiguration}' \
  | awk -F': *' '/serviceSubnet:/{print $2}' | tr -d ' \r')"
echo "serviceSubnet trong kubeadm-config = $KUBEADM_SUBNET"

test -n "$SVC_CIDR" && test "$SVC_CIDR" = "$KUBEADM_SUBNET" \
  && echo 'PASS: dai Service doc tu manifest apiserver khop voi kubeadm-config'
```

**Ý nghĩa:** con số này là **thật của cluster bạn đang chạy**, không phải con số chép từ tài liệu.
Hai nguồn độc lập — cờ trên apiserver và ConfigMap `kubeadm-config` — phải khớp; lệch nhau nghĩa là
ai đó đã sửa tay một trong hai.

**PASS:** dòng `PASS: dai Service doc tu manifest apiserver khop voi kubeadm-config` xuất hiện.

### B5.2. ClusterIP đã cấp nằm trong dải đó

```bash
ip2int() { local a b c d; IFS=. read -r a b c d <<<"$1"; echo "$(( (a<<24)+(b<<16)+(c<<8)+d ))"; }
int2ip() { local n=$1; echo "$(( (n>>24)&255 )).$(( (n>>16)&255 )).$(( (n>>8)&255 )).$(( n&255 ))"; }

NET_STR="${SVC_CIDR%%/*}"
BITS="${SVC_CIDR##*/}"
NET_INT="$(ip2int "$NET_STR")"
SIZE=$(( 1 << (32 - BITS) ))
MASK=$(( (0xFFFFFFFF << (32 - BITS)) & 0xFFFFFFFF ))
echo "network=$NET_STR bits=$BITS kich thuoc dai=$SIZE"

in_range() {
  local ipi; ipi="$(ip2int "$1")"
  test $(( ipi & MASK )) -eq $(( NET_INT & MASK ))
}

for svc_ip in "$CIP" "$K8S_IP" "$DNS_IP"; do
  if in_range "$svc_ip"; then echo "$svc_ip nam trong $SVC_CIDR"
  else echo "FAIL: $svc_ip nam ngoai $SVC_CIDR"; fi
done | tee ~/lab-evidence/5a/b5-in-range.txt

grep -q '^FAIL' ~/lab-evidence/5a/b5-in-range.txt \
  && echo 'FAIL: co ClusterIP nam ngoai dai' \
  || echo 'PASS: moi ClusterIP dang dung deu nam trong dai da cau hinh'
```

**Ý nghĩa:** ClusterIP không phải số ngẫu nhiên. Nó luôn là một địa chỉ còn trống **bên trong dải đã
cấu hình cho API server**, và điều đó đúng cho cả Service bạn vừa tạo lẫn hai Service hệ thống.

**PASS:** dòng `PASS: moi ClusterIP dang dung deu nam trong dai da cau hinh` xuất hiện.

### B5.3. Băng tĩnh tính bằng công thức, và địa chỉ thứ 10

```bash
OFFSET=$(( SIZE / 16 ))
test "$OFFSET" -lt 16  && OFFSET=16
test "$OFFSET" -gt 256 && OFFSET=256
STATIC_START="$(int2ip $(( NET_INT + 1 )))"
STATIC_END="$(int2ip $(( NET_INT + OFFSET )))"
TENTH="$(int2ip $(( NET_INT + 10 )))"
echo "do lech bang = $OFFSET"
echo "bang tinh: $STATIC_START .. $STATIC_END"
echo "dia chi thu 10 cua dai = $TENTH"

test "$DNS_IP" = "$TENTH" \
  && echo 'PASS: DNS Service dung dia chi thu 10 cua dai, dung quy uoc bai 88 mo ta'

if in_range "$DNS_IP" && test "$(ip2int "$DNS_IP")" -le "$(ip2int "$STATIC_END")"; then
  echo 'PASS: dia chi cua DNS Service roi dung vao bang tinh'
fi
```

**Ý nghĩa:** công thức `min(max(16, cidrSize / 16), 256)` được áp vào **dải thật của bạn**, không
phải vào ví dụ trong bài. Kết quả giải thích luôn vì sao địa chỉ quen thuộc của DNS Service lại nằm
gọn trong băng tĩnh: cấp phát động mặc định lấy từ băng trên, nên băng dưới ít bị tranh chấp hơn.

Lưu ý cách nói của bài 88: quy ước "địa chỉ thứ 10" là **quy ước không chính thức của một số trình
cài đặt**, và băng tĩnh chỉ **giảm** nguy cơ va chạm chứ không đặt chỗ. Đừng viết công cụ dựa trên
giả định đó — hãy đọc từ Service `kube-dns` thật như B0.3 đã làm.

**PASS:** hai dòng `PASS:` xuất hiện.

### B5.4. Xin một ClusterIP tĩnh trong băng dưới

```bash
STATIC_IP="$(int2ip $(( NET_INT + 99 )))"
echo "se xin ClusterIP tinh = $STATIC_IP"

cat > ~/lab-work/5a/svc-static.yaml <<YAML
apiVersion: v1
kind: Service
metadata:
  name: web-static
  namespace: lab-5a
spec:
  clusterIP: $STATIC_IP
  selector:
    app: web
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http-web
YAML

kubectl apply -f ~/lab-work/5a/svc-static.yaml
GOT="$(kubectl get svc web-static -n lab-5a -o jsonpath='{.spec.clusterIP}')"
echo "ClusterIP thuc te = $GOT"

test "$GOT" = "$STATIC_IP" \
  && echo 'PASS: API server cap dung dia chi tinh da xin'

kubectl exec -n lab-5a net-w1 -- wget -q -T 5 -O- "http://$STATIC_IP:80/" >/dev/null \
  && echo 'PASS: Service dia chi tinh phuc vu binh thuong'
```

**PASS:** hai dòng `PASS:` xuất hiện. Nếu `apply` báo xung đột thì địa chỉ đó đã bị chiếm — đổi số
`99` sang một giá trị khác trong băng tĩnh rồi chạy lại.

### B5.5. ClusterIP ngoài dải bị từ chối

```bash
OUT_IP="$(int2ip $(( NET_INT + SIZE + 1 )))"
echo "se thu mot dia chi chac chan ngoai dai: $OUT_IP"

cat > ~/lab-work/5a/svc-outrange.yaml <<YAML
apiVersion: v1
kind: Service
metadata:
  name: web-outrange
  namespace: lab-5a
spec:
  clusterIP: $OUT_IP
  selector:
    app: web
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http-web
YAML

if kubectl apply --dry-run=server -f ~/lab-work/5a/svc-outrange.yaml \
     >~/lab-evidence/5a/b5-outrange.txt 2>&1; then
  echo 'FAIL: API server chap nhan ClusterIP ngoai dai'
else
  cat ~/lab-evidence/5a/b5-outrange.txt
  kubectl get svc web-outrange -n lab-5a >/dev/null 2>&1 \
    && echo 'FAIL: Service van duoc tao' \
    || echo 'PASS: API server tu choi ClusterIP ngoai dai va khong tao Service nao'
fi
```

**Ý nghĩa:** dải không phải quy ước lỏng lẻo — API server thực thi nó ngay ở tầng validation. Dùng
`--dry-run=server` để phép thử này không để lại rác trên cluster.

**PASS:** dòng `PASS: API server tu choi ClusterIP ngoai dai va khong tao Service nao` xuất hiện.

## B6. NodePort, ExternalName và LoadBalancer

### B6.1. NodePort mở cùng một port trên **mọi** node

```bash
cat > ~/lab-work/5a/svc-np.yaml <<'YAML'
apiVersion: v1
kind: Service
metadata:
  name: web-np
  namespace: lab-5a
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http-web
YAML

kubectl apply -f ~/lab-work/5a/svc-np.yaml
kubectl get svc web-np -n lab-5a -o wide | tee ~/lab-evidence/5a/b6-np.txt

NP="$(kubectl get svc web-np -n lab-5a -o jsonpath='{.spec.ports[0].nodePort}')"
NP_CIP="$(kubectl get svc web-np -n lab-5a -o jsonpath='{.spec.clusterIP}')"
echo "nodePort = $NP   clusterIP = $NP_CIP"

test "$NP" -ge 30000 && test "$NP" -le 32767 \
  && echo 'PASS: nodePort duoc cap trong dai mac dinh 30000-32767'

sudo grep -c 'service-node-port-range' /etc/kubernetes/manifests/kube-apiserver.yaml \
  | grep -qx '0' \
  && echo 'PASS: apiserver khong khai --service-node-port-range nen dai mac dinh dang co hieu luc'

test -n "$NP_CIP" && test "$NP_CIP" != 'None' \
  && echo 'PASS: Service NodePort van co cluster IP, dung thiet ke long nhau'

: > ~/lab-evidence/5a/b6-np-curl.txt
for ip in "$M_IP" "$W1_IP" "$W2_IP"; do
  code="$(curl -s -o /dev/null -m 5 -w '%{http_code}' "http://$ip:$NP/")"
  echo "$ip -> $code" | tee -a ~/lab-evidence/5a/b6-np-curl.txt
done

test "$(grep -c ' -> 200$' ~/lab-evidence/5a/b6-np-curl.txt)" -eq 3 \
  && echo 'PASS: ca ba node deu tra 200 tren cung mot nodePort'
```

**Ý nghĩa:** `NodePort` **cộng thêm** vào `ClusterIP` chứ không thay thế: Service vẫn có cluster IP,
và mọi node — kể cả node control plane không chạy backend nào — đều lắng nghe cùng một số port rồi
chuyển tiếp vào một endpoint sẵn sàng. Đây là ý nghĩa cụ thể của "thiết kế lồng nhau" ở bài 82.

**PASS:** bốn dòng `PASS:` xuất hiện.

### B6.2. `ExternalName` chỉ là một bản ghi CNAME

```bash
cat > ~/lab-work/5a/svc-extname.yaml <<'YAML'
apiVersion: v1
kind: Service
metadata:
  name: ext-db
  namespace: lab-5a
spec:
  type: ExternalName
  externalName: kubernetes.default.svc.cluster.local
YAML

kubectl apply -f ~/lab-work/5a/svc-extname.yaml
kubectl get svc ext-db -n lab-5a -o wide | tee ~/lab-evidence/5a/b6-extname.txt

EXT_CIP="$(kubectl get svc ext-db -n lab-5a -o jsonpath='{.spec.clusterIP}')"
EXT_SLICES="$(kubectl get endpointslice -n lab-5a \
  -l kubernetes.io/service-name=ext-db -o name | grep -c . || true)"
echo "clusterIP = '${EXT_CIP}'   so EndpointSlice = $EXT_SLICES"

test -z "$EXT_CIP" && test "$EXT_SLICES" -eq 0 \
  && echo 'PASS: ExternalName khong duoc cap cluster IP va khong co EndpointSlice nao'

kubectl exec -n lab-5a net-w1 -- nslookup ext-db.lab-5a.svc.cluster.local \
  | tee ~/lab-evidence/5a/b6-extname-nslookup.txt

kubectl exec -n lab-5a net-w1 -- nslookup ext-db.lab-5a.svc.cluster.local \
  | grep -q "$K8S_IP" \
  && echo 'PASS: ten ext-db duoc chuyen huong o tang DNS toi dung dich externalName'
```

**Ý nghĩa:** không có cluster IP, không có EndpointSlice, không có kube-proxy nào tham gia. Việc
chuyển hướng xảy ra **ở tầng DNS**: DNS Service của cluster trả về một bản ghi `CNAME` với giá trị
`externalName`, rồi client tự đi tiếp. Đây cũng là lý do bài 82 cảnh báo với HTTP và TLS: hostname
mà client dùng khác hostname mà server nhận, nên header `Host:` và certificate có thể không khớp.

**PASS:** hai dòng `PASS:` xuất hiện.

### B6.3. `LoadBalancer` treo vô thời hạn trên cluster bare metal

```bash
cat > ~/lab-work/5a/svc-lb.yaml <<'YAML'
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  namespace: lab-5a
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http-web
YAML

kubectl apply -f ~/lab-work/5a/svc-lb.yaml
kubectl get svc web-lb -n lab-5a -o wide | tee ~/lab-evidence/5a/b6-lb.txt

LB_CIP="$(kubectl get svc web-lb -n lab-5a -o jsonpath='{.spec.clusterIP}')"
LB_NP="$(kubectl get svc web-lb -n lab-5a -o jsonpath='{.spec.ports[0].nodePort}')"
LB_ING="$(kubectl get svc web-lb -n lab-5a -o jsonpath='{.status.loadBalancer.ingress}')"
LB_EXT="$(kubectl get svc web-lb -n lab-5a -o jsonpath='{.spec.externalIPs}')"
echo "clusterIP=$LB_CIP nodePort=$LB_NP ingress='${LB_ING}' externalIPs='${LB_EXT}'"

test -n "$LB_CIP" && test -n "$LB_NP" && test -z "$LB_ING" \
  && echo 'PASS: LoadBalancer van cap ClusterIP va NodePort, nhung status.loadBalancer rong'

curl -s -o /dev/null -m 5 -w 'nodePort cua web-lb -> %{http_code}\n' "http://$W1_IP:$LB_NP/"
```

**Ý nghĩa:** đây là toàn bộ phần `type: LoadBalancer` mà cluster lab kiểm chứng được. Kubernetes đã
làm xong phần việc của nó — cấp ClusterIP, cấp NodePort — và dừng lại chờ một thành phần **bên
ngoài** điền `.status.loadBalancer`. Trên bare metal không có cloud-controller-manager, nên
`EXTERNAL-IP` sẽ ở `<pending>` mãi mãi. Đó là hành vi đúng, không phải lỗi cluster.

Không cài MetalLB hay bất kỳ hiện thực load balancer nào ở lab này: nó đổi hạ tầng cluster, tức
phải nằm ở một lab tạo snapshot mới.

**PASS:** dòng `PASS: LoadBalancer van cap ClusterIP va NodePort, ...` xuất hiện; lời gọi qua
nodePort trả `200` — chứng tỏ tầng dưới của thiết kế lồng nhau vẫn hoạt động.

### B6.4. Bốn `type` cạnh nhau

```bash
kubectl get svc -n lab-5a \
  -o custom-columns='NAME:.metadata.name,TYPE:.spec.type,CLUSTER-IP:.spec.clusterIP,NODEPORT:.spec.ports[0].nodePort,EXTERNALNAME:.spec.externalName' \
  | tee ~/lab-evidence/5a/b6-types.txt

test "$(kubectl get svc -n lab-5a -o jsonpath='{range .items[*]}{.spec.type}{"\n"}{end}' \
  | sort -u | grep -c .)" -ge 3 \
  && echo 'PASS: bang tren cho thay it nhat ba type cung ton tai trong mot namespace'
```

**PASS:** bảng cho thấy `ClusterIP` có cluster IP và không có nodePort; `NodePort` và
`LoadBalancer` có cả hai; `ExternalName` không có cột nào ngoài `externalName`.

## B7. Hai chính sách traffic

Hai trường `externalTrafficPolicy` và `internalTrafficPolicy` đều làm đúng một việc: **bảo
kube-proxy lọc bớt tập endpoint** mà nó định tuyến tới. Muốn thấy hiệu ứng thì tập endpoint phải
lệch giữa các node, nên B7.0 ghim toàn bộ backend về một worker.

### B7.0. Backend chỉ nằm trên một node

```bash
cat > ~/lab-work/5a/one-node.yaml <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: onenode
  namespace: lab-5a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: onenode
  template:
    metadata:
      labels:
        app: onenode
    spec:
      nodeSelector:
        kubernetes.io/hostname: lab-k8s-worker1
      containers:
      - name: httpd
        image: busybox:1.37
        command:
          - sh
          - -c
          - 'mkdir -p /www && hostname > /www/index.html && httpd -f -p 8080 -h /www'
        ports:
        - containerPort: 8080
          name: http-one
YAML

kubectl apply -f ~/lab-work/5a/one-node.yaml
kubectl rollout status deployment/onenode -n lab-5a --timeout=180s
kubectl get pods -n lab-5a -l app=onenode -o wide | tee ~/lab-evidence/5a/b7-pods.txt

ON_NODES="$(kubectl get pods -n lab-5a -l app=onenode \
  -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort -u)"
echo "backend nam tren: $ON_NODES"
test "$ON_NODES" = 'lab-k8s-worker1' \
  && echo 'PASS: toan bo backend nam tren lab-k8s-worker1, worker2 khong co endpoint nao'
```

**PASS:** dòng `PASS: toan bo backend nam tren lab-k8s-worker1, ...` xuất hiện.

### B7.1. `externalTrafficPolicy`: `Cluster` so với `Local`

```bash
cat > ~/lab-work/5a/svc-etp.yaml <<'YAML'
apiVersion: v1
kind: Service
metadata:
  name: etp-cluster
  namespace: lab-5a
spec:
  type: NodePort
  externalTrafficPolicy: Cluster
  selector:
    app: onenode
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http-one
---
apiVersion: v1
kind: Service
metadata:
  name: etp-local
  namespace: lab-5a
spec:
  type: NodePort
  externalTrafficPolicy: Local
  selector:
    app: onenode
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http-one
YAML

kubectl apply -f ~/lab-work/5a/svc-etp.yaml

NP_C="$(kubectl get svc etp-cluster -n lab-5a -o jsonpath='{.spec.ports[0].nodePort}')"
NP_L="$(kubectl get svc etp-local   -n lab-5a -o jsonpath='{.spec.ports[0].nodePort}')"
HCNP="$(kubectl get svc etp-local -n lab-5a -o jsonpath='{.spec.healthCheckNodePort}')"
echo "nodePort Cluster=$NP_C  nodePort Local=$NP_L  healthCheckNodePort=$HCNP"

: > ~/lab-evidence/5a/b7-etp.txt
for pair in "master:$M_IP" "worker1:$W1_IP" "worker2:$W2_IP"; do
  name="${pair%%:*}"; ip="${pair##*:}"
  c="$(curl -s -o /dev/null -m 5 -w '%{http_code}' "http://$ip:$NP_C/")"
  l="$(curl -s -o /dev/null -m 5 -w '%{http_code}' "http://$ip:$NP_L/")"
  echo "$name  Cluster=$c  Local=$l" | tee -a ~/lab-evidence/5a/b7-etp.txt
done

test "$(grep -c 'Cluster=200' ~/lab-evidence/5a/b7-etp.txt)" -eq 3 \
  && echo 'PASS: externalTrafficPolicy Cluster — moi node deu phuc vu duoc'

test "$(grep -c 'Local=200' ~/lab-evidence/5a/b7-etp.txt)" -eq 1 \
  && grep -q '^worker1 .*Local=200' ~/lab-evidence/5a/b7-etp.txt \
  && echo 'PASS: externalTrafficPolicy Local — chi node co endpoint cuc bo moi phuc vu duoc'

test -n "$HCNP" \
  && echo 'PASS: Service Local duoc cap them mot healthCheckNodePort'
```

**Ý nghĩa:** `Cluster` là mặc định — mọi node nhận traffic rồi có thể nhảy thêm một chặng sang node
khác, đổi lại nó **che mất IP nguồn của client** vì gói tin bị viết lại địa chỉ nguồn ở chặng đó.
`Local` bỏ chặng nhảy thứ hai, giữ được IP nguồn, nhưng cái giá hiện ra ngay ở bảng trên: hai node
không có endpoint cục bộ **không phục vụ được**. Đó cũng là lý do Kubernetes cấp thêm
`healthCheckNodePort` — để load balancer bên ngoài biết node nào đang có backend mà bỏ qua node
không có.

Lab **không** đo trực tiếp giá trị IP nguồn mà backend nhìn thấy: busybox trong baseline không có
sẵn công cụ nào đọc được địa chỉ đối tác của một kết nối HTTP đã đóng, và thêm image khác là cài
phần mềm mới. Khác biệt về khả năng phục vụ ở trên là phần đo được, và nó đủ để phân biệt hai giá
trị.

**PASS:** ba dòng `PASS:` xuất hiện.

### B7.2. `internalTrafficPolicy: Local` nhìn từ hai worker

```bash
cat > ~/lab-work/5a/svc-itp.yaml <<'YAML'
apiVersion: v1
kind: Service
metadata:
  name: itp-local
  namespace: lab-5a
spec:
  internalTrafficPolicy: Local
  selector:
    app: onenode
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http-one
YAML

kubectl apply -f ~/lab-work/5a/svc-itp.yaml

ITP="$(kubectl get svc itp-local -n lab-5a -o jsonpath='{.spec.internalTrafficPolicy}')"
ITP_DEF="$(kubectl get svc web -n lab-5a -o jsonpath='{.spec.internalTrafficPolicy}')"
echo "itp-local = $ITP   web (khong khai gi) = $ITP_DEF"
test "$ITP" = 'Local' && test "$ITP_DEF" = 'Cluster' \
  && echo 'PASS: gia tri mac dinh la Cluster, chi Service khai tuong minh moi la Local'
```

`net-w1` nằm trên `lab-k8s-worker1` — node **có** endpoint. `net-w2` nằm trên `lab-k8s-worker2` —
node **không có** endpoint nào. Đây là bước duy nhất của lab cố ý tạo ra một lời gọi thất bại, và
nó chạy trên đúng node dành cho fault injection:

```bash
kubectl exec -n lab-5a net-w1 -- wget -q -T 5 -O- 'http://itp-local/' \
  | tee ~/lab-evidence/5a/b7-itp-w1.txt \
  && echo 'PASS: Pod tren worker1 goi duoc Service internalTrafficPolicy=Local'

if kubectl exec -n lab-5a net-w2 -- wget -q -T 5 -O- 'http://itp-local/' \
     >~/lab-evidence/5a/b7-itp-w2.txt 2>&1; then
  echo 'FAIL: Pod tren worker2 van goi duoc'
else
  echo 'PASS: Pod tren worker2 khong goi duoc — Service hanh xu nhu khong co endpoint nao'
fi

kubectl exec -n lab-5a net-w2 -- nslookup itp-local >/dev/null 2>&1 \
  && echo 'PASS: ten van phan giai duoc tu worker2 — van de nam o dinh tuyen, khong phai o DNS'
```

**Ý nghĩa:** đây chính là câu cảnh báo của bài 87 ở dạng đo được — với Pod trên node không có
endpoint, Service hành xử **như thể nó không có endpoint nào**, dù thực tế nó vẫn có hai endpoint
trên worker1. Chú ý bước cuối: DNS vẫn trả về ClusterIP bình thường. Người vận hành hay chẩn đoán
nhầm sang DNS, trong khi thứ đã lọc endpoint là **kube-proxy trên chính node đó**.

Trả Service về mặc định để không ảnh hưởng các bước sau:

```bash
kubectl delete svc itp-local -n lab-5a
```

**PASS:** bốn dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

## B8. Định tuyến nhận biết topology: bật được, không có tác dụng

### B8.1. Cluster lab không có zone

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{" zone="}{.metadata.labels.topology\.kubernetes\.io/zone}{"\n"}{end}' \
  | tee ~/lab-evidence/5a/b8-zones.txt

ZONES="$(kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.labels.topology\.kubernetes\.io/zone}{"\n"}{end}' \
  | grep -c . || true)"
echo "so node co label zone = $ZONES"
test "$ZONES" -eq 0 \
  && echo 'PASS: khong node nao mang label topology.kubernetes.io/zone'
```

**PASS:** dòng `PASS: khong node nao mang label topology.kubernetes.io/zone` xuất hiện.

### B8.2. Bật annotation và đọc lại

```bash
kubectl annotate svc web -n lab-5a service.kubernetes.io/topology-mode=Auto --overwrite
MODE="$(kubectl get svc web -n lab-5a \
  -o jsonpath='{.metadata.annotations.service\.kubernetes\.io/topology-mode}')"
echo "topology-mode = $MODE"
test "$MODE" = 'Auto' \
  && echo 'PASS: annotation topology-mode=Auto da duoc luu tren Service'
```

**Ý nghĩa:** phần **cấu hình được** dừng ở đây. Annotation là API, nó luôn ghi được. Việc nó có tác
dụng hay không lại do EndpointSlice controller quyết định, và đó là nội dung B8.3.

**PASS:** dòng `PASS: annotation topology-mode=Auto da duoc luu tren Service` xuất hiện.

### B8.3. Cơ chế bảo vệ số 3 chặn lại

Cho controller vài vòng đồng bộ sau khi annotation được ghi, rồi mới đọc kết quả:

```bash
for i in $(seq 1 10); do sleep 1; done

kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web -o yaml \
  | grep -nE 'hints|forZones|zone:' | tee ~/lab-evidence/5a/b8-hints.txt

HINTS="$(kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web \
  -o jsonpath='{range .items[*].endpoints[*]}{.hints}{"\n"}{end}' | grep -c . || true)"
ZONEF="$(kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=web \
  -o jsonpath='{range .items[*].endpoints[*]}{.zone}{"\n"}{end}' | grep -c . || true)"
echo "so endpoint co truong hints = $HINTS ; so endpoint co truong zone = $ZONEF"

test "$HINTS" -eq 0 && test "$ZONEF" -eq 0 \
  && echo 'PASS: khong endpoint nao duoc gan hints — dung nhu co che bao ve so 3 mo ta'

kubectl exec -n lab-5a net-w2 -- wget -q -T 5 -O- "http://$CIP:80/" >/dev/null \
  && echo 'PASS: kube-proxy van dung endpoint o moi noi, dinh tuyen khong doi'
```

**Ý nghĩa:** đây là kết quả **đúng**, không phải lỗi. Bài 86 liệt kê năm cơ chế bảo vệ; cơ chế số 3
nói rằng nếu bất kỳ node nào không có label `topology.kubernetes.io/zone` hoặc không báo cáo CPU
allocatable thì control plane **không đặt hint nào**, và kube-proxy do đó không lọc endpoint theo
zone. Cluster lab rơi đúng vào trường hợp đó với cả ba node.

Hai thứ nữa của bài 86 chỉ đọc, không dựng được ở đây: heuristic phân bổ theo **tỷ lệ CPU
allocatable của từng zone**, và ràng buộc "không dùng chung với `internalTrafficPolicy: Local` trên
cùng một Service". Ràng buộc thứ hai không phải luật API — API server vẫn nhận cả hai trường cùng
lúc — mà là quy tắc của controller, nên không có gate nào bắt được nó.

Gỡ annotation để trạng thái sạch trước các mục sau:

```bash
kubectl annotate svc web -n lab-5a service.kubernetes.io/topology-mode-
```

**PASS:** hai dòng `PASS:` xuất hiện.

## B9. Service không có selector, EndpointSlice tự viết và `port-forward`

### B9.1. Service không selector thì không có EndpointSlice nào tự sinh

```bash
cat > ~/lab-work/5a/svc-noselector.yaml <<'YAML'
apiVersion: v1
kind: Service
metadata:
  name: legacy
  namespace: lab-5a
spec:
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
YAML

kubectl apply -f ~/lab-work/5a/svc-noselector.yaml
kubectl get svc legacy -n lab-5a -o wide

LEG_CIP="$(kubectl get svc legacy -n lab-5a -o jsonpath='{.spec.clusterIP}')"
LEG_SEL="$(kubectl get svc legacy -n lab-5a -o jsonpath='{.spec.selector}')"
LEG_SLICES="$(kubectl get endpointslice -n lab-5a \
  -l kubernetes.io/service-name=legacy -o name | grep -c . || true)"
echo "clusterIP=$LEG_CIP selector='${LEG_SEL}' so slice=$LEG_SLICES"

test -n "$LEG_CIP" && test -z "$LEG_SEL" && test "$LEG_SLICES" -eq 0 \
  && echo 'PASS: Service khong selector van co cluster IP nhung khong co EndpointSlice tu sinh'

kubectl exec -n lab-5a net-w1 -- wget -q -T 5 -O- "http://$LEG_CIP:80/" >/dev/null 2>&1 \
  && echo 'FAIL: Service khong backend ma van tra loi' \
  || echo 'PASS: chua co endpoint nao thi khong ai tra loi'
```

**Ý nghĩa:** không có selector thì **không có gì để khớp**, nên control plane không tạo
EndpointSlice. Địa chỉ vẫn được cấp, tên vẫn phân giải được, nhưng phía sau trống rỗng.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B9.2. Tự viết EndpointSlice cho Service đó

```bash
cat > ~/lab-work/5a/eps-legacy.yaml <<YAML
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: legacy-1
  namespace: lab-5a
  labels:
    kubernetes.io/service-name: legacy
    endpointslice.kubernetes.io/managed-by: staff
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 8080
endpoints:
  - addresses:
      - "$B_IP"
    conditions:
      ready: true
YAML

kubectl apply -f ~/lab-work/5a/eps-legacy.yaml
kubectl get endpointslice legacy-1 -n lab-5a -o wide | tee ~/lab-evidence/5a/b9-eps.txt

for i in $(seq 1 30); do
  kubectl exec -n lab-5a net-w1 -- wget -q -T 3 -O- "http://legacy.lab-5a.svc.cluster.local/" \
    2>/dev/null | grep -qx 'net-w2-ok' && break
  sleep 1
done

kubectl exec -n lab-5a net-w1 -- wget -q -T 5 -O- "http://legacy.lab-5a.svc.cluster.local/" \
  | grep -qx 'net-w2-ok' \
  && echo 'PASS: Service khong selector van dinh tuyen duoc nho EndpointSlice ban tu viet'

MB="$(kubectl get endpointslice legacy-1 -n lab-5a \
  -o jsonpath='{.metadata.labels.endpointslice\.kubernetes\.io/managed-by}')"
OWN="$(kubectl get endpointslice legacy-1 -n lab-5a \
  -o jsonpath='{.metadata.ownerReferences}')"
echo "managed-by=$MB   ownerReferences='${OWN}'"
test "$MB" = 'staff' && test -z "$OWN" \
  && echo 'PASS: slice tu viet mang managed-by rieng va khong co owner reference'
```

**Ý nghĩa:** truy cập một Service không selector hoạt động **y hệt** khi nó có selector — client
không phân biệt được. Khác biệt nằm ở chỗ ai điền EndpointSlice: control plane, hay bạn. Giá trị
`managed-by` phải là giá trị riêng của bạn; giá trị dành riêng `"controller"` để đánh dấu slice do
chính control plane quản lý, và slice do bạn tạo cũng **không** có owner reference nào.

Bài 82 khuyên dùng đúng cách này cho backend nằm ngoài cluster, thay vì tạo tài nguyên `Endpoints`
cũ rồi để control plane phản chiếu — cơ chế phản chiếu đó đã bị loại bỏ dần.

**PASS:** hai dòng `PASS:` xuất hiện.

### B9.3. `port-forward`: tới Service có selector thì được, không selector thì không

```bash
kubectl port-forward -n lab-5a svc/web 18080:80 >~/lab-evidence/5a/b9-pf.log 2>&1 &
PF_PID=$!
for i in $(seq 1 20); do
  curl -fsS -m 2 http://127.0.0.1:18080/ >/dev/null 2>&1 && break
  sleep 1
done
curl -s -m 5 http://127.0.0.1:18080/ | tee -a ~/lab-evidence/5a/b9-pf.log
curl -s -o /dev/null -m 5 -w '%{http_code}\n' http://127.0.0.1:18080/ | grep -qx '200' \
  && echo 'PASS: port-forward toi Service co selector chay duoc'
kill "$PF_PID"

if kubectl port-forward -n lab-5a svc/legacy 18081:80 \
     >~/lab-evidence/5a/b9-pf-legacy.log 2>&1; then
  echo 'FAIL: port-forward toi Service khong selector lai thanh cong'
else
  cat ~/lab-evidence/5a/b9-pf-legacy.log
  echo 'PASS: port-forward toi Service khong selector bi API server tu choi'
fi
```

**Ý nghĩa:** `kubectl port-forward` không đi qua kube-proxy. Nó nhờ **API server** proxy tới một
Pod, nên nó cần Service phải ánh xạ được về Pod thật. Với Service không selector, endpoint có thể
trỏ tới bất cứ địa chỉ nào — API server từ chối để không bị biến thành proxy tới nơi mà bên gọi
không được phép truy cập.

Bài 366 cũng nhắc phần ủy quyền: `port-forward` cần quyền trên tài nguyên đích **và** subresource
`portforward`. Phần RBAC thuộc giai đoạn 9; ở đây bạn đang dùng kubeconfig quản trị của Lab 00 nên
quyền không phải vấn đề.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

## B10. Headless Service và định danh DNS cấp Pod

### B10.1. `clusterIP: None` và bốn Pod dùng chung một subdomain

```bash
cat > ~/lab-work/5a/headless.yaml <<'YAML'
apiVersion: v1
kind: Service
metadata:
  name: hl
  namespace: lab-5a
spec:
  clusterIP: None
  selector:
    app: hlpod
  ports:
  - name: http
    protocol: TCP
    port: 8080
    targetPort: 8080
---
apiVersion: v1
kind: Pod
metadata:
  name: hl-a
  namespace: lab-5a
  labels:
    app: hlpod
spec:
  hostname: h0
  subdomain: hl
  nodeName: lab-k8s-worker1
  containers:
  - name: httpd
    image: busybox:1.37
    command:
      - sh
      - -c
      - 'mkdir -p /www && hostname > /www/index.html && httpd -f -p 8080 -h /www'
---
apiVersion: v1
kind: Pod
metadata:
  name: hl-b
  namespace: lab-5a
  labels:
    app: hlpod
spec:
  hostname: h1
  subdomain: hl
  nodeName: lab-k8s-worker2
  containers:
  - name: httpd
    image: busybox:1.37
    command:
      - sh
      - -c
      - 'mkdir -p /www && hostname > /www/index.html && httpd -f -p 8080 -h /www'
---
apiVersion: v1
kind: Pod
metadata:
  name: hl-nohost
  namespace: lab-5a
  labels:
    app: hlpod
spec:
  subdomain: hl
  nodeName: lab-k8s-worker1
  containers:
  - name: httpd
    image: busybox:1.37
    command:
      - sh
      - -c
      - 'mkdir -p /www && hostname > /www/index.html && httpd -f -p 8080 -h /www'
---
apiVersion: v1
kind: Pod
metadata:
  name: hl-fqdn
  namespace: lab-5a
  labels:
    app: hlpod
spec:
  hostname: hf
  subdomain: hl
  setHostnameAsFQDN: true
  nodeName: lab-k8s-worker2
  containers:
  - name: httpd
    image: busybox:1.37
    command:
      - sh
      - -c
      - 'mkdir -p /www && hostname > /www/index.html && httpd -f -p 8080 -h /www'
YAML

kubectl apply -f ~/lab-work/5a/headless.yaml
kubectl wait --for=condition=Ready pod/hl-a pod/hl-b pod/hl-nohost pod/hl-fqdn \
  -n lab-5a --timeout=180s
kubectl get svc hl -n lab-5a -o wide | tee ~/lab-evidence/5a/b10-svc.txt

HL_CIP="$(kubectl get svc hl -n lab-5a -o jsonpath='{.spec.clusterIP}')"
HL_TYPE="$(kubectl get svc hl -n lab-5a -o jsonpath='{.spec.type}')"
echo "clusterIP=$HL_CIP  type=$HL_TYPE"
test "$HL_CIP" = 'None' && test "$HL_TYPE" = 'ClusterIP' \
  && echo 'PASS: headless Service co type ClusterIP nhung clusterIP la chuoi None'
```

**Ý nghĩa:** `None` là **trường hợp đặc biệt**, không giống với việc bỏ trống `.spec.clusterIP`. Bỏ
trống thì Kubernetes cấp một địa chỉ; ghi `None` thì nó không cấp gì cả và kube-proxy không xử lý
Service này.

**PASS:** dòng `PASS: headless Service co type ClusterIP nhung clusterIP la chuoi None` xuất hiện.

### B10.2. Tên Service phân giải thành **tập** IP của mọi Pod

```bash
kubectl get endpointslice -n lab-5a -l kubernetes.io/service-name=hl \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{" hostname="}{.hostname}{"\n"}{end}' \
  | tee ~/lab-evidence/5a/b10-eps.txt

kubectl exec -n lab-5a net-w1 -- nslookup hl.lab-5a.svc.cluster.local \
  | tee ~/lab-evidence/5a/b10-nslookup-svc.txt

MISS=0
for p in hl-a hl-b hl-nohost hl-fqdn; do
  ip="$(kubectl get pod "$p" -n lab-5a -o jsonpath='{.status.podIP}')"
  kubectl exec -n lab-5a net-w1 -- nslookup hl.lab-5a.svc.cluster.local 2>/dev/null \
    | grep -q "$ip" || { echo "thieu $p ($ip)"; MISS=1; }
done
test "$MISS" -eq 0 \
  && echo 'PASS: ten headless Service phan giai ra IP cua tat ca Pod ma no chon'
```

**Ý nghĩa:** Service bình thường phân giải thành **một** cluster IP; headless Service phân giải
thành **tập IP của tất cả Pod** mà nó chọn. Client được kỳ vọng dùng cả tập đó, hoặc tự chọn luân
phiên — không có ai cân bằng tải hộ.

Headless Service **có** selector nên control plane vẫn tạo EndpointSlice cho nó; cột `hostname`
trong slice chính là chỗ tên cấp Pod được lưu.

**PASS:** dòng `PASS: ten headless Service phan giai ra IP cua tat ca Pod ma no chon` xuất hiện.

### B10.3. `hostname` và `subdomain` quyết định bản ghi cấp Pod

```bash
A_HL_IP="$(kubectl get pod hl-a -n lab-5a -o jsonpath='{.status.podIP}')"
B_HL_IP="$(kubectl get pod hl-b -n lab-5a -o jsonpath='{.status.podIP}')"

kubectl exec -n lab-5a net-w1 -- nslookup h0.hl.lab-5a.svc.cluster.local \
  | tee ~/lab-evidence/5a/b10-nslookup-pod.txt
kubectl exec -n lab-5a net-w1 -- nslookup h0.hl.lab-5a.svc.cluster.local | grep -q "$A_HL_IP" \
  && echo 'PASS: h0.hl.lab-5a.svc.cluster.local tra ve dung IP cua Pod hl-a'
kubectl exec -n lab-5a net-w1 -- nslookup h1.hl.lab-5a.svc.cluster.local | grep -q "$B_HL_IP" \
  && echo 'PASS: h1.hl.lab-5a.svc.cluster.local tra ve dung IP cua Pod hl-b'

test "$(kubectl exec -n lab-5a hl-a -- hostname)" = 'h0' \
  && echo 'PASS: spec.hostname thang metadata.name khi dat hostname ben trong Pod'

if kubectl exec -n lab-5a net-w1 -- nslookup hl-nohost.hl.lab-5a.svc.cluster.local \
     >/dev/null 2>&1; then
  echo 'FAIL: Pod thieu hostname ma van co ban ghi cap Pod'
else
  echo 'PASS: Pod chi co subdomain ma thieu hostname thi khong co ban ghi cap Pod'
fi

NOHOST_IP="$(kubectl get pod hl-nohost -n lab-5a -o jsonpath='{.status.podIP}')"
kubectl exec -n lab-5a net-w1 -- nslookup hl.lab-5a.svc.cluster.local | grep -q "$NOHOST_IP" \
  && echo 'PASS: nhung no van nam trong tap IP cua chinh headless Service do'

FQDN_HOST="$(kubectl exec -n lab-5a hl-fqdn -- hostname)"
echo "hostname ben trong hl-fqdn = $FQDN_HOST"
test "$FQDN_HOST" = 'hf.hl.lab-5a.svc.cluster.local' \
  && echo 'PASS: setHostnameAsFQDN ghi ca FQDN vao hostname cua Pod'

kubectl exec -n lab-5a hl-a -- wget -q -T 5 -O- 'http://h1.hl.lab-5a.svc.cluster.local:8080/' \
  | grep -qx 'h1' \
  && echo 'PASS: mot Pod goi thang mot Pod khac bang ten, khong qua can bang tai'
```

**Ý nghĩa:** bản ghi cấp Pod cần **cả hai** trường. Thiếu `hostname` thì DNS không có tên nào để
tạo bản ghi, dù Pod vẫn là endpoint hợp lệ của Service. `setHostnameAsFQDN` chỉ đổi thứ Pod tự thấy
về mình — nhớ giới hạn 64 ký tự của hostname kernel Linux, FQDN dài hơn thì Pod kẹt ở
`ContainerCreating`.

Lời gọi cuối cùng là điểm khác biệt lớn nhất so với B2.5: ở đó bạn gọi một địa chỉ và không biết
Pod nào trả lời; ở đây bạn **chọn đích danh** một Pod.

**PASS:** bảy dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

## B11. Trả nợ #3 — Service quản trị cho StatefulSet

> **Đọc lại bài [65 — StatefulSets](../65-statefulset-vi.md), mục *Định danh mạng ổn định*, trước
> khi làm mục này.** Lab 4b đã dựng StatefulSet `web` với `serviceName` trỏ tới một Service chưa
> tồn tại và dừng lại đúng ở đó. Mục này dựng nốt phần còn thiếu.

Bài 65 xếp "StatefulSet yêu cầu một Headless Service do **bạn** tự tạo" vào mục *Hạn chế*: control
plane không tạo hộ. Miền do Service đó quản lý có dạng `$(service name).$(namespace).svc.cluster.local`,
và mỗi Pod nhận một subdomain `$(podname).$(governing service domain)`.

### B11.1. Service quản trị trước, StatefulSet sau

Thứ tự này quan trọng. Bài 65 cảnh báo về **negative caching**: nếu có client hỏi tên Pod trước khi
Pod tồn tại, kết quả thất bại được ghi nhớ và tái sử dụng trong một khoảng thời gian phụ thuộc cấu
hình cache của CoreDNS. Tạo Service trước và đừng tra tên trước khi Pod chạy.

```bash
cat > ~/lab-work/5a/sts.yaml <<'YAML'
apiVersion: v1
kind: Service
metadata:
  name: web-headless
  namespace: lab-5a
spec:
  clusterIP: None
  selector:
    app: sts-web
  ports:
  - name: web
    protocol: TCP
    port: 8080
    targetPort: 8080
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: lab-5a
spec:
  serviceName: web-headless
  replicas: 3
  selector:
    matchLabels:
      app: sts-web
  template:
    metadata:
      labels:
        app: sts-web
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: httpd
        image: busybox:1.37
        command:
          - sh
          - -c
          - 'mkdir -p /www && hostname > /www/index.html && httpd -f -p 8080 -h /www'
        ports:
        - containerPort: 8080
          name: web
YAML

kubectl apply -f ~/lab-work/5a/sts.yaml
kubectl rollout status statefulset/web -n lab-5a --timeout=300s
kubectl get pods -n lab-5a -l app=sts-web -o wide | tee ~/lab-evidence/5a/b11-pods.txt

SVCNAME="$(kubectl get statefulset web -n lab-5a -o jsonpath='{.spec.serviceName}')"
HL2_CIP="$(kubectl get svc web-headless -n lab-5a -o jsonpath='{.spec.clusterIP}')"
VCT="$(kubectl get statefulset web -n lab-5a -o jsonpath='{.spec.volumeClaimTemplates}')"
echo "serviceName=$SVCNAME  clusterIP cua Service quan tri=$HL2_CIP"
echo "volumeClaimTemplates='${VCT}'"

test "$SVCNAME" = 'web-headless' && test "$HL2_CIP" = 'None' \
  && echo 'PASS: Service quan tri da ton tai va la headless dung nhu bai 65 yeu cau'
test -z "$VCT" \
  && echo 'PASS: van khong co volumeClaimTemplates — no #2 chua tra, dung ke hoach'
```

**PASS:** hai dòng `PASS:` xuất hiện.

### B11.2. Nợ #3 được trả: mỗi Pod có tên DNS riêng

```bash
DOMAIN='web-headless.lab-5a.svc.cluster.local'
: > ~/lab-evidence/5a/b11-dns.txt
OK=1
for i in 0 1 2; do
  PIP="$(kubectl get pod "web-$i" -n lab-5a -o jsonpath='{.status.podIP}')"
  RES=''
  for t in $(seq 1 60); do
    RES="$(kubectl exec -n lab-5a net-w1 -- nslookup "web-$i.$DOMAIN" 2>/dev/null || true)"
    case "$RES" in *"$PIP"*) break;; esac
    sleep 2
  done
  echo "web-$i.$DOMAIN -> $PIP" | tee -a ~/lab-evidence/5a/b11-dns.txt
  case "$RES" in *"$PIP"*) ;; *) OK=0; echo "FAIL: web-$i chua phan giai duoc";; esac
done
test "$OK" -eq 1 \
  && echo 'PASS: NO #3 DA TRA — moi Pod cua StatefulSet co ban ghi DNS cap Pod rieng'

kubectl exec -n lab-5a web-0 -- wget -q -T 5 -O- "http://web-2.$DOMAIN:8080/" \
  | grep -qx 'web-2' \
  && echo 'PASS: web-0 goi dich danh web-2 bang ten, khong can biet IP'

kubectl exec -n lab-5a web-0 -- wget -q -T 5 -O- 'http://web-1.web-headless:8080/' \
  | grep -qx 'web-1' \
  && echo 'PASS: dang rut gon web-1.web-headless cung chay nho danh sach search'

kubectl exec -n lab-5a net-w1 -- nslookup "$DOMAIN" | tee ~/lab-evidence/5a/b11-domain.txt
```

**Ý nghĩa:** đây chính xác là thứ Lab 4b bỏ trống. Hostname `web-0` và các label
`statefulset.kubernetes.io/pod-name`, `apps.kubernetes.io/pod-index` đã có từ Lab 4b **mà không cần
Service nào**; thứ duy nhất Service quản trị thêm vào là **miền DNS**. Ghép hai phần lại mới ra
định danh mạng ổn định đầy đủ mà bài 65 mô tả.

Cột `Pod DNS` trong bảng của bài 65 giờ đọc được bằng số thật của cluster bạn:
`web-{0..2}.web-headless.lab-5a.svc.cluster.local`.

**PASS:** dòng `PASS: NO #3 DA TRA — ...` và hai dòng `PASS:` còn lại xuất hiện, không có dòng
`FAIL:`.

### B11.3. Job nói chuyện với nhau bằng hostname

Bài [354](../354-job-pod-to-pod-communication-vi.md) dùng đúng cơ chế trên cho Job: một headless
Service với selector `job-name`, cộng `subdomain` trong pod template, cộng chế độ hoàn thành
`Indexed` để hostname trở nên đoán trước được theo mẫu `${jobName}-${completionIndex}`.

```bash
cat > ~/lab-work/5a/ptp-job.yaml <<'YAML'
apiVersion: v1
kind: Service
metadata:
  name: ptp-headless
  namespace: lab-5a
spec:
  clusterIP: None
  selector:
    job-name: ptp-job
  ports:
  - name: http
    protocol: TCP
    port: 8080
    targetPort: 8080
---
apiVersion: batch/v1
kind: Job
metadata:
  name: ptp-job
  namespace: lab-5a
spec:
  completions: 3
  parallelism: 3
  completionMode: Indexed
  backoffLimit: 2
  template:
    spec:
      subdomain: ptp-headless
      restartPolicy: Never
      containers:
      - name: peer
        image: busybox:1.37
        command:
          - sh
          - -c
          - |
            mkdir -p /www
            hostname > /www/index.html
            httpd -p 8080 -h /www
            for i in 0 1 2; do
              until wget -q -T 2 -O- "http://ptp-job-${i}.ptp-headless:8080/" >/dev/null 2>&1
              do
                echo "chua goi duoc ptp-job-${i}.ptp-headless, thu lai"
                sleep 2
              done
              echo "da goi duoc ptp-job-${i}.ptp-headless"
            done
YAML

kubectl apply -f ~/lab-work/5a/ptp-job.yaml
kubectl wait --for=condition=Complete job/ptp-job -n lab-5a --timeout=300s \
  && echo 'PASS: Job chi Complete khi moi Pod goi duoc moi Pod bang hostname'

kubectl get pods -n lab-5a -l job-name=ptp-job \
  -o jsonpath='{range .items[*]}{.metadata.name}{" index="}{.metadata.annotations.batch\.kubernetes\.io/job-completion-index}{" hostname="}{.spec.hostname}{" subdomain="}{.spec.subdomain}{"\n"}{end}' \
  | tee ~/lab-evidence/5a/b11-job.txt

kubectl logs -n lab-5a -l job-name=ptp-job --tail=5 | tee -a ~/lab-evidence/5a/b11-job.txt
kubectl logs -n lab-5a -l job-name=ptp-job | grep -c 'da goi duoc' | \
  { read -r n; test "$n" -ge 3 && echo "PASS: co $n lan goi thanh cong ghi trong log"; }
```

**Ý nghĩa:** không Pod nào trong Job phải hỏi API server để biết địa chỉ của Pod khác. Chúng dựng
tên từ một công thức và để DNS lo phần còn lại — đúng lý do bài 354 đưa ra: workload không nên phụ
thuộc vào kết nối tới control plane chỉ để tìm nhau. Chú ý dạng tên rút gọn
`<pod-hostname>.<tên-headless-service>` chỉ chạy được khi `dnsPolicy` là `ClusterFirst`; đặt `None`
hay `Default` là hỏng.

**PASS:** hai dòng `PASS:` xuất hiện.

## B12. Hai tầng frontend–backend nối bằng Service

Bài [363](../363-connecting-frontend-backend-vi.md) dựng một backend sau Service ClusterIP và một
frontend tìm backend **bằng tên DNS của Service**. Bài expose frontend bằng `type: LoadBalancer`,
và chính bài cho phép thay bằng `NodePort` khi môi trường không hỗ trợ — cluster lab đúng vào
trường hợp đó, như B6.3 đã chứng minh.

### B12.1. Backend và Service của nó

```bash
cat > ~/lab-work/5a/hello.yaml <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: lab-5a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
      tier: backend
  template:
    metadata:
      labels:
        app: hello
        tier: backend
    spec:
      containers:
      - name: hello
        image: busybox:1.37
        command:
          - sh
          - -c
          - 'mkdir -p /www && echo hello-backend-ok > /www/index.html && httpd -f -p 8080 -h /www'
        ports:
        - containerPort: 8080
          name: http
---
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: lab-5a
spec:
  selector:
    app: hello
    tier: backend
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http
YAML

kubectl apply -f ~/lab-work/5a/hello.yaml
kubectl rollout status deployment/hello -n lab-5a --timeout=180s

kubectl exec -n lab-5a net-w1 -- wget -q -T 5 -O- 'http://hello/' | grep -qx 'hello-backend-ok' \
  && echo 'PASS: backend goi duoc tu trong cluster bang ten Service'

curl -s -o /dev/null -m 5 -w '%{http_code}\n' "http://$W1_IP:80/" | grep -qx '200' \
  && echo 'FAIL: backend lo ra ngoai cluster' \
  || echo 'PASS: Service backend chua truy cap duoc tu ngoai cluster'
```

**Ý nghĩa:** selector của Service `hello` dùng **hai** label `app` và `tier`. Đó là cách bài 363
tách backend khỏi frontend dù cả hai cùng mang `app: hello`.

**PASS:** hai dòng `PASS:` xuất hiện, không có dòng `FAIL:`.

### B12.2. Frontend tìm backend bằng tên DNS

```bash
cat > ~/lab-work/5a/frontend.yaml <<'YAML'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: lab-5a
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
      tier: frontend
  template:
    metadata:
      labels:
        app: hello
        tier: frontend
    spec:
      containers:
      - name: proxy
        image: busybox:1.37
        command:
          - sh
          - -c
          - |
            mkdir -p /www
            echo chua-goi-duoc-backend > /www/index.html
            ( while true; do
                if wget -q -T 3 -O /www/index.new 'http://hello/' 2>/dev/null; then
                  mv /www/index.new /www/index.html
                fi
                sleep 3
              done ) &
            httpd -f -p 8080 -h /www
        ports:
        - containerPort: 8080
          name: http
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: lab-5a
spec:
  type: NodePort
  selector:
    app: hello
    tier: frontend
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: http
YAML

kubectl apply -f ~/lab-work/5a/frontend.yaml
kubectl rollout status deployment/frontend -n lab-5a --timeout=180s

FNP="$(kubectl get svc frontend -n lab-5a -o jsonpath='{.spec.ports[0].nodePort}')"
echo "nodePort cua frontend = $FNP"

OUT=''
for i in $(seq 1 30); do
  OUT="$(curl -s -m 5 "http://$W1_IP:$FNP/")"
  test "$OUT" = 'hello-backend-ok' && break
  sleep 2
done
echo "$OUT" | tee ~/lab-evidence/5a/b12-frontend.txt

test "$OUT" = 'hello-backend-ok' \
  && echo 'PASS: goi frontend tu ngoai cluster va nhan duoc noi dung do backend sinh ra'

kubectl get svc -n lab-5a hello frontend -o wide | tee -a ~/lab-evidence/5a/b12-frontend.txt
```

**Ý nghĩa:** frontend **không** biết địa chỉ IP nào của backend. Nó chỉ biết chuỗi `hello` — chính
là trường `metadata.name` của Service backend — và mọi thứ còn lại do DNS cùng kube-proxy lo. Bạn
có thể scale backend, xóa Pod, đổi node, cấu hình frontend vẫn nguyên.

Đó là câu trả lời trọn vẹn cho vấn đề mà bài 82 đặt ra ở dòng đầu tiên: frontend tìm ra và theo dõi
backend bằng cách nào.

**PASS:** dòng `PASS: goi frontend tu ngoai cluster va nhan duoc noi dung do backend sinh ra`
xuất hiện.

## B13. Cleanup và gate cuối

**Mục đích:** xóa mọi object lab tạo ra và chứng minh cluster trở về `01-cluster-ready`.

Trước khi xóa, đếm lại số Service đã dựng để biết mình phải dọn những gì:

```bash
kubectl get svc,endpointslice,statefulset,deployment,job,pods -n lab-5a \
  | tee ~/lab-evidence/5a/b13-truoc-cleanup.txt
kubectl get svc -n lab-5a --no-headers | wc -l
```

Bài [340](../340-delete-stateful-set-vi.md) — đã thực hành ở Lab 4b — nhắc rằng Service quản trị
của StatefulSet **không** bị xóa theo khi bạn xóa StatefulSet: nó là object độc lập do bạn tạo.
Ở đây xóa cả namespace nên mọi thứ đi cùng một lượt, nhưng hãy nhớ điều đó khi dọn tay trên cluster
thật:

```bash
kubectl delete namespace lab-5a --wait=true --timeout=300s

if kubectl get namespace lab-5a >/dev/null 2>&1; then
  echo 'FAIL: namespace lab-5a van ton tai'
else
  echo 'PASS: namespace lab-5a da bien mat'
fi
```

Gỡ manifest tạm. `rmdir` cố ý không dùng `-rf`: nó fail nếu còn file lạ, và `test ! -e` biến điều đó
thành gate thay vì im lặng bỏ qua. Bằng chứng ở `~/lab-evidence/5a/` được giữ lại.

```bash
rm -f ~/lab-work/5a/nettest.yaml ~/lab-work/5a/web.yaml ~/lab-work/5a/svc-web.yaml \
      ~/lab-work/5a/web-extra.yaml ~/lab-work/5a/svc-static.yaml \
      ~/lab-work/5a/svc-outrange.yaml ~/lab-work/5a/svc-np.yaml \
      ~/lab-work/5a/svc-extname.yaml ~/lab-work/5a/svc-lb.yaml \
      ~/lab-work/5a/one-node.yaml ~/lab-work/5a/svc-etp.yaml ~/lab-work/5a/svc-itp.yaml \
      ~/lab-work/5a/svc-noselector.yaml ~/lab-work/5a/eps-legacy.yaml \
      ~/lab-work/5a/headless.yaml ~/lab-work/5a/sts.yaml ~/lab-work/5a/ptp-job.yaml \
      ~/lab-work/5a/hello.yaml ~/lab-work/5a/frontend.yaml
rmdir ~/lab-work/5a
test ! -e ~/lab-work/5a && echo 'PASS: manifest tam da duoc xoa het'

jobs -p | xargs -r kill 2>/dev/null
```

Gate cuối, chạy trên `lab-k8s-master`:

```bash
kubectl wait --for=condition=Ready node --all --timeout=180s
test "$(kubectl get --raw='/readyz')" = 'ok' && echo 'PASS: readyz ok'
kubectl get nodes
kubectl get pods -n default
kubectl get svc -n default
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded
kubectl -n kube-system get deployment coredns

kubectl get svc,endpointslice -A | grep -E 'lab-5a' \
  && echo 'FAIL: con Service hoac EndpointSlice cua lab-5a' \
  || echo 'PASS: khong con Service hay EndpointSlice nao cua lab-5a'

kubectl get deployment,statefulset,job,pods -A | grep -E 'lab-5a' \
  && echo 'FAIL: con workload cua lab-5a' \
  || echo 'PASS: khong con workload nao cua lab-5a tren cluster'

test "$(kubectl get svc -n default --no-headers | wc -l)" -eq 1 \
  && echo 'PASS: namespace default chi con Service kubernetes'
```

**PASS:** không có dòng `FAIL:` nào; ba node `Ready`; `PASS: readyz ok` xuất hiện; namespace
`default` không có Pod và chỉ còn Service `kubernetes`; lệnh field selector trả `No resources found`;
CoreDNS đủ replica. Cluster trở về `01-cluster-ready`; **không** chụp snapshot mới.

---

## 3. Checkpoint 5a

Tự trả lời không nhìn tài liệu:

- [ ] Phát biểu mô hình mạng Kubernetes bằng đúng ba ràng buộc của bài 81, rồi chỉ ra trên cluster
      của mình phép đo nào chứng minh từng ràng buộc — và nói ai hiện thực chúng, Kubernetes hay
      thứ khác.
- [ ] Giải thích vì sao ClusterIP được gọi là "địa chỉ IP ảo", dựa vào chính kết quả B2.4: địa chỉ
      không nằm trên interface nào mà TCP tới nó vẫn tới được backend. Trên cùng một Service đó,
      nói `port`, `targetPort` và port ghi trong EndpointSlice khác nhau ở đâu, và lợi ích cụ thể
      của việc đặt `targetPort` bằng **tên** thay vì bằng số.
- [ ] Mô tả đường đi hai chiều giữa label của Pod và nội dung EndpointSlice; nói vì sao đọc một
      slice rồi kết luận số backend là sai về nguyên tắc.
- [ ] Kể ba condition của một endpoint, thời điểm mỗi cái đổi giá trị khi Pod bị xóa, và ngoại lệ
      duy nhất khiến service proxy vẫn gửi traffic tới endpoint đang `terminating`.
- [ ] Xếp bốn `type` của Service theo thiết kế lồng nhau; nói `ExternalName` phá vỡ mạch lồng nhau
      đó ở chỗ nào, và vì sao dùng nó với HTTP hay TLS lại dễ hỏng.
- [ ] Đọc `/etc/resolv.conf` của một Pod và giải thích được từng dòng; nói vì sao tên ngắn chỉ chạy
      trong namespace của chính Pod, và `ndots:5` đổi thứ tự thử tên như thế nào.
- [ ] So sánh hai cách khám phá Service; nêu ràng buộc thứ tự của cách thứ nhất và chứng minh bằng
      chính hai Pod bạn đã tạo ở B1 và B4.4.
- [ ] Với dải Service thật của cluster mình: tính độ lệch băng, chỉ ra băng tĩnh chạy từ đâu đến
      đâu, nói nên xin ClusterIP tĩnh ở vùng nào và vì sao cơ chế đó **không** phải một chỗ đặt chỗ.
- [ ] Mô tả hệ quả của `externalTrafficPolicy: Local` và `internalTrafficPolicy: Local` bằng chính
      số đo ở B7; nói mỗi trường chi phối loại lưu lượng nào và cái giá phải trả là gì.
- [ ] Cluster lab đặt được annotation `topology-mode: Auto` nhưng không có hint nào được sinh. Chỉ
      ra đúng cơ chế bảo vệ nào đã chặn, và nói thành phần nào **đặt** hint, thành phần nào **dùng**.
- [ ] Nêu đủ điều kiện để một Pod có bản ghi DNS riêng; giải thích vì sao `hl-nohost` không có bản
      ghi cấp Pod nhưng vẫn nằm trong tập IP của headless Service.
- [ ] **Nợ #3 đã trả ở lab này:** nói StatefulSet lấy được gì từ Service quản trị headless và
      **không** lấy được gì từ nó; chứng minh bằng việc Lab 4b vẫn có định danh ổn định dù chưa có
      Service nào. Xác nhận nợ #2 và nợ #1 của bài 65 **vẫn còn treo** sau lab này.

### Bài giải thích cuối cùng

Trong tối đa 10 phút, kể lại luồng sau mà không mở tài liệu:

1. Bạn `kubectl apply` một Deployment ba replica. CNI cấp cho mỗi Pod một địa chỉ IP duy nhất trên
   toàn cluster; mọi Pod gọi được mọi Pod, không proxy, không NAT.
2. Bạn tạo một Service có selector. API server cấp cho nó một ClusterIP còn trống trong dải
   `service-cluster-ip-range`, và controller của Service ghi tập Pod khớp selector ra EndpointSlice.
3. CoreDNS theo dõi API và tạo bản ghi cho `web.lab-5a.svc.cluster.local` trỏ về ClusterIP đó.
4. Một Pod client gọi `http://web/`. Danh sách `search` trong `/etc/resolv.conf` ghép tên ngắn
   thành FQDN, DNS trả về ClusterIP, và kube-proxy trên node của client viết lại đích tới một
   endpoint sẵn sàng. Không có tiến trình nào lắng nghe trên ClusterIP.
5. Một Pod backend bị xóa. Endpoint của nó mang `terminating=true` và `ready=false` ngay khi nhận
   timestamp xóa, service proxy bỏ qua nó, rồi endpoint biến mất khỏi slice.
6. Bạn đổi `type` sang `NodePort`. Service giữ nguyên ClusterIP và có thêm một port tĩnh mở trên
   **mọi** node. Đổi tiếp sang `LoadBalancer` thì có thêm một chỗ chờ thành phần bên ngoài điền —
   trên bare metal, chỗ đó rỗng mãi mãi.
7. Bạn cần gọi đích danh từng Pod chứ không phải một địa chỉ chung. Đặt `clusterIP: None`, DNS
   chuyển từ trả một ClusterIP sang trả tập IP của mọi Pod, và cặp `hostname` + `subdomain` sinh
   thêm bản ghi cho từng Pod.
8. StatefulSet là chỗ cần đúng thứ đó: hostname `web-0` do controller đặt, miền DNS do Service quản
   trị headless đặt; ghép lại thành `web-0.web-headless.lab-5a.svc.cluster.local`.

Khi mọi checkbox được đánh dấu và bạn không còn nhầm ClusterIP với một địa chỉ có thật, không nhầm
selector với EndpointSlice, không nhầm `internalTrafficPolicy` với `externalTrafficPolicy`, thì Lab
5a hoàn tất. Phần còn lại của giai đoạn 5 — NetworkPolicy, Ingress và CNI — nằm ở **Lab 5b**, và lab
đó đổi hạ tầng mạng nên nó tạo snapshot `02-net-ready`.

---

## 4. Troubleshooting của lab này

Sự cố khi dựng môi trường nằm ở [mục 4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#4-troubleshooting-môi-trường).
Bảng dưới chỉ liệt kê sự cố phát sinh từ nội dung bài học 5a.

| Triệu chứng | Kiểm tra | Xử lý đúng phạm vi |
| --- | --- | --- |
| B1.3 không gọi được xuyên node | `kubectl get pod -o wide`; UDP `8472` giữa hai node | Đây là sự cố tầng 3 của Lab 00, không phải nội dung 5a; chạy lại [A5.4.4](LAB-00-MOI-TRUONG-1.35.7.md#a544-tầng-3--pod-networking-xuyên-node) trước khi tiếp tục |
| B2.3 báo `EP_PORT` khác 8080 | `kubectl get pod -o jsonpath='{.spec.containers[0].ports}'` | Tên port trên Pod không phải `http-web` nên `targetPort` không giải ra được; sửa lại `web.yaml` rồi apply lại cả Deployment lẫn Service |
| B2.4 thấy ClusterIP nằm trên một interface | `~/lab-evidence/5a/b0-kube-proxy-mode.txt` | kube-proxy đang chạy `mode: ipvs`, chế độ này gắn cluster IP lên `kube-ipvs0`. Kết luận "địa chỉ ảo" vẫn đúng nhưng gate viết cho chế độ mặc định; ghi lại và đi tiếp, **không** đổi chế độ kube-proxy |
| B2.5 chỉ một backend trả lời trong 12 lần gọi | `kubectl get endpointslice ... -o yaml` | Nếu EndpointSlice có đủ ba địa chỉ thì đây là ngẫu nhiên của phân phối; chạy lại vòng lặp. Nếu chỉ có một địa chỉ thì hai Pod kia chưa Ready — chờ `rollout status` xong rồi làm lại |
| B3.1 đếm ra 3 thay vì 4 endpoint | `kubectl get pod web-extra --show-labels` | Pod chưa Ready, hoặc label `app` bị ghi sai; endpoint chưa Ready thì `ready=false` nhưng vẫn nằm trong slice — đọc kỹ output của `read_cond` |
| B3.3 không bắt được cửa sổ `terminating` | `kubectl get pod web-extra -o yaml \| grep -A3 preStop` | `preStop` hoặc `terminationGracePeriodSeconds` bị mất khỏi manifest nên Pod biến mất quá nhanh; apply lại đúng `web-extra.yaml` rồi chạy lại B3.1 và B3.3 |
| B4.1 dòng `search` có thêm hậu tố lạ ở cuối | `cat /etc/resolv.conf` **trên node** | Với `dnsPolicy: ClusterFirst`, kubelet nối thêm search domain của node vào sau ba hậu tố của cluster. Gate dùng `grep -E '^search ...'` nên vẫn PASS; đó là hành vi bình thường |
| B5.1 `SVC_CIDR` rỗng | `sudo grep -n 'cluster-ip-range' /etc/kubernetes/manifests/kube-apiserver.yaml` | Lệnh chạy không có `sudo`, hoặc cờ nằm trên nhiều dòng; đọc thẳng file rồi gán tay biến `SVC_CIDR` trước khi sang B5.2 |
| B5.4 báo xung đột ClusterIP | Thông báo lỗi của `kubectl apply` | Địa chỉ tĩnh đã bị Service khác chiếm — đúng điều bài 88 cảnh báo. Đổi số `99` sang giá trị khác trong băng tĩnh rồi chạy lại; **không** hạ tay ClusterIP của Service hệ thống |
| B6.1 chỉ 1/3 node trả `200` | `~/lab-evidence/5a/b6-np-curl.txt`; UFW trên hai node còn lại | NodePort không forward xuyên node; đây lại là tầng 3 của Lab 00 — xử lý ở đó trước, không sửa Service |
| B6.2 `nslookup ext-db` không ra địa chỉ | `kubectl get svc ext-db -o yaml` | `externalName` trỏ tới một tên không phân giải được; lab dùng `kubernetes.default.svc.cluster.local` để không phụ thuộc internet — kiểm tra lại chuỗi đó có bị gõ sai không |
| B6.3 `EXTERNAL-IP` không ở `<pending>` mà có địa chỉ | `kubectl get pods -A \| grep -iE 'metallb\|cloud-controller'` | Có một hiện thực load balancer nào đó đang chạy — cluster không còn đúng baseline. Dừng lại và đối chiếu với [A1.4 của Lab 00](LAB-00-MOI-TRUONG-1.35.7.md#a14-phần-stack-không-thuộc-lab-00) |
| B7.1 cả ba node đều trả `200` với `etp-local` | `kubectl get pods -l app=onenode -o wide` | Backend không nằm hết trên một node; `nodeSelector` bị sửa hoặc `kubernetes.io/hostname` khác tên. Xóa Deployment `onenode`, sửa lại rồi chạy lại B7.0 |
| B7.2 Pod trên worker2 vẫn gọi được | `kubectl get svc itp-local -o jsonpath='{.spec.internalTrafficPolicy}'` | Trường bị bỏ sót nên Service đang ở `Cluster`; apply lại `svc-itp.yaml`. Nếu trường đúng là `Local` mà vẫn gọi được thì có backend đã lên worker2 — kiểm lại B7.0 |
| B8.3 thấy có trường `hints` | `kubectl get nodes --show-labels \| grep zone` | Có ai đó đã gắn label zone cho node. Cluster không còn đúng baseline; gỡ label do mình gắn, **không** gắn thêm label để "cho tính năng chạy" |
| B9.2 gọi `legacy` không ra kết quả | `kubectl get endpointslice legacy-1 -o yaml` | Tên port trong EndpointSlice phải khớp tên port trong Service (`http`), và `conditions.ready` phải là `true`; sửa rồi apply lại, chờ vài vòng lặp cho kube-proxy cập nhật |
| B9.3 `port-forward` tới `svc/web` cũng lỗi | `~/lab-evidence/5a/b9-pf.log` | Port `18080` đang bị chiếm trên master, hoặc tiến trình `port-forward` cũ chưa `kill`; đổi số port cục bộ rồi chạy lại |
| B10.3 `h0.hl...` không phân giải được | `kubectl get endpointslice -l kubernetes.io/service-name=hl -o yaml` | Trường `hostname` trong slice rỗng nghĩa là Pod thiếu `spec.hostname`; hoặc bạn đã tra tên này **trước** khi Pod Ready và đang gặp negative caching — chờ hết chu kỳ cache rồi tra lại |
| B11.2 một vài `web-N` không phân giải được | `kubectl get pods -l app=sts-web -o wide` | Pod chưa Ready thì chưa có bản ghi; hoặc đã có truy vấn thất bại trước đó bị cache lại. Vòng lặp trong bước này đã thử lại nhiều lần — nếu vẫn fail, xóa StatefulSet, chờ Pod biến mất hết rồi apply lại `sts.yaml` |
| B11.3 Job không `Complete` | `kubectl logs -l job-name=ptp-job` | Log lặp "chua goi duoc" nghĩa là selector `job-name` của headless Service không khớp tên Job, hoặc `subdomain` trong pod template gõ sai. Xóa Job và Service rồi apply lại cả file |
| B12.2 luôn trả `chua-goi-duoc-backend` | `kubectl exec deploy/frontend -- wget -q -O- http://hello/` | Frontend không phân giải được tên `hello`; kiểm tra Service backend còn tồn tại và selector hai label còn khớp. Vòng lặp trong container chỉ thử lại, nó không báo lỗi ra ngoài |
| Namespace `lab-5a` kẹt `Terminating` | `kubectl get all -n lab-5a`; `kubectl get ns lab-5a -o yaml` | Chờ Pod hết `terminationGracePeriodSeconds`; nếu state khác baseline thì restore **cả ba VM** về `01-cluster-ready` |

---

## 5. Nguồn chính thức

- [Kubernetes v1.35 — Services, Load Balancing, and Networking](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/)
- [Kubernetes v1.35 — Service](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/service/)
- [Kubernetes v1.35 — EndpointSlices](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
- [Kubernetes v1.35 — DNS for Services and Pods](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Kubernetes v1.35 — Pod Hostname](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/pods/pod-hostname/)
- [Kubernetes v1.35 — Topology Aware Routing](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)
- [Kubernetes v1.35 — Service Internal Traffic Policy](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/service-traffic-policy/)
- [Kubernetes v1.35 — Service ClusterIP allocation](https://v1-35.docs.kubernetes.io/docs/concepts/services-networking/cluster-ip-allocation/)
- [Kubernetes v1.35 — Virtual IPs and Service Proxies](https://v1-35.docs.kubernetes.io/docs/reference/networking/virtual-ips/)
- [Kubernetes v1.35 — StatefulSets](https://v1-35.docs.kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Kubernetes v1.35 — Configure DNS for a Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/configure-dns-cluster/)
- [Kubernetes v1.35 — Connect a Frontend to a Backend Using Services](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/connecting-frontend-backend/)
- [Kubernetes v1.35 — Create an External Load Balancer](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/)
- [Kubernetes v1.35 — Job with Pod-to-Pod Communication](https://v1-35.docs.kubernetes.io/docs/tasks/job/job-with-pod-to-pod-communication/)
- [Kubernetes v1.35 — Use Port Forwarding to Access Applications in a Cluster](https://v1-35.docs.kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)
